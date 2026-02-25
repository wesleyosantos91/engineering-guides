# Observabilidade Avançada — Guia Completo

> **Objetivo deste guia:** Servir como referência completa sobre **Observabilidade Avançada** — fundamentos, distributed tracing, SLOs/SLIs, OpenTelemetry e observability-driven development, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
>
> **Público-alvo:** Engenheiros de software, SREs e arquitetos que buscam implementar observabilidade moderna em sistemas distribuídos.

---

## Sumário

| Documento | Descrição |
|-----------|-----------|
| [01-observability-foundations](01-observability-foundations.md) | Fundamentos — conceitos, os Três Pilares expandidos, sinais modernos, telemetry pipelines, cultura e anti-patterns |
| [02-distributed-tracing](02-distributed-tracing.md) | Distributed Tracing — context propagation, sampling strategies, trace analysis, async tracing, AWS X-Ray e OpenTelemetry |
| [03-slos-slis-error-budgets](03-slos-slis-error-budgets.md) | SLOs, SLIs e Error Budgets — definições, como definir indicadores, error budget policies, multi-window alerting e cultura SRE |
| [04-opentelemetry](04-opentelemetry.md) | OpenTelemetry — arquitetura OTel, SDK, Collector, auto-instrumentação, semantic conventions, deployment patterns e integração com backends |
| [05-observability-driven-development](05-observability-driven-development.md) | Observability-Driven Development — ODD principles, testing in production, progressive delivery, chaos engineering, incident management e on-call |

---

## Mapa de Decisão

```
O que você quer entender sobre observabilidade?
│
├─ Conceitos e fundamentos → 01-observability-foundations
├─ Tracing distribuído → 02-distributed-tracing
├─ SLOs, SLIs e Error Budgets → 03-slos-slis-error-budgets
├─ OpenTelemetry (OTel) → 04-opentelemetry
└─ Observability-Driven Development → 05-observability-driven-development
```
