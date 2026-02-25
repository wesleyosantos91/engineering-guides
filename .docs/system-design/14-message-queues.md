# 14. Message Queues

> **Categoria:** Comunicação Assíncrona e Desacoplamento  
> **Nível:** Essencial para qualquer entrevista de System Design  
> **Complexidade:** Média-Alta (conceito simples, garantias de entrega são complexas)

---

## Definição

**Message Queue** é um middleware de comunicação assíncrona que permite que **producers** enviem mensagens para uma fila e **consumers** as processem em um ritmo independente. A fila atua como um **buffer** entre serviços, desacoplando produtores e consumidores temporal e espacialmente.

---

## Por Que é Importante?

- **Desacoplamento** — Serviços não precisam se conhecer ou estar online ao mesmo tempo
- **Resiliência** — Mensagens persistem na fila mesmo se o consumer falhar
- **Rate leveling** — Absorve picos de tráfego (buffer de requests)
- **Escalabilidade** — Adicionar consumers para aumentar throughput (scale-out)
- **Async processing** — Operações demoradas (envio de email, processamento de vídeo) saem do fluxo principal
- **Fundamento de microservices** — Uma das principais formas de comunicação inter-serviço

---

## Diagrama de Arquitetura

```
  Modelo Básico — Point-to-Point:
  
  ┌──────────┐         ┌─────────────────┐         ┌──────────┐
  │ Producer │──msg──▶ │     Queue       │──msg──▶ │ Consumer │
  │ (API)    │         │ ┌─────────────┐ │         │ (Worker) │
  └──────────┘         │ │ msg4        │ │         └──────────┘
                       │ │ msg3        │ │
                       │ │ msg2        │ │
                       │ │ msg1        │ │
                       │ └─────────────┘ │
                       └─────────────────┘
  
  Propriedades:
  → FIFO (First-In, First-Out) — geralmente
  → Cada mensagem é processada por UM consumer
  → Mensagem é removida após ACK do consumer
```

### Modelo com Competing Consumers

```
  Scale-out: múltiplos consumers disputam mensagens
  
  ┌──────────┐         ┌──────────────┐         ┌────────────┐
  │Producer 1│──msg──▶ │              │──msg──▶ │ Consumer 1 │
  └──────────┘         │              │         └────────────┘
                       │    Queue     │
  ┌──────────┐         │              │         ┌────────────┐
  │Producer 2│──msg──▶ │  msg5        │──msg──▶ │ Consumer 2 │
  └──────────┘         │  msg4        │         └────────────┘
                       │  msg3        │
  ┌──────────┐         │  msg2        │         ┌────────────┐
  │Producer 3│──msg──▶ │  msg1        │──msg──▶ │ Consumer 3 │
  └──────────┘         │              │         └────────────┘
                       └──────────────┘
  
  → msg1 → Consumer 1 (ACK ✅)
  → msg2 → Consumer 2 (ACK ✅)
  → msg3 → Consumer 3 (processing...)
  → msg4 → Consumer 1 (next available)
  
  Cada mensagem é processada por exatamente UM consumer
```

### Modelo com Dead Letter Queue (DLQ)

```
  ┌──────────┐       ┌──────────┐       ┌──────────┐
  │ Producer │──────▶│  Queue   │──────▶│ Consumer │
  └──────────┘       └────┬─────┘       └────┬─────┘
                          │                   │
                          │ retry 3x falhou   │ processing
                          │                   │ failed!
                          ▼                   │
                    ┌────────────┐            │
                    │   DLQ      │◀───────────┘
                    │ (poison    │
                    │  messages) │  → Análise manual
                    └────────────┘  → Alertas
                                    → Reprocessamento
```

---

## Garantias de Entrega

| Garantia | Descrição | Trade-off | Quando Usar |
|----------|-----------|-----------|-------------|
| **At-most-once** | Mensagem pode ser perdida, nunca duplicada | Sem retry, fire-and-forget | Logs, métricas não-críticas |
| **At-least-once** | Mensagem nunca é perdida, pode ser duplicada | Retry + ACK, consumer deve ser idempotente | ⭐ Maioria dos casos |
| **Exactly-once** | Mensagem nunca é perdida nem duplicada | Complexo, transacional | Financeiro, Kafka Streams |

