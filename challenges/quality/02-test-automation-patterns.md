# Level 2 — Padrões de Automação de Testes

> **Objetivo:** Dominar padrões reutilizáveis de automação — builders, factories, fixtures,
> asserções expressivas, testes de API, testes de banco de dados e refatoração de testes —
> aplicados ao domínio Digital Wallet em Go e Java.

**Referência:** [.docs/QUALITY-ENGINEERING/02-test-automation-patterns.md](../../.docs/QUALITY-ENGINEERING/02-test-automation-patterns.md)

---

## Contexto do Domínio

Neste nível, você criará uma **infraestrutura de testes robusta e reutilizável** para a Digital Wallet.
O foco é em padrões que reduzem duplicação, aumentam legibilidade e facilitam manutenção
da suíte de testes à medida que o projeto cresce.

---

## Desafios

### Desafio 2.1 — Test Data Builders

**Contexto:** A Digital Wallet tem entidades complexas (`User`, `Wallet`, `Transaction`) com muitos
campos. Criar esses objetos em cada teste é verboso e frágil. Builders resolvem isso.

**Requisitos:**

- Implementar **Builder Pattern** para todas as entidades:
  - `UserBuilder` — valores default sensatos, sobrescrever apenas o relevante
  - `WalletBuilder` — default: saldo 0, moeda BRL, status ACTIVE
  - `TransactionBuilder` — default: tipo DEPOSIT, status PENDING, valor 100.00
- Cada builder deve:
  - Ter um método estático factory (`aUser()`, `aWallet()`, `aTransaction()`)
  - Suportar encadeamento fluent (`aWallet().withBalance("500.00").withCurrency("USD").build()`)
  - Gerar IDs automaticamente (UUID)
  - Gerar timestamps automaticamente (`Instant.now()` / `time.Now()`)

**Java 25:**
```java
public class WalletBuilder {
    private UUID id = UUID.randomUUID();
    private UUID userId = UUID.randomUUID();
    private BigDecimal balance = BigDecimal.ZERO;
    private String currency = "BRL";
    private WalletStatus status = WalletStatus.ACTIVE;
    private Instant createdAt = Instant.now();
    private Instant updatedAt = Instant.now();

    public static WalletBuilder aWallet() {
        return new WalletBuilder();
    }

    public WalletBuilder withId(UUID id) { this.id = id; return this; }
    public WalletBuilder withUserId(UUID userId) { this.userId = userId; return this; }
    public WalletBuilder withBalance(String balance) {
        this.balance = new BigDecimal(balance);
        return this;
    }
    public WalletBuilder withCurrency(String currency) { this.currency = currency; return this; }
    public WalletBuilder withStatus(WalletStatus status) { this.status = status; return this; }

    public Wallet build() {
        return new Wallet(id, userId, balance, currency, status, createdAt, updatedAt);
    }

    // Pré-configurados para cenários comuns
    public static WalletBuilder aWalletWithBalance(String balance) {
        return aWallet().withBalance(balance);
    }

    public static WalletBuilder aBlockedWallet() {
        return aWallet().withStatus(WalletStatus.BLOCKED);
    }
}
```

**Go 1.26 (functional options pattern):**
```go
type WalletOption func(*model.Wallet)

func WithBalance(balance float64) WalletOption {
    return func(w *model.Wallet) { w.Balance = balance }
}

func WithCurrency(currency string) WalletOption {
    return func(w *model.Wallet) { w.Currency = currency }
}

func WithWalletStatus(status model.WalletStatus) WalletOption {
    return func(w *model.Wallet) { w.Status = status }
}

func WithUserID(userID uuid.UUID) WalletOption {
    return func(w *model.Wallet) { w.UserID = userID }
}

func NewTestWallet(opts ...WalletOption) model.Wallet {
    w := model.Wallet{
        ID:        uuid.New(),
        UserID:    uuid.New(),
        Balance:   0.00,
        Currency:  "BRL",
        Status:    model.WalletStatusActive,
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }
    for _, opt := range opts {
        opt(&w)
    }
    return w
}

// Pré-configurados
func NewTestWalletWithBalance(balance float64) model.Wallet {
    return NewTestWallet(WithBalance(balance))
}

func NewTestBlockedWallet() model.Wallet {
    return NewTestWallet(WithWalletStatus(model.WalletStatusBlocked))
}
```

**Critérios de aceite:**

- [ ] Builders para `User`, `Wallet`, `Transaction` (Java e Go)
- [ ] Valores default sensatos para todos os campos
- [ ] API fluent com encadeamento
- [ ] Métodos pré-configurados para cenários comuns
- [ ] Usados em pelo menos 5 testes existentes (refatorar)
- [ ] Nenhum teste cria entidades com construtores diretamente

