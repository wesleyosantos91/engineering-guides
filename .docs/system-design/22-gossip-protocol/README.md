# 22. Gossip Protocol

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Avançado — frequente em entrevistas de System Design  
> **Complexidade:** Média-Alta

---

## Definição

**Gossip Protocol** (ou Epidemic Protocol) é um protocolo de comunicação **peer-to-peer** onde cada nó compartilha informações com **vizinhos aleatórios**, propagando "rumores" de forma **exponencial** pela rede — similar a como um boato se espalha em uma multidão.

Cada nó seleciona periodicamente um ou mais nós aleatórios e troca informações com eles. Após $O(\log N)$ rodadas, **todos os nós no cluster** recebem a informação.

---

## Por Que é Importante?

- **Base de membership e failure detection** em sistemas distribuídos (Cassandra, DynamoDB)
- **Alternativa descentralizada** a soluções centralizadas (sem single point of failure)
- **Escalável para clusters enormes** — overhead cresce lentamente com o tamanho
- **Pergunta frequente** em entrevistas sobre sistemas distribuídos

---

## Analogia: Fofoca na Escola

```
t=0: Apenas Alice sabe a novidade
     ┌───┐
     │ A │ ★ (sabe)
     └───┘
     ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐
     │ B │ │ C │ │ D │ │ E │ │ F │  (não sabem)
     └───┘ └───┘ └───┘ └───┘ └───┘

t=1: Alice conta para B e D (aleatório)
     ┌───┐ ┌───┐       ┌───┐
     │ A │→│ B │       │ D │←│ A │
     └───┘ └───┘       └───┘
     ★      ★     ○     ★     ○     ○

t=2: A→C, B→E, D→F (cada nó que sabe conta para outro)
     ★      ★     ★     ★     ★     ★
     TODOS SABEM! Em apenas 2 rodadas para 6 nós.
     
     Matematicamente: log₂(6) ≈ 2.6 rodadas
```

---

## Como Funciona

### Ciclo de Gossip

```
┌─────────────────────────────────────────┐
│           GOSSIP CYCLE                   │
│                                          │
│  Cada T segundos (gossip interval):      │
│                                          │
│  1. Nó seleciona K peers aleatórios      │
│     (fanout, tipicamente K=1-3)          │
│                                          │
│  2. Envia "digest" (resumo do estado)    │
│     para os peers selecionados           │
│                                          │
│  3. Peer compara com seu estado local    │
│                                          │
│  4. Troca informações divergentes        │
│     (push, pull, ou push-pull)           │
│                                          │
│  5. Ambos os nós atualizam estado local  │
└─────────────────────────────────────────┘
```

### Modos de Propagação

```
PUSH:
  Node A ──── estado ────▶ Node B
  (A envia o que sabe)

PULL:
  Node A ◀──── "o que você sabe?" ──── Node B
  Node A ──── resposta ────────────▶ Node B
  (A pede informação de B)

PUSH-PULL (mais eficiente):
  Node A ──── digest ──────▶ Node B
  Node A ◀──── delta ──────── Node B
  Node A ──── delta ──────▶ Node B
  (ambos trocam o que um tem e o outro não)
```

| Modo | Velocidade | Overhead | Convergência |
|------|-----------|----------|--------------|
| **Push** | Rápida no início | Desperdício quando maioria sabe | Boa |
| **Pull** | Rápida no final | Desperdício quando poucos sabem | Boa |
| **Push-Pull** | Rápida sempre | Ótimo | Excelente ✓ |

---

## Propriedades Matemáticas

### Convergência

```
Dado um cluster de N nós e fanout f:

Rounds para convergência ≈ O(log N)

Probabilidade de todos os nós saberem após t rounds:
  P(full propagation) ≈ 1 - N × e^(-f×t)

Exemplo: N=1000, f=3
  t=10 rounds → P ≈ 99.99%
  
  Mesmo em cluster com 1000 nós, ~10 rounds bastam!
```

