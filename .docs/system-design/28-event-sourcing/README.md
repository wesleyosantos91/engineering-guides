# 28. Event Sourcing

> **Categoria:** Padrões Arquiteturais  
> **Nível:** Avançado — essencial para domínios financeiros e auditáveis  
> **Complexidade:** Alta

---

## Definição

**Event Sourcing** é um padrão onde o **estado** de uma entidade é determinado pela **sequência de eventos** que ocorreram sobre ela, em vez de armazenar apenas o estado atual. O **Event Store** (append-only log) é a **source of truth**.

```
Tradicional (State Sourcing):
  Conta Bancária: { saldo: 150.00 }  ← só o resultado final

Event Sourcing:
  Evento 1: ContaCriada      { saldo_inicial: 0.00 }
  Evento 2: DepósitoRealizado { valor: 200.00 }
  Evento 3: SaqueRealizado    { valor: 50.00 }
  → Estado atual = replay: 0 + 200 - 50 = 150.00
```

---

## Por Que é Importante?

- **Audit trail completo** — sabe-se exatamente o que aconteceu e quando
- **Time-travel** — reconstrói o estado em qualquer ponto do tempo
- **Debugging poderoso** — reproduz bugs com sequência exata de eventos
- **Base natural para CQRS** — eventos alimentam read models
- **Usado em domínios críticos** — finanças, saúde, compliance, gaming

---

## Diagrama de Arquitetura

```
Modelo Tradicional (State Sourcing):

  Command ──▶ Service ──▶ UPDATE table SET status='shipped' WHERE id=123
                          (sobrescreveu o estado anterior — PERDEU o "pending")

Event Sourcing:

  Command ──▶ Aggregate ──▶ APPEND evento ao Event Store
  
  Event Store (append-only):
  ┌────┬──────────────────┬────────────────────────────┬─────────────┐
  │ #  │ Event Type        │ Data                       │ Timestamp   │
  ├────┼──────────────────┼────────────────────────────┼─────────────┤
  │ 1  │ OrderCreated     │ {customerId, items, total}  │ 10:00:00   │
  │ 2  │ PaymentReceived  │ {orderId, amount, method}   │ 10:00:05   │
  │ 3  │ OrderShipped     │ {orderId, trackingId}       │ 10:30:00   │
  │ 4  │ OrderDelivered   │ {orderId, signature}        │ 11:45:00   │
  └────┴──────────────────┴────────────────────────────┴─────────────┘
  
  Estado em T1: { status: "created" }
  Estado em T2: { status: "paid" }
  Estado em T3: { status: "shipped", trackingId: "TRK123" }
  Estado em T4: { status: "delivered", signature: "..." }
  
  NUNCA atualiza, NUNCA deleta — só APPEND
```

---

## Conceitos Fundamentais

### Aggregate

```
Aggregate = Entidade principal que gera e aplica eventos

  class OrderAggregate:
    - id, status, items, total, ...
    
    Recebe Command:
      createOrder(cmd) → gera OrderCreatedEvent
      shipOrder(cmd)   → gera OrderShippedEvent
      cancelOrder(cmd) → gera OrderCancelledEvent
    
    Aplica Event:
      on(OrderCreatedEvent) → this.status = "CREATED"
      on(OrderShippedEvent) → this.status = "SHIPPED"
      on(OrderCancelledEvent) → this.status = "CANCELLED"
    
    Regras de negócio no Command:
      cancelOrder: if status == "SHIPPED" → EXCEPTION!
```

### Event Store

