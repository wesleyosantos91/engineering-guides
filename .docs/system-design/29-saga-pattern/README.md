# 29. Saga Pattern

> **Categoria:** Padrões Arquiteturais  
> **Nível:** Avançado — fundamental para transações distribuídas  
> **Complexidade:** Alta

---

## Definição

O **Saga Pattern** é um padrão para gerenciar **transações distribuídas** em microservices, onde uma **transação ACID global não é possível**. Uma saga é uma sequência de **transações locais**, onde cada step publica um evento que trigger o próximo step. Se um step falha, **compensating transactions** são executadas para desfazer os steps anteriores.

---

## Por Que é Importante?

- **Microservices não compartilham banco** — cada serviço tem seu DB → sem transação ACID global
- **2PC (Two-Phase Commit) não escala** — bloqueante, ponto único de falha
- **Pattern padrão em Big Techs** — Uber, Amazon, Netflix, Airbnb
- **Pergunta clássica de entrevista** — "Como garantir consistência entre microservices?"

---

## O Problema

```
Monolith (transação ACID simples):

  BEGIN TRANSACTION
    INSERT INTO orders (...)        -- Criar pedido
    UPDATE inventory SET qty = ...  -- Reservar estoque
    INSERT INTO payments (...)      -- Cobrar pagamento
    INSERT INTO shipments (...)     -- Agendar envio
  COMMIT
  
  → Se qualquer step falha, ROLLBACK automático ✅

Microservices (cada serviço tem seu DB):

  Order Service    → orders_db
  Inventory Service → inventory_db    ❌ Sem transação global!
  Payment Service  → payments_db
  Shipping Service → shipping_db
  
  → Se Payment falha DEPOIS de Inventory reservar,
    como desfazer a reserva de estoque?
```

---

## Dois Tipos de Saga

### 1. Choreography (Coreografia)

```
Cada serviço REAGE a eventos e publica eventos. Sem coordenador central.

  Order        Inventory       Payment        Shipping
    │              │               │              │
    │──OrderCreated──▶             │              │
    │              │               │              │
    │         InventoryReserved───▶│              │
    │              │               │              │
    │              │          PaymentCharged─────▶│
    │              │               │              │
    │              │               │         ShipmentCreated
    │◀─────────────────────────────────────── │
    │          OrderCompleted                  │

Compensação (se Payment falha):

  Order        Inventory       Payment
    │              │               │
    │──OrderCreated──▶             │
    │              │               │
    │         InventoryReserved───▶│
    │              │               │
    │              │          PaymentFailed!
    │              │               │
    │         ◀──InventoryReleased │   ← Compensação!
    │              │               │
    │◀──OrderCancelled             │   ← Compensação!

Prós: Simples, desacoplado, nenhum SPOF
Contras: Difícil visualizar fluxo completo, difícil debugar
```

### 2. Orchestration (Orquestração)

```
Um ORQUESTRADOR central coordena os steps da saga.

                  ┌──────────────────┐
                  │  Saga Orchestrator│
                  │  (Order Saga)     │
                  └────────┬─────────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
     ┌────────────┐ ┌───────────┐ ┌────────────┐
     │  Inventory │ │  Payment  │ │  Shipping  │
     │  Service   │ │  Service  │ │  Service   │
     └────────────┘ └───────────┘ └────────────┘

Fluxo:
  1. Orchestrator: "Inventory, reserve items" → OK
  2. Orchestrator: "Payment, charge customer" → OK
  3. Orchestrator: "Shipping, create shipment" → OK
  4. Orchestrator: "Order = COMPLETED"

Compensação:
  1. Orchestrator: "Inventory, reserve items" → OK
  2. Orchestrator: "Payment, charge customer" → FAILED!
  3. Orchestrator: "Inventory, RELEASE items" (compensação)
  4. Orchestrator: "Order = CANCELLED"

Prós: Fluxo claro e centralizado, fácil debugar
Contras: Orchestrator pode ser SPOF, mais acoplamento
```

---

## Compensating Transactions

```
Compensação ≠ Rollback

  Rollback (ACID): desfaz como se nunca tivesse acontecido
  Compensação:     executa ação INVERSA (semanticamente)

  Step                  │ Compensação
  ──────────────────────┼─────────────────────────
  ReserveInventory()    │ ReleaseInventory()
  ChargePayment()       │ RefundPayment()
  CreateShipment()      │ CancelShipment()
  SendEmail()           │ SendCancellationEmail()
  DebitAccount()        │ CreditAccount()

  Importante:
  - Compensações devem ser IDEMPOTENTES
  - Compensações podem FALHAR (retry necessário)
  - Nem tudo é compensável (ex: email enviado não "desenvia")
    → Pivot transaction: point of no return
```

