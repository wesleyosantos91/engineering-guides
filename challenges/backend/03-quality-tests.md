# Level 3 — Qualidade e Testes

> **Objetivo:** Atingir cobertura de testes abrangente com testes unitários, de integração e E2E, aplicando as ferramentas e padrões idiomáticos de cada stack.

---

## Objetivo de Aprendizado

- Escrever testes unitários com mocks/stubs
- Escrever testes de integração com banco real (Testcontainers)
- Implementar table-driven tests (Go) / testes parametrizados (Java)
- Configurar cobertura de código (≥ 80%)
- Introduzir mutation testing como medida de qualidade
- Implementar contract testing para API
- Estabelecer CI pipeline básico com quality gate

---

## Escopo Funcional

### Cenários de Teste Obrigatórios

```
── User ──
✅ Criar usuário com dados válidos → 201
✅ Criar usuário com email duplicado → 409
✅ Criar usuário com dados inválidos → 400 (ProblemDetail)
✅ Buscar usuário existente → 200
✅ Buscar usuário inexistente → 404
✅ Deletar (soft) usuário → 204
✅ Buscar usuário deletado → 404

── Wallet ──
✅ Criar wallet para usuário existente → 201
✅ Criar wallet para usuário inexistente → 404
✅ Buscar wallet por ID → 200

── Transaction ──
✅ Depósito com valor válido → 201 + saldo atualizado
✅ Depósito com valor ≤ 0 → 400
✅ Saque com saldo suficiente → 201 + saldo atualizado
✅ Saque sem saldo → 422 (INSUFFICIENT_BALANCE)
✅ Transferência atômica entre wallets → 201 + saldos atualizados
✅ Transferência para wallet inexistente → 404
✅ Listagem paginada de transações → 200 com metadados
```

---

## Escopo Técnico

### Pirâmide de Testes

```
        ╱╲
       ╱ E2E ╲         → Poucos: fluxo completo com HTTP client
      ╱────────╲
     ╱ Integration╲    → Médio: com banco real (Testcontainers)
    ╱──────────────╲
   ╱  Unit Tests     ╲  → Muitos: service layer + validações + mappers
  ╱────────────────────╲
```

### Ferramentas por Stack

| Aspecto | Go (Gin) | Spring Boot | Quarkus | Micronaut | Jakarta EE |
|---|---|---|---|---|---|
| **Unit** | `testing` + testify | JUnit 5 + Mockito | JUnit 5 + Mockito | JUnit 5 + Mockito | JUnit 5 + Mockito |
| **Mocks** | mockery / gomock | Mockito `@Mock` | Mockito `@Mock` | Mockito `@Mock` | Mockito `@Mock` |
| **Integration** | testcontainers-go | `@SpringBootTest` + Testcontainers | `@QuarkusTest` + Dev Services | `@MicronautTest` + Testcontainers | Arquillian + Testcontainers |
| **HTTP client** | `net/http/httptest` | `WebTestClient` / `MockMvc` | `RestAssured` | `HttpClient` (blocking) | `RestAssured` / JAX-RS Client |
| **Coverage** | `go test -cover` | JaCoCo | JaCoCo | JaCoCo | JaCoCo |
| **Mutation** | — | PIT (pitest) | PIT (pitest) | PIT (pitest) | PIT (pitest) |
| **BDD** | godog (opcional) | Cucumber (opcional) | Cucumber (opcional) | Cucumber (opcional) | Cucumber (opcional) |

---

## Critérios de Aceite

- [ ] ≥ 80% cobertura de linha no service layer
- [ ] ≥ 70% cobertura geral do projeto
- [ ] Todos os cenários listados acima cobertos
- [ ] Testes unitários rodam sem Docker/rede (< 5s total)
- [ ] Testes de integração usam Testcontainers/Dev Services
- [ ] Nenhum teste depende de ordem de execução
- [ ] Nenhum teste usa `Thread.sleep()` / `time.Sleep()` para sincronização
- [ ] Mutation score ≥ 60% no service layer (Java stacks)
- [ ] CI pipeline roda: lint → test → coverage → report

---

## Definição de Pronto (DoD)

- [ ] Testes unitários para todos os services (mock do repository)
- [ ] Testes de integração para todos os endpoints (banco real)
- [ ] Coverage report gerado (`go tool cover -html` / JaCoCo HTML)
- [ ] Factory/Builder para criação de fixtures de teste
- [ ] PIT mutation report gerado (pelo menos para 1 service em Java)
- [ ] `go test ./...` / `./mvnw verify` passa com 0 falhas
- [ ] Commit: `feat(level-3): add unit and integration tests with 80%+ coverage`

---

## Checklist

### Go (Gin)