### Overhead

```
Mensagens por round por nó:  f (fanout)
Total mensagens por round:   N × f
Total para convergência:     N × f × O(log N)

Para N=1000, f=3:
  ~1000 × 3 × 10 = 30,000 mensagens totais
  (vs broadcast: 1000 × 999 ≈ 1,000,000 mensagens)
```

---

## Casos de Uso

### 1. Failure Detection (Gossip Heartbeat)

```
Cada nó mantém tabela de heartbeats:

Node A heartbeat table:
┌──────┬───────────┬────────────────┐
│ Node │ Heartbeat │ Last Updated   │
├──────┼───────────┼────────────────┤
│  A   │    147    │ 12:00:01.000   │
│  B   │    142    │ 12:00:00.800   │
│  C   │    145    │ 12:00:00.900   │
│  D   │    138    │ 11:59:58.000   │ ← Stale! (>T timeout)
└──────┴───────────┴────────────────┘

Processo:
1. Cada nó incrementa seu heartbeat counter periodicamente
2. No gossip round, envia sua tabela para peer
3. Peer merge: max(local.heartbeat, received.heartbeat)
4. Se heartbeat de um nó não é atualizado por T segundos:
   → Nó marcado como SUSPECT
   → Após mais T' segundos → marcado como DEAD
```

### 2. Cluster Membership (SWIM Protocol)

```
SWIM (Scalable Weakly-consistent Infection-style Membership):

  Direct Probe:
    Node A ──── ping ────▶ Node B
    Node A ◀──── ack ─────── Node B   ✓ Alive

  Indirect Probe (se direct falha):
    Node A ──── "ping-req B" ──▶ Node C
    Node C ──── ping ──────────▶ Node B
    Node C ◀──── ack ──────────── Node B
    Node A ◀──── ack ──────────── Node C   ✓ Alive via C

  Se ambos falharem:
    Node A marca B como SUSPECT
    Propaga via gossip: "B is suspect"
    Se B não refuta em T → marcado como DEAD
```

### 3. State Dissemination

```
Propagação de metadados pelo cluster:

Exemplo: Cassandra schema changes

  DBA executa ALTER TABLE em Node A
  
  Round 1: A → B (gossip schema version)
  Round 2: A → D, B → C
  Round 3: A → E, B → F, C → G, D → H
  ...
  
  Após ~log₂(N) rounds: todos os nós têm o novo schema
  
  Cada nó compara schema version:
    Se local < received → pull schema details
    Se local == received → noop
    Se local > received → push schema details
```

---

## Implementação Simplificada

```python
import random
import time
import threading
from dataclasses import dataclass, field
from typing import Dict, List

@dataclass
class NodeState:
    node_id: str
    heartbeat: int = 0
    timestamp: float = 0.0
    status: str = "ALIVE"  # ALIVE, SUSPECT, DEAD

class GossipNode:
    def __init__(self, node_id: str, peers: List[str], 
                 fanout: int = 3, interval: float = 1.0,
                 suspect_timeout: float = 5.0):
        self.node_id = node_id
        self.peers = peers
        self.fanout = fanout
        self.interval = interval
        self.suspect_timeout = suspect_timeout
        
        # Membership table
        self.members: Dict[str, NodeState] = {
            node_id: NodeState(node_id, 0, time.time())
        }
    
    def gossip_round(self):
        """Um round de gossip"""
        # 1. Incrementa próprio heartbeat
        self.members[self.node_id].heartbeat += 1
        self.members[self.node_id].timestamp = time.time()
        
        # 2. Seleciona peers aleatórios
        targets = random.sample(
            [p for p in self.peers if p != self.node_id],
            min(self.fanout, len(self.peers) - 1)
        )
        
        # 3. Envia digest para cada target
        digest = self._create_digest()
        for target in targets:
            self._send_gossip(target, digest)
    
    def receive_gossip(self, remote_digest: Dict[str, NodeState]):
        """Processa gossip recebido — merge state"""
        for node_id, remote_state in remote_digest.items():
            local_state = self.members.get(node_id)
            
            if local_state is None:
                # Nó desconhecido — adiciona
                self.members[node_id] = remote_state
            elif remote_state.heartbeat > local_state.heartbeat:
                # Remote tem info mais recente
                self.members[node_id] = remote_state
    
    def check_failures(self):
        """Detecta nós suspeitos/mortos"""
        now = time.time()
        for node_id, state in self.members.items():
            if node_id == self.node_id:
                continue
            
            elapsed = now - state.timestamp
            if elapsed > self.suspect_timeout * 2:
                state.status = "DEAD"
            elif elapsed > self.suspect_timeout:
                state.status = "SUSPECT"
            else:
                state.status = "ALIVE"
    
    def _create_digest(self) -> Dict[str, NodeState]:
        return dict(self.members)
    
    def _send_gossip(self, target: str, digest):
        """Envia gossip via rede (simplificado)"""
        # Na prática: UDP datagram ou TCP
        pass
```

