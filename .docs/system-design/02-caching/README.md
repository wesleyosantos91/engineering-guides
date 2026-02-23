# 2. Caching

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial — mencionado em praticamente toda entrevista de System Design  
> **Complexidade:** Média-Alta (estratégias de invalidação são notoriamente difíceis)

---

## Definição

**Caching** é o armazenamento temporário de dados frequentemente acessados em um meio de acesso mais rápido (geralmente memória RAM), para reduzir latência, diminuir carga no banco de dados e melhorar throughput.

> *"There are only two hard things in Computer Science: cache invalidation and naming things."* — Phil Karlton

---

## Por Que é Importante?

| Sem Cache | Com Cache |
|-----------|-----------|
| Toda request bate no DB | Maioria resolvida em memória |
| Latência ~5-50ms (DB) | Latência ~0.1-1ms (Redis) |
| DB se torna bottleneck | DB aliviado, escala melhor |
| Custo alto em IOPS | Menos queries = menos custo |

**Regra 80/20:** Em muitos sistemas, 20% dos dados geram 80% dos acessos — esses são candidatos perfeitos para cache.

---

## Diagrama Geral

```
┌────────┐    ┌─────────────────────────────────────┐
│ Client │    │           Application Server         │
└───┬────┘    │                                     │
    │         │  1. Check Cache ──▶ [Cache Layer]   │
    │ request │       │                  │          │
    │────────▶│       │ HIT ✓           │ MISS ✗   │
    │         │       ▼                  ▼          │
    │         │  Return cached    Query Database    │
    │         │  data             ┌──────────┐     │
    │         │                   │    DB    │     │
    │         │  ◀── Update ───── └──────────┘     │
    │◀────────│      Cache                          │
    │ response│                                     │
              └─────────────────────────────────────┘
```

---

## Níveis de Cache

Cache pode existir em **múltiplas camadas** de um sistema:

```
Client ──▶ [Browser Cache] ──▶ [CDN Cache] ──▶ [API Gateway Cache]
    ──▶ [Application Cache (local)] ──▶ [Distributed Cache (Redis)]
    ──▶ [Database Cache (query cache)] ──▶ [OS/Disk Cache]
```

| Nível | Onde | Latência | Exemplo |
|-------|------|----------|---------|
| **Browser** | Client | ~0 ms | HTTP Cache-Control headers |
| **CDN** | Edge PoP | ~5-50 ms | CloudFront, Cloudflare |
| **API Gateway** | Edge | ~1-5 ms | Kong cache plugin |
| **Local (in-process)** | App memory | ~0.01 ms | Caffeine, Guava Cache |
| **Distributed** | Rede interna | ~0.5-2 ms | Redis, Memcached |
| **Database** | DB server | ~1-5 ms | MySQL Query Cache, pg_stat |
| **OS / Disk** | Kernel | Variável | Page cache, buffer cache |

---

## Estratégias de Cache

### 1. Cache-Aside (Lazy Loading)

A **aplicação** gerencia o cache explicitamente.

```
Read:
1. App consulta Cache
2. Se HIT → retorna dados
3. Se MISS → consulta DB → armazena no Cache → retorna

Write:
1. App escreve no DB
2. App invalida/remove entry do Cache
```

```
         ┌───────┐
         │  App  │
         └─┬───┬─┘
     read  │   │ write
   ┌───────▼┐  │
   │ Cache  │  │ (invalidate on write)
   └───┬────┘  │
       │miss   │
   ┌───▼────┐  │
   │   DB   │◀─┘
   └────────┘
```

| Prós | Contras |
|------|---------|
| Simples de implementar | Cache pode ficar stale |
| Só cacheia dados acessados | Cache MISS = latência extra (DB + cache write) |
| Resiliente: se cache falha, app continua | Thundering herd em cache miss simultâneo |

**Quando usar:** Padrão mais comum. Read-heavy workloads.

### 2. Read-Through

O **cache** busca no DB automaticamente em caso de MISS.

```
1. App consulta Cache
2. Se HIT → retorna
3. Se MISS → Cache consulta DB → armazena → retorna
```

- App não sabe da existência do DB (cache abstrai)
- Cache provider precisa suportar (ex: NCache, Ehcache)
- Diferença vs Cache-Aside: quem busca no banco é o cache, não a app

### 3. Write-Through

Escrita vai para o cache E para o DB **sincronamente** (na mesma operação).

