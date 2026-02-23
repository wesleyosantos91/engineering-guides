# 6. API Gateway

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial — padrão obrigatório em arquiteturas de microservices  
> **Complexidade:** Média-Alta

---

## Definição

**API Gateway** é um ponto de entrada único (single entry point) que recebe todas as chamadas dos clients, aplica cross-cutting concerns (autenticação, rate limiting, logging) e roteia para os microservices adequados.

> É o **"front door"** de toda a arquitetura de microservices.

---

## Por Que é Importante?

### Sem API Gateway

```
┌────────┐     ┌──────────────┐
│ Mobile │────▶│ User Service │
│  App   │────▶│ Order Service│
│        │────▶│ Product Svc  │
│        │────▶│ Payment Svc  │   Client precisa conhecer TODOS os services
└────────┘     └──────────────┘   Auth duplicada em cada service
                                  N endpoints diferentes para o client
```

### Com API Gateway

```
┌────────┐     ┌─────────────┐     ┌──────────────┐
│ Mobile │     │             │     │ User Service │
│  App   │────▶│ API Gateway │────▶│ Order Service│
│        │     │             │     │ Product Svc  │
└────────┘     └─────────────┘     │ Payment Svc  │
                                   └──────────────┘
               Um endpoint          Auth centralizada
               Auth centralizada    Rate limiting global
               Rate limiting        Observability
```

---

## Funcionalidades Principais

