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
│   ├── README.md
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
│   └── cost-optimization/
│       └── cost-optimization-guide.md
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
├── microservice-pattern/       # Padrões de Microsserviços
│   ├── microservice-patterns.md
│   └── patterns/
│       ├── api-composition.md
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
├── quality-engineering/        # Qualidade & Testes
│   └── testing-strategy.md
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
| **AWS** | Well-Architected, Serverless, Security, Networking, Containers, Databases, Observability, Cost Optimization |
| **Design Patterns** | Padrões criacionais, estruturais, comportamentais e arquiteturais |
| **Java** | Estrutura de projetos para Spring Boot, Quarkus e Micronaut |
| **Go** | Estrutura de projetos com Gin |
| **Microservice Patterns** | 26 padrões essenciais para arquitetura de microsserviços |
| **Quality Engineering** | Estratégias de testes e qualidade |
| **SOLID** | Princípios SOLID aplicados |
| **System Design** | 30 conceitos fundamentais de design de sistemas |
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
