# Zero to Hero — Backend Challenge (Go + Java Multi-Framework)

> **Programa de especialização progressiva** em Go e Java, com desafio unificado implementado em 5 stacks:
> **Go (Gin)** · **Spring Boot** · **Quarkus** · **Micronaut** · **Jakarta EE**

---

## 1. Resumo Executivo

Este programa transforma um desenvolvedor em **especialista** em duas linguagens (Go e Java) e seus principais frameworks backend, através de um **desafio único de domínio** implementado em 5 stacks diferentes — com mesma regra de negócio, mesmos contratos funcionais e mesmos critérios de aceite.

**Domínio escolhido:** Sistema de **Gestão de Carteira Digital (Digital Wallet)** — um domínio com valor real de negócio que cobre CRUD, transações financeiras, integrações externas, mensageria, segurança, observabilidade, resiliência e cenários de produção.

**Por que Digital Wallet?**
- Não é CRUD genérico — envolve **lógica transacional**, saldos, concorrência, idempotência
- Relevante para **fintechs, bancos digitais, e-commerce** (portfólio forte)
- Cobre **todos os cenários de produção**: race conditions, retry, circuit breaker, audit trail
- Perguntas de entrevista Staff/Principal frequentemente envolvem sistemas financeiros
- Permite crescer de MVP simples até enterprise com mensageria e observabilidade completa

---

## 2. Leitura dos READMEs — Padrões Comuns e Diferenças

### 2.1 Padrões Comuns (Todos os 5 Stacks)

| Aspecto | Padrão Unificado |
|---|---|
| **Arquitetura de pacotes** | 3-4 camadas: API/Handler → Service → Store/Repository → Model/Entity |
| **Error handling** | RFC 9457 (ProblemDetail) — formato JSON padronizado |
| **Validação** | Bean Validation (Java) / go-playground/validator (Go) |
| **Paginação** | `page`, `size`, `sort` — resposta com `content`, `totalElements`, `totalPages` |
| **Versionamento API** | Por path: `/v1/`, `/v2/` |
| **Migrations** | SQL versionado (Flyway / golang-migrate) — nunca auto-DDL |
| **Testes** | Unitário + Integração (Testcontainers/Dev Services) + Carga (k6) |
| **Observabilidade** | Métricas (Micrometer/Prometheus) + Tracing (OpenTelemetry) + Logging estruturado (JSON) |
| **Resiliência** | Circuit Breaker + Retry + Timeout + Bulkhead |
| **Segurança** | JWT (OAuth2 Resource Server) + RBAC |
| **Feature Flags** | Togglz (Java) / go-feature-flag (Go) |
| **Docs API** | OpenAPI/Swagger gerado automaticamente |
| **CORS** | Configurável por environment/profile |
| **Docker** | Multi-stage build |
| **Naming** | Tabelas `tb_*` (Java) / plural snake_case (Go), UUID PKs |

### 2.2 Diferenças Estruturais Relevantes

