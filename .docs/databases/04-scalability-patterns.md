# Databases — Escalabilidade, Performance & Patterns

> **Objetivo deste documento:** Servir como referência completa sobre **escalabilidade, performance e patterns de banco de dados**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Cobre sharding, replicação, caching, connection pooling, query patterns e estratégias para escala infinita, eficiência e flexibilidade.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **Vertical Scaling** | Scale up primeiro — é mais simples | Sharding prematuro | Escalar verticalmente até o limite; horizontalmente depois |
| **Sharding** | Shard key com alta cardinalidade e distribuição uniforme | Shard key com hot spots (ex: data, status) | Chave composta ou hash-based sharding |
| **Read Replicas** | Escalar leitura sem afetar escrita | Ignorar replication lag | Read-after-write no primary; stale reads aceitáveis nas réplicas |
| **Caching** | Cache é aceleração, não fonte de verdade | Cache sem TTL ou invalidation strategy | TTL + invalidação por evento; cache-aside pattern |
| **Connection Pool** | Pool size = CPU cores × 2 + disco | Conexão por request sem pool | HikariCP, PgBouncer, RDS Proxy |
| **Polyglot Persistence** | Banco certo para cada workload | Um banco para tudo | SQL para transações, NoSQL para escala, cache para leitura |

---

## Sumário

