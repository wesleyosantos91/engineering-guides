# Level 17 — Capstone: CloudShop Production Platform

> **Objetivo:** Construir a plataforma CloudShop **do zero ao production-grade** em um cluster Kubernetes,
> integrando **todos** os conceitos dos levels 0–16. Este é o projeto final que demonstra maestria
> completa em Kubernetes — da criação do cluster à operação em produção.

**Pré-requisito:** Todos os levels anteriores (0–16) completos.

---

## Contexto

Você foi contratado como **Staff Platform Engineer** da CloudShop. A empresa decidiu migrar
de VMs para Kubernetes e você é responsável por construir a plataforma inteira.

O Capstone simula **4 fases reais** de uma migração para Kubernetes:
1. **Semana 1:** Platform Bootstrap — cluster, namespaces, RBAC, políticas
2. **Semana 2:** Workloads — deploy de todos os microsserviços e infraestrutura stateful
3. **Semana 3:** Production Hardening — segurança, observabilidade, GitOps
4. **Semana 4:** Production Day — go-live, chaos engineering, DR drill

**Escala alvo:**
- **6 microsserviços** + **3 StatefulSets** (PostgreSQL, Redis, Kafka)
- **3 ambientes** (dev, staging, prod) isolados por namespace
- **Multi-AZ** com topology spread constraints
- **Zero-downtime deploys** com progressive delivery
- **DR testado** com RPO ≤ 1h, RTO ≤ 15min
- **SLOs:** 99.9% availability, p99 < 500ms para APIs

---

## Arquitetura Alvo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Internet                                       │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────────┐
│  ingress-nginx namespace                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  Ingress Controller (nginx) + cert-manager (TLS auto-renewal)       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
     ┌───────────────────────────┼───────────────────────────────┐
     │                cloudshop-prod namespace                    │
     │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │
     │  │  order   │ │ product  │ │   user   │ │ payment  │     │
     │  │  svc     │ │  svc     │ │   svc    │ │  svc     │     │
     │  │ HPA:2-10 │ │ HPA:2-10 │ │ HPA:2-8  │ │ HPA:2-8  │     │
     │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘     │
     │       │             │             │             │           │
     │  ┌────┴─────┐ ┌────┴─────┐                                │
     │  │inventory │ │ notific. │  ← async via Kafka              │
     │  │  svc     │ │  svc     │                                 │
     │  │ HPA:2-6  │ │ HPA:1-4  │                                 │
     │  └──────────┘ └──────────┘                                 │
     │                                                            │
     │  Jobs: report-generator (CronJob), data-migration (Job)    │
     └────────────────────────────────────────────────────────────┘
                                 │
     ┌───────────────────────────┼───────────────────────────────┐
     │                cloudshop-data namespace                    │
     │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐      │
     │  │  PostgreSQL  │ │    Redis     │ │    Kafka     │      │
     │  │ StatefulSet  │ │ StatefulSet  │ │ StatefulSet  │      │
     │  │  3 replicas  │ │  3 replicas  │ │  3 brokers   │      │
     │  │  PVC 20Gi    │ │  PVC 5Gi     │ │  PVC 50Gi    │      │
     │  └──────────────┘ └──────────────┘ └──────────────┘      │
     └──────────────────────────────────────────────────────────┘
                                 │
     ┌───────────────────────────┼───────────────────────────────┐
     │         monitoring / logging / argocd namespaces           │
     │  ┌───────────┐ ┌────────┐ ┌──────┐ ┌─────────┐           │
     │  │Prometheus │ │Grafana │ │Tempo │ │FluentBit│           │
     │  │+ AlertMgr │ │        │ │      │ │DaemonSet│           │
     │  └───────────┘ └────────┘ └──────┘ └─────────┘           │
     │  ┌──────┐ ┌────────────┐ ┌──────────────┐                │
     │  │ Loki │ │  ArgoCD    │ │ External     │                │
     │  │      │ │+ Image Upd │ │ Secrets Op   │                │
     │  └──────┘ └────────────┘ └──────────────┘                │
     │  ┌──────┐ ┌──────────┐                                    │
     │  │Velero│ │ Kyverno  │                                    │
     │  └──────┘ └──────────┘                                    │
     └──────────────────────────────────────────────────────────┘
