# Level 12 — Capstone: Enterprise DataFlow Platform

> **Objetivo:** Construir uma plataforma de dados empresarial completa que
> integre **todos** os conceitos dos Levels 00–11: ingestão batch e streaming,
> data lake Iceberg, warehouse PostgreSQL, dbt transformations, CDC, data quality,
> governance, observabilidade e serving layer.

**Referência:** Todos os `.docs/data-engineering/*.md`

**Pré-requisito:** Levels 00–11 concluídos.

---

## Contexto

O **Enterprise DataFlow Platform** é um projeto end-to-end que simula uma
plataforma de dados real de uma empresa de e-commerce. O sistema processa
eventos de pedidos, clientes e produtos em tempo real (streaming) e em batch,
aplica transformações com dbt (Medallion Architecture), garante qualidade com
Great Expectations, implementa governance (catalog + lineage + PII masking) e
serve dados via API REST e queries analíticas.

O capstone valida a capacidade de **integrar** componentes independentes em um
sistema coeso, resiliente e observável.

---

## Parte 1 — ADRs (6 obrigatórios)

### ADR-029: Platform Architecture

**Arquivo:** `docs/adrs/ADR-029-platform-architecture.md`

**Context:** Precisamos definir a arquitetura geral da plataforma, combinando
batch e streaming em um sistema unificado.

**Options:**
1. **Lambda Architecture** — batch layer + speed layer + serving layer
2. **Kappa Architecture** — streaming-only, reprocessamento via replay
3. **Lakehouse-first** — Iceberg como camada unificada, batch e streaming
4. **Hybrid (Lakehouse + Streaming)** — Iceberg para batch, Kafka para real-time

**Decision Drivers:**
- Complexidade operacional
- Latência (real-time vs near-real-time vs batch)
- Reprocessamento e idempotência
- Custo de infraestrutura
- Maturidade do time

---

### ADR-030: Data Ingestion Strategy

**Arquivo:** `docs/adrs/ADR-030-data-ingestion.md`

**Context:** A plataforma recebe dados de múltiplas fontes com diferentes
padrões de ingestão.

**Options:**
1. **Kafka + CDC (Debezium)** — eventos de banco + streaming
2. **API polling + batch files** — periodic extraction
3. **Change Data Capture only** — Debezium para tudo
4. **Event-driven only** — microservices publicam eventos

**Decision Drivers:**
- Latência de ingestão (< 1 min para CDC, < 1h para batch)
- Garantia de exactly-once/at-least-once
- Schema evolution suportada
- Overhead operacional

---

### ADR-031: Storage Architecture

**Arquivo:** `docs/adrs/ADR-031-storage-architecture.md`

**Context:** Precisamos definir como os dados são armazenados em cada camada
da plataforma.

**Options:**
1. **S3 Iceberg (Bronze/Silver) + PostgreSQL (Gold)** — lakehouse + RDBMS
2. **S3 Iceberg everywhere** — Iceberg em todas as camadas
3. **PostgreSQL only** — warehouse tradicional
4. **S3 Parquet (Bronze) + Iceberg (Silver/Gold)** — progressão de formatos

**Decision Drivers:**
- Custo de storage
- Performance de query
- ACID guarantees por camada
- Compatibilidade com dbt/Trino/DuckDB

---

### ADR-032: Transformation Pipeline

**Arquivo:** `docs/adrs/ADR-032-transformation-pipeline.md`

**Context:** Precisamos de um pipeline de transformação que cubra batch
(dbt) e streaming (Flink/Kafka), mantendo consistência entre as camadas.

**Options:**
1. **dbt (batch) + Flink (streaming)** — melhor de cada mundo
2. **dbt only** — simplificação, scheduled batch
3. **Spark (batch + streaming)** — unificado em Spark
4. **Custom Python** — flexível mas difícil manutenção

**Decision Drivers:**
- Unificação de lógica batch/streaming
- Testabilidade
- Linhagem e documentação
- Curva de aprendizado

---

### ADR-033: Quality & Governance Integration

**Arquivo:** `docs/adrs/ADR-033-quality-governance.md`

**Context:** Quality gates e governance devem ser integrados ao pipeline,
não como processos separados.

**Options:**
1. **Inline quality (Great Expectations + circuit breaker)** — validação dentro do pipeline
2. **Post-processing quality** — validação após cada batch
3. **dbt tests only** — testes nativos do dbt
4. **External quality service** — serviço dedicado

**Decision Drivers:**
- Automação (zero intervenção humana em fluxo normal)
- Blocking vs non-blocking failures
- Métricas de qualidade ao longo do tempo
- PII detection/masking integrado

---

### ADR-034: Observability Strategy

**Arquivo:** `docs/adrs/ADR-034-observability-strategy.md`

**Context:** A plataforma precisa de observabilidade end-to-end: métricas
de pipeline, data freshness, alertas e health checks.

**Options:**
1. **Custom metrics (DynamoDB + CloudWatch)** — build interno
2. **OpenTelemetry + Prometheus + Grafana** — padrão CNCF
3. **dbt Cloud + Great Expectations Cloud** — SaaS
4. **Airflow metrics + pipeline metrics** — métricas do orquestrador

**Decision Drivers:**
- Visibilidade end-to-end (ingestão → serving)
- Alertas proativos (freshness, quality degradation)
- Custo
- Integração com stack existente

---

## Parte 2 — Diagramas DrawIO (4 obrigatórios)

### DrawIO-023: Enterprise DataFlow — Full Architecture

