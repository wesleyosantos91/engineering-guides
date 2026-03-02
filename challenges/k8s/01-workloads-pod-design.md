# Level 1 — Workloads & Pod Design

> **Objetivo:** Dominar os recursos de workload do Kubernetes — Deployments, StatefulSets, Jobs, CronJobs —
> e aplicar corretamente probes, init containers, topologia e estratégias de update na plataforma CloudShop.

**Referência:** [Workloads & Pod Design Best Practices](../../.docs/k8s/01-workloads-pod-design.md)

---

## Objetivo de Aprendizado

- Implementar Deployments com rolling update e zero-downtime
- Configurar probes (liveness, readiness, startup) corretamente
- Usar StatefulSets para workloads stateful (PostgreSQL, Redis, Kafka)
- Criar Jobs e CronJobs para tarefas batch
- Aplicar init containers para dependências
- Configurar Pod Topology Spread para distribuição multi-AZ

---

## Contexto do Domínio

Neste nível, você criará os **workloads reais da plataforma CloudShop**: APIs como Deployments, databases como StatefulSets, migrações como Jobs e relatórios como CronJobs.

---

## Desafios

### Desafio 1.1 — Deployments com Rolling Update

Crie Deployments para os microsserviços da CloudShop com estratégia de rolling update zero-downtime.

