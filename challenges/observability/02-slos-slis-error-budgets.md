# Level 2 — SLOs, SLIs & Error Budgets

> **Objetivo:** Definir SLIs baseados na experiência do usuário, implementar SLOs com error budget tracking, configurar multi-window burn-rate alerting e criar dashboards de SLO em Grafana.

---

## Objetivo de Aprendizado

- Entender o framework SLI/SLO/SLA e suas diferenças
- Definir SLIs como ratios (good events / total events) para cada serviço
- Definir SLOs com targets realistas (nunca 100%) e rolling windows
- Calcular error budgets e entender burn rate
- Implementar multi-window, multi-burn-rate alerting
- Criar SLO dashboards no Grafana (budget remaining, burn rate, compliance)
- Implementar recording rules no Prometheus para SLIs
- Definir error budget policies (o que acontece quando budget acaba)
- Filtrar health checks e synthetic traffic dos SLIs

---

## Escopo Funcional

### SLIs para o Order Processing Platform

```
┌─────────────────────────────────────────────────────────────────┐
│                 SLIs DEFINIDOS                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ORDER SERVICE:                                                  │
│  ├── Availability SLI: requests HTTP com status < 500            │
│  │    good: status < 500 (excluindo health checks)               │
│  │    valid: todos os requests (excluindo /health/*)             │
│  │    SLI = count(status < 500) / count(all - health)           │
│  │                                                               │
│  └── Latency SLI: requests HTTP com duração < 500ms             │
│       good: duration < 500ms AND status < 500                    │
│       valid: requests com status < 500 (excluindo health)        │
│       SLI = count(duration < 500ms AND status < 500) / count(OK)│
│                                                                  │
│  PAYMENT SERVICE:                                                │
│  ├── Availability SLI: pagamentos processados com sucesso        │
│  │    good: status 200 em POST /api/v1/payments                  │
│  │    valid: todos os POST /api/v1/payments                      │
│  │                                                               │
│  └── Latency SLI: pagamentos processados em < 1s                │
│       good: duration < 1000ms AND status 200                     │
│       valid: requests 200                                        │
│                                                                  │
│  INVENTORY SERVICE:                                              │
│  └── Availability SLI: reservas processadas com sucesso          │
│       good: status 200 em POST /api/v1/inventory/reserve         │
│       valid: todos os POST /api/v1/inventory/reserve             │
│                                                                  │
│  NOTIFICATION SERVICE (pipeline):                                │
│  └── Freshness SLI: notificações enviadas em < 30s              │
│       good: time_since_order_created < 30s                       │
│       valid: todas as notificações esperadas                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### SLO Targets

| Serviço | SLI | SLO Target | Rolling Window | Error Budget (30d) |
|---------|-----|-----------|----------------|-------------------|
| **Order Service** | Availability | 99.9% | 30 dias | 43.2 min downtime |
| **Order Service** | Latency p99 < 500ms | 99.5% | 30 dias | 3.6 horas slow |
| **Payment Service** | Availability | 99.95% | 30 dias | 21.6 min downtime |
| **Payment Service** | Latency p99 < 1s | 99.0% | 30 dias | 7.2 horas slow |
| **Inventory Service** | Availability | 99.9% | 30 dias | 43.2 min downtime |
| **Notification Service** | Freshness < 30s | 99.0% | 30 dias | 7.2 horas stale |

### Novas Funcionalidades

```
── Métricas de SLI (Prometheus recording rules) ──
sli:order_service:availability          → ratio de disponibilidade
sli:order_service:latency_500ms         → ratio de latência < 500ms
sli:payment_service:availability        → ratio de disponibilidade
slo:order_service:error_budget_remaining → budget restante (%)
slo:order_service:burn_rate_1h          → burn rate janela 1h
slo:order_service:burn_rate_6h          → burn rate janela 6h

── Dashboard Grafana ──
SLO Overview Dashboard: todos os serviços, budget, compliance
SLO Detail Dashboard: por serviço com burn rate e timeline