| Conceito | Go (Gin) | Spring Boot | Quarkus | Micronaut | Jakarta EE |
|---|---|---|---|---|---|
| **Camada HTTP** | `handler/` | `api/rest/v1/controller/` | `api/rest/v1/resource/` | `api/rest/v1/controller/` | `api/rest/v1/resource/` |
| **Lógica de negócio** | `service/` | `domain/service/` | `domain/service/` | `domain/service/` | `domain/service/` |
| **Persistência** | `store/` | `domain/repository/mysql/` | `domain/repository/mysql/` | `domain/repository/mysql/` | `domain/repository/mysql/` |
| **Modelos** | `model/` (entity + DTOs juntos) | `domain/entity/` + `api/.../request/` + `response/` | idem Spring | idem Spring | idem Spring |
| **Mapeamento** | Métodos no model (`ToOutput()`) | MapStruct (`componentModel="spring"`) | MapStruct (`componentModel="cdi"`) | MapStruct (`componentModel="jsr330"`) | MapStruct (`componentModel="cdi"`) |
| **DI** | Manual ou `uber-go/fx` | Spring IoC (runtime) | CDI (runtime, build-time otimizado) | Compile-time DI | CDI 4.1 (runtime) |
| **Config** | Env vars + struct (`caarlos0/env`) | `application.yml` + profiles | `application.properties` + `%profile.` prefix | `application.yml` + environments | `microprofile-config.properties` |
| **REST framework** | Gin (`gin.HandlerFunc`) | Spring MVC (`@RestController`) | RESTEasy Reactive (JAX-RS `@Path`) | Micronaut HTTP (`@Controller`) | JAX-RS 4.0 (`@Path`) |
| **Error global** | Middleware recovery + `c.Error()` | `@RestControllerAdvice` | `ExceptionMapper<T>` | `ExceptionHandler<T, R>` | `ExceptionMapper<T>` |
| **ORM** | GORM (ou sqlx) | Spring Data JPA + Hibernate | Hibernate ORM with Panache | Micronaut Data JPA | JPA 3.2 + Jakarta Data 1.0 |
| **Resiliência** | sony/gobreaker + cenkalti/backoff | Resilience4j | SmallRye Fault Tolerance | Micronaut Retry/CircuitBreaker | MicroProfile Fault Tolerance |
| **HTTP Client** | `net/http` / go-resty | `@HttpExchange` | `@RegisterRestClient` | `@Client` | MicroProfile REST Client |
| **Build nativo** | Binário estático (go build) | GraalVM (via Spring Native) | GraalVM nativo (excelente) | GraalVM nativo (excelente) | N/A (WAR/EAR em app server) |
| **Virtual Threads** | Goroutines (nativas) | `spring.threads.virtual.enabled` | `@RunOnVirtualThread` | `@ExecuteOn(VIRTUAL)` | Jakarta Concurrency 3.1 |
| **Deployment** | Binário único | Fat JAR (CDS) | JVM JAR ou Native | JVM JAR ou Native | WAR em app server (WildFly, Payara, etc.) |

### 2.3 Equivalências Conceituais

```
Go (Gin)                    Java (Spring/Quarkus/Micronaut/Jakarta EE)
─────────────               ──────────────────────────────────────────
handler/                  → api/rest/v1/controller/ (ou resource/)
service/                  → domain/service/
store/                    → domain/repository/
model/ (entity+DTO)       → domain/entity/ + api/.../request/ + response/
model.ToOutput()          → core/mapper/ (MapStruct)
middleware/               → infrastructure/ (configs, filters, interceptors)
platform/                 → infrastructure/
errs/                     → domain/exception/
platform/httperr/         → api/exception/ (global handler)
platform/pagination/      → Spring Pageable / Panache / manual Page
config/config.go          → application.yml / application.properties / microprofile-config.properties
router/                   → @RequestMapping / @Path annotations
internal/                 → package-private (convenção Java)
```

### 2.4 Trade-offs entre Frameworks

| Dimensão | Go (Gin) | Spring Boot | Quarkus | Micronaut | Jakarta EE |
|---|---|---|---|---|---|
| **Startup** | ~50ms | ~2-5s (CDS: ~1s) | ~0.5-1s (JVM), ~20ms (native) | ~0.5-1s (JVM), ~20ms (native) | ~3-10s (app server) |
| **Footprint (RAM)** | ~10-20MB | ~150-300MB | ~50-100MB (JVM), ~30MB (native) | ~50-100MB (JVM), ~30MB (native) | ~200-500MB |
| **Throughput** | Excelente | Muito bom | Excelente | Excelente | Bom |
| **DX (Dev Experience)** | Simples, minimalista | Melhor ecossistema | Dev Mode excelente | Rápido, menos magia | Mais verboso |
| **Curva de aprendizado** | Moderada (idiomática) | Moderada | Baixa (se vem de Spring) | Baixa (se vem de Spring) | Alta (especificações) |
| **Ecossistema** | Crescente | O maior do Java | Crescente, cloud-native | Crescente | Estável, enterprise |
| **Build nativo** | Default | Possível (GraalVM) | Excelente | Excelente | N/A |
| **Portabilidade** | Multi-plataforma | JVM anywhere | JVM + Native | JVM + Native | Qualquer app server certificado |
| **Complexidade operacional** | Baixa (binário) | Média | Baixa-Média | Baixa-Média | Alta (app server) |

