# Zero to Hero — Kubernetes Challenge

> **Programa de especialização progressiva** em Kubernetes, cobrindo boas práticas de mercado,
> padrões de arquitetura e preparação para as **5 certificações CNCF** (Kubestronaut path).

---

## 1. Resumo Executivo

Este programa transforma um desenvolvedor/ops em **especialista em Kubernetes** através de desafios práticos que cobrem desde fundamentos de cluster até operações avançadas de produção, culminando na preparação para o título **Kubestronaut** (todas as 5 certificações CNCF).

**Domínio escolhido:** **CloudShop Platform** — uma plataforma de e-commerce cloud-native com múltiplos microsserviços que exige todos os conceitos de Kubernetes de forma natural.

**Por que CloudShop?**
- **Workloads variados** — APIs stateless (Deployment), banco de dados (StatefulSet), jobs de processamento (Job/CronJob), sidecars de logging
- **Segurança real** — RBAC por time, Pod Security Standards, Network Policies entre namespaces, secrets de pagamento
- **Networking complexo** — Ingress com TLS, Service Mesh para mTLS, Gateway API para traffic splitting
- **Resource management** — HPA para lidar com picos de Black Friday, VPA para rightsizing, quotas por time
- **Observabilidade** — Prometheus + Grafana + Loki + Tempo, alertas de SLO para checkout
- **GitOps** — ArgoCD para deploy em múltiplos ambientes (dev, staging, prod)
- **Storage** — PVCs para databases, backups com Velero, VolumeSnapshots
- **Alta disponibilidade** — Multi-AZ, PDB, graceful shutdown, DR drills
- Perguntas de entrevista Staff/Principal frequentemente envolvem design de plataformas em Kubernetes

---

## 2. Arquitetura do Domínio CloudShop

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Web App    │     │  Mobile App  │     │    Admin     │
│   (React)    │     │  (Flutter)   │     │  Dashboard   │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       ▼                    ▼                    ▼
┌──────────────────────────────────────────────────────────┐
│              Ingress / Gateway API                       │
│         (TLS, routing, rate limiting)                    │
└──────┬────────────────┬───────────────────┬──────────────┘
       │                │                   │
  ┌────▼────┐    ┌──────▼──────┐     ┌──────▼──────┐
  │  Order  │    │  Product    │     │   User      │
  │ Service │    │  Service    │     │  Service    │
  │(Deploy) │    │ (Deploy)    │     │ (Deploy)    │
  └────┬────┘    └──────┬──────┘     └──────┬──────┘
       │                │                   │
  ┌────▼────┐    ┌──────▼──────┐     ┌──────▼──────┐
  │ Payment │    │  Inventory  │     │Notification │
  │ Service │    │  Service    │     │  Service    │
  │(Deploy) │    │ (Deploy)    │     │ (Deploy)    │
  └────┬────┘    └──────┬──────┘     └──────┬──────┘
       │                │                   │
       └────────────────┼───────────────────┘
                        │
              ┌─────────▼─────────┐
              │    Kafka/NATS     │
              │  (StatefulSet)    │
              └─────────┬─────────┘
                        │
              ┌─────────▼─────────┐     ┌───────────────┐
              │   PostgreSQL      │     │     Redis      │
              │  (StatefulSet)    │     │ (StatefulSet)  │
              └───────────────────┘     └───────────────┘

