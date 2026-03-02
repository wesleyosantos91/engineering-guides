# Level 2 — Caching & CDN

> **Objetivo:** Implementar um sistema de cache multi-camada (in-memory + distributed) e
> simular um CDN com edge caching, documentando decisões em ADR e diagramando no DrawIO.

**Referência:**
- [02-caching.md](../../.docs/SYSTEM-DESIGN/02-caching.md)
- [03-cdn.md](../../.docs/SYSTEM-DESIGN/03-cdn.md)

**Pré-requisito:** Level 1 completo.

---

## Contexto

**Caching** é a técnica mais eficaz para reduzir latência e carga no banco de dados. Um **CDN** (Content Delivery Network) é um cache distribuído geograficamente para conteúdo estático e dinâmico. Entender cache invalidation, eviction policies e cache-aside vs write-through é fundamental.

---

## Parte 1 — ADR: Estratégia de Caching

**Arquivo:** `docs/adrs/ADR-001-caching-strategy.md`

**Decisão:** Qual estratégia de caching adotar para o sistema.

**Options:**
1. **Cache-Aside (Lazy Loading)** — app lê/escreve no cache explicitamente
2. **Write-Through** — toda escrita vai ao cache e ao DB simultaneamente
3. **Write-Behind (Write-Back)** — escrita no cache, flush assíncrono ao DB
4. **Read-Through** — cache busca do DB automaticamente em miss
5. **Refresh-Ahead** — cache atualiza proativamente antes do TTL

**Decision Drivers:**
- Consistência de dados (strong vs eventual)
- Latência de leitura e escrita
- Complexidade de implementação
- Padrão de acesso (read-heavy vs write-heavy)
- Tolerância a cache miss storms (thundering herd)

**Critérios de aceite:**
- [ ] 5 estratégias documentadas com trade-offs
- [ ] Diagrama de fluxo para cada estratégia (pode ser ASCII no ADR)
- [ ] Cenários de uso recomendados para cada estratégia
- [ ] Eviction policies comparadas (LRU, LFU, TTL, FIFO)
- [ ] Análise de thundering herd e soluções (singleflight, probabilistic early expiration)

---

## Parte 2 — Diagrama DrawIO

**Arquivo:** `docs/diagrams/02-caching-cdn-architecture.drawio`

**View 1 — Multi-Layer Cache:**
```
┌────────────┐
│   Client   │
└─────┬──────┘
      │
┌─────▼──────┐     Cache Hit?
│    CDN     │────────────────▶ Response (< 5ms)
│ (Edge L1)  │
└─────┬──────┘     Cache Miss
      │
┌─────▼──────┐     Cache Hit?
│  App Cache │────────────────▶ Response (< 10ms)
│(In-Memory) │
│   (L2)     │
└─────┬──────┘     Cache Miss
      │
┌─────▼──────┐     Cache Hit?
│Redis/Memcac│────────────────▶ Response (< 20ms)
│(Distributed│
│    L3)     │
└─────┬──────┘     Cache Miss
      │
┌─────▼──────┐
│  Database  │────────────────▶ Response (< 100ms)
│ (Source of │                   + Populate cache
│   Truth)   │
└────────────┘
```

**View 2 — Cache-Aside Flow:** Sequence diagram para read + write

**View 3 — CDN Architecture:** Diagrama com origin server, edge nodes, POP (Points of Presence)

**View 4 — Cache Invalidation:** Fluxo de invalidação (TTL, event-driven, manual purge)

**Critérios de aceite:**
- [ ] 4 views no DrawIO
- [ ] Cache hit/miss paths claramente indicados
- [ ] Latências anotadas em cada camada
- [ ] CDN com múltiplos edge nodes

---

## Parte 3 — Implementação

### 3.1 — Go: Multi-Layer Cache

**Estrutura:**
```
go/
├── cmd/
│   ├── server/main.go            ← API server com cache
│   └── cdn-sim/main.go           ← CDN simulator
├── internal/
│   ├── cache/
│   │   ├── cache.go              ← Interface Cache
│   │   ├── memory.go             ← In-memory LRU cache
│   │   ├── redis.go              ← Redis cache adapter
│   │   ├── multilayer.go         ← Multi-layer cache orchestrator
│   │   ├── singleflight.go       ← Thundering herd protection
│   │   └── metrics.go            ← Cache hit/miss metrics
│   ├── eviction/
│   │   ├── lru.go                ← Least Recently Used
│   │   ├── lfu.go                ← Least Frequently Used
│   │   └── ttl.go                ← TTL-based expiration
│   ├── cdn/
│   │   ├── edge.go               ← Edge node simulator
│   │   ├── origin.go             ← Origin server
│   │   └── routing.go            ← Request routing to nearest edge
│   └── handler/
│       └── api.go                ← HTTP handlers
├── go.mod
└── Makefile
```

