# Level 4 — GraphQL Advanced: Performance & Escala

> **Objetivo:** Tornar a API GraphQL do Level 3 production-ready com DataLoader,
> segurança contra queries maliciosas, caching, observability e testing.

**Referência:** [.docs/API/02-graphql-best-practices.md](../../../.docs/API/02-graphql-best-practices.md) — Seções 9–17

---

## Desafio 4.1 — DataLoader (N+1 Problem)

**Contexto:** Sem DataLoader, uma query que lista 20 produtos com seus sellers
gera 1 query para produtos + 20 queries para sellers (N+1).
DataLoader agrupa todas as chamadas em um único batch.

### Requisitos

**O problema:**

```graphql
query {
  products(first: 20) {
    edges {
      node {
        name
        seller { name }     # ← sem DataLoader: 20 chamadas individuais
        category { name }   # ← sem DataLoader: mais 20 chamadas
      }
    }
  }
}
# Sem DataLoader: 1 + 20 + 20 = 41 "queries"
# Com DataLoader: 1 + 1 + 1 = 3 "queries" (batched)
```

**Implementar DataLoader para:**
- `User` (seller) — batch por IDs
- `Category` — batch por IDs
- `Reviews` — batch por productIds
- `OrderItems` — batch por orderIds

**Java 25:**

```java
class DataLoader<K, V> {
    private final Function<Set<K>, Map<K, V>> batchFn;
    private final Map<K, CompletableFuture<V>> cache = new ConcurrentHashMap<>();
    private final Set<K> pending = ConcurrentHashMap.newKeySet();
    private final ScheduledExecutorService scheduler;

    DataLoader(Function<Set<K>, Map<K, V>> batchFn) {
        this.batchFn = batchFn;
        this.scheduler = Executors.newSingleThreadScheduledExecutor(Thread.ofVirtual().factory());
    }

    CompletableFuture<V> load(K key) {
        return cache.computeIfAbsent(key, k -> {
            pending.add(k);
            // Tick: batch all pending keys on next microtask
            var future = new CompletableFuture<V>();
            scheduler.schedule(this::dispatch, 1, java.util.concurrent.TimeUnit.MILLISECONDS);
            return future;
        });
    }

    private void dispatch() {
        if (pending.isEmpty()) return;
        var keys = Set.copyOf(pending);
        pending.clear();
        var results = batchFn.apply(keys);
        keys.forEach(key -> cache.get(key).complete(results.get(key)));
    }
}

// Uso
var userLoader = new DataLoader<String, User>(ids ->
    userStore.findByIds(ids) // batch: SELECT * FROM users WHERE id IN (...)
);
```

**Go 1.26:**

```go
type DataLoader[K comparable, V any] struct {
    batchFn func(keys []K) map[K]V
    cache   sync.Map
    pending chan K
    results map[K]chan V
    mu      sync.Mutex
}

func NewDataLoader[K comparable, V any](batchFn func(keys []K) map[K]V) *DataLoader[K, V] {
    dl := &DataLoader[K, V]{
        batchFn: batchFn,
        pending: make(chan K, 100),
        results: make(map[K]chan V),
    }
    go dl.batchLoop()
    return dl
}

func (dl *DataLoader[K, V]) Load(key K) V {
    if val, ok := dl.cache.Load(key); ok {
        return val.(V)
    }
    ch := make(chan V, 1)
    dl.mu.Lock()
    dl.results[key] = ch
    dl.mu.Unlock()
    dl.pending <- key
    return <-ch
}
```

**Métricas a coletar:**
- Número de batch calls vs chamadas individuais (savings ratio)
- Cache hit rate
- Batch size médio

### Critérios de Aceite

- [ ] DataLoader implementado com batching (agrupa keys pendentes)
- [ ] N+1 eliminado: query com 20 products + sellers = 3 batch calls (não 41)
- [ ] Cache por request (limpo entre requests)
- [ ] DataLoaders para User, Category, Reviews, OrderItems
- [ ] Métricas de batch size e cache hit rate logadas
- [ ] Testes comparando chamadas com/sem DataLoader
- [ ] Thread-safe (múltiplas goroutines/virtual threads)

---

## Desafio 4.2 — Segurança: Query Depth & Complexity

