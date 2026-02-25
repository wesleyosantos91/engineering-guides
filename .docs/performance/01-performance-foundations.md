# Performance Engineering — Fundamentos

> **Objetivo deste documento:** Servir como referência completa sobre **fundamentos de Performance Engineering** — métricas, modelos mentais, taxonomia de problemas e cultura de performance, otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: o que é performance engineering, métricas fundamentais, leis e modelos, taxonomia de gargalos, performance como disciplina contínua.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **Latência** | Meça percentis (p50/p95/p99), NUNCA média | "Latência média é 50ms, tudo bem" | p99 pode ser 2s — meça tail latency |
| **Throughput** | req/s é vaidade; req/s **dentro do SLO** é o que importa | "Processamos 10K req/s" (com 20% de erros) | Throughput = goodput (successful requests) |
| **Utilização** | Não otimize para 100% — filas explodem acima de ~70% | "CPU a 95% é eficiente" | Queuing theory: latência ∝ 1/(1-ρ) |
| **Saturação** | É o sinal que antecede degradação | Alerta só quando já degradou | Alertar em saturação antes da degradação |
| **Percentis** | p99 afeta 1 em 100 users, p99.9 afeta 1 em 1000 | "Só 1% dos users" = milhares em escala | Em 10M req/dia, p99 = 100K req lentas |
| **Amdahl's Law** | Speedup limitado pela parte não-paralelizável | "Vamos jogar mais CPUs" | Se 5% é serial, max speedup = 20x |
| **Little's Law** | L = λ × W (concurrent = arrival × latency) | Não dimensionar connection pool | Pool = target_throughput × avg_latency |
| **Premature opt.** | Profile PRIMEIRO, otimize DEPOIS | "Reescrevi em assembly para performance" | Dados > intuição. Profile before optimize |

---

## Sumário

