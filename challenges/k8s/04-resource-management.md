# Level 4 — Resource Management

> **Objetivo:** Gerenciar recursos computacionais da plataforma CloudShop com requests/limits,
> QoS classes, HPA, VPA, LimitRange e ResourceQuota para garantir estabilidade e eficiência.

**Referência:** [Resource Management Best Practices](../../.docs/k8s/04-resource-management.md)

---

## Objetivo de Aprendizado

- Definir requests e limits para CPU e memória
- Compreender QoS classes (Guaranteed, Burstable, BestEffort)
- Configurar LimitRange e ResourceQuota por namespace
- Implementar HPA com métricas de CPU, memória e custom metrics
- Configurar VPA para right-sizing automático
- Monitorar utilização real vs alocada

---

## Contexto do Domínio

A plataforma CloudShop roda em cluster compartilhado entre dev, staging e produção. Cada namespace deve ter quotas bem definidas. Os microsserviços devem Auto-escalar baseado em carga real e os banco de dados devem ter recursos garantidos (QoS Guaranteed).

---

## Desafios

### Desafio 4.1 — Requests e Limits

Defina requests e limits para todos os workloads da CloudShop.

**Requisitos:**
- Microsserviços (Deployment): Burstable QoS
- Databases (StatefulSet): Guaranteed QoS (request == limit)
- FluentBit (DaemonSet): Burstable QoS com limits baixos

```yaml
# Burstable — order-service
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
---
# Guaranteed — postgresql
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 500m    # == request
    memory: 1Gi  # == request
---
# DaemonSet — fluentbit (low overhead)
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
```

**Critérios de aceite:**
- [ ] Todos os Pods têm requests e limits definidos
- [ ] QoS class verificado: `kubectl get pod <pod> -o jsonpath='{.status.qosClass}'`
- [ ] PostgreSQL/Redis → Guaranteed
- [ ] Microsserviços → Burstable
- [ ] Nenhum Pod com BestEffort

---

### Desafio 4.2 — LimitRange

Configure LimitRange para garantir defaults e limites nos namespaces.

**Requisitos:**
- Default request/limit para containers sem definição
- Min/Max por container
- Max por Pod

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: cloudshop-prod
spec:
  limits:
    - type: Container
      default:           # limits padrão
        cpu: 250m
        memory: 256Mi
      defaultRequest:    # requests padrão
        cpu: 50m
        memory: 64Mi
      min:
        cpu: 10m
        memory: 16Mi
      max:
        cpu: "2"
        memory: 2Gi
    - type: Pod
      max:
        cpu: "4"
        memory: 4Gi
```

**Critérios de aceite:**
- [ ] LimitRange aplicado em cloudshop-dev, cloudshop-staging, cloudshop-prod
- [ ] Pod sem resources definidos recebe defaults
- [ ] Pod que excede max é rejeitado
- [ ] Pod que fica abaixo de min é rejeitado
- [ ] `kubectl describe limitrange -n cloudshop-prod`

---

### Desafio 4.3 — ResourceQuota

Limite recursos totais por namespace para evitar consumo excessivo.

**Requisitos:**
- Quotas diferentes para dev, staging e prod
- Limitar total de CPU, memória, pods e PVCs
- Quota para objetos (configmaps, secrets, services)

```yaml
# Quota para produção (mais recursos)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: cloudshop-prod
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "50"
    persistentvolumeclaims: "20"
---
# Quota para dev (recursos limitados)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: cloudshop-dev
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "20"
    persistentvolumeclaims: "5"
---
# Object count quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota
  namespace: cloudshop-prod
spec:
  hard:
    configmaps: "30"
    secrets: "30"
    services: "20"
    services.loadbalancers: "2"
    services.nodeports: "0"  # proibir NodePort em prod
```

**Critérios de aceite:**
- [ ] ResourceQuota em todos os namespaces CloudShop
- [ ] prod tem mais recursos que dev
- [ ] `kubectl describe resourcequota -n cloudshop-prod` mostra used vs hard
- [ ] Deploy que excede quota é rejeitado
- [ ] NodePort proibido em produção

---

### Desafio 4.4 — HPA (Horizontal Pod Autoscaler)

Configure HPA para os microsserviços escalarem com base na carga.

**Requisitos:**
- HPA com CPU para todos os microsserviços
- HPA com custom metrics (requests per second) para order-service
- Behavior para scale-up rápido e scale-down gradual

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service
  namespace: cloudshop-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 10
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
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100          # dobrar replicas
          periodSeconds: 60
        - type: Pods
          value: 4
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300   # esperar 5min antes de reduzir
      policies:
        - type: Percent
          value: 25           # reduzir no máximo 25%
          periodSeconds: 60
```

