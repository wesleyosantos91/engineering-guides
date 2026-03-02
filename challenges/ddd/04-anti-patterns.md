# Level 4 — Anti-Patterns & Refactoring

> **Objetivo:** Reconhecer, diagnosticar e refatorar os anti-patterns mais comuns em projetos
> que tentam adotar DDD — transformando código problemático em modelo de domínio rico,
> bem encapsulado e alinhado com os princípios dos níveis anteriores.

**Referência:** [04-anti-patterns.md](../../.docs/ddd/04-anti-patterns.md)

**Pré-requisito:** [Level 3 — Context Mapping](03-context-mapping.md) completo.

---

## Contexto do Domínio

Neste nível, você vai receber **código propositalmente problemático** do MerchantHub e
refatorá-lo aplicando os conceitos aprendidos nos Levels 0–3. Cada desafio apresenta
um anti-pattern com código "antes" e exige que você produza o código "depois" com
justificativa técnica.

> **Regra de ouro:** Para cada anti-pattern você deve produzir:
> 1. **Diagnóstico** — qual anti-pattern, onde, por que é problemático
> 2. **Refactoring** — código refatorado em Java 25 + Go 1.26
> 3. **Testes** — provando que o comportamento foi preservado e invariantes protegidas
> 4. **Documentação** — ADR (Architecture Decision Record) justificando a mudança

---

## Desafios

### Desafio 4.1 — Anti-Pattern: Anemic Domain Model → Rich Domain Model

**O problema (código "antes"):**

Modelo anêmico onde Entities são apenas data containers e toda a lógica de negócio
está em "Services" procedurais.

**Java 25 — Código Anêmico (RUIM):**
```java
// Entity anêmica — só getters/setters
public class Order {
    private String id;
    private String customerId;
    private List<OrderItem> items = new ArrayList<>();
    private String status;
    private BigDecimal total;

    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public List<OrderItem> getItems() { return items; }
    public void setItems(List<OrderItem> items) { this.items = items; }
    public BigDecimal getTotal() { return total; }
    public void setTotal(BigDecimal total) { this.total = total; }
}

// "Service" que contém TODA a lógica — Entity é mero DTO
public class OrderService {
    public void addItem(Order order, String productId, int qty, BigDecimal price) {
        var item = new OrderItem();
        item.setProductId(productId);
        item.setQuantity(qty);
        item.setUnitPrice(price);
        order.getItems().add(item);
        recalculateTotal(order);
    }

    public void confirmOrder(Order order) {
        if (order.getItems().isEmpty()) {
            throw new RuntimeException("Cannot confirm empty order");
        }
        if (!order.getStatus().equals("PENDING")) {
            throw new RuntimeException("Order must be PENDING to confirm");
        }
        order.setStatus("CONFIRMED");
    }

    private void recalculateTotal(Order order) {
        BigDecimal total = BigDecimal.ZERO;
        for (var item : order.getItems()) {
            total = total.add(item.getUnitPrice().multiply(BigDecimal.valueOf(item.getQuantity())));
        }
        order.setTotal(total);
    }
}
```

**Go 1.26 — Código Anêmico (RUIM):**
```go
type Order struct {
    ID         string
    CustomerID string
    Items      []OrderItem  // exportado — qualquer um modifica
    Status     string       // string mágica
    Total      float64      // float para dinheiro!
}

func AddItem(order *Order, productID string, qty int, price float64) {
    order.Items = append(order.Items, OrderItem{
        ProductID: productID, Quantity: qty, UnitPrice: price,
    })
    recalculateTotal(order)
}

func ConfirmOrder(order *Order) error {
    if len(order.Items) == 0 { return errors.New("cannot confirm empty order") }
    if order.Status != "PENDING" { return errors.New("order must be PENDING") }
    order.Status = "CONFIRMED"
    return nil
}
```

**Sua tarefa:**
Refatorar para Rich Domain Model usando o `Order` Aggregate do Level 2 como referência.

