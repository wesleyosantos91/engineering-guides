# Level 03 — Batch Processing

> **Objetivo:** Implementar pipelines de processamento batch com PySpark, dbt e
> Spring Batch para transformar grandes volumes de dados no data lake.

**Referência:** [03 — Data Processing](../../.docs/data-engineering/03-data-processing.md)

**Pré-requisito:** Level 02 concluído.

---

## Contexto

Batch processing é o paradigma mais comum em data engineering. Dados são coletados
em janelas de tempo (hourly, daily) e processados em lotes. Neste nível, usamos
**PySpark** para transformações distribuídas, **dbt** para SQL-based transforms
e **Spring Batch** para pipelines Java robustos.

---

## Parte 1 — ADRs (3 obrigatórios)

### ADR-006: Batch Processing Engine

**Arquivo:** `docs/adrs/ADR-006-batch-engine.md`

**Context:** Precisamos processar datasets de 10GB–1TB em batch diário.

**Options:**
1. **PySpark (local/EMR)** — engine distribuído, ecossistema Spark completo
2. **Pandas + PyArrow** — single-node, simples, limitado a ~10GB RAM
3. **DuckDB** — analytical engine, SQL-first, single-node performático
4. **Polars** — DataFrame library em Rust, 10-100x mais rápido que Pandas
5. **AWS Glue (PySpark)** — serverless Spark gerenciado

**Decision Drivers:**
- Escalabilidade (10GB → 1TB)
- Custo de infraestrutura
- Experiência do time
- Integração com S3/data lake

---

### ADR-007: ETL vs ELT Strategy

**Arquivo:** `docs/adrs/ADR-007-etl-vs-elt.md`

**Context:** Definir se transformações acontecem antes (ETL) ou depois (ELT)
do carregamento no warehouse/lake.

**Options:**
1. **ETL clássico** — Extract → Transform (PySpark) → Load (warehouse)
2. **ELT moderno** — Extract → Load (raw) → Transform (dbt/SQL in-warehouse)
3. **Hybrid** — ETL para heavy lifting (Spark) + ELT para refinamento (dbt)
4. **Zero-ETL** — integração direta sem pipeline (ex: Aurora → Redshift)

**Decision Drivers:**
- Complexidade de transformação
- Latência aceitável
- Custo de compute vs storage
- Manutenibilidade dos pipelines

---

### ADR-008: Transformation Framework

**Arquivo:** `docs/adrs/ADR-008-transformation-framework.md`

**Context:** Precisamos de um framework padronizado para transformações SQL.

**Options:**
1. **dbt-core** — SQL transforms com lineage, tests, docs
2. **SQLMesh** — alternativa ao dbt com virtual environments
3. **Custom Python scripts** — flexibilidade total, sem framework
4. **Stored Procedures** — transformações no banco

**Decision Drivers:**
- Testabilidade (unit tests em SQL)
- Lineage e documentação automática
- Versionamento de modelos
- Comunidade e ecossistema

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

### DrawIO-005: Batch Pipeline Architecture

**Arquivo:** `docs/diagrams/03-batch-pipeline.drawio`

```
┌─────────┐    ┌──────────┐    ┌───────────┐    ┌─────────┐
│ Sources │───▶│ Extract  │───▶│ Transform │───▶│  Load   │
│         │    │          │    │           │    │         │
│ CSV     │    │ Python   │    │ PySpark   │    │ S3      │
│ API     │    │ boto3    │    │ dbt       │    │ Parquet │
│ DB      │    │ JDBC     │    │ SQL       │    │ Postgres│
└─────────┘    └──────────┘    └───────────┘    └─────────┘
                                    │
                              ┌─────▼──────┐
                              │  Quality   │
                              │  Checks    │
                              │  (GE/dbt)  │
                              └────────────┘
```

### DrawIO-006: dbt Transformation DAG

**Arquivo:** `docs/diagrams/03-dbt-dag.drawio`

```
stg_orders ──────┐
                  ├──▶ int_orders_enriched ──┐
stg_customers ───┘                           ├──▶ mart_sales_summary
                                             │
stg_products ─── int_product_metrics ────────┘
```

---

## Parte 3 — Implementação