```

---

## Desafios

### Desafio 17.1 — Platform Bootstrap: Cluster e Fundamentos

Crie o cluster e toda a fundação da plataforma.

**Requisitos:**

**1. Cluster creation (kind/minikube para local, EKS/GKE para cloud):**

```yaml
# kind-cloudshop.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cloudshop
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "topology.kubernetes.io/zone=us-east-1a"
  - role: worker
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "topology.kubernetes.io/zone=us-east-1a"
  - role: worker
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "topology.kubernetes.io/zone=us-east-1b"
  - role: worker
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "topology.kubernetes.io/zone=us-east-1c"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/16"
```

**2. Namespace strategy:**

```bash
# Criar namespaces com labels
for ns in cloudshop-dev cloudshop-staging cloudshop-prod cloudshop-data \
          monitoring logging argocd ingress-nginx cert-manager external-secrets; do
  kubectl create namespace $ns
done

# Pod Security Standards
kubectl label namespace cloudshop-prod \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

kubectl label namespace cloudshop-staging \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted

kubectl label namespace cloudshop-dev \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=baseline
```

**3. RBAC por time:**

```yaml
# rbac/dev-team-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: cloudshop-dev
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["pods", "deployments", "services", "configmaps", "jobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec", "pods/portforward"]
    verbs: ["get", "create"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]  # read-only secrets
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: cloudshop-prod
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "deployments", "services"]
    verbs: ["get", "list", "watch"]  # read-only em prod
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
  # NO exec, NO delete, NO create in prod
```

**4. Resource quotas e LimitRanges:**

```yaml
# quotas/prod-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: cloudshop-prod
spec:
  hard:
    requests.cpu: "16"
    requests.memory: 32Gi
    limits.cpu: "32"
    limits.memory: 64Gi
    pods: "100"
    services: "20"
    persistentvolumeclaims: "20"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: cloudshop-prod
spec:
  limits:
    - default:
        cpu: "500m"
        memory: 512Mi
      defaultRequest:
        cpu: "100m"
        memory: 128Mi
      type: Container
```

**5. Network Policies (default deny + allow):**

```yaml
# netpol/default-deny-prod.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: cloudshop-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# netpol/allow-order-to-db.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-order-to-postgres
  namespace: cloudshop-data
spec:
  podSelector:
    matchLabels:
      app: postgresql
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: cloudshop-prod
          podSelector:
            matchLabels:
              app: order-service
      ports:
        - port: 5432
          protocol: TCP
```

**Critérios de aceite:**
- [ ] Cluster com 1 control plane + 3 workers (multi-AZ labels)
- [ ] 10 namespaces criados e organizados
- [ ] PSS enforce=restricted em cloudshop-prod
- [ ] RBAC: developer role com permissões diferenciadas dev vs prod
- [ ] ResourceQuota e LimitRange em todos os cloudshop-* namespaces
- [ ] NetworkPolicy default-deny + allow-list em cloudshop-prod e cloudshop-data
- [ ] `kubectl auth can-i` validando permissões RBAC

---

### Desafio 17.2 — Stateful Infrastructure: Databases e Messaging

Deploy dos componentes stateful com alta disponibilidade.

**Requisitos:**

**1. PostgreSQL StatefulSet:**

```yaml
# stateful/postgresql.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: cloudshop-data
spec:
  serviceName: postgresql
  replicas: 3
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        fsGroup: 999
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: postgresql
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: cloudshop
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgresql-credentials
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgresql-credentials
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: "2"
              memory: 2Gi
          readinessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - $(POSTGRES_USER)
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - $(POSTGRES_USER)
            initialDelaySeconds: 30
            periodSeconds: 30
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false  # PostgreSQL needs writable
            capabilities:
              drop: ["ALL"]
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
```

**2. Redis StatefulSet com Sentinel ou Cluster mode**

**3. Kafka StatefulSet (KRaft mode, 3 brokers)**

**4. PDBs para todos os StatefulSets:**

```yaml
# pdb/postgresql-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgresql-pdb
  namespace: cloudshop-data
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: postgresql
```

**5. Headless Services para StatefulSets:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  namespace: cloudshop-data
spec:
  clusterIP: None
  selector:
    app: postgresql
  ports:
    - port: 5432
```

