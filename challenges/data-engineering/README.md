# Data Engineering — Engineering Challenges

> **Programa de especialização progressiva em Data Engineering** — do conceito à plataforma completa.
> Cada desafio exige: **ADR documentando decisões**, **diagrama DrawIO** e **implementação com AWS (LocalStack) + Python/Spark + Go + Java**.

**Referência:** [Data Engineering — Guia Completo](../../.docs/data-engineering/README.md)
**ADR Guide:** [Architecture Decision Records](../../.docs/adrs/README.md)

---

## Filosofia

```
Conceito → ADR → Diagrama DrawIO → Implementação (LocalStack + Código) → Validação → Review
```

Cada desafio segue o ciclo:
1. **Estude** o conceito na documentação de referência
2. **Escreva um ADR** documentando decisões de arquitetura de dados
3. **Desenhe no DrawIO** o pipeline/arquitetura com todos os componentes
4. **Implemente** usando LocalStack (serviços AWS) + código em Python/PySpark, Go e/ou Java
5. **Valide** com dados reais (datasets públicos) e métricas de qualidade
6. **Documente** lineage, SLAs e custos estimados

---

## Stack Tecnológica

### Infraestrutura (100% local via LocalStack + Docker)

| Serviço AWS | Equivalente Local | Uso |
|---|---|---|
| S3 | LocalStack S3 / MinIO | Data lake, staging, archival |
| Kinesis | LocalStack Kinesis | Streaming ingestão |
| DynamoDB | LocalStack DynamoDB | Metadata store, state |
| SQS/SNS | LocalStack SQS/SNS | Eventos, dead letter queues |
| Glue | PySpark local | ETL batch |
| Athena | Trino / DuckDB | SQL-on-S3, queries ad-hoc |
| Redshift | PostgreSQL (analytics) | Data warehouse |
| Step Functions | LocalStack SF / Prefect | Orchestration |
| Lambda | LocalStack Lambda | Event-driven transforms |
| EventBridge | LocalStack EB | Event routing |

### Linguagens e Frameworks

| Linguagem | Frameworks / Libs | Uso |
|---|---|---|
| **Python 3.12+** | PySpark 3.5, Pandas, boto3, Great Expectations, dbt | ETL, data quality, transformações |
| **Go 1.24+** | aws-sdk-go-v2, pgx, kafka-go, parquet-go | Ingestão, CDC producers, APIs de dados |
| **Java 25+** | Spring Boot, Spring Batch, Apache Flink, Iceberg SDK | Batch processing, streaming, connectors |
| **SQL** | PostgreSQL, Trino, DuckDB | Analytics, modeling, queries |

### Formatos de Dados

| Formato | Tipo | Uso |
|---|---|---|
| **Parquet** | Columnar, comprimido | Tabelas fato/dimensão, analytics |
| **Avro** | Row-based, schema evolution | Streaming, CDC events |
| **JSON / JSONL** | Semi-structured | Raw data, APIs |
| **CSV** | Tabular simples | Ingestão legada |
| **Apache Iceberg** | Open table format | Lakehouse (ACID sobre S3) |
| **Delta Lake** | Open table format | Lakehouse alternativo |

---

## Estrutura de Entregáveis

```
data-engineering-challenges/
├── level-NN-topic/
│   ├── README.md
│   ├── docs/
│   │   ├── adrs/
│   │   │   └── ADR-001-decision.md        ← MADR format
│   │   └── diagrams/
│   │       └── pipeline.drawio            ← DrawIO diagram
│   ├── infra/
│   │   ├── docker-compose.yml             ← LocalStack + deps
│   │   ├── localstack/
│   │   │   └── init-aws.sh               ← Provisiona S3, Kinesis, etc.
│   │   └── terraform/                     ← (opcional) IaC local
│   ├── python/
│   │   ├── etl/                           ← PySpark jobs
│   │   ├── quality/                       ← Great Expectations
│   │   ├── dbt/                           ← dbt models
│   │   └── requirements.txt
│   ├── go/
│   │   ├── cmd/
│   │   ├── internal/
│   │   ├── go.mod
│   │   └── Makefile
│   └── java/
│       ├── src/
│       ├── pom.xml
│       └── mvnw
```

