# Level 0 — Fundamentos HTTP & API Design

> **Objetivo:** Construir um servidor HTTP do zero, compreendendo cada camada do protocolo
> antes de adotar qualquer paradigma de API.

**Referência:** [.docs/API/README.md](../../../.docs/API/README.md)

---

## Desafio 0.1 — HTTP Server From Scratch

**Contexto:** Antes de criar APIs, você precisa entender como HTTP funciona na prática.
Implemente um servidor HTTP que responda a requests básicos do Marketplace.

### Requisitos

- Criar um servidor HTTP que escute na porta `8080`
- Implementar routing manual para pelo menos 3 rotas:
  - `GET /health` → retorna `200 OK` com `{"status": "healthy", "timestamp": "..."}`
  - `GET /` → retorna `200 OK` com informações do sistema
  - Qualquer outra rota → retorna `404 Not Found`
- Suportar `Content-Type: application/json` em todas as respostas
- Incluir headers padrão: `Content-Type`, `X-Request-Id` (UUID único por request)
- Logging estruturado de cada request: método, path, status code, duração

### Java 25

```java
import com.sun.net.httpserver.HttpServer;
import com.sun.net.httpserver.HttpExchange;
import java.net.InetSocketAddress;
import java.util.UUID;
import java.time.Instant;

void main() {
    var server = HttpServer.create(new InetSocketAddress(8080), 0);
    server.setExecutor(Thread.ofVirtual().factory()); // Virtual Threads

    server.createContext("/health", exchange -> {
        var requestId = UUID.randomUUID().toString();
        var start = Instant.now();

        var body = """
            {"status":"healthy","timestamp":"%s","requestId":"%s"}
            """.formatted(Instant.now(), requestId);

        exchange.getResponseHeaders().set("Content-Type", "application/json");
        exchange.getResponseHeaders().set("X-Request-Id", requestId);
        exchange.sendResponseHeaders(200, body.getBytes().length);
        exchange.getResponseBody().write(body.getBytes());
        exchange.close();

        System.out.println("[%s] GET /health → 200 (%dms)"
            .formatted(requestId, java.time.Duration.between(start, Instant.now()).toMillis()));
    });

    server.start();
    System.out.println("Server started on :8080");
}
```

### Go 1.26

```go
package main

import (
    "encoding/json"
    "log/slog"
    "net/http"
    "time"

    "github.com/google/uuid"
)

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        requestID := uuid.New().String()

        w.Header().Set("Content-Type", "application/json")
        w.Header().Set("X-Request-Id", requestID)

        json.NewEncoder(w).Encode(map[string]string{
            "status":    "healthy",
            "timestamp": time.Now().Format(time.RFC3339),
            "requestId": requestID,
        })

        slog.Info("request", "method", "GET", "path", "/health",
            "status", 200, "duration_ms", time.Since(start).Milliseconds())
    })

    slog.Info("server started", "port", 8080)
    http.ListenAndServe(":8080", mux)
}
```

### Critérios de Aceite

- [ ] Servidor inicia e responde em < 100ms
- [ ] `GET /health` retorna JSON válido com status 200
- [ ] `GET /unknown` retorna status 404 com body JSON de erro
- [ ] Cada response inclui `X-Request-Id` único
- [ ] Logs estruturados contêm método, path, status, duração
- [ ] Virtual Threads (Java) / goroutines (Go) para concorrência

---

## Desafio 0.2 — HTTP Methods & Semântica

**Contexto:** Cada método HTTP tem semântica específica. Compreender as propriedades
`safe`, `idempotent` e `cacheable` é fundamental para API design correto.

### Requisitos

Implementar handlers para uma coleção `products` respeitando a semântica correta:

| Método | Rota | Safe | Idempotent | Semântica |
|--------|------|:----:|:----------:|-----------|
| `GET` | `/products` | ✅ | ✅ | Listar produtos (sem side-effects) |
| `GET` | `/products/{id}` | ✅ | ✅ | Detalhe de um produto |
| `POST` | `/products` | ❌ | ❌ | Criar novo produto |
| `PUT` | `/products/{id}` | ❌ | ✅ | Substituir produto completamente |
| `PATCH` | `/products/{id}` | ❌ | ❌ | Atualizar parcialmente |
| `DELETE` | `/products/{id}` | ❌ | ✅ | Remover produto |
| `HEAD` | `/products/{id}` | ✅ | ✅ | Metadados sem body |
| `OPTIONS` | `/products` | ✅ | ✅ | Métodos permitidos |

