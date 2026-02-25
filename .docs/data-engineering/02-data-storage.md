# Data Engineering — Storage & Databases na AWS

> **Objetivo deste documento:** Servir como referência completa sobre **serviços de armazenamento e bancos de dados para dados na AWS**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Foco em **quando usar cada serviço, configurações otimizadas, anti-patterns e heurísticas de decisão**.

---

## Quick Reference — Cheat Sheet

| Serviço | Tipo | Latência | Escala | Quando usar |
|---------|------|----------|--------|-------------|
| **S3** | Object Store | ms | Ilimitada | Data lake, raw data, backups, arquivos |
| **Redshift** | Columnar DW | s | PB | Analytics SQL, BI, reporting |
| **DynamoDB** | Key-Value / Document | ms (single-digit) | Ilimitada | Operacional, alta throughput, serverless |
| **Aurora** | Relational (MySQL/PG) | ms | 128TB | OLTP, transacional, compatibilidade SQL |
| **RDS** | Relational (multi-engine) | ms | 64TB | OLTP simples, lift-and-shift |
| **ElastiCache** | In-Memory (Redis/Memcached) | sub-ms | TBs | Cache, session store, leaderboards |
| **MemoryDB** | In-Memory (Redis-compatible) | sub-ms | TBs | Primary database in-memory com durability |
| **Neptune** | Graph | ms | 64TB | Grafos, redes sociais, fraud, knowledge graph |
| **Timestream** | Time Series | ms | Ilimitada | IoT, métricas, DevOps, series temporais |
| **OpenSearch** | Search / Analytics | ms | PB | Full-text search, log analytics, SIEM |
| **DocumentDB** | Document (MongoDB-compat) | ms | 128TB | Workloads MongoDB na AWS |
| **Keyspaces** | Wide Column (Cassandra-compat) | ms | Ilimitada | Workloads Cassandra na AWS |
| **QLDB** | Ledger | ms | — | Audit trail imutável, compliance |

---

## Sumário

