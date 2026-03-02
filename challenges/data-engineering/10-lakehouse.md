# Level 10 — Lakehouse Architecture

> **Objetivo:** Implementar uma arquitetura Lakehouse com Apache Iceberg e/ou
> Delta Lake sobre S3, combinando a flexibilidade do data lake com as garantias
> ACID de um data warehouse.

**Referência:** [06 — Architecture Patterns](../../.docs/data-engineering/06-architecture-patterns.md)

**Pré-requisito:** Level 09 concluído.

---

## Contexto

O **Lakehouse** resolve as limitações de data lakes (sem ACID, sem schema enforcement)
e data warehouses (custo alto, vendor lock-in). Usando **table formats** como
Apache Iceberg ou Delta Lake, obtemos: transações ACID sobre S3, time travel,
schema evolution, partition evolution e compaction — tudo sobre object storage
de baixo custo.

---

## Parte 1 — ADRs (3 obrigatórios)

### ADR-024: Table Format

**Arquivo:** `docs/adrs/ADR-024-table-format.md`

**Context:** Precisamos de um table format open que ofereça ACID, time travel
e schema evolution sobre S3.

**Options:**
1. **Apache Iceberg** — Netflix, spec aberta, hidden partitioning, partition evolution
2. **Delta Lake** — Databricks, transaction log em JSON, time travel
3. **Apache Hudi** — Uber, copy-on-write / merge-on-read, upserts nativos
4. **Apache Paimon** — Flink-first, streaming lakehouse
5. **Plain Parquet** — sem table format (baseline)

**Decision Drivers:**
- ACID guarantees
- Time travel & rollback
- Schema/partition evolution
- Engine compatibility (Spark, Trino, Flink, DuckDB)
- Community & adoption
- Merge/upsert performance

---

### ADR-025: Compaction Strategy

**Arquivo:** `docs/adrs/ADR-025-compaction-strategy.md`

**Context:** Streaming ingest gera muitos small files. Precisamos de compaction
para manter performance de leitura.

**Options:**
1. **Iceberg rewrite_data_files** — procedure nativa, bin-pack ou sort
2. **Spark compaction job** — custom job de merge em Spark
3. **Flink compaction** — real-time compaction durante streaming
4. **Scheduled compaction** — cron job periódico (hourly, daily)

**Decision Drivers:**
- Performance impact durante compaction
- Frequency vs file count tradeoff
- Concurrency (leitura durante compaction)
- Target file size (128MB–512MB ideal)

---

### ADR-026: Query Engine for Lakehouse

**Arquivo:** `docs/adrs/ADR-026-lakehouse-query-engine.md`

**Context:** Precisamos de query engines para ler tabelas Iceberg/Delta no S3.

**Options:**
1. **Trino** — distributed SQL, Iceberg/Delta connectors
2. **DuckDB** — embedded, Iceberg support nativo, single-node
3. **Spark SQL** — distributed, suporte nativo a todos os formatos
4. **Athena v3** — serverless, Iceberg nativo na AWS

**Decision Drivers:**
- Latência de query (interactive < 5s)
- Custo de infraestrutura
- Suporte a time travel queries
- Metadata caching

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

### DrawIO-019: Lakehouse Architecture

**Arquivo:** `docs/diagrams/10-lakehouse-architecture.drawio`

```
┌────────────────────────────────────────────────────────────────┐
│                    LAKEHOUSE ARCHITECTURE                      │
│                                                                │
│  ┌─────────┐    ┌──────────────────────────────────────┐       │
│  │ Sources │───▶│         S3 OBJECT STORAGE            │       │
│  │         │    │                                      │       │
│  │ API     │    │  ┌────────────────────────────────┐  │       │
│  │ CDC     │    │  │     TABLE FORMAT (Iceberg)     │  │       │
│  │ Files   │    │  │                                │  │       │
│  │ Streams │    │  │  metadata/       data/         │  │       │
│  └─────────┘    │  │  ├── snap-001   ├── 0001.parq  │  │       │
│                 │  │  ├── snap-002   ├── 0002.parq  │  │       │
│                 │  │  └── manifest   └── 0003.parq  │  │       │
│                 │  │                                │  │       │
│                 │  │  ACID │ Time Travel │ Schema   │  │       │
│                 │  │  Txns │ Snapshots   │ Evolution│  │       │
│                 │  └────────────────────────────────┘  │       │
│                 └──────────────────────────────────────┘       │
│                              │                                 │
│         ┌────────────────────┼────────────────────┐            │
│         ▼                    ▼                    ▼            │
│  ┌──────────┐        ┌──────────┐        ┌──────────┐         │
│  │  Trino   │        │  DuckDB  │        │  Spark   │         │
│  │ (distrib)│        │(embedded)│        │  (batch) │         │
│  └──────────┘        └──────────┘        └──────────┘         │
└────────────────────────────────────────────────────────────────┘
```

