# CKS — Certified Kubernetes Security Specialist

## Informações do Exame

| Item | Detalhe |
|------|---------|
| **Formato** | Hands-on (terminal Linux) |
| **Duração** | 2 horas |
| **Questões** | 15-20 tarefas |
| **Passing Score** | 67% |
| **Validade** | 2 anos |
| **Pré-requisitos** | CKA ativo (obrigatório) |
| **Preço** | $395 USD (inclui 1 retake + killer.sh) |
| **Ambiente** | PSI Bridge, 1 aba extra para docs incluindo Falco, Trivy, AppArmor |

## Domínios do Exame

| Domínio | Peso | Tópicos |
|---------|:---:|---------|
| **Cluster Setup** | 10% | CIS Benchmark, Network Policies, Ingress TLS, Node security |
| **Cluster Hardening** | 15% | RBAC, Service Accounts, API restrictions, upgrades |
| **System Hardening** | 15% | OS footprint, IAM, Network, seccomp, AppArmor |
| **Minimize Microservice Vulnerabilities** | 20% | PSA/PSS, OPA, Secrets management, runtime sandboxes |
| **Supply Chain Security** | 20% | Image scanning, signing, allowed registries, SBOM |
| **Monitoring, Logging, and Runtime Security** | 20% | Falco, audit logs, immutable containers, behavioral analysis |

## Conteúdo por Domínio

### 1. Cluster Setup (10%)

#### CIS Benchmark
```bash
# kube-bench — verifica CIS Benchmark
kube-bench run --targets=master
kube-bench run --targets=node

# Remediações comuns:
# - API Server: disable anonymous auth, enable audit logging
# - etcd: encrypt at rest, restrict access
# - kubelet: disable anonymous auth, set authorization mode
```

#### Network Policies — Segurança
```yaml
# Default deny ALL em namespace
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

---
# Liberar apenas o necessário
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 443
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: db
      ports:
        - port: 5432
    - to: []
      ports:
        - port: 53
          protocol: UDP
```

#### Ingress com TLS
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - secure.example.com
      secretName: tls-secret
  rules:
    - host: secure.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: secure-svc
                port:
                  number: 443
```

### 2. Cluster Hardening (15%)

#### RBAC Avançado
```yaml
# Princípio do menor privilégio
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: restricted-viewer
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list"]
    # ❌ Nunca use: verbs: ["*"] ou resources: ["*"]
```

```bash
# Auditar RBAC
k auth can-i --list --as system:serviceaccount:production:default
k auth can-i create deployments --as dev-user -n production

# Verificar que service account default não tem permissões
k get rolebindings,clusterrolebindings -A -o wide | grep default
```

#### Service Account Hardening
```yaml
# ✅ Desabilitar automount do token
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
automountServiceAccountToken: false

---
# ✅ No Pod
spec:
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false  # redundante mas explícito
```

#### API Server Hardening
```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
    - command:
        - kube-apiserver
        - --anonymous-auth=false
        - --authorization-mode=Node,RBAC
        - --enable-admission-plugins=NodeRestriction,PodSecurity
        - --audit-log-path=/var/log/kubernetes/audit.log
        - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
        - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
        - --tls-min-version=VersionTLS12
        - --profiling=false
```

### 3. System Hardening (15%)

#### Seccomp
```yaml
# Usar RuntimeDefault
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      securityContext:
        seccompProfile:
          type: RuntimeDefault  # ou Localhost para profile custom
```

```json
// Custom seccomp profile (/var/lib/kubelet/seccomp/profiles/restricted.json)
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "exit", "exit_group", "open", "close",
                "stat", "fstat", "lstat", "poll", "mmap", "mprotect",
                "brk", "ioctl", "access", "pipe", "select", "sched_yield",
                "dup2", "nanosleep", "getpid", "socket", "connect",
                "accept", "sendto", "recvfrom", "bind", "listen",
                "clone", "execve", "wait4", "kill", "fcntl", "sigreturn",
                "arch_prctl", "futex", "epoll_create", "epoll_ctl",
                "epoll_wait", "set_tid_address", "set_robust_list",
                "getrandom"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

#### AppArmor
```bash
# Verificar profiles disponíveis
aa-status
cat /sys/kernel/security/apparmor/profiles

# Carregar profile
apparmor_parser -r /etc/apparmor.d/myprofile
```

```yaml
# Anotar Pod com AppArmor profile
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/myapp: localhost/restricted-profile
```

#### OS Hardening
```bash
# Remover packages desnecessários
apt purge -y netcat socat nmap
apt autoremove -y

# Desabilitar serviços desnecessários
systemctl disable --now snapd
systemctl disable --now avahi-daemon

# Users e privileges
useradd -r -s /usr/sbin/nologin appuser
# Verificar sudoers
visudo  # remover acessos desnecessários

# Fileystem permissions
chmod 600 /etc/kubernetes/manifests/*.yaml
chmod 600 /etc/kubernetes/pki/*.key
```

### 4. Minimize Microservice Vulnerabilities (20%)

#### Pod Security Standards (PSS)
```yaml
# Namespace com PSS Restricted
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

#### Security Context Completo
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
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        privileged: false
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

#### OPA Gatekeeper
```yaml
# ConstraintTemplate
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing labels: %v", [missing])
        }

---
# Constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    labels: ["team", "environment"]
```

#### Secrets Encryption at Rest
```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

```bash
# Aplicar ao API Server
# Adicionar ao kube-apiserver.yaml:
# --encryption-provider-config=/etc/kubernetes/encryption-config.yaml

# Re-encriptar secrets existentes
kubectl get secrets -A -o json | kubectl replace -f -

# Verificar
ETCDCTL_API=3 etcdctl get /registry/secrets/default/mysecret | hexdump -C
```

