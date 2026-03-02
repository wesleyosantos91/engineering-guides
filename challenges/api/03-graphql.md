# Level 3 — GraphQL: Schema & Operações

> **Objetivo:** Projetar e implementar uma API GraphQL schema-first para o Marketplace,
> dominando queries, mutations, subscriptions e o Relay Connection specification.

**Referência:** [.docs/API/02-graphql-best-practices.md](../../../.docs/API/02-graphql-best-practices.md) — Seções 1–8

---

## Dependências Permitidas

| Java 25 | Go 1.26 |
|---------|---------|
| `graphql-java` (engine) | `github.com/99designs/gqlgen` (codegen) |
| Jackson (JSON) | `encoding/json` (stdlib) |
| `com.sun.net.httpserver` (HTTP) | `net/http` (stdlib) |

> **Nota:** GraphQL é transportado sobre HTTP POST. Você usará o HTTP server do Level 0
> com um único endpoint `POST /graphql`.

---

## Desafio 3.1 — Schema Design (Schema-First)

**Contexto:** No GraphQL, o schema é o contrato. Projete-o antes de escrever qualquer
resolver, usando convenções de nomenclatura consistentes.

### Requisitos

Criar o schema completo do Marketplace em `schema.graphql`:

```graphql
"""
Produto disponível no marketplace.
Preços em centavos para evitar imprecisão de ponto flutuante.
"""
type Product implements Node {
  "Identificador único (UUID v7)"
  id: ID!
  name: String!
  description: String
  pricing: Pricing!
  category: Category!
  seller: User!
  stock: Int!
  status: ProductStatus!
  tags: [String!]!
  reviews(first: Int, after: String): ReviewConnection!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type Pricing {
  amount: Int!
  currency: Currency!
  "Preço formatado para display (ex: 'R$ 159,99')"
  formatted: String!
}

enum ProductStatus {
  ACTIVE
  INACTIVE
  OUT_OF_STOCK
  DISCONTINUED
}

enum Currency {
  BRL
  USD
  EUR
}

type User implements Node {
  id: ID!
  name: String!
  email: String!
  role: UserRole!
  products(first: Int, after: String): ProductConnection!
  orders(first: Int, after: String): OrderConnection!
  createdAt: DateTime!
}

enum UserRole {
  BUYER
  SELLER
  ADMIN
}

type Order implements Node {
  id: ID!
  buyer: User!
  items: [OrderItem!]!
  status: OrderStatus!
  subtotal: Int!
  shippingCost: Int!
  total: Int!
  currency: Currency!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type OrderItem {
  id: ID!
  product: Product!
  quantity: Int!
  unitPrice: Int!
  totalPrice: Int!
}

enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
  CANCELLED
}

type Review implements Node {
  id: ID!
  product: Product!
  author: User!
  rating: Int!
  comment: String
  createdAt: DateTime!
}

type Category implements Node {
  id: ID!
  name: String!
  slug: String!
  parent: Category
  children: [Category!]!
  products(first: Int, after: String): ProductConnection!
}

"Interface Node para Relay Global Object Identification"
interface Node {
  id: ID!
}

"Scalar customizado para datas ISO 8601"
scalar DateTime
```

**Convenções de Nomenclatura:**

| Elemento | Convenção | Exemplo |
|----------|-----------|---------|
| Types | PascalCase | `Product`, `OrderItem` |
| Fields | camelCase | `createdAt`, `unitPrice` |
| Enums | SCREAMING_SNAKE_CASE | `OUT_OF_STOCK`, `PENDING` |
| Input types | sufixo `Input` | `CreateProductInput` |
| Mutation payloads | sufixo `Payload` | `CreateProductPayload` |
| Connections | sufixo `Connection` | `ProductConnection` |

### Critérios de Aceite

- [ ] Schema cobre todas as entidades do Marketplace (Product, User, Order, Review, Category)
- [ ] Convenções de nomenclatura consistentes (PascalCase, camelCase, SCREAMING_SNAKE)
- [ ] Interface `Node` implementada por todos os tipos com ID
- [ ] Scalar `DateTime` definido para datas ISO 8601
- [ ] Docstrings em todos os types e campos principais
- [ ] Enums para status e valores fixos (não strings)
- [ ] `Pricing` como type separado (não campos soltos)
- [ ] Schema válido (sem erros de validação)

---

## Desafio 3.2 — Queries & Fragments

