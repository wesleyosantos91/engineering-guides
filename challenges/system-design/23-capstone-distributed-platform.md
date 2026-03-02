# Level 23 — Capstone: Distributed E-Commerce Platform

> **Objetivo:** Projetar e implementar uma plataforma de e-commerce distribuída **end-to-end**
> que integra **todos** os conceitos dos levels 0–22. Este é o projeto final que demonstra
> domínio completo de System Design.

**Pré-requisito:** Todos os levels anteriores (0–22) completos.

---

## Contexto

Plataforma de e-commerce de grande escala — **"MegaStore"** — que suporta catálogo
de produtos, busca, carrinho, checkout, pagamento, entrega, notificações e
analytics em tempo real. O sistema é composto por múltiplos microserviços,
usa event-driven architecture e deve resistir a falhas parciais.

**Escala alvo:**
- **50M usuários** ativos mensais
- **500K pedidos/dia**
- **10M produtos** no catálogo
- **Black Friday:** 10× traffic spike
- **Downtime:** < 4 horas/ano (99.95% availability)
- **Latency P99:** < 500ms

---

## Parte 1 — ADRs (6 obrigatórios)

### ADR-001: Overall Architecture Style

**Arquivo:** `docs/adrs/ADR-001-architecture-style.md`

**Context:** Decidir a macroarquitetura da plataforma.

**Options:**
1. **Monolith modular** — um deploy, módulos internos
2. **Microservices** — serviços independentes, communication via API/events
3. **Hybrid** — core services separados, features complementares como módulos
4. **Cell-based architecture** — cells independentes por região/tenant

**Decision Drivers:**
- Escalabilidade independente por domínio
- Team autonomy (cada equipe own de 1-2 serviços)
- Complexidade operacional aceitável
- Resilience: falha em um serviço não derruba outros

---

### ADR-002: Inter-service Communication

**Options:**
1. **Synchronous REST** — simples, forte acoplamento temporal
2. **Async Events (Kafka)** — desacoplamento, eventual consistency
3. **gRPC** — eficiente, type-safe, mas acoplamento de schema
4. **Hybrid** — gRPC sync para queries, Kafka async para commands/events

---

### ADR-003: Data Architecture

**Options:**
1. **Shared database** — todos serviços usam um DB
2. **Database per service** — cada serviço tem seu DB (polyglot persistence)
3. **Shared schema** — DB compartilhado com schemas separados
4. **CQRS** — write models + read models separados

**Polyglot candidates:**
- Product Catalog: PostgreSQL + Elasticsearch (search)
- Shopping Cart: Redis
- Orders: PostgreSQL (ACID)
- Recommendations: Redis + in-memory
- Analytics: ClickHouse / TimescaleDB
- Media: MinIO/S3

---

### ADR-004: Checkout & Payment Saga

**Options:**
1. **Orchestration Saga** — central orchestrator coordena steps
2. **Choreography Saga** — serviços reagem a eventos
3. **Hybrid** — orchestration para checkout, choreography para notificações

**Saga steps:** Reserve Inventory → Process Payment → Create Order → Notify

---

### ADR-005: Observability Strategy

**Options:**
1. **ELK Stack** — logs centralizados (Elasticsearch + Logstash + Kibana)
2. **OpenTelemetry** — traces + metrics + logs unificados
3. **Prometheus + Grafana + Jaeger** — metrics + dashboards + tracing
4. **All-in-one** — OpenTelemetry collector → Prometheus + Jaeger + Loki

---

### ADR-006: Resilience & Traffic Management

**Options:**
1. **API Gateway** + rate limiting + circuit breaker
2. **Service Mesh** (Envoy sidecar) — observability + retry + circuit breaker
3. **Application-level resilience** — cada serviço implementa seus patterns
4. **Gateway + application-level** — defesa em profundidade

**Critérios de aceite:**
- [ ] 6 ADRs completos com Decision Drivers e Consequences
- [ ] Cada ADR referencia problemas reais de escalabilidade
- [ ] Trade-offs claros documentados

---

## Parte 2 — Diagramas DrawIO (4 obrigatórios)

### Diagrama 1: System Context (C4 Level 1)

**Arquivo:** `docs/diagrams/23-megastore-context.drawio`

- MegaStore system boundaries
- External actors: Customers, Admin, Payment Provider, Delivery Partner
- External systems: Email/SMS, CDN, Payment Gateway

### Diagrama 2: Container Diagram (C4 Level 2)

