# Level 6 — CI/CD & GitOps

> **Objetivo:** Implementar pipeline CI/CD e GitOps para a plataforma CloudShop usando
> Kustomize, Helm, ArgoCD e Image Automation para deployments declarativos e auditáveis.

**Referência:** [CI/CD & GitOps Best Practices](../../.docs/k8s/06-cicd-gitops.md)

---

## Objetivo de Aprendizado

- Estruturar manifests com Kustomize (base + overlays)
- Criar Helm Charts para microsserviços
- Instalar e configurar ArgoCD para GitOps
- Implementar ApplicationSet para multi-environment
- Configurar Image Automation (auto-update)
- Implementar promotion pipeline (dev → staging → prod)

---

## Contexto do Domínio

A plataforma CloudShop precisa de deployments automatizados, auditáveis e reprodutíveis em 3 ambientes (dev, staging, prod). Um push no repositório Git deve ser a única forma de alterar o estado do cluster (GitOps).

---

## Desafios

### Desafio 6.1 — Kustomize (Base + Overlays)

Estruture os manifests da CloudShop usando Kustomize com base e overlays por ambiente.

**Requisitos:**
- Base com resources comuns (Deployment, Service, ConfigMap)
- Overlays para dev, staging e prod
- Patches para réplicas, resources, imagens e labels
- Strategic merge patches e JSON patches

```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── replica-patch.yaml
    │   └── resources-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── prod/
        ├── kustomization.yaml
        ├── replica-patch.yaml
        ├── resources-patch.yaml
        └── hpa.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
commonLabels:
  app.kubernetes.io/part-of: cloudshop
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
---
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: cloudshop-prod
namePrefix: ""
resources:
  - ../../base
  - hpa.yaml
patches:
  - path: replica-patch.yaml
  - path: resources-patch.yaml
images:
  - name: order-service
    newName: registry.cloudshop.io/order-service
    newTag: v1.2.0
---
# overlays/prod/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
```

```bash
# Preview
kubectl kustomize overlays/dev
kubectl kustomize overlays/prod

# Apply
kubectl apply -k overlays/dev
kubectl apply -k overlays/prod

# Diff
kubectl diff -k overlays/prod
```

**Critérios de aceite:**
- [ ] Base com Deployment + Service + ConfigMap para order-service
- [ ] Overlays para dev (1 réplica), staging (2), prod (3)
- [ ] Resources diferentes por ambiente
- [ ] `kubectl kustomize` gera YAML válido por overlay
- [ ] `kubectl diff -k` mostra diferenças antes de aplicar

---

### Desafio 6.2 — Helm Charts

Crie um Helm Chart para os microsserviços da CloudShop.

**Requisitos:**
- Chart genérico reutilizável para todos os microsserviços
- values.yaml com defaults sensatos
- values por ambiente (values-dev.yaml, values-prod.yaml)
- Helpers para labels, selectors e nomes
- Testes de template

```
charts/
└── microservice/
    ├── Chart.yaml
    ├── values.yaml
    ├── values-dev.yaml
    ├── values-prod.yaml
    ├── templates/
    │   ├── _helpers.tpl
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── hpa.yaml
    │   ├── pdb.yaml
    │   └── servicemonitor.yaml
    └── tests/
        └── test-connection.yaml
```

```yaml
# Chart.yaml
apiVersion: v2
name: microservice
description: CloudShop generic microservice chart
version: 0.1.0
appVersion: "1.0.0"
---
# values.yaml (defaults)
replicaCount: 1
image:
  repository: ""
  tag: latest
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPU: 70
probes:
  liveness:
    path: /health
    initialDelaySeconds: 10
  readiness:
    path: /ready
    initialDelaySeconds: 5
```

```bash
# Validate
helm lint charts/microservice

# Template render
helm template order-service charts/microservice \
  -f charts/microservice/values-prod.yaml \
  --set image.repository=registry.cloudshop.io/order-service \
  --set image.tag=v1.2.0

# Install
helm install order-service charts/microservice \
  -n cloudshop-prod \
  -f charts/microservice/values-prod.yaml

# Test
helm test order-service -n cloudshop-prod
```

**Critérios de aceite:**
- [ ] Chart genérico funciona para qualquer microsserviço
- [ ] `helm lint` sem erros
- [ ] `helm template` gera YAML válido
- [ ] values-dev e values-prod com configurações diferentes
- [ ] `helm test` passa

---

### Desafio 6.3 — ArgoCD Setup

Instale ArgoCD e configure GitOps para a CloudShop.

**Requisitos:**
- Instalar ArgoCD no namespace `argocd`
- Configurar repositório Git
- Criar AppProject para CloudShop
- Criar Application para cada microsserviço

