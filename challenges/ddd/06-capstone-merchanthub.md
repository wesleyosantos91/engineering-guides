# Level 6 — Capstone: MerchantHub Completo

> **Objetivo:** Consolidar **todos** os conceitos de DDD dos Levels 0-5 em uma implementação
> completa do MerchantHub — um sistema de e-commerce multi-tenant com 6 Bounded Contexts,
> modelado como Modular Monolith com arquitetura hexagonal, CQRS e Event Sourcing.

**Referência:** Todos os documentos anteriores:
- [01-strategic-design.md](../../.docs/ddd/01-strategic-design.md)
- [02-tactical-design.md](../../.docs/ddd/02-tactical-design.md)
- [03-context-mapping.md](../../.docs/ddd/03-context-mapping.md)
- [04-anti-patterns.md](../../.docs/ddd/04-anti-patterns.md)
- [05-architecture-integration.md](../../.docs/ddd/05-architecture-integration.md)

**Pré-requisito:** Levels 0-5 completos.

---

## Visão Geral do Capstone

Este é o projeto integrador. Você vai implementar o **MerchantHub** como um sistema
funcional que demonstra domínio de todas as disciplinas de DDD aprendidas nos níveis anteriores.

### Arquitetura Alvo

```
┌─────────────────────────────────────────────────────────────────┐
│                        MerchantHub                              │
│                     (Modular Monolith)                          │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Catalog  │  │  Sales   │  │Inventory │  │ Billing  │       │
│  │  (Clean  │  │  (Hex +  │  │  (Hex)   │  │ (Hex)    │       │
│  │   Arch)  │  │ CQRS+ES) │  │          │  │          │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │  OHS/PL      │  C-S       │Partnership   │             │
│       └──────►───────┘──────►─────┘◄─────────────┘             │
│                      │                                          │
│  ┌──────────┐  ┌─────▼────┐                                    │
│  │ Identity │  │  Event   │                                    │
│  │(Generic) │  │   Bus    │                                    │
│  └──────────┘  └──────────┘                                    │
│                                                                 │
│  ┌──────────────────────────────────────────────────┐          │
│  │              Shared Kernel (mínimo)               │          │
│  │  Money · DateRange · DomainEvent · DomainId       │          │
│  └──────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Parte 1 — Fundações (Levels 0-1)

### 1.1 — Glossário Ubíquo e Shared Kernel

Consolide o glossário e o shared kernel usado por todo o sistema.

**Entregáveis:**
- `GLOSSARY.md` — mínimo 8 termos por BC (48 termos total), com homônimos identificados
- `shared-kernel/` package com: `Money`, `DateRange`, `DomainEvent`, `DomainId` base,
  `Percentage`, `Quantity`
- Testes de contrato para cada tipo do shared kernel

**Critérios de aceite:**
- [ ] Glossário com 48+ termos organizados por BC
- [ ] Homônimos documentados (ex: "Product" em Catalog vs Inventory vs Sales)
- [ ] Shared Kernel com ≤ 8 tipos, testes de contrato, changelog
- [ ] Value Objects imutáveis, validados no construtor, com igualdade por valor

---

### 1.2 — Bounded Contexts e Subdomains

Formalize o Context Map e a classificação de subdomínios.

**Entregáveis:**
- `CONTEXT-MAP.md` com diagrama e todas as relações documentadas
- `SUBDOMAINS.md` com classificação Core/Supporting/Generic e nível de investimento
- Estrutura de diretórios por BC (6 módulos)

**Critérios de aceite:**
- [ ] 6 Bounded Contexts com responsabilidades claras
- [ ] Todas as relações entre BCs mapeadas com padrão (ACL, OHS/PL, C-S, Partnership, etc.)
- [ ] Subdomains classificados: Core (Catalog, Sales), Supporting (Inventory, Shipping, Billing), Generic (Identity)
- [ ] Estrutura de diretórios reflete BCs, não camadas técnicas

---

## Parte 2 — Modelagem Tática (Level 2)

### 2.1 — Catalog BC (Clean Architecture)

Implemente o Catalog como módulo com Clean Architecture.

**Aggregates:**
- `CatalogProduct` (root) com `Image` (VO), `ProductStatus` (enum)
- `Category` (root) com hierarquia (parent/children)

**Use Cases:**
- `CreateProductUseCase` — cria produto em DRAFT
- `ActivateProductUseCase` — ativa produto (requer imagens + preço)
- `UpdatePriceUseCase` — atualiza preço e publica evento
- `DiscontinueProductUseCase` — discontinua irreversivelmente
- `GetProductUseCase` — busca por ID (retorna DTO, não Entity)
- `SearchProductsUseCase` — busca por nome/categoria

**Critérios de aceite:**
- [ ] Clean Architecture: 4 camadas com Dependency Rule
- [ ] Use Cases com Input/Output DTOs
- [ ] `CatalogProduct` com comportamento rico e invariantes
- [ ] Domain Events: ProductCreated, ProductActivated, ProductPriceChanged, ProductDiscontinued
- [ ] Testes: unit (entity, usecase), integration (adapter)

---

### 2.2 — Sales BC (Hexagonal + CQRS + Event Sourcing)

Implemente o Sales como módulo com a arquitetura mais sofisticada.

**Aggregates:**
- `Order` (root) com `OrderItem` (entidade interna), Event-Sourced
- Estado reconstruído a partir de eventos

**Write Side (Commands):**
- `PlaceOrderCommand` → cria Order, valida contra Catalog
- `ConfirmOrderCommand` → confirma, publica OrderConfirmed
- `CancelOrderCommand` → cancela com motivo, publica OrderCancelled
- `AddItemCommand` → adiciona item a Order PENDING
- `ChangeItemQuantityCommand` → altera quantidade

**Read Side (Queries):**
- `GetOrderQuery` → retorna OrderReadModel
- `ListOrdersByCustomerQuery` → lista por customer
- `OrderSummaryQuery` → dashboard com contagem por status

**Critérios de aceite:**
- [ ] Hexagonal: Ports inbound/outbound, Adapters in/out
- [ ] Event Sourcing: Event Store, reconstituição, optimistic concurrency
- [ ] CQRS: Command handlers (write), Query handlers (read), Projeção
- [ ] Domain Events gerados pelo Aggregate, persistidos no Event Store
- [ ] Flow completo: command → event store → projeção → query
- [ ] Testes: reconstructão de aggregate, projeções, concorrência

---

### 2.3 — Inventory BC (Hexagonal)

Implemente o Inventory com Hexagonal Architecture simples.

**Aggregates:**
- `InventoryItem` (root) — SKU, quantity, reorderPoint, warehouse

**Comportamento:**
- `receiveStock(quantity)` — aumenta estoque, publica StockReceived
- `reserveStock(quantity)` — reserva para pedido, publica StockReserved
- `releaseStock(quantity)` — libera reserva cancelada
- `depleteStock()` — marca estoque zerado, publica StockDepleted

**Critérios de aceite:**
- [ ] Hexagonal com ports/adapters
- [ ] Invariantes: estoque nunca negativo, reserva ≤ disponível
- [ ] Reage a `ProductCreated` do Catalog (Partnership) para criar InventoryItem
- [ ] Publica `StockDepleted` para Catalog marcar OUT_OF_STOCK

---

### 2.4 — Billing BC (Hexagonal)

Implemente o Billing com geração de faturas.

**Aggregates:**
- `Invoice` (root) com `InvoiceLine` (entidade interna)

**Comportamento:**
- Reage a `OrderConfirmed` do Sales (Customer-Supplier) para criar Invoice
- `issue()` — emite fatura, publica InvoiceIssued
- `markPaid()` — marca como paga

**Critérios de aceite:**
- [ ] Invoice criado automaticamente a partir de OrderConfirmed
- [ ] Modelo próprio (Invoice, InvoiceLine) — não reusa modelo do Sales
- [ ] Customer-Supplier: Billing (downstream) usa contrato do Sales (upstream)

---

### 2.5 — Shipping BC (Hexagonal)

Implemente o Shipping com rastreamento de entregas.

**Aggregates:**
- `Shipment` (root) com `TrackingHistory` (VO)

**Comportamento:**
- Reage a `OrderConfirmed` para criar Shipment
- `dispatch()` → `addTrackingUpdate()` → `deliver()`
- Publica `ShipmentDispatched`, `ShipmentDelivered`

**Critérios de aceite:**
- [ ] Shipment com máquina de estados (PENDING → DISPATCHED → IN_TRANSIT → DELIVERED)
- [ ] ACL para traduzir dados do Identity quando necessário
- [ ] Histórico de rastreamento como coleção de VOs

---

### 2.6 — Identity BC (Generic — Simplificado)

Implemente o Identity como módulo genérico simplificado.

**Modelo:**
- `User` com `UserId`, `Email`, `FullName`, `Roles`
- Autenticação simulada (in-memory)

**Critérios de aceite:**
- [ ] Módulo genérico com investimento mínimo
- [ ] Provê dados para ACLs em Sales e Shipping
- [ ] API pública para consulta de usuário

---

## Parte 3 — Integração (Level 3)

### 3.1 — Context Mapping Implementado

Implemente todas as integrações entre BCs.

| Relação | Padrão | Implementação |
|---------|--------|---------------|
| Catalog ↔ Inventory | Partnership | Eventos bidirecionais via Event Bus |
| Catalog → Sales | OHS/PL | Published Language (CatalogProductView) |
| Sales → Billing | Customer-Supplier | OrderConfirmedForBilling event |
| Sales → Shipping | Customer-Supplier | OrderConfirmedForShipping event |
| Identity → Sales | ACL | IdentityACL traduz User → CustomerProfile |
| Identity → Shipping | ACL | IdentityACL traduz User → ShippingContact |

**Critérios de aceite:**
- [ ] 6 integrações implementadas com padrões corretos
- [ ] Nenhum import direto entre módulos (apenas via ports/eventos)
- [ ] Testes de integração cross-module passando

---

## Parte 4 — Qualidade e Governança (Levels 4-5)

### 4.1 — Zero Anti-Patterns

Valide que nenhum anti-pattern está presente no código.

**Checklist de validação:**
- [ ] Sem Anemic Domain Model — todas as Entities têm comportamento
- [ ] Sem God Aggregate — cada Aggregate com < 7 campos diretos
- [ ] Sem Primitive Obsession — todos os primitivos semânticos são VOs
- [ ] Sem Shared Database — cada BC com seu store
- [ ] Sem Smart UI — controllers sem lógica de negócio
- [ ] Sem CRUD masquerading — operações com intenção de negócio (confirm, não setStatus)

### 4.2 — Testes de Arquitetura

Implemente testes automatizados que validam regras arquiteturais.

**Java 25 (ArchUnit):**
```java
public class ArchitectureTest {
    @Test
    void domain_must_not_depend_on_infrastructure() {
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..adapter..")
            .check(classes);
    }