---

## Mapa de Progressão

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║                   DATA ENGINEERING — MAPA DE PROGRESSÃO                            ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                    ║
║  PARTE I — Fundamentos (conceitos + ambiente + primeiros pipelines)                ║
║  ┌───────────────────────────────────────────────────────────────────────────────┐  ║
║  │  00. Foundations & Environment Setup                                         │  ║
║  │  01. Data Lake & Data Warehouse Foundations                                  │  ║
║  │  02. Data Modeling & Formats                                                 │  ║
║  └───────────────────────────────────────────────────────────────────────────────┘  ║
║                                      │                                             ║
║                                      ▼                                             ║
║  PARTE II — Data Processing (batch + streaming + orchestration)                    ║
║  ┌───────────────────────────────────────────────────────────────────────────────┐  ║
║  │  03. Batch Processing (ETL/ELT)                                              │  ║
║  │  04. Stream Processing (Real-Time)                                           │  ║
║  │  05. Pipeline Orchestration                                                  │  ║
║  └───────────────────────────────────────────────────────────────────────────────┘  ║
║                                      │                                             ║
║                                      ▼                                             ║
║  PARTE III — Data Integration (CDC, migração, eventos)                             ║
║  ┌───────────────────────────────────────────────────────────────────────────────┐  ║
║  │  06. CDC & Change Data Capture                                               │  ║
║  │  07. Data Migration & Integration Patterns                                   │  ║
║  └───────────────────────────────────────────────────────────────────────────────┘  ║
║                                      │                                             ║
║                                      ▼                                             ║
║  PARTE IV — Governança & Qualidade                                                 ║
║  ┌───────────────────────────────────────────────────────────────────────────────┐  ║
║  │  08. Data Quality & Validation                                               │  ║
║  │  09. Governance, Catalog & Lineage                                           │  ║
║  └───────────────────────────────────────────────────────────────────────────────┘  ║
║                                      │                                             ║
║                                      ▼                                             ║
║  PARTE V — Arquiteturas de Referência                                              ║
║  ┌───────────────────────────────────────────────────────────────────────────────┐  ║
║  │  10. Data Lakehouse & Open Table Formats                                     │  ║
║  │  11. Modern Data Stack & Analytics Platform                                  │  ║
║  └───────────────────────────────────────────────────────────────────────────────┘  ║
║                                      │                                             ║
║                                      ▼                                             ║
║  CAPSTONE                                                                          ║
║  ┌───────────────────────────────────────────────────────────────────────────────┐  ║
║  │  12. Capstone — Enterprise Data Platform "DataFlow"                          │  ║
║  └───────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                    ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

---

## Níveis por Desafio

| Level | Título | ADRs | DrawIO | Ref. Doc | Foco |
|:-----:|--------|:----:|:------:|----------|------|
| 00 | [Foundations & Setup](00-foundations.md) | 1 | 1 | README | Ambiente, tooling, conceitos |
| 01 | [Data Lake & Warehouse](01-data-lake-warehouse.md) | 2 | 2 | 01, 02 | S3 zones, Redshift basics |
| 02 | [Data Modeling & Formats](02-data-modeling-formats.md) | 2 | 1 | 01, 02 | Star schema, Parquet, Avro |
| 03 | [Batch Processing](03-batch-processing.md) | 3 | 2 | 03 | PySpark ETL, ELT, dbt |
| 04 | [Stream Processing](04-stream-processing.md) | 3 | 2 | 03 | Kinesis, Kafka, Flink |
| 05 | [Pipeline Orchestration](05-pipeline-orchestration.md) | 2 | 2 | 03 | Airflow, Step Functions, DAGs |
| 06 | [CDC & Change Data Capture](06-cdc.md) | 3 | 2 | 04 | Debezium, DMS, outbox |
| 07 | [Migration & Integration](07-migration-integration.md) | 2 | 2 | 04 | Schema evolution, federation |
| 08 | [Data Quality & Validation](08-data-quality.md) | 2 | 2 | 05 | Great Expectations, circuit breaker |
| 09 | [Governance, Catalog & Lineage](09-governance.md) | 3 | 2 | 05 | Catalog, lineage, compliance |
| 10 | [Lakehouse & Open Table Formats](10-lakehouse.md) | 3 | 2 | 06 | Iceberg, Delta, time travel |
| 11 | [Modern Data Stack](11-modern-data-stack.md) | 2 | 2 | 06 | dbt, ELT, analytics platform |
| 12 | [Capstone — DataFlow Platform](12-capstone-dataflow.md) | 6 | 4 | ALL | Plataforma completa |
| | **TOTAL** | **34** | **26** | | |