### Tipos de Steps

```
Compensable:    Pode ser desfeito (tem compensação)
                ex: ReserveInventory → ReleaseInventory

Pivot:          Point of no return. Se funcionar, saga continua.
                Se falhar, compensar tudo anterior.
                ex: ChargePayment (se cobrou, não volta)

Retriable:      Garantido que eventualmente funciona (retry)
                ex: SendConfirmationEmail (vai enviar em algum momento)

Ordem ideal em uma Saga:
  [Compensable steps] → [Pivot step] → [Retriable steps]
  
  Reserve Inventory → Charge Payment → Create Shipment → Send Email
  (compensable)       (pivot)          (retriable)       (retriable)
```

---

## Implementação: Choreography

### Com Kafka (Spring Boot)

```java
// ─── STEP 1: Order Service cria pedido e publica evento ───
@Service
public class OrderService {
    
    @Transactional
    public Order createOrder(CreateOrderCommand cmd) {
        Order order = Order.create(cmd);
        order.setStatus(OrderStatus.PENDING);
        orderRepository.save(order);
        
        // Publica evento → Inventory escuta
        eventPublisher.publish(new OrderCreatedEvent(
            order.getId(), order.getItems()
        ));
        
        return order;
    }
    
    // Listener para compensação
    @KafkaListener(topics = "payment-events")
    public void handlePaymentFailed(PaymentFailedEvent event) {
        Order order = orderRepository.findById(event.getOrderId());
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
    }
}

// ─── STEP 2: Inventory Service reserva e publica evento ───
@Service
public class InventoryService {
    
    @KafkaListener(topics = "order-events")
    @Transactional
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            inventoryRepository.reserveItems(event.getItems());
            
            eventPublisher.publish(new InventoryReservedEvent(
                event.getOrderId(), event.getItems()
            ));
        } catch (InsufficientStockException e) {
            eventPublisher.publish(new InventoryReservationFailedEvent(
                event.getOrderId(), e.getMessage()
            ));
        }
    }
    
    // Compensação
    @KafkaListener(topics = "payment-events")
    public void handlePaymentFailed(PaymentFailedEvent event) {
        inventoryRepository.releaseItems(event.getOrderId());
    }
}

// ─── STEP 3: Payment Service cobra e publica ───
@Service
public class PaymentService {
    
    @KafkaListener(topics = "inventory-events")
    @Transactional
    public void handleInventoryReserved(InventoryReservedEvent event) {
        try {
            paymentGateway.charge(event.getOrderId(), event.getTotal());
            
            eventPublisher.publish(new PaymentChargedEvent(
                event.getOrderId()
            ));
        } catch (PaymentDeclinedException e) {
            eventPublisher.publish(new PaymentFailedEvent(
                event.getOrderId(), e.getMessage()
            ));
        }
    }
}
```

---

## Implementação: Orchestration

### Saga Orchestrator

```java
@Component
public class CreateOrderSaga {
    
    private final SagaManager sagaManager;
    
    // Define os steps da saga
    private final SagaDefinition<CreateOrderSagaState> sagaDefinition =
        SagaDefinition.<CreateOrderSagaState>builder()
            .step()
                .invokeParticipant(this::reserveInventory)
                .withCompensation(this::releaseInventory)
            .step()
                .invokeParticipant(this::chargePayment)
                .withCompensation(this::refundPayment)
            .step()
                .invokeParticipant(this::createShipment)
                // Retriable — sem compensação
            .step()
                .invokeParticipant(this::confirmOrder)
            .build();
    
    public void execute(CreateOrderCommand cmd) {
        CreateOrderSagaState state = new CreateOrderSagaState(cmd);
        sagaManager.execute(sagaDefinition, state);
    }
    
    // ─── Participants ───
    
    private CommandMessage reserveInventory(CreateOrderSagaState state) {
        return new ReserveInventoryCommand(
            state.getOrderId(), state.getItems()
        );
    }
    
    private CommandMessage releaseInventory(CreateOrderSagaState state) {
        return new ReleaseInventoryCommand(
            state.getOrderId(), state.getItems()
        );
    }
    
    private CommandMessage chargePayment(CreateOrderSagaState state) {
        return new ChargePaymentCommand(
            state.getOrderId(), state.getTotal()
        );
    }
    
    private CommandMessage refundPayment(CreateOrderSagaState state) {
        return new RefundPaymentCommand(
            state.getOrderId(), state.getPaymentId()
        );
    }
    
    private CommandMessage createShipment(CreateOrderSagaState state) {
        return new CreateShipmentCommand(
            state.getOrderId(), state.getAddress()
        );
    }
    
    private CommandMessage confirmOrder(CreateOrderSagaState state) {
        return new ConfirmOrderCommand(state.getOrderId());
    }
}
```

