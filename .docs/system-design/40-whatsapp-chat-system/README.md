# 40. WhatsApp / Messenger — Chat System

> **Categoria:** Classic System Design  
> **Nível:** Pergunta frequente em entrevistas — Meta, Google, Uber, Amazon  
> **Complexidade:** Alta

---

## Definição

Design de um **sistema de chat em tempo real** como WhatsApp ou Messenger, suportando mensagens 1:1, grupos, presença (online/offline), entrega garantida, e criptografia end-to-end. O desafio central é manter **conexões persistentes** com bilhões de dispositivos e garantir **entrega confiável** de mensagens.

---

## Requisitos

### Funcionais

```
1. Mensagens 1:1 (texto, imagem, vídeo, áudio, documento)
2. Mensagens em grupo (até 1024 membros)
3. Status de entrega: sent ✓, delivered ✓✓, read ✓✓ (azul)
4. Presença: online, offline, last seen, typing...
5. Push notifications (quando app está em background)
6. Histórico de mensagens (sincronizado entre dispositivos)
7. Criptografia end-to-end (E2E)
8. Media sharing (fotos, vídeos, documentos)
```

### Não-Funcionais

```
1. Baixa latência: mensagem entregue em < 500ms (same region)
2. Alta disponibilidade: 99.99% (WhatsApp: quase nenhum downtime)
3. Entrega garantida: zero message loss (at-least-once)
4. Consistência de ordering: mensagens na ordem correta por conversa
5. Escala: 2B users, 100B mensagens/dia
6. Minimal server-side storage (E2E = server não lê conteúdo)
```

---

## Estimativas

```
Scale:
  DAU: 1B
  Messages/dia: 100B
  QPS: 100B / 86,400 ≈ 1.2M QPS
  Peak: ~3.5M QPS

Connections:
  Concurrent users (peak): 500M
  WebSocket connections: 500M simultâneas
  Se cada server suporta 50K connections:
    Servers = 500M / 50K = 10,000 connection servers

Storage:
  Avg message: 100 bytes
  100B × 100B = 10 TB/dia (texto only)
  Com mídia: muito mais → S3/CDN

  Cassandra storage: 10 TB/dia × 30 dias = 300 TB
  (WhatsApp deleta do server após delivery)
```

---

## Arquitetura

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                                                                      │
  │  ┌──────────┐      ┌────────────────────────────────────────┐        │
  │  │ Client A │◀────▶│  WebSocket Connection Gateway           │        │
  │  │ (mobile) │  WS  │  (manages persistent connections)       │        │
  │  └──────────┘      │                                        │        │
  │                    │  - 10K+ servers                         │        │
  │  ┌──────────┐      │  - Each handles ~50K connections        │        │
  │  │ Client B │◀────▶│  - Stateful: knows which user on which│        │
  │  │ (mobile) │  WS  │    server                              │        │
  │  └──────────┘      └──────────────┬─────────────────────────┘        │
  │                                   │                                   │
  │                    ┌──────────────▼──────────────────────────┐        │
  │                    │  Session Service                        │        │
  │                    │  (maps user_id → gateway server)        │        │
  │                    │  Redis: session:user123 → gateway-42    │        │
  │                    └──────────────┬─────────────────────────┘        │
  │                                   │                                   │
  │                    ┌──────────────▼──────────────────────────┐        │
  │                    │  Message Service                        │        │
  │                    │  - Route messages to correct gateway    │        │
  │                    │  - Persist to Message Store             │        │
  │                    │  - Group message fan-out                │        │
  │                    │  - Delivery receipts                    │        │
  │                    └──────────────┬─────────────────────────┘        │
  │                                   │                                   │
  │          ┌────────────────────────┼──────────────────────┐           │
  │          ▼                        ▼                      ▼           │
  │  ┌────────────────┐   ┌────────────────┐   ┌─────────────────────┐  │
  │  │ Message Store  │   │ Presence        │   │ Push Notification   │  │
  │  │ (Cassandra)    │   │ Service         │   │ Service             │  │
  │  │                │   │                 │   │                     │  │
  │  │ Partition key: │   │ Online/Offline  │   │ APNs (iOS)         │  │
  │  │ conversation_id│   │ Last Seen       │   │ FCM (Android)      │  │
  │  │ Cluster key:   │   │ Typing status   │   │                     │  │
  │  │ message_id     │   │ (Redis pub/sub) │   │ When user offline:  │  │
  │  └────────────────┘   └────────────────┘   │ send push to device │  │
  │                                             └─────────────────────┘  │
  │                                                                      │
  │  ┌────────────────────────────────────────────────────────────┐      │
  │  │  Media Service                                             │      │
  │  │  - Upload: Client → Pre-signed URL → S3                   │      │
  │  │  - Download: S3 → CDN → Client                            │      │
  │  │  - Thumbnail generation (async)                            │      │
  │  │  - Encryption: media encrypted client-side (E2E)          │      │
  │  └────────────────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## Message Flow: 1:1 Chat

