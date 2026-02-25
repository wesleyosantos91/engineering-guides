# Data Engineering — Fundamentos de Arquitetura de Dados

> **Objetivo deste documento:** Servir como referência completa sobre **fundamentos de Data Architecture**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Foco em **AWS** — conceitos, motivações, heurísticas, boas práticas e diagramas ilustrativos com serviços AWS.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **Data Lake** | Armazene raw data no formato original, processe depois | Transformar dados na ingestão e perder o original | Manter zona raw imutável no S3 |
| **Data Warehouse** | Schema-on-write para dados estruturados e analíticos | Jogar dados não estruturados em tabelas relacionais | Usar data lake para raw, warehouse para curado |
| **Data Lakehouse** | Combine lake flexibility com warehouse performance | Duplicar dados entre lake e warehouse sem estratégia | Usar formatos open-table (Iceberg/Delta) sobre S3 |
| **Data Mesh** | Ownership por domínio, dados como produto | Time centralizado gargalo de todas as pipelines | Descentralizar ownership, manter governança federada |
| **Data Fabric** | Metadados ativos para integração automatizada | Integração manual ponto-a-ponto entre fontes | Usar Glue Data Catalog + Lake Formation como fabric layer |

---

## Sumário

- [Data Engineering — Fundamentos de Arquitetura de Dados](#data-engineering--fundamentos-de-arquitetura-de-dados)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que é Data Engineering?](#o-que-é-data-engineering)
  - [Pilares da Arquitetura de Dados](#pilares-da-arquitetura-de-dados)
  - [Ecossistema AWS para Dados](#ecossistema-aws-para-dados)
    - [Mapa de serviços por categoria](#mapa-de-serviços-por-categoria)
    - [Árvore de decisão — Qual serviço usar?](#árvore-de-decisão--qual-serviço-usar)
  - [Data Lake](#data-lake)
    - [Conceito](#conceito)
    - [Arquitetura Multi-Zone no S3](#arquitetura-multi-zone-no-s3)
    - [Organização de prefixos no S3](#organização-de-prefixos-no-s3)
    - [Boas Práticas AWS](#boas-práticas-aws)
    - [Anti-patterns](#anti-patterns)
  - [Data Warehouse](#data-warehouse)
    - [Conceito](#conceito-1)
    - [Amazon Redshift — Arquitetura](#amazon-redshift--arquitetura)
    - [Boas Práticas](#boas-práticas)
    - [Modelagem dimensional](#modelagem-dimensional)
  - [Data Lakehouse](#data-lakehouse)
    - [Conceito](#conceito-2)
    - [Implementação na AWS](#implementação-na-aws)
    - [Formatos Open Table](#formatos-open-table)
    - [Funcionalidades do Lakehouse](#funcionalidades-do-lakehouse)
  - [Data Mesh](#data-mesh)
    - [Os 4 Princípios](#os-4-princípios)
    - [Implementação na AWS](#implementação-na-aws-1)
    - [Quando usar](#quando-usar)
  - [Data Fabric](#data-fabric)
  - [Batch vs Stream vs Hybrid](#batch-vs-stream-vs-hybrid)
    - [Lambda Architecture (AWS)](#lambda-architecture-aws)
    - [Kappa Architecture (AWS)](#kappa-architecture-aws)
  - [ETL vs ELT](#etl-vs-elt)
  - [Maturidade de Dados](#maturidade-de-dados)
    - [Modelo de maturidade](#modelo-de-maturidade)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que é Data Engineering?

**Data Engineering** é a disciplina que projeta, constrói e mantém a infraestrutura e os pipelines que tornam os dados **confiáveis, acessíveis e utilizáveis** para toda a organização.

```
┌─────────────────────────────────────────────────────────────────┐
│                      DATA ENGINEERING                           │
│                                                                 │
│  Sources → Ingestion → Storage → Processing → Serving → Consume │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐│
│  │ Databases │  │ APIs     │  │ Events   │  │ Files/Logs       ││
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬──────────┘│
│       └──────────────┴─────────────┴────────────────┘           │
│                          │                                       │
│                    ┌─────▼──────┐                                │
│                    │ INGESTION  │  Kinesis, DMS, AppFlow,       │
│                    │            │  EventBridge, Glue             │
│                    └─────┬──────┘                                │
│                          │                                       │
│                    ┌─────▼──────┐                                │
│                    │  STORAGE   │  S3, Redshift, DynamoDB,      │
│                    │            │  RDS/Aurora                    │
│                    └─────┬──────┘                                │
│                          │                                       │
│                    ┌─────▼──────┐                                │
│                    │ PROCESSING │  Glue, EMR, Kinesis Analytics, │
│                    │            │  Lambda, Step Functions         │
│                    └─────┬──────┘                                │
│                          │                                       │
│                    ┌─────▼──────┐                                │
│                    │  SERVING   │  Athena, Redshift Spectrum,   │
│                    │            │  QuickSight, OpenSearch        │
│                    └────────────┘                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Pilares da Arquitetura de Dados

| Pilar | Descrição | Serviços AWS chave |
|-------|-----------|-------------------|
| **Confiabilidade** | Dados corretos, consistentes e completos | Glue Data Quality, CloudWatch, EventBridge |
| **Escalabilidade** | Cresce com o volume sem redesign | S3, Kinesis, Redshift Serverless, Glue |
| **Segurança** | Acesso controlado, criptografia, compliance | Lake Formation, KMS, IAM, Macie |
| **Custo** | Pay-per-use, tiering, lifecycle | S3 Intelligent-Tiering, Redshift Serverless, Athena |
| **Operabilidade** | Monitoramento, alertas, self-healing | CloudWatch, Step Functions, EventBridge |
| **Descobribilidade** | Dados fáceis de encontrar e entender | Glue Data Catalog, DataZone |

---

## Ecossistema AWS para Dados

### Mapa de serviços por categoria

```
┌──────────────────────────────────────────────────────────────────┐
│                    AWS DATA ECOSYSTEM                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  INGESTION              STORAGE              PROCESSING           │
│  ─────────              ───────              ──────────           │
│  Kinesis Data Streams   S3                   AWS Glue (ETL)      │
│  Kinesis Firehose       Redshift             EMR (Spark/Hive)    │
│  DMS                    DynamoDB             Kinesis Analytics    │
│  AppFlow                RDS / Aurora         Lambda              │
│  EventBridge            ElastiCache          Step Functions      │
│  MSK (Kafka)            Neptune              Athena              │
│  Transfer Family        Timestream           MWAA (Airflow)      │
│  DataSync               MemoryDB             EMR Serverless      │
│  Snow Family            OpenSearch                                │
│                                                                   │
│  ANALYTICS              GOVERNANCE           ML / AI              │
│  ─────────              ──────────           ─────                │
│  Athena                 Lake Formation       SageMaker            │
│  Redshift Spectrum      Glue Data Catalog    Bedrock              │
│  QuickSight             DataZone             Comprehend           │
│  OpenSearch             Macie                Forecast             │
│  Clean Rooms            CloudTrail           Personalize          │
│                         Config                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Árvore de decisão — Qual serviço usar?

| Cenário | Serviço recomendado | Por quê |
|---------|-------------------|---------|
| Armazenamento ilimitado de dados raw | **S3** | Custo baixo, durabilidade 11 9s, formato aberto |
| Data warehouse para analytics | **Redshift** | Columnar, massively parallel, SQL |
| Queries ad-hoc sobre S3 | **Athena** | Serverless, pay-per-query, suporta Iceberg |
| Stream processing em tempo real | **Kinesis Data Streams + Analytics** | Sub-segundo latência, managed |
| Stream processing com ecossistema Kafka | **MSK** | Compatibilidade Kafka nativa |
| ETL/ELT batch | **Glue** | Serverless Spark, crawlers, data catalog |
| ETL/ELT batch complexo | **EMR** | Spark/Hive/Presto com controle total |
| Orquestração de pipelines | **Step Functions / MWAA** | State machines / Airflow managed |
| Dados operacionais chave-valor | **DynamoDB** | Single-digit ms latency, serverless |
| Dados relacionais transacionais | **Aurora** | MySQL/PostgreSQL managed, auto-scaling |
| Cache de dados frequentes | **ElastiCache / MemoryDB** | Sub-ms latency, Redis/Memcached |
| Grafos e relacionamentos | **Neptune** | Graph queries, SPARQL/Gremlin |
| Séries temporais (IoT, métricas) | **Timestream** | Otimizado para time-series, retenção automática |
| Full-text search e logs | **OpenSearch** | Elasticsearch managed, dashboards |
| Visualização e dashboards | **QuickSight** | BI serverless, SPICE engine, ML insights |
| CDC de banco relacional | **DMS** | Replica ongoing changes, multi-engine |
| Integração SaaS → AWS | **AppFlow** | Salesforce, SAP, Zendesk → S3/Redshift |

---

## Data Lake

### Conceito

Um **Data Lake** é um repositório centralizado que armazena dados **em qualquer formato** (estruturado, semi-estruturado e não-estruturado) em seu **formato original**, permitindo processamento e análise posterior.

Na AWS, o Data Lake é construído sobre o **Amazon S3**.

### Arquitetura Multi-Zone no S3

```
┌──────────────────────────────────────────────────────────────┐
│                    DATA LAKE — S3                              │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐     │
│  │   RAW ZONE   │   │ CURATED ZONE │   │ REFINED ZONE │     │
│  │  (Landing)   │──▶│  (Cleaned)   │──▶│ (Analytics)  │     │
│  │              │   │              │   │              │      │
│  │ s3://lake/   │   │ s3://lake/   │   │ s3://lake/   │     │
│  │   raw/       │   │   curated/   │   │   refined/   │     │
│  │              │   │              │   │              │      │
│  │ • JSON, CSV  │   │ • Parquet    │   │ • Parquet    │     │
│  │ • Avro       │   │ • ORC        │   │ • Iceberg    │     │
│  │ • Logs       │   │ • Deduped    │   │ • Aggregated │     │
│  │ • Raw dumps  │   │ • Typed      │   │ • Joined     │     │
│  │ • Imutável   │   │ • Partitioned│   │ • Business   │     │
│  └──────────────┘   └──────────────┘   └──────────────┘     │
│                                                               │
│  ┌──────────────┐   ┌──────────────┐                         │
│  │  SANDBOX     │   │  ARCHIVE     │                         │
│  │  (Exploração)│   │  (Cold)      │                         │
│  │              │   │              │                         │
│  │ Data science │   │ Glacier      │                         │
│  │ experiments  │   │ Deep Archive │                         │
│  └──────────────┘   └──────────────┘                         │
└──────────────────────────────────────────────────────────────┘
```

### Organização de prefixos no S3

```
s3://company-data-lake/
├── raw/
│   ├── source=orders/
│   │   ├── year=2025/month=01/day=15/
│   │   │   └── orders-20250115-001.json
│   │   └── year=2025/month=01/day=16/
│   │       └── orders-20250116-001.json
│   ├── source=customers/
│   │   └── year=2025/month=01/day=15/
│   │       └── customers-full-20250115.csv
│   └── source=clickstream/
│       └── year=2025/month=01/day=15/hour=14/
│           └── clicks-20250115-14.parquet
├── curated/
│   ├── domain=sales/
│   │   └── entity=orders/
│   │       └── year=2025/month=01/
│   │           └── part-00001.parquet
│   └── domain=marketing/
│       └── entity=clickstream/
├── refined/
│   ├── star-schema/
│   │   ├── fact_sales/
│   │   └── dim_customer/
│   └── feature-store/
│       └── customer_features/
├── sandbox/
│   └── user=data-scientist-01/
└── archive/
    └── source=legacy-system/
```

### Boas Práticas AWS

| Prática | Detalhes |
|---------|---------|
| **Particionamento por data** | Use Hive-style: `year=YYYY/month=MM/day=DD/` para pruning eficiente |
| **Formato colunar** | Parquet ou ORC para curated/refined — 90% menos scan no Athena |
| **Compressão** | Snappy (Parquet default) ou ZSTD para melhor ratio |
| **Tamanho de arquivo** | 128MB–1GB por arquivo — evite small files problem |
| **Imutabilidade da raw zone** | Nunca altere/delete dados raw — é seu backup e audit trail |
| **Lifecycle policies** | Raw → IA após 90d, Archive → Glacier após 1 ano |
| **Encryption** | SSE-S3 (default) ou SSE-KMS para compliance |
| **Bucket policies** | Deny unencrypted uploads, deny public access |
| **Versionamento** | Ativo na raw zone para recovery de ingestão incorreta |
| **Event notifications** | S3 Events → EventBridge para trigger de pipelines |
| **Access logging** | S3 Server Access Logging ou CloudTrail Data Events |
| **Tags** | `environment`, `domain`, `classification`, `owner` em cada prefix |

### Anti-patterns

| Anti-pattern | Problema | Solução |
|-------------|----------|---------|
| **Data Swamp** | Dados sem catálogo, sem schema, sem owner — ninguém consegue usar | Glue Data Catalog obrigatório, tagging, data quality checks |
| **Small Files Hell** | Milhares de arquivos pequenos (< 1MB) | Glue compaction jobs, Firehose buffer size, Iceberg compaction |
| **Sem particionamento** | Full scan em toda query | Particionar por data ou campo de alta cardinalidade |
| **Formato raw em produção** | CSV/JSON em queries analíticas — lento e caro | Converter para Parquet/ORC na curated zone |
| **Sem lifecycle** | Custos crescentes com dados nunca acessados | S3 Lifecycle Rules, Intelligent-Tiering |
| **Single bucket** | Sem separação de ambientes/domínios | Bucket por estágio (raw/curated/refined) ou por domínio |

---

## Data Warehouse

### Conceito

Um **Data Warehouse** é um repositório otimizado para **análise** de dados estruturados, com schema-on-write, modelagem dimensional e queries SQL de alta performance.

### Amazon Redshift — Arquitetura

```
┌────────────────────────────────────────────────────────┐
│                    AMAZON REDSHIFT                       │
├────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────────────────────────────────────┐      │
│  │             LEADER NODE                       │      │
│  │  • Query planning e optimization              │      │
│  │  • Distribui queries aos compute nodes        │      │
│  │  • Agrega resultados                          │      │
│  └──────────────────┬───────────────────────────┘      │
│                     │                                    │
│         ┌───────────┼───────────┐                       │
│         │           │           │                       │
│  ┌──────▼─────┐ ┌──────▼──────┐ ┌──────▼──────┐       │
│  │ COMPUTE    │ │ COMPUTE     │ │ COMPUTE     │       │
│  │ NODE 1     │ │ NODE 2      │ │ NODE N      │       │
│  │            │ │             │ │             │       │
│  │ ┌────────┐ │ │ ┌────────┐  │ │ ┌────────┐  │       │
│  │ │Slice 1 │ │ │ │Slice 1 │  │ │ │Slice 1 │  │       │
│  │ │Slice 2 │ │ │ │Slice 2 │  │ │ │Slice 2 │  │       │
│  │ └────────┘ │ │ └────────┘  │ │ └────────┘  │       │
│  └────────────┘ └─────────────┘ └─────────────┘       │
│                                                         │
│  REDSHIFT SPECTRUM                                      │
│  ┌──────────────────────────────────────────────┐      │
│  │  Query data diretamente no S3                 │      │
│  │  sem precisar carregar no Redshift             │      │
│  │  Exabyte-scale, pay-per-scan                  │      │
│  └──────────────────────────────────────────────┘      │
│                                                         │
│  REDSHIFT SERVERLESS                                    │
│  ┌──────────────────────────────────────────────┐      │
│  │  Auto-scaling, pay-per-query                  │      │
│  │  Sem gerenciamento de cluster                 │      │
│  │  Ideal para workloads variáveis               │      │
│  └──────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────┘
```

### Boas Práticas

| Prática | Detalhes |
|---------|---------|
| **Distribution style** | KEY para joins frequentes, EVEN para balanced, ALL para pequenas dimensões |
| **Sort keys** | Compound para queries com filtro fixo, Interleaved para ad-hoc |
| **Compression** | AZ64 para numéricos, LZO para strings — Redshift auto-detecta |
| **COPY command** | Prefira COPY do S3 (paralelo) vs INSERT (row-by-row) |
| **Workload Management (WLM)** | Configure filas por prioridade: loading, reporting, ad-hoc |
| **Vacuum & Analyze** | Agende VACUUM FULL e ANALYZE para manter performance |
| **Concurrency Scaling** | Ative para picos de queries — escala automaticamente |
| **Materialized Views** | Use para queries repetitivas — refresh incremental |
| **Spectrum para cold data** | Mantenha dados antigos no S3, acesse via Spectrum |
| **Redshift Serverless** | Prefira para workloads imprevisíveis ou times pequenos |
| **Data sharing** | Compartilhe dados entre clusters sem copiar |

### Modelagem dimensional

| Modelo | Descrição | Quando usar |
|--------|-----------|-------------|
| **Star Schema** | Fato central + dimensões desnormalizadas | Default para analytics — simples e performático |
| **Snowflake Schema** | Dimensões normalizadas | Quando storage é prioridade sobre query speed |
| **Data Vault** | Hub + Link + Satellite | Quando flexibilidade e auditabilidade são essenciais |
| **Wide Table** | Tabela desnormalizada única | Dashboards específicos, feature store |

---

## Data Lakehouse

### Conceito

O **Data Lakehouse** combina a **flexibilidade do Data Lake** (storage barato, formatos abertos) com as **garantias do Data Warehouse** (ACID transactions, schema enforcement, time-travel).

### Implementação na AWS

```
┌──────────────────────────────────────────────────────────┐
│                    DATA LAKEHOUSE — AWS                    │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────────────────────────────────────┐        │
│  │              Amazon S3 (Storage)              │        │
│  │        Open Table Format sobre S3              │        │
│  │  ┌────────────┐ ┌────────┐ ┌────────────┐    │        │
│  │  │ Apache     │ │ Delta  │ │ Apache     │    │        │
│  │  │ Iceberg    │ │ Lake   │ │ Hudi       │    │        │
│  │  └────────────┘ └────────┘ └────────────┘    │        │
│  └──────────────────┬───────────────────────────┘        │
│                     │                                     │
│  ┌──────────────────▼───────────────────────────┐        │
│  │           Query Engines                       │        │
│  │  ┌────────┐ ┌──────────┐ ┌──────────────┐   │        │
│  │  │ Athena │ │ Redshift │ │ EMR (Spark)  │   │        │
│  │  │        │ │ Spectrum  │ │              │   │        │
│  │  └────────┘ └──────────┘ └──────────────┘   │        │
│  └──────────────────────────────────────────────┘        │
│                                                           │
│  ┌──────────────────────────────────────────────┐        │
│  │           Governance Layer                    │        │
│  │  ┌────────────────┐ ┌──────────────────┐     │        │
│  │  │ Glue Data      │ │ Lake Formation   │     │        │
│  │  │ Catalog        │ │ (Access Control) │     │        │
│  │  └────────────────┘ └──────────────────┘     │        │
│  └──────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────┘
```

### Formatos Open Table

| Formato | Criador | Suporte AWS | Destaque |
|---------|---------|-------------|----------|
| **Apache Iceberg** | Netflix/Apache | Athena, EMR, Glue, Redshift | **Recomendado na AWS** — melhor integração nativa |
| **Delta Lake** | Databricks | EMR, Glue | Forte ecossistema Spark |
| **Apache Hudi** | Uber/Apache | EMR, Glue | Melhor para CDC e upserts incrementais |

### Funcionalidades do Lakehouse

| Feature | Benefício | Como na AWS |
|---------|-----------|-------------|
| **ACID transactions** | Leituras consistentes, writes atômicos | Iceberg commit log no S3 |
| **Schema evolution** | Adicionar/renomear colunas sem rewrite | Iceberg schema evolution via Glue |
| **Time travel** | Consultar dados de qualquer ponto no tempo | `SELECT * FROM table FOR TIMESTAMP AS OF '2025-01-01'` |
| **Partition evolution** | Mudar particionamento sem rewrite | Iceberg hidden partitioning |
| **Compaction** | Consolidar small files automaticamente | Glue Iceberg compaction job |
| **Row-level deletes** | GDPR compliance — deletar registros | Iceberg merge-on-read ou copy-on-write |

---

## Data Mesh

### Os 4 Princípios

```
┌──────────────────────────────────────────────────────────────┐
│                       DATA MESH                               │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. DOMAIN OWNERSHIP               2. DATA AS A PRODUCT      │
│  ┌─────────────────────┐           ┌─────────────────────┐   │
│  │ Cada domínio é dono │           │ Dados com SLA,      │   │
│  │ dos seus dados       │           │ qualidade, docs,    │   │
│  │                      │           │ descobribilidade    │   │
│  │ Time de Vendas →     │           │                     │   │
│  │   dados de vendas    │           │ Tratados como um    │   │
│  │ Time de Marketing →  │           │ produto para        │   │
│  │   dados de marketing │           │ consumidores        │   │
│  └─────────────────────┘           └─────────────────────┘   │
│                                                               │
│  3. SELF-SERVE PLATFORM            4. FEDERATED GOVERNANCE   │
│  ┌─────────────────────┐           ┌─────────────────────┐   │
│  │ Plataforma que       │           │ Governança global   │   │
│  │ abstrai complexidade │           │ + autonomia local   │   │
│  │                      │           │                     │   │
│  │ Templates, CI/CD,   │           │ Standards globais:  │   │
│  │ infra compartilhada  │           │ naming, security,   │   │
│  │ para data products   │           │ quality, formats    │   │
│  └─────────────────────┘           └─────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### Implementação na AWS

| Princípio | Serviços AWS | Implementação |
|-----------|-------------|---------------|
| **Domain Ownership** | AWS Organizations, contas separadas por domínio | Cada domínio tem sua conta AWS com S3, Glue, Redshift |
| **Data as a Product** | DataZone, Glue Data Catalog | DataZone para publicar/descobrir data products com SLAs |
| **Self-Serve Platform** | Service Catalog, CDK, Terraform modules | Templates de infra para domínios criarem data products |
| **Federated Governance** | Lake Formation, DataZone policies | Políticas centrais + autonomia local por domínio |

### Quando usar

| Cenário | Data Mesh? | Alternativa |
|---------|-----------|-------------|
| < 5 domínios de dados | ❌ Over-engineering | Data Lake centralizado com Lake Formation |
| Time central de dados = gargalo | ✅ Sim | — |
| > 10 times produzindo dados | ✅ Sim | — |
| Startup / time pequeno | ❌ Complexidade desnecessária | Data Lake + Glue simples |
| Empresa com múltiplas BUs | ✅ Sim | — |
| Dados com ownership claro por domínio | ✅ Sim | — |

---

## Data Fabric

**Data Fabric** é uma abordagem **centrada em metadados** onde uma camada inteligente automatiza a integração, governança e acesso a dados distribuídos.

| Aspecto | Data Mesh | Data Fabric |
|---------|-----------|-------------|
| **Abordagem** | Organizacional (people-first) | Tecnológica (automation-first) |
| **Ownership** | Descentralizado por domínio | Pode ser centralizado |
| **Driver** | Cultura e organização | Metadados e AI/ML |
| **Governança** | Federada | Automatizada via metadados |
| **AWS** | DataZone + contas por domínio | Glue Data Catalog + Lake Formation + ML |

> **Dica:** Na prática, muitas organizações usam elementos de ambos. Data Mesh para organização + Data Fabric para automação técnica.

---

## Batch vs Stream vs Hybrid

| Aspecto | Batch | Stream | Hybrid (Lambda/Kappa) |
|---------|-------|--------|-----------------------|
| **Latência** | Minutos a horas | Milissegundos a segundos | Ambos |
| **Volume** | Alto (bulk) | Contínuo (event-by-event) | Ambos |
| **Complexidade** | Baixa | Alta | Muito alta |
| **Custo** | Menor (spot, scheduling) | Maior (always-on) | Intermediário |
| **AWS Batch** | Glue, EMR, Step Functions | Kinesis, MSK, Lambda | Glue + Kinesis |
| **Quando usar** | Reports diários, ETL noturno | Alertas, fraud detection, real-time dashboards | Quando precisa dos dois |

### Lambda Architecture (AWS)

```
┌─────────────────────────────────────────────────────────────┐
│                   LAMBDA ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Source ──────┬────────────────────────────────────┐        │
│               │                                    │        │
│         ┌─────▼──────┐                    ┌───────▼──────┐ │
│         │ BATCH LAYER │                   │ SPEED LAYER  │ │
│         │             │                   │              │ │
│         │ Glue ETL    │                   │ Kinesis      │ │
│         │ EMR jobs    │                   │ Analytics    │ │
│         │ Scheduled   │                   │ Lambda       │ │
│         │             │                   │ Real-time    │ │
│         └─────┬──────┘                    └──────┬──────┘ │
│               │                                   │        │
│         ┌─────▼──────┐                    ┌──────▼──────┐ │
│         │ BATCH VIEW  │                   │ REAL-TIME   │ │
│         │ (Redshift)   │                  │ VIEW        │ │
│         │              │                  │ (DynamoDB)  │ │
│         └──────┬──────┘                   └──────┬─────┘  │
│                │                                  │        │
│                └──────────────┬───────────────────┘        │
│                        ┌─────▼──────┐                      │
│                        │ SERVING    │                      │
│                        │ LAYER      │                      │
│                        │ (Merge)    │                      │
│                        └────────────┘                      │
└─────────────────────────────────────────────────────────────┘
```

### Kappa Architecture (AWS)

```
Source ──▶ Kinesis/MSK ──▶ Flink/Kinesis Analytics ──▶ Serving (Redshift/DynamoDB)
                │
                └──▶ S3 (archival / replay)

Tudo é stream. Batch é um caso especial de stream (reprocessamento).
```

---

## ETL vs ELT

| Aspecto | ETL | ELT |
|---------|-----|-----|
| **Ordem** | Extract → Transform → Load | Extract → Load → Transform |
| **Onde transforma** | No pipeline (Glue, EMR) | No destino (Redshift, Athena) |
| **Quando usar** | Dados precisam de limpeza antes de carregar | Storage é barato, compute no destino é potente |
| **AWS pattern** | Glue ETL → Redshift | Firehose → S3 → Athena/dbt |
| **Vantagem** | Dados limpos no destino | Flexibilidade, raw preservado |
| **Desvantagem** | Perda do raw, pipeline complexo | Requer compute potente no destino |

> **Tendência de mercado:** ELT está dominando com a popularização de data lakehouses. Raw no S3, transformação com dbt/Glue/Athena.

---

## Maturidade de Dados

### Modelo de maturidade

| Nível | Nome | Características | AWS típico |
|-------|------|----------------|-----------|
| **1** | Reativo | Planilhas, extrações manuais, sem pipeline | RDS + exports manuais |
| **2** | Gerenciado | ETL básico, warehouse, reports agendados | Glue + Redshift + QuickSight |
| **3** | Proativo | Data Lake, catálogo, qualidade, SLAs | S3 + Lake Formation + Glue Data Quality |
| **4** | Otimizado | Lakehouse, streaming, ML integrado, self-serve | Iceberg + Kinesis + SageMaker + DataZone |
| **5** | Inteligente | Data Mesh, AI-driven governance, tempo real em tudo | DataZone + Bedrock + fully event-driven |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e configurações de data engineering na AWS, verifique:

1. **S3 bucket sem encryption** — Todo bucket deve ter SSE-S3 ou SSE-KMS habilitado
2. **S3 bucket com public access** — Block Public Access deve estar ativo em 100% dos buckets de dados
3. **Dados em formato CSV/JSON na camada analítica** — Sugira conversão para Parquet/ORC
4. **Glue jobs sem bookmark** — Job bookmarks evitam reprocessamento; devem estar ativos
5. **Sem particionamento em tabelas do Glue Catalog** — Sugira particionamento por data
6. **Redshift sem sort key** — Tabelas acessadas frequentemente precisam de sort key
7. **Redshift COPY sem manifest/IAM role** — Use manifest para controle e IAM role (nunca credentials)
8. **Kinesis sem enhanced fan-out** — Para múltiplos consumers, sugira enhanced fan-out
9. **Sem lifecycle policy em S3** — Dados não acessados devem migrar para IA/Glacier
10. **Sem Glue Data Catalog** — Todo dataset deve ser catalogado para descobribilidade
11. **Small files no S3** — Se detectar padrão de arquivos < 1MB, sugira compaction
12. **Sem tags em recursos de dados** — Exija `domain`, `classification`, `owner`, `environment`

---

## Referências

- **AWS Well-Architected — Analytics Lens** — Framework oficial AWS para cargas analíticas
- **Fundamentals of Data Engineering** — Joe Reis & Matt Housley — Bíblia moderna de DE
- **Data Mesh: Delivering Data-Driven Value at Scale** — Zhamak Dehghani — Conceitos de Data Mesh
- **The Data Warehouse Toolkit** — Ralph Kimball — Modelagem dimensional clássica
- **Building the Data Lakehouse** — Bill Inmon — Evolução para lakehouse
- **AWS Big Data Blog** — Implementações práticas e case studies
- **Apache Iceberg Documentation** — Formato open-table recomendado pela AWS
