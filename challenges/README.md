# Zero to Hero — Engineering Challenges

> **Programa de especialização progressiva** organizado em 6 fases com dependências claras.
> Siga as fases na ordem para construir conhecimento sólido e incremental.

---

## Cronograma de Estudo

> As trilhas estão organizadas em fases que respeitam pré-requisitos reais.
> **Dentro de cada fase**, as trilhas podem ser estudadas em paralelo.
> **Entre fases**, respeite a ordem — cada fase depende das anteriores.

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║                          MAPA DE PROGRESSÃO DE ESTUDOS                             ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                    ║
║  FASE 1 ─ Fundamentos de Código (linguagem pura, sem frameworks)                   ║
║  ┌─────────────────────────┐    ┌─────────────────────────┐                        ║
║  │  1. Design Patterns     │───▶│  2. DDD                 │                        ║
║  │     8 levels · ~16 sem  │    │     7 levels · ~14 sem  │                        ║
║  │     Payment Processor   │    │     MerchantHub         │                        ║
║  └─────────────────────────┘    └─────────────────────────┘                        ║
║              │                            │                                        ║
║              ▼                            ▼                                        ║
║  FASE 2 ─ Dados e Comunicação                                                      ║
║  ┌─────────────────────────┐    ┌─────────────────────────┐                        ║
║  │  3. Databases           │    │  4. API Engineering     │  ◀── paralelo          ║
║  │     8 levels · ~16 sem  │    │     8 levels · ~16 sem  │                        ║
║  │     StreamX Platform    │    │     Online Marketplace  │                        ║
║  └─────────────────────────┘    └─────────────────────────┘                        ║
║              │                            │                                        ║
║              ▼                            ▼                                        ║
║  FASE 3 ─ Backend Completo (frameworks + produção)                                  ║
║  ┌──────────────────────────────────────────────────────────┐                      ║
║  │  5. Backend Challenge                                    │                      ║
║  │     9 levels · ~18 sem · Digital Wallet                  │                      ║
║  │     Go (Gin) · Spring Boot · Quarkus · Micronaut · JEE  │                      ║
║  └──────────────────────────────────────────────────────────┘                      ║
║              │                                                                     ║
║              ▼                                                                     ║
║  FASE 4 ─ Qualidade e Observabilidade                                               ║
║  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐                 ║
║  │ 6. Quality       │  │ 7. Observability │  │ 8. Performance   │                 ║
║  │    8 levels      │─▶│    6 levels      │─▶│    10 levels     │                 ║
║  │    Digital Wallet│  │    Order Proc.   │  │    Wallet+Orders │                 ║
║  └──────────────────┘  └──────────────────┘  └──────────────────┘                 ║
║              │                                        │                            ║
║              ▼                                        ▼                            ║
║  FASE 5 ─ Arquitetura Distribuída                                                   ║
║  ┌──────────────────────────────────────────────────────────┐                      ║
║  │  9. Microservice Patterns                                │                      ║
║  │     10 levels · ~20 sem · RideFlow Platform              │                      ║
║  │     27 padrões: resiliência, consistência, eventos,      │                      ║
║  │     infraestrutura, deploy, observabilidade              │                      ║
║  └──────────────────────────────┬───────────────────────────┘                      ║
║                                 ▼                                                  ║
║  ┌──────────────────────────────────────────────────────────┐                      ║
║  │  10. System Design                                       │                      ║
║  │      24 levels · ~48 sem · MegaStore Platform            │                      ║
║  │      ADRs + DrawIO + Go + Java p/ cada design            │                      ║
║  │      Building Blocks → Classic Designs → Capstone        │                      ║
║  └──────────────────────────────────────────────────────────┘                      ║
║              │                                                                     ║
║              ▼                                                                     ║
║  ┌──────────────────────────────────────────────────────────┐                      ║
║  │  11. Data Engineering                                    │                      ║
║  │      13 levels · ~26 sem · Enterprise DataFlow           │                      ║
║  │      ADRs + DrawIO + Python + Go + Java                  │                      ║
║  │      Batch + Streaming + Lakehouse + dbt + Governance    │                      ║
║  └──────────────────────────────────────────────────────────┘                      ║
║              │                                                                     ║
║              ▼                                                                     ║
║  FASE 6 ─ Infraestrutura e Operações                                                ║
║  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐                 ║
║  │ 12. Terraform    │  │ 13. AWS Infra    │  │ 14. Kubernetes   │                 ║
║  │     8 levels     │─▶│     8 levels     │─▶│     17 levels    │                 ║
║  │     ShopFlow     │  │     OrderFlow    │  │     CloudShop    │                 ║
║  └──────────────────┘  └──────────────────┘  └──────────────────┘                 ║
║                                                                                    ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

