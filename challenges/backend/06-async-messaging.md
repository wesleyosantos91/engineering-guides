# Level 6 — Mensageria Assíncrona

> **Objetivo:** Integrar Kafka como message broker para processamento assíncrono de eventos de transação, notificações e auditoria, aplicando os padrões idiomáticos de cada stack.

---

## Objetivo de Aprendizado

- Implementar producer/consumer Kafka
- Aplicar event-driven architecture para domínios desacoplados
- Implementar dead letter queue (DLQ) para mensagens com falha
- Garantir at-least-once delivery com idempotência no consumer
- Implementar outbox pattern para consistência entre DB e mensageria
- Processar notificações assíncronas (email/push simulado)
- Entender particionamento, consumer groups e offset management

---

## Escopo Funcional

### Eventos Publicados

```
── Transações ──
TransactionCreated   → após depósito, saque ou transferência
TransactionFailed    → ao falhar (saldo insuficiente, etc.)

── Usuários ──
UserRegistered       → após registro de novo usuário
UserDeactivated      → após soft delete de usuário

── Wallets ──
WalletCreated        → após criação de wallet
LowBalanceAlert      → saldo abaixo de threshold configurável
```

### Tópicos Kafka

| Tópico | Producer | Consumer(s) |
|---|---|---|
| `wallet.transactions` | TransactionService | NotificationConsumer, AuditConsumer |
| `wallet.users` | UserService / AuthService | NotificationConsumer |
| `wallet.wallets` | WalletService | AuditConsumer |
| `wallet.notifications` | NotificationConsumer | EmailConsumer (simulado) |
| `wallet.dlq` | qualquer consumer com falha | DLQ monitor (log) |

### Regras de Negócio

1. **Después de cada transação**, publicar evento com `transactionId`, `walletId`, `type`, `amount`, `balanceAfter`, `timestamp`
2. **NotificationConsumer** recebe `TransactionCreated` → cria notificação simulada (log ou banco)
3. **AuditConsumer** recebe todos os eventos → persiste em tabela de auditoria
4. **Idempotência**: consumer usa `eventId` como chave de deduplicação — processar no máximo 1 vez
5. **DLQ**: após 3 retries de um event, enviar para tópico `wallet.dlq`
6. **LowBalanceAlert**: se saldo após transação < R$ 10.00, publicar alerta

---

## Escopo Técnico

### Formato de Evento

```json
{
  "eventId": "uuid",
  "eventType": "TRANSACTION_CREATED",
  "aggregateId": "wallet-uuid",
  "aggregateType": "WALLET",
  "timestamp": "2025-01-15T10:30:00Z",
  "payload": {
    "transactionId": "uuid",
    "walletId": "uuid",
    "type": "DEPOSIT",
    "amount": 100.00,
    "balanceAfter": 350.00,
    "currency": "BRL"
  },
  "metadata": {
    "correlationId": "trace-id",
    "userId": "uuid"
  }
}
```

### Kafka Client por Stack

| Stack | Producer | Consumer | Serialização |
|---|---|---|---|
| **Go (Gin)** | segmentio/kafka-go | segmentio/kafka-go (consumer group) | JSON (`encoding/json`) |
| **Spring Boot** | Spring Kafka (`KafkaTemplate`) | `@KafkaListener` | JSON (Jackson) com `JsonSerializer` |
| **Quarkus** | SmallRye Reactive Messaging (`@Outgoing`) | `@Incoming` | JSON (Jackson) com `JsonbSerializer` |
| **Micronaut** | Micronaut Kafka (`@KafkaClient`) | `@KafkaListener` | JSON (Jackson/Serde) |
| **Jakarta EE** | MicroProfile Reactive Messaging (`@Outgoing`) | `@Incoming` | JSON (JSON-B) |

### Schema Adicional