### 2.5 Arquitetura do Sistema — Visão Geral

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          NovaPay Digital Wallet                                  │
│                                                                                 │
│  ┌─────────┐     ┌──────────────┐     ┌──────────────────────────────────────┐  │
│  │ Client   │────→│ Kong API GW  │────→│          Wallet API                  │  │
│  │ (REST/   │     │ (Rate Limit  │     │  ┌──────────────────────────────┐   │  │
│  │  gRPC)   │     │  OIDC, CORS) │     │  │  Handler / Controller        │   │  │
│  └─────────┘     └──────────────┘     │  ├──────────────────────────────┤   │  │
│                         │              │  │  Service (Business Logic)     │   │  │
│                  ┌──────▼──────┐       │  ├──────────────────────────────┤   │  │
│                  │  Keycloak    │       │  │  Repository / Store          │   │  │
│                  │  (OIDC IdP)  │       │  └──────────┬───────────────────┘   │  │
│                  └─────────────┘       └─────────────┼───────────────────────┘  │
│                                                       │                          │
│  ┌────────────────────┐    ┌──────────────────────────┼──────────────────────┐  │
│  │  Observability      │    │        Data Layer         │                      │  │
│  │                     │    │  ┌──────────────┐  ┌──────▼──────┐              │  │
│  │  Prometheus         │    │  │  Kafka/SQS   │  │ PostgreSQL  │              │  │
│  │  Grafana            │    │  │  (Events)    │  │ (Write DB)  │              │  │
│  │  Jaeger/Tempo       │    │  └──────┬───────┘  └──────┬──────┘              │  │
│  │  Loki               │    │         │                  │                     │  │
│  └────────────────────┘    │  ┌──────▼───────┐  ┌──────▼──────┐              │  │
│                             │  │  Consumers   │  │  Debezium   │              │  │
│  ┌────────────────────┐    │  │  (Notify,    │  │  (CDC)      │              │  │
│  │  Chaos / SRE        │    │  │   Audit)     │  └──────┬──────┘              │  │
│  │                     │    │  └──────────────┘         │                     │  │
│  │  LitmusChaos        │    │                    ┌──────▼──────┐              │  │
│  │  k6 Load Test       │    │                    │  Read Model  │              │  │
│  │  Runbooks           │    │                    │  (CQRS)     │              │  │
│  └────────────────────┘    │                    └─────────────┘              │  │
│                             └────────────────────────────────────────────────┘  │
│                                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │  Deployment: Docker → Helm → Argo CD (GitOps) → K8s + Service Mesh       │ │
│  │  Security: JWT/OIDC → RBAC → Supply Chain (Trivy, Cosign, SBOM)          │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 2.6 Conceitos Transversais — Referência Rápida

| Conceito | Level | Descrição |
|---|---|---|
| **RED Metrics** | 4 | Rate, Errors, Duration — monitoramento de serviços |
| **USE Metrics** | 4 | Utilization, Saturation, Errors — monitoramento de recursos |
| **VALET Metrics** | 4 | Volume, Availability, Latency, Errors, Tickets — métricas de negócio |
| **Golden Signals** | 4 | Latency, Traffic, Errors, Saturation — Google SRE |
| **SLIs / SLOs** | 4, 9 | Service Level Indicators & Objectives + Error Budget |
| **API Gateway** | 7 | Kong — rate limiting, OIDC, routing centralizado |
| **Service Mesh** | 7 | Istio/Linkerd — mTLS, canary deploy, fault injection |
| **Helm Charts** | 7 | Pacotes K8s parametrizáveis para multi-environment |
| **GitOps (Argo CD)** | 7 | Estado desejado no Git, reconciliação automática |
| **Keycloak** | 5 | Identity Provider corporativo (OIDC, SSO, MFA) |
| **Supply Chain Security** | 5 | Trivy, Cosign, SBOM — segurança da cadeia de software |
| **gRPC** | 1 | Comunicação binária entre serviços (Protobuf/HTTP2) |
| **CDC (Debezium)** | 6 | Change Data Capture via WAL/binlog do banco |
| **CQRS** | 6 | Separação de modelos de leitura e escrita |
| **Event Sourcing** | 6 | Estado como sequência de eventos imutáveis |
| **SNS/SQS** | 6 | AWS messaging via LocalStack (fan-out pattern) |
| **Schema Registry** | 6 | Versionamento e compatibilidade de schemas (Avro) |
| **LitmusChaos** | 9 | Chaos engineering declarativo no Kubernetes |
| **SRE Runbooks** | 9 | Procedimentos operacionais para incident response |

---

## 3. Desafio Unificado — Digital Wallet

### 3.1 Descrição do Domínio

