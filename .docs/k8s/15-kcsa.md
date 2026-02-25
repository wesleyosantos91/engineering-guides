# KCSA — Kubernetes and Cloud Native Security Associate

## Informações do Exame

| Item | Detalhe |
|------|---------|
| **Formato** | Multiple choice |
| **Duração** | 90 minutos |
| **Questões** | ~60 questões |
| **Passing Score** | 75% |
| **Validade** | 3 anos |
| **Pré-requisitos** | Nenhum (recomenda-se KCNA ou experiência K8s) |
| **Preço** | $250 USD (inclui 1 retake) |
| **Idioma** | Inglês |

## Domínios do Exame

| Domínio | Peso | Tópicos |
|---------|:---:|---------|
| **Overview of Cloud Native Security** | 14% | 4Cs, threat modeling, security principles |
| **Kubernetes Cluster Component Security** | 22% | API server, etcd, kubelet, control plane |
| **Kubernetes Security Fundamentals** | 22% | Pod Security, RBAC, Network Policies, Secrets |
| **Kubernetes Threat Model** | 16% | STRIDE, attack surfaces, persistence |
| **Platform Security** | 16% | Supply chain, image security, admission control |
| **Compliance and Security Frameworks** | 10% | CIS, NIST, PCI-DSS, auditing |

## Conteúdo Detalhado por Domínio

### 1. Overview of Cloud Native Security (14%)

#### The 4Cs of Cloud Native Security
```
┌─────────────────────────────────┐
│            Code                  │  ← Application security
│    ┌───────────────────────┐    │
│    │      Container         │    │  ← Container security
│    │  ┌───────────────┐    │    │
│    │  │   Cluster      │    │    │  ← K8s security
│    │  │  ┌─────────┐  │    │    │
│    │  │  │  Cloud   │  │    │    │  ← Infrastructure security
│    │  │  └─────────┘  │    │    │
│    │  └───────────────┘    │    │
│    └───────────────────────┘    │
└─────────────────────────────────┘
```

| Camada | Responsabilidades |
|--------|-------------------|
| **Cloud** | IAM, network, encryption, compliance |
| **Cluster** | API server, etcd, RBAC, admission, audit |
| **Container** | Image scanning, runtime security, least privilege |
| **Code** | SAST, DAST, dependency scanning, secrets management |

#### Security Principles
- **Defense in Depth** — múltiplas camadas de segurança.
- **Least Privilege** — mínimo de permissões necessárias.
- **Zero Trust** — nunca confie, sempre verifique.
- **Shift Left** — segurança desde o início do desenvolvimento.
- **Separation of Duty** — dividir responsabilidades.
- **Fail Secure** — falhar de forma segura (deny by default).

#### Threat Modeling
- **STRIDE**:
  - **S**poofing — falsificação de identidade
  - **T**ampering — alteração de dados
  - **R**epudiation — negar ações realizadas
  - **I**nformation Disclosure — vazamento de dados
  - **D**enial of Service — indisponibilidade
  - **E**levation of Privilege — escalação de privilégios

### 2. Kubernetes Cluster Component Security (22%)

#### API Server Security
| Configuração | Descrição |
|-------------|-----------|
| `--anonymous-auth=false` | Desabilitar acesso anônimo |
| `--authorization-mode=Node,RBAC` | Modes de autorização |
| `--enable-admission-plugins` | Admission controllers |
| `--tls-cert-file`, `--tls-private-key-file` | TLS para API |
| `--audit-log-path` | Audit logging |
| `--encryption-provider-config` | Criptografia at rest |
| `--profiling=false` | Desabilitar profiling |

#### etcd Security
- Comunicação via **TLS** (client e peer).
- Acesso **restrito** (apenas API server).
- **Encriptação at rest** para secrets.
- **Backup regular** e testado.
- Firewall: **porta 2379** (client), **2380** (peer).