**Arquivo:** `docs/diagrams/12-dataflow-full-architecture.drawio`

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    ENTERPRISE DATAFLOW PLATFORM                         │
│                                                                         │
│  ┌────────────────────────── SOURCES ──────────────────────────┐        │
│  │ PostgreSQL (OLTP)  │  REST APIs  │  CSV/JSON Files  │ IoT  │        │
│  └─────┬──────────────┴──────┬──────┴────────┬─────────┴──────┘        │
│        │                     │               │                          │
│        ▼                     ▼               ▼                          │
│  ┌──────────┐         ┌──────────┐    ┌──────────┐                     │
│  │ Debezium │         │  Kafka   │    │   S3     │                     │
│  │  (CDC)   │────────▶│ Cluster  │    │ Landing  │                     │
│  └──────────┘         └────┬─────┘    └────┬─────┘                     │
│                            │               │                            │
│  ┌─────────────────────────┼───────────────┼────────────────┐          │
│  │                    BRONZE (Raw)                           │          │
│  │  S3 + Iceberg: raw events, CDC logs, files               │          │
│  └─────────────────────────┬────────────────────────────────┘          │
│                            │                                            │
│  ┌─────────────────────────┼────────────────────────────────┐          │
│  │              PROCESSING & QUALITY                         │          │
│  │                                                           │          │
│  │  ┌─────────┐  ┌───────────────┐  ┌──────────────────┐   │          │
│  │  │ Airflow │  │     dbt       │  │ Great Expectations│   │          │
│  │  │  (DAG)  │─▶│  (transform)  │─▶│  (quality gate)  │   │          │
│  │  └─────────┘  └───────────────┘  └──────────────────┘   │          │
│  │                                                           │          │
│  │  ┌─────────────────────┐  ┌────────────────────────┐     │          │
│  │  │    Flink (stream)   │  │  Schema Registry       │     │          │
│  │  │  windows, aggregate │  │  (Avro evolution)      │     │          │
│  │  └─────────────────────┘  └────────────────────────┘     │          │
│  └─────────────────────────┬────────────────────────────────┘          │
│                            │                                            │
│  ┌─────────────────────────┼────────────────────────────────┐          │
│  │                    SILVER (Cleaned)                        │          │
│  │  Iceberg: deduplicated, typed, validated, enriched        │          │
│  └─────────────────────────┬────────────────────────────────┘          │
│                            │                                            │
│  ┌─────────────────────────┼────────────────────────────────┐          │
│  │                    GOLD (Aggregated)                       │          │
│  │  PostgreSQL: facts, dimensions, KPIs, materialized views  │          │
│  └─────────────────────────┬────────────────────────────────┘          │
│                            │                                            │
│  ┌─────────────────────────┼────────────────────────────────┐          │
│  │                   SERVING LAYER                            │          │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐  │          │
│  │  │  REST    │  │  Trino   │  │  DuckDB  │  │ dbt docs│  │          │
│  │  │  API     │  │  (SQL)   │  │ (ad-hoc) │  │ (docs)  │  │          │
│  │  └──────────┘  └──────────┘  └──────────┘  └─────────┘  │          │
│  └──────────────────────────────────────────────────────────┘          │
│                                                                         │
│  ┌─────────────────── GOVERNANCE & OBSERVABILITY ──────────────┐       │
│  │  Glue Catalog │ OpenLineage │ PII Masking │ Pipeline Metrics│       │
│  └────────────────────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────────────────────┘
```

### DrawIO-024: Data Flow — Batch Pipeline

**Arquivo:** `docs/diagrams/12-batch-pipeline-flow.drawio`

```
┌───────────────────────────────────────────────────────────────┐
│                    BATCH PIPELINE (Airflow DAG)               │
│                                                               │
│  [Sensor]         [Extract]        [Validate]                │
│  wait_for_   ───▶ extract_    ───▶ validate_                │
│  source_ready     to_bronze        source_freshness          │
│                                         │                     │
│                                         ▼                     │
│  [Quality Gate]   [Transform]      [Stage]                   │
│  circuit_    ◀─── run_great_  ◀─── dbt_run                  │
│  breaker_check    expectations     (stg→int→marts)           │
│       │                                                       │
│  ┌────┴────┐                                                 │
│  │ PASS?   │                                                 │
│  └────┬────┘                                                 │
│  Yes  │  No → alert + pause                                  │
│       ▼                                                       │
│  [Publish]        [Catalog]        [Metrics]                 │
│  publish_    ───▶ update_     ───▶ publish_                  │
│  to_gold          catalog          pipeline_metrics           │
│                                                               │
│  Schedule: daily @ 06:00 UTC                                 │
│  SLA: complete before 08:00 UTC                              │
│  Retries: 3 (exponential backoff)                            │
└───────────────────────────────────────────────────────────────┘
```

### DrawIO-025: Data Flow — Streaming Pipeline

**Arquivo:** `docs/diagrams/12-streaming-pipeline-flow.drawio`

```
┌───────────────────────────────────────────────────────────────┐
│                  STREAMING PIPELINE                           │
│                                                               │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐        │
│  │PostgreSQL│───▶│   Debezium   │───▶│    Kafka     │        │
│  │  (OLTP)  │    │ (CDC Source) │    │  orders.cdc  │        │
│  └──────────┘    └──────────────┘    └──────┬───────┘        │
│                                             │                 │
│                  ┌──────────────────────────┤                 │
│                  ▼                          ▼                 │
│  ┌───────────────────────┐  ┌────────────────────────┐       │
│  │  Python Consumer      │  │  Flink Job             │       │
│  │  ├── deserialize      │  │  ├── tumbling_window   │       │
│  │  ├── validate schema  │  │  │   (1 min)           │       │
│  │  ├── mask PII         │  │  ├── aggregate         │       │
│  │  └── write S3 Bronze  │  │  │   (count, sum, avg) │       │
│  └───────────────────────┘  │  └── write DynamoDB    │       │
│                              └────────────────────────┘       │
│                                                               │
│  ┌────────────────────────────────────────────────────┐      │
│  │  REAL-TIME DASHBOARDS                              │      │
│  │  ├── Orders/minute                                 │      │
│  │  ├── Revenue last 5 min                           │      │
│  │  └── Active customers                             │      │
│  └────────────────────────────────────────────────────┘      │
└───────────────────────────────────────────────────────────────┘
```

### DrawIO-026: Infrastructure Components

**Arquivo:** `docs/diagrams/12-infrastructure-components.drawio`

```
┌───────────────────────────────────────────────────────────────┐
│                   DOCKER COMPOSE STACK                        │
│                                                               │
│  ┌─────────────────── Core Infrastructure ──────────────┐    │
│  │ LocalStack (S3, Kinesis, DynamoDB, SQS, SNS,        │    │
│  │            Lambda, Step Functions, Glue, IAM)        │    │
│  │ PostgreSQL 16 (OLTP + Warehouse)                     │    │
│  │ Kafka (KRaft) + Schema Registry + Kafka UI           │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌─────────────────── Data Processing ──────────────────┐    │
│  │ Debezium 2.5 (Kafka Connect + PostgreSQL connector)  │    │
│  │ Flink 1.18 (JobManager + TaskManager)                │    │
│  │ Airflow 2.8 (Webserver + Scheduler + PostgreSQL)     │    │
│  │ Trino 435 (Coordinator + Hive/Iceberg/PostgreSQL)    │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌─────────────────── Applications ─────────────────────┐    │
│  │ Python Pipeline Workers (extract, transform, quality)│    │
│  │ Go Analytics API (:8090)                             │    │
│  │ Java CDC Processor + Governance Service              │    │
│  │ dbt Core (scheduled via Airflow)                     │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  Ports: LocalStack=4566 │ PostgreSQL=5432 │ Kafka=9092      │
│         Airflow=8080    │ Kafka UI=8081   │ Trino=8082      │
│         Flink=8083      │ Analytics=8090  │ SchemaReg=8085  │
└───────────────────────────────────────────────────────────────┘
```

---

## Parte 3 — Implementação

### 3.1 — Docker Compose: Full Stack

```yaml
# docker-compose.yml
version: "3.9"

