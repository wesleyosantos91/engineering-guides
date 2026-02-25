# Observabilidade Avançada — Distributed Tracing Strategies

> **Objetivo deste documento:** Servir como referência completa sobre **estratégias avançadas de distributed tracing**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: context propagation, sampling strategies, trace analysis patterns, async tracing, e AWS X-Ray / OpenTelemetry.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **Context Propagation** | W3C TraceContext em TODA chamada cross-service | Header custom ou propagation missing | Use `traceparent` / `tracestate` (W3C) |
| **Head-based sampling** | Decisão no entry point, propagada | Cada serviço decide independente → trace fragmentado | Decisão no edge, `sampled` flag propagada |
| **Tail-based sampling** | Decisão após trace completo | Samplear 100% → custo explode | Collector gateway com tail-based sampling |
| **Span attributes** | Contexto de negócio nos spans | Spans com apenas `http.method` e `http.url` | Adicione `order.id`, `user.tier`, `payment.method` |
| **Span naming** | `<verb> <noun>` semântico | `span-1`, `my-span`, `/api/v1/users/123` | `GET /api/v1/users/{id}`, `process_order` |
| **Async tracing** | Link entre producer e consumer | Producer trace termina, consumer é novo trace | SpanLinks: consumer → producer span |

---

## Sumário