```
Event Store = Log append-only de eventos

  Características:
  ├── Append-only (nunca UPDATE, nunca DELETE)
  ├── Ordered (por aggregate, por global position)
  ├── Immutable (eventos são fatos, não mudam)
  └── Source of Truth (estado é derivado dos eventos)

  Schema típico:
  ┌──────────────────────────────────────────────────┐
  │ events                                           │
  ├──────────────┬──────────┬────────────────────────┤
  │ event_id     │ UUID     │ PK                     │
  │ aggregate_id │ VARCHAR  │ FK (ex: order-123)     │
  │ aggregate_type│ VARCHAR │ (ex: "Order")          │
  │ event_type   │ VARCHAR  │ (ex: "OrderCreated")   │
  │ event_data   │ JSONB    │ payload do evento      │
  │ metadata     │ JSONB    │ correlation, causation  │
  │ version      │ INTEGER  │ sequence por aggregate │
  │ created_at   │ TIMESTAMP│ quando ocorreu         │
  │ global_pos   │ BIGSERIAL│ posição global         │
  └──────────────┴──────────┴────────────────────────┘
  
  Índicesresses: (aggregate_id, version) — unique
                   global_pos — para projections
```

### Reconstrução de Estado (Replay)

```
Para obter estado atual do aggregate:

  1. Ler TODOS os eventos do aggregate (sorted by version)
  2. Aplicar cada evento sequencialmente
  3. Resultado = estado atual

  events = eventStore.getEvents(aggregateId: "order-123")
  
  order = new OrderAggregate()
  for event in events:
      order.apply(event)     ← reconstrói estado evento por evento
  
  // order agora tem o estado ATUAL
  
  Exemplo:
    #1 OrderCreated → status=CREATED, items=[A,B]
    #2 ItemRemoved  → status=CREATED, items=[A]
    #3 PaymentReceived → status=PAID, items=[A]
    
    Estado final: { status: PAID, items: [A] }
```

---

## Implementação

### Event Store (Python)

```python
class EventStore:
    def __init__(self, db):
        self.db = db
    
    def append(self, aggregate_id: str, events: list, expected_version: int):
        """Append eventos com optimistic concurrency."""
        with self.db.transaction() as tx:
            # Verifica versão atual (optimistic locking)
            current_version = tx.query(
                "SELECT MAX(version) FROM events WHERE aggregate_id = %s",
                [aggregate_id]
            ) or 0
            
            if current_version != expected_version:
                raise ConcurrencyException(
                    f"Expected version {expected_version}, "
                    f"but found {current_version}"
                )
            
            # Append eventos
            for i, event in enumerate(events):
                version = expected_version + i + 1
                tx.execute("""
                    INSERT INTO events 
                    (event_id, aggregate_id, aggregate_type, event_type, 
                     event_data, version, metadata)
                    VALUES (%s, %s, %s, %s, %s, %s, %s)
                """, [
                    str(uuid4()),
                    aggregate_id,
                    event.aggregate_type,
                    event.event_type,
                    json.dumps(event.data),
                    version,
                    json.dumps(event.metadata)
                ])
    
    def get_events(self, aggregate_id: str) -> list:
        """Retorna todos os eventos de um aggregate, ordenados."""
        return self.db.query("""
            SELECT * FROM events 
            WHERE aggregate_id = %s 
            ORDER BY version ASC
        """, [aggregate_id])
    
    def get_events_since(self, global_position: int) -> list:
        """Para projections: eventos desde uma posição global."""
        return self.db.query("""
            SELECT * FROM events 
            WHERE global_pos > %s 
            ORDER BY global_pos ASC
            LIMIT 1000
        """, [global_position])
```

### Aggregate (Java)