- [ ] `internal/service/user_test.go` — table-driven tests com testify
- [ ] `internal/service/wallet_test.go`
- [ ] `internal/service/transaction_test.go` — testar saque sem saldo, transferência
- [ ] `internal/store/user_test.go` — integration test com testcontainers-go (PostgreSQL)
- [ ] `internal/store/wallet_test.go` — integration test
- [ ] `internal/handler/user_test.go` — HTTP tests com `httptest.NewRecorder`
- [ ] `internal/handler/wallet_test.go`
- [ ] `internal/handler/transaction_test.go`
- [ ] `internal/testutil/factory.go` — factories: `NewTestUser()`, `NewTestWallet()`
- [ ] `internal/testutil/containers.go` — helper para subir PostgreSQL com testcontainers
- [ ] `Makefile` — target `test`, `test-cover`, `test-integration`

### Java — Spring Boot

- [ ] `domain/service/UserServiceTest.java` — `@ExtendWith(MockitoExtension.class)`
- [ ] `domain/service/WalletServiceTest.java`
- [ ] `domain/service/TransactionServiceTest.java`
- [ ] `api/rest/v1/controller/UserControllerTest.java` — `@WebMvcTest`
- [ ] `api/rest/v1/controller/WalletControllerTest.java`
- [ ] `api/rest/v1/controller/TransactionControllerTest.java`
- [ ] `integration/UserIntegrationTest.java` — `@SpringBootTest` + `@Testcontainers`
- [ ] `integration/TransactionIntegrationTest.java` — testar transferência atômica
- [ ] `testutil/UserFactory.java` — builder/factory para entidades
- [ ] `testutil/PostgresContainerConfig.java` — `@TestConfiguration` + `@DynamicPropertySource`
- [ ] `pom.xml` — JaCoCo plugin + PIT plugin configurados
- [ ] `application-test.yml` — profile de teste

### Quarkus

- [ ] `@QuarkusTest` para integration tests (Dev Services sobe o banco)
- [ ] `@QuarkusTestResource` para configuração customizada
- [ ] `RestAssured.given()...` para testes HTTP
- [ ] `@InjectMock` para mock de CDI beans em testes unitários
- [ ] Testes nativos: `@QuarkusIntegrationTest` (rodar com `-Dnative`)

### Micronaut

- [ ] `@MicronautTest` para integration tests
- [ ] `@MockBean` para substituir beans em testes
- [ ] `HttpClient` injetado para testes HTTP
- [ ] `@Requires(env = "test")` para beans de teste
- [ ] Testcontainers via `@TestResourcesScope`

### Jakarta EE

- [ ] Arquillian para integration tests (micro deployment)
- [ ] `@ShrinkWrap` para montar o WAR de teste
- [ ] RestAssured para testes HTTP
- [ ] CDI-Unit ou Weld-Testing para testes unitários de CDI
- [ ] `persistence-test.xml` com datasource de teste

---

## Tarefas Sugeridas por Stack

### Go — Table-Driven Test

```go
func TestUserService_Create(t *testing.T) {
    tests := []struct {
        name      string
        input     model.CreateUserInput
        mockSetup func(store *mocks.MockUserStore)
        wantErr   error
    }{
        {
            name:  "success",
            input: model.CreateUserInput{Name: "João", Email: "joao@test.com", Document: "123.456.789-00"},
            mockSetup: func(s *mocks.MockUserStore) {
                s.EXPECT().ExistsByEmail(gomock.Any(), "joao@test.com").Return(false, nil)
                s.EXPECT().Create(gomock.Any(), gomock.Any()).Return(nil)
            },
            wantErr: nil,
        },
        {
            name:  "duplicate email",
            input: model.CreateUserInput{Name: "João", Email: "joao@test.com", Document: "123.456.789-00"},
            mockSetup: func(s *mocks.MockUserStore) {
                s.EXPECT().ExistsByEmail(gomock.Any(), "joao@test.com").Return(true, nil)
            },
            wantErr: errs.ErrConflict,
        },
        {
            name:  "insufficient balance on withdrawal",
            // ...
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            ctrl := gomock.NewController(t)
            defer ctrl.Finish()

            store := mocks.NewMockUserStore(ctrl)
            tt.mockSetup(store)

            svc := service.NewUserService(store, zap.NewNop())
            _, err := svc.Create(context.Background(), tt.input)

            if tt.wantErr != nil {
                assert.ErrorIs(t, err, tt.wantErr)
            } else {
                assert.NoError(t, err)
            }
        })
    }
}
```

### Go — Testcontainers Integration

```go
func TestUserStore_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    ctx := context.Background()
    pg, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image:        "postgres:16-alpine",
            ExposedPorts: []string{"5432/tcp"},
            Env: map[string]string{
                "POSTGRES_DB":       "testdb",
                "POSTGRES_USER":     "test",
                "POSTGRES_PASSWORD": "test",
            },
            WaitingFor: wait.ForListeningPort("5432/tcp").WithStartupTimeout(30 * time.Second),
        },
        Started: true,
    })
    require.NoError(t, err)
    defer pg.Terminate(ctx)

    // ... conectar, migrar, testar
}
```

