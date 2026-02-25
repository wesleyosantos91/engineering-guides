# 25. REST vs GraphQL vs gRPC

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial para qualquer entrevista de System Design  
> **Complexidade:** Média

---

## Definição

**REST**, **GraphQL** e **gRPC** são os três paradigmas dominantes de comunicação entre serviços e clientes. Cada um foi projetado para resolver problemas específicos e tem trade-offs distintos:

- **REST** — O padrão da web; resource-oriented, HTTP nativo, amplamente adotado
- **GraphQL** — Client-driven queries; flexível, resolvendo over-fetching
- **gRPC** — Alta performance; binário (Protocol Buffers), bidirecional, ideal para service-to-service

---

## Por Que é Importante?

- **Decisão arquitetural fundamental** — afeta toda a stack de comunicação
- **Pergunta frequente em entrevistas** — "por que escolher X sobre Y?"
- **Big Techs criaram essas tecnologias** — Google (gRPC), Meta (GraphQL), Roy Fielding (REST)
- **Sistemas modernos usam múltiplas** — REST público, gRPC interno, GraphQL para BFF

---

## Comparativo Geral

| Aspecto | REST | GraphQL | gRPC |
|---------|------|---------|------|
| **Criador** | Roy Fielding (2000) | Meta/Facebook (2015) | Google (2016) |
| **Protocolo** | HTTP/1.1, HTTP/2 | HTTP/1.1, HTTP/2 | HTTP/2 (obrigatório) |
| **Formato** | JSON, XML | JSON | Protocol Buffers (binário) |
| **Schema** | OpenAPI/Swagger (opcional) | SDL (obrigatório) | .proto (obrigatório) |
| **Contrato** | Fraco (sem schema = sem type safety) | Forte (typed schema) | Forte (typed proto) |
| **Versionamento** | URI (/v1/) ou headers | Campo deprecation | Backward-compatible fields |
| **Streaming** | Não nativo | Subscriptions (WS) | Bidirecional nativo |
| **Caching** | HTTP Cache nativo (ETags, CDN) | Difícil (POST queries) | Não nativo |
| **Over-fetching** | Sim (endpoint retorna tudo) | Não (client seleciona campos) | Não (message definido) |
| **Under-fetching** | Sim (precisa de múltiplos endpoints) | Não (query nested) | Resolúvel com streaming |
| **Performance** | Boa | Boa (parsing overhead) | Excelente (binário + HTTP/2) |
| **Browser Support** | Nativo | Libs necessárias | Necessita grpc-web |
| **Curva de Aprendizado** | Baixa | Média | Alta |
| **Tooling** | Maduro (Postman, curl) | Playground, Apollo | Necessita tooling específico |

---

## REST (Representational State Transfer)

### Princípios

```
1. Client-Server:      Separação de responsabilidades
2. Stateless:          Cada request contém toda informação necessária
3. Cacheable:          Responses podem ser cacheadas (CDN, ETags)
4. Uniform Interface:  Recursos identificados por URIs
5. Layered System:     Client não sabe se está falando com server ou proxy
6. Code-on-Demand:     (Opcional) Server pode enviar código executável
```

### Design de APIs REST

```
Resources e verbos HTTP:

  GET    /users              → Lista users
  POST   /users              → Cria user
  GET    /users/{id}         → Retorna user específico
  PUT    /users/{id}         → Atualiza user (completo)
  PATCH  /users/{id}         → Atualiza user (parcial)
  DELETE /users/{id}         → Remove user
  
  Nested resources:
  GET    /users/{id}/orders  → Orders do user
  POST   /users/{id}/orders  → Cria order para user
  
  Filtering, sorting, pagination:
  GET    /users?role=admin&sort=name&page=1&limit=20
```

### Exemplo Completo

```http
-- Request --
GET /api/v1/users/123 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...

-- Response --
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "abc123"
Cache-Control: max-age=300

{
  "id": 123,
  "name": "Maria",
  "email": "maria@example.com",
  "role": "admin",
  "created_at": "2024-01-15T10:30:00Z",
  "orders_count": 42,
  "_links": {
    "self": "/api/v1/users/123",
    "orders": "/api/v1/users/123/orders"
  }
}
```

### Problemas Clássicos do REST

