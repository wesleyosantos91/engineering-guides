# Level 17 — Twitter / Social Feed

> **Objetivo:** Projetar e implementar uma plataforma de micro-posting (estilo Twitter) com
> timeline feed, fan-out, trending topics e busca em tempo real.

**Referência:** [38-twitter-social-feed.md](../../.docs/SYSTEM-DESIGN/38-twitter-social-feed.md)

**Pré-requisito:** Level 16 completo.

---

## Contexto

Sistema onde usuários publicam posts curtos (280 chars), seguem outros usuários e
visualizam um feed personalizado. O desafio principal é a **fan-out** strategy para
construir timelines: push (fan-out on write) vs pull (fan-out on read).

**Escala alvo:**
- **300M usuários** ativos
- **600 tweets/segundo**
- **Average follows:** 200 por usuário
- **Celebridades:** até 50M seguidores
- **Timeline latência:** < 200ms

---

## Parte 1 — ADRs (4 obrigatórios)

### ADR-001: Fan-Out Strategy

**Arquivo:** `docs/adrs/ADR-001-fanout-strategy.md`

**Options:**
1. **Fan-out on Write (Push Model)** — pré-computa timeline de cada follower no write
2. **Fan-out on Read (Pull Model)** — monta timeline sob demanda no read
3. **Hybrid** — push para usuários normais, pull para celebridades (> 10K followers)

### ADR-002: Feed Storage Model

**Options:**
1. **Redis Sorted Set** — timeline pré-computada por user
2. **Cassandra/ScyllaDB** — wide column para timeline
3. **PostgreSQL materialized views** — views por user
4. **Redis + PostgreSQL** — hot timeline em Redis, cold em PG

### ADR-003: Tweet Storage & Search

**Options:**
1. **PostgreSQL** para tweets + Elasticsearch para busca
2. **Cassandra** para tweets + Solr para busca
3. **PostgreSQL** para tudo (FTS nativo)

### ADR-004: Real-time Delivery

**Options:**
1. **WebSocket** — conexão persistente para updates
2. **SSE** — server push para novos tweets
3. **Polling** — cliente puxa a cada N segundos
4. **Hybrid** — WebSocket para ativos, polling para inativos

**Critérios de aceite:**
- [ ] 4 ADRs completos
- [ ] Fan-out analysis com cálculos de write amplification
- [ ] Celebrity problem documentado

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

**Arquivo 1:** `docs/diagrams/17-twitter-hld.drawio`

```
                    ┌──────────────────────────────────────────┐
                    │            API Gateway / LB              │
                    └──────┬──────────┬──────────┬─────────────┘
                           │          │          │
                    ┌──────▼──┐ ┌─────▼────┐ ┌──▼─────────┐
                    │  Tweet  │ │ Timeline │ │   Search   │
                    │ Service │ │ Service  │ │  Service   │
                    └────┬────┘ └────┬─────┘ └──────┬─────┘
                         │          │               │
           ┌─────────────┼──────────┼───────────────┤
           │             │          │               │
      ┌────▼────┐  ┌─────▼──┐ ┌────▼────┐  ┌──────▼──────┐
      │ Tweet   │  │ Fan-out│ │ Redis   │  │Elasticsearch│
      │   DB    │  │ Worker │ │Timelines│  │   (Search)  │
      └─────────┘  └────┬───┘ └─────────┘  └─────────────┘
                        │
                   ┌────▼────┐
                   │  Kafka  │
                   │ (events)│
                   └─────────┘
```

**Arquivo 2:** `docs/diagrams/17-twitter-sequence.drawio`
- Post a tweet (write path — fan-out flow)
- Load home timeline (read path — merge flow)
- Follow a user (graph update + timeline backfill)