---

## Trilhas por Fase

### Fase 1 — Fundamentos de Código

> Linguagem pura (Java 25 + Go 1.26), sem frameworks. Construa a base sólida.

| Ordem | Trilha | Domínio | Níveis | Por que primeiro? |
|:-----:|--------|---------|:------:|-------------------|
| 1 | [**Design Patterns**](design-patterns/README.md) | Payment Processor | 8 (0-7) | OOP, SOLID e GoF são pré-requisito para tudo |
| 2 | [**DDD**](ddd/README.md) | MerchantHub | 7 (0-6) | Modelagem de domínio depende de Design Patterns |

### Fase 2 — Dados e Comunicação

> Como persistir dados e como serviços se comunicam. Pode ser feito em paralelo.

| Ordem | Trilha | Domínio | Níveis | Por que agora? |
|:-----:|--------|---------|:------:|----------------|
| 3 | [**Databases**](databases/README.md) | StreamX | 8 (0-7) | SQL + NoSQL são base para qualquer backend |
| 4 | [**API Engineering**](api/README.md) | Online Marketplace | 8 (0-7) | REST, GraphQL, gRPC — protocolos de comunicação |

### Fase 3 — Backend Completo

> Combine padrões + dados + APIs em aplicações reais com frameworks.

| Ordem | Trilha | Domínio | Níveis | Por que agora? |
|:-----:|--------|---------|:------:|----------------|
| 5 | [**Backend Challenge**](backend/README.md) | Digital Wallet | 9 (0-8) | Integra tudo das fases 1-2 em 5 stacks diferentes |

### Fase 4 — Qualidade e Observabilidade

> Garanta que o que você constrói funciona, é observável e performático.

| Ordem | Trilha | Domínio | Níveis | Por que agora? |
|:-----:|--------|---------|:------:|----------------|
| 6 | [**Quality Engineering**](quality/README.md) | Digital Wallet | 8 (0-7) | Testes, quality gates — usa o mesmo domínio do Backend |
| 7 | [**Observability**](observability/README.md) | Order Processing | 6 (0-5) | Métricas, traces, logs — precisa de apps para observar |
| 8 | [**Performance**](performance/README.md) | Wallet + Orders | 10 (0-9) | Load testing, profiling — depende de Quality + Observability |

### Fase 5 — Arquitetura Distribuída

> Padrões avançados para sistemas distribuídos. Requer domínio de todas as fases anteriores.

| Ordem | Trilha | Domínio | Níveis | Por que agora? |
|:-----:|--------|---------|:------:|----------------|
| 9 | [**Microservice Patterns**](microservice-patterns/README.md) | RideFlow | 10 (0-9) | 27 padrões que dependem de Design Patterns, DDD, APIs, Observability |
| 10 | [**System Design**](system-design/README.md) | MegaStore | 24 (0-23) | ADRs + DrawIO + Go + Java para cada design clássico; integra TUDO |
| 11 | [**Data Engineering**](data-engineering/README.md) | Enterprise DataFlow | 13 (0-12) | ADRs + DrawIO + Python + Go + Java; batch, streaming, lakehouse, governance |

### Fase 6 — Infraestrutura e Operações

> Deploy, provisione e opere tudo que foi construído.

| Ordem | Trilha | Domínio | Níveis | Por que agora? |
|:-----:|--------|---------|:------:|----------------|
| 12 | [**Terraform**](terraform/README.md) | ShopFlow | 8 (0-7) | IaC foundations com LocalStack ($0) |
| 13 | [**AWS Infrastructure**](aws/README.md) | OrderFlow | 8 (0-7) | AWS services reais com Terraform + LocalStack |
| 14 | [**Kubernetes**](k8s/README.md) | CloudShop | 17 (0-16) | Orquestração + 5 certificações (KCNA→Kubestronaut) |

---

## Resumo do Cronograma

| Fase | Trilhas | Total Níveis | Foco |
|:----:|---------|:------------:|------|
| 1 | Design Patterns → DDD | 15 | Código puro, modelagem |
| 2 | Databases ∥ API Engineering | 16 | Dados, comunicação |
| 3 | Backend Challenge | 9 | Frameworks, produção |
| 4 | Quality → Observability → Performance | 24 | Garantia, monitoramento |
| 5 | Microservice Patterns → System Design → Data Engineering | 47 | Sistemas distribuídos, design clássico, engenharia de dados |
| 6 | Terraform → AWS → Kubernetes | 33 | Infraestrutura, operações |
| | **TOTAL** | **144 levels** | |

