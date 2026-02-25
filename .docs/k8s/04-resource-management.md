# Resource Management — Best Practices

## 1. Requests e Limits

### 1.1 Conceitos Fundamentais
| Campo | Descrição | Impacto |
|-------|-----------|---------|
| **requests.cpu** | CPU garantida para o Pod | Usado para scheduling |
| **requests.memory** | Memória garantida para o Pod | Usado para scheduling |
| **limits.cpu** | CPU máxima (throttling se exceder) | Pod é throttled |
| **limits.memory** | Memória máxima (OOMKilled se exceder) | Pod é terminado |

### 1.2 Configuração Correta
```yaml
# ✅ BOM — requests e limits configurados
spec:
  containers:
    - name: api
      image: myapp:v1.2.0
      resources:
        requests:
          cpu: 250m       # 0.25 vCPU
          memory: 256Mi   # 256 MiB
        limits:
          cpu: 1000m      # 1 vCPU (4x request)
          memory: 512Mi   # 512 MiB (2x request)
```

### 1.3 Guidelines de Ratio
```
# Recomendações de ratio request:limit
CPU:    1:2 até 1:4  (throttling é soft)
Memory: 1:1 até 1:2  (OOMKill é hard — seja conservador)

# Workloads burstable (APIs web)
CPU requests: P50 usage
CPU limits:   P99 usage * 1.5

# Workloads steady (workers, batch)
CPU requests: ≈ CPU limits (Guaranteed QoS)
Memory requests: = Memory limits (sempre para workers)
```

### 1.4 QoS Classes
| QoS Class | Condição | Prioridade de Eviction |
|-----------|----------|:---:|
| **Guaranteed** | requests = limits (CPU e memory) | Última (mais protegido) |
| **Burstable** | requests < limits | Intermediária |
| **BestEffort** | Sem requests/limits | Primeira (menos protegido) |

```yaml
# Guaranteed QoS — para workloads críticos
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 500m      # iguais
    memory: 512Mi  # iguais

# Burstable QoS — para APIs web
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m     # diferentes
    memory: 512Mi  # diferentes
```

**Regras:**
- **SEMPRE** defina requests e limits para TODOS os containers.
- Use **Guaranteed QoS** para workloads críticos (databases, message queues).
- Use **Burstable QoS** para APIs e serviços web.
- **Nunca** use BestEffort em produção.
- Memory limits deve ser >= memory requests (OOMKill é fatal).

## 2. LimitRange

```yaml
# LimitRange — defaults e limites por namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: 4
        memory: 8Gi
      min:
        cpu: 50m
        memory: 64Mi
    - type: Pod
      max:
        cpu: 8
        memory: 16Gi
    - type: PersistentVolumeClaim
      max:
        storage: 100Gi
      min:
        storage: 1Gi
```

## 3. ResourceQuota

```yaml
# ResourceQuota — limites por namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    # Compute
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    
    # Object counts
    pods: "100"
    services: "20"
    services.loadbalancers: "2"
    persistentvolumeclaims: "20"
    secrets: "50"
    configmaps: "50"
    
    # Storage
    requests.storage: 500Gi
```

## 4. Horizontal Pod Autoscaler (HPA)

### 4.1 HPA Básico (CPU/Memory)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 20
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # 5min para evitar flapping
      policies:
        - type: Percent
          value: 25
          periodSeconds: 120
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### 4.2 HPA com Métricas Custom
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa-custom
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 50
  metrics:
    # Métrica custom do Prometheus
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 1000
    # Métrica externa (ex: fila SQS)
    - type: External
      external:
        metric:
          name: sqs_queue_length
          selector:
            matchLabels:
              queue: orders
        target:
          type: AverageValue
          averageValue: 10
```

### 4.3 Boas Práticas de HPA
- Configure **`stabilizationWindowSeconds`** para evitar flapping.
- ScaleDown deve ser **mais lento** que scaleUp.
- Use **métricas de negócio** (requests/s, queue length) quando possível.
- CPU requests deve refletir uso real para HPA funcionar corretamente.
- **Não use HPA com VPA** no modo Auto para o mesmo recurso (CPU/memory).
- **minReplicas >= 2** para alta disponibilidade.

## 5. Vertical Pod Autoscaler (VPA)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Off"  # ✅ Comece com "Off" para apenas recomendar
  resourcePolicy:
    containerPolicies:
      - containerName: api
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 4
          memory: 8Gi
        controlledResources: ["cpu", "memory"]
```

**Modos do VPA:**
| Modo | Descrição | Recomendação |
|------|-----------|:---:|
| **Off** | Apenas recomendações | ✅ Sempre comece aqui |
| **Initial** | Define no schedule, não atualiza | Para workloads estáveis |
| **Auto** | Atualiza requests (pode restartar Pods) | Com cuidado |

```bash
# Verificar recomendações do VPA
kubectl describe vpa api-vpa
```

## 6. KEDA (Event-Driven Autoscaling)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 1
  maxReplicaCount: 100
  cooldownPeriod: 300
  pollingInterval: 15
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789/orders
        queueLength: "5"
        awsRegion: us-east-1
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: http_requests_total
        query: sum(rate(http_requests_total{service="order-processor"}[2m]))
        threshold: "100"
```

## 7. Pod Priority e Preemption

```yaml
# PriorityClass
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Para workloads críticos de produção"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high
value: 100000
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low
value: 1000
---
# Uso
spec:
  priorityClassName: critical
  containers:
    - name: api
      image: myapp:v1.2.0
```

**Hierarquia sugerida:**
```
system-cluster-critical: 2000000000 (sistema)
system-node-critical:    2000001000 (sistema)
critical:                1000000    (produção)
high:                    100000     (staging)
default:                 0          (padrão)
low:                     1000       (batch, dev)
```

## 8. Node Affinity e Taints

### 8.1 Node Affinity
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-type
                operator: In
                values: ["compute-optimized"]
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["us-east-1a"]
```

### 8.2 Taints e Tolerations
```bash
# Taint para nodes especiais
kubectl taint nodes gpu-node-1 gpu=true:NoSchedule
```

```yaml
# Toleration no Pod
spec:
  tolerations:
    - key: gpu
      operator: Equal
      value: "true"
      effect: NoSchedule
  nodeSelector:
    gpu: "true"
```

## 9. Rightsizing — Processo

```
1. Deploy com estimates iniciais (baseados em load testing)
2. Ativar VPA em modo "Off" (recomendações)
3. Monitorar uso real por 1-2 semanas
4. Ajustar requests/limits baseado em:
   - CPU: P95 usage → request, P99 * 1.5 → limit
   - Memory: P99 usage * 1.2 → request = limit
5. Revisar mensalmente
6. Usar ferramentas: Kubecost, OpenCost, Goldilocks
```

### Ferramentas de Rightsizing
| Ferramenta | Descrição |
|-----------|-----------|
| **Goldilocks** | Dashboard VPA recommendations |
| **Kubecost** | Cost monitoring + recommendations |
| **OpenCost** | Open source cost monitoring |
| **Robusta KRR** | CLI para recomendações baseadas em Prometheus |