- Usar armazenamento em memória (`Map`/`map`)
- `PUT` deve exigir **todos** os campos (replace completo)
- `PATCH` deve aceitar campos parciais (merge)
- `DELETE` deve ser idempotente (deletar inexistente → 204/404 consistente)
- `HEAD` retorna mesmos headers que `GET`, mas sem body
- `OPTIONS` retorna `Allow: GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS`

### Critérios de Aceite

- [ ] Cada método retorna o status code correto (201 para POST, 204 para DELETE, etc.)
- [ ] GET é safe (não altera estado)
- [ ] PUT é idempotent (múltiplas chamadas = mesmo resultado)
- [ ] DELETE é idempotent
- [ ] HEAD retorna mesmos headers que GET, body vazio
- [ ] OPTIONS retorna `Allow` header correto
- [ ] Testes automatizados validam cada método

---

## Desafio 0.3 — Status Codes Corretos

**Contexto:** Status codes mal utilizados são uma das fontes mais comuns de APIs confusas.
Implemente tratamento completo de status codes.

### Requisitos

Implementar responses corretas para cada família de status codes no contexto do Marketplace:

```
2xx — Sucesso
├── 200 OK              → GET com resultado, PUT/PATCH com body atualizado
├── 201 Created         → POST bem-sucedido (+ Location header)
├── 202 Accepted        → Operação assíncrona aceita
├── 204 No Content      → DELETE bem-sucedido, PUT/PATCH sem body

3xx — Redirecionamento
├── 301 Moved Permanently → Recurso movido (ex: /api/v1 → /api/v2)
├── 304 Not Modified      → Cache válido (com ETag)

4xx — Erro do Cliente
├── 400 Bad Request     → JSON inválido, campo obrigatório ausente
├── 401 Unauthorized    → Sem token de autenticação
├── 403 Forbidden       → Token válido, sem permissão
├── 404 Not Found       → Recurso inexistente
├── 405 Method Not Allowed → Método HTTP não suportado para a rota
├── 409 Conflict        → Duplicate (ex: email já cadastrado)
├── 422 Unprocessable Entity → Validação de negócio falhou
├── 429 Too Many Requests → Rate limit excedido

5xx — Erro do Servidor
├── 500 Internal Server Error → Erro inesperado (nunca expor stacktrace)
├── 502 Bad Gateway     → Dependência upstream falhou
├── 503 Service Unavailable → Servidor em manutenção
```

- Implementar um middleware que captura erros e retorna o status correto
- `POST /products` com body inválido → 400 com detalhes do erro
- `POST /products` com email duplicado → 409 Conflict
- `GET /products/{id}` inexistente → 404
- `DELETE /products/{id}` sem auth → 401
- `201 Created` deve incluir `Location: /products/{id}` header
- Simular `429` com um counter simples (> 10 requests/segundo)

### Critérios de Aceite

- [ ] Cada cenário retorna o status code exato da tabela
- [ ] `201 Created` inclui `Location` header
- [ ] `400` inclui detalhes dos campos inválidos
- [ ] `401` vs `403` diferenciados corretamente
- [ ] `409` para duplicatas com mensagem clara
- [ ] `429` com `Retry-After` header
- [ ] `5xx` nunca expõe stack traces ou detalhes internos
- [ ] Testes para pelo menos 10 status codes diferentes

---

## Desafio 0.4 — Content Negotiation

**Contexto:** APIs robustas suportam múltiplos formatos. Implemente content negotiation
para que o mesmo endpoint sirva JSON e outros formatos.

### Requisitos

- Implementar `Accept` header parsing:
  - `application/json` → resposta em JSON (padrão)
  - `application/xml` → resposta em XML
  - `text/plain` → resposta em texto legível
  - `*/*` → JSON (fallback)
  - Tipo não suportado → `406 Not Acceptable`
