# Level 09 — Data Governance

> **Objetivo:** Implementar catálogo de dados, data lineage, controle de acesso
> granular (Lake Formation), conformidade LGPD/GDPR e data discovery.

**Referência:** [05 — Data Governance](../../.docs/data-engineering/05-data-governance.md)

**Pré-requisito:** Level 08 concluído.

---

## Contexto

Data governance garante que dados sejam **descobríveis** (catálogo), **rastreáveis**
(lineage), **seguros** (acesso granular), e **conformes** (LGPD/GDPR). Sem
governance, organizações perdem confiança nos dados, cometem violações regulatórias
e duplicam esforços. Neste nível, implementamos um ecossistema completo de governance.

---

## Parte 1 — ADRs (3 obrigatórios)

### ADR-021: Data Catalog Platform

**Arquivo:** `docs/adrs/ADR-021-data-catalog.md`

**Context:** Precisamos de um catálogo centralizado para discovery e metadata
management de todos os datasets.

**Options:**
1. **AWS Glue Data Catalog** — catálogo serverless, integração Athena/Spark
2. **Apache Atlas** — open-source, lineage nativo, Hadoop ecosystem
3. **DataHub** — LinkedIn open-source, GraphQL API, modern UI
4. **OpenMetadata** — open-source, data quality integrado, lineage
5. **Amundsen** — Lyft open-source, search-focused, Neo4j backend

**Decision Drivers:**
- Search e discovery UX
- Lineage visualization
- API programmatic access
- Integração com Airflow, Spark, dbt
- Self-hosted vs managed

---

### ADR-022: Data Lineage Strategy

**Arquivo:** `docs/adrs/ADR-022-data-lineage.md`

**Context:** Precisamos rastrear a origem, transformações e destino de cada dataset
para auditoria e impact analysis.

**Options:**
1. **OpenLineage** — standard aberto, integração Airflow/Spark/dbt
2. **dbt lineage** — automático via ref(), visual no dbt docs
3. **Custom lineage** — metadata manual em cada pipeline
4. **Glue Data Catalog** — lineage básico via crawlers
5. **Marquez** — backend para OpenLineage, API REST

**Decision Drivers:**
- Granularidade (dataset vs column-level)
- Automatismo (zero-config vs annotate)
- Cross-system (Airflow → Spark → dbt → warehouse)
- Impact analysis capability

---

### ADR-023: Data Privacy & Compliance

**Arquivo:** `docs/adrs/ADR-023-privacy-compliance.md`

**Context:** Dados pessoais (PII) precisam de proteção conforme LGPD/GDPR:
anonimização, pseudonimização, right to erasure, consent management.

**Options:**
1. **Column-level encryption** — encrypt PII columns at rest (KMS)
2. **Tokenization** — substitui PII por tokens reversíveis
3. **Pseudonymization** — hash irreversível (HMAC-SHA256)
4. **Dynamic data masking** — mascaramento na query (view-based)
5. **Lake Formation** — row/column-level security no data lake

**Decision Drivers:**
- Reversibilidade (precisa des-pseudonimizar para right to erasure?)
- Performance (overhead de encrypt/decrypt)
- Granularidade (table, column, row, cell level)
- Auditoria e compliance reporting
- Custo operacional

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

### DrawIO-017: Governance Architecture

**Arquivo:** `docs/diagrams/09-governance-architecture.drawio`

```
┌──────────────────────────────────────────────────────────┐
│                 DATA GOVERNANCE LAYER                    │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │ Catalog  │  │ Lineage  │  │ Access   │  │Privacy  │ │
│  │          │  │          │  │ Control  │  │         │ │
│  │ Glue /   │  │ Open     │  │ Lake     │  │ PII     │ │
│  │ DataHub  │  │ Lineage  │  │ Formation│  │ Masking │ │
│  │          │  │ + Marquez│  │ + IAM    │  │ + KMS   │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬────┘ │
│       │              │             │              │      │
│       └──────────────┴─────────────┴──────────────┘      │
│                          │                               │
└──────────────────────────┼───────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌──────────┐    ┌──────────┐    ┌──────────┐
   │ Data Lake│    │Warehouse │    │ Streams  │
   │ (S3)    │    │(Postgres)│    │ (Kafka)  │
   └──────────┘    └──────────┘    └──────────┘
```

