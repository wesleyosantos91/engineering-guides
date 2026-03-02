# Level 7 — Latency Budgets e Performance SLOs

> **Objetivo:** Dominar latency budgets — alocação de orçamento de latência por componente, tail latency management, fan-out problem, framework SLI→SLO→Error Budget, monitoramento/alerting para SLO violations e budget governance.

---

## Objetivo de Aprendizado

- Definir SLIs (Service Level Indicators) de performance
- Estabelecer SLOs (Service Level Objectives) com targets de latência
- Calcular Error Budgets e usar para tomar decisões (feature velocity vs reliability)
- Decompor latência end-to-end em budgets por componente
- Entender e mitigar o fan-out problem (tail-at-scale)
- Implementar monitoramento de SLO burn rate com alertas
- Criar governance de latency budgets em equipe

---

## Escopo Funcional

### Digital Wallet — Latency Budget Decomposition

```
REQUEST END-TO-END: POST /api/v1/wallets/{id}/transactions/transfer

LATÊNCIA TOTAL SLO: p99 < 500ms

DECOMPOSIÇÃO DO BUDGET:

              ┌─────────────────────────────────────────────────────┐
              │              500ms Budget Total (p99)               │
              ├────────┬────────┬────────┬────────┬────────┬───────┤
              │  LB    │ Auth   │ Biz    │  DB    │  Event │ Ser.  │
              │ 10ms   │ 20ms   │ 50ms   │ 150ms  │ 50ms   │ 20ms  │
              │ (2%)   │ (4%)   │ (10%)  │ (30%)  │ (10%)  │ (4%)  │
              └────────┴────────┴────────┴────────┴────────┴───────┘
              
              + Headroom: 200ms (40%) ← margem para degradação
              = 500ms total

COMPONENTES:
  LB    = Load Balancer (routing, TLS termination)
  Auth  = Autenticação (JWT validation, token lookup)
  Biz   = Lógica de negócio (validação, cálculos)
  DB    = Database (2 queries: debit wallet + credit wallet)
  Event = Publicar evento (Kafka/RabbitMQ produce)
  Ser.  = Serialização (JSON marshal response)
```

---

## Escopo Técnico

### 1. SLI → SLO → Error Budget Framework

```
── SLI (Service Level Indicator) ──

Um SLI é uma MÉTRICA que indica a qualidade do serviço.
Para performance, os SLIs principais são:

  LATÊNCIA:
    SLI = proporção de requests com latência < threshold
    Exemplo: % de requests com p99 < 500ms
    Fórmula: count(latency < 500ms) / count(total_requests)

  AVAILABILITY:
    SLI = proporção de requests que retornam sucesso
    Fórmula: count(status < 500) / count(total_requests)

  THROUGHPUT:
    SLI = taxa de requests processados com sucesso
    Fórmula: rate(successful_requests[5m])

── SLO (Service Level Objective) ──

Um SLO é um TARGET para o SLI em uma janela de tempo.

  Exemplo:
    SLI: latência p99
    SLO: 99.9% dos requests com p99 < 500ms em 30 dias
    
  Significado:
    Em 30 dias com ~2.6M requests:
    Permitido ter até 2.600 requests > 500ms (0.1%)

── Error Budget ──

Error Budget = 1 - SLO target = tolerância permitida

  SLO = 99.9% → Error Budget = 0.1%
  
  Em 30 dias:
    Total requests ≈ 2,600,000
    Budget = 2,600,000 × 0.001 = 2,600 requests "ruins"
    
  Se consumiu 1,300 (50% do budget) em 15 dias → on track
  Se consumiu 2,000 (77% do budget) em 15 dias → slow down deploys
  Se consumiu 2,600 (100% do budget) → FREEZE deploys, focus on reliability
```

### 2. Definindo SLOs por Endpoint