```sql
-- V6__create_audit_events.sql
CREATE TABLE audit_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL UNIQUE,    -- deduplicação
    event_type      VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    aggregate_type  VARCHAR(30) NOT NULL,
    payload         JSONB NOT NULL,
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_events_aggregate ON audit_events(aggregate_type, aggregate_id);
CREATE INDEX idx_audit_events_type ON audit_events(event_type);

-- V7__create_notifications.sql
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    type            VARCHAR(30) NOT NULL,    -- TRANSACTION, LOW_BALANCE, WELCOME
    title           VARCHAR(200) NOT NULL,
    message         TEXT NOT NULL,
    read            BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- V8__create_outbox.sql (para outbox pattern)
CREATE TABLE outbox_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(30) NOT NULL,
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
    payload         JSONB NOT NULL,
    published       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_outbox_unpublished ON outbox_events(published) WHERE published = false;
```

---

## Critérios de Aceite

- [ ] Após transação, evento publicado no tópico `wallet.transactions`
- [ ] NotificationConsumer recebe e cria notificação para o usuário
- [ ] AuditConsumer persiste evento na tabela `audit_events`
- [ ] Evento duplicado (mesmo `eventId`) é ignorado (idempotência)
- [ ] Após 3 falhas de processamento, evento vai para DLQ
- [ ] LowBalanceAlert publicado quando saldo < R$ 10.00
- [ ] Consumer group permite balanceamento entre instâncias
- [ ] Outbox pattern: evento salvo na mesma transação do banco
- [ ] Outbox poller/CDC publica eventos pendentes
- [ ] Testes de integração com Kafka embarcado (Testcontainers)

---

## Definição de Pronto (DoD)

- [ ] `docker-compose.yml` inclui Kafka + Zookeeper (ou KRaft)
- [ ] 3+ migrações adicionais (audit_events, notifications, outbox_events)
- [ ] Producer publica no formato de evento padronizado
- [ ] Consumer com retry + DLQ funcional
- [ ] Idempotência comprovada por teste
- [ ] Outbox pattern implementado com polling ou CDC
- [ ] Testes de integração com Testcontainers Kafka
- [ ] Commit: `feat(level-6): add Kafka messaging with outbox pattern`

---

## Checklist

### Go (Gin)

- [ ] `internal/platform/kafka/producer.go` — wrapper kafka-go producer
- [ ] `internal/platform/kafka/consumer.go` — consumer group wrapper
- [ ] `internal/event/types.go` — structs de evento (`TransactionEvent`, `UserEvent`, etc.)
- [ ] `internal/event/publisher.go` — interface `EventPublisher` + implementação Kafka
- [ ] `internal/event/handler/notification.go` — consumer que cria notificações
- [ ] `internal/event/handler/audit.go` — consumer que persiste auditoria
- [ ] `internal/event/handler/dlq.go` — consumer DLQ (apenas log)
- [ ] `internal/store/audit.go` — persistência de audit_events
- [ ] `internal/store/notification.go` — persistência de notifications
- [ ] `internal/store/outbox.go` — persistência de outbox + poller
- [ ] `internal/service/transaction.go` — modificar para publicar evento após transação
- [ ] `cmd/consumer/main.go` — processo separado para consumers

### Java — Spring Boot

- [ ] `pom.xml` — `spring-kafka`
- [ ] `core/config/KafkaConfig.java` — `ProducerFactory`, `ConsumerFactory`, tópicos
- [ ] `core/config/KafkaTopics.java` — constantes de tópicos
- [ ] `domain/event/TransactionEvent.java` — record
- [ ] `domain/event/UserEvent.java` — record
- [ ] `infrastructure/kafka/TransactionEventPublisher.java` — `KafkaTemplate.send()`
- [ ] `infrastructure/kafka/NotificationEventConsumer.java` — `@KafkaListener`
- [ ] `infrastructure/kafka/AuditEventConsumer.java` — `@KafkaListener`
- [ ] `infrastructure/entity/AuditEventEntity.java`
- [ ] `infrastructure/entity/NotificationEntity.java`
- [ ] `infrastructure/entity/OutboxEventEntity.java`
- [ ] `infrastructure/outbox/OutboxPoller.java` — `@Scheduled` polling
- [ ] `core/config/KafkaErrorConfig.java` — `DefaultErrorHandler` + DLQ

