# Level 5 — Capstone: Production-Ready Observability Platform

> **Objetivo:** Integrar todos os níveis anteriores em uma plataforma de observabilidade production-ready com correlação completa (logs ↔ traces ↔ metrics), hierarquia de dashboards (Executive → Service → Debug), game day exercise, post-mortem real, e relatório de maturidade de observabilidade.

---

## Objetivo de Aprendizado

- Consolidar toda a stack de observabilidade em um sistema coeso
- Implementar correlação bidirecional entre logs, traces e métricas
- Projetar hierarquia de dashboards para diferentes audiências
- Conduzir game day exercício com cenário de incidente realista
- Escrever post-mortem baseado em dados reais da plataforma
- Avaliar maturidade de observabilidade com framework estruturado
- Produzir relatório técnico e apresentação de resultados

---

## Escopo Funcional

### Visão Geral — Plataforma Completa

```
┌────────────────────────────────────────────────────────────────────┐
│                PRODUCTION OBSERVABILITY PLATFORM                    │
│            (Integração de todos os Levels 0-4)                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                APPLICATION LAYER                          │      │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │      │
│  │  │  Order   │ │ Payment  │ │Inventory │ │  Notif   │    │      │
│  │  │ Service  │→│ Service  │→│ Service  │→│ Service  │    │      │
│  │  │          │ │          │ │          │ │          │    │      │
│  │  │ L0: Logs │ │ L0: Logs │ │ L0: Logs │ │ L0: Logs │    │      │
│  │  │ L0: Metr │ │ L0: Metr │ │ L0: Metr │ │ L0: Metr │    │      │
│  │  │ L1: Trac │ │ L1: Trac │ │ L1: Trac │ │ L1: Trac │    │      │
│  │  │ L2: SLIs │ │ L2: SLIs │ │ L2: SLIs │ │ L2: SLIs │    │      │
│  │  │ L4: Flags│ │ L4: Flags│ │          │ │          │    │      │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘    │      │
│  └───────┼─────────────┼───────────┼─────────────┼──────────┘      │
│          │             │           │             │                   │
│          ▼             ▼           ▼             ▼                   │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │              TELEMETRY PIPELINE (L3)                      │      │
│  │  ┌─────────────┐          ┌─────────────────┐            │      │
│  │  │ OTel Agent  │ ────────>│ OTel Gateway    │            │      │
│  │  │ (DaemonSet) │          │ (Deployment)    │            │      │
│  │  │             │          │                  │            │      │
│  │  │ • resourced │          │ • tail_sampling  │            │      │
│  │  │ • batch     │          │ • spanmetrics    │            │      │
│  │  │ • env attrs │          │ • transform      │            │      │
│  │  └─────────────┘          └────────┬─────────┘            │      │
│  └─────────────────────────────────────┼─────────────────────┘      │
│                                        │                             │
│          ┌─────────────────────────────┼──────────────────┐         │
│          ▼                             ▼                  ▼         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │   Jaeger /   │  │  Prometheus  │  │    Loki      │             │
│  │   Tempo      │  │              │  │              │             │
│  │   (Traces)   │  │  (Metrics)   │  │  (Logs)      │             │
│  │              │  │              │  │              │             │
│  │  L1: Spans   │  │  L0: RED/USE │  │  L0: Struct  │             │
│  │  L3: Sampled │  │  L2: SLI/SLO │  │  L1: TraceID │             │
│  │  L3: SpanMet │  │  L3: SpanMet │  │  L3: OTel    │             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│         │                  │                  │                     │
│         └──────────────────┼──────────────────┘                     │
│                            ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                    GRAFANA (Hub)                           │      │
│  │                                                           │      │
│  │  ┌─────────────────────────────────────────────────────┐ │      │
│  │  │ CORRELATION ENGINE                                   │ │      │
│  │  │                                                      │ │      │
│  │  │  Metric → Trace: exemplar link                       │ │      │
│  │  │  Trace → Log: traceID query                          │ │      │
│  │  │  Log → Trace: derived field link                     │ │      │
│  │  │  Metric → Log: label → Loki query                    │ │      │
│  │  └─────────────────────────────────────────────────────┘ │      │
│  │                                                           │      │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │      │
│  │  │Executive│  │ Service │  │  Debug  │  │  SLO   │    │      │
│  │  │Overview │  │ Detail  │  │  Deep   │  │ Comply │    │      │
│  │  │Dashboard│  │Dashboard│  │  Dive   │  │ Report │    │      │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                 OPERATIONAL LAYER (L4)                     │      │
│  │                                                           │      │
│  │  Feature Flags │ Canary Deploy │ Chaos Eng │ Incidents   │      │
│  │  + métricas    │ + SLO gate    │ + observe │ + runbooks  │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

### Correlação Completa — Logs ↔ Traces ↔ Metrics

```
┌──────────────────────────────────────────────────────────────┐
│                CORRELATION MAP                                │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  METRIC → TRACE (via Exemplar):                               │
│  ┌──────────────────────────────┐                             │
│  │ http_request_duration_seconds│                             │
│  │ p99 = 1.2s                   │                             │
│  │ exemplar: traceID=abc123     │──→ Jaeger: trace abc123     │
│  └──────────────────────────────┘                             │
│                                                               │
│  TRACE → LOG (via TraceID):                                   │
│  ┌──────────────────────────────┐                             │
│  │ Jaeger: span "CreateOrder"   │                             │
│  │ traceId: abc123              │──→ Loki: {traceID="abc123"} │
│  │ duration: 1.2s               │                             │
│  └──────────────────────────────┘                             │
│                                                               │
│  LOG → TRACE (via Derived Field):                             │
│  ┌──────────────────────────────┐                             │
│  │ Loki log line:               │                             │
│  │ {"trace_id":"abc123",        │──→ Jaeger: trace abc123     │
│  │  "msg":"order created"}      │                             │
│  └──────────────────────────────┘                             │
│                                                               │
│  METRIC → LOG (via Label):                                    │
│  ┌──────────────────────────────┐                             │
│  │ http_requests_total{         │                             │
│  │   service="order-service",   │──→ Loki: {service=          │
│  │   status="500"               │    "order-service"} |=      │
│  │ }                            │    "error"                   │
│  └──────────────────────────────┘                             │
│                                                               │
│  GRAFANA CONFIG NECESSÁRIO:                                   │
│  1. Prometheus datasource: exemplars enabled                  │
│  2. Loki datasource: derived fields → Jaeger                  │
│  3. Jaeger datasource: trace-to-logs → Loki                   │
│  4. Dashboard: links entre painéis via variables               │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Dashboard Hierarchy