```java
public class OrderAggregate {
    
    private String id;
    private OrderStatus status;
    private List<OrderItem> items;
    private BigDecimal total;
    private int version;
    
    // Lista de novos eventos (uncommitted)
    private final List<DomainEvent> uncommittedEvents = new ArrayList<>();
    
    // --- COMMAND HANDLERS (geram eventos) ---
    
    public static OrderAggregate create(CreateOrderCommand cmd) {
        OrderAggregate order = new OrderAggregate();
        order.apply(new OrderCreatedEvent(
            cmd.getOrderId(),
            cmd.getCustomerId(),
            cmd.getItems(),
            cmd.getTotal()
        ));
        return order;
    }
    
    public void ship(ShipOrderCommand cmd) {
        if (status != OrderStatus.PAID) {
            throw new IllegalStateException("Cannot ship unpaid order");
        }
        apply(new OrderShippedEvent(id, cmd.getTrackingId()));
    }
    
    public void cancel(CancelOrderCommand cmd) {
        if (status == OrderStatus.SHIPPED || status == OrderStatus.DELIVERED) {
            throw new IllegalStateException("Cannot cancel shipped order");
        }
        apply(new OrderCancelledEvent(id, cmd.getReason()));
    }
    
    // --- EVENT HANDLERS (aplicam estado) ---
    
    private void on(OrderCreatedEvent event) {
        this.id = event.getOrderId();
        this.status = OrderStatus.CREATED;
        this.items = event.getItems();
        this.total = event.getTotal();
    }
    
    private void on(OrderShippedEvent event) {
        this.status = OrderStatus.SHIPPED;
    }
    
    private void on(OrderCancelledEvent event) {
        this.status = OrderStatus.CANCELLED;
    }
    
    // --- INFRASTRUCTURE ---
    
    private void apply(DomainEvent event) {
        // Aplica mudança de estado
        handle(event);
        version++;
        uncommittedEvents.add(event);
    }
    
    public void rehydrate(List<DomainEvent> history) {
        for (DomainEvent event : history) {
            handle(event);
            version++;
        }
    }
    
    private void handle(DomainEvent event) {
        switch (event) {
            case OrderCreatedEvent e -> on(e);
            case OrderShippedEvent e -> on(e);
            case OrderCancelledEvent e -> on(e);
            default -> throw new IllegalArgumentException("Unknown event");
        }
    }
}
```

### Repository

```java
@Repository
public class EventSourcedOrderRepository {
    
    private final EventStore eventStore;
    
    public OrderAggregate load(String orderId) {
        // 1. Busca todos os eventos
        List<DomainEvent> events = eventStore.getEvents(orderId);
        
        if (events.isEmpty()) {
            throw new AggregateNotFoundException(orderId);
        }
        
        // 2. Reconstrói aggregate
        OrderAggregate order = new OrderAggregate();
        order.rehydrate(events);
        
        return order;
    }
    
    public void save(OrderAggregate order) {
        // 3. Salva novos eventos (optimistic concurrency)
        eventStore.append(
            order.getId(),
            order.getUncommittedEvents(),
            order.getVersion() - order.getUncommittedEvents().size()
        );
        order.clearUncommittedEvents();
    }
}
```

---

## Snapshots

```
Problema: aggregate com 10.000 eventos → replay lento!

Solução: snapshots periódicos

  Eventos: [1] [2] [3] ... [999] [1000] [SNAPSHOT] [1001] [1002] [1003]
  
  Para reconstruir:
  1. Carrega snapshot mais recente (version 1000)
  2. Carrega apenas eventos após o snapshot (1001, 1002, 1003)
  3. Aplica eventos sobre o snapshot
  
  → Replay de 3 eventos em vez de 1003!

  Quando criar snapshot:
  - A cada N eventos (ex: a cada 100)
  - Quando aggregate é salvo
  - Em background periodicamente
```

### Implementação de Snapshot

```python
class SnapshotStore:
    def save_snapshot(self, aggregate_id: str, version: int, state: dict):
        self.db.upsert("snapshots", {
            "aggregate_id": aggregate_id,
            "version": version,
            "state": json.dumps(state),
            "created_at": datetime.utcnow()
        })
    
    def get_latest_snapshot(self, aggregate_id: str):
        return self.db.query_one("""
            SELECT * FROM snapshots 
            WHERE aggregate_id = %s 
            ORDER BY version DESC LIMIT 1
        """, [aggregate_id])


class EventSourcedRepository:
    def load(self, aggregate_id: str):
        # 1. Tenta carregar snapshot
        snapshot = self.snapshot_store.get_latest_snapshot(aggregate_id)
        
        if snapshot:
            # 2. Reconstrói do snapshot
            aggregate = OrderAggregate.from_snapshot(snapshot.state)
            # 3. Carrega apenas eventos APÓS o snapshot
            events = self.event_store.get_events_after(
                aggregate_id, snapshot.version
            )
        else:
            aggregate = OrderAggregate()
            events = self.event_store.get_events(aggregate_id)
        
        # 4. Aplica eventos restantes
        aggregate.rehydrate(events)
        
        # 5. Cria snapshot se necessário
        if len(events) > 100:  # threshold
            self.snapshot_store.save_snapshot(
                aggregate_id, 
                aggregate.version, 
                aggregate.to_snapshot()
            )
        
        return aggregate
```

