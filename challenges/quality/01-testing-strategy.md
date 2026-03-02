# Level 1 — Estratégia de Testes por Camada

> **Objetivo:** Implementar a estratégia de testes completa para a Digital Wallet — testes unitários,
> de integração, de contrato e E2E — com padrões de escrita, test doubles e naming conventions
> aplicados em Go e Java (multi-framework).

**Referência:** [.docs/QUALITY-ENGINEERING/01-testing-strategy.md](../../.docs/QUALITY-ENGINEERING/01-testing-strategy.md)

---

## Contexto do Domínio

Neste nível, você implementará a **estratégia de testes por camada** da Digital Wallet.
O foco é escrever testes idiomáticos em cada linguagem, aplicando o padrão AAA/Given-When-Then,
naming conventions e a proporção correta da pirâmide de testes.

---

## Desafios

### Desafio 1.1 — Testes Unitários de Domínio (Regras de Negócio)

**Contexto:** A camada de domínio da Digital Wallet contém regras críticas: validação de saldo,
cálculo de limites, verificação de status de wallet e regras de transferência. Esta camada
deve ter a **maior densidade** de testes.

**Requisitos:**

- Implementar testes unitários para **todas as regras de negócio**:
  - Depósito: valor > 0, wallet ativa, atualização de saldo
  - Saque: valor > 0, saldo suficiente, wallet ativa
  - Transferência: wallets diferentes, ambas ativas, saldo suficiente, débito + crédito atômico
  - Validação de documento (CPF/CNPJ): formato, dígitos verificadores
  - Transição de status: `PENDING → COMPLETED`, `PENDING → FAILED`, proibir `COMPLETED → PENDING`
- Seguir convenção de naming: `[unidade]_[cenário]_[resultado]`
- Usar padrão AAA (Arrange-Act-Assert) rigorosamente
- **Zero I/O**: nenhum teste unitário toca banco, rede ou filesystem

**Java 25 (JUnit 5 + AssertJ):**
```java
class WalletTest {

    @Test
    void debit_withSufficientBalance_updatesBalance() {
        // Arrange
        var wallet = WalletBuilder.aWallet()
                .withBalance("100.00")
                .withStatus(WalletStatus.ACTIVE)
                .build();

        // Act
        wallet.debit(new BigDecimal("30.00"));

        // Assert
        assertThat(wallet.getBalance()).isEqualByComparingTo("70.00");
    }

    @Test
    void debit_withInsufficientBalance_throwsInsufficientBalanceException() {
        // Arrange
        var wallet = WalletBuilder.aWallet()
                .withBalance("20.00")
                .build();

        // Act + Assert
        assertThatThrownBy(() -> wallet.debit(new BigDecimal("50.00")))
                .isInstanceOf(InsufficientBalanceException.class)
                .hasMessageContaining("20.00");
    }

    @Test
    void debit_onBlockedWallet_throwsWalletBlockedException() {
        var wallet = WalletBuilder.aWallet()
                .withBalance("100.00")
                .withStatus(WalletStatus.BLOCKED)
                .build();

        assertThatThrownBy(() -> wallet.debit(new BigDecimal("10.00")))
                .isInstanceOf(WalletBlockedException.class);
    }

    @ParameterizedTest
    @ValueSource(strings = {"-1.00", "0.00", "-0.01"})
    void debit_withNonPositiveAmount_throwsValidationException(String amount) {
        var wallet = WalletBuilder.aWallet().withBalance("100.00").build();

        assertThatThrownBy(() -> wallet.debit(new BigDecimal(amount)))
                .isInstanceOf(InvalidAmountException.class);
    }
}
```

