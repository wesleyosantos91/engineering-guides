# Level 7 — Projeto Capstone: Payment Processor Production-Ready

> **Objetivo:** Consolidar TODOS os padrões e práticas dos Levels 0-6 em um Payment Processor
> completo, production-ready, implementado em Java 25 puro e Go 1.26 puro.
> Este é o projeto de portfólio final da trilha de Design Patterns.

---

## Pré-requisito

[Level 6 — Boas Práticas & Integração](06-best-practices-integration.md) completo.

---

## Visão Geral

O Capstone é a implementação final do Payment Processor que você evoluiu ao longo dos 7 levels.
Não é código novo do zero — é a consolidação e polimento de tudo que foi construído, com foco em:

1. **Completude** — todos os padrões aplicados onde fazem sentido
2. **Coerência** — padrões integrados harmoniosamente, sem conflitos
3. **Qualidade** — código limpo, testado, documentado
4. **Pragmatismo** — padrões aplicados por necessidade, não por dogma

---

## Arquitetura Final

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Framework & Drivers                          │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                     Interface Adapters                         │  │
│  │  ┌──────────────┐  ┌───────────────┐  ┌─────────────────────┐│  │
│  │  │ HTTP (MVC)   │  │ CLI Adapter   │  │ Event Bus Adapter   ││  │
│  │  │ Controller   │  │               │  │                     ││  │
│  │  │ View (JSON)  │  │               │  │                     ││  │
│  │  └──────┬───────┘  └──────┬────────┘  └────────┬────────────┘│  │
│  │         │                 │                     │             │  │
│  │  ┌──────▼─────────────────▼─────────────────────▼───────────┐│  │
│  │  │              Application (Use Cases / CQRS)              ││  │
│  │  │  ┌─────────────────────┐  ┌────────────────────────────┐ ││  │
│  │  │  │   Command Side      │  │    Query Side              │ ││  │
│  │  │  │  ChargeHandler      │  │  GetTransactionHandler     │ ││  │
│  │  │  │  RefundHandler      │  │  ListTransactionsHandler   │ ││  │
│  │  │  │  VoidHandler        │  │  ReportHandler             │ ││  │
│  │  │  └─────────┬───────────┘  └─────────────┬──────────────┘ ││  │
│  │  │            │                             │                ││  │
│  │  │  ┌─────────▼─────────────────────────────▼──────────────┐ ││  │
│  │  │  │                   Domain Core                        │ ││  │
│  │  │  │  Entities: Transaction, Money, PaymentMethod         │ ││  │
│  │  │  │  Value Objects: TransactionId, CustomerId            │ ││  │
│  │  │  │  States: Created → Authorized → Captured → Settled  │ ││  │
│  │  │  │  Events: TransactionCreated, Authorized, Captured... │ ││  │
│  │  │  │  Ports: Gateway, Repository, Notifier, EventStore    │ ││  │
│  │  │  └──────────────────────────────────────────────────────┘ ││  │
│  │  └──────────────────────────────────────────────────────────┘│  │
│  │  ┌──────────────────────────────────────────────────────────┐│  │
│  │  │              Outbound Adapters                           ││  │
│  │  │  Gateway: Stripe, PagSeguro, PayPal (Adapter pattern)   ││  │
│  │  │  Repository: InMemory (Event Sourcing + Projections)    ││  │
│  │  │  Notification: Email, Webhook (Observer + Decorator)    ││  │
│  │  │  EventStore: InMemory append-only                       ││  │
│  │  └──────────────────────────────────────────────────────────┘│  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Desafios

### Desafio 7.1 — Project Setup & Skeleton

**Crie a estrutura final do projeto em ambas as linguagens.**

