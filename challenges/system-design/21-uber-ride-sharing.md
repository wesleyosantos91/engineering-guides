# Level 21 — Uber / Ride Sharing

> **Objetivo:** Projetar e implementar uma plataforma de ride-sharing com matching
> geoespacial, tracking em tempo real, pricing dinâmico e ETA calculation.

**Referência:** [42-uber-ride-sharing.md](../../.docs/SYSTEM-DESIGN/42-uber-ride-sharing.md)

**Pré-requisito:** Level 20 completo.

---

## Contexto

Sistema de **mobilidade sob demanda** que conecta riders a drivers em tempo real.
O desafio envolve **geospatial indexing** (encontrar drivers próximos), **real-time
tracking** (posição atualizada a cada 3s), **matching algorithm** e **dynamic pricing**.

**Escala alvo:**
- **100M rides/dia**
- **1M drivers** ativos simultâneos
- **Location updates:** a cada 3 segundos por driver
- **Matching:** < 5 segundos para encontrar um driver
- **ETA accuracy:** ±2 minutos

---

## Parte 1 — ADRs (4 obrigatórios)

### ADR-001: Geospatial Indexing Strategy

**Arquivo:** `docs/adrs/ADR-001-geospatial-indexing.md`

**Options:**
1. **Geohash** — encode lat/lng em string, prefix matching para proximidade
2. **Quadtree** — subdivisão dinâmica de espaço 2D
3. **H3 (Uber hexagons)** — hexagonal grid hierárquico
4. **R-Tree** — index espacial para ranges
5. **S2 Geometry (Google)** — esfera para cells hierárquicas

### ADR-002: Driver-Rider Matching Algorithm

**Options:**
1. **Nearest driver** — simples, driver mais próximo aceita
2. **Batch matching** — acumula requests, otimiza em batch (bipartite matching)
3. **Score-based** — pontuação (distância + rating + acceptance rate)
4. **Supply-demand aware** — considera zones com escassez de drivers

### ADR-003: Real-time Location Tracking

**Options:**
1. **WebSocket** — conexão persistente, push de location updates
2. **gRPC streaming** — bidirecional para mobile
3. **MQTT** — lightweight protocol, bom para mobile (battery)
4. **HTTP polling** — simple, no persistent connection

### ADR-004: Dynamic Pricing (Surge)

**Options:**
1. **Zone-based surge** — multiplier por zona geográfica
2. **Real-time supply-demand** — cálculo contínuo baseado em oferta/demanda
3. **Time-based** — horário de pico pré-definido
4. **ML-based** — predição de demanda para pricing proativo

**Critérios de aceite:**
- [ ] 4 ADRs completos
- [ ] Geospatial comparison com prós/contras de cada approach
- [ ] Matching algorithm pseudocode/flowchart

---

## Parte 2 — Diagramas DrawIO (3 obrigatórios)

**Arquivo 1:** `docs/diagrams/21-rideshare-hld.drawio`

```
┌──────────┐         ┌────────────────────────────────┐
│  Rider   │◀═══WS══▶│     Real-time Gateway          │
│   App    │         │  (WebSocket + Location)         │
└──────────┘         └──────────┬─────────────────────┘
                                │
┌──────────┐         ┌──────────┼──────────┬───────────┐
│  Driver  │◀═══WS══▶│         │          │           │
│   App    │         │   ┌─────▼──┐ ┌─────▼──┐ ┌─────▼──┐
└──────────┘         │   │Matching│ │ Trip   │ │Pricing │
                     │   │Service │ │Service │ │Service │
                     │   └───┬────┘ └───┬────┘ └───┬────┘
                     │       │          │          │
                     │  ┌────▼────┐┌────▼───┐┌────▼───┐
                     │  │GeoIndex ││  Trip  ││ Surge  │
                     │  │ (Redis  ││  DB    ││  Data  │
                     │  │ GEO)   ││        ││(Redis) │
                     │  └─────────┘└────────┘└────────┘
                     └──────────────────────────────────┘
```

**Arquivo 2:** `docs/diagrams/21-rideshare-matching-sequence.drawio`
- Ride request flow: rider → pricing → matching → driver notification → accept → trip start
- Location tracking flow: driver → gateway → geospatial update → rider view

**Arquivo 3:** `docs/diagrams/21-rideshare-geospatial.drawio`
- Geohash/H3 grid visualization
- Nearest driver search radius expansion
- Zone-based surge pricing map

