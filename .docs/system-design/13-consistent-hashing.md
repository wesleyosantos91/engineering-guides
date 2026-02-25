# 13. Consistent Hashing

> **Categoria:** Fundamentos de Sistemas Distribuídos  
> **Nível:** Essencial para entrevistas de System Design  
> **Complexidade:** Média (conceito elegante, implementação sutil)

---

## Definição

**Consistent Hashing** é uma técnica de distribuição de dados em que tanto os **nós (servidores)** quanto as **keys (dados)** são mapeados para posições em um **anel (ring)** usando uma função hash. Cada key é servida pelo **próximo nó no sentido horário** do ring.

A principal vantagem: quando um nó é adicionado ou removido, **apenas uma fração das keys** precisa ser redistribuída — ao contrário do hash tradicional, onde **todas** as keys são afetadas.

---

## Por Que é Importante?

- **Resolve o problema** de redistribuição massiva ao escalar horizontalmente
- **Fundamento** de praticamente todo sistema distribuído moderno
- **Usado por** Amazon DynamoDB, Apache Cassandra, Akamai CDN, Discord
- **Pergunta frequente** em entrevistas de System Design (design distributed cache, key-value store)
- **Habilita** auto-scaling e elasticidade em ambientes cloud

---

## O Problema: Hash Simples

### Mod-Based Hashing

```
  Hash simples: server = hash(key) % N
  
  N = 4 servidores
  
  hash("user:1") % 4 = 2  → Server 2
  hash("user:2") % 4 = 0  → Server 0
  hash("user:3") % 4 = 1  → Server 1
  hash("user:4") % 4 = 3  → Server 3
  
  ┌──────────┬──────────┬──────────┬──────────┐
  │ Server 0 │ Server 1 │ Server 2 │ Server 3 │
  │ user:2   │ user:3   │ user:1   │ user:4   │
  └──────────┴──────────┴──────────┴──────────┘
```

### O Problema ao Adicionar/Remover Servidor

```
  Server 3 falha → N passa de 4 para 3
  
  ANTES (N=4):                    DEPOIS (N=3):
  hash("user:1") % 4 = 2         hash("user:1") % 3 = 0  ← MUDOU!
  hash("user:2") % 4 = 0         hash("user:2") % 3 = 2  ← MUDOU!
  hash("user:3") % 4 = 1         hash("user:3") % 3 = 1  (mesma)
  hash("user:4") % 4 = 3         hash("user:4") % 3 = 0  ← MUDOU!
  
  ⚠️ 75% das keys foram redistribuídas!
  ⚠️ Cache miss massivo (thundering herd)
  ⚠️ Rebalanceamento de dados enorme
  
  Em produção com milhões de keys → DESASTRE
```

---

## A Solução: Consistent Hashing

### O Hash Ring

```
  1. Mapear espaço de hash para um anel (0 a 2^32 - 1)
  2. Fazer hash dos servidores → posição no anel
  3. Fazer hash das keys → posição no anel
  4. Key pertence ao PRÓXIMO servidor no sentido horário
  
                    0°
                    │
            K4 ─────┤──── N1 (hash("Server1"))
                    │
                    │
       N4 ──────────┤
       (Server4)    │
                    │
            K1 ─────┤──── N2 (hash("Server2"))
                    │
                    │
       K2, K3 ─────┤──── N3 (hash("Server3"))
                    │
                   360° (= 0°)
  
  K1 → pertence a N2 (próximo nó no sentido horário)
  K2 → pertence a N3
  K3 → pertence a N3
  K4 → pertence a N1
```

### Adicionar um Servidor