### DrawIO-020: Iceberg Table Internals

**Arquivo:** `docs/diagrams/10-iceberg-internals.drawio`

```
┌──────────────────────────────────────────────────────────┐
│                ICEBERG TABLE STRUCTURE                    │
│                                                          │
│  Catalog ──▶ Table Metadata (version-N.metadata.json)   │
│                    │                                     │
│                    ▼                                     │
│              Snapshot List                               │
│              ├── snap-001 (2024-01-15 08:00)            │
│              ├── snap-002 (2024-01-15 12:00)            │
│              └── snap-003 (2024-01-16 08:00) ← current │
│                    │                                     │
│                    ▼                                     │
│              Manifest List (snap-003)                    │
│              ├── manifest-A.avro                         │
│              └── manifest-B.avro                         │
│                    │                                     │
│                    ▼                                     │
│              Manifest Files                              │
│              ├── file: s3://lake/data/part-00001.parquet │
│              │   partition: year=2024, month=01          │
│              │   record_count: 100,000                   │
│              │   min/max: amount [10.5, 999.9]          │
│              ├── file: s3://lake/data/part-00002.parquet │
│              └── ...                                     │
│                                                          │
│  Hidden Partitioning:                                    │
│    year(ts) → no partition column needed in data         │
│    bucket(id, 16) → automatic bucketing                  │
└──────────────────────────────────────────────────────────┘
```

---

## Parte 3 — Implementação

### 3.1 — Apache Iceberg com PySpark

```python
# python/lakehouse/iceberg_spark.py
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

def create_spark_iceberg():
    """Cria SparkSession com suporte a Iceberg."""
    return (SparkSession.builder
        .appName("Lakehouse-Iceberg")
        .config("spark.jars.packages",
                "org.apache.iceberg:iceberg-spark-runtime-3.5_2.12:1.5.0,"
                "org.apache.hadoop:hadoop-aws:3.3.4")
        .config("spark.sql.catalog.lakehouse", "org.apache.iceberg.spark.SparkCatalog")
        .config("spark.sql.catalog.lakehouse.type", "hadoop")
        .config("spark.sql.catalog.lakehouse.warehouse", "s3a://data-lake-curated/iceberg/")
        .config("spark.hadoop.fs.s3a.endpoint", "http://localhost:4566")
        .config("spark.hadoop.fs.s3a.access.key", "test")
        .config("spark.hadoop.fs.s3a.secret.key", "test")
        .config("spark.hadoop.fs.s3a.path.style.access", "true")
        .config("spark.sql.defaultCatalog", "lakehouse")
        .getOrCreate())

def create_iceberg_table(spark):
    """Cria tabela Iceberg com hidden partitioning."""
    spark.sql("""
        CREATE TABLE IF NOT EXISTS lakehouse.orders (
            order_id    BIGINT,
            customer_id INT,
            product     STRING,
            amount      DOUBLE,
            status      STRING,
            created_at  TIMESTAMP
        )
        USING iceberg
        PARTITIONED BY (year(created_at), month(created_at))
        TBLPROPERTIES (
            'format-version' = '2',
            'write.parquet.compression-codec' = 'zstd'
        )
    """)
    print("✓ Created Iceberg table: lakehouse.orders")

def insert_data(spark, df):
    """Insert data into Iceberg table."""
    df.writeTo("lakehouse.orders").append()
    print(f"✓ Inserted {df.count()} rows")

def time_travel_query(spark, snapshot_id: int = None, timestamp: str = None):
    """Query historical data via time travel."""
    if snapshot_id:
        df = spark.read.option("snapshot-id", snapshot_id).table("lakehouse.orders")
    elif timestamp:
        df = spark.read.option("as-of-timestamp", timestamp).table("lakehouse.orders")
    else:
        df = spark.table("lakehouse.orders")
    
    return df

def list_snapshots(spark):
    """Lista snapshots da tabela (time travel history)."""
    snapshots = spark.sql("""
        SELECT snapshot_id, committed_at, operation, summary
        FROM lakehouse.orders.snapshots
        ORDER BY committed_at DESC
    """)
    snapshots.show(truncate=False)
    return snapshots

def rollback_to_snapshot(spark, snapshot_id: int):
    """Rollback para um snapshot anterior."""
    spark.sql(f"""
        CALL lakehouse.system.rollback_to_snapshot('orders', {snapshot_id})
    """)
    print(f"✓ Rolled back to snapshot {snapshot_id}")

def schema_evolution(spark):
    """Evolução de schema sem rewrite."""
    spark.sql("ALTER TABLE lakehouse.orders ADD COLUMNS (currency STRING)")
    spark.sql("ALTER TABLE lakehouse.orders ADD COLUMNS (region STRING)")
    print("✓ Schema evolved: added currency, region")

def partition_evolution(spark):
    """Muda particionamento sem rewrite de dados existentes."""
    spark.sql("""
        ALTER TABLE lakehouse.orders 
        ADD PARTITION FIELD day(created_at)
    """)
    print("✓ Partition evolved: added day(created_at)")

def compact_files(spark, target_size_mb: int = 256):
    """Compaction: merge small files into larger ones."""
    spark.sql(f"""
        CALL lakehouse.system.rewrite_data_files(
            table => 'orders',
            options => map(
                'target-file-size-bytes', '{target_size_mb * 1024 * 1024}',
                'min-file-size-bytes', '{target_size_mb * 1024 * 1024 // 4}'
            )
        )
    """)
    print(f"✓ Compaction complete (target: {target_size_mb}MB)")

def expire_snapshots(spark, days: int = 7):
    """Remove snapshots antigos para economizar storage."""
    spark.sql(f"""
        CALL lakehouse.system.expire_snapshots(
            table => 'orders',
            older_than => TIMESTAMP '{days} days ago',
            retain_last => 5
        )
    """)
    print(f"✓ Expired snapshots older than {days} days")
```