Sistema de **Carteira Digital** que permite:
- Criar e gerenciar **usuários** (donos de carteiras)
- Criar e consultar **carteiras** (wallets) com saldo
- Realizar **transações** (depósito, saque, transferência entre wallets)
- Consultar **extrato** com paginação, filtros e ordenação
- Integrar com **API externa** para cotação de moeda (ou validação de fraude)
- Emitir **eventos** de transação para processamento assíncrono
- **Notificações** via mensageria após transações

### 3.2 Entidades do Domínio

```
User (1) ──── (N) Wallet (1) ──── (N) Transaction
                                        │
                                        └── type: DEPOSIT | WITHDRAWAL | TRANSFER
                                        └── status: PENDING | COMPLETED | FAILED | REVERSED
```

| Entidade | Campos Principais |
|---|---|
| **User** | `id` (UUID), `name`, `email` (unique), `document` (CPF/CNPJ), `status` (ACTIVE/INACTIVE), `created_at`, `updated_at` |
| **Wallet** | `id` (UUID), `user_id` (FK), `currency` (BRL/USD/EUR), `balance` (decimal), `status` (ACTIVE/BLOCKED), `created_at`, `updated_at` |
| **Transaction** | `id` (UUID), `wallet_id` (FK), `target_wallet_id` (nullable FK), `type`, `amount` (decimal), `currency`, `status`, `description`, `idempotency_key` (unique), `created_at` |

---

## 4. Contrato Funcional Comum

### 4.1 Endpoints (todos os stacks, mesma API)

```
Base path: /api/v1

── Users ──
POST   /api/v1/users                         → Criar usuário
GET    /api/v1/users/{id}                     → Buscar por ID
GET    /api/v1/users?page=0&size=20&sort=name,asc → Listar paginado
PUT    /api/v1/users/{id}                     → Atualizar
DELETE /api/v1/users/{id}                     → Soft delete

── Wallets ──
POST   /api/v1/users/{userId}/wallets         → Criar wallet
GET    /api/v1/wallets/{id}                    → Buscar wallet
GET    /api/v1/users/{userId}/wallets          → Listar wallets do usuário

── Transactions ──
POST   /api/v1/wallets/{walletId}/transactions/deposit    → Depósito
POST   /api/v1/wallets/{walletId}/transactions/withdrawal → Saque
POST   /api/v1/wallets/{walletId}/transactions/transfer   → Transferência
GET    /api/v1/wallets/{walletId}/transactions             → Extrato paginado
GET    /api/v1/transactions/{id}                           → Buscar transação

── Health ──
GET    /api/v1/health                          → Liveness
GET    /api/v1/health/ready                    → Readiness
```

### 4.2 Payloads

**CreateUserRequest:**
```json
{
  "name": "João Silva",
  "email": "joao@example.com",
  "document": "123.456.789-00"
}
```

**CreateWalletRequest:**
```json
{
  "currency": "BRL"
}
```

**DepositRequest:**
```json
{
  "amount": 100.50,
  "description": "Depósito inicial",
  "idempotency_key": "550e8400-e29b-41d4-a716-446655440000"
}
```

**TransferRequest:**
```json
{
  "target_wallet_id": "...",
  "amount": 50.00,
  "description": "Pagamento",
  "idempotency_key": "..."
}
```

**Resposta paginada (padrão para todos):**
```json
{
  "content": [ ... ],
  "page": {
    "size": 20,
    "number": 0,
    "totalElements": 150,
    "totalPages": 8
  }
}
```

**Erro (RFC 9457 ProblemDetail):**
```json
{
  "type": "https://api.example.com/errors/insufficient-balance",
  "title": "Saldo insuficiente",
  "status": 422,
  "detail": "Wallet xyz não possui saldo suficiente para saque de R$ 100.00",
  "instance": "/api/v1/wallets/xyz/transactions/withdrawal",
  "errorCode": "INSUFFICIENT_BALANCE",
  "timestamp": "2026-03-01T10:30:00Z"
}
```

### 4.3 Regras de Negócio (invariantes)

1. **Email é único** — conflito retorna 409
2. **Saldo nunca negativo** — saque/transferência sem saldo retorna 422
3. **Idempotência** — `idempotency_key` duplicada retorna a transação existente (200), não cria nova
4. **Transferência é atômica** — débito + crédito na mesma transação DB
5. **Soft delete** — usuários são desativados, nunca removidos fisicamente
6. **Wallet bloqueada** — não aceita transações (422)
7. **Moedas devem ser iguais** em transferência — se diferentes, retorna 422