**Go 1.26 (table-driven tests + testify):**
```go
func TestWallet_Debit(t *testing.T) {
    tests := []struct {
        name        string
        balance     float64
        status      WalletStatus
        amount      float64
        wantBalance float64
        wantErr     error
    }{
        {
            name:        "sufficient balance updates balance",
            balance:     100.00,
            status:      WalletStatusActive,
            amount:      30.00,
            wantBalance: 70.00,
        },
        {
            name:    "insufficient balance returns error",
            balance: 20.00,
            status:  WalletStatusActive,
            amount:  50.00,
            wantErr: ErrInsufficientBalance,
        },
        {
            name:    "blocked wallet returns error",
            balance: 100.00,
            status:  WalletStatusBlocked,
            amount:  10.00,
            wantErr: ErrWalletBlocked,
        },
        {
            name:    "non-positive amount returns error",
            balance: 100.00,
            status:  WalletStatusActive,
            amount:  -1.00,
            wantErr: ErrInvalidAmount,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            wallet := NewTestWallet(
                WithBalance(tt.balance),
                WithWalletStatus(tt.status),
            )

            err := wallet.Debit(tt.amount)

            if tt.wantErr != nil {
                assert.ErrorIs(t, err, tt.wantErr)
            } else {
                require.NoError(t, err)
                assert.InDelta(t, tt.wantBalance, wallet.Balance, 0.001)
            }
        })
    }
}
```

**Critérios de aceite:**

- [ ] ≥ 15 testes unitários cobrindo todas as regras de negócio (Java e Go)
- [ ] Testes seguem padrão AAA rigorosamente
- [ ] Naming convention consistente em todo o projeto
- [ ] Zero I/O em testes unitários
- [ ] Testes parametrizados para validações com múltiplos cenários
- [ ] Cobertura ≥ 90% na camada de domínio
- [ ] Execução total < 2s

---

### Desafio 1.2 — Testes Unitários de Service (com Test Doubles)

**Contexto:** A camada de service orquestra operações: chama repositories, publica eventos,
coordena transações. Para testá-la unitariamente, é necessário usar **test doubles** (mocks, stubs, spies).

**Requisitos:**

- Implementar testes unitários para `TransactionService` usando mocks:
  - Depósito bem-sucedido: verifica que repository.save() é chamado, saldo atualizado
  - Saque com saldo insuficiente: verifica que repository.save() **NÃO** é chamado
  - Transferência: verifica que ambas as wallets são atualizadas
  - Idempotência: se `idempotency_key` já existe, retorna transação existente sem reprocessar
  - Wallet não encontrada: retorna erro 404
- Documentar quando usar cada tipo de test double:

| Tipo | Quando usar | Exemplo na Digital Wallet |
|------|-------------|--------------------------|
| **Mock** | Verificar chamadas de método (interação) | `walletRepository.save()` foi chamado 1 vez |
| **Stub** | Retornar dados pré-definidos (estado) | `walletRepository.findById()` retorna wallet com saldo 100 |
| **Spy** | Capturar argumentos passados | Capturar o objeto `Transaction` salvo para verificar campos |
| **Fake** | Implementação simplificada funcional | `InMemoryWalletRepository` para testes de integração leve |

**Java 25 (Mockito):**
```java
@ExtendWith(MockitoExtension.class)
class TransactionServiceTest {

    @Mock WalletRepository walletRepository;
    @Mock TransactionRepository transactionRepository;
    @Mock EventPublisher eventPublisher;
    @InjectMocks TransactionService transactionService;

    @Test
    void deposit_withValidAmount_savesTransactionAndUpdatesBalance() {
        // Arrange
        var wallet = WalletBuilder.aWallet().withId(WALLET_ID).withBalance("100.00").build();
        when(walletRepository.findById(WALLET_ID)).thenReturn(Optional.of(wallet));
        when(transactionRepository.existsByIdempotencyKey(any())).thenReturn(false);

        var request = new DepositRequest(new BigDecimal("50.00"), "Test deposit", UUID.randomUUID());

        // Act
        var result = transactionService.deposit(WALLET_ID, request);

        // Assert
        assertThat(result.status()).isEqualTo(TransactionStatus.COMPLETED);
        verify(walletRepository).save(argThat(w ->
                w.getBalance().compareTo(new BigDecimal("150.00")) == 0));
        verify(transactionRepository).save(any(Transaction.class));
        verify(eventPublisher).publish(any(TransactionCompletedEvent.class));
    }

    @Test
    void deposit_withExistingIdempotencyKey_returnsExistingTransaction() {
        // Arrange
        var idempotencyKey = UUID.randomUUID();
        var existingTx = TransactionBuilder.aTransaction().withStatus(COMPLETED).build();
        when(transactionRepository.existsByIdempotencyKey(idempotencyKey)).thenReturn(true);
        when(transactionRepository.findByIdempotencyKey(idempotencyKey)).thenReturn(Optional.of(existingTx));

        // Act
        var result = transactionService.deposit(WALLET_ID,
                new DepositRequest(new BigDecimal("50.00"), "Dup", idempotencyKey));

        // Assert — não salva novamente
        verify(walletRepository, never()).save(any());
        verify(transactionRepository, never()).save(any());
        assertThat(result.id()).isEqualTo(existingTx.getId());
    }
}
```

