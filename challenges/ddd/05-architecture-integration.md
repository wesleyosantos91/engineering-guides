# Level 5 — Architecture & Integration

> **Objetivo:** Implementar o MerchantHub usando arquiteturas que maximizam o isolamento do
> domínio — Hexagonal (Ports & Adapters), Clean Architecture, CQRS e Event Sourcing —
> organizando o código como Modular Monolith pronto para eventual extração em Microservices.

**Referência:** [05-architecture-integration.md](../../.docs/ddd/05-architecture-integration.md)

**Pré-requisito:** [Level 4 — Anti-Patterns & Refactoring](04-anti-patterns.md) completo.

---

## Contexto do Domínio

Até agora você modelou o domínio (Levels 0-2), integrou Bounded Contexts (Level 3) e
refatorou anti-patterns (Level 4). Agora é hora de **organizar esse código em uma
arquitetura que proteja o domínio** de detalhes de infraestrutura, frameworks e UI.

O MerchantHub será implementado como **Modular Monolith** com a opção de extrair
módulos para microservices no futuro.

---

## Desafios

### Desafio 5.1 — Hexagonal Architecture (Ports & Adapters)

Implemente o Bounded Context **Sales** usando Hexagonal Architecture.

**Requisitos:**
- **Domínio** no centro (nenhuma dependência para fora)
- **Ports** (interfaces) definidos pelo domínio:
  - **Inbound Ports** (Driving): casos de uso que o mundo externo pode invocar
  - **Outbound Ports** (Driven): interfaces que o domínio precisa do mundo externo
- **Adapters** na camada externa:
  - **Inbound Adapters**: REST controller, CLI, testes
  - **Outbound Adapters**: InMemory Repository, Event Publisher, Identity Client
- **Regra de Dependência**: adapters dependem de ports, nunca o contrário

**Estrutura de pacotes/diretórios:**

```
sales/
├── domain/                      # Núcleo — ZERO dependências externas
│   ├── model/
│   │   ├── Order.java           # Aggregate Root
│   │   ├── OrderItem.java       # Entidade interna
│   │   ├── OrderId.java         # Value Object
│   │   └── OrderStatus.java     # Enum
│   ├── event/
│   │   ├── OrderConfirmed.java  # Domain Event
│   │   └── OrderCancelled.java
│   └── service/
│       └── PricingService.java  # Domain Service
│
├── port/
│   ├── inbound/                 # Driving Ports (Use Cases)
│   │   ├── PlaceOrderUseCase.java
│   │   ├── ConfirmOrderUseCase.java
│   │   └── CancelOrderUseCase.java
│   └── outbound/                # Driven Ports (SPIs)
│       ├── OrderRepository.java
│       ├── EventPublisher.java
│       └── CatalogClient.java   # Port para acessar Catalog BC
│
├── application/                 # Implementação dos Inbound Ports
│   ├── PlaceOrderService.java   # implementa PlaceOrderUseCase
│   ├── ConfirmOrderService.java
│   └── CancelOrderService.java
│
└── adapter/
    ├── inbound/                 # Driving Adapters
    │   ├── rest/
    │   │   └── OrderController.java
    │   └── cli/
    │       └── OrderCLI.java
    └── outbound/                # Driven Adapters
        ├── persistence/
        │   └── InMemoryOrderRepository.java
        ├── messaging/
        │   └── InMemoryEventPublisher.java
        └── catalog/
            └── InProcessCatalogClient.java
```

