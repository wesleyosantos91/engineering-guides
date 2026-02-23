# 10. SQL vs NoSQL

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial — uma das decisões mais importantes em system design  
> **Complexidade:** Média (entender trade-offs é o que importa)

---

## Definição

**SQL (Relational Databases)** armazenam dados em tabelas com esquema fixo, relacionamentos via foreign keys, e garantem transações ACID.

**NoSQL (Not Only SQL)** é um termo genérico para bancos de dados não-relacionais que oferecem modelos de dados flexíveis, escalabilidade horizontal e otimizações para padrões de acesso específicos.

---

## Comparativo Geral

| Aspecto | SQL | NoSQL |
|---------|-----|-------|
| **Modelo** | Tabelas, linhas, colunas | Document, Key-Value, Wide-Column, Graph |
| **Schema** | Fixo (schema-on-write) | Flexível (schema-on-read) |
| **Scaling** | Vertical (scale up) | Horizontal (scale out) |
| **Transactions** | ACID completo | BASE (eventual consistency) |
| **Joins** | Nativo, eficiente | Não suportado / application-level |
| **Consistência** | Strong by default | Eventual / tunable |
| **Query** | SQL padronizado | API proprietária por DB |
| **Schema changes** | ALTER TABLE (pode ser lento) | Schema-less (sem migration) |

---

## SQL — Relational Databases

### Modelo de Dados

```
┌──────────────────────────────────────────────────┐
│ users                                            │
│ ┌────┬──────────┬─────────────────┬────────────┐ │
│ │ id │ name     │ email           │ created_at │ │
│ ├────┼──────────┼─────────────────┼────────────┤ │
│ │ 1  │ João     │ joao@x.com      │ 2024-01-01 │ │
│ │ 2  │ Maria    │ maria@x.com     │ 2024-01-02 │ │
│ └────┴──────────┴─────────────────┴────────────┘ │
└──────────────────────────────────────────────────┘
         │ 1:N
         ▼
┌──────────────────────────────────────────────────┐
│ orders                                           │
│ ┌────┬─────────┬────────┬────────────┬─────────┐ │
│ │ id │ user_id │ amount │ status     │ created │ │
│ ├────┼─────────┼────────┼────────────┼─────────┤ │
│ │ 1  │ 1       │ 99.90  │ completed  │ 2024-01 │ │
│ │ 2  │ 1       │ 50.00  │ pending    │ 2024-02 │ │
│ │ 3  │ 2       │ 75.50  │ completed  │ 2024-01 │ │
│ └────┴─────────┴────────┴────────────┴─────────┘ │
└──────────────────────────────────────────────────┘
```

### ACID

| Propriedade | Descrição | Exemplo |
|-------------|-----------|---------|
| **Atomicity** | Tudo ou nada. Se uma parte falha, tudo é revertido | Transferência: débito + crédito (ambos ou nenhum) |
| **Consistency** | DB sempre fica em estado válido | Constraints, foreign keys respeitados |
| **Isolation** | Transações concorrentes não interferem | Serializable, Repeatable Read |
| **Durability** | Uma vez committed, sobrevive a crash | WAL (Write-Ahead Log) |

```sql
-- ACID em ação: transferência bancária
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Débito
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Crédito
  -- Se qualquer UPDATE falhar → ROLLBACK automático (Atomicity)
COMMIT;
-- Após COMMIT → dados persistidos em disco (Durability)
```

### Strengths

```
✓ Dados altamente estruturados e relacionados
✓ Transações ACID (financeiro, inventário)
✓ JOINs eficientes entre tabelas
✓ SQL padronizado (portabilidade)
✓ Integridade referencial (foreign keys, constraints)
✓ Ferramentas maduras (decades de evolução)
✓ Query optimizer sofisticado
```

### Principais Bancos SQL

