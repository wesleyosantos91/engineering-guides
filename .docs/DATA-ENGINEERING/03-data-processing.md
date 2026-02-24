# Data Engineering — Data Processing na AWS

> **Objetivo deste documento:** Servir como referência completa sobre **processamento de dados (batch e streaming) na AWS**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Foco em **AWS Glue, EMR, Kinesis, Lambda, Step Functions, MWAA** — quando usar, boas práticas, anti-patterns e patterns de implementação.

---

## Quick Reference — Cheat Sheet

| Serviço | Tipo | Latência | Managed | Quando usar |
|---------|------|----------|---------|-------------|
| **AWS Glue** | Batch ETL (Spark) | Minutos | Serverless | ETL padrão, crawlers, data catalog |
| **AWS Glue Streaming** | Micro-batch | Segundos | Serverless | Streaming simples com Spark |
| **EMR** | Batch/Stream (Spark/Flink/Hive) | Minutos | Managed Cluster | Workloads Spark complexos, controle total |
| **EMR Serverless** | Batch/Stream (Spark/Flink) | Minutos | Serverless | Spark/Flink sem cluster management |
| **Kinesis Data Streams** | Stream ingestion | ms | Serverless | Ingestão real-time, fan-out |
| **Kinesis Data Firehose** | Stream delivery | Segundos (buffer) | Serverless | Entrega no S3/Redshift/OpenSearch |
| **Kinesis Data Analytics** | Stream processing (Flink) | ms | Managed | SQL/Flink sobre streams |
| **Amazon MSK** | Stream (Kafka) | ms | Managed | Ecossistema Kafka existente |
| **MSK Serverless** | Stream (Kafka) | ms | Serverless | Kafka sem broker management |
| **Lambda** | Event-driven | ms | Serverless | Transformações leves, < 15min |
| **Step Functions** | Orchestration | — | Serverless | Pipeline orchestration, state machines |
| **MWAA** | Orchestration (Airflow) | — | Managed | DAGs complexos, dependências entre jobs |
| **Athena** | Interactive query | Segundos | Serverless | SQL ad-hoc sobre S3 |

---

## Sumário

