# Level 07 — Data Migration & Integration

> **Objetivo:** Implementar migração de dados entre sistemas, schema evolution,
> data federation com Trino e integração multi-source com qualidade.

**Referência:** [04 — Data Integration](../../.docs/data-engineering/04-data-integration.md)

**Pré-requisito:** Level 06 concluído.

---

## Contexto

Sistemas reais evoluem: schemas mudam, bancos são migrados, dados precisam ser
federados entre múltiplas fontes. Neste nível, implementamos **schema evolution**
(backward/forward compatibility), **data federation** com Trino (query across
multiple data stores) e migração de dados com versionamento.

---

## Parte 1 — ADRs (2 obrigatórios)

### ADR-017: Schema Evolution Strategy

**Arquivo:** `docs/adrs/ADR-017-schema-evolution.md`

**Context:** Schemas de dados evoluem (novos campos, tipos alterados, campos
removidos) sem poder parar consumers existentes.

**Options:**
1. **Avro + Schema Registry** — compatibility checks (backward, forward, full)
2. **Parquet schema merging** — Spark/PyArrow merge schemas automaticamente
3. **Protocol Buffers** — strong typing, field numbers, backward compatible
4. **JSON Schema** — flexible, human readable, validação em runtime

**Decision Drivers:**
- Backward compatibility garantida
- Tamanho do payload (serialization)
- Tooling (Kafka, Spark, Flink support)
- Developer experience
- Breaking change detection

---

### ADR-018: Data Federation Approach

**Arquivo:** `docs/adrs/ADR-018-data-federation.md`

**Context:** Precisamos consultar dados em múltiplos stores (S3, PostgreSQL,
DynamoDB) com SQL unificado.

**Options:**
1. **Trino** — distributed SQL engine, catalogs para múltiplos data sources
2. **Apache Drill** — schema-free SQL on Hadoop, JSON, Parquet
3. **Presto** — fork do Trino (Facebook), similar capabilities
4. **DuckDB** — embedded analytics, S3 + Parquet nativo
5. **AWS Athena** — serverless Presto, integração S3 nativa

**Decision Drivers:**
- Número de data sources suportados
- Performance em cross-source joins
- Custo operacional
- Push-down predicates (query optimization)

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

### DrawIO-013: Data Federation Architecture

**Arquivo:** `docs/diagrams/07-data-federation.drawio`

```
                    ┌──────────────────┐
                    │   Application    │
                    │   (SQL clients)  │
                    └────────┬─────────┘
                             │ SQL
                    ┌────────▼─────────┐
                    │      Trino       │
                    │  (Coordinator)   │
                    └────────┬─────────┘
                             │
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                  ▼
  ┌────────────────┐ ┌──────────────┐ ┌────────────────┐
  │ Hive Catalog   │ │   JDBC       │ │  DynamoDB      │
  │ (S3/Parquet)   │ │ (PostgreSQL) │ │  Catalog       │
  │                │ │              │ │                │
  │ data_lake.raw  │ │ warehouse.*  │ │ nosql.*        │
  │ data_lake.     │ │              │ │                │
  │   curated      │ │              │ │                │
  └────────────────┘ └──────────────┘ └────────────────┘
```

### DrawIO-014: Schema Evolution Flow

**Arquivo:** `docs/diagrams/07-schema-evolution.drawio`

```
Version 1          Version 2          Version 3
┌──────────┐      ┌──────────┐      ┌──────────┐
│ id: INT  │      │ id: INT  │      │ id: INT  │
│ name: STR│ ───▶ │ name: STR│ ───▶ │ name: STR│
│ email:STR│      │ email:STR│      │ email:STR│
│          │      │ phone:STR│ NEW  │ phone:STR│
│          │      │          │      │ age: INT │ NEW
│          │      │          │      │ -email   │ DEPRECATED
└──────────┘      └──────────┘      └──────────┘
     │                 │                  │
     ▼                 ▼                  ▼
Schema Registry: BACKWARD compatible
  ✓ v2 can read v1 (phone has default null)
  ✓ v3 can read v2 (age has default null)
  ✗ v3 cannot remove email if consumers need it
```

---

## Parte 3 — Implementação

### 3.1 — Schema Evolution com Avro (Python)

