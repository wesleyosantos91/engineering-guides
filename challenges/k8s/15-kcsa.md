# Level 15 — KCSA (Kubernetes and Cloud Native Security Associate)

> **Objetivo:** Preparar-se para a certificação KCSA — prova de múltipla escolha que valida
> conhecimentos de segurança em ambientes Kubernetes e Cloud Native.

**Referência:** [KCSA Certification Guide](../../.docs/k8s/15-kcsa.md)

---

## Sobre a Certificação

| Campo | Detalhe |
|-------|---------|
| Tipo | Múltipla escolha (60 questões) |
| Duração | 90 minutos |
| Aprovação | 75% |
| Validade | 3 anos |
| Custo | $250 USD |
| Pré-requisito | Nenhum (recomendado: KCNA + experiência com segurança) |

### Domínios

| Domínio | Peso |
|---------|------|
| Overview of Cloud Native Security | 14% |
| Kubernetes Cluster Component Security | 22% |
| Kubernetes Security Fundamentals | 22% |
| Kubernetes Threat Model | 16% |
| Platform Security | 16% |
| Compliance and Security Frameworks | 10% |

---

## Contexto do Domínio

A KCSA é a certificação de segurança conceitual (diferente da CKS que é hands-on). O foco é entender ameaças, frameworks de segurança, melhores práticas e como proteger cada camada do stack Kubernetes.

---

## Desafios

### Desafio 15.1 — Cloud Native Security Overview (14%)

Entenda os princípios de segurança cloud native.

**Tópicos para estudo:**
- 4 Cs da segurança: Cloud, Cluster, Container, Code
- Defense in Depth
- Zero Trust Architecture
- Least Privilege
- CNCF security projects

**Exercícios:**
```markdown
## Quiz — Cloud Native Security

1. Explique os 4 Cs da segurança Cloud Native:
   - Cloud: segurança da infraestrutura (IAM, VPC, encryption at rest)
   - Cluster: segurança do K8s (RBAC, PSS, Network Policies)
   - Container: segurança de imagens (scanning, non-root, readonly)
   - Code: segurança do código (SAST, DAST, dependency scanning)

2. O que é Defense in Depth? Como se aplica ao CloudShop?

3. O que é Zero Trust? Cite 3 implementações no Kubernetes:
   - mTLS via Service Mesh
   - Network Policies (deny by default)
   - RBAC (least privilege)

4. Cite 5 projetos CNCF focados em segurança:
   - Falco: runtime threat detection
   - OPA/Gatekeeper: policy enforcement
   - Notary/TUF: image signing
   - cert-manager: certificate management
   - SPIFFE/SPIRE: identity framework

5. O que é shift-left security? Como implementar?
```

**Critérios de aceite:**
- [ ] 4 Cs explicados com exemplos para CloudShop
- [ ] Defense in Depth explicado
- [ ] Zero Trust Architecture explicado com exemplos K8s
- [ ] 5 projetos CNCF de segurança identificados
- [ ] Shift-left security explicado

---

### Desafio 15.2 — Cluster Component Security (22%)

Entenda segurança dos componentes do cluster.

**Tópicos para estudo:**
- kube-apiserver: authentication, authorization, admission
- etcd: encryption at rest, TLS
- kubelet: authn/authz, read-only port
- kube-proxy: modos e segurança
- Container runtime: security features

**Exercícios:**
```bash
# 1. Verificar flags de segurança do kube-apiserver
kubectl get pod kube-apiserver-control-plane -n kube-system -o yaml | \
  grep -E "anonymous-auth|enable-admission|authorization-mode|audit"

# Flags importantes:
# --anonymous-auth=false
# --authorization-mode=Node,RBAC
# --enable-admission-plugins=NodeRestriction,PodSecurity
# --audit-log-path=...
# --encryption-provider-config=...

# 2. Verificar etcd encryption
cat /etc/kubernetes/enc/encryption-config.yaml
# Deve ter aescbc ou secretbox como provider

# 3. Verificar kubelet security
# Em cada node:
cat /var/lib/kubelet/config.yaml | grep -E "authentication|authorization|readOnlyPort"
# readOnlyPort: 0 (desabilitado)
# authentication.anonymous.enabled: false

# 4. Admission controllers ativos
kubectl get pod kube-apiserver-control-plane -n kube-system -o yaml | \
  grep enable-admission
```

```markdown
## Quiz — Cluster Components

1. Quais são os 3 estágios de acesso ao API server?
   - Authentication (quem é você?)
   - Authorization (o que você pode fazer?)
   - Admission Control (validação/mutação do request)

2. Por que etcd deve ter encryption at rest?
   - Secrets armazenados em plain text por default
   - Se disco for comprometido, secrets expostos

3. Por que desabilitar readOnlyPort do kubelet?
   - Porta 10255 permite acesso sem autenticação
   - Expõe informações sobre Pods e métricas

4. Quais admission controllers são essenciais para segurança?
   - PodSecurity (enforce PSS)
   - NodeRestriction
   - ValidatingAdmissionWebhook (Kyverno/OPA)

5. Diferença entre ABAC e RBAC no Kubernetes?
```

