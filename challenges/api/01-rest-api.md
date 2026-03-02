# Level 1 — REST API: Design & Implementação

> **Objetivo:** Projetar e implementar uma REST API completa para o Marketplace,
> aplicando URI design, semântica HTTP e convenções de payload do mundo real.

**Referência:** [.docs/API/01-rest-best-practices.md](../../../.docs/API/01-rest-best-practices.md) — Seções 1–6

---

## Desafio 1.1 — URI Design & Resource Modeling

**Contexto:** URIs são o contrato público da sua API. Um design ruim de URI gera
acoplamento, confusão e quebra de clientes. Modele os recursos do Marketplace.

### Requisitos

Projetar e implementar a hierarquia de recursos:

```
/products                       → Coleção de produtos
/products/{productId}           → Produto específico
/products/{productId}/reviews   → Reviews de um produto
/products/{productId}/reviews/{reviewId}

/users                          → Coleção de usuários
/users/{userId}                 → Usuário específico
/users/{userId}/orders          → Pedidos de um usuário
/users/{userId}/addresses       → Endereços de um usuário

/orders                         → Coleção de pedidos (admin)
/orders/{orderId}               → Pedido específico
/orders/{orderId}/items         → Itens de um pedido
/orders/{orderId}/payments      → Pagamentos de um pedido

/categories                     → Coleção de categorias
/categories/{categoryId}/products → Produtos de uma categoria
```

**Regras de URI Design:**

| Regra | Correto | Incorreto |
|-------|---------|-----------|
| Substantivos no plural | `/products` | `/product`, `/getProducts` |
| Lowercase com hífens | `/order-items` | `/orderItems`, `/OrderItems` |
| Sem verbos na URI | `/products` + POST | `/createProduct` |
| Máximo 3 níveis | `/products/{id}/reviews` | `/users/{id}/orders/{id}/items/{id}/details` |
| IDs opacos | `/products/a1b2c3` | `/products/1` (sequencial expõe volume) |
| Sem trailing slash | `/products` | `/products/` |

### Critérios de Aceite

- [ ] Todas as URIs usam substantivos no plural
- [ ] Hierarquia de no máximo 3 níveis de profundidade
- [ ] IDs opacos (UUID v7 para ordenação temporal)
- [ ] Sem verbos nas URIs
- [ ] Lowercase com hífens (não camelCase/snake_case)
- [ ] Router configurado para todas as rotas da tabela
- [ ] `405 Method Not Allowed` para métodos não suportados em uma rota
- [ ] Testes para cada rota retornando o status correto

---

## Desafio 1.2 — CRUD Completo de Produtos

**Contexto:** O catálogo de produtos é o core do Marketplace.
Implemente CRUD completo com semântica HTTP correta.

### Requisitos

**Modelo Product:**
```json
{
  "id": "01J5A3K...",
  "name": "Mechanical Keyboard",
  "description": "RGB mechanical keyboard with Cherry MX switches",
  "price": 15999,
  "currency": "BRL",
  "category": "electronics",
  "sellerId": "01J5A3M...",
  "stock": 42,
  "status": "active",
  "tags": ["keyboard", "gaming", "rgb"],
  "createdAt": "2025-01-15T10:30:00Z",
  "updatedAt": "2025-01-15T10:30:00Z"
}
```

**Operações:**

| Operação | Método + URI | Status Sucesso | Body Response |
|----------|-------------|:--------------:|---------------|
| Listar | `GET /products` | 200 | Array de products |
| Detalhar | `GET /products/{id}` | 200 | Product |
| Criar | `POST /products` | 201 | Product criado + `Location` header |
| Substituir | `PUT /products/{id}` | 200 | Product atualizado |
| Atualizar parcial | `PATCH /products/{id}` | 200 | Product atualizado |
| Remover | `DELETE /products/{id}` | 204 | Sem body |

**Convenções de Payload:**
- Campos em `camelCase`
- Datas em ISO 8601 com timezone (`Z` ou offset)
- Preços em **centavos** (integer) — nunca float para dinheiro
- `currency` explícito (ISO 4217)
- `createdAt` / `updatedAt` gerenciados pelo servidor
- `id` gerado pelo servidor (UUID v7)

### Java 25

