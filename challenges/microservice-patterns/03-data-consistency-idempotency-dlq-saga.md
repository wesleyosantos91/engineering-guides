# Level 3 — Consistência de Dados: Idempotência, DLQ & Saga

> **Objetivo:** Garantir consistência de dados em sistemas distribuídos usando Idempotência para operações seguras,
> Dead Letter Queue para tratamento robusto de falhas em mensageria, e Saga para transações distribuídas.

**Pré-requisito:** [Level 2 — Resiliência II](02-resilience-rate-limiter-bulkhead-composition.md)

**Referências:**
- [07-idempotencia.md](../../.docs/microservice-patterns/07-idempotencia.md)
- [08-dlq.md](../../.docs/microservice-patterns/08-dlq.md)
- [09-saga.md](../../.docs/microservice-patterns/09-saga.md)

---

## Contexto do Domínio

No RideFlow, consistência é crítica:

- **Idempotência** — Pagamento NÃO pode ser cobrado duas vezes; solicitar corrida com a mesma request não cria duplicatas
- **Dead Letter Queue** — Notificações e eventos de GPS que falharam repetidamente vão para DLQ para análise
- **Saga** — Reservar corrida = orquestrar motorista + preço + pagamento + notificação com compensação

---

## Desafios

### Desafio 3.1 — Idempotência: Pagamento Seguro

Implemente idempotência no endpoint de pagamento para garantir que cobranças não sejam duplicadas.

**Cenário:** O passageiro finaliza uma corrida e o sistema processa o pagamento. Se a rede falhar entre enviar a cobrança e receber a confirmação, o retry pode causar cobrança dupla. Idempotência resolve isso.

**Requisitos:**
- `Idempotency-Key` header obrigatório no `POST /payments` (UUID gerado pelo cliente)
- Armazenar o resultado na **mesma transação** que o pagamento (atomicidade)
- Tabela `idempotency_keys` (key, request_hash, response, status, created_at, expires_at)
- TTL de 24h para chaves de idempotência HTTP, 7d para mensageria
- Se a mesma key vier com payload **diferente**: retornar HTTP 422 (Unprocessable Entity)
- Se a mesma key vier com payload **igual**: retornar o resultado original (HTTP 200)
- Validar payload hash para detectar tentativas de replay com dados alterados

**Java 25 (Spring Boot):**
```java
@Entity
@Table(name = "idempotency_keys")
public class IdempotencyRecord {
    @Id
    private String key;
    private String requestHash;
    
    @Column(columnDefinition = "jsonb")
    private String response;
    
    @Enumerated(EnumType.STRING)
    private IdempotencyStatus status; // PROCESSING, COMPLETED, FAILED
    
    private Instant createdAt;
    private Instant expiresAt;
}

@Service
@Transactional
public class IdempotentPaymentService {
    
    private final IdempotencyKeyRepository idempotencyRepo;
    private final PaymentRepository paymentRepo;
    
    public PaymentResult processPayment(String idempotencyKey, PaymentRequest request) {
        var requestHash = DigestUtils.sha256Hex(objectMapper.writeValueAsString(request));
        
        // 1. Verificar se a key já existe
        var existing = idempotencyRepo.findById(idempotencyKey);
        if (existing.isPresent()) {
            var record = existing.get();
            
            // Mesmo key, payload diferente → rejeitar
            if (!record.getRequestHash().equals(requestHash)) {
                throw new IdempotencyConflictException(
                    "Idempotency key reused with different payload");
            }
            
            // Mesmo key, mesmo payload → retornar resultado original
            if (record.getStatus() == IdempotencyStatus.COMPLETED) {
                return objectMapper.readValue(record.getResponse(), PaymentResult.class);
            }
            
            // Ainda processando → retornar 409 Conflict
            if (record.getStatus() == IdempotencyStatus.PROCESSING) {
                throw new IdempotencyInProgressException(
                    "Payment is still being processed");
            }
        }
        
        // 2. Registrar chave como PROCESSING (mesma transação)
        var record = new IdempotencyRecord(
            idempotencyKey, requestHash,
            IdempotencyStatus.PROCESSING,
            Instant.now(), Instant.now().plus(24, ChronoUnit.HOURS));
        idempotencyRepo.save(record);
        
        // 3. Processar pagamento
        try {
            var result = executePayment(request);
            record.setStatus(IdempotencyStatus.COMPLETED);
            record.setResponse(objectMapper.writeValueAsString(result));
            idempotencyRepo.save(record); // na mesma transação
            return result;
        } catch (Exception e) {
            record.setStatus(IdempotencyStatus.FAILED);
            idempotencyRepo.save(record);
            throw e;
        }
    }
}
```