**6. VolumeSnapshots para backup:**

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgresql-snapshot
  namespace: cloudshop-data
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: data-postgresql-0
```

**Critérios de aceite:**
- [ ] PostgreSQL: 3 replicas StatefulSet, PVC 20Gi cada, probes configurados
- [ ] Redis: 3 replicas StatefulSet, PVC 5Gi, modo Sentinel ou Cluster
- [ ] Kafka: 3 brokers StatefulSet (KRaft), PVC 50Gi, topics pré-criados
- [ ] PDBs: minAvailable=2 para PostgreSQL e Kafka, minAvailable=1 para Redis
- [ ] Headless Services para todos os StatefulSets
- [ ] VolumeSnapshots funcionais para PostgreSQL
- [ ] Secrets gerenciados via External Secrets Operator (ou Sealed Secrets)
- [ ] Data migration Job rodando Flyway/migrate com sucesso

---

### Desafio 17.3 — Application Workloads: Deploy dos Microsserviços

Deploy de todos os microsserviços com configuração production-grade.

**Requisitos:**

**1. Deployment template (aplicar para os 6 microsserviços):**

```yaml
# workloads/order-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: cloudshop-prod
  labels:
    app: order-service
    version: v1.0.0
    team: checkout
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: order-service
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: order-service
      containers:
        - name: order-service
          image: cloudshop/order-service:v1.0.0
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: order-service-db
                  key: url
            - name: KAFKA_BROKERS
              value: "kafka-0.kafka.cloudshop-data:9092,kafka-1.kafka.cloudshop-data:9092,kafka-2.kafka.cloudshop-data:9092"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector.monitoring:4317"
            - name: OTEL_SERVICE_NAME
              value: "order-service"
          envFrom:
            - configMapRef:
                name: order-service-config
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
          startupProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 30
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      terminationGracePeriodSeconds: 60
```

**2. HPA para todos os microsserviços:**

```yaml
# hpa/order-service-hpa.yaml
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
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

**3. PDBs para microsserviços:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: cloudshop-prod
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: order-service
```

**4. CronJob para relatórios:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: report-generator
  namespace: cloudshop-prod
spec:
  schedule: "0 2 * * *"  # Diário às 2h
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: report-generator
              image: cloudshop/report-generator:v1.0.0
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: report-db
                      key: url
          restartPolicy: OnFailure
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
```

**5. Services + Ingress:**

```yaml
# ingress/cloudshop-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cloudshop
  namespace: cloudshop-prod
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - cloudshop.local
      secretName: cloudshop-tls
  rules:
    - host: cloudshop.local
      http:
        paths:
          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8080
          - path: /api/products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 8080
          - path: /api/users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8080
          - path: /api/payments
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 8080
```

**Critérios de aceite:**
- [ ] 6 Deployments em cloudshop-prod (order, product, user, payment, inventory, notification)
- [ ] Todos com: 3 probes (startup, readiness, liveness), securityContext, resource requests/limits
- [ ] Topology spread across zones (maxSkew=1)
- [ ] Graceful shutdown: preStop sleep + terminationGracePeriodSeconds
- [ ] HPA configurado (CPU 70%, Memory 80%, min=2, max=10)
- [ ] PDBs para todos os Deployments (minAvailable=1)
- [ ] CronJob: report-generator rodando diariamente
- [ ] Job: data-migration executado com sucesso
- [ ] Ingress com TLS (cert-manager) + rate limiting
- [ ] Services ClusterIP para cada microsserviço
- [ ] readOnlyRootFilesystem=true com emptyDir para /tmp

---

### Desafio 17.4 — Observability Stack Completo

Deploy e configuração do stack de observabilidade completo.

**Requisitos:**

**1. Prometheus + AlertManager:**

```bash
# Instalar via Helm (kube-prometheus-stack)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set alertmanager.config.route.receiver=slack \
  -f values/prometheus-values.yaml
```

**2. ServiceMonitors para microsserviços:**

