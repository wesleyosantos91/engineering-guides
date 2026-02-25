```markdown
# API Gateway / BFF (Backend for Frontend)

> **Categoria:** Infraestrutura e Deploy
> **Complementa:** Service Discovery, Sidecar, Rate Limiter, Circuit Breaker
> **Keywords:** API Gateway, BFF, Backend for Frontend, edge service, cross-cutting concerns, roteamento, agregação, autenticação, throttling

---

## Problema

Em uma arquitetura de microsserviços, cada serviço expõe sua própria API. Sem um ponto de entrada centralizado:

- **Clientes precisam conhecer** N serviços com N endpoints diferentes
- **Cross-cutting concerns** (autenticação, rate limiting, logging) são duplicados em cada serviço
- **Frontends diferentes** (web, mobile, TV) precisam de dados agregados de formas diferentes
- **Protocolo interno** (gRPC, mensageria) precisa ser traduzido para HTTP/REST externo
- **Mudanças internas** (split de serviço, renomeação) impactam todos os clientes

---

## Solução

Um **API Gateway** atua como ponto de entrada único (front door) para todos os clientes. Ele roteia requests para os serviços internos, aplica cross-cutting concerns e pode agregar respostas.

```
                        ┌──────────────────────────────┐
                        │        API Gateway           │
                        │                              │
 Web App ─────────────▶ │  ┌─ Auth ─────────────────┐  │ ─────▶ order-service
 Mobile App ──────────▶ │  │ Rate Limiting           │  │ ─────▶ payment-service
 Partner API ─────────▶ │  │ Logging / Tracing       │  │ ─────▶ user-service
                        │  │ Protocol Translation    │  │ ─────▶ notification-service
                        │  │ Request Routing         │  │
                        │  └─────────────────────────┘  │
                        └──────────────────────────────┘
```

---

## Responsabilidades do API Gateway

| Responsabilidade | Descrição |
|-----------------|-----------|
| **Roteamento** | Direciona requests para o serviço correto baseado no path, header ou método |
| **Autenticação / Autorização** | Valida tokens (JWT, OAuth2) antes de encaminhar ao serviço |
| **Rate Limiting** | Limita requisições por client/API key/tenant |
| **Load Balancing** | Distribui carga entre instâncias do serviço destino |
| **Circuit Breaker** | Protege serviços internos contra sobrecarga |
| **Protocol Translation** | Traduz HTTP ↔ gRPC, REST ↔ GraphQL, WebSocket ↔ HTTP |
| **Request/Response Transformation** | Modifica headers, body, query params |
| **Caching** | Cache de respostas para reduzir carga nos serviços |
| **Logging / Tracing** | Registra access logs, propaga traceId/correlationId |
| **TLS Termination** | Termina HTTPS no gateway, encaminha HTTP internamente |
| **CORS** | Gerencia Cross-Origin Resource Sharing centralmente |
| **API Versioning** | Roteia por versão (/v1/..., /v2/...) |

---

## API Gateway vs. BFF

### API Gateway Genérico

Um único gateway para todos os clientes:

```
Web App ──────┐
Mobile App ───┤──▶ API Gateway ──▶ Serviços
Partner API ──┘
```

**Problema:** Endpoints genéricos que tentam servir todos os clientes. Mobile precisa de menos dados que web; partner precisa de formato diferente. O gateway fica complexo e acoplado a todos os frontends.

### BFF (Backend for Frontend)

Um gateway **dedicado por tipo de cliente**:

```
Web App ──────────▶ BFF Web ─────────▶ Serviços
Mobile App ────────▶ BFF Mobile ──────▶ Serviços
Partner API ───────▶ BFF Partner ─────▶ Serviços
```

Cada BFF:
- Agrega dados na forma que **seu** frontend precisa
- Retorna apenas os campos necessários para aquele client
- É mantido pelo **time do frontend** correspondente

### Comparação

| Aspecto | API Gateway Genérico | BFF |
|---------|---------------------|-----|
| **Foco** | Cross-cutting concerns | Agregação + adaptação por frontend |
| **Quantidade** | 1 (único ponto de entrada) | 1 por tipo de frontend |
| **Quem mantém** | Time de plataforma/infra | Time do frontend |
| **Complexidade** | Menor (1 componente) | Maior (N componentes) |
| **Flexibilidade** | Menor (serve todos igual) | Maior (cada BFF é customizado) |
| **Quando usar** | APIs padronizadas, poucos tipos de client | Frontends com necessidades muito diferentes |

### Combinação: Gateway + BFF

Na prática, muitas equipes combinam ambos:

```
                   ┌─────────────┐
Web App ──────────▶│             │──▶ BFF Web ──────────▶ Serviços
Mobile App ───────▶│ API Gateway │──▶ BFF Mobile ───────▶ Serviços
Partner API ──────▶│ (auth, rate │──▶ BFF Partner ──────▶ Serviços
                   │  limit, TLS)│
                   └─────────────┘

