# 24. WebSockets / Long Polling / SSE

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial para entrevistas de System Design  
> **Complexidade:** Média

---

## Definição

**Comunicação Real-Time** entre client e server é fundamental para aplicações modernas como chat, notificações, dashboards e trading. Existem quatro abordagens principais, cada uma com trade-offs distintos:

| Técnica | Direção | Persistência | Uso Típico |
|---------|---------|-------------|------------|
| **Short Polling** | Client → Server | Não (request-response) | Status checks simples |
| **Long Polling** | Client → Server | Semi (server segura) | Chat (fallback), notifications |
| **SSE** | Server → Client | Sim (stream unidirecional) | Feeds, notificações, dashboards |
| **WebSocket** | Bidirecional | Sim (full-duplex) | Chat, gaming, trading, collab |

---

## Por Que é Importante?

- **Pilar de real-time systems** — chat, notifications, live updates
- **Trade-off latência vs recurso** — cada técnica tem custo diferente
- **Pergunta constante em entrevistas** — "como atualizar o client em tempo real?"
- **Decisão arquitetural** que impacta escalabilidade e custo

---

## Diagrama Comparativo

```
Short Polling:
  Client ──req──▶ Server   (200 OK, empty)
  Client ──req──▶ Server   (200 OK, empty)   ← desperdiça bandwidth
  Client ──req──▶ Server   (200 OK, data!)
  Client ──req──▶ Server   (200 OK, empty)

Long Polling:
  Client ──req──▶ Server .............. (holding)
                  Server ──resp──▶ Client  (data available!)
  Client ──req──▶ Server .............. (new long poll)

SSE (Server-Sent Events):
  Client ──req──▶ Server
  Client ◀─────── stream ──────── Server   (event 1)
  Client ◀─────── stream ──────── Server   (event 2)
  Client ◀─────── stream ──────── Server   (event 3)

WebSocket:
  Client ←─── HTTP Upgrade ───→ Server
  Client ◀═══ full-duplex ═══▶ Server   (msg from server)
  Client ◀═══ full-duplex ═══▶ Server   (msg from client)
  Client ◀═══ full-duplex ═══▶ Server   (msg from server)
```

---

## 1. Short Polling

### Como Funciona

```
Client envia requests periódicos (ex: a cada 5 segundos):

  Client                         Server
    │                              │
    │──── GET /messages ─────────▶│
    │◀──── 200 OK (empty) ────────│  ← nada novo
    │                              │
    │ (espera 5 segundos)          │
    │                              │
    │──── GET /messages ─────────▶│
    │◀──── 200 OK (empty) ────────│  ← nada novo
    │                              │
    │ (espera 5 segundos)          │
    │                              │
    │──── GET /messages ─────────▶│
    │◀──── 200 OK [{msg: "hi"}] ──│  ← dados!
```

### Implementação

```javascript
// Client-side: Short Polling
function pollMessages() {
    setInterval(async () => {
        const response = await fetch('/api/messages?since=' + lastTimestamp);
        const messages = await response.json();
        if (messages.length > 0) {
            renderMessages(messages);
            lastTimestamp = messages[messages.length - 1].timestamp;
        }
    }, 5000); // Poll a cada 5 segundos
}
```

### Trade-offs

| Prós | Contras |
|------|---------|
| Simples de implementar | Desperdiça bandwidth (respostas vazias) |
| Funciona com qualquer infraestrutura | Latência = intervalo de polling |
| Stateless no server | Muitas conexões desnecessárias |
| Firewall/proxy friendly | Não escala bem para muitos clients |

---

## 2. Long Polling

### Como Funciona

```
Client envia request; server SEGURA até ter dados ou timeout:

  Client                         Server
    │                              │
    │──── GET /messages ─────────▶│
    │         (server holds...)    │
    │         (waiting for data...) │
    │         (data arrives!)      │
    │◀──── 200 OK [{msg: "hi"}] ──│
    │                              │
    │──── GET /messages ─────────▶│  ← novo long poll imediato
    │         (server holds...)    │
    │         ...timeout (30s)...  │
    │◀──── 204 No Content ────────│
    │                              │
    │──── GET /messages ─────────▶│  ← reconecta
```

### Implementação

```javascript
// Client-side: Long Polling
async function longPoll() {
    while (true) {
        try {
            const response = await fetch('/api/messages/poll', {
                signal: AbortSignal.timeout(35000) // timeout > server hold
            });
            
            if (response.status === 200) {
                const messages = await response.json();
                renderMessages(messages);
            }
            // Reconecta imediatamente
        } catch (error) {
            // Timeout ou erro — reconecta com backoff
            await sleep(1000);
        }
    }
}
```

