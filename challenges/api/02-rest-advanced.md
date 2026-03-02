# Level 2 — REST API Advanced: Produção & Operação

> **Objetivo:** Elevar a REST API do Level 1 para production-ready com paginação,
> caching, segurança, rate limiting, versionamento, idempotência e webhooks.

**Referência:** [.docs/API/01-rest-best-practices.md](../../../.docs/API/01-rest-best-practices.md) — Seções 7–19

---

## Desafio 2.1 — Paginação: Offset & Cursor

**Contexto:** O catálogo do Marketplace tem milhares de produtos.
Sem paginação, a API trava consumidores e o servidor.

### Requisitos

Implementar **duas** estratégias de paginação:

**Offset-based (padrão para UIs com "página 1, 2, 3..."):**

```
GET /products?page=2&size=20

{
  "data": [...],
  "meta": {
    "total": 350,
    "page": 2,
    "size": 20,
    "totalPages": 18
  },
  "_links": {
    "self":  {"href": "/products?page=2&size=20"},
    "first": {"href": "/products?page=1&size=20"},
    "prev":  {"href": "/products?page=1&size=20"},
    "next":  {"href": "/products?page=3&size=20"},
    "last":  {"href": "/products?page=18&size=20"}
  }
}
```

**Cursor-based (para feeds, timelines, dados que mudam):**

```
GET /products?cursor=eyJpZCI6IjAxSjVBM0suLi4ifQ&size=20

{
  "data": [...],
  "meta": {
    "size": 20,
    "hasMore": true
  },
  "cursors": {
    "before": "eyJpZCI6IjAxSjVBM0ouLi4ifQ",
    "after": "eyJpZCI6IjAxSjVBM0wuLi4ifQ"
  },
  "_links": {
    "self": {"href": "/products?cursor=eyJpZCI6...&size=20"},
    "next": {"href": "/products?cursor=eyJpZCI6IjAxSjVBM0wuLi4ifQ&size=20"}
  }
}
```

**Regras:**
- Cursor = base64 encode de `{"id": "...", "sortField": "..."}` (opaco para o cliente)
- `size` máximo = 100, default = 20
- `page` e `size` inválidos → 400 com detalhes
- Links de navegação condicionais (sem `prev` na primeira página, sem `next` na última)
- Ambas as estratégias disponíveis (client escolhe via query param `pagination=offset|cursor`)

### Critérios de Aceite

- [ ] Offset pagination funciona com `page` + `size`
- [ ] Cursor pagination funciona com `cursor` + `size`
- [ ] `size` respeitado (max 100, default 20)
- [ ] Links de navegação corretos e condicionais
- [ ] Cursor é opaco (base64, não expõe estrutura interna)
- [ ] `page=0` ou `size=-1` retorna 400
- [ ] Performance: cursor não degrada com offset alto
- [ ] Testes para primeira página, última página, página intermediária

---

## Desafio 2.2 — Filtering, Sorting & Search

**Contexto:** Consumidores precisam filtrar produtos por categoria, preço, status
e ordená-los por relevância, preço ou data.

### Requisitos

**Filtering (query params):**

```
GET /products?category=electronics&status=active&minPrice=5000&maxPrice=50000
GET /products?sellerId=01J5A3M...&tags=gaming,rgb
GET /orders?status=pending,confirmed&createdAfter=2025-01-01
```

**Sorting:**

```
GET /products?sort=price:asc
GET /products?sort=createdAt:desc,name:asc    (múltiplos campos)
GET /orders?sort=-createdAt                   (prefixo - = desc)
```

**Search (full-text simples):**

```
GET /products?q=mechanical+keyboard
```

**Regras:**
- Filtros combinam com AND
- Campos de filtro inválidos → 400 (não ignorar silenciosamente)
- Sort fields limitados a campos indexáveis (whitelist)
- Sort padrão: `createdAt:desc`
- Search: busca em `name` + `description` (case-insensitive, contains)
- Paginação funciona junto com filtros e sort

