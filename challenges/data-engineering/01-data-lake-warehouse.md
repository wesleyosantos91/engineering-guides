# Level 01 — Data Lake & Data Warehouse

> **Objetivo:** Projetar e implementar um Data Lake multi-zona no S3 (LocalStack)
> e um Data Warehouse analítico no PostgreSQL, entendendo trade-offs
> entre as duas abordagens.

**Referência:** [01 — Architecture Foundations](../../.docs/data-engineering/01-data-architecture-foundations.md)

**Pré-requisito:** Level 00 concluído.

---

## Contexto

Toda empresa precisa decidir **onde e como** armazenar seus dados. As duas
abordagens clássicas são **Data Lake** (schema-on-read, armazenamento de objetos)
e **Data Warehouse** (schema-on-write, banco analítico). Neste nível, você
implementa ambas e compara os trade-offs.

---

## Parte 1 — ADRs (2 obrigatórios)

### ADR-002: Data Lake Zone Strategy

**Arquivo:** `docs/adrs/ADR-002-data-lake-zones.md`

**Context:** Precisamos organizar dados brutos e processados no S3 com governança
e rastreabilidade.

**Options:**
1. **3 zonas** — Raw → Curated → Refined (medalha bronze/silver/gold)
2. **4 zonas** — Landing → Raw → Curated → Refined
3. **2 zonas** — Raw → Processed (simplificado)
4. **Domain-based** — Uma zona por domínio de negócio (Data Mesh style)

**Decision Drivers:**
- Rastreabilidade de transformações
- Governança e acesso granular
- Complexidade operacional
- Evolução futura para Lakehouse

**Critérios de aceite:**
- [ ] ADR fundamenta escolha com métricas de complexidade
- [ ] Convenção de naming S3 documentada

---

### ADR-003: Analytical Store — Data Lake vs Data Warehouse

**Arquivo:** `docs/adrs/ADR-003-lake-vs-warehouse.md`

**Context:** Precisamos servir queries analíticas para dashboards e relatórios.

**Options:**
1. **Data Lake + Query Engine** — S3/Parquet + Trino/Athena (schema-on-read)
2. **Data Warehouse** — PostgreSQL/Redshift (schema-on-write)
3. **Lakehouse** — S3 + Iceberg/Delta + DuckDB (schema-on-read com ACID)
4. **Hybrid** — Lake para exploração + Warehouse para serving

**Decision Drivers:**
- Latência de query (<2s para dashboards)
- Custo de storage vs compute
- Flexibilidade de schema
- Complexidade de manutenção

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

### DrawIO-002: Data Lake Architecture

**Arquivo:** `docs/diagrams/01-data-lake-architecture.drawio`

```
                        ┌─────────────────────────────────┐
                        │         DATA SOURCES            │
                        │  API │ Database │ Files │ Events│
                        └──────────┬──────────────────────┘
                                   │
                     ┌─────────────▼──────────────┐
                     │     INGESTION LAYER        │
                     │  Python scripts │ Lambda   │
                     └─────────────┬──────────────┘
                                   │
     ┌─────────────────────────────▼─────────────────────────────┐
     │                    S3 DATA LAKE                           │
     │                                                           │
     │  ┌──────────┐    ┌──────────┐    ┌──────────┐            │
     │  │   RAW    │───▶│ CURATED  │───▶│ REFINED  │            │
     │  │  (JSON)  │    │(Parquet) │    │(Parquet) │            │
     │  │          │    │ cleaned  │    │ aggregated│           │
     │  └──────────┘    └──────────┘    └──────────┘            │
     └───────────────────────────────────────────────────────────┘
                                   │
                     ┌─────────────▼──────────────┐
                     │     QUERY ENGINES          │
                     │   Trino │ DuckDB │ Athena  │
                     └────────────────────────────┘
```

### DrawIO-003: Data Warehouse Schema

**Arquivo:** `docs/diagrams/01-warehouse-schema.drawio`

Representar o star schema dimensional:
- Fato: `fact_orders` / `fact_events`
- Dimensões: `dim_customer`, `dim_product`, `dim_time`, `dim_location`
- Relações FK entre fatos e dimensões