**Go 1.26:**
```go
type IdempotencyRecord struct {
    Key         string    `db:"key"`
    RequestHash string    `db:"request_hash"`
    Response    []byte    `db:"response"` // JSON
    Status      string    `db:"status"`   // PROCESSING, COMPLETED, FAILED
    CreatedAt   time.Time `db:"created_at"`
    ExpiresAt   time.Time `db:"expires_at"`
}

func (s *PaymentService) ProcessPayment(
    ctx context.Context, idempotencyKey string, req PaymentRequest,
) (*PaymentResult, error) {
    requestHash := sha256Hash(req)
    
    // Transação única para idempotência + pagamento
    tx, err := s.db.BeginTxx(ctx, nil)
    if err != nil {
        return nil, fmt.Errorf("begin transaction: %w", err)
    }
    defer tx.Rollback()
    
    // 1. Verificar chave existente (SELECT FOR UPDATE para evitar race condition)
    var existing IdempotencyRecord
    err = tx.GetContext(ctx, &existing,
        "SELECT * FROM idempotency_keys WHERE key = $1 FOR UPDATE", idempotencyKey)
    
    if err == nil {
        // Chave existe
        if existing.RequestHash != requestHash {
            return nil, &IdempotencyConflictError{Key: idempotencyKey}
        }
        if existing.Status == "COMPLETED" {
            var result PaymentResult
            json.Unmarshal(existing.Response, &result)
            return &result, nil
        }
        if existing.Status == "PROCESSING" {
            return nil, &IdempotencyInProgressError{Key: idempotencyKey}
        }
    }
    
    // 2. Inserir como PROCESSING
    _, err = tx.ExecContext(ctx,
        `INSERT INTO idempotency_keys (key, request_hash, status, created_at, expires_at)
         VALUES ($1, $2, 'PROCESSING', NOW(), NOW() + INTERVAL '24 hours')`,
        idempotencyKey, requestHash)
    if err != nil {
        return nil, fmt.Errorf("insert idempotency key: %w", err)
    }
    
    // 3. Processar pagamento (mesma transação)
    result, err := s.executePayment(ctx, tx, req)
    if err != nil {
        tx.ExecContext(ctx,
            "UPDATE idempotency_keys SET status = 'FAILED' WHERE key = $1",
            idempotencyKey)
        tx.Commit()
        return nil, err
    }
    
    // 4. Atualizar como COMPLETED
    responseJSON, _ := json.Marshal(result)
    tx.ExecContext(ctx,
        "UPDATE idempotency_keys SET status = 'COMPLETED', response = $1 WHERE key = $2",
        responseJSON, idempotencyKey)
    
    if err := tx.Commit(); err != nil {
        return nil, fmt.Errorf("commit transaction: %w", err)
    }
    
    return result, nil
}
```

