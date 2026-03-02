# Level 04 — Stream Processing

> **Objetivo:** Implementar pipelines de processamento em tempo real com Kafka,
> Kinesis, Apache Flink e Go consumers para dados em streaming.

**Referência:** [03 — Data Processing](../../.docs/data-engineering/03-data-processing.md)

**Pré-requisito:** Level 03 concluído.

---

## Contexto

Stream processing permite reagir a eventos **em tempo real** (sub-segundo a minutos).
Diferente do batch (horas/dias), streaming lida com dados infinitos, janelas de tempo
e garantias de entrega (at-least-once, exactly-once). Usamos **Kafka** como message
broker, **Kinesis** (LocalStack) para integração AWS, e **Flink** para complex
event processing.

---

## Parte 1 — ADRs (3 obrigatórios)

### ADR-009: Streaming Platform

**Arquivo:** `docs/adrs/ADR-009-streaming-platform.md`

**Context:** Precisamos de uma plataforma para ingestão e processamento de eventos
em tempo real.

**Options:**
1. **Apache Kafka (KRaft)** — broker distribuído, ecossistema rico, auto-gerenciado
2. **AWS Kinesis Data Streams** — serverless, integração nativa AWS
3. **Apache Pulsar** — multi-tenancy, tiered storage nativo
4. **AWS MSK (Managed Kafka)** — Kafka gerenciado na AWS
5. **Redpanda** — Kafka-compatible, sem JVM, menor latência

**Decision Drivers:**
- Throughput (events/sec)
- Latência end-to-end (p99)
- Durabilidade e replicação
- Custo operacional
- Ecossistema de conectores

---

### ADR-010: Stream Processing Engine

**Arquivo:** `docs/adrs/ADR-010-stream-processing-engine.md`

**Context:** Precisamos processar, agregar e enriquecer streams de eventos.

**Options:**
1. **Apache Flink** — stateful stream processing, exactly-once, event time
2. **Kafka Streams** — library Java, sem cluster separado
3. **Apache Spark Structured Streaming** — micro-batch, integração Spark
4. **Custom consumers** — Go/Python consumers simples, sem framework
5. **AWS Kinesis Data Analytics** — Flink serverless na AWS

**Decision Drivers:**
- Windowing (tumbling, sliding, session)
- State management (RocksDB, checkpoints)
- Exactly-once semantics
- Latência (ms vs seconds)
- Operabilidade (backpressure, checkpoints, savepoints)

---

### ADR-011: Delivery Guarantees

**Arquivo:** `docs/adrs/ADR-011-delivery-guarantees.md`

**Context:** Precisamos definir garantia de entrega para diferentes casos de uso.

**Options:**
1. **At-most-once** — fire-and-forget, possível perda (métricas)
2. **At-least-once** — retry com possível duplicação (default Kafka)
3. **Exactly-once** — idempotent producers + transactional consumers
4. **Effectively-once** — at-least-once + deduplication no consumer

**Decision Drivers:**
- Criticidade dos dados (financeiro vs logs)
- Throughput vs overhead
- Complexidade de implementação
- Suporte do broker/framework

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

### DrawIO-007: Streaming Architecture

**Arquivo:** `docs/diagrams/04-streaming-architecture.drawio`

```
┌─────────┐     ┌────────────────┐     ┌─────────────────┐
│ Sources │     │  MESSAGE BUS   │     │   PROCESSORS    │
│         │     │                │     │                 │
│ App API ├────▶│   Kafka        │────▶│ Flink Jobs      │
│ CDC     │     │   Topic:       │     │ - enrichment    │
│ IoT     │     │   events       │     │ - aggregation   │
│ Logs    │     │   orders       │     │ - windowing     │
│         │     │   clickstream  │     │                 │
└─────────┘     └────────┬───────┘     └────────┬────────┘
                         │                      │
                ┌────────▼───────┐     ┌────────▼────────┐
                │   Kinesis      │     │    SINKS        │
                │   (LocalStack) │     │                 │
                │   firehose     │     │ S3 (parquet)    │
                │   → S3         │     │ PostgreSQL      │
                └────────────────┘     │ DynamoDB        │
                                       │ Elasticsearch   │
                                       └─────────────────┘
```

### DrawIO-008: Windowing Strategies

**Arquivo:** `docs/diagrams/04-windowing.drawio`

```
Tumbling Window (non-overlapping, fixed size):
|--[0-5min]--|--[5-10min]--|--[10-15min]--|

Sliding Window (overlapping, fixed size + slide interval):
|--[0-5min]--|
   |--[1-6min]--|
      |--[2-7min]--|

Session Window (gap-based, variable size):
|--[events]--| gap |--[events]---|  gap  |--[events]--|

Hopping Window (fixed size + hop interval):
|-----[0-10min]-----|
      |-----[5-15min]-----|
            |-----[10-20min]-----|
```