---

### Desafio 2.2 — Fixture Management e Setup/Cleanup

**Contexto:** Testes de integração precisam de dados no banco. O gerenciamento de fixtures
(criação e limpeza de dados) deve ser consistente e isolado.

**Requisitos:**

- Implementar **3 padrões** de fixture management e documentar quando usar cada um:

| Padrão | Quando usar | Trade-off |
|--------|-------------|-----------|
| **Truncate before each** | Suíte inteira compartilha container | Mais lento, isolamento total |
| **Transaction rollback** | Spring/Quarkus com `@Transactional` | Rápido, mas não testa commit real |
| **Unique data per test** | Dados com IDs únicos, sem cleanup | Rápido, acumula dados |

- Implementar helper de limpeza:

**Java 25:**
```java
@TestConfiguration
public class DatabaseCleaner {

    @Autowired
    private JdbcTemplate jdbc;

    @Transactional
    public void cleanAll() {
        jdbc.execute("TRUNCATE TABLE transactions CASCADE");
        jdbc.execute("TRUNCATE TABLE wallets CASCADE");
        jdbc.execute("TRUNCATE TABLE users CASCADE");
    }
}

// Uso no teste
@SpringBootTest
class WalletRepositoryIT {
    @Autowired DatabaseCleaner cleaner;

    @BeforeEach
    void setUp() {
        cleaner.cleanAll();
    }
}
```

**Go 1.26:**
```go
type DatabaseCleaner struct {
    db *sql.DB
}

func NewDatabaseCleaner(db *sql.DB) *DatabaseCleaner {
    return &DatabaseCleaner{db: db}
}

func (c *DatabaseCleaner) CleanAll(ctx context.Context) error {
    tables := []string{"transactions", "wallets", "users"}
    for _, table := range tables {
        if _, err := c.db.ExecContext(ctx, "TRUNCATE TABLE "+table+" CASCADE"); err != nil {
            return fmt.Errorf("truncate %s: %w", table, err)
        }
    }
    return nil
}
```

- Implementar **Container per Suite** compartilhado:

**Java 25:**
```java
// Classe base para todos os integration tests
@Testcontainers
abstract class AbstractDatabaseIT {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine")
                    .withDatabaseName("wallet_test")
                    .withReuse(true);  // Reutilizar entre suítes
}
```

**Critérios de aceite:**

- [ ] DatabaseCleaner implementado (Java e Go)
- [ ] Container compartilhado entre testes de integração
- [ ] 3 padrões de fixture documentados com prós/contras
- [ ] Todos os testes de integração isolados (rodam em qualquer ordem)
- [ ] Nenhum dado residual entre testes

---

### Desafio 2.3 — Asserções Expressivas e Custom Matchers

**Contexto:** Asserções genéricas (`assertEquals`, `assert.Equal`) são pouco expressivas
e produzem mensagens de erro ruins. Asserções customizadas aumentam legibilidade e debuggabilidade.

**Requisitos:**

- Implementar **custom assertions** para o domínio Digital Wallet:

**Java 25 (AssertJ custom assertions):**
```java
public class WalletAssert extends AbstractAssert<WalletAssert, Wallet> {

    public WalletAssert(Wallet actual) {
        super(actual, WalletAssert.class);
    }

    public static WalletAssert assertThatWallet(Wallet wallet) {
        return new WalletAssert(wallet);
    }

    public WalletAssert hasBalance(String expectedBalance) {
        isNotNull();
        var expected = new BigDecimal(expectedBalance);
        if (actual.getBalance().compareTo(expected) != 0) {
            failWithMessage("Expected wallet balance to be <%s> but was <%s>",
                    expected, actual.getBalance());
        }
        return this;
    }

    public WalletAssert isActive() {
        isNotNull();
        if (actual.getStatus() != WalletStatus.ACTIVE) {
            failWithMessage("Expected wallet to be ACTIVE but was <%s>", actual.getStatus());
        }
        return this;
    }

    public WalletAssert belongsToUser(UUID userId) {
        isNotNull();
        if (!actual.getUserId().equals(userId)) {
            failWithMessage("Expected wallet to belong to user <%s> but belongs to <%s>",
                    userId, actual.getUserId());
        }
        return this;
    }
}

// Uso no teste
assertThatWallet(wallet)
    .hasBalance("150.00")
    .isActive()
    .belongsToUser(userId);
```

