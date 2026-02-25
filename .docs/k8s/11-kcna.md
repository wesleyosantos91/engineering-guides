# KCNA — Kubernetes and Cloud Native Associate

## Informações do Exame

| Item | Detalhe |
|------|---------|
| **Formato** | Multiple choice |
| **Duração** | 90 minutos |
| **Questões** | ~60 questões |
| **Passing Score** | 75% |
| **Validade** | 3 anos |
| **Pré-requisitos** | Nenhum |
| **Preço** | $250 USD (inclui 1 retake) |
| **Idioma** | Inglês, Japonês, Chinês |

## Domínios do Exame

| Domínio | Peso | Tópicos |
|---------|:---:|---------|
| **Kubernetes Fundamentals** | 46% | K8s architecture, API, objects, containers |
| **Container Orchestration** | 22% | Container primitives, runtime, security |
| **Cloud Native Architecture** | 16% | Autoscaling, serverless, community, roles |
| **Cloud Native Observability** | 8% | Prometheus, Fluentd, Jaeger, cost management |
| **Cloud Native Application Delivery** | 8% | GitOps, CI/CD, Helm |

## Conteúdo Detalhado por Domínio

### 1. Kubernetes Fundamentals (46%)

#### Architecture
```
Control Plane:
  ├── kube-apiserver      → API gateway, authn/authz
  ├── etcd                → Key-value store (estado do cluster)
  ├── kube-scheduler      → Scheduling de Pods
  └── kube-controller-manager → Controllers (Deployment, ReplicaSet, etc.)

Node Components:
  ├── kubelet             → Agente que roda Pods no node
  ├── kube-proxy          → Networking (iptables/IPVS)
  └── Container Runtime   → containerd, CRI-O
```

#### Core Objects
| Objeto | Descrição |
|--------|-----------|
| **Pod** | Menor unidade deployable, 1+ containers |
| **ReplicaSet** | Garante N replicas de um Pod |
| **Deployment** | Declarativo, rolling updates, rollbacks |
| **Service** | Networking estável para Pods (ClusterIP, NodePort, LB) |
| **Namespace** | Isolamento lógico de recursos |
| **ConfigMap** | Configuração não-sensível |
| **Secret** | Dados sensíveis (base64) |
| **PersistentVolume** | Storage persistente |
| **StatefulSet** | Pods com identidade e storage estáveis |
| **DaemonSet** | Um Pod por node |
| **Job/CronJob** | Tarefas batch e agendadas |
| **Ingress** | Roteamento HTTP/S externo |

#### API e kubectl
```bash
# Estrutura da API
/api/v1/namespaces/{namespace}/pods/{name}
/apis/apps/v1/namespaces/{namespace}/deployments/{name}

# Grupos de API
Core:     /api/v1          → Pods, Services, Namespaces
Apps:     /apis/apps/v1    → Deployments, StatefulSets, DaemonSets
Batch:    /apis/batch/v1   → Jobs, CronJobs
RBAC:     /apis/rbac.authorization.k8s.io/v1
```

#### Labels e Selectors
```yaml
# Labels — key-value pairs para identificação
metadata:
  labels:
    app: frontend
    environment: production

# Selectors
selector:
  matchLabels:
    app: frontend          # equality-based
  matchExpressions:
    - key: environment
      operator: In         # set-based
      values: [production, staging]
```

### 2. Container Orchestration (22%)

#### Container Basics
- **Container**: processo isolado com filesystem próprio.
- **Image**: template imutável para criar containers.
- **Registry**: repositório de imagens (Docker Hub, ECR, GCR).
- **OCI**: padrão aberto para container images e runtime.

#### Container Runtime
| Runtime | Descrição |
|---------|-----------|
| **containerd** | Padrão no K8s, leve, CRI-compliant |
| **CRI-O** | Minimali, focado em K8s |
| **Docker** (dockershim) | Removido no K8s 1.24 |

