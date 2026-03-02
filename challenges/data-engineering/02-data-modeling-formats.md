# Level 02 — Data Modeling & Storage Formats

> **Objetivo:** Dominar modelagem dimensional (star/snowflake), escolher formatos
> de armazenamento (Parquet, Avro, ORC, CSV, JSON) e implementar particionamento
> eficiente para queries analíticas.

**Referência:** [02 — Data Storage](../../.docs/data-engineering/02-data-storage.md)

**Pré-requisito:** Level 01 concluído.

---

## Contexto

A performance de pipelines de dados depende diretamente de **como os dados são
modelados** e **em qual formato são armazenados**. Formatos colunares (Parquet)
reduzem I/O em queries analíticas, enquanto formatos row-based (Avro) são melhores
para ingestão. O particionamento correto evita full scans em datasets massivos.

---

## Parte 1 — ADRs (2 obrigatórios)

### ADR-004: Storage Format Strategy

**Arquivo:** `docs/adrs/ADR-004-storage-format-strategy.md`

**Context:** Precisamos definir formatos padrão para cada zona do data lake.

**Options:**
1. **Parquet everywhere** — formato columnar único para todas as zonas
2. **JSON/CSV → Parquet → Parquet** — raw em texto, curated/refined em columnar
3. **Avro → Parquet → Parquet** — raw com schema evolution (Avro), analytics em Parquet
4. **JSON → ORC → Parquet** — mix de formatos por caso de uso

**Decision Drivers:**
- Compression ratio (custo de storage)
- Query performance (columnar pruning)
- Schema evolution (backward/forward compatibility)
- Ecossistema tooling (Spark, DuckDB, Trino, Athena)
- Streaming compatibility (Kafka, Kinesis)

**Critérios de aceite:**
- [ ] Benchmark de compression: CSV vs JSON vs Avro vs Parquet vs ORC
- [ ] Benchmark de query: tempo de scan com 1M+ rows
- [ ] Regra por zona documentada

---

### ADR-005: Partitioning Strategy

**Arquivo:** `docs/adrs/ADR-005-partitioning-strategy.md`

**Context:** Datasets grandes (>100GB) precisam de particionamento para performance.

**Options:**
1. **Date-based** — `year=2024/month=01/day=15/`
2. **Domain + Date** — `domain=orders/year=2024/month=01/`
3. **Hash-based** — `bucket=00/` a `bucket=99/`
4. **Hive-style** — `key=value/` partitions (compatível com Athena/Trino)
5. **Z-order / Hilbert curve** — multi-dimensional clustering (Iceberg/Delta)

**Decision Drivers:**
- Partition pruning em queries
- Small files problem (< 128MB por partition)
- Write amplification
- Compatibilidade com Athena/Trino/DuckDB

---

## Parte 2 — Diagrama DrawIO (1 obrigatório)

### DrawIO-004: Format & Partitioning Decision Tree

**Arquivo:** `docs/diagrams/02-format-partitioning.drawio`

```
┌─────────────────────────────────────────────────────────────────┐
│              FORMAT & PARTITIONING DECISION TREE                │
│                                                                 │
│  Dado chega ──▶ É streaming?                                   │
│                  │                                              │
│           ┌──Yes─┤──No──┐                                      │
│           │              │                                      │
│        Avro/JSON      Batch?                                   │
│           │              │                                      │
│           │       ┌──Yes─┤──No──┐                              │
│           │       │             │                               │
│           ▼       ▼             ▼                               │
│        Kinesis  CSV/JSON     API JSON                          │
│           │       │             │                               │
│           └───────┴──────┬──────┘                              │
│                          ▼                                      │
│                   RAW Zone (original format)                   │
│                          │                                      │
│                    Transform to Parquet                         │
│                          │                                      │
│                    Partition by:                                │
│              Date (Hive-style) + Domain                        │
│                          │                                      │
│                          ▼                                      │
│              CURATED Zone (Parquet/Snappy)                      │
│              year=YYYY/month=MM/day=DD/                         │
│                          │                                      │
│                    Aggregate + Compact                          │
│                          │                                      │
│                          ▼                                      │
│              REFINED Zone (Parquet/Zstd)                        │
│              Compacted files ≥ 128MB                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Parte 3 — Implementação

### 3.1 — Format Benchmark (Python)

**Objetivo:** Comparar formatos em compression, write time, read time.

```python
# python/benchmarks/format_benchmark.py
import time
import os
import json
from io import BytesIO

import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
import pyarrow.csv as pcsv
import fastavro