- [Performance Engineering — Fundamentos](#performance-engineering--fundamentos)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que é Performance Engineering](#o-que-é-performance-engineering)
  - [Métricas Fundamentais](#métricas-fundamentais)
  - [Percentis e Distribuições](#percentis-e-distribuições)
  - [Leis Fundamentais de Performance](#leis-fundamentais-de-performance)
  - [Queuing Theory para Engenheiros](#queuing-theory-para-engenheiros)
  - [Taxonomia de Gargalos](#taxonomia-de-gargalos)
  - [Modelos Mentais — USE e RED](#modelos-mentais--use-e-red)
  - [Performance como Disciplina Contínua](#performance-como-disciplina-contínua)
  - [Performance Engineering Maturity Model](#performance-engineering-maturity-model)
  - [Anti-Patterns de Performance](#anti-patterns-de-performance)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que é Performance Engineering

```
PERFORMANCE ENGINEERING ≠ PERFORMANCE TESTING

  Performance Testing:
    → Atividade pontual antes do release
    → "O sistema aguenta a carga?"
    → Reativo: descobre problemas in the end

  Performance Engineering:
    → Disciplina contínua desde o design
    → "O sistema é PROJETADO para a carga"
    → Proativo: previne problemas by design

PERFORMANCE ENGINEERING ABRANGE:

  ┌─────────────────────────────────────────────────────────────┐
  │                                                              │
  │  DESIGN TIME                                                 │
  │  ├── Capacity planning: quanto preciso provisionar?          │
  │  ├── Latency budgets: quanto cada componente pode gastar?    │
  │  ├── Architecture review: padrões que afetam performance     │
  │  └── Data model review: queries que vão ser problema         │
  │                                                              │
  │  BUILD TIME                                                  │
  │  ├── Profiling: onde o código gasta tempo/memória?           │
  │  ├── Benchmarking: microbenchmarks e componentes isolados    │
  │  ├── Code review: patterns de performance (N+1, lock, etc)  │
  │  └── Load modeling: qual é o perfil de carga real?           │
  │                                                              │
  │  TEST TIME                                                   │
  │  ├── Load testing: comportamento sob carga esperada          │
  │  ├── Stress testing: ponto de quebra                         │
  │  ├── Soak testing: degradação ao longo de horas/dias         │
  │  └── Chaos engineering: comportamento sob falhas             │
  │                                                              │
  │  RUN TIME                                                    │
  │  ├── Continuous profiling: overhead < 2%                     │
  │  ├── Observability: métricas, traces, logs correlacionados   │
  │  ├── Auto-scaling: reagir a mudanças de carga               │
  │  └── Performance regression detection: CI/CD gates           │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘
```

### O custo de NÃO fazer performance engineering

```
CUSTO DA IGNORÂNCIA:

  1. DEGRADAÇÃO GRADUAL
     → Latência cresce 5% ao mês
     → Ninguém percebe até virar incidente P1
     → "Boiling frog" — sapo na panela quente

  2. OVER-PROVISIONING
     → Sem capacity planning: "coloca instância maior"
     → Custo cresce linearmente enquanto eficiência cai
     → 60-80% de cloud spend é desperdício (Gartner)

  3. INCIDENTES EM PICO
     → Black Friday, lançamento de feature, viralização
     → Sistema não foi dimensionado
     → Downtime = receita perdida + reputação

  4. REWRITE TARDIO
     → Arquitetura não suporta escala
     → "Precisamos reescrever tudo"
     → Custo 10-100x maior que acertar no design

  5. PERCEPÇÃO DO USUÁRIO
     → +100ms latência = -1% receita (Amazon)
     → +500ms load time = -20% traffic (Google)
     → Users não esperam > 3s (53% abandono mobile)
```

---

## Métricas Fundamentais

### As quatro métricas-chave

```
┌─────────────────────────────────────────────────────────────┐
│          MÉTRICAS FUNDAMENTAIS DE PERFORMANCE                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. LATÊNCIA (Response Time)                                 │
│     Quanto tempo leva para completar uma operação.           │
│     ┌───────────────────────────────────────────┐            │
│     │ Request ──► Processing ──► Response        │            │
│     │ ←───── Latência total (end-to-end) ──────→│            │
│     └───────────────────────────────────────────┘            │
│     Componentes:                                             │
│       Network latency + Queue wait + Processing + I/O        │
│                                                              │
│  2. THROUGHPUT (Vazão)                                       │
│     Quantas operações por unidade de tempo.                  │
│     → req/s, TPS, msg/s, MB/s                               │
│     → GOODPUT = throughput de requests BEM-SUCEDIDAS         │
│     → Throughput sem SLO compliance é vaidade                │
│                                                              │
│  3. UTILIZAÇÃO (Resource Usage)                              │
│     Fração do tempo que um recurso está ocupado.             │
│     → CPU%, Memory%, Disk I/O%, Network bandwidth            │
│     → 0% = ocioso, 100% = saturado                          │
│     → Atenção: utilização alta ≠ performance boa             │
│                                                              │
│  4. SATURAÇÃO (Queue Depth)                                  │
│     Trabalho que não pode ser atendido — está na fila.       │
│     → Thread pool queue depth                                │
│     → Connection pool wait count                             │
│     → Disk I/O queue length                                  │
│     → Leading indicator: sinal ANTES da degradação           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Relação entre métricas

```
RELAÇÃO ENTRE LATÊNCIA, THROUGHPUT E UTILIZAÇÃO:

  Throughput (req/s)
  ▲
  │            ┌────── Throughput máximo (saturação)
  │           /│
  │          / │
  │         /  │
  │        /   │
  │       /    │  ← Throughput degrada após saturação
  │      /     │     (timeouts, retries, erros)
  │     /      │
  │    /       │\
  │   /        │ \
  │  /         │  \
  │ /          │   \
  ├─────────────┼────────────────► Utilização (%)
  0%          ~70%    100%

  ZONA SEGURA: utilização < 70%
  ZONA DE RISCO: 70-85% (latência cresce não-linearmente)
  ZONA DE PERIGO: > 85% (filas explodem, timeouts)
  
  REGRA: Nunca planeje para 100% de utilização.
  Queuing theory: latência ∝ 1/(1 - ρ)
  A 90% utilização, latência é ~10x a de 50%.
```

---

## Percentis e Distribuições

### Por que média é mentira

```
MÉDIA É MENTIRA EM PERFORMANCE:

  Cenário: 100 requests
  99 requests: 10ms
  1 request: 10,000ms (10 segundos)

  Média: (99 × 10 + 1 × 10000) / 100 = 109ms
  → "Latência média de 109ms" parece aceitável

  Mas:
  p50 (mediana) = 10ms    ← metade das requests
  p99           = 10000ms ← 1% das requests = LENTA
  
  Se você tem 1M requests/dia:
  p99 = 10,000 requests com 10 SEGUNDOS de latência
  
  → Média esconde o sofrimento dos piores casos
  → Percentis revelam a distribuição real
```

### O espectro de percentis

```
PERCENTIS ESSENCIAIS:

  p50  (mediana) → Experiência "típica"
  p90            → 10% piores requests
  p95            → 5% piores requests (common SLO target)
  p99            → 1% piores requests (principal SLO target)
  p99.9          → 1 em 1000 (tail latency)
  p99.99         → 1 em 10000 (extreme tail)

  EM ESCALA:

  │ Volume diário │ p99 afeta  │ p99.9 afeta │
  │ 100K req/dia  │ 1,000 req  │ 100 req     │
  │ 1M req/dia    │ 10,000 req │ 1,000 req   │
  │ 100M req/dia  │ 1M req     │ 100,000 req │

  REGRA: Em sistemas de escala, TAIL LATENCY IMPORTA.
  Jeff Dean (Google): "At scale, tail latency is everything."
```

### Coordinated omission

```
COORDINATED OMISSION — O ERRO QUE QUASE TODOS COMETEM:

  CENÁRIO:
  Load test: enviar 1 request por segundo
  Normal: response em 50ms
  Spike: requests 50-60 demoram 5 segundos cada

  MEDIÇÃO ERRADA (coordinated omission):
    Tool envia request 50 → demora 5s
    Tool espera terminar → envia request 51 no segundo 55
    Resultado: "p99 = 5s" (parece 1 request lenta)

  MEDIÇÃO CORRETA:
    Request 50 deveria ter ido no segundo 50 → foi
    Request 51 deveria ter ido no segundo 51 → mas ficou esperando
    Requests 51-55 TODAS experimentaram ~5s de delay
    Resultado: "p99 = ~5s, p95 = ~4s" (múltiplas requests afetadas)

  FERRAMENTAS CORRETAS:
    ✅ wrk2 (corrige coordinated omission)
    ✅ Gatling (modelo correto por default)
    ❌ wrk original (não corrige)
    ❌ Muitos scripts custom (não corrigem)

  Gil Tene (Azul Systems) → Referência sobre coordinated omission
```

---

## Leis Fundamentais de Performance

### Amdahl's Law

```
AMDAHL'S LAW — Limite do Paralelismo:

  Speedup máximo com N processadores:

              1
  S(N) = ─────────────
          f + (1-f)/N

  Onde: f = fração serial (não-paralelizável)

  EXEMPLOS:

  f = 5% serial (95% paralelizável):
    N = 2   → Speedup = 1.9x
    N = 8   → Speedup = 5.9x
    N = 64  → Speedup = 14.3x
    N = ∞   → Speedup = 20x  ← MÁXIMO TEÓRICO

  f = 25% serial (75% paralelizável):
    N = 2   → Speedup = 1.6x
    N = 8   → Speedup = 2.9x
    N = 64  → Speedup = 3.7x
    N = ∞   → Speedup = 4x   ← MÁXIMO TEÓRICO

  IMPLICAÇÃO:
  → Prima otimize a parte SERIAL antes de jogar mais hardware.
  → 5% serial = max 20x speedup, não importa quantas CPUs.
  → Profile para encontrar a parte serial.

      Speedup
      ▲
  20x │              ─────────────── f = 5%
      │         ╱
  10x │       ╱
      │     ╱
   4x │   ╱──────────────────────── f = 25%
   2x │ ╱╱
   1x │╱
      ├──────────────────────────────► N (processadores)
      1    4    16    64   256  1024
```

### Little's Law

```
LITTLE'S LAW — A lei mais útil de queuing theory:

  L = λ × W

  L = número médio de requests no sistema (concurrent)
  λ = taxa de chegada (arrival rate / throughput)
  W = tempo médio no sistema (latência end-to-end)

  APLICAÇÕES PRÁTICAS:

  1. DIMENSIONAR CONNECTION POOL:
     λ = 500 req/s (throughput desejado)
     W = 20ms = 0.02s (latência média ao DB)
     L = 500 × 0.02 = 10 conexões necessárias
     → Pool de 10 conexões (+ margem = 15-20)

  2. DIMENSIONAR THREAD POOL:
     λ = 1000 req/s
     W = 100ms = 0.1s (processamento)
     L = 1000 × 0.1 = 100 threads concorrentes
     → Thread pool de ~100-150 (com margem)

  3. CALCULAR THROUGHPUT MÁXIMO:
     L = 50 (connection pool size)
     W = 10ms = 0.01s
     λ = L / W = 50 / 0.01 = 5000 req/s
     → Throughput máximo teórico: 5000 req/s

  4. ESTIMAR IMPACTO DE LATÊNCIA:
     Cenário atual: L=20, λ=1000 req/s, W=20ms
     Se W dobra para 40ms:
     L = 1000 × 0.04 = 40 conexões necessárias
     → Dobrou a demanda de connection pool!
     → Se pool fixo em 20: throughput cai para 500 req/s

  REGRA: Qualquer degradação de latência cascateia.
  Dobrar latência = dobrar ocupação de recursos.
```

### Universal Scalability Law (USL)

```
UNIVERSAL SCALABILITY LAW (Neil Gunther):

  Vai além de Amdahl: inclui COHERENCE OVERHEAD.

  C(N) = N / (1 + α(N-1) + β⋅N(N-1))

  α = contention (serialização, locks) — linear
  β = coherence (comunicação entre nós) — quadrática

  COMPORTAMENTO:

  Throughput
  ▲
  │        ╱╲
  │       ╱  ╲        ← Retrograde (β > 0)
  │      ╱    ╲          Adicionar nós PIORA performance
  │     ╱
  │    ╱──────── ← Amdahl limit (β = 0)
  │   ╱              Platô assintótico
  │  ╱
  │ ╱
  │╱  ── Linear ideal
  ├──────────────────► N (nós/servidores)

  IMPLICAÇÕES:
  → Se β > 0: existe um ponto onde adicionar nós PIORA
  → Cache invalidation, distributed locks, consensus = coherence
  → Sharding > replication para escala (reduz coherence)
  → Medir antes de escalar: talvez já passou do ponto ótimo
```

---

## Queuing Theory para Engenheiros

### O modelo M/M/1

```
M/M/1 QUEUE — Modelo mais simples:

  1 fila, 1 servidor, chegadas aleatórias (Poisson)

  ρ = λ/μ (utilização = taxa chegada / taxa serviço)

  Tempo médio no sistema (latência):
  W = 1 / (μ - λ) = (1/μ) / (1 - ρ)

  TABELA DE IMPACTO:

  │ Utilização (ρ) │ Multiplicador de latência │
  │ 10%            │ 1.1x                      │
  │ 30%            │ 1.4x                      │
  │ 50%            │ 2.0x ← latência dobra     │
  │ 70%            │ 3.3x                      │
  │ 80%            │ 5.0x                      │
  │ 90%            │ 10.0x ← 10 vezes!         │
  │ 95%            │ 20.0x                      │
  │ 99%            │ 100.0x ← explode          │

  Latência
  ▲
  │                                          │
  │                                         ╱│
  │                                        ╱ │
  │                                      ╱   │
  │                                   ╱─     │
  │                              ╱──         │
  │                        ╱───              │
  │               ╱────────                  │
  │    ╱──────────                           │
  ├──────────────────────────────────────────► ρ
  0%    20%    40%    60%    80%   100%

  LIÇÃO: A relação latência × utilização é NÃO-LINEAR.
  Nunca planeje para rodar acima de 70% de utilização sustentada.
```

### M/M/c — Múltiplos servidores

```
M/M/c QUEUE — c servidores em paralelo:

  c = número de servidores (threads, pods, instâncias)

  MAGIA DOS MÚLTIPLOS SERVIDORES:

  Cenário: λ = 80 req/s, μ = 100 req/s por servidor

  1 servidor (ρ = 80%):
    Latência média ≈ 50ms    ← 5x latência base
  
  2 servidores (ρ = 40% cada):
    Latência média ≈ 11.5ms  ← ~1.15x latência base

  4 servidores (ρ = 20% cada):
    Latência média ≈ 10.2ms  ← ~1.02x latência base

  → Diminishing returns: 1→2 servidores = enorme ganho
  → 4→8 servidores = ganho marginal se ρ já é baixo

  APLICAÇÃO: Auto-scaling thresholds
  → Scale up quando ρ > 60-70%
  → Scale down quando ρ < 30%
  → Hysteresis: thresholds diferentes para up/down
```

---

## Taxonomia de Gargalos

```
CATEGORIAS DE GARGALOS DE PERFORMANCE:

┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  1. CPU-BOUND                                                    │
│     Sintoma: CPU próximo de 100%, throughput platô               │
│     Causas típicas:                                              │
│     → Serialização/desserialização (JSON, Protobuf)             │
│     → Criptografia/hashing                                      │
│     → Regex complexo, string manipulation                        │
│     → GC pressure (linguagens managed)                           │
│     → Algoritmo O(n²) ou pior                                   │
│     Diagnóstico: CPU profiler, flame graph                      │
│                                                                  │
│  2. MEMORY-BOUND                                                 │
│     Sintoma: OOM, GC pauses longas, swap                        │
│     Causas típicas:                                              │
│     → Memory leaks (objetos não liberados)                      │
│     → Caches sem eviction policy / sem size limit               │
│     → Buffering excessivo (carregar arquivo inteiro na memória) │
│     → Object allocation rate alta → GC pressure                 │
│     Diagnóstico: heap dump, GC logs, memory profiler            │
│                                                                  │
│  3. I/O-BOUND                                                    │
│     Sintoma: CPU idle, threads esperando I/O                    │
│     Causas típicas:                                              │
│     → Database queries lentas (N+1, missing index, full scan)   │
│     → Network calls síncronos em cadeia                         │
│     → Disk I/O (logging excessivo, file operations)             │
│     → External API com latência alta                            │
│     Diagnóstico: distributed tracing, DB explain plans          │
│                                                                  │
│  4. CONTENTION-BOUND                                             │
│     Sintoma: CPU não satura, mas throughput não cresce           │
│     Causas típicas:                                              │
│     → Lock contention (synchronized, mutex)                     │
│     → Connection pool exausto                                   │
│     → Thread pool saturado                                      │
│     → Database row-level locks                                  │
│     Diagnóstico: thread dump, lock profiler, pool metrics       │
│                                                                  │
│  5. NETWORK-BOUND                                                │
│     Sintoma: Bandwidth saturado, RTT alto                       │
│     Causas típicas:                                              │
│     → Payload excessivo (API retorna dados desnecessários)      │
│     → Chatty protocols (muitas chamadas pequenas)               │
│     → DNS resolution lenta                                      │
│     → TLS handshake overhead                                    │
│     Diagnóstico: network profiler, packet capture, MTR          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Decision tree para diagnóstico

```
INÍCIO: "O sistema está lento"
  │
  ├─ CPU > 80%?
  │   ├─ SIM → CPU-Bound
  │   │   └─ CPU profiler / flame graph
  │   └─ NÃO ↓
  │
  ├─ Memory > 80% ou GC pauses > 200ms?
  │   ├─ SIM → Memory-Bound
  │   │   └─ Heap dump / GC log analysis
  │   └─ NÃO ↓
  │
  ├─ Threads bloqueadas esperando I/O?
  │   ├─ SIM → I/O-Bound
  │   │   └─ Distributed tracing / DB explain
  │   └─ NÃO ↓
  │
  ├─ Threads bloqueadas em locks/waits?
  │   ├─ SIM → Contention-Bound
  │   │   └─ Thread dump / lock profiler
  │   └─ NÃO ↓
  │
  ├─ Network utilization alta / RTT alto?
  │   ├─ SIM → Network-Bound
  │   │   └─ Network profiler / tcpdump
  │   └─ NÃO ↓
  │
  └─ Saturação em componente externo?
      └─ SIM → Gargalo externo (DB, cache, API)
          └─ Trace end-to-end + métricas do componente
```

---

## Modelos Mentais — USE e RED

### USE Method (Brendan Gregg)

```
USE METHOD — Para cada RECURSO:

  U — Utilization: % de tempo ocupado
  S — Saturation:  trabalho em fila (não atendido)
  E — Errors:      erros do recurso

APLICAÇÃO:

  │ Recurso          │ Utilization        │ Saturation          │ Errors            │
  │ CPU              │ cpu_utilization_%  │ run queue length    │ machine checks    │
  │ Memory           │ memory_used_%      │ swap usage, OOM     │ allocation fails  │
  │ Disk I/O         │ disk_utilization_% │ I/O queue length    │ disk errors       │
  │ Network          │ bandwidth_used_%   │ TCP retransmits     │ interface errors  │
  │ Connection Pool  │ active/total %     │ pending requests    │ connection errors │
  │ Thread Pool      │ active/max %       │ queue size          │ rejected tasks    │

  → Itere por TODOS os recursos. O gargalo está no primeiro 
    que mostra alta utilização OU alta saturação.
```

### RED Method (Tom Wilkie)

```
RED METHOD — Para cada SERVIÇO:

  R — Rate:     requests per second
  E — Errors:   erros per second (ou error rate %)
  D — Duration: latência (distribuição / percentis)

APLICAÇÃO:

  │ Serviço        │ Rate         │ Errors      │ Duration         │
  │ API Gateway    │ 5,000 req/s  │ 0.1%        │ p50=10ms p99=80ms│
  │ Order Service  │ 500 req/s    │ 0.5%        │ p50=30ms p99=200ms│
  │ Payment API    │ 100 req/s    │ 1.2%        │ p50=500ms p99=3s  │
  │ Database (PG)  │ 2,000 qps    │ 0.01%       │ p50=5ms p99=50ms │

USE + RED JUNTOS:
  → RED detecta QUAL serviço está com problema
  → USE detecta QUAL recurso é o gargalo
  → Combinados: "Order Service está lento (RED:Duration)
     porque connection pool está saturado (USE:Saturation)"
```

---

## Performance como Disciplina Contínua

### Shift-left performance

```
SHIFT-LEFT PERFORMANCE:

  ❌ MODELO TRADICIONAL:
  Design → Build → Test → PERF TEST → Deploy
                           ↑
                    Descobre problemas aqui
                    Custo de fix: ALTO

  ✅ SHIFT-LEFT:
  Design → Build → Test → Deploy
    ↑        ↑       ↑       ↑
    │        │       │       └── Continuous profiling + alertas
    │        │       └────────── Load test no CI/CD
    │        └────────────────── Profiling + benchmarks locais
    └─────────────────────────── Latency budgets + capacity planning

  PRÁTICAS SHIFT-LEFT:

  1. DESIGN TIME:
     → Latency budgets para cada endpoint
     → Capacity planning antes de construir
     → Review: "este design suporta 10x a carga atual?"

  2. BUILD TIME:
     → Microbenchmarks para código crítico
     → Profile durante desenvolvimento
     → Linters de performance (N+1 detection, etc)

  3. CI/CD:
     → Benchmark regression gate (falha se p99 > threshold)
     → Load test automatizado em staging
     → Comparar com baseline de performance

  4. PRODUCTION:
     → Continuous profiling (< 2% overhead)
     → Performance SLOs com alertas
     → Runbooks para degradação de performance
```

### Performance budget na cultura

```
INCORPORANDO PERFORMANCE NA CULTURA:

  1. PERFORMANCE COMO REQUISITO (não afterthought)
     → User story: "como user, quero resposta em < 200ms"
     → Definition of Done inclui performance criteria
     → PR review inclui check de performance impact

  2. PERFORMANCE REVIEWS PERIÓDICAS
     → Quinzenal: review de métricas de performance
     → Mensal: review de capacity planning
     → Trimestral: review de latency budgets

  3. OWNERSHIP CLARO
     → Cada serviço tem SLO de performance
     → Time é dono do SLO do seu serviço
     → Principal/Staff facilita cross-service performance

  4. TOOLING ACESSÍVEL
     → Qualquer dev pode rodar profiler
     → Dashboards de performance são self-service
     → Runbooks para diagnóstico são atualizados
```

---

## Performance Engineering Maturity Model

```
┌──────────────────────────────────────────────────────────────┐
│           PERFORMANCE ENGINEERING MATURITY MODEL               │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  LEVEL 0: REACTIVE                                            │
│  → Performance só é considerada quando há incidente           │
│  → Sem testes de carga                                        │
│  → Sem SLOs de performance                                    │
│  → "Está lento? Aumenta a instância."                         │
│                                                               │
│  LEVEL 1: TESTING                                             │
│  → Load tests antes do release (manual)                       │
│  → Profiling quando há problema                               │
│  → Performance é responsabilidade do "time de QA"             │
│                                                               │
│  LEVEL 2: INTEGRATED                                          │
│  → Load tests automatizados no CI/CD                          │
│  → SLOs de performance definidos                              │
│  → Capacity planning existe (atualizado ocasionalmente)       │
│  → Latency budgets para endpoints críticos                    │
│                                                               │
│  LEVEL 3: PROACTIVE                                           │
│  → Continuous profiling em produção                           │
│  → Performance regression gates no CI                         │
│  → Capacity planning atualizado continuamente                 │
│  → Load modeling baseado em dados reais                       │
│  → Performance é responsabilidade de TODOS                    │
│                                                               │
│  LEVEL 4: OPTIMIZED                                           │
│  → Performance informa decisões de arquitetura                │
│  → Benchmarks automatizados comparam alternativas             │
│  → Capacity planning é predictive (ML-based)                  │
│  → Performance budget é parte da cultura                      │
│  → Chaos engineering inclui cenários de performance           │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Anti-Patterns de Performance

| Anti-pattern | Sintoma | Consequência | Correção |
|-------------|---------|-------------|----------|
| **"CPU média"** | "CPU média é 30%, tudo bem" | Picos a 100% não detectados | Meça p95/p99 de CPU, não média |
| **"Latência média"** | "Latência média é 50ms" | p99 é 5s, users sofrem | Sempre percentis: p50, p95, p99 |
| **"Benchmark em laptop"** | "No meu Mac roda em 5ms" | Production: 200ms | Bench em ambiente similar a prod |
| **"Premature optimization"** | Otimiza sem profiling | Otimiza parte errada | Profile primeiro, otimize depois |
| **"Golden hammer"** | "Cache resolve tudo" | Cache stampede, stale data | Entenda o gargalo antes de cachear |
| **"Escala vertical infinita"** | "Coloca instância maior" | Custo exponencial, limite físico | Escala horizontal + otimização |
| **"Load test uma vez"** | Load test no Q4, nunca mais | Code muda, gargalos novos | Load test contínuo no CI/CD |
| **"Micro-otimização"** | StringBuilder em Java no lugar de + | Ganho de 0.001ms | Otimize hotspots, não cold paths |
| **"Ignorar GC"** | "Java faz GC automático" | GC pauses de 2s em produção | Tune GC, monitorar GC metrics |
| **"Sync everywhere"** | Tudo síncrono, thread-per-request | Thread pool esgota a 200 req/s | Async I/O para operações I/O-bound |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e arquitetura de performance, verifique:

1. **Latência medida por média** — Exija percentis (p50, p95, p99), média esconde tail latency
2. **Connection/thread pool sem dimensionamento** — Use Little's Law: L = λ × W para calcular
3. **Query N+1** — Detecte loops que fazem query individual; prefira batch/join
4. **Serialização ineficiente** — JSON em hot path com alto volume; considere Protobuf/Avro
5. **Lock contention** — synchronized/mutex em hot path; considere lock-free structures
6. **Cache sem eviction** — Caches devem ter TTL e size limit; sem eviction = memory leak
7. **I/O síncrono bloqueante** — Em I/O-bound paths, considere async/non-blocking
8. **Utilização planejada > 70%** — Queuing theory: latência cresce exponencialmente acima de 70%
9. **Benchmark sem warmup** — JVM/runtime precisa de warmup; primeiras iterações são outliers
10. **Log em hot path** — Logging síncrono em paths de alta vazão degrada throughput
11. **Allocation rate alta em hot path** — Objetos efêmeros causam GC pressure; considere pooling
12. **Ausência de timeout** — Toda chamada externa precisa de timeout para evitar thread starvation

---

## Referências

- **Systems Performance** — Brendan Gregg (Addison-Wesley, 2020) — Referência definitiva
- **BPF Performance Tools** — Brendan Gregg (Addison-Wesley, 2019) — Linux observability
- **The Art of Capacity Planning** — John Allspaw (O'Reilly, 2008)
- **Release It!** — Michael Nygard (Pragmatic Bookshelf, 2018) — Stability patterns
- **High Performance Browser Networking** — Ilya Grigorik (O'Reilly) — Network performance
- **Java Performance** — Scott Oaks (O'Reilly, 2020) — JVM performance
- **USE Method** — Brendan Gregg — https://www.brendangregg.com/usemethod.html
- **Little's Law** — John Little (1961) — Queuing theory fundamental
- **Amdahl's Law** — Gene Amdahl (1967) — Parallel computing limits
- **Universal Scalability Law** — Neil Gunther — http://www.perfdynamics.com/Manifesto/USLscalability.html
- **Coordinated Omission** — Gil Tene — https://www.azul.com/files/HowNotToMeasureLatency_LLSummit_NYC_12Nov2013.pdf
- **Google SRE Book** — Chapter on Distributed Systems and Latency
- **Jeff Dean** — "Latency Numbers Every Programmer Should Know"
