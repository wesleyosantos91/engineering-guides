# Observabilidade

> **Categoria:** Observabilidade e Monitoramento
> **Os 3 Pilares:** Métricas, Logs, Traces Distribuídos
> **Complementa:** Health Checks, Circuit Breaker, Rate Limiter

---

## Problema

Em sistemas distribuídos (microsserviços), quando algo falha:

- Qual serviço falhou? Há **dezenas** de serviços.
- A latência aumentou — mas **onde** está o gargalo?
- Um request traversa **5 serviços** — como rastrear a jornada completa?
- Erros estão ocorrendo — são **transientes** ou **persistentes**?
- O sistema está **degradando** gradualmente e ninguém percebe?

**Monitoramento** responde "algo quebrou?". **Observabilidade** responde "**por que** quebrou e **onde**?".

---

## Os 3 Pilares da Observabilidade

```
┌─────────────────────────────────────────────────────────────┐
│                    OBSERVABILIDADE                          │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐   │
│  │  MÉTRICAS   │  │    LOGS     │  │ TRACES DISTRIBUÍDOS│  │
│  │             │  │             │  │                    │   │
│  │ Números     │  │ Eventos     │  │ Jornada do         │   │
│  │ agregados   │  │ detalhados  │  │ request entre      │   │
│  │ ao longo    │  │ por         │  │ serviços           │   │
│  │ do tempo    │  │ ocorrência  │  │                    │   │
│  │             │  │             │  │ A → B → C → D      │   │
│  │ Ex: p99=45ms│  │ Ex: "Erro   │  │ traceId: abc-123   │   │
│  │ error=0.1%  │  │  ao salvar" │  │                    │   │
│  └─────────────┘  └─────────────┘  └──────────────────────┘│
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              CORRELATION ID / TRACE ID               │  │
│  │   Liga os 3 pilares: mesmo ID em métricas,          │  │
│  │   logs e traces para um request específico           │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Pilar 1: Métricas

Métricas são **valores numéricos agregados** ao longo do tempo. Respondem "**quanto**" e "**como está a tendência?**".

### Tipos de Métricas

| Tipo | O que mede | Exemplo |
|------|-----------|---------|
| **Counter** | Quantidade acumulada (só cresce) | Total de requests, total de erros |
| **Gauge** | Valor pontual (sobe e desce) | Memória usada, connections ativas |
| **Histogram** | Distribuição de valores | Latência p50, p95, p99 |
| **Summary** | Percentis calculados client-side | Similar ao histogram |

### RED Method (para serviços)

| Métrica | Descrição | Exemplo |
|---------|-----------|---------|
| **R**ate | Requests por segundo | 500 req/s |
| **E**rrors | Taxa de erro | 0.5% (5xx) |
| **D**uration | Latência dos requests | p50=12ms, p99=85ms |

### USE Method (para infraestrutura)

| Métrica | Descrição | Exemplo |
|---------|-----------|---------|
| **U**tilization | % de uso do recurso | CPU: 65% |
| **S**aturation | Fila de trabalho pendente | Queue depth: 150 |
| **E**rrors | Erros do recurso | Disk errors: 0 |

### Exemplo de Métricas (Pseudocódigo)

```
// Instrumentação de um endpoint
class OrderController:
    requestCounter    // Counter
    errorCounter      // Counter
    latencyHistogram  // Histogram
    
    function createOrder(request):
        timer = latencyHistogram.startTimer()
        requestCounter.increment(labels: {method: "POST", endpoint: "/orders"})
        
        try:
            result = orderService.create(request)
            return result
        catch Exception as e:
            errorCounter.increment(labels: {method: "POST", endpoint: "/orders", error: e.type})
            throw e
        finally:
            timer.observe()  // registra a duração
```

---

## Pilar 2: Logs

Logs são **eventos textuais** com informação detalhada sobre o que aconteceu. Respondem "**o que aconteceu**" em um momento específico.

### Log Estruturado

Logs não estruturados (texto puro) são difíceis de parsear. **Logs estruturados** (JSON) permitem busca, filtragem e agregação:

```
// ❌ Log não estruturado
"2025-01-15 10:30:45 ERROR - Falha ao criar pedido para user 123: timeout no payment-service"