```yaml
# slo-definitions.yaml — SLOs da Digital Wallet API
slos:
  # ── Read endpoints ──
  - name: wallet-balance-latency
    description: "Consulta de saldo deve ser rápida"
    sli:
      type: latency
      endpoint: "GET /api/v1/wallets/{id}"
      threshold_ms: 50
      percentile: 0.99
    target: 0.999     # 99.9%
    window: 30d
    error_budget: 0.001  # 0.1% = ~2600 requests ruins
    owner: team-wallet
    tier: 1  # critical

  - name: transaction-list-latency
    description: "Listagem de transações paginada"
    sli:
      type: latency
      endpoint: "GET /api/v1/wallets/{id}/transactions"
      threshold_ms: 100
      percentile: 0.99
    target: 0.999
    window: 30d
    owner: team-wallet
    tier: 1

  # ── Write endpoints ──
  - name: deposit-latency
    description: "Depósito deve completar rapidamente"
    sli:
      type: latency
      endpoint: "POST /api/v1/wallets/{id}/transactions/deposit"
      threshold_ms: 200
      percentile: 0.99
    target: 0.999
    window: 30d
    owner: team-wallet
    tier: 1

  - name: transfer-latency
    description: "Transferência end-to-end"
    sli:
      type: latency
      endpoint: "POST /api/v1/wallets/{id}/transactions/transfer"
      threshold_ms: 500
      percentile: 0.99
    target: 0.999
    window: 30d
    owner: team-wallet
    tier: 1

  # ── Availability ──
  - name: api-availability
    description: "API deve estar disponível"
    sli:
      type: availability
      endpoints: "all"
      success_status: "<500"
    target: 0.999
    window: 30d
    owner: team-platform
    tier: 0  # highest priority
```

### 3. Implementando SLO Monitoring

```go
// slo/monitor.go — SLO monitoring com Prometheus
package slo

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    // Histograma de latência com buckets alinhados aos SLOs
    requestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "http_request_duration_seconds",
            Help: "Request duration in seconds",
            // Buckets alinhados com SLO thresholds
            Buckets: []float64{
                0.010, // 10ms  (LB budget)
                0.025, // 25ms
                0.050, // 50ms  (wallet-balance SLO)
                0.100, // 100ms (transaction-list SLO)
                0.200, // 200ms (deposit SLO)
                0.500, // 500ms (transfer SLO)
                1.000, // 1s
                2.500, // 2.5s
                5.000, // 5s
                10.00, // 10s (timeout)
            },
        },
        []string{"method", "endpoint", "status_code"},
    )

    // Contador de requests totais (para SLI de availability)
    requestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "endpoint", "status_code"},
    )
)
```

#### Prometheus Recording Rules para SLOs

```yaml
# prometheus/rules/slo-recording.yml
groups:
  - name: slo_recording_rules
    interval: 30s
    rules:
      # ── SLI: % de requests dentro do budget de latência ──
      
      # Wallet Balance: p99 < 50ms → % requests < 50ms
      - record: sli:wallet_balance:latency_good_ratio
        expr: |
          sum(rate(http_request_duration_seconds_bucket{
            endpoint="/api/v1/wallets/{id}",
            method="GET",
            le="0.05"
          }[5m]))
          /
          sum(rate(http_request_duration_seconds_count{
            endpoint="/api/v1/wallets/{id}",
            method="GET"
          }[5m]))

      # Transfer: p99 < 500ms → % requests < 500ms
      - record: sli:transfer:latency_good_ratio
        expr: |
          sum(rate(http_request_duration_seconds_bucket{
            endpoint="/api/v1/wallets/{id}/transactions/transfer",
            method="POST",
            le="0.5"
          }[5m]))
          /
          sum(rate(http_request_duration_seconds_count{
            endpoint="/api/v1/wallets/{id}/transactions/transfer",
            method="POST"
          }[5m]))

      # ── Error Budget: quanto do budget foi consumido ──
      
      # Budget remaining (30 day window)
      - record: slo:wallet_balance:error_budget_remaining
        expr: |
          1 - (
            (1 - sli:wallet_balance:latency_good_ratio)
            /
            (1 - 0.999)  # SLO target = 99.9%
          )

      - record: slo:transfer:error_budget_remaining
        expr: |
          1 - (
            (1 - sli:transfer:latency_good_ratio)
            /
            (1 - 0.999)
          )
```

#### Alertas baseados em Burn Rate

