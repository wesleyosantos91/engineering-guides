# Level 11 — Modern Data Stack

> **Objetivo:** Construir uma plataforma analytics moderna centrada em dbt,
> com Medallion Architecture (Bronze → Silver → Gold), observabilidade de
> pipelines e serving layer para consumo por BI/API.

**Referência:** [06 — Architecture Patterns](../../.docs/data-engineering/06-architecture-patterns.md)

**Pré-requisito:** Level 10 concluído.

---

## Contexto

O **Modern Data Stack (MDS)** é a convergência de ferramentas especializadas
que, juntas, formam uma plataforma de dados completa: ingestão → storage →
transformação → quality → serving → observabilidade. Neste nível, integramos
dbt como engine de transformação central com a **Medallion Architecture**
(Bronze/Silver/Gold), adicionamos observabilidade com métricas de pipeline e
construímos uma camada de serving para queries analíticas.

---

## Parte 1 — ADRs (2 obrigatórios)

### ADR-027: Transformation Layer Strategy

**Arquivo:** `docs/adrs/ADR-027-transformation-layer.md`

**Context:** Precisamos definir a estratégia de transformação de dados para
a plataforma analytics. A abordagem deve suportar versionamento, testes,
documentação e linhagem.

**Options:**
1. **dbt Core + PostgreSQL** — SQL-first, modular, testes nativos, linhagem
2. **dbt + Trino** — SQL sobre data lake (Iceberg), distributed
3. **Spark SQL** — transformações programáticas, mais flexível
4. **Custom Python ETL** — scripts ad-hoc, difícil manutenção
5. **Stored Procedures** — transformações no banco, vendor lock-in

**Decision Drivers:**
- Testabilidade (unit tests, data tests)
- Documentação automática (catalog)
- Linhagem (lineage graph)
- Modularidade (staging → intermediate → marts)
- Developer experience

---

### ADR-028: Medallion Architecture Design

**Arquivo:** `docs/adrs/ADR-028-medallion-architecture.md`

**Context:** Precisamos de uma arquitetura em camadas para organizar dados
desde a ingestão (raw) até o consumo (analytics-ready).

**Options:**
1. **Bronze → Silver → Gold** (Medallion clássico)
2. **Raw → Curated → Refined** (3-zone data lake)
3. **Landing → Staging → DWH → Marts** (warehouse tradicional)
4. **Raw → Enriched → Aggregated** (streaming-oriented)

**Decision Drivers:**
- Clareza de responsabilidades por camada
- Reprocessamento e idempotência
- Qualidade incremental (cada camada melhora)
- Compatibilidade com batch e streaming

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

### DrawIO-021: Modern Data Stack Architecture

**Arquivo:** `docs/diagrams/11-modern-data-stack.drawio`

