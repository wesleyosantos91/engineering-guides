# 33. Replication Strategies

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial para sistemas distribuídos  
> **Complexidade:** Média-Alta

---

## Definição

**Replication** é o processo de manter **cópias idênticas de dados** em múltiplos nós, para garantir **alta disponibilidade**, **tolerância a falhas** e **performance de leitura**. A escolha da estratégia de replicação define o trade-off entre **consistência** e **latência/disponibilidade**.

---

## Por Que é Importante?

- **Disponibilidade** — se um nó morre, replicas servem os dados
- **Performance de leitura** — distribui reads entre replicas
- **Disaster recovery** — dados sobrevivem a falhas de datacenter
- **Geo-distribution** — replicas próximas ao usuário = menor latência
- **Trade-off central no CAP Theorem** — consistência vs disponibilidade

---

## Tipos de Replicação

### 1. Synchronous Replication

```
Write é confirmado SOMENTE quando TODAS as replicas persistiram:

  Client ──write──▶ Primary ───sync──▶ Replica 1 ✅
                       │         └────▶ Replica 2 ✅
                       │
                  Espera AMBAS confirmarem
                       │
                  ◀── ACK ao Client

  Timeline:
  ├─ Client envia write ─────────────────────┤
  │  Primary escreve localmente  │           │
  │  Propaga para Replica 1 ─────┤           │
  │  Propaga para Replica 2 ─────┤           │
  │  Ambas confirmam ────────────┤           │
  │  ACK ao Client ──────────────────────────┤
  
  Latência total = Primary write + MAX(Replica 1, Replica 2) + network

  ✅ Strong consistency (todas as replicas SEMPRE em sync)
  ✅ Zero data loss (RPO = 0)
  ❌ Alta latência (espera o mais lento)
  ❌ Disponibilidade reduzida (1 replica down → writes bloqueiam)
```

### 2. Asynchronous Replication

```
Write é confirmado assim que o Primary persiste:

  Client ──write──▶ Primary ──ACK──▶ Client  (imediato!)
                       │
                  Background:
                       ├──async──▶ Replica 1 (eventualmente)
                       └──async──▶ Replica 2 (eventualmente)

  Timeline:
  ├─ Client envia write ────────┤
  │  Primary escreve            │
  │  ACK ao Client ─────────────┤  ← RÁPIDO!
  │  ...                        │
  │  Replica 1 recebe ──────────────────┤  (lag)
  │  Replica 2 recebe ───────────────────────┤  (mais lag)

  ✅ Baixa latência (não espera replicas)
  ✅ Alta disponibilidade (replica down não afeta writes)
  ❌ Eventual consistency (replicas podem estar atrasadas)
  ❌ Possível data loss se Primary falha antes de propagar
```

### 3. Semi-Synchronous Replication

```
Write confirmado quando Primary + pelo menos 1 replica confirmam:

  Client ──write──▶ Primary ───sync──▶ Replica 1 ✅ ← Espera esta
                       │         └────async─▶ Replica 2 (background)
                       │
                  ACK quando Primary + Replica 1 confirmam
                       │
                  ◀── ACK ao Client

  ✅ Compromisso: pelo menos 1 cópia durável além do Primary
  ✅ Menor latência que sync completo
  ✅ Tolera falha de 1 replica sem perder writes
  ❌ Ainda mais lento que async puro
  
  Usado por: MySQL semi-sync replication
```

### 4. Quorum-Based Replication