### At-Most-Once

```
  Producer envia → Queue recebe → Consumer processa
  
  Se consumer falha ANTES de processar:
  → Mensagem já foi removida da fila → PERDIDA
  
  ┌──────────┐    ┌───────┐    ┌──────────┐
  │ Producer │──▶ │ Queue │──▶ │ Consumer │
  └──────────┘    └───────┘    └──────────┘
                  (remove msg            ╳ crash!
                   ao entregar)     msg perdida
```

### At-Least-Once

```
  Producer envia → Queue recebe → Consumer processa → ACK
  
  Se consumer falha ANTES do ACK:
  → Queue re-entrega a mensagem → pode haver DUPLICAÇÃO
  
  ┌──────────┐    ┌───────┐    ┌──────────┐
  │ Producer │──▶ │ Queue │──▶ │ Consumer │
  └──────────┘    └───────┘    └────┬─────┘
                      ▲             │ processa ✅
                      │             │ crash antes
                      │             │ do ACK! ╳
                      │             │
                      └─── re-deliver msg
                           → consumer processa DE NOVO
                           → potencial duplicação!
  
  Solução: Consumer IDEMPOTENTE
  
  // Idempotência com deduplication ID
  if (processedIds.contains(msg.id)) {
      ack(msg);  // já processou, ignore
      return;
  }
  process(msg);
  processedIds.add(msg.id);
  ack(msg);
```

### Exactly-Once (Kafka)

```
  Kafka Transactions (Producer + Consumer):
  
  Producer:
  producer.initTransactions();
  producer.beginTransaction();
  producer.send(record);
  producer.commitTransaction();  // atômico
  
  Consumer (read-process-write pattern):
  consumer.poll() → process → producer.send(output)
  → tudo dentro de UMA transação Kafka
  
  Internally:
  → Producer usa sequence numbers para deduplicar
  → Consumer usa offsets transacionais
  → Brokers coordenam via transaction coordinator
  
  ⚠️ Exactly-once é DENTRO do Kafka.
  ⚠️ Side effects externos (DB, API) ainda precisam de idempotência.
```

---

## Padrões de Consumo

### 1. Point-to-Point (Queue)

```
  Uma mensagem → um consumer
  
  Queue: [msg1, msg2, msg3]
  
  Consumer A recebe msg1 → processa → ACK
  Consumer B recebe msg2 → processa → ACK
  Consumer C recebe msg3 → processa → ACK
  
  Cada mensagem é processada por EXATAMENTE um consumer
  → Usado para Work Queue / Task Distribution
```

### 2. Competing Consumers

```
  Múltiplos consumers no mesmo grupo disputam mensagens
  
  Queue: [msg1, msg2, msg3, msg4, msg5, msg6]
  
  Consumer Pool (3 workers):
  → Consumer 1: msg1, msg4 (round-robin)
  → Consumer 2: msg2, msg5
  → Consumer 3: msg3, msg6
  
  → Scale-out: adicionar consumers para aumentar throughput
  → Se Consumer 2 falha: msg5 é re-entregue a Consumer 1 ou 3
```

### 3. Fan-out

```
  Uma mensagem → múltiplos consumers (via exchange/topic)
  
  Producer ──▶ Exchange/Topic ──┬──▶ Queue A ──▶ Consumer A (email)
                                ├──▶ Queue B ──▶ Consumer B (analytics)
                                └──▶ Queue C ──▶ Consumer C (audit log)
  
  → Cada consumer recebe uma CÓPIA da mensagem
  → Processamento independente em paralelo
  → Usado para notificações, event-driven architecture
```

### 4. Request-Reply

```
  Producer envia request → Consumer processa → Reply via reply queue
  
  ┌──────────┐    Request Queue    ┌──────────┐
  │  Client  │────────────────────▶│  Server  │
  │          │    Reply Queue      │          │
  │          │◀────────────────────│          │
  └──────────┘                     └──────────┘
  
  Correlation ID: vincula request ao reply
  → Usado quando precisa de resposta assíncrona
```

---

