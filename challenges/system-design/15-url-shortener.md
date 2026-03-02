# Level 15 — URL Shortener (TinyURL)

> **Objetivo:** Projetar e implementar um URL Shortener completo — do design à implementação —
> com ADRs documentando cada decisão arquitetural e diagramas DrawIO da arquitetura.

**Referência:** [36-url-shortener.md](../../.docs/SYSTEM-DESIGN/36-url-shortener.md)

**Pré-requisito:** Levels 0-14 completos (building blocks).

---

## Contexto

Sistema que converte URLs longas em URLs curtas únicas com redirecionamento. Escala alvo:
- **100M URLs/dia** criadas
- **Read:Write ratio** = 100:1
- **p99 latência redirect** < 50ms

---

## Parte 1 — ADRs (3 obrigatórios)

### ADR-001: Short URL ID Generation

**Arquivo:** `docs/adrs/ADR-001-id-generation-strategy.md`

**Options:**
1. **Base62 encoding de auto-increment ID** — sequencial, previsível
2. **MD5/SHA256 hash + truncate** — collision-prone
3. **Pre-generated ID pool** (counter service / Snowflake) — distributed, unique
4. **UUID v7** — time-ordered, globally unique
5. **Base62 de timestamp + random** — time-sortable

### ADR-002: Storage Design

**Arquivo:** `docs/adrs/ADR-002-storage-design.md`

**Options:**
1. **PostgreSQL** — ACID, mature, rich queries
2. **DynamoDB** — key-value optimized, serverless
3. **Cassandra** — write-heavy optimized, horizontally scalable
4. **Redis** — in-memory, ultra-fast reads

### ADR-003: Caching Strategy

**Arquivo:** `docs/adrs/ADR-003-caching-strategy.md`

**Options:**
1. **Redis cache-aside** com TTL
2. **Local cache + Redis** (multi-layer)
3. **CDN caching** para URLs populares
4. **Write-through cache**

**Critérios de aceite:**
- [ ] 3 ADRs completos seguindo MADR
- [ ] Back-of-the-envelope estimation incluído
- [ ] Cada ADR com ≥ 3 opções documentadas
- [ ] Trade-offs quantificados (latência, custo, complexidade)

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

**Arquivo 1:** `docs/diagrams/15-url-shortener-hld.drawio` (High-Level Design)

```
┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Clients │───▶│   CDN    │───▶│   Load   │───▶│   API    │
│         │    │(hot URLs)│    │ Balancer │    │ Servers  │
└─────────┘    └──────────┘    └──────────┘    └────┬─────┘
                                                    │
                                          ┌─────────┼─────────┐
                                          │         │         │
                                     ┌────▼──┐ ┌───▼───┐ ┌───▼───┐
                                     │ Redis │ │Counter│ │  DB   │
                                     │ Cache │ │Service│ │(Write)│
                                     └───────┘ │(ID   │ └───┬───┘
                                               │ gen) │     │
                                               └──────┘  ┌──▼──┐
                                                         │ DB  │
                                                         │(Read│
                                                         │Repl)│
                                                         └─────┘
```

**Arquivo 2:** `docs/diagrams/15-url-shortener-sequence.drawio` (Sequence Diagrams)
- Create Short URL (happy path)
- Redirect (cache hit vs miss)
- Custom alias (conflict check)
- Analytics tracking

**Critérios de aceite:**
- [ ] HLD com todos os componentes
- [ ] Sequence diagrams para 4 fluxos
- [ ] Latências anotadas
- [ ] Scaling annotations (replicas, partitions)

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/api/main.go
├── internal/
│   ├── domain/
│   │   ├── url.go                 ← URL entity
│   │   ├── analytics.go           ← Click analytics
│   │   └── errors.go
│   ├── shortener/
│   │   ├── service.go             ← URL shortening service
│   │   ├── generator.go           ← ID generator (base62)
│   │   ├── validator.go           ← URL validation
│   │   └── service_test.go
│   ├── repository/
│   │   ├── repository.go          ← Interface
│   │   ├── postgres.go            ← PostgreSQL impl
│   │   └── redis_cache.go         ← Redis cache layer
│   ├── handler/
│   │   ├── shorten.go             ← POST /api/v1/shorten
│   │   ├── redirect.go            ← GET /:shortCode (301/302)
│   │   ├── analytics.go           ← GET /api/v1/analytics/:code
│   │   └── handler_test.go
│   ├── analytics/
│   │   ├── collector.go           ← Async click tracking
│   │   ├── aggregator.go          ← Click aggregation
│   │   └── collector_test.go
│   └── middleware/
│       ├── ratelimit.go
│       └── logging.go
├── migrations/
│   └── 001_create_urls.sql
├── docker-compose.yml
├── go.mod
└── Makefile
```

**Endpoints:**
```
POST /api/v1/shorten        → { "long_url": "...", "custom_alias": "...", "expires_at": "..." }
GET  /:shortCode             → 301 Redirect to long URL
GET  /api/v1/analytics/:code → { "clicks": N, "by_country": {...}, "by_referrer": {...} }
DELETE /api/v1/urls/:code    → Soft delete URL
```

**Funcionalidades Go:**
1. **Base62 Encoding** de counter ID (Snowflake-style)
2. **Custom aliases** com uniqueness check
3. **URL expiration** com TTL
4. **301 vs 302 redirect** (configurable per URL)
5. **Redis cache** para hot URLs (cache-aside)
6. **Async analytics** com goroutine + channel (não bloquear redirect)
7. **Click tracking** com geo e referrer (IP → Country via GeoIP)
8. **Rate limiting** per IP
9. **Bulk shortening** endpoint
10. **Health check** e readiness probe

**Critérios de aceite Go:**
- [ ] Shorten + redirect funcionando end-to-end
- [ ] Base62: IDs únicos e curtos (7 chars)
- [ ] Custom alias com conflict detection
- [ ] Cache hit rate ≥ 80% em benchmark
- [ ] Analytics: contagem de cliques (async, non-blocking)
- [ ] URL expiration funcional
- [ ] Rate limiting: 100 req/min per IP
- [ ] Benchmark: redirect p99 < 20ms (com cache hit)
- [ ] ≥ 20 testes (unit + integration com Testcontainers)
- [ ] Docker Compose: app + postgres + redis

---

### 3.2 — Java (Spring Boot)

**Funcionalidades Java:**
1. **Spring Boot 3.x** com WebFlux (reactive)
2. **Spring Data JPA** + PostgreSQL
3. **Spring Data Redis** para cache
4. **Spring Scheduler** para analytics aggregation
5. **Records** para request/response DTOs
6. **Virtual Threads** para I/O
7. **Micrometer** metrics (create rate, redirect rate, cache hit ratio)
8. **Testcontainers** para testes de integração

**Critérios de aceite Java:**
- [ ] Mesmas funcionalidades do Go
- [ ] Spring WebFlux reactive stack
- [ ] Spring Data Redis cache
- [ ] Actuator metrics
- [ ] Testes com Testcontainers
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] 3 ADRs documentando ID generation, storage e caching
- [ ] 2 DrawIO diagrams (HLD + Sequence)
- [ ] Go: implementação completa + tests + benchmark
- [ ] Java: implementação completa + tests + metrics
- [ ] Docker Compose funcional
- [ ] Commit: `feat(system-design-15): url shortener`