> **Legenda:** `→` = sequencial (respeitar ordem) · `∥` = paralelo (pode intercalar)

---

## Tempo Estimado e Progressão de Carreira

> Estimativas baseadas em **~2 semanas por nível**, estudando **~10-15h/semana** fora do trabalho.
> O tempo real varia conforme experiência prévia, dedicação e complexidade dos capstones.

### Visão Geral

```
╔══════════════════════════════════════════════════════════════════════════════════════════════════╗
║                              MAPA DE PROGRESSÃO DE CARREIRA                                    ║
╠══════════════════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                                ║
║  Fase 1-3 ──▶ ~64 semanas (~15 meses)                                                         ║
║  ┌───────────────────────────────────────────────────────────────────────────────────────┐      ║
║  │  Design Patterns (16 sem) → DDD (14 sem)                                             │      ║
║  │  + Databases (16 sem) ∥ API (16 sem) → Backend (18 sem)                               │      ║
║  │  Resultado: OOP, SOLID, GoF, DDD, backend multi-stack, SQL/NoSQL, REST/GraphQL/gRPC   │      ║
║  │  🎯 Nível: SENIOR — entrega e excelência no squad                                     │      ║
║  └───────────────────────────────────────────────────────────────────────────────────────┘      ║
║              │                                                                                 ║
║              ▼                                                                                 ║
║  Fase 1-4 ──▶ ~112 semanas (~26 meses / ~2.2 anos)                                            ║
║  ┌───────────────────────────────────────────────────────────────────────────────────────┐      ║
║  │  + Quality (16 sem) → Observability (12 sem) → Performance (20 sem)                   │      ║
║  │  Resultado: Test strategy, distributed tracing, SLOs/SLIs, load testing, profiling    │      ║
║  │  🎯 Nível: STAFF — alinhamento e escala entre times                                   │      ║
║  └───────────────────────────────────────────────────────────────────────────────────────┘      ║
║              │                                                                                 ║
║              ▼                                                                                 ║
║  Fase 1-5 ──▶ ~206 semanas (~48 meses / ~4 anos)                                              ║
║  ┌───────────────────────────────────────────────────────────────────────────────────────┐      ║
║  │  + Microservices (20 sem) → System Design (48 sem) → Data Engineering (26 sem)        │      ║
║  │  Resultado: 27 padrões distribuídos, 8 case studies, data pipelines completos          │      ║
║  │  🎯 Nível: PRINCIPAL — direção técnica de área/plataforma                              │      ║
║  └───────────────────────────────────────────────────────────────────────────────────────┘      ║
║              │                                                                                 ║
║              ▼                                                                                 ║
║  Fase 1-6 ──▶ ~272 semanas (~64 meses / ~5.2 anos)                                            ║
║  ┌───────────────────────────────────────────────────────────────────────────────────────┐      ║
║  │  + Terraform (16 sem) → AWS (16 sem) → Kubernetes (34 sem)                            │      ║
║  │  Resultado: IaC, AWS production-grade, K8s + 5 certificações (Kubestronaut)           │      ║
║  │  🎯 Nível: DISTINGUISHED — arquitetura e capacidade organizacional                    │      ║
║  └───────────────────────────────────────────────────────────────────────────────────────┘      ║
║              │                                                                                 ║
║              ▼                                                                                 ║
║  Fase 1-6 + Trilhas Transversais ──▶ ~320 semanas (~75 meses / ~6.2 anos)                     ║
║  ┌───────────────────────────────────────────────────────────────────────────────────────┐      ║
║  │  + Tech Leadership (7 sem) + Strategic Communication (6 sem)                          │      ║
║  │  + Contribuições open-source, palestras, publicações técnicas                         │      ║
║  │  Resultado: Visão técnica organizacional, thought leadership, estratégia de longo     │      ║
║  │  prazo, influência na indústria                                                       │      ║
║  │  🎯 Nível: FELLOW — estratégia tecnológica de longo prazo                             │      ║
║  └───────────────────────────────────────────────────────────────────────────────────────┘      ║
║                                                                                                ║
╚══════════════════════════════════════════════════════════════════════════════════════════════════╝
```

### Tabela Resumo

