# Zero to Hero — Observability Challenge (Go + Java Multi-Framework)

> **Programa de especialização progressiva** em Observabilidade Avançada, implementado com
> **Go (Gin)** · **Spring Boot** · **Quarkus** · **Micronaut** · **Jakarta EE**

---

## 1. Resumo Executivo

Este programa transforma um desenvolvedor em **especialista em observabilidade de sistemas distribuídos** usando Go e Java com seus principais frameworks, através de desafios práticos que cobrem **fundamentos, distributed tracing, SLOs/SLIs, OpenTelemetry e Observability-Driven Development**.

**Domínio escolhido:** Sistema de **Processamento de Pedidos (Order Processing Platform)** — um domínio multi-serviço que exige observabilidade completa de forma natural.

**Por que Order Processing?**
- **Multi-serviço real** — order-service, payment-service, inventory-service, notification-service
- **Fluxo transacional** — traces end-to-end com spans em múltiplos serviços
- **Métricas de negócio** — taxa de conversão, tempo de processamento, falhas de pagamento
- **SLOs naturais** — latência de checkout < 500ms, availability > 99.9%, freshness de inventário
- **Cenários de falha** — timeout de pagamento, estoque inconsistente, notificação perdida
- **Event-driven** — Kafka/messaging entre serviços gera necessidade de async tracing
- Perguntas de entrevista Staff/Principal frequentemente envolvem design de sistemas observáveis

---

## 2. Arquitetura do Domínio

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Web App    │     │  Mobile App  │     │   Admin      │
│   (Cliente)  │     │  (Cliente)   │     │  Dashboard   │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       ▼                    ▼                    ▼
┌──────────────────────────────────────────────────────────┐
│                    API Gateway                           │
│          (routing, auth, rate limiting)                   │
└──────┬────────────────┬───────────────────┬──────────────┘
       │                │                   │
  ┌────▼────┐    ┌──────▼──────┐     ┌──────▼──────┐
  │  Order  │    │  Payment    │     │  Inventory  │
  │ Service │    │  Service    │     │   Service   │
  └────┬────┘    └──────┬──────┘     └──────┬──────┘
       │                │                   │
       └────────────────┼───────────────────┘
                        │
               ┌────────▼────────┐
               │  Notification   │
               │    Service      │
               └────────┬────────┘
                        │
               ┌────────▼────────┐
               │   Event Bus     │
               │  (Kafka/NATS)   │
               └─────────────────┘

   Observability Stack:
   ┌─────────────────────────────────────────────────────┐
   │  OTel Collector → Prometheus (Metrics)              │
   │                 → Tempo/Jaeger (Traces)             │
   │                 → Loki (Logs)                       │
   │                 → Grafana (Dashboards + Correlation)│
   └─────────────────────────────────────────────────────┘
