# 11. CAP Theorem

> **Categoria:** Fundamentos de Sistemas Distribuídos  
> **Nível:** Essencial para qualquer entrevista de System Design  
> **Complexidade:** Média-Alta (a teoria é simples, os trade-offs são profundos)

---

## Definição

O **CAP Theorem** (também chamado de **Teorema de Brewer**, proposto por Eric Brewer em 2000 e provado formalmente por Gilbert & Lynch em 2002) afirma que em um sistema distribuído é **impossível garantir simultaneamente** as três propriedades:

- **C** — Consistency (Consistência)
- **A** — Availability (Disponibilidade)
- **P** — Partition Tolerance (Tolerância a Partição)

Quando uma **partição de rede** ocorre (e em sistemas distribuídos, ela **vai** ocorrer), você é forçado a escolher entre **Consistency** ou **Availability**.

---

## Por Que é Importante?

- **Define o trade-off fundamental** de qualquer sistema distribuído
- **Guia a escolha de banco de dados** — CP vs AP
- **Pergunta clássica e obrigatória** em entrevistas de System Design
- **Impacta diretamente** a experiência do usuário sob falhas
- **Base para entender** PACELC, eventual consistency, quorum-based systems

---

## Diagrama do Teorema

```
              Consistency (C)
                   /\
                  /  \
                 / CP \        CP: Consistency + Partition Tolerance
                /______\       → Sacrifica Availability
               /\      /\
              /  \    /  \
             / CA \  / AP \    AP: Availability + Partition Tolerance
            /______\/______\   → Sacrifica Consistency
     Availability (A)    Partition
                          Tolerance (P)

  ┌─────────────────────────────────────────────────────────┐
  │  Em sistemas distribuídos reais, P é OBRIGATÓRIO.       │
  │  Partições de rede VÃO acontecer.                       │
  │  Portanto, a escolha real é entre C e A.                │
  └─────────────────────────────────────────────────────────┘
```

---

## As Três Propriedades em Detalhe

### Consistency (C) — Linearizability

Cada read retorna o **write mais recente** ou um erro. Todos os nós veem os **mesmos dados ao mesmo tempo**.

```
  Consistency FORTE (Linearizable):
  
  Client A: WRITE x=5   ──▶ [Node 1] ──replicate──▶ [Node 2]
  Client B: READ x       ──▶ [Node 2] ──▶ retorna x=5 ✅
  
  ──────────────────────────────────────────────────────────
  
  SEM Consistency:
  
  Client A: WRITE x=5   ──▶ [Node 1] ──replicating...──▶ [Node 2]
  Client B: READ x       ──▶ [Node 2] ──▶ retorna x=3 ❌ (stale!)
```

**Modelos de Consistência (do mais forte ao mais fraco):**

| Modelo | Descrição | Performance |
|--------|-----------|-------------|
| **Linearizability** | Operações parecem instantâneas e globalmente ordenadas | Mais lento |
| **Sequential Consistency** | Operações de cada processo são respeitadas em ordem | Lento |
| **Causal Consistency** | Operações causalmente relacionadas são ordenadas | Médio |
| **Eventual Consistency** | Todos os nós convergem eventualmente | Mais rápido |

### Availability (A)

**Todo request** recebe uma **resposta** (que não é um erro), sem garantia de que contém o write mais recente.

```
  Alta Availability:
  
  Client ──▶ [Node 1] ──▶ Response 200 OK ✅  (mas pode ser stale)
  Client ──▶ [Node 2] ──▶ Response 200 OK ✅  (mas pode ser stale)
  Client ──▶ [Node 3] ──▶ Response 200 OK ✅  (mas pode ser stale)
  
  Nenhum request retorna erro ou timeout
  
  ──────────────────────────────────────────────────────────
  
  SEM Availability (escolheu Consistency):
  
  Client ──▶ [Node 2] ──▶ Error 503 ❌ (node não confirmou sync)
  
  Retorna erro até ter certeza que dado é consistente
```