**Go 1.26 (custom test helpers):**
```go
// testutil/assertions.go
func AssertWalletBalance(t *testing.T, wallet model.Wallet, expectedBalance float64) {
    t.Helper()
    if math.Abs(wallet.Balance-expectedBalance) > 0.001 {
        t.Errorf("Expected wallet %s balance to be %.2f but was %.2f",
            wallet.ID, expectedBalance, wallet.Balance)
    }
}

func AssertWalletActive(t *testing.T, wallet model.Wallet) {
    t.Helper()
    if wallet.Status != model.WalletStatusActive {
        t.Errorf("Expected wallet %s to be ACTIVE but was %s", wallet.ID, wallet.Status)
    }
}

func AssertTransactionCompleted(t *testing.T, tx model.Transaction) {
    t.Helper()
    assert.Equal(t, model.StatusCompleted, tx.Status,
        "Transaction %s should be COMPLETED", tx.ID)
}

// Uso
AssertWalletBalance(t, wallet, 150.00)
AssertWalletActive(t, wallet)
```

- Implementar **soft assertions** (coletar todas as falhas antes de reportar):

```java
@Test
void transfer_updatesSourceAndTargetBalances() {
    // ... realizar transferência

    SoftAssertions.assertSoftly(softly -> {
        softly.assertThat(source.getBalance()).isEqualByComparingTo("50.00");
        softly.assertThat(target.getBalance()).isEqualByComparingTo("150.00");
        softly.assertThat(transaction.getStatus()).isEqualTo(COMPLETED);
        softly.assertThat(transaction.getType()).isEqualTo(TRANSFER);
    });
}
```

**Critérios de aceite:**

- [ ] Custom assertions para `Wallet` e `Transaction` (Java e Go)
- [ ] Mensagens de erro descritivas com valores esperado/atual
- [ ] Soft assertions usadas em pelo menos 2 testes
- [ ] Usadas em pelo menos 5 testes existentes (refatorar)
- [ ] `t.Helper()` usado em todas as helpers de Go

---

### Desafio 2.4 — Testes de API com WireMock / HTTP Stubs

**Contexto:** A Digital Wallet integra com uma API externa de **cotação de moeda** ou
**validação de fraude**. Testes de integração dessa camada devem usar stubs HTTP.

**Requisitos:**

- Implementar stubs para a API externa:
  - `GET /api/exchange-rate?from=BRL&to=USD` → retorna `{ "rate": 0.20 }`
  - `POST /api/fraud-check` → retorna `{ "approved": true, "score": 0.1 }`
  - Simular cenários de erro: timeout (5s+), 500 Internal Server Error, resposta malformada
- Implementar testes que validem:
  - Resposta normal: client retorna dados corretos
  - Timeout: client retorna erro após threshold (ex.: 3s)
  - 500: client retorna erro e não faz retry (ou faz retry controlado)
  - Resposta malformada: client retorna erro de parsing

**Java 25 (WireMock):**
```java
@WireMockTest(httpPort = 8089)
class ExchangeRateClientTest {

    @Autowired ExchangeRateClient client;

    @Test
    void getRate_success_returnsRate() {
        stubFor(get(urlPathEqualTo("/api/exchange-rate"))
                .withQueryParam("from", equalTo("BRL"))
                .withQueryParam("to", equalTo("USD"))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                            { "rate": 0.20, "timestamp": "2026-03-01T10:00:00Z" }
                            """)));

        var rate = client.getRate("BRL", "USD");

        assertThat(rate.rate()).isEqualByComparingTo("0.20");
    }

    @Test
    void getRate_timeout_throwsTimeoutException() {
        stubFor(get(urlPathEqualTo("/api/exchange-rate"))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withFixedDelay(5000)));  // 5s delay

        assertThatThrownBy(() -> client.getRate("BRL", "USD"))
                .isInstanceOf(TimeoutException.class);
    }

    @Test
    void getRate_serverError_throwsServiceUnavailableException() {
        stubFor(get(urlPathEqualTo("/api/exchange-rate"))
                .willReturn(aResponse().withStatus(500)));

        assertThatThrownBy(() -> client.getRate("BRL", "USD"))
                .isInstanceOf(ServiceUnavailableException.class);
    }
}
```