| Banco | Tipo | Diferencial | Usado Por |
|-------|------|-------------|-----------|
| **PostgreSQL** | Open source | Mais avançado, extensível, JSONB, full-text | Uber, Instagram, Stripe |
| **MySQL** | Open source | Ecossistema enorme, InnoDB, replication | Meta, GitHub, Twitter |
| **Oracle** | Comercial | Enterprise, RAC, partitioning | Banks, telecom |
| **SQL Server** | Microsoft | .NET integration, SSAS, SSRS | Enterprise |
| **CockroachDB** | NewSQL | Distributed SQL, auto-sharding, ACID | DoorDash |
| **Google Spanner** | Managed | Global SQL, strong consistency | Google |
| **Amazon Aurora** | Managed | MySQL/PG compatible, 5x performance | Netflix |

---

## NoSQL — Not Only SQL

### Tipos de NoSQL

```
┌────────────────────────────────────────────────────────────────────┐
│                          NoSQL Types                               │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │  Key-Value   │  │   Document   │  │ Wide-Column  │            │
│  │              │  │              │  │              │            │
│  │  Redis       │  │  MongoDB     │  │  Cassandra   │            │
│  │  DynamoDB    │  │  Couchbase   │  │  HBase       │            │
│  │  Memcached   │  │  Firestore   │  │  ScyllaDB    │            │
│  └──────────────┘  └──────────────┘  └──────────────┘            │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │    Graph     │  │ Time-Series  │  │   Search     │            │
│  │              │  │              │  │              │            │
│  │  Neo4j       │  │  InfluxDB    │  │ Elasticsearch│            │
│  │  Neptune     │  │  TimescaleDB │  │  OpenSearch  │            │
│  │  JanusGraph  │  │  QuestDB     │  │  Solr        │            │
│  └──────────────┘  └──────────────┘  └──────────────┘            │
└────────────────────────────────────────────────────────────────────┘
```

---

### 1. Key-Value Store

O modelo mais simples: chave → valor.

```
┌──────────────────────────────────────┐
│ Key-Value Store                      │
│                                      │
│  "user:101"  → '{"name":"João",...}' │
│  "session:abc" → '{"token":"xyz"}'   │
│  "cart:101"  → '["item1","item2"]'   │
│  "count:page:home" → 42             │
└──────────────────────────────────────┘
```

| Aspecto | Detalhes |
|---------|----------|
| **Modelo** | Key → Value (string, blob, JSON) |
| **Operações** | GET, SET, DELETE (simples!) |
| **Latência** | Sub-millisecond |
| **Uso** | Cache, sessions, config, rate limiting |
| **Bancos** | Redis, DynamoDB, Memcached, etcd |

```redis
# Redis
SET user:101 '{"name":"João","email":"j@x.com"}' EX 3600
GET user:101
DEL user:101

# DynamoDB
aws dynamodb put-item --table-name Users --item '{"id": {"S": "101"}, "name": {"S": "João"}}'
```

### 2. Document Store

Armazena documentos semi-estruturados (JSON/BSON). Cada documento pode ter estrutura diferente.

```json
// MongoDB - coleção "orders"
{
  "_id": ObjectId("65b1a2b3c4d5e6f7a8b9c0d1"),
  "user_id": 101,
  "status": "completed",
  "items": [
    { "product": "Notebook", "qty": 1, "price": 4500.00 },
    { "product": "Mouse", "qty": 2, "price": 89.90 }
  ],
  "shipping": {
    "address": "Rua X, 123",
    "city": "São Paulo",
    "tracking": "BR123456789"
  },
  "total": 4679.80,
  "created_at": ISODate("2024-01-25T10:00:00Z")
}
```

| Aspecto | Detalhes |
|---------|----------|
| **Modelo** | Documents (JSON/BSON) em collections |
| **Schema** | Flexível (cada doc pode diferir) |
| **Queries** | Rich queries, aggregation pipeline |
| **Uso** | CMS, catálogos, perfis de usuário, eventos |
| **Bancos** | MongoDB, Couchbase, Firestore, CosmosDB |

```javascript
// MongoDB queries
db.orders.find({ "user_id": 101, "status": "completed" })
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$user_id", total: { $sum: "$total" } } },
  { $sort: { total: -1 } }
])
```

### 3. Wide-Column Store

Dados organizados em **column families**. Cada row pode ter colunas diferentes.

