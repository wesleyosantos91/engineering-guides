# 27. CQRS (Command Query Responsibility Segregation)

> **Categoria:** Padrões Arquiteturais  
> **Nível:** Avançado — frequente em entrevistas de System Design  
> **Complexidade:** Alta

---

## Definição

**CQRS** é um padrão arquitetural que **separa o modelo de escrita (Command)** do **modelo de leitura (Query)**, permitindo que cada lado seja **otimizado, escalado e evoluído independentemente**.

- **Command side:** Recebe comandos de escrita, aplica regras de negócio, persiste dados (normalizado)
- **Query side:** Serve leituras otimizadas a partir de views pré-computadas (desnormalizado)

---

## Por Que é Importante?

- **Resolve o conflito** entre modelos otimizados para escrita vs leitura
- **Habilita escalabilidade independente** — reads e writes escalam separadamente
- **Combinação natural com Event Sourcing** — events alimentam o read model
- **Usado amplamente em Big Techs** — LinkedIn, Twitter, Microsoft
- **Pergunta avançada em entrevistas** — demonstra conhecimento arquitetural profundo

---

## Diagrama Central

```
                Modelo Tradicional (CRUD):
                ┌─────────────────┐
  Read/Write ──▶│   Same Model    │──▶ Same Database
                │  (compromisso)  │
                └─────────────────┘

                CQRS:
                ┌─────────────────┐     ┌──────────────┐
  Commands ────▶│   Write Model   │────▶│   Write DB   │
  (create,      │  (normalized,   │     │ (PostgreSQL) │
   update,      │   domain-rich)  │     └──────┬───────┘
   delete)      └─────────────────┘            │
                                          Event / CDC
                                               │
                ┌─────────────────┐     ┌──────▼───────┐
  Queries ─────▶│   Read Model    │◀────│   Read DB    │
  (get, list,   │ (denormalized,  │     │ (Elastic,    │
   search,      │  optimized)     │     │  Redis, etc) │
   aggregate)   └─────────────────┘     └──────────────┘
```

---

## Problema que Resolve

### Modelo Único (CRUD Tradicional)

```
Conflito: modelo de escrita ≠ modelo de leitura ideal

  Write (normalizado):                Read (denormalizado):
  ┌──────────┐  ┌──────────┐         ┌─────────────────────────┐
  │ orders   │  │ users    │         │ order_details_view      │
  │ ─────    │  │ ─────    │         │ ───────────────────     │
  │ id       │  │ id       │    ──▶  │ order_id               │
  │ user_id  │  │ name     │         │ user_name              │
  │ total    │  │ email    │         │ total                  │
  │ status   │  │          │         │ status                 │
  └──────────┘  └──────────┘         │ items_count            │
                                     │ last_updated           │
  3NF: sem duplicação,               └─────────────────────────┘
  bom para writes                    Flat: sem JOINs, 
                                     excelente para reads

  Com modelo único:
  - Writes são lentos (triggers, indexed columns)
  - Reads são lentos (JOINs complexos)
  - Ambos competem pelo mesmo recurso (lock contention)
```

---

## Command Side (Escrita)

### Anatomia de um Command

```
Command = Intenção de mudança (pode ser REJEITADO)

  CreateOrderCommand {
    customerId: "cust-123"
    items: [
      { productId: "prod-456", quantity: 2 }
    ]
    shippingAddress: { ... }
  }

  Fluxo:
  1. API recebe command
  2. Command Handler valida regras de negócio
  3. Se válido → aplica mudança no aggregate
  4. Persiste no Write DB
  5. Publica evento (OrderCreated)
  6. Retorna confirmação (ou erro)
```

### Implementação do Command Side

