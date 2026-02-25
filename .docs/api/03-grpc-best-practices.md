# gRPC — Best Practices

> Guia completo de boas práticas para design, implementação e manutenção de APIs gRPC, alinhado com padrões de mercado adotados por empresas como Google, Netflix, Uber, Lyft e Dropbox.

---

## Sumário

1. [Fundamentos e Quando Usar gRPC](#1-fundamentos-e-quando-usar-grpc)
2. [Design de Protobuf (Protocol Buffers)](#2-design-de-protobuf-protocol-buffers)
3. [Naming Conventions](#3-naming-conventions)
4. [Padrões de Comunicação](#4-padrões-de-comunicação)
5. [Tratamento de Erros](#5-tratamento-de-erros)
6. [Deadlines, Timeouts e Cancelamento](#6-deadlines-timeouts-e-cancelamento)
7. [Interceptors (Middleware)](#7-interceptors-middleware)
8. [Segurança](#8-segurança)
9. [Load Balancing](#9-load-balancing)
10. [Health Checking](#10-health-checking)
11. [Versionamento e Evolução de APIs](#11-versionamento-e-evolução-de-apis)
12. [Streaming Best Practices](#12-streaming-best-practices)
13. [Performance e Otimização](#13-performance-e-otimização)
14. [Observabilidade](#14-observabilidade)
15. [Testing](#15-testing)
16. [Padrões de Arquitetura](#16-padrões-de-arquitetura)
17. [Graceful Shutdown](#17-graceful-shutdown)
18. [gRPC Reflection](#18-grpc-reflection)

---

## 1. Fundamentos e Quando Usar gRPC

### gRPC vs REST vs GraphQL

| Aspecto | gRPC | REST | GraphQL |
|---------|------|------|---------|
| Protocolo | HTTP/2 | HTTP/1.1 ou HTTP/2 | HTTP/1.1 ou HTTP/2 |
| Formato | Protobuf (binário) | JSON (texto) | JSON (texto) |
| Contrato | `.proto` (forte) | OpenAPI (opcional) | Schema (forte) |
| Streaming | Bidirecional nativo | Limitado (SSE/WS) | Subscriptions |
| Performance | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| Browser support | Via gRPC-Web/Envoy | Nativo | Nativo |
| Code generation | Nativo e obrigatório | Opcional | Opcional |
| Latência | Muito baixa | Média | Média |

### Quando Usar gRPC

| Cenário | Recomendado | Motivo |
|---------|:-----------:|--------|
| Microserviços internos | ✅ | Alta performance, contrato forte |
| Comunicação real-time | ✅ | Streaming bidirecional |
| Polyglot (multi-linguagem) | ✅ | Code-gen para 12+ linguagens |
| IoT / Dispositivos com pouca banda | ✅ | Payload binário compacto |
| APIs públicas para browsers | ⚠️ | Requer gRPC-Web proxy |
| CRUD simples / APIs públicas | ❌ | REST é mais adequado |
| Queries flexíveis client-driven | ❌ | GraphQL é mais adequado |

---

## 2. Design de Protobuf (Protocol Buffers)

### Estrutura de Arquivos `.proto`

```protobuf
syntax = "proto3";

package company.platform.user.v1;

option java_package = "com.company.platform.user.v1";
option java_multiple_files = true;
option go_package = "github.com/company/platform/user/v1;userv1";
option csharp_namespace = "Company.Platform.User.V1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/field_mask.proto";
import "google/protobuf/empty.proto";
import "google/api/annotations.proto";
```

### Organização de Arquivos

```
proto/
├── company/
│   └── platform/
│       ├── common/
│       │   └── v1/
│       │       ├── pagination.proto
│       │       ├── error.proto
│       │       └── money.proto
│       ├── user/
│       │   └── v1/
│       │       ├── user.proto          # Mensagens
│       │       ├── user_service.proto   # Service definition
│       │       └── user_enums.proto     # Enums
│       └── order/
│           └── v1/
│               ├── order.proto
│               └── order_service.proto
```

### Definição de Mensagens

```protobuf
// Mensagem principal do recurso
message User {
  // Identificador único do usuário (UUID)
  string id = 1;

  // Nome completo do usuário
  string name = 2;

  // Email do usuário (único)
  string email = 3;

  // Número de telefone no formato E.164
  string phone = 4;

  // Status atual do usuário
  UserStatus status = 5;

  // Endereço do usuário
  Address address = 6;

  // Data de criação
  google.protobuf.Timestamp created_at = 7;

  // Data da última atualização
  google.protobuf.Timestamp updated_at = 8;
}

enum UserStatus {
  // Valor padrão — nunca use como status real
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_SUSPENDED = 3;
  USER_STATUS_DELETED = 4;
}

message Address {
  string street = 1;
  string city = 2;
  string state = 3;
  string zip_code = 4;
  string country = 5;     // ISO 3166-1 alpha-2
}
```

### Regras de Numeração de Campos

| Regra | Detalhes |
|-------|---------|
| Nunca reutilize números | Mesmo após remover um campo |
| Reserve campos removidos | `reserved 5, 6; reserved "old_field";` |
| Campos 1-15: 1 byte | Use para campos mais frequentes |
| Campos 16-2047: 2 bytes | Campos menos frequentes |
| Evite 19000-19999 | Reservados pelo Protobuf |

```protobuf
message User {
  reserved 9, 10, 15;
  reserved "legacy_id", "deprecated_field";

  string id = 1;         // 1 byte — campo frequente
  string name = 2;       // 1 byte — campo frequente
  string metadata = 100; // 2 bytes — campo raro
}
```

---

## 3. Naming Conventions

### Regras de Nomenclatura

| Elemento | Convenção | Exemplo |
|----------|-----------|---------|
| Package | `lowercase.dot.separated` | `company.platform.user.v1` |
| Service | PascalCase + `Service` | `UserService` |
| RPC methods | PascalCase (verbo + substantivo) | `GetUser`, `ListUsers` |
| Messages | PascalCase | `CreateUserRequest` |
| Fields | snake_case | `first_name`, `created_at` |
| Enums | PascalCase | `UserStatus` |
| Enum values | SCREAMING_SNAKE_CASE com prefixo | `USER_STATUS_ACTIVE` |
| Arquivos | snake_case.proto | `user_service.proto` |
| oneof | snake_case | `delivery_method` |

### Padrão de Request/Response

```protobuf
// Padrão: {Verbo}{Recurso}Request / {Verbo}{Recurso}Response
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
}

// Enum values SEMPRE com prefixo do enum
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;   // ✅ Com prefixo
  ORDER_STATUS_PENDING = 1;
  // PENDING = 1;                 // ❌ Sem prefixo — pode colidir
}
```

---

## 4. Padrões de Comunicação

### Unary RPC (Request/Response)

```protobuf
// Padrão mais comum — similar a REST
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest {
  string id = 1;
}

message GetUserResponse {
  User user = 1;
}
```

### Server Streaming

```protobuf
// Server envia múltiplas respostas para um request
service ReportService {
  rpc GenerateReport(GenerateReportRequest) returns (stream ReportChunk);
}

message GenerateReportRequest {
  string report_type = 1;
  google.protobuf.Timestamp start_date = 2;
  google.protobuf.Timestamp end_date = 3;
}

message ReportChunk {
  bytes data = 1;
  int32 chunk_number = 2;
  int32 total_chunks = 3;
}
```

### Client Streaming

```protobuf
// Client envia múltiplas mensagens, server responde uma vez
service FileService {
  rpc UploadFile(stream UploadFileRequest) returns (UploadFileResponse);
}

message UploadFileRequest {
  oneof data {
    FileMetadata metadata = 1;   // Primeira mensagem
    bytes chunk = 2;              // Mensagens subsequentes
  }
}

message FileMetadata {
  string filename = 1;
  string content_type = 2;
  int64 size_bytes = 3;
}

message UploadFileResponse {
  string file_id = 1;
  string url = 2;
  int64 size_bytes = 3;
}
```

### Bidirectional Streaming

```protobuf
// Ambos enviam stream de mensagens independentemente
service ChatService {
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message ChatMessage {
  string sender_id = 1;
  string content = 2;
  google.protobuf.Timestamp timestamp = 3;
  MessageType type = 4;
}

enum MessageType {
  MESSAGE_TYPE_UNSPECIFIED = 0;
  MESSAGE_TYPE_TEXT = 1;
  MESSAGE_TYPE_IMAGE = 2;
  MESSAGE_TYPE_TYPING = 3;
}
```

### Quando Usar Cada Padrão

| Padrão | Caso de Uso |
|--------|------------|
| **Unary** | CRUD, queries simples, a maioria das RPCs |
| **Server Streaming** | Feeds de dados, downloads grandes, real-time updates |
| **Client Streaming** | Upload de arquivos, envio de métricas em batch |
| **Bidirectional** | Chat, gaming, sincronização em tempo real |

---

## 5. Tratamento de Erros

### gRPC Status Codes

| Code | Nome | Uso |
|------|------|-----|
| 0 | `OK` | Sucesso |
| 1 | `CANCELLED` | Operação cancelada pelo client |
| 2 | `UNKNOWN` | Erro desconhecido |
| 3 | `INVALID_ARGUMENT` | Argumento inválido (ex: email malformado) |
| 4 | `DEADLINE_EXCEEDED` | Timeout |
| 5 | `NOT_FOUND` | Recurso não encontrado |
| 6 | `ALREADY_EXISTS` | Recurso já existe (conflito) |
| 7 | `PERMISSION_DENIED` | Sem permissão (autenticado mas não autorizado) |
| 8 | `RESOURCE_EXHAUSTED` | Rate limit, quota excedida |
| 9 | `FAILED_PRECONDITION` | Pré-condição não atendida |
| 10 | `ABORTED` | Operação abortada (concorrência) |
| 11 | `OUT_OF_RANGE` | Valor fora do range permitido |
| 12 | `UNIMPLEMENTED` | RPC não implementado |
| 13 | `INTERNAL` | Erro interno do servidor |
| 14 | `UNAVAILABLE` | Serviço temporariamente indisponível (retry) |
| 15 | `DATA_LOSS` | Perda de dados irrecuperável |
| 16 | `UNAUTHENTICATED` | Não autenticado |

### Mapeamento gRPC ↔ HTTP

| gRPC Status | HTTP Status |
|-------------|-------------|
| `OK` | 200 |
| `INVALID_ARGUMENT` | 400 |
| `UNAUTHENTICATED` | 401 |
| `PERMISSION_DENIED` | 403 |
| `NOT_FOUND` | 404 |
| `ALREADY_EXISTS` | 409 |
| `RESOURCE_EXHAUSTED` | 429 |
| `CANCELLED` | 499 |
| `INTERNAL` | 500 |
| `UNAVAILABLE` | 503 |
| `DATA_LOSS` | 500 |
| `DEADLINE_EXCEEDED` | 504 |

### Rich Error Model (Google Error Model)

```protobuf
import "google/rpc/status.proto";
import "google/rpc/error_details.proto";

// Usar google.rpc.Status com details para erros ricos
```

```java
// Java — Criando erros detalhados
Status status = Status.newBuilder()
    .setCode(Code.INVALID_ARGUMENT.getNumber())
    .setMessage("Validation failed")
    .addDetails(Any.pack(
        BadRequest.newBuilder()
            .addFieldViolations(
                BadRequest.FieldViolation.newBuilder()
                    .setField("email")
                    .setDescription("Must be a valid email address")
                    .build()
            )
            .addFieldViolations(
                BadRequest.FieldViolation.newBuilder()
                    .setField("name")
                    .setDescription("Must not be empty")
                    .build()
            )
            .build()
    ))
    .build();

throw StatusProto.toStatusRuntimeException(status);
```

```go
// Go — Criando erros detalhados
st, _ := status.New(codes.InvalidArgument, "Validation failed").
    WithDetails(&errdetails.BadRequest{
        FieldViolations: []*errdetails.BadRequest_FieldViolation{
            {Field: "email", Description: "Must be a valid email address"},
            {Field: "name", Description: "Must not be empty"},
        },
    })
return nil, st.Err()
```

### Boas Práticas de Erros

- **Sempre use o status code correto** — não abuse de `INTERNAL`.
- **Mensagens claras e acionáveis** para o desenvolvedor.
- **Use Rich Error Model** para erros de validação.
- **Inclua informações de retry** em `UNAVAILABLE` e `RESOURCE_EXHAUSTED`.
- **Não exponha detalhes internos** (stack traces, queries SQL).
- **Log com trace ID** para correlação.

---

## 6. Deadlines, Timeouts e Cancelamento

### Deadlines (sempre defina!)

```go
// Go — SEMPRE defina deadline/timeout
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: "123"})
if err != nil {
    st, ok := status.FromError(err)
    if ok && st.Code() == codes.DeadlineExceeded {
        // Handle timeout
    }
}
```

```java
// Java — Definir deadline
UserResponse response = userServiceStub
    .withDeadlineAfter(5, TimeUnit.SECONDS)
    .getUser(GetUserRequest.newBuilder().setId("123").build());
```

### Propagação de Deadlines

```
Client (deadline: 10s)
  └─▶ Service A (remaining: 10s, consumes 2s)
       └─▶ Service B (remaining: 8s, consumes 3s)
            └─▶ Service C (remaining: 5s)
```

### Boas Práticas

| Prática | Detalhes |
|---------|---------|
| **Sempre defina deadline** | Nunca faça chamadas sem timeout |
| **Propague deadlines** | Repasse o contexto para chamadas downstream |
| **Subtraia margem** | Reserve tempo para processamento local |
| **Cancele quando possível** | Se o client cancelou, pare o processamento |
| **Timeouts recomendados** | Unary: 1-30s, Streaming: 5-300s |

### Timeouts Recomendados por Tipo de Operação

| Operação | Timeout | Justificativa |
|----------|---------|---------------|
| Leitura simples | 1-5s | Rápido, sem side-effects |
| Escrita | 5-10s | Pode envolver validação |
| Operação complexa | 10-30s | Múltiplas chamadas internas |
| Upload/Download | 60-300s | Dados grandes |
| Long-polling/Stream | 300s+ | Conexão persistente |

---

## 7. Interceptors (Middleware)

### Tipos de Interceptors

```
Client Request ──▶ [Client Interceptor] ──▶ [Server Interceptor] ──▶ Handler
                                                                         │
Client Response ◀── [Client Interceptor] ◀── [Server Interceptor] ◀──────┘
```

### Server Interceptors (Go)

```go
// Logging interceptor
func loggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()

    // Extrair metadata
    md, _ := metadata.FromIncomingContext(ctx)
    traceID := md.Get("x-trace-id")

    // Executar handler
    resp, err := handler(ctx, req)

    // Log estruturado
    log.Info("gRPC call",
        "method", info.FullMethod,
        "duration", time.Since(start),
        "traceId", traceID,
        "error", err,
    )

    return resp, err
}

// Compor interceptors
server := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        recoveryInterceptor,   // 1. Recuperar panics
        loggingInterceptor,    // 2. Logging
        authInterceptor,       // 3. Autenticação
        rateLimitInterceptor,  // 4. Rate limiting
        validationInterceptor, // 5. Validação
    ),
    grpc.ChainStreamInterceptor(
        streamRecoveryInterceptor,
        streamLoggingInterceptor,
        streamAuthInterceptor,
    ),
)
```

### Ordem Recomendada de Interceptors

```
1. Recovery (panic handler)
2. Tracing (OpenTelemetry)
3. Logging
4. Metrics
5. Authentication
6. Rate Limiting
7. Validation
8. → Handler (sua lógica de negócio)
```

---

## 8. Segurança

### TLS (Obrigatório em Produção)

```go
// Server com TLS
creds, _ := credentials.NewServerTLSFromFile("server.crt", "server.key")
server := grpc.NewServer(grpc.Creds(creds))

// Client com TLS
creds, _ := credentials.NewClientTLSFromFile("ca.crt", "api.example.com")
conn, _ := grpc.Dial("api.example.com:443", grpc.WithTransportCredentials(creds))
```

### mTLS (Mutual TLS — recomendado para service-to-service)

```go
// Server com mTLS
cert, _ := tls.LoadX509KeyPair("server.crt", "server.key")
certPool := x509.NewCertPool()
certPool.AppendCertsFromPEM(caCert)

tlsConfig := &tls.Config{
    Certificates: []tls.Certificate{cert},
    ClientAuth:   tls.RequireAndVerifyClientCert,
    ClientCAs:    certPool,
}
creds := credentials.NewTLS(tlsConfig)
server := grpc.NewServer(grpc.Creds(creds))
```

### Token-based Authentication

```go
// Client — Adicionar token via metadata
type tokenAuth struct {
    token string
}

func (t *tokenAuth) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
    return map[string]string{
        "authorization": "Bearer " + t.token,
    }, nil
}

func (t *tokenAuth) RequireTransportSecurity() bool {
    return true // Obrigatório com TLS
}

// Conectar com token
conn, _ := grpc.Dial("api.example.com:443",
    grpc.WithTransportCredentials(creds),
    grpc.WithPerRPCCredentials(&tokenAuth{token: "jwt-token"}),
)
```

### Checklist de Segurança gRPC

- [ ] **TLS obrigatório** em produção (nunca plaintext).
- [ ] **mTLS** para comunicação entre serviços internos.
- [ ] **Autenticação** por JWT/OAuth2 via metadata.
- [ ] **Autorização** por RPC (field-level se necessário).
- [ ] **Rate limiting** por client/IP/serviço.
- [ ] **Payload size limit** — `MaxRecvMsgSize` (default: 4MB).
- [ ] **Keepalive** configurado para detectar conexões mortas.
- [ ] **Interceptor de validação** para sanitizar inputs.
- [ ] **Não logar dados sensíveis** do payload.
- [ ] **Rotação de certificados** automatizada.

---

## 9. Load Balancing

### Estratégias

```
┌─────────────────────────────────────────┐
│        Proxy-based (L7 / Envoy)         │  ← Recomendado para Kubernetes
│  Client ──▶ Load Balancer ──▶ Servers   │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│         Client-side (gRPC built-in)      │  ← Para comunicação direta
│  Client ──▶ (round-robin) ──▶ Servers   │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│         Look-aside (xDS / Consul)        │  ← Para service mesh
│  Client ◀──▶ LB Service ──▶ Servers    │
└─────────────────────────────────────────┘
```

### Por que Load Balancing gRPC é diferente?

> gRPC usa HTTP/2 com **conexões persistentes e multiplexação**. Um load balancer L4 (TCP) distribui apenas a conexão, não os requests. Use **L7 (HTTP/2-aware)** como Envoy, Istio ou Linkerd.

### Configuração de Client-side LB

```go
// Round-robin com service discovery
conn, _ := grpc.Dial(
    "dns:///user-service.default.svc.cluster.local:50051",
    grpc.WithDefaultServiceConfig(`{"loadBalancingConfig":[{"round_robin":{}}]}`),
    grpc.WithTransportCredentials(creds),
)
```

### Envoy como gRPC Proxy (Recomendado)

```yaml
# envoy.yaml
static_resources:
  listeners:
    - name: grpc_listener
      address:
        socket_address: { address: 0.0.0.0, port_value: 8080 }
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                codec_type: AUTO
                stat_prefix: grpc
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: grpc_service
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/" }
                          route:
                            cluster: grpc_backend
                            timeout: 30s
                            retry_policy:
                              retry_on: "unavailable,resource-exhausted"
                              num_retries: 3
                http_filters:
                  - name: envoy.filters.http.router
  clusters:
    - name: grpc_backend
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      typed_extension_protocol_options:
        envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
          "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
          explicit_http_config:
            http2_protocol_options: {}
      load_assignment:
        cluster_name: grpc_backend
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address: { address: user-service, port_value: 50051 }
```

---

## 10. Health Checking

### gRPC Health Checking Protocol (padrão oficial)

```protobuf
// Definido em grpc/health/v1/health.proto (padrão do gRPC)
syntax = "proto3";

package grpc.health.v1;

service Health {
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}

message HealthCheckRequest {
  string service = 1;  // Nome do serviço (vazio = status geral)
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
    SERVICE_UNKNOWN = 3;
  }
  ServingStatus status = 1;
}
```

### Implementação (Go)

```go
import "google.golang.org/grpc/health"
import healthpb "google.golang.org/grpc/health/grpc_health_v1"

// Registrar health service
healthServer := health.NewServer()
healthpb.RegisterHealthServer(grpcServer, healthServer)

// Atualizar status
healthServer.SetServingStatus("user.v1.UserService", healthpb.HealthCheckResponse_SERVING)
healthServer.SetServingStatus("", healthpb.HealthCheckResponse_SERVING) // Status geral

// Em caso de problema com dependência
healthServer.SetServingStatus("user.v1.UserService", healthpb.HealthCheckResponse_NOT_SERVING)
```

### Kubernetes gRPC Health Probes

```yaml
# kubernetes deployment
spec:
  containers:
    - name: user-service
      ports:
        - containerPort: 50051
      livenessProbe:
        grpc:
          port: 50051
        initialDelaySeconds: 10
        periodSeconds: 10
      readinessProbe:
        grpc:
          port: 50051
          service: "user.v1.UserService"
        initialDelaySeconds: 5
        periodSeconds: 5
      startupProbe:
        grpc:
          port: 50051
        failureThreshold: 30
        periodSeconds: 2
```

---

## 11. Versionamento e Evolução de APIs

### Versionamento via Package

```protobuf
// v1
package company.platform.user.v1;

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}

// v2 — novo package, novo serviço
package company.platform.user.v2;

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc SearchUsers(SearchUsersRequest) returns (SearchUsersResponse);  // Novo RPC
}
```

### Regras de Compatibilidade (Wire Compatibility)

#### Mudanças SEGURAS (backward compatible)

```protobuf
// ✅ Adicionar novo campo (número novo)
message User {
  string id = 1;
  string name = 2;
  string email = 3;
  string phone = 4;          // NOVO — safe, campo opcional
}

// ✅ Adicionar novo RPC
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc SearchUsers(SearchUsersRequest) returns (SearchUsersResponse);  // NOVO
}

// ✅ Adicionar novo valor a enum
enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_SUSPENDED = 3;   // NOVO
}
```

#### Mudanças BREAKING (evitar!)

```protobuf
// ❌ Remover ou renumerar campo
message User {
  string id = 1;
  // string name = 2;   // REMOVIDO — BREAKING!
  string email = 3;
}

// ❌ Mudar tipo de campo
message User {
  int64 id = 1;   // ERA string — BREAKING!
}

// ❌ Renomear campo (okay em proto, BREAKING em JSON)
message User {
  string user_name = 2;  // ERA "name" — BREAKING em JSON mapping!
}

// ❌ Mudar campo de singular para repeated (ou vice-versa)
```

### Uso de `reserved`

```protobuf
message User {
  reserved 5, 6, 9 to 11;
  reserved "old_name", "legacy_field";

  string id = 1;
  string name = 2;
  string email = 3;
}
```

### Buf Schema Registry (Recomendado)

```yaml
# buf.yaml
version: v2
modules:
  - path: proto
    lint:
      use:
        - DEFAULT
      except:
        - PACKAGE_VERSION_SUFFIX
    breaking:
      use:
        - FILE
```

```bash
# Verificar breaking changes
buf breaking --against '.git#branch=main'

# Lint do schema
buf lint

# Gerar código
buf generate
```

---

## 12. Streaming Best Practices

### Flow Control

```go
// Server streaming com flow control
func (s *Server) GenerateReport(req *pb.ReportRequest, stream pb.ReportService_GenerateReportServer) error {
    for i := 0; i < totalChunks; i++ {
        chunk := generateChunk(i)

        // Verificar se o client cancelou
        if stream.Context().Err() != nil {
            return status.Error(codes.Cancelled, "client cancelled")
        }

        if err := stream.Send(chunk); err != nil {
            return err
        }
    }
    return nil
}
```

### Bidirectional Streaming Pattern

```go
// Padrão goroutine para bidi streaming
func (s *Server) Chat(stream pb.ChatService_ChatServer) error {
    // Channel para erros
    errCh := make(chan error, 1)

    // Goroutine para receber mensagens do client
    go func() {
        for {
            msg, err := stream.Recv()
            if err == io.EOF {
                errCh <- nil
                return
            }
            if err != nil {
                errCh <- err
                return
            }
            // Processar mensagem recebida
            s.handleMessage(msg)
        }
    }()

    // Loop principal — enviar mensagens para o client
    for {
        select {
        case err := <-errCh:
            return err
        case msg := <-s.outgoing:
            if err := stream.Send(msg); err != nil {
                return err
            }
        case <-stream.Context().Done():
            return stream.Context().Err()
        }
    }
}
```

### Boas Práticas de Streaming

| Prática | Detalhes |
|---------|---------|
| **Verifique Context** | Sempre cheque `ctx.Err()` / `stream.Context().Done()` |
| **Chunk size** | 16KB-64KB para boa throughput sem excesso de memória |
| **Keepalive** | Configure keepalive para detectar conexões mortas |
| **Back-pressure** | gRPC HTTP/2 flow control cuida automaticamente |
| **Graceful shutdown** | Termine streams ativas antes de derrubar o server |
| **Metadata no início** | Envie metadata via header, não no payload do stream |
| **Idempotência** | Client deve poder reconectar e retomar stream |

---

## 13. Performance e Otimização

### Configurações de Performance

```go
// Server
server := grpc.NewServer(
    grpc.MaxRecvMsgSize(4 * 1024 * 1024),      // 4MB max receive
    grpc.MaxSendMsgSize(4 * 1024 * 1024),      // 4MB max send
    grpc.MaxConcurrentStreams(1000),             // Streams simultâneas
    grpc.KeepaliveParams(keepalive.ServerParameters{
        MaxConnectionIdle:     15 * time.Minute,
        MaxConnectionAge:      30 * time.Minute,
        MaxConnectionAgeGrace: 5 * time.Second,
        Time:                  5 * time.Minute,
        Timeout:               1 * time.Second,
    }),
    grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
        MinTime:             5 * time.Second,
        PermitWithoutStream: true,
    }),
)

// Client
conn, _ := grpc.Dial(address,
    grpc.WithDefaultCallOptions(
        grpc.MaxCallRecvMsgSize(4 * 1024 * 1024),
        grpc.MaxCallSendMsgSize(4 * 1024 * 1024),
    ),
    grpc.WithKeepaliveParams(keepalive.ClientParameters{
        Time:                10 * time.Second,
        Timeout:             1 * time.Second,
        PermitWithoutStream: true,
    }),
)
```

### Connection Pooling

```go
// Reutilize conexões — não crie uma nova por request
// Uma conexão gRPC suporta múltiplos streams HTTP/2
var conn *grpc.ClientConn

func init() {
    var err error
    conn, err = grpc.Dial("user-service:50051",
        grpc.WithTransportCredentials(creds),
        grpc.WithDefaultServiceConfig(`{"loadBalancingConfig":[{"round_robin":{}}]}`),
    )
    if err != nil {
        log.Fatalf("failed to dial: %v", err)
    }
}

// Reutilize o stub
var userClient = pb.NewUserServiceClient(conn)
```

### Retry Policy

```go
// Configurar retry via service config
serviceConfig := `{
    "methodConfig": [{
        "name": [{"service": "company.platform.user.v1.UserService"}],
        "retryPolicy": {
            "maxAttempts": 3,
            "initialBackoff": "0.1s",
            "maxBackoff": "1s",
            "backoffMultiplier": 2,
            "retryableStatusCodes": ["UNAVAILABLE", "RESOURCE_EXHAUSTED"]
        }
    }]
}`

conn, _ := grpc.Dial(address,
    grpc.WithDefaultServiceConfig(serviceConfig),
)
```

### Benchmarks Comparativos

| Métrica | gRPC (Protobuf) | REST (JSON) | Diferença |
|---------|:---------------:|:-----------:|:---------:|
| Tamanho do payload | ~30 bytes | ~120 bytes | ~75% menor |
| Serialização | ~0.5μs | ~5μs | ~10x mais rápido |
| Latência (p50) | ~1ms | ~5ms | ~5x menor |
| Throughput | ~50k req/s | ~10k req/s | ~5x maior |

> *Valores aproximados — variam conforme hardware e payload.

---

## 14. Observabilidade

### OpenTelemetry Integration

```go
import (
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

// Server com OpenTelemetry
server := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)

// Client com OpenTelemetry
conn, _ := grpc.Dial(address,
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
)
```

### Métricas Essenciais

| Métrica | Descrição | Labels |
|---------|-----------|--------|
| `grpc_server_handled_total` | Total de RPCs completadas | method, code |
| `grpc_server_handling_seconds` | Latência de RPCs | method |
| `grpc_server_msg_received_total` | Mensagens recebidas (streaming) | method |
| `grpc_server_msg_sent_total` | Mensagens enviadas (streaming) | method |
| `grpc_server_started_total` | RPCs iniciadas | method |
| `grpc_client_handled_total` | RPCs completadas no client | method, code |
| `grpc_client_handling_seconds` | Latência no client | method |

### Logging Estruturado

```json
{
  "level": "info",
  "timestamp": "2026-02-23T10:30:00Z",
  "service": "user-service",
  "traceId": "abc-123",
  "spanId": "def-456",
  "grpc": {
    "method": "/company.platform.user.v1.UserService/GetUser",
    "type": "unary",
    "code": "OK",
    "duration_ms": 12,
    "peer": "10.0.0.5:42000"
  }
}
```

### Distributed Tracing

```
Client ──[span: GetUser]──▶ API Gateway ──[span: GetUser]──▶ User Service
                                                                    │
                                                     [span: DB Query]
                                                     [span: Cache Check]
```

---

## 15. Testing

### Unit Testing (Resolvers/Handlers)

```go
func TestGetUser(t *testing.T) {
    // Arrange
    mockRepo := &MockUserRepository{
        users: map[string]*User{
            "123": {ID: "123", Name: "João", Email: "joao@ex.com"},
        },
    }
    handler := NewUserServiceHandler(mockRepo)

    // Act
    resp, err := handler.GetUser(context.Background(), &pb.GetUserRequest{Id: "123"})

    // Assert
    assert.NoError(t, err)
    assert.Equal(t, "João", resp.User.Name)
}

func TestGetUser_NotFound(t *testing.T) {
    mockRepo := &MockUserRepository{users: map[string]*User{}}
    handler := NewUserServiceHandler(mockRepo)

    _, err := handler.GetUser(context.Background(), &pb.GetUserRequest{Id: "999"})

    st, ok := status.FromError(err)
    assert.True(t, ok)
    assert.Equal(t, codes.NotFound, st.Code())
}
```

### Integration Testing com bufconn

```go
import "google.golang.org/grpc/test/bufconn"

const bufSize = 1024 * 1024

var lis *bufconn.Listener

func init() {
    lis = bufconn.Listen(bufSize)
    server := grpc.NewServer()
    pb.RegisterUserServiceServer(server, &UserServiceServer{})
    go server.Serve(lis)
}

func bufDialer(context.Context, string) (net.Conn, error) {
    return lis.Dial()
}

func TestIntegration_GetUser(t *testing.T) {
    ctx := context.Background()
    conn, _ := grpc.DialContext(ctx, "bufnet",
        grpc.WithContextDialer(bufDialer),
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    defer conn.Close()

    client := pb.NewUserServiceClient(conn)
    resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: "123"})

    assert.NoError(t, err)
    assert.Equal(t, "João", resp.User.Name)
}
```

### Contract Testing

```bash
# Buf breaking change detection
buf breaking --against '.git#branch=main'

# Protovalidate para validação de mensagens
buf lint
```

### Tipos de Teste

| Tipo | O que Testa | Ferramenta |
|------|------------|-----------|
| **Unit** | Handler/resolver isolado | Go testing, JUnit |
| **Integration** | Server + handler + DB | bufconn, testcontainers |
| **Contract** | Compatibilidade do schema | buf breaking |
| **E2E** | Fluxo completo | grpcurl, ghz |
| **Load** | Performance sob carga | ghz, k6 |

### Load Testing com ghz

```bash
# Install
go install github.com/bojand/ghz/cmd/ghz@latest

# Run load test
ghz --insecure \
    --proto ./proto/user/v1/user_service.proto \
    --call company.platform.user.v1.UserService.GetUser \
    --data '{"id": "123"}' \
    --concurrency 50 \
    --total 10000 \
    --timeout 5s \
    localhost:50051
```

---

## 16. Padrões de Arquitetura

### gRPC Gateway (REST ↔ gRPC)

```protobuf
import "google/api/annotations.proto";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {
    option (google.api.http) = {
      get: "/api/v1/users/{id}"
    };
  }

  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse) {
    option (google.api.http) = {
      post: "/api/v1/users"
      body: "*"
    };
  }

  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse) {
    option (google.api.http) = {
      patch: "/api/v1/users/{user.id}"
      body: "user"
    };
  }
}
```

```
         REST Clients                     gRPC Clients
              │                                │
              ▼                                │
    ┌──────────────────┐                      │
    │  gRPC Gateway    │                      │
    │  (REST → gRPC)   │                      │
    └────────┬─────────┘                      │
             │                                │
             ▼                                ▼
    ┌─────────────────────────────────────────────┐
    │              gRPC Server                     │
    │         (UserService, OrderService)          │
    └─────────────────────────────────────────────┘
```

### Service Mesh (Istio/Linkerd)

```
┌──────────────────────────────────────────────────┐
│                  Service Mesh                     │
│                                                  │
│  ┌────────────────┐      ┌────────────────┐     │
│  │  User Service  │      │ Order Service  │     │
│  │  ┌──────────┐  │ gRPC │  ┌──────────┐  │     │
│  │  │   App    │◀─┼──────┼─▶│   App    │  │     │
│  │  └────┬─────┘  │      │  └────┬─────┘  │     │
│  │  ┌────▼─────┐  │      │  ┌────▼─────┐  │     │
│  │  │  Sidecar │  │      │  │  Sidecar │  │     │
│  │  │ (Envoy)  │  │      │  │ (Envoy)  │  │     │
│  │  └──────────┘  │      │  └──────────┘  │     │
│  └────────────────┘      └────────────────┘     │
│                                                  │
│  mTLS automático | Load Balancing | Observability│
└──────────────────────────────────────────────────┘
```

### gRPC-Web para Browsers

```
Browser (gRPC-Web)
       │
       ▼
┌──────────────┐
│  Envoy Proxy │ ← Traduz gRPC-Web para gRPC nativo
│  (gRPC-Web)  │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ gRPC Server  │
└──────────────┘
```

### API Design Patterns (Google AIP)

```protobuf
// Seguir Google API Improvement Proposals (AIPs)

// AIP-131: Get
rpc GetUser(GetUserRequest) returns (User);

// AIP-132: List com paginação
rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);

message ListUsersRequest {
  int32 page_size = 1;       // Máximo de itens por página
  string page_token = 2;     // Token da próxima página
  string filter = 3;         // Filtro CEL (Common Expression Language)
  string order_by = 4;       // Ex: "created_at desc"
}

message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;
  int32 total_size = 3;
}

// AIP-133: Create
rpc CreateUser(CreateUserRequest) returns (User);

// AIP-134: Update com FieldMask
rpc UpdateUser(UpdateUserRequest) returns (User);

message UpdateUserRequest {
  User user = 1;
  google.protobuf.FieldMask update_mask = 2;  // Campos a atualizar
}

// AIP-135: Delete
rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);

// AIP-136: Custom methods
rpc ActivateUser(ActivateUserRequest) returns (User);  // POST /users/{id}:activate
```

---

## Referências

- [gRPC Official Documentation](https://grpc.io/docs/)
- [Protocol Buffers Language Guide (proto3)](https://protobuf.dev/programming-guides/proto3/)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [Google API Improvement Proposals (AIPs)](https://google.aip.dev/)
- [gRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md)
- [gRPC Performance Best Practices](https://grpc.io/docs/guides/performance/)
- [Buf — Schema Registry and Tooling](https://buf.build/)
- [gRPC-Web](https://grpc.io/docs/platforms/web/)
- [Envoy gRPC Bridge](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_protocols/grpc)
- [OpenTelemetry gRPC Instrumentation](https://opentelemetry.io/docs/instrumentation/)
- [ghz — gRPC Benchmarking Tool](https://ghz.sh/)

---

## 17. Graceful Shutdown

### Padrão de Shutdown (Go)

```go
func main() {
    lis, _ := net.Listen("tcp", ":50051")
    server := grpc.NewServer()
    pb.RegisterUserServiceServer(server, &UserServiceServer{})

    // Iniciar server em goroutine
    go func() {
        if err := server.Serve(lis); err != nil {
            log.Fatalf("failed to serve: %v", err)
        }
    }()

    // Aguardar sinal de shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("Shutting down gRPC server...")

    // GracefulStop aguarda RPCs em andamento terminarem
    // Novas conexões são recusadas imediatamente
    server.GracefulStop()

    log.Println("Server stopped")
}
```

### Kubernetes Graceful Shutdown

```yaml
spec:
  containers:
    - name: user-service
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]  # Tempo para LB remover o pod
  terminationGracePeriodSeconds: 30  # Tempo total para shutdown
```

### Sequência de Shutdown

```
1. Receber SIGTERM
2. Parar de aceitar novas conexões (health check → NOT_SERVING)
3. Aguardar drain period (5-10s) para LB remover o pod
4. GracefulStop() — aguardar RPCs em andamento terminarem
5. Se timeout (30s), forçar Stop()
6. Fechar conexões com dependências (DB, cache)
7. Exit
```

### Boas Práticas

| Prática | Detalhes |
|---------|---------|
| **Sempre use GracefulStop** | Nunca use `Stop()` diretamente — mata RPCs em andamento |
| **Drain period** | Aguarde 5-10s após sinalizar NOT_SERVING antes do GracefulStop |
| **Timeout** | Defina um timeout máximo para GracefulStop (ex: 25s) |
| **Health check update** | Marque como NOT_SERVING antes de parar |
| **Close dependencies** | Feche conexões com DB/cache após stop do server |

---

## 18. gRPC Reflection

### O que é?

gRPC Reflection permite que clients descubram os serviços e métodos disponíveis em runtime, sem precisar dos arquivos `.proto`. Útil para debugging e ferramentas como `grpcurl` e `grpcui`.

### Implementação (Go)

```go
import "google.golang.org/grpc/reflection"

func main() {
    server := grpc.NewServer()
    pb.RegisterUserServiceServer(server, &UserServiceServer{})

    // Habilitar reflection (apenas em dev/staging!)
    reflection.Register(server)

    server.Serve(lis)
}
```

### Implementação (Java)

```java
import io.grpc.protobuf.services.ProtoReflectionService;

Server server = ServerBuilder.forPort(50051)
    .addService(new UserServiceImpl())
    .addService(ProtoReflectionService.newInstance())  // Reflection
    .build()
    .start();
```

### Uso com grpcurl

```bash
# Listar serviços disponíveis
grpcurl -plaintext localhost:50051 list

# Descrever um serviço
grpcurl -plaintext localhost:50051 describe company.platform.user.v1.UserService

# Chamar um RPC
grpcurl -plaintext -d '{"id": "123"}' localhost:50051 company.platform.user.v1.UserService/GetUser
```

### Segurança

| Ambiente | Reflection |
|----------|:-----------:|
| Desenvolvimento | ✅ Habilitado |
| Staging | ✅ Habilitado (com autenticação) |
| Produção | ❌ Desabilitado (ou com acesso restrito) |

> **Atenção**: Reflection em produção expõe toda a API surface. Se necessário, habilite apenas para IPs internos ou com autenticação.
