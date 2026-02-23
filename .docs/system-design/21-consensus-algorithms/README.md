# 21. Consensus Algorithms (Raft / Paxos)

> **Categoria:** Fundamentos de Sistemas Distribuídos  
> **Nível:** Avançado — diferencial em entrevistas de Big Techs  
> **Complexidade:** Alta

---

## Definição

**Consensus** (consenso) é o problema fundamental de fazer múltiplos nós em um sistema distribuído **concordarem sobre um único valor**, mesmo na presença de falhas. **Algoritmos de consenso** garantem que todos os nós não-faltosos concordam com o mesmo resultado (safety) e eventualmente tomam uma decisão (liveness). Os dois algoritmos mais influentes são **Paxos** (1989, Lamport) e **Raft** (2014, Ongaro & Ousterhout).

---

## Por Que é Importante?

- **Base de TUDO em sistemas distribuídos** — replicação, leader election, distributed locks
- **Garante consistência forte** (linearizability) em ambientes com falhas
- **Usado em** etcd (Raft), ZooKeeper (ZAB), Google Spanner (Paxos), CockroachDB (Raft)
- **Diferencial em entrevistas** — demonstra conhecimento profundo de distribuídos
- **Resolve** o problema de replicação de state machines (SMR)

---

## O Problema do Consenso

```
  5 servidores precisam concordar sobre a mesma sequência de operações:
  
  Cliente envia: SET x=5, SET y=10, SET x=7
  
  Sem consenso:                  Com consenso:
  ┌─────┐ x=5, y=10, x=7       ┌─────┐ x=5, y=10, x=7  ✓
  │  A  │                       │  A  │
  ├─────┤ x=5, x=7, y=10       ├─────┤ x=5, y=10, x=7  ✓
  │  B  │                       │  B  │
  ├─────┤ y=10, x=5, x=7       ├─────┤ x=5, y=10, x=7  ✓
  │  C  │                       │  C  │
  ├─────┤ x=7, x=5, y=10       ├─────┤ x=5, y=10, x=7  ✓
  │  D  │                       │  D  │
  ├─────┤ (perdeu msgs)         ├─────┤ x=5, y=10, x=7  ✓
  │  E  │                       │  E  │
  └─────┘                       └─────┘
  
  ❌ Cada nó com ordem diferente   ✅ Todos concordam na mesma ordem
  → Estado inconsistente           → Estado consistente
```

### Propriedades Fundamentais (FLP)

```
  Resultado de Fischer-Lynch-Paterson (FLP, 1985):
  
  Em um sistema assíncrono com pelo menos 1 falha:
  → É IMPOSSÍVEL garantir consenso determinístico
  
  Na prática, contornamos com:
  → Timeouts (detectores de falha imperfeitos)
  → Randomização (Raft: randomized election timeout)
  → Partial synchrony assumptions
  
  Propriedades que buscamos:
  
  ┌───────────────────────────────────────────────────┐
  │ AGREEMENT:    Todos os nós corretos decidem       │
  │               o mesmo valor                       │
  │                                                   │
  │ VALIDITY:     O valor decidido foi proposto      │
  │               por algum nó (não inventado)        │
  │                                                   │
  │ TERMINATION:  Todos os nós corretos              │
  │               eventualmente decidem               │
  │                                                   │
  │ INTEGRITY:    Cada nó decide no máximo uma vez   │
  └───────────────────────────────────────────────────┘
```

---

## Replicated State Machine (RSM)

```
  Consensus é usado para implementar RSM:
  
  ┌─────────────────────────────────────────────────────────┐
  │              Replicated State Machine                    │
  │                                                         │
  │  Client ──request──▶ ┌────────────────────┐             │
  │                      │   Consensus Module │             │
  │                      │  (Raft / Paxos)    │             │
  │                      └────────┬───────────┘             │
  │                               │                         │
  │              ┌────────────────┼────────────────┐        │
  │              ▼                ▼                ▼        │
  │     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │
  │     │   Server 1   │ │   Server 2   │ │   Server 3   │ │
  │     │              │ │              │ │              │ │
  │     │ Log:         │ │ Log:         │ │ Log:         │ │
  │     │ [1] SET x=5  │ │ [1] SET x=5  │ │ [1] SET x=5  │ │
  │     │ [2] SET y=10 │ │ [2] SET y=10 │ │ [2] SET y=10 │ │
  │     │ [3] SET x=7  │ │ [3] SET x=7  │ │ [3] SET x=7  │ │
  │     │              │ │              │ │              │ │
  │     │ State:       │ │ State:       │ │ State:       │ │
  │     │ x=7, y=10   │ │ x=7, y=10   │ │ x=7, y=10   │ │
  │     └──────────────┘ └──────────────┘ └──────────────┘ │
  │                                                         │
  │  Se todos aplicam o MESMO log na MESMA ordem:          │
  │  → Todos chegam ao MESMO estado                        │
  └─────────────────────────────────────────────────────────┘
```