── Alertas ──
Multi-window burn-rate alerts (P1, P2, P3)
```

---

## Escopo Técnico

### Prometheus Recording Rules para SLIs

```yaml
# prometheus/rules/sli-rules.yml
groups:
  - name: sli.order_service
    interval: 30s
    rules:
      # ── Availability SLI ──
      # Numerador: requests com sucesso (status < 500)
      - record: sli:order_service:availability:good:rate5m
        expr: |
          sum(rate(http_requests_total{
            service="order-service",
            status!~"5..",
            path!~"/health.*|/metrics"
          }[5m]))

      # Denominador: todos os requests válidos
      - record: sli:order_service:availability:total:rate5m
        expr: |
          sum(rate(http_requests_total{
            service="order-service",
            path!~"/health.*|/metrics"
          }[5m]))

      # SLI ratio
      - record: sli:order_service:availability:ratio
        expr: |
          sli:order_service:availability:good:rate5m
          /
          sli:order_service:availability:total:rate5m

      # ── Latency SLI ──
      - record: sli:order_service:latency_500ms:good:rate5m
        expr: |
          sum(rate(http_request_duration_seconds_bucket{
            service="order-service",
            le="0.5",
            path!~"/health.*|/metrics"
          }[5m]))

      - record: sli:order_service:latency_500ms:total:rate5m
        expr: |
          sum(rate(http_request_duration_seconds_count{
            service="order-service",
            path!~"/health.*|/metrics"
          }[5m]))

      - record: sli:order_service:latency_500ms:ratio
        expr: |
          sli:order_service:latency_500ms:good:rate5m
          /
          sli:order_service:latency_500ms:total:rate5m

  - name: slo.order_service
    rules:
      # ── Error Budget Remaining ──
      - record: slo:order_service:availability:error_budget_remaining
        expr: |
          1 - (
            (1 - sli:order_service:availability:ratio)
            /
            (1 - 0.999)
          )
        # Resultado: 1.0 = 100% budget, 0.0 = budget zerado, <0 = violação

      # ── Burn Rate (1h window) ──
      - record: slo:order_service:availability:burn_rate_1h
        expr: |
          (
            1 - (
              sum(increase(http_requests_total{
                service="order-service",
                status!~"5..",
                path!~"/health.*|/metrics"
              }[1h]))
              /
              sum(increase(http_requests_total{
                service="order-service",
                path!~"/health.*|/metrics"
              }[1h]))
            )
          )
          /
          (1 - 0.999)

      # ── Burn Rate (6h window) ──
      - record: slo:order_service:availability:burn_rate_6h
        expr: |
          (
            1 - (
              sum(increase(http_requests_total{
                service="order-service",
                status!~"5..",
                path!~"/health.*|/metrics"
              }[6h]))
              /
              sum(increase(http_requests_total{
                service="order-service",
                path!~"/health.*|/metrics"
              }[6h]))
            )
          )
          /
          (1 - 0.999)
```

### Multi-Window, Multi-Burn-Rate Alerting

```
┌─────────────────────────────────────────────────────────────────┐
│        MULTI-WINDOW BURN-RATE ALERTING                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CONCEITO:                                                       │
│  Burn rate = taxa de consumo do error budget.                   │
│  Burn rate 1 = consumo normal (budget dura 30 dias).            │
│  Burn rate 14.4 = budget se esgota em ~2 horas.                 │
│                                                                  │
│  REGRA: Duas janelas devem violar para disparar alerta.         │
│  Long window: confirma tendência sustentada.                     │
│  Short window: confirma que o problema é AGORA.                 │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Severity │ Long Win │ Short Win │ Burn Rate │ Budget in  │   │
│  │          │          │           │           │            │   │
│  │ P1/SEV1  │   1h     │    5min   │  14.4x    │  ~2h       │   │
│  │ P2/SEV2  │   6h     │   30min   │   6.0x    │  ~5 dias   │   │
│  │ P3/SEV3  │   3d     │    6h     │   1.0x    │  ~30 dias  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  P1: "Estamos queimando budget MUITO rápido, vai acabar em 2h" │
│  P2: "Budget queimando rápido, vai acabar em 5 dias"            │
│  P3: "Budget sendo consumido no ritmo exato, pode acabar"       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Alerting Rules