```yaml
# prometheus/rules/slo-alerts.yml
groups:
  - name: slo_burn_rate_alerts
    rules:
      # ── Multi-window, multi-burn-rate alerts ──
      # Baseado em: https://sre.google/workbook/alerting-on-slos/
      
      # CRITICAL: burn rate 14.4x em 1h (consome 2% do budget em 1h)
      - alert: SLO_TransferLatency_BurnRate_Critical
        expr: |
          (
            sli:transfer:latency_error_rate:1h > (14.4 * 0.001)
            and
            sli:transfer:latency_error_rate:5m > (14.4 * 0.001)
          )
        for: 2m
        labels:
          severity: critical
          slo: transfer-latency
          team: wallet
        annotations:
          summary: "Transfer latency SLO burn rate critical"
          description: |
            Transfer latency SLO is burning error budget at 14.4x rate.
            At this rate, 30-day budget will be exhausted in ~2 days.
            Current error rate (1h): {{ $value | humanizePercentage }}
            SLO target: 99.9% requests < 500ms
          runbook: "https://wiki.internal/runbooks/slo-transfer-latency"

      # WARNING: burn rate 6x em 6h (consome 5% do budget em 6h)
      - alert: SLO_TransferLatency_BurnRate_Warning
        expr: |
          (
            sli:transfer:latency_error_rate:6h > (6 * 0.001)
            and
            sli:transfer:latency_error_rate:30m > (6 * 0.001)
          )
        for: 5m
        labels:
          severity: warning
          slo: transfer-latency
          team: wallet
        annotations:
          summary: "Transfer latency SLO burn rate elevated"
          description: |
            Transfer latency SLO is burning error budget at 6x rate.
            At this rate, 30-day budget will be exhausted in ~5 days.

      # INFO: burn rate 3x em 1d (consome 10% do budget em 1d)
      - alert: SLO_TransferLatency_BurnRate_Info
        expr: |
          (
            sli:transfer:latency_error_rate:1d > (3 * 0.001)
            and
            sli:transfer:latency_error_rate:2h > (3 * 0.001)
          )
        for: 15m
        labels:
          severity: info
          slo: transfer-latency
          team: wallet

      # ── Error Budget Exhaustion ──
      - alert: SLO_ErrorBudget_Low
        expr: slo:transfer:error_budget_remaining < 0.25
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Error budget below 25%"
          description: "Only {{ $value | humanizePercentage }} of error budget remaining"

      - alert: SLO_ErrorBudget_Exhausted
        expr: slo:transfer:error_budget_remaining < 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error budget exhausted!"
          description: "SLO target has been violated. Freeze feature deploys."
```

### 4. Fan-out Problem (Tail-at-Scale)

```
O QUE É O FAN-OUT PROBLEM?

Quando um request depende de N sub-requests paralelos,
a latência do request é determinada pelo MAIS LENTO sub-request.

Exemplo: GET /dashboard faz fan-out para 5 microsserviços
  Service A: p99 = 50ms
  Service B: p99 = 50ms
  Service C: p99 = 50ms
  Service D: p99 = 50ms
  Service E: p99 = 50ms

Probabilidade de TODOS responderem < p99:
  P(all < p99) = 0.99^5 = 95.1%

Ou seja: 4.9% dos requests do dashboard excedem 50ms!
Para p99.9: 0.999^5 = 99.5% → ainda aceitável.

COM 20 SERVIÇOS:
  P(all < p99) = 0.99^20 = 81.8% → 18.2% dos requests lentos!
  P(all < p99.9) = 0.999^20 = 98.0% → 2% dos requests lentos!

MITIGAÇÕES:
  1. Hedged requests: enviar para 2 réplicas, usar a resposta mais rápida
  2. Timeouts agressivos: timeout < SLO per component
  3. Degradação graceful: responder parcialmente se 1 serviço falha
  4. Cache: reduzir fan-out com cache em cada serviço
  5. Batch: combinar múltiplos sub-requests em 1
```

```go
// fanout/hedged.go — Hedged requests para mitigar tail latency
package fanout

import (
    "context"
    "time"
)

// HedgedRequest envia request para 2+ backends, retorna o primeiro resultado.
func HedgedRequest(ctx context.Context, backends []string, do func(ctx context.Context, backend string) ([]byte, error), hedgeDelay time.Duration) ([]byte, error) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    type result struct {
        data []byte
        err  error
    }

    ch := make(chan result, len(backends))

    // Enviar para o primeiro backend imediatamente
    go func() {
        data, err := do(ctx, backends[0])
        ch <- result{data, err}
    }()

    // Após hedgeDelay, enviar para o segundo backend
    timer := time.NewTimer(hedgeDelay)
    defer timer.Stop()

    select {
    case r := <-ch:
        return r.data, r.err // Primeiro respondeu antes do hedge
    case <-timer.C:
        // Hedge: enviar para backend adicional
        for _, backend := range backends[1:] {
            go func(b string) {
                data, err := do(ctx, b)
                ch <- result{data, err}
            }(backend)
        }
    }

    // Retornar o primeiro resultado válido
    for range backends {
        select {
        case r := <-ch:
            if r.err == nil {
                return r.data, nil
            }
        case <-ctx.Done():
            return nil, ctx.Err()
        }
    }

    return nil, context.DeadlineExceeded
}
```