**Contexto:** GraphQL permite queries recursivas que podem derrubar o servidor.
Implemente proteções contra queries maliciosas.

### Requisitos

**1. Query Depth Limiting:**

```graphql
# Depth 1 — OK
{ products { name } }

# Depth 3 — OK
{ products { edges { node { seller { name } } } } }

# Depth 7 — BLOCKED
{ products { edges { node { seller { products { edges { node { seller { name } } } } } } } } }
```

- Limite padrão: depth ≤ 5
- Queries que excedem → erro antes de executar

**2. Query Complexity Analysis:**

Atribuir custo a cada campo:

| Campo | Custo | Multiplicador |
|-------|:-----:|:------------:|
| Scalar field | 0 | — |
| Object field | 1 | — |
| Connection field | 2 | `first` argument |
| Resolver com I/O | 3 | — |

```graphql
# Complexidade: products(first:20)*2 + 20*(seller*1 + reviews(first:5)*2 + 5*(author*1)) = 40 + 20 + 200 + 100 = 360
query {
  products(first: 20) {       # custo: 20 * 2 = 40
    edges {
      node {
        seller { name }       # custo: 20 * 1 = 20
        reviews(first: 5) {   # custo: 20 * 5 * 2 = 200
          edges {
            node {
              author { name } # custo: 20 * 5 * 1 = 100
            }
          }
        }
      }
    }
  }
}
# Total: 360 (limite: 1000)
```

- Limite de complexidade: 1000 pontos
- Calcular antes de executar → rejeitar se excede

**3. Persisted Queries (APQ — Automatic Persisted Queries):**

```
# Primeira request: envia query completa + hash
POST /graphql
{
  "query": "query GetProduct($id: ID!) { ... }",
  "extensions": {
    "persistedQuery": {
      "version": 1,
      "sha256Hash": "abc123..."
    }
  }
}

# Requests seguintes: apenas o hash
POST /graphql
{
  "extensions": {
    "persistedQuery": {
      "version": 1,
      "sha256Hash": "abc123..."
    }
  }
}
```

- Armazenar hash → query em cache
- Em produção: bloquear queries que não estão no registro (whitelist)

**4. Field-Level Authorization:**

```graphql
type User {
  id: ID!
  name: String!               # público
  email: String!              # @auth(requires: OWNER)    — só o próprio user
  orders: OrderConnection!    # @auth(requires: OWNER)    — só o próprio user
  role: UserRole!             # @auth(requires: ADMIN)    — só admin
}
```

- Campos protegidos retornam `null` se não autorizado
- Log de tentativa de acesso não autorizado
- Implementar como directive `@auth` ou middleware no resolver

### Critérios de Aceite

- [ ] Query depth limit (≤ 5) bloqueia queries profundas antes da execução
- [ ] Complexity analysis calcula custo e bloqueia queries caras (> 1000)
- [ ] Persisted queries cacheia hash → query
- [ ] Mode whitelist bloqueia queries não registradas
- [ ] Field-level auth: campos protegidos retornam null se não autorizado
- [ ] Mensagens de erro claras para cada tipo de bloqueio
- [ ] Testes: query aceita (depth 3), query bloqueada (depth 7), complexity no limite

---

## Desafio 4.3 — Caching & Performance

**Contexto:** GraphQL não se beneficia de caching HTTP nativo como REST.
Implemente caching em múltiplas camadas.

### Requisitos

**1. Response Caching (server-side):**

```graphql
type Product @cacheControl(maxAge: 300) {
  id: ID!
  name: String!
  pricing: Pricing! @cacheControl(maxAge: 60)     # preço muda mais frequentemente
  reviews: ReviewConnection! @cacheControl(maxAge: 30)
}

type User @cacheControl(maxAge: 0, scope: PRIVATE) {
  id: ID!
  email: String!   # nunca cachear PII
}
```

- Implementar directive `@cacheControl(maxAge: Int, scope: PUBLIC | PRIVATE)`
- Cache key = hash da query normalizada + variáveis
- Cache mais restritivo ganha (se um campo é `maxAge: 30`, response inteira é `maxAge: 30`)
- Setar `Cache-Control` header na response HTTP baseado no `@cacheControl`

**2. Normalized Cache (simulação client-side):**

