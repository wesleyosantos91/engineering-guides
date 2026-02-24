# Test Automation Patterns — Receitas e Padrões Reutilizáveis

> **Versão:** 1.0
> **Última atualização:** 2026-02-24
> **Público-alvo:** Times de engenharia (back-end, front-end, plataforma, QA)
> **Documentos relacionados:** [Testing Strategy](testing-strategy.md) · [Performance Testing](performance-testing.md)

---

## Sumário

1. [Visão Geral](#1-visão-geral)
2. [Padrões de Organização de Testes](#2-padrões-de-organização-de-testes)
3. [Padrões de Criação de Dados (Test Data)](#3-padrões-de-criação-de-dados-test-data)
4. [Padrões de Setup e Cleanup](#4-padrões-de-setup-e-cleanup)
5. [Padrões de Asserção](#5-padrões-de-asserção)
6. [Padrões para Testes de Integração](#6-padrões-para-testes-de-integração)
7. [Padrões para Testes de API](#7-padrões-para-testes-de-api)
8. [Padrões para Testes Assíncronos](#8-padrões-para-testes-assíncronos)
9. [Padrões para Testes de Banco de Dados](#9-padrões-para-testes-de-banco-de-dados)
10. [Padrões para Refatoração de Testes](#10-padrões-para-refatoração-de-testes)
11. [Receitas por Linguagem](#11-receitas-por-linguagem)

---

## 1. Visão Geral

Este documento contém **padrões reutilizáveis** e **receitas prontas** para automação de testes. O objetivo é fornecer implementações concretas que os times possam adaptar, reduzindo tempo de decisão e aumentando consistência.

### 1.1 Quando Usar Este Documento

| Situação                                       | Seção recomendada                    |
|------------------------------------------------|--------------------------------------|
| Começando testes em um novo serviço            | §2 Organização + §3 Test Data        |
| Testes de integração com banco de dados        | §6 + §9                              |
| Testes de API REST                             | §7                                   |
| Testes com filas/eventos                       | §8                                   |
| Refatorando testes legados                     | §10                                  |
| Buscando exemplos em linguagem específica      | §11                                  |

---

## 2. Padrões de Organização de Testes

### 2.1 Estrutura de Diretórios

```
src/
├── main/
│   └── ... (código de produção)
└── test/
    ├── unit/                    # Testes unitários (sem I/O)
    │   ├── domain/              # Espelho da estrutura de domínio
    │   └── application/         # Espelho da estrutura de aplicação
    ├── integration/             # Testes de integração (com I/O)
    │   ├── repository/          # Testes de persistência
    │   ├── messaging/           # Testes de fila/evento
    │   └── http/                # Testes de API HTTP
    ├── contract/                # Testes de contrato
    ├── e2e/                     # Testes end-to-end
    ├── fixtures/                # Dados de teste compartilhados
    └── support/                 # Builders, factories, helpers
        ├── builders/
        ├── factories/
        └── testcontainers/
```

### 2.2 Convenção de Nomenclatura

| Tipo de teste   | Sufixo / Padrão                  | Exemplo                                |
|-----------------|----------------------------------|----------------------------------------|
| Unitário        | `*Test` / `*_test.go`           | `CalculadoraDescontoTest`              |
| Integração      | `*IntegrationTest` / `*IT`      | `PedidoRepositoryIntegrationTest`      |
| Contrato        | `*ContractTest` / `*Pact`       | `PedidoCriadoContractTest`             |
| E2E             | `*E2ETest`                       | `FluxoCompraE2ETest`                   |

### 2.3 Tags / Categorias

```java
// JUnit 5 — Categorização por tag
@Tag("unit")
class CalculadoraDescontoTest { ... }

@Tag("integration")
@Tag("database")
class PedidoRepositoryIT { ... }

@Tag("contract")
class PedidoCriadoContractTest { ... }
```

```go
// Go — Build tags
//go:build integration
package repository_test
```

```javascript
// Jest — projects no jest.config.js
module.exports = {
  projects: [
    { displayName: 'unit', testMatch: ['<rootDir>/src/**/*.test.ts'] },
    { displayName: 'integration', testMatch: ['<rootDir>/src/**/*.integration.test.ts'] },
  ],
};
```

---

## 3. Padrões de Criação de Dados (Test Data)

### 3.1 Builder Pattern

> **Propósito:** Criar objetos complexos com valores default sensatos, permitindo sobrescrever apenas o que importa para cada cenário.

#### Java

```java
public class PedidoBuilder {
    private String id = UUID.randomUUID().toString();
    private String clienteId = "cliente-default";
    private BigDecimal valorTotal = new BigDecimal("100.00");
    private StatusPedido status = StatusPedido.PENDENTE;
    private List<ItemPedido> itens = List.of(
        new ItemPedido("SKU-DEFAULT", 1, new BigDecimal("100.00"))
    );
    private Instant criadoEm = Instant.now();

    public static PedidoBuilder umPedido() {
        return new PedidoBuilder();
    }

    public PedidoBuilder comCliente(String clienteId) {
        this.clienteId = clienteId;
        return this;
    }

    public PedidoBuilder comValor(String valor) {
        this.valorTotal = new BigDecimal(valor);
        return this;
    }

    public PedidoBuilder comStatus(StatusPedido status) {
        this.status = status;
        return this;
    }

    public PedidoBuilder comItens(List<ItemPedido> itens) {
        this.itens = itens;
        return this;
    }

    public Pedido build() {
        return new Pedido(id, clienteId, valorTotal, status, itens, criadoEm);
    }
}

// Uso no teste
var pedido = PedidoBuilder.umPedido()
    .comCliente("cliente-vip")
    .comValor("500.00")
    .build();
```

#### TypeScript

```typescript
class OrderBuilder {
  private order: Partial<Order> = {
    id: randomUUID(),
    customerId: 'default-customer',
    totalAmount: 100.00,
    status: 'PENDING',
    items: [{ sku: 'SKU-DEFAULT', quantity: 1, unitPrice: 100.00 }],
    createdAt: new Date(),
  };

  static anOrder(): OrderBuilder {
    return new OrderBuilder();
  }

  withCustomer(customerId: string): this {
    this.order.customerId = customerId;
    return this;
  }

  withAmount(amount: number): this {
    this.order.totalAmount = amount;
    return this;
  }

  withStatus(status: OrderStatus): this {
    this.order.status = status;
    return this;
  }

  build(): Order {
    return this.order as Order;
  }
}

// Uso no teste
const order = OrderBuilder.anOrder()
  .withCustomer('vip-customer')
  .withAmount(500.00)
  .build();
```

#### Go

```go
type OrderBuilder struct {
    order Order
}

func AnOrder() *OrderBuilder {
    return &OrderBuilder{
        order: Order{
            ID:          uuid.New().String(),
            CustomerID:  "default-customer",
            TotalAmount: 100.00,
            Status:      StatusPending,
            Items:       []OrderItem{{SKU: "SKU-DEFAULT", Quantity: 1, UnitPrice: 100.00}},
            CreatedAt:   time.Now(),
        },
    }
}

func (b *OrderBuilder) WithCustomer(customerID string) *OrderBuilder {
    b.order.CustomerID = customerID
    return b
}

func (b *OrderBuilder) WithAmount(amount float64) *OrderBuilder {
    b.order.TotalAmount = amount
    return b
}

func (b *OrderBuilder) Build() Order {
    return b.order
}

// Uso no teste
order := AnOrder().WithCustomer("vip-customer").WithAmount(500.00).Build()
```

### 3.2 Factory Pattern (Funções simples)

Para objetos mais simples, uma função factory pode ser suficiente:

```java
// Java
static Pedido pedidoPendente() {
    return new Pedido("cli-1", BigDecimal.TEN, StatusPedido.PENDENTE);
}

static Pedido pedidoPago(String clienteId) {
    return new Pedido(clienteId, BigDecimal.TEN, StatusPedido.PAGO);
}
```

```typescript
// TypeScript
const pendingOrder = (overrides?: Partial<Order>): Order => ({
  id: randomUUID(),
  customerId: 'default-customer',
  totalAmount: 100.00,
  status: 'PENDING',
  createdAt: new Date(),
  ...overrides,
});
```

```go
// Go
func PendingOrder(opts ...func(*Order)) Order {
    o := Order{ID: uuid.New().String(), Status: StatusPending, TotalAmount: 100.00}
    for _, opt := range opts {
        opt(&o)
    }
    return o
}

// Uso: order := PendingOrder(func(o *Order) { o.CustomerID = "vip" })
```

### 3.3 Quando usar Builder vs Factory

| Situação                                | Usar                    |
|-----------------------------------------|-------------------------|
| Objeto com muitos campos opcionais      | Builder                 |
| Objeto simples (< 5 campos)           | Factory function         |
| Muitas variações do mesmo objeto        | Builder                 |
| Apenas 2-3 variações fixas              | Factory functions nomeadas |
| Precisão em quais campos são relevantes | Builder (explicita intent) |

---

## 4. Padrões de Setup e Cleanup

### 4.1 Test Fixture Pattern

```java
// JUnit 5 — Lifecycle hooks
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class PedidoRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @BeforeAll
    static void setupInfra() {
        // Configurar datasource com postgres.getJdbcUrl()
    }

    @BeforeEach
    void cleanDatabase() {
        // Limpar tabelas relevantes antes de cada teste
        jdbcTemplate.execute("TRUNCATE TABLE pedidos CASCADE");
    }

    @AfterAll
    static void teardown() {
        // Container é parado automaticamente pelo Testcontainers
    }
}
```

### 4.2 Transaction Rollback Pattern

```java
// Spring Boot — cada teste roda em transação que é revertida automaticamente
@SpringBootTest
@Transactional  // Reverte após cada teste
class PedidoRepositoryIT {

    @Test
    void salvar_pedidoValido_persisteCorretamente() {
        var pedido = PedidoBuilder.umPedido().build();
        repository.save(pedido);
        // Asserts...
        // Não precisa limpar — transação será revertida
    }
}
```

### 4.3 Container per Suite Pattern

```java
// Testcontainers com container compartilhado entre testes da mesma suíte
abstract class AbstractDatabaseIT {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withInitScript("schema.sql");
}

class PedidoRepositoryIT extends AbstractDatabaseIT {
    // Herda o container compartilhado
}

class ClienteRepositoryIT extends AbstractDatabaseIT {
    // Mesmo container, schema diferente se necessário
}
```

---

## 5. Padrões de Asserção

### 5.1 Fluent Assertions

```java
// Java — AssertJ (preferência sobre assertEquals)
assertThat(pedido.getStatus()).isEqualTo(StatusPedido.PAGO);
assertThat(pedido.getItens()).hasSize(3);
assertThat(pedido.getValorTotal()).isEqualByComparingTo("150.00");
assertThat(pedido.getItens())
    .extracting(ItemPedido::getSku)
    .containsExactlyInAnyOrder("SKU-1", "SKU-2", "SKU-3");
```

```typescript
// TypeScript — Jest expectations
expect(order.status).toBe('PAID');
expect(order.items).toHaveLength(3);
expect(order.items.map(i => i.sku)).toEqual(
  expect.arrayContaining(['SKU-1', 'SKU-2', 'SKU-3'])
);
```

```go
// Go — testify assertions
assert.Equal(t, StatusPaid, order.Status)
assert.Len(t, order.Items, 3)
require.NoError(t, err) // fail fast se erro
```

### 5.2 Custom Assertions

```java
// Java — assertion customizada para domínio
public class PedidoAssert extends AbstractAssert<PedidoAssert, Pedido> {

    public PedidoAssert(Pedido actual) {
        super(actual, PedidoAssert.class);
    }

    public static PedidoAssert assertThat(Pedido actual) {
        return new PedidoAssert(actual);
    }

    public PedidoAssert estaPendente() {
        isNotNull();
        if (actual.getStatus() != StatusPedido.PENDENTE) {
            failWithMessage("Esperava pedido PENDENTE, mas era %s", actual.getStatus());
        }
        return this;
    }

    public PedidoAssert temValorTotal(String valor) {
        isNotNull();
        if (actual.getValorTotal().compareTo(new BigDecimal(valor)) != 0) {
            failWithMessage("Esperava valor %s, mas era %s", valor, actual.getValorTotal());
        }
        return this;
    }
}

// Uso
PedidoAssert.assertThat(pedido).estaPendente().temValorTotal("150.00");
```

### 5.3 Assertion Patterns — O que verificar

| Cenário                         | O que assert                                     | Anti-pattern                        |
|---------------------------------|--------------------------------------------------|-------------------------------------|
| Operação de criação             | ID gerado, campos persistidos, status inicial    | Verificar apenas `assertNotNull`    |
| Operação de atualização         | Campo alterado + campos inalterados              | Verificar apenas o campo alterado   |
| Operação de erro                | Tipo da exceção, código de erro, mensagem útil   | Verificar apenas que "lançou algo"  |
| Listagem/busca                  | Tamanho, conteúdo, ordenação                     | Verificar apenas `isNotEmpty`       |
| Efeito colateral                | Evento publicado, notificação enviada, log gerado| Não verificar efeitos colaterais    |

---

## 6. Padrões para Testes de Integração

### 6.1 Testcontainers — Padrão Base

```java
// Java — Testcontainers com Spring Boot 3.x
@SpringBootTest
@Testcontainers
class PedidoRepositoryIT {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    @ServiceConnection
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @Autowired
    private PedidoRepository repository;

    @Test
    void buscarPorCliente_comPedidosExistentes_retornaOrdenadoPorData() {
        // Arrange
        repository.save(PedidoBuilder.umPedido().comCliente("cli-1").comData("2026-01-15").build());
        repository.save(PedidoBuilder.umPedido().comCliente("cli-1").comData("2026-01-10").build());
        repository.save(PedidoBuilder.umPedido().comCliente("cli-2").build()); // outro cliente

        // Act
        var pedidos = repository.buscarPorCliente("cli-1");

        // Assert
        assertThat(pedidos).hasSize(2);
        assertThat(pedidos.get(0).getCriadoEm()).isAfter(pedidos.get(1).getCriadoEm());
    }
}
```

```go
// Go — Testcontainers
func TestOrderRepository_SaveAndFind(t *testing.T) {
    ctx := context.Background()

    container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image:        "postgres:16-alpine",
            ExposedPorts: []string{"5432/tcp"},
            Env:          map[string]string{"POSTGRES_PASSWORD": "test", "POSTGRES_DB": "testdb"},
            WaitingFor:   wait.ForListeningPort("5432/tcp"),
        },
        Started: true,
    })
    require.NoError(t, err)
    defer container.Terminate(ctx)

    host, _ := container.Host(ctx)
    port, _ := container.MappedPort(ctx, "5432")
    dsn := fmt.Sprintf("postgres://postgres:test@%s:%s/testdb?sslmode=disable", host, port.Port())

    repo := NewOrderRepository(dsn)

    order := AnOrder().WithCustomer("cust-1").Build()
    err = repo.Save(ctx, &order)
    require.NoError(t, err)

    found, err := repo.FindByID(ctx, order.ID)
    require.NoError(t, err)
    assert.Equal(t, "cust-1", found.CustomerID)
}
```

### 6.2 WireMock — Mock de APIs Externas

```java
// Java — WireMock para simular API de pagamento
@WireMockTest(httpPort = 8089)
class PaymentClientIT {

    @Test
    void processarPagamento_apiRetorna201_retornaSucesso() {
        // Arrange — configurar stub
        stubFor(post(urlEqualTo("/v1/payments"))
            .willReturn(aResponse()
                .withStatus(201)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"paymentId":"pay-123","status":"APPROVED"}
                """)
            ));

        var client = new PaymentClient("http://localhost:8089");

        // Act
        var resultado = client.processar(new PaymentRequest("100.00", "BRL"));

        // Assert
        assertThat(resultado.getStatus()).isEqualTo("APPROVED");
    }

    @Test
    void processarPagamento_apiRetorna503_lancarExcecaoComRetry() {
        // Arrange — simular falha temporária + sucesso no retry
        stubFor(post(urlEqualTo("/v1/payments"))
            .inScenario("Retry")
            .whenScenarioStateIs(Scenario.STARTED)
            .willReturn(aResponse().withStatus(503))
            .willSetStateTo("RECOVERED"));

        stubFor(post(urlEqualTo("/v1/payments"))
            .inScenario("Retry")
            .whenScenarioStateIs("RECOVERED")
            .willReturn(aResponse()
                .withStatus(201)
                .withBody("""{"paymentId":"pay-123","status":"APPROVED"}""")
            ));

        var client = new PaymentClient("http://localhost:8089");

        // Act
        var resultado = client.processar(new PaymentRequest("100.00", "BRL"));

        // Assert
        assertThat(resultado.getStatus()).isEqualTo("APPROVED");
        verify(2, postRequestedFor(urlEqualTo("/v1/payments")));
    }
}
```

---

## 7. Padrões para Testes de API

### 7.1 REST API Test Pattern

```java
// Java — Spring Boot MockMvc
@WebMvcTest(PedidoController.class)
class PedidoControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private CriarPedidoUseCase criarPedido;

    @Test
    void criarPedido_bodyValido_retorna201ComLocation() throws Exception {
        // Arrange
        when(criarPedido.executar(any())).thenReturn(new PedidoCriado("ped-123"));

        // Act + Assert
        mockMvc.perform(post("/api/v1/pedidos")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"clienteId":"cli-1","itens":[{"sku":"ABC","quantidade":2}]}
                """))
            .andExpect(status().isCreated())
            .andExpect(header().string("Location", containsString("/pedidos/ped-123")))
            .andExpect(jsonPath("$.pedidoId").value("ped-123"));
    }

    @Test
    void criarPedido_bodyInvalido_retorna400ComDetalhes() throws Exception {
        mockMvc.perform(post("/api/v1/pedidos")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"clienteId":"","itens":[]}
                """))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").isArray())
            .andExpect(jsonPath("$.errors[*].field", hasItems("clienteId", "itens")));
    }

    @Test
    void criarPedido_semContentType_retorna415() throws Exception {
        mockMvc.perform(post("/api/v1/pedidos")
                .content("some body"))
            .andExpect(status().isUnsupportedMediaType());
    }
}
```

### 7.2 API Test Checklist

| Item                                      | Teste                                                     |
|-------------------------------------------|-----------------------------------------------------------|
| Happy path com body válido                | Status correto, response body, headers (Location, etc.)  |
| Validação de campos obrigatórios          | 400 com detalhes de quais campos falharam                 |
| Validação de formato/tipo                 | 400 para email inválido, data inválida, número negativo   |
| Autenticação ausente                      | 401 Unauthorized                                          |
| Autorização insuficiente                  | 403 Forbidden                                             |
| Recurso não encontrado                    | 404 Not Found                                             |
| Content-Type incorreto                    | 415 Unsupported Media Type                                |
| Paginação                                 | Default page size, next/prev links, empty page            |
| Ordenação                                 | Parâmetros de sort, direção default                       |
| Concorrência (If-Match)                   | 409 Conflict ou 412 Precondition Failed                   |
| Rate limiting                             | 429 Too Many Requests com Retry-After header              |

---

## 8. Padrões para Testes Assíncronos

### 8.1 Awaitility Pattern (evitar sleep)

```java
// Java — Awaitility
@Test
void processarEvento_pedidoCriado_atualizaEstoque() {
    // Arrange
    var evento = new PedidoCriadoEvent("ped-1", List.of(new Item("SKU-1", 2)));

    // Act
    eventPublisher.publish(evento);

    // Assert — aguardar processamento assíncrono
    await()
        .atMost(Duration.ofSeconds(10))
        .pollInterval(Duration.ofMillis(200))
        .until(() -> estoqueRepository.buscarPorSku("SKU-1").getReservado() == 2);
}
```

```go
// Go — polling manual
func TestProcessEvent_UpdatesInventory(t *testing.T) {
    publisher.Publish(ctx, OrderCreatedEvent{OrderID: "ord-1", Items: []Item{{SKU: "SKU-1", Qty: 2}}})

    require.Eventually(t, func() bool {
        stock, err := stockRepo.FindBySKU(ctx, "SKU-1")
        return err == nil && stock.Reserved == 2
    }, 10*time.Second, 200*time.Millisecond, "estoque não foi atualizado")
}
```

```typescript
// TypeScript — waitFor pattern
import { waitFor } from '@testing-library/react'; // ou implementação custom

test('processar evento atualiza estoque', async () => {
  await publisher.publish({ type: 'ORDER_CREATED', sku: 'SKU-1', qty: 2 });

  await waitFor(
    async () => {
      const stock = await stockRepo.findBySku('SKU-1');
      expect(stock.reserved).toBe(2);
    },
    { timeout: 10000, interval: 200 }
  );
});
```

### 8.2 Idempotency Test Pattern

```java
@Test
void processarMensagem_duplicada_naoGeraEfeitoColateralDuplicado() {
    var mensagem = new PagamentoAprovadoEvent("ped-1", new BigDecimal("100.00"));

    // Act — processar a mesma mensagem duas vezes
    consumer.handle(mensagem);
    consumer.handle(mensagem); // duplicada

    // Assert
    var pagamentos = pagamentoRepository.buscarPorPedido("ped-1");
    assertThat(pagamentos).hasSize(1); // apenas 1 pagamento criado
}
```

### 8.3 DLQ (Dead Letter Queue) Test Pattern

```java
@Test
void processarMensagem_falhaAposMaxRetries_enviaParaDLQ() {
    // Arrange — mensagem que causa erro
    var mensagemInvalida = new PagamentoEvent(/* dados inválidos */);

    // Act — publicar mensagem que falhará
    producer.send(TOPIC_PAGAMENTOS, mensagemInvalida);

    // Assert — aguardar mensagem chegar na DLQ
    await().atMost(Duration.ofSeconds(30)).until(() -> {
        var dlqMessages = dlqConsumer.poll(Duration.ofMillis(100));
        return dlqMessages.stream()
            .anyMatch(m -> m.value().contains(mensagemInvalida.getId()));
    });
}
```

---

## 9. Padrões para Testes de Banco de Dados

### 9.1 Schema Migration Test

```java
@Test
void migracoes_devemExecutarComSucesso_emBancoLimpo() {
    // Arrange — banco vazio (container recém-criado)
    var flyway = Flyway.configure()
        .dataSource(postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword())
        .load();

    // Act
    var resultado = flyway.migrate();

    // Assert
    assertThat(resultado.success).isTrue();
    assertThat(resultado.migrationsExecuted).isGreaterThan(0);
}
```

### 9.2 Query Result Test

```java
@Test
void buscarPedidosPorStatus_multiplosStatus_retornaFiltrasdosPorStatusOrdenados() {
    // Arrange — popular com dados variados
    repository.save(PedidoBuilder.umPedido().comStatus(PENDENTE).comData("2026-01-15").build());
    repository.save(PedidoBuilder.umPedido().comStatus(PAGO).comData("2026-01-14").build());
    repository.save(PedidoBuilder.umPedido().comStatus(PENDENTE).comData("2026-01-13").build());
    repository.save(PedidoBuilder.umPedido().comStatus(CANCELADO).build());

    // Act
    var resultado = repository.buscarPorStatus(List.of(PENDENTE, PAGO), Pageable.ofSize(10));

    // Assert
    assertThat(resultado.getContent()).hasSize(3);
    assertThat(resultado.getContent()).allSatisfy(p ->
        assertThat(p.getStatus()).isIn(PENDENTE, PAGO)
    );
    // Verificar ordenação
    assertThat(resultado.getContent())
        .extracting(Pedido::getCriadoEm)
        .isSortedAccordingTo(Comparator.reverseOrder());
}
```

### 9.3 Concurrent Access Test

```java
@Test
void atualizarEstoque_acessoConcorrente_naoPerdeAtualizacoes() throws Exception {
    // Arrange
    estoqueRepository.save(new Estoque("SKU-1", 100));  // 100 unidades

    // Act — 10 threads decrementando 1 unidade cada
    int threads = 10;
    var latch = new CountDownLatch(threads);
    var executor = Executors.newFixedThreadPool(threads);

    for (int i = 0; i < threads; i++) {
        executor.submit(() -> {
            try {
                estoqueService.decrementar("SKU-1", 1);
            } finally {
                latch.countDown();
            }
        });
    }

    latch.await(10, TimeUnit.SECONDS);

    // Assert — deve ter exatamente 90 (100 - 10)
    var estoque = estoqueRepository.buscarPorSku("SKU-1");
    assertThat(estoque.getQuantidade()).isEqualTo(90);
}
```

---

## 10. Padrões para Refatoração de Testes

### 10.1 Sinais de que Testes Precisam Refatorar

| Sinal                                         | Ação recomendada                                           |
|-----------------------------------------------|------------------------------------------------------------|
| Setup > 20 linhas                             | Extrair Builder/Factory                                     |
| Asserts repetidos entre testes                | Extrair Custom Assertion                                    |
| Mesmo cenário testado em unitário e integração | Escolher o nível correto; eliminar duplicidade             |
| Teste com `if/for/try-catch`                  | Dividir em testes separados; usar parametrizado            |
| Nome do teste não descreve cenário            | Renomear seguindo convenção [unidade]_[cenário]_[resultado] |
| Mock com mais de 5 `when()`                   | Extrair para Fake ou redesenhar o código sob teste         |
| Teste depende de ordem de execução            | Isolar com cleanup/setup por teste                          |

### 10.2 Testes Parametrizados (eliminar duplicação)

```java
// Java — JUnit 5 @ParameterizedTest
@ParameterizedTest(name = "validarCPF({0}) = {1}")
@CsvSource({
    "11144477735, true",       // CPF válido
    "111.444.777-35, true",    // CPF válido com máscara
    "12345678901, false",      // CPF inválido
    "'', false",               // vazio
    "abc, false",              // não numérico
    "1234567890, false",       // tamanho errado
    "11111111111, false",      // todos iguais
})
void validarCPF(String cpf, boolean esperado) {
    assertThat(CpfValidator.isValid(cpf)).isEqualTo(esperado);
}
```

```go
// Go — Table-driven tests
func TestValidateCPF(t *testing.T) {
    tests := []struct {
        name     string
        cpf      string
        expected bool
    }{
        {"CPF válido", "11144477735", true},
        {"CPF com máscara", "111.444.777-35", true},
        {"CPF inválido", "12345678901", false},
        {"vazio", "", false},
        {"todos iguais", "11111111111", false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := ValidateCPF(tt.cpf)
            assert.Equal(t, tt.expected, result)
        })
    }
}
```

```typescript
// TypeScript — Jest each
test.each([
  ['11144477735', true],
  ['111.444.777-35', true],
  ['12345678901', false],
  ['', false],
  ['11111111111', false],
])('validateCPF(%s) = %s', (cpf, expected) => {
  expect(validateCPF(cpf)).toBe(expected);
});
```

---

## 11. Receitas por Linguagem

### 11.1 Java — Setup Recomendado

```xml
<!-- Maven — dependências de teste -->
<dependencies>
    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- AssertJ -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- Mockito -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- Testcontainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- Awaitility -->
    <dependency>
        <groupId>org.awaitility</groupId>
        <artifactId>awaitility</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 11.2 TypeScript — Setup Recomendado

```json
// package.json — devDependencies
{
  "devDependencies": {
    "vitest": "^2.0.0",
    "@testing-library/react": "^16.0.0",
    "@testing-library/jest-dom": "^6.0.0",
    "@testing-library/user-event": "^14.0.0",
    "msw": "^2.0.0",
    "fast-check": "^3.0.0",
    "testcontainers": "^10.0.0"
  }
}
```

### 11.3 Go — Setup Recomendado

```go
// go.mod — test dependencies
require (
    github.com/stretchr/testify v1.9.0
    github.com/testcontainers/testcontainers-go v0.33.0
    github.com/google/uuid v1.6.0
)
```

```go
// Helpers reutilizáveis — testhelpers/testhelpers.go
package testhelpers

import (
    "context"
    "testing"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
)

func StartPostgres(t *testing.T) (dsn string, cleanup func()) {
    t.Helper()
    ctx := context.Background()

    container, err := postgres.Run(ctx, "postgres:16-alpine",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
    )
    if err != nil {
        t.Fatalf("failed to start postgres: %v", err)
    }

    connStr, _ := container.ConnectionString(ctx, "sslmode=disable")
    return connStr, func() { container.Terminate(ctx) }
}
```

### 11.4 Python — Setup Recomendado

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
markers = [
    "unit: Unit tests (no I/O)",
    "integration: Integration tests (requires infra)",
    "e2e: End-to-end tests",
]
addopts = "-v --strict-markers"

[project.optional-dependencies]
test = [
    "pytest>=8.0",
    "pytest-asyncio>=0.24",
    "pytest-mock>=3.14",
    "hypothesis>=6.100",
    "testcontainers>=4.0",
    "httpx>=0.27",  # para testes de API async
    "factory-boy>=3.3",
]
```

```python
# conftest.py — fixtures compartilhadas
import pytest
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg.get_connection_url()

@pytest.fixture(autouse=True)
def clean_db(db_session):
    yield
    db_session.rollback()
```
