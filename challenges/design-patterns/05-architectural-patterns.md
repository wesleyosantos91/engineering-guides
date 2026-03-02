# Level 5 — Padrões Arquiteturais

> **Objetivo:** Restructurar o Payment Processor aplicando 7 padrões arquiteturais progressivamente,
> evoluindo de uma arquitetura monolítica para uma arquitetura orientada a eventos.

**Referência:** [05-architectural-patterns.md](../../../.docs/DESIGN-PATTERNS/05-architectural-patterns.md)

---

## Pré-requisito

[Level 4 — Padrões Comportamentais](04-behavioral-patterns.md) completo.

---

## Desafios

### Desafio 5.1 — Layered Architecture: Organização em Camadas

> *"Separar o sistema em camadas com responsabilidades distintas e dependências unidirecionais."*

**Cenário:** O Payment Processor do Level 4 cresceu organicamente. Reorganize-o em camadas claras com regras rígidas de dependência.

**Camadas:**

```
┌─────────────────────────────┐
│     Presentation Layer      │  ← HTTP handlers, CLI, formatação de saída
├─────────────────────────────┤
│     Application Layer       │  ← Casos de uso, orquestração, DTOs
├─────────────────────────────┤
│       Domain Layer          │  ← Entidades, Value Objects, interfaces, regras de negócio
├─────────────────────────────┤
│    Infrastructure Layer     │  ← Repositórios, gateways, adaptadores, I/O
└─────────────────────────────┘
```

**Regras de dependência:**
- Cada camada depende APENAS da camada imediatamente abaixo
- Domain Layer não depende de nenhuma outra camada
- Infrastructure implementa interfaces definidas no Domain

**Organização de pacotes/packages:**

```
src/
├── presentation/
│   ├── http/          // handlers HTTP (net/http, java.net.http)
│   ├── cli/           // interface de linha de comando
│   └── dto/           // objetos de transferência de dados
├── application/
│   ├── usecase/       // ChargeUseCase, RefundUseCase, etc.
│   ├── service/       // application services (orquestração)
│   └── mapper/        // conversão DTO ↔ Domain
├── domain/
│   ├── model/         // Transaction, Money, PaymentMethod
│   ├── repository/    // TransactionRepository (interface)
│   ├── service/       // domain services (FeeCalculator, etc.)
│   └── event/         // domain events
└── infrastructure/
    ├── persistence/   // InMemoryTransactionRepository
    ├── gateway/       // StripeGateway, PagSeguroGateway
    ├── notification/  // EmailSender, WebhookClient
    └── config/        // configuração, bootstrap
```

**Critérios de aceite:**
- [ ] 4 camadas claramente separadas em pacotes/packages
- [ ] Regras de dependência respeitadas (nenhum import de cima para baixo)
- [ ] Domain Layer sem dependências externas (zero import de infra/app/presentation)
- [ ] Use Cases no Application Layer fazem orquestração — não contêm regras de negócio
- [ ] DTOs na Presentation Layer — domain models nunca expostos diretamente
- [ ] Testes unitários para Domain + Application; testes de integração para Infrastructure
- [ ] Documentar limitações do Layered Architecture (rigidez, acoplamento vertical)

---

### Desafio 5.2 — Hexagonal Architecture (Ports & Adapters)

> *"Isolar o core de negócio de detalhes técnicos via Ports (interfaces) e Adapters (implementações)."*

**Cenário:** Evolua a Layered Architecture para Hexagonal. O core (domain + application) define Ports; os Adapters implementam esses Ports para se conectar ao mundo externo. O core é testável sem qualquer infraestrutura.

**Organização:**