```
Armazenamento normalizado por __typename + id:
{
  "Product:01J5A3K": { "id": "01J5A3K", "name": "Keyboard", "pricing": "ref:Pricing:01J5A3K" },
  "User:01J5A3M": { "id": "01J5A3M", "name": "John" },
  "Pricing:01J5A3K": { "amount": 15999, "currency": "BRL" }
}

Query: product(id: "01J5A3K") { name } → cache hit
Mutation: updateProduct → invalidate "Product:01J5A3K"
```

- Implementar store normalizado no servidor (para demonstrar o conceito)
- Write-through: mutations atualizam o cache
- Cache invalidation em mutations

**3. Query Optimization:**
- `@defer`: marcar campos lentos para carregamento incremental
- Implementar response streaming com multipart para `@defer`

### Critérios de Aceite

- [ ] `@cacheControl` directive funcional com maxAge e scope
- [ ] Cache key baseada em query + variáveis normalizadas
- [ ] Scope `PRIVATE` nunca retorna `Cache-Control: public`
- [ ] Response max-age = min(todos os campos da query)
- [ ] Normalized cache com invalidation em mutations
- [ ] `@defer` funcional com streaming multipart
- [ ] Testes: cache hit, cache miss, cache invalidation após mutation

---

## Desafio 4.4 — Schema Evolution & Deprecation

**Contexto:** Schemas GraphQL evoluem sem versionamento de URL.
Implemente deprecation gradual de campos.

### Requisitos

**Deprecation de campos:**

```graphql
type Product {
  id: ID!
  name: String!

  "Use pricing.amount instead"
  price: Int @deprecated(reason: "Use pricing.amount instead")

  pricing: Pricing!

  "Use categories instead"
  category: String @deprecated(reason: "Use categories instead. Will be removed 2026-03-01")

  categories: [Category!]!
}
```

**Regras de evolução:**
- Nunca remover campos — deprecar primeiro, remover depois
- Novos campos são sempre opcionais (nullable) ou com default
- Novos enum values não quebram clients existentes
- Campos deprecated logados quando usados (para medir adoção)

**Implementar campo deprecated tracker:**

```json
// GET /graphql/deprecated-usage (admin endpoint)
{
  "deprecatedFields": [
    {
      "field": "Product.price",
      "usageCount": 1523,
      "lastUsed": "2025-01-14T23:59:00Z",
      "deprecatedSince": "2025-06-01",
      "removalDate": "2026-03-01",
      "replacement": "pricing.amount"
    }
  ]
}
```

**Schema Registry (em memória):**
- Manter histórico de versões do schema
- Validar backward compatibility entre versões
- Detectar breaking changes: campo removido, tipo alterado, non-null adicionado

### Critérios de Aceite

- [ ] `@deprecated` funcional com reason
- [ ] Campos deprecated ainda funcionam (backward compatible)
- [ ] Uso de campos deprecated logado com contador
- [ ] Endpoint admin mostra uso de campos deprecated
- [ ] Novos campos adicionados sem quebrar queries existentes
- [ ] Schema registry com histórico de versões
- [ ] Breaking change detection (campo removido = error)
- [ ] Testes: query com campo deprecated funciona mas é logada

---

## Desafio 4.5 — Testing GraphQL

**Contexto:** GraphQL APIs requerem estratégias de teste específicas:
unit, integration, contract e performance.

### Requisitos

**1. Unit Tests (Resolvers):**

```java
// Java — testar resolver isoladamente
@Test
void productResolver_returnsProduct_whenExists() {
    var store = new InMemoryProductStore();
    store.save(new Product("p1", "Keyboard", 15999, "BRL"));

    var resolver = new ProductResolver(store);
    var result = resolver.product("p1");

    assertThat(result).isNotNull();
    assertThat(result.name()).isEqualTo("Keyboard");
    assertThat(result.pricing().amount()).isEqualTo(15999);
}

@Test
void createProduct_returnsErrors_whenInvalidInput() {
    var resolver = new ProductMutationResolver(store, validator);
    var input = new CreateProductInput("", -100, "XXX", null, 0, List.of());

    var payload = resolver.createProduct(input);

    assertThat(payload.product()).isNull();
    assertThat(payload.errors()).hasSize(3);
    assertThat(payload.errors()).extracting(UserError::field)
        .containsExactlyInAnyOrder("name", "price", "currency");
}
```