---

## Raft — In Detail

### Visão Geral

```
  Raft divide o consenso em 3 sub-problemas:
  
  1. LEADER ELECTION  — eleger um líder
  2. LOG REPLICATION  — líder replica entries para followers
  3. SAFETY          — garantir propriedades de consistência
  
  Roles:
  ┌──────────┐      ┌───────────┐      ┌────────┐
  │ FOLLOWER │─────▶│ CANDIDATE │─────▶│ LEADER │
  │          │      │           │      │        │
  │ Passivo  │      │ Solicita  │      │ Ativo  │
  │ Responde │      │ votos     │      │ Envia  │
  │ a RPCs   │      │           │      │ entries│
  └──────────┘      └───────────┘      └────────┘
  
  Terms (mandatos):
  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
  │ Term 1 │ │ Term 2 │ │ Term 3 │ │ Term 4 │
  │ Leader:│ │ Leader:│ │(no     │ │ Leader:│
  │   A    │ │   B    │ │leader) │ │   C    │
  └────────┘ └────────┘ └────────┘ └────────┘
  
  Cada term começa com uma eleição
  No máximo 1 líder por term
  Term funciona como relógio lógico
```

### Log Replication

```
  ┌──────────────────────────────────────────────────────────┐
  │                  RAFT LOG REPLICATION                      │
  │                                                          │
  │  Client: SET x=5                                         │
  │     │                                                    │
  │     ▼                                                    │
  │  ┌────────────────────────────────────────────┐          │
  │  │ LEADER (term=3)                            │          │
  │  │                                            │          │
  │  │ 1. Append to local log:                    │          │
  │  │    Log: [..., {term:3, cmd:"SET x=5"}]     │          │
  │  │                                            │          │
  │  │ 2. Send AppendEntries RPC to all followers │          │
  │  └──────────┬──────────────┬──────────────────┘          │
  │             │              │                              │
  │             ▼              ▼                              │
  │  ┌────────────────┐ ┌────────────────┐                   │
  │  │ FOLLOWER B     │ │ FOLLOWER C     │   Follower D     │
  │  │ Append to log  │ │ Append to log  │   (offline)      │
  │  │ Respond: OK    │ │ Respond: OK    │                   │
  │  └────────┬───────┘ └────────┬───────┘                   │
  │           │                  │                            │
  │           ▼                  ▼                            │
  │  ┌──────────────────────────────────────┐                │
  │  │ LEADER recebe majority (B + C + self)│                │
  │  │                                      │                │
  │  │ 3. COMMIT: entry é committed          │                │
  │  │    (commitIndex avança)              │                │
  │  │                                      │                │
  │  │ 4. Apply to state machine            │                │
  │  │    x = 5                             │                │
  │  │                                      │                │
  │  │ 5. Respond to client: SUCCESS        │                │
  │  │                                      │                │
  │  │ 6. Next AppendEntries informa        │                │
  │  │    followers do novo commitIndex     │                │
  │  │    → Followers também aplicam        │                │
  │  └──────────────────────────────────────┘                │
  │                                                          │
  │  Follower D (quando voltar online):                      │
  │  → Leader detecta log inconsistency                      │
  │  → Envia entries faltantes via AppendEntries             │
  │  → D "catch up" com o log do Leader                     │
  └──────────────────────────────────────────────────────────┘
```

### Raft Safety Properties