### State Machine do Orchestrator

```
Saga states:

  ┌─────────┐    reserve    ┌──────────────────┐
  │ STARTED │─────OK───────▶│INVENTORY_RESERVED│
  └─────────┘               └────────┬─────────┘
       │                             │
    reserve                     charge payment
     FAILED                          │
       │                    ┌────────▼─────────┐
       ▼                    │ PAYMENT_CHARGED   │
  ┌──────────┐              └────────┬─────────┘
  │ CANCELLED│                       │
  └──────────┘                  create shipment
       ▲                             │
       │                    ┌────────▼─────────┐
    payment                 │SHIPMENT_CREATED   │
     FAILED                 └────────┬─────────┘
    (compensate                      │
     inventory)                 confirm order
                                     │
                            ┌────────▼─────────┐
                            │   COMPLETED       │
                            └──────────────────┘

Compensação path (payment failed):
  PAYMENT_FAILED → release inventory → COMPENSATING 
                 → order cancelled → CANCELLED
```

---

## Choreography vs Orchestration

| Aspecto | Choreography | Orchestration |
|---------|-------------|---------------|
| **Coordenação** | Distribuída (events) | Centralizada (orchestrator) |
| **Acoplamento** | Baixo | Médio (orchestrator conhece steps) |
| **Visibilidade** | Difícil ver fluxo completo | Fácil (state machine) |
| **SPOF** | Nenhum | Orchestrator pode falhar |
| **Complexidade** | Cresce rápido com +steps | Controlada no orchestrator |
| **Debugging** | Difícil (trace distribuído) | Mais fácil (estado centralizado) |
| **Ideal para** | 2-3 serviços | 4+ serviços com fluxo complexo |
| **Evolução** | Difícil adicionar/remover steps | Fácil (modifica orchestrator) |

### Quando Usar Cada Um

```
Choreography:
  ✅ Poucos serviços (2-4)
  ✅ Fluxo simples e linear
  ✅ Equipes independentes querem total desacoplamento
  ✅ Volume muito alto (sem bottleneck centralizado)

Orchestration:
  ✅ Muitos serviços (4+)
  ✅ Fluxo complexo com branches e conditions
  ✅ Necessidade de visibilidade e monitoramento
  ✅ Compensações complexas com ordering
  ✅ Business analysts precisam entender o fluxo
```

---

## Saga + Outbox Pattern

```
Problema: publish event + save to DB deve ser atômico.
  Se salvo no DB mas event não publica → inconsistência.

Solução: Outbox Pattern (ver tópico 30)

  @Transactional
  public void createOrder(CreateOrderCommand cmd) {
      Order order = orderRepository.save(new Order(cmd));
      
      // Salva evento na OUTBOX TABLE (mesma transação!)
      outboxRepository.save(new OutboxEvent(
          "OrderCreated", 
          serialize(new OrderCreatedEvent(order))
      ));
  }
  // Background: Outbox Relay lê outbox e publica no Kafka
```

---

## AWS Step Functions (Saga as a Service)

```json
{
  "Comment": "Order Processing Saga",
  "StartAt": "ReserveInventory",
  "States": {
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:*:*:function:reserve-inventory",
      "Catch": [{
        "ErrorEquals": ["InsufficientStock"],
        "Next": "CancelOrder"
      }],
      "Next": "ChargePayment"
    },
    "ChargePayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:*:*:function:charge-payment",
      "Catch": [{
        "ErrorEquals": ["PaymentDeclined"],
        "Next": "ReleaseInventory"
      }],
      "Next": "CreateShipment"
    },
    "CreateShipment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:*:*:function:create-shipment",
      "Retry": [{"ErrorEquals": ["States.ALL"], "MaxAttempts": 3}],
      "Next": "OrderCompleted"
    },
    "OrderCompleted": {
      "Type": "Succeed"
    },
    "ReleaseInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:*:*:function:release-inventory",
      "Next": "CancelOrder"
    },
    "CancelOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:*:*:function:cancel-order",
      "Next": "OrderFailed"
    },
    "OrderFailed": {
      "Type": "Fail",
      "Error": "SagaFailed",
      "Cause": "Order processing saga failed"
    }
  }
}
```