- [Data Engineering — Storage \& Databases na AWS](#data-engineering--storage--databases-na-aws)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Árvore de Decisão](#árvore-de-decisão)
  - [Amazon S3 — Object Storage](#amazon-s3--object-storage)
  - [Amazon Redshift — Data Warehouse](#amazon-redshift--data-warehouse)
  - [Amazon DynamoDB — NoSQL](#amazon-dynamodb--nosql)
  - [Amazon Aurora — Relacional](#amazon-aurora--relacional)
  - [Amazon ElastiCache e MemoryDB](#amazon-elasticache-e-memorydb)
  - [Amazon Neptune — Grafos](#amazon-neptune--grafos)
  - [Amazon Timestream — Séries Temporais](#amazon-timestream--séries-temporais)
  - [Amazon OpenSearch — Search e Analytics](#amazon-opensearch--search-e-analytics)
  - [Estratégia Multi-Database](#estratégia-multi-database)
  - [Formatos de dados no S3](#formatos-de-dados-no-s3)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Árvore de Decisão

```
                    Qual é o padrão de acesso principal?
                              │
              ┌───────────────┼───────────────────┐
              │               │                   │
         Analítico      Operacional          Busca/Search
              │               │                   │
    ┌─────────┤         ┌─────┤                   │
    │         │         │     │              OpenSearch
    │         │         │     │
  SQL?    Time-series?  │   Grafo?
    │         │         │     │
   Sim       Sim       Não   Sim
    │         │         │     │
    │    Timestream     │   Neptune
    │                   │
    │          ┌────────┤
    │          │        │
    │     Key-Value  Relacional
    │          │        │
    │     DynamoDB   ┌──┤
    │                │  │
    │            Alta    Standard
    │            escala
    │                │  │
    │           Aurora  RDS
    │
    ├─── Dados no S3? ──── Sim ──── Athena (serverless)
    │
    ├─── Volume < 100GB? ── Sim ── Redshift Serverless
    │
    └─── Volume > 100GB? ── Sim ── Redshift Provisioned / RA3
```

---

## Amazon S3 — Object Storage

### Fundamentos

O **Amazon S3** é o serviço fundamental para data engineering na AWS. Todo data lake é construído sobre S3.

### Storage Classes

| Classe | Acesso | Latência | Custo/GB | Caso de uso |
|--------|--------|----------|----------|-------------|
| **S3 Standard** | Frequente | ms | $$$ | Raw zone, dados ativos |
| **S3 Standard-IA** | Infrequente (1x/mês) | ms | $$ | Curated zone > 30 dias |
| **S3 One Zone-IA** | Infrequente, não-crítico | ms | $ | Dados reproduzíveis |
| **S3 Intelligent-Tiering** | Imprevisível | ms | $$* | Quando não sabe o padrão de acesso |
| **S3 Glacier Instant Retrieval** | Raro (1x/trimestre) | ms | $ | Archive com acesso rápido |
| **S3 Glacier Flexible Retrieval** | Raro (1-2x/ano) | min-hrs | ¢ | Backups, compliance 7 anos |
| **S3 Glacier Deep Archive** | Quase nunca | 12-48hrs | ¢¢ | Compliance 10+ anos |

### Lifecycle Strategy para Data Lake

```
Raw Zone:
  Day 0-90   → S3 Standard
  Day 90-365 → S3 Standard-IA
  Day 365+   → Glacier Flexible Retrieval

Curated Zone:
  Day 0-180  → S3 Standard
  Day 180+   → S3 Standard-IA

Refined Zone:
  Day 0-90   → S3 Standard (queries ativas)
  Day 90-365 → S3 Standard-IA
  Day 365+   → Glacier Instant Retrieval

Sandbox:
  Day 0-30   → S3 Standard
  Day 30+    → AUTO-DELETE (dados exploratórios temporários)
```

### Configurações obrigatórias para Data Engineering

| Configuração | Valor | Por quê |
|-------------|-------|---------|
| **Block Public Access** | ALL ON | Dados nunca devem ser públicos |
| **Default Encryption** | SSE-S3 ou SSE-KMS | Encryption at rest obrigatório |
| **Versioning** | ON (raw zone) | Recovery de ingestão incorreta |
| **Object Lock** | COMPLIANCE (se regulatório) | WORM — write-once-read-many |
| **Access Logging** | ON → bucket separado | Audit trail de acessos |
| **EventBridge integration** | ON | Trigger automático de pipelines |
| **Transfer Acceleration** | ON (se multi-region ingest) | Upload acelerado via CloudFront edge |
| **Inventory** | ON (weekly) | Auditoria de objetos, custos, classes |
| **Replication** | CRR (se DR multi-region) | Disaster recovery |

### S3 Performance Optimization

| Técnica | Quando | Como |
|---------|--------|------|
| **Multipart upload** | Arquivos > 100MB | SDK faz automaticamente; configure threshold |
| **Prefixos distribuídos** | > 5.500 GET/s ou 3.500 PUT/s | Distribuir entre múltiplos prefixos |
| **S3 Select** | Precisa de subset de CSV/JSON | `SELECT s.name FROM S3Object s WHERE s.age > 30` |
| **Byte-range fetches** | Parquet — ler só colunas necessárias | Combina com Athena/Redshift Spectrum |
| **S3 Express One Zone** | Ultra-low latency (single-digit ms) | Para cache de dados frequentes |

---

## Amazon Redshift — Data Warehouse

### Quando usar Redshift

| Cenário | Redshift? | Alternativa |
|---------|-----------|-------------|
| Analytics SQL sobre dados estruturados > 100GB | ✅ | — |
| BI dashboards com queries complexas | ✅ | — |
| Queries ad-hoc sobre data lake (S3) | ✅ (Spectrum) | Athena |
| OLTP (transações curtas, alta concorrência) | ❌ | Aurora/DynamoDB |
| Dados < 10GB | ❌ Over-engineering | Athena ou Aurora |
| Workload imprevisível / esporádico | Serverless ✅ | Athena |

### Redshift Provisioned vs Serverless

| Aspecto | Provisioned | Serverless |
|---------|-------------|------------|
| **Modelo** | Clusters fixos (RA3, DC2, DS2) | Auto-scaling RPU |
| **Custo** | Por hora de nó | Por RPU-hora consumido |
| **Gerenciamento** | Manual (resize, vacuum) | Zero-management |
| **Performance** | Previsível, tunável | Boa, mas variável |
| **Concurrency** | Configurable WLM | Automática |
| **Data Sharing** | ✅ | ✅ |
| **Spectrum** | ✅ | ✅ |
| **Melhor para** | Workloads constantes, alta performance | Workloads variáveis, times pequenos |

### Boas Práticas de Design

```
-- Distribution Styles
-- KEY: joins frequentes entre tabelas grandes
CREATE TABLE fact_sales (
    sale_id BIGINT,
    customer_id BIGINT,
    product_id BIGINT,
    sale_date DATE,
    amount DECIMAL(12,2)
)
DISTSTYLE KEY
DISTKEY (customer_id)        -- Join frequente com dim_customer
SORTKEY (sale_date);          -- Queries frequentes filtram por data

-- ALL: tabelas pequenas de dimensão (< 5M rows)
CREATE TABLE dim_region (
    region_id INT,
    region_name VARCHAR(100),
    country VARCHAR(50)
)
DISTSTYLE ALL;                -- Copia em cada nó — joins locais

-- EVEN: tabelas sem join pattern claro
CREATE TABLE staging_events (
    event_id BIGINT,
    payload VARCHAR(65535)
)
DISTSTYLE EVEN;               -- Distribuição uniforme
```

### Otimizações de carga

| Técnica | Impacto | Como |
|---------|---------|------|
| **COPY do S3** | 10-100x mais rápido que INSERT | `COPY table FROM 's3://bucket/prefix/' IAM_ROLE 'arn:...'` |
| **Manifest file** | Controle exato dos arquivos a carregar | JSON list de S3 paths |
| **Múltiplos arquivos** | Paralelismo = nº de slices | Dividir em N arquivos onde N = múltiplo de slices |
| **Compressão** | Reduz I/O e storage | GZIP, LZO, ZSTD, BZIP2 |
| **ANALYZE após COPY** | Atualiza estatísticas | `ANALYZE table;` |
| **VACUUM após DELETE** | Recupera espaço e re-sort | `VACUUM FULL table;` |

---

## Amazon DynamoDB — NoSQL

### Quando usar DynamoDB

| Cenário | DynamoDB? | Alternativa |
|---------|-----------|-------------|
| Key-value com latência < 10ms | ✅ | — |
| Alta throughput previsível | ✅ | — |
| Serverless / pay-per-request | ✅ | — |
| Queries analíticas complexas (JOINs) | ❌ | Redshift/Athena |
| Full-text search | ❌ | OpenSearch |
| Transações ACID cross-table | ❌* | Aurora (*DynamoDB suporta transactions limitadas) |
| Time series | ❌ | Timestream |

### Modelagem para Data Engineering

```
Single Table Design — Padrão recomendado para DynamoDB
═══════════════════════════════════════════════════════

Partition Key (PK)         Sort Key (SK)            Attributes
──────────────────         ──────────────           ──────────
CUSTOMER#C001              PROFILE                  name, email, tier
CUSTOMER#C001              ORDER#2025-001           total, status, date
CUSTOMER#C001              ORDER#2025-002           total, status, date
CUSTOMER#C001              ORDER#2025-002#ITEM#1    product, qty, price
PRODUCT#P001               METADATA                 name, category, price
PRODUCT#P001               REVIEW#R001              rating, comment

GSI1-PK                    GSI1-SK
───────                    ───────
ORDER#2025-001             CUSTOMER#C001            (acesso por order ID)
STATUS#PENDING             2025-01-15               (orders por status + data)
```

### DynamoDB Streams para Data Engineering

```
DynamoDB ──▶ DynamoDB Streams ──▶ Lambda ──▶ Kinesis Firehose ──▶ S3 (Data Lake)
                                     │
                                     ├──▶ OpenSearch (search index)
                                     │
                                     └──▶ EventBridge (event-driven)

Alternative:
DynamoDB ──▶ Export to S3 ──▶ Glue ──▶ Athena/Redshift
(Full export, zero RCU impact, PITR-based)
```

### Boas Práticas

| Prática | Detalhes |
|---------|---------|
| **On-Demand para imprevisível** | Pay-per-request, sem capacity planning |
| **Provisioned + Auto Scaling** | Para workloads previsíveis — mais barato |
| **DynamoDB Export to S3** | Export full table sem consumir RCU — ideal para analytics |
| **DynamoDB Streams** | CDC nativo — propaga mudanças para lake/search |
| **TTL** | Expire dados automaticamente — reduz custos |
| **DAX (Accelerator)** | Cache in-memory para reads repetitivas — microseconds |
| **Global Tables** | Multi-region active-active para DR |
| **Point-in-Time Recovery** | Ative sempre — recovery até 35 dias |
| **Encryption** | Default com AWS-owned key; use CMK para compliance |

---

## Amazon Aurora — Relacional

### Aurora vs RDS

| Aspecto | Aurora | RDS |
|---------|--------|-----|
| **Performance** | 5x MySQL, 3x PostgreSQL | Standard |
| **Storage** | Auto-scaling até 128TB | Manual, até 64TB |
| **Replication** | 6 cópias em 3 AZs | Multi-AZ (1 standby) |
| **Read Replicas** | Até 15, failover automático | Até 5, manual |
| **Serverless** | Aurora Serverless v2 | ❌ |
| **Global Database** | Cross-region < 1s lag | ❌ |
| **Custo** | ~20% mais que RDS | Baseline |
| **Quando usar** | Produção crítica, alta disponibilidade | Dev/test, workloads simples |

### Aurora para Data Engineering

```
Pattern: Aurora como fonte de dados para o Data Lake

Aurora (OLTP) ──▶ DMS (CDC) ──▶ Kinesis Data Streams ──▶ S3 (Raw Zone)
                                                              │
                                                         Glue ETL
                                                              │
                                                         S3 (Curated)
                                                              │
                                                    Athena / Redshift

Alternative (Zero-ETL):
Aurora ──▶ Zero-ETL Integration ──▶ Redshift (near real-time)
```

### Zero-ETL Integration

O **Zero-ETL** da Aurora elimina a necessidade de pipeline ETL para Redshift:

| Aspecto | Detalhes |
|---------|---------|
| **Como funciona** | Aurora replica automaticamente para Redshift |
| **Latência** | Near real-time (segundos) |
| **Impacto no Aurora** | Mínimo — usa WAL logs |
| **Custo** | Sem charge adicional além dos serviços |
| **Quando usar** | Analytics sobre dados transacionais sem pipeline complexo |
| **Limitação** | Aurora MySQL 3.05+ ou PostgreSQL 16.4+ |

---

## Amazon ElastiCache e MemoryDB

### Quando usar cada um

| Aspecto | ElastiCache (Redis) | ElastiCache (Memcached) | MemoryDB |
|---------|-------------------|------------------------|----------|
| **Persistência** | Opcional (AOF/RDB) | ❌ | ✅ Durável (multi-AZ WAL) |
| **Data structures** | Rich (sorted sets, streams) | Simple (key-value) | Rich (Redis-compatible) |
| **Use case** | Cache, sessions, real-time | Simple caching | Primary database in-memory |
| **Replication** | Read replicas | ❌ | Multi-AZ replicas |
| **Transactions** | ✅ | ❌ | ✅ |
| **Para DE** | Cache de lookups, feature store | Cache de resultados | Real-time aggregations |

### Patterns para Data Engineering

```
Pattern 1: Cache-Aside para lookups frequentes em pipelines
═══════════════════════════════════════════════════════════

Pipeline Step:
  1. Check ElastiCache for enrichment data
  2. If MISS → query Aurora/DynamoDB → populate cache (TTL 1h)
  3. If HIT  → use cached data
  4. Enrich event with lookup data

Pattern 2: Real-time aggregations com MemoryDB
═══════════════════════════════════════════════

Kinesis → Lambda → MemoryDB (sorted sets / hashes)
                        │
                   API Gateway → consumers (real-time dashboards)

Pattern 3: Feature Store para ML
════════════════════════════════

Glue ETL → compute features → ElastiCache (online store, low latency)
                            → S3 (offline store, training)
```

---

## Amazon Neptune — Grafos

### Quando usar Neptune

| Cenário | Neptune? | Alternativa |
|---------|----------|-------------|
| Knowledge graphs | ✅ | — |
| Fraud detection (relações complexas) | ✅ | — |
| Social network analysis | ✅ | — |
| Identity graphs | ✅ | — |
| Recommendation engine | ✅ | — |
| Simple lookups | ❌ | DynamoDB |
| Analytics SQL | ❌ | Redshift/Athena |

### Neptune para Data Engineering

```
Pattern: Construir knowledge graph a partir do data lake

S3 (Curated Zone) ──▶ Neptune Bulk Loader ──▶ Neptune
                                                  │
                                          Gremlin / SPARQL queries
                                                  │
                                          API / Applications

Graph models:
- Property Graph (Gremlin) → Social, fraud, recomendações
- RDF (SPARQL) → Knowledge management, linked data, ontologias
```

---

## Amazon Timestream — Séries Temporais

### Quando usar Timestream

| Cenário | Timestream? | Alternativa |
|---------|-------------|-------------|
| IoT sensor data (milhões de dispositivos) | ✅ | — |
| Métricas de aplicação/infra | ✅ | CloudWatch (se simples) |
| DevOps monitoring | ✅ | — |
| Log analytics | ❌ | OpenSearch |
| Business analytics | ❌ | Redshift |

### Arquitetura Timestream

```
┌──────────────────────────────────────────────────┐
│              AMAZON TIMESTREAM                    │
├──────────────────────────────────────────────────┤
│                                                   │
│  ┌─────────────────┐    ┌─────────────────┐      │
│  │  MEMORY STORE   │    │ MAGNETIC STORE  │      │
│  │  (hot data)     │    │ (cold data)     │      │
│  │                 │    │                 │      │
│  │  Recent writes  │──▶│  Auto-tiered    │      │
│  │  Fast queries   │    │  Cost-effective │      │
│  │  Hours-days     │    │  Days-years     │      │
│  └─────────────────┘    └─────────────────┘      │
│                                                   │
│  Retention policies: memory 1-8766h, magnetic 1d+ │
│  Auto-tiering: memory → magnetic (automatic)      │
│                                                   │
│  Data flow:                                       │
│  IoT Core / Kinesis / SDK ──▶ WriteRecords API    │
│  Queries: SQL-like + time functions               │
│  Export: Scheduled queries → S3 (for data lake)   │
└──────────────────────────────────────────────────┘
```

---

## Amazon OpenSearch — Search e Analytics

### Quando usar OpenSearch

| Cenário | OpenSearch? | Alternativa |
|---------|-------------|-------------|
| Full-text search | ✅ | — |
| Log analytics (ELK replacement) | ✅ | — |
| SIEM / security analytics | ✅ | — |
| Application search | ✅ | — |
| Observability (traces, metrics) | ✅ | — |
| SQL analytics | ❌ | Redshift/Athena |
| Data lake storage | ❌ | S3 |

### OpenSearch para Data Engineering

```
Pattern: Search index alimentado pelo data lake

S3 (Curated) ──▶ Lambda ──▶ OpenSearch (search)
                                  │
DynamoDB Streams ──▶ Lambda ──▶ OpenSearch (real-time index)
                                  │
Kinesis ──▶ Firehose ──▶ OpenSearch (log analytics)
                                  │
                          OpenSearch Dashboards (visualização)

OpenSearch Serverless:
  - Collections sem cluster management
  - Auto-scaling
  - Ideal para workloads variáveis
  - Suporta Vector Search (para RAG / GenAI)
```

---

## Estratégia Multi-Database

### Polyglot Persistence

Na prática, uma arquitetura de dados madura usa **múltiplos bancos** — cada um para seu caso de uso:

```
┌──────────────────────────────────────────────────────────────┐
│              POLYGLOT PERSISTENCE — E-COMMERCE                │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │  Aurora     │  │ DynamoDB   │  │ ElastiCache │            │
│  │  (Orders,  │  │ (Cart,     │  │ (Sessions,  │            │
│  │   Payments)│  │  Catalog)  │  │  Hot cache) │            │
│  └─────┬──────┘  └─────┬──────┘  └────────────┘             │
│        │               │                                      │
│        └───────┬───────┘                                      │
│                │ CDC (DMS + Streams)                          │
│                ▼                                              │
│  ┌────────────────────────┐  ┌────────────────┐             │
│  │  S3 Data Lake          │  │  OpenSearch    │             │
│  │  (Raw + Curated)       │  │  (Product     │             │
│  │                        │  │   Search)     │             │
│  └─────────┬──────────────┘  └────────────────┘             │
│            │                                                  │
│     ┌──────┴──────┐                                          │
│     │  Redshift    │     ┌────────────┐                      │
│     │  (Analytics) │     │ Neptune    │                      │
│     │              │     │ (Product   │                      │
│     └──────────────┘     │  Recommend)│                      │
│                          └────────────┘                      │
└──────────────────────────────────────────────────────────────┘
```

### Regra de decisão

| Pergunta | Se SIM | Serviço |
|----------|--------|---------|
| Precisa de JOIN e transactions? | → | Aurora |
| Precisa de latência < 10ms e escala horizontal? | → | DynamoDB |
| Precisa de analytics SQL sobre TB/PB? | → | Redshift |
| Precisa de full-text search? | → | OpenSearch |
| Precisa de cache sub-millisecond? | → | ElastiCache |
| Precisa de relações complexas (grafo)? | → | Neptune |
| Precisa de time-series? | → | Timestream |
| Precisa de storage ilimitado e barato? | → | S3 |
| Precisa de queries ad-hoc sobre S3? | → | Athena |

---

## Formatos de dados no S3

### Comparação de formatos

| Formato | Tipo | Compressão | Splittable | Schema evolution | Melhor para |
|---------|------|-----------|------------|-----------------|-------------|
| **Parquet** | Colunar | Snappy, ZSTD, Gzip | ✅ | ✅ (append) | Analytics, Athena, dbt |
| **ORC** | Colunar | ZLIB, Snappy, LZO | ✅ | ✅ (append) | Hive, EMR |
| **Avro** | Row-based | Deflate, Snappy | ✅ | ✅ (full) | Streaming, schema evolution |
| **JSON** | Row-based | Gzip | ❌ (se gzip) | N/A | Raw ingestion, APIs |
| **CSV** | Row-based | Gzip | ❌ (se gzip) | ❌ | Legacy, imports |
| **Iceberg** | Table format | Parquet-based | ✅ | ✅ (full) | Lakehouse, ACID |

### Regra prática

```
Raw Zone:    JSON, CSV, Avro (formato da fonte)
Curated Zone: Parquet (colunar, comprimido, tipado)
Refined Zone: Iceberg (ACID, time-travel, schema evolution)
Streaming:    Avro (schema registry, compact, schema evolution)
```

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e configurações de storage/databases na AWS, verifique:

1. **DynamoDB sem On-Demand ou Auto Scaling** — Risco de throttling; exija capacity management
2. **DynamoDB scan operations** — SEMPRE prefira query com partition key; scan é O(n) e caro
3. **Aurora sem Multi-AZ** — Para produção, Multi-AZ deve estar ativo
4. **Aurora sem encryption** — Encryption at rest deve estar ativado na criação
5. **Redshift sem sort key em tabelas frequentes** — Perda de performance significativa
6. **Redshift COPY sem compressão** — Dados devem ser comprimidos antes do COPY
7. **S3 bucket sem lifecycle** — Sugira lifecycle policy baseada no padrão de acesso
8. **ElastiCache sem encryption in-transit** — TLS deve estar habilitado para dados sensíveis
9. **OpenSearch sem fine-grained access control** — Habilite FGAC para multi-tenant
10. **Formato CSV/JSON na curated zone** — Sugira conversão para Parquet
11. **DynamoDB sem Point-in-Time Recovery** — Deve estar ativo para tabelas de produção
12. **Redshift INSERT row-by-row** — Substitua por COPY do S3

---

## Referências

- **Amazon S3 User Guide** — Documentação oficial AWS
- **Amazon Redshift Best Practices** — Otimizações de performance
- **Amazon DynamoDB Developer Guide** — Modelagem e patterns
- **Amazon Aurora User Guide** — Configuração e tuning
- **AWS Database Blog** — Case studies e implementações
- **Designing Data-Intensive Applications** — Martin Kleppmann — Teoria fundamental de storage
- **DynamoDB Book** — Alex DeBrie — Single Table Design e modelagem avançada