Gateway: cross-cutting concerns (auth, rate limit, TLS, logging)
BFF:     agregação e adaptação específica p/ cada frontend
```

---

## Padrões de Roteamento

### Path-Based Routing

```
/api/orders/**     → order-service
/api/payments/**   → payment-service
/api/users/**      → user-service
```

### Header-Based Routing

```
X-API-Version: v1  → order-service-v1
X-API-Version: v2  → order-service-v2

X-Client-Type: mobile → bff-mobile
X-Client-Type: web    → bff-web
```

### Weight-Based Routing (Canary)

```
/api/orders/** → 95% order-service-stable
               → 5%  order-service-canary
```

---

## Agregação no Gateway / BFF

O gateway ou BFF pode agregar dados de múltiplos serviços em uma única resposta:

```
Client:
  GET /api/dashboard/user-123

BFF:
  ┌── GET /users/123          → user-service    (nome, email)
  ├── GET /orders?user=123    → order-service   (últimos pedidos)
  └── GET /wallet/123         → wallet-service  (saldo)
  
  Resposta agregada:
  {
    "user": { "name": "João", "email": "..." },
    "lastOrders": [...],
    "walletBalance": 1500.00
  }
```

**Atenção:** Agregação complexa deve estar no BFF ou em um serviço compositor dedicado — não no gateway genérico. O gateway deve ser leve e focar em cross-cutting concerns.

---

## Diagrama de Decisão

```
Precisa de ponto de entrada único com auth/rate limit?
     │
    SIM → API Gateway
     │
     ▼
Frontends têm necessidades de dados muito diferentes?
     │
    SIM → BFF por frontend (+ Gateway para cross-cutting)
     │
    NÃO → API Gateway genérico é suficiente
```

---

## Exemplo Conceitual (Pseudocódigo)

```
// API Gateway — roteamento + cross-cutting
class ApiGateway:
    authService
    rateLimiter
    serviceRegistry
    
    function handle(request):
        // 1. Autenticação
        if not authService.validate(request.headers.authorization):
            return Response(401, "Unauthorized")
        
        // 2. Rate Limiting
        if not rateLimiter.allow(request.apiKey):
            return Response(429, "Too Many Requests",
                headers: { "Retry-After": "30" })
        
        // 3. Logging / Tracing
        correlationId = request.header("X-Correlation-Id") OR generateUUID()
        request.addHeader("X-Correlation-Id", correlationId)
        
        // 4. Roteamento
        targetService = resolveRoute(request.path)
        
        // 5. Forward
        response = forward(request, targetService)
        
        // 6. Logging
        log.info("Request processed", 
            correlationId: correlationId,
            path: request.path,
            status: response.status,
            duration: timer.elapsed())
        
        return response

// BFF — adaptação para mobile
class MobileBFF:
    userClient
    orderClient
    walletClient
    
    function getDashboard(userId):
        // Paralelo — mobile precisa de poucos dados
        user = userClient.getBasicInfo(userId)    // só nome e avatar
        orders = orderClient.getRecent(userId, limit: 3) // só últimos 3
        balance = walletClient.getBalance(userId) // só saldo
        
        return MobileDashboard(
            userName: user.name,
            avatar: user.avatarUrl,
            recentOrdersCount: orders.size,
            recentOrders: orders.map(o -> o.summary),  // resumo, sem detalhes
            balance: balance.amount
        )
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Gateway com lógica de negócio | Gateway vira monolito; acoplamento | Gateway: só cross-cutting. Lógica: nos serviços. |
| Gateway único para tudo | Um componente faz roteamento, agregação, auth, transformação | Separe Gateway (infra) de BFF (agregação) |
| BFF que chama BFF | Cadeia de BFFs = latência e complexidade | BFF chama serviços diretamente |
| Gateway sem Circuit Breaker | Serviço interno indisponível derruba o gateway | CB por rota/serviço no gateway |
| Gateway como single point of failure | Gateway cai = tudo cai | Deploy em cluster com múltiplas instâncias |
| Sem rate limiting no gateway | Abuso ou bug de client sobrecarrega serviços | Rate limit por API key/tenant no gateway |
| Transformação excessiva no gateway | Performance degradada, lógica difícil de manter | Transformações simples no gateway; complexas no BFF |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Service Discovery** | Gateway usa discovery para encontrar instâncias dos serviços internos |
| **Rate Limiter** | Rate limiting centralizado no gateway |
| **Circuit Breaker** | CB por rota/serviço protege contra falhas em cascata |
| **API Composition** | BFF é um tipo de compositor adaptado por frontend |
| **Sidecar / Service Mesh** | Gateway: tráfego North-South (externo→interno). Mesh: tráfego East-West (interno). |
| **Canary Release** | Gateway implementa roteamento por peso para canary |
| **Strangler Fig** | Gateway é o proxy natural para roteamento legado ↔ novo |
| **Health Checks** | Gateway roteia apenas para backends saudáveis |
| **Observabilidade** | Gateway é o ponto ideal para logging e tracing de entrada |

---

## Boas Práticas

1. Gateway deve ser **leve** — foco em cross-cutting concerns (auth, rate limit, tracing). **Sem** lógica de negócio.
2. Use **BFF** quando frontends diferentes precisam de dados/formatos distintos.
3. Aplique **Circuit Breaker** por rota/serviço no gateway.
4. **Rate limit** por API key, tenant ou IP no gateway.
5. Sempre propague **Correlation ID / Trace ID** pelo gateway.
6. Deploy do gateway em **múltiplas instâncias** — ele é o single point of entry.
7. Monitore **latência do gateway** — é o primeiro indicador de problemas.
8. Use **path-based routing** como padrão; header-based para cenários avançados (canary, versioning).

---

## Referências

- Chris Richardson — *Microservices Patterns* (Manning) — API Gateway Pattern
- Sam Newman — *Building Microservices* (O'Reilly) — API Gateway e BFF
- Microsoft — [Gateway Routing Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/gateway-routing)
- Microsoft — [Backends for Frontends Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/backends-for-frontends)
- Netflix — [Zuul Gateway](https://github.com/Netflix/zuul)

```
