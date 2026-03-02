# System Design — Engineering Challenges

> **Programa de especialização progressiva em System Design** — do conceito à implementação.
> Cada desafio exige: **ADR documentando decisões**, **diagrama DrawIO** e **implementação em Go + Java**.

**Referência:** [System Design — 50 Conceitos Essenciais](../../.docs/SYSTEM-DESIGN/README.md)
**ADR Guide:** [Architecture Decision Records](../../.docs/adrs/README.md)

---

## Filosofia

```
Conceito → ADR → Diagrama DrawIO → Implementação Go + Java → Testes → Review
```

Cada desafio segue o ciclo completo:
1. **Estude** o conceito na documentação de referência
2. **Escreva um ADR** documentando decisões de design, trade-offs e alternativas
3. **Desenhe no DrawIO** a arquitetura com componentes, fluxos e interações
4. **Implemente em Go** usando frameworks idiomáticos (net/http, gRPC, etc.)
5. **Implemente em Java** usando Spring Boot / Quarkus / Java puro
6. **Teste** com cenários realistas de carga e falha

---

## Estrutura de Entregáveis por Desafio

```
system-design-challenges/
├── level-NN-topic-name/
│   ├── README.md                          ← Descrição e instruções
│   ├── docs/
│   │   ├── adrs/
│   │   │   └── ADR-001-design-decision.md ← ADR no formato MADR
│   │   └── diagrams/
│   │       └── architecture.drawio        ← Diagrama do sistema
│   ├── go/
│   │   ├── go.mod
│   │   ├── cmd/
│   │   ├── internal/
│   │   └── Makefile
│   └── java/
│       ├── pom.xml
│       ├── src/
│       └── mvnw
```

---

## Mapa de Progressão

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║                     SYSTEM DESIGN — MAPA DE PROGRESSÃO                             ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                    ║
║  PARTE I — Building Blocks (implemente cada componente)                            ║
║  ┌───────────────────────────────────────────────────────────────────────────────┐  ║
║  │  00. Foundations & Setup                                                     │  ║
║  │  01. Load Balancing & Reverse Proxy          │  02. Caching & CDN            │  ║
║  │  03. DNS & API Gateway                       │  04. Database Indexing & Repl. │  ║
║  │  05. Database Sharding & SQL vs NoSQL        │  06. Distributed Theory       │  ║
║  │  07. Messaging Systems (Queues & Pub/Sub)    │  08. Resilience Patterns      │  ║
║  │  09. Service Coordination                    │  10. Distributed Algorithms   │  ║
║  │  11. Communication Protocols                 │                               │  ║
║  └───────────────────────────────────────────────────────────────────────────────┘  ║
║                                          │                                         ║
║                                          ▼                                         ║
║  PARTE II — Padrões Arquiteturais (combine building blocks)                        ║
║  ┌───────────────────────────────────────────────────────────────────────────────┐  ║
║  │  12. Event-Driven Architecture & CQRS        │  13. Data Consistency Patterns │  ║
║  │  14. Estimation, Partitioning & Auth                                         │  ║
║  └───────────────────────────────────────────────────────────────────────────────┘  ║
║                                          │                                         ║
║                                          ▼                                         ║
║  PARTE III — Classic System Designs (designe sistemas completos)                   ║
║  ┌───────────────────────────────────────────────────────────────────────────────┐  ║
║  │  15. URL Shortener          │  16. Pastebin                                  │  ║
║  │  17. Twitter / Social Feed  │  18. Instagram / Photo Sharing                 │  ║
║  │  19. WhatsApp / Chat System │  20. YouTube/Netflix / Video Streaming         │  ║
║  │  21. Uber / Ride Sharing    │  22. Google Maps / Navigation                  │  ║
║  └───────────────────────────────────────────────────────────────────────────────┘  ║
║                                          │                                         ║
║                                          ▼                                         ║
║  CAPSTONE                                                                          ║
║  ┌───────────────────────────────────────────────────────────────────────────────┐  ║
║  │  23. Capstone — Distributed Platform (combine tudo)                          │  ║
║  └───────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                    ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

