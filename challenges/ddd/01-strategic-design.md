# Level 1 — Strategic Design

> **Objetivo:** Dominar Design Estratégico de DDD — identificar Bounded Contexts, classificar Subdomains,
> definir fronteiras e aplicar Context Discovery no domínio MerchantHub.

**Referência:** [01-strategic-design.md](../../.docs/ddd/01-strategic-design.md)

**Pré-requisito:** [Level 0 — Fundações DDD](00-ddd-foundations.md) completo.

---

## Contexto do Domínio

Neste nível, você vai **desenhar as fronteiras** do MerchantHub. O foco é entender que DDD Estratégico é sobre **onde** modelar, não **como**. Você vai identificar Bounded Contexts, classificar subdomains e garantir que cada contexto tem sua própria linguagem ubíqua refletida no código.

Atividades:
- Identificar e justificar os Bounded Contexts do MerchantHub
- Classificar cada subdomain (Core, Supporting, Generic)
- Implementar modelos independentes para o mesmo conceito em contextos diferentes
- Criar fronteiras claras entre módulos no código
- Validar que a Ubiquitous Language está refletida em cada contexto

---

## Desafios

### Desafio 1.1 — Identificação de Bounded Contexts

Analise o domínio MerchantHub e defina formalmente os Bounded Contexts.

**Requisitos:**
- Documentar **pelo menos 6 Bounded Contexts** com justificativa
- Para cada contexto, definir:
  - Nome (linguagem de domínio)
  - Responsabilidade principal
  - Ubiquitous Language (termos-chave do contexto)
  - Entidades/conceitos principais
  - O que **não** pertence ao contexto (fronteira explícita)
- Usar heurísticas de Context Discovery para justificar decisões:
  - Linguagem diverge?
  - Equipes diferentes?
  - Ciclo de vida diferente?
  - Requisitos de escalabilidade distintos?

**Entregável: `BOUNDED-CONTEXTS.md`**

```markdown
## Bounded Context: Sales

**Responsabilidade:** Gerenciar o ciclo de vida de pedidos — do carrinho à confirmação.

**Linguagem ubíqua:**
- Order: pedido confirmado com itens, endereço e pagamento
- Cart: carrinho em andamento (não é pedido)
- OrderItem: linha de item no pedido com quantidade e preço unitário
- Discount: redução de preço por regra promocional

**Conceitos principais:** Order, Cart, OrderItem, Discount, Promotion

**Não pertence aqui:**
- Descrição de produto (→ Catalog)
- Quantidade em estoque (→ Inventory)
- Nota fiscal (→ Billing)
- Rastreamento de entrega (→ Shipping)

**Heurísticas aplicadas:**
- Linguagem diverge: "Product" em Sales = { productId, quantity, unitPrice };
  em Catalog = { name, description, images, price }
- Ciclo de vida: Order evolui (created→confirmed→paid); Product no Catalog é relativamente estável
```

**Critérios de aceite:**
- [ ] 6+ Bounded Contexts documentados com justificativa
- [ ] Cada contexto com linguagem ubíqua, responsabilidade e fronteira explícita
- [ ] Heurísticas de Context Discovery aplicadas
- [ ] Demonstração clara de que termos iguais significam coisas diferentes em contextos diferentes

---

### Desafio 1.2 — Classificação de Subdomains

Classifique cada Bounded Context como Core, Supporting ou Generic e justifique decisões de investimento.

**Requisitos:**
- Classificar cada contexto identificado no Desafio 1.1
- Para cada classificação, responder às perguntas-chave:
  - "Isso nos diferencia da concorrência?"
  - "Precisamos construir do zero?"
  - "Compramos pronto no mercado?"
  - "Complexidade de regras de negócio?"
- Definir **nível de investimento** de modelagem para cada subdomain:
  - **Core** → DDD tático completo (Aggregates, Events, modelagem rica)
  - **Supporting** → DDD simplificado (Entities ricas, mas sem necessariamente Events/CQRS)
  - **Generic** → CRUD simples, bibliotecas prontas, SaaS

**Entregável: `SUBDOMAINS.md`**

```markdown
| Bounded Context | Subdomain | Diferencial? | Complexidade | Investimento |
|----------------|-----------|-------------|-------------|-------------|
| Catalog | Core | Sim — pricing/promoções | Alta | DDD tático completo |
| Sales | Core | Sim — checkout, regras de pedido | Alta | DDD tático completo |
| Inventory | Supporting | Não — necessário mas não diferencial | Média | DDD simplificado |
| Shipping | Supporting | Não — necessário mas não diferencial | Média | DDD simplificado |
| Billing | Supporting | Não — NF/impostos são genéricos | Média | DDD simplificado |
| Identity | Generic | Não — qualquer empresa precisa | Baixa | CRUD / lib pronta |
```

