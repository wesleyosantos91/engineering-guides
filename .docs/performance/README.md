# Performance Engineering — Guia Completo

> **Objetivo deste guia:** Servir como referência completa sobre **Performance Engineering** — fundamentos, profiling, capacity planning, benchmarking e latency budgets, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
>
> **Público-alvo:** Engenheiros de software, SREs e arquitetos que buscam garantir performance adequada em sistemas de produção.

---

## Sumário

| Documento | Descrição |
|-----------|-----------|
| [01-performance-foundations](01-performance-foundations.md) | Fundamentos — métricas, modelos mentais, taxonomia de problemas e cultura de performance |
| [02-profiling-and-diagnostics](02-profiling-and-diagnostics.md) | Profiling e Diagnóstico — CPU/memory/I/O profiling, flame graphs, GC analysis e continuous profiling em produção |
| [03-capacity-planning](03-capacity-planning.md) | Capacity Planning — load modeling, forecasting, right-sizing, scaling strategies e cost optimization |
| [04-benchmarking](04-benchmarking.md) | Benchmarking e Load Testing — microbenchmarks, load/stress testing, rigor estatístico, CI/CD integration e regression detection |
| [05-latency-budgets](05-latency-budgets.md) | Latency Budgets e Performance SLOs — alocação por componente, tail latency, fan-out problem e otimização end-to-end |
| [06-dora-space-metrics](06-dora-space-metrics.md) | DORA Metrics & SPACE Framework — métricas de delivery (DF, LT, CFR, MTTR), developer productivity (5 dimensões SPACE), benchmarks, instrumentação e anti-patterns |

---

## Mapa de Decisão

```
O que você quer entender sobre performance?
│
├─ Conceitos e métricas fundamentais → 01-performance-foundations
├─ Identificar gargalos (profiling) → 02-profiling-and-diagnostics
├─ Planejar capacidade → 03-capacity-planning
├─ Medir e validar (benchmarking) → 04-benchmarking
├─ Definir budgets de latência e SLOs → 05-latency-budgets
└─ Medir delivery & produtividade (DORA/SPACE) → 06-dora-space-metrics
```