---

## Parte 3 — Implementação

### 3.1 — Kafka Producer/Consumer (Python)

```python
# python/streaming/kafka_producer.py
import json
import time
import uuid
from datetime import datetime
from confluent_kafka import Producer

conf = {"bootstrap.servers": "localhost:9092"}
producer = Producer(conf)

def produce_events(topic: str, n_events: int = 1000, rate: float = 0.01):
    """Produz eventos de ordem para Kafka."""
    for i in range(n_events):
        event = {
            "event_id": str(uuid.uuid4()),
            "event_type": "order_created",
            "timestamp": datetime.utcnow().isoformat(),
            "payload": {
                "order_id": i + 1,
                "customer_id": (i % 100) + 1,
                "product": f"PROD-{(i % 50) + 1:03d}",
                "amount": round(10.0 + (i * 0.73 % 990), 2),
                "currency": "BRL",
            },
        }
        producer.produce(
            topic,
            key=str(event["payload"]["customer_id"]),
            value=json.dumps(event),
            callback=delivery_report,
        )
        
        if i % 100 == 0:
            producer.flush()
        time.sleep(rate)
    
    producer.flush()
    print(f"✓ Produced {n_events} events to {topic}")

def delivery_report(err, msg):
    if err:
        print(f"✗ Delivery failed: {err}")

if __name__ == "__main__":
    produce_events("orders", n_events=10000)
```

```python
# python/streaming/kafka_consumer.py
import json
from confluent_kafka import Consumer, KafkaError

def consume_and_process(topic: str, group_id: str = "data-eng-group"):
    """Consumer que processa eventos e grava no S3."""
    conf = {
        "bootstrap.servers": "localhost:9092",
        "group.id": group_id,
        "auto.offset.reset": "earliest",
        "enable.auto.commit": False,
    }
    
    consumer = Consumer(conf)
    consumer.subscribe([topic])
    
    batch = []
    batch_size = 100
    
    try:
        while True:
            msg = consumer.poll(1.0)
            if msg is None:
                continue
            if msg.error():
                if msg.error().code() == KafkaError._PARTITION_EOF:
                    continue
                raise Exception(msg.error())
            
            event = json.loads(msg.value())
            batch.append(event)
            
            if len(batch) >= batch_size:
                flush_to_s3(batch)
                consumer.commit()
                batch = []
    except KeyboardInterrupt:
        pass
    finally:
        if batch:
            flush_to_s3(batch)
            consumer.commit()
        consumer.close()

def flush_to_s3(batch: list):
    """Micro-batch: converte batch para Parquet e grava no S3."""
    import pandas as pd
    import pyarrow.parquet as pq
    import pyarrow as pa
    import boto3
    from datetime import datetime
    from io import BytesIO
    
    df = pd.json_normalize(batch)
    table = pa.Table.from_pandas(df)
    buf = BytesIO()
    pq.write_table(table, buf, compression="snappy")
    buf.seek(0)
    
    s3 = boto3.client("s3", endpoint_url="http://localhost:4566",
                       aws_access_key_id="test", aws_secret_access_key="test")
    ts = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
    key = f"streaming/orders/{ts}.parquet"
    s3.put_object(Bucket="data-lake-raw", Key=key, Body=buf.getvalue())
    print(f"✓ Flushed {len(batch)} events to s3://data-lake-raw/{key}")
```

### 3.2 — Kinesis Producer/Consumer (Python + LocalStack)

```python
# python/streaming/kinesis_producer.py
import json
import boto3
from datetime import datetime

kinesis = boto3.client("kinesis", endpoint_url="http://localhost:4566",
                        aws_access_key_id="test", aws_secret_access_key="test",
                        region_name="us-east-1")

def put_events(stream_name: str, events: list[dict]):
    """Envia batch de eventos para Kinesis."""
    records = [
        {
            "Data": json.dumps(event).encode(),
            "PartitionKey": str(event.get("customer_id", "default")),
        }
        for event in events
    ]
    
    response = kinesis.put_records(StreamName=stream_name, Records=records)
    failed = response.get("FailedRecordCount", 0)
    print(f"✓ Put {len(records)} records, {failed} failed")
    return response
```

### 3.3 — Go: High-Performance Kafka Consumer

