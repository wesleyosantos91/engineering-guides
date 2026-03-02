# Level 14 — CKS (Certified Kubernetes Security Specialist)

> **Objetivo:** Preparar-se para a certificação CKS — prova prática avançada focada em
> segurança de clusters Kubernetes, hardening, supply chain e runtime security.

**Referência:** [CKS Certification Guide](../../.docs/k8s/14-cks.md)

---

## Sobre a Certificação

| Campo | Detalhe |
|-------|---------|
| Tipo | Performance-based (hands-on CLI) |
| Duração | 120 minutos |
| Aprovação | 67% |
| Validade | 2 anos |
| Custo | $395 USD (inclui 1 retake) |
| Pré-requisito | **CKA ativa** |

### Domínios

| Domínio | Peso |
|---------|------|
| Cluster Setup | 10% |
| Cluster Hardening | 15% |
| System Hardening | 15% |
| Minimize Microservice Vulnerabilities | 20% |
| Supply Chain Security | 20% |
| Monitoring, Logging and Runtime Security | 20% |

---

## Contexto do Domínio

A CKS é a certificação mais avançada de Kubernetes. Exige conhecimento profundo de segurança em todas as camadas. Use a CloudShop como cenário para implementar defesa em profundidade.

---

## Desafios

### Desafio 14.1 — Cluster Setup (10%)

Pratique setup seguro do cluster.

**Tópicos da prova:**
- Network Policies para isolamento
- CIS Benchmark para Kubernetes
- Ingress com TLS
- Verificar binários do cluster
- Proteger GUI (Dashboard)

**Exercícios no estilo da prova:**
```bash
# Ex 1: CIS Benchmark com kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench

# Ex 2: Network Policy default deny em todos os namespaces
for ns in cloudshop-prod cloudshop-staging cloudshop-dev; do
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: $ns
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
EOF
done

# Ex 3: Ingress com TLS
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: cloudshop-prod
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.cloudshop.local
      secretName: api-tls
  rules:
    - host: api.cloudshop.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
EOF

# Ex 4: Verificar integridade de binários
sha256sum /usr/bin/kubelet
# Comparar com hash oficial do release
curl -sL https://dl.k8s.io/v1.30.0/bin/linux/amd64/kubelet.sha256
```

**Critérios de aceite:**
- [ ] kube-bench executado e resultados analisados
- [ ] Network Policy default-deny em todos os namespaces
- [ ] Ingress com TLS forçado
- [ ] Integridade de binários verificada
- [ ] Dashboard protegido (ou desabilitado em prod)

---

### Desafio 14.2 — Cluster Hardening (15%)

Pratique hardening de componentes do cluster.

**Tópicos da prova:**
- RBAC (menor privilégio)
- Service Accounts restrictions
- Upgrading do cluster
- Restricting API access

**Exercícios no estilo da prova:**
```bash
# Ex 1: Least privilege RBAC
kubectl create role order-service-role \
  --verb=get,list \
  --resource=pods,services,configmaps \
  --namespace=cloudshop-prod

kubectl create rolebinding order-service-binding \
  --role=order-service-role \
  --serviceaccount=cloudshop-prod:order-service-sa \
  --namespace=cloudshop-prod

# Ex 2: Disable automount service account token
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service-sa
  namespace: cloudshop-prod
automountServiceAccountToken: false
EOF

# Ex 3: Restrict anonymous access to API server
# Verificar /etc/kubernetes/manifests/kube-apiserver.yaml
# Deve ter: --anonymous-auth=false

# Ex 4: Restrict API access via RBAC
# Verificar ClusterRoleBindings perigosas
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name == "cluster-admin") | .subjects[]'

# Ex 5: Disable insecure port (deprecated but checked)
# kube-apiserver deve ter: --insecure-port=0 (default em versões recentes)
```

