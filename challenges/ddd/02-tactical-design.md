# Level 2 — Tactical Design (Building Blocks)

> **Objetivo:** Implementar todos os Building Blocks do DDD Tático — Entities ricas, Value Objects,
> Aggregates com invariantes, Domain Events, Repositories, Domain Services e Factories —
> dentro dos Bounded Contexts do MerchantHub.

**Referência:** [02-tactical-design.md](../../.docs/ddd/02-tactical-design.md)

**Pré-requisito:** [Level 1 — Strategic Design](01-strategic-design.md) completo.

---

## Contexto do Domínio

Neste nível, você vai **modelar o coração do domínio**. Enquanto o Level 1 definiu **onde** modelar (Bounded Contexts), o Level 2 define **como** modelar dentro de cada contexto. Você criará Entities ricas (não anêmicas), Aggregates com fronteiras de consistência transacional, Domain Events como fatos de negócio e Repositories que persistem Aggregates inteiros.

Foco nos dois Core Domains: **Catalog** e **Sales**.

---

## Desafios

### Desafio 2.1 — Entity Rica: `CatalogProduct` (Combate ao Anemic Domain Model)

Implemente a Entity `CatalogProduct` com **comportamento de domínio rico** — não apenas getters/setters.

**Requisitos:**
- `CatalogProduct` é uma Entity com identidade (`ProductId`)
- Campos: `name`, `description`, `images` (lista), `basePrice` (Money), `categoryId`, `status` (ProductStatus)
- Comportamento de domínio:
  - `activate()` — muda status para ACTIVE (só se DRAFT ou INACTIVE)
  - `deactivate()` — muda status para INACTIVE (só se ACTIVE)
  - `discontinue()` — muda status para DISCONTINUED (irreversível, qualquer status exceto DISCONTINUED)
  - `updatePrice(newPrice: Money)` — atualiza preço com validação (apenas se ACTIVE ou DRAFT)
  - `addImage(image: Image)` — max 10 imagens
  - `removeImage(imageId: ImageId)` — não pode ficar sem imagens se ACTIVE
- **Invariantes protegidas:**
  - Produto ACTIVE deve ter pelo menos 1 imagem
  - Preço deve ser > 0 para produto ACTIVE
  - Transições de status validadas (máquina de estados)
  - Produto DISCONTINUED não pode ser modificado
- **Igualdade por identidade** (ID), não por atributos
- Produto nunca existe em estado inválido (validação no construtor)

**Java 25:**
```java
public final class CatalogProduct {
    private final ProductId id;
    private String name;
    private String description;
    private final List<Image> images;
    private Money basePrice;
    private final CategoryId categoryId;
    private ProductStatus status;
    private final List<DomainEvent> domainEvents;

    // Factory method (não construtor público)
    public static CatalogProduct create(ProductId id, String name,
            String description, Money basePrice, CategoryId categoryId) {
        // Validações no construtor — nunca inválido
        var product = new CatalogProduct(id, name, description,
            basePrice, categoryId, ProductStatus.DRAFT);
        product.domainEvents.add(new ProductCreated(id, name, basePrice));
        return product;
    }

    public void activate() {
        require(!images.isEmpty(), "Product must have at least one image to activate");
        require(basePrice.isPositive(), "Price must be positive to activate");
        requireTransition(ProductStatus.ACTIVE);
        this.status = ProductStatus.ACTIVE;
        domainEvents.add(new ProductActivated(id));
    }

    public void updatePrice(Money newPrice) {
        require(status != ProductStatus.DISCONTINUED, "Cannot modify discontinued product");
        require(newPrice.isPositive(), "Price must be positive");
        var oldPrice = this.basePrice;
        this.basePrice = newPrice;
        domainEvents.add(new ProductPriceChanged(id, oldPrice, newPrice));
    }

    @Override
    public boolean equals(Object o) {
        return o instanceof CatalogProduct other && this.id.equals(other.id);
    }
}
```

**Go 1.26:**
```go
type CatalogProduct struct {
    id           ProductId
    name         string
    description  string
    images       []Image
    basePrice    Money
    categoryId   CategoryId
    status       ProductStatus
    domainEvents []DomainEvent
}

func NewCatalogProduct(id ProductId, name, description string,
    basePrice Money, categoryId CategoryId) (*CatalogProduct, error) { ... }

func (p *CatalogProduct) Activate() error {
    if len(p.images) == 0 {
        return &BusinessRuleViolation{Rule: "Product must have at least one image"}
    }
    // ...
}

func (p *CatalogProduct) UpdatePrice(newPrice Money) error { ... }
```