**Java 25:**
```java
// --- Inbound Port (Driving) ---
public interface PlaceOrderUseCase {
    OrderId execute(PlaceOrderCommand command);
}

public record PlaceOrderCommand(CustomerId customerId, Address shippingAddress,
        List<OrderItemRequest> items) {}

// --- Outbound Port (Driven) ---
public interface CatalogClient {
    Optional<CatalogProductView> findProduct(String productId);
}

// --- Application Service (implementa Inbound Port) ---
public class PlaceOrderService implements PlaceOrderUseCase {
    private final OrderRepository orderRepo;      // Outbound Port
    private final CatalogClient catalogClient;     // Outbound Port
    private final PricingService pricingService;   // Domain Service

    @Override
    public OrderId execute(PlaceOrderCommand command) {
        // Orquestração: buscar produtos, calcular preço, criar Order, salvar
        var order = Order.create(
            orderRepo.nextId(), command.customerId(),
            command.shippingAddress(), resolvedItems);
        orderRepo.save(order);
        return order.id();
    }
}

// --- Inbound Adapter (REST) ---
public class OrderController {
    private final PlaceOrderUseCase placeOrder; // depende do PORT, não da implementação

    public OrderResponse handlePost(OrderRequest request) {
        var command = new PlaceOrderCommand(
            CustomerId.from(request.customerId()),
            Address.from(request.address()),
            request.items().stream().map(this::toOrderItemRequest).toList());
        var orderId = placeOrder.execute(command);
        return new OrderResponse(orderId.value());
    }
}

// --- Outbound Adapter ---
public class InProcessCatalogClient implements CatalogClient {
    private final CatalogService catalogService; // acessa Catalog BC diretamente (monolith)

    @Override
    public Optional<CatalogProductView> findProduct(String productId) {
        return Optional.ofNullable(catalogService.getProduct(productId));
    }
}
```

**Go 1.26:**
```go
// Inbound Port
type PlaceOrderUseCase interface {
    Execute(cmd PlaceOrderCommand) (OrderId, error)
}

// Outbound Port
type CatalogClient interface {
    FindProduct(productID string) (*CatalogProductView, error)
}

// Application Service
type PlaceOrderService struct {
    orderRepo      OrderRepository
    catalogClient  CatalogClient
    pricingService PricingService
}

func (s *PlaceOrderService) Execute(cmd PlaceOrderCommand) (OrderId, error) { ... }
```

**Critérios de aceite:**
- [ ] Domínio sem imports de framework, IO ou infraestrutura
- [ ] Inbound Ports como interfaces para cada caso de uso
- [ ] Outbound Ports como interfaces que o domínio define (DIP)
- [ ] Adapters inbound dependem dos Ports, não do domínio diretamente
- [ ] Adapters outbound implementam Ports definidos pelo domínio
- [ ] Teste do domínio sem qualquer adapter (unit test puro)
- [ ] Teste do adapter com mock/stub dos ports
- [ ] Verificação arquitetural: nenhuma dependência de adapter → domain

---

### Desafio 5.2 — Clean Architecture: Camadas e Use Cases

Implemente o Bounded Context **Catalog** usando Clean Architecture.

**Requisitos:**
- 4 camadas concêntricas (de dentro para fora):
  1. **Entities** (Enterprise Business Rules) — Aggregates, VOs, Domain Services
  2. **Use Cases** (Application Business Rules) — orquestração, DTOs de entrada/saída
  3. **Interface Adapters** — Controllers, Presenters, Gateways
  4. **Frameworks & Drivers** — InMemory impl, REST, CLI
- **Dependency Rule**: dependências apontam para dentro (nunca para fora)
- **Use Cases** retornam **Output DTOs**, não Entities (nunca expõe o domínio)

**Estrutura:**

```
catalog/
├── entity/                      # Camada 1: Enterprise Business Rules
│   ├── CatalogProduct.java
│   ├── ProductId.java
│   ├── CategoryId.java
│   └── ProductStatus.java
│
├── usecase/                     # Camada 2: Application Business Rules
│   ├── CreateProductUseCase.java
│   ├── CreateProductInput.java
│   ├── CreateProductOutput.java
│   ├── GetProductUseCase.java
│   ├── GetProductOutput.java
│   ├── ActivateProductUseCase.java
│   ├── ProductGateway.java      # Interface para persistência
│   └── EventGateway.java        # Interface para eventos
│
├── adapter/                     # Camada 3: Interface Adapters
│   ├── controller/
│   │   └── ProductController.java
│   ├── presenter/
│   │   └── ProductPresenter.java
│   └── gateway/
│       └── InMemoryProductGateway.java
│
└── driver/                      # Camada 4: Frameworks & Drivers
    └── rest/
        └── ProductRoutes.java
```