---

## 5. Trilha Zero to Hero (Nível 0 ao Nível 8)

| Nível | Foco | Semana |
|---|---|---|
| [Level 0](00-foundations.md) | Fundamentos da linguagem e tooling | Semana 1-2 |
| [Level 1](01-crud-architecture.md) | CRUD + Arquitetura base | Semana 3-4 |
| [Level 2](02-persistence-migrations.md) | Persistência real + Migrations | Semana 5-6 |
| [Level 3](03-quality-tests.md) | Qualidade + Testes | Semana 7-8 |
| [Level 4](04-observability-resilience.md) | Observabilidade + Resiliência | Semana 9-10 |
| [Level 5](05-security-authz.md) | Segurança + Autorização | Semana 11-12 |
| [Level 6](06-async-messaging.md) | Assíncrono + Mensageria | Semana 13-14 |
| [Level 7](07-production-deploy.md) | Produção + Deploy | Semana 15-16 |
| [Level 8](08-benchmark-comparative.md) | Benchmark + Comparativo | Semana 17-18 |
| [Level 9](09-capstone-digital-wallet.md) | Capstone: Production Launch | Semana 19-22 |

---

## 6. Plano de Especialização — Go

### Progressão Recomendada

| Fase | Tópico | Nível associado |
|---|---|---|
| 1 | Go fundamentals (types, interfaces, errors, structs, pointers) | Level 0 |
| 2 | Módulos, packages, `internal/`, testing stdlib | Level 0 |
| 3 | `net/http`, Gin, routing, middleware, JSON | Level 1 |
| 4 | `context.Context`, propagação, cancellation | Level 1-2 |
| 5 | `database/sql`, GORM, connection pool, transactions | Level 2 |
| 6 | Table-driven tests, testify, mocks, httptest, testcontainers | Level 3 |
| 7 | Goroutines, channels, `sync`, worker pools | Level 4-6 |
| 8 | Zap logging, Prometheus, OpenTelemetry | Level 4 |
| 9 | JWT, middleware auth, RBAC | Level 5 |
| 10 | Kafka (kafka-go), producer/consumer patterns | Level 6 |
| 11 | Docker multi-stage, graceful shutdown, 12-factor | Level 7 |
| 12 | pprof, benchmarks, allocation profiling, GC tuning | Level 8 |