### Partition Tolerance (P)

O sistema continua **operando** mesmo quando mensagens entre nós são **perdidas ou atrasadas** (partição de rede).

```
  Partição de Rede:
  
  ┌────────┐                    ┌────────┐
  │ Node 1 │ ══╳══╳══╳══╳═══  │ Node 2 │
  │ x = 5  │   (rede cortada)  │ x = 3  │
  └────────┘                    └────────┘
  
  Node 1 não consegue se comunicar com Node 2.
  O que fazer?
  
  Opção CP: Recusar requests até partição ser resolvida (sacrifica A)
  Opção AP: Continuar servindo, mas dados podem divergir (sacrifica C)
```

---

## Os Três Modos: CP, AP e CA

### CP — Consistency + Partition Tolerance

**Sacrifica:** Availability  
**Comportamento:** Quando há partição, nós que não podem confirmar consistência **recusam requests** (retornam erro).

```
  Partição de rede ocorre:
  
  ┌────────┐         ╳         ┌────────┐
  │ Node 1 │ ═══════╳═════════│ Node 2 │
  │ (líder)│    partição!      │(follower)│
  └────┬───┘                   └────┬───┘
       │                            │
  Client A                     Client B
  WRITE x=5 → ✅              READ x → ❌ Error 503
  (aceita pelo líder)          (não pode confirmar
                                consistência)
  
  → Dados são CONSISTENTES
  → Mas sistema INDISPONÍVEL para leituras no Node 2
```

**Exemplos:**

| Sistema | Como é CP |
|---------|-----------|
| **MongoDB** | Write para primary; se primary cai, eleição de novo primary (downtime) |
| **HBase** | Depende do HDFS NameNode; se cai, região fica indisponível |
| **Redis Cluster** | Slots sem master ficam indisponíveis até failover |
| **Zookeeper** | Precisa de quorum para responder; sem quorum → recusa |
| **etcd** | Raft consensus; sem leader → cluster read-only ou down |
| **Google Spanner** | CP com alta disponibilidade via TrueTime |

### AP — Availability + Partition Tolerance

**Sacrifica:** Consistency (temporariamente)  
**Comportamento:** Todos os nós continuam servindo requests, mas dados podem estar **desatualizados** (eventual consistency).

```
  Partição de rede ocorre:
  
  ┌────────┐         ╳         ┌────────┐
  │ Node 1 │ ═══════╳═════════│ Node 2 │
  │ x = 5  │    partição!     │ x = 3  │
  └────┬───┘                   └────┬───┘
       │                            │
  Client A                     Client B
  WRITE x=7 → ✅              READ x → ✅ (retorna x=3, stale!)
  (aceito localmente)          (continua servindo)
  
  → Sistema DISPONÍVEL
  → Mas dados INCONSISTENTES entre nós
  → Quando partição resolve: conflict resolution (last-write-wins, CRDT, etc.)
```

**Exemplos:**

| Sistema | Como é AP |
|---------|-----------|
| **Cassandra** | Tunable consistency; por padrão eventual consistency |
| **DynamoDB** | Eventually consistent reads por padrão |
| **CouchDB** | Multi-master replication com conflict resolution |
| **Riak** | Sempre disponível; vector clocks para resolução |
| **Voldemort** | Eventual consistency com read-repair |

### CA — Consistency + Availability (sem Partition Tolerance)

**Sacrifica:** Partition Tolerance  
**Na prática:** Só funciona em **single-node** ou rede **100% confiável** (não existe em sistemas distribuídos reais).

```
  Sem partição (rede perfeita):
  
  ┌────────┐  ══════════════  ┌────────┐
  │ Node 1 │   rede confiável │ Node 2 │
  │ x = 5  │  ══════════════  │ x = 5  │
  └────────┘                   └────────┘
  
  → Consistente ✅  e  Disponível ✅
  → MAS: se a rede falhar, o sistema PARA
  
  Exemplos: PostgreSQL single-node, MySQL single-node
  (não são distribuídos, logo CAP não se aplica 100%)
```

