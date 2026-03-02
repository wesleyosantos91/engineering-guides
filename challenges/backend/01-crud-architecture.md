# Level 1 — CRUD e Arquitetura Base

> **Objetivo:** Implementar os endpoints CRUD do domínio Digital Wallet com dados em memória, estabelecendo a arquitetura de camadas correta conforme os READMEs de referência.

---

## Objetivo de Aprendizado

- Implementar API REST completa com CRUD
- Aplicar a arquitetura de camadas definida no README de cada stack
- Implementar validação de entrada (Bean Validation / go-playground/validator)
- Implementar tratamento global de erros com ProblemDetail (RFC 9457)
- Implementar versionamento de API por path (`/api/v1/`)
- Entender o fluxo Handler/Controller → Service → Store/Repository

---

## Escopo Funcional

### Endpoints Obrigatórios

```
── Users ──
POST   /api/v1/users                     → Criar usuário (201)
GET    /api/v1/users/{id}                 → Buscar por ID (200 / 404)
GET    /api/v1/users                      → Listar todos (200) — sem paginação ainda
PUT    /api/v1/users/{id}                 → Atualizar (200 / 404)
DELETE /api/v1/users/{id}                 → Soft delete (204 / 404)

── Wallets ──
POST   /api/v1/users/{userId}/wallets     → Criar wallet (201)
GET    /api/v1/wallets/{id}               → Buscar wallet (200 / 404)
GET    /api/v1/users/{userId}/wallets     → Listar wallets do usuário (200)

── Transactions (básico) ──
POST   /api/v1/wallets/{walletId}/transactions/deposit    → Depósito (201)
POST   /api/v1/wallets/{walletId}/transactions/withdrawal → Saque (201 / 422)
GET    /api/v1/wallets/{walletId}/transactions             → Listar transações (200)
```

### Regras de Negócio (Level 1)

1. Email é único → 409 Conflict
2. Saldo nunca negativo → 422 ao sacar sem saldo
3. UUID como identificador em todas as entidades
4. Soft delete (marcar como INACTIVE, não remover)
5. Validação obrigatória: `name` (2-100 chars), `email` (formato válido), `amount` (> 0)

---

## Escopo Técnico

### Persistência: In-Memory (ainda sem banco)

Usar map/slice em memória como store. Isso permite focar exclusivamente na arquitetura HTTP.

### Por Stack

| Stack | Camada HTTP | Camada Serviço | Camada Store | DTOs |
|---|---|---|---|---|
| **Go (Gin)** | `internal/handler/user.go` | `internal/service/user.go` | `internal/store/user.go` (map) | `internal/model/user.go` (struct) |
| **Spring Boot** | `api/rest/v1/controller/UserController.java` | `domain/service/UserService.java` | `domain/repository/UserRepository.java` (Map) | `api/.../request/UserRequest.java` + `response/UserResponse.java` |
| **Quarkus** | `api/rest/v1/resource/UserResource.java` | `domain/service/UserService.java` | `domain/repository/UserRepository.java` (Map) | `api/.../request/UserRequest.java` + `response/UserResponse.java` |
| **Micronaut** | `api/rest/v1/controller/UserController.java` | `domain/service/UserService.java` | `domain/repository/UserRepository.java` (Map) | `api/.../request/UserRequest.java` + `response/UserResponse.java` |
| **Jakarta EE** | `api/rest/v1/resource/UserResource.java` | `domain/service/UserService.java` | `domain/repository/UserRepository.java` (Map) | `api/.../request/UserRequest.java` + `response/UserResponse.java` |

---

## Critérios de Aceite

- [ ] Todos os endpoints listados funcionam corretamente (verificar via curl/httpie/Insomnia)
- [ ] Validação de entrada retorna 400 com ProblemDetail JSON
- [ ] Recurso não encontrado retorna 404 com ProblemDetail JSON
- [ ] Email duplicado retorna 409 com ProblemDetail JSON
- [ ] Saque sem saldo retorna 422 com ProblemDetail JSON
- [ ] Respostas de sucesso seguem o formato padrão (JSON)
- [ ] UUIDs gerados automaticamente
- [ ] Soft delete funciona (GET após delete retorna 404 ou status INACTIVE)
- [ ] Logging básico em cada operação (info para sucesso, warn/error para falhas)