```
Row Key          │ Column Family: profile      │ Column Family: activity
─────────────────┼─────────────────────────────┼──────────────────────────
user:101         │ name="João"                 │ login:2024-01-25=true
                 │ email="j@x.com"             │ login:2024-01-24=true
                 │ city="São Paulo"             │ purchase:2024-01-20=order123
─────────────────┼─────────────────────────────┼──────────────────────────
user:205         │ name="Maria"                │ login:2024-01-25=true
                 │ phone="+55..."               │
                 │ (sem email!)                 │
```

| Aspecto | Detalhes |
|---------|----------|
| **Modelo** | Row key → Column families → Columns |
| **Scaling** | Excelente horizontal (partitioned by row key) |
| **Uso** | Timeseries, IoT, messaging (writes massivos) |
| **Bancos** | Cassandra, HBase, ScyllaDB, Bigtable |

```cql
-- Cassandra CQL
CREATE TABLE user_activity (
    user_id     UUID,
    activity_ts TIMESTAMP,
    activity    TEXT,
    details     MAP<TEXT, TEXT>,
    PRIMARY KEY (user_id, activity_ts)
) WITH CLUSTERING ORDER BY (activity_ts DESC);

-- Buscar últimas 20 atividades do user
SELECT * FROM user_activity WHERE user_id = ? LIMIT 20;
```

### 4. Graph Database

Dados modelados como **nodes** e **edges** (relacionamentos).

```
        ┌──────────┐    FOLLOWS     ┌──────────┐
        │ João     │ ──────────────▶│ Maria    │
        │ (User)   │                │ (User)   │
        └────┬─────┘                └────┬─────┘
             │ POSTED                    │ LIKES
             ▼                           ▼
        ┌──────────┐                ┌──────────┐
        │ "Hello"  │ ◀─── LIKES ───│ Pedro    │
        │ (Post)   │               │ (User)   │
        └──────────┘                └──────────┘
```

| Aspecto | Detalhes |
|---------|----------|
| **Modelo** | Nodes + Edges + Properties |
| **Consulta** | Graph traversal (BFS, DFS, shortest path) |
| **Uso** | Redes sociais, fraud detection, recommendations |
| **Bancos** | Neo4j, Amazon Neptune, JanusGraph, ArangoDB |

```cypher
// Neo4j Cypher
// Amigos de amigos que João não segue
MATCH (joao:User {name: "João"})-[:FOLLOWS]->(:User)-[:FOLLOWS]->(suggested:User)
WHERE NOT (joao)-[:FOLLOWS]->(suggested) AND suggested <> joao
RETURN DISTINCT suggested.name, COUNT(*) AS mutual_friends
ORDER BY mutual_friends DESC
LIMIT 10;
```

### 5. Time-Series Database

Otimizado para dados **timestamped** com alta taxa de ingestão.

| Aspecto | Detalhes |
|---------|----------|
| **Modelo** | (timestamp, metric, tags, value) |
| **Uso** | Métricas, IoT sensors, stock prices, logs |
| **Bancos** | InfluxDB, TimescaleDB, QuestDB, Prometheus |

```sql
-- TimescaleDB (PostgreSQL extension)
SELECT time_bucket('5 minutes', time) AS bucket,
       avg(cpu_usage) AS avg_cpu,
       max(cpu_usage) AS max_cpu
FROM server_metrics
WHERE time > now() - interval '24 hours'
  AND server_id = 'web-01'
GROUP BY bucket
ORDER BY bucket DESC;
```

### 6. Search Engine

Otimizado para **full-text search** e analytics em texto.

| Aspecto | Detalhes |
|---------|----------|
| **Modelo** | Inverted index (term → document list) |
| **Uso** | Busca textual, log analytics, e-commerce search |
| **Bancos** | Elasticsearch, OpenSearch, Apache Solr, Meilisearch |

```json
// Elasticsearch query
POST /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "notebook gamer" } }
      ],
      "filter": [
        { "range": { "price": { "lte": 5000 } } },
        { "term": { "brand": "lenovo" } }
      ]
    }
  },
  "sort": [{ "_score": "desc" }, { "price": "asc" }],
  "size": 20
}
```

