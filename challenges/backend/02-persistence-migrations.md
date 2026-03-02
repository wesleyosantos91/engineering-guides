# Level 2 — Persistência e Migrações

> **Objetivo:** Substituir o store in-memory por banco de dados relacional real, aplicar migrações versionadas e dominar os padrões de persistência idiomáticos de cada stack.

---

## Objetivo de Aprendizado

- Integrar banco de dados relacional (PostgreSQL)
- Criar e evoluir schema via migrações versionadas (Flyway / golang-migrate)
- Implementar repository pattern com ORM/data framework de cada stack
- Gerenciar transações de forma correta
- Configurar connection pool
- Implementar paginação por cursor/offset
- Aplicar UUID como PK no banco

---

## Escopo Funcional

### Mesmos endpoints do Level 1 — agora com persistência real

**Novos endpoints:**

```
GET /api/v1/users?page=0&size=20&sort=name,asc    → Paginação
GET /api/v1/wallets/{id}/transactions?page=0&size=50&sort=createdAt,desc
```

### Regras de Negócio Adicionais

1. **Email único** → constraint no banco (UNIQUE)
2. **Transferência atômica** — saque + depósito na mesma transação
3. **Audit fields** — `created_at`, `updated_at` automáticos
4. **Soft delete** — coluna `deleted_at` (NULL = ativo)
5. **Optimistic locking** — `version` column para Wallet (evitar race condition no saldo)

---

## Escopo Técnico

### Banco de Dados

- **PostgreSQL 16+** via Docker
- UUID como PK (`gen_random_uuid()` no PostgreSQL)
- Timezone UTC em todas as timestamps

### Schema (3 tabelas)

```sql
-- V1__create_users.sql
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(100)  NOT NULL,
    email       VARCHAR(255)  NOT NULL UNIQUE,
    document    VARCHAR(20)   NOT NULL UNIQUE,
    status      VARCHAR(20)   NOT NULL DEFAULT 'ACTIVE',
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ
);

-- V2__create_wallets.sql
CREATE TABLE wallets (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID          NOT NULL REFERENCES users(id),
    currency    VARCHAR(3)    NOT NULL DEFAULT 'BRL',
    balance     NUMERIC(19,4) NOT NULL DEFAULT 0.0000,
    version     BIGINT        NOT NULL DEFAULT 0,
    status      VARCHAR(20)   NOT NULL DEFAULT 'ACTIVE',
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ
);

-- V3__create_transactions.sql
CREATE TABLE transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    wallet_id       UUID          NOT NULL REFERENCES wallets(id),
    type            VARCHAR(20)   NOT NULL,  -- DEPOSIT, WITHDRAWAL, TRANSFER_IN, TRANSFER_OUT
    amount          NUMERIC(19,4) NOT NULL,
    balance_after   NUMERIC(19,4) NOT NULL,
    idempotency_key UUID          UNIQUE,
    description     VARCHAR(255),
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT now()
);

CREATE INDEX idx_transactions_wallet_id ON transactions(wallet_id);
CREATE INDEX idx_transactions_created_at ON transactions(created_at);
CREATE INDEX idx_wallets_user_id ON wallets(user_id);
```

### Migrações por Stack

| Stack | Ferramenta | Diretório |
|---|---|---|
| Go (Gin) | golang-migrate | `migrations/` |
| Spring Boot | Flyway | `src/main/resources/db/migration/` |
| Quarkus | Flyway | `src/main/resources/db/migration/` |
| Micronaut | Flyway | `src/main/resources/db/migration/` |
| Jakarta EE | Flyway | `src/main/resources/db/migration/` |

### ORM/Data Framework por Stack

| Stack | Framework | Padrão |
|---|---|---|
| **Go (Gin)** | GORM | `store.UserStore` interface + `store.gormUserStore` struct |
| **Spring Boot** | Spring Data JPA + Hibernate | `JpaRepository<User, UUID>` |
| **Quarkus** | Hibernate ORM with Panache | `PanacheRepositoryBase<User, UUID>` |
| **Micronaut** | Micronaut Data JPA | `@Repository interface UserRepository extends JpaRepository<User, UUID>` |
| **Jakarta EE** | JPA 3.2 + Jakarta Data 1.0 | `@Repository interface UserRepository extends BasicRepository<User, UUID>` |

---

## Critérios de Aceite