- [Data Engineering — Data Processing na AWS](#data-engineering--data-processing-na-aws)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Árvore de Decisão — Qual motor de processamento?](#árvore-de-decisão--qual-motor-de-processamento)
  - [AWS Glue — ETL Serverless](#aws-glue--etl-serverless)
  - [Amazon EMR — Spark / Flink at Scale](#amazon-emr--spark--flink-at-scale)
  - [Amazon Kinesis — Streaming](#amazon-kinesis--streaming)
  - [Amazon MSK — Kafka Managed](#amazon-msk--kafka-managed)
  - [AWS Lambda — Event-Driven Processing](#aws-lambda--event-driven-processing)
  - [Amazon Athena — Interactive Queries](#amazon-athena--interactive-queries)
  - [Orquestração de Pipelines](#orquestração-de-pipelines)
  - [Patterns de Pipeline End-to-End](#patterns-de-pipeline-end-to-end)
  - [Anti-Patterns de Processamento](#anti-patterns-de-processamento)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Árvore de Decisão — Qual motor de processamento?

```
                         Qual é o requisito?
                              │
              ┌───────────────┼────────────────┐
              │               │                │
          Batch           Streaming       Interactive
              │               │             Query
              │               │                │
         ┌────┤          ┌────┤           Athena
         │    │          │    │
     Simples  Complexo  Simples Complexo
     ETL      ETL/ML    stream  stream
         │    │          │    │
     Glue     EMR    Kinesis  Kinesis Analytics
              │      Firehose (Flink) / MSK
              │      + Lambda  + Flink
              │
         ┌────┤
         │    │
    Serverless Precisa de
    ok?        controle total?
         │         │
    EMR         EMR on EC2
    Serverless  ou EKS
```

---

## AWS Glue — ETL Serverless

### Componentes do Glue

```
┌──────────────────────────────────────────────────────────────┐
│                        AWS GLUE                               │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ DATA CATALOG │  │ CRAWLERS     │  │ ETL JOBS     │       │
│  │              │  │              │  │              │        │
│  │ Hive-compat  │  │ Auto-discover│  │ PySpark /    │       │
│  │ metastore    │  │ schema from  │  │ Scala Spark  │       │
│  │              │  │ S3, RDS, etc │  │              │       │
│  │ Tables,      │  │              │  │ Visual ETL   │       │
│  │ Databases,   │  │ Schedule or  │  │ (no-code)    │       │
│  │ Partitions   │  │ on-demand    │  │              │       │
│  └──────────────┘  └──────────────┘  │ Flex/Standard│       │
│                                       │ execution    │       │
│  ┌──────────────┐  ┌──────────────┐  └──────────────┘       │
│  │ DATA QUALITY │  │ GLUE         │                          │
│  │              │  │ STREAMING    │  ┌──────────────┐       │
│  │ Rules,       │  │              │  │ GLUE         │       │
│  │ Validation,  │  │ Micro-batch  │  │ INTERACTIVE  │       │
│  │ Anomaly      │  │ from Kinesis │  │ SESSIONS     │       │
│  │ detection    │  │ or MSK       │  │              │       │
│  └──────────────┘  └──────────────┘  │ Jupyter/     │       │
│                                       │ Notebook     │       │
│  ┌──────────────┐                    └──────────────┘       │
│  │ WORKFLOWS    │                                            │
│  │              │  Triggers → Crawlers → Jobs → Crawlers    │
│  │ Orchestrate  │                                            │
│  │ multi-step   │                                            │
│  └──────────────┘                                            │
└──────────────────────────────────────────────────────────────┘
```

### Glue ETL Job — Boas Práticas

| Prática | Detalhes |
|---------|---------|
| **Job bookmarks** | Ative para evitar reprocessamento — rastreia o que já foi processado |
| **Worker type** | G.1X (standard), G.2X (memory-intensive), G.025X (flex low-cost) |
| **Flex execution** | Use para jobs não-urgentes — até 60% mais barato (usa spare capacity) |
| **Pushdown predicates** | Filtre no source antes de carregar — reduz dados processados |
| **Particionamento de saída** | Sempre particione output por data: `partitionKeys=["year","month","day"]` |
| **Formato de saída** | Parquet com Snappy compression — default para curated zone |
| **DPU sizing** | Comece com 2 DPUs, monitore e ajuste — over-provisioning é desperdício |
| **Timeout** | Configure timeout adequado — default é 48h (excessivo para maioria) |
| **Retry** | Configure 1-2 retries com backoff para resiliência |
| **Catalogar output** | Atualize Data Catalog com schema do output — descobribilidade |

### Glue ETL — Pattern Típico

```python
# Pseudocódigo — Glue ETL Job (PySpark)

# 1. Ler dados do Data Catalog (ou S3 diretamente)
raw_data = glue_context.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="orders",
    push_down_predicate="year='2025' AND month='01'"  # Pushdown!
)

# 2. Transformações
# Remover duplicatas
deduped = raw_data.drop_duplicates(["order_id"])

# Resolver tipos (DynamicFrame → tipado)
resolved = deduped.resolveChoice(specs=[
    ("price", "cast:double"),
    ("quantity", "cast:int")
])

# Filtrar registros inválidos
filtered = resolved.filter(
    lambda row: row["status"] in ["COMPLETED", "PENDING", "SHIPPED"]
)

# Enriquecer com lookup
customers = glue_context.create_dynamic_frame.from_catalog(
    database="curated_db",
    table_name="customers"
)
enriched = filtered.join(
    paths1=["customer_id"],
    paths2=["id"],
    frame2=customers
)

# 3. Escrever na curated zone (Parquet, particionado)
glue_context.write_dynamic_frame.from_options(
    frame=enriched,
    connection_type="s3",
    connection_options={
        "path": "s3://data-lake/curated/domain=sales/entity=orders/",
        "partitionKeys": ["year", "month"]
    },
    format="parquet",
    format_options={"compression": "snappy"}
)

# 4. Atualizar Data Catalog
# (automático se usar Crawler pós-job, ou manual via boto3)
```

### Glue Data Quality

```python
# Pseudocódigo — Glue Data Quality Rules

rules = """
    Rules = [
        ColumnExists "order_id",
        ColumnExists "customer_id",
        ColumnExists "total",
        
        Completeness "order_id" > 0.99,        # 99% non-null
        Completeness "customer_id" > 0.95,
        
        ColumnValues "total" > 0,               # Positivo
        ColumnValues "status" in ["PENDING", "COMPLETED", "SHIPPED", "CANCELLED"],
        
        Uniqueness "order_id" > 0.999,          # Quase unique
        
        RowCount > 1000,                        # Mínimo esperado
        RowCount < 10000000,                    # Máximo esperado (anomaly)
        
        CustomSql "SELECT COUNT(*) FROM primary WHERE total > 100000" < 10  # Outliers
    ]
"""

# Integrar no pipeline
quality_result = evaluate_data_quality(
    frame=enriched,
    ruleset=rules,
    publishing_options={"cloudwatch_metrics": True}
)

# Se qualidade falha → alertar e não promover para curated
if quality_result.overall_status == "FAIL":
    publish_to_sns("Data Quality FAILED for orders pipeline")
    quarantine_bad_records(quality_result.failed_rows, "s3://lake/quarantine/")
```

---

## Amazon EMR — Spark / Flink at Scale

### Quando usar EMR vs Glue

| Critério | Glue | EMR | EMR Serverless |
|----------|------|-----|----------------|
| **Complexidade do ETL** | Simples a médio | Complexo, ML, custom | Médio a complexo |
| **Runtime** | Spark (managed) | Spark, Flink, Hive, Presto, Trino | Spark, Flink |
| **Controle** | Limitado (Glue API) | Total (SSH, custom AMI) | Limitado |
| **Custo** | DPU/hora | EC2/hora (spot!) | vCPU+mem/hora |
| **Startup time** | ~1-2 min (warm) | ~5-15 min (cluster) | ~1-2 min |
| **Notebooks** | Glue Interactive Sessions | EMR Studio / JupyterHub | EMR Studio |
| **Caso ideal** | ETL standard, Data Catalog | Data science, ML, custom libs | Spark/Flink sem ops |

### EMR — Deployment Options

```
EMR on EC2 (classic):
  - Cluster management completo
  - Spot instances (até 90% economia)
  - Bootstrap actions, custom AMI
  - GPU instances para ML
  
EMR on EKS:
  - Spark/Flink sobre Kubernetes
  - Compartilha cluster K8s com outros workloads
  - Melhor utilização de recursos
  
EMR Serverless:
  - Sem cluster management
  - Auto-scaling por job
  - Pay-per-use (vCPU/GB-hour)
  - Startup rápido (~1-2 min)
  - Recomendado para novos projetos
```

### Boas Práticas EMR

| Prática | Detalhes |
|---------|---------|
| **Spot instances** | 60-90% dos workers em Spot — economia massiva |
| **Instance fleets** | Diversifique tipos de instância para Spot availability |
| **S3 como storage** | EMRFS (S3) > HDFS — persistência e custo |
| **Iceberg** | Use Iceberg para tabelas com ACID requirements |
| **Autoscaling** | Configure managed scaling para ajustar workers |
| **Graviton** | Use instâncias Graviton (M6g, R6g) — 20% melhor preço/performance |
| **Cluster per job** | Transient clusters para jobs batch; persistent para interativo |
| **Step Functions** | Orquestre EMR jobs com Step Functions |
| **EMR Serverless** | Prefira para novos projetos — zero ops |

---

## Amazon Kinesis — Streaming

### Kinesis Data Streams

```
┌──────────────────────────────────────────────────────┐
│              KINESIS DATA STREAMS                      │
├──────────────────────────────────────────────────────┤
│                                                       │
│  Producers                 Stream              Consumers
│  ─────────                 ──────              ─────────
│  SDK (PutRecord)     ┌──────────────┐    Lambda
│  KPL (batch)         │  Shard 1     │    KCL (Java/Python)
│  Kinesis Agent       │  Shard 2     │    Kinesis Firehose
│  IoT Core            │  Shard 3     │    Kinesis Analytics
│  CloudWatch Logs     │  ...         │    EMR (Spark Streaming)
│  DynamoDB Streams    │  Shard N     │    Glue Streaming
│                      └──────────────┘    
│                                                       │
│  Capacity per shard:                                  │
│  - Write: 1 MB/s ou 1000 records/s                   │
│  - Read: 2 MB/s (shared) ou 2 MB/s per consumer      │
│           (enhanced fan-out)                          │
│                                                       │
│  Retention: 24h (default) → 7 days → 365 days        │
│                                                       │
│  Modes:                                               │
│  - On-Demand: auto-scaling, pay-per-throughput       │
│  - Provisioned: manual shard management              │
└──────────────────────────────────────────────────────┘
```

### Kinesis Data Firehose

```
Sources ──▶ Firehose ──▶ Destinations
                │
        ┌───────┤
        │       │
   Buffer    Transform
   (60s-900s  (Lambda)
    1MB-128MB)
                │
        ┌───────┼───────────┐──────────┐
        │       │           │          │
       S3    Redshift   OpenSearch   Splunk
              (via S3)

Firehose features:
- Formato conversion: JSON → Parquet/ORC (nativo!)
- Dynamic partitioning: particionar por campo do record
- Compression: GZIP, Snappy, ZIP
- Encryption: SSE-KMS
- Error handling: failed records → S3 error bucket
```

### Boas Práticas Kinesis

| Prática | Detalhes |
|---------|---------|
| **On-Demand mode** | Prefira para workloads imprevisíveis — escala automaticamente |
| **Enhanced Fan-Out** | Use quando > 2 consumers — 2MB/s dedicado por consumer |
| **KPL (Producer Library)** | Batch, retry, aggregation automáticos — melhor throughput |
| **Firehose buffer** | Configure buffer size/time baseado no trade-off latência vs custo |
| **Firehose Parquet conversion** | Converta JSON→Parquet no Firehose — evita ETL adicional |
| **Dynamic partitioning** | Particione output do Firehose por campo (ex: `customer_id`) |
| **Error handling** | Configure S3 error output para records que falharam |
| **Monitoring** | `IteratorAgeMilliseconds` no CloudWatch — se cresce, consumer está atrasado |
| **Retry com backoff** | Configure retry no producer com exponential backoff |
| **Encryption** | SSE com KMS para dados sensíveis |

### Pattern: Real-time Analytics Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│          REAL-TIME ANALYTICS PIPELINE                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  App/IoT ──▶ Kinesis Data Streams                           │
│                    │                                         │
│              ┌─────┼─────────────────┐                      │
│              │     │                 │                       │
│              ▼     ▼                 ▼                       │
│         Firehose  Kinesis         Lambda                    │
│              │    Analytics         │                        │
│              │    (Flink)          │                        │
│              │     │               │                        │
│              ▼     ▼               ▼                        │
│          S3     DynamoDB      ElastiCache                   │
│        (lake)  (real-time    (real-time                     │
│                 aggregates)   cache)                        │
│              │                                               │
│              ▼                                               │
│          Athena / Redshift                                   │
│          (historical analytics)                              │
│              │                                               │
│              ▼                                               │
│          QuickSight (dashboards)                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Amazon MSK — Kafka Managed

### Quando usar MSK vs Kinesis

| Critério | Kinesis | MSK | MSK Serverless |
|----------|---------|-----|----------------|
| **Ecossistema** | AWS nativo | Kafka nativo | Kafka nativo |
| **Protocolo** | AWS SDK / HTTP | Kafka protocol | Kafka protocol |
| **Consumers** | Lambda, KCL, Firehose | Kafka consumers, Connect | Kafka consumers |
| **Retenção** | 1-365 dias | Ilimitada (storage) | Ilimitada |
| **Ordering** | Por shard | Por partition | Por partition |
| **Management** | Zero (serverless) | Broker management | Zero (serverless) |
| **Kafka Connect** | ❌ | ✅ | ❌ |
| **Schema Registry** | ❌ (use Glue) | Glue Schema Registry | Glue Schema Registry |
| **Caso ideal** | AWS-native, simples | Kafka existente, Connect | Kafka sem ops |

### MSK para Data Engineering

```
Pattern: Event-Driven Data Platform com MSK

Sources (CDC, Apps, IoT)
         │
         ▼
┌─────────────────┐
│   Amazon MSK    │
│   ┌───────────┐ │
│   │ Topic:    │ │     ┌──────────────────┐
│   │ orders    │─┼────▶│ Flink on EMR     │──▶ DynamoDB (real-time)
│   │           │ │     └──────────────────┘
│   │ Topic:    │ │     ┌──────────────────┐
│   │ customers │─┼────▶│ MSK Connect      │──▶ S3 (data lake)
│   │           │ │     │ (S3 Sink)        │
│   │ Topic:    │ │     └──────────────────┘
│   │ clickstrm │ │     ┌──────────────────┐
│   └───────────┘ │     │ Glue Streaming   │──▶ S3 (Parquet)
│                 │────▶│                  │
└─────────────────┘     └──────────────────┘

MSK Connect: Kafka Connect managed
  - S3 Sink Connector (→ S3 lake)
  - Debezium (CDC from RDS/Aurora)
  - OpenSearch Sink (→ search index)
  - JDBC Sink (→ Redshift/Aurora)
```

---

## AWS Lambda — Event-Driven Processing

### Lambda para Data Engineering

| Cenário | Lambda? | Alternativa |
|---------|---------|-------------|
| Transformação leve (< 15 min, < 10GB) | ✅ | — |
| Trigger de pipeline (S3 event → start Glue) | ✅ | EventBridge |
| Enriquecimento de stream (Kinesis/MSK) | ✅ | Flink |
| ETL pesado (> 15 min, > 10GB RAM) | ❌ | Glue / EMR |
| Processamento stateful complexo | ❌ | Flink / Step Functions |
| Firehose transformation | ✅ | — |

### Patterns com Lambda

```
Pattern 1: S3 Event → Lambda → Glue trigger
═══════════════════════════════════════════
S3 (PutObject raw/) ──▶ EventBridge ──▶ Lambda
                                          │
                                     Start Glue Job
                                     (pass S3 path as parameter)

Pattern 2: Kinesis → Lambda → DynamoDB (enrichment)
════════════════════════════════════════════════════
Kinesis ──▶ Lambda (batch 100 records)
                 │
            For each record:
              1. Parse JSON
              2. Validate schema
              3. Enrich (lookup DynamoDB/ElastiCache)
              4. Write to DynamoDB / S3
              5. Dead letter → SQS

Pattern 3: Firehose Transform
═════════════════════════════
Firehose ──▶ Lambda (transform)
                 │
            For each record:
              1. Base64 decode
              2. Parse / transform
              3. Add metadata (timestamp, source)
              4. Return: Ok | Dropped | ProcessingFailed
                 │
            ──▶ S3 (Parquet, partitioned)

Pattern 4: Step Functions → Lambda chain (micro-pipeline)
═════════════════════════════════════════════════════════
Step Functions:
  1. Lambda: validate input
  2. Lambda: fetch reference data
  3. Lambda: transform & enrich
  4. Lambda: write output
  5. Lambda: update catalog
  Error → SNS notification
```

### Boas Práticas Lambda para DE

| Prática | Detalhes |
|---------|---------|
| **Batch size** | Kinesis/SQS: configure batch size e window para eficiência |
| **Powertools** | Use Lambda Powertools para logging, tracing, metrics |
| **Layers** | Dependências pesadas (pandas, pyarrow) em Lambda Layers |
| **Memory = CPU** | Mais memória = mais CPU — ajuste para workload |
| **Reserved concurrency** | Limite concurrency para não sobrecarregar downstream |
| **Dead Letter Queue** | DLQ (SQS) para records que falharam — SEMPRE configure |
| **Idempotência** | Use Powertools Idempotency para evitar duplicatas |
| **Timeout** | Configure timeout < 15 min com margem — evite zombie executions |

---

## Amazon Athena — Interactive Queries

### Quando usar Athena

| Cenário | Athena? | Alternativa |
|---------|---------|-------------|
| SQL ad-hoc sobre S3 | ✅ | — |
| Queries recorrentes sobre data lake | ✅ | — |
| Dashboard BI complexo | ❌ (latência variável) | Redshift |
| Sub-second latency | ❌ | Redshift / DynamoDB |
| Queries sobre Iceberg tables | ✅ | — |
| Federated query (RDS, DynamoDB, etc.) | ✅ | — |

### Boas Práticas Athena

| Prática | Impacto | Detalhes |
|---------|---------|---------|
| **Parquet/ORC** | -90% custo | Formato colunar = scan apenas colunas necessárias |
| **Particionamento** | -90% custo | WHERE year='2025' AND month='01' → scan mínimo |
| **Compressão** | -50% custo | Snappy/ZSTD sobre Parquet |
| **Tamanho de arquivo** | -30% tempo | 128MB-1GB por arquivo — evite small files |
| **CTAS para materializar** | Reduz re-scan | `CREATE TABLE AS SELECT ...` para resultados frequentes |
| **Iceberg tables** | ACID + perf | Time-travel, compaction, schema evolution |
| **Workgroups** | Governance | Separe workgroups por time — controle custo e acesso |
| **Query result reuse** | -100% custo | Ative reuso de resultados para queries repetitivas |
| **Prepared statements** | Segurança | Evite SQL injection em queries parametrizadas |

### Athena + Iceberg

```sql
-- Criar tabela Iceberg no Athena
CREATE TABLE curated_db.orders (
    order_id BIGINT,
    customer_id BIGINT,
    total DECIMAL(12,2),
    status STRING,
    created_at TIMESTAMP
)
PARTITIONED BY (month(created_at))  -- Hidden partitioning!
LOCATION 's3://data-lake/curated/orders/'
TBLPROPERTIES (
    'table_type' = 'ICEBERG',
    'format' = 'PARQUET',
    'write_compression' = 'ZSTD'
);

-- Time travel
SELECT * FROM curated_db.orders
FOR TIMESTAMP AS OF TIMESTAMP '2025-01-01 00:00:00';

-- Schema evolution
ALTER TABLE curated_db.orders ADD COLUMNS (
    discount DECIMAL(5,2),
    coupon_code STRING
);

-- Compaction (otimizar small files)
OPTIMIZE curated_db.orders REWRITE DATA USING BIN_PACK;

-- Expire snapshots (liberar storage)
ALTER TABLE curated_db.orders SET TBLPROPERTIES (
    'vacuum_max_snapshot_age_seconds' = '604800'  -- 7 dias
);
VACUUM curated_db.orders;
```

---

## Orquestração de Pipelines

### Step Functions vs MWAA (Airflow)

| Aspecto | Step Functions | MWAA (Airflow) |
|---------|---------------|----------------|
| **Modelo** | State machine (JSON/YAML) | DAG (Python) |
| **Complexidade** | Simples a médio | Médio a complexo |
| **Integração AWS** | Nativa (200+ AWS APIs) | Via operators/hooks |
| **Custo** | Pay-per-transition | Ambiente fixo + workers |
| **UI** | Visual workflow | Airflow UI completa |
| **Scheduling** | EventBridge | Built-in scheduler |
| **Dependências entre DAGs** | Via EventBridge | Nativo (DAG dependencies) |
| **Caso ideal** | Pipelines AWS-native | DAGs complexos, multi-tool |

### Step Functions — Data Pipeline Pattern

```json
// Pseudocódigo — Step Functions (ASL simplificado)

{
  "StartAt": "StartCrawler",
  "States": {
    "StartCrawler": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startCrawler",
      "Parameters": {"Name": "raw-orders-crawler"},
      "Next": "WaitForCrawler"
    },
    "WaitForCrawler": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "CheckCrawlerStatus"
    },
    "CheckCrawlerStatus": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:getCrawler",
      "Parameters": {"Name": "raw-orders-crawler"},
      "Next": "IsCrawlerDone"
    },
    "IsCrawlerDone": {
      "Type": "Choice",
      "Choices": [{
        "Variable": "$.Crawler.State",
        "StringEquals": "READY",
        "Next": "StartGlueJob"
      }],
      "Default": "WaitForCrawler"
    },
    "StartGlueJob": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": {
        "JobName": "orders-etl",
        "Arguments": {"--source_date": "$.date"}
      },
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "NotifyFailure"
      }],
      "Next": "RunDataQuality"
    },
    "RunDataQuality": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startDataQualityRulesetEvaluationRun.sync",
      "Next": "CheckQuality"
    },
    "CheckQuality": {
      "Type": "Choice",
      "Choices": [{
        "Variable": "$.Status",
        "StringEquals": "SUCCEEDED",
        "Next": "PromoteToCurated"
      }],
      "Default": "QuarantineData"
    },
    "PromoteToCurated": {
      "Type": "Task",
      "Resource": "Lambda:promote-to-curated",
      "Next": "Success"
    },
    "QuarantineData": {
      "Type": "Task",
      "Resource": "Lambda:quarantine-bad-data",
      "Next": "NotifyFailure"
    },
    "NotifyFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:...:data-pipeline-alerts",
        "Message": "Pipeline failed"
      },
      "Next": "Failed"
    },
    "Success": {"Type": "Succeed"},
    "Failed": {"Type": "Fail"}
  }
}
```

### MWAA — DAG Pattern

```python
# Pseudocódigo — Airflow DAG para data pipeline

dag = DAG(
    "orders_daily_pipeline",
    schedule="0 6 * * *",      # 6h UTC diário
    catchup=False,
    default_args={
        "retries": 2,
        "retry_delay": timedelta(minutes=5),
        "on_failure_callback": notify_slack
    }
)

# Tasks
check_source = S3KeySensor(
    task_id="check_source_data",
    bucket_name="raw-data",
    bucket_key="orders/{{ ds }}/",
    timeout=3600
)

run_crawler = GlueCrawlerOperator(
    task_id="crawl_raw_orders",
    crawler_name="raw-orders-crawler"
)

run_etl = GlueJobOperator(
    task_id="etl_orders",
    job_name="orders-etl",
    script_args={"--source_date": "{{ ds }}"},
    num_of_dpus=5
)

check_quality = GlueDataQualityOperator(
    task_id="validate_quality",
    ruleset_name="orders-quality-rules"
)

promote = LambdaInvokeFunctionOperator(
    task_id="promote_to_curated",
    function_name="promote-to-curated",
    payload={"date": "{{ ds }}"}
)

refresh_athena = AthenaOperator(
    task_id="refresh_athena_partitions",
    query="MSCK REPAIR TABLE curated_db.orders",
    database="curated_db"
)

notify = SlackWebhookOperator(
    task_id="notify_success",
    message="Orders pipeline completed for {{ ds }}"
)

# Dependencies
check_source >> run_crawler >> run_etl >> check_quality >> promote >> refresh_athena >> notify
```

---

## Patterns de Pipeline End-to-End

### Pattern 1: Batch — S3 Landing → Lakehouse

```
S3 (Raw)
   │
   ▼ (EventBridge — new file)
Step Functions
   │
   ├── 1. Glue Crawler (discover schema)
   ├── 2. Glue ETL (clean, dedup, type)
   ├── 3. Glue Data Quality (validate)
   ├── 4. Write Iceberg (curated zone)
   ├── 5. Update Data Catalog
   └── 6. Notify (SNS/Slack)
            │
            ▼
   Athena / Redshift Spectrum / QuickSight
```

### Pattern 2: Streaming — Real-time + Historical

```
App Events ──▶ Kinesis Data Streams ──┬──▶ Kinesis Analytics (Flink)
                                      │         │
                                      │    Real-time aggregations
                                      │         │
                                      │    DynamoDB (hot data)
                                      │
                                      └──▶ Kinesis Firehose
                                                │
                                           Buffer 5min
                                           JSON → Parquet
                                                │
                                           S3 (data lake)
                                                │
                                           Athena (historical)
                                                │
                                           QuickSight
```

### Pattern 3: CDC — Database → Lake

```
Aurora (source of truth)
   │
   ▼ (DMS — CDC — ongoing replication)
Kinesis Data Streams
   │
   ├──▶ Lambda (transform CDC format)
   │         │
   │    S3 Raw (CDC events: INSERT/UPDATE/DELETE)
   │         │
   │    Glue ETL (apply CDC to Iceberg — MERGE INTO)
   │         │
   │    S3 Curated (Iceberg table — current state)
   │
   └──▶ Lambda (real-time materialized view)
              │
         DynamoDB (latest state for API)
```

---

## Anti-Patterns de Processamento

| Anti-pattern | Problema | Solução |
|-------------|----------|---------|
| **Lambda para ETL pesado** | Timeout 15min, 10GB RAM max | Use Glue ou EMR |
| **Glue para streaming complexo** | Micro-batch não serve para sub-second | Use Kinesis Analytics (Flink) |
| **EMR cluster always-on para batch** | Desperdício de custo | Transient clusters ou EMR Serverless |
| **Kinesis sem DLQ** | Records perdidos silenciosamente | On-failure → SQS DLQ |
| **Firehose buffer muito pequeno** | Small files no S3 | Buffer ≥ 128MB ou 5min |
| **Athena sem particionamento** | Full scan = caro e lento | Particione por data |
| **Glue job sem bookmark** | Reprocessa tudo a cada execução | Ative job bookmarks |
| **Pipeline sem retry** | Falha transitória derruba o pipeline | Retry com exponential backoff |
| **Pipeline sem alerting** | Falhas passam despercebidas | SNS/Slack em cada failure |
| **Over-engineering real-time** | Streaming quando batch de 1h resolve | Avalie se latência de minutos/horas é suficiente |
| **Sem data quality checks** | Dados ruins propagam silenciosamente | Glue Data Quality em cada estágio |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código de data processing na AWS, verifique:

1. **Glue job sem bookmark** — Bookmark evita reprocessamento; deve estar ativo
2. **Glue sem pushdown predicate** — Ler tabela inteira quando precisa de partition específica
3. **Lambda processando > 10GB** — Excede limite; sugira Glue ou EMR Serverless
4. **Kinesis consumer sem checkpoint** — Risco de reprocessamento ou perda de dados
5. **Firehose sem error output** — Records que falham devem ir para S3 error bucket
6. **Athena query sem WHERE em partition** — Full scan caro; exija filtro de partition
7. **EMR cluster sem Spot instances** — Workers devem usar Spot para economia
8. **Step Functions sem Catch** — Todo Task deve ter error handling
9. **MWAA DAG sem retries** — Configure retries com backoff
10. **Pipeline sem data quality check** — Validação de dados é obrigatória entre estágios
11. **Output em CSV/JSON** — Curated zone deve ser Parquet/ORC; sugira conversão
12. **Sem monitoramento de IteratorAge** — Para Kinesis consumers, monitore atraso
13. **Lambda sem DLQ** — Records que falham devem ir para SQS DLQ
14. **Hardcoded credentials** — Use IAM roles, nunca access keys em código

---

## Referências

- **AWS Glue Developer Guide** — ETL, crawlers, Data Catalog
- **Amazon EMR Best Practices Guide** — Sizing, Spot, performance
- **Amazon Kinesis Developer Guide** — Streaming patterns
- **AWS Step Functions Developer Guide** — Orchestration patterns
- **Streaming Data** — Andrew Psaltis — Conceitos de stream processing
- **Spark: The Definitive Guide** — Bill Chambers, Matei Zaharia — Spark internals
- **Designing Data-Intensive Applications** — Martin Kleppmann — Fundamental de processamento