**Arquivo:** `docs/diagrams/23-megastore-containers.drawio`

```
┌──────────────────────────────────────────────────────────────────┐
│                        MegaStore Platform                        │
│                                                                  │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌──────────────────┐   │
│  │   Web   │  │  Mobile  │  │  Admin  │  │  API Gateway     │   │
│  │   SPA   │  │   App    │  │  Panel  │  │(rate limit+auth) │   │
│  └────┬────┘  └────┬─────┘  └────┬────┘  └────────┬─────────┘   │
│       └─────────────┴─────────────┴────────────────┘             │
│                              │                                    │
│  ┌───────────┬───────────┬───┴───────┬──────────┬────────────┐   │
│  │           │           │           │          │            │   │
│  │ Product   │ Cart      │ Order     │ Payment  │ Delivery   │   │
│  │ Catalog   │ Service   │ Service   │ Service  │ Service    │   │
│  │           │           │ (Saga     │          │            │   │
│  │ [PG+ES]  │ [Redis]   │  Orch.)   │ [PG]    │ [PG]      │   │
│  │           │           │ [PG]     │          │            │   │
│  └─────┬─────┴─────┬─────┴─────┬────┴────┬─────┴──────┬─────┘   │
│        │           │           │         │            │          │
│        └───────────┴─────┬─────┴─────────┴────────────┘          │
│                          │                                        │
│                    ┌─────▼─────┐                                  │
│                    │   Kafka   │                                  │
│                    │  (Events) │                                  │
│                    └─────┬─────┘                                  │
│                          │                                        │
│  ┌───────────┬───────────┼──────────┬──────────────┐             │
│  │           │           │          │              │             │
│  │Notifica-  │ Analytics │ Search   │ Recommen-    │             │
│  │tion Svc   │ Service   │ Indexer  │ dation Svc   │             │
│  │[Email/SMS]│[ClickHse] │[Elastic] │ [Redis+ML]   │             │
│  └───────────┴───────────┴──────────┴──────────────┘             │
└──────────────────────────────────────────────────────────────────┘
```

### Diagrama 3: Checkout Saga Sequence

**Arquivo:** `docs/diagrams/23-megastore-checkout-saga.drawio`

- Happy path: Cart → Validate → Reserve Inventory → Process Payment → Create Order → Notify
- Compensation: Payment failed → Release Inventory → Notify Error
- Timeout handling

### Diagrama 4: Data Flow & Event Architecture

**Arquivo:** `docs/diagrams/23-megastore-data-flow.drawio`

- Kafka topics e event flows
- CQRS read/write model separation
- Event sourcing para orders (opcional)
- Outbox pattern para reliable publishing

**Critérios de aceite:**
- [ ] C4 Context e Container diagrams
- [ ] Saga sequence com happy path e compensação
- [ ] Data flow com Kafka topics e event routing

---

## Parte 3 — Implementação

### Serviços a implementar (Go + Java)

Escolha **Go** para 3 serviços e **Java** para 3 serviços, demonstrando interoperabilidade polyglot:

| Serviço | Linguagem Recomendada | Justificativa |
|---|---|---|
| API Gateway | Go | Performance-critical, baixa latência |
| Product Catalog | Java (Spring Boot) | Spring Data JPA + Elasticsearch |
| Cart Service | Go | Simple data model, Redis-native |
| Order Service (Saga) | Java (Spring Boot) | Spring State Machine para saga |
| Payment Service | Go | Lightweight, external API integration |
| Notification Service | Java (Spring Boot) | Spring Integration, templates |

---

### 3.1 — API Gateway (Go)

```
gateway/
├── cmd/main.go
├── internal/
│   ├── proxy/
│   │   ├── router.go              ← Route matching + forwarding
│   │   └── load_balancer.go       ← Round-robin entre instâncias
│   ├── middleware/
│   │   ├── auth.go                ← JWT validation
│   │   ├── rate_limiter.go        ← Token bucket per user
│   │   ├── circuit_breaker.go     ← Per-service circuit breaker
│   │   ├── cors.go
│   │   ├── logging.go             ← Structured logging
│   │   └── tracing.go             ← OpenTelemetry span propagation
│   └── config/
│       └── routes.go              ← Route configuration
├── go.mod
└── Dockerfile
```

### 3.2 — Product Catalog (Java Spring Boot)

