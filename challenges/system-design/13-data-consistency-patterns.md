# Level 13 — Data Consistency: Saga, Outbox & Idempotency

> **Objetivo:** Implementar o Saga Pattern (choreography + orchestration), Transactional
> Outbox para reliable event publishing e Idempotency para operações seguras contra retries.

**Referência:**
- [29-saga-pattern.md](../../.docs/SYSTEM-DESIGN/29-saga-pattern.md)
- [30-outbox-pattern.md](../../.docs/SYSTEM-DESIGN/30-outbox-pattern.md)
- [34-idempotency.md](../../.docs/SYSTEM-DESIGN/34-idempotency.md)

**Pré-requisito:** Level 12 completo.

---

## Parte 1 — ADRs

### ADR-001: Saga Pattern Design

**Arquivo:** `docs/adrs/ADR-001-saga-pattern.md`

**Options — Saga Coordination:**
1. **Choreography** — serviços emitem/consomem eventos autonomamente
2. **Orchestration** — saga orchestrator coordena steps
3. **Hybrid** — choreography intra-domain, orchestration inter-domain

**Compensation strategy:**
- Semantic compensation (undo via business logic)
- Compensating transactions
- Pivot transactions (point of no return)

### ADR-002: Outbox + Idempotency Design

**Arquivo:** `docs/adrs/ADR-002-outbox-idempotency.md`

**Options — Outbox:**
1. **Polling Publisher** — poll outbox table periodically
2. **Transaction Log Tailing (CDC)** — Debezium reads WAL
3. **Dual Write with reconciliation** — write to DB + broker, reconcile

**Options — Idempotency:**
1. **Idempotency Key** — client sends unique key, server dedup
2. **Database constraints** — unique index prevents duplicates
3. **At-most-once delivery** — broker handles dedup

**Critérios de aceite:**
- [ ] Choreography vs Orchestration decision justificada
- [ ] Compensation steps documentados para 3+ scenarios
- [ ] Outbox polling vs CDC trade-offs
- [ ] Idempotency key strategy documentada

---

## Parte 2 — Diagrama DrawIO

**Arquivo:** `docs/diagrams/13-data-consistency.drawio`

**View 1 — Saga Orchestration (Order Flow):**
```
┌──────────────┐
│ Saga         │
│ Orchestrator │
└──┬───┬───┬──┘
   │   │   │
   │   │   │  1. Create Order
   │   │   └───────────▶ Order Service ───▶ OrderCreated
   │   │
   │   │  2. Reserve Stock
   │   └──────────────▶ Inventory Service ──▶ StockReserved
   │
   │  3. Process Payment
   └─────────────────▶ Payment Service ───▶ PaymentProcessed
   
   IF any step fails:
   ◀── Compensate in reverse order
   Payment: Refund → Inventory: Release → Order: Cancel
```

**View 2 — Outbox Pattern:**
```
┌─────────────────────────────────┐
│ Service                         │
│                                 │
│  BEGIN TRANSACTION              │
│  ┌──────────┐  ┌─────────────┐ │
│  │ Business │  │   Outbox    │ │
│  │  Table   │  │   Table     │ │
│  │ (update) │  │ (insert evt)│ │
│  └──────────┘  └─────────────┘ │
│  COMMIT                         │
└─────────────────────────────────┘
         │
         │  Outbox Relay (polls or CDC)
         ▼
┌──────────────┐
│ Message      │
│ Broker       │
│ (Kafka)      │
└──────────────┘
```

**View 3 — Idempotency Flow:** Request with idempotency key → check → execute or return cached

**Critérios de aceite:**
- [ ] Saga com happy path e compensation path
- [ ] Outbox pattern com transaction boundary claro
- [ ] Idempotency key flow

---

## Parte 3 — Implementação

### Domínio: E-Commerce Order Processing

Fluxo: **CreateOrder → ReserveInventory → ProcessPayment → ConfirmOrder**
Compensação: **RefundPayment → ReleaseInventory → CancelOrder**

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/
│   ├── orchestrator/main.go      ← Saga orchestrator
│   ├── order-svc/main.go         ← Order service
│   ├── inventory-svc/main.go     ← Inventory service
│   ├── payment-svc/main.go       ← Payment service
│   └── outbox-relay/main.go      ← Outbox publisher
├── internal/
│   ├── saga/
│   │   ├── orchestrator.go       ← Saga orchestrator engine
│   │   ├── step.go               ← Saga step definition
│   │   ├── state.go              ← Saga state machine
│   │   ├── compensation.go       ← Compensation executor
│   │   └── saga_test.go
│   ├── outbox/
│   │   ├── outbox.go             ← Outbox table writer
│   │   ├── relay.go              ← Outbox relay (polling)
│   │   ├── cdc.go                ← CDC-based relay (Debezium)
│   │   └── outbox_test.go
│   ├── idempotency/
│   │   ├── middleware.go         ← HTTP middleware
│   │   ├── store.go              ← Idempotency key store
│   │   └── idempotency_test.go
│   ├── order/
│   │   ├── service.go
│   │   └── repository.go
│   ├── inventory/
│   │   ├── service.go
│   │   └── repository.go
│   └── payment/
│       ├── service.go
│       └── repository.go
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **Saga Orchestrator** com state machine (pending, executing, compensating, completed, failed)
2. **Saga Steps** definidas declarativamente (action + compensation)
3. **Outbox** transactional write (same DB transaction)
4. **Outbox Relay** polling com batch processing
5. **Idempotency Middleware** com key + TTL + response caching
6. **3 services** (Order, Inventory, Payment) comunicando via broker
7. **Compensation** executa em ordem reversa
8. **Timeout handling** para steps que não respondem
9. **Persistent saga state** para recovery após crash
10. **Metrics** (saga duration, success/failure rate, compensation count)

**Critérios de aceite Go:**
- [ ] Saga happy path: Order → Inventory → Payment → Confirm
- [ ] Saga compensation: Payment fails → Release Stock → Cancel Order
- [ ] Saga timeout: step não responde → compensate
- [ ] Outbox: evento publicado atomicamente com business write
- [ ] Outbox relay: processa pendentes, marca como published
- [ ] Idempotency: retry com mesma key retorna cached response
- [ ] Idempotency: key diferente executa novamente
- [ ] Persistent saga state sobrevive restart
- [ ] ≥ 20 testes (unit + integration)
- [ ] Chaos test: kill service during saga

---

### 3.2 — Java

**Funcionalidades Java:**
1. **Saga** com Spring State Machine ou implementação custom
2. **Outbox** com Spring Data JPA (same transaction)
3. **Outbox Relay** com `@Scheduled` + polling
4. **Debezium** CDC integration (optional extra credit)
5. **Idempotency** com Spring AOP `@Idempotent` annotation
6. **Records** para saga steps e events
7. **Sealed interface** para saga states
8. **Virtual Threads** para I/O operations

**Critérios de aceite Java:**
- [ ] Saga orchestrator funcional
- [ ] Outbox transactional
- [ ] Idempotency via annotation
- [ ] Testes com Testcontainers (Postgres + Kafka)
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] 2 ADRs (Saga, Outbox + Idempotency)
- [ ] DrawIO com 3 views
- [ ] Go e Java: Saga + Outbox + Idempotency + 3 services + tests
- [ ] Chaos test: service failure during saga → compensation executa
- [ ] Commit: `feat(system-design-13): saga outbox idempotency`