def generate_data(n_rows: int = 1_000_000) -> pd.DataFrame:
    """Gera dataset sintético para benchmark."""
    import numpy as np
    np.random.seed(42)
    return pd.DataFrame({
        "id": range(n_rows),
        "customer_id": np.random.randint(1, 10000, n_rows),
        "product": np.random.choice(["A", "B", "C", "D", "E"], n_rows),
        "amount": np.random.uniform(10.0, 1000.0, n_rows).round(2),
        "quantity": np.random.randint(1, 100, n_rows),
        "timestamp": pd.date_range("2024-01-01", periods=n_rows, freq="s"),
        "region": np.random.choice(["US", "EU", "APAC", "LATAM"], n_rows),
        "status": np.random.choice(["pending", "completed", "cancelled"], n_rows),
    })

def benchmark_csv(df: pd.DataFrame):
    start = time.time()
    buf = BytesIO()
    df.to_csv(buf, index=False)
    write_time = time.time() - start
    size = buf.tell()
    
    buf.seek(0)
    start = time.time()
    pd.read_csv(buf)
    read_time = time.time() - start
    
    return {"format": "CSV", "size_mb": size / 1e6, 
            "write_s": write_time, "read_s": read_time}

def benchmark_parquet(df: pd.DataFrame, compression: str = "snappy"):
    table = pa.Table.from_pandas(df)
    buf = BytesIO()
    
    start = time.time()
    pq.write_table(table, buf, compression=compression)
    write_time = time.time() - start
    size = buf.tell()
    
    buf.seek(0)
    start = time.time()
    pq.read_table(buf)
    read_time = time.time() - start
    
    return {"format": f"Parquet ({compression})", "size_mb": size / 1e6,
            "write_s": write_time, "read_s": read_time}

def benchmark_json(df: pd.DataFrame):
    start = time.time()
    buf = BytesIO()
    df.to_json(buf, orient="records", lines=True)
    write_time = time.time() - start
    size = buf.tell()
    
    buf.seek(0)
    start = time.time()
    pd.read_json(buf, lines=True)
    read_time = time.time() - start
    
    return {"format": "JSONL", "size_mb": size / 1e6,
            "write_s": write_time, "read_s": read_time}

if __name__ == "__main__":
    df = generate_data(1_000_000)
    results = [
        benchmark_csv(df),
        benchmark_json(df),
        benchmark_parquet(df, "snappy"),
        benchmark_parquet(df, "gzip"),
        benchmark_parquet(df, "zstd"),
    ]
    
    print(f"{'Format':<25} {'Size (MB)':>10} {'Write (s)':>10} {'Read (s)':>10}")
    print("-" * 60)
    for r in results:
        print(f"{r['format']:<25} {r['size_mb']:>10.2f} {r['write_s']:>10.3f} {r['read_s']:>10.3f}")
```

### 3.2 — Partitioned Writes (Python + PyArrow)

**Objetivo:** Escrever Parquet particionado no S3 (Hive-style).

```python
# python/etl/partitioned_writer.py
import pyarrow as pa
import pyarrow.parquet as pq
import pyarrow.fs as pafs

def write_partitioned(df, bucket: str, prefix: str, partition_cols: list[str]):
    """Escreve DataFrame particionado em S3 no formato Hive."""
    table = pa.Table.from_pandas(df)
    
    # S3 filesystem via LocalStack
    fs = pafs.S3FileSystem(
        endpoint_override="http://localhost:4566",
        access_key="test",
        secret_key="test",
        region="us-east-1",
        scheme="http",
    )
    
    pq.write_to_dataset(
        table,
        root_path=f"{bucket}/{prefix}",
        partition_cols=partition_cols,
        filesystem=fs,
        use_legacy_dataset=False,
    )
    print(f"✓ Partitioned write: s3://{bucket}/{prefix}/ by {partition_cols}")

def read_partitioned(bucket: str, prefix: str, filters=None):
    """Lê dataset particionado com partition pruning."""
    fs = pafs.S3FileSystem(
        endpoint_override="http://localhost:4566",
        access_key="test",
        secret_key="test",
        region="us-east-1",
        scheme="http",
    )
    
    dataset = pq.ParquetDataset(
        f"{bucket}/{prefix}",
        filesystem=fs,
        filters=filters,  # ex: [("year", "=", 2024), ("month", "=", 1)]
    )
    
    table = dataset.read()
    print(f"✓ Read {table.num_rows} rows with pruning: {filters}")
    return table.to_pandas()
```

### 3.3 — Dimensional Modeling (SQL)

**Objetivo:** Implementar snowflake schema com hierarquias.

```sql
-- sql/warehouse/002_snowflake_schema.sql

-- Snowflake: dimensão de localização normalizada
CREATE TABLE warehouse.dim_country (
    country_id   SERIAL PRIMARY KEY,
    country_name VARCHAR(100) NOT NULL,
    country_code CHAR(2) NOT NULL
);