```
  ┌─────────────────────────────────────────────────────────┐
  │ RAFT SAFETY GUARANTEES                                   │
  │                                                         │
  │ 1. Election Safety:                                     │
  │    No máximo 1 líder por term                           │
  │    (cada nó só vota 1 vez por term)                     │
  │                                                         │
  │ 2. Leader Append-Only:                                  │
  │    Líder NUNCA sobrescreve ou deleta entries do log      │
  │    (só appends)                                         │
  │                                                         │
  │ 3. Log Matching:                                        │
  │    Se 2 logs contêm entry com mesmo index e term,       │
  │    então os logs são idênticos até aquele index          │
  │                                                         │
  │ 4. Leader Completeness:                                 │
  │    Se um entry é committed no term T,                   │
  │    estará presente no log de TODOS os leaders de        │
  │    terms > T                                            │
  │                                                         │
  │ 5. State Machine Safety:                                │
  │    Se um nó aplicou log entry no index I,               │
  │    nenhum outro nó aplicará entry diferente no index I  │
  │                                                         │
  │ Implicação: se majority confirmou → é permanente        │
  │ → Dados committed NUNCA são perdidos (se majority vive) │
  └─────────────────────────────────────────────────────────┘
```

### Log Consistency Repair

```
  Quando um novo líder é eleito, logs podem estar inconsistentes:
  
  Leader (term 8):
  Log: [1:1] [2:1] [3:2] [4:3] [5:3] [6:3] [7:4] [8:6] [9:6] [10:8]
  
  Follower A:
  Log: [1:1] [2:1] [3:2] [4:3] [5:3] [6:3] [7:4]          ← missing entries
  
  Follower B:
  Log: [1:1] [2:1] [3:2] [4:3] [5:3] [6:3] [7:4] [8:5] [9:5]  ← conflicting entries
  
  Leader repair process para Follower B:
  1. Leader envia AppendEntries(prevLogIndex=9, prevLogTerm=6)
  2. B: meu entry 9 tem term=5, não 6 → REJECT
  3. Leader decrementa: AppendEntries(prevLogIndex=8, prevLogTerm=6)
  4. B: meu entry 8 tem term=5, não 6 → REJECT
  5. Leader decrementa: AppendEntries(prevLogIndex=7, prevLogTerm=4)
  6. B: meu entry 7 tem term=4 → MATCH! → OK
  7. Leader envia entries 8, 9, 10 → B sobrescreve suas entries 8, 9
  
  Resultado: B agora tem o mesmo log que o Leader
```

---

## Paxos — In Detail

### Single-Decree Paxos (Basic Paxos)

```
  Consenso sobre UM ÚNICO VALOR:
  
  Roles:
  • Proposer — propõe valores
  • Acceptor — vota/aceita propostas
  • Learner  — aprende o valor decidido
  (um nó pode ter múltiplos roles)
  
  ┌─────────────────────────────────────────────────────────┐
  │           PAXOS — Two-Phase Protocol                     │
  │                                                         │
  │  PHASE 1: PREPARE                                       │
  │                                                         │
  │  Proposer ──Prepare(n)──▶ Acceptors (majority)          │
  │           ◀──Promise(n)── Acceptors                     │
  │                                                         │
  │  "Prepare(n)": "Vou propor com número n, vocês          │
  │                 prometem não aceitar nada < n?"          │
  │                                                         │
  │  "Promise(n)": "Prometo não aceitar propostas com       │
  │                 número < n. Aqui está o último valor     │
  │                 que aceitei (se houver)."                │
  │                                                         │
  │  ─────────────────────────────────────────────────      │
  │                                                         │
  │  PHASE 2: ACCEPT                                        │
  │                                                         │
  │  Proposer ──Accept(n, v)──▶ Acceptors (majority)        │
  │           ◀──Accepted(n, v)── Acceptors                 │
  │                                                         │
  │  "Accept(n, v)": "Aceitem o valor v com número n"       │
  │                                                         │
  │  Se Acceptor não prometeu para n' > n:                  │
  │  → Aceita e responde Accepted                           │
  │                                                         │
  │  Se majority aceita → VALOR DECIDIDO!                   │
  │  → Learners são notificados                             │
  └─────────────────────────────────────────────────────────┘
```

### Exemplo Concreto — Basic Paxos