```bash
# Instalar metrics-server (se não instalado)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verificar HPA
kubectl get hpa -n cloudshop-prod
kubectl describe hpa order-service -n cloudshop-prod

# Simular carga
kubectl run load-gen --rm -it --image=busybox -- sh -c \
  "while true; do wget -q -O- http://order-service.cloudshop-prod/health; done"
```

**Critérios de aceite:**
- [ ] metrics-server instalado e funcional
- [ ] HPA criado para todos os microsserviços
- [ ] Scale-up funciona sob carga (CPU > 70%)
- [ ] Scale-down gradual (25% a cada 60s, após 5min estável)
- [ ] `kubectl top pods -n cloudshop-prod` mostra métricas reais
- [ ] Behavior customizado validado

---

### Desafio 4.5 — VPA (Vertical Pod Autoscaler)

Use VPA para right-sizing automático dos Pods — descobrir resources ideais.

**Requisitos:**
- Instalar VPA
- Modo `Off` (apenas recomendação) nos microsserviços
- Modo `Auto` no namespace de dev (para experimentação)
- Analisar recomendações e ajustar requests/limits

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service
  namespace: cloudshop-prod
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  updatePolicy:
    updateMode: "Off"  # apenas recomendações
  resourcePolicy:
    containerPolicies:
      - containerName: order-service
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: "2"
          memory: 2Gi
        controlledResources:
          - cpu
          - memory
```

```bash
# Verificar recomendações VPA
kubectl describe vpa order-service -n cloudshop-prod

# Output esperado:
# Recommendation:
#   Container Recommendations:
#     Container Name: order-service
#     Lower Bound:  Cpu: 50m,  Memory: 100Mi
#     Target:       Cpu: 150m, Memory: 200Mi
#     Upper Bound:  Cpu: 400m, Memory: 600Mi
```

**Critérios de aceite:**
- [ ] VPA instalado no cluster
- [ ] VPA `Off` criado para cada microsserviço em prod
- [ ] VPA `Auto` criado em dev (para validação)
- [ ] Recomendações coletadas após carga
- [ ] Requests/limits ajustados baseado nas recomendações
- [ ] Documentado: VPA vs HPA — quando usar cada um

---

### Desafio 4.6 — Monitoramento de Recursos

Monitore utilização real vs alocada para identificar waste e right-size.

**Requisitos:**
- `kubectl top` para pods e nodes
- Calcular request vs utilização real
- Identificar pods over-provisioned e under-provisioned

```bash
# Utilização por node
kubectl top nodes

# Utilização por pod
kubectl top pods -n cloudshop-prod --sort-by=cpu
kubectl top pods -n cloudshop-prod --sort-by=memory

# Requests vs Limits vs Utilização
kubectl get pods -n cloudshop-prod -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].resources.requests.cpu}{"\t"}{.spec.containers[0].resources.limits.cpu}{"\n"}{end}'

# Pods sem resources (BestEffort check)
kubectl get pods --all-namespaces -o json | jq '.items[] | select(.status.qosClass == "BestEffort") | .metadata.name'

# Node allocatable vs alocado
kubectl describe node <node-name> | grep -A 10 "Allocated resources"
```

**Critérios de aceite:**
- [ ] `kubectl top nodes` funciona (metrics-server)
- [ ] `kubectl top pods` para todos os namespaces
- [ ] Over-provisioned pods identificados
- [ ] Under-provisioned pods identificados
- [ ] Relatório: utilização real vs alocada (tabela)

---

## Definição de Pronto (DoD)

- [ ] Requests/limits definidos para todos os workloads
- [ ] LimitRange em todos os namespaces
- [ ] ResourceQuota restringindo consumo por namespace
- [ ] HPA escalando microsserviços automaticamente
- [ ] VPA gerando recomendações de right-sizing
- [ ] Monitoramento de utilização implementado
- [ ] Commit: `feat(level-4): add resource management`

---

## Checklist

- [ ] QoS Guaranteed para databases
- [ ] QoS Burstable para microsserviços
- [ ] LimitRange com defaults e min/max
- [ ] ResourceQuota diferenciada por ambiente
- [ ] metrics-server instalado
- [ ] HPA com CPU + behavior customizado
- [ ] VPA em modo recomendação
- [ ] `kubectl top` funcional
- [ ] Nenhum Pod BestEffort