**Critérios de aceite:**
- [ ] Entity com identidade (`ProductId`) e igualdade por ID
- [ ] Comportamento rico — todos os métodos de domínio implementados
- [ ] Invariantes protegidas: imagens para ACTIVE, preço positivo, transições válidas
- [ ] Produto nunca existe em estado inválido
- [ ] Factory method (não construtor público) para criação controlada
- [ ] Domain Events registrados a cada ação significativa
- [ ] Sem setters públicos — estado muda via métodos com intenção de negócio
- [ ] Testes: criação, ativação, transições válidas/inválidas, invariantes

---

### Desafio 2.2 — Aggregate: `Order` com Root + Entidades Internas

Implemente o Aggregate `Order` com `OrderItem` como entidade interna.

**Requisitos:**
- `Order` é o **Aggregate Root** com identidade global (`OrderId`)
- `OrderItem` é uma **entidade interna** com identidade local (`OrderItemId`)
- Campos Order: `orderId`, `customerId` (ref por ID), `items` (lista), `shippingAddress`, `status`, `version` (optimistic locking)
- Campos OrderItem: `orderItemId`, `productId` (ref por ID), `quantity`, `unitPrice`, `subtotal()`

**Regras de Aggregate:**
- **Root é o ponto de entrada**: OrderItem só é acessado via Order
- **Consistência transacional**: invariantes dentro do Aggregate garantidas em 1 transação
- **Referência por ID**: Order referencia Customer por `customerId`, não por objeto
- **Um Aggregate por transação**: modificar Order não pode modificar Inventory

**Invariantes do Order Aggregate:**
- Pedido deve ter pelo menos 1 item
- Itens não podem ser duplicados (mesmo productId)
- Total não pode exceder limite máximo (ex: R$ 100.000)
- Status governa quais operações são permitidas
- Quantidade de cada item > 0

**Comportamento:**
- `addItem(productId, quantity, unitPrice)` — adiciona ou incrementa
- `removeItem(productId)` — remove (não pode ficar sem itens)
- `changeItemQuantity(productId, newQuantity)` — altera quantidade
- `confirm()` — muda para CONFIRMED (publica `OrderConfirmed`)
- `cancel(reason)` — muda para CANCELLED (publica `OrderCancelled`)
- `totalAmount()` — soma dos subtotais dos itens

**Java 25:**
```java
public final class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private Address shippingAddress;
    private OrderStatus status;
    private int version;
    private final List<DomainEvent> domainEvents;

    public static Order create(OrderId id, CustomerId customerId,
            Address shippingAddress, List<OrderItemData> itemsData) {
        require(!itemsData.isEmpty(), "Order must have at least one item");
        // ...
    }

    public void addItem(ProductId productId, Quantity quantity, Money unitPrice) {
        requireModifiable();
        var existing = findItem(productId);
        if (existing.isPresent()) {
            existing.get().increaseQuantity(quantity);
        } else {
            items.add(OrderItem.create(productId, quantity, unitPrice));
        }
        validateTotalLimit();
    }

    public void confirm() {
        requireTransition(OrderStatus.CONFIRMED);
        require(!items.isEmpty(), "Cannot confirm empty order");
        this.status = OrderStatus.CONFIRMED;
        domainEvents.add(new OrderConfirmed(id, customerId, totalAmount(), shippingAddress));
    }

    public Money totalAmount() {
        return items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// Entidade interna — identidade local
public final class OrderItem {
    private final OrderItemId id;
    private final ProductId productId;
    private Quantity quantity;
    private final Money unitPrice;

    Money subtotal() { return unitPrice.multiply(quantity.value()); }
    void increaseQuantity(Quantity additional) { ... }
}
```

**Go 1.26:**
```go
type Order struct {
    id              OrderId
    customerId      CustomerId
    items           []*OrderItem
    shippingAddress Address
    status          OrderStatus
    version         int
    domainEvents    []DomainEvent
}

func CreateOrder(id OrderId, customerId CustomerId,
    address Address, itemsData []OrderItemData) (*Order, error) { ... }

func (o *Order) AddItem(productId ProductId, qty Quantity, unitPrice Money) error { ... }
func (o *Order) Confirm() error { ... }
func (o *Order) TotalAmount() Money { ... }
```

