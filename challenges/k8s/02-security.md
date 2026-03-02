# Level 2 — Security

> **Objetivo:** Implementar segurança completa na plataforma CloudShop — RBAC, Pod Security Standards,
> Network Policies, Secrets management e supply chain security.

**Referência:** [Security Best Practices](../../.docs/k8s/02-security.md) · [CKS](../../.docs/k8s/14-cks.md)

---

## Objetivo de Aprendizado

- Implementar RBAC com princípio do menor privilégio
- Aplicar Pod Security Standards (Restricted) em todos os namespaces
- Configurar Security Context em todos os Pods
- Criar Network Policies com default deny
- Gerenciar Secrets com External Secrets Operator
- Implementar supply chain security (scanning, signing, admission control)

---

## Contexto do Domínio

A plataforma CloudShop precisa de segurança em camadas: cada time (frontend, backend, data, platform) tem permissões restritas via RBAC; todos os Pods rodam como non-root com filesystem read-only; e o tráfego entre namespaces é controlado por Network Policies.

---

## Desafios

### Desafio 2.1 — RBAC por Time

Crie Roles e RoleBindings para 3 times com permissões diferenciadas.

**Requisitos:**
- Time `backend`: get, list, watch, create, update em pods, deployments, services no namespace `cloudshop-prod`
- Time `data`: get, list, watch em pods; exec em pods do namespace `cloudshop-data`
- Time `platform`: cluster-admin nos namespaces de monitoring/logging; read-only em todos os namespaces
- Nunca usar ClusterRole `cluster-admin` para times

```yaml
# Role para time backend
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: cloudshop-prod
  name: backend-developer
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: cloudshop-prod
  name: backend-team-binding
subjects:
  - kind: Group
    name: backend-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: backend-developer
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Verificar permissões
kubectl auth can-i list pods --as-group=backend-team --as=user1 -n cloudshop-prod
kubectl auth can-i delete deployments --as-group=backend-team --as=user1 -n cloudshop-prod  # deve ser "no"
kubectl auth can-i create pods --as-group=data-team --as=user2 -n cloudshop-data  # deve ser "no"
```

**Critérios de aceite:**
- [ ] 3 Roles criadas (backend, data, platform)
- [ ] RoleBindings associam Groups (não Users individuais)
- [ ] Sem wildcards (`*`) em verbs ou resources
- [ ] `kubectl auth can-i` confirma permissões corretas
- [ ] Time data NÃO pode criar/deletar pods

---

### Desafio 2.2 — Service Accounts

Crie ServiceAccounts dedicados para cada microsserviço.

**Requisitos:**
- Um ServiceAccount por microsserviço
- `automountServiceAccountToken: false` por padrão
- Montar token apenas quando necessário (ex: service que precisa acessar API)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service-sa
  namespace: cloudshop-prod
  labels:
    app.kubernetes.io/name: order-service
automountServiceAccountToken: false
---
# Deployment referenciando o SA
spec:
  template:
    spec:
      serviceAccountName: order-service-sa
      automountServiceAccountToken: false
```

**Critérios de aceite:**
- [ ] ServiceAccount dedicado para cada microsserviço
- [ ] Token NÃO montado automaticamente
- [ ] Nenhum workload usa o ServiceAccount `default`
- [ ] ServiceAccounts com labels padronizados

---

### Desafio 2.3 — Pod Security Standards

Aplique Pod Security Standards (Restricted) em todos os namespaces de produção.

**Requisitos:**
- Enforce `restricted` no namespace `cloudshop-prod`
- Warn + audit `restricted` nos namespaces de staging e dev
- Security context completo em todos os Pods

```yaml
# Namespace com PSS
apiVersion: v1
kind: Namespace
metadata:
  name: cloudshop-prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Pod com security context compliant
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: order-service
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
        privileged: false
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
  volumes:
    - name: tmp
      emptyDir:
        sizeLimit: 100Mi
    - name: cache
      emptyDir:
        sizeLimit: 500Mi
