# Level 06 — Change Data Capture (CDC)

> **Objetivo:** Implementar captura de mudanças em bancos de dados em tempo real
> usando Debezium, AWS DMS e padrão Outbox para sincronizar dados entre sistemas.

**Referência:** [04 — Data Integration](../../.docs/data-engineering/04-data-integration.md)

**Pré-requisito:** Level 05 concluído.

---

## Contexto

**Change Data Capture (CDC)** permite capturar inserções, updates e deletes em
bancos de dados e propagá-los como eventos em tempo real. É a base para
sincronização entre sistemas, denormalização para read models, alimentação de
data lakes sem batch pesado e event sourcing.

---

## Parte 1 — ADRs (3 obrigatórios)

### ADR-014: CDC Platform

**Arquivo:** `docs/adrs/ADR-014-cdc-platform.md`

**Context:** Precisamos capturar mudanças em PostgreSQL e propagar para Kafka/S3.

**Options:**
1. **Debezium** — CDC open-source baseado em Kafka Connect, log-based
2. **AWS DMS** — Database Migration Service, suporte a CDC contínuo
3. **PostgreSQL Logical Replication** — nativo, WAL-based, pub/sub
4. **Airbyte** — EL(T) com change detection mode
5. **Custom WAL reader** — leitura direta do WAL com pg_logical

**Decision Drivers:**
- Latência de captura (ms vs min)
- Suporte a schema evolution
- Impacto no banco source (CPU/IO overhead)
- Formatos de output (Avro, JSON, Protobuf)
- Exactly-once delivery

---

### ADR-015: CDC Event Format

**Arquivo:** `docs/adrs/ADR-015-cdc-event-format.md`

**Context:** Precisamos padronizar o formato dos eventos CDC.

**Options:**
1. **Debezium envelope** — before/after + metadata (op, source, ts_ms)
2. **CloudEvents** — specification standard para cloud events
3. **Custom envelope** — formato proprietário simplificado
4. **Avro + Schema Registry** — schema evolution com compatibility checks

**Decision Drivers:**
- Schema evolution (backward/forward compatible)
- Payload size (overhead do envelope)
- Interoperabilidade com consumers
- Dead letter handling

---

### ADR-016: Outbox Pattern Implementation

**Arquivo:** `docs/adrs/ADR-016-outbox-pattern.md`

**Context:** Operações de negócio precisam publicar eventos de forma transacional
(atomicidade entre write + event publish).

**Options:**
1. **Transactional Outbox + Debezium** — tabela outbox capturada via CDC
2. **Transactional Outbox + Polling** — job poll da tabela outbox
3. **Event Sourcing** — log de eventos como source of truth
4. **Dual Write** — write no DB + publish no Kafka (risco de inconsistência)

**Decision Drivers:**
- Consistência transacional (zero message loss)
- Latência de propagação
- Complexidade operacional
- Ordering guarantees

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

### DrawIO-011: CDC Architecture

**Arquivo:** `docs/diagrams/06-cdc-architecture.drawio`

```
┌──────────────┐    ┌──────────────────┐    ┌──────────────┐
│  PostgreSQL  │    │    Debezium      │    │    Kafka      │
│              │    │  (Kafka Connect) │    │              │
│  orders      │───▶│                  │───▶│  db.orders   │
│  customers   │WAL │  Captures:       │    │  db.customers│
│  products    │    │  INSERT          │    │  db.products │
│              │    │  UPDATE          │    │              │
│              │    │  DELETE          │    │              │
└──────────────┘    └──────────────────┘    └──────┬───────┘
                                                   │
                              ┌─────────────────────┤
                              ▼                     ▼
                    ┌──────────────┐      ┌──────────────┐
                    │   S3 Sink    │      │   Consumer   │
                    │  Connector   │      │  (stream     │
                    │              │      │   processing)│
                    │  Parquet in  │      │              │
                    │  data lake   │      │  Read models │
                    └──────────────┘      │  DynamoDB    │
                                          └──────────────┘
```

### DrawIO-012: Outbox Pattern Flow

**Arquivo:** `docs/diagrams/06-outbox-pattern.drawio`