```

### Entidades Principais

| Entidade | Descrição | Serviço |
|----------|-----------|---------|
| `Order` | Pedido com items, status, total, timestamps | Order Service |
| `Payment` | Pagamento com método, status, transaction_id | Payment Service |
| `InventoryItem` | Item com SKU, quantidade, reservas | Inventory Service |
| `Notification` | Notificação (email, push) com status de entrega | Notification Service |
| `OrderEvent` | Eventos do ciclo de vida (created, paid, shipped) | Event Store |

---

## 3. Progressão dos Desafios

```
Level 0 — Observability Foundations
│  Structured logging (JSON), métricas básicas (RED/USE),
│  health checks, Prometheus + Grafana, correlation ID
│
Level 1 — Distributed Tracing
│  OpenTelemetry SDK, spans customizados, context propagation,
│  Jaeger/Tempo, tracing async (Kafka), span design
│
Level 2 — SLOs, SLIs & Error Budgets
│  Definir SLIs, implementar SLOs, error budget tracking,
│  multi-window burn-rate alerting, SLO dashboards
│
Level 3 — OpenTelemetry Deep Dive
│  OTel Collector pipelines, auto-instrumentação,
│  semantic conventions, sampling strategies, ADOT
│
Level 4 — Observability-Driven Development
│  Feature flags + métricas, canary com SLO gates,
│  chaos engineering, incident management, runbooks
│
Level 5 — Capstone: Production Observability Platform
│  Sistema completo observável, correlação logs↔traces↔metrics,
│  game day, post-mortem, relatório de maturidade
```

---

## 4. Mapeamento Conceito → Desafio

| Conceito (Docs) | Desafio | Aplicação Prática |
|-----------------|---------|-------------------|
| **Três Pilares (Logs, Metrics, Traces)** | Level 0 | Structured logging + Prometheus metrics + health checks |
| **USE/RED Models** | Level 0 | Dashboards com RED para HTTP e USE para infra |
| **Context Propagation** | Level 1 | W3C TraceContext entre order → payment → inventory |
| **Sampling Strategies** | Level 3 | Head-based no SDK, tail-based no Collector |
| **SLI/SLO/SLA** | Level 2 | SLI de availability e latency para Order Service |
| **Error Budgets** | Level 2 | Budget tracking com burn-rate alerting |
| **Multi-Window Alerting** | Level 2 | 1h + 5min windows para P1, 6h + 30min para P2 |
| **OTel SDK** | Level 1, 3 | Instrumentação custom + auto-instrumentation |
| **OTel Collector** | Level 3 | Pipelines: receivers → processors → exporters |
| **Semantic Conventions** | Level 3 | `http.request.method`, `db.system`, `messaging.system` |
| **Feature Flags + Observability** | Level 4 | Flag com métricas de adoption e performance |
| **Canary + SLO Gates** | Level 4 | SLO-gated rollout com rollback automático |
| **Chaos Engineering** | Level 4 | Inject failures + observar SLIs |
| **Incident Management** | Level 4 | IC workflow, blameless post-mortem |
| **ODD Lifecycle** | Level 4, 5 | Design → Build → Verify → Deploy → Release → Observe |

---

## 5. Ferramentas por Stack

| Aspecto | Go (Gin) | Spring Boot | Quarkus | Micronaut | Jakarta EE |
|---|---|---|---|---|---|
| **Métricas** | prometheus/client_golang | Micrometer + Actuator | Micrometer (SmallRye) | Micrometer (built-in) | MicroProfile Metrics |
| **Tracing** | go.opentelemetry.io/otel | Micrometer Tracing + OTel | OpenTelemetry (SmallRye) | Micronaut Tracing + OTel | MicroProfile Telemetry |
| **Logging** | uber-go/zap (structured) | Logback + Logstash encoder | JBoss Logging / SLF4J + JSON | Logback + JSON | JBoss Logging |
| **Health** | custom `/health/*` | Spring Actuator | SmallRye Health | Micronaut Management | MicroProfile Health |
| **OTel SDK** | go.opentelemetry.io/otel | opentelemetry-java | OTel via SmallRye | opentelemetry-java | opentelemetry-java |
| **Auto-instr.** | otelgin middleware | javaagent (-javaagent) | SmallRye auto-instr. | Micronaut OTel | javaagent |
| **Feature Flags** | go-feature-flag | Togglz / OpenFeature | Quarkus OpenFeature | OpenFeature | OpenFeature |

---

## 6. Pré-requisitos

| Requisito | Mínimo | Recomendado |
|-----------|--------|-------------|
| **Go** | 1.24+ | 1.26+ |
| **Java** | JDK 21+ | JDK 25 |
| **Docker** | Docker Compose v2 | Docker Desktop / Podman |
| **Observability Stack** | Prometheus + Grafana | Prometheus + Grafana + Tempo + Loki |
| **Load Test** | k6 | k6 + Grafana Cloud |
| **Backend Experience** | Challenges backend Level 0-3 concluídos | Challenges backend Level 0-7 concluídos |

---

## 7. Referências

- **Observability Engineering** — Charity Majors, Liz Fong-Jones, George Miranda (O'Reilly)
- **Implementing Service Level Objectives** — Alex Hidalgo (O'Reilly)
- **OpenTelemetry Documentation** — https://opentelemetry.io/docs/
- **Grafana Labs Tutorials** — https://grafana.com/tutorials/
- **Site Reliability Engineering** — Betsy Beyer et al. (Google)
- **Docs de referência:** `../.docs/observability/`