### 3.2 — Iceberg MERGE (Upserts)

```python
# python/lakehouse/iceberg_merge.py
def merge_incremental(spark, source_df, target_table: str):
    """MERGE INTO: upsert (insert or update existing rows)."""
    source_df.createOrReplaceTempView("source_data")
    
    spark.sql(f"""
        MERGE INTO {target_table} t
        USING source_data s
        ON t.order_id = s.order_id
        WHEN MATCHED THEN
            UPDATE SET
                t.status = s.status,
                t.amount = s.amount
        WHEN NOT MATCHED THEN
            INSERT *
    """)
    print("✓ MERGE complete (upsert)")

def delete_by_customer(spark, table: str, customer_id: int):
    """GDPR/LGPD right to erasure via Iceberg DELETE."""
    spark.sql(f"""
        DELETE FROM {table} 
        WHERE customer_id = {customer_id}
    """)
    print(f"✓ Deleted all data for customer {customer_id}")
```

### 3.3 — DuckDB Iceberg Reader

```python
# python/lakehouse/duckdb_iceberg.py
import duckdb

def query_iceberg_with_duckdb():
    """Query Iceberg tables com DuckDB (single node, fast)."""
    conn = duckdb.connect()
    
    conn.execute("INSTALL iceberg; LOAD iceberg;")
    conn.execute("INSTALL httpfs; LOAD httpfs;")
    conn.execute("SET s3_endpoint='localhost:4566';")
    conn.execute("SET s3_access_key_id='test';")
    conn.execute("SET s3_secret_access_key='test';")
    conn.execute("SET s3_use_ssl=false;")
    conn.execute("SET s3_url_style='path';")
    
    # Read Iceberg table
    result = conn.execute("""
        SELECT 
            status,
            COUNT(*) as count,
            SUM(amount) as total,
            AVG(amount) as avg_amount
        FROM iceberg_scan('s3://data-lake-curated/iceberg/orders')
        GROUP BY status
        ORDER BY total DESC
    """).fetchdf()
    
    print(result)
    return result
```

### 3.4 — Go: Iceberg Metadata Reader