```java
record Product(
    String id,
    String name,
    String description,
    int price,          // centavos
    String currency,
    String category,
    String sellerId,
    int stock,
    String status,
    List<String> tags,
    Instant createdAt,
    Instant updatedAt
) {
    Product {
        if (name == null || name.isBlank()) throw new IllegalArgumentException("name is required");
        if (price < 0) throw new IllegalArgumentException("price must be >= 0");
        if (stock < 0) throw new IllegalArgumentException("stock must be >= 0");
    }
}

// POST /products
case "POST" -> {
    var input = parseJson(exchange, CreateProductRequest.class);
    var product = new Product(
        UUID.randomUUID().toString(), input.name(), input.description(),
        input.price(), input.currency(), input.category(), input.sellerId(),
        input.stock(), "active", input.tags(), Instant.now(), Instant.now()
    );
    products.put(product.id(), product);

    exchange.getResponseHeaders().set("Location", "/products/" + product.id());
    sendJson(exchange, 201, product);
}
```

### Go 1.26

```go
type Product struct {
    ID          string    `json:"id"`
    Name        string    `json:"name"`
    Description string    `json:"description"`
    Price       int       `json:"price"`       // centavos
    Currency    string    `json:"currency"`
    Category    string    `json:"category"`
    SellerID    string    `json:"sellerId"`
    Stock       int       `json:"stock"`
    Status      string    `json:"status"`
    Tags        []string  `json:"tags"`
    CreatedAt   time.Time `json:"createdAt"`
    UpdatedAt   time.Time `json:"updatedAt"`
}

// POST /products
mux.HandleFunc("POST /products", func(w http.ResponseWriter, r *http.Request) {
    var input CreateProductRequest
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        writeError(w, ProblemDetail{Status: 400, Detail: "invalid JSON"})
        return
    }
    product := Product{
        ID:        uuid.New().String(),
        Name:      input.Name,
        Price:     input.Price,
        Currency:  input.Currency,
        Stock:     input.Stock,
        Status:    "active",
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }
    store[product.ID] = product

    w.Header().Set("Location", "/products/"+product.ID)
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(product)
})
```

### Critérios de Aceite

- [ ] CRUD completo funciona para Product
- [ ] `POST` retorna 201 + `Location` header com URI do recurso criado
- [ ] `PUT` exige todos os campos (replace completo), retorna 200
- [ ] `PATCH` aceita campos parciais (merge), retorna 200
- [ ] `DELETE` retorna 204 sem body
- [ ] Preços em centavos (integer, nunca float)
- [ ] Datas em ISO 8601 com timezone
- [ ] `id`, `createdAt`, `updatedAt` gerenciados pelo servidor
- [ ] Validação retorna 400 com Problem Details (RFC 9457) do Level 0
- [ ] Recurso inexistente retorna 404

---

## Desafio 1.3 — Relacionamentos e Sub-Recursos

**Contexto:** Usuários fazem pedidos que contêm itens. Modele os relacionamentos
sem expor a estrutura interna do banco de dados.

### Requisitos

Implementar CRUD para `Order` com sub-recursos:

```json
// POST /users/{userId}/orders
{
  "items": [
    {"productId": "01J5A3K...", "quantity": 2},
    {"productId": "01J5A3M...", "quantity": 1}
  ],
  "shippingAddressId": "01J5A3N..."
}

// Response: 201 Created
{
  "id": "01J5A3P...",
  "userId": "01J5A3Q...",
  "status": "pending",
  "items": [
    {
      "id": "01J5A3R...",
      "productId": "01J5A3K...",
      "productName": "Mechanical Keyboard",
      "quantity": 2,
      "unitPrice": 15999,
      "totalPrice": 31998
    }
  ],
  "subtotal": 47997,
  "shippingCost": 1500,
  "total": 49497,
  "currency": "BRL",
  "createdAt": "2025-01-15T10:30:00Z"
}
```

**Operações de Order:**

| Operação | Método + URI | Notas |
|----------|-------------|-------|
| Criar pedido | `POST /users/{userId}/orders` | Valida stock, calcula totais |
| Listar pedidos do user | `GET /users/{userId}/orders` | Somente pedidos do owner |
| Detalhar pedido | `GET /orders/{orderId}` | Acesso global por ID |
| Listar itens do pedido | `GET /orders/{orderId}/items` | Sub-recurso |
| Cancelar pedido | `POST /orders/{orderId}/cancel` | Ação (exceção à regra sem verbos) |

**Regras de negócio:**
- Verificar estoque antes de criar pedido
- Calcular `subtotal` (soma de `unitPrice * quantity`)
- Definir `shippingCost` fixo por enquanto
- Status workflow: `pending` → `confirmed` → `shipped` → `delivered`
- Cancelar só é possível em status `pending` ou `confirmed`

