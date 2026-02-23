# 15. Pub/Sub (Publish-Subscribe)

> **Categoria:** Comunicação Assíncrona e Event-Driven Architecture  
> **Nível:** Essencial para qualquer entrevista de System Design  
> **Complexidade:** Média

---

## Definição

**Pub/Sub (Publish-Subscribe)** é um padrão de messaging onde **publishers** enviam mensagens para **tópicos** sem saber quem são os subscribers, e **subscribers** recebem mensagens de tópicos sem saber quem são os publishers. O **broker** intermedia a comunicação, garantindo desacoplamento total entre produtores e consumidores.

---

## Por Que é Importante?

- **Desacoplamento total** — Publisher não conhece os subscribers (e vice-versa)
- **Fan-out nativo** — Uma mensagem entregue a TODOS os subscribers de um tópico
- **Extensibilidade** — Adicionar novos subscribers sem alterar publishers
- **Event-driven architecture** — Base para sistemas reativos e orientados a eventos
- **Escalabilidade** — Subscribers processam independentemente e podem escalar individualmente
- **Fundamento de microservices** — Comunicação assíncrona entre domínios

---

## Diagrama de Arquitetura

```
  Pub/Sub Básico:
  
  ┌─────────────┐         ┌────────────────┐         ┌──────────────┐
  │ Publisher 1  │──msg──▶│                │──msg──▶ │ Subscriber A │
  └─────────────┘         │                │         └──────────────┘
                          │  Topic:        │
  ┌─────────────┐         │  "order.events"│         ┌──────────────┐
  │ Publisher 2  │──msg──▶│                │──msg──▶ │ Subscriber B │
  └─────────────┘         │                │         └──────────────┘
                          │                │
                          │                │         ┌──────────────┐
                          │                │──msg──▶ │ Subscriber C │
                          └────────────────┘         └──────────────┘
  
  → Publisher publica no tópico
  → TODOS os subscribers recebem uma CÓPIA da mensagem
  → Subscribers são independentes entre si
```

### Múltiplos Tópicos

```
  ┌────────────┐     ┌─────────────────┐     ┌───────────────────┐
  │ Order      │────▶│ Topic: orders   │────▶│ Billing Service   │
  │ Service    │     └─────────────────┘  ┌─▶│                   │
  └────────────┘            │             │  └───────────────────┘
                            │             │
                            ├─────────────┤  ┌───────────────────┐
                            │             ├─▶│ Notification Svc  │
  ┌────────────┐     ┌──────▼──────────┐  │  └───────────────────┘
  │ Payment    │────▶│ Topic: payments │──┘
  │ Service    │     └─────────────────┘     ┌───────────────────┐
  └────────────┘            │            ├──▶│ Analytics Service │
                            │            │   └───────────────────┘
                            └────────────┘
                                             ┌───────────────────┐
  ┌────────────┐     ┌─────────────────┐ ├──▶│ Audit Service     │
  │ Inventory  │────▶│ Topic: inventory│─┘   └───────────────────┘
  │ Service    │     └─────────────────┘
  └────────────┘
  
  → Cada serviço publica em seu tópico
  → Subscribers escolhem de quais tópicos receber
  → Analytics e Audit assinam TODOS os tópicos
```

---

## Queue vs Pub/Sub — Diferença Fundamental

```
  ┌─────────────────────────────────────────────────────────────┐
  │  QUEUE (Point-to-Point):                                    │
  │                                                             │
  │  Producer ──▶ [Queue] ──▶ Consumer A  ✅ (recebe msg1)     │
  │                       ╳── Consumer B  ❌ (não recebe msg1)  │
  │                       ╳── Consumer C  ❌ (não recebe msg1)  │
  │                                                             │
  │  → UMA mensagem para UM consumer (competing consumers)     │
  │  → Usado para: work distribution, task processing          │
  ├─────────────────────────────────────────────────────────────┤
  │  PUB/SUB (Publish-Subscribe):                               │
  │                                                             │
  │  Publisher ──▶ [Topic] ──▶ Subscriber A  ✅ (cópia msg1)   │
  │                        ──▶ Subscriber B  ✅ (cópia msg1)   │
  │                        ──▶ Subscriber C  ✅ (cópia msg1)   │
  │                                                             │
  │  → UMA mensagem para TODOS os subscribers                  │
  │  → Usado para: event broadcasting, notifications           │
  └─────────────────────────────────────────────────────────────┘
```

