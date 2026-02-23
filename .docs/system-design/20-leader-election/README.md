# 20. Leader Election

> **Categoria:** Coordenação em Sistemas Distribuídos  
> **Nível:** Avançado — frequente em entrevistas de Big Techs  
> **Complexidade:** Alta

---

## Definição

**Leader Election** é o processo pelo qual um grupo de nós distribuídos **elege um único nó como líder** para coordenar ações que exigem decisão centralizada. O líder é responsável por tarefas como ordenar writes, coordenar replicação, gerenciar locks distribuídos ou tomar decisões de particionamento. Quando o líder falha, um novo líder é eleito automaticamente.

---

## Por Que é Importante?

- **Coordenação de writes** — evitar conflitos (só um nó escreve por vez)
- **Consistência** — líder garante ordem total de operações
- **Replicação** — líder distribui log entries para followers
- **Simplificação** — muitos problemas distribuídos ficam simples com um líder
- **Evitar split-brain** — mecanismo seguro para ter exatamente UM líder
- **Usado em** bancos distribuídos, message brokers, filas, caches, locks

---

## O Problema

```
  SEM líder: quem decide?
  
  ┌────────┐   write(x=1)   ┌────────┐
  │ Client │────────────────▶│ Node A │  x=1
  └────────┘                 └────────┘
                                          CONFLITO!
  ┌────────┐   write(x=2)   ┌────────┐
  │ Client │────────────────▶│ Node B │  x=2
  └────────┘                 └────────┘
  
  Node A diz x=1, Node B diz x=2  → qual é o correto?
  
  ────────────────────────────────────────────────
  
  COM líder: decisão centralizada
  
  ┌────────┐   write(x=1)   ┌────────┐  replicate  ┌────────┐
  │ Client │────────────────▶│ Leader │────────────▶│Follower│
  └────────┘                 │(Node A)│────────────▶│(Node B)│
                             └────────┘  replicate  ├────────┤
  ┌────────┐   write(x=2)        │                  │Follower│
  │ Client │──────────────────────┘                  │(Node C)│
  └────────┘   (também vai pro Leader)               └────────┘
  
  Líder ordena: primeiro x=1, depois x=2
  Todos concordam: x=2 é o valor final
```

---

## Propriedades Desejáveis

```
  Leader Election deve garantir:
  
  ┌───────────────────────────────────────────────────────────────┐
  │                                                               │
  │  1. SAFETY (segurança):                                      │
  │     No máximo 1 líder a qualquer momento                     │
  │     → Evitar split-brain (2 líderes simultâneos)             │
  │                                                               │
  │  2. LIVENESS (vivacidade):                                   │
  │     Eventualmente um líder será eleito                        │
  │     → Sistema não fica parado para sempre                    │
  │                                                               │
  │  3. FAIRNESS (justiça) — opcional:                           │
  │     Qualquer nó elegível pode se tornar líder                │
  │                                                               │
  │  4. STABILITY (estabilidade):                                │
  │     Líder saudável permanece líder                           │
  │     → Evitar "election storms" (re-eleições desnecessárias)  │
  │                                                               │
  └───────────────────────────────────────────────────────────────┘
```

---

## Algoritmos de Leader Election

### 1. Bully Algorithm

```
  Regra: nó com MAIOR ID vence a eleição
  
  Cenário: Node 3 (líder atual) falha
  
  Nós: [1, 2, 3(dead), 4, 5]
  
  Passo 1: Node 2 detecta falha do líder (timeout)
           Node 2 inicia eleição → envia ELECTION para nós com ID maior
  
  ┌────┐  ELECTION  ┌────┐  ELECTION  ┌────┐  ELECTION  ┌────┐
  │ N2 │──────────▶│ N3 │──────────▶│ N4 │──────────▶│ N5 │
  └────┘           │DEAD│           └──┬─┘           └──┬─┘
                   └────┘              │                │
                                       ▼                ▼
  Passo 2: N4 e N5 respondem OK     OK para N2        OK para N2
           N4 também inicia eleição → envia ELECTION para N5
  
  Passo 3: N5 responde OK para N4
           N5 não tem ninguém maior → N5 se declara LÍDER
  
  Passo 4: N5 envia COORDINATOR para todos
  
  ┌────┐  COORD  ┌────┐  COORD  ┌────┐  COORD  ┌────┐
  │ N5 │────────▶│ N4 │────────▶│ N2 │────────▶│ N1 │
  │LIDER│        └────┘         └────┘         └────┘
  └────┘
  
  Complexidade: O(n²) mensagens no pior caso
  Problema: nó com maior ID pode ser o mais fraco/lento
```

### 2. Ring Election

