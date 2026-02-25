# Domain-Driven Design — Tactical Design (Building Blocks)

> **Objetivo deste documento:** Servir como referência teórica completa sobre os **Building Blocks do DDD (Design Tático)**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Abordagem **agnóstica de linguagem** — foca em conceitos, motivações, heurísticas, boas práticas e **pseudocódigo ilustrativo** sem vínculo a tecnologia específica.

---

## Quick Reference — Cheat Sheet

| Building Block | Regra de ouro | Violação típica | Correção |
|---------------|--------------|------------------|----------|
| **Entity** | Identidade persiste ao longo do tempo | Entity sem comportamento (Anemic Model) | Mover lógica de negócio para a entidade |
| **Value Object** | Igualdade por valor, imutabilidade | Primitive Obsession — usar primitivos | Criar VO com validação interna |
| **Aggregate** | Fronteira de consistência transacional | Aggregate referencia outro por objeto | Referenciar por ID; um Aggregate por transação |
| **Domain Event** | Registrar fatos que ocorreram no domínio | Lógica distribuída em side-effects implícitos | Publicar eventos explícitos |
| **Repository** | Coleção de Aggregates com persistência | Repository com lógica de negócio | Repositório apenas persiste/recupera Aggregates |
| **Domain Service** | Lógica que não pertence a nenhuma entidade | Service com estado ou que substitui Entity | Service stateless para operações cross-entity |
| **Factory** | Encapsular criação complexa de objetos | Construtores com lógica de negócio complexa | Factory Method/Abstract Factory |

---

## Sumário