| Aspecto | Queue | Pub/Sub |
|---------|-------|---------|
| **Entrega** | Para UM consumer | Para TODOS subscribers |
| **Mensagem após consumo** | Removida da fila | Mantida (para outros subs) |
| **Scaling** | + consumers = + throughput | + subscribers = + fan-out |
| **Use case** | Task distribution | Event broadcasting |
| **Acoplamento** | Consumer conhece a fila | Desacoplamento total |
| **Analogia** | Fila do banco (próximo!) | Rádio FM (todos ouvem) |

---

## Kafka Consumer Groups — O Melhor dos Dois Mundos

```
  Kafka combina Queue + Pub/Sub:
  
  Topic: "orders" (3 partições)
  
  Consumer Group A (Billing):     Consumer Group B (Analytics):
  ┌─────────────────────────┐    ┌─────────────────────────┐
  │ C1 ← P0               │    │ C1 ← P0, P1            │
  │ C2 ← P1               │    │ C2 ← P2                │
  │ C3 ← P2               │    └─────────────────────────┘
  └─────────────────────────┘
  
  DENTRO do mesmo grupo → Queue behavior (competing consumers)
    C1 recebe msg de P0, C2 recebe msg de P1 (não duplica)
  
  ENTRE grupos diferentes → Pub/Sub behavior (broadcast)
    Billing E Analytics recebem TODAS as mensagens
  
  ┌──────────────────────────────────────────────────────────┐
  │  Mensagem publicada em "orders":                         │
  │  → Billing Group: UM consumer processa                   │
  │  → Analytics Group: UM consumer processa                 │
  │  → Notification Group: UM consumer processa              │
  │                                                          │
  │  Resultado: cada grupo processa TODAS as mensagens,      │
  │  mas distribui o trabalho DENTRO do grupo                │
  └──────────────────────────────────────────────────────────┘
```

---

## Modelos de Subscription

### 1. Topic-Based (Mais Comum)

```
  Subscriber assina tópicos específicos:
  
  Subscriber A: subscribe("orders")
  Subscriber B: subscribe("payments")
  Subscriber C: subscribe("orders", "payments", "inventory")
  
  → Simples e direto
  → Usado por: Kafka, Google Pub/Sub, AWS SNS
```

### 2. Content-Based (Filtering)

```
  Subscriber define FILTROS sobre o conteúdo da mensagem:
  
  Subscriber A: subscribe(topic="orders", filter="amount > 1000")
  Subscriber B: subscribe(topic="orders", filter="region = 'US'")
  Subscriber C: subscribe(topic="orders")  // todas
  
  Mensagem: { topic: "orders", amount: 5000, region: "US" }
  → Subscriber A recebe ✅ (amount > 1000)
  → Subscriber B recebe ✅ (region = US)
  → Subscriber C recebe ✅ (sem filtro)
  
  Mensagem: { topic: "orders", amount: 50, region: "BR" }
  → Subscriber A NÃO recebe ❌
  → Subscriber B NÃO recebe ❌
  → Subscriber C recebe ✅
  
  Suportado por: AWS SNS (filter policies), Google Pub/Sub (filters)
```

### 3. Pattern-Based (Wildcard)

```
  RabbitMQ Topic Exchange:
  
  Routing key: "order.created.us"
  
  Subscriber A: bind("order.created.*")     → ✅ recebe
  Subscriber B: bind("order.#")             → ✅ recebe (# = zero ou mais words)
  Subscriber C: bind("payment.created.*")   → ❌ não recebe
  Subscriber D: bind("*.created.*")         → ✅ recebe
  
  Wildcards:
  * = exatamente UMA word
  # = zero ou mais words
```

---

## Delivery Guarantees em Pub/Sub

### At-Least-Once (Padrão)