---

## Gossip em Sistemas Reais

### Apache Cassandra

```
Cassandra Gossip:

  Gossip Interval: 1 segundo
  
  Informações propagadas via gossip:
  ├── Heartbeat state (generation + version)
  ├── Application state:
  │   ├── STATUS (joining, leaving, normal)
  │   ├── LOAD (data size no nó)
  │   ├── SCHEMA (schema version hash)
  │   ├── DC / RACK (datacenter, rack)
  │   ├── TOKENS (token ranges do nó)
  │   └── SEVERITY (IO pressure)
  
  Failure Detection:
  ├── Phi Accrual Failure Detector
  │   (probabilístico, não binário)
  │   φ > 8 → suspect (configurável: phi_convict_threshold)
  └── Mais adaptativo que timeout fixo

  nodetool gossipinfo:
  /10.0.0.1
    generation:1677012345
    heartbeat:12847
    STATUS:NORMAL,-4567890123456789
    LOAD:42.5 GB
    SCHEMA:abc123-def456
    DC:us-east-1
    RACK:rack1
```

### Amazon DynamoDB (Dynamo Paper)

```
Dynamo usa gossip para:

1. Membership & Failure Detection:
   - Cada nó mantém history of membership changes
   - Gossip propaga membership events
   - Nó detecta falha via gossip heartbeats
   
2. Token Assignment:
   - Mapping de nós para virtual nodes no consistent hash ring
   - Propagado via gossip
   
3. Partitioning Metadata:
   - Quais nós são responsáveis por quais partitions
   - Atualizado quando nós entram/saem do cluster
```

### HashiCorp Consul & Serf

```
Consul usa SWIM protocol (gossip-based):

  Membership:
  ├── Join: novo nó contacta seed nodes
  ├── Gossip propaga novo membro para cluster
  ├── Probe: ping direto + indirect probe
  └── Dead: detectado via gossip, removido após timeout

  Health Checking:
  ├── Anti-entropy: estado local vs catalog
  ├── Gossip propaga health status changes
  └── Deregistration após critical state timeout

  Consul Agent logs:
  [INFO]  serf: EventMemberJoin: node-1 10.0.0.1
  [INFO]  memberlist: Suspect node-2 has failed
  [INFO]  serf: EventMemberFailed: node-2 10.0.0.2
```

---

## CRDT + Gossip — Convergência Garantida

```
CRDT (Conflict-free Replicated Data Types):
  Estruturas de dados que merge automaticamente sem conflito.

  G-Counter (grow-only counter):
  
    Node A: {A: 5, B: 0, C: 0}  → total = 5
    Node B: {A: 0, B: 3, C: 0}  → total = 3
    Node C: {A: 0, B: 0, C: 7}  → total = 7
    
    Merge (element-wise max):
    Result: {A: 5, B: 3, C: 7}  → total = 15 ✓
    
  Gossip propaga os CRDTs → merge é automático e correto.

  Usado em:
  - Riak (distributed DB)
  - Redis CRDT (enterprise)
  - Soundcloud (Roshi — CRDT set para feeds)
```