```yaml
# prometheus/rules/slo-alerts.yml
groups:
  - name: slo.alerts.order_service
    rules:
      # ── P1: Burn rate 14.4x (budget esgota em ~2h) ──
      - alert: OrderServiceAvailabilityBudgetBurnP1
        expr: |
          slo:order_service:availability:burn_rate_1h > 14.4
          AND
          slo:order_service:availability:burn_rate_5m > 14.4
        for: 2m
        labels:
          severity: page
          team: order-platform
          slo: order-service-availability
        annotations:
          summary: "Order Service: error budget burning 14.4x faster (P1)"
          description: |
            Error budget será esgotado em ~2 horas.
            Burn rate 1h: {{ $value | humanize }}x
            Ação imediata necessária.
          runbook: "https://wiki.internal/runbooks/order-service-availability"
          dashboard: "https://grafana.internal/d/slo-order-service"

      # ── P2: Burn rate 6x (budget esgota em ~5 dias) ──
      - alert: OrderServiceAvailabilityBudgetBurnP2
        expr: |
          slo:order_service:availability:burn_rate_6h > 6
          AND
          slo:order_service:availability:burn_rate_30m > 6
        for: 5m
        labels:
          severity: ticket
          team: order-platform
          slo: order-service-availability
        annotations:
          summary: "Order Service: error budget burning 6x faster (P2)"
          description: |
            Error budget será esgotado em ~5 dias.
            Burn rate 6h: {{ $value | humanize }}x
          runbook: "https://wiki.internal/runbooks/order-service-availability"

      # ── P3: Burn rate 1x (budget esgota no prazo) ──
      - alert: OrderServiceAvailabilityBudgetBurnP3
        expr: |
          slo:order_service:availability:burn_rate_3d > 1
          AND
          slo:order_service:availability:burn_rate_6h > 1
        for: 30m
        labels:
          severity: warning
          team: order-platform
          slo: order-service-availability
        annotations:
          summary: "Order Service: error budget at risk (P3)"
          description: |
            Error budget sendo consumido no ritmo de toda a janela.
            Se a tendência continuar, budget esgotará antes do fim do período.
```

### Métricas de SLI por Stack

| Stack | Como expor SLI metrics | Filtrar health checks |
|-------|----------------------|----------------------|
| **Go (Gin)** | prometheus/client_golang counters + histograms | Middleware: `path != "/health/*"` |
| **Spring Boot** | Micrometer `http.server.requests` auto-metric | Tag filter: `uri!~"/actuator.*"` |
| **Quarkus** | SmallRye Micrometer auto-metric | `quarkus.micrometer.binder.http-server.ignore-patterns=/health.*` |
| **Micronaut** | Micrometer auto-metric | `micronaut.metrics.web.server.filter-pattern=^(?!/health).*` |
| **Jakarta EE** | MicroProfile Metrics `@Timed`, `@Counted` | Filter no ContainerRequestFilter |

### SLO Dashboard Design (Grafana)

