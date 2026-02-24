# Domain-Driven Design — Arquitetura & Integração

> **Objetivo deste documento:** Servir como referência completa sobre como **DDD se integra com padrões arquiteturais modernos**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Abordagem **agnóstica de linguagem** — foca em conceitos, motivações, heurísticas, boas práticas e **pseudocódigo/diagramas ilustrativos** sem vínculo a tecnologia específica.

---

## Quick Reference — Arquiteturas x DDD

| Arquitetura | Sinergia com DDD | Quando usar |
|-------------|------------------|-------------|
| **Layered (Camadas)** | Básica — separa domínio de infra | Aplicações monolíticas tradicionais |
| **Hexagonal (Ports & Adapters)** | Forte — domínio no centro, infra nos adapters | Quando testabilidade e isolamento são prioridade |
| **Clean Architecture** | Forte — regra de dependência protege domínio | Quando múltiplas interfaces (API, CLI, eventos) |
| **CQRS** | Forte — separa escrita (domínio rico) de leitura | Domínios com leitura/escrita assimétricas |
| **Event Sourcing** | Complementar — persiste eventos de domínio | Audit trail, time-travel, domínio orientado a eventos |
| **Microservices** | Natural — Bounded Context ≈ service boundary | Quando autonomia e escalabilidade são requisitos |
| **Modular Monolith** | Excelente — Bounded Contexts como módulos | Início de projeto; antes de migrar para microservices |

---

## Sumário