Plataforma:
┌────────────────────────────────────────────────────────────┐
│  Prometheus + Grafana  │  Loki + FluentBit  │  Tempo      │
│  ArgoCD                │  cert-manager      │  Velero     │
│  External Secrets      │  Kyverno           │  Karpenter  │
└────────────────────────────────────────────────────────────┘
```

### Microsserviços

| Serviço | Tipo Workload | Descrição |
|---------|:---:|-----------|
| `order-service` | Deployment | Gerencia pedidos, checkout, status |
| `product-service` | Deployment | Catálogo de produtos, busca, categorias |
| `user-service` | Deployment | Autenticação, perfil, endereços |
| `payment-service` | Deployment | Processamento de pagamentos |
| `inventory-service` | Deployment | Controle de estoque, reservas |
| `notification-service` | Deployment | E-mail, push, SMS |
| `report-generator` | CronJob | Relatórios diários de vendas |
| `data-migration` | Job | Migrações de schema de banco de dados |

### Infraestrutura

| Componente | Tipo Workload | Descrição |
|-----------|:---:|-----------|
| `postgresql` | StatefulSet | Banco de dados principal |
| `redis` | StatefulSet | Cache e sessões |
| `kafka` | StatefulSet | Event bus |
| `fluent-bit` | DaemonSet | Coletor de logs |

---

## 3. Namespace Strategy

```
├── cloudshop-prod/          ← Workloads de produção
├── cloudshop-staging/       ← Ambiente de staging
├── cloudshop-dev/           ← Ambiente de desenvolvimento
├── cloudshop-data/          ← Databases (PostgreSQL, Redis, Kafka)
├── monitoring/              ← Prometheus, Grafana, Tempo
├── logging/                 ← Loki, FluentBit
├── argocd/                  ← ArgoCD
├── ingress-nginx/           ← Ingress Controller
├── cert-manager/            ← Certificados TLS
└── external-secrets/        ← External Secrets Operator
```

---

## 4. Progressão dos Desafios

### Boas Práticas (Levels 0-10)

```
Level 0 — Kubernetes Foundations
│  kubectl, namespaces, pods, labels, API structure
│
Level 1 — Workloads & Pod Design
│  Deployments, StatefulSets, Jobs, CronJobs, Probes,
│  Init Containers, multi-container patterns
│
Level 2 — Security
│  RBAC, Pod Security Standards, Network Policies,
│  Service Accounts, Secrets Management, Supply Chain
│
Level 3 — Networking
│  Services, Ingress, Gateway API, DNS, Service Mesh,
│  Load Balancing, Network Troubleshooting
│
Level 4 — Resource Management
│  Requests/Limits, QoS, HPA, VPA, LimitRange,
│  ResourceQuota, Cost Optimization
│
Level 5 — Observability
│  Logging (FluentBit + Loki), Metrics (Prometheus),
│  Tracing (Tempo), Alerting, Dashboards
│
Level 6 — CI/CD & GitOps
│  Kustomize, Helm, ArgoCD, Flux, Image Automation,
│  Progressive Delivery
│
Level 7 — Storage
│  PV, PVC, StorageClass, CSI Drivers, Backup (Velero),
│  VolumeSnapshots
│
Level 8 — High Availability & DR
│  PDB, Multi-AZ, Topology Spread, Graceful Shutdown,
│  Multi-Cluster, Disaster Recovery
│
Level 9 — Configuration Management
│  ConfigMaps, Secrets, External Secrets, Sealed Secrets,
│  Helm values, Config Reload
│
Level 10 — Cluster Operations
│  Upgrades, etcd backup/restore, Node Management,
│  Troubleshooting, Cost Optimization, Compliance
```

### Certificações Kubestronaut (Levels 11-16)

```
Level 11 — KCNA (Kubernetes and Cloud Native Associate)
│  Conceitos fundamentais, teoria, múltipla escolha
│
Level 12 — CKAD (Certified Kubernetes Application Developer)
│  Hands-on: criar, configurar e expor aplicações
│
Level 13 — CKA (Certified Kubernetes Administrator)
│  Hands-on: instalar, administrar e troubleshoot clusters
│
Level 14 — CKS (Certified Kubernetes Security Specialist)
│  Hands-on: hardening, scanning, runtime security
│
Level 15 — KCSA (Kubernetes Cloud Native Security Associate)
│  Conceitos de segurança, múltipla escolha
│
Level 16 — Kubestronaut
│  Capstone: integração de todas as certificações,
│  DR drill completo, plataforma production-grade
```

---

## 5. Trilha Zero to Hero

| Nível | Foco | Semana |
|---|---|---|
| [Level 0](00-k8s-foundations.md) | Fundamentos do Kubernetes | Semana 1-2 |
| [Level 1](01-workloads-pod-design.md) | Workloads & Pod Design | Semana 3-4 |
| [Level 2](02-security.md) | Segurança | Semana 5-6 |
| [Level 3](03-networking.md) | Networking | Semana 7-8 |
| [Level 4](04-resource-management.md) | Resource Management | Semana 9-10 |
| [Level 5](05-observability.md) | Observabilidade | Semana 11-12 |
| [Level 6](06-cicd-gitops.md) | CI/CD & GitOps | Semana 13-14 |
| [Level 7](07-storage.md) | Storage | Semana 15-16 |
| [Level 8](08-high-availability.md) | Alta Disponibilidade & DR | Semana 17-18 |
| [Level 9](09-configuration.md) | Configuration Management | Semana 19-20 |
| [Level 10](10-cluster-operations.md) | Cluster Operations | Semana 21-22 |
| [Level 11](11-kcna.md) | KCNA Prep | Semana 23-24 |
| [Level 12](12-ckad.md) | CKAD Prep | Semana 25-28 |
| [Level 13](13-cka.md) | CKA Prep | Semana 29-32 |
| [Level 14](14-cks.md) | CKS Prep | Semana 33-36 |
| [Level 15](15-kcsa.md) | KCSA Prep | Semana 37-38 |
| [Level 16](16-kubestronaut.md) | Capstone Kubestronaut | Semana 39-40 |

---

## 6. Pré-requisitos

### Ferramentas Necessárias

| Ferramenta | Versão | Uso |
|-----------|--------|-----|
| `kubectl` | 1.31+ | CLI principal |
| `kind` ou `minikube` | Latest | Cluster local |
| `helm` | 3.x | Package manager |
| `kustomize` | 5.x | Gestão de manifests |
| `k9s` | Latest | TUI para cluster |
| `docker` | 24+ | Container runtime |
| `kubectx/kubens` | Latest | Switch de contexto/namespace |
| `stern` | Latest | Multi-pod log tailing |
| `pluto` | Latest | Deprecated API detection |

### Conhecimento Prévio

- Linux básico (shell, filesystem, networking)
- Docker & containers (build, run, compose)
- YAML fluente
- Networking básico (TCP/IP, DNS, HTTP/S, TLS)
- Git (branching, commits, PRs)

---

## 7. Ambiente de Prática

### Local (kind)
```bash
# Criar cluster multi-node
kind create cluster --config kind-cluster.yaml

# kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/16"
```

### Cloud (para certificações)
| Provider | Serviço | Free Tier |
|----------|---------|-----------|
| AWS | EKS | Não (control plane $0.10/h) |
| GCP | GKE Autopilot | Free tier (1 cluster) |
| Azure | AKS | Free (control plane) |
| Civo | Civo K8s | $250 credit trial |

---

## 8. Rubrica de Avaliação (0–100)

| Critério | Peso | 0-25 (Básico) | 26-50 (Intermediário) | 51-75 (Avançado) | 76-100 (Expert) |
|---|---|---|---|---|---|
| **Manifests** | 20% | YAML funcional | Best practices labels | Kustomize + overlays | Helm charts + tests |
| **Segurança** | 20% | Sem security context | PSS Baseline | PSS Restricted + RBAC | Network Policies + Admission + mTLS |
| **Observabilidade** | 15% | Sem monitoring | Probes configurados | Prometheus + Grafana | SLOs + Alerting + Tracing |
| **Networking** | 15% | ClusterIP apenas | Ingress + TLS | Gateway API + DNS | Service Mesh + Traffic Management |
| **Resiliência** | 15% | Single replica | Multi-replica + PDB | Multi-AZ + Graceful Shutdown | DR testado + Multi-cluster |
| **Operações** | 10% | kubectl manual | Scripts de automação | GitOps (ArgoCD) | Full platform automation |
| **Documentação** | 5% | Sem docs | README básico | Runbooks | ADRs + DR Playbooks + Dashboards |

---

## 9. Critérios de "Pronto para Produção"

- [ ] Todos os workloads com resource requests/limits
- [ ] Pod Security Standards (Restricted) aplicado
- [ ] Network Policies (default deny + allow específico)
- [ ] RBAC por time/namespace
- [ ] Secrets gerenciados externamente (External Secrets Operator)
- [ ] Ingress com TLS (cert-manager)
- [ ] Health checks (liveness + readiness + startup probes)
- [ ] HPA configurado para serviços stateless
- [ ] PDB em todos os workloads de produção
- [ ] Multi-AZ distribution (topologySpreadConstraints)
- [ ] Graceful shutdown implementado
- [ ] Logging estruturado (JSON) → FluentBit → Loki
- [ ] Métricas → Prometheus → Grafana com dashboards
- [ ] Tracing → OpenTelemetry → Tempo
- [ ] Alertas configurados (PrometheusRules)
- [ ] GitOps com ArgoCD (auto-sync + self-heal)
- [ ] Backup automatizado (Velero)
- [ ] VolumeSnapshots para databases
- [ ] Kustomize/Helm para gestão de ambientes
- [ ] DR testado e documentado

---

## 10. Referências

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [CNCF Certifications](https://www.cncf.io/training/certification/)
- [Kubernetes Patterns (O'Reilly)](https://k8spatterns.io/)
- [Production Kubernetes (O'Reilly)](https://www.oreilly.com/library/view/production-kubernetes/9781492092292/)
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [killer.sh (Exam Simulators)](https://killer.sh/)
- [KodeKloud](https://kodekloud.com/)
- [CNCF Landscape](https://landscape.cncf.io/)