```
  User A envia "Hello!" para User B:

  ┌─────────┐   1. WS send     ┌─────────────────┐
  │ User A  │──────────────────▶│ Gateway Server 7│
  │ (online)│                   │ (A's connection)│
  └─────────┘                   └────────┬────────┘
                                         │
                                    2. Route to Message Service
                                         │
                                ┌────────▼─────────┐
                                │ Message Service   │
                                │                   │
                                │ a. Validate msg   │
                                │ b. Generate msg_id│  
                                │    (Snowflake)    │
                                │ c. Persist to     │
                                │    Cassandra      │
                                │ d. Lookup: where  │
                                │    is User B?     │
                                └───┬───────────┬───┘
                                    │           │
                          User B online?    User B offline?
                                    │           │
                               ┌────▼────┐  ┌───▼────────────┐
                               │ Session │  │ Push Service   │
                               │ Service │  │ (APNs / FCM)  │
                               │ → GW-12 │  │ → Push notif   │
                               └────┬────┘  │ + Store in     │
                                    │       │   offline queue │
                          ┌─────────▼──────┐└────────────────┘
                          │ Gateway Srv 12 │
                          │ (B's WS conn)  │
                          └────────┬───────┘
                                   │
                              3. WS push
                                   │
                          ┌────────▼───────┐
                          │    User B      │
                          │   (online)     │
                          └────────────────┘

  Delivery Receipts:
  
  ✓  Sent:      Server received and persisted
  ✓✓ Delivered: User B's device received the message
  ✓✓ Read:      User B opened the conversation (blue ticks)
  
  Each receipt is a separate message back through the same pipeline:
  B → Gateway → Message Service → update status → Gateway → A
```

---

## Group Messages

```
  Group "Family" com 50 membros:
  
  User A envia msg no grupo:
  
  1. Gateway recebe msg de A
  2. Message Service:
     a. Lookup grupo "Family" → [members: A, B, C, ..., Z]
     b. Persistir msg (partition: group_conversation_id)
     c. Para CADA membro (exceto sender):
        - Lookup Session Service: onde está?
        - Online → enviar via Gateway
        - Offline → Push notification + offline queue
  
  ┌─────────┐     ┌──────────────┐     ┌────────────────────────┐
  │ User A  │────▶│ Message Svc  │────▶│ Fan-out para 49 members│
  │ "Hello" │     │              │     │                        │
  └─────────┘     │ Persist msg  │     │ B → Gateway-5  (online)│
                  │              │     │ C → Gateway-12 (online)│
                  └──────────────┘     │ D → Push (offline)     │
                                       │ E → Gateway-3  (online)│
                                       │ ...                     │
                                       └────────────────────────┘
  
  Otimização para grupos grandes (1000+ members):
  - Batch fan-out (não 1 msg per member, mas batch per gateway)
  - Se 100 members estão no Gateway-5:
    → 1 message to Gateway-5, Gateway faz local broadcast
  - Lazy delivery: offline members recebem quando conectam
```

---

## Componentes em Detalhe

### WebSocket Connection Gateway

```
  Responsável por manter conexões persistentes:
  
  ┌──────────────────────────────────────────────────────┐
  │  WebSocket Gateway Server                            │
  │                                                      │
  │  - ~50K concurrent connections per server            │
  │  - Epoll/kqueue para event-driven I/O                │
  │  - Heartbeat: client pings every 30s                 │
  │    → Se no ping por 60s → mark offline               │
  │                                                      │
  │  Connection lifecycle:                                │
  │  1. Client connects via WS (wss://chat.app/ws)       │
  │  2. Client authenticates (JWT token)                  │
  │  3. Server registers in Session Service:              │
  │     SET session:user123 → gateway-42                  │
  │  4. Client receives queued offline messages           │
  │  5. Bidirectional messaging via WS                    │
  │  6. Disconnect → cleanup Session Service              │
  │                                                      │
  │  Reconnection:                                        │
  │  - Client auto-reconnects with exponential backoff    │
  │  - Resume from last_received_message_id               │
  │  - Server delivers missed messages                    │
  └──────────────────────────────────────────────────────┘
```

### Session Service (Redis)