```
┌─────────────────────────────────────────────────────────────┐
│                   MODERN DATA STACK                         │
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│  │  Sources  │   │  CDC     │   │ APIs     │               │
│  │  (OLTP)  │   │(Debezium)│   │ (REST)   │               │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘               │
│       │              │              │                       │
│       └──────────────┼──────────────┘                       │
│                      ▼                                      │
│  ┌─────────────────────────────────┐                       │
│  │         BRONZE (Raw)            │  S3 / Iceberg          │
│  │  Raw events, CDC logs, API     │                         │
│  │  responses. No transformation. │                         │
│  └──────────────┬──────────────────┘                       │
│                 ▼                                           │
│  ┌─────────────────────────────────┐                       │
│  │  ┌───────────────────────────┐  │                       │
│  │  │      dbt CORE             │  │                       │
│  │  │  staging → intermediate   │  │                       │
│  │  │  → marts                  │  │                       │
│  │  │  tests │ docs │ lineage   │  │                       │
│  │  └───────────────────────────┘  │                       │
│  └──────────────┬──────────────────┘                       │
│                 ▼                                           │
│  ┌─────────────────────────────────┐                       │
│  │         SILVER (Cleaned)        │  PostgreSQL / Iceberg  │
│  │  Deduplicated, typed, validated │                        │
│  │  Business rules applied.        │                        │
│  └──────────────┬──────────────────┘                       │
│                 ▼                                           │
│  ┌─────────────────────────────────┐                       │
│  │         GOLD (Aggregated)       │  PostgreSQL / View     │
│  │  KPIs, metrics, aggregations   │                         │
│  │  Ready for BI / API serving    │                         │
│  └──────────────┬──────────────────┘                       │
│       ┌─────────┴─────────┐                                │
│       ▼                   ▼                                 │
│  ┌──────────┐      ┌──────────┐                            │
│  │ API/REST │      │ DuckDB   │                            │
│  │ Serving  │      │ Analytics│                            │
│  └──────────┘      └──────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

### DrawIO-022: dbt Project Structure

**Arquivo:** `docs/diagrams/11-dbt-project-structure.drawio`

```
┌──────────────────────────────────────────────────────────┐
│                 dbt PROJECT STRUCTURE                     │
│                                                          │
│  models/                                                 │
│  ├── staging/         ← 1:1 com source, clean/rename    │
│  │   ├── stg_orders.sql                                 │
│  │   ├── stg_customers.sql                              │
│  │   └── stg_products.sql                               │
│  ├── intermediate/    ← joins, business logic           │
│  │   ├── int_orders_enriched.sql                        │
│  │   └── int_customer_orders.sql                        │
│  └── marts/           ← final aggregations (Gold)       │
│      ├── finance/                                        │
│      │   └── fct_revenue.sql                            │
│      └── marketing/                                      │
│          └── dim_customers.sql                           │
│                                                          │
│  tests/                                                  │
│  ├── generic/         ← reusable tests                  │
│  │   └── test_positive_amount.sql                       │
│  └── singular/        ← specific tests                  │
│      └── assert_revenue_not_negative.sql                │
│                                                          │
│  macros/                                                 │
│  └── generate_schema_name.sql                           │
│                                                          │
│  ┌─────────────────────────────────────────┐             │
│  │  LINEAGE GRAPH (dbt docs generate)      │             │
│  │                                          │             │
│  │  source → stg_orders ─┐                 │             │
│  │  source → stg_customers ┼→ int_enriched │             │
│  │  source → stg_products ─┘       │       │             │
│  │                               ▼       │             │
│  │                          fct_revenue   │             │
│  │                          dim_customers │             │
│  └─────────────────────────────────────────┘             │
└──────────────────────────────────────────────────────────┘
```

---

## Parte 3 — Implementação

### 3.1 — dbt Project: Medallion Architecture

```yaml
# dbt/dbt_project.yml
name: 'medallion_platform'
version: '1.0.0'
config-version: 2

profile: 'medallion'

model-paths: ["models"]
test-paths: ["tests"]
macro-paths: ["macros"]

models:
  medallion_platform:
    staging:
      +materialized: view
      +schema: bronze
      +tags: ['staging', 'bronze']
    intermediate:
      +materialized: table
      +schema: silver
      +tags: ['intermediate', 'silver']
    marts:
      +materialized: table
      +schema: gold
      +tags: ['marts', 'gold']
```

```yaml
# dbt/models/staging/sources.yml
version: 2

sources:
  - name: raw
    schema: public
    description: "Raw data ingested from source systems"
    tables:
      - name: orders
        description: "Raw e-commerce orders"
        columns:
          - name: id
            tests: [not_null, unique]
          - name: customer_id
            tests: [not_null]
          - name: amount
            tests: [not_null]
        freshness:
          warn_after: {count: 12, period: hour}
          error_after: {count: 24, period: hour}
        loaded_at_field: created_at

      - name: customers
        description: "Customer master data"
        columns:
          - name: id
            tests: [not_null, unique]

      - name: products
        description: "Product catalog"
        columns:
          - name: id
            tests: [not_null, unique]
```

### 3.2 — Staging Models (Bronze → Silver)

```sql
-- dbt/models/staging/stg_orders.sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'orders') }}
),
renamed AS (
    SELECT
        id                                AS order_id,
        customer_id,
        product_id,
        CAST(amount AS DECIMAL(12,2))     AS amount,
        LOWER(TRIM(status))               AS status,
        CAST(created_at AS TIMESTAMP)     AS ordered_at,
        CAST(updated_at AS TIMESTAMP)     AS updated_at
    FROM source
    WHERE id IS NOT NULL
      AND amount > 0
)
SELECT * FROM renamed
```

```sql
-- dbt/models/staging/stg_customers.sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'customers') }}
),
renamed AS (
    SELECT
        id              AS customer_id,
        TRIM(name)      AS customer_name,
        LOWER(email)    AS email,
        region,
        CAST(created_at AS TIMESTAMP) AS registered_at
    FROM source
    WHERE id IS NOT NULL
)
SELECT * FROM renamed
```

```sql
-- dbt/models/staging/stg_products.sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'products') }}
),
renamed AS (
    SELECT
        id                              AS product_id,
        TRIM(name)                      AS product_name,
        category,
        CAST(price AS DECIMAL(10,2))    AS unit_price
    FROM source
)
SELECT * FROM renamed
```

### 3.3 — Intermediate Models (Silver)

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
),
enriched AS (
    SELECT
        o.order_id,
        o.customer_id,
        c.customer_name,
        c.email,
        c.region,
        o.product_id,
        p.product_name,
        p.category,
        p.unit_price,
        o.amount,
        o.status,
        o.ordered_at,
        DATE_TRUNC('day', o.ordered_at) AS order_date,
        DATE_TRUNC('month', o.ordered_at) AS order_month
    FROM orders o
    LEFT JOIN customers c ON o.customer_id = c.customer_id
    LEFT JOIN products p ON o.product_id = p.product_id
)
SELECT * FROM enriched
```

