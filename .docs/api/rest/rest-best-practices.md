# REST API — Best Practices

> Guia completo de boas práticas para design, implementação e manutenção de APIs RESTful, alinhado com padrões de mercado adotados por empresas como Google, Microsoft, Stripe e Twilio.

---

## Sumário

1. [Design de Recursos e URIs](#1-design-de-recursos-e-uris)
2. [Métodos HTTP](#2-métodos-http)
3. [Status Codes](#3-status-codes)
4. [Versionamento](#4-versionamento)
5. [Paginação, Filtragem e Ordenação](#5-paginação-filtragem-e-ordenação)
6. [Tratamento de Erros](#6-tratamento-de-erros)
7. [Segurança](#7-segurança)
8. [Rate Limiting e Throttling](#8-rate-limiting-e-throttling)
9. [Caching](#9-caching)
10. [HATEOAS e Hypermedia](#10-hateoas-e-hypermedia)
11. [Documentação (OpenAPI/Swagger)](#11-documentação-openapiswagger)
12. [Idempotência](#12-idempotência)
13. [Compressão e Performance](#13-compressão-e-performance)
14. [Health Checks e Observabilidade](#14-health-checks-e-observabilidade)
15. [Boas Práticas de Payload](#15-boas-práticas-de-payload)
16. [Deprecation Strategy](#16-deprecation-strategy)
17. [Content Negotiation](#17-content-negotiation)
18. [Webhooks e Eventos](#18-webhooks-e-eventos)
19. [Anti-patterns Comuns](#19-anti-patterns-comuns)

---

## 1. Design de Recursos e URIs

### Princípios

- **Use substantivos no plural** para representar coleções de recursos.
- **Evite verbos** na URI — o método HTTP já expressa a ação.
- **Hierarquia lógica** deve refletir o relacionamento entre recursos.
- **Use kebab-case** para URIs com múltiplas palavras.

### Exemplos

```
✅ Correto
GET    /api/v1/users
GET    /api/v1/users/{userId}
GET    /api/v1/users/{userId}/orders
GET    /api/v1/users/{userId}/orders/{orderId}
POST   /api/v1/users

❌ Incorreto
GET    /api/v1/getUsers
GET    /api/v1/user/get/123
POST   /api/v1/createUser
GET    /api/v1/users/123/getOrders
```

### Regras de Nomeação

| Regra | Exemplo |
|-------|---------|
| Plural para coleções | `/users`, `/products` |
| Singular para singletons | `/users/{id}/profile` |
| Kebab-case para compostos | `/order-items`, `/payment-methods` |
| Sem trailing slash | `/users` (não `/users/`) |
| Lowercase sempre | `/users` (não `/Users`) |
| Máximo 3 níveis de profundidade | `/users/{id}/orders/{id}` |

---

## 2. Métodos HTTP

| Método | Uso | Idempotente | Safe |
|--------|-----|:-----------:|:----:|
| `GET` | Leitura de recurso(s) | ✅ | ✅ |
| `POST` | Criação de recurso | ❌ | ❌ |
| `PUT` | Substituição completa do recurso | ✅ | ❌ |
| `PATCH` | Atualização parcial do recurso | ❌* | ❌ |
| `DELETE` | Remoção de recurso | ✅ | ❌ |
| `HEAD` | Metadados sem body | ✅ | ✅ |
| `OPTIONS` | Métodos suportados (CORS) | ✅ | ✅ |

> *`PATCH` pode ser implementado como idempotente usando JSON Merge Patch (RFC 7396).

### Semântica correta

```http
# Criar recurso
POST /api/v1/users
Content-Type: application/json

{
  "name": "João Silva",
  "email": "joao@example.com"
}

# Atualizar recurso completo
PUT /api/v1/users/123
Content-Type: application/json

{
  "name": "João Silva",
  "email": "joao.silva@example.com",
  "phone": "+5511999999999"
}

# Atualizar parcialmente
PATCH /api/v1/users/123
Content-Type: application/merge-patch+json

{
  "email": "novo@example.com"
}
```

---

## 3. Status Codes

### Categorias Principais

| Faixa | Significado |
|-------|-------------|
| `2xx` | Sucesso |
| `3xx` | Redirecionamento |
| `4xx` | Erro do cliente |
| `5xx` | Erro do servidor |

### Códigos Recomendados por Operação

| Operação | Sucesso | Erros Comuns |
|----------|---------|--------------|
| `GET` | `200 OK` | `404 Not Found` |
| `POST` (criação) | `201 Created` | `400 Bad Request`, `409 Conflict` |
| `PUT` | `200 OK` ou `204 No Content` | `400`, `404`, `409` |
| `PATCH` | `200 OK` ou `204 No Content` | `400`, `404`, `422` |
| `DELETE` | `204 No Content` | `404 Not Found` |
| Lista vazia | `200 OK` com `[]` | — |
| Assíncrono | `202 Accepted` | — |

### Códigos que você deve conhecer

```
200 OK                    — Requisição bem-sucedida
201 Created               — Recurso criado com sucesso
202 Accepted              — Processamento assíncrono aceito
204 No Content            — Sucesso sem body de resposta
301 Moved Permanently     — Recurso movido permanentemente
304 Not Modified          — Cache válido (ETag/Last-Modified)
400 Bad Request           — Payload inválido / validação falhou
401 Unauthorized          — Autenticação necessária
403 Forbidden             — Sem permissão para o recurso
404 Not Found             — Recurso não encontrado
405 Method Not Allowed    — Método HTTP não suportado
409 Conflict              — Conflito de estado do recurso
415 Unsupported Media Type— Content-Type não suportado
422 Unprocessable Entity  — Entidade semanticamente inválida
429 Too Many Requests     — Rate limit excedido
500 Internal Server Error — Erro inesperado do servidor
502 Bad Gateway           — Erro no upstream
503 Service Unavailable   — Serviço temporariamente indisponível
504 Gateway Timeout       — Timeout no upstream
```

---

## 4. Versionamento

### Estratégias

| Estratégia | Exemplo | Prós | Contras |
|-----------|---------|------|---------|
| **URI Path** (recomendado) | `/api/v1/users` | Simples, visível, cacheável | Pode duplicar rotas |
| Header customizado | `Api-Version: 1` | URI limpa | Menos visível |
| Accept header | `Accept: application/vnd.api+json;version=1` | Segue content negotiation | Complexo |
| Query parameter | `/api/users?version=1` | Fácil de testar | Poluição de cache |

### Recomendação de Mercado

```
✅ URI Path é o padrão mais adotado (Google, Stripe, Twilio):
   /api/v1/resources
   /api/v2/resources

✅ Mantenha no máximo 2 versões ativas simultaneamente.
✅ Defina política de deprecation com prazos claros.
✅ Use headers de resposta para comunicar deprecation:

HTTP/1.1 200 OK
Deprecation: Mon, 01 Jun 2026 00:00:00 GMT
Sunset: Tue, 01 Dec 2026 00:00:00 GMT
Link: </api/v2/users>; rel="successor-version"
```

---

## 5. Paginação, Filtragem e Ordenação

### Paginação

#### Offset-based (mais comum)

```http
GET /api/v1/users?page=2&size=20

Response:
{
  "data": [...],
  "pagination": {
    "page": 2,
    "size": 20,
    "totalElements": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrevious": true
  }
}
```

#### Cursor-based (recomendado para grandes volumes)

```http
GET /api/v1/users?cursor=eyJpZCI6MTAwfQ&limit=20

Response:
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTIwfQ",
    "previousCursor": "eyJpZCI6ODB9",
    "hasMore": true
  }
}
```

### Filtragem

```http
# Filtros simples
GET /api/v1/products?category=electronics&status=active

# Filtros com operadores
GET /api/v1/products?price[gte]=100&price[lte]=500

# Busca textual
GET /api/v1/products?q=smartphone

# Seleção de campos (Sparse Fieldsets)
GET /api/v1/users?fields=id,name,email
```

### Ordenação

```http
# Ascendente
GET /api/v1/products?sort=price

# Descendente (prefixo -)
GET /api/v1/products?sort=-created_at

# Múltiplos campos
GET /api/v1/products?sort=-created_at,name
```

---

## 6. Tratamento de Erros

### Formato Padrão (RFC 9457 — Problem Details for HTTP APIs)

> **Nota**: RFC 9457 substituiu a RFC 7807 em 2023 como padrão para Problem Details. A estrutura é compatível, com melhorias na clareza das definições.

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "One or more fields failed validation.",
  "instance": "/api/v1/users",
  "timestamp": "2026-02-23T10:30:00Z",
  "traceId": "abc123-def456-ghi789",
  "errors": [
    {
      "field": "email",
      "message": "Must be a valid email address",
      "code": "INVALID_EMAIL"
    },
    {
      "field": "name",
      "message": "Must not be blank",
      "code": "REQUIRED_FIELD"
    }
  ]
}
```

### Princípios

- **Consistência**: sempre o mesmo formato de erro em toda a API.
- **Sem stack traces** em produção — use `traceId` para correlação.
- **Mensagens úteis** para o consumidor, não genéricas.
- **Códigos de erro internos** (`code`) para automação no client.
- **Localize** mensagens quando suportar múltiplos idiomas.

### Exemplo de Erro 401

```json
{
  "type": "https://api.example.com/errors/unauthorized",
  "title": "Unauthorized",
  "status": 401,
  "detail": "The access token is expired or invalid.",
  "instance": "/api/v1/users/me",
  "timestamp": "2026-02-23T10:30:00Z"
}
```

---

## 7. Segurança

### Autenticação e Autorização

| Método | Uso Recomendado |
|--------|----------------|
| **OAuth 2.0 + OIDC** | APIs públicas e internas com SSO |
| **JWT (Bearer Token)** | APIs stateless |
| **API Keys** | Integrações server-to-server simples |
| **mTLS** | Comunicação entre serviços internos |

### Headers de Segurança Obrigatórios

```http
# Requisição
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
Content-Type: application/json

# Resposta
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'none'
Cache-Control: no-store
X-Request-Id: uuid-trace-id
```

### Checklist de Segurança

- [ ] **HTTPS obrigatório** — nunca exponha APIs via HTTP.
- [ ] **Validação de input** — sanitize e valide todos os campos.
- [ ] **Rate limiting** — proteja contra abuso.
- [ ] **CORS configurado** — origens permitidas explicitamente.
- [ ] **Tokens com expiração curta** — access tokens de 15-30 min.
- [ ] **Refresh token rotation** — invalidar tokens usados.
- [ ] **Não exponha dados sensíveis** em URLs (use body ou headers).
- [ ] **Log de auditoria** — registre acessos e operações críticas.
- [ ] **Princípio do menor privilégio** — scopes granulares.
- [ ] **Proteção contra injection** — SQL, NoSQL, LDAP, OS commands.

### CORS

```http
# Preflight
OPTIONS /api/v1/users HTTP/1.1
Origin: https://app.example.com

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, PATCH
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

---

## 8. Rate Limiting e Throttling

### Headers Padrão

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1708700000
Retry-After: 60
```

### Estratégias

| Estratégia | Descrição |
|-----------|-----------|
| **Fixed Window** | Limite por janela de tempo fixa (ex: 1000/hora) |
| **Sliding Window** | Janela deslizante para suavizar picos |
| **Token Bucket** | Permite bursts controlados |
| **Leaky Bucket** | Taxa constante de processamento |

### Resposta 429

```json
{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Too Many Requests",
  "status": 429,
  "detail": "Rate limit of 1000 requests per hour exceeded. Try again in 60 seconds.",
  "retryAfter": 60
}
```

### Tiers de Rate Limiting (padrão de mercado)

| Tier | Limite | Uso |
|------|--------|-----|
| Free | 100 req/min | Desenvolvimento/testes |
| Standard | 1.000 req/min | Produção básica |
| Premium | 10.000 req/min | Alto volume |
| Enterprise | Customizado | Negociado |

---

## 9. Caching

### Headers de Cache

```http
# Response cacheável
HTTP/1.1 200 OK
Cache-Control: public, max-age=3600
ETag: "33a64df5"
Last-Modified: Sun, 23 Feb 2026 10:00:00 GMT
Vary: Accept, Authorization

# Request condicional
GET /api/v1/products/123 HTTP/1.1
If-None-Match: "33a64df5"
If-Modified-Since: Sun, 23 Feb 2026 10:00:00 GMT

# Response 304 (não modificado)
HTTP/1.1 304 Not Modified
ETag: "33a64df5"
```

### Estratégias por Tipo de Recurso

| Tipo | Cache-Control | TTL |
|------|---------------|-----|
| Dados estáticos (config) | `public, max-age=86400` | 24h |
| Listagens | `public, max-age=60` | 1 min |
| Recurso individual | `private, max-age=300` | 5 min |
| Dados pessoais | `private, no-cache` | Validação a cada request |
| Dados sensíveis | `no-store` | Sem cache |

---

## 10. HATEOAS e Hypermedia

### Exemplo com HAL (Hypertext Application Language)

```json
{
  "id": 123,
  "name": "João Silva",
  "email": "joao@example.com",
  "status": "ACTIVE",
  "_links": {
    "self": {
      "href": "/api/v1/users/123"
    },
    "orders": {
      "href": "/api/v1/users/123/orders"
    },
    "deactivate": {
      "href": "/api/v1/users/123/deactivate",
      "method": "POST"
    }
  }
}
```

### Quando Usar

- APIs públicas com múltiplos consumidores.
- Quando o client precisa navegar dinamicamente.
- Para reduzir acoplamento entre client e server.

> **Nota**: HATEOAS adiciona complexidade — avalie se os consumidores realmente se beneficiam dele. Muitas APIs de mercado (Stripe, Twilio) optam por documentação clara em vez de HATEOAS completo.

---

## 11. Documentação (OpenAPI/Swagger)

### Estrutura Mínima OpenAPI 3.1

```yaml
openapi: "3.1.0"
info:
  title: Users API
  version: "1.0.0"
  description: API para gerenciamento de usuários.
  contact:
    name: Engineering Team
    email: engineering@example.com

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://staging-api.example.com/v1
    description: Staging

paths:
  /users:
    get:
      summary: List users
      operationId: listUsers
      tags: [Users]
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: size
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        "200":
          description: Successful response
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/UserListResponse"
```

### Checklist de Documentação

- [ ] Todos os endpoints documentados com exemplos.
- [ ] Schemas de request e response definidos.
- [ ] Códigos de erro documentados com exemplos.
- [ ] Autenticação descrita com exemplos.
- [ ] Rate limits documentados.
- [ ] Changelog mantido e acessível.
- [ ] Ambientes (sandbox, staging, prod) listados.
- [ ] SDKs/code samples em múltiplas linguagens.

---

## 12. Idempotência

### Idempotency Key (padrão Stripe)

```http
POST /api/v1/payments HTTP/1.1
Idempotency-Key: unique-request-id-abc-123
Content-Type: application/json

{
  "amount": 5000,
  "currency": "BRL",
  "method": "credit_card"
}
```

### Regras

- O servidor deve armazenar e validar Idempotency Keys.
- Retornar a mesma resposta para requests repetidos com a mesma key.
- Keys devem expirar após 24-48 horas.
- Use UUIDs v4 para gerar keys no client.
- Obrigatório em operações financeiras e side-effects críticos.

### Implementação Sugerida

```
1. Client envia request com header Idempotency-Key
2. Server verifica se key já existe no cache/DB
   → Se sim: retorna resposta armazenada (sem reprocessar)
   → Se não: processa, armazena resposta com a key, retorna
3. TTL da key: 24-48h
4. Em caso de request em andamento com mesma key: retornar 409 Conflict
```

---

## 13. Compressão e Performance

### Compressão

```http
# Request
GET /api/v1/products HTTP/1.1
Accept-Encoding: gzip, br

# Response
HTTP/1.1 200 OK
Content-Encoding: gzip
```

### Boas Práticas de Performance

| Prática | Descrição |
|---------|-----------|
| **Compressão gzip/brotli** | Reduz tamanho do payload em 70-90% |
| **Sparse Fieldsets** | `?fields=id,name` retorna apenas campos solicitados |
| **Bulk Operations** | Endpoint para operações em lote |
| **Connection Keep-Alive** | Reutilizar conexões TCP |
| **HTTP/2** | Multiplexação de requests |
| **Payloads enxutos** | Não retorne dados desnecessários |
| **Database optimization** | Paginação, índices, projeções |

### Bulk/Batch Endpoint

```http
POST /api/v1/users/bulk
Content-Type: application/json

{
  "operations": [
    { "method": "POST", "body": { "name": "User 1", "email": "u1@ex.com" } },
    { "method": "POST", "body": { "name": "User 2", "email": "u2@ex.com" } }
  ]
}
```

---

## 14. Health Checks e Observabilidade

### Health Check Endpoint

```http
GET /api/v1/health

{
  "status": "UP",
  "timestamp": "2026-02-23T10:30:00Z",
  "version": "1.2.3",
  "checks": {
    "database": { "status": "UP", "responseTime": "5ms" },
    "cache": { "status": "UP", "responseTime": "2ms" },
    "externalService": { "status": "UP", "responseTime": "120ms" }
  }
}
```

### Observabilidade

| Pilar | Implementação |
|-------|--------------|
| **Logging** | Structured logging (JSON) com correlation ID |
| **Metrics** | Latência (p50/p95/p99), throughput, error rate |
| **Tracing** | Distributed tracing com OpenTelemetry |

### Headers de Correlação

```http
# Request
X-Request-Id: 550e8400-e29b-41d4-a716-446655440000
X-Correlation-Id: parent-trace-id

# Response
X-Request-Id: 550e8400-e29b-41d4-a716-446655440000
```

---

## 15. Boas Práticas de Payload

### Request

```json
{
  "name": "João Silva",
  "email": "joao@example.com",
  "birthDate": "1990-05-15",
  "address": {
    "street": "Rua Example, 123",
    "city": "São Paulo",
    "state": "SP",
    "zipCode": "01001-000",
    "country": "BR"
  }
}
```

### Convenções

| Convenção | Padrão |
|-----------|--------|
| Formato de propriedades | `camelCase` (mais comum) ou `snake_case` (consistente) |
| Datas | ISO 8601: `2026-02-23T10:30:00Z` |
| Moedas | Menor unidade (centavos): `5000` = R$ 50,00 |
| Enums | `UPPER_SNAKE_CASE`: `"PAYMENT_PENDING"` |
| IDs | String (UUID) ou Number — seja consistente |
| Booleanos | Prefixo `is`/`has`: `"isActive": true` |
| Nulls | Omitir campos nulos ou retornar `null` explicitamente |
| Arrays vazios | Retornar `[]`, nunca `null` |

### Envelope de Resposta (Padrão Recomendado)

```json
{
  "data": {
    "id": "uuid-123",
    "name": "João Silva"
  },
  "meta": {
    "requestId": "req-abc-123",
    "timestamp": "2026-02-23T10:30:00Z"
  }
}
```

---

## 16. Deprecation Strategy

### Comunicação de Deprecation

```http
HTTP/1.1 200 OK
Deprecation: Mon, 01 Jun 2026 00:00:00 GMT
Sunset: Tue, 01 Dec 2026 00:00:00 GMT
Link: </api/v2/users>; rel="successor-version"
Warning: 299 - "This endpoint is deprecated. Migrate to /api/v2/users by Dec 2026."
```

### Ciclo de Vida da API

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Preview    │ ──▶ │    Active    │ ──▶ │  Deprecated  │ ──▶ │   Retired    │
│  (alpha/beta)│     │  (stable)    │     │  (sunset)    │     │  (removed)   │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
     ~3 meses           ~2+ anos           6-12 meses          Comunicação
                                                                com 6m antec.
```

### Boas Práticas

- Comunique deprecation com **mínimo 6 meses** de antecedência.
- Forneça **guia de migração** detalhado.
- **Monitore uso** da versão deprecated para garantir migração.
- **Notifique consumidores** por email/webhook/dashboard.
- Mantenha **changelog** público e atualizado.

---

## 17. Content Negotiation

### Accept Header

```http
# Client solicita formato específico
GET /api/v1/users/123 HTTP/1.1
Accept: application/json

# Resposta com Content-Type correspondente
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

### Formatos Suportados

| Accept Header | Formato | Uso |
|---------------|---------|-----|
| `application/json` | JSON | Padrão para a maioria das APIs |
| `application/xml` | XML | Legado ou requisitos regulatórios |
| `text/csv` | CSV | Exportação de dados tabulares |
| `application/pdf` | PDF | Relatórios |
| `application/vnd.api+json` | JSON:API | APIs que seguem spec JSON:API |

### Regras

- Sempre retorne `Content-Type` na resposta.
- Se o formato solicitado não for suportado, retorne `406 Not Acceptable`.
- Defina um formato padrão (JSON) quando `Accept` não for enviado.
- Use `Vary: Accept` para cache correto com múltiplos formatos.

---

## 18. Webhooks e Eventos

### Design de Webhooks

```http
# Payload de webhook enviado pelo servidor
POST https://client.example.com/webhooks HTTP/1.1
Content-Type: application/json
X-Webhook-Signature: sha256=abc123...
X-Webhook-Id: evt_abc123
X-Webhook-Timestamp: 2026-02-23T10:30:00Z

{
  "id": "evt_abc123",
  "type": "order.completed",
  "createdAt": "2026-02-23T10:30:00Z",
  "data": {
    "orderId": "ord_456",
    "status": "COMPLETED",
    "totalAmount": 5000
  }
}
```

### Boas Práticas de Webhooks

| Prática | Descrição |
|---------|-----------|
| **Assinatura HMAC** | Assine o payload com HMAC-SHA256 para que o client valide autenticidade |
| **Idempotency** | Inclua `X-Webhook-Id` único — client deve deduplicar |
| **Retry com backoff** | Reenvie em caso de falha (5xx ou timeout) com backoff exponencial |
| **Timeout curto** | Aguarde no máximo 5-10s pela resposta do client |
| **Payload enxuto** | Envie dados essenciais — client busca detalhes via API se necessário |
| **Registro de eventos** | Permita que clients listem eventos passados via endpoint de histórico |
| **Versionamento** | Versione o payload de webhooks junto com a API |

### Retry Policy

```
Tentativa 1: imediata
Tentativa 2: após 1 minuto
Tentativa 3: após 5 minutos
Tentativa 4: após 30 minutos
Tentativa 5: após 2 horas
Tentativa 6: após 24 horas (final)
```

### Gerenciamento de Webhooks (API para o client)

```http
# Criar subscription
POST /api/v1/webhooks
{
  "url": "https://client.example.com/webhooks",
  "events": ["order.completed", "order.cancelled"],
  "secret": "whsec_..."
}

# Listar subscriptions
GET /api/v1/webhooks

# Listar eventos enviados (histórico)
GET /api/v1/webhooks/{webhookId}/events?status=failed
```

---

## 19. Anti-patterns Comuns

### Evite Estes Erros

| Anti-pattern | Problema | Solução |
|--------------|----------|--------|
| Verbos na URI | `POST /createUser` | Use `POST /users` — método HTTP expressa a ação |
| Status code genérico | Sempre retornar `200` com erro no body | Use status codes HTTP corretos (400, 404, etc.) |
| Expor IDs do banco | `/users/AUTO_INCREMENT_47` | Use UUIDs ou IDs opacos |
| Não paginar listas | `GET /users` retorna 100k registros | Sempre pagine com `page`/`size` ou cursor |
| Ignorar versionamento | Mudar campos sem aviso | Use `/api/v1/` e respeite backward compatibility |
| Aninhamento excessivo | `/a/{id}/b/{id}/c/{id}/d/{id}` | Máximo 3 níveis de profundidade |
| Payload sensível na URL | `/users?token=abc123` | Dados sensíveis no body ou headers |
| Ignorar `Content-Type` | Não validar conteúdo recebido | Valide e rejeite com `415 Unsupported Media Type` |
| Cache sem validação | Cache sem `ETag`/`Last-Modified` | Sempre inclua headers de validação |
| Erros inconsistentes | Formatos de erro diferentes por endpoint | Use RFC 9457 Problem Details em toda a API |

---

## Referências

- [RFC 7231 — HTTP Semantics](https://tools.ietf.org/html/rfc7231)
- [RFC 9457 — Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457) (substitui RFC 7807)
- [RFC 7396 — JSON Merge Patch](https://tools.ietf.org/html/rfc7396)
- [RFC 8288 — Web Linking](https://tools.ietf.org/html/rfc8288)
- [RFC 6585 — Additional HTTP Status Codes (429)](https://tools.ietf.org/html/rfc6585)
- [RFC 9110 — HTTP Semantics (substitui RFC 7231)](https://www.rfc-editor.org/rfc/rfc9110)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [Stripe API Reference](https://stripe.com/docs/api)
- [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/)
- [OpenAPI Specification 3.1](https://spec.openapis.org/oas/latest.html)
- [JSON:API Specification](https://jsonapi.org/)