```yaml
# AppProject
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: cloudshop
  namespace: argocd
spec:
  description: CloudShop Platform
  sourceRepos:
    - "https://github.com/org/cloudshop-k8s.git"
  destinations:
    - namespace: "cloudshop-*"
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
  roles:
    - name: developer
      description: Developer access
      policies:
        - p, proj:cloudshop:developer, applications, get, cloudshop/*, allow
        - p, proj:cloudshop:developer, applications, sync, cloudshop/*, allow
---
# Application — order-service
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: cloudshop
  source:
    repoURL: https://github.com/org/cloudshop-k8s.git
    targetRevision: main
    path: overlays/prod/order-service
  destination:
    server: https://kubernetes.default.svc
    namespace: cloudshop-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

```bash
# Instalar ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Obter senha admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# CLI
argocd login localhost:8080
argocd app list
argocd app sync order-service-prod
```

**Critérios de aceite:**
- [ ] ArgoCD instalado e acessível (UI + CLI)
- [ ] AppProject criado com source/destination corretos
- [ ] Application para order-service com auto-sync
- [ ] selfHeal ativo (alteração manual é revertida)
- [ ] Sync status "Healthy + Synced"

---

### Desafio 6.4 — ApplicationSet (Multi-Environment)

Use ApplicationSet para gerar Applications automaticamente para todos os microsserviços e ambientes.

**Requisitos:**
- ApplicationSet com generator Matrix (serviço × ambiente)
- Geração automática: 6 serviços × 3 ambientes = 18 Applications
- Labels e annotations para organização

```yaml
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
                  revision: develop
                - env: staging
                  namespace: cloudshop-staging
                  revision: release
                - env: prod
                  namespace: cloudshop-prod
                  revision: main
  template:
    metadata:
      name: "{{service}}-{{env}}"
      namespace: argocd
      labels:
        app.kubernetes.io/part-of: cloudshop
        environment: "{{env}}"
    spec:
      project: cloudshop
      source:
        repoURL: https://github.com/org/cloudshop-k8s.git
        targetRevision: "{{revision}}"
        path: "overlays/{{env}}/{{service}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

**Critérios de aceite:**
- [ ] ApplicationSet criado com Matrix generator
- [ ] 18 Applications geradas automaticamente (6 × 3)
- [ ] Cada Application aponta para branch correto
- [ ] Novo serviço adicionado à lista gera 3 Applications
- [ ] ArgoCD UI mostra todas as Applications organizadas

---

### Desafio 6.5 — Image Automation

Configure auto-update de imagens quando uma nova versão é publicada.

**Requisitos:**
- ArgoCD Image Updater (ou Flux Image Automation)
- Detectar novas tags no registry
- Atualizar automaticamente kustomization ou values
- Commit automático no Git

```yaml
# ArgoCD Image Updater — annotation na Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-prod
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: "order=registry.cloudshop.io/order-service"
    argocd-image-updater.argoproj.io/order.update-strategy: semver
    argocd-image-updater.argoproj.io/order.allow-tags: "regexp:^v[0-9]+\\.[0-9]+\\.[0-9]+$"
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
spec:
  # ... (mesma spec de antes)
```

```bash
# Instalar ArgoCD Image Updater
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

# Verificar logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater

# Simular: push nova imagem → image updater detecta → commit no Git → ArgoCD sync
```

**Critérios de aceite:**
- [ ] ArgoCD Image Updater instalado
- [ ] Annotations de image update na Application
- [ ] Strategy semver configurada
- [ ] Allow-tags regexp valida tags semânticas
- [ ] Write-back via Git commit (não imperativo)

---

### Desafio 6.6 — Promotion Pipeline

Implemente pipeline de promoção: dev → staging → prod.

**Requisitos:**
- dev: auto-sync, deploy em push
- staging: auto-sync, deploy após merge em release branch
- prod: manual sync, requer aprovação

```yaml
# dev — auto deploy
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-dev
spec:
  source:
    targetRevision: develop
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
# staging — auto deploy (release branch)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-staging
spec:
  source:
    targetRevision: release
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
# prod — MANUAL sync
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-prod
spec:
  source:
    targetRevision: main
  syncPolicy:
    automated: null  # MANUAL — requer aprovação
    syncOptions:
      - ApplyOutOfSyncOnly=true
```

```bash
# Fluxo de promoção
# 1. Developer merge em develop → auto-deploy em dev
# 2. Team merge develop → release → auto-deploy em staging
# 3. QA valida staging
# 4. Team merge release → main
# 5. argocd app sync order-service-prod  # manual
```

**Critérios de aceite:**
- [ ] dev: auto-sync ao push em develop
- [ ] staging: auto-sync ao push em release
- [ ] prod: manual sync apenas
- [ ] Pipeline documentado (fluxo de branches)
- [ ] Rollback funciona: `argocd app rollback order-service-prod`

---

## Definição de Pronto (DoD)

- [ ] Kustomize base + overlays para 3 ambientes
- [ ] Helm Chart genérico para microsserviços
- [ ] ArgoCD instalado com AppProject
- [ ] ApplicationSet gerando 18 Applications
- [ ] Image Automation configurada
- [ ] Promotion pipeline (dev → staging → prod)
- [ ] Commit: `feat(level-6): add cicd gitops`

---

## Checklist

- [ ] Kustomize base + overlays (dev, staging, prod)
- [ ] Helm Chart com values por ambiente
- [ ] ArgoCD instalado e funcional
- [ ] AppProject com RBAC
- [ ] Application com auto-sync e selfHeal
- [ ] ApplicationSet Matrix (serviço × ambiente)
- [ ] Image Updater configurado
- [ ] Promotion pipeline documentado
- [ ] Rollback testado