#### Kubelet Security
| Configuração | Descrição |
|-------------|-----------|
| `--anonymous-auth=false` | Sem acesso anônimo |
| `--authorization-mode=Webhook` | Autorização via API server |
| `--read-only-port=0` | Desabilitar porta read-only |
| `--protect-kernel-defaults=true` | Proteger configurações do kernel |
| `--tls-cert-file`, `--tls-private-key-file` | TLS |

#### Control Plane Communication
```
External → Load Balancer → API Server (TLS)
                              ↓
                           etcd (mTLS)
                              ↑
                    Controller Manager (TLS)
                    Scheduler (TLS)

API Server → kubelet (TLS)
kubelet → Container Runtime (CRI)
```

### 3. Kubernetes Security Fundamentals (22%)

#### Pod Security
```yaml
# Pod Security Admission (PSA) Levels:
# Privileged → Baseline → Restricted

# Labels no namespace
pod-security.kubernetes.io/enforce: restricted
pod-security.kubernetes.io/audit: restricted
pod-security.kubernetes.io/warn: restricted
```

**Restricted Level Requirements:**
- `runAsNonRoot: true`
- `allowPrivilegeEscalation: false`
- `capabilities.drop: ["ALL"]`
- `seccompProfile.type: RuntimeDefault`
- Sem `hostNetwork`, `hostPID`, `hostIPC`
- Sem `privileged: true`
- Sem `hostPath` volumes

#### RBAC
| Recurso | Escopo | Descrição |
|---------|--------|-----------|
| **Role** | Namespace | Permissões em um namespace |
| **ClusterRole** | Cluster | Permissões cluster-wide |
| **RoleBinding** | Namespace | Liga Role/ClusterRole a users/groups |
| **ClusterRoleBinding** | Cluster | Liga ClusterRole a users/groups |

**Regras de RBAC:**
- Princípio do **menor privilégio**.
- Prefira **Role** sobre ClusterRole quando possível.
- Use **Groups** em vez de Users individuais.
- Audite permissões regularmente.
- Evite wildcards (`*`) em verbs e resources.

#### Network Policies
```
Sem NetworkPolicy → Todos os pods se comunicam (default allow)
Com NetworkPolicy → Apenas tráfego explicitamente permitido

Recomendação:
1. Default deny ALL (ingress + egress)
2. Liberar tráfego necessário por pod/namespace
3. Sempre liberar DNS (porta 53 UDP)
```

#### Secrets
- Secrets são apenas **base64 encoded** (não encriptados por padrão).
- Habilite **encryption at rest** no etcd.
- Use **External Secrets** (Vault, cloud KMS) para produção.
- Prefira **volume mount** sobre env vars.
- Nunca commite secrets no Git.

### 4. Kubernetes Threat Model (16%)

#### Attack Surfaces
```
External Attacks:
  ├── Exposed API Server
  ├── Exposed Services (NodePort, LoadBalancer)
  ├── Ingress vulnerabilities
  ├── Supply chain (malicious images)
  └── Stolen credentials

Internal Attacks (Compromised Pod):
  ├── Lateral movement (pod to pod)
  ├── Privilege escalation
  ├── Service account token theft
  ├── etcd access
  ├── Node escape (container breakout)
  └── Cloud metadata API access
```

#### Common Attack Vectors
| Vetor | Mitigação |
|-------|-----------|
| Exposed Dashboard | Autenticação, RBAC, NetworkPolicy |
| Privileged containers | PSA Restricted, admission control |
| Default SA token | automountServiceAccountToken: false |
| etcd access | Firewall, mTLS, encryption |
| Cloud metadata | NetworkPolicy egress, IMDS v2 |
| Malicious images | Registry allowlist, image scanning |
| RBAC misconfiguration | Audit, least privilege |
| Unpatched CVEs | Regular upgrades, scanning |

#### Kubernetes-Specific Threats
- **Container escape** — kernel exploits, `privileged: true`.
- **Denial of Service** — resource exhaustion, fork bombs.
- **Credential theft** — SA tokens, cloud IAM.
- **Supply chain** — compromised base images, typosquatting.
- **Persistence** — CronJobs, DaemonSets, webhooks maliciosos.