---

## Definição de Pronto (DoD)

- [ ] Código segue a estrutura de diretórios do README de referência
- [ ] Handler/Controller → Service → Store corretamente separados
- [ ] Nenhuma lógica de negócio no handler/controller
- [ ] Nenhum acesso a dados no service (apenas via store/repository)
- [ ] ProblemDetail (RFC 9457) implementado para todos os erros
- [ ] `go test` / `./mvnw test` passa (pelo menos 1 teste de smoke)
- [ ] Commit: `feat(level-1): add user/wallet/transaction CRUD endpoints`

---

## Checklist

### Go (Gin)
- [ ] `internal/handler/user.go` — handlers `Create`, `FindByID`, `FindAll`, `Update`, `Delete`
- [ ] `internal/handler/wallet.go` — handlers `Create`, `FindByID`, `FindByUserID`
- [ ] `internal/handler/transaction.go` — handlers `Deposit`, `Withdraw`, `FindByWalletID`
- [ ] `internal/service/user.go` — lógica de negócio + interface `UserStore`
- [ ] `internal/service/wallet.go` — lógica de negócio + interface `WalletStore`
- [ ] `internal/service/transaction.go` — lógica com validação de saldo
- [ ] `internal/store/user.go` — implementação in-memory com `sync.RWMutex`
- [ ] `internal/store/wallet.go` — implementação in-memory
- [ ] `internal/store/transaction.go` — implementação in-memory
- [ ] `internal/model/user.go` — `User`, `CreateUserInput`, `UpdateUserInput`, `UserOutput`
- [ ] `internal/model/wallet.go` — `Wallet`, `CreateWalletInput`, `WalletOutput`
- [ ] `internal/model/transaction.go` — `Transaction`, `DepositInput`, `WithdrawalInput`, `TransactionOutput`
- [ ] `internal/router/router.go` — registro de rotas sob `/api/v1/`
- [ ] `internal/middleware/recovery.go` — panic recovery → ProblemDetail
- [ ] `internal/middleware/requestid.go` — X-Request-ID
- [ ] `internal/middleware/logger.go` — log estruturado por request
- [ ] `internal/platform/httperr/problem.go` — ProblemDetail struct + helpers
- [ ] `internal/errs/errors.go` — `ErrNotFound`, `ErrConflict`, `ErrBadRequest`, `ErrInsufficientBalance`

### Java (Spring Boot)
- [ ] `api/rest/v1/controller/UserController.java` — record `@RestController`
- [ ] `api/rest/v1/controller/WalletController.java`
- [ ] `api/rest/v1/controller/TransactionController.java`
- [ ] `api/rest/v1/openapi/UserOpenApi.java` — interface com `@Tag`, `@Operation`
- [ ] `api/rest/v1/request/UserRequest.java` — record com `@NotBlank`, `@Email`
- [ ] `api/rest/v1/request/DepositRequest.java`
- [ ] `api/rest/v1/response/UserResponse.java` — record
- [ ] `api/rest/v1/response/WalletResponse.java`
- [ ] `api/rest/v1/response/TransactionResponse.java`
- [ ] `api/exception/ApiExceptionHandler.java` — `@RestControllerAdvice`
- [ ] `domain/service/UserService.java` — `@Service`
- [ ] `domain/service/WalletService.java`
- [ ] `domain/service/TransactionService.java`
- [ ] `domain/repository/UserRepository.java` — in-memory (ConcurrentHashMap)
- [ ] `domain/exception/ResourceNotFoundException.java`
- [ ] `domain/exception/BusinessException.java`
- [ ] `domain/exception/InsufficientBalanceException.java`
- [ ] `core/mapper/UserMapper.java` — MapStruct interface

### Quarkus
- [ ] Mesma estrutura do Spring, mas com `resource/` ao invés de `controller/`
- [ ] `@Path` + `@ApplicationScoped` ao invés de `@RestController` + `@Service`
- [ ] `ExceptionMapper<T>` ao invés de `@RestControllerAdvice`
- [ ] MapStruct com `componentModel = "cdi"`