```
  Adicionando N5 entre N2 e N3:
  
                    0°
                    │
            K4 ─────┤──── N1
                    │
       N4 ──────────┤
                    │
            K1 ─────┤──── N2
                    │
            K2 ─────┤──── N5 (novo!)    ← K2 migra de N3 → N5
                    │
            K3 ─────┤──── N3            ← K3 permanece em N3
                    │
                   360°
  
  ✅ Apenas K2 foi redistribuída (de N3 para N5)
  ✅ K1, K3, K4: NÃO foram afetados
  ✅ Redistribuição mínima: apenas K/N keys em média
```

### Remover um Servidor

```
  Removendo N2:
  
                    0°
                    │
            K4 ─────┤──── N1
                    │
       N4 ──────────┤
                    │
            K1 ─────┤          ← K1 migra para N3 (próximo no sentido horário)
                    │
            K2 ─────┤──── N3
            K3 ─────┤
                    │
                   360°
  
  ✅ Apenas keys de N2 são redistribuídas
  ✅ Outras keys: NÃO são afetadas
```

### Comparativo de Redistribuição

| Operação | Hash Simples (% N) | Consistent Hashing |
|----------|--------------------|--------------------|
| **Adicionar 1 nó (4→5)** | ~80% keys migram | ~K/N = ~20% keys migram |
| **Remover 1 nó (4→3)** | ~75% keys migram | ~K/N = ~25% keys migram |
| **Adicionar 1 nó (100→101)** | ~99% keys migram | ~1% keys migram |
| **Impacto em produção** | Cache miss massivo | Impacto mínimo |

---

## Problema: Distribuição Desigual

### Sem Virtual Nodes

```
  Com poucos nós, a distribuição pode ser muito desigual:
  
                    0°
                    │
                    │
                    │
       N1 ──────────┤
                    │          N1 é responsável por ~60% do ring!
                    │          N2 é responsável por ~30%
                    │          N3 é responsável por ~10%
       N2 ──────────┤
                    │
       N3 ──────────┤
                    │
                   360°
  
  ⚠️ Hotspot: N1 recebe a maioria das requests
  ⚠️ Load balancing desigual
```

---

## Virtual Nodes (vNodes)

**Solução:** Cada servidor físico recebe **múltiplos pontos** (virtual nodes) no ring.

```
  Servidor físico → múltiplos vNodes no ring
  
  Server A: vA1, vA2, vA3, vA4 (4 vNodes)
  Server B: vB1, vB2, vB3, vB4 (4 vNodes)
  Server C: vC1, vC2, vC3, vC4 (4 vNodes)
  
                    0°
                    │
            vA1 ────┤
                    │
            vB1 ────┤
                    │
            vC1 ────┤
                    │
            vA2 ────┤
                    │
            vC2 ────┤
                    │
            vB2 ────┤
                    │
            vA3 ────┤
                    │
            vB3 ────┤
                    │
            vC3 ────┤
                    │
            vA4, vB4, vC4...
                   360°
  
  ✅ Distribuição muito mais uniforme
  ✅ Cada servidor ocupa múltiplos segmentos do ring
```

### Benefícios dos Virtual Nodes

| Benefício | Descrição |
|-----------|-----------|
| **Distribuição uniforme** | Mais vNodes = distribuição mais equilibrada |
| **Heterogeneidade** | Servidor mais potente → mais vNodes → mais carga |
| **Rebalanceamento suave** | Ao remover nó, carga distribui entre TODOS os outros |
| **Gradual scaling** | Adicionar vNodes um a um para scaling incremental |

### Ponderação por Capacidade

```
  Server A (16 CPU, 64 GB): 200 vNodes  → ~40% da carga
  Server B (8 CPU, 32 GB):  100 vNodes  → ~20% da carga
  Server C (32 CPU, 128 GB): 300 vNodes  → ~40% da carga
  
  → Carga proporcional à capacidade de cada servidor
```

### Quantos Virtual Nodes?

| # vNodes | Distribuição | Memória | Rebalancing |
|----------|-------------|---------|-------------|
| 1 | Muito desigual | Mínima | Rápido |
| 10-50 | Razoável | Baixa | Rápido |
| 100-200 | Boa | Moderada | Médio |
| 500+ | Excelente | Alta | Mais lento |

