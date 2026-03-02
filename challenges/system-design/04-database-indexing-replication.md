# Level 4 — Database Indexing & Replication

> **Objetivo:** Implementar um mecanismo de indexação (B-Tree, Hash Index) e um sistema
> de database replication (leader-follower) com failover automático.

**Referência:**
- [07-database-indexing.md](../../.docs/SYSTEM-DESIGN/07-database-indexing.md)
- [08-database-replication.md](../../.docs/SYSTEM-DESIGN/08-database-replication.md)

**Pré-requisito:** Levels 0-3 completos.

---

## Parte 1 — ADR: Estratégia de Indexação e Replicação

**Arquivo:** `docs/adrs/ADR-001-indexing-replication-strategy.md`

**Decisões a documentar:**
1. Tipo de índice para diferentes workloads (B-Tree vs Hash vs LSM-Tree)
2. Topologia de replicação (single-leader vs multi-leader vs leaderless)
3. Replicação (síncrona vs assíncrona vs semi-síncrona)

**Options — Indexação:**
1. **B-Tree** — index balanceado, bom para range queries
2. **Hash Index** — O(1) lookup, sem range queries
3. **LSM-Tree** — log-structured, otimizado para write-heavy
4. **Bitmap Index** — eficiente para low-cardinality columns

**Options — Replicação:**
1. **Single-Leader** com followers síncronos
2. **Single-Leader** com followers assíncronos
3. **Multi-Leader** (multi-datacenter)
4. **Leaderless** (Dynamo-style quorum)

**Critérios de aceite:**
- [ ] Trade-offs claros entre tipos de índice
- [ ] Análise de consistência vs latência na replicação
- [ ] Cenários de failover documentados

---

## Parte 2 — Diagrama DrawIO

**Arquivo:** `docs/diagrams/04-indexing-replication.drawio`

**View 1 — B-Tree Structure:** Visualização de um B-Tree com insert/search/delete

**View 2 — Leader-Follower Replication:**
```
                    ┌─────────────────────┐
                    │    Leader (Write)    │
                    │  ┌───────────────┐  │
                    │  │  WAL (Write   │  │
                    │  │  Ahead Log)   │  │
                    │  └───────┬───────┘  │
                    └──────────┼──────────┘
                         ┌─────┼─────┐
                         │     │     │
                    ┌────▼┐ ┌─▼──┐ ┌▼────┐
                    │Foll.│ │Fol.│ │Foll.│
                    │ #1  │ │ #2 │ │ #3  │
                    │(sync│ │(asy│ │(asy │
                    │)    │ │nc) │ │nc)  │
                    └─────┘ └────┘ └─────┘
                     Read    Read   Read
```

**View 3 — Failover Sequence:** Leader failure → detection → promotion → reconfiguration

**Critérios de aceite:**
- [ ] B-Tree com visualização de nós e operações
- [ ] Replication topology clara com sync/async labels
- [ ] Failover sequence com timeline

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/
│   ├── indexdemo/main.go          ← Demo de indexação
│   └── replication/main.go        ← Demo de replicação
├── internal/
│   ├── index/
│   │   ├── btree.go               ← B-Tree implementation
│   │   ├── btree_test.go
│   │   ├── hash.go                ← Hash index
│   │   ├── hash_test.go
│   │   └── benchmark_test.go      ← Comparativo B-Tree vs Hash
│   ├── storage/
│   │   ├── page.go                ← Page-based storage (4KB pages)
│   │   ├── wal.go                 ← Write-Ahead Log
│   │   └── buffer.go              ← Buffer pool manager
│   ├── replication/
│   │   ├── leader.go              ← Leader node
│   │   ├── follower.go            ← Follower node
│   │   ├── wal_shipper.go         ← WAL-based replication
│   │   ├── failover.go            ← Automatic failover
│   │   └── replication_test.go
│   └── engine/
│       ├── engine.go              ← Mini storage engine
│       └── engine_test.go
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **B-Tree** genérico com insert, search, delete, range scan
2. **Hash Index** com linear probing e resize
3. **Write-Ahead Log** (WAL) persistente para durabilidade
4. **Leader node** que aceita writes e envia WAL para followers
5. **Follower nodes** que aplicam WAL entries (sync e async modes)
6. **Automatic failover** com heartbeat detection e leader election
7. **Buffer Pool** para cache de páginas do disco
8. **Benchmarks** comparando B-Tree vs Hash para diferentes workloads

**Critérios de aceite Go:**
- [ ] B-Tree funcional com insert, search, delete, range [a, z)
- [ ] B-Tree balanceado com splits e merges
- [ ] Hash Index com collision handling e dynamic resize
- [ ] WAL persistente com fsync
- [ ] Leader-follower replication via WAL shipping
- [ ] Failover automático (≤ 5s detection + promotion)
- [ ] Benchmark: B-Tree vs Hash (1M keys, point lookup vs range)
- [ ] ≥ 15 testes unitários

---

### 3.2 — Java

**Estrutura:**
```
java/
├── src/main/java/com/challenge/db/
│   ├── Application.java
│   ├── index/
│   │   ├── Index.java                  ← sealed interface
│   │   ├── BTreeIndex.java
│   │   ├── HashIndex.java
│   │   └── IndexEntry.java             ← Record
│   ├── storage/
│   │   ├── Page.java
│   │   ├── WriteAheadLog.java
│   │   └── BufferPool.java
│   ├── replication/
│   │   ├── LeaderNode.java
│   │   ├── FollowerNode.java
│   │   ├── WalShipper.java
│   │   ├── FailoverManager.java
│   │   └── ReplicationEvent.java       ← sealed interface
│   └── engine/
│       └── StorageEngine.java
├── src/test/java/com/challenge/db/
│   ├── BTreeIndexTest.java
│   ├── HashIndexTest.java
│   ├── ReplicationTest.java
│   └── BenchmarkTest.java              ← JMH benchmarks
└── pom.xml
```

**Funcionalidades Java:**
1. **B-Tree** com generics e `Comparable<T>`
2. **Hash Index** com `ConcurrentHashMap` para thread safety
3. **WAL** com `FileChannel` e `fsync`
4. **Replication** usando Virtual Threads para I/O
5. **Sealed interfaces** para index types e replication events
6. **Records** para index entries e WAL records
7. **JMH Benchmarks** para comparativo de performance

**Critérios de aceite Java:**
- [ ] B-Tree e Hash Index implementados com generics
- [ ] WAL com durabilidade (fsync)
- [ ] Replication funcional (leader → followers)
- [ ] Failover com Virtual Threads
- [ ] Sealed interfaces para tipos
- [ ] JMH benchmark comparativo
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] ADR com decisões de indexação e replicação
- [ ] DrawIO com 3 views
- [ ] Go: B-Tree + Hash + WAL + Replication + Failover + tests
- [ ] Java: B-Tree + Hash + WAL + Replication + Failover + tests
- [ ] Benchmark report documentado
- [ ] Commit: `feat(system-design-04): database indexing and replication`
