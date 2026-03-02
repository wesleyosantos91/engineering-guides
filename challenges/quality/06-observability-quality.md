# Level 6 — Observabilidade & Qualidade em Produção

> **Objetivo:** Implementar observabilidade completa para a Digital Wallet — logging estruturado,
> métricas de negócio, distributed tracing, SLIs/SLOs, alertas baseados em error budget e
> dashboards — garantindo qualidade contínua após o deploy.

**Referência:** [.docs/QUALITY-ENGINEERING/06-observability-quality.md](../../.docs/QUALITY-ENGINEERING/06-observability-quality.md)

---

## Contexto do Domínio

Qualidade não termina no merge. Um sistema financeiro que **não pode ser observado em produção**
é um sistema cuja qualidade **não pode ser verificada**. A Digital Wallet precisa detectar
anomalias (saldo negativo, transações duplicadas, latência crescente) em **segundos**, não horas.
Neste nível, você implementará os três pilares de observabilidade e criará uma cultura
de monitoramento proativo.

---

## Desafios

### Desafio 6.1 — Logging Estruturado com Contexto de Negócio

**Contexto:** Logs são a primeira linha de investigação quando algo dá errado. Para a Digital Wallet,
logs precisam ser estruturados (JSON), conter contexto de negócio (`walletId`, `transactionId`,
`amount`) e **nunca** incluir dados sensíveis (CPF, token).

**Requisitos:**

- Implementar logging estruturado em JSON com campos obrigatórios:

```json
{
  "timestamp": "2026-03-01T10:30:45.123Z",
  "level": "INFO",
  "logger": "com.wallet.domain.service.TransactionService",
  "message": "Deposit completed successfully",
  "traceId": "abc123def456",
  "spanId": "span-789",
  "correlationId": "req-uuid-001",
  "userId": "user-42",
  "walletId": "wallet-99",
  "transactionId": "txn-555",
  "amount": 150.00,
  "currency": "BRL",
  "transactionType": "DEPOSIT",
  "duration_ms": 45
}
```

- **Java 25 (SLF4J + Logback + MDC):**
```java
@Service
public class TransactionService {
    private static final Logger log = LoggerFactory.getLogger(TransactionService.class);

    public Transaction deposit(UUID walletId, DepositRequest request, AuthContext auth) {
        MDC.put("userId", auth.getUserId().toString());
        MDC.put("walletId", walletId.toString());
        MDC.put("correlationId", auth.getCorrelationId());

        log.info("Processing deposit: amount={}, currency={}, idempotencyKey={}",
                request.getAmount(), request.getCurrency(), request.getIdempotencyKey());

        try {
            var wallet = walletRepository.findById(walletId)
                    .orElseThrow(() -> new WalletNotFoundException(walletId));

            wallet.credit(request.getAmount());
            var transaction = createTransaction(wallet, request);
            walletRepository.save(wallet);
            transactionRepository.save(transaction);

            log.info("Deposit completed: transactionId={}, newBalance={}, duration_ms={}",
                    transaction.getId(), wallet.getBalance(), elapsed());

            return transaction;
        } catch (WalletNotFoundException e) {
            log.warn("Wallet not found for deposit: walletId={}", walletId);
            throw e;
        } catch (Exception e) {
            log.error("Deposit failed: amount={}, error={}",
                    request.getAmount(), e.getMessage(), e);
            throw e;
        } finally {
            MDC.clear();
        }
    }
}
```

- **Go 1.26 (slog structured):**
```go
func (s *TransactionService) Deposit(ctx context.Context, walletID uuid.UUID, req DepositRequest) (*Transaction, error) {
    logger := slog.With(
        "userId", middleware.UserIDFromContext(ctx),
        "walletId", walletID.String(),
        "correlationId", middleware.CorrelationIDFromContext(ctx),
    )

    logger.InfoContext(ctx, "processing deposit",
        "amount", req.Amount,
        "currency", req.Currency,
        "idempotencyKey", req.IdempotencyKey,
    )

    start := time.Now()

    wallet, err := s.walletStore.FindByID(ctx, walletID)
    if err != nil {
        logger.WarnContext(ctx, "wallet not found for deposit", "error", err)
        return nil, fmt.Errorf("find wallet: %w", err)
    }

    wallet.Credit(req.Amount)
    txn, err := s.createTransaction(ctx, wallet, req)
    if err != nil {
        logger.ErrorContext(ctx, "deposit failed",
            "amount", req.Amount,
            "error", err,
        )
        return nil, fmt.Errorf("create transaction: %w", err)
    }

    logger.InfoContext(ctx, "deposit completed",
        "transactionId", txn.ID.String(),
        "newBalance", wallet.Balance,
        "duration_ms", time.Since(start).Milliseconds(),
    )

    return txn, nil
}
```

- Configurar **mascaramento de dados sensíveis**:

**Java 25:**
```java
// Logback pattern com mascaramento
public class SensitiveDataMaskingLayout extends LayoutBase<ILoggingEvent> {
    private static final Pattern CPF_PATTERN =
            Pattern.compile("\\b(\\d{3})\\.?(\\d{3})\\.?(\\d{3})-?(\\d{2})\\b");
    private static final Pattern EMAIL_PATTERN =
            Pattern.compile("([a-zA-Z0-9._%+-]+)@([a-zA-Z0-9.-]+\\.[a-zA-Z]{2,})");

    @Override
    public String doLayout(ILoggingEvent event) {
        String message = event.getFormattedMessage();
        message = CPF_PATTERN.matcher(message).replaceAll("***.$2.***-**");
        message = EMAIL_PATTERN.matcher(message).replaceAll("***@$2");
        return message;
    }
}
```

- Implementar **testes de logging**:

**Java 25:**
```java
@Test
void deposit_shouldLogTransactionWithBusinessContext() {
    try (LogCaptor logCaptor = LogCaptor.forClass(TransactionService.class)) {
        transactionService.deposit(walletId, depositRequest, authContext);

        assertThat(logCaptor.getInfoLogs())
                .anyMatch(log -> log.contains("Deposit completed"))
                .anyMatch(log -> log.contains("transactionId="))
                .anyMatch(log -> log.contains("newBalance="))
                .anyMatch(log -> log.contains("duration_ms="));
    }
}

@Test
void deposit_shouldNotLogSensitiveData() {
    try (LogCaptor logCaptor = LogCaptor.forRoot()) {
        transactionService.deposit(walletId, depositRequest, authContext);

        String allLogs = String.join("\n", logCaptor.getLogs());
        assertThat(allLogs).doesNotContain(user.getCpf());
        assertThat(allLogs).doesNotContain(user.getEmail());
        assertThat(allLogs).doesNotContain(authContext.getToken());
    }
}
```

**Go 1.26:**
```go
func TestDeposit_LogsBusinessContext(t *testing.T) {
    var buf bytes.Buffer
    logger := slog.New(slog.NewJSONHandler(&buf, nil))
    svc := NewTransactionService(logger, walletStore, txnStore)

    _, err := svc.Deposit(ctx, walletID, depositReq)
    require.NoError(t, err)

    logs := buf.String()
    assert.Contains(t, logs, "deposit completed")
    assert.Contains(t, logs, "transactionId")
    assert.Contains(t, logs, "newBalance")
    assert.Contains(t, logs, "duration_ms")
}

func TestDeposit_DoesNotLogSensitiveData(t *testing.T) {
    var buf bytes.Buffer
    logger := slog.New(slog.NewJSONHandler(&buf, nil))
    svc := NewTransactionService(logger, walletStore, txnStore)

    _, _ = svc.Deposit(ctx, walletID, depositReq)

    logs := buf.String()
    assert.NotContains(t, logs, user.Document)
    assert.NotContains(t, logs, user.Email)
}
```

**Critérios de aceite:**

- [ ] Logging estruturado em JSON (Go + Java)
- [ ] MDC/context com correlationId, userId, walletId
- [ ] Mascaramento de dados sensíveis implementado
- [ ] Testes de logging: contexto de negócio presente
- [ ] Testes de logging: dados sensíveis ausentes
- [ ] Logback (Java) / slog (Go) configurados para JSON em produção

---

### Desafio 6.2 — Métricas de Negócio com Micrometer/Prometheus

**Contexto:** Métricas técnicas (CPU, memória) são importantes, mas **métricas de negócio**
(transações/min, valor médio de depósito, taxa de rejeição) são o que realmente indicam
se a Digital Wallet está funcionando corretamente.

**Requisitos:**

- Implementar métricas de negócio usando RED method + business KPIs:

**Java 25 (Micrometer + Spring Boot):**
```java
@Service
public class TransactionService {
    private final Counter depositsCompleted;
    private final Counter depositsFailed;
    private final Counter withdrawalsCompleted;
    private final Counter transfersCompleted;
    private final Timer depositTimer;
    private final Timer transferTimer;
    private final DistributionSummary depositValue;
    private final Gauge activeWallets;

    public TransactionService(MeterRegistry registry, WalletRepository walletRepo) {
        this.depositsCompleted = Counter.builder("wallet.transactions.completed")
                .tag("type", "DEPOSIT")
                .description("Total de depósitos concluídos")
                .register(registry);

        this.depositsFailed = Counter.builder("wallet.transactions.failed")
                .tag("type", "DEPOSIT")
                .description("Total de depósitos que falharam")
                .register(registry);

        this.withdrawalsCompleted = Counter.builder("wallet.transactions.completed")
                .tag("type", "WITHDRAWAL")
                .register(registry);

        this.transfersCompleted = Counter.builder("wallet.transactions.completed")
                .tag("type", "TRANSFER")
                .register(registry);

        this.depositTimer = Timer.builder("wallet.transactions.duration")
                .tag("type", "DEPOSIT")
                .publishPercentiles(0.5, 0.95, 0.99)
                .register(registry);

        this.transferTimer = Timer.builder("wallet.transactions.duration")
                .tag("type", "TRANSFER")
                .publishPercentiles(0.5, 0.95, 0.99)
                .register(registry);

        this.depositValue = DistributionSummary.builder("wallet.deposit.value")
                .baseUnit("BRL")
                .publishPercentiles(0.5, 0.95)
                .register(registry);

        this.activeWallets = Gauge.builder("wallet.active.count",
                walletRepo, repo -> repo.countByStatus(WalletStatus.ACTIVE))
                .register(registry);
    }

    public Transaction deposit(UUID walletId, DepositRequest request, AuthContext auth) {
        return depositTimer.record(() -> {
            try {
                Transaction txn = processDeposit(walletId, request, auth);
                depositsCompleted.increment();
                depositValue.record(request.getAmount().doubleValue());
                return txn;
            } catch (Exception e) {
                depositsFailed.increment();
                throw e;
            }
        });
    }
}
```

