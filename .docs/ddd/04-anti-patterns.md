# Domain-Driven Design — Anti-Patterns & Code Smells

> **Objetivo deste documento:** Servir como referência completa sobre **anti-patterns e code smells** relacionados a DDD, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Abordagem **agnóstica de linguagem** — foca em diagnóstico, sintomas, impactos e correções com **pseudocódigo ilustrativo**.

---

## Quick Reference — Anti-Patterns por Severidade

| Severidade | Anti-pattern | Impacto | Dificuldade de correção |
|-----------|-------------|---------|------------------------|
| **Crítico** | Anemic Domain Model | Lógica espalhada, sem encapsulamento | Alta |
| **Crítico** | Big Ball of Mud | Sem fronteiras, tudo acoplado | Muito alta |
| **Crítico** | Shared Database | Contextos acoplados via banco | Alta |
| **Alto** | God Aggregate | Performance, concorrência, complexidade | Média |
| **Alto** | Domain Logic in Application Layer | Regras duplicadas, difícil de testar | Média |
| **Alto** | Primitive Obsession | Validação ausente, conceitos implícitos | Média |
| **Médio** | CRUD Masquerading as DDD | Over-engineering em domínio simples | Baixa |
| **Médio** | Over-engineering Generic Subdomains | Esforço desnecessário | Baixa |
| **Médio** | Repository per Entity | Fronteiras de Aggregate violadas | Média |
| **Baixo** | Technical Module Organization | Navegação difícil | Baixa |

---

## Sumário

