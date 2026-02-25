# 42. Uber / Lyft — Ride Sharing

## Índice

- [Visão Geral](#visão-geral)
- [Requisitos](#requisitos)
- [Estimativas (Back-of-the-Envelope)](#estimativas-back-of-the-envelope)
- [Arquitetura de Alto Nível](#arquitetura-de-alto-nível)
- [Location Service](#location-service)
- [Geospatial Indexing](#geospatial-indexing)
- [Matching Service](#matching-service)
- [Trip Service](#trip-service)
- [ETA e Routing](#eta-e-routing)
- [Surge Pricing](#surge-pricing)
- [Real-Time Tracking](#real-time-tracking)
- [Modelo de Dados](#modelo-de-dados)
- [Código Ilustrativo](#código-ilustrativo)
- [Uber Real Architecture](#uber-real-architecture)
- [Trade-offs e Decisões](#trade-offs-e-decisões)
- [Perguntas Comuns em Entrevistas](#perguntas-comuns-em-entrevistas)
- [Referências](#referências)

---

## Visão Geral

Um sistema de ride-sharing conecta **passageiros** (riders) a **motoristas** (drivers) em tempo real, com **matching geoespacial**, **tracking em tempo real**, **cálculo de ETA** e **pagamentos**. É um dos sistemas mais desafiadores por combinar **alta frequência de updates de localização**, **matching em tempo real** e **otimização geoespacial**.

**Big Techs que operam este tipo de sistema:**
- **Uber** — 20M+ viagens/dia em 70+ países
- **Lyft** — mercado norte-americano
- **DiDi** — maior da China (25M+ viagens/dia)
- **Grab** — Sudeste Asiático
- **99** (DiDi) — Brasil

---

## Requisitos

### Funcionais
| Requisito | Descrição |
|-----------|-----------|
| Request ride | Passageiro solicita viagem com pickup/dropoff |
| Match driver | Encontrar motorista mais adequado nearby |
| Real-time tracking | Acompanhar posição do motorista em tempo real |
| ETA calculation | Tempo estimado de chegada preciso |
| Payments | Processar pagamento automaticamente ao fim da viagem |
| Rating | Avaliar motorista e passageiro |
| Trip history | Histórico completo de viagens |
| Surge pricing | Preço dinâmico baseado em demanda |

### Não-Funcionais
| Requisito | Valor |
|-----------|-------|
| Latência de matching | < 1 segundo |
| ETA accuracy | ±2 minutos |
| Rides/dia | 20M (Uber-scale) |
| Location update frequency | A cada 3-4 segundos |
| Disponibilidade | 99.99% |
| Consistência | Eventual (location), Strong (payment) |

---

## Estimativas (Back-of-the-Envelope)

```
Motoristas:
  Total registrados: 5M
  Online simultâneos (peak): 1M
  Location updates: 1M × every 3s = 333K updates/segundo

Viagens:
  20M rides/dia
  Peak QPS: 20M / 86400 × 3 (peak factor) ≈ 700 rides/segundo

Storage:
  Location history/driver/dia: 4s GPS × 28,800 updates = ~1 MB
  All drivers/dia: 5M × 1 MB = 5 TB (se armazenar tudo)
  Trip data: 20M × 2 KB = 40 GB/dia

Bandwidth:
  Location updates (incoming): 333K × 100 bytes = 33 MB/s
  Map tiles + tracking (outgoing): muito maior (CDN-cached)
```

---

## Arquitetura de Alto Nível

```
┌──────────────────────────────────────────────────────────────────────┐
│                        RIDE SHARING PLATFORM                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────┐            ┌──────────────┐          ┌────────────┐  │
│  │ Rider App │───────────▶│  API Gateway │◀─────────│ Driver App │  │
│  └───────────┘            └──────┬───────┘          └──────┬─────┘  │
│                                  │                         │        │
│         ┌────────────────────────┼─────────────────────────┤        │
│         │                        │                         │        │
│         ▼                        ▼                         ▼        │
│  ┌──────────────┐    ┌───────────────────┐    ┌────────────────┐   │
│  │  Trip Service │    │ Matching Service  │    │Location Service│   │
│  │ (lifecycle)   │    │ (find best driver)│    │(GPS updates)   │   │
│  └──────┬───────┘    └────────┬──────────┘    └────────┬───────┘   │
│         │                     │                         │           │
│         │              ┌──────▼──────────┐     ┌────────▼──────┐   │
│         │              │  Geospatial     │     │  Location     │   │
│         │              │  Index          │     │  Store        │   │
│         │              │ (Geohash/S2/QT) │     │  (Redis)      │   │
│         │              └─────────────────┘     └───────────────┘   │
│         │                                                          │
│         ▼                                                          │
│  ┌──────────────────────────────────────────┐                      │
│  │            Supporting Services            │                      │
│  ├──────────┬──────────┬──────────┬─────────┤                      │
│  │ Payment  │ Pricing  │ Routing  │Notific. │                      │
│  │ Service  │ (Surge)  │ (ETA)   │ Service │                      │
│  └──────────┴──────────┴──────────┴─────────┘                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Location Service

O Location Service é o componente mais write-heavy do sistema:

```
┌──────────────────────────────────────────────────────────┐
│                  LOCATION SERVICE                         │
│                                                           │
│  Driver App ──GPS update──▶ [Location Service]            │
│  (every 3-4s)               │                             │
│                              ├── Update Redis             │
│                              │   Key: driver:{id}:loc     │
│                              │   Val: {lat, lng, ts,      │
│                              │         heading, speed}     │
│                              │   TTL: 30s                  │
│                              │                             │
│                              ├── Update Geospatial Index   │
│                              │   (para matching queries)   │
│                              │                             │
│                              └── Publish to Kafka           │
│                                  (para analytics, tracking) │
│                                                           │
│  Throughput: 333K writes/segundo (peak)                   │
│  Latency: < 10ms per update                              │
│  Storage: Redis (in-memory, TTL-based)                    │
└──────────────────────────────────────────────────────────┘
```

### Por que Redis?

```
✅ In-memory = latência < 1ms
✅ TTL automático = cleanup de drivers offline
✅ GEO commands nativos: GEOADD, GEORADIUS, GEOSEARCH
✅ 100K+ ops/s por instância
✅ Cluster mode para escalar horizontalmente
```

---

## Geospatial Indexing

O coração do matching é encontrar motoristas próximos eficientemente:

### Abordagens

| Estrutura | Como Funciona | Prós | Contras |
|-----------|---------------|------|---------|
| **Geohash** | Divide mapa em grid cells com string prefix | Simples, range queries SQL-friendly | Boundary problem, grid fixo |
| **Google S2** | Projeta esfera em cubo, cells hierárquicas | Áreas uniformes, covering preciso | Complexo de implementar |
| **Quadtree** | Divide espaço em 4 recursivamente | Adaptável à densidade | Harder to distribute |
| **R-Tree** | B-tree para dados espaciais | Range queries ótimas | Updates custosos |

### Geohash em Detalhe

```
┌────────────────────────────────────────────────┐
│              GEOHASH GRID                       │
│                                                 │
│  Precisão e Tamanho:                            │
│  ┌──────────────┬──────────────┬───────────┐   │
│  │ Caracteres   │ Cell Size    │ Uso       │   │
│  ├──────────────┼──────────────┼───────────┤   │
│  │ 4            │ ~39km × 20km │ Região    │   │
│  │ 5            │ ~5km × 5km   │ Cidade    │   │
│  │ 6            │ ~1.2km×0.6km │ Bairro    │   │
│  │ 7            │ ~150m × 150m │ Quarteirão│   │
│  │ 8            │ ~38m × 19m   │ Esquina   │   │
│  └──────────────┴──────────────┴───────────┘   │
│                                                 │
│  Exemplo: São Paulo, Brasil                     │
│  (-23.5505, -46.6333) → geohash: "6gycf"       │
│                                                 │
│  Query "motoristas em 3km":                     │
│  1. Calcular geohash do rider (precision 6)     │
│  2. Buscar drivers no cell + 8 cells adjacentes │
│  3. Filtrar por distância real (Haversine)       │
│                                                 │
│  ┌───┬───┬───┐                                  │
│  │ A │ B │ C │  9 cells total                   │
│  ├───┼───┼───┤  (center + 8 neighbors)          │
│  │ D │ X │ E │  X = rider's cell                │
│  ├───┼───┼───┤                                  │
│  │ F │ G │ H │                                  │
│  └───┴───┴───┘                                  │
└────────────────────────────────────────────────┘
```

### Google S2 (Uber real)

```
Uber usa Google S2 Geometry Library:

• Esfera → projeção em 6 faces de cubo
• Cada face subdividida recursivamente (Hilbert curve)
• S2 Cell IDs: uint64 com nível de precisão 0-30
• Level 12 ≈ 3.31km² — ideal para matching urbano
• Level 16 ≈ 0.05km² — ideal para ETA preciso

Vantagens do S2:
✅ Áreas mais uniformes que Geohash (sem distorção nos polos)
✅ Region covering: dado um círculo, retorna set mínimo de cells
✅ Hilbert curve: cells próximas no ID são próximas no espaço
✅ Nativo em Go (linguagem principal do Uber)
```

---

## Matching Service

```
┌───────────────────────────────────────────────────────────┐
│                  MATCHING ALGORITHM                        │
│                                                            │
│  1. Rider solicita viagem                                  │
│     {pickup: (-23.55, -46.63), type: "UberX"}             │
│                                                            │
│  2. Query Geospatial Index                                 │
│     "Motoristas available dentro de 3km do pickup"         │
│     → Retorna: [D1(0.5km), D3(1.2km), D7(2.8km)]        │
│                                                            │
│  3. Filtrar                                                │
│     ✓ Status = AVAILABLE                                   │
│     ✓ Vehicle type = UberX                                 │
│     ✓ Rating ≥ 4.0                                        │
│     ✓ Acceptance rate ≥ 70%                                │
│                                                            │
│  4. Ranquear (scoring function)                            │
│     Score = w1 × (1/ETA) +                                │
│             w2 × rating +                                  │
│             w3 × acceptance_rate +                         │
│             w4 × (1/surge_in_area)                         │
│                                                            │
│  5. Dispatch: enviar request para top driver               │
│     → Driver tem 15s para aceitar                          │
│     → Se não aceita → próximo driver                       │
│     → Se ninguém aceita → expandir raio para 5km           │
│                                                            │
│  6. Match confirmado → Trip criada                         │
└───────────────────────────────────────────────────────────┘
```

### Batch Matching vs Greedy

```
Greedy (Uber early days):
  Rider chega → match com melhor driver disponível
  ✅ Simples
  ❌ Matching global subótimo

Batch Matching (Uber atual):
  Acumular riders + drivers por N segundos (ex: 2s)
  Resolver matching como problema de otimização (Hungarian algorithm)
  ✅ Matching globalmente ótimo
  ❌ 2s extra de latência
  
  Uber: "batched matching reduces ETAs by ~20%"
```

---

## Trip Service

```
┌──────────────────────────────────────────────────┐
│              TRIP STATE MACHINE                    │
│                                                   │
│  ┌──────────┐    ┌───────────┐    ┌───────────┐ │
│  │REQUESTING│───▶│  MATCHING │───▶│ DRIVER_   │ │
│  │          │    │           │    │ ASSIGNED  │ │
│  └──────────┘    └───────────┘    └─────┬─────┘ │
│                        │                 │       │
│                        ▼                 ▼       │
│                  ┌───────────┐    ┌───────────┐ │
│                  │ NO_DRIVER │    │ ARRIVING  │ │
│                  │ _FOUND    │    │           │ │
│                  └───────────┘    └─────┬─────┘ │
│                                         │       │
│                                  ┌──────▼─────┐ │
│                                  │  PICKED_UP │ │
│                                  │            │ │
│                                  └──────┬─────┘ │
│                                         │       │
│                                  ┌──────▼─────┐ │
│                                  │ IN_PROGRESS│ │
│                                  │            │ │
│                                  └──────┬─────┘ │
│                                    ┌────┴────┐  │
│                                    ▼         ▼  │
│                              ┌─────────┐┌──────┐│
│                              │COMPLETED││CANCEL││
│                              │         ││LED   ││
│                              └─────────┘└──────┘│
└──────────────────────────────────────────────────┘
```

---

## ETA e Routing

```
┌──────────────────────────────────────────────────┐
│              ETA CALCULATION                       │
│                                                   │
│  Camadas:                                        │
│                                                   │
│  1. Road Network Graph                            │
│     • Nós: interseções                           │
│     • Arestas: segmentos de rua                   │
│     • Peso: tempo de viagem (não distância)       │
│     • Dados: OpenStreetMap + dados proprietários  │
│                                                   │
│  2. Routing Algorithm                             │
│     ┌───────────────────────────────────────┐    │
│     │ Contraction Hierarchies (CH)          │    │
│     │ • Pre-processa atalhos no grafo       │    │
│     │ • Query time: microseconds            │    │
│     │ • Usado pela maioria dos serviços     │    │
│     │                                       │    │
│     │ A* com CH:                            │    │
│     │ • Busca bidirecional                  │    │
│     │ • Heurística geográfica               │    │
│     │ • ~100x mais rápido que Dijkstra      │    │
│     └───────────────────────────────────────┘    │
│                                                   │
│  3. Real-Time Traffic Adjustment                  │
│     • GPS probes de drivers → speed por segmento │
│     • Atualiza pesos do grafo a cada 1-2 min     │
│     • Historical patterns para previsão          │
│                                                   │
│  4. ML Model (Uber DeepETA)                       │
│     • Features: hora, dia, clima, eventos        │
│     • Historical trip data                       │
│     • Road-level features                         │
│     • Output: ETA em minutos (±2 min accuracy)   │
└──────────────────────────────────────────────────┘
```

---

## Surge Pricing

```
┌──────────────────────────────────────────────────┐
│              SURGE PRICING                        │
│                                                   │
│  Objetivo: Equilibrar supply (drivers) e          │
│            demand (riders)                         │
│                                                   │
│  ┌─────────────────────────────┐                 │
│  │   Calcular por zona (S2 cell)                │ │
│  │                              │                 │
│  │   demand = ride_requests/min │                 │
│  │   supply = available_drivers │                 │
│  │                              │                 │
│  │   ratio = demand / supply    │                 │
│  │                              │                 │
│  │   Se ratio > threshold:      │                 │
│  │     surge = f(ratio)         │                 │
│  │     ex: 1.5x, 2.0x, 3.5x   │                 │
│  └─────────────────────────────┘                 │
│                                                   │
│  Atualização: A cada 1-2 minutos                 │
│  Granularidade: Hexagonal cells (H3) ou S2 cells │
│  Smoothing: Média móvel para evitar oscilação    │
│                                                   │
│  Uber Surge Map:                                 │
│  ┌─────────────────────────┐                     │
│  │  ░░░░░░░░░░░░░░░░░░░░ │ ░ = 1.0x (normal)  │
│  │  ░░░░▒▒▒▒░░░░░░░░░░░ │ ▒ = 1.5x           │
│  │  ░░▒▒▓▓▓▓▒▒░░░░░░░░ │ ▓ = 2.0x           │
│  │  ░░▒▒▓▓██▓▓▒▒░░░░░░ │ █ = 3.0x+ (evento) │
│  │  ░░░░▒▒▓▓▒▒░░░░░░░░ │                     │
│  │  ░░░░░░▒▒░░░░░░░░░░░ │                     │
│  └─────────────────────────┘                     │
└──────────────────────────────────────────────────┘
```

---

## Real-Time Tracking

```
┌──────────────────────────────────────────────────┐
│            REAL-TIME TRACKING                     │
│                                                   │
│  Driver App                   Rider App           │
│  ┌─────────┐                 ┌─────────┐         │
│  │ GPS     │                 │ Map     │         │
│  │ Sensor  │                 │ View    │         │
│  └────┬────┘                 └────▲────┘         │
│       │ every 3-4s                │              │
│       ▼                          │              │
│  ┌─────────────┐          ┌──────┴──────┐       │
│  │  Location   │──Kafka──▶│  Tracking   │       │
│  │  Service    │          │  Service    │       │
│  └─────────────┘          └─────────────┘       │
│                                                   │
│  Protocolo para Rider:                            │
│  Option A: WebSocket (full-duplex, persistent)    │
│  Option B: Server-Sent Events (SSE, read-only)    │
│  Option C: HTTP Polling every 2s                  │
│                                                   │
│  Uber usa: gRPC bidirectional streaming           │
│  (eficiente, multiplexed, HTTP/2 based)          │
└──────────────────────────────────────────────────┘
```

---

## Modelo de Dados

### Trip (PostgreSQL)

```sql
CREATE TABLE trips (
    trip_id         UUID PRIMARY KEY,
    rider_id        UUID NOT NULL,
    driver_id       UUID,
    status          VARCHAR(20),   -- 'requesting', 'matched', 'in_progress', 'completed'
    vehicle_type    VARCHAR(20),   -- 'economy', 'comfort', 'premium'
    
    -- Locations
    pickup_lat      DECIMAL(10, 7),
    pickup_lng      DECIMAL(10, 7),
    pickup_address  TEXT,
    dropoff_lat     DECIMAL(10, 7),
    dropoff_lng     DECIMAL(10, 7),
    dropoff_address TEXT,
    
    -- Pricing
    estimated_fare  DECIMAL(10, 2),
    actual_fare     DECIMAL(10, 2),
    surge_multiplier DECIMAL(3, 2) DEFAULT 1.0,
    currency        VARCHAR(3),
    
    -- Timestamps
    requested_at    TIMESTAMP WITH TIME ZONE,
    matched_at      TIMESTAMP WITH TIME ZONE,
    pickup_at       TIMESTAMP WITH TIME ZONE,
    dropoff_at      TIMESTAMP WITH TIME ZONE,
    
    -- Metrics
    distance_km     DECIMAL(8, 2),
    duration_min    DECIMAL(8, 2),
    
    -- Ratings
    rider_rating    SMALLINT CHECK (rider_rating BETWEEN 1 AND 5),
    driver_rating   SMALLINT CHECK (driver_rating BETWEEN 1 AND 5)
);

-- Índices para queries comuns
CREATE INDEX idx_trips_rider ON trips(rider_id, requested_at DESC);
CREATE INDEX idx_trips_driver ON trips(driver_id, requested_at DESC);
CREATE INDEX idx_trips_status ON trips(status) WHERE status IN ('requesting', 'matched', 'in_progress');
```

### Driver Location (Redis)

```redis
# Última posição do driver (hash)
HSET driver:loc:d123 lat -23.5505 lng -46.6333 heading 180 speed 45 ts 1700000000

# Geo index (sorted set interno)
GEOADD drivers:available -46.6333 -23.5505 d123
GEOADD drivers:available -46.6400 -23.5600 d456

# Query: motoristas em 3km do rider
GEOSEARCH drivers:available FROMLONLAT -46.6350 -23.5520 BYRADIUS 3 km ASC COUNT 20
# → ["d123", "d456"]
```

---

## Código Ilustrativo

### Geospatial Matching (Go)

```go
package matching

import (
    "context"
    "math"
    "sort"
    "time"
)

type Location struct {
    Lat float64 `json:"lat"`
    Lng float64 `json:"lng"`
}

type Driver struct {
    ID           string
    Location     Location
    VehicleType  string
    Rating       float64
    AcceptRate   float64
    IsAvailable  bool
}

type RideRequest struct {
    RiderID     string
    Pickup      Location
    Dropoff     Location
    VehicleType string
}

type MatchResult struct {
    DriverID string
    ETA      time.Duration
    Distance float64
    Score    float64
}

type MatchingService struct {
    geoIndex   GeoIndexer      // Geospatial index (S2/Geohash)
    etaService ETACalculator
    maxRadius  float64         // km
    maxResults int
}

func (s *MatchingService) FindBestDriver(
    ctx context.Context, req RideRequest,
) (*MatchResult, error) {
    // 1. Query nearby drivers
    candidates, err := s.geoIndex.DriversInRadius(
        ctx, req.Pickup, s.maxRadius,
    )
    if err != nil {
        return nil, err
    }

    // 2. Filter
    var eligible []Driver
    for _, d := range candidates {
        if d.IsAvailable &&
            d.VehicleType == req.VehicleType &&
            d.Rating >= 4.0 &&
            d.AcceptRate >= 0.7 {
            eligible = append(eligible, d)
        }
    }

    if len(eligible) == 0 {
        return nil, ErrNoDriversAvailable
    }

    // 3. Score and rank
    results := make([]MatchResult, 0, len(eligible))
    for _, d := range eligible {
        eta, _ := s.etaService.Calculate(ctx, d.Location, req.Pickup)
        dist := haversine(d.Location, req.Pickup)
        
        score := computeScore(d, eta, dist)
        results = append(results, MatchResult{
            DriverID: d.ID,
            ETA:      eta,
            Distance: dist,
            Score:    score,
        })
    }

    sort.Slice(results, func(i, j int) bool {
        return results[i].Score > results[j].Score
    })

    return &results[0], nil
}

func computeScore(d Driver, eta time.Duration, dist float64) float64 {
    etaMinutes := eta.Minutes()
    if etaMinutes == 0 {
        etaMinutes = 0.1
    }
    return (1.0 / etaMinutes) * 0.5 +
        d.Rating * 0.2 +
        d.AcceptRate * 0.2 +
        (1.0 / (dist + 0.1)) * 0.1
}

// Haversine formula: distância entre dois pontos na esfera terrestre
func haversine(a, b Location) float64 {
    const R = 6371.0 // raio da Terra em km
    dLat := (b.Lat - a.Lat) * math.Pi / 180
    dLng := (b.Lng - a.Lng) * math.Pi / 180
    
    lat1 := a.Lat * math.Pi / 180
    lat2 := b.Lat * math.Pi / 180
    
    h := math.Sin(dLat/2)*math.Sin(dLat/2) +
        math.Cos(lat1)*math.Cos(lat2)*math.Sin(dLng/2)*math.Sin(dLng/2)
    
    return 2 * R * math.Asin(math.Sqrt(h))
}
```

### Location Update Handler (Python)

```python
import redis
import json
import time
from dataclasses import dataclass, asdict
from typing import Optional

@dataclass
class LocationUpdate:
    driver_id: str
    lat: float
    lng: float
    heading: float
    speed: float
    timestamp: float

class LocationService:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.geo_key = "drivers:available"
        self.loc_prefix = "driver:loc:"
        self.ttl = 30  # seconds — driver considered offline after 30s
    
    def update_location(self, update: LocationUpdate) -> None:
        """Processa location update de um driver."""
        pipe = self.redis.pipeline()
        
        # 1. Atualizar posição atual (hash)
        key = f"{self.loc_prefix}{update.driver_id}"
        pipe.hset(key, mapping={
            "lat": update.lat,
            "lng": update.lng,
            "heading": update.heading,
            "speed": update.speed,
            "ts": update.timestamp,
        })
        pipe.expire(key, self.ttl)
        
        # 2. Atualizar geo index
        pipe.geoadd(self.geo_key, (update.lng, update.lat, update.driver_id))
        
        # 3. Executar pipeline (1 round-trip)
        pipe.execute()
    
    def find_nearby_drivers(
        self, lat: float, lng: float, radius_km: float, count: int = 20
    ) -> list[dict]:
        """Busca motoristas próximos usando GEOSEARCH."""
        results = self.redis.geosearch(
            self.geo_key,
            longitude=lng,
            latitude=lat,
            radius=radius_km,
            unit="km",
            sort="ASC",
            count=count,
            withcoord=True,
            withdist=True,
        )
        
        drivers = []
        for driver_id, dist, (d_lng, d_lat) in results:
            drivers.append({
                "driver_id": driver_id,
                "lat": d_lat,
                "lng": d_lng,
                "distance_km": dist,
            })
        return drivers
    
    def remove_driver(self, driver_id: str) -> None:
        """Remove driver do geo index (ficou offline)."""
        self.redis.zrem(self.geo_key, driver_id)
        self.redis.delete(f"{self.loc_prefix}{driver_id}")
```

---

## Uber Real Architecture

```
┌──────────────────────────────────────────────────────────┐
│                UBER ARCHITECTURE (2024)                    │
│                                                           │
│  ┌─────────────────────────────────────────────┐         │
│  │              Frontend                        │         │
│  │  • Rider App (React Native)                 │         │
│  │  • Driver App (React Native)                │         │
│  │  • Web (Next.js)                            │         │
│  └──────────────────┬──────────────────────────┘         │
│                     │                                     │
│  ┌──────────────────▼──────────────────────────┐         │
│  │        Edge / API Layer                      │         │
│  │  • API Gateway (custom, gRPC-based)         │         │
│  │  • Edge authentication                      │         │
│  └──────────────────┬──────────────────────────┘         │
│                     │                                     │
│  ┌──────────────────▼──────────────────────────┐         │
│  │         Core Services (~4000 microservices)  │         │
│  │                                              │         │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐    │         │
│  │  │ Dispatch │ │ Pricing  │ │ Routing  │    │         │
│  │  │ (Go)     │ │ (Java)   │ │ (C++)    │    │         │
│  │  └──────────┘ └──────────┘ └──────────┘    │         │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐    │         │
│  │  │ Maps     │ │ Payment  │ │  Safety  │    │         │
│  │  │ (Go)     │ │ (Java)   │ │ (Python) │    │         │
│  │  └──────────┘ └──────────┘ └──────────┘    │         │
│  └─────────────────────────────────────────────┘         │
│                                                           │
│  ┌─────────────────────────────────────────────┐         │
│  │              Data Layer                      │         │
│  │  • MySQL (Vitess/Docstore) — transactional  │         │
│  │  • Cassandra — time-series, location        │         │
│  │  • Redis — caching, geo index               │         │
│  │  • Kafka — event streaming                  │         │
│  │  • Hive/Presto — analytics                  │         │
│  │  • H3 — hexagonal geospatial indexing       │         │
│  └─────────────────────────────────────────────┘         │
└──────────────────────────────────────────────────────────┘

Dados técnicos reais do Uber:
• ~4000 microservices
• ~2000 engenheiros
• Linguagens: Go (maioria), Java, Python, Node.js, C++
• H3: hexagonal grid system (open-source pelo Uber)
• Schemaless (on top of MySQL): distributed datastore
• Ringpop: consistent hashing + gossip (open-source)
• TChannel: RPC protocol (open-source, predecessor gRPC interno)
```

---

## Trade-offs e Decisões

| Decisão | Opção A | Opção B | Escolha Recomendada |
|---------|---------|---------|--------------------|
| Geo index | Geohash | S2 / H3 | S2/H3 (mais preciso) |
| Location store | Redis | Cassandra | Redis (latência) |
| Matching | Greedy | Batch | Batch (melhor global) |
| Driver-rider comm | WebSocket | gRPC stream | gRPC stream (eficiente) |
| Tracking protocol | Polling | SSE/WebSocket | WebSocket (bidirecional) |
| ETA | Graph-only | Graph + ML | Graph + ML (Uber DeepETA) |
| Pricing | Fixed | Dynamic (surge) | Dynamic (equilibra S&D) |
| Trip DB | PostgreSQL | Cassandra | PostgreSQL (ACID para $$$) |

---

## Perguntas Comuns em Entrevistas

1. **"Como encontrar motoristas próximos de forma eficiente?"**
   - Geospatial index (S2/H3/Geohash), Redis GEOSEARCH, query center cell + neighbors, filter + rank

2. **"Como lidar com 333K location updates/segundo?"**
   - Redis cluster (in-memory), pipeline batching, Kafka para async processing, TTL para cleanup automático

3. **"Como o matching funciona?"**
   - Query nearby → filter (available, vehicle type) → score (ETA, rating, acceptance) → dispatch to top; batch matching para otimizar globalmente

4. **"E se o driver recusar?"**
   - Timeout de 15s → próximo driver no ranking → expandir raio se necessário → "no drivers" se esgotou

5. **"Como calcular ETA preciso?"**
   - Road graph + Contraction Hierarchies + real-time traffic (GPS probes) + ML model (features: hora, dia, clima, eventos)

6. **"Como surge pricing evita abuso?"**
   - Smoothing (média móvel), cap máximo, transparência para rider, recalcular a cada 1-2 min, granularidade geográfica fina

---

## Referências

| Recurso | Descrição |
|---------|-----------|
| **Uber Engineering Blog** | Dispatch, H3, DeepETA, Schemaless |
| **H3: Hexagonal Hierarchical Geospatial Index** | Sistema de grid do Uber (open-source) |
| **Google S2 Geometry** | Geospatial library usada no core |
| **Alex Xu — System Design Interview Vol. 2** | Capítulo proximity service |
| **Designing Data-Intensive Applications** | Cap. sobre partitioning e replicação |
| **Uber DeepETA Paper** | ML model para previsão de ETA |
| **Ringpop** | Consistent hashing library do Uber |
| **Real-Time Bidding at Uber** | Batch matching algorithms |
