# Zero to Hero — Domain-Driven Design Challenge (Java 25 + Go 1.26)

> **Programa de especialização progressiva** em Domain-Driven Design, implementado com
> **Java 25** (sem Spring, Quarkus, Micronaut) · **Go 1.26** (sem Gin, Echo, Fiber)

---

## 1. Resumo Executivo

Este programa transforma um desenvolvedor em **especialista em Domain-Driven Design** através de desafios práticos que cobrem todos os pilares do DDD — desde **Design Estratégico** e **Design Tático** até **Context Mapping**, **Anti-Patterns** e **Integração Arquitetural**. Implementado em **Java 25 e Go 1.26 puros** — sem nenhum framework.

**Domínio escolhido:** **Plataforma de E-commerce Multi-Tenant (MerchantHub)** — um domínio rico com múltiplos Bounded Contexts, regras de negócio complexas e integrações entre contextos que permitem exercitar todo o espectro de DDD.

**Por que MerchantHub (E-commerce Multi-Tenant)?**
- **Múltiplos Bounded Contexts naturais** — Catalog, Sales, Inventory, Shipping, Billing, Identity — cada um com sua linguagem ubíqua
- **Core Domain claro** — Motor de pricing/promoções e gestão de pedidos com regras ricas
- **Aggregates naturais** — Order (com OrderItems), Product (com Variants), Cart (com CartItems)
- **Domain Events orgânicos** — `OrderPlaced`, `PaymentReceived`, `ShipmentDispatched`, `StockReserved`
- **Context Mapping real** — Catalog ↔ Sales (Customer-Supplier), Billing ↔ Payment Gateway (ACL), Sales ↔ Inventory (Domain Events)
- **Anti-patterns evidentes** — modelo `Product` compartilhado entre contextos, entidades anêmicas, God Aggregates
- **Arquitetura evolutiva** — de Modular Monolith a Microservices com CQRS e Event Sourcing
- Perguntas de entrevista Senior/Staff frequentemente envolvem design de sistemas de e-commerce

**Por que linguagem pura (sem frameworks)?**
- DDD é sobre **modelagem de domínio**, não sobre frameworks — implementar sem frameworks força compreensão profunda
- Frameworks escondem building blocks — você usa Repository via `@Repository`, mas não sabe implementar o padrão
- Java 25 oferece **records** (Value Objects naturais), **sealed classes** (Aggregates controlados), **pattern matching**
- Go 1.26 força **composição**, **interfaces implícitas**, **error handling explícito** — visão pragmática dos building blocks
- O contraste Java vs Go revela **quando um padrão é essencial** e quando é artefato da linguagem

---

## 2. Arquitetura do Domínio MerchantHub