```
┌──────────────────────────────────────────────────────────────┐
│                DASHBOARD HIERARCHY                            │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  TIER 1: Executive Overview                                   │
│  Audiência: VP/Director, on-call lead                         │
│  Refresh: 1 min                                               │
│  ┌──────────────────────────────────────────────┐             │
│  │ • Platform availability (all services): 99.95%│             │
│  │ • Error budget remaining: 72% (order), 45%   │             │
│  │   (payment), 91% (inventory)                  │             │
│  │ • Active incidents: 0 SEV1, 1 SEV3            │             │
│  │ • SLO compliance (30d): 4/4 services green    │             │
│  │ • Total requests/sec: 2,450                    │             │
│  │ • Global p50/p99: 45ms / 380ms                │             │
│  └──────────────────────────────────────────────┘             │
│       │                                                       │
│       ▼                                                       │
│  TIER 2: Service Detail                                       │
│  Audiência: Team lead, on-call engineer                       │
│  Refresh: 30 sec                                              │
│  ┌──────────────────────────────────────────────┐             │
│  │ • RED metrics: Rate, Errors, Duration          │             │
│  │ • SLI ratio timeline (last 24h, 7d, 30d)      │             │
│  │ • Error budget burn rate (1h, 6h, 3d windows) │             │
│  │ • Top 5 endpoints by error rate                │             │
│  │ • Top 5 endpoints by p99 latency               │             │
│  │ • Dependency health: payment ✅ inventory ✅    │             │
│  │ • Recent deploys: v1.9.0 (2h ago)              │             │
│  │ • Feature flags: new-shipping ON (30%)         │             │
│  └──────────────────────────────────────────────┘             │
│       │                                                       │
│       ▼                                                       │
│  TIER 3: Debug Deep Dive                                      │
│  Audiência: Developer debugging                                │
│  Refresh: 15 sec                                              │
│  ┌──────────────────────────────────────────────┐             │
│  │ • USE metrics: Utilization, Saturation, Errors │             │
│  │ • Connection pool: active/idle/max              │             │
│  │ • GC pause times, heap usage                    │             │
│  │ • Kafka consumer lag per partition              │             │
│  │ • DB query times (p50, p95, p99)                │             │
│  │ • Trace search panel (embedded Jaeger)          │             │
│  │ • Log stream panel (Loki query)                 │             │
│  │ • Goroutines / Thread pool active               │             │
│  └──────────────────────────────────────────────┘             │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Escopo Técnico

### Grafana — Correlation Configuration

```yaml
# grafana/provisioning/datasources/datasources.yml
apiVersion: 1