**Go 1.26 (Prometheus client):**
```go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    TransactionsCompleted = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "wallet_transactions_completed_total",
        Help: "Total de transações concluídas",
    }, []string{"type"})

    TransactionsFailed = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "wallet_transactions_failed_total",
        Help: "Total de transações que falharam",
    }, []string{"type", "reason"})

    TransactionDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "wallet_transactions_duration_seconds",
        Help:    "Duração das transações",
        Buckets: []float64{0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5},
    }, []string{"type"})

    DepositValue = promauto.NewHistogram(prometheus.HistogramOpts{
        Name:    "wallet_deposit_value_brl",
        Help:    "Valor dos depósitos em BRL",
        Buckets: prometheus.ExponentialBuckets(10, 2, 12), // 10, 20, 40, ..., 40960
    })

    ActiveWallets = promauto.NewGauge(prometheus.GaugeOpts{
        Name: "wallet_active_count",
        Help: "Número de wallets ativas",
    })

    WalletBalance = promauto.NewHistogram(prometheus.HistogramOpts{
        Name:    "wallet_balance_brl",
        Help:    "Distribuição de saldos das wallets",
        Buckets: prometheus.ExponentialBuckets(1, 5, 10),
    })
)
```

- Implementar **testes de métricas**:

**Java 25:**
```java
@Test
void deposit_shouldRecordMetrics() {
    SimpleMeterRegistry registry = new SimpleMeterRegistry();
    var service = new TransactionService(registry, walletRepo, txnRepo);

    service.deposit(walletId, depositRequest("100.00"), authContext);

    assertThat(registry.counter("wallet.transactions.completed",
            "type", "DEPOSIT").count())
            .isEqualTo(1.0);

    assertThat(registry.timer("wallet.transactions.duration",
            "type", "DEPOSIT").count())
            .isEqualTo(1);

    assertThat(registry.summary("wallet.deposit.value").count())
            .isEqualTo(1);
    assertThat(registry.summary("wallet.deposit.value").totalAmount())
            .isEqualTo(100.0);
}

@Test
void deposit_onFailure_shouldRecordErrorMetrics() {
    SimpleMeterRegistry registry = new SimpleMeterRegistry();
    var service = new TransactionService(registry, failingWalletRepo, txnRepo);

    assertThrows(WalletNotFoundException.class,
            () -> service.deposit(walletId, depositRequest("100.00"), authContext));

    assertThat(registry.counter("wallet.transactions.failed",
            "type", "DEPOSIT").count())
            .isEqualTo(1.0);
}
```

**Go 1.26:**
```go
func TestDeposit_RecordsMetrics(t *testing.T) {
    // Reset metrics
    metrics.TransactionsCompleted.Reset()

    svc := NewTransactionService(walletStore, txnStore)
    _, err := svc.Deposit(ctx, walletID, depositReq)
    require.NoError(t, err)

    // Verificar counter
    counter, err := metrics.TransactionsCompleted.GetMetricWithLabelValues("DEPOSIT")
    require.NoError(t, err)

    m := &dto.Metric{}
    counter.Write(m)
    assert.Equal(t, float64(1), m.GetCounter().GetValue())
}
```

**Critérios de aceite:**

- [ ] Métricas RED implementadas (Rate, Errors, Duration) por tipo de transação
- [ ] Métricas de negócio: valor de depósito, wallets ativas, taxa de rejeição
- [ ] Endpoint `/metrics` (Prometheus) exposto e funcional
- [ ] Testes de métricas (sucesso e falha)
- [ ] Pelo menos 8 métricas customizadas definidas
- [ ] Métricas com labels adequados (type, status, reason)

---

### Desafio 6.3 — Distributed Tracing com OpenTelemetry

**Contexto:** Quando uma transferência falha, precisamos rastrear o caminho completo:
API Gateway → TransactionService → WalletService → PostgreSQL → EventPublisher.
Distributed tracing conecta todos os spans de uma requisição em um único trace.

**Requisitos:**

- Instrumentar a Digital Wallet com OpenTelemetry:
  - Trace de cada operação financeira (deposit, withdrawal, transfer)
  - Spans para chamadas de banco de dados
  - Spans para publicação de eventos
  - Atributos de negócio em cada span