```
Over-fetching:
  GET /users/123 → retorna 20 campos quando eu só preciso de "name"
  
Under-fetching (N+1 problem):
  GET /users/123          → { name, order_ids: [1,2,3] }
  GET /orders/1           → { total: 50 }
  GET /orders/2           → { total: 30 }
  GET /orders/3           → { total: 20 }
  → 4 requests para compor uma view!

Versioning headache:
  /api/v1/users → schema antigo
  /api/v2/users → novo campo, removeu outro
  → Manter ambos = duplicação de código
```

### Quando Usar REST

```
✅ APIs públicas (amplamente entendido)
✅ CRUD simples com resources claros
✅ Caching agressivo via HTTP (CDN, browser)
✅ Ecossistema que usa HTTP nativo
✅ Time sem experiência com GraphQL/gRPC
```

---

## GraphQL

### Como Funciona

```
Client define EXATAMENTE o que quer; Server retorna EXATAMENTE isso:

Query:                          Response:
┌──────────────────────┐        ┌──────────────────────┐
│ query {               │        │ {                     │
│   user(id: 123) {    │   ──▶  │   "user": {          │
│     name             │        │     "name": "Maria", │
│     orders {         │        │     "orders": [      │
│       total          │        │       {"total": 50}, │
│       status         │        │       {"total": 30}  │
│     }                │        │     ]                │
│   }                  │        │   }                  │
│ }                    │        │ }                    │
└──────────────────────┘        └──────────────────────┘

  1 request, dados exatos, sem over/under-fetching!
```

### Schema Definition Language (SDL)

```graphql
# Schema (servidor define)
type User {
  id: ID!
  name: String!
  email: String!
  role: Role!
  orders: [Order!]!
  createdAt: DateTime!
}

type Order {
  id: ID!
  total: Float!
  status: OrderStatus!
  items: [OrderItem!]!
}

enum Role { USER ADMIN }
enum OrderStatus { PENDING PAID SHIPPED DELIVERED }

type Query {
  user(id: ID!): User
  users(role: Role, limit: Int = 20, offset: Int = 0): [User!]!
  order(id: ID!): Order
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}

type Subscription {
  orderStatusChanged(orderId: ID!): Order!
}

input CreateUserInput {
  name: String!
  email: String!
  role: Role = USER
}
```

### Operações

```graphql
# Query — leitura
query GetUserWithOrders {
  user(id: "123") {
    name
    email
    orders {
      total
      status
    }
  }
}

# Mutation — escrita  
mutation CreateUser {
  createUser(input: {
    name: "João"
    email: "joao@example.com"
  }) {
    id
    name
  }
}

# Subscription — real-time (via WebSocket)
subscription OnOrderUpdate {
  orderStatusChanged(orderId: "456") {
    status
    updatedAt
  }
}

# Fragments — reutilização
fragment UserBasic on User {
  id
  name
  email
}

query {
  currentUser { ...UserBasic }
  user(id: "456") { ...UserBasic role }
}
```

### Problemas do GraphQL

```
1. Security: query complexity attacks
   query Evil {
     users {
       orders {
         items {
           product {
             reviews {
               author {
                 orders {
                   items { ... }  ← nested infinito!
                 }
               }
             }
           }
         }
       }
     }
   }
   Solução: query depth limiting, complexity analysis, persisted queries

2. Caching é mais difícil:
   REST:    GET /users/123 → CDN cacheia facilmente (URL-based)
   GraphQL: POST /graphql  → body muda a cada query
   Solução: persisted queries (hash da query como cache key), Apollo Cache

3. N+1 em Resolvers:
   users → 100 users
   cada user.orders → 100 queries ao DB!
   Solução: DataLoader (batching + caching)

4. File uploads não nativo:
   Spec graphql-multipart-request-spec (não oficial)
```

### Quando Usar GraphQL

```
✅ Mobile apps (bandwidth limitada, evitar over-fetching)
✅ Frontend-driven development (equipe front define queries)
✅ BFF (Backend for Frontend) pattern
✅ Múltiplos clients com necessidades diferentes
✅ API com grafos de dados complexos
```

---

## gRPC (Google Remote Procedure Call)

### Como Funciona

```
Service definido em .proto → code generation → client chama como método local:

  ┌─────────────────┐                    ┌─────────────────┐
  │  Client (Java)  │                    │  Server (Go)    │
  │                 │                    │                 │
  │  stub.GetUser() │── HTTP/2 + Proto ─▶│  GetUser(req)   │
  │  → User object  │◀── binary frame ──│  → return User  │
  └─────────────────┘                    └─────────────────┘

  Proto → gera código em Java, Go, Python, C++, etc.
  Ambos os lados usam o MESMO contrato (.proto)
```