**Java 25:**
```java
// --- Use Case com Input/Output DTOs ---
public class CreateProductUseCase {
    private final ProductGateway productGateway;
    private final EventGateway eventGateway;

    public CreateProductOutput execute(CreateProductInput input) {
        var product = CatalogProduct.create(
            productGateway.nextId(),
            input.name(), input.description(),
            Money.of(input.price(), input.currency()),
            CategoryId.from(input.categoryId()));

        productGateway.save(product);
        eventGateway.publishAll(product.domainEvents());
        product.clearDomainEvents();

        // Retorna Output DTO — nunca a Entity
        return new CreateProductOutput(
            product.id().value(), product.name(), product.status().name());
    }
}

public record CreateProductInput(String name, String description,
        String price, String currency, String categoryId) {}

public record CreateProductOutput(String productId, String name,
        String status) {}

// Gateway (equiv. a Outbound Port na Hexagonal)
public interface ProductGateway {
    void save(CatalogProduct product);
    Optional<CatalogProduct> findById(ProductId id);
    ProductId nextId();
}
```

**Go 1.26:**
```go
type CreateProductUseCase struct {
    productGateway ProductGateway
    eventGateway   EventGateway
}

type CreateProductInput struct {
    Name, Description string
    Price, Currency   string
    CategoryID        string
}

type CreateProductOutput struct {
    ProductID, Name, Status string
}

func (uc *CreateProductUseCase) Execute(input CreateProductInput) (*CreateProductOutput, error) { ... }
```

**Critérios de aceite:**
- [ ] 4 camadas identificáveis na estrutura de diretórios
- [ ] Dependency Rule: nenhuma camada interna importa camada externa
- [ ] Use Cases com Input/Output DTOs (nunca expõe Entities)
- [ ] Gateways como interfaces (equivalentes a Outbound Ports)
- [ ] Use Cases testáveis com mocks de Gateway (sem infra)
- [ ] Comparação documentada: Clean Architecture vs Hexagonal (semelhanças, diferenças)
- [ ] Testes: unit (entity, use case), integration (adapter), e2e (driver)

---

### Desafio 5.3 — CQRS (Command Query Responsibility Segregation)

Implemente CQRS no **Sales** BC, separando modelo de escrita e leitura.

**Requisitos:**
- **Command Side** (Write Model):
  - Commands: `PlaceOrderCommand`, `ConfirmOrderCommand`, `CancelOrderCommand`
  - Command Handlers processam commands e modificam Aggregates
  - Persistência otimizada para escrita (normalizado)
- **Query Side** (Read Model):
  - Queries: `GetOrderQuery`, `ListOrdersByCustomerQuery`, `OrderSummaryQuery`
  - Query Handlers leem de uma projeção otimizada para leitura (desnormalizado)
  - Read Model atualizado por eventos do Write Model
- **Separação clara**: Command nunca retorna dados (void ou ID), Query nunca modifica estado

**Java 25:**
```java
// --- Command Side ---
public sealed interface OrderCommand permits
        PlaceOrderCommand, ConfirmOrderCommand, CancelOrderCommand {}

public record PlaceOrderCommand(CustomerId customerId, Address address,
        List<OrderItemRequest> items) implements OrderCommand {}

public interface CommandHandler<C extends OrderCommand> {
    void handle(C command);
}

public class PlaceOrderHandler implements CommandHandler<PlaceOrderCommand> {
    private final OrderRepository writeRepo; // repositório de escrita

    @Override
    public void handle(PlaceOrderCommand command) {
        var order = Order.create(...);
        writeRepo.save(order);
        // Eventos publicados → Read Model atualizado assincronamente
    }
}

// --- Query Side ---
public sealed interface OrderQuery permits
        GetOrderQuery, ListOrdersByCustomerQuery {}

public record GetOrderQuery(String orderId) implements OrderQuery {}

// Read Model — desnormalizado, otimizado para leitura
public record OrderReadModel(
    String orderId, String customerName, String status,
    String totalAmount, int itemCount, Instant createdAt,
    List<OrderItemReadModel> items
) {}

public interface QueryHandler<Q extends OrderQuery, R> {
    R handle(Q query);
}

public class GetOrderQueryHandler
        implements QueryHandler<GetOrderQuery, OrderReadModel> {
    private final OrderReadModelRepository readRepo; // repositório de leitura

    @Override
    public OrderReadModel handle(GetOrderQuery query) {
        return readRepo.findById(query.orderId())
            .orElseThrow(() -> new EntityNotFound("Order", query.orderId()));
    }
}

// --- Projeção (atualiza Read Model a partir de Domain Events) ---
public class OrderProjection implements EventHandler<DomainEvent> {
    private final OrderReadModelRepository readRepo;

    public void on(OrderConfirmed event) {
        readRepo.upsert(new OrderReadModel(
            event.orderId().value(),
            /* dados desnormalizados... */));
    }
}
```