    @Test
    void modules_must_not_cross_import() {
        noClasses().that().resideInAPackage("..catalog..")
            .should().dependOnClassesThat().resideInAPackage("..sales..internal..")
            .check(classes);
    }

    @Test
    void domain_events_must_be_records() {
        classes().that().implement(DomainEvent.class)
            .should().beRecords()
            .check(classes);
    }
}
```

**Go 1.26:**
```go
func TestDomainNoInfraImports(t *testing.T) {
    // Verifica que pacotes /domain/ não importam pacotes /adapter/ ou /infrastructure/
}

func TestModuleIsolation(t *testing.T) {
    // Verifica que catalog/ não importa sales/internal/
}
```

**Critérios de aceite:**
- [ ] Testes ArchUnit/import que falham se regras forem violadas
- [ ] Regras: domain → 0 deps externas, modules → 0 cross-imports, events → imutáveis
- [ ] CI-ready: testes rodam em build automatizada

### 4.3 — Documentação

Produza documentação arquitetural completa.

**Entregáveis:**
- `ARCHITECTURE.md` — visão geral com diagramas C4 (Context, Container, Component)
- `CONTEXT-MAP.md` — mapa de contextos atualizado
- `ADRs/` — mínimo 5 Architecture Decision Records:
  - ADR-001: Escolha de Modular Monolith
  - ADR-002: Event Sourcing para Sales
  - ADR-003: Clean Architecture para Catalog
  - ADR-004: Shared Kernel mínimo
  - ADR-005: Event Bus in-process
- `GLOSSARY.md` — glossário ubíquo completo

**Critérios de aceite:**
- [ ] Diagrama C4 com 3 níveis (Context, Container, Component)
- [ ] 5+ ADRs com formato: Context, Decision, Consequences
- [ ] Glossário com 48+ termos
- [ ] Documentação navegável com links entre artefatos

---

## Parte 5 — Demonstração de Fluxo End-to-End

### 5.1 — Fluxo Completo: Pedido do Início ao Fim

Implemente e teste o fluxo completo de negócio:

```
1. [Identity] Usuário se autentica
2. [Catalog]  Busca produtos disponíveis (Query)
3. [Sales]    Cria pedido — PlaceOrderCommand
4. [Catalog → Sales]  OHS/PL: Sales busca preço via Published Language
5. [Sales]    Confirma pedido — ConfirmOrderCommand
   → Event: OrderConfirmed
