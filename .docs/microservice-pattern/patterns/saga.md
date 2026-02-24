# Saga Pattern

> **Categoria:** Persistência e Processamento de Dados
> **Origem:** Hector Garcia-Molina & Kenneth Salem (1987)
> **Complementa:** Idempotência, Outbox Pattern, DLQ
> **Keywords:** saga, transação distribuída, orquestração, coreografia, compensação, consistência eventual

---

## Problema

Uma operação de negócio envolve **múltiplos microsserviços**, cada um com seu próprio banco de dados. Transações distribuídas (2PC — Two-Phase Commit) são inviáveis em microsserviços porque:

- **Alto acoplamento** — todos os participantes devem estar disponíveis simultaneamente
- **Alta latência** — locks são mantidos em todos os bancos durante a coordenação
- **Single point of failure** — o coordenador do 2PC é crítico
- **Não escala** — lock distribuído não funciona com centenas de serviços

Sem coordenação, se um serviço falhar no meio do fluxo, os dados ficam **inconsistentes** entre os serviços.

---

## Solução

Coordenar uma sequência de **transações locais**. Cada serviço executa sua transação local e, em caso de falha, **compensações** são executadas para desfazer as etapas já concluídas, restaurando a consistência.

**Saga = sequência de transações locais T₁, T₂, T₃ ... Tₙ, onde cada Tᵢ tem uma compensação Cᵢ**

```
Sucesso total: T₁ → T₂ → T₃ → ... → Tₙ ✅

Falha em T₃:   T₁ → T₂ → T₃ ❌ → C₂ → C₁  (compensa na ordem reversa)
```

---

## Duas Abordagens: Orquestração vs. Coreografia

### Comparação

| Aspecto | Orquestração | Coreografia |
|---------|-------------|-------------|
| **Coordenação** | Um **orquestrador central** controla o fluxo | Cada serviço **reage a eventos** e publica o próximo |
| **Acoplamento** | Serviços acoplados ao orquestrador | Serviços desacoplados (event-driven) |
| **Visibilidade** | Fluxo centralizado — fácil de rastrear e depurar | Fluxo distribuído — mais difícil de visualizar |
| **Complexidade** | Orquestrador pode ficar complexo; single point of failure | Sem ponto central, mas lógica espalhada por todos os serviços |
| **Testabilidade** | Mais fácil — testa o orquestrador isoladamente | Mais difícil — precisa testar interações entre serviços |
| **Quando usar** | Fluxos com **mais de 3 steps** ou compensações complexas | Fluxos **simples** com poucos participantes (2–3) |

---

## Orquestração (Saga Orchestrator)

Um componente central (o **orquestrador**) conhece todos os steps, executa-os na ordem correta e, em caso de falha, dispara as compensações.

```
┌───────────────┐       ┌──────────────────┐       ┌────────────────┐
│ Order Service │──1──▶ │ Saga             │──2──▶ │ Payment Service│
│ (inicia saga) │       │ Orchestrator     │       │                │
│               │       │                  │──3──▶ │ Stock Service  │
│               │       │                  │──4──▶ │ Shipping       │
│               │◀──5── │ (completo/falha) │       │ Service        │
└───────────────┘       └──────────────────┘       └────────────────┘
```

### Fluxo de Sucesso

```
Orquestrador:
  Step 1: paymentService.charge(orderId, amount)    → OK ✅
  Step 2: stockService.reserve(orderId, items)      → OK ✅
  Step 3: shippingService.schedule(orderId, address) → OK ✅
  
  Resultado: SAGA COMPLETED ✅
```

### Fluxo de Falha + Compensação

```
Orquestrador:
  Step 1: paymentService.charge(orderId, amount)    → OK ✅
  Step 2: stockService.reserve(orderId, items)      → FALHA ❌
  
  Compensação (ordem reversa):
  Compensate 1: paymentService.refund(orderId)      → OK ✅
  
  Resultado: SAGA COMPENSATED (rollback)
```

### Interface do Step

Cada step da saga implementa duas operações:

```
interface SagaStep:
    execute(context)     // Ação principal (ex: cobrar pagamento)
    compensate(context)  // Desfazer a ação (ex: estornar pagamento)
```

### Estado da Saga

O orquestrador persiste o **estado** da saga para recuperação em caso de crash:

| Estado | Significado |
|--------|------------|
| `STARTED` | Saga iniciada |
| `PAYMENT_COMPLETED` | Pagamento OK, próximo step: estoque |
| `STOCK_RESERVED` | Estoque reservado, próximo step: envio |
| `COMPLETED` | Todos os steps concluídos com sucesso |
| `COMPENSATING` | Algum step falhou, executando compensações |
| `COMPENSATED` | Todas as compensações executadas |
| `FAILED` | Falha em uma compensação (requer intervenção manual) |

---

## Coreografia (Event-Driven Saga)

Não há orquestrador. Cada serviço **escuta eventos** e **publica eventos** para o próximo participante.

```
OrderService ──(OrderCreated)──▶ PaymentService
                                      │
                              ──(PaymentCompleted)──▶ StockService
                                                          │
                                                ──(StockReserved)──▶ ShippingService
                                                                          │
                                                              ──(ShipmentScheduled)──▶ OrderService
                                                                                          │
                                                                                   Marca como COMPLETED

Em caso de falha:
StockService ──(StockFailed)──▶ PaymentService ──(compensa: estorna pagamento)
                               ──(PaymentRefunded)──▶ OrderService ──(marca como FAILED)
```

### Vantagens da Coreografia
- **Zero acoplamento central** — cada serviço é autônomo
- **Escalabilidade** — adicionar participantes é desacoplado
- **Sem single point of failure** — não há orquestrador

