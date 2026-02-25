# Workloads & Pod Design — Best Practices

## 1. Pod Design Principles

### 1.1 Um Processo por Container
```yaml
# ✅ BOM — cada container com uma responsabilidade
spec:
  containers:
    - name: api
      image: myapp/api:v1.2.0
    - name: log-shipper  # sidecar com responsabilidade separada
      image: fluentbit:latest
```
- Cada container deve executar **um único processo principal**.
- Use **sidecar pattern** para funcionalidades auxiliares (logging, proxy, monitoring).
- Evite rodar múltiplos processos dentro do mesmo container (não use supervisord).

### 1.2 Imagens de Container
- **Sempre use tags imutáveis** (SHA digest ou semver), nunca `latest` em produção.
- Prefira **imagens distroless ou Alpine** para reduzir superfície de ataque.
- Use **multi-stage builds** para minimizar o tamanho da imagem.
- Escaneie imagens com ferramentas como **Trivy, Grype ou Snyk**.

```yaml
# ✅ BOM — tag imutável
image: myapp/api:v1.2.0@sha256:abc123...

# ❌ RUIM — tag mutável
image: myapp/api:latest
```

### 1.3 Labels e Annotations
```yaml
metadata:
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/version: "1.2.0"
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: ecommerce
    app.kubernetes.io/managed-by: helm
    environment: production
    team: platform
  annotations:
    description: "API principal do e-commerce"
    oncall: "team-platform@company.com"
```
- Use **labels padronizados** do Kubernetes (`app.kubernetes.io/*`).
- Labels para **seleção e filtragem**, annotations para **metadata informativa**.
- Defina um **esquema de labels** consistente em toda a organização.

## 2. Probes (Health Checks)

### 2.1 Liveness Probe
Verifica se o container está vivo. Se falhar, o kubelet **reinicia o container**.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
```

**Regras:**
- Deve ser **leve e rápido** (sem dependências externas).
- Não verifique banco de dados ou serviços externos na liveness.
- Configure `initialDelaySeconds` adequadamente para evitar restart loops.

### 2.2 Readiness Probe
Verifica se o container está pronto para receber tráfego.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

**Regras:**
- **Pode verificar dependências** (DB, cache, etc.).
- Se falhar, o Pod é removido dos endpoints do Service.
- Ideal para warm-up de caches e connection pools.

### 2.3 Startup Probe
Para aplicações com tempo de inicialização longo.

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30  # 30 * 10 = 300s = 5min para iniciar
```

**Regras:**
- Use quando a aplicação demora para iniciar (JVM, ML models, etc.).
- Enquanto o startup probe não passar, liveness e readiness não são executados.

## 3. Deployments

### 3.1 Rolling Update Strategy
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # máximo de pods extras durante update
      maxUnavailable: 0   # zero downtime
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: myapp
          image: myapp:v1.2.0
          ports:
            - containerPort: 8080
```

### 3.2 Boas Práticas de Deployment
- **`maxUnavailable: 0`** para zero-downtime deployments.
- Configure **`terminationGracePeriodSeconds`** adequadamente.
- Implemente **graceful shutdown** na aplicação (SIGTERM handling).
- Use **`preStop` hook** para aguardar draining de conexões.

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]  # aguarda remoção dos endpoints
```

### 3.3 Rollback
- Mantenha **revision history** (`revisionHistoryLimit: 10`).
- Use `kubectl rollout undo` para rollbacks rápidos.
- Implemente **canary ou blue-green** com ferramentas como Argo Rollouts ou Flagger.

## 4. StatefulSets

### Quando Usar
- Bancos de dados (PostgreSQL, MySQL, MongoDB).
- Sistemas de mensageria (Kafka, RabbitMQ).
- Aplicações que precisam de **identidade estável** e **storage persistente**.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  podManagementPolicy: OrderedReady  # ou Parallel
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
```

**Boas Práticas:**
- Use **headless Service** (`clusterIP: None`) para DNS estável.
- Configure **PodDisruptionBudget** para evitar perda de quórum.
- Use **`podManagementPolicy: Parallel`** quando a ordem não importa.
- Teste **failover e recovery** regularmente.

## 5. Jobs e CronJobs

### 5.1 Jobs
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 600  # timeout de 10min
  ttlSecondsAfterFinished: 3600  # limpa após 1h
  template:
    spec:
      restartPolicy: Never  # ou OnFailure
      containers:
        - name: migration
          image: myapp/migration:v1.2.0
```

### 5.2 CronJobs
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"  # 2AM diariamente
  concurrencyPolicy: Forbid  # não rodar em paralelo
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  startingDeadlineSeconds: 300
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: myapp/backup:v1.0.0
```

**Boas Práticas:**
- Sempre defina **`activeDeadlineSeconds`** para evitar jobs infinitos.
- Use **`ttlSecondsAfterFinished`** para limpeza automática.
- Configure **`concurrencyPolicy`** adequadamente em CronJobs.
- Jobs devem ser **idempotentes** (podem ser re-executados com segurança).

## 6. Init Containers

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']
    - name: migrate
      image: myapp/migration:v1.2.0
      command: ['./migrate', 'up']
  containers:
    - name: myapp
      image: myapp:v1.2.0
```

**Casos de uso:**
- Aguardar dependências ficarem disponíveis.
- Executar migrações de banco de dados.
- Popular caches ou baixar configurações.
- Ajustar permissões de volumes.

## 7. Pod Topology Spread Constraints

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: myapp
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: myapp
```

- Distribua Pods entre **zonas de disponibilidade**.
- Use `DoNotSchedule` para garantias fortes de distribuição.
- Combine com **Pod Anti-Affinity** para evitar co-localização.

## 8. Namespace Strategy

```
├── production/
│   ├── namespace: prod-frontend
│   ├── namespace: prod-backend
│   ├── namespace: prod-data
│   └── namespace: prod-monitoring
├── staging/
│   ├── namespace: stg-frontend
│   ├── namespace: stg-backend
│   └── namespace: stg-data
└── system/
    ├── namespace: kube-system
    ├── namespace: cert-manager
    ├── namespace: ingress-nginx
    └── namespace: monitoring
```

**Boas Práticas:**
- Separe por **ambiente** e/ou **time/domínio**.
- Aplique **ResourceQuotas** e **LimitRanges** por namespace.
- Use **NetworkPolicies** para isolar namespaces.
- Padronize nomes: `{env}-{team}` ou `{env}-{domain}`.