- [Domain-Driven Design — Arquitetura \& Integração](#domain-driven-design--arquitetura--integração)
  - [Quick Reference — Arquiteturas x DDD](#quick-reference--arquiteturas-x-ddd)
  - [Sumário](#sumário)
  - [Layered Architecture](#layered-architecture)
  - [Hexagonal Architecture (Ports \& Adapters)](#hexagonal-architecture-ports--adapters)
  - [Clean Architecture / Onion Architecture](#clean-architecture--onion-architecture)
  - [CQRS — Command Query Responsibility Segregation](#cqrs--command-query-responsibility-segregation)
  - [Event Sourcing](#event-sourcing)
  - [CQRS + Event Sourcing combinados](#cqrs--event-sourcing-combinados)
  - [Microservices e DDD](#microservices-e-ddd)
  - [Modular Monolith](#modular-monolith)
  - [Event-Driven Architecture](#event-driven-architecture)
  - [Estrutura de projeto recomendada](#estrutura-de-projeto-recomendada)
  - [Application Service — O orquestrador](#application-service--o-orquestrador)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Layered Architecture

A arquitetura em camadas clássica com DDD:

```
┌─────────────────────────────────────────┐
│           PRESENTATION LAYER            │
│  Controllers, Views, DTOs de request    │
├─────────────────────────────────────────┤
│           APPLICATION LAYER             │
│  Use Cases, Application Services        │
│  Orquestração, transação, segurança     │
├─────────────────────────────────────────┤
│             DOMAIN LAYER                │  ◄── Coração do DDD
│  Entities, Value Objects, Aggregates    │
│  Domain Services, Domain Events         │
│  Repository interfaces                  │
├─────────────────────────────────────────┤
│          INFRASTRUCTURE LAYER           │
│  Repository implementations, ORM        │
│  Message brokers, HTTP clients          │
│  Framework configurations               │
└─────────────────────────────────────────┘
```

### Regra de dependência

```
Presentation → Application → Domain ← Infrastructure

O Domain Layer NÃO depende de nada. 
Infrastructure IMPLEMENTA interfaces definidas no Domain.
```

### Responsabilidades por camada

| Camada | Responsabilidade | Contém | NÃO contém |
|--------|-----------------|--------|------------|
| **Presentation** | Interface com o usuário/cliente | Controllers, DTOs, validação de formato | Lógica de negócio |
| **Application** | Orquestração de use cases | Application Services, Commands, Queries | Regras de negócio, acesso a dados direto |
| **Domain** | Regras de negócio | Entities, VOs, Aggregates, Domain Services, Events, Repository interfaces | Frameworks, SQL, HTTP, dependências externas |
| **Infrastructure** | Implementações técnicas | Repository impls, ORM config, Message broker, API clients | Regras de negócio |

### Boas Práticas

1. **Domain Layer é puro** — Zero dependências de framework ou infraestrutura
2. **Application Service é fino** — Apenas orquestra; 10-20 linhas tipicamente
3. **DTOs na fronteira** — Mapeie DTOs ↔ Domain Objects nas bordas (Application/Presentation)
4. **Interface no domínio, implementação na infra** — Dependency Inversion em ação

---

## Hexagonal Architecture (Ports & Adapters)

Criada por **Alistair Cockburn**, coloca o domínio no centro com "ports" (interfaces) e "adapters" (implementações).

```
                    ┌──────────────────────────┐
                    │        PRIMARY           │
                    │       ADAPTERS            │
                    │  (driving / de entrada)   │
                    │                          │
                    │  REST Controller         │
                    │  GraphQL Resolver        │
                    │  CLI Command             │
                    │  Event Consumer          │
                    └──────────┬───────────────┘
                               │
                    ┌──────────▼───────────────┐
                    │     PRIMARY PORTS         │
                    │  (Input / Use Case)       │
                    │                          │
                    │  PlaceOrderUseCase       │
                    │  CancelOrderUseCase      │
                    ├──────────────────────────┤
                    │                          │
                    │      DOMAIN CORE         │
                    │                          │
                    │  Entities                │
                    │  Value Objects           │
                    │  Aggregates              │
                    │  Domain Events           │
                    │  Domain Services         │
                    │                          │
                    ├──────────────────────────┤
                    │    SECONDARY PORTS        │
                    │  (Output / Driven)        │
                    │                          │
                    │  OrderRepository         │
                    │  PaymentGateway          │
                    │  EventPublisher          │
                    └──────────┬───────────────┘
                               │
                    ┌──────────▼───────────────┐
                    │      SECONDARY            │
                    │       ADAPTERS            │
                    │  (driven / de saída)      │
                    │                          │
                    │  PostgresOrderRepository │
                    │  StripePaymentGateway    │
                    │  KafkaEventPublisher     │
                    └──────────────────────────┘
```

### Conceitos-chave

| Conceito | Descrição |
|----------|-----------|
| **Port** | Interface que define como o mundo externo interage com o domínio (entrada) ou como o domínio interagem com o mundo externo (saída) |
| **Primary Port** | Define os use cases disponíveis (interface de entrada) |
| **Secondary Port** | Define o que o domínio precisa do mundo externo (repository, gateway) |
| **Primary Adapter** | Implementa a chamada ao port primário (REST, CLI, event consumer) |
| **Secondary Adapter** | Implementa o port secundário (banco, API externa, broker) |

### Boas Práticas

1. **Domínio não sabe da infraestrutura** — Nenhum import de framework no core
2. **Ports são interfaces do domínio** — Definidas na camada de domínio
3. **Adapters são descartáveis** — Trocar Postgres por MongoDB = novo adapter, domínio intacto
4. **Testes de domínio sem infraestrutura** — Use adapters fake/in-memory para testar

### Exemplo (pseudocódigo)

```
// ═══ Primary Port (Input) ═══
interface PlaceOrderUseCase:
    execute(command: PlaceOrderCommand): OrderId

// ═══ Secondary Port (Output) ═══
interface OrderRepository:
    save(order: Order): void
    findById(id: OrderId): Order?

interface PaymentGateway:
    charge(amount: Money, method: PaymentMethod): PaymentResult

// ═══ Application Service implementa Primary Port ═══
class PlaceOrderService implements PlaceOrderUseCase:
    orderRepository: OrderRepository      // Secondary port
    paymentGateway: PaymentGateway        // Secondary port

    execute(command: PlaceOrderCommand): OrderId
        order = Order.create(
            command.customerId, 
            command.items, 
            command.shippingAddress)
        
        paymentResult = paymentGateway.charge(
            order.totalAmount(), command.paymentMethod)
        
        order.confirmPayment(paymentResult.transactionId)
        orderRepository.save(order)
        
        return order.id

// ═══ Primary Adapter (REST) ═══
class OrderController:
    placeOrderUseCase: PlaceOrderUseCase

    POST /orders (request: PlaceOrderRequest):
        command = mapToCommand(request)
        orderId = placeOrderUseCase.execute(command)
        return Response(201, { orderId: orderId })

// ═══ Secondary Adapter (Database) ═══
class PostgresOrderRepository implements OrderRepository:
    save(order: Order): void
        // SQL/ORM implementation

// ═══ Secondary Adapter (External API) ═══
class StripePaymentGateway implements PaymentGateway:
    charge(amount: Money, method: PaymentMethod): PaymentResult
        // Stripe API call + ACL translation
```

---

## Clean Architecture / Onion Architecture

Variação da Hexagonal proposta por **Robert C. Martin**, com a **Dependency Rule**: dependências apontam para dentro (domínio).

```
┌─────────────────────────────────────────────────────┐
│                FRAMEWORKS & DRIVERS                  │
│  Web Framework, DB Driver, UI, External APIs         │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │          INTERFACE ADAPTERS                  │    │
│  │  Controllers, Gateways, Presenters           │    │
│  │                                             │    │
│  │  ┌─────────────────────────────────────┐    │    │
│  │  │        APPLICATION LAYER            │    │    │
│  │  │    Use Cases / Application Services │    │    │
│  │  │                                     │    │    │
│  │  │  ┌─────────────────────────────┐    │    │    │
│  │  │  │      DOMAIN LAYER           │    │    │    │
│  │  │  │                             │    │    │    │
│  │  │  │  Entities, Value Objects    │    │    │    │
│  │  │  │  Domain Services            │    │    │    │
│  │  │  │  Domain Events              │    │    │    │
│  │  │  │  Repository Interfaces      │    │    │    │
│  │  │  │                             │    │    │    │
│  │  │  └─────────────────────────────┘    │    │    │
│  │  └─────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘

Dependency Rule: → → → para dentro (nunca para fora)
```

### Regras

1. Camadas internas **nunca** conhecem camadas externas
2. Dados cruzam fronteiras via **DTOs/Value Objects simples**
3. O domínio é o centro — estável, testável, framework-free

---

## CQRS — Command Query Responsibility Segregation

> *"Use um modelo para ler informações e outro modelo diferente para atualizar informações."*
> — Greg Young

### Conceito

CQRS separa o modelo de **escrita** (Commands) do modelo de **leitura** (Queries):

```
┌──────────────────────────────────────────────┐
│                   CLIENT                      │
│                                              │
│  ┌─────────────┐          ┌─────────────┐    │
│  │   COMMAND    │          │    QUERY     │    │
│  │   (Write)    │          │   (Read)     │    │
│  └──────┬───────┘          └──────┬───────┘    │
└─────────┼──────────────────────────┼──────────┘
          │                          │
          ▼                          ▼
┌─────────────────┐        ┌─────────────────┐
│  COMMAND HANDLER │        │  QUERY HANDLER  │
│                 │        │                 │
│  Domain Model   │        │  Read Model     │
│  (Aggregates,   │        │  (Projections,  │
│   rich logic)   │        │   views, DTOs)  │
│                 │        │                 │
│  Write Store    │        │  Read Store     │
│  (normalized)   │        │  (denormalized) │
└─────────────────┘        └─────────────────┘
```

### Quando usar CQRS

| Sim | Não |
|-----|-----|
| Leitura e escrita têm necessidades diferentes de performance | CRUD simples sem regras de negócio |
| Modelo de leitura é muito diferente do modelo de escrita | Aplicação pequena com poucos usuários |
| Necessidade de read replicas otimizadas | Equipe sem experiência com o pattern |
| Domain model rico com Aggregates | Leitura e escrita são simétricos |

### Boas Práticas com DDD

1. **Commands modificam Aggregates** — O lado de escrita é o DDD tático completo
2. **Queries não passam pelo domínio** — Read side pode consultar views/projections direto
3. **Projections são eventualmente consistentes** — Atualizadas via Domain Events
4. **Comandos retornam void (ou ID)** — Não retornam dados de leitura

### Exemplo (pseudocódigo)

```
// ═══ COMMAND SIDE (Write - DDD tático) ═══

command PlaceOrder:
    customerId: CustomerId
    items: List<{productId, quantity}>
    shippingAddress: Address

class PlaceOrderHandler:
    orderRepository: OrderRepository

    handle(command: PlaceOrder):
        order = Order.create(command.customerId, command.items, command.shippingAddress)
        orderRepository.save(order)
        // Domain Events publicados: OrderCreated


// ═══ QUERY SIDE (Read - Projections otimizadas) ═══

query GetOrderDetails:
    orderId: OrderId

class GetOrderDetailsHandler:
    readStore: OrderReadStore  // View denormalizada

    handle(query: GetOrderDetails): OrderDetailsDTO
        return readStore.getById(query.orderId)
        // Retorna DTO flat, otimizado para leitura
        // Sem carregar Aggregate, sem regras de negócio


// ═══ PROJECTION (Atualiza read store via eventos) ═══

class OrderProjection:
    readStore: OrderReadStore

    on(event: OrderCreated):
        readStore.insert(OrderDetailsDTO(
            orderId = event.orderId,
            customerName = event.customerName,   // denormalizado
            items = event.items,
            totalAmount = event.totalAmount,
            status = "CREATED",
            createdAt = event.occurredAt
        ))

    on(event: OrderConfirmed):
        readStore.updateStatus(event.orderId, "CONFIRMED")

    on(event: OrderCancelled):
        readStore.updateStatus(event.orderId, "CANCELLED")
```

---

## Event Sourcing

> Em vez de armazenar o **estado atual**, armazena todos os **eventos** que levaram ao estado atual.

```
Abordagem tradicional:
  Account { id: 1, balance: 750 }  ← estado atual (lost history)

Event Sourcing:
  AccountCreated  { id: 1, balance: 0 }
  AmountDeposited { id: 1, amount: 1000 }
  AmountWithdrawn { id: 1, amount: 200 }
  AmountWithdrawn { id: 1, amount: 50 }
  ─────────────────────────────────────
  Replay → balance: 750  ← estado reconstruído dos eventos
```

### Quando usar

| Sim | Não |
|-----|-----|
| Audit trail é requisito de negócio (financeiro, legal) | CRUD simples sem necessidade de histórico |
| Precisa de time-travel (ver estado em qualquer momento) | Operações de leitura são 90%+ do workload |
| Domínio naturalmente orientado a eventos | Equipe sem experiência com o pattern |
| Debugging complexo requer replay de eventos | Generic/Supporting subdomains |

### Boas Práticas

1. **Aggregate reconstrói estado a partir de eventos:**

```
class Account:
    id: AccountId
    balance: Money
    version: Integer

    // Reconstituição
    static fromEvents(events: List<DomainEvent>): Account
        account = new Account()
        for event in events:
            account.apply(event)
        return account

    // Aplicar evento (sem side-effects)
    apply(event: DomainEvent):
        match event:
            AccountCreated   → this.id = event.accountId; this.balance = Money.ZERO
            AmountDeposited  → this.balance = this.balance.add(event.amount)
            AmountWithdrawn  → this.balance = this.balance.subtract(event.amount)

    // Comando de domínio (gera novo evento)
    deposit(amount: Money):
        require amount.isPositive()
        this.raiseEvent(AmountDeposited(this.id, amount, now()))

    withdraw(amount: Money):
        require this.balance.isGreaterOrEqual(amount), "Insufficient funds"
        this.raiseEvent(AmountWithdrawn(this.id, amount, now()))
```

2. **Eventos são imutáveis** — Nunca altere ou delete eventos
3. **Snapshots para performance** — Periodicamente salve estado completo para evitar replay de milhares de eventos
4. **Upcasting para evolução** — Quando o formato de evento muda, transforme eventos antigos (upcasting)

---

## CQRS + Event Sourcing combinados

```
┌─────────────┐     ┌──────────────────┐     ┌────────────────┐
│   COMMAND    │────►│  COMMAND HANDLER │────►│   EVENT STORE  │
│             │     │                  │     │                │
│ PlaceOrder  │     │ Order.create()   │     │ OrderCreated   │
│             │     │ order.confirm()  │     │ OrderConfirmed │
└─────────────┘     └──────────────────┘     └───────┬────────┘
                                                     │
                                              Domain Events
                                                     │
                                                     ▼
                                             ┌───────────────┐
                                             │  PROJECTION   │
                                             │               │
                                             │ Builds read   │
                                             │ model from    │
                                             │ events        │
                                             └───────┬───────┘
                                                     │
                                                     ▼
┌─────────────┐     ┌──────────────────┐     ┌───────────────┐
│   QUERY     │────►│  QUERY HANDLER   │────►│  READ STORE   │
│             │     │                  │     │               │
│ GetOrder    │     │ readStore.get()  │     │ Denormalized  │
│ Details     │     │                  │     │ views/tables  │
└─────────────┘     └──────────────────┘     └───────────────┘
```

---

## Microservices e DDD

### Bounded Context ≈ Service Boundary

> *"A microservice should be no bigger than my head."* — James Lewis

A fronteira de um microservice geralmente corresponde a um **Bounded Context**:

| Conceito DDD | Correspondência em Microservices |
|-------------|--------------------------------|
| Bounded Context | Service boundary |
| Ubiquitous Language | API/Contract vocabulary |
| Context Map | Service dependency map |
| Anti-Corruption Layer | API Gateway / Adapter |
| Domain Events | Async messages entre services |
| Shared Kernel | Shared library (com cuidado) |

### Boas Práticas

1. **Database per Service** — Cada microservice (= Bounded Context) tem seu próprio banco
2. **Eventos para sincronização** — Comunicação assíncrona via Domain Events
3. **API como contrato** — OpenAPI/AsyncAPI define o "Published Language"
4. **ACL em cada service** — Traduza modelos externos na fronteira
5. **Não comece com microservices** — Comece com **Modular Monolith**, extraia quando necessário

### Quando NÃO usar microservices

- Equipe pequena (< 5-8 devs)
- Startup em fase de descoberta (domínio ainda não está claro)
- Complexidade operacional é maior que complexidade do domínio
- Quando um Modular Monolith resolve

---

## Modular Monolith

> O melhor dos dois mundos: fronteiras claras de DDD com simplicidade operacional.

```
┌─────────────────────────────────────────────────────┐
│                 MONOLITH DEPLOYMENT                   │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │   MODULE:    │  │   MODULE:    │  │  MODULE:   │ │
│  │   SALES      │  │  INVENTORY   │  │  BILLING   │ │
│  │              │  │              │  │            │ │
│  │ ┌──────────┐│  │ ┌──────────┐ │  │┌──────────┐│ │
│  │ │ Domain   ││  │ │ Domain   │ │  ││ Domain   ││ │
│  │ │ Model    ││  │ │ Model    │ │  ││ Model    ││ │
│  │ └──────────┘│  │ └──────────┘ │  │└──────────┘│ │
│  │ ┌──────────┐│  │ ┌──────────┐ │  │┌──────────┐│ │
│  │ │ App Layer││  │ │ App Layer│ │  ││ App Layer││ │
│  │ └──────────┘│  │ └──────────┘ │  │└──────────┘│ │
│  │ ┌──────────┐│  │ ┌──────────┐ │  │┌──────────┐│ │
│  │ │ Infra    ││  │ │ Infra    │ │  ││ Infra    ││ │
│  │ └──────────┘│  │ └──────────┘ │  │└──────────┘│ │
│  │             │  │              │  │            │ │
│  │ Own Schema  │  │ Own Schema   │  │ Own Schema │ │
│  └──────┬──────┘  └──────┬───────┘  └─────┬──────┘ │
│         │                │                │        │
│         └────── Events ──┴───── Events ───┘        │
│                  (in-process event bus)              │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │            SHARED DATABASE SERVER            │    │
│  │  (mas schemas separados por módulo!)         │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

### Regras de um Modular Monolith

1. **Módulos = Bounded Contexts** — Cada módulo tem seu próprio domínio
2. **Comunicação via interface pública** — Módulos não acessam internals uns dos outros
3. **Schema por módulo** — Mesmo DB server, schemas separados
4. **Events in-process** — Publicação de eventos pode ser síncrona inicialmente
5. **Preparado para extração** — Se um módulo precisa ser extraído para microservice, a fronteira já existe

### Boas Práticas

- Enforçe fronteiras via **ArchUnit** / testes de arquitetura
- Módulos expõem **API pública** (interface/facade), internals são `package-private`/`internal`
- Compartilhe apenas **tipos de integração** (IDs, eventos), nunca modelos internos

---

## Event-Driven Architecture

DDD com arquitetura orientada a eventos:

```
┌──────────┐  OrderPlaced  ┌─────────────┐
│  SALES   │──────────────►│   MESSAGE   │
│  Context │               │   BROKER    │
└──────────┘               │             │
                           │  (Kafka,    │
┌──────────┐  PaymentDone  │  RabbitMQ,  │
│ BILLING  │──────────────►│  SQS/SNS)   │
│ Context  │               │             │
└──────────┘               └──────┬──────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
                    ▼             ▼             ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │INVENTORY │ │SHIPPING  │ │NOTIFIC.  │
              │ Context  │ │ Context  │ │ Context  │
              └──────────┘ └──────────┘ └──────────┘
```

### Padrões de entrega

| Padrão | Descrição | Garantia |
|--------|-----------|----------|
| **At-most-once** | Faz melhor esforço, pode perder | Sem duplicatas |
| **At-least-once** | Garante entrega, pode duplicar | Consumidor deve ser idempotente |
| **Exactly-once** | Garante entrega sem duplicação | Complexo; geralmente at-least-once + idempotência |

### Transactional Outbox

Padrão para garantir consistência entre persistência do Aggregate e publicação de eventos:

```
┌──────────────────────────────────────────────┐
│               SAME TRANSACTION               │
│                                              │
│  1. Save Aggregate ──► Orders table          │
│  2. Save Event     ──► Outbox table          │
│                                              │
└──────────────────────────────────────────────┘
                    │
                    │  (Poller ou CDC)
                    ▼
           ┌───────────────┐
           │ Message Broker │
           └───────────────┘
```

### Boas Práticas

1. **Eventos como contratos** — Schema versionado (Avro, Protobuf, JSON Schema)
2. **Idempotência** — Todo consumidor deve ser idempotente
3. **Dead Letter Queue** — Eventos que falharam vão para DLQ para análise
4. **Ordering** — Garantir ordem de eventos por Aggregate ID (partition key)
5. **Schema Registry** — Registrar e validar schemas de eventos
6. **Outbox Pattern** — Para consistência entre DB e broker

---

## Estrutura de projeto recomendada

### Por Bounded Context (recomendado)

```
src/
├── sales/                          # Bounded Context: Sales
│   ├── domain/                     # Camada de Domínio
│   │   ├── model/
│   │   │   ├── Order.ext           # Aggregate Root
│   │   │   ├── OrderItem.ext       # Entity interna
│   │   │   ├── OrderId.ext         # Value Object (ID)
│   │   │   ├── OrderStatus.ext     # Value Object (enum)
│   │   │   └── Money.ext           # Value Object
│   │   ├── event/
│   │   │   ├── OrderCreated.ext    # Domain Event
│   │   │   ├── OrderConfirmed.ext
│   │   │   └── OrderCancelled.ext
│   │   ├── service/
│   │   │   └── DiscountPolicy.ext  # Domain Service
│   │   └── repository/
│   │       └── OrderRepository.ext # Interface (Port)
│   │
│   ├── application/                # Camada de Aplicação
│   │   ├── command/
│   │   │   ├── PlaceOrderCommand.ext
│   │   │   └── PlaceOrderHandler.ext
│   │   ├── query/
│   │   │   ├── GetOrderQuery.ext
│   │   │   └── GetOrderHandler.ext
│   │   └── dto/
│   │       └── OrderDTO.ext
│   │
│   └── infrastructure/             # Camada de Infraestrutura
│       ├── persistence/
│       │   └── SqlOrderRepository.ext
│       ├── messaging/
│       │   └── KafkaOrderEventPublisher.ext
│       └── api/
│           └── OrderController.ext
│
├── inventory/                      # Bounded Context: Inventory
│   ├── domain/
│   ├── application/
│   └── infrastructure/
│
├── billing/                        # Bounded Context: Billing
│   ├── domain/
│   ├── application/
│   └── infrastructure/
│
└── shared/                         # Shared Kernel (mínimo!)
    └── kernel/
        ├── Money.ext
        ├── EventPublisher.ext
        └── DomainEvent.ext
```

### Regras da estrutura

1. **Domínio não importa infraestrutura** — Nenhum import de framework/DB no domain/
2. **Módulos não acessam internals uns dos outros** — Sales não importa `inventory.domain.model`
3. **Comunicação entre módulos via interfaces públicas ou eventos**
4. **Shared kernel é mínimo** — Apenas o estritamente necessário

---

## Application Service — O orquestrador

O Application Service (ou Use Case Handler) é a **cola** entre o mundo externo e o domínio:

```
class PlaceOrderHandler:                        // Application Service
    orderRepository: OrderRepository             // Port (interface)
    customerRepository: CustomerRepository       // Port (interface)
    eventPublisher: EventPublisher               // Port (interface)

    handle(command: PlaceOrderCommand): OrderId
        // 1. Buscar dados
        customer = customerRepository.findById(command.customerId)
            ?? throw CustomerNotFound(command.customerId)

        // 2. Chamar domínio (REGRA DE NEGÓCIO ESTÁ AQUI)
        order = customer.placeOrder(
            command.items, 
            command.shippingAddress)

        // 3. Persistir
        orderRepository.save(order)

        // 4. Publicar eventos
        eventPublisher.publishAll(order.domainEvents)

        // 5. Retornar resultado
        return order.id

    // NOTA: Application Service NÃO contém regras de negócio
    // Apenas: buscar → delegar ao domínio → salvar → publicar
```

### O que NÃO deve estar no Application Service

| Pertence ao Application Service | NÃO pertence |
|--------------------------------|--------------|
| Buscar Aggregate do Repository | if/else com regras de negócio |
| Chamar método no Aggregate | Validações de domínio |
| Salvar no Repository | Cálculos de negócio |
| Publicar Domain Events | Lógica que deveria estar no Aggregate |
| Gerenciar transação | Acesso direto a banco de dados |
| Autenticação/autorização | Regras de pricing, desconto, etc. |

---

## Diretrizes para Code Review assistido por AI

> Regras acionáveis para identificar violações arquiteturais em projetos DDD.

| # | Regra de detecção | Conceito | Severidade | Ação sugerida |
|---|-------------------|----------|-----------|---------------|
| 1 | Import de framework/DB na camada de domínio | Dependency Rule | **Crítico** | Remover dependência; usar interface/port |
| 2 | Application Service com if/else de regras de negócio | Layered Architecture | **Alto** | Mover lógica para Entity/Domain Service |
| 3 | Application Service com mais de 30 linhas | Application Layer | **Médio** | Provavelmente contém lógica de domínio; extrair |
| 4 | Controller acessa Repository diretamente | Hexagonal | **Alto** | Passar por Application Service/Use Case |
| 5 | Módulo importa internals de outro módulo | Bounded Context | **Alto** | Comunicar via interface pública ou eventos |
| 6 | Ausência de separação domain/ application/ infrastructure/ | Layered Architecture | **Médio** | Reorganizar em camadas |
| 7 | Domain Event publicado fora do Aggregate | Event-Driven | **Médio** | Mover publicação para dentro do Aggregate |
| 8 | Transação engloba múltiplos Aggregates | Aggregate Rules | **Alto** | Usar eventual consistency via Domain Events |
| 9 | Query side carrega Aggregate completo para ler dados | CQRS | **Médio** | Criar read model/projection otimizada |
| 10 | Módulos organizados por tipo técnico (entities/, services/) | Project Structure | **Médio** | Reorganizar por Bounded Context |
| 11 | Shared kernel com centenas de classes | Shared Kernel | **Alto** | Minimizar; cada contexto com modelo próprio |
| 12 | DTO do mundo externo usado como Domain Object | ACL | **Alto** | Criar modelo de domínio e traduzir na fronteira |

---

## Referências

| Recurso | Autor | Tipo |
|---------|-------|------|
| *Domain-Driven Design: Tackling Complexity in the Heart of Software* | Eric Evans | Livro |
| *Implementing Domain-Driven Design* | Vaughn Vernon | Livro |
| *Clean Architecture: A Craftsman's Guide to Software Structure and Design* | Robert C. Martin | Livro |
| *Hexagonal Architecture* (original) | Alistair Cockburn | Artigo |
| *CQRS Documents* | Greg Young | Paper |
| *Microservices Patterns* | Chris Richardson | Livro |
| *Building Microservices* (2nd ed.) | Sam Newman | Livro |
| *Learning Domain-Driven Design* | Vlad Khononov | Livro |
| *Software Architecture: The Hard Parts* | Ford, Richards, Sadalage, Dehghani | Livro |
| *Monolith to Microservices* | Sam Newman | Livro |