### Quarkus

- [ ] `@Outgoing("transactions-out")` em `Channel<TransactionEvent>`
- [ ] `@Incoming("transactions-in")` para consumer
- [ ] `application.properties` — `mp.messaging.outgoing.transactions-out.connector=smallrye-kafka`
- [ ] `@Acknowledgment(Strategy.POST_PROCESSING)` para at-least-once
- [ ] Dev Services para Kafka automático em testes

### Micronaut

- [ ] `@KafkaClient` interface para producer
- [ ] `@KafkaListener` para consumer
- [ ] `@Topic("wallet.transactions")` para especificar tópico
- [ ] `application.yml` — `kafka.bootstrap.servers`, `kafka.consumers.*`
- [ ] `@Retryable` no consumer para retry automático

### Jakarta EE

- [ ] `@Outgoing("transactions-out")` — MicroProfile Reactive Messaging
- [ ] `@Incoming("transactions-in")`
- [ ] `microprofile-config.properties` — `mp.messaging.outgoing.*.connector=smallrye-kafka`
- [ ] Alternative: JMS com ActiveMQ para runtime Jakarta EE puro

---

## Tarefas Sugeridas por Stack

### Go — Kafka Producer

```go
// internal/event/publisher.go
type EventPublisher interface {
    Publish(ctx context.Context, topic string, event Event) error
}

type kafkaPublisher struct {
    writer *kafka.Writer
    logger *zap.Logger
}

func NewKafkaPublisher(brokers []string, logger *zap.Logger) EventPublisher {
    return &kafkaPublisher{
        writer: &kafka.Writer{
            Addr:         kafka.TCP(brokers...),
            Balancer:     &kafka.LeastBytes{},
            BatchTimeout: 10 * time.Millisecond,
        },
        logger: logger,
    }
}

func (p *kafkaPublisher) Publish(ctx context.Context, topic string, event Event) error {
    payload, err := json.Marshal(event)
    if err != nil {
        return fmt.Errorf("marshal event: %w", err)
    }

    msg := kafka.Message{
        Topic: topic,
        Key:   []byte(event.AggregateID),
        Value: payload,
        Headers: []kafka.Header{
            {Key: "eventType", Value: []byte(event.EventType)},
            {Key: "correlationId", Value: []byte(event.Metadata.CorrelationID)},
        },
    }

    if err := p.writer.WriteMessages(ctx, msg); err != nil {
        p.logger.Error("failed to publish event",
            zap.String("topic", topic),
            zap.String("eventType", event.EventType),
            zap.Error(err),
        )
        return fmt.Errorf("publish to %s: %w", topic, err)
    }

    p.logger.Info("event published",
        zap.String("topic", topic),
        zap.String("eventId", event.EventID),
        zap.String("eventType", event.EventType),
    )
    return nil
}
```

### Go — Idempotent Consumer

```go
// internal/event/handler/audit.go
type AuditHandler struct {
    auditStore store.AuditStore
    logger     *zap.Logger
}

func (h *AuditHandler) Handle(ctx context.Context, event Event) error {
    // Idempotência: verificar se eventId já foi processado
    exists, err := h.auditStore.ExistsByEventID(ctx, event.EventID)
    if err != nil {
        return fmt.Errorf("check event exists: %w", err)
    }
    if exists {
        h.logger.Warn("duplicate event, skipping", zap.String("eventId", event.EventID))
        return nil // skip — já processado
    }

    auditEntry := model.AuditEvent{
        EventID:       event.EventID,
        EventType:     event.EventType,
        AggregateID:   event.AggregateID,
        AggregateType: event.AggregateType,
        Payload:       event.Payload,
    }

    return h.auditStore.Save(ctx, &auditEntry)
}
```

### Spring Boot — KafkaListener