- [Domain-Driven Design — Anti-Patterns \& Code Smells](#domain-driven-design--anti-patterns--code-smells)
  - [Quick Reference — Anti-Patterns por Severidade](#quick-reference--anti-patterns-por-severidade)
  - [Sumário](#sumário)
  - [Anti-Patterns Estratégicos](#anti-patterns-estratégicos)
    - [Big Ball of Mud](#big-ball-of-mud)
    - [Ambiguous Bounded Contexts](#ambiguous-bounded-contexts)
    - [Shared Database](#shared-database)
    - [Over-engineering Generic Subdomains](#over-engineering-generic-subdomains)
    - [Linguistic Chaos](#linguistic-chaos)
  - [Anti-Patterns Táticos](#anti-patterns-táticos)
    - [Anemic Domain Model](#anemic-domain-model)
    - [God Aggregate](#god-aggregate)
    - [Primitive Obsession](#primitive-obsession)
    - [Domain Logic in Application Layer](#domain-logic-in-application-layer)
    - [Repository per Entity](#repository-per-entity)
    - [Smart Entity (Infrastructure Leak)](#smart-entity-infrastructure-leak)
    - [Event Sourcing Everywhere](#event-sourcing-everywhere)
    - [CRUD Masquerading as DDD](#crud-masquerading-as-ddd)
  - [Code Smells que indicam problemas de DDD](#code-smells-que-indicam-problemas-de-ddd)
  - [Checklist de revisão](#checklist-de-revisão)
  - [Receitas de refatoração](#receitas-de-refatoração)
  - [Referências](#referências)

---

## Anti-Patterns Estratégicos

### Big Ball of Mud

> Sistema sem fronteiras claras entre contextos, onde tudo acessa tudo.

**Sintomas:**
- Um único modelo de dados usado por toda a aplicação
- Mudanças em uma área quebram funcionalidades não relacionadas
- Não é possível identificar onde começa/termina cada funcionalidade
- `God classes` com milhares de linhas
- Schema de banco de dados com centenas de tabelas sem separação

**Impacto:**
- Impossível fazer mudanças com confiança
- Onboarding de novos devs leva meses
- Deploy é all-or-nothing
- Testes são frágeis e lentos

**Correção progressiva:**

```
1. IDENTIFICAR SEAMS
   └── Pontos naturais de divisão no código (linguagem diferente, equipes diferentes)

2. ISOLAR COM ACL
   └── Criar camada de proteção entre o mud e novos contextos

3. STRANGLER FIG PATTERN
   └── Substituir funcionalidades gradualmente:
       ┌──────────────────┐
       │  Nova Feature A  │──┐
       └──────────────────┘  │    ┌─────────────┐
                             ├───►│  Big Ball   │
       ┌──────────────────┐  │    │  of Mud     │
       │  Nova Feature B  │──┘    │  (shrinking)│
       └──────────────────┘       └─────────────┘

4. BUBBLE CONTEXT
   └── Criar "bolhas" de DDD limpo dentro do legado
       └── ACL protege a bolha do mud
```

---

### Ambiguous Bounded Contexts

> Fronteiras de contexto não são claras, gerando sobreposição e conflito.

**Sintomas:**
- Duas equipes editam os mesmos arquivos frequentemente
- O mesmo conceito (ex: `Product`) tem significados diferentes mas usa a mesma classe
- Debates infinitos sobre "onde colocar" uma funcionalidade
- Merges conflitantes constantes

**Exemplo: ANTES (ambíguo)**
```
// Mesma classe Product usada em Catalog, Inventory, e Shipping
class Product:
    id: ProductId
    name: String
    description: String    // ← Catalog precisa
    images: List<Image>    // ← Catalog precisa
    stockQuantity: Integer // ← Inventory precisa
    weight: Decimal        // ← Shipping precisa
    dimensions: Dimensions // ← Shipping precisa
    price: Money           // ← Catalog + Sales
    costPrice: Money       // ← Finance precisa

    // Métodos de todos os contextos misturados
    updateDescription(...)    // Catalog
    reserveStock(quantity)   // Inventory  
    calculateShippingCost()  // Shipping
    calculateMargin()        // Finance
```

**DEPOIS (contextos separados)**
```
// Bounded Context: Catalog
class CatalogProduct:
    id: ProductId
    name: String
    description: String
    images: List<Image>
    price: Money
    updateDescription(...)

// Bounded Context: Inventory
class InventoryItem:
    productId: ProductId   // referência por ID
    stockQuantity: Quantity
    reserveStock(quantity)
    releaseStock(quantity)

// Bounded Context: Shipping
class ShippableItem:
    productId: ProductId   // referência por ID
    weight: Weight
    dimensions: Dimensions
    calculateShippingCost()
```

**Correção:**
1. Identifique onde a **linguagem diverge** (termos diferentes para "mesma" entidade)
2. Separe modelos por contexto, cada um com apenas os atributos relevantes
3. Comunique via IDs e eventos, não compartilhando objetos

---

### Shared Database

> Múltiplos Bounded Contexts acessam as mesmas tabelas do banco de dados.

**Sintomas:**
- Consultas JOIN entre tabelas de domínios diferentes
- Uma migração de schema pode quebrar múltiplos serviços
- Não é possível escalar ou deployar contextos independentemente
- Stored procedures com lógica de múltiplos domínios

**Impacto:**
- Acoplamento forte via schema compartilhado
- Impossível evoluir modelos independentemente
- Lock contention entre contextos
- Impossível migrar para microservices

**Correção:**
```
ANTES:
┌──────┐  ┌──────┐  ┌──────┐
│Sales │  │Invent│  │Shipp.│
└──┬───┘  └──┬───┘  └──┬───┘
   │         │         │
   └─────────┴─────────┘
             │
     ┌───────▼───────┐
     │  SHARED DB    │
     │  (100 tabelas)│
     └───────────────┘

DEPOIS:
┌──────┐  ┌──────┐  ┌──────┐
│Sales │  │Invent│  │Shipp.│
└──┬───┘  └──┬───┘  └──┬───┘
   │         │         │
┌──▼──┐  ┌──▼──┐  ┌──▼────┐
│Sales│  │Inv. │  │Ship.  │
│ DB  │  │ DB  │  │ DB    │
└─────┘  └─────┘  └───────┘

Comunicação via Domain Events (eventual consistency)
```

---

### Over-engineering Generic Subdomains

> Aplicar DDD tático completo (Aggregates, Domain Events, CQRS) em subdomínios genéricos.

**Sintomas:**
- CRUD simples embrulhado em 15 classes (Entity, VO, Repository, Factory, Service, Event...)
- Cadastro de usuários com Event Sourcing
- Gestão de configurações com Aggregates e Domain Events
- Semanas de esforço em funcionalidade que poderia ser resolvida com biblioteca pronta

**Regra:**

| Subdomain | Abordagem recomendada |
|----------|----------------------|
| **Core** | DDD tático completo; investimento máximo |
| **Supporting** | DDD simplificado; Aggregates quando faz sentido |
| **Generic** | CRUD, bibliotecas prontas, SaaS |

**Correção:**
- Antes de aplicar DDD tático, pergunte: "Este subdomínio é o diferencial competitivo do negócio?"
- Se não → simplifique. CRUD, Active Record, ou solução pronta são válidos.

---

### Linguistic Chaos

> Termos do domínio são inconsistentes, ambíguos ou inexistentes no código.

**Sintomas:**
- Mesmo conceito com nomes diferentes em partes do código (`Client`, `Customer`, `User`, `Account`)
- Termos técnicos no domínio (`DataHandler`, `StringProcessor`, `EntityManager`)
- Siglas e abreviações obscuras (`OrdMgr`, `PmtSvc`, `CustRec`)
- Comentários explicando "o que o nome realmente significa"
- Desenvolvedores e stakeholders usam vocabulários diferentes

**Correção:**
1. Criar **glossário do domínio** (documento vivo)
2. **Renomear** classes, métodos e variáveis para linguagem ubíqua
3. Usar o glossário em code reviews como referência
4. Alinhar em **Event Storming** ou sessões com domain experts

---

## Anti-Patterns Táticos

### Anemic Domain Model

> *"O anti-pattern mais comum e mais prejudicial em DDD."* — Martin Fowler

Entities e Aggregates são meros **containers de dados** (DTOs), sem comportamento de negócio. Toda lógica está em **Service classes** procedurais.

**Sintomas:**
- Entities com apenas getters/setters
- `setStatus()`, `setAmount()` como API pública
- Services fazem toda validação e transformação
- Entities podem existir em estado inválido
- Testes de Entity são triviais (só testam get/set)

**Exemplo: ANTES (anêmico)**
```
class Account:
    id: AccountId
    balance: Decimal
    status: String
    
    getBalance(): Decimal
    setBalance(balance: Decimal)
    getStatus(): String
    setStatus(status: String)

class AccountService:
    withdraw(accountId, amount):
        account = repository.findById(accountId)
        if account.getStatus() != "ACTIVE":
            throw "Account is not active"
        if account.getBalance() < amount:
            throw "Insufficient funds"
        account.setBalance(account.getBalance() - amount)
        repository.save(account)
```

**DEPOIS (modelo rico)**
```
class Account:
    id: AccountId
    balance: Money              // Value Object, não primitivo
    status: AccountStatus       // Value Object (enum rico)

    withdraw(amount: Money):
        require this.status.isActive(), "Account is not active"
        require this.balance.isGreaterOrEqual(amount), "Insufficient funds"
        this.balance = this.balance.subtract(amount)
        this.registerEvent(AmountWithdrawn(this.id, amount, now()))

    deposit(amount: Money):
        require this.status.isActive()
        require amount.isPositive(), "Deposit must be positive"
        this.balance = this.balance.add(amount)
        this.registerEvent(AmountDeposited(this.id, amount, now()))

    close():
        require this.balance.isZero(), "Cannot close account with balance"
        this.status = AccountStatus.CLOSED
        this.registerEvent(AccountClosed(this.id, now()))
```

**Como detectar:**
- Se você pode deletar todas as Entities e o sistema ainda funciona (via Services), seu modelo é anêmico
- Se Entities não têm testes significativos de lógica de negócio, provavelmente são anêmicas

---

### God Aggregate

> Aggregate que engloba muitas entidades e cresce sem limite.

**Sintomas:**
- Aggregate com 5+ entidades internas
- Carregamento do Aggregate é lento (muitos dados)
- Conflitos de concorrência frequentes
- Transações demoradas e locks extensos
- Mudanças em qualquer parte exigem carregar tudo

**Exemplo: ANTES (God Aggregate)**
```
class Order:                      // Aggregate Root
    items: List<OrderItem>         // OK
    payments: List<Payment>        // ← Deveria ser Aggregate separado
    shipments: List<Shipment>      // ← Deveria ser Aggregate separado
    invoices: List<Invoice>        // ← Deveria ser Aggregate separado
    reviews: List<Review>          // ← Deveria ser Aggregate separado
    auditLog: List<AuditEntry>     // ← Deveria ser separado
```

**DEPOIS (Aggregates menores)**
```
class Order:                      // Aggregate 1
    items: List<OrderItem>
    // referencia outros por ID

class Payment:                    // Aggregate 2
    orderId: OrderId              // referência por ID
    amount: Money
    status: PaymentStatus

class Shipment:                   // Aggregate 3
    orderId: OrderId              // referência por ID
    trackingCode: TrackingCode
    status: ShipmentStatus

// Comunicação via Domain Events:
// OrderConfirmed → cria Payment
// PaymentReceived → cria Shipment
```

**Regra de ouro:**
> *"Prefira Aggregates pequenos. Inclua na fronteira apenas o que é necessário para proteger invariantes transacionais imediatas."*
> — Vaughn Vernon

---

### Primitive Obsession

> Conceitos de domínio representados por tipos primitivos.

**Sintomas:**
- `string email`, `string cpf`, `decimal money`, `int quantity`
- Validação duplicada em múltiplos lugares
- Bugs por confusão de tipos (passar `orderId` onde era `customerId`)
- Formatação/conversão espalhada

**ANTES (primitivos)**
```
class Customer:
    name: String
    email: String          // ← pode ser inválido
    cpf: String            // ← pode ser "banana"
    phone: String          // ← formato?
    creditLimit: Decimal   // ← moeda?
```

**DEPOIS (Value Objects)**
```
class Customer:
    name: PersonName       // valida comprimento, caracteres
    email: EmailAddress    // valida formato, normaliza
    cpf: CPF               // valida dígitos verificadores
    phone: PhoneNumber     // valida formato, DDD
    creditLimit: Money     // valor + moeda, operações seguras
```

---

### Domain Logic in Application Layer

> Regras de negócio implementadas em Application Services em vez de no domínio.

**Sintomas:**
- Application Service com `if/else` de regras de negócio
- Domain objects são apenas passados como parâmetros, sem decidir nada
- Regras duplicadas entre múltiplos Application Services
- Application Service com centenas de linhas

**Diferença correta:**

| Camada | Responsabilidade | Exemplo |
|--------|-----------------|---------|
| **Application Service** | Orquestração, transação, segurança | Buscar Aggregate, chamar método de domínio, salvar |
| **Domain (Entity/Service)** | Regras de negócio, invariantes | Validar, calcular, decidir, publicar evento |

```
// ═══ RUIM — Application Service com lógica de negócio ═══
class PlaceOrderUseCase:
    execute(command):
        customer = customerRepo.findById(command.customerId)
        
        // Lógica de negócio NÃO deveria estar aqui
        if customer.status != "ACTIVE":
            throw "Customer is not active"
        if customer.creditLimit < command.totalAmount:
            throw "Exceeds credit limit"
        if command.items.isEmpty():
            throw "Order must have items"
        
        order = new Order()
        order.setCustomerId(command.customerId)
        order.setItems(command.items)
        order.setStatus("CREATED")
        orderRepo.save(order)


// ═══ BOM — Application Service orquestra, Domínio decide ═══
class PlaceOrderUseCase:
    execute(command):
        customer = customerRepo.findById(command.customerId)
            ?? throw CustomerNotFound(command.customerId)

        // Domínio decide
        order = customer.placeOrder(command.items, command.shippingAddress)

        orderRepo.save(order)
        eventPublisher.publishAll(order.domainEvents)
```

---

### Repository per Entity

> Criar Repositories para entidades internas do Aggregate, violando fronteiras.

**Sintomas:**
- `OrderItemRepository`, `AddressRepository` quando são internos de Aggregates
- Entidades internas carregadas/salvas independentemente da root
- Não é claro quem é o Aggregate Root

**Correção:**
- Um **Repository por Aggregate Root**
- Entidades internas são carregadas/salvas junto com a root
- Acesso externo sempre via Aggregate Root

---

### Smart Entity (Infrastructure Leak)

> Entity que contém lógica de infraestrutura (acesso a banco, HTTP, filesystem).

**Sintomas:**
- Entity que faz queries no banco dentro de métodos de negócio
- Entity que chama APIs HTTP
- Entity que lê configurações de arquivo
- Testes de Entity requerem infraestrutura real

```
// ═══ RUIM — Entity com infraestrutura ═══
class Order:
    calculateDiscount():
        // Entity acessando banco ← VIOLAÇÃO
        customerHistory = database.query(
            "SELECT count(*) FROM orders WHERE customer_id = ?", this.customerId)
        if customerHistory > 10:
            return this.totalAmount() * 0.1
        return 0

// ═══ BOM — Entity pura + Domain Service para regra cross-entity ═══
class DiscountPolicy:
    customerRepository: CustomerRepository

    calculateDiscount(order: Order): Money
        customer = customerRepository.findById(order.customerId)
        if customer.isLoyalCustomer():
            return order.totalAmount().multiply(0.10)
        return Money.ZERO
```

---

### Event Sourcing Everywhere

> Aplicar Event Sourcing em todos os contextos, mesmo quando não é necessário.

**Quando Event Sourcing FAZ sentido:**
- Audit trail é requisito de negócio
- Time-travel (ver estado em qualquer ponto no tempo)
- Core domain com regras complexas que evoluem
- Integração via eventos é o padrão primário

**Quando NÃO faz sentido:**
- CRUD simples (cadastros, configurações)
- Generic subdomains
- Quando a equipe não tem experiência com o pattern
- Quando queries simples são a operação primária

---

### CRUD Masquerading as DDD

> Aplicação CRUD simples disfarçada com Aggregates, Domain Events e Repositories sem necessidade.

**Sintomas:**
- Aggregates que são apenas CRUD sem invariantes
- Domain Events que são apenas `EntityCreated`, `EntityUpdated`, `EntityDeleted`
- Toda operação é uma simples leitura/escrita sem regras
- Overhead de abstrações sem benefício

**Regra:** Se o subdomínio não tem regras de negócio complexas, **CRUD é a escolha correta**. DDD tático só vale para **Core Domain** com complexidade genuína.

---

## Code Smells que indicam problemas de DDD

| # | Code Smell | Anti-pattern provável | Ação |
|---|-----------|----------------------|------|
| 1 | Entity com apenas getters/setters | Anemic Domain Model | Mover lógica para Entity |
| 2 | `setStatus()` como método público | Anemic Domain Model | Criar métodos com intenção de negócio |
| 3 | `String email`, `BigDecimal amount` | Primitive Obsession | Criar Value Objects |
| 4 | Service com 500+ linhas de if/else de negócio | Domain Logic in App Layer | Mover para Entity/Domain Service |
| 5 | `OrderItemRepository` | Repository per Entity | Acessar via Aggregate Root |
| 6 | Entity que faz query no banco | Smart Entity | Extrair para Domain Service |
| 7 | Mesma classe `Product` usada em 3 módulos | Ambiguous Bounded Context | Separar modelos por contexto |
| 8 | Classe com `Manager`, `Handler`, `Processor` no nome | Linguistic Chaos | Renomear com linguagem ubíqua |
| 9 | JOIN entre tabelas de módulos diferentes | Shared Database | Separar storage por contexto |
| 10 | Aggregate carrega 1000+ registros | God Aggregate | Dividir em Aggregates menores |
| 11 | Value Object com setter | Mutable VO | Tornar imutável |
| 12 | Testes de domínio requerem banco de dados | Infrastructure Leak / DIP | Abstrair dependências de infra |

---

## Checklist de revisão

Use este checklist ao revisar código que usa DDD:

### Estratégico
- [ ] Bounded Contexts estão claramente definidos?
- [ ] Context Map está documentado?
- [ ] Subdomains estão classificados (Core/Supporting/Generic)?
- [ ] Linguagem ubíqua está refletida no código?
- [ ] Cada contexto tem seu próprio modelo de dados?
- [ ] Contextos se comunicam por contratos (API/Events), não por DB compartilhado?

### Tático
- [ ] Entities têm comportamento rico (não são anêmicas)?
- [ ] Value Objects são imutáveis e auto-validantes?
- [ ] Aggregates são pequenos e protegem invariantes?
- [ ] Referências entre Aggregates são por ID?
- [ ] Um Repository por Aggregate Root?
- [ ] Domain Events nomeados como fatos de negócio no passado?
- [ ] Domain Services são stateless e encapsulam regras cross-entity?
- [ ] Interface do Repository está na camada de domínio?
- [ ] Camada de domínio não depende de infraestrutura?
- [ ] Application Service apenas orquestra, não contém regras de negócio?

---

## Receitas de refatoração

### Receita 1: Anemic → Rich Domain Model

```
PASSO 1: Identificar Services com lógica de negócio sobre uma Entity
PASSO 2: Para cada regra, perguntar "a quem pertence naturalmente?"
PASSO 3: Mover método para a Entity/Aggregate
PASSO 4: Tornar setters privados; criar métodos com intenção
PASSO 5: Adicionar validação no construtor (never invalid)
PASSO 6: Escrever testes de domínio (sem infraestrutura)
```

### Receita 2: God Aggregate → Aggregates menores

```
PASSO 1: Listar todas as invariantes do Aggregate
PASSO 2: Agrupar invariantes que DEVEM ser transacionais
PASSO 3: Cada grupo = um Aggregate candidato
PASSO 4: Substituir referências por objeto → referências por ID
PASSO 5: Usar Domain Events para consistência entre Aggregates
PASSO 6: Criar Repository para cada novo Aggregate Root
```

### Receita 3: Primitivos → Value Objects

```
PASSO 1: Identificar primitivos que representam conceitos de domínio
PASSO 2: Criar classe imutável com validação no construtor
PASSO 3: Implementar igualdade por valor
PASSO 4: Adicionar métodos de domínio relevantes (format, compare, etc.)
PASSO 5: Substituir uso de primitivo no código
PASSO 6: Remover validações duplicadas (agora estão no VO)
```

### Receita 4: App Service Fat → Domain-driven

```
PASSO 1: Identificar if/else de regras de negócio no Application Service
PASSO 2: Mover validações para Entity/Aggregate (preconditions)
PASSO 3: Mover cálculos para Entity, VO ou Domain Service
PASSO 4: Application Service fica apenas com: buscar, chamar domínio, salvar
PASSO 5: Testes de regra testam Entity/Domain Service diretamente
```

---

## Referências

| Recurso | Autor | Tipo |
|---------|-------|------|
| *Domain-Driven Design: Tackling Complexity in the Heart of Software* | Eric Evans | Livro |
| *Implementing Domain-Driven Design* | Vaughn Vernon | Livro |
| *AnemicDomainModel* | Martin Fowler | Artigo/bliki |
| *Effective Aggregate Design* (3-part series) | Vaughn Vernon | Paper |
| *Learning Domain-Driven Design* | Vlad Khononov | Livro |
| *Refactoring: Improving the Design of Existing Code* | Martin Fowler | Livro |
