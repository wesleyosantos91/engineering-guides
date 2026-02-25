# GraphQL — Best Practices

> Guia completo de boas práticas para design, implementação e manutenção de APIs GraphQL, alinhado com padrões de mercado adotados por empresas como GitHub, Shopify, Airbnb e Facebook/Meta.

---

## Sumário

1. [Design de Schema](#1-design-de-schema)
2. [Naming Conventions](#2-naming-conventions)
3. [Queries e Fragments](#3-queries-e-fragments)
4. [Mutations](#4-mutations)
5. [Subscriptions](#5-subscriptions)
6. [Paginação (Connections Pattern)](#6-paginação-connections-pattern)
7. [Tratamento de Erros](#7-tratamento-de-erros)
8. [Segurança](#8-segurança)
9. [Performance e Otimização](#9-performance-e-otimização)
10. [Versionamento e Evolução do Schema](#10-versionamento-e-evolução-do-schema)
11. [Caching](#11-caching)
12. [Documentação e Tooling](#12-documentação-e-tooling)
13. [Testing](#13-testing)
14. [Observabilidade](#14-observabilidade)
15. [Padrões de Arquitetura](#15-padrões-de-arquitetura)
16. [File Uploads](#16-file-uploads)
17. [Query Batching](#17-query-batching)

---

## 1. Design de Schema

### Princípios Fundamentais

- **Schema-first design**: defina o schema antes da implementação.
- **Pense no grafo**: modele relacionamentos como um grafo de entidades.
- **Client-driven**: o schema deve atender às necessidades dos consumidores.
- **Um grafo unificado**: evite múltiplos schemas — use Federation para compor.

### Tipos Básicos

```graphql
# Tipos escalares customizados
scalar DateTime
scalar Email
scalar URL
scalar UUID
scalar Money

# Enums com descrições
"""Status do pedido no ciclo de vida"""
enum OrderStatus {
  """Pedido aguardando pagamento"""
  PENDING
  """Pagamento confirmado"""
  CONFIRMED
  """Pedido em preparação"""
  PROCESSING
  """Pedido enviado"""
  SHIPPED
  """Pedido entregue"""
  DELIVERED
  """Pedido cancelado"""
  CANCELLED
}

# Interface para herança
interface Node {
  """Identificador global único"""
  id: ID!
}

interface Timestamped {
  createdAt: DateTime!
  updatedAt: DateTime!
}

# Tipo principal
"""Representa um usuário do sistema"""
type User implements Node & Timestamped {
  id: ID!
  name: String!
  email: Email!
  avatar: URL
  status: UserStatus!
  orders(first: Int, after: String): OrderConnection!
  createdAt: DateTime!
  updatedAt: DateTime!
}
```

### Regras de Design

| Regra | Detalhes |
|-------|---------|
| Campos não-nulos (`!`) | Use com critério — apenas quando o campo é garantidamente presente |
| Listas | `[Type!]!` = lista não-nula de itens não-nulos |
| IDs globais | Use `ID!` com encoding (base64 de `Type:id`) |
| Descrições | Documente todos os tipos e campos com `"""docstring"""` |
| Deprecation | Use `@deprecated(reason: "Use newField")` |

### Anti-patterns a Evitar

```graphql
# ❌ Resolver tudo em um tipo gigante
type Query {
  everything: JSON  # Evite tipos genéricos
}

# ❌ Duplicar dados entre tipos
type User {
  orderCount: Int  # Desnecessário se User.orders já existe
}

# ❌ Expor detalhes de implementação (IDs do banco)
type Product {
  databaseId: Int  # Exponha apenas IDs opacos
}

# ✅ Preferir tipos bem definidos
type Query {
  user(id: ID!): User
  users(filter: UserFilter, first: Int, after: String): UserConnection!
}
```

---

## 2. Naming Conventions

### Regras de Nomenclatura

| Elemento | Convenção | Exemplo |
|----------|-----------|---------|
| Tipos | PascalCase | `User`, `OrderItem` |
| Campos | camelCase | `firstName`, `totalAmount` |
| Enums | SCREAMING_SNAKE_CASE | `PAYMENT_PENDING` |
| Input types | PascalCase + sufixo `Input` | `CreateUserInput` |
| Payloads de mutação | PascalCase + sufixo `Payload` | `CreateUserPayload` |
| Connections | PascalCase + sufixo `Connection` | `UserConnection` |
| Edges | PascalCase + sufixo `Edge` | `UserEdge` |
| Mutations | camelCase verboso | `createUser`, `updateOrderStatus` |
| Queries | camelCase | `user`, `users`, `orderById` |
| Subscriptions | camelCase com `on` prefix | `onOrderStatusChanged` |
| Arguments | camelCase | `firstName`, `orderBy` |

### Exemplos

```graphql
# ✅ Naming correto
type Query {
  user(id: ID!): User
  users(filter: UserFilter, first: Int, after: String): UserConnection!
  searchProducts(query: String!, first: Int): ProductConnection!
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(input: UpdateUserInput!): UpdateUserPayload!
  deleteUser(id: ID!): DeleteUserPayload!
}

type Subscription {
  onOrderStatusChanged(orderId: ID!): OrderStatusEvent!
}
```

---

## 3. Queries e Fragments

### Queries Bem Estruturadas

```graphql
# Query nomeada com variáveis tipadas
query GetUserWithOrders($userId: ID!, $first: Int = 10) {
  user(id: $userId) {
    ...UserBasicInfo
    orders(first: $first) {
      edges {
        node {
          ...OrderSummary
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}

# Fragments reutilizáveis
fragment UserBasicInfo on User {
  id
  name
  email
  avatar
  status
}

fragment OrderSummary on Order {
  id
  status
  totalAmount
  createdAt
  items {
    ...OrderItemInfo
  }
}

fragment OrderItemInfo on OrderItem {
  id
  product {
    id
    name
  }
  quantity
  unitPrice
}
```

### Boas Práticas de Queries

- **Sempre nomeie suas queries** — facilita debugging e analytics.
- **Use variáveis** — nunca interpole valores diretamente.
- **Fragments para reutilização** — DRY nos campos solicitados.
- **Peça apenas o que precisa** — evite over-fetching.
- **Aliases** para queries múltiplas ao mesmo campo.

```graphql
# Aliases
query CompareUsers {
  admin: user(id: "1") {
    ...UserBasicInfo
  }
  customer: user(id: "2") {
    ...UserBasicInfo
  }
}
```

---

## 4. Mutations

### Padrão de Input/Payload

```graphql
# Input type dedicado
input CreateUserInput {
  name: String!
  email: Email!
  password: String!
  role: UserRole = CUSTOMER
}

# Payload com resultado + erros
type CreateUserPayload {
  """O usuário criado, null se houver erro"""
  user: User
  """Lista de erros de validação, se houver"""
  errors: [UserError!]!
}

type UserError {
  """Caminho do campo com erro"""
  field: [String!]
  """Mensagem legível para o usuário"""
  message: String!
  """Código de erro para automação"""
  code: ErrorCode!
}

enum ErrorCode {
  INVALID_INPUT
  NOT_FOUND
  ALREADY_EXISTS
  UNAUTHORIZED
  FORBIDDEN
  INTERNAL_ERROR
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(input: UpdateUserInput!): UpdateUserPayload!
  deleteUser(id: ID!): DeleteUserPayload!
}
```

### Boas Práticas de Mutations

| Prática | Detalhes |
|---------|---------|
| **Um Input type por mutation** | Não reutilize inputs entre mutations diferentes |
| **Payload com campo de erros** | Retorne erros no payload, não apenas via extensions |
| **Retorne o objeto mutado** | Permite ao client atualizar o cache |
| **Mutations atômicas** | Uma mutation = uma operação de negócio |
| **Idempotência** | Use `clientMutationId` para deduplicação |

```graphql
# Mutation com clientMutationId
input CreatePaymentInput {
  clientMutationId: String
  orderId: ID!
  amount: Money!
  method: PaymentMethod!
}

type CreatePaymentPayload {
  clientMutationId: String
  payment: Payment
  errors: [UserError!]!
}
```

---

## 5. Subscriptions

### Implementação

```graphql
type Subscription {
  """Emitido quando o status de um pedido muda"""
  onOrderStatusChanged(orderId: ID!): OrderStatusEvent!

  """Emitido quando uma nova mensagem é enviada em um chat"""
  onMessageSent(chatId: ID!): Message!

  """Emitido quando o estoque de um produto muda"""
  onInventoryChanged(productId: ID): InventoryEvent!
}

type OrderStatusEvent {
  order: Order!
  previousStatus: OrderStatus!
  newStatus: OrderStatus!
  changedAt: DateTime!
  changedBy: User
}
```

### Boas Práticas

- **Use WebSocket** (protocolo `graphql-ws`) como transporte.
- **Filtre eventos no server** — não envie tudo para o client filtrar.
- **Autentique na conexão** — valide token ao estabelecer WebSocket.
- **Heartbeat/Keep-alive** — implemente ping/pong para detectar conexões mortas.
- **Reconexão automática** no client com backoff exponencial.
- **Limite subscriptions por conexão** — evite abuso de recursos.
- Prefira **subscriptions para eventos reais** (não polling disfarçado).

---

## 6. Paginação (Connections Pattern)

### Relay Connection Specification

```graphql
# Schema
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type Query {
  users(
    first: Int
    after: String
    last: Int
    before: String
    filter: UserFilter
    orderBy: UserOrderBy
  ): UserConnection!
}

input UserFilter {
  status: UserStatus
  search: String
  createdAfter: DateTime
}

input UserOrderBy {
  field: UserSortField!
  direction: SortDirection!
}

enum UserSortField {
  NAME
  CREATED_AT
  EMAIL
}

enum SortDirection {
  ASC
  DESC
}
```

### Query de Paginação

```graphql
# Primeira página
query ListUsers {
  users(first: 20, filter: { status: ACTIVE }) {
    edges {
      node {
        id
        name
        email
      }
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
query ListUsersNextPage($cursor: String!) {
  users(first: 20, after: $cursor) {
    edges {
      node {
        id
        name
        email
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

### Por que Connection Pattern?

| Vantagem | Descrição |
|----------|-----------|
| Cursor opaco | Client não sabe detalhes de paginação |
| Metadados na edge | Dados do relacionamento (ex: role em group) |
| Bidirecional | Suporta `first/after` e `last/before` |
| Estável | Não sofre com inserções durante paginação |

---

## 7. Tratamento de Erros

### Estratégias de Erro

#### 1. Erros no Payload (Recomendado para erros esperados)

```json
{
  "data": {
    "createUser": {
      "user": null,
      "errors": [
        {
          "field": ["input", "email"],
          "message": "Email already registered",
          "code": "ALREADY_EXISTS"
        }
      ]
    }
  }
}
```

#### 2. Erros GraphQL (Para erros inesperados/infraestrutura)

```json
{
  "data": null,
  "errors": [
    {
      "message": "Not authenticated",
      "locations": [{ "line": 2, "column": 3 }],
      "path": ["user"],
      "extensions": {
        "code": "UNAUTHENTICATED",
        "timestamp": "2026-02-23T10:30:00Z",
        "traceId": "abc-123"
      }
    }
  ]
}
```

### Union Types para Resultados (Padrão avançado)

```graphql
union CreateUserResult = CreateUserSuccess | ValidationError | NotAuthorizedError

type CreateUserSuccess {
  user: User!
}

type ValidationError {
  errors: [FieldError!]!
}

type NotAuthorizedError {
  message: String!
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserResult!
}
```

```graphql
# Query com union
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    ... on CreateUserSuccess {
      user {
        id
        name
      }
    }
    ... on ValidationError {
      errors {
        field
        message
      }
    }
    ... on NotAuthorizedError {
      message
    }
  }
}
```

### Hierarquia de Erros (Recomendação)

| Tipo de Erro | Onde Tratar | Exemplo |
|-------------|------------|---------|
| Validação de input | Payload `errors` | Campo obrigatório |
| Regra de negócio | Payload `errors` ou Union | Saldo insuficiente |
| Autenticação | GraphQL error extensions | Token expirado |
| Autorização | GraphQL error extensions | Sem permissão |
| Infraestrutura | GraphQL error extensions | Timeout de banco |

---

## 8. Segurança

### Query Depth Limiting

```javascript
// Limite profundidade de queries para evitar ataques
// Recomendado: máximo 10-15 níveis de profundidade
const depthLimit = require('graphql-depth-limit');

const server = new ApolloServer({
  validationRules: [depthLimit(10)]
});
```

### Query Complexity Analysis

```graphql
# Cada campo tem um custo. Limite o custo total da query.
type Query {
  users(first: Int): UserConnection! @cost(complexity: 5, multipliers: ["first"])
  user(id: ID!): User @cost(complexity: 1)
}
```

```javascript
// Defina um custo máximo por query
const maxComplexity = 1000;

// Exemplo de cálculo:
// users(first: 50) → 5 * 50 = 250
// users(first: 50) { orders(first: 10) } → 250 + (3 * 10 * 50) = 1750 ❌
```

### Checklist de Segurança GraphQL

- [ ] **Desabilitar introspection em produção** (ou limitar acesso).
- [ ] **Query depth limiting** — máximo 10-15 níveis.
- [ ] **Query complexity limiting** — custo máximo por request.
- [ ] **Rate limiting** — por IP, por usuário, por query.
- [ ] **Timeout por query** — máximo 30s para execução.
- [ ] **Persistent queries** — aceitar apenas queries pré-registradas.
- [ ] **Validação de input** — tamanho máximo de strings, limites numéricos.
- [ ] **Autenticação** — validar token antes de resolver.
- [ ] **Autorização granular** — field-level authorization.
- [ ] **Mascarar erros internos** — não expor stack traces/detalhes do banco.
- [ ] **CORS configurado** — origens permitidas explicitamente.
- [ ] **Limitar tamanho do body** — máximo 1MB para queries.

### Field-Level Authorization

```graphql
type User {
  id: ID!
  name: String!
  email: String! @auth(requires: OWNER)        # Apenas o próprio usuário
  phone: String @auth(requires: ADMIN)          # Apenas admins
  orders: OrderConnection! @auth(requires: AUTHENTICATED)
  internalNotes: String @auth(requires: ADMIN)  # Apenas admins
}
```

---

## 9. Performance e Otimização

### DataLoader Pattern (N+1 Problem)

```javascript
// ❌ SEM DataLoader — N+1 queries
// Para 50 users com orders: 1 query users + 50 queries orders = 51 queries

// ✅ COM DataLoader — Batch + Cache
const userLoader = new DataLoader(async (userIds) => {
  const users = await db.users.findByIds(userIds);
  return userIds.map(id => users.find(u => u.id === id));
});

// Para 50 users com orders: 1 query users + 1 query orders (batched) = 2 queries
```

### Persisted Queries

```javascript
// Client envia hash da query em vez do texto completo
// Reduz tamanho do request e previne queries arbitrárias

// Request com persisted query
POST /graphql
{
  "extensions": {
    "persistedQuery": {
      "version": 1,
      "sha256Hash": "ecf4edb46db40b5132295c0291d62fb65d6759a9eedfa4d5d612dd5ec54a6b38"
    }
  },
  "variables": { "userId": "123" }
}
```

### Automatic Persisted Queries (APQ)

```
Client                            Server
  |                                  |
  |-- Hash only ------------------>  |
  |                                  |-- Cache miss
  |<---- PersistedQueryNotFound ---- |
  |                                  |
  |-- Hash + Full Query ----------> |
  |                                  |-- Store in cache
  |<---- Response ------------------ |
  |                                  |
  |-- Hash only (next time) ------> |
  |                                  |-- Cache hit!
  |<---- Response ------------------ |
```

### Response Caching

| Estratégia | Escopo | Uso |
|-----------|--------|-----|
| **Full response cache** | CDN/reverse proxy | Queries públicas |
| **Partial cache** | Application layer | Resolvers individuais |
| **DataLoader cache** | Per-request | Dentro de uma query |
| **Normalized cache** | Client-side | Apollo Client / Relay |

### Defer e Stream (Experimental)

```graphql
# @defer para campos lentos
query GetProduct($id: ID!) {
  product(id: $id) {
    name
    price
    ... @defer {
      reviews(first: 10) {
        edges {
          node {
            rating
            comment
          }
        }
      }
    }
  }
}

# @stream para listas incrementais
query GetFeed {
  feed(first: 50) @stream(initialCount: 10) {
    title
    content
  }
}
```

---

## 10. Versionamento e Evolução do Schema

### GraphQL NÃO usa versionamento por URL

> Em GraphQL, a evolução do schema substitui o versionamento. Adicione campos, deprecie gradualmente e remova apenas quando seguro.

### Estratégia de Evolução

```graphql
# 1. Adicionar novo campo (não breaking)
type User {
  name: String! @deprecated(reason: "Use firstName and lastName instead")
  firstName: String!
  lastName: String!
  fullName: String!  # Novo campo de conveniência
}

# 2. Deprecar campo antigo com mensagem clara
type Product {
  price: Float @deprecated(reason: "Use priceInCents (Int) for precision. Will be removed on 2026-12-01.")
  priceInCents: Int!
}
```

### Processo de Deprecation

```
1. Adicionar novo campo alternativo
2. Marcar campo antigo com @deprecated
3. Monitorar uso do campo deprecated (field-level analytics)
4. Notificar consumidores
5. Aguardar migração (mínimo 6 meses)
6. Remover campo quando uso = 0
```

### Schema Registry e Breaking Change Detection

```bash
# Usar ferramentas como GraphQL Inspector
graphql-inspector diff old-schema.graphql new-schema.graphql

# Resultado:
# ✅ Field 'User.fullName' was added
# ⚠️ Field 'User.name' was deprecated
# ❌ Field 'User.age' was removed (BREAKING CHANGE)
```

---

## 11. Caching

### Diretivas de Cache

```graphql
# Schema com hints de cache
type Query {
  user(id: ID!): User @cacheControl(maxAge: 300, scope: PRIVATE)
  products: [Product!]! @cacheControl(maxAge: 3600, scope: PUBLIC)
  currentUser: User @cacheControl(maxAge: 0, scope: PRIVATE)
}

type User @cacheControl(maxAge: 300) {
  id: ID!
  name: String!
  email: String! @cacheControl(maxAge: 0)  # Nunca cachear email
  orders: OrderConnection! @cacheControl(maxAge: 60)
}
```

### Cache por Tipo de Dado

| Tipo | maxAge | Scope | Exemplo |
|------|--------|-------|---------|
| Configurações | 86400 (24h) | PUBLIC | Feature flags |
| Catálogos | 3600 (1h) | PUBLIC | Produtos |
| Perfil do usuário | 300 (5min) | PRIVATE | Dados do user |
| Dados em tempo real | 0 | PRIVATE | Saldo, status |

### Normalized Cache no Client

```javascript
// Apollo Client com cache normalizado
const cache = new InMemoryCache({
  typePolicies: {
    User: {
      keyFields: ["id"],
      fields: {
        orders: {
          // Merge para paginação
          keyArgs: ["filter"],
          merge(existing, incoming, { args }) {
            if (!args?.after) return incoming;
            return {
              ...incoming,
              edges: [...(existing?.edges || []), ...incoming.edges]
            };
          }
        }
      }
    }
  }
});
```

---

## 12. Documentação e Tooling

### Schema Documentation

```graphql
"""
Serviço de gerenciamento de usuários.

## Autenticação
Todas as queries e mutations requerem um Bearer token válido
no header Authorization.

## Rate Limiting
- Queries: 1000 req/min
- Mutations: 100 req/min
"""
type Query {
  """
  Busca um usuário pelo ID.

  ### Permissões
  - `AUTHENTICATED`: campos públicos
  - `ADMIN`: todos os campos

  ### Exemplo
  ```graphql
  query {
    user(id: "abc-123") {
      name
      email
    }
  }
  ```
  """
  user(
    """O identificador único do usuário (UUID)"""
    id: ID!
  ): User
}
```

### Ferramentas Recomendadas

| Ferramenta | Uso |
|-----------|-----|
| **GraphQL Playground / Apollo Studio** | Explorar e testar o schema |
| **GraphQL Inspector** | Detectar breaking changes |
| **GraphQL Codegen** | Gerar tipos TypeScript/Java/etc |
| **Apollo Router / Gateway** | Federation e composition |
| **GraphQL Voyager** | Visualizar o grafo interativamente |
| **Spectaql** | Gerar documentação estática do schema |

### Schema Linting

```yaml
# .graphql-lint.yml
rules:
  defined-types-are-used: error
  deprecations-have-a-reason: error
  descriptions-are-capitalized: warn
  fields-have-descriptions: warn
  input-name: 
    severity: error
    style: Input-suffix
  naming-convention:
    types: PascalCase
    fields: camelCase
    enums: UPPER_CASE
  no-unreachable-types: error
  require-deprecation-reason: error
```

---

## 13. Testing

### Testes de Schema

```javascript
// Validar que o schema é válido e resolve corretamente
describe('User Query', () => {
  it('should return user by id', async () => {
    const query = `
      query GetUser($id: ID!) {
        user(id: $id) {
          id
          name
          email
        }
      }
    `;

    const result = await graphql({
      schema,
      source: query,
      variableValues: { id: '123' },
      contextValue: { user: authenticatedUser }
    });

    expect(result.errors).toBeUndefined();
    expect(result.data.user).toMatchObject({
      id: '123',
      name: 'João Silva',
      email: 'joao@example.com'
    });
  });

  it('should return UNAUTHENTICATED error without token', async () => {
    const result = await graphql({
      schema,
      source: '{ user(id: "123") { id } }',
      contextValue: { user: null }
    });

    expect(result.errors[0].extensions.code).toBe('UNAUTHENTICATED');
  });
});
```

### Tipos de Teste

| Tipo | O que Testa | Ferramenta |
|------|------------|-----------|
| **Unit** | Resolvers isolados | Jest, Vitest |
| **Integration** | Schema + resolvers + DB | Supertest |
| **Contract** | Schema não tem breaking changes | GraphQL Inspector |
| **E2E** | Fluxo completo client → server | Cypress, Playwright |
| **Performance** | Queries complexas, N+1 | k6, Artillery |

---

## 14. Observabilidade

### Logging Estruturado

```json
{
  "level": "info",
  "timestamp": "2026-02-23T10:30:00Z",
  "service": "user-service",
  "traceId": "abc-123",
  "graphql": {
    "operationName": "GetUserWithOrders",
    "operationType": "query",
    "complexity": 45,
    "depth": 4,
    "duration": 125,
    "fieldCount": 12,
    "errors": 0
  },
  "user": {
    "id": "user-456",
    "role": "CUSTOMER"
  }
}
```

### Métricas Essenciais

| Métrica | Descrição |
|---------|-----------|
| `graphql.request.duration` | Latência por operação (p50, p95, p99) |
| `graphql.request.count` | Total de requests por operação |
| `graphql.error.count` | Erros por tipo (validation, auth, internal) |
| `graphql.field.usage` | Frequência de uso por campo (para deprecation) |
| `graphql.complexity` | Complexidade das queries recebidas |
| `graphql.depth` | Profundidade das queries |
| `graphql.resolver.duration` | Latência por resolver |

### Tracing com OpenTelemetry

```javascript
// Cada resolver como um span no trace
const resolverSpan = tracer.startSpan('resolver.User.orders', {
  attributes: {
    'graphql.field.name': 'orders',
    'graphql.parent.type': 'User',
    'graphql.args.first': args.first
  }
});
```

---

## 15. Padrões de Arquitetura

### Schema Federation (Apollo Federation v2)

```graphql
# User Service
type User @key(fields: "id") {
  id: ID!
  name: String!
  email: String!
}

# Order Service
type User @key(fields: "id") {
  id: ID!
  orders(first: Int, after: String): OrderConnection!
}

type Order @key(fields: "id") {
  id: ID!
  user: User!
  status: OrderStatus!
  totalAmount: Money!
}
```

```
┌─────────────────────────────────────────┐
│           Apollo Router/Gateway          │
│         (Schema Composition)            │
└────────┬──────────┬──────────┬──────────┘
         │          │          │
    ┌────▼───┐ ┌───▼────┐ ┌──▼─────┐
    │ Users  │ │ Orders │ │Products│
    │Service │ │Service │ │Service │
    └────────┘ └────────┘ └────────┘
```

### Schema Stitching vs Federation

| Aspecto | Stitching | Federation |
|---------|-----------|-----------|
| Ownership | Centralizado | Distribuído |
| Acoplamento | Alto | Baixo |
| Escalabilidade | Limitada | Alta |
| Independência | Baixa | Alta |
| Recomendação | Legado | **Padrão atual** |

### BFF (Backend for Frontend) com GraphQL

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Web App │  │ Mobile   │  │ Smart TV │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │              │
┌────▼─────┐ ┌────▼─────┐  ┌────▼─────┐
│ Web BFF  │ │Mobile BFF│  │  TV BFF  │
│(GraphQL) │ │(GraphQL) │  │(GraphQL) │
└────┬─────┘ └────┬─────┘  └────┬─────┘
     │             │              │
     └──────┬──────┴──────┬───────┘
            │             │
     ┌──────▼──┐   ┌──────▼──┐
     │ Service │   │ Service │
     │    A    │   │    B    │
     └─────────┘   └─────────┘
```

---

## Referências

- [GraphQL Specification](https://spec.graphql.org/)
- [Relay Connection Specification](https://relay.dev/graphql/connections.htm)
- [Apollo Federation v2](https://www.apollographql.com/docs/federation/)
- [GitHub GraphQL API Guide](https://docs.github.com/en/graphql)
- [Shopify GraphQL Design Tutorial](https://github.com/Shopify/graphql-design-tutorial)
- [Principled GraphQL](https://principledgraphql.com/)
- [Production Ready GraphQL](https://book.productionreadygraphql.com/)
- [GraphQL Best Practices (graphql.org)](https://graphql.org/learn/best-practices/)
- [Apollo Server Best Practices](https://www.apollographql.com/docs/apollo-server/)
- [GraphQL Security Cheat Sheet (OWASP)](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html)

---

## 16. File Uploads

### Abordagens

| Abordagem | Descrição | Recomendado |
|-----------|-----------|:-----------:|
| **Signed URLs** | GraphQL retorna URL pré-assinada, client faz upload direto (S3, GCS) | ✅ Preferível |
| **Multipart Request** | Upload via `multipart/form-data` na mesma request GraphQL | ⚠️ Aceitável |
| **Base64 no payload** | Encode do arquivo como string Base64 | ❌ Evitar |

### Signed URL (Abordagem Recomendada)

```graphql
# 1. Solicitar URL de upload
mutation RequestUpload($input: RequestUploadInput!) {
  requestUpload(input: $input) {
    uploadUrl        # URL pré-assinada (ex: S3 presigned URL)
    fileId           # ID do arquivo para referência futura
    expiresAt        # Expiração da URL
  }
}

input RequestUploadInput {
  filename: String!
  contentType: String!
  sizeBytes: Int!
}

# 2. Client faz upload direto para o storage (fora do GraphQL)
# PUT {uploadUrl} com o arquivo

# 3. Confirmar upload
mutation ConfirmUpload($fileId: ID!) {
  confirmUpload(fileId: $fileId) {
    file {
      id
      url
      filename
      sizeBytes
    }
    errors {
      message
      code
    }
  }
}
```

### Multipart Upload (graphql-upload spec)

```javascript
// Client-side com Apollo Upload Client
import { createUploadLink } from 'apollo-upload-client';

const link = createUploadLink({ uri: '/graphql' });

// Mutation
const UPLOAD_FILE = gql`
  mutation UploadFile($file: Upload!) {
    uploadFile(file: $file) {
      id
      url
      filename
    }
  }
`;
```

### Por que Signed URLs são preferíveis?

- **Sem carga no servidor GraphQL** — upload vai direto para o storage
- **Suporte a arquivos grandes** — sem limite de payload do GraphQL
- **Resumable uploads** — possível com S3 multipart
- **CDN friendly** — arquivos já estão no storage correto

---

## 17. Query Batching

### HTTP Batching

```http
# Enviar múltiplas operações em um único request HTTP
POST /graphql HTTP/1.1
Content-Type: application/json

[
  {
    "query": "query GetUser($id: ID!) { user(id: $id) { name } }",
    "variables": { "id": "1" }
  },
  {
    "query": "query GetProducts { products(first: 5) { edges { node { name } } } }"
  }
]
```

### Boas Práticas de Batching

| Prática | Detalhes |
|---------|---------|
| **Limite de operações** | Máximo 10-20 operações por batch |
| **Timeout por operação** | Cada operação deve ter seu próprio timeout |
| **Erros independentes** | Falha em uma operação não deve afetar as outras |
| **Complexidade total** | Some a complexidade de todas as queries do batch |
| **Preferência** | Prefira compor queries GraphQL em vez de HTTP batching |

### Quando Usar Batching vs Query Composition

```graphql
# ✅ Preferível: compor em uma única query
query DashboardData {
  currentUser { name avatar }
  notifications(first: 5) { edges { node { message } } }
  recentOrders(first: 3) { edges { node { status totalAmount } } }
}

# ⚠️ Batching: apenas quando as queries são de contextos diferentes
# ou gerenciadas por componentes independentes
```