```
  Acceptors: [A1, A2, A3]  (majority = 2)
  
  Proposer P1 quer propor v="alpha"
  Proposer P2 quer propor v="beta"
  
  ──── Timeline ────────────────────────────────
  
  P1: Prepare(1) → A1, A2, A3
  
  A1: Promise(1, null)  → P1   (nunca aceitou nada)
  A2: Promise(1, null)  → P1
  A3: (mensagem perdida)
  
  P1 tem majority (A1, A2) → Phase 2
  P1: Accept(1, "alpha") → A1, A2, A3
  
  A1: Accepted(1, "alpha") → P1
  
  *** Nesse momento, P2 entra ***
  
  P2: Prepare(2) → A1, A2, A3   (n=2 > n=1)
  
  A2: Promise(2, accepted=<1,"alpha">)  → P2
  ^^^^^ A2 já aceitou "alpha" com n=1
  
  A3: Promise(2, null) → P2
  
  P2 tem majority (A2, A3)
  P2 VÊ que A2 já aceitou "alpha"
  → P2 DEVE propor "alpha" (não "beta"!)
  
  P2: Accept(2, "alpha") → A1, A2, A3
  
  Resultado: TODOS concordam com "alpha"
  → Paxos garante que uma vez que majority aceita,
    o valor é permanente
```

### Multi-Paxos

```
  Basic Paxos: consenso sobre 1 valor (2 fases por valor = caro!)
  Multi-Paxos: otimização para sequência de valores
  
  ┌─────────────────────────────────────────────────────────┐
  │ MULTI-PAXOS                                             │
  │                                                         │
  │ Otimização: um LÍDER estável pula Phase 1               │
  │                                                         │
  │ Basic Paxos (cada valor):                               │
  │ Prepare → Promise → Accept → Accepted (4 mensagens)    │
  │                                                         │
  │ Multi-Paxos (com líder estável):                        │
  │ Accept → Accepted (2 mensagens!)                        │
  │                                                         │
  │ Leader faz Phase 1 UMA VEZ:                             │
  │ "Para TODOS os slots futuros, eu sou o proposer"        │
  │                                                         │
  │ Depois, para cada novo valor:                           │
  │ Leader: Accept(slot, value) → acceptors                 │
  │ Acceptors: Accepted → leader                            │
  │ Majority → committed!                                   │
  │                                                         │
  │ Multi-Paxos com líder estável ≈ Raft                    │
  │ (Raft é basicamente Multi-Paxos "understandable")       │
  └─────────────────────────────────────────────────────────┘
```

---

## Comparativo: Raft vs Paxos

| Aspecto | Raft | Paxos |
|---------|------|-------|
| **Publicação** | 2014 (Ongaro) | 1989 (Lamport) |
| **Complexidade** | Projetado para ser entendível | Notoriamente difícil de entender |
| **Leader** | Obrigatório (strong leader) | Opcional (Multi-Paxos usa) |
| **Log** | Log contíguo (sem lacunas) | Pode ter lacunas no log |
| **Eleição** | Randomized timeout | Prepare/Promise (conflito-prone) |
| **Implementações** | etcd, CockroachDB, TiKV | Spanner, Chubby, Megastore |
| **Provas** | Formal (TLA+) | Formal (original paper) |
| **Variantes** | Poucas (vanilla Raft) | Muitas (Multi, Fast, Cheap, Egalitarian) |
| **Ensino** | Fácil (projetado para isso) | Difícil (múltiplos papers para entender) |
| **Latência** | 1 RTT (líder estável) | 2 RTTs (basic) / 1 RTT (multi com líder) |

---

## Quorum e Fault Tolerance