---

## Template ADR (MADR)

```markdown
# ADR-NNN: [Título da decisão]

## Status
Accepted

## Context
[Cenário de data engineering que motivou a decisão]

## Decision Drivers
- [Requisito não-funcional 1]
- [Volume / velocidade de dados]
- [Custo operacional]
- [Maturidade do time]

## Considered Options
1. [Option A] — description
2. [Option B] — description
3. [Option C] — description

## Decision Outcome
Chosen option: "[Option X]", because [justification]

### Consequences
- Good: [benefício]
- Bad: [trade-off aceito]

## Data Architecture Impact
- **Storage:** [impacto em armazenamento]
- **Processing:** [impacto em processamento]
- **Latency:** [impacto em latência]
- **Cost:** [impacto em custo estimado]
```

---

## DrawIO Conventions

| Layer | Cor | Uso |
|---|---|---|
| Sources (origens de dados) | 🔵 Azul | Databases, APIs, SaaS, files |
| Ingestion | 🟢 Verde | Kinesis, DMS, AppFlow, CDC |
| Storage | 🟡 Amarelo | S3, Redshift, DynamoDB |
| Processing | 🟠 Laranja | Spark, Flink, Lambda, Glue |
| Orchestration | 🟣 Roxo | Airflow, Step Functions |
| Serving / Consumption | 🔴 Vermelho | Athena, QuickSight, APIs |
| Governance | ⚪ Cinza | Catalog, quality, lineage |

---

## Definição de Pronto Global

Cada nível está completo quando:

- [ ] ADR(s) no formato MADR com data architecture impact
- [ ] Diagrama(s) DrawIO seguindo convenções de cores
- [ ] Pipeline funcional com LocalStack (S3, Kinesis, etc.)
- [ ] Código testado (unit + integration com Testcontainers/LocalStack)
- [ ] Dados de exemplo processados end-to-end
- [ ] Métricas de qualidade documentadas (volumes, latência, completude)
- [ ] Estimativa de custo AWS documentada
- [ ] Commit semântico: `feat(data-engineering-NN): ...`

---

## Pré-requisitos

| Requisito | Versão | Verificação |
|---|---|---|
| Docker Desktop | ≥ 4.x | `docker --version` |
| Python | ≥ 3.12 | `python --version` |
| PySpark | ≥ 3.5 | `pyspark --version` |
| Go | ≥ 1.24 | `go version` |
| Java (JDK) | ≥ 25 | `java --version` |
| LocalStack | ≥ 3.x | `localstack --version` |
| awscli-local | latest | `awslocal --version` |
| Terraform | ≥ 1.7 | `terraform --version` |
| DuckDB | ≥ 1.x | `duckdb --version` |
| dbt-core | ≥ 1.8 | `dbt --version` |
| Great Expectations | ≥ 1.x | `great_expectations --version` |

---

## Referências

- Reis, Joe & Housley, Matt. *Fundamentals of Data Engineering* (O'Reilly, 2022)
- Kleppmann, Martin. *Designing Data-Intensive Applications* (O'Reilly, 2017)
- AWS Well-Architected — [Data Analytics Lens](https://docs.aws.amazon.com/wellarchitected/latest/analytics-lens/)
- [Apache Iceberg](https://iceberg.apache.org/)
- [dbt documentation](https://docs.getdbt.com/)
- [Great Expectations docs](https://docs.greatexpectations.io/)