### 5. Latency Budget Allocation por Componente

```go
// budget/allocator.go — Budget allocation e tracking
package budget

import (
    "context"
    "fmt"
    "time"

    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/metric"
)

type LatencyBudget struct {
    Component string
    BudgetMs  int64
    meter     metric.Meter
    histogram metric.Int64Histogram
}

func NewLatencyBudget(component string, budgetMs int64, meter metric.Meter) *LatencyBudget {
    hist, _ := meter.Int64Histogram(
        fmt.Sprintf("latency_budget_%s_ms", component),
        metric.WithDescription(fmt.Sprintf("Latency vs budget for %s (budget=%dms)", component, budgetMs)),
        metric.WithUnit("ms"),
    )
    return &LatencyBudget{
        Component: component,
        BudgetMs:  budgetMs,
        meter:     meter,
        histogram: hist,
    }
}

// Track mede a latência e registra se excedeu o budget.
func (lb *LatencyBudget) Track(ctx context.Context, operation func() error) error {
    start := time.Now()
    err := operation()
    elapsed := time.Since(start)
    elapsedMs := elapsed.Milliseconds()

    overBudget := elapsedMs > lb.BudgetMs

    lb.histogram.Record(ctx, elapsedMs,
        metric.WithAttributes(
            attribute.String("component", lb.Component),
            attribute.Bool("over_budget", overBudget),
            attribute.Int64("budget_ms", lb.BudgetMs),
        ),
    )

    if overBudget {
        // Log warning — componente excedeu seu budget
        fmt.Printf("⚠️ BUDGET EXCEEDED: %s took %dms (budget: %dms, over by %dms)\n",
            lb.Component, elapsedMs, lb.BudgetMs, elapsedMs-lb.BudgetMs)
    }

    return err
}

// Exemplo de uso no handler
func TransferHandler(db *sql.DB, eventBus EventBus) http.HandlerFunc {
    authBudget := NewLatencyBudget("auth", 20, meter)
    bizBudget := NewLatencyBudget("business_logic", 50, meter)
    dbBudget := NewLatencyBudget("database", 150, meter)
    eventBudget := NewLatencyBudget("event_publish", 50, meter)

    return func(w http.ResponseWriter, r *http.Request) {
        // Auth: budget = 20ms
        var claims *Claims
        authBudget.Track(r.Context(), func() error {
            var err error
            claims, err = validateJWT(r)
            return err
        })

        // Business logic: budget = 50ms
        var transfer *Transfer
        bizBudget.Track(r.Context(), func() error {
            var err error
            transfer, err = validateAndBuildTransfer(r, claims)
            return err
        })

        // Database: budget = 150ms
        dbBudget.Track(r.Context(), func() error {
            return executeTransfer(r.Context(), db, transfer)
        })

        // Event: budget = 50ms
        eventBudget.Track(r.Context(), func() error {
            return eventBus.Publish(r.Context(), "transfer.completed", transfer)
        })

        respondJSON(w, http.StatusCreated, transfer)
    }
}
```

### 6. Dashboard de SLO e Error Budget

```
── Grafana Dashboard: SLO & Error Budget ──

PAINEL 1: SLI por Endpoint (gauge)
  Cada endpoint como gauge:
    VERDE:  SLI > SLO target (99.9%)
    AMARELO: SLI > 99.5% mas < 99.9%
    VERMELHO: SLI < 99.5%

PAINEL 2: Error Budget Remaining (bar gauge)
  Cada SLO como barra:
    100% ────────────────── 0%
    VERDE   AMARELO   VERMELHO
    
  Query: slo:transfer:error_budget_remaining × 100

PAINEL 3: Error Budget Burn Rate (time series)
  Linhas: burn rate 1h, 6h, 1d
  Thresholds:
    14.4x → critical (2% budget/hora)
    6x   → warning (5% budget/6h)
    3x   → info (10% budget/dia)
    1x   → normal (on track)

PAINEL 4: Latency Budget Breakdown (stacked bar)
  Por componente para o endpoint de transfer:
    |██ Auth 15ms |████ Biz 30ms |████████ DB 120ms |██ Event 40ms |
    Budget: |20ms        |50ms         |150ms              |50ms          |
    Status: ✅ OK        ✅ OK         ✅ OK               ✅ OK          |

PAINEL 5: Budget Violations (table)
  Últimas 24h:
  | Timestamp  | Component | Budget | Actual | Over by |
  | 14:32:15   | database  | 150ms  | 312ms  | 162ms   |
  | 14:35:01   | auth      | 20ms   | 45ms   | 25ms    |
  
PAINEL 6: Fan-out Latency Distribution
  Heatmap por sub-serviço no fan-out
  Highlight: tail latency (p99, p99.9)
```