- Implementar `Content-Type` validation em requests:
  - `POST`/`PUT`/`PATCH` com `Content-Type` ausente ou incorreto → `415 Unsupported Media Type`
- Implementar `Accept-Encoding` para compressão:
  - `gzip` → resposta comprimida com `Content-Encoding: gzip`
  - `identity` → sem compressão
- Implementar `Accept-Language` para mensagens de erro:
  - `pt-BR` → mensagens em português
  - `en-US` → mensagens em inglês (padrão)

### Critérios de Aceite

- [ ] `Accept: application/json` → resposta JSON
- [ ] `Accept: application/xml` → resposta XML
- [ ] Tipo não suportado → `406 Not Acceptable`
- [ ] `Content-Type` inválido em POST → `415 Unsupported Media Type`
- [ ] `Accept-Encoding: gzip` → resposta comprimida
- [ ] Mensagens de erro localizadas (pelo menos 2 idiomas)
- [ ] Testes para cada combinação de `Accept` e `Content-Type`

---

## Desafio 0.5 — Request/Response Lifecycle & Middleware

**Contexto:** APIs de produção precisam de tratamento transversal: logging, autenticação,
rate limiting. Implemente um pipeline de middleware.

### Requisitos

Criar uma cadeia de middleware (Chain of Responsibility) aplicada a todas as requests:

```
Request → [Logging] → [RequestId] → [CORS] → [Auth] → [RateLimit] → Handler → Response
                                                                          ↓
Response ← [Logging] ← [RequestId] ← [CORS] ← [Auth] ← [RateLimit] ← Handler
```

1. **Logging Middleware** — Registra método, path, status, duração, request-id
2. **RequestId Middleware** — Gera/propaga `X-Request-Id` (aceita de upstream ou gera novo)
3. **CORS Middleware** — Configura `Access-Control-Allow-Origin`, `Allow-Methods`, `Allow-Headers`
4. **Auth Middleware** — Valida header `Authorization: Bearer <token>` (token fixo para teste)
5. **Rate Limit Middleware** — Limita 100 req/min por IP usando sliding window

### Java 25

```java
// Middleware como Function composition
@FunctionalInterface
interface Middleware {
    HttpHandler wrap(HttpHandler next);
}

Middleware logging = next -> exchange -> {
    var start = Instant.now();
    next.handle(exchange);
    var duration = Duration.between(start, Instant.now());
    System.out.printf("[%s] %s %s → %dms%n",
        exchange.getResponseHeaders().getFirst("X-Request-Id"),
        exchange.getRequestMethod(),
        exchange.getRequestURI(),
        duration.toMillis());
};

Middleware requestId = next -> exchange -> {
    var id = Optional.ofNullable(exchange.getRequestHeaders().getFirst("X-Request-Id"))
        .orElse(UUID.randomUUID().toString());
    exchange.getResponseHeaders().set("X-Request-Id", id);
    next.handle(exchange);
};

// Composição
HttpHandler handler = logging.wrap(requestId.wrap(cors.wrap(auth.wrap(rateLimit.wrap(productHandler)))));
```

### Go 1.26

```go
type Middleware func(http.Handler) http.Handler

func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        slog.Info("request",
            "method", r.Method,
            "path", r.URL.Path,
            "duration_ms", time.Since(start).Milliseconds(),
            "request_id", w.Header().Get("X-Request-Id"),
        )
    })
}

// Composição
handler := loggingMiddleware(requestIdMiddleware(corsMiddleware(authMiddleware(rateLimitMiddleware(productHandler)))))
```

### Critérios de Aceite

- [ ] Middleware executam na ordem correta (logging primeiro, handler por último)
- [ ] `X-Request-Id` propagado de upstream ou gerado automaticamente
- [ ] CORS headers presentes em todas as responses
- [ ] `OPTIONS` preflight retorna 204 com CORS headers
- [ ] Requests sem `Authorization` → 401 (exceto rotas públicas)
- [ ] Rate limit retorna 429 com `Retry-After` quando excedido
- [ ] Middleware são composíveis e reutilizáveis (não acoplados)
- [ ] Testes unitários para cada middleware isoladamente