```
  ┌─────────────────────────────────────────────────────┐
  │ QUORUM = Majority = ⌊N/2⌋ + 1                      │
  │                                                     │
  │ Cluster de 5 nós:                                   │
  │ Quorum = 3 (qualquer 3 de 5)                        │
  │ Tolera: 2 falhas simultâneas                        │
  │                                                     │
  │ Por que majority?                                   │
  │ → Qualquer 2 majorities compartilham pelo menos     │
  │   1 nó em comum                                     │
  │ → Isso garante que informação committed é            │
  │   preservada entre eleições                         │
  │                                                     │
  │ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐                     │
  │ │ A │ │ B │ │ C │ │ D │ │ E │                     │
  │ └───┘ └───┘ └───┘ └───┘ └───┘                     │
  │                                                     │
  │ Majority 1: {A, B, C}                              │
  │ Majority 2: {C, D, E}                              │
  │              ^^^^                                   │
  │              C está em ambos!                        │
  │              → C garante continuidade de informação  │
  │                                                     │
  │ ┌─────────────────────────────────────────────┐     │
  │ │  N (total) │ Quorum │ Max falhas toleradas  │     │
  │ ├────────────┼────────┼───────────────────────┤     │
  │ │     3      │   2    │          1            │     │
  │ │     5      │   3    │          2            │     │
  │ │     7      │   4    │          3            │     │
  │ │     9      │   5    │          4            │     │
  │ └─────────────────────────────────────────────┘     │
  │                                                     │
  │ Por que usar número ímpar?                          │
  │ → 4 nós: quorum=3, tolera 1 falha                  │
  │ → 5 nós: quorum=3, tolera 2 falhas                 │
  │ → 5 nós é melhor que 4 com mesmo quorum!            │
  └─────────────────────────────────────────────────────┘
```

---

## Implementação Simplificada — Raft (Java)

```java
public class RaftNode {
    
    enum State { FOLLOWER, CANDIDATE, LEADER }
    
    private State state = State.FOLLOWER;
    private int currentTerm = 0;
    private Integer votedFor = null;
    private List<LogEntry> log = new ArrayList<>();
    private int commitIndex = 0;
    
    // Leader state
    private int[] nextIndex;   // para cada follower
    private int[] matchIndex;  // para cada follower
    
    private final int nodeId;
    private final int clusterSize;
    private final Random random = new Random();
    
    // Timer
    private long electionTimeout;
    private long lastHeartbeat;
    
    public RaftNode(int nodeId, int clusterSize) {
        this.nodeId = nodeId;
        this.clusterSize = clusterSize;
        resetElectionTimeout();
    }
    
    private void resetElectionTimeout() {
        // Random timeout: 150-300ms (evita split vote)
        this.electionTimeout = 150 + random.nextInt(150);
        this.lastHeartbeat = System.currentTimeMillis();
    }
    
    // ==================== LEADER ELECTION ====================
    
    public void tick() {
        if (state != State.LEADER && electionTimeoutElapsed()) {
            startElection();
        }
    }
    
    private boolean electionTimeoutElapsed() {
        return System.currentTimeMillis() - lastHeartbeat > electionTimeout;
    }
    
    private void startElection() {
        currentTerm++;
        state = State.CANDIDATE;
        votedFor = nodeId;
        int votesReceived = 1; // voto próprio
        
        resetElectionTimeout();
        
        // Enviar RequestVote para todos os outros nós
        for (int peerId = 0; peerId < clusterSize; peerId++) {
            if (peerId == nodeId) continue;
            
            RequestVoteResponse response = sendRequestVote(peerId, 
                new RequestVoteRequest(currentTerm, nodeId, 
                    getLastLogIndex(), getLastLogTerm()));
            
            if (response != null && response.voteGranted) {
                votesReceived++;
            }
            
            if (response != null && response.term > currentTerm) {
                // Descobriu term maior → volta a follower
                currentTerm = response.term;
                state = State.FOLLOWER;
                votedFor = null;
                return;
            }
        }
        
        if (votesReceived > clusterSize / 2) {
            // Maioria! → virar líder
            becomeLeader();
        }
    }
    
    private void becomeLeader() {
        state = State.LEADER;
        nextIndex = new int[clusterSize];
        matchIndex = new int[clusterSize];
        Arrays.fill(nextIndex, log.size() + 1);
        
        System.out.println("Node " + nodeId + " is now LEADER (term=" + currentTerm + ")");
        
        // Enviar heartbeats imediatamente
        sendHeartbeats();
    }
    
    // ==================== REQUEST VOTE HANDLER ====================
    
    public RequestVoteResponse handleRequestVote(RequestVoteRequest request) {
        if (request.term < currentTerm) {
            return new RequestVoteResponse(currentTerm, false);
        }
        
        if (request.term > currentTerm) {
            currentTerm = request.term;
            state = State.FOLLOWER;
            votedFor = null;
        }
        
        // Votar se: não votou neste term E candidato tem log "up-to-date"
        boolean logOk = request.lastLogTerm > getLastLogTerm() ||
            (request.lastLogTerm == getLastLogTerm() && 
             request.lastLogIndex >= getLastLogIndex());
        
        if ((votedFor == null || votedFor == request.candidateId) && logOk) {
            votedFor = request.candidateId;
            resetElectionTimeout();
            return new RequestVoteResponse(currentTerm, true);
        }
        
        return new RequestVoteResponse(currentTerm, false);
    }
    
    // ==================== LOG REPLICATION ====================
    
    public boolean appendEntry(String command) {
        if (state != State.LEADER) {
            return false; // só o líder aceita writes
        }
        
        // 1. Append to local log
        log.add(new LogEntry(currentTerm, command));
        
        // 2. Replicate to followers
        int acks = 1; // self
        for (int peerId = 0; peerId < clusterSize; peerId++) {
            if (peerId == nodeId) continue;
            
            boolean success = sendAppendEntries(peerId);
            if (success) acks++;
        }
        
        // 3. Commit if majority
        if (acks > clusterSize / 2) {
            commitIndex = log.size();
            applyToStateMachine(log.get(log.size() - 1));
            return true; // committed!
        }
        
        return false;
    }
    
    // ==================== HELPER CLASSES ====================
    
    static class LogEntry {
        int term;
        String command;
        LogEntry(int term, String command) {
            this.term = term;
            this.command = command;
        }
    }
    
    static class RequestVoteRequest {
        int term, candidateId, lastLogIndex, lastLogTerm;
        RequestVoteRequest(int term, int candidateId, int lastLogIndex, int lastLogTerm) {
            this.term = term;
            this.candidateId = candidateId;
            this.lastLogIndex = lastLogIndex;
            this.lastLogTerm = lastLogTerm;
        }
    }
    
    static class RequestVoteResponse {
        int term;
        boolean voteGranted;
        RequestVoteResponse(int term, boolean voteGranted) {
            this.term = term;
            this.voteGranted = voteGranted;
        }
    }
    
    // Métodos de rede (simplificados)
    private RequestVoteResponse sendRequestVote(int peerId, RequestVoteRequest req) {
        // Implementação de rede real aqui (gRPC, etc.)
        return null;
    }
    
    private boolean sendAppendEntries(int peerId) {
        // Implementação de rede real aqui
        return true;
    }
    
    private void sendHeartbeats() { /* ... */ }
    private void applyToStateMachine(LogEntry entry) { /* aplicar comando */ }
    private int getLastLogIndex() { return log.size(); }
    private int getLastLogTerm() { 
        return log.isEmpty() ? 0 : log.get(log.size() - 1).term; 
    }
}
```

