# Level 12 — CKAD (Certified Kubernetes Application Developer)

> **Objetivo:** Preparar-se para a certificação CKAD — prova prática hands-on que valida
> habilidades de desenvolvimento e deploy de aplicações no Kubernetes.

**Referência:** [CKAD Certification Guide](../../.docs/k8s/12-ckad.md)

---

## Sobre a Certificação

| Campo | Detalhe |
|-------|---------|
| Tipo | Performance-based (hands-on CLI) |
| Duração | 120 minutos |
| Aprovação | 66% |
| Validade | 2 anos |
| Custo | $395 USD (inclui 1 retake) |
| Ambiente | Browser com terminal + cluster real |

### Domínios

| Domínio | Peso |
|---------|------|
| Application Design and Build | 20% |
| Application Deployment | 20% |
| Application Observability and Maintenance | 15% |
| Application Environment, Configuration and Security | 25% |
| Services and Networking | 20% |

---

## Contexto do Domínio

A prova CKAD é 100% prática — você resolve tarefas em clusters reais via terminal. Use a CloudShop como playground para praticar cenários semelhantes aos da prova. Velocidade com `kubectl` é crucial.

---

## Desafios

### Desafio 12.1 — Application Design and Build (20%)

Pratique criação e design de aplicações Kubernetes.

**Tópicos da prova:**
- Definir, construir e modificar container images
- Jobs e CronJobs
- Multi-container Pods (sidecar, init containers)
- Volumes (emptyDir, PVC)

**Exercícios no estilo da prova:**
```bash
# ⏱️ Tempo-alvo: 3-5 minutos por exercício

# Ex 1: Criar um Pod multi-container com sidecar
# Pod "app-with-sidecar" no namespace "ckad-practice"
# Container 1: nginx:1.25, porta 80
# Container 2 (sidecar): busybox, comando "tail -f /var/log/nginx/access.log"
# Compartilhar volume emptyDir para /var/log/nginx

kubectl create namespace ckad-practice

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
  namespace: ckad-practice
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      volumeMounts:
        - name: logs
          mountPath: /var/log/nginx
    - name: log-reader
      image: busybox:1.36
      command: ["sh", "-c", "tail -f /var/log/nginx/access.log"]
      volumeMounts:
        - name: logs
          mountPath: /var/log/nginx
  volumes:
    - name: logs
      emptyDir: {}
EOF

# Ex 2: Criar um CronJob que roda a cada 5 minutos
kubectl create cronjob report-gen \
  --image=busybox:1.36 \
  --schedule="*/5 * * * *" \
  --namespace=ckad-practice \
  -- sh -c "echo 'Report generated at $(date)' >> /tmp/report.txt"

# Ex 3: Criar um Job com completions=3 e parallelism=2
kubectl create job batch-process \
  --image=busybox:1.36 \
  --namespace=ckad-practice \
  -- sh -c "echo Processing item; sleep 5"
kubectl patch job batch-process -n ckad-practice \
  -p '{"spec":{"completions":3,"parallelism":2}}'

# Ex 4: Pod com init container
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
  namespace: ckad-practice
spec:
  initContainers:
    - name: init-db
      image: busybox:1.36
      command: ["sh", "-c", "until nslookup postgresql.cloudshop-data; do echo waiting for db; sleep 2; done"]
  containers:
    - name: app
      image: nginx:1.25
EOF
```

**Critérios de aceite:**
- [ ] Multi-container Pod com volume compartilhado (< 5 min)
- [ ] CronJob com schedule correto (< 3 min)
- [ ] Job com completions e parallelism (< 3 min)
- [ ] Pod com init container (< 3 min)
- [ ] Usar `--dry-run=client -o yaml` para gerar templates rápido

---

### Desafio 12.2 — Application Deployment (20%)

Pratique deployment strategies e rollouts.

**Tópicos da prova:**
- Deployments com rolling update
- Rollbacks
- Scaling (manual e HPA)
- Helm basics