```
┌───────────────────────────────────────────────┐
│            Application Service                │
│                                               │
│  BEGIN TRANSACTION                            │
│  ├── INSERT INTO orders (...)                 │
│  └── INSERT INTO outbox (                     │
│        aggregate_type, aggregate_id,          │
│        event_type, payload                    │
│      )                                        │
│  COMMIT                                       │
└───────────────────────┬───────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────┐
│              Outbox Table                     │
│  id | aggregate_type | event_type | payload   │
│  1  | Order          | Created    | {...}     │
│  2  | Order          | Updated    | {...}     │
└───────────────────────┬───────────────────────┘
                        │ Debezium CDC
                        ▼
┌───────────────────────────────────────────────┐
│          Kafka Topic: outbox.events           │
│                                               │
│  Routed by aggregate_type to:                 │
│  ├── orders.events                            │
│  ├── customers.events                         │
│  └── payments.events                          │
└───────────────────────────────────────────────┘
```

---

## Parte 3 — Implementação

### 3.1 — Debezium Setup (Docker Compose)

```yaml
# infra/docker-compose.cdc.yml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka-connect:
    image: debezium/connect:2.5
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: cdc-connect
      CONFIG_STORAGE_TOPIC: _connect-configs
      OFFSET_STORAGE_TOPIC: _connect-offsets
      STATUS_STORAGE_TOPIC: _connect-status
    depends_on:
      - kafka
      - postgres

  schema-registry:
    image: confluentinc/cp-schema-registry:7.6.0
    ports:
      - "8085:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
```

### 3.2 — Debezium Connector Configuration

```python
# python/cdc/register_connector.py
import requests
import json

CONNECT_URL = "http://localhost:8083"

def register_postgres_connector():
    """Registra connector Debezium para PostgreSQL."""
    config = {
        "name": "postgres-cdc",
        "config": {
            "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
            "database.hostname": "postgres",
            "database.port": "5432",
            "database.user": "dataeng",
            "database.password": "dataeng",
            "database.dbname": "datawarehouse",
            "database.server.name": "dbserver1",
            "topic.prefix": "cdc",
            "schema.include.list": "public",
            "table.include.list": "public.orders,public.customers,public.products",
            
            # Outbox pattern
            "transforms": "outbox",
            "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
            "transforms.outbox.table.expand.json.payload": "true",
            "transforms.outbox.route.topic.replacement": "${routedByValue}.events",
            
            # Format
            "key.converter": "org.apache.kafka.connect.json.JsonConverter",
            "value.converter": "org.apache.kafka.connect.json.JsonConverter",
            "key.converter.schemas.enable": "false",
            "value.converter.schemas.enable": "true",
            
            # Snapshot
            "snapshot.mode": "initial",
            
            # Heartbeat
            "heartbeat.interval.ms": "10000",
        },
    }
    
    response = requests.post(f"{CONNECT_URL}/connectors", json=config)
    print(f"Status: {response.status_code}")
    print(json.dumps(response.json(), indent=2))

def check_connector_status():
    response = requests.get(f"{CONNECT_URL}/connectors/postgres-cdc/status")
    status = response.json()
    print(f"Connector: {status['connector']['state']}")
    for task in status.get("tasks", []):
        print(f"  Task {task['id']}: {task['state']}")
```

### 3.3 — Outbox Table + Application (Python)

```python
# python/cdc/outbox.py
import json
import uuid
from datetime import datetime
from contextlib import contextmanager
import psycopg2

DSN = "dbname=datawarehouse user=dataeng password=dataeng host=localhost port=5432"

def setup_outbox_table():
    """Cria tabela outbox para Transactional Outbox Pattern."""
    with get_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("""
                CREATE TABLE IF NOT EXISTS outbox (
                    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
                    aggregate_type  VARCHAR(100) NOT NULL,
                    aggregate_id    VARCHAR(100) NOT NULL,
                    event_type      VARCHAR(100) NOT NULL,
                    payload         JSONB NOT NULL,
                    created_at      TIMESTAMP DEFAULT NOW()
                );
            """)
        conn.commit()

@contextmanager
def get_connection():
    conn = psycopg2.connect(DSN)
    try:
        yield conn
    finally:
        conn.close()

def create_order_with_event(order_data: dict):
    """Cria order + evento na mesma transação (Outbox Pattern)."""
    with get_connection() as conn:
        with conn.cursor() as cur:
            # 1. Insert order
            cur.execute("""
                INSERT INTO orders (customer_id, product_id, quantity, amount, status)
                VALUES (%(customer_id)s, %(product_id)s, %(quantity)s, %(amount)s, 'pending')
                RETURNING id
            """, order_data)
            order_id = cur.fetchone()[0]
            
            # 2. Insert outbox event (same transaction!)
            event = {
                "order_id": order_id,
                "customer_id": order_data["customer_id"],
                "amount": float(order_data["amount"]),
                "status": "pending",
                "timestamp": datetime.utcnow().isoformat(),
            }
            cur.execute("""
                INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
                VALUES ('Order', %s, 'OrderCreated', %s)
            """, (str(order_id), json.dumps(event)))
        
        conn.commit()
        print(f"✓ Order {order_id} created with outbox event")
        return order_id
```

