# Level 0 — Fundamentos do Kubernetes

> **Objetivo:** Dominar os conceitos fundamentais do Kubernetes — cluster, API, kubectl, namespaces,
> pods e labels — antes de avançar para workloads complexos. Sem atalhos.

**Referência:** [Kubernetes Fundamentals](../../.docs/k8s/01-workloads-pod-design.md) · [KCNA](../../.docs/k8s/11-kcna.md)

---

## Objetivo de Aprendizado

- Entender a arquitetura de um cluster Kubernetes (Control Plane + Nodes)
- Dominar `kubectl` como ferramenta principal de interação
- Criar e gerenciar Pods, Namespaces e Labels
- Entender a API do Kubernetes (grupos, versões, recursos)
- Configurar ambiente de desenvolvimento local (kind ou minikube)

---

## Contexto do Domínio

Neste nível, você criará o **esqueleto da plataforma CloudShop** — setup do cluster local, namespaces para os ambientes (dev, staging, prod) e os primeiros Pods manuais para entender o ciclo de vida.

---

## Desafios

### Desafio 0.1 — Setup do Cluster Local

Configure um cluster Kubernetes local com múltiplos nodes usando kind.

**Requisitos:**
- Criar cluster com 1 control-plane e 3 workers
- Verificar nodes com `kubectl get nodes`
- Verificar componentes do control plane em `kube-system`
- Instalar `k9s`, `kubectx`, `kubens`

```yaml
# kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
  - role: worker
```

```bash
# Criar cluster
kind create cluster --name cloudshop --config kind-cluster.yaml

# Verificar
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

**Critérios de aceite:**
- [ ] Cluster criado com 4 nodes (1 control-plane + 3 workers)
- [ ] `kubectl get nodes` mostra todos os nodes Ready
- [ ] Componentes do kube-system estão Running
- [ ] `k9s` instalado e funcionando

---

### Desafio 0.2 — Explorar a Arquitetura do Cluster

Identifique e documente todos os componentes do Control Plane e dos Worker Nodes.

**Requisitos:**
- Listar todos os pods em `kube-system` e identificar o papel de cada um
- Usar `kubectl describe` em cada componente
- Documentar a função de: kube-apiserver, etcd, kube-scheduler, kube-controller-manager, kube-proxy, coredns

```bash
# Listar componentes
kubectl get pods -n kube-system -o wide

# Explorar API server
kubectl get --raw /api/v1 | jq '.resources[].name' | head -20

# Grupos de API
kubectl api-resources --sort-by=name | head -30
kubectl api-versions
```

**Critérios de aceite:**
- [ ] Documento criado com diagrama da arquitetura do cluster
- [ ] Função de cada componente do Control Plane descrita
- [ ] Função do kubelet e kube-proxy descrita
- [ ] API groups listados (core, apps, batch, rbac)

---

### Desafio 0.3 — Namespaces da Plataforma CloudShop

Crie a estrutura de namespaces para a plataforma CloudShop.

**Requisitos:**
- Criar namespaces: `cloudshop-dev`, `cloudshop-staging`, `cloudshop-prod`, `cloudshop-data`, `monitoring`, `logging`
- Aplicar labels padrão em todos os namespaces
- Configurar namespace default via `kubens`

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cloudshop-prod
  labels:
    app.kubernetes.io/part-of: cloudshop
    environment: production
    team: platform
    cost-center: engineering
```

```bash
# Criar todos os namespaces
kubectl apply -f namespaces/

# Verificar
kubectl get namespaces --show-labels

# Mudar namespace default
kubens cloudshop-dev
```

**Critérios de aceite:**
- [ ] 6 namespaces criados com labels padronizados
- [ ] Labels incluem: `environment`, `team`, `cost-center`
- [ ] `kubectl get ns --show-labels` mostra todos com labels
- [ ] Namespace default configurado para `cloudshop-dev`

---

### Desafio 0.4 — Primeiros Pods

Crie Pods manualmente para entender o ciclo de vida e conceitos básicos.

**Requisitos:**
- Criar Pod imperativo (`kubectl run`)
- Criar Pod declarativo (YAML)
- Criar Pod com múltiplos containers (sidecar)
- Observar eventos do Pod (`kubectl describe`)
- Entender status: Pending → ContainerCreating → Running → Terminated

```yaml
# pod-simple.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-simple
  namespace: cloudshop-dev
  labels:
    app.kubernetes.io/name: nginx
    app.kubernetes.io/component: webserver
    environment: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports:
        - containerPort: 80
          name: http
      resources:
        requests:
          cpu: 50m
          memory: 64Mi
        limits:
          cpu: 100m
          memory: 128Mi
```

```yaml
# pod-sidecar.yaml — Pod com sidecar de logging
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
  namespace: cloudshop-dev
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ['sh', '-c', 'while true; do echo "$(date) - Request processed" >> /var/log/app.log; sleep 5; done']
      volumeMounts:
        - name: logs
          mountPath: /var/log
    - name: log-shipper
      image: busybox:1.36
      command: ['sh', '-c', 'tail -f /var/log/app.log']
      volumeMounts:
        - name: logs
          mountPath: /var/log
  volumes:
    - name: logs
      emptyDir: {}
```