```go
// go/cmd/stream-consumer/main.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "os/signal"
    "syscall"
    "time"

    "github.com/segmentio/kafka-go"
)

type OrderEvent struct {
    EventID   string    `json:"event_id"`
    EventType string    `json:"event_type"`
    Timestamp time.Time `json:"timestamp"`
    Payload   struct {
        OrderID    int     `json:"order_id"`
        CustomerID int     `json:"customer_id"`
        Product    string  `json:"product"`
        Amount     float64 `json:"amount"`
    } `json:"payload"`
}

func main() {
    ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer cancel()

    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers:     []string{"localhost:9092"},
        Topic:       "orders",
        GroupID:      "go-data-eng",
        MinBytes:    1e3,  // 1KB
        MaxBytes:    10e6, // 10MB
        StartOffset: kafka.FirstOffset,
    })
    defer reader.Close()

    var processed int
    for {
        msg, err := reader.ReadMessage(ctx)
        if err != nil {
            if ctx.Err() != nil {
                break
            }
            log.Printf("Error: %v", err)
            continue
        }

        var event OrderEvent
        if err := json.Unmarshal(msg.Value, &event); err != nil {
            log.Printf("Parse error: %v", err)
            continue
        }

        processed++
        if processed%1000 == 0 {
            fmt.Printf("Processed %d events (latest: %s, amount: %.2f)\n",
                processed, event.EventType, event.Payload.Amount)
        }
    }

    fmt.Printf("\nTotal processed: %d events\n", processed)
}
```

### 3.4 — Java: Apache Flink Streaming Job

```java
// java/src/.../flink/OrderStreamJob.java
public class OrderStreamJob {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(2);
        env.enableCheckpointing(30_000); // 30s checkpoints

        // Source: Kafka
        KafkaSource<OrderEvent> source = KafkaSource.<OrderEvent>builder()
            .setBootstrapServers("localhost:9092")
            .setTopics("orders")
            .setGroupId("flink-data-eng")
            .setStartingOffsets(OffsetsInitializer.earliest())
            .setDeserializer(new OrderEventDeserializer())
            .build();

        DataStream<OrderEvent> events = env.fromSource(
            source, WatermarkStrategy
                .<OrderEvent>forBoundedOutOfOrderness(Duration.ofSeconds(5))
                .withTimestampAssigner((event, ts) -> event.getTimestamp()),
            "Kafka Source"
        );

        // Tumbling window aggregation: revenue per product per 5 minutes
        DataStream<ProductRevenue> revenue = events
            .keyBy(OrderEvent::getProduct)
            .window(TumblingEventTimeWindows.of(Time.minutes(5)))
            .aggregate(new RevenueAggregator());

        // Sliding window: moving average per customer
        DataStream<CustomerAvg> avgOrders = events
            .keyBy(OrderEvent::getCustomerId)
            .window(SlidingEventTimeWindows.of(Time.minutes(10), Time.minutes(5)))
            .process(new MovingAverageFunction());

        // Sink: write to S3 as Parquet
        revenue.sinkTo(createS3Sink("revenue"));
        avgOrders.sinkTo(createS3Sink("customer-avg"));

        env.execute("Order Stream Processing");
    }
}
```

### 3.5 — Docker Compose: Streaming Stack

```yaml
# infra/docker-compose.streaming.yml (extends base)
services:
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: data-eng
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092

  flink-jobmanager:
    image: flink:1.18
    ports:
      - "8081:8081"
    command: jobmanager
    environment:
      FLINK_PROPERTIES: |
        jobmanager.rpc.address: flink-jobmanager
        state.checkpoints.dir: file:///tmp/flink-checkpoints

  flink-taskmanager:
    image: flink:1.18
    command: taskmanager
    environment:
      FLINK_PROPERTIES: |
        jobmanager.rpc.address: flink-jobmanager
        taskmanager.numberOfTaskSlots: 4
```

---

## Critérios de Aceite

- [ ] Kafka producer gerando ≥ 10K events
- [ ] Python consumer com micro-batch → S3 (Parquet)
- [ ] Kinesis producer/consumer funcional (LocalStack)
- [ ] Go consumer processando ≥ 1K events/sec
- [ ] Flink job com tumbling + sliding windows
- [ ] Flink checkpointing configurado
- [ ] Kafka UI acessível (visualizar topics, consumer groups, lag)
- [ ] Pipeline end-to-end: producer → Kafka → Flink → S3
- [ ] ADR-009, ADR-010, ADR-011 completos
- [ ] 2 DrawIOs (streaming arch + windowing)

---

## Definição de Pronto (DoD)

- [ ] 3 ADRs (platform + engine + guarantees)
- [ ] 2 DrawIOs (streaming architecture + windowing)
- [ ] Kafka producer/consumer Python funcional
- [ ] Kinesis producer/consumer funcional
- [ ] Go high-performance consumer
- [ ] Flink streaming job com windows
- [ ] Streaming stack Docker Compose
- [ ] Commit: `feat(data-engineering-04): stream processing`
