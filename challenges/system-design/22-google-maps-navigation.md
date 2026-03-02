# Level 22 — Google Maps / Navigation

> **Objetivo:** Projetar e implementar um sistema de navegação com graph-based routing,
> ETA calculation, geocoding e real-time traffic overlay.

**Referência:** [43-google-maps-navigation.md](../../.docs/SYSTEM-DESIGN/43-google-maps-navigation.md)

**Pré-requisito:** Level 21 completo.

---

## Contexto

Sistema de **navegação e rotas** que encontra o melhor caminho entre dois pontos
considerando distância, tempo estimado e condições de tráfego em tempo real.
O desafio envolve **graph algorithms** (Dijkstra, A*), **map tiling**,
**geocoding** e **live traffic data**.

**Escala alvo:**
- **1B route requests/dia**
- **Mapa:** milhões de nós e arestas (road network)
- **Route latency:** < 500ms
- **ETA accuracy:** ±3 minutos
- **Map tiles:** zoom levels 0-18

---

## Parte 1 — ADRs (4 obrigatórios)

### ADR-001: Routing Algorithm

**Arquivo:** `docs/adrs/ADR-001-routing-algorithm.md`

**Options:**
1. **Dijkstra** — shortest path clássico, correto mas lento para grafos grandes
2. **A\*** — heuristic-guided, mais rápido que Dijkstra
3. **Contraction Hierarchies** — pre-processamento heavy, query ultra-rápida
4. **A\* + Contraction Hierarchies** — A\* para local, CH para long distance

### ADR-002: Graph Data Model

**Options:**
1. **Adjacency list** in memory — rápido, consome muita RAM
2. **Database-backed graph** — PostgreSQL com pgRouting
3. **Memory-mapped files** — grafo serializado em disco com mmap
4. **Tile-based partitioning** — grafo dividido em tiles carregados sob demanda

### ADR-003: Map Tile Serving

**Options:**
1. **Pre-rendered raster tiles** — PNG/JPEG para cada zoom level
2. **Vector tiles** — Protobuf-encoded geometries (MapBox style)
3. **On-demand rendering** — render sob demanda com cache
4. **Hybrid** — pre-render popular areas, on-demand para sparse areas

### ADR-004: Traffic Data Integration

**Options:**
1. **Segment-based speed data** — velocidade média por segmento de estrada
2. **Probe data** — GPS traces de dispositivos para inferir tráfego
3. **Historical patterns** — day-of-week + time-of-day profiling
4. **Real-time + historical** — blend de dados live e patterns

**Critérios de aceite:**
- [ ] 4 ADRs completos
- [ ] Routing algorithm complexity analysis (time + space)
- [ ] Graph size estimation (nodes, edges, memory)

---

## Parte 2 — Diagramas DrawIO (3 obrigatórios)

**Arquivo 1:** `docs/diagrams/22-navigation-hld.drawio`

```
┌──────────┐    ┌──────────────────────────────────────┐
│  Client  │───▶│           API Gateway / LB           │
│(browser/ │    └────┬─────────┬──────────┬────────────┘
│ mobile)  │         │         │          │
└────┬─────┘   ┌─────▼──┐ ┌───▼────┐ ┌───▼──────┐
     │         │Routing │ │Geocode │ │  Tile    │
     │         │Service │ │Service │ │ Service  │
     │         └───┬────┘ └───┬────┘ └────┬─────┘
     │             │          │           │
     │        ┌────▼───┐ ┌───▼────┐ ┌────▼─────┐
     │        │ Graph  │ │Address │ │  Tile    │
     │        │ Store  │ │  DB    │ │  Cache   │
     │        │(memory)│ │        │ │ (Redis)  │
     │        └────────┘ └────────┘ └──────────┘
     │             │
     │        ┌────▼──────┐
     │        │  Traffic  │
     │        │  Service  │
     │        │(real-time)│
     │        └───────────┘
     │
     │        ┌───────────┐
     └───tile─▶│    CDN    │
               │(map tiles)│
               └───────────┘
```

**Arquivo 2:** `docs/diagrams/22-navigation-routing-sequence.drawio`
- Route request: geocode origin/destination → load graph partition → A* search → ETA
- Turn-by-turn: route → instruction generation → real-time rerouting on deviation

**Arquivo 3:** `docs/diagrams/22-navigation-graph.drawio`
- Road network as graph (nodes = intersections, edges = road segments)
- Graph partitioning into tiles
- A* search illustration (open set, closed set, heuristic)