### Java 25

```java
record ProductFilter(
    String category,
    String status,
    Integer minPrice,
    Integer maxPrice,
    String sellerId,
    List<String> tags,
    String q,
    String sort
) {
    static ProductFilter from(Map<String, String> params) {
        return new ProductFilter(
            params.get("category"),
            params.get("status"),
            parseIntOrNull(params.get("minPrice")),
            parseIntOrNull(params.get("maxPrice")),
            params.get("sellerId"),
            parseTags(params.get("tags")),
            params.get("q"),
            params.getOrDefault("sort", "createdAt:desc")
        );
    }

    Predicate<Product> toPredicate() {
        Predicate<Product> predicate = p -> true;
        if (category != null) predicate = predicate.and(p -> p.category().equals(category));
        if (minPrice != null) predicate = predicate.and(p -> p.price() >= minPrice);
        if (maxPrice != null) predicate = predicate.and(p -> p.price() <= maxPrice);
        if (q != null) predicate = predicate.and(p ->
            p.name().toLowerCase().contains(q.toLowerCase()) ||
            p.description().toLowerCase().contains(q.toLowerCase()));
        return predicate;
    }
}
```

### Go 1.26

```go
type ProductFilter struct {
    Category string
    Status   string
    MinPrice *int
    MaxPrice *int
    SellerID string
    Tags     []string
    Query    string
    Sort     string
}

func parseFilter(r *http.Request) (ProductFilter, error) {
    q := r.URL.Query()
    f := ProductFilter{
        Category: q.Get("category"),
        Status:   q.Get("status"),
        SellerID: q.Get("sellerId"),
        Query:    q.Get("q"),
        Sort:     q.Get("sort"),
    }
    if f.Sort == "" {
        f.Sort = "createdAt:desc"
    }
    if v := q.Get("minPrice"); v != "" {
        price, err := strconv.Atoi(v)
        if err != nil { return f, fmt.Errorf("minPrice must be integer") }
        f.MinPrice = &price
    }
    return f, nil
}

func (f ProductFilter) Match(p Product) bool {
    if f.Category != "" && p.Category != f.Category { return false }
    if f.MinPrice != nil && p.Price < *f.MinPrice { return false }
    if f.MaxPrice != nil && p.Price > *f.MaxPrice { return false }
    if f.Query != "" && !strings.Contains(
        strings.ToLower(p.Name), strings.ToLower(f.Query)) { return false }
    return true
}
```

### Critérios de Aceite

- [ ] Filtragem por cada campo (category, status, price range, seller, tags)
- [ ] Filtros combinam com AND
- [ ] Sort por múltiplos campos (ascending e descending)
- [ ] Search full-text em name + description
- [ ] Campo de filtro/sort inválido retorna 400
- [ ] Filtros combinam com paginação
- [ ] Sort padrão aplicado quando nenhum é especificado
- [ ] Testes para combinações de filtros + sort + pagination

---

## Desafio 2.3 — Caching HTTP (ETag & Cache-Control)

**Contexto:** Reduzir carga no servidor e latência para o cliente com caching HTTP nativo.

### Requisitos

**ETag (Conditional Requests):**

```
# Primeira request
GET /products/01J5A3K...
→ 200 OK
→ ETag: "a1b2c3d4"
→ Body: { ... }

# Request subsequente com cache
GET /products/01J5A3K...
If-None-Match: "a1b2c3d4"
→ 304 Not Modified (sem body)

# Após atualização
GET /products/01J5A3K...
If-None-Match: "a1b2c3d4"
→ 200 OK
→ ETag: "e5f6g7h8"  (novo hash)
→ Body: { ... atualizado }
```

**Last-Modified:**

```
GET /products/01J5A3K...
→ Last-Modified: Wed, 15 Jan 2025 10:30:00 GMT

GET /products/01J5A3K...
If-Modified-Since: Wed, 15 Jan 2025 10:30:00 GMT
→ 304 Not Modified
```

