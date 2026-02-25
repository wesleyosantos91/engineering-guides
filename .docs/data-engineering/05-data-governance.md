# Data Engineering — Governança, Segurança & Qualidade de Dados na AWS

> **Objetivo deste documento:** Servir como referência completa sobre **governança, segurança, qualidade e observabilidade de dados na AWS**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Foco em **Lake Formation, Glue Data Catalog, DataZone, Macie, KMS, Data Quality, Observabilidade de pipelines**.

---

## Quick Reference — Cheat Sheet

| Pilar | Serviço AWS | Responsabilidade |
|-------|-------------|-----------------|
| **Catálogo** | Glue Data Catalog | Metadados, schemas, partitions, crawlers |
| **Acesso** | Lake Formation | Fine-grained access control (column/row/cell) |
| **Descobribilidade** | DataZone | Data marketplace, domínios, data products |
| **Classificação** | Macie | Detectar PII e dados sensíveis automaticamente |
| **Encryption** | KMS | Gerenciamento de chaves de criptografia |
| **Qualidade** | Glue Data Quality | Rules, validação, anomaly detection |
| **Lineage** | DataZone / Glue | Data lineage (origem → destino) |
| **Auditoria** | CloudTrail + Config | Quem acessou, quando, o quê mudou |
| **Compliance** | Config Rules + Macie | LGPD, GDPR, HIPAA compliance checks |
| **Observabilidade** | CloudWatch + EventBridge | Monitoramento de pipelines, alertas |

---

## Sumário