```sql
-- dbt/models/intermediate/int_customer_orders.sql
WITH enriched AS (
    SELECT * FROM {{ ref('int_orders_enriched') }}
),
customer_agg AS (
    SELECT
        customer_id,
        customer_name,
        region,
        COUNT(DISTINCT order_id)     AS total_orders,
        SUM(amount)                  AS total_spent,
        AVG(amount)                  AS avg_order_value,
        MIN(ordered_at)              AS first_order_at,
        MAX(ordered_at)              AS last_order_at
    FROM enriched
    WHERE status != 'cancelled'
    GROUP BY customer_id, customer_name, region
)
SELECT * FROM customer_agg
```

### 3.4 — Mart Models (Gold)

```sql
-- dbt/models/marts/finance/fct_revenue.sql
{{ config(
    materialized='table',
    tags=['finance', 'gold']
) }}

WITH enriched AS (
    SELECT * FROM {{ ref('int_orders_enriched') }}
),
revenue AS (
    SELECT
        order_date,
        order_month,
        category,
        region,
        COUNT(DISTINCT order_id)     AS num_orders,
        COUNT(DISTINCT customer_id)  AS unique_customers,
        SUM(amount)                  AS gross_revenue,
        AVG(amount)                  AS avg_order_value,
        SUM(CASE WHEN status = 'completed' THEN amount ELSE 0 END) AS net_revenue,
        SUM(CASE WHEN status = 'refunded' THEN amount ELSE 0 END)  AS refunds
    FROM enriched
    GROUP BY order_date, order_month, category, region
)
SELECT
    *,
    net_revenue - refunds AS adjusted_revenue,
    ROUND(refunds / NULLIF(gross_revenue, 0) * 100, 2) AS refund_rate
FROM revenue
```

```sql
-- dbt/models/marts/marketing/dim_customers.sql
{{ config(
    materialized='table',
    tags=['marketing', 'gold']
) }}

WITH customer_orders AS (
    SELECT * FROM {{ ref('int_customer_orders') }}
),
segmented AS (
    SELECT
        *,
        CASE
            WHEN total_orders >= 10 AND total_spent >= 1000 THEN 'VIP'
            WHEN total_orders >= 5 THEN 'Regular'
            WHEN total_orders >= 1 THEN 'New'
            ELSE 'Inactive'
        END AS customer_segment,
        CURRENT_DATE - last_order_at::date AS days_since_last_order
    FROM customer_orders
)
SELECT * FROM segmented
```

### 3.5 — dbt Tests & Documentation

```yaml
# dbt/models/marts/finance/schema.yml
version: 2

models:
  - name: fct_revenue
    description: "Daily revenue fact table aggregated by category and region"
    columns:
      - name: order_date
        description: "Date of the order"
        tests: [not_null]
      - name: gross_revenue
        description: "Total gross revenue for the day/category/region"
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
      - name: net_revenue
        tests:
          - not_null
      - name: refund_rate
        tests:
          - dbt_utils.accepted_range:
              min_value: 0
              max_value: 100

  - name: dim_customers
    description: "Customer dimension with segmentation"
    columns:
      - name: customer_id
        tests: [not_null, unique]
      - name: customer_segment
        tests:
          - accepted_values:
              values: ['VIP', 'Regular', 'New', 'Inactive']
```

```sql
-- dbt/tests/generic/test_positive_amount.sql
{% test positive_amount(model, column_name) %}
SELECT *
FROM {{ model }}
WHERE {{ column_name }} < 0
{% endtest %}
```

### 3.6 — Pipeline Observability