### Desvantagens da Coreografia
- **Difícil visualizar** — fluxo espalhado por múltiplos serviços
- **Difícil depurar** — precisa correlacionar eventos entre serviços
- **Ciclos** — risco de loops infinitos se eventos se retroalimentarem
- **Compensação distribuída** — cada serviço precisa saber quando e como compensar

---

## Compensações: Regras

| Regra | Descrição |
|-------|-----------|
| **Ordem reversa** | Compensações executam na ordem reversa dos steps completados |
| **Idempotente** | Compensações devem ser idempotentes — podem ser executadas mais de uma vez |
| **Semântica** | Compensação nem sempre é "desfazer"; pode ser uma ação de negócio (ex: criar nota de crédito em vez de estornar) |
| **Não falha** | Compensações devem ser robustas; se falhar, retente ou escale para intervenção manual |
| **Sem novas operações** | Não inicie novos fluxos dentro de uma compensação |

### Exemplos de Compensações

| Step (ação) | Compensação |
|-------------|-------------|
| Cobrar pagamento | Estornar pagamento |
| Reservar estoque | Liberar reserva de estoque |
| Agendar envio | Cancelar agendamento |
| Criar pedido | Cancelar pedido (status = CANCELLED) |
| Emitir nota fiscal | Emitir nota de cancelamento |
| Reservar assento (voo) | Liberar assento |

---

## Diagrama de Estados da Saga (Orquestração)

```
    ┌──────────┐
    │ STARTED  │
    └────┬─────┘
         │ Step 1: payment
    ┌────▼─────────────┐
    │ PAYMENT_COMPLETED│
    └────┬─────────────┘
         │ Step 2: stock           ┌──────────────┐
    ┌────▼─────────────┐     ❌──▶│ COMPENSATING │
    │ STOCK_RESERVED   │          └──────┬───────┘
    └────┬─────────────┘                 │
         │ Step 3: shipping              ▼
    ┌────▼─────────────┐          ┌──────────────┐
    │ COMPLETED    ✅  │          │ COMPENSATED  │
    └──────────────────┘          └──────────────┘
```

---

## Exemplo Conceitual (Pseudocódigo)

```
class OrderSagaOrchestrator:
    steps = [PaymentStep, StockStep, ShippingStep]
    
    function execute(context):
        completedSteps = []
        
        for step in steps:
            try:
                step.execute(context)
                completedSteps.add(step)
                saveSagaState(context, step, "COMPLETED")
            catch SagaStepException:
                log.error("Saga falhou no step {}. Compensando...", step.name)
                compensate(completedSteps, context)
                saveSagaState(context, step, "COMPENSATED")
                return SagaResult.FAILED
        
        return SagaResult.COMPLETED
    
    function compensate(completedSteps, context):
        // Ordem reversa!
        for step in reverse(completedSteps):
            try:
                step.compensate(context)
            catch:
                log.error("Falha na compensação do step {}!", step.name)
                // Registra para intervenção manual
                alertOps(step, context)
```

---

## Saga + Outbox Pattern

Para garantir que o estado da saga e os eventos sejam consistentes, combine com o **Outbox Pattern**:

```
Saga Step executa:
  BEGIN TRANSACTION
    1. Executa ação de negócio (ex: reservar estoque)
    2. Salva evento na tabela outbox (ex: StockReserved)
    3. Atualiza estado da saga
  COMMIT

Outbox Publisher:
  Lê outbox → Publica evento no message broker
```

Sem Outbox, se o banco commitar mas o evento não for publicado, o próximo step nunca é executado e a saga fica "travada".

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| 2PC disfarçado | Tenta lock distribuído com sagas | Sagas usam consistência eventual, não locks |
| Steps não-idempotentes | Re-execução causa duplicação | Cada step deve ser idempotente |
| Compensação que pode falhar | Saga fica em estado inválido | Compensações devem ser robustas; escale para manual |
| Sem persistência de estado | Crash = saga perdida | Persista o estado de cada step |
| Coreografia com muitos steps | Complexidade incontrolável | Use orquestração para >3 steps |
| Saga sem timeout | Saga "viva" para sempre em estado intermediário | Defina timeout global e compense se ultrapassar |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Outbox Pattern** | Garante consistência entre estado do step e publicação de evento |
| **Idempotência** | Cada step e compensação devem ser idempotentes |
| **DLQ** | Eventos de saga que falham repetidamente vão para DLQ |
| **CQRS** | Saga pode publicar eventos que atualizam modelos de leitura |
| **Event Sourcing** | O estado da saga pode ser reconstruído a partir dos eventos |

---

## Boas Práticas

1. Prefira **orquestração** para fluxos com mais de 3 steps ou compensações complexas.
2. Prefira **coreografia** para fluxos simples com poucos participantes.
3. **Persista o estado** da saga para recuperação após crash.
4. Cada step deve ser **idempotente** (a saga pode ser re-executada).
5. Implemente **timeout** na saga — se não completar em X minutos, compense automaticamente.
6. Combine com **Outbox Pattern** para garantir consistência entre dados e eventos.
7. Compensações devem ser **robustas** — se falharem, tenha mecanismo de escalonamento.
8. **Teste cenários de falha** — simule falha em cada step e valide que a compensação funciona.

---

## Referências

- Chris Richardson — *Microservices Patterns* (Manning) — Capítulo 4: Managing Transactions with Sagas
- Hector Garcia-Molina & Kenneth Salem — *Sagas* (ACM SIGMOD, 1987)
- Microsoft — [Saga distributed transactions pattern](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga)