```
Configurável: W writes + R reads > N total replicas

  N = 3 (total replicas)
  W = 2 (write quorum — quantas precisam confirmar write)
  R = 2 (read quorum — quantas precisam responder read)
  
  W + R > N ⟹ 2 + 2 = 4 > 3 ✅ (strong consistency)

  Write:
  Client ──write──▶ Coordinator
                       ├──▶ Node 1 ✅ (ack)
                       ├──▶ Node 2 ✅ (ack)  ← 2 acks = W: confirma!
                       └──▶ Node 3 ❌ (slow/down)
                       │
                  ◀── ACK (2 de 3 = quorum atingido)

  Read:
  Client ──read──▶ Coordinator
                       ├──▶ Node 1 → valor V1 (version 5)
                       ├──▶ Node 2 → valor V2 (version 5)  ← 2 respostas = R
                       └──▶ Node 3 → (timeout)
                  Retorna version mais recente

  Configurações comuns (N=3):
  ┌─────────────────────────────────────────────┐
  │  W=1, R=3: Fast writes, slow reads          │
  │  W=3, R=1: Slow writes, fast reads          │
  │  W=2, R=2: Balanced (mais comum)            │
  │  W=1, R=1: Fast but NO consistency ❌       │
  │            (1+1=2 ≤ 3, pode ler stale data) │
  └─────────────────────────────────────────────┘
```

### 5. Chain Replication

```
Writes vão para o HEAD da chain; reads vão para o TAIL:

  Write: Client ──▶ Head ──▶ Middle ──▶ Tail ──ACK──▶ Client
  Read:  Client ──▶ Tail ──response──▶ Client

  ┌──────┐    ┌──────┐    ┌──────┐
  │ Head │───▶│Middle│───▶│ Tail │
  │(write)│    │      │    │(read)│
  └──────┘    └──────┘    └──────┘

  ✅ Strong consistency (read do tail = sempre latest committed)
  ✅ Alto throughput (write e read em nós diferentes)
  ❌ Write latência = soma de todos os nós na chain
  ❌ Um nó down quebra a chain (precisa reconstruir)
  
  Usado por: HDFS pipeline, Microsoft FAWN
```

---

## Topologias de Replicação

### Single-Leader (Primary-Replica)

```
  ┌─────────┐
  │ Primary │ ◀── ALL writes
  │ (Leader)│
  └────┬────┘
       │ replication stream
  ┌────┼────────────┐
  ▼    ▼            ▼
┌────┐┌────┐    ┌────┐
│Rep1││Rep2│    │Rep3│  ◀── reads distribuídos
└────┘└────┘    └────┘

  ✅ Simples, conflito impossível (1 writer)
  ❌ Primary é SPOF (failover necessário)
  ❌ Write não escala (1 nó)
  
  Usado por: PostgreSQL, MySQL, MongoDB (default)
```

### Multi-Leader (Active-Active)

```
  ┌─────────┐     sync     ┌─────────┐
  │Leader 1 │◀════════════▶│Leader 2 │
  │ (DC-US) │              │ (DC-EU) │
  └────┬────┘              └────┬────┘
       │                        │
  ┌────┼────┐             ┌────┼────┐
  │Rep1│Rep2│             │Rep3│Rep4│
  └────┴────┘             └────┴────┘

  ✅ Write escala (múltiplos writers)
  ✅ Geo-distributed (write local no DC mais próximo)
  ❌ CONFLITOS possíveis (2 leaders escrevem no mesmo dado)
  ❌ Conflict resolution necessário (LWW, merge, manual)
  
  Usado por: CouchDB, Cassandra (all nodes are leaders), MySQL Group Replication
```

### Leaderless (Peer-to-Peer)

```
  ┌───────┐   ┌───────┐   ┌───────┐
  │ Node1 │◀─▶│ Node2 │◀─▶│ Node3 │
  └───────┘   └───────┘   └───────┘
       ▲           ▲           ▲
       └───── Client writes to ALL (or W nodes) ──┘

  Client envia write para TODOS os nós (ou W deles)
  Client lê de R nós e pega a versão mais recente

  ✅ Alta disponibilidade (qualquer nó aceita write)
  ✅ Sem SPOF (nenhum nó é "especial")
  ❌ Conflitos possíveis → vector clocks, LWW
  ❌ Mais complexo
  
  Usado por: DynamoDB, Cassandra, Riak
```

---

## Conflict Resolution (Multi-Leader / Leaderless)