```
  ┌───────────┐    ┌───────┐    ┌──────────────┐
  │ Publisher │──▶ │ Topic │──▶ │ Subscriber   │
  └───────────┘    └───────┘    └──────┬───────┘
                       │               │
                       │ timeout!      │ processou ✅
                       │ sem ACK       │ mas crash antes
                       │               │ do ACK ╳
                       ▼               │
                   re-deliver ─────────┘
                   → subscriber recebe DE NOVO
                   → deve ser IDEMPOTENTE
```

### Ordering em Pub/Sub

```
  Problema: mensagens de publishers diferentes podem chegar desordenadas
  
  Publisher A (t=1): publish("order.created", {id: 123})
  Publisher B (t=2): publish("order.paid", {id: 123})
  
  Subscriber pode receber:
  ✅ order.created → order.paid (correto)
  ❌ order.paid → order.created (desordenado!)
  
  Soluções:
  → Google Pub/Sub: ordering key (mensagens com mesma key são ordenadas)
  → Kafka: partition key
  → AWS SNS+SQS FIFO: message group ID
```

---

## Tecnologias de Pub/Sub

### Comparativo

| Tecnologia | Tipo | Persistence | Ordering | Fan-out | Filtering |
|------------|------|-------------|----------|---------|-----------|
| **Apache Kafka** | Distributed Log | Sim (configurable) | Per-partition | Consumer groups | No (application-level) |
| **Google Pub/Sub** | Managed | Sim (7 dias) | Ordering key | Nativo | Sim (attribute filters) |
| **AWS SNS** | Managed | Não (delivery only) | FIFO topics | Nativo | Sim (filter policies) |
| **AWS SNS + SQS** | Combo | Sim (via SQS) | FIFO queues | SNS fan-out | Sim |
| **Redis Pub/Sub** | In-memory | Não (fire-forget) | No | Nativo | Pattern subscribe |
| **Redis Streams** | In-memory | Sim (RDB/AOF) | Global | Consumer groups | No |
| **RabbitMQ** | Broker | Sim (durable) | Per-queue | Exchanges | Routing keys |
| **Apache Pulsar** | Distributed Log | Sim (tiered) | Per-partition | Subscriptions | No |
| **NATS** | Lightweight | JetStream | Per-subject | Nativo | Subject hierarchy |

### AWS SNS + SQS (Fan-out Pattern)

```
  Padrão mais comum na AWS para Pub/Sub:
  
  ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────────────┐
  │ Publisher │─────▶│   SNS    │─────▶│  SQS 1   │─────▶│ Billing Consumer │
  └──────────┘      │  Topic   │      └──────────┘      └──────────────────┘
                    │          │
                    │          │      ┌──────────┐      ┌──────────────────┐
                    │          │─────▶│  SQS 2   │─────▶│ Analytics Consumer│
                    │          │      └──────────┘      └──────────────────┘
                    │          │
                    │          │      ┌──────────┐      ┌──────────────────┐
                    │          │─────▶│  SQS 3   │─────▶│ Notification Svc │
                    └──────────┘      └──────────┘      └──────────────────┘
  
  SNS fan-out → SQS para buffering e retry → Consumer processa
  
  Benefícios:
  → SNS: broadcast para N subscribers
  → SQS: buffering, retry, DLQ, visibility timeout
  → Cada consumer independente (pode falhar sem afetar outros)
  → Filter Policies: cada SQS recebe apenas msgs relevantes
```

### Google Cloud Pub/Sub

```
  ┌──────────┐      ┌──────────┐      ┌───────────────┐      ┌──────────────┐
  │ Publisher │─────▶│  Topic   │─────▶│ Subscription A │─────▶│ Subscriber A │
  └──────────┘      │          │      │  (pull)        │      └──────────────┘
                    │          │      └───────────────┘
                    │          │
                    │          │      ┌───────────────┐      ┌──────────────┐
                    │          │─────▶│ Subscription B │─────▶│ Subscriber B │
                    │          │      │  (push)        │      │ (HTTP endpoint)│
                    └──────────┘      └───────────────┘      └──────────────┘
  
  Subscription = "durable subscriber"
  → Pull: consumer controla o ritmo
  → Push: Pub/Sub envia para HTTP endpoint
  → Message retention: 7 dias por padrão
  → Ordering: via ordering key
  → Exactly-once: per-subscription deduplication
```