```python
# python/integration/schema_evolution.py
import json
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer, AvroDeserializer

SCHEMA_REGISTRY_URL = "http://localhost:8085"

# Schema v1
SCHEMA_V1 = {
    "type": "record",
    "name": "Order",
    "namespace": "com.dataeng",
    "fields": [
        {"name": "id", "type": "int"},
        {"name": "customer_id", "type": "int"},
        {"name": "amount", "type": "double"},
        {"name": "status", "type": "string"},
    ],
}

# Schema v2: added 'currency' with default (backward compatible)
SCHEMA_V2 = {
    "type": "record",
    "name": "Order",
    "namespace": "com.dataeng",
    "fields": [
        {"name": "id", "type": "int"},
        {"name": "customer_id", "type": "int"},
        {"name": "amount", "type": "double"},
        {"name": "status", "type": "string"},
        {"name": "currency", "type": "string", "default": "BRL"},
    ],
}

# Schema v3: added 'region', deprecated 'status' → 'order_status'
SCHEMA_V3 = {
    "type": "record",
    "name": "Order",
    "namespace": "com.dataeng",
    "fields": [
        {"name": "id", "type": "int"},
        {"name": "customer_id", "type": "int"},
        {"name": "amount", "type": "double"},
        {"name": "status", "type": "string"},
        {"name": "currency", "type": "string", "default": "BRL"},
        {"name": "region", "type": ["null", "string"], "default": None},
        {"name": "order_status", "type": ["null", "string"], "default": None},
    ],
}

def register_schema(subject: str, schema: dict, compatibility: str = "BACKWARD"):
    """Registra schema no Schema Registry com compatibility check."""
    client = SchemaRegistryClient({"url": SCHEMA_REGISTRY_URL})
    
    # Set compatibility level
    client.set_compatibility(subject_name=subject, level=compatibility)
    
    from confluent_kafka.schema_registry import Schema
    schema_obj = Schema(json.dumps(schema), "AVRO")
    
    schema_id = client.register_schema(subject, schema_obj)
    print(f"✓ Registered schema v{schema_id} for {subject} ({compatibility})")
    return schema_id

def check_compatibility(subject: str, schema: dict):
    """Verifica se novo schema é compatível com versões existentes."""
    client = SchemaRegistryClient({"url": SCHEMA_REGISTRY_URL})
    from confluent_kafka.schema_registry import Schema
    schema_obj = Schema(json.dumps(schema), "AVRO")
    
    is_compatible = client.test_compatibility(subject, schema_obj)
    print(f"{'✓' if is_compatible else '✗'} Schema compatibility: {is_compatible}")
    return is_compatible
```

### 3.2 — Parquet Schema Merge (Python)

```python
# python/integration/parquet_merge.py
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path

def merge_parquet_schemas(paths: list[str]) -> pa.Schema:
    """Merge schemas de múltiplos Parquet files."""
    schemas = []
    for path in paths:
        meta = pq.read_metadata(path)
        schemas.append(pq.read_schema(path))
    
    merged = pa.unify_schemas(schemas, promote_options="permissive")
    print(f"✓ Merged {len(schemas)} schemas → {len(merged)} fields")
    for field in merged:
        print(f"  {field.name}: {field.type} (nullable={field.nullable})")
    return merged

def read_with_evolved_schema(directory: str) -> pa.Table:
    """Lê Parquet files com schemas diferentes e merges automaticamente."""
    dataset = pq.ParquetDataset(directory, use_legacy_dataset=False)
    table = dataset.read()
    print(f"✓ Read {table.num_rows} rows with unified schema ({len(table.schema)} cols)")
    return table
```

### 3.3 — Trino Data Federation (Docker + SQL)

```yaml
# infra/trino/docker-compose.trino.yml
services:
  trino:
    image: trinodb/trino:435
    ports:
      - "8084:8080"
    volumes:
      - ./trino/etc:/etc/trino
      - ./trino/catalog:/etc/trino/catalog
```

```properties
# infra/trino/catalog/hive.properties
connector.name=hive
hive.metastore=file
hive.metastore.catalog.dir=s3a://data-lake-curated/
hive.s3.endpoint=http://localstack:4566
hive.s3.aws-access-key=test
hive.s3.aws-secret-key=test
hive.s3.path-style-access=true
```

```properties
# infra/trino/catalog/postgresql.properties
connector.name=postgresql
connection-url=jdbc:postgresql://postgres:5432/datawarehouse
connection-user=dataeng
connection-password=dataeng
```

```sql
-- Federated query: join S3 data lake with PostgreSQL warehouse
SELECT
    l.product_category,
    l.total_events,
    w.total_revenue,
    w.total_revenue / l.total_events AS revenue_per_event
FROM hive.data_lake.product_events l
JOIN postgresql.warehouse.mart_sales_summary w
    ON l.product_category = w.product_category
ORDER BY revenue_per_event DESC;
```

### 3.4 — Go: Migration Runner