**Java 25:**
```
payment-processor-java/
├── build.gradle (ou pom.xml)
├── src/
│   ├── main/java/com/payment/
│   │   ├── entity/
│   │   │   ├── Money.java                    // record, Value Object
│   │   │   ├── Transaction.java              // Aggregate Root
│   │   │   ├── TransactionId.java            // Value Object
│   │   │   ├── CustomerId.java               // Value Object
│   │   │   ├── PaymentMethod.java            // sealed interface
│   │   │   ├── TransactionStatus.java        // enum (State pattern)
│   │   │   └── Currency.java                 // enum
│   │   ├── state/
│   │   │   ├── TransactionState.java         // sealed interface
│   │   │   ├── CreatedState.java
│   │   │   ├── AuthorizedState.java
│   │   │   ├── CapturedState.java
│   │   │   ├── SettledState.java
│   │   │   ├── CancelledState.java
│   │   │   └── RefundedState.java
│   │   ├── event/
│   │   │   ├── DomainEvent.java              // sealed interface
│   │   │   ├── TransactionCreated.java
│   │   │   ├── TransactionAuthorized.java
│   │   │   └── ... (8 event types)
│   │   ├── port/
│   │   │   ├── inbound/
│   │   │   │   ├── ChargePaymentPort.java
│   │   │   │   ├── RefundPaymentPort.java
│   │   │   │   └── QueryTransactionPort.java
│   │   │   └── outbound/
│   │   │       ├── PaymentGatewayPort.java
│   │   │       ├── TransactionRepositoryPort.java
│   │   │       ├── EventStorePort.java
│   │   │       ├── NotificationPort.java
│   │   │       └── EventPublisherPort.java
│   │   ├── usecase/
│   │   │   ├── command/
│   │   │   │   ├── ChargePaymentUseCase.java
│   │   │   │   ├── RefundPaymentUseCase.java
│   │   │   │   └── VoidPaymentUseCase.java
│   │   │   └── query/
│   │   │       ├── GetTransactionUseCase.java
│   │   │       └── ListTransactionsUseCase.java
│   │   ├── adapter/
│   │   │   ├── inbound/
│   │   │   │   ├── http/
│   │   │   │   │   ├── PaymentController.java
│   │   │   │   │   ├── JsonView.java
│   │   │   │   │   └── HttpServer.java
│   │   │   │   └── cli/
│   │   │   │       └── CliAdapter.java
│   │   │   └── outbound/
│   │   │       ├── gateway/
│   │   │       │   ├── StripeGatewayAdapter.java
│   │   │       │   └── PagSeguroGatewayAdapter.java
│   │   │       ├── persistence/
│   │   │       │   ├── InMemoryEventStore.java
│   │   │       │   └── InMemoryProjectionRepository.java
│   │   │       └── notification/
│   │   │           ├── ConsoleNotificationAdapter.java
│   │   │           └── WebhookNotificationAdapter.java
│   │   ├── pattern/
│   │   │   ├── strategy/                     // FeeStrategy, FraudStrategy
│   │   │   ├── observer/                     // TransactionObserver, EventBus
│   │   │   ├── command/                      // PaymentCommand, History
│   │   │   ├── chain/                        // ValidationHandler pipeline
│   │   │   ├── decorator/                    // Gateway decorators
│   │   │   ├── factory/                      // PaymentMethodFactory
│   │   │   ├── builder/                      // TransactionBuilder
│   │   │   └── visitor/                      // Report visitors
│   │   └── Main.java                         // Bootstrap & composition root
│   └── test/java/com/payment/
│       ├── entity/
│       ├── usecase/
│       ├── adapter/
│       ├── pattern/
│       └── integration/
├── ARCHITECTURE.md
├── DECISIONS.md
├── PATTERNS.md
└── README.md
```