- [ ] PostgreSQL rodando via `docker-compose.yml`
- [ ] Migrações executam sem erro no startup
- [ ] Dados persistem entre restarts da aplicação
- [ ] Email duplicado retorna 409 (constraint violation tratada)
- [ ] Paginação funciona com `page`, `size`, `sort`
- [ ] Resposta paginada contém metadados: `totalElements`, `totalPages`, `page`, `size`
- [ ] Soft delete funciona (DELETE não remove fisicamente)
- [ ] Transferência é atômica (saque + depósito na mesma tx)
- [ ] Optimistic locking funciona na Wallet (testar com 2 requests concorrentes)
- [ ] Audit fields (`created_at`, `updated_at`) preenchidos automaticamente
- [ ] Connection pool configurado (HikariCP / pgxpool / c3p0)

---

## Definição de Pronto (DoD)

- [ ] 3+ migrações Flyway/golang-migrate aplicadas sem conflito
- [ ] Entity ≠ DTO: entidade JPA/GORM nunca exposta na API
- [ ] Mapper (MapStruct / manual) converte Entity → Response
- [ ] Queries otimizadas (sem N+1 — verificar com logs SQL)
- [ ] Testes rodam com Testcontainers (PostgreSQL real)
- [ ] `docker-compose up` + `go run ./cmd/api` / `./mvnw spring-boot:run` funciona do zero
- [ ] Commit: `feat(level-2): add PostgreSQL persistence with Flyway migrations`

---

## Checklist

### Go (Gin)
- [ ] `docker-compose.yml` com PostgreSQL 16
- [ ] `migrations/000001_create_users.up.sql` / `.down.sql`
- [ ] `migrations/000002_create_wallets.up.sql` / `.down.sql`
- [ ] `migrations/000003_create_transactions.up.sql` / `.down.sql`
- [ ] `internal/platform/database/postgres.go` — conexão GORM + pool config
- [ ] `internal/store/user_gorm.go` — implementação `UserStore` com GORM
- [ ] `internal/store/wallet_gorm.go`
- [ ] `internal/store/transaction_gorm.go`
- [ ] `internal/model/user.go` — adicionar GORM tags (`gorm:"primaryKey;type:uuid;default:gen_random_uuid()"`)
- [ ] `internal/model/pagination.go` — `PageRequest` + `PageResponse[T]`
- [ ] `internal/platform/database/migrate.go` — execução de migrações no startup
- [ ] Variáveis de ambiente: `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_SSL_MODE`

### Java — Spring Boot
- [ ] `docker-compose.yml` com PostgreSQL 16
- [ ] `src/main/resources/db/migration/V1__create_users.sql`
- [ ] `src/main/resources/db/migration/V2__create_wallets.sql`
- [ ] `src/main/resources/db/migration/V3__create_transactions.sql`
- [ ] `infrastructure/entity/UserEntity.java` — `@Entity` com `@Table`
- [ ] `infrastructure/entity/WalletEntity.java` — `@Version` para optimistic locking
- [ ] `infrastructure/entity/TransactionEntity.java`
- [ ] `infrastructure/jpa/UserJpaRepository.java` — `extends JpaRepository<UserEntity, UUID>`
- [ ] `domain/repository/UserRepository.java` — interface de domínio (port)
- [ ] `infrastructure/adapter/UserRepositoryAdapter.java` — implementação com JPA
- [ ] `core/mapper/UserMapper.java` — MapStruct `@Mapper(componentModel="spring")`
- [ ] `core/mapper/WalletMapper.java`
- [ ] `api/rest/v1/response/PageResponse.java` — record genérico
- [ ] `application.yml` — `spring.datasource.*`, `spring.jpa.hibernate.ddl-auto=validate`

### Quarkus
- [ ] `@Entity` com Panache: `extends PanacheEntityBase` (UUID PK, não auto-id)
- [ ] `@Transactional` do Jakarta Transactions
- [ ] `application.properties` → `quarkus.datasource.jdbc.url`, `quarkus.flyway.migrate-at-start=true`
- [ ] Dev Services para PostgreSQL automático em teste (`quarkus.datasource.devservices.enabled=true`)
- [ ] Repository pattern Panache: `PanacheRepositoryBase<UserEntity, UUID>`

### Micronaut
- [ ] `@MappedEntity` com Micronaut Data
- [ ] `@Repository interface UserRepository extends PageableRepository<UserEntity, UUID>`
- [ ] `@Transactional` do Jakarta Transactions
- [ ] `application.yml` → `datasources.default.url`, `flyway.datasources.default.enabled: true`
- [ ] Compile-time query generation (verificar que não usa reflection)

### Jakarta EE
- [ ] `@Entity` JPA 3.2 com `persistence.xml`
- [ ] `@Repository interface UserRepository extends BasicRepository<UserEntity, UUID>` (Jakarta Data 1.0)
- [ ] `@Transactional` do Jakarta Transactions
- [ ] `EntityManager` injection via CDI onde necessário
- [ ] Flyway via `org.flywaydb:flyway-core` dependency + startup hook