```java
// Command
public record CreateOrderCommand(
    String customerId,
    List<OrderItem> items,
    Address shippingAddress
) {}

// Command Handler
@Service
public class CreateOrderHandler {
    
    private final OrderRepository repository;
    private final EventPublisher eventPublisher;
    
    @Transactional
    public OrderId handle(CreateOrderCommand cmd) {
        // 1. Validação de negócio
        Customer customer = customerService.findById(cmd.customerId());
        if (!customer.isActive()) {
            throw new BusinessException("Customer is not active");
        }
        
        // 2. Criar aggregate
        Order order = Order.create(
            customer,
            cmd.items(),
            cmd.shippingAddress()
        );
        
        // 3. Persistir (Write DB — normalizado)
        repository.save(order);
        
        // 4. Publicar evento
        eventPublisher.publish(new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getItems(),
            order.getTotal()
        ));
        
        return order.getId();
    }
}
```

---

## Query Side (Leitura)

### Read Model / Projection

```
Read Model = Views otimizadas para queries específicas

  Fonte: eventos do Command Side
  
  OrderCreatedEvent → Projection Handler → Atualiza Read DB
  
  Read DB pode ser:
  ├── Elasticsearch (full-text search)
  ├── Redis (cache de consultas frequentes)
  ├── MongoDB (views desnormalizadas)
  ├── PostgreSQL materialized views
  └── DynamoDB (key-value lookups)
```

### Implementação do Query Side

```java
// Projection Handler — consome eventos e atualiza Read Model
@Component
public class OrderProjectionHandler {
    
    private final OrderViewRepository viewRepository;
    
    @EventHandler
    public void on(OrderCreatedEvent event) {
        OrderView view = OrderView.builder()
            .orderId(event.getOrderId())
            .customerName(event.getCustomerName()) // desnormalizado!
            .items(event.getItems())
            .total(event.getTotal())
            .status("CREATED")
            .createdAt(event.getTimestamp())
            .build();
        
        viewRepository.save(view); // Read DB (ex: Elasticsearch)
    }
    
    @EventHandler
    public void on(OrderShippedEvent event) {
        viewRepository.updateStatus(event.getOrderId(), "SHIPPED");
        viewRepository.updateTrackingId(event.getOrderId(), 
                                         event.getTrackingId());
    }
    
    @EventHandler
    public void on(OrderDeliveredEvent event) {
        viewRepository.updateStatus(event.getOrderId(), "DELIVERED");
        viewRepository.updateDeliveredAt(event.getOrderId(), 
                                          event.getTimestamp());
    }
}

// Query Service — lê do Read Model
@RestController
public class OrderQueryController {
    
    private final OrderViewRepository viewRepository;
    
    @GetMapping("/orders/{id}")
    public OrderView getOrder(@PathVariable String id) {
        return viewRepository.findById(id); // Read DB — sem JOINs!
    }
    
    @GetMapping("/orders")
    public Page<OrderView> searchOrders(
            @RequestParam(required = false) String status,
            @RequestParam(required = false) String customerName,
            Pageable pageable) {
        return viewRepository.search(status, customerName, pageable);
        // Elasticsearch query — otimizado para search
    }
}
```

---

## Sincronização entre Write e Read

### Via Eventos (mais comum)

```
Write DB ──▶ Event Publisher ──▶ Event Bus (Kafka) ──▶ Projection Handler ──▶ Read DB

  ┌─────────┐    ┌───────┐    ┌──────────┐    ┌─────────┐
  │Write DB │───▶│ Kafka │───▶│Projection│───▶│ Read DB │
  │(Postgres)│    │       │    │ Handler  │    │(Elastic)│
  └─────────┘    └───────┘    └──────────┘    └─────────┘
  
  Latência: ~50-500ms (eventual consistency)
  
  Vantagem: desacoplado, resiliente
  Desvantagem: delay entre write e read
```

### Via CDC (Change Data Capture)