```
  Nós organizados em anel lógico:
  
       ┌────┐
       │ N1 │
     ╱ └────┘ ╲
  ┌────┐       ┌────┐
  │ N5 │       │ N2 │
  └────┘       └────┘
     ╲ ┌────┐ ╱
       │ N4 │
     ╱ └────┘ ╲
  ┌────┐
  │ N3 │
  └────┘
  
  N2 detecta falha do líder:
  1. N2 envia ELECTION(2) para N3
  2. N3 adiciona seu ID: ELECTION(2,3) → envia para N4
  3. N4: ELECTION(2,3,4) → N5
  4. N5: ELECTION(2,3,4,5) → N1
  5. N1: ELECTION(2,3,4,5,1) → N2
  6. N2 recebe de volta → maior ID = 5 → N5 é líder
  7. N2 envia COORDINATOR(5) pelo anel
  
  Complexidade: O(n) mensagens
```

### 3. Raft Leader Election

```
  ┌─────────────────────────────────────────────────────────┐
  │                RAFT LEADER ELECTION                      │
  │                                                         │
  │  Estados: FOLLOWER → CANDIDATE → LEADER                │
  │                                                         │
  │  ╔═══════════╗  timeout   ╔═══════════╗  majority   ╔═══════╗
  │  ║ FOLLOWER  ║──────────▶║ CANDIDATE ║────────────▶║ LEADER║
  │  ╚═══════════╝           ╚═══════════╝             ╚═══════╝
  │       ▲                       │ ▲                      │
  │       │    discovers leader   │ │ election timeout     │
  │       │    or higher term     │ │ (no majority)        │
  │       └───────────────────────┘ └──────────────────────│
  │                                                  heartbeat
  │                                                  to followers
  │                                                         │
  │  FLUXO:                                                 │
  │  1. Follower não recebe heartbeat por election_timeout  │
  │  2. Incrementa currentTerm, vira Candidate              │
  │  3. Vota em si mesmo                                    │
  │  4. Envia RequestVote RPC para todos os outros nós      │
  │  5. Se recebe maioria dos votos → vira Leader           │
  │  6. Se outro líder aparece com term maior → volta a     │
  │     Follower                                           │
  │  7. Se timeout sem maioria → nova eleição (term+1)      │
  │                                                         │
  │  RANDOMIZED TIMEOUT:                                    │
  │  election_timeout = random(150ms, 300ms)                │
  │  → Evita múltiplos candidates simultâneos               │
  └─────────────────────────────────────────────────────────┘
```

#### Exemplo de Eleição Raft

```
  Cluster: [A, B, C, D, E]  — Líder atual: A (term=3)
  
  t=0:    A (Leader, term=3) enviando heartbeats normalmente
  t=100:  A FALHA (crash)
  
  t=250:  C atinge election_timeout (random: 250ms)
          C incrementa term → term=4
          C vira Candidate
          C vota em si mesmo (1 voto)
          C envia RequestVote(term=4) para B, D, E
  
  t=260:  B recebe RequestVote(term=4)
          B não votou neste term → vota em C → responde YES
          D recebe RequestVote → vota em C → YES
          
  t=270:  C tem 3 votos (C, B, D) → MAIORIA (3/5)
          C se torna LEADER (term=4)
          C envia AppendEntries(heartbeat) para todos
  
  t=280:  E recebe heartbeat de C (term=4)
          E reconhece C como novo Leader
  
  Resultado: C é o novo líder com term=4
  Total de downtime: ~170ms (de A crash até C virar leader)
```

### 4. ZAB (ZooKeeper Atomic Broadcast)

```
  Similar a Raft, usado pelo ZooKeeper:
  
  Fases:
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │   Phase 0    │───▶│   Phase 1    │───▶│   Phase 2    │
  │  Discovery   │    │  Sync        │    │  Broadcast   │
  │              │    │              │    │              │
  │ Encontrar    │    │ Novo líder   │    │ Líder aceita │
  │ líder        │    │ sincroniza   │    │ e replica    │
  │ prospectivo  │    │ followers    │    │ transações   │
  └──────────────┘    └──────────────┘    └──────────────┘
  
  Leader election em ZAB:
  → Nó com MAIOR zxid (transaction ID) é preferido
  → Se empate, nó com MAIOR ID (myid) vence
  → Garante que líder tem os dados mais atualizados
```

---

## Fencing e Split-Brain Prevention

### O Problema do Split-Brain

```
  ┌──────────────────────────────────────────────────────────┐
  │ SPLIT-BRAIN: 2 líderes simultâneos                       │
  │                                                          │
  │  Leader A (term=3)            Leader B (term=3)          │
  │  "Eu sou o líder!"           "Eu sou o líder!"          │
  │       │                           │                      │
  │       ▼                           ▼                      │
  │  write(x=1)                  write(x=2)                 │
  │  replicate para F1, F2       replicate para F3, F4      │
  │                                                          │
  │  DADOS INCONSISTENTES! x=1 vs x=2                       │
  │                                                          │
  │  Causas:                                                 │
  │  → Network partition (nós não se veem)                   │
  │  → GC pause longa (líder parece morto, mas volta)       │
  │  → Clock skew (timeouts inconsistentes)                  │
  └──────────────────────────────────────────────────────────┘
```

