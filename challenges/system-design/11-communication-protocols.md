# Level 11 — Communication Protocols: WebSockets, SSE & REST/GraphQL/gRPC

> **Objetivo:** Implementar um servidor que suporte os 3 modelos de comunicação real-time
> (WebSocket, SSE, Long Polling) e os 3 paradigmas de API (REST, GraphQL, gRPC) no mesmo domínio.

**Referência:**
- [24-websockets-long-polling-sse.md](../../.docs/SYSTEM-DESIGN/24-websockets-long-polling-sse.md)
- [25-rest-graphql-grpc.md](../../.docs/SYSTEM-DESIGN/25-rest-graphql-grpc.md)

**Pré-requisito:** Level 10 completo.

---

## Parte 1 — ADR: Communication Protocol Selection

**Arquivo:** `docs/adrs/ADR-001-communication-protocol-selection.md`

**Decisão:** Qual protocolo de comunicação usar para cada tipo de interação.

**Options — Real-time:**
1. **WebSocket** — full-duplex, bidirecional
2. **SSE (Server-Sent Events)** — server push, unidirecional
3. **Long Polling** — client polling com resposta demorada
4. **gRPC Streaming** — server/client/bidirectional streams

**Options — API Paradigm:**
1. **REST** — resources + HTTP verbs
2. **GraphQL** — query language, flexible responses
3. **gRPC** — binary protocol, strongly typed, high performance

**Decision Matrix:**

| Cenário | Protocolo Recomendado | Justificativa |
|---------|----------------------|---------------|
| Chat | WebSocket | Bidirecional, baixa latência |
| Feed de notícias | SSE | Server push, unidirecional |
| Dashboard | Long Polling (fallback) | Compatibilidade com firewalls |
| Serviço interno | gRPC | Performance, contratos fortes |
| API pública | REST | Universal, cacheable |
| Mobile BFF | GraphQL | Flexível, reduz over-fetching |

**Critérios de aceite:**
- [ ] Decision matrix preenchida com justificativas
- [ ] Trade-offs de cada protocolo (latência, overhead, compatibilidade)
- [ ] Fallback strategy documentada
- [ ] Protocol negotiation design

---

## Parte 2 — Diagrama DrawIO

**Arquivo:** `docs/diagrams/11-communication-protocols.drawio`

**View 1 — Protocol Comparison:**
```
WebSocket:    Client ◄═══════════════► Server (full-duplex, persistent)
SSE:          Client ◄─────────────── Server (server push, HTTP)
Long Polling: Client ───request──────► Server (holds, responds, repeat)
gRPC Stream:  Client ◄═══protobuf═══► Server (HTTP/2 streams)
```

**View 2 — Unified Server Architecture:** Servidor que expõe REST + GraphQL + gRPC + WS

**View 3 — Protocol Negotiation & Fallback:** WebSocket → SSE → Long Polling fallback chain

**Critérios de aceite:**
- [ ] Comparação visual dos 4 protocolos real-time
- [ ] Unified server com todos os endpoints
- [ ] Fallback chain documentada

---

## Parte 3 — Implementação

### Domínio: Live Dashboard de Métricas

Um dashboard que mostra métricas em tempo real (CPU, memory, requests/s) de múltiplos servidores. O mesmo domínio é exposto via REST, GraphQL, gRPC e real-time.

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/
│   └── server/main.go              ← Unified server
├── internal/
│   ├── domain/
│   │   ├── metric.go               ← Metric entity
│   │   ├── server.go               ← Server entity
│   │   └── repository.go           ← Interface
│   ├── rest/
│   │   └── handler.go              ← REST API (Gin/Chi)
│   ├── graphql/
│   │   ├── schema.go               ← GraphQL schema
│   │   ├── resolver.go             ← Resolvers
│   │   └── handler.go              ← GraphQL endpoint
│   ├── grpc/
│   │   ├── proto/
│   │   │   └── metrics.proto       ← Protobuf definition
│   │   ├── server.go               ← gRPC server
│   │   └── client.go               ← gRPC client
│   ├── realtime/
│   │   ├── websocket.go            ← WebSocket handler
│   │   ├── sse.go                  ← SSE handler
│   │   ├── longpoll.go             ← Long Polling handler
│   │   └── hub.go                  ← Connection hub (fan-out)
│   └── generator/
│       └── metrics.go              ← Synthetic metric generator
├── api/
│   └── proto/metrics.proto
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **REST API** — CRUD de servers e metrics (GET, POST, PUT, DELETE)
2. **GraphQL** — query metrics com filtros, subscriptions para real-time
3. **gRPC** — serviço de métricas com server streaming
4. **WebSocket** — push de métricas a cada 1s para connected clients
5. **SSE** — stream de métricas (Content-Type: text/event-stream)
6. **Long Polling** — endpoint que responde quando há novas métricas
7. **Connection Hub** — fan-out de métricas para todos os clientes
8. **Protocol negotiation** — Upgrade: websocket ou Accept: text/event-stream
9. **Graceful degradation** — fallback WebSocket → SSE → Long Polling

**Critérios de aceite Go:**
- [ ] REST: CRUD completo com responses JSON
- [ ] GraphQL: queries com filtros + subscriptions
- [ ] gRPC: server streaming de métricas
- [ ] WebSocket: 100+ concurrent connections sem leak
- [ ] SSE: streaming com retry e last-event-id
- [ ] Long Polling: timeout configurável, immediate response em novo dado
- [ ] Hub: fan-out correto para todos os subscribers
- [ ] Benchmark: comparar throughput REST vs gRPC vs WS
- [ ] ≥ 20 testes

---

### 3.2 — Java

**Funcionalidades Java:**
1. **REST** com Spring WebFlux (reactive)
2. **GraphQL** com Spring for GraphQL
3. **gRPC** com grpc-java / Spring gRPC starter
4. **WebSocket** com Spring WebSocket + STOMP
5. **SSE** com Spring WebFlux `Flux<ServerSentEvent>`
6. **Records** para todas as DTOs
7. **Virtual Threads** para connection handling

**Critérios de aceite Java:**
- [ ] REST + GraphQL + gRPC + WS + SSE funcionais
- [ ] Spring WebFlux reactive streams
- [ ] gRPC com protobuf
- [ ] WebSocket com STOMP
- [ ] Testes para cada protocolo
- [ ] JaCoCo ≥ 80%

---

## Parte 4 — Benchmark Comparativo

| Métrica | REST | GraphQL | gRPC | WebSocket | SSE |
|---------|------|---------|------|-----------|-----|
| Latência (p50) | | | | | |
| Latência (p99) | | | | | |
| Throughput (msg/s) | | | | | |
| Overhead/msg (bytes) | | | | | |
| Max connections | | | | | |

---

## Definição de Pronto (DoD)

- [ ] ADR com protocol selection matrix
- [ ] DrawIO com 3 views
- [ ] Go e Java: REST + GraphQL + gRPC + WS + SSE + tests
- [ ] Benchmark comparativo documentado
- [ ] Commit: `feat(system-design-11): communication protocols`