**Cache-Control por tipo de recurso:**

| Recurso | Cache-Control | Razão |
|---------|--------------|-------|
| `GET /products` (lista) | `public, max-age=60` | Muda frequentemente |
| `GET /products/{id}` | `public, max-age=300` | Relativamente estável |
| `GET /categories` | `public, max-age=3600` | Raramente muda |
| `GET /orders/{id}` | `private, no-cache` | Dados sensíveis |
| `GET /users/{id}` | `private, no-store` | PII, nunca cachear |
| `POST/PUT/DELETE` | `no-store` | Mutações |

- ETag gerado como hash do conteúdo (MD5/SHA256 do JSON serializado)
- Implementar `If-None-Match` para ETag
- Implementar `If-Modified-Since` para Last-Modified
- `PUT`/`PATCH` com `If-Match` para otimistic locking:

```
PUT /products/01J5A3K...
If-Match: "a1b2c3d4"
→ 200 OK (se ETag match)
→ 412 Precondition Failed (se ETag mudou — outro cliente atualizou)
```

### Critérios de Aceite

- [ ] ETag gerado para todos os GET de recurso individual
- [ ] `If-None-Match` → 304 quando recurso não mudou
- [ ] `If-Modified-Since` → 304 quando data não mudou
- [ ] `If-Match` no PUT/PATCH para optimistic locking → 412 quando conflito
- [ ] `Cache-Control` headers corretos por tipo de recurso
- [ ] Mutações invalidam cache (novo ETag)
- [ ] 304 response não contém body
- [ ] Testes para cache hit, cache miss, cache invalidation

---

## Desafio 2.4 — Segurança: JWT, CORS & Security Headers

**Contexto:** APIs expostas à internet precisam de autenticação, autorização
e headers de segurança.

### Requisitos

**Autenticação JWT:**
- Validar token JWT no header `Authorization: Bearer <token>`
- Extrair claims: `sub` (userId), `roles` (admin/seller/buyer), `exp` (expiration)
- Token expirado → 401 com `WWW-Authenticate: Bearer error="invalid_token"`
- Token ausente → 401 com `WWW-Authenticate: Bearer`
- Rotas públicas (sem auth): `GET /products`, `GET /categories`, `GET /health/*`
- Rotas autenticadas: `POST /products` (seller), `POST /orders` (buyer)
- Rotas admin: `DELETE /users/{id}` (admin only)

**RBAC (Role-Based Access Control):**

| Rota | buyer | seller | admin |
|------|:-----:|:------:|:-----:|
| GET /products | ✅ | ✅ | ✅ |
| POST /products | ❌ | ✅ | ✅ |
| PUT /products/{id} | ❌ | ✅ (own) | ✅ |
| POST /orders | ✅ | ❌ | ✅ |
| GET /users | ❌ | ❌ | ✅ |
| DELETE /users/{id} | ❌ | ❌ | ✅ |

**CORS:**
```
Access-Control-Allow-Origin: https://marketplace.com
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type, X-Request-Id
Access-Control-Max-Age: 3600
Access-Control-Allow-Credentials: true
```

**Security Headers:**
```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'none'
Referrer-Policy: strict-origin-when-cross-origin
```

### Java 25 — JWT Validation (Manual)

```java
// JWT sem library (para entender o protocolo)
record JwtClaims(String sub, List<String> roles, Instant exp) {
    boolean isExpired() { return Instant.now().isAfter(exp); }
    boolean hasRole(String role) { return roles.contains(role); }
}

JwtClaims parseJwt(String token) {
    // JWT = header.payload.signature (Base64URL encoded)
    var parts = token.split("\\.");
    if (parts.length != 3) throw new AuthException("invalid token format");

    var payload = new String(Base64.getUrlDecoder().decode(parts[1]));
    // Parse JSON payload to extract claims
    // Verify signature with HMAC-SHA256 (shared secret for simplicity)
    return new JwtClaims(sub, roles, exp);
}
```