// ✅ Log estruturado (JSON)
{
    "timestamp": "2025-01-15T10:30:45.123Z",
    "level": "ERROR",
    "service": "order-service",
    "traceId": "abc-123-def-456",
    "spanId": "span-789",
    "userId": "user-123",
    "orderId": "order-456",
    "message": "Falha ao criar pedido",
    "error": "TimeoutException",
    "dependency": "payment-service",
    "duration_ms": 5000,
    "context": {
        "retry_attempt": 3,
        "circuit_breaker_state": "HALF_OPEN"
    }
}
```

### Níveis de Log

| Nível | Quando usar | Exemplo |
|-------|-------------|---------|
| **TRACE** | Detalhamento extremo (debug fino) | "Entrando no método X com params Y" |
| **DEBUG** | Informação para debugging | "Query executada em 12ms, 5 resultados" |
| **INFO** | Eventos normais e significativos | "Pedido criado: order-456" |
| **WARN** | Situação inesperada mas recuperável | "Retry 2/3 para payment-service" |
| **ERROR** | Erro que afeta a operação | "Falha ao processar pagamento" |
| **FATAL** | Erro irrecuperável, serviço vai parar | "Não conseguiu conectar ao banco" |

### Boas Práticas de Log

```
// ✅ Inclua SEMPRE o traceId
log.info("Pedido criado", traceId: context.traceId, orderId: order.id)

// ✅ Inclua contexto relevante
log.error("Falha ao processar", 
    traceId: context.traceId, 
    userId: user.id, 
    error: e.message,
    retryAttempt: attempt)

// ❌ NUNCA logue dados sensíveis
log.info("User login", password: user.password)  // NUNCA!
log.info("Payment", creditCard: card.number)      // NUNCA!

// ❌ NUNCA logue dentro de loops de alta frequência
for item in millionsOfItems:
    log.debug("Processing item", item: item.id)  // vai gerar milhões de logs!
```

---

## Pilar 3: Traces Distribuídos

Traces rastreiam a **jornada completa** de um request através de múltiplos serviços. Respondem "**por onde passou**" e "**onde demorou**?".

### Conceitos

| Conceito | Descrição |
|----------|-----------|
| **Trace** | A jornada completa de um request (do início ao fim) |
| **Span** | Uma operação dentro do trace (ex: chamada a um serviço) |
| **TraceId** | ID único que identifica todo o trace |
| **SpanId** | ID único para cada span dentro do trace |
| **ParentSpanId** | Span "pai" — quem chamou este span |

### Visualização

```
Trace: abc-123
├── Span 1: API Gateway (8ms)
│   └── Span 2: order-service (45ms)
│       ├── Span 3: inventory-service (12ms)
│       ├── Span 4: payment-service (25ms)  ← GARGALO
│       │   └── Span 5: fraud-check (18ms)
│       └── Span 6: notification-service (5ms)

Waterfall (timeline):
|-- API Gateway --------|
  |-- order-service ----------------------------------|
    |-- inventory ---|
                      |-- payment-service ------------|
                        |-- fraud-check ---------|
                                                 |-- notification --|
0ms    8ms        20ms        32ms        50ms        55ms
```

### Propagação de Contexto

O **traceId** precisa ser propagado entre serviços:

```
// Serviço A → Serviço B (via HTTP)
Request:
  GET /api/orders/123
  Headers:
    traceparent: 00-abc123def456-span789-01
    // Formato W3C: versão-traceId-spanId-flags

// Serviço B recebe o header e:
// 1. Extrai o traceId
// 2. Cria novo spanId
// 3. Registra parentSpanId = span do serviço A
// 4. Propaga para serviço C com mesmo traceId

// Serviço A → Serviço B (via mensageria/eventos)
Mensagem:
  headers:
    traceparent: "00-abc123def456-span789-01"
  body:
    { "orderId": "order-123", ... }
```

### Exemplo de Instrumentação (Pseudocódigo)

```
// Instrumentação automática com interceptor
class TracingInterceptor:
    tracer
    
    function beforeRequest(request):
        // Extrai ou cria contexto
        parentContext = extractContext(request.headers)
        
        span = tracer.startSpan(
            name: request.method + " " + request.path,
            parent: parentContext,
            attributes: {
                "http.method": request.method,
                "http.url": request.path,
                "service.name": "order-service"
            }
        )
        
        return span
    
    function afterRequest(response, span):
        span.setAttribute("http.status_code", response.status)
        
        if response.status >= 400:
            span.setStatus(ERROR)
        
        span.end()
    
    // Ao chamar outro serviço, propagar o contexto
    function callService(targetUrl, request, currentSpan):
        childSpan = tracer.startSpan("call " + targetUrl, parent: currentSpan)
        
        // Injeta headers de trace na request
        injectContext(request.headers, childSpan)
        
        response = httpClient.send(targetUrl, request)
        childSpan.end()
        return response
```

---

## Correlation ID

O **Correlation ID** (ou Request ID) é um identificador único que acompanha o request por todos os serviços, logs e métricas:

```
Client → API Gateway            → order-service    → payment-service
         correlationId: X-123      correlationId: X-123   correlationId: X-123