### Micronaut
- [ ] `@Controller` + `@Singleton` ao invés de `@RestController` + `@Service`
- [ ] `ExceptionHandler<T, R>` ao invés de `@RestControllerAdvice`
- [ ] MapStruct com `componentModel = "jsr330"`
- [ ] Records com `@Introspected`

### Jakarta EE
- [ ] `@Path` + `@ApplicationScoped` (CDI puro)
- [ ] `ExceptionMapper<T>` (JAX-RS)
- [ ] MapStruct com `componentModel = "cdi"`
- [ ] `Response.status().entity().build()` ao invés de `ResponseEntity`

---

## Tarefas Sugeridas por Stack

### Go (Gin) — Detalhamento

```go
// cmd/api/main.go — Bootstrap mínimo
func main() {
    cfg, err := config.Load()
    if err != nil {
        log.Fatal(err)
    }

    // Stores (in-memory)
    userStore := store.NewInMemoryUserStore()
    walletStore := store.NewInMemoryWalletStore()
    txStore := store.NewInMemoryTransactionStore()

    // Services
    userSvc := service.NewUserService(userStore, logger)
    walletSvc := service.NewWalletService(walletStore, userStore, logger)
    txSvc := service.NewTransactionService(txStore, walletStore, logger)

    // Handlers
    userHandler := handler.NewUserHandler(userSvc)
    walletHandler := handler.NewWalletHandler(walletSvc)
    txHandler := handler.NewTransactionHandler(txSvc)

    // Router
    r := gin.New()
    r.Use(middleware.RequestID(), middleware.Logger(logger), middleware.Recovery(logger))
    router.Setup(r, userHandler, walletHandler, txHandler)

    r.Run(":" + cfg.Server.Port)
}
```

### Spring Boot — Detalhamento

```java
// Controller como record (conforme README)
@RestController
@RequestMapping("/api/v1/users")
public record UserController(UserService userService) implements UserOpenApi {

    @PostMapping
    @Override
    public ResponseEntity<UserResponse> create(@Valid @RequestBody UserRequest request) {
        UserResponse response = userService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping("/{id}")
    @Override
    public ResponseEntity<UserResponse> findById(@PathVariable UUID id) {
        return ResponseEntity.ok(userService.findById(id));
    }
}
```

---

## Extensões Opcionais

- [ ] Adicionar endpoint `PATCH /api/v1/users/{id}` para atualização parcial
- [ ] Implementar busca por email (`GET /api/v1/users?email=...`)
- [ ] Adicionar `Transfer` (transferência entre wallets) neste nível
- [ ] Implementar Swagger/OpenAPI docs
- [ ] Adicionar `X-Request-ID` correlation em logs

---

## Extensão: gRPC como Protocolo Alternativo

Implemente os mesmos endpoints do Level 1 usando **gRPC** como alternativa ao REST, para entender trade-offs entre os dois paradigmas.

### Por que gRPC?

| Aspecto | REST (JSON/HTTP) | gRPC (Protobuf/HTTP2) |
|---|---|---|
| **Formato** | JSON (texto) | Protobuf (binário) |
| **Performance** | Bom | ~7-10x mais rápido (serialização) |
| **Contract** | OpenAPI (opt-in) | `.proto` (obrigatório) |
| **Streaming** | SSE / WebSocket | Nativo (unary, server, client, bidi) |
| **Code gen** | Opcional | Obrigatório (protoc / buf) |
| **Ecossistema** | Universal (browsers, curl, etc) | Backends, microserviços |
| **Quando usar** | APIs públicas, CRUD | Comunicação interna entre serviços |

### Contrato Protobuf

```protobuf
// proto/wallet/v1/user.proto
syntax = "proto3";
package wallet.v1;

option go_package = "github.com/org/wallet/gen/wallet/v1";
option java_package = "com.novapay.wallet.grpc.v1";

service UserService {
  rpc CreateUser(CreateUserRequest) returns (UserResponse);
  rpc GetUser(GetUserRequest) returns (UserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc UpdateUser(UpdateUserRequest) returns (UserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  string document = 3;
}

message UserResponse {
  string id = 1;
  string name = 2;
  string email = 3;
  string document = 4;
  string status = 5;
  google.protobuf.Timestamp created_at = 6;
}

message ListUsersRequest {
  int32 page = 1;
  int32 size = 2;
  string sort = 3;
}

message ListUsersResponse {
  repeated UserResponse users = 1;
  int32 total_elements = 2;
  int32 total_pages = 3;
}
```