**Critérios de aceite:**
- [ ] `Idempotency-Key` header obrigatório no POST /payments (401 se ausente)
- [ ] Mesma key + mesmo payload → retorna resultado original sem reprocessar
- [ ] Mesma key + payload diferente → HTTP 422 (conflict)
- [ ] Key em PROCESSING → HTTP 409 (Conflict)
- [ ] Idempotência e pagamento na MESMA transação (atomicidade)
- [ ] TTL de 24h com cleanup automático de keys expiradas
- [ ] SELECT FOR UPDATE para evitar race condition em requests concorrentes
- [ ] Testes: enviar mesma requisição 3x → cobra apenas 1x, retorna mesmo resultado

---

### Desafio 3.2 — Dead Letter Queue: Tratamento Robusto de Falhas

Implemente DLQ para mensagens que falharam após todas as tentativas de retry.

**Cenário:** O Notification Service consome eventos `RideCompleted` para enviar push notifications. Quando o serviço de push (Firebase/APNS) está fora, as mensagens falham. Após 3 retries, a mensagem vai para DLQ com metadata de erro.

**Requisitos:**
- Classificação de erros:
  - **Transitório** (503, timeout, connection reset) → retry automático
  - **Irrecuperável** (400, payload inválido, token expirado) → DLQ imediato
  - **Poison message** (parsing falha, formato desconhecido) → DLQ imediato
- Tópico DLQ: `ride-events.dlq` (convenção: `{topic-original}.dlq`)
- Metadata enriquecida na DLQ:
  - `x-original-topic`, `x-error-message`, `x-retry-count`
  - `x-first-failure-at`, `x-sent-to-dlq-at`, `x-error-class`
- 3 estratégias de reprocessamento:
  1. **Manual** — operador revisa e reenvia via API admin
  2. **Automático com cooldown** — job que tenta reprocessar a cada 5 min
  3. **Persist para auditoria** — salva em banco para análise posterior
- Dashboard com métricas: mensagens na DLQ por tipo de erro, taxa de entrada vs reprocessamento

**Java 25 (Spring Kafka):**
```java
@Configuration
public class KafkaConsumerConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerFactory(
            ConsumerFactory<String, String> consumerFactory,
            KafkaTemplate<String, String> kafkaTemplate) {
        
        var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
        factory.setConsumerFactory(consumerFactory);
        
        // Error handler com DLQ
        var errorHandler = new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate,
                (record, ex) -> new TopicPartition(
                    record.topic() + ".dlq", record.partition())),
            new FixedBackOff(1000L, 3) // 3 retries, 1s intervalo
        );
        
        // Erros irrecuperáveis → DLQ imediato (sem retry)
        errorHandler.addNotRetryableExceptions(
            InvalidPayloadException.class,
            DeserializationException.class);
        
        factory.setCommonErrorHandler(errorHandler);
        return factory;
    }
}

@Service
public class NotificationEventConsumer {
    
    @KafkaListener(topics = "ride-events", groupId = "notification-service")
    public void onRideEvent(
            @Payload String payload,
            @Header(KafkaHeaders.RECEIVED_KEY) String key,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
        
        var event = objectMapper.readValue(payload, RideEvent.class);
        
        switch (event.type()) {
            case "RIDE_COMPLETED" -> sendCompletionNotification(event);
            case "RIDE_CANCELLED" -> sendCancellationNotification(event);
            default -> log.warn("Unknown event type: {}", event.type());
        }
    }
}

// API Admin para reprocessar DLQ
@RestController
@RequestMapping("/admin/dlq")
public class DlqAdminController {
    
    @PostMapping("/reprocess")
    public ResponseEntity<DlqReprocessResult> reprocess(
            @RequestParam String topic,
            @RequestParam(defaultValue = "10") int batchSize) {
        var result = dlqReprocessor.reprocess(topic + ".dlq", batchSize);
        return ResponseEntity.ok(result);
    }
    
    @GetMapping("/stats")
    public ResponseEntity<DlqStats> stats(@RequestParam String topic) {
        return ResponseEntity.ok(dlqService.getStats(topic + ".dlq"));
    }
}
```