---

## Desafio 0.6 — Error Handling Padronizado (RFC 9457)

**Contexto:** Erros inconsistentes geram confusão nos consumidores da API.
Implemente o padrão **RFC 9457 Problem Details** para todas as respostas de erro.

### Requisitos

Todas as respostas de erro devem seguir RFC 9457 (`application/problem+json`):

```json
{
  "type": "https://api.marketplace.com/errors/product-not-found",
  "title": "Product Not Found",
  "status": 404,
  "detail": "Product with ID 'abc-123' does not exist in the catalog.",
  "instance": "/products/abc-123",
  "timestamp": "2025-01-15T10:30:00Z",
  "traceId": "req-550e8400-e29b"
}
```

- Implementar tipos de erro para cada cenário do Marketplace:
  - `validation-error` → 400 (com array `errors` contendo campo, mensagem, valor rejeitado)
  - `authentication-required` → 401
  - `insufficient-permissions` → 403
  - `resource-not-found` → 404
  - `conflict` → 409
  - `rate-limit-exceeded` → 429 (com `retryAfter` em extensão)
  - `internal-error` → 500 (sem detalhes internos)
- `Content-Type` da response de erro: `application/problem+json`
- Erros de validação devem incluir extensão com array de violações:

```json
{
  "type": "https://api.marketplace.com/errors/validation-error",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Request body contains 2 validation errors.",
  "errors": [
    {"field": "name", "message": "must not be blank", "rejectedValue": ""},
    {"field": "price", "message": "must be greater than 0", "rejectedValue": -10}
  ]
}
```

### Java 25

```java
sealed interface ApiError permits ValidationError, NotFoundError, ConflictError, InternalError {
    int status();
    String type();
    String title();
    String detail();
    String instance();
}

record ValidationError(String detail, String instance, List<FieldViolation> errors)
    implements ApiError {
    public int status() { return 400; }
    public String type() { return "https://api.marketplace.com/errors/validation-error"; }
    public String title() { return "Validation Failed"; }
}

record FieldViolation(String field, String message, Object rejectedValue) {}
```

### Go 1.26

```go
type ProblemDetail struct {
    Type     string `json:"type"`
    Title    string `json:"title"`
    Status   int    `json:"status"`
    Detail   string `json:"detail"`
    Instance string `json:"instance"`
}

type ValidationProblem struct {
    ProblemDetail
    Errors []FieldViolation `json:"errors"`
}

type FieldViolation struct {
    Field         string `json:"field"`
    Message       string `json:"message"`
    RejectedValue any    `json:"rejectedValue"`
}

func writeError(w http.ResponseWriter, problem ProblemDetail) {
    w.Header().Set("Content-Type", "application/problem+json")
    w.WriteHeader(problem.Status)
    json.NewEncoder(w).Encode(problem)
}
```

### Critérios de Aceite

- [ ] Todas as respostas de erro usam `application/problem+json`
- [ ] Formato segue RFC 9457 com `type`, `title`, `status`, `detail`, `instance`
- [ ] Erros de validação incluem array `errors` com campo, mensagem, valor rejeitado
- [ ] `500 Internal Server Error` nunca expõe detalhes internos (stack trace, SQL, etc.)
- [ ] `traceId` presente para correlação de logs
- [ ] Testes validam formato de erro para cada cenário
- [ ] Erros são modelados como sealed interface (Java) / tipos Go com método `Error()`

---

## Anti-Patterns a Evitar

| Anti-Pattern | Correto |
|-------------|---------|
| `200 OK` para erros (`{"success": false}`) | Usar o status code HTTP correto |
| Stack trace na resposta de erro | Mensagens user-friendly, logs internos |
| `GET` com side-effects | GET é safe; usar POST/PUT/DELETE para mutações |
| `POST` para tudo | Cada método HTTP tem semântica definida |
| Ignorar `Content-Type` | Validar e negociar formato |
| Logs não estruturados (`println`) | Structured logging com campos semânticos |
| Middleware monolítico | Composição de middleware independentes |

---

## Próximo Nível

→ [Level 1 — REST API Design & Implementação](01-rest-api.md)