**Critérios de aceite:**
- [ ] Aggregate Root (`Order`) com identidade global
- [ ] Entidade interna (`OrderItem`) com identidade local
- [ ] Invariantes protegidas dentro da fronteira do Aggregate
- [ ] Referência entre Aggregates por ID (`customerId`, `productId`)
- [ ] Root é ponto de entrada — OrderItem não acessível diretamente de fora
- [ ] Domain Events publicados a cada operação significativa
- [ ] Optimistic locking via `version`
- [ ] Testes: criação, adição/remoção de itens, confirm/cancel, invariantes, transições

---

### Desafio 2.3 — Domain Events: Definição, Publicação e Handling

Implemente o sistema de Domain Events para comunicar fatos entre Aggregates.

**Requisitos:**
- Definir **pelo menos 8 Domain Events** para os contextos Catalog e Sales:
  - Catalog: `ProductCreated`, `ProductActivated`, `ProductPriceChanged`, `ProductDiscontinued`
  - Sales: `OrderPlaced`, `OrderConfirmed`, `OrderCancelled`, `OrderItemAdded`
- Cada evento deve conter:
  - `eventId` (UUID único)
  - `occurredAt` (timestamp)
  - Dados relevantes do fato (autônomo — consumidor não precisa consultar produtor)
- Implementar `EventPublisher` (interface no domínio)
- Implementar `InMemoryEventPublisher` (infra) com registro de handlers
- Implementar **handlers** que reagem a eventos:
  - `ProductPriceChanged` → log/notificação
  - `OrderConfirmed` → handler que simula reserva de estoque

**Java 25:**
```java
// Domain Event como sealed interface + records
public sealed interface DomainEvent permits
        ProductCreated, ProductActivated, ProductPriceChanged,
        ProductDiscontinued, OrderPlaced, OrderConfirmed,
        OrderCancelled, OrderItemAdded {
    UUID eventId();
    Instant occurredAt();
}

public record OrderConfirmed(
    UUID eventId,
    OrderId orderId,
    CustomerId customerId,
    Money totalAmount,
    Address shippingAddress,
    Instant occurredAt
) implements DomainEvent {}

// Publisher interface (domínio)
public interface EventPublisher {
    void publish(DomainEvent event);
    void publishAll(List<DomainEvent> events);
}

// Handler (domínio ou application)
public interface EventHandler<E extends DomainEvent> {
    void handle(E event);
}
```

**Go 1.26:**
```go
type DomainEvent interface {
    EventID() string
    OccurredAt() time.Time
    EventType() string
}

type OrderConfirmed struct {
    eventId         string
    OrderId         OrderId
    CustomerId      CustomerId
    TotalAmount     Money
    ShippingAddress Address
    occurredAt      time.Time
}

func (e OrderConfirmed) EventID() string        { return e.eventId }
func (e OrderConfirmed) OccurredAt() time.Time   { return e.occurredAt }
func (e OrderConfirmed) EventType() string       { return "OrderConfirmed" }

type EventPublisher interface {
    Publish(event DomainEvent) error
    PublishAll(events []DomainEvent) error
}
```

**Critérios de aceite:**
- [ ] 8+ Domain Events definidos com naming correto (fato no passado, linguagem ubíqua)
- [ ] Cada evento com eventId, occurredAt e dados autônomos
- [ ] Eventos são imutáveis (records em Java, structs em Go)
- [ ] `EventPublisher` interface no domínio, implementação na infra
- [ ] Handlers que reagem a eventos implementados
- [ ] Aggregates publicam eventos via método interno (collect, publish no save)
- [ ] Testes: criação de eventos, publicação, handling, idempotência

---

### Desafio 2.4 — Repository: Persistência de Aggregates

Implemente Repositories para os Aggregates `CatalogProduct` e `Order`.

**Requisitos:**
- **Interface no domínio, implementação na infra** (Dependency Inversion)
- Um Repository por Aggregate Root (nunca para entidades internas)
- Repository persiste e recupera **Aggregate inteiro** (root + internos)
- Após salvar, Domain Events pendentes são publicados e limpos
- Operações: `save`, `findById`, `findByXxx` (critérios de domínio), `nextId`
- Implementação `InMemory` com thread-safety