| Marco | Fases Concluídas | Níveis | Tempo Acumulado | Nível de Carreira | Competências-Chave |
|-------|:----------------:|:------:|:---------------:|:-----------------:|---------------------|
| 1 | Fases 1–3 | 40 | ~15 meses | **Senior** | OOP, SOLID, GoF, DDD, backend multi-stack (5 frameworks), SQL/NoSQL, REST/GraphQL/gRPC, segurança |
| 2 | Fases 1–4 | 64 | ~2.2 anos | **Staff** | Test strategy, quality gates, distributed tracing, SLOs/SLIs, load testing, profiling, capacity planning |
| 3 | Fases 1–5 | 111 | ~4 anos | **Principal** | System design end-to-end, 27 microservice patterns, ADRs, data pipelines batch/streaming, event-driven |
| 4 | Fases 1–6 | 144 | ~5.2 anos | **Distinguished** | IaC production-grade, AWS Well-Architected, Kubernetes Kubestronaut, arquitetura organizacional |
| 5 | Todas + Transversais | 144+ | ~6.2 anos | **Fellow** | Tech leadership, strategic communication, thought leadership, estratégia tecnológica de longo prazo |

### O que cada nível significa

| Nível | Escopo de Impacto | Definição | Características |
|-------|-------------------|-----------|-----------------|
| **Senior** | Squad | Entrega e excelência no squad | Projeta e implementa serviços completos, escolhe tecnologias com trade-offs documentados, mentora juniors/mids, ownership de features end-to-end |
| **Staff** | Cross-team | Alinhamento e escala entre times | Define padrões de qualidade e observabilidade para múltiplos times, garante consistência técnica, escreve RFCs, facilita decisões cross-squad |
| **Principal** | Área/plataforma | Direção técnica de área/plataforma | Projeta arquiteturas distribuídas, define roadmap técnico de área, resolve problemas ambíguos, escreve ADRs estratégicos, influencia direção de produto |
| **Distinguished** | Organização | Arquitetura e capacidade organizacional | Ownership de plataforma end-to-end (code → infra → observability → data), define capacidades organizacionais, elimina classes inteiras de problemas |
| **Fellow** | Indústria | Estratégia tecnológica de longo prazo | Define visão técnica de longo prazo, thought leadership, influência na indústria, referência técnica interna e externa, molda cultura de engenharia |

### Aceleradores

> Formas de reduzir o tempo total significativamente:

| Estratégia | Impacto |
|------------|---------|
| **Dedicação full-time** (bootcamp/sabbatical) | Reduz para **~2-3 anos** |
| **Experiência prévia** em alguma fase | Pule os níveis iniciais (00-foundations) |
| **Foco em uma linguagem** (só Java ou só Go) | Reduz ~30% do tempo por trilha |
| **Paralelize** Fase 2 agressivamente | Economiza ~4 meses |
| **Estudo em par/grupo** | Acelera capstones e code reviews |

> ⚠️ **Nota:** Os níveis de carreira são **referenciais** baseados nas competências desenvolvidas.
> Promoções reais dependem também de soft skills, impacto no negócio, liderança e contexto organizacional.
> Este programa garante a **profundidade técnica** — o pilar fundamental de qualquer Staff+ engineer.
>
> As trilhas transversais ([Tech Leadership](tech-leadership/README.md) e [Strategic Communication](strategic-communication/README.md))
> são **essenciais para Distinguished e Fellow** — competência técnica sozinha não escala sem influência e comunicação.

---

## Grafo de Dependências

```
Design Patterns ──┬──▶ DDD
                  │
                  ├──▶ Databases ──────┐
                  │                    ├──▶ Backend ──┬──▶ Quality ──▶ Observability ──▶ Performance
                  └──▶ API Engineering ┘             │
                                                     └──▶ Microservice Patterns ──▶ System Design ──▶ Data Engineering
                                                                                                          │
                                                     Terraform ──▶ AWS Infra ──▶ Kubernetes ◀─────────────┘
```

---

## Estrutura do Repositório