```yaml
# monitoring/servicemonitor-order.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-service
  namespace: monitoring
  labels:
    release: kube-prometheus
spec:
  namespaceSelector:
    matchNames:
      - cloudshop-prod
  selector:
    matchLabels:
      app: order-service
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

**3. PrometheusRules (alertas):**

```yaml
# monitoring/alerts-cloudshop.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cloudshop-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus
spec:
  groups:
    - name: cloudshop.availability
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_server_requests_seconds_count{namespace="cloudshop-prod",status=~"5.."}[5m]))
            / sum(rate(http_server_requests_seconds_count{namespace="cloudshop-prod"}[5m]))
            > 0.001
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Error rate > 0.1% em cloudshop-prod"
            runbook: "https://wiki.cloudshop.io/runbooks/high-error-rate"

        - alert: HighLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_server_requests_seconds_bucket{namespace="cloudshop-prod"}[5m])) by (le, service)
            ) > 0.5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "p99 latency > 500ms para {{ $labels.service }}"

    - name: cloudshop.resources
      rules:
        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total{namespace=~"cloudshop-.*"}[15m]) > 0
          for: 5m
          labels:
            severity: critical

        - alert: PodNotReady
          expr: kube_pod_status_ready{namespace=~"cloudshop-.*", condition="true"} == 0
          for: 10m
          labels:
            severity: warning

        - alert: PVCAlmostFull
          expr: |
            kubelet_volume_stats_used_bytes{namespace="cloudshop-data"}
            / kubelet_volume_stats_capacity_bytes{namespace="cloudshop-data"}
            > 0.8
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} > 80% used"

    - name: cloudshop.kafka
      rules:
        - alert: KafkaConsumerLag
          expr: kafka_consumer_group_lag > 10000
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Consumer lag > 10K messages"
```

**4. Grafana Dashboards (3 obrigatórios):**

| Dashboard | Panels | Fonte |
|-----------|--------|-------|
| **CloudShop RED** | Request Rate, Error Rate %, p50/p95/p99 por serviço | Prometheus |
| **CloudShop USE** | CPU, Memory, DB Connections, Kafka Lag por pod | Prometheus |
| **CloudShop SLOs** | Availability SLO (99.9%), Error Budget remaining, Burn rate | Prometheus |

**5. Logging: FluentBit → Loki:**

```yaml
# logging/fluentbit-values.yaml (Helm)
config:
  inputs: |
    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/cloudshop-*.log
        Parser            cri
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10

  filters: |
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Merge_Log           On
        K8S-Logging.Parser  On

  outputs: |
    [OUTPUT]
        Name          loki
        Match         kube.*
        Host          loki.logging.svc
        Port          3100
        Labels        job=fluentbit, namespace=$kubernetes['namespace_name'], pod=$kubernetes['pod_name']
```

**6. Tracing: OpenTelemetry Collector → Tempo:**

```yaml
# monitoring/otel-collector.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      batch:
        timeout: 5s
        send_batch_size: 1000
    exporters:
      otlp/tempo:
        endpoint: tempo.monitoring:4317
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlp/tempo]
```

**Critérios de aceite:**
- [ ] Prometheus: scraping métricas de 6 microsserviços + 3 StatefulSets
- [ ] ServiceMonitors para todos os workloads
- [ ] PrometheusRules: ≥ 5 alertas (error rate, latency, pod crash, PVC, Kafka lag)
- [ ] AlertManager: configurado para enviar notificações (Slack/email)
- [ ] Grafana: 3 dashboards (RED, USE, SLOs) importados e funcionais
- [ ] FluentBit DaemonSet coletando logs de todos os nodes
- [ ] Loki armazenando logs pesquisáveis
- [ ] OpenTelemetry Collector → Tempo para traces
- [ ] Trace end-to-end visível: order-service → payment-service → Kafka → notification-service
- [ ] SLO dashboard: 99.9% availability com error budget

---

### Desafio 17.5 — GitOps com ArgoCD

Implemente GitOps completo com ArgoCD.

**Requisitos:**

**1. Repositório GitOps estruturado com Kustomize:**

```
gitops/
├── base/
│   ├── order-service/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   ├── pdb.yaml
│   │   └── kustomization.yaml
│   ├── product-service/
│   │   └── ...
│   ├── user-service/
│   │   └── ...
│   ├── payment-service/
│   │   └── ...
│   ├── inventory-service/
│   │   └── ...
│   └── notification-service/
│       └── ...
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml      ← replicas=1, resources reduzidos
│   │   ├── namespace.yaml
│   │   └── patches/
│   │       └── replicas.yaml
│   ├── staging/
│   │   ├── kustomization.yaml      ← replicas=2, quase-prod
│   │   └── patches/
│   │       └── replicas.yaml
│   └── prod/
│       ├── kustomization.yaml      ← replicas=3, full resources
│       └── patches/
│           ├── replicas.yaml
│           ├── resources.yaml
│           └── hpa.yaml
└── infrastructure/
    ├── prometheus/
    ├── grafana/
    ├── fluentbit/
    ├── loki/
    ├── tempo/
    ├── argocd/
    ├── cert-manager/
    ├── ingress-nginx/
    ├── external-secrets/
    ├── velero/
    └── kyverno/
```

**2. ArgoCD ApplicationSet (matrix generator):**

```yaml
# argocd/applicationset-cloudshop.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cloudshop-services
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          - list:
              elements:
                - service: order-service
                - service: product-service
                - service: user-service
                - service: payment-service
                - service: inventory-service
                - service: notification-service
          - list:
              elements:
                - env: dev
                  namespace: cloudshop-dev
                - env: staging
                  namespace: cloudshop-staging
                - env: prod
                  namespace: cloudshop-prod
  template:
    metadata:
      name: "{{service}}-{{env}}"
    spec:
      project: cloudshop
      source:
        repoURL: https://github.com/cloudshop/gitops.git
        targetRevision: main
        path: "overlays/{{env}}"
        kustomize:
          namePrefix: ""
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=false
          - ApplyOutOfSyncOnly=true