```
Write:
1. App escreve no Cache
2. Cache escreve no DB (síncrono)
3. Confirmação para App

Read:
1. App lê do Cache (sempre atualizado)
```

| Prós | Contras |
|------|---------|
| Cache sempre consistente | Latência de escrita maior (2 writes) |
| Simples de raciocinar | Cache pode ter dados que nunca serão lidos |

**Quando usar:** Dados que requerem consistência forte entre cache e DB.

### 4. Write-Behind (Write-Back)

Escrita vai somente para o cache; cache escreve no DB **assincronamente**.

```
Write:
1. App escreve no Cache → ACK imediato ao client
2. Cache acumula writes
3. Cache faz batch write no DB (async, periódico)
```

| Prós | Contras |
|------|---------|
| Escrita ultra-rápida | Risco de perda de dados se cache falha antes do flush |
| Batch writes = menos I/O no DB | Complexidade de implementação |
| Absorve picos de escrita | Inconsistência temporária |

**Quando usar:** Write-heavy workloads. Dados onde perda temporária é aceitável (analytics, counters).

### 5. Write-Around

Escrita vai direto para o DB, **bypassa o cache** completamente.

```
Write:
1. App escreve diretamente no DB (não toca o cache)

Read:
1. Cache-aside normal (MISS → DB → preenche cache)
```

| Prós | Contras |
|------|---------|
| Evita "cache pollution" | Reads recentes sempre dão cache MISS |
| Bom para dados escritos mas raramente lidos | |

**Quando usar:** Dados escritos frequentemente mas lidos raramente (logs, audit).

### Comparativo Visual

```
                    Cache-Aside     Read-Through    Write-Through    Write-Behind    Write-Around
Read:               App→Cache→DB    App→Cache→DB    App→Cache        App→Cache       App→Cache→DB
Write:              App→DB→inval    App→DB→inval    App→Cache→DB     App→Cache→DB*   App→DB
Consistência:       Eventual        Eventual        Forte            Eventual        Eventual
Latência Write:     Normal          Normal          Alta             Muito baixa     Normal
Complexidade:       Baixa           Média           Média            Alta            Baixa

* Write-Behind: cache→DB é assíncrono
```

---

## Eviction Policies (Políticas de Remoção)

Quando o cache está cheio, qual entry remover?

| Policy | Descrição | Uso |
|--------|-----------|-----|
| **LRU** (Least Recently Used) | Remove o item acessado há mais tempo | Mais popular; bom default |
| **LFU** (Least Frequently Used) | Remove o item menos acessado | Bom quando popularidade é estável |
| **FIFO** (First In, First Out) | Remove o mais antigo | Simples; dados com TTL natural |
| **TTL** (Time To Live) | Expira após tempo fixo | Dados que ficam stale com o tempo |
| **Random** | Remove aleatoriamente | Surpreendentemente bom em certos cenários |
| **ARC** (Adaptive Replacement Cache) | Combina LRU + LFU dinamicamente | IBM/ZFS; melhor hit rate |
| **W-TinyLFU** | Frequency sketch + LRU window | Caffeine (Java); estado da arte |

### Configuração TTL

```
# Redis
SET user:123 '{"name":"João"}' EX 3600    # expira em 1 hora
SET session:abc 'data' PX 900000           # expira em 15 min (ms)

# TTL com jitter (evita cache avalanche)
base_ttl = 3600  # 1 hora
jitter = random(0, 600)  # 0-10 min
final_ttl = base_ttl + jitter  # 3600-4200s
```

---

## Problemas Comuns e Soluções

### 1. Cache Stampede (Thundering Herd)

**Problema:** Key popular expira → milhares de requests simultâneos dão cache MISS → todas batem no DB ao mesmo tempo → DB sobrecarregado.

```
TTL expira para key popular
    │
    ├── Thread 1: MISS → Query DB
    ├── Thread 2: MISS → Query DB
    ├── Thread 3: MISS → Query DB    ← DB overwhelmed!
    ├── Thread 4: MISS → Query DB
    └── ... (milhares)
```

**Soluções:**

| Solução | Como |
|---------|------|
| **Locking** | Primeira thread que dá MISS adquire lock; outras esperam |
| **Request Coalescing** | Múltiplas requests idênticas são agrupadas em uma só |
| **Pre-warming / Refresh-ahead** | Renova cache antes de expirar |
| **Stale-while-revalidate** | Serve dado stale enquanto renova em background |
| **Probabilistic early expiration** | XFetch: some requests renovam antes do TTL real |