---

## Trilha Completa

### Parte I — Building Blocks

| Level | Desafio | Conceitos SD | ADRs | DrawIO | Impl. |
|:-----:|---------|-------------|:----:|:------:|:-----:|
| 00 | [Foundations & Setup](00-foundations.md) | Setup, tooling | — | — | Setup |
| 01 | [Load Balancing & Reverse Proxy](01-load-balancing-reverse-proxy.md) | 01, 05 | 1 | 1 | Go + Java |
| 02 | [Caching & CDN](02-caching-cdn.md) | 02, 03 | 1 | 1 | Go + Java |
| 03 | [DNS & API Gateway](03-dns-api-gateway.md) | 04, 06 | 1 | 1 | Go + Java |
| 04 | [Database Indexing & Replication](04-database-indexing-replication.md) | 07, 08 | 1 | 1 | Go + Java |
| 05 | [Database Sharding & SQL vs NoSQL](05-database-sharding.md) | 09, 10 | 1 | 1 | Go + Java |
| 06 | [Distributed Theory](06-distributed-theory.md) | 11, 12, 13 | 1 | 1 | Go + Java |
| 07 | [Messaging Systems](07-messaging-systems.md) | 14, 15 | 1 | 1 | Go + Java |
| 08 | [Resilience Patterns](08-resilience-patterns.md) | 16, 17 | 1 | 1 | Go + Java |
| 09 | [Service Coordination](09-service-coordination.md) | 18, 19, 20 | 1 | 1 | Go + Java |
| 10 | [Distributed Algorithms](10-distributed-algorithms.md) | 21, 22, 23 | 1 | 1 | Go + Java |
| 11 | [Communication Protocols](11-communication-protocols.md) | 24, 25 | 1 | 1 | Go + Java |

### Parte II — Padrões Arquiteturais

| Level | Desafio | Conceitos SD | ADRs | DrawIO | Impl. |
|:-----:|---------|-------------|:----:|:------:|:-----:|
| 12 | [Event Architecture & CQRS](12-event-architecture.md) | 26, 27, 28 | 2 | 1 | Go + Java |
| 13 | [Data Consistency Patterns](13-data-consistency-patterns.md) | 29, 30, 34 | 2 | 1 | Go + Java |
| 14 | [Estimation, Partitioning & Auth](14-estimation-partitioning-auth.md) | 31, 32, 33, 35 | 1 | 1 | Go + Java |

### Parte III — Classic System Designs

| Level | Desafio | Conceito SD | ADRs | DrawIO | Impl. |
|:-----:|---------|-------------|:----:|:------:|:-----:|
| 15 | [URL Shortener](15-url-shortener.md) | 36 | 3 | 2 | Go + Java |
| 16 | [Pastebin](16-pastebin.md) | 37 | 3 | 2 | Go + Java |
| 17 | [Twitter / Social Feed](17-twitter-social-feed.md) | 38 | 4 | 2 | Go + Java |
| 18 | [Instagram / Photo Sharing](18-instagram-photo-sharing.md) | 39 | 4 | 2 | Go + Java |
| 19 | [WhatsApp / Chat System](19-whatsapp-chat-system.md) | 40 | 4 | 2 | Go + Java |
| 20 | [YouTube/Netflix / Video Streaming](20-youtube-netflix-streaming.md) | 41 | 4 | 2 | Go + Java |
| 21 | [Uber / Ride Sharing](21-uber-ride-sharing.md) | 42 | 4 | 3 | Go + Java |
| 22 | [Google Maps / Navigation](22-google-maps-navigation.md) | 43 | 4 | 3 | Go + Java |

### Capstone

| Level | Desafio | ADRs | DrawIO | Impl. |
|:-----:|---------|:----:|:------:|:-----:|
| 23 | [Capstone — Distributed Platform](23-capstone-distributed-platform.md) | 6+ | 4+ | Go + Java |

---

## Stack Tecnológico

### Go

