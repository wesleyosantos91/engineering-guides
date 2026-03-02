# Level 3 — Context Mapping (Integração entre Bounded Contexts)

> **Objetivo:** Implementar os padrões de Context Mapping do DDD para integrar os 6 Bounded
> Contexts do MerchantHub, definindo relações, direção de dependência e estratégias de tradução
> entre modelos de domínio distintos.

**Referência:** [03-context-mapping.md](../../.docs/ddd/03-context-mapping.md)

**Pré-requisito:** [Level 2 — Tactical Design](02-tactical-design.md) completo.

---

## Contexto do Domínio

Agora que cada BC tem seus Aggregates, Entities e VOs modelados, é hora de **conectá-los sem
acoplá-los**. O Context Map do MerchantHub define como cada par de contextos se relaciona:

```
┌─────────────┐  Partnership   ┌─────────────┐
│   Catalog    │◄─────────────►│  Inventory   │
│   (Core)     │               │ (Supporting) │
└──────┬───────┘               └──────────────┘
       │ OHS/PL
       ▼
┌─────────────┐  Customer-     ┌─────────────┐
│    Sales     │  Supplier      │   Billing    │
│   (Core)     │──────────────►│ (Supporting) │
└──────┬───────┘               └──────────────┘
       │ Customer-Supplier
       ▼
┌─────────────┐                ┌─────────────┐
│  Shipping   │                │  Identity    │
│ (Supporting) │◄─── ACL ──────│  (Generic)   │
└─────────────┘                └──────────────┘
```

---

## Desafios

### Desafio 3.1 — Context Map: Mapeamento Completo das Relações

Crie o **Context Map** do MerchantHub documentando todas as relações entre BCs.

**Requisitos:**
- Definir **todas as relações** entre os 6 Bounded Contexts
- Para cada relação, especificar:
  - Tipo do padrão (Partnership, Customer-Supplier, Conformist, ACL, OHS/PL, Shared Kernel, Separate Ways)
  - Direção (Upstream → Downstream, ou bidirecional)
  - Motivo da escolha
  - Dados que cruzam a fronteira

**Relações a mapear (mínimo 6):**
1. **Catalog ↔ Inventory** — Partnership (evolução conjunta, sincronização de SKU)
2. **Catalog → Sales** — OHS/PL (Catalog publica contrato estável, Sales consome)
3. **Sales → Billing** — Customer-Supplier (Sales upstream, Billing downstream)
4. **Sales → Shipping** — Customer-Supplier (Sales upstream, Shipping downstream)
5. **Identity → Shipping** — ACL (Shipping traduz modelo de Identity para seu contexto)
6. **Identity → Sales** — ACL (Sales traduz autenticação para CustomerId)

**Entregável:** `CONTEXT-MAP.md` com diagrama, tabela de relações e justificativas.

**Critérios de aceite:**
- [ ] Todas as relações entre BCs documentadas (mínimo 6)
- [ ] Cada relação com padrão, direção, motivo e dados trafegados
- [ ] Diagrama visual do Context Map (ASCII ou Mermaid)
- [ ] Nenhum BC diretamente dependente de modelo interno de outro
- [ ] Clear distinction between Upstream and Downstream teams/contexts

---

### Desafio 3.2 — Anti-Corruption Layer (ACL): Tradução de Modelos

Implemente uma ACL entre Identity (Generic) e Sales (Core), protegendo o Core Domain de
conceitos do contexto genérico.

**Requisitos:**
- **Identity Context** expõe: `User { userId, email, fullName, roles, authProvider, lastLogin }`
- **Sales Context** precisa apenas: `CustomerId`, `CustomerName`, `CustomerEmail`
- O ACL deve:
  - Traduzir `User` → `CustomerProfile` (modelo do Sales)
  - Rejeitar concepts que não fazem sentido no Sales (roles, authProvider, lastLogin)
  - Ser uma camada no Sales, não uma camada no Identity
  - Isolar Sales de mudanças no modelo de Identity

**Java 25:**
```java
// Modelo do Identity Context (externo)
public record UserDTO(String userId, String email, String fullName,
        List<String> roles, String authProvider, Instant lastLogin) {}

// Modelo do Sales Context (interno, protegido pelo ACL)
public record CustomerProfile(CustomerId id, CustomerName name,
        EmailAddress email) {}

// ACL — camada de tradução no Sales
public class IdentityACL {
    private final IdentityClient identityClient; // porta para o Identity

    public CustomerProfile translateUser(String userId) {
        UserDTO user = identityClient.findUser(userId);
        return new CustomerProfile(
            CustomerId.from(user.userId()),
            CustomerName.from(user.fullName()),
            EmailAddress.from(user.email())
            // roles, authProvider, lastLogin — ignorados (não são conceito de Sales)
        );
    }
}

// Interface (porta) no Sales — o Sales define o que precisa
public interface IdentityClient {
    UserDTO findUser(String userId);
}
```