---

## Trade-offs

| Aspecto | Prós | Contras |
|---------|------|---------|
| **Descentralização** | Sem single point of failure | Sem garantia de ordem |
| **Escalabilidade** | O(log N) convergência | Overhead constante mesmo sem mudanças |
| **Simplicidade** | Fácil de implementar | Difícil de debugar/testar |
| **Resiliência** | Tolera falhas de nós | Informação pode ser stale temporariamente |
| **Bandwidth** | Baixo por nó por round | Duplicação de mensagens |

### Gossip vs Alternativas

| Approach | Convergência | Overhead | Fault Tolerance | Ordering |
|----------|-------------|----------|-----------------|----------|
| **Gossip** | O(log N) | Baixo | Excelente | Nenhuma |
| **Broadcast** | O(1) | Alto (N²) | Baixa | Possível |
| **Centralized** | O(1) | Médio | SPOF | Forte |
| **Tree-based** | O(log N) | Baixo | Média | Possível |

---

## Parâmetros de Tuning

| Parâmetro | Descrição | Trade-off |
|-----------|-----------|-----------|
| **Gossip Interval** | Tempo entre rounds (ex: 1s) | Menor = mais rápido, mais overhead |
| **Fanout** | Nós contactados por round (ex: 3) | Maior = convergência mais rápida, mais bandwidth |
| **Suspect Timeout** | Tempo antes de marcar como dead | Menor = detecção rápida, mais false positives |
| **Message Size** | Digest completo vs delta | Completo = simples; delta = eficiente |
| **Protocol** | UDP vs TCP | UDP = rápido; TCP = confiável |

---

## Uso em Big Techs

| Empresa | Sistema | Detalhes |
|---------|---------|----------|
| **Amazon** | DynamoDB | Gossip para membership e failure detection (Dynamo paper) |
| **Apache** | Cassandra | Gossip para topology, schema, load balancing |
| **HashiCorp** | Consul/Serf | SWIM protocol para membership e health |
| **Uber** | Ringpop | Gossip-based membership para microservices |
| **Redis** | Cluster | Gossip para config, failure detection entre nós |
| **ScyllaDB** | Database | Gossip otimizado com compressão |

---

## Perguntas Frequentes em Entrevistas

1. **"Explique o Gossip Protocol"**
   - Comunicação P2P onde cada nó conta para vizinhos aleatórios
   - Propagação exponencial: O(log N) rounds
   - Usado para membership, failure detection, state dissemination

2. **"Por que não usar broadcast?"**
   - Broadcast: O(N²) mensagens — não escala
   - Gossip: O(N × log N) — escala para milhares de nós
   - Gossip também tolera falhas de nós

3. **"Como Cassandra detecta nós com falha?"**
   - Gossip heartbeats + Phi Accrual Failure Detector
   - Probabilístico: suspicion level (φ) em vez de threshold fixo
   - Mais adaptativo a variações de rede

4. **"Gossip garante consistência?"**
   - Eventual consistency — todos convergem eventualmente
   - Combine com CRDTs para merge automático sem conflitos
   - Não garante ordem de operações

---

## Referências

- Demers et al. (1987) — *"Epidemic Algorithms for Replicated Database Maintenance"* — Paper pioneiro
- DeCandia et al. (2007) — *"Dynamo: Amazon's Highly Available Key-value Store"*
- Das, Gupta, Stump (2002) — *"SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol"*
- Lakshman & Malik (2010) — *"Cassandra: A Decentralized Structured Storage System"*
- Shapiro et al. (2011) — *"Conflict-free Replicated Data Types"* — CRDTs
- van Renesse et al. (1998) — *"Astrolabe: A Robust and Scalable Technology for Distributed System Monitoring"*
