# Level 19 — WhatsApp / Chat System

> **Objetivo:** Projetar e implementar um sistema de mensagens instantâneas com
> real-time delivery, group chats, read receipts e armazenamento de mensagens.

**Referência:** [40-whatsapp-chat-system.md](../../.docs/SYSTEM-DESIGN/40-whatsapp-chat-system.md)

**Pré-requisito:** Level 18 completo.

---

## Contexto

Sistema de **mensagens em tempo real** entre usuários e grupos. O desafio é manter
conexões persistentes, garantir entrega ordenada de mensagens e suportar indicadores
de presença (online/offline/typing).

**Escala alvo:**
- **2B usuários** registrados
- **100B mensagens/dia**
- **Average message size:** 100 bytes
- **Delivery latency:** < 100ms (online-to-online)
- **Groups:** até 256 membros

---

## Parte 1 — ADRs (4 obrigatórios)

### ADR-001: Real-time Transport Protocol

**Arquivo:** `docs/adrs/ADR-001-transport-protocol.md`

**Options:**
1. **WebSocket** — bidirecional full-duplex
2. **MQTT** — lightweight pub/sub (IoT-origin)
3. **Custom TCP protocol** — protocolo binário proprietário
4. **gRPC streaming** — bidirecional com protobuf

### ADR-002: Message Storage Architecture

**Options:**
1. **Write-optimized store (Cassandra)** — wide column para chat history
2. **PostgreSQL** — relacional com partitioning por chat_id
3. **HBase** — wide column com bom suporte a range scans
4. **Hybrid** — recent messages em Redis, cold em Cassandra

### ADR-003: Message Delivery Guarantees

**Options:**
1. **At-most-once** — fire and forget (sem ACK)
2. **At-least-once** — retry com idempotency (ACK required)
3. **Exactly-once** — deduplicated delivery (message_id-based)

### ADR-004: Group Messaging Strategy

**Options:**
1. **Fan-out per message** — cada mensagem copia para cada membro
2. **Shared inbox** — membros leem de uma fila compartilhada
3. **Pull model** — membros puxam mensagens do grupo
4. **Hybrid** — small groups push, large groups pull

**Critérios de aceite:**
- [ ] 4 ADRs completos
- [ ] WebSocket scalability analysis (connections per server)
- [ ] Message ordering guarantees documentados

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

**Arquivo 1:** `docs/diagrams/19-whatsapp-hld.drawio`

```
┌──────────┐         ┌────────────────────────────────┐
│  Client  │◀═══WS══▶│     Connection Gateway         │
│  (App)   │         │  (WebSocket connection mgmt)   │
└──────────┘         └──────────┬─────────────────────┘
                                │
                     ┌──────────┼──────────┐
                     │          │          │
               ┌─────▼──┐ ┌────▼───┐ ┌───▼──────┐
               │  Chat  │ │Presence│ │  Group   │
               │ Service│ │Service │ │ Service  │
               └───┬────┘ └───┬────┘ └────┬─────┘
                   │          │           │
              ┌────▼───┐ ┌───▼────┐      │
              │  Msg   │ │ Redis  │      │
              │ Queue  │ │(status)│      │
              │(Kafka) │ └────────┘      │
              └───┬────┘                 │
                  │                      │
            ┌─────▼──────────────────────▼───┐
            │     Message Store              │
            │  (Cassandra / PostgreSQL)      │
            └────────────────────────────────┘
```

**Arquivo 2:** `docs/diagrams/19-whatsapp-message-flow.drawio`
- 1-to-1 message: sender → server → recipient (online) with ACK
- 1-to-1 message: sender → server → store (offline) → deliver on reconnect
- Group message: sender → server → fan-out to N members
- Read receipts: double-check flow

