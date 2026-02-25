# CI/CD & GitOps — Best Practices

## 1. GitOps Principles

### 1.1 Os Quatro Princípios
1. **Declarativo** — Todo o sistema descrito declarativamente.
2. **Versionado e Imutável** — Estado desejado no Git como source of truth.
3. **Automaticamente Aplicado** — Agentes aplicam mudanças automaticamente.
4. **Continuamente Reconciliado** — Agentes garantem convergência contínua.

### 1.2 Push vs Pull Model
```
Push Model (CI/CD Tradicional):
  CI Pipeline → kubectl apply → Cluster
  ⚠️ Requer credenciais do cluster no CI
  ⚠️ Drift detection manual

Pull Model (GitOps):
  Git Repo ← Agent (ArgoCD/Flux) → Cluster
  ✅ Credenciais ficam no cluster
  ✅ Drift detection automático
  ✅ Self-healing
```

## 2. Estrutura de Repositórios

### 2.1 Repo Strategy
```
Opção 1: Monorepo (recomendado para times pequenos)
├── apps/
│   ├── api/
│   │   ├── src/
│   │   ├── Dockerfile
│   │   └── k8s/
│   │       ├── base/
│   │       └── overlays/
│   └── worker/
│       ├── src/
│       ├── Dockerfile
│       └── k8s/
└── infrastructure/
    ├── cluster/
    └── platform/

Opção 2: Multi-repo (recomendado para times grandes)
├── app-repo/          # código da aplicação
├── config-repo/       # manifests K8s (GitOps)
└── infra-repo/        # IaC (Terraform, Pulumi)
```

### 2.2 Kustomize Structure
```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   └── networkpolicy.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── replicas-patch.yaml
    │   └── resources-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replicas-patch.yaml
    └── production/
        ├── kustomization.yaml
        ├── replicas-patch.yaml
        ├── resources-patch.yaml
        └── hpa-patch.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - hpa.yaml
  - networkpolicy.yaml
commonLabels:
  app: myapp

---
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: production
patches:
  - path: replicas-patch.yaml
  - path: resources-patch.yaml
images:
  - name: myapp
    newName: registry.example.com/myapp
    newTag: v1.2.0
```

### 2.3 Helm Chart Structure
```
charts/
├── myapp/
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── values-dev.yaml
│   ├── values-staging.yaml
│   ├── values-production.yaml
│   └── templates/
│       ├── _helpers.tpl
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       ├── hpa.yaml
│       ├── pdb.yaml
│       ├── serviceaccount.yaml
│       ├── networkpolicy.yaml
│       └── tests/
│           └── test-connection.yaml
```

## 3. ArgoCD

