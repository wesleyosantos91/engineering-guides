# Level 10 — Distributed Algorithms: Consensus, Gossip & Bloom Filters

> **Objetivo:** Implementar Raft consensus (simplified), Gossip protocol para membership
> e propagação de informação, e Bloom Filters para membership testing probabilístico.

**Referência:**
- [21-consensus-algorithms.md](../../.docs/SYSTEM-DESIGN/21-consensus-algorithms.md)
- [22-gossip-protocol.md](../../.docs/SYSTEM-DESIGN/22-gossip-protocol.md)
- [23-bloom-filters.md](../../.docs/SYSTEM-DESIGN/23-bloom-filters.md)

**Pré-requisito:** Level 9 completo.

---

## Parte 1 — ADR: Distributed Algorithm Selection

**Arquivo:** `docs/adrs/ADR-001-distributed-algorithms.md`

**Decisões:**
1. Consensus algorithm (Raft vs Paxos vs ZAB)
2. Gossip protocol use cases
3. Bloom filter sizing e false positive rate

**Options — Consensus:**
1. **Raft** — understandable consensus (leader-based)
2. **Multi-Paxos** — original, mais flexível, mais complexo
3. **ZAB** — ZooKeeper protocol (Raft-like)
4. **PBFT** — Byzantine fault tolerance (3f+1 nodes)

**Critérios de aceite:**
- [ ] Raft vs Paxos trade-offs documentados
- [ ] Gossip convergence time analysis
- [ ] Bloom filter false positive rate calculado para target size
- [ ] Cenários de uso real para cada algoritmo

---

## Parte 2 — Diagrama DrawIO

**Arquivo:** `docs/diagrams/10-distributed-algorithms.drawio`

**View 1 — Raft Consensus:**
```
Term 1:  Leader Election
   Node A ──▶ RequestVote ──▶ Node B (votes yes)
   Node A ──▶ RequestVote ──▶ Node C (votes yes)
   Node A becomes LEADER (2/3 majority)

Term 1:  Log Replication
   Leader A ──▶ AppendEntries ──▶ Node B (appends)
   Leader A ──▶ AppendEntries ──▶ Node C (appends)
   Committed when majority acknowledges
```

**View 2 — Gossip Protocol Propagation:** Estado se espalha exponencialmente (O(log N) rounds)

**View 3 — Bloom Filter Structure:**
```
Insert "hello": hash1("hello")=2, hash2("hello")=5, hash3("hello")=9
Bit Array: [0, 0, 1, 0, 0, 1, 0, 0, 0, 1, 0, 0]

Check "world": hash1("world")=1, hash2("world")=5, hash3("world")=7
Bits: [0, 0, 1, 0, 0, 1, 0, 0, 0, 1, 0, 0]
       bit[1]=0 → DEFINITELY NOT IN SET
```

**Critérios de aceite:**
- [ ] Raft com election + log replication
- [ ] Gossip com convergence visualization
- [ ] Bloom filter com bit array visualization

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/
│   ├── raft/main.go               ← Raft cluster demo
│   ├── gossip/main.go             ← Gossip protocol demo
│   └── bloom/main.go              ← Bloom filter demo
├── internal/
│   ├── raft/
│   │   ├── node.go                ← Raft node
│   │   ├── log.go                 ← Replicated log
│   │   ├── rpc.go                 ← RequestVote, AppendEntries RPCs
│   │   ├── state.go               ← Follower/Candidate/Leader states
│   │   ├── election.go            ← Election timer + voting
│   │   ├── replication.go         ← Log replication
│   │   └── raft_test.go
│   ├── gossip/
│   │   ├── node.go                ← Gossip node
│   │   ├── protocol.go           ← Push, Pull, Push-Pull gossip
│   │   ├── membership.go          ← Cluster membership
│   │   ├── detector.go            ← SWIM failure detector
│   │   └── gossip_test.go
│   ├── bloom/
│   │   ├── filter.go              ← Standard Bloom filter
│   │   ├── counting.go            ← Counting Bloom filter (supports delete)
│   │   ├── scalable.go            ← Scalable Bloom filter (grows)
│   │   └── bloom_test.go
│   └── transport/
│       └── tcp.go                 ← TCP transport for inter-node comm
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **Raft Node** com states (Follower, Candidate, Leader)
2. **Leader Election** com randomized election timeout
3. **Log Replication** com AppendEntries RPC
4. **Log Compaction** (snapshotting)
5. **Gossip Protocol** (push, pull, push-pull modes)
6. **SWIM Failure Detector** (ping, ping-req, suspect)
7. **Cluster Membership** via gossip (join, leave, suspect, dead)
8. **Standard Bloom Filter** com configurable hash functions
9. **Counting Bloom Filter** com delete support
10. **Scalable Bloom Filter** que cresce automaticamente

**Critérios de aceite Go:**
- [ ] Raft: 3-node cluster elege leader corretamente
- [ ] Raft: log replication com majority commit
- [ ] Raft: leader failure → new election em < 2s
- [ ] Gossip: informação converge em O(log N) rounds
- [ ] SWIM: detecta node failure em < 5s
- [ ] Bloom: 0 false negatives (mathematically guaranteed)
- [ ] Bloom: false positive rate < target (ex: < 1%)
- [ ] Counting Bloom: delete funciona corretamente
- [ ] Benchmarks para cada data structure
- [ ] ≥ 20 testes

---

### 3.2 — Java

**Funcionalidades Java:**
1. **Raft** com Virtual Threads para RPCs assíncronos
2. **Gossip** com `DatagramSocket` (UDP)
3. **Bloom Filter** com `BitSet`
4. **Records** para RPC messages
5. **Sealed interface** para node states (Follower, Candidate, Leader)
6. **JMH Benchmarks** para Bloom filter operations

**Critérios de aceite Java:**
- [ ] Raft cluster funcional
- [ ] Gossip com UDP
- [ ] Bloom filter com BitSet
- [ ] Sealed interfaces e records
- [ ] JaCoCo ≥ 80%

---

## Parte 4 — Experimentos

### Raft Chaos Testing
- Kill leader → medir tempo de eleição
- Network partition → validar split-brain prevention
- Slow follower → validar log catch-up

### Gossip Convergence
- 10 nodes, medir rounds para convergência
- 50 nodes, medir rounds para convergência
- Plotar: nodes vs rounds (deve ser logarítmico)

### Bloom Filter Analysis
- Medir false positive rate vs items inseridos
- Comparar: Standard vs Counting vs Scalable (memory usage)

---

## Definição de Pronto (DoD)

- [ ] ADR com seleção de algoritmos
- [ ] DrawIO com 3 views
- [ ] Go e Java: Raft + Gossip + Bloom filters + tests
- [ ] Chaos testing report (Raft)
- [ ] Convergence analysis (Gossip)
- [ ] False positive analysis (Bloom)
- [ ] Commit: `feat(system-design-10): consensus gossip bloom filters`
