# Zero to Hero — Microservice Patterns Challenge (Java 25 + Go 1.26)

> **Programa de especialização progressiva** em padrões de microsserviços, implementado com
> **Java 25** (Spring Boot 3.x, Resilience4j, Spring Cloud, Spring Kafka) · **Go 1.26** (gobreaker, Watermill, go-kit, stdlib)

---

## 1. Resumo Executivo

Este programa transforma um desenvolvedor em **especialista em arquitetura de microsserviços** através de desafios práticos que cobrem **27 padrões essenciais** — de resiliência e consistência de dados a deploy progressivo e observabilidade. Diferente da trilha de Design Patterns (linguagem pura), esta trilha usa **frameworks e bibliotecas do ecossistema real** de cada linguagem.

**Domínio escolhido:** **Plataforma de Mobilidade Urbana (RideFlow)** — um sistema de ride-sharing que exige todos os padrões de microsserviços de forma natural.

**Por que RideFlow (Ride-Sharing)?**
- **Resiliência crítica** — falha no serviço de matching ou pagamento afeta milhões de corridas; circuit breaker, retry e timeout são obrigatórios
- **Consistência distribuída** — reservar motorista + calcular preço + reter pagamento + iniciar corrida = saga orquestrada
- **Escala massiva** — picos de demanda (chuva, eventos) exigem rate limiting, bulkhead, backpressure e sharding por região
- **Event-driven nativo** — eventos de GPS, status de corrida, notificações, analytics são streams contínuos (Outbox + CDC)
- **Múltiplos clientes** — app do passageiro, app do motorista, painel admin = BFF + API Gateway
- **Deploy contínuo** — zero-downtime obrigatório; blue-green, canary, feature flags para experimentos de pricing
- **Observabilidade real** — SLOs de latência de matching < 3s, tracing distribuído entre 10+ serviços
- Perguntas de entrevista Staff/Principal frequentemente envolvem design de sistemas de ride-sharing

**Por que frameworks (não linguagem pura)?**
- Microsserviços em produção **exigem** frameworks — ninguém implementa circuit breaker do zero em produção
- Spring Boot + Resilience4j é o **padrão da indústria** para microsserviços Java
- Go com gobreaker + Watermill reflete o **ecossistema real** de microsserviços Go
- O contraste Java (declarativo com annotations) vs Go (explícito com composição) revela **trade-offs reais** de cada abordagem
- Dominar os frameworks **com entendimento dos padrões** (trilha Design Patterns como pré-requisito) é a combinação ideal

---

## 2. Arquitetura do Domínio RideFlow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Passenger   │     │   Driver     │     │    Admin     │
│  Mobile App  │     │  Mobile App  │     │   Dashboard  │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       ▼                    ▼                    ▼
┌──────────────────────────────────────────────────────────┐
│                    API Gateway                           │
│            (routing, auth, rate limiting)                 │
└──────┬───────────────┬───────────────────┬───────────────┘
       │               │                   │
  ┌────▼────┐    ┌─────▼─────┐     ┌──────▼──────┐
  │ Ride    │    │ Driver    │     │  Pricing    │
  │ Service │    │ Service   │     │  Service    │
  └────┬────┘    └─────┬─────┘     └──────┬──────┘
       │               │                   │
  ┌────▼────┐    ┌─────▼─────┐     ┌──────▼──────┐
  │ Payment │    │ Location  │     │Notification │
  │ Service │    │ Service   │     │  Service    │
  └─────────┘    └───────────┘     └─────────────┘
       │               │                   │
       └───────────────┴───────────────────┘
                       │
              ┌────────▼────────┐
              │   Event Bus     │
              │  (Kafka/NATS)   │
              └─────────────────┘