```json
// grafana/slo-dashboard.json (trecho simplificado)
{
  "title": "SLO & Error Budget Dashboard",
  "panels": [
    {
      "title": "Error Budget Remaining",
      "type": "bargauge",
      "targets": [
        {
          "expr": "slo:wallet_balance:error_budget_remaining * 100",
          "legendFormat": "Wallet Balance"
        },
        {
          "expr": "slo:transfer:error_budget_remaining * 100",
          "legendFormat": "Transfer"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "thresholds": {
            "steps": [
              {"value": 0, "color": "red"},
              {"value": 25, "color": "orange"},
              {"value": 50, "color": "yellow"},
              {"value": 75, "color": "green"}
            ]
          },
          "min": 0,
          "max": 100,
          "unit": "percent"
        }
      }
    }
  ]
}
```

---

## Critérios de Aceite

- [ ] **SLO definitions** em YAML para todos os endpoints da Digital Wallet
- [ ] **Prometheus recording rules** calculando SLIs de latência e availability
- [ ] **Prometheus alerting rules** com multi-window burn rate (3 severidades)
- [ ] **Latency budget decomposition** para POST /transfer (6 componentes)
- [ ] **Budget tracking code** em Go/Java medindo latência por componente
- [ ] **Fan-out analysis**: calcular P(all < p99) para N serviços
- [ ] **Hedged requests** implementado para mitigar tail latency
- [ ] **Grafana dashboard** com Error Budget Remaining + Burn Rate + Budget Breakdown
- [ ] **Error budget policy** documentada: o que fazer quando budget < 25%, < 0%
- [ ] Relatório `LATENCY_BUDGET.md` com:
  - Tabela de SLOs por endpoint
  - Budget decomposition diagram
  - Fan-out analysis
  - Error budget policy

---

## Definição de Pronto (DoD)

- [ ] `slo-definitions.yaml` em `slo/`
- [ ] `prometheus/rules/slo-recording.yml` com SLI recording rules
- [ ] `prometheus/rules/slo-alerts.yml` com burn rate alerts
- [ ] `slo/monitor.go` / SLO middleware instrumentado
- [ ] `budget/allocator.go` com budget tracking por componente
- [ ] `fanout/hedged.go` com hedged requests
- [ ] Dashboard Grafana exportado
- [ ] `LATENCY_BUDGET.md` com análise completa
- [ ] Commit: `feat(perf-level-7): add SLO monitoring, latency budgets, burn rate alerts`

---

## Checklist

### SLI/SLO

- [ ] Definir SLI para cada endpoint (latência + availability)
- [ ] Definir SLO target (99.9% para tier 1, 99.5% para tier 2)
- [ ] Calcular error budget (requests "ruins" permitidos em 30 dias)
- [ ] Histogram buckets alinhados com SLO thresholds

### Prometheus

- [ ] Recording rules para SLI ratio (good / total)
- [ ] Recording rules para error budget remaining
- [ ] Alerting rules com multi-window burn rate
- [ ] Verificar que alerts não são flappy (stabilization windows)

### Latency Budgets

- [ ] Decompor latência end-to-end em componentes
- [ ] Alocar budget por componente (com 30-40% headroom)
- [ ] Instrumentar cada componente com budget tracking
- [ ] Alertar quando componente exceder budget consistentemente

### Fan-out

- [ ] Calcular P(all fast) para fan-out atual
- [ ] Implementar hedged requests onde aplicável
- [ ] Timeout por componente = budget do componente × 2
- [ ] Degradação graceful para componentes não-críticos

---

## Tarefas Sugeridas por Stack

### Go (Gin)