datasources:
  # ── Prometheus ──
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    access: proxy
    isDefault: true
    jsonData:
      exemplarTraceIdDestinations:
        - name: traceID
          datasourceUid: jaeger
          urlDisplayLabel: "View Trace"

  # ── Jaeger ──
  - name: Jaeger
    type: jaeger
    uid: jaeger
    url: http://jaeger:16686
    access: proxy
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki
        spanStartTimeShift: "-1h"
        spanEndTimeShift: "1h"
        filterByTraceID: true
        filterBySpanID: false
        customQuery: true
        query: '{service="$${__span.tags.service.name}"} | json | trace_id = "$${__span.traceId}"'
      tracesToMetrics:
        datasourceUid: prometheus
        spanStartTimeShift: "-1h"
        spanEndTimeShift: "1h"
        tags:
          - key: service.name
            value: service
        queries:
          - name: "Request Rate"
            query: 'rate(http_requests_total{service="$${__tags.service}"}[5m])'
          - name: "Error Rate"
            query: 'rate(http_requests_total{service="$${__tags.service}",status=~"5.."}[5m])'

  # ── Loki ──
  - name: Loki
    type: loki
    uid: loki
    url: http://loki:3100
    access: proxy
    jsonData:
      derivedFields:
        - datasourceUid: jaeger
          matcherRegex: '"trace_id":"(\w+)"'
          name: TraceID
          url: "$${__value.raw}"
          urlDisplayLabel: "View Trace"
```

### Grafana — Dashboard Provisioning

```yaml
# grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1

providers:
  - name: 'Observability Platform'
    orgId: 1
    folder: 'Order Platform'
    folderUid: 'order-platform'
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

```
grafana/dashboards/
├── executive/
│   └── platform-overview.json         # Tier 1
├── service/
│   ├── order-service-detail.json      # Tier 2
│   ├── payment-service-detail.json    # Tier 2
│   └── inventory-service-detail.json  # Tier 2
├── debug/
│   ├── order-service-debug.json       # Tier 3
│   └── infrastructure-debug.json      # Tier 3
└── slo/
    ├── slo-overview.json              # SLO compliance
    └── slo-order-service.json         # SLO detail
```

### Game Day — Cenário Completo