services:
  # === Core Infrastructure ===
  localstack:
    image: localstack/localstack:3
    ports: ["4566:4566"]
    environment:
      SERVICES: s3,kinesis,dynamodb,sqs,sns,lambda,stepfunctions,glue,iam
      DEFAULT_REGION: us-east-1
    volumes:
      - "./scripts/init-aws.sh:/etc/localstack/init/ready.d/init-aws.sh"

  postgres:
    image: postgres:16-alpine
    ports: ["5432:5432"]
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin
      POSTGRES_DB: warehouse
    volumes:
      - "./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql"

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    ports: ["9092:9092"]
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:29093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:29093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      CLUSTER_ID: "capstone-dataflow-001"
      KAFKA_LOG_DIRS: /var/lib/kafka/data

  schema-registry:
    image: confluentinc/cp-schema-registry:7.6.0
    ports: ["8085:8081"]
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
    depends_on: [kafka]

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports: ["8081:8080"]
    environment:
      KAFKA_CLUSTERS_0_NAME: dataflow
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
    depends_on: [kafka, schema-registry]

  # === Data Processing ===
  debezium:
    image: debezium/connect:2.5
    ports: ["8083:8083"]
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: debezium-dataflow
      CONFIG_STORAGE_TOPIC: _debezium_configs
      OFFSET_STORAGE_TOPIC: _debezium_offsets
      STATUS_STORAGE_TOPIC: _debezium_status
    depends_on: [kafka, postgres]

  flink-jobmanager:
    image: flink:1.18-java17
    ports: ["8084:8081"]
    command: jobmanager
    environment:
      FLINK_PROPERTIES: |
        jobmanager.rpc.address: flink-jobmanager

  flink-taskmanager:
    image: flink:1.18-java17
    command: taskmanager
    environment:
      FLINK_PROPERTIES: |
        jobmanager.rpc.address: flink-jobmanager
        taskmanager.numberOfTaskSlots: 4
    depends_on: [flink-jobmanager]

  airflow:
    image: apache/airflow:2.8.1-python3.12
    ports: ["8080:8080"]
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://admin:admin@postgres:5432/airflow
      AIRFLOW__CORE__LOAD_EXAMPLES: "false"
      AIRFLOW__WEBSERVER__EXPOSE_CONFIG: "true"
    volumes:
      - "./dags:/opt/airflow/dags"
      - "./dbt:/opt/airflow/dbt"
    depends_on: [postgres]
    command: >
      bash -c "airflow db init &&
               airflow users create --role Admin --username admin
                 --password admin --email admin@local --firstname Admin
                 --lastname User &&
               airflow webserver & airflow scheduler"

  trino:
    image: trinodb/trino:435
    ports: ["8082:8080"]
    volumes:
      - "./config/trino/catalog:/etc/trino/catalog"
    depends_on: [postgres, localstack]

volumes:
  kafka-data:
  postgres-data:
```

### 3.2 — Init Scripts

```bash
#!/bin/bash
# scripts/init-aws.sh
set -e

echo "=== Initializing DataFlow Platform AWS Resources ==="

# S3 Buckets (Medallion)
awslocal s3 mb s3://dataflow-bronze
awslocal s3 mb s3://dataflow-silver
awslocal s3 mb s3://dataflow-gold
awslocal s3 mb s3://dataflow-landing

# Kinesis Streams
awslocal kinesis create-stream --stream-name orders-stream --shard-count 2
awslocal kinesis create-stream --stream-name events-stream --shard-count 1

# DynamoDB Tables
awslocal dynamodb create-table \
  --table-name pipeline_metrics \
  --attribute-definitions \
    AttributeName=pipeline_name,AttributeType=S \
    AttributeName=timestamp,AttributeType=S \
  --key-schema \
    AttributeName=pipeline_name,KeyType=HASH \
    AttributeName=timestamp,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

awslocal dynamodb create-table \
  --table-name data_catalog \
  --attribute-definitions \
    AttributeName=dataset_name,AttributeType=S \
    AttributeName=version,AttributeType=S \
  --key-schema \
    AttributeName=dataset_name,KeyType=HASH \
    AttributeName=version,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

awslocal dynamodb create-table \
  --table-name data_lineage \
  --attribute-definitions \
    AttributeName=dataset,AttributeType=S \
    AttributeName=timestamp,AttributeType=S \
  --key-schema \
    AttributeName=dataset,KeyType=HASH \
    AttributeName=timestamp,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

awslocal dynamodb create-table \
  --table-name audit_log \
  --attribute-definitions \
    AttributeName=entity_type,AttributeType=S \
    AttributeName=timestamp,AttributeType=S \
  --key-schema \
    AttributeName=entity_type,KeyType=HASH \
    AttributeName=timestamp,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

