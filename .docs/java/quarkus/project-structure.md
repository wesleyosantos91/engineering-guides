# Project Structure — Quarkus

> **Objetivo deste documento:** Servir como referência organizacional para o GitHub Copilot e desenvolvedores.
> Foca exclusivamente na **forma de organizar pacotes, diretórios e convenções**, sem detalhar regras de negócio.
>
> **Localização recomendada:** `.github/copilot-instructions.md` (veja [Onde colocar este arquivo](#onde-colocar-este-arquivo-para-o-copilot)).

---

## Sumário

- [Stack Tecnológica](#stack-tecnológica)
- [Estrutura de Diretórios](#estrutura-de-diretórios)
- [Organização dos Pacotes](#organização-dos-pacotes)
  - [api — Resources, DTOs e Exception Mappers](#api--resources-dtos-e-exception-mappers)
  - [core — Mappers e Validação](#core--mappers-e-validação)
  - [domain — Entities, Repositories e Services](#domain--entities-repositories-e-services)
  - [infrastructure — Configurações Técnicas](#infrastructure--configurações-técnicas)
- [Pontos de Extensão](#pontos-de-extensão)
  - [API — REST / gRPC / GraphQL](#api--rest--grpc--graphql)
  - [Mensageria — Kafka / SQS / RabbitMQ](#mensageria--kafka--sqs--rabbitmq)
  - [Banco de Dados — MySQL / PostgreSQL / MongoDB](#banco-de-dados--mysql--postgresql--mongodb)
- [Convenções de Código](#convenções-de-código)
- [Configuração por Profile](#configuração-por-profile)
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
- [Virtual Threads](#virtual-threads)
- [Observabilidade](#observabilidade)
- [Docker (Native / JVM)](#docker-native--jvm)
- [Guia para Criar Novo Recurso](#guia-para-criar-novo-recurso)
- [Onde colocar este arquivo para o Copilot](#onde-colocar-este-arquivo-para-o-copilot)

---

## Stack Tecnológica

| Categoria           | Tecnologia                                              |
|---------------------|----------------------------------------------------------|
| **Java**            | 25                                                       |
| **Quarkus**         | 3.x.x                                                   |
| **Build Tool**      | Maven (wrapper incluído)                                 |
| Web / API           | RESTEasy Reactive (JAX-RS) — *extensível para gRPC/GraphQL* |
| Persistência        | Hibernate ORM with Panache                               |
| Migrations          | Flyway                                                   |
| Validação           | Hibernate Validator (Jakarta Bean Validation)            |
| Mapeamento          | MapStruct                                                |
| Documentação API    | SmallRye OpenAPI (Swagger UI)                            |
| HTTP Client         | Quarkus REST Client Reactive (`@RegisterRestClient`)     |
| Métricas            | Micrometer (Quarkus Micrometer extension)                |
| Tracing             | OpenTelemetry (Quarkus OpenTelemetry extension)          |
| Logging             | JBoss Logging + JSON formatter                           |
| Testes Unitários    | JUnit 5 + Mockito + REST Assured                         |
| Testes Integração   | `@QuarkusTest` + Dev Services + Cucumber                 |
| Testes de Carga     | k6 (smoke, load, stress)                                 |
| Resiliência         | SmallRye Fault Tolerance (Circuit Breaker, Retry, Timeout, Bulkhead, Fallback) |
| Segurança           | Quarkus Security + SmallRye JWT / OIDC (OAuth2)          |
| Feature Flags       | Togglz (CDI integration)                                 |
| Virtual Threads     | Quarkus Virtual Threads extension (`@RunOnVirtualThread`) |
| Qualidade           | JaCoCo, PIT (mutation testing)                           |

---

## Estrutura de Diretórios

```
app/
├── pom.xml
├── Dockerfile.jvm
├── Dockerfile.native
├── docker-compose.yml
│
├── src/
│   ├── main/
│   │   ├── java/{group-id}/
│   │   │   │
│   │   │   ├── api/                                      # ── RESOURCES, DTOs, EXCEPTION MAPPERS ──
│   │   │   │   ├── exception/
│   │   │   │   │   └── ApiExceptionMapper.java           # implements ExceptionMapper<T>
│   │   │   │   └── rest/                                 # protocolo: rest/ | grpc/ | graphql/
│   │   │   │       └── v1/                               # versionamento por path
│   │   │   │           ├── commons/enums/                # enums exclusivos da API
│   │   │   │           ├── resource/                     # @Path JAX-RS resources (records)
│   │   │   │           ├── openapi/                      # interfaces @Tag/@Operation (Swagger)
│   │   │   │           ├── request/                      # records de entrada (DTOs)
│   │   │   │           └── response/                     # records de saída (DTOs)
│   │   │   │
│   │   │   ├── core/                                     # ── MAPPERS E VALIDAÇÃO ──
│   │   │   │   ├── mapper/                               # MapStruct mappers
│   │   │   │   └── validation/                           # grupos de validação
│   │   │   │
│   │   │   ├── domain/                                   # ── ENTITIES, REPOSITORIES, SERVICES ──
│   │   │   │   ├── entity/                               # @Entity (Panache)
│   │   │   │   │   └── enums/                            # enums persistidos
│   │   │   │   ├── exception/                            # exceções
│   │   │   │   ├── repository/                           # tecnologia do banco como prefixo
│   │   │   │   │   └── mysql/                            # mysql/ | postgresql/ | mongodb/
│   │   │   │   └── service/                              # @ApplicationScoped
│   │   │   │
│   │   │   └── infrastructure/                           # ── CONFIGURAÇÕES TÉCNICAS ──
│   │   │       ├── metrics/
│   │   │       ├── openapi/
│   │   │       ├── resilience/                            # Fault Tolerance configs
│   │   │       ├── security/                              # Quarkus Security / JWT config
│   │   │       ├── cors/                                  # CORS config
│   │   │       ├── featureflag/                           # Togglz feature flags
│   │   │       └── messaging/                            # kafka/ | sqs/ | rabbitmq/
│   │   │
│   │   └── resources/
│   │       ├── application.properties                    # config padrão (produção)
│   │       └── db/
│   │           ├── migration/                            # DDL — Flyway (V{n}.0__)
│   │           └── testdata/                             # DML — seeds (somente profile dev)
│   │
│   └── test/
│       ├── java/{group-id}/
│       │   ├── api/rest/v1/resource/                     # @QuarkusTest + REST Assured (unit)
│       │   ├── domain/
│       │   │   ├── entity/                               # TemplateLoader (fixtures)
│       │   │   ├── request/                              # TemplateLoader (fixtures)
│       │   │   ├── response/                             # TemplateLoader (fixtures)
│       │   │   └── service/                              # unit + @ParameterizedTest
│       │   ├── cucumber/                                 # BDD integration
│       │   │   ├── config/
│       │   │   ├── step/
│       │   │   └── utils/
│       │   └── infrastructure/                           # Dev Services base
│       └── resources/
│           ├── application.properties                    # config de teste
│           └── features/                                 # .feature (Cucumber)
```

---

## Organização dos Pacotes

O projeto é organizado em **4 pacotes raiz**, cada um com uma responsabilidade clara:

```
{group-id}
├── api/                       → Resources JAX-RS, DTOs (request/response), exception mappers
│   ├── rest/v1/               → REST API versionada (ou grpc/, graphql/)
│   └── messaging/kafka/       → Consumers e producers Kafka (ou sqs/, rabbitmq/)
├── core/                      → Mappers (MapStruct) e grupos de validação
├── domain/                    → Entities, repositories, services, exceções
│   └── repository/mysql/      → Repositories com prefixo da tecnologia de banco
└── infrastructure/            → Configurações técnicas (métricas, docs, clients, messaging)
```

> **Nota:** Essa organização é uma decisão prática de pacotes — não se vincula a nenhum padrão arquitetural específico (Clean Architecture, Hexagonal, etc.).

### api — Resources, DTOs e Exception Mappers

Pacote responsável por **receber requisições** e **devolver respostas**.

| Pacote                        | Responsabilidade                                             | Convenção                              |
|-------------------------------|--------------------------------------------------------------|----------------------------------------|
| `api.exception`               | Tratamento global de exceções (`ExceptionMapper<T>`)         | Uma classe por tipo de exceção          |
| `api.rest.v1.resource`        | JAX-RS resources (`@Path` como `record`)                     | Um resource por recurso                |
| `api.rest.v1.openapi`         | Interfaces com anotações OpenAPI (`@Tag`, `@Operation`)      | Uma interface por resource             |
| `api.rest.v1.request`         | DTOs de entrada (`record` com Bean Validation)               | Sufixo `Request`                       |
| `api.rest.v1.response`        | DTOs de saída (`record`)                                     | Sufixo `Response`                      |
| `api.rest.v1.commons.enums`   | Enums exclusivos da API                                      | Espelham domain enums quando necessário |

**Padrões adotados:**

- **Protocolo como pacote**: `api.rest.`, `api.grpc.`, `api.graphql.`
- **Versionamento por path**: `/v1/`, `/v2/`, etc.
- **Resources como `record`**: Imutabilidade + injeção pelo construtor canônico (CDI).
- **Interface OpenAPI separada**: Mantém annotations Swagger fora do resource.
- **`ProblemDetail` (RFC 9457)**: Padrão de resposta de erro via `ExceptionMapper`.

### core — Mappers e Validação

Código **reutilizável** entre pacotes.

| Pacote              | Responsabilidade                              | Convenção                           |
|---------------------|-----------------------------------------------|-------------------------------------|
| `core.mapper`       | MapStruct mappers (request ↔ entity ↔ response) | Sufixo `Mapper`, interface + `MAPPER` singleton |
| `core.validation`   | Grupos de validação (Bean Validation groups)  | Interface marker (`Create`, `Update`) |

**Padrão de mapper:**

```java
@Mapper(componentModel = "cdi", nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface FooMapper {

    FooEntity toEntity(FooRequest request);
    FooResponse toResponse(FooEntity entity);
    void update(FooRequest request, @MappingTarget FooEntity entity);
}
```

> No Quarkus, use `componentModel = "cdi"` para que o mapper seja um bean CDI injetável via `@Inject`.

### domain — Entities, Repositories e Services

Contém **entidades, repositórios, serviços e exceções**.

| Pacote                      | Responsabilidade                              | Convenção                             |
|-----------------------------|-----------------------------------------------|---------------------------------------|
| `domain.entity`             | JPA entities com Panache (`PanacheEntityBase`) | Sufixo `Entity`, tabela `tb_*`       |
| `domain.entity.enums`       | Enums persistidos (com `fromValue()` estático)| Enum com `HashMap` lookup             |
| `domain.repository.mysql`   | Repositories para MySQL                       | Sufixo `Repository`, `PanacheRepositoryBase<Entity, UUID>` |
| `domain.repository.postgresql` | Repositories para PostgreSQL (se aplicável)| Mesmo padrão                          |
| `domain.repository.mongodb` | Repositories para MongoDB (se aplicável)      | `PanacheMongoRepositoryBase<Document, String>` |
| `domain.service`            | Serviços (`@ApplicationScoped`)               | Sufixo `Service`, injeção via construtor |
| `domain.exception`          | Exceções específicas                          | `ResourceNotFoundException`, `BusinessException` |

**Padrões adotados:**

- **Tecnologia do banco como pacote**: `domain.repository.mysql.`, `domain.repository.postgresql.`, `domain.repository.mongodb.`
- **Chaves primárias UUID** com `@GeneratedValue(strategy = GenerationType.UUID)`.
- **Relacionamentos LAZY** entre entidades.
- **`@Transactional`** nos services: `@Transactional` em métodos de escrita.
- **`@Counted` e `@Timed`** (Micrometer) nos métodos de service para métricas.
- **Panache Repository Pattern**: Repositories estendem `PanacheRepositoryBase` para queries type-safe.

### infrastructure — Configurações Técnicas

Configurações de infraestrutura e integrações técnicas.

| Pacote                             | Responsabilidade                         | Convenção                    |
|------------------------------------|------------------------------------------|------------------------------|
| `infrastructure.metrics`           | Beans de métricas customizados           | Sufixo `Config`              |
| `infrastructure.openapi`           | Config OpenAPI/Swagger                   | Ativo apenas em `dev`        |
| `infrastructure.messaging.kafka`   | Config Kafka                             | Sufixo `Config`              |
| `infrastructure.messaging.sqs`     | Config SQS                               | Sufixo `Config`              |
| `infrastructure.messaging.rabbitmq`| Config RabbitMQ                          | Sufixo `Config`              |
| `infrastructure.resilience`        | Config Fault Tolerance (Circuit Breaker, Retry, etc.) | Sufixo `Config`     |
| `infrastructure.security`          | Quarkus Security / JWT / OIDC config   | Sufixo `Config`              |
| `infrastructure.cors`              | CORS configuration                     | Sufixo `Config`              |
| `infrastructure.featureflag`       | Togglz feature flags                   | Sufixo `Config`              |

**Extensões esperadas nesta camada:**

- `infrastructure.client` — REST Clients declarativos via `@RegisterRestClient`.
- `infrastructure.cache` — Configuração de cache (`@CacheResult`, Redis).

---

## Pontos de Extensão

### API — REST / gRPC / GraphQL

A estrutura permite múltiplos protocolos sob o pacote `api`:

```
api/
├── exception/                       # compartilhado entre protocolos
│   └── ApiExceptionMapper.java
├── rest/                            # ── REST ──
│   └── v1/
│       ├── resource/
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

| Protocolo  | Dependência (Extension)                                  |
|------------|----------------------------------------------------------|
| REST       | `quarkus-rest` (RESTEasy Reactive)                       |
| gRPC       | `quarkus-grpc`                                           |
| GraphQL    | `quarkus-smallrye-graphql`                               |

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
│   ├── consumer/                    # @Incoming
│   ├── producer/                    # @Outgoing / Emitter
│   └── event/                       # DTOs/records dos eventos
├── sqs/                             # ── AWS SQS ──
│   ├── consumer/                    # SqsClient consumer
│   ├── producer/                    # SqsClient producer
│   └── event/
└── rabbitmq/                        # ── RabbitMQ ──
    ├── consumer/                    # @Incoming (SmallRye AMQP)
    ├── producer/                    # @Outgoing / Emitter
    └── event/

infrastructure/messaging/
├── kafka/
│   └── KafkaConfig.java
├── sqs/
│   └── SqsConfig.java
└── rabbitmq/
    └── RabbitConfig.java
```

| Broker     | Dependência (Extension)                                   |
|------------|-----------------------------------------------------------|
| Kafka      | `quarkus-messaging-kafka` (SmallRye Reactive Messaging)   |
| SQS        | `quarkus-amazon-sqs`                                      |
| RabbitMQ   | `quarkus-messaging-rabbitmq` (SmallRye Reactive Messaging)|

**Padrão de consumer (Kafka):**

```java
@ApplicationScoped
public class TransactionConsumer {

    private final TransactionService transactionService;

    TransactionConsumer(TransactionService transactionService) {
        this.transactionService = transactionService;
    }

    @Incoming("transaction-created-in")
    public void consume(TransactionCreatedEvent event) {
        transactionService.process(event);
    }
}
```

**Padrão de producer (Kafka):**

```java
@ApplicationScoped
public class NotificationProducer {

    @Inject
    @Channel("notification-request-out")
    Emitter<NotificationRequestEvent> emitter;

    public void send(NotificationRequestEvent event) {
        emitter.send(event);
    }
}
```

> No Quarkus, os canais (`@Incoming`/`@Outgoing`/`@Channel`) são mapeados para tópicos Kafka via `application.properties`:
> ```properties
> mp.messaging.incoming.transaction-created-in.connector=smallrye-kafka
> mp.messaging.incoming.transaction-created-in.topic=transaction-created
> mp.messaging.outgoing.notification-request-out.connector=smallrye-kafka
> mp.messaging.outgoing.notification-request-out.topic=notification-request
> ```

### Banco de Dados — MySQL / PostgreSQL / MongoDB

Os repositories são organizados com **prefixo da tecnologia de banco**:

```
domain/repository/
├── mysql/                           # ── MySQL ──
│   ├── UserRepository.java         # PanacheRepositoryBase<UserEntity, UUID>
│   ├── WalletRepository.java
│   └── TransactionRepository.java
├── postgresql/                      # ── PostgreSQL ──
│   └── AuditRepository.java        # PanacheRepositoryBase<AuditEntity, UUID>
└── mongodb/                         # ── MongoDB ──
    └── EventLogRepository.java      # PanacheMongoRepositoryBase<EventLogDocument, String>
```

| Banco       | Dependência (Extension)                              | Interface base                              |
|-------------|------------------------------------------------------|---------------------------------------------|
| MySQL       | `quarkus-jdbc-mysql` + `quarkus-hibernate-orm-panache` | `PanacheRepositoryBase<Entity, UUID>`      |
| PostgreSQL  | `quarkus-jdbc-postgresql` + `quarkus-hibernate-orm-panache` | `PanacheRepositoryBase<Entity, UUID>` |
| MongoDB     | `quarkus-mongodb-panache`                            | `PanacheMongoRepositoryBase<Document, String>` |

**Motivo:** quando a aplicação lida com múltiplos bancos (ex: MySQL para dados transacionais + MongoDB para logs de eventos), o prefixo torna explícito qual datasource cada repository utiliza.

---

## Convenções de Código

### Nomeação

| Elemento           | Convenção                                    | Exemplo                              |
|--------------------|----------------------------------------------|--------------------------------------|
| Pacote             | `lowercase` sem underscores                  | `api.rest.v1.resource`               |
| Classe             | `PascalCase`                                 | `UserResource`                       |
| Resource           | `{Recurso}Resource`                          | `UserResource`                       |
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

### Resources (JAX-RS)

- **Implementados como `record`**: Garante imutabilidade e injeção de dependências via CDI.
- **Implementam interface OpenAPI**: Mantém documentação separada da implementação.
- **Retornam `Response` ou `RestResponse<T>`**: Permite controle explícito de headers e status codes.
- **Anotações JAX-RS**: `@Path`, `@GET`, `@POST`, `@PUT`, `@DELETE`, `@Produces`, `@Consumes`.

### Entities

- **UUID como chave primária** em todas as entidades.
- **Extends `PanacheEntityBase`**: Para entidades que usam Panache (UUID não suporta `PanacheEntity` diretamente).
- **Fetch LAZY** em todos os relacionamentos.
- **Enums com `fromValue()`**: Lookup via `HashMap` estático para conversão segura.

---

## Configuração por Profile

O Quarkus usa **prefixos de profile** no `application.properties`:

| Profile     | Prefixo / Arquivo                  | Uso                                           |
|-------------|------------------------------------|-----------------------------------------------|
| *(default)* | `application.properties`           | Produção — configs via variáveis de ambiente  |
| `dev`       | `%dev.` prefix                     | Desenvolvimento local — Dev Services, Swagger  |
| `test`      | `%test.` prefix                    | Testes — Dev Services injeta configs automaticamente |

### Particularidades do Profile `dev`

- **Dev Services** habilitados (containers automáticos para banco, Kafka, etc.)
- **Quarkus Dev UI** disponível em `/q/dev-ui`
- **Swagger UI** ativo em `/q/swagger-ui`
- **Live Coding**: alterações refletem automaticamente (`quarkus dev`)
- **Continuous Testing**: testes rodam automaticamente durante o dev

### Exemplo de configuração:

```properties
# Produção (default)
quarkus.datasource.db-kind=mysql
quarkus.datasource.jdbc.url=${DB_URL}
quarkus.datasource.username=${DB_USER}
quarkus.datasource.password=${DB_PASS}

# Dev — usa Dev Services (container automático)
%dev.quarkus.datasource.devservices.enabled=true
%dev.quarkus.swagger-ui.always-include=true
%dev.quarkus.log.console.json=false

# Test — Dev Services para testes
%test.quarkus.datasource.devservices.enabled=true
```

---

## Banco de Dados e Migrations

### Convenção de Migrations

- **DDL (schema)**: Em `db/migration/`, prefixo `V{n}.0__`
- **DML (seeds)**: Em `db/testdata/`, prefixo `V{ano.mes.dia.seq}.0__seed_`
- Seeds são carregados **somente no profile `dev`**
- Hibernate DDL auto: `none` — Flyway é o único responsável pelo schema

### Configuração Flyway:

```properties
quarkus.flyway.migrate-at-start=true
quarkus.flyway.locations=db/migration
%dev.quarkus.flyway.locations=db/migration,db/testdata
```

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
- **`@QuarkusTest`**: Para testes de integração que carregam o contexto CDI completo.
- **REST Assured**: Para testar endpoints REST declarativamente.
- **`@ParameterizedTest`**: Com `@ValueSource` para testar variações de input.
- **Dev Services**: Containers de banco/Kafka/Redis são provisionados automaticamente nos testes.
- **Cucumber**: Features em `src/test/resources/features/`, steps em `cucumber.step`.

### Organização de Testes

```
test/
├── java/{group-id}/
│   ├── api/rest/v1/resource/
│   │   └── FooResourceTest.java             # @QuarkusTest + REST Assured
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
│       └── DevServicesBaseTest.java          # Dev Services base
└── resources/
    ├── application.properties               # %test configs
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

# Continuous Testing (durante dev mode)
./mvnw quarkus:dev     # testes rodam automaticamente ao alterar código
```

---

## Build e Qualidade

| Plugin                      | Fase            | Função                            |
|-----------------------------|-----------------|-----------------------------------|
| `quarkus-maven-plugin`      | package         | Gera uber-jar ou fast-jar         |
| `maven-surefire-plugin`     | test            | Roda testes unitários (`*Test`)   |
| `maven-failsafe-plugin`     | verify          | Roda testes de integração (`*IT`) |
| `jacoco-maven-plugin`       | test/verify     | Cobertura de código               |
| `pitest-maven`              | manual          | Testes de mutação                 |

### Build Modes

```bash
# JVM mode (fast-jar)
./mvnw package

# JVM mode (uber-jar)
./mvnw package -Dquarkus.package.jar.type=uber-jar

# Native mode (GraalVM)
./mvnw package -Dnative

# Native mode (container build — sem GraalVM local)
./mvnw package -Dnative -Dquarkus.native.container-build=true
```

---

## Resiliência

O projeto utiliza **SmallRye Fault Tolerance** (implementação da spec MicroProfile Fault Tolerance) para padrões de resiliência em chamadas a serviços externos.

### Dependência (Extension)

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-fault-tolerance</artifactId>
</dependency>
```

### Padrões Disponíveis

| Padrão            | Anotação                        | Quando usar                                           |
|-------------------|---------------------------------|-------------------------------------------------------|
| **Circuit Breaker** | `@CircuitBreaker`             | Chamadas a serviços externos instáveis                |
| **Retry**           | `@Retry`                      | Falhas transitórias (timeout, 503)                    |
| **Timeout**         | `@Timeout`                    | Definir tempo máximo para chamadas                    |
| **Bulkhead**        | `@Bulkhead`                   | Isolar chamadas para evitar esgotamento de threads    |
| **Fallback**        | `@Fallback`                   | Resposta alternativa quando todas as tentativas falham |
| **Rate Limit**      | `@RateLimit`                  | Proteger endpoints contra excesso de requisições      |

### Organização de Pacotes

```
infrastructure/resilience/
├── FaultToleranceConfig.java         # Customizações globais (se necessário)
└── CircuitBreakerObserver.java       # CDI observer para logging de eventos
```

### Configuração (application.properties)

```properties
# ── Circuit Breaker (global defaults) ──
PaymentService/charge/CircuitBreaker/requestVolumeThreshold=10
PaymentService/charge/CircuitBreaker/failureRatio=0.5
PaymentService/charge/CircuitBreaker/delay=30000
PaymentService/charge/CircuitBreaker/successThreshold=3

# ── Retry (global defaults) ──
PaymentService/charge/Retry/maxRetries=3
PaymentService/charge/Retry/delay=500
PaymentService/charge/Retry/retryOn=java.io.IOException,java.util.concurrent.TimeoutException

# ── Timeout ──
PaymentService/charge/Timeout/value=5000

# ── Bulkhead ──
PaymentService/charge/Bulkhead/value=25

# ── Global overrides (aplica a todos os métodos) ──
# Retry/maxRetries=3
# CircuitBreaker/delay=30000
```

> No Quarkus, a configuração de Fault Tolerance segue o padrão MicroProfile: `{classe}/{método}/{anotação}/{propriedade}` no `application.properties`.

### Padrão de uso no Service

```java
@ApplicationScoped
public class PaymentService {

    private final PaymentClient paymentClient;

    PaymentService(@RestClient PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }

    @CircuitBreaker(requestVolumeThreshold = 10, failureRatio = 0.5, delay = 30_000)
    @Retry(maxRetries = 3, delay = 500)
    @Timeout(5000)
    @Fallback(fallbackMethod = "fallbackCharge")
    public PaymentResponse charge(PaymentRequest request) {
        return paymentClient.charge(request);
    }

    private PaymentResponse fallbackCharge(PaymentRequest request) {
        Log.warnf("Fallback acionado para payment charge");
        return PaymentResponse.unavailable();
    }
}
```

> **Ordem de execução das anotações** (de fora para dentro): `Fallback → Retry → CircuitBreaker → Timeout → Bulkhead → chamada real`.

### Integração com REST Client

Fault Tolerance pode ser aplicado diretamente na interface `@RegisterRestClient`:

```java
@RegisterRestClient(configKey = "payment-api")
public interface PaymentClient {

    @POST
    @Path("/charges")
    @CircuitBreaker(requestVolumeThreshold = 10)
    @Retry(maxRetries = 3)
    @Timeout(5000)
    PaymentResponse charge(PaymentRequest request);
}
```

### Métricas de Resiliência

O SmallRye Fault Tolerance expõe automaticamente métricas via Micrometer:
- `ft.circuitbreaker.state.total`
- `ft.circuitbreaker.calls.total`
- `ft.retry.calls.total`
- `ft.retry.retries.total`
- `ft.timeout.calls.total`
- `ft.bulkhead.concurrent.executions`

Todas acessíveis via `/q/metrics`.

---

## Segurança (AuthN/AuthZ)

O projeto usa **Quarkus Security** com **SmallRye JWT** ou **OIDC** para autenticação via JWT/OAuth2.

### Dependências (Extensions)

```xml
<!-- Opção 1: SmallRye JWT (validação local de tokens) -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-jwt</artifactId>
</dependency>

<!-- Opção 2: OIDC (validação via Identity Provider — Keycloak, Auth0, etc.) -->
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-oidc</artifactId>
</dependency>
```

### Organização de Pacotes

```
infrastructure/security/
├── SecurityConfig.java              # Customizações de segurança (se necessário)
├── RolesConstants.java              # Constantes de roles
└── JwtClaimExtractor.java           # Utilitário para extrair claims customizados
```

### Configuração (application.properties)

```properties
# ── SmallRye JWT ──
mp.jwt.verify.publickey.location=META-INF/resources/publicKey.pem
mp.jwt.verify.issuer=${JWT_ISSUER_URI}
smallrye.jwt.path.groups=realm_access/roles

# ── Ou OIDC ──
quarkus.oidc.auth-server-url=${OIDC_SERVER_URL}
quarkus.oidc.client-id=${OIDC_CLIENT_ID}
quarkus.oidc.application-type=service
quarkus.oidc.token.issuer=${JWT_ISSUER_URI}
```

### Configuração de Permissões (HTTP Policy)

```properties
# Paths públicos
quarkus.http.auth.permission.public.paths=/q/health/*,/q/openapi,/q/swagger-ui/*
quarkus.http.auth.permission.public.policy=permit

# Paths autenticados
quarkus.http.auth.permission.authenticated.paths=/v1/*
quarkus.http.auth.permission.authenticated.policy=authenticated
```

### RBAC (Role-Based Access Control)

```java
@Path("/v1/users")
@Authenticated
public record UserResource(UserService userService) {

    @DELETE
    @Path("/{id}")
    @RolesAllowed("ADMIN")
    public Response delete(@PathParam("id") UUID id) {
        userService.delete(id);
        return Response.noContent().build();
    }

    @GET
    @RolesAllowed({"ADMIN", "MANAGER"})
    public List<UserResponse> findAll() {
        return userService.findAll();
    }
}
```

| Anotação           | Uso                                        |
|--------------------|--------------------------------------------|
| `@Authenticated`   | Requer token válido (qualquer role)        |
| `@RolesAllowed`    | Requer role específica                     |
| `@DenyAll`         | Bloqueia acesso total                      |
| `@PermitAll`       | Permite acesso sem autenticação            |

> **Dev Services:** Quarkus provisiona automaticamente um Keycloak em dev mode com `quarkus-oidc`. Use `quarkus.keycloak.devservices.realm-path=dev-realm.json` para pré-configurar roles.

---

## Tratamento Global de Erros

### Padrão: ProblemDetail (RFC 9457) via ExceptionMapper

No Quarkus, o tratamento global de exceções é feito com JAX-RS `ExceptionMapper<T>`.

### Organização

```
api/exception/
├── ApiExceptionMapper.java            # @Provider — mapper global
├── ProblemDetailFactory.java          # Factory para ProblemDetail padronizados
└── ErrorCode.java                     # enum com códigos de erro do domínio
```

### Padrão de ExceptionMapper

```java
@Provider
public class ApiExceptionMapper implements ExceptionMapper<Exception> {

    @Override
    public Response toResponse(Exception exception) {
        return switch (exception) {
            case ResourceNotFoundException e -> buildResponse(
                Response.Status.NOT_FOUND, "Recurso não encontrado", e.getMessage(), "RESOURCE_NOT_FOUND"
            );
            case BusinessException e -> buildResponse(
                Response.Status.fromStatusCode(422), "Erro de negócio", e.getMessage(), e.getCode()
            );
            case ConstraintViolationException e -> buildResponse(
                Response.Status.BAD_REQUEST, "Erro de validação", formatViolations(e), "VALIDATION_ERROR"
            );
            default -> buildResponse(
                Response.Status.INTERNAL_SERVER_ERROR, "Erro interno", "Erro inesperado", "INTERNAL_ERROR"
            );
        };
    }

    private Response buildResponse(Response.Status status, String title, String detail, String errorCode) {
        var problem = Map.of(
            "type", "https://api.example.com/errors/" + errorCode.toLowerCase().replace("_", "-"),
            "title", title,
            "status", status.getStatusCode(),
            "detail", detail,
            "errorCode", errorCode,
            "timestamp", Instant.now().toString()
        );
        return Response.status(status).entity(problem).type(MediaType.APPLICATION_JSON).build();
    }
}
```

### Formato de Resposta (JSON)

```json
{
  "type": "https://api.example.com/errors/resource-not-found",
  "title": "Recurso não encontrado",
  "status": 404,
  "detail": "Usuário com ID 550e8400-e29b-41d4-a716-446655440000 não encontrado",
  "errorCode": "RESOURCE_NOT_FOUND",
  "timestamp": "2026-02-22T10:30:00Z"
}
```

### Mapeamento de Exceções

| Exceção                        | Status HTTP | errorCode              |
|--------------------------------|-------------|------------------------|
| `ResourceNotFoundException`    | 404         | `RESOURCE_NOT_FOUND`   |
| `BusinessException`           | 422         | Dinâmico (do domínio)  |
| `ConstraintViolationException` | 400         | `VALIDATION_ERROR`     |
| `ForbiddenException`          | 403         | `ACCESS_DENIED`        |
| `NotAuthorizedException`      | 401         | `UNAUTHORIZED`         |
| `Exception` (genérica)        | 500         | `INTERNAL_ERROR`       |

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
│   ├── resource/
│   ├── request/
│   ├── response/
│   └── openapi/
└── v2/
    ├── resource/
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

### Padrão com Panache

```java
@Path("/v1/users")
@Authenticated
public record UserResource(UserService userService) {

    @GET
    public Response findAll(
            @QueryParam("page") @DefaultValue("0") int page,
            @QueryParam("size") @DefaultValue("20") int size,
            @QueryParam("sort") @DefaultValue("createdAt,desc") String sort) {

        var result = userService.findAll(page, size, sort);
        return Response.ok(result).build();
    }
}
```

### Padrão no Service

```java
@ApplicationScoped
public class UserService {

    private final UserRepository userRepository;
    private final UserMapper userMapper;

    @Transactional(Transactional.TxType.SUPPORTS)
    public PaginatedResponse<UserResponse> findAll(int page, int size, String sort) {
        var panacheQuery = userRepository.findAll(Sort.by(parseSortField(sort)));
        var list = panacheQuery.page(Page.of(page, size)).list();
        var total = panacheQuery.count();

        return new PaginatedResponse<>(
            list.stream().map(userMapper::toResponse).toList(),
            page, size, total
        );
    }
}
```

### Record de Resposta Paginada

```java
public record PaginatedResponse<T>(
    List<T> content,
    PageInfo page
) {
    public PaginatedResponse(List<T> content, int number, int size, long totalElements) {
        this(content, new PageInfo(size, number, totalElements, (int) Math.ceil((double) totalElements / size)));
    }

    public record PageInfo(int size, int number, long totalElements, int totalPages) {}
}
```

### Limites

```properties
# Validação no Resource — valores padrão e limites
# page >= 0, size entre 1 e 100
app.pagination.max-size=100
app.pagination.default-size=20
```

---

## CORS

### Configuração (application.properties)

```properties
# ── Produção ──
quarkus.http.cors=true
quarkus.http.cors.origins=${APP_CORS_ORIGINS:https://app.example.com}
quarkus.http.cors.methods=GET,POST,PUT,DELETE,PATCH,OPTIONS
quarkus.http.cors.headers=Content-Type,Authorization,Accept
quarkus.http.cors.exposed-headers=Location,X-Total-Count
quarkus.http.cors.access-control-max-age=1H
quarkus.http.cors.access-control-allow-credentials=true
```

### Configuração por Profile

```properties
# ── application.properties (produção) ──
quarkus.http.cors.origins=https://app.example.com

# ── %dev profile ──
%dev.quarkus.http.cors.origins=http://localhost:3000,http://localhost:5173

# ── %test profile ──
%test.quarkus.http.cors.origins=*
```

### Organização de Pacotes

```
infrastructure/cors/
└── CorsConfig.java              # Apenas se precisar de config programática customizada
```

> **Nota:** No Quarkus, a maioria das configurações CORS é feita em `application.properties`. Use classe de configuração somente para lógica dinâmica (ex: CORS origins de banco de dados).

---

## Feature Flags

### Biblioteca: Togglz (CDI Integration)

```xml
<dependency>
    <groupId>org.togglz</groupId>
    <artifactId>togglz-cdi</artifactId>
</dependency>
<dependency>
    <groupId>org.togglz</groupId>
    <artifactId>togglz-console</artifactId>
</dependency>
```

### Organização de Pacotes

```
infrastructure/featureflag/
├── FeatureFlagConfig.java            # @Produces — FeatureManager CDI producer
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

### CDI Producer

```java
@ApplicationScoped
public class FeatureFlagConfig {

    @Produces
    @ApplicationScoped
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
@ApplicationScoped
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

### Configuração (Quarkus 3.x + JDK 25)

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-virtual-threads</artifactId>
</dependency>
```

No Quarkus, virtual threads são **opt-in por endpoint** com `@RunOnVirtualThread`.

### Uso no Resource

```java
@Path("/v1/reports")
@Authenticated
public record ReportResource(ReportService reportService) {

    @GET
    @RunOnVirtualThread
    public List<ReportResponse> generate() {
        return reportService.generateAll();  // operação blocking pesada
    }
}
```

### Uso no Consumer (Mensageria)

```java
@ApplicationScoped
public class OrderConsumer {

    @Incoming("orders")
    @RunOnVirtualThread
    public void process(OrderEvent event) {
        // processamento blocking em virtual thread
    }
}
```

### Impacto

| Componente          | Efeito                                                          |
|---------------------|-----------------------------------------------------------------|
| **RESTEasy Reactive** | Endpoint roda em virtual thread (blocking sem impacto)        |
| **Messaging**       | Consumer roda em virtual thread                                |
| **Scheduler**       | `@Scheduled` + `@RunOnVirtualThread`                           |
| **JPA/JDBC**        | Chamadas blocking funcionam sem degradar throughput             |

### Cuidados

- **Synchronized blocks**: Evite `synchronized` — use `ReentrantLock` para evitar thread pinning.
- **ThreadLocal**: Use `ScopedValue` (JDK 25) em vez de `ThreadLocal`.
- **Connection pools**: Ajuste `quarkus.datasource.jdbc.max-size` pois virtual threads podem criar muitas conexões simultâneas.

```properties
quarkus.datasource.jdbc.max-size=50
quarkus.datasource.jdbc.acquisition-timeout=10S
```

### Verificação

```java
@PostConstruct
void checkVirtualThreads() {
    Log.infof("Virtual threads? %s", Thread.currentThread().isVirtual());
}
```

> **Nota:** Diferente do Spring Boot, no Quarkus virtual threads são **explícitos** por endpoint — isso dá controle preciso sobre onde usar.

---

## Observabilidade

| Aspecto         | Tecnologia                       | Detalhes                                                    |
|-----------------|----------------------------------|-------------------------------------------------------------|
| **Métricas**    | Micrometer                       | `@Timed`, `@Counted` nos services                           |
| **Tracing**     | OpenTelemetry                    | Propagação automática de traceId; exportador configurável   |
| **Logging**     | JBoss Logging + JSON formatter   | JSON estruturado com traceId, spanId                        |
| **Health**      | SmallRye Health                  | `/q/health/live`, `/q/health/ready`, `/q/health/started`    |
| **API Docs**    | SmallRye OpenAPI / Swagger UI    | `/q/swagger-ui` (apenas em dev)                              |
| **Dev UI**      | Quarkus Dev UI                   | `/q/dev-ui` (apenas em dev mode)                             |

---

## Docker (Native / JVM)

### Dockerfile JVM

```dockerfile
# 1) Build — compila o JAR
FROM maven:3.9-eclipse-temurin-25 AS build
WORKDIR /workspace
COPY . .
RUN mvn -DskipTests package

# 2) Runtime — usa o fast-jar
FROM eclipse-temurin:25-jre
WORKDIR /opt/app
COPY --from=build /workspace/target/quarkus-app/ /opt/app/quarkus-app/
ENV JAVA_OPTS="-Djava.util.logging.manager=org.jboss.logmanager.LogManager"
ENTRYPOINT ["java", "-jar", "/opt/app/quarkus-app/quarkus-run.jar"]
```

### Dockerfile Native

```dockerfile
# 1) Build — compila o executável nativo
FROM quay.io/quarkus/ubi-quarkus-mandrel-builder-image:jdk-25 AS build
WORKDIR /workspace
COPY --chown=quarkus:quarkus . .
RUN ./mvnw -DskipTests package -Dnative

# 2) Runtime — imagem mínima
FROM quay.io/quarkus/quarkus-micro-image:2.0
WORKDIR /opt/app
COPY --from=build /workspace/target/*-runner /opt/app/application
EXPOSE 8080
ENTRYPOINT ["/opt/app/application", "-Dquarkus.http.host=0.0.0.0"]
```

| Estágio   | Base image                              | O que faz                                  |
|-----------|-----------------------------------------|--------------------------------------------|
| `build`   | `maven:3.9-eclipse-temurin-25` (JVM) ou `mandrel-builder` (Native) | Compila o código |
| `runtime` | `eclipse-temurin:25-jre` (JVM) ou `quarkus-micro-image` (Native)   | Imagem final leve |

**Benefício Native:** Startup em milissegundos, footprint de memória reduzido — ideal para serverless e containers.

### Inicialização Local

```bash
# 1. Dev mode (Dev Services provisiona containers automaticamente)
./mvnw quarkus:dev

# 2. Ou com docker-compose (sem Dev Services)
docker-compose up -d
./mvnw quarkus:dev -Dquarkus.devservices.enabled=false
```

---

## Guia para Criar Novo Recurso

Ao adicionar um novo recurso, siga esta checklist:

### 1. Domain

- [ ] `domain/entity/FooEntity.java` — `@Entity`, `PanacheEntityBase`, tabela `tb_foo`, UUID PK
- [ ] `domain/entity/enums/` — Enums necessários (se houver)
- [ ] `domain/repository/mysql/FooRepository.java` — `PanacheRepositoryBase<FooEntity, UUID>`
- [ ] `domain/service/FooService.java` — `@ApplicationScoped`, `@Transactional`, `@Timed`/`@Counted`
- [ ] `domain/exception/` — Exceções específicas (se necessário)

### 2. Core

- [ ] `core/mapper/FooMapper.java` — MapStruct interface com `componentModel = "cdi"`

### 3. API (REST)

- [ ] `api/rest/v1/request/FooRequest.java` — `record` com Bean Validation
- [ ] `api/rest/v1/request/FooQueryRequest.java` — `record` para filtros (se necessário)
- [ ] `api/rest/v1/response/FooResponse.java` — `record`
- [ ] `api/rest/v1/openapi/FooOpenApi.java` — Interface OpenAPI
- [ ] `api/rest/v1/resource/FooResource.java` — `record` `@Path` JAX-RS

### 4. API (Mensageria) — se aplicável

- [ ] `api/messaging/kafka/event/FooCreatedEvent.java` — `record` do evento
- [ ] `api/messaging/kafka/producer/FooProducer.java` — `@Channel` + `Emitter`
- [ ] `api/messaging/kafka/consumer/FooConsumer.java` — `@Incoming`

### 5. Database

- [ ] `db/migration/V{n}.0__create_tb_foo.sql` — DDL
- [ ] `db/testdata/V{data}.0__seed_tb_foo_data.sql` — Seeds (opcional)

### 6. Testes

- [ ] `test/.../domain/entity/FooEntityTemplateLoader.java` — Fixture
- [ ] `test/.../domain/request/FooRequestTemplateLoader.java` — Fixture
- [ ] `test/.../domain/response/FooResponseTemplateLoader.java` — Fixture
- [ ] `test/.../domain/service/FooServiceTest.java` — Unit test
- [ ] `test/.../api/rest/v1/resource/FooResourceTest.java` — `@QuarkusTest` + REST Assured
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