**Go 1.26 (gomock):**
```go
func TestTransactionService_Deposit(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    walletStore := mocks.NewMockWalletStore(ctrl)
    txStore := mocks.NewMockTransactionStore(ctrl)
    publisher := mocks.NewMockEventPublisher(ctrl)
    svc := service.NewTransactionService(walletStore, txStore, publisher, zap.NewNop())

    t.Run("valid deposit saves transaction and updates balance", func(t *testing.T) {
        wallet := NewTestWallet(WithBalance(100.00), WithID(walletID))
        walletStore.EXPECT().FindByID(gomock.Any(), walletID).Return(&wallet, nil)
        txStore.EXPECT().ExistsByIdempotencyKey(gomock.Any(), gomock.Any()).Return(false, nil)
        walletStore.EXPECT().Save(gomock.Any(), gomock.Any()).Return(nil)
        txStore.EXPECT().Save(gomock.Any(), gomock.Any()).Return(nil)
        publisher.EXPECT().Publish(gomock.Any(), gomock.Any()).Return(nil)

        input := model.DepositInput{Amount: 50.00, Description: "Test", IdempotencyKey: uuid.New()}
        result, err := svc.Deposit(context.Background(), walletID, input)

        require.NoError(t, err)
        assert.Equal(t, model.StatusCompleted, result.Status)
    })
}
```

**Critérios de aceite:**

- [ ] ≥ 10 testes unitários de service com mocks (Java e Go)
- [ ] Cenários de sucesso, erro e edge cases cobertos
- [ ] Idempotência testada (duplicate request retorna existente)
- [ ] Verificação de interações (verify/EXPECT) quando relevante
- [ ] Tabela de test doubles documentada com exemplos
- [ ] Nenhum mock excessivo (não mockar value objects/DTOs)

---

### Desafio 1.3 — Testes de Integração com Testcontainers

**Contexto:** Testes de integração validam que o código funciona com infraestrutura real.
Para a Digital Wallet, isso significa testar com PostgreSQL real (via Testcontainers).

**Requisitos:**

- Implementar testes de integração para a camada de persistência:
  - `WalletRepository`: save, findById, findByUserId, update balance
  - `TransactionRepository`: save, findByWalletId (paginado), findByIdempotencyKey
  - `UserRepository`: save, findById, findByEmail (unique constraint)
- Cada teste deve:
  - Usar container PostgreSQL efêmero (Testcontainers)
  - Rodar migrations antes dos testes
  - Criar e limpar seus próprios dados (isolamento)
  - Validar queries, mapeamento e constraints

**Java 25 (Spring Boot + Testcontainers):**
```java
@SpringBootTest(webEnvironment = WebEnvironment.NONE)
@Testcontainers
class WalletRepositoryIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("wallet_test");

    @Autowired
    WalletRepository walletRepository;

    @Autowired
    UserRepository userRepository;

    @Test
    void save_validWallet_persistsAndRetrievesCorrectly() {
        // Arrange
        var user = userRepository.save(UserBuilder.aUser().build());
        var wallet = WalletBuilder.aWallet().withUserId(user.getId()).build();

        // Act
        var saved = walletRepository.save(wallet);
        var found = walletRepository.findById(saved.getId());

        // Assert
        assertThat(found).isPresent();
        assertThat(found.get().getBalance()).isEqualByComparingTo(wallet.getBalance());
        assertThat(found.get().getCurrency()).isEqualTo(wallet.getCurrency());
    }

    @Test
    void findByUserId_multipleWallets_returnsAll() {
        var user = userRepository.save(UserBuilder.aUser().build());
        walletRepository.save(WalletBuilder.aWallet().withUserId(user.getId()).withCurrency("BRL").build());
        walletRepository.save(WalletBuilder.aWallet().withUserId(user.getId()).withCurrency("USD").build());

        var wallets = walletRepository.findByUserId(user.getId());

        assertThat(wallets).hasSize(2);
        assertThat(wallets).extracting(Wallet::getCurrency)
                .containsExactlyInAnyOrder("BRL", "USD");
    }
}
```