**Recomendação:** 100-200 vNodes por servidor é um bom sweet spot.

---

## Implementação

### Python — Consistent Hash Ring

```python
import hashlib
from bisect import bisect_right
from collections import defaultdict

class ConsistentHashRing:
    def __init__(self, nodes=None, vnodes=150):
        """
        Args:
            nodes: Lista de nós (servidores)
            vnodes: Número de virtual nodes por servidor
        """
        self.vnodes = vnodes
        self.ring = {}           # hash_value → node
        self.sorted_keys = []    # sorted hash values
        self.node_count = defaultdict(int)  # keys per node
        
        if nodes:
            for node in nodes:
                self.add_node(node)
    
    def _hash(self, key: str) -> int:
        """Consistent hash function using MD5"""
        digest = hashlib.md5(key.encode()).hexdigest()
        return int(digest, 16)
    
    def add_node(self, node: str):
        """Adiciona nó com virtual nodes ao ring"""
        for i in range(self.vnodes):
            vnode_key = f"{node}:vnode{i}"
            hash_val = self._hash(vnode_key)
            self.ring[hash_val] = node
            self.sorted_keys.append(hash_val)
        
        self.sorted_keys.sort()
    
    def remove_node(self, node: str):
        """Remove nó e seus virtual nodes do ring"""
        for i in range(self.vnodes):
            vnode_key = f"{node}:vnode{i}"
            hash_val = self._hash(vnode_key)
            del self.ring[hash_val]
            self.sorted_keys.remove(hash_val)
    
    def get_node(self, key: str) -> str:
        """Retorna o nó responsável pela key"""
        if not self.ring:
            return None
        
        hash_val = self._hash(key)
        # Encontra o próximo nó no sentido horário
        idx = bisect_right(self.sorted_keys, hash_val)
        
        # Wrap around: se passou do final, volta ao início
        if idx == len(self.sorted_keys):
            idx = 0
        
        return self.ring[self.sorted_keys[idx]]
    
    def get_nodes(self, key: str, replicas: int = 3) -> list:
        """Retorna N nós distintos para replicação"""
        if not self.ring:
            return []
        
        hash_val = self._hash(key)
        idx = bisect_right(self.sorted_keys, hash_val)
        
        nodes = []
        seen = set()
        
        for i in range(len(self.sorted_keys)):
            pos = (idx + i) % len(self.sorted_keys)
            node = self.ring[self.sorted_keys[pos]]
            
            if node not in seen:
                nodes.append(node)
                seen.add(node)
            
            if len(nodes) == replicas:
                break
        
        return nodes


# Uso
ring = ConsistentHashRing(
    nodes=["server-A", "server-B", "server-C"],
    vnodes=150
)

# Encontrar servidor para uma key
server = ring.get_node("user:12345")
print(f"user:12345 → {server}")

# Encontrar 3 réplicas para uma key
replicas = ring.get_nodes("user:12345", replicas=3)
print(f"user:12345 replicas → {replicas}")

# Adicionar novo servidor — redistribuição mínima
ring.add_node("server-D")

# Verificar: a maioria das keys continua nos mesmos servidores
server_after = ring.get_node("user:12345")
print(f"user:12345 → {server_after}")  # provavelmente mesmo servidor
```

### Java — Consistent Hash Ring