```java
// Server-side: Long Polling (Spring Boot)
@GetMapping("/messages/poll")
public DeferredResult<List<Message>> longPoll(
        @RequestParam String userId) {
    
    DeferredResult<List<Message>> result = 
        new DeferredResult<>(30000L); // 30s timeout
    
    // Registra listener
    messageService.addListener(userId, messages -> {
        result.setResult(messages);
    });
    
    result.onTimeout(() -> {
        result.setResult(Collections.emptyList());
    });
    
    return result;
}
```

### Trade-offs

| Prós | Contras |
|------|---------|
| Menor latência que short polling | Server mantém conexões abertas |
| Menos requests desnecessários | Complexidade de timeout management |
| Funciona com HTTP/1.1 standard | Reconexão necessária após cada resposta |
| Bom fallback quando WS indisponível | Limit de conexões simultâneas no server |

---

## 3. Server-Sent Events (SSE)

### Como Funciona

```
Conexão HTTP persistente, unidirecional (server → client):

  Client                         Server
    │                              │
    │──── GET /events ───────────▶│
    │◀──── 200 OK                  │
    │      Content-Type: text/     │
    │      event-stream            │
    │                              │
    │◀──── data: {"msg": "hi"}    │  ← event 1
    │                              │
    │◀──── data: {"msg": "hello"} │  ← event 2
    │                              │
    │◀──── event: notification    │  ← named event
    │      data: {"type": "like"} │
    │                              │
    │    (conexão mantida aberta)  │
```

### Protocolo SSE

```http
GET /events HTTP/1.1
Accept: text/event-stream

HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

: comment (ignored by client)

data: {"message": "Hello"}

data: {"message": "World"}

event: notification
data: {"type": "like", "count": 42}

id: 12345
data: {"message": "with ID"}

retry: 5000
data: {"message": "client should reconnect after 5s on failure"}
```

### Implementação

```javascript
// Client-side: SSE (nativo no browser!)
const eventSource = new EventSource('/api/events?userId=123');

// Eventos genéricos
eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log('Message:', data);
};

// Eventos nomeados
eventSource.addEventListener('notification', (event) => {
    const data = JSON.parse(event.data);
    showNotification(data);
});

// Reconexão automática!
eventSource.onerror = (error) => {
    console.log('SSE error, reconnecting...');
    // Browser reconecta automaticamente com Last-Event-ID
};
```

```java
// Server-side: SSE (Spring Boot)
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<Message>> stream(
        @RequestParam String userId) {
    
    return messageService.getMessageStream(userId)
        .map(msg -> ServerSentEvent.<Message>builder()
            .id(msg.getId())
            .event("message")
            .data(msg)
            .build());
}
```

```go
// Server-side: SSE (Go)
func sseHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    
    flusher, _ := w.(http.Flusher)
    
    for {
        select {
        case msg := <-messageChan:
            fmt.Fprintf(w, "event: message\ndata: %s\n\n", msg)
            flusher.Flush()
        case <-r.Context().Done():
            return
        }
    }
}
```

### Features Nativas do SSE

| Feature | Descrição |
|---------|-----------|
| **Auto-reconnect** | Browser reconecta automaticamente após perda |
| **Last-Event-ID** | Browser envia último ID recebido → server retoma de onde parou |
| **Named events** | Permite filtrar tipos de evento no client |
| **Retry interval** | Server controla delay de reconexão |

### Trade-offs

| Prós | Contras |
|------|---------|
| Simples (HTTP padrão) | Unidirecional (server → client apenas) |
| Auto-reconnect nativo | Limite de 6 conexões/domínio em HTTP/1.1 |
| Suporte nativo no browser (EventSource API) | Não suporta binary data |
| Funciona through proxies/CDNs | Client não envia dados pelo mesmo canal |
| Compressão HTTP nativa | Sem suporte em IE (polyfill necessário) |

---

## 4. WebSocket

### Como Funciona