### Critérios de Aceite

- [ ] Order criada com validação de estoque
- [ ] Totais calculados corretamente (centavos)
- [ ] Sub-recursos (`/orders/{id}/items`) retornam itens do pedido
- [ ] Ações de estado (`/orders/{id}/cancel`) usam POST
- [ ] `GET /users/{userId}/orders` retorna somente pedidos daquele user
- [ ] Status workflow respeitado (não pula estados)
- [ ] Cancelar pedido `shipped` retorna 409 Conflict
- [ ] Testes para fluxo completo: criar → confirmar → cancelar

---

## Desafio 1.4 — HATEOAS & Discoverability

**Contexto:** APIs verdadeiramente RESTful são auto-descritivas.
Implemente HATEOAS (Hypermedia as the Engine of Application State) usando HAL.

### Requisitos

Adicionar links HAL (`_links`) a todas as respostas:

```json
// GET /products/01J5A3K...
{
  "id": "01J5A3K...",
  "name": "Mechanical Keyboard",
  "price": 15999,
  "_links": {
    "self": {"href": "/products/01J5A3K..."},
    "reviews": {"href": "/products/01J5A3K.../reviews"},
    "category": {"href": "/categories/electronics"},
    "seller": {"href": "/users/01J5A3M..."},
    "update": {"href": "/products/01J5A3K...", "method": "PUT"},
    "delete": {"href": "/products/01J5A3K...", "method": "DELETE"}
  }
}

// GET /products (coleção com links de navegação)
{
  "data": [...],
  "total": 150,
  "_links": {
    "self": {"href": "/products?page=2&size=20"},
    "first": {"href": "/products?page=1&size=20"},
    "prev": {"href": "/products?page=1&size=20"},
    "next": {"href": "/products?page=3&size=20"},
    "last": {"href": "/products?page=8&size=20"}
  }
}
```

- Links condicionais: `delete` e `update` só aparecem se o user tem permissão
- Orders devem incluir links de ações disponíveis baseado no status:
  - `pending` → links para `confirm`, `cancel`
  - `confirmed` → link para `ship`, `cancel`
  - `shipped` → link para `deliver`
  - `delivered` → nenhuma ação

### Critérios de Aceite

- [ ] Todas as respostas incluem `_links.self`
- [ ] Recursos relacionados linkados (product → reviews, category, seller)
- [ ] Coleções incluem links de navegação (first, prev, next, last)
- [ ] Links de ação condicionais baseados em permissão
- [ ] Links de ação de Order variam conforme status
- [ ] `Content-Type: application/hal+json`
- [ ] Testes validam presença de links em cada cenário

---

## Desafio 1.5 — Validação Robusta

**Contexto:** Dados inválidos devem ser rejeitados antes de atingir a lógica de negócio.
Implemente validação em camadas.

### Requisitos

**Camadas de validação:**

```
Request → [Sintática] → [Semântica] → [Negócio] → Handler
              │              │              │
          JSON válido?   Campos OK?    Regras OK?
          Tipos corretos? Formatos?     Estoque?
```

1. **Validação Sintática:**
   - JSON bem-formado (400 se malformado)
   - Campos obrigatórios presentes
   - Tipos corretos (string, number, array)

2. **Validação Semântica:**
   - `name`: 3–200 caracteres, não apenas espaços
   - `price`: ≥ 0, integer
   - `currency`: ISO 4217 válido (BRL, USD, EUR)
   - `stock`: ≥ 0, integer
   - `tags`: array com 0–10 itens, cada tag 2–50 caracteres
   - `email`: formato válido (regex)

3. **Validação de Negócio:**
   - `sellerId` deve existir
   - Categoria deve existir
   - Não pode criar produto com estoque negativo
   - Email não pode estar duplicado (para User)

**Response para múltiplos erros (retorna TODOS os erros, não apenas o primeiro):**

```json
{
  "type": "https://api.marketplace.com/errors/validation-error",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Request body contains 3 validation errors.",
  "errors": [
    {"field": "name", "message": "must be between 3 and 200 characters", "rejectedValue": "AB"},
    {"field": "price", "message": "must be greater than or equal to 0", "rejectedValue": -100},
    {"field": "currency", "message": "must be a valid ISO 4217 code", "rejectedValue": "XXX"}
  ]
}
```

### Critérios de Aceite