```markdown
## Game Day: Order Platform Resilience Exercise

### Objetivo
Validar que a plataforma de observabilidade detecta, diagnostica e suporta 
a resolução de incidentes em tempo real.

### Equipe
| Role | Pessoa | Responsabilidade |
|------|--------|------------------|
| Game Day Lead | @lead | Injeta falhas, controla cenário |
| Incident Commander | @ic | Declara incidente, coordena |
| On-call Engineer 1 | @eng1 | Investiga Order Service |
| On-call Engineer 2 | @eng2 | Investiga Payment Service |
| Observer | @obs | Documenta timeline, avalia |

### Pré-requisitos
- [ ] Plataforma rodando com todos os serviços (Levels 0-4)
- [ ] Dashboards Tier 1/2/3 configurados
- [ ] SLO alerts habilitados
- [ ] Runbooks escritos e acessíveis
- [ ] k6 gerando tráfego baseline (100 req/s)
- [ ] Equipe NÃO sabe qual falha será injetada

### Cenário (revelado apenas para Game Day Lead)

#### Fase 1: Degradação Gradual (minutos 0-10)
- Injetar latência de 500ms no PostgreSQL do Order Service
- **Expectativa**: p99 sobe, latency SLI começa a degradar
- **Observar**: Dashboard Tier 2 mostra latência subindo?

#### Fase 2: Falha de Dependência (minutos 10-20)
- Parar Payment Service (`docker stop payment-service`)
- **Expectativa**: Order Service erros aumentam, P2 alert dispara
- **Observar**: Circuit breaker abre? Logs mostram fallback?

#### Fase 3: Cascading Effect (minutos 20-30)
- Manter Payment down + aumentar tráfego 3x com k6
- **Expectativa**: P1 alert dispara, burn rate > 14.4x
- **Observar**: IC declarado? Runbook seguido?

#### Fase 4: Recovery (minutos 30-40)
- Restaurar Payment Service
- Remover latência do PostgreSQL
- **Expectativa**: Métricas normalizam, alerts resolvem
- **Observar**: Tempo de recovery? Budget consumed?

### Avaliação (após game day)

| Área | Pergunta | Score (1-5) |
|------|----------|-------------|
| Detection | Quanto tempo até a equipe perceber o problema? | |
| Diagnosis | Os dashboards/traces ajudaram a encontrar root cause? | |
| Correlation | A equipe usou log→trace→metric correlation? | |
| Response | A equipe seguiu o runbook? | |
| Communication | IC comunicou status appropriately? | |
| Recovery | Quanto tempo até full recovery? | |
| SLO | A equipe monitorou error budget durante? | |
| Post-mortem | A equipe produziu post-mortem com action items? | |
```

### Observability Maturity Assessment

```
┌──────────────────────────────────────────────────────────────┐
│             OBSERVABILITY MATURITY MODEL                      │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  LEVEL 1: REACTIVE (Firefighting)                            │
│  ├── Logs: unstructured, grep-based                           │
│  ├── Metrics: basic CPU/memory only                           │
│  ├── Alerts: threshold-based, noisy                           │
│  └── Incidents: ad-hoc, no process                            │
│                                                               │
│  LEVEL 2: MANAGED (Baselines)                                │
│  ├── Logs: structured JSON, centralized                       │
│  ├── Metrics: RED/USE, dashboards exist                       │
│  ├── Traces: basic distributed tracing                        │
│  ├── Alerts: some SLO-based, less noise                       │
│  └── Incidents: severity levels, basic runbooks               │
│                                                               │
│  LEVEL 3: PROACTIVE (SLO-Driven)                             │
│  ├── SLIs/SLOs defined for all critical paths                 │
│  ├── Error budgets tracked, budget policies enforced          │
│  ├── Multi-window burn-rate alerting                          │
│  ├── Full correlation: logs ↔ traces ↔ metrics                │
│  ├── Tail-based sampling, semantic conventions                │
│  └── Blameless post-mortems, game days regular                │
│                                                               │
│  LEVEL 4: OPTIMIZED (ODD)                                    │
│  ├── Observability informs every deploy (canary + SLO gate)  │
│  ├── Feature flags with metrics-driven decisions              │
│  ├── Chaos engineering with measurable hypotheses             │
│  ├── Automated rollback on SLO violations                     │
│  ├── On-call practices maturas, escalation policies           │
│  └── Observability as competitive advantage                   │
│                                                               │
│  META: Seu sistema deve atingir Level 3+ neste capstone.     │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Self-Assessment Checklist

```markdown
## Observability Maturity Self-Assessment

### Instrumentation
- [ ] Structured logging (JSON) em todos os serviços
- [ ] Métricas RED para HTTP e USE para infraestrutura
- [ ] Distributed tracing com OpenTelemetry
- [ ] Trace/span context propagation entre todos os serviços
- [ ] Semantic conventions aplicadas consistentemente
- [ ] Logs incluem trace_id e span_id