### 3.1 — PySpark ETL Pipeline

**Objetivo:** Pipeline Spark que lê do S3 raw, transforma, grava no curated.

```python
# python/etl/batch/spark_etl.py
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.types import StructType, StructField, StringType, DoubleType, IntegerType, TimestampType

def create_spark():
    return (SparkSession.builder
        .appName("DataEngineeringBatch")
        .master("local[*]")
        .config("spark.jars.packages", "org.apache.hadoop:hadoop-aws:3.3.4")
        .config("spark.hadoop.fs.s3a.endpoint", "http://localhost:4566")
        .config("spark.hadoop.fs.s3a.access.key", "test")
        .config("spark.hadoop.fs.s3a.secret.key", "test")
        .config("spark.hadoop.fs.s3a.path.style.access", "true")
        .config("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
        .getOrCreate())

def extract(spark: SparkSession, source: str):
    """Extract: ler raw data do S3."""
    return spark.read.json(f"s3a://data-lake-raw/{source}")

def transform(df):
    """Transform: limpeza, enriquecimento, agregação."""
    return (df
        .filter(F.col("id").isNotNull())
        .dropDuplicates(["id"])
        .withColumn("amount_brl", F.col("amount") * F.lit(5.0))
        .withColumn("order_date", F.to_date("timestamp"))
        .withColumn("year", F.year("order_date"))
        .withColumn("month", F.month("order_date"))
        .withColumn("processed_at", F.current_timestamp())
    )

def load(df, target: str):
    """Load: gravar Parquet particionado no curated zone."""
    (df.write
        .mode("overwrite")
        .partitionBy("year", "month")
        .parquet(f"s3a://data-lake-curated/{target}"))

def run_pipeline(source: str, target: str):
    spark = create_spark()
    raw = extract(spark, source)
    transformed = transform(raw)
    load(transformed, target)
    
    print(f"✓ Batch pipeline: {source} → {target}")
    print(f"  Rows processed: {transformed.count()}")
    spark.stop()

if __name__ == "__main__":
    run_pipeline("orders/", "orders_curated/")
```

### 3.2 — dbt Project Setup

**Objetivo:** Projeto dbt para transformações ELT no PostgreSQL warehouse.

```yaml
# dbt/dbt_project.yml
name: 'data_engineering'
version: '1.0.0'
profile: 'data_engineering'

model-paths: ["models"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

models:
  data_engineering:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: ephemeral
    marts:
      +materialized: table
      +schema: marts
```

```sql
-- dbt/models/staging/stg_orders.sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'orders') }}
),

renamed AS (
    SELECT
        id              AS order_id,
        customer_id,
        product_id,
        quantity,
        amount          AS total_amount,
        status,
        created_at::timestamp AS ordered_at,
        CURRENT_TIMESTAMP     AS loaded_at
    FROM source
    WHERE id IS NOT NULL
)

SELECT * FROM renamed
```

```sql
-- dbt/models/intermediate/int_orders_enriched.sql
WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
),

customers AS (
    SELECT * FROM {{ ref('stg_customers') }}
),

products AS (
    SELECT * FROM {{ ref('stg_products') }}
)

SELECT
    o.order_id,
    o.ordered_at,
    c.name          AS customer_name,
    c.segment       AS customer_segment,
    p.name          AS product_name,
    p.category      AS product_category,
    o.quantity,
    o.total_amount,
    o.status
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN products p ON o.product_id = p.product_id
```

```sql
-- dbt/models/marts/mart_sales_summary.sql
WITH enriched AS (
    SELECT * FROM {{ ref('int_orders_enriched') }}
)

SELECT
    DATE_TRUNC('month', ordered_at)  AS month,
    product_category,
    customer_segment,
    COUNT(*)                         AS total_orders,
    SUM(total_amount)                AS total_revenue,
    AVG(total_amount)                AS avg_order_value,
    COUNT(DISTINCT customer_name)    AS unique_customers
FROM enriched
WHERE status = 'completed'
GROUP BY 1, 2, 3
```

```yaml
# dbt/models/marts/mart_sales_summary.yml
version: 2
models:
  - name: mart_sales_summary
    description: "Monthly sales summary by category and segment"
    columns:
      - name: month
        tests:
          - not_null
      - name: total_orders
        tests:
          - not_null
      - name: total_revenue
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
```