**Go 1.26:**
```go
// Modelo externo (Identity)
type UserDTO struct {
    UserID       string
    Email        string
    FullName     string
    Roles        []string
    AuthProvider string
    LastLogin    time.Time
}

// Modelo interno (Sales) — protegido pelo ACL
type CustomerProfile struct {
    ID    CustomerId
    Name  CustomerName
    Email EmailAddress
}

// ACL — tradução
type IdentityACL struct {
    client IdentityClient
}

func (acl *IdentityACL) TranslateUser(userID string) (*CustomerProfile, error) {
    user, err := acl.client.FindUser(userID)
    if err != nil {
        return nil, fmt.Errorf("identity lookup failed: %w", err)
    }
    return &CustomerProfile{
        ID:    CustomerIdFrom(user.UserID),
        Name:  CustomerNameFrom(user.FullName),
        Email: EmailAddressFrom(user.Email),
    }, nil
}
```

**Critérios de aceite:**
- [ ] ACL como camada no contexto downstream (Sales), não no upstream (Identity)
- [ ] Tradução de `User` → `CustomerProfile` descartando campos irrelevantes
- [ ] Sales nunca importa tipos do Identity diretamente
- [ ] Interface `IdentityClient` definida no Sales (Ports & Adapters)
- [ ] Se Identity muda seu modelo, apenas o ACL precisa mudar no Sales
- [ ] Testes: tradução correta, campos ignorados, erro do upstream tratado

---

### Desafio 3.3 — Open Host Service + Published Language (OHS/PL)

Implemente o Catalog como Open Host Service com Published Language para o Sales.

**Requisitos:**
- **Catalog** (upstream) publica uma **API pública estável** (Open Host Service)
- A API tem uma **Published Language**: um contrato formal (DTOs + eventos) versionado
- **Sales** (downstream) consome a Published Language, NÃO o modelo interno do Catalog
- A Published Language define:
  - `CatalogProductView` — visão pública de produto (ID, nome, preço, status)
  - `ProductPriceChanged` event — evento publicado pelo Catalog
  - `ProductDiscontinued` event — evento para Sales descontinuar itens

**Java 25:**
```java
// --- Published Language (módulo compartilhado, versionado) ---
// Estes tipos pertencem à Published Language do Catalog
package merchanthub.catalog.published;

public record CatalogProductView(
    String productId,      // String, NÃO ProductId (publicado como tipo simples)
    String name,
    String priceAmount,    // Serializado como string para interoperabilidade
    String priceCurrency,
    String status
) {}

public record ProductPriceChangedEvent(
    String eventId,
    String productId,
    String oldPriceAmount,
    String newPriceAmount,
    String currency,
    Instant occurredAt
) {}

// --- Open Host Service (no Catalog) ---
public class CatalogService {
    private final CatalogProductRepository repository;

    // API pública — retorna Published Language, não modelo interno
    public CatalogProductView getProduct(String productId) {
        var product = repository.findById(ProductId.from(productId))
            .orElseThrow(() -> new EntityNotFound("Product", productId));
        return toCatalogProductView(product); // traduz interno → publicado
    }

    private CatalogProductView toCatalogProductView(CatalogProduct product) {
        return new CatalogProductView(
            product.id().value(),
            product.name(),
            product.basePrice().amount().toPlainString(),
            product.basePrice().currency().code(),
            product.status().name()
        );
    }
}
```

**Go 1.26:**
```go
// Published Language
type CatalogProductView struct {
    ProductID     string `json:"productId"`
    Name          string `json:"name"`
    PriceAmount   string `json:"priceAmount"`
    PriceCurrency string `json:"priceCurrency"`
    Status        string `json:"status"`
}

// Open Host Service
type CatalogService struct {
    repo CatalogProductRepository
}

func (s *CatalogService) GetProduct(productID string) (*CatalogProductView, error) { ... }
```

**Critérios de aceite:**
- [ ] Published Language com tipos simples (strings, não Value Objects internos) para interoperabilidade
- [ ] CatalogService retorna Published Language, nunca modelo interno
- [ ] Eventos publicados em formato serializado (Published Language)
- [ ] Sales consome `CatalogProductView`, não `CatalogProduct`
- [ ] Versioning: Published Language pode evoluir sem quebrar consumidores
- [ ] Testes: tradução interna → publicada, consumo pelo Sales

---

### Desafio 3.4 — Customer-Supplier: Sales → Billing

Implemente o padrão Customer-Supplier entre Sales (upstream/supplier) e Billing (downstream/customer).