**Critérios de aceite:**
- [ ] Todos os contextos classificados com justificativa
- [ ] Perguntas-chave respondidas para cada subdomain
- [ ] Nível de investimento definido e coerente com a classificação
- [ ] Reconhecimento de que DDD tático completo em Generic Subdomain é over-engineering

---

### Desafio 1.3 — Modelos Independentes por Contexto (`Product` em 3 Bounded Contexts)

Implemente o conceito de "Product" em **3 Bounded Contexts diferentes**, mostrando que são modelos completamente independentes.

**Requisitos:**
- **Catalog Context** — `CatalogProduct`: id, name, description, images, price, category, status
- **Inventory Context** — `InventoryItem`: productId (referência por ID), quantity, warehouseLocation, reorderLevel
- **Shipping Context** — `ShippableItem`: productId (referência por ID), weight, dimensions, isFragile

Cada modelo deve:
- Estar em **package/module separado** (fronteira de contexto no código)
- Ter **apenas os atributos relevantes** para seu contexto
- Referenciar o produto original apenas por **ID** (nunca por objeto compartilhado)
- Ter comportamento próprio (métodos de domínio específicos do contexto)

**Java 25:**
```java
// Package: com.merchanthub.catalog.domain.model
public final class CatalogProduct {
    private final ProductId id;
    private final String name;
    private final String description;
    private final List<Image> images;
    private final Money price;
    private final CategoryId categoryId;
    private ProductStatus status;

    public void activate() { ... }
    public void updatePrice(Money newPrice) { ... }
    public void discontinue() { ... }
}

// Package: com.merchanthub.inventory.domain.model
public final class InventoryItem {
    private final ProductId productId;  // referência por ID
    private Quantity quantityOnHand;
    private final WarehouseLocation location;
    private final Quantity reorderLevel;

    public void reserve(Quantity qty) { ... }
    public void restock(Quantity qty) { ... }
    public boolean needsReorder() { ... }
}

// Package: com.merchanthub.shipping.domain.model
public final class ShippableItem {
    private final ProductId productId;  // referência por ID
    private final Weight weight;
    private final Dimensions dimensions;
    private final boolean fragile;

    public ShippingCategory shippingCategory() { ... }
    public Money estimateShippingCost(Address destination) { ... }
}
```

**Go 1.26:**
```go
// Package: catalog
type CatalogProduct struct { ... }
func (p *CatalogProduct) Activate() { ... }

// Package: inventory
type InventoryItem struct { ... }
func (i *InventoryItem) Reserve(qty Quantity) error { ... }

// Package: shipping
type ShippableItem struct { ... }
func (s ShippableItem) ShippingCategory() ShippingCategory { ... }
```

**Critérios de aceite:**
- [ ] 3 modelos distintos em packages/modules separados
- [ ] Cada modelo com apenas os atributos relevantes para seu contexto
- [ ] Referência entre contextos apenas por ID (`ProductId`), nunca por objeto
- [ ] Cada modelo com comportamento de domínio específico do contexto
- [ ] Sem import entre packages de domínio (fronteira real no código)
- [ ] Testes unitários para cada modelo independentemente

---

### Desafio 1.4 — Fronteiras de Contexto no Código (Module Structure)

Organize a estrutura de projeto de forma que as fronteiras de Bounded Context sejam **reforçadas pelo código**.

**Requisitos:**
- Estrutura de diretórios por Bounded Context, **não** por tipo técnico
- Cada contexto com sua própria camada de domínio isolada
- Shared Kernel mínimo (apenas Value Objects compartilhados: `Money`, `ProductId`)
- Criar **testes de arquitetura** que verifiquem que fronteiras não foram violadas

**Estrutura esperada:**
```
src/
├── catalog/
│   └── domain/
│       ├── model/
│       │   ├── CatalogProduct
│       │   ├── Category
│       │   └── ProductStatus
│       ├── event/
│       │   └── ProductCreated
│       └── repository/
│           └── CatalogProductRepository (interface)
│
├── sales/
│   └── domain/
│       ├── model/
│       │   ├── Order
│       │   ├── Cart
│       │   └── OrderItem
│       ├── event/
│       │   └── OrderPlaced
│       └── repository/
│           └── OrderRepository (interface)
│
├── inventory/
│   └── domain/
│       ├── model/
│       │   └── InventoryItem
│       ├── event/
│       │   └── StockReserved
│       └── repository/
│           └── InventoryItemRepository (interface)
│
└── shared/
    └── kernel/
        ├── Money
        ├── ProductId
        ├── CustomerId
        └── DomainEvent (interface)
```