**Exercícios no estilo da prova:**
```bash
# Ex 1: Criar Deployment com rolling update strategy
kubectl create deployment web-app \
  --image=nginx:1.24 \
  --replicas=4 \
  --namespace=ckad-practice

# Configurar strategy
kubectl patch deployment web-app -n ckad-practice -p '{
  "spec": {
    "strategy": {
      "type": "RollingUpdate",
      "rollingUpdate": {
        "maxSurge": "25%",
        "maxUnavailable": 1
      }
    }
  }
}'

# Ex 2: Atualizar imagem e verificar rollout
kubectl set image deployment/web-app nginx=nginx:1.25 -n ckad-practice
kubectl rollout status deployment/web-app -n ckad-practice

# Ex 3: Rollback para versão anterior
kubectl rollout history deployment/web-app -n ckad-practice
kubectl rollout undo deployment/web-app -n ckad-practice
kubectl rollout undo deployment/web-app --to-revision=1 -n ckad-practice

# Ex 4: Scale e HPA
kubectl scale deployment/web-app --replicas=6 -n ckad-practice
kubectl autoscale deployment/web-app \
  --min=2 --max=10 --cpu-percent=70 \
  -n ckad-practice

# Ex 5: Blue/Green deployment
kubectl create deployment web-app-blue --image=nginx:1.24 --replicas=3 -n ckad-practice
kubectl create deployment web-app-green --image=nginx:1.25 --replicas=3 -n ckad-practice
# Service apontando para blue
kubectl create service clusterip web-app --tcp=80:80 -n ckad-practice
kubectl patch service web-app -n ckad-practice -p '{"spec":{"selector":{"app":"web-app-blue"}}}'
# Switch para green
kubectl patch service web-app -n ckad-practice -p '{"spec":{"selector":{"app":"web-app-green"}}}'
```

**Critérios de aceite:**
- [ ] Deployment com rolling update configurado (< 3 min)
- [ ] Rollout e rollback executados (< 2 min)
- [ ] Scale manual e HPA (< 3 min)
- [ ] Blue/Green switch via Service selector (< 5 min)
- [ ] `kubectl rollout history` mostra revisões

---

### Desafio 12.3 — Observability and Maintenance (15%)

Pratique monitoramento e troubleshooting de aplicações.

**Tópicos da prova:**
- Probes (liveness, readiness, startup)
- Logs
- Debugging
- Resource usage (kubectl top)

**Exercícios no estilo da prova:**
```bash
# Ex 1: Adicionar probes a um Deployment existente
kubectl patch deployment web-app -n ckad-practice --type=json -p '[
  {"op": "add", "path": "/spec/template/spec/containers/0/livenessProbe", "value": {
    "httpGet": {"path": "/", "port": 80},
    "initialDelaySeconds": 10,
    "periodSeconds": 5
  }},
  {"op": "add", "path": "/spec/template/spec/containers/0/readinessProbe", "value": {
    "httpGet": {"path": "/", "port": 80},
    "initialDelaySeconds": 5,
    "periodSeconds": 3
  }}
]'

# Ex 2: Troubleshooting — encontrar e corrigir problemas
# Cenário: Pod está CrashLoopBackOff
kubectl describe pod <crashing-pod> -n ckad-practice
kubectl logs <crashing-pod> -n ckad-practice --previous

# Ex 3: Verificar resource usage
kubectl top pods -n ckad-practice --sort-by=memory
kubectl top nodes

# Ex 4: Debug com ephemeral container
kubectl debug -it <pod-name> -n ckad-practice --image=busybox:1.36 --target=nginx

# Ex 5: Verificar eventos
kubectl get events -n ckad-practice --sort-by='.lastTimestamp'
```

**Critérios de aceite:**
- [ ] Probes adicionados via patch JSON (< 3 min)
- [ ] CrashLoopBackOff diagnosticado via logs/describe (< 3 min)
- [ ] `kubectl top` funciona e interpreta resultados
- [ ] Debug container usado para troubleshooting
- [ ] Events filtrados e analisados

---

### Desafio 12.4 — Environment, Configuration and Security (25%)

Pratique configuração e segurança (domínio de maior peso).

**Tópicos da prova:**
- ConfigMaps e Secrets
- ServiceAccounts
- SecurityContext
- ResourceRequirements
- Network Policies

**Exercícios no estilo da prova:**
```bash
# Ex 1: Criar ConfigMap e montar como volume
kubectl create configmap app-config \
  --from-literal=DB_HOST=postgresql \
  --from-literal=DB_PORT=5432 \
  --namespace=ckad-practice

# Montar no Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
  namespace: ckad-practice
spec:
  containers:
    - name: app
      image: nginx:1.25
      envFrom:
        - configMapRef:
            name: app-config
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: app-config
EOF

# Ex 2: Criar Secret e usar como env var
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=secret123 \
  --namespace=ckad-practice

# Ex 3: SecurityContext
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: ckad-practice
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
    - name: app
      image: nginx:1.25
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 250m
          memory: 256Mi
EOF

# Ex 4: ServiceAccount
kubectl create serviceaccount app-sa -n ckad-practice
# Usar no Pod: spec.serviceAccountName: app-sa

# Ex 5: Network Policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-db
  namespace: ckad-practice
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: web
      ports:
        - port: 5432
          protocol: TCP
EOF
```

