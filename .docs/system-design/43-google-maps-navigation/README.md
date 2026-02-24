# 43. Google Maps — Navigation

## Índice

- [Visão Geral](#visão-geral)
- [Requisitos](#requisitos)
- [Estimativas (Back-of-the-Envelope)](#estimativas-back-of-the-envelope)
- [Arquitetura de Alto Nível](#arquitetura-de-alto-nível)
- [Map Tiles Service](#map-tiles-service)
- [Routing Engine](#routing-engine)
- [Search / Places Service](#search--places-service)
- [Traffic Service](#traffic-service)
- [Navigation Service](#navigation-service)
- [Geocoding e Reverse Geocoding](#geocoding-e-reverse-geocoding)
- [Modelo de Dados](#modelo-de-dados)
- [Código Ilustrativo](#código-ilustrativo)
- [Google Maps Real Architecture](#google-maps-real-architecture)
- [Trade-offs e Decisões](#trade-offs-e-decisões)
- [Perguntas Comuns em Entrevistas](#perguntas-comuns-em-entrevistas)
- [Referências](#referências)

---

## Visão Geral

Um sistema de mapas e navegação permite **visualizar mapas interativos**, **buscar locais**, **calcular rotas** e **navegar em tempo real** com informações de tráfego. O desafio principal está em servir **mapa do mundo inteiro** com **baixa latência**, calcular rotas em **grafos com bilhões de arestas** e incorporar **dados de tráfego em tempo real**.

**Big Techs que operam este tipo de sistema:**
- **Google Maps** — 1B+ MAU, mapa mais detalhado
- **Apple Maps** — integrado ao iOS
- **Waze** (Google) — crowdsourced traffic
- **HERE** (ex-Nokia) — usado na indústria automotiva
- **Mapbox** — platform para developers

---

## Requisitos

### Funcionais
| Requisito | Descrição |
|-----------|-----------|
| Mapa interativo | Pan, zoom, tilt com tiles renderizados |
| Busca de locais | Restaurants, endereços, POIs |
| Routing | Calcular melhor rota (driving, walking, transit, cycling) |
| Navegação | Turn-by-turn com voz e recálculo automático |
| Tráfego | Real-time traffic overlay no mapa |
| Street View | Fotos panorâmicas de ruas |
| Offline maps | Download de regiões para uso offline |

### Não-Funcionais
| Requisito | Valor |
|-----------|-------|
| MAU | 1B+ |
| Latência de routing | < 200ms para rota curta, < 2s para rota longa |
| Tile loading | < 200ms com cache |
| Cobertura | 220+ países, 50M+ km de estradas |
| Disponibilidade | 99.99% |
| Freshness de tráfego | < 2 minutos de delay |

---

## Estimativas (Back-of-the-Envelope)

```
Map Tiles:
  Zoom levels: 0-21 (22 níveis)
  Tiles no zoom 21: 4^21 ≈ 4.4 trilhões de tiles
  Na prática: ~1-5 bilhões de tiles (terra/oceano ratio)
  Tile size (raster): ~20-50 KB
  Tile size (vector): ~5-15 KB
  Total storage: ~50-100 TB (all zoom levels)

Routing:
  Road graph nodes: ~500M interseções
  Road graph edges: ~1B segmentos de estrada
  Graph data: ~100-200 GB (comprimido)
  QPS routing: ~10K req/s (Google-scale)

Traffic:
  Data sources: ~1B smartphones Android contribuindo
  Updates: millions of GPS probes/minute
  Processamento: streaming (Dataflow/Flink)
```

---

## Arquitetura de Alto Nível

```
┌──────────────────────────────────────────────────────────────────────┐
│                        MAPS & NAVIGATION PLATFORM                     │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┐          ┌──────────────┐                              │
│  │ Map App  │─────────▶│  API Gateway │                              │
│  │ (Client) │◀─────────│              │                              │
│  └──────────┘          └──────┬───────┘                              │
│                               │                                      │
│      ┌────────────────────────┼────────────────────────┐             │
│      │                        │                        │             │
│      ▼                        ▼                        ▼             │
│  ┌──────────────┐   ┌──────────────────┐   ┌──────────────────┐     │
│  │  Map Tiles   │   │  Routing Engine  │   │  Search / Places │     │
│  │  Service     │   │  (shortest path) │   │  Service         │     │
│  └──────┬───────┘   └────────┬─────────┘   └────────┬─────────┘     │
│         │                    │                       │               │
│  ┌──────▼───────┐   ┌───────▼──────────┐   ┌────────▼─────────┐     │
│  │  Tile Store  │   │  Road Graph      │   │  Elasticsearch   │     │
│  │  + CDN       │   │  (weighted edges)│   │  (POI index)     │     │
│  └──────────────┘   └───────┬──────────┘   └──────────────────┘     │
│                             │                                        │
│                      ┌──────▼──────────┐                             │
│                      │  Traffic Service│                             │
│                      │  (real-time)    │                             │
│                      └────────┬────────┘                             │
│                               │                                      │
│                      ┌────────▼────────┐                             │
│                      │  GPS Probes     │                             │
│                      │  (smartphones)  │                             │
│                      └─────────────────┘                             │
│                                                                      │
│  ┌──────────────────────────────────────────┐                        │
│  │         Supporting Services               │                        │
│  ├──────────┬────────────┬──────────────────┤                        │
│  │Geocoding │ Navigation │ Offline Maps     │                        │
│  │ (addr↔   │ (turn-by-  │ (downloadable   │                        │
│  │  coord)  │  turn)     │  regions)       │                        │
│  └──────────┴────────────┴──────────────────┘                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Map Tiles Service

Mapas são divididos em **tiles** (quadrados) em cada nível de zoom:

```
┌──────────────────────────────────────────────────────────┐
│                    MAP TILE PYRAMID                        │
│                                                           │
│  Zoom 0: 1 tile (mundo inteiro)                          │
│  ┌─────────────────────┐                                 │
│  │                     │                                 │
│  │     World           │     1 tile                      │
│  │                     │                                 │
│  └─────────────────────┘                                 │
│                                                           │
│  Zoom 1: 4 tiles                                         │
│  ┌─────────┬───────────┐                                 │
│  │    0,0  │    1,0    │                                 │
│  ├─────────┼───────────┤     4 tiles                     │
│  │    0,1  │    1,1    │                                 │
│  └─────────┴───────────┘                                 │
│                                                           │
│  Zoom 2: 16 tiles                                        │
│  Zoom 3: 64 tiles                                        │
│  ...                                                      │
│  Zoom 15: ~1 bilhão tiles (ruas visíveis)                │
│  Zoom 21: ~4 trilhões tiles (building-level)             │
│                                                           │
│  Fórmula: tiles no zoom Z = 4^Z                          │
│  URL pattern: /tiles/{z}/{x}/{y}.png                     │
│                                                           │
│  Caching Strategy:                                        │
│  • CDN edge: tiles populares (cidades grandes)           │
│  • Browser cache: tiles recentemente vistos              │
│  • Pre-fetch: tiles adjacentes ao viewport               │
└──────────────────────────────────────────────────────────┘
```

### Raster Tiles vs Vector Tiles

| Aspecto | Raster (.png) | Vector (.pbf) |
|---------|--------------|---------------|
| Renderização | Server-side | Client-side |
| Tamanho | ~20-50 KB | ~5-15 KB |
| Rotação/inclinação | Pixelado | Suave |
| Customização | Nenhuma | Styling no client |
| Labels | Fixos na imagem | Dinâmicos (rotação) |
| Zoom suave | Não (discreto) | Sim (contínuo) |
| Offline | Mais storage | Menos storage |
| Adoção | Google Maps (antigo) | Google Maps (atual), Mapbox |

```
Vector Tiles (moderno):
• Dados geométricos (pontos, linhas, polígonos) em Protocol Buffers
• Client aplica estilo (cores, labels, filtros)
• Suporte a 3D buildings, rotação, tilt
• ~70% menos bandwidth que raster
• Google Maps migrou para vector tiles em 2020
```

---

## Routing Engine

O core do sistema de navegação é o **grafo de estradas**:

```
┌──────────────────────────────────────────────────────────┐
│                    ROAD NETWORK AS GRAPH                   │
│                                                           │
│  Nó = Interseção (lat, lng)                              │
│  Aresta = Segmento de rua (peso = tempo de viagem)       │
│                                                           │
│      A ──(5min)──▶ B ──(3min)──▶ C                       │
│      │              │              │                      │
│   (2min)         (7min)         (1min)                   │
│      │              │              │                      │
│      ▼              ▼              ▼                      │
│      D ──(4min)──▶ E ──(2min)──▶ F                       │
│                                                           │
│  Atributos da aresta:                                    │
│  • Tempo (principal peso)                                │
│  • Distância                                             │
│  • Speed limit                                           │
│  • Road type (highway, residential, etc.)                │
│  • Restrições (one-way, no U-turn, time-based)          │
│  • Toll (pedágio)                                        │
│  • Current traffic speed                                  │
└──────────────────────────────────────────────────────────┘
```

### Algoritmos de Routing

| Algoritmo | Complexidade | Pre-proc. | Query Time | Uso |
|-----------|-------------|-----------|------------|-----|
| **Dijkstra** | O(V log V + E) | Nenhum | ~segundos | Acadêmico |
| **A*** | O(V log V) | Nenhum | ~100ms | Grafos médios |
| **Contraction Hierarchies** | — | Horas | **~microseconds** | **Google, OSRM** |
| **ALT (A* + Landmarks)** | — | Minutos | ~10ms | Alternativa ao CH |
| **Transit Node Routing** | — | Horas | ~microseconds | Long-distance |

### Contraction Hierarchies (CH) em Detalhe

```
┌──────────────────────────────────────────────────────────┐
│            CONTRACTION HIERARCHIES                         │
│                                                           │
│  Ideia: Pre-processar "atalhos" no grafo para que         │
│  queries sejam ultra-rápidas.                             │
│                                                           │
│  Pre-processamento (offline, horas):                      │
│  1. Ordenar nós por "importância"                        │
│     (highways > nós de bairro)                            │
│  2. "Contrair" nós de baixa importância:                 │
│     Se A→B→C e B é pouco importante,                     │
│     criar shortcut A→C (peso = peso(A→B) + peso(B→C))   │
│  3. Repetir para todos os nós                            │
│                                                           │
│  Exemplo:                                                 │
│  Original: A ─(2)─ B ─(3)─ C ─(1)─ D                    │
│  Contrai B: A ─(5)─────── C ─(1)─ D  (shortcut A→C=5)  │
│  Contrai C: A ─(6)──────────────── D  (shortcut A→D=6)  │
│                                                           │
│  Query (microseconds):                                    │
│  • Bidirectional Dijkstra no grafo com shortcuts          │
│  • Forward search sobe a hierarquia                      │
│  • Backward search sobe a hierarquia                     │
│  • Encontram-se em nó de alta importância (highway)      │
│                                                           │
│  Performance:                                             │
│  • Dijkstra: explora milhões de nós                      │
│  • CH: explora ~1000 nós                                 │
│  • Speedup: ~1000-10000x                                 │
└──────────────────────────────────────────────────────────┘
```

### Multi-Modal Routing

```
┌─────────────────────────────────────────────────┐
│         MULTI-MODAL ROUTING                      │
│                                                  │
│  Modo        │ Grafo Usado                       │
│  ────────────┼──────────────────────────         │
│  Driving     │ Road graph + traffic              │
│  Walking     │ Pedestrian graph (calçadas)       │
│  Cycling     │ Bike lane graph                   │
│  Transit     │ GTFS schedule graph +              │
│              │ walking for first/last mile         │
│                                                  │
│  Transit Routing (complexo):                     │
│  • GTFS (General Transit Feed Specification)     │
│  • Time-dependent graph (horários)               │
│  • RAPTOR algorithm (Round-bAsed Public          │
│    Transit Optimized Router)                     │
│  • Combina: walk → station → bus/train →         │
│    transfer → bus → walk                         │
└─────────────────────────────────────────────────┘
```

---

## Search / Places Service

```
┌──────────────────────────────────────────────────────────┐
│                  PLACES SEARCH                            │
│                                                           │
│  Query: "restaurante italiano perto de mim"               │
│                                                           │
│  Pipeline:                                                │
│  1. Parse query → {type: "restaurant",                    │
│                     cuisine: "italian",                    │
│                     near: user_location}                   │
│                                                           │
│  2. Geocode "perto de mim" → (lat, lng, radius)          │
│                                                           │
│  3. Search Elasticsearch:                                 │
│     • geo_distance filter (within radius)                │
│     • text match (name, category, cuisine)               │
│     • Sort by: relevance × distance × rating × recency  │
│                                                           │
│  4. Enrich results:                                       │
│     • Business hours, phone, website                     │
│     • Photos, reviews, rating (stars)                    │
│     • Real-time: open/closed, busy level                 │
│                                                           │
│  Elasticsearch Mapping:                                   │
│  {                                                        │
│    "name": { "type": "text", "analyzer": "custom" },     │
│    "location": { "type": "geo_point" },                  │
│    "category": { "type": "keyword" },                    │
│    "rating": { "type": "float" },                        │
│    "reviews_count": { "type": "integer" }                │
│  }                                                        │
└──────────────────────────────────────────────────────────┘
```

---

## Traffic Service

```
┌──────────────────────────────────────────────────────────┐
│                  TRAFFIC SERVICE                          │
│                                                           │
│  Data Sources:                                            │
│  ┌─────────────────────────────────────────┐             │
│  │ • GPS probes (1B+ Android phones)       │             │
│  │ • Waze user reports                     │             │
│  │ • Road sensors / cameras                │             │
│  │ • Historical patterns                   │             │
│  │ • Events (acidentes, obras, shows)      │             │
│  └──────────────┬──────────────────────────┘             │
│                 │                                         │
│  Processing:    ▼                                         │
│  ┌─────────────────────────────────────────┐             │
│  │ Streaming Pipeline (Dataflow/Flink)     │             │
│  │                                          │             │
│  │ 1. Aggregate GPS probes by road segment │             │
│  │ 2. Calculate avg speed per segment      │             │
│  │ 3. Compare with free-flow speed         │             │
│  │ 4. Classify: green/yellow/red           │             │
│  │ 5. Update edge weights no road graph    │             │
│  │ 6. Predict next 30-60 min (ML)          │             │
│  └──────────────┬──────────────────────────┘             │
│                 │                                         │
│  Output:        ▼                                         │
│  ┌─────────────────────────────────────────┐             │
│  │ • Traffic overlay tiles (colored roads) │             │
│  │ • Updated ETA calculations              │             │
│  │ • Route recalculation triggers          │             │
│  │ • Incident alerts                       │             │
│  └─────────────────────────────────────────┘             │
│                                                           │
│  Classificação:                                          │
│  ┌────────┬──────────────┬─────────────────┐            │
│  │ Cor    │ Velocidade   │ Significado     │            │
│  ├────────┼──────────────┼─────────────────┤            │
│  │ 🟢     │ > 80% free   │ Trânsito livre  │            │
│  │ 🟡     │ 40-80% free  │ Lento           │            │
│  │ 🔴     │ 20-40% free  │ Congestionado   │            │
│  │ 🟤     │ < 20% free   │ Parado          │            │
│  └────────┴──────────────┴─────────────────┘            │
└──────────────────────────────────────────────────────────┘
```

---

## Navigation Service

```
┌──────────────────────────────────────────────────────────┐
│                NAVIGATION (TURN-BY-TURN)                  │
│                                                           │
│  State Machine:                                           │
│                                                           │
│  ┌────────┐    ┌──────────┐    ┌───────────────┐        │
│  │ ROUTE  │───▶│NAVIGATING│───▶│  ARRIVED      │        │
│  │PREVIEW │    │          │    │               │        │
│  └────────┘    └────┬─────┘    └───────────────┘        │
│                     │                                    │
│                     ▼                                    │
│              ┌─────────────┐                             │
│              │ REROUTING   │ (saiu da rota)              │
│              │ (recalc)    │                              │
│              └──────┬──────┘                             │
│                     │                                    │
│                     ▼                                    │
│              ┌─────────────┐                             │
│              │ NAVIGATING  │ (nova rota)                 │
│              └─────────────┘                             │
│                                                           │
│  Turn-by-Turn Logic:                                      │
│  1. Client envia GPS position a cada 1-2s                │
│  2. Server (ou client offline) faz map-matching:         │
│     GPS impreciso → segmento de rua mais provável       │
│  3. Calcula distância até próxima instrução              │
│  4. Se > 50m da rota → trigger rerouting                │
│  5. Gera instrução de voz:                               │
│     "Em 200 metros, vire à direita na Av. Paulista"     │
│                                                           │
│  Map Matching Algorithm:                                  │
│  • Hidden Markov Model (HMM)                             │
│  • Emission: distância GPS → segmento de rua            │
│  • Transition: conectividade entre segmentos             │
│  • Viterbi algorithm para caminho mais provável          │
└──────────────────────────────────────────────────────────┘
```

---

## Geocoding e Reverse Geocoding

```
┌──────────────────────────────────────────────────────────┐
│           GEOCODING SERVICE                               │
│                                                           │
│  Geocoding (endereço → coordenada):                      │
│  "Av. Paulista, 1578, São Paulo"                         │
│  → (-23.5615, -46.6559)                                  │
│                                                           │
│  Reverse Geocoding (coordenada → endereço):              │
│  (-23.5615, -46.6559)                                    │
│  → "Av. Paulista, 1578 - Bela Vista, São Paulo - SP"    │
│                                                           │
│  Pipeline de Geocoding:                                   │
│  1. Normalizar input (remover acentos, abreviações)      │
│  2. Tokenize (rua, número, bairro, cidade, CEP)          │
│  3. Busca em address database (interpolação de números)  │
│  4. Retorna coordenada + confidence score                │
│                                                           │
│  Pipeline de Reverse Geocoding:                           │
│  1. Spatial index query (R-Tree / S2)                    │
│  2. Encontrar segmento de rua mais próximo               │
│  3. Interpolar número da casa                            │
│  4. Compor endereço formatado                            │
│                                                           │
│  Data Sources:                                            │
│  • Government datasets (IBGE, USPS, etc.)                │
│  • OpenStreetMap contributors                            │
│  • Street View OCR (captura de números/placas)           │
│  • User corrections                                      │
└──────────────────────────────────────────────────────────┘
```

---

## Modelo de Dados

### Road Graph (Adjacency List)

```
Formato compacto para grafo de bilhões de arestas:

┌─────────────────────────────────────────────────┐
│  Node Table (500M nodes)                        │
│  ┌────────┬─────────────────┬─────────────────┐│
│  │ NodeID │ Lat             │ Lng             ││
│  ├────────┼─────────────────┼─────────────────┤│
│  │ 0      │ -23.561500      │ -46.655900      ││
│  │ 1      │ -23.562100      │ -46.656200      ││
│  └────────┴─────────────────┴─────────────────┘│
│                                                  │
│  Edge Table (1B edges)                           │
│  ┌──────┬──────┬───────┬────────┬──────┬──────┐│
│  │ From │ To   │ Dist  │Time(s) │Speed │ Type ││
│  ├──────┼──────┼───────┼────────┼──────┼──────┤│
│  │ 0    │ 1    │ 120m  │ 18     │ 24   │ res. ││
│  │ 1    │ 42   │ 450m  │ 30     │ 54   │ hwy  ││
│  └──────┴──────┴───────┴────────┴──────┴──────┘│
│                                                  │
│  Storage:                                        │
│  • Memory-mapped files para acesso rápido       │
│  • Flat arrays (CSR format) para locality        │
│  • Partitioned por região geográfica            │
│  • ~100-200 GB total (comprimido)               │
└─────────────────────────────────────────────────┘
```

### Places / POI (Elasticsearch + SQL)

```sql
-- Points of Interest
CREATE TABLE places (
    place_id        UUID PRIMARY KEY,
    name            VARCHAR(500),
    address         TEXT,
    lat             DECIMAL(10, 7),
    lng             DECIMAL(10, 7),
    category        VARCHAR(100),     -- 'restaurant', 'gas_station', 'hospital'
    subcategory     VARCHAR(100),     -- 'italian', 'japanese'
    rating          DECIMAL(2, 1),
    reviews_count   INT,
    phone           VARCHAR(20),
    website         VARCHAR(500),
    hours           JSONB,            -- {"mon": "08:00-22:00", ...}
    price_level     SMALLINT,         -- 1-4 ($-$$$$)
    photos          TEXT[],
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP
);

-- Spatial index para queries geográficas
CREATE INDEX idx_places_geo ON places USING GIST (
    ST_MakePoint(lng, lat)::geography
);

-- Full-text search index
CREATE INDEX idx_places_search ON places USING GIN (
    to_tsvector('portuguese', name || ' ' || category || ' ' || address)
);
```

---

## Código Ilustrativo

### Contraction Hierarchies Query (Python)

```python
import heapq
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple

@dataclass
class Edge:
    to: int
    weight: float  # tempo em segundos
    is_shortcut: bool = False
    
@dataclass
class CHGraph:
    """Grafo com Contraction Hierarchies pre-processado."""
    # Adjacency lists separadas para forward/backward search
    forward: Dict[int, List[Edge]]   # upward edges
    backward: Dict[int, List[Edge]]  # upward edges (reversed)
    rank: Dict[int, int]             # node importance ranking

class CHRouter:
    def __init__(self, graph: CHGraph):
        self.graph = graph
    
    def shortest_path(self, source: int, target: int) -> Optional[float]:
        """
        Bidirectional Dijkstra no grafo CH.
        Forward search: source → up
        Backward search: target → up
        Encontram-se no meeting point.
        """
        INF = float('inf')
        
        # Forward distances
        dist_f: Dict[int, float] = {source: 0}
        # Backward distances
        dist_b: Dict[int, float] = {target: 0}
        
        # Priority queues: (distance, node)
        pq_f = [(0, source)]
        pq_b = [(0, target)]
        
        visited_f = set()
        visited_b = set()
        
        best = INF
        
        while pq_f or pq_b:
            # Forward step
            if pq_f:
                d, u = heapq.heappop(pq_f)
                if d > best:
                    pq_f = []  # prune
                elif u not in visited_f:
                    visited_f.add(u)
                    # Check if backward search already reached u
                    if u in dist_b:
                        best = min(best, d + dist_b[u])
                    # Relax upward edges only
                    for edge in self.graph.forward.get(u, []):
                        if self.graph.rank[edge.to] > self.graph.rank[u]:
                            new_dist = d + edge.weight
                            if new_dist < dist_f.get(edge.to, INF):
                                dist_f[edge.to] = new_dist
                                heapq.heappush(pq_f, (new_dist, edge.to))
            
            # Backward step
            if pq_b:
                d, u = heapq.heappop(pq_b)
                if d > best:
                    pq_b = []  # prune
                elif u not in visited_b:
                    visited_b.add(u)
                    if u in dist_f:
                        best = min(best, d + dist_f[u])
                    for edge in self.graph.backward.get(u, []):
                        if self.graph.rank[edge.to] > self.graph.rank[u]:
                            new_dist = d + edge.weight
                            if new_dist < dist_b.get(edge.to, INF):
                                dist_b[edge.to] = new_dist
                                heapq.heappush(pq_b, (new_dist, edge.to))
        
        return best if best < INF else None
```

### Tile URL Generator (JavaScript)

```javascript
/**
 * Converte lat/lng + zoom level em coordenadas de tile.
 * Padrão Slippy Map (OpenStreetMap/Google Maps).
 */
function latLngToTile(lat, lng, zoom) {
  const n = Math.pow(2, zoom);
  const x = Math.floor(((lng + 180) / 360) * n);
  const latRad = (lat * Math.PI) / 180;
  const y = Math.floor(
    ((1 - Math.log(Math.tan(latRad) + 1 / Math.cos(latRad)) / Math.PI) / 2) * n
  );
  return { x, y, z: zoom };
}

/**
 * Gera URLs para todos os tiles visíveis no viewport.
 */
function getVisibleTiles(bounds, zoom, tileServerUrl) {
  const topLeft = latLngToTile(bounds.north, bounds.west, zoom);
  const bottomRight = latLngToTile(bounds.south, bounds.east, zoom);
  
  const tiles = [];
  for (let x = topLeft.x; x <= bottomRight.x; x++) {
    for (let y = topLeft.y; y <= bottomRight.y; y++) {
      tiles.push({
        url: `${tileServerUrl}/${zoom}/${x}/${y}.pbf`,
        x, y, z: zoom,
      });
    }
  }
  return tiles;
}

// Exemplo: São Paulo, zoom 15
const tile = latLngToTile(-23.5505, -46.6333, 15);
console.log(tile); // { x: 10472, y: 17693, z: 15 }
// URL: /tiles/15/10472/17693.pbf
```

### Geohash Encoding (Go)

```go
package geohash

const base32 = "0123456789bcdefghjkmnpqrstuvwxyz"

// Encode converte lat/lng em geohash string com precisão dada.
func Encode(lat, lng float64, precision int) string {
    minLat, maxLat := -90.0, 90.0
    minLng, maxLng := -180.0, 180.0
    
    result := make([]byte, 0, precision)
    bit := 0
    ch := 0
    isLng := true // alterna entre lng e lat
    
    for len(result) < precision {
        if isLng {
            mid := (minLng + maxLng) / 2
            if lng >= mid {
                ch |= 1 << (4 - bit)
                minLng = mid
            } else {
                maxLng = mid
            }
        } else {
            mid := (minLat + maxLat) / 2
            if lat >= mid {
                ch |= 1 << (4 - bit)
                minLat = mid
            } else {
                maxLat = mid
            }
        }
        isLng = !isLng
        
        bit++
        if bit == 5 {
            result = append(result, base32[ch])
            bit = 0
            ch = 0
        }
    }
    return string(result)
}

// Neighbors retorna os 8 geohashes adjacentes.
func Neighbors(hash string) [8]string {
    lat, lng := Decode(hash)
    precision := len(hash)
    latErr, lngErr := errorForPrecision(precision)
    
    return [8]string{
        Encode(lat+latErr, lng, precision),         // N
        Encode(lat+latErr, lng+lngErr, precision),  // NE
        Encode(lat, lng+lngErr, precision),          // E
        Encode(lat-latErr, lng+lngErr, precision),  // SE
        Encode(lat-latErr, lng, precision),          // S
        Encode(lat-latErr, lng-lngErr, precision),  // SW
        Encode(lat, lng-lngErr, precision),          // W
        Encode(lat+latErr, lng-lngErr, precision),  // NW
    }
}
```

---

## Google Maps Real Architecture

```
┌──────────────────────────────────────────────────────────┐
│            GOOGLE MAPS ARCHITECTURE                       │
│                                                           │
│  Data Pipeline:                                           │
│  ┌──────────────────────────────────────────┐            │
│  │ • Satellite imagery (DigitalGlobe, etc.) │            │
│  │ • Street View cars (360° cameras)        │            │
│  │ • Android GPS probes (1B+ devices)       │            │
│  │ • Partner data (municipalities, transit) │            │
│  │ • User-contributed edits                 │            │
│  │ • Ground Truth (AI + humans for quality) │            │
│  └──────────────┬───────────────────────────┘            │
│                 │                                         │
│  Processing:    ▼                                         │
│  ┌──────────────────────────────────────────┐            │
│  │ • Bigtable: road graph, tile data        │            │
│  │ • Spanner: metadata, places              │            │
│  │ • Colossus: imagery, Street View         │            │
│  │ • MapReduce/Dataflow: batch processing   │            │
│  │ • Borg (K8s): orchestration              │            │
│  └──────────────────────────────────────────┘            │
│                                                           │
│  Key Technologies:                                        │
│  • Contraction Hierarchies para routing                  │
│  • S2 Geometry para geospatial indexing                  │
│  • Ground Truth: ML pipeline para extrair dados de       │
│    Street View (nomes de ruas, números, business)        │
│  • DeepMind AI para ETA prediction                       │
│  • Immersive View: NeRF + satellite + Street View 3D    │
└──────────────────────────────────────────────────────────┘
```

---

## Trade-offs e Decisões

| Decisão | Opção A | Opção B | Escolha Recomendada |
|---------|---------|---------|--------------------|
| Tiles | Raster (server render) | Vector (client render) | Vector (moderno, eficiente) |
| Routing algorithm | Dijkstra/A* | Contraction Hierarchies | CH (1000x mais rápido) |
| Traffic data source | Sensors fixos | Crowdsourced GPS | GPS crowdsourced (escala) |
| Graph storage | SQL | Memory-mapped flat files | Memory-mapped (performance) |
| Places search | PostgreSQL FTS | Elasticsearch | Elasticsearch (features) |
| Tile delivery | Origin server | CDN + edge cache | CDN (latência) |
| Map style | Server-side | Client-side (vector) | Client-side (customizável) |
| Offline | Cache tiles | Download region package | Region package (eficiente) |

---

## Perguntas Comuns em Entrevistas

1. **"Como calcular a rota mais rápida entre dois pontos?"**
   - Road network como grafo, Contraction Hierarchies para query rápida, bidirectional search, traffic-aware edge weights

2. **"Como servir mapa de todo o mundo com baixa latência?"**
   - Tile pyramid (22 zoom levels), pre-rendered + CDN edge caching, vector tiles para reduzir bandwidth, client-side rendering

3. **"Como o traffic overlay funciona em tempo real?"**
   - GPS probes de smartphones, aggregate speed por segmento (streaming pipeline), classifica em verde/amarelo/vermelho, atualiza a cada 1-2 min

4. **"Como Google Maps sabe o nome das ruas?"**
   - OpenStreetMap + data de governos + Street View OCR (ML extrai texto das imagens) + user edits

5. **"Como lidar com routing offline?"**
   - Download grafo comprimido da região, CH pre-processado funciona offline, tiles em cache local, fallback para GPS-only

6. **"Como o ETA é tão preciso?"**
   - Contraction Hierarchies + real-time traffic + historical patterns + ML model (DeepMind) com features temporais e espaciais

---

## Referências

| Recurso | Descrição |
|---------|-----------|
| **Google Maps Blog** | Technical deep dives sobre features |
| **Contraction Hierarchies (Geisberger et al.)** | Paper original do algoritmo |
| **Google S2 Geometry** | Biblioteca geoespacial open-source |
| **Alex Xu — System Design Interview Vol. 2** | Capítulo Google Maps |
| **OSRM (Open Source Routing Machine)** | Implementação open-source de CH |
| **OpenStreetMap** | Dados de mapa open-source |
| **Mapbox Vector Tile Spec** | Especificação de vector tiles |
| **DeepMind + Google Maps (2020)** | ML para previsão de ETA e tráfego |