**Go 1.26:**
```
payment-processor-go/
├── go.mod
├── cmd/
│   ├── server/main.go                        // HTTP server bootstrap
│   └── cli/main.go                           // CLI bootstrap
├── internal/
│   ├── entity/
│   │   ├── money.go
│   │   ├── transaction.go
│   │   ├── payment_method.go
│   │   ├── status.go
│   │   └── currency.go
│   ├── state/
│   │   ├── state.go                          // TransactionState interface
│   │   ├── created.go
│   │   ├── authorized.go
│   │   ├── captured.go
│   │   └── ...
│   ├── event/
│   │   ├── event.go                          // DomainEvent interface
│   │   ├── transaction_created.go
│   │   └── ...
│   ├── port/
│   │   ├── inbound.go                        // driving ports
│   │   └── outbound.go                       // driven ports
│   ├── usecase/
│   │   ├── charge.go
│   │   ├── refund.go
│   │   └── query.go
│   ├── adapter/
│   │   ├── http/
│   │   │   ├── controller.go
│   │   │   ├── view.go
│   │   │   └── server.go
│   │   ├── cli/
│   │   │   └── adapter.go
│   │   ├── gateway/
│   │   │   ├── stripe.go
│   │   │   └── pagseguro.go
│   │   ├── persistence/
│   │   │   ├── event_store.go
│   │   │   └── projection.go
│   │   └── notification/
│   │       ├── console.go
│   │       └── webhook.go
│   └── pattern/
│       ├── strategy/
│       ├── observer/
│       ├── command/
│       ├── chain/
│       ├── decorator/
│       ├── factory/
│       └── visitor/
├── ARCHITECTURE.md
├── DECISIONS.md
├── PATTERNS.md
└── README.md
```

**Critérios de aceite:**
- [ ] Estrutura de diretórios criada em ambas as linguagens
- [ ] `build.gradle`/`pom.xml` (Java) e `go.mod` (Go) configurados
- [ ] Main compila e roda (even if does nothing yet)
- [ ] Todos os packages/pacotes importam corretamente

---

### Desafio 7.2 — Domain Core Implementation

**Implemente o core do domínio usando todos os padrões dos Levels 0-4.**

**Entidades e Value Objects (Level 0):**
- `Money` (record/struct) — operações aritméticas, comparação, imutável
- `Transaction` — aggregate root com estado, eventos, comportamento
- `PaymentMethod` — sealed interface/interface com 4+ implementações
- `TransactionId`, `CustomerId` — Value Objects tipados

**SOLID (Level 1):**
- SRP: cada classe/struct com responsabilidade única
- OCP: extensível via Strategy, Factory, Observer
- LSP: subtipos substituíveis sem quebrar contrato
- ISP: interfaces pequenas e focadas
- DIP: dependências injetadas como interfaces

**Padrões Creacionais (Level 2):**
- `PaymentMethodFactory` (Factory Method)
- `TransactionBuilder` (Builder / Functional Options)
- `PaymentConfig` (Singleton — config global)

**Padrões Estruturais (Level 3):**
- Gateway Adapters (Adapter)
- Gateway Decorators: Logging, Metrics, Retry, Idempotency (Decorator)
- `SplitPayment` (Composite)
- `PaymentFacade` (Facade)

**Padrões Comportamentais (Level 4):**
- `TransactionState` (State) — máquina de estados
- `FeeStrategy` + `FraudDetectionStrategy` (Strategy)
- `PaymentCommand` + History (Command)
- `ValidationPipeline` (Chain of Responsibility)
- `TransactionObserver` + Event Bus (Observer)
- `TransactionMemento` (Memento)
- Report Visitors (Visitor)
- Transaction Iterators (Iterator)

**Critérios de aceite:**
- [ ] Todas as entidades e VOs implementados e imutáveis
- [ ] State machine funcional com todas as transições
- [ ] Event sourcing: Transaction emite eventos para cada mudança
- [ ] ≥ 15 padrões GoF aplicados organicamente
- [ ] Testes unitários cobrindo domain core ≥ 90%

---

### Desafio 7.3 — Application Layer (CQRS)

**Implemente a camada de aplicação com CQRS.**

**Command Side:**
- `ChargePaymentUseCase` — valida, detecta fraude, cria transação, processa via gateway
- `RefundPaymentUseCase` — valida estado, processa estorno
- `VoidPaymentUseCase` — cancela transação autorizada
- Cada use case emite domain events ao Event Store

**Query Side:**
- `GetTransactionUseCase` — busca por ID (reconstrução ou projeção)
- `ListTransactionsUseCase` — lista com filtros e paginação
- `TransactionReportUseCase` — relatórios via Visitor pattern
- Projeções atualizadas assincronamente via eventos