```
product-catalog/
├── src/main/java/.../
│   ├── ProductCatalogApplication.java
│   ├── domain/
│   │   ├── Product.java           ← JPA Entity
│   │   ├── Category.java
│   │   └── ProductSearchDocument.java  ← Elasticsearch document
│   ├── service/
│   │   ├── ProductService.java
│   │   └── SearchService.java     ← Elasticsearch queries
│   ├── repository/
│   │   ├── ProductRepository.java ← JPA
│   │   └── ProductSearchRepo.java ← Spring Data Elasticsearch
│   ├── controller/
│   │   └── ProductController.java
│   └── event/
│       └── ProductEventPublisher.java  ← Kafka producer
├── src/main/resources/
│   └── application.yml
├── pom.xml
└── Dockerfile
```

### 3.3 — Cart Service (Go)

```
cart-service/
├── cmd/main.go
├── internal/
│   ├── domain/cart.go             ← Cart com items
│   ├── service/
│   │   ├── cart_service.go        ← Add/remove/update/clear
│   │   └── cart_service_test.go
│   ├── handler/cart_handler.go
│   └── repository/
│       └── redis_repo.go          ← Cart stored in Redis Hash
├── go.mod
└── Dockerfile
```

### 3.4 — Order Service (Java Spring Boot — Saga Orchestrator)

```
order-service/
├── src/main/java/.../
│   ├── OrderServiceApplication.java
│   ├── domain/
│   │   ├── Order.java
│   │   ├── OrderItem.java
│   │   ├── OrderStatus.java       ← Enum lifecycle
│   │   └── SagaState.java         ← Saga execution state
│   ├── saga/
│   │   ├── CheckoutSaga.java      ← Saga orchestrator
│   │   ├── SagaStep.java          ← Step interface
│   │   ├── ReserveInventoryStep.java
│   │   ├── ProcessPaymentStep.java
│   │   └── CreateOrderStep.java
│   ├── service/
│   │   └── OrderService.java
│   ├── outbox/
│   │   ├── OutboxEvent.java       ← Outbox entity
│   │   └── OutboxRelay.java       ← @Scheduled polling relay
│   ├── repository/
│   │   ├── OrderRepository.java
│   │   └── OutboxRepository.java
│   ├── controller/
│   │   └── OrderController.java
│   └── event/
│       └── OrderEventConsumer.java ← Kafka consumer
├── pom.xml
└── Dockerfile
```

### 3.5 — Payment Service (Go)

```
payment-service/
├── cmd/main.go
├── internal/
│   ├── domain/
│   │   ├── payment.go             ← Payment entity
│   │   └── transaction.go         ← Transaction log
│   ├── service/
│   │   ├── payment_service.go     ← Process, refund
│   │   ├── idempotency.go         ← Idempotency key check
│   │   └── payment_service_test.go
│   ├── gateway/
│   │   ├── gateway.go             ← Interface
│   │   ├── stripe_mock.go         ← Mock payment gateway
│   │   └── gateway_test.go
│   ├── handler/payment_handler.go
│   └── repository/
│       ├── payment_repo.go        ← PostgreSQL
│       └── idempotency_repo.go    ← Redis
├── go.mod
└── Dockerfile
```

### 3.6 — Notification Service (Java Spring Boot)

```
notification-service/
├── src/main/java/.../
│   ├── NotificationServiceApplication.java
│   ├── domain/
│   │   ├── Notification.java
│   │   └── NotificationType.java  ← EMAIL, SMS, PUSH
│   ├── service/
│   │   ├── NotificationService.java
│   │   └── TemplateService.java   ← Template rendering
│   ├── channel/
│   │   ├── NotificationChannel.java ← Interface
│   │   ├── EmailChannel.java      ← JavaMail (mock SMTP)
│   │   ├── SmsChannel.java        ← Mock SMS
│   │   └── PushChannel.java       ← Mock push notification
│   ├── consumer/
│   │   └── NotificationConsumer.java  ← Kafka consumer
│   └── repository/
│       └── NotificationRepository.java
├── pom.xml
└── Dockerfile
```

---

### 3.7 — Infrastructure

**Arquivo:** `docker-compose.yml`

```yaml
services:
  # --- Infrastructure ---
  postgres:
    image: postgres:16
    # databases: products, orders, payments, notifications

  redis:
    image: redis:7-alpine
    # usage: cart, cache, rate limiting, idempotency

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    # topics: product-events, order-events, payment-events, notification-events

  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0

  elasticsearch:
    image: elasticsearch:8.13.0
    # usage: product search

  minio:
    image: minio/minio
    # usage: product images

  # --- Services ---
  gateway:
    build: ./gateway
    ports: ["8080:8080"]

  product-catalog:
    build: ./product-catalog

  cart-service:
    build: ./cart-service

  order-service:
    build: ./order-service

  payment-service:
    build: ./payment-service

  notification-service:
    build: ./notification-service
```