**Java 25 (OpenTelemetry + Spring Boot):**
```java
@Service
public class TransactionService {
    private final Tracer tracer;

    public TransactionService(Tracer tracer) {
        this.tracer = tracer;
    }

    public Transaction transfer(UUID sourceWalletId, TransferRequest request, AuthContext auth) {
        Span span = tracer.spanBuilder("transaction.transfer")
                .setAttribute("wallet.source.id", sourceWalletId.toString())
                .setAttribute("wallet.target.id", request.getTargetWalletId().toString())
                .setAttribute("transaction.amount", request.getAmount().doubleValue())
                .setAttribute("transaction.currency", request.getCurrency())
                .setAttribute("user.id", auth.getUserId().toString())
                .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // Span filho: débito
            Span debitSpan = tracer.spanBuilder("wallet.debit")
                    .setAttribute("wallet.id", sourceWalletId.toString())
                    .startSpan();
            try (Scope debitScope = debitSpan.makeCurrent()) {
                sourceWallet.debit(request.getAmount());
                debitSpan.setAttribute("wallet.new_balance",
                        sourceWallet.getBalance().doubleValue());
                debitSpan.setStatus(StatusCode.OK);
            } finally {
                debitSpan.end();
            }

            // Span filho: crédito
            Span creditSpan = tracer.spanBuilder("wallet.credit")
                    .setAttribute("wallet.id", request.getTargetWalletId().toString())
                    .startSpan();
            try (Scope creditScope = creditSpan.makeCurrent()) {
                targetWallet.credit(request.getAmount());
                creditSpan.setStatus(StatusCode.OK);
            } finally {
                creditSpan.end();
            }

            // Span filho: publicar evento
            Span eventSpan = tracer.spanBuilder("event.publish")
                    .setAttribute("event.type", "TransferCompleted")
                    .startSpan();
            try (Scope eventScope = eventSpan.makeCurrent()) {
                eventPublisher.publish(new TransferCompletedEvent(transaction));
                eventSpan.setStatus(StatusCode.OK);
            } finally {
                eventSpan.end();
            }

            span.setStatus(StatusCode.OK);
            span.setAttribute("transaction.id", transaction.getId().toString());
            return transaction;

        } catch (InsufficientBalanceException e) {
            span.setStatus(StatusCode.ERROR, "Insufficient balance");
            span.recordException(e);
            throw e;
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

**Go 1.26 (OpenTelemetry Go SDK):**
```go
func (s *TransactionService) Transfer(ctx context.Context, sourceWalletID uuid.UUID, req TransferRequest) (*Transaction, error) {
    ctx, span := s.tracer.Start(ctx, "transaction.transfer",
        trace.WithAttributes(
            attribute.String("wallet.source.id", sourceWalletID.String()),
            attribute.String("wallet.target.id", req.TargetWalletID.String()),
            attribute.Float64("transaction.amount", req.Amount),
            attribute.String("transaction.currency", req.Currency),
        ),
    )
    defer span.End()

    // Span: débito
    ctx, debitSpan := s.tracer.Start(ctx, "wallet.debit",
        trace.WithAttributes(attribute.String("wallet.id", sourceWalletID.String())),
    )
    if err := sourceWallet.Debit(req.Amount); err != nil {
        debitSpan.SetStatus(codes.Error, err.Error())
        debitSpan.RecordError(err)
        debitSpan.End()
        span.SetStatus(codes.Error, "insufficient balance")
        return nil, err
    }
    debitSpan.SetAttributes(attribute.Float64("wallet.new_balance", sourceWallet.Balance))
    debitSpan.SetStatus(codes.Ok, "")
    debitSpan.End()

    // Span: crédito
    _, creditSpan := s.tracer.Start(ctx, "wallet.credit",
        trace.WithAttributes(attribute.String("wallet.id", req.TargetWalletID.String())),
    )
    targetWallet.Credit(req.Amount)
    creditSpan.SetStatus(codes.Ok, "")
    creditSpan.End()

    // Span: evento
    _, eventSpan := s.tracer.Start(ctx, "event.publish",
        trace.WithAttributes(attribute.String("event.type", "TransferCompleted")),
    )
    s.eventPublisher.Publish(ctx, TransferCompletedEvent{Transaction: txn})
    eventSpan.SetStatus(codes.Ok, "")
    eventSpan.End()

    span.SetAttributes(attribute.String("transaction.id", txn.ID.String()))
    span.SetStatus(codes.Ok, "")
    return txn, nil
}
```

- Implementar **testes de trace propagation**:

**Java 25:**
```java
@Test
void transfer_shouldPropagateTraceContext() {
    // Given — mock payment service com WireMock
    MockWebServer paymentService = new MockWebServer();
    paymentService.enqueue(new MockResponse().setResponseCode(200).setBody("{}"));

    // When
    transactionService.transfer(sourceWalletId, transferRequest, authContext);

    // Then — verificar que traceparent header foi propagado
    RecordedRequest recorded = paymentService.takeRequest();
    assertThat(recorded.getHeader("traceparent")).isNotNull();
    assertThat(recorded.getHeader("traceparent")).startsWith("00-");
}