**Go 1.26:**
```go
// Command Side
type PlaceOrderCommand struct {
    CustomerID CustomerId
    Address    Address
    Items      []OrderItemRequest
}

type PlaceOrderHandler struct {
    writeRepo OrderRepository
}

func (h *PlaceOrderHandler) Handle(cmd PlaceOrderCommand) error { ... }

// Query Side
type OrderReadModel struct {
    OrderID      string
    CustomerName string
    Status       string
    TotalAmount  string
    ItemCount    int
    CreatedAt    time.Time
}

type GetOrderQueryHandler struct {
    readRepo OrderReadModelRepository
}

func (h *GetOrderQueryHandler) Handle(orderID string) (*OrderReadModel, error) { ... }
```

**Critérios de aceite:**
- [ ] Command e Query em pacotes/módulos separados
- [ ] Commands não retornam dados (void ou apenas ID criado)
- [ ] Queries não modificam estado (read-only)
- [ ] Read Model desnormalizado (otimizado para queries comuns)
- [ ] Projeção atualiza Read Model a partir de Domain Events
- [ ] Write Model e Read Model podem usar stores diferentes
- [ ] Testes: commands alteram write model, projeção sincroniza read model, queries leem corretamente

---

### Desafio 5.4 — Event Sourcing

Implemente Event Sourcing para o Aggregate `Order` do Sales BC.

**Requisitos:**
- **Estado derivado de eventos**: Order não tem campos mutáveis — estado é reconstruído
  aplicando sequência de eventos
- **Event Store**: armazena eventos em append-only (nunca update/delete)
- Cada evento tem: `aggregateId`, `eventType`, `eventData`, `version`, `occurredAt`
- **Reconstituição**: para carregar um Order, buscar todos os eventos e aplicar em sequência
- **Snapshots**: otimização para aggregates com muitos eventos (a cada N eventos)
- Operações:
  - `save(aggregate)` → persiste novos eventos no event store
  - `load(aggregateId)` → busca eventos, aplica, retorna aggregate reconstituído
  - Optimistic concurrency via version