---

## Parte 4 — Cross-cutting Concerns

### 4.1 Observability

- [ ] **Structured logging** em todos os serviços (JSON format)
- [ ] **Distributed tracing** — trace ID propagado via headers (Go: OpenTelemetry, Java: Micrometer Tracing)
- [ ] **Metrics** — RED metrics (Rate, Errors, Duration) por serviço
- [ ] **Health checks** — `/health` endpoint em cada serviço

### 4.2 Resilience

- [ ] **Circuit breaker** no API Gateway (por upstream service)
- [ ] **Rate limiting** — per-user e per-IP no gateway
- [ ] **Retry with backoff** — communication entre serviços
- [ ] **Timeout** — deadline propagation
- [ ] **Bulkhead** — thread/goroutine isolation

### 4.3 Security

- [ ] **JWT authentication** no gateway
- [ ] **Service-to-service auth** — mutual TLS ou API keys
- [ ] **Input validation** em todos os endpoints
- [ ] **SQL injection protection** — parameterized queries

### 4.4 Testing

- [ ] **Unit tests** — domínio + serviços (≥ 80% coverage cada serviço)
- [ ] **Integration tests** — serviço + DB/Redis/Kafka (Testcontainers)
- [ ] **Contract tests** — API contracts entre serviços
- [ ] **End-to-end test** — checkout flow completo (happy path + failure)

---

## Critérios de Aceite Globais

- [ ] 6 serviços: 3 em Go + 3 em Java
- [ ] gRPC ou REST para sync communication entre serviços
- [ ] Kafka para async events
- [ ] Checkout saga funcional (happy path + compensação)
- [ ] Outbox pattern para reliable event publishing (Order Service)
- [ ] Product search via Elasticsearch
- [ ] Cart persistence em Redis
- [ ] URL curta/profunda para product images (MinIO + CDN URL)
- [ ] Rate limiting + circuit breaker no gateway
- [ ] Distributed tracing end-to-end
- [ ] Docker Compose com todos os serviços + infra

---

## Definição de Pronto (DoD)

- [ ] 6 ADRs (architecture, communication, data, saga, observability, resilience)
- [ ] 4 DrawIO (C4 context, C4 containers, checkout saga, data flow)
- [ ] 6 serviços implementados (3 Go + 3 Java)
- [ ] Docker Compose: `docker compose up` → tudo funciona
- [ ] Checkout E2E: browse → cart → checkout → payment → order → notification
- [ ] Tests: unit + integration + E2E
- [ ] Commit: `feat(system-design-23): capstone megastore platform`

---

## Checklist Final

```
CAPSTONE SYSTEM DESIGN — MEGASTORE
═══════════════════════════════════

ARCHITECTURE DECISIONS
  □ ADR-001  Architecture Style
  □ ADR-002  Inter-service Communication
  □ ADR-003  Data Architecture
  □ ADR-004  Checkout Saga
  □ ADR-005  Observability Strategy
  □ ADR-006  Resilience & Traffic

DIAGRAMS
  □ C4 System Context
  □ C4 Container Diagram
  □ Checkout Saga Sequence
  □ Data Flow & Events

GO SERVICES
  □ API Gateway (proxy + auth + rate limit + circuit breaker)
  □ Cart Service (Redis-based, CRUD)
  □ Payment Service (idempotent, mock gateway)

JAVA SERVICES
  □ Product Catalog (JPA + Elasticsearch)
  □ Order Service (Saga Orchestrator + Outbox)
  □ Notification Service (multi-channel, Kafka consumer)

INFRASTRUCTURE
  □ Docker Compose (all services + infra)
  □ PostgreSQL (products, orders, payments, notifications)
  □ Redis (cart, cache, rate limiting)
  □ Kafka (event bus)
  □ Elasticsearch (product search)
  □ MinIO (product images)

CROSS-CUTTING
  □ JWT auth (gateway)
  □ Distributed tracing (all services)
  □ Structured logging (all services)
  □ Health checks (all services)
  □ Circuit breaker (gateway)
  □ Rate limiting (gateway)

TESTING
  □ Unit tests (≥ 80% per service)
  □ Integration tests (Testcontainers)
  □ Checkout E2E test
  □ Contract tests

═══════════════════════════════════
```