---

## Parte 3 — Implementação

### 3.1 — Data Lake Multi-Zona (S3/LocalStack)

**Objetivo:** Pipeline Python que ingere dados em 3 zonas progressivas.

```python
# python/etl/data_lake.py
import json
from datetime import datetime
from pathlib import Path

import boto3
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
from io import BytesIO

S3 = boto3.client("s3", endpoint_url="http://localhost:4566",
                   aws_access_key_id="test", aws_secret_access_key="test")

def ingest_to_raw(source_path: str, domain: str):
    """Zone 1 — Raw: dados brutos exatamente como vieram."""
    ts = datetime.utcnow().strftime("%Y/%m/%d/%H%M%S")
    key = f"{domain}/{ts}/data.json"
    
    with open(source_path) as f:
        data = f.read()
    
    S3.put_object(Bucket="data-lake-raw", Key=key, Body=data)
    print(f"✓ Raw: s3://data-lake-raw/{key}")
    return key

def transform_to_curated(raw_key: str, domain: str):
    """Zone 2 — Curated: limpo, tipado, formato Parquet."""
    obj = S3.get_object(Bucket="data-lake-raw", Key=raw_key)
    records = json.loads(obj["Body"].read())
    
    df = pd.DataFrame(records)
    # Data cleansing
    df = df.dropna(subset=["id"])
    df = df.drop_duplicates(subset=["id"])
    df.columns = [c.lower().replace(" ", "_") for c in df.columns]
    
    # Convert to Parquet
    table = pa.Table.from_pandas(df)
    buffer = BytesIO()
    pq.write_table(table, buffer, compression="snappy")
    buffer.seek(0)
    
    curated_key = f"{domain}/curated.parquet"
    S3.put_object(Bucket="data-lake-curated", Key=curated_key, Body=buffer.getvalue())
    print(f"✓ Curated: s3://data-lake-curated/{curated_key}")
    return curated_key

def aggregate_to_refined(curated_key: str, domain: str):
    """Zone 3 — Refined: agregado, pronto para consumo."""
    obj = S3.get_object(Bucket="data-lake-curated", Key=curated_key)
    buffer = BytesIO(obj["Body"].read())
    table = pq.read_table(buffer)
    df = table.to_pandas()
    
    # Business aggregation
    agg = df.groupby(df.columns[1]).agg({df.columns[0]: "count"}).reset_index()
    agg.columns = ["category", "total_count"]
    
    table = pa.Table.from_pandas(agg)
    out = BytesIO()
    pq.write_table(table, out, compression="snappy")
    out.seek(0)
    
    refined_key = f"{domain}/summary.parquet"
    S3.put_object(Bucket="data-lake-refined", Key=refined_key, Body=out.getvalue())
    print(f"✓ Refined: s3://data-lake-refined/{refined_key}")
```

### 3.2 — Data Warehouse (PostgreSQL)

**Objetivo:** Dimensional modeling com star schema.

```sql
-- sql/warehouse/001_create_dimensions.sql

CREATE SCHEMA IF NOT EXISTS warehouse;

-- Dim Time
CREATE TABLE warehouse.dim_time (
    time_id      SERIAL PRIMARY KEY,
    date_actual  DATE NOT NULL,
    year         INT NOT NULL,
    quarter      INT NOT NULL,
    month        INT NOT NULL,
    week         INT NOT NULL,
    day_of_week  INT NOT NULL,
    is_weekend   BOOLEAN NOT NULL
);

-- Dim Customer
CREATE TABLE warehouse.dim_customer (
    customer_id  SERIAL PRIMARY KEY,
    name         VARCHAR(200) NOT NULL,
    email        VARCHAR(200),
    segment      VARCHAR(50),
    region       VARCHAR(100),
    created_at   TIMESTAMP DEFAULT NOW()
);

-- Dim Product
CREATE TABLE warehouse.dim_product (
    product_id   SERIAL PRIMARY KEY,
    name         VARCHAR(200) NOT NULL,
    category     VARCHAR(100),
    subcategory  VARCHAR(100),
    unit_price   NUMERIC(12,2)
);

-- Fact Orders (star schema center)
CREATE TABLE warehouse.fact_orders (
    order_id     BIGSERIAL PRIMARY KEY,
    time_id      INT REFERENCES warehouse.dim_time(time_id),
    customer_id  INT REFERENCES warehouse.dim_customer(customer_id),
    product_id   INT REFERENCES warehouse.dim_product(product_id),
    quantity     INT NOT NULL,
    total_amount NUMERIC(14,2) NOT NULL,
    discount     NUMERIC(5,2) DEFAULT 0,
    status       VARCHAR(20) NOT NULL
);

-- Indexes for analytical queries
CREATE INDEX idx_fact_orders_time ON warehouse.fact_orders(time_id);
CREATE INDEX idx_fact_orders_customer ON warehouse.fact_orders(customer_id);
CREATE INDEX idx_fact_orders_product ON warehouse.fact_orders(product_id);
```