### Redis Pub/Sub

```python
import redis

r = redis.Redis()

# Publisher
r.publish('orders', '{"id": 123, "status": "created"}')

# Subscriber
pubsub = r.pubsub()
pubsub.subscribe('orders')       # Tópico exato
pubsub.psubscribe('order.*')     # Pattern matching

for message in pubsub.listen():
    if message['type'] == 'message':
        print(f"Received: {message['data']}")
```

```
  ⚠️ Redis Pub/Sub publica FIRE-AND-FORGET:
  → Se subscriber está offline, mensagem é PERDIDA
  → Sem persistência, sem replay, sem ACK
  → Bom para: real-time notifications, cache invalidation
  → NÃO bom para: critical events, reliable delivery
  
  Para persistência → usar Redis Streams (XADD/XREAD)
```

---

## Implementação — Event Bus com Pub/Sub

### Java + Spring Boot — Event Publishing

```java
// Domain Event
public record OrderCreatedEvent(
    String orderId,
    String userId,
    BigDecimal amount,
    Instant timestamp
) {}

// Publisher (Application Service)
@Service
public class OrderService {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        
        // Publish domain event
        eventPublisher.publishEvent(new OrderCreatedEvent(
            order.getId(),
            order.getUserId(),
            order.getAmount(),
            Instant.now()
        ));
        
        return order;
    }
}

// Subscriber 1: Billing
@Component
public class BillingEventHandler {
    
    @EventListener
    @Async
    public void handleOrderCreated(OrderCreatedEvent event) {
        billingService.createInvoice(event.orderId(), event.amount());
    }
}

// Subscriber 2: Notification
@Component
public class NotificationEventHandler {
    
    @EventListener
    @Async
    public void handleOrderCreated(OrderCreatedEvent event) {
        notificationService.sendOrderConfirmation(
            event.userId(), event.orderId()
        );
    }
}

// Subscriber 3: Analytics
@Component
public class AnalyticsEventHandler {
    
    @EventListener
    @Async
    public void handleOrderCreated(OrderCreatedEvent event) {
        analyticsService.trackOrderCreated(event);
    }
}
```

### Node.js + AWS SNS/SQS

```typescript
import { SNSClient, PublishCommand } from "@aws-sdk/client-sns";
import { SQSClient, ReceiveMessageCommand, DeleteMessageCommand } from "@aws-sdk/client-sqs";

// Publisher → SNS
const sns = new SNSClient({ region: "us-east-1" });

async function publishOrderEvent(order: Order) {
  await sns.send(new PublishCommand({
    TopicArn: "arn:aws:sns:us-east-1:123456:order-events",
    Message: JSON.stringify({
      eventType: "ORDER_CREATED",
      orderId: order.id,
      amount: order.amount,
      timestamp: new Date().toISOString(),
    }),
    MessageAttributes: {
      eventType: {
        DataType: "String",
        StringValue: "ORDER_CREATED",
      },
      region: {
        DataType: "String",
        StringValue: order.region,
      },
    },
  }));
}

// Subscriber (SQS consumer) — Billing
const sqs = new SQSClient({ region: "us-east-1" });

async function pollMessages() {
  while (true) {
    const response = await sqs.send(new ReceiveMessageCommand({
      QueueUrl: "https://sqs.us-east-1.amazonaws.com/123456/billing-queue",
      MaxNumberOfMessages: 10,
      WaitTimeSeconds: 20,  // long polling
      VisibilityTimeout: 60,
    }));

    for (const message of response.Messages ?? []) {
      try {
        const event = JSON.parse(JSON.parse(message.Body!).Message);
        await processOrderEvent(event);
        
        // ACK: delete message
        await sqs.send(new DeleteMessageCommand({
          QueueUrl: "https://sqs.us-east-1.amazonaws.com/123456/billing-queue",
          ReceiptHandle: message.ReceiptHandle!,
        }));
      } catch (error) {
        console.error("Failed to process message:", error);
        // Message becomes visible again after VisibilityTimeout
      }
    }
  }
}
```

---

## Patterns com Pub/Sub