**Critérios de aceite:**
- [ ] Write path com fan-out workers
- [ ] Read path com timeline merge
- [ ] Celebrity path (pull on read) separado

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/api/main.go
├── internal/
│   ├── domain/
│   │   ├── tweet.go               ← Tweet entity (280 chars)
│   │   ├── user.go                ← User entity
│   │   ├── follow.go              ← Follow relationship
│   │   └── timeline.go            ← Timeline model
│   ├── tweet/
│   │   ├── service.go             ← Create, delete, search
│   │   ├── service_test.go
│   │   └── validator.go           ← 280 char limit, content rules
│   ├── timeline/
│   │   ├── service.go             ← Home timeline construction
│   │   ├── fanout.go              ← Fan-out on write (push)
│   │   ├── merger.go              ← Fan-out on read (pull merge)
│   │   ├── hybrid.go              ← Hybrid strategy (celebrity detection)
│   │   └── service_test.go
│   ├── social/
│   │   ├── follow_service.go      ← Follow/unfollow
│   │   ├── graph.go               ← Social graph (adjacency list)
│   │   └── graph_test.go
│   ├── search/
│   │   ├── indexer.go             ← Index tweets for search
│   │   └── searcher.go            ← Full-text search
│   ├── trending/
│   │   ├── tracker.go             ← Sliding window counter
│   │   ├── top_k.go               ← Min-heap top-K trending
│   │   └── tracker_test.go
│   ├── handler/
│   │   ├── tweet.go               ← POST/GET /api/v1/tweets
│   │   ├── timeline.go            ← GET /api/v1/timeline
│   │   ├── follow.go              ← POST/DELETE /api/v1/follow
│   │   ├── search.go              ← GET /api/v1/search
│   │   ├── trending.go            ← GET /api/v1/trending
│   │   └── ws.go                  ← WebSocket for real-time
│   ├── repository/
│   │   ├── tweet_repo.go          ← PostgreSQL
│   │   ├── graph_repo.go          ← PostgreSQL (follow relationships)
│   │   ├── timeline_repo.go       ← Redis Sorted Set
│   │   └── search_repo.go         ← Elasticsearch (optional: PG FTS)
│   └── worker/
│       ├── fanout_worker.go       ← Kafka consumer → fan-out
│       └── trending_worker.go     ← Aggregate trending data
├── docker-compose.yml
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **Tweet CRUD** com validação de 280 chars
2. **Follow/Unfollow** com social graph
3. **Fan-out on Write** — Kafka consumer distribui para timelines Redis
4. **Celebrity detection** — se followers > threshold, usa pull model
5. **Home Timeline** — Redis Sorted Set (score = timestamp)
6. **Timeline merge** — para celebridades, merge on-the-fly
7. **Trending topics** — sliding window + min-heap top-K
8. **Search** — PostgreSQL Full-Text Search (ou Elasticsearch)
9. **WebSocket** — real-time timeline updates
10. **Pagination** — cursor-based (tweet_id-based)

**Critérios de aceite Go:**
- [ ] Post tweet → fan-out → aparece na timeline dos followers
- [ ] Celebrity path: pull on read com merge
- [ ] Trending: top-10 topics atualizados em sliding window
- [ ] Search com Full-Text Search
- [ ] WebSocket: novos tweets em real-time
- [ ] Cursor-based pagination na timeline
- [ ] ≥ 20 testes
- [ ] Docker Compose funcional

---

### 3.2 — Java (Spring Boot)

**Funcionalidades Java:**
1. **Spring Boot** REST API
2. **Spring Data JPA** para tweets e social graph
3. **Spring Data Redis** para timelines
4. **Spring Kafka** para fan-out workers
5. **Spring WebSocket** (STOMP) para real-time
6. **Elasticsearch** (Spring Data Elasticsearch) para busca
7. **Records** para DTOs

**Critérios de aceite Java:**
- [ ] Mesmas funcionalidades do Go
- [ ] Kafka fan-out workers
- [ ] WebSocket com STOMP
- [ ] Testes com Testcontainers
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] 4 ADRs (fan-out, storage, search, real-time)
- [ ] 2 DrawIO (HLD + Sequence)
- [ ] Go e Java: implementação completa + tests
- [ ] Fan-out funcional: post → timeline dos followers
- [ ] Docker Compose: app + postgres + redis + kafka + elasticsearch
- [ ] Commit: `feat(system-design-17): twitter social feed`