**Critérios de aceite:**
- [ ] 3 estágios de acesso ao API server explicados
- [ ] Flags de segurança do kube-apiserver conhecidas
- [ ] etcd encryption at rest entendido
- [ ] kubelet security features explicadas
- [ ] Admission controllers críticos listados

---

### Desafio 15.3 — Kubernetes Security Fundamentals (22%)

Domine os fundamentos de segurança do Kubernetes.

**Tópicos para estudo:**
- RBAC em profundidade
- Pod Security Standards (Privileged, Baseline, Restricted)
- Network Policies
- Secrets management
- SecurityContext

**Exercícios:**
```markdown
## Quiz — Security Fundamentals

1. Compare Role vs ClusterRole:
   - Role: namespace-scoped
   - ClusterRole: cluster-scoped (ou namespace com ClusterRoleBinding)

2. Quais são os 3 níveis de Pod Security Standards?
   - Privileged: sem restrições
   - Baseline: previne escalação conhecida
   - Restricted: hardened (non-root, drop ALL, readOnly)

3. Network Policy: o que acontece se não há policies no namespace?
   - Todo tráfego é permitido (default allow all)
   - Primeira policy criada → tráfego não coberto é NEGADO

4. Por que Secrets no K8s NÃO são seguros por default?
   - Armazenados em base64 (não encriptação)
   - Sem encryption at rest por default
   - Qualquer um com acesso ao namespace pode ler

5. SecurityContext essencial para produção:
   - runAsNonRoot: true
   - allowPrivilegeEscalation: false
   - readOnlyRootFilesystem: true
   - capabilities: drop ALL
   - seccompProfile: RuntimeDefault
```

```bash
# Laboratório: verificar segurança dos Pods da CloudShop
# Pods sem securityContext adequado
kubectl get pods -n cloudshop-prod -o json | jq '
  .items[] |
  select(.spec.containers[].securityContext.runAsNonRoot != true) |
  .metadata.name
'

# Pods com capabilities perigosas
kubectl get pods -n cloudshop-prod -o json | jq '
  .items[] |
  select(.spec.containers[].securityContext.capabilities.add != null) |
  {name: .metadata.name, caps: .spec.containers[].securityContext.capabilities.add}
'
```

**Critérios de aceite:**
- [ ] RBAC: Role vs ClusterRole vs Bindings explicados
- [ ] PSS 3 níveis explicados com exemplos
- [ ] Network Policy behavior explicado (default allow vs primeira policy)
- [ ] Secrets: limitações de segurança documentadas
- [ ] SecurityContext checklist para produção

---

### Desafio 15.4 — Kubernetes Threat Model (16%)

Entenda o modelo de ameaças do Kubernetes.

**Tópicos para estudo:**
- STRIDE threat model
- Attack vectors em Kubernetes
- MITRE ATT&CK for Containers
- Threat actors e motivações
- Attack surface do cluster

**Exercícios:**
```markdown
## Quiz — Threat Model

1. STRIDE aplicado ao Kubernetes:
   - Spoofing: roubo de ServiceAccount token
   - Tampering: alteração de ConfigMaps/Secrets
   - Repudiation: falta de audit logging
   - Information Disclosure: secrets em env vars, etcd sem encryption
   - Denial of Service: resource exhaustion, fork bomb em container
   - Elevation of Privilege: container escape, privileged pods

2. Top 5 attack vectors em Kubernetes:
   - Exposed API server (sem firewall)
   - Compromised container image (supply chain)
   - Misconfigured RBAC (cluster-admin para todos)
   - Unpatched vulnerabilities (CVEs)
   - Secrets management inadequado

3. MITRE ATT&CK for Containers:
   - Initial Access: exploited public-facing app
   - Execution: exec into container
   - Persistence: backdoored image, cronjob
   - Privilege Escalation: privileged container, node access
   - Defense Evasion: log clearing
   - Credential Access: mounted service account tokens
   - Discovery: API server enumeration
   - Lateral Movement: pod-to-pod via service mesh
   - Impact: resource hijacking (crypto mining)

4. Como reduzir attack surface do CloudShop?
   - API server: firewall, RBAC, audit
   - Nodes: minimal OS, CIS hardening
   - Pods: PSS restricted, non-root, readOnly
   - Network: default deny, mTLS
   - Images: scan, sign, whitelist

5. Quem são os threat actors para um cluster K8s?
   - External attacker (internet)
   - Malicious insider (developer com muito acesso)
   - Compromised workload (container escapee)
   - Supply chain (compromised dependency)
```