---

## CAP Theorem

Qualquer sistema distribuído pode garantir apenas **2 de 3** propriedades:

```
                Consistency
                    /\
                   /  \
                  /    \
                 / CA   \
                /  (SQL) \
               /──────────\
              / CP    AP   \
             / (HBase)(Cass)\
            /________________\
       Partition              Availability
       Tolerance
```

| Tipo | Garante | Sacrifica | Exemplo |
|------|---------|-----------|---------|
| **CA** | Consistency + Availability | Partition Tolerance | PostgreSQL (single node) |
| **CP** | Consistency + Partition Tolerance | Availability (durante partição) | HBase, MongoDB, etcd |
| **AP** | Availability + Partition Tolerance | Consistency (eventual) | Cassandra, DynamoDB, CouchDB |

> **Na prática:** Em sistemas distribuídos, partições de rede **sempre ocorrem**, então a escolha real é entre **CP** e **AP**.

---

## ACID vs BASE

| ACID (SQL) | BASE (NoSQL) |
|-----------|-------------|
| **A**tomicity | **B**asically **A**vailable |
| **C**onsistency | **S**oft state |
| **I**solation | **E**ventual consistency |
| **D**urability | |

```
ACID: "A qualquer momento, o estado do DB é correto e consistente"
  → Necessário para: pagamentos, inventário, dados financeiros

BASE: "Eventualmente o estado converge para correto"
  → Aceitável para: feed de notícias, contadores de likes, recomendações
```

---

## Decision Matrix — Quando Usar Qual?

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SQL vs NoSQL Decision Tree                       │
│                                                                     │
│  Dados altamente relacionados (JOINs)? ────── SIM ──▶ SQL          │
│       │                                                             │
│       NÃO                                                           │
│       │                                                             │
│  ACID transactions necessárias? ──────────── SIM ──▶ SQL           │
│       │                                                | ou NewSQL  │
│       NÃO                                                           │
│       │                                                             │
│  Schema muda frequentemente? ─────────────── SIM ──▶ Document DB   │
│       │                                                             │
│       NÃO                                                           │
│       │                                                             │
│  Write-heavy (>100K writes/s)? ──────────── SIM ──▶ Wide-Column   │
│       │                                                             │
│       NÃO                                                           │
│       │                                                             │
│  Key-based access (cache, sessions)? ────── SIM ──▶ Key-Value     │
│       │                                                             │
│       NÃO                                                           │
│       │                                                             │
│  Graph traversal (social, fraud)? ──────── SIM ──▶ Graph DB       │
│       │                                                             │
│       NÃO                                                           │
│       │                                                             │
│  Full-text search? ────────────────────── SIM ──▶ Search Engine   │
│       │                                                             │
│       NÃO                                                           │
│       │                                                             │
│  Time-series data? ───────────────────── SIM ──▶ Time-Series DB   │
│       │                                                             │
│       NÃO                                                           │
│       ▼                                                             │
│  Default: PostgreSQL (cobre 80% dos casos)                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Polyglot Persistence

Big Techs usam **múltiplos bancos** para diferentes necessidades:

```
┌──────────────────────────────────────────────────────────────┐
│                    E-Commerce Platform                        │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │ PostgreSQL │  │  MongoDB   │  │   Redis    │            │
│  │ Orders,    │  │ Product    │  │ Sessions,  │            │
│  │ Payments,  │  │ Catalog    │  │ Cart,      │            │
│  │ Inventory  │  │ (flexible  │  │ Cache      │            │
│  │ (ACID!)    │  │  schema)   │  │ (speed!)   │            │
│  └────────────┘  └────────────┘  └────────────┘            │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │Elasticsearch│ │ Cassandra  │  │   Neo4j    │            │
│  │ Product    │  │ User       │  │ Recommend- │            │
│  │ Search     │  │ Activity   │  │ ations     │            │
│  │ (full-text)│  │ (writes!)  │  │ (graph!)   │            │
│  └────────────┘  └────────────┘  └────────────┘            │
└──────────────────────────────────────────────────────────────┘
```