```
src/
├── core/                          // Hexágono interno
│   ├── domain/
│   │   ├── model/                 // Transaction, Money, PaymentMethod
│   │   ├── service/               // DomainFeeCalculator
│   │   └── event/                 // TransactionCreated, PaymentProcessed
│   ├── port/
│   │   ├── inbound/               // DRIVING PORTS (use cases)
│   │   │   ├── ChargePaymentPort
│   │   │   ├── RefundPaymentPort
│   │   │   └── GetTransactionPort
│   │   └── outbound/              // DRIVEN PORTS (SPI)
│   │       ├── PaymentGatewayPort
│   │       ├── TransactionRepositoryPort
│   │       ├── NotificationPort
│   │       └── EventPublisherPort
│   └── usecase/
│       ├── ChargePaymentUseCase   // implementa ChargePaymentPort
│       ├── RefundPaymentUseCase
│       └── GetTransactionUseCase
└── adapter/                       // Fora do hexágono
    ├── inbound/                   // DRIVING ADAPTERS
    │   ├── http/                  // HttpPaymentAdapter
    │   └── cli/                   // CliPaymentAdapter
    └── outbound/                  // DRIVEN ADAPTERS
        ├── persistence/           // InMemoryTransactionAdapter
        ├── gateway/               // StripeGatewayAdapter
        ├── notification/          // ConsoleNotificationAdapter
        └── event/                 // InMemoryEventBusAdapter
```

**Driving vs Driven:**

| Tipo | Direção | Port | Adapter (exemplo) |
|------|---------|------|-------------------|
| **Driving** (inbound) | Mundo → Core | `ChargePaymentPort` | `HttpPaymentAdapter` |
| **Driven** (outbound) | Core → Mundo | `PaymentGatewayPort` | `StripeGatewayAdapter` |

**Requisitos:**
- Core define TODAS as interfaces (ports) — zero dependência de adapters
- Driving Adapters chamam Ports inbound (use cases)
- Driven Adapters implementam Ports outbound (SPI)
- Bootstrap/main faz a composição (dependency injection manual)

**Critérios de aceite:**
- [ ] Core (domain + ports + usecases) com zero import de adapter
- [ ] ≥ 3 driving ports + ≥ 4 driven ports
- [ ] ≥ 2 driving adapters (HTTP + CLI) para o mesmo port
- [ ] ≥ 2 driven adapters para `PaymentGatewayPort` (Stripe + PagSeguro)
- [ ] Bootstrap/main compõe todas as dependências via constructor injection
- [ ] Core testável com adapters mock/stub sem nenhuma dependência real
- [ ] Documentar: como trocar adapter sem alterar o core

---

### Desafio 5.3 — Clean Architecture

> *"Regra de Dependência: dependências apontam para o centro (Entities)."*

**Cenário:** Refine a Hexagonal Architecture aplicando os 4 anéis da Clean Architecture de Robert C. Martin. Garanta que a Regra de Dependência é estritamente seguida.

**Anéis (de dentro para fora):**

```
┌───────────────────────────────────────────────┐
│              Frameworks & Drivers              │  ← HTTP server, DB driver
│  ┌───────────────────────────────────────┐    │
│  │           Interface Adapters           │    │  ← Controllers, Presenters, Gateways
│  │  ┌───────────────────────────────┐    │    │
│  │  │       Application Business     │    │    │  ← Use Cases
│  │  │  ┌───────────────────────┐    │    │    │
│  │  │  │  Enterprise Business   │    │    │    │  ← Entities, Value Objects
│  │  │  │      (Domain)          │    │    │    │
│  │  │  └───────────────────────┘    │    │    │
│  │  └───────────────────────────────┘    │    │
│  └───────────────────────────────────────┘    │
└───────────────────────────────────────────────┘
```

**Organização:**