**2. Integration Tests (Full GraphQL execution):**

```go
// Go — enviar query real ao engine
func TestProductQuery(t *testing.T) {
    server := setupTestServer() // com seed data
    defer server.Close()

    query := `query { product(id: "p1") { id name pricing { formatted } } }`
    resp := executeGraphQL(t, server.URL, query, nil)

    assert.Nil(t, resp.Errors)
    assert.Equal(t, "Keyboard", resp.Data["product"].(map[string]any)["name"])
}

func TestCreateProductMutation(t *testing.T) {
    server := setupTestServer()
    defer server.Close()

    mutation := `mutation($input: CreateProductInput!) {
        createProduct(input: $input) {
            product { id name }
            errors { field message code }
        }
    }`
    variables := map[string]any{
        "input": map[string]any{
            "name": "New Product", "price": 9999,
            "currency": "BRL", "categoryId": "cat1", "stock": 10,
        },
    }

    resp := executeGraphQL(t, server.URL, mutation, variables)
    assert.Nil(t, resp.Errors)
    assert.NotNil(t, resp.Data["createProduct"].(map[string]any)["product"])
}
```

**3. Schema Contract Tests:**

- Validar que schema não tem breaking changes entre versões
- Queries de clientes conhecidos continuam funcionando
- Tipos não removidos, campos non-null não adicionados

**4. Performance Tests:**

- Query simples (`product(id)`) deve resolver em < 10ms
- Query complexa (products + sellers + reviews) em < 100ms
- Medir impacto do DataLoader (before/after)
- Medir complexity de queries reais

### Critérios de Aceite

- [ ] Unit tests para cada resolver (query + mutation)
- [ ] Integration tests com GraphQL engine real
- [ ] Tests para cada tipo de erro (validation, not found, unauthorized)
- [ ] Schema contract tests (backward compatibility)
- [ ] Performance tests com timing assertions
- [ ] DataLoader tests (batch size, cache hit rate)
- [ ] Mínimo 80% code coverage nos resolvers

---

## Desafio 4.6 — Observability (Logging, Metrics, Tracing)

**Contexto:** Monitorar uma API GraphQL requer instrumentação específica
porque todas as queries passam pelo mesmo endpoint.

### Requisitos

**1. Structured Logging:**

```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "INFO",
  "message": "graphql_request",
  "requestId": "req-550e8400",
  "operationName": "GetProduct",
  "operationType": "query",
  "complexity": 42,
  "depth": 3,
  "duration_ms": 15,
  "errors": 0,
  "userId": "user-123",
  "deprecatedFieldsUsed": ["Product.price"]
}
```

**2. Metrics (counters e histograms):**

| Métrica | Tipo | Labels |
|---------|------|--------|
| `graphql_requests_total` | Counter | operation_type, operation_name, status |
| `graphql_request_duration_ms` | Histogram | operation_type, operation_name |
| `graphql_errors_total` | Counter | error_type (validation, auth, resolver) |
| `graphql_complexity` | Histogram | operation_name |
| `graphql_depth` | Histogram | operation_name |
| `graphql_deprecated_field_usage` | Counter | field_name |
| `dataloader_batch_size` | Histogram | loader_name |
| `dataloader_cache_hit_rate` | Gauge | loader_name |

**3. Distributed Tracing (OpenTelemetry-style):**

```
Trace: GetProduct (15ms)
├── Parse Query (0.5ms)
├── Validate Query (0.3ms)
├── Execute (14ms)
│   ├── Resolve product (2ms)
│   ├── Resolve product.seller (via DataLoader, 5ms)
│   ├── Resolve product.reviews (via DataLoader, 6ms)
│   └── Resolve product.category (cache hit, 0.1ms)
└── Serialize Response (0.2ms)
```

- Cada resolver cria um span filho
- DataLoader batch cria span com batch size
- Cache hit/miss no span attributes

**Implementação:**

```java
// Java — middleware de tracing
class TracingInstrumentation {
    record Span(String name, Instant start, Map<String, Object> attributes) {
        Span complete() {
            attributes.put("duration_ms", Duration.between(start, Instant.now()).toMillis());
            return this;
        }
    }

    List<Span> spans = new CopyOnWriteArrayList<>();

    Object instrumentResolver(String fieldName, Supplier<Object> resolver) {
        var span = new Span("resolve." + fieldName, Instant.now(), new ConcurrentHashMap<>());
        try {
            var result = resolver.get();
            span.attributes().put("status", "ok");
            return result;
        } catch (Exception e) {
            span.attributes().put("status", "error");
            span.attributes().put("error.message", e.getMessage());
            throw e;
        } finally {
            spans.add(span.complete());
        }
    }
}
```