### Protocol Buffers (.proto)

```protobuf
// user.proto
syntax = "proto3";

package user.v1;

option java_package = "com.example.user.v1";
option go_package = "github.com/example/user/v1";

// Mensagens (data types)
message User {
  string id = 1;
  string name = 2;
  string email = 3;
  Role role = 4;
  google.protobuf.Timestamp created_at = 5;
}

message GetUserRequest {
  string id = 1;
}

message ListUsersRequest {
  Role role = 1;
  int32 limit = 2;
  int32 offset = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
}

enum Role {
  ROLE_UNSPECIFIED = 0;
  ROLE_USER = 1;
  ROLE_ADMIN = 2;
}

// Serviço (API)
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser(User) returns (User);
  
  // Streaming
  rpc WatchUser(GetUserRequest) returns (stream User);           // Server stream
  rpc UploadUsers(stream User) returns (ListUsersResponse);      // Client stream
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);     // Bidirecional
}
```

### 4 Tipos de RPC

```
1. Unary (request-response):
   Client ── request ──▶ Server
   Client ◀── response ── Server

2. Server Streaming:
   Client ── request ──▶ Server
   Client ◀── stream ──── Server  (múltiplas respostas)
   Client ◀── stream ──── Server
   Client ◀── stream ──── Server

3. Client Streaming:
   Client ── stream ──▶ Server    (múltiplos requests)
   Client ── stream ──▶ Server
   Client ── stream ──▶ Server
   Client ◀── response ── Server  (uma resposta final)

4. Bidirectional Streaming:
   Client ◀══ stream ══▶ Server   (ambos enviam livremente)
```

### Performance: JSON vs Protocol Buffers

```
Comparação para a mesma mensagem User:

JSON (REST):
{
  "id": "user-12345",
  "name": "Maria Silva",
  "email": "maria@example.com",
  "role": "ADMIN",
  "created_at": "2024-01-15T10:30:00Z"
}
→ ~157 bytes (text, human-readable)

Protocol Buffers (gRPC):
→ ~48 bytes (binary, compact)

  Serialização:   Protobuf ~6x mais rápido que JSON
  Tamanho:        Protobuf ~3x menor que JSON
  Desserialização: Protobuf ~6x mais rápido que JSON
  
  Em alta escala (milhões de RPCs/s): diferença enorme em CPU e bandwidth
```

### HTTP/2 Benefits

```
HTTP/1.1 (REST):
  ┌──── Connection 1 ────┐
  │ Request 1             │
  │ Response 1            │
  │ Request 2 (wait...)   │  ← Head-of-line blocking
  │ Response 2            │
  └───────────────────────┘

HTTP/2 (gRPC):
  ┌──── Single Connection ────┐
  │ Stream 1: Request ──▶     │
  │ Stream 2: Request ──▶     │  ← Multiplexing!
  │ Stream 1: ◀── Response    │
  │ Stream 3: Request ──▶     │
  │ Stream 2: ◀── Response    │
  │ Stream 3: ◀── Response    │
  └───────────────────────────┘
  
  + Header compression (HPACK)
  + Server push
  + Binary framing (mais eficiente que text)
```

### Problemas do gRPC

```
1. Browser Support:
   gRPC usa HTTP/2 features que browsers não expõem para JS
   Solução: grpc-web (proxy Envoy traduz HTTP/2 ↔ HTTP/1.1)

2. Debugging:
   Binary data → não legível em ferramentas HTTP padrão
   Solução: grpcurl, Postman gRPC, Bloom RPC, grpc-tools

3. Load Balancing:
   HTTP/2 multiplexing = uma connection por client
   → L4 LB distribui mal (1 connection = 1 backend)
   Solução: L7 LB (Envoy) ou client-side LB

4. Não cacheable nativamente:
   Sem suporte a HTTP cache, ETags, CDN
   Solução: application-level caching
```

### Quando Usar gRPC

```
✅ Service-to-service interno (microservices)
✅ Alta performance / baixa latência obrigatória
✅ Streaming bidirecional (chat, IoT, telemetria)
✅ Polyglot environment (Go, Java, Python, C++)
✅ Schema enforcement obrigatório
```

---

## GraphQL Federation (Netflix / Apollo)