**Critérios de aceite:**
- [ ] RBAC least privilege implementado (< 3 min)
- [ ] automountServiceAccountToken: false
- [ ] Anonymous auth desabilitado
- [ ] cluster-admin bindings auditadas
- [ ] API server flags de segurança verificadas

---

### Desafio 14.3 — System Hardening (15%)

Pratique hardening do sistema operacional e containers.

**Tópicos da prova:**
- AppArmor profiles
- Seccomp profiles
- Reduzir attack surface do host
- Kernel hardening

**Exercícios no estilo da prova:**
```yaml
# Ex 1: Seccomp profile no Pod
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: cloudshop-prod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault  # usar perfil default do runtime
  containers:
    - name: app
      image: registry.cloudshop.io/order-service:v1.2.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        runAsUser: 1000
        capabilities:
          drop: ["ALL"]
```

```bash
# Ex 2: AppArmor
# Verificar profiles disponíveis
cat /sys/kernel/security/apparmor/profiles

# Aplicar profile no Pod via annotation
# metadata.annotations:
#   container.apparmor.security.beta.kubernetes.io/app: runtime/default

# Ex 3: Reduzir attack surface
# Verificar serviços desnecessários no node
systemctl list-units --type=service --state=running
# Desabilitar serviços não necessários
systemctl disable --now snapd

# Ex 4: Verificar processos rodando como root
ps aux | grep -v "^root" | head
# Containers devem rodar como non-root
kubectl get pods -n cloudshop-prod -o json | \
  jq '.items[].spec.containers[].securityContext.runAsNonRoot'
```

**Critérios de aceite:**
- [ ] Seccomp RuntimeDefault em todos os Pods
- [ ] runAsNonRoot: true em todos os containers
- [ ] readOnlyRootFilesystem: true
- [ ] capabilities: drop ALL
- [ ] Serviços desnecessários desabilitados no host

---

### Desafio 14.4 — Microservice Vulnerabilities (20%)

Pratique mitigação de vulnerabilidades em microsserviços.

**Tópicos da prova:**
- Pod Security Standards (PSS) / Pod Security Admission (PSA)
- SecurityContext em profundidade
- Secrets management
- Sandbox runtimes (gVisor/Kata)

**Exercícios no estilo da prova:**
```bash
# Ex 1: Enforce PSS Restricted no namespace
kubectl label namespace cloudshop-prod \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Ex 2: Testar — Pod privileged deve ser rejeitado
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  namespace: cloudshop-prod
spec:
  containers:
    - name: bad
      image: nginx
      securityContext:
        privileged: true
EOF
# DEVE ser rejeitado!

# Ex 3: Secrets — verificar não estão em env
kubectl get pods -n cloudshop-prod -o json | \
  jq '.items[].spec.containers[].env[]? | select(.name | test("PASSWORD|SECRET")) | select(.value != null)'
# Se retornar algo → vulnerabilidade!

# Ex 4: Completar SecurityContext para compliance
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
  namespace: cloudshop-prod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534
    fsGroup: 65534
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: registry.cloudshop.io/order-service:v1.2.0
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
          cpu: 500m
          memory: 512Mi
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir:
        medium: Memory
        sizeLimit: 64Mi
EOF
```

**Critérios de aceite:**
- [ ] PSS Restricted enforced no namespace prod
- [ ] Pod privileged rejeitado
- [ ] Nenhum secret em plain text em env vars
- [ ] SecurityContext completo e correto
- [ ] readOnlyRootFilesystem + emptyDir para /tmp

---

### Desafio 14.5 — Supply Chain Security (20%)

Pratique segurança da cadeia de supply de imagens.

**Tópicos da prova:**
- Image scanning (Trivy)
- Whitelist de registries
- Image signing e verification
- Admission controllers (OPA/Kyverno)