**Checklist do refactoring:**
- [ ] Mover lógica de `OrderService.addItem()` → `Order.addItem()` (comportamento na Entity)
- [ ] Mover lógica de `OrderService.confirmOrder()` → `Order.confirm()`
- [ ] Eliminar setters públicos — estado muda via métodos com intenção de negócio
- [ ] Substituir `String` status por enum `OrderStatus` (sealed em Java)
- [ ] Substituir `BigDecimal` / `float64` por VO `Money`
- [ ] Substituir `String` ids por VOs tipados (`OrderId`, `CustomerId`)
- [ ] Proteger invariantes no construtor (factory method)
- [ ] Testes: comportamento preservado, invariantes protegidas

---

### Desafio 4.2 — Anti-Pattern: God Aggregate → Aggregates com Fronteira Correta

**O problema (código "antes"):**

Um Aggregate monstruoso que contém tudo: pedido, itens, pagamentos, endereço de cobrança,
endereço de entrega, histórico de status, avaliações, cupons — violando a regra de fronteira
de consistência.

**Java 25 — God Aggregate (RUIM):**
```java
public class Order {
    private OrderId id;
    private Customer customer;                    // Objeto inteiro, não referência por ID
    private List<OrderItem> items;
    private List<Payment> payments;               // Deveria ser Aggregate separado
    private Address billingAddress;
    private Address shippingAddress;
    private List<StatusChange> statusHistory;     // Deveria ser Aggregate separado
    private ShippingTracking tracking;            // Deveria ser outro BC!
    private List<Review> reviews;                 // Deveria ser outro BC!
    private List<Coupon> appliedCoupons;          // Deveria ser Aggregate separado
    private LoyaltyPoints earnedPoints;           // Deveria ser outro BC!

    // 40+ métodos que manipulam tudo isso
    public void addPayment(Payment payment) { ... }
    public void updateTracking(String trackingCode) { ... }
    public void addReview(Review review) { ... }
    public void applyCoupon(Coupon coupon) { ... }
    public void addLoyaltyPoints(int points) { ... }
}
```

**Go 1.26 — God Aggregate (RUIM):**
```go
type Order struct {
    ID              OrderId
    Customer        Customer         // Objeto inteiro!
    Items           []OrderItem
    Payments        []Payment        // Aggregate separado?
    BillingAddress  Address
    ShippingAddress Address
    StatusHistory   []StatusChange   // Aggregate separado?
    Tracking        ShippingTracking // Outro BC!
    Reviews         []Review         // Outro BC!
    AppliedCoupons  []Coupon
    LoyaltyPoints   int
}
```

**Sua tarefa:**
Quebrar o God Aggregate em múltiplos Aggregates e BCs corretos.

**Refactoring esperado:**
1. **Order Aggregate** (Sales BC): `OrderId`, `customerId` (ref por ID), `items`, `shippingAddress`, `status`
2. **Payment Aggregate** (Billing BC): `PaymentId`, `orderId` (ref por ID), `amount`, `method`, `status`
3. **Shipment Aggregate** (Shipping BC): `ShipmentId`, `orderId` (ref por ID), `tracking`, `status`
4. **Review Aggregate** (Catalog BC): `ReviewId`, `productId`, `customerId`, `rating`, `comment`
5. **Coupon** e **LoyaltyPoints** → seus respectivos BCs
6. `Customer` substituído por `customerId` (referência por ID)

**Checklist do refactoring:**
- [ ] God Aggregate quebrado em 4+ Aggregates menores
- [ ] Cada Aggregate em seu Bounded Context correto
- [ ] Referências entre Aggregates por ID, não por objeto
- [ ] Cada Aggregate com suas próprias invariantes delimitadas
- [ ] Comunicação entre Aggregates via Domain Events
- [ ] Testes: cada Aggregate testável independentemente
- [ ] Documento comparando antes/depois (tamanho, responsabilidades, testabilidade)

---

### Desafio 4.3 — Anti-Pattern: Primitive Obsession → Value Objects