**Command/Query Bus:**
```java
// Java
CommandBus bus = new InMemoryCommandBus();
bus.register(ChargeCommand.class, new ChargePaymentUseCase(gateway, repo, eventStore));
bus.dispatch(new ChargeCommand(amount, method, customerId));

// Go
bus := command.NewBus()
bus.Register(reflect.TypeOf(ChargeCommand{}), chargeHandler)
bus.Dispatch(ctx, ChargeCommand{Amount: amount, Method: method})
```

**Critérios de aceite:**
- [ ] ≥ 3 command handlers + ≥ 3 query handlers
- [ ] CommandBus despacha por tipo de command
- [ ] QueryBus retorna dados sem efeito colateral
- [ ] Events sincronizam write → read model
- [ ] Testes: cada handler isolado com ports mock

---

### Desafio 7.4 — Adapters & Infrastructure

**Implemente todos os adapters necessários.**

**Inbound Adapters:**
- HTTP Server (MVC) com 5 endpoints:
  - `POST /payments` — charge
  - `POST /payments/{id}/refund` — refund
  - `POST /payments/{id}/void` — void
  - `GET /payments/{id}` — get by id
  - `GET /payments` — list (com query params: status, dateRange, page)
- CLI Adapter com comandos interativos

**Outbound Adapters:**
- `StripeGatewayAdapter` — simula chamada ao Stripe (in-memory)
- `PagSeguroGatewayAdapter` — simula chamada ao PagSeguro (in-memory)
- `InMemoryEventStore` — append-only, getEvents(aggregateId)
- `InMemoryProjectionRepository` — read model com filtros
- `ConsoleNotificationAdapter` — imprime notificações no console
- `WebhookNotificationAdapter` — simula POST HTTP (log apenas)

**Gateway com Decorators:**
```
Gateway real
  └── IdempotencyDecorator (evita duplicação)
    └── RetryDecorator (retry com backoff)
      └── MetricsDecorator (conta chamadas, latência)
        └── LoggingDecorator (log entrada/saída)
```

**Critérios de aceite:**
- [ ] HTTP server funcional com 5 endpoints (sem framework)
- [ ] Requisição → Controller → UseCase → Domain → Adapter → Resposta
- [ ] Gateway decorators stackable e configuráveis
- [ ] CLI funcional com pelo menos charge, refund, list
- [ ] Event Store append-only com reconstruct
- [ ] Testes de integração: HTTP request end-to-end

---

### Desafio 7.5 — Documentation & Architecture Decision Records

**Documente a arquitetura e as decisões de design.**

**`ARCHITECTURE.md`:**
- Diagrama da arquitetura (ASCII art ou Mermaid se suportado)
- Descrição de cada camada e responsabilidade
- Fluxo de dados para charge (request → response)
- Fluxo de dados para query (request → response)
- Mapa de dependências entre módulos

**`DECISIONS.md`** — Architecture Decision Records (ADR):

| # | Decisão | Contexto | Alternativas | Escolha | Consequências |
|---|---------|----------|--------------|---------|---------------|
| ADR-001 | Event Sourcing para persistência | Audit trail completo | Snapshot apenas | Event Sourcing + Snapshot | Complexidade de reconstruct |
| ADR-002 | CQRS | Diferentes necessidades read/write | CRUD único | CQRS | Eventual consistency |
| ADR-003 | State Pattern para ciclo de vida | Eliminar switch/if sobre status | Enum + switch | State Pattern | Mais classes |
| ADR-004 | Strategy para fees | Extensibilidade de cálculo | If/else | Strategy + Registry | Indireção |
| ADR-005 | Decorator para cross-cutting | Logging, retry, metrics | AOP, interceptors | Decorator chain | Composability |
| ... | ... | ... | ... | ... | ... |

**`PATTERNS.md`:**
- Tabela com todos os padrões usados, onde, e por quê
- Para cada padrão: problema que resolve, alternativa considerada, trade-off