### Critérios de Aceite

- [ ] JWT validado: formato, expiração, signature
- [ ] Claims extraídos e usados para autorização
- [ ] RBAC aplicado: role insuficiente → 403
- [ ] Rotas públicas acessíveis sem token
- [ ] CORS headers em todas as responses
- [ ] `OPTIONS` preflight retorna 204 com CORS headers
- [ ] Security headers presentes em todas as responses
- [ ] Token expirado → 401 com `WWW-Authenticate`
- [ ] Seller só edita seus próprios produtos (resource-level authz)
- [ ] Testes para cada role e cenário de autorização

---

## Desafio 2.5 — Rate Limiting

**Contexto:** Proteger a API contra abuso e garantir fair use entre consumidores.

### Requisitos

Implementar **4 estratégias** de rate limiting (todas em memória):

**1. Fixed Window:**
- Janela fixa de 1 minuto
- 100 requests/minuto por IP
- Reset no início de cada minuto

**2. Sliding Window:**
- Janela deslizante de 1 minuto
- Peso proporcional entre janela anterior e atual
- Mais preciso que fixed window

**3. Token Bucket:**
- Bucket com capacidade de 100 tokens
- Refill de 10 tokens/segundo
- Permite burst de até 100 requests

**4. Leaky Bucket:**
- Fila com capacidade de 100 requests
- Processamento constante de 10 requests/segundo
- Suaviza o tráfego

**Headers de Rate Limit (RFC 6585 + draft-ietf-httpapi-ratelimit-headers):**

```
RateLimit-Limit: 100
RateLimit-Remaining: 42
RateLimit-Reset: 1705312260    (Unix timestamp)
Retry-After: 30                (segundos, apenas quando 429)
```

**Rate limit diferenciado por tier:**

| Tier | Limite | Identificação |
|------|--------|---------------|
| Anonymous | 30 req/min | IP |
| Authenticated | 100 req/min | User ID (do JWT) |
| Premium | 1000 req/min | API Key |

**Quando excede:**
```json
{
  "type": "https://api.marketplace.com/errors/rate-limit-exceeded",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "You have exceeded the rate limit of 100 requests per minute.",
  "retryAfter": 30
}
```

### Critérios de Aceite

- [ ] 4 estratégias implementadas e intercambiáveis
- [ ] Token Bucket permite burst controlado
- [ ] Headers `RateLimit-*` em todas as responses
- [ ] `429 Too Many Requests` com `Retry-After` quando excede
- [ ] Rate limit por IP (anônimo) e por User ID (autenticado)
- [ ] Tiers diferenciados (anonymous < authenticated < premium)
- [ ] Thread-safe (concurrent access ao counter)
- [ ] Testes para cada estratégia nos limites (99, 100, 101 requests)

---

## Desafio 2.6 — Versionamento & Deprecation

**Contexto:** APIs evoluem. Versionar corretamente garante que clientes existentes
não quebrem enquanto novas funcionalidades são adicionadas.

### Requisitos

**Estratégia de versionamento: URI Path (recomendado):**

```
/api/v1/products      → Versão 1 (ativa)
/api/v2/products      → Versão 2 (preview)
```

**Implementar V1 e V2 do endpoint Products:**

- V1: Product com campos `name`, `price`, `category` (string)
- V2: Product com campos `name`, `pricing` (object com `amount`, `currency`, `discount`), `categories` (array)

```json
// V1 Response
{"id": "...", "name": "Keyboard", "price": 15999, "category": "electronics"}

// V2 Response
{
  "id": "...",
  "name": "Keyboard",
  "pricing": {"amount": 15999, "currency": "BRL", "discount": null},
  "categories": [{"id": "...", "name": "Electronics"}]
}
```

**Deprecation Lifecycle:**