```
Problema: Em microservices, cada serviço tem seu schema.
  Como unificar em uma API GraphQL única?

Apollo Federation:

  ┌──────────────┐
  │   Gateway    │ ← Client fala com 1 endpoint
  │  (Apollo     │
  │   Router)    │
  └──────┬───────┘
         │
    ┌────┼────┬────────────┐
    ▼    ▼    ▼            ▼
  ┌────┐┌────┐┌─────────┐┌──────┐
  │User││Order││Inventory││Ship- │  ← Cada serviço tem seu subgraph
  │Svc ││Svc  ││Service  ││ping  │
  └────┘└────┘└─────────┘└──────┘

  User subgraph:
    type User @key(fields: "id") {
      id: ID!
      name: String!
    }

  Order subgraph:
    extend type User @key(fields: "id") {
      id: ID! @external
      orders: [Order!]!  ← estende User com orders
    }
    
  Gateway compõe o schema completo automaticamente.
```

---

## Decision Matrix

```
┌─────────────────────────────────────────────────────────┐
│  Cenário                        │ Recomendação          │
├─────────────────────────────────┼───────────────────────┤
│  API pública para 3rd parties   │ REST                  │
│  Mobile app com dados variados  │ GraphQL               │
│  Microservice ↔ Microservice    │ gRPC                  │
│  Real-time streaming            │ gRPC (bidirecional)   │
│  BFF (Backend for Frontend)     │ GraphQL               │
│  CRUD simples em browser        │ REST                  │
│  Internal tools / admin         │ REST ou GraphQL       │
│  IoT / embedded devices         │ gRPC (compact)        │
│  High-throughput data pipeline  │ gRPC                  │
│  Multiple clients (web+mobile)  │ GraphQL               │
└─────────────────────────────────┴───────────────────────┘

Combinação comum em Big Techs:
  Público:  REST (ou GraphQL)
  BFF:      GraphQL
  Interno:  gRPC
```

---

## Uso em Big Techs

| Empresa | REST | GraphQL | gRPC | Detalhes |
|---------|------|---------|------|----------|
| **Meta** | APIs externas | Core (inventaram) | Interno | GraphQL para Facebook, Instagram APIs |
| **Google** | APIs públicas (Maps, etc.) | - | Core (inventaram) | gRPC para todo serviço interno |
| **Netflix** | APIs públicas | Federation (Apollo) | Interno | GraphQL DGS Framework (open-source) |
| **Uber** | APIs externas | - | Core interno | gRPC para 2500+ microservices |
| **Airbnb** | APIs externas | BFF | Interno | Migrou BFF para GraphQL |
| **Shopify** | Admin API | Storefront API | - | GraphQL para lojistas / developers |
| **GitHub** | REST v3 | GraphQL v4 | - | GraphQL API para flexibilidade |
| **Stripe** | Core (REST v1) | - | - | REST com excelente DX |
| **Twitter/X** | REST v2 | GraphQL (interno) | Interno | GraphQL para mobile app |
| **Spotify** | REST público | - | Interno | gRPC entre microservices |

---

## Perguntas Frequentes em Entrevistas

1. **"Quando usar REST vs GraphQL vs gRPC?"**
   - REST: APIs públicas, CRUD, caching HTTP
   - GraphQL: mobile, múltiplos clients, evitar over-fetching
   - gRPC: service-to-service, alta performance, streaming

2. **"Quais os problemas do REST?"**
   - Over-fetching (muitos dados retornados)
   - Under-fetching (N+1 requests)
   - Versionamento (manter v1 e v2)

3. **"Por que gRPC é mais rápido?"**
   - Protocol Buffers: binário, ~3x menor, ~6x mais rápido
   - HTTP/2: multiplexing, header compression
   - Code generation: sem reflection, tipado

4. **"Como Netflix usa GraphQL?"**
   - GraphQL Federation com Apollo Router
   - Cada microservice expõe subgraph
   - Gateway compõe schema unificado
   - DGS Framework (Java/Kotlin, open-source)

5. **"GraphQL tem problemas de segurança?"**
   - Query complexity attacks (nested queries infinitos)
   - Solução: depth limiting, persisted queries, complexity analysis
   - Rate limiting por complexity (não por request)

---

## Referências

- Fielding, R. (2000) — *"Architectural Styles and the Design of Network-based Software Architectures"* — REST Dissertation
- Facebook (2015) — *"GraphQL: A data query language"* — graphql.org
- Google (2016) — *"gRPC: A high performance, open-source universal RPC framework"* — grpc.io
- Netflix Tech Blog — *"How Netflix Scales its API with GraphQL Federation"*
- Uber Engineering — *"Uber's gRPC Ecosystem"*
- Apollo GraphQL — *"Introduction to Apollo Federation"*