```java
@Component
@RequiredArgsConstructor
public class NotificationEventConsumer {

    private final NotificationService notificationService;
    private final AuditEventRepository auditRepository;

    @KafkaListener(
        topics = "${kafka.topics.transactions}",
        groupId = "notification-group",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleTransactionEvent(@Payload TransactionEvent event,
                                        @Header(KafkaHeaders.RECEIVED_KEY) String key,
                                        Acknowledgment ack) {
        try {
            // Idempotência
            if (auditRepository.existsByEventId(event.eventId())) {
                ack.acknowledge();
                return;
            }

            notificationService.createTransactionNotification(event);
            ack.acknowledge();
        } catch (Exception e) {
            // Não acknowledge → retry automático
            throw e;
        }
    }
}
```

### Spring Boot — Outbox Pattern

```java
@Service
@Transactional
public class TransactionService {

    private final TransactionRepository transactionRepository;
    private final WalletRepository walletRepository;
    private final OutboxRepository outboxRepository;

    public TransactionResponse deposit(UUID walletId, DepositRequest request) {
        Wallet wallet = walletRepository.findByIdOrThrow(walletId);
        wallet.deposit(request.amount());

        Transaction tx = Transaction.createDeposit(wallet, request.amount());
        transactionRepository.save(tx);
        walletRepository.save(wallet);

        // Outbox — salvo na MESMA transação do banco
        OutboxEvent outbox = OutboxEvent.of(
            "WALLET", walletId.toString(),
            "TRANSACTION_CREATED",
            Map.of("transactionId", tx.getId(), "amount", request.amount(), "balanceAfter", wallet.getBalance())
        );
        outboxRepository.save(outbox);

        return mapper.toResponse(tx);
    }
}

// Poller publicador
@Component
@RequiredArgsConstructor
public class OutboxPoller {

    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 1000)
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> pending = outboxRepository.findByPublishedFalse();
        for (OutboxEvent event : pending) {
            kafkaTemplate.send(event.getTopic(), event.getAggregateId(), event.getPayload());
            event.markPublished();
            outboxRepository.save(event);
        }
    }
}
```

---

## docker-compose.yml (Kafka)

```yaml
services:
  # ... postgres, prometheus, grafana, jaeger (dos levels anteriores)

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:29093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:29093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_LOG_DIRS: /tmp/kraft-combined-logs
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    volumes:
      - kafka-data:/tmp/kraft-combined-logs

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
    depends_on:
      - kafka

volumes:
  kafka-data:
```

---

## Extensões Opcionais

- [ ] Implementar saga pattern para transferências distribuídas
- [ ] Adicionar Kafka Streams para agregação em tempo real
- [ ] Implementar consumer lag monitoring via Prometheus

---

## Extensão: AWS SNS/SQS (via LocalStack)

Além do Kafka (broker de eventos interno), implemente integração com **AWS SNS + SQS** para cenários de notificação e fan-out — típicos de arquiteturas cloud-native na AWS.

### Padrão Fan-Out: SNS → SQS

```
                         ┌──── SQS: notification-queue ──→ Notification Consumer
                         │
Transaction Event ──→ SNS Topic ──── SQS: audit-queue ──→ Audit Consumer
                         │
                         └──── SQS: analytics-queue ──→ Analytics Consumer
```

### Setup com LocalStack

```yaml
# docker-compose.yml
services:
  localstack:
    image: localstack/localstack:3.5
    ports:
      - "4566:4566"
    environment:
      SERVICES: sns,sqs,lambda
      DEFAULT_REGION: us-east-1
    volumes:
      - ./infra/localstack/init-aws.sh:/etc/localstack/init/ready.d/init-aws.sh
```

```bash
# infra/localstack/init-aws.sh
#!/bin/bash
awslocal sns create-topic --name wallet-transactions
awslocal sqs create-queue --queue-name notification-queue
awslocal sqs create-queue --queue-name audit-queue
awslocal sqs create-queue --queue-name analytics-queue

# Subscribe queues to SNS topic (fan-out)
TOPIC_ARN=$(awslocal sns list-topics --query 'Topics[0].TopicArn' --output text)
awslocal sns subscribe --topic-arn $TOPIC_ARN --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:000000000000:notification-queue
awslocal sns subscribe --topic-arn $TOPIC_ARN --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:000000000000:audit-queue
awslocal sns subscribe --topic-arn $TOPIC_ARN --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:000000000000:analytics-queue
```