- [Databases — Escalabilidade, Performance \& Patterns](#databases--escalabilidade-performance--patterns)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Estratégias de Escalabilidade](#estratégias-de-escalabilidade)
    - [Vertical vs Horizontal Scaling](#vertical-vs-horizontal-scaling)
    - [Read Scaling](#read-scaling)
    - [Write Scaling](#write-scaling)
  - [Sharding (Particionamento Horizontal)](#sharding-particionamento-horizontal)
    - [Estratégias de Sharding](#estratégias-de-sharding)
    - [Shard Key — Como Escolher](#shard-key--como-escolher)
    - [Desafios do Sharding](#desafios-do-sharding)
  - [Caching Patterns](#caching-patterns)
    - [Cache-Aside (Lazy Loading)](#cache-aside-lazy-loading)
    - [Write-Through](#write-through)
    - [Write-Behind (Write-Back)](#write-behind-write-back)
    - [Read-Through](#read-through)
    - [Cache Invalidation](#cache-invalidation)
    - [Problemas comuns de cache](#problemas-comuns-de-cache)
  - [Replication Patterns](#replication-patterns)
  - [Polyglot Persistence](#polyglot-persistence)
  - [CQRS (Command Query Responsibility Segregation)](#cqrs-command-query-responsibility-segregation)
  - [Event Sourcing com Databases](#event-sourcing-com-databases)
  - [Database per Service (Microservices)](#database-per-service-microservices)
  - [Connection Patterns](#connection-patterns)
  - [Performance Patterns](#performance-patterns)
    - [Materialized Views](#materialized-views)
    - [Batch Processing](#batch-processing)
    - [Pagination Patterns](#pagination-patterns)
    - [Bulk Operations](#bulk-operations)
  - [Infinite Scale — Patterns para Escala Massiva](#infinite-scale--patterns-para-escala-massiva)
  - [Efficiency — Otimização de Custos e Recursos](#efficiency--otimização-de-custos-e-recursos)
  - [Flexibility — Patterns para Evolução](#flexibility--patterns-para-evolução)
  - [Disaster Recovery](#disaster-recovery)
  - [Anti-patterns de Escalabilidade](#anti-patterns-de-escalabilidade)
  - [Referências](#referências)

---

## Estratégias de Escalabilidade

### Vertical vs Horizontal Scaling

```
┌──────────────────────────────────────────────────────────────────┐
│         VERTICAL vs HORIZONTAL SCALING                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Vertical (Scale Up)                Horizontal (Scale Out)      │
│  ┌──────────┐                      ┌────┐ ┌────┐ ┌────┐       │
│  │          │                      │ DB │ │ DB │ │ DB │       │
│  │   BIG    │  Mais CPU,           │  1 │ │  2 │ │  3 │       │
│  │   DB     │  RAM, Storage        └────┘ └────┘ └────┘       │
│  │          │                         │      │      │          │
│  │          │                         └──────┼──────┘          │
│  └──────────┘                               │                  │
│                                        Distribuído              │
│                                                                  │
│  Vertical Scaling:                                              │
│  ✅ Simples — sem mudança de arquitetura                        │
│  ✅ Sem complexidade distribuída                                │
│  ✅ Transações ACID nativas                                     │
│  ❌ Limite físico de hardware                                   │
│  ❌ Single point of failure                                      │
│  ❌ Downtime para upgrade                                       │
│                                                                  │
│  Horizontal Scaling:                                            │
│  ✅ Escala "infinita" (teoricamente)                            │
│  ✅ Sem single point of failure                                  │
│  ✅ Scale out sob demanda                                        │
│  ❌ Complexidade em distributed transactions                    │
│  ❌ Trade-offs de consistência (CAP/PACELC)                     │
│  ❌ Cross-shard queries são caras                               │
│                                                                  │
│  🎯 SEQUÊNCIA RECOMENDADA:                                      │
│  1. Otimizar queries/índices (0 custo infra)                    │
│  2. Vertical scaling (simples)                                   │
│  3. Read replicas (escalar leituras)                            │
│  4. Caching layer (Redis/Memcached)                             │
│  5. Sharding (último recurso para SQL)                          │
│  6. Migrar workloads para bancos especializados                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Read Scaling

```
┌──────────────────────────────────────────────────────────────────┐
│              READ SCALING                                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Application                                                     │
│       │                                                          │
│  ┌────▼────┐                                                    │
│  │  Load   │                                                    │
│  │ Balancer│                                                    │
│  └──┬───┬──┘                                                    │
│     │   │                                                        │
│  Writes │  Reads                                                │
│     │   └────────┬──────────┬──────────┐                        │
│  ┌──▼──┐   ┌─────▼───┐  ┌──▼──────┐  ┌▼────────┐             │
│  │Write│──►│Replica 1│  │Replica 2│  │Replica 3│             │
│  │(Prim│   │ (R/O)   │  │ (R/O)   │  │ (R/O)   │             │
│  │ary) │   └─────────┘  └─────────┘  └─────────┘             │
│  └─────┘                                                        │
│                                                                  │
│  Técnicas:                                                       │
│  1. Read Replicas (PostgreSQL, MySQL, Aurora)                   │
│  2. Cache Layer (Redis, Memcached)                              │
│  3. CDN para dados estáticos                                    │
│  4. Materialized Views para queries complexas                   │
│  5. Search Index (Elasticsearch) para full-text                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Write Scaling

```
┌──────────────────────────────────────────────────────────────────┐
│              WRITE SCALING                                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Técnicas para escalar escritas:                                │
│                                                                  │
│  1. Sharding (distribuir dados entre nós)                       │
│     • Cada shard recebe parte dos writes                        │
│     • Linear scaling: 2x shards ≈ 2x write throughput          │
│                                                                  │
│  2. Batching / Buffering                                        │
│     • Acumular writes e flush em batch                          │
│     • Reduz overhead de transação individual                    │
│     • Ex: Kinesis → Lambda → DynamoDB batch write               │
│                                                                  │
│  3. Async Writes                                                │
│     • Write para fila (SQS, Kafka) → consumer persiste         │
│     • Desacopla latência de escrita do banco                    │
│     • Trade-off: eventual consistency                           │
│                                                                  │
│  4. Write-Optimized Engines                                     │
│     • LSM-tree: Cassandra, RocksDB, LevelDB                    │
│     • Append-only / WAL-based writes                            │
│     • Compaction em background                                  │
│                                                                  │
│  5. Partitioning (dentro do banco)                              │
│     • Tabelas particionadas por range/hash                      │
│     • Paralelismo de I/O em partições diferentes                │
│                                                                  │
│  6. Multi-Primary / Active-Active                               │
│     • Ambos os nós aceitam writes                               │
│     • Conflict resolution necessário                            │
│     • Ex: Aurora Multi-Master, CockroachDB, Cassandra           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Sharding (Particionamento Horizontal)

### Estratégias de Sharding

```
┌──────────────────────────────────────────────────────────────────┐
│          ESTRATÉGIAS DE SHARDING                                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Range-Based Sharding                                        │
│     ┌─────────┐  ┌─────────┐  ┌─────────┐                     │
│     │Shard 1  │  │Shard 2  │  │Shard 3  │                     │
│     │A-H      │  │I-P      │  │Q-Z      │                     │
│     └─────────┘  └─────────┘  └─────────┘                     │
│     ✅ Range queries eficientes                                 │
│     ❌ Risco de hot spots (nomes com mesma letra)               │
│                                                                  │
│  2. Hash-Based Sharding                                         │
│     shard = hash(key) % num_shards                              │
│     ┌─────────┐  ┌─────────┐  ┌─────────┐                     │
│     │Shard 1  │  │Shard 2  │  │Shard 3  │                     │
│     │hash%3=0 │  │hash%3=1 │  │hash%3=2 │                     │
│     └─────────┘  └─────────┘  └─────────┘                     │
│     ✅ Distribuição uniforme                                    │
│     ❌ Range queries precisam consultar todos os shards          │
│                                                                  │
│  3. Directory-Based Sharding                                    │
│     ┌────────────────┐                                          │
│     │ Lookup Service │  key → shard mapping                    │
│     └───────┬────────┘                                          │
│     ┌───────┼───────┐                                           │
│     ▼       ▼       ▼                                           │
│   Shard1  Shard2  Shard3                                       │
│     ✅ Flexibilidade total de mapeamento                        │
│     ❌ Lookup service é ponto de falha                           │
│                                                                  │
│  4. Consistent Hashing                                          │
│     • Minimiza redistribuição ao adicionar/remover nós          │
│     • Usado em: Cassandra, DynamoDB, Riak                       │
│     • Virtual nodes para distribuição mais uniforme             │
│     ✅ Adicionar shard move ~1/N dos dados                      │
│     ❌ Mais complexo de implementar                              │
│                                                                  │
│  5. Geo-Based Sharding                                          │
│     • Shard por região geográfica                               │
│     • Dados do BR → shard sa-east-1                             │
│     • Dados do US → shard us-east-1                             │
│     ✅ Compliance (dados no país), baixa latência                │
│     ❌ Cross-region queries são caras                            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Shard Key — Como Escolher

| Critério | Boa shard key | Shard key ruim |
|----------|-------------|---------------|
| **Cardinalidade** | `user_id` (milhões) | `status` (3-5 valores) |
| **Distribuição** | `hash(order_id)` (uniforme) | `created_date` (hotspot no dia atual) |
| **Query alignment** | Chave usada em WHERE frequente | Chave que força cross-shard queries |
| **Crescimento** | Monotônica composta | Sequencial simples (hot shard no final) |
| **Imutabilidade** | ID natural imutável | Atributo que pode mudar (região, status) |

> **Regra de ouro:** Se > 80% das queries filtram por uma coluna, ela é candidata a shard key.

### Desafios do Sharding

```
┌──────────────────────────────────────────────────────────────────┐
│          DESAFIOS DO SHARDING                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Cross-Shard Queries                                         │
│     • Queries que precisam de dados em múltiplos shards         │
│     • Scatter-gather: consultar N shards + merge resultado      │
│     • Solução: desnormalizar ou manter dados de lookup local    │
│                                                                  │
│  2. Distributed Transactions                                    │
│     • 2PC (Two-Phase Commit) é lento e bloqueia               │
│     • Solução: Saga pattern, eventual consistency               │
│                                                                  │
│  3. Resharding                                                   │
│     • Adicionar/remover shards requer migração de dados        │
│     • Solução: consistent hashing, virtual shards               │
│     • Planeje capacidade para evitar resharding frequente       │
│                                                                  │
│  4. Hotspots                                                     │
│     • Um shard recebe tráfego desproporcional                  │
│     • Solução: hash-based, write sharding com suffix            │
│                                                                  │
│  5. Referential Integrity                                       │
│     • Foreign keys não funcionam cross-shard                    │
│     • Solução: validação na aplicação, eventual consistency     │
│                                                                  │
│  6. Schema Changes                                               │
│     • ALTER TABLE em N shards = N migrações                    │
│     • Solução: rolling migrations, feature flags                │
│                                                                  │
│  7. Backup & Restore                                            │
│     • Backup consistente cross-shard é complexo                 │
│     • Solução: snapshots coordenados, point-in-time recovery    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Caching Patterns

### Cache-Aside (Lazy Loading)

```
┌──────────────────────────────────────────────────────────────────┐
│              CACHE-ASIDE (LAZY LOADING)                          │
│                                                                  │
│  App ──► Cache hit? ──► SIM → Retorna do cache                 │
│              │                                                   │
│              NÃO                                                │
│              │                                                   │
│              ▼                                                   │
│          Database ──► Retorna dado ──► Popula cache ──► Retorna │
│                                                                  │
│  ✅ Somente dados acessados são cached                          │
│  ✅ Cache failures não bloqueiam a aplicação                    │
│  ❌ Cache miss = latência extra (DB + cache write)              │
│  ❌ Dados podem ficar stale até o TTL expirar                   │
└──────────────────────────────────────────────────────────────────┘
```

### Write-Through

```
┌──────────────────────────────────────────────────────────────────┐
│              WRITE-THROUGH                                       │
│                                                                  │
│  App ──► Escreve Cache ──► Cache escreve Database               │
│              │                                                   │
│              ▼                                                   │
│          Confirma após ambos persistirem                         │
│                                                                  │
│  ✅ Cache sempre consistente com DB                             │
│  ✅ Reads nunca são stale                                       │
│  ❌ Latência de write maior (cache + DB)                        │
│  ❌ Dados nunca lidos ocupam cache (combine com TTL)            │
└──────────────────────────────────────────────────────────────────┘
```

### Write-Behind (Write-Back)

```
┌──────────────────────────────────────────────────────────────────┐
│              WRITE-BEHIND (WRITE-BACK)                           │
│                                                                  │
│  App ──► Escreve Cache ──► Retorna imediatamente                │
│                │                                                 │
│                ▼ (async, batch)                                  │
│            Database                                              │
│                                                                  │
│  ✅ Write latency muito baixa                                   │
│  ✅ Batch writes para DB (eficiente)                            │
│  ❌ Risco de data loss se cache falhar antes do flush           │
│  ❌ Complexidade de implementação                               │
│  ❌ Eventual consistency entre cache e DB                       │
└──────────────────────────────────────────────────────────────────┘
```

### Read-Through

```
┌──────────────────────────────────────────────────────────────────┐
│              READ-THROUGH                                        │
│                                                                  │
│  App ──► Cache ──► hit? ──► SIM → Retorna                      │
│                      │                                           │
│                      NÃO                                        │
│                      │                                           │
│                 Cache busca no DB ──► Armazena + Retorna        │
│                                                                  │
│  Similar ao Cache-Aside, mas o CACHE gerencia o DB lookup      │
│  ✅ Aplicação simplificada (cache abstrai o DB)                 │
│  ❌ First request sempre lento (cold cache)                     │
└──────────────────────────────────────────────────────────────────┘
```

### Cache Invalidation

| Estratégia | Como funciona | Quando usar |
|-----------|--------------|-------------|
| **TTL (Time-To-Live)** | Cache expira após tempo fixo | Dados que aceitam stale window |
| **Event-Driven** | Publish invalidação após write no DB | Dados que precisam de freshness |
| **Write-Through** | Cache atualizado junto com DB | Strong consistency necessária |
| **Version-Based** | Key inclui versão; novo dado = nova key | Imutabilidade (assets, configs) |
| **Tag-Based** | Invalidar por tag (ex: todos de user:123) | Invalidação em grupo |

### Problemas comuns de cache

```
┌──────────────────────────────────────────────────────────────────┐
│          PROBLEMAS DE CACHING                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Cache Stampede (Thundering Herd)                            │
│     Problema: TTL expira → N requests simultâneas ao DB        │
│     Solução:                                                     │
│     • Lock: apenas 1 request reconstrói o cache                 │
│     • Probabilistic early expiration (jitter no TTL)            │
│     • Background refresh antes do TTL expirar                   │
│                                                                  │
│  2. Cache Penetration                                            │
│     Problema: Requests para keys que NÃO existem no DB         │
│     (cache sempre miss → DB sempre hit → resultado vazio)       │
│     Solução:                                                     │
│     • Cache null results com TTL curto                          │
│     • Bloom filter para filtrar keys inexistentes               │
│                                                                  │
│  3. Cache Avalanche                                              │
│     Problema: Muitas keys expiram ao mesmo tempo                │
│     Solução:                                                     │
│     • Jitter no TTL (TTL base + random seconds)                 │
│     • Stagger cache warming em deploy                           │
│                                                                  │
│  4. Hot Key                                                      │
│     Problema: Uma key recebe tráfego desproporcional            │
│     Solução:                                                     │
│     • Local cache (L1) + distributed cache (L2)                 │
│     • Key replication across cache nodes                        │
│                                                                  │
│  5. Stale Data                                                   │
│     Problema: Cache desatualizado serve dado errado             │
│     Solução:                                                     │
│     • Event-driven invalidation (CDC, pub/sub)                  │
│     • Versioning (ETag-like)                                    │
│     • Short TTL para dados críticos                             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Replication Patterns

```
┌──────────────────────────────────────────────────────────────────┐
│          REPLICATION PATTERNS                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Single-Leader (Primary-Replica)                             │
│     ┌────────┐    ┌────────┐    ┌────────┐                     │
│     │Leader  │───►│Follower│    │Follower│                     │
│     │  R/W   │───►│  R/O   │    │  R/O   │                     │
│     └────────┘    └────────┘    └────────┘                     │
│     Uso: PostgreSQL, MySQL, MongoDB (default)                   │
│                                                                  │
│  2. Multi-Leader (Active-Active)                                │
│     ┌────────┐◄───►┌────────┐                                  │
│     │Leader 1│     │Leader 2│                                  │
│     │  R/W   │     │  R/W   │                                  │
│     └────────┘     └────────┘                                  │
│     Uso: Multi-region, Aurora Multi-Master                      │
│     Conflict resolution: LWW, merge, custom                    │
│                                                                  │
│  3. Leaderless (Masterless)                                     │
│     ┌────────┐  ┌────────┐  ┌────────┐                        │
│     │Node 1  │  │Node 2  │  │Node 3  │                        │
│     │  R/W   │  │  R/W   │  │  R/W   │                        │
│     └────────┘  └────────┘  └────────┘                        │
│     Uso: Cassandra, DynamoDB, Riak                             │
│     Quorum: W + R > N para strong consistency                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Polyglot Persistence

```
┌──────────────────────────────────────────────────────────────────┐
│          POLYGLOT PERSISTENCE                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Conceito: Usar o banco certo para cada tipo de workload        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    E-COMMERCE EXAMPLE                     │   │
│  │                                                           │   │
│  │  Catálogo ──► MongoDB (documents flexíveis)              │   │
│  │  Pedidos  ──► PostgreSQL (transações ACID)               │   │
│  │  Sessões  ──► Redis (key-value, TTL)                     │   │
│  │  Busca    ──► Elasticsearch (full-text search)           │   │
│  │  Analytics──► ClickHouse (columnar, aggregations)        │   │
│  │  Social   ──► Neo4j (recomendações de amigos)            │   │
│  │  Metrics  ──► TimescaleDB (time-series)                  │   │
│  │                                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ✅ Performance ótima por workload                              │
│  ✅ Escala independente por serviço                             │
│  ❌ Complexidade operacional (N bancos para operar)             │
│  ❌ Consistência cross-database é eventual                     │
│                                                                  │
│  🎯 Não adote cegamente — comece com 1-2 bancos e adicione    │
│     quando houver requisitos claros que justifiquem             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## CQRS (Command Query Responsibility Segregation)

```
┌──────────────────────────────────────────────────────────────────┐
│                       CQRS                                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌────────────┐                                │
│                    │  API Layer │                                │
│                    └──┬─────┬──┘                                │
│               Commands│     │Queries                            │
│                    ┌──▼──┐ ┌▼─────┐                             │
│                    │Write│ │ Read │                              │
│                    │Model│ │ Model│                              │
│                    └──┬──┘ └──▲───┘                              │
│                       │      │                                   │
│                    ┌──▼──┐   │ Projection /                     │
│                    │Write│───┘ Sync (CDC, Events)                │
│                    │ DB  │                                       │
│                    │(SQL)│──────────► ┌────────┐                │
│                    └─────┘           │Read DB │                │
│                                       │(NoSQL/ │                │
│                                       │ Cache) │                │
│                                       └────────┘                │
│                                                                  │
│  Write DB: Normalizado (SQL), otimizado para consistência       │
│  Read DB:  Desnormalizado, otimizado para queries rápidas       │
│                                                                  │
│  Sync: CDC (Debezium), Domain Events, ou Change Streams         │
│                                                                  │
│  ✅ Leitura e escrita escalam independentemente                 │
│  ✅ Modelo de leitura otimizado por use case                    │
│  ❌ Eventual consistency entre read e write                     │
│  ❌ Complexidade de infraestrutura                              │
│                                                                  │
│  Quando usar:                                                    │
│  • Read/Write ratio >> 10:1                                     │
│  • Queries complexas que degradam o write DB                    │
│  • Necessidade de múltiplas "views" dos mesmos dados            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Event Sourcing com Databases

```
┌──────────────────────────────────────────────────────────────────┐
│              EVENT SOURCING                                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Em vez de armazenar o ESTADO ATUAL, armazena todos os EVENTOS  │
│  que levaram ao estado atual.                                    │
│                                                                  │
│  Tradicional:                                                    │
│  account.balance = 1000                                         │
│                                                                  │
│  Event Sourcing:                                                 │
│  1. AccountCreated  { balance: 0 }                              │
│  2. MoneyDeposited  { amount: 1500 }                            │
│  3. MoneyWithdrawn  { amount: 500 }                             │
│  → Estado derivado: balance = 0 + 1500 - 500 = 1000            │
│                                                                  │
│  Event Store (append-only):                                     │
│  ┌──────────┬────────────────┬──────────┬──────────────────┐   │
│  │ event_id │ aggregate_id   │ type     │ data             │   │
│  ├──────────┼────────────────┼──────────┼──────────────────┤   │
│  │ 1        │ account-123    │ CREATED  │ {balance: 0}     │   │
│  │ 2        │ account-123    │ DEPOSIT  │ {amount: 1500}   │   │
│  │ 3        │ account-123    │ WITHDRAW │ {amount: 500}    │   │
│  └──────────┴────────────────┴──────────┴──────────────────┘   │
│                                                                  │
│  Bancos para Event Store:                                       │
│  • EventStoreDB (purpose-built)                                 │
│  • PostgreSQL (com tabela append-only)                          │
│  • DynamoDB (com partition key = aggregate_id)                  │
│  • Kafka (como event log distribuído)                            │
│                                                                  │
│  ✅ Audit trail completo e imutável                             │
│  ✅ Reconstruir estado em qualquer ponto no tempo               │
│  ✅ Debug temporal (o que aconteceu e quando)                   │
│  ❌ Complexidade de leitura (projeções necessárias)             │
│  ❌ Eventual consistency entre projections                      │
│  ❌ Schema evolution de eventos                                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Database per Service (Microservices)

```
┌──────────────────────────────────────────────────────────────────┐
│          DATABASE PER SERVICE                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Princípio: Cada microservice é dono do seu banco de dados.     │
│  Nenhum outro service acessa o banco diretamente.               │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│  │ Orders   │  │ Users    │  │ Products │                     │
│  │ Service  │  │ Service  │  │ Service  │                     │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                     │
│       │             │             │                              │
│  ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐                     │
│  │PostgreSQL│  │PostgreSQL│  │ MongoDB  │                     │
│  │ (Orders) │  │ (Users)  │  │(Products)│                     │
│  └──────────┘  └──────────┘  └──────────┘                     │
│                                                                  │
│  Comunicação entre services:                                    │
│  • API calls (sync)                                              │
│  • Events/Messages (async) — preferível                         │
│  • Saga pattern para transações distribuídas                    │
│                                                                  │
│  Dados compartilhados:                                          │
│  • Event-driven replication (cada service projeta o que precisa)│
│  • API Composition para queries cross-service                   │
│  • Shared data via eventos (não shared database!)               │
│                                                                  │
│  ✅ Isolamento e autonomia por time                             │
│  ✅ Evolução independente de schema                             │
│  ✅ Polyglot persistence natural                                │
│  ❌ Consistência eventual entre services                        │
│  ❌ Queries cross-service são complexas                         │
│  ❌ Operacional: N bancos para gerenciar                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Connection Patterns

```
┌──────────────────────────────────────────────────────────────────┐
│          CONNECTION PATTERNS                                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Connection Pooling                                          │
│     App ──► Pool (HikariCP) ──► Database                       │
│     • Reutiliza conexões TCP/SSL                                │
│     • Pool size = CPU cores × 2 + effective_spindle_count       │
│                                                                  │
│  2. Connection Proxy                                            │
│     App ──► Proxy (PgBouncer) ──► Database                     │
│     • Multiplexing: centenas de app connections → poucas DB     │
│     • Modos: session, transaction, statement                    │
│     • Ideal para serverless (Lambda → PgBouncer → RDS)         │
│                                                                  │
│  3. Database Proxy (Cloud)                                      │
│     App ──► RDS Proxy ──► Aurora/RDS                           │
│     • Connection pooling gerenciado                             │
│     • Failover handling automático                              │
│     • IAM authentication                                        │
│     • Ideal para Lambda + RDS                                   │
│                                                                  │
│  4. Sidecar Proxy                                               │
│     App + Sidecar (in pod) ──► Database                        │
│     • Cloud SQL Auth Proxy (GCP)                                │
│     • Istio para mTLS                                           │
│     • Transparente para a aplicação                             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Performance Patterns

### Materialized Views

```sql
-- PostgreSQL — Materialized View para dashboard
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT 
    DATE_TRUNC('day', created_at) AS sale_date,
    category_id,
    COUNT(*) AS total_orders,
    SUM(total_amount) AS revenue,
    AVG(total_amount) AS avg_order_value
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE o.status = 'completed'
GROUP BY DATE_TRUNC('day', created_at), category_id
WITH DATA;

-- Índice na materialized view
CREATE UNIQUE INDEX idx_mv_daily_sales 
    ON mv_daily_sales(sale_date, category_id);

-- Refresh (pode ser agendado via pg_cron)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales;
-- CONCURRENTLY: não bloqueia reads durante refresh
```

### Batch Processing

```sql
-- ✅ Batch update com controle de tamanho
-- Evita transações gigantes que lockam a tabela toda
DO $$
DECLARE
    batch_size INT := 5000;
    affected INT := 1;
BEGIN
    WHILE affected > 0 LOOP
        WITH batch AS (
            SELECT id FROM large_table
            WHERE status = 'old'
            LIMIT batch_size
            FOR UPDATE SKIP LOCKED
        )
        UPDATE large_table SET status = 'migrated'
        WHERE id IN (SELECT id FROM batch);
        
        GET DIAGNOSTICS affected = ROW_COUNT;
        COMMIT;  -- Libera locks a cada batch
        PERFORM pg_sleep(0.1);  -- Pausa para não sobrecarregar
    END LOOP;
END $$;
```

### Pagination Patterns

```
┌──────────────────────────────────────────────────────────────────┐
│          PAGINATION PATTERNS                                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Offset-Based (simples, lento para páginas profundas)        │
│     SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 100;    │
│     ❌ OFFSET 100000 → lê 100020 linhas                        │
│                                                                  │
│  2. Keyset (Cursor-Based) — RECOMENDADO                         │
│     SELECT * FROM products                                       │
│     WHERE id > :last_seen_id                                    │
│     ORDER BY id LIMIT 20;                                        │
│     ✅ Performance constante independente da página              │
│                                                                  │
│  3. Seek Method (multi-column)                                   │
│     SELECT * FROM products                                       │
│     WHERE (created_at, id) < (:last_date, :last_id)            │
│     ORDER BY created_at DESC, id DESC                           │
│     LIMIT 20;                                                    │
│     ✅ Para ordenação por colunas não-únicas                    │
│                                                                  │
│  4. DynamoDB Pagination                                          │
│     Query com LastEvaluatedKey → ExclusiveStartKey              │
│     Sem OFFSET — sempre cursor-based                            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Bulk Operations

```sql
-- ✅ Bulk INSERT eficiente (PostgreSQL)
INSERT INTO events (type, data, created_at)
VALUES 
    ('click', '{"page": "/home"}', NOW()),
    ('click', '{"page": "/products"}', NOW()),
    ('purchase', '{"amount": 100}', NOW())
ON CONFLICT DO NOTHING;  -- Skip duplicatas

-- ✅ COPY para loads massivos (PostgreSQL)
-- 10-100x mais rápido que INSERT individual
COPY events (type, data, created_at) 
FROM '/tmp/events.csv' WITH CSV HEADER;

-- ✅ Bulk upsert com UNNEST (PostgreSQL)
INSERT INTO products (sku, name, price)
SELECT * FROM UNNEST(
    ARRAY['SKU-1', 'SKU-2', 'SKU-3'],
    ARRAY['Product 1', 'Product 2', 'Product 3'],
    ARRAY[10.00, 20.00, 30.00]::DECIMAL[]
)
ON CONFLICT (sku) DO UPDATE SET
    name = EXCLUDED.name,
    price = EXCLUDED.price;
```

---

## Infinite Scale — Patterns para Escala Massiva

```
┌──────────────────────────────────────────────────────────────────┐
│          INFINITE SCALE — PATTERNS                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Serverless / Auto-Scaling Databases                         │
│     • DynamoDB On-Demand: 0 → milhões de TPS                   │
│     • Aurora Serverless v2: 0.5 → 128 ACUs                     │
│     • Cosmos DB: auto-scale RU/s por container                  │
│     • CockroachDB Serverless                                    │
│     → Pague pelo uso; escala transparente                      │
│                                                                  │
│  2. Tiered Architecture                                         │
│     Hot  → Redis / DynamoDB (< 10ms, dados recentes)           │
│     Warm → PostgreSQL / MongoDB (< 100ms, dados ativos)        │
│     Cold → S3 + Athena / Glacier (segundos, dados históricos)  │
│     → Custo otimizado por temperatura do dado                  │
│                                                                  │
│  3. Event-Driven + CQRS                                        │
│     Writers → Event Bus → Multiple Read Stores                 │
│     → Cada read store escala independentemente                 │
│     → Read stores podem ser reconstruídos dos eventos          │
│                                                                  │
│  4. Global Distribution                                         │
│     • DynamoDB Global Tables (multi-region, active-active)     │
│     • Cosmos DB (multi-region, 5 consistency levels)           │
│     • CockroachDB (geo-partitioned, follower reads)            │
│     • Spanner (global strong consistency com TrueTime)         │
│     → Dados perto do usuário, globally consistent              │
│                                                                  │
│  5. Data Partitioning + Federation                              │
│     • Dados particionados por tenant/região/tempo              │
│     • Federation layer para queries cross-partition             │
│     • CDC para sincronizar stores especializados               │
│     → Cada partição escala independentemente                   │
│                                                                  │
│  6. Log-Structured Storage                                      │
│     • LSM-tree based: writes sempre sequenciais → rápidos      │
│     • Compaction em background                                  │
│     • Cassandra, RocksDB, LevelDB, ScyllaDB                    │
│     → Write throughput quase ilimitado                          │
│                                                                  │
│  7. Time-Series Partitioning                                    │
│     • Dados naturalmente particionados por tempo               │
│     • Partições antigas → compactadas/arquivadas               │
│     • Retenção automática: DROP PARTITION vs DELETE             │
│     → Tabelas nunca crescem além do janela de retenção         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Efficiency — Otimização de Custos e Recursos

```
┌──────────────────────────────────────────────────────────────────┐
│          EFFICIENCY — OTIMIZAÇÃO                                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Right-Sizing                                                │
│     • Monitorar CPU, memória, IOPS e connections reais          │
│     • Muitos DBs são over-provisioned (pagando por 80% idle)   │
│     • Aurora Serverless v2: escala de 0.5 a 128 ACUs           │
│     • DynamoDB On-Demand vs Provisioned: avaliar workload      │
│                                                                  │
│  2. Storage Optimization                                        │
│     • Compressão: TOAST (PostgreSQL), InnoDB page compression  │
│     • Arquivamento: mover dados antigos para cold storage      │
│     • Partitioning: DROP PARTITION vs DELETE (sem VACUUM)       │
│     • TTL nativo (DynamoDB, Redis, Cassandra)                  │
│                                                                  │
│  3. Query Efficiency                                            │
│     • SELECT apenas colunas necessárias (não SELECT *)         │
│     • Covering indexes (evitar table lookups)                  │
│     • Connection pooling (menos conexões ociosas)              │
│     • Prepared statements (cache de query plan)                │
│     • Batch operations (reduz round-trips)                     │
│                                                                  │
│  4. Caching ROI                                                 │
│     • Cache hit ratio > 95% → bom ROI                          │
│     • Cache hit ratio < 50% → rever estratégia de caching     │
│     • Monitorar cache evictions e memory usage                 │
│                                                                  │
│  5. Reserved Capacity                                           │
│     • RDS Reserved Instances: até 60% de desconto              │
│     • DynamoDB Reserved Capacity: até 77% de desconto          │
│     • ElastiCache Reserved Nodes: até 55% de desconto          │
│     • Avaliar para workloads estáveis (>1 ano)                 │
│                                                                  │
│  6. Read Replicas para Offloading                               │
│     • Queries de reporting → read replica (não primary)        │
│     • Analytics → export para OLAP ou data lake                │
│     • CDC → stream para consumidores em vez de polling         │
│                                                                  │
│  7. Connection Efficiency                                       │
│     • Pool size adequado (não excessivo)                        │
│     • PgBouncer/RDS Proxy para serverless                      │
│     • Idle connection timeout configurado                       │
│     • Monitoring de connection leaks                            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flexibility — Patterns para Evolução

```
┌──────────────────────────────────────────────────────────────────┐
│          FLEXIBILITY — EVOLUÇÃO E ADAPTABILIDADE                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Schema Evolution                                            │
│     SQL:                                                         │
│     • Expand-and-Contract para zero-downtime changes            │
│     • Migrations versionadas e idempotentes                     │
│     • JSONB columns para dados semi-estruturados                │
│     NoSQL:                                                       │
│     • Schema version field em cada documento                    │
│     • Additive changes only (nunca remover campos)              │
│     • Lazy migration (migrar on-read)                           │
│                                                                  │
│  2. Polyglot Persistence Strategy                               │
│     • Começar com 1-2 bancos (PostgreSQL + Redis)              │
│     • Adicionar bancos especializados conforme necessidade      │
│     • Abstração: Repository pattern para trocar implementação   │
│     • CDC como backbone de sincronização                        │
│                                                                  │
│  3. Multi-Model Databases                                       │
│     • PostgreSQL: relational + JSONB + full-text + geo          │
│     • ArangoDB: document + graph + key-value                    │
│     • Cosmos DB: document + graph + column-family + table       │
│     → Reduz número de bancos mantendo flexibilidade            │
│                                                                  │
│  4. Data Abstraction Layer                                      │
│     • Repository Pattern: abstrai o banco da lógica             │
│     • Port & Adapter: banco é detalhe de infra                 │
│     • API Gateway para dados: GraphQL Federation                │
│     → Trocar banco sem mudar lógica de negócio                 │
│                                                                  │
│  5. Feature Flags para Migrations                               │
│     • Testar novo banco com % do tráfego                       │
│     • Dual-write + comparação de resultados                    │
│     • Rollback instantâneo via flag                             │
│     → Migração gradual e segura                                │
│                                                                  │
│  6. Database-Agnostic Patterns                                  │
│     • Idempotent operations: retry-safe                         │
│     • Outbox pattern: transactional messaging                  │
│     • Saga pattern: distributed transactions                   │
│     • CDC: Change Data Capture como interface universal         │
│     → Patterns que funcionam com qualquer banco                │
│                                                                  │
│  7. Backwards-Compatible Changes                                │
│     • Novos campos são sempre OPTIONAL                          │
│     • Campos removidos viram deprecated primeiro                │
│     • Enum values: só adicionar, nunca remover                  │
│     • API versioning alinhado com schema versioning             │
│     → Deploy independente de schema e aplicação                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Disaster Recovery

```
┌──────────────────────────────────────────────────────────────────┐
│          DISASTER RECOVERY — DATABASES                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RPO (Recovery Point Objective): Quantos dados posso perder?    │
│  RTO (Recovery Time Objective): Quanto tempo para recuperar?    │
│                                                                  │
│  ┌──────────────┬──────────┬──────────┬─────────────────────┐  │
│  │ Tier         │ RPO      │ RTO      │ Estratégia          │  │
│  ├──────────────┼──────────┼──────────┼─────────────────────┤  │
│  │ Tier 1       │ 0        │ Minutos  │ Multi-AZ Sync       │  │
│  │ (Crítico)    │          │          │ + Cross-Region      │  │
│  │              │          │          │   Replica            │  │
│  ├──────────────┼──────────┼──────────┼─────────────────────┤  │
│  │ Tier 2       │ < 1h     │ < 1h     │ Multi-AZ +          │  │
│  │ (Importante) │          │          │ Automated Backups   │  │
│  ├──────────────┼──────────┼──────────┼─────────────────────┤  │
│  │ Tier 3       │ < 24h    │ < 4h     │ Daily Snapshots     │  │
│  │ (Normal)     │          │          │ + Cross-Region Copy │  │
│  ├──────────────┼──────────┼──────────┼─────────────────────┤  │
│  │ Tier 4       │ < 72h    │ < 24h    │ Weekly Backups      │  │
│  │ (Low)        │          │          │                     │  │
│  └──────────────┴──────────┴──────────┴─────────────────────┘  │
│                                                                  │
│  Backups:                                                        │
│  • Automated: RDS automated backups, DynamoDB PITR             │
│  • Snapshots: manual ou scheduled, cross-region copy           │
│  • Logical: pg_dump, mysqldump (portáveis mas lentos)          │
│  • Physical: snapshot de storage (rápido, vendor-specific)     │
│                                                                  │
│  Testes:                                                         │
│  • Testar restore MENSALMENTE                                    │
│  • Medir RTO real (não estimado)                                │
│  • Game days / Chaos Engineering em DR                          │
│  • Runbooks documentados e praticados                           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Anti-patterns de Escalabilidade

| # | Anti-pattern | Problema | Solução |
|---|-------------|----------|---------|
| 1 | **Premature Sharding** | Sharding com < 100M rows, complexidade sem necessidade | Scale up + read replicas + cache primeiro |
| 2 | **Shared Database** | Múltiplos services no mesmo DB → acoplamento | Database per service com CDC |
| 3 | **Cache as Source of Truth** | Confiar no cache como dado principal | Cache é aceleração; DB é verdade |
| 4 | **N+1 in Distributed Systems** | Service A chama Service B N vezes | Batch API, data loading, GraphQL DataLoader |
| 5 | **Unbounded Queries** | `SELECT * FROM orders` sem LIMIT | Sempre paginar; sempre LIMIT |
| 6 | **Missing Circuit Breaker** | DB lento → cascading failure → downtime total | Circuit breaker + fallback + timeout |
| 7 | **No Connection Limits** | App abre conexões sem limite → DB sobrecarregado | Pool sizing + proxy + max_connections adequado |
| 8 | **Sync Distributed TX** | 2PC bloqueante entre services | Saga pattern, eventual consistency |
| 9 | **No Read Replicas** | All reads no primary → primary sobrecarregado | Read replicas para queries de leitura |
| 10 | **Backup Without Restore Test** | Backups existem mas nunca testados | Restore test mensal com metrics de RTO/RPO |

---

## Referências

- Kleppmann, M. (2017). *Designing Data-Intensive Applications* — O'Reilly
- Newman, S. (2021). *Building Microservices, 2nd Edition* — O'Reilly
- Nygard, M. (2018). *Release It!, 2nd Edition* — Pragmatic Bookshelf
- Amazon. *DynamoDB Best Practices* — https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/
- Amazon. *Aurora Best Practices* — https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/
- Redis. *Data Types Tutorial* — https://redis.io/docs/data-types/tutorial/
- Martin Fowler. *CQRS* — https://martinfowler.com/bliki/CQRS.html
- Microsoft. *Cloud Design Patterns* — https://learn.microsoft.com/en-us/azure/architecture/patterns/