## Message Ordering

### O Desafio

```
  Sem ordering garantido:
  
  Producer envia: [order_created, order_paid, order_shipped]
  
  Consumer pode receber: [order_paid, order_created, order_shipped]
  → Tentou pagar order que não existe! ❌
```

### Soluções

| Abordagem | Como | Trade-off |
|-----------|------|-----------|
| **Single partition** | Todas as msgs na mesma partição/fila | Sem paralelismo |
| **Partition key** | Msgs do mesmo entity na mesma partição | Balanceado |
| **Sequence number** | Consumer reordena por sequence | Mais complexo |
| **Saga / State machine** | Consumer valida estado antes de processar | Robusto |

### Kafka — Ordering com Partition Key

```
  Topic "orders" com 4 partições
  Partition key = order_id
  
  order:123 → hash("123") % 4 = 2 → Partição 2
  order:456 → hash("456") % 4 = 0 → Partição 0
  order:123 → hash("123") % 4 = 2 → Partição 2 (mesma!)
  
  Partição 0: [order:456_created, order:456_paid]
  Partição 2: [order:123_created, order:123_paid, order:123_shipped]
  
  ✅ Mensagens do MESMO order_id são sempre ordenadas
  ✅ Diferentes orders podem ser processados em paralelo
```

---

## Visibility Timeout & Acknowledgment

```
  Fluxo de processamento com SQS-style visibility timeout:
  
  ┌──────────┐    ┌─────────────────────────────┐    ┌──────────┐
  │ Producer │──▶ │         Queue               │──▶ │ Consumer │
  └──────────┘    │                             │    └──────────┘
                  │  msg1 [visible]             │
                  │  msg2 [visible]             │
                  │  msg3 [invisible: 30s]      │ ← Consumer processando
                  │  msg4 [visible]             │
                  └─────────────────────────────┘
  
  1. Consumer recebe msg3 → msg3 fica INVISIBLE por 30s
  2. Consumer processa msg3
  3a. Consumer envia ACK/DELETE → msg3 removida permanentemente ✅
  3b. Consumer falha → após 30s, msg3 volta a ser VISIBLE → re-delivery
  
  ⚠️ Se processamento demora > visibility timeout:
  → msg3 fica visible de novo → OUTRO consumer recebe → duplicação!
  → Solução: Extend visibility timeout durante processamento
```

---

## Backpressure & Rate Leveling

```
  Sem Queue (síncrono):
  
  Pico de 10,000 req/s ──▶ Backend (capacity: 2,000 req/s) → 💥 CRASH
  
  ──────────────────────────────────────────────────────────
  
  Com Queue (assíncrono):
  
  Pico de 10,000 req/s ──▶ [Queue buffer] ──▶ Backend (2,000 req/s)
                            (absorve pico)     (processa no seu ritmo)
  
  ┌─────────────────────────────────────────────────────────┐
  │  Tráfego:  ████████████████                             │
  │  Backend:  ████████                                     │
  │  Queue:    ████████ (acumulando)                        │
  │            └─ drena conforme backend processa           │
  └─────────────────────────────────────────────────────────┘
  
  → Queue absorve spikes
  → Backend processa a taxa constante
  → Nenhum request é perdido
```

---

## Tecnologias

### Comparativo

| Tecnologia | Tipo | Modelo | Ordering | Throughput | Persistence |
|------------|------|--------|----------|------------|-------------|
| **Apache Kafka** | Distributed Log | Pull | Per-partition | Muito alto (millions/s) | Disco (retenção configurável) |
| **RabbitMQ** | Message Broker | Push/Pull | Por fila (FIFO) | Alto (10-50K/s) | Opcional (durable queues) |
| **AWS SQS** | Managed Queue | Pull | Standard: best-effort; FIFO: garantido | Alto | Managed (4-14 dias) |
| **AWS SNS** | Fan-out Pub/Sub | Push | Não garante | Alto | Não (delivery only) |
| **Redis Streams** | In-memory Stream | Pull | Global | Muito alto | RDB/AOF |
| **Apache Pulsar** | Distributed Log | Pull/Push | Per-partition | Muito alto | Tiered storage |
| **Google Pub/Sub** | Managed Pub/Sub | Pull/Push | Não por padrão | Muito alto | Managed |
| **NATS** | Lightweight Broker | Push | Per-subject | Extremamente alto | JetStream (opcional) |
| **ActiveMQ** | Message Broker | Push/Pull | Por fila | Médio | Sim (KahaDB) |