Todos os logs incluem correlationId: X-123
Todas as métricas incluem correlationId: X-123
O trace tem traceId que corresponde ao correlationId

Buscar "X-123" retorna: logs + métricas + trace de toda a jornada
```

```
// Middleware que garante correlationId em todos os requests
class CorrelationIdMiddleware:
    function handle(request, next):
        correlationId = request.header("X-Correlation-Id")
        
        if correlationId is null:
            correlationId = generateUUID()  // gera se não vier no header
        
        // Disponibiliza para todo o processamento
        context.set("correlationId", correlationId)
        
        // Inclui na resposta
        response = next(request)
        response.header("X-Correlation-Id", correlationId)
        
        return response
```

---

## SLO / SLI / SLA

| Conceito | Definição | Exemplo |
|----------|-----------|---------|
| **SLI** (Service Level Indicator) | Métrica que mede o nível de serviço | Latência p99, error rate, availability |
| **SLO** (Service Level Objective) | Meta interna para o SLI | "p99 latência < 200ms", "availability > 99.9%" |
| **SLA** (Service Level Agreement) | Contrato com o cliente (com penalidades) | "99.9% uptime, senão crédito de 10%" |

```
Relação:
  SLI = o que você MEDE
  SLO = o que você ALMEJA (meta interna)
  SLA = o que você PROMETE (contrato externo)

  SLA ≤ SLO ≤ real
  Ex: SLA = 99.9% | SLO = 99.95% | Real = 99.97%
```

### Error Budget

```
SLO: 99.9% availability
  Em 30 dias: 30 × 24 × 60 = 43.200 minutos
  Error budget: 0.1% × 43.200 = 43.2 minutos de downtime permitido

  Se já usou 30 min de downtime este mês:
    Budget restante = 13.2 min
    → Freezar deploys arriscados
    → Priorizar estabilidade
```

---

## Alertas

### Boas Práticas de Alertas

| Prática | Descrição |
|---------|-----------|
| **Alerte em sintomas, não em causas** | "Error rate > 1%" (sintoma) é melhor que "CPU > 90%" (causa) |
| **Alerte em SLOs** | "SLO de latência vai ser violado em 2h" |
| **Níveis de severidade** | Critical (pager), Warning (Slack), Info (dashboard) |
| **Runbook** | Todo alerta deve ter um link para runbook de resolução |
| **Sem alert fatigue** | Se o alerta dispara e ninguém age → remover ou ajustar |

### Exemplo de Regras de Alerta

```
// Alerta baseado em SLO (burn rate)
alert: HighErrorRate
  condition: error_rate > 1% por 5 min
  severity: CRITICAL
  action: página SRE de plantão
  runbook: "https://wiki/runbooks/high-error-rate"

alert: LatencySLOBreach
  condition: latency_p99 > 500ms por 15 min
  severity: WARNING
  action: notifica canal #ops no chat
  runbook: "https://wiki/runbooks/high-latency"

alert: ErrorBudgetBurning
  condition: error_budget_consumed > 50% com 50% do mês restante
  severity: WARNING
  action: notifica equipe
  ação: considerar freeze de deploys

alert: ServiceDown
  condition: health_check falha por 2 min
  severity: CRITICAL
  action: página SRE + escala para time owner
```

---

## Dashboard Structure

### Dashboard por Serviço

```
┌─────────────────────────────────────────────┐
│  ORDER-SERVICE Dashboard                     │
│                                              │
│  ┌── RED Metrics ──────────────────────┐    │
│  │ Request Rate:  500 req/s            │    │
│  │ Error Rate:    0.2%                 │    │
│  │ Duration p50:  12ms | p99: 85ms     │    │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌── Resources (USE) ──────────────────┐    │
│  │ CPU: 45%  | Memory: 62%            │    │
│  │ Connections: 80/100 | Queue: 12    │    │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌── Dependencies ─────────────────────┐    │
│  │ payment-service:  OK  (p99: 25ms)  │    │
│  │ inventory-service: OK (p99: 8ms)   │    │
│  │ notification-svc: WARN (p99: 200ms)│    │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌── SLO Status ───────────────────────┐    │
│  │ Availability: 99.97% (target: 99.9%)│    │
│  │ Error Budget: 72% remaining         │    │
│  │ Latency SLO:  OK (p99 < 200ms)     │    │
│  └────────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### Dashboard Global (System Overview)