**Critérios de aceite:**
- [ ] `ARCHITECTURE.md` com diagrama e descrição de camadas
- [ ] `DECISIONS.md` com ≥ 10 ADRs completos
- [ ] `PATTERNS.md` com mapa de ≥ 15 padrões
- [ ] `README.md` do projeto com setup, execução, exemplos
- [ ] Cada padrão justificado (não aplicado por dogma)

---

### Desafio 7.6 — Testing Strategy & Coverage

**Implemente uma estratégia de testes em pirâmide.**

**Pirâmide de testes:**

```
         ╱╲
        ╱ E2E ╲         ← Poucos: HTTP request → response
       ╱────────╲
      ╱Integration╲     ← Médio: UseCase + Adapters reais
     ╱──────────────╲
    ╱   Unit Tests    ╲  ← Muitos: Entities, VOs, States, Strategies
   ╱────────────────────╲
```

**Testes unitários (≥ 70% do total):**
- Entidades e Value Objects: Money, Transaction, PaymentMethod
- State machine: todas as transições válidas e inválidas
- Strategies: cada strategy com múltiplos cenários
- Commands: execute + undo
- Validators: cada handler da chain
- Visitors: cada visitor com dados variados
- Factories: criação com parâmetros válidos e inválidos

**Testes de integração (≥ 20% do total):**
- UseCase com adapters reais (in-memory)
- Event Store: append + reconstruct
- Command Bus + Query Bus dispatch

**Testes end-to-end (≥ 10% do total):**
- HTTP: charge → get → refund → get (fluxo completo)
- CLI: charge → list (se implementado)

**Critérios de aceite:**
- [ ] Cobertura total ≥ 85%
- [ ] Cobertura domain core ≥ 90%
- [ ] Testes unitários, integração e E2E separados
- [ ] Testes nomeados com padrão: `should_[behavior]_when_[condition]`
- [ ] Java: JUnit 5 + AssertJ + Mockito
- [ ] Go: `testing` + `testify` + `httptest`
- [ ] Zero teste flaky (determinísticos)

---

### Desafio 7.7 — Performance & Concurrency

**Teste e otimize cenários de concorrência usando recursos puros da linguagem.**

**Cenários de concorrência:**

| Cenário | Problema | Solução |
|---------|----------|---------|
| Charge simultâneo | Mesma transação processada 2x | Idempotency key + lock |
| Refund race condition | Refund antes do capture completar | State machine + sync |
| Event Store append | Múltiplas goroutines/threads escrevendo | `sync.RWMutex` / `ConcurrentHashMap` |
| Read model update | Projeção desatualizada durante write | Eventual consistency documentada |

**Java 25 — Virtual Threads:**
```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    // cada request processada em sua own virtual thread
    server.setExecutor(executor);
}
```

**Go 1.26 — Goroutines & Channels:**
```go
// Event Bus com goroutines
func (bus *EventBus) Publish(event DomainEvent) {
    for _, handler := range bus.handlers[event.Type()] {
        go handler.Handle(event)
    }
}
```

**Benchmarks a implementar:**
- Throughput: N charges por segundo (Java: JMH-style; Go: `testing.B`)
- Latência: P50, P95, P99 de processamento
- Concorrência: N goroutines/virtual threads simultâneas

**Critérios de aceite:**
- [ ] Zero race condition: testes com `-race` (Go), concurrent test scenarios (Java)
- [ ] Idempotency key previne processamento duplicado
- [ ] Java: virtual threads para I/O concorrente
- [ ] Go: goroutines + channels para event bus assíncrono
- [ ] Benchmark: charge throughput medido e documentado
- [ ] `PERFORMANCE.md` com resultados e otimizações aplicadas

---

### Desafio 7.8 — Final Delivery & Portfolio

**Prepare o projeto para apresentação como item de portfólio.**

**README.md do projeto completo:**