### 3.4 — Go: CDC Event Consumer

```go
// go/cmd/cdc-consumer/main.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"

    "github.com/segmentio/kafka-go"
)

type DebeziumEnvelope struct {
    Before  json.RawMessage `json:"before"`
    After   json.RawMessage `json:"after"`
    Source  struct {
        Table    string `json:"table"`
        TsMs     int64  `json:"ts_ms"`
        Snapshot string `json:"snapshot"`
    } `json:"source"`
    Op string `json:"op"` // c=create, u=update, d=delete, r=read(snapshot)
}

func main() {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers: []string{"localhost:9092"},
        Topic:   "cdc.public.orders",
        GroupID: "go-cdc-consumer",
    })
    defer reader.Close()

    ctx := context.Background()
    for {
        msg, err := reader.ReadMessage(ctx)
        if err != nil {
            log.Printf("Error: %v", err)
            continue
        }

        var envelope DebeziumEnvelope
        if err := json.Unmarshal(msg.Value, &envelope); err != nil {
            log.Printf("Parse error: %v", err)
            continue
        }

        switch envelope.Op {
        case "c":
            fmt.Printf("INSERT on %s: %s\n", envelope.Source.Table, string(envelope.After))
            // Materialize to read model (DynamoDB, etc.)
        case "u":
            fmt.Printf("UPDATE on %s: %s → %s\n", envelope.Source.Table,
                string(envelope.Before), string(envelope.After))
        case "d":
            fmt.Printf("DELETE on %s: %s\n", envelope.Source.Table, string(envelope.Before))
        }
    }
}
```

### 3.5 — Java: CDC Event Processor (Spring)

```java
// java/src/.../cdc/CdcEventProcessor.java
@Component
public class CdcEventProcessor {

    private final DynamoDbClient dynamoDb;

    @KafkaListener(topics = "cdc.public.orders", groupId = "java-cdc")
    public void processOrderChange(ConsumerRecord<String, String> record) {
        var envelope = objectMapper.readValue(record.value(), DebeziumEnvelope.class);
        
        switch (envelope.getOp()) {
            case "c" -> handleCreate(envelope.getAfter());
            case "u" -> handleUpdate(envelope.getBefore(), envelope.getAfter());
            case "d" -> handleDelete(envelope.getBefore());
        }
    }

    private void handleCreate(OrderRecord after) {
        // Materialize to DynamoDB read model
        dynamoDb.putItem(PutItemRequest.builder()
            .tableName("order-read-model")
            .item(Map.of(
                "pk", AttributeValue.fromS("ORDER#" + after.getId()),
                "sk", AttributeValue.fromS("DETAIL"),
                "data", AttributeValue.fromS(toJson(after))
            ))
            .build());
        
        log.info("Created read model for order {}", after.getId());
    }
}
```

---

## Critérios de Aceite

- [ ] Debezium connector registrado e capturando mudanças do PostgreSQL
- [ ] Topics Kafka criados: `cdc.public.orders`, `cdc.public.customers`
- [ ] INSERT/UPDATE/DELETE refletidos como eventos no Kafka
- [ ] Outbox table criada com transactional writes
- [ ] Outbox events roteados por aggregate_type
- [ ] Go consumer processando Debezium envelope (before/after/op)
- [ ] Java consumer materializando read model no DynamoDB
- [ ] Schema Registry com schemas Avro/JSON
- [ ] Latência < 5s entre write no PostgreSQL e evento no Kafka
- [ ] ADR-014, ADR-015, ADR-016 completos
- [ ] 2 DrawIOs (CDC architecture + outbox pattern)

---

## Definição de Pronto (DoD)

- [ ] 3 ADRs (CDC platform + event format + outbox pattern)
- [ ] 2 DrawIOs (CDC architecture + outbox pattern)
- [ ] Debezium CDC funcional (PostgreSQL → Kafka)
- [ ] Outbox pattern implementado
- [ ] Go CDC consumer
- [ ] Java CDC consumer com read model
- [ ] Docker Compose com Debezium stack
- [ ] Commit: `feat(data-engineering-06): change data capture`