**Requisitos:**
- **Sales** (Supplier/Upstream) define o contrato que **Billing** consome
- **Billing** (Customer/Downstream) pode influenciar a evolução do contrato
- Quando `OrderConfirmed` acontece no Sales, Billing deve gerar fatura
- O contrato entre eles:
  - Sales publica `OrderConfirmed` event com dados suficientes para Billing
  - Billing consome o evento e cria `Invoice`
  - Billing não acessa o banco de dados do Sales
  - Billing pode solicitar novos campos no evento (customer influence)

**Java 25:**
```java
// Contrato definido por Sales (Upstream) — Billing influencia
public record OrderConfirmedForBilling(
    String eventId,
    String orderId,
    String customerId,
    String customerEmail,           // adicionado a pedido do Billing
    String totalAmount,
    String currency,
    List<LineItem> lineItems,       // detalhamento a pedido do Billing
    Instant occurredAt
) {
    public record LineItem(String productId, String productName,
        int quantity, String unitPrice) {}
}

// Handler no Billing Context (Downstream)
public class OrderConfirmedHandler implements EventHandler<OrderConfirmedForBilling> {
    private final InvoiceRepository invoiceRepo;

    @Override
    public void handle(OrderConfirmedForBilling event) {
        var invoice = Invoice.createFrom(
            InvoiceId.generate(),
            CustomerId.from(event.customerId()),
            Money.of(event.totalAmount(), event.currency()),
            event.lineItems().stream()
                .map(li -> InvoiceLine.of(li.productName(), li.quantity(),
                    Money.of(li.unitPrice(), event.currency())))
                .toList()
        );
        invoiceRepo.save(invoice);
    }
}
```

**Go 1.26:**
```go
type OrderConfirmedForBilling struct {
    EventID       string
    OrderID       string
    CustomerID    string
    CustomerEmail string
    TotalAmount   string
    Currency      string
    LineItems     []LineItem
    OccurredAt    time.Time
}

type OrderConfirmedHandler struct {
    invoiceRepo InvoiceRepository
}

func (h *OrderConfirmedHandler) Handle(event OrderConfirmedForBilling) error { ... }
```

**Critérios de aceite:**
- [ ] Sales (Upstream) define o formato do evento
- [ ] Billing (Downstream) pode solicitar campos adicionais (customer influence)
- [ ] Billing cria `Invoice` a partir do evento — sem acesso direto ao Sales
- [ ] Billing tem seu próprio modelo (`Invoice`, `InvoiceLine`) — não reusa modelo do Sales
- [ ] Evento contém dados suficientes para Billing operar autonomamente
- [ ] Testes: handler de billing cria invoice correto a partir do evento

---

### Desafio 3.5 — Partnership: Catalog ↔ Inventory

Implemente o padrão Partnership entre Catalog e Inventory.

**Requisitos:**
- **Parceria**: ambos os times evoluem juntos, sem upstream/downstream fixo
- Ambos compartilham o conceito de SKU mas com significados diferentes:
  - Catalog: SKU identifica produto para listagem e preço
  - Inventory: SKU identifica item para controle de estoque (quantidade, localização)
- Sincronização bidirecional via eventos:
  - Catalog publica `ProductCreated` → Inventory cria `InventoryItem`
  - Inventory publica `StockDepleted` → Catalog marca produto como `OUT_OF_STOCK`
- Cada contexto tem seu próprio modelo e reage a eventos do parceiro

**Java 25:**
```java
// --- Catalog publica ---
public record ProductCreatedEvent(String eventId, String sku,
        String productName, Instant occurredAt) {}

// --- Inventory reage ---
public class ProductCreatedInventoryHandler
        implements EventHandler<ProductCreatedEvent> {
    private final InventoryItemRepository inventoryRepo;

    @Override
    public void handle(ProductCreatedEvent event) {
        var item = InventoryItem.createNew(
            InventoryItemId.generate(),
            SKU.from(event.sku()),
            Quantity.zero(), // estoque inicial zero
            Location.defaultWarehouse()
        );
        inventoryRepo.save(item);
    }
}

// --- Inventory publica ---
public record StockDepletedEvent(String eventId, String sku,
        Instant occurredAt) {}

// --- Catalog reage ---
public class StockDepletedCatalogHandler
        implements EventHandler<StockDepletedEvent> {
    private final CatalogProductRepository catalogRepo;

    @Override
    public void handle(StockDepletedEvent event) {
        catalogRepo.findBySku(SKU.from(event.sku())).ifPresent(product -> {
            product.markOutOfStock();
            catalogRepo.save(product);
        });
    }
}
```

**Go 1.26:**
```go
type ProductCreatedInventoryHandler struct {
    inventoryRepo InventoryItemRepository
}

func (h *ProductCreatedInventoryHandler) Handle(event ProductCreatedEvent) error { ... }

type StockDepletedCatalogHandler struct {
    catalogRepo CatalogProductRepository
}

func (h *StockDepletedCatalogHandler) Handle(event StockDepletedEvent) error { ... }
```