@Test
void transfer_shouldCreateChildSpans() {
    InMemorySpanExporter spanExporter = InMemorySpanExporter.create();
    // Configure test tracer with in-memory exporter
    SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(SimpleSpanProcessor.create(spanExporter))
            .build();

    transactionService.transfer(sourceWalletId, transferRequest, authContext);

    List<SpanData> spans = spanExporter.getFinishedSpanItems();
    assertThat(spans).extracting(SpanData::getName)
            .contains("transaction.transfer", "wallet.debit",
                    "wallet.credit", "event.publish");

    // Parent-child relationship
    SpanData transferSpan = spans.stream()
            .filter(s -> s.getName().equals("transaction.transfer"))
            .findFirst().orElseThrow();
    SpanData debitSpan = spans.stream()
            .filter(s -> s.getName().equals("wallet.debit"))
            .findFirst().orElseThrow();
    assertThat(debitSpan.getParentSpanId())
            .isEqualTo(transferSpan.getSpanContext().getSpanId());
}
```

**Critérios de aceite:**

- [ ] OpenTelemetry SDK integrado (Go + Java)
- [ ] Traces para deposit, withdrawal e transfer
- [ ] Child spans para operações internas (debit, credit, event publish)
- [ ] Atributos de negócio nos spans (walletId, amount, transactionId)
- [ ] Teste de trace propagation (traceparent header)
- [ ] Teste de parent-child span relationship
- [ ] Exportador configurado (Jaeger, Tempo ou console para dev)

---

### Desafio 6.4 — SLIs, SLOs e Error Budget

**Contexto:** SLOs (Service Level Objectives) transformam expectativas vagas ("o sistema deve ser rápido")
em **targets mensuráveis** ("p99 latência ≤ 300ms em 99.5% do tempo"). Error budget define
quanto de "falha" é aceitável.

**Requisitos:**

- Definir SLIs e SLOs para a Digital Wallet:

```yaml
# slo-definitions.yaml
service: digital-wallet
owner: team-wallet

slos:
  - name: availability
    description: "Proporção de requests bem-sucedidas (non-5xx)"
    sli:
      type: availability
      good_events: "http_status < 500"
      total_events: "all http requests"
    target: 99.9          # 0.1% error budget → ~43 min/mês
    window: 30d

  - name: deposit-latency
    description: "Latência P99 do endpoint de depósito"
    sli:
      type: latency
      endpoint: "POST /api/v1/wallets/{id}/transactions/deposit"
      threshold_ms: 300
      percentile: 99
    target: 99.5
    window: 30d

  - name: transfer-latency
    description: "Latência P99 do endpoint de transferência"
    sli:
      type: latency
      endpoint: "POST /api/v1/wallets/{id}/transactions/transfer"
      threshold_ms: 500
      percentile: 99
    target: 99.0
    window: 30d

  - name: data-correctness
    description: "Transações processadas sem inconsistência de saldo"
    sli:
      type: correctness
      good_events: "transactions where debit_amount == credit_amount"
      total_events: "all completed transfers"
    target: 99.99
    window: 30d

error_budget_policy:
  - budget_remaining: "> 50%"
    action: "Desenvolvimento normal, features e experimentos"
  - budget_remaining: "25-50%"
    action: "Aumentar cautela; reviews mais rigorosos"
  - budget_remaining: "< 25%"
    action: "Foco em estabilidade; só bug fixes"
  - budget_remaining: "0%"
    action: "Feature freeze; foco total em confiabilidade"
```

- Implementar cálculo de SLI/SLO com Prometheus queries:

```yaml
# prometheus-rules/slo.yml
groups:
  - name: wallet-slos
    rules:
      # SLI: Availability
      - record: wallet:availability:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{service="digital-wallet",status!~"5.."}[5m]))
          /
          sum(rate(http_requests_total{service="digital-wallet"}[5m]))

      # SLI: Deposit latency (% below threshold)
      - record: wallet:deposit_latency:good_ratio_rate5m
        expr: |
          sum(rate(http_request_duration_seconds_bucket{
            service="digital-wallet",
            endpoint="/api/v1/wallets/{id}/transactions/deposit",
            le="0.3"
          }[5m]))
          /
          sum(rate(http_request_duration_seconds_count{
            service="digital-wallet",
            endpoint="/api/v1/wallets/{id}/transactions/deposit"
          }[5m]))

      # Error Budget remaining (30d window)
      - record: wallet:error_budget:remaining_ratio
        expr: |
          1 - (
            (1 - wallet:availability:ratio_rate5m)
            /
            (1 - 0.999)
          )