```

**3. Promotion pipeline (dev → staging → prod):**

```bash
# === PROMOTION PIPELINE ===

# 1. Image built and pushed to registry
# CI: build, test, push cloudshop/order-service:v1.1.0

# 2. Dev: Image Updater auto-updates
# ArgoCD Image Updater detects new tag → updates dev overlay
# Auto-sync deploys to cloudshop-dev

# 3. Staging: Manual PR
# Create PR: update staging overlay with v1.1.0
git checkout -b promote/order-service-v1.1.0-staging
cd gitops/overlays/staging
kustomize edit set image cloudshop/order-service=cloudshop/order-service:v1.1.0
git commit -am "promote: order-service v1.1.0 → staging"
git push && gh pr create

# 4. Staging: Smoke test after merge & sync
kubectl wait --for=condition=available deployment/order-service -n cloudshop-staging --timeout=120s
./smoke-test.sh https://staging.cloudshop.local

# 5. Prod: Approval-gated PR
# Same process, but requires 2 approvals
```

**Critérios de aceite:**
- [ ] ArgoCD instalado com ApplicationSet matrix generator
- [ ] 18 Applications gerenciadas (6 serviços × 3 ambientes)
- [ ] Todas Applications: Synced + Healthy
- [ ] Kustomize: base + 3 overlays com diferenças claras (replicas, resources, etc.)
- [ ] Image Updater funcional para dev (auto-update)
- [ ] Promotion pipeline documentado e testado (dev → staging → prod)
- [ ] Helm charts para componentes de infraestrutura
- [ ] Git como single source of truth (nenhum kubectl apply manual em prod)

---

### Desafio 17.6 — Security Hardening e Compliance

Aplique segurança defense-in-depth na plataforma.

**Requisitos:**

**1. Kyverno policies:**

```yaml
# kyverno/require-labels.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-app-label
      match:
        any:
          - resources:
              kinds:
                - Deployment
                - StatefulSet
              namespaces:
                - "cloudshop-*"
      validate:
        message: "Label 'app' é obrigatório"
        pattern:
          metadata:
            labels:
              app: "?*"
---
# kyverno/disallow-latest.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
    - name: disallow-latest
      match:
        any:
          - resources:
              kinds: ["Pod"]
              namespaces: ["cloudshop-prod"]
      validate:
        message: "Tag ':latest' não é permitida em produção"
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

**2. External Secrets Operator:**

```yaml
# secrets/external-secret-order.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service-db
  namespace: cloudshop-prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager  # ou vault, GCP, etc.
    kind: ClusterSecretStore
  target:
    name: order-service-db
  data:
    - secretKey: url
      remoteRef:
        key: cloudshop/prod/order-service/database-url
```

**3. Image scanning + Supply chain:**

```bash
# Scan todas as imagens em uso na plataforma
for deploy in order product user payment inventory notification; do
  image=$(kubectl get deploy ${deploy}-service -n cloudshop-prod \
    -o jsonpath='{.spec.template.spec.containers[0].image}')
  echo "=== Scanning $image ==="
  trivy image --severity HIGH,CRITICAL "$image"
done
```

**4. Audit logging habilitado no API server**

**5. Falco para runtime security monitoring**

**Critérios de aceite:**
- [ ] Kyverno: ≥ 5 policies enforce (labels, latest tag, limits, non-root, read-only FS)
- [ ] External Secrets Operator sincronizando secrets de fonte externa
- [ ] Zero CRITICAL CVEs em imagens (Trivy scan)
- [ ] Audit logging configurado no kube-apiserver
- [ ] Falco monitorando runtime com regras para cloudshop namespaces
- [ ] ServiceAccounts dedicados por microsserviço (não usar default)
- [ ] Secrets rotacionados automaticamente (refresh interval)
- [ ] Supply chain: imagens assinadas (Cosign) — opcional

---

### Desafio 17.7 — Resilience Testing e Chaos Engineering

Valide que a plataforma sobrevive a falhas reais.

**Requisitos:**

**Cenário 1: Node failure**

