# Project Structure — Jakarta EE
> **Objetivo deste documento:** Servir como referencia organizacional para o GitHub Copilot e desenvolvedores.
> Foca exclusivamente na **forma de organizar pacotes, diretorios e convencoes**, sem detalhar regras de negocio.
>
> **Localizacao recomendada:** `.github/copilot-instructions.md` (veja [Onde colocar este arquivo](#onde-colocar-este-arquivo-para-o-copilot)).
---
## Sumario
- [Project Structure — Jakarta EE](#project-structure--jakarta-ee)
  - [Sumario](#sumario)
  - [Stack Tecnologica](#stack-tecnologica)
  - [| JSON                | Jakarta JSON Binding (JSON-B 3.0) + Jakarta JSON Processing (JSON-P 2.1) |](#-json-----------------jakarta-json-binding-json-b-30--jakarta-json-processing-json-p-21-)
  - [Runtimes Compativeis](#runtimes-compativeis)
  - [Estrutura de Diretorios](#estrutura-de-diretorios)
  - [Organizacao dos Pacotes](#organizacao-dos-pacotes)
    - [api — Resources, DTOs e Exception Mappers](#api--resources-dtos-e-exception-mappers)
    - [core — Mappers e Validacao](#core--mappers-e-validacao)
    - [domain — Entities, Repositories e Services](#domain--entities-repositories-e-services)
    - [infrastructure — Configuracoes Tecnicas](#infrastructure--configuracoes-tecnicas)
  - [| `infrastructure.messaging`  | Configuracao JMS / Kafka                      | `@JMSConnectionFactory`, producers CDI |](#-infrastructuremessaging---configuracao-jms--kafka-----------------------jmsconnectionfactory-producers-cdi-)
  - [Pontos de Extensao](#pontos-de-extensao)
    - [API — REST / WebSocket / gRPC](#api--rest--websocket--grpc)
    - [Mensageria — JMS / Kafka / RabbitMQ](#mensageria--jms--kafka--rabbitmq)
    - [Banco de Dados — MySQL / PostgreSQL / MongoDB](#banco-de-dados--mysql--postgresql--mongodb)
  - [Convencoes de Codigo](#convencoes-de-codigo)
    - [Nomeacao](#nomeacao)
    - [DTOs](#dtos)
    - [Resources (JAX-RS)](#resources-jax-rs)
    - [Entities](#entities)
  - [Configuracao por Profile](#configuracao-por-profile)
    - [Particularidades do Profile `dev`](#particularidades-do-profile-dev)
    - [MicroProfile Config](#microprofile-config)
  - [Banco de Dados e Migrations](#banco-de-dados-e-migrations)
    - [Convencao de Migrations](#convencao-de-migrations)
    - [Configuracao do DataSource](#configuracao-do-datasource)
  - [Testes](#testes)
    - [Estrutura](#estrutura)
    - [Convencoes de Teste](#convencoes-de-teste)
    - [Organizacao de Testes](#organizacao-de-testes)
    - [Comandos de Teste](#comandos-de-teste)
  - [Build e Qualidade](#build-e-qualidade)
  - [Resiliencia](#resiliencia)
    - [Dependencia (MicroProfile Fault Tolerance)](#dependencia-microprofile-fault-tolerance)
    - [Padroes Disponiveis](#padroes-disponiveis)
    - [Organizacao de Pacotes](#organizacao-de-pacotes)
    - [Configuracao](#configuracao)
    - [Padrao de uso no Service](#padrao-de-uso-no-service)
    - [Metricas de Resiliencia](#metricas-de-resiliencia)
  - [Seguranca (AuthN/AuthZ)](#seguranca-authnauthz)
    - [Dependencias](#dependencias)
    - [Organizacao de Pacotes](#organizacao-de-pacotes-1)
    - [Configuracao](#configuracao-1)
    - [RBAC (Role-Based Access Control)](#rbac-role-based-access-control)
  - [Tratamento Global de Erros](#tratamento-global-de-erros)
    - [Padrao: ProblemDetail (RFC 9457) via ExceptionMapper](#padrao-problemdetail-rfc-9457-via-exceptionmapper)
    - [Organizacao](#organizacao)
    - [Padrao de ExceptionMapper](#padrao-de-exceptionmapper)
    - [Formato de Resposta (JSON)](#formato-de-resposta-json)
    - [Mapeamento de Excecoes](#mapeamento-de-excecoes)
  - [| `Exception` (generico)        | 500         | `errors/internal`              |](#-exception-generico---------500----------errorsinternal--------------)
  - [Versionamento de API](#versionamento-de-api)
    - [Estrategia: Versionamento por Path](#estrategia-versionamento-por-path)
    - [Organizacao de Pacotes](#organizacao-de-pacotes-2)
    - [Regras](#regras)
  - [Paginacao e Ordenacao](#paginacao-e-ordenacao)
    - [Convencao de Query Parameters](#convencao-de-query-parameters)
    - [Formato de Resposta Paginada](#formato-de-resposta-paginada)
    - [Padrao no Resource](#padrao-no-resource)
    - [Padrao no Service](#padrao-no-service)
    - [Limites](#limites)
  - [| Campos ordenacao permitidos | Definidos por resource |](#-campos-ordenacao-permitidos--definidos-por-resource-)
  - [CORS](#cors)
    - [Configuracao via Filter](#configuracao-via-filter)
    - [Configuracao por Profile](#configuracao-por-profile-1)
  - [Feature Flags](#feature-flags)
    - [Biblioteca: Togglz (CDI Integration)](#biblioteca-togglz-cdi-integration)
    - [Definicao das Features](#definicao-das-features)
    - [Uso no Service](#uso-no-service)
  - [Virtual Threads](#virtual-threads)
    - [Configuracao (Jakarta EE 11 + JDK 21+)](#configuracao-jakarta-ee-11--jdk-21)
    - [Impacto](#impacto)
    - [Cuidados](#cuidados)
    - [Verificacao](#verificacao)
  - [Observabilidade](#observabilidade)
  - [Docker](#docker)
    - [Dockerfile (WildFly)](#dockerfile-wildfly)
    - [Inicializacao Local](#inicializacao-local)
  - [Guia para Criar Novo Recurso](#guia-para-criar-novo-recurso)
    - [1. Domain](#1-domain)
    - [2. Core](#2-core)
    - [3. API (REST)](#3-api-rest)
    - [4. API (Mensageria) — se aplicavel](#4-api-mensageria--se-aplicavel)
    - [5. Database](#5-database)
    - [6. Testes](#6-testes)
  - [Onde colocar este arquivo para o Copilot](#onde-colocar-este-arquivo-para-o-copilot)
    - [1. Instrucoes no nivel do repositorio (recomendado)](#1-instrucoes-no-nivel-do-repositorio-recomendado)
    - [2. Instrucoes por workspace (VS Code settings)](#2-instrucoes-por-workspace-vs-code-settings)
    - [3. Instrucao inline (via `.instructions.md`)](#3-instrucao-inline-via-instructionsmd)
    - [Recomendacao final](#recomendacao-final)
---
## Stack Tecnologica
| Categoria           | Tecnologia                                                   |
|---------------------|--------------------------------------------------------------|
| **Java**            | 21+ (recomendado 25)                                         |
| **Jakarta EE**      | 11 (Platform / Web Profile)                                  |
| **Build Tool**      | Maven (wrapper incluido)                                     |
| Web / API           | Jakarta RESTful Web Services (JAX-RS 4.0) — *extensivel para WebSocket/gRPC* |
| Persistencia        | Jakarta Persistence (JPA 3.2) + Jakarta Data 1.0             |
| Migrations          | Flyway                                                       |
| Validacao           | Jakarta Bean Validation 3.1                                  |
| Mapeamento          | MapStruct                                                    |
| Documentacao API    | MicroProfile OpenAPI (Swagger UI)                            |
| HTTP Client         | Jakarta REST Client / MicroProfile REST Client               |
| Metricas            | MicroProfile Metrics / Micrometer                            |
| Tracing             | MicroProfile Telemetry (OpenTelemetry)                       |
| Logging             | JBoss Logging / SLF4J + JSON formatter                       |
| Testes Unitarios    | JUnit 5 + Mockito + REST Assured                             |
| Testes Integracao   | Arquillian + Testcontainers + Cucumber                       |
| Testes de Carga     | k6 (smoke, load, stress)                                     |
| Resiliencia         | MicroProfile Fault Tolerance (Circuit Breaker, Retry, Timeout, Bulkhead, Fallback) |
| Seguranca           | Jakarta Security 4.0 + MicroProfile JWT                     |
| Feature Flags       | Togglz (CDI integration)                                     |
| Virtual Threads     | Jakarta Concurrency 3.1 (`ManagedExecutorService` + Virtual Threads) |
| Qualidade           | JaCoCo, PIT (mutation testing)                               |
| CDI                 | Jakarta CDI 4.1 (Contexts and Dependency Injection)          |
| JSON                | Jakarta JSON Binding (JSON-B 3.0) + Jakarta JSON Processing (JSON-P 2.1) |
---
## Runtimes Compativeis
Jakarta EE e baseado em especificacoes — o mesmo WAR/EAR roda em qualquer runtime certificado:
| Runtime        | Vendor           | Perfil           | Descricao                                          |
|----------------|------------------|------------------|----------------------------------------------------|
| **WildFly**    | Red Hat          | Full Platform    | Referencia deste guia — leve, modular, open source |
| **Payara**     | Payara Services  | Full Platform    | Fork do GlassFish com suporte comercial            |
| **Open Liberty** | IBM            | Full Platform    | Modular, cloud-native, excelente para containers   |
| **TomEE**      | Apache           | Web Profile      | Apache Tomcat + Jakarta EE Web Profile             |
| **GlassFish**  | Eclipse          | Full Platform    | Implementacao de referencia (RI)                   |
> **Nota:** Este guia usa WildFly como runtime de referencia, mas a estrutura de codigo e 100% portavel para qualquer runtime certificado.
---
## Estrutura de Diretorios
```
app/
├── pom.xml
├── Dockerfile
├── docker-compose.yml
│
├── src/
│   ├── main/
│   │   ├── java/{group-id}/
│   │   │   │
│   │   │   ├── api/                                      # ── RESOURCES, DTOs, EXCEPTION MAPPERS ──
│   │   │   │   ├── exception/
│   │   │   │   │   └── ApiExceptionMapper.java           # implements ExceptionMapper<T>
│   │   │   │   └── rest/                                 # protocolo: rest/ | websocket/ | grpc/
│   │   │   │       └── v1/                               # versionamento por path
│   │   │   │           ├── commons/enums/                # enums exclusivos da API
│   │   │   │           ├── resource/                     # @Path JAX-RS resources
│   │   │   │           ├── openapi/                      # interfaces @Tag/@Operation (OpenAPI)
│   │   │   │           ├── request/                      # records de entrada (DTOs)
│   │   │   │           └── response/                     # records de saida (DTOs)
│   │   │   │
│   │   │   ├── core/                                     # ── MAPPERS E VALIDACAO ──
│   │   │   │   ├── mapper/                               # MapStruct mappers
│   │   │   │   └── validation/                           # grupos de validacao
│   │   │   │
│   │   │   ├── domain/                                   # ── ENTITIES, REPOSITORIES, SERVICES ──
│   │   │   │   ├── entity/                               # @Entity JPA
│   │   │   │   │   └── enums/                            # enums persistidos
│   │   │   │   ├── exception/                            # excecoes de dominio
│   │   │   │   ├── repository/                           # tecnologia do banco como prefixo
│   │   │   │   │   └── mysql/                            # mysql/ | postgresql/ | mongodb/
│   │   │   │   └── service/                              # @ApplicationScoped (CDI)
│   │   │   │
│   │   │   └── infrastructure/                           # ── CONFIGURACOES TECNICAS ──
│   │   │       ├── config/                               # @ConfigProperty producers
│   │   │       ├── metrics/                              # custom metrics
│   │   │       ├── security/                             # Jakarta Security configs
│   │   │       ├── cors/                                 # CORS filter
│   │   │       ├── featureflag/                          # Togglz / Feature Flags
│   │   │       ├── json/                                 # JSON-B configuracao
│   │   │       └── messaging/                            # jms/ | kafka/ | rabbitmq/
│   │   │
│   │   ├── resources/
│   │   │   └── META-INF/
│   │   │       ├── persistence.xml                       # JPA persistence unit
│   │   │       ├── beans.xml                             # CDI bean discovery (bean-discovery-mode="all")
│   │   │       └── microprofile-config.properties        # MicroProfile Config
│   │   │
│   │   └── webapp/
│   │       └── WEB-INF/
│   │           └── web.xml                               # (opcional) configuracao do servlet
│   │
│   └── test/
│       ├── java/{group-id}/
│       │   ├── api/rest/v1/resource/                     # Arquillian / REST Assured (integration)
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
│           ├── arquillian.xml                            # Arquillian container config
│           ├── META-INF/
│           │   └── microprofile-config.properties        # config de teste
│           └── features/                                 # .feature (Cucumber)
```
---
## Organizacao dos Pacotes
O projeto e organizado em **4 pacotes raiz**, cada um com uma responsabilidade clara:
```
{group-id}
├── api/                       → Resources JAX-RS, DTOs (request/response), exception mappers
│   ├── rest/v1/               → REST API versionada (ou websocket/, grpc/)
│   └── messaging/jms/         → Consumers JMS (ou kafka/, rabbitmq/)
├── core/                      → Mappers (MapStruct) e grupos de validacao
├── domain/                    → Entities, repositories, services, excecoes
│   └── repository/mysql/      → Repositories com prefixo da tecnologia de banco
└── infrastructure/            → Configuracoes tecnicas (metricas, docs, clients, messaging)
```
> **Nota:** Essa organizacao e uma decisao pratica de pacotes — nao se vincula a nenhum padrao arquitetural especifico (Clean Architecture, Hexagonal, etc.).
### api — Resources, DTOs e Exception Mappers
Pacote responsavel por **receber requisicoes** e **devolver respostas**.
| Pacote                        | Responsabilidade                                             | Convencao                              |
|-------------------------------|--------------------------------------------------------------|----------------------------------------|
| `api.exception`               | Tratamento global de excecoes (`ExceptionMapper<T>`)         | Uma classe por tipo de excecao          |
| `api.rest.v1.resource`        | JAX-RS resources (`@Path`)                                   | Um resource por recurso                |
| `api.rest.v1.openapi`         | Interfaces com anotacoes OpenAPI (`@Tag`, `@Operation`)      | Uma interface por resource             |
| `api.rest.v1.request`         | DTOs de entrada (`record` com Bean Validation)               | Sufixo `Request`                       |
| `api.rest.v1.response`        | DTOs de saida (`record`)                                     | Sufixo `Response`                      |
| `api.rest.v1.commons.enums`   | Enums exclusivos da API                                      | Espelham domain enums quando necessario |
**Padroes adotados:**
- **Protocolo como pacote**: `api.rest.`, `api.websocket.`, `api.grpc.`
- **Versionamento por path**: `/v1/`, `/v2/`, etc.
- **Interface OpenAPI separada**: Mantem annotations fora do resource.
- **`ProblemDetail` (RFC 9457)**: Padrao de resposta de erro via `ExceptionMapper`.
### core — Mappers e Validacao
Codigo **reutilizavel** entre pacotes.
| Pacote              | Responsabilidade                              | Convencao                           |
|---------------------|-----------------------------------------------|-------------------------------------|
| `core.mapper`       | MapStruct mappers (request <-> entity <-> response) | Sufixo `Mapper`, interface + CDI bean |
| `core.validation`   | Grupos de validacao (Bean Validation groups)  | Interface marker (`Create`, `Update`) |
**Padrao de mapper:**
```java
@Mapper(componentModel = "cdi", nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface FooMapper {
    FooEntity toEntity(FooRequest request);
    FooResponse toResponse(FooEntity entity);
    void update(FooRequest request, @MappingTarget FooEntity entity);
}
```
> No Jakarta EE, use `componentModel = "cdi"` para que o mapper seja um bean CDI injetavel via `@Inject`.
### domain — Entities, Repositories e Services
Contem **entidades, repositorios, servicos e excecoes**.
| Pacote                      | Responsabilidade                              | Convencao                             |
|-----------------------------|-----------------------------------------------|---------------------------------------|
| `domain.entity`             | JPA entities (`@Entity`)                      | Sufixo `Entity`, tabela `tb_*`       |
| `domain.entity.enums`       | Enums persistidos (com `fromValue()` estatico)| Enum com `HashMap` lookup             |
| `domain.repository.mysql`   | Repositories para MySQL                       | Sufixo `Repository`, `@ApplicationScoped` |
| `domain.repository.postgresql` | Repositories para PostgreSQL (se aplicavel)| Mesmo padrao                          |
| `domain.repository.mongodb` | Repositories para MongoDB (se aplicavel)      | Mesmo padrao com Jakarta NoSQL        |
| `domain.service`            | Servicos (`@ApplicationScoped`)               | Sufixo `Service`, injecao via construtor |
| `domain.exception`          | Excecoes especificas                          | `ResourceNotFoundException`, `BusinessException` |
**Padroes adotados:**
- **Entities como classes JPA**: `@Entity` + `@Table(name = "tb_foo")`.
- **Repository com `EntityManager`**: Injecao via `@PersistenceContext` ou Jakarta Data.
- **Services como CDI beans**: `@ApplicationScoped` com `@Transactional`.
- **Excecoes unchecked**: Estendem `RuntimeException`.
**Padrao de Entity:**
```java
@Entity
@Table(name = "tb_foo")
public class FooEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    @Column(nullable = false, length = 100)
    private String name;
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private FooStatus status;
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    @PrePersist
    void prePersist() {
        this.createdAt = LocalDateTime.now();
    }
    @PreUpdate
    void preUpdate() {
        this.updatedAt = LocalDateTime.now();
    }
    // getters e setters
}
```
**Padrao de Repository (classico com EntityManager):**
```java
@ApplicationScoped
public class FooRepository {
    @PersistenceContext
    private EntityManager em;
    public FooEntity save(FooEntity entity) {
        em.persist(entity);
        return entity;
    }
    public Optional<FooEntity> findById(UUID id) {
        return Optional.ofNullable(em.find(FooEntity.class, id));
    }
    public List<FooEntity> findAll(int page, int size) {
        return em.createQuery("SELECT f FROM FooEntity f ORDER BY f.createdAt DESC", FooEntity.class)
                .setFirstResult(page * size)
                .setMaxResults(size)
                .getResultList();
    }
    public long count() {
        return em.createQuery("SELECT COUNT(f) FROM FooEntity f", Long.class)
                .getSingleResult();
    }
    public void delete(FooEntity entity) {
        em.remove(em.contains(entity) ? entity : em.merge(entity));
    }
}
```
**Padrao de Repository (Jakarta Data 1.0 — Jakarta EE 11):**
```java
@Repository
public interface FooRepository extends CrudRepository<FooEntity, UUID> {
    @Find
    Optional<FooEntity> findByName(String name);
    @Query("SELECT f FROM FooEntity f WHERE f.status = :status")
    List<FooEntity> findByStatus(@Param("status") FooStatus status);
    @Find
    Page<FooEntity> findAll(PageRequest pageRequest);
}
```
> Jakarta Data 1.0 e a nova especificacao do Jakarta EE 11 que simplifica o acesso a dados com interfaces declarativas, similar ao Spring Data.
**Padrao de Service:**
```java
@ApplicationScoped
@Transactional
public class FooService {
    private final FooRepository fooRepository;
    private final FooMapper fooMapper;
    @Inject
    public FooService(FooRepository fooRepository, FooMapper fooMapper) {
        this.fooRepository = fooRepository;
        this.fooMapper = fooMapper;
    }
    public FooResponse create(FooRequest request) {
        FooEntity entity = fooMapper.toEntity(request);
        FooEntity saved = fooRepository.save(entity);
        return fooMapper.toResponse(saved);
    }
    public FooResponse findById(UUID id) {
        FooEntity entity = fooRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Foo", id));
        return fooMapper.toResponse(entity);
    }
    public void delete(UUID id) {
        FooEntity entity = fooRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Foo", id));
        fooRepository.delete(entity);
    }
}
```
### infrastructure — Configuracoes Tecnicas
Pacote para **configuracao de infraestrutura** que nao pertence ao dominio.
| Pacote                      | Responsabilidade                              | Convencao                           |
|-----------------------------|-----------------------------------------------|-------------------------------------|
| `infrastructure.config`     | Producers de configuracao (`@ConfigProperty`)  | Classe `@ApplicationScoped`         |
| `infrastructure.metrics`    | Metricas customizadas                         | `@Counted`, `@Timed`, `@Gauge`     |
| `infrastructure.security`   | Configuracao Jakarta Security                 | `@FormAuthenticationMechanismDefinition` ou JWT |
| `infrastructure.cors`       | Filtro CORS                                   | `@Provider` + `ContainerResponseFilter` |
| `infrastructure.json`       | Configuracao JSON-B                           | `ContextResolver<Jsonb>`           |
| `infrastructure.messaging`  | Configuracao JMS / Kafka                      | `@JMSConnectionFactory`, producers CDI |
---
## Pontos de Extensao
### API — REST / WebSocket / gRPC
```
api/
├── rest/v1/           → JAX-RS @Path resources (padrao)
├── websocket/         → @ServerEndpoint WebSocket endpoints
└── grpc/              → gRPC services (via extensao do runtime)
```
A extensao e feita **criando um novo pacote** dentro de `api/`:
```java
// api/websocket/FooWebSocket.java
@ServerEndpoint("/ws/foo")
@ApplicationScoped
public class FooWebSocket {
    @OnOpen
    public void onOpen(Session session) { /* ... */ }
    @OnMessage
    public void onMessage(String message, Session session) { /* ... */ }
    @OnClose
    public void onClose(Session session) { /* ... */ }
}
```
### Mensageria — JMS / Kafka / RabbitMQ
```
api/messaging/
├── jms/               → @MessageDriven MDBs (padrao Jakarta EE)
├── kafka/             → Consumers/Producers Kafka (via MicroProfile Reactive Messaging)
└── rabbitmq/          → Consumers/Producers RabbitMQ
```
**Padrao JMS (Message-Driven Bean):**
```java
@MessageDriven(activationConfig = {
    @ActivationConfigProperty(propertyName = "destinationType", propertyValue = "jakarta.jms.Queue"),
    @ActivationConfigProperty(propertyName = "destination", propertyValue = "java:/jms/queue/FooQueue"),
    @ActivationConfigProperty(propertyName = "acknowledgeMode", propertyValue = "Auto-acknowledge")
})
public class FooMessageConsumer implements MessageListener {
    @Inject
    private FooService fooService;
    @Override
    public void onMessage(Message message) {
        try {
            TextMessage textMessage = (TextMessage) message;
            FooRequest request = JsonbBuilder.create().fromJson(textMessage.getText(), FooRequest.class);
            fooService.create(request);
        } catch (JMSException e) {
            throw new RuntimeException("Erro ao processar mensagem JMS", e);
        }
    }
}
```
**Padrao Kafka (MicroProfile Reactive Messaging):**
```java
@ApplicationScoped
public class FooKafkaConsumer {
    @Inject
    private FooService fooService;
    @Incoming("foo-events")
    public CompletionStage<Void> consume(Message<String> message) {
        FooRequest request = JsonbBuilder.create().fromJson(message.getPayload(), FooRequest.class);
        fooService.create(request);
        return message.ack();
    }
}
```
### Banco de Dados — MySQL / PostgreSQL / MongoDB
```
domain/repository/
├── mysql/             → EntityManager + JPA (MySQL)
├── postgresql/        → EntityManager + JPA (PostgreSQL)
└── mongodb/           → Jakarta NoSQL / Panache MongoDB
```
A troca de banco e feita:
1. Alterando o `persistence.xml` (driver, dialect, URL)
2. Movendo repositories para o pacote correspondente
3. Ajustando o datasource no runtime (standalone.xml / server.xml)
---
## Convencoes de Codigo
### Nomeacao
| Tipo             | Convencao                      | Exemplo                          |
|------------------|--------------------------------|----------------------------------|
| Pacote           | `lowercase` singular           | `entity`, `service`, `resource`  |
| Classe           | `PascalCase` + sufixo          | `FooResource`, `FooService`      |
| Interface        | `PascalCase` + sufixo          | `FooMapper`, `FooOpenApi`        |
| Record (DTO)     | `PascalCase` + `Request`/`Response` | `FooRequest`, `FooResponse` |
| Entity           | `PascalCase` + `Entity`        | `FooEntity`                      |
| Tabela (DB)      | `snake_case` + prefixo `tb_`   | `tb_foo`, `tb_foo_status`        |
| Enum             | `PascalCase`                   | `FooStatus`                      |
| Coluna (DB)      | `snake_case`                   | `created_at`, `full_name`        |
| REST path        | `kebab-case` plural            | `/v1/foos`, `/v1/foo-items`      |
| Config property  | `dot.case` lowercase           | `app.foo.max-retries`            |
| Constante        | `UPPER_SNAKE_CASE`             | `MAX_PAGE_SIZE`                  |
### DTOs
- **Records imutaveis**: Usar `record` Java 17+ para DTOs.
- **Bean Validation**: Anotacoes diretamente nos componentes do record.
- **Sem logica de negocio**: Apenas dados e validacao.
```java
public record FooRequest(
    @NotBlank(message = "Nome e obrigatorio")
    @Size(min = 2, max = 100, message = "Nome deve ter entre 2 e 100 caracteres")
    String name,
    @NotNull(message = "Status e obrigatorio")
    FooStatus status
) {}
public record FooResponse(
    UUID id,
    String name,
    FooStatus status,
    LocalDateTime createdAt
) {}
```
### Resources (JAX-RS)
- **Um resource por recurso**: `FooResource` para `/v1/foos`.
- **Interface OpenAPI separada**: Anotacoes `@Tag`, `@Operation` em interface dedicada.
- **Injecao via CDI**: `@Inject` no construtor.
- **Retorno `Response`**: Usar `Response.ok()`, `Response.created()`, etc.
```java
@Path("/v1/foos")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@ApplicationScoped
public class FooResource implements FooOpenApi {
    private final FooService fooService;
    @Inject
    public FooResource(FooService fooService) {
        this.fooService = fooService;
    }
    @POST
    @Override
    public Response create(@Valid FooRequest request) {
        FooResponse response = fooService.create(request);
        return Response.status(Response.Status.CREATED).entity(response).build();
    }
    @GET
    @Path("/{id}")
    @Override
    public Response findById(@PathParam("id") UUID id) {
        FooResponse response = fooService.findById(id);
        return Response.ok(response).build();
    }
    @GET
    @Override
    public Response findAll(
            @QueryParam("page") @DefaultValue("0") int page,
            @QueryParam("size") @DefaultValue("20") int size) {
        PageResponse<FooResponse> response = fooService.findAll(page, size);
        return Response.ok(response).build();
    }
    @PUT
    @Path("/{id}")
    @Override
    public Response update(@PathParam("id") UUID id, @Valid FooRequest request) {
        FooResponse response = fooService.update(id, request);
        return Response.ok(response).build();
    }
    @DELETE
    @Path("/{id}")
    @Override
    public Response delete(@PathParam("id") UUID id) {
        fooService.delete(id);
        return Response.noContent().build();
    }
}
```
**Interface OpenAPI:**
```java
@Tag(name = "Foos", description = "Operacoes de Foo")
public interface FooOpenApi {
    @Operation(summary = "Criar Foo", description = "Cria um novo Foo")
    @APIResponse(responseCode = "201", description = "Foo criado com sucesso")
    Response create(FooRequest request);
    @Operation(summary = "Buscar Foo por ID")
    @APIResponse(responseCode = "200", description = "Foo encontrado")
    @APIResponse(responseCode = "404", description = "Foo nao encontrado")
    Response findById(UUID id);
    @Operation(summary = "Listar Foos", description = "Lista paginada de Foos")
    @APIResponse(responseCode = "200", description = "Lista de Foos")
    Response findAll(int page, int size);
    @Operation(summary = "Atualizar Foo")
    @APIResponse(responseCode = "200", description = "Foo atualizado")
    Response update(UUID id, FooRequest request);
    @Operation(summary = "Deletar Foo")
    @APIResponse(responseCode = "204", description = "Foo deletado")
    Response delete(UUID id);
}
```
### Entities
- **`@Entity` + `@Table`**: Nome da tabela com prefixo `tb_`.
- **`@Id` + `@GeneratedValue`**: UUID como identificador.
- **Callbacks JPA**: `@PrePersist`, `@PreUpdate` para timestamps.
- **Enums como `@Enumerated(EnumType.STRING)`**.
---
## Configuracao por Profile
Jakarta EE usa o **MicroProfile Config** para gerenciar configuracao por ambiente:
```
src/main/resources/META-INF/
├── microprofile-config.properties           # config padrao (producao)
```
Para profiles por ambiente, usar config sources com ordinal:
```properties
# microprofile-config.properties (producao — ordinal padrao 100)
app.datasource.url=jdbc:mysql://db-prod:3306/appdb
app.datasource.username=app_user
app.cors.allowed-origins=https://app.example.com
app.feature.new-dashboard=false
```
### Particularidades do Profile `dev`
Para desenvolvimento local, usar variavel de ambiente ou arquivo de override:
```properties
# ENV vars (ordinal 300 — maior prioridade)
APP_DATASOURCE_URL=jdbc:mysql://localhost:3306/appdb
APP_DATASOURCE_USERNAME=root
APP_CORS_ALLOWED_ORIGINS=http://localhost:3000
APP_FEATURE_NEW_DASHBOARD=true
```
Em WildFly, a configuracao de datasource e feita no `standalone.xml` ou via CLI:
```xml
<datasource jndi-name="java:jboss/datasources/AppDS" pool-name="AppDS">
    <connection-url>jdbc:mysql://localhost:3306/appdb</connection-url>
    <driver>mysql</driver>
    <security user-name="root" password="root"/>
</datasource>
```
### MicroProfile Config
Uso no codigo via CDI:
```java
@ApplicationScoped
public class AppConfig {
    @Inject
    @ConfigProperty(name = "app.datasource.url")
    private String datasourceUrl;
    @Inject
    @ConfigProperty(name = "app.feature.new-dashboard", defaultValue = "false")
    private boolean newDashboardEnabled;
    @Inject
    @ConfigProperty(name = "app.cors.allowed-origins", defaultValue = "http://localhost:3000")
    private String allowedOrigins;
    // getters
}
```
---
## Banco de Dados e Migrations
### Convencao de Migrations
Flyway e a ferramenta de migrations recomendada. A convencao de nomenclatura:
```
src/main/resources/db/migration/
├── V1.0__create_table_tb_foo.sql
├── V2.0__add_column_status_to_tb_foo.sql
├── V3.0__create_table_tb_bar.sql
└── V4.0__create_index_tb_foo_status.sql
```
| Tipo | Prefixo | Separador | Descricao |
|------|---------|-----------|-----------|
| DDL  | `V{n}.0__` | `__` (duplo underscore) | `create_table_tb_foo`, `add_column_*`, `create_index_*` |
| DML  | `V{n}.1__` | `__` | `seed_tb_foo` (somente dev) |
**Padrao de migration DDL:**
```sql
-- V1.0__create_table_tb_foo.sql
CREATE TABLE tb_foo (
    id         BINARY(16)   NOT NULL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    status     VARCHAR(20)  NOT NULL,
    created_at DATETIME(6)  NOT NULL,
    updated_at DATETIME(6),
    CONSTRAINT uk_foo_name UNIQUE (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
CREATE INDEX idx_foo_status ON tb_foo (status);
```
### Configuracao do DataSource
No `persistence.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="3.2" xmlns="https://jakarta.ee/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="https://jakarta.ee/xml/ns/persistence
             https://jakarta.ee/xml/ns/persistence/persistence_3_2.xsd">
    <persistence-unit name="AppPU" transaction-type="JTA">
        <jta-data-source>java:jboss/datasources/AppDS</jta-data-source>
        <properties>
            <property name="jakarta.persistence.schema-generation.database.action" value="none"/>
            <property name="hibernate.dialect" value="org.hibernate.dialect.MySQLDialect"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.show_sql" value="false"/>
        </properties>
    </persistence-unit>
</persistence>
```
> **Nota:** `schema-generation.database.action=none` — o Flyway gerencia o schema. Nunca usar `create`, `drop-and-create` em producao.
---
## Testes
### Estrutura
```
src/test/
├── java/{group-id}/
│   ├── api/rest/v1/resource/                 # Integration tests (Arquillian + REST Assured)
│   ├── domain/
│   │   ├── entity/                           # TemplateLoader (fixtures)
│   │   ├── request/                          # TemplateLoader (fixtures)
│   │   ├── response/                         # TemplateLoader (fixtures)
│   │   └── service/                          # Unit tests + @ParameterizedTest
│   ├── cucumber/                             # BDD integration
│   │   ├── config/
│   │   ├── step/
│   │   └── utils/
│   └── infrastructure/                       # Testcontainers base config
└── resources/
    ├── arquillian.xml                        # Arquillian container config
    ├── META-INF/
    │   └── microprofile-config.properties    # Config de teste
    └── features/                             # .feature (Cucumber)
```
### Convencoes de Teste
| Tipo | Localizacao | Framework | Sufixo |
|------|-------------|-----------|--------|
| Unit (service) | `domain/service/` | JUnit 5 + Mockito | `*Test.java` |
| Unit (mapper) | `core/mapper/` | JUnit 5 | `*Test.java` |
| Integration (API) | `api/rest/v1/resource/` | Arquillian + REST Assured | `*IT.java` |
| Integration (DB) | `domain/repository/` | Testcontainers | `*IT.java` |
| BDD | `cucumber/` | Cucumber + JUnit 5 | `*Steps.java` |
| Carga | `k6/` (raiz) | k6 | `*.js` |
### Organizacao de Testes
**Unit Test (Service):**
```java
@ExtendWith(MockitoExtension.class)
class FooServiceTest {
    @InjectMocks
    private FooService fooService;
    @Mock
    private FooRepository fooRepository;
    @Mock
    private FooMapper fooMapper;
    @Test
    @DisplayName("Deve criar Foo com sucesso")
    void shouldCreateFoo() {
        // given
        var request = new FooRequest("Bar", FooStatus.ACTIVE);
        var entity = new FooEntity();
        entity.setName("Bar");
        var saved = new FooEntity();
        saved.setId(UUID.randomUUID());
        saved.setName("Bar");
        var expected = new FooResponse(saved.getId(), "Bar", FooStatus.ACTIVE, LocalDateTime.now());
        when(fooMapper.toEntity(request)).thenReturn(entity);
        when(fooRepository.save(entity)).thenReturn(saved);
        when(fooMapper.toResponse(saved)).thenReturn(expected);
        // when
        FooResponse result = fooService.create(request);
        // then
        assertThat(result).isNotNull();
        assertThat(result.name()).isEqualTo("Bar");
        verify(fooRepository).save(entity);
    }
    @Test
    @DisplayName("Deve lancar excecao quando Foo nao encontrado")
    void shouldThrowWhenFooNotFound() {
        UUID id = UUID.randomUUID();
        when(fooRepository.findById(id)).thenReturn(Optional.empty());
        assertThatThrownBy(() -> fooService.findById(id))
                .isInstanceOf(ResourceNotFoundException.class)
                .hasMessageContaining("Foo");
    }
}
```
**Integration Test (Arquillian + REST Assured):**
```java
@ExtendWith(ArquillianExtension.class)
class FooResourceIT {
    @ArquillianResource
    private URL baseURL;
    @Deployment
    public static WebArchive createDeployment() {
        return ShrinkWrap.create(WebArchive.class, "test.war")
                .addPackages(true, "com.example")
                .addAsResource("META-INF/persistence.xml")
                .addAsResource("META-INF/beans.xml")
                .addAsWebInfResource(EmptyAsset.INSTANCE, "beans.xml");
    }
    @Test
    @DisplayName("POST /v1/foos deve retornar 201")
    void shouldCreateFoo() {
        given()
            .baseUri(baseURL.toString())
            .contentType(ContentType.JSON)
            .body("""
                {"name": "Bar", "status": "ACTIVE"}
                """)
        .when()
            .post("/v1/foos")
        .then()
            .statusCode(201)
            .body("name", equalTo("Bar"))
            .body("id", notNullValue());
    }
}
```
### Comandos de Teste
```bash
# Unitarios
mvn test
# Integracao (Arquillian + Testcontainers)
mvn verify -Parquillian-managed
# Tudo
mvn verify
# Com cobertura
mvn verify -Pcoverage
# Mutation testing
mvn test -Ppitest
```
---
## Build e Qualidade
```bash
# Build (WAR)
mvn clean package
# Build sem testes
mvn clean package -DskipTests
# Deploy no WildFly (via plugin)
mvn wildfly:deploy
# Undeploy
mvn wildfly:undeploy
# Qualidade
mvn verify                    # testes + integracao
mvn jacoco:report             # cobertura
mvn pitest:mutationCoverage   # mutation testing
```
**Plugins Maven recomendados:**
```xml
<build>
    <plugins>
        <!-- WildFly Maven Plugin -->
        <plugin>
            <groupId>org.wildfly.plugins</groupId>
            <artifactId>wildfly-maven-plugin</artifactId>
            <version>5.1.1.Final</version>
            <configuration>
                <hostname>localhost</hostname>
                <port>9990</port>
            </configuration>
        </plugin>
        <!-- JaCoCo -->
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.12</version>
            <executions>
                <execution>
                    <goals><goal>prepare-agent</goal></goals>
                </execution>
                <execution>
                    <id>report</id>
                    <phase>verify</phase>
                    <goals><goal>report</goal></goals>
                </execution>
            </executions>
        </plugin>
        <!-- Compiler -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.13.0</version>
            <configuration>
                <release>21</release>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>1.6.3</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```
**Dependencia principal (`pom.xml`):**
```xml
<dependencies>
    <!-- Jakarta EE 11 API (provided pelo runtime) -->
    <dependency>
        <groupId>jakarta.platform</groupId>
        <artifactId>jakarta.jakartaee-api</artifactId>
        <version>11.0.0</version>
        <scope>provided</scope>
    </dependency>
    <!-- MicroProfile (provided pelo runtime) -->
    <dependency>
        <groupId>org.eclipse.microprofile</groupId>
        <artifactId>microprofile</artifactId>
        <version>7.0</version>
        <type>pom</type>
        <scope>provided</scope>
    </dependency>
    <!-- MapStruct -->
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>1.6.3</version>
    </dependency>
    <!-- Flyway -->
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
        <version>10.22.0</version>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-mysql</artifactId>
        <version>10.22.0</version>
    </dependency>
    <!-- Test -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.11.4</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>5.15.2</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <version>5.5.0</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.27.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```
---
## Resiliencia
### Dependencia (MicroProfile Fault Tolerance)
MicroProfile Fault Tolerance ja esta incluido no runtime Jakarta EE compativel com MicroProfile. Basta anotar os metodos:
```xml
<!-- Ja incluido via microprofile BOM (scope provided) -->
<dependency>
    <groupId>org.eclipse.microprofile.fault-tolerance</groupId>
    <artifactId>microprofile-fault-tolerance-api</artifactId>
    <scope>provided</scope>
</dependency>
```
### Padroes Disponiveis
| Padrao           | Anotacao           | Descricao                              |
|------------------|--------------------|----------------------------------------|
| Circuit Breaker  | `@CircuitBreaker`  | Abre circuito apos falhas consecutivas |
| Retry            | `@Retry`           | Reexecuta em caso de falha             |
| Timeout          | `@Timeout`         | Limita tempo de execucao               |
| Bulkhead         | `@Bulkhead`        | Limita execucoes concorrentes          |
| Fallback         | `@Fallback`        | Metodo alternativo em caso de falha    |
### Organizacao de Pacotes
```
infrastructure/resilience/ → nao necessario (anotacoes diretamente nos services)
```
> Diferente do Resilience4j (que precisa de configuracao programatica), o MicroProfile Fault Tolerance usa anotacoes declarativas diretamente nos metodos dos services.
### Configuracao
Via `microprofile-config.properties`:
```properties
# Circuit Breaker global
mp.fault.tolerance.circuitbreaker.requestVolumeThreshold=20
mp.fault.tolerance.circuitbreaker.failureRatio=0.5
mp.fault.tolerance.circuitbreaker.delay=5000
# Retry global
mp.fault.tolerance.retry.maxRetries=3
mp.fault.tolerance.retry.delay=1000
# Timeout global
mp.fault.tolerance.timeout.value=5000
# Override por metodo
com.example.domain.service.FooService/callExternalApi/Timeout/value=10000
com.example.domain.service.FooService/callExternalApi/Retry/maxRetries=5
```
### Padrao de uso no Service
```java
@ApplicationScoped
public class ExternalApiService {
    @Inject
    @RestClient
    private ExternalApiClient client;
    @Retry(maxRetries = 3, delay = 1000, retryOn = {ConnectException.class, TimeoutException.class})
    @Timeout(5000)
    @CircuitBreaker(requestVolumeThreshold = 20, failureRatio = 0.5, delay = 5000)
    @Fallback(fallbackMethod = "fallbackGetData")
    public ExternalData getData(String id) {
        return client.getData(id);
    }
    private ExternalData fallbackGetData(String id) {
        return new ExternalData(id, "Dados indisponiveis", LocalDateTime.now());
    }
}
```
### Metricas de Resiliencia
MicroProfile Fault Tolerance expoe metricas automaticamente via MicroProfile Metrics:
```
# Circuit Breaker
application_ft_com_example_domain_service_ExternalApiService_getData_circuitbreaker_open_total
application_ft_com_example_domain_service_ExternalApiService_getData_circuitbreaker_callsSucceeded_total
application_ft_com_example_domain_service_ExternalApiService_getData_circuitbreaker_callsFailed_total
# Retry
application_ft_com_example_domain_service_ExternalApiService_getData_retry_retries_total
application_ft_com_example_domain_service_ExternalApiService_getData_retry_callsSucceededRetried_total
# Timeout
application_ft_com_example_domain_service_ExternalApiService_getData_timeout_callsTimedOut_total
```
---
## Seguranca (AuthN/AuthZ)
### Dependencias
Jakarta Security e MicroProfile JWT ja estao incluidos no runtime:
```xml
<!-- Ja incluido via jakarta.jakartaee-api e microprofile BOM (scope provided) -->
```
### Organizacao de Pacotes
```
infrastructure/security/
├── SecurityConfig.java              # @LoginConfig / mecanismo de auth
├── IdentityStoreConfig.java         # (opcional) identity store customizado
└── RoleMapping.java                 # (opcional) mapeamento de roles
```
### Configuracao
Via `microprofile-config.properties`:
```properties
# MicroProfile JWT
mp.jwt.verify.publickey.location=META-INF/publicKey.pem
mp.jwt.verify.issuer=https://auth.example.com
mp.jwt.verify.audiences=my-app
# Token expiration
mp.jwt.token.header=Authorization
mp.jwt.token.cookie=jwt-token
```
Ou via Jakarta Security annotation:
```java
@ApplicationScoped
@LoginConfig(authMethod = "MP-JWT", realmName = "my-app")
public class SecurityConfig {
}
```
### RBAC (Role-Based Access Control)
```java
@Path("/v1/admin/foos")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@ApplicationScoped
@RolesAllowed("ADMIN")
public class AdminFooResource {
    @GET
    @RolesAllowed({"ADMIN", "MANAGER"})
    public Response findAll() { /* ... */ }
    @DELETE
    @Path("/{id}")
    @RolesAllowed("ADMIN")
    public Response delete(@PathParam("id") UUID id) { /* ... */ }
}
```
**Acessando o Principal (usuario autenticado):**
```java
@ApplicationScoped
public class AuditService {
    @Inject
    private JsonWebToken jwt;
    @Inject
    @Claim(standard = Claims.sub)
    private String subject;
    @Inject
    @Claim(standard = Claims.groups)
    private Set<String> roles;
    public String getCurrentUserId() {
        return jwt.getSubject();
    }
    public boolean hasRole(String role) {
        return roles.contains(role);
    }
}
```
---
## Tratamento Global de Erros
### Padrao: ProblemDetail (RFC 9457) via ExceptionMapper
Jakarta EE usa `ExceptionMapper<T>` para mapear excecoes em respostas HTTP padronizadas.
### Organizacao
```
api/exception/
├── ApiExceptionMapper.java          # ExceptionMapper generico
├── ResourceNotFoundExceptionMapper.java  # 404
├── BusinessExceptionMapper.java     # 422
├── ConstraintViolationExceptionMapper.java  # 400
└── model/
    └── ProblemDetail.java           # Record RFC 9457
```
### Padrao de ExceptionMapper
```java
@Provider
public class ResourceNotFoundExceptionMapper implements ExceptionMapper<ResourceNotFoundException> {
    @Override
    public Response toResponse(ResourceNotFoundException e) {
        ProblemDetail problem = new ProblemDetail(
            URI.create("https://api.example.com/errors/not-found"),
            "Resource Not Found",
            Response.Status.NOT_FOUND.getStatusCode(),
            e.getMessage(),
            URI.create("/v1/" + e.getResource() + "/" + e.getId())
        );
        return Response.status(Response.Status.NOT_FOUND).entity(problem).build();
    }
}
@Provider
public class ConstraintViolationExceptionMapper implements ExceptionMapper<ConstraintViolationException> {
    @Override
    public Response toResponse(ConstraintViolationException e) {
        List<String> violations = e.getConstraintViolations().stream()
                .map(v -> v.getPropertyPath() + ": " + v.getMessage())
                .sorted()
                .toList();
        ProblemDetail problem = new ProblemDetail(
            URI.create("https://api.example.com/errors/validation"),
            "Validation Error",
            Response.Status.BAD_REQUEST.getStatusCode(),
            "Erro de validacao: " + String.join(", ", violations),
            null
        );
        return Response.status(Response.Status.BAD_REQUEST).entity(problem).build();
    }
}
```
### Formato de Resposta (JSON)
```json
{
  "type": "https://api.example.com/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "Foo com ID 123e4567-e89b-12d3-a456-426614174000 nao encontrado",
  "instance": "/v1/foos/123e4567-e89b-12d3-a456-426614174000"
}
```
### Mapeamento de Excecoes
| Excecao                        | HTTP Status | Tipo                           |
|--------------------------------|-------------|--------------------------------|
| `ResourceNotFoundException`    | 404         | `errors/not-found`             |
| `BusinessException`           | 422         | `errors/business-rule`         |
| `ConstraintViolationException` | 400         | `errors/validation`            |
| `NotAuthorizedException`       | 401         | `errors/unauthorized`          |
| `ForbiddenException`          | 403         | `errors/forbidden`             |
| `Exception` (generico)        | 500         | `errors/internal`              |
---
## Versionamento de API
### Estrategia: Versionamento por Path
```
/v1/foos       → versao atual
/v2/foos       → proxima versao (quando houver breaking changes)
```
### Organizacao de Pacotes
```
api/rest/
├── v1/
│   ├── resource/FooResource.java       # @Path("/v1/foos")
│   ├── request/FooRequest.java
│   └── response/FooResponse.java
└── v2/
    ├── resource/FooResource.java       # @Path("/v2/foos")
    ├── request/FooRequestV2.java
    └── response/FooResponseV2.java
```
### Regras
1. **Nunca quebrar contratos** — adicionar campos e OK, remover/renomear nao.
2. **Nova versao somente para breaking changes** — mudanca de tipo, remocao de campo, mudanca de semantica.
3. **Manter v1 funcionando** — deprecar com `@Deprecated` e header `Sunset`.
4. **Maximo 2 versoes ativas** simultaneamente.
---
## Paginacao e Ordenacao
### Convencao de Query Parameters
| Parametro | Tipo | Default | Descricao        |
|-----------|------|---------|------------------|
| `page`    | int  | 0       | Numero da pagina (0-indexed) |
| `size`    | int  | 20      | Itens por pagina |
| `sort`    | string | `createdAt,desc` | Campo e direcao |
### Formato de Resposta Paginada
```json
{
  "content": [ { "id": "...", "name": "..." } ],
  "page": 0,
  "size": 20,
  "totalElements": 150,
  "totalPages": 8,
  "first": true,
  "last": false
}
```
### Padrao no Resource
```java
@GET
public Response findAll(
        @QueryParam("page") @DefaultValue("0") int page,
        @QueryParam("size") @DefaultValue("20") int size,
        @QueryParam("sort") @DefaultValue("createdAt,desc") String sort) {
    PageResponse<FooResponse> response = fooService.findAll(page, size, sort);
    return Response.ok(response).build();
}
```
### Padrao no Service
```java
public PageResponse<FooResponse> findAll(int page, int size, String sort) {
    // Parse sort
    String[] sortParts = sort.split(",");
    String sortField = sortParts[0];
    String sortDirection = sortParts.length > 1 ? sortParts[1] : "desc";
    String orderClause = "ORDER BY f." + sortField + " " + sortDirection.toUpperCase();
    List<FooEntity> content = em.createQuery(
            "SELECT f FROM FooEntity f " + orderClause, FooEntity.class)
            .setFirstResult(page * size)
            .setMaxResults(size)
            .getResultList();
    long totalElements = em.createQuery("SELECT COUNT(f) FROM FooEntity f", Long.class)
            .getSingleResult();
    int totalPages = (int) Math.ceil((double) totalElements / size);
    List<FooResponse> contentResponse = content.stream()
            .map(fooMapper::toResponse)
            .toList();
    return new PageResponse<>(contentResponse, page, size, totalElements, totalPages,
            page == 0, page >= totalPages - 1);
}
```
**Record de Resposta Paginada:**
```java
public record PageResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean first,
    boolean last
) {}
```
### Limites
| Regra | Valor |
|-------|-------|
| `size` maximo | 100 |
| `size` padrao | 20 |
| `page` minimo | 0 |
| Campos ordenacao permitidos | Definidos por resource |
---
## CORS
### Configuracao via Filter
Em Jakarta EE, CORS e configurado via `ContainerResponseFilter`:
```java
@Provider
@ApplicationScoped
public class CorsFilter implements ContainerResponseFilter {
    @Inject
    @ConfigProperty(name = "app.cors.allowed-origins", defaultValue = "http://localhost:3000")
    private String allowedOrigins;
    @Inject
    @ConfigProperty(name = "app.cors.allowed-methods", defaultValue = "GET,POST,PUT,DELETE,PATCH,OPTIONS")
    private String allowedMethods;
    @Inject
    @ConfigProperty(name = "app.cors.allowed-headers", defaultValue = "Content-Type,Authorization,X-Requested-With")
    private String allowedHeaders;
    @Inject
    @ConfigProperty(name = "app.cors.max-age", defaultValue = "3600")
    private int maxAge;
    @Override
    public void filter(ContainerRequestContext requestContext, ContainerResponseContext responseContext) {
        responseContext.getHeaders().add("Access-Control-Allow-Origin", allowedOrigins);
        responseContext.getHeaders().add("Access-Control-Allow-Methods", allowedMethods);
        responseContext.getHeaders().add("Access-Control-Allow-Headers", allowedHeaders);
        responseContext.getHeaders().add("Access-Control-Max-Age", String.valueOf(maxAge));
        responseContext.getHeaders().add("Access-Control-Allow-Credentials", "true");
    }
}
```
### Configuracao por Profile
```properties
# producao
app.cors.allowed-origins=https://app.example.com
# dev (via env var)
APP_CORS_ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
```
---
## Feature Flags
### Biblioteca: Togglz (CDI Integration)
```xml
<dependency>
    <groupId>org.togglz</groupId>
    <artifactId>togglz-cdi</artifactId>
    <version>4.4.0</version>
</dependency>
<dependency>
    <groupId>org.togglz</groupId>
    <artifactId>togglz-console</artifactId>
    <version>4.4.0</version>
</dependency>
```
### Definicao das Features
```java
public enum AppFeatures implements Feature {
    @Label("Novo Dashboard")
    @EnabledByDefault
    NEW_DASHBOARD,
    @Label("Exportacao CSV")
    CSV_EXPORT,
    @Label("Notificacoes Push")
    PUSH_NOTIFICATIONS;
    public boolean isActive() {
        return FeatureContext.getFeatureManager().isActive(this);
    }
}
```
**Producer CDI para o FeatureManager:**
```java
@ApplicationScoped
public class FeatureManagerProducer {
    @Produces
    @ApplicationScoped
    public FeatureManager produceFeatureManager() {
        return new FeatureManagerBuilder()
                .featureEnum(AppFeatures.class)
                .stateRepository(new InMemoryStateRepository())
                .userProvider(new NoOpUserProvider())
                .build();
    }
}
```
### Uso no Service
```java
@ApplicationScoped
public class DashboardService {
    public DashboardResponse getDashboard() {
        if (AppFeatures.NEW_DASHBOARD.isActive()) {
            return buildNewDashboard();
        }
        return buildLegacyDashboard();
    }
}
```
---
## Virtual Threads
### Configuracao (Jakarta EE 11 + JDK 21+)
Jakarta Concurrency 3.1 suporta Virtual Threads nativamente. A configuracao depende do runtime:
**WildFly (standalone.xml):**
```xml
<subsystem xmlns="urn:jboss:domain:ee:6.0">
    <concurrent>
        <managed-executor-services>
            <managed-executor-service name="default"
                                     jndi-name="java:comp/DefaultManagedExecutorService"
                                     virtual="true"/>
        </managed-executor-services>
    </concurrent>
</subsystem>
```
**Open Liberty (server.xml):**
```xml
<managedExecutorService jndiName="java:comp/DefaultManagedExecutorService"
                        virtual="true"/>
```
**Uso programatico:**
```java
@ApplicationScoped
public class AsyncService {
    @Resource
    private ManagedExecutorService executorService;
    public CompletableFuture<String> processAsync(String data) {
        return CompletableFuture.supplyAsync(() -> {
            // executa em virtual thread
            return heavyComputation(data);
        }, executorService);
    }
}
```
**Uso com `@Asynchronous` (MicroProfile):**
```java
@ApplicationScoped
public class NotificationService {
    @Asynchronous
    public CompletionStage<Void> sendNotification(String userId, String message) {
        // Executa em virtual thread quando configurado
        notificationClient.send(userId, message);
        return CompletableFuture.completedFuture(null);
    }
}
```
### Impacto
| Aspecto | Antes (Platform Threads) | Com Virtual Threads |
|---------|--------------------------|---------------------|
| Throughput REST | ~200 req/s por core | ~1000+ req/s por core |
| Memoria por thread | ~1 MB | ~poucos KB |
| Blocking I/O | Bloqueia thread do pool | Libera carrier thread |
| Pool size | Fixo (ex: 200) | Ilimitado (virtual) |
### Cuidados
1. **Evitar `synchronized`** — preferir `ReentrantLock` para nao "pin" virtual threads.
2. **Evitar thread-local extenso** — virtual threads multiplicam o numero de threads.
3. **Nao usar para CPU-bound** — virtual threads sao otimizados para I/O.
4. **ThreadLocal com cuidado** — scoped values (JDK 25) sao preferidos.
### Verificacao
```bash
# Verificar se virtual threads estao ativas (log no startup do runtime)
# WildFly
grep "virtual" standalone/log/server.log
# Verificar em runtime
curl http://localhost:8080/health | jq '.checks[] | select(.name == "thread-info")'
```
---
## Observabilidade
A observabilidade e composta por **3 pilares** + health checks:
| Pilar | Tecnologia | Configuracao |
|-------|-----------|--------------|
| **Metricas** | MicroProfile Metrics / Micrometer | `@Counted`, `@Timed`, `@Gauge` |
| **Tracing** | MicroProfile Telemetry (OpenTelemetry) | `microprofile-config.properties` |
| **Logging** | JBoss Logging + JSON formatter | `logging.properties` / runtime config |
| **Health** | MicroProfile Health | `@Liveness`, `@Readiness`, `@Startup` |
**Metricas customizadas:**
```java
@ApplicationScoped
public class FooService {
    @Inject
    private MetricRegistry metricRegistry;
    @Counted(name = "foo_created_total", description = "Total de Foos criados")
    @Timed(name = "foo_creation_duration", description = "Duracao da criacao de Foo")
    public FooResponse create(FooRequest request) {
        // ...
    }
}
```
**Health checks:**
```java
@Liveness
@ApplicationScoped
public class LivenessCheck implements HealthCheck {
    @Override
    public HealthCheckResponse call() {
        return HealthCheckResponse.up("alive");
    }
}
@Readiness
@ApplicationScoped
public class DatabaseHealthCheck implements HealthCheck {
    @PersistenceContext
    private EntityManager em;
    @Override
    public HealthCheckResponse call() {
        try {
            em.createNativeQuery("SELECT 1").getSingleResult();
            return HealthCheckResponse.up("database");
        } catch (Exception e) {
            return HealthCheckResponse.down("database");
        }
    }
}
```
**Logging estruturado (JSON):**
```java
@ApplicationScoped
public class FooService {
    private static final Logger LOG = Logger.getLogger(FooService.class.getName());
    public FooResponse create(FooRequest request) {
        LOG.info("Criando Foo: " + request.name());
        // ...
        LOG.info("Foo criado com sucesso: " + saved.getId());
        return fooMapper.toResponse(saved);
    }
}
```
**Endpoints de observabilidade:**
```
GET /health          → Health geral (liveness + readiness)
GET /health/live     → Liveness
GET /health/ready    → Readiness
GET /metrics         → Metricas (Prometheus format)
GET /openapi         → OpenAPI spec (JSON/YAML)
GET /openapi/ui      → Swagger UI
```
---
## Docker
### Dockerfile (WildFly)
```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-21-alpine AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn clean package -DskipTests -B
# Runtime stage
FROM quay.io/wildfly/wildfly:35.0.1.Final-jdk21
# Copiar o WAR
COPY --from=build /app/target/*.war /opt/jboss/wildfly/standalone/deployments/app.war
# Configuracoes customizadas (datasource, etc.)
COPY docker/standalone-custom.xml /opt/jboss/wildfly/standalone/configuration/standalone.xml
# Expor portas
EXPOSE 8080 9990
# Healthcheck
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8080/health/live || exit 1
# Startup
CMD ["/opt/jboss/wildfly/bin/standalone.sh", "-b", "0.0.0.0", "-bmanagement", "0.0.0.0"]
```
**Dockerfile alternativo (Open Liberty):**
```dockerfile
FROM icr.io/appcafe/open-liberty:full-java21-openj9-ubi-minimal
COPY --chown=1001:0 target/*.war /config/dropins/
COPY --chown=1001:0 src/main/liberty/config/ /config/
EXPOSE 9080 9443
RUN configure.sh
```
### Inicializacao Local
```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8080:8080"
      - "9990:9990"
    environment:
      - JAVA_OPTS=-Xms512m -Xmx1g
      - DB_HOST=mysql
      - DB_PORT=3306
      - DB_NAME=appdb
    depends_on:
      mysql:
        condition: service_healthy
  mysql:
    image: mysql:8.4
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: appdb
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
volumes:
  mysql_data:
```
```bash
# Subir tudo
docker compose up -d
# Logs
docker compose logs -f app
# Parar
docker compose down
# Rebuild
docker compose up -d --build
```
---
## Guia para Criar Novo Recurso
### 1. Domain
```
domain/
├── entity/FooEntity.java                 # @Entity + @Table("tb_foo")
├── entity/enums/FooStatus.java           # enum com fromValue()
├── exception/ResourceNotFoundException.java
├── repository/mysql/FooRepository.java   # @ApplicationScoped + EntityManager
└── service/FooService.java               # @ApplicationScoped + @Transactional
```
### 2. Core
```
core/
├── mapper/FooMapper.java                 # @Mapper(componentModel = "cdi")
└── validation/groups/                    # (se necessario)
```
### 3. API (REST)
```
api/rest/v1/
├── resource/FooResource.java             # @Path("/v1/foos") + CRUD
├── openapi/FooOpenApi.java               # @Tag + @Operation
├── request/FooRequest.java               # record + Bean Validation
└── response/FooResponse.java             # record
```
### 4. API (Mensageria) — se aplicavel
```
api/messaging/
├── jms/FooMessageConsumer.java           # @MessageDriven
└── jms/FooMessageProducer.java           # @Inject JMSContext
```
### 5. Database
```
src/main/resources/db/migration/
└── V{n}.0__create_table_tb_foo.sql       # DDL Flyway
```
### 6. Testes
```
test/
├── domain/service/FooServiceTest.java    # Unit (Mockito)
├── api/rest/v1/resource/FooResourceIT.java  # Integration (Arquillian)
└── resources/features/foo.feature        # BDD (opcional)
```
**Checklist:**
- [ ] Entity criada com `@Entity`, `@Table`, `@Id`, `@PrePersist`
- [ ] Repository com `EntityManager` ou Jakarta Data
- [ ] Service com `@ApplicationScoped`, `@Transactional`, `@Inject`
- [ ] Mapper MapStruct com `componentModel = "cdi"`
- [ ] Resource JAX-RS com `@Path`, `@Produces`, `@Consumes`
- [ ] Interface OpenAPI com `@Tag`, `@Operation`
- [ ] Request/Response como `record` com Bean Validation
- [ ] ExceptionMapper para `ResourceNotFoundException`
- [ ] Migration Flyway criada
- [ ] Testes unitarios do Service
- [ ] Testes de integracao do Resource
- [ ] Health check atualizado (se necessario)
---
## Onde colocar este arquivo para o Copilot
### 1. Instrucoes no nivel do repositorio (recomendado)
```
.github/copilot-instructions.md
```
Copie o conteudo deste arquivo para `.github/copilot-instructions.md` no repositorio alvo. O Copilot usara automaticamente como contexto para sugestoes.
### 2. Instrucoes por workspace (VS Code settings)
```json
// .vscode/settings.json
{
  "github.copilot.chat.codeGeneration.instructions": [
    { "file": ".docs/java/04-jakarta-ee.md" }
  ]
}
```
### 3. Instrucao inline (via `.instructions.md`)
Crie um arquivo `.instructions.md` na raiz ou em subpastas:
```markdown
Use o guia Jakarta EE definido em `.docs/java/04-jakarta-ee.md` como referencia
para estrutura de pacotes, convencoes e padroes de codigo.
```
### Recomendacao final
| Escopo | Metodo | Quando usar |
|--------|--------|-------------|
| Repositorio inteiro | `.github/copilot-instructions.md` | Projeto novo ou padrao unico |
| Workspace | `.vscode/settings.json` | Monorepo com multiplos padroes |
| Pasta especifica | `.instructions.md` | Modulo com convencoes diferentes |