```

**Critérios de aceite:**
- [ ] PSS `restricted` enforced em `cloudshop-prod`
- [ ] PSS `restricted` warn/audit em staging e dev
- [ ] Todos os Pods com security context completo:
  - [ ] `runAsNonRoot: true`
  - [ ] `readOnlyRootFilesystem: true`
  - [ ] `allowPrivilegeEscalation: false`
  - [ ] `capabilities.drop: ["ALL"]`
  - [ ] `seccompProfile.type: RuntimeDefault`
- [ ] EmptyDir para diretórios temporários (substituindo filesystem writable)
- [ ] Pods que violam a política são rejeitados

---

### Desafio 2.4 — Network Policies

Implemente default deny e permitir tráfego específico entre serviços.

**Requisitos:**
- Default deny (ingress + egress) em todos os namespaces
- Permitir API services → PostgreSQL (5432)
- Permitir API services → Redis (6379)
- Permitir Ingress Controller → API services (8080)
- Permitir DNS (53 UDP/TCP) no egress

```yaml
# Default deny all
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
# Permitir order-service → PostgreSQL
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-to-postgres
  namespace: cloudshop-prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: order-service
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              app.kubernetes.io/part-of: cloudshop
          podSelector:
            matchLabels:
              app: postgresql
      ports:
        - port: 5432
          protocol: TCP
    - to: []  # DNS
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
---
# Permitir Ingress → order-service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-to-order
  namespace: cloudshop-prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: order-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - port: 8080
          protocol: TCP
```

**Critérios de aceite:**
- [ ] Default deny aplicado em TODOS os namespaces
- [ ] Tráfego entre serviços explicitamente permitido
- [ ] DNS (porta 53) permitido no egress
- [ ] Comunicação cross-namespace controlada
- [ ] Teste com `kubectl exec` verifica que tráfego bloqueado é negado
- [ ] Teste verifica que tráfego permitido funciona

---

### Desafio 2.5 — Secrets Management

Implemente gestão segura de secrets com External Secrets Operator.

**Requisitos:**
- External Secrets Operator instalado
- Secrets de banco de dados gerenciados externamente
- Secrets montados como volume (não env vars)
- Sem secrets em plain text no Git

```yaml
# ExternalSecret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: cloudshop-prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: cluster-secret-store
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: DB_HOST
      remoteRef:
        key: prod/cloudshop/db
        property: host
    - secretKey: DB_PASSWORD
      remoteRef:
        key: prod/cloudshop/db
        property: password
---
# Secret como volume (mais seguro que env var)
spec:
  containers:
    - name: order-service
      volumeMounts:
        - name: db-creds
          mountPath: /etc/secrets/db
          readOnly: true
  volumes:
    - name: db-creds
      secret:
        secretName: db-credentials
        defaultMode: 0400
```

**Critérios de aceite:**
- [ ] External Secrets Operator instalado e funcionando
- [ ] Secrets sincronizados de vault externo
- [ ] Secrets montados como volume com permissões `0400`
- [ ] Nenhum secret hardcoded nos manifests
- [ ] `refreshInterval` configurado para rotação

---

### Desafio 2.6 — Supply Chain Security

Implemente controles de supply chain: scanning, signing e admission.

**Requisitos:**
- Scanning de imagens com Trivy
- Política Kyverno para bloquear imagens com tag `latest`
- Política Kyverno para exigir registry confiável
- Image pull policy `Always` ou tag imutável

```yaml
# Kyverno — bloquear latest
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-image-tag
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Image tag 'latest' is not allowed. Use a specific version tag."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
---
# Kyverno — exigir registry confiável
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-registries
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Images must come from trusted registries."
        pattern:
          spec:
            containers:
              - image: "registry.example.com/* | ghcr.io/org/*"
```

```bash
# Scanning com Trivy
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp/order-service:v1.0.0
trivy image --format json --output results.json myapp/order-service:v1.0.0
```

**Critérios de aceite:**
- [ ] Kyverno instalado e políticas aplicadas
- [ ] Pod com `:latest` é rejeitado
- [ ] Pod de registry não confiável é rejeitado
- [ ] Pipeline de CI inclui scanning com Trivy
- [ ] Imagens usam tags imutáveis (semver ou SHA digest)

---

## Definição de Pronto (DoD)

- [ ] RBAC configurado para 3 times
- [ ] ServiceAccounts dedicados por microsserviço
- [ ] PSS Restricted enforced em produção
- [ ] Network Policies com default deny + allow explícito
- [ ] External Secrets Operator gerenciando secrets
- [ ] Kyverno com políticas de admission
- [ ] Commit: `feat(level-2): add security hardening`

---

## Checklist

- [ ] 3 Roles + RoleBindings criados
- [ ] ServiceAccounts criados (sem token automount)
- [ ] PSS labels aplicados nos namespaces
- [ ] Security context em TODOS os pods
- [ ] Default deny NetworkPolicy em TODOS os namespaces
- [ ] Allow policies para tráfego necessário
- [ ] External Secrets Operator instalado
- [ ] Secrets montados como volume
- [ ] Kyverno instalado com 2+ políticas
- [ ] Trivy scanning no pipeline
- [ ] `kubectl auth can-i` verificado para cada time