**Go 1.26 (testcontainers-go):**
```go
func TestWalletStore_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    ctx := context.Background()
    pgContainer, connStr := setupPostgres(t, ctx) // helper com testcontainers
    defer pgContainer.Terminate(ctx)

    db := connectAndMigrate(t, connStr)
    store := store.NewWalletStore(db)
    userStore := store.NewUserStore(db)

    t.Run("save and retrieve wallet", func(t *testing.T) {
        user := createTestUser(t, userStore)
        wallet := model.Wallet{
            ID:       uuid.New(),
            UserID:   user.ID,
            Currency: "BRL",
            Balance:  100.00,
            Status:   model.WalletStatusActive,
        }

        err := store.Save(ctx, &wallet)
        require.NoError(t, err)

        found, err := store.FindByID(ctx, wallet.ID)
        require.NoError(t, err)
        assert.Equal(t, wallet.Currency, found.Currency)
        assert.InDelta(t, wallet.Balance, found.Balance, 0.001)
    })
}
```

**Critérios de aceite:**

- [ ] ≥ 8 testes de integração com PostgreSQL real (Java e Go)
- [ ] Testcontainers configurado e funcional
- [ ] Migrations rodam antes dos testes
- [ ] Cada teste é isolado (sem dependência de ordem)
- [ ] Unique constraints testadas (email duplicado → erro)
- [ ] Paginação testada (offset/limit)
- [ ] Testes rodam com `go test -run Integration` / `./mvnw verify -Pintegration`

---

### Desafio 1.4 — Testes de API (HTTP Layer)

**Contexto:** A camada HTTP precisa ser testada para validar: deserialização de requests,
validação de entrada, status codes, headers e formato da resposta (ProblemDetail).

**Requisitos:**

- Implementar testes HTTP para todos os endpoints da Digital Wallet:
  - `POST /api/v1/users` → 201 com body, 400 com ProblemDetail, 409 para duplicado
  - `POST /api/v1/wallets/{id}/transactions/deposit` → 201, 400, 404, 422
  - `GET /api/v1/wallets/{id}/transactions` → 200 com paginação
- Testar formato de erro RFC 9457 (ProblemDetail):

```json
{
    "type": "INSUFFICIENT_BALANCE",
    "title": "Insufficient balance for withdrawal",
    "status": 422,
    "detail": "Wallet abc-123 has balance 50.00, withdrawal amount 100.00",
    "instance": "/api/v1/wallets/abc-123/transactions/withdrawal"
}
```

**Java 25 (Spring Boot — WebTestClient / MockMvc):**
```java
@WebMvcTest(TransactionController.class)
class TransactionControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean TransactionService transactionService;

    @Test
    void deposit_withValidRequest_returns201() throws Exception {
        var response = TransactionResponse.builder()
                .id(UUID.randomUUID()).status(COMPLETED).build();
        when(transactionService.deposit(any(), any())).thenReturn(response);

        mockMvc.perform(post("/api/v1/wallets/{id}/transactions/deposit", WALLET_ID)
                        .contentType(APPLICATION_JSON)
                        .content("""
                            {
                                "amount": 100.50,
                                "description": "Test deposit",
                                "idempotency_key": "550e8400-e29b-41d4-a716-446655440000"
                            }
                            """))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").isNotEmpty())
                .andExpect(jsonPath("$.status").value("COMPLETED"));
    }

    @Test
    void deposit_withNegativeAmount_returns400WithProblemDetail() throws Exception {
        mockMvc.perform(post("/api/v1/wallets/{id}/transactions/deposit", WALLET_ID)
                        .contentType(APPLICATION_JSON)
                        .content("""
                            { "amount": -10.00, "description": "Invalid", "idempotency_key": "..." }
                            """))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.type").value("VALIDATION_ERROR"))
                .andExpect(jsonPath("$.status").value(400));
    }
}
```

