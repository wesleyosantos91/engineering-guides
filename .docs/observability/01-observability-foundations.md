# Observabilidade Avançada — Fundamentos

> **Objetivo deste documento:** Servir como referência completa sobre **fundamentos de observabilidade avançada**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: conceitos, os Três Pilares expandidos, sinais modernos, telemetry pipelines, cultura e anti-patterns.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **Logs** | Structured JSON, correlation ID em todo log | Log texto livre sem contexto | Structured logging com trace_id, span_id |
| **Metrics** | USE/RED para infra/serviço, cardinalidade controlada | Métrica com label de user_id (high cardinality) | Labels bounded: status, method, endpoint |
| **Traces** | Propagação de contexto end-to-end obrigatória | Trace quebrado entre serviço A e B | W3C TraceContext header propagation |
| **Profiling** | Continuous profiling em produção (low overhead) | Profiling só em dev/staging | Always-on profiler (< 2% CPU overhead) |
| **Events** | Eventos de negócio correlacionados com telemetria | Deploy sem marker no dashboard | Deployment events como annotations |
| **Correlation** | Um trace_id conecta logs + metrics + traces | Dados isolados em silos | Exemplars em métricas, trace_id em logs |
| **Alerting** | Alerte sobre sintomas, não causas | Alerta em CPU > 80% sem impacto em SLI | Alerte quando error_rate > SLO threshold |
| **Dashboards** | USE/RED + Golden Signals no topo | 47 dashboards que ninguém olha | Hierarquia: Exec → Service → Debug |

---

## Sumário