```

### Entidades Principais

| Entidade | Descrição | Serviço |
|----------|-----------|---------|
| `Ride` | Corrida com origem, destino, status, preço | Ride Service |
| `Passenger` | Usuário que solicita corridas | Ride Service |
| `Driver` | Motorista com localização, status, rating | Driver Service |
| `Vehicle` | Veículo do motorista (placa, modelo, categoria) | Driver Service |
| `Pricing` | Cálculo dinâmico (surge, distância, tempo) | Pricing Service |
| `Payment` | Pagamento da corrida (cartão, saldo, PIX) | Payment Service |
| `Location` | Coordenadas GPS em tempo real | Location Service |
| `Notification` | Push, SMS, e-mail para passageiro/motorista | Notification Service |
| `RideEvent` | Eventos do ciclo de vida da corrida | Event Store |

---

## 3. Mapeamento Padrão → Domínio RideFlow

| Padrão | Aplicação no RideFlow |
|--------|----------------------|
| **Circuit Breaker** | Proteger Ride Service contra falha do Payment Service ou Pricing Service |
| **Retry** | Retentar chamadas ao Location Service (GPS instável) e Notification Service |
| **Timeout** | Timeout de 3s para matching de motorista, 5s para cálculo de preço |
| **Rate Limiter** | Limitar requisições de preço por passageiro (anti-abuse), throttle no API Gateway |
| **Bulkhead** | Isolar thread pool de booking (crítico) do pool de analytics (best-effort) |
| **Composição de Resiliência** | Retry → CB → RateLimiter → Timeout → Bulkhead no Payment Service |
| **Idempotência** | Garantir que pagamento não é cobrado duas vezes; Idempotency-Key no booking |
| **Dead Letter Queue** | Notificações que falharam repetidamente vão para DLQ com metadata de erro |
| **Saga** | Orquestrar booking: reservar motorista → calcular preço → reter pagamento → confirmar corrida |
| **API Composition** | Tela "Detalhes da Corrida" agrega dados de Ride + Driver + Payment + Location |
| **CQRS** | Comando: processar corrida / Query: histórico de corridas, dashboard de analytics |
| **Event Sourcing** | Armazenar todos os eventos da corrida (requested → matched → started → completed) |
| **Outbox Pattern** | Garantir que evento RideCompleted é publicado atomicamente com atualização do banco |
| **CDC** | Capturar mudanças no banco de corridas para sincronizar com serviço de analytics |
| **Backpressure** | Controlar volume de eventos de GPS do Location Service (milhares/segundo) |
| **Sharding** | Particionar corridas por região/cidade para escala horizontal |
| **Service Discovery** | Ride Service descobre instâncias disponíveis do Driver Service e Payment Service |
| **Configuração Externa** | Surge multiplier, raio de matching, timeouts — tudo configurável sem redeploy |
| **Health Checks** | Liveness/readiness/startup de cada microsserviço com verificação de dependências |
| **API Gateway / BFF** | Gateway comum + BFF específico para app passageiro vs app motorista |
| **Sidecar** | Proxy sidecar para mTLS, observabilidade e traffic management |
| **Strangler Fig** | Migrar monólito legado RideFlow v1 para microsserviços gradualmente |
| **Blue-Green** | Deploy zero-downtime do Ride Service com rollback instantâneo |
| **Canary Release** | Deploy progressivo do novo algoritmo de pricing (5% → 25% → 50% → 100%) |
| **Feature Flags** | Habilitar "surge pricing 2.0" por cidade/percentual de usuários |
| **Shadow Deployment** | Testar novo serviço de matching espelhando tráfego real sem afetar passageiros |
| **Observabilidade** | Tracing distribuído da corrida (Gateway → Ride → Driver → Payment → Notification) |

---

## 4. Tecnologias e Frameworks

### Java 25

| Categoria | Tecnologia | Uso |
|-----------|-----------|-----|
| **Framework** | Spring Boot 3.x | Core de cada microsserviço |
| **Resiliência** | Resilience4j | Circuit Breaker, Retry, Rate Limiter, Bulkhead, TimeLimiter |
| **API Gateway** | Spring Cloud Gateway | Roteamento, filtros, rate limiting |
| **Config** | Spring Cloud Config | Configuração centralizada |
| **Discovery** | Spring Cloud Netflix Eureka / Consul | Service Registry |
| **Messaging** | Spring Kafka / Spring AMQP | Event-driven, Outbox, DLQ |
| **Persistência** | Spring Data JPA + Hibernate | CRUD, transações, Outbox table |
| **HTTP Client** | WebClient (Spring WebFlux) | Comunicação entre serviços |
| **Observabilidade** | Micrometer + OpenTelemetry | Metrics, traces, logs estruturados |
| **Feature Flags** | Togglz / FF4j | Feature flag management |
| **Testes** | JUnit 5 + Testcontainers + WireMock | Testes de integração com containers |

### Go 1.26

| Categoria | Tecnologia | Uso |
|-----------|-----------|-----|
| **HTTP** | `net/http` (stdlib) + Chi router | Endpoints REST |
| **Resiliência** | `sony/gobreaker` + custom middleware | Circuit Breaker |
| **Retry** | `avast/retry-go` | Retry com backoff exponencial |
| **Rate Limiting** | `golang.org/x/time/rate` | Token bucket rate limiter |
| **Messaging** | Watermill + `kafka-go` | Event-driven, Outbox, DLQ |
| **Persistência** | `sqlx` + `pgx` | PostgreSQL driver + query builder |
| **HTTP Client** | `net/http` + `context` | Comunicação com timeout/cancelamento |
| **Discovery** | HashiCorp Consul client | Service Registry |
| **Config** | Viper | Configuração de múltiplas fontes |
| **Observabilidade** | OpenTelemetry Go SDK | Metrics, traces, logs |
| **Feature Flags** | `thomaspoignant/go-feature-flag` | Feature flag evaluation |
| **Testes** | `testing` + `testify` + `testcontainers-go` | Testes de integração |

---

## 5. Equivalências Java ↔ Go (Framework)

| Conceito | Java 25 (Spring) | Go 1.26 |
|----------|-------------------|---------|
| **Circuit Breaker** | `@CircuitBreaker` (Resilience4j) | `gobreaker.NewCircuitBreaker()` |
| **Retry** | `@Retry` (Resilience4j) | `retry.Do()` (retry-go) |
| **Rate Limiter** | `@RateLimiter` (Resilience4j) | `rate.NewLimiter()` (x/time) |
| **Bulkhead** | `@Bulkhead` (Resilience4j) | `semaphore` channel pattern |
| **Timeout** | `@TimeLimiter` + `CompletableFuture` | `context.WithTimeout()` |
| **HTTP Server** | `@RestController` + `@RequestMapping` | `http.HandleFunc()` + Chi |
| **HTTP Client** | `WebClient` (reactive) | `http.Client{Timeout: 5*time.Second}` |
| **DI** | `@Autowired` / constructor injection | Manual constructor / `fx` (Uber) |
| **Config** | `@Value` / `@ConfigurationProperties` | `viper.GetString()` |
| **Messaging** | `@KafkaListener` / `KafkaTemplate` | `watermill.Subscribe()` / `Publish()` |
| **Health Check** | Spring Actuator `/actuator/health` | Custom `/health` endpoint |
| **Metrics** | Micrometer `@Timed` / Counter | `otel.Meter().NewCounter()` |
| **Tracing** | Micrometer Tracing + OTLP | `otel.Tracer().Start()` |
| **Feature Flag** | Togglz `FeatureManager.isActive()` | `ffclient.BoolVariation()` |
| **Transações** | `@Transactional` | `sqlx.BeginTxx()` + defer |
| **Service Discovery** | `@EnableDiscoveryClient` | `consul.Agent().ServiceRegister()` |
| **API Gateway** | Spring Cloud Gateway (filtros) | Reverse proxy (`httputil.ReverseProxy`) |

---

## 6. Mapa de Níveis

```
Level 0 — Fundações de Microsserviços & Setup do Domínio
  │       (Arquitetura, comunicação síncrona, setup projeto, entidades base)
  ▼