**Testes de Arquitetura (Java — ArchUnit):**
```java
@Test
void salesContextDoesNotDependOnCatalogInternals() {
    noClasses()
        .that().resideInAPackage("..sales..")
        .should().dependOnClassesThat()
        .resideInAPackage("..catalog.domain.model..")
        .check(importedClasses);
}

@Test
void domainDoesNotDependOnInfrastructure() {
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..infrastructure..", "..adapter..")
        .check(importedClasses);
}
```

**Testes de Arquitetura (Go — verificação por imports):**
```go
func TestBoundaryViolation(t *testing.T) {
    // Verificar que sales/ não importa catalog/domain/model
    salesFiles := findGoFiles("sales/")
    for _, file := range salesFiles {
        imports := extractImports(file)
        for _, imp := range imports {
            if strings.Contains(imp, "catalog/domain/model") {
                t.Errorf("Boundary violation: %s imports catalog internals", file)
            }
        }
    }
}
```

**Critérios de aceite:**
- [ ] Estrutura de diretórios por Bounded Context implementada
- [ ] Cada contexto com seu domínio isolado
- [ ] Shared Kernel com **no máximo** 5-6 tipos (IDs + Money + DomainEvent interface)
- [ ] Testes de arquitetura que validam fronteiras (Java: ArchUnit, Go: verificação de imports)
- [ ] Sem imports cruzados entre camadas de domínio de contextos diferentes
- [ ] Comunicação entre contextos preparada via interfaces públicas ou eventos

---

### Desafio 1.5 — Context Discovery Workshop (Event Storming Simplificado)

Realize um exercício de **Context Discovery** usando Event Storming simplificado para validar as fronteiras definidas.

**Requisitos:**
- Listar **todos os Domain Events** que podem ocorrer no MerchantHub (pelo menos 20)
- Agrupar eventos por **fluxo de negócio** (não por tela ou API)
- Identificar **pivotal events** — eventos que marcam transição entre contextos
- Validar se os Bounded Contexts definidos no Desafio 1.1 são coerentes com os eventos
- Documentar em `EVENT-STORMING.md`

**Exemplo de fluxo:**

```
Fluxo: Compra de Produto
─────────────────────────────────────────────────────
ProductCreated → ProductActivated → ProductAddedToCart →
CartItemQuantityChanged → CheckoutStarted →
OrderPlaced → PaymentRequested → PaymentReceived →
StockReserved → InvoiceGenerated → ShipmentCreated →
ShipmentDispatched → ShipmentDelivered → OrderCompleted

Pivotal Events (transição entre contextos):
  • OrderPlaced (Sales → Inventory, Billing)
  • PaymentReceived (Billing → Sales, Shipping)
  • StockReserved (Inventory → Sales)
  • ShipmentDispatched (Shipping → Sales, Notification)
```

**Entregável: `EVENT-STORMING.md`** com:
- Lista de 20+ Domain Events nomeados como fatos de negócio (passado)
- Agrupamento por fluxo de negócio
- Pivotal events identificados
- Mapa de quais contextos produzem e consomem cada evento

**Critérios de aceite:**
- [ ] 20+ Domain Events listados com naming correto (passado, linguagem ubíqua)
- [ ] Eventos agrupados por fluxo de negócio
- [ ] Pivotal events claramente identificados
- [ ] Mapa de produtor/consumidor por evento
- [ ] Eventos coerentes com os Bounded Contexts definidos
- [ ] Nenhum evento com nome técnico (`EntityUpdated`, `StatusChanged`)

---

## Entregáveis

| Artefato | Descrição |
|----------|-----------|
| `BOUNDED-CONTEXTS.md` | Documentação dos 6+ Bounded Contexts com justificativa |
| `SUBDOMAINS.md` | Classificação de Subdomains (Core/Supporting/Generic) |
| 3 modelos de `Product` | Modelos independentes em 3 contextos (Catalog, Inventory, Shipping) |
| Estrutura de módulos | Projeto organizado por Bounded Context com fronteiras reais |
| Testes de arquitetura | Validação automatizada de fronteiras entre contextos |
| `EVENT-STORMING.md` | Context Discovery com 20+ Domain Events mapeados |

---

## O que você vai exercitar

| Conceito DDD | O que pratica |
|-------------|---------------|
| **Ubiquitous Language** | Glossário por contexto, naming no código |
| **Bounded Context** | Fronteiras semânticas e técnicas, modelos independentes |
| **Subdomains** | Classificação Core/Supporting/Generic, decisão de investimento |
| **Context Discovery** | Event Storming, identificar fronteiras por linguagem e eventos |
| **Module Organization** | Estrutura por domínio (não por tipo técnico), testes de arquitetura |

---

## Próximo Nível

Quando completar todos os desafios deste nível, avance para:
→ [Level 2 — Tactical Design (Building Blocks)](02-tactical-design.md)

As fronteiras e os modelos definidos aqui serão a base para implementar Entities ricas, Aggregates e Domain Events nos próximos níveis.