**Go 1.26 (Watermill):**
```go
type DLQMiddleware struct {
    publisher message.Publisher
    maxRetries int
}

func (m *DLQMiddleware) Middleware(h message.HandlerFunc) message.HandlerFunc {
    return func(msg *message.Message) ([]*message.Message, error) {
        msgs, err := h(msg)
        if err == nil {
            return msgs, nil
        }
        
        retryCount := getRetryCount(msg)
        
        // Classificar erro
        if isIrrecoverable(err) || retryCount >= m.maxRetries {
            // Enviar para DLQ com metadata
            dlqMsg := message.NewMessage(msg.UUID, msg.Payload)
            dlqMsg.Metadata.Set("x-original-topic", msg.Metadata.Get("topic"))
            dlqMsg.Metadata.Set("x-error-message", err.Error())
            dlqMsg.Metadata.Set("x-retry-count", strconv.Itoa(retryCount))
            dlqMsg.Metadata.Set("x-sent-to-dlq-at", time.Now().Format(time.RFC3339))
            dlqMsg.Metadata.Set("x-error-class", classifyError(err))
            
            originalTopic := msg.Metadata.Get("topic")
            dlqTopic := originalTopic + ".dlq"
            
            if pubErr := m.publisher.Publish(dlqTopic, dlqMsg); pubErr != nil {
                slog.Error("failed to publish to DLQ",
                    "topic", dlqTopic, "error", pubErr.Error())
                return nil, pubErr // não perder a mensagem
            }
            
            slog.Warn("message sent to DLQ",
                "topic", dlqTopic, "retries", retryCount, "error", err.Error())
            
            return nil, nil // mensagem na DLQ, não re-throw
        }
        
        // Transitório: incrementar retry e re-throw para retry
        msg.Metadata.Set("x-retry-count", strconv.Itoa(retryCount+1))
        return nil, err
    }
}

func isIrrecoverable(err error) bool {
    var httpErr *HTTPError
    if errors.As(err, &httpErr) {
        return httpErr.StatusCode >= 400 && httpErr.StatusCode < 500
    }
    var parseErr *json.SyntaxError
    return errors.As(err, &parseErr)
}
```

**Critérios de aceite:**
- [ ] Classificação de erros: transitório (retry), irrecuperável (DLQ), poison (DLQ)
- [ ] Tópico DLQ nomeado `{topic-original}.dlq`
- [ ] Metadata enriquecida: original-topic, error-message, retry-count, timestamps
- [ ] 3 retries antes de enviar para DLQ (erros transitórios)
- [ ] Erros irrecuperáveis vão para DLQ sem retry
- [ ] API admin para visualizar e reprocessar mensagens da DLQ
- [ ] Métricas: contagem por tipo de erro, taxa entrada vs reprocessamento
- [ ] Testes: mensagem com erro transitório → 3 retries → DLQ; mensagem irrecuperável → DLQ imediato

---

### Desafio 3.3 — Saga Orquestrada: Booking de Corrida

Implemente o padrão Saga (orquestração) para o fluxo completo de booking de corrida.

**Cenário:** Solicitar uma corrida envolve 4 serviços que devem ser coordenados:
1. **Reservar motorista** (Driver Service) — marcar como BUSY
2. **Calcular preço** (Pricing Service) — preço dinâmico
3. **Reter pagamento** (Payment Service) — hold no cartão
4. **Confirmar corrida** (Ride Service) — status MATCHED

Se qualquer passo falhar, os anteriores devem ser compensados (rollback distribuído).

**Requisitos:**
- **Orquestrador central** (`RideBookingSagaOrchestrator`) coordena os 4 passos
- Cada passo tem uma **ação** e uma **compensação**:
  - Reservar motorista → Liberar motorista
  - Calcular preço → (sem compensação, operação read-only)
  - Reter pagamento → Cancelar hold
  - Confirmar corrida → Cancelar corrida