**Go 1.26 (httptest):**
```go
func TestTransactionHandler_Deposit(t *testing.T) {
    ctrl := gomock.NewController(t)
    svc := mocks.NewMockTransactionService(ctrl)
    handler := handler.NewTransactionHandler(svc)
    router := setupTestRouter(handler)

    t.Run("valid deposit returns 201", func(t *testing.T) {
        svc.EXPECT().Deposit(gomock.Any(), gomock.Any(), gomock.Any()).
            Return(&model.TransactionOutput{ID: uuid.New(), Status: "COMPLETED"}, nil)

        body := `{"amount": 100.50, "description": "Test", "idempotency_key": "550e8400-..."}`
        req := httptest.NewRequest(http.MethodPost,
            "/api/v1/wallets/"+walletID.String()+"/transactions/deposit",
            strings.NewReader(body))
        req.Header.Set("Content-Type", "application/json")
        rec := httptest.NewRecorder()

        router.ServeHTTP(rec, req)

        assert.Equal(t, http.StatusCreated, rec.Code)
        var resp map[string]any
        json.Unmarshal(rec.Body.Bytes(), &resp)
        assert.Equal(t, "COMPLETED", resp["status"])
    })
}
```

**Critérios de aceite:**

- [ ] ≥ 12 testes HTTP cobrindo todos os endpoints
- [ ] Status codes validados (201, 400, 404, 409, 422)
- [ ] Formato ProblemDetail (RFC 9457) validado em erros
- [ ] Paginação validada (content, totalElements, totalPages)
- [ ] Headers Content-Type validados
- [ ] Validação de input testada (campos obrigatórios, formatos)

---

### Desafio 1.5 — Testes de Contrato

**Contexto:** A Digital Wallet expõe APIs consumidas por outros serviços (mobile, web, parceiros).
Testes de contrato garantem que **mudanças na API não quebram consumidores**.

**Requisitos:**

- Implementar testes de contrato para os endpoints críticos:
  - `POST /api/v1/wallets/{id}/transactions/deposit` — schema de request e response
  - `GET /api/v1/wallets/{id}/transactions` — schema de resposta paginada
  - Evento `TransactionCompleted` — schema do evento publicado
- Usar **consumer-driven contracts** (Pact ou Spring Cloud Contract):
  - Consumidor define expectativa (quais campos espera na resposta)
  - Produtor valida que atende a expectativa
- Validar que **breaking changes** são detectadas:
  - Remover campo obrigatório da resposta → teste falha
  - Alterar tipo de campo → teste falha
  - Adicionar campo opcional → teste passa (backward compatible)

**Java 25 (Spring Cloud Contract / Pact):**
```java
// Contract definition (Groovy DSL ou YAML)
// contracts/deposit-success.groovy
Contract.make {
    description "Should process deposit successfully"
    request {
        method POST()
        url "/api/v1/wallets/550e8400-e29b-41d4-a716-446655440000/transactions/deposit"
        headers { contentType applicationJson() }
        body([
            amount: 100.50,
            description: "Test deposit",
            idempotency_key: "key-123"
        ])
    }
    response {
        status 201
        headers { contentType applicationJson() }
        body([
            id: $(anyUuid()),
            wallet_id: "550e8400-e29b-41d4-a716-446655440000",
            type: "DEPOSIT",
            amount: 100.50,
            status: "COMPLETED",
            created_at: $(anyIso8601DateTime())
        ])
    }
}
```

**Go 1.26 (pact-go):**
```go
func TestDepositContract_ConsumerSide(t *testing.T) {
    pact := dsl.Pact{
        Consumer: "mobile-app",
        Provider: "wallet-service",
    }
    defer pact.Teardown()

    pact.AddInteraction().
        Given("wallet exists with balance 100.00").
        UponReceiving("a deposit request").
        WithRequest(dsl.Request{
            Method: "POST",
            Path:   dsl.String("/api/v1/wallets/abc-123/transactions/deposit"),
            Headers: dsl.MapMatcher{"Content-Type": dsl.String("application/json")},
            Body:    map[string]interface{}{"amount": 50.00, "description": "Test"},
        }).
        WillRespondWith(dsl.Response{
            Status:  201,
            Headers: dsl.MapMatcher{"Content-Type": dsl.String("application/json")},
            Body: dsl.Like(map[string]interface{}{
                "id":     dsl.Like("uuid"),
                "type":   "DEPOSIT",
                "amount": dsl.Like(50.00),
                "status": "COMPLETED",
            }),
        })

    err := pact.Verify(func() error {
        // Chamar o endpoint real do consumer
        return nil
    })
    assert.NoError(t, err)
}
```

