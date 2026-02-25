# Performance Engineering — Latency Budgets e Performance SLOs

> **Objetivo deste documento:** Servir como referência completa sobre **latency budgets, performance SLOs/SLIs, tail latency management e otimização end-to-end** — como definir, alocar, monitorar e proteger budgets de latência em sistemas distribuídos, otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: latency budget design, alocação por componente, tail latency, fan-out problem, performance SLOs, error budgets, otimização de latência.

---

## Quick Reference — Cheat Sheet

| Conceito | Definição | Exemplo |
|----------|----------|---------|
| **Latency budget** | Tempo total alocado para uma operação end-to-end | /checkout: 500ms total |
| **Performance SLO** | Objetivo mensurável de performance | p99 < 500ms para 99.9% das requests |
| **Performance SLI** | Métrica que alimenta o SLO | p99 latency do endpoint /checkout |
| **Error budget** | Margem de violação tolerada do SLO | 0.1% das requests podem exceder SLO |
| **Tail latency** | Latência nos percentis mais altos (p99+) | p99.9 = 2s (10x o p50) |
| **Fan-out** | Request que depende de N chamadas paralelas | p99 piora: 1-(1-p99)^N |
| **Latency tax** | Overhead fixo por camada (TLS, serialization...) | ~5-15ms por hop |

---

## Sumário