```

- Implementar **testes de SLO compliance**:

**Java 25:**
```java
@Test
void depositEndpoint_shouldMeetLatencySLO() {
    // SLO: P99 ≤ 300ms
    var latencies = new ArrayList<Long>();

    for (int i = 0; i < 100; i++) {
        long start = System.nanoTime();
        restTemplate.postForEntity(
                "/api/v1/wallets/{id}/transactions/deposit",
                depositRequest(), TransactionResponse.class, walletId);
        latencies.add(TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start));
    }

    Collections.sort(latencies);
    long p99 = latencies.get(98); // P99

    assertThat(p99).as("P99 latency should be ≤ 300ms (SLO)").isLessThanOrEqualTo(300);
}
```

**Critérios de aceite:**

- [ ] SLIs definidos para availability, latency e correctness
- [ ] SLOs com targets numéricos e janela de tempo
- [ ] Error budget policy com 4 níveis de ação
- [ ] Prometheus recording rules para cálculo de SLI
- [ ] Teste de SLO compliance para pelo menos 2 endpoints
- [ ] Documento: `decisions/06-slo-definitions.yaml`

---

### Desafio 6.5 — Alertas Baseados em Error Budget (Multi-Window Burn Rate)

**Contexto:** Alertas tradicionais ("error rate > 1%") geram muitos falsos positivos.
Alertas baseados em **burn rate** alertam quando o error budget está sendo consumido
mais rápido que o esperado — mais preciso e menos ruidoso.

**Requisitos:**

- Implementar alertas com multi-window burn rate:

| Alerta | Janela curta | Janela longa | Burn rate | Severidade |
|---|---|---|---|---|
| Consumo rápido | 5 min | 1 hora | 14.4x | 🔴 Critical (P1) |
| Consumo moderado | 30 min | 6 horas | 6x | 🟠 Warning (P2) |
| Consumo lento | 2 horas | 24 horas | 3x | 🟡 Ticket (P3) |

```yaml
# prometheus-rules/alerts.yml
groups:
  - name: wallet-slo-alerts
    rules:
      # P1: Consumo rápido de error budget (14.4x burn rate)
      - alert: WalletHighErrorBurnRate
        expr: |
          (
            # Janela curta: 5min
            (1 - wallet:availability:ratio_rate5m) > (14.4 * (1 - 0.999))
          )
          and
          (
            # Janela longa: 1h
            (1 - wallet:availability:ratio_rate1h) > (14.4 * (1 - 0.999))
          )
        for: 2m
        labels:
          severity: critical
          team: wallet
          slo: availability
        annotations:
          summary: "Digital Wallet error budget being consumed 14.4x faster than expected"
          description: >
            Error rate (5min): {{ $value | humanizePercentage }}.
            At this rate, the entire 30-day error budget will be consumed in ~2 days.
          runbook: "https://wiki.example.com/runbooks/wallet-high-error-rate"
          dashboard: "https://grafana.example.com/d/wallet-slo"

      # P2: Consumo moderado (6x burn rate)
      - alert: WalletModerateErrorBurnRate
        expr: |
          (
            (1 - wallet:availability:ratio_rate30m) > (6 * (1 - 0.999))
          )
          and
          (
            (1 - wallet:availability:ratio_rate6h) > (6 * (1 - 0.999))
          )
        for: 5m
        labels:
          severity: warning
          team: wallet
        annotations:
          summary: "Digital Wallet error budget being consumed 6x faster than expected"
          runbook: "https://wiki.example.com/runbooks/wallet-moderate-errors"

      # Latência: deposit P99 > SLO
      - alert: WalletDepositHighLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket{
              service="digital-wallet",
              endpoint=~".*deposit.*"
            }[10m])) by (le)
          ) > 0.3
        for: 10m
        labels:
          severity: warning
          team: wallet
        annotations:
          summary: "Deposit P99 latency > 300ms (SLO threshold)"
          description: "Current P99: {{ $value | humanizeDuration }}"
          runbook: "https://wiki.example.com/runbooks/wallet-deposit-latency"

      # Business metric: poucas transações
      - alert: WalletLowTransactionRate
        expr: |
          sum(rate(wallet_transactions_completed_total[15m])) < 0.1
        for: 15m
        labels:
          severity: critical
          team: wallet
        annotations:
          summary: "Quase nenhuma transação nos últimos 15 minutos"
          description: "Rate atual: {{ $value }} txn/sec. Pode indicar problema grave."
          runbook: "https://wiki.example.com/runbooks/wallet-low-txn-rate"
```

- Cada alerta **deve ter** um runbook com:
  - Descrição do problema
  - Impacto no negócio
  - Passos de diagnóstico
  - Passos de remediação
  - Escalation path

```markdown
## Runbook: WalletHighErrorBurnRate

### Descrição
O error budget de disponibilidade está sendo consumido 14.4x mais rápido que o esperado.

### Impacto
- Transações podem estar falhando para os clientes
- Se mantido, esgotará o budget mensal em ~2 dias

### Diagnóstico
1. Verificar dashboard de SLO: [link]
2. Verificar se há deploy recente (annotation no Grafana)
3. Verificar logs de erro: `kubectl logs -l app=wallet-service --tail=100`
4. Verificar dependências: PostgreSQL health, Redis health, Kafka health
5. Verificar métricas de infraestrutura: CPU, memória, connection pool