### Apache Kafka — Deep Dive

```
  Arquitetura:
  
  ┌─────────────────────────────────────────────────────────────┐
  │                      Kafka Cluster                         │
  │                                                             │
  │  Topic: "orders" (3 partições, RF=3)                       │
  │                                                             │
  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                    │
  │  │Broker 1 │  │Broker 2 │  │Broker 3 │                    │
  │  │         │  │         │  │         │                    │
  │  │ P0(L)   │  │ P0(F)   │  │ P0(F)   │  P0: Partition 0  │
  │  │ P1(F)   │  │ P1(L)   │  │ P1(F)   │  L: Leader        │
  │  │ P2(F)   │  │ P2(F)   │  │ P2(L)   │  F: Follower      │
  │  └─────────┘  └─────────┘  └─────────┘                    │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
  
  Producers ──▶ Partition Leaders
  Consumers ◀── Partition Leaders (ou Followers com KIP-392)
  
  Conceitos-chave:
  → Topic: stream de mensagens categorizadas
  → Partition: unidade de paralelismo e ordering
  → Offset: posição da mensagem na partição
  → Consumer Group: conjunto de consumers que dividem partições
  → Replication Factor: número de cópias de cada partição
```

```
  Consumer Groups:
  
  Topic: "orders" (4 partições: P0, P1, P2, P3)
  
  Consumer Group A (3 consumers):        Consumer Group B (2 consumers):
  ┌─────────────────────────────┐       ┌─────────────────────────────┐
  │ C1 ← P0, P1                │       │ C1 ← P0, P1                │
  │ C2 ← P2                    │       │ C2 ← P2, P3                │
  │ C3 ← P3                    │       └─────────────────────────────┘
  └─────────────────────────────┘
  
  → Dentro do grupo: cada partição vai para UM consumer (competing)
  → Entre grupos: todos recebem TODAS as mensagens (pub/sub)
  → Se consumer falha → rebalance: partições redistribuídas
```

### RabbitMQ — Deep Dive

```
  Arquitetura:
  
  ┌──────────┐     ┌─────────────────────────────┐     ┌──────────┐
  │ Producer │────▶│         RabbitMQ             │────▶│ Consumer │
  └──────────┘     │                             │     └──────────┘
                   │  ┌──────────┐  ┌─────────┐ │
                   │  │ Exchange │──│ Binding │──── Queue ──▶ Consumer
                   │  │          │  │ (rules) │ │
                   │  │ Types:   │  └─────────┘ │
                   │  │ • direct │               │
                   │  │ • fanout │  ┌─────────┐ │
                   │  │ • topic  │──│ Binding │──── Queue ──▶ Consumer
                   │  │ • headers│  └─────────┘ │
                   │  └──────────┘               │
                   └─────────────────────────────┘
  
  Exchange Types:
  → direct:  routing_key exato (1:1)
  → fanout:  todas as queues (broadcast)
  → topic:   routing_key com wildcards (*.order.#)
  → headers: match por headers da mensagem
```

### AWS SQS — Standard vs FIFO

```
  Standard Queue:
  ┌──────────────────────────────────────┐
  │ • At-least-once delivery             │
  │ • Best-effort ordering               │
  │ • Nearly unlimited throughput        │
  │ • Pode duplicar mensagens            │
  │                                      │
  │ Use: background jobs, notifications  │
  └──────────────────────────────────────┘
  
  FIFO Queue:
  ┌──────────────────────────────────────┐
  │ • Exactly-once processing            │
  │ • Strict ordering (por group ID)     │
  │ • 300 msg/s (3000 com batching)      │
  │ • Deduplication (5 min window)       │
  │                                      │
  │ Use: financial transactions, orders  │
  └──────────────────────────────────────┘
```

---

## Implementação — Producer/Consumer Pattern