- [Domain-Driven Design — Tactical Design (Building Blocks)](#domain-driven-design--tactical-design-building-blocks)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Entities](#entities)
    - [Definição](#definição)
    - [Características](#características)
    - [Boas Práticas](#boas-práticas)
    - [Heurísticas para identificar violações](#heurísticas-para-identificar-violações)
    - [Exemplo (pseudocódigo)](#exemplo-pseudocódigo)
    - [Anti-patterns](#anti-patterns)
  - [Value Objects](#value-objects)
    - [Definição](#definição-1)
    - [Características](#características-1)
    - [Boas Práticas](#boas-práticas-1)
    - [Heurísticas para identificar candidatos a Value Object](#heurísticas-para-identificar-candidatos-a-value-object)
    - [Exemplo (pseudocódigo)](#exemplo-pseudocódigo-1)
    - [Anti-patterns](#anti-patterns-1)
  - [Aggregates](#aggregates)
    - [Definição](#definição-2)
    - [Regras Fundamentais](#regras-fundamentais)
    - [Design de Aggregates — Boas Práticas](#design-de-aggregates--boas-práticas)
    - [Heurísticas para identificar violações](#heurísticas-para-identificar-violações-1)
    - [Exemplo (pseudocódigo)](#exemplo-pseudocódigo-2)
    - [Anti-patterns](#anti-patterns-2)
  - [Domain Events](#domain-events)
    - [Definição](#definição-3)
    - [Características](#características-2)
    - [Boas Práticas](#boas-práticas-2)
    - [Heurísticas para identificar candidatos a Domain Event](#heurísticas-para-identificar-candidatos-a-domain-event)
    - [Exemplo (pseudocódigo)](#exemplo-pseudocódigo-3)
    - [Anti-patterns](#anti-patterns-3)
  - [Repositories](#repositories)
    - [Definição](#definição-4)
    - [Boas Práticas](#boas-práticas-3)
    - [Heurísticas para identificar violações](#heurísticas-para-identificar-violações-2)
    - [Exemplo (pseudocódigo)](#exemplo-pseudocódigo-4)
    - [Anti-patterns](#anti-patterns-4)
  - [Domain Services](#domain-services)
    - [Definição](#definição-5)
    - [Quando usar](#quando-usar)
    - [Boas Práticas](#boas-práticas-4)
    - [Exemplo (pseudocódigo)](#exemplo-pseudocódigo-5)
    - [Anti-patterns](#anti-patterns-5)
  - [Factories](#factories)
    - [Definição](#definição-6)
    - [Quando usar](#quando-usar-1)
    - [Boas Práticas](#boas-práticas-5)
    - [Exemplo (pseudocódigo)](#exemplo-pseudocódigo-6)
  - [Modules (Pacotes/Namespaces)](#modules-pacotesnamespaces)
    - [Boas Práticas](#boas-práticas-6)
  - [Relações entre Building Blocks](#relações-entre-building-blocks)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Entities

### Definição

> *"Um objeto que é definido não por seus atributos, mas por uma linha de continuidade e sua identidade."*
> — Eric Evans

Uma **Entity** é um objeto do domínio que possui **identidade única** que persiste ao longo do tempo, independentemente das mudanças em seus atributos.

### Características

| Característica | Descrição |
|---------------|-----------|
| **Identidade** | Tem um identificador único que a distingue de outras instâncias |
| **Continuidade** | Sua identidade persiste ao longo do ciclo de vida |
| **Mutabilidade controlada** | Pode mudar de estado, mas de forma controlada via métodos de domínio |
| **Igualdade por identidade** | Duas entities são iguais se têm o mesmo ID, independente dos atributos |
| **Comportamento rico** | Contém lógica de negócio que protege suas invariantes |

### Boas Práticas

1. **Encapsule estado** — Atributos devem ser privados. Mudanças acontecem através de métodos de domínio que expressam intenção de negócio.

2. **Proteja invariantes** — A Entity é responsável por garantir que suas regras de negócio nunca sejam violadas.

3. **Nomeie métodos com linguagem de domínio** — Use `order.cancel()`, não `order.setStatus("CANCELLED")`. Use `account.deposit(amount)`, não `account.setBalance(balance + amount)`.

4. **Evite setters anêmicos** — Setters para cada propriedade destroem encapsulamento. Se precisa alterar algo, crie um método que expresse a **intenção de negócio**.

5. **Validação no construtor** — Uma Entity nunca deve existir em estado inválido. Valide no momento da criação.

6. **Igualdade por ID** — Implemente igualdade baseada no identificador, não nos atributos.

7. **IDs como Value Objects** — Considere representar identificadores como Value Objects tipados para evitar confusão entre IDs de tipos diferentes.

### Heurísticas para identificar violações

| Sinal | O que indica |
|-------|-------------|
| Entity com apenas getters/setters e sem comportamento | Anemic Domain Model |
| Lógica de negócio em serviços que manipulam Entities | Lógica vazou da Entity |
| Entity que pode existir em estado inválido | Falta validação no construtor |
| `setStatus()`, `setState()` como métodos públicos | Falta encapsulamento; métodos devem expressar intenção |
| Entity sem identidade clara | Pode ser um Value Object |

### Exemplo (pseudocódigo)

**ANTES — Entity anêmica (anti-pattern):**

```
class Order:
    id: OrderId
    status: String
    items: List<OrderItem>
    totalAmount: Money

    // Apenas getters e setters
    getStatus(): String
    setStatus(status: String)
    getItems(): List<OrderItem>
    setItems(items: List<OrderItem>)
    getTotalAmount(): Money
    setTotalAmount(amount: Money)

// Lógica de negócio em serviço externo
class OrderService:
    cancelOrder(order):
        if order.getStatus() == "DELIVERED":
            throw "Cannot cancel delivered order"
        order.setStatus("CANCELLED")
        // recalculate refund...
```

**DEPOIS — Entity rica (correto):**

```
class Order:
    id: OrderId               // Value Object tipado
    status: OrderStatus        // Value Object (enum rico)
    items: List<OrderItem>
    
    // Construtor valida invariantes
    constructor(id, items):
        require items.isNotEmpty(), "Order must have at least one item"
        this.id = id
        this.items = items
        this.status = OrderStatus.CREATED

    // Comportamento de domínio com linguagem ubíqua
    cancel():
        require this.status.isCancellable(), 
            "Cannot cancel order in status " + this.status
        this.status = OrderStatus.CANCELLED
        this.registerEvent(OrderCancelled(this.id, now()))

    addItem(item: OrderItem):
        require this.status == OrderStatus.CREATED, 
            "Cannot modify confirmed order"
        require item not in this.items, "Duplicate item"
        this.items.add(item)

    confirm():
        require this.items.isNotEmpty(), "Cannot confirm empty order"
        this.status = OrderStatus.CONFIRMED
        this.registerEvent(OrderConfirmed(this.id, this.totalAmount(), now()))

    totalAmount(): Money
        return this.items.sum(item -> item.subtotal())

    // Igualdade por identidade
    equals(other): return this.id == other.id
    hashCode(): return this.id.hashCode()
```

### Anti-patterns

| Anti-pattern | Descrição | Correção |
|-------------|-----------|----------|
| **Anemic Domain Model** | Entity sem comportamento, apenas dados | Mover lógica de negócio dos Services para a Entity |
| **God Entity** | Entity com centenas de métodos e responsabilidades | Separar em Aggregates menores ou extrair Value Objects |
| **Identity Crisis** | Entity sem identidade clara ou com identidade composta frágil | Usar ID natural forte ou surrogate ID como VO |
| **Public State** | Todos os atributos são públicos ou têm setters | Encapsular e expor apenas métodos de domínio |

---

## Value Objects

### Definição

> *"Um objeto que descreve alguma característica ou atributo mas que não tem conceito de identidade."*
> — Eric Evans

Um **Value Object** é definido **pelo valor de seus atributos**, não por uma identidade. Dois Value Objects com os mesmos atributos são **iguais e intercambiáveis**.

### Características

| Característica | Descrição |
|---------------|-----------|
| **Sem identidade** | Não tem ID; igualdade é por valor dos atributos |
| **Imutável** | Uma vez criado, não muda. Para "alterar", crie uma nova instância |
| **Auto-validante** | Valida invariantes no construtor; nunca existe em estado inválido |
| **Side-effect free** | Métodos retornam novos objetos em vez de modificar estado |
| **Substituível** | Pode ser substituído por outro VO com os mesmos valores |

### Boas Práticas

1. **Prefira Value Objects a primitivos** — Em vez de `string email`, use `EmailAddress`. Em vez de `decimal amount`, use `Money`. Isso é chamado **"Primitive Obsession cure"**.

2. **Imutabilidade total** — Nenhum setter. Operações retornam novas instâncias:

```
// BOM — imutável
class Money:
    amount: Decimal
    currency: Currency

    add(other: Money): Money
        require this.currency == other.currency
        return new Money(this.amount + other.amount, this.currency)
```

3. **Validação no construtor** — Um VO inválido nunca deve existir:

```
class EmailAddress:
    value: String
    
    constructor(email: String):
        require email matches EMAIL_REGEX, "Invalid email format"
        require email.length <= 254, "Email too long"
        this.value = email.toLowerCase().trim()
```

4. **Igualdade estrutural** — Dois VOs são iguais se todos os atributos são iguais.

5. **Sem referência a Entities** — Value Objects não devem referenciar Entities. Se precisa, use o ID da Entity.

6. **Nomeie como conceitos de domínio** — `Money`, `Address`, `DateRange`, `Temperature`, não `AmountWrapper`, `StringValue`.

### Heurísticas para identificar candidatos a Value Object

| Sinal | Exemplo |
|-------|---------|
| Primitivos que sempre aparecem juntos | `amount` + `currency` → `Money` |
| Strings com formato/validação específica | Email, CPF, CNPJ, telefone → VOs tipados |
| Valores que são comparados por conteúdo, não identidade | Endereço, coordenada, período |
| Propriedade com regras de negócio próprias | Preço (não pode ser negativo), percentual (0-100) |
| Conceitos do domínio representados como primitivos | "Primitive Obsession" smell |
| Atributo com métodos de formatação/cálculo | Dinheiro (somar, converter), distância (converter unidades) |

### Exemplo (pseudocódigo)

```
// Value Object: Money
class Money:
    amount: Decimal   // imutável
    currency: Currency // imutável (também um VO)

    constructor(amount, currency):
        require amount >= 0, "Amount cannot be negative"
        require currency is not null, "Currency is required"
        this.amount = amount
        this.currency = currency

    add(other: Money): Money
        require this.currency == other.currency, 
            "Cannot add different currencies"
        return new Money(this.amount + other.amount, this.currency)

    subtract(other: Money): Money
        require this.currency == other.currency
        require this.amount >= other.amount, "Insufficient amount"
        return new Money(this.amount - other.amount, this.currency)

    multiply(factor: Decimal): Money
        return new Money(this.amount * factor, this.currency)

    isGreaterThan(other: Money): Boolean
        require this.currency == other.currency
        return this.amount > other.amount

    equals(other): Boolean
        return this.amount == other.amount 
            and this.currency == other.currency

    toString(): String
        return this.currency.symbol + " " + this.amount.format(2)


// Value Object: Address
class Address:
    street: String
    city: String
    state: String
    zipCode: ZipCode  // outro VO
    country: Country  // outro VO

    constructor(street, city, state, zipCode, country):
        require street.isNotBlank(), "Street is required"
        require city.isNotBlank(), "City is required"
        // ... demais validações
        this.street = street
        this.city = city
        this.state = state
        this.zipCode = zipCode
        this.country = country

    // Imutável — retorna nova instância
    withZipCode(newZipCode: ZipCode): Address
        return new Address(this.street, this.city, this.state, newZipCode, this.country)

    equals(other): Boolean
        return this.street == other.street 
            and this.city == other.city
            and this.state == other.state
            and this.zipCode == other.zipCode
            and this.country == other.country


// Value Object: DateRange (período temporal)
class DateRange:
    start: Date
    end: Date

    constructor(start, end):
        require end >= start, "End date must be after start date"
        this.start = start
        this.end = end

    contains(date: Date): Boolean
        return date >= this.start and date <= this.end

    overlaps(other: DateRange): Boolean
        return this.start <= other.end and this.end >= other.start

    durationInDays(): Integer
        return (this.end - this.start).days
```

### Anti-patterns

| Anti-pattern | Descrição | Correção |
|-------------|-----------|----------|
| **Primitive Obsession** | Usar `string`, `int`, `decimal` para conceitos de domínio | Criar Value Objects tipados |
| **Mutable Value Object** | VO com setters que alteram estado | Tornar imutável; retornar novas instâncias |
| **Value Object with Identity** | VO que tem ID ou é tratado como Entity | Remover identidade ou promover a Entity |
| **Anemic Value Object** | VO que é apenas um wrapper sem comportamento | Adicionar métodos de domínio relevantes |
| **VO com dependência de infra** | VO que acessa banco, API ou filesystem | VOs devem ser puros; mover I/O para fora |

---

## Aggregates

### Definição

> *"Um cluster de objetos de domínio que pode ser tratado como uma unidade única. Todo Aggregate tem uma raiz (root) e uma fronteira."*
> — Eric Evans

Um **Aggregate** é a **unidade fundamental de consistência** no DDD. Define:
- Quais objetos formam um grupo coeso
- Qual é o ponto de entrada único (**Aggregate Root**)
- Quais **invariantes** devem ser garantidas dentro da fronteira

### Regras Fundamentais

| Regra | Descrição |
|-------|-----------|
| **Root é o ponto de entrada** | Objetos externos só acessam o Aggregate via a Root Entity |
| **Identidade global pela root** | Apenas a root tem identidade global; internos têm identidade local |
| **Consistência transacional** | Invariantes dentro do Aggregate são garantidas em uma transação |
| **Consistência eventual entre Aggregates** | Regras entre Aggregates diferentes usam eventual consistency (Domain Events) |
| **Referência entre Aggregates por ID** | Nunca referencie outro Aggregate por objeto; use apenas o ID |
| **Um Aggregate por transação** | Cada transação deve modificar no máximo um Aggregate |
| **Delete em cascata** | Se a root é deletada, todos os internos também são |

### Design de Aggregates — Boas Práticas

1. **Prefira Aggregates pequenos** — A fronteira do Aggregate deve ser a mínima necessária para proteger invariantes. Aggregates grandes causam:
   - Contenção de lock
   - Problemas de performance
   - Conflitos de concorrência
   - Dificuldade de escalar

2. **Proteja invariantes de negócio** — O Aggregate existe para garantir regras de negócio. Pergunte: "Quais dados DEVEM ser consistentes de forma imediata (transacional)?"

3. **Referencie outros Aggregates por ID** — Isso garante:
   - Fronteiras claras
   - Lazy loading natural
   - Facilidade de distribuição (microservices)

4. **Um Aggregate Root = Um Repository** — Cada Aggregate tem exatamente um Repository. O Repository persiste e recupera o Aggregate inteiro.

5. **Use Domain Events para consistência entre Aggregates** — Se uma ação em um Aggregate deve afetar outro, use eventos:

```
// Order Aggregate publica evento → Inventory Aggregate consome
order.confirm()  // publica OrderConfirmed
// assincronamente...
inventoryService.handle(OrderConfirmed) // reserva estoque
```

6. **Valide no Aggregate, não fora** — A validação de regras de negócio pertence ao Aggregate, não ao Application Service.

7. **Concorrência otimista** — Use versionamento (optimistic locking) para detectar conflitos em Aggregates modificados concorrentemente.

### Heurísticas para identificar violações

| Sinal | O que indica |
|-------|-------------|
| Aggregate com muitas entidades internas (>3-4) | Aggregate muito grande; considere dividir |
| Transação modifica múltiplos Aggregates | Viola regra de 1 Aggregate por transação |
| Aggregate referencia outro por objeto (não ID) | Fronteira não está clara |
| Repository retorna partes internas do Aggregate | Fronteira violada; root não é respeitada |
| Problemas de concorrência frequentes | Aggregate contém dados que mudam independentemente |
| Performance ruim ao carregar Aggregate | Aggregate muito grande; dados desnecessários |

### Exemplo (pseudocódigo)

```
// ═══════════════════════════════════════════
// AGGREGATE: Order (Root = Order)
// ═══════════════════════════════════════════

class Order:                          // ◄── AGGREGATE ROOT
    id: OrderId                       // Identidade global
    customerId: CustomerId            // Referência por ID (outro Aggregate)
    status: OrderStatus               // Value Object
    items: List<OrderItem>            // Entidade interna (identidade local)
    shippingAddress: Address          // Value Object
    version: Integer                  // Para optimistic locking
    domainEvents: List<DomainEvent>   // Eventos pendentes

    // ── Invariante: pedido deve ter ao menos 1 item ──
    // ── Invariante: total não pode exceder limite de crédito ──
    // ── Invariante: itens não podem ser duplicados ──

    static create(customerId, shippingAddress, items): Order
        require items.isNotEmpty(), "Order must have at least one item"
        order = new Order()
        order.id = OrderId.generate()
        order.customerId = customerId
        order.shippingAddress = shippingAddress
        order.items = items
        order.status = OrderStatus.CREATED
        order.domainEvents.add(OrderCreated(order.id, customerId, now()))
        return order

    addItem(productId: ProductId, quantity: Quantity, unitPrice: Money):
        require this.status == OrderStatus.CREATED
        existing = this.items.find(i -> i.productId == productId)
        if existing:
            existing.increaseQuantity(quantity)
        else:
            this.items.add(OrderItem.create(productId, quantity, unitPrice))

    removeItem(productId: ProductId):
        require this.status == OrderStatus.CREATED
        this.items.removeWhere(i -> i.productId == productId)
        require this.items.isNotEmpty(), "Cannot remove last item"

    confirm():
        require this.status == OrderStatus.CREATED
        require this.items.isNotEmpty()
        this.status = OrderStatus.CONFIRMED
        this.domainEvents.add(
            OrderConfirmed(this.id, this.totalAmount(), now()))

    cancel(reason: CancellationReason):
        require this.status.isCancellable()
        this.status = OrderStatus.CANCELLED
        this.domainEvents.add(
            OrderCancelled(this.id, reason, now()))

    totalAmount(): Money
        return this.items.sum(item -> item.subtotal())


// ── ENTIDADE INTERNA (identidade local dentro do Aggregate) ──
class OrderItem:
    id: OrderItemId           // Identidade LOCAL
    productId: ProductId      // Referência por ID
    quantity: Quantity         // Value Object
    unitPrice: Money          // Value Object

    static create(productId, quantity, unitPrice): OrderItem
        return new OrderItem(OrderItemId.generate(), productId, quantity, unitPrice)

    increaseQuantity(additional: Quantity):
        this.quantity = this.quantity.add(additional)

    subtotal(): Money
        return this.unitPrice.multiply(this.quantity.value)
```

### Anti-patterns

| Anti-pattern | Descrição | Correção |
|-------------|-----------|----------|
| **Mega Aggregate** | Aggregate com 10+ entidades, carregamento lento | Dividir, referenciar por ID, usar Domain Events |
| **Anemic Aggregate** | Root sem comportamento, lógica no Service | Mover invariantes para o Aggregate |
| **Cross-Aggregate Transaction** | Uma transação modifica 2+ Aggregates | Usar eventual consistency via Domain Events |
| **Direct Reference** | Aggregate contém referência a outro Aggregate | Substituir por ID reference |
| **Repository per Entity** | Repository para entidades internas do Aggregate | Um Repository por Aggregate Root |
| **Exposed Internals** | Entidades internas acessíveis diretamente de fora | Acessar apenas via Aggregate Root |

---

## Domain Events

### Definição

> *"Algo que aconteceu no domínio que os especialistas de domínio se importam."*
> — Eric Evans (refinado por Vaughn Vernon)

Um **Domain Event** captura um **fato** que ocorreu no domínio. Eventos são imutáveis, no passado, e carregam informação sobre o que aconteceu.

### Características

| Característica | Descrição |
|---------------|-----------|
| **Imutável** | Uma vez criado, nunca muda |
| **Passado** | Nomear no passado: `OrderPlaced`, `PaymentReceived`, não `PlaceOrder` |
| **Fato** | Registra algo que já aconteceu, não uma intenção |
| **Autônomo** | Contém toda informação necessária para ser processado |
| **Temporal** | Inclui timestamp de quando ocorreu |

### Boas Práticas

1. **Nomeie como fatos de negócio** — Use linguagem ubíqua no passado:

| BOM (fato de negócio) | RUIM (técnico/comando) |
|-----------------------|------------------------|
| `OrderConfirmed` | `OrderStatusChanged` |
| `PaymentReceived` | `UpdatePayment` |
| `ItemAddedToCart` | `CartModified` |
| `CustomerRegistered` | `SaveCustomer` |
| `ShipmentDispatched` | `ShipmentEvent` |

2. **Inclua dados suficientes** — O consumidor não deve precisar consultar o produtor:

```
// BOM — evento autônomo
event OrderConfirmed:
    orderId: OrderId
    customerId: CustomerId
    totalAmount: Money
    items: List<OrderItemSummary>
    confirmedAt: DateTime

// RUIM — evento pobre que força consulta
event OrderConfirmed:
    orderId: OrderId  // consumidor precisa buscar o resto
```

3. **Publique do Aggregate** — O Aggregate é quem registra os eventos durante operações de domínio.

4. **Consistência eventual** — Use eventos para sincronizar estado entre Aggregates e Bounded Contexts.

5. **Idempotência** — Consumidores devem ser idempotentes — processar o mesmo evento mais de uma vez não deve causar efeitos duplicados.

6. **Ordering e dedup** — Inclua `eventId` único e `timestamp` para possibilitar ordenação e deduplicação.

### Heurísticas para identificar candidatos a Domain Event

| Pergunta | Se "sim" → provavelmente é um evento |
|----------|---------------------------------------|
| "O especialista de negócio se importa com isso?" | `CustomerUpgraded`, `PolicyExpired` |
| "Outros contextos precisam reagir a isso?" | `OrderPlaced` → Inventory, Billing |
| "Isso representa uma mudança de estado significativa?" | `AccountActivated`, `ContractSigned` |
| "Precisamos de auditoria/histórico?" | Todos os eventos servem como audit trail |
| "O negócio tem regras tipo 'quando X acontece, faça Y'?" | `PaymentFailed` → NotifyCustomer |

### Exemplo (pseudocódigo)

```
// ── Definição de Domain Events ──

event OrderCreated:
    eventId: UUID
    orderId: OrderId
    customerId: CustomerId
    items: List<{productId, quantity, unitPrice}>
    occurredAt: DateTime

event OrderConfirmed:
    eventId: UUID
    orderId: OrderId
    customerId: CustomerId
    totalAmount: Money
    shippingAddress: Address
    occurredAt: DateTime

event OrderCancelled:
    eventId: UUID
    orderId: OrderId
    reason: CancellationReason
    cancelledBy: UserId
    occurredAt: DateTime


// ── Aggregate publica eventos ──

class Order:
    domainEvents: List<DomainEvent>

    confirm():
        require this.status == OrderStatus.CREATED
        this.status = OrderStatus.CONFIRMED
        this.domainEvents.add(
            OrderConfirmed(
                eventId = UUID.random(),
                orderId = this.id,
                customerId = this.customerId,
                totalAmount = this.totalAmount(),
                shippingAddress = this.shippingAddress,
                occurredAt = DateTime.now()
            ))


// ── Handlers reagem a eventos ──

class InventoryEventHandler:
    handle(event: OrderConfirmed):
        for item in event.items:
            inventory.reserve(item.productId, item.quantity)

class NotificationEventHandler:
    handle(event: OrderConfirmed):
        notification.send(event.customerId, 
            "Your order " + event.orderId + " has been confirmed!")

class BillingEventHandler:
    handle(event: OrderConfirmed):
        billing.createInvoice(event.orderId, event.totalAmount)
```

### Anti-patterns

| Anti-pattern | Descrição | Correção |
|-------------|-----------|----------|
| **Event as Command** | Evento que ordena ação (`ProcessPayment`) | Renomear como fato ocorrido (`PaymentRequested`) |
| **Fat Event** | Evento com todo o estado do Aggregate serializado | Incluir apenas dados necessários |
| **Missing Event** | Side-effects distribuídos sem eventos explícitos | Fazer explícito o que aconteceu com um evento |
| **Coupled Handler** | Handler que chama diretamente outro Aggregate em transação | Usar mensageria/eventual consistency |
| **Generic Event** | `EntityChanged`, `StatusUpdated` — sem semântica de domínio | Nomear com linguagem ubíqua |

---

## Repositories

### Definição

> *"Repository dá a ilusão de uma coleção em memória de todos os objetos daquele tipo."*
> — Eric Evans

Um **Repository** encapsula o acesso a dados, oferecendo uma interface de **coleção** para persistir e recuperar **Aggregates inteiros**.

### Boas Práticas

1. **Um Repository por Aggregate Root** — Nunca para entidades internas.

2. **Interface no domínio, implementação na infraestrutura** — Segue o DIP (Dependency Inversion):

```
// Na camada de domínio (interface)
interface OrderRepository:
    findById(id: OrderId): Order?
    save(order: Order): void
    nextId(): OrderId

// Na camada de infraestrutura (implementação)
class SqlOrderRepository implements OrderRepository:
    findById(id: OrderId): Order?
        // SQL, ORM, etc.
    save(order: Order): void
        // Persiste o Aggregate inteiro
```

3. **Persista Aggregates inteiros** — O Repository salva e carrega o Aggregate completo, não partes dele.

4. **Sem lógica de negócio** — O Repository apenas persiste e recupera. Regras ficam no Aggregate ou Domain Service.

5. **Operações de coleção** — Métodos do Repository devem parecer operações de coleção: `add`, `save`, `findById`, `findByCriteria`, `remove`.

6. **Especificações para queries complexas** — Use o **Specification Pattern** para queries complexas, mantendo a lógica de seleção no domínio:

```
interface Specification<T>:
    isSatisfiedBy(candidate: T): Boolean

class OverdueOrdersSpecification implements Specification<Order>:
    isSatisfiedBy(order):
        return order.status == CONFIRMED 
            and order.confirmedAt < now() - 7.days
```

### Heurísticas para identificar violações

| Sinal | O que indica |
|-------|-------------|
| Repository para entidade interna do Aggregate | Fronteira do Aggregate violada |
| Repository com métodos de lógica de negócio | Responsabilidade errada; lógica deve estar no domínio |
| Repository retorna DTOs ou projections na camada de domínio | Mistura de concerns; use CQRS para leituras |
| Interface do Repository na camada de infraestrutura | Viola DIP; interface deve estar no domínio |
| Repository com dezenas de métodos de query | Considere Specification pattern ou CQRS |

### Exemplo (pseudocódigo)

```
// ═══ Interface no Domínio ═══
interface OrderRepository:
    findById(id: OrderId): Order?
    save(order: Order): void
    remove(order: Order): void
    findByCustomer(customerId: CustomerId): List<Order>
    findBySpecification(spec: Specification<Order>): List<Order>

// ═══ Implementação na Infraestrutura ═══
class JpaOrderRepository implements OrderRepository:
    
    findById(id: OrderId): Order?
        // Carrega Aggregate inteiro (root + items)
        return entityManager.find(Order, id)

    save(order: Order): void
        entityManager.persist(order)
        // Publica Domain Events pendentes
        for event in order.domainEvents:
            eventPublisher.publish(event)
        order.clearDomainEvents()

    remove(order: Order): void
        entityManager.remove(order)

    findByCustomer(customerId: CustomerId): List<Order>
        return query("SELECT o FROM Order o WHERE o.customerId = :id")
            .param("id", customerId)
            .list()
```

### Anti-patterns

| Anti-pattern | Descrição | Correção |
|-------------|-----------|----------|
| **Generic Repository** | `Repository<T>` genérico com CRUD para tudo | Cada Aggregate define sua interface específica |
| **Repository per Table** | Repository para cada tabela do banco | Repository por Aggregate Root |
| **Business Logic in Repo** | Repository que calcula desconto, valida regras | Mover lógica para Aggregate/Service |
| **Leaking Infrastructure** | Queries SQL/HQL expostas no domínio | Encapsular na implementação do Repo |
| **DAO Revival** | Repository que é basicamente um DAO procedural | Repensar como coleção de Aggregates |

---

## Domain Services

### Definição

> *"Quando uma operação significativa do domínio não pertence naturalmente a nenhuma Entity ou Value Object."*
> — Eric Evans

Um **Domain Service** encapsula lógica de negócio que:
- Envolve múltiplas Entities/Aggregates
- Não pertence naturalmente a nenhuma Entity específica
- Representa um processo ou política do domínio

### Quando usar

| Use Domain Service quando... | NÃO use quando... |
|------------------------------|---------------------|
| A operação envolve múltiplos Aggregates | A lógica pertence naturalmente a uma Entity |
| Seria artificial forçar em uma Entity | É apenas orquestração (use Application Service) |
| Representa uma política/regra de negócio cross-entity | É acesso a dados (use Repository) |
| É stateless — não mantém estado | Precisa manter estado (use Entity) |

### Boas Práticas

1. **Stateless** — Domain Services não têm estado. Recebem tudo via parâmetros.

2. **Nomeie com linguagem de domínio** — `PricingService`, `TransferService`, `FraudDetectionPolicy`, não `CalculatorUtil`.

3. **Interface no domínio** — Assim como Repositories, defina interface na camada de domínio.

4. **Não substitua Entities** — Se a lógica pertence naturalmente a uma Entity, coloque-a lá. Domain Service é último recurso.

5. **Diferente de Application Service** — Domain Service contém **regras de negócio**. Application Service apenas **orquestra** (coordena chamadas a domínio, transações, etc.).

### Exemplo (pseudocódigo)

```
// ═══ Domain Service: Transferência entre contas ═══
// Não pertence nem a Account A nem a Account B

interface TransferService:
    transfer(from: AccountId, to: AccountId, amount: Money): TransferResult

class TransferServiceImpl implements TransferService:
    accountRepository: AccountRepository
    exchangeRateService: ExchangeRateService  // anti-corruption para API externa

    transfer(from: AccountId, to: AccountId, amount: Money):
        sourceAccount = accountRepository.findById(from)
            ?? throw AccountNotFound(from)
        targetAccount = accountRepository.findById(to)
            ?? throw AccountNotFound(to)

        // Regra de negócio: conversão cambial se moedas diferentes
        transferAmount = amount
        if sourceAccount.currency != targetAccount.currency:
            rate = exchangeRateService.getRate(
                sourceAccount.currency, targetAccount.currency)
            transferAmount = amount.convertTo(targetAccount.currency, rate)

        // Operações no domínio
        sourceAccount.debit(amount)
        targetAccount.credit(transferAmount)

        // Persistência
        accountRepository.save(sourceAccount)
        accountRepository.save(targetAccount)

        return TransferResult.success(amount, transferAmount)


// ═══ Domain Service: Política de desconto ═══
interface DiscountPolicy:
    calculateDiscount(order: Order, customer: Customer): Money

class VolumeDiscountPolicy implements DiscountPolicy:
    calculateDiscount(order, customer):
        if order.totalAmount().isGreaterThan(Money(1000, BRL)):
            return order.totalAmount().multiply(0.10)  // 10%
        if customer.isVIP():
            return order.totalAmount().multiply(0.05)  // 5%
        return Money.ZERO
```

### Anti-patterns

| Anti-pattern | Descrição | Correção |
|-------------|-----------|----------|
| **Service para tudo** | Toda lógica em Services, Entities anêmicas | Mover lógica para Entities quando natural |
| **Stateful Service** | Service que mantém estado entre chamadas | Tornar stateless; estado pertence a Entities |
| **Technical Service como Domain** | `EmailService`, `HttpService` na camada de domínio | São services de infraestrutura, não de domínio |
| **Orchestration Service** | Service que só "cola" chamadas sem regrar de negócio | É um Application Service, não Domain Service |

---

## Factories

### Definição

> *"Encapsular a criação de objetos complexos e Aggregates, fornecendo uma interface que abstraia a complexidade da construção."*
> — Eric Evans

### Quando usar

- Criação de Aggregate envolve lógica de negócio
- Construtor exige muitos parâmetros complexos
- Criação depende de informações externas (ex: geração de ID)
- Reconstitução a partir de persistência precisa ser separada da criação de domínio

### Boas Práticas

1. **Factory Method no Aggregate Root** — Para criações simples:

```
class Order:
    static create(customerId, items, shippingAddress): Order
        // Validações e lógica de criação
        order = new Order(OrderId.generate(), customerId, items, shippingAddress)
        order.registerEvent(OrderCreated(...))
        return order
```

2. **Factory Class separada** — Para criações complexas:

```
class LoanApplicationFactory:
    create(applicant, requestedAmount, term):
        creditScore = creditService.evaluate(applicant)
        riskLevel = riskAssessment.calculate(applicant, requestedAmount)
        interestRate = rateCalculator.determine(creditScore, riskLevel, term)
        
        return LoanApplication.create(
            applicant, requestedAmount, term, interestRate, riskLevel)
```

3. **Separe criação de reconstituição** — Factory de domínio (criar novos) vs. Repository (reconstituir do banco) têm preocupações diferentes.

### Exemplo (pseudocódigo)

```
// Factory Method no Aggregate Root (simples)
class Account:
    static openNew(ownerId: CustomerId, type: AccountType): Account
        account = new Account(
            id = AccountId.generate(),
            ownerId = ownerId,
            type = type,
            balance = Money.ZERO,
            status = AccountStatus.ACTIVE,
            openedAt = DateTime.now()
        )
        account.registerEvent(AccountOpened(account.id, ownerId))
        return account


// Factory Class (criação complexa envolvendo regras externas)
class InsurancePolicyFactory:
    riskAssessor: RiskAssessor
    premiumCalculator: PremiumCalculator

    createPolicy(applicant, coverage, term):
        risk = riskAssessor.assess(applicant, coverage)
        
        if risk.isUnacceptable():
            throw PolicyCreationDenied("Risk level too high")

        premium = premiumCalculator.calculate(coverage, risk, term)

        return InsurancePolicy.create(
            applicant = applicant,
            coverage = coverage,
            term = term,
            premium = premium,
            riskLevel = risk
        )
```

---

## Modules (Pacotes/Namespaces)

### Boas Práticas

1. **Organize por Bounded Context / Subdomain**, não por tipo técnico:

```
// BOM — por domínio
src/
  order/
    Order.ext
    OrderItem.ext
    OrderRepository.ext
    OrderConfirmed.ext
  catalog/
    Product.ext
    Category.ext
    ProductRepository.ext
  shipping/
    Shipment.ext
    Route.ext

// RUIM — por tipo técnico
src/
  entities/
    Order.ext
    Product.ext
    Shipment.ext
  repositories/
    OrderRepository.ext
    ProductRepository.ext
  services/
    OrderService.ext
    ShippingService.ext
  events/
    OrderConfirmed.ext
```

2. **Módulos devem ter alta coesão e baixo acoplamento** — Coisas que mudam juntas ficam juntas.

3. **Nomes de módulos = termos da Ubiquitous Language** — `order`, `catalog`, `billing`, não `utils`, `common`, `base`.

4. **Minimize dependências entre módulos** — Se dois módulos são altamente acoplados, talvez devam ser um só.

---

## Relações entre Building Blocks

```
┌─────────────────────────────────────────────────┐
│                AGGREGATE BOUNDARY                │
│                                                 │
│  ┌─────────────────┐                            │
│  │ AGGREGATE ROOT  │──── possui ──→ VALUE OBJECTS │
│  │   (Entity)      │                            │
│  │                 │──── contém ──→ ENTITIES     │
│  │                 │     (internas)              │
│  │                 │──── publica ─→ DOMAIN EVENTS │
│  └────────┬────────┘                            │
│           │                                     │
│           │ referencia por ID                    │
│           ▼                                     │
│    outros Aggregates                            │
└─────────────────────────────────────────────────┘
         │                    │
         │ persiste/          │ contém regras
         │ recupera           │ cross-aggregate
         ▼                    ▼
    REPOSITORY          DOMAIN SERVICE
    (interface no       (interface no
     domínio)            domínio)

    FACTORY → cria → AGGREGATE
```

---

## Diretrizes para Code Review assistido por AI

> Regras acionáveis para que um assistente de código identifique violações dos Building Blocks do DDD.

| # | Regra de detecção | Building Block | Severidade | Ação sugerida |
|---|-------------------|---------------|-----------|---------------|
| 1 | Entity com apenas getters/setters, sem métodos de negócio | Entity | **Alto** | Mover lógica dos Services para a Entity |
| 2 | Primitivos (`string`, `int`, `decimal`) para conceitos de domínio (email, money, cpf) | Value Object | **Alto** | Criar Value Object tipado com validação |
| 3 | Value Object mutável (com setters) | Value Object | **Alto** | Tornar imutável; retornar novas instâncias |
| 4 | Aggregate com mais de 4-5 entidades internas | Aggregate | **Médio** | Considerar divisão e referência por ID |
| 5 | Transação modifica mais de um Aggregate | Aggregate | **Alto** | Usar Domain Events para eventual consistency |
| 6 | Aggregate referencia outro por objeto (não ID) | Aggregate | **Alto** | Substituir por referência de ID |
| 7 | Repository para entidade interna do Aggregate | Repository | **Alto** | Remover; acessar via Aggregate Root |
| 8 | Repository com lógica de negócio | Repository | **Médio** | Mover lógica para Aggregate ou Domain Service |
| 9 | Domain Event nomeado como comando (`ProcessPayment`) | Domain Event | **Médio** | Renomear como fato ocorrido (`PaymentProcessed`) |
| 10 | Toda lógica em Services, nenhuma em Entities | Domain Service | **Alto** | Identificar e redistribuir para Entities |
| 11 | Interface do Repository definida na camada de infraestrutura | Repository | **Médio** | Mover interface para camada de domínio |
| 12 | Módulos organizados por tipo técnico (entities/, services/) | Modules | **Médio** | Reorganizar por domínio/bounded context |

---

## Referências

| Recurso | Autor | Tipo |
|---------|-------|------|
| *Domain-Driven Design: Tackling Complexity in the Heart of Software* | Eric Evans | Livro |
| *Implementing Domain-Driven Design* | Vaughn Vernon | Livro |
| *Domain-Driven Design Distilled* | Vaughn Vernon | Livro |
| *Learning Domain-Driven Design* | Vlad Khononov | Livro |
| *Hands-On Domain-Driven Design with .NET Core* (conceitos são agnósticos) | Alexey Zimarev | Livro |
| *Effective Aggregate Design* (3-part series) | Vaughn Vernon | Paper |
