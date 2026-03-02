# Level 12 — Event-Driven Architecture, CQRS & Event Sourcing

> **Objetivo:** Implementar um sistema completo com Event-Driven Architecture, separação
> de command/query models (CQRS) e Event Sourcing como fonte da verdade.

**Referência:**
- [26-event-driven-architecture.md](../../.docs/SYSTEM-DESIGN/26-event-driven-architecture.md)
- [27-cqrs.md](../../.docs/SYSTEM-DESIGN/27-cqrs.md)
- [28-event-sourcing.md](../../.docs/SYSTEM-DESIGN/28-event-sourcing.md)

**Pré-requisito:** Level 11 completo.

---

## Parte 1 — ADRs

### ADR-001: Event-Driven vs Request-Driven

**Arquivo:** `docs/adrs/ADR-001-event-driven-architecture.md`

**Options:**
1. **Synchronous (Request-Driven)** — REST/gRPC calls diretos
2. **Asynchronous (Event-Driven)** — eventos via broker
3. **Hybrid** — sync para queries, async para commands
4. **Choreography** — serviços reagem a eventos autonomamente
5. **Orchestration** — orquestrador central coordena

**Critérios de aceite:**
- [ ] Sync vs Async trade-offs
- [ ] Choreography vs Orchestration comparados
- [ ] Event schema evolution strategy

### ADR-002: CQRS + Event Sourcing Design

**Arquivo:** `docs/adrs/ADR-002-cqrs-event-sourcing.md`

**Options:**
1. **CRUD tradicional** — single model
2. **CQRS sem Event Sourcing** — read/write models separados
3. **Event Sourcing sem CQRS** — events como source of truth, single model
4. **CQRS + Event Sourcing** — full pattern

**Critérios de aceite:**
- [ ] Quando usar CQRS sem ES e vice-versa
- [ ] Event store design (schema, partitioning)
- [ ] Projection rebuilding strategy
- [ ] Snapshot strategy para aggregates com muitos eventos

---

## Parte 2 — Diagrama DrawIO

**Arquivo:** `docs/diagrams/12-event-architecture.drawio`

**View 1 — CQRS + Event Sourcing:**
```
┌─────────┐  Command  ┌──────────┐  Events  ┌──────────┐
│  Client  │─────────▶│ Command  │─────────▶│  Event   │
│          │          │  Handler │          │  Store   │
└─────────┘          └──────────┘          └────┬─────┘
                                                │ Events
     ┌──────────────────────────────────────────┘
     │              │              │
┌────▼─────┐  ┌─────▼────┐  ┌─────▼────┐
│Projection│  │Projection│  │Projection│
│  (List)  │  │ (Detail) │  │(Analytics│
└────┬─────┘  └─────┬────┘  └─────┬────┘
     │              │              │
┌────▼─────┐  ┌─────▼────┐  ┌─────▼────┐
│ Read DB  │  │ Read DB  │  │  Redis   │
│(Postgres)│  │ (Mongo)  │  │ (Cache)  │
└──────────┘  └──────────┘  └──────────┘
     ▲              ▲              ▲
     │    Query     │              │
┌─────────┐  ┌──────────┐
│  Client  │  │  Query   │
│          │──│  Handler │
└─────────┘  └──────────┘
```

**View 2 — Event Flow:** Producer → Event Bus → Consumers (fan-out)

**View 3 — Event Store Schema:** Tabela de eventos com aggregate_id, version, event_type, payload

**View 4 — Snapshot + Replay:** Como reconstruir aggregate state a partir de eventos

**Critérios de aceite:**
- [ ] CQRS com read/write sides claros
- [ ] Event store e projections
- [ ] Snapshot e replay flow

---

## Parte 3 — Implementação

### Domínio: Order Management System

Um sistema de pedidos onde:
- **Commands:** CreateOrder, AddItem, RemoveItem, ConfirmOrder, CancelOrder, ShipOrder
- **Events:** OrderCreated, ItemAdded, ItemRemoved, OrderConfirmed, OrderCancelled, OrderShipped
- **Queries:** GetOrder, ListOrders, GetOrderHistory, GetOrdersByStatus

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/
│   ├── command/main.go          ← Command side API
│   ├── query/main.go            ← Query side API
│   └── projector/main.go       ← Event projector
├── internal/
│   ├── domain/
│   │   ├── order.go             ← Order aggregate
│   │   ├── events.go            ← Domain events
│   │   ├── commands.go          ← Commands
│   │   └── errors.go
│   ├── command/
│   │   ├── handler.go           ← Command handlers
│   │   ├── validator.go         ← Command validation
│   │   └── handler_test.go
│   ├── eventstore/
│   │   ├── store.go             ← Interface EventStore
│   │   ├── postgres.go          ← PostgreSQL event store
│   │   ├── snapshot.go          ← Snapshot support
│   │   └── store_test.go
│   ├── projection/
│   │   ├── projector.go         ← Event projector engine
│   │   ├── order_list.go        ← Order list projection
│   │   ├── order_detail.go      ← Order detail projection
│   │   ├── analytics.go         ← Analytics projection
│   │   └── projector_test.go
│   ├── query/
│   │   ├── handler.go           ← Query handlers
│   │   └── repository.go       ← Read model repository
│   └── eventbus/
│       ├── bus.go               ← In-process event bus
│       ├── kafka.go             ← Kafka event bus
│       └── bus_test.go
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **Order Aggregate** que aplica eventos e mantém estado
2. **Event Store** em PostgreSQL (append-only, versioned)
3. **Command Handlers** que validam e emitem eventos
4. **Event Bus** (in-memory + Kafka)
5. **Projections:** list (PostgreSQL), detail (MongoDB), analytics (Redis)
6. **Snapshots** a cada N eventos para performance
7. **Replay** de eventos para reconstruir projections
8. **Idempotent projectors** (handle duplicate events)
9. **Optimistic concurrency** com version numbers
10. **Event schema versioning** (upcasting)

**Critérios de aceite Go:**
- [ ] Order aggregate reconstruído a partir de eventos
- [ ] Event store: append-only, optimistic concurrency
- [ ] 3 projections distintas de dados diferentes
- [ ] Snapshots reduzem tempo de load (medir)
- [ ] Replay recria projections do zero
- [ ] Idempotent: reprojetar não duplica dados
- [ ] Event bus com Kafka (Testcontainers)
- [ ] ≥ 25 testes
- [ ] Command side e Query side em processos separados

---

### 3.2 — Java

**Funcionalidades Java:**
1. **Order Aggregate** com pattern matching (switch expressions)
2. **Event Store** com Spring Data JPA
3. **CQRS** com Spring modulith (separate modules)
4. **Projections** com Spring Data (JPA + MongoDB + Redis)
5. **Kafka** com Spring Kafka para event bus
6. **Sealed interface** para events e commands
7. **Records** para todos os events e commands
8. **Axon Framework** comparison (optional)

**Critérios de aceite Java:**
- [ ] Aggregate com sealed events
- [ ] Event store funcional
- [ ] 3 projections
- [ ] Kafka event bus
- [ ] Replay e snapshots
- [ ] Testes com Testcontainers
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] 2 ADRs (EDA, CQRS+ES)
- [ ] DrawIO com 4 views
- [ ] Go e Java: CQRS + ES + Event Bus + Projections + tests
- [ ] Replay demonstration
- [ ] Commit: `feat(system-design-12): event driven cqrs event sourcing`