```
Write DB ──▶ Debezium (CDC) ──▶ Kafka ──▶ Projection Handler ──▶ Read DB

  ┌─────────┐    ┌──────────┐    ┌───────┐    ┌─────────┐
  │Write DB │───▶│ Debezium │───▶│ Kafka │───▶│ Read DB │
  │(Postgres)│    │(reads WAL)│   │       │    │(Elastic)│
  └─────────┘    └──────────┘    └───────┘    └─────────┘
  
  Vantagem: não precisa modificar código da aplicação
  Desvantagem: captura mudanças low-level (not domain events)
```

### Via Materialized Views (mais simples)

```sql
-- PostgreSQL Materialized View
CREATE MATERIALIZED VIEW order_details AS
SELECT 
    o.id AS order_id,
    u.name AS customer_name,
    o.total,
    o.status,
    COUNT(oi.id) AS items_count,
    o.created_at
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items oi ON o.id = oi.order_id
GROUP BY o.id, u.name, o.total, o.status, o.created_at;

-- Refresh periodicamente
REFRESH MATERIALIZED VIEW CONCURRENTLY order_details;

-- Simples mas NÃO escala para volumes Big Tech
```

---

## CQRS + Event Sourcing

```
Combinação poderosa e frequente:

  Command ──▶ Aggregate ──▶ Event Store (append-only)
                                    │
                               ┌────┴────┐
                               ▼         ▼
                          Projection  Projection
                          (Orders)    (Analytics)
                               │         │
                               ▼         ▼
                          Read DB 1   Read DB 2
                          (Elastic)   (ClickHouse)

  Event Store = Write DB (source of truth)
  Read DBs = Projeções construídas a partir dos eventos
  
  Vantagens combinadas:
  - Audit trail completo (Event Sourcing)
  - Leituras otimizadas (CQRS)
  - Múltiplas views dos mesmos dados
  - Time-travel (replay events para novo Read Model)
```

---

## Múltiplas Read Models

```
Um dos grandes benefícios do CQRS: mesmos dados, múltiplas views:

  Events
    │
    ├──▶ Projection A → Elasticsearch (full-text search)
    │    "orders by keyword, status, date range"
    │
    ├──▶ Projection B → Redis (cache de dashboards)
    │    "total orders today, revenue, top customers"
    │
    ├──▶ Projection C → ClickHouse (analytics)
    │    "monthly trends, cohort analysis"
    │
    └──▶ Projection D → Neo4j (graph queries)
         "customers who also bought X"

  Cada Read Model:
  - Usa o DB ideal para o tipo de query
  - Escala independentemente
  - Pode ser reconstruída do zero (replay events)
```

---

## Quando Usar (e Quando NÃO)

### Use CQRS Quando

```
✅ Read e Write têm escalas drasticamente diferentes
   (ex: 100:1 read:write ratio)
   
✅ Read e Write models são muito diferentes
   (ex: writes normalizados, reads desnormalizados com JOINs)
   
✅ Precisa de múltiplas views otimizadas dos mesmos dados
   (ex: search, dashboard, analytics, recommendations)
   
✅ Event Sourcing já está em uso
   (CQRS é complemento natural)
   
✅ Domínio complexo com muitas regras de negócio
   (Command side encapsula lógica; Query side é simples)
```

### NÃO Use CQRS Quando

```
❌ CRUD simples onde read e write models são iguais
   (overhead de complexidade sem benefício)
   
❌ Eventual consistency é inaceitável
   (CQRS implica delay entre write e read)
   
❌ Time pequeno sem experiência em EDA
   (curva de aprendizado significativa)
   
❌ Volume baixo que não justifica separação
   (PostgreSQL com índices resolve)
   
❌ Dados simples sem necessidade de múltiplas views
```

---

## Trade-offs

| Aspecto | Prós | Contras |
|---------|------|---------|
| **Performance** | Read e Write otimizados separadamente | Eventual consistency entre models |
| **Escalabilidade** | Escala reads e writes independente | Infraestrutura mais complexa |
| **Flexibilidade** | Múltiplas views / DBs diferentes | Schema evolution em eventos |
| **Domain Model** | Command side rico em lógica | Duplicação de dados (read models) |
| **Resiliência** | Read e Write falham independente | Projection lag (delay na leitura) |
| **Manutenção** | Separação clara de responsabilidades | Mais código (handlers, projections) |

