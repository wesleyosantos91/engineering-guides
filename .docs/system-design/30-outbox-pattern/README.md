# 30. Outbox Pattern

> **Categoria:** Padrões Arquiteturais  
> **Nível:** Avançado — essencial para consistência em Event-Driven  
> **Complexidade:** Média-Alta

---

## Definição

O **Outbox Pattern** resolve o problema de **dual write** — a necessidade de atualizar o banco de dados E publicar um evento de forma **atômica**. O padrão consiste em escrever o evento em uma **tabela outbox** dentro da mesma transação do banco, e um processo separado (CDC ou poller) lê essa tabela e publica os eventos no message broker.

---

## Por Que é Importante?

- **Problema mais comum** em Event-Driven Architecture — dual write causa inconsistências
- **Garantia de consistência** — DB e eventos sempre em sincronia
- **Padrão fundamental** — base para Sagas, CQRS, Event Sourcing confiáveis
- **Usado por todas as Big Techs** que fazem EDA
- **Pergunta de entrevista** — "como garantir que o evento é publicado quando o DB é atualizado?"

---

## O Problema: Dual Write

```
Cenário: ao criar um pedido, preciso:
  1. Salvar no banco de dados
  2. Publicar evento no Kafka

Abordagem ingênua:

  def create_order(cmd):
      order = Order(cmd)
      db.save(order)           # Step 1: salva no DB
      kafka.publish(event)     # Step 2: publica no Kafka ← PODE FALHAR!

Cenários de falha:

  A) DB salva OK, Kafka falha:
     → Pedido existe no DB mas ninguém sabe (evento perdido!)
     → Inventory não reserva, Payment não cobra
     
  B) Kafka publica OK, DB falha:  
     → Evento publicado mas pedido NÃO existe!
     → Inventory reserva para pedido fantasma
     
  C) Processo morre entre Step 1 e Step 2:
     → Estado inconsistente sem nenhuma indicação

  Ordem invertida não resolve:
  
  def create_order(cmd):
      kafka.publish(event)     # Step 1: publica primeiro
      db.save(order)           # Step 2: salva depois ← PODE FALHAR!
      
  → Mesmo problema, só muda a direção da inconsistência
```

---

## A Solução: Outbox Pattern

```
Ideia central: escrever evento na MESMA transação do DB

  ┌─────────── Transação ACID ───────────┐
  │                                       │
  │  1. INSERT INTO orders VALUES (...)   │  ← Salva pedido
  │  2. INSERT INTO outbox VALUES (...)   │  ← Salva evento
  │                                       │
  │  COMMIT                               │  ← Atômico!
  └───────────────────────────────────────┘
  
  3. Outbox Relay (processo separado) lê outbox e publica no Kafka
  
  ┌──────────┐    ┌──────────┐    ┌───────┐
  │ orders   │    │ outbox   │───▶│ Kafka │
  │ table    │    │ table    │    │       │
  └──────────┘    └──────────┘    └───────┘
       ▲               ▲               │
       │               │               ▼
       └── Same DB ────┘          Consumers
       (mesma transação)

  Garantia: se o pedido existe, o evento NA outbox também existe.
  Se o evento está na outbox, será publicado (at-least-once).
```

---

## Componentes

### 1. Outbox Table

```sql
CREATE TABLE outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(255) NOT NULL,    -- "Order"
    aggregate_id    VARCHAR(255) NOT NULL,    -- "order-123"
    event_type      VARCHAR(255) NOT NULL,    -- "OrderCreated"
    payload         JSONB NOT NULL,           -- event data
    created_at      TIMESTAMP DEFAULT NOW(),
    published       BOOLEAN DEFAULT FALSE,    -- para polling approach
    published_at    TIMESTAMP                 -- quando foi publicado
);

-- Índice para consulta eficiente
CREATE INDEX idx_outbox_unpublished 
    ON outbox (created_at) 
    WHERE published = FALSE;
```

### 2. Escrita Atômica

```java
@Service
public class OrderService {
    
    @Transactional  // ← MESMA transação para order + outbox
    public Order createOrder(CreateOrderCommand cmd) {
        // 1. Cria e salva o pedido
        Order order = Order.create(cmd);
        orderRepository.save(order);
        
        // 2. Salva evento na outbox (MESMA transação!)
        OutboxEvent event = OutboxEvent.builder()
            .aggregateType("Order")
            .aggregateId(order.getId())
            .eventType("OrderCreated")
            .payload(objectMapper.writeValueAsString(
                new OrderCreatedEvent(
                    order.getId(),
                    order.getCustomerId(),
                    order.getItems(),
                    order.getTotal()
                )
            ))
            .build();
        
        outboxRepository.save(event);
        
        return order;
    }
}

// Se a transação rola back → AMBOS são desfeitos ✅
// Se a transação commita   → AMBOS são persistidos ✅
```