### 3.3 — Go: File Compaction Job

**Objetivo:** Job Go que compacta small files no S3 em arquivos maiores.

```go
// go/cmd/compactor/main.go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

const (
    targetSizeMB = 128 // Ideal Parquet file size
)

func main() {
    cfg, _ := config.LoadDefaultConfig(context.TODO(),
        config.WithRegion("us-east-1"),
    )

    client := s3.NewFromConfig(cfg, func(o *s3.Options) {
        o.BaseEndpoint = aws.String("http://localhost:4566")
        o.UsePathStyle = true
    })

    // List small files in curated zone
    paginator := s3.NewListObjectsV2Paginator(client, &s3.ListObjectsV2Input{
        Bucket: aws.String("data-lake-curated"),
        Prefix: aws.String("orders/"),
    })

    var smallFiles []string
    for paginator.HasMorePages() {
        page, err := paginator.NextPage(context.TODO())
        if err != nil {
            log.Fatal(err)
        }
        for _, obj := range page.Contents {
            sizeMB := *obj.Size / (1024 * 1024)
            if sizeMB < targetSizeMB {
                smallFiles = append(smallFiles, *obj.Key)
            }
        }
    }

    fmt.Printf("Found %d small files (<128MB) to compact\n", len(smallFiles))
    // Compaction logic: read all → merge → write single file
}
```

### 3.4 — Java: Spring Batch Pipeline

**Objetivo:** Job Spring Batch com steps para ETL completo.

```java
// java/src/.../batch/OrderEtlJobConfig.java
@Configuration
@EnableBatchProcessing
public class OrderEtlJobConfig {

    @Bean
    public Job orderEtlJob(JobRepository jobRepository,
                            Step extractStep,
                            Step transformStep,
                            Step loadStep,
                            Step qualityCheckStep) {
        return new JobBuilder("orderEtlJob", jobRepository)
            .start(extractStep)
            .next(transformStep)
            .next(qualityCheckStep)
            .next(loadStep)
            .listener(new JobCompletionListener())
            .build();
    }

    @Bean
    public Step extractStep(JobRepository repo,
                             PlatformTransactionManager tx) {
        return new StepBuilder("extract", repo)
            .<RawOrder, RawOrder>chunk(1000, tx)
            .reader(s3ParquetReader())
            .writer(stagingWriter())
            .faultTolerant()
            .retryLimit(3)
            .retry(S3Exception.class)
            .build();
    }

    @Bean
    public Step transformStep(JobRepository repo,
                               PlatformTransactionManager tx) {
        return new StepBuilder("transform", repo)
            .<RawOrder, EnrichedOrder>chunk(500, tx)
            .reader(stagingReader())
            .processor(enrichmentProcessor())
            .writer(curatedWriter())
            .build();
    }

    @Bean
    public Step qualityCheckStep(JobRepository repo,
                                  PlatformTransactionManager tx) {
        return new StepBuilder("qualityCheck", repo)
            .tasklet(qualityCheckTasklet(), tx)
            .build();
    }
}
```

---

## Critérios de Aceite

- [ ] PySpark pipeline: S3 raw → transform → S3 curated (particionado)
- [ ] dbt project: staging → intermediate → marts (3 camadas)
- [ ] dbt tests passando (not_null, accepted_range)
- [ ] `dbt docs generate` produz documentação navegável
- [ ] Go compactor identifica small files no S3
- [ ] Spring Batch job com extract → transform → quality → load
- [ ] Pipeline processa ≥ 100K rows sem erro
- [ ] ADR-006, ADR-007, ADR-008 completos
- [ ] 2 DrawIOs (batch pipeline + dbt DAG)

---

## Definição de Pronto (DoD)

- [ ] 3 ADRs (engine + ETL/ELT + framework)
- [ ] 2 DrawIOs (batch pipeline + dbt DAG)
- [ ] PySpark ETL funcional
- [ ] dbt project com tests e docs
- [ ] Go compactor funcional
- [ ] Spring Batch job funcional
- [ ] Commit: `feat(data-engineering-03): batch processing`