### Telemetry Pipeline
- [ ] OTel Collector Agent configurado (resource detection, batch)
- [ ] OTel Collector Gateway com tail-based sampling
- [ ] SpanMetrics gerando RED metrics a partir de traces
- [ ] Memory limiter protegendo Collectors
- [ ] Pipeline handles backpressure gracefully

### SLOs
- [ ] SLIs definidos como ratios (good/valid) para cada serviço
- [ ] SLO targets documentados com rolling windows
- [ ] Error budgets calculados e trackados
- [ ] Multi-window burn-rate alerting (P1/P2/P3)
- [ ] Error budget policy documentada e em uso

### Dashboards & Correlation
- [ ] Tier 1: Executive Overview
- [ ] Tier 2: Service Detail (per service)
- [ ] Tier 3: Debug Deep Dive
- [ ] SLO Overview + Detail dashboards
- [ ] Metric → Trace correlation (exemplars)
- [ ] Trace → Log correlation
- [ ] Log → Trace correlation (derived fields)

### Operational Practices
- [ ] Feature flags com métricas de avaliação
- [ ] Canary deployment com SLO gate
- [ ] Chaos experiments documentados e executados
- [ ] Runbooks para top alerts
- [ ] Incident management com severity levels e roles
- [ ] Blameless post-mortem template em uso
- [ ] Game day exercised at least once

