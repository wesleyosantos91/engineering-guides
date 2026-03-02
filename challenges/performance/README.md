# Zero to Hero — Performance Engineering Challenge (Go + Java Multi-Framework)

> **Programa de especialização progressiva** em Performance Engineering, implementado com
> **Go (Gin)** · **Spring Boot** · **Quarkus** · **Micronaut** · **Jakarta EE**

---

## 1. Resumo Executivo

Este programa transforma um desenvolvedor em **especialista em Performance Engineering** usando Go e Java com seus principais frameworks, através de desafios práticos que cobrem **fundamentos, profiling, load testing (k6, JMeter, Gatling, Locust), benchmarking, capacity planning e latency budgets**.

**Domínio escolhido:** Sistema de **Carteira Digital (Digital Wallet)** + **Order Processing Platform** — os mesmos domínios dos challenges backend e observabilidade, permitindo reutilizar APIs já implementadas e medir performance de cenários reais.

**Por que Performance Engineering?**
- **Performance é percepção do usuário** — +100ms latência = -1% receita (Amazon)
- **Caro se ignorado** — 60-80% de cloud spend é desperdício por falta de capacity planning (Gartner)
- **Transversal** — afeta design, build, test e production
- **Entrevistas Staff/Principal** — quase sempre envolvem diagnóstico de performance, profiling e capacity planning
- **Diferenciador** — poucos engenheiros dominam profiling, load testing com rigor estatístico e latency budgets

---

## 2. Arquitetura do Domínio

```
┌──────────────────────────────────────────────────────────────────┐
│                    SISTEMA ALVO (System Under Test)               │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │  API     │  │  Order   │  │  Payment │  │ Inventory│          │
│  │ Gateway  │→ │ Service  │→ │ Service  │→ │ Service  │          │
│  └──────────┘  └────┬─────┘  └──────────┘  └──────────┘          │
│                     │                                              │
│              ┌──────▼──────┐                                       │
│              │  Database   │                                       │
│              │ (MySQL/PG)  │                                       │
│              └─────────────┘                                       │
│                                                                    │
│  Endpoints alvo:                                                   │
│  POST /api/v1/users                   (CRUD)                      │
│  POST /api/v1/wallets/{id}/deposit    (Transacional)              │
│  POST /api/v1/wallets/{id}/transfer   (Atômico, concorrente)     │
│  GET  /api/v1/wallets/{id}/transactions (Paginado, I/O-bound)    │
│  POST /api/v1/orders                  (Multi-serviço)             │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE TOOLING                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  PROFILING               LOAD TESTING          BENCHMARKING       │
│  ├── async-profiler      ├── k6 (JS)           ├── JMH (Java)    │
│  ├── pprof (Go)          ├── JMeter (GUI)      ├── testing.B (Go)│
│  ├── JFR / JMC           ├── Gatling (Scala)   ├── benchstat     │
│  ├── Pyroscope           ├── Locust (Python)    └── wrk2 (CLI)   │
│  ├── VisualVM            ├── wrk2 (CLI)                          │
│  └── FlameGraph          └── Artillery (YAML)                    │
│                                                                    │
│  OBSERVABILITY           ANALYSIS                                 │
│  ├── Prometheus          ├── HdrHistogram                         │
│  ├── Grafana             ├── benchstat                            │
│  ├── Jaeger/Tempo        ├── GCViewer / GCEasy                   │
│  └── OTel Collector      └── Flame Graph (Brendan Gregg)         │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Progressão dos Desafios

```
Level 0 — Performance Foundations
│  Métricas fundamentais, percentis, USE/RED, Amdahl's Law,
│  Little's Law, queuing theory, anti-patterns, cultura
│
Level 1 — Profiling & Diagnostics
│  CPU/memory/I/O profiling, flame graphs, GC analysis,
│  thread dumps, continuous profiling (Pyroscope)
│
Level 2 — Load Testing com k6
│  Scripts k6, cenários (load/stress/soak/spike),
│  thresholds, custom metrics, CI/CD integration
│
Level 3 — Load Testing com JMeter
│  Test Plans, Thread Groups, Samplers, Listeners,
│  assertions, parametrização, distributed testing
│
Level 4 — Load Testing com Gatling, Locust & Ferramentas
│  Gatling simulations, Locust tasks, wrk2, Artillery,
│  comparação entre ferramentas, coordinated omission
│
Level 5 — Benchmarking & Microbenchmarks
│  JMH (Java), testing.B (Go), benchstat, rigor estatístico,
│  regression detection, CI/CD performance gates
│
Level 6 — Capacity Planning
│  Load modeling, forecasting, right-sizing, scaling strategies,
│  cost optimization, Little's Law aplicado
│
Level 7 — Latency Budgets & Performance SLOs
│  Budget allocation, tail latency, fan-out problem,
│  SLI/SLO/error budget, monitoring, governance
│
Level 8 — Capstone: Performance Engineering Platform
│  Sistema completo com profiling contínuo, load tests
│  automatizados, benchmarks em CI, capacity plan, SLOs,
│  game day de performance, relatório executivo
│
Level 9 — DORA Metrics & SPACE Framework
   Coleta automatizada das 4 métricas DORA (DF, LT, CFR, MTTR),
   classificação Elite/High/Medium/Low, SPACE Framework (5 dimensões),
   dashboards Grafana, instrumentação CI/CD, anti-patterns
