# AWS Serverless — Padrões Arquiteturais

Catálogo de padrões arquiteturais para aplicações serverless na AWS.

---

## Padrões por Categoria

### 1. API Patterns

#### REST API com Lambda (Single-Purpose Functions)

```
                    ┌─ GET /orders     → Lambda: ListOrders
Client → API GW ───┼─ POST /orders    → Lambda: CreateOrder
                    ├─ GET /orders/{id}→ Lambda: GetOrder
                    └─ PUT /orders/{id}→ Lambda: UpdateOrder
```

**Quando usar**: APIs com operações bem definidas, cada função com responsabilidade única.

#### GraphQL com AppSync

```
Client → AppSync ─→ Lambda Resolvers ─→ DynamoDB
                 ├→ Direct DynamoDB Resolvers (sem Lambda)
                 └→ HTTP Data Sources
```

**Quando usar**: Clientes com necessidades de dados variáveis (mobile, SPAs), relações complexas entre entidades.

---

### 2. Messaging Patterns

#### Queue-Based Load Leveling

```
API GW → Lambda Producer → SQS Queue → Lambda Consumer → DynamoDB
                                    │
                          (absorve spikes de tráfego)
```

**Quando usar**: Proteger backend de spikes, desacoplar produtor/consumidor.

#### Priority Queue

```
                    ┌─ SQS High Priority (concorrência 10) → Lambda
Producer ───────────┤
                    └─ SQS Low Priority  (concorrência 2)  → Lambda
```

#### Topic-Queue-Chaining (Fan-Out + Processing)

```
Producer → SNS Topic ──┬→ SQS Queue A → Lambda A (email)
                       ├→ SQS Queue B → Lambda B (analytics)
                       └→ SQS Queue C → Lambda C (inventory)
```

**Quando usar**: Um evento precisa ser processado por múltiplos consumers de forma independente e resiliente.

---

### 3. Event-Driven Patterns

#### Event Router (EventBridge)

```
Service A ──┐
Service B ──┤→ EventBridge ──┬→ Rule: OrderCreated    → Lambda
Service C ──┘                ├→ Rule: PaymentProcessed → Step Functions
                             ├→ Rule: *.Failed         → SNS (alertas)
                             └→ Archive (replay)
```

#### Change Data Capture (DynamoDB Streams)

```
DynamoDB → DynamoDB Streams → Lambda → EventBridge → Consumers
                                    → OpenSearch (search index)
                                    → S3 (audit log)
```

**Quando usar**: Reagir a mudanças de dados, manter sistemas sincronizados, audit trail.

#### Pipes Pattern (EventBridge Pipes)

```
SQS Queue → EventBridge Pipe ─→ Filter ─→ Enrich (Lambda) ─→ Target (Step Functions)
DynamoDB Streams → Pipe ─→ Filter ─→ Transform ─→ EventBridge Bus
```

**Quando usar**: Conectar sources a targets com filtering, enrichment e transformation sem Lambda intermediária.

---

### 4. Orchestration Patterns

#### Saga — Orchestration (Step Functions)

```
Step Functions:
  ┌─ CreateOrder ─→ ReserveInventory ─→ ProcessPayment ─→ ConfirmOrder
  │                                                         │
  └─ (Catch: any failure) ─→ CompensatePayment ─→ ReleaseInventory ─→ CancelOrder
```

**Quando usar**: Transações distribuídas com necessidade de compensação, fluxos complexos com muitas etapas.

#### Saga — Choreography (EventBridge)

```
OrderService ─→ EventBridge: OrderCreated
           InventoryService ←─ (reage)
           InventoryService ─→ EventBridge: InventoryReserved
                          PaymentService ←─ (reage)
                          PaymentService ─→ EventBridge: PaymentProcessed
                                       ShippingService ←─ (reage)

Compensação (reversa):
PaymentService ─→ EventBridge: PaymentFailed
           InventoryService ←─ (compensa: ReleaseInventory)
           OrderService ←─ (compensa: CancelOrder)
```

**Quando usar**: Serviços altamente desacoplados, equipes independentes, menor complexidade central.

---

### 5. Data Processing Patterns

#### Batch Processing (S3 + Lambda)

```
S3 Upload → S3 Event → Lambda → Processo → DynamoDB / S3 Output
                              ↓ (arquivo grande)
                     Step Functions → Map State → Lambda (paralelo)
```

#### Real-Time Streaming

```
Kinesis Data Stream → Lambda (batch processing) → DynamoDB
                   → Kinesis Data Firehose → S3 (data lake)
                   → Kinesis Data Analytics → Real-time dashboards
```

#### ETL Serverless

```
S3 Raw Data → EventBridge → Step Functions:
  ┌─ Lambda: Validate Schema
  ├─ Lambda: Transform Data
  ├─ Lambda: Enrich Data  
  └─ Lambda: Load to DynamoDB/Redshift
```

---

### 6. Security Patterns

#### API Key + Usage Plan

```
Client → API Key → API GW (Usage Plan: rate limit) → Lambda
```

#### JWT Authorization

```
Client → Cognito (authenticate) → JWT Token
Client → API GW (JWT Authorizer validates token) → Lambda
```

#### Resource-Based Authorization

```
Client → API GW → Lambda Authorizer ─→ (generate IAM policy)
                                      → API GW (enforce policy) → Lambda Backend
```

---

### 7. Resilience Patterns

#### Retry with Exponential Backoff

```json
// Step Functions Retry
{
  "Retry": [{
    "ErrorEquals": ["States.TaskFailed"],
    "IntervalSeconds": 2,
    "MaxAttempts": 3,
    "BackoffRate": 2.0
  }]
}
// Waits: 2s, 4s, 8s
```

#### Circuit Breaker (com DynamoDB)

```
Lambda → Check Circuit State (DynamoDB) → CLOSED → Call External Service
                                        → OPEN → Return Cached/Fallback
                                        → HALF-OPEN → Try Once → Update State
```

#### Bulkhead (Concurrency Isolation)

```hcl
# Isolar funções críticas com reserved concurrency
resource "aws_lambda_function" "critical" {
  reserved_concurrent_executions = 100  # Reservado para esta função
}

resource "aws_lambda_function" "non_critical" {
  reserved_concurrent_executions = 20   # Não compete com critical
}
```

---

## Decisão de Padrão

| Cenário | Padrão Recomendado |
|---------|-------------------|
| API CRUD simples | REST API + Lambda + DynamoDB |
| API com relações complexas | AppSync (GraphQL) + DynamoDB |
| Processamento assíncrono | SQS + Lambda |
| Um evento, múltiplos consumers | SNS → SQS (Fan-Out) |
| Roteamento de eventos complexo | EventBridge rules |
| Transação distribuída (controle central) | Step Functions Saga |
| Transação distribuída (equipes independentes) | EventBridge Choreography |
| Processamento de arquivo | S3 Event → Lambda / Step Functions |
| Streaming real-time | Kinesis → Lambda |
| Workflow com aprovação humana | Step Functions + Callback |
| Processamento batch grande | Step Functions Map State |

---

## Referências

- [Serverless Land — Patterns Collection](https://serverlessland.com/patterns)
- [AWS Step Functions Workflow Studio](https://docs.aws.amazon.com/step-functions/latest/dg/workflow-studio.html)
- [EventBridge Pipes](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-pipes.html)