Level 1 — Resiliência I: Circuit Breaker, Retry & Timeout
  │       (Proteger chamadas entre serviços, fallbacks, backoff exponencial)
  ▼
Level 2 — Resiliência II: Rate Limiter, Bulkhead & Composição
  │       (Controle de taxa, isolamento de recursos, orquestração de padrões)
  ▼
Level 3 — Consistência de Dados: Idempotência, DLQ & Saga
  │       (Operações seguras, tratamento de falhas, transações distribuídas)
  ▼
Level 4 — Data Patterns: API Composition, CQRS & Event Sourcing
  │       (Agregação de dados, separação comando/query, eventos como fonte de verdade)
  ▼
Level 5 — Event-Driven: Outbox, CDC, Backpressure & Sharding
  │       (Consistência eventual, captura de mudanças, controle de fluxo, particionamento)
  ▼
Level 6 — Infraestrutura: Discovery, Config, Health, Gateway/BFF & Sidecar
  │       (Service mesh, configuração dinâmica, probes, roteamento, cross-cutting concerns)
  ▼
Level 7 — Deploy & Modernização: Strangler Fig, Blue-Green, Canary, Feature Flags & Shadow
  │       (Migração gradual, zero-downtime, deploy progressivo, experimentação)
  ▼
Level 8 — Observabilidade: Métricas, Logs, Traces & SLOs
  │       (3 pilares, dashboards, alertas baseados em SLO, error budget)
  ▼