**O problema (código "antes"):**

Primitivos usados em toda a base de código, causando duplicação de validação,
bugs de tipo e falta de expressividade.

**Java 25 — Primitive Obsession (RUIM):**
```java
public class OrderService {
    // Todos os parâmetros são String ou primitivos — nenhuma proteção de tipo
    public Order createOrder(String customerId, String productId,
            int quantity, BigDecimal price, String currency,
            String street, String city, String state,
            String zipCode, String country, String email) {

        // Validação dispersa e duplicada
        if (customerId == null || customerId.isBlank())
            throw new IllegalArgumentException("customerId required");
        if (quantity <= 0)
            throw new IllegalArgumentException("quantity must be positive");
        if (price.compareTo(BigDecimal.ZERO) <= 0)
            throw new IllegalArgumentException("price must be positive");
        if (!email.matches("^[\\w.-]+@[\\w.-]+\\.[a-zA-Z]{2,}$"))
            throw new IllegalArgumentException("invalid email");
        if (!zipCode.matches("\\d{5}-\\d{3}"))
            throw new IllegalArgumentException("invalid zipCode");

        // Fácil trocar parâmetros: createOrder(email, customerId, ...)
        // Compilador não ajuda — tudo é String!
    }
}
```

**Go 1.26 — Primitive Obsession (RUIM):**
```go
func CreateOrder(customerId, productId string,
    quantity int, price float64, currency string,
    street, city, state, zipCode, country, email string) (*Order, error) {
    // mesma validação dispersa...
}
```

**Sua tarefa:**
Substituir todos os primitivos por Value Objects do Level 0.

**Refactoring esperado:**
```java
// DEPOIS — VOs tipados protegem contra bugs de tipo
public Order createOrder(CustomerId customerId, ProductId productId,
        Quantity quantity, Money price, Address address, EmailAddress email) {
    // Sem validação aqui — VOs já são válidos por construção
    // Impossível trocar parâmetros — tipos diferentes
}
```

**Checklist do refactoring:**
- [ ] Todo `String` de identidade → VO tipado (`CustomerId`, `ProductId`, `SKU`, etc.)
- [ ] Todo `BigDecimal`/`float64` monetário → `Money`
- [ ] Todo `int` semântico → VO (`Quantity`, `Percentage`)
- [ ] Todo `String` de email → `EmailAddress`
- [ ] Todo conjunto de strings de endereço → `Address` composto
- [ ] Validação centralizada no construtor do VO (nunca mais dispersa)
- [ ] Testes: compilação já impede troca de parâmetros, VOs rejeitam valores inválidos

---

### Desafio 4.4 — Anti-Pattern: Shared Database → Database per BC

**O problema (código "antes"):**

Todos os Bounded Contexts acessam a mesma tabela de banco de dados, criando acoplamento
estrutural — mudanças no schema quebram múltiplos contextos.

**Cenário problemático:**
```
┌────────────┐  ┌────────────┐  ┌────────────┐
│  Catalog   │  │   Sales    │  │  Inventory  │
│  Service   │  │  Service   │  │  Service    │
└─────┬──────┘  └─────┬──────┘  └─────┬───────┘
      │               │               │
      └───────────────┼───────────────┘
                      │
              ┌───────▼───────┐
              │  products     │  ← TABELA COMPARTILHADA!
              │  ----------   │
              │  id           │
              │  name         │
              │  description  │
              │  price        │
              │  stock_qty    │  ← Inventory!
              │  category_id  │  ← Catalog!
              │  is_active    │  ← Sales!
              │  weight       │  ← Shipping!
              │  tax_rate     │  ← Billing!
              └───────────────┘
```

**Sua tarefa:**
Separar em schemas/tabelas por BC com comunicação via eventos.

**Refactoring esperado:**
```
Catalog BC:  catalog_products (id, name, description, price, category_id, status)
Sales BC:    sale_products (product_id, current_price, is_available)
Inventory BC: inventory_items (sku, stock_qty, warehouse_id, reorder_point)
Shipping BC:  shippable_items (product_id, weight, dimensions)
Billing BC:   billable_items (product_id, tax_rate, tax_category)
```