---

## Projections

```
Projection = materialização dos eventos em uma view otimizada para leitura

  Event Store (source of truth)
    │
    ├──▶ Projection: "Orders by status"
    │    SELECT status, COUNT(*) FROM orders_view GROUP BY status
    │    → { CREATED: 150, PAID: 420, SHIPPED: 890 }
    │
    ├──▶ Projection: "Customer order history"  
    │    → { customer_123: [order_1, order_2, order_3] }
    │
    └──▶ Projection: "Revenue dashboard"
         → { today: $45,000, this_week: $312,000 }

  Cada projection:
  - Consome eventos do Event Store
  - Mantém estado derivado em Read DB próprio
  - Pode ser RECONSTRUÍDA do zero (replay ALL events)
  - Escala independentemente
```

---

## Event Upcasting (Schema Evolution)

```
Problema: evento V1 é diferente de V2

  V1 (2023): OrderCreated { orderId, items, total }
  V2 (2024): OrderCreated { orderId, lineItems, subtotal, tax, total }
  
  Event Store contém AMBAS as versões — como lidar?

Solução: Upcasters

  Upcaster V1→V2:
    input:  { orderId, items, total }
    output: { orderId, lineItems: items, subtotal: total, tax: 0, total }
    
  Aplicado NO MOMENTO DA LEITURA:
    events = eventStore.getEvents("order-123")
    events = upcaster.upcast(events)  ← transforma V1 em V2
    aggregate.rehydrate(events)
    
  Vantagem: Event Store nunca é modificado (imutável)
  Eventos antigos são transformados em runtime
```

---

## Temporal Queries (Time Travel)

```
Uma das features mais poderosas do Event Sourcing:

  "Qual era o estado do pedido ontem às 15:00?"
  
  events = eventStore.getEvents(
      aggregateId="order-123",
      upTo=datetime(2024, 1, 14, 15, 0, 0)
  )
  
  order = OrderAggregate()
  order.rehydrate(events)
  
  → Estado exato naquele momento!

Casos de uso:
  - Auditoria: "quem mudou o preço e quando?"
  - Debugging: "qual era o estado quando o bug ocorreu?"
  - Compliance: "estado do sistema em data X para regulador"
  - Bisect: "a partir de qual evento o estado ficou inconsistente?"
```

---

## Event Sourcing vs State Sourcing

| Aspecto | State Sourcing (CRUD) | Event Sourcing |
|---------|----------------------|----------------|
| **Armazenamento** | Estado atual (UPDATE) | Todos os eventos (APPEND) |
| **Source of truth** | Tabela com estado final | Event Store |
| **Histórico** | Perdido (ou log table separado) | Completo por natureza |
| **Time travel** | Impossível (sem histórico) | Nativo |
| **Performance de escrita** | UPDATE (pode ter lock) | APPEND-only (sem lock) |
| **Performance de leitura** | Direto (SELECT) | Replay necessário (ou snapshots) |
| **Complexidade** | Simples | Alta |
| **Debugging** | Difícil (estado sobrescrito) | Fácil (replay eventos) |
| **Schema evolution** | ALTER TABLE | Upcasting |
| **Storage** | Menos (só estado) | Mais (todos os eventos) |

---

## Desafios

### 1. Performance de Replay

```
Solução: Snapshots (descrito acima)
  - Salvar snapshot a cada N eventos
  - Replay apenas eventos após snapshot
```

### 2. Event Schema Evolution

```
Solução: Upcasting
  - Transforma eventos antigos no formato novo em runtime
  - Event Store permanece imutável
```

### 3. Eventual Consistency

```
Solução: CQRS read models eventualmente consistentes
  - Projections consomem eventos assincronamente
  - Read-your-own-writes para o próprio command issuer
```

### 4. Deleting Data (GDPR)