```
Preview → Active → Deprecated → Retired

       ┌───────────────────────────────────────┐
       │ Sunset header indica data de retirada  │
       └───────────────────────────────────────┘
```

**Headers de deprecação (RFC 8594):**

```
Deprecation: true
Sunset: Sat, 01 Mar 2026 00:00:00 GMT
Link: </api/v2/products>; rel="successor-version"
```

**API Changelog:**
- Implementar `GET /api/changelog` com histórico de mudanças
- Formato: `[{"version": "v2", "date": "2025-06-01", "changes": [...]}]`

### Critérios de Aceite

- [ ] V1 e V2 coexistem e funcionam simultaneamente
- [ ] V1 retorna formato antigo, V2 retorna formato novo
- [ ] Header `Deprecation: true` em endpoints V1 deprecated
- [ ] Header `Sunset` com data de retirada
- [ ] Header `Link` com successor-version
- [ ] `GET /api/changelog` lista mudanças entre versões
- [ ] Código internamente compartilha lógica (não duplicação total)
- [ ] Testes para ambas as versões lado a lado

---

## Desafio 2.7 — Idempotência (Stripe Pattern)

**Contexto:** Em redes instáveis, clientes podem reenviar requests.
Operações com side-effects (pagamentos, pedidos) devem ser idempotentes.

### Requisitos

Implementar o **Stripe Idempotency Pattern:**

```
POST /orders
Idempotency-Key: "unique-key-from-client-abc123"
Content-Type: application/json

{"items": [...]}
```

**Fluxo:**

```
Cliente envia request com Idempotency-Key
         │
         ▼
┌─ Key já existe no cache? ──┐
│                             │
│ SIM                         │ NÃO
│ ┌──────────────────┐        │ ┌──────────────────┐
│ │ Request em curso?│        │ │ Salvar key + lock │
│ │ → 409 Conflict   │        │ │ Processar request │
│ │                  │        │ │ Salvar resultado  │
│ │ Já completou?    │        │ │ Retornar response │
│ │ → Retorna cached │        │ └──────────────────┘
│ └──────────────────┘        │
└─────────────────────────────┘
```

**Implementação:**

```java
record IdempotencyRecord(
    String key,
    String requestHash,     // Hash do body para detectar requests diferentes com mesma key
    int statusCode,
    String responseBody,
    Instant createdAt,
    Instant expiresAt,      // 24h TTL
    IdempotencyState state  // PENDING, COMPLETED, FAILED
) {}

enum IdempotencyState { PENDING, COMPLETED, FAILED }
```

**Regras:**
- `Idempotency-Key` obrigatório em `POST` (exceto se explicitamente opt-out)
- Key ausente em POST → 400 com mensagem clara
- Mesma key + mesmo body → retorna response cacheada (200/201)
- Mesma key + body diferente → 422 (key reuse com payload diferente)
- Key com request em andamento → 409 Conflict
- TTL de 24 horas para idempotency records
- Implementar para: `POST /orders`, `POST /payments`
- `PUT` e `DELETE` são naturalmente idempotentes (não precisam da key)

### Go 1.26

```go
type IdempotencyStore struct {
    mu      sync.RWMutex
    records map[string]*IdempotencyRecord
}

func (s *IdempotencyStore) Process(key, bodyHash string, handler func() (int, []byte)) (int, []byte) {
    s.mu.Lock()
    if record, exists := s.records[key]; exists {
        s.mu.Unlock()
        if record.State == StatePending {
            return 409, []byte(`{"error": "request in progress"}`)
        }
        if record.RequestHash != bodyHash {
            return 422, []byte(`{"error": "idempotency key reused with different payload"}`)
        }
        return record.StatusCode, record.ResponseBody
    }

    record := &IdempotencyRecord{Key: key, RequestHash: bodyHash, State: StatePending}
    s.records[key] = record
    s.mu.Unlock()

    status, body := handler()

    s.mu.Lock()
    record.StatusCode = status
    record.ResponseBody = body
    record.State = StateCompleted
    s.mu.Unlock()

    return status, body
}
```