**Java 25 — Cada BC com seu Repository e modelo:**
```java
// Catalog BC
public interface CatalogProductRepository {
    void save(CatalogProduct product);
    Optional<CatalogProduct> findById(ProductId id);
    // Opera sobre catalog_products — seu schema exclusivo
}

// Inventory BC
public interface InventoryItemRepository {
    void save(InventoryItem item);
    Optional<InventoryItem> findBySku(SKU sku);
    // Opera sobre inventory_items — seu schema exclusivo
}

// Sincronização via eventos, não via tabela compartilhada
// Quando Catalog atualiza preço → ProductPriceChanged evento → Sales atualiza sua cópia local
```

**Go 1.26:**
```go
type CatalogProductRepository interface {
    Save(product *CatalogProduct) error
    FindByID(id ProductId) (*CatalogProduct, error)
}

type InventoryItemRepository interface {
    Save(item *InventoryItem) error
    FindBySKU(sku SKU) (*InventoryItem, error)
}
```

**Checklist do refactoring:**
- [ ] Tabela monolítica separada em schemas por BC
- [ ] Cada BC tem seu próprio modelo de dados (colunas diferentes)
- [ ] Sincronização entre BCs via Domain Events (não consulta direta)
- [ ] Cada Repository acessa apenas seu schema
- [ ] Diagrama antes/depois mostrando desacoplamento
- [ ] ADR documentando trade-offs (eventual consistency, duplicação controlada)
- [ ] Testes: cada BC funciona independentemente

---

### Desafio 4.5 — Anti-Pattern: Smart UI / Logic in Controller → Application Layer

**O problema (código "antes"):**

Lógica de negócio misturada em controladores/handlers HTTP.

**Java 25 — Lógica no Controller (RUIM):**
```java
@RestController
public class OrderController {
    @Autowired private JdbcTemplate jdbc;

    @PostMapping("/orders")
    public ResponseEntity<?> createOrder(@RequestBody Map<String, Object> body) {
        // Parsing manual
        String customerId = (String) body.get("customerId");
        List<Map<String, Object>> items = (List) body.get("items");

        // Validação de negócio no controller!
        if (items.isEmpty())
            return ResponseEntity.badRequest().body("Need at least one item");

        // Lógica de negócio no controller!
        BigDecimal total = BigDecimal.ZERO;
        for (var item : items) {
            int qty = (int) item.get("quantity");
            var price = jdbc.queryForObject(
                "SELECT price FROM products WHERE id = ?",
                BigDecimal.class, item.get("productId"));
            total = total.add(price.multiply(BigDecimal.valueOf(qty)));
        }

        // Desconto no controller!
        if (total.compareTo(new BigDecimal("500")) > 0) {
            total = total.multiply(new BigDecimal("0.95")); // 5% discount
        }

        // Persistência direta no controller!
        jdbc.update("INSERT INTO orders...", customerId, total);
        return ResponseEntity.ok().build();
    }
}
```

**Sua tarefa:**
Extrair para camadas corretas: Controller → Application Service → Domain → Infrastructure.

**Refactoring esperado:**
1. **Controller** — só recebe request, chama use case, retorna response
2. **Application Service** (`PlaceOrderUseCase`) — orquestra sem regras de negócio
3. **Domain** (`Order`, `PricingService`) — contém todas as regras de negócio
4. **Infrastructure** (`OrderRepository`, `ProductRepository`) — persistência

**Checklist do refactoring:**
- [ ] Controller sem lógica de negócio — apenas mapeia request → command → response
- [ ] Application Service orquestra — chama Repository e Domain Service
- [ ] Regras de desconto no `PricingService` (Domain Service)
- [ ] Criação de Order no `Order.create()` (Factory method no Aggregate)
- [ ] Persistência via Repository (sem JDBC/SQL no controller)
- [ ] Cada camada testável isoladamente (domain sem infra, app com mocks)
- [ ] Testes: unit (domain), integration (app + infra), e2e (controller)