Level 9 — Projeto Capstone: RideFlow Completo
          (Todos os padrões integrados em plataforma de mobilidade production-grade)
```

---

## 7. Estrutura de Cada Desafio

Cada nível segue a mesma estrutura:

```
level-N-nome/
├── java/                    ← Implementação Java 25 + Spring Boot
│   ├── src/
│   │   ├── main/java/
│   │   └── test/java/
│   ├── src/main/resources/
│   │   └── application.yml
│   └── pom.xml / build.gradle
└── go/                      ← Implementação Go 1.26 + libs
    ├── cmd/
    ├── internal/
    ├── go.mod
    └── *_test.go
```

### Dependências Permitidas

| Categoria | Java 25 | Go 1.26 |
|-----------|---------|---------|
| **Core** | Spring Boot 3.x | `net/http` (stdlib) + Chi |
| **Resiliência** | Resilience4j | gobreaker, retry-go, x/time/rate |
| **Messaging** | Spring Kafka | Watermill + kafka-go |
| **Persistência** | Spring Data JPA / JDBC | sqlx + pgx |
| **Config** | Spring Cloud Config | Viper |
| **Observabilidade** | Micrometer + OpenTelemetry | OpenTelemetry Go SDK |
| **Feature Flags** | Togglz / FF4j | go-feature-flag |
| **Testes** | JUnit 5, Testcontainers, WireMock | testing, testify, testcontainers-go |
| **Containers** | Docker + docker-compose | Docker + docker-compose |
| **Infra external** | PostgreSQL, Kafka, Redis, Consul | PostgreSQL, Kafka, Redis, Consul |

---

## 8. Critérios Globais

### Qualidade de Código

| Critério | Java 25 | Go 1.26 |
|----------|---------|---------|
| **Idiomático** | Spring conventions, annotations, DI | Interfaces implícitas, error handling, context |
| **Testes** | ≥ 80% cobertura + testes de integração | ≥ 80% cobertura + testes de integração |
| **Documentação** | Javadoc nos tipos públicos | GoDoc nos tipos exportados |
| **Linting** | Checkstyle + SpotBugs | `golangci-lint` |
| **Containerização** | Dockerfile multi-stage | Dockerfile multi-stage |
| **CI** | Pipeline com build + test + lint | Pipeline com build + test + lint |

### Documentação Esperada

Cada nível deve conter:
- `README.md` — Descrição do desafio com critérios de aceite
- `DECISIONS.md` — Trade-offs e justificativas de design
- `docker-compose.yml` — Infraestrutura local (DB, Kafka, etc.)
- Diagramas de arquitetura (Mermaid recomendado)

---

## 9. Referência Cruzada com Documentação

| Nível | Documentos de Referência |
|-------|--------------------------|
| Level 0 | Conceitos fundamentais de microsserviços (pré-requisito) |
| Level 1 | [01-circuit-breaker.md](../../.docs/microservice-patterns/01-circuit-breaker.md), [02-retry.md](../../.docs/microservice-patterns/02-retry.md), [03-timeout.md](../../.docs/microservice-patterns/03-timeout.md) |
| Level 2 | [04-rate-limiter.md](../../.docs/microservice-patterns/04-rate-limiter.md), [05-bulkhead.md](../../.docs/microservice-patterns/05-bulkhead.md), [06-composicao-resiliencia.md](../../.docs/microservice-patterns/06-composicao-resiliencia.md) |
| Level 3 | [07-idempotencia.md](../../.docs/microservice-patterns/07-idempotencia.md), [08-dlq.md](../../.docs/microservice-patterns/08-dlq.md), [09-saga.md](../../.docs/microservice-patterns/09-saga.md) |
| Level 4 | [10-api-composition.md](../../.docs/microservice-patterns/10-api-composition.md), [11-cqrs.md](../../.docs/microservice-patterns/11-cqrs.md), [12-event-sourcing.md](../../.docs/microservice-patterns/12-event-sourcing.md) |
| Level 5 | [13-outbox-pattern.md](../../.docs/microservice-patterns/13-outbox-pattern.md), [14-cdc.md](../../.docs/microservice-patterns/14-cdc.md), [15-sharding-partitioning.md](../../.docs/microservice-patterns/15-sharding-partitioning.md), [16-backpressure.md](../../.docs/microservice-patterns/16-backpressure.md) |
| Level 6 | [17-service-discovery.md](../../.docs/microservice-patterns/17-service-discovery.md), [18-configuracao-externa.md](../../.docs/microservice-patterns/18-configuracao-externa.md), [19-health-checks.md](../../.docs/microservice-patterns/19-health-checks.md), [20-api-gateway-bff.md](../../.docs/microservice-patterns/20-api-gateway-bff.md), [21-sidecar.md](../../.docs/microservice-patterns/21-sidecar.md) |
| Level 7 | [22-strangler-fig.md](../../.docs/microservice-patterns/22-strangler-fig.md), [23-blue-green.md](../../.docs/microservice-patterns/23-blue-green.md), [24-canary-release.md](../../.docs/microservice-patterns/24-canary-release.md), [25-feature-flags.md](../../.docs/microservice-patterns/25-feature-flags.md), [26-shadow-deployment.md](../../.docs/microservice-patterns/26-shadow-deployment.md) |
| Level 8 | [27-observabilidade.md](../../.docs/microservice-patterns/27-observabilidade.md) |
| Level 9 | Todos os documentos acima integrados |

---

## 10. Navegação

| Nível | Título | Padrões | Status |
|-------|--------|---------|--------|
| [Level 0](00-microservice-foundations.md) | Fundações de Microsserviços | Arquitetura, Comunicação, Setup | 🔲 |
| [Level 1](01-resilience-circuit-breaker-retry-timeout.md) | Resiliência I | Circuit Breaker, Retry, Timeout | 🔲 |
| [Level 2](02-resilience-rate-limiter-bulkhead-composition.md) | Resiliência II | Rate Limiter, Bulkhead, Composição | 🔲 |
| [Level 3](03-data-consistency-idempotency-dlq-saga.md) | Consistência de Dados | Idempotência, DLQ, Saga | 🔲 |
| [Level 4](04-data-patterns-composition-cqrs-eventsourcing.md) | Data Patterns | API Composition, CQRS, Event Sourcing | 🔲 |
| [Level 5](05-event-driven-outbox-cdc-backpressure-sharding.md) | Event-Driven | Outbox, CDC, Backpressure, Sharding | 🔲 |
| [Level 6](06-infrastructure-discovery-config-health-gateway-sidecar.md) | Infraestrutura | Discovery, Config, Health, Gateway/BFF, Sidecar | 🔲 |
| [Level 7](07-deployment-strangler-bluegreen-canary-flags-shadow.md) | Deploy & Modernização | Strangler Fig, Blue-Green, Canary, Feature Flags, Shadow | 🔲 |
| [Level 8](08-observability-metrics-logs-traces-slos.md) | Observabilidade | Métricas, Logs, Traces, SLOs | 🔲 |
| [Level 9](09-capstone-rideflow-platform.md) | Projeto Capstone | Todos os padrões integrados | 🔲 |

---

## 11. Pré-requisitos

| Requisito | Java | Go |
|-----------|------|----|
| **Versão** | JDK 25 | Go 1.26 |
| **Framework** | Spring Boot 3.x | Chi + Watermill |
| **Build** | Maven 3.9+ ou Gradle 8.x | `go` toolchain |
| **Containers** | Docker + docker-compose | Docker + docker-compose |
| **IDE** | IntelliJ IDEA ou VS Code | GoLand ou VS Code (gopls) |
| **Git** | Git 2.x | Git 2.x |
| **Pré-trilha** | Trilha 2 (Design Patterns) recomendada | Trilha 2 (Design Patterns) recomendada |
| **Conhecimento** | Spring Boot básico, REST APIs, SQL | Go básico, HTTP, SQL |

---

> **Legenda de status:** 🔲 Não iniciado · 🔶 Em progresso · ✅ Completo