- **Máquina de estados da saga:**
  ```
  STARTED → DRIVER_RESERVED → PRICE_CALCULATED → PAYMENT_HELD → COMPLETED
       ↓              ↓                ↓               ↓
  FAILED   COMPENSATING_DRIVER  COMPENSATING_PRICE  COMPENSATING_PAYMENT
                     ↓                ↓               ↓
                COMPENSATED      COMPENSATED      COMPENSATED
  ```
- Compensações executam na **ordem reversa** (payment → price → driver)
- Compensações são **idempotentes** (podem ser chamadas mais de uma vez)
- Timeout da saga: 30 segundos (se não completar, compensar)
- Persistir estado da saga no banco (para recovery após crash)

**Java 25 (Spring Boot):**
```java
@Entity
@Table(name = "ride_booking_sagas")
public class RideBookingSaga {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    private UUID rideId;
    private UUID passengerId;
    
    @Enumerated(EnumType.STRING)
    private SagaState state;
    
    private UUID reservedDriverId;
    private BigDecimal calculatedPrice;
    private UUID paymentHoldId;
    
    @Column(columnDefinition = "jsonb")
    private String stepResults; // resultado de cada passo
    
    private Instant startedAt;
    private Instant completedAt;
    private String failureReason;
}

@Service
public class RideBookingSagaOrchestrator {
    
    private final SagaRepository sagaRepo;
    private final DriverServiceClient driverClient;
    private final PricingServiceClient pricingClient;
    private final PaymentServiceClient paymentClient;
    private final RideServiceClient rideClient;
    
    @Transactional
    public RideBookingResult execute(RideBookingRequest request) {
        var saga = createSaga(request);
        
        try {
            // Step 1: Reservar motorista
            saga.setState(SagaState.RESERVING_DRIVER);
            sagaRepo.save(saga);
            var driver = driverClient.reserveNearestDriver(
                request.origin(), request.vehicleCategory());
            saga.setReservedDriverId(driver.id());
            
            // Step 2: Calcular preço
            saga.setState(SagaState.CALCULATING_PRICE);
            sagaRepo.save(saga);
            var price = pricingClient.calculatePrice(
                request.origin(), request.destination(), request.vehicleCategory());
            saga.setCalculatedPrice(price.amount());
            
            // Step 3: Reter pagamento
            saga.setState(SagaState.HOLDING_PAYMENT);
            sagaRepo.save(saga);
            var hold = paymentClient.holdPayment(
                request.passengerId(), price.amount(), saga.getId().toString());
            saga.setPaymentHoldId(hold.holdId());
            
            // Step 4: Confirmar corrida
            saga.setState(SagaState.CONFIRMING_RIDE);
            sagaRepo.save(saga);
            rideClient.confirmRide(saga.getRideId(), driver.id(), price.amount());
            
            // Sucesso
            saga.setState(SagaState.COMPLETED);
            saga.setCompletedAt(Instant.now());
            sagaRepo.save(saga);
            
            return RideBookingResult.success(saga);
            
        } catch (Exception e) {
            saga.setFailureReason(e.getMessage());
            compensate(saga);
            return RideBookingResult.failed(saga);
        }
    }
    
    private void compensate(RideBookingSaga saga) {
        saga.setState(SagaState.COMPENSATING);
        sagaRepo.save(saga);
        
        // Compensar na ordem reversa
        try {
            if (saga.getPaymentHoldId() != null) {
                paymentClient.cancelHold(saga.getPaymentHoldId());
            }
        } catch (Exception e) {
            log.error("Compensation failed for payment hold: {}", e.getMessage());
        }
        
        try {
            if (saga.getReservedDriverId() != null) {
                driverClient.releaseDriver(saga.getReservedDriverId());
            }
        } catch (Exception e) {
            log.error("Compensation failed for driver: {}", e.getMessage());
        }
        
        saga.setState(SagaState.COMPENSATED);
        saga.setCompletedAt(Instant.now());
        sagaRepo.save(saga);
    }
}
```