```
  Maps user → which gateway server they're connected to:
  
  SET session:user123 → "gateway-42"    (TTL: 120s, refreshed by heartbeat)
  SET session:user456 → "gateway-7"
  SET session:user789 → null            (offline)
  
  Lookup: O(1) per user
  Scale: 500M online users × ~50 bytes = ~25 GB → Redis cluster
```

### Presence Service

```
  Gerencia status online/offline/typing:
  
  ┌──────────────────────────────────────────────────────┐
  │  Presence States:                                     │
  │                                                       │
  │  Online:   WS connected + recent heartbeat            │
  │  Offline:  WS disconnected OR no heartbeat > 60s      │
  │  Last Seen: timestamp do último heartbeat             │
  │  Typing:   transient state, expires in 5s             │
  │                                                       │
  │  Publish/Subscribe:                                   │
  │  - User A opens chat with B                           │
  │  - Client subscribes to presence:user_B               │
  │  - When B's status changes → publish to subscribers   │
  │                                                       │
  │  Redis Pub/Sub ou dedicated presence channel:          │
  │  SUBSCRIBE presence:user_B                            │
  │  → Receives: { status: "online" }                     │
  │  → Receives: { status: "typing" }                     │
  │  → Receives: { status: "offline", last_seen: ts }     │
  │                                                       │
  │  Otimização para grupos:                              │
  │  - Não broadcast presença para todos os membros        │
  │  - Só para quem tem o chat ABERTO                     │
  │  - Lazy: fetch on-demand quando abrir chat            │
  └──────────────────────────────────────────────────────┘
```

### Message Store (Cassandra)

```
  Por que Cassandra?
  - Write-optimized (100B msgs/dia = 1.2M writes/s)
  - Partition by conversation → todas msgs de um chat no mesmo nó
  - Time-ordered dentro da partition
  - Auto-compaction de dados antigos
  - Linear scalability (adicionar nós = mais throughput)
  
  Schema:
  CREATE TABLE messages (
      conversation_id UUID,         -- partition key
      message_id      TIMEUUID,     -- clustering key (time-ordered)
      sender_id       BIGINT,
      content         BLOB,         -- encrypted content (E2E)
      content_type    TEXT,         -- text, image, video, audio
      media_url       TEXT,         -- S3 URL for media
      status          TEXT,         -- sent, delivered, read
      created_at      TIMESTAMP,
      PRIMARY KEY (conversation_id, message_id)
  ) WITH CLUSTERING ORDER BY (message_id DESC);
  
  -- Queries eficientes:
  -- Últimas 50 msgs de um chat:
  SELECT * FROM messages 
  WHERE conversation_id = ? 
  LIMIT 50;
  
  -- Msgs após um timestamp (sync / pagination):
  SELECT * FROM messages 
  WHERE conversation_id = ? 
  AND message_id > ?
  LIMIT 50;
```

---

## End-to-End Encryption (E2E)

```
  WhatsApp usa Signal Protocol:
  
  ┌──────────┐                              ┌──────────┐
  │  User A  │                              │  User B  │
  │          │                              │          │
  │ Private  │                              │ Private  │
  │ Key: Ka  │                              │ Key: Kb  │
  │ Public   │                              │ Public   │
  │ Key: Pa  │                              │ Key: Pb  │
  └────┬─────┘                              └────┬─────┘
       │                                         │
       │  1. Key Exchange (via server):           │
       │  Server stores public keys ONLY          │
       │  A gets Pb, B gets Pa                    │
       │                                         │
       │  2. A encrypts:                          │
       │  ciphertext = encrypt(msg, shared_secret)│
       │  shared_secret = DH(Ka, Pb)              │
       │                                         │
       │  3. Send ciphertext through server       │
       │──────────▶ Server ──────────────────────▶│
       │     (server sees ONLY ciphertext)        │
       │     (server CANNOT read the message)     │
       │                                         │
       │  4. B decrypts:                          │
       │  shared_secret = DH(Kb, Pa)              │
       │  msg = decrypt(ciphertext, shared_secret)│
       │                                         │

  Server NUNCA vê o conteúdo das mensagens!
  Apenas roteia bytes criptografados entre clients.
  
  Para grupos: Sender encrypts uma vez com group key,
  cada member tem a group key (Signal Group Protocol).
```

---

## Offline Message Handling

```
  Quando User B está offline:
  
  1. Message Service detecta: B não tem session ativa
  2. Persiste msg no Cassandra (como sempre)
  3. Envia Push Notification (APNs/FCM):
     { title: "User A", body: "New message", badge: 3 }
  4. Quando B reconecta:
     a. B informa last_received_message_id
     b. Server envia todas msgs após esse ID
     c. Batch delivery: enviar em chunks de 50
  
  Guaranteed delivery:
  - At-least-once: server mantém msg até ACK do client
  - Deduplication no client (message_id é unique)
  - Retry com exponential backoff se ACK não recebido
```