**Critérios de aceite:**
- [ ] Parceria bidirecional — ambos publicam e consomem eventos
- [ ] SKU como conceito compartilhado mas com semânticas distintas
- [ ] Catalog e Inventory têm modelos independentes (não reusam Entities)
- [ ] Eventos como mecanismo de sincronização (não chamadas diretas)
- [ ] Cada handler traduz evento para modelo do seu contexto
- [ ] Testes: handler de cada lado, sincronização bidirecional

---

### Desafio 3.6 — Shared Kernel e Separate Ways

Implemente um Shared Kernel mínimo e identifique um caso de Separate Ways.

**Requisitos:**

**Shared Kernel (mínimo e deliberado):**
- Identificar tipos que **devem** ser compartilhados entre todos os BCs
- Candidatos: `Money`, `DateRange`, tipos de ID base
- Colocar em módulo/pacote compartilhado (`shared-kernel`)
- **Regras rígidas:**
  - Mudanças no Shared Kernel exigem acordo de todos os BCs consumidores
  - Deve ter testes de contrato para garantir retrocompatibilidade
  - Mudanças versionadas (changelog)
  - Deve ser o menor possível

**Separate Ways:**
- Identificar ao menos 1 par de BCs que **não precisa se integrar**
- Exemplo: Billing ↔ Inventory — Billing não precisa saber de estoque, Inventory não precisa saber de faturas
- Documentar o motivo da separação e como cada um resolve suas necessidades independentemente

**Java 25:**
```java
// shared-kernel (módulo compartilhado mínimo)
package merchanthub.sharedkernel;

// Value Objects universais — usados por todos os BCs
public record Money(BigDecimal amount, Currency currency) {
    // ... (já implementado no Level 0)
}

public sealed interface DomainId {
    String value();
}

public record DateRange(LocalDate start, LocalDate end) {
    // ... (já implementado no Level 0)
}

// Testes de contrato para o Shared Kernel
public class SharedKernelContractTest {
    @Test
    void money_must_support_all_currencies_used_by_contexts() { ... }

    @Test
    void dateRange_must_be_inclusive_on_both_ends() { ... }

    @Test
    void domainId_must_serialize_to_string() { ... }
}
```

**Go 1.26:**
```go
// sharedkernel/money.go
package sharedkernel

type Money struct { ... }  // implementado no Level 0
type DateRange struct { ... }

// sharedkernel/contract_test.go
func TestMoneyContract(t *testing.T) { ... }
```

**Critérios de aceite:**
- [ ] Shared Kernel com no máximo 5 tipos (Money, DateRange, DomainId, etc.)
- [ ] Módulo/pacote separado (`shared-kernel`)
- [ ] Testes de contrato que validam retrocompatibilidade
- [ ] Changelog para mudanças no Shared Kernel
- [ ] Separate Ways: ao menos 1 par de BCs documentado como não integrado
- [ ] Justificativa de por que cada tipo está (ou não) no Shared Kernel

---

## Entregáveis

| Artefato | Descrição |
|----------|-----------|
| `CONTEXT-MAP.md` | Documento com todas as relações mapeadas e diagrama |
| `IdentityACL` | Anti-Corruption Layer protegendo Sales de Identity |
| `CatalogService` + Published Language | OHS/PL expondo Catalog para Sales |
| Event Handlers Billing | Customer-Supplier: Sales → Billing via eventos |
| Event Handlers bidirecionais | Partnership: Catalog ↔ Inventory |
| `shared-kernel` module | Shared Kernel mínimo com testes de contrato |
| Separate Ways doc | Documentação de BCs não integrados |
| Testes | ≥ 85% cobertura, incluindo testes de integração entre BCs |

---

## O que você vai exercitar

| Padrão | O que pratica |
|--------|---------------|
| **Context Map** | Visão holística das relações entre BCs, documentação arquitetural |
| **ACL** | Tradução de modelos, proteção do Core Domain, isolamento de mudanças |
| **OHS/PL** | API pública estável, tipos serializáveis, versionamento de contrato |
| **Customer-Supplier** | Contrato dirigido pelo upstream com influência do downstream |
| **Partnership** | Sincronização bidirecional via eventos, evolução conjunta |
| **Shared Kernel** | Tipos mínimos compartilhados, testes de contrato, governança |
| **Separate Ways** | Decisão consciente de não integrar, autonomia de BCs |

---

## Próximo Nível

Quando completar todos os desafios deste nível, avance para:
→ [Level 4 — Anti-Patterns & Refactoring](04-anti-patterns.md)

Os padrões de integração implementados aqui serão contrastados com anti-patterns comuns e refatorados no próximo nível.