### Score
- 0-10 items: Level 1 (Reactive)
- 11-20 items: Level 2 (Managed)
- 21-30 items: Level 3 (Proactive) ← TARGET
- 31+ items: Level 4 (Optimized)
```

---

## Critérios de Aceite

### Integração Completa
- [ ] Todos os 4 serviços rodando com: structured logs, metrics, traces, SLIs
- [ ] OTel Collector (Agent + Gateway) processando toda a telemetria
- [ ] Tail-based sampling mantendo errors e slow traces
- [ ] SpanMetrics gerando RED metrics a partir dos spans

### Correlação
- [ ] Metric → Trace: exemplar funcional (clicar em ponto no gráfico → abrir trace)
- [ ] Trace → Log: trace no Jaeger → ver logs correlacionados no Loki
- [ ] Log → Trace: log no Loki com link clicável para trace no Jaeger
- [ ] Metric → Log: drill-down de painel de metric para logs do serviço

### Dashboards
- [ ] Tier 1 Executive Overview: availability, budget, active incidents, total RPS
- [ ] Tier 2 Service Detail: RED, SLI timeline, burn rate, top endpoints, deploys
- [ ] Tier 3 Debug: USE, connection pool, GC, consumer lag, trace/log panels
- [ ] SLO Overview: budget remaining, compliance, burn rates
- [ ] Todos os dashboards exportados como JSON no repositório

### Game Day
- [ ] Game day executado com pelo menos 2 fases de falha
- [ ] Timeline documentada: quando detectou, quanto tempo para diagnosticar
- [ ] Equipe usou dashboards e correlation para debug
- [ ] Post-mortem escrito com 5 Whys e Action Items
- [ ] Score de avaliação preenchido

### Maturity Assessment
- [ ] Self-assessment completado
- [ ] Score calculado: Level 3 (Proactive) atingido
- [ ] Gaps identificados com plano de melhoria
- [ ] Relatório técnico produzido

---

## Definição de Pronto (DoD)

- [ ] Todos os critérios de aceite ✅
- [ ] Plataforma completa rodando com Docker Compose
- [ ] `README.md` do projeto com: arquitetura, setup, how-to-run
- [ ] Game day executado e documentado
- [ ] Post-mortem escrito
- [ ] Maturity assessment score ≥ Level 3
- [ ] Relatório técnico: arquitetura, decisões, tradeoffs, resultados
- [ ] Apresentação (5-10 slides): demo + key learnings
- [ ] Commit final: `feat(capstone): production-ready observability platform`

---

## Checklist

### Infrastructure
- [ ] `docker-compose.yml` completo: 4 serviços + Kafka + PostgreSQL + OTel Agent + OTel Gateway + Prometheus + Grafana + Jaeger + Loki
- [ ] `prometheus/prometheus.yml` com scrape configs e recording rules
- [ ] `prometheus/rules/sli-rules.yml` e `prometheus/rules/slo-alerts.yml`
- [ ] `otel-collector/agent-config.yaml` e `otel-collector/gateway-config.yaml`
- [ ] `grafana/provisioning/` com datasources e dashboard providers
- [ ] `grafana/dashboards/` com JSON de todos os dashboards
- [ ] `scripts/` — k6 load test, chaos scripts, canary gate

### Application (por serviço)
- [ ] Structured logging com trace_id/span_id em cada log
- [ ] Métricas HTTP (RED) + métricas de negócio + health checks
- [ ] OTel SDK: tracing + metrics + logs exportados via OTLP
- [ ] Semantic conventions aplicadas nos spans
- [ ] Feature flag no Order Service com métricas

### Documentation
- [ ] `README.md` — Setup, arquitetura, como rodar
- [ ] `docs/sli-specification.md` — SLIs definidos por serviço
- [ ] `docs/slo-targets.md` — SLO targets e error budget policy
- [ ] `docs/runbooks/` — Runbooks para P1 e P2 alerts
- [ ] `docs/incident-management.md` — Severity, roles, escalation
- [ ] `docs/post-mortem-template.md` — Template reutilizável
- [ ] `docs/game-day-report.md` — Relatório do game day
- [ ] `docs/post-mortem-game-day.md` — Post-mortem do incidente simulado
- [ ] `docs/maturity-assessment.md` — Self-assessment com score

### Deliverables Finais
- [ ] `docs/technical-report.md` — Relatório técnico completo
- [ ] `docs/presentation.md` — Slides / outline da apresentação

---

## Tarefas Sugeridas

### Docker Compose — Plataforma Completa

```yaml
# docker-compose.yml (capstone completo)
services:
  # ── Application Services ──
  order-service:
    build: ./order-service
    ports: ["8080:8080"]
    environment:
      DATABASE_URL: postgres://user:pass@postgres:5432/orders
      PAYMENT_SERVICE_URL: http://payment-service:8081
      INVENTORY_SERVICE_URL: http://inventory-service:8082
      KAFKA_BROKERS: kafka:9092
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-agent:4317
      OTEL_SERVICE_NAME: order-service
      OTEL_RESOURCE_ATTRIBUTES: "service.version=1.0.0,deployment.environment.name=staging"
      FEATURE_FLAG_PROVIDER_URL: http://flagd:8013
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
      otel-agent: { condition: service_started }

  payment-service:
    build: ./payment-service
    ports: ["8081:8081"]
    environment:
      DATABASE_URL: postgres://user:pass@postgres:5432/payments
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-agent:4317
      OTEL_SERVICE_NAME: payment-service
    depends_on:
      postgres: { condition: service_healthy }
      otel-agent: { condition: service_started }

  inventory-service:
    build: ./inventory-service
    ports: ["8082:8082"]
    environment:
      DATABASE_URL: postgres://user:pass@postgres:5432/inventory
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-agent:4317
      OTEL_SERVICE_NAME: inventory-service
    depends_on:
      postgres: { condition: service_healthy }
      otel-agent: { condition: service_started }

  notification-service:
    build: ./notification-service
    environment:
      KAFKA_BROKERS: kafka:9092
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-agent:4317
      OTEL_SERVICE_NAME: notification-service
    depends_on:
      kafka: { condition: service_healthy }
      otel-agent: { condition: service_started }

  # ── Infrastructure ──
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.7.0
    ports: ["9092:9092"]
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:29093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:29093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      CLUSTER_ID: 'capstone-observability-01'
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 10s
      timeout: 5s
      retries: 5

  flagd:
    image: ghcr.io/open-feature/flagd:latest
    ports: ["8013:8013"]
    volumes:
      - ./flags/flags.json:/etc/flagd/flags.json
    command: ["start", "--uri", "file:/etc/flagd/flags.json"]

  # ── Telemetry Pipeline ──
  otel-agent:
    image: otel/opentelemetry-collector-contrib:0.115.0
    command: ["--config=/etc/otelcol/agent-config.yaml"]
    volumes:
      - ./otel-collector/agent-config.yaml:/etc/otelcol/agent-config.yaml
    ports:
      - "4317:4317"
      - "4318:4318"
    environment:
      DEPLOYMENT_ENV: staging
    depends_on: [otel-gateway]

  otel-gateway:
    image: otel/opentelemetry-collector-contrib:0.115.0
    command: ["--config=/etc/otelcol/gateway-config.yaml"]
    volumes:
      - ./otel-collector/gateway-config.yaml:/etc/otelcol/gateway-config.yaml
    ports:
      - "14317:4317"
      - "13133:13133"
    depends_on: [jaeger, prometheus, loki]

  # ── Observability Backends ──
  prometheus:
    image: prom/prometheus:v2.54.0
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules:/etc/prometheus/rules
    ports: ["9090:9090"]
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--web.enable-remote-write-receiver'
      - '--enable-feature=exemplar-storage'

  jaeger:
    image: jaegertracing/jaeger:2.1.0
    ports:
      - "16686:16686"
      - "4327:4317"
    environment:
      COLLECTOR_OTLP_ENABLED: "true"

  loki:
    image: grafana/loki:3.0.0
    ports: ["3100:3100"]

  grafana:
    image: grafana/grafana:11.0.0
    ports: ["3000:3000"]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_INSTALL_PLUGINS: ""
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    depends_on: [prometheus, jaeger, loki]
```

### Technical Report — Outline

```markdown
## Relatório Técnico: Production-Ready Observability Platform