---

## Monitoramento de Sagas

```
Saga pode ficar stuck. Monitoramento é CRÍTICO.

  Saga Dashboard:
  ┌──────────────────────────────────────────────────────┐
  │ Active Sagas: 1,234                                  │
  │ Completed (24h): 45,678                              │
  │ Failed (24h): 12  ⚠️                                 │
  │ Compensating: 3                                      │
  │ Stuck (>5min): 1  🔴                                 │
  │                                                      │
  │ Avg Duration: 2.3s                                   │
  │ P99 Duration: 8.5s                                   │
  │ Compensation Rate: 0.03%                             │
  └──────────────────────────────────────────────────────┘

  Alertas:
  - Saga stuck por mais de X minutos
  - Compensation rate > threshold
  - Saga duration P99 > SLA
  - Dead letter queue growing
```

---

## Uso em Big Techs

| Empresa | Tipo | Cenário | Detalhes |
|---------|------|---------|----------|
| **Uber** | Orchestration | Trip booking | Reserve driver → Start trip → Process payment → Rate |
| **Amazon** | Orchestration | Order processing | Reserve → Pay → Ship → Deliver |
| **Netflix** | Choreography | Content pipeline | Ingest → Encode → Validate → Publish |
| **Airbnb** | Orchestration | Booking saga | Reserve dates → Process payment → Confirm host |
| **Stripe** | Orchestration | Payment flow | Authorize → Capture → Transfer → Payout |
| **Shopify** | Hybrid | Checkout | Choreography para inventory + Orchestration para payment |

### Uber Trip Saga

```
Orchestration Saga:

  ┌─────────────────────────────┐
  │ Trip Saga Orchestrator      │
  └──────────┬──────────────────┘
             │
  Step 1: MatchDriver
    → OK: DriverAssigned
    → FAIL: NotifyRider("no drivers")
             │
  Step 2: ConfirmPickup
    → OK: TripStarted
    → TIMEOUT: CancelMatch (compensação)
             │
  Step 3: ProcessTrip
    → OK: TripCompleted
    → FAIL: retry (retriable)
             │
  Step 4: ChargeRider (pivot)
    → OK: PaymentProcessed
    → FAIL: FlagForManualReview
             │
  Step 5: PayDriver (retriable)
    → Retry until success
             │
  Step 6: SendReceipts (retriable)
    → Retry until success
             
  Compensação (se ConfirmPickup timeout):
    CancelMatch → NotifyDriver → NotifyRider → Rematch
```

---

## Perguntas Frequentes em Entrevistas

1. **"O que é o Saga Pattern?"**
   - Sequência de transações locais para consistência distribuída
   - Cada step gera evento para o próximo
   - Se falha, compensating transactions desfazem steps anteriores

2. **"Choreography vs Orchestration?"**
   - Choreography: eventos distribuídos, sem coordenador central
   - Orchestration: coordenador central gerencia steps
   - Choreography para simples, Orchestration para complexo

3. **"Compensação é o mesmo que rollback?"**
   - Não. Rollback desfaz como se nunca existiu (ACID)
   - Compensação é uma NOVA transação que semanticamente reverte
   - Exemplo: RefundPayment() não é rollback, é novo crédito

4. **"O que é pivot transaction?"**
   - Ponto de não-retorno na saga
   - Antes: steps compensáveis (pode desfazer)
   - Pivot: se falha, compensa tudo; se sucede, continua
   - Depois: steps retriáveis (retry até sucesso)

5. **"Como lidar com saga stuck?"**
   - Timeout em cada step
   - Monitoring de sagas ativas por tempo
   - Dead letter queue para steps que falharam
   - Manual intervention dashboard

---

## Referências

- Hector Garcia-Molina & Kenneth Salem (1987) — *"Sagas"* — Princeton
- Chris Richardson — *"Microservices Patterns"*, Cap. 4: Managing transactions with sagas
- Caitie McCaffrey — *"Distributed Sagas: A Protocol for Coordinating Microservices"*
- Uber Engineering — *"Cadence: The Scalable Workflow Orchestration Platform"*
- AWS — *"Implement the serverless saga pattern by using AWS Step Functions"*
- Eventuate.io — Saga Framework (Chris Richardson)