```python
# python/observability/pipeline_metrics.py
import time
import json
import boto3
from datetime import datetime, timezone
from functools import wraps


class PipelineMetrics:
    """Coleta e publica métricas de pipeline."""

    def __init__(self, pipeline_name: str):
        self.pipeline_name = pipeline_name
        self.dynamodb = boto3.resource(
            "dynamodb",
            endpoint_url="http://localhost:4566",
            region_name="us-east-1"
        )
        self.table = self.dynamodb.Table("pipeline_metrics")

    def track_step(self, step_name: str):
        """Decorator para rastrear métricas de cada step."""
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                start = time.time()
                status = "success"
                error_msg = None
                rows_processed = 0

                try:
                    result = func(*args, **kwargs)
                    if isinstance(result, int):
                        rows_processed = result
                    return result
                except Exception as e:
                    status = "failure"
                    error_msg = str(e)
                    raise
                finally:
                    elapsed = time.time() - start
                    self._publish_metric(
                        step_name=step_name,
                        duration_seconds=round(elapsed, 3),
                        status=status,
                        rows_processed=rows_processed,
                        error=error_msg
                    )
            return wrapper
        return decorator

    def _publish_metric(self, step_name, duration_seconds,
                        status, rows_processed, error):
        ts = datetime.now(timezone.utc).isoformat()
        metric = {
            "pipeline_name": self.pipeline_name,
            "step_name": step_name,
            "timestamp": ts,
            "duration_seconds": str(duration_seconds),
            "status": status,
            "rows_processed": rows_processed,
        }
        if error:
            metric["error"] = error

        self.table.put_item(Item=metric)
        print(f"[METRIC] {self.pipeline_name}/{step_name}: "
              f"{status} in {duration_seconds}s ({rows_processed} rows)")


# Usage example:
metrics = PipelineMetrics("medallion_daily")

@metrics.track_step("extract_orders")
def extract_orders():
    time.sleep(0.5)  # Simulate extraction
    return 15000  # rows processed

@metrics.track_step("transform_silver")
def transform_silver():
    time.sleep(1.2)  # Simulate dbt run
    return 14500

@metrics.track_step("aggregate_gold")
def aggregate_gold():
    time.sleep(0.8)
    return 365  # aggregated rows
```

### 3.7 — Go: Analytics API (Serving Layer)

```go
// go/cmd/analytics-api/main.go
package main

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "time"

    _ "github.com/lib/pq"
)

type RevenueMetric struct {
    Date            string  `json:"date"`
    Category        string  `json:"category"`
    Region          string  `json:"region"`
    GrossRevenue    float64 `json:"gross_revenue"`
    NetRevenue      float64 `json:"net_revenue"`
    NumOrders       int     `json:"num_orders"`
    UniqueCustomers int     `json:"unique_customers"`
    RefundRate      float64 `json:"refund_rate"`
}

type CustomerSegment struct {
    CustomerID   int     `json:"customer_id"`
    Name         string  `json:"name"`
    Segment      string  `json:"segment"`
    TotalOrders  int     `json:"total_orders"`
    TotalSpent   float64 `json:"total_spent"`
    DaysSinceLast int    `json:"days_since_last_order"`
}

var db *sql.DB

func main() {
    var err error
    db, err = sql.Open("postgres",
        "host=localhost port=5432 user=admin password=admin dbname=warehouse sslmode=disable")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    http.HandleFunc("/api/v1/revenue", revenueHandler)
    http.HandleFunc("/api/v1/customers", customersHandler)
    http.HandleFunc("/health", healthHandler)

    log.Println("Analytics API listening on :8090")
    log.Fatal(http.ListenAndServe(":8090", nil))
}

func revenueHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    category := r.URL.Query().Get("category")
    region := r.URL.Query().Get("region")

    query := `SELECT order_date, category, region, gross_revenue,
                     net_revenue, num_orders, unique_customers, refund_rate
              FROM gold.fct_revenue WHERE 1=1`
    args := []interface{}{}
    idx := 1

    if category != "" {
        query += fmt.Sprintf(" AND category = $%d", idx)
        args = append(args, category)
        idx++
    }
    if region != "" {
        query += fmt.Sprintf(" AND region = $%d", idx)
        args = append(args, region)
    }
    query += " ORDER BY order_date DESC LIMIT 100"

    rows, err := db.QueryContext(ctx, query, args...)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    defer rows.Close()

    var metrics []RevenueMetric
    for rows.Next() {
        var m RevenueMetric
        rows.Scan(&m.Date, &m.Category, &m.Region, &m.GrossRevenue,
            &m.NetRevenue, &m.NumOrders, &m.UniqueCustomers, &m.RefundRate)
        metrics = append(metrics, m)
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(metrics)
}

func customersHandler(w http.ResponseWriter, r *http.Request) {
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

    rows, err := db.QueryContext(ctx, query, args...)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    defer rows.Close()

    var customers []CustomerSegment
    for rows.Next() {
        var c CustomerSegment
        rows.Scan(&c.CustomerID, &c.Name, &c.Segment,
            &c.TotalOrders, &c.TotalSpent, &c.DaysSinceLast)
        customers = append(customers, c)
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(customers)
}

func healthHandler(w http.ResponseWriter, _ *http.Request) {
    w.Write([]byte(`{"status":"ok"}`))
}
```

