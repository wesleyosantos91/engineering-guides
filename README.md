# Engineering Guides

Guias pessoais de organização, convenções e padrões de código por tecnologia.

## Estrutura

```
.docs/
├── api/                        # API Design
│   ├── api-guide.md
│   ├── graphql/
│   │   └── graphql-best-practices.md
│   ├── grpc/
│   │   └── grpc-best-practices.md
│   └── rest/
│       └── rest-best-practices.md
├── aws/                        # AWS Best Practices & Well-Architected
│   ├── well-architected-framework.md
│   ├── serverless/
│   │   ├── serverless-best-practices.md
│   │   └── serverless-patterns.md
│   ├── security/
│   │   ├── security-best-practices.md
│   │   └── compliance-checklist.md
│   ├── networking/
│   │   └── networking-best-practices.md
│   ├── compute/
│   │   └── containers-best-practices.md
│   ├── databases/
│   │   └── databases-best-practices.md
│   ├── observability/
│   │   └── observability-best-practices.md
│   ├── cost-optimization/
│   │   └── cost-optimization-guide.md
│   ├── cicd/
│   │   └── cicd-best-practices.md
│   ├── reliability/
│   │   └── disaster-recovery.md
│   ├── storage/
│   │   └── storage-best-practices.md
│   └── governance/
│       └── multi-account-strategy.md
├── data-engineering/            # Data Engineering & Architecture (AWS)
│   ├── 01-data-architecture-foundations.md
│   ├── 02-data-storage.md
│   ├── 03-data-processing.md
│   ├── 04-data-integration.md
│   ├── 05-data-governance.md
│   └── 06-architecture-patterns.md
├── ddd/                        # Domain-Driven Design
│   ├── 01-strategic-design.md
│   ├── 02-tactical-design.md
│   ├── 03-context-mapping.md
│   ├── 04-anti-patterns.md
│   └── 05-architecture-integration.md
├── design-patterns/            # Design Patterns (agnóstico a linguagem)
│   ├── README.md
│   ├── architectural-patterns.md
│   ├── behavioral-patterns.md
│   ├── best-practices.md
│   ├── creational-patterns.md
│   ├── solid-principles.md
│   └── structural-patterns.md
├── golang/                     # Go
│   └── gin/
│       └── project-structure.md
├── java/                       # Java / JVM
│   ├── micronaut/
│   │   └── project-structure.md
│   ├── quarkus/
│   │   └── project-structure.md
│   └── spring/
│       └── project-structure.md
├── microservice-pattern/       # Padrões de Microsserviços
│   ├── microservice-patterns.md
│   └── patterns/
│       ├── api-composition.md
│       ├── api-gateway-bff.md
│       ├── backpressure.md
│       ├── blue-green.md
│       ├── bulkhead.md
│       ├── canary-release.md
│       ├── cdc.md
│       ├── circuit-breaker.md
│       ├── composicao-resiliencia.md
│       ├── configuracao-externa.md
│       ├── cqrs.md
│       ├── dlq.md
│       ├── event-sourcing.md
│       ├── feature-flags.md
│       ├── health-checks.md
│       ├── idempotencia.md
│       ├── observabilidade.md
│       ├── outbox-pattern.md
│       ├── rate-limiter.md
│       ├── retry.md
│       ├── saga.md
│       ├── service-discovery.md
│       ├── shadow-deployment.md
│       ├── sharding-partitioning.md
│       ├── sidecar.md
│       ├── strangler-fig.md
│       └── timeout.md
├── k8s/                        # Kubernetes
│   ├── README.md
│   ├── best-practices/
│   │   ├── 01-workloads-pod-design.md
│   │   ├── 02-security.md
│   │   ├── 03-networking.md
│   │   ├── 04-resource-management.md
│   │   ├── 05-observability.md
│   │   ├── 06-cicd-gitops.md
│   │   ├── 07-storage.md
│   │   ├── 08-high-availability.md
│   │   ├── 09-configuration.md
│   │   └── 10-cluster-operations.md
│   └── certifications/
│       ├── KCNA.md
│       ├── KCSA.md
│       ├── CKAD.md
│       ├── CKA.md
│       ├── CKS.md
│       └── KUBESTRONAUT.md
├── quality-engineering/        # Qualidade & Testes
│   ├── testing-strategy.md
│   ├── test-automation-patterns.md
│   ├── code-review-quality.md
│   ├── security-testing.md
│   ├── performance-testing.md
│   └── observability-quality.md
├── solid/                      # Princípios SOLID
│   └── solid-principles.md
├── system-design/              # System Design
│   ├── 01-load-balancing/
│   ├── 02-caching/
│   ├── 03-cdn/
│   ├── 04-dns/
│   ├── 05-reverse-proxy/
│   ├── 06-api-gateway/
│   ├── 07-database-indexing/
│   ├── 08-database-replication/
│   ├── 09-database-sharding/
│   ├── 10-sql-vs-nosql/
│   ├── 11-cap-theorem/
│   ├── 12-acid-vs-base/
│   ├── 13-consistent-hashing/
│   ├── 14-message-queues/
│   ├── 15-pub-sub/
│   ├── 16-rate-limiting/
│   ├── 17-circuit-breaker/
│   ├── 18-service-discovery/
│   ├── 19-heartbeat-health-checks/
│   ├── 20-leader-election/
│   ├── 21-consensus-algorithms/
│   ├── 22-gossip-protocol/
│   ├── 23-bloom-filters/
│   ├── 24-websockets-long-polling-sse/
│   ├── 25-rest-graphql-grpc/
│   ├── 26-event-driven-architecture/
│   ├── 27-cqrs/
│   ├── 28-event-sourcing/
│   ├── 29-saga-pattern/
│   ├── 30-outbox-pattern/
│   ├── 31-back-of-the-envelope-estimation/
│   ├── 32-data-partitioning-strategies/
│   ├── 33-replication-strategies/
│   ├── 34-idempotency/
│   ├── 35-oauth-jwt-authentication/
│   ├── 36-url-shortener/
│   ├── 37-pastebin/
│   ├── 38-twitter-social-feed/
│   ├── 39-instagram-photo-sharing/
│   ├── 40-whatsapp-chat-system/
│   ├── 41-youtube-netflix-video-streaming/
│   ├── 42-uber-ride-sharing/
│   ├── 43-google-maps-navigation/
│   └── system-design-big-techs.md
└── terraform/                  # Terraform / IaC
    └── aws/
        ├── README.md
        ├── best-practices.md
        ├── modules.md
        ├── project-structure.md
        ├── security.md
        ├── state-management.md
        └── testing.md
```