```
┌─────────────────────────────────────────────────────┐
│  SYSTEM OVERVIEW                                     │
│                                                      │
│  ┌── Service Health ──────────────────────────┐     │
│  │ order-service:     ● HEALTHY               │     │
│  │ payment-service:   ● HEALTHY               │     │
│  │ inventory-service: ● HEALTHY               │     │
│  │ notification-svc:  ◐ DEGRADED              │     │
│  │ report-service:    ● HEALTHY               │     │
│  └───────────────────────────────────────────────┘  │
│                                                      │
│  ┌── Global Metrics ─────────────────────────┐     │
│  │ Total requests:  2,500 req/s              │     │
│  │ Global error rate: 0.15%                  │     │
│  │ Active traces:  1,200                     │     │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## Pipeline de Observabilidade

```
┌───────────┐     ┌──────────────┐     ┌──────────────┐     ┌─────────────┐
│  Serviços │────▶│  Collector   │────▶│   Storage    │────▶│ Visualização│
│           │     │ (Agent/SDK)  │     │              │     │             │
│ Métricas  │     │              │     │ Time-series  │     │ Dashboards  │
│ Logs      │     │ Recebe,      │     │ DB (métricas)│     │ Alertas     │
│ Traces    │     │ processa,    │     │              │     │ Queries     │
│           │     │ exporta      │     │ Log store    │     │             │
└───────────┘     └──────────────┘     │              │     └─────────────┘
                                       │ Trace store  │
                                       └──────────────┘
```

### Padrão: Coleta → Processamento → Armazenamento → Visualização

```
1. COLETA: Serviços emitem métricas, logs, traces via SDK/agent
2. PROCESSAMENTO: Collector recebe, enriquece, filtra, amostra
3. ARMAZENAMENTO: Dados armazenados em backends especializados
4. VISUALIZAÇÃO: Dashboards, alertas, queries ad-hoc
```

### Sampling (Amostragem)

Em produção com alto volume, coletar **100% dos traces** é caro. Sampling reduz o volume:

```
// Head-based sampling (decisão no início)
function shouldSample(traceId):
    return hash(traceId) % 100 < 10  // amostra 10% dos traces

// Tail-based sampling (decisão no final)
function shouldKeep(trace):
    if trace.hasErrors():         return true   // sempre guarda traces com erro
    if trace.duration > 500ms:    return true   // sempre guarda traces lentos
    return random() < 0.05                       // 5% do resto
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Logs sem traceId | Impossível correlacionar logs entre serviços | Inclua traceId em TODO log |
| Logs não estruturados | Difícil de parsear e buscar | Use logs JSON estruturados |
| Métricas de alta cardinalidade | Explosão de séries temporais (ex: userId como label) | Use labels de baixa cardinalidade |
| Alertas em causas (CPU > 90%) | Falsos positivos, alert fatigue | Alerte em sintomas e SLOs |
| Sem sampling em produção | Custo exorbitante de storage | Head-based + tail-based sampling |
| Dashboard sem RED/USE | Não sabe o que monitorar | RED para serviços, USE para infra |
| Observabilidade como afterthought | Adicionada só quando há incidente | Inclua desde o início do projeto |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Health Checks** | Health checks são a forma mais básica de monitoramento |
| **Circuit Breaker** | Estado do CB (OPEN/CLOSED) deve ser uma métrica |
| **Rate Limiter** | Taxa de rejeição do rate limiter é uma métrica importante |
| **Retry** | Número de retries é uma métrica; retries devem propagar traceId |
| **Saga** | Cada step da saga é um span no trace |
| **Canary Release** | Métricas comparam canary vs. stable (observabilidade é essencial!) |

---

## Boas Práticas

1. **Logs estruturados** (JSON) com traceId em toda entry.
2. Use **RED method** para serviços e **USE method** para infraestrutura.
3. Defina **SLIs, SLOs e Error Budgets** desde o início.
4. **Alerte em sintomas e SLOs**, não em causas.
5. Todo alerta deve ter **runbook** vinculado.
6. Propague **traceId/correlationId** entre todos os serviços (HTTP headers, message headers).
7. Use **sampling** em produção para controlar custos (head + tail-based).
8. **Dashboard global** + **dashboard por serviço** — ambos necessários.
9. Inclua observabilidade **desde o início** do projeto, não como afterthought.
10. **Não logue dados sensíveis** (passwords, tokens, PII).

---

## Referências

- Google SRE Book — [Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- Cindy Sridharan — *Distributed Systems Observability* (O'Reilly)
- OpenTelemetry — [Specification](https://opentelemetry.io/docs/)
- W3C — [Trace Context](https://www.w3.org/TR/trace-context/)
- Tom Wilkie — [RED Method](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/)
- Brendan Gregg — [USE Method](https://www.brendangregg.com/usemethod.html)