```java
// Exemplo: Locking com Redis (SETNX)
String value = redis.get(key);
if (value == null) {
    boolean locked = redis.setnx(lockKey, "1", Duration.ofSeconds(10));
    if (locked) {
        value = db.query(key);
        redis.set(key, value, Duration.ofHours(1));
        redis.del(lockKey);
    } else {
        // Wait and retry
        Thread.sleep(100);
        value = redis.get(key);
    }
}
```

### 2. Cache Penetration

**Problema:** Queries para dados que **não existem** (nem no cache, nem no DB). Sempre dão MISS, sempre batem no DB.

```
GET /user/999999999  (não existe)
  → Cache MISS → DB query → empty → não cacheia → loop infinito
```

**Soluções:**

| Solução | Como |
|---------|------|
| **Cache null values** | Armazena null/empty com TTL curto |
| **Bloom Filter** | Verifica se key possivelmente existe antes de ir ao DB |
| **Input validation** | Rejeita IDs claramente inválidos |

```java
// Cachear null values
String value = redis.get(key);
if (value == null) {
    value = db.query(key);
    if (value == null) {
        redis.set(key, "NULL_MARKER", Duration.ofMinutes(5)); // cache miss com TTL curto
    } else {
        redis.set(key, value, Duration.ofHours(1));
    }
}
```

### 3. Cache Avalanche

**Problema:** Muitas keys expiram **ao mesmo tempo** → DB recebe spike massivo de queries.

**Soluções:**

| Solução | Como |
|---------|------|
| **TTL com Jitter** | Adiciona variação aleatória ao TTL |
| **Staggered expiration** | Distribui TTLs uniformemente |
| **Multi-tier cache** | L1 (local) + L2 (Redis) com TTLs diferentes |
| **Cache never expire** | Use background refresh em vez de TTL |

### 4. Hot Key

**Problema:** Uma key específica recebe volume desproporcional de tráfego (ex: perfil de celebridade).

**Soluções:**
- **Local cache (L1):** Cada app instance cacheia localmente
- **Key replication:** Cria cópias (`key:1`, `key:2`, ...) distribuídas em shards diferentes
- **Read replicas:** Redis com replicas de leitura

---

## Tecnologias

### Redis

```
Tipo: In-memory data structure store
Modelo: Key-Value com estruturas ricas (String, Hash, List, Set, Sorted Set, Stream)
Persistência: RDB snapshots + AOF (append-only file)
HA: Redis Sentinel (failover) ou Redis Cluster (sharding)
Performance: ~100K-200K ops/s por instância
```

**Uso comum:**
```redis
# String (cache simples)
SET user:123 '{"name":"João","email":"j@x.com"}' EX 3600

# Hash (objeto com campos)
HSET user:123 name "João" email "j@x.com"
EXPIRE user:123 3600

# Sorted Set (leaderboard, timeline)
ZADD timeline:user:123 1706198400 "tweet:456"
ZRANGEBYSCORE timeline:user:123 -inf +inf LIMIT 0 20

# HyperLogLog (count unique)
PFADD page:views:2024-01-25 "user:123" "user:456"
PFCOUNT page:views:2024-01-25
```

### Memcached

```
Tipo: In-memory key-value store
Modelo: Strings only (simples)
Persistência: Nenhuma (puro cache)
HA: Client-side consistent hashing
Performance: ~100K+ ops/s por instância
Multi-threaded: Sim (vs Redis single-threaded core)
```

**Quando Redis vs Memcached:**

| Critério | Redis | Memcached |
|----------|-------|-----------|
| Data structures | Rich (Hash, Set, List...) | Strings only |
| Persistência | Sim | Não |
| Pub/Sub | Sim | Não |
| Multi-threaded | Parcial (I/O threads) | Sim |
| Memória eficiente | Overhead por key | Mais eficiente para strings |
| Cluster nativo | Sim | Não (client-side) |
| Caso de uso | Versatile | Puro cache simples |

### Caffeine (Java — Local Cache)

```java
Cache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(5))
    .recordStats()  // hit rate, eviction count
    .build();

// Sync loading
User user = cache.get("user:123", key -> userRepository.findById(key));

// Manual
cache.put("user:123", user);
cache.invalidate("user:123");
```