### 1. Resumo Executivo
- Problema: falta de visibilidade em sistema distribuído
- Solução: plataforma de observabilidade baseada em OTel + Prometheus + Grafana
- Resultado: detecção de incidentes em < 5min, MTTR reduzido em ~60%

### 2. Arquitetura
- Diagrama da plataforma (application → pipeline → backends → grafana)
- Decisão: Agent + Gateway pattern (why?)
- Decisão: Tail-based sampling (why?)
- Decisão: SpanMetrics vs métricas diretas (tradeoffs)

### 3. Instrumentação
- Structured logging: formato, campos obrigatórios, correlation IDs
- Metrics: RED para HTTP, USE para infraestrutura, business metrics
- Tracing: OTel SDK, semantic conventions, context propagation
- Decisão: auto-instrumentação vs manual (when each)

### 4. SLOs & Alerting
- SLI specification por serviço
- SLO targets e justificativa
- Error budget policy
- Multi-window burn-rate alerting design
- Resultado: false positive rate dos alertas

### 5. Dashboards
- Hierarquia Tier 1/2/3 e audiência
- Correlation setup (metric↔trace↔log)
- Screenshots / dashboard JSON links

### 6. Operational Practices
- Feature flags integration
- Canary com SLO gate
- Chaos experiments e resultados
- Incident management process
- Game day report

### 7. Maturity Assessment
- Self-assessment score
- Gaps identificados
- Plano de melhoria

### 8. Lições Aprendidas
- O que funcionou bem
- O que foi mais difícil
- O que faria diferente
- Recomendações para outras equipes

### 9. Apêndices
- Docker Compose completo
- OTel Collector configs
- Prometheus rules
- Grafana dashboard JSONs
- Runbooks
- Post-mortem
```

---

## Extensões Opcionais

- [ ] Substituir Jaeger por Grafana Tempo para backend all-in-one
- [ ] Implementar Grafana Alerting (Unified Alerting) em vez de Prometheus Alertmanager
- [ ] Deploy em Kubernetes com Helm charts (kube-prometheus-stack, OTel Operator)
- [ ] Implementar ADOT (AWS Distro) exportando para CloudWatch e X-Ray
- [ ] Criar cost analysis: quanto custa o storage de métricas/traces/logs por mês
- [ ] Implementar retention policies: traces 7d, metrics 30d, logs 14d
- [ ] Comparar OpenTelemetry vs Datadog Agent vs New Relic (paper/presentation)
- [ ] Implementar Continuous Profiling com Pyroscope integrado ao Grafana
- [ ] Criar CI/CD pipeline que valida SLO compliance antes de merge (SLO-gate no PR)