### Event Notification

```
  Simples: evento informa que ALGO aconteceu, sem detalhes.
  
  Publisher: { event: "order.created", orderId: "123" }
  
  Subscriber A: Recebe → faz GET /orders/123 para obter detalhes
  Subscriber B: Recebe → faz GET /orders/123 para obter detalhes
  
  ✅ Prós: Payload pequeno, publisher simples
  ❌ Contras: Subscribers fazem requests extras (carga no publisher)
```

### Event-Carried State Transfer

```
  Evento carrega TODOS os dados necessários:
  
  Publisher: { 
    event: "order.created", 
    orderId: "123",
    userId: "456",
    items: [...],
    total: 99.90,
    address: {...}
  }
  
  Subscriber: Recebe → tem tudo que precisa, sem requests extras
  
  ✅ Prós: Subscribers autônomos, sem roundtrip
  ❌ Contras: Payload maior, publisher precisa incluir tudo
  ⭐ Recomendado para microservices (reduz acoplamento)
```

### Event Sourcing + Pub/Sub

```
  Eventos são a fonte da verdade E são publicados via Pub/Sub:
  
  Command: CreateOrder
  ↓
  Event Store: [OrderCreated, ItemAdded, PaymentReceived]
  ↓
  Pub/Sub: Cada evento é publicado para subscribers
  ↓
  Subscriber A (Read Model): Materializa view para queries
  Subscriber B (Analytics): Processa para dashboards
  Subscriber C (Notification): Envia emails
  
  → Event Store = source of truth
  → Pub/Sub = distribuição dos eventos
  → CQRS: write model gera eventos, read model consome
```

### Choreography (Saga com Pub/Sub)

```
  Saga com eventos — sem orquestrador central:
  
  1. Order Service: publish("order.created")
  2. Payment Service: subscribe("order.created")
     → Processa pagamento
     → publish("payment.completed")
  3. Inventory Service: subscribe("payment.completed")
     → Reserva estoque
     → publish("inventory.reserved")
  4. Shipping Service: subscribe("inventory.reserved")
     → Cria envio
     → publish("shipping.started")
  
  ┌─────────┐ order.created  ┌─────────┐ payment.ok  ┌───────────┐
  │  Order  │──────────────▶│ Payment │────────────▶│ Inventory │
  └─────────┘               └─────────┘             └─────┬─────┘
       ▲                                                   │
       │                    inventory.reserved             │
       │              ┌──────────────────────────────────┘
       │              ▼
       │        ┌──────────┐
       └────────│ Shipping │  shipping.started
                └──────────┘
  
  Compensação (se algo falhar):
  → Payment falha → publish("payment.failed")
  → Order Service: subscribe("payment.failed") → cancela order
```

---

## Desafios e Soluções

### 1. Subscriber Lento

```
  Problema: Subscriber C é 10x mais lento que A e B
  
  Soluções:
  → Buffer individual (SQS por subscriber)
  → Backpressure (subscriber controla ritmo)
  → Scaling (mais instâncias do subscriber lento)
  → Dead Letter Queue para mensagens que timeout
```

### 2. Ordering Across Topics

```
  Problema: Evento em "orders" e "payments" precisam ser processados em ordem
  
  Publisher 1: "orders" → { orderId: 123, event: "created" }
  Publisher 2: "payments" → { orderId: 123, event: "paid" }
  
  Subscriber pode receber "paid" antes de "created"!
  
  Soluções:
  → Tópico único com partitioning
  → Sequence numbers + reordering buffer
  → State machine no subscriber (ignora evento se estado inválido)
```

### 3. Exactly-Once Delivery

```
  ⚠️ Exactly-once end-to-end é MUITO difícil

  Na prática:
  → At-least-once delivery + idempotência no subscriber
  
  Idempotência:
  const processEvent = async (event) => {
    // Dedup by event ID
    const exists = await db.query(
      "SELECT 1 FROM processed_events WHERE event_id = $1",
      [event.id]
    );
    if (exists) return; // já processou, skip
    
    await db.transaction(async (tx) => {
      await tx.query(
        "INSERT INTO processed_events (event_id) VALUES ($1)",
        [event.id]
      );
      await processBusinessLogic(event, tx);
    });
  };
```