### Java + Spring Boot + Kafka

```java
// Producer
@Service
public class OrderEventProducer {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderCreated(Order order) {
        OrderEvent event = new OrderEvent("ORDER_CREATED", order);
        
        // Partition key = orderId → garante ordering por order
        kafkaTemplate.send("orders", order.getId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish event", ex);
                    // Retry or save to outbox table
                } else {
                    log.info("Published to partition {} offset {}",
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}

// Consumer
@Service
public class OrderEventConsumer {
    
    @KafkaListener(topics = "orders", groupId = "order-processor")
    public void consume(OrderEvent event, Acknowledgment ack) {
        try {
            // Idempotência: verificar se já processou
            if (processedEventStore.exists(event.getId())) {
                ack.acknowledge();
                return;
            }
            
            // Processar evento
            orderService.processEvent(event);
            
            // Salvar como processado
            processedEventStore.save(event.getId());
            
            // ACK manual
            ack.acknowledge();
            
        } catch (RetryableException e) {
            // Não faz ACK → Kafka re-entrega
            throw e;
        } catch (Exception e) {
            // Envia para DLQ
            dlqProducer.send("orders-dlq", event);
            ack.acknowledge();  // ACK para não ficar em loop
        }
    }
}
```

### Python + RabbitMQ (pika)

```python
import pika
import json

# Producer
def publish_message(queue_name: str, message: dict):
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    # Declare queue (durable = survives broker restart)
    channel.queue_declare(queue=queue_name, durable=True)
    
    channel.basic_publish(
        exchange='',
        routing_key=queue_name,
        body=json.dumps(message),
        properties=pika.BasicProperties(
            delivery_mode=2,  # persistent message
            content_type='application/json',
        )
    )
    
    connection.close()

# Consumer
def consume_messages(queue_name: str, callback):
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    
    channel.queue_declare(queue=queue_name, durable=True)
    channel.basic_qos(prefetch_count=1)  # fair dispatch
    
    def on_message(ch, method, properties, body):
        try:
            message = json.loads(body)
            callback(message)
            ch.basic_ack(delivery_tag=method.delivery_tag)
        except Exception as e:
            # Negative ACK → re-queue
            ch.basic_nack(
                delivery_tag=method.delivery_tag,
                requeue=True  # ou False para DLQ
            )
    
    channel.basic_consume(
        queue=queue_name,
        on_message_callback=on_message,
        auto_ack=False  # manual ACK
    )
    
    channel.start_consuming()
```

---

## Patterns Avançados

### Outbox Pattern (Reliable Publishing)

```
  Problema: Como garantir que a DB e a queue estão em sync?
  
  ❌ Problema:
  1. Save to DB ✅
  2. Publish to Queue ❌ (falhou!)
  → DB tem dados, queue não → inconsistência
  
  ✅ Outbox Pattern:
  1. Save to DB + Save to Outbox Table (MESMA transação)
  2. Background worker lê Outbox Table → publica na Queue
  3. Marca como publicado na Outbox Table
  
  ┌────────────────────────────┐
  │ Transaction:               │
  │   INSERT INTO orders ...   │
  │   INSERT INTO outbox ...   │
  │ COMMIT                     │
  └─────────────┬──────────────┘
                │
  ┌─────────────▼──────────────┐
  │ Outbox Worker (poll/CDC):  │
  │   SELECT * FROM outbox     │
  │   WHERE published = false  │
  │   → Publish to Queue       │
  │   → UPDATE published=true  │
  └────────────────────────────┘
```

### Retry com Exponential Backoff

```
  Tentativa 1: falhou → retry em 1s
  Tentativa 2: falhou → retry em 2s
  Tentativa 3: falhou → retry em 4s
  Tentativa 4: falhou → retry em 8s
  Tentativa 5: falhou → envia para DLQ
  
  delay = min(base * 2^attempt, max_delay) + random_jitter
  
  ┌────────┐   retry 1   ┌────────┐   retry 2   ┌────────┐
  │ Queue  │────(1s)────▶│ Queue  │────(2s)────▶│ Queue  │──...──▶ DLQ
  │        │   failed     │        │   failed     │        │
  └────────┘              └────────┘              └────────┘
```