**Go 1.26 (httptest):**
```go
func TestExchangeRateClient(t *testing.T) {
    t.Run("success returns rate", func(t *testing.T) {
        server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            assert.Equal(t, "/api/exchange-rate", r.URL.Path)
            assert.Equal(t, "BRL", r.URL.Query().Get("from"))
            w.Header().Set("Content-Type", "application/json")
            w.WriteHeader(http.StatusOK)
            json.NewEncoder(w).Encode(map[string]any{"rate": 0.20})
        }))
        defer server.Close()

        client := exchange.NewClient(server.URL, 3*time.Second)
        rate, err := client.GetRate(context.Background(), "BRL", "USD")

        require.NoError(t, err)
        assert.InDelta(t, 0.20, rate.Rate, 0.001)
    })

    t.Run("timeout returns error", func(t *testing.T) {
        server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            time.Sleep(5 * time.Second) // simulate slow response
        }))
        defer server.Close()

        client := exchange.NewClient(server.URL, 1*time.Second) // 1s timeout
        _, err := client.GetRate(context.Background(), "BRL", "USD")

        assert.Error(t, err)
        assert.True(t, errors.Is(err, context.DeadlineExceeded))
    })
}
```

**Critérios de aceite:**

- [ ] WireMock / httptest configurado para API externa
- [ ] Cenário de sucesso testado
- [ ] Timeout testado (delay > threshold)
- [ ] Erro 500 testado
- [ ] Resposta malformada testada
- [ ] ≥ 5 testes de HTTP client

---

### Desafio 2.5 — Testes de Banco de Dados (Queries Complexas)

**Contexto:** O extrato de transações da Digital Wallet tem queries complexas com filtros,
paginação, ordenação e aggregation. Essas queries precisam de testes específicos.

**Requisitos:**

- Implementar testes de integração para queries complexas:
  - Extrato paginado: `findByWalletId(walletId, page, size, sort)` — validar offset, limit, total
  - Filtro por tipo: apenas `DEPOSIT` ou `WITHDRAWAL`
  - Filtro por período: `startDate` e `endDate`
  - Saldo calculado: `SELECT SUM(amount) WHERE type = 'DEPOSIT' - SUM(amount) WHERE type = 'WITHDRAWAL'`
  - Unique constraint: `idempotency_key` duplicada → retorna constraint violation
- Cada teste deve:
  - Inserir dados conhecidos antes de executar a query
  - Validar resultado com precisão (valores exatos, não apenas "não vazio")
  - Testar com volume: inserir 50+ registros para validar paginação real

**Java 25:**
```java
@Test
void findByWalletId_withPagination_returnsCorrectPage() {
    // Arrange — inserir 25 transações
    for (int i = 0; i < 25; i++) {
        transactionRepository.save(TransactionBuilder.aTransaction()
                .withWalletId(walletId)
                .withAmount(String.valueOf(10 + i))
                .build());
    }

    // Act — buscar página 2 com 10 por página
    var page = transactionRepository.findByWalletId(walletId, PageRequest.of(1, 10));

    // Assert
    assertThat(page.getContent()).hasSize(10);
    assertThat(page.getTotalElements()).isEqualTo(25);
    assertThat(page.getTotalPages()).isEqualTo(3);
    assertThat(page.getNumber()).isEqualTo(1);
}

@Test
void findByWalletId_filteredByType_returnsOnlyDeposits() {
    transactionRepository.save(TransactionBuilder.aTransaction()
            .withWalletId(walletId).withType(DEPOSIT).build());
    transactionRepository.save(TransactionBuilder.aTransaction()
            .withWalletId(walletId).withType(WITHDRAWAL).build());

    var deposits = transactionRepository.findByWalletIdAndType(walletId, DEPOSIT);

    assertThat(deposits).hasSize(1);
    assertThat(deposits).allSatisfy(tx ->
            assertThat(tx.getType()).isEqualTo(DEPOSIT));
}
```

**Critérios de aceite:**

- [ ] ≥ 6 testes de queries complexas (paginação, filtros, aggregation)
- [ ] Volume de dados realista (50+ registros para paginação)
- [ ] Unique constraints testadas
- [ ] Edge cases testados (página vazia, filtro sem resultado)
- [ ] Banco real via Testcontainers

---

### Desafio 2.6 — Testes Assíncronos (Filas e Eventos)

**Contexto:** A Digital Wallet publica eventos de transação (`TransactionCompleted`,
`TransactionFailed`) para processamento assíncrono. Testar fluxos assíncronos requer
padrões especiais.

**Requisitos:**

- Implementar testes para:
  - Publicação de evento após transação: `TransactionCompletedEvent` contém dados corretos
  - Consumo de evento: consumer processa e atualiza estado
  - Evento com dados inválidos: consumer rejeita (DLQ)
- Usar padrão **poll-and-wait** (não `Thread.sleep()`):