---

## Comparativo

| Aspecto | CP | AP | CA |
|---------|----|----|-----|
| **Durante partição** | Recusa alguns requests | Serve dados possivelmente stale | N/A (sistema falha) |
| **Consistência** | Forte (linearizable) | Eventual | Forte |
| **Disponibilidade** | Parcial (pode rejeitar) | Total (sempre responde) | Total |
| **Resolução** | Espera partição resolver | Conflict resolution depois | N/A |
| **Latência** | Maior (consensus) | Menor (local read) | Baixa |
| **Use case** | Financeiro, inventário | Social feed, analytics | Single-node DB |
| **Exemplos** | Zookeeper, HBase, Spanner | Cassandra, DynamoDB, Riak | PostgreSQL (single) |

---

## PACELC — Extensão do CAP

O **PACELC** (proposto por Daniel Abadi) estende o CAP para cenários **sem partição**:

> **P**artition → escolha entre **A**vailability e **C**onsistency  
> **E**lse (sem partição) → escolha entre **L**atency e **C**onsistency

```
  ┌────────────────────────────────────────────────────┐
  │                    PACELC                          │
  │                                                    │
  │  Se PARTIÇÃO:     Escolha entre A (Availability)   │
  │                   ou C (Consistency)                │
  │                                                    │
  │  Se NORMAL (Else): Escolha entre L (Latency)       │
  │                    ou C (Consistency)               │
  └────────────────────────────────────────────────────┘
```

| Sistema | P → A ou C | E → L ou C | Classificação |
|---------|------------|------------|---------------|
| **DynamoDB** | PA | EL | PA/EL — Prioriza availability e latência |
| **Cassandra** | PA | EL | PA/EL — Tunable, mas default é eventual |
| **MongoDB** | PC | EC | PC/EC — Prioriza consistência sempre |
| **Google Spanner** | PC | EC | PC/EC — Strong consistency global |
| **Cosmos DB** | PA | EL | PA/EL — Default eventual, tunable |
| **PNUTS (Yahoo)** | PC | EL | PC/EL — Consistência em partição, baixa latência normal |
| **VoltDB** | PC | EC | PC/EC — Transações ACID distribuídas |

---

## Tunable Consistency — O Mundo Real

Na prática, muitos bancos permitem **configurar** o nível de consistência por operação:

### Cassandra — Quorum-Based Consistency

```
  N = número de réplicas (ex: 3)
  W = réplicas que confirmam o write
  R = réplicas lidas no read
  
  Regra: W + R > N → strong consistency
  
  ┌─────────────────────────────────────────────┐
  │  Configurações comuns (N=3):                │
  │                                              │
  │  ONE/ONE:     W=1, R=1 → Fastest, eventual  │
  │  QUORUM/QUORUM: W=2, R=2 → Strong, balanced │
  │  ALL/ONE:     W=3, R=1 → Slow write, fast read │
  │  ONE/ALL:     W=1, R=3 → Fast write, slow read │
  └─────────────────────────────────────────────┘
  
  QUORUM = ⌊N/2⌋ + 1
  
  Exemplo com N=3, QUORUM write (W=2) + QUORUM read (R=2):
  
  WRITE x=5:
  ┌─────┐ ✅ ACK
  │Node1│ x=5
  └─────┘
  ┌─────┐ ✅ ACK   → W=2 confirmaram → WRITE success
  │Node2│ x=5
  └─────┘
  ┌─────┐ (ainda replicando...)
  │Node3│ x=3
  └─────┘
  
  READ x (QUORUM):
  ┌─────┐ → x=5
  │Node1│
  └─────┘
  ┌─────┐ → x=3 (stale)   → R=2, retorna o mais recente: x=5 ✅
  │Node3│
  └─────┘
  
  W(2) + R(2) = 4 > N(3) → STRONG consistency garantida!
```