**Contexto:** Queries GraphQL eliminam over-fetching e under-fetching.
Implemente queries com field selection e fragments reutilizáveis.

### Requisitos

**Query type:**

```graphql
type Query {
  "Buscar qualquer node por ID (Relay Global Object Identification)"
  node(id: ID!): Node

  "Listar produtos com filtros"
  products(
    first: Int
    after: String
    filter: ProductFilterInput
    sort: ProductSortInput
  ): ProductConnection!

  "Buscar produto por ID"
  product(id: ID!): Product

  "Buscar usuário por ID"
  user(id: ID!): User

  "Listar pedidos (autenticado)"
  orders(
    first: Int
    after: String
    filter: OrderFilterInput
  ): OrderConnection!

  "Buscar pedido por ID"
  order(id: ID!): Order

  "Listar categorias raiz"
  categories: [Category!]!

  "Buscar categoria por slug"
  category(slug: String!): Category
}

input ProductFilterInput {
  category: String
  status: ProductStatus
  minPrice: Int
  maxPrice: Int
  sellerId: ID
  query: String
}

input ProductSortInput {
  field: ProductSortField!
  direction: SortDirection!
}

enum ProductSortField {
  NAME
  PRICE
  CREATED_AT
  RATING
}

enum SortDirection {
  ASC
  DESC
}
```

**Exemplos de queries que devem funcionar:**

```graphql
# Query simples — client pede exatamente o que precisa
query GetProduct($id: ID!) {
  product(id: $id) {
    id
    name
    pricing {
      amount
      formatted
    }
    seller {
      name
    }
  }
}

# Query com fragment reutilizável
fragment ProductCard on Product {
  id
  name
  pricing {
    amount
    currency
    formatted
  }
  status
  category {
    name
  }
}

query ProductList {
  products(first: 20, filter: { status: ACTIVE }) {
    edges {
      node {
        ...ProductCard
        reviews(first: 3) {
          edges {
            node {
              rating
              comment
            }
          }
        }
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}

# Multiple queries in one request (batch)
query DashboardData($userId: ID!) {
  user(id: $userId) {
    name
    orders(first: 5) {
      edges {
        node {
          id
          status
          total
        }
      }
    }
  }
  categories {
    name
    slug
  }
}
```

**Resolvers:**
- Cada field no schema tem um resolver correspondente
- Resolvers de relacionamento (Product → Category, Order → User) carregam sob demanda
- `node` query resolve qualquer tipo pelo ID global (pattern Relay)

### Critérios de Aceite

- [ ] Todas as queries do schema funcionam com field selection
- [ ] Client recebe exatamente os campos solicitados (sem over-fetching)
- [ ] Fragments funcionam e são reutilizáveis
- [ ] Filtros `ProductFilterInput` aplicados corretamente
- [ ] Sort por múltiplos campos
- [ ] `node(id)` resolve qualquer tipo (Relay Global Object Identification)
- [ ] Variáveis de query funcionam ($id, $first, etc.)
- [ ] Testes para queries com diferentes seleções de campos

---

## Desafio 3.3 — Relay Connection Specification (Paginação)

**Contexto:** O Relay Connection specification é o padrão para paginação em GraphQL.
Implemente cursor-based pagination seguindo a spec.

### Requisitos

**Connection types:**