```java
import java.security.MessageDigest;
import java.util.*;

public class ConsistentHashRing<T> {
    private final int vnodes;
    private final TreeMap<Long, T> ring = new TreeMap<>();
    
    public ConsistentHashRing(int vnodes) {
        this.vnodes = vnodes;
    }
    
    public void addNode(T node) {
        for (int i = 0; i < vnodes; i++) {
            long hash = hash(node.toString() + ":vnode" + i);
            ring.put(hash, node);
        }
    }
    
    public void removeNode(T node) {
        for (int i = 0; i < vnodes; i++) {
            long hash = hash(node.toString() + ":vnode" + i);
            ring.remove(hash);
        }
    }
    
    public T getNode(String key) {
        if (ring.isEmpty()) return null;
        
        long hash = hash(key);
        // Próximo nó no sentido horário (ceiling)
        Map.Entry<Long, T> entry = ring.ceilingEntry(hash);
        
        // Wrap around
        if (entry == null) {
            entry = ring.firstEntry();
        }
        
        return entry.getValue();
    }
    
    public List<T> getNodes(String key, int replicas) {
        if (ring.isEmpty()) return Collections.emptyList();
        
        long hash = hash(key);
        List<T> nodes = new ArrayList<>();
        Set<T> seen = new HashSet<>();
        
        // Itera a partir do hash no sentido horário
        for (Map.Entry<Long, T> entry : ring.tailMap(hash).entrySet()) {
            if (addUniqueNode(entry.getValue(), nodes, seen, replicas)) return nodes;
        }
        // Wrap around
        for (Map.Entry<Long, T> entry : ring.entrySet()) {
            if (addUniqueNode(entry.getValue(), nodes, seen, replicas)) return nodes;
        }
        
        return nodes;
    }
    
    private boolean addUniqueNode(T node, List<T> nodes, Set<T> seen, int replicas) {
        if (!seen.contains(node)) {
            nodes.add(node);
            seen.add(node);
        }
        return nodes.size() >= replicas;
    }
    
    private long hash(String key) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] digest = md.digest(key.getBytes());
            return ((long)(digest[3] & 0xFF) << 24) |
                   ((long)(digest[2] & 0xFF) << 16) |
                   ((long)(digest[1] & 0xFF) << 8) |
                   ((long)(digest[0] & 0xFF));
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

---

## Consistent Hashing com Replicação

```
  Replicação: cada key é armazenada em N nós consecutivos no ring
  
  Exemplo com N=3 (3 réplicas):
  
                    0°
                    │
            K1 ─────┤──── N1 (primary para K1)
                    │
                    ├──── N2 (replica 1 para K1)
                    │
                    ├──── N3 (replica 2 para K1)
                    │
                    ├──── N4
                    │
                   360°
  
  K1 é armazenada em: N1 (primary), N2 (replica), N3 (replica)
  
  Se N1 falha:
  → Reads de K1 vão para N2 ou N3
  → Novo write para K1 vai para N2 (novo primary)
  → Quando N1 volta, faz catch-up de N2/N3
```

---

## Variações e Otimizações

### Jump Consistent Hashing (Google)

```
  - Mais simples e rápido que ring-based
  - Usa apenas uma função matemática (sem ring)
  - Distribuição perfeitamente uniforme
  - Limitação: nós só podem ser adicionados/removidos do FINAL
  
  int jump_consistent_hash(uint64_t key, int num_buckets) {
      int64_t b = -1, j = 0;
      while (j < num_buckets) {
          b = j;
          key = key * 2862933555777941757ULL + 1;
          j = (int64_t)((b + 1) * 
              ((double)(1LL << 31) / 
               (double)((key >> 33) + 1)));
      }
      return (int)b;
  }
```

### Rendezvous Hashing (Highest Random Weight)

```
  Para cada key, calcula score para TODOS os nós:
  score(key, node) = hash(key + node)
  
  Key vai para o nó com MAIOR score.
  
  key="user:1":
    score("user:1", "server-A") = 0.82  ← WINNER
    score("user:1", "server-B") = 0.45
    score("user:1", "server-C") = 0.71
  
  → user:1 vai para server-A
  
  Prós: Distribuição perfeita, simples
  Contras: O(N) por lookup (precisa calcular para todos os nós)
