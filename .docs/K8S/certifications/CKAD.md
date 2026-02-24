# CKAD — Certified Kubernetes Application Developer

## Informações do Exame

| Item | Detalhe |
|------|---------|
| **Formato** | Hands-on (terminal Linux) |
| **Duração** | 2 horas |
| **Questões** | 15-20 tarefas |
| **Passing Score** | 66% |
| **Validade** | 2 anos |
| **Pré-requisitos** | Nenhum (recomenda-se experiência básica K8s) |
| **Preço** | $395 USD (inclui 1 retake + killer.sh) |
| **Ambiente** | PSI Bridge, 1 aba extra para docs |

## Domínios do Exame

| Domínio | Peso | Tópicos |
|---------|:---:|---------|
| **Application Design and Build** | 20% | Container images, Jobs, multi-container patterns |
| **Application Deployment** | 20% | Deployments, rolling updates, Helm |
| **Application Observability and Maintenance** | 15% | Probes, logging, debugging, deprecation |
| **Application Environment, Configuration and Security** | 25% | ConfigMaps, Secrets, ServiceAccounts, SecurityContext, ResourceRequirements |
| **Services and Networking** | 20% | Services, Ingress, NetworkPolicy |

## Comandos Essenciais

### Setup Inicial (faça no começo do exame)
```bash
alias k=kubectl
alias kn='kubectl config set-context --current --namespace'
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'
# Autocompletion
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
```

## Conteúdo por Domínio

### 1. Application Design and Build (20%)

#### Criar Pods rapidamente
```bash
# Pod simples
k run nginx --image=nginx:1.25

# Pod com porta
k run nginx --image=nginx:1.25 --port=80

# Pod com comando
k run busybox --image=busybox --command -- sleep 3600

# Gerar YAML
k run nginx --image=nginx:1.25 --port=80 $do > pod.yaml

# Pod temporário para debug
k run debug --rm -it --image=busybox -- sh
```

#### Multi-Container Patterns
```yaml
# Sidecar — funcionalidade auxiliar
spec:
  containers:
    - name: app
      image: app:v1
    - name: log-agent
      image: fluentbit:latest  # sidecar

# Init Container — inicialização
spec:
  initContainers:
    - name: init-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
  containers:
    - name: app
      image: app:v1

# Ambassador — proxy para serviços externos
# Adapter — transforma dados (ex: formato de métricas)
```

#### Jobs e CronJobs
```bash
# Job
k create job myjob --image=busybox -- echo "Hello"

# CronJob
k create cronjob mybackup --image=busybox --schedule="0 2 * * *" -- /bin/sh -c "echo backup"
```

```yaml
# Job com configurações
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  completions: 3      # total de completions
  parallelism: 2      # execuções paralelas
  backoffLimit: 4     # retries
  activeDeadlineSeconds: 300
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: pi
          image: perl:5.34
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
```

#### Dockerfile e Build
```dockerfile
# Multi-stage build
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/server /server
USER 65534:65534
ENTRYPOINT ["/server"]
```

### 2. Application Deployment (20%)

#### Deployments
```bash
# Criar
k create deployment nginx --image=nginx:1.25 --replicas=3

# Escalar
k scale deployment nginx --replicas=5

# Atualizar imagem
k set image deployment/nginx nginx=nginx:1.26

# Rollout
k rollout status deployment/nginx
k rollout history deployment/nginx
k rollout undo deployment/nginx
k rollout undo deployment/nginx --to-revision=2

# Rollout pause/resume (canary manual)
k rollout pause deployment/nginx
k set image deployment/nginx nginx=nginx:1.26
k rollout resume deployment/nginx
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

#### Helm
```bash
# Básicos que caem na prova
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx
helm install my-nginx bitnami/nginx
helm upgrade my-nginx bitnami/nginx --set replicaCount=3
helm rollback my-nginx 1
helm uninstall my-nginx
helm list
helm history my-nginx
```

### 3. Application Observability and Maintenance (15%)

#### Probes
```yaml
# Liveness — container está vivo?
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10

# Readiness — container está pronto?
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5

# Startup — para apps que demoram para iniciar
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

#### Logging e Debug
```bash
# Logs
k logs pod-name
k logs pod-name -c container-name  # multi-container
k logs pod-name --previous          # container anterior
k logs pod-name -f                  # follow
k logs -l app=myapp                 # por label

# Debug
k describe pod pod-name
k get events --sort-by='.lastTimestamp'
k exec -it pod-name -- sh
k debug pod/pod-name -it --image=busybox
k top pods
```

### 4. Application Environment, Configuration and Security (25%)

#### ConfigMaps
```bash
# Criar
k create configmap myconfig --from-literal=key1=val1 --from-literal=key2=val2
k create configmap myconfig --from-file=config.properties
k create configmap myconfig --from-env-file=.env
```

```yaml
# Usar como env
envFrom:
  - configMapRef:
      name: myconfig
# Ou
env:
  - name: KEY1
    valueFrom:
      configMapKeyRef:
        name: myconfig
        key: key1
# Como volume
volumes:
  - name: config
    configMap:
      name: myconfig
```

#### Secrets
```bash
k create secret generic mysecret --from-literal=password=s3cret
k create secret docker-registry regcred \
  --docker-server=registry.io \
  --docker-username=user \
  --docker-password=pass
```

#### Security Context
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
          add: ["NET_BIND_SERVICE"]
```

#### Service Accounts
```bash
k create serviceaccount mysa
```

```yaml
spec:
  serviceAccountName: mysa
  automountServiceAccountToken: false
```

#### Resource Requirements
```yaml
resources:
  requests:
    cpu: 250m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

### 5. Services and Networking (20%)

#### Services
```bash
# Expose deployment
k expose deployment nginx --port=80 --target-port=8080 --type=ClusterIP
k expose deployment nginx --port=80 --type=NodePort

# Criar service
k create service clusterip mysvc --tcp=80:8080
```

#### Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

#### Network Policies
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: db
      ports:
        - port: 5432
    - ports:  # DNS
        - port: 53
          protocol: UDP
```

## Dicas Específicas para CKAD

1. **Velocidade** — use imperative commands, gere YAML com `--dry-run=client -o yaml`.
2. **kubectl explain** — `k explain pod.spec.containers.livenessProbe --recursive`.
3. **Não decore YAML** — copie da docs e modifique.
4. **Contexto** — verifique `kubectl config use-context` no início de cada questão.
5. **Multi-container** — saiba os padrões (sidecar, init, ambassador, adapter).
6. **NetworkPolicy** — pratique bastante, é frequente e difícil.
7. **Volumes** — ConfigMap, Secret, EmptyDir, PVC.
8. **Tempo** — ~6 min por questão, pule as difíceis.

## Recursos de Estudo

- [CKAD Curriculum](https://github.com/cncf/curriculum)
- [Kubernetes Docs — Tasks](https://kubernetes.io/docs/tasks/)
- [Kubernetes Docs — Concepts](https://kubernetes.io/docs/concepts/)
- KodeKloud CKAD Course (Mumshad)
- Udemy CKAD Course
- killer.sh (simulador oficial — incluso no voucher)
- [CKAD Exercises (dgkanatsios)](https://github.com/dgkanatsios/CKAD-exercises)
- Killercoda Scenarios