**Java 25:**
```java
public final class Order {
    private OrderId id;
    private CustomerId customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    private Money totalAmount;
    private int version;

    // Estado reconstruído a partir de eventos
    private Order() { this.items = new ArrayList<>(); }

    public static Order recreateFrom(List<DomainEvent> events) {
        var order = new Order();
        events.forEach(order::apply);
        return order;
    }

    // Comportamento gera novos eventos (não muda estado diretamente)
    public List<DomainEvent> confirm() {
        require(status == OrderStatus.PENDING, "Can only confirm PENDING orders");
        require(!items.isEmpty(), "Cannot confirm empty order");
        var event = new OrderConfirmed(id, customerId, totalAmount, Instant.now());
        apply(event); // aplica ao estado local
        return List.of(event); // retorna para persistir
    }

    // Apply atualiza estado interno a partir de um evento
    private void apply(DomainEvent event) {
        switch (event) {
            case OrderPlaced e -> {
                this.id = e.orderId();
                this.customerId = e.customerId();
                this.status = OrderStatus.PENDING;
            }
            case OrderItemAdded e -> {
                this.items.add(new OrderItem(e.productId(), e.quantity(), e.unitPrice()));
                this.totalAmount = recalculateTotal();
            }
            case OrderConfirmed e -> {
                this.status = OrderStatus.CONFIRMED;
            }
            case OrderCancelled e -> {
                this.status = OrderStatus.CANCELLED;
            }
            default -> { }
        }
        this.version++;
    }
}

// Event Store
public interface EventStore {
    void append(String aggregateId, List<DomainEvent> events, int expectedVersion);
    List<DomainEvent> loadEvents(String aggregateId);
    List<DomainEvent> loadEventsAfterVersion(String aggregateId, int afterVersion);
}

public class InMemoryEventStore implements EventStore {
    private final Map<String, List<StoredEvent>> store = new ConcurrentHashMap<>();

    @Override
    public void append(String aggregateId, List<DomainEvent> events, int expectedVersion) {
        var stored = store.computeIfAbsent(aggregateId, k -> new ArrayList<>());
        if (stored.size() != expectedVersion) {
            throw new ConcurrencyConflict(aggregateId, expectedVersion, stored.size());
        }
        for (var event : events) {
            stored.add(new StoredEvent(aggregateId, event, stored.size()));
        }
    }
}

// Event-Sourced Repository
public class EventSourcedOrderRepository implements OrderRepository {
    private final EventStore eventStore;

    @Override
    public void save(Order order) {
        eventStore.append(order.id().value(),
            order.uncommittedEvents(), order.loadedVersion());
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        var events = eventStore.loadEvents(id.value());
        if (events.isEmpty()) return Optional.empty();
        return Optional.of(Order.recreateFrom(events));
    }
}
```

**Go 1.26:**
```go
type Order struct {
    id                OrderId
    customerId        CustomerId
    items             []OrderItem
    status            OrderStatus
    totalAmount       Money
    version           int
    uncommittedEvents []DomainEvent
}

func RecreateOrderFrom(events []DomainEvent) (*Order, error) {
    order := &Order{}
    for _, event := range events {
        order.apply(event)
    }
    return order, nil
}

func (o *Order) Confirm() ([]DomainEvent, error) { ... }
func (o *Order) apply(event DomainEvent) { ... }

type EventStore interface {
    Append(aggregateID string, events []DomainEvent, expectedVersion int) error
    LoadEvents(aggregateID string) ([]DomainEvent, error)
}
```

**Critérios de aceite:**
- [ ] Aggregate sem campos mutáveis externamente — estado via apply de eventos
- [ ] Event Store append-only com optimistic concurrency (version check)
- [ ] Reconstituição funcional: `recreateFrom(events)` reproduz estado
- [ ] Comportamento retorna novos eventos + aplica ao estado local
- [ ] Event-Sourced Repository que salva eventos (não estado) e reconstrói aggregate
- [ ] Serialização de eventos (JSON) para persistência
- [ ] Testes: criação, reconstituição, concorrência, sequência de eventos

---

### Desafio 5.5 — CQRS + Event Sourcing Combinados

Combine CQRS (Desafio 5.3) e Event Sourcing (Desafio 5.4) em uma implementação integrada.

**Requisitos:**
- **Write Side**: Event Sourcing — Commands geram eventos, estado reconstruído de eventos
- **Read Side**: Projeções — Eventos projetados em Read Models desnormalizados
- **Flow completo**:
  1. Command recebido → Command Handler carrega Aggregate do Event Store
  2. Aggregate executa lógica → gera novos eventos
  3. Eventos salvos no Event Store (append)
  4. Eventos publicados para Projeções
  5. Projeções atualizam Read Models
  6. Queries leem dos Read Models

**Java 25:**
```java
// Flow integrado
public class ConfirmOrderHandler implements CommandHandler<ConfirmOrderCommand> {
    private final EventStore eventStore;
    private final EventPublisher eventPublisher;

    @Override
    public void handle(ConfirmOrderCommand command) {
        // 1. Carregar do Event Store
        var events = eventStore.loadEvents(command.orderId().value());
        var order = Order.recreateFrom(events);

        // 2. Executar lógica de domínio → novos eventos
        var newEvents = order.confirm();

        // 3. Append no Event Store
        eventStore.append(command.orderId().value(), newEvents, order.loadedVersion());

        // 4. Publicar para projeções
        eventPublisher.publishAll(newEvents);
    }
}

// Projeção para Read Model
public class OrderProjection {
    private final OrderReadModelStore readStore;

    public void on(OrderPlaced event) {
        readStore.create(new OrderReadModel(
            event.orderId().value(),
            event.customerId().value(),
            "PENDING", "0.00", 0, event.occurredAt(), List.of()));
    }

    public void on(OrderConfirmed event) {
        readStore.updateStatus(event.orderId().value(), "CONFIRMED");
    }
}
```