```
src/
├── entity/                  // Anel 1 — Enterprise Business Rules
│   ├── Transaction
│   ├── Money
│   ├── PaymentMethod
│   └── TransactionState
├── usecase/                 // Anel 2 — Application Business Rules
│   ├── ChargePaymentUseCase
│   ├── RefundPaymentUseCase
│   ├── port/                // Input/Output ports (boundaries)
│   │   ├── ChargePaymentInput
│   │   ├── ChargePaymentOutput
│   │   ├── PaymentGatewayOutputPort
│   │   └── TransactionDataAccessPort
│   └── interactor/          // Use Case implementations
│       └── ChargePaymentInteractor
├── adapter/                 // Anel 3 — Interface Adapters
│   ├── controller/          // HTTP controllers
│   ├── presenter/           // formatação de resposta
│   ├── gateway/             // implementation of output ports
│   └── repository/          // implementation of data access ports
└── framework/               // Anel 4 — Frameworks & Drivers
    ├── http/                // HTTP server setup (net/http)
    ├── config/              // configuração
    └── main/                // composição e bootstrap
```

**Clean vs Hexagonal (documentar):**

| Aspecto | Hexagonal (5.2) | Clean (5.3) |
|---------|-----------------|-------------|
| **Camadas** | 2 zonas (core + adapter) | 4 anéis concêntricos |
| **Terminologia** | Ports & Adapters | Entities, Use Cases, Adapters, Frameworks |
| **Data crossing** | N/A | Regra de Dependência + DTOs nas fronteiras |
| **Foco** | Isolamento via interfaces | Regra de Dependência estrita |

**Critérios de aceite:**
- [ ] 4 anéis claramente separados com Regra de Dependência estrita
- [ ] Entities: sem dependências externas, puro domínio
- [ ] Use Cases: dependem apenas de Entities + Output Ports
- [ ] Interface Adapters: implementam Output Ports, convertem dados
- [ ] Data crossing boundaries via DTOs (Input/Output) — nunca Entities
- [ ] `DECISIONS.md`: Clean Architecture vs Hexagonal — diferenças e quando escolher cada
- [ ] Testes: Entities isolados, Use Cases com ports mock, Adapters com integração

---

### Desafio 5.4 — MVC: Interface HTTP Estruturada

> *"Separar Model, View e Controller para interfaces de usuário."*

**Cenário:** Implemente uma interface HTTP REST para o Payment Processor usando o padrão MVC. Sem framework — use apenas `net/http` (Go) e `com.sun.net.httpserver` ou `java.net.http` (Java) para construir o servidor.

**Componentes:**

| Componente | Responsabilidade | Exemplos |
|------------|------------------|----------|
| **Model** | Dados + regras de negócio | `Transaction`, `Money`, `PaymentMethod` |
| **View** | Serialização/formatação de resposta | `JsonTransactionView`, `PlainTextView` |
| **Controller** | Recebe request, delega ao Model, escolhe View | `PaymentController` |

**Endpoints:**

```
POST   /payments           → PaymentController.charge()
POST   /payments/{id}/refund → PaymentController.refund()
GET    /payments/{id}      → PaymentController.getById()
GET    /payments            → PaymentController.list() (com filtros)
DELETE /payments/{id}      → PaymentController.cancel()
```

**Requisitos:**
- Controller NÃO contém lógica de negócio — delega ao Use Case (Model)
- View é responsável por serializar em JSON (ou outro formato)
- Content negotiation: `Accept: application/json` ou `Accept: text/plain`
- Error handling com respostas HTTP padronizadas

**Critérios de aceite:**
- [ ] 5 endpoints REST funcionais com servidor HTTP puro (sem framework)
- [ ] Controller delega ao Use Case — zero lógica de negócio no controller
- [ ] View serializa resposta (JSON/text) — content negotiation funcional
- [ ] Error handling: 400 (validação), 404 (not found), 409 (conflict), 500 (internal)
- [ ] Java: usar `com.sun.net.httpserver.HttpServer` ou `java.net.http`
- [ ] Go: usar `net/http` com `http.ServeMux`
- [ ] Testes: HTTP handler tests com request/response mock

---

### Desafio 5.5 — CQRS: Command Query Responsibility Segregation

> *"Separar modelos de leitura e escrita para otimizar cada lado independentemente."*