- [ ] Middleware SLI com histograma de latência
- [ ] `budget/allocator.go` com OTel metrics
- [ ] `fanout/hedged.go` com context deadline
- [ ] Dashboard Grafana importável

### Spring Boot

- [ ] Micrometer `Timer` por endpoint com SLO buckets
  ```java
  @Bean
  MeterRegistryCustomizer<MeterRegistry> metricsConfig() {
      return registry -> registry.config()
          .meterFilter(new MeterFilter() {
              @Override
              public DistributionStatisticConfig configure(Meter.Id id, DistributionStatisticConfig config) {
                  if (id.getName().equals("http.server.requests")) {
                      return DistributionStatisticConfig.builder()
                          .serviceLevelObjectives(
                              Duration.ofMillis(50).toNanos(),
                              Duration.ofMillis(100).toNanos(),
                              Duration.ofMillis(200).toNanos(),
                              Duration.ofMillis(500).toNanos()
                          )
                          .build().merge(config);
                  }
                  return config;
              }
          });
  }
  ```
- [ ] Budget tracking com Spring AOP / interceptors
- [ ] `@Timed` annotation com SLO percentiles

### Quarkus / Micronaut / Jakarta EE

- [ ] Micrometer / MicroProfile Metrics com SLO buckets
- [ ] CDI interceptor para budget tracking
- [ ] Health check que inclui error budget status

---

## Error Budget Policy

```
── Política de Error Budget ──

BUDGET > 50% (saudável):
  → Deploy normal
  → Experimentação permitida
  → Frequency: deploy diário

BUDGET 25-50% (atenção):
  → Deploy com feature flag (rollback rápido)
  → Sem mudanças arriscadas de infraestrutura
  → Review obrigatório de performance para PRs grandes
  → Frequency: deploy a cada 2-3 dias

BUDGET 10-25% (alerta):
  → Só deploys com rollback automático
  → Priorizar bugs de performance
  → Reduzir batch size dos deploys
  → Frequency: deploy semanal

BUDGET < 10% (critical):
  → FREEZE de feature deploys
  → Apenas bug fixes e melhorias de reliability
  → Post-mortem obrigatório
  → War room: equipe focada em recuperar budget

BUDGET EXHAUSTED (0%):
  → SLO violado
  → Stakeholder communication
  → Incident response ativado
  → Post-mortem com action items obrigatórios
```

---

## Extensões Opcionais

- [ ] SLO dashboard com Sloth (Prometheus SLO framework)
- [ ] OpenSLO spec para definir SLOs como código
- [ ] Error budget alerting no Slack com burn rate report diário
- [ ] Latency budget heatmap: visualizar qual componente consome mais budget ao longo do tempo
- [ ] Simular fan-out com 10, 20, 50 serviços e medir degradação
- [ ] Implementar adaptive hedging: hedge delay baseado em p50 dinâmico

---

## Erros Comuns

| Erro | Consequência | Correção |
|------|-------------|----------|
| SLO muito agressivo (99.99%) | Budget esgotado constantemente | Começar com 99.9%, ajustar com dados |
| SLO sem janela de tempo | Ambiguidade sobre período de medição | Sempre especificar: 30 dias rolling |
| Alertar em SLI instantâneo | Alerts flaky, ruído | Burn rate com multi-window |
| Bucket de histograma não alinhado | SLI impreciso (interpolação) | Buckets = SLO thresholds exatos |
| Budget sem política de ação | Budget esgota sem resposta | Documentar policy antes |
| Latency budget sem headroom | Qualquer degradação viola SLO | 30-40% headroom obrigatório |
| Ignorar fan-out | p99 do gateway >> p99 dos serviços | Calcular P(all fast) para N serviços |
| SLO = SLA | Sem margem interna | SLO mais apertado que SLA (ex: SLO 99.95% para SLA 99.9%) |

---

## Como Isso Aparece em Entrevistas

- "Explique a relação entre SLI, SLO e Error Budget"
- "Como calcular um error budget? O que fazer quando esgota?"
- "O que é burn rate e por que é melhor que alertar no SLI direto?"
- "Como decompor latência end-to-end em budgets por componente?"
- "O que é o fan-out problem? Como hedged requests ajudam?"
- "Descreva uma política de error budget para uma equipe de produto"
- "Como multi-window burn rate alerts reduzem falsos positivos?"
- "Qual a diferença entre SLO e SLA?"