```bash
# 1. Identificar node com mais pods de produção
kubectl get pods -n cloudshop-prod -o wide --sort-by='{.spec.nodeName}'

# 2. Cordon + drain node
kubectl cordon worker-2
kubectl drain worker-2 --ignore-daemonsets --delete-emptydir-data --timeout=120s

# 3. Verificar redistribuição
kubectl get pods -n cloudshop-prod -o wide
# Pods devem ter migrado para worker-1 e worker-3
# PDBs devem ter prevenido indisponibilidade

# 4. Verificar que APIs continuam respondendo
curl -s https://cloudshop.local/api/orders | jq .
# Deve retornar 200

# 5. Uncordon e verificar rebalancing
kubectl uncordon worker-2
# Pods podem ou não migrar de volta (depende de resource pressure)
```

**Cenário 2: StatefulSet pod failure**

```bash
# 1. Deletar pod do PostgreSQL primary
kubectl delete pod postgresql-0 -n cloudshop-data

# 2. Verificar recovery
kubectl get pods -n cloudshop-data -w
# Pod deve ser recriado pelo StatefulSet controller
# PVC deve ser reattachado (dados preservados)

# 3. Verificar integridade dos dados
kubectl exec -n cloudshop-data postgresql-0 -- \
  psql -U cloudshop -c "SELECT count(*) FROM orders;"
```

**Cenário 3: Network partition (NetworkPolicy test)**

```bash
# 1. Aplicar NetworkPolicy que bloqueia order-service → PostgreSQL
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-order-to-db
  namespace: cloudshop-data
spec:
  podSelector:
    matchLabels:
      app: postgresql
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: cloudshop-prod
          podSelector:
            matchLabels:
              app: product-service  # only product, NOT order
      ports:
        - port: 5432
EOF

# 2. Testar: order-service deve falhar com timeout
curl -s https://cloudshop.local/api/orders
# Esperado: 503 ou timeout

# 3. Restaurar acesso
kubectl delete networkpolicy block-order-to-db -n cloudshop-data
```

**Cenário 4: HPA scale-up under load**

```bash
# 1. Gerar carga com k6
k6 run --vus 200 --duration 5m k6/scenarios/product-search.js

# 2. Observar HPA scaling
kubectl get hpa -n cloudshop-prod -w
# product-service deve escalar de 2 → 10 pods

# 3. Após carga parar: observar scale-down
# stabilizationWindowSeconds: 300 (5 min)
kubectl get hpa -n cloudshop-prod -w
# Deve desescalar gradualmente após cooldown
```

**Cenário 5: Disaster Recovery drill**

```bash
# 1. Backup com Velero
velero backup create cloudshop-pre-dr --include-namespaces cloudshop-prod,cloudshop-data

# 2. Simular desastre: deletar namespace prod
kubectl delete namespace cloudshop-prod

# 3. Restore
velero restore create cloudshop-dr --from-backup cloudshop-pre-dr

# 4. Validar
kubectl get pods -n cloudshop-prod
curl -s https://cloudshop.local/api/products | head

# 5. Medir RPO e RTO
echo "RPO: $(velero backup describe cloudshop-pre-dr | grep 'Started:')"
echo "RTO: tempo entre delete e restore completo"
```

**Critérios de aceite:**
- [ ] Cenário 1: Node drain sem downtime (PDBs + topology spread)
- [ ] Cenário 2: StatefulSet pod recovery com dados preservados (PVC intact)
- [ ] Cenário 3: NetworkPolicy block/unblock com impacto observado
- [ ] Cenário 4: HPA scale-up 2→10 sob carga, scale-down após cooldown
- [ ] Cenário 5: DR drill com RPO ≤ 1h e RTO ≤ 15min
- [ ] Todos os cenários documentados com evidências (screenshots, logs, métricas)
- [ ] Nenhum cenário causou perda de dados

---

### Desafio 17.8 — Production Day: Go-Live Simulation

Simule o dia de lançamento da plataforma CloudShop.

**Requisitos:**

**Fase 1: Pre-launch checklist (T-2h)**