---

## Consistency Handling

```
Problema: User cria order e imediatamente tenta ver → Read Model ainda não atualizou

Soluções:

1. Read-your-own-writes:
   - Após command, retorna dados do Write DB para o mesmo client
   - Reads subsequentes do Read Model (já atualizado)
   
2. Polling com version:
   - Command retorna version/sequence number
   - Client polls Read Model até version >= expected
   
3. WebSocket notification:
   - Projection atualizada → notifica client via WS
   - UI atualiza quando Read Model pronto
   
4. Sync projection para UI crítica:
   - Projection específica atualizada SINCRONA no command handler
   - Troca latência por consistência (use com moderação)
```

---

## Uso em Big Techs

| Empresa | Write Side | Read Side | Detalhes |
|---------|-----------|-----------|----------|
| **LinkedIn** | MySQL sharded | Espresso (read-optimized) | Feed storage separado de feed delivery |
| **Twitter/X** | MySQL (tweets) | Redis + cache layers | Timeline pré-computada (fan-out-on-write) |
| **Netflix** | Cassandra + EVCache | Elasticsearch + Redis | Catálogo: writes em Cassandra, search em Elastic |
| **Uber** | Schemaless (MySQL) | Cassandra + Elasticsearch | Trip data: write in Schemaless, query in Elastic |
| **Microsoft** | SQL Server | Azure Search, Cosmos DB | Pattern documentation + Azure Architecture Center |
| **Airbnb** | PostgreSQL | Elasticsearch | Listings: writes em PG, search em Elastic |

### LinkedIn Feed — CQRS na Prática

```
Write Path (Command):
  User posts → Write to MySQL (posts table, normalized)
  → Publish "PostCreated" event to Kafka

Fan-out (Projection):
  PostCreated event → Fan-out service
  → Pre-compute timeline for each follower
  → Write to Espresso (read-optimized store)

Read Path (Query):
  User opens feed → Read from Espresso
  → Denormalized, pre-sorted, paginated
  → ~10ms response time

  Write: MySQL (ACID, normalized)
  Read: Espresso (denormalized, indexed, replicated)
  Sync: Kafka events → fan-out projection
```

---

## Perguntas Frequentes em Entrevistas

1. **"O que é CQRS?"**
   - Separar modelo de escrita (Command) do modelo de leitura (Query)
   - Write: normalizado, regras de negócio
   - Read: desnormalizado, otimizado para queries

2. **"Qual a diferença para CRUD simples?"**
   - CRUD: mesmo modelo para read e write (compromisso)
   - CQRS: modelos separados, cada um otimizado

3. **"Como sincronizar Write e Read models?"**
   - Eventos via Kafka/message bus (mais comum)
   - CDC com Debezium
   - Materialized views (mais simples, menos escalável)

4. **"CQRS precisa de Event Sourcing?"**
   - Não obrigatoriamente, mas são complementares
   - CQRS sem ES: Write DB normal + eventos para read model
   - CQRS com ES: Event Store como Write DB + projections como Read Models

5. **"Quando NÃO usar CQRS?"**
   - CRUD simples, volume baixo, equipe inexperiente
   - Quando eventual consistency é inaceitável
   - Quando read e write models são iguais

---

## Referências

- Greg Young (2010) — *"CQRS Documents"* — cqrs.files.wordpress.com
- Martin Fowler — *"CQRS"* — martinfowler.com
- Microsoft — *"CQRS Pattern"* — Azure Architecture Center
- Chris Richardson — *"Microservices Patterns"*, Cap. CQRS
- Vaughn Vernon — *"Implementing Domain-Driven Design"*
- Udi Dahan — *"Clarified CQRS"*