### Spring Boot — Integration Test

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
class UserIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    WebTestClient webClient;

    @Test
    void shouldCreateUser() {
        webClient.post().uri("/api/v1/users")
                .bodyValue(new UserRequest("João", "joao@test.com", "123.456.789-00"))
                .exchange()
                .expectStatus().isCreated()
                .expectBody()
                .jsonPath("$.id").isNotEmpty()
                .jsonPath("$.name").isEqualTo("João");
    }

    @Test
    void shouldReturn409ForDuplicateEmail() {
        // Criar primeiro
        webClient.post().uri("/api/v1/users")
                .bodyValue(new UserRequest("João", "dup@test.com", "111.222.333-44"))
                .exchange()
                .expectStatus().isCreated();

        // Tentar duplicar
        webClient.post().uri("/api/v1/users")
                .bodyValue(new UserRequest("Maria", "dup@test.com", "555.666.777-88"))
                .exchange()
                .expectStatus().isEqualTo(409)
                .expectBody()
                .jsonPath("$.type").isEqualTo("CONFLICT")
                .jsonPath("$.detail").isNotEmpty();
    }
}
```

---

## Configuração de Coverage

### Go

```bash
# Coverage com relatório HTML
go test ./internal/... -coverprofile=coverage.out
go tool cover -html=coverage.out -o coverage.html
go tool cover -func=coverage.out | tail -1  # total

# Rodar apenas testes curtos (sem integração)
go test -short ./internal/...
```

### Java (JaCoCo — pom.xml)

```xml
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
        <execution>
            <id>check</id>
            <phase>verify</phase>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.70</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Java (PIT — Mutation Testing)

```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.16.1</version>
    <dependencies>
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
    <configuration>
        <targetClasses>
            <param>com.example.wallet.domain.service.*</param>
        </targetClasses>
        <mutationThreshold>60</mutationThreshold>
    </configuration>
</plugin>
```

```bash
# Rodar mutation testing
./mvnw org.pitest:pitest-maven:mutationCoverage
# Report em target/pit-reports/
```

---

## Extensões Opcionais

- [ ] Adicionar contract testing com Pact (consumer-driven)
- [ ] Implementar BDD com Cucumber/godog para cenários de negócio
- [ ] Adicionar snapshot testing para respostas JSON
- [ ] Configurar GitHub Actions CI com coverage badge
- [ ] Implementar property-based testing (jqwik / rapid)
- [ ] Adicionar testes de performance com JMH (microbenchmark)

---

## Erros Comuns

| Erro | Stack | Como evitar |
|---|---|---|
| Testar implementação e não comportamento | Todos | Teste o "o quê", não o "como" — assertions em output, não calls internas |
| Não limpar dados entre testes | Todos | `@Transactional` (rollback) ou truncate tables no `@BeforeEach` |
| Usar `Thread.sleep()` para esperar async | Todos | Usar `Awaitility` (Java) ou polling pattern (Go) |
| Mock de tudo → teste não testa nada | Todos | Mocke apenas dependências externas, nunca o SUT |
| Testes de integração lentos | Todos | Singleton container pattern — 1 container por suite |
| Teste depende de ordem de execução | Todos | Cada teste deve ser independente e idempotente |
| Coverage alta mas mutation score baixo | Java | Assertions fracas — adicione verificações de valor |
| `httptest.NewRecorder` sem middleware | Go | Router completo no teste inclui middleware (recovery, etc.) |

---

## Como Isso Aparece em Entrevistas

- "Qual a diferença entre teste unitário e de integração? Quando usar cada um?"
- "O que é Testcontainers e por que não usar H2 em testes?"
- "Explique table-driven tests em Go"
- "O que é mutation testing? O que um mutation score de 60% significa?"
- "Como você garante que testes são independentes entre si?"
- "Qual a diferença entre `@WebMvcTest` e `@SpringBootTest`?"
- "Como funciona o singleton container pattern com Testcontainers?"
- "O que são Dev Services no Quarkus e como eles simplificam testes?"
- "Qual cobertura mínima você considera aceitável e por quê?"

---

## Comandos de Execução

```bash
# Go
go test ./internal/... -v -cover
go test ./internal/... -v -run Integration -count=1
go test -short ./internal/...

# Spring Boot
./mvnw test                          # unitários
./mvnw verify                        # unitários + integração
./mvnw verify -Pjacoco               # com coverage report
./mvnw org.pitest:pitest-maven:mutationCoverage  # mutation

# Quarkus
./mvnw test                          # unitários + @QuarkusTest
./mvnw verify -Dnative               # testes em modo nativo

# Micronaut
./mvnw test

# Jakarta EE
./mvnw test -Parquillian-managed
```
