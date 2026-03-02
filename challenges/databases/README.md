# Zero to Hero — Databases Challenge (SQL + NoSQL + Escalabilidade)

> **Programa de especialização progressiva** em banco de dados, cobrindo teoria, modelagem,
> operação e escalabilidade com implementações práticas em **Java 25** e **Go 1.26**.

---

## 1. Resumo Executivo

Este programa transforma um desenvolvedor em **especialista em banco de dados** — da teoria fundamental (CAP, ACID, modelos de dados) à operação de sistemas em escala massiva (sharding, CQRS, event sourcing, polyglot persistence). Todos os desafios são aplicados em um **domínio único e consistente**, com implementações em Java 25 e Go 1.26.

**Domínio escolhido:** **Plataforma de Streaming de Conteúdo (StreamX)** — um domínio que exercita naturalmente todos os paradigmas de banco de dados.

**Por que Streaming Platform?**
- **SQL transacional** — assinaturas, billing, pagamentos exigem ACID rigoroso
- **Document Store** — catálogo com metadata flexível (filmes vs músicas vs podcasts)
- **Key-Value** — sessões de reprodução, cache de conteúdo popular, feature flags
- **Wide-Column** — bilhões de eventos de reprodução (play, pause, seek, stop)
- **Graph** — recomendações sociais (amigos assistem, "quem viu X também viu Y")
- **Time-Series** — métricas de engajamento, analytics de audiência em tempo real
- **Escala massiva** — milhões de usuários concorrentes, petabytes de dados
- Perguntas de entrevista Staff/Principal frequentemente envolvem design de sistemas de streaming

**Referência:** [.docs/databases/](../../.docs/databases/)

---

## 2. Entidades do Domínio

```
User (1) ──── (N) Subscription ──── (1) Plan
  │
  ├──── (N) ViewingHistory
  │              │
  │              └── Content (1) ──── (N) Episode
  │                    │
  │                    ├── Genre (N:M)
  │                    └── ContentCreator (N:M)
  │
  ├──── (N) WatchEvent (play, pause, seek, stop)
  │
  ├──── (N) Review / Rating
  │
  └──── (N) Recommendation
              └── baseada em: social graph, viewing patterns, content similarity
```

| Entidade | Tipo de Dado | Banco Ideal | Justificativa |
|----------|-------------|-------------|---------------|
| `User` | Relacional (ACID) | PostgreSQL | Transações, constraints, integridade |
| `Subscription` | Relacional (ACID) | PostgreSQL | Billing, planos, pagamento recorrente |
| `Plan` | Relacional | PostgreSQL | Poucos registros, relação 1:N com Subscription |
| `Content` | Document | MongoDB | Metadata flexível por tipo (filme/série/música/podcast) |
| `Episode` | Document (embedded) | MongoDB | Sempre lido junto com Content |
| `Genre` | Relacional (N:M) | PostgreSQL | Categorização hierárquica |
| `ViewingHistory` | Wide-Column / Time-Series | Cassandra / TimescaleDB | Alto volume de escritas, queries temporais |
| `WatchEvent` | Wide-Column | Cassandra | Bilhões de eventos, append-only, TTL |
| `Review` | Document | MongoDB | Texto livre, ratings, dados semi-estruturados |
| `Recommendation` | Graph | Neo4j | Relações entre usuários e conteúdos, traversal |
| `Session` | Key-Value | Redis | TTL curto, acesso rápido, estado temporário |
| `Cache` | Key-Value | Redis | Conteúdo popular, catálogo, rankings |
| `Analytics` | Columnar (OLAP) | ClickHouse / Redshift | Aggregations, dashboards, relatórios |

---

## 3. Tecnologias e Dependências

### Bancos de Dados

| Categoria | Banco | Uso no Domínio |
|-----------|-------|---------------|
| SQL (OLTP) | PostgreSQL 17 | Users, Subscriptions, Plans, Billing |
| Document | MongoDB 8 | Content Catalog, Reviews |
| Key-Value | Redis 7 | Sessions, Cache, Rate Limiting, Leaderboards |
| Wide-Column | Cassandra 5 (ou ScyllaDB) | WatchEvents, ViewingHistory |
| Graph | Neo4j 5 | Recommendations, Social Graph |
| Time-Series | TimescaleDB (ou InfluxDB) | Engagement Metrics, Analytics |

