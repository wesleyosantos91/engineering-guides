# Level 5 — Database Sharding & SQL vs NoSQL

> **Objetivo:** Implementar sharding strategies (range, hash, directory) e construir
> adaptadores para SQL e NoSQL, documentando decisões de quando usar cada um.

**Referência:**
- [09-database-sharding.md](../../.docs/SYSTEM-DESIGN/09-database-sharding.md)
- [10-sql-vs-nosql.md](../../.docs/SYSTEM-DESIGN/10-sql-vs-nosql.md)

**Pré-requisito:** Level 4 completo.

---

## Parte 1 — ADR: Sharding Strategy & Database Selection

**Arquivo:** `docs/adrs/ADR-001-sharding-and-database-selection.md`

**Decisões:**
1. Estratégia de sharding (range-based vs hash-based vs directory-based)
2. Shard key selection
3. SQL vs NoSQL para diferentes partes do sistema

**Options — Sharding:**
1. **Range-based** — shards por faixa de valores (ex: A-M, N-Z)
2. **Hash-based** — hash do shard key mod N
3. **Directory-based** — lookup table para mapear key → shard
4. **Geo-based** — shards por região geográfica

**Options — Database:**
1. **PostgreSQL** (SQL) — transações ACID, queries complexas
2. **MongoDB** (Document NoSQL) — schema flexível, horizontal scaling
3. **Cassandra** (Wide-column) — write-heavy, alta disponibilidade
4. **Redis** (Key-Value) — cache, sessões, real-time

**Critérios de aceite:**
- [ ] Comparativo de sharding strategies com cenários de uso
- [ ] Shard key selection criteria documentados
- [ ] Decisão SQL vs NoSQL justificada por workload
- [ ] Problemas de cross-shard queries documentados

---

## Parte 2 — Diagrama DrawIO

**Arquivo:** `docs/diagrams/05-sharding-architecture.drawio`

**View 1 — Hash-Based Sharding:**
```
              ┌────────────┐
              │  Router /  │
              │  Proxy     │
              └─────┬──────┘
                    │ hash(key) % 4
        ┌───────┬───┴───┬───────┐
        │       │       │       │
   ┌────▼──┐┌──▼───┐┌──▼───┐┌──▼───┐
   │Shard 0││Shard1││Shard2││Shard3│
   │[0-24] ││[25-49││[50-74││[75-99│
   │       ││]     ││]     ││]     │
   └───────┘└──────┘└──────┘└──────┘
```

**View 2 — Resharding (adding new shard):** Processo de split e data migration

**View 3 — SQL vs NoSQL Decision Tree:** Flowchart de decisão

**Critérios de aceite:**
- [ ] Sharding com router e multiple shards
- [ ] Resharding flow documentado
- [ ] Decision tree SQL vs NoSQL

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/
│   ├── router/main.go             ← Shard router/proxy
│   └── demo/main.go               ← Demo SQL vs NoSQL
├── internal/
│   ├── shard/
│   │   ├── strategy.go            ← Interface ShardStrategy
│   │   ├── hash.go                ← Hash-based sharding
│   │   ├── range.go               ← Range-based sharding
│   │   ├── directory.go           ← Directory-based sharding
│   │   ├── router.go              ← Shard router
│   │   ├── rebalancer.go          ← Shard rebalancing
│   │   └── strategy_test.go
│   ├── store/
│   │   ├── store.go               ← Interface Store
│   │   ├── postgres.go            ← PostgreSQL adapter
│   │   ├── mongo.go               ← MongoDB adapter
│   │   ├── redis.go               ← Redis adapter
│   │   └── store_test.go
│   └── migration/
│       ├── migrator.go            ← Data migration between shards
│       └── migrator_test.go
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **Shard Router** que distribui queries para shards corretos
2. **Hash Sharding** com consistent hashing (evita resharding massivo)
3. **Range Sharding** com configuração de ranges
4. **Directory Sharding** com lookup table
5. **Store Interface** unificada para SQL e NoSQL
6. **PostgreSQL adapter** (SQL — `pgx`)
7. **MongoDB adapter** (NoSQL — `mongo-driver`)
8. **Redis adapter** (Key-Value)
9. **Data Migration** tool para resharding
10. **Cross-shard query** com scatter-gather pattern

**Critérios de aceite Go:**
- [ ] 3 sharding strategies implementadas
- [ ] Router distribuindo corretamente para N shards
- [ ] Adapters para PostgreSQL, MongoDB e Redis
- [ ] Interface unificada `Store` com operações CRUD
- [ ] Resharding sem downtime (dual-write + migration)
- [ ] Cross-shard aggregation (scatter-gather)
- [ ] Testes com Testcontainers (postgres, mongo, redis)
- [ ] ≥ 15 testes unitários + 5 integration

---

### 3.2 — Java

**Funcionalidades Java:**
1. **Shard Router** com Spring Boot
2. **3 strategies** como `@Component` beans
3. **Spring Data JPA** para PostgreSQL
4. **Spring Data MongoDB** para MongoDB
5. **Spring Data Redis** para Redis
6. **Repository Pattern** unificado
7. **Virtual Threads** para cross-shard queries paralelas
8. **Sealed interface** para shard strategies

**Critérios de aceite Java:**
- [ ] 3 sharding strategies com Strategy Pattern
- [ ] Spring Data para 3 databases
- [ ] Testes com Testcontainers
- [ ] Cross-shard query com `CompletableFuture`
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] ADR com decisões de sharding e database selection
- [ ] DrawIO com 3 views
- [ ] Go e Java: 3 strategies + multi-db adapters + migration + tests
- [ ] Comparativo SQL vs NoSQL com dados de benchmark
- [ ] Commit: `feat(system-design-05): database sharding and sql vs nosql`