---

## Variantes e Otimizações

### Raft Optimizations

```
  ┌─────────────────────────────────────────────────────┐
  │ RAFT OPTIMIZATIONS                                   │
  │                                                     │
  │ 1. Batching:                                        │
  │    Agrupar múltiplos commands em um AppendEntries   │
  │    → Reduz número de RPCs                           │
  │                                                     │
  │ 2. Pipeline:                                        │
  │    Enviar próximo AppendEntries sem esperar ack     │
  │    → Maior throughput                               │
  │                                                     │
  │ 3. Read optimization (ReadIndex):                   │
  │    Leituras sem escrever no log                     │
  │    → Leader confirm que ainda é líder (heartbeat)   │
  │    → Read direto da state machine                   │
  │                                                     │
  │ 4. Lease-based reads:                               │
  │    Leader tem "lease" garantindo liderança           │
  │    → Read sem nenhuma round-trip                    │
  │    → Requer clock sincronizado (bounded skew)       │
  │                                                     │
  │ 5. Log compaction (Snapshot):                       │
  │    Periodicamente salvar state machine snapshot     │
  │    → Truncar log antigo                             │
  │    → Novos followers recebem snapshot + log recente │
  │                                                     │
  │ 6. Pre-vote:                                        │
  │    Antes de iniciar eleição, perguntar se há líder  │
  │    → Evita eleições desnecessárias por network issue │
  └─────────────────────────────────────────────────────┘
```

### Paxos Variants

