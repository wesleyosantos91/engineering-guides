# Data Engineering — Guia Completo (AWS)

> **Objetivo deste guia:** Servir como referência completa sobre **Data Engineering na AWS**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Foco em **AWS** — conceitos, motivações, heurísticas, boas práticas, diagramas e patterns de implementação com serviços AWS.
>
> **Público-alvo:** Engenheiros de dados, arquitetos de soluções e desenvolvedores projetando plataformas de dados na AWS.

---

## Quick Reference — Stack de Data Engineering na AWS

| Camada | Serviços AWS Principais | Função |
|--------|------------------------|--------|
| **Ingestão** | Kinesis, MSK, DMS, AppFlow, DataSync | Captura e movimentação de dados |
| **Armazenamento** | S3, Redshift, DynamoDB, Aurora, OpenSearch | Data lake, warehouse, bancos operacionais |
| **Processamento** | Glue, EMR, Lambda, Step Functions, MWAA | ETL batch, streaming, orquestração |
| **Integração** | DMS, EventBridge, AppFlow, API Gateway | CDC, eventos, integrações SaaS |
| **Governança** | Lake Formation, Glue Data Catalog, DataZone, Macie | Catálogo, acesso, qualidade, compliance |
| **Consumo** | Athena, Redshift, QuickSight | SQL ad-hoc, BI, analytics |

---

## Documentos

### 1. [Fundamentos de Arquitetura de Dados](01-data-architecture-foundations.md)

Conceitos fundamentais — **Data Lake**, **Data Warehouse**, **Data Lakehouse**, **Data Mesh** e **Data Fabric** — com o ecossistema AWS para dados e árvores de decisão.

- Ecossistema AWS para dados — mapa de serviços por categoria
- Data Lake — S3 como foundation, zonas raw/curated/refined
- Data Warehouse — Redshift, modelagem dimensional
- Data Lakehouse — Open-table formats (Iceberg, Delta) sobre S3
- Data Mesh — Ownership por domínio, dados como produto
- Data Fabric — Metadados ativos e integração automatizada

### 2. [Storage & Databases](02-data-storage.md)

Serviços de armazenamento e bancos de dados na AWS — **quando usar cada serviço**, configurações otimizadas, anti-patterns e heurísticas de decisão.

- S3 — Object store, tiers, lifecycle
- Redshift — Columnar DW para analytics
- DynamoDB — Key-value serverless de alta throughput
- Aurora — OLTP relacional compatível MySQL/PostgreSQL
- ElastiCache / MemoryDB — In-memory
- Neptune, Timestream, OpenSearch — Bancos especializados

### 3. [Data Processing](03-data-processing.md)

Processamento batch e streaming na AWS — **Glue, EMR, Kinesis, Lambda, Step Functions, MWAA** — patterns, boas práticas e quando usar cada um.

- AWS Glue — ETL serverless com Spark
- EMR / EMR Serverless — Spark/Flink com controle total
- Kinesis Data Streams / Firehose / Analytics — Streaming real-time
- Amazon MSK — Ecossistema Kafka na AWS
- Lambda — Transformações event-driven leves
- Step Functions / MWAA — Orquestração de pipelines

### 4. [Integração & Migração de Dados](04-data-integration.md)

CDC, migração, movimentação e integrações — **DMS, EventBridge, AppFlow, DataSync, Snow Family, SCT** — patterns e quando usar cada cenário.

- CDC — Change Data Capture com DMS + Kinesis
- DMS — Migração full load e ongoing replication
- SCT — Schema Conversion Tool (Oracle → Aurora)
- AppFlow — Integrações SaaS (Salesforce, SAP) → S3/Redshift
- EventBridge — Event routing entre aplicações
- DataSync / Snow Family — Transferência on-premises → AWS

### 5. [Governança, Segurança & Qualidade](05-data-governance.md)

Governança completa de dados — **Lake Formation, Glue Data Catalog, DataZone, Macie, KMS, Data Quality** — catálogo, acesso, compliance e observabilidade.

- Glue Data Catalog — Metadados, schemas, crawlers
- Lake Formation — Fine-grained access (column/row/cell)
- DataZone — Data marketplace e data products
- Macie — Detecção automática de PII/dados sensíveis
- Glue Data Quality — Rules e anomaly detection
- CloudTrail + Config — Auditoria e compliance (LGPD, GDPR, HIPAA)

### 6. [Arquiteturas de Referência & Anti-Patterns](06-architecture-patterns.md)

Arquiteturas de referência end-to-end, decision frameworks, anti-patterns e checklists de produção.

- Data Lake Simples — S3, Glue, Athena
- Data Lakehouse Moderno — S3, Iceberg, Athena, Redshift
- Real-Time Analytics — Kinesis/MSK + Flink
- CDC-Driven Data Platform — DMS + Kinesis + Glue
- Modern Data Stack (ELT) — Firehose + S3 + dbt + Redshift
- ML Feature Platform — SageMaker Feature Store

---

## Mapa de Decisão — Pipeline

```
Dados a serem processados
│
├─ Streaming (real-time)?
│   ├─ Kafka existente? → MSK / MSK Serverless
│   ├─ Apenas entrega no S3/Redshift? → Kinesis Firehose
│   ├─ Processamento complexo? → Kinesis Analytics (Flink) / EMR Serverless
│   └─ Transformação simples? → Lambda
│
├─ Batch (agendado)?
│   ├─ ETL padrão? → AWS Glue
│   ├─ Spark complexo / controle total? → EMR
│   ├─ Orquestração simples? → Step Functions
│   └─ DAGs complexos com dependências? → MWAA (Airflow)
│
├─ Migração de banco?
│   ├─ Mesmo engine? → DMS (full load + CDC)
│   └─ Engine diferente? → SCT + DMS
│
└─ Consulta ad-hoc?
    ├─ Dados no S3? → Athena
    └─ Dados estruturados alto volume? → Redshift
```

---

## Referências

- Reis, Joe & Housley, Matt. *Fundamentals of Data Engineering* (O'Reilly, 2022)
- Kleppmann, Martin. *Designing Data-Intensive Applications* (O'Reilly, 2017)
- AWS Well-Architected — [Data Analytics Lens](https://docs.aws.amazon.com/wellarchitected/latest/analytics-lens/analytics-lens.html)
- AWS Big Data Blog — [aws.amazon.com/blogs/big-data](https://aws.amazon.com/blogs/big-data/)
- AWS Architecture Center — [aws.amazon.com/architecture](https://aws.amazon.com/architecture/)