### 3.8 — Java: dbt Runner & Metrics Collector

```java
// java/src/.../modernstack/DbtRunnerService.java
@Service
public class DbtRunnerService {

    private static final Logger log = LoggerFactory.getLogger(DbtRunnerService.class);

    public record DbtResult(String command, int exitCode,
                            Duration duration, List<String> output) {}

    public DbtResult runDbt(String command, String projectDir) {
        var start = Instant.now();
        try {
            var process = new ProcessBuilder("dbt", command, "--project-dir", projectDir)
                .redirectErrorStream(true)
                .start();

            var output = new BufferedReader(
                new InputStreamReader(process.getInputStream()))
                .lines().toList();

            int exitCode = process.waitFor();
            var duration = Duration.between(start, Instant.now());

            output.forEach(line -> log.info("[dbt] {}", line));

            return new DbtResult(command, exitCode, duration, output);
        } catch (Exception e) {
            throw new RuntimeException("dbt execution failed", e);
        }
    }

    public DbtResult build(String projectDir) {
        return runDbt("build", projectDir);
    }

    public DbtResult test(String projectDir) {
        return runDbt("test", projectDir);
    }

    public DbtResult generateDocs(String projectDir) {
        return runDbt("docs generate", projectDir);
    }
}
```

```java
// java/src/.../modernstack/MetricsCollectorService.java
@Service
public class MetricsCollectorService {

    private final DynamoDbClient dynamoDb;

    public MetricsCollectorService() {
        this.dynamoDb = DynamoDbClient.builder()
            .endpointOverride(URI.create("http://localhost:4566"))
            .region(Region.US_EAST_1)
            .build();
    }

    public record PipelineMetric(String pipeline, String step,
                                 String status, double durationSec,
                                 long rowsProcessed) {}

    public void publish(PipelineMetric metric) {
        var item = Map.of(
            "pipeline_name", AttributeValue.fromS(metric.pipeline()),
            "step_name", AttributeValue.fromS(metric.step()),
            "timestamp", AttributeValue.fromS(Instant.now().toString()),
            "status", AttributeValue.fromS(metric.status()),
            "duration_seconds", AttributeValue.fromN(String.valueOf(metric.durationSec())),
            "rows_processed", AttributeValue.fromN(String.valueOf(metric.rowsProcessed()))
        );

        dynamoDb.putItem(PutItemRequest.builder()
            .tableName("pipeline_metrics")
            .item(item)
            .build());
    }

    public List<Map<String, AttributeValue>> getRecentMetrics(String pipeline, int limit) {
        return dynamoDb.query(QueryRequest.builder()
            .tableName("pipeline_metrics")
            .keyConditionExpression("pipeline_name = :p")
            .expressionAttributeValues(Map.of(
                ":p", AttributeValue.fromS(pipeline)))
            .scanIndexForward(false)
            .limit(limit)
            .build()).items();
    }
}
```

---

## Critérios de Aceite

- [ ] dbt project funcional com staging → intermediate → marts
- [ ] Medallion architecture: Bronze (raw) → Silver (cleaned) → Gold (aggregated)
- [ ] ≥ 3 staging models, ≥ 2 intermediate models, ≥ 2 mart models
- [ ] dbt tests: ≥ 10 testes (not_null, unique, accepted_values, custom)
- [ ] dbt docs generate: catalog + lineage graph
- [ ] Source freshness configurada e testada
- [ ] Pipeline observability: métricas de execução salvas (DynamoDB)
- [ ] Go analytics API: /api/v1/revenue e /api/v1/customers
- [ ] Java dbt runner + metrics collector funcionais
- [ ] ADR-027, ADR-028 completos
- [ ] 2 DrawIOs (modern data stack + dbt project structure)

---

## Definição de Pronto (DoD)

- [ ] 2 ADRs (transformation layer + medallion architecture)
- [ ] 2 DrawIOs (modern data stack architecture + dbt project structure)
- [ ] dbt project: staging/intermediate/marts com testes
- [ ] Medallion Architecture (Bronze → Silver → Gold) funcional
- [ ] Pipeline observability com métricas
- [ ] Go serving layer (analytics API)
- [ ] Java dbt runner + metrics collector
- [ ] Commit: `feat(data-engineering-11): modern data stack`
