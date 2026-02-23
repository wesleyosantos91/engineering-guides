# Project Structure — Go (Gin)

> **Objetivo deste documento:** Servir como referência organizacional para o GitHub Copilot e desenvolvedores.
> Convenções seguem os padrões da comunidade Go — [Effective Go](https://go.dev/doc/effective_go),
> [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md),
> [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments) e práticas comuns
> em projetos de produção de empresas como Uber, Mercado Livre, Cloudflare, Datadog e HashiCorp.
>
> **Localização recomendada:** `.github/copilot-instructions.md` (veja [Onde colocar este arquivo](#onde-colocar-este-arquivo-para-o-copilot)).

---

## Sumário

- [Stack Tecnológica](#stack-tecnológica)
- [Estrutura de Diretórios](#estrutura-de-diretórios)
- [Organização dos Pacotes](#organização-dos-pacotes)
  - [handler — HTTP Handlers](#handler--http-handlers)
  - [service — Regras de Negócio](#service--regras-de-negócio)
  - [store — Persistência](#store--persistência)
  - [model — Structs de Domínio](#model--structs-de-domínio)
  - [middleware — Middlewares HTTP](#middleware--middlewares-http)
  - [platform — Infraestrutura Compartilhada](#platform--infraestrutura-compartilhada)
- [Pontos de Extensão](#pontos-de-extensão)
  - [API — REST / gRPC / GraphQL](#api--rest--grpc--graphql)
  - [Mensageria — Kafka / SQS / RabbitMQ](#mensageria--kafka--sqs--rabbitmq)
  - [Banco de Dados — MySQL / PostgreSQL / MongoDB](#banco-de-dados--mysql--postgresql--mongodb)
- [Convenções de Código](#convenções-de-código)
- [Configuração por Environment](#configuração-por-environment)
- [Banco de Dados e Migrations](#banco-de-dados-e-migrations)
- [Testes](#testes)
- [Build e Qualidade](#build-e-qualidade)
- [Resiliência](#resiliência)
- [Segurança (AuthN/AuthZ)](#segurança-authnauthz)
- [Tratamento Global de Erros](#tratamento-global-de-erros)
- [Versionamento de API](#versionamento-de-api)
- [Paginação e Ordenação](#paginação-e-ordenação)
- [CORS](#cors)
- [Feature Flags](#feature-flags)
- [Goroutines e Concorrência](#goroutines-e-concorrência)
- [Observabilidade](#observabilidade)
- [Docker](#docker)
- [Guia para Criar Novo Recurso](#guia-para-criar-novo-recurso)
- [Onde colocar este arquivo para o Copilot](#onde-colocar-este-arquivo-para-o-copilot)

---

## Stack Tecnológica

| Categoria           | Tecnologia                                                          |
|---------------------|---------------------------------------------------------------------|
| **Linguagem**       | Go 1.24+                                                            |
| **Framework HTTP**  | Gin (`github.com/gin-gonic/gin`)                                    |
| **Módulos**         | Go Modules (`go.mod`)                                                |
| Persistência (SQL)  | GORM (`gorm.io/gorm`) — *ou sqlx para projetos query-heavy*         |
| Migrations          | golang-migrate (`github.com/golang-migrate/migrate/v4`)             |
| Validação           | go-playground/validator (`github.com/go-playground/validator/v10`)   |
| Documentação API    | swaggo/swag (`github.com/swaggo/swag`) + gin-swagger                 |
| HTTP Client         | `net/http` stdlib — *ou go-resty para APIs REST complexas*           |
| Logging             | Zap (`go.uber.org/zap`) — *ou slog (stdlib Go 1.21+)*               |
| Métricas            | Prometheus (`github.com/prometheus/client_golang`)                   |
| Tracing             | OpenTelemetry (`go.opentelemetry.io/otel`)                           |
| DI                  | uber-go/fx (`go.uber.org/fx`) — *ou constructor injection manual*    |
| Testes Unitários    | `testing` (stdlib) + testify (`github.com/stretchr/testify`)         |
| Mocks               | mockery (`github.com/vektra/mockery`) ou testify mock                |
| Testes Integração   | testcontainers-go (`github.com/testcontainers/testcontainers-go`)    |
| Testes de Carga     | k6 (smoke, load, stress)                                             |
| Resiliência         | sony/gobreaker (`github.com/sony/gobreaker`)                         |
| Retry               | cenkalti/backoff (`github.com/cenkalti/backoff/v4`)                  |
| Segurança           | golang-jwt (`github.com/golang-jwt/jwt/v5`)                          |
| Feature Flags       | go-feature-flag (`github.com/thomaspoignant/go-feature-flag`)        |
| Lint                | golangci-lint (`github.com/golangci/golangci-lint`)                  |
| Vulnerabilidades    | govulncheck (`golang.org/x/vuln/cmd/govulncheck`)                   |

---

## Estrutura de Diretórios

```
app/
├── go.mod
├── go.sum
├── Dockerfile
├── docker-compose.yml
├── Makefile
├── .golangci.yml
│
├── cmd/
│   └── api/
│       └── main.go                    # entrypoint — bootstrap, DI, graceful shutdown
│
├── internal/                          # código privado do módulo (Go enforced)
│   ├── config/
│   │   └── config.go                  # struct Config + Load()
│   │
│   ├── handler/                       # HTTP handlers (por recurso)
│   │   ├── user.go                    # UserHandler — gin.HandlerFunc
│   │   ├── user_test.go
│   │   ├── order.go
│   │   └── order_test.go
│   │
│   ├── service/                       # lógica de negócio
│   │   ├── user.go                    # UserService
│   │   ├── user_test.go
│   │   ├── order.go
│   │   └── order_test.go
│   │
│   ├── store/                         # persistência (repositories)
│   │   ├── user.go                    # UserStore — interface + GORM impl
│   │   ├── user_test.go
│   │   ├── order.go
│   │   └── order_test.go
│   │
│   ├── model/                         # structs de domínio + DTOs
│   │   ├── user.go                    # User, CreateUserInput, UserOutput
│   │   └── order.go
│   │
│   ├── middleware/                     # Gin middlewares
│   │   ├── auth.go                    # JWT validation
│   │   ├── cors.go
│   │   ├── recovery.go                # panic recovery + ProblemDetail
│   │   ├── requestid.go
│   │   ├── logger.go
│   │   └── ratelimit.go
│   │
│   ├── router/                        # registro de rotas
│   │   └── router.go                  # SetupRouter() — agrupa todos os handlers
│   │
│   ├── platform/                      # infraestrutura técnica compartilhada
│   │   ├── database/
│   │   │   └── database.go            # NewDB() — conexão GORM
│   │   ├── httperr/
│   │   │   └── problem.go             # ProblemDetail (RFC 9457)
│   │   ├── pagination/
│   │   │   └── pagination.go          # Page[T], Params
│   │   ├── resilience/
│   │   │   ├── circuitbreaker.go
│   │   │   └── retry.go
│   │   ├── auth/
│   │   │   ├── jwt.go                 # JWT parser, claims
│   │   │   └── roles.go
│   │   ├── featureflag/
│   │   │   └── featureflag.go
│   │   ├── messaging/                 # kafka/ | sqs/ | rabbitmq/
│   │   │   └── kafka/
│   │   │       ├── producer.go
│   │   │       └── consumer.go
│   │   └── observe/
│   │       ├── metrics.go             # Prometheus
│   │       ├── tracing.go             # OpenTelemetry
│   │       └── health.go              # liveness + readiness
│   │
│   └── errs/                          # erros de domínio (sentinel + custom types)
│       └── errors.go
│
├── migrations/
│   ├── 000001_create_users.up.sql
│   ├── 000001_create_users.down.sql
│   └── testdata/                      # seeds (dev/test only)
│
├── docs/                              # gerado pelo swaggo
│   ├── docs.go
│   ├── swagger.json
│   └── swagger.yaml
│
└── test/
    ├── integration/
    │   ├── setup_test.go              # TestMain + testcontainers
    │   └── user_test.go
    └── fixture/
        └── user.go                    # builders e fábricas de dados de teste
```

> **Por que `internal/`?** Go **enforce** que pacotes dentro de `internal/` só podem ser importados pelo módulo pai. Isso substitui o `package-private` de outras linguagens.

> **Por que `store/` e não `repository/`?** O termo _store_ é mais idiomático em Go — veja projetos como Gitea, Grafana e Thanos. _Repository_ também é aceitável, mas evite o sufixo `Repository` em cada struct (o pacote já dá contexto).

> **Por que `platform/`?** Agrupa código de infraestrutura que não é lógica de negócio. Alternativas comuns: `infra/`, `pkg/` (quando exportável), `foundation/`.

---

## Organização dos Pacotes

```
internal/
├── config/        → Carregamento de config (env vars / YAML)
├── handler/       → Gin handlers — recebe HTTP, delega ao service, devolve JSON
├── service/       → Regras de negócio — orquestra stores, valida invariantes
├── store/         → Persistência — GORM queries, Redis, cache
├── model/         → Structs: entidades, DTOs (input/output), value objects
├── middleware/     → Middlewares Gin (auth, cors, logging, recovery, rate limit)
├── router/        → Registro de rotas em RouterGroup
├── platform/      → Infraestrutura técnica (db, metrics, tracing, messaging)
└── errs/          → Erros de domínio (sentinel vars + custom error types)
```

### handler — HTTP Handlers

| Convenção                                      | Exemplo                                    |
|------------------------------------------------|--------------------------------------------|
| Um arquivo por recurso                         | `user.go`, `order.go`                      |
| Struct com dependências injetadas              | `UserHandler` recebe `service.UserService` |
| Métodos são `gin.HandlerFunc`                  | `func (h *UserHandler) Create(c *gin.Context)` |
| Registra rotas no `router/`                    | `h.RegisterRoutes(rg *gin.RouterGroup)`    |
| Erros enviados via `c.Error(err)` + middleware | Não fazer `c.JSON(500, ...)` em cada handler |

```go
// internal/handler/user.go
package handler

type UserHandler struct {
    svc *service.UserService
}

func NewUserHandler(svc *service.UserService) *UserHandler {
    return &UserHandler{svc: svc}
}

func (h *UserHandler) Create(c *gin.Context) {
    var input model.CreateUserInput
    if err := c.ShouldBindJSON(&input); err != nil {
        _ = c.Error(err)
        return
    }

    user, err := h.svc.Create(c.Request.Context(), input)
    if err != nil {
        _ = c.Error(err)
        return
    }

    c.JSON(http.StatusCreated, user.ToOutput())
}

func (h *UserHandler) FindByID(c *gin.Context) {
    id, err := uuid.Parse(c.Param("id"))
    if err != nil {
        _ = c.Error(errs.NewBadRequest("invalid uuid"))
        return
    }

    user, err := h.svc.FindByID(c.Request.Context(), id)
    if err != nil {
        _ = c.Error(err)
        return
    }

    c.JSON(http.StatusOK, user.ToOutput())
}
```

> **Accept interfaces, return structs** (Uber Go Style Guide) — handlers recebem interfaces de service; services recebem interfaces de store.

### service — Regras de Negócio

| Convenção                                          | Exemplo                                   |
|----------------------------------------------------|-------------------------------------------|
| Struct recebe interfaces de dependência             | `UserService{ store UserStore }`          |
| Interface definida **no mesmo pacote que consome**  | `service/user.go` define `UserStore`      |
| Primeiro argumento é sempre `context.Context`       | `func (s *UserService) Create(ctx context.Context, ...)` |
| Retorna `(T, error)` — nunca pânico para erros de negócio | Erros do pacote `errs/`           |

```go
// internal/service/user.go
package service

// UserStore — interface definida no consumer (princípio Go: accept interfaces)
type UserStore interface {
    FindByID(ctx context.Context, id uuid.UUID) (*model.User, error)
    FindByEmail(ctx context.Context, email string) (*model.User, error)
    FindAll(ctx context.Context, params pagination.Params) (*pagination.Page[model.User], error)
    Create(ctx context.Context, user *model.User) error
    Update(ctx context.Context, user *model.User) error
    Delete(ctx context.Context, id uuid.UUID) error
}

type UserService struct {
    store UserStore
    log   *zap.Logger
}

func NewUserService(store UserStore, log *zap.Logger) *UserService {
    return &UserService{
        store: store,
        log:   log.Named("user_service"),
    }
}

func (s *UserService) Create(ctx context.Context, input model.CreateUserInput) (*model.User, error) {
    existing, err := s.store.FindByEmail(ctx, input.Email)
    if err != nil && !errors.Is(err, errs.ErrNotFound) {
        return nil, fmt.Errorf("checking existing user: %w", err)
    }
    if existing != nil {
        return nil, errs.NewConflict("email already registered")
    }

    user := input.ToUser()
    if err := s.store.Create(ctx, user); err != nil {
        return nil, fmt.Errorf("creating user: %w", err)
    }

    s.log.Info("user created", zap.String("id", user.ID.String()))
    return user, nil
}
```

> **Interface segregation:** Cada service define apenas as operações do store que ele precisa. Se `OrderService` só lê usuários, a interface dele pode ter apenas `FindByID`.

### store — Persistência

| Convenção                                    | Exemplo                                 |
|----------------------------------------------|-----------------------------------------|
| Um arquivo por recurso                       | `user.go`, `order.go`                   |
| Struct recebe `*gorm.DB`                     | `UserStore{ db *gorm.DB }`              |
| Construtor `New*Store`                       | `NewUserStore(db *gorm.DB)`             |
| Retorna erros de domínio (não erros do GORM) | `gorm.ErrRecordNotFound` → `errs.ErrNotFound` |

```go
// internal/store/user.go
package store

type UserStore struct {
    db *gorm.DB
}

func NewUserStore(db *gorm.DB) *UserStore {
    return &UserStore{db: db}
}

func (s *UserStore) FindByID(ctx context.Context, id uuid.UUID) (*model.User, error) {
    var user model.User
    err := s.db.WithContext(ctx).First(&user, "id = ?", id).Error
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, errs.ErrNotFound
    }
    if err != nil {
        return nil, fmt.Errorf("finding user by id: %w", err)
    }
    return &user, nil
}

func (s *UserStore) Create(ctx context.Context, user *model.User) error {
    if err := s.db.WithContext(ctx).Create(user).Error; err != nil {
        return fmt.Errorf("inserting user: %w", err)
    }
    return nil
}

func (s *UserStore) FindAll(ctx context.Context, params pagination.Params) (*pagination.Page[model.User], error) {
    var total int64
    s.db.WithContext(ctx).Model(&model.User{}).Count(&total)

    var users []model.User
    q := s.db.WithContext(ctx).Offset(params.Offset()).Limit(params.Size)
    if params.Sort != "" {
        q = q.Order(params.OrderClause())
    }
    if err := q.Find(&users).Error; err != nil {
        return nil, fmt.Errorf("listing users: %w", err)
    }

    return pagination.NewPage(users, params, total), nil
}
```

### model — Structs de Domínio

| Convenção                                | Exemplo                                  |
|------------------------------------------|------------------------------------------|
| **Sem sufixo `Entity`** — o pacote já dá contexto | `model.User`, não `model.UserEntity` |
| Input e Output no mesmo arquivo          | `CreateUserInput`, `UserOutput`          |
| Métodos de conversão **no model**        | `(u *User) ToOutput() UserOutput`        |
| Tags GORM para mapeamento               | `gorm:"column:name;type:varchar(100)"` |
| Tags JSON para serialização              | `json:"name"`                            |
| Tags binding para validação de input     | `binding:"required,email"`               |

```go
// internal/model/user.go
package model

// --- Entity (persistência) ---

type User struct {
    ID        uuid.UUID      `gorm:"type:char(36);primaryKey" json:"id"`
    Name      string         `gorm:"type:varchar(100);not null" json:"name"`
    Email     string         `gorm:"type:varchar(255);uniqueIndex;not null" json:"email"`
    CreatedAt time.Time      `gorm:"autoCreateTime" json:"created_at"`
    UpdatedAt time.Time      `gorm:"autoUpdateTime" json:"updated_at"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"-"`
}

func (User) TableName() string { return "users" }

// --- Input (request DTO) ---

type CreateUserInput struct {
    Name  string `json:"name"  binding:"required,min=2,max=100"`
    Email string `json:"email" binding:"required,email"`
}

func (i CreateUserInput) ToUser() *User {
    return &User{
        ID:    uuid.New(),
        Name:  i.Name,
        Email: i.Email,
    }
}

type UpdateUserInput struct {
    Name  *string `json:"name"  binding:"omitempty,min=2,max=100"`
    Email *string `json:"email" binding:"omitempty,email"`
}

// --- Output (response DTO) ---

type UserOutput struct {
    ID        uuid.UUID `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}

func (u *User) ToOutput() UserOutput {
    return UserOutput{
        ID:        u.ID,
        Name:      u.Name,
        Email:     u.Email,
        CreatedAt: u.CreatedAt,
    }
}

func ToUserOutputList(users []User) []UserOutput {
    out := make([]UserOutput, len(users))
    for i := range users {
        out[i] = users[i].ToOutput()
    }
    return out
}
```

> **Por que métodos de conversão no model e não em um pacote `mapper/`?**
> Em Go, co-localizar conversão com o tipo é idiomático. Mappers como pacote separado vêm do mundo Java — em Go, um método `ToOutput()` no model evita imports circulares e mantém o código próximo dos tipos que ele transforma.

> **Ponteiros para campos opcionais** (`*string`) — diferencia zero-value de campo ausente em updates parciais (PATCH).

### middleware — Middlewares HTTP

| Arquivo         | Responsabilidade                                      |
|-----------------|-------------------------------------------------------|
| `auth.go`       | Valida JWT Bearer, extrai claims, seta no context     |
| `cors.go`       | Configura allowed origins / methods / headers          |
| `recovery.go`   | Captura panics, retorna ProblemDetail 500              |
| `requestid.go`  | Gera ou propaga `X-Request-ID`                         |
| `logger.go`     | Log estruturado de cada request (method, path, status) |
| `ratelimit.go`  | Rate limiting por IP ou token                          |

### platform — Infraestrutura Compartilhada

| Subpacote          | Responsabilidade                                     |
|--------------------|------------------------------------------------------|
| `database/`        | `NewDB()` — abre conexão GORM, configura pool        |
| `httperr/`         | `ProblemDetail` struct (RFC 9457) + helpers           |
| `pagination/`      | `Params`, `Page[T]` (generics), `NewPage()`          |
| `resilience/`      | Factory de circuit breaker + retry helpers            |
| `auth/`            | JWT parsing, claims struct, JWKS key func             |
| `featureflag/`     | Setup go-feature-flag, constantes de flags            |
| `messaging/kafka/` | Producer e Consumer wrappers sobre kafka-go           |
| `observe/`         | Prometheus middleware, OTEL tracing, health checks    |

---

## Pontos de Extensão

### API — REST / gRPC / GraphQL

A camada `handler/` é protocolo-específica. Para adicionar gRPC ou GraphQL, crie pacotes paralelos:

```
internal/
├── handler/          # ── REST (Gin) ──
├── grpc/             # ── gRPC ──
│   ├── interceptor/
│   ├── server/
│   └── proto/
└── graphql/          # ── GraphQL ──
    ├── resolver/
    ├── model/
    └── schema/
```

| Protocolo  | Dependência                                             |
|------------|---------------------------------------------------------|
| REST       | `github.com/gin-gonic/gin`                              |
| gRPC       | `google.golang.org/grpc` + `protoc-gen-go-grpc`         |
| GraphQL    | `github.com/99designs/gqlgen`                            |

**Fluxo — independente do protocolo:**
1. Handler/resolver recebe request
2. Converte para model (input)
3. Delega ao `service/`
4. Converte e retorna output

### Mensageria — Kafka / SQS / RabbitMQ

```
internal/
├── handler/               # REST handlers
├── consumer/              # event consumers (paralelo ao handler/)
│   ├── transaction.go     # consome eventos de transação
│   └── notification.go
│
└── platform/messaging/
    ├── kafka/
    │   ├── producer.go    # Producer wrapper genérico
    │   └── consumer.go    # ConsumerGroup setup
    ├── sqs/
    │   └── ...
    └── rabbitmq/
        └── ...
```

| Broker     | Dependência                                               |
|------------|-----------------------------------------------------------|
| Kafka      | `github.com/segmentio/kafka-go`                           |
| SQS        | `github.com/aws/aws-sdk-go-v2/service/sqs`               |
| RabbitMQ   | `github.com/rabbitmq/amqp091-go`                          |

**Padrão de consumer:**

```go
// internal/consumer/transaction.go
type TransactionConsumer struct {
    reader *kafka.Reader
    svc    *service.TransactionService
    log    *zap.Logger
}

func (c *TransactionConsumer) Run(ctx context.Context) error {
    for {
        msg, err := c.reader.ReadMessage(ctx)
        if err != nil {
            if errors.Is(err, context.Canceled) {
                return nil
            }
            c.log.Error("read message failed", zap.Error(err))
            continue
        }

        var event model.TransactionCreatedEvent
        if err := json.Unmarshal(msg.Value, &event); err != nil {
            c.log.Error("unmarshal event", zap.Error(err))
            continue
        }

        if err := c.svc.Process(ctx, event); err != nil {
            c.log.Error("process event", zap.Error(err), zap.String("key", string(msg.Key)))
        }
    }
}
```

**Padrão de producer:**

```go
func (p *NotificationProducer) Send(ctx context.Context, event model.NotificationEvent) error {
    data, err := json.Marshal(event)
    if err != nil {
        return fmt.Errorf("marshal notification event: %w", err)
    }
    return p.writer.WriteMessages(ctx, kafka.Message{
        Key:   []byte(event.UserID.String()),
        Value: data,
    })
}
```

### Banco de Dados — MySQL / PostgreSQL / MongoDB

O `store/` é o pacote de persistência. Se múltiplos bancos são necessários, crie subpacotes:

```
internal/store/
├── user.go                # interface UserStore (se precisar abstrair)
├── pguser.go              # PostgreSQL impl
├── mongouser.go           # MongoDB impl
└── mysql/                 # ou agrupe por tecnologia se muitos recursos
    ├── user.go
    └── order.go
```

| Banco       | Dependência                          | Conexão                        |
|-------------|--------------------------------------|--------------------------------|
| MySQL       | `gorm.io/driver/mysql`               | `mysql.Open(dsn)`              |
| PostgreSQL  | `gorm.io/driver/postgres`            | `postgres.Open(dsn)`           |
| MongoDB     | `go.mongodb.org/mongo-driver`         | `mongo.Connect()` (driver nativo) |

> **Projeto com apenas um banco (caso mais comum):** Coloque a implementação direto no `store/` sem subpacote. Crie subpacote somente se houver mais de uma tecnologia de banco.

---

## Convenções de Código

### Nomeação (Uber Go Style Guide + Effective Go)

| Elemento         | Convenção                                           | Exemplo                          |
|------------------|-----------------------------------------------------|----------------------------------|
| Pacote           | `lowercase`, singular, curto, **sem underscore**     | `handler`, `store`, `model`      |
| Arquivo          | `snake_case.go`                                     | `user.go`, `order_test.go`       |
| Struct           | `PascalCase`                                        | `UserHandler`, `User`            |
| Interface        | Verbos `-er` para 1 método; descritivo para múltiplos | `Reader`, `UserStore`           |
| Exported func    | `PascalCase`                                        | `NewUserService()`               |
| Unexported       | `camelCase`                                         | `parseToken()`, `maxRetries`     |
| Constante        | `PascalCase` (exported) ou `camelCase` (unexported) | `MaxPageSize`, `defaultTimeout`  |
| Erro sentinel    | `Err` prefix                                        | `ErrNotFound`, `ErrUnauthorized` |
| Acrônimos        | All caps se exported                                | `HTTPClient`, `JSONAPI`, `ID`    |
| Tabela DB        | `snake_case`, plural                                | `users`, `order_items`           |
| Coluna DB        | `snake_case`                                        | `created_at`, `user_id`          |
| Migration        | `{seq}_{descricao}.{up\|down}.sql`                  | `000001_create_users.up.sql`     |

### Anti-stuttering (Uber Go Style Guide)

O nome do pacote é parte da chamada — não repita:

```go
// ✅ Correto
store.NewUser(db)       // pacote store, struct User
model.User{}            // pacote model, struct User
handler.NewUser(svc)    // pacote handler, struct User

// ❌ Errado (stutter)
store.NewUserStore(db)  // "store.UserStore" repete "store"
model.UserModel{}       // "model.UserModel" repete "model"
```

> **Exceção prática:** Quando há risco real de colisão de nomes ou quando o projeto é grande o suficiente para justificar clareza extra, o sufixo é aceitável. O importante é ser **consistente** no projeto inteiro.

### Error Wrapping (Go 1.13+)

```go
// ✅ Sempre adicionar contexto ao wrappear
if err != nil {
    return fmt.Errorf("creating user: %w", err)
}

// ✅ Verificar com errors.Is / errors.As
if errors.Is(err, errs.ErrNotFound) { ... }

var conflict *errs.ConflictError
if errors.As(err, &conflict) { ... }

// ❌ Nunca ignorar erro silenciosamente
result, _ := doSomething() // nope
```

### Functional Options (Uber Go Style Guide)

Para structs com configuração complexa:

```go
type Server struct {
    port    string
    timeout time.Duration
    logger  *zap.Logger
}

type Option func(*Server)

func WithPort(port string) Option {
    return func(s *Server) { s.port = port }
}

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func NewServer(opts ...Option) *Server {
    s := &Server{
        port:    "8080",
        timeout: 30 * time.Second,
    }
    for _, o := range opts {
        o(s)
    }
    return s
}
```

### Struct Initialization (Uber Go Style Guide)

```go
// ✅ Usar field names (não posicional)
user := model.User{
    Name:  "Alice",
    Email: "alice@example.com",
}

// ❌ Evitar inicialização posicional
user := model.User{"Alice", "alice@example.com"} // frágil
```

### Context Propagation

```go
// ✅ Sempre primeiro parâmetro, nunca em struct
func (s *UserService) Create(ctx context.Context, input model.CreateUserInput) (*model.User, error)

// ❌ Nunca armazene context em struct
type Bad struct {
    ctx context.Context // errado
}
```

---

## Configuração por Environment

### Padrão: Struct + env vars (caarlos0/env ou envconfig)

```go
// internal/config/config.go
package config

import "github.com/caarlos0/env/v11"

type Config struct {
    Server   Server
    Database Database
    JWT      JWT
    CORS     CORS
    Log      Log
}

type Server struct {
    Port string `env:"SERVER_PORT" envDefault:"8080"`
    Mode string `env:"GIN_MODE"   envDefault:"release"`
}

type Database struct {
    DSN             string        `env:"DATABASE_DSN,required"`
    MaxOpenConns    int           `env:"DB_MAX_OPEN_CONNS"    envDefault:"25"`
    MaxIdleConns    int           `env:"DB_MAX_IDLE_CONNS"    envDefault:"10"`
    ConnMaxLifetime time.Duration `env:"DB_CONN_MAX_LIFETIME" envDefault:"5m"`
}

type JWT struct {
    IssuerURL string `env:"JWT_ISSUER_URI,required"`
    JWKSURL   string `env:"JWK_SET_URI,required"`
}

type CORS struct {
    AllowedOrigins []string `env:"CORS_ALLOWED_ORIGINS" envSeparator:"," envDefault:"*"`
}

type Log struct {
    Level string `env:"LOG_LEVEL" envDefault:"info"`
}

func Load() (*Config, error) {
    cfg := &Config{}
    if err := env.Parse(cfg); err != nil {
        return nil, fmt.Errorf("parsing config: %w", err)
    }
    return cfg, nil
}
```

> **Por que env vars e não YAML?** É o padrão 12-factor: env vars são o mecanismo mais portável e seguro para configuração em containers e CI/CD. Se precisar de YAML, use Viper.

### Environments

| Environment  | `GIN_MODE` | Uso                                             |
|--------------|------------|--------------------------------------------------|
| production   | `release`  | Produção — configs via env vars / secrets manager |
| development  | `debug`    | Dev local — logs verbose, Swagger ativo           |
| test         | `test`     | Testes — Testcontainers injeta configs             |

---

## Banco de Dados e Migrations

### golang-migrate

Migrations em SQL puro, versionadas em `migrations/`.

| Tipo     | Diretório          | Formato                           | Exemplo                          |
|----------|--------------------|-----------------------------------|----------------------------------|
| DDL up   | `migrations/`      | `{seq}_{descricao}.up.sql`        | `000001_create_users.up.sql`     |
| DDL down | `migrations/`      | `{seq}_{descricao}.down.sql`      | `000001_create_users.down.sql`   |
| Seeds    | `migrations/testdata/` | `{seq}_seed_{tabela}.up.sql`  | `000001_seed_users.up.sql`       |

### Comandos

```bash
# Aplicar todas
migrate -path migrations -database "$DATABASE_DSN" up

# Rollback última
migrate -path migrations -database "$DATABASE_DSN" down 1

# Criar nova migration
migrate create -ext sql -dir migrations -seq create_orders
```

### Integração no bootstrap

```go
func RunMigrations(dsn string) error {
    m, err := migrate.New("file://migrations", dsn)
    if err != nil {
        return fmt.Errorf("init migrate: %w", err)
    }
    if err := m.Up(); err != nil && !errors.Is(err, migrate.ErrNoChange) {
        return fmt.Errorf("run migrations: %w", err)
    }
    return nil
}
```

> **Nunca use `db.AutoMigrate()`** — Migrations SQL explícitas são o único mecanismo para alterar schema. AutoMigrate não suporta rollback, não é revisável em code review e causa surpresas em produção.

---

## Testes

### Pirâmide de Testes

| Tipo         | Onde                               | Sufixo       | Runner                        |
|--------------|-------------------------------------|-------------|-------------------------------|
| Unitário     | Junto ao código (`*_test.go`)       | `_test.go`  | `go test ./internal/...`      |
| Integração   | `test/integration/`                 | `_test.go`  | `go test -tags=integration`   |
| Carga        | `scripts/k6/`                       | `.js`       | k6 (externo)                  |

### Convenções Go para Testes

- **Table-driven tests** — padrão Go para variações de cenário.
- **Subtests com `t.Run()`** — cada caso roda isolado, visível no output.
- **testify/assert** para asserções, **testify/require** para precondições fatais.
- **Fixture builders** no `test/fixture/` — funções `NewUser()`, `NewOrder()`.
- **`httptest.NewRecorder()`** para testes de handler sem servidor real.
- **testcontainers-go** para integração com banco real.
- **`t.Parallel()`** em testes que não compartilham estado.

### Estrutura de Teste Table-Driven

```go
func TestUserService_Create(t *testing.T) {
    tests := []struct {
        name    string
        input   model.CreateUserInput
        setup   func(s *mocks.UserStore)
        wantErr error
    }{
        {
            name:  "success",
            input: fixture.NewCreateUserInput(),
            setup: func(s *mocks.UserStore) {
                s.On("FindByEmail", mock.Anything, "alice@test.com").Return(nil, errs.ErrNotFound)
                s.On("Create", mock.Anything, mock.Anything).Return(nil)
            },
            wantErr: nil,
        },
        {
            name:  "duplicate email",
            input: fixture.NewCreateUserInput(),
            setup: func(s *mocks.UserStore) {
                s.On("FindByEmail", mock.Anything, "alice@test.com").Return(&model.User{}, nil)
            },
            wantErr: errs.ErrConflict,
        },
        {
            name:  "store error",
            input: fixture.NewCreateUserInput(),
            setup: func(s *mocks.UserStore) {
                s.On("FindByEmail", mock.Anything, "alice@test.com").Return(nil, errs.ErrNotFound)
                s.On("Create", mock.Anything, mock.Anything).Return(errors.New("db error"))
            },
            wantErr: cmpopts.AnyError,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            store := new(mocks.UserStore)
            tt.setup(store)

            svc := service.NewUserService(store, zap.NewNop())
            _, err := svc.Create(context.Background(), tt.input)

            if tt.wantErr != nil {
                require.ErrorIs(t, err, tt.wantErr)
            } else {
                require.NoError(t, err)
            }
            store.AssertExpectations(t)
        })
    }
}
```

### Teste de Handler com httptest

```go
func TestUserHandler_Create(t *testing.T) {
    svc := new(mocks.UserService)
    h := handler.NewUserHandler(svc)

    router := gin.New()
    router.POST("/users", h.Create)

    body := `{"name":"Alice","email":"alice@test.com"}`
    req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()

    svc.On("Create", mock.Anything, mock.Anything).Return(&model.User{
        ID: uuid.New(), Name: "Alice", Email: "alice@test.com",
    }, nil)

    router.ServeHTTP(w, req)

    assert.Equal(t, http.StatusCreated, w.Code)
    svc.AssertExpectations(t)
}
```

### Teste de Integração com Testcontainers

```go
//go:build integration

package integration

var testDB *gorm.DB

func TestMain(m *testing.M) {
    ctx := context.Background()

    mysql, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image:        "mysql:8.4",
            ExposedPorts: []string{"3306/tcp"},
            Env: map[string]string{
                "MYSQL_ROOT_PASSWORD": "test",
                "MYSQL_DATABASE":      "testdb",
            },
            WaitingFor: wait.ForListeningPort("3306/tcp").WithStartupTimeout(60 * time.Second),
        },
        Started: true,
    })
    if err != nil {
        log.Fatal(err)
    }
    defer mysql.Terminate(ctx)

    host, _ := mysql.Host(ctx)
    port, _ := mysql.MappedPort(ctx, "3306")
    dsn := fmt.Sprintf("root:test@tcp(%s:%s)/testdb?parseTime=true", host, port.Port())

    testDB, _ = gorm.Open(mysqlDriver.Open(dsn))
    database.RunMigrations(dsn)

    os.Exit(m.Run())
}
```

### Comandos

```bash
# Unitários
go test ./internal/...

# Com cobertura
go test ./internal/... -coverprofile=coverage.out && go tool cover -html=coverage.out

# Integração
go test -tags=integration ./test/integration/...

# Race detector
go test -race ./...
```

---

## Build e Qualidade

| Ferramenta        | Função                              | Comando                          |
|-------------------|-------------------------------------|----------------------------------|
| `go build`        | Compila binário estático            | `go build -o bin/api ./cmd/api`  |
| `go test`         | Roda testes                         | `go test ./...`                  |
| `golangci-lint`   | Linting (50+ linters configuráveis) | `golangci-lint run`              |
| `govulncheck`     | CVEs em dependências                | `govulncheck ./...`              |
| `go vet`          | Análise estática do Go toolchain    | `go vet ./...`                   |
| `swag init`       | Gera docs Swagger                   | `swag init -g cmd/api/main.go`   |

### Makefile

```makefile
.PHONY: build test lint swagger run

build:
	CGO_ENABLED=0 go build -ldflags="-s -w" -o bin/api ./cmd/api

run:
	GIN_MODE=debug go run ./cmd/api

test:
	go test -race ./internal/... -coverprofile=coverage.out

test-integration:
	go test -tags=integration -race ./test/integration/...

lint:
	golangci-lint run ./...

vet:
	go vet ./...

vuln:
	govulncheck ./...

swagger:
	swag init -g cmd/api/main.go -o docs

check: lint vet vuln test ## CI quality gate
```

### golangci-lint config mínimo

```yaml
# .golangci.yml
linters:
  enable:
    - errcheck
    - govet
    - staticcheck
    - unused
    - gosimple
    - ineffassign
    - typecheck
    - gofmt
    - goimports
    - misspell
    - prealloc
    - revive
    - gocritic
    - nilerr
    - errorlint     # verifica error wrapping correto
    - wrapcheck     # garante que erros de libs externas são wrapped
```

---

## Resiliência

### Padrões

| Padrão             | Biblioteca/Stdlib            | Quando usar                                       |
|--------------------|------------------------------|---------------------------------------------------|
| **Circuit Breaker** | sony/gobreaker              | Chamadas a serviços externos instáveis             |
| **Retry**          | cenkalti/backoff             | Falhas transitórias (timeout, 503, network)        |
| **Timeout**        | `context.WithTimeout`        | Deadline em qualquer chamada que pode bloquear     |
| **Rate Limiter**   | `golang.org/x/time/rate`     | Proteger endpoints contra overload                 |
| **Bulkhead**       | Channel semaphore            | Isolar chamadas e limitar concorrência             |

### Organização

```
internal/platform/resilience/
├── circuitbreaker.go       # NewCircuitBreaker() factory
└── retry.go                # Retry() com backoff exponencial
```

### Circuit Breaker

```go
// internal/platform/resilience/circuitbreaker.go
package resilience

func NewCircuitBreaker(name string) *gobreaker.CircuitBreaker {
    return gobreaker.NewCircuitBreaker(gobreaker.Settings{
        Name:        name,
        MaxRequests: 3,
        Interval:    10 * time.Second,
        Timeout:     30 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            return counts.Requests >= 10 &&
                float64(counts.TotalFailures)/float64(counts.Requests) >= 0.5
        },
        OnStateChange: func(name string, from, to gobreaker.State) {
            zap.L().Warn("circuit breaker state change",
                zap.String("name", name),
                zap.String("from", from.String()),
                zap.String("to", to.String()),
            )
        },
    })
}
```

### Retry com Backoff

```go
func Retry(ctx context.Context, op func() error) error {
    bo := backoff.NewExponentialBackOff()
    bo.MaxElapsedTime = 10 * time.Second
    bo.InitialInterval = 500 * time.Millisecond
    bo.MaxInterval = 5 * time.Second
    bo.Multiplier = 2.0

    return backoff.Retry(op, backoff.WithContext(bo, ctx))
}
```

### Uso no Service

```go
type PaymentService struct {
    client *PaymentClient
    cb     *gobreaker.CircuitBreaker
    log    *zap.Logger
}

func (s *PaymentService) Charge(ctx context.Context, req model.ChargeInput) (*model.ChargeResult, error) {
    result, err := s.cb.Execute(func() (interface{}, error) {
        ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
        defer cancel()
        return s.client.Charge(ctx, req)
    })
    if err != nil {
        s.log.Warn("payment charge failed, using fallback", zap.Error(err))
        return &model.ChargeResult{Status: "unavailable"}, nil
    }
    return result.(*model.ChargeResult), nil
}
```

---

## Segurança (AuthN/AuthZ)

### JWT Middleware

```
internal/platform/auth/
├── jwt.go        # ParseToken(), NewJWKSKeyFunc()
├── claims.go     # Claims struct
└── roles.go      # constantes de roles

internal/middleware/
└── auth.go       # AuthMiddleware(), RequireRoles()
```

### Claims

```go
// internal/platform/auth/claims.go
type Claims struct {
    jwt.RegisteredClaims
    Roles []string `json:"roles"`
    Email string   `json:"email"`
}
```

### Auth Middleware

```go
// internal/middleware/auth.go
func Auth(jwtCfg config.JWT) gin.HandlerFunc {
    keyFunc := auth.NewJWKSKeyFunc(jwtCfg.JWKSURL)

    return func(c *gin.Context) {
        token := extractBearer(c)
        if token == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, httperr.Problem{
                Status: 401,
                Title:  "Unauthorized",
                Detail: "Bearer token is required",
            })
            return
        }

        parsed, err := jwt.ParseWithClaims(token, &auth.Claims{}, keyFunc,
            jwt.WithIssuer(jwtCfg.IssuerURL),
            jwt.WithValidMethods([]string{"RS256"}),
        )
        if err != nil || !parsed.Valid {
            c.AbortWithStatusJSON(http.StatusUnauthorized, httperr.Problem{
                Status: 401,
                Title:  "Unauthorized",
                Detail: "Invalid or expired token",
            })
            return
        }

        claims := parsed.Claims.(*auth.Claims)
        c.Set("claims", claims)
        c.Set("userID", claims.Subject)
        c.Next()
    }
}

func extractBearer(c *gin.Context) string {
    h := c.GetHeader("Authorization")
    if len(h) > 7 && h[:7] == "Bearer " {
        return h[7:]
    }
    return ""
}
```

### RBAC Middleware

```go
func RequireRoles(roles ...string) gin.HandlerFunc {
    allowed := make(map[string]struct{}, len(roles))
    for _, r := range roles {
        allowed[r] = struct{}{}
    }

    return func(c *gin.Context) {
        claims, ok := c.Get("claims")
        if !ok {
            c.AbortWithStatusJSON(http.StatusForbidden, httperr.Problem{
                Status: 403, Title: "Forbidden", Detail: "No claims in context",
            })
            return
        }

        for _, role := range claims.(*auth.Claims).Roles {
            if _, ok := allowed[role]; ok {
                c.Next()
                return
            }
        }

        c.AbortWithStatusJSON(http.StatusForbidden, httperr.Problem{
            Status: 403,
            Title:  "Forbidden",
            Detail: fmt.Sprintf("Required roles: %v", roles),
        })
    }
}
```

### Registro de Rotas

```go
// internal/router/router.go
func Setup(engine *gin.Engine, cfg *config.Config, h *handler.UserHandler) {
    v1 := engine.Group("/v1")
    v1.Use(middleware.Auth(cfg.JWT))
    {
        users := v1.Group("/users")
        users.GET("",     middleware.RequireRoles("admin", "manager"), h.FindAll)
        users.GET("/:id", h.FindByID)
        users.POST("",    h.Create)
        users.PUT("/:id", h.Update)
        users.DELETE("/:id", middleware.RequireRoles("admin"), h.Delete)
    }
}
```

---

## Tratamento Global de Erros

### Padrão: ProblemDetail (RFC 9457) via Error Middleware

### Erros de Domínio

```go
// internal/errs/errors.go
package errs

import "errors"

// Sentinel errors — use com errors.Is()
var (
    ErrNotFound     = errors.New("not found")
    ErrConflict     = errors.New("conflict")
    ErrUnauthorized = errors.New("unauthorized")
    ErrForbidden    = errors.New("forbidden")
)

// Typed errors — use com errors.As() quando precisar de dados extras
type BadRequestError struct {
    Message string
}

func (e *BadRequestError) Error() string { return e.Message }

func NewBadRequest(msg string) *BadRequestError {
    return &BadRequestError{Message: msg}
}

type BusinessError struct {
    Code    string
    Message string
}

func (e *BusinessError) Error() string { return e.Message }

func NewBusinessError(code, msg string) *BusinessError {
    return &BusinessError{Code: code, Message: msg}
}
```

### ProblemDetail Struct

```go
// internal/platform/httperr/problem.go
package httperr

type Problem struct {
    Type      string `json:"type"`
    Title     string `json:"title"`
    Status    int    `json:"status"`
    Detail    string `json:"detail"`
    Instance  string `json:"instance,omitempty"`
    ErrorCode string `json:"error_code,omitempty"`
    Timestamp string `json:"timestamp"`
}

func New(status int, title, detail string) Problem {
    return Problem{
        Type:      fmt.Sprintf("https://api.example.com/errors/%d", status),
        Title:     title,
        Status:    status,
        Detail:    detail,
        Timestamp: time.Now().UTC().Format(time.RFC3339),
    }
}
```

### Error Middleware

```go
// internal/middleware/recovery.go
func Recovery(log *zap.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if r := recover(); r != nil {
                log.Error("panic recovered",
                    zap.Any("panic", r),
                    zap.String("path", c.Request.URL.Path),
                )
                c.AbortWithStatusJSON(http.StatusInternalServerError,
                    httperr.New(500, "Internal Server Error", "An unexpected error occurred"))
            }
        }()

        c.Next()

        if len(c.Errors) == 0 {
            return
        }

        err := c.Errors.Last().Err
        problem := mapError(err, c.Request.URL.Path)
        if problem.Status >= 500 {
            log.Error("unhandled error", zap.Error(err), zap.String("path", c.Request.URL.Path))
        }
        c.JSON(problem.Status, problem)
    }
}

func mapError(err error, instance string) httperr.Problem {
    var (
        badReq   *errs.BadRequestError
        business *errs.BusinessError
        valErr   validator.ValidationErrors
    )

    switch {
    case errors.Is(err, errs.ErrNotFound):
        return httperr.Problem{Status: 404, Title: "Not Found", Detail: err.Error(), Instance: instance,
            Timestamp: time.Now().UTC().Format(time.RFC3339)}
    case errors.Is(err, errs.ErrConflict):
        return httperr.Problem{Status: 409, Title: "Conflict", Detail: err.Error(), Instance: instance,
            Timestamp: time.Now().UTC().Format(time.RFC3339)}
    case errors.As(err, &badReq):
        return httperr.Problem{Status: 400, Title: "Bad Request", Detail: badReq.Message, Instance: instance,
            Timestamp: time.Now().UTC().Format(time.RFC3339)}
    case errors.As(err, &business):
        return httperr.Problem{Status: 422, Title: "Unprocessable Entity", Detail: business.Message,
            ErrorCode: business.Code, Instance: instance, Timestamp: time.Now().UTC().Format(time.RFC3339)}
    case errors.As(err, &valErr):
        return httperr.Problem{Status: 400, Title: "Validation Error", Detail: formatValidation(valErr),
            Instance: instance, Timestamp: time.Now().UTC().Format(time.RFC3339)}
    default:
        return httperr.Problem{Status: 500, Title: "Internal Server Error",
            Detail: "An unexpected error occurred", Instance: instance,
            Timestamp: time.Now().UTC().Format(time.RFC3339)}
    }
}
```

### Mapeamento de Erros

| Erro                     | Status | Quando                                 |
|--------------------------|--------|----------------------------------------|
| `errs.ErrNotFound`       | 404    | Recurso não encontrado no store        |
| `errs.ErrConflict`       | 409    | Duplicata (e-mail, código único)       |
| `*errs.BadRequestError`  | 400    | Input inválido (UUID malformado, etc.) |
| `*errs.BusinessError`    | 422    | Violação de regra de negócio           |
| `validator.ValidationErrors` | 400 | Binding/validação de request payload  |
| Auth middleware           | 401    | Token ausente ou inválido              |
| RBAC middleware           | 403    | Role insuficiente                      |
| `error` genérico          | 500    | Tudo que não foi tratado acima         |

---

## Versionamento de API

### Estratégia: Path-based

```
/v1/users     → versão 1
/v2/users     → versão 2
```

### Organização

Para a maioria dos projetos, versionamento é feito no router:

```go
func Setup(engine *gin.Engine, v1Handlers, v2Handlers *Handlers) {
    v1 := engine.Group("/v1")
    {
        v1.GET("/users",     v1Handlers.User.FindAll)
        v1.POST("/users",    v1Handlers.User.Create)
    }

    v2 := engine.Group("/v2")
    {
        v2.GET("/users",     v2Handlers.User.FindAll) // response diferente
        v2.POST("/users",    v2Handlers.User.Create)
    }
}
```

Se as versões divergem muito, crie subpacotes no handler:

```
internal/handler/
├── v1/
│   ├── user.go
│   └── order.go
└── v2/
    └── user.go          # novo formato de response
```

### Regras

- **Cada versão tem seus próprios DTOs** — Input/Output de v2 não herda de v1.
- **Services são compartilhados** — a versão só afeta handler e model (input/output).
- **Deprecation header**: Versões antigas retornam `Deprecation: true` e `Sunset: <date>`.
- **Máximo 2 versões ativas** em produção.

---

## Paginação e Ordenação

### Query Parameters

| Parâmetro | Tipo   | Default | Exemplo            |
|-----------|--------|---------|--------------------|
| `page`    | int    | `0`     | `?page=2`          |
| `size`    | int    | `20`    | `?size=50`         |
| `sort`    | string | —       | `?sort=name,asc`   |

### Structs (generics Go 1.18+)

```go
// internal/platform/pagination/pagination.go
package pagination

type Params struct {
    Page int    `form:"page" binding:"min=0"`
    Size int    `form:"size" binding:"min=1,max=100"`
    Sort string `form:"sort"`
}

func (p *Params) SetDefaults() {
    if p.Size == 0 {
        p.Size = 20
    }
}

func (p Params) Offset() int {
    return p.Page * p.Size
}

func (p Params) OrderClause() string {
    // "name,asc" → "name asc"
    return strings.Replace(p.Sort, ",", " ", 1)
}

type Page[T any] struct {
    Content []T      `json:"content"`
    Meta    PageMeta `json:"page"`
}

type PageMeta struct {
    Size          int   `json:"size"`
    Number        int   `json:"number"`
    TotalElements int64 `json:"total_elements"`
    TotalPages    int   `json:"total_pages"`
}

func NewPage[T any](items []T, params Params, total int64) *Page[T] {
    pages := int(math.Ceil(float64(total) / float64(params.Size)))
    return &Page[T]{
        Content: items,
        Meta: PageMeta{
            Size:          params.Size,
            Number:        params.Page,
            TotalElements: total,
            TotalPages:    pages,
        },
    }
}
```

### Uso no Handler

```go
func (h *UserHandler) FindAll(c *gin.Context) {
    var params pagination.Params
    if err := c.ShouldBindQuery(&params); err != nil {
        _ = c.Error(err)
        return
    }
    params.SetDefaults()

    page, err := h.svc.FindAll(c.Request.Context(), params)
    if err != nil {
        _ = c.Error(err)
        return
    }
    c.JSON(http.StatusOK, page)
}
```

---

## CORS

### gin-contrib/cors

```go
// internal/middleware/cors.go
package middleware

import (
    "time"
    "github.com/gin-contrib/cors"
    "github.com/gin-gonic/gin"
)

func CORS(allowedOrigins []string) gin.HandlerFunc {
    return cors.New(cors.Config{
        AllowOrigins:     allowedOrigins,
        AllowMethods:     []string{"GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"},
        AllowHeaders:     []string{"Content-Type", "Authorization", "Accept", "X-Request-ID"},
        ExposeHeaders:    []string{"Location", "X-Request-ID"},
        AllowCredentials: true,
        MaxAge:           1 * time.Hour,
    })
}
```

### Config por environment

```bash
# Produção
CORS_ALLOWED_ORIGINS=https://app.example.com,https://admin.example.com

# Dev
CORS_ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
```

---

## Feature Flags

### go-feature-flag

```
internal/platform/featureflag/
├── featureflag.go         # setup + client
└── flags.go               # constantes
```

### Setup

```go
func NewClient() (*ffclient.GoFeatureFlag, error) {
    return ffclient.New(ffclient.Config{
        PollingInterval: 60 * time.Second,
        Retriever: &fileretriever.Retriever{
            Path: "config/flags.yaml",
        },
    })
}
```

### Flag Definitions (YAML)

```yaml
# config/flags.yaml
new-checkout-flow:
  variations:
    enabled: true
    disabled: false
  defaultRule:
    variation: disabled
  targeting:
    - query: key eq "beta-user"
      variation: enabled

push-notifications:
  variations:
    enabled: true
    disabled: false
  defaultRule:
    variation: enabled
```

### Constantes

```go
const (
    FlagNewCheckoutFlow  = "new-checkout-flow"
    FlagPushNotifications = "push-notifications"
)
```

### Uso no Service

```go
func (s *CheckoutService) Process(ctx context.Context, req model.CheckoutInput) (*model.CheckoutResult, error) {
    evalCtx := ffcontext.NewEvaluationContextBuilder(req.UserID.String()).Build()

    if enabled, _ := s.ff.BoolVariation(featureflag.FlagNewCheckoutFlow, evalCtx, false); enabled {
        return s.processV2(ctx, req)
    }
    return s.processV1(ctx, req)
}
```

> **Higiene de flags:** Remova flags 100% habilitadas. Mantenha max ~10 ativas.

---

## Goroutines e Concorrência

Go usa **goroutines** nativamente — cada request Gin já é uma goroutine. Não há configuração especial como "virtual threads" — concorrência é first-class.

### Padrões Essenciais

| Padrão              | Quando usar                                    | Ferramenta                     |
|---------------------|------------------------------------------------|--------------------------------|
| **errgroup**        | Fan-out de N tarefas com cancelamento           | `golang.org/x/sync/errgroup`  |
| **Channel**         | Comunicação entre goroutines                    | `chan T`, buffered ou não      |
| **sync.WaitGroup**  | Esperar N goroutines sem retorno de erro        | `sync.WaitGroup`              |
| **context cancel**  | Propagar cancelamento/timeout                   | `context.WithTimeout/Cancel`  |
| **semaphore**       | Limitar concorrência máxima                     | `chan struct{}` ou `semaphore` |

### Worker Pool com errgroup

```go
func (s *ReportService) GenerateAll(ctx context.Context, ids []uuid.UUID) ([]model.Report, error) {
    results := make([]model.Report, len(ids))
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10) // max 10 goroutines simultâneas

    for i, id := range ids {
        g.Go(func() error {
            report, err := s.generate(ctx, id)
            if err != nil {
                return fmt.Errorf("generate report %s: %w", id, err)
            }
            results[i] = *report
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

### Graceful Shutdown

```go
func main() {
    cfg := mustLoadConfig()
    log := mustInitLogger(cfg.Log)
    defer log.Sync()

    engine := gin.New()
    // ... setup middlewares e rotas ...

    srv := &http.Server{
        Addr:         ":" + cfg.Server.Port,
        Handler:      engine,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Start server
    go func() {
        log.Info("server starting", zap.String("port", cfg.Server.Port))
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            log.Fatal("server error", zap.Error(err))
        }
    }()

    // Wait for interrupt
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    sig := <-quit
    log.Info("received signal, shutting down", zap.String("signal", sig.String()))

    // Graceful shutdown with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Error("server forced to shutdown", zap.Error(err))
    }
    log.Info("server stopped")
}
```

### Cuidados (Uber Go Style Guide)

- **Nunca lance goroutines sem ownership** — quem criou deve garantir que termina.
- **Sempre propague `context.Context`** — necessário para cancellation e deadline.
- **Goroutines em produtor/consumidor** devem reagir a `ctx.Done()`.
- **Use `errgroup`** quando precisa do erro de qualquer goroutine.
- **Use `sync.WaitGroup`** quando não precisa de erros.
- **Avoid goroutine leaks** — todo `go func()` deve ter condição de saída.

---

## Observabilidade

| Aspecto      | Tecnologia                      | Detalhes                                                        |
|--------------|---------------------------------|-----------------------------------------------------------------|
| **Métricas** | Prometheus client_golang        | `/metrics`, histogramas por rota, contadores de erro            |
| **Tracing**  | OpenTelemetry                   | Propagação W3C TraceContext, exporter OTLP                      |
| **Logging**  | Zap (uber-go/zap)               | JSON estruturado, nível por env, `trace_id` e `request_id`     |
| **Health**   | Endpoints customizados          | `/health/live`, `/health/ready` (com check de DB)               |
| **API Docs** | swaggo/swag + gin-swagger       | `/swagger/*` (habilitado apenas em GIN_MODE=debug)              |

### Health Check

```go
func RegisterHealthRoutes(r *gin.Engine, db *gorm.DB) {
    r.GET("/health/live", func(c *gin.Context) {
        c.JSON(200, gin.H{"status": "UP"})
    })
    r.GET("/health/ready", func(c *gin.Context) {
        sqlDB, _ := db.DB()
        if err := sqlDB.Ping(); err != nil {
            c.JSON(503, gin.H{"status": "DOWN", "db": err.Error()})
            return
        }
        c.JSON(200, gin.H{"status": "UP"})
    })
}
```

### Prometheus Middleware

```go
func Metrics() gin.HandlerFunc {
    requests := prometheus.NewCounterVec(prometheus.CounterOpts{
        Name: "http_requests_total",
    }, []string{"method", "path", "status"})

    duration := prometheus.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Buckets: prometheus.DefBuckets,
    }, []string{"method", "path"})

    prometheus.MustRegister(requests, duration)

    return func(c *gin.Context) {
        start := time.Now()
        c.Next()

        status := strconv.Itoa(c.Writer.Status())
        requests.WithLabelValues(c.Request.Method, c.FullPath(), status).Inc()
        duration.WithLabelValues(c.Request.Method, c.FullPath()).Observe(time.Since(start).Seconds())
    }
}
```

### Structured Logging com request context

```go
func Logger(log *zap.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = uuid.NewString()
        }
        c.Set("request_id", requestID)
        c.Header("X-Request-ID", requestID)

        c.Next()

        log.Info("request",
            zap.String("method", c.Request.Method),
            zap.String("path", c.Request.URL.Path),
            zap.Int("status", c.Writer.Status()),
            zap.Duration("latency", time.Since(start)),
            zap.String("request_id", requestID),
            zap.String("client_ip", c.ClientIP()),
        )
    }
}
```

---

## Docker

### Dockerfile (multi-stage)

```dockerfile
# ── Build ──
FROM golang:1.24-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /bin/api ./cmd/api

# ── Runtime ──
FROM gcr.io/distroless/static-debian12
WORKDIR /app
COPY --from=build /bin/api /app/api
COPY --from=build /src/migrations /app/migrations
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/app/api"]
```

| Estágio   | Base                           | Resultado                                |
|-----------|--------------------------------|------------------------------------------|
| `build`   | `golang:1.24-alpine`          | Compila binário estático (~10MB)          |
| `runtime` | `distroless/static-debian12`  | Imagem final ~15MB, sem shell, sem OS tools |

> **Por que distroless?** Zero shell + zero package manager = menor superfície de ataque. Alternativa: `scratch` (ainda mais enxuto, mas sem CA certs — precisa copiar manualmente).

### docker-compose.yml

```yaml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      DATABASE_DSN: root:root@tcp(mysql:3306)/app?parseTime=true
      GIN_MODE: release
      JWT_ISSUER_URI: https://auth.example.com
      JWK_SET_URI: https://auth.example.com/.well-known/jwks.json
    depends_on:
      mysql:
        condition: service_healthy

  mysql:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: app
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 3s
      retries: 10
```

### Dev local

```bash
docker-compose up -d mysql
migrate -path migrations -database "mysql://root:root@tcp(localhost:3306)/app" up
GIN_MODE=debug go run ./cmd/api
```

---

## Guia para Criar Novo Recurso

Ao adicionar um novo recurso (ex.: `order`), siga esta checklist:

### 1. Model

- [ ] `internal/model/order.go` — struct `Order` (GORM tags), `CreateOrderInput`, `OrderOutput`, `ToOutput()`

### 2. Store

- [ ] `internal/store/order.go` — `OrderStore` struct + métodos GORM (FindByID, FindAll, Create, Update, Delete)

### 3. Service

- [ ] `internal/service/order.go` — `OrderService` struct + interface `OrderStore` (consumer-side) + regras de negócio

### 4. Handler

- [ ] `internal/handler/order.go` — `OrderHandler` struct + Gin handlers (Create, FindByID, FindAll, Update, Delete)

### 5. Router

- [ ] `internal/router/router.go` — registrar rotas `v1.Group("/orders")` com middlewares

### 6. Migration

- [ ] `migrations/{seq}_create_orders.up.sql`
- [ ] `migrations/{seq}_create_orders.down.sql`

### 7. Testes

- [ ] `internal/service/order_test.go` — table-driven, mocks
- [ ] `internal/handler/order_test.go` — httptest + mock service
- [ ] `test/fixture/order.go` — `NewOrder()`, `NewCreateOrderInput()`
- [ ] `test/integration/order_test.go` — testcontainers (se aplicável)

### 8. Bootstrap

- [ ] `cmd/api/main.go` — wire `store → service → handler`, registrar rotas

---

## Onde colocar este arquivo para o Copilot

O GitHub Copilot reconhece **automaticamente** arquivos de instruções em locais específicos.

### 1. Instruções no nível do repositório (recomendado)

Copie para:

```
.github/copilot-instructions.md
```

O Copilot lê automaticamente em todas as conversas do repositório.

### 2. Instruções por workspace (VS Code)

```jsonc
// .vscode/settings.json
{
  "github.copilot.chat.codeGeneration.instructions": [
    { "file": "project-structure.md" }
  ]
}
```

### 3. Instruction files (`.instructions.md`)

Crie arquivos `*.instructions.md` em qualquer pasta e habilite:

```jsonc
{
  "github.copilot.chat.codeGeneration.useInstructionFiles": true
}
```

### Recomendação

Use **`.github/copilot-instructions.md`** — funciona automaticamente no VS Code e GitHub.com.