### Implementação por Stack

| Stack | SNS/SQS Client |
|---|---|
| **Go** | `aws-sdk-go-v2/service/sns` + `aws-sdk-go-v2/service/sqs` |
| **Spring Boot** | `spring-cloud-aws-messaging` ou AWS SDK v2 + `@SqsListener` |
| **Quarkus** | `quarkus-amazon-sns` + `quarkus-amazon-sqs` |
| **Micronaut** | `micronaut-aws-sdk-v2` |
| **Jakarta EE** | AWS SDK v2 direto |

> **Critérios de aceite (SNS/SQS):**
> - [ ] LocalStack rodando com SNS topic e 3 SQS queues
> - [ ] Fan-out: 1 publish no SNS → mensagem em 3 filas
> - [ ] Consumer consome de SQS com idempotência
> - [ ] DLQ configurada para mensagens com falha de processamento
> - [ ] Testes de integração usando LocalStack (Testcontainers ou direto)

---

## Extensão: Schema Evolution & Schema Registry

Evolua seus schemas de eventos de forma segura com versionamento e compatibilidade.

### Schema Registry com Confluent

```yaml
# docker-compose.yml
services:
  schema-registry:
    image: confluentinc/cp-schema-registry:7.6.0
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
```

### Estratégias de Compatibilidade

| Estratégia | Quando usar | Exemplo |
|---|---|---|
| **BACKWARD** (default) | Consumer novo lê dados antigos | Adicionar campo optional |
| **FORWARD** | Producer novo, consumer antigo | Remover campo optional |
| **FULL** | Ambos | Adicionar campo optional + default value |
| **NONE** | Desenvolvimento/testes | Breaking changes permitidos |

### Formato de Eventos (Avro)

```json
{
  "type": "record",
  "name": "TransactionEvent",
  "namespace": "com.novapay.wallet.events",
  "fields": [
    {"name": "eventId", "type": "string"},
    {"name": "eventType", "type": {"type": "enum", "name": "EventType", "symbols": ["DEPOSIT", "WITHDRAWAL", "TRANSFER"]}},
    {"name": "walletId", "type": "string"},
    {"name": "amount", "type": {"type": "bytes", "logicalType": "decimal", "precision": 18, "scale": 2}},
    {"name": "currency", "type": "string"},
    {"name": "timestamp", "type": {"type": "long", "logicalType": "timestamp-millis"}},
    {"name": "metadata", "type": ["null", {"type": "map", "values": "string"}], "default": null}
  ]
}
```

> **Critérios de aceite (Schema Registry):**
> - [ ] Schema Registry rodando no Docker Compose
> - [ ] Eventos serializados em Avro (ou JSON Schema) com registro no Schema Registry
> - [ ] Compatibilidade BACKWARD configurada — adicionar um campo novo sem quebrar consumers existentes
> - [ ] Testes de compatibilidade de schema no CI

---

## Extensão: CDC — Change Data Capture com Debezium

Capture mudanças no banco de dados em tempo real e propague como eventos, sem alterar o código da aplicação.

### Arquitetura CDC

```
┌─────────────┐     WAL/Binlog     ┌──────────┐     Kafka     ┌──────────────┐
│  PostgreSQL  │ ─────────────────→ │ Debezium  │ ───────────→ │ Kafka Topics  │
│  (wallets,   │                    │ Connector │              │ dbserver.     │
│  transactions)│                   └──────────┘              │ wallet.tb_*   │
└─────────────┘                                               └──────────────┘
                                                                     │
                                                               ┌─────┴─────┐
                                                               │  Consumer  │
                                                               │  (CQRS     │
                                                               │   read     │
                                                               │   model)   │
                                                               └───────────┘
```

### Setup Debezium