- Eviction: **W-TinyLFU** (near-optimal hit rate)
- Performance: ~milhões de ops/s (tudo em memória local, zero latência de rede)

### Hazelcast / Apache Ignite

- **Distributed cache** embeddable em JVM
- Suporta near-cache (local) + distributed
- Bom para compute grid + cache

---

## Caching em HTTP

### Cache-Control Headers

```http
# Resposta cacheável por 1 hora em browser e CDN
Cache-Control: public, max-age=3600

# Cacheável apenas no browser, não em CDN
Cache-Control: private, max-age=600

# Sempre revalidar com origin antes de usar cache
Cache-Control: no-cache

# Nunca cachear
Cache-Control: no-store

# Stale-while-revalidate (serve stale enquanto revalida)
Cache-Control: max-age=60, stale-while-revalidate=300
```

### ETag (Conditional Request)

```http
# Response com ETag
HTTP/1.1 200 OK
ETag: "abc123"
Cache-Control: no-cache

# Client revalida
GET /api/users/123
If-None-Match: "abc123"

# Se não mudou
HTTP/1.1 304 Not Modified  (sem body, economiza bandwidth)
```

---

## Cache em Arquitetura Multi-Tier

```
┌───────────┐     ┌───────────┐     ┌───────────┐     ┌──────────┐
│  Browser  │────▶│    CDN    │────▶│  App +    │────▶│    DB    │
│  Cache    │     │  Cache    │     │  Redis    │     │          │
│  (local)  │     │  (edge)   │     │  Cache    │     │          │
└───────────┘     └───────────┘     └───────────┘     └──────────┘
    L1               L2                L3                L4
  ~0ms             ~5-50ms          ~0.5-2ms           ~5-50ms
```

**Exemplo prático:**
1. **Request para avatar de usuário popular**
2. Browser cache (Cache-Control) → HIT → serve do disco local
3. Se expirou → CDN (CloudFront) → HIT → serve do edge
4. Se CDN MISS → App server → Redis → HIT → retorna + atualiza CDN
5. Se Redis MISS → DB → retorna + preenche Redis + CDN

---

## Uso em Big Techs

### Meta — TAO (The Associations and Objects)
- Cache distribuído graph-aware
- Serve **bilhões** de queries/sec para social graph
- Duas camadas: L1 (leaf caches) + L2 (root caches)
- Read-through com write-through

### Twitter — Redis para Timelines
- Cada user tem um sorted set no Redis com seus tweet IDs
- Fan-out-on-write: ao postar, escreve na timeline cache de cada follower
- Cluster Redis com **terabytes** de dados

### Netflix — EVCache
- Wrapper em cima de Memcached
- Distribuído globalmente com replicação cross-region
- Serve dados para o recommendation engine e UI

### Amazon — ElastiCache + DAX
- ElastiCache: Redis/Memcached managed
- DAX: Cache transparente para DynamoDB (microsecond reads)

---

## Perguntas Comuns em Entrevistas

1. **Qual estratégia de cache usar para um sistema read-heavy?** → Cache-Aside com TTL
2. **Como evitar inconsistência entre cache e DB?** → Write-through ou invalidate-on-write
3. **O que fazer quando uma key popular expira?** → Locking, pre-warming, stale-while-revalidate
4. **Quando NÃO usar cache?** → Dados que mudam a cada request, writes > reads, dados muito grandes
5. **LRU vs LFU vs TTL?** → LRU para acesso recente; LFU para popularidade estável; TTL para freshness

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Local vs Distributed** | Caffeine (0 latência, per-instance) | Redis (compartilhado, consistente) |
| **Write Strategy** | Write-through (consistência) | Write-behind (performance) |
| **Eviction** | LRU (simples, bom default) | W-TinyLFU (melhor hit rate) |
| **TTL** | Curto (mais fresh) | Longo (mais hits) |
| **Managed vs Self** | ElastiCache (zero-ops) | Redis self-hosted (controle) |

---

## Referências

- [Redis Documentation](https://redis.io/docs/)
- [Caffeine GitHub](https://github.com/ben-manes/caffeine)
- [Facebook TAO Paper](https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf)
- [Netflix EVCache](https://netflixtechblog.com/announcing-evcache-distributed-in-memory-datastore-for-cloud-c26a698c27f7)
- Designing Data-Intensive Applications — Martin Kleppmann, Chapter 5
- System Design Interview — Alex Xu, Vol 1