```markdown
# Payment Processor — Design Patterns Showcase

## Overview
Production-ready payment processor built with pure Java 25 / Go 1.26
demonstrating 20+ GoF design patterns and architectural patterns.

## Patterns Implemented
[tabela com todos os padrões]

## Architecture
[diagrama da arquitetura]

## Quick Start
[instruções de build e execução]

## API Documentation
[endpoints com exemplos]

## Design Decisions
[link para DECISIONS.md]

## Test Coverage
[métricas e como rodar]
```

**Checklist final:**

| # | Item | Status |
|---|------|--------|
| 1 | Projeto compila e roda em ambas as linguagens | |
| 2 | 5 endpoints HTTP funcionais | |
| 3 | CLI funcional | |
| 4 | ≥ 15 GoF patterns aplicados organicamente | |
| 5 | CQRS + Event Sourcing funcionais | |
| 6 | Cobertura ≥ 85% | |
| 7 | Zero anti-patterns | |
| 8 | ARCHITECTURE.md completo | |
| 9 | DECISIONS.md com ≥ 10 ADRs | |
| 10 | PATTERNS.md com mapa completo | |
| 11 | README.md de portfólio | |
| 12 | Concorrência testada e segura | |
| 13 | PERFORMANCE.md com benchmarks | |
| 14 | QUALITY-REPORT.md passando | |

**Critérios de aceite:**
- [ ] 14/14 itens do checklist completos
- [ ] Projeto funciona standalone (clone → build → run)
- [ ] Código limpo, idiomático em ambas as linguagens
- [ ] Documentação completa e auto-explicativa
- [ ] Projeto pronto para apresentação em entrevista técnica

---

## Mapa de Padrões no Projeto Final

| Padrão | Componente | Level de Origem |
|--------|-----------|-----------------|
| Value Object | `Money`, `TransactionId` | 0 |
| Sealed Interface | `PaymentMethod`, `DomainEvent` | 0 |
| Enum (State Machine) | `TransactionStatus` | 0 |
| Repository | `TransactionRepository` | 0 |
| SRP | Todas as classes | 1 |
| OCP | Strategy registry, Factory | 1 |
| LSP | `PaymentMethod` subtypes | 1 |
| ISP | Ports pequenos e focados | 1 |
| DIP | Constructor injection | 1 |
| Singleton | `PaymentConfig` | 2 |
| Factory Method | `PaymentMethodFactory` | 2 |
| Builder / Func Options | `TransactionBuilder` | 2 |
| Adapter | Gateway adapters (Stripe, PagSeguro) | 3 |
| Composite | `SplitPayment` | 3 |
| Decorator | Gateway pipeline (Log, Retry, Metrics) | 3 |
| Facade | `PaymentFacade` | 3 |
| Strategy | `FeeStrategy`, `FraudDetectionStrategy` | 4 |
| Observer | `TransactionObserver`, Event Bus | 4 |
| Command | `PaymentCommand` + History | 4 |
| Template Method | `PaymentProcessingTemplate` | 4 |
| State | `TransactionState` (7 states) | 4 |
| Chain of Responsibility | `ValidationPipeline` | 4 |
| Mediator | `PaymentMediator` | 4 |
| Iterator | Transaction filters | 4 |
| Memento | `TransactionCheckpoint` | 4 |
| Visitor | Report generators | 4 |
| Hexagonal / Ports & Adapters | Toda a estrutura | 5 |
| Clean Architecture | 4 anéis de dependência | 5 |
| MVC | HTTP layer | 5 |
| CQRS | Command/Query separation | 5 |
| Event Sourcing | Event Store + rehydrate | 5 |

---

## Parabéns! 🎉

Ao completar este Capstone, você terá:

- Implementado **20+ design patterns** em contexto real
- Construído um sistema **production-grade** sem nenhum framework
- Dominado **Java 25** e **Go 1.26** idiomáticos
- Produzido documentação de **portfólio profissional**
- Demonstrado capacidade de tomar **decisões arquiteturais** conscientes

**Próximo passo:** Aplique esses padrões nos desafios da trilha Backend com frameworks reais.

→ [Voltar para trilha Design Patterns](../README.md)
→ [Trilha Backend](../../backend/README.md)