```yaml
# docker-compose.yml
services:
  debezium:
    image: debezium/connect:2.6
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: debezium-wallet
      CONFIG_STORAGE_TOPIC: debezium_configs
      OFFSET_STORAGE_TOPIC: debezium_offsets
      STATUS_STORAGE_TOPIC: debezium_statuses
```

```bash
# Registrar connector para capturar mudanças nas tabelas do wallet
curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{
  "name": "wallet-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "wallet_user",
    "database.password": "wallet_pass",
    "database.dbname": "digital_wallet",
    "topic.prefix": "dbserver",
    "table.include.list": "public.wallets,public.transactions",
    "plugin.name": "pgoutput",
    "slot.name": "wallet_slot",
    "publication.name": "wallet_publication"
  }
}'
```

> **Critérios de aceite (CDC):**
> - [ ] Debezium rodando e conectado ao PostgreSQL
> - [ ] Mudanças em `wallets` e `transactions` publicadas automaticamente no Kafka
> - [ ] Consumer processa eventos CDC para mantener read model atualizado
> - [ ] Tombstone events (deletes) tratados corretamente

---

## Extensão: CQRS — Command Query Responsibility Segregation

Separe o modelo de escrita (commands) do modelo de leitura (queries) para otimizar cada um independentemente.

### Arquitetura CQRS

```
┌──────────────────────────────────────────────────────────────────┐
│                        CQRS Architecture                          │
│                                                                    │
│  ┌─── Commands ───┐              ┌─── Queries ───┐               │
│  │ POST deposit    │              │ GET statement  │               │
│  │ POST transfer   │              │ GET balance    │               │
│  │ POST withdrawal │              │ GET analytics  │               │
│  └───────┬─────────┘              └───────┬────────┘              │
│          │                                │                        │
│  ┌───────▼─────────┐              ┌───────▼────────┐              │
│  │  Write Model     │   ──CDC──→  │  Read Model     │              │
│  │  (PostgreSQL)    │   ou Event  │  (PostgreSQL/   │              │
│  │  Normalizado     │   Sourcing  │   Elasticsearch/│              │
│  │                  │             │   Redis)        │              │
│  └──────────────────┘             └─────────────────┘              │
└──────────────────────────────────────────────────────────────────┘
```

### Implementação Simplificada

```sql
-- Read model: view materializada para consultas rápidas de extrato
CREATE MATERIALIZED VIEW wallet_statement_view AS
SELECT
  t.id, t.wallet_id, t.type, t.amount, t.currency,
  t.status, t.description, t.created_at,
  w.balance AS wallet_balance_after,
  u.name AS user_name
FROM transactions t
JOIN wallets w ON t.wallet_id = w.id
JOIN users u ON w.user_id = u.id
ORDER BY t.created_at DESC;

-- Refresh via CDC event ou scheduler
REFRESH MATERIALIZED VIEW CONCURRENTLY wallet_statement_view;
```

> **Critérios de aceite (CQRS):**
> - [ ] Commands (escrita) e Queries (leitura) separados em services/handlers distintos
> - [ ] Read model atualizado via CDC (Debezium) ou projeção de eventos
> - [ ] Queries de extrato consultam read model (mais rápido que joins em write model)
> - [ ] Eventual consistency documentada — read model pode ter lag de N ms

---

## Extensão: Event Sourcing

Armazene o estado como sequência de eventos imutáveis ao invés de estado mutável.

### Event Store

```sql
CREATE TABLE event_store (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_type   VARCHAR(50) NOT NULL,      -- 'Wallet'
  aggregate_id     UUID NOT NULL,             -- wallet_id
  event_type       VARCHAR(100) NOT NULL,     -- 'BalanceCredited'
  event_data       JSONB NOT NULL,            -- payload do evento
  metadata         JSONB,                     -- traceId, userId, etc.
  version          BIGINT NOT NULL,           -- para optimistic locking
  created_at       TIMESTAMPTZ DEFAULT now(),
  UNIQUE(aggregate_id, version)               -- garante ordem
);
```

### Reconstrução de Estado (Replay)

