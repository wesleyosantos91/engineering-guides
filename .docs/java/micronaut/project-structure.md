# Project Structure — Micronaut

> **Objetivo deste documento:** Servir como referência organizacional para o GitHub Copilot e desenvolvedores.
> Foca exclusivamente na **forma de organizar pacotes, diretórios e convenções**, sem detalhar regras de negócio.
>
> **Localização recomendada:** `.github/copilot-instructions.md` (veja [Onde colocar este arquivo](#onde-colocar-este-arquivo-para-o-copilot)).

---

## Sumário

- [Project Structure — Micronaut](#project-structure--micronaut)
  - [Sumário](#sumário)
  - [Stack Tecnológica](#stack-tecnológica)
  - [Estrutura de Diretórios](#estrutura-de-diretórios)
  - [Organização dos Pacotes](#organização-dos-pacotes)
    - [api — Controllers, DTOs e Exception Handlers](#api--controllers-dtos-e-exception-handlers)
    - [core — Mappers e Validação](#core--mappers-e-validação)
    - [domain — Entities, Repositories e Services](#domain--entities-repositories-e-services)
    - [infrastructure — Configurações Técnicas](#infrastructure--configurações-técnicas)
  - [Pontos de Extensão](#pontos-de-extensão)
    - [API — REST / gRPC / GraphQL](#api--rest--grpc--graphql)
    - [Mensageria — Kafka / SQS / RabbitMQ](#mensageria--kafka--sqs--rabbitmq)
    - [Banco de Dados — MySQL / PostgreSQL / MongoDB](#banco-de-dados--mysql--postgresql--mongodb)
  - [Convenções de Código](#convenções-de-código)
    - [Nomeação](#nomeação)
    - [DTOs](#dtos)
    - [Controllers](#controllers)
    - [Entities](#entities)
  - [Configuração por Environment](#configuração-por-environment)
    - [Ativação de environment:](#ativação-de-environment)
    - [Particularidades do Environment `local`](#particularidades-do-environment-local)
    - [Exemplo de configuração:](#exemplo-de-configuração)
  - [Banco de Dados e Migrations](#banco-de-dados-e-migrations)
    - [Convenção de Migrations](#convenção-de-migrations)
  - [Testes](#testes)
    - [Estrutura](#estrutura)
    - [Convenções de Teste](#convenções-de-teste)
    - [Organização de Testes](#organização-de-testes)
    - [Exemplo de teste de controller:](#exemplo-de-teste-de-controller)
    - [Comandos de Teste](#comandos-de-teste)
  - [Build e Qualidade](#build-e-qualidade)
    - [Build Modes](#build-modes)
  - [Resiliência](#resiliência)
    - [Dependência](#dependência)
    - [Padrões Disponíveis](#padrões-disponíveis)
    - [Organização de Pacotes](#organização-de-pacotes)
    - [Configuração (application.yml)](#configuração-applicationyml)
    - [Padrão de uso no Service](#padrão-de-uso-no-service)
    - [Integração com HTTP Client](#integração-com-http-client)
    - [Fallback como bean separado](#fallback-como-bean-separado)
    - [Métricas de Resiliência](#métricas-de-resiliência)
  - [Segurança (AuthN/AuthZ)](#segurança-authnauthz)
  - [Tratamento Global de Erros](#tratamento-global-de-erros)
  - [Versionamento de API](#versionamento-de-api)
  - [Paginação e Ordenação](#paginação-e-ordenação)
  - [CORS](#cors)
  - [Feature Flags](#feature-flags)
  - [Virtual Threads](#virtual-threads)
  - [Observabilidade](#observabilidade)
  - [Docker (Native / JVM)](#docker-native--jvm)
    - [Dockerfile JVM](#dockerfile-jvm)
    - [Dockerfile Native](#dockerfile-native)
    - [Inicialização Local](#inicialização-local)
  - [Guia para Criar Novo Recurso](#guia-para-criar-novo-recurso)
    - [1. Domain](#1-domain)
    - [2. Core](#2-core)
    - [3. API (REST)](#3-api-rest)
    - [4. API (Mensageria) — se aplicável](#4-api-mensageria--se-aplicável)
    - [5. Database](#5-database)
    - [6. Testes](#6-testes)
  - [Onde colocar este arquivo para o Copilot](#onde-colocar-este-arquivo-para-o-copilot)
    - [1. Instruções no nível do repositório (recomendado)](#1-instruções-no-nível-do-repositório-recomendado)
    - [2. Instruções por workspace (VS Code settings)](#2-instruções-por-workspace-vs-code-settings)
    - [3. Instrução inline (via `.instructions.md`)](#3-instrução-inline-via-instructionsmd)
    - [Recomendação final](#recomendação-final)

---

## Stack Tecnológica

| Categoria           | Tecnologia                                               |
|---------------------|-----------------------------------------------------------|
| **Java**            | 25                                                        |
| **Micronaut**       | 4.x.x                                                    |
| **Build Tool**      | Maven (wrapper incluído)                                  |
| Web / API           | Micronaut HTTP Server (`@Controller`) — *extensível para gRPC/GraphQL* |
| Persistência        | Micronaut Data JPA (Hibernate)                            |
| Migrations          | Flyway (Micronaut Flyway)                                 |
| Validação           | Micronaut Validation (Jakarta Bean Validation)            |
| Mapeamento          | MapStruct                                                 |
| Documentação API    | Micronaut OpenAPI (Swagger UI)                            |
| HTTP Client         | Micronaut Declarative HTTP Client (`@Client`)             |
| Métricas            | Micronaut Micrometer                                      |
| Tracing             | Micronaut Tracing (OpenTelemetry)                         |
| Logging             | Logback + Logstash Encoder (JSON estruturado)             |
| Testes Unitários    | JUnit 5 + Mockito + Micronaut HTTP Client (test)          |
| Testes Integração   | `@MicronautTest` + Testcontainers + Cucumber              |
| Testes de Carga     | k6 (smoke, load, stress)                                  |
| Resiliência         | Micronaut Retry + Circuit Breaker (Retry, CircuitBreaker, Fallback, Timeout) |
| Segurança           | Micronaut Security + JWT / OAuth2                         |
| Feature Flags       | Togglz (CDI-style integration)                            |
| Virtual Threads     | Suporte nativo Micronaut 4.x (`@ExecuteOn(VIRTUAL)`)      |
| Qualidade           | JaCoCo, PIT (mutation testing)                            |

---

## Estrutura de Diretórios

```
app/
├── pom.xml
├── Dockerfile
├── docker-compose.yml
│
├── src/
│   ├── main/
│   │   ├── java/{group-id}/
│   │   │   ├── Application.java                          # Micronaut.run()
│   │   │   │
│   │   │   ├── api/                                      # ── CONTROLLERS, DTOs, EXCEPTION HANDLERS ──
│   │   │   │   ├── exception/
│   │   │   │   │   └── ApiExceptionHandler.java          # @Singleton implements ExceptionHandler
│   │   │   │   └── rest/                                 # protocolo: rest/ | grpc/ | graphql/
│   │   │   │       └── v1/                               # versionamento por path
│   │   │   │           ├── commons/enums/                # enums exclusivos da API
│   │   │   │           ├── controller/                   # @Controller (records)
│   │   │   │           ├── openapi/                      # interfaces @Tag/@Operation (Swagger)
│   │   │   │           ├── request/                      # records de entrada (DTOs)
│   │   │   │           └── response/                     # records de saída (DTOs)
│   │   │   │
│   │   │   ├── core/                                     # ── MAPPERS E VALIDAÇÃO ──
│   │   │   │   ├── mapper/                               # MapStruct mappers
│   │   │   │   └── validation/                           # grupos de validação
│   │   │   │
│   │   │   ├── domain/                                   # ── ENTITIES, REPOSITORIES, SERVICES ──
│   │   │   │   ├── entity/                               # @Entity JPA
│   │   │   │   │   └── enums/                            # enums persistidos
│   │   │   │   ├── exception/                            # exceções
│   │   │   │   ├── repository/                           # tecnologia do banco como prefixo
│   │   │   │   │   └── mysql/                            # mysql/ | postgresql/ | mongodb/
│   │   │   │   └── service/                              # @Singleton
│   │   │   │
│   │   │   └── infrastructure/                           # ── CONFIGURAÇÕES TÉCNICAS ──
│   │   │       ├── metrics/
│   │   │       ├── openapi/
│   │   │       ├── resilience/                            # Retry/CircuitBreaker configs
│   │   │       ├── security/                              # Micronaut Security / JWT config
│   │   │       ├── cors/                                  # CORS config
│   │   │       ├── featureflag/                           # Togglz feature flags
│   │   │       └── messaging/                            # kafka/ | sqs/ | rabbitmq/
│   │   │
│   │   └── resources/
│   │       ├── application.yml                           # config padrão (produção)
│   │       ├── application-local.yml                     # config local (dev)
│   │       ├── logback.xml                               # logging estruturado (JSON)
│   │       └── db/
│   │           ├── migration/                            # DDL — Flyway (V{n}.0__)
│   │           └── testdata/                             # DML — seeds (somente environment local)
│   │
│   └── test/
│       ├── java/{group-id}/
│       │   ├── api/rest/v1/controller/                   # @MicronautTest (unit)
│       │   ├── domain/
│       │   │   ├── entity/                               # TemplateLoader (fixtures)
│       │   │   ├── request/                              # TemplateLoader (fixtures)
│       │   │   ├── response/                             # TemplateLoader (fixtures)
│       │   │   └── service/                              # unit + @ParameterizedTest
│       │   ├── cucumber/                                 # BDD integration
│       │   │   ├── config/
│       │   │   ├── step/
│       │   │   └── utils/
│       │   └── infrastructure/                           # Testcontainers base
│       └── resources/
│           ├── application-test.yml
│           └── features/                                 # .feature (Cucumber)
```

---

## Organização dos Pacotes

O projeto é organizado em **4 pacotes raiz**, cada um com uma responsabilidade clara:

```
{group-id}
├── api/                       → Controllers, DTOs (request/response), exception handlers
│   ├── rest/v1/               → REST API versionada (ou grpc/, graphql/)
│   └── messaging/kafka/       → Consumers e producers Kafka (ou sqs/, rabbitmq/)
├── core/                      → Mappers (MapStruct) e grupos de validação
├── domain/                    → Entities, repositories, services, exceções
│   └── repository/mysql/      → Repositories com prefixo da tecnologia de banco
└── infrastructure/            → Configurações técnicas (métricas, docs, clients, messaging)
```

> **Nota:** Essa organização é uma decisão prática de pacotes — não se vincula a nenhum padrão arquitetural específico (Clean Architecture, Hexagonal, etc.).

### api — Controllers, DTOs e Exception Handlers

Pacote responsável por **receber requisições** e **devolver respostas**.

| Pacote                        | Responsabilidade                                                     | Convenção                              |
|-------------------------------|----------------------------------------------------------------------|----------------------------------------|
| `api.exception`               | Tratamento global de exceções (`@Singleton ExceptionHandler<T, R>`)  | Uma classe por tipo de exceção          |
| `api.rest.v1.controller`      | Controllers Micronaut (`@Controller` como `record`)                  | Um controller por recurso              |
| `api.rest.v1.openapi`         | Interfaces com anotações OpenAPI (`@Tag`, `@Operation`)              | Uma interface por controller           |
| `api.rest.v1.request`         | DTOs de entrada (`record` com Bean Validation)                       | Sufixo `Request`                       |
| `api.rest.v1.response`        | DTOs de saída (`record`)                                             | Sufixo `Response`                      |
| `api.rest.v1.commons.enums`   | Enums exclusivos da API                                              | Espelham domain enums quando necessário |

**Padrões adotados:**

- **Protocolo como pacote**: `api.rest.`, `api.grpc.`, `api.graphql.`
- **Versionamento por path**: `/v1/`, `/v2/`, etc.
- **Controllers como `record`**: Imutabilidade + injeção pelo construtor canônico (compile-time DI).
- **Interface OpenAPI separada**: Mantém annotations Swagger fora do controller.
- **`ProblemDetail`-like error response**: Padrão de resposta de erro via `ExceptionHandler`.

### core — Mappers e Validação

Código **reutilizável** entre pacotes.

| Pacote              | Responsabilidade                              | Convenção                           |
|---------------------|-----------------------------------------------|-------------------------------------|
| `core.mapper`       | MapStruct mappers (request ↔ entity ↔ response) | Sufixo `Mapper`, interface + `MAPPER` singleton |
| `core.validation`   | Grupos de validação (Bean Validation groups)  | Interface marker (`Create`, `Update`) |

**Padrão de mapper:**

```java
@Mapper(componentModel = "jsr330", nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface FooMapper {

    FooEntity toEntity(FooRequest request);
    FooResponse toResponse(FooEntity entity);
    void update(FooRequest request, @MappingTarget FooEntity entity);
    default Page<FooResponse> toPageResponse(Page<FooEntity> page) {
        return page.map(this::toResponse);
    }
}
```

> No Micronaut, use `componentModel = "jsr330"` para que o mapper seja um bean injetável via `@Inject`. Como o DI é em compile-time, MapStruct se integra naturalmente.

### domain — Entities, Repositories e Services

Contém **entidades, repositórios, serviços e exceções**.

| Pacote                      | Responsabilidade                              | Convenção                             |
|-----------------------------|-----------------------------------------------|---------------------------------------|
| `domain.entity`             | JPA entities (`@Entity`)                      | Sufixo `Entity`, tabela `tb_*`       |
| `domain.entity.enums`       | Enums persistidos (com `fromValue()` estático)| Enum com `HashMap` lookup             |
| `domain.repository.mysql`   | Repositories para MySQL                       | Sufixo `Repository`, `JpaRepository<Entity, UUID>` |
| `domain.repository.postgresql` | Repositories para PostgreSQL (se aplicável)| Mesmo padrão                          |
| `domain.repository.mongodb` | Repositories para MongoDB (se aplicável)      | `MongoRepository<Document, String>`   |
| `domain.service`            | Serviços (`@Singleton`)                       | Sufixo `Service`, injeção via construtor |
| `domain.exception`          | Exceções específicas                          | `ResourceNotFoundException`, `BusinessException` |

**Padrões adotados:**

- **Tecnologia do banco como pacote**: `domain.repository.mysql.`, `domain.repository.postgresql.`, `domain.repository.mongodb.`
- **Chaves primárias UUID** com `@GeneratedValue(strategy = GenerationType.UUID)`.
- **Relacionamentos LAZY** entre entidades.
- **`@Transactional`** (de `jakarta.transaction.Transactional`) nos services para escrita; **`@ReadOnly`** (de `io.micronaut.transaction.annotation`) para leituras.
- **`@Timed` e `@Counted`** (Micrometer) nos métodos de service para métricas.
- **Micronaut Data**: Repositories estendem `JpaRepository` com query derivation em compile-time.

### infrastructure — Configurações Técnicas

Configurações de infraestrutura e integrações técnicas.

| Pacote                             | Responsabilidade                         | Convenção                    |
|------------------------------------|------------------------------------------|------------------------------|
| `infrastructure.metrics`           | Beans de métricas customizados           | Sufixo `Config`, `@Factory`  |
| `infrastructure.openapi`           | Config OpenAPI/Swagger                   | Ativo apenas em `local`      |
| `infrastructure.messaging.kafka`   | Config Kafka                             | Sufixo `Config`              |
| `infrastructure.messaging.sqs`     | Config SQS                               | Sufixo `Config`              |
| `infrastructure.messaging.rabbitmq`| Config RabbitMQ                          | Sufixo `Config`              |
| `infrastructure.resilience`        | Config Retry/CircuitBreaker               | Sufixo `Config`, `@Factory`  |
| `infrastructure.security`          | Micronaut Security / JWT config           | Sufixo `Config`, `@Factory`  |
| `infrastructure.cors`              | CORS configuration                        | Sufixo `Config`, `@Factory`  |
| `infrastructure.featureflag`       | Togglz feature flags                      | Sufixo `Config`, `@Factory`  |

**Extensões esperadas nesta camada:**

- `infrastructure.client` — HTTP Clients declarativos via `@Client`.
- `infrastructure.cache` — Configuração de cache (`@Cacheable`, Redis/Caffeine).

---

## Pontos de Extensão

### API — REST / gRPC / GraphQL

A estrutura permite múltiplos protocolos sob o pacote `api`:

```
api/
├── exception/                       # compartilhado entre protocolos
│   └── ApiExceptionHandler.java
├── rest/                            # ── REST ──
│   └── v1/
│       ├── controller/
│       ├── openapi/
│       ├── request/
│       └── response/
├── grpc/                            # ── gRPC ──
│   ├── interceptor/
│   ├── service/
│   └── proto/
└── graphql/                         # ── GraphQL ──
    ├── resolver/
    ├── input/
    └── type/
```

| Protocolo  | Dependência                                              |
|------------|----------------------------------------------------------|
| REST       | `micronaut-http-server-netty`                            |
| gRPC       | `micronaut-grpc-server-runtime`                          |
| GraphQL    | `micronaut-graphql`                                      |

**Fluxo esperado** — Independente do protocolo, o pacote `api` segue o mesmo padrão:
1. Recebe a requisição (REST request, gRPC message, GraphQL query/mutation)
2. Converte para objetos internos (via `core.mapper`)
3. Delega ao `domain.service`
4. Converte e retorna a resposta

### Mensageria — Kafka / SQS / RabbitMQ

A estrutura de mensageria usa o **nome da tecnologia** como pacote:

```
api/messaging/
├── kafka/                           # ── Apache Kafka ──
│   ├── consumer/                    # @KafkaListener
│   ├── producer/                    # @KafkaClient
│   └── event/                       # DTOs/records dos eventos
├── sqs/                             # ── AWS SQS ──
│   ├── consumer/                    # SqsClient consumer
│   ├── producer/                    # SqsClient producer
│   └── event/
└── rabbitmq/                        # ── RabbitMQ ──
    ├── consumer/                    # @RabbitListener
    ├── producer/                    # @RabbitClient
    └── event/

infrastructure/messaging/
├── kafka/
│   └── KafkaConfig.java
├── sqs/
│   └── SqsConfig.java
└── rabbitmq/
    └── RabbitConfig.java
```

| Broker     | Dependência                                               |
|------------|-----------------------------------------------------------|
| Kafka      | `micronaut-kafka`                                         |
| SQS        | `micronaut-aws-sdk-v2` + SQS                              |
| RabbitMQ   | `micronaut-rabbitmq`                                       |

**Padrão de consumer (Kafka):**

```java
@KafkaListener(groupId = "${app.kafka.consumer.group-id}")
public class TransactionConsumer {

    private final TransactionService transactionService;

    TransactionConsumer(TransactionService transactionService) {
        this.transactionService = transactionService;
    }

    @Topic("${app.kafka.topics.transaction-created}")
    public void consume(TransactionCreatedEvent event) {
        transactionService.process(event);
    }
}
```

**Padrão de producer (Kafka):**

```java
@KafkaClient
public interface NotificationProducer {

    @Topic("notification-request")
    void send(@KafkaKey String key, NotificationRequestEvent event);
}
```

> No Micronaut, producers Kafka são **interfaces declarativas** anotadas com `@KafkaClient` — o framework gera a implementação em compile-time. Consumers usam `@KafkaListener` + `@Topic`.

### Banco de Dados — MySQL / PostgreSQL / MongoDB

Os repositories são organizados com **prefixo da tecnologia de banco**:

```
domain/repository/
├── mysql/                           # ── MySQL ──
│   ├── UserRepository.java         # JpaRepository<UserEntity, UUID>
│   ├── WalletRepository.java
│   └── TransactionRepository.java
├── postgresql/                      # ── PostgreSQL ──
│   └── AuditRepository.java        # JpaRepository<AuditEntity, UUID>
└── mongodb/                         # ── MongoDB ──
    └── EventLogRepository.java      # MongoRepository<EventLogDocument, String>
```

| Banco       | Dependência                                          | Interface base                     |
|-------------|------------------------------------------------------|------------------------------------|
| MySQL       | `micronaut-data-hibernate-jpa` + `mysql-connector-j` | `JpaRepository<Entity, UUID>`      |
| PostgreSQL  | `micronaut-data-hibernate-jpa` + `postgresql`         | `JpaRepository<Entity, UUID>`      |
| MongoDB     | `micronaut-data-mongodb`                              | `MongoRepository<Document, String>`|

**Motivo:** quando a aplicação lida com múltiplos bancos (ex: MySQL para dados transacionais + MongoDB para logs de eventos), o prefixo torna explícito qual datasource cada repository utiliza.

**Exemplo de repository Micronaut Data:**

```java
@Repository
public interface UserRepository extends JpaRepository<UserEntity, UUID> {

    Optional<UserEntity> findByEmail(String email);

    Page<UserEntity> findAll(Pageable pageable);
}
```

> Micronaut Data gera as queries em **compile-time** (sem reflection), garantindo startup rápido e validação antecipada.

---

## Convenções de Código

### Nomeação

| Elemento           | Convenção                                    | Exemplo                              |
|--------------------|----------------------------------------------|--------------------------------------|
| Pacote             | `lowercase` sem underscores                  | `api.rest.v1.controller`             |
| Classe             | `PascalCase`                                 | `UserController`                     |
| Controller         | `{Recurso}Controller`                        | `UserController`                     |
| Service            | `{Recurso}Service`                           | `UserService`                        |
| Repository         | `{Recurso}Repository`                        | `UserRepository`                     |
| Entity             | `{Recurso}Entity`                            | `UserEntity`                         |
| Mapper             | `{Recurso}Mapper`                            | `UserMapper`                         |
| Request DTO        | `{Recurso}Request` ou `{Recurso}QueryRequest`| `UserRequest`, `UserQueryRequest`    |
| Response DTO       | `{Recurso}Response`                          | `UserResponse`                       |
| OpenAPI interface  | `{Recurso}OpenApi`                           | `UserOpenApi`                        |
| Consumer           | `{Recurso}Consumer`                          | `TransactionConsumer`                |
| Producer           | `{Recurso}Producer`                          | `NotificationProducer`               |
| Event              | `{Recurso}{Ação}Event`                       | `TransactionCreatedEvent`            |
| Tabela             | `tb_{recurso}` (snake_case)                  | `tb_user`, `tb_wallet`               |
| Migration DDL      | `V{n}.0__{descricao}.sql`                    | `V2.0__create_tb_user.sql`           |
| Migration seed     | `V{ano.mes.dia.seq}.0__seed_{tabela}_data.sql` | `V2024.11.08.1.0__seed_tb_user_data.sql` |

### DTOs

- **Usar `record`** para todos os DTOs (request e response).
- **Validação** com Jakarta Bean Validation annotations nos campos do request.
- **`@Introspected`**: Anotar records com `@Introspected` para habilitar serialização/validação sem reflection.
- **Grupos de validação** (`Groups.Create`, `Groups.Update`) para regras condicionais.

### Controllers

- **Implementados como `record`**: Garante imutabilidade e injeção de dependências em compile-time.
- **Implementam interface OpenAPI**: Mantém documentação separada da implementação.
- **Retornam `HttpResponse<T>`**: Permite controle explícito de headers e status codes.
- **Anotações Micronaut**: `@Controller`, `@Get`, `@Post`, `@Put`, `@Delete`, `@Produces`, `@Consumes`.

### Entities

- **UUID como chave primária** em todas as entidades.
- **Fetch LAZY** em todos os relacionamentos.
- **Enums com `fromValue()`**: Lookup via `HashMap` estático para conversão segura.

---

## Configuração por Environment

O Micronaut usa **environments** para gerenciar configurações:

| Environment  | Arquivo                       | Uso                                          |
|--------------|-------------------------------|----------------------------------------------|
| *(default)*  | `application.yml`             | Produção — configs via variáveis de ambiente |
| `local`      | `application-local.yml`       | Desenvolvimento local — seeds, Swagger       |
| `test`       | `application-test.yml`        | Testes — Testcontainers injeta configs dinamicamente |

### Ativação de environment:

```bash
# Via variável de ambiente
MICRONAUT_ENVIRONMENTS=local ./mvnw mn:run

# Via system property
./mvnw mn:run -Dmicronaut.environments=local
```

### Particularidades do Environment `local`

- **Swagger UI** ativo
- **Flyway**: carrega seeds de `db/testdata/` além de `db/migration/`
- **Log em texto puro** (sem JSON) para facilitar leitura

### Exemplo de configuração:

```yaml
# application.yml (produção)
micronaut:
  application:
    name: my-app
  server:
    port: 8080

datasources:
  default:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASS}
    dialect: MYSQL
    driver-class-name: com.mysql.cj.jdbc.Driver

jpa:
  default:
    properties:
      hibernate:
        hbm2ddl:
          auto: none

flyway:
  datasources:
    default:
      enabled: true
      locations:
        - classpath:db/migration

# application-local.yml
flyway:
  datasources:
    default:
      locations:
        - classpath:db/migration
        - classpath:db/testdata
```

---

## Banco de Dados e Migrations

### Convenção de Migrations

- **DDL (schema)**: Em `db/migration/`, prefixo `V{n}.0__`
- **DML (seeds)**: Em `db/testdata/`, prefixo `V{ano.mes.dia.seq}.0__seed_`
- Seeds são carregados **somente no environment `local`**
- Hibernate DDL auto: `none` — Flyway é o único responsável pelo schema

---

## Testes

### Estrutura

| Tipo                | Runner                | Diretório                  | Sufixo         |
|---------------------|-----------------------|----------------------------|----------------|
| Unitário            | Surefire (`test`)     | `src/test/java/.../`       | `*Test.java`   |
| Integração (BDD)    | Failsafe (`verify`)   | `src/test/java/.../cucumber/` | `*IT.java`  |
| Carga               | k6 (externo)          | `src/main/resources/k6/`  | `.js`          |

### Convenções de Teste

- **Template Loaders**: Classes `*TemplateLoader` para criar fixtures de dados reutilizáveis.
- **`@MicronautTest`**: Para testes que carregam o contexto de DI completo.
- **Micronaut HTTP Client**: `@Client("/")` para testar endpoints declarativamente.
- **`@ParameterizedTest`**: Com `@ValueSource` para testar variações de input.
- **Testcontainers**: Container do banco compartilhado via base class e `@TestPropertyProvider`.
- **Cucumber**: Features em `src/test/resources/features/`, steps em `cucumber.step`.

### Organização de Testes

```
test/
├── java/{group-id}/
│   ├── api/rest/v1/controller/
│   │   └── FooControllerTest.java           # @MicronautTest + @Client
│   ├── domain/
│   │   ├── entity/
│   │   │   └── FooEntityTemplateLoader.java # fixture
│   │   ├── request/
│   │   │   └── FooRequestTemplateLoader.java
│   │   ├── response/
│   │   │   └── FooResponseTemplateLoader.java
│   │   └── service/
│   │       └── FooServiceTest.java          # unit
│   ├── cucumber/
│   │   ├── CucumberIT.java                  # runner
│   │   ├── config/
│   │   ├── step/
│   │   └── utils/
│   └── infrastructure/
│       └── ContainerBaseTest.java           # Testcontainers base
└── resources/
    ├── application-test.yml
    └── features/
        └── *.feature
```

### Exemplo de teste de controller:

```java
@MicronautTest
class FooControllerTest {

    @Inject
    @Client("/")
    HttpClient client;

    @Test
    void shouldReturnFoo() {
        var response = client.toBlocking()
                .exchange(HttpRequest.GET("/v1/foos/123"), FooResponse.class);

        assertEquals(HttpStatus.OK, response.status());
        assertNotNull(response.body());
    }
}
```

### Comandos de Teste

```bash
# Testes unitários
./mvnw test

# Testes de integração (pula unitários)
./mvnw verify -Dskip.ut=true -Dskip.it=false

# Testes de mutação (PIT)
./mvnw clean test-compile org.pitest:pitest-maven:mutationCoverage
```

---

## Build e Qualidade

| Plugin                       | Fase            | Função                            |
|------------------------------|-----------------|-----------------------------------|
| `micronaut-maven-plugin`     | package         | Gera JAR executável               |
| `maven-surefire-plugin`      | test            | Roda testes unitários (`*Test`)   |
| `maven-failsafe-plugin`      | verify          | Roda testes de integração (`*IT`) |
| `jacoco-maven-plugin`        | test/verify     | Cobertura de código               |
| `pitest-maven`               | manual          | Testes de mutação                 |

### Build Modes

```bash
# JVM mode
./mvnw package

# Native mode (GraalVM)
./mvnw package -Dpackaging=native-image

# Docker image (JVM)
./mvnw package -Dpackaging=docker

# Docker image (Native)
./mvnw package -Dpackaging=docker-native
```

---

## Resiliência

O projeto utiliza as **annotations nativas do Micronaut** (`micronaut-retry`) para padrões de resiliência em chamadas a serviços externos.

### Dependência

```xml
<dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-retry</artifactId>
</dependency>
```

### Padrões Disponíveis

| Padrão              | Anotação / API                   | Quando usar                                           |
|---------------------|----------------------------------|-------------------------------------------------------|
| **Retry**           | `@Retryable`                     | Falhas transitórias (timeout, 503)                    |
| **Circuit Breaker** | `@CircuitBreaker`                | Chamadas a serviços externos instáveis                |
| **Fallback**        | `@Fallback`                      | Resposta alternativa quando todos falham               |
| **Timeout**         | `@Timeout` (Reactive)            | Definir tempo máximo para chamadas                    |
| **Rate Limiter**    | `@RateLimiter` (Micronaut 4+)    | Proteger endpoints contra excesso de requisições      |

### Organização de Pacotes

```
infrastructure/resilience/
├── RetryConfig.java                  # @Factory — retry event listeners customizados
└── CircuitBreakerEventListener.java  # Listener para logging de eventos
```

### Configuração (application.yml)

```yaml
micronaut:
  retry:
    delay: 500ms
    attempts: 3
    multiplier: 1.5

# Configuração por bean (via annotation attributes ou config)
# Não há arquivo centralizado como Resilience4j —
# a configuração é feita diretamente nas annotations.
```

### Padrão de uso no Service

```java
@Singleton
public class PaymentService {

    private final PaymentClient paymentClient;

    PaymentService(PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }

    @CircuitBreaker(reset = "30s", delay = "5s", attempts = 3, throwWrappedException = true)
    @Retryable(attempts = "3", delay = "500ms", multiplier = "1.5")
    public PaymentResponse charge(PaymentRequest request) {
        return paymentClient.charge(request);
    }

    @Fallback
    public PaymentResponse fallbackCharge(PaymentRequest request) {
        LOG.warn("Fallback acionado para payment charge");
        return PaymentResponse.unavailable();
    }
}
```

> **Ordem de execução**: `Fallback → Retry → CircuitBreaker → chamada real`.

### Integração com HTTP Client

Fault Tolerance pode ser aplicado diretamente na interface `@Client`:

```java
@Client("payment-api")
public interface PaymentClient {

    @Post("/charges")
    @Retryable(attempts = "3", delay = "500ms")
    @CircuitBreaker(reset = "30s")
    PaymentResponse charge(@Body PaymentRequest request);
}
```

### Fallback como bean separado

Para fallbacks complexos, use uma classe `@Fallback` que implementa a mesma interface:

```java
@Fallback
@Singleton
public class PaymentClientFallback implements PaymentClient {

    @Override
    public PaymentResponse charge(PaymentRequest request) {
        return PaymentResponse.unavailable();
    }
}
```

### Métricas de Resiliência

Com `micronaut-micrometer`, eventos de retry e circuit breaker são observáveis:
- Retry events via `RetryEventListener`
- Circuit breaker state changes via `CircuitBreakerEventListener`
- Métricas customizadas podem ser adicionadas nos listeners com `MeterRegistry`

---

## Segurança (AuthN/AuthZ)

O projeto usa **Micronaut Security** com **JWT** para autenticação stateless.

### Dependências

```xml
<dependency>
    <groupId>io.micronaut.security</groupId>
    <artifactId>micronaut-security</artifactId>
</dependency>
<dependency>
    <groupId>io.micronaut.security</groupId>
    <artifactId>micronaut-security-jwt</artifactId>
</dependency>
<!-- Opcional: OAuth2 / OpenID Connect -->
<dependency>
    <groupId>io.micronaut.security</groupId>
    <artifactId>micronaut-security-oauth2</artifactId>
</dependency>
```

### Organização de Pacotes

```
infrastructure/security/
├── SecurityConfig.java              # @Factory — customizações de segurança
├── RolesConstants.java              # Constantes de roles
└── JwtClaimExtractor.java           # Utilitário para extrair claims customizados
```

### Configuração (application.yml)

```yaml
micronaut:
  security:
    enabled: true
    authentication: bearer
    token:
      jwt:
        signatures:
          jwks:
            issuer:
              url: ${JWK_SET_URI}
        claims-validators:
          issuer: ${JWT_ISSUER_URI}
    intercept-url-map:
      - pattern: /health/**
        http-method: GET
        access:
          - isAnonymous()
      - pattern: /swagger-ui/**
        http-method: GET
        access:
          - isAnonymous()
      - pattern: /v1/**
        access:
          - isAuthenticated()
```

### RBAC (Role-Based Access Control)

```java
@Controller("/v1/users")
@Secured(SecurityRule.IS_AUTHENTICATED)
public record UserController(UserService userService) {

    @Delete("/{id}")
    @Secured({"ADMIN"})
    public HttpResponse<Void> delete(@PathVariable UUID id) {
        userService.delete(id);
        return HttpResponse.noContent();
    }

    @Get
    @Secured({"ADMIN", "MANAGER"})
    public List<UserResponse> findAll() {
        return userService.findAll();
    }
}
```

| Anotação                        | Uso                                        |
|---------------------------------|--------------------------------------------|
| `@Secured(IS_AUTHENTICATED)`   | Requer token válido (qualquer role)        |
| `@Secured({"ADMIN"})`          | Requer role específica                     |
| `@Secured(IS_ANONYMOUS)`       | Permite acesso sem autenticação            |
| `@Secured(DENY_ALL)`           | Bloqueia acesso total                      |

> **Diferencial Micronaut:** A segurança é processada em **compile-time** — sem reflection ou proxies em runtime, resultando em startup mais rápido.

---

## Tratamento Global de Erros

### Padrão: ExceptionHandler (compile-time)

No Micronaut, o tratamento global de exceções é feito com `ExceptionHandler<T, R>`, resolvido em compile-time.

### Organização

```
api/exception/
├── ApiExceptionHandler.java           # @Singleton implements ExceptionHandler — handler global
├── ProblemDetailFactory.java          # Factory para resposta padronizada
├── NotFoundExceptionHandler.java      # Handler específico para 404
└── ErrorCode.java                     # enum com códigos de erro do domínio
```

### Padrão de ExceptionHandler

```java
@Singleton
@Requires(classes = ResourceNotFoundException.class)
public class NotFoundExceptionHandler implements ExceptionHandler<ResourceNotFoundException, HttpResponse<Map<String, Object>>> {

    @Override
    public HttpResponse<Map<String, Object>> handle(HttpRequest request, ResourceNotFoundException exception) {
        var problem = Map.of(
            "type", "https://api.example.com/errors/not-found",
            "title", "Recurso não encontrado",
            "status", 404,
            "detail", exception.getMessage(),
            "errorCode", "RESOURCE_NOT_FOUND",
            "timestamp", Instant.now().toString(),
            "instance", request.getPath()
        );
        return HttpResponse.<Map<String, Object>>notFound().body(problem);
    }
}
```

### Handler Genérico (catch-all)

```java
@Singleton
@Requires(classes = Exception.class)
@Order(Ordered.LOWEST_PRECEDENCE)
public class GlobalExceptionHandler implements ExceptionHandler<Exception, HttpResponse<Map<String, Object>>> {

    private static final Logger LOG = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @Override
    public HttpResponse<Map<String, Object>> handle(HttpRequest request, Exception exception) {
        LOG.error("Erro não tratado: {}", exception.getMessage(), exception);
        var problem = Map.of(
            "type", "https://api.example.com/errors/internal",
            "title", "Erro interno",
            "status", 500,
            "detail", "Erro inesperado",
            "errorCode", "INTERNAL_ERROR",
            "timestamp", Instant.now().toString()
        );
        return HttpResponse.<Map<String, Object>>serverError().body(problem);
    }
}
```

### Formato de Resposta (JSON)

```json
{
  "type": "https://api.example.com/errors/not-found",
  "title": "Recurso não encontrado",
  "status": 404,
  "detail": "Usuário com ID 550e8400-e29b-41d4-a716-446655440000 não encontrado",
  "errorCode": "RESOURCE_NOT_FOUND",
  "timestamp": "2026-02-22T10:30:00Z",
  "instance": "/v1/users/550e8400-e29b-41d4-a716-446655440000"
}
```

### Mapeamento de Exceções

| Exceção                        | Status HTTP | errorCode              | Handler                        |
|--------------------------------|-------------|------------------------|--------------------------------|
| `ResourceNotFoundException`    | 404         | `RESOURCE_NOT_FOUND`   | `NotFoundExceptionHandler`     |
| `BusinessException`           | 422         | Dinâmico (do domínio)  | `BusinessExceptionHandler`     |
| `ConstraintViolationException`| 400         | `VALIDATION_ERROR`     | Built-in Micronaut             |
| `AuthorizationException`      | 403         | `ACCESS_DENIED`        | Built-in Micronaut Security    |
| `AuthenticationException`     | 401         | `UNAUTHORIZED`         | Built-in Micronaut Security    |
| `Exception` (genérica)        | 500         | `INTERNAL_ERROR`       | `GlobalExceptionHandler`       |

> **Diferencial Micronaut:** `ExceptionHandler` é resolvido em **compile-time** — sem overhead de reflection.

---

## Versionamento de API

### Estratégia: Versionamento por Path

```
/v1/users          → versão 1
/v2/users          → versão 2
```

### Organização de Pacotes

```
api/rest/
├── v1/
│   ├── controller/
│   ├── request/
│   ├── response/
│   └── openapi/
└── v2/
    ├── controller/
    ├── request/
    ├── response/
    └── openapi/
```

### Regras

- **Cada versão tem seus próprios DTOs** (request/response) — não compartilhar entre versões.
- **Services e entities são compartilhados** — a versão só afeta a camada `api`.
- **Mappers podem ser versionados** se a conversão mudar: `FooMapperV1`, `FooMapperV2`.
- **Deprecation**: Versões antigas devem ter header `Deprecated: true` e `Sunset` date.
- **Mínimo de versões ativas**: Manter no máximo 2 versões simultâneas.

---

## Paginação e Ordenação

### Convenção de Query Parameters

| Parâmetro  | Tipo     | Default  | Exemplo                      |
|------------|----------|----------|------------------------------|
| `page`     | `int`    | `0`      | `?page=0`                    |
| `size`     | `int`    | `20`     | `?size=10`                   |
| `sort`     | `string` | —        | `?sort=name,asc`             |

### Formato de Resposta Paginada

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

### Padrão no Controller

```java
@Controller("/v1/users")
@Secured(SecurityRule.IS_AUTHENTICATED)
public record UserController(UserService userService) {

    @Get
    public HttpResponse<Page<UserResponse>> findAll(
            @QueryValue(defaultValue = "0") int page,
            @QueryValue(defaultValue = "20") int size,
            @Nullable @QueryValue String sort) {

        var pageable = Pageable.from(page, size, parseSort(sort));
        return HttpResponse.ok(userService.findAll(pageable));
    }
}
```

### Padrão no Service

```java
@Singleton
public class UserService {

    private final UserRepository userRepository;
    private final UserMapper userMapper;

    @ReadOnly
    public Page<UserResponse> findAll(Pageable pageable) {
        return userRepository.findAll(pageable)
                .map(userMapper::toResponse);
    }
}
```

### Limites (application.yml)

```yaml
app:
  pagination:
    default-size: 20
    max-size: 100
```

> **Micronaut Data JPA** suporta `Pageable` e `Page<T>` nativamente em repositories que estendem `PageableRepository`.

---

## CORS

### Configuração (application.yml)

```yaml
micronaut:
  server:
    cors:
      enabled: true
      configurations:
        default:
          allowed-origins:
            - ${APP_CORS_ORIGINS:https://app.example.com}
          allowed-methods:
            - GET
            - POST
            - PUT
            - DELETE
            - PATCH
            - OPTIONS
          allowed-headers:
            - Content-Type
            - Authorization
            - Accept
          exposed-headers:
            - Location
            - X-Total-Count
          allow-credentials: true
          max-age: 3600
```

### Configuração por Environment

```yaml
# application.yml (produção)
micronaut:
  server:
    cors:
      configurations:
        default:
          allowed-origins:
            - https://app.example.com

# application-local.yml
micronaut:
  server:
    cors:
      configurations:
        default:
          allowed-origins:
            - http://localhost:3000
            - http://localhost:5173
```

### Organização de Pacotes

```
infrastructure/cors/
└── CorsConfig.java              # @Factory — apenas se precisar de config programática
```

> **Nota:** No Micronaut, CORS é configurado em `application.yml`. Use `@Factory` somente para lógica dinâmica (ex: origins de banco de dados).

---

## Feature Flags

### Biblioteca: Togglz

```xml
<dependency>
    <groupId>org.togglz</groupId>
    <artifactId>togglz-core</artifactId>
</dependency>
```

### Organização de Pacotes

```
infrastructure/featureflag/
├── FeatureFlagConfig.java            # @Factory — FeatureManager producer
└── Features.java                     # enum com todas as feature flags
```

### Definição das Features

```java
public enum Features implements Feature {

    @Label("Novo fluxo de pagamento")
    NEW_PAYMENT_FLOW,

    @Label("Notificação push")
    PUSH_NOTIFICATION,

    @Label("Relatório exportável")
    EXPORTABLE_REPORT;

    public boolean isActive() {
        return FeatureContext.getFeatureManager().isActive(this);
    }
}
```

### Factory para Micronaut

```java
@Factory
public class FeatureFlagConfig {

    @Singleton
    public FeatureManager featureManager() {
        return new FeatureManagerBuilder()
            .featureEnum(Features.class)
            .stateRepository(new EnvVariableStateRepository())
            .build();
    }
}
```

### Uso no Service

```java
@Singleton
public class PaymentService {

    private final PaymentRepository repository;

    public PaymentResponse process(PaymentRequest request) {
        if (Features.NEW_PAYMENT_FLOW.isActive()) {
            return processV2(request);
        }
        return processV1(request);
    }
}
```

> **Boas práticas:** Remova flags antigas assim que estiverem 100% habilitadas. Mantenha max ~10 flags ativas simultâneas.

---

## Virtual Threads

### Configuração (Micronaut 4.x + JDK 25)

Micronaut 4.x suporta virtual threads de forma **explícita por endpoint**.

### Uso no Controller

```java
@Controller("/v1/reports")
@Secured(SecurityRule.IS_AUTHENTICATED)
public record ReportController(ReportService reportService) {

    @Get
    @ExecuteOn(TaskExecutors.VIRTUAL)
    public List<ReportResponse> generate() {
        return reportService.generateAll();  // operação blocking pesada
    }
}
```

### Configuração Global via application.yml

```yaml
micronaut:
  server:
    thread-selection: AUTO        # ou MANUAL para controle explícito
  executors:
    io:
      type: VIRTUAL               # executor "io" usando virtual threads
```

### Impacto

| Componente          | Efeito                                                          |
|---------------------|-----------------------------------------------------------------|
| **HTTP Server**     | Endpoint roda em virtual thread com `@ExecuteOn(VIRTUAL)`       |
| **Messaging**       | Consumer pode usar `@ExecuteOn(TaskExecutors.VIRTUAL)`          |
| **Scheduled**       | `@Scheduled` com executor virtual                               |
| **JPA/JDBC**        | Chamadas blocking funcionam sem degradar throughput              |

### Cuidados

- **Synchronized blocks**: Evite `synchronized` — use `ReentrantLock` para evitar thread pinning.
- **ThreadLocal**: Use `ScopedValue` (JDK 25) em vez de `ThreadLocal`.
- **Connection pools**: Ajuste `datasources.default.maximum-pool-size` pois virtual threads podem criar muitas conexões simultâneas.

```yaml
datasources:
  default:
    maximum-pool-size: 50
    connection-timeout: 10000
```

### Verificação

```java
@PostConstruct
void checkVirtualThreads() {
    LOG.info("Virtual threads? {}", Thread.currentThread().isVirtual());
}
```

> **Diferencial Micronaut:** Virtual threads são opt-in por endpoint via `@ExecuteOn(TaskExecutors.VIRTUAL)`, dando controle granular.

---

## Observabilidade

| Aspecto         | Tecnologia                       | Detalhes                                                    |
|-----------------|----------------------------------|-------------------------------------------------------------|
| **Métricas**    | Micronaut Micrometer             | `@Timed`, `@Counted` nos services                           |
| **Tracing**     | Micronaut Tracing (OpenTelemetry)| Propagação automática de traceId; exportador configurável   |
| **Logging**     | Logback + Logstash encoder       | JSON estruturado com traceId, spanId, correlationID         |
| **Health**      | Micronaut Management             | `/health`, `/health/readiness`, `/health/liveness`           |
| **API Docs**    | Micronaut OpenAPI / Swagger UI   | Gerado em compile-time, Swagger UI em `/swagger-ui`          |

> **Diferencial Micronaut:** A documentação OpenAPI é gerada em **compile-time** via annotation processing, sem overhead em runtime.

---

## Docker (Native / JVM)

### Dockerfile JVM

```dockerfile
# 1) Build — compila o JAR
FROM maven:3.9-eclipse-temurin-25 AS build
WORKDIR /workspace
COPY . .
RUN mvn -DskipTests package

# 2) Runtime — usa o JAR
FROM eclipse-temurin:25-jre
WORKDIR /opt/app
COPY --from=build /workspace/target/app-*.jar app.jar
ENTRYPOINT ["java", "-jar", "/opt/app/app.jar"]
```

### Dockerfile Native

```dockerfile
# 1) Build — compila o executável nativo
FROM ghcr.io/graalvm/native-image-community:25 AS build
WORKDIR /workspace
COPY . .
RUN ./mvnw -DskipTests package -Dpackaging=native-image

# 2) Runtime — imagem mínima
FROM debian:bookworm-slim
WORKDIR /opt/app
COPY --from=build /workspace/target/app /opt/app/application
EXPOSE 8080
ENTRYPOINT ["/opt/app/application"]
```

| Estágio   | Base image                                        | O que faz                        |
|-----------|---------------------------------------------------|----------------------------------|
| `build`   | `maven:3.9-eclipse-temurin-25` (JVM) ou `native-image-community` (Native) | Compila o código |
| `runtime` | `eclipse-temurin:25-jre` (JVM) ou `debian:bookworm-slim` (Native)         | Imagem final leve |

**Benefício:** Micronaut foi projetado para **compile-time DI**, resultando em startup extremamente rápido mesmo no modo JVM. No modo nativo, startup em milissegundos.

### Inicialização Local

```bash
# 1. Subir infraestrutura
docker-compose up -d

# 2. Rodar aplicação
MICRONAUT_ENVIRONMENTS=local ./mvnw mn:run
```

---

## Guia para Criar Novo Recurso

Ao adicionar um novo recurso, siga esta checklist:

### 1. Domain

- [ ] `domain/entity/FooEntity.java` — `@Entity`, tabela `tb_foo`, UUID PK
- [ ] `domain/entity/enums/` — Enums necessários (se houver)
- [ ] `domain/repository/mysql/FooRepository.java` — `@Repository` extends `JpaRepository<FooEntity, UUID>`
- [ ] `domain/service/FooService.java` — `@Singleton`, `@Transactional`, `@Timed`/`@Counted`
- [ ] `domain/exception/` — Exceções específicas (se necessário)

### 2. Core

- [ ] `core/mapper/FooMapper.java` — MapStruct interface com `componentModel = "jsr330"`

### 3. API (REST)

- [ ] `api/rest/v1/request/FooRequest.java` — `record` + `@Introspected` com Bean Validation
- [ ] `api/rest/v1/request/FooQueryRequest.java` — `record` para filtros (se necessário)
- [ ] `api/rest/v1/response/FooResponse.java` — `record` + `@Introspected`
- [ ] `api/rest/v1/openapi/FooOpenApi.java` — Interface OpenAPI
- [ ] `api/rest/v1/controller/FooController.java` — `record` `@Controller`

### 4. API (Mensageria) — se aplicável

- [ ] `api/messaging/kafka/event/FooCreatedEvent.java` — `record` + `@Introspected`
- [ ] `api/messaging/kafka/producer/FooProducer.java` — Interface `@KafkaClient`
- [ ] `api/messaging/kafka/consumer/FooConsumer.java` — `@KafkaListener`

### 5. Database

- [ ] `db/migration/V{n}.0__create_tb_foo.sql` — DDL
- [ ] `db/testdata/V{data}.0__seed_tb_foo_data.sql` — Seeds (opcional)

### 6. Testes

- [ ] `test/.../domain/entity/FooEntityTemplateLoader.java` — Fixture
- [ ] `test/.../domain/request/FooRequestTemplateLoader.java` — Fixture
- [ ] `test/.../domain/response/FooResponseTemplateLoader.java` — Fixture
- [ ] `test/.../domain/service/FooServiceTest.java` — Unit test
- [ ] `test/.../api/rest/v1/controller/FooControllerTest.java` — `@MicronautTest` + `@Client`
- [ ] `test/resources/features/Foo.feature` — BDD scenario (se aplicável)

---

## Onde colocar este arquivo para o Copilot

O GitHub Copilot reconhece **automaticamente** arquivos de instruções em locais específicos. Existem 3 níveis:

### 1. Instruções no nível do repositório (recomendado)

Copie o conteúdo deste arquivo para:

```
.github/copilot-instructions.md
```

O Copilot **lê automaticamente** este arquivo e o usa como contexto em **todas** as conversas do repositório. Não precisa de nenhuma configuração extra.

### 2. Instruções por workspace (VS Code settings)

No `.vscode/settings.json` do workspace:

```jsonc
{
  "github.copilot.chat.codeGeneration.instructions": [
    {
      "file": "project-structure.md"
    }
  ]
}
```

| Setting                                           | Quando é usado                          |
|---------------------------------------------------|-----------------------------------------|
| `github.copilot.chat.codeGeneration.instructions` | Ao gerar código                         |
| `github.copilot.chat.testGeneration.instructions`  | Ao gerar testes                         |
| `github.copilot.chat.codeGeneration.useInstructionFiles` | `true` para habilitar instruction files |

### 3. Instrução inline (via `.instructions.md`)

Crie arquivos `*.instructions.md` em qualquer pasta. Habilite com:

```jsonc
// .vscode/settings.json
{
  "github.copilot.chat.codeGeneration.useInstructionFiles": true
}
```

### Recomendação final

Use **`.github/copilot-instructions.md`** — é o método mais simples e funciona automaticamente no VS Code e no GitHub.com.
