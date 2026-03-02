# Databases — Guia Completo

> **Objetivo deste guia:** Servir como referência completa sobre **Bancos de Dados (SQL e NoSQL)**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Cobre fundamentos teóricos (CAP, PACELC, PIE, ACID/BASE), boas práticas SQL e NoSQL, modelagem, indexação, escalabilidade, performance e patterns arquiteturais.
>
> **Público-alvo:** Engenheiros de software, arquitetos de soluções e desenvolvedores que projetam, implementam e operam sistemas com bancos de dados.

---

## Quick Reference — Banco Certo para Cada Caso

| Workload | Banco recomendado | Modelo | Por quê |
|----------|------------------|--------|---------|
| **Transações ACID** | PostgreSQL, MySQL, Aurora | Relacional | Strong consistency, constraints, JOINs |
| **Alta throughput key-value** | DynamoDB, Redis | Key-Value | Single-digit ms, auto-scaling |
| **Documents flexíveis** | MongoDB, DocumentDB | Document | Schema evolution, dados semi-estruturados |
| **Escritas massivas** | Cassandra, ScyllaDB | Wide-Column | LSM-tree, write-optimized, escala linear |
| **Relações complexas** | Neo4j, Neptune | Graph | Traversal eficiente, shortest path |
| **Time-series / IoT** | TimescaleDB, InfluxDB | Time-Series | Compressão temporal, aggregations |
| **Full-text search** | Elasticsearch, OpenSearch | Search | Inverted index, relevância |
| **Cache / sessões** | Redis, Memcached | Key-Value | Sub-millisecond, TTL nativo |
| **Busca semântica** | Pinecone, pgvector | Vector | Similarity search, embeddings |
| **Analytics / OLAP** | ClickHouse, Redshift | Columnar | Aggregations, compressão colunar |

---

## Documentos

### 1. [Fundamentos de Banco de Dados](01-database-foundations.md)

Teoremas fundamentais e conceitos base — **CAP**, **PACELC**, **PIE**, **ACID vs BASE**, modelos de dados e decisões arquiteturais.

- **Teorema CAP** — Consistency, Availability, Partition Tolerance: trade-offs e classificação de bancos
- **Teorema PACELC** — Extensão do CAP: Latency vs Consistency em condições normais
- **Teorema PIE** — The Iron Triangle of Purpose: Pattern Flexibility, Infinite Scale, Efficiency (Pick Two)
- **ACID vs BASE** — Garantias transacionais: quando usar cada modelo
- **Modelos de Dados** — Relacional, Document, Key-Value, Wide-Column, Graph, Time-Series, Vector
- **Níveis de Consistência** — Linearizability → Eventual Consistency: espectro completo
- **Árvore de Decisão** — Guia para escolher o tipo de banco baseado no workload

### 2. [SQL & Bancos Relacionais — Boas Práticas](02-sql-best-practices.md)

Boas práticas para bancos relacionais — **modelagem**, **indexação**, **query optimization**, **transactions** e operações.

- **Normalização** — 1NF a BCNF: regras, quando desnormalizar
- **Modelagem** — Convenções de nomenclatura, tipos de dados, constraints
- **Indexação** — B-Tree, GIN, GiST, BRIN, composite indexes, regra ESR
- **Query Optimization** — EXPLAIN, keyset pagination, CTEs, anti-patterns
- **Transactions** — Níveis de isolamento, optimistic/pessimistic locking, deadlocks
- **Connection Management** — Pool sizing (HikariCP, PgBouncer, RDS Proxy)
- **Particionamento** — Range, List, Hash partitioning
- **Replicação** — Primary-Replica, síncrona vs assíncrona
- **Migrations** — Zero-downtime, expand-and-contract pattern
- **OLTP vs OLAP** — Star Schema, modelagem dimensional
- **Monitoramento** — Métricas, queries de diagnóstico PostgreSQL

### 3. [NoSQL — Boas Práticas](03-nosql-best-practices.md)

Boas práticas para bancos NoSQL — **Document**, **Key-Value**, **Wide-Column**, **Graph** — modelagem por access patterns.