**Java 25 (Awaitility):**
```java
@Test
void deposit_publishesTransactionCompletedEvent() {
    // Arrange
    var request = new DepositRequest(new BigDecimal("100.00"), "Test", UUID.randomUUID());

    // Act
    transactionService.deposit(walletId, request);

    // Assert — aguardar evento assíncrono com timeout
    await().atMost(5, SECONDS)
            .pollInterval(100, MILLISECONDS)
            .untilAsserted(() -> {
                var events = eventCaptor.getCapturedEvents();
                assertThat(events).hasSize(1);
                assertThat(events.get(0))
                        .isInstanceOf(TransactionCompletedEvent.class);
                assertThat(((TransactionCompletedEvent) events.get(0)).amount())
                        .isEqualByComparingTo("100.00");
            });
}
```

**Go 1.26 (eventually pattern):**
```go
func TestDeposit_PublishesEvent(t *testing.T) {
    // Arrange
    eventCh := make(chan model.TransactionEvent, 10)
    publisher := &CapturingPublisher{events: eventCh}
    svc := service.NewTransactionService(walletStore, txStore, publisher, logger)

    // Act
    _, err := svc.Deposit(ctx, walletID, depositInput)
    require.NoError(t, err)

    // Assert — aguardar evento com timeout
    select {
    case event := <-eventCh:
        assert.Equal(t, "TransactionCompleted", event.Type)
        assert.InDelta(t, 100.00, event.Amount, 0.001)
    case <-time.After(5 * time.Second):
        t.Fatal("timeout waiting for TransactionCompleted event")
    }
}
```

**Critérios de aceite:**

- [ ] ≥ 4 testes assíncronos (publicação, consumo, DLQ, timeout)
- [ ] Nenhum `Thread.sleep()` / `time.Sleep()` fixo
- [ ] Padrão poll-and-wait com timeout configurável
- [ ] Eventos com dados corretos validados
- [ ] DLQ testada (evento malformado vai para dead letter)

---

### Desafio 2.7 — Refatoração de Testes Legados

**Contexto:** Projetos reais acumulam testes de baixa qualidade. Este desafio treina a habilidade
de **identificar e refatorar** anti-patterns em testes existentes.

**Requisitos:**

- Dado o seguinte teste "legado", identificar **todos os anti-patterns** e refatorar:

```java
// ❌ ANTES — teste com múltiplos anti-patterns
@Test
void test1() throws Exception {
    // cria usuário
    User u = new User();
    u.setName("Test");
    u.setEmail("test@test.com");
    u.setDocument("12345678900");
    u.setStatus("ACTIVE");
    u.setCreatedAt(Instant.now());
    u.setUpdatedAt(Instant.now());
    User saved = userRepository.save(u);
    assertNotNull(saved);

    // cria wallet
    Wallet w = new Wallet();
    w.setUserId(saved.getId());
    w.setBalance(new BigDecimal(100));
    w.setCurrency("BRL");
    w.setStatus("ACTIVE");
    Wallet savedW = walletRepository.save(w);
    assertNotNull(savedW);

    // deposita
    Thread.sleep(1000); // esperar
    TransactionResponse resp = transactionService.deposit(
        savedW.getId(), new DepositRequest(new BigDecimal(50), "test", UUID.randomUUID()));
    assertTrue(resp != null);
    assertTrue(resp.getStatus().equals("COMPLETED"));

    // verifica saldo
    Wallet updated = walletRepository.findById(savedW.getId()).get();
    assertTrue(updated.getBalance().equals(new BigDecimal(150)));
}
```

- Anti-patterns a identificar:
  1. Nome genérico (`test1`)
  2. Múltiplas ações em um único teste
  3. Criação verbosa de entidades (deveria usar Builder)
  4. `Thread.sleep()` para sincronização
  5. `assertNotNull` / `assertTrue` genéricos
  6. `new BigDecimal(100)` sem escala (deveria ser `"100.00"`)
  7. Sem separação AAA (tudo junto)
  8. Teste testa 3 coisas: save user, save wallet, deposit

- Refatorar em **3 testes separados**, usando builders, asserções expressivas e sem sleep

**Critérios de aceite:**

- [ ] Todos os anti-patterns identificados e documentados
- [ ] Teste legado refatorado em 3+ testes individuais
- [ ] Builders usados para criação de entidades
- [ ] Asserções expressivas (AssertJ / testify)
- [ ] Zero `Thread.sleep()` / `time.Sleep()`
- [ ] Naming convention aplicada
- [ ] Documento: `decisions/02-test-refactoring-guide.md`