**Critérios de aceite:**

- [ ] ≥ 3 contratos definidos (deposit, transactions list, evento)
- [ ] Consumer-driven contracts implementados
- [ ] Breaking change detection testada (remover campo → falha)
- [ ] Backward compatibility verificada (adicionar campo → passa)
- [ ] Contratos versionados no repositório

---

### Desafio 1.6 — Testes E2E / Smoke

**Contexto:** Testes E2E validam o fluxo completo da Digital Wallet de ponta a ponta,
com todas as integrações reais. Devem ser **poucos** (2-5) e cobrir apenas happy paths críticos.

**Requisitos:**

- Implementar **no máximo 5** cenários E2E:
  1. Criar usuário → criar wallet → depositar → verificar saldo
  2. Depositar → sacar → verificar saldo final
  3. Criar 2 wallets → depositar na w1 → transferir para w2 → verificar saldos de ambas
  4. Depósito com idempotency_key duplicada → retorna mesma transação
  5. Smoke test: health check → readiness → criar usuário (end-to-end mínimo)
- Toda a stack deve estar rodando (app + PostgreSQL + Redis, se aplicável)
- Setup/teardown automático (criar dados antes, limpar depois)
- Timeout máximo: 60 segundos por teste

**Critérios de aceite:**

- [ ] ≤ 5 testes E2E implementados
- [ ] Fluxo de transferência completo testado (depósito → transferência → extrato)
- [ ] Docker Compose configurado para subir toda a stack
- [ ] Tests rodam em < 60s total
- [ ] Setup/teardown automático (sem dados residuais)
- [ ] Testes rodam via `make test-e2e` / `./mvnw verify -Pe2e`

---

### Desafio 1.7 — Convenções e Qualidade da Suíte de Testes

**Contexto:** A suíte de testes precisa ser organizada, confiável e rápida. Aplique convenções
e meça a qualidade dos próprios testes.

**Requisitos:**

- Aplicar **tags/categorias** em todos os testes:

```java
// Java — JUnit 5 tags
@Tag("unit") class WalletTest { }
@Tag("integration") class WalletRepositoryIT { }
@Tag("contract") class DepositContractTest { }
@Tag("e2e") class TransferFlowE2ETest { }
```

```go
// Go — build tags
//go:build unit
//go:build integration
//go:build e2e
```

- Configurar execução seletiva:
  - `go test -tags=unit ./...` / `./mvnw test -Dgroups=unit`
  - `go test -tags=integration ./...` / `./mvnw verify -Dgroups=integration`
- Medir e documentar métricas da suíte:

| Métrica | Target | Como medir |
|---------|--------|------------|
| Tempo total (unit) | < 5s | `go test -count=1 ./... \| tail -1` |
| Tempo total (integration) | < 60s | Testcontainers startup + testes |
| Flaky rate | 0% | Rodar 5x consecutivas, zero falhas |
| Cobertura service layer | ≥ 80% | `go tool cover` / JaCoCo |
| Cobertura geral | ≥ 70% | `go tool cover` / JaCoCo |
| Ratio unit/integration | ≥ 3:1 | Contar testes por tag |

- Identificar e eliminar **anti-patterns** existentes:
  - [ ] Nenhum `Thread.sleep()` / `time.Sleep()` para sincronização
  - [ ] Nenhum teste depende de ordem de execução
  - [ ] Nenhum teste usa dados hardcoded do environment
  - [ ] Nenhum teste ignora erros (`@Disabled` sem justificativa)

**Critérios de aceite:**

- [ ] Todos os testes categorizados com tags
- [ ] Execução seletiva por tag funcional
- [ ] Métricas da suíte documentadas e dentro do target
- [ ] Zero flaky tests (5 execuções consecutivas sem falha)
- [ ] Anti-patterns checklist verificado
- [ ] Relatório de cobertura HTML gerado