**Cenário:** O Payment Processor tem necessidades muito diferentes para escrita (charge, refund — precisa de validações, consistência) e leitura (listar transações, gerar relatórios — precisa de performance, projeções). Separe os modelos.

**Organização:**

```
src/
├── command/                     // WRITE SIDE
│   ├── model/
│   │   └── Transaction          // modelo rico com regras de negócio
│   ├── handler/
│   │   ├── ChargeCommandHandler
│   │   ├── RefundCommandHandler
│   │   └── VoidCommandHandler
│   ├── command/
│   │   ├── ChargeCommand
│   │   ├── RefundCommand
│   │   └── VoidCommand
│   └── repository/
│       └── TransactionWriteRepository  // otimizado para escrita
├── query/                       // READ SIDE
│   ├── model/
│   │   ├── TransactionSummary   // projeção simplificada
│   │   ├── TransactionDetail    // projeção completa
│   │   └── TransactionReport    // projeção analítica
│   ├── handler/
│   │   ├── GetTransactionQueryHandler
│   │   ├── ListTransactionsQueryHandler
│   │   └── TransactionReportQueryHandler
│   ├── query/
│   │   ├── GetTransactionQuery
│   │   └── ListTransactionsQuery
│   └── repository/
│       └── TransactionReadRepository   // otimizado para leitura
└── shared/
    ├── bus/
    │   ├── CommandBus
    │   └── QueryBus
    └── event/
        └── DomainEvent          // sincroniza read/write
```

**Command vs Query:**

| Aspecto | Command (Write) | Query (Read) |
|---------|-----------------|--------------|
| **Modelo** | `Transaction` (rico, com regras) | `TransactionSummary` (projeção, DTO) |
| **Retorno** | void ou ID | Dados |
| **Efeito** | Muda estado | Sem efeito colateral |
| **Repositório** | Otimizado para escrita | Otimizado para leitura (projections) |
| **Validação** | Completa (regras de negócio) | Mínima (parâmetros de query) |

**Requisitos:**
- `CommandBus` despacha commands para handlers
- `QueryBus` despacha queries para handlers
- Read model sincronizado via eventos ou projeção direta
- Write model rico (entidades do Domain); Read model leve (DTOs/projections)

**Critérios de aceite:**
- [ ] Modelos de escrita e leitura claramente separados
- [ ] `CommandBus` + `QueryBus` com dispatch por tipo
- [ ] ≥ 3 commands + ≥ 3 queries com handlers
- [ ] Read model: ≥ 2 projeções diferentes (summary, detail, report)
- [ ] Sincronização write → read via evento ou projeção automática
- [ ] Documentar: quando CQRS compensa vs overhead desnecessário
- [ ] Testes: command handlers, query handlers, sincronização read/write

---

### Desafio 5.6 — Event Sourcing: Transaction Event Store

> *"Armazenar o estado como sequência de eventos em vez de snapshot atual."*

**Cenário:** Em vez de armazenar o estado atual da transação, armazene TODOS os eventos que geraram esse estado. O estado atual é reconstruído (rehydrated) a partir do event stream. Isso provê audit trail completo e possibilidade de time-travel debugging.

**Eventos do domínio:**

| Evento | Dados | Efeito no estado |
|--------|-------|------------------|
| `TransactionCreated` | id, amount, paymentMethod, customer | Cria transação em CREATED |
| `TransactionAuthorized` | authCode, gatewayRef | Status → AUTHORIZED |
| `TransactionCaptured` | capturedAmount, capturedAt | Status → CAPTURED |
| `TransactionSettled` | settledAt, settlementRef | Status → SETTLED |
| `TransactionRefunded` | refundedAmount, reason | Status → REFUNDED |
| `TransactionCancelled` | reason, cancelledBy | Status → CANCELLED |
| `TransactionFailed` | error, failedAt | Status → FAILED |
| `FeeCalculated` | feeAmount, feeType | Adiciona fee à transação |