---

### Desafio 4.6 — Consolidação: Refactoring Recipe Completa

Aplique uma **Refactoring Recipe** completa a uma funcionalidade do MerchantHub.

**Cenário:** A funcionalidade "Aplicar Cupom a Pedido" está implementada com múltiplos anti-patterns.

**Código problemático contém todos os anti-patterns anteriores:**
- Modelo anêmico (`Coupon` com getters/setters)
- Primitive obsession (código do cupom é `String`, desconto é `double`)
- Lógica no service (validação de cupom no `CouponService`)
- Shared database (cupom e pedido na mesma tabela de "order_coupons")
- God aggregate (`Order` tem lista de `Coupon` internamente)

**Recipe de Refactoring (4 etapas):**

1. **Strangler Fig**: Criar novo modelo ao lado do antigo
   - Criar `CouponCode` VO, `Discount` VO, `Coupon` Aggregate
   - Nova implementação coexiste com a antiga

2. **Branch by Abstraction**: Interface intermediária
   - Criar `CouponValidationService` interface
   - Implementação antiga e nova sob mesma interface
   - Feature flag para alternar

3. **Parallel Run**: Executar ambos e comparar
   - Ambas as implementações executam em paralelo
   - Log de divergências para validar
   - Nova implementação deve produzir mesmos resultados

4. **Cut Over**: Remover implementação antiga
   - Remover código antigo
   - Remover feature flag
   - Remover tabela compartilhada

**Checklist do refactoring:**
- [ ] Refactoring em 4 etapas (Strangler → Abstraction → Parallel → Cut Over)
- [ ] Cada etapa é um commit separado (código sempre funcional)
- [ ] `CouponCode` e `Discount` como Value Objects
- [ ] `Coupon` como Aggregate separado de `Order`
- [ ] Validação de cupom no `Coupon` Aggregate (não em service externo)
- [ ] Comunicação Order ↔ Coupon via eventos ou Application Service (nunca acoplamento direto)
- [ ] ADR para cada decisão de refactoring
- [ ] Testes em cada etapa comprovando que comportamento é preservado

---

## Entregáveis

| Artefato | Descrição |
|----------|-----------|
| Anemic → Rich | `Order` refatorado com comportamento rico (antes/depois) |
| God → Small Aggregates | 4+ Aggregates extraídos com fronteiras corretas |
| Primitives → VOs | Todos os primitivos substituídos por Value Objects |
| Shared DB → DB per BC | Esquemas separados por BC com sincronização por eventos |
| Smart UI → Layers | Lógica extraída para camadas corretas (Domain, App, Infra) |
| Refactoring Recipe | Cupom refatorado em 4 etapas incrementais |
| ADRs | 1 ADR por refactoring documentando decisão e trade-offs |
| Testes | ≥ 90% cobertura — antes/depois, invariantes, preservação de comportamento |

---

## O que você vai exercitar

| Anti-Pattern | Habilidade desenvolvida |
|-------------|------------------------|
| **Anemic Domain Model** | Mover lógica para Entities, encapsulamento, Tell Don't Ask |
| **God Aggregate** | Identificar fronteiras de consistência, quebrar acoplamento |
| **Primitive Obsession** | Criar VOs expressivos, type safety, validação no construtor |
| **Shared Database** | Separar schemas, eventual consistency, comunicação por eventos |
| **Smart UI** | Layered architecture, separação de responsabilidades |
| **Refactoring Recipe** | Mudança incremental segura, Strangler Fig, feature flags |

---

## Próximo Nível

Quando completar todos os desafios deste nível, avance para:
→ [Level 5 — Architecture & Integration](05-architecture-integration.md)

As refatorações deste nível preparam o código para ser organizado em arquiteturas limpas (Hexagonal, Clean, CQRS, Event Sourcing) no próximo nível.
