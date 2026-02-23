# Project Structure

> **Objetivo deste documento:** Servir como referência organizacional para o GitHub Copilot e desenvolvedores.
> Foca exclusivamente na **forma de organizar pacotes, diretórios e convenções**, sem detalhar regras de negócio.
>
> **Localização recomendada:** `.github/copilot-instructions.md` (veja [Onde colocar este arquivo](#onde-colocar-este-arquivo-para-o-copilot)).

---

## Sumário

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
- [Configuração por Profile](#configuração-por-profile)
- [Banco de Dados e Migrations](#banco-de-dados-e-migrations)
- [Testes](#testes)
- [Build e Qualidade](#build-e-qualidade)
- [Observabilidade](#observabilidade)
- [Docker (CDS)](#docker-cds)
- [Guia para Criar Novo Recurso](#guia-para-criar-novo-recurso)
- [Onde colocar este arquivo para o Copilot](#onde-colocar-este-arquivo-para-o-copilot)

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

**Extensões esperadas nesta camada:**

- `infrastructure.client` — HTTP clients declarativos via `@HttpExchange`.
- `infrastructure.security` — Configuração Spring Security.
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
