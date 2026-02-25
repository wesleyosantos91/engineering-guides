# Engineering Guides

Guias pessoais de organização, convenções e padrões de código por tecnologia.

## Estrutura

```
.docs/
├── adrs/                       # Architecture Decision Records (ADRs)
│   ├── 01-adr-foundations.md
│   ├── 02-adr-writing-guide.md
│   ├── 03-adr-governance-at-scale.md
│   └── 04-adr-templates-examples.md
├── api/                        # API Design
│   ├── api-guide.md
│   ├── graphql/
│   │   └── graphql-best-practices.md
│   ├── grpc/
│   │   └── grpc-best-practices.md
│   └── rest/
│       └── rest-best-practices.md
├── aws/                        # AWS Best Practices & Well-Architected
│   ├── README.md
│   ├── 01-well-architected-framework.md
│   ├── 02-security-best-practices.md
│   ├── 03-compliance-checklist.md
│   ├── 04-containers-best-practices.md
│   ├── 05-networking-best-practices.md
│   ├── 06-storage-best-practices.md
│   ├── 07-databases-best-practices.md
│   ├── 08-serverless-best-practices.md
│   ├── 09-serverless-patterns.md
│   ├── 10-observability-best-practices.md
│   ├── 11-cicd-best-practices.md
│   ├── 12-cost-optimization-guide.md
│   ├── 13-multi-account-strategy.md
│   └── 14-disaster-recovery.md
├── data-engineering/            # Data Engineering (AWS-focused)
│   ├── README.md
│   ├── 01-data-architecture-foundations.md
│   ├── 02-data-storage.md
│   ├── 03-data-processing.md
│   ├── 04-data-integration.md
│   ├── 05-data-governance.md
│   └── 06-architecture-patterns.md
├── ddd/                        # Domain-Driven Design
│   ├── README.md
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
├── java/                       # Java
│   ├── micronaut/
│   │   └── project-structure.md
│   ├── quarkus/
│   │   └── project-structure.md
│   └── spring/
│       └── project-structure.md
├── k8s/                        # Kubernetes
│   ├── README.md
│   ├── best-practices/
│   └── certifications/
├── microservice-patterns/      # Padrões de Microsserviços
│   ├── README.md
│   ├── 01-circuit-breaker.md
│   ├── 02-retry.md
│   ├── 03-timeout.md
│   ├── 04-rate-limiter.md
│   ├── 05-bulkhead.md
│   ├── 06-composicao-resiliencia.md
│   ├── 07-idempotencia.md
│   ├── 08-dlq.md
│   ├── 09-saga.md
│   ├── 10-api-composition.md
│   ├── 11-cqrs.md
│   ├── 12-event-sourcing.md
│   ├── 13-outbox-pattern.md
│   ├── 14-cdc.md
│   ├── 15-sharding-partitioning.md
│   ├── 16-backpressure.md
│   ├── 17-service-discovery.md
│   ├── 18-configuracao-externa.md
│   ├── 19-health-checks.md
│   ├── 20-api-gateway-bff.md
│   ├── 21-sidecar.md
│   ├── 22-strangler-fig.md
│   ├── 23-blue-green.md
│   ├── 24-canary-release.md
│   ├── 25-feature-flags.md
│   ├── 26-shadow-deployment.md
│   └── 27-observabilidade.md
├── quality-engineering/        # Qualidade & Testes
│   ├── README.md
│   ├── 01-testing-strategy.md
│   ├── 02-test-automation-patterns.md
│   ├── 03-performance-testing.md
│   ├── 04-security-testing.md
│   ├── 05-code-review-quality.md
│   └── 06-observability-quality.md
├── observability/              # Observabilidade Avançada
│   ├── README.md
│   ├── 01-observability-foundations.md
│   ├── 02-distributed-tracing.md
│   ├── 03-slos-slis-error-budgets.md
│   ├── 04-opentelemetry.md
│   └── 05-observability-driven-development.md
├── performance/                # Performance Engineering
│   ├── README.md
│   ├── 01-performance-foundations.md
│   ├── 02-profiling-and-diagnostics.md
│   ├── 03-capacity-planning.md
│   ├── 04-benchmarking.md
│   └── 05-latency-budgets.md
├── tech-leadership/            # Tech Leadership & Staff+ Competencies
│   ├── README.md
│   ├── 01-leadership-foundations.md
│   ├── 02-technical-vision.md
│   ├── 03-organizational-influence.md
│   ├── 04-cross-team-leadership.md
│   ├── 05-mentoring-seniors.md
│   ├── 06-business-acumen.md
│   ├── 07-written-communication.md
│   └── 08-failure-leadership.md
├── tech-strategy/              # Technical Strategy & Roadmaps
│   ├── README.md
│   ├── 01-tech-strategy-foundations.md
│   ├── 02-build-vs-buy.md
│   ├── 03-migration-strategies.md
│   ├── 04-tech-debt-quantification.md
│   └── 05-tech-roadmaps.md
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
    ├── README.md
    └── aws/
        ├── README.md
        ├── 01-project-structure.md
        ├── 02-best-practices.md
        ├── 03-testing.md
        ├── 04-modules.md
        ├── 05-state-management.md
        └── 06-security.md
```

## Conteúdo

| Seção | Descrição |
|-------|-----------|
| **ADRs** | Architecture Decision Records — fundamentos, guia de escrita, governança em escala, templates e exemplos reais |
| **API** | Boas práticas para design de APIs REST, GraphQL e gRPC |
| **AWS** | Well-Architected, Serverless, Security, Networking, Containers, Databases, Observability, Cost Optimization, CI/CD, Disaster Recovery, Storage, Multi-Account |
| **Data Engineering** | Arquitetura de dados na AWS — storage, processing, integração, governança e padrões de arquitetura |
| **DDD** | Domain-Driven Design — strategic design, tactical design, context mapping, anti-patterns, integração com arquitetura |
| **Design Patterns** | Padrões criacionais, estruturais, comportamentais e arquiteturais |
| **Java** | Estrutura de projetos para Spring Boot, Quarkus e Micronaut |
| **Go** | Estrutura de projetos com Gin |
| **Kubernetes** | Best practices e certificações Kubernetes |
| **Microservice Patterns** | 27 padrões essenciais para arquitetura de microsserviços |
| **Quality Engineering** | Estratégia de testes, automação, performance testing, security testing, code review e observabilidade aplicada à qualidade |
| **Observability** | Observabilidade avançada — fundamentos, distributed tracing, SLOs/SLIs/Error Budgets, OpenTelemetry, observability-driven development |
| **Performance Engineering** | Performance engineering — fundamentos e métricas, profiling e diagnóstico, capacity planning, benchmarking e load testing, latency budgets e performance SLOs |
| **Tech Leadership** | Competências Staff+ — fundamentos de liderança técnica, visão técnica, influência organizacional, liderança cross-team, mentoria de seniors, business acumen, comunicação escrita, liderança em falhas |
| **Tech Strategy** | Estratégia técnica — fundamentos e Tech Radar, build vs buy, estratégias de migração, quantificação de tech debt, construção e execução de roadmaps |
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
| Data Engineering | `.docs/data-engineering/` |
| DDD | `.docs/ddd/` |
| Kubernetes | `.docs/k8s/` |
| ADRs | `.docs/adrs/` |
| Observability | `.docs/observability/` |
| Performance Engineering | `.docs/performance/` |
| Tech Leadership | `.docs/tech-leadership/` |
| Tech Strategy | `.docs/tech-strategy/` |