- [Observabilidade Avançada — Distributed Tracing Strategies](#observabilidade-avançada--distributed-tracing-strategies)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Anatomia de um Trace](#anatomia-de-um-trace)
  - [Context Propagation — O Fundamento](#context-propagation--o-fundamento)
  - [Sampling Strategies — Decisão Crítica](#sampling-strategies--decisão-crítica)
  - [Span Design — Best Practices](#span-design--best-practices)
  - [Tracing Async \& Event-Driven](#tracing-async--event-driven)
  - [Tracing em Arquiteturas Complexas](#tracing-em-arquiteturas-complexas)
  - [AWS X-Ray vs OpenTelemetry](#aws-x-ray-vs-opentelemetry)
  - [Trace Analysis Patterns](#trace-analysis-patterns)
  - [Performance \& Cost Optimization](#performance--cost-optimization)
  - [Anti-Patterns de Tracing](#anti-patterns-de-tracing)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Anatomia de um Trace

### Conceitos fundamentais

```
┌─────────────────────────────────────────────────────────────────┐
│                      TRACE ANATOMY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TRACE: Representação de uma transação end-to-end                │
│  ├── trace_id: identificador único (128-bit, hex)                │
│  ├── Composto por SPANS                                          │
│  └── Forma uma DAG (Directed Acyclic Graph)                      │
│                                                                  │
│  SPAN: Uma unidade de trabalho dentro do trace                   │
│  ├── span_id: identificador único (64-bit, hex)                  │
│  ├── parent_span_id: quem criou este span                        │
│  ├── operation_name: "GET /api/orders"                           │
│  ├── start_time / end_time                                       │
│  ├── status: OK, ERROR, UNSET                                    │
│  ├── kind: CLIENT, SERVER, PRODUCER, CONSUMER, INTERNAL         │
│  ├── attributes: key-value pairs (contexto)                      │
│  ├── events: timestamped logs dentro do span                     │
│  └── links: referências a outros traces/spans                    │
│                                                                  │
│  SPAN KINDS:                                                     │
│  ┌──────────┐  "Eu estou chamando outro serviço"                │
│  │ CLIENT   │  Ex: HTTP client, gRPC client, DB query            │
│  └──────────┘                                                    │
│  ┌──────────┐  "Eu estou recebendo uma chamada"                 │
│  │ SERVER   │  Ex: HTTP server handler, gRPC server              │
│  └──────────┘                                                    │
│  ┌──────────┐  "Eu estou enviando uma mensagem assíncrona"      │
│  │ PRODUCER │  Ex: SQS SendMessage, Kafka Produce                │
│  └──────────┘                                                    │
│  ┌──────────┐  "Eu estou processando uma mensagem assíncrona"   │
│  │ CONSUMER │  Ex: SQS ReceiveMessage, Kafka Consume             │
│  └──────────┘                                                    │
│  ┌──────────┐  "Operação interna, não cross-service"            │
│  │ INTERNAL │  Ex: business logic, computation                   │
│  └──────────┘                                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Visualização — Waterfall View

```
trace_id: a1b2c3d4e5f6

Time ──────────────────────────────────────────────▶

API Gateway    │████████████████████████████████│  250ms  (root)
               │                                │
Auth Svc       │  │██████│                      │   15ms
               │  │      │                      │
Redis          │  │ │██│ │                      │    2ms
               │  │      │                      │
Order Svc      │         │█████████████████████ │  180ms
               │         │                     ││
DynamoDB       │         │ │████│              ││    8ms
               │         │                     ││
Payment Svc    │         │      │█████████████│││  150ms
               │         │      │             │││
Stripe API     │         │      │ │███████████│││  145ms ← SLOW
               │         │                     ││
SQS            │         │                │███│ │    5ms
               │                                │
Notif Svc      │                         │████│ │   20ms
               │                         │    │ │
SES            │                         │ │██│ │   18ms

LEITURA DO WATERFALL:
1. Total: 250ms
2. Caminho crítico: Gateway → Order → Payment → Stripe (145ms)
3. Stripe é o bottleneck (58% do total)
4. Parallelism: Auth e Order executam em sequência
5. SQS + Notification executam após Order (não blocking)
```

---

## Context Propagation — O Fundamento

### W3C Trace Context (Standard)

```
HTTP Headers propagados automaticamente:

traceparent: 00-a1b2c3d4e5f60718a1b2c3d4e5f60718-b7c8d9e0f1a2b3c4-01
             ││                                  │                  ││
             ││                                  │                  │└─ sampled (01=yes, 00=no)
             ││                                  │                  │
             ││                                  └──────────────────└── parent-id (span_id, 16 hex)
             ││
             │└────────────────────────────────────── trace-id (32 hex)
             └─────────────────────────────────────── version (00)

tracestate: vendor1=value1,vendor2=value2
            (vendor-specific data, e.g., aws=Root=1-abc-def)
```

### Propagation em diferentes protocolos

| Protocolo | Header/Campo | Mecanismo |
|-----------|-------------|-----------|
| **HTTP** | `traceparent`, `tracestate` | HTTP headers |
| **gRPC** | `traceparent` | gRPC metadata |
| **SQS** | `AWSTraceHeader` ou Message Attributes | SQS attributes |
| **SNS** | Message Attributes | SNS attributes |
| **Kafka** | Record Headers | Kafka headers |
| **EventBridge** | Detail metadata | Event detail |
| **Step Functions** | Propagação automática com X-Ray | Built-in |
| **Lambda** | `_X_AMZN_TRACE_ID` env var | Environment variable |

### Propagation Patterns

```
PATTERN 1: Síncrono (HTTP/gRPC)
═══════════════════════════════
Service A ──traceparent──▶ Service B ──traceparent──▶ Service C

Span hierarchy: A (parent) → B (child) → C (grandchild)
Context: Automaticamente propagado pelos SDKs

─────────────────────────────────────────────────────────────

PATTERN 2: Assíncrono (Queue/Topic)
════════════════════════════════════
Service A ──▶ SQS ──▶ Service B

Producer (A):  Injeta traceparent nos Message Attributes
Consumer (B):  Extrai traceparent e cria NOVO span com LINK

Span hierarchy:
  Trace 1: A.produce → SQS.send
  Trace 2: B.consume (SpanLink → A.produce)

─────────────────────────────────────────────────────────────

PATTERN 3: Fan-out (1 → N)
═══════════════════════════
Service A ──▶ SNS ──┬──▶ Service B
                    ├──▶ Service C
                    └──▶ Service D

Cada consumer cria Trace próprio com SpanLink para A
Visualização: "Star topology" — A no centro, B/C/D como rays

─────────────────────────────────────────────────────────────

PATTERN 4: Batch processing
═══════════════════════════
Service A: processa 1000 mensagens do SQS

Opção 1: Um span para o batch inteiro (recomendado para alto volume)
         span: "process_batch" attributes: {batch_size: 1000}
         Links: para os 1000 producer spans (limitado, use amostragem)

Opção 2: Um span por mensagem (para baixo volume ou itens críticos)
         1000 consumer spans, cada um linkado ao producer
```

### Pseudocode — Propagation manual

```python
# PRODUCER — Injetar contexto no SQS
from opentelemetry import trace, context
from opentelemetry.propagators import inject

tracer = trace.get_tracer("order-service")

with tracer.start_as_current_span("send_order_event", kind=SpanKind.PRODUCER) as span:
    span.set_attribute("messaging.system", "aws_sqs")
    span.set_attribute("messaging.destination.name", "order-events")
    span.set_attribute("order.id", order_id)
    
    # Injetar context nos message attributes
    carrier = {}
    inject(carrier)  # carrier agora tem {"traceparent": "00-..."}
    
    sqs.send_message(
        QueueUrl=queue_url,
        MessageBody=json.dumps(order_event),
        MessageAttributes={
            "traceparent": {
                "DataType": "String",
                "StringValue": carrier["traceparent"]
            }
        }
    )

# ────────────────────────────────────────────

# CONSUMER — Extrair contexto do SQS
from opentelemetry.propagators import extract

def process_message(message):
    # Extrair context dos message attributes
    carrier = {
        "traceparent": message.attributes.get("traceparent")
    }
    parent_context = extract(carrier)
    
    # Criar span como LINK (não child, pois é async)
    links = [trace.Link(trace.get_current_span(parent_context).get_span_context())]
    
    with tracer.start_as_current_span(
        "process_order_event",
        kind=SpanKind.CONSUMER,
        links=links
    ) as span:
        span.set_attribute("messaging.system", "aws_sqs")
        span.set_attribute("messaging.operation", "process")
        # ... process order
```

---

## Sampling Strategies — Decisão Crítica

### Por que sampling?

```
SEM SAMPLING:
  10,000 req/s × 86,400s/day × 10 spans/trace × 1KB/span
  = 8.64 TB/dia de trace data
  = ~$500/dia em storage + ingest
  = ~$15,000/mês

COM SAMPLING (10%):
  = 864 GB/dia
  = ~$1,500/mês

COM SMART SAMPLING (1% normal + 100% errors):
  = ~100 GB/dia normal + all errors
  = ~$300/mês + visibilidade completa de erros
```

### Head-Based Sampling

```
┌─────────────────────────────────────────────────────────────┐
│                 HEAD-BASED SAMPLING                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  DECISÃO: No início do trace (entry point)                   │
│  PROPAGAÇÃO: sampled flag viaja com o traceparent            │
│                                                              │
│  API Gateway                                                 │
│     │                                                        │
│     │ Random: 10% → sampled=01 (yes)                        │
│     │         90% → sampled=00 (no)                         │
│     │                                                        │
│     │ traceparent: 00-traceId-spanId-01  ← sampled           │
│     │                                                        │
│     ├──▶ Service A (vê sampled=01, gera spans)              │
│     │    ├──▶ Service B (mesmo, sampled=01)                 │
│     │    └──▶ Service C (mesmo, sampled=01)                 │
│     │                                                        │
│  PRÓS:                                                       │
│  ✅ Simples de implementar                                   │
│  ✅ Baixo overhead (decisão no entry point)                  │
│  ✅ Trace completo ou nada (sem fragmentação)                │
│  ✅ Previsível: 10% = 10% do tráfego                        │
│                                                              │
│  CONTRAS:                                                    │
│  ❌ Não sabe se o trace vai ser interessante (erro/lento)    │
│  ❌ Pode perder 90% dos errors se error rate < 10%           │
│  ❌ Não adapta a picos de tráfego                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Tail-Based Sampling

```
┌─────────────────────────────────────────────────────────────┐
│                 TAIL-BASED SAMPLING                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  DECISÃO: Após trace completo (todos spans coletados)        │
│  ONDE: No OTel Collector Gateway (centralizado)              │
│                                                              │
│  Services → Collector Agents → Collector Gateway             │
│                                      │                       │
│                                Buffer + Wait                 │
│                                      │                       │
│                                DECISÃO baseada em:           │
│                                • Status = ERROR? → KEEP      │
│                                • Latency > P95?  → KEEP      │
│                                • Has exception?  → KEEP      │
│                                • Random 5%       → KEEP      │
│                                • Else            → DROP      │
│                                      │                       │
│                                 ┌────┼────┐                  │
│                                 │         │                  │
│                               KEEP      DROP                 │
│                               → Backend  → /dev/null         │
│                                                              │
│  PRÓS:                                                       │
│  ✅ Captura 100% dos traces com erro                         │
│  ✅ Captura traces lentos (outliers)                         │
│  ✅ Decisão inteligente baseada em dados completos            │
│  ✅ Melhor custo-benefício                                    │
│                                                              │
│  CONTRAS:                                                    │
│  ❌ Collector Gateway precisa de memória para buffer          │
│  ❌ Latência adicional (esperar trace completar)              │
│  ❌ Collector Gateway = SPOF (precisa HA)                    │
│  ❌ Mais complexo de operar                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Estratégia de sampling recomendada

```
┌─────────────────────────────────────────────────────────────┐
│            SAMPLING STRATEGY — RECOMENDAÇÃO                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  TRÁFEGO BAIXO (< 100 req/s):                               │
│  → 100% traces (sem sampling)                                │
│  → Custo mensal: < $100                                      │
│                                                              │
│  TRÁFEGO MÉDIO (100-1000 req/s):                             │
│  → Head-based: 10% random                                    │
│  → Always sample: errors, slow requests (> P95)              │
│  → Custo mensal: < $500                                      │
│                                                              │
│  TRÁFEGO ALTO (> 1000 req/s):                                │
│  → Tail-based sampling no Collector Gateway                  │
│  → Policies:                                                 │
│    1. 100% errors (status = ERROR)                           │
│    2. 100% slow (latency > P95 threshold)                    │
│    3. 100% specific operations (payment, auth)               │
│    4. 1-5% probabilistic (baseline coverage)                 │
│  → Custo mensal: < $1,500                                    │
│                                                              │
│  TRÁFEGO EXTREMO (> 10K req/s):                              │
│  → Head-based 1% + Tail-based com policies agressivas       │
│  → Rate limiting no collector (max spans/s)                  │
│  → Sampling adaptativo baseado em load                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Pseudocode — Tail-based sampling config (OTel Collector)

```yaml
# OTel Collector Gateway — tail_sampling processor
processors:
  tail_sampling:
    decision_wait: 30s        # Esperar 30s pelo trace completo
    num_traces: 100000        # Buffer para 100K traces simultâneos
    policies:
      # Policy 1: SEMPRE manter traces com erro
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]

      # Policy 2: SEMPRE manter traces lentos
      - name: latency-policy
        type: latency
        latency:
          threshold_ms: 500   # > 500ms = manter

      # Policy 3: SEMPRE manter operações críticas
      - name: critical-ops
        type: string_attribute
        string_attribute:
          key: operation.critical
          values: [payment, auth, checkout]

      # Policy 4: Manter 5% aleatório (baseline)
      - name: probabilistic-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 5

      # Policy 5: Rate limiting global (segurança)
      - name: rate-limiting
        type: rate_limiting
        rate_limiting:
          spans_per_second: 1000
```

---

## Span Design — Best Practices

### Naming conventions

| ❌ Ruim | ✅ Bom | Por quê |
|---------|--------|---------|
| `span-1` | `GET /api/orders/{id}` | Semântico, agrupável |
| `/api/v1/users/12345` | `GET /api/v1/users/{id}` | Não inclua IDs no nome (high cardinality) |
| `doStuff` | `validate_payment` | Descreve a operação |
| `handler` | `POST /api/checkout` | Identifica verb + resource |
| `query` | `SELECT orders` | Identifica a operação de DB |
| `my-span` | `process_order_batch` | Ação clara |

### Span attributes — O que incluir

```
OBRIGATÓRIO (Semantic Conventions OpenTelemetry):
├── service.name          = "order-service"
├── service.version       = "2.3.1"
├── deployment.environment = "production"
│
├── HTTP spans:
│   ├── http.request.method  = "POST"
│   ├── http.route           = "/api/v1/orders"  (template, não URL)
│   ├── http.response.status_code = 201
│   ├── url.scheme           = "https"
│   └── server.address       = "api.example.com"
│
├── Database spans:
│   ├── db.system            = "dynamodb"
│   ├── db.operation.name    = "GetItem"
│   ├── db.collection.name   = "orders"
│   └── aws.dynamodb.table_names = ["orders"]
│
├── Messaging spans:
│   ├── messaging.system     = "aws_sqs"
│   ├── messaging.destination.name = "order-events"
│   ├── messaging.operation  = "publish"
│   └── messaging.message.id = "msg-123"
│
└── gRPC spans:
    ├── rpc.system           = "grpc"
    ├── rpc.service          = "PaymentService"
    ├── rpc.method           = "ProcessPayment"
    └── rpc.grpc.status_code = 0

CUSTOM (negócio):
├── order.id               = "ord-999"
├── order.total_cents       = 15000
├── order.items_count       = 3
├── user.tier              = "premium"
├── payment.method         = "credit_card"
├── payment.provider       = "stripe"
└── feature_flag.new_checkout = true
```

### Span events — Logs dentro do trace

```python
# Pseudocode — Span events
with tracer.start_as_current_span("process_order") as span:
    # Evento: validação
    span.add_event("order.validated", {
        "items_count": len(items),
        "total": total
    })
    
    # Evento: retry
    span.add_event("payment.retry", {
        "attempt": 2,
        "reason": "timeout",
        "provider": "stripe"
    })
    
    # Evento: exceção (captura automática com record_exception)
    try:
        result = payment_client.charge(order)
    except TimeoutError as e:
        span.record_exception(e)
        span.set_status(StatusCode.ERROR, "Payment timeout")
        raise
```

### Granularidade de spans — Quando criar

| Cenário | Criar span? | Justificativa |
|---------|------------|---------------|
| Chamada HTTP/gRPC outbound | ✅ Sempre | Cross-service boundary |
| Query de banco de dados | ✅ Sempre | I/O externo, potencial bottleneck |
| Cache lookup (Redis, ElastiCache) | ✅ Sempre | I/O externo |
| Chamada a API externa (Stripe, etc.) | ✅ Sempre | Dependência externa |
| Business logic complexa (> 10ms) | ✅ Sim | Identificar bottleneck no código |
| Loop iterando sobre items | ⚠️ Um span para o loop, não por item | Evitar span explosion |
| Validação simples (< 1ms) | ❌ Não | Overhead > benefício |
| Getter/setter trivial | ❌ Nunca | Noise desnecessário |

---

## Tracing Async & Event-Driven

### O desafio

```
SINCRONO: A → B → C  (um trace, parent-child simples)

ASSÍNCRONO: A → SQS → B (horas depois)
            │              │
            Trace 1        Trace 2
            (request)      (processing)
            
PROBLEMA: Como conectar Trace 1 e Trace 2?
SOLUÇÃO: SpanLinks
```

### SpanLinks — Conectando traces assíncronos

```
┌─────────────────────────────────────────────────────────────┐
│                     SPAN LINKS                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Parent-Child (síncrono):                                    │
│  Trace: abc123                                               │
│  A ──parent──▶ B ──parent──▶ C                              │
│  "B é CAUSADO por A, dentro do mesmo request"                │
│                                                              │
│  SpanLink (assíncrono):                                      │
│  Trace 1: abc123        Trace 2: def456                      │
│  A ──▶ SQS               B (consumer)                       │
│                           │                                  │
│                           └──link──▶ A (span de Trace 1)    │
│                                                              │
│  "B é RELACIONADO a A, mas em contexto diferente"            │
│                                                              │
│  VISUALIZAÇÃO NO BACKEND:                                    │
│  Trace 2 (def456):                                           │
│  └── B: process_order (consumer)                             │
│      ├── attributes: {order.id: "ord-999"}                   │
│      └── links: [{trace_id: abc123, span_id: spanA}]        │
│          → Click: navega para trace abc123                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Patterns por tipo de async

| Pattern | Mecanismo | Como tracear |
|---------|-----------|-------------|
| **SQS** | Message Attributes | Inject `traceparent` no SendMessage, extract no ReceiveMessage, SpanLink |
| **SNS** | Message Attributes | Inject no Publish, extract em cada subscriber |
| **SNS → SQS** | Propagação automática | SNS preserva attributes para SQS |
| **Kafka/MSK** | Record Headers | Inject no ProducerRecord, extract no ConsumerRecord |
| **EventBridge** | Detail metadata | Inject no PutEvents, extract na Rule target |
| **Step Functions** | X-Ray native | Propagação automática (se X-Ray habilitado) |
| **Lambda (async invoke)** | X-Ray active tracing | SpanLink entre invocações |
| **S3 Event → Lambda** | X-Ray subsegment | Novo trace com link para S3 PutObject |
| **DynamoDB Streams → Lambda** | Stream record metadata | SpanLink para write original |

### Batch consumer pattern

```python
# Pseudocode — Batch consumer com SpanLinks
def process_sqs_batch(messages):
    # Coletar links de todos os producers
    links = []
    for msg in messages:
        carrier = extract_trace_context(msg.attributes)
        if carrier:
            links.append(trace.Link(
                carrier.span_context,
                attributes={"messaging.message.id": msg.id}
            ))
    
    # Criar UM span para o batch com todos os links
    with tracer.start_as_current_span(
        "process_order_batch",
        kind=SpanKind.CONSUMER,
        links=links[:128]  # OTel limit: 128 links por span
    ) as span:
        span.set_attribute("messaging.batch.message_count", len(messages))
        span.set_attribute("messaging.system", "aws_sqs")
        
        for msg in messages:
            # Processar cada mensagem (sem span individual em batch grande)
            process_single_order(msg)
        
        span.set_attribute("messaging.batch.success_count", success)
        span.set_attribute("messaging.batch.failure_count", failures)
```

---

## Tracing em Arquiteturas Complexas

### Service Mesh (Envoy / App Mesh)

```
┌─────────────────────────────────────────────────────────────┐
│              SERVICE MESH TRACING                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────┐     ┌─────────────────────┐        │
│  │   Pod A              │     │   Pod B              │        │
│  │  ┌───────┐ ┌──────┐ │     │  ┌──────┐ ┌───────┐ │        │
│  │  │ App A │→│Envoy │─┼────▶┼─▶│Envoy │→│ App B │ │        │
│  │  └───────┘ └──────┘ │     │  └──────┘ └───────┘ │        │
│  └─────────────────────┘     └─────────────────────┘        │
│                                                              │
│  Envoy automaticamente:                                      │
│  1. Gera spans para cada request in/out                      │
│  2. Propaga traceparent headers                              │
│  3. Mede latência (proxy overhead vs app processing)         │
│                                                              │
│  MAS: App DEVE propagar headers recebidos!                   │
│       Se App A recebe traceparent e não repassa para         │
│       chamadas outbound → trace quebrado após App A          │
│                                                              │
│  REGRA: Mesmo com service mesh, configure propagation no SDK │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Lambda tracing (AWS)

```
┌─────────────────────────────────────────────────────────────┐
│                    LAMBDA TRACING                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  OPÇÃO 1: X-Ray (nativo)                                     │
│  • Habilitar Active Tracing na Lambda                        │
│  • Automático: Cold start, invocation, downstream calls      │
│  • aws-xray-sdk para subsegments customizados               │
│                                                              │
│  OPÇÃO 2: OTel Lambda Layer (recomendado para portabilidade) │
│  • AWS Distro for OpenTelemetry (ADOT) Lambda Layer          │
│  • Auto-instrumentação de HTTP, SDK calls, DB               │
│  • Exportar para X-Ray ou qualquer OTLP backend             │
│                                                              │
│  COLD START IMPACT:                                          │
│  ADOT Layer adiciona ~300-500ms no cold start (Java)         │
│  ADOT Layer adiciona ~100-200ms no cold start (Python/Node)  │
│                                                              │
│  MITIGAÇÃO:                                                  │
│  • Provisioned Concurrency para Lambda crítica               │
│  • Use X-Ray nativo se cold start é crítico                  │
│  • Para Java: use OTel agent em vez do Layer                 │
│                                                              │
│  TRACE PROPAGATION EM LAMBDA:                                │
│  API GW → Lambda: automático (X-Ray header)                  │
│  SQS → Lambda: Message Attributes → extract manually         │
│  SNS → Lambda: preserve headers                              │
│  S3 Event → Lambda: novo trace (link para upload trace)      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## AWS X-Ray vs OpenTelemetry

### Comparação detalhada

| Critério | AWS X-Ray | OpenTelemetry |
|----------|----------|---------------|
| **Lock-in** | AWS only | Vendor-neutral |
| **Integração AWS** | Nativa (Lambda, API GW, ECS...) | Via ADOT Layer |
| **Cold start overhead** | Baixo (~50ms) | Médio (~200-500ms) |
| **Sampling** | Reservoir + fixed rate | Head + tail-based |
| **Trace analytics** | X-Ray Analytics (basic) | Backend choice (Tempo, Jaeger, Datadog) |
| **Service map** | ✅ Automático | Depende do backend |
| **Custom instrumentation** | aws-xray-sdk | OTel SDK (mais rico) |
| **Métricas** | ❌ (CloudWatch separado) | ✅ Unified (traces + metrics + logs) |
| **Community** | AWS | CNCF (enorme, multi-vendor) |
| **Multi-cloud** | ❌ | ✅ |
| **Custo** | Pay per traces sampled | Backend dependent |

### Decisão

```
ESCOLHA AWS X-RAY quando:
• 100% AWS, sem planos de multi-cloud
• Precisa de service map com zero config
• Lambda-heavy com cold start critical
• Time pequeno, quer simplicidade

ESCOLHA OPENTELEMETRY quando:
• Multi-cloud ou hybrid
• Quer escolher backend (Grafana Tempo, Jaeger, Datadog)
• Precisa de métricas + traces + logs unificados
• Time maduro em observabilidade
• Quer portabilidade e vendor-neutrality

MELHOR DOS DOIS MUNDOS:
• Use OTel SDK para instrumentação
• Use ADOT Collector para export
• Exporte para X-Ray (service map) + Tempo/Jaeger (trace analysis)
```

---

## Trace Analysis Patterns

### Pattern 1: Critical Path Analysis

```
Identificar o caminho mais lento no trace:

Trace total: 500ms

Gateway (500ms)
├── Auth (50ms)
├── Order (400ms)           ← no critical path
│   ├── DB Read (20ms)
│   ├── Payment (350ms)     ← no critical path
│   │   └── Stripe (340ms)  ← ROOT CAUSE (68% do total)
│   └── SQS (10ms)
└── Response (50ms)

Critical Path: Gateway → Order → Payment → Stripe
Ação: Investigar latência do Stripe (timeout? região? connection pool?)
```

### Pattern 2: Fan-out Analysis

```
Identificar N+1 queries e fan-outs excessivos:

Order Service (1200ms)
├── GET order (5ms)
├── GET item 1 (8ms)     ← N+1 PROBLEM!
├── GET item 2 (7ms)
├── GET item 3 (9ms)
├── GET item 4 (8ms)
├── ... (mais 96 items)
└── Total: 100 queries × ~8ms = 800ms

FIX: Batch query → GET items WHERE order_id = X
     New total: 1 query × 15ms = 15ms (53x faster)
```

### Pattern 3: Dependency Failure Analysis

```
Quando um serviço downstream falha:

Order Service (timeout 30s → ERROR)
├── Auth (OK, 15ms)
├── Inventory (OK, 20ms)
├── Payment (ERROR, timeout 30s)     ← DEPENDENCY FAILURE
│   └── Stripe API (no response)     ← EXTERNAL DEPENDENCY DOWN
└── Notification (never called)      ← CASCADING IMPACT

Análises:
1. Circuit breaker configurado? (deveria abrir após N falhas)
2. Timeout adequado? (30s é muito para payment, use 5s)
3. Fallback existe? (queue for retry later?)
4. Retry storm? (10 instances × 3 retries = 30 calls to failing service)
```

---

## Performance & Cost Optimization

### Overhead do tracing

| Componente | Overhead típico | Como mitigar |
|-----------|----------------|-------------|
| **Auto-instrumentation** | 1-3% CPU, 50-200ms cold start | Aceitável para maioria |
| **Custom spans** | < 0.1% per span creation | Evite spans em loops tight |
| **Context propagation** | < 0.01% (header injection) | Desprezível |
| **OTel Collector (agent)** | 50-200MB RAM per node | Size adequadamente |
| **OTel Collector (gateway)** | 1-4GB RAM (tail sampling) | Scale horizontalmente |
| **Batch exporter** | Flush a cada 5s (default) | Tune batch size/interval |

### Cost optimization strategies

| Estratégia | Economia | Trade-off |
|-----------|---------|-----------|
| **Tail-based sampling** | 80-95% | Mais infra (collector gateway) |
| **Drop debug spans** | 30-50% | Menos detalhe em debug |
| **Shorter retention** | 40-60% | Menos histórico |
| **Attribute filtering** | 10-20% | Menos contexto por span |
| **Compression (gzip)** | 50-70% network | CPU overhead (mínimo) |
| **Local pre-aggregation** | 60-80% para metrics | Latência para ver raw data |

---

## Anti-Patterns de Tracing

| Anti-pattern | Problema | Solução |
|-------------|---------|---------|
| **Broken propagation** | Trace termina no Service A, Service B sem contexto | Verificar propagation em TODOS os protocols |
| **No sampling** | 100% dos traces em 10K req/s → $$$$ | Tail-based sampling com policies |
| **Span explosion** | 10K spans por trace (loop interno) | Um span para o batch, não por item |
| **ID no span name** | `GET /users/12345` → millions de operation names | Template: `GET /users/{id}` |
| **No business context** | Spans com apenas HTTP info | Adicionar domain attributes |
| **Always new trace** | Cada serviço cria trace novo | Propagar, não criar novo |
| **Trace ≠ Log** | Span event para cada log line | Logs no logger, events para trace-relevant |
| **Sync-only tracing** | Async flows sem SpanLinks | Implementar SpanLinks para queues/topics |
| **No error recording** | Span com status OK mas internal error | `record_exception()` + `set_status(ERROR)` |
| **Overhead ignorado** | Instrumentação sem medir impacto | Benchmark antes e depois |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código relacionado a distributed tracing, verifique:

1. **Context propagation missing** — Toda chamada outbound (HTTP, gRPC, SQS, Kafka) deve propagar `traceparent`
2. **Span name com high cardinality** — Nomes como `/users/12345` → use template `/users/{id}`
3. **Sem SpanLinks em async** — Consumer de SQS/Kafka deve ter SpanLink para o producer span
4. **Span sem attributes de negócio** — `order.id`, `user.tier`, `payment.method` devem estar nos spans
5. **Sem error recording** — Exceções devem usar `record_exception()` + `set_status(ERROR)`
6. **Sem sampling strategy** — Em produção com > 100 req/s exija configuração de sampling
7. **Span em loop** — Criar 1000 spans em loop → use 1 span para o batch
8. **PII em span attributes** — CPF, email, nome nunca em plaintext nos attributes
9. **Sem span.kind** — Spans devem ter kind correto: CLIENT, SERVER, PRODUCER, CONSUMER
10. **Timeout no exporter** — OTel exporter deve ter timeout configurado (default pode ser alto)
11. **Trace-log desconectado** — Todo log entry deve incluir `trace_id` e `span_id`
12. **Sem semantic conventions** — Use OTel Semantic Conventions para attribute names (não invente)

---

## Referências

- **OpenTelemetry Specification — Tracing** — https://opentelemetry.io/docs/specs/otel/trace/
- **W3C Trace Context** — https://www.w3.org/TR/trace-context/
- **OpenTelemetry Semantic Conventions** — https://opentelemetry.io/docs/specs/semconv/
- **AWS X-Ray Developer Guide** — https://docs.aws.amazon.com/xray/
- **ADOT (AWS Distro for OpenTelemetry)** — https://aws-otel.github.io/
- **Distributed Systems Observability** — Cindy Sridharan (O'Reilly)
- **Mastering Distributed Tracing** — Yuri Shkuro (criador do Jaeger)