## Conteúdo

| Seção | Descrição |
|-------|-----------|
| **API** | Boas práticas para design de APIs REST, GraphQL e gRPC |
| **AWS** | Well-Architected, Serverless, Security, Networking, Containers, Databases, Observability, Cost Optimization, CI/CD, Disaster Recovery, Storage, Multi-Account |
| **Data Engineering** | Data Engineering & Architecture na AWS — Fundamentos, Storage (S3/Redshift/DynamoDB/Aurora), Processing (Glue/EMR/Kinesis/MSK), Integração (CDC/DMS/EventBridge), Governance (Lake Formation/DataZone/Quality) e Arquiteturas de Referência |
| **DDD** | Domain-Driven Design — Strategic Design, Tactical Design (Building Blocks), Context Mapping, Anti-Patterns e Integração com Arquiteturas |
| **Design Patterns** | Padrões criacionais, estruturais, comportamentais e arquiteturais |
| **Java** | Estrutura de projetos para Spring Boot, Quarkus e Micronaut |
| **Go** | Estrutura de projetos com Gin |
| **K8s** | Kubernetes — 10 guias de boas práticas (Workloads, Security, Networking, Resources, Observability, CI/CD GitOps, Storage, HA, Config, Cluster Ops) + 6 guias de certificações (KCNA, KCSA, CKAD, CKA, CKS, Kubestronaut) |
| **Microservice Patterns** | 27 padrões essenciais para arquitetura de microsserviços |
| **Quality Engineering** | Estratégia de testes, padrões de automação, code review, security testing, performance testing e observability |
| **SOLID** | Princípios SOLID — SRP, OCP, LSP, ISP, DIP com heurísticas, anti-patterns e diretrizes para code review assistido por AI |
| **System Design** | 43 tópicos de design de sistemas — conceitos fundamentais e case studies |
| **Terraform** | IaC com Terraform para AWS — estrutura, módulos, segurança e testes |

## Como usar com o GitHub Copilot

Cada guia de tecnologia (`project-structure.md`) já contém instruções para o Copilot embutidas. Copie o conteúdo do guia da tecnologia desejada para `.github/copilot-instructions.md` no repositório alvo:

| Tecnologia | Arquivo |
|------------|---------|
| Spring Boot | `.docs/java/spring/project-structure.md` |
| Quarkus | `.docs/java/quarkus/project-structure.md` |
| Micronaut | `.docs/java/micronaut/project-structure.md` |
| Go (Gin) | `.docs/golang/gin/project-structure.md` |
| Terraform (AWS) | `.docs/terraform/aws/` |
| AWS (Best Practices) | `.docs/aws/` |
| Data Engineering (AWS) | `.docs/data-engineering/` |
| DDD | `.docs/ddd/` |
| K8s | `.docs/k8s/` |
| Quality Engineering | `.docs/quality-engineering/` |
| SOLID | `.docs/solid/` |