| Componente | Tecnologia |
|-----------|-----------|
| **HTTP Server** | `net/http` / Gin / Chi |
| **gRPC** | `google.golang.org/grpc` |
| **WebSocket** | `gorilla/websocket` / `nhooyr.io/websocket` |
| **Database** | `database/sql` + `pgx` / GORM |
| **Cache** | `go-redis/redis` |
| **Message Queue** | `segmentio/kafka-go` / `rabbitmq/amqp091-go` |
| **Testing** | `testing` stdlib + `testify` |
| **Observability** | OpenTelemetry Go SDK |

### Java

| Componente | Tecnologia |
|-----------|-----------|
| **Framework** | Spring Boot 3.x / Quarkus 3.x |
| **HTTP** | Spring WebFlux / JAX-RS |
| **gRPC** | `grpc-java` / Spring gRPC |
| **WebSocket** | Spring WebSocket / Jakarta WebSocket |
| **Database** | Spring Data JPA / Hibernate / JDBC |
| **Cache** | Spring Cache + Redis (Lettuce) |
| **Message Queue** | Spring Kafka / Spring AMQP |
| **Testing** | JUnit 5 + Testcontainers |
| **Observability** | Micrometer + OpenTelemetry |

---

## Template de ADR (MADR)

Cada desafio exige ADRs seguindo o formato:

```markdown
# ADR-NNN: [Título da Decisão]

Date: YYYY-MM-DD
Status: Proposed | Accepted | Deprecated
Owner: [seu nome]

## Context and Problem Statement

[Descreva o problema e o contexto]

## Decision Drivers

* [Driver 1]
* [Driver 2]

## Considered Options

1. [Opção A]
2. [Opção B]
3. [Opção C]

## Decision Outcome

Chosen option: "[Opção escolhida]", because [justificativa].

### Consequences

#### Good
- [Consequência positiva]

#### Bad
- [Consequência negativa]

## Pros and Cons of the Options

### [Opção A]
- ✅ Pro 1
- ❌ Con 1

### [Opção B]
- ✅ Pro 1  
- ❌ Con 1
```

**Referência completa:** [04-adr-templates-examples.md](../../.docs/adrs/04-adr-templates-examples.md)

---

## Template DrawIO

Cada diagrama `.drawio` deve conter:

1. **System Context** — visão de alto nível (usuários, serviços externos)
2. **Component Diagram** — componentes internos e interações
3. **Data Flow** — fluxo de dados entre componentes
4. **Sequence Diagram** — fluxos críticos (happy path + failure)

Convenções:
- Use cores consistentes: 🔵 azul = serviços, 🟢 verde = databases, 🟡 amarelo = caches, 🔴 vermelho = external
- Inclua notas com protocolos (HTTP, gRPC, TCP)
- Indique latências esperadas nas setas
- Marque pontos de falha com ⚠️

---

## Pré-requisitos

- Levels completos: Design Patterns, DDD, Databases, API Engineering, Backend
- Go 1.24+ instalado
- Java 25+ (JDK) instalado
- Docker + Docker Compose
- DrawIO (VS Code extension ou desktop app)
- Ferramentas: `make`, `maven`, `protoc` (para gRPC)

---

## Como Usar

1. **Estude o conceito** na documentação de referência (`.docs/SYSTEM-DESIGN/`)
2. **Leia o desafio** do nível correspondente
3. **Crie o ADR** antes de codificar — documente decisões
4. **Desenhe no DrawIO** — visualize antes de implementar
5. **Implemente em Go** — foque no idiomático
6. **Implemente em Java** — use Spring Boot ou Quarkus
7. **Teste** — unitários, integração e (quando aplicável) carga
8. **Review** — valide contra os critérios de aceite

---

## Definição de Pronto (DoD) Global

- [ ] ADR(s) escritos e revisados
- [ ] Diagrama DrawIO com todas as views necessárias
- [ ] Implementação Go compilando e testada
- [ ] Implementação Java compilando e testada
- [ ] README do desafio com instruções de setup e execução
- [ ] Testes passando (unitários + integração)
- [ ] Commit semântico: `feat(system-design-NN): <descrição>`