---

## Trade-offs

| Aspecto | WebSocket | Long Polling | Server-Sent Events |
|---------|:---------:|:------------:|:------------------:|
| **Latência** | 🟢 Mais baixa | 🟡 Moderada | 🟡 Moderada |
| **Bidirecional** | 🟢 Sim | 🟡 Simulado | 🔴 Unidirecional |
| **Conexões** | 🔴 Stateful | 🟡 Semi-stateful | 🟡 Semi-stateful |
| **Mobile battery** | 🟡 Heartbeat drena | 🟡 Similar | 🟡 Similar |
| **Escalabilidade** | 🔴 Mais complexa | 🟡 Mais fácil | 🟡 Mais fácil |
| **Ideal para** | Chat, gaming | Fallback | Notifications |

---

## Uso em Big Techs

| Empresa | Stack | Escala | Detalhes |
|---------|-------|--------|----------|
| **WhatsApp** | Erlang/BEAM | 100B msgs/dia, 2B users | 50 engineers (~2014), Mnesia → Cassandra |
| **Messenger** | C++/Hack | 100B msgs/dia | TAO, Iris (msg queue), MQTT para mobile |
| **Telegram** | C++ / Java | 700M MAU | MTProto protocol, cloud-based (não E2E default) |
| **Discord** | Elixir/Rust | 150M MAU | Cassandra → ScyllaDB (2023), real-time voice |
| **Slack** | PHP/Go/Java | Millions of teams | MySQL + Vitess, Kafka, WebSocket |
| **Signal** | Java/Rust | Privacy-focused | Signal Protocol (gold standard E2E) |

### WhatsApp — Escala Impressionante

```
  2014 (aquisição por Meta):
    - 600M users
    - Apenas ~50 engineers!
    - Erlang/BEAM VM: 2M connections per server
    - FreeBSD (não Linux) para tuning de connections
  
  Stack:
    - Erlang para connection handling (massive concurrency)
    - Mnesia (Erlang DB) inicialmente → Cassandra
    - XMPP-based protocol (custom modifications)
    - E2E encryption: Signal Protocol (desde 2016)
    
  O segredo: Erlang/BEAM suporta millions de lightweight processes
  por VM, perfeito para chat (1 process per connection).
```

---

## Perguntas Frequentes em Entrevistas

1. **"Como entregar mensagens em real-time?"**
   - WebSocket para conexão persistente bidirecional
   - Session Service mapeia user → gateway server
   - Message routing: sender gateway → message svc → receiver gateway

2. **"Como garantir entrega de mensagens?"**
   - Persist first, then deliver (write-ahead)
   - Client ACK após receber (sent ✓ → delivered ✓✓)
   - Offline queue + push notification
   - Retry até ACK recebido

3. **"Como escalar para 500M conexões simultâneas?"**
   - Connection gateway cluster: 10K+ servers
   - 50K connections per server (epoll/io_uring)
   - Consistent hashing para distribuir users entre gateways
   - Session Service (Redis) para lookup rápido

4. **"Como funciona E2E encryption?"**
   - Signal Protocol: Double Ratchet + X3DH key agreement
   - Server armazena APENAS ciphertext
   - Keys trocadas via server, mas server não conhece private keys
   - Group: shared group key, rekeyed quando membros mudam

5. **"Cassandra vs MySQL para messages?"**
   - Cassandra: write-optimized (1.2M writes/s), partition by conversation
   - MySQL: melhor para metadata (users, groups), NOT for message volume
   - Hybrid: Cassandra para messages, MySQL/PostgreSQL para metadata

6. **"Como otimizar battery usage em mobile?"**
   - Heartbeat adaptivo (WiFi: 30s, cellular: 60s-5min)
   - Push notifications via OS (APNs/FCM) para background
   - Batch message delivery (não 1 push per msg)
   - Compression (protobuf ao invés de JSON)

---

## Referências

- WhatsApp Engineering — *"1 Million is so 2011"* (Rick Reed, Erlang Factory)
- Meta Engineering — *"Building Mobile-First Infrastructure for Messenger"*
- Signal Protocol Specification — signal.org/docs
- Discord Engineering — *"How Discord Stores Trillions of Messages"*
- Alex Xu — *"System Design Interview"*, Cap. 12: Design a Chat System
- Slack Engineering — *"Scaling Slack"*
- Martin Kleppmann — *"Designing Data-Intensive Applications"*, Cap. 11: Stream Processing