**Java 25:**
```java
// Interface no domínio
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomer(CustomerId customerId);
    List<Order> findByStatus(OrderStatus status);
    OrderId nextId();
}

// Implementação na infra
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<OrderId, Order> store = new ConcurrentHashMap<>();
    private final EventPublisher eventPublisher;

    @Override
    public void save(Order order) {
        store.put(order.id(), order);
        eventPublisher.publishAll(order.domainEvents());
        order.clearDomainEvents();
    }
}

public interface CatalogProductRepository {
    void save(CatalogProduct product);
    Optional<CatalogProduct> findById(ProductId id);
    List<CatalogProduct> findByCategory(CategoryId categoryId);
    List<CatalogProduct> findByStatus(ProductStatus status);
    ProductId nextId();
}
```

**Go 1.26:**
```go
// Interface no domínio
type OrderRepository interface {
    Save(order *Order) error
    FindByID(id OrderId) (*Order, error)
    FindByCustomer(customerId CustomerId) ([]*Order, error)
    FindByStatus(status OrderStatus) ([]*Order, error)
    NextID() OrderId
}

// Implementação na infra
type InMemoryOrderRepository struct {
    mu             sync.RWMutex
    store          map[OrderId]*Order
    eventPublisher EventPublisher
}

func (r *InMemoryOrderRepository) Save(order *Order) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.store[order.ID()] = order
    r.eventPublisher.PublishAll(order.DomainEvents())
    order.ClearDomainEvents()
    return nil
}
```

**Critérios de aceite:**
- [ ] Interface de Repository no domínio, implementação na infra
- [ ] Um Repository por Aggregate Root (`OrderRepository`, `CatalogProductRepository`)
- [ ] Sem `OrderItemRepository` — OrderItem é salvo junto com Order
- [ ] Aggregate inteiro persistido e recuperado (root + entidades internas)
- [ ] Domain Events publicados e limpos após save
- [ ] Thread-safety na implementação InMemory
- [ ] `nextId()` para geração de identidade
- [ ] Testes: CRUD, queries por critério, publicação de eventos no save

---

### Desafio 2.5 — Domain Service: Lógica Cross-Aggregate

Implemente Domain Services para lógica que não pertence naturalmente a nenhum Aggregate.

**Requisitos:**
- `PricingService` — calcula preço final considerando regras cross-entity:
  - Preço base do produto
  - Desconto por volume (> 5 itens = 5%, > 10 itens = 10%)
  - Desconto por fidelidade (cliente VIP = 3% adicional)
  - Resultado: preço final como `Money`
- `OrderPlacementService` — orquestra a criação de um pedido validando regras cross-aggregate:
  - Verifica se todos os produtos existem no catálogo
  - Verifica se todos os produtos estão ACTIVE
  - Obtém preço de cada produto
  - Cria o Order Aggregate

**Diferença importante:**
- `PricingService` é um **Domain Service** (contém regra de negócio: política de desconto)
- `PlaceOrderUseCase` seria um **Application Service** (apenas orquestra)
- No desafio, implemente ambos para entender a diferença

**Java 25:**
```java
// Domain Service — regra de negócio
public interface PricingService {
    Money calculateFinalPrice(Money basePrice, Quantity quantity, CustomerType customerType);
}

public class VolumePricingService implements PricingService {
    @Override
    public Money calculateFinalPrice(Money basePrice, Quantity quantity,
            CustomerType customerType) {
        var subtotal = basePrice.multiply(quantity.value());
        var volumeDiscount = determineVolumeDiscount(quantity);
        var loyaltyDiscount = determineLoyaltyDiscount(customerType);
        return subtotal.subtract(subtotal.percentage(volumeDiscount.add(loyaltyDiscount)));
    }
}

// Application Service — orquestração (não é Domain Service)
public class PlaceOrderUseCase {
    private final CatalogProductRepository catalogRepo;
    private final OrderRepository orderRepo;
    private final PricingService pricingService;

    public OrderId execute(PlaceOrderCommand command) {
        // 1. Buscar produtos e verificar status (orquestração)
        // 2. Calcular preço via PricingService (delega ao domínio)
        // 3. Criar Order Aggregate (domínio decide)
        // 4. Salvar (orquestração)
    }
}
```

**Go 1.26:**
```go
type PricingService interface {
    CalculateFinalPrice(basePrice Money, quantity Quantity,
        customerType CustomerType) Money
}

type VolumePricingService struct{}

func (s *VolumePricingService) CalculateFinalPrice(basePrice Money,
    quantity Quantity, customerType CustomerType) Money { ... }
```