---

## Tarefas Sugeridas por Stack

### Go — Detalhamento de Interface

```go
// internal/store/interfaces.go
type UserStore interface {
    Create(ctx context.Context, user *model.User) error
    FindByID(ctx context.Context, id uuid.UUID) (*model.User, error)
    FindAll(ctx context.Context, page model.PageRequest) (*model.PageResponse[model.User], error)
    Update(ctx context.Context, user *model.User) error
    SoftDelete(ctx context.Context, id uuid.UUID) error
    ExistsByEmail(ctx context.Context, email string) (bool, error)
}
```

### Spring Boot — Entity Pattern

```java
@Entity
@Table(name = "users")
public class UserEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false, unique = true, length = 20)
    private String document;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private Status status = Status.ACTIVE;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @Column(name = "deleted_at")
    private Instant deletedAt;

    @PrePersist
    void prePersist() {
        this.createdAt = Instant.now();
        this.updatedAt = Instant.now();
    }

    @PreUpdate
    void preUpdate() {
        this.updatedAt = Instant.now();
    }
}
```

### Quarkus — Panache Repository

```java
@ApplicationScoped
public class UserPanacheRepository implements PanacheRepositoryBase<UserEntity, UUID> {

    public Optional<UserEntity> findByEmail(String email) {
        return find("email", email).firstResultOptional();
    }

    public PanacheQuery<UserEntity> findAllActive(Sort sort) {
        return find("deletedAt IS NULL", sort);
    }
}
```

---

## docker-compose.yml (compartilhado)

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: digital_wallet
      POSTGRES_USER: wallet_user
      POSTGRES_PASSWORD: wallet_pass
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U wallet_user -d digital_wallet"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

---

## Extensões Opcionais

- [ ] Implementar idempotency_key para transações (evitar depósito duplicado)
- [ ] Adicionar índice GIN para busca por nome parcial (`ILIKE`)
- [ ] Configurar read replicas (datasource secundário read-only)
- [ ] Implementar batch insert para carga inicial de dados
- [ ] Adicionar Liquibase como alternativa ao Flyway

---

## Erros Comuns

| Erro | Stack | Como evitar |
|---|---|---|
| Expor Entity diretamente na API | Todos | Entity ≠ DTO — sempre converter com mapper |
| N+1 queries em listagem com relacionamentos | Todos | Usar `JOIN FETCH` / `Preload` / `@EntityGraph` |
| Não tratar `DataIntegrityViolationException` | Java | Capturar e traduzir para 409 no exception handler |
| Esquecer `@Transactional` em operações compostas | Java | Transferência = saque + depósito → 1 transação |
| Não usar `context.Context` em queries | Go | Sempre propagar ctx para timeout/cancelamento |
| Auto-DDL em produção (`ddl-auto=update`) | Java | Usar `validate` e migrar apenas com Flyway |
| `NUMERIC` sem precisão adequada | Todos | Usar `NUMERIC(19,4)` para valores monetários |
| Não fechar conexões em Go | Go | Usar `defer db.Close()` e configurar pool |

---

## Como Isso Aparece em Entrevistas

- "Explique a diferença entre repository de domínio e repository de infraestrutura"
- "Como você garante atomicidade em uma transferência entre wallets?"
- "O que é optimistic locking e quando usar?"
- "Como funciona o Flyway? Qual o fluxo de versionamento?"
- "Qual a diferença entre `ddl-auto=update` e Flyway? Qual é melhor?"
- "Como você lida com N+1 queries? Dê um exemplo prático"
- "Como funciona`@Version` no JPA?"
- "Em Go, como você injeta o repositório no service sem annotation de DI?"
- "Como funciona o Panache comparado ao Spring Data JPA?"

---

## Comandos de Execução

```bash
# Subir banco
docker-compose up -d postgres

# Verificar se está healthy
docker-compose ps

# Rodar aplicação (escolha sua stack)
# Go
go run ./cmd/api

# Spring Boot
./mvnw spring-boot:run -Dspring-boot.run.profiles=local

# Quarkus (Dev Services sobe o banco automaticamente)
./mvnw quarkus:dev

# Micronaut
./mvnw mn:run -Dmicronaut.environments=local

# Jakarta EE
./mvnw package && ./mvnw wildfly:deploy

# Verificar migrações aplicadas
docker exec -it <container> psql -U wallet_user -d digital_wallet -c "SELECT * FROM flyway_schema_history;"
```