**Go 1.26:**
```go
type ConfirmOrderHandler struct {
    eventStore     EventStore
    eventPublisher EventPublisher
}

func (h *ConfirmOrderHandler) Handle(cmd ConfirmOrderCommand) error { ... }

type OrderProjection struct {
    readStore OrderReadModelStore
}

func (p *OrderProjection) OnOrderPlaced(event OrderPlaced) error { ... }
func (p *OrderProjection) OnOrderConfirmed(event OrderConfirmed) error { ... }
```

**Critérios de aceite:**
- [ ] Write Side usa Event Sourcing (Event Store, reconstituição)
- [ ] Read Side usa projeções de eventos → Read Models
- [ ] Flow completo funcional: command → event store → projeção → query
- [ ] Eventual consistency entre Write e Read (documentar trade-off)
- [ ] Read Model atualizável (rebuild) re-processando todos os eventos
- [ ] Teste end-to-end: command → evento salvo → projeção atualizada → query retorna dado correto

---

### Desafio 5.6 — Modular Monolith: Integração dos BCs

Organize todos os Bounded Contexts do MerchantHub como um **Modular Monolith**.

**Requisitos:**
- Cada BC é um **módulo** com fronteira de compilação clara
- **Comunicação entre módulos**: apenas via interfaces públicas (Ports) e eventos
- **Nenhum módulo importa classes internas de outro**
- **In-process event bus** para comunicação entre módulos
- **Módulo Shared Kernel** com tipos mínimos compartilhados

**Estrutura do Modular Monolith:**

```
merchanthub/
├── shared-kernel/               # Shared Kernel (mínimo)
│   ├── Money.java
│   ├── DateRange.java
│   └── DomainEvent.java
│
├── catalog/                     # BC: Catalog (Clean Architecture)
│   ├── entity/
│   ├── usecase/
│   ├── adapter/
│   └── CatalogModule.java       # Ponto de entrada do módulo
│
├── sales/                       # BC: Sales (Hexagonal + CQRS + ES)
│   ├── domain/
│   ├── port/
│   ├── application/
│   ├── adapter/
│   └── SalesModule.java
│
├── inventory/                   # BC: Inventory (Hexagonal)
│   ├── domain/
│   ├── port/
│   ├── application/
│   ├── adapter/
│   └── InventoryModule.java
│
├── billing/                     # BC: Billing
│   └── BillingModule.java
│
├── shipping/                    # BC: Shipping
│   └── ShippingModule.java
│
├── identity/                    # BC: Identity (Generic)
│   └── IdentityModule.java
│
└── infrastructure/
    ├── EventBus.java            # In-process event bus
    └── ModuleRegistry.java      # Registro e inicialização
```