### Remediação
1. Se correlacionado com deploy → rollback imediato
2. Se dependência down → verificar e restaurar
3. Se resource exhaustion → scale up pods/connections
4. Se bug → hotfix com review prioritário (SLA: 2h)

### Escalation
- P1: @team-wallet (Slack) + PagerDuty
- Se > 30min sem resolução: @sre-oncall + Engineering Manager
```

**Critérios de aceite:**

- [ ] Alertas de burn rate em 3 níveis (critical, warning, ticket)
- [ ] Alerta de latência baseado em SLO
- [ ] Alerta de business metric (low transaction rate)
- [ ] Cada alerta tem runbook com diagnóstico e remediação
- [ ] Alertas testados com exemplos de trigger (simulação)
- [ ] Documento: `prometheus-rules/alerts.yml`

---

### Desafio 6.6 — Dashboards de Observabilidade

**Contexto:** Dashboards são a interface visual entre o time e o sistema. Um bom dashboard
conta uma história: "O sistema está saudável? Se não, onde está o problema?"

**Requisitos:**

- Criar **hierarquia de dashboards**:

| Dashboard | Audiência | Conteúdo |
|---|---|---|
| Executive / Business | Product, C-level | KPIs: transações/min, valor total, taxa de sucesso |
| Service Overview | Dev team, SRE | SLO status, error budget, RED metrics |
| Service Detail | Dev team (debug) | Breakdown: endpoint, DB queries, dependencies |
| Infrastructure | SRE, Platform | CPU, memória, pods, connections, GC |
| Debug / Investigation | Dev team | Traces, logs correlacionados, flame graphs |

- Implementar **Service Overview Dashboard** com os painéis:

| Painel | Prometheus Query | Tipo |
|---|---|---|
| SLO Status | `wallet:availability:ratio_rate5m * 100` | Stat/Gauge |
| Error Budget Remaining | `wallet:error_budget:remaining_ratio * 100` | Stat |
| Request Rate (by status) | `sum(rate(http_requests_total{service="digital-wallet"}[5m])) by (status)` | Time series |
| Error Rate | `rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])` | Time series |
| Latency P50/P95/P99 | `histogram_quantile(0.99, ...)` | Time series |
| Transactions by Type | `sum(rate(wallet_transactions_completed_total[5m])) by (type)` | Time series |
| Deposit Value Distribution | `wallet_deposit_value_brl_bucket` | Histogram |
| Active Wallets | `wallet_active_count` | Stat |
| Top 5 Slow Endpoints | `topk(5, ...)` | Table |
| Recent Deploys | Deploy annotations | Annotations |

- Configurar dashboard como código (**Grafana JSON ou Jsonnet**):

```jsonnet
// dashboards/wallet-overview.jsonnet
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local row = grafana.row;
local singlestat = grafana.singlestat;
local graphPanel = grafana.graphPanel;
local prometheus = grafana.prometheus;

dashboard.new(
  'Digital Wallet — Service Overview',
  schemaVersion=30,
  tags=['wallet', 'slo'],
  time_from='now-1h',
)
.addRow(
  row.new(title='SLO Status')
  .addPanel(
    singlestat.new(
      'Availability SLO',
      datasource='Prometheus',
      format='percentunit',
      thresholds='99.5,99.9',
      colors=['red', 'orange', 'green'],
    )
    .addTarget(
      prometheus.target('wallet:availability:ratio_rate5m')
    )
  )
)
```

- Implementar **teste de health check com verificação de dependências**:

**Java 25 (Spring Boot Actuator):**
```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class HealthCheckTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void healthEndpoint_shouldReturnUpWithDependencies() {
        var response = restTemplate.getForEntity("/actuator/health", Map.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().get("status")).isEqualTo("UP");

        @SuppressWarnings("unchecked")
        Map<String, Object> components = (Map<String, Object>) response.getBody().get("components");
        assertThat(components).containsKeys("db", "diskSpace");
    }

    @Test
    void metricsEndpoint_shouldExposeWalletMetrics() {
        var response = restTemplate.getForEntity("/actuator/prometheus", String.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody())
                .contains("wallet_transactions_completed_total")
                .contains("wallet_transactions_duration_seconds")
                .contains("wallet_deposit_value_brl");
    }
}
```

**Go 1.26:**
```go
func TestHealthEndpoint(t *testing.T) {
    app := setupTestApp(t)

    resp := app.GET(t, "/health", "")

    assert.Equal(t, http.StatusOK, resp.StatusCode)
    var health map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&health)
    assert.Equal(t, "UP", health["status"])
    assert.Contains(t, health, "components")
}