- [ ] JSON malformado retorna 400 com mensagem clara
- [ ] Campos obrigatórios ausentes listados individualmente
- [ ] Todos os erros retornados de uma vez (não fail-fast)
- [ ] Validações semânticas para: tamanho, formato, range, enum
- [ ] Validações de negócio separadas das sintáticas/semânticas
- [ ] Response segue RFC 9457 com array `errors`
- [ ] Testes com pelo menos 10 cenários de validação diferentes
- [ ] Validação é reutilizável (não está inline no handler)

---

## Desafio 1.6 — Payload Conventions & Serialização

**Contexto:** Convenções de payload consistentes reduzem a fricção de integração.
Padronize serialização, campos nulos e envelope de resposta.

### Requisitos

**Convenções obrigatórias:**
- Campos em `camelCase` (JSON) mapeados para `PascalCase` (Go) ou records (Java)
- Datas em ISO 8601 UTC: `"2025-01-15T10:30:00Z"`
- Valores monetários em centavos: `15999` (não `159.99`)
- Currency explícito: `"currency": "BRL"`
- Campos nulos: **omitir** do JSON (não enviar `"field": null`)
- Arrays vazios: enviar `[]` (não omitir)
- Booleanos: usar `true`/`false` (não `0`/`1`)

**Envelope de resposta para coleções:**
```json
{
  "data": [...],
  "meta": {
    "total": 150,
    "page": 2,
    "size": 20,
    "totalPages": 8
  }
}
```

**Envelope de resposta para item único:**
```json
{
  "data": { ... },
  "_links": { ... }
}
```

**Serialização customizada:**
- Implementar serialização de `Instant` → ISO 8601 string
- Implementar serialização de `Money` → `{"amount": 15999, "currency": "BRL"}`
- Implementar deserialização com validação de tipo

### Java 25

```java
record Money(int amount, String currency) {
    Money {
        if (amount < 0) throw new IllegalArgumentException("amount cannot be negative");
        if (!Set.of("BRL", "USD", "EUR").contains(currency))
            throw new IllegalArgumentException("invalid currency: " + currency);
    }

    String formatted() {
        return "%s %.2f".formatted(currency, amount / 100.0);
    }
}

record CollectionResponse<T>(List<T> data, Meta meta) {}
record Meta(int total, int page, int size, int totalPages) {}
```

### Go 1.26

```go
type Money struct {
    Amount   int    `json:"amount"`
    Currency string `json:"currency"`
}

type CollectionResponse[T any] struct {
    Data []T  `json:"data"`
    Meta Meta `json:"meta"`
}

type Meta struct {
    Total      int `json:"total"`
    Page       int `json:"page"`
    Size       int `json:"size"`
    TotalPages int `json:"totalPages"`
}

// Custom JSON: omit null fields
func (p Product) MarshalJSON() ([]byte, error) {
    type Alias Product
    aux := struct {
        Alias
        Description *string `json:"description,omitempty"`
    }{Alias: Alias(p)}
    if p.Description != "" {
        aux.Description = &p.Description
    }
    return json.Marshal(aux)
}
```

### Critérios de Aceite

- [ ] Todos os campos JSON em `camelCase`
- [ ] Datas serializadas em ISO 8601 UTC
- [ ] Preços em centavos (integer, nunca float)
- [ ] Campos nulos omitidos do JSON
- [ ] Arrays vazios enviados como `[]`
- [ ] Coleções usam envelope com `data` + `meta`
- [ ] `Money` type encapsula amount + currency
- [ ] Serialização/deserialização customizada testada
- [ ] Testes de round-trip: serialize → deserialize → igualdade

---

## Desafio 1.7 — OpenAPI 3.1 Specification

**Contexto:** Documentação é parte da API. Escreva a especificação OpenAPI 3.1
para todas as rotas implementadas.

### Requisitos

Criar arquivo `openapi.yaml` com:

```yaml
openapi: "3.1.0"
info:
  title: Marketplace API
  version: 1.0.0
  description: REST API for the Online Marketplace
  contact:
    name: API Team
    email: api@marketplace.com

servers:
  - url: http://localhost:8080
    description: Local development

paths:
  /products:
    get:
      summary: List products
      operationId: listProducts
      tags: [Products]
      parameters:
        - name: page
          in: query
          schema: { type: integer, default: 1, minimum: 1 }
        - name: size
          in: query
          schema: { type: integer, default: 20, minimum: 1, maximum: 100 }
      responses:
        "200":
          description: Products list
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ProductList"
    post:
      summary: Create product
      operationId: createProduct
      tags: [Products]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CreateProductRequest"
      responses:
        "201":
          description: Product created
          headers:
            Location:
              schema: { type: string }
              description: URI of created product

components:
  schemas:
    Product:
      type: object
      required: [id, name, price, currency, status]
      properties:
        id: { type: string, format: uuid }
        name: { type: string, minLength: 3, maxLength: 200 }
        price: { type: integer, minimum: 0, description: "Price in cents" }
        currency: { type: string, enum: [BRL, USD, EUR] }
        # ... demais campos

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

- Cobrir **todos** os endpoints implementados (Products, Orders, Users, Reviews)
- Documentar **todos** os schemas com exemplos
- Documentar **todos** os status codes de erro com `$ref` para `ProblemDetail`
- Incluir `securitySchemes` (Bearer JWT)
- Incluir `tags` para agrupamento
- Implementar endpoint `GET /openapi.yaml` que serve a spec
- Validar spec com ferramenta (swagger-cli ou similar)

### Critérios de Aceite

- [ ] Spec OpenAPI 3.1 cobre todos os endpoints
- [ ] Cada response documenta status code, headers e body schema
- [ ] Schemas incluem `required`, `type`, `format`, `example`
- [ ] `ProblemDetail` schema reutilizado para erros
- [ ] `securitySchemes` definido para Bearer JWT
- [ ] Endpoint `GET /openapi.yaml` serve a spec
- [ ] Spec válida (sem erros de validação)
- [ ] Exemplos funcionais em cada operação

---

## Desafio 1.8 — Health Check & Readiness

**Contexto:** Em ambientes de produção, health checks são essenciais para
load balancers, orquestradores (Kubernetes) e monitoramento.

### Requisitos

Implementar 3 endpoints de health check:

```
GET /health/live     → Liveness (processo vivo?)
GET /health/ready    → Readiness (pronto para receber tráfego?)
GET /health/startup  → Startup (inicialização completa?)
```

**Liveness (`/health/live`)**:
```json
{
  "status": "UP",
  "timestamp": "2025-01-15T10:30:00Z"
}
```

**Readiness (`/health/ready`)** — verifica dependências:
```json
{
  "status": "UP",
  "checks": {
    "datastore": {"status": "UP", "responseTime": "2ms"},
    "diskSpace": {"status": "UP", "free": "10GB", "threshold": "1GB"}
  },
  "timestamp": "2025-01-15T10:30:00Z"
}
```

**Startup (`/health/startup`)** — verifica inicialização:
```json
{
  "status": "UP",
  "uptime": "5m30s",
  "version": "1.0.0",
  "startedAt": "2025-01-15T10:24:30Z"
}
```

- Status `UP` → 200, status `DOWN` → 503
- Se **qualquer** check falha, status geral é `DOWN`
- Liveness **nunca** deve depender de serviços externos (apenas verifica se o processo está vivo)
- Readiness verifica dependências (storage, etc.)
- Incluir `Cache-Control: no-cache` (não cachear health checks)
- Incluir versão e build info no startup

### Critérios de Aceite

- [ ] 3 endpoints distintos: live, ready, startup
- [ ] Liveness sempre retorna rápido (< 10ms), sem dependências externas
- [ ] Readiness verifica pelo menos 2 dependências
- [ ] Status 200 para UP, 503 para DOWN
- [ ] `Cache-Control: no-cache` em todas as health responses
- [ ] Startup inclui versão e uptime
- [ ] Se uma dependência falha, readiness retorna DOWN
- [ ] Testes simulam dependência unhealthy

---

## Anti-Patterns a Evitar

| Anti-Pattern | Correto |
|-------------|---------|
| Verbos na URI (`/getProducts`) | Substantivos: `/products` + GET |
| IDs sequenciais (`/products/1`) | UUIDs opacos: `/products/01J5A3K...` |
| Float para dinheiro (`159.99`) | Integer em centavos: `15999` |
| `camelCase` inconsistente | Padronizar em toda a API |
| Validação fail-fast (só 1 erro) | Retornar todos os erros de uma vez |
| `200 OK` para tudo | Status codes semânticos corretos |
| Spec OpenAPI desatualizada | Spec como source of truth, testada |
| Health check que verifica tudo | Liveness ≠ Readiness ≠ Startup |

---

## Próximo Nível

→ [Level 2 — REST API Advanced: Produção & Operação](02-rest-advanced.md)