**Critérios de aceite:**
- [ ] HLD com todos os serviços e data stores
- [ ] Matching sequence completo (request → accept)
- [ ] Geospatial index visualization

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/
│   ├── api/main.go                ← REST + WebSocket server
│   └── matching/main.go           ← Matching worker
├── internal/
│   ├── domain/
│   │   ├── rider.go               ← Rider entity
│   │   ├── driver.go              ← Driver entity + status
│   │   ├── trip.go                ← Trip entity (lifecycle)
│   │   ├── location.go            ← Location (lat, lng, timestamp)
│   │   ├── pricing.go             ← Fare calculation
│   │   └── zone.go                ← Geographic zone
│   ├── geo/
│   │   ├── geohash.go             ← Geohash encode/decode
│   │   ├── quadtree.go            ← Quadtree spatial index
│   │   ├── h3_index.go            ← H3 hexagonal index (simplified)
│   │   ├── distance.go            ← Haversine distance calculation
│   │   └── geo_test.go
│   ├── matching/
│   │   ├── service.go             ← Match rider to driver
│   │   ├── nearest.go             ← Nearest driver strategy
│   │   ├── scored.go              ← Scored matching (distance+rating)
│   │   ├── batch.go               ← Batch matching (bipartite)
│   │   └── service_test.go
│   ├── trip/
│   │   ├── service.go             ← Trip lifecycle (request → complete)
│   │   ├── fare.go                ← Fare calculation (base + distance + time + surge)
│   │   └── service_test.go
│   ├── pricing/
│   │   ├── surge.go               ← Surge multiplier calculation
│   │   ├── zone_tracker.go        ← Demand/supply per zone
│   │   ├── estimator.go           ← Price estimation (before ride)
│   │   └── surge_test.go
│   ├── tracking/
│   │   ├── service.go             ← Real-time location tracking
│   │   ├── eta.go                 ← ETA calculation (distance/speed based)
│   │   └── service_test.go
│   ├── handler/
│   │   ├── ride.go                ← POST /api/v1/rides (request ride)
│   │   ├── estimate.go            ← GET /api/v1/estimate (price estimate)
│   │   ├── trip.go                ← GET/PATCH /api/v1/trips/:id
│   │   ├── driver.go              ← PATCH /api/v1/drivers/status
│   │   ├── ws_tracking.go         ← WebSocket: location updates + trip tracking
│   │   └── handler_test.go
│   ├── repository/
│   │   ├── driver_location.go     ← Redis GEO (driver positions)
│   │   ├── trip_repo.go           ← PostgreSQL
│   │   ├── driver_repo.go         ← PostgreSQL
│   │   └── surge_repo.go          ← Redis (zone demand/supply)
│   └── worker/
│       ├── matching_worker.go     ← Kafka consumer → match
│       └── surge_calculator.go    ← Periodic surge recalculation
├── docker-compose.yml
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **Geospatial indexing** — Redis GEO (GEOADD, GEORADIUS) + custom Geohash
2. **Quadtree** — implementação from scratch para spatial search
3. **Haversine distance** — cálculo de distância entre coordenadas
4. **Driver matching** — 3 strategies: nearest, scored, batch
5. **Trip lifecycle** — REQUEST → MATCHED → DRIVER_ARRIVING → IN_PROGRESS → COMPLETED
6. **Fare calculation** — base_fare + (distance × rate) + (time × rate) × surge_multiplier
7. **Surge pricing** — supply/demand ratio por zone, recalculado a cada 30s
8. **ETA calculation** — distância / velocidade média + traffic factor
9. **Real-time tracking** — WebSocket: driver location broadcast para rider
10. **Location updates** — driver publica posição a cada 3s via WebSocket

**Critérios de aceite Go:**
- [ ] Geohash encode/decode funcional
- [ ] Quadtree: insert/search/remove drivers
- [ ] Redis GEO: find drivers within radius
- [ ] Matching: request → match driver → accept → trip start
- [ ] Trip lifecycle completo (request → complete)
- [ ] Fare: cálculo com base + distance + time + surge
- [ ] Surge: varia com supply/demand
- [ ] WebSocket: tracking em tempo real
- [ ] ETA funcional
- [ ] ≥ 22 testes
- [ ] Docker Compose: api + matcher + postgres + redis + kafka

---

### 3.2 — Java (Spring Boot)

**Funcionalidades Java:**
1. **Spring Boot** REST API + WebSocket
2. **Redis GEO** via Spring Data Redis (`GeoOperations`)
3. **Spring Data JPA** para trips, drivers, riders
4. **Spring Kafka** para matching events
5. **Spring WebSocket** para real-time tracking
6. **Strategy Pattern** para matching algorithms
7. **`@Scheduled`** para surge recalculation

**Critérios de aceite Java:**
- [ ] Mesmas funcionalidades do Go
- [ ] Redis GEO integration via Spring Data
- [ ] Strategy Pattern para matching
- [ ] Testes com Testcontainers
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] 4 ADRs (geospatial, matching, tracking, pricing)
- [ ] 3 DrawIO (HLD + Matching Sequence + Geospatial)
- [ ] Go e Java: implementação completa + tests
- [ ] Geospatial: find nearest drivers within radius
- [ ] Matching: rider request → driver match → trip
- [ ] Tracking: real-time via WebSocket
- [ ] Docker Compose: api + matcher + postgres + redis + kafka
- [ ] Commit: `feat(system-design-21): uber ride sharing`