awslocal dynamodb create-table \
  --table-name realtime_aggregates \
  --attribute-definitions \
    AttributeName=window_key,AttributeType=S \
    AttributeName=window_start,AttributeType=S \
  --key-schema \
    AttributeName=window_key,KeyType=HASH \
    AttributeName=window_start,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

# SQS Queues
awslocal sqs create-queue --queue-name quality-alerts
awslocal sqs create-queue --queue-name pipeline-events
awslocal sqs create-queue --queue-name governance-events

# SNS Topics
awslocal sns create-topic --name pipeline-notifications
awslocal sns create-topic --name quality-alerts

# Glue Database
awslocal glue create-database \
  --database-input '{"Name":"dataflow","Description":"Enterprise DataFlow catalog"}'

echo "=== DataFlow Platform AWS Resources Initialized ==="
```

```sql
-- scripts/init-db.sql

-- Airflow database
CREATE DATABASE airflow;

-- OLTP schema (source system)
CREATE SCHEMA IF NOT EXISTS oltp;

CREATE TABLE oltp.customers (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    email       VARCHAR(200) UNIQUE NOT NULL,
    cpf         VARCHAR(14),
    phone       VARCHAR(20),
    region      VARCHAR(50),
    created_at  TIMESTAMP DEFAULT NOW(),
    updated_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE oltp.products (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    category    VARCHAR(100),
    price       DECIMAL(10,2) NOT NULL,
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE oltp.orders (
    id          SERIAL PRIMARY KEY,
    customer_id INT REFERENCES oltp.customers(id),
    product_id  INT REFERENCES oltp.products(id),
    amount      DECIMAL(12,2) NOT NULL,
    status      VARCHAR(20) DEFAULT 'pending',
    created_at  TIMESTAMP DEFAULT NOW(),
    updated_at  TIMESTAMP DEFAULT NOW()
);

-- Outbox table for CDC
CREATE TABLE oltp.outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(100) NOT NULL,
    aggregate_id    VARCHAR(100) NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    payload         JSONB NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Warehouse schemas (Medallion)
CREATE SCHEMA IF NOT EXISTS bronze;
CREATE SCHEMA IF NOT EXISTS silver;
CREATE SCHEMA IF NOT EXISTS gold;

-- Gold: materialized views / fact tables
CREATE TABLE gold.fct_revenue (
    order_date       DATE,
    category         VARCHAR(100),
    region           VARCHAR(50),
    num_orders       INT,
    unique_customers INT,
    gross_revenue    DECIMAL(14,2),
    net_revenue      DECIMAL(14,2),
    refunds          DECIMAL(14,2),
    adjusted_revenue DECIMAL(14,2),
    refund_rate      DECIMAL(5,2)
);

CREATE TABLE gold.dim_customers (
    customer_id      INT PRIMARY KEY,
    customer_name    VARCHAR(200),
    region           VARCHAR(50),
    total_orders     INT,
    total_spent      DECIMAL(14,2),
    avg_order_value  DECIMAL(10,2),
    first_order_at   TIMESTAMP,
    last_order_at    TIMESTAMP,
    customer_segment VARCHAR(20),
    days_since_last_order INT
);

-- Seed data
INSERT INTO oltp.customers (name, email, cpf, phone, region) VALUES
    ('Alice Santos', 'alice@example.com', '123.456.789-00', '11999990001', 'SP'),
    ('Bruno Costa', 'bruno@example.com', '234.567.890-11', '21999990002', 'RJ'),
    ('Carla Lima', 'carla@example.com', '345.678.901-22', '31999990003', 'MG');

INSERT INTO oltp.products (name, category, price) VALUES
    ('Notebook Pro', 'Electronics', 4500.00),
    ('Wireless Mouse', 'Electronics', 120.00),
    ('Desk Chair', 'Furniture', 890.00),
    ('Mechanical Keyboard', 'Electronics', 350.00);

INSERT INTO oltp.orders (customer_id, product_id, amount, status) VALUES
    (1, 1, 4500.00, 'completed'),
    (1, 2, 120.00, 'completed'),
    (2, 3, 890.00, 'completed'),
    (2, 4, 350.00, 'pending'),
    (3, 1, 4500.00, 'completed'),
    (3, 2, 120.00, 'refunded');
```

### 3.3 — Airflow Enterprise DAG

```python
# dags/dataflow_enterprise_dag.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.operators.bash import BashOperator
from airflow.sensors.external_task import ExternalTaskSensor
from airflow.utils.trigger_rule import TriggerRule

default_args = {
    "owner": "data-engineering",
    "depends_on_past": False,
    "email_on_failure": False,
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
    "execution_timeout": timedelta(hours=2),
}


def _extract_to_bronze(**context):
    """Extract from sources to Bronze layer (S3 + Iceberg)."""
    import boto3
    import json
    import psycopg2
    from datetime import timezone
    
    conn = psycopg2.connect(
        host="postgres", port=5432,
        user="admin", password="admin", dbname="warehouse"
    )
    s3 = boto3.client("s3", endpoint_url="http://localstack:4566")
    
    execution_date = context["ds"]
    
    for table in ["orders", "customers", "products"]:
        cur = conn.cursor()
        cur.execute(f"""
            SELECT row_to_json(t)
            FROM oltp.{table} t
            WHERE DATE(updated_at) = '{execution_date}'
        """)
        rows = [row[0] for row in cur.fetchall()]
        cur.close()
        
        if rows:
            body = "\n".join(json.dumps(r, default=str) for r in rows)
            key = f"bronze/{table}/{execution_date}/{table}.jsonl"
            s3.put_object(Bucket="dataflow-bronze", Key=key, Body=body)
            print(f"✓ Extracted {len(rows)} rows to s3://dataflow-bronze/{key}")
    
    conn.close()
    return len(rows)


def _validate_source_freshness(**context):
    """Check source freshness before processing."""
    import psycopg2
    from datetime import timezone
    
    conn = psycopg2.connect(
        host="postgres", port=5432,
        user="admin", password="admin", dbname="warehouse"
    )
    cur = conn.cursor()
    cur.execute("""
        SELECT MAX(updated_at) FROM oltp.orders
    """)
    last_update = cur.fetchone()[0]
    cur.close()
    conn.close()
    
    now = datetime.now(timezone.utc)
    lag = (now - last_update.replace(tzinfo=timezone.utc)).total_seconds() / 3600
    
    if lag > 24:
        raise ValueError(f"Source freshness SLA violated: {lag:.1f}h lag")
    
    print(f"✓ Source freshness OK: {lag:.1f}h lag")
    return lag


def _run_quality_gate(**context):
    """Run Great Expectations quality checks with circuit breaker."""
    import json
    
    # Simulate quality check results
    results = {
        "success": True,
        "statistics": {
            "evaluated_expectations": 15,
            "successful_expectations": 15,
            "unsuccessful_expectations": 0,
            "success_percent": 100.0
        }
    }
    
    success_rate = results["statistics"]["success_percent"]
    
    if success_rate < 95:
        context["ti"].xcom_push(key="quality_status", value="FAIL")
        return "quality_failed"
    
    context["ti"].xcom_push(key="quality_status", value="PASS")
    return "publish_to_gold"


def _publish_to_gold(**context):
    """Publish validated data to Gold layer."""
    import psycopg2
    
    conn = psycopg2.connect(
        host="postgres", port=5432,
        user="admin", password="admin", dbname="warehouse"
    )
    cur = conn.cursor()
    
    # Refresh gold tables from dbt outputs
    cur.execute("""
        INSERT INTO gold.fct_revenue
        SELECT
            DATE(o.created_at),
            p.category,
            c.region,
            COUNT(DISTINCT o.id),
            COUNT(DISTINCT o.customer_id),
            SUM(o.amount),
            SUM(CASE WHEN o.status = 'completed' THEN o.amount ELSE 0 END),
            SUM(CASE WHEN o.status = 'refunded' THEN o.amount ELSE 0 END),
            SUM(CASE WHEN o.status = 'completed' THEN o.amount ELSE 0 END)
                - SUM(CASE WHEN o.status = 'refunded' THEN o.amount ELSE 0 END),
            ROUND(
                SUM(CASE WHEN o.status='refunded' THEN o.amount ELSE 0 END)
                / NULLIF(SUM(o.amount), 0) * 100, 2
            )
        FROM oltp.orders o
        JOIN oltp.customers c ON o.customer_id = c.id
        JOIN oltp.products p ON o.product_id = p.id
        GROUP BY DATE(o.created_at), p.category, c.region
        ON CONFLICT DO NOTHING
    """)
    
    conn.commit()
    cur.close()
    conn.close()
    print("✓ Published to Gold layer")


def _publish_metrics(**context):
    """Publish pipeline execution metrics."""
    import boto3
    from datetime import timezone
    
    dynamodb = boto3.resource("dynamodb", endpoint_url="http://localstack:4566",
                               region_name="us-east-1")
    table = dynamodb.Table("pipeline_metrics")
    
    table.put_item(Item={
        "pipeline_name": "dataflow_enterprise",
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "status": "success",
        "dag_run_id": context["run_id"],
        "execution_date": context["ds"],
    })
    print("✓ Metrics published")


def _quality_failed(**context):
    """Handle quality gate failure."""
    import boto3
    
    sqs = boto3.client("sqs", endpoint_url="http://localstack:4566",
                       region_name="us-east-1")
    sqs.send_message(
        QueueUrl="http://localstack:4566/000000000000/quality-alerts",
        MessageBody='{"alert":"Quality gate FAILED","pipeline":"dataflow_enterprise"}'
    )
    raise ValueError("Quality gate failed — pipeline paused")


with DAG(
    dag_id="dataflow_enterprise",
    default_args=default_args,
    description="Enterprise DataFlow — full batch pipeline",
    schedule_interval="0 6 * * *",  # Daily 06:00 UTC
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["dataflow", "enterprise", "medallion"],
    max_active_runs=1,
) as dag:

    validate_freshness = PythonOperator(
        task_id="validate_source_freshness",
        python_callable=_validate_source_freshness,
    )

    extract = PythonOperator(
        task_id="extract_to_bronze",
        python_callable=_extract_to_bronze,
    )

    dbt_run = BashOperator(
        task_id="dbt_run",
        bash_command="cd /opt/airflow/dbt && dbt run --profiles-dir .",
    )

    dbt_test = BashOperator(
        task_id="dbt_test",
        bash_command="cd /opt/airflow/dbt && dbt test --profiles-dir .",
    )

    quality_gate = BranchPythonOperator(
        task_id="quality_gate",
        python_callable=_run_quality_gate,
    )

    publish_gold = PythonOperator(
        task_id="publish_to_gold",
        python_callable=_publish_to_gold,
    )

    quality_fail = PythonOperator(
        task_id="quality_failed",
        python_callable=_quality_failed,
    )

    publish_metrics = PythonOperator(
        task_id="publish_metrics",
        python_callable=_publish_metrics,
        trigger_rule=TriggerRule.ONE_SUCCESS,
    )

    # DAG flow
    (validate_freshness >> extract >> dbt_run >> dbt_test
     >> quality_gate >> [publish_gold, quality_fail])
    publish_gold >> publish_metrics
```

### 3.4 — Python: Streaming Consumer

```python
# python/streaming/enterprise_consumer.py
import json
import time
import boto3
from confluent_kafka import Consumer, KafkaError
from datetime import datetime, timezone


class EnterpriseStreamConsumer:
    """Consome eventos CDC e escreve no S3 Bronze (micro-batch)."""

    def __init__(self, topics: list[str], batch_size: int = 100,
                 flush_interval_sec: int = 60):
        self.consumer = Consumer({
            "bootstrap.servers": "localhost:9092",
            "group.id": "dataflow-enterprise-consumer",
            "auto.offset.reset": "earliest",
            "enable.auto.commit": False,
        })
        self.consumer.subscribe(topics)
        self.s3 = boto3.client("s3", endpoint_url="http://localhost:4566")
        self.batch_size = batch_size
        self.flush_interval = flush_interval_sec
        self.buffer: dict[str, list] = {}
        self.last_flush = time.time()

    def run(self):
        """Main consumer loop."""
        print(f"Starting Enterprise Stream Consumer...")
        try:
            while True:
                msg = self.consumer.poll(1.0)
                if msg is None:
                    self._maybe_flush()
                    continue
                if msg.error():
                    if msg.error().code() == KafkaError._PARTITION_EOF:
                        continue
                    raise Exception(msg.error())

                topic = msg.topic()
                value = json.loads(msg.value().decode("utf-8"))
                
                # Mask PII before storing
                value = self._mask_pii(value)

                if topic not in self.buffer:
                    self.buffer[topic] = []
                self.buffer[topic].append(value)

                if self._should_flush():
                    self._flush_all()
                    self.consumer.commit()

        except KeyboardInterrupt:
            self._flush_all()
        finally:
            self.consumer.close()

    def _mask_pii(self, record: dict) -> dict:
        """Mask PII fields in CDC events."""
        payload = record.get("after", record)
        
        if "email" in payload:
            parts = payload["email"].split("@")
            payload["email"] = f"{parts[0][:2]}***@{parts[1]}"
        if "cpf" in payload:
            payload["cpf"] = f"***.***.{payload['cpf'][-6:]}"
        if "phone" in payload:
            payload["phone"] = f"****{payload['phone'][-4:]}"
        
        return record

    def _should_flush(self) -> bool:
        total = sum(len(v) for v in self.buffer.values())
        time_exceeded = (time.time() - self.last_flush) > self.flush_interval
        return total >= self.batch_size or time_exceeded

    def _maybe_flush(self):
        if self.buffer and (time.time() - self.last_flush) > self.flush_interval:
            self._flush_all()
            self.consumer.commit()

    def _flush_all(self):
        for topic, records in self.buffer.items():
            if not records:
                continue
            
            ts = datetime.now(timezone.utc)
            key = (f"bronze/{topic}/{ts.strftime('%Y/%m/%d')}/"
                   f"{ts.strftime('%H%M%S')}-{len(records)}.jsonl")
            
            body = "\n".join(json.dumps(r, default=str) for r in records)
            self.s3.put_object(
                Bucket="dataflow-bronze", Key=key,
                Body=body, ContentType="application/x-ndjson"
            )
            print(f"✓ Flushed {len(records)} records to s3://dataflow-bronze/{key}")

        self.buffer.clear()
        self.last_flush = time.time()


if __name__ == "__main__":
    consumer = EnterpriseStreamConsumer(
        topics=["orders.cdc", "customers.cdc"],
        batch_size=100,
        flush_interval_sec=30
    )
    consumer.run()
```

### 3.5 — Go: Enterprise Analytics API

```go
// go/cmd/dataflow-api/main.go
package main

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "time"

    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/dynamodb"
    _ "github.com/lib/pq"
)

type Server struct {
    db       *sql.DB
    dynamoDB *dynamodb.Client
}

type HealthResponse struct {
    Status    string `json:"status"`
    Database  string `json:"database"`
    Timestamp string `json:"timestamp"`
}

type RevenueResponse struct {
    Date            string  `json:"date"`
    Category        string  `json:"category"`
    Region          string  `json:"region"`
    GrossRevenue    float64 `json:"gross_revenue"`
    NetRevenue      float64 `json:"net_revenue"`
    RefundRate      float64 `json:"refund_rate"`
    NumOrders       int     `json:"num_orders"`
    UniqueCustomers int     `json:"unique_customers"`
}

type PlatformStats struct {
    TotalOrders       int     `json:"total_orders"`
    TotalRevenue      float64 `json:"total_revenue"`
    TotalCustomers    int     `json:"total_customers"`
    AvgOrderValue     float64 `json:"avg_order_value"`
    PipelineLastRunAt string  `json:"pipeline_last_run_at"`
    DataFreshnessHrs  float64 `json:"data_freshness_hours"`
}

func main() {
    db, err := sql.Open("postgres",
        "host=localhost port=5432 user=admin password=admin dbname=warehouse sslmode=disable")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    cfg, _ := config.LoadDefaultConfig(context.TODO(), config.WithRegion("us-east-1"))
    dynClient := dynamodb.NewFromConfig(cfg, func(o *dynamodb.Options) {
        o.BaseEndpoint = aws.String("http://localhost:4566")
    })

    srv := &Server{db: db, dynamoDB: dynClient}

    http.HandleFunc("/health", srv.healthHandler)
    http.HandleFunc("/api/v1/stats", srv.statsHandler)
    http.HandleFunc("/api/v1/revenue", srv.revenueHandler)
    http.HandleFunc("/api/v1/customers", srv.customersHandler)

    log.Println("DataFlow Analytics API on :8090")
    log.Fatal(http.ListenAndServe(":8090", nil))
}

func (s *Server) healthHandler(w http.ResponseWriter, _ *http.Request) {
    dbStatus := "ok"
    if err := s.db.Ping(); err != nil {
        dbStatus = "error"
    }
    json.NewEncoder(w).Encode(HealthResponse{
        Status:    "ok",
        Database:  dbStatus,
        Timestamp: time.Now().UTC().Format(time.RFC3339),
    })
}

func (s *Server) statsHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    var stats PlatformStats
    s.db.QueryRowContext(ctx, `
        SELECT COUNT(*), COALESCE(SUM(gross_revenue),0),
               COALESCE(SUM(unique_customers),0),
               COALESCE(AVG(gross_revenue / NULLIF(num_orders,0)),0)
        FROM gold.fct_revenue
    `).Scan(&stats.TotalOrders, &stats.TotalRevenue,
        &stats.TotalCustomers, &stats.AvgOrderValue)

    var lastUpdate time.Time
    s.db.QueryRowContext(ctx, "SELECT MAX(updated_at) FROM oltp.orders").Scan(&lastUpdate)
    stats.DataFreshnessHrs = time.Since(lastUpdate).Hours()
    stats.PipelineLastRunAt = lastUpdate.Format(time.RFC3339)

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(stats)
}