#### Container Security
- **rootless containers** — não rodar como root.
- **read-only filesystem** — imutabilidade.
- **capabilities** — drop ALL, add apenas necessário.
- **seccomp** — filtro de syscalls.
- **AppArmor/SELinux** — MAC (Mandatory Access Control).

### 3. Cloud Native Architecture (16%)

#### Twelve-Factor App
1. Codebase — um repo, muitos deploys
2. Dependencies — explícitas e isoladas
3. Config — em variáveis de ambiente
4. Backing Services — como attached resources
5. Build, Release, Run — estágios separados
6. Processes — stateless, share-nothing
7. Port Binding — self-contained
8. Concurrency — scale out via processes
9. Disposability — fast startup, graceful shutdown
10. Dev/Prod Parity — ambientes similares
11. Logs — event streams
12. Admin Processes — one-off tasks

#### Cloud Native Patterns
- **Microservices** — serviços independentes e loosely coupled.
- **Serverless** — Knative, AWS Lambda, Cloud Functions.
- **Service Mesh** — Istio, Linkerd, Cilium.
- **GitOps** — Git como source of truth para infra.

#### Autoscaling
| Tipo | Descrição |
|------|-----------|
| **HPA** | Horizontal — mais/menos Pods |
| **VPA** | Vertical — mais/menos resources por Pod |
| **Cluster Autoscaler** | Mais/menos nodes |
| **KEDA** | Event-driven autoscaling |

#### CNCF Ecosystem
- **Graduated**: Kubernetes, Prometheus, Envoy, CoreDNS, containerd, Fluentd, Helm, etcd, Argo, Flux...
- **Incubating**: Knative, Dapr, Backstage, Kyverno, OpenCost...
- **Sandbox**: Centenas de projetos iniciais.

### 4. Cloud Native Observability (8%)

#### Os Três Pilares
| Pilar | Ferramenta CNCF | Descrição |
|-------|:---:|-----------|
| **Metrics** | Prometheus | Time-series, PromQL |
| **Logging** | Fluentd/FluentBit | Log aggregation |
| **Tracing** | Jaeger | Distributed tracing |
| **All-in-one** | OpenTelemetry | Vendor-neutral SDK |

#### Prometheus
```
Scrape → Store → Query → Alert
         ↓
     PromQL: rate(http_requests_total[5m])
```

#### Cost Management
- **FinOps** — práticas para otimizar custo cloud.
- **Kubecost / OpenCost** — alocação de custos por workload.
- **Rightsizing** — ajustar resources baseado em uso real.

### 5. Cloud Native Application Delivery (8%)

#### CI/CD
- **CI**: Build → Test → Scan → Push
- **CD**: Deploy → Verify → Promote

#### GitOps Tools
| Ferramenta | Modelo | Projeto CNCF |
|-----------|--------|:---:|
| **ArgoCD** | Pull | ✅ Graduated |
| **Flux** | Pull | ✅ Graduated |
| **Tekton** | Push | ✅ (CD Foundation) |

#### Helm
- **Chart**: pacote de manifests K8s.
- **Release**: instância de um chart no cluster.
- **Repository**: coleção de charts.

## Dicas para o Exame

1. **Foco em conceitos** — não é hands-on, é teórico.
2. **CNCF Landscape** — saiba os projetos principais e seus papéis.
3. **Kubernetes docs** — leia a documentação oficial.
4. O exame testa **compreensão ampla**, não profundidade.
5. **Cloud Native Glossary** — familiarize-se com terminologia.
6. **Eliminação** — nas questões de múltipla escolha, elimine opções erradas.
7. **Tempo** — 90 min para ~60 questões ≈ 1.5 min/questão.

## Recursos de Estudo

- [Kubernetes Documentation](https://kubernetes.io/docs/concepts/)
- [CNCF Cloud Native Glossary](https://glossary.cncf.io/)
- [CNCF Landscape](https://landscape.cncf.io/)
- [12 Factor App](https://12factor.net/)
- [KCNA Curriculum](https://github.com/cncf/curriculum)
- KodeKloud KCNA Course
- Udemy — KCNA courses