**Exercícios no estilo da prova:**
```bash
# Ex 1: Scan de imagens com Trivy
trivy image nginx:1.25
trivy image registry.cloudshop.io/order-service:v1.2.0
# Verificar CVEs critical/high

# Ex 2: Kyverno — whitelist de registries
cat <<EOF | kubectl apply -f -
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
        message: "Imagens devem vir de registries aprovados"
        pattern:
          spec:
            containers:
              - image: "registry.cloudshop.io/* | docker.io/library/*"
            initContainers:
              - image: "registry.cloudshop.io/* | docker.io/library/*"
EOF

# Ex 3: Deny images sem digest
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-digest
spec:
  validationFailureAction: Audit
  rules:
    - name: check-digest
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Imagens devem usar digest (@sha256:) em produção"
        pattern:
          spec:
            containers:
              - image: "*@sha256:*"
EOF

# Ex 4: ImagePolicyWebhook (admission controller)
# Configurar no kube-apiserver:
# --enable-admission-plugins=...,ImagePolicyWebhook
# --admission-control-config-file=/etc/kubernetes/admission/config.yaml
```

**Critérios de aceite:**
- [ ] Trivy scan executado em imagens da CloudShop
- [ ] CVEs críticas identificadas
- [ ] Registry whitelist via Kyverno
- [ ] Image sem tag aprovada rejeitada
- [ ] Admission controller para images avaliado

---

### Desafio 14.6 — Runtime Security (20%)

Pratique segurança em tempo de execução.

**Tópicos da prova:**
- Falco para detecção de ameaças runtime
- Audit logging
- Imutabilidade de containers
- Incident response

**Exercícios no estilo da prova:**
```bash
# Ex 1: Instalar Falco
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set driver.kind=modern_ebpf

# Ex 2: Verificar alertas Falco
kubectl logs -l app.kubernetes.io/name=falco -n falco --tail=20

# Ex 3: Simular atividade suspeita
kubectl exec -it order-service-xxx -n cloudshop-prod -- sh -c "cat /etc/shadow"
# Falco deve alertar: "Sensitive file opened for reading"

kubectl exec -it order-service-xxx -n cloudshop-prod -- sh -c "apt-get update"
# Falco deve alertar: "Package management process launched"

# Ex 4: Audit logging
# Configurar audit policy no kube-apiserver
cat <<EOF > /etc/kubernetes/audit/policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods"]
    namespaces: ["cloudshop-prod"]
  - level: None
    resources:
      - group: ""
        resources: ["configmaps"]
        resourceNames: ["controller-leader"]
  - level: Metadata
    omitStages:
      - RequestReceived
EOF

# kube-apiserver flags:
# --audit-policy-file=/etc/kubernetes/audit/policy.yaml
# --audit-log-path=/var/log/kubernetes/audit.log
# --audit-log-maxage=30
# --audit-log-maxbackup=10

# Ex 5: Verificar audit logs
cat /var/log/kubernetes/audit.log | jq '.verb, .objectRef.resource, .user.username'
```

**Critérios de aceite:**
- [ ] Falco instalado e gerando alertas
- [ ] Atividade suspeita detectada (exec em container, leitura /etc/shadow)
- [ ] Audit policy configurada
- [ ] Audit logs verificados
- [ ] Processo de incident response documentado

---

## Definição de Pronto (DoD)

- [ ] 6 domínios praticados com exercícios hands-on
- [ ] CIS Benchmark executado
- [ ] PSS Restricted enforced
- [ ] Supply chain security implementado
- [ ] Falco para runtime security
- [ ] Simulado com score ≥ 67%

---

## Checklist

- [ ] Cluster Setup — CIS benchmark, network policies, TLS
- [ ] Cluster Hardening — RBAC, SA, API restrictions
- [ ] System Hardening — seccomp, AppArmor, capabilities
- [ ] Microservice Vulns — PSS, SecurityContext, secrets
- [ ] Supply Chain — Trivy, registry whitelist, admission
- [ ] Runtime Security — Falco, audit logs
- [ ] 2+ simulados com score ≥ 67%