```graphql
type ProductConnection {
  edges: [ProductEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type ProductEdge {
  node: Product!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

**Argumentos de paginação:**
- `first` + `after` → forward pagination
- `last` + `before` → backward pagination
- `first` e `last` são mutuamente exclusivos

**Implementação:**

```graphql
# Primeira página
query {
  products(first: 10) {
    edges {
      node { id name }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}

# Próxima página
query {
  products(first: 10, after: "cursor_from_previous") {
    edges {
      node { id name }
      cursor
    }
    pageInfo {
      hasNextPage
      hasPreviousPage
      endCursor
      startCursor
    }
  }
}
```

### Java 25

```java
record Connection<T>(List<Edge<T>> edges, PageInfo pageInfo, int totalCount) {}
record Edge<T>(T node, String cursor) {}
record PageInfo(boolean hasNextPage, boolean hasPreviousPage, String startCursor, String endCursor) {}

<T> Connection<T> paginate(List<T> items, String after, String before, Integer first, Integer last,
                            Function<T, String> cursorExtractor) {
    // 1. Filter by cursor position (after/before)
    // 2. Slice by first/last
    // 3. Build edges with cursors
    // 4. Compute pageInfo
    var startIndex = after != null ? findIndexAfterCursor(items, after, cursorExtractor) : 0;
    var sliced = items.subList(startIndex, items.size());

    if (first != null) sliced = sliced.subList(0, Math.min(first, sliced.size()));

    var edges = sliced.stream()
        .map(item -> new Edge<>(item, encodeCursor(cursorExtractor.apply(item))))
        .toList();

    var pageInfo = new PageInfo(
        startIndex + sliced.size() < items.size(), // hasNextPage
        startIndex > 0,                             // hasPreviousPage
        edges.isEmpty() ? null : edges.getFirst().cursor(),
        edges.isEmpty() ? null : edges.getLast().cursor()
    );

    return new Connection<>(edges, pageInfo, items.size());
}
```

### Go 1.26

```go
type Connection[T any] struct {
    Edges      []Edge[T] `json:"edges"`
    PageInfo   PageInfo  `json:"pageInfo"`
    TotalCount int       `json:"totalCount"`
}

type Edge[T any] struct {
    Node   T      `json:"node"`
    Cursor string `json:"cursor"`
}

type PageInfo struct {
    HasNextPage     bool    `json:"hasNextPage"`
    HasPreviousPage bool    `json:"hasPreviousPage"`
    StartCursor     *string `json:"startCursor"`
    EndCursor       *string `json:"endCursor"`
}

func Paginate[T any](items []T, after, before *string, first, last *int,
    cursorFn func(T) string) Connection[T] {
    // Implementation follows Relay spec
    startIdx := 0
    if after != nil {
        startIdx = findIndexAfterCursor(items, *after, cursorFn)
    }
    sliced := items[startIdx:]
    if first != nil && *first < len(sliced) {
        sliced = sliced[:*first]
    }

    edges := make([]Edge[T], len(sliced))
    for i, item := range sliced {
        cursor := encodeCursor(cursorFn(item))
        edges[i] = Edge[T]{Node: item, Cursor: cursor}
    }

    return Connection[T]{
        Edges:      edges,
        PageInfo:   PageInfo{ /* ... */ },
        TotalCount: len(items),
    }
}
```

### Critérios de Aceite

- [ ] Connection type com `edges`, `pageInfo`, `totalCount`
- [ ] Edge type com `node` e `cursor`
- [ ] Forward pagination com `first` + `after`
- [ ] Backward pagination com `last` + `before`
- [ ] `first` e `last` mutuamente exclusivos
- [ ] `hasNextPage` e `hasPreviousPage` corretos
- [ ] Cursors opacos (base64 encoded)
- [ ] Funciona combinado com filtros e sort
- [ ] Testes: primeira página, página intermediária, última página, lista vazia

---

## Desafio 3.4 — Mutations (Input/Payload Pattern)

**Contexto:** Mutations são operações de escrita. Siga o padrão Input/Payload
com `clientMutationId` para rastreabilidade.

### Requisitos

**Mutation type:**

```graphql
type Mutation {
  createProduct(input: CreateProductInput!): CreateProductPayload!
  updateProduct(input: UpdateProductInput!): UpdateProductPayload!
  deleteProduct(input: DeleteProductInput!): DeleteProductPayload!

  createOrder(input: CreateOrderInput!): CreateOrderPayload!
  cancelOrder(input: CancelOrderInput!): CancelOrderPayload!

  createReview(input: CreateReviewInput!): CreateReviewPayload!

  register(input: RegisterInput!): RegisterPayload!
  login(input: LoginInput!): LoginPayload!
}

input CreateProductInput {
  clientMutationId: String
  name: String!
  description: String
  price: Int!
  currency: Currency!
  categoryId: ID!
  stock: Int!
  tags: [String!]
}

type CreateProductPayload {
  clientMutationId: String
  product: Product
  errors: [UserError!]
}

"""
Erro de domínio retornado no payload (não como GraphQL error).
Permite que o client trate erros de negócio de forma tipada.
"""
type UserError {
  field: String
  message: String!
  code: ErrorCode!
}

enum ErrorCode {
  VALIDATION_ERROR
  NOT_FOUND
  CONFLICT
  UNAUTHORIZED
  INSUFFICIENT_STOCK
  INVALID_STATUS_TRANSITION
}
```

**Exemplos de mutations:**

```graphql
mutation CreateProduct($input: CreateProductInput!) {
  createProduct(input: $input) {
    clientMutationId
    product {
      id
      name
      pricing { formatted }
    }
    errors {
      field
      message
      code
    }
  }
}

# Variables:
{
  "input": {
    "clientMutationId": "create-product-1",
    "name": "Mechanical Keyboard",
    "price": 15999,
    "currency": "BRL",
    "categoryId": "cat-electronics",
    "stock": 42,
    "tags": ["gaming", "rgb"]
  }
}
```

**Regras:**
- Toda mutation recebe um único argumento `input` do tipo `*Input`
- Toda mutation retorna um `*Payload` com o resultado + `errors`
- `clientMutationId` propagado de input para payload (rastreabilidade)
- Erros de negócio retornados no `errors` do payload (não como GraphQL errors)
- GraphQL errors reservados para erros de infraestrutura (auth, rate limit)
- Validação de input (campos obrigatórios, ranges, formatos)

### Critérios de Aceite

- [ ] Todas as mutations seguem padrão Input/Payload
- [ ] `clientMutationId` propagado corretamente
- [ ] Erros de negócio retornados em `payload.errors` (não GraphQL errors)
- [ ] `UserError` com `field`, `message`, `code`
- [ ] Validação retorna todos os erros (não fail-fast)
- [ ] Mutations de estado (cancelOrder) validam transições
- [ ] Testes: mutation sucesso, mutation com erro de validação, mutation com erro de negócio

---

## Desafio 3.5 — Subscriptions (Eventos Realtime)

**Contexto:** O Marketplace precisa notificar clientes em tempo real sobre
mudanças de status de pedidos e novos produtos.

### Requisitos

**Subscription type:**

```graphql
type Subscription {
  "Notifica quando o status de um pedido muda"
  orderStatusChanged(orderId: ID!): OrderStatusEvent!

  "Notifica quando um novo produto é adicionado a uma categoria"
  productAdded(categoryId: ID): Product!

  "Notifica sobre novos reviews de um produto"
  reviewAdded(productId: ID!): Review!
}

type OrderStatusEvent {
  order: Order!
  previousStatus: OrderStatus!
  newStatus: OrderStatus!
  changedAt: DateTime!
}
```

**Implementação com WebSocket (`graphql-ws` protocol):**

```
1. Client conecta via WebSocket: ws://localhost:8080/graphql
2. Client envia: { "type": "connection_init", "payload": { "Authorization": "Bearer ..." } }
3. Server responde: { "type": "connection_ack" }
4. Client subscribe: { "id": "1", "type": "subscribe", "payload": { "query": "subscription { orderStatusChanged(orderId: \"...\") { ... } }" } }
5. Server envia eventos: { "id": "1", "type": "next", "payload": { "data": { ... } } }
6. Client ou Server encerra: { "id": "1", "type": "complete" }
```

**Event bus em memória:**

```java
// Java — pub/sub simples com Virtual Threads
class EventBus<T> {
    private final Map<String, List<Consumer<T>>> subscribers = new ConcurrentHashMap<>();

    void subscribe(String topic, Consumer<T> listener) {
        subscribers.computeIfAbsent(topic, k -> new CopyOnWriteArrayList<>()).add(listener);
    }

    void publish(String topic, T event) {
        subscribers.getOrDefault(topic, List.of()).forEach(listener ->
            Thread.ofVirtual().start(() -> listener.accept(event))
        );
    }

    void unsubscribe(String topic, Consumer<T> listener) {
        subscribers.getOrDefault(topic, List.of()).remove(listener);
    }
}
```

```go
// Go — pub/sub com channels
type EventBus[T any] struct {
    mu          sync.RWMutex
    subscribers map[string][]chan T
}

func (eb *EventBus[T]) Subscribe(topic string) <-chan T {
    ch := make(chan T, 10)
    eb.mu.Lock()
    eb.subscribers[topic] = append(eb.subscribers[topic], ch)
    eb.mu.Unlock()
    return ch
}

func (eb *EventBus[T]) Publish(topic string, event T) {
    eb.mu.RLock()
    defer eb.mu.RUnlock()
    for _, ch := range eb.subscribers[topic] {
        select {
        case ch <- event:
        default: // drop if subscriber is slow
        }
    }
}
```

### Critérios de Aceite

- [ ] WebSocket endpoint funcional em `/graphql` (ws://)
- [ ] Protocol `graphql-ws` implementado (connection_init, subscribe, next, complete)
- [ ] `orderStatusChanged` envia evento quando mutation muda status
- [ ] `productAdded` filtra por categoria (se fornecida)
- [ ] `reviewAdded` filtra por productId
- [ ] Autenticação na connection_init (token JWT)
- [ ] Unsubscribe limpa recursos
- [ ] Event bus thread-safe (Java: ConcurrentHashMap, Go: sync.RWMutex + channels)
- [ ] Testes: subscribe → trigger mutation → receive event → unsubscribe

---

## Desafio 3.6 — Error Handling (Union Types)

**Contexto:** GraphQL oferece múltiplas estratégias de error handling.
Implemente a abordagem mais type-safe: Result unions.

### Requisitos

**3 camadas de erros:**

```
Camada 1: GraphQL Errors (infraestrutura)
├── Syntax errors, validation errors, auth errors
├── Retornados no array "errors" da response
└── Client não tem type safety

Camada 2: Payload Errors (domínio — UserError)
├── Erros de negócio no campo "errors" do payload
├── Tipados com field + code + message
└── ✅ Recomendado para maioria dos casos

Camada 3: Result Unions (máximo type safety)
├── Union type que pode ser Success ou Error
├── Client faz pattern matching no __typename
└── ✅ Recomendado para operações críticas
```

**Implementar Result Unions para operações críticas:**

```graphql
union CreateOrderResult = CreateOrderSuccess | InsufficientStockError | InvalidAddressError | UnauthorizedError

type CreateOrderSuccess {
  order: Order!
}

type InsufficientStockError {
  message: String!
  productId: ID!
  requested: Int!
  available: Int!
}

type InvalidAddressError {
  message: String!
  field: String!
}

type UnauthorizedError {
  message: String!
}

type Mutation {
  createOrder(input: CreateOrderInput!): CreateOrderResult!
}
```

**Query com union type:**

```graphql
mutation CreateOrder($input: CreateOrderInput!) {
  createOrder(input: $input) {
    ... on CreateOrderSuccess {
      order {
        id
        status
        total
      }
    }
    ... on InsufficientStockError {
      message
      productId
      requested
      available
    }
    ... on InvalidAddressError {
      message
      field
    }
    ... on UnauthorizedError {
      message
    }
  }
}
```

**Regras:**
- Usar Result Unions para: createOrder, processPayment (operações críticas)
- Usar Payload Errors (UserError) para: createProduct, updateProduct (CRUD padrão)
- GraphQL errors apenas para: autenticação, rate limit, erros de infraestrutura
- Nunca misturar abordagens no mesmo resolver

### Critérios de Aceite

- [ ] Result Unions implementados para createOrder
- [ ] Client pode fazer `... on TypeName` para pattern matching
- [ ] Cada error type tem campos específicos (não genérico)
- [ ] Payload errors (`UserError`) para CRUD padrão
- [ ] GraphQL errors apenas para infraestrutura
- [ ] `__typename` disponível para discriminação
- [ ] Testes para cada branch da union (sucesso + cada tipo de erro)

---

## Desafio 3.7 — Input Validation & Sanitization

**Contexto:** Validar inputs em GraphQL é diferente de REST — o schema já garante tipos,
mas regras de negócio precisam de validação adicional.

### Requisitos

**Validação em camadas:**

```
GraphQL Schema (automático)
├── Tipo correto (String, Int, etc.)
├── Non-null (!) vs nullable
├── Enum values válidos
└── ✅ Engine faz automaticamente

Validação Custom (manual nos resolvers)
├── String: min/max length, regex, sanitize HTML
├── Int: ranges (price >= 0, quantity >= 1)
├── Business: seller exists, category exists, stock available
└── ⚠️ Você implementa
```

**Implementar custom scalar types com validação:**

```graphql
"Email validado por regex"
scalar Email

"Slug: lowercase, hífens, 3-50 chars"
scalar Slug

"Preço em centavos: >= 0"
scalar PositiveInt

"String sanitizada: sem HTML tags, 1-1000 chars"
scalar SafeString
```

**Validação de input completa:**

```graphql
input CreateProductInput {
  name: SafeString!          # 3-200 chars, sem HTML
  description: SafeString    # max 2000 chars
  price: PositiveInt!        # >= 0
  currency: Currency!        # enum validation automática
  categoryId: ID!            # deve existir
  stock: PositiveInt!        # >= 0
  tags: [SafeString!]        # max 10 items, cada 2-50 chars
}
```

**Response de validação (múltiplos erros):**

```graphql
mutation {
  createProduct(input: {
    name: ""
    price: -100
    currency: BRL
    categoryId: "nonexistent"
    stock: 0
    tags: ["a", "valid-tag", "this-tag-is-way-too-long-and-should-be-rejected-by-validation"]
  }) {
    product { id }
    errors {
      field
      message
      code
    }
  }
}

# Response:
{
  "data": {
    "createProduct": {
      "product": null,
      "errors": [
        {"field": "name", "message": "must be between 3 and 200 characters", "code": "VALIDATION_ERROR"},
        {"field": "price", "message": "must be greater than or equal to 0", "code": "VALIDATION_ERROR"},
        {"field": "categoryId", "message": "category not found", "code": "NOT_FOUND"},
        {"field": "tags[2]", "message": "must be between 2 and 50 characters", "code": "VALIDATION_ERROR"}
      ]
    }
  }
}
```

### Critérios de Aceite

- [ ] Custom scalars com validação: `Email`, `Slug`, `PositiveInt`, `SafeString`
- [ ] Scalar inválido rejeitado na parsing (antes do resolver)
- [ ] Validação de negócio no resolver retorna todos os erros
- [ ] HTML tags sanitizados em `SafeString`
- [ ] Array limits validados (max 10 tags)
- [ ] Erros indexados para arrays (`tags[2]`)
- [ ] Testes para cada custom scalar com valores válidos e inválidos
- [ ] Testes para validação de negócio (referência inexistente)

---

## Desafio 3.8 — GraphQL Server Completo

**Contexto:** Integre todos os conceitos em um servidor GraphQL funcional
que serve o Marketplace completo.

### Requisitos

**Servidor GraphQL com:**
1. Endpoint `POST /graphql` que aceita queries, mutations e variáveis
2. Endpoint `GET /graphql` com playground/documentation (GraphiQL ou similar)
3. Todos os resolvers implementados (queries + mutations de products, orders, users, reviews)
4. Subscriptions via WebSocket no mesmo path
5. Introspection habilitado (para tooling)

**Request format:**

```json
POST /graphql
Content-Type: application/json

{
  "query": "query GetProduct($id: ID!) { product(id: $id) { id name } }",
  "variables": {"id": "01J5A3K..."},
  "operationName": "GetProduct"
}
```

**Response format (sucesso):**

```json
{
  "data": {
    "product": {
      "id": "01J5A3K...",
      "name": "Mechanical Keyboard"
    }
  }
}
```

**Response format (erro):**

```json
{
  "data": null,
  "errors": [
    {
      "message": "Product not found",
      "locations": [{"line": 1, "column": 37}],
      "path": ["product"],
      "extensions": {
        "code": "NOT_FOUND",
        "timestamp": "2025-01-15T10:30:00Z"
      }
    }
  ]
}
```

**Integrar com store em memória:**
- Seed data: 10 products, 3 users, 5 orders, 15 reviews, 5 categories
- Mutations alteram o store em memória
- Subscriptions notificam sobre mudanças
- Paginação funcional em todas as connections

### Critérios de Aceite

- [ ] `POST /graphql` aceita e executa queries, mutations
- [ ] Variáveis de query processadas corretamente
- [ ] Introspection funcional (`{ __schema { types { name } } }`)
- [ ] Seed data carregado no startup
- [ ] Mutations alteram o estado consistentemente
- [ ] Subscriptions recebem eventos após mutations
- [ ] Formato de response segue GraphQL spec (data + errors)
- [ ] `extensions` nos errors com code e timestamp
- [ ] Testes end-to-end para fluxo completo (query → mutation → subscription)

---

## Anti-Patterns a Evitar

| Anti-Pattern | Correto |
|-------------|---------|
| REST-style: um endpoint por recurso | Single endpoint `POST /graphql` |
| Versionar URL (`/graphql/v2`) | Schema evolution sem versão de URL |
| Erros de negócio como GraphQL errors | Usar Payload errors ou Result Unions |
| Mutations sem Input/Payload pattern | Sempre usar `*Input` → `*Payload` |
| Offset-based pagination | Relay Connection specification |
| Campos snake_case no schema | camelCase para fields |
| Enums em camelCase | SCREAMING_SNAKE_CASE para enums |

---

## Próximo Nível

→ [Level 4 — GraphQL Advanced: Performance & Escala](04-graphql-advanced.md)