CREATE TABLE warehouse.dim_state (
    state_id     SERIAL PRIMARY KEY,
    state_name   VARCHAR(100) NOT NULL,
    state_code   CHAR(2),
    country_id   INT REFERENCES warehouse.dim_country(country_id)
);

CREATE TABLE warehouse.dim_city (
    city_id      SERIAL PRIMARY KEY,
    city_name    VARCHAR(100) NOT NULL,
    state_id     INT REFERENCES warehouse.dim_state(state_id),
    population   INT
);

-- SCD Type 2 (Slowly Changing Dimension)
CREATE TABLE warehouse.dim_customer_scd2 (
    surrogate_id   BIGSERIAL PRIMARY KEY,
    customer_id    INT NOT NULL,        -- natural key
    name           VARCHAR(200) NOT NULL,
    email          VARCHAR(200),
    segment        VARCHAR(50),
    city_id        INT REFERENCES warehouse.dim_city(city_id),
    valid_from     TIMESTAMP NOT NULL DEFAULT NOW(),
    valid_to       TIMESTAMP,           -- NULL = current
    is_current     BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_scd2_customer ON warehouse.dim_customer_scd2(customer_id, is_current);
```

### 3.4 — Go: Parquet Reader/Writer

**Objetivo:** Ler e escrever Parquet nativamente em Go.

```go
// go/pkg/parquet/readwrite.go
package parquet

import (
    "fmt"
    "os"

    "github.com/parquet-go/parquet-go"
)

type OrderRecord struct {
    ID         int64   `parquet:"id"`
    CustomerID int64   `parquet:"customer_id"`
    Product    string  `parquet:"product"`
    Amount     float64 `parquet:"amount"`
    Quantity   int32   `parquet:"quantity"`
    Region     string  `parquet:"region"`
    Status     string  `parquet:"status"`
}

func WriteParquet(path string, records []OrderRecord) error {
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    defer f.Close()

    w := parquet.NewGenericWriter[OrderRecord](f)
    _, err = w.Write(records)
    if err != nil {
        return err
    }
    return w.Close()
}

func ReadParquet(path string) ([]OrderRecord, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer f.Close()

    stat, _ := f.Stat()
    reader := parquet.NewGenericReader[OrderRecord](f, parquet.FileSize(stat.Size()))
    
    records := make([]OrderRecord, reader.NumRows())
    n, err := reader.Read(records)
    if err != nil {
        return nil, err
    }
    
    fmt.Printf("✓ Read %d records from %s\n", n, path)
    return records[:n], nil
}
```

### 3.5 — Java: Parquet via Apache Arrow

```java
// java/src/.../parquet/ParquetService.java
@Service
public class ParquetService {

    public void writeParquet(List<OrderRecord> records, Path outputPath) 
            throws IOException {
        Schema schema = new Schema(List.of(
            Field.nullable("id", new ArrowType.Int(64, true)),
            Field.nullable("customer_id", new ArrowType.Int(64, true)),
            Field.nullable("product", ArrowType.Utf8.INSTANCE),
            Field.nullable("amount", new ArrowType.FloatingPoint(FloatingPointPrecision.DOUBLE))
        ));

        try (var allocator = new RootAllocator();
             var root = VectorSchemaRoot.create(schema, allocator);
             var writer = new ArrowFileWriter(root, null, 
                 new FileOutputStream(outputPath.toFile()).getChannel())) {
            
            writer.start();
            // Populate vectors from records...
            writer.writeBatch();
            writer.end();
        }
    }
}
```

---

## Critérios de Aceite

- [ ] Benchmark executado com 1M rows: CSV vs JSON vs Parquet (snappy/gzip/zstd)
- [ ] Tabela comparativa documentada (size, write time, read time)
- [ ] Parquet particionado gravado no S3 (Hive-style)
- [ ] Partition pruning validado (query com filtro lê menos partições)
- [ ] Star schema + Snowflake schema implementados
- [ ] SCD Type 2 implementado
- [ ] Go: Parquet read/write funcional
- [ ] Java: Parquet read/write funcional
- [ ] ADR-004 e ADR-005 completos
- [ ] DrawIO do decision tree de formatos

---

## Definição de Pronto (DoD)

- [ ] 2 ADRs (format strategy + partitioning)
- [ ] 1 DrawIO (format & partitioning decision tree)
- [ ] Benchmark de formatos executado e documentado
- [ ] Writes particionados no S3
- [ ] Dimensional modeling (star + snowflake + SCD2)
- [ ] Multi-language Parquet I/O (Python + Go + Java)
- [ ] Commit: `feat(data-engineering-02): data modeling & formats`
