# Level 8 — Resilience Patterns: Rate Limiting & Circuit Breaker

> **Objetivo:** Implementar Rate Limiter (token bucket, sliding window, fixed window)
> e Circuit Breaker com estados e recovery automático.

**Referência:**
- [16-rate-limiting.md](../../.docs/SYSTEM-DESIGN/16-rate-limiting.md)
- [17-circuit-breaker.md](../../.docs/SYSTEM-DESIGN/17-circuit-breaker.md)

**Pré-requisito:** Level 7 completo.

---

## Parte 1 — ADR: Resilience Strategy

**Arquivo:** `docs/adrs/ADR-001-resilience-strategy.md`

**Decisões:**
1. Algoritmo de rate limiting
2. Configuração do circuit breaker (thresholds, timeouts)
3. Rate limiting centralizado vs distribuído

**Options — Rate Limiting:**
1. **Token Bucket** — tokens regenerados a taxa fixa
2. **Leaky Bucket** — processa a taxa constante
3. **Fixed Window Counter** — conta por janela fixa de tempo
4. **Sliding Window Log** — log de timestamps
5. **Sliding Window Counter** — híbrido fixed + sliding

**Options — Circuit Breaker:**
1. **Count-based** — abre após N falhas consecutivas
2. **Time-based** — abre se error rate > X% em janela de tempo
3. **Mixed** — combina count e time

**Critérios de aceite:**
- [ ] 5 algoritmos de rate limiting comparados
- [ ] Circuit breaker states (closed → open → half-open) documentados
- [ ] Rate limiting distribuído (Redis) vs local
- [ ] Configurações recomendadas por tipo de serviço

---

## Parte 2 — Diagrama DrawIO

**Arquivo:** `docs/diagrams/08-resilience-patterns.drawio`

**View 1 — Rate Limiting Algorithms:** Visualização dos 5 algoritmos lado a lado

**View 2 — Circuit Breaker State Machine:**
```
    ┌────────────────────────────────────────┐
    │                                        │
    ▼                                        │
┌────────┐  failure threshold  ┌──────────┐  │  success threshold
│ CLOSED │────────────────────▶│   OPEN   │──┤
│(normal)│                     │(blocking)│  │
└────────┘                     └────┬─────┘  │
    ▲                               │        │
    │          timeout expired      │        │
    │                          ┌────▼─────┐  │
    │    success               │HALF-OPEN │──┘
    └──────────────────────────│(testing) │ failure
                               └──────────┘
```

**View 3 — Distributed Rate Limiting:** Redis + sliding window com múltiplas instâncias

**Critérios de aceite:**
- [ ] 5 algoritmos visualizados
- [ ] State machine do circuit breaker
- [ ] Rate limiting distribuído com Redis

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/
│   ├── server/main.go
│   └── loadtest/main.go           ← Load test client
├── internal/
│   ├── ratelimit/
│   │   ├── limiter.go             ← Interface RateLimiter
│   │   ├── tokenbucket.go         ← Token Bucket
│   │   ├── leakybucket.go         ← Leaky Bucket
│   │   ├── fixedwindow.go         ← Fixed Window Counter
│   │   ├── slidinglog.go          ← Sliding Window Log
│   │   ├── slidingcounter.go      ← Sliding Window Counter
│   │   ├── distributed.go         ← Redis-based distributed limiter
│   │   └── limiter_test.go
│   ├── circuitbreaker/
│   │   ├── breaker.go             ← Circuit Breaker
│   │   ├── state.go               ← State machine
│   │   ├── metrics.go             ← Failure/success tracking
│   │   └── breaker_test.go
│   ├── middleware/
│   │   ├── ratelimit.go           ← HTTP middleware
│   │   └── circuitbreaker.go      ← HTTP middleware
│   └── config/
│       └── config.go
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **5 Rate Limiting algorithms** implementados do zero
2. **Distributed Rate Limiter** usando Redis Lua scripts (atomic)
3. **Circuit Breaker** com state machine completa
4. **Per-client** e **per-endpoint** rate limiting
5. **HTTP middleware** para ambos os patterns
6. **Metrics** (requests allowed/denied, circuit state, error rate)
7. **Configuration** hot reload
8. **Exponential backoff** no circuit breaker half-open state
9. **Rate limit headers** (X-RateLimit-Limit, X-RateLimit-Remaining, Retry-After)
10. **Graceful degradation** (fallback responses quando circuit open)

**Critérios de aceite Go:**
- [ ] 5 rate limiting algorithms passando testes de taxa
- [ ] Token Bucket: permite burst, rate estável no longo prazo
- [ ] Sliding Window: sem boundary issues como Fixed Window
- [ ] Distributed limiter com Redis Lua scripts (atomic operations)
- [ ] Circuit Breaker: transições corretas (closed→open→half-open→closed)
- [ ] Rate limit HTTP headers presentes nas respostas
- [ ] Load test provando que rate limiting funciona sob carga
- [ ] ≥ 25 testes (unit + integration)
- [ ] Benchmarks para cada algoritmo

---

### 3.2 — Java

**Funcionalidades Java:**
1. **5 algorithms** como `@Component` beans
2. **Resilience4j** integration para circuit breaker
3. **Redis-based** distributed limiter com Spring Data Redis
4. **Spring AOP** para aplicar rate limiting via annotations
5. **Custom annotation** `@RateLimit(requests=100, window="1m")`
6. **Micrometer** metrics para rate limiting e circuit breaker
7. **Records** para rate limit configs

**Critérios de aceite Java:**
- [ ] 5 algorithms implementados
- [ ] Custom annotation `@RateLimit`
- [ ] Resilience4j circuit breaker
- [ ] Redis distributed limiter
- [ ] Testes com Testcontainers (Redis)
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] ADR com estratégia de resiliência
- [ ] DrawIO com 3 views
- [ ] Go e Java: 5 rate limiters + circuit breaker + distributed + tests
- [ ] Load test report provando eficácia
- [ ] Commit: `feat(system-design-08): rate limiting and circuit breaker`