```
Problema: Node A e Node B escrevem valores diferentes para a mesma key.

  Node A: SET user.name = "Maria" (timestamp: 100)
  Node B: SET user.name = "João"  (timestamp: 101)

Estratégias:

1. LWW (Last-Writer-Wins):
   Timestamp maior ganha → "João" (101 > 100)
   ✅ Simples  ❌ Pode perder writes

2. Vector Clocks:
   Detecta conflitos sem perder dados
   A: [A:1, B:0]  B: [A:0, B:1] → CONFLITO! (concurrent)
   App ou user resolve manualmente

3. CRDTs (Conflict-free Replicated Data Types):
   Estruturas de dados que ALWAYS merge sem conflitos
   Counter, Set, Register → merge automático

4. Application-level merge:
   App define regra de merge por domínio
   Ex: shopping cart → UNION de items
```

---

## Replication Lag e Seus Efeitos

```
Problema: Async replication → replica atrasada → leituras inconsistentes

  Cenário "Read-after-Write":
    1. User escreve (vai para Primary)
    2. User lê imediatamente (vai para Replica)
    3. Replica ainda não tem o write → user vê dado antigo!
  
  Soluções:
  ├── Read-your-own-writes: redirecionar reads do MESMO user para Primary
  ├── Monotonic reads: sticky sessions (user sempre lê da MESMA replica)
  ├── Causal consistency: track dependency entre writes e reads
  └── Sync read-after-write: bloqueia read até replica confirmar

  Cenário "Stale Read":
    1. User A escreve "status=shipped" no Primary
    2. User B lê de Replica → vê "status=processing" (stale!)
    
  Aceitável se: eventual consistency é OK (feed, analytics)
  Inaceitável se: financeiro, status de pagamento
```

---

## Failover

```
Quando o Primary falha, um Replica precisa ser promovido:

  Antes:            Primary (FAIL!) ──▶ Replica1, Replica2
  
  Failover:         Replica1 promovido → NEW Primary
                    Replica2 → segue NEW Primary
                    Old Primary (quando retorna) → torna-se Replica

  Tipos:
  1. Manual: DBA promove replica manualmente
     → Mais seguro, mais lento (minutos/horas)
     
  2. Automatic: Sentinel/orchestrator detecta falha e promove
     → Mais rápido (~seconds), risco de split-brain
     
  3. Consensus-based: Raft/Paxos elegem novo leader
     → Mais robusto, usado em sistemas avançados

  Problemas de Failover:
  ├── Async lag → dados não replicados PERDIDOS
  ├── Split-brain → dois nós acham que são Primary
  ├── Cascading failure → nova Primary sobrecarregada
  └── Client reconnection → precisa descobrir nova Primary
```

---

## Comparativo Geral

| Estratégia | Latência Write | Consistência | Disponibilidade | Data Loss Risk |
|------------|:-----------:|:------------:|:---------------:|:--------------:|
| **Sync** | 🔴 Alta | 🟢 Strong | 🔴 Baixa | 🟢 Zero |
| **Async** | 🟢 Baixa | 🔴 Eventual | 🟢 Alta | 🔴 Possível |
| **Semi-Sync** | 🟡 Média | 🟡 Mostly strong | 🟡 Média | 🟡 Baixo |
| **Quorum** | 🟡 Configurável | 🟡 Configurável | 🟡 Configurável | 🟡 Configurável |
| **Chain** | 🔴 Alta | 🟢 Strong | 🟡 Média | 🟢 Zero |

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Sync vs Async** | Sync (zero data loss, mais lento) | Async (rápido, pode perder dados) |
| **Single-leader vs Multi** | Single (simples, sem conflitos) | Multi (geo-distributed, write scaling) |
| **Quorum tuning** | W=N, R=1 (durável, write lento) | W=1, R=N (write rápido, read lento) |
| **Conflict resolution** | LWW (simples, perde dado) | CRDTs (sem perda, complexo) |
| **Failover** | Manual (seguro, lento) | Automático (rápido, risco de split-brain) |

---

## Configuração Prática

### PostgreSQL — Streaming Replication

```sql
-- No Primary: verificar status de replicação
SELECT client_addr, state, sent_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS byte_lag
FROM pg_stat_replication;
```