**Requisitos:**
- `EventStore` armazena eventos em ordem (append-only)
- `Transaction.apply(event)` aplica um evento e modifica estado
- `Transaction.rehydrate(events)` reconstrói estado a partir de stream
- Snapshot para performance: a cada N eventos, salva snapshot

**Critérios de aceite:**
- [ ] `EventStore` in-memory com append e `getEvents(aggregateId)`
- [ ] 8+ event types com dados específicos para cada mudança de estado
- [ ] `Transaction.apply(event)` modifica estado com base no evento
- [ ] `Transaction.rehydrate(events)` reconstrói estado completo do zero
- [ ] Event stream é append-only — eventos nunca são removidos ou alterados
- [ ] Snapshot a cada N eventos para otimizar reconstruct
- [ ] Integração com CQRS (5.5): eventos do write side projetam no read side
- [ ] Testes: rehydrate completo, snapshot + eventos delta, ordering

---

### Desafio 5.7 — Integração: Arquitetura Final Completa

**Cenário:** Combine os padrões arquiteturais em uma arquitetura coesa:

```
Clean Architecture (estrutura de camadas)
  └── Hexagonal Ports & Adapters (isolamento core)
        ├── CQRS (separação read/write)
        │     ├── Command Side + Event Sourcing (persistência por eventos)
        │     └── Query Side (projeções otimizadas)
        ├── MVC (interface HTTP)
        └── Event-Driven (comunicação assíncrona interna)
```

**Requisitos:**
- MVC na camada de apresentação (HTTP controllers + views)
- Clean Architecture organizando os anéis de dependência
- Hexagonal Ports & Adapters isolando o core
- CQRS + Event Sourcing no write side
- Projeções no read side atualizadas por eventos
- Event Bus in-memory para comunicação assíncrona

**Organização final:**

```
src/
├── entity/                      // Clean: Enterprise Rules
├── usecase/                     // Clean: Application Rules
│   ├── command/                 // CQRS: write side
│   │   ├── handler/
│   │   └── port/
│   └── query/                   // CQRS: read side
│       ├── handler/
│       └── port/
├── adapter/                     // Clean: Interface Adapters
│   ├── inbound/
│   │   ├── controller/          // MVC: Controllers
│   │   └── view/                // MVC: Views
│   └── outbound/
│       ├── eventstore/          // Event Sourcing: event store
│       ├── projection/          // CQRS: read model projections
│       └── gateway/             // Hexagonal: gateway adapters
└── framework/                   // Clean: Frameworks & Drivers
    ├── http/                    // HTTP server
    ├── eventbus/                // Event-Driven: in-memory bus
    └── main/                    // Bootstrap & composition
```

**Critérios de aceite:**
- [ ] Arquitetura final combina Clean + Hexagonal + CQRS + Event Sourcing + MVC
- [ ] `ARCHITECTURE.md` com diagrama das camadas e fluxo de dados
- [ ] Write side: command → handler → domain event → event store
- [ ] Read side: event → projection → read model → query handler
- [ ] HTTP API funcional com 5 endpoints (via MVC puro)
- [ ] Core testável sem nenhuma dependência de infraestrutura
- [ ] `DECISIONS.md`: ADR para cada decisão arquitetural + trade-offs

---

## Entregáveis

| Artefato | Padrão |
|----------|--------|
| 4 camadas organizadas | Layered Architecture |
| Ports + Adapters com core isolado | Hexagonal Architecture |
| 4 anéis com Regra de Dependência | Clean Architecture |
| HTTP server puro com MVC | MVC |
| CommandBus + QueryBus + Read/Write Models | CQRS |
| EventStore + Event Stream + Rehydrate | Event Sourcing |
| Arquitetura completa integrada | Todos combinados |
| `ARCHITECTURE.md` + `DECISIONS.md` | Documentação |

---

## Próximo Nível

→ [Level 6 — Boas Práticas & Integração](06-best-practices-integration.md)