**Critérios de aceite:**
- [ ] `PricingService` stateless com regra de desconto por volume + fidelidade
- [ ] Interface no domínio, implementação pode estar no domínio (é regra de negócio)
- [ ] `PlaceOrderUseCase` como Application Service (orquestração, NÃO regra de negócio)
- [ ] Diferença clara entre Domain Service e Application Service documentada
- [ ] Domain Service não mantém estado entre chamadas
- [ ] Testes unitários para PricingService sem infraestrutura
- [ ] Testes para PlaceOrderUseCase com mocks dos Repositories

---

### Desafio 2.6 — Factory: Criação Complexa de Aggregates

Implemente Factories para criação de Aggregates com lógica complexa.

**Requisitos:**
- **Factory Method no Aggregate Root** — para criações simples:
  - `Order.create(...)` — já implementado no Desafio 2.2
  - `CatalogProduct.create(...)` — já implementado no Desafio 2.1
- **Factory Class separada** — para criações que envolvem regras externas:
  - `OrderFactory` — cria Order validando contra catálogo e pricing:
    1. Busca preço atual de cada produto no CatalogProductRepository
    2. Calcula preço final via PricingService
    3. Valida que todos os produtos estão ACTIVE
    4. Cria o Order Aggregate com preços corretos
  - Separar criação de domínio (novo) de reconstituição (do banco)

**Java 25:**
```java
public class OrderFactory {
    private final CatalogProductRepository catalogRepo;
    private final PricingService pricingService;

    public Order createOrder(CustomerId customerId, Address shippingAddress,
            List<OrderItemRequest> itemRequests, CustomerType customerType) {

        var items = itemRequests.stream().map(request -> {
            var product = catalogRepo.findById(request.productId())
                .orElseThrow(() -> new EntityNotFound("CatalogProduct", request.productId()));

            require(product.isActive(), "Product " + request.productId() + " is not active");

            var finalPrice = pricingService.calculateFinalPrice(
                product.basePrice(), request.quantity(), customerType);

            return OrderItem.create(request.productId(), request.quantity(), finalPrice);
        }).toList();

        return Order.create(OrderId.generate(), customerId, shippingAddress, items);
    }
}
```

**Go 1.26:**
```go
type OrderFactory struct {
    catalogRepo    CatalogProductRepository
    pricingService PricingService
}

func (f *OrderFactory) CreateOrder(customerId CustomerId, address Address,
    items []OrderItemRequest, customerType CustomerType) (*Order, error) { ... }
```

**Critérios de aceite:**
- [ ] Factory Method no Aggregate Root para criação simples
- [ ] Factory Class para criação complexa com dependências externas
- [ ] Factory busca dados do catálogo e calcula preços antes de criar Order
- [ ] Validação de que produtos estão ACTIVE antes de criar pedido
- [ ] Separação clara entre criação (Factory) e reconstituição (Repository)
- [ ] Testes com mocks para validar lógica de criação

---

## Entregáveis

| Artefato | Descrição |
|----------|-----------|
| `CatalogProduct` | Entity rica com comportamento de domínio e invariantes protegidas |
| `Order` + `OrderItem` | Aggregate com root + entidade interna + regras transacionais |
| 8 Domain Events | Eventos como fatos imutáveis de negócio |
| `EventPublisher` + handlers | Sistema de publicação e reação a eventos |
| `OrderRepository` + `CatalogProductRepository` | Interfaces no domínio + InMemory na infra |
| `PricingService` | Domain Service stateless com regra de desconto |
| `PlaceOrderUseCase` | Application Service que orquestra (não contém regras) |
| `OrderFactory` | Factory para criação complexa com validação cross-aggregate |
| Testes | ≥ 85% cobertura em ambas as linguagens |

---

## O que você vai exercitar

| Building Block | O que pratica |
|---------------|---------------|
| **Entity** | Comportamento rico, identidade, encapsulamento, igualdade por ID |
| **Value Object** | Reutilizar VOs do Level 0 como componentes de Entities |
| **Aggregate** | Fronteira de consistência, root como ponto de entrada, referência por ID |
| **Domain Event** | Naming no passado, dados autônomos, publicação no Aggregate |
| **Repository** | Interface no domínio, persistência de Aggregate inteiro, DIP |
| **Domain Service** | Lógica cross-aggregate stateless vs Application Service (orquestração) |
| **Factory** | Criação complexa com validação, separação de criação vs reconstituição |

---

## Próximo Nível

Quando completar todos os desafios deste nível, avance para:
→ [Level 3 — Context Mapping](03-context-mapping.md)

Os Aggregates, Events e Repositories criados aqui serão conectados entre Bounded Contexts usando padrões de Context Mapping.