```

---

## 4. Mapeamento Conceito → Desafio

| Conceito (Docs) | Desafio | Aplicação Prática |
|-----------------|---------|-------------------|
| **Métricas (latência, throughput, utilização, saturação)** | Level 0 | Instrumentar e medir os 4 golden signals |
| **Percentis e coordinated omission** | Level 0, 2 | Configurar medição correta com k6/wrk2 |
| **Amdahl's Law, Little's Law** | Level 0, 6 | Dimensionar pools e calcular limites teóricos |
| **USE/RED Models** | Level 0, 1 | Dashboards com métricas por recurso e serviço |
| **CPU/Memory/I/O profiling** | Level 1 | Identificar hotspots com async-profiler e pprof |
| **Flame Graphs** | Level 1 | Gerar e interpretar flame graphs interativos |
| **GC Analysis** | Level 1 | Analisar GC logs, tunar G1/ZGC |
| **Continuous Profiling** | Level 1 | Pyroscope em produção com < 2% overhead |
| **Load Test (load/stress/soak/spike)** | Level 2, 3, 4 | Implementar todos os tipos com k6, JMeter, Gatling |
| **k6 scripting** | Level 2 | Scripts JS com cenários, thresholds, custom metrics |
| **JMeter test plans** | Level 3 | GUI → CLI, Thread Groups, parametrização |
| **Gatling simulations** | Level 4 | DSL Scala/Java, relatórios HTML |
| **Locust tasks** | Level 4 | Python, distributed mode, custom shapes |
| **Microbenchmarks (JMH/testing.B)** | Level 5 | Benchmark de hot paths identificados por profiler |
| **Regression detection** | Level 5 | benchstat + CI gates |
| **Capacity Planning** | Level 6 | Load model → resource sizing → cost |
| **Latency Budgets** | Level 7 | Alocar budget por componente do call chain |
| **Performance SLOs** | Level 7 | SLI → SLO → Error Budget com alerting |
| **Fan-out Problem** | Level 7 | Medir e mitigar tail latency amplification |
| **DORA Metrics (DF, LT, CFR, MTTR)** | Level 9 | Coletar, calcular e classificar métricas DORA automaticamente |
| **SPACE Framework** | Level 9 | Instrumentar 5 dimensões (Satisfaction, Performance, Activity, Communication, Efficiency) |
| **DORA Classification** | Level 9 | Classificar equipe: Elite/High/Medium/Low per DORA benchmarks |
| **Engineering Metrics Dashboards** | Level 9 | Dashboards Grafana com trends, breakdown e radar SPACE |

---

## 5. Ferramentas por Stack

| Aspecto | Go (Gin) | Spring Boot | Quarkus | Micronaut | Jakarta EE |
|---|---|---|---|---|---|
| **CPU Profiling** | pprof (built-in) | async-profiler, JFR | async-profiler, JFR | async-profiler, JFR | async-profiler, JFR |
| **Memory Profiling** | pprof heap | MAT, VisualVM | MAT, VisualVM | MAT, VisualVM | MAT, VisualVM |
| **GC Analysis** | runtime/debug | GC logs, GCEasy | GC logs, GCEasy | GC logs, GCEasy | GC logs, GCEasy |
| **Flame Graph** | pprof web | async-profiler HTML | async-profiler HTML | async-profiler HTML | async-profiler HTML |
| **Continuous Profiling** | Pyroscope Go SDK | Pyroscope Java agent | Pyroscope Java agent | Pyroscope Java agent | Pyroscope Java agent |
| **Microbenchmark** | testing.B + benchstat | JMH | JMH | JMH | JMH |
| **Load Test** | k6, wrk2 | k6, JMeter, Gatling | k6, JMeter, Gatling | k6, JMeter, Gatling | k6, JMeter, Gatling |
| **Métricas** | Prometheus client | Micrometer + Actuator | Micrometer (SmallRye) | Micrometer (built-in) | MicroProfile Metrics |
| **Thread Dump** | goroutine dump | jstack, Thread Dump API | jstack | jstack | jstack |

---

## 6. Pré-requisitos

| Requisito | Mínimo | Recomendado |
|-----------|--------|-------------|
| **Go** | 1.24+ | 1.26+ |
| **Java** | JDK 21+ | JDK 25 |
| **Docker** | Docker Compose v2 | Docker Desktop / Podman |
| **k6** | v0.50+ | Última versão |
| **JMeter** | 5.6+ | 5.6.3+ |
| **Gatling** | 3.10+ | 3.12+ |
| **Python** | 3.10+ (para Locust) | 3.12+ |
| **Grafana** | 10+ | 11+ |
| **Prometheus** | 2.50+ | Última versão |
| **Backend Experience** | Challenges backend Level 0-3 | Challenges backend Level 0-7 |

---

## 7. Referências

- **Systems Performance** — Brendan Gregg (Addison-Wesley, 2020)
- **Java Performance** — Scott Oaks (O'Reilly, 2020)
- **BPF Performance Tools** — Brendan Gregg (Addison-Wesley, 2019)
- **The Art of Capacity Planning** — John Allspaw (O'Reilly, 2008)
- **Release It!** — Michael Nygard (Pragmatic Bookshelf, 2018)
- **High Performance Browser Networking** — Ilya Grigorik (O'Reilly)
- **k6 Documentation** — https://k6.io/docs/
- **JMeter Documentation** — https://jmeter.apache.org/usermanual/
- **Gatling Documentation** — https://gatling.io/docs/
- **Locust Documentation** — https://docs.locust.io/
- **JMH** — https://openjdk.org/projects/code-tools/jmh/
- **async-profiler** — https://github.com/async-profiler/async-profiler
- **Pyroscope** — https://pyroscope.io/
- **"How NOT to Measure Latency"** — Gil Tene (Azul Systems)
- **Docs de referência:** `../.docs/performance/`