- **Modelagem NoSQL** — Access patterns first, embedding vs referencing
- **MongoDB/DocumentDB** — Schema design, indexação, aggregation pipeline
- **DynamoDB** — Single-table design, partition key, GSI, capacity modes
- **Redis** — Data structures, caching patterns, distributed lock, rate limiter
- **Cassandra/ScyllaDB** — Query-driven modeling, partition sizing, consistency tuning
- **Neo4j/Neptune** — Graph modeling, Cypher queries, community detection
- **Consistency Tuning** — Configuração per-query por banco
- **Schema Evolution** — Version field, additive changes, dual-write migration

### 4. [Escalabilidade, Performance & Patterns](04-scalability-patterns.md)

Patterns de escalabilidade e performance — **sharding**, **caching**, **CQRS**, **event sourcing** e estratégias para escala infinita.

- **Vertical vs Horizontal Scaling** — Sequência recomendada de escala
- **Sharding** — Range, Hash, Consistent Hashing, Geo-based, shard key selection
- **Caching Patterns** — Cache-Aside, Write-Through, Write-Behind, Read-Through
- **Cache Problems** — Stampede, penetration, avalanche, hot key, invalidation
- **Replication Patterns** — Single-Leader, Multi-Leader, Leaderless
- **Polyglot Persistence** — Banco certo para cada workload
- **CQRS** — Command Query Responsibility Segregation com CDC
- **Event Sourcing** — Append-only event store, projeções
- **Database per Service** — Isolamento em microservices
- **Infinite Scale** — Serverless DBs, tiered architecture, global distribution
- **Efficiency** — Right-sizing, storage optimization, reserved capacity
- **Flexibility** — Schema evolution, multi-model DBs, data abstraction
- **Disaster Recovery** — RPO/RTO tiers, backup strategies, restore testing

---

## Mapa de Conceitos

```
┌──────────────────────────────────────────────────────────────────┐
│                    DATABASE KNOWLEDGE MAP                         │
│                                                                  │
│  Fundamentos                                                     │
│  ├── Teoremas: CAP, PACELC                                      │
│  ├── Garantias: ACID, BASE                                      │
│  ├── Modelos: Relacional, Document, Key-Value, Graph, ...       │
│  └── Consistência: Strong ◄──────────► Eventual                 │
│                                                                  │
│  SQL (Relacional)                                                │
│  ├── Modelagem: Normalização (3NF), Tipos, Constraints          │
│  ├── Performance: Índices (ESR), Query Plans, Partitioning      │
│  ├── Concorrência: Isolamento, Locking, Deadlocks              │
│  └── Operações: Pool, Replicas, Migrations, Monitoring          │
│                                                                  │
│  NoSQL                                                           │
│  ├── Modelagem: Access Patterns, Embedding, Denormalization     │
│  ├── Document: MongoDB, DocumentDB — schemas flexíveis          │
│  ├── Key-Value: DynamoDB, Redis — latência mínima               │
│  ├── Wide-Column: Cassandra, ScyllaDB — writes massivos         │
│  └── Graph: Neo4j, Neptune — relações complexas                 │
│                                                                  │
│  Escalabilidade                                                  │
│  ├── Read: Replicas, Cache, Materialized Views                  │
│  ├── Write: Sharding, Batching, Async, LSM-tree                │
│  ├── Patterns: CQRS, Event Sourcing, Polyglot, DB per Service  │
│  └── Infra: Connection Pool, Proxy, DR, Backup                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Referências Consolidadas

- Kleppmann, M. (2017). *Designing Data-Intensive Applications* — O'Reilly
- Fowler, M. (2012). *NoSQL Distilled* — Addison-Wesley
- Winand, M. *Use The Index, Luke* — https://use-the-index-luke.com/
- Brewer, E. (2000). *Towards Robust Distributed Systems* — CAP Theorem
- Abadi, D. (2012). *Consistency Tradeoffs in Modern Distributed Database System Design* — PACELC
- PostgreSQL Documentation — https://www.postgresql.org/docs/
- MongoDB Data Modeling — https://www.mongodb.com/docs/manual/data-modeling/
- DynamoDB Best Practices — https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/
- Redis Documentation — https://redis.io/docs/
- Cassandra Documentation — https://cassandra.apache.org/doc/
- Neo4j Documentation — https://neo4j.com/docs/