**Java 25:**
```java
// Módulo como unidade coesa com API pública explícita
public class SalesModule {
    private final PlaceOrderUseCase placeOrder;
    private final ConfirmOrderUseCase confirmOrder;
    private final GetOrderQueryHandler getOrder;

    public SalesModule(EventBus eventBus, CatalogModule catalog) {
        // Wiring interno — DI manual, sem framework
        var eventStore = new InMemoryEventStore();
        var orderRepo = new EventSourcedOrderRepository(eventStore);
        var catalogClient = new InProcessCatalogClient(catalog);
        var pricingService = new VolumePricingService();

        this.placeOrder = new PlaceOrderService(orderRepo, catalogClient, pricingService);
        this.confirmOrder = new ConfirmOrderService(orderRepo, eventBus);
        this.getOrder = new GetOrderQueryHandler(new InMemoryOrderReadModelStore());

        // Registrar handlers de eventos de outros módulos
        eventBus.subscribe(ProductPriceChangedEvent.class,
            new ProductPriceChangedSalesHandler(orderRepo));
    }

    // API pública do módulo
    public OrderId placeOrder(PlaceOrderCommand command) { return placeOrder.execute(command); }
    public void confirmOrder(ConfirmOrderCommand command) { confirmOrder.execute(command); }
    public OrderReadModel getOrder(String orderId) { return getOrder.handle(new GetOrderQuery(orderId)); }
}

// Event Bus in-process
public class EventBus implements EventPublisher {
    private final Map<Class<? extends DomainEvent>, List<EventHandler<?>>> handlers = new ConcurrentHashMap<>();

    public <E extends DomainEvent> void subscribe(Class<E> eventType, EventHandler<E> handler) {
        handlers.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>()).add(handler);
    }

    @Override
    @SuppressWarnings("unchecked")
    public void publish(DomainEvent event) {
        var eventHandlers = handlers.getOrDefault(event.getClass(), List.of());
        for (var handler : eventHandlers) {
            ((EventHandler<DomainEvent>) handler).handle(event);
        }
    }
}
```

**Go 1.26:**
```go
type SalesModule struct {
    placeOrder   PlaceOrderUseCase
    confirmOrder ConfirmOrderUseCase
    getOrder     *GetOrderQueryHandler
}

func NewSalesModule(eventBus *EventBus, catalog *CatalogModule) *SalesModule { ... }
func (m *SalesModule) PlaceOrder(cmd PlaceOrderCommand) (OrderId, error) { ... }

type EventBus struct {
    mu       sync.RWMutex
    handlers map[string][]func(DomainEvent) error
}

func (bus *EventBus) Subscribe(eventType string, handler func(DomainEvent) error) { ... }
func (bus *EventBus) Publish(event DomainEvent) error { ... }
```

**Critérios de aceite:**
- [ ] Cada BC é um módulo com ponto de entrada explícito (Module class)
- [ ] Módulos comunicam via interfaces públicas e eventos (nunca imports internos)
- [ ] Event Bus in-process para comunicação assíncrona entre módulos
- [ ] DI manual via construtores (nenhum framework de injeção)
- [ ] Shared Kernel como módulo separado com tipos mínimos
- [ ] Teste de isolamento: cada módulo inicializa e funciona independentemente
- [ ] Verificação arquitetural: nenhum import cruzado entre módulos (ArchUnit / go vet)
- [ ] Teste de integração: fluxo completo crossing 3+ módulos via eventos

---

## Entregáveis

| Artefato | Descrição |
|----------|-----------|
| Sales BC Hexagonal | Ports & Adapters completo com testes de isolamento |
| Catalog BC Clean Arch | 4 camadas com Use Cases e DTOs de I/O |
| CQRS Sales | Command/Query separados com Read Model projetado |
| Event Sourcing Order | Event Store, reconstituição, optimistic concurrency |
| CQRS + ES integrado | Flow completo command → event store → projeção → query |
| Modular Monolith | 6 módulos com Event Bus, isolamento e DI manual |
| Comparação arquitetural | Documento comparando Hexagonal vs Clean vs Layered |
| Testes | ≥ 85% cobertura, incluindo testes de arquitetura (imports) |

---

## O que você vai exercitar

| Arquitetura | O que pratica |
|------------|---------------|
| **Hexagonal** | Ports & Adapters, Dependency Inversion, isolamento do domínio |
| **Clean Architecture** | Camadas concêntricas, Use Cases, DTOs I/O, Dependency Rule |
| **CQRS** | Separação de modelos de leitura e escrita, projeções |
| **Event Sourcing** | Estado derivado de eventos, Event Store, reconstituição |
| **CQRS + ES** | Integração completa, eventual consistency |
| **Modular Monolith** | Módulos isolados, Event Bus, preparação para microservices |

---

## Próximo Nível

Quando completar todos os desafios deste nível, avance para:
→ [Level 6 — Capstone: MerchantHub Completo](06-capstone-merchanthub.md)

O capstone consolida **todos** os conceitos dos Levels 0-5 em uma implementação completa e integrada.