func (s *Server) revenueHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    query := `SELECT order_date, category, region, gross_revenue,
                     net_revenue, refund_rate, num_orders, unique_customers
              FROM gold.fct_revenue ORDER BY order_date DESC LIMIT 100`

    rows, err := s.db.QueryContext(ctx, query)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    defer rows.Close()

    var results []RevenueResponse
    for rows.Next() {
        var r RevenueResponse
        rows.Scan(&r.Date, &r.Category, &r.Region, &r.GrossRevenue,
            &r.NetRevenue, &r.RefundRate, &r.NumOrders, &r.UniqueCustomers)
        results = append(results, r)
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(results)
}

func (s *Server) customersHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    segment := r.URL.Query().Get("segment")
    query := `SELECT customer_id, customer_name, customer_segment,
                     total_orders, total_spent, days_since_last_order
              FROM gold.dim_customers WHERE 1=1`
    args := []interface{}{}

    if segment != "" {
        query += " AND customer_segment = $1"
        args = append(args, segment)
    }
    query += " ORDER BY total_spent DESC LIMIT 50"

    rows, err := s.db.QueryContext(ctx, query, args...)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    defer rows.Close()

    type Customer struct {
        ID        int     `json:"customer_id"`
        Name      string  `json:"name"`
        Segment   string  `json:"segment"`
        Orders    int     `json:"total_orders"`
        Spent     float64 `json:"total_spent"`
        DaysLast  int     `json:"days_since_last"`
    }
    var results []Customer
    for rows.Next() {
        var c Customer
        rows.Scan(&c.ID, &c.Name, &c.Segment, &c.Orders, &c.Spent, &c.DaysLast)
        results = append(results, c)
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(results)
}
```

### 3.6 — Java: Enterprise CDC Processor

```java
// java/src/.../capstone/EnterpriseCdcProcessor.java
@Service
public class EnterpriseCdcProcessor {