### 3. Outbox Relay (Publicador)

Duas abordagens para ler a outbox e publicar:

---

## Abordagem 1: Polling Publisher

```
Processo background consulta outbox periodicamente:

  Every 100ms:
    1. SELECT * FROM outbox WHERE published = FALSE ORDER BY created_at LIMIT 100
    2. Para cada evento: kafka.publish(event)
    3. UPDATE outbox SET published = TRUE WHERE id IN (...)
    4. Opcionalmente: DELETE FROM outbox WHERE published = TRUE AND age > 7 days
```

### Implementação

```java
@Component
public class OutboxPollingPublisher {
    
    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    
    @Scheduled(fixedDelay = 100) // Poll a cada 100ms
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> events = outboxRepository
            .findByPublishedFalseOrderByCreatedAt(
                PageRequest.of(0, 100)
            );
        
        for (OutboxEvent event : events) {
            try {
                // Publica no Kafka
                kafkaTemplate.send(
                    topicFor(event.getAggregateType()),
                    event.getAggregateId(), // partition key
                    event.getPayload()
                ).get(); // Espera confirmação
                
                // Marca como publicado
                event.setPublished(true);
                event.setPublishedAt(Instant.now());
                outboxRepository.save(event);
                
            } catch (Exception e) {
                log.error("Failed to publish event {}", event.getId(), e);
                // Será retentado no próximo poll
                break; // Preserva ordering
            }
        }
    }
    
    // Cleanup de eventos antigos já publicados
    @Scheduled(cron = "0 0 2 * * *") // Daily às 2am
    public void cleanupPublishedEvents() {
        outboxRepository.deleteByPublishedTrueAndCreatedAtBefore(
            Instant.now().minus(7, ChronoUnit.DAYS)
        );
    }
}
```

### Trade-offs do Polling

```
Prós:
  ✅ Simples de implementar
  ✅ Sem dependência de infra adicional
  ✅ Funciona com qualquer banco relacional

Contras:
  ❌ Latência (depende do intervalo de polling)
  ❌ Carga no DB (queries constantes, mesmo sem eventos)
  ❌ Trade-off: polling frequente = mais carga / infrequente = mais latência
  ❌ Precisa gerenciar ordering e at-least-once
```

---

## Abordagem 2: CDC (Change Data Capture) com Debezium

```
CDC lê o WAL (Write-Ahead Log) do banco e publica automaticamente:

  Application       PostgreSQL          Debezium         Kafka
      │                  │                  │              │
      │── INSERT ──▶     │                  │              │
      │  (order + outbox)│                  │              │
      │                  │                  │              │
      │              WAL write ──▶          │              │
      │                  │     CDC reads ──▶│              │
      │                  │        WAL       │── publish ──▶│
      │                  │                  │              │
      
  Debezium monitora o WAL do PostgreSQL (ou binlog do MySQL)
  Quando row aparece na outbox → Debezium publica no Kafka
  Latência: ~milliseconds (muito menor que polling)
```

### Configuração Debezium

```json
{
  "name": "outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "...",
    "database.dbname": "orders_db",
    "database.server.name": "orders",
    
    "table.include.list": "public.outbox",
    
    "transforms": "outbox",
    "transforms.outbox.type": 
        "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.route.by.field": "aggregate_type",
    "transforms.outbox.route.topic.replacement": "${routedByValue}-events",
    "transforms.outbox.table.field.event.id": "id",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.table.field.event.type": "event_type",
    "transforms.outbox.table.expand.json.payload": true,
    
    "transforms.outbox.table.fields.additional.placement": 
        "event_type:header:eventType"
  }
}
```

### Como Debezium Funciona