### Recursos Essenciais
- [Effective Go](https://go.dev/doc/effective_go)
- [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)
- [Go by Example](https://gobyexample.com)
- [Let's Go (Alex Edwards)](https://lets-go.alexedwards.net/)

---

## 7. Plano de Especialização — Java + Frameworks

### Progressão da Linguagem Java

| Fase | Tópico | Nível associado |
|---|---|---|
| 1 | Java 25+: records, sealed classes, pattern matching, text blocks | Level 0 |
| 2 | Collections, Streams API, Optional | Level 0 |
| 3 | JVM internals: memória, GC (G1, ZGC), JIT, class loading | Level 0-8 |
| 4 | Concurrency: Virtual Threads, CompletableFuture, ExecutorService | Level 4-6 |
| 5 | Reflection vs compile-time processing (annotation processors) | Level 0 |

### Progressão por Framework (Ordem obrigatória)

```
Spring Boot (primeiro — mainstream, maior ecossistema)
    ↓
Quarkus (segundo — cloud-native, comparação direta)
    ↓
Micronaut (terceiro — compile-time DI, startup)
    ↓
Jakarta EE (quarto — especificações, portabilidade)
```

**Justificativa da ordem:**
1. **Spring Boot** primeiro porque é o mais usado no mercado, maior base de conhecimento e mais fácil de encontrar exemplos
2. **Quarkus** segundo porque reutiliza muitos conceitos de CDI/JAX-RS e permite comparação direta de performance
3. **Micronaut** terceiro porque o compile-time DI é um paradigma diferente que vale entender após CDI/Spring
4. **Jakarta EE** por último porque entender as especificações dá profundidade conceitual sobre o que os frameworks abstraem

### Comparações Contínuas (por nível)

Ao completar cada nível em cada framework, documente:

| Aspecto | Spring Boot | Quarkus | Micronaut | Jakarta EE |
|---|---|---|---|---|
| DI | `@Autowired` / construtor | CDI `@Inject` | Compile-time `@Inject` | CDI `@Inject` |
| REST | `@RestController` | `@Path` (JAX-RS) | `@Controller` | `@Path` (JAX-RS) |
| Config | `application.yml` + profiles | `application.properties` + `%profile.` | `application.yml` + environments | `microprofile-config.properties` |
| Validação | Bean Validation + `@Valid` | Bean Validation + `@Valid` | `@Valid` + `@Introspected` | Bean Validation + `@Valid` |
| Error | `@RestControllerAdvice` | `ExceptionMapper` | `ExceptionHandler` | `ExceptionMapper` |
| Segurança | Spring Security | Quarkus Security | Micronaut Security | Jakarta Security |
| Resiliência | Resilience4j | SmallRye Fault Tolerance | Micronaut Retry/CB | MicroProfile FT |
| Testes | `@WebMvcTest` / `@SpringBootTest` | `@QuarkusTest` | `@MicronautTest` | Arquillian |
| Observabilidade | Micrometer + Actuator | SmallRye Metrics/OTel | Micronaut Micrometer | MicroProfile Metrics/Telemetry |
| Data access | Spring Data JPA | Panache | Micronaut Data | JPA + Jakarta Data |

---

## 8. Matriz de Equivalência entre os 5 Modelos

| Aspecto | Go (Gin) | Spring Boot | Quarkus | Micronaut | Jakarta EE |
|---|---|---|---|---|---|
| **Endpoint handler** | `gin.HandlerFunc` | `@RestController` record | `@Path` resource record | `@Controller` record | `@Path` resource |
| **DTO request** | `model.CreateUserInput` (struct + binding tags) | `UserRequest` record + `@Valid` | `UserRequest` record + `@Valid` | `UserRequest` record + `@Valid` + `@Introspected` | `UserRequest` record + `@Valid` |
| **DTO response** | `model.UserOutput` (struct + json tags) | `UserResponse` record | `UserResponse` record | `UserResponse` record | `UserResponse` record |
| **Validação** | `binding:"required,email"` | `@NotBlank`, `@Email` | `@NotBlank`, `@Email` | `@NotBlank`, `@Email` | `@NotBlank`, `@Email` |
| **Error global** | `middleware/recovery.go` + `c.Error()` | `@RestControllerAdvice` | `ExceptionMapper<T>` | `ExceptionHandler<T,R>` | `ExceptionMapper<T>` |
| **Config** | `config.Load()` (env vars) | `application.yml` | `application.properties` | `application.yml` | `microprofile-config.properties` |
| **Migration tool** | golang-migrate | Flyway | Flyway | Flyway (Micronaut Flyway) | Flyway |
| **Migration format** | `000001_create_users.up.sql` | `V2.0__create_tb_user.sql` | `V2.0__create_tb_user.sql` | `V2.0__create_tb_user.sql` | `V2.0__create_tb_user.sql` |
| **Test unitário** | `go test` + testify | JUnit 5 + Mockito + `@WebMvcTest` | JUnit 5 + Mockito + REST Assured | JUnit 5 + Mockito + `@MicronautTest` | JUnit 5 + Mockito + REST Assured |
| **Test integração** | testcontainers-go | Testcontainers + Cucumber | Dev Services + Cucumber | Testcontainers + Cucumber | Arquillian + Testcontainers |
| **Observabilidade** | Prometheus + OTel + Zap | Micrometer + OTel + Logback | SmallRye Metrics + OTel + JBoss Log | Micronaut Micrometer + OTel + Logback | MicroProfile Metrics + OTel + JBoss Log |
| **Auth** | `golang-jwt` + middleware | Spring Security + OAuth2 RS | Quarkus Security + SmallRye JWT | Micronaut Security + JWT | Jakarta Security + MicroProfile JWT |
| **Resiliência** | gobreaker + backoff | Resilience4j | SmallRye Fault Tolerance | Micronaut Retry/CB | MicroProfile Fault Tolerance |
| **Build/Run** | `go build` / `go run` | `./mvnw spring-boot:run` | `./mvnw quarkus:dev` | `./mvnw mn:run` | `./mvnw package` + deploy WAR |
| **Dev mode** | `air` (hot reload) | Spring DevTools | Live Coding (`quarkus dev`) | `mn:run` + watch | WildFly Dev Mode |

---

## 9. Roadmap de Execução (18 semanas)

| Semana | Atividade | Stack Foco | Entregável |
|---|---|---|---|
| 1-2 | Level 0: Fundamentos Go + Java 25 | Go + Java | Exercícios + setup do projeto |
| 3-4 | Level 1: CRUD em Go (Gin) + Spring Boot | Go, Spring | API funcionando com mock data |
| 5-6 | Level 2: Persistência + Migrations | Go, Spring | DB real + Flyway/migrate |
| 7-8 | Level 3: Testes completos | Go, Spring | Cobertura >80%, table-driven tests |
| 9-10 | Level 4: Observabilidade + Resiliência | Go, Spring | Prometheus + Grafana + Circuit Breaker |
| 11-12 | Level 5: Segurança | Go, Spring | JWT + RBAC funcionando |
| 13-14 | Level 6: Mensageria | Go, Spring | Kafka producer/consumer |
| 15-16 | Level 7: Docker + Deploy | Todos os 5 | Docker Compose completo |
| 17-18 | Level 8: Benchmark | Todos os 5 | Relatório comparativo k6 |
| 19-22 | Level 9: Capstone | Todos os 5 | Production launch + report |

**Estratégia:** Completar cada nível primeiro em Go + Spring Boot, depois replicar em Quarkus → Micronaut → Jakarta EE. A replicação fica cada vez mais rápida conforme os conceitos são internalizados.

---

## 10. Rubrica de Avaliação (0–100)

| Critério | Peso | 0-25 (Básico) | 26-50 (Intermediário) | 51-75 (Avançado) | 76-100 (Expert) |
|---|---|---|---|---|---|
| **Código** | 25% | Funciona com bugs | Clean code, sem bugs | Idiomático, performático | Production-grade, optimized |
| **Arquitetura** | 20% | Monolítico, sem camadas | Camadas separadas | Segue README conventions | Extensível, bem justificado |
| **Testes** | 20% | Sem testes | Unit tests básicos | Unit + Integration + >80% | Mutation testing + Table-driven + BDD |
| **Observabilidade** | 10% | Sem logs | Logs básicos | Métricas + Tracing | Dashboards + Alertas + SLI/SLO |
| **Segurança** | 10% | Sem auth | Basic auth | JWT + RBAC | OAuth2 + Rate limiting + Audit |
| **DevOps** | 10% | Rode local | Docker | Docker Compose + CI | Kubernetes + CD + Observability stack |
| **Documentação** | 5% | Sem README | README básico | OpenAPI + README completo | ADRs + Changelog + Portfólio-ready |

---

## 11. Critérios de "Pronto para Produção"

- [ ] Todos os endpoints testados (unit + integration)
- [ ] Cobertura de testes > 80%
- [ ] ProblemDetail (RFC 9457) em todos os erros
- [ ] Health check (liveness + readiness)
- [ ] Métricas expostas (Prometheus format)
- [ ] Tracing distribuído (OpenTelemetry)
- [ ] Logging estruturado (JSON)
- [ ] JWT authentication + RBAC
- [ ] Rate limiting
- [ ] Circuit breaker em integrações externas
- [ ] Idempotência em transações
- [ ] Graceful shutdown
- [ ] Docker multi-stage build
- [ ] Migrations versionadas (sem auto-DDL)
- [ ] Documentação OpenAPI/Swagger
- [ ] README completo no repositório

---

## 12. Critérios de "Especialista" por Linguagem/Framework

### Go Specialist
- [ ] Domina idiomático Go: interfaces implícitas, error wrapping, context propagation
- [ ] Escreve table-driven tests naturalmente
- [ ] Usa pprof para profiling de CPU e memória
- [ ] Implementa graceful shutdown com signal handling
- [ ] Entende GC do Go, alocação no heap vs stack
- [ ] Implementa worker pools com goroutines e channels
- [ ] Domina `sync.Mutex`, `sync.WaitGroup`, `sync.Once`
- [ ] Benchmarks com `testing.B`

### Java Specialist (por framework)
- [ ] Entende JVM: JIT compilation, GC (G1, ZGC), class loading
- [ ] Domina Virtual Threads e sabe quando não usar
- [ ] Usa records, sealed classes, pattern matching naturalmente
- [ ] Entende diferenças entre runtime DI e compile-time DI
- [ ] Sabe configurar e tunar Hibernate (N+1, batch fetching, lazy loading)
- [ ] Implementa observabilidade completa (métricas custom + tracing + structured logging)
- [ ] Domina pelo menos 2 frameworks Java e sabe explicar trade-offs

---

## 13. Versões do Desafio

### MVP (2-4h)
- CRUD de User com validação
- In-memory storage (sem DB)
- Apenas Go + Spring Boot
- HTTP apenas

### Intermediária (1-2 semanas)
- CRUD completo (User + Wallet + Transaction)
- Banco de dados real (MySQL/PostgreSQL)
- Migrations
- Testes unitários
- ProblemDetail errors

### Avançada (3-4 semanas)
- + Paginação/ordenação
- + Observabilidade (métricas + tracing)
- + Circuit breaker em API externa
- + JWT auth
- + Docker Compose

### Enterprise (6-8 semanas)
- + Mensageria (Kafka)
- + Notificações async
- + Idempotência completa
- + Rate limiting
- + Feature flags
- + k6 load tests
- + Benchmark comparativo entre 5 stacks
- + CI/CD pipeline
- + Kubernetes manifests

---

## 14. Próximos Passos Recomendados

### Portfólio GitHub
- 5 repositórios separados (1 por stack) com README padronizado
- Ou 1 monorepo com pastas por stack
- Incluir badges de CI, cobertura, versão
- README com arquitetura, trade-offs, benchmarks

### Posts Técnicos
1. "Implementando o mesmo backend em 5 frameworks: o que aprendi"
2. "Go vs Java: benchmarks reais em um sistema financeiro"
3. "Circuit Breaker patterns: Resilience4j vs gobreaker vs SmallRye"
4. "Virtual Threads vs Goroutines: quando usar cada um"
5. "Do Spring Boot ao Jakarta EE: entendendo as especificações por trás dos frameworks"

### Benchmark Comparativo (ideias)
- **Latência p50/p95/p99** por endpoint (k6)
- **Throughput** (RPS máximo sustentável)
- **Consumo de memória** em idle e sob carga
- **Tempo de startup** (cold start)
- **Tamanho da imagem Docker**
- **Tempo de build**
- **Developer Experience** (subjetivo, mas documentável)

### Commits Sugeridos (por nível)

```
feat(level-0): setup project structure and tooling
feat(level-1): add user CRUD endpoints with validation
feat(level-1): add wallet and transaction endpoints
feat(level-2): add database persistence with GORM/JPA
feat(level-2): add Flyway/golang-migrate migrations
feat(level-3): add unit tests with table-driven/parameterized
feat(level-3): add integration tests with testcontainers
feat(level-4): add Prometheus metrics and health checks
feat(level-4): add OpenTelemetry tracing
feat(level-4): add circuit breaker for external API
feat(level-5): add JWT authentication
feat(level-5): add RBAC authorization
feat(level-6): add Kafka producer for transaction events
feat(level-6): add Kafka consumer for notifications
feat(level-7): add Docker multi-stage build
feat(level-7): add docker-compose with full stack
feat(level-8): add k6 load test scripts
docs(level-8): add benchmark comparison report
```

---

## 15. Riscos e Anti-padrões Comuns

| Anti-padrão | Stack | Como evitar |
|---|---|---|
| Forçar DI do Spring em Go | Go | Use constructor injection manual ou fx — Go não precisa de container DI |
| AutoMigrate do GORM | Go | Use golang-migrate com SQL explícito |
| `c.JSON(500, ...)` em cada handler | Go | Use `c.Error(err)` + middleware de recovery |
| `@Autowired` em campo (field injection) | Spring | Use injeção por construtor (record) |
| Não usar `@Transactional(readOnly=true)` | Java | Leituras com readOnly para otimização |
| Ignorar N+1 queries | Java | Configure `@BatchSize`, `JOIN FETCH`, ou projections |
| Não propagar context.Context | Go | Sempre primeiro parâmetro em todas as funções |
| Misturar DTOs entre versões de API | Todos | Cada versão tem seus próprios DTOs |
| Testes que dependem de ordem | Todos | Table-driven tests isolados, `t.Parallel()` |
| Não configurar connection pool | Todos | Defina `maxOpenConns`, `maxIdleConns`, `HikariCP` settings |
| Usar `fmt.Println` como logging | Go | Use `zap.Logger` ou `slog` |
| Não fazer graceful shutdown | Todos | Escute `os.Signal`, drene requisições em andamento |