**Go 1.26:**
```go
type SagaStep struct {
    Name       string
    Execute    func(ctx context.Context, saga *RideBookingSaga) error
    Compensate func(ctx context.Context, saga *RideBookingSaga) error
}

type RideBookingSagaOrchestrator struct {
    db            *sqlx.DB
    steps         []SagaStep
    driverClient  *DriverServiceClient
    pricingClient *PricingServiceClient
    paymentClient *PaymentServiceClient
    rideClient    *RideServiceClient
}

func NewOrchestrator(/* dependencies */) *RideBookingSagaOrchestrator {
    o := &RideBookingSagaOrchestrator{/* ... */}
    
    o.steps = []SagaStep{
        {
            Name: "reserve_driver",
            Execute: func(ctx context.Context, saga *RideBookingSaga) error {
                driver, err := o.driverClient.ReserveNearest(ctx,
                    saga.Origin, saga.VehicleCategory)
                if err != nil {
                    return fmt.Errorf("reserve driver: %w", err)
                }
                saga.ReservedDriverID = &driver.ID
                return nil
            },
            Compensate: func(ctx context.Context, saga *RideBookingSaga) error {
                if saga.ReservedDriverID == nil {
                    return nil // nada para compensar
                }
                return o.driverClient.ReleaseDriver(ctx, *saga.ReservedDriverID)
            },
        },
        {
            Name: "calculate_price",
            Execute: func(ctx context.Context, saga *RideBookingSaga) error {
                price, err := o.pricingClient.Calculate(ctx,
                    saga.Origin, saga.Destination, saga.VehicleCategory)
                if err != nil {
                    return fmt.Errorf("calculate price: %w", err)
                }
                saga.CalculatedPrice = &price.Amount
                return nil
            },
            Compensate: nil, // read-only, sem compensação
        },
        {
            Name: "hold_payment",
            Execute: func(ctx context.Context, saga *RideBookingSaga) error {
                hold, err := o.paymentClient.HoldPayment(ctx,
                    saga.PassengerID, *saga.CalculatedPrice, saga.ID.String())
                if err != nil {
                    return fmt.Errorf("hold payment: %w", err)
                }
                saga.PaymentHoldID = &hold.HoldID
                return nil
            },
            Compensate: func(ctx context.Context, saga *RideBookingSaga) error {
                if saga.PaymentHoldID == nil {
                    return nil
                }
                return o.paymentClient.CancelHold(ctx, *saga.PaymentHoldID)
            },
        },
        {
            Name: "confirm_ride",
            Execute: func(ctx context.Context, saga *RideBookingSaga) error {
                return o.rideClient.ConfirmRide(ctx,
                    saga.RideID, *saga.ReservedDriverID, *saga.CalculatedPrice)
            },
            Compensate: func(ctx context.Context, saga *RideBookingSaga) error {
                return o.rideClient.CancelRide(ctx, saga.RideID)
            },
        },
    }
    
    return o
}

func (o *RideBookingSagaOrchestrator) Execute(
    ctx context.Context, req RideBookingRequest,
) (*RideBookingResult, error) {
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    
    saga := &RideBookingSaga{
        ID:          uuid.New(),
        RideID:      req.RideID,
        PassengerID: req.PassengerID,
        State:       SagaStateStarted,
        StartedAt:   time.Now(),
    }
    o.saveSaga(ctx, saga)
    
    completedSteps := 0
    for i, step := range o.steps {
        saga.State = SagaState("EXECUTING_" + strings.ToUpper(step.Name))
        o.saveSaga(ctx, saga)
        
        if err := step.Execute(ctx, saga); err != nil {
            saga.FailureReason = err.Error()
            o.compensate(ctx, saga, i-1)
            return &RideBookingResult{Status: "FAILED", Saga: saga}, nil
        }
        completedSteps = i
    }
    
    saga.State = SagaStateCompleted
    saga.CompletedAt = timePtr(time.Now())
    o.saveSaga(ctx, saga)
    
    return &RideBookingResult{Status: "SUCCESS", Saga: saga}, nil
}

func (o *RideBookingSagaOrchestrator) compensate(
    ctx context.Context, saga *RideBookingSaga, lastCompleted int,
) {
    saga.State = SagaStateCompensating
    o.saveSaga(ctx, saga)
    
    // Compensar na ordem reversa
    for i := lastCompleted; i >= 0; i-- {
        step := o.steps[i]
        if step.Compensate == nil {
            continue
        }
        if err := step.Compensate(ctx, saga); err != nil {
            slog.Error("compensation failed",
                "step", step.Name, "saga", saga.ID, "error", err.Error())
            // Continuar compensando os outros passos
        }
    }
    
    saga.State = SagaStateCompensated
    saga.CompletedAt = timePtr(time.Now())
    o.saveSaga(ctx, saga)
}
```