**Critérios de aceite:**
- [ ] Online/offline delivery paths
- [ ] Group message fan-out
- [ ] ACK + read receipt flow
- [ ] Presence (typing indicator) flow

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/
│   ├── gateway/main.go            ← WebSocket gateway
│   └── chat/main.go               ← Chat service
├── internal/
│   ├── domain/
│   │   ├── message.go             ← Message entity
│   │   ├── conversation.go        ← 1-to-1 conversation
│   │   ├── group.go               ← Group chat
│   │   ├── user.go                ← User + presence status
│   │   └── receipt.go             ← Read receipt
│   ├── gateway/
│   │   ├── server.go              ← WebSocket server
│   │   ├── connection.go          ← Connection management
│   │   ├── hub.go                 ← Connection hub (user → conn)
│   │   ├── protocol.go            ← Wire protocol (JSON/Protobuf)
│   │   └── server_test.go
│   ├── chat/
│   │   ├── service.go             ← Send, receive, history
│   │   ├── delivery.go            ← Delivery manager (online/offline)
│   │   ├── ordering.go            ← Message ordering (sequence numbers)
│   │   └── service_test.go
│   ├── group/
│   │   ├── service.go             ← Create group, add/remove members
│   │   ├── fanout.go              ← Group message fan-out
│   │   └── service_test.go
│   ├── presence/
│   │   ├── tracker.go             ← Online/offline/typing status
│   │   ├── heartbeat.go           ← Connection heartbeat
│   │   └── tracker_test.go
│   ├── receipt/
│   │   ├── service.go             ← Delivered/read receipts
│   │   └── service_test.go
│   ├── repository/
│   │   ├── message_repo.go        ← PostgreSQL (partitioned by conversation)
│   │   ├── conversation_repo.go   ← PostgreSQL
│   │   ├── group_repo.go          ← PostgreSQL
│   │   ├── offline_repo.go        ← Redis (offline message queue)
│   │   └── presence_repo.go       ← Redis (online status + TTL)
│   └── handler/
│       ├── ws_handler.go          ← WebSocket message routing
│       ├── rest_handler.go        ← REST API (history, groups)
│       └── handler_test.go
├── proto/
│   └── chat.proto                 ← Message wire format
├── docker-compose.yml
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **WebSocket Gateway** — connection hub com goroutine per connection
2. **Wire protocol** — JSON (ou Protobuf para eficiência)
3. **1-to-1 messaging** — send com ACK, retry se perdido
4. **Message ordering** — per-conversation sequence numbers
5. **Offline delivery** — enfileira em Redis, entrega no reconnect
6. **Group messaging** — fan-out para membros online, queue para offline
7. **Read receipts** — sent → delivered → read estado por mensagem
8. **Typing indicator** — broadcast presença via Redis pub/sub
9. **Presence** — online/offline/last_seen via heartbeat
10. **Message history** — pagination por conversation com cursor

**Critérios de aceite Go:**
- [ ] WebSocket: connect/disconnect/reconnect
- [ ] 1-to-1: enviar e receber mensagens em tempo real
- [ ] Offline: mensagens acumuladas, entregues no reconnect
- [ ] Groups: criar grupo, enviar para todos os membros
- [ ] Read receipts: sent → delivered → read
- [ ] Typing indicator funcional
- [ ] Message ordering consistente
- [ ] History com cursor-based pagination
- [ ] ≥ 20 testes
- [ ] Docker Compose: gateway + chat + postgres + redis + kafka

---

### 3.2 — Java (Spring Boot)

**Funcionalidades Java:**
1. **Spring WebSocket** + STOMP para real-time
2. **Spring Data JPA** para mensagens e conversas
3. **Spring Data Redis** para presença e offline queue
4. **Spring Kafka** para message routing entre instâncias
5. **Virtual Threads (JDK 25)** para connection handling
6. **Records** para wire protocol messages

**Critérios de aceite Java:**
- [ ] Mesmas funcionalidades do Go
- [ ] STOMP protocol com destinations por conversation
- [ ] Virtual Threads para escalabilidade
- [ ] Testes com Testcontainers (test WebSocket client)
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] 4 ADRs (transport, storage, delivery, groups)
- [ ] 2 DrawIO (HLD + Message Flow)
- [ ] Go e Java: implementação completa + tests
- [ ] Real-time: mensagem enviada chega em < 100ms
- [ ] Docker Compose: gateway + chat-service + postgres + redis + kafka
- [ ] Commit: `feat(system-design-19): whatsapp chat system`