```
challenges/
├── README.md                          ← você está aqui
│
├── design-patterns/                   ← Fase 1: Design Patterns (Pure Language)
│   ├── README.md
│   ├── 00-oop-foundations.md
│   ├── 01-solid-principles.md
│   ├── 02-creational-patterns.md
│   ├── 03-structural-patterns.md
│   ├── 04-behavioral-patterns.md
│   ├── 05-architectural-patterns.md
│   ├── 06-best-practices-integration.md
│   └── 07-capstone-project.md
│
├── ddd/                               ← Fase 1: Domain-Driven Design (Pure Language)
│   ├── README.md
│   ├── 00-ddd-foundations.md
│   ├── 01-strategic-design.md
│   ├── 02-tactical-design.md
│   ├── 03-context-mapping.md
│   ├── 04-anti-patterns.md
│   ├── 05-architecture-integration.md
│   └── 06-capstone-merchanthub.md
│
├── databases/                         ← Fase 2: Databases (SQL + NoSQL)
│   ├── README.md
│   ├── 00-database-foundations.md
│   ├── 01-sql-modeling.md
│   ├── 02-sql-advanced.md
│   ├── 03-sql-operations.md
│   ├── 04-nosql-document-keyvalue.md
│   ├── 05-nosql-wide-column-graph.md
│   ├── 06-scalability-patterns.md
│   └── 07-capstone-polyglot.md
│
├── api/                               ← Fase 2: API Engineering (REST + GraphQL + gRPC)
│   ├── README.md
│   ├── 00-http-foundations.md
│   ├── 01-rest-api.md
│   ├── 02-rest-advanced.md
│   ├── 03-graphql.md
│   ├── 04-graphql-advanced.md
│   ├── 05-grpc.md
│   ├── 06-grpc-advanced.md
│   └── 07-capstone-multi-paradigm.md
│
├── backend/                           ← Fase 3: Backend Multi-Framework
│   ├── README.md
│   ├── 00-foundations.md
│   ├── 01-crud-architecture.md
│   ├── 02-persistence-migrations.md
│   ├── 03-quality-tests.md
│   ├── 04-observability-resilience.md
│   ├── 05-security-authz.md
│   ├── 06-async-messaging.md
│   ├── 07-production-deploy.md
│   └── 08-benchmark-comparative.md
│
├── quality/                           ← Fase 4: Quality Engineering
│   ├── README.md
│   ├── 00-quality-foundations.md
│   ├── 01-testing-strategy.md
│   ├── 02-test-automation-patterns.md
│   └── 03-performance-testing.md
│
├── observability/                     ← Fase 4: Observability
│   ├── README.md
│   ├── 00-observability-foundations.md
│   ├── 01-distributed-tracing.md
│   ├── 02-slos-slis-error-budgets.md
│   ├── 03-opentelemetry-deep-dive.md
│   ├── 04-observability-driven-development.md
│   └── 05-capstone-production-observability.md
│
├── performance/                       ← Fase 4: Performance Engineering
│   ├── README.md
│   ├── 00-performance-foundations.md
│   ├── 01-profiling-diagnostics.md
│   ├── 02-load-testing-k6.md
│   ├── 03-load-testing-jmeter.md
│   ├── 04-gatling-locust-tools.md
│   ├── 05-benchmarking.md
│   ├── 06-capacity-planning.md
│   ├── 07-latency-budgets.md
│   ├── 08-capstone-performance-platform.md
│   └── 09-dora-space-metrics.md
│
├── microservice-patterns/             ← Fase 5: Microservice Patterns
│   ├── README.md
│   ├── 00-microservice-foundations.md
│   ├── 01-resilience-circuit-breaker-retry-timeout.md
│   ├── 02-resilience-rate-limiter-bulkhead-composition.md
│   ├── 03-data-consistency-idempotency-dlq-saga.md
│   ├── 04-data-patterns-composition-cqrs-eventsourcing.md
│   ├── 05-event-driven-outbox-cdc-backpressure-sharding.md
│   ├── 06-infrastructure-discovery-config-health-gateway-sidecar.md
│   ├── 07-deployment-strangler-bluegreen-canary-flags-shadow.md
│   ├── 08-observability-metrics-logs-traces-slos.md
│   └── 09-capstone-rideflow-platform.md
│
├── system-design/                     ← Fase 5: System Design (ADR + DrawIO + Go + Java)
│   ├── README.md
│   ├── 00-foundations.md
│   ├── 01-load-balancing-reverse-proxy.md
│   ├── 02-caching-cdn.md
│   ├── 03-dns-api-gateway.md
│   ├── 04-database-indexing-replication.md
│   ├── 05-database-sharding.md
│   ├── 06-distributed-theory.md
│   ├── 07-messaging-systems.md
│   ├── 08-resilience-patterns.md
│   ├── 09-service-coordination.md
│   ├── 10-distributed-algorithms.md
│   ├── 11-communication-protocols.md
│   ├── 12-event-architecture.md
│   ├── 13-data-consistency-patterns.md
│   ├── 14-estimation-partitioning-auth.md
│   ├── 15-url-shortener.md
│   ├── 16-pastebin.md
│   ├── 17-twitter-social-feed.md
│   ├── 18-instagram-photo-sharing.md
│   ├── 19-whatsapp-chat-system.md
│   ├── 20-youtube-netflix-streaming.md
│   ├── 21-uber-ride-sharing.md
│   ├── 22-google-maps-navigation.md
│   └── 23-capstone-distributed-platform.md
│
├── data-engineering/                   ← Fase 5: Data Engineering (Python + Go + Java)
│   ├── README.md
│   ├── 00-foundations.md
│   ├── 01-data-lake-warehouse.md
│   ├── 02-data-modeling-formats.md
│   ├── 03-batch-processing.md
│   ├── 04-stream-processing.md
│   ├── 05-pipeline-orchestration.md
│   ├── 06-cdc.md
│   ├── 07-migration-integration.md
│   ├── 08-data-quality.md
│   ├── 09-governance.md
│   ├── 10-lakehouse.md
│   ├── 11-modern-data-stack.md
│   └── 12-capstone-dataflow.md
│
├── terraform/                         ← Fase 6: Terraform AWS (LocalStack)
│   ├── README.md
│   ├── 00-foundations.md
│   ├── 01-project-structure.md
│   ├── 02-best-practices.md
│   ├── 03-testing.md
│   ├── 04-modules.md
│   ├── 05-state-management.md
│   ├── 06-security.md
│   └── 07-capstone-shopflow.md
│
├── aws/                               ← Fase 6: AWS Infrastructure
│   ├── README.md
│   ├── 00-foundations.md
│   ├── 01-project-structure-networking.md
│   └── 02-compute-storage.md
│
└── k8s/                               ← Fase 6: Kubernetes + Certificações
    ├── README.md
    ├── 00-k8s-foundations.md
    ├── 01-workloads-pod-design.md
    ├── 02-security.md
    ├── 03-networking.md
    ├── 04-resource-management.md
    ├── 05-observability.md
    ├── 06-cicd-gitops.md
    ├── 07-storage.md
    ├── 08-high-availability.md
    ├── 09-configuration.md
    ├── 10-cluster-operations.md
    ├── 11-kcna.md
    ├── 12-ckad.md
    ├── 13-cka.md
    ├── 14-cks.md
    ├── 15-kcsa.md
    └── 16-kubestronaut.md
│
├── tech-leadership/                   ← Transversal: Tech Leadership & People Excellence
│   ├── README.md
│   ├── 00-foundations.md
│   ├── 01-senior-squad-excellence.md
│   ├── 02-staff-cross-team-alignment.md
│   ├── 03-principal-area-direction.md
│   ├── 04-distinguished-org-capability.md
│   ├── 05-fellow-technology-strategy.md
│   └── 06-capstone-leadership-portfolio.md
│
└── strategic-communication/           ← Transversal: Strategic Communication & Decision Making
    ├── README.md
    ├── 00-foundations.md
    ├── 01-senior-technical-communication.md
    ├── 02-staff-influence-persuasion.md
    ├── 03-principal-executive-communication.md
    ├── 04-distinguished-negotiation-change.md
    ├── 05-fellow-thought-leadership.md
    └── 06-capstone-communication-portfolio.md
```

---

## Filosofia

1. **Progressão baseada em dependências** — Cada fase constrói sobre as anteriores
2. **Domínio unificado por trilha** — Cada trilha usa um domínio de negócio consistente em todos os níveis
3. **Multi-linguagem** — Sempre Java + Go, explorando idiomas e trade-offs de cada linguagem
4. **Critérios de aceite claros** — Cada desafio tem checklist objetivo e verificável
5. **Portfólio real** — Os projetos resultantes são apresentáveis em entrevistas técnicas

---

## Como usar

1. **Siga a ordem das fases** — Cada fase depende das anteriores (veja o grafo de dependências)
2. **Dentro de cada fase**, trilhas marcadas com `∥` podem ser feitas em paralelo
3. **Siga a ordem dos níveis** dentro de cada trilha — cada nível depende do anterior
4. **Implemente nas duas linguagens** — Java 25 e Go 1.26 (ou stacks da trilha backend)
5. **Valide com os critérios de aceite** — Não avance sem completar o checklist
6. **Documente decisões** — Mantenha um `DECISIONS.md` com trade-offs e justificativas