```bash
#!/bin/bash
echo "=== CloudShop Pre-Launch Checklist ==="

# 1. All pods running
EXPECTED_PODS=21  # 6 services × 3 replicas + 3 StatefulSets
RUNNING=$(kubectl get pods -n cloudshop-prod -n cloudshop-data --field-selector=status.phase=Running --no-headers | wc -l)
echo "Pods Running: $RUNNING / $EXPECTED_PODS"

# 2. All services healthy
for svc in order product user payment inventory notification; do
  STATUS=$(kubectl exec deploy/${svc}-service -n cloudshop-prod -- \
    wget -qO- http://localhost:8080/health/ready 2>/dev/null | jq -r '.status')
  echo "${svc}-service: $STATUS"
done

# 3. ArgoCD sync status
kubectl get applications -n argocd -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status' | grep prod

# 4. Prometheus targets
UP=$(kubectl exec -n monitoring prometheus-k8s-0 -- \
  wget -qO- 'localhost:9090/api/v1/targets' 2>/dev/null | jq '[.data.activeTargets[] | select(.health=="up")] | length')
echo "Prometheus targets UP: $UP"

# 5. No critical alerts
ALERTS=$(kubectl exec -n monitoring prometheus-k8s-0 -- \
  wget -qO- 'localhost:9090/api/v1/alerts' 2>/dev/null | jq '[.data.alerts[] | select(.state=="firing" and .labels.severity=="critical")] | length')
echo "Critical alerts: $ALERTS (must be 0)"

# 6. etcd & Velero backup
echo "Last Velero backup:"
velero backup get --output json | jq -r '.items | sort_by(.status.completionTimestamp) | last | "\(.metadata.name) at \(.status.completionTimestamp)"'

echo "=== Checklist Complete ==="
```

**Fase 2: Soft launch (T+0)**

```bash
# 1. Abrir tráfego gradualmente (canary weight)
# nginx ingress annotation ou Gateway API HTTPRoute weight

# 2. Monitor: watch dashboards
# - Error rate < 0.1%
# - p99 < 500ms
# - No pod restarts
# - Kafka consumer lag = 0

# 3. Load test fase 1: 10 VUs por 5 min
k6 run --vus 10 --duration 5m k6/scenarios/cloudshop-journey.js
```

**Fase 3: Full launch (T+1h)**

```bash
# 1. Aumentar traffic weight para 100%

# 2. Load test fase 2: pico (100 VUs, 15 min)
k6 run --vus 100 --duration 15m k6/scenarios/cloudshop-journey.js

# 3. Observar HPA scaling
kubectl get hpa -n cloudshop-prod -w

# 4. Verificar SLOs
# Availability: > 99.9% (< 1 request failed per 1000)
# Latency p99: < 500ms
```

**Fase 4: Incident response (T+3h)**

```bash
# Simular: payment-service com memory leak (OOMKilled)
kubectl set resources deployment/payment-service -n cloudshop-prod \
  --limits=memory=64Mi  # artificialmente baixo para trigger OOM

# Observar:
# 1. Pod OOMKilled → restart
# 2. Alerta PodCrashLooping dispara
# 3. PDB impede que todas as replicas caiam
# 4. HPA tenta escalar (mas novas pods também OOM)

# Resolução:
kubectl set resources deployment/payment-service -n cloudshop-prod \
  --limits=memory=512Mi  # restaurar

# Post-incident:
# - Timeline do incidente
# - Tempo de detecção (alertas)
# - Tempo de resolução
# - Root cause
# - Action items
```

**Fase 5: Post-launch report**

```markdown
# CloudShop Production Launch Report

## Timeline
| Horário | Evento | Status |
|---------|--------|--------|
| T-2h | Pre-launch checklist | ✅ All green |
| T+0 | Soft launch (10 VUs) | ✅ |
| T+1h | Full launch (100 VUs) | ✅ |
| T+3h | Incident: payment-service OOM | ⚠️ Resolved in __ min |
| T+4h | Post-incident review | ✅ |

## SLO Results
| SLO | Target | Actual |
|-----|--------|--------|
| Availability | 99.9% | ___% |
| Latency p99 | < 500ms | ___ms |
| Error Rate | < 0.1% | ___% |

## Scaling Events
| Service | Min → Max | Trigger |
|---------|-----------|---------|
| product-service | 3 → 7 | CPU > 70% |
| order-service | 3 → 5 | CPU > 70% |

## Action Items
1. [ ] payment-service memory limits: increase to 1Gi
2. [ ] Add memory-based HPA metric
3. [ ] Create runbook for OOM incidents
4. [ ] Review alert thresholds for faster detection
```

**Critérios de aceite:**
- [ ] Pre-launch checklist automatizado e 100% green
- [ ] Soft launch sem erros (10 VUs, 5 min)
- [ ] Full launch: error rate < 0.1%, p99 < 500ms (100 VUs, 15 min)
- [ ] HPA scaling observado e documentado
- [ ] Incident simulado, detectado por alertas, resolvido
- [ ] Post-incident review documentado com timeline
- [ ] SLO results tabulados
- [ ] Production launch report completo

---

## Definição de Pronto (DoD)