### Linguagens e Drivers

| Conceito | Java 25 | Go 1.26 |
|----------|---------|---------|
| SQL Client | JDBC + HikariCP | `database/sql` + `pgx` |
| Migrations | Flyway | `golang-migrate` |
| MongoDB | MongoDB Java Driver | `go.mongodb.org/mongo-driver` |
| Redis | Jedis / Lettuce | `github.com/redis/go-redis` |
| Cassandra | DataStax Java Driver | `github.com/gocql/gocql` |
| Neo4j | Neo4j Java Driver | `github.com/neo4j/neo4j-go-driver` |
| Testes | JUnit 5 + Testcontainers | `testing` + `testcontainers-go` |

---

## 4. Mapa de Níveis

```
Level 0 — Fundamentos de Banco de Dados
  │       (CAP, PACELC, PIE, ACID vs BASE, modelos de dados, decisão)
  ▼
Level 1 — Modelagem SQL & Normalização
  │       (1NF–BCNF, constraints, convenções, tipos de dados)
  ▼
Level 2 — SQL Avançado: Indexação, Queries & Transações
  │       (B-Tree, GIN, ESR, EXPLAIN, locking, concorrência)
  ▼
Level 3 — SQL Operacional: Particionamento, Replicação & Migrations
  │       (Range/Hash/List, Primary-Replica, zero-downtime, monitoramento)
  ▼
Level 4 — NoSQL: Document & Key-Value (MongoDB, Redis, DynamoDB)
  │       (Access patterns, embedding vs referencing, single-table, caching)
  ▼
Level 5 — NoSQL: Wide-Column & Graph (Cassandra, Neo4j)
  │       (Query-driven modeling, Cypher, consistency tuning, TTL)
  ▼
Level 6 — Escalabilidade: Sharding, Caching, CQRS & Event Sourcing
  │       (Estratégias de sharding, cache patterns, write/read scaling)
  ▼
Level 7 — Capstone: Polyglot Persistence & Disaster Recovery
          (Integração de todos os bancos, CDC, DR, produção real)
```

| Level | Tema | Bancos | Desafios |
|-------|------|--------|----------|
| [0](00-database-foundations.md) | Fundamentos de Banco de Dados | Teórico + PostgreSQL | 6 |
| [1](01-sql-modeling.md) | Modelagem SQL & Normalização | PostgreSQL | 7 |
| [2](02-sql-advanced.md) | SQL Avançado: Indexação, Queries & Transações | PostgreSQL | 8 |
| [3](03-sql-operations.md) | SQL Operacional | PostgreSQL | 7 |
| [4](04-nosql-document-keyvalue.md) | NoSQL: Document & Key-Value | MongoDB, Redis, DynamoDB | 8 |
| [5](05-nosql-wide-column-graph.md) | NoSQL: Wide-Column & Graph | Cassandra, Neo4j | 7 |
| [6](06-scalability-patterns.md) | Escalabilidade & Performance Patterns | Multi-DB | 8 |
| [7](07-capstone-polyglot.md) | Capstone: Polyglot Persistence | Todos | 6 |

---

## 5. Estrutura de Cada Desafio

Cada desafio inclui:
- **Contexto** — cenário de negócio no domínio StreamX
- **Requisitos** — o que implementar com critérios objetivos
- **SQL/NoSQL** — queries, schemas, código de aplicação em Java 25 e Go 1.26
- **Critérios de aceite** — checklist verificável

---

## 6. Filosofia

1. **Access Patterns First** — modelar pelos padrões de acesso, não pelas entidades
2. **O banco certo para cada workload** — SQL para transações, Document para flexibilidade, Key-Value para velocidade, Wide-Column para escala, Graph para relações
3. **Teoria aplicada** — CAP/PACELC/PIE não são conceitos abstratos; são decisões de design
4. **Produção real** — migrations, monitoramento, particionamento, disaster recovery
5. **Multi-linguagem** — Java 25 e Go 1.26 para código de aplicação, SQL/CQL/Cypher para queries
6. **Portfólio real** — os projetos resultantes demonstram profundidade em entrevistas Staff/Principal