```
  ┌─────────────────────────────────────────────────────┐
  │ PAXOS FAMILY                                         │
  │                                                     │
  │ Basic Paxos:     Consenso sobre 1 valor             │
  │ Multi-Paxos:     Sequência de valores (com líder)   │
  │ Fast Paxos:      Reduz latência (2 → 1.5 RTTs)     │
  │                  Requer 3f+1 nós (ao invés de 2f+1) │
  │ Cheap Paxos:     Usa f+1 nós ativos + f backup     │
  │ Egalitarian:     Sem líder (EPaxos, 2016)           │
  │                  Qualquer nó aceita, commit em 1 RTT│
  │ Flexible Paxos:  Permite quorums não-majority       │
  │                  Ex: write quorum=4, read quorum=2  │
  └─────────────────────────────────────────────────────┘
```

---

## Uso em Big Techs

### Google
- **Spanner**: usa Paxos para replicação cross-datacenter (cada shard é um Paxos group)
- **Chubby**: lock service com Paxos para consenso
- **Megastore**: Paxos para replicação WAN síncrona
- **Bigtable**: coordena tablet servers via Chubby (Paxos underneath)

### Amazon (AWS)
- **DynamoDB**: Paxos-based para leader election e metadata
- **Aurora**: custom consensus para replicação de storage (quorum writes 4/6, reads 3/6)
- **EBS**: Paxos para metadata management

### Hashicorp
- **Consul**: usa Raft para consistência do key-value store e service catalog
- **Vault**: Raft como storage backend (integrated storage)
- **Nomad**: Raft para scheduling state

### CockroachDB / TiDB
- **CockroachDB**: Raft per range (cada range de dados tem seu Raft group)
- **TiDB/TiKV**: Multi-Raft — milhares de Raft groups (um por region de dados)

### LinkedIn (Apache Kafka)
- **KRaft**: Kafka Raft Metadata mode (substitui ZooKeeper)
- Controller usa Raft para consenso sobre metadata de partitions/brokers

---

## Perguntas Comuns em Entrevistas

1. **Explique como Raft funciona em alto nível**
   - Leader election via randomized timeout → Log replication via AppendEntries → Commit quando majority confirma.

2. **Raft vs Paxos — quando usar qual?**
   - Raft: maioria dos casos (mais fácil de implementar e debugar). Paxos: workloads acadêmicos ou quando precisa de variantes (EPaxos para multi-leader).

3. **O que é Quorum e por que majority?**
   - Majority (⌊N/2⌋+1) garante que qualquer 2 quorums compartilham pelo menos 1 nó — preserva informação committed entre eleições.

4. **Como Raft lida com network partitions?**
   - Partição com majority elege líder e continua operando. Partição minoritária não consegue commit (sem quorum). Ao reconectar, followers atrasados fazem catch-up.

5. **O que é a propriedade Leader Completeness?**
   - Se um entry é committed, estará no log de todos os futuros leaders. Garantida pela voting rule: candidato precisa ter log tão atualizado quanto voter.

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Algoritmo** | Raft (entendível, strong leader) | Paxos (flexível, multi-leader) |
| **Cluster size** | 3 nós (1 falha, menor latência) | 5 nós (2 falhas, mais resiliente) |
| **Reads** | Via líder (linearizable) | Via followers (stale, menos carga) |
| **Replication** | Síncrona (consistent, slower) | Assíncrona (fast, risk of data loss) |
| **Log compaction** | Snapshots (simples) | Incremental (eficiente) |
| **Scope** | Raft group por cluster | Multi-Raft (group por shard) |
| **Leader lease** | Com lease (fast reads) | Sem lease (safe, mais RTTs) |

---

## Referências

- [Raft Paper — In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)
- [Raft Visualization — Interactive](https://raft.github.io/)
- [Lamport — The Part-Time Parliament (Paxos)](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf)
- [Lamport — Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [Martin Kleppmann — Designing Data-Intensive Applications, Chapters 8-9](https://dataintensive.net/)
- [Diego Ongaro — Raft PhD Dissertation](https://github.com/ongardie/dissertation)
- [etcd — Raft Implementation](https://github.com/etcd-io/raft)
- [TiKV — Multi-Raft Design](https://tikv.org/deep-dive/scalability/multi-raft/)