    private static final Logger log = LoggerFactory.getLogger(EnterpriseCdcProcessor.class);

    private final DynamoDbClient dynamoDb;
    private final S3Client s3;

    public EnterpriseCdcProcessor() {
        var endpoint = URI.create("http://localhost:4566");
        this.dynamoDb = DynamoDbClient.builder()
            .endpointOverride(endpoint).region(Region.US_EAST_1).build();
        this.s3 = S3Client.builder()
            .endpointOverride(endpoint).region(Region.US_EAST_1)
            .forcePathStyle(true).build();
    }

    public void processOrderEvent(Map<String, Object> cdcEvent) {
        var operation = (String) cdcEvent.get("op");
        var after = (Map<String, Object>) cdcEvent.get("after");

        switch (operation) {
            case "c", "r" -> handleInsert(after);
            case "u" -> handleUpdate(after);
            case "d" -> handleDelete((Map<String, Object>) cdcEvent.get("before"));
        }

        // Track lineage
        trackLineage("orders", operation, after);

        // Update real-time aggregate
        if ("c".equals(operation) || "u".equals(operation)) {
            updateRealTimeAggregate(after);
        }
    }

    private void handleInsert(Map<String, Object> record) {
        // Write to DynamoDB read model
        var item = Map.of(
            "order_id", AttributeValue.fromN(String.valueOf(record.get("id"))),
            "customer_id", AttributeValue.fromN(String.valueOf(record.get("customer_id"))),
            "amount", AttributeValue.fromN(String.valueOf(record.get("amount"))),
            "status", AttributeValue.fromS((String) record.get("status")),
            "processed_at", AttributeValue.fromS(Instant.now().toString())
        );
        dynamoDb.putItem(PutItemRequest.builder()
            .tableName("realtime_aggregates").item(item).build());
        log.info("Processed INSERT: order_id={}", record.get("id"));
    }