```bash
# Imperativo
kubectl run nginx-test --image=nginx:1.27 --port=80 -n cloudshop-dev

# Declarativo
kubectl apply -f pod-simple.yaml
kubectl apply -f pod-sidecar.yaml

# Observar
kubectl get pods -n cloudshop-dev -o wide
kubectl describe pod nginx-simple -n cloudshop-dev
kubectl logs app-with-sidecar -c log-shipper -n cloudshop-dev
```

**Critérios de aceite:**
- [ ] Pod simples criado e Running
- [ ] Pod com sidecar criado — ambos containers Running
- [ ] Logs do sidecar mostram output do app container
- [ ] `kubectl describe` mostra Events do Pod
- [ ] Entendimento documentado do ciclo de vida do Pod

---

### Desafio 0.5 — Labels, Selectors e Annotations

Aplique labels padronizados e use selectors para filtrar recursos.

**Requisitos:**
- Criar 5+ Pods com labels diferentes (app, component, environment, team, version)
- Usar selectors para filtrar: equality-based e set-based
- Aplicar annotations para metadata informativa
- Usar `kubectl label` para adicionar/remover labels

```bash
# Criar pods com labels variados
kubectl run web-v1 --image=nginx --labels="app=cloudshop,component=web,version=v1,team=frontend"
kubectl run web-v2 --image=nginx --labels="app=cloudshop,component=web,version=v2,team=frontend"
kubectl run api-v1 --image=nginx --labels="app=cloudshop,component=api,version=v1,team=backend"
kubectl run worker-v1 --image=busybox --labels="app=cloudshop,component=worker,version=v1,team=data" --command -- sleep 3600

# Filtrar por label
kubectl get pods -l app=cloudshop
kubectl get pods -l component=web
kubectl get pods -l 'version in (v1, v2)'
kubectl get pods -l 'team notin (data)'
kubectl get pods -l app=cloudshop,component=api

# Adicionar label
kubectl label pod web-v1 environment=dev
kubectl label pod web-v1 environment-  # remover

# Annotations
kubectl annotate pod web-v1 description="Frontend web server v1" oncall="team-frontend@company.com"
```

**Critérios de aceite:**
- [ ] 5+ Pods com labels seguindo convenção `app.kubernetes.io/*`
- [ ] Filtros por equality-based (`-l app=cloudshop`)
- [ ] Filtros por set-based (`-l 'version in (v1, v2)'`)
- [ ] Labels adicionados/removidos via `kubectl label`
- [ ] Annotations aplicadas para metadata informativa

---

### Desafio 0.6 — kubectl Mastery

Domine os comandos essenciais do kubectl para produtividade máxima.

**Requisitos:**
- Usar `kubectl explain` para explorar a API
- Usar dry-run + output YAML para gerar manifests
- Usar `kubectl diff` antes de aplicar mudanças
- Configurar aliases e autocompletion

```bash
# Setup de produtividade
alias k=kubectl
alias kn='kubectl config set-context --current --namespace'
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'
source <(kubectl completion bash)
complete -o default -F __start_kubectl k

# Explorar API
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy --recursive

# Gerar YAML (nunca escreva do zero)
kubectl run nginx --image=nginx:1.27 --port=80 $do > nginx-pod.yaml
kubectl create deployment web --image=nginx --replicas=3 $do > web-deploy.yaml
kubectl create service clusterip web --tcp=80:80 $do > web-svc.yaml

# Diff antes de apply
kubectl diff -f updated-manifest.yaml

# Múltiplos formatos de output
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName
```

**Critérios de aceite:**
- [ ] Aliases configurados (`k`, `kn`, `do`, `now`)
- [ ] Autocompletion funcionando
- [ ] Pelo menos 3 manifests gerados via dry-run
- [ ] `kubectl explain` usado para explorar 3+ recursos
- [ ] `kubectl diff` demonstrado
- [ ] Output customizado (jsonpath, custom-columns)

---

## Escopo Técnico

| Conceito | O que dominar |
|---|---|
| **Cluster** | Control Plane (API Server, etcd, Scheduler, Controller Manager), Worker Nodes (kubelet, kube-proxy) |
| **kubectl** | CRUD, dry-run, explain, diff, output formats, jsonpath |
| **Namespaces** | Criação, labels, isolamento lógico, namespace default |
| **Pods** | Lifecycle, multi-container, volumes compartilhados |
| **Labels** | Convenção `app.kubernetes.io/*`, selectors, annotations |
| **API** | Groups (core, apps, batch), versions, CRUD operations |

---

## Critérios de Aceite Globais

- [ ] Cluster local funcional com 4 nodes
- [ ] 6 namespaces criados com labels padronizados
- [ ] Pods simples e multi-container criados e Running
- [ ] Labels e selectors dominados
- [ ] kubectl com aliases, autocompletion e dry-run

---

## Definição de Pronto (DoD)

- [ ] Todos os desafios completados
- [ ] Documento de arquitetura do cluster criado
- [ ] Aliases e autocompletion configurados no shell profile
- [ ] Pelo menos 5 exercícios com `kubectl explain`
- [ ] Commit: `feat(level-0): setup cluster and kubernetes foundations`

---

## Checklist

- [ ] kind instalado e cluster criado
- [ ] kubectl configurado e funcionando
- [ ] k9s, kubectx, kubens instalados
- [ ] Namespaces criados (dev, staging, prod, data, monitoring, logging)
- [ ] Pods criados (simples, multi-container, com labels)
- [ ] Selectors testados (equality e set-based)
- [ ] kubectl aliases e autocompletion configurados
- [ ] Documento de arquitetura escrito