### 3.3 — Go: Query Engine Client (DuckDB over S3)

**Objetivo:** Ler Parquet do S3 usando DuckDB via Go.

```go
// go/cmd/query-lake/main.go
package main

import (
    "database/sql"
    "fmt"
    "log"

    _ "github.com/marcboeker/go-duckdb"
)

func main() {
    db, err := sql.Open("duckdb", "")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Configure S3 endpoint (LocalStack)
    db.Exec("INSTALL httpfs; LOAD httpfs;")
    db.Exec("SET s3_endpoint='localhost:4566';")
    db.Exec("SET s3_access_key_id='test';")
    db.Exec("SET s3_secret_access_key='test';")
    db.Exec("SET s3_use_ssl=false;")
    db.Exec("SET s3_url_style='path';")

    // Query Parquet files directly from S3
    rows, err := db.Query(`
        SELECT * FROM read_parquet('s3://data-lake-curated/orders/curated.parquet')
        LIMIT 10
    `)
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()

    cols, _ := rows.Columns()
    fmt.Printf("Columns: %v\n", cols)
}
```

### 3.4 — Java: Warehouse Loader (Spring Batch)

**Objetivo:** Carregar dados do S3 (Parquet) para o warehouse PostgreSQL.

```java
// java/src/.../batch/WarehouseLoaderConfig.java
@Configuration
@EnableBatchProcessing
public class WarehouseLoaderConfig {

    @Bean
    public Job warehouseLoadJob(JobRepository jobRepository,
                                 Step loadDimensionsStep,
                                 Step loadFactsStep) {
        return new JobBuilder("warehouseLoadJob", jobRepository)
            .start(loadDimensionsStep)
            .next(loadFactsStep)
            .build();
    }

    @Bean
    public Step loadDimensionsStep(JobRepository jobRepository,
                                    PlatformTransactionManager txManager) {
        return new StepBuilder("loadDimensions", jobRepository)
            .<Map<String, Object>, Customer>chunk(100, txManager)
            .reader(parquetReader())
            .processor(customerProcessor())
            .writer(jdbcWriter())
            .build();
    }
}
```

---

## Critérios de Aceite

- [ ] Data Lake com 3 zonas funcionais no S3 (raw → curated → refined)
- [ ] Naming convention: `s3://{bucket}/{domain}/{partition}/file.{format}`
- [ ] Warehouse: star schema com ≥ 3 dimensões + 1 fato
- [ ] Python: pipeline completo raw → curated → refined
- [ ] Go: DuckDB query sobre Parquet no S3
- [ ] Java: Spring Batch loader S3 → PostgreSQL
- [ ] Queries analíticas funcionando no warehouse
- [ ] ADR-002 e ADR-003 completos
- [ ] 2 DrawIOs (lake architecture + warehouse schema)

---

## Definição de Pronto (DoD)

- [ ] 2 ADRs (zones strategy + lake vs warehouse)
- [ ] 2 DrawIOs (lake architecture + warehouse schema)
- [ ] Data Lake 3 zonas funcional
- [ ] Warehouse star schema funcional
- [ ] Pipeline Python end-to-end
- [ ] Query Go/DuckDB funcional
- [ ] Loader Java/Spring Batch funcional
- [ ] Commit: `feat(data-engineering-01): data lake & warehouse`