```
┌──────────────────────────────────────────────────────────────┐
│                SLO OVERVIEW DASHBOARD                         │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌────────────────────┐  ┌────────────────────┐              │
│  │ Error Budget        │  │ SLO Compliance     │              │
│  │ Remaining (30d)     │  │ (rolling 30d)      │              │
│  │                     │  │                     │              │
│  │  order: ████ 72%    │  │  order: 99.93% ✅   │              │
│  │  payment: ██ 45%    │  │  payment: 99.96% ✅ │              │
│  │  inventory: █████ 91│  │  inventory: 99.99%✅│              │
│  │  notif: ████ 68%    │  │  notif: 99.2% ✅    │              │
│  └────────────────────┘  └────────────────────┘              │
│                                                               │
│  ┌────────────────────────────────────────────┐              │
│  │ Error Budget Consumption Timeline          │              │
│  │                                            │              │
│  │  100% ──────────╲                          │              │
│  │                   ╲                        │              │
│  │   72% ─────────────────── current          │              │
│  │                                            │              │
│  │    0% ─────────────────────────── expected  │              │
│  │      Day 1    Day 10    Day 20    Day 30   │              │
│  └────────────────────────────────────────────┘              │
│                                                               │
│  ┌────────────────────┐  ┌────────────────────┐              │
│  │ Burn Rate (1h)     │  │ Burn Rate (6h)     │              │
│  │                     │  │                     │              │
│  │  order: 0.8x ✅     │  │  order: 0.5x ✅     │              │
│  │  payment: 3.2x ⚠️   │  │  payment: 1.8x ⚠️   │              │
│  └────────────────────┘  └────────────────────┘              │
│                                                               │
│  ┌────────────────────────────────────────────┐              │
│  │ SLI Ratio Over Time (per service)          │              │
│  │                                            │              │
│  │  100% ─── ── ─── SLO target (99.9%)       │              │
│  │  99.9%─────────────────────────────        │              │
│  │  99.8%     ╲  incident  ╱                  │              │
│  │  99.7%      ╲──────────╱                   │              │
│  │      00:00  06:00  12:00  18:00  24:00     │              │
│  └────────────────────────────────────────────┘              │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Critérios de Aceite

### SLIs
- [ ] SLI de availability definido e calculado para Order Service e Payment Service
- [ ] SLI de latency definido e calculado para Order Service (< 500ms) e Payment (< 1s)
- [ ] SLI de freshness definido para Notification Service (< 30s)
- [ ] Health checks (`/health/*`) e synthetic traffic filtrados dos SLIs
- [ ] Recording rules no Prometheus calculando SLI ratio a cada 30s

### SLOs
- [ ] SLO target documentado para cada SLI (nunca 100%)
- [ ] Error budget calculado: `budget = 1 - SLO_target` × total events × period
- [ ] Error budget remaining visível em tempo real no Grafana

### Alerting
- [ ] Multi-window burn-rate alerting implementado (P1 + P2 + P3)
- [ ] P1: 1h + 5min window, burn rate > 14.4x → severity `page`
- [ ] P2: 6h + 30min window, burn rate > 6x → severity `ticket`
- [ ] P3: 3d + 6h window, burn rate > 1x → severity `warning`
- [ ] Cada alerta com annotations: `summary`, `description`, `runbook`, `dashboard`

### Dashboard
- [ ] SLO Overview: error budget remaining, SLO compliance, burn rates
- [ ] SLO Detail (por serviço): SLI ratio timeline, budget consumption curve
- [ ] Dashboard exportável como JSON no repositório

### Documentação
- [ ] SLI Specification para cada serviço (tabela: what/good/valid/formula)
- [ ] Error Budget Policy documentada (o que fazer quando budget < 20%, < 0%)

---

## Definição de Pronto (DoD)

- [ ] Todos os critérios de aceite ✅
- [ ] Prometheus rules carregadas e recording rules funcionando
- [ ] Alertas testados (simular falhas e verificar que alerta dispara)
- [ ] Dashboard Grafana com pelo menos 8 painéis de SLO
- [ ] Documentação: SLI spec + SLO targets + Error Budget Policy
- [ ] Commit: `feat(level-2): implement SLOs, SLIs, error budgets with burn-rate alerting`

---

## Checklist

### Prometheus (todos os stacks)

- [ ] `prometheus/rules/sli-rules.yml` — Recording rules para SLIs (availability, latency)
- [ ] `prometheus/rules/slo-alerts.yml` — Alerting rules multi-window burn-rate
- [ ] `prometheus.yml` — Carregar rules files
- [ ] Testar recording rules: `sli:order_service:availability:ratio` retorna valor

### Grafana (todos os stacks)

- [ ] `grafana/dashboards/slo-overview.json` — Overview de todos os SLOs
- [ ] `grafana/dashboards/slo-order-service.json` — Detail do Order Service
- [ ] Variáveis de dashboard: `service`, `slo_target`
- [ ] Panel de error budget remaining com threshold colors (green > 50%, yellow > 20%, red < 20%)

### Go (Gin)

- [ ] Métricas `http_requests_total` e `http_request_duration_seconds` com labels adequados
- [ ] Middleware filtra `/health/*` das métricas usadas para SLI
- [ ] Custom metric: `orders_processing_duration_seconds` (histogram para SLI de negócio)

### Spring Boot

- [ ] Micrometer tags `uri`, `method`, `status` no `http.server.requests`
- [ ] Filtrar Actuator paths do SLI: `management.metrics.web.server.request.autotime.enabled=true`
- [ ] Custom `MeterBinder` para métricas de SLI de negócio (orders processed)

### Quarkus / Micronaut / Jakarta EE

- [ ] Métricas HTTP automáticas com labels adequados
- [ ] Health endpoints filtrados dos SLIs
- [ ] Métricas de negócio expostas para uso em recording rules

---

## Tarefas Sugeridas por Stack

### Error Budget Policy — Template

```markdown
## Error Budget Policy — Order Service

### Thresholds e Ações

| Budget Remaining | Ação | Responsável |
|-----------------|------|-------------|
| **> 50%** | Velocidade normal de desenvolvimento. Feature releases livres. | Team Lead |
| **20% - 50%** | Atenção: priorizar estabilidade. PRs de reliability ganham prioridade. | Team Lead |
| **5% - 20%** | Freeze de features de risco. Foco em bug fixes e reliability. | Engineering Manager |
| **< 5%** | Feature freeze total. Apenas bug fixes e reliability improvements. | Engineering Manager |
| **0% (esgotado)** | Incident review obrigatório. Post-mortem com action items. Freeze até recovery. | VP of Engineering |

### Exceções
- Hotfixes de segurança são sempre permitidos, independente do budget.
- Rollbacks não contam como "mudança" para fins de freeze.

### Revisão
- Budget reviewed semanalmente no standup de segunda-feira.
- SLO targets revisados trimestralmente.
```

### Go — Custom Metrics para SLI de Negócio

```go
// internal/platform/metrics/sli.go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
)

var (
    OrderProcessingDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "orders_processing_duration_seconds",
            Help:    "Time to process an order end-to-end",
            Buckets: []float64{.1, .25, .5, 1, 2.5, 5, 10},
        },
        []string{"status"}, // success, failure
    )

    OrdersProcessedTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "orders_processed_total",
            Help: "Total orders processed",
        },
        []string{"status"}, // success, failure
    )

    NotificationDeliveryDuration = prometheus.NewHistogram(
        prometheus.HistogramOpts{
            Name:    "notification_delivery_duration_seconds",
            Help:    "Time from order creation to notification delivery",
            Buckets: []float64{1, 5, 10, 15, 30, 60},
        },
    )
)

func init() {
    prometheus.MustRegister(
        OrderProcessingDuration,
        OrdersProcessedTotal,
        NotificationDeliveryDuration,
    )
}
```

### Spring Boot — Custom SLI Metrics

```java
// infrastructure/config/SliMetricsConfig.java
@Configuration
public class SliMetricsConfig {

    @Bean
    public MeterBinder orderSliMetrics() {
        return registry -> {
            // Estas métricas são incrementadas no OrderService
            // O Prometheus recording rules calcula o SLI ratio
        };
    }

    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}
```

```java
// domain/service/OrderService.java — com métricas de SLI
@Service
@RequiredArgsConstructor
public class OrderService {

    private final MeterRegistry meterRegistry;
    private final Timer orderProcessingTimer;

    public OrderService(MeterRegistry meterRegistry, /* ... */) {
        this.meterRegistry = meterRegistry;
        this.orderProcessingTimer = Timer.builder("orders.processing.duration")
            .description("Time to process an order end-to-end")
            .publishPercentileHistogram()
            .sla(Duration.ofMillis(500))
            .register(meterRegistry);
    }

    public Order createOrder(CreateOrderRequest request) {
        return orderProcessingTimer.record(() -> {
            try {
                var order = processOrder(request);
                meterRegistry.counter("orders.processed.total", "status", "success").increment();
                return order;
            } catch (Exception e) {
                meterRegistry.counter("orders.processed.total", "status", "failure").increment();
                throw e;
            }
        });
    }
}
```

### Teste de Alertas — Script k6

```javascript
// k6/scenarios/slo-burn-test.js
// Gera erros suficientes para disparar alerta P1 (burn rate > 14.4x)
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  scenarios: {
    error_injection: {
      executor: 'constant-vus',
      vus: 20,
      duration: '10m',
    },
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

export default function () {
  // 30% das requests vão para endpoint que gera 500
  // Isso deve disparar P1 alert em ~5 minutos
  if (Math.random() < 0.3) {
    http.post(`${BASE_URL}/api/v1/orders`, JSON.stringify({
      items: [{ sku: "INVALID-SKU-FORCE-ERROR", quantity: -1 }],
    }), {
      headers: { 'Content-Type': 'application/json' },
    });
  } else {
    http.get(`${BASE_URL}/api/v1/orders?page=0&size=10`);
  }
  sleep(0.1);
}
```

---

## Extensões Opcionais

- [ ] Implementar SLO para Data Pipeline (freshness + correctness)
- [ ] Configurar Grafana Alerting (em vez de Prometheus Alertmanager)
- [ ] Implementar SLO API: endpoint que retorna error budget remaining como JSON
- [ ] Integrar alertas com Slack/Discord via webhook
- [ ] Implementar SLO burn rate como metric Prometheus (observável no service graph)
- [ ] Criar SLO compliance report mensal automatizado
- [ ] Estudar e implementar `sloth` ou `pyrra` como SLO generator