**Critérios de aceite:**
- [ ] Saga com 4 passos: reservar motorista → calcular preço → reter pagamento → confirmar corrida
- [ ] Cada passo tem ação + compensação (exceto read-only)
- [ ] Compensações executam na ordem reversa quando qualquer passo falha
- [ ] Compensações são idempotentes (chamar 2x não causa problema)
- [ ] Estado da saga persistido no banco (recovery após crash)
- [ ] Timeout da saga: 30 segundos
- [ ] Testes: happy path (4 passos OK), falha no step 3 (compensa steps 2,1), falha no step 1 (sem compensação)

---

### Desafio 3.4 — Saga Coreografada: Notificações por Eventos

Implemente uma variação da saga como coreografia para fluxo de notificações.

**Cenário:** Quando uma corrida é completada, vários serviços reagem ao evento `RideCompleted`:
1. Payment Service captura o pagamento (converte hold em cobrança)
2. Notification Service envia recibo ao passageiro
3. Driver Service atualiza rating e ganhos do motorista
4. Analytics Service registra métricas da corrida

Neste caso, não há orquestrador central — cada serviço reage ao evento e publica seu resultado.

**Requisitos:**
- Eventos publicados no Kafka: `RideCompleted`, `PaymentCaptured`, `NotificationSent`, `DriverStatsUpdated`
- Cada serviço consome o evento relevante e publica o resultado
- Sem orquestrador central — serviços são independentes
- Contraste com saga orquestrada: quando usar cada abordagem
- Documentar no `DECISIONS.md` os trade-offs:
  - Orquestração: visibilidade centralizada, mais fácil de debugar, acoplamento ao orquestrador
  - Coreografia: serviços independentes, escalável, difícil de debugar, complexidade emergente

**Java 25 (Spring Kafka Events):**
```java
// Payment Service reage a RideCompleted
@Service
public class PaymentEventHandler {
    
    @KafkaListener(topics = "ride-events", 
                   groupId = "payment-service",
                   containerFactory = "kafkaListenerFactory")
    public void onRideEvent(@Payload RideEvent event) {
        if (!"RIDE_COMPLETED".equals(event.type())) return;
        
        var rideCompleted = objectMapper.convertValue(event.data(), RideCompletedData.class);
        
        var captureResult = paymentService.captureHold(rideCompleted.paymentHoldId());
        
        // Publicar resultado como novo evento
        var paymentEvent = new PaymentEvent("PAYMENT_CAPTURED", Map.of(
            "rideId", rideCompleted.rideId(),
            "amount", captureResult.amount(),
            "capturedAt", Instant.now().toString()
        ));
        kafkaTemplate.send("payment-events", 
            rideCompleted.rideId().toString(), paymentEvent);
    }
}
```