- [Data Engineering — Governança, Segurança \& Qualidade de Dados na AWS](#data-engineering--governança-segurança--qualidade-de-dados-na-aws)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [AWS Glue Data Catalog](#aws-glue-data-catalog)
  - [AWS Lake Formation](#aws-lake-formation)
  - [Amazon DataZone](#amazon-datazone)
  - [Segurança de Dados](#segurança-de-dados)
  - [Data Quality](#data-quality)
  - [Data Lineage](#data-lineage)
  - [Observabilidade de Pipelines](#observabilidade-de-pipelines)
  - [Compliance — LGPD, GDPR, HIPAA](#compliance--lgpd-gdpr-hipaa)
  - [Custo — FinOps para Data](#custo--finops-para-data)
  - [Governança Framework](#governança-framework)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## AWS Glue Data Catalog

### O que é

O **Glue Data Catalog** é o **metastore central** para todos os dados na AWS. É compatível com Apache Hive e funciona como catálogo unificado para Athena, Redshift Spectrum, EMR, Glue ETL e Lake Formation.

```
┌──────────────────────────────────────────────────────────┐
│                   GLUE DATA CATALOG                       │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ DATABASE │  │ DATABASE │  │ DATABASE │               │
│  │ raw_db   │  │curated_db│  │refined_db│               │
│  │          │  │          │  │          │               │
│  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │               │
│  │ │TABLE │ │  │ │TABLE │ │  │ │TABLE │ │               │
│  │ │orders│ │  │ │orders│ │  │ │fact_ │ │               │
│  │ │      │ │  │ │      │ │  │ │sales │ │               │
│  │ │Schema│ │  │ │Schema│ │  │ │      │ │               │
│  │ │Partns│ │  │ │Partns│ │  │ │Schema│ │               │
│  │ │Stats │ │  │ │Stats │ │  │ │Partns│ │               │
│  │ └──────┘ │  │ └──────┘ │  │ └──────┘ │               │
│  └──────────┘  └──────────┘  └──────────┘               │
│                                                           │
│  CRAWLERS: auto-discover schema from S3/RDS/DynamoDB     │
│  CLASSIFIERS: identify format (Parquet, JSON, CSV...)     │
│  CONNECTIONS: JDBC to RDS, Redshift, on-prem             │
│  SCHEMA REGISTRY: Avro/JSON schema versioning             │
│                                                           │
│  Consumers:                                               │
│  Athena | Redshift Spectrum | EMR | Glue ETL | Lake Form │
└──────────────────────────────────────────────────────────┘
```

### Boas Práticas do Data Catalog

| Prática | Detalhes |
|---------|---------|
| **Naming convention** | `{zone}_{domain}` → `raw_sales`, `curated_sales`, `refined_analytics` |
| **Crawlers schedulados** | Agende crawlers para atualizar partitions automaticamente |
| **Crawler per source** | Um crawler por source type — não misture schemas |
| **Table properties** | Adicione metadata: `classification`, `owner`, `pii`, `sla` |
| **Schema Registry** | Use para schemas de streaming (Avro/JSON) — versionamento |
| **Partition projection** | Use ao invés de MSCK REPAIR TABLE — mais rápido no Athena |
| **Resource policies** | Controle cross-account access ao catalog |
| **Encryption** | Ative encryption do metadata catalog (SSE-KMS) |

### Partition Projection (Athena)

```sql
-- Ao invés de: MSCK REPAIR TABLE (lento para milhares de partitions)
-- Use: Partition Projection (Athena calcula partitions sem listar S3)

CREATE EXTERNAL TABLE raw_db.clickstream (
    user_id STRING,
    event_type STRING,
    page_url STRING,
    event_timestamp TIMESTAMP
)
PARTITIONED BY (
    year STRING,
    month STRING,
    day STRING,
    hour STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
STORED AS PARQUET
LOCATION 's3://data-lake/raw/clickstream/'
TBLPROPERTIES (
    'projection.enabled' = 'true',
    'projection.year.type' = 'integer',
    'projection.year.range' = '2020,2030',
    'projection.month.type' = 'integer',
    'projection.month.range' = '1,12',
    'projection.month.digits' = '2',
    'projection.day.type' = 'integer',
    'projection.day.range' = '1,31',
    'projection.day.digits' = '2',
    'projection.hour.type' = 'integer',
    'projection.hour.range' = '0,23',
    'projection.hour.digits' = '2',
    'storage.location.template' =
        's3://data-lake/raw/clickstream/year=${year}/month=${month}/day=${day}/hour=${hour}/'
);
```

---

## AWS Lake Formation

### O que é

**Lake Formation** adiciona uma camada de **governança e controle de acesso** sobre o Data Lake, integrando com o Glue Data Catalog.

```
┌──────────────────────────────────────────────────────────┐
│                   LAKE FORMATION                          │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │              ACCESS CONTROL                       │    │
│  │                                                   │    │
│  │  Database-level:  GRANT SELECT ON db TO role     │    │
│  │  Table-level:     GRANT SELECT ON table TO user  │    │
│  │  Column-level:    GRANT SELECT(col1,col2) TO ... │    │
│  │  Row-level:       Filter rows by condition       │    │
│  │  Cell-level:      Column + Row combined          │    │
│  │  Tag-based:       GRANT based on LF-Tags        │    │
│  └──────────────────────────────────────────────────┘    │
│                                                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │              DATA FILTERS                         │    │
│  │                                                   │    │
│  │  Row filter:                                      │    │
│  │    "region = 'us-east-1'"                        │    │
│  │    → User só vê dados da sua região              │    │
│  │                                                   │    │
│  │  Column filter:                                   │    │
│  │    Include: [name, email]  Exclude: [ssn, salary]│    │
│  │    → User não vê colunas sensíveis               │    │
│  └──────────────────────────────────────────────────┘    │
│                                                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │              LF-TAGS (Tag-Based Access Control)   │    │
│  │                                                   │    │
│  │  Tag: classification = [public, internal, pii]   │    │
│  │  Tag: domain = [sales, marketing, finance]       │    │
│  │                                                   │    │
│  │  Grant: role/data-analyst                        │    │
│  │    → classification IN (public, internal)        │    │
│  │    → domain IN (sales, marketing)                │    │
│  │                                                   │    │
│  │  Resultado: auto-grant em qualquer tabela que    │    │
│  │  tenha essas tags — ESCALÁVEL!                   │    │
│  └──────────────────────────────────────────────────┘    │
│                                                           │
│  Integrations:                                            │
│  Athena | Redshift Spectrum | EMR | Glue ETL             │
└──────────────────────────────────────────────────────────┘
```

### Boas Práticas Lake Formation

| Prática | Detalhes |
|---------|---------|
| **LF-Tags** | Prefira tag-based access control sobre grants individuais — escala melhor |
| **Hybrid mode** | Migre gradualmente de IAM policies para Lake Formation |
| **Cross-account** | Use Lake Formation para compartilhar tabelas cross-account (vs S3 bucket policies) |
| **Data Filters** | Row/Column filters para multi-tenant access — cada time vê seus dados |
| **Governed Tables** | Use para ACID transactions no data lake |
| **Audit** | CloudTrail logs todas as operações de acesso via Lake Formation |
| **Principle of least privilege** | Comece negando tudo, grant por tag incremental |
| **Data locations** | Registre S3 locations no Lake Formation para governança |

---

## Amazon DataZone

### O que é

**Amazon DataZone** é o serviço de **data marketplace** — permite publicar, descobrir e consumir data products com governança.

```
┌──────────────────────────────────────────────────────────────┐
│                      AMAZON DATAZONE                          │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────┐                                         │
│  │    DOMAINS       │  Organizacional (Sales, Marketing, etc) │
│  │   ┌───────────┐ │                                         │
│  │   │  Projects  │ │  Workspaces para times                 │
│  │   │  ┌──────┐  │ │                                         │
│  │   │  │Assets│  │ │  Tabelas, dashboards, ML models        │
│  │   │  └──────┘  │ │                                         │
│  │   └───────────┘ │                                         │
│  └─────────────────┘                                         │
│                                                               │
│  WORKFLOW:                                                    │
│  1. Producer publica um Data Product (table/view)            │
│     → Metadata, description, schema, SLA, tags              │
│  2. Consumer busca no DataZone Catalog                       │
│  3. Consumer solicita acesso (subscription request)          │
│  4. Domain owner aprova (manual ou auto-approve)             │
│  5. Lake Formation grants são aplicados automaticamente      │
│  6. Consumer acessa via Athena/Redshift do seu project       │
│                                                               │
│  INTEGRATIONS:                                                │
│  - Glue Data Catalog (metadata)                              │
│  - Lake Formation (access control)                            │
│  - Redshift (data warehouse assets)                          │
│  - SageMaker (ML model assets)                               │
│  - Custom sources via Glue connections                       │
└──────────────────────────────────────────────────────────────┘
```

### Quando usar DataZone

| Cenário | DataZone? | Alternativa |
|---------|-----------|-------------|
| > 5 times consumindo dados | ✅ | — |
| Data Mesh / domain ownership | ✅ | — |
| Data marketplace interno | ✅ | — |
| Time único usando data lake | ❌ Over-engineering | Lake Formation + Catalog |
| Startup < 20 pessoas | ❌ | Glue Data Catalog apenas |

---

## Segurança de Dados

### Camadas de segurança

```
┌──────────────────────────────────────────────────────────┐
│                 DATA SECURITY LAYERS                      │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  Layer 1: NETWORK                                        │
│  ┌────────────────────────────────────────────┐          │
│  │ VPC, Security Groups, NACLs, PrivateLink   │          │
│  │ VPC Endpoints para S3, Glue, Kinesis       │          │
│  └────────────────────────────────────────────┘          │
│                                                           │
│  Layer 2: IDENTITY & ACCESS                              │
│  ┌────────────────────────────────────────────┐          │
│  │ IAM Roles (not users), Lake Formation RBAC │          │
│  │ Service Control Policies (SCPs)             │          │
│  │ Resource-based policies (S3, KMS)           │          │
│  └────────────────────────────────────────────┘          │
│                                                           │
│  Layer 3: ENCRYPTION                                     │
│  ┌────────────────────────────────────────────┐          │
│  │ At rest: SSE-S3, SSE-KMS, CSE             │          │
│  │ In transit: TLS 1.2+                       │          │
│  │ Key management: KMS CMK, key rotation      │          │
│  └────────────────────────────────────────────┘          │
│                                                           │
│  Layer 4: DATA PROTECTION                                │
│  ┌────────────────────────────────────────────┐          │
│  │ Macie (PII detection), Column masking      │          │
│  │ Row-level security (Lake Formation)         │          │
│  │ Tokenization, anonymization                 │          │
│  └────────────────────────────────────────────┘          │
│                                                           │
│  Layer 5: AUDIT & COMPLIANCE                             │
│  ┌────────────────────────────────────────────┐          │
│  │ CloudTrail (API audit), S3 Access Logs     │          │
│  │ Config Rules, Macie findings               │          │
│  │ VPC Flow Logs                               │          │
│  └────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────┘
```

### Amazon Macie — Detecção de PII

```
Macie Workflow:
═══════════════

1. Enable Macie na conta/Organization
2. Configure discovery jobs para S3 buckets
3. Macie classifica automaticamente:
   - PII: CPF, email, telefone, endereço, cartão de crédito
   - Financial: conta bancária, dados fiscais
   - Health: dados médicos (PHI)
   - Credentials: access keys, passwords, tokens
4. Findings → EventBridge → Lambda → remediate

Remediation automática:
  Macie finding (PII detected in raw zone)
      │
      ▼
  EventBridge rule
      │
      ▼
  Lambda:
    1. Tag S3 object como "contains-pii"
    2. Move para bucket segregado com Lake Formation controls
    3. Notify data owner via SNS
    4. Create Jira ticket for review
```

### Encryption Strategy

| Dados | Encryption | Serviço | Key Management |
|-------|-----------|---------|---------------|
| S3 (data lake) | SSE-KMS | KMS | CMK por domínio |
| Redshift | AES-256 | KMS | Cluster-level key |
| DynamoDB | AES-256 | KMS | AWS-owned ou CMK |
| Aurora | AES-256 | KMS | Cluster-level key |
| Kinesis | SSE | KMS | Stream-level key |
| Glue jobs | SSE | KMS | Job-level key |
| In-transit | TLS 1.2+ | — | Certificate Manager |

### KMS Best Practices para Data

| Prática | Detalhes |
|---------|---------|
| **CMK por domínio** | Cada domínio de dados tem sua CMK — separação de controle |
| **Key rotation** | Ative auto-rotation (anual) para CMKs |
| **Key policies** | Principle of least privilege — somente roles necessários |
| **Audit** | CloudTrail loga todo uso de KMS key |
| **Cross-account** | Grants para compartilhar keys cross-account |
| **Alias** | Use aliases descritivos: `alias/data-lake-sales-key` |

---

## Data Quality

### Framework de Data Quality

```
┌──────────────────────────────────────────────────────────┐
│              DATA QUALITY DIMENSIONS                      │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐         │
│  │COMPLETENESS│  │ ACCURACY   │  │ CONSISTENCY │         │
│  │            │  │            │  │             │         │
│  │ Non-null   │  │ Correct    │  │ Same value  │         │
│  │ values?    │  │ values?    │  │ across      │         │
│  │            │  │            │  │ systems?    │         │
│  └────────────┘  └────────────┘  └────────────┘         │
│                                                           │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐         │
│  │ TIMELINESS │  │ UNIQUENESS │  │ VALIDITY    │         │
│  │            │  │            │  │             │         │
│  │ Dados      │  │ Sem        │  │ Formato     │         │
│  │ chegam no  │  │ duplicatas?│  │ correto?    │         │
│  │ SLA?       │  │            │  │ Enum válido?│         │
│  └────────────┘  └────────────┘  └────────────┘         │
└──────────────────────────────────────────────────────────┘
```

### Glue Data Quality — Rules

```python
# Regras de Data Quality por zona

RAW_ZONE_RULES = """
Rules = [
    # Structural checks
    ColumnExists "order_id",
    ColumnExists "customer_id",
    ColumnExists "created_at",
    
    # Volume checks (anomaly detection)
    RowCount between 1000 and 10000000,
    
    # Basic completeness
    Completeness "order_id" = 1.0,
    IsComplete "created_at"
]
"""

CURATED_ZONE_RULES = """
Rules = [
    # All raw rules +
    
    # Completeness
    Completeness "order_id" = 1.0,
    Completeness "customer_id" > 0.99,
    Completeness "total" > 0.99,
    
    # Uniqueness
    Uniqueness "order_id" = 1.0,
    IsPrimaryKey "order_id",
    
    # Validity
    ColumnValues "status" in ["PENDING","COMPLETED","SHIPPED","CANCELLED"],
    ColumnValues "total" > 0,
    ColumnValues "total" < 1000000,
    ColumnValues "created_at" between "2020-01-01" and "2030-01-01",
    
    # Referential integrity
    ReferentialIntegrity "customer_id" "customers_db.customers.id" > 0.99,
    
    # Freshness
    Freshness "created_at" <= 2 days,
    
    # Statistical
    StandardDeviation "total" between 10 and 500,
    Mean "total" between 50 and 300
]
"""

REFINED_ZONE_RULES = """
Rules = [
    # All curated rules +
    
    # Cross-dataset consistency
    DatasetMatch "curated_db.orders" "refined_db.fact_sales" 
        "order_id" > 0.999,
    
    # Aggregation checks
    CustomSql "SELECT ABS(SUM(total) - 
        (SELECT SUM(amount) FROM raw_db.payments)) 
        FROM primary" < 100
]
"""
```

### Data Quality Pipeline Pattern

```
S3 (incoming data)
     │
     ▼
┌──────────────┐
│  STAGE 1:    │  Structural validation
│  Schema      │  (columns exist, types match)
│  Validation  │
└──────┬───────┘
       │ PASS?
       ├── NO → Quarantine + Alert
       │
       ▼
┌──────────────┐
│  STAGE 2:    │  Completeness, validity, ranges
│  Data Quality│  (Glue Data Quality Rules)
│  Rules       │
└──────┬───────┘
       │ PASS?
       ├── NO → Quarantine bad records
       │        Promote good records
       │        Alert on failure rate
       │
       ▼
┌──────────────┐
│  STAGE 3:    │  SLA met, anomaly detection
│  Business    │  (volume, freshness, statistical)
│  Validation  │
└──────┬───────┘
       │ PASS?
       ├── NO → Alert (may still promote with warning)
       │
       ▼
  Promote to Curated Zone ✅
  Update Catalog
  Publish CloudWatch Metrics

Metrics to track:
  - data_quality_pass_rate (per rule)
  - data_quality_records_quarantined
  - data_quality_freshness_seconds
  - data_quality_volume_anomaly
```

---

## Data Lineage

### O que é e por que importa

**Data Lineage** rastreia o **caminho dos dados** — de onde vieram, quais transformações sofreram, e para onde foram.

```
LINEAGE EXAMPLE:
════════════════

Source:                  Transform:               Destination:
Aurora.public.orders     Glue Job: orders-etl     curated_db.orders
  │                        │                         │
  │ columns:               │ operations:             │ columns:
  │  - id                  │  - dedup by id          │  - order_id (renamed)
  │  - cust_id             │  - filter status        │  - customer_id (renamed)
  │  - total_value         │  - rename columns       │  - total (renamed)
  │  - status              │  - cast types            │  - status (filtered)
  │  - created             │  - join customers        │  - customer_name (joined)
  │                        │  - partition by date     │  - created_at (cast)
  │                        │                         │
  └────────────────────────┴─────────────────────────┘
                    DataZone / Glue lineage tracks this
```

### AWS Services para Lineage

| Serviço | Tipo de lineage | Detalhes |
|---------|----------------|---------|
| **DataZone** | Business lineage | Asset → Asset relationships |
| **Glue** | Technical lineage | Job → Table → Column transformations |
| **CloudTrail** | Access lineage | Quem acessou qual dado quando |
| **Athena Query History** | Query lineage | Quais tabelas foram consultadas |

---

## Observabilidade de Pipelines

### O que monitorar

```
┌──────────────────────────────────────────────────────────┐
│           PIPELINE OBSERVABILITY                          │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  METRICS (CloudWatch)                                    │
│  ┌────────────────────────────────────────────┐          │
│  │ • Pipeline duration (p50, p95, p99)        │          │
│  │ • Records processed (input vs output)      │          │
│  │ • Data quality pass rate                   │          │
│  │ • Pipeline failure rate                    │          │
│  │ • Data freshness (lag)                     │          │
│  │ • Cost per run                             │          │
│  │ • Glue DPU utilization                     │          │
│  │ • Kinesis IteratorAge                      │          │
│  │ • Redshift query queue time                │          │
│  └────────────────────────────────────────────┘          │
│                                                           │
│  LOGS (CloudWatch Logs)                                  │
│  ┌────────────────────────────────────────────┐          │
│  │ • Glue job logs (Spark driver/executor)    │          │
│  │ • Lambda execution logs                    │          │
│  │ • Step Functions execution history         │          │
│  │ • DMS task logs                            │          │
│  │ • EMR step logs                            │          │
│  └────────────────────────────────────────────┘          │
│                                                           │
│  ALERTS (CloudWatch Alarms + SNS + EventBridge)          │
│  ┌────────────────────────────────────────────┐          │
│  │ • Pipeline failed → PagerDuty/Slack        │          │
│  │ • SLA breach (data not fresh) → escalate   │          │
│  │ • Quality < threshold → quarantine + alert │          │
│  │ • Cost anomaly → team notification         │          │
│  │ • Kinesis lag > 5min → auto-scale alert    │          │
│  └────────────────────────────────────────────┘          │
│                                                           │
│  DASHBOARDS (CloudWatch Dashboard / QuickSight)          │
│  ┌────────────────────────────────────────────┐          │
│  │ • Pipeline health overview                 │          │
│  │ • Data freshness per domain                │          │
│  │ • Quality trends over time                 │          │
│  │ • Cost per pipeline / domain               │          │
│  │ • SLA compliance                           │          │
│  └────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────┘
```

### Alertas essenciais

| Alerta | Métrica | Threshold sugerido | Ação |
|--------|---------|-------------------|------|
| **Pipeline Failed** | Step Functions FAILED | Any failure | Investigate + retry |
| **Data not fresh** | Custom metric: last_update | > SLA (ex: 2h) | Escalate |
| **Quality degraded** | data_quality_pass_rate | < 95% | Quarantine + investigate |
| **Kinesis lag** | IteratorAgeMilliseconds | > 300000 (5min) | Scale consumers |
| **Glue job slow** | Glue job duration | > 2x normal | Check data volume + DPU |
| **Cost spike** | Estimated charges | > 120% of baseline | Review + optimize |
| **DMS lag** | CDCLatencySource | > 300 (5min) | Check replication instance size |

---

## Compliance — LGPD, GDPR, HIPAA

### Checklist por regulação

| Requisito | LGPD | GDPR | HIPAA | Serviço AWS |
|-----------|------|------|-------|-------------|
| **Identificar PII/PHI** | ✅ | ✅ | ✅ | Macie |
| **Consentimento / Base legal** | ✅ | ✅ | ✅ | Application-level |
| **Direito ao esquecimento** | ✅ | ✅ | — | Iceberg DELETE, DynamoDB TTL |
| **Portabilidade de dados** | ✅ | ✅ | — | Athena EXPORT, S3 download |
| **Encryption at rest** | ✅ | ✅ | ✅ | KMS (CMK) |
| **Encryption in transit** | ✅ | ✅ | ✅ | TLS 1.2+ |
| **Access control** | ✅ | ✅ | ✅ | Lake Formation, IAM |
| **Audit trail** | ✅ | ✅ | ✅ | CloudTrail, S3 Access Logs |
| **Data minimization** | ✅ | ✅ | ✅ | Column masking, Lake Formation |
| **Breach notification** | ✅ | ✅ | ✅ | GuardDuty, Macie → SNS |
| **Data residency** | ✅ | ✅ | — | S3 region-specific buckets |
| **Retention policies** | ✅ | ✅ | ✅ | S3 Lifecycle, Object Lock |

### Direito ao Esquecimento (LGPD/GDPR) — Implementação

```
Request: "Delete all my data" (user_id = 12345)
═══════════════════════════════════════════════

1. Receive request via API / support ticket

2. Identify all data locations (using Data Catalog + Lineage):
   - Aurora: users table (row delete)
   - DynamoDB: user_sessions (TTL or delete)
   - S3 Raw: raw/customers/... (Iceberg DELETE)
   - S3 Curated: curated/customers/... (Iceberg DELETE)
   - Redshift: dim_customer (SQL DELETE)
   - OpenSearch: user_index (document delete)
   - ElastiCache: session cache (invalidate)
   - Backups: flag for exclusion on next retention cycle

3. Execute deletion pipeline (Step Functions):
   Step 1: Lambda → Aurora DELETE WHERE user_id = 12345
   Step 2: Lambda → DynamoDB DeleteItem
   Step 3: Athena → DELETE FROM iceberg_table WHERE customer_id = 12345
   Step 4: Lambda → Redshift DELETE
   Step 5: Lambda → OpenSearch delete by query
   Step 6: Lambda → ElastiCache invalidate
   Step 7: Log deletion in QLDB (immutable audit)
   Step 8: Notify requester (SNS → email)

4. Verify deletion (audit):
   - Query all systems for user_id = 12345
   - Confirm zero results
   - Record in compliance log
```

---

## Custo — FinOps para Data

### Estratégias de otimização

| Serviço | Otimização | Economia estimada |
|---------|-----------|-------------------|
| **S3** | Lifecycle → IA/Glacier | 40-90% em dados antigos |
| **S3** | Intelligent-Tiering | 20-40% automático |
| **Athena** | Parquet + partitioning | 90% por query |
| **Glue** | Flex execution | 34% em jobs não-urgentes |
| **Glue** | Auto-scaling DPU | 20-50% evitando over-provisioning |
| **EMR** | Spot instances (workers) | 60-90% em compute |
| **Redshift** | Serverless para variável | 30-60% vs provisioned ocioso |
| **Redshift** | Reserved instances | 30-75% para steady-state |
| **Kinesis** | On-Demand mode | Evita over-provisioning |
| **DynamoDB** | On-Demand para variável | Evita over-provisioning |

### Cost allocation

| Tag | Exemplo | Por quê |
|-----|---------|---------|
| `domain` | sales, marketing, finance | Chargeback por domínio |
| `environment` | prod, staging, dev | Separar custos |
| `pipeline` | orders-daily, clickstream | Custo por pipeline |
| `team` | data-engineering, analytics | Custo por time |
| `classification` | public, internal, pii | Compliance tracking |

---

## Governança Framework

### Modelo de maturidade em governança

| Nível | Descrição | Ferramentas |
|-------|-----------|-------------|
| **1 — Ad-hoc** | Sem catálogo, sem controle de acesso, sem qualidade | — |
| **2 — Básico** | Glue Data Catalog, IAM policies, S3 encryption | Catalog + IAM + KMS |
| **3 — Gerenciado** | Lake Formation RBAC, Data Quality rules, Macie | + Lake Formation + Macie |
| **4 — Governado** | LF-Tags, DataZone, lineage, compliance automation | + DataZone + Automation |
| **5 — Data Mesh** | Domain ownership, data products, federated governance | Full DataZone + Org-level |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código de governança, segurança e qualidade de dados na AWS, verifique:

1. **S3 bucket sem Block Public Access** — TODOS os buckets de dados devem ter Block Public Access
2. **Sem encryption at rest** — S3, Redshift, DynamoDB, Aurora devem ter encryption ativado
3. **IAM user com access key para data access** — Use IAM Roles, nunca access keys
4. **Lake Formation sem LF-Tags** — Para mais de 10 tabelas, prefira tag-based access control
5. **Sem Glue Data Catalog para datasets** — Todo dataset em S3 deve estar catalogado
6. **Sem Data Quality rules** — Cada pipeline deve ter quality checks entre estágios
7. **PII em clear text** — Use Macie para detectar; mascarar ou criptografar PII
8. **Sem CloudTrail** — Audit trail deve estar ativo para todas as contas
9. **Sem lifecycle policy** — Dados não acessados devem ter lifecycle para reduzir custo
10. **Sem tags obrigatórias** — Exija tags: domain, classification, owner, environment
11. **Sem monitoramento de pipeline** — Toda pipeline deve ter alertas de falha e SLA
12. **Sem plano de retenção** — Defina retention policy baseada em compliance requirements
13. **Glue Crawler sem schedule** — Crawlers devem rodar após cada ingestão para atualizar catalog
14. **Cross-account sem Lake Formation** — Compartilhamento de dados cross-account deve usar LF

---

## Referências

- **AWS Lake Formation Developer Guide** — Governança e access control
- **AWS Glue Data Quality** — Rules e validação
- **Amazon DataZone User Guide** — Data marketplace
- **Amazon Macie User Guide** — PII detection
- **LGPD (Lei 13.709/2018)** — Legislação brasileira de proteção de dados
- **GDPR** — Regulamento europeu de proteção de dados
- **AWS Well-Architected — Security Pillar** — Best practices de segurança
- **Data Governance: How to Design, Deploy, and Sustain an Effective Data Governance Program** — John Ladley
- **DMBOK (Data Management Body of Knowledge)** — DAMA International