func TestMetricsEndpoint(t *testing.T) {
    app := setupTestApp(t)

    resp := app.GET(t, "/metrics", "")

    assert.Equal(t, http.StatusOK, resp.StatusCode)
    body, _ := io.ReadAll(resp.Body)
    assert.Contains(t, string(body), "wallet_transactions_completed_total")
    assert.Contains(t, string(body), "wallet_transactions_duration_seconds")
}
```

**Critérios de aceite:**

- [ ] Hierarquia de 5 dashboards definida
- [ ] Service Overview dashboard com ≥ 8 painéis
- [ ] Dashboard como código (Jsonnet, JSON ou Terraform)
- [ ] Testes de health check com verificação de dependências
- [ ] Testes de metrics endpoint (expõe métricas customizadas)
- [ ] Deploy annotations configuradas

---

### Desafio 6.7 — Observabilidade no CI/CD e Canary Analysis

**Contexto:** Observabilidade não é apenas produção. O pipeline de CI/CD também precisa
de métricas, e deploys devem ser validados automaticamente comparando métricas
do canary com o baseline.

**Requisitos:**

- Implementar checklist de deploy observável:

```yaml
# deploy-checklist.yaml
pre_deploy:
  - name: "Baseline metrics captured"
    check: "curl -s http://prometheus:9090/api/v1/query?query=wallet:availability:ratio_rate5m"
  - name: "SLO alerts active"
    check: "Verify alert rules loaded in AlertManager"
  - name: "Dashboard accessible"
    check: "curl -sf http://grafana:3000/api/dashboards/uid/wallet-overview"

during_deploy:
  - name: "Deploy annotation created"
    action: |
      curl -X POST http://grafana:3000/api/annotations \
        -H "Content-Type: application/json" \
        -d '{"text": "Deploy v1.2.3", "tags": ["deploy", "wallet"]}'
  - name: "Monitor error rate for 15 min"
    check: "rate(http_requests_total{status=~'5..'}[5m]) / rate(http_requests_total[5m]) < 0.01"

post_deploy:
  - name: "Canary vs baseline comparison"
    check: "Error rate delta < 0.5%, latency P99 delta < 20%"
  - name: "No new error types in logs"
    check: "Compare error types in last 15min vs previous day"
```

- Implementar **DORA metrics** tracking:

| Métrica DORA | Target (Elite) | Como medir |
|---|---|---|
| Deployment Frequency | Múltiplas/dia | Count deploys per day |
| Lead Time for Changes | < 1 hora | Commit timestamp → deploy timestamp |
| Change Failure Rate | < 5% | Deploys que causam rollback / total deploys |
| Mean Time to Recovery (MTTR) | < 1 hora | Incident start → resolution |

- Implementar pipeline de canary analysis:

```yaml
# Argo Rollouts canary analysis
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: wallet-canary-analysis
spec:
  metrics:
    - name: error-rate
      interval: 60s
      count: 10
      successCondition: result < 0.01
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{
              service="digital-wallet",
              version="{{args.canary-version}}",
              status=~"5.."
            }[5m]))
            /
            sum(rate(http_requests_total{
              service="digital-wallet",
              version="{{args.canary-version}}"
            }[5m]))

    - name: latency-p99
      interval: 60s
      count: 10
      successCondition: result < 0.3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{
                service="digital-wallet",
                version="{{args.canary-version}}"
              }[5m])) by (le))

    - name: transaction-success-rate
      interval: 60s
      count: 10
      successCondition: result > 0.99
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(wallet_transactions_completed_total{
              version="{{args.canary-version}}"
            }[5m]))
            /
            (
              sum(rate(wallet_transactions_completed_total{
                version="{{args.canary-version}}"
              }[5m]))
              +
              sum(rate(wallet_transactions_failed_total{
                version="{{args.canary-version}}"
              }[5m]))
            )
```

**Critérios de aceite:**

- [ ] Checklist de deploy observável (pré, durante, pós)
- [ ] DORA metrics definidos com targets
- [ ] Canary analysis template com pelo menos 3 métricas
- [ ] Auto-rollback configurado se métricas violam threshold
- [ ] Deploy annotations integradas ao Grafana
- [ ] Documento: `decisions/06-deploy-observability.md`

---

## Definição de Pronto (DoD) — Observabilidade

- [ ] Logging estruturado em JSON com mascaramento de PII
- [ ] Métricas de negócio expostas via Prometheus
- [ ] Distributed tracing com OpenTelemetry implementado
- [ ] SLIs/SLOs/Error Budget definidos e calculados
- [ ] Alertas baseados em burn rate configurados
- [ ] Dashboards criados (pelo menos Service Overview)
- [ ] Health check com verificação de dependências
- [ ] Canary analysis template definido
- [ ] Commit semântico: `feat(level-6): implement observability and quality monitoring`

---

## Checklist

- [ ] Logging: JSON estruturado com MDC/context (Go + Java)
- [ ] Logging: mascaramento de dados sensíveis
- [ ] Logging: testes de log com contexto e sem PII
- [ ] Métricas: RED + business KPIs implementados
- [ ] Métricas: testes de registro de métricas
- [ ] Tracing: OpenTelemetry integrado com spans e atributos
- [ ] Tracing: teste de propagation e parent-child
- [ ] SLO: definições com targets e error budget policy
- [ ] Alertas: burn rate em 3 níveis + business metric
- [ ] Alertas: runbooks para cada alerta
- [ ] Dashboards: hierarquia definida, overview implementado
- [ ] CI/CD: deploy checklist e canary analysis
- [ ] DORA metrics definidos