### Fencing Tokens

```
  Solução: cada líder recebe um FENCING TOKEN (número monotonicamente crescente)
  
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  Lock Service emite tokens:                              │
  │  Leader A recebe token=33                                │
  │  (A falha, B eleito)                                     │
  │  Leader B recebe token=34                                │
  │                                                          │
  │  Storage verifica token ANTES de aceitar write:          │
  │                                                          │
  │  A (token=33): write(x=1) ──▶ Storage                   │
  │  B (token=34): write(x=2) ──▶ Storage                   │
  │                                                          │
  │  Se A envia write após B (network delay):                │
  │  A (token=33): write(x=1) ──▶ Storage rejeita!          │
  │  (Storage já viu token=34, que é maior)                  │
  │                                                          │
  │  → Garante que write de líder stale é rejeitado          │
  └──────────────────────────────────────────────────────────┘
```

### Epoch/Term Numbers

```
  Raft/ZAB usam term/epoch para prevenir split-brain:
  
  Regra: mensagem com term menor que o currentTerm é IGNORADA
  
  Cenário:
  1. Leader A (term=3) sofre GC pause de 10s
  2. Followers acham que A morreu → elegem B (term=4)
  3. A "acorda" da GC pause
  4. A tenta enviar heartbeat (term=3)
  5. Followers REJEITAM (term=3 < currentTerm=4)
  6. A recebe resposta com term=4 → A percebe que é stale
  7. A volta a ser FOLLOWER
  
  → Term/Epoch é o fencing token do Raft/ZAB
```

---

## Implementação com ZooKeeper

### Leader Election com Ephemeral Znodes

```java
public class LeaderElection implements Watcher {
    
    private final ZooKeeper zk;
    private final String electionPath = "/election";
    private String currentZnodeName;
    private boolean isLeader = false;
    
    public LeaderElection(ZooKeeper zk) throws Exception {
        this.zk = zk;
        // Criar znode pai se não existir
        if (zk.exists(electionPath, false) == null) {
            zk.create(electionPath, new byte[0], 
                ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }
    }
    
    public void volunteerForLeadership() throws Exception {
        // Criar ephemeral sequential znode
        String znodeFullPath = zk.create(
            electionPath + "/c_",   // prefixo
            new byte[0],
            ZooDefs.Ids.OPEN_ACL_UNSAFE,
            CreateMode.EPHEMERAL_SEQUENTIAL  // ← chave: sequential + ephemeral
        );
        // znodeFullPath = "/election/c_0000000001"
        this.currentZnodeName = znodeFullPath.replace(electionPath + "/", "");
        System.out.println("Created znode: " + currentZnodeName);
    }
    
    public void electLeader() throws Exception {
        List<String> children = zk.getChildren(electionPath, false);
        Collections.sort(children); // ordem numérica
        
        String smallestChild = children.get(0);
        
        if (smallestChild.equals(currentZnodeName)) {
            // EU sou o menor → EU sou o líder
            isLeader = true;
            System.out.println("I am the LEADER!");
            onElectedAsLeader();
        } else {
            // Não sou o líder → watch no nó anterior (herd effect prevention)
            int myIndex = children.indexOf(currentZnodeName);
            String predecessorZnode = children.get(myIndex - 1);
            
            System.out.println("I am a follower. Watching: " + predecessorZnode);
            
            // Watch no predecessor — se ele morrer, re-avaliar
            Stat predecessorStat = zk.exists(
                electionPath + "/" + predecessorZnode, this);
            
            if (predecessorStat == null) {
                // Predecessor já morreu — re-avaliar
                electLeader();
            }
        }
    }
    
    @Override
    public void process(WatchedEvent event) {
        if (event.getType() == Event.EventType.NodeDeleted) {
            // Predecessor morreu — tentar virar líder
            try {
                electLeader();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
    
    private void onElectedAsLeader() {
        // Lógica do líder
        System.out.println("Starting leader duties...");
    }
}
```

### Como Funciona

```
  ZooKeeper Leader Election:
  
  1. Cada nó cria znode EPHEMERAL SEQUENTIAL:
     /election/c_0000000001  ← Node A
     /election/c_0000000002  ← Node B
     /election/c_0000000003  ← Node C
  
  2. Nó com MENOR sequence number = LÍDER
     Node A (c_0000000001) é o líder
  
  3. Cada nó watch no PREDECESSOR (não no líder):
     Node C watches c_0000000002 (Node B)
     Node B watches c_0000000001 (Node A, líder)
  
  4. Se Node A morre → znode ephemeral deletado automaticamente:
     /election/c_0000000002  ← Node B  → NOVO LÍDER!
     /election/c_0000000003  ← Node C  → watches Node B
  
  Por que ephemeral?
  → Se o processo morre, ZK automaticamente deleta o znode
  → Não precisa de deregistration manual
  
  Por que watch no predecessor (não no líder)?
  → Evita "herd effect": se 1000 nós todos assistem o líder,
    quando líder morre, 1000 nós recebem notificação simultânea
  → Com predecessor watch, só 1 nó recebe notificação
```