### Critérios de Aceite

- [ ] Cada request logado com operationName, type, complexity, depth, duration
- [ ] Métricas coletadas: requests total, duration histogram, errors, deprecated usage
- [ ] Tracing: span por resolver com parent-child relationship
- [ ] DataLoader métricas: batch size, cache hit rate
- [ ] Campos deprecated logados quando usados
- [ ] Endpoint `/metrics` expõe métricas em formato Prometheus-like (text)
- [ ] Testes verificam que métricas são incrementadas corretamente

---

## Desafio 4.7 — Federation & BFF Pattern

**Contexto:** Em arquiteturas de microserviços, cada serviço pode expor seu
próprio schema GraphQL. Federation compõe um schema unificado.

### Requisitos

**Implementar 3 subgraphs + 1 gateway:**

```
┌─────────────────────────────────────────────┐
│                 API Gateway                  │
│            (Schema Composition)              │
└──────┬──────────────┬──────────────┬────────┘
       │              │              │
┌──────▼──────┐ ┌─────▼──────┐ ┌────▼─────────┐
│  Products   │ │   Orders   │ │    Users     │
│  Subgraph   │ │  Subgraph  │ │  Subgraph    │
│  :4001      │ │  :4002     │ │  :4003       │
└─────────────┘ └────────────┘ └──────────────┘
```

**Products Subgraph:**

```graphql
type Product @key(fields: "id") {
  id: ID!
  name: String!
  pricing: Pricing!
  category: Category!
  stock: Int!
}
```

**Orders Subgraph:**

```graphql
type Order @key(fields: "id") {
  id: ID!
  items: [OrderItem!]!
  status: OrderStatus!
  total: Int!
  buyer: User!      # referência externa
}

extend type User @key(fields: "id") {
  id: ID! @external
  orders: [Order!]!
}
```

**Users Subgraph:**

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
  email: String!
  role: UserRole!
}
```

**Gateway:** Compõe os 3 schemas em um único schema e roteia queries para os subgraphs corretos.

**BFF Pattern (Backend for Frontend):**

Criar 2 "views" otimizadas:

```graphql
# Mobile BFF — mínimo de dados
query MobileProductList {
  products(first: 10) {
    edges {
      node {
        id
        name
        pricing { formatted }
      }
    }
  }
}

# Web BFF — dados completos
query WebProductPage($id: ID!) {
  product(id: $id) {
    id
    name
    description
    pricing { amount currency formatted }
    category { name slug }
    seller { name }
    reviews(first: 10) {
      edges {
        node { rating comment author { name } }
      }
      pageInfo { hasNextPage endCursor }
    }
  }
}
```

### Critérios de Aceite

- [ ] 3 subgraphs rodando em portas diferentes
- [ ] Gateway compõe schema unificado
- [ ] Query cross-subgraph funciona (Order → Product → Category)
- [ ] `@key` directive para entity resolution
- [ ] `extend type` funcional para estender tipos de outro subgraph
- [ ] BFF: mesma API serve queries mobile (leves) e web (completas)
- [ ] Performance: subgraph calls em paralelo quando independentes
- [ ] Testes: query que toca os 3 subgraphs

---

## Anti-Patterns a Evitar

| Anti-Pattern | Correto |
|-------------|---------|
| N+1 queries (sem DataLoader) | DataLoader com batching |
| Sem limite de depth/complexity | Query depth ≤ 5, complexity ≤ 1000 |
| Aceitar queries arbitrárias em prod | Persisted queries (whitelist) |
| Sem field-level auth | `@auth` directive por campo sensível |
| Cache HTTP = cache GraphQL | Normalized cache + `@cacheControl` |
| Monolith schema (tudo junto) | Federation com subgraphs |
| Log genérico (path, status) | Logs com operationName, complexity, depth |

---

## Próximo Nível

→ [Level 5 — gRPC: Protobuf & Comunicação](05-grpc.md)