**Critérios de aceite:**
- [ ] ConfigMap criado e montado (env + volume) (< 3 min)
- [ ] Secret criado e montado (< 3 min)
- [ ] SecurityContext configurado (runAsUser, readOnlyRootFilesystem) (< 3 min)
- [ ] ServiceAccount criado e atribuído (< 2 min)
- [ ] Network Policy criada (< 5 min)

---

### Desafio 12.5 — Services and Networking (20%)

Pratique exposição e conectividade de serviços.

**Tópicos da prova:**
- Services (ClusterIP, NodePort, LoadBalancer)
- Ingress
- Network Policies (mais profundo)

**Exercícios no estilo da prova:**
```bash
# Ex 1: Criar Service ClusterIP
kubectl expose deployment web-app \
  --port=80 --target-port=80 \
  --type=ClusterIP \
  --namespace=ckad-practice

# Ex 2: Criar Service NodePort
kubectl expose deployment web-app \
  --port=80 --target-port=80 \
  --type=NodePort --name=web-app-np \
  --namespace=ckad-practice

# Ex 3: Criar Ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: ckad-practice
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: web.ckad.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-app
                port:
                  number: 80
EOF

# Ex 4: DNS testing
kubectl run dns-test --rm -it --image=busybox:1.36 -n ckad-practice -- nslookup web-app

# Ex 5: Network Policy — deny all + allow specific
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: ckad-practice
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: ckad-practice
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to: []
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
EOF
```

**Critérios de aceite:**
- [ ] ClusterIP Service criado (< 1 min)
- [ ] NodePort Service criado (< 1 min)
- [ ] Ingress com host-based routing (< 3 min)
- [ ] DNS resolution verificado (< 2 min)
- [ ] Network Policy default-deny + allow (< 5 min)

---

### Desafio 12.6 — Simulado CKAD

Execute simulado prático no estilo da prova.

**Requisitos:**
- Ambiente isolado com cluster real (kind ou similar)
- 15-20 questões práticas
- 120 minutos
- Apenas terminal + documentação K8s oficial (kubernetes.io/docs)

**Dicas de velocidade:**
```bash
# Aliases essenciais
alias k=kubectl
alias kn='kubectl config set-context --current --namespace'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'

# Autocompletion
source <(kubectl completion bash)
complete -o default -F __start_kubectl k

# Dry-run para gerar templates
k run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
k create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
k create service clusterip web --tcp=80:80 --dry-run=client -o yaml > svc.yaml
k create configmap cm --from-literal=key=val --dry-run=client -o yaml
k create secret generic sec --from-literal=key=val --dry-run=client -o yaml

# Vim config para YAML
echo 'set expandtab shiftwidth=2 tabstop=2' >> ~/.vimrc

# Bookmarks úteis (kubernetes.io/docs)
# - Cheat sheet: /docs/reference/kubectl/cheatsheet
# - API reference: /docs/reference/generated/kubernetes-api
# - Tasks: /docs/tasks
```

**Recursos para simulado:**
```markdown
- https://killer.sh (simulado oficial — incluído na compra da prova)
- Praticar em kind/minikube com timer
- FOCO: velocidade + precisão com kubectl
```

**Critérios de aceite:**
- [ ] Simulado completo (120 min)
- [ ] Score ≥ 66%
- [ ] Aliases e autocompletion configurados
- [ ] Dry-run para templates dominado
- [ ] Questões erradas revisadas com foco em velocidade

---

## Definição de Pronto (DoD)

- [ ] 5 domínios praticados com exercícios hands-on
- [ ] Aliases e autocompletion configurados
- [ ] Simulado killer.sh completado
- [ ] Score ≥ 66% no simulado
- [ ] Fraquezas identificadas e revisadas

---

## Checklist

- [ ] Application Design and Build — exercícios práticos
- [ ] Application Deployment — rollouts e scaling
- [ ] Observability and Maintenance — probes e debugging
- [ ] Configuration and Security — ConfigMaps, Secrets, RBAC
- [ ] Services and Networking — Services, Ingress, NetworkPolicy
- [ ] Aliases kubectl configurados
- [ ] 2+ simulados com score ≥ 66%