### Critérios de Aceite

- [ ] `Idempotency-Key` obrigatório em POST de Orders e Payments
- [ ] Key ausente → 400
- [ ] Retry com mesma key + mesmo body → response cacheada idêntica
- [ ] Mesma key + body diferente → 422
- [ ] Request em andamento com mesma key → 409
- [ ] TTL de 24h para records (expiram automaticamente)
- [ ] Thread-safe (concurrent access)
- [ ] Testes: retry imediato, retry após completar, key expirada

---

## Desafio 2.8 — Webhooks

**Contexto:** O Marketplace precisa notificar sistemas externos sobre eventos
(pedido criado, pagamento confirmado, entrega realizada).

### Requisitos

**Registro de webhooks:**

```
POST /webhooks
{
  "url": "https://partner.com/webhook",
  "events": ["order.created", "order.shipped", "payment.confirmed"],
  "secret": "whsec_..."
}
```

**Evento enviado (payload assinado):**

```json
{
  "id": "evt_01J5A3K...",
  "type": "order.created",
  "timestamp": "2025-01-15T10:30:00Z",
  "data": {
    "orderId": "01J5A3P...",
    "userId": "01J5A3Q...",
    "total": 49497,
    "currency": "BRL"
  }
}
```

**Assinatura HMAC (Stripe-style):**

```
Webhook-Id: evt_01J5A3K...
Webhook-Timestamp: 1705312200
Webhook-Signature: v1=5257a869e7ecebeda32affa62cdca3fa51cad7e77a0e56ff536d0ce8e108d8bd

// Assinatura = HMAC-SHA256(secret, "{webhook-id}.{timestamp}.{body}")
```

**Retry policy com exponential backoff:**

```
Tentativa 1: imediato
Tentativa 2: 30 segundos
Tentativa 3: 5 minutos
Tentativa 4: 30 minutos
Tentativa 5: 2 horas
Tentativa 6: abandona → marca como "failed"
```

**Regras:**
- Webhook endpoint deve responder com 2xx em até 30 segundos (timeout)
- Retry apenas para 5xx e timeouts (não para 4xx)
- Log de delivery attempts com status
- Endpoint para listar delivery attempts: `GET /webhooks/{id}/deliveries`
- Verificação de URL no registro (não aceitar localhost, IPs privados)
- Implementar validação de assinatura no "receptor" (simular um consumer)

### Critérios de Aceite

- [ ] CRUD de webhook subscriptions
- [ ] Eventos disparados quando ações ocorrem (order.created, etc.)
- [ ] Payload assinado com HMAC-SHA256
- [ ] Assinatura verificável pelo receptor
- [ ] Retry com exponential backoff (até 6 tentativas)
- [ ] Retry apenas para 5xx/timeout (não para 4xx)
- [ ] `GET /webhooks/{id}/deliveries` lista tentativas de entrega
- [ ] Testes: entrega bem-sucedida, retry após falha, assinatura inválida

---

## Anti-Patterns a Evitar

| Anti-Pattern | Correto |
|-------------|---------|
| Offset pagination para dados que mudam | Usar cursor-based para feeds/timelines |
| `Cache-Control` em dados sensíveis | `private, no-store` para PII |
| Rate limit global (sem diferenciação) | Tiers por autenticação/plano |
| Versão no header `Accept` | URI path versioning (`/v1/`, `/v2/`) |
| POST sem idempotência | `Idempotency-Key` para operações com side-effects |
| Webhook sem assinatura | HMAC-SHA256 em todo payload |
| Retry em 4xx | Retry apenas para 5xx e timeout |
| `size=1000` sem limite | Max `size=100`, default `size=20` |

---

## Próximo Nível

→ [Level 3 — GraphQL: Schema & Operações](03-graphql.md)