### Priority Queue

```
  Mensagens com diferentes prioridades:
  
  ┌─────────────────────────────┐
  │ Priority Queue              │
  │                             │
  │ [HIGH]   msg: payment_fail  │ ← processado primeiro
  │ [HIGH]   msg: fraud_alert   │
  │ [MEDIUM] msg: order_update  │
  │ [LOW]    msg: analytics     │
  │ [LOW]    msg: notification  │ ← processado por último
  └─────────────────────────────┘
  
  Suportado por: RabbitMQ (x-max-priority), ActiveMQ
  Kafka: usar topics separados por prioridade
```

---

## Uso em Big Techs

### LinkedIn — Apache Kafka (criadores)
- Kafka foi criado no LinkedIn para processar activity stream data
- Trilhões de mensagens por dia
- Use cases: activity tracking, metrics, log aggregation
- Open-sourced em 2011, Apache top-level em 2012

### Netflix — Kafka + SQS
- Kafka para event sourcing e telemetria em real-time
- SQS para async workflows de encoding de vídeo
- Processamento de bilhões de eventos por dia
- Dead Letter Queues para mensagens problemáticas

### Uber — Kafka para Trip Lifecycle
- Cada trip gera dezenas de eventos (request, match, start, end, payment)
- Kafka garante ordering por trip_id (partition key)
- Consumer groups separados: billing, analytics, ETA, fraud
- Processam milhões de trips/dia com Kafka

### Stripe — Event-Driven Architecture
- Webhooks baseados em message queues
- Retry com exponential backoff para webhook delivery
- At-least-once delivery + idempotency keys para consumers
- Garantia de entrega de eventos de pagamento

### Slack — Message Processing
- RabbitMQ/Kafka para processamento assíncrono de mensagens
- Fan-out para notificações push, email, desktop
- Priority queues para mensagens críticas vs bulk

---

## Perguntas Comuns em Entrevistas

1. **Quando usar message queue vs chamada síncrona?**
   - Queue: operações demoradas, desacoplamento, tolerância a falhas. Síncrono: baixa latência, request-response simples.

2. **Qual a diferença entre at-least-once e exactly-once?**
   - At-least-once: consumer pode receber duplicatas (precisa de idempotência). Exactly-once: sem duplicatas (mais complexo, Kafka Streams).

3. **Como garantir ordering em uma message queue?**
   - Partition key: mensagens do mesmo entity vão para mesma partição. Dentro da partição, ordering é garantido.

4. **O que é uma Dead Letter Queue?**
   - Fila para mensagens que falharam após N tentativas. Permite análise, alertas e reprocessamento manual.

5. **Kafka vs RabbitMQ — quando usar cada um?**
   - Kafka: alto throughput, event sourcing, replay, stream processing. RabbitMQ: routing complexo, priority queues, request-reply.

6. **Como lidar com mensagens duplicadas?**
   - Idempotência no consumer: deduplication ID, upserts, check-then-act com mutex.

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Delivery** | At-least-once (simples + idempotência) | Exactly-once (complexo, Kafka) |
| **Ordering** | Per-partition (escalável) | Global FIFO (bottleneck) |
| **Tecnologia** | Kafka (throughput, replay) | RabbitMQ (routing, simplicity) |
| **Managed** | AWS SQS/SNS (zero-ops) | Self-hosted Kafka (controle total) |
| **Push vs Pull** | Pull (consumer controla ritmo) | Push (menor latência) |
| **Persistence** | Durable (seguro, mais lento) | In-memory (rápido, pode perder) |
| **ACK** | Manual (seguro) | Auto (simples, pode perder) |
| **DLQ** | Sim (resiliente) | Não (msgs perdidas em falha) |

---

## Referências

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/tutorials)
- [AWS SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/)
- [Martin Kleppmann — Designing Data-Intensive Applications](https://dataintensive.net/) — Cap. 11
- [Kafka: The Definitive Guide — O'Reilly](https://www.oreilly.com/library/view/kafka-the-definitive/9781492043072/)
- System Design Interview — Alex Xu, Vol. 1 & Vol. 2