```

### Bounded Load Consistent Hashing (Google, 2017)

```
  Problema: Mesmo com vNodes, hot keys podem sobrecarregar um nó
  
  Solução: Cada nó tem capacidade máxima = (1 + ε) × carga média
  
  Se nó está "cheio" → key vai para PRÓXIMO nó no ring
  
  ε = 0.25:
  Carga média = 100 keys/nó
  Capacidade máxima = 125 keys/nó
  
  → Garante que nenhum nó recebe mais que 25% acima da média
  → Usado pelo Google no Vimeo CDN
```

---

## Uso em Big Techs

### Amazon DynamoDB — Partitioning
- Consistent hashing para distribuir items entre partitions
- Cada partition é um range do hash ring
- Auto-splitting quando partition fica quente
- Virtual nodes para distribuição uniforme

### Apache Cassandra — Data Distribution
- Token ring: cada nó é responsável por um range de tokens
- `Murmur3Partitioner` como hash function padrão
- vNodes (256 por padrão) para balanceamento
- Replicação: N nós consecutivos no ring armazenam réplicas

### Akamai CDN — Content Distribution
- **Inventores** do consistent hashing (1997, Karger et al.)
- Distribui conteúdo web entre edge servers globalmente
- Minimiza redistribuição quando servidores entram/saem

### Discord — Guild Distribution
- Distribui guilds (servers) entre processos de backend
- Consistent hashing garante mínima redistribuição no scaling
- Cada guild tem um "owner" process determinado pelo hash

### Memcached / Redis — Cache Distribution
- Clientes usam consistent hashing para escolher shard
- `ketama` algorithm (libmemcached) — consistent hashing padrão
- Minimiza cache misses ao adicionar/remover nós

### Netflix — EVCache
- Cache distribuído baseado em Memcached
- Consistent hashing para distribuir keys entre shards
- Zone-aware: réplicas em diferentes availability zones

---

## Perguntas Comuns em Entrevistas

1. **O que é consistent hashing e por que é necessário?**
   - Técnica que minimiza redistribuição de dados quando nós são adicionados/removidos. Necessário porque hash simples (`% N`) redistribui quase tudo.

2. **Como virtual nodes resolvem o problema de distribuição desigual?**
   - Cada servidor tem múltiplos pontos no ring, criando segmentos menores e mais uniformes.

3. **Quantos virtual nodes usar?**
   - 100-200 por servidor é um bom sweet spot. Mais = melhor distribuição, mas mais memória.

4. **Como funciona a replicação com consistent hashing?**
   - Cada key é armazenada nos N nós consecutivos no sentido horário do ring.

5. **Design um distributed cache usando consistent hashing.**
   - Hash ring com vNodes, replication factor N=3, read from closest replica, write to primary + async replicate.

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Mais vNodes** | Melhor distribuição | Mais memória, rebalancing mais lento |
| **Hash function** | MD5/SHA (uniforme) | MurmurHash (rápido) |
| **Replicação** | Mais réplicas (durável) | Menos réplicas (menos storage) |
| **Tipo** | Ring-based (flexível) | Jump hash (mais rápido, menos flexível) |
| **Lookup** | O(log N) com binary search | O(N) com rendezvous hashing |
| **Hot key handling** | Bounded load (fair) | Cache aside (simples) |

---

## Referências

- [Karger et al. — Consistent Hashing (1997)](https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/chash.pdf)
- [Jump Consistent Hashing — Google (2014)](https://arxiv.org/abs/1406.2294)
- [Bounded Load Consistent Hashing — Google (2017)](https://ai.googleblog.com/2017/04/consistent-hashing-with-bounded-loads.html)
- [DynamoDB — Partitioning](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.Partitions.html)
- [Cassandra — Consistent Hashing](https://cassandra.apache.org/doc/latest/cassandra/architecture/overview.html)
- System Design Interview — Alex Xu, Vol. 1, Ch. 5: Design Consistent Hashing