```
┌─────────────────────────────────────────────────────────┐
│                  MERCHANTHUB DOMAIN                       │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   CATALOG    │  │    SALES     │  │  SHIPPING    │  │
│  │   (Core)     │  │   (Core)     │  │ (Supporting) │  │
│  │              │  │              │  │              │  │
│  │ • Product    │  │ • Order      │  │ • Shipment   │  │
│  │ • Category   │  │ • Cart       │  │ • Package    │  │
│  │ • Pricing    │  │ • Discount   │  │ • Route      │  │
│  │ • Review     │  │ • Promotion  │  │ • Tracking   │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                 │           │
│         └────── Events ───┴──── Events ─────┘           │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  INVENTORY   │  │   BILLING    │  │   IDENTITY   │  │
│  │ (Supporting) │  │ (Supporting) │  │  (Generic)   │  │
│  │              │  │              │  │              │  │
│  │ • Stock      │  │ • Invoice    │  │ • Account    │  │
│  │ • Warehouse  │  │ • Tax        │  │ • Auth       │  │
│  │ • Allocation │  │ • Refund     │  │ • Profile    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Entidades Principais por Contexto

| Bounded Context | Entidades/Aggregates | Subdomain |
|----------------|---------------------|-----------|
| **Catalog** | `CatalogProduct`, `Category`, `PricingRule`, `Review` | Core |
| **Sales** | `Order`, `Cart`, `Discount`, `Promotion` | Core |
| **Inventory** | `InventoryItem`, `Warehouse`, `StockAllocation` | Supporting |
| **Shipping** | `Shipment`, `Package`, `Route`, `TrackingEvent` | Supporting |
| **Billing** | `Invoice`, `Payment`, `Refund`, `TaxCalculation` | Supporting |
| **Identity** | `Account`, `Profile`, `Credential` | Generic |

---

## 3. Mapeamento DDD → Domínio MerchantHub

| Conceito DDD | Aplicação no MerchantHub |
|-------------|-------------------------|
| **Ubiquitous Language** | Glossário por contexto — `Product` no Catalog ≠ `InventoryItem` no Inventory |
| **Bounded Context** | 6 contextos com modelos independentes e linguagem própria |
| **Subdomains** | Core (Catalog, Sales), Supporting (Inventory, Shipping, Billing), Generic (Identity) |
| **Context Map** | Catalog→Sales (C/S), Sales→Inventory (Events), Billing→PaymentGateway (ACL) |
| **Entity** | `Order` (identidade + behavior), `CatalogProduct` (lifecycle) |
| **Value Object** | `Money`, `Address`, `EmailAddress`, `OrderId`, `SKU`, `Quantity` |
| **Aggregate** | `Order` (root) + `OrderItem` (interno), `CatalogProduct` (root) + `ProductVariant` (interno) |
| **Domain Event** | `OrderPlaced`, `PaymentReceived`, `StockReserved`, `ShipmentDispatched` |
| **Repository** | `OrderRepository`, `CatalogProductRepository` — um por Aggregate Root |
| **Domain Service** | `PricingService` (cálculo cross-entity), `DiscountPolicy`, `TaxCalculationService` |
| **Factory** | `OrderFactory` (criação complexa com validação de estoque + preço) |
| **ACL** | `PaymentGatewayACL` (traduz modelo externo de pagamento para domínio interno) |
| **CQRS** | Command: processar pedido / Query: catálogo, histórico de pedidos |
| **Event Sourcing** | Histórico completo de eventos do Order Aggregate |
| **Hexagonal** | Ports (interfaces no domínio) + Adapters (infra externa) |
| **Modular Monolith** | 6 módulos/Bounded Contexts no mesmo deployment |

---

## 4. Equivalências Java 25 ↔ Go 1.26

| Conceito DDD | Java 25 | Go 1.26 |
|-------------|---------|---------|
| **Value Object** | `record` (imutável, equals por valor) | `struct` com factory + métodos |
| **Entity** | `class` com ID + métodos de domínio | `struct` + pointer receivers |
| **Aggregate Root** | `class` com invariantes protegidas | `struct` + métodos que validam |
| **Domain Event** | `sealed interface` + `record` por evento | `interface` + structs tipados |
| **Repository** | `interface` no domínio, `class` na infra | `interface` (duck typing) |
| **Domain Service** | `interface` + implementação stateless | `func` ou `struct` sem estado |
| **Factory** | `static` method ou classe dedicada | Constructor function (`NewXxx`) |
| **Module** | `package` por contexto | `package` por contexto |
| **Sealed types** | `sealed interface permits` | Type switch por convenção |
| **Pattern matching** | `switch` com patterns | `switch v := x.(type)` |
| **Imutabilidade** | `record`, `final`, `Collections.unmodifiable*` | Retorno por valor, sem ponteiros |
| **Encapsulamento** | `private` + API pública mínima | Campos lowercase (não exportados) |

---

## 5. Mapa de Níveis

```
Level 0 — Fundações DDD & Linguagem Ubíqua
  │       (Conceitos base, glossário, Value Objects fundamentais)
  ▼
Level 1 — Strategic Design
  │       (Bounded Contexts, Subdomains, Context Discovery)
  ▼
Level 2 — Tactical Design (Building Blocks)
  │       (Entities, Value Objects, Aggregates, Domain Events, Repositories, Services, Factories)
  ▼
Level 3 — Context Mapping
  │       (Partnership, Shared Kernel, Customer-Supplier, ACL, OHS, Published Language)
  ▼
Level 4 — Anti-Patterns & Refactoring
  │       (Anemic Model, God Aggregate, Primitive Obsession, Shared Database, receitas de refatoração)
  ▼
Level 5 — Arquitetura & Integração
  │       (Hexagonal, Clean Architecture, CQRS, Event Sourcing, Modular Monolith, Microservices)
  ▼
Level 6 — Projeto Capstone: MerchantHub Completo
          (Todos os conceitos DDD integrados em uma plataforma e-commerce coesa)
```

---

## 6. Estrutura de Cada Desafio

Cada nível segue a mesma estrutura:

```
level-N-nome/
├── README.md                ← Desafio detalhado com instruções
├── java/                    ← Implementação Java 25 pura
│   ├── src/
│   │   ├── main/java/
│   │   └── test/java/
│   └── build.gradle
└── go/                      ← Implementação Go 1.26 pura
    ├── cmd/
    ├── internal/
    ├── go.mod
    └── *_test.go