### DynamoDB — Consistency per Read

```
  Eventually Consistent Read (default):
  → Lê de qualquer réplica (pode estar stale)
  → ~50% mais barato, menor latência
  → Consistência em ~1 segundo
  
  Strongly Consistent Read (opção):
  → Lê do leader/primary
  → Garantia de ver o write mais recente
  → Maior latência, mais caro
  → Pode falhar se leader indisponível
  
  // AWS SDK
  const params = {
    TableName: "Users",
    Key: { userId: "123" },
    ConsistentRead: true  // Strong consistency
  };
  dynamodb.getItem(params);
```

---

## Conflict Resolution em Sistemas AP

Quando um sistema AP aceita writes em nós diferentes durante uma partição, **conflitos** podem surgir:

```
  Partição:
  Node 1: UPDATE user.name = "João"    (Client A)
  Node 2: UPDATE user.name = "Maria"   (Client B)
  
  Partição resolve... quem ganha?
```

| Estratégia | Como Funciona | Usado Por |
|------------|---------------|-----------|
| **Last-Write-Wins (LWW)** | Timestamp mais recente vence | Cassandra, DynamoDB |
| **Vector Clocks** | Detecta conflitos, delega ao client | Riak, Voldemort |
| **CRDTs** | Tipos de dados que convergem automaticamente | Redis (CRDTs), Riak |
| **Application-Level** | App resolve (merge manual) | CouchDB |
| **Operational Transform** | Merge de edições concorrentes | Google Docs |

### Last-Write-Wins (LWW)

```
  Node 1 (t=100): SET name = "João"
  Node 2 (t=102): SET name = "Maria"
  
  Após merge: name = "Maria" (t=102 > t=100)
  
  ⚠️ Problema: client A acha que salvou "João" mas foi sobrescrito
  ⚠️ Clock skew: relógios de nós diferentes podem divergir
```

### Vector Clocks

```
  Cada nó mantém um vetor de versões:
  
  Initial:   {A:0, B:0}
  
  Node A write: {A:1, B:0} → name = "João"
  Node B write: {A:0, B:1} → name = "Maria"
  
  Merge: {A:1, B:0} vs {A:0, B:1}
  → Nenhum domina o outro → CONFLITO DETECTADO
  → Client deve resolver (ex: mostrar ambas opções)
```

---

## CAP na Prática — Cenários Reais

### Cenário 1: E-Commerce — Estoque

```
  Problema: Item com 1 unidade em estoque
  
  ❌ AP (eventual consistency):
  Node 1: stock = 1 → user A compra → stock = 0
  Node 2: stock = 1 → user B compra → stock = 0  (oversell!)
  
  ✅ CP (strong consistency):
  Node 1: stock = 1 → user A compra → stock = 0
  Node 2: stock = 0 → user B tenta → "Out of Stock" ✅
  
  → Para inventário crítico, use CP
```

### Cenário 2: Social Media — Timeline

```
  Problema: User posta tweet, followers querem ler
  
  ✅ AP (eventual consistency):
  User posta → Node 1 aceita
  Follower lê de Node 2 → não vê ainda (delay ~200ms)
  Follower lê de Node 2 → vê agora ✅
  
  → Delay de milissegundos é aceitável para social media
  → Availability e baixa latência são mais importantes
```

### Cenário 3: Banco — Transferência

```
  Problema: Transferir R$100 de conta A para conta B
  
  ❌ AP:
  Node 1: A = $1000, debita $100 → A = $900
  Node 2: A = $1000 (stale), debita $100 → A = $900
  → Debitou 2x, creditou 1x = dinheiro perdido!
  
  ✅ CP:
  Transação distribuída com 2PC ou Saga + idempotência
  → Corretude > disponibilidade para operações financeiras
```

---

## Uso em Big Techs