- [Performance Engineering — Latency Budgets e Performance SLOs](#performance-engineering--latency-budgets-e-performance-slos)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que são Latency Budgets](#o-que-são-latency-budgets)
  - [Definindo Performance SLOs](#definindo-performance-slos)
  - [Alocação de Latency Budget](#alocação-de-latency-budget)
  - [Tail Latency](#tail-latency)
  - [Fan-Out Problem](#fan-out-problem)
  - [Latency Optimization Techniques](#latency-optimization-techniques)
  - [Monitoramento e Alerting](#monitoramento-e-alerting)
  - [Latency Budget Governance](#latency-budget-governance)
  - [Case Studies](#case-studies)
  - [Anti-Patterns de Latency Management](#anti-patterns-de-latency-management)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que são Latency Budgets

```
LATENCY BUDGET = TEMPO TOTAL ALOCADO PARA UMA OPERAÇÃO END-TO-END

  Analogy: ORÇAMENTO FINANCEIRO
  → Receita (SLO): $1000/mês para alimentação
  → Despesas alocadas: supermercado $500, restaurante $300, delivery $200
  → Se restaurante gasta $600: estoura o budget → cortar em outro lugar

  LATENCY BUDGET:
  → Budget total (SLO): 500ms para /checkout (p99)
  → Alocação:
    - API Gateway + LB:    20ms
    - Order Service:       50ms
    - Payment Service:     150ms
    - Inventory Service:   80ms
    - Database queries:    100ms
    - Network hops (4×):   40ms
    - Serialization:       30ms
    - Buffer/headroom:     30ms
    ─────────────────────────────
    TOTAL:                 500ms ✅

  ┌──────────────────────────── 500ms total ────────────────────────────┐
  │                                                                      │
  │  ┌──┐ ┌─────┐ ┌──────────┐ ┌──────┐ ┌────────┐ ┌──┐ ┌──┐ ┌──┐    │
  │  │GW│ │Order│ │ Payment  │ │Inven-│ │Database│ │Net│ │Ser│ │Buf│    │
  │  │20│ │ 50  │ │   150    │ │tory  │ │  100   │ │40 │ │30 │ │30 │    │
  │  │ms│ │ ms  │ │   ms     │ │80 ms │ │  ms    │ │ms │ │ms │ │ms │    │
  │  └──┘ └─────┘ └──────────┘ └──────┘ └────────┘ └──┘ └──┘ └──┘    │
  │                                                                      │
  └──────────────────────────────────────────────────────────────────────┘

  POR QUE LATENCY BUDGETS:

  1. DECOMPOSIÇÃO: transformar SLO vago em targets por equipe
  2. OWNERSHIP: cada equipe sabe seu limite
  3. TRADE-OFFS: adicionar feature? Precisa "caber" no budget
  4. DETECTION: se um componente estourar, afeta o SLO total
  5. NEGOTIATION: "preciso de 50ms a mais" → de onde vem?
```

---

## Definindo Performance SLOs

### SLI → SLO → Error Budget

```
PERFORMANCE SLI, SLO E ERROR BUDGET:

  SLI (Service Level Indicator)
  → MÉTRICA mensurável de performance
  → Exemplo: "latência p99 do endpoint /checkout"
  → Medido continuamente em produção

  SLO (Service Level Objective)
  → TARGET para o SLI
  → Exemplo: "p99 latency of /checkout < 500ms"
  → Com window: "99.9% of requests in 30-day window"

  ERROR BUDGET
  → MARGEM permitida de violação
  → "99.9% dentro do SLO" = 0.1% pode violar
  → 30 dias × 24h × 60min × 60s = 2,592,000s
  → 0.1% = 2,592 requests podem violar em 30 dias
  → (assumindo 1 req/s; para 1000 req/s = 2,592,000 violations)

  FRAMEWORK DE PERFORMANCE SLO:

  │ Endpoint   │ SLI           │ SLO          │ Window │ Error Budget │
  │ /checkout  │ p99 latency   │ < 500ms      │ 30 day │ 99.9%        │
  │ /catalog   │ p95 latency   │ < 100ms      │ 30 day │ 99.95%       │
  │ /search    │ p95 latency   │ < 200ms      │ 30 day │ 99.9%        │
  │ /health    │ p99 latency   │ < 50ms       │ 30 day │ 99.99%       │
  │ All        │ error rate    │ < 0.1%       │ 30 day │ 99.9%        │
  │ All        │ throughput    │ ≥ 5,000 rps  │ 1 hour │ 99.5%        │
```

### Como escolher SLOs de performance

```
COMO DEFINIR PERFORMANCE SLOs:

  1. COMECE PELO USUÁRIO
     → Qual latência o usuário percebe como "rápido"?
     → Web: < 200ms = instantâneo, < 1s = aceitável, > 3s = abandono
     → Mobile: tolera mais (network variável)
     → API B2B: SLA contratual (e.g., p99 < 500ms)

  2. MEÇA O BASELINE ATUAL
     → Qual é o p50, p95, p99, p99.9 HOJE?
     → Se p99 hoje é 800ms, SLO de 100ms é irreal
     → SLO deve ser: baseline + margem de melhoria razoável

  3. DIFERENCIE POR CRITICIDADE
     → Checkout: SLO agressivo (revenue-impacting)
     → Admin panel: SLO relaxado (internal users)
     → Background jobs: sem SLO de latência (throughput SLO)

  4. PERCENTILE CHOICE
     
     │ Percentile │ Quando usar                              │
     │ p50        │ Experiência "típica" do usuário           │
     │ p95        │ SLO padrão para maioria dos endpoints    │
     │ p99        │ SLO para endpoints críticos              │
     │ p99.9      │ SLO para sistemas de alta criticidade    │

  5. ITERAR
     → SLO apertado demais → queima error budget rápido → fadiga
     → SLO frouxo demais → usuários sofrem sem alerta
     → Revise trimestralmente baseado em dados

  NÃO FAÇA:
  → SLO = "availability 99.99%" sem medir (vanity SLO)
  → SLO baseado em média (esconde tail latency)
  → SLO sem medição automatizada (SLO manual = mentira)
  → SLO sem consequence (se violar, e daí?)
```

---

## Alocação de Latency Budget

### Metodologia de alocação

```
COMO ALOCAR LATENCY BUDGET:

  STEP 1: DEFINIR BUDGET TOTAL (do SLO)
  → /checkout SLO: p99 < 500ms
  → Budget total: 500ms

  STEP 2: MAPEAR CALL CHAIN
  
  User → CDN → LB → API GW → Order Service → Payment API
                                    │
                                    ├──→ Inventory Service → Cache/DB
                                    │
                                    └──→ Notification Service (async)

  STEP 3: IDENTIFICAR OVERHEAD FIXO (latency tax)
  
  │ Componente              │ Overhead (típico)     │
  │ CDN                     │ 2-10ms                │
  │ Load Balancer           │ 1-5ms                 │
  │ API Gateway             │ 5-20ms                │
  │ TLS handshake           │ 10-50ms (primeiro req)│
  │ Network hop (intra-AZ)  │ 0.5-2ms               │
  │ Network hop (cross-AZ)  │ 1-5ms                 │
  │ Network hop (cross-region)│ 50-150ms             │
  │ JSON serialization      │ 1-10ms                │
  │ Protobuf serialization  │ 0.1-2ms               │
  │ Auth/JWT validation     │ 1-5ms                 │

  Total overhead fixo estimado: ~40ms

  STEP 4: ALOCAR BUDGET RESTANTE
  
  Budget disponível: 500ms - 40ms overhead = 460ms
  
  Alocação proporcional à complexidade:
  │ Componente        │ Budget │ Justificativa                     │
  │ Order Service     │ 80ms   │ Lógica de negócio + validação     │
  │ Payment API       │ 180ms  │ External API, mais lento          │
  │ Inventory Service │ 80ms   │ Cache hit: 5ms, miss: 80ms       │
  │ DB queries (total)│ 90ms   │ 2-3 queries × 30ms               │
  │ Headroom          │ 30ms   │ Para absorver variance/jitter    │
  │ ────────────────  │ ─────  │                                   │
  │ TOTAL             │ 460ms  │                                   │

  STEP 5: COMUNICAR E MONITORAR
  → Cada equipe conhece SEU budget
  → Dashboard: "quanto do budget cada componente usa?"
  → Alerta: componente usando > 80% do budget
```

### Budget de path paralelo vs sequencial

```
PARALELO vs SEQUENCIAL AFETA O BUDGET:

  SEQUENCIAL:
  A → B → C
  Total = A + B + C = 100 + 150 + 100 = 350ms

  PARALELO:
  A → (B ∥ C) → D
  Total = A + max(B, C) + D = 100 + max(150, 100) + 50 = 300ms

  REGRA: PARALELIZE QUANDO POSSÍVEL
  → Reduz latência total
  → Mas: complexidade de error handling
  → Fan-out aumenta tail latency (ver seção específica)

  EXEMPLO — Order Checkout:

  ANTES (sequencial):
  validate(50ms) → checkInventory(80ms) → processPayment(150ms) 
    → sendConfirmation(30ms)
  Total: 310ms

  DEPOIS (paralelo onde possível):
  validate(50ms) → [checkInventory(80ms) ∥ reservePayment(60ms)] 
    → confirmPayment(90ms) → sendConfirmation(async)
  Total: 50 + max(80, 60) + 90 = 220ms (29% reduction)
  
  E notificação é ASSÍNCRONA (não afeta latência do usuário)
```

---

## Tail Latency

### Por que tail latency importa

```
TAIL LATENCY — POR QUE p99/p99.9 DOMINAM:

  "JEFF DEAN NUMBERS":
  
  │ Percentile │ Latência │ Impact at Scale               │
  │ p50        │ 10ms     │ 1 em 2 requests               │
  │ p99        │ 100ms    │ 1 em 100 requests             │
  │ p99.9      │ 500ms    │ 1 em 1,000 requests           │
  │ p99.99     │ 2,000ms  │ 1 em 10,000 requests          │

  A 1,000,000 req/s:
  → p99.9 (500ms) afeta 1,000 requests POR SEGUNDO
  → Em 1 minuto: 60,000 requests com latência de 500ms+
  → Em 1 hora: 3,600,000 requests com experiência degradada

  CAUSAS COMUNS DE TAIL LATENCY:

  1. GARBAGE COLLECTION
     → Full GC: 200ms-2s
     → Predictable: ocorre quando heap enche
     → Mitigação: GC tuning, ZGC, allocation reduction

  2. CONTENTION
     → Lock contention: threads esperam lock
     → Connection pool exhaustion: esperam conexão
     → Mitigação: lock-free structures, pool sizing

  3. NETWORK VARIABILITY
     → TCP retransmit: +200ms (RTO padrão)
     → DNS resolution: +50-200ms (cache miss)
     → Cross-AZ: +1-5ms (mas consistente)
     → Mitigação: DNS caching, keep-alive, retry

  4. RESOURCE INTERFERENCE (Noisy Neighbor)
     → Shared CPU: outro pod/VM saturando
     → Disk I/O: burst de outro tenant
     → Mitigação: dedicated instances, resource isolation

  5. QUEUE DEPTH
     → Request entra em queue quando sistema saturado
     → Queuing delay: (utilization / (1-utilization)) × service_time
     → 80% util → 4x service time em queue
     → Mitigação: load shedding, backpressure
```

### Técnicas para reduzir tail latency

```
TÉCNICAS PARA REDUZIR TAIL LATENCY:

  1. HEDGED REQUESTS (Google, Jeff Dean)
     → Enviar request para 2 réplicas simultaneamente
     → Usar primeira resposta, cancelar segunda
     → Tail: max(p99_A, p99_B) → min(A, B) ≈ p95
     → Custo: ~2% mais requests (95% não precisa hedge)
     
     Implementação:
     → Enviar request normal
     → Se não respondeu em p95 (e.g., 50ms):
       enviar segunda request para outra réplica
     → Usar primeira resposta
     
     ⚠️  Só funciona para operações IDEMPOTENTES (reads)

  2. TIED REQUESTS
     → Similar a hedged, mas coordenado
     → Enviar para 2 réplicas, primeira que iniciar cancela outra
     → Reduz desperdício vs hedged puro

  3. SELECTIVE RETRY
     → Se tempo em flight > p95: retry em outra réplica
     → Timeout agressivo + retry rápido
     → Budget de retry: max 1 retry por request

  4. LOAD SHEDDING
     → Se queue depth > threshold: rejeitar request imediatamente
     → 503 rápido é MELHOR que timeout lento
     → Priority-based: rejeitar requests de menor prioridade
     
     if queue.size() > MAX_QUEUE {
         return 503, "Service Overloaded"
     }

  5. DEADLINE PROPAGATION
     → Propagar deadline (timeout) ao longo do call chain
     → Se deadline restante < expected time: fail fast
     
     // gRPC deadline propagation
     deadline = incoming_request.deadline() - elapsed
     if deadline < 0 { return DEADLINE_EXCEEDED }
     outgoing_call.set_deadline(deadline)

  6. ADAPTIVE CONCURRENCY LIMITS
     → Ajustar limite de requests in-flight dinamicamente
     → Se latência cresce: REDUZIR concurrency
     → Netflix Concurrency Limiter: 
       https://github.com/Netflix/concurrency-limits
```

---

## Fan-Out Problem

```
FAN-OUT AMPLIFIES TAIL LATENCY:

  Uma request que chama N serviços em paralelo:
  → Response time = max(service_1, service_2, ..., service_N)
  → p99 da request = probabilidade de pelo menos 1 ser lento

  MATH:
  P(all fast) = (1 - p_slow)^N
  P(at least one slow) = 1 - (1 - p_slow)^N

  │ Fan-out │ p99 de cada │ P(request > p99) │ Effective percentile │
  │ N=1     │ 1%          │ 1.0%             │ p99                  │
  │ N=5     │ 1%          │ 4.9%             │ ~p95                 │
  │ N=10    │ 1%          │ 9.6%             │ ~p90                 │
  │ N=25    │ 1%          │ 22.2%            │ ~p78                 │
  │ N=50    │ 1%          │ 39.5%            │ ~p60                 │
  │ N=100   │ 1%          │ 63.4%            │ ~p37                 │

  COM N=50: 40% das requests TEM pelo menos um serviço no p99
  → Tail latency dos backends vira COMMON case do frontend!

  ILUSTRAÇÃO:

  Request                    Response
  ────────┐                 ┌────────
          │  ┌── Service 1: 10ms ──┐
          │  ├── Service 2: 12ms ──┤
          │  ├── Service 3: 8ms  ──┤
          ├──┤── Service 4: 450ms ──┤──► Total: 450ms (max)
          │  ├── Service 5: 11ms ──┤
          │  └── ...               ┘
  ────────┘

  Service 4 tendo um p99 moment (GC?) → TODA request é 450ms

  MITIGAÇÕES:

  1. REDUZIR FAN-OUT
     → Batch requests (1 call com N items vs N calls)
     → Denormalize: dados locais em vez de consultar
     → Async: nem tudo precisa ser síncrono

  2. HEDGED REQUESTS (já descrito)
     → Para os N backends individualmente
     → Retry seletivo no mais lento

  3. PARTIAL RESULTS
     → Timeout curto: se 1 de N falha, retornar parcial
     → "49 de 50 resultados" é melhor que 0
     → Degradação graceful

  4. REDUCE BACKEND p99
     → Se p99 de cada backend é 10ms (em vez de 100ms):
     → Fan-out 50 com p99=10ms: P(slow) = 39.5%
       MAS "slow" = 10ms, não 100ms → impacto menor
     → Investir em tail latency dos backends
```

---

## Latency Optimization Techniques

### Por camada

```
OTIMIZAÇÕES POR CAMADA:

  ┌──────────────────────────────────────────────────────────┐
  │ CAMADA: NETWORK                                           │
  │                                                           │
  │ • Connection pooling (evitar handshake repetido)          │
  │   - HTTP/2 multiplexing (1 connection, N streams)         │
  │   - gRPC: connection reuse nativo                         │
  │                                                           │
  │ • DNS caching (evitar resolution em cada request)         │
  │   - JVM: networkaddress.cache.ttl=30                      │
  │                                                           │
  │ • Keep-alive (evitar TCP handshake + TLS negotiation)     │
  │   - TCP keepalive: net.ipv4.tcp_keepalive_time=60         │
  │   - HTTP keepalive: Connection: keep-alive                │
  │                                                           │
  │ • Compression (reduzir bytes transferidos)                │
  │   - gzip para JSON payloads                               │
  │   - Protobuf/FlatBuffers para serialização binária        │
  │                                                           │
  │ • Proximity (reduzir RTT)                                 │
  │   - CDN para assets e APIs edge                           │
  │   - Multi-region para latência global                     │
  │   - Same-AZ para serviços frequentemente chamados         │
  └──────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────┐
  │ CAMADA: APPLICATION                                       │
  │                                                           │
  │ • Caching (evitar recomputation/refetch)                  │
  │   - Local cache (Caffeine, Guava): < 1ms                  │
  │   - Distributed cache (Redis): 1-5ms                      │
  │   - CDN cache: 0ms (edge)                                 │
  │   - Cache invalidation: TTL, event-driven, write-through  │
  │                                                           │
  │ • Async processing (remover do critical path)             │
  │   - Email/notification → queue (async)                    │
  │   - Analytics/logging → fire-and-forget                   │
  │   - Eventual consistency onde aceitável                   │
  │                                                           │
  │ • Parallelism (reduzir wall-clock time)                   │
  │   - CompletableFuture (Java), goroutines (Go)             │
  │   - Promise.all (Node.js)                                 │
  │   - Cuidado: fan-out amplifica tail latency               │
  │                                                           │
  │ • Batch operations (reduzir overhead per-item)            │
  │   - Batch DB queries (IN clause vs N queries)             │
  │   - Batch external API calls                              │
  │   - DataLoader pattern (GraphQL)                          │
  │                                                           │
  │ • Efficient serialization                                 │
  │   - Protobuf > JSON para inter-service                    │
  │   - Evitar reflection-based serializers em hot path       │
  │   - Pre-compile schemas quando possível                   │
  └──────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────┐
  │ CAMADA: DATABASE                                          │
  │                                                           │
  │ • Query optimization                                      │
  │   - EXPLAIN ANALYZE antes de deploy                       │
  │   - Indexes para queries frequentes                       │
  │   - Avoid SELECT * (transfere dados desnecessários)       │
  │                                                           │
  │ • Connection pooling                                      │
  │   - PgBouncer / ProxySQL                                 │
  │   - Pool size via Little's Law                            │
  │                                                           │
  │ • Read replicas                                           │
  │   - Direcionar reads para replicas                        │
  │   - Accepts eventual consistency lag                      │
  │                                                           │
  │ • Caching layer                                           │
  │   - Redis/Memcached para hot data                        │
  │   - Write-through ou cache-aside pattern                  │
  │   - Cache hit ratio target > 90%                          │
  │                                                           │
  │ • Sharding / Partitioning                                 │
  │   - Horizontal: distribuir por key                        │
  │   - Temporal: partition por data (archive old)            │
  └──────────────────────────────────────────────────────────┘
```

### Latency budget para common operations

```
REFERENCE LATENCY NUMBERS (ATUALIZADO):

  (Adaptado de Jeff Dean, Peter Norvig, e medições modernas)

  │ Operação                          │ Latência típica     │
  │ L1 cache reference                │ 0.5 ns              │
  │ Branch mispredict                 │ 5 ns                │
  │ L2 cache reference                │ 7 ns                │
  │ Mutex lock/unlock                 │ 25 ns               │
  │ Main memory reference             │ 100 ns              │
  │ Compress 1KB (Snappy)             │ 3,000 ns = 3 µs     │
  │ Send 1KB over 1 Gbps network     │ 10,000 ns = 10 µs   │
  │ Read 4KB randomly from SSD       │ 150,000 ns = 150 µs │
  │ Read 1MB sequentially from SSD   │ 1,000,000 ns = 1 ms │
  │ Same datacenter roundtrip        │ 500,000 ns = 0.5 ms │
  │ Redis GET (same AZ)              │ 200-500 µs          │
  │ PostgreSQL simple query (indexed) │ 1-5 ms              │
  │ Read 1MB sequentially from disk  │ 20 ms               │
  │ Internet: Brasil → US roundtrip  │ 120-180 ms          │
  │ Internet: US → Europe roundtrip  │ 70-120 ms           │
  │ TLS handshake (full)             │ 30-100 ms           │
  │ DNS resolution (uncached)        │ 20-120 ms           │

  USE ESTES NÚMEROS PARA:
  → Estimar latency budget por componente
  → Identificar dominantes (se DB roundtrip = 5ms,
    otimizar serialização de 1ms→0.5ms é irrelevante)
  → Detectar anomalias ("Redis está em 50ms? Normal é <1ms")
```

---

## Monitoramento e Alerting

### Dashboard de latency budget

```
DASHBOARD DE LATENCY BUDGET:

  ┌─── Latency Budget: /checkout (SLO: p99 < 500ms) ───────────────┐
  │                                                                  │
  │  Component          Budget   Actual p99   Status   % Used       │
  │  ──────────────────────────────────────────────────────────      │
  │  API Gateway         20ms      15ms        ✅       75%         │
  │  Order Service       80ms      65ms        ✅       81%         │
  │  Payment Service    180ms     170ms        ⚠️       94%  ←!     │
  │  Inventory Service   80ms      45ms        ✅       56%         │
  │  DB Queries         100ms      80ms        ✅       80%         │
  │  Network Overhead    40ms      35ms        ✅       88%         │
  │  ──────────────────────────────────────────────────────────      │
  │  TOTAL              500ms     410ms        ✅       82%         │
  │  Headroom                      90ms                             │
  │                                                                  │
  │  ⚠️  Payment Service: 94% do budget — TRENDING UP               │
  │  Trend: +5ms/week — estourará budget em ~2 semanas              │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘

  ERROR BUDGET STATUS:

  ┌─── Error Budget: /checkout (30-day window) ─────────────────────┐
  │                                                                  │
  │  SLO: 99.9% of requests with p99 < 500ms                        │
  │  Window: Last 30 days                                            │
  │                                                                  │
  │  Total requests:      125,000,000                                │
  │  Budget (0.1%):           125,000   violations allowed           │
  │  Violations so far:       45,000                                 │
  │  Budget remaining:        80,000   (64% remaining)              │
  │  Days remaining:              18                                 │
  │  Burn rate:                2,500/day                             │
  │  Projected:            OK — will have 35,000 remaining ✅       │
  │                                                                  │
  │  ████████████████████░░░░░░░░░░ 64% budget remaining            │
  │                                                                  │
  └──────────────────────────────────────────────────────────────────┘
```

### Alerting strategy

```
ALERTING PARA LATENCY BUDGETS:

  MULTI-WINDOW, MULTI-BURN-RATE (Google SRE):

  │ Severidade │ Burn Rate │ Window      │ Ação                  │
  │ Critical   │ 10x       │ 5 min       │ Page (wake up)        │
  │ Warning    │ 5x        │ 30 min      │ Page (business hours) │
  │ Ticket     │ 2x        │ 6 hours     │ Create ticket         │
  │ Info       │ 1.5x      │ 24 hours    │ Dashboard alert       │

  Exemplo:
  → Budget: 125,000 violations em 30 dias = ~4,167/dia
  → Critical: burning 41,670/dia (10x) por 5 min → page
  → Warning: burning 20,835/dia (5x) por 30 min → page
  → Ticket: burning 8,334/dia (2x) por 6 horas → ticket

  PROMETHEUS ALERTING RULES (exemplo):

  # Latência p99 excedendo SLO
  - alert: LatencyBudgetExceeded
    expr: |
      histogram_quantile(0.99, 
        sum(rate(http_request_duration_seconds_bucket{
          endpoint="/checkout"
        }[5m])) by (le)
      ) > 0.5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "/checkout p99 latency exceeding 500ms SLO"

  # Error budget burn rate
  - alert: ErrorBudgetHighBurnRate
    expr: |
      (
        sum(rate(http_request_duration_seconds_count{
          endpoint="/checkout",
          le="0.5"
        }[1h]))
        /
        sum(rate(http_request_duration_seconds_count{
          endpoint="/checkout"
        }[1h]))
      ) < 0.999
    for: 30m
    labels:
      severity: critical
```

---

## Latency Budget Governance

```
GOVERNANCE — COMO MANTER BUDGETS SAUDÁVEIS:

  1. BUDGET REVIEW QUINZENAL/MENSAL
     → Dashboard review com tech leads
     → Trending: algum componente crescendo?
     → Ação: quem está > 80% do budget → action item

  2. CHANGE MANAGEMENT
     → Nova feature adiciona latência? Precisa "pagar" de algum lugar
     → Exemplo: "adds 30ms for fraud check"
       → Opção A: otimizar payment de 170ms → 140ms
       → Opção B: relaxar SLO de 500ms → 530ms (produto decide)
       → Opção C: async fraud check (não bloqueante)

  3. REGRESSION GATE EM CI/CD
     → Load test em staging: p99 increased > 10%?
     → Block deploy até investigar
     → Performance benchmark em cada PR

  4. BUDGET ALLOCATION PARA NOVOS SERVIÇOS
     → Novo microserviço no path? 
     → Precisa alocar budget + overhead de network hop
     → Template: "request para adicionar componente ao critical path"

  5. ANNUAL REVIEW
     → SLOs ainda fazem sentido?
     → Usuários mais exigentes? → apertar SLO
     → Complexidade aumentou? → talvez relaxar ou otimizar
     → Cost of latency: cada 100ms de latência = X% conversion drop

  LATENCY IMPACT NO NEGÓCIO (references):
  
  │ Empresa  │ Achado                                              │
  │ Amazon   │ 100ms extra latency → 1% queda em revenue           │
  │ Google   │ 500ms extra → 20% drop em searches                  │
  │ Walmart  │ Cada 100ms de melhoria → 1% incremento em revenue   │
  │ Akamai   │ 2s load time → 103% bounce rate increase            │
```

---

## Case Studies

### Case 1: E-commerce checkout

```
CASE STUDY: OTIMIZAÇÃO DE CHECKOUT

  ANTES (medido):
  /checkout p99 = 1.2s (SLO: 500ms) ❌

  BREAKDOWN (distributed trace):
  │ Component          │ p99     │ % Total │
  │ API Gateway        │ 18ms   │ 1.5%    │
  │ Auth/JWT           │ 8ms    │ 0.7%    │
  │ Order Validation   │ 35ms   │ 2.9%    │
  │ Inventory Check    │ 180ms  │ 15.0%   │ ← DB full scan
  │ Payment Processing │ 650ms  │ 54.2%   │ ← external API + retry
  │ Fraud Check        │ 220ms  │ 18.3%   │ ← síncrono (bloqueia)
  │ Send Confirmation  │ 45ms   │ 3.8%    │ ← síncrono (por quê?)
  │ Network/Overhead   │ 44ms   │ 3.7%    │
  │ TOTAL              │ 1200ms │ 100%    │

  AÇÕES:

  1. Inventory Check: ADD INDEX on product_id
     180ms → 12ms (save 168ms)

  2. Payment Processing: reduce timeout, add hedged request
     650ms → 320ms (save 330ms)

  3. Fraud Check: move to ASYNC (não bloqueia checkout)
     220ms → 0ms no critical path (save 220ms)

  4. Send Confirmation: move to ASYNC (event/queue)
     45ms → 0ms no critical path (save 45ms)

  DEPOIS:
  │ Component          │ p99    │
  │ API Gateway        │ 18ms  │
  │ Auth/JWT           │ 8ms   │
  │ Order Validation   │ 35ms  │
  │ Inventory Check    │ 12ms  │ ✅ indexed
  │ Payment Processing │ 320ms │ ✅ hedged
  │ Network/Overhead   │ 42ms  │
  │ TOTAL              │ 435ms │ ✅ (era 1200ms)

  Melhoria: 1200ms → 435ms (64% reduction)
  SLO p99 < 500ms: MET ✅
  Headroom: 65ms (13%)
```

### Case 2: Search API fan-out

```
CASE STUDY: SEARCH COM FAN-OUT

  PROBLEMA: /search p99 = 3s (SLO: 300ms)
  
  Arquitetura: search query → fan-out para 20 shards
  Cada shard: p99 = 200ms
  P(at least one shard > 200ms) = 1-(0.99)^20 = 18%
  → 18% das requests têm pelo menos 1 shard lento
  → Response = max(shards) → p99 da response >>> 200ms

  SOLUÇÃO:

  1. REDUCE SHARD p99: 200ms → 50ms
     → Profiling: GC pauses (ZGC), query optimization
     → Resultado: p99 individual 50ms

  2. PARTIAL RESULTS com timeout
     → Timeout: 100ms per shard
     → Se shard não responde em 100ms: usar partial results
     → Ex: 18/20 shards responderam em 100ms → retornar parcial

  3. HEDGED REQUESTS para shards lentos
     → Se shard não respondeu em p95 (40ms):
       → Enviar request para replica do shard
       → Usar primeira resposta

  RESULTADO:
  → p99 total: 3s → 120ms ✅
  → SLO 300ms: MET com 60% headroom
```

---

## Anti-Patterns de Latency Management

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **"Média é boa"** | p50=10ms esconde p99=2s | Monitorar e alertar em p99, p99.9 |
| **SLO sem medição** | "Nosso SLO é 200ms" — quem mede? | Medição automatizada, dashboards, alertas |
| **Latency budget informal** | "Cada serviço deve ser rápido" | Budget explícito por componente, documentado |
| **Sync everything** | Notification síncrona no checkout | Async para operações não-críticas |
| **Retry without budget** | Retry 3x com backoff = 3x latência | Deadline propagation, retry budget |
| **Ignorar fan-out** | "Cada shard é rápido" → request é lenta | Math: 1-(1-p)^N, hedging, partial results |
| **Timeout = infinity** | Sem timeout = threads bloqueadas para sempre | Timeout explícito em TODA chamada externa |
| **Cache sem invalidation** | Dados stale por horas/dias | TTL, invalidation explícita, stale-while-revalidate |
| **Otimizar o errado** | "Otimizar serialization" quando DB é 80% | Profile → identificar dominante → otimizar |
| **Log síncrono em hot path** | Console.log/logger.info no critical path | Async logging ou sampling em hot paths |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código com foco em latência e performance SLOs, verifique:

1. **Timeout ausente em chamada externa** — TODA chamada HTTP, gRPC, DB DEVE ter timeout explícito; sem timeout = risco de thread starvation
2. **Operação síncrona não-essencial no critical path** — Notificação, analytics, logging pesado devem ser async
3. **Fan-out sem proteção** — Chamadas paralelas devem ter: timeout individual, partial results, hedging ou combinação
4. **Retry sem deadline propagation** — Retries devem respeitar deadline total da request; não criar retry loops infinitos
5. **Cache sem TTL** — TODA key de cache precisa de TTL; cache sem expiração = dados stale
6. **N+1 queries** — Loop com DB query individual: substituir por batch/IN clause; usar DataLoader pattern
7. **SELECT * em queries** — Retornar apenas colunas necessárias; payload menor = menos serialization + network
8. **Connection pool hardcoded** — Pool size deve ser derivado de Little's Law; documentar cálculo
9. **Sem métricas de latência por endpoint** — Histograma de latência por endpoint é obrigatório para calcular SLIs
10. **Sem trace context propagation** — OpenTelemetry context deve ser propagado em chamadas inter-service
11. **DNS resolution não cacheado** — JVM: networkaddress.cache.ttl configurado; outras linguagens: resolver caching
12. **Serialização reflection-based em hot path** — Preferir serializers code-generated (Protobuf, pre-compiled Jackson) em paths de alta vazão

---

## Referências

- **The Tail at Scale** — Jeff Dean, Luiz André Barroso (Google, 2013) — Communications of the ACM
- **Site Reliability Engineering** — Google (O'Reilly) — Capítulos 4 (SLOs), 6 (Monitoring), 31 (Communication)
- **Implementing Service Level Objectives** — Alex Hidalgo (O'Reilly, 2020)
- **Release It!** — Michael Nygard (Pragmatic Bookshelf, 2018) — Stability patterns, timeouts
- **Systems Performance** — Brendan Gregg (Addison-Wesley, 2020) — Latency chapter
- **Jeff Dean's Latency Numbers** — https://gist.github.com/jboner/2841832
- **Netflix Concurrency Limiter** — https://github.com/Netflix/concurrency-limits
- **Google SRE Alerting on SLOs** — https://sre.google/workbook/alerting-on-slos/
- **Hedged Requests** — Jeff Dean — "Achieving Rapid Response Times in Large Online Services"
- **OpenTelemetry** — https://opentelemetry.io/ — Distributed tracing specification
- **Gil Tene — Coordinated Omission** — latency measurement accuracy
- **Amazon: Every 100ms costs 1% revenue** — Greg Linden, Amazon (2006)
