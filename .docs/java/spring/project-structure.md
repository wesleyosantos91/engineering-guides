# Project Structure

> **Objetivo deste documento:** Servir como referência organizacional para o GitHub Copilot e desenvolvedores.
> Foca exclusivamente na **forma de organizar pacotes, diretórios e convenções**, sem detalhar regras de negócio.
>
> **Localização recomendada:** `.github/copilot-instructions.md` (veja [Onde colocar este arquivo](#onde-colocar-este-arquivo-para-o-copilot)).

---

## Sumário

- [Project Structure](#project-structure)
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
  - [Configuração por Profile](#configuração-por-profile)
    - [Particularidades do Profile `local`](#particularidades-do-profile-local)
  - [Banco de Dados e Migrations](#banco-de-dados-e-migrations)
    - [Convenção de Migrations](#convenção-de-migrations)
  - [Testes](#testes)
    - [Estrutura](#estrutura)
    - [Convenções de Teste](#convenções-de-teste)
    - [Organização de Testes](#organização-de-testes)
    - [Comandos de Teste](#comandos-de-teste)
  - [Build e Qualidade](#build-e-qualidade)
  - [Resiliência](#resiliência)
    - [Dependência](#dependência)
    - [Padrões Disponíveis](#padrões-disponíveis)
    - [Organização de Pacotes](#organização-de-pacotes)
    - [Configuração (application.yml)](#configuração-applicationyml)
    - [Padrão de uso no Service](#padrão-de-uso-no-service)
    - [Métricas de Resiliência](#métricas-de-resiliência)
  - [Segurança (AuthN/AuthZ)](#segurança-authnauthz)
    - [Dependências](#dependências)
    - [Organização de Pacotes](#organização-de-pacotes-1)
    - [Configuração (application.yml)](#configuração-applicationyml-1)
    - [Padrão de SecurityFilterChain](#padrão-de-securityfilterchain)
    - [RBAC (Role-Based Access Control)](#rbac-role-based-access-control)
  - [Tratamento Global de Erros](#tratamento-global-de-erros)
    - [Padrão: ProblemDetail (RFC 9457)](#padrão-problemdetail-rfc-9457)
    - [Organização](#organização)
    - [Configuração](#configuração)
    - [Padrão de ExceptionHandler](#padrão-de-exceptionhandler)
    - [Formato de Resposta (JSON)](#formato-de-resposta-json)
    - [Mapeamento de Exceções](#mapeamento-de-exceções)
  - [Versionamento de API](#versionamento-de-api)
    - [Estratégia: Versionamento por Path](#estratégia-versionamento-por-path)
    - [Organização de Pacotes](#organização-de-pacotes-2)
    - [Regras](#regras)
  - [Paginação e Ordenação](#paginação-e-ordenação)
    - [Convenção de Query Parameters](#convenção-de-query-parameters)
    - [Formato de Resposta Paginada](#formato-de-resposta-paginada)
    - [Padrão no Controller](#padrão-no-controller)
    - [Padrão no Service](#padrão-no-service)
    - [Limites](#limites)
  - [CORS](#cors)
    - [Configuração Centralizada](#configuração-centralizada)
    - [Padrão de Configuração](#padrão-de-configuração)
    - [Configuração por Profile](#configuração-por-profile-1)
  - [Feature Flags](#feature-flags)
    - [Biblioteca: Togglz](#biblioteca-togglz)
    - [Organização de Pacotes](#organização-de-pacotes-3)
    - [Definição das Features](#definição-das-features)
    - [Uso no Service](#uso-no-service)
    - [Configuração](#configuração-1)
  - [Virtual Threads](#virtual-threads)
    - [Configuração (Spring Boot 4.x + JDK 25)](#configuração-spring-boot-4x--jdk-25)
    - [Impacto](#impacto)
    - [Cuidados](#cuidados)
    - [Verificação](#verificação)
  - [Observabilidade](#observabilidade)
  - [Docker (CDS)](#docker-cds)
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

| Categoria           | Tecnologia                                         |
|---------------------|-----------------------------------------------------|
| **Java**            | 25                                                  |
| **Spring Boot**     | 4.x.x                                              |
| **Spring Cloud**    | 2025.x.x                                           |
| **Build Tool**      | Maven (wrapper incluído)                            |
| Web / API           | Spring Web (REST) — *extensível para gRPC/GraphQL*  |
| Persistência        | Spring Data JPA + Hibernate                         |
| Migrations          | Flyway                                              |
| Validação           | Jakarta Bean Validation                             |
| Mapeamento          | MapStruct                                           |
| Documentação API    | SpringDoc OpenAPI (Swagger UI)                      |
| HTTP Client         | Spring Interface Clients (`@HttpExchange`)          |
| Métricas            | Micrometer                                          |
| Tracing             | Micrometer Tracing                                  |
| Logging             | Logback + Logstash Encoder (JSON estruturado)       |
| Testes Unitários    | JUnit 5 + Mockito + MockMvc                         |
| Testes Integração   | Testcontainers + Cucumber                           |
| Testes de Carga     | k6 (smoke, load, stress)                            |
| Resiliência         | Resilience4j (Circuit Breaker, Retry, Rate Limiter, Bulkhead, TimeLimiter) |
| Segurança           | Spring Security + OAuth2 Resource Server (JWT)      |
| Feature Flags       | Togglz (ou Spring Feature Flags)                    |
| Virtual Threads     | Project Loom (JDK 25 + `spring.threads.virtual.enabled`) |
| Qualidade           | JaCoCo, PIT (mutation testing)                      |

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
│   │   │   ├── Application.java                          # @SpringBootApplication
│   │   │   │
│   │   │   ├── api/                                      # ── CONTROLLERS, DTOs, EXCEPTION HANDLERS ──
│   │   │   │   ├── exception/
│   │   │   │   │   └── ApiExceptionHandler.java          # @RestControllerAdvice
│   │   │   │   └── rest/                                 # protocolo: rest/ | grpc/ | graphql/
│   │   │   │       └── v1/                               # versionamento por path
│   │   │   │           ├── commons/enums/                # enums exclusivos da API
│   │   │   │           ├── controller/                   # @RestController (records)
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
│   │   │   │   └── service/                              # @Service
│   │   │   │
│   │   │   └── infrastructure/                           # ── CONFIGURAÇÕES TÉCNICAS ──
│   │   │       ├── metrics/
│   │   │       ├── resilience/                            # Resilience4j configs
│   │   │       ├── security/                             # Spring Security configs
│   │   │       ├── featureflag/                           # Togglz / Feature Flags
│   │   │       ├── cors/                                 # CORS config
│   │   │       ├── springdoc/
│   │   │       └── messaging/                            # kafka/ | sqs/ | rabbitmq/
│   │   │
│   │   └── resources/
│   │       ├── application.yml                           # config padrão (produção)
│   │       ├── application-local.yml                     # config local (dev)
│   │       ├── logback-spring.xml                        # logging estruturado (JSON)
│   │       └── db/
│   │           ├── migration/                            # DDL — Flyway (V{n}.0__)
│   │           └── testdata/                             # DML — seeds (somente profile local)
│   │
│   └── test/
│       ├── java/{group-id}/
│       │   ├── api/rest/v1/controller/                   # @WebMvcTest (unit)
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

| Pacote                        | Responsabilidade                                       | Convenção                              |
|-------------------------------|--------------------------------------------------------|----------------------------------------|
| `api.exception`               | Tratamento global de exceções (`@RestControllerAdvice`) | Uma classe por escopo de exceções       |
| `api.rest.v1.controller`      | Controllers REST (`@RestController` como `record`)      | Um controller por recurso              |
| `api.rest.v1.openapi`         | Interfaces com anotações Swagger (`@Tag`, `@Operation`) | Uma interface por controller           |
| `api.rest.v1.request`         | DTOs de entrada (`record` com Bean Validation)          | Sufixo `Request`                       |
| `api.rest.v1.response`        | DTOs de saída (`record`)                                | Sufixo `Response`                      |
| `api.rest.v1.commons.enums`   | Enums exclusivos da API                                 | Espelham domain enums quando necessário |

**Padrões adotados:**

- **Protocolo como pacote**: `api.rest.`, `api.grpc.`, `api.graphql.`
- **Versionamento por path**: `/v1/`, `/v2/`, etc.
- **Controllers como `record`**: Imutabilidade + injeção pelo construtor canônico.
- **Interface OpenAPI separada**: Mantém annotations Swagger fora do controller.
- **`ProblemDetail` (RFC 9457)**: Padrão de resposta de erro.

### core — Mappers e Validação

Código **reutilizável** entre pacotes.

| Pacote              | Responsabilidade                              | Convenção                           |
|---------------------|-----------------------------------------------|-------------------------------------|
| `core.mapper`       | MapStruct mappers (request ↔ entity ↔ response) | Sufixo `Mapper`, interface + `MAPPER` singleton |
| `core.validation`   | Grupos de validação (Bean Validation groups)  | Interface marker (`Create`, `Update`) |

**Padrão de mapper:**

```java
@Mapper(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface FooMapper {
    FooMapper MAPPER = Mappers.getMapper(FooMapper.class);
    
    FooEntity toEntity(FooRequest request);
    FooResponse toResponse(FooEntity entity);
    void update(FooRequest request, @MappingTarget FooEntity entity);
    default Page<FooResponse> toPageResponse(Page<FooEntity> page) {
        return page.map(this::toResponse);
    }
}
```

### domain — Entities, Repositories e Services

Contém **entidades, repositórios, serviços e exceções**.

| Pacote                      | Responsabilidade                              | Convenção                             |
|-----------------------------|-----------------------------------------------|---------------------------------------|
| `domain.entity`             | JPA entities (`@Entity`)                      | Sufixo `Entity`, tabela `tb_*`       |
| `domain.entity.enums`       | Enums persistidos (com `fromValue()` estático)| Enum com `HashMap` lookup             |
| `domain.repository.mysql`   | Repositories para MySQL                       | Sufixo `Repository`, `JpaRepository<Entity, UUID>` |
| `domain.repository.postgresql` | Repositories para PostgreSQL (se aplicável)| Mesmo padrão                          |
| `domain.repository.mongodb` | Repositories para MongoDB (se aplicável)      | `MongoRepository<Document, String>`   |
| `domain.service`            | Serviços (`@Service`)                         | Sufixo `Service`, injeção via construtor |
| `domain.exception`          | Exceções específicas                          | `ResourceNotFoundException` (checked), `BusinessException` (unchecked) |

**Padrões adotados:**

- **Tecnologia do banco como pacote**: `domain.repository.mysql.`, `domain.repository.postgresql.`, `domain.repository.mongodb.`
- **Chaves primárias UUID** com `@GeneratedValue(strategy = GenerationType.UUID)`.
- **Relacionamentos LAZY** entre entidades.
- **`@Transactional`** nos services: `readOnly=true` para reads, padrão para writes.
- **`@Counted` e `@Timed`** (Micrometer) nos métodos de service para métricas.

### infrastructure — Configurações Técnicas

Configurações de infraestrutura e integrações técnicas.

| Pacote                             | Responsabilidade                         | Convenção                    |
|------------------------------------|------------------------------------------|------------------------------|
| `infrastructure.metrics`           | Beans de métricas (`TimedAspect`, etc.)  | Sufixo `Config`              |
| `infrastructure.springdoc`         | Config OpenAPI/Swagger                   | `@Profile("local")` quando aplicável |
| `infrastructure.messaging.kafka`   | Config Kafka                             | Sufixo `Config`              |
| `infrastructure.messaging.sqs`     | Config SQS                               | Sufixo `Config`              |
| `infrastructure.messaging.rabbitmq`| Config RabbitMQ                          | Sufixo `Config`              |
| `infrastructure.resilience`        | Config Resilience4j (Circuit Breaker, Retry, etc.) | Sufixo `Config`     |
| `infrastructure.security`          | Spring Security (JWT, OAuth2, RBAC)              | Sufixo `Config`              |
| `infrastructure.featureflag`       | Togglz (feature toggles)                         | Sufixo `Config`              |
| `infrastructure.cors`              | CORS (Cross-Origin Resource Sharing)             | Sufixo `Config`              |

**Extensões esperadas nesta camada:**

- `infrastructure.client` — HTTP clients declarativos via `@HttpExchange`.
- `infrastructure.cache` — Configuração de cache (Redis, Caffeine).

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
| REST       | `spring-boot-starter-web`                                |
| gRPC       | `grpc-spring-boot-starter` + `protobuf-maven-plugin`    |
| GraphQL    | `spring-boot-starter-graphql`                            |

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
│   ├── producer/                    # KafkaTemplate
│   └── event/                       # DTOs/records dos eventos
├── sqs/                             # ── AWS SQS ──
│   ├── consumer/                    # @SqsListener
│   ├── producer/                    # SqsTemplate
│   └── event/
└── rabbitmq/                        # ── RabbitMQ ──
    ├── consumer/                    # @RabbitListener
    ├── producer/                    # RabbitTemplate
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
| Kafka      | `spring-kafka`                                            |
| SQS        | `spring-cloud-aws-starter-sqs`                            |
| RabbitMQ   | `spring-boot-starter-amqp`                                |

**Padrão de consumer:**

```java
@Component
@RequiredArgsConstructor
public class TransactionConsumer {

    private final TransactionService transactionService;

    @KafkaListener(topics = "${app.kafka.topics.transaction-created}", groupId = "${app.kafka.consumer.group-id}")
    public void consume(TransactionCreatedEvent event) {
        transactionService.process(event);
    }
}
```

**Padrão de producer:**

```java
@Component
@RequiredArgsConstructor
public class NotificationProducer {

    private final KafkaTemplate<String, NotificationRequestEvent> kafkaTemplate;

    public void send(NotificationRequestEvent event) {
        kafkaTemplate.send("notification-request", event.id().toString(), event);
    }
}
```

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
| MySQL       | `mysql-connector-j` + `spring-boot-starter-data-jpa` | `JpaRepository<Entity, UUID>`      |
| PostgreSQL  | `postgresql` + `spring-boot-starter-data-jpa`         | `JpaRepository<Entity, UUID>`      |
| MongoDB     | `spring-boot-starter-data-mongodb`                    | `MongoRepository<Document, String>`|

**Motivo:** quando a aplicação lida com múltiplos bancos (ex: MySQL para dados transacionais + MongoDB para logs de eventos), o prefixo torna explícito qual datasource cada repository utiliza.

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
- **Grupos de validação** (`Groups.Create`, `Groups.Update`) para regras condicionais.

### Controllers

- **Implementados como `record`**: Garante imutabilidade e injeção de dependências.
- **Implementam interface OpenAPI**: Mantém documentação separada da implementação.
- **Retornam `ResponseEntity<?>`**: Permite controle explícito de headers e status codes.

### Entities

- **UUID como chave primária** em todas as entidades.
- **Fetch LAZY** em todos os relacionamentos.
- **Enums com `fromValue()`**: Lookup via `HashMap` estático para conversão segura.

---

## Configuração por Profile

| Profile     | Arquivo                  | Uso                                          |
|-------------|--------------------------|----------------------------------------------|
| *(default)* | `application.yml`        | Produção — configs via variáveis de ambiente |
| `local`     | `application-local.yml`  | Desenvolvimento local — seeds, Swagger, virtual threads |
| `test`      | `application-test.yml`   | Testes — Testcontainers injeta configs dinamicamente |

### Particularidades do Profile `local`

- **Virtual threads** habilitados (`spring.threads.virtual.enabled: true`)
- **Flyway**: carrega seeds de `db/testdata/` além de `db/migration/`
- **Tracing**: sampling 100%
- **SpringDoc**: ativo com Swagger UI

---

## Banco de Dados e Migrations

### Convenção de Migrations

- **DDL (schema)**: Em `db/migration/`, prefixo `V{n}.0__`
- **DML (seeds)**: Em `db/testdata/`, prefixo `V{ano.mes.dia.seq}.0__seed_`
- Seeds são carregados **somente no profile `local`**
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
- **`@WebMvcTest`**: Para testes unitários de controllers (sem carregar o contexto completo).
- **`@ParameterizedTest`**: Com `@ValueSource` para testar variações de input.
- **Testcontainers**: Container do banco compartilhado via base class e `@DynamicPropertySource`.
- **Cucumber**: Features em `src/test/resources/features/`, steps em `cucumber.step`.

### Organização de Testes

```
test/
├── java/{group-id}/
│   ├── api/rest/v1/controller/
│   │   └── FooControllerTest.java           # @WebMvcTest
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

| Plugin                      | Fase            | Função                            |
|-----------------------------|-----------------|-----------------------------------|
| `spring-boot-maven-plugin`  | package         | Gera fat JAR executável           |
| `maven-surefire-plugin`     | test            | Roda testes unitários (`*Test`)   |
| `maven-failsafe-plugin`     | verify          | Roda testes de integração (`*IT`) |
| `jacoco-maven-plugin`       | test/verify     | Cobertura de código               |
| `pitest-maven`              | manual          | Testes de mutação                 |

---

## Resiliência

O projeto utiliza **Resilience4j** integrado ao Spring Boot para padrões de resiliência em chamadas a serviços externos (HTTP clients, bancos, mensageria).

### Dependência

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

### Padrões Disponíveis

| Padrão            | Anotação / API                  | Quando usar                                           |
|-------------------|---------------------------------|-------------------------------------------------------|
| **Circuit Breaker** | `@CircuitBreaker`             | Chamadas a serviços externos instáveis                |
| **Retry**           | `@Retry`                      | Falhas transitórias (timeout, 503)                    |
| **Rate Limiter**    | `@RateLimiter`                | Proteger endpoints contra excesso de requisições      |
| **Bulkhead**        | `@Bulkhead`                   | Isolar chamadas para evitar esgotamento de threads    |
| **Time Limiter**    | `@TimeLimiter`                | Definir timeout máximo para chamadas assíncronas      |

### Organização de Pacotes

```
infrastructure/resilience/
├── ResilienceConfig.java             # @Configuration — beans e customizações globais
└── CircuitBreakerEventListener.java  # Listener para logging de eventos do circuit breaker
```

### Configuração (application.yml)

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
    instances:
      paymentService:
        base-config: default
  retry:
    configs:
      default:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
    instances:
      paymentService:
        base-config: default
  ratelimiter:
    configs:
      default:
        limit-for-period: 100
        limit-refresh-period: 1s
        timeout-duration: 0s
  bulkhead:
    configs:
      default:
        max-concurrent-calls: 25
        max-wait-duration: 0s
  timelimiter:
    configs:
      default:
        timeout-duration: 5s
```

### Padrão de uso no Service

```java
@Service
public record PaymentService(PaymentClient paymentClient) {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackCharge")
    @Retry(name = "paymentService")
    @TimeLimiter(name = "paymentService")
    public CompletableFuture<PaymentResponse> charge(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> paymentClient.charge(request));
    }

    private CompletableFuture<PaymentResponse> fallbackCharge(PaymentRequest request, Throwable t) {
        log.warn("Fallback acionado para payment charge: {}", t.getMessage());
        return CompletableFuture.completedFuture(PaymentResponse.unavailable());
    }
}
```

> **Ordem de execução das anotações** (de fora para dentro): `Retry → CircuitBreaker → RateLimiter → TimeLimiter → Bulkhead → chamada real`.

### Métricas de Resiliência

O Resilience4j expõe automaticamente métricas via Micrometer:
- `resilience4j_circuitbreaker_state`
- `resilience4j_circuitbreaker_failure_rate`
- `resilience4j_retry_calls_total`
- `resilience4j_ratelimiter_available_permissions`
- `resilience4j_bulkhead_available_concurrent_calls`

Todas são acessíveis em `/actuator/prometheus`.

---

## Segurança (AuthN/AuthZ)

O projeto usa **Spring Security** com **OAuth2 Resource Server** para autenticação via JWT.

### Dependências

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### Organização de Pacotes

```
infrastructure/security/
├── SecurityConfig.java              # @Configuration — SecurityFilterChain, CORS, CSRF
├── JwtAuthConverter.java            # Extrai roles/authorities do token JWT
└── SecurityConstants.java           # Constantes de roles e paths públicos
```

### Configuração (application.yml)

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${JWT_ISSUER_URI}
          # ou jwk-set-uri: ${JWK_SET_URI}
```

### Padrão de SecurityFilterChain

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**", "/v3/api-docs/**", "/swagger-ui/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/v1/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter()))
            )
            .build();
    }
}
```

### RBAC (Role-Based Access Control)

```java
// No controller ou service
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<UserResponse> delete(@PathVariable UUID id) { ... }

@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
public ResponseEntity<ReportResponse> generate() { ... }
```

| Anotação             | Uso                                    |
|----------------------|----------------------------------------|
| `@PreAuthorize`      | Antes da execução do método            |
| `@PostAuthorize`     | Após execução (filtra resultado)       |
| `@Secured`           | Alternativa simplificada (roles)       |

---

## Tratamento Global de Erros

### Padrão: ProblemDetail (RFC 9457)

O Spring Boot 4.x suporta nativamente `ProblemDetail` como resposta padrão de erro.

### Organização

```
api/exception/
├── ApiExceptionHandler.java          # @RestControllerAdvice — handler global
├── ProblemDetailFactory.java         # Factory para criar ProblemDetail padronizados
└── ErrorCode.java                    # enum com códigos de erro do domínio
```

### Configuração

```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

### Padrão de ExceptionHandler

```java
@RestControllerAdvice
public class ApiExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Recurso não encontrado");
        pd.setType(URI.create("https://api.example.com/errors/not-found"));
        pd.setProperty("errorCode", "RESOURCE_NOT_FOUND");
        pd.setProperty("timestamp", Instant.now());
        return pd;
    }

    @ExceptionHandler(BusinessException.class)
    public ProblemDetail handleBusiness(BusinessException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.UNPROCESSABLE_ENTITY, ex.getMessage());
        pd.setTitle("Erro de negócio");
        pd.setType(URI.create("https://api.example.com/errors/business"));
        pd.setProperty("errorCode", ex.getCode());
        return pd;
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
  "instance": "/v1/users/550e8400-e29b-41d4-a716-446655440000",
  "errorCode": "RESOURCE_NOT_FOUND",
  "timestamp": "2026-02-22T10:30:00Z"
}
```

### Mapeamento de Exceções

| Exceção                     | Status HTTP | errorCode              |
|-----------------------------|-------------|------------------------|
| `ResourceNotFoundException` | 404         | `RESOURCE_NOT_FOUND`   |
| `BusinessException`         | 422         | Dinâmico (do domínio)  |
| `MethodArgumentNotValidException` | 400   | `VALIDATION_ERROR`     |
| `ConstraintViolationException`    | 400   | `CONSTRAINT_VIOLATION` |
| `AccessDeniedException`     | 403         | `ACCESS_DENIED`        |
| `Exception` (genérica)      | 500         | `INTERNAL_ERROR`       |

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
| `sort`     | `string` | —        | `?sort=name,asc&sort=createdAt,desc` |

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
@GetMapping
public ResponseEntity<Page<UserResponse>> findAll(
        @ParameterObject Pageable pageable) {
    return ResponseEntity.ok(userService.findAll(pageable));
}
```

### Padrão no Service

```java
@Transactional(readOnly = true)
public Page<UserResponse> findAll(Pageable pageable) {
    return userRepository.findAll(pageable)
            .map(UserMapper.MAPPER::toResponse);
}
```

### Limites

```yaml
spring:
  data:
    web:
      pageable:
        default-page-size: 20
        max-page-size: 100
```

---

## CORS

### Configuração Centralizada

```
infrastructure/cors/
└── CorsConfig.java                  # @Configuration — WebMvcConfigurer
```

### Padrão de Configuração

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
            .allowedOrigins("${app.cors.allowed-origins}")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
            .allowedHeaders("*")
            .exposedHeaders("Location", "X-Total-Count")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

### Configuração por Profile

```yaml
# application.yml (produção)
app:
  cors:
    allowed-origins: https://app.example.com

# application-local.yml
app:
  cors:
    allowed-origins: http://localhost:3000,http://localhost:5173
```

> **Nota:** Quando Spring Security está habilitado, o CORS também deve ser configurado no `SecurityFilterChain` via `.cors(Customizer.withDefaults())`.

---

## Feature Flags

### Biblioteca: Togglz

```xml
<dependency>
    <groupId>org.togglz</groupId>
    <artifactId>togglz-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.togglz</groupId>
    <artifactId>togglz-console</artifactId>
</dependency>
```

### Organização de Pacotes

```
infrastructure/featureflag/
├── FeatureFlagConfig.java            # @Configuration — Togglz config
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

### Uso no Service

```java
@Service
public record PaymentService(PaymentRepository repository) {

    public PaymentResponse process(PaymentRequest request) {
        if (Features.NEW_PAYMENT_FLOW.isActive()) {
            return processV2(request);
        }
        return processV1(request);
    }
}
```

### Configuração

```yaml
togglz:
  enabled: true
  features:
    NEW_PAYMENT_FLOW:
      enabled: false
    PUSH_NOTIFICATION:
      enabled: true
  console:
    enabled: true               # UI em /togglz-console (apenas local)
    secured: false
    path: /togglz-console
```

> **Boas práticas:** Remova flags antigas assim que estiverem 100% habilitadas. Mantenha max ~10 flags ativas simultâneas.

---

## Virtual Threads

### Configuração (Spring Boot 4.x + JDK 25)

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Com essa configuração, **todas as requisições HTTP** são processadas em virtual threads automaticamente.

### Impacto

| Componente          | Efeito                                                         |
|---------------------|----------------------------------------------------------------|
| **Tomcat**          | Usa virtual threads no thread pool de requisições              |
| **@Async**          | Tarefas assíncronas rodam em virtual threads                   |
| **@Scheduled**      | Tasks agendadas rodam em virtual threads                       |
| **Kafka Listeners** | Consumers processam em virtual threads                         |
| **JPA/JDBC**        | Chamadas blocking funcionam sem degradar throughput             |

### Cuidados

- **Synchronized blocks**: Evite `synchronized` — use `ReentrantLock` para evitar thread pinning.
- **ThreadLocal**: Virtual threads compartilham carrier threads; use `ScopedValue` (JDK 25) em vez de `ThreadLocal`.
- **Connection pools**: Ajuste `maximumPoolSize` do HikariCP, pois virtual threads podem criar muito mais conexões simultâneas.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 50          # ajustar conforme carga
      connection-timeout: 10000
```

### Verificação

Para confirmar que virtual threads estão ativos:

```java
@PostConstruct
void checkVirtualThreads() {
    log.info("Virtual threads? {}", Thread.currentThread().isVirtual());
}
```

---

## Observabilidade

| Aspecto         | Tecnologia                       | Detalhes                                                |
|-----------------|----------------------------------|---------------------------------------------------------|
| **Métricas**    | Micrometer                       | `@Timed`, `@Counted` nos services                       |
| **Tracing**     | Micrometer Tracing               | Propagação de traceId; sampling configurável por profile |
| **Logging**     | Logback + Logstash encoder       | JSON estruturado com traceId, spanId, correlationID      |
| **Health**      | Spring Actuator                  | `/actuator/health`                                       |
| **API Docs**    | SpringDoc / Swagger UI           | Apenas no profile `local`                                |

---

## Docker (CDS)

O Dockerfile usa **Class Data Sharing (CDS)** em 3 estágios para startup mais rápido:

```dockerfile
# 1) Build — compila o JAR
FROM maven:3.9-eclipse-temurin-25 AS build
WORKDIR /workspace
COPY . .
RUN mvn -DskipTests package

# 2) Training — gera o arquivo CDS (.jsa)
FROM eclipse-temurin:25-jdk AS train
WORKDIR /opt/app
COPY --from=build /workspace/target/*.jar app.jar
RUN java -XX:ArchiveClassesAtExit=/opt/app/application.jsa \
         -Dspring.context.exit=onRefresh \
         -jar app.jar

# 3) Runtime — usa o .jsa para startup rápido
FROM eclipse-temurin:25-jre
WORKDIR /opt/app
COPY --from=build /workspace/target/*.jar app.jar
COPY --from=train /opt/app/application.jsa application.jsa
ENV JAVA_TOOL_OPTIONS="-Xshare:on -XX:SharedArchiveFile=/opt/app/application.jsa"
ENTRYPOINT ["java", "-jar", "/opt/app/app.jar"]
```

**Estágios:**

| Estágio   | Base image                  | O que faz                                              |
|-----------|-----------------------------|--------------------------------------------------------|
| `build`   | `maven:3.9-eclipse-temurin-25` | Compila o código e gera o fat JAR                    |
| `train`   | `eclipse-temurin:25-jdk`    | Roda a app até o contexto Spring carregar, gera `.jsa` |
| `runtime` | `eclipse-temurin:25-jre`    | Imagem final leve (JRE), usa o `.jsa` no startup       |

**Benefício:** O CDS pré-carrega as classes em memória compartilhada, reduzindo significativamente o tempo de startup.

### Inicialização Local

```bash
# 1. Subir infraestrutura
docker-compose up -d

# 2. Rodar aplicação
./mvnw spring-boot:run -Dspring-boot.run.profiles=local
```

---

## Guia para Criar Novo Recurso

Ao adicionar um novo recurso, siga esta checklist:

### 1. Domain

- [ ] `domain/entity/FooEntity.java` — `@Entity`, tabela `tb_foo`, UUID PK
- [ ] `domain/entity/enums/` — Enums necessários (se houver)
- [ ] `domain/repository/mysql/FooRepository.java` — `JpaRepository<FooEntity, UUID>`
- [ ] `domain/service/FooService.java` — `@Service`, `@Transactional`, `@Timed`/`@Counted`
- [ ] `domain/exception/` — Exceções específicas (se necessário)

### 2. Core

- [ ] `core/mapper/FooMapper.java` — MapStruct interface com `MAPPER` singleton

### 3. API (REST)

- [ ] `api/rest/v1/request/FooRequest.java` — `record` com Bean Validation
- [ ] `api/rest/v1/request/FooQueryRequest.java` — `record` para filtros (se necessário)
- [ ] `api/rest/v1/response/FooResponse.java` — `record`
- [ ] `api/rest/v1/openapi/FooOpenApi.java` — Interface Swagger
- [ ] `api/rest/v1/controller/FooController.java` — `record` `@RestController`

### 4. API (Mensageria) — se aplicável

- [ ] `api/messaging/kafka/event/FooCreatedEvent.java` — `record` do evento
- [ ] `api/messaging/kafka/producer/FooProducer.java` — Produtor
- [ ] `api/messaging/kafka/consumer/FooConsumer.java` — Consumidor

### 5. Database

- [ ] `db/migration/V{n}.0__create_tb_foo.sql` — DDL
- [ ] `db/testdata/V{data}.0__seed_tb_foo_data.sql` — Seeds (opcional)

### 6. Testes

- [ ] `test/.../domain/entity/FooEntityTemplateLoader.java` — Fixture
- [ ] `test/.../domain/request/FooRequestTemplateLoader.java` — Fixture
- [ ] `test/.../domain/response/FooResponseTemplateLoader.java` — Fixture
- [ ] `test/.../domain/service/FooServiceTest.java` — Unit test
- [ ] `test/.../api/rest/v1/controller/FooControllerTest.java` — `@WebMvcTest`
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