6. [Sales → Inventory] Reserva estoque — StockReserved
7. [Sales → Billing]   Gera fatura — InvoiceIssued
8. [Sales → Shipping]  Cria envio — ShipmentCreated
9. [Billing]  Pagamento confirmado — InvoicePaid
10.[Shipping] Despacha — ShipmentDispatched
11.[Shipping] Entrega — ShipmentDelivered
```

**Java 25 — Integration Test:**
```java
@Test
void complete_order_flow() {
    // 1. Setup modules
    var eventBus = new EventBus();
    var identity = new IdentityModule();
    var catalog = new CatalogModule(eventBus);
    var inventory = new InventoryModule(eventBus, catalog);
    var sales = new SalesModule(eventBus, catalog);
    var billing = new BillingModule(eventBus);
    var shipping = new ShippingModule(eventBus, identity);

    // 2. Create product in Catalog
    var productId = catalog.createProduct(new CreateProductInput(
        "Mechanical Keyboard", "Cherry MX Blue", "299.90", "BRL", "electronics"));
    catalog.activateProduct(productId);

    // 3. Receive stock in Inventory (via Partnership event)
    // ProductCreated → InventoryItem created automatically
    inventory.receiveStock(productId, 100);

    // 4. Place order in Sales
    var orderId = sales.placeOrder(new PlaceOrderCommand(
        CustomerId.from("customer-1"),
        Address.from("Rua A", "São Paulo", "SP", "01000-000", "BR"),
        List.of(new OrderItemRequest(productId, 2))));

    // 5. Confirm order → triggers cross-module events
    sales.confirmOrder(new ConfirmOrderCommand(orderId));

    // 6. Verify cross-module effects
    // Inventory: stock reserved
    var item = inventory.findBySku(productId);
    assertThat(item.availableQuantity()).isEqualTo(98);

    // Billing: invoice created
    var invoice = billing.findByOrderId(orderId);
    assertThat(invoice).isPresent();
    assertThat(invoice.get().status()).isEqualTo("ISSUED");

    // Shipping: shipment created
    var shipment = shipping.findByOrderId(orderId);
    assertThat(shipment).isPresent();
    assertThat(shipment.get().status()).isEqualTo("PENDING");

    // 7. Complete flow
    billing.markPaid(invoice.get().id());
    shipping.dispatch(shipment.get().id());
    shipping.deliver(shipment.get().id());

    // 8. Verify final state via CQRS query
    var orderView = sales.getOrder(orderId.value());
    assertThat(orderView.status()).isEqualTo("CONFIRMED");
}
```

**Critérios de aceite:**
- [ ] Fluxo de 11 passos executável como teste de integração
- [ ] 6 módulos instanciados e comunicando via Event Bus
- [ ] Eventos cruzando fronteiras de BC corretamente
- [ ] ACLs traduzindo modelos nas fronteiras
- [ ] CQRS query retornando visão atualizada
- [ ] Event Sourcing reconstruindo Order corretamente
- [ ] Nenhum acoplamento direto entre módulos (apenas via ports e eventos)

---

## Critérios Globais do Capstone

### Cobertura de Conceitos DDD

| Conceito | Onde aplicado | Level de origem |
|----------|--------------|-----------------|
| Ubiquitous Language | GLOSSARY.md, nomes em código | L0 |
| Value Objects | 10+ VOs no Shared Kernel e BCs | L0 |
| Bounded Contexts | 6 BCs como módulos | L1 |
| Subdomains | Core/Supporting/Generic classificados | L1 |
| Entities | CatalogProduct, InventoryItem, etc. | L2 |
| Aggregates | Order (ES), CatalogProduct, Invoice, etc. | L2 |
| Domain Events | 15+ eventos cross-module | L2 |
| Repositories | 1 por Aggregate Root | L2 |
| Domain Services | PricingService | L2 |
| Factories | OrderFactory | L2 |
| Context Map | 6 relações implementadas | L3 |
| ACL | IdentityACL | L3 |
| OHS/PL | CatalogService + Published Language | L3 |
| Customer-Supplier | Sales → Billing, Sales → Shipping | L3 |
| Partnership | Catalog ↔ Inventory | L3 |
| Anti-pattern free | Zero anti-patterns validados | L4 |
| Hexagonal | Sales, Inventory, Billing, Shipping | L5 |
| Clean Architecture | Catalog | L5 |
| CQRS | Sales (Command/Query separation) | L5 |
| Event Sourcing | Sales Order Aggregate | L5 |
| Modular Monolith | 6 módulos + Event Bus | L5 |

### Métricas de Qualidade

| Métrica | Meta |
|---------|------|
| Cobertura de testes | ≥ 85% (domain ≥ 95%) |
| Anti-patterns | Zero |
| Testes de arquitetura | Todos passando |
| ADRs | ≥ 5 |
| Tipos no Shared Kernel | ≤ 8 |
| Aggregate size | ≤ 7 campos diretos cada |
| Domain Events | ≥ 15 tipos definidos |
| Value Objects | ≥ 10 tipos |

---

## Entregáveis Finais

| Artefato | Descrição |
|----------|-----------|
| Código Java 25 | MerchantHub completo com 6 módulos |
| Código Go 1.26 | MerchantHub completo com 6 módulos |
| `GLOSSARY.md` | 48+ termos com homônimos |
| `CONTEXT-MAP.md` | Diagrama + tabela de relações |
| `ARCHITECTURE.md` | C4 com 3 níveis |
| `SUBDOMAINS.md` | Classificação e investimento |
| `ADRs/` | 5+ decisões documentadas |
| Testes unitários | ≥ 95% cobertura no domínio |
| Testes de integração | Fluxo E2E cross-module |
| Testes de arquitetura | ArchUnit / import checks |
| `README.md` | Instruções de build, execução e navegação |

---

## O que você vai exercitar

| Disciplina | O que pratica |
|-----------|---------------|
| **DDD Estratégico** | Bounded Contexts, Subdomains, Context Map aplicados |
| **DDD Tático** | Entities, VOs, Aggregates, Events, Repos, Services, Factories |
| **Context Mapping** | ACL, OHS/PL, C-S, Partnership implementados entre módulos |
| **Qualidade** | Zero anti-patterns, testes de arquitetura, ADRs |
| **Arquitetura** | Hexagonal, Clean, CQRS, Event Sourcing, Modular Monolith |
| **Integração** | Event Bus, comunicação assíncrona, eventual consistency |
| **Documentação** | C4, glossário, ADRs, context map como artefatos vivos |

---

## Conclusão

Parabéns! Ao completar este capstone, você terá:

1. **Modelado** um domínio complexo usando DDD estratégico e tático
2. **Implementado** 6 Bounded Contexts com arquiteturas adequadas a cada contexto
3. **Integrado** módulos usando padrões de Context Mapping sem acoplamento
4. **Validado** a ausência de anti-patterns com testes automatizados
5. **Documentado** decisões arquiteturais com ADRs e diagramas C4
6. **Demonstrado** fluência em DDD em duas linguagens (Java 25 + Go 1.26)

→ Voltar ao [README — Mapa de Níveis](README.md)