```

### Dependências Permitidas

| Categoria | Java 25 | Go 1.26 |
|-----------|---------|---------|
| **Linguagem** | JDK 25 (standard library) | Go 1.26 (standard library) |
| **Testes** | JUnit 5, AssertJ, Mockito | `testing`, `testify` |
| **Build** | Gradle 8.x ou Maven 3.9+ | `go build` / `go test` |
| **Serialização** | Jackson (apenas para I/O) | `encoding/json` (stdlib) |
| **Logging** | `java.util.logging` ou SLF4J + Logback | `log/slog` (stdlib) |
| **HTTP** | `java.net.http.HttpClient` (stdlib) | `net/http` (stdlib) |
| **Frameworks** | **PROIBIDO** (sem Spring, Quarkus, Micronaut, Jakarta EE) | **PROIBIDO** (sem Gin, Echo, Fiber, Chi) |
| **DI containers** | **PROIBIDO** — DI manual via construtor | **PROIBIDO** — DI manual via construtor/funções |

---

## 7. Critérios Globais

### Qualidade de Código

| Critério | Java 25 | Go 1.26 |
|----------|---------|---------|
| **Idiomático** | Records, sealed, pattern matching, streams | Interfaces implícitas, embedding, error handling |
| **Testes** | ≥ 85% cobertura (JaCoCo) | ≥ 85% cobertura (`go test -cover`) |
| **Documentação** | Javadoc nos tipos públicos | GoDoc nos tipos exportados |
| **Linting** | Checkstyle + PMD ou SonarLint | `golangci-lint` |
| **Naming** | Ubiquitous Language no código | Ubiquitous Language no código |
| **Error handling** | Checked exceptions + `Optional<T>` | `(T, error)` pattern + `errors.Is/As` |

### Documentação Esperada

Cada nível deve conter:
- `README.md` — Descrição do desafio com critérios de aceite
- `DECISIONS.md` — Trade-offs e justificativas de design
- `GLOSSARY.md` — Glossário da Ubiquitous Language por Bounded Context
- `CONTEXT-MAP.md` — Mapa de relações entre contextos (quando aplicável)
- Diagramas (opcional mas recomendado) — Mermaid ou PlantUML

---

## 8. Referência Cruzada com Documentação

| Nível | Documento de Referência |
|-------|------------------------|
| Level 0 | [README.md (DDD Overview)](../../.docs/ddd/README.md) |
| Level 1 | [01-strategic-design.md](../../.docs/ddd/01-strategic-design.md) |
| Level 2 | [02-tactical-design.md](../../.docs/ddd/02-tactical-design.md) |
| Level 3 | [03-context-mapping.md](../../.docs/ddd/03-context-mapping.md) |
| Level 4 | [04-anti-patterns.md](../../.docs/ddd/04-anti-patterns.md) |
| Level 5 | [05-architecture-integration.md](../../.docs/ddd/05-architecture-integration.md) |
| Level 6 | Todos os documentos acima integrados |

---

## 9. Navegação

| Nível | Título | Conceitos | Status |
|-------|--------|-----------|--------|
| [Level 0](00-ddd-foundations.md) | Fundações DDD & Linguagem Ubíqua | Domínio, Ubiquitous Language, glossário, VOs base | 🔲 |
| [Level 1](01-strategic-design.md) | Strategic Design | Bounded Contexts, Subdomains, Context Discovery | 🔲 |
| [Level 2](02-tactical-design.md) | Tactical Design (Building Blocks) | Entities, VOs, Aggregates, Events, Repos, Services, Factories | 🔲 |
| [Level 3](03-context-mapping.md) | Context Mapping | Partnership, Shared Kernel, C/S, ACL, OHS, PL, Separate Ways | 🔲 |
| [Level 4](04-anti-patterns.md) | Anti-Patterns & Refactoring | Anemic Model, God Aggregate, Primitive Obsession, refatoração | 🔲 |
| [Level 5](05-architecture-integration.md) | Arquitetura & Integração | Hexagonal, Clean, CQRS, Event Sourcing, Modular Monolith | 🔲 |
| [Level 6](06-capstone-merchanthub.md) | Projeto Capstone: MerchantHub | Todos os conceitos DDD em um sistema coeso | 🔲 |

---

## 10. Pré-requisitos

| Requisito | Java | Go |
|-----------|------|----|
| **Versão** | JDK 25 | Go 1.26 |
| **Build tool** | Gradle 8.x ou Maven 3.9+ | `go` toolchain |
| **IDE** | IntelliJ IDEA ou VS Code | GoLand ou VS Code (gopls) |
| **Git** | Git 2.x | Git 2.x |
| **Conhecimento prévio** | OOP, interfaces, collections | Structs, interfaces, packages |
| **Trilha anterior (recomendada)** | [Design Patterns Challenge](../design-patterns/README.md) | [Design Patterns Challenge](../design-patterns/README.md) |

---

> **Legenda de status:** 🔲 Não iniciado · 🔶 Em progresso · ✅ Completo