```
Upgrade de HTTP para protocolo WebSocket (TCP persistente, full-duplex):

  Client                            Server
    │                                 │
    │── GET /chat HTTP/1.1 ─────────▶│
    │   Upgrade: websocket            │
    │   Connection: Upgrade           │
    │   Sec-WebSocket-Key: abc...     │
    │                                 │
    │◀── 101 Switching Protocols ─────│
    │    Upgrade: websocket           │
    │    Sec-WebSocket-Accept: xyz... │
    │                                 │
    │◀══════ Full-Duplex TCP ════════▶│
    │                                 │
    │══▶ {"type":"msg","text":"hi"} ══│  ← client → server
    │◀══ {"type":"msg","text":"hey"} ═│  ← server → client
    │══▶ {"type":"typing"}           ══│  ← client → server
    │◀══ {"type":"presence","online":5}│  ← server → client
```

### WebSocket Handshake

```http
-- Client Request --
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat, superchat

-- Server Response --
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

### Implementação

```javascript
// Client-side: WebSocket
const ws = new WebSocket('wss://server.example.com/chat');

ws.onopen = () => {
    console.log('Connected');
    ws.send(JSON.stringify({ type: 'join', room: 'general' }));
};

ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    switch (data.type) {
        case 'message': renderMessage(data); break;
        case 'typing':  showTypingIndicator(data.user); break;
        case 'presence': updatePresence(data); break;
    }
};

ws.onclose = (event) => {
    console.log(`Disconnected: ${event.code}`);
    // Reconectar com exponential backoff
    setTimeout(() => reconnect(), backoff);
};

ws.onerror = (error) => console.error('WebSocket error:', error);
```

```java
// Server-side: WebSocket (Spring Boot)
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(chatHandler(), "/chat")
                .setAllowedOrigins("*");
    }
}

@Component
public class ChatHandler extends TextWebSocketHandler {
    
    private final Map<String, WebSocketSession> sessions = 
        new ConcurrentHashMap<>();
    
    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        sessions.put(session.getId(), session);
    }
    
    @Override
    protected void handleTextMessage(WebSocketSession session, 
                                      TextMessage message) {
        // Broadcast para todos
        sessions.values().forEach(s -> {
            try { s.sendMessage(message); } 
            catch (IOException e) { /* handle */ }
        });
    }
    
    @Override
    public void afterConnectionClosed(WebSocketSession session, 
                                       CloseStatus status) {
        sessions.remove(session.getId());
    }
}
```

### Escalando WebSockets

```
Problema: WebSocket mantém conexão stateful por client.
  1 server com 64GB RAM → ~1M conexões (64KB/conn)
  10M users online → 10+ servers

Solução: Connection Registry + Pub/Sub

  ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ WS Server│   │ WS Server│   │ WS Server│
  │    1     │   │    2     │   │    3     │
  │ Users:   │   │ Users:   │   │ Users:   │
  │ A, B, C  │   │ D, E, F  │   │ G, H, I  │
  └────┬─────┘   └────┬─────┘   └────┬─────┘
       │              │              │
       └──────────────┼──────────────┘
                      │
              ┌───────▼────────┐
              │  Redis Pub/Sub │  (ou Kafka)
              │  Channel: chat │
              └────────────────┘

  User A (Server 1) envia msg para User F (Server 2):
  1. Server 1 publica msg no Redis channel
  2. Server 2 (subscrito) recebe msg
  3. Server 2 envia para User F via WebSocket
```

```
Sticky Sessions para WebSocket:

  Load Balancer (L7):
  ├── Primeira conexão: hash(user_id) → Server N
  ├── Upgrade request roteado para Server N
  └── Todas as frames subsequentes → Server N

  Alternativa: Connection-aware routing
  ├── Service Registry mapeia user → server
  └── Qualquer server consulta registry para rotear
