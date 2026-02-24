# Data Engineering — Integração & Migração de Dados na AWS

> **Objetivo deste documento:** Servir como referência completa sobre **integração, CDC, migração e movimentação de dados na AWS**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Foco em **DMS, CDC patterns, EventBridge, AppFlow, DataSync, Snow Family, Schema Conversion Tool** — quando usar, boas práticas e patterns.

---

## Quick Reference — Cheat Sheet

| Cenário | Serviço AWS | Tipo |
|---------|-------------|------|
| CDC de banco relacional → Lake/Stream | **DMS + Kinesis** | Ongoing replication |
| Migração de banco (one-time) | **DMS** | Full load |
| Conversão de schema (Oracle→Aurora) | **SCT** | Schema conversion |
| SaaS → S3/Redshift (Salesforce, SAP) | **AppFlow** | API-based |
| Evento de aplicação → múltiplos targets | **EventBridge** | Event routing |
| Arquivo on-premises → S3 | **DataSync** | File transfer |
| Petabytes on-premises → S3 | **Snow Family** | Physical transfer |
| API → API (microservices) | **API Gateway + Lambda** | Sync integration |
| Banco → Redshift (zero-ETL) | **Aurora Zero-ETL** | Near real-time |

---

## Sumário

- [Data Engineering — Integração \& Migração de Dados na AWS](#data-engineering--integração--migração-de-dados-na-aws)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [CDC — Change Data Capture](#cdc--change-data-capture)
    - [O que é CDC](#o-que-é-cdc)
    - [CDC com DMS](#cdc-com-dms)
    - [CDC Patterns](#cdc-patterns)
    - [Boas Práticas de CDC](#boas-práticas-de-cdc)
    - [Anti-patterns de CDC](#anti-patterns-de-cdc)
  - [AWS DMS — Database Migration Service](#aws-dms--database-migration-service)
  - [AWS SCT — Schema Conversion Tool](#aws-sct--schema-conversion-tool)
  - [Amazon EventBridge — Event Bus](#amazon-eventbridge--event-bus)
  - [Amazon AppFlow — SaaS Integration](#amazon-appflow--saas-integration)
  - [AWS DataSync — File Transfer](#aws-datasync--file-transfer)
  - [AWS Snow Family — Physical Transfer](#aws-snow-family--physical-transfer)
  - [Zero-ETL Integrations](#zero-etl-integrations)
  - [Patterns de Integração End-to-End](#patterns-de-integração-end-to-end)
  - [Árvore de Decisão — Migração](#árvore-de-decisão--migração)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## CDC — Change Data Capture

### O que é CDC

**Change Data Capture (CDC)** é a técnica de capturar mudanças (INSERT, UPDATE, DELETE) de um banco de dados e propagar para outros sistemas em near real-time.

```
┌───────────────────────────────────────────────────────┐
│                    CDC FLOW                             │
├───────────────────────────────────────────────────────┤
│                                                        │
│  Source DB (Aurora/RDS)                                │
│       │                                                │
│       │ WAL / Binlog / Redo Log                       │
│       │ (transaction log)                              │
│       ▼                                                │
│  ┌──────────┐                                         │
│  │   DMS    │  Lê o transaction log                   │
│  │  (CDC)   │  Mínimo impacto no source               │
│  └────┬─────┘                                         │
│       │                                                │
│       │ CDC Records (INSERT/UPDATE/DELETE + metadata)  │
│       │                                                │
│       ├──▶ Kinesis Data Streams (fan-out)             │
│       │         │                                      │
│       │         ├──▶ Lambda → DynamoDB (real-time view)│
│       │         ├──▶ Firehose → S3 (data lake)        │
│       │         └──▶ Lambda → OpenSearch (search)     │
│       │                                                │
│       ├──▶ S3 directly (simple lake ingestion)        │
│       │                                                │
│       └──▶ Kafka / MSK (event-driven architecture)    │
│                                                        │
└───────────────────────────────────────────────────────┘
```

### CDC com DMS

```
DMS CDC Record Format:
═══════════════════════

{
  "data": {
    "order_id": 12345,
    "customer_id": 678,
    "total": 99.90,
    "status": "SHIPPED"
  },
  "metadata": {
    "timestamp": "2025-01-15T10:30:00Z",
    "record-type": "data",
    "operation": "update",              // insert | update | delete
    "partition-key-type": "schema-table",
    "schema-name": "public",
    "table-name": "orders",
    "transaction-id": 12345678
  }
}
```

### CDC Patterns

#### Pattern 1: CDC → Data Lake (Iceberg MERGE)

```
Aurora ──▶ DMS (CDC) ──▶ S3 Raw (CDC events)
                              │
                         Glue ETL (scheduled)
                              │
                         MERGE INTO iceberg_table
                         USING cdc_events
                         ON target.id = source.id
                         WHEN MATCHED AND op='delete' THEN DELETE
                         WHEN MATCHED AND op='update' THEN UPDATE
                         WHEN NOT MATCHED THEN INSERT
                              │
                         S3 Curated (Iceberg — current state)
```

#### Pattern 2: CDC → Real-time Materialized View

```
Aurora ──▶ DMS (CDC) ──▶ Kinesis Data Streams
                              │
                         Lambda (process CDC)
                              │
                    ┌─────────┴─────────┐
                    │                   │
               DynamoDB            ElastiCache
           (materialized view)   (hot cache)
                    │
               API Gateway
                    │
               Applications (read-optimized view)
```

#### Pattern 3: CDC → Event-Driven Architecture

```
Aurora ──▶ DMS (CDC) ──▶ MSK (Kafka)
                              │
                    ┌─────────┼─────────────┐
                    │         │             │
               Consumer 1  Consumer 2  Consumer 3
               (Inventory) (Analytics) (Notifications)
```

### Boas Práticas de CDC

| Prática | Detalhes |
|---------|---------|
| **Source sem impacto** | DMS lê WAL/binlog — impacto mínimo no source (< 5% overhead) |
| **Kinesis como target** | Prefira Kinesis para fan-out a múltiplos consumers |
| **Ordering** | CDC garante ordem por primary key — use PK como partition key no Kinesis |
| **Full Load + CDC** | Inicie com full load, depois ligue CDC para ongoing — garante consistência |
| **Schema changes** | DMS detecta DDL — configure handling (stop task, ignore, log) |
| **LOB handling** | Colunas LOB (BLOB, CLOB) — configure `LobMaxSize` ou `LimitedLobMode` |
| **Validation** | Use DMS Data Validation para comparar source vs target |
| **Monitoring** | CloudWatch: `CDCLatencySource`, `CDCLatencyTarget`, `CDCThroughputRows` |
| **Multi-AZ** | Ative Multi-AZ para DMS replication instances em produção |

### Anti-patterns de CDC

| Anti-pattern | Problema | Solução |
|-------------|----------|---------|
| **Polling table com timestamp** | Impacto no source, perde deletes, inconsistente | Use CDC baseado em WAL (DMS) |
| **CDC sem ordering** | Updates fora de ordem corrompem o target | Use partition key = PK no Kinesis |
| **Full load diário como CDC** | Lento, caro, window de indisponibilidade | CDC ongoing para mudanças incrementais |
| **CDC sem schema evolution** | DDL quebra o pipeline silenciosamente | Configure DMS para alertar em DDL |
| **CDC sem monitoramento** | Lag cresce sem ser detectado | Alerte em CDCLatencySource > threshold |
| **CDC direto para warehouse** | Lock no warehouse durante ingestão | CDC → S3 → Glue → Warehouse (stage) |

---

## AWS DMS — Database Migration Service

### Modos de operação

| Modo | Descrição | Caso de uso |
|------|-----------|-------------|
| **Full Load** | Copia todos os dados do source → target uma vez | Migração inicial |
| **Full Load + CDC** | Full load + captura ongoing changes | Migração com zero downtime |
| **CDC only** | Só captura mudanças ongoing | Após full load completar; data lake feeding |

### Engines suportados

| Source | Target (popular) |
|--------|-----------------|
| Oracle, SQL Server, MySQL, PostgreSQL, Aurora | Aurora, RDS, DynamoDB |
| MongoDB, DocumentDB | S3, Kinesis, Kafka |
| SAP ASE, IBM Db2 | Redshift, OpenSearch |
| S3 (CSV, Parquet) | Neptune, Timestream |

### DMS Serverless

| Aspecto | DMS Classic | DMS Serverless |
|---------|------------|----------------|
| **Infra** | Replication Instance (EC2) | Auto-provisioned |
| **Sizing** | Manual (dms.r5.large, etc.) | Auto-scaling (min/max capacity) |
| **Custo** | Por hora da instância | Por DCU-hora (capacity units) |
| **Operação** | Gerencia instância, storage | Zero management |
| **Quando usar** | Migrations grandes, custom config | Default para novos setups |

### Boas Práticas DMS

| Prática | Detalhes |
|---------|---------|
| **Pre-migration assessment** | Execute o DMS Pre-migration Assessment antes de migrar |
| **SCT primeiro** | Use Schema Conversion Tool para converter schema antes de DMS |
| **Table mappings** | Configure selection rules para migrar apenas tabelas necessárias |
| **Transformation rules** | Renomeie schemas/tables/columns durante migração |
| **Parallel load** | Configure parallel load por tabela para acelerar full load |
| **LOB columns** | Use LimitedLobMode com tamanho adequado — FullLobMode é muito lento |
| **Multi-AZ** | Ative para produção — failover automático da replication instance |
| **Validation** | Ative Data Validation para comparar source vs target row-by-row |
| **Monitoring** | Alarmes em CDCLatencySource, FreeableMemory, SwapUsage |

---

## AWS SCT — Schema Conversion Tool

### O que é SCT

O **Schema Conversion Tool** converte schemas de banco de dados de uma engine para outra (ex: Oracle → Aurora PostgreSQL).

```
SCT Workflow:
═════════════

1. Connect to source (Oracle)
2. Connect to target (Aurora PostgreSQL)
3. SCT analyzes schema differences
4. Generates conversion report:
   - Green: auto-converted (80-90%)
   - Yellow: needs review
   - Red: manual conversion required
5. Apply converted schema to target
6. Manual fix for Red items
7. Validate with DMS
```

### Cenários de migração

| De | Para | Complexidade | SCT converte |
|----|------|-------------|-------------|
| Oracle → Aurora PostgreSQL | Alta | ~85% schema, ~70% PL/SQL |
| SQL Server → Aurora MySQL | Média | ~90% schema, ~75% T-SQL |
| Oracle → Redshift | Média | ~80% (DW optimization) |
| Teradata → Redshift | Alta | ~75% (complete rewrite recomendado) |
| MySQL → Aurora MySQL | Baixa | ~99% (compatível) |

---

## Amazon EventBridge — Event Bus

### EventBridge para Data Engineering

```
┌──────────────────────────────────────────────────────────┐
│                   EVENTBRIDGE                              │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  Sources                  Event Bus           Targets     │
│  ───────                  ─────────           ───────     │
│  AWS Services      ┌──────────────────┐                   │
│  (S3, DynamoDB,    │                  │    Lambda          │
│   Glue, EMR,       │   Default Bus    │    Step Functions  │
│   Step Functions)   │   ou             │    Kinesis         │
│                     │   Custom Bus     │    SQS             │
│  SaaS Partners     │                  │    SNS             │
│  (Datadog, PagerDuty)│  Rules →       │    Glue            │
│                     │  Filter →       │    Redshift        │
│  Custom Apps        │  Transform →    │    ECS Tasks       │
│  (PutEvents API)    │  Route          │    API Destination │
│                     └──────────────────┘                   │
│                                                           │
│  Features:                                                │
│  - Event Archive (replay events)                          │
│  - Schema Registry (auto-discover schemas)                │
│  - Pipes (source → filter → enrich → target)             │
│  - Scheduler (cron/rate for pipelines)                    │
└──────────────────────────────────────────────────────────┘
```

### EventBridge Patterns para DE

```
Pattern 1: S3 event → trigger pipeline
═══════════════════════════════════════
S3 (PutObject) ──▶ EventBridge Rule
                        │
                   Filter: prefix = "raw/orders/"
                   Filter: suffix = ".json"
                        │
                   ──▶ Step Functions (orders pipeline)

Pattern 2: Glue Job completion → next step
═══════════════════════════════════════════
Glue Job (SUCCEEDED) ──▶ EventBridge Rule
                              │
                         ──▶ Lambda (start next pipeline stage)

Pattern 3: Cross-account data events
════════════════════════════════════
Account A (producer):
  EventBridge ──▶ Cross-account event bus (Account B)
                         │
Account B (consumer):
  ──▶ Lambda ──▶ process data product

Pattern 4: EventBridge Pipes (point-to-point)
═════════════════════════════════════════════
DynamoDB Streams ──▶ EventBridge Pipe
                         │
                    Filter (only INSERT/UPDATE)
                    Enrich (Lambda: add metadata)
                    Target (Kinesis: for analytics)
```

### Boas Práticas EventBridge

| Prática | Detalhes |
|---------|---------|
| **Event Archive** | Ative para replay de eventos — debugging e reprocessamento |
| **Schema Registry** | Use para documentar formato dos eventos automaticamente |
| **DLQ per rule** | Configure SQS DLQ para cada rule — não perca eventos que falharam |
| **Input transformer** | Transforme o evento antes de enviar ao target — reduza payload |
| **Cross-account** | Use resource-based policies para compartilhar event buses |
| **Pipes** | Prefira Pipes para integrações point-to-point simples (DynamoDB→Kinesis) |
| **Scheduler** | Use EventBridge Scheduler para triggers de pipeline (cron/rate) |

---

## Amazon AppFlow — SaaS Integration

### Quando usar AppFlow

| Cenário | AppFlow? | Alternativa |
|---------|----------|-------------|
| Salesforce → S3/Redshift | ✅ | Custom API integration |
| SAP → S3 | ✅ | — |
| Zendesk → S3 | ✅ | — |
| Google Analytics → S3 | ✅ | — |
| Slack → S3 | ✅ | — |
| Custom REST API → S3 | ❌ | Lambda + API Gateway |
| Database → S3 | ❌ | DMS |

### AppFlow Flow Types

| Tipo | Trigger | Caso de uso |
|------|---------|-------------|
| **On-demand** | Manual | Carga inicial, teste |
| **Scheduled** | Cron/Rate | Sync diário/horário |
| **Event-driven** | Source event (ex: Salesforce record change) | Near real-time sync |

### Capabilities

```
Source (Salesforce)
     │
     ▼
AppFlow:
  1. Extract (API call to source)
  2. Filter (field-level filtering)
  3. Map (rename fields, add/remove)
  4. Transform (mask, truncate, validate)
  5. Partition (by field or date)
     │
     ▼
Destination:
  - S3 (Parquet, JSON, CSV)
  - Redshift
  - Snowflake
  - EventBridge (route events)
  - Custom connector (Lambda)
```

---

## AWS DataSync — File Transfer

### Quando usar DataSync

| Cenário | DataSync? | Alternativa |
|---------|-----------|-------------|
| NFS/SMB on-prem → S3 | ✅ | — |
| EFS → S3 (cross-service) | ✅ | — |
| S3 → S3 (cross-account/region) | ✅ | S3 Replication |
| HDFS on-prem → S3 | ✅ | — |
| Scheduled file sync | ✅ | — |
| Streaming data | ❌ | Kinesis / MSK |
| Database migration | ❌ | DMS |

### Arquitetura DataSync

```
On-Premises                           AWS
─────────────                         ───
┌──────────┐     ┌──────────┐     ┌──────────┐
│ NFS/SMB  │     │ DataSync │     │ DataSync │
│ Server   │────▶│ Agent    │────▶│ Service  │
│          │     │ (VM)     │  │  │          │
└──────────┘     └──────────┘  │  └────┬─────┘
                            TLS │       │
                        encrypted│      ▼
                               │  ┌──────────┐
                               │  │ S3 / EFS │
                               │  │ / FSx    │
                               │  └──────────┘

Features:
- Automatic encryption (TLS in transit)
- Data integrity verification
- Bandwidth throttling
- Scheduling (hourly, daily, weekly)
- Incremental transfer (only changed files)
- Up to 10 Gbps throughput per agent
```

---

## AWS Snow Family — Physical Transfer

### Quando não é viável transferir pela rede

| Dispositivo | Capacidade | Caso de uso |
|------------|-----------|-------------|
| **Snowcone** | 8TB HDD / 14TB SSD | Edge computing, escritórios remotos |
| **Snowball Edge Storage** | 80TB | Migração de dados medium-scale |
| **Snowball Edge Compute** | 80TB + compute (EC2/Lambda) | Edge computing + storage |
| **Snowmobile** | 100PB (caminhão!) | Data center migration |

### Regra de decisão: rede vs Snow

```
Calcule: Dados(TB) / Bandwidth(Gbps) = Tempo de transferência

Exemplo: 100TB / 1Gbps = ~9.3 dias
         100TB / 10Gbps = ~0.9 dias
         1PB / 1Gbps = ~93 dias → use Snowball!
         
Regra prática:
  < 10TB  → Transferência pela rede (DataSync)
  10-80TB → Depende do bandwidth (pode ir qualquer um)
  > 80TB  → Snowball Edge
  > 10PB  → Multiple Snowballs
  > 50PB  → Snowmobile
```

---

## Zero-ETL Integrations

A AWS está investindo em **Zero-ETL** — eliminar pipelines de ETL entre serviços.

### Integrações disponíveis

| Source | Target | Latência | Status |
|--------|--------|----------|--------|
| **Aurora MySQL** | Redshift | Segundos | GA |
| **Aurora PostgreSQL** | Redshift | Segundos | GA |
| **RDS MySQL** | Redshift | Segundos | GA |
| **DynamoDB** | Redshift | Near real-time | GA |
| **DynamoDB** | OpenSearch | Near real-time | GA |
| **Aurora** | OpenSearch | Near real-time | GA |

### Arquitetura Zero-ETL

```
Before (traditional):
Aurora ──▶ DMS ──▶ S3 ──▶ Glue ETL ──▶ Redshift
           │         │        │
      Config    Storage    Config+Run
      Manage    Cost       Cost+Manage
           │         │        │
           └────── COMPLEXITY + LATENCY ──────┘

After (Zero-ETL):
Aurora ──────────────────────────────▶ Redshift
              │
         Zero-ETL integration
         (automatic, managed, near real-time)
         
Benefits:
  - No pipeline to manage
  - Near real-time (seconds)
  - No intermediate storage cost
  - Automatic schema sync
```

### Quando usar Zero-ETL vs CDC pipeline

| Critério | Zero-ETL | CDC Pipeline (DMS) |
|----------|----------|-------------------|
| **Simplicidade** | ✅ Muito simples | ❌ Complexo |
| **Transformação** | ❌ Sem transform | ✅ Transform no pipeline |
| **Multi-target** | ❌ Source → 1 target | ✅ Source → N targets |
| **Customização** | ❌ Limitada | ✅ Total |
| **Custo** | Lower (no infra) | Higher (DMS + processing) |
| **Quando usar** | Replicação direta sem transform | Precisa de enrich/transform/multi-target |

---

## Patterns de Integração End-to-End

### Pattern: Enterprise Data Integration

```
┌──────────────────────────────────────────────────────────────┐
│            ENTERPRISE DATA INTEGRATION                        │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  SOURCES                                                      │
│  ───────                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │ Aurora   │  │ Legacy   │  │Salesforce│  │ IoT/Apps │     │
│  │ (OLTP)   │  │ Oracle   │  │ (SaaS)   │  │ (Events) │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘    │
│       │              │              │              │          │
│       │ DMS (CDC)    │ DMS          │ AppFlow      │ Kinesis  │
│       │              │ (Full+CDC)   │              │          │
│       ▼              ▼              ▼              ▼          │
│  ┌───────────────────────────────────────────────────┐       │
│  │                   S3 RAW ZONE                      │       │
│  │           (landing, imutável, formato original)     │       │
│  └──────────────────────┬────────────────────────────┘       │
│                         │                                     │
│                    EventBridge                                │
│                    (trigger pipeline)                         │
│                         │                                     │
│                    Step Functions                              │
│                    (orchestrate)                               │
│                         │                                     │
│              ┌──────────┼──────────┐                         │
│              │          │          │                          │
│         Glue ETL   Data Quality  Catalog                     │
│         (transform)(validate)   (register)                   │
│              │          │          │                          │
│              └──────────┼──────────┘                         │
│                         │                                     │
│  ┌──────────────────────▼────────────────────────────┐       │
│  │              S3 CURATED ZONE (Iceberg)             │       │
│  │      (clean, typed, partitioned, ACID)             │       │
│  └──────────────────────┬────────────────────────────┘       │
│                         │                                     │
│              ┌──────────┼──────────────┐                     │
│              │          │              │                      │
│          Redshift    Athena      QuickSight                  │
│          (DW)       (ad-hoc)    (dashboards)                 │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Árvore de Decisão — Migração

```
           Preciso migrar dados para AWS?
                      │
              ┌───────┼───────┐
              │       │       │
          Database  Files   Physical
              │       │    (> 80TB)
              │       │       │
          ┌───┤   DataSync  Snow Family
          │   │
     Same    Different
     engine? engine?
          │   │
       DMS   DMS +
     (simple) SCT
          │   │
          │   │
     ┌────┤   └── Convert schema first
     │    │
  One-time  Ongoing
     │         │
  Full Load  Full Load
             + CDC
```

---

## Diretrizes para Code Review assistido por AI

Ao revisar código de integração e migração de dados na AWS, verifique:

1. **DMS sem Multi-AZ** — Para produção, replication instance deve ser Multi-AZ
2. **DMS com FullLobMode** — Muito lento; prefira LimitedLobMode com tamanho adequado
3. **CDC sem ordering garantido** — Primary key deve ser partition key no Kinesis/Kafka
4. **CDC sem monitoramento de lag** — Alarme em CDCLatencySource > threshold (ex: 5 min)
5. **DMS sem validation** — Data Validation deve estar ativo para comparar source vs target
6. **EventBridge rule sem DLQ** — Eventos que falharam são perdidos; configure SQS DLQ
7. **AppFlow sem encryption** — Dados in-transit e at-rest devem ser criptografados
8. **DataSync sem bandwidth throttle** — Pode saturar a rede; configure throttle em produção
9. **Zero-ETL onde precisa de transform** — Zero-ETL não transforma; use CDC pipeline se precisar
10. **Full load diário substituindo CDC** — Ineficiente; use CDC para mudanças incrementais
11. **CDC sem tratamento de schema change** — Configure DMS para alertar em DDL do source
12. **Integração sem idempotência** — Writes devem ser idempotentes para retry seguro

---

## Referências

- **AWS DMS User Guide** — Database Migration Service completo
- **AWS SCT User Guide** — Schema conversion
- **Amazon EventBridge User Guide** — Event routing patterns
- **AWS DataSync User Guide** — File transfer
- **AWS Snow Family Documentation** — Physical data transfer
- **Designing Data-Intensive Applications** — Martin Kleppmann — CDC fundamentals
- **Building Event-Driven Microservices** — Adam Bellemare — Event integration patterns