```ini
# postgresql.conf (Primary)
wal_level = replica
max_wal_senders = 10
synchronous_standby_names = 'replica1'
```

### MySQL — GTID Replication

```sql
-- Configurar replica com GTID
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = '10.0.1.100',
    SOURCE_USER = 'replicator',
    SOURCE_AUTO_POSITION = 1;
START REPLICA;

-- Verificar lag
SHOW REPLICA STATUS\G
-- Seconds_Behind_Source: 0  ← ideal
```

### Redis — Replication

```redis
# No replica
REPLICAOF 10.0.1.100 6379

# Verificar info
INFO replication
# role:slave
# master_link_status:up
# master_last_io_seconds_ago:0
```

---

## Uso em Big Techs

| Empresa | Topologia | Estratégia | Detalhes |
|---------|-----------|-----------|----------|
| **Google Spanner** | Multi-leader (Paxos) | Sync (TrueTime) | Strong consistency global |
| **Amazon DynamoDB** | Leaderless | Quorum (W+R>N) | Configurable per-request |
| **Cassandra** | Leaderless | Quorum | Tunable consistency levels |
| **MySQL (Meta)** | Single-leader | Semi-sync | Raft-based failover (MySQL Raft) |
| **PostgreSQL** | Single-leader | Sync/Async | Streaming replication |
| **MongoDB** | Single-leader | Async (default) | Replica sets com automatic failover |
| **CockroachDB** | Multi-leader (Raft) | Sync (Raft) | Distributed SQL with strong consistency |
| **Redis** | Single-leader | Async | Sentinel para failover |

### Google Spanner — Replicação Global

```
Spanner: banco distribuído GLOBALMENTE com STRONG CONSISTENCY

  ┌──────────────┐     Paxos     ┌──────────────┐
  │   DC-US      │◀═════════════▶│   DC-EU      │
  │ (Leader zone)│               │ (Replica)    │
  └──────┬───────┘               └──────────────┘
         │ Paxos
  ┌──────▼───────┐
  │   DC-Asia    │
  │ (Replica)    │
  └──────────────┘

  Truques:
  1. TrueTime API: GPS + atomic clocks → timestamp global preciso
  2. Commit wait: espera incerteza do clock → garante ordering
  3. Paxos per split: cada range de dados tem seu Paxos group
  
  Resultado: Transações distribuídas com strong consistency global!
  Latência: ~10-15ms (cross-region commit)
```

---

## Perguntas Frequentes em Entrevistas

1. **"Sync vs Async replication?"**
   - Sync: strong consistency, mais lento, RPO=0
   - Async: eventual consistency, mais rápido, pode perder dados

2. **"O que é replication lag?"**
   - Delay entre Primary e Replica em async replication
   - Causa stale reads, read-after-write inconsistency
   - Soluções: sticky sessions, read-your-own-writes

3. **"Quorum replication — o que é W+R>N?"**
   - N=total, W=write acks, R=read responses
   - Se W+R>N → pelo menos 1 nó tem AMBOS (latest write + read)
   - Garante que read sempre vê a escrita mais recente

4. **"Como lidar com failover?"**
   - Detectar falha (heartbeat), promover replica, redirect clients
   - Consensus-based (Raft) é o mais robusto
   - Risco: split-brain, data loss (async lag)

5. **"Single-leader vs Multi-leader?"**
   - Single: simples, sem conflitos, write não escala
   - Multi: write escala, geo-distributed, conflitos possíveis

---

## Referências

- Martin Kleppmann — *"Designing Data-Intensive Applications"*, Cap. 5: Replication
- Amazon DynamoDB Paper (2007) — *"Dynamo: Amazon's Highly Available Key-value Store"*
- Google Spanner Paper (2012) — *"Spanner: Google's Globally-Distributed Database"*
- Raft Paper — *"In Search of an Understandable Consensus Algorithm"*
- PostgreSQL Docs — *"High Availability, Load Balancing, and Replication"*
- Aphyr (Jepsen) — *"Call me maybe"* — distributed systems testing blog