### DrawIO-018: Data Lineage Graph

**Arquivo:** `docs/diagrams/09-lineage-graph.drawio`

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐
│ orders_api  │───▶│ CDC Debezium │───▶│ Kafka topic  │
│ (source)    │    │              │    │ cdc.orders   │
└─────────────┘    └──────────────┘    └──────┬───────┘
                                              │
                                    ┌─────────▼────────┐
                                    │  S3 Raw Zone     │
                                    │  raw/orders/     │
                                    └─────────┬────────┘
                                              │
                                    ┌─────────▼────────┐
                                    │  Spark ETL       │
                                    │  (transform)     │
                                    └─────────┬────────┘
                                              │
                              ┌───────────────┼───────────────┐
                              ▼                               ▼
                   ┌──────────────┐                 ┌──────────────┐
                   │ S3 Curated   │                 │ Warehouse    │
                   │ curated/     │                 │ fact_orders  │
                   │ orders/      │                 └──────┬───────┘
                   └──────────────┘                        │
                                                  ┌────────▼───────┐
                                                  │ dbt            │
                                                  │ mart_sales_    │
                                                  │ summary        │
                                                  └────────────────┘
```

---

## Parte 3 — Implementação

### 3.1 — Data Catalog (Glue via LocalStack)

```python
# python/governance/catalog.py
import boto3
import json

glue = boto3.client("glue", endpoint_url="http://localhost:4566",
                     aws_access_key_id="test", aws_secret_access_key="test",
                     region_name="us-east-1")

def create_database(name: str, description: str):
    """Cria database no Glue Data Catalog."""
    glue.create_database(
        DatabaseInput={
            "Name": name,
            "Description": description,
            "Parameters": {
                "owner": "data-engineering",
                "classification": "internal",
            },
        }
    )
    print(f"✓ Created database: {name}")

def register_table(database: str, table_name: str, s3_location: str,
                    columns: list[dict], partition_keys: list[dict] = None):
    """Registra tabela no Glue Data Catalog apontando para S3."""
    table_input = {
        "Name": table_name,
        "Description": f"Data lake table: {table_name}",
        "StorageDescriptor": {
            "Columns": columns,
            "Location": s3_location,
            "InputFormat": "org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat",
            "OutputFormat": "org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat",
            "SerdeInfo": {
                "SerializationLibrary": "org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe",
            },
        },
        "PartitionKeys": partition_keys or [],
        "TableType": "EXTERNAL_TABLE",
        "Parameters": {
            "classification": "parquet",
            "compressionType": "snappy",
        },
    }
    
    glue.create_table(DatabaseName=database, TableInput=table_input)
    print(f"✓ Registered table: {database}.{table_name}")

def setup_catalog():
    """Configura catálogo completo do data lake."""
    create_database("data_lake_raw", "Raw zone - dados brutos")
    create_database("data_lake_curated", "Curated zone - dados limpos")
    create_database("data_lake_refined", "Refined zone - dados agregados")
    
    order_columns = [
        {"Name": "order_id", "Type": "bigint", "Comment": "Unique order identifier"},
        {"Name": "customer_id", "Type": "int", "Comment": "Customer FK"},
        {"Name": "product_id", "Type": "int", "Comment": "Product FK"},
        {"Name": "amount", "Type": "double", "Comment": "Order total (BRL)"},
        {"Name": "status", "Type": "string", "Comment": "Order status"},
        {"Name": "created_at", "Type": "timestamp", "Comment": "Creation timestamp"},
    ]
    
    partition_keys = [
        {"Name": "year", "Type": "int"},
        {"Name": "month", "Type": "int"},
    ]
    
    register_table("data_lake_curated", "orders", 
                    "s3://data-lake-curated/orders/", 
                    order_columns, partition_keys)
```

### 3.2 — Data Lineage (OpenLineage + Custom)

```python
# python/governance/lineage.py
import json
import uuid
from datetime import datetime
from dataclasses import dataclass, asdict
from typing import Optional
import boto3