### 5. Platform Security (16%)

#### Supply Chain Security
```
Build:
  ✅ Scan source code (SAST)
  ✅ Scan dependencies (SCA)
  ✅ Multi-stage builds
  ✅ Minimal base images (distroless)

Registry:
  ✅ Private registry
  ✅ Image scanning (Trivy, Grype)
  ✅ Image signing (Cosign, Notary)
  ✅ SBOM (Software Bill of Materials)

Deploy:
  ✅ Admission control (registry allowlist)
  ✅ Runtime scanning
  ✅ Immutable tags (no :latest)
```

#### Admission Controllers
| Controller | Tipo | Descrição |
|-----------|------|-----------|
| **PodSecurity** | Built-in | PSA/PSS enforcement |
| **NodeRestriction** | Built-in | Limita kubelet |
| **OPA Gatekeeper** | External | Policy engine (Rego) |
| **Kyverno** | External | Policy engine (YAML) |
| **ImagePolicyWebhook** | External | Registry allowlist |

#### Image Security
- Use imagens **distroless** ou **scratch** quando possível.
- **Nunca use `latest`** tag.
- **Scan** com Trivy em pipeline e runtime.
- **Sign** com Cosign (Sigstore).
- Mantenha imagem **atualizada** (patch CVEs).
- **SBOM** para cada imagem.

### 6. Compliance and Security Frameworks (10%)

#### CIS Kubernetes Benchmark
| Seção | Alvo |
|-------|------|
| 1 | Control Plane Components |
| 2 | etcd |
| 3 | Control Plane Configuration |
| 4 | Worker Nodes |
| 5 | Policies |

```bash
# kube-bench para verificar
kube-bench run --targets=master,node
```

#### Frameworks
| Framework | Foco |
|-----------|------|
| **CIS Benchmark** | Hardening K8s |
| **NIST SP 800-190** | Container security |
| **NIST SP 800-204** | Microservices security |
| **PCI-DSS** | Payment card data |
| **HIPAA** | Healthcare data |
| **SOC 2** | Service organization controls |
| **MITRE ATT&CK** | Threat framework (containers) |
| **NSA/CISA K8s Hardening** | Government hardening guide |

#### Audit e Compliance
- **Audit logs** — quem fez o quê, quando.
- **Policy enforcement** — preventivo (admission control).
- **Scanning** — detectivo (vulnerabilities, misconfigurations).
- **RBAC review** — periódico (quem tem acesso a quê).
- **Image scanning** — periódico (novas CVEs).

## Dicas para o Exame

1. **Formato teórico** — similar ao KCNA, múltipla escolha.
2. **Foco em conceitos** — entenda "por que" não apenas "como".
3. **4Cs** — saiba cada camada e responsabilidades.
4. **STRIDE** — conheça cada categoria de ameaça.
5. **Pod Security Standards** — saiba os três níveis e o que cada um restringe.
6. **Supply chain** — pipeline completo (build → scan → sign → deploy).
7. **CIS Benchmark** — conheça os principais pontos.
8. **Defense in depth** — múltiplas camadas de proteção.
9. **Zero Trust** — princípio fundamental.
10. **Eliminação** — nas questões, elimine opções claramente erradas.

## Recursos de Estudo

- [KCSA Curriculum](https://github.com/cncf/curriculum)
- [Kubernetes Security Docs](https://kubernetes.io/docs/concepts/security/)
- [NIST SP 800-190](https://csrc.nist.gov/publications/detail/sp/800-190/final)
- [NSA/CISA Kubernetes Hardening Guide](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [MITRE ATT&CK for Containers](https://attack.mitre.org/matrices/enterprise/containers/)
- KodeKloud KCSA Course
- [Cloud Native Security Whitepaper](https://github.com/cncf/tag-security/blob/main/security-whitepaper/v2/cloud-native-security-whitepaper.md)