```
PostgreSQL WAL (Write-Ahead Log):

  ┌─────────────────────────────────────────────┐
  │ LSN: 0/1234    INSERT outbox                │
  │   id: evt-001                               │
  │   aggregate_type: "Order"                   │
  │   aggregate_id: "order-123"                 │
  │   event_type: "OrderCreated"                │
  │   payload: {"orderId": "order-123", ...}    │
  └─────────────────────────────────────────────┘
                    │
                    ▼
  Debezium (reads WAL via logical replication slot)
                    │
                    ▼
  Kafka topic: "Order-events"
    Key: "order-123"
    Value: {"orderId": "order-123", ...}
    Headers: { eventType: "OrderCreated" }
```

### Trade-offs do CDC

```
Prós:
  ✅ Latência muito baixa (~ms)
  ✅ Zero carga no DB (lê WAL, sem queries)
  ✅ Ordering garantido (WAL é ordered)
  ✅ At-least-once nativamente
  ✅ Não modifica código da aplicação

Contras:
  ❌ Infraestrutura adicional (Debezium, Kafka Connect)
  ❌ Complexidade operacional
  ❌ Dependência de features do DB (WAL/binlog)
  ❌ Debugging mais difícil (processo assíncrono fora da app)
```

---

## Polling vs CDC

| Aspecto | Polling | CDC (Debezium) |
|---------|---------|----------------|
| **Latência** | 100ms-5s (depende do intervalo) | ~10-50ms |
| **Carga no DB** | Queries constantes | Zero (lê WAL) |
| **Complexidade** | Baixa (código simples) | Média (infra Debezium) |
| **Infraestrutura** | Nenhuma adicional | Debezium + Kafka Connect |
| **Ordering** | Possível (com cuidado) | Garantido (WAL order) |
| **Operacional** | Simples | Mais complexo |
| **Scale** | Limitado (polling overhead) | Excelente |
| **Ideal para** | MVPs, volume baixo-médio | Produção, volume alto |

---

## Garantias de Entrega

```
At-least-once delivery:

  Cenário: Outbox Relay publica no Kafka, mas falha ao marcar published=true
  
  Polling:
    1. Publica evento no Kafka ✅
    2. Marca published=true... CRASH! ❌
    3. Próximo poll: encontra evento novamente
    4. Publica novamente → DUPLICATA
  
  CDC:
    1. Debezium lê WAL e publica ✅
    2. Commit offset... CRASH! ❌
    3. Restart: relê WAL desde último offset
    4. Publica novamente → DUPLICATA
  
  Solução: CONSUMERS DEVEM SER IDEMPOTENTES!
  
  Consumer idempotente:
    processed_events = { evt-001, evt-002, ... }
    
    if event.id in processed_events:
        skip  ← já processou, ignora duplicata
    else:
        process(event)
        processed_events.add(event.id)
```

---

## Outbox Pattern Completo

```
Fluxo end-to-end:

  Client
    │
    ▼
  API Gateway
    │
    ▼
  Order Service
    │
    │  @Transactional
    │  ┌────────────────────────────┐
    │  │ INSERT INTO orders (...)   │
    │  │ INSERT INTO outbox (...)   │
    │  │ COMMIT                     │
    │  └────────────────────────────┘
    │
    │                    ┌──────────────────┐
    │                    │ PostgreSQL       │
    │                    │ ┌──────────────┐ │
    │                    │ │ orders table │ │
    │                    │ └──────────────┘ │
    │                    │ ┌──────────────┐ │
    │                    │ │ outbox table │───── WAL ──▶ Debezium
    │                    │ └──────────────┘ │              │
    │                    └──────────────────┘              │
    │                                                     │
    │                                               ┌─────▼─────┐
    │                                               │   Kafka    │
    │                                               │ (events)   │
    │                                               └─────┬──────┘
    │                                                     │
    │                                        ┌────────────┼────────────┐
    │                                        ▼            ▼            ▼
    │                                   Inventory    Payment    Notification
    │                                   Service      Service    Service
```

---

## Variações

### 1. Inbox Pattern (para o Consumer)

```
Problema: Consumer recebe at-least-once → precisa deduplicar.

  Consumer Inbox Table:
  ┌──────────────────────────────────────┐
  │ inbox                                │
  ├──────────────┬──────────┬────────────┤
  │ event_id     │ UUID     │ PK/UNIQUE  │
  │ event_type   │ VARCHAR  │            │
  │ payload      │ JSONB    │            │
  │ processed    │ BOOLEAN  │            │
  │ received_at  │ TIMESTAMP│            │
  └──────────────┴──────────┴────────────┘

  Consumer flow:
  1. Recebe evento do Kafka
  2. INSERT INTO inbox (event_id, ...) ON CONFLICT DO NOTHING
  3. Se inserted (new): processa evento
  4. Se conflict (duplicate): ignora

  Outbox + Inbox = reliable messaging end-to-end!
```