```go
// go/cmd/iceberg-reader/main.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"

    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

type IcebergMetadata struct {
    FormatVersion int `json:"format-version"`
    TableUUID     string `json:"table-uuid"`
    Location      string `json:"location"`
    LastUpdated   int64  `json:"last-updated-ms"`
    Snapshots     []struct {
        SnapshotID  int64  `json:"snapshot-id"`
        TimestampMs int64  `json:"timestamp-ms"`
        Summary     map[string]string `json:"summary"`
    } `json:"snapshots"`
    CurrentSnapshotID int64 `json:"current-snapshot-id"`
}

func main() {
    cfg, _ := config.LoadDefaultConfig(context.TODO(), config.WithRegion("us-east-1"))
    client := s3.NewFromConfig(cfg, func(o *s3.Options) {
        o.BaseEndpoint = aws.String("http://localhost:4566")
        o.UsePathStyle = true
    })

    // Read Iceberg metadata
    result, err := client.GetObject(context.TODO(), &s3.GetObjectInput{
        Bucket: aws.String("data-lake-curated"),
        Key:    aws.String("iceberg/orders/metadata/v1.metadata.json"),
    })
    if err != nil {
        log.Fatal(err)
    }
    defer result.Body.Close()

    var meta IcebergMetadata
    json.NewDecoder(result.Body).Decode(&meta)

    fmt.Printf("Table: %s (format v%d)\n", meta.TableUUID, meta.FormatVersion)
    fmt.Printf("Snapshots: %d\n", len(meta.Snapshots))
    for _, snap := range meta.Snapshots {
        current := ""
        if snap.SnapshotID == meta.CurrentSnapshotID {
            current = " ← CURRENT"
        }
        fmt.Printf("  #%d: %s records, %s files%s\n",
            snap.SnapshotID,
            snap.Summary["total-records"],
            snap.Summary["total-data-files"],
            current)
    }
}
```

### 3.5 — Java: Iceberg SDK

```java
// java/src/.../lakehouse/IcebergService.java
@Service
public class IcebergService {

    private final Table ordersTable;

    public IcebergService() {
        HadoopCatalog catalog = new HadoopCatalog();
        catalog.initialize("lakehouse", Map.of(
            "warehouse", "s3a://data-lake-curated/iceberg/"
        ));
        this.ordersTable = catalog.loadTable(TableIdentifier.of("orders"));
    }

    public List<Snapshot> listSnapshots() {
        return Lists.newArrayList(ordersTable.snapshots());
    }

    public long currentSnapshotId() {
        return ordersTable.currentSnapshot().snapshotId();
    }

    public void expireOldSnapshots(Duration olderThan) {
        ordersTable.expireSnapshots()
            .expireOlderThan(System.currentTimeMillis() - olderThan.toMillis())
            .retainLast(5)
            .commit();
    }

    public TableScan planScan(Expression filter) {
        return ordersTable.newScan()
            .filter(filter)
            .select("order_id", "customer_id", "amount", "status");
    }
}
```

---

## Critérios de Aceite

- [ ] Tabela Iceberg criada no S3 com hidden partitioning
- [ ] Insert, update (MERGE), delete funcionais
- [ ] ≥ 3 snapshots criados (time travel history)
- [ ] Time travel query: ler dados de snapshot anterior
- [ ] Rollback para snapshot anterior funcional
- [ ] Schema evolution: adicionar colunas sem rewrite
- [ ] Partition evolution funcional
- [ ] Compaction: small files → target size (256MB)
- [ ] Expire snapshots: remover snapshots antigos
- [ ] DuckDB query sobre Iceberg funcional
- [ ] Go metadata reader funcional
- [ ] Java Iceberg SDK funcional
- [ ] ADR-024, ADR-025, ADR-026 completos
- [ ] 2 DrawIOs (lakehouse architecture + Iceberg internals)

---

## Definição de Pronto (DoD)

- [ ] 3 ADRs (table format + compaction + query engine)
- [ ] 2 DrawIOs (lakehouse architecture + Iceberg internals)
- [ ] Iceberg CRUD + MERGE funcional
- [ ] Time travel + rollback
- [ ] Schema + partition evolution
- [ ] Compaction + expire snapshots
- [ ] DuckDB + Go + Java reading Iceberg
- [ ] Commit: `feat(data-engineering-10): lakehouse architecture`