### 3.1 Application
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: https://github.com/org/config-repo.git
    targetRevision: main
    path: apps/myapp/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true      # remove recursos deletados do Git
      selfHeal: true   # reconcilia drift
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### 3.2 ApplicationSet (Multi-cluster/Multi-env)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: dev
            url: https://dev-cluster.example.com
            namespace: dev
          - cluster: staging
            url: https://staging-cluster.example.com
            namespace: staging
          - cluster: production
            url: https://prod-cluster.example.com
            namespace: production
  template:
    metadata:
      name: 'myapp-{{cluster}}'
    spec:
      project: '{{cluster}}'
      source:
        repoURL: https://github.com/org/config-repo.git
        targetRevision: main
        path: 'apps/myapp/overlays/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### 3.3 AppProject (RBAC)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production project
  sourceRepos:
    - 'https://github.com/org/config-repo.git'
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
  roles:
    - name: deployer
      policies:
        - p, proj:production:deployer, applications, sync, production/*, allow
      groups:
        - platform-team
```

## 4. Flux CD

### 4.1 GitRepository + Kustomization
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: config-repo
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/org/config-repo.git
  ref:
    branch: main
  secretRef:
    name: git-credentials

---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp-production
  namespace: flux-system
spec:
  interval: 5m
  retryInterval: 2m
  timeout: 3m
  sourceRef:
    kind: GitRepository
    name: config-repo
  path: ./apps/myapp/overlays/production
  prune: true
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: myapp
      namespace: production
```

### 4.2 HelmRelease
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: myapp
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: myapp
      version: "1.2.x"
      sourceRef:
        kind: HelmRepository
        name: internal
        namespace: flux-system
  values:
    replicaCount: 3
    image:
      repository: registry.example.com/myapp
      tag: v1.2.0
  upgrade:
    remediation:
      retries: 3
  rollback:
    cleanupOnFail: true
```

## 5. CI Pipeline Best Practices

### 5.1 Pipeline Stages
```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Build   │──▶│   Test   │──▶│   Scan   │──▶│  Publish │──▶│  Deploy  │
│          │   │          │   │          │   │          │   │ (GitOps) │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
   Docker        Unit         Trivy/Grype    Push to        Update Git
   Build         Integration  SAST/DAST      Registry       Config Repo
                 E2E          License        Sign (Cosign)
```

### 5.2 Image Promotion
```
dev:
  tag: sha-abc1234 (commit SHA)
  auto-deploy on merge to main

staging:
  tag: v1.2.0-rc.1 (release candidate)
  promote from dev after tests pass

production:
  tag: v1.2.0 (semantic version)
  promote from staging after approval
```

```yaml
# GitHub Actions — Update config repo
- name: Update image tag
  run: |
    cd config-repo
    kustomize edit set image myapp=registry.example.com/myapp:v${VERSION}
    git add .
    git commit -m "chore: update myapp to v${VERSION}"
    git push
```

### 5.3 Boas Práticas de CI
- **Immutable tags** — nunca sobrescreva tags de imagem.
- **Sign images** com Cosign/Notary.
- **Scan** em todas as etapas (build, push, runtime).
- **Cache** layers do Docker para builds mais rápidos.
- **Multi-arch builds** quando necessário (arm64/amd64).
- **SBOM** (Software Bill of Materials) em cada imagem.

## 6. Deployment Strategies

### 6.1 Argo Rollouts
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: {duration: 5m}
        - setWeight: 20
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 80
        - pause: {duration: 5m}
      canaryService: myapp-canary
      stableService: myapp-stable
      trafficRouting:
        istio:
          virtualServices:
            - name: myapp-vsvc
              routes:
                - primary
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 2
        args:
          - name: service-name
            value: myapp-canary
```

### 6.2 Blue-Green
```yaml
strategy:
  blueGreen:
    activeService: myapp-active
    previewService: myapp-preview
    autoPromotionEnabled: false
    prePromotionAnalysis:
      templates:
        - templateName: smoke-test
    postPromotionAnalysis:
      templates:
        - templateName: success-rate
```

## 7. Secrets em CI/CD

- **Nunca exponha secrets** em logs de CI.
- Use **OIDC** para autenticação com cloud providers (não static credentials).
- Injete secrets via **vault integration** (HashiCorp Vault, AWS Secrets Manager).
- Use **Sealed Secrets** ou **External Secrets** para secrets no Git.
- Rotacione secrets de CI/CD **automaticamente**.

```yaml
# GitHub Actions — OIDC com AWS (sem credentials estáticos)
permissions:
  id-token: write
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/github-actions
      aws-region: us-east-1
```

## 8. Checklist CI/CD

- [ ] GitOps como modelo principal (push via Git, não kubectl)
- [ ] Imagens imutáveis com tags semânticas
- [ ] Image scanning em pipeline (Trivy, Grype)
- [ ] Image signing (Cosign)
- [ ] SBOM gerado para cada imagem
- [ ] Testes automatizados (unit, integration, e2e)
- [ ] Kustomize ou Helm para ambientes
- [ ] ArgoCD/Flux com auto-sync e self-heal
- [ ] Canary/Blue-Green para deploys críticos
- [ ] Rollback automático com análise de métricas
- [ ] OIDC para autenticação (sem static secrets)
- [ ] Policy enforcement (Kyverno/OPA Gatekeeper)