@dataclass
class LineageEvent:
    run_id: str
    job_name: str
    job_namespace: str
    event_type: str  # START, COMPLETE, FAIL
    event_time: str
    inputs: list[dict]
    outputs: list[dict]
    producer: str = "data-engineering/custom"

def emit_lineage(event: LineageEvent):
    """Emite evento de lineage para S3 (metadata store)."""
    s3 = boto3.client("s3", endpoint_url="http://localhost:4566",
                       aws_access_key_id="test", aws_secret_access_key="test")
    
    key = f"lineage/{event.job_namespace}/{event.job_name}/{event.run_id}.json"
    s3.put_object(
        Bucket="data-lake-raw",
        Key=key,
        Body=json.dumps(asdict(event), indent=2),
    )
    print(f"✓ Lineage event: {event.event_type} for {event.job_name}")

def track_etl_lineage(job_name: str, source_tables: list[str], 
                       target_table: str, status: str = "COMPLETE"):
    """Wrapper para rastrear lineage de um ETL job."""
    run_id = str(uuid.uuid4())
    
    # START event
    emit_lineage(LineageEvent(
        run_id=run_id,
        job_name=job_name,
        job_namespace="data-engineering",
        event_type="START",
        event_time=datetime.utcnow().isoformat(),
        inputs=[{"namespace": "data-lake", "name": t} for t in source_tables],
        outputs=[],
    ))
    
    # COMPLETE event
    emit_lineage(LineageEvent(
        run_id=run_id,
        job_name=job_name,
        job_namespace="data-engineering",
        event_type=status,
        event_time=datetime.utcnow().isoformat(),
        inputs=[{"namespace": "data-lake", "name": t} for t in source_tables],
        outputs=[{"namespace": "data-lake", "name": target_table}],
    ))
    
    return run_id
```

### 3.3 — PII Detection & Masking

```python
# python/governance/pii_masking.py
import hashlib
import re
from typing import Callable
import pandas as pd

PII_PATTERNS = {
    "email": re.compile(r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}"),
    "cpf": re.compile(r"\d{3}\.\d{3}\.\d{3}-\d{2}"),
    "phone": re.compile(r"\(\d{2}\)\s?\d{4,5}-\d{4}"),
    "credit_card": re.compile(r"\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}"),
}

def detect_pii_columns(df: pd.DataFrame, sample_size: int = 100) -> dict:
    """Detecta colunas com PII via pattern matching."""
    pii_columns = {}
    sample = df.head(sample_size)
    
    for col in sample.select_dtypes(include=["object"]).columns:
        for pii_type, pattern in PII_PATTERNS.items():
            matches = sample[col].astype(str).apply(lambda x: bool(pattern.search(x)))
            if matches.sum() > sample_size * 0.1:  # >10% match
                pii_columns[col] = pii_type
                break
    
    return pii_columns

def pseudonymize(value: str, salt: str = "data-eng-2024") -> str:
    """Pseudonimização irreversível via HMAC-SHA256."""
    return hashlib.sha256(f"{salt}:{value}".encode()).hexdigest()[:16]

def mask_email(email: str) -> str:
    """Mascaramento de email: j***@e***.com"""
    parts = email.split("@")
    if len(parts) != 2:
        return "***@***.***"
    local = parts[0][0] + "***"
    domain_parts = parts[1].split(".")
    domain = domain_parts[0][0] + "***." + domain_parts[-1]
    return f"{local}@{domain}"

def apply_pii_policy(df: pd.DataFrame, policy: dict) -> pd.DataFrame:
    """Aplica política de PII: masking ou pseudonimização."""
    df = df.copy()
    for col, action in policy.items():
        if col not in df.columns:
            continue
        if action == "pseudonymize":
            df[col] = df[col].apply(lambda x: pseudonymize(str(x)) if pd.notna(x) else None)
        elif action == "mask":
            df[col] = df[col].apply(lambda x: mask_email(str(x)) if pd.notna(x) else None)
        elif action == "redact":
            df[col] = "***REDACTED***"
    return df