---

## Uso em Big Techs

### Meta (Facebook)
| Dado | Banco | Motivo |
|------|-------|--------|
| Social graph | TAO (MySQL) | Relações entre entities |
| Messaging | RocksDB + HBase | Write-heavy, wide-column |
| Search | Custom (Unicorn) | Full-text search |
| Analytics | Presto + Hive | OLAP, data warehouse |

### Netflix
| Dado | Banco | Motivo |
|------|-------|--------|
| User data | Cassandra | AP, multi-region, always available |
| Billing | MySQL | ACID transactions |
| Viewing history | Cassandra | Write-heavy |
| Search | Elasticsearch | Full-text |
| Cache | EVCache (Memcached) | Speed |

### Uber
| Dado | Banco | Motivo |
|------|-------|--------|
| Trip data | MySQL (sharded) | ACID, relational |
| Maps/Geo | Custom (H3) + PostGIS | Geospatial |
| Real-time | Redis + Cassandra | Speed + persistence |
| Analytics | Hudi + Spark | Data lake |

### Amazon
| Dado | Banco | Motivo |
|------|-------|--------|
| Product catalog | DynamoDB | Key-value, scale |
| Orders/Payments | Aurora (MySQL) | ACID |
| Sessions/Cart | DynamoDB / ElastiCache | Speed |
| Search | OpenSearch | Full-text |
| Recommendations | Neptune (Graph) | Graph traversals |

---

## NewSQL — O Melhor dos Dois Mundos?

Bancos que combinam **SQL interface + ACID** com **distribuição horizontal do NoSQL**:

| Banco | Base | Diferencial |
|-------|------|-------------|
| **CockroachDB** | PostgreSQL wire | Distributed SQL, auto-sharding, survives AZ failure |
| **TiDB** | MySQL compatible | HTAP (OLTP + OLAP), TiKV storage |
| **YugabyteDB** | PostgreSQL compatible | Distributed, multi-region |
| **Google Spanner** | Custom | Global strong consistency, TrueTime |
| **PlanetScale** | MySQL (Vitess) | Serverless, branching, non-blocking schema changes |

```
Trade-off NewSQL:
  ✓ SQL + ACID + horizontally scalable
  ✗ Mais complexo de operar
  ✗ Latência de escrita maior que single-node SQL
  ✗ Vendor lock-in (Spanner) ou ecossistema menor
```

---

## Perguntas Comuns em Entrevistas

1. **SQL vs NoSQL — quando usar cada?** → SQL para relações + ACID; NoSQL para scale + flexibilidade + padrão de acesso específico
2. **O que é CAP theorem?** → Não pode ter Consistency + Availability + Partition Tolerance ao mesmo tempo
3. **ACID vs BASE?** → ACID = transações fortes (pagamentos); BASE = eventual consistency (social feed)
4. **Projeto X: qual banco usar?** → Depende: tipo de dado, padrão de acesso, escala, consistência necessária
5. **Polyglot persistence?** → Usar múltiplos bancos especializados para diferentes necessidades

---

## Trade-offs

| Decisão | SQL | NoSQL |
|---------|-----|-------|
| **Consistência** | Strong (ACID) | Eventual (BASE) |
| **Schema** | Fixo (safe, migration) | Flexível (rápido, risco) |
| **Scale** | Vertical (caro) | Horizontal (commodity) |
| **Joins** | Nativo, eficiente | Application-level, caro |
| **Maturidade** | Decades, ecosystem enorme | Mais recente, em evolução |
| **Custo de infra** | Scale up = expensive | Scale out = commodity |

---

## Referências

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [MongoDB Manual](https://www.mongodb.com/docs/manual/)
- [Apache Cassandra](https://cassandra.apache.org/_/index.html)
- [Redis Documentation](https://redis.io/docs/)
- [Neo4j Documentation](https://neo4j.com/docs/)
- Designing Data-Intensive Applications — Martin Kleppmann
- System Design Interview — Alex Xu, Vol 1 & 2