#### Runtime Sandboxes
| Sandbox | Descrição |
|---------|-----------|
| **gVisor (runsc)** | Kernel userspace, isolação forte |
| **Kata Containers** | VM-based containers |

```yaml
# RuntimeClass para gVisor
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc

---
# Pod usando gVisor
spec:
  runtimeClassName: gvisor
  containers:
    - name: untrusted
      image: untrusted-app:v1
```

### 5. Supply Chain Security (20%)

#### Image Scanning com Trivy
```bash
# Scan de imagem
trivy image nginx:1.25
trivy image --severity HIGH,CRITICAL nginx:1.25
trivy image --exit-code 1 --severity CRITICAL myapp:v1.2.0

# Scan de filesystem/config
trivy fs --security-checks vuln,config .
trivy config /etc/kubernetes/manifests/

# Scan de cluster
trivy k8s --report summary cluster
```

#### Admission Controller para Registries
```yaml
# Kyverno — permitir apenas registries confiáveis
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
              kinds: ["Pod"]
      validate:
        message: "Images must be from approved registries"
        pattern:
          spec:
            containers:
              - image: "registry.company.com/*"
            initContainers:
              - image: "registry.company.com/*"
```

#### Image Signing com Cosign
```bash
# Assinar imagem
cosign sign --key cosign.key registry.company.com/myapp:v1.2.0

# Verificar assinatura
cosign verify --key cosign.pub registry.company.com/myapp:v1.2.0
```

#### ImagePolicyWebhook
```yaml
# Admission Configuration
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: ImagePolicyWebhook
    configuration:
      imagePolicy:
        kubeConfigFile: /etc/kubernetes/image-policy-webhook.kubeconfig
        allowTTL: 50
        denyTTL: 50
        retryBackoff: 500
        defaultAllow: false  # ✅ deny by default
```

### 6. Monitoring, Logging, and Runtime Security (20%)

#### Falco
```yaml
# Falco rule — detectar shell em container
- rule: Terminal shell in container
  desc: Detect a shell spawned in a container
  condition: >
    spawned_process and
    container and
    shell_procs and
    proc.tty != 0
  output: >
    Shell spawned in container
    (user=%user.name container=%container.name
    shell=%proc.name parent=%proc.pname
    image=%container.image.repository)
  priority: WARNING
  tags: [container, shell, mitre_execution]

- rule: Read sensitive files
  desc: Detect reading of sensitive files
  condition: >
    open_read and
    (fd.name startswith /etc/shadow or
     fd.name startswith /etc/passwd) and
    container
  output: >
    Sensitive file read (file=%fd.name container=%container.name image=%container.image.repository)
  priority: CRITICAL
```

```bash
# Falco commands
falco --list  # listar regras
falco -r /etc/falco/falco_rules.yaml  # rodar com regras
# Verificar output
cat /var/log/falco/falco.log
journalctl -u falco
```

#### Audit Logging
```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Não auditar health checks
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: ""
        resources: ["endpoints", "services", "services/status"]
  
  # Auditar secrets com RequestResponse
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
  
  # Auditar tudo mais com Metadata
  - level: Metadata
    omitStages:
      - "RequestReceived"
```

```yaml
# Configurar no kube-apiserver
- --audit-log-path=/var/log/kubernetes/audit.log
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-log-maxage=30
- --audit-log-maxbackup=10
- --audit-log-maxsize=100
```

#### Immutable Containers
```yaml
# Container imutável — filesystem read-only
spec:
  containers:
    - name: app
      securityContext:
        readOnlyRootFilesystem: true
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: var-run
          mountPath: /var/run
  volumes:
    - name: tmp
      emptyDir: {}
    - name: var-run
      emptyDir: {}
```

## Checklist de Segurança CKS

### Cluster
- [ ] CIS Benchmark aplicado (kube-bench)
- [ ] RBAC com menor privilégio
- [ ] Anonymous auth desabilitado
- [ ] Audit logging habilitado
- [ ] Encryption at rest configurado
- [ ] Admission controllers (PSA, OPA/Kyverno)
- [ ] API Server hardening

### Workloads
- [ ] Pod Security Standards (Restricted)
- [ ] Security Context em todos os Pods
- [ ] Service Account com automount disabled
- [ ] Network Policies (default deny)
- [ ] RuntimeClass (gVisor) para untrusted workloads
- [ ] Immutable containers (readOnlyRootFilesystem)

### Supply Chain
- [ ] Image scanning (Trivy)
- [ ] Registry allowlist
- [ ] Image signing (Cosign)
- [ ] SBOM

### Monitoring
- [ ] Falco para runtime detection
- [ ] Audit logs centralizados
- [ ] Alertas para eventos de segurança

## Dicas Específicas para CKS

1. **Requer CKA ativo** — passe no CKA primeiro.
2. **Falco** — saiba analisar e criar regras simples.
3. **Trivy** — saiba escanear imagens e interpretar resultados.
4. **AppArmor/Seccomp** — saiba aplicar profiles.
5. **NetworkPolicy** — default deny + liberação granular.
6. **Audit Policy** — saiba criar e aplicar.
7. **Encryption Provider** — configurar encryption at rest.
8. **PSA/PSS** — labels de namespace para Pod Security.
9. **kube-bench** — interpretar e remediar findings.
10. **Docs permitidas**: falco.org, trivy docs, apparmor docs.

## Recursos de Estudo

- [CKS Curriculum](https://github.com/cncf/curriculum)
- [Kubernetes Security Docs](https://kubernetes.io/docs/concepts/security/)
- KodeKloud CKS Course (Mumshad)
- Udemy CKS Course
- killer.sh (simulador oficial)
- [Falco Documentation](https://falco.org/docs/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
