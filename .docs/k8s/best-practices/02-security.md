# Security — Best Practices

## 1. RBAC (Role-Based Access Control)

### 1.1 Princípio do Menor Privilégio
```yaml
# ✅ BOM — Role específica com permissões mínimas
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: app-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: production
  name: dev-team-read
subjects:
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: app-reader
  apiGroup: rbac.authorization.k8s.io
```

**Regras:**
- **Nunca use `cluster-admin`** para workloads ou usuários comuns.
- Prefira **Role/RoleBinding** (namespace-scoped) sobre ClusterRole/ClusterRoleBinding.
- Use **Groups** em vez de Users para facilitar gerenciamento.
- Audite RBAC regularmente com `kubectl auth can-i --list`.
- Evite wildcards (`*`) em verbs e resources.

### 1.2 Service Accounts
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
  annotations:
    # AWS EKS — IRSA
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/myapp-role
    # GKE — Workload Identity
    iam.gke.io/gcp-service-account: myapp@project.iam.gserviceaccount.com
automountServiceAccountToken: false  # ✅ desabilitar se não necessário
```

**Boas Práticas:**
- Crie **ServiceAccounts dedicados** por aplicação.
- **Desabilite `automountServiceAccountToken`** quando não necessário.
- Use **Workload Identity** (GKE) ou **IRSA** (EKS) para acesso a cloud APIs.
- Nunca use o ServiceAccount `default`.

## 2. Pod Security

### 2.1 Pod Security Standards (PSS)
Kubernetes define três níveis de segurança:

| Nível | Descrição |
|-------|-----------|
| **Privileged** | Sem restrições (apenas para system workloads) |
| **Baseline** | Restrições mínimas (previne escalações conhecidas) |
| **Restricted** | Hardening completo (recomendado para produção) |

```yaml
# Aplicar via labels no namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 2.2 Security Context (Pod-Level)
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: myapp
      image: myapp:v1.2.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
          # add: ["NET_BIND_SERVICE"]  # apenas se necessário
        privileged: false
```

**Checklist de Security Context:**
- [ ] `runAsNonRoot: true`
- [ ] `readOnlyRootFilesystem: true`
- [ ] `allowPrivilegeEscalation: false`
- [ ] `capabilities.drop: ["ALL"]`
- [ ] `privileged: false`
- [ ] `seccompProfile.type: RuntimeDefault`

### 2.3 Volumes Temporários para Read-Only Filesystem
```yaml
spec:
  containers:
    - name: myapp
      securityContext:
        readOnlyRootFilesystem: true
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

## 3. Network Policies

### 3.1 Default Deny
```yaml
# ✅ SEMPRE aplique default deny por namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### 3.2 Permitir Tráfego Específico
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-traffic
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
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
          protocol: TCP
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
          protocol: TCP
    - to:  # permitir DNS
        - namespaceSelector: {}
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```

**Boas Práticas:**
- Sempre comece com **default deny** em todos os namespaces.
- Libere tráfego de forma **explícita e granular**.
- Não esqueça de liberar **DNS (porta 53)** no egress.
- Use **Calico ou Cilium** para Network Policies avançadas (L7).
- Teste policies com ferramentas como **netpol-viewer** ou **kubectl-np-viewer**.

## 4. Secrets Management

### 4.1 Nunca em Plain Text
```yaml
# ❌ RUIM — Secret no manifesto YAML
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: bXlwYXNzd29yZA==  # apenas base64, NÃO é encriptação

# ✅ BOM — Use External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
    - secretKey: password
      remoteRef:
        key: prod/db/password
```

### 4.2 Soluções Recomendadas
| Solução | Provider | Descrição |
|---------|----------|-----------|
| **External Secrets Operator** | Multi-cloud | Sincroniza secrets de vaults externos |
| **Sealed Secrets** | Bitnami | Encripta secrets para armazenar no Git |
| **HashiCorp Vault** | Multi-cloud | Vault completo com rotação automática |
| **AWS Secrets Manager + CSI** | AWS | Mount secrets como volumes |
| **Azure Key Vault** | Azure | Integração nativa com AKS |
| **GCP Secret Manager** | GCP | Integração nativa com GKE |

### 4.3 Boas Práticas de Secrets
- **Nunca commite secrets** no Git (use `.gitignore`, pre-commit hooks).
- Use **encriptação at rest** no etcd (`EncryptionConfiguration`).
- Implemente **rotação automática** de secrets.
- Use **CSI Secret Store Driver** para mount como volume (mais seguro que env vars).
- Env vars com secrets podem aparecer em logs de crash, `docker inspect`, `/proc`.

## 5. Supply Chain Security

### 5.1 Image Scanning
```yaml
# Pipeline de CI — exemplo
stages:
  - build:
      - docker build -t myapp:${VERSION} .
  - scan:
      - trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:${VERSION}
      - grype myapp:${VERSION} --fail-on high
  - sign:
      - cosign sign --key cosign.key myapp:${VERSION}
  - push:
      - docker push myapp:${VERSION}
```

### 5.2 Admission Controllers
```yaml
# Kyverno — Bloquear imagens sem tag ou com latest
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
        message: "Image tag 'latest' is not allowed."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

### 5.3 Ferramentas de Policy Engine
| Ferramenta | Tipo | Descrição |
|-----------|------|-----------|
| **Kyverno** | Policy Engine | Políticas nativas K8s (YAML) |
| **OPA Gatekeeper** | Policy Engine | Políticas em Rego |
| **Kubewarden** | Policy Engine | Policies em WebAssembly |
| **Cosign** | Image Signing | Assinatura de imagens (Sigstore) |
| **Notary v2** | Image Signing | Assinatura de artefatos OCI |

## 6. Audit Logging

### 6.1 Audit Policy
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Não logar health checks
  - level: None
    resources:
      - group: ""
        resources: ["endpoints", "services", "services/status"]
    users: ["system:kube-proxy"]
  
  # Log completo para secrets
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
  
  # Log de metadata para tudo mais
  - level: Metadata
    resources:
      - group: ""
      - group: "apps"
      - group: "batch"
```

### 6.2 Runtime Security
| Ferramenta | Descrição |
|-----------|-----------|
| **Falco** | Detecção de ameaças em runtime |
| **Tetragon** | Observabilidade e enforcement eBPF |
| **KubeArmor** | Runtime security enforcement |
| **Tracee** | Detecção baseada em eBPF |

## 7. Checklist de Segurança

### Cluster
- [ ] API Server com autenticação forte (OIDC)
- [ ] RBAC habilitado e configurado
- [ ] Audit logging ativado
- [ ] Etcd encriptado (at rest e in transit)
- [ ] Admission controllers configurados
- [ ] Versão do Kubernetes atualizada

### Workloads
- [ ] Pod Security Standards aplicados
- [ ] Security Context configurado em todos os Pods
- [ ] Network Policies aplicadas
- [ ] Secrets gerenciados externamente
- [ ] Imagens escaneadas e assinadas
- [ ] Service Accounts dedicados
- [ ] Resource Limits definidos

### Network
- [ ] Default deny em todos os namespaces
- [ ] TLS para toda comunicação (mTLS com service mesh)
- [ ] Ingress com TLS terminado
- [ ] Egress controlado
- [ ] DNS protegido