**Requisitos:**
- Deployment `order-service` com 3 replicas
- Rolling update com `maxSurge: 1`, `maxUnavailable: 0`
- `terminationGracePeriodSeconds: 60`
- `preStop` hook com sleep 10s
- Labels padronizados (`app.kubernetes.io/*`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: cloudshop-prod
  labels:
    app.kubernetes.io/name: order-service
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: cloudshop
    app.kubernetes.io/managed-by: kubectl
spec:
  replicas: 3
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app.kubernetes.io/name: order-service
  template:
    metadata:
      labels:
        app.kubernetes.io/name: order-service
        app.kubernetes.io/component: api
        app.kubernetes.io/part-of: cloudshop
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: order-service
          image: nginx:1.27-alpine  # placeholder
          ports:
            - containerPort: 8080
              name: http
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
```

**Critérios de aceite:**
- [ ] Deployment com 3 replicas Running
- [ ] Rolling update com zero-downtime verificado
- [ ] `kubectl rollout status` mostra progresso
- [ ] Rollback funciona (`kubectl rollout undo`)
- [ ] preStop hook configurado

---

### Desafio 1.2 — Probes (Health Checks)

Configure liveness, readiness e startup probes para cada microsserviço.

**Requisitos:**
- Liveness probe leve (sem dependências externas)
- Readiness probe verificando dependências
- Startup probe para serviços com inicialização lenta
- Valores adequados de `initialDelaySeconds`, `periodSeconds`, `failureThreshold`

```yaml
spec:
  containers:
    - name: order-service
      image: myapp/order-service:v1.0.0
      ports:
        - containerPort: 8080
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 15
        periodSeconds: 10
        timeoutSeconds: 3
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 3
      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        periodSeconds: 10
        failureThreshold: 30  # 30 * 10s = 5min para iniciar
```

**Critérios de aceite:**
- [ ] Liveness probe NÃO verifica dependências externas
- [ ] Readiness probe verifica dependências (DB, cache)
- [ ] Startup probe configurado para serviços lentos
- [ ] Pod removido dos endpoints quando readiness falha
- [ ] Pod restartado quando liveness falha

---

### Desafio 1.3 — StatefulSets para Databases

Crie StatefulSets para PostgreSQL e Redis com storage persistente.

**Requisitos:**
- StatefulSet `postgresql` com 3 replicas
- Headless Service (`clusterIP: None`)
- VolumeClaimTemplate com StorageClass
- PodDisruptionBudget (`minAvailable: 2`)

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
      name: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: cloudshop-data
spec:
  serviceName: postgresql
  replicas: 3
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      terminationGracePeriodSeconds: 120
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: POSTGRES_DB
              value: cloudshop
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
              subPath: pgdata
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
---
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

**Critérios de aceite:**
- [ ] StatefulSet com 3 replicas Running (ordered: postgresql-0, 1, 2)
- [ ] Headless Service criado (`clusterIP: None`)
- [ ] PVCs criados automaticamente (1 por replica)
- [ ] DNS estável funciona (`postgresql-0.postgresql.cloudshop-data.svc.cluster.local`)
- [ ] PDB configurado com `minAvailable: 2`
- [ ] Dados persistem após restart do Pod

---

### Desafio 1.4 — Jobs e CronJobs

Crie Jobs para migrações e CronJobs para relatórios.

**Requisitos:**
- Job `db-migration` com `backoffLimit: 3` e `activeDeadlineSeconds: 600`
- CronJob `daily-report` com schedule `0 2 * * *`
- `ttlSecondsAfterFinished` para limpeza automática
- Jobs devem ser idempotentes

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-v1
  namespace: cloudshop-data
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: Never
      initContainers:
        - name: wait-for-db
          image: busybox:1.36
          command: ['sh', '-c', 'until nc -z postgresql-0.postgresql 5432; do sleep 2; done']
      containers:
        - name: migration
          image: myapp/migration:v1.0.0
          command: ['./migrate', 'up']
          env:
            - name: DATABASE_URL
              value: "postgres://user:pass@postgresql-0.postgresql:5432/cloudshop?sslmode=disable"
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
  namespace: cloudshop-prod
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  startingDeadlineSeconds: 300
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 1800
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: report
              image: myapp/report-generator:v1.0.0
              resources:
                requests:
                  cpu: 500m
                  memory: 512Mi
                limits:
                  cpu: 1000m
                  memory: 1Gi
```

**Critérios de aceite:**
- [ ] Job de migration completa com sucesso
- [ ] Init container aguarda PostgreSQL ficar disponível
- [ ] CronJob `daily-report` criado com schedule correto
- [ ] `concurrencyPolicy: Forbid` impede execução paralela
- [ ] `ttlSecondsAfterFinished` limpa jobs completados
- [ ] Jobs são idempotentes (podem ser re-executados)

---

### Desafio 1.5 — Init Containers e Multi-Container Patterns

Use init containers para dependências e sidecars para funcionalidades auxiliares.

**Requisitos:**
- Init container que aguarda PostgreSQL
- Init container que executa migration
- Sidecar de logging (FluentBit)
- Sidecar de proxy (envoy) — Ambassador pattern

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-service-complete
  namespace: cloudshop-dev
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ['sh', '-c', 'until nc -z postgresql.cloudshop-data 5432; do echo waiting for db; sleep 2; done']
    - name: wait-for-redis
      image: busybox:1.36
      command: ['sh', '-c', 'until nc -z redis.cloudshop-data 6379; do echo waiting for redis; sleep 2; done']
  containers:
    - name: order-service
      image: myapp/order-service:v1.0.0
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
    - name: log-shipper
      image: fluent/fluent-bit:3.0
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
          readOnly: true
  volumes:
    - name: logs
      emptyDir:
        sizeLimit: 200Mi
```

**Critérios de aceite:**
- [ ] Init containers executam em ordem sequencial
- [ ] App container só inicia após todos os init containers completarem
- [ ] Sidecar de logging lê logs do container principal
- [ ] Volume compartilhado (`emptyDir`) funciona entre containers
- [ ] `kubectl logs <pod> -c <container>` funciona para cada container

---

### Desafio 1.6 — Pod Topology Spread

Distribua Pods entre zones e nodes para alta disponibilidade.

**Requisitos:**
- TopologySpreadConstraints por zone e hostname
- `maxSkew: 1` com `DoNotSchedule` para zones
- `maxSkew: 1` com `ScheduleAnyway` para nodes
- Pod Anti-Affinity para evitar co-localização

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: order-service
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: order-service
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app.kubernetes.io/name
                  operator: In
                  values: ["order-service"]
            topologyKey: kubernetes.io/hostname
```

**Critérios de aceite:**
- [ ] Pods distribuídos entre 3 nodes (1 por node)
- [ ] `kubectl get pods -o wide` mostra distribuição nos nodes
- [ ] Topology spread respeitado (`maxSkew: 1`)
- [ ] Anti-affinity impede 2+ pods no mesmo node (se possível)

---

## Definição de Pronto (DoD)

- [ ] Deployments para todos os microsserviços (order, product, user, payment, inventory, notification)
- [ ] StatefulSets para PostgreSQL e Redis
- [ ] Jobs para migrações e CronJobs para relatórios
- [ ] Probes configurados em todos os workloads
- [ ] Init containers para dependências
- [ ] Topology spread para distribuição multi-node
- [ ] Commit: `feat(level-1): add workloads and pod design`

---

## Checklist

- [ ] 6 Deployments criados (microsserviços)
- [ ] 2 StatefulSets criados (PostgreSQL, Redis)
- [ ] 1 Job criado (migration)
- [ ] 1 CronJob criado (daily-report)
- [ ] Probes configurados em todos os Deployments
- [ ] Init containers configurados
- [ ] Sidecar pattern implementado (logging)
- [ ] Rolling update testado com zero-downtime
- [ ] Rollback testado com `kubectl rollout undo`
- [ ] PDB configurado para StatefulSets
- [ ] Topology spread configurado