**Go 1.26 (Watermill Events):**
```go
// Payment Service reage a RideCompleted
func (h *PaymentEventHandler) HandleRideCompleted(msg *message.Message) error {
    var event RideEvent
    if err := json.Unmarshal(msg.Payload, &event); err != nil {
        return fmt.Errorf("unmarshal ride event: %w", err)
    }
    
    if event.Type != "RIDE_COMPLETED" {
        return nil // ignorar outros tipos
    }
    
    captureResult, err := h.paymentService.CaptureHold(
        msg.Context(), event.Data.PaymentHoldID)
    if err != nil {
        return fmt.Errorf("capture hold: %w", err)
    }
    
    // Publicar resultado como novo evento
    paymentEvent := PaymentEvent{
        Type: "PAYMENT_CAPTURED",
        Data: map[string]any{
            "rideId":     event.Data.RideID,
            "amount":     captureResult.Amount,
            "capturedAt": time.Now().Format(time.RFC3339),
        },
    }
    
    payload, _ := json.Marshal(paymentEvent)
    return h.publisher.Publish("payment-events",
        message.NewMessage(watermill.NewUUID(), payload))
}
```

**Critérios de aceite:**
- [ ] Evento `RideCompleted` consumido por 3 serviços independentes
- [ ] Cada serviço publica seu resultado como novo evento
- [ ] Sem orquestrador central — coreografia pura
- [ ] Comparação documentada: orquestração vs coreografia com critérios de decisão
- [ ] Tabela: quando usar orquestração (>3 passos, compensação complexa) vs coreografia (2-3 passos, serviços independentes)
- [ ] Testes: publicar RideCompleted → verificar que todos os 3 consumers processaram
- [ ] Lidar com evento duplicado (consumer deve ser idempotente)

---

### Desafio 3.5 — Saga Recovery: Tratando Sagas Pendentes

Implemente mecanismo de recovery para sagas que ficaram em estado intermediário (crash, timeout).

**Cenário:** O orquestrador da saga pode crashar entre os passos. Quando reiniciar, precisa:
1. Encontrar sagas pendentes (estado != COMPLETED e != COMPENSATED)
2. Decidir se continua ou compensa baseado no último estado
3. Reexecutar com as mesmas garantias de idempotência

**Requisitos:**
- Job periódico (a cada 1 minuto) que busca sagas pendentes (startedAt > 5 min atrás)
- Regra de recovery: se último estado < PAYMENT_HELD → compensar; se >= PAYMENT_HELD → tentar completar
- Cada passo da saga deve ser idempotente (recovery pode chamar novamente)
- Alertas quando sagas ficam pendentes por mais de 10 minutos
- Dashboard mostrando sagas por estado (STARTED, COMPENSATING, COMPLETED, etc.)

**Java 25:**
```java
@Scheduled(fixedRate = 60_000) // a cada 1 minuto
public void recoverPendingSagas() {
    var cutoff = Instant.now().minus(5, ChronoUnit.MINUTES);
    var pending = sagaRepo.findPendingSagasBefore(cutoff);
    
    for (var saga : pending) {
        log.warn("Recovering saga {} in state {}", saga.getId(), saga.getState());
        
        try {
            if (shouldCompensate(saga.getState())) {
                compensate(saga);
            } else {
                resume(saga);
            }
        } catch (Exception e) {
            log.error("Failed to recover saga {}: {}", saga.getId(), e.getMessage());
            if (saga.getStartedAt().isBefore(Instant.now().minus(10, ChronoUnit.MINUTES))) {
                alertService.sendAlert("Saga stuck: " + saga.getId());
            }
        }
    }
}
```

**Critérios de aceite:**
- [ ] Job periódico identifica sagas pendentes (> 5 min sem completar)
- [ ] Recovery decide corretamente: compensar ou resumir baseado no estado
- [ ] Cada operação de recovery é idempotente
- [ ] Alerta quando saga pendente > 10 minutos
- [ ] Logs detalhados de cada recovery (saga ID, estado anterior, ação tomada)
- [ ] Testes: simular crash do orquestrador → reiniciar → sagas pendentes são recuperadas