**Critérios de aceite:**
- [ ] HLD com routing, geocoding, tiles e traffic
- [ ] Routing sequence com A* steps
- [ ] Graph structure visualization

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/api/main.go
├── internal/
│   ├── domain/
│   │   ├── node.go                ← Graph Node (intersection, lat/lng)
│   │   ├── edge.go                ← Graph Edge (road segment, weight)
│   │   ├── graph.go               ← Road Network Graph
│   │   ├── route.go               ← Route (path + ETA + distance)
│   │   ├── location.go            ← Coordinates
│   │   └── instruction.go         ← Turn-by-turn instruction
│   ├── graph/
│   │   ├── loader.go              ← Load graph from file/DB
│   │   ├── adjacency.go           ← Adjacency list representation
│   │   ├── partition.go           ← Graph partitioning (tile-based)
│   │   └── loader_test.go
│   ├── routing/
│   │   ├── dijkstra.go            ← Dijkstra (min-heap priority queue)
│   │   ├── astar.go               ← A* with haversine heuristic
│   │   ├── bidirectional.go       ← Bidirectional Dijkstra/A*
│   │   ├── contraction.go         ← Contraction Hierarchies (simplified)
│   │   ├── priority_queue.go      ← Min-heap implementation
│   │   ├── route_builder.go       ← Build route from path
│   │   └── routing_test.go
│   ├── geocoding/
│   │   ├── service.go             ← Forward geocode (address → lat/lng)
│   │   ├── reverse.go             ← Reverse geocode (lat/lng → address)
│   │   ├── trie.go                ← Autocomplete trie for addresses
│   │   └── service_test.go
│   ├── traffic/
│   │   ├── service.go             ← Traffic data management
│   │   ├── segment_speed.go       ← Speed data per road segment
│   │   ├── historical.go          ← Historical patterns (day/time)
│   │   └── service_test.go
│   ├── eta/
│   │   ├── calculator.go          ← ETA = sum(segment_length / speed)
│   │   ├── traffic_aware.go       ← ETA with live traffic weights
│   │   └── calculator_test.go
│   ├── tile/
│   │   ├── service.go             ← Map tile server
│   │   ├── renderer.go            ← Simple tile renderer (SVG/PNG)
│   │   ├── cache.go               ← Tile cache (Redis)
│   │   └── service_test.go
│   ├── navigation/
│   │   ├── turn_by_turn.go        ← Generate turn instructions
│   │   ├── rerouting.go           ← Detect deviation → reroute
│   │   └── navigation_test.go
│   ├── handler/
│   │   ├── route.go               ← GET /api/v1/route?from=...&to=...
│   │   ├── geocode.go             ← GET /api/v1/geocode?q=...
│   │   ├── reverse_geocode.go     ← GET /api/v1/reverse?lat=...&lng=...
│   │   ├── eta.go                 ← GET /api/v1/eta
│   │   ├── tile.go                ← GET /tile/{z}/{x}/{y}.png
│   │   ├── navigate.go            ← WebSocket: turn-by-turn navigation
│   │   └── handler_test.go
│   └── repository/
│       ├── graph_repo.go          ← Load graph from PostgreSQL/file
│       ├── address_repo.go        ← PostgreSQL (address lookups)
│       ├── traffic_repo.go        ← Redis (segment speeds)
│       └── tile_repo.go           ← Redis (tile cache)
├── data/
│   └── sample_graph.json          ← Sample road network
├── docker-compose.yml
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **Graph model** — adjacency list com weighted edges (distance + time)
2. **Dijkstra** — priority queue (container/heap) based shortest path
3. **A\*** — haversine heuristic, faster para point-to-point routing
4. **Bidirectional search** — forward + backward meet in middle
5. **Contraction Hierarchies** (simplified) — shortcut edges for speedup
6. **Geocoding** — forward (text → coordinates) + reverse (coordinates → address)
7. **Address autocomplete** — trie-based prefix search
8. **ETA calculation** — sum of (segment_distance / segment_speed) with traffic
9. **Traffic overlay** — segment speed data (historical + simulated live)
10. **Turn-by-turn** — generate instructions (turn left, continue, etc.)
11. **Map tiles** — simple tile server with Redis cache
12. **Live navigation** — WebSocket for real-time turn-by-turn + rerouting

**Critérios de aceite Go:**
- [ ] Dijkstra: shortest path correto em sample graph
- [ ] A*: faster que Dijkstra, same optimal result
- [ ] Bidirectional: correctness + speedup
- [ ] Geocoding: "Av Paulista" → lat/lng
- [ ] Reverse geocoding: lat/lng → nearest address
- [ ] ETA: time-based routing com traffic weights
- [ ] Turn-by-turn: lista de instruções de navegação
- [ ] Tile server: /tile/{z}/{x}/{y} retorna tile
- [ ] ≥ 25 testes (algorithms devem ser bem testados)
- [ ] Docker Compose: api + postgres + redis

---

### 3.2 — Java (Spring Boot)

**Funcionalidades Java:**
1. **Spring Boot** REST API
2. **Graph** com adjacency list (TreeMap para ordered neighbors)
3. **PriorityQueue** para Dijkstra/A*
4. **Spring Data JPA** para addresses e graph persistence
5. **Spring Data Redis** para tile cache e traffic data
6. **Spring WebSocket** para live navigation
7. **JMH benchmarks** para comparar algoritmos de routing

**Critérios de aceite Java:**
- [ ] Mesmas funcionalidades do Go
- [ ] JMH benchmark: Dijkstra vs A* vs Bidirectional
- [ ] Generics para Graph<N, E>
- [ ] Testes com Testcontainers
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] 4 ADRs (routing algorithm, graph model, tiles, traffic)
- [ ] 3 DrawIO (HLD + Routing Sequence + Graph Structure)
- [ ] Go e Java: implementação completa + tests
- [ ] Routing: A* funcional com heurística haversine
- [ ] Geocoding: forward + reverse
- [ ] ETA com traffic-aware weights
- [ ] Docker Compose: api + postgres + redis
- [ ] Commit: `feat(system-design-22): google maps navigation`