    private void handleUpdate(Map<String, Object> record) {
        dynamoDb.updateItem(UpdateItemRequest.builder()
            .tableName("realtime_aggregates")
            .key(Map.of("order_id",
                AttributeValue.fromN(String.valueOf(record.get("id")))))
            .updateExpression("SET #s = :s, amount = :a, updated_at = :t")
            .expressionAttributeNames(Map.of("#s", "status"))
            .expressionAttributeValues(Map.of(
                ":s", AttributeValue.fromS((String) record.get("status")),
                ":a", AttributeValue.fromN(String.valueOf(record.get("amount"))),
                ":t", AttributeValue.fromS(Instant.now().toString())))
            .build());
        log.info("Processed UPDATE: order_id={}", record.get("id"));
    }

    private void handleDelete(Map<String, Object> record) {
        dynamoDb.deleteItem(DeleteItemRequest.builder()
            .tableName("realtime_aggregates")
            .key(Map.of("order_id",
                AttributeValue.fromN(String.valueOf(record.get("id")))))
            .build());
        log.info("Processed DELETE: order_id={}", record.get("id"));
    }

    private void trackLineage(String dataset, String operation,
                                Map<String, Object> record) {
        var item = Map.of(
            "dataset", AttributeValue.fromS(dataset),
            "timestamp", AttributeValue.fromS(Instant.now().toString()),
            "operation", AttributeValue.fromS(operation),
            "source", AttributeValue.fromS("oltp.orders"),
            "target", AttributeValue.fromS("realtime_aggregates"),
            "record_id", AttributeValue.fromS(String.valueOf(record.get("id")))
        );
        dynamoDb.putItem(PutItemRequest.builder()
            .tableName("data_lineage").item(item).build());
    }

    private void updateRealTimeAggregate(Map<String, Object> record) {
        var windowKey = LocalDateTime.now()
            .truncatedTo(java.time.temporal.ChronoUnit.MINUTES)
            .toString();

        dynamoDb.updateItem(UpdateItemRequest.builder()
            .tableName("realtime_aggregates")
            .key(Map.of(
                "window_key", AttributeValue.fromS("orders_per_minute"),
                "window_start", AttributeValue.fromS(windowKey)))
            .updateExpression(
                "SET order_count = if_not_exists(order_count, :zero) + :one, "
                + "total_amount = if_not_exists(total_amount, :zero) + :amt")
            .expressionAttributeValues(Map.of(
                ":zero", AttributeValue.fromN("0"),
                ":one", AttributeValue.fromN("1"),
                ":amt", AttributeValue.fromN(String.valueOf(record.get("amount")))))
            .build());
    }
}
```

### 3.7 — Trino Configuration

```properties
# config/trino/catalog/iceberg.properties
connector.name=iceberg
hive.metastore=file
hive.metastore.catalog.dir=s3a://dataflow-silver/iceberg/
fs.native-s3.enabled=true
s3.endpoint=http://localstack:4566
s3.path-style-access=true
s3.aws-access-key=test
s3.aws-secret-key=test
s3.region=us-east-1
```

```properties
# config/trino/catalog/postgresql.properties
connector.name=postgresql
connection-url=jdbc:postgresql://postgres:5432/warehouse
connection-user=admin
connection-password=admin
```

---

## Parte 4 — Validação End-to-End

### 4.1 — Script de Validação

```bash
#!/bin/bash
# scripts/validate-platform.sh
set -e