**Interface:**
```go
type Cache interface {
    Get(ctx context.Context, key string) ([]byte, error)
    Set(ctx context.Context, key string, value []byte, ttl time.Duration) error
    Delete(ctx context.Context, key string) error
    Exists(ctx context.Context, key string) (bool, error)
}

type MultiLayerCache struct {
    layers []Cache  // L1 (memory) → L2 (redis) → origin (db)
    sf     *singleflight.Group
}
```

**Funcionalidades Go:**
1. **In-Memory Cache** com LRU eviction (doubly linked list + hashmap)
2. **Redis Cache** adapter com connection pooling
3. **Multi-Layer Cache** que busca sequencialmente (L1 → L2 → DB)
4. **Singleflight** para evitar thundering herd
5. **CDN Simulator** com múltiplos edge nodes e geo-routing
6. **Cache warming** na inicialização (popular cache com hot data)
7. **Probabilistic Early Expiration** para evitar stampede
8. **Métricas** (hit rate, miss rate, eviction count, latência por camada)

**Critérios de aceite Go:**
- [ ] LRU implementado do zero (sem biblioteca) com `O(1)` Get/Set
- [ ] LFU implementado com min-heap
- [ ] Multi-layer cache passando por L1 → L2 → DB
- [ ] Singleflight evitando requests duplicados ao DB
- [ ] CDN simulator com 3+ edge nodes
- [ ] Cache hit rate ≥ 80% com dataset sintético
- [ ] Table-driven tests (≥ 15 cenários)
- [ ] Benchmark: `BenchmarkLRUGet`, `BenchmarkLRUSet`, `BenchmarkMultiLayer`

---

### 3.2 — Java: Multi-Layer Cache

**Estrutura:**
```
java/
├── src/main/java/com/challenge/cache/
│   ├── Application.java
│   ├── cache/
│   │   ├── Cache.java                    ← Interface
│   │   ├── InMemoryLruCache.java         ← LRU com LinkedHashMap
│   │   ├── InMemoryLfuCache.java         ← LFU com PriorityQueue
│   │   ├── RedisCacheAdapter.java        ← Spring Data Redis
│   │   ├── MultiLayerCache.java          ← Orchestrator
│   │   └── SingleFlightCache.java        ← Thundering herd protection
│   ├── cdn/
│   │   ├── EdgeNode.java                 ← Record
│   │   ├── CdnSimulator.java
│   │   └── GeoRouter.java
│   ├── config/
│   │   ├── CacheConfig.java
│   │   └── RedisConfig.java
│   ├── controller/
│   │   └── CacheController.java
│   └── metrics/
│       └── CacheMetrics.java             ← Micrometer gauges
├── src/test/java/com/challenge/cache/
│   ├── InMemoryLruCacheTest.java
│   ├── MultiLayerCacheTest.java
│   └── CdnSimulatorTest.java
└── pom.xml
```

**Funcionalidades Java:**
1. **LRU Cache** usando `LinkedHashMap` com `removeEldestEntry`
2. **LFU Cache** com `ConcurrentHashMap` + frequency tracking
3. **Redis adapter** via Spring Data Redis (Lettuce)
4. **Multi-Layer** com `CompletableFuture` para async lookup
5. **CDN Simulator** com Virtual Threads para simular edge nodes
6. **Caffeine** integration como alternativa produção (comparar com implementação manual)
7. **Métricas** com Micrometer (cache.hit, cache.miss, cache.eviction)

**Critérios de aceite Java:**
- [ ] LRU implementado manualmente (não usar Caffeine para esta parte)
- [ ] LFU com tracking de frequência
- [ ] Multi-layer com fallback automático
- [ ] Testes com Testcontainers (Redis real)
- [ ] Comparativo: implementação manual vs Caffeine (benchmark JMH)
- [ ] JaCoCo ≥ 80% coverage
- [ ] `./mvnw test` passa sem erros

---

## Parte 4 — Experimento

### Cache Performance Analysis

1. Gere um dataset de 100K chaves com distribuição Zipf (80/20)
2. Execute 1M reads com diferentes configurações de cache
3. Meça e compare:

| Métrica | Sem Cache | L1 Only | L1+L2 | L1+L2+CDN |
|---------|-----------|---------|-------|-----------|
| p50 latency | | | | |
| p99 latency | | | | |
| Hit rate | | | | |
| DB load (QPS) | | | | |

---

## Definição de Pronto (DoD)

- [ ] ADR documentando estratégia de caching
- [ ] DrawIO com 4 views (multi-layer, cache-aside, CDN, invalidation)
- [ ] Go: LRU + LFU + multi-layer + singleflight + CDN sim + tests + bench
- [ ] Java: LRU + LFU + multi-layer + Redis + CDN sim + tests + metrics
- [ ] Experimento de performance documentado com resultados
- [ ] Commit: `feat(system-design-02): multi-layer cache and CDN simulator`