### Amazon DynamoDB — AP com Opção CP
- **Default:** Eventual consistency (AP)
- **Opção:** Strongly consistent reads (CP por operação)
- **Global Tables:** Multi-region, eventual consistency entre regiões
- **Conflict Resolution:** Last-write-wins com timestamps
- **Use case:** Shopping cart (AP), order placement (strong read)

### Google Spanner — CP com Alta Disponibilidade
- **Modelo:** CP com 99.999% availability (five nines)
- **Segredo:** TrueTime API — relógios atômicos + GPS em cada datacenter
- **Consistência:** External consistency (mais forte que linearizability)
- **Latência:** ~10ms para reads, ~15ms para writes (cross-region)
- **Use case:** Google Ads, Google Play — onde consistência é obrigatória

### Apache Cassandra — AP com Tunable Consistency
- **Default:** Eventual consistency (AP)
- **Tunable:** ONE, QUORUM, ALL, LOCAL_QUORUM, EACH_QUORUM
- **Regra:** W + R > N para strong consistency
- **Use case:** Netflix (user activity), Instagram (feeds), Discord (messages)

### MongoDB — CP por Padrão
- **Write concern:** Majority (espera quorum confirmar)
- **Read concern:** Majority, linearizable, snapshot
- **Failover:** Se primary cai, election de novo primary (~10-30s de downtime)
- **Use case:** Uber (data platform), eBay (product catalog)

### CockroachDB — CP com SQL Distribuído
- **Modelo:** CP com strong consistency (serializable isolation)
- **Protocolo:** Raft consensus entre réplicas
- **SQL compatibility:** PostgreSQL wire protocol
- **Use case:** DoorDash, Netflix (billing)

---

## Perguntas Comuns em Entrevistas

1. **Explique o CAP Theorem.**
   - Em sistema distribuído, é impossível garantir C, A e P simultaneamente. Como P é inevitável, a escolha é entre C e A durante partições.

2. **Por que não podemos ter CA em sistemas distribuídos?**
   - Partições de rede são inevitáveis em sistemas distribuídos. CA só existe em single-node (que não é distribuído).

3. **DynamoDB é CP ou AP?**
   - AP por padrão (eventual consistency reads), mas pode ser CP por operação (strongly consistent reads).

4. **Como o Google Spanner oferece CP com alta availability?**
   - TrueTime API com relógios atômicos garante ordenação global de transações, permitindo consistência sem sacrificar muito a disponibilidade.

5. **Qual a diferença entre CAP e PACELC?**
   - CAP só fala de partições. PACELC estende: sem partição, a escolha é entre Latency e Consistency.

6. **Quando escolher CP vs AP?**
   - CP: dados financeiros, inventário, reservas (corretude > disponibilidade)
   - AP: social feeds, analytics, catálogos (disponibilidade > corretude imediata)

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Modelo** | CP (dados corretos) | AP (sempre disponível) |
| **Reads** | Strongly consistent (lento) | Eventually consistent (rápido) |
| **Writes** | Quorum/All (durável) | One/Local (rápido) |
| **Conflitos** | Prevenir (locks, consensus) | Resolver depois (LWW, CRDTs) |
| **Latência** | Maior (consensus overhead) | Menor (local ops) |
| **Complexidade** | Distributed transactions | Conflict resolution |
| **User Experience** | Pode mostrar erros | Pode mostrar dados stale |

---

## Referências

- [Brewer's CAP Theorem (2000) — Original Keynote](https://www.cs.berkeley.edu/~brewer/cs262b-2004/PODC-keynote.pdf)
- [Gilbert & Lynch — Formal Proof (2002)](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf)
- [Daniel Abadi — PACELC](http://cs-www.cs.yale.edu/homes/dna/papers/abadi-pacelc.pdf)
- [Martin Kleppmann — Designing Data-Intensive Applications](https://dataintensive.net/) — Cap. 9
- [AWS DynamoDB — Consistency Models](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
- [Google Spanner — TrueTime](https://cloud.google.com/spanner/docs/true-time-external-consistency)
- System Design Interview — Alex Xu, Vol. 1