### Tooling por Stack

| Stack | gRPC Framework | Code Gen | Reflection |
|---|---|---|---|
| **Go** | `google.golang.org/grpc` | `buf generate` ou `protoc-gen-go-grpc` | `grpcurl` + reflection server |
| **Spring Boot** | `grpc-spring-boot-starter` (LogNet ou grpc-ecosystem) | `protobuf-maven-plugin` | Spring gRPC Server auto-config |
| **Quarkus** | `quarkus-grpc` | Build-time geração automática | SmallRye gRPC |
| **Micronaut** | `micronaut-grpc` | `protobuf-gradle-plugin` | Micronaut gRPC server |
| **Jakarta EE** | Manual (io.grpc + CDI beans) | `protoc` direto | — |

### Buf (Build Tool para Protobuf)

```yaml
# buf.yaml
version: v2
modules:
  - path: proto
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
```

```bash
# Gerar código Go + Java
buf generate

# Lint nos .proto
buf lint

# Verificar breaking changes
buf breaking --against 'https://github.com/org/wallet.git#branch=main'
```

> **Critérios de aceite (gRPC):**
> - [ ] Contratos `.proto` definidos para `UserService` (CRUD)
> - [ ] Servidor gRPC rodando em paralelo à API REST (porta diferente)
> - [ ] Testável via `grpcurl` ou `evans` (gRPC CLI)
> - [ ] Code generation via `buf` (Go) ou protobuf-maven-plugin (Java)
> - [ ] Breaking change detection configurado (`buf breaking`)

---

## Erros Comuns

| Erro | Stack | Como evitar |
|---|---|---|
| Lógica de negócio no handler/controller | Todos | Handler recebe, valida input, delega ao service, retorna output |
| Store/Repository acessa model de fora | Go | Store retorna `*model.User`, nunca `gin.Context` |
| Não usar `sync.RWMutex` no store in-memory | Go | Maps não são thread-safe — use `sync.RWMutex` ou `sync.Map` |
| Retornar entity diretamente no controller | Java | Sempre converter para Response DTO via mapper |
| Não usar record para controller | Java | Records garantem imutabilidade e DI por construtor |
| Misturar exceções HTTP no service | Java | Service lança exceções de domínio, handler traduz para HTTP |
| Esquecer `@Valid` no request body | Java | Sem `@Valid`, Bean Validation não é acionado |

---

## Como Isso Aparece em Entrevistas

- "Desenhe a arquitetura de um endpoint REST — quais camadas existem e qual a responsabilidade de cada uma?"
- "Por que separar controller/handler do service?"
- "O que é RFC 9457 (ProblemDetail) e por que usar?"
- "Como você garante que o email é único sem banco de dados?"
- "Como funciona a validação de entrada no Spring/Quarkus/Micronaut?"
- "Em Go, por que definir a interface do store no pacote do service?"
- "Qual a diferença entre resource (JAX-RS) e controller (Spring MVC)?"

---

## Comandos de Execução/Teste

### Go
```bash
# Rodar
GIN_MODE=debug go run ./cmd/api

# Testar endpoints
curl -X POST http://localhost:8080/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name":"João","email":"joao@test.com","document":"123.456.789-00"}'

curl http://localhost:8080/api/v1/users

# Testes
go test ./internal/... -v
```

### Spring Boot
```bash
# Rodar
./mvnw spring-boot:run -Dspring-boot.run.profiles=local

# Testar endpoints
curl -X POST http://localhost:8080/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name":"João","email":"joao@test.com","document":"123.456.789-00"}'

# Testes
./mvnw test
```

### Quarkus
```bash
# Rodar (Dev Mode com live reload)
./mvnw quarkus:dev

# Testes
./mvnw test
```

### Micronaut
```bash
# Rodar
./mvnw mn:run -Dmicronaut.environments=local

# Testes
./mvnw test
```

### Jakarta EE
```bash
# Build WAR
./mvnw package

# Deploy em WildFly (local)
# Copiar WAR para standalone/deployments/ ou usar WildFly Maven Plugin
./mvnw wildfly:deploy

# Testes
./mvnw test
```