```go
// go/cmd/migrator/main.go
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "path/filepath"
    "sort"
    "strings"
    "time"

    "github.com/jackc/pgx/v5"
)

type Migration struct {
    Version     int
    Name        string
    SQL         string
    AppliedAt   *time.Time
}

func main() {
    ctx := context.Background()
    conn, err := pgx.Connect(ctx, "postgres://dataeng:dataeng@localhost:5432/datawarehouse")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close(ctx)

    // Create migrations tracking table
    conn.Exec(ctx, `
        CREATE TABLE IF NOT EXISTS _migrations (
            version     INT PRIMARY KEY,
            name        VARCHAR(200) NOT NULL,
            applied_at  TIMESTAMP DEFAULT NOW()
        )
    `)

    // Load migration files
    migrations := loadMigrations("sql/migrations/")
    
    // Get applied versions
    rows, _ := conn.Query(ctx, "SELECT version FROM _migrations ORDER BY version")
    applied := map[int]bool{}
    for rows.Next() {
        var v int
        rows.Scan(&v)
        applied[v] = true
    }
    rows.Close()

    // Apply pending
    for _, m := range migrations {
        if applied[m.Version] {
            continue
        }
        
        tx, _ := conn.Begin(ctx)
        _, err := tx.Exec(ctx, m.SQL)
        if err != nil {
            tx.Rollback(ctx)
            log.Fatalf("Migration %d failed: %v", m.Version, err)
        }
        tx.Exec(ctx, "INSERT INTO _migrations (version, name) VALUES ($1, $2)",
            m.Version, m.Name)
        tx.Commit(ctx)
        
        fmt.Printf("✓ Applied migration %03d: %s\n", m.Version, m.Name)
    }
}

func loadMigrations(dir string) []Migration {
    var migrations []Migration
    entries, _ := os.ReadDir(dir)
    for _, e := range entries {
        if !strings.HasSuffix(e.Name(), ".sql") {
            continue
        }
        content, _ := os.ReadFile(filepath.Join(dir, e.Name()))
        var version int
        fmt.Sscanf(e.Name(), "%03d", &version)
        migrations = append(migrations, Migration{
            Version: version,
            Name:    strings.TrimSuffix(e.Name(), ".sql"),
            SQL:     string(content),
        })
    }
    sort.Slice(migrations, func(i, j int) bool {
        return migrations[i].Version < migrations[j].Version
    })
    return migrations
}
```

### 3.5 — Java: Data Federation Client

```java
// java/src/.../federation/TrinoQueryService.java
@Service
public class TrinoQueryService {

    private final JdbcTemplate trinoJdbc;

    public TrinoQueryService(
        @Qualifier("trinoDataSource") DataSource dataSource) {
        this.trinoJdbc = new JdbcTemplate(dataSource);
    }

    public List<FederatedResult> crossSourceQuery(String category) {
        String sql = """
            SELECT
                l.product_category,
                l.event_count,
                w.total_revenue,
                CAST(w.total_revenue AS DOUBLE) / l.event_count AS rev_per_event
            FROM hive.data_lake.product_events l
            JOIN postgresql.warehouse.mart_sales_summary w
                ON l.product_category = w.product_category
            WHERE l.product_category = ?
            ORDER BY rev_per_event DESC
            """;
        
        return trinoJdbc.query(sql, (rs, rowNum) -> new FederatedResult(
            rs.getString("product_category"),
            rs.getLong("event_count"),
            rs.getBigDecimal("total_revenue"),
            rs.getDouble("rev_per_event")
        ), category);
    }
}
```

---

## Critérios de Aceite

- [ ] Schema evolution: v1 → v2 → v3 com backward compatibility
- [ ] Schema Registry: registrar e testar compatibilidade
- [ ] Parquet schema merge funcional (ler arquivos com schemas diferentes)
- [ ] Trino configurado com ≥ 2 catalogs (Hive/S3 + PostgreSQL)
- [ ] Federated query funcional (JOIN entre S3 e PostgreSQL via Trino)
- [ ] Go migration runner com versão tracking
- [ ] Java federation client funcional
- [ ] ADR-017 e ADR-018 completos
- [ ] 2 DrawIOs (federation architecture + schema evolution)

---

## Definição de Pronto (DoD)

- [ ] 2 ADRs (schema evolution + data federation)
- [ ] 2 DrawIOs (federation architecture + schema evolution flow)
- [ ] Schema evolution com Avro + Schema Registry
- [ ] Parquet schema merge
- [ ] Trino data federation (S3 + PostgreSQL)
- [ ] Go migration runner
- [ ] Java federation client
- [ ] Commit: `feat(data-engineering-07): migration & integration`