### Platform Foundation
- [ ] Cluster: 1 CP + 3 workers multi-AZ
- [ ] 10 namespaces com PSS, RBAC, quotas, LimitRanges
- [ ] NetworkPolicies default-deny + allow-list

### Stateful Infrastructure
- [ ] PostgreSQL, Redis, Kafka: 3 replicas cada, PDBs, probes, PVCs
- [ ] VolumeSnapshots funcionais
- [ ] External Secrets sincronizando

### Application Workloads
- [ ] 6 Deployments com securityContext, probes, resources, topology spread
- [ ] HPAs, PDBs, graceful shutdown
- [ ] Ingress com TLS + rate limiting
- [ ] CronJob + data migration Job

### Observability
- [ ] Prometheus + ServiceMonitors + AlertRules + AlertManager
- [ ] Grafana: 3 dashboards (RED, USE, SLOs)
- [ ] FluentBit → Loki (centralized logging)
- [ ] OTel Collector → Tempo (distributed tracing)

### GitOps
- [ ] ArgoCD ApplicationSet: 18 Applications (6×3)
- [ ] Kustomize: base + 3 overlays
- [ ] Image Updater para dev
- [ ] Promotion pipeline testado

### Security
- [ ] Kyverno: ≥ 5 enforce policies
- [ ] Trivy: zero CRITICAL CVEs
- [ ] Audit logging habilitado
- [ ] Falco monitorando runtime

### Resilience
- [ ] Node drain sem downtime
- [ ] StatefulSet recovery com dados intactos
- [ ] HPA scale-up/down funcional
- [ ] DR drill: RPO ≤ 1h, RTO ≤ 15min

### Production Launch
- [ ] Pre-launch checklist automatizado
- [ ] Go-live simulation executado (soft + full)
- [ ] Incident simulado e resolvido
- [ ] Production launch report documentado

---

## Rubrica de Avaliação (0–100)

| Critério | Peso | 0-25 | 26-50 | 51-75 | 76-100 |
|---|---|---|---|---|---|
| **Platform Bootstrap** | 15% | Cluster básico | + namespaces + RBAC | + PSS + quotas | + NetPol + multi-AZ |
| **Stateful Infra** | 10% | Single replica DBs | StatefulSets + PVCs | + PDBs + probes | + Snapshots + External Secrets |
| **Workloads** | 15% | Deployments básicos | + probes + resources | + HPA + PDB + topology | + graceful shutdown + Ingress TLS |
| **Observability** | 15% | Sem monitoring | Prometheus + Grafana | + alerting + tracing | + SLOs + logging + dashboards completos |
| **GitOps** | 15% | kubectl apply manual | ArgoCD básico | + ApplicationSet + overlays | + Image Updater + promotion pipeline |
| **Security** | 10% | Sem security | PSS + RBAC | + Kyverno + scanning | + Falco + audit + supply chain |
| **Resilience** | 10% | Sem testes | 1-2 cenários | 3-4 cenários | 5 cenários + DR drill com métricas |
| **Production Day** | 10% | Sem simulação | Checklist manual | Soft + full launch | + incident + report + SLOs |

---

## Como Isso Aparece em Entrevistas

- "Descreva como você montaria uma plataforma Kubernetes production-grade do zero"
- "Como você implementaria isolamento entre ambientes (dev/staging/prod)?"
- "Qual sua estratégia de observabilidade para microsserviços no Kubernetes?"
- "Como funciona seu pipeline de GitOps? Como promove entre ambientes?"
- "Descreva um DR drill que você executou. Quais foram RPO e RTO?"
- "Como você lida com segurança em Kubernetes? Quais policies enforce?"
- "O que acontece quando um node falha? Como a plataforma se recupera?"
- "Como você faria canary deploy de um microsserviço?"
- "Descreva como monitoraria SLOs em produção"
- "Qual é seu process de incident response em Kubernetes?"

---

## Commits Sugeridos

```
feat(level-17): bootstrap cluster with namespaces, RBAC, PSS, quotas, NetworkPolicies
feat(level-17): deploy stateful infrastructure (PostgreSQL, Redis, Kafka StatefulSets)
feat(level-17): deploy 6 microservices with full production config
feat(level-17): setup observability stack (Prometheus, Grafana, Loki, Tempo, OTel)
feat(level-17): configure ArgoCD GitOps with ApplicationSet matrix
feat(level-17): apply security hardening (Kyverno, External Secrets, Falco, Trivy)
feat(level-17): execute resilience testing (5 chaos scenarios + DR drill)
feat(level-17): execute production launch simulation with incident response
docs(level-17): add production launch report with SLO results
```