---

## Leader Election com etcd (Go)

```go
import (
    "context"
    "fmt"
    "go.etcd.io/etcd/client/v3"
    "go.etcd.io/etcd/client/v3/concurrency"
)

func main() {
    cli, _ := clientv3.New(clientv3.Config{
        Endpoints: []string{"localhost:2379"},
    })
    defer cli.Close()
    
    // Criar session (com lease TTL = heartbeat)
    session, _ := concurrency.NewSession(cli, concurrency.WithTTL(10))
    defer session.Close()
    
    // Criar election
    election := concurrency.NewElection(session, "/leader-election/")
    
    // Candidatar-se a líder (bloqueia até ser eleito)
    ctx := context.Background()
    if err := election.Campaign(ctx, "node-1"); err != nil {
        fmt.Println("Failed to campaign:", err)
        return
    }
    
    fmt.Println("I am the leader!")
    
    // Fazer trabalho de líder...
    doLeaderWork()
    
    // Resignar (abrir mão da liderança)
    election.Resign(ctx)
}
```

---

## Uso em Big Techs

### Google — Chubby / Paxos
- **Chubby**: lock service distribuído usado internamente
- Paxos para leader election em muitos sistemas (Spanner, Bigtable)
- GFS Master: líder único coordena metadata (com failover automático)

### Amazon — Leader Election em DynamoDB
- DynamoDB usa leader election para coordenar partições
- Conditional writes como mecanismo de "lock" (compare-and-swap)
- Internal: Paxos-based leader election

### Netflix — Leader Election via Curator/ZooKeeper
- Apache Curator framework (criado pelo Netflix) para ZooKeeper
- Leader election recipe: `LeaderSelector` / `LeaderLatch`
- Usado para coordenar batch jobs e maintenance tasks

### LinkedIn — Apache Kafka (Raft/ZAB-like)
- Kafka Controller: líder que gerencia partitions e replicas
- Partition Leader: cada partition tem um líder para reads/writes
- KRaft mode (Kafka Raft): substitui ZooKeeper para metadata

### Uber — Cadence/Temporal
- Leader election para workflow executors
- Sharding + leader per shard para escalabilidade
- Ring-based membership com consistent hashing

---

## Perguntas Comuns em Entrevistas

1. **Por que precisamos de Leader Election?**
   - Para coordenar operações que precisam de decisão centralizada (ordenar writes, replicação, locks).

2. **O que é Split-Brain e como prevenir?**
   - 2 líderes simultâneos causando inconsistência. Prevenção: fencing tokens, term/epoch numbers, majority quorum.

3. **Bully Algorithm vs Raft?**
   - Bully: simples, maior ID vence, O(n²) mensagens. Raft: sofisticado, logs replicados, usado em produção.

4. **O que são Fencing Tokens?**
   - Números monotonicamente crescentes emitidos a cada nova eleição. Storage rejeita writes com tokens antigos.

5. **Como ZooKeeper implementa Leader Election?**
   - Ephemeral sequential znodes: menor sequence = líder. Watch no predecessor para evitar herd effect.

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Algoritmo** | Bully (simples, O(n²)) | Raft (robusto, O(n)) |
| **Infraestrutura** | ZooKeeper/etcd (extra) | Built-in (Kafka KRaft) |
| **Failover speed** | Rápido (election timeout curto) | Estável (timeout longo) |
| **Quorum** | Maioria estrita (seguro) | Flexível (roda com menos nós) |
| **Leader duties** | Strong leader (tudo via líder) | Leaderless (peer-to-peer) |
| **Split-brain** | Fencing tokens (extra) | Term/epoch (built-in) |
| **Election trigger** | Timeout-based (reativo) | Lease-based (proativo) |

---

## Referências

- [Raft Consensus Algorithm — Visualization](https://raft.github.io/)
- [Ongaro & Ousterhout — In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)
- [Apache ZooKeeper — Recipes and Solutions](https://zookeeper.apache.org/doc/current/recipes.html)
- [Martin Kleppmann — Designing Data-Intensive Applications, Chapter 8](https://dataintensive.net/)
- [Apache Curator — Leader Election](https://curator.apache.org/docs/recipes-leader-election)
- [etcd — Concurrency Package](https://pkg.go.dev/go.etcd.io/etcd/client/v3/concurrency)