---

## Uso em Big Techs

### Google — Cloud Pub/Sub
- **Scale:** Trilhões de mensagens por dia
- **Use cases:** Pipelines de dados, analytics em real-time, IoT
- **Diferencial:** Global, serverless, exactly-once per subscription
- **Integração:** Dataflow, BigQuery, Cloud Functions

### Uber — Event Bus
- Event bus baseado em Kafka para comunicação entre microservices
- Milhares de tópicos para diferentes domínios (trips, payments, drivers)
- AVRO schema registry para evolução de schema
- Dead Letter Queues para eventos problemáticos

### Stripe — Webhooks como Pub/Sub
- Eventos de pagamento publicados para integrators via webhooks
- At-least-once delivery com retry exponential
- Subscribers (merchants) registram HTTP endpoints
- Assinatura por tipo de evento (payment_intent.succeeded, etc.)

### Netflix — Event-Driven Microservices
- Kafka como backbone de comunicação entre centenas de microservices
- Domain events para desacoplamento entre equipes
- Consumer groups para diferentes visões dos mesmos eventos
- Schema evolution com Avro + Schema Registry

### Slack — Real-time Messaging
- Pub/Sub para distribuição de mensagens em channels
- WebSocket connections como subscribers
- Redis Pub/Sub para notificações real-time
- Kafka para persistência e replay

### Twitter/X — Timeline Fan-out
- Quando um usuário posta, evento é publicado
- Fan-out: distribui para timelines de todos os followers
- Celebridades (high fanout): read-time fan-out em vez de write-time
- Kafka para event streaming entre services

---

## Perguntas Comuns em Entrevistas

1. **Qual a diferença entre Queue e Pub/Sub?**
   - Queue: mensagem para UM consumer (competing). Pub/Sub: mensagem para TODOS subscribers (broadcast).

2. **Como Kafka implementa Pub/Sub?**
   - Consumer Groups: dentro do grupo é queue, entre grupos é pub/sub. Cada grupo recebe todas as mensagens.

3. **Como garantir que um subscriber não perca mensagens?**
   - Durable subscription (SQS, Kafka consumer group, Google Pub/Sub subscription) — mensagens persistem até serem ACK'd.

4. **Queue vs Pub/Sub — quando usar cada?**
   - Queue: task distribution (um worker processa). Pub/Sub: event notification (todos precisam saber).

5. **Como lidar com fan-out para milhões de subscribers?**
   - Hierarquia de tópicos, batching, push vs pull, regional fan-out. Ex: Twitter celebridade → fan-out on read.

6. **Descreva o padrão SNS + SQS da AWS.**
   - SNS broadcast para múltiplas SQS queues. Cada SQS é um subscriber com buffering, retry e DLQ independentes.

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Modelo** | Topic-based (simples) | Content-based filtering (granular) |
| **Delivery** | Push (baixa latência) | Pull (consumer controla ritmo) |
| **Persistência** | Durable (replay, recovery) | Fire-and-forget (baixa latência) |
| **Ordering** | Ordering key (per-key order) | Unordered (maior throughput) |
| **Payload** | Event Notification (pequeno) | State Transfer (autônomo) |
| **Coordination** | Choreography (desacoplado) | Orchestration (centralizado) |
| **Filtering** | Broker-side (eficiente) | Client-side (flexível) |
| **Platform** | Managed (SNS/GCP Pub/Sub) | Self-hosted (Kafka/RabbitMQ) |

---

## Referências

- [Google Cloud Pub/Sub Documentation](https://cloud.google.com/pubsub/docs)
- [AWS SNS Developer Guide](https://docs.aws.amazon.com/sns/latest/dg/)
- [Apache Kafka — Consumer Groups](https://kafka.apache.org/documentation/#consumerconfigs)
- [Martin Fowler — Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
- [Enterprise Integration Patterns — Hohpe & Woolf](https://www.enterpriseintegrationpatterns.com/)
- [Martin Kleppmann — Designing Data-Intensive Applications](https://dataintensive.net/) — Cap. 11
- System Design Interview — Alex Xu, Vol. 1
