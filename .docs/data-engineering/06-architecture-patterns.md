# Data Engineering — Arquiteturas de Referência & Anti-Patterns

> **Objetivo deste documento:** Servir como referência completa sobre **arquiteturas de referência, decision frameworks, anti-patterns e checklist de produção** para data engineering na AWS, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.

---

## Quick Reference — Arquiteturas

| Arquitetura | Complexidade | Quando usar | Serviços AWS chave |
|-------------|-------------|-------------|-------------------|
| **Data Lake Simples** | Baixa | Time pequeno, primeiros passos | S3, Glue, Athena |
| **Data Lakehouse** | Média | Analytics + transactions | S3, Iceberg, Athena, Redshift |
| **Lambda Architecture** | Alta | Batch + real-time obrigatórios | Glue + Kinesis + Redshift |
| **Kappa Architecture** | Média | Tudo como stream | MSK/Kinesis + Flink |
| **Data Mesh** | Muito alta | Multi-domain, times autônomos | DataZone + Lake Formation |
| **Modern Data Stack** | Média | Analytics-first, ELT | Firehose + S3 + dbt + Redshift |

---

## Sumário

- [Data Engineering — Arquiteturas de Referência \& Anti-Patterns](#data-engineering--arquiteturas-de-referência--anti-patterns)
  - [Quick Reference — Arquiteturas](#quick-reference--arquiteturas)
  - [Sumário](#sumário)
  - [Arquitetura 1: Data Lake Simples](#arquitetura-1-data-lake-simples)
  - [Arquitetura 2: Data Lakehouse Moderno](#arquitetura-2-data-lakehouse-moderno)
  - [Arquitetura 3: Real-Time Analytics](#arquitetura-3-real-time-analytics)
  - [Arquitetura 4: CDC-Driven Data Platform](#arquitetura-4-cdc-driven-data-platform)
  - [Arquitetura 5: Modern Data Stack (ELT)](#arquitetura-5-modern-data-stack-elt)
  - [Arquitetura 6: ML Feature Platform](#arquitetura-6-ml-feature-platform)
  - [Árvore de Decisão — Qual Arquitetura?](#árvore-de-decisão--qual-arquitetura)
  - [Anti-Patterns de Arquitetura de Dados](#anti-patterns-de-arquitetura-de-dados)
  - [Checklist de Produção](#checklist-de-produção)
  - [Estimativa de Custos](#estimativa-de-custos)
  - [Evolução da Arquitetura](#evolução-da-arquitetura)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Arquitetura 1: Data Lake Simples

### Caso de uso
Time pequeno (2-5 pessoas), primeiros passos com data lake, analytics sobre dados batch.

```
┌──────────────────────────────────────────────────────────────┐
│                 DATA LAKE SIMPLES                              │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Sources                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                     │
│  │ Aurora   │ │ SaaS     │ │ Files    │                     │
│  │ (OLTP)   │ │ (APIs)   │ │ (CSV)    │                     │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘                     │
│       │             │            │                            │
│       │ DMS         │ AppFlow    │ DataSync                  │
│       │             │ /Lambda    │                            │
│       ▼             ▼            ▼                            │
│  ┌───────────────────────────────────────────┐               │
│  │              S3 RAW ZONE                   │               │
│  │  s3://lake/raw/source=xxx/year=.../        │               │
│  └──────────────────┬────────────────────────┘               │
│                     │                                         │
│               Glue Crawler                                    │
│                     │                                         │
│               Glue ETL Job                                    │
│               (clean, dedup, type, partition)                 │
│                     │                                         │
│  ┌──────────────────▼────────────────────────┐               │
│  │           S3 CURATED ZONE (Parquet)        │               │
│  │  s3://lake/curated/domain=xxx/entity=xxx/  │               │
│  └──────────────────┬────────────────────────┘               │
│                     │                                         │
│              Glue Data Catalog                                │
│                     │                                         │
│              ┌──────┤                                         │
│              │      │                                         │
│           Athena  QuickSight                                 │
│          (ad-hoc) (dashboards)                               │
│                                                               │
│  Orchestration: Step Functions (EventBridge trigger)          │
│  Governance: Lake Formation (basic RBAC)                     │
│  Monitoring: CloudWatch + SNS alerts                         │
│  Cost: $$  (S3 + Glue + Athena pay-per-use)                │
└──────────────────────────────────────────────────────────────┘
```

### Prós e contras

| Prós | Contras |
|------|---------|
| ✅ Simples de implementar e operar | ❌ Sem ACID transactions |
| ✅ 100% serverless, pay-per-use | ❌ Sem real-time |
| ✅ Baixo custo inicial | ❌ Athena latência variável |
| ✅ Escala naturalmente com S3 | ❌ Sem update/delete granular no S3 |

---

## Arquitetura 2: Data Lakehouse Moderno

### Caso de uso
Time médio (5-15 pessoas), precisa de ACID transactions, time-travel, schema evolution. Evolução natural do Data Lake Simples.

```
┌──────────────────────────────────────────────────────────────┐
│                  DATA LAKEHOUSE MODERNO                        │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Sources ──▶ [DMS/AppFlow/Kinesis Firehose] ──▶ S3 RAW      │
│                                                               │
│  S3 RAW ──▶ EventBridge ──▶ Step Functions                   │
│                                   │                           │
│                    ┌──────────────┼──────────────┐           │
│                    │              │              │           │
│               Glue ETL      Data Quality    Update Catalog  │
│               (to Iceberg)  (validate)     (register)       │
│                    │                                         │
│  ┌─────────────────▼─────────────────────────────────┐      │
│  │         S3 CURATED ZONE (Apache Iceberg)           │      │
│  │                                                     │      │
│  │  Features:                                          │      │
│  │  • ACID transactions                                │      │
│  │  • Time travel (point-in-time queries)              │      │
│  │  • Schema evolution (add/rename columns)            │      │
│  │  • Row-level deletes (GDPR/LGPD compliance)         │      │
│  │  • Hidden partitioning (partition evolution)         │      │
│  │  • Compaction (auto-optimize file sizes)            │      │
│  └────────────────────┬──────────────────────────────┘      │
│                       │                                      │
│            ┌──────────┼──────────┐                          │
│            │          │          │                          │
│         Athena    Redshift    EMR                           │
│        (ad-hoc)  Spectrum   (Spark)                        │
│            │      (DW)                                      │
│            │          │                                      │
│         QuickSight  dbt                                     │
│        (dashboards) (transforms)                            │
│                                                               │
│  Governance: Lake Formation (LF-Tags)                        │
│  Quality: Glue Data Quality (cada estágio)                   │
│  Monitoring: CloudWatch + custom pipeline metrics            │
│  Cost: $$$                                                   │
└──────────────────────────────────────────────────────────────┘
```

### Delta vs Iceberg vs Hudi — Decisão para AWS

| Critério | Apache Iceberg | Delta Lake | Apache Hudi |
|----------|---------------|-----------|-------------|
| **Integração AWS nativa** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **Athena support** | ✅ Full | Via connector | Via connector |
| **Redshift support** | ✅ Native | ❌ | ❌ |
| **Glue support** | ✅ Native | ✅ | ✅ |
| **EMR support** | ✅ | ✅ | ✅ |
| **Partition evolution** | ✅ Hidden partitioning | ❌ | ❌ |
| **Schema evolution** | ✅ Full | ✅ Partial | ✅ Partial |
| **Community** | Apache (vendor-neutral) | Databricks (open-source) | Uber/Apache |
| **Recomendação AWS** | ⭐ **Recomendado** | Ok | Ok para CDC-heavy |

> **Regra prática:** Na AWS, escolha **Apache Iceberg** como default para lakehouse.

---

## Arquitetura 3: Real-Time Analytics

### Caso de uso
Fraud detection, real-time dashboards, alerting, IoT analytics. Precisa de latência sub-second.

```
┌──────────────────────────────────────────────────────────────┐
│                  REAL-TIME ANALYTICS                           │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                     │
│  │ App      │ │ IoT Core │ │ CDC (DMS)│                     │
│  │ Events   │ │          │ │          │                     │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘                     │
│       └─────────────┼───────────┘                            │
│                     │                                         │
│              Kinesis Data Streams                             │
│                     │                                         │
│              ┌──────┼──────────┐                             │
│              │      │          │                             │
│              ▼      ▼          ▼                             │
│        ┌─────────┐ ┌───────┐ ┌──────────┐                   │
│        │ Kinesis │ │Lambda │ │ Firehose │                   │
│        │Analytics│ │(enrich│ │          │                   │
│        │ (Flink) │ │+ alert│ │ Buffer   │                   │
│        │         │ │)      │ │ 5min     │                   │
│        │ Windows │ │       │ │ JSON→    │                   │
│        │ Agg     │ └───┬───┘ │ Parquet  │                   │
│        └────┬────┘     │     └────┬─────┘                   │
│             │          │          │                           │
│             ▼          ▼          ▼                           │
│        DynamoDB   SNS/SQS    S3 (lake)                      │
│        (real-time (alerts)   (historical)                    │
│         state)                    │                           │
│             │               Athena/Redshift                  │
│             │              (historical analytics)            │
│             ▼                     │                           │
│        API Gateway           QuickSight                     │
│        (real-time API)      (dashboards)                    │
│                                                               │
│  Cost: $$$$  (streaming always-on)                           │
└──────────────────────────────────────────────────────────────┘
```

---

## Arquitetura 4: CDC-Driven Data Platform

### Caso de uso
OLTP databases como fonte principal. Precisa de dados fresh no lake/warehouse sem impacto no source.

```
┌──────────────────────────────────────────────────────────────┐
│                CDC-DRIVEN DATA PLATFORM                       │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                     │
│  │ Aurora   │ │ RDS      │ │ Legacy   │                     │
│  │ (orders) │ │ (users)  │ │ (Oracle) │                     │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘                     │
│       │             │            │                            │
│       └─────────────┼────────────┘                            │
│                     │                                         │
│              DMS (CDC — ongoing)                              │
│              Full Load + CDC                                  │
│                     │                                         │
│              Kinesis Data Streams                             │
│              (fan-out hub)                                    │
│                     │                                         │
│       ┌─────────────┼─────────────┐                          │
│       │             │             │                          │
│       ▼             ▼             ▼                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                     │
│  │ Firehose │ │ Lambda   │ │ Lambda   │                     │
│  │ → S3 Raw │ │→ DynamoDB│ │→OpenSearc│                     │
│  │ (archive)│ │(API view)│ │(search)  │                     │
│  └────┬─────┘ └──────────┘ └──────────┘                     │
│       │                                                       │
│  Glue ETL (MERGE INTO Iceberg)                               │
│       │                                                       │
│  S3 Curated (Iceberg — current state)                        │
│       │                                                       │
│  ┌────┼────────┐                                             │
│  │    │        │                                             │
│  Athena  Redshift                                            │
│  (ad-hoc) (DW)                                               │
│                                                               │
│  OR: Aurora Zero-ETL → Redshift (simpler path)              │
│                                                               │
│  Cost: $$$                                                    │
└──────────────────────────────────────────────────────────────┘
```

---

## Arquitetura 5: Modern Data Stack (ELT)

### Caso de uso
Analytics-first, SQL-centric team, dbt como transform layer.

```
┌──────────────────────────────────────────────────────────────┐
│                  MODERN DATA STACK (ELT)                      │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  EXTRACT + LOAD (EL)                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                     │
│  │ AppFlow  │ │ Firehose │ │ DMS      │                     │
│  │ (SaaS)   │ │ (events) │ │ (DBs)    │                     │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘                     │
│       └─────────────┼────────────┘                            │
│                     │                                         │
│                     ▼                                         │
│              S3 (Raw — as-is)                                │
│              or Redshift (staging schema)                     │
│                                                               │
│  TRANSFORM (T)                                               │
│  ┌──────────────────────────────────────┐                    │
│  │              dbt                      │                    │
│  │                                       │                    │
│  │  models/                              │                    │
│  │  ├── staging/     (1:1 source clean)  │                    │
│  │  ├── intermediate/ (business logic)   │                    │
│  │  └── marts/       (final — star schema)│                   │
│  │                                       │                    │
│  │  tests/           (data quality)      │                    │
│  │  docs/            (auto-generated)    │                    │
│  │                                       │                    │
│  │  Runs on: Redshift / Athena / Glue    │                    │
│  │  Orchestration: MWAA / Step Functions │                    │
│  └──────────────────────────────────────┘                    │
│                                                               │
│  SERVE                                                       │
│  ┌──────────────────────────────────────┐                    │
│  │  Redshift (warehouse) / Athena (lake)│                    │
│  │         │                             │                    │
│  │    QuickSight (dashboards)            │                    │
│  │    Notebooks (data science)           │                    │
│  │    APIs (reverse ETL)                 │                    │
│  └──────────────────────────────────────┘                    │
│                                                               │
│  Cost: $$$                                                   │
└──────────────────────────────────────────────────────────────┘
```

### dbt na AWS — Opções

| dbt + Engine | Como | Quando |
|-------------|------|--------|
| **dbt + Redshift** | dbt-redshift adapter | DW-centric, BI heavy |
| **dbt + Athena** | dbt-athena adapter | Lake-centric, serverless |
| **dbt + Glue** | dbt-glue adapter | Spark transforms via Glue |
| **Orchestration** | MWAA DAG que executa `dbt run` | Production scheduling |

---

## Arquitetura 6: ML Feature Platform

### Caso de uso
ML/AI investido, precisa de feature store, training data, online serving.

```
┌──────────────────────────────────────────────────────────────┐
│                ML FEATURE PLATFORM                            │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Raw Data (S3 Lake) ──▶ Glue ETL ──▶ Feature Engineering    │
│                                            │                  │
│                              ┌──────────────┼──────┐         │
│                              │              │      │         │
│                              ▼              ▼      ▼         │
│                     ┌──────────────┐ ┌────────┐ ┌───────┐   │
│                     │ SageMaker    │ │  S3    │ │ElastiC│   │
│                     │ Feature      │ │(offline│ │ache   │   │
│                     │ Store        │ │ store) │ │(online│   │
│                     │              │ │        │ │ store)│   │
│                     │ Online +     │ │training│ │<10ms  │   │
│                     │ Offline      │ │ data   │ │serving│   │
│                     └──────┬───────┘ └───┬────┘ └───┬───┘   │
│                            │             │          │        │
│                            │             ▼          │        │
│                            │      SageMaker         │        │
│                            │      Training          │        │
│                            │             │          │        │
│                            │             ▼          │        │
│                            │      SageMaker         │        │
│                            │      Endpoint     ◄────┘        │
│                            │      (inference)                │
│                            │             │                    │
│                            │             ▼                    │
│                            │      API Gateway                │
│                            │      (predictions API)          │
│                            │                                  │
│  Monitoring: SageMaker Model Monitor (data drift, bias)     │
│  Cost: $$$$$                                                 │
└──────────────────────────────────────────────────────────────┘
```

---

## Árvore de Decisão — Qual Arquitetura?

```
                    Qual é o cenário principal?
                              │
          ┌───────────────────┼─────────────────────┐
          │                   │                     │
    Analytics             Real-time              ML/AI
    (batch BI)            (sub-second)           (features + training)
          │                   │                     │
    ┌─────┤              Real-Time              ML Feature
    │     │              Analytics              Platform
    │     │
 Simples  Complexo
 (< 5TB)  (> 5TB)
    │     │
    │   ┌─┤
    │   │ │
  Data  │ ACID/time-travel
  Lake  │ needed?
 Simples│    │
    │   │   Sim
    │   │    │
    │  Data Lakehouse
    │
    └── Com dbt/SQL-centric? ── YES ── Modern Data Stack
                              │
                              NO
                              │
                    Sources são DBs? ── YES ── CDC-Driven Platform
```

---

## Anti-Patterns de Arquitetura de Dados

### Anti-patterns estratégicos

| Anti-pattern | Descrição | Consequência | Solução |
|-------------|-----------|--------------|---------|
| **Data Swamp** | Data lake sem catálogo, sem quality, sem owner | Ninguém confia nos dados | Glue Catalog obrigatório, DQ rules, ownership |
| **Golden Hammer** | Usar Redshift para tudo (OLTP + analytics) | Performance ruim, custo alto | Polyglot persistence — ferramenta certa para cada caso |
| **ETL Spaghetti** | Centenas de jobs Glue sem orquestração | Impossível debugar, cascading failures | Step Functions/MWAA, pipeline-as-code |
| **Real-time Everything** | Streaming quando batch de 1h resolve | 5x mais custo, 10x mais complexidade | Avalie SLA real antes de escolher streaming |
| **Copy-Paste Architecture** | Copiar dados para N sistemas sem CDC | N versões diferentes da "verdade" | CDC centralizado, single source of truth |
| **Premature Data Mesh** | Data Mesh com < 3 domínios | Overhead sem benefício | Centralize até ter escala para descentralizar |
| **Vendor Lock-in** | Formato proprietário, sem portabilidade | Migração cara e arriscada | Formatos abertos (Parquet, Iceberg), standard SQL |
| **No Idempotency** | Pipelines que duplicam ao retry | Dados incorretos, double-counting | Idempotent writes, dedup, Iceberg MERGE INTO |

### Anti-patterns operacionais

| Anti-pattern | Descrição | Solução |
|-------------|-----------|---------|
| **No monitoring** | Pipeline falha silenciosamente | Alertas em cada step, SLA tracking |
| **No data quality** | Bad data propaga downstream | Quality gates entre zonas |
| **No retry** | Falha transiente derruba tudo | Retry com exponential backoff |
| **No DLQ** | Records perdidos para sempre | Dead Letter Queue para investigação |
| **No lineage** | Não sabe de onde vem o número | Data Catalog + lineage tracking |
| **No cost tracking** | Fatura AWS = surpresa mensal | Tags, cost allocation, budgets |
| **No documentation** | Novo membro demora 3 meses | DataZone, Catalog descriptions, README |

---

## Checklist de Produção

### Antes de ir para produção, valide:

#### Storage
- [ ] S3 Block Public Access ativado
- [ ] S3 Encryption (SSE-S3 ou SSE-KMS)
- [ ] S3 Versioning na raw zone
- [ ] S3 Lifecycle Policies configuradas
- [ ] Formato Parquet/ORC na curated zone (não CSV/JSON)
- [ ] Particionamento adequado (por data typically)
- [ ] Tamanho de arquivo 128MB-1GB (sem small files)

#### Pipeline
- [ ] Orquestração (Step Functions ou MWAA) — não jobs manuais
- [ ] Retry com backoff em cada step
- [ ] DLQ para records que falharam
- [ ] Idempotência — pipeline pode rodar 2x sem duplicar dados
- [ ] Timeout configurado em cada job
- [ ] Job bookmarks ativos (Glue)

#### Data Quality
- [ ] Quality rules em cada zona (raw → curated → refined)
- [ ] Quarantine bucket para dados que falharam
- [ ] Quality metrics no CloudWatch
- [ ] Alertas quando quality < threshold

#### Governance
- [ ] Todos os datasets no Glue Data Catalog
- [ ] Lake Formation access control (não só IAM)
- [ ] Tags obrigatórias: domain, classification, owner, environment
- [ ] PII identificado e protegido (Macie)
- [ ] Audit trail (CloudTrail)

#### Security
- [ ] IAM Roles (nunca access keys em código)
- [ ] Encryption at rest em todos os serviços
- [ ] TLS in transit
- [ ] VPC endpoints para S3/Glue/Kinesis
- [ ] Least privilege access

#### Observability
- [ ] CloudWatch dashboards para pipeline health
- [ ] Alertas de falha → Slack/PagerDuty
- [ ] SLA tracking (data freshness)
- [ ] Cost monitoring com budget alerts
- [ ] Log retention policy

#### DR & Resiliência
- [ ] S3 Cross-Region Replication (se DR multi-region)
- [ ] DMS Multi-AZ (se CDC)
- [ ] Aurora Global Database (se source crítico)
- [ ] Pipeline replay capability (archive raw data)
- [ ] RTO e RPO definidos

---

## Estimativa de Custos

### Custo mensal estimado por arquitetura (exemplo: 1TB dados/dia)

| Arquitetura | S3 | Compute | Analytics | Governance | Total estimado |
|-------------|-----|---------|-----------|-----------|---------------|
| **Data Lake Simples** | $50 | $200 (Glue) | $50 (Athena) | $0 | ~$300/mês |
| **Data Lakehouse** | $50 | $400 (Glue) | $200 (Athena+Redshift SL) | $50 | ~$700/mês |
| **Real-Time Analytics** | $50 | $800 (Kinesis+Flink) | $200 | $50 | ~$1,100/mês |
| **CDC-Driven** | $50 | $500 (DMS+Glue) | $200 | $50 | ~$800/mês |
| **Modern Data Stack** | $50 | $300 (Glue) | $500 (Redshift SL) | $50 | ~$900/mês |
| **ML Feature Platform** | $50 | $400 | $200 | $50 + $500 SM | ~$1,200/mês |

> **Nota:** Estimativas simplificadas. Custo real depende de volume, queries, retenção, região.

---

## Evolução da Arquitetura

### Caminho típico de evolução

```
FASE 1: Início (Meses 1-3)
═══════════════════════════
Data Lake Simples
  ↓ Necessidade de ACID/time-travel

FASE 2: Lakehouse (Meses 3-6)
═══════════════════════════════
Data Lakehouse (Iceberg)
  ↓ Necessidade de dados mais frescos

FASE 3: CDC + Near Real-Time (Meses 6-12)
═══════════════════════════════════════════
CDC-Driven Platform + Lakehouse
  ↓ Necessidade de real-time para casos específicos

FASE 4: Hybrid (Meses 12-18)
═════════════════════════════
Lakehouse + Streaming (casos específicos)
  ↓ Múltiplos times consumindo dados

FASE 5: Governed Platform (Meses 18+)
═══════════════════════════════════════
Data Mesh / DataZone + Lakehouse + Streaming

REGRA: Comece simples, evolua com necessidade real, não antecipada.
```

---

## Diretrizes para Code Review assistido por AI

Ao revisar arquiteturas e configurações de data engineering na AWS, verifique:

1. **Over-engineering** — Streaming quando batch resolve; Data Mesh com 2 times; Redshift para 10GB
2. **Under-engineering** — CSV em produção; sem catálogo; sem quality; sem monitoring
3. **Sem orquestração** — Jobs executados manualmente ou por cron sem retry/alertas
4. **Sem idempotência** — Pipeline que duplica dados ao re-executar
5. **Formato errado** — CSV/JSON no curated zone; sugira Parquet/ORC/Iceberg
6. **Sem particionamento** — Tabelas no S3 sem partição → full scan caro
7. **Small files** — Milhares de arquivos < 1MB → sugira compaction
8. **Sem DLQ** — Records que falharam devem ir para dead letter queue
9. **Sem data quality** — Cada zona deve ter quality checks
10. **Sem cost tags** — Recursos sem tags impossibilitam chargeback
11. **Sem lifecycle** — Dados antigos sem lifecycle policy → custo crescente
12. **Vendor lock-in** — Formatos proprietários quando existem alternativas abertas
13. **Single point of failure** — DMS sem Multi-AZ; sem DR plan
14. **Hardcoded config** — Parameterize: bucket names, table names, thresholds

---

## Referências

- **AWS Well-Architected — Analytics Lens** — Framework oficial para data architecture
- **AWS Architecture Center — Analytics** — Diagramas de referência
- **Fundamentals of Data Engineering** — Joe Reis & Matt Housley — Bíblia moderna de DE
- **Designing Data-Intensive Applications** — Martin Kleppmann — Sistemas distribuídos
- **Data Mesh: Delivering Data-Driven Value at Scale** — Zhamak Dehghani
- **The Data Warehouse Toolkit** — Ralph Kimball — Modelagem dimensional
- **Apache Iceberg: The Definitive Guide** — Tomer Shiran, Jason Hughes — Lakehouse
- **AWS Big Data Blog** — Implementações e case studies AWS