# Usage
PII_POLICY = {
    "email": "mask",
    "cpf": "pseudonymize",
    "phone": "redact",
    "name": "pseudonymize",
}
```

### 3.4 — Go: Access Control & Audit Log

```go
// go/cmd/audit-logger/main.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/dynamodb"
    "github.com/aws/aws-sdk-go-v2/service/dynamodb/types"
)

type AuditEvent struct {
    EventID     string    `json:"event_id"`
    Actor       string    `json:"actor"`       // user/service
    Action      string    `json:"action"`      // READ, WRITE, DELETE
    Resource    string    `json:"resource"`     // table/bucket/key
    Result      string    `json:"result"`      // ALLOWED, DENIED
    PIIAccessed bool      `json:"pii_accessed"`
    Timestamp   time.Time `json:"timestamp"`
    Details     string    `json:"details"`
}

func logAudit(ctx context.Context, client *dynamodb.Client, event AuditEvent) error {
    item := map[string]types.AttributeValue{
        "pipeline_id": &types.AttributeValueMemberS{Value: fmt.Sprintf("AUDIT#%s", event.EventID)},
        "actor":       &types.AttributeValueMemberS{Value: event.Actor},
        "action":      &types.AttributeValueMemberS{Value: event.Action},
        "resource":    &types.AttributeValueMemberS{Value: event.Resource},
        "result":      &types.AttributeValueMemberS{Value: event.Result},
        "pii":         &types.AttributeValueMemberBOOL{Value: event.PIIAccessed},
        "timestamp":   &types.AttributeValueMemberS{Value: event.Timestamp.Format(time.RFC3339)},
    }
    
    _, err := client.PutItem(ctx, &dynamodb.PutItemInput{
        TableName: aws.String("pipeline-state"),
        Item:      item,
    })
    return err
}
```

### 3.5 — Java: Governance Service

```java
// java/src/.../governance/GovernanceService.java
@Service
public class GovernanceService {

    private final GlueClient glue;
    private final DynamoDbClient dynamoDb;

    public CatalogEntry discoverTable(String database, String tableName) {
        var response = glue.getTable(GetTableRequest.builder()
            .databaseName(database)
            .name(tableName)
            .build());
        
        var table = response.table();
        return new CatalogEntry(
            table.name(),
            table.description(),
            table.storageDescriptor().location(),
            table.storageDescriptor().columns().stream()
                .map(c -> new ColumnInfo(c.name(), c.type(), c.comment()))
                .toList(),
            table.parameters()
        );
    }

    public LineageGraph getLineage(String tableName, int depth) {
        // Query lineage events from DynamoDB
        // Build directed graph of data flow
        var graph = new LineageGraph(tableName);
        
        // Traverse upstream (where did data come from?)
        graph.addUpstream(queryLineageEvents(tableName, "output"));
        
        // Traverse downstream (who consumes this data?)
        graph.addDownstream(queryLineageEvents(tableName, "input"));
        
        return graph;
    }
}
```

---

## Critérios de Aceite

- [ ] Glue Data Catalog: ≥ 3 databases + 5 tabelas registradas
- [ ] Data lineage rastreando ≥ 3 hops (source → ETL → curated → mart)
- [ ] PII detection identificando email, CPF, phone em datasets
- [ ] PII masking/pseudonimização aplicada antes de gravar no curated zone
- [ ] Audit log gravando acessos a dados sensíveis no DynamoDB
- [ ] Go audit logger funcional
- [ ] Java governance service com catalog discovery + lineage
- [ ] LGPD compliance: right to erasure implementado (delete by customer_id)
- [ ] ADR-021, ADR-022, ADR-023 completos
- [ ] 2 DrawIOs (governance architecture + lineage graph)

---

## Definição de Pronto (DoD)

- [ ] 3 ADRs (catalog + lineage + privacy)
- [ ] 2 DrawIOs (governance architecture + lineage graph)
- [ ] Glue Data Catalog configurado
- [ ] Data lineage rastreável
- [ ] PII detection + masking
- [ ] Audit logging
- [ ] LGPD right to erasure
- [ ] Commit: `feat(data-engineering-09): data governance`