**Critérios de aceite:**
- [ ] STRIDE aplicado ao Kubernetes
- [ ] 5 attack vectors principais listados
- [ ] MITRE ATT&CK for Containers: fases explicadas
- [ ] Attack surface do CloudShop mapeado
- [ ] Plano de mitigação documentado

---

### Desafio 15.5 — Platform Security (16%)

Entenda segurança da plataforma e infra.

**Tópicos para estudo:**
- Supply chain security (SLSA, Sigstore)
- Image scanning e signing
- Runtime security (Falco, Sysdig)
- Network security (CNI, service mesh, mTLS)
- Observability for security (audit logs, monitoring)

**Exercícios:**
```markdown
## Quiz — Platform Security

1. O que é SLSA (Supply-chain Levels for Software Artifacts)?
   - Framework de segurança de supply chain
   - 4 níveis de maturidade
   - SLSA 1: documentação do build
   - SLSA 4: hermetic build, provenance verificável

2. Como funciona image signing com Sigstore/Cosign?
   - Gerar par de chaves
   - Assinar imagem após build
   - Verificar assinatura no admission controller
   - Keyless signing com OIDC

3. Falco vs Sysdig:
   - Falco: open source, detecção de ameaças runtime
   - Sysdig: commercial, Falco + response + compliance

4. Compare CNI plugins para segurança:
   - Calico: network policies, eBPF, encryption
   - Cilium: eBPF native, L7 policies, mTLS
   - Flannel: simples, SEM network policies nativas

5. Audit logging: o que monitorar?
   - Secrets access (quem leu/alterou)
   - exec em containers
   - RBAC changes
   - Pod creation/deletion
   - Authentication failures
```

**Critérios de aceite:**
- [ ] SLSA framework explicado (4 níveis)
- [ ] Image signing workflow documentado
- [ ] Runtime security tools comparados
- [ ] CNI plugins comparados por segurança
- [ ] Audit logging: eventos críticos listados

---

### Desafio 15.6 — Compliance and Frameworks (10%)

Entenda frameworks de compliance para Kubernetes.

**Tópicos para estudo:**
- CIS Kubernetes Benchmark
- NIST 800-190 (Container Security)
- SOC 2 para Kubernetes
- PCI-DSS para workloads em containers
- GDPR e data protection

**Exercícios:**
```markdown
## Quiz — Compliance

1. CIS Kubernetes Benchmark — seções principais:
   - Control Plane Components (API server, etcd, scheduler, controller)
   - Worker Node (kubelet, kube-proxy)
   - Policies (RBAC, PSP/PSS, network, secrets)
   - Ferramentas: kube-bench

2. NIST 800-190 — recomendações para containers:
   - Image hardening (minimal base, no root)
   - Registry security (private, scanning)
   - Container runtime security (seccomp, AppArmor)
   - Orchestrator security (RBAC, network policies)

3. SOC 2 para Kubernetes:
   - Audit logging (principle: auditing)
   - Access controls (principle: security)
   - Encryption (principle: confidentiality)
   - Monitoring (principle: availability)

4. PCI-DSS em Kubernetes:
   - Segmentação de rede (network policies)
   - Encryption de dados sensíveis
   - Access control (RBAC)
   - Logging e monitoring
   - Vulnerability management (scanning)

5. GDPR:
   - Data encryption at rest e in transit
   - Right to be forgotten (data deletion)
   - Data residency (region-specific clusters)
   - Audit trails
```

```bash
# Executar CIS Benchmark
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench
# Analisar: PASS, FAIL, WARN
# Focar em FAILs para remediation
```

**Critérios de aceite:**
- [ ] CIS Benchmark executado e FAILs analisados
- [ ] NIST 800-190 recomendações entendidas
- [ ] SOC 2 mapping para K8s documentado
- [ ] PCI-DSS requirements mapeados para K8s
- [ ] Compliance checklist criado para CloudShop

---

## Definição de Pronto (DoD)

- [ ] 6 domínios estudados com quizzes e exercícios
- [ ] 4 Cs da segurança dominados
- [ ] Threat model para CloudShop documentado
- [ ] Compliance frameworks entendidos
- [ ] Simulado com score ≥ 75%

---

## Checklist

- [ ] Cloud Native Security — 4 Cs, Zero Trust
- [ ] Cluster Components — API server, etcd, kubelet
- [ ] Security Fundamentals — RBAC, PSS, NetworkPolicy
- [ ] Threat Model — STRIDE, MITRE ATT&CK
- [ ] Platform Security — supply chain, runtime, CNI
- [ ] Compliance — CIS, NIST, SOC 2
- [ ] 2+ simulados com score ≥ 75%