```

### Trade-offs

| Prós | Contras |
|------|---------|
| Full-duplex (bidirecional) | Mais memória por conexão |
| Latência mínima (conexão persistente) | Sticky sessions ou state management |
| Suporte a binary data | Mais complexo que HTTP |
| Ideal para high-frequency messages | Proxies/firewalls podem bloquear |
| Protocolo maduro e suportado | Reconexão manual (não automática) |

---

## Comparativo Detalhado

| Aspecto | Short Polling | Long Polling | SSE | WebSocket |
|---------|--------------|-------------|-----|-----------|
| **Direção** | Client → Server | Client → Server | Server → Client | Bidirecional |
| **Protocolo** | HTTP | HTTP | HTTP | WS (TCP) |
| **Latência** | Alta (intervalo) | Média | Baixa | Muito baixa |
| **Overhead** | Alto (headers repetidos) | Médio | Baixo | Muito baixo |
| **Conexões** | Muitas curtas | Médias | Uma persistente | Uma persistente |
| **Binary** | Sim | Sim | Não | Sim |
| **Auto-reconnect** | N/A | Manual | Nativo | Manual |
| **Browser** | fetch/XHR | fetch/XHR | EventSource | WebSocket API |
| **HTTP/2 compat** | ✓ | ✓ | ✓✓ (multiplexing) | Independente |
| **Proxy/CDN** | ✓✓ | ✓ | ✓ | ⚠️ (pode bloquear) |
| **Server resources** | Baixo | Médio | Médio | Alto |

---

## Quando Usar Cada Um

```
Decision Tree:

  Precisa de comunicação bidirecional real-time?
  ├── SIM → WebSocket
  │        (chat, gaming, collaborative editing, trading)
  └── NÃO
      │
      Server precisa enviar updates para o client?
      ├── SIM → SSE
      │        (notifications, live feeds, dashboards, stock tickers)
      └── NÃO
          │
          Precisa de updates relativamente rápidos?
          ├── SIM → Long Polling
          │        (fallback quando SSE/WS indisponível)
          └── NÃO → Short Polling
                     (health checks, status polling, simple dashboards)
```

---

## Uso em Big Techs

| Empresa | Sistema | Tecnologia | Detalhes |
|---------|---------|-----------|----------|
| **Slack** | Messaging | WebSocket | Gateway WS para todos os eventos real-time |
| **Discord** | Voice/Chat | WebSocket | Gateway com heartbeat, resume capability |
| **Uber** | Driver tracking | WebSocket | Location updates do motorista em tempo real |
| **Binance** | Trading | WebSocket | Streaming de preços, order book updates |
| **Twitter/X** | Live feed | SSE + WS | Streaming API para tweets em tempo real |
| **GitHub** | Notifications | SSE | Live updates de PRs, issues, actions |
| **Figma** | Collaboration | WebSocket | Cursors e edições simultâneas |
| **Google Docs** | Collaboration | WebSocket | OT (Operational Transform) via WS |
| **Notion** | Collaboration | WebSocket + Long Polling | Fallback para ambientes restritos |

### Discord Gateway — Arquitetura

```
Discord WebSocket Gateway:

  Client ──── wss://gateway.discord.gg/?v=10 ────▶ Gateway
  
  Connection lifecycle:
  1. Connect → receive HELLO (heartbeat_interval)
  2. Send IDENTIFY (token, intents)
  3. Receive READY (user, guilds, session_id)
  4. Heartbeat loop (every heartbeat_interval ms)
  5. Receive DISPATCH events (MESSAGE_CREATE, etc.)
  
  Resume (reconnection):
  1. Connect → receive HELLO
  2. Send RESUME (token, session_id, last_sequence)
  3. Receive missed events → RESUMED
  
  ┌───────────┐        ┌───────────┐
  │  Client   │◀══WS══▶│  Gateway  │──▶ Pub/Sub (Redis)
  │ (Discord  │        │  Server   │       │
  │  App)     │        │           │       ▼
  └───────────┘        └───────────┘    Backend
                                        Services
```

---

## Perguntas Frequentes em Entrevistas

1. **"Como implementar chat real-time?"**
   - WebSocket para mensagens bidirecionais
   - Redis Pub/Sub para cross-server routing
   - Connection registry para mapear user → server
   - Long polling como fallback

2. **"WebSocket vs SSE — quando usar cada um?"**
   - WebSocket: bidirecional (chat, gaming, collab)
   - SSE: unidirecional server→client (notifications, feeds)
   - SSE é mais simples e tem auto-reconnect

3. **"Como escalar WebSockets?"**
   - Sticky sessions ou connection-aware routing
   - Redis/Kafka Pub/Sub para cross-server messaging
   - Connection state em Redis (session, rooms)
   - Horizontal scaling com gateway pattern

4. **"Long Polling vs WebSocket?"**
   - Long Polling: HTTP padrão, funciona em qualquer lugar
   - WebSocket: menor latência, bidirecional, mais eficiente
   - Long Polling como fallback quando WS bloqueado

---

## Referências

- RFC 6455 — *"The WebSocket Protocol"*
- WHATWG — *"Server-Sent Events"* specification
- Fette & Melnikov (2011) — WebSocket RFC
- Discord Developer Docs — *"Gateway"*
- Slack Engineering — *"Scaling WebSockets"*
- Uber Engineering — *"Real-Time Exactly-Once Event Processing"*