echo "=== Enterprise DataFlow Platform Validation ==="

# 1. Infrastructure
echo "[1/8] Checking infrastructure..."
curl -sf http://localhost:4566/_localstack/health | python3 -c "
import sys, json; d = json.load(sys.stdin)
print(f'  LocalStack: {len(d[\"services\"])} services running')
"
docker compose exec -T postgres pg_isready -U admin > /dev/null && echo "  PostgreSQL: OK"
docker compose exec -T kafka kafka-broker-api-versions --bootstrap-server localhost:9092 2>/dev/null | head -1 && echo "  Kafka: OK"

# 2. S3 Buckets
echo "[2/8] Checking S3 buckets..."
for bucket in dataflow-bronze dataflow-silver dataflow-gold dataflow-landing; do
    aws --endpoint-url=http://localhost:4566 s3 ls "s3://$bucket/" > /dev/null 2>&1 && echo "  $bucket: OK"
done

# 3. DynamoDB Tables
echo "[3/8] Checking DynamoDB tables..."
aws --endpoint-url=http://localhost:4566 dynamodb list-tables --output text | tr '\t' '\n' | grep -v "^TABLENAMES$" | while read t; do
    echo "  $t: OK"
done

# 4. Bronze layer (data ingested)
echo "[4/8] Checking Bronze layer..."
count=$(aws --endpoint-url=http://localhost:4566 s3 ls s3://dataflow-bronze/ --recursive | wc -l)
echo "  Bronze objects: $count"

# 5. dbt
echo "[5/8] Running dbt..."
cd dbt && dbt run --profiles-dir . && dbt test --profiles-dir . && cd ..
echo "  dbt: PASS"

# 6. Gold layer
echo "[6/8] Checking Gold layer..."
docker compose exec -T postgres psql -U admin -d warehouse -c "SELECT COUNT(*) FROM gold.fct_revenue;" | head -3
docker compose exec -T postgres psql -U admin -d warehouse -c "SELECT COUNT(*) FROM gold.dim_customers;" | head -3

# 7. API
echo "[7/8] Checking Analytics API..."
curl -sf http://localhost:8090/health | python3 -c "import sys,json; print(f'  API: {json.load(sys.stdin)[\"status\"]}')"
curl -sf http://localhost:8090/api/v1/stats | python3 -c "
import sys,json; d=json.load(sys.stdin)
print(f'  Revenue: {d[\"total_revenue\"]} | Orders: {d[\"total_orders\"]}')
"

# 8. Trino
echo "[8/8] Checking Trino..."
docker compose exec -T trino trino --execute "SELECT count(*) FROM postgresql.gold.fct_revenue" 2>/dev/null | head -1

echo ""
echo "=== Platform Validation Complete ==="
```

---

## Critérios de Aceite

### Infraestrutura
- [ ] Docker Compose levanta todos os serviços (≥ 12 containers)
- [ ] LocalStack: S3 (4 buckets), DynamoDB (5 tables), Kinesis (2 streams), SQS, SNS, Glue
- [ ] PostgreSQL: OLTP schema + warehouse schemas (bronze, silver, gold)
- [ ] Kafka + Schema Registry + Debezium + Kafka UI funcionais

### Ingestão
- [ ] Batch extraction: OLTP → S3 Bronze (JSONL)
- [ ] CDC: Debezium captura changes de orders/customers
- [ ] Streaming consumer: Kafka → S3 Bronze (micro-batch)
- [ ] PII masking no streaming consumer (email, CPF, phone)

### Processamento
- [ ] dbt project: staging → intermediate → marts (Medallion)
- [ ] dbt testes: ≥ 10 testes passando
- [ ] Flink ou streaming aggregation funcional

### Storage
- [ ] Bronze: S3 (raw JSONL/Parquet)
- [ ] Silver: Iceberg ou PostgreSQL (cleaned, typed)
- [ ] Gold: PostgreSQL (facts + dimensions)

### Quality & Governance
- [ ] Quality gate integrado ao Airflow DAG
- [ ] Circuit breaker: pipeline pausa em falha de qualidade
- [ ] Data lineage tracking (DynamoDB)
- [ ] PII detection/masking

### Observability
- [ ] Pipeline metrics: DynamoDB (duration, status, rows)
- [ ] Source freshness check
- [ ] Health endpoint no Analytics API

### Serving
- [ ] Go Analytics API: /health, /api/v1/stats, /api/v1/revenue, /api/v1/customers
- [ ] Trino: queries sobre PostgreSQL + Iceberg
- [ ] DuckDB: queries ad-hoc sobre Parquet/Iceberg

### ADRs & Diagrams
- [ ] ADR-029 até ADR-034 completos (6 ADRs)
- [ ] DrawIO-023 até DrawIO-026 completos (4 DrawIOs)

---

## Definição de Pronto (DoD)

- [ ] 6 ADRs completos (platform architecture, ingestion, storage, transformation, quality/governance, observability)
- [ ] 4 DrawIOs (full architecture, batch pipeline, streaming pipeline, infrastructure)
- [ ] Docker Compose: full stack (≥ 12 services)
- [ ] Batch pipeline completo (Airflow → Extract → dbt → Quality → Gold)
- [ ] Streaming pipeline completo (CDC → Kafka → Consumer → S3 Bronze)
- [ ] Medallion Architecture funcional (Bronze → Silver → Gold)
- [ ] Quality gate + circuit breaker integrados
- [ ] Governance: lineage + PII masking + catalog
- [ ] Analytics API (Go) + CDC Processor (Java) funcionais
- [ ] Validação end-to-end script passando
- [ ] Commit: `feat(data-engineering-12): capstone enterprise dataflow`
