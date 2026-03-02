# Level 6 — Distributed Theory: CAP, ACID vs BASE, Consistent Hashing

> **Objetivo:** Demonstrar na prática o teorema CAP, implementar consistent hashing
> com virtual nodes e construir um store com semântica ACID e outro com BASE.

**Referência:**
- [11-cap-theorem.md](../../.docs/SYSTEM-DESIGN/11-cap-theorem.md)
- [12-acid-vs-base.md](../../.docs/SYSTEM-DESIGN/12-acid-vs-base.md)
- [13-consistent-hashing.md](../../.docs/SYSTEM-DESIGN/13-consistent-hashing.md)

**Pré-requisito:** Level 5 completo.

---

## Parte 1 — ADR: Consistência vs Disponibilidade

**Arquivo:** `docs/adrs/ADR-001-consistency-vs-availability.md`

**Decisão:** Para cada componente do sistema, escolher entre CP (Consistency + Partition tolerance) e AP (Availability + Partition tolerance).

**Options:**
1. **CP System** — rejeita writes durante partição (ex: Zookeeper, HBase)
2. **AP System** — aceita writes, resolve conflitos depois (ex: Cassandra, DynamoDB)
3. **Tunable Consistency** — quorum reads/writes (ex: Cassandra com CL=QUORUM)

**Decision Drivers:**
- Requisitos de negócio (pagamento = CP, timeline = AP)
- Latência aceitável
- Complexidade de conflict resolution
- SLA de disponibilidade

**Critérios de aceite:**
- [ ] Classificação CP vs AP para 5+ componentes reais
- [ ] ACID vs BASE trade-offs tabulados
- [ ] Cenários de network partition documentados
- [ ] Quorum math explicado (R + W > N)

---

## Parte 2 — Diagrama DrawIO

**Arquivo:** `docs/diagrams/06-distributed-theory.drawio`

**View 1 — CAP Triangle:**
```
              Consistency
                 /\
                /  \
               /    \
              / CP   \
             /________\
            /    CA    \
    Avail. /____________\ Partition Tol.
            \    AP    /
             \________/
```

**View 2 — Consistent Hashing Ring:**
```
              0°
              │
     Node A ──┤── VNode A1
              │
     90° ─────┤── Node B
              │── VNode B1
              │
    180° ─────┤── Node C
              │── VNode A2
              │
    270° ─────┤── VNode C1
              │── VNode B2
              │
```

**View 3 — ACID Transaction Flow vs BASE Eventual Consistency Flow**

**Critérios de aceite:**
- [ ] CAP com exemplos reais em cada região
- [ ] Consistent hashing ring com virtual nodes
- [ ] ACID vs BASE comparison side-by-side

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/
│   ├── cap-demo/main.go
│   └── hashring/main.go
├── internal/
│   ├── hashring/
│   │   ├── ring.go              ← Consistent hash ring
│   │   ├── ring_test.go
│   │   ├── vnode.go             ← Virtual nodes
│   │   └── benchmark_test.go
│   ├── store/
│   │   ├── acid.go              ← ACID store (transactions)
│   │   ├── base.go              ← BASE store (eventual consistency)
│   │   ├── quorum.go            ← Quorum reads/writes
│   │   └── conflict.go          ← Conflict resolution (LWW, vector clocks)
│   └── cap/
│       ├── simulator.go         ← Network partition simulator
│       ├── cp_node.go           ← CP behavior node
│       └── ap_node.go           ← AP behavior node
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **Consistent Hash Ring** com virtual nodes (150+ vnodes per node)
2. **Node addition/removal** com minimal key redistribution
3. **ACID Store** com transaction isolation (serializable)
4. **BASE Store** com eventual consistency e conflict resolution
5. **Quorum** read/write (configurable R, W, N)
6. **Vector Clocks** para conflict detection
7. **Last-Write-Wins** como conflict resolution strategy
8. **CAP Simulator** que simula network partitions e mostra trade-offs

**Critérios de aceite Go:**
- [ ] Hash ring distribui keys uniformemente (< 10% deviation com vnodes)
- [ ] Adding/removing node redistribui apenas K/N keys
- [ ] ACID store: read-after-write consistency demonstrada
- [ ] BASE store: eventual consistency demonstrada (com delay)
- [ ] Quorum: R=2, W=2, N=3 garante strong consistency
- [ ] Vector clocks detectam conflitos corretamente
- [ ] CAP simulator mostra CP vs AP behavior em partição
- [ ] Benchmarks para hash ring (1M keys, vary nodes)

---

### 3.2 — Java

**Funcionalidades Java:**
1. **Consistent Hash Ring** com `TreeMap` para O(log N) lookup
2. **Virtual Nodes** com configurable replication factor
3. **ACID Store** com `ReentrantReadWriteLock`
4. **BASE Store** com `CompletableFuture` para async replication
5. **Quorum** com Virtual Threads para parallel reads/writes
6. **Sealed interface** para conflict resolution strategies

**Critérios de aceite Java:**
- [ ] Hash ring com TreeMap
- [ ] ACID vs BASE demonstrado com testes
- [ ] Quorum implementation
- [ ] Vector clocks ou LWW
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] ADR documentando CP vs AP decisions
- [ ] DrawIO com 3 views
- [ ] Go e Java: hash ring + ACID/BASE stores + quorum + CAP simulator
- [ ] Relatório comparativo ACID vs BASE com métricas
- [ ] Commit: `feat(system-design-06): cap theorem acid base consistent hashing`