```
Wallet abc-123 — replay:
  Event 1: WalletCreated      { currency: BRL }                     → balance: 0
  Event 2: BalanceCredited     { amount: 1000, type: DEPOSIT }      → balance: 1000
  Event 3: BalanceDebited      { amount: 250, type: TRANSFER }      → balance: 750
  Event 4: BalanceDebited      { amount: 100, type: WITHDRAWAL }    → balance: 650
  Event 5: WalletBlocked       { reason: "FRAUD_SUSPICION" }        → status: BLOCKED

  Estado atual: { balance: 650, currency: BRL, status: BLOCKED }
```

### Snapshot para Performance

```sql
CREATE TABLE wallet_snapshots (
  wallet_id    UUID PRIMARY KEY,
  balance      DECIMAL(18,2) NOT NULL,
  currency     VARCHAR(3) NOT NULL,
  status       VARCHAR(20) NOT NULL,
  version      BIGINT NOT NULL,           -- último evento aplicado
  snapshot_at  TIMESTAMPTZ DEFAULT now()
);

-- Reconstrução: carregar snapshot + replay eventos com version > snapshot.version
```

> **Critérios de aceite (Event Sourcing):**
> - [ ] Event store com tabela de eventos imutáveis
> - [ ] Saldo da wallet reconstruído a partir de replay de eventos
> - [ ] Optimistic locking via campo `version` (409 Conflict em concorrência)
> - [ ] Snapshot implementado para performance (não replay 100% dos eventos)
> - [ ] Audit trail completo — todos os eventos preservados

---

## Erros Comuns

| Erro | Stack | Como evitar |
|---|---|---|
| Publicar evento ANTES de salvar no banco | Todos | Outbox pattern — salvar na mesma tx |
| Consumer sem idempotência | Todos | Verificar eventId antes de processar |
| Consumer sem DLQ | Todos | Após N retries, enviar para DLQ |
| Serialização incompatível (producer ≠ consumer) | Todos | Definir schema compartilhado e versionado |
| Auto-commit de offset → perda de mensagens | Todos | Manual ack após processamento com sucesso |
| Partition key errada (null) → sem order garantida | Todos | Usar aggregateId como partition key |
| Consumer não fecha conexão gracefully | Todos | Shutdown hook com `consumer.Close()` / `@PreDestroy` |
| Tópico com 1 partição → sem paralelismo | Todos | ≥ 3 partições em produção |

---

## Como Isso Aparece em Entrevistas

- "O que é event-driven architecture? Quais são os trade-offs?"
- "Explique at-least-once vs at-most-once vs exactly-once delivery"
- "O que é o outbox pattern e por que é necessário?"
- "Como garantir que um consumer é idempotente?"
- "O que é uma dead letter queue?"
- "Qual a diferença entre Kafka e RabbitMQ?"
- "O que são consumer groups e como fazem balanceamento?"
- "Como funciona o SmallRye Reactive Messaging comparado ao Spring Kafka?"
- "O que é back-pressure em streaming?"
- "Como monitorar consumer lag?"

---

## Comandos de Execução

```bash
# Subir Kafka
docker-compose up -d kafka kafka-ui

# Criar tópicos
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --create \
  --topic wallet.transactions --partitions 3 --replication-factor 1

docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --create \
  --topic wallet.users --partitions 3 --replication-factor 1

docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --create \
  --topic wallet.dlq --partitions 1 --replication-factor 1

# Listar tópicos
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list

# Consumer CLI (debug)
docker exec -it kafka kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic wallet.transactions --from-beginning

# Kafka UI
open http://localhost:8090

# Testar: fazer um depósito e verificar no consumer CLI e audit_events
curl -X POST http://localhost:8080/api/v1/wallets/{walletId}/transactions/deposit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"amount": 100.00, "description": "Test deposit"}'

# Verificar audit
docker exec -it postgres psql -U wallet_user -d digital_wallet -c "SELECT * FROM audit_events ORDER BY processed_at DESC LIMIT 5;"
```