### 2. Transactional Outbox com diferentes DBs

```
PostgreSQL: WAL + Debezium (logical replication)
MySQL:      Binlog + Debezium
MongoDB:    Change Streams (nativo)
DynamoDB:   DynamoDB Streams → Lambda → Kafka
Cosmos DB:  Change Feed → Azure Functions → Event Hub

Cada DB tem seu mecanismo de CDC:
  DB-specific ──▶ CDC mechanism ──▶ Message Broker
```

### 3. Listen/Notify (PostgreSQL-specific)

```sql
-- Approach mais simples para PostgreSQL:

-- Trigger que notifica quando outbox tem novo evento
CREATE OR REPLACE FUNCTION notify_outbox()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('outbox_channel', NEW.id::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER outbox_notify
    AFTER INSERT ON outbox
    FOR EACH ROW EXECUTE FUNCTION notify_outbox();

-- Application listener (em vez de polling):
-- Recebe notificação imediata quando há novo evento
-- Latência ~0ms (muito melhor que polling)
-- Mas: pg_notify não é durável (se falhar, perde)
-- → Use como complemento ao polling (notifica + fallback poll)
```

---

## Uso em Big Techs

| Empresa | Implementação | CDC Tool | Detalhes |
|---------|---------------|----------|----------|
| **Uber** | CDC-based | Custom | Evento-driven payment processing |
| **Netflix** | CDC + custom | DBLog | Open-source CDC framework do Netflix |
| **Airbnb** | Outbox + polling | Custom | SpinalTap (MySQL CDC) |
| **Stripe** | Outbox pattern | Custom | Garantia de entrega de payment events |
| **LinkedIn** | CDC | Databus → Brook | Databus (deprecated) → Brook |
| **Zalando** | Outbox + Nakadi | Debezium | Nakadi = event bus open-source |
| **WePay (Chase)** | CDC | Debezium | Blog post referência para Debezium + Outbox |

### Netflix DBLog

```
Netflix criou DBLog para CDC sem depender de WAL:

  Regular approach:
    PostgreSQL WAL → Debezium → Kafka
    (depende de features específicas do DB)

  Netflix DBLog:
    1. Full dump inicial (para bootstrap)
    2. Change capture via triggers ou WAL
    3. Watermark-based approach para consistency
    
    Vantagem: funciona com qualquer DB
    Open-source: github.com/Netflix/DBLog
```

---

## Perguntas Frequentes em Entrevistas

1. **"O que é o Outbox Pattern?"**
   - Salva evento em tabela outbox na mesma transação do DB
   - Processo separado lê outbox e publica no message broker
   - Resolve dual write (DB + broker atômico)

2. **"Qual o problema que resolve?"**
   - Dual write: não é possível fazer DB write + Kafka publish atomicamente
   - Se um falha e outro sucede → inconsistência
   - Outbox garante: se DB write ocorreu, evento SERÁ publicado

3. **"Polling vs CDC?"**
   - Polling: simples, mais latência, carga no DB
   - CDC (Debezium): baixa latência, zero carga, mais infra
   - Produção Big Tech geralmente usa CDC

4. **"O que é Debezium?"**
   - Open-source CDC platform (Red Hat)
   - Lê WAL/binlog do DB e publica no Kafka
   - Kafka Connect connector
   - Suporta PostgreSQL, MySQL, MongoDB, Oracle, SQL Server

5. **"Como garantir exactly-once?"**
   - Outbox garante at-least-once (evento pode duplicar)
   - Consumers DEVEM ser idempotentes (deduplicação por event_id)
   - Inbox Pattern no consumer para deduplicação
   - Exactly-once é at-least-once + idempotent consumer

---

## Referências

- Chris Richardson — *"Microservices Patterns"*, Cap. Transactional Messaging
- Debezium Documentation — debezium.io — Outbox Event Router
- Gunnar Morling — *"Reliable Microservices Data Exchange With the Outbox Pattern"*
- Netflix Engineering — *"DBLog: A Watermark Based Change-Data-Capture Framework"*
- Pat Helland — *"Life beyond Distributed Transactions: an Apostate's Opinion"* — CIDR 2007
- Vaughn Vernon — *"Implementing Domain-Driven Design"*, Cap. messaging