```
Problema: "Right to be forgotten" vs eventos imutáveis

Soluções:
  a) Crypto-shredding:
     - Dados sensíveis encriptados com chave por usuário
     - Deletar chave = dados indecifráveis
     
  b) Tombstone events:
     - DataPurgedEvent { userId } 
     - Projections removem dados
     - Event Store mantém tombstone
     
  c) Event rewriting (último recurso):
     - Reescreve stream removendo dados pessoais
     - Quebra imutabilidade — use com cuidado
```

### 5. Storage Growth

```
Problema: Event Store cresce indefinidamente

Soluções:
  - Archival: mover eventos antigos para cold storage (S3)
  - Compaction: substituir N eventos por snapshot + eventos recentes
  - TTL: em domínios onde histórico antigo não importa
```

---

## Uso em Big Techs

| Empresa | Uso | Event Store | Detalhes |
|---------|-----|-------------|----------|
| **Stripe** | Payment events | Custom | Todo pagamento é sequência de eventos (charge.created → charge.succeeded) |
| **LinkedIn** | Feed events | Kafka | Eventos de posts, likes, comments alimentam timeline |
| **Uber** | Trip lifecycle | Kafka + Cassandra | TripCreated → DriverAssigned → Started → Completed |
| **Netflix** | Viewing history | Kafka + Cassandra | Eventos de play, pause, seek, complete |
| **EventStoreDB** | Open-source ES DB | EventStoreDB | DB especializado para Event Sourcing (Greg Young) |
| **Axon Framework** | Java ES framework | Axon Server | Framework completo CQRS + Event Sourcing |

### Stripe — Event Sourcing de Pagamentos

```
Payment lifecycle é Event Sourcing natural:

  Event Store para payment_intent pi_123:
  
  #1  PaymentIntentCreated     { amount: 5000, currency: "usd" }
  #2  PaymentMethodAttached    { pm: "pm_visa_4242" }
  #3  PaymentIntentConfirmed   { }
  #4  ChargeCreated            { charge_id: "ch_abc" }
  #5  ChargeSucceeded          { charge_id: "ch_abc" }
  
  Estado atual: pi_123 = { status: "succeeded", amount: 5000 }
  
  Webhook events = exatamente os eventos do Event Store!
  Developers recebem: charge.succeeded, invoice.paid, etc.
  
  Benefícios para Stripe:
  - Audit trail completo (compliance financeiro)
  - Dispute resolution (replay exato do que aconteceu)
  - Time travel para debugging
  - Webhooks são eventos naturais
```

---

## Perguntas Frequentes em Entrevistas

1. **"O que é Event Sourcing?"**
   - Estado = sequência de eventos, não snapshot
   - Event Store é append-only, imutável
   - Reconstrói estado fazendo replay

2. **"Qual a diferença de Event Sourcing vs Event-Driven?"**
   - Event-Driven: comunicação entre serviços via eventos
   - Event Sourcing: persistência de estado via eventos
   - São complementares, não sinônimos

3. **"Como resolver performance de replay?"**
   - Snapshots a cada N eventos
   - Carrega snapshot + eventos após snapshot
   - Projections para queries (CQRS)

4. **"Como lidar com GDPR no Event Sourcing?"**
   - Crypto-shredding (criptografa dados sensíveis, deleta chave)
   - Tombstone events
   - Event rewriting (último recurso)

5. **"Quando usar Event Sourcing?"**
   - Domínios onde audit trail é crítico (finanças, saúde)
   - Quando time-travel é necessário
   - Complex domain logic com business rules
   - Combinado com CQRS para múltiplas views

---

## Referências

- Greg Young (2010) — *"Event Sourcing"* — Inventor do padrão moderno
- Martin Fowler — *"Event Sourcing"* — martinfowler.com
- EventStoreDB — eventstore.com — Database especializado para ES
- Axon Framework — axoniq.io — Framework Java para CQRS + ES
- Chris Richardson — *"Microservices Patterns"*, Cap. Event Sourcing
- Pat Helland — *"Immutability Changes Everything"* — CIDR 2015