```
┌─────────────────────────────────────────────────────────────┐
│                        API Gateway                          │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐ │
│  │ Routing     │  │ Auth/AuthZ   │  │ Rate Limiting     │ │
│  │ /api/users→ │  │ JWT verify   │  │ 100 req/s per     │ │
│  │  user-svc   │  │ OAuth2       │  │ client            │ │
│  └─────────────┘  └──────────────┘  └───────────────────┘ │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐ │
│  │ Protocol    │  │ Request      │  │ Response          │ │
│  │ Translation │  │ Transform    │  │ Aggregation       │ │
│  │ REST→gRPC   │  │ Add headers  │  │ Combine multiple  │ │
│  │ HTTP→WS     │  │ Modify body  │  │ service responses │ │
│  └─────────────┘  └──────────────┘  └───────────────────┘ │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐ │
│  │ Caching     │  │ Load Shed    │  │ Observability     │ │
│  │ Response    │  │ Circuit      │  │ Logging           │ │
│  │ caching     │  │ Breaker      │  │ Metrics           │ │
│  │             │  │ Retry        │  │ Tracing           │ │
│  └─────────────┘  └──────────────┘  └───────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 1. Request Routing

```
/api/v1/users/*        → user-service:8080
/api/v1/orders/*       → order-service:8081
/api/v1/products/*     → product-service:8082
/api/v1/payments/*     → payment-service:8083
/api/v2/users/*        → user-service-v2:8080  (API versioning)
```

### 2. Authentication & Authorization

```
Client ──▶ [Gateway] ──▶ [Auth Middleware] ──▶ Backend

1. Client envia: Authorization: Bearer <JWT>
2. Gateway valida JWT (signature, expiration, issuer)
3. Se válido: extrai claims, adiciona X-User-Id header
4. Se inválido: 401 Unauthorized (backend não é atingido)
```

```
┌────────┐     ┌─────────┐     ┌──────────┐     ┌─────────┐
│ Client │────▶│ Gateway │────▶│Auth Svc  │     │ Backend │
│        │     │         │     │(optional)│     │         │
└────────┘     └────┬────┘     └──────────┘     └─────────┘
                    │                                │
          JWT verify (local)                         │
          ou call Auth Service                       │
                    │                                │
                    │  X-User-Id: 123               │
                    │  X-User-Role: admin            │
                    │───────────────────────────────▶│
```

### 3. Rate Limiting

```
Algoritmos comuns:

Fixed Window:    [100 req/min] │——————│——————│——————│
                               0     60    120   180s

Sliding Window:  [100 req/min] ──────────────────────▶
                               Conta os últimos 60s a qualquer momento

Token Bucket:    [10 tokens/s, burst=50]
                 Tokens acumulam; requests consomem tokens

Leaky Bucket:    [10 req/s max]
                 Requests enfileiram; processadas a taxa constante
```

**Exemplo com Redis:**
```lua
-- Sliding Window Rate Limiter (Redis Lua script)
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- Remove requests fora da window
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- Conta requests na window atual
local count = redis.call('ZCARD', key)

if count < limit then
    redis.call('ZADD', key, now, now .. ':' .. math.random())
    redis.call('EXPIRE', key, window)
    return 1  -- allowed
end

return 0  -- rate limited (429)
```

### 4. Protocol Translation

```
Client (REST/JSON) ──▶ [Gateway] ──▶ Backend (gRPC/Protobuf)

Mobile App (REST)  ──▶ [Gateway] ──▶ user-service (gRPC)
Browser (GraphQL)  ──▶ [Gateway] ──▶ multiple services (REST)
IoT (MQTT)         ──▶ [Gateway] ──▶ telemetry-service (Kafka)
```

### 5. Response Aggregation (API Composition)

```
Mobile App precisa de:
  - User profile     (user-service)
  - Last 5 orders    (order-service)  
  - Recomendações    (recommendation-service)

Sem Gateway: 3 requests paralelas do mobile (lento, bateria)
Com Gateway: 1 request → Gateway agrega → 1 response

GET /api/mobile/home
  └── Gateway faz:
      ├── GET user-service/users/123
      ├── GET order-service/users/123/orders?limit=5
      └── GET recommendation-service/users/123/feed
      ← Combina tudo em uma response JSON
```

### 6. Request/Response Transformation

```
# Request transformation
Client envia: POST /api/users { "fullName": "João Silva" }
Gateway transforma: POST /internal/users { "first_name": "João", "last_name": "Silva" }

# Response transformation
Backend retorna: { "usr_id": 123, "usr_nm": "João" }
Gateway transforma: { "id": 123, "name": "João" }  (API pública limpa)

# Header injection
Gateway adiciona: X-Request-Id, X-Trace-Id, X-User-Id (do JWT)
```

---

## Padrão BFF (Backend for Frontend)

Um API Gateway **dedicado por tipo de client**:

```
┌──────────┐     ┌────────────┐
│  Mobile  │────▶│ Mobile BFF │──┐
│   App    │     │  Gateway   │  │
└──────────┘     └────────────┘  │
                                  │     ┌──────────────┐
┌──────────┐     ┌────────────┐  ├────▶│ User Service │
│  Web     │────▶│  Web BFF   │──┤     │ Order Service│
│  App     │     │  Gateway   │  │     │ Product Svc  │
└──────────┘     └────────────┘  │     └──────────────┘
                                  │
┌──────────┐     ┌────────────┐  │
│  IoT     │────▶│  IoT BFF   │──┘
│ Devices  │     │  Gateway   │
└──────────┘     └────────────┘
```

**Por que BFF?**
- **Mobile:** Payload menor, menos campos, imagens otimizadas
- **Web:** Payload completo, suporta paginação avançada
- **IoT:** Payload mínimo, protocolos diferentes (MQTT)
- Cada BFF adapta a API para as necessidades específicas do client

---

## Tecnologias

### Kong Gateway

```yaml
# Kong declarativo (kong.yml)
_format_version: "3.0"

services:
  - name: user-service
    url: http://user-service:8080
    routes:
      - name: user-route
        paths:
          - /api/v1/users
        strip_path: true
    plugins:
      - name: jwt
        config:
          claims_to_verify:
            - exp
      - name: rate-limiting
        config:
          minute: 100
          policy: redis
          redis_host: redis
      - name: proxy-cache
        config:
          response_code: [200]
          request_method: ["GET"]
          content_type: ["application/json"]
          cache_ttl: 300

  - name: order-service
    url: http://order-service:8081
    routes:
      - name: order-route
        paths:
          - /api/v1/orders
    plugins:
      - name: correlation-id
      - name: prometheus
```

### AWS API Gateway

```yaml
# SAM template
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  ApiGateway:
    Type: AWS::Serverless::HttpApi
    Properties:
      StageName: prod
      CorsConfiguration:
        AllowOrigins: ["https://example.com"]
        AllowMethods: ["GET", "POST", "PUT", "DELETE"]
      Auth:
        DefaultAuthorizer: JwtAuthorizer
        Authorizers:
          JwtAuthorizer:
            AuthorizationScopes:
              - read
            IdentitySource: $request.header.Authorization
            JwtConfiguration:
              issuer: "https://auth.example.com"
              audience:
                - "api.example.com"

  UserFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler
      Runtime: nodejs20.x
      Events:
        GetUsers:
          Type: HttpApi
          Properties:
            ApiId: !Ref ApiGateway
            Path: /api/v1/users
            Method: GET
```

### Spring Cloud Gateway (Java)

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r
                .path("/api/v1/users/**")
                .filters(f -> f
                    .stripPrefix(2)
                    .addRequestHeader("X-Gateway", "spring-cloud")
                    .retry(config -> config
                        .setRetries(3)
                        .setStatuses(HttpStatus.SERVICE_UNAVAILABLE))
                    .circuitBreaker(config -> config
                        .setName("user-cb")
                        .setFallbackUri("forward:/fallback/users")))
                .uri("lb://user-service"))
            
            .route("order-service", r -> r
                .path("/api/v1/orders/**")
                .filters(f -> f
                    .stripPrefix(2)
                    .requestRateLimiter(config -> config
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver())))
                .uri("lb://order-service"))
            .build();
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(100, 150); // 100 req/s, burst 150
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getHeaders()
                .getFirst("X-User-Id"));
    }
}
```

### Comparativo de Tecnologias

| Feature | Kong | AWS API GW | Spring Cloud GW | Envoy | Traefik |
|---------|------|------------|-----------------|-------|---------|
| **Tipo** | Open-source + Enterprise | Managed (serverless) | Framework (Java) | Proxy (C++) | Cloud-native |
| **Deploy** | Self-hosted ou Cloud | AWS only | Embeddable | Sidecar ou standalone | Self-hosted |
| **Plugins** | 100+ (Lua, Go) | Lambda authorizers | Spring filters | C++, Lua, Wasm | Middleware |
| **Performance** | Alta | Depende (cold start) | Alta (JVM) | Muito alta | Boa |
| **gRPC** | Sim | Sim (HTTP API) | Sim | Nativo | Sim |
| **Config** | DB/declarativo | Console/IaC | Java code | xDS/YAML | Auto-discovery |
| **Preço** | Free/Enterprise | Pay per request | Free (infra) | Free | Free/Enterprise |

---

## Padrões de Resiliência no Gateway

### Circuit Breaker

```
          CLOSED ──── falhas > threshold ────▶ OPEN
            ▲                                    │
            │                              timeout expira
            │                                    │
         sucesso                                 ▼
            │                                HALF-OPEN
            └───────── teste ok ─────────────────┘
            
CLOSED:    Requests passam normalmente
OPEN:      Requests bloqueadas (503 imediato)
HALF-OPEN: Permite poucos requests para testar
```

### Retry com Backoff

```
Tentativa 1: falha → espera 100ms
Tentativa 2: falha → espera 200ms
Tentativa 3: falha → espera 400ms
Tentativa 4: falha → espera 800ms + jitter
→ Circuit Breaker abre
```

### Load Shedding

```
Gateway monitoring: CPU > 80% ou latency > 2s
  → Rejeita requests de baixa prioridade (429)
  → Mantém requests críticas (pagamentos, auth)
```

---

## API Versioning via Gateway

| Estratégia | Exemplo | Prós | Contras |
|------------|---------|------|---------|
| **URL path** | `/api/v1/users`, `/api/v2/users` | Simples, explícito | URLs mudam |
| **Header** | `Accept: application/vnd.api.v2+json` | URL limpa | Menos visível |
| **Query param** | `/api/users?version=2` | Fácil de usar | Poluição da URL |
| **Subdomain** | `v2.api.example.com` | Isolamento total | DNS management |

```
# Gateway routing por versão
/api/v1/users  → user-service-v1:8080
/api/v2/users  → user-service-v2:8080
/api/v3/users  → user-service-v3:8080  (canary: 10% do tráfego)
```

---

## Uso em Big Techs

### Netflix — Zuul / Spring Cloud Gateway
- Processa **bilhões** de requests/dia
- Routing, canary deployments, failover
- Zuul 2: async, Netty-based
- Migração para Spring Cloud Gateway em andamento

### Amazon — API Gateway
- Managed service para APIs serverless
- Integração Lambda, DynamoDB, Step Functions
- WebSocket, REST, HTTP APIs
- Throttling automático

### Uber — Edge Gateway
- API Gateway customizado
- gRPC internamente, REST externamente
- Rate limiting por tenant
- Migrou de monolith gateway para mesh-based

### Google — Extensible Service Proxy (ESP)
- Baseado em Envoy
- gRPC + REST gateway
- Integrado com Cloud Endpoints
- Authentication via Firebase Auth / OAuth

### Stripe — API Gateway
- Versioning sofisticado (header-based)
- Idempotency keys
- Rate limiting por API key
- Backward compatibility garantida

---

## Perguntas Comuns em Entrevistas

1. **API Gateway vs Reverse Proxy?** → Gateway é um reverse proxy com funcionalidades de API management (auth, rate limit, transform)
2. **API Gateway vs Service Mesh?** → Gateway é north-south (client → services); Mesh é east-west (service → service)
3. **O que é BFF?** → Backend for Frontend: gateway dedicado por tipo de client
4. **Como lidar com latência do gateway?** → Cache, async processing, connection pooling, minimal processing
5. **Single gateway vs multiple?** → Single para simplicidade; multiple (BFF) para necessidades distintas por client

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Single vs BFF** | Um gateway (simples) | BFF por client (otimizado) |
| **Managed vs Self** | AWS API GW (zero-ops) | Kong/Spring (controle) |
| **Auth no GW vs Service** | Centralized (simples) | Per-service (granular) |
| **Fat vs Thin GW** | Gateway faz tudo | Gateway mínimo + service mesh |
| **Sync vs Async** | Spring Cloud GW (WebFlux) | NGINX/Kong (event-loop) |

---

## Referências

- [Kong Gateway Docs](https://docs.konghq.com/)
- [AWS API Gateway](https://docs.aws.amazon.com/apigateway/)
- [Spring Cloud Gateway](https://spring.io/projects/spring-cloud-gateway)
- [Netflix Zuul](https://github.com/Netflix/zuul/wiki)
- [Microservices.io - API Gateway](https://microservices.io/patterns/apigateway.html)
- Building Microservices — Sam Newman, Chapter 8