- [Observabilidade Avançada — Fundamentos](#observabilidade-avançada--fundamentos)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Observabilidade vs Monitoramento](#observabilidade-vs-monitoramento)
  - [Os Três Pilares — Expandidos](#os-três-pilares--expandidos)
  - [Sinais Modernos — Além dos Três Pilares](#sinais-modernos--além-dos-três-pilares)
  - [Modelos Mentais para Métricas](#modelos-mentais-para-métricas)
  - [Telemetry Pipeline Architecture](#telemetry-pipeline-architecture)
  - [Instrumentação — Estratégias](#instrumentação--estratégias)
  - [High Cardinality — O Problema Central](#high-cardinality--o-problema-central)
  - [Correlation — A Cola da Observabilidade](#correlation--a-cola-da-observabilidade)
  - [Alerting Strategy](#alerting-strategy)
  - [Dashboard Design](#dashboard-design)
  - [Anti-Patterns de Observabilidade](#anti-patterns-de-observabilidade)
  - [Observability Maturity Model](#observability-maturity-model)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Observabilidade vs Monitoramento

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  MONITORAMENTO                    OBSERVABILIDADE                │
│  ════════════                     ═══════════════                │
│                                                                  │
│  "O sistema está funcionando?"    "POR QUE o sistema está       │
│                                    lento para o user X?"         │
│                                                                  │
│  Dashboards pré-definidos         Exploração ad-hoc              │
│  Perguntas conhecidas             Perguntas desconhecidas        │
│  Thresholds estáticos             Análise de correlação          │
│  Alertas binários (up/down)       Contexto rico (who/what/why)   │
│  Reativo                          Proativo + Reativo             │
│                                                                  │
│  ┌─────────────┐                 ┌─────────────────────────┐    │
│  │ Is it up?   │                 │ Why is P99 latency 3x   │    │
│  │ CPU > 80%?  │                 │ higher for users in     │    │
│  │ Disk full?  │                 │ region=eu-west-1 since  │    │
│  └─────────────┘                 │ deploy v2.3.1?          │    │
│                                  └─────────────────────────┘    │
│                                                                  │
│  Monitoramento ⊂ Observabilidade                                │
│  (Monitoring is a subset of Observability)                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Definição formal

> **Observabilidade** é a capacidade de entender o estado interno de um sistema a partir de seus outputs externos (telemetria), sem precisar modificar o sistema para investigar um problema novo.

Um sistema é **observável** quando:
1. Você pode fazer **perguntas arbitrárias** sobre o comportamento
2. Sem **deploy de novo código** para responder
3. Com **contexto suficiente** para entender o "porquê"

---

## Os Três Pilares — Expandidos

### Pilar 1: Logs

```
┌─────────────────────────────────────────────────────────┐
│                        LOGS                              │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  O QUE: Registros imutáveis de eventos discretos         │
│                                                          │
│  TIPOS:                                                  │
│  • Application logs  (business events)                   │
│  • Access logs       (HTTP requests)                     │
│  • Audit logs        (who did what, compliance)          │
│  • Debug logs        (troubleshooting, ephemeral)        │
│  • Transaction logs  (DB operations)                     │
│                                                          │
│  FORMATO OBRIGATÓRIO EM PRODUÇÃO: Structured JSON        │
│                                                          │
│  ✅ BOM:                                                 │
│  {                                                       │
│    "timestamp": "2026-02-24T10:15:30.123Z",             │
│    "level": "ERROR",                                     │
│    "service": "order-service",                           │
│    "trace_id": "abc123def456",                           │
│    "span_id": "span789",                                 │
│    "user_id": "u-42",                                    │
│    "message": "Payment failed",                          │
│    "error_code": "PAYMENT_TIMEOUT",                      │
│    "duration_ms": 5023,                                  │
│    "metadata": {                                         │
│      "order_id": "ord-999",                              │
│      "payment_provider": "stripe",                       │
│      "retry_count": 2                                    │
│    }                                                     │
│  }                                                       │
│                                                          │
│  ❌ RUIM:                                                │
│  "ERROR: Payment failed for order ord-999 after 5s"      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### Log Levels — Quando usar cada um

| Level | Quando | Exemplo | Produção |
|-------|--------|---------|----------|
| **TRACE** | Detalhes internos de algoritmo | Entry/exit de funções | ❌ Nunca (custo proibitivo) |
| **DEBUG** | Informação útil para debugging | Query SQL executada, payload recebido | ⚠️ Sampling ou desligado |
| **INFO** | Eventos esperados do fluxo normal | "Order created", "Payment processed" | ✅ Sempre |
| **WARN** | Algo inesperado mas não crítico | Retry necessário, fallback ativado | ✅ Sempre |
| **ERROR** | Falha que impacta funcionalidade | Exception não tratada, timeout | ✅ Sempre + alerta |
| **FATAL** | Sistema não pode continuar | OOM, config inválida, dependency down | ✅ Sempre + page |

#### Structured Logging — Regras

1. **Sempre JSON** — Parseável por máquina
2. **Sempre trace_id** — Correlação com traces
3. **Sempre timestamp ISO 8601** com timezone UTC
4. **Nunca PII em plaintext** — Mascare/tokenize
5. **Nunca log credentials** — Jamais passwords, tokens, keys
6. **Context propagation** — trace_id, span_id, request_id, user_id
7. **Bounded values** — Tipo de erro (enum), não mensagem livre
8. **Custo-consciente** — Log o necessário, não tudo

### Pilar 2: Métricas

```
┌─────────────────────────────────────────────────────────┐
│                      METRICS                             │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  O QUE: Valores numéricos agregados ao longo do tempo    │
│                                                          │
│  TIPOS DE MÉTRICAS:                                      │
│  ┌────────────┐ ┌────────────┐ ┌───────────────┐        │
│  │  Counter   │ │   Gauge    │ │  Histogram    │        │
│  │            │ │            │ │               │        │
│  │ Monotônico │ │ Sobe/desce │ │ Distribuição  │        │
│  │ crescente  │ │            │ │ de valores    │        │
│  │            │ │            │ │               │        │
│  │ Ex: total  │ │ Ex: CPU %  │ │ Ex: latency   │        │
│  │ requests   │ │ mem usage  │ │ P50/P95/P99   │        │
│  │ errors     │ │ queue size │ │ request size  │        │
│  └────────────┘ └────────────┘ └───────────────┘        │
│                                                          │
│  TIPO EXTRA:                                             │
│  ┌────────────┐                                          │
│  │  Summary   │  Similar a Histogram, mas calcula        │
│  │            │  quantiles client-side.                   │
│  │            │  Prefira Histogram (server-side agg).     │
│  └────────────┘                                          │
│                                                          │
│  LABELS / DIMENSIONS:                                    │
│  http_requests_total{method="GET", status="200",         │
│                      endpoint="/api/orders"}             │
│                                                          │
│  ⚠️  REGRA: Labels com cardinalidade BOUNDED             │
│  ✅ method, status_code, endpoint, region                 │
│  ❌ user_id, request_id, order_id (high cardinality)     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Pilar 3: Traces (Distributed Tracing)

```
┌─────────────────────────────────────────────────────────┐
│                   DISTRIBUTED TRACES                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  O QUE: Grafo de causalidade de uma request              │
│         atravessando múltiplos serviços                   │
│                                                          │
│  ANATOMIA:                                               │
│                                                          │
│  Trace (trace_id: abc123)                                │
│  │                                                       │
│  ├─ Span A: API Gateway (root span)                     │
│  │  duration: 250ms                                      │
│  │  ├─ Span B: auth-service                             │
│  │  │  duration: 15ms                                    │
│  │  │  └─ Span C: Redis lookup                          │
│  │  │     duration: 2ms                                  │
│  │  │                                                    │
│  │  ├─ Span D: order-service                            │
│  │  │  duration: 180ms                                   │
│  │  │  ├─ Span E: DynamoDB GetItem                      │
│  │  │  │  duration: 8ms                                  │
│  │  │  ├─ Span F: payment-service (gRPC)                │
│  │  │  │  duration: 150ms  ← BOTTLENECK                 │
│  │  │  │  └─ Span G: Stripe API call                    │
│  │  │  │     duration: 145ms  ← ROOT CAUSE              │
│  │  │  └─ Span H: SQS SendMessage                      │
│  │  │     duration: 5ms                                  │
│  │  │                                                    │
│  │  └─ Span I: notification-service                     │
│  │     duration: 20ms                                    │
│  │     └─ Span J: SES SendEmail                         │
│  │        duration: 18ms                                 │
│                                                          │
│  CONTEXTO PROPAGADO:                                     │
│  traceparent: 00-abc123-spanA-01                        │
│  (W3C Trace Context standard)                            │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Sinais Modernos — Além dos Três Pilares

### Profiling (4º Pilar)

```
┌─────────────────────────────────────────────────────────┐
│                   CONTINUOUS PROFILING                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  O QUE: CPU/Memory flame graphs em produção              │
│  OVERHEAD: < 2% CPU (always-on safe)                     │
│                                                          │
│  RESPONDE:                                               │
│  • "Qual função consome mais CPU?"                       │
│  • "Onde estão as alocações de memória?"                 │
│  • "Por que GC está demorando?"                          │
│  • "Qual código path é lento?"                           │
│                                                          │
│  FERRAMENTAS:                                            │
│  • AWS CodeGuru Profiler (Java, Python)                  │
│  • Grafana Pyroscope (multi-language)                    │
│  • Datadog Continuous Profiler                           │
│  • async-profiler (Java)                                 │
│  • pprof (Go nativo)                                     │
│                                                          │
│  INTEGRAÇÃO COM TRACES:                                  │
│  Trace span → Flame graph do span                        │
│  "Este span demorou 150ms — ONDE no código?"             │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Eventos (5º Sinal)

| Tipo de evento | Exemplo | Correlação |
|---------------|---------|------------|
| **Deployment** | Deploy v2.3.1 at 14:32 UTC | Annotation no dashboard |
| **Feature flag** | Flag `new-checkout` enabled 50% | Segmentar métricas |
| **Config change** | DB pool size: 10 → 50 | Correlacionar com latência |
| **Incident** | Incident #142 opened | Timeline de incidente |
| **Scale event** | ASG scaled 3 → 8 instances | Correlação com load |
| **Infrastructure** | RDS failover completed | Impacto em error rate |

### Tabela de Sinais Completa

| Sinal | Natureza | Custo | Alta cardinalidade | Melhor para |
|-------|----------|-------|-------------------|-------------|
| **Logs** | Eventos discretos | $$$$$ | Sim (cada evento) | Debug, audit, compliance |
| **Metrics** | Agregações numéricas | $ | Não (bounded labels) | Alerting, trending, SLOs |
| **Traces** | Grafos de causalidade | $$$ | Sim (cada request) | Latency, dependencies |
| **Profiles** | Flame graphs contínuos | $$ | Moderada | CPU/Mem optimization |
| **Events** | Mudanças pontuais | $ | Baixa | Change correlation |

---

## Modelos Mentais para Métricas

### USE Method (Brendan Gregg) — Para recursos de infraestrutura

```
┌─────────────────────────────────────────────────┐
│              USE METHOD                          │
│          (para cada RECURSO)                     │
├─────────────────────────────────────────────────┤
│                                                  │
│  U — Utilization: % do tempo que está ocupado    │
│  S — Saturation:  comprimento da fila de espera  │
│  E — Errors:      contagem de erros              │
│                                                  │
│  Recurso       │ U              │ S        │ E   │
│  ─────────────┼────────────────┼──────────┼─────│
│  CPU           │ cpu.usage %    │ load avg │ -   │
│  Memory        │ mem.used %     │ swap use │ OOM │
│  Network       │ bandwidth %    │ queue    │ drop│
│  Disk I/O      │ io.util %      │ io.queue │ err │
│  Thread Pool   │ active/max     │ pending  │ rej │
│  Connection    │ active/max     │ waiting  │ err │
│  Pool                                            │
│                                                  │
└─────────────────────────────────────────────────┘
```

### RED Method (Tom Wilkie) — Para serviços request-driven

```
┌─────────────────────────────────────────────────┐
│              RED METHOD                          │
│         (para cada SERVIÇO)                      │
├─────────────────────────────────────────────────┤
│                                                  │
│  R — Rate:      requests por segundo             │
│  E — Errors:    requests com erro por segundo    │
│  D — Duration:  distribuição de latência         │
│                                                  │
│  Serviço        │ R           │ E       │ D      │
│  ──────────────┼─────────────┼─────────┼────────│
│  API Gateway    │ req/s       │ 5xx/s   │ P50/99 │
│  Order Service  │ orders/s    │ fail/s  │ P50/99 │
│  Payment Svc    │ payments/s  │ err/s   │ P50/99 │
│  Auth Service   │ auth/s      │ deny/s  │ P50/99 │
│                                                  │
│  → RED alinha diretamente com SLIs               │
│                                                  │
└─────────────────────────────────────────────────┘
```

### Google's Four Golden Signals

| Signal | Descrição | Equivalência RED/USE |
|--------|-----------|---------------------|
| **Latency** | Tempo para processar request (success vs error) | RED: Duration |
| **Traffic** | Demanda no sistema (req/s, sessions) | RED: Rate |
| **Errors** | Taxa de erros (explícitos e implícitos) | RED: Errors |
| **Saturation** | Quão "cheio" está o serviço | USE: Saturation |

> **Regra prática:** USE para infra, RED para serviços, Golden Signals como framework mental universal.

---

## Telemetry Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    TELEMETRY PIPELINE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐              │
│  │ Service │ │ Service │ │ Service │ │   Infra  │              │
│  │    A    │ │    B    │ │    C    │ │ (nodes)  │              │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘              │
│       │           │           │           │                     │
│       └───────────┼───────────┼───────────┘                     │
│                   │                                              │
│           SDK / Auto-instrumentation                             │
│           (OpenTelemetry SDK)                                    │
│                   │                                              │
│                   ▼                                              │
│  ┌──────────────────────────────────────┐                       │
│  │       OTel Collector (Agent)          │  ← Sidecar ou        │
│  │                                       │    DaemonSet          │
│  │  Receivers → Processors → Exporters   │                       │
│  │                                       │                       │
│  │  Processors:                          │                       │
│  │  • Batch (buffer antes de enviar)     │                       │
│  │  • Filter (drop debug em prod)        │                       │
│  │  • Attributes (add env, service)      │                       │
│  │  • Sampling (tail-based)              │                       │
│  │  • Transform (rename, redact PII)     │                       │
│  └────────────────┬─────────────────────┘                       │
│                   │                                              │
│                   ▼                                              │
│  ┌──────────────────────────────────────┐                       │
│  │    OTel Collector (Gateway)           │  ← Centralizado      │
│  │                                       │    (opcional)         │
│  │  • Aggregação cross-service           │                       │
│  │  • Tail-based sampling decisão final  │                       │
│  │  • Routing para múltiplos backends    │                       │
│  │  • Rate limiting                      │                       │
│  └────────────────┬─────────────────────┘                       │
│                   │                                              │
│         ┌─────────┼─────────┐                                   │
│         │         │         │                                   │
│         ▼         ▼         ▼                                   │
│  ┌──────────┐ ┌────────┐ ┌──────────┐                          │
│  │  Metrics │ │  Logs  │ │  Traces  │                          │
│  │ Backend  │ │Backend │ │ Backend  │                          │
│  │          │ │        │ │          │                          │
│  │Prometheus│ │  Loki  │ │  Tempo   │                          │
│  │CloudWatch│ │  ES/OS │ │  X-Ray   │                          │
│  │Datadog   │ │CW Logs │ │  Jaeger  │                          │
│  └──────────┘ └────────┘ └──────────┘                          │
│                                                                  │
│  Visualization: Grafana / CloudWatch Dashboards / Datadog       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Deployment Patterns do Collector

| Pattern | Onde roda | Prós | Contras |
|---------|----------|------|---------|
| **Sidecar** | Pod junto com app (K8s) | Isolamento, per-service config | Mais recursos, mais collectors |
| **DaemonSet** | Um por node (K8s) | Menos overhead, node-level metrics | Config compartilhada |
| **Gateway** | Pool centralizado | Sampling global, routing | SPOF se não HA |
| **Agent + Gateway** | Híbrido | Melhor dos dois mundos | Mais complexidade |

> **Recomendação:** DaemonSet (agent) + Gateway (centralizado) para produção.

---

## Instrumentação — Estratégias

### Nível 0: Auto-instrumentação (zero-code)

```
Application ──▶ OTel Agent (Java agent / Python middleware / .NET agent)
                  │
                  Automaticamente captura:
                  • HTTP requests in/out
                  • Database queries
                  • gRPC calls
                  • Message queue operations
                  • Cache operations
```

**Cobertura:** ~60-70% sem escrever código. Suficiente para começar.

### Nível 1: Instrumentação semântica (library-level)

```python
# Pseudocode — Instrumentação com OTel SDK
tracer = get_tracer("order-service")

with tracer.start_span("process_order") as span:
    span.set_attribute("order.id", order_id)
    span.set_attribute("order.total", total)
    span.set_attribute("order.items_count", len(items))
    
    # Business logic
    result = validate_order(order)
    
    if result.has_warnings:
        span.add_event("order.validation_warnings", {
            "warnings": result.warnings
        })
    
    span.set_status(StatusCode.OK)
```

**Cobertura:** ~85%. Adiciona contexto de negócio.

### Nível 2: Instrumentação de domínio (business-level)

```python
# Pseudocode — Métricas de negócio
order_counter = meter.create_counter(
    "orders.created",
    description="Total orders created",
    unit="orders"
)

order_value_histogram = meter.create_histogram(
    "orders.value",
    description="Order value distribution",
    unit="USD"
)

# No código de negócio
order_counter.add(1, {
    "payment_method": "credit_card",
    "channel": "mobile",
    "region": "br-south"
})

order_value_histogram.record(order.total, {
    "category": order.category
})
```

**Cobertura:** ~95%. Métricas que importam para o negócio.

### Estratégia de instrumentação recomendada

```
Semana 1-2:  Auto-instrumentação (zero-code)
             → Visibilidade básica imediata
             
Semana 3-4:  Adicionar trace_id em todos os logs
             → Correlation logs ↔ traces
             
Mês 2:      Instrumentação semântica (spans customizados)
             → Contexto de negócio nos traces
             
Mês 3:      Métricas de negócio (custom metrics)
             → SLIs baseados em behavior real
             
Mês 4+:     Continuous profiling + Events
             → Observabilidade completa
```

---

## High Cardinality — O Problema Central

### O que é

```
CARDINALIDADE = número de combinações únicas de label values

                    Baixa cardinalidade           Alta cardinalidade
                    ─────────────────             ─────────────────
Labels:             method=GET|POST|PUT|DELETE     user_id=1..10M
                    status=2xx|3xx|4xx|5xx         request_id=uuid
                    region=us|eu|ap                order_id=1..50M

Combinações:        4 × 4 × 3 = 48               10.000.000+
Séries temporais:   ~48                            ~10.000.000+
Custo mensal:       ~$5                            ~$50.000+
```

### Regras para controlar cardinalidade

| Regra | Exemplo | Por quê |
|-------|---------|---------|
| **Labels = bounded enums** | `method`, `status_code`, `region` | Previsível, baixo custo |
| **IDs → traces/logs, não metrics** | `user_id` no span, não no counter | Traces suportam high cardinality |
| **Bucketing** | Latência em buckets (5ms, 10ms, 50ms...) | Histogram controlado |
| **Drop em produção** | Debug labels só em staging | Custo proporcional a cardinalidade |
| **Aggregate no collector** | Collector remove labels antes do backend | Reduce antes de store |

### Onde cada sinal brilha

```
HIGH CARDINALITY                              LOW CARDINALITY
(per-request/per-user)                        (aggregated)
        │                                            │
        ▼                                            ▼
   ┌─────────┐                                 ┌──────────┐
   │ TRACES  │  ← per-request, full context    │ METRICS  │
   │ LOGS    │  ← per-event, full context      │          │
   │ PROFILES│  ← per-function                 │ bounded  │
   └─────────┘                                 │ labels   │
                                               └──────────┘

→ Use TRACES para investigar um request específico
→ Use METRICS para alertar e trend sobre agregados
→ Use LOGS para audit e contexto textual
→ Use EXEMPLARS para conectar métricas → traces
```

---

## Correlation — A Cola da Observabilidade

### Como conectar todos os sinais

```
┌─────────────────────────────────────────────────────────────┐
│                      CORRELATION MAP                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  MÉTRICA (alerta dispara)                                    │
│  http_request_duration_seconds{p99} > 500ms                  │
│       │                                                      │
│       │ exemplar: trace_id=abc123                            │
│       ▼                                                      │
│  TRACE (investiga latência)                                  │
│  trace_id: abc123                                            │
│  → Span: payment-service (150ms)                             │
│       │                                                      │
│       │ span_id: span789                                     │
│       ▼                                                      │
│  LOGS (detalhe do erro)                                      │
│  {"trace_id":"abc123", "span_id":"span789",                  │
│   "message":"Stripe timeout after 145ms",                    │
│   "retry_count": 2}                                          │
│       │                                                      │
│       │ timestamp correlation                                │
│       ▼                                                      │
│  PROFILE (onde no código)                                    │
│  Flame graph: 70% do tempo em http.Client.Do()               │
│  → TLS handshake consuming 60ms                              │
│       │                                                      │
│       │ deploy event at -5min                                │
│       ▼                                                      │
│  EVENT (causa raiz)                                          │
│  "Deploy v2.3.1 changed TLS config"                          │
│                                                              │
│  FLUXO: Alerta → Trace → Log → Profile → Event → Root Cause │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Implementação da correlation

| De → Para | Mecanismo | Como |
|-----------|-----------|------|
| Metric → Trace | **Exemplars** | Attach trace_id a data points |
| Trace → Log | **trace_id/span_id** | Incluir IDs em cada log line |
| Log → Trace | **trace_id field** | Click no log → abre trace |
| Trace → Profile | **span profiling** | Profile linked ao span_id |
| Metric → Event | **Annotations** | Deploy markers nos dashboards |
| Trace → Metric | **Span metrics** | Gerar RED metrics de spans |

### Exemplars — O elo métrica ↔ trace

```
# Métrica com exemplar
http_request_duration_seconds_bucket{le="0.5", method="POST"} 1234
    # exemplar: {trace_id="abc123"} 0.48 1708776930

# Significado:
# "Desses 1234 requests ≤ 500ms, aqui está UM exemplo real
#  com trace_id=abc123 que demorou 480ms"
# Click → abre o trace completo no Tempo/Jaeger/X-Ray
```

---

## Alerting Strategy

### Princípio: Alerte sobre SINTOMAS, não CAUSAS

```
❌ CAUSA (não alerte):                ✅ SINTOMA (alerte):
"CPU > 80%"                          "Error rate > SLO threshold"
"Memory > 90%"                       "P99 latency > 500ms"
"Pod restarted"                      "Success rate < 99.9%"
"Disk > 85%"                         "User-facing errors > 0.1%"

POR QUÊ?
• CPU 80% pode ser normal (well-sized)
• Pod restart pode ser update
• Sintomas = impacto real no usuário
```

### Severidade de alertas

| Severidade | Critério | Ação | Notificação | Exemplo |
|-----------|---------|------|-------------|---------|
| **P1 — Critical** | SLO violado, usuários impactados agora | Page imediato, incident | PagerDuty + Slack | Error rate > 5% por 5min |
| **P2 — High** | SLO em risco, degradação detectada | Resposta em < 30min | Slack #oncall | P99 > 2x baseline por 15min |
| **P3 — Medium** | Anomalia, sem impacto imediato | Próximo dia útil | Slack #alerts | Error budget burn rate alto |
| **P4 — Low** | Informativo, otimização | Backlog | Email digest | Cost anomaly, unused resources |

### Multi-window, multi-burn-rate alerting

```
CONCEITO: Alertar baseado em VELOCIDADE de consumo do error budget

Error Budget = 1 - SLO
Se SLO = 99.9%, Error Budget = 0.1% (de 30d = ~43 min de downtime)

┌──────────────────────────────────────────────────┐
│  Burn Rate   │ Janela longa │ Janela curta │ Sev │
│──────────────┼──────────────┼──────────────┼─────│
│  14.4x       │ 1h           │ 5min         │ P1  │
│  6x          │ 6h           │ 30min        │ P2  │
│  3x          │ 3d           │ 6h           │ P3  │
│  1x          │ 30d          │ 3d           │ P4  │
└──────────────────────────────────────────────────┘

Burn rate 14.4x = consome 100% do budget em ~2h → PAGE!
Burn rate 1x    = consome exatamente o budget em 30d → ticket
```

---

## Dashboard Design

### Hierarquia de dashboards

```
┌─────────────────────────────────────────────────────────┐
│  NÍVEL 1: Executive / Platform Overview                  │
│  ─────────────────────────────────────                   │
│  • SLO status de TODOS os serviços (verde/amarelo/verm) │
│  • Error budget remaining (%)                            │
│  • Incident count (last 30d)                             │
│  • Up/Down status                                        │
│  Audience: VP Eng, CTO, Platform Lead                    │
├─────────────────────────────────────────────────────────┤
│  NÍVEL 2: Service Overview (RED)                         │
│  ─────────────────────────────                           │
│  • Rate (req/s), Errors (error%), Duration (P50/P95/P99)│
│  • SLI actual vs SLO target                              │
│  • Error budget burn rate                                │
│  • Dependencies health                                   │
│  • Deploy markers                                        │
│  Audience: Service owner, on-call engineer               │
├─────────────────────────────────────────────────────────┤
│  NÍVEL 3: Debug / Investigation                          │
│  ──────────────────────────────                          │
│  • USE metrics (CPU, Mem, Disk, Network, Pools)          │
│  • Per-endpoint breakdown                                │
│  • Per-instance breakdown                                │
│  • Slow queries, cache hit rates                         │
│  • Recent logs, traces link                              │
│  Audience: On-call engineer during incident              │
├─────────────────────────────────────────────────────────┤
│  NÍVEL 4: Infrastructure                                 │
│  ──────────────────────                                  │
│  • Kubernetes cluster health                             │
│  • Node resources (USE)                                  │
│  • Network topology                                      │
│  • Database replication lag                               │
│  Audience: Platform/SRE team                             │
└─────────────────────────────────────────────────────────┘
```

### Dashboard anti-patterns

| Anti-pattern | Problema | Solução |
|-------------|---------|---------|
| **Wall of graphs** | 47 gráficos sem hierarquia | Organize: Overview → Detail (drill-down) |
| **Vanity metrics** | "1M users!" sem ação possível | Métricas acionáveis que mudam comportamento |
| **No SLO context** | Latência P99 = 200ms — bom ou ruim? | Sempre mostrar target SLO como referência |
| **Missing time context** | Sem markers de deploy/incident | Annotations para correlação temporal |
| **Inconsistent scales** | Cada gráfico com escala diferente | Padronizar unidades e escalas |

---

## Anti-Patterns de Observabilidade

| Anti-pattern | Descrição | Impacto | Solução |
|-------------|-----------|---------|---------|
| **Log and pray** | Logar tudo sem estrutura, torcer para achar | $$$$$ em storage, impossível debugar | Structured logging + sampling |
| **Metric explosion** | Label com user_id = 10M séries | Backend OOM, custo 100x | Bounded labels, use traces para IDs |
| **Alert fatigue** | 200 alertas/dia, ninguém olha | Incidentes reais perdidos | Alerte sintomas, revise regras mensalmente |
| **Dashboard museum** | 47 dashboards, 3 usados | Confusão, dados desatualizados | Archive unused, hierarquia clara |
| **Traces without context** | Trace mostra spans mas sem attributes | "Foi lento mas não sei por quê" | Adicionar business attributes |
| **Silo signals** | Logs, metrics, traces desconectados | Tempo de investigação 10x maior | Correlation: trace_id everywhere |
| **No sampling** | 100% traces em produção, 10K req/s | $$$$$, backend sobrecarregado | Head-based + tail-based sampling |
| **Observability afterthought** | Adicionado 6 meses após o launch | Refactoring doloroso | Instrumente desde o início |
| **Copy-paste dashboards** | Mesmos gráficos para todos os serviços | Métricas irrelevantes, ruído | Templates parametrizados |
| **No runbooks** | Alerta dispara, on-call não sabe o que fazer | MTTR alto, stress do on-call | Alerts vinculados a runbooks |

---

## Observability Maturity Model

| Nível | Nome | Características | Ferramentas típicas |
|-------|------|----------------|-------------------|
| **0 — Ad-hoc** | "Chaotic" | SSH nos servers, `tail -f`, sem dashboards | `grep`, `top` |
| **1 — Reactive** | "Monitoring" | Dashboards básicos, alertas de infra (CPU/disk) | CloudWatch basic, Nagios |
| **2 — Proactive** | "APM" | RED/USE metrics, traces em serviços principais, structured logs | Datadog/New Relic APM |
| **3 — Observability** | "Instrumented" | OTel em tudo, SLOs definidos, correlation, sampling | OTel + Grafana Stack / X-Ray |
| **4 — Data-Driven** | "ODD" | Observability-driven development, error budgets governam releases, feature flags observados | OTel + SLO frameworks + feature flags |
| **5 — Autonomous** | "AIOps" | Anomaly detection, auto-remediation, predictive alerting | ML-powered alerting, runbook automation |

> **Meta realista:** Atingir nível 3-4 em 12-18 meses.

---

## Diretrizes para Code Review assistido por AI

Ao revisar código relacionado a observabilidade, verifique:

1. **Logs não estruturados** — Todos os logs devem ser JSON com `trace_id`, `timestamp`, `level`, `service`
2. **Sem trace propagation** — Toda chamada cross-service deve propagar `traceparent` (W3C)
3. **High cardinality em métricas** — Labels com IDs, emails, caminhos dinâmicos → recuse
4. **Sem correlation** — Logs sem `trace_id` → impossível correlacionar com traces
5. **Alerta em causa, não sintoma** — `CPU > 80%` → substitua por SLI-based alert
6. **PII em logs/spans** — Nunca logar passwords, tokens, CPF, emails em plaintext
7. **Sem error handling em instrumentation** — SDK de telemetria deve ter try/catch (não quebrar a app)
8. **Missing span attributes** — Spans devem ter attributes de negócio relevantes
9. **Sem sampling strategy** — 100% traces em prod → exija configuração de sampling
10. **Sem timeout no exporter** — OTel exporter sem timeout pode bloquear a app
11. **Métrica sem unit** — Toda métrica deve ter unidade definida (ms, bytes, requests)
12. **Counter usado como gauge** — Counters são monotônicos crescentes, gauges sobem/descem
13. **Dashboard sem SLO reference** — Gráficos de latência/erro devem mostrar a linha do SLO target
14. **Alerta sem runbook** — Toda regra de alerta deve ter link para runbook

---

## Referências

- **Observability Engineering** — Charity Majors, Liz Fong-Jones, George Miranda (O'Reilly)
- **Site Reliability Engineering** — Google SRE Book (https://sre.google/sre-book/)
- **The Site Reliability Workbook** — Google (practical companion)
- **Distributed Systems Observability** — Cindy Sridharan (O'Reilly)
- **OpenTelemetry documentation** — https://opentelemetry.io/docs/
- **Brendan Gregg — USE Method** — https://www.brendangregg.com/usemethod.html
- **Tom Wilkie — RED Method** — https://grafana.com/blog/2018/08/02/the-red-method/
- **Google — Four Golden Signals** — SRE Book, Chapter 6
