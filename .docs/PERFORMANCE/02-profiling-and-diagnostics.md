# Performance Engineering — Profiling e Diagnóstico

> **Objetivo deste documento:** Servir como referência completa sobre **profiling, flame graphs, diagnóstico de gargalos e continuous profiling** — técnicas para identificar WHERE o sistema gasta tempo e recursos, otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: CPU profiling, memory profiling, I/O profiling, flame graphs, GC analysis, continuous profiling em produção.

---

## Quick Reference — Cheat Sheet

| Técnica | Quando usar | Ferramenta | Overhead |
|---------|------------|-----------|----------|
| **CPU profiling** | Latência alta + CPU alta | async-profiler, perf, pprof | 1-5% |
| **Flame graph** | Visualizar CPU hotspots | FlameGraph (Brendan Gregg), speedscope | 0% (visualização) |
| **Memory profiling** | OOM, GC pauses, leaks | MAT, VisualVM, pprof, heaptrack | 5-15% |
| **GC analysis** | GC pauses > 100ms, throughput drop | GC logs, GCViewer, GCEasy | < 1% |
| **Thread dump** | Deadlocks, contention | jstack, kill -3, thread dump API | < 1% |
| **I/O profiling** | Threads idle, alto wait time | strace, perf, eBPF | 2-10% |
| **Continuous profiling** | Sempre (produção) | Pyroscope, Grafana Phlare, Datadog | < 2% |
| **Distributed tracing** | Latência cross-service | OpenTelemetry, Jaeger, Tempo | 1-3% |

---

## Sumário

- [Performance Engineering — Profiling e Diagnóstico](#performance-engineering--profiling-e-diagnóstico)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Filosofia: Profile First, Optimize Second](#filosofia-profile-first-optimize-second)
  - [CPU Profiling](#cpu-profiling)
  - [Flame Graphs](#flame-graphs)
  - [Memory Profiling](#memory-profiling)
  - [GC Analysis](#gc-analysis)
  - [Thread Dump e Contention Analysis](#thread-dump-e-contention-analysis)
  - [I/O Profiling](#io-profiling)
  - [Continuous Profiling em Produção](#continuous-profiling-em-produção)
  - [Profiling por Linguagem](#profiling-por-linguagem)
  - [Workflow de Diagnóstico](#workflow-de-diagnóstico)
  - [Anti-Patterns de Profiling](#anti-patterns-de-profiling)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Filosofia: Profile First, Optimize Second

```
"PREMATURE OPTIMIZATION IS THE ROOT OF ALL EVIL" — Donald Knuth

  Mas a frase completa é:
  "We should forget about small efficiencies, say about 97% 
   of the time: premature optimization is the root of all evil.
   Yet we should not pass up our opportunities in that 
   critical 3%."

  O PROCESSO CORRETO:

  1. MEDIR
     → Observability: métricas, traces, logs
     → "Onde está o gargalo?" (USE/RED)
     → Dado: "p99 do endpoint /orders é 2s"

  2. PROFILE
     → Profiler aponta: "80% do tempo em serializeOrder()"
     → Flame graph mostra exatamente onde
     → Dado: "serializeOrder faz 50 allocations por call"

  3. HYPOTHESIZE
     → "Se reduzirmos allocations, latência cai"
     → Estimar impacto: "80% do tempo → potencial 4-5x speedup"

  4. OPTIMIZE
     → Mudar código com base em dados
     → Object pooling para reduzir allocations

  5. VALIDATE
     → Re-profile para confirmar
     → Benchmark before/after
     → "p99 foi de 2s para 400ms ✅"

  ❌ O QUE A MAIORIA FAZ:
  "Acho que JSON parsing é lento" → reescreve em Protobuf
  → Resultado: nenhuma diferença — o gargalo era DB query

  ✅ O QUE DEVERIA FAZER:
  Profile → descobre que 90% é DB query → otimiza query
  → Resultado: p99 de 2s para 200ms
```

---

## CPU Profiling

### Sampling vs Instrumentation

```
DOIS MODOS DE CPU PROFILING:

1. SAMPLING PROFILER (recomendado para produção)
   
   Como funciona:
   → A cada N ms, interrompe a thread
   → Registra o stack trace naquele momento
   → Acumula: "função X apareceu em 45% das amostras"
   
   ✅ Overhead baixo (1-5%)
   ✅ Pode rodar em produção
   ✅ Não altera timing do programa
   ❌ Pode perder funções muito rápidas (< intervalo)
   
   Ferramentas:
   → Java: async-profiler, JFR (Java Flight Recorder)
   → Go: pprof (built-in)
   → Python: py-spy, cProfile
   → Node.js: 0x, clinic.js
   → Linux: perf, eBPF

2. INSTRUMENTATION PROFILER
   
   Como funciona:
   → Injeta código no início/fim de cada função
   → Mede tempo real de cada chamada
   → Registra call count e total time
   
   ✅ 100% preciso (nenhuma chamada perdida)
   ✅ Call count exato
   ❌ Alto overhead (10-100x slowdown)
   ❌ Altera timing (Heisenberg effect)
   ❌ Inviável em produção
   
   Ferramentas:
   → Java: JProfiler, YourKit (instrumentação mode)
   → Go: go tool trace
   → Python: cProfile (instrumentação)

RECOMENDAÇÃO:
  → Produção: SEMPRE sampling
  → Dev/debug: instrumentação quando sampling não basta
```

### async-profiler (Java/JVM)

```bash
# CPU profiling — async-profiler (JVM)
# Melhor profiler para JVM: sem safepoint bias

# Profile por 30 segundos, output flame graph
./profiler.sh -d 30 -f /tmp/flamegraph.html <PID>

# Profile com evento específico
./profiler.sh -e cpu -d 60 -f /tmp/cpu.html <PID>
./profiler.sh -e alloc -d 60 -f /tmp/alloc.html <PID>
./profiler.sh -e lock -d 60 -f /tmp/lock.html <PID>

# Wall-clock profiling (inclui tempo em I/O wait)
./profiler.sh -e wall -d 30 -f /tmp/wall.html <PID>

# QUANDO USAR CADA EVENTO:
# cpu   → threads consumindo CPU (computação)
# wall  → tempo total incluindo I/O wait (mais útil para serviços)
# alloc → onde memória é alocada (GC pressure)
# lock  → onde threads ficam bloqueadas esperando locks
```

### pprof (Go)

```go
// Go — pprof built-in
import (
    "net/http"
    _ "net/http/pprof"
)

func main() {
    // Expõe endpoints de profiling
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    // ... aplicação
}
```

```bash
# Coletar CPU profile (30s)
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Coletar heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Coletar goroutine profile (concurrency)
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Gerar flame graph (em pprof interactive)
# (pprof) web      → abre no browser
# (pprof) top      → funções mais custosas
# (pprof) list foo → código-fonte anotado da função foo
```

---

## Flame Graphs

### Anatomia de um flame graph

```
COMO LER UM FLAME GRAPH:

  ┌───────────────────────────────────────────────┐
  │            main()                              │ ← base: entry point
  ├─────────────────┬─────────────────────────────┤
  │ handleRequest() │       processQueue()        │ ← chamadas de main
  ├────────┬────────┤                             │
  │ parse()│ serialize()          │               │
  │        ├────────┤             │               │
  │        │toJSON()│             │               │
  │        │ 35%    │             │               │
  └────────┴────────┴─────────────┴───────────────┘

  REGRAS DE LEITURA:

  1. EIXO X = proporção de tempo (NÃO é timeline)
     → Largura = % de CPU samples
     → Mais largo = mais tempo gasto
     → toJSON() ocupa 35% das amostras = hotspot

  2. EIXO Y = profundidade do stack
     → Base (bottom) = entry point
     → Topo = função folha (onde CPU gasta tempo)
     → Preste atenção nos TOPS (platôs largos)

  3. COR = geralmente aleatória (para distinguir frames)
     → Alguns tools colorem por: pacote, módulo, tipo

  4. O QUE PROCURAR:
     → Platôs largos no topo = CPU hotspots
     → Torres estreitas e altas = call chains profundas
     → Funções que aparecem em múltiplas torres = 
        chamadas de múltiplos call sites

  5. NÃO É TIMELINE:
     → Ordem horizontal é ALPHABETICAL (não temporal)
     → Duas colunas lado a lado ≠ uma depois da outra
```

### Tipos de flame graph

```
TIPOS DE FLAME GRAPH:

1. CPU FLAME GRAPH (ON-CPU)
   → Mostra onde THREADS GASTAM CPU
   → Eixo X: % de CPU samples
   → Útil para: CPU-bound, computação pesada
   → Gerado por: perf, async-profiler -e cpu

2. OFF-CPU FLAME GRAPH
   → Mostra onde THREADS FICAM ESPERANDO (blocked/sleep)
   → Eixo X: % de tempo OFF-CPU
   → Útil para: I/O waits, lock contention, sleep
   → Gerado por: eBPF, async-profiler -e wall

3. DIFFERENTIAL FLAME GRAPH
   → Compara DOIS profiles (before/after)
   → Vermelho: função ficou MAIS lenta
   → Azul: função ficou MAIS rápida
   → Útil para: validar otimizações, detectar regressões
   → Gerado por: difffolded.pl + flamegraph.pl

4. MEMORY FLAME GRAPH (ALLOCATION)
   → Mostra onde MEMÓRIA É ALOCADA
   → Eixo X: bytes alocados (ou # de allocations)
   → Útil para: GC pressure, memory leaks, object churn
   → Gerado por: async-profiler -e alloc, pprof heap

RECOMENDAÇÃO:
  → Para serviços de backend: comece com WALL-CLOCK 
    (inclui on-cpu + off-cpu)
  → Depois refine com on-cpu ou off-cpu específico
```

### Gerando flame graphs

```bash
# Linux perf → flame graph (qualquer linguagem)
perf record -F 99 -p <PID> -g -- sleep 30
perf script | stackcollapse-perf.pl | flamegraph.pl > perf.svg

# async-profiler → flame graph (JVM)
./profiler.sh -d 30 -f /tmp/flame.html <PID>

# Go pprof → flame graph
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/profile

# py-spy → flame graph (Python)
py-spy record -o profile.svg --pid <PID>

# Node.js → flame graph
npx 0x app.js       # gera flame graph automaticamente
npx clinic flame -- node app.js

# Speedscope (visualizador universal)
# Upload qualquer profile em: https://www.speedscope.app/
```

---

## Memory Profiling

### Heap analysis

```
MEMORY PROFILING — O QUE PROCURAR:

1. HEAP SIZE GROWTH (possível leak)
   
   Heap (MB)
   ▲
   │    ╱╲  ╱╲    ╱╲  ╱╲    ╱╲  ╱╲   ← Normal: saw-tooth (GC)
   │   ╱  ╲╱  ╲  ╱  ╲╱  ╲  ╱  ╲╱  ╲
   │  ╱         ╲╱         ╲╱
   ├─────────────────────────────────► Tempo
   
   Heap (MB)
   ▲                                ╱
   │                           ╱╲ ╱   ← LEAK: baseline sobe
   │                      ╱╲ ╱  ╲╱
   │                 ╱╲ ╱  ╲╱
   │            ╱╲ ╱  ╲╱
   │       ╱╲ ╱  ╲╱
   │  ╱╲ ╱  ╲╱
   ├─────────────────────────────────► Tempo

2. ALLOCATION HOTSPOTS
   → Funções que alocam mais objetos
   → Objeto grande alocado frequentemente
   → Allocation de objetos no hot path

3. RETAINED SIZE vs SHALLOW SIZE
   → Shallow: tamanho do objeto só
   → Retained: objeto + tudo que só ele referencia
   → Leak: objetos com retained size grande que não são GCed

4. DOMINATOR TREE
   → "Quem está impedindo estes objetos de serem GCed?"
   → Objeto X domina Y se remover X permite GC de Y
   → Procure: caches, listeners, static collections
```

### Heap dump analysis workflow

```
WORKFLOW DE HEAP DUMP (Java/JVM):

  1. CAPTURAR HEAP DUMP
     # Via jmap
     jmap -dump:live,format=b,file=heap.hprof <PID>
     
     # Automaticamente em OOM
     -XX:+HeapDumpOnOutOfMemoryError
     -XX:HeapDumpPath=/var/log/heap-dump.hprof
     
     # Via JFR (lightweight, ongoing)
     jcmd <PID> JFR.start duration=60s filename=recording.jfr

  2. ANALISAR COM ECLIPSE MAT
     → Open heap dump (.hprof)
     → Leak Suspect Report (automático)
     → Dominator Tree: quem retém mais memória?
     → Histogram: quantas instâncias de cada classe?
     → OQL: queries sobre o heap
     
     Exemplo OQL:
     SELECT * FROM java.util.HashMap WHERE size > 10000

  3. INVESTIGAR PATTERNS COMUNS DE LEAK
     → Cache sem eviction (HashMap crescendo sem limite)
     → Listeners não removidos (event handlers acumulando)
     → ThreadLocal não cleared
     → ClassLoader leaks (em containers/app servers)
     → Static collections (List/Map em campo static)

  4. COMPARAR DOIS DUMPS
     → Dump 1: logo após startup
     → Dump 2: após horas de operação
     → Diff: "que objetos cresceram?"
```

---

## GC Analysis

### GC fundamentals para performance

```
GARBAGE COLLECTION — IMPACTO EM PERFORMANCE:

  GC PAUSES AFETAM:
  → Latência p99/p99.9 (tail latency = GC pause)
  → Throughput (CPU usada em GC ≠ CPU para requests)
  → Jitter (latência inconsistente)

  GC GENERATIONS (JVM):

  ┌──────────────── Heap ──────────────────────────┐
  │                                                 │
  │  ┌─── Young Gen ───┐  ┌──── Old Gen ────────┐  │
  │  │ Eden │ S0 │ S1  │  │                      │  │
  │  │      │    │     │  │                      │  │
  │  └──────┴────┴─────┘  └──────────────────────┘  │
  │                                                 │
  │  Minor GC: Eden → S0/S1           (rápido, ms)  │
  │  Major/Full GC: Old Gen           (lento, s)    │
  │  Mixed GC (G1): Young + parte Old (balanceado)  │
  │                                                 │
  └─────────────────────────────────────────────────┘

  MÉTRICAS DE GC PARA MONITORAR:

  │ Métrica              │ Target                │ Alerta se        │
  │ GC pause time        │ < 50ms (p99)          │ > 200ms          │
  │ GC frequency         │ Minor: ~10/min        │ > 60/min         │
  │ GC throughput         │ > 95% (tempo em app)  │ < 90%            │
  │ Allocation rate      │ Estável                │ Crescendo        │
  │ Promotion rate       │ Estável, baixa         │ Alta = premature │
  │ Long-lived objects   │ Estável                │ Crescendo = leak │
```

### GC tuning guidelines

```
GC COLLECTORS E QUANDO USAR (JVM):

  │ Collector            │ Latency Target │ Quando usar              │
  │ G1 GC (default)      │ < 200ms p99    │ Default para maioria     │
  │ ZGC                  │ < 10ms p99     │ Ultra-low latency        │
  │ Shenandoah           │ < 10ms p99     │ Low latency (OpenJDK)    │
  │ Parallel GC          │ Throughput     │ Batch processing         │
  │ Serial GC            │ N/A            │ Containers < 2 cores     │

  TUNING RULES:
  
  1. REGRA DE OURO: NÃO tune sem dados
     → Habilite GC logs SEMPRE em produção
     → -Xlog:gc*:file=gc.log:time,uptime,level,tags
     → Analise logs com GCEasy.io ou GCViewer

  2. HEAP SIZING
     → -Xms = -Xmx (evite resize dinâmico)
     → Heap = 3-4x live set (objetos retained após Full GC)
     → Container: heap ≤ 75% do memory limit

  3. PARA LOW LATENCY (serviços web):
     → G1 ou ZGC
     → -XX:MaxGCPauseMillis=100 (G1, target)
     → ZGC: -XX:+UseZGC (Java 21+: generational ZGC)

  4. EVITE ALLOCATION EM HOT PATH:
     → Object pooling para objetos frequentes
     → Primitive types em vez de boxed
     → StringBuilder em loops (mas profile antes!)
     → Evite lambdas que capturam variáveis (closures)

  5. MONITORE CONTINUAMENTE:
     → GC pause time trend
     → Allocation rate trend
     → Se piorou após deploy → performance regression
```

---

## Thread Dump e Contention Analysis

### Thread dump analysis

```
THREAD DUMP — QUANDO E COMO:

  QUANDO:
  → Throughput não cresce com mais load
  → CPU baixa mas latência alta
  → Deadlock suspeito (requests travadas)

  COMO CAPTURAR:
  
  # JVM
  jstack <PID> > thread-dump.txt
  kill -3 <PID>   # imprime no stdout
  jcmd <PID> Thread.print
  
  # Capturar 3 dumps com 10s de intervalo (trend)
  for i in 1 2 3; do jstack <PID> > dump-$i.txt; sleep 10; done

  # Go
  curl http://localhost:6060/debug/pprof/goroutine?debug=2

  O QUE PROCURAR:

  1. THREAD STATES
     RUNNABLE  → executando (CPU)
     BLOCKED   → esperando lock
     WAITING   → esperando notify/signal
     TIMED_WAITING → esperando com timeout (sleep, poll)

  2. PATTERN: MUITAS THREADS BLOCKED NO MESMO LOCK
     "Thread-42" BLOCKED on java.util.HashMap
       waiting to lock <0x00000007c0045678>
       owned by "Thread-1"
     
     → Lock contention! Thread-1 segura lock, 42+ esperam

  3. PATTERN: DEADLOCK
     "Thread-A" blocked, waiting for lock held by Thread-B
     "Thread-B" blocked, waiting for lock held by Thread-A
     → JVM detecta e reporta: "Found one Java-level deadlock"

  4. PATTERN: THREAD POOL EXAUSTO
     200 threads em TIMED_WAITING (poll de queue vazia)
     → Pool grande demais (desperdício de memória)
     OU
     200 threads em BLOCKED (todas esperando resource)
     → Pool certo, mas downstream está lento
```

---

## I/O Profiling

### Diagnóstico de I/O

```
I/O-BOUND DIAGNOSIS:

  SINAIS:
  → CPU idle (< 30%) mas latência alta
  → Threads em WAITING/TIMED_WAITING
  → Network ou Disk I/O utils alta

  FERRAMENTAS:

  1. DISTRIBUTED TRACING (primeiro!)
     → Mostra ONDE tempo é gasto entre serviços
     → "85% da latência é call ao PostgreSQL"
     → OpenTelemetry → Jaeger/Tempo

  2. DATABASE:
     → EXPLAIN ANALYZE para queries lentas
     → pg_stat_statements (PostgreSQL) — top queries por tempo
     → Slow query log (MySQL, PostgreSQL)
     → Connection pool metrics (active, idle, wait time)

  3. OS-LEVEL:
     → iostat — disk I/O metrics
     → iotop — I/O por processo
     → ss / netstat — socket stats
     → tcpdump / Wireshark — packet analysis

  4. eBPF/BPF (Linux avançado):
     → biolatency — distribuição de latência de block I/O
     → tcplife — duração de conexões TCP
     → ext4slower — filesystem operations lentas
     → Ferramentas: bcc, bpftrace

  I/O DIAGNOSIS CHECKLIST:
  □ Distributed trace mostra qual call é lento?
  □ É query (EXPLAIN → missing index? full scan?)
  □ É connection (pool exausto? DNS resolving?)
  □ É network (RTT? bandwidth? packet loss?)
  □ É disk (IOPS? throughput? queue depth?)
```

---

## Continuous Profiling em Produção

### O que é continuous profiling

```
CONTINUOUS PROFILING:

  Tradicional: profile quando há problema (reactive)
  Continuous: profile SEMPRE, overhead < 2% (proactive)

  ┌──────────────────────────────────────────────────────┐
  │                                                       │
  │  PRODUÇÃO                                             │
  │  ┌─────┐ ┌─────┐ ┌─────┐                            │
  │  │Pod 1│ │Pod 2│ │Pod 3│ ... N pods                  │
  │  │ CP  │ │ CP  │ │ CP  │     (continuous profiler)   │
  │  └──┬──┘ └──┬──┘ └──┬──┘                             │
  │     │       │       │                                 │
  │     └───────┴───────┘                                 │
  │             │                                         │
  │     ┌───────▼───────┐                                 │
  │     │  Profile Store │  (Pyroscope, Phlare, etc)      │
  │     │  + Aggregation │                                │
  │     └───────┬───────┘                                 │
  │             │                                         │
  │     ┌───────▼───────┐                                 │
  │     │  UI / Query    │  ← flame graphs on-demand      │
  │     │  Diff profiles │  ← compare before/after deploy │
  │     └───────────────┘                                 │
  │                                                       │
  └──────────────────────────────────────────────────────┘

  POR QUE CONTINUOUS:
  → Problema de performance nem sempre é reproduzível
  → "Estava lento ontem às 3h da manhã" → sem profile :(
  → Continuous: profile de QUALQUER momento no passado
  → Compare: "flame graph de hoje vs semana passada"
  → Diff: "que função ficou mais lenta após o deploy?"
```

### Ferramentas de continuous profiling

```
FERRAMENTAS:

  │ Ferramenta        │ Linguagens          │ Tipo           │
  │ Pyroscope         │ Java, Go, Python,   │ Open-source    │
  │                   │ Node, Ruby, .NET    │ + Grafana Cloud│
  │ Grafana Phlare    │ Qualquer (pprof)    │ Open-source    │
  │ Datadog CP        │ Java, Go, Python,   │ SaaS           │
  │                   │ Ruby, .NET, Node    │                │
  │ Amazon CodeGuru   │ Java, Python        │ AWS managed    │
  │ Google Cloud Prof │ Java, Go, Python,   │ GCP managed    │
  │                   │ Node               │                │
  │ async-profiler    │ JVM                 │ OS (agent)     │
  │ + custom push     │                     │                │

  SETUP MÍNIMO (Pyroscope + Java):

  # Agent como JVM arg
  java -javaagent:pyroscope.jar \
    -Dpyroscope.server.address=http://pyroscope:4040 \
    -Dpyroscope.application.name=order-service \
    -jar app.jar

  # Go SDK
  pyroscope.Start(pyroscope.Config{
      ApplicationName: "order-service",
      ServerAddress:   "http://pyroscope:4040",
      ProfileTypes: []pyroscope.ProfileType{
          pyroscope.ProfileCPU,
          pyroscope.ProfileAllocObjects,
          pyroscope.ProfileAllocSpace,
          pyroscope.ProfileInuseObjects,
          pyroscope.ProfileInuseSpace,
      },
  })
```

### Tag-based profiling

```
TAG-BASED PROFILING — FILTRAGEM POR CONTEXTO:

  Profile normal: "onde o service gasta CPU?"
  Tag-based: "onde gasta CPU PARA REQUESTS DO /checkout?"

  // Java (Pyroscope)
  Pyroscope.LabelsWrapper.run(
      new LabelsSet("endpoint", "/checkout"),
      () -> {
          // código do handler
          processCheckout(request);
      }
  );

  RESULTADO:
  → Flame graph filtrado por tag endpoint=/checkout
  → "O /checkout gasta 60% do CPU em serialize()"
  → Mas o /catalog gasta 60% em queryDB()
  → Sem tag: tudo misturado, impossível distinguir

  TAGS ÚTEIS:
  → endpoint, http_method
  → tenant_id (multi-tenant)
  → region
  → version (A/B deploy comparison)
```

---

## Profiling por Linguagem

### Java/JVM

```
JAVA PROFILING TOOLKIT:

  1. async-profiler (recomendado para produção)
     → Sem safepoint bias
     → CPU, alloc, lock, wall-clock
     → Flame graph direto

  2. Java Flight Recorder (JFR)
     → Built-in desde JDK 11
     → Low overhead (< 1%)
     → Eventos: CPU, GC, I/O, locks, exceptions
     → Análise com JDK Mission Control (JMC)
     
     # Iniciar recording
     jcmd <PID> JFR.start duration=60s filename=rec.jfr
     
     # Ou via JVM arg
     -XX:StartFlightRecording=duration=60s,filename=rec.jfr

  3. JMH (Java Microbenchmark Harness)
     → Para micro-benchmarks (não profiling)
     → Warmup, GC control, blackhole
     → Ver seção de Benchmarking

  4. VisualVM
     → GUI: CPU profiling, heap, threads
     → Bom para desenvolvimento
     → Overhead alto — não usar em prod
```

### Go

```
GO PROFILING TOOLKIT:

  1. pprof (built-in, recomendado)
     → import _ "net/http/pprof"
     → CPU, heap, goroutine, mutex, block
     → go tool pprof para análise
     
     # CPU profile
     go tool pprof http://localhost:6060/debug/pprof/profile
     
     # Heap profile (inuse_space)
     go tool pprof -inuse_space http://localhost:6060/debug/pprof/heap
     
     # Mutex contention
     runtime.SetMutexProfileFraction(5)
     go tool pprof http://localhost:6060/debug/pprof/mutex

  2. trace (execution tracer)
     → Mostra scheduling de goroutines ao longo do tempo
     → Útil para: contention, scheduling delays, STW GC
     
     curl http://localhost:6060/debug/pprof/trace?seconds=5 > trace.out
     go tool trace trace.out

  3. GOGC tuning
     → GOGC=100 (default): GC quando heap = 2x live set
     → GOGC=200: menos GC, mais memória
     → GOMEMLIMIT (Go 1.19+): hard memory limit
```

### Python

```
PYTHON PROFILING TOOLKIT:

  1. py-spy (recomendado para produção)
     → Sampling profiler, zero overhead
     → Não requer modificação do código
     → Flame graph direto
     
     py-spy record -o profile.svg --pid <PID>
     py-spy top --pid <PID>   # realtime top-like

  2. cProfile (built-in)
     → Instrumentation profiler
     → Alto overhead — dev only
     
     python -m cProfile -o output.prof app.py
     # Analisar com snakeviz
     snakeviz output.prof

  3. memory_profiler
     → Memória por linha
     
     from memory_profiler import profile
     @profile
     def my_function():
         ...

  4. Scalene
     → CPU + memory + GPU profiling
     → Per-line granularity
     → Distingue Python vs C time
```

---

## Workflow de Diagnóstico

```
WORKFLOW SISTEMÁTICO DE DIAGNÓSTICO:

  ┌───────────────────────────────────────────────────────┐
  │ STEP 1: SINTOMA                                        │
  │ "Endpoint /orders p99 = 2s (SLO: 200ms)"              │
  └───────────────┬───────────────────────────────────────┘
                  │
  ┌───────────────▼───────────────────────────────────────┐
  │ STEP 2: METRICS (RED)                                  │
  │ Rate: 500 req/s, Errors: 0.1%, Duration: p99=2s       │
  │ Degradou quando? Correlação com deploy? Carga?         │
  └───────────────┬───────────────────────────────────────┘
                  │
  ┌───────────────▼───────────────────────────────────────┐
  │ STEP 3: RESOURCE (USE)                                 │
  │ CPU: 45% │ Memory: 60% │ Disk: 10% │ Network: 20%    │
  │ Nenhum recurso saturado → não é CPU/memory/disk/net   │
  └───────────────┬───────────────────────────────────────┘
                  │
  ┌───────────────▼───────────────────────────────────────┐
  │ STEP 4: DISTRIBUTED TRACING                            │
  │ /orders → order-svc (10ms) → payment-svc (1.8s!) → DB│
  │ Gargalo: payment-svc                                   │
  └───────────────┬───────────────────────────────────────┘
                  │
  ┌───────────────▼───────────────────────────────────────┐
  │ STEP 5: PROFILE (payment-svc)                          │
  │ Wall-clock flame graph:                                │
  │   60% em DB query (findPaymentsByOrderId)              │
  │   30% em JSON serialization                            │
  │   10% em business logic                                │
  └───────────────┬───────────────────────────────────────┘
                  │
  ┌───────────────▼───────────────────────────────────────┐
  │ STEP 6: ROOT CAUSE                                     │
  │ EXPLAIN: full table scan (missing index on order_id)   │
  │ Fix: CREATE INDEX idx_payments_order_id ON payments... │
  └───────────────┬───────────────────────────────────────┘
                  │
  ┌───────────────▼───────────────────────────────────────┐
  │ STEP 7: VALIDATE                                       │
  │ Deploy fix → p99 = 180ms ✅ (era 2s)                  │
  │ Re-profile: DB query agora é 5ms (era 1.8s)           │
  └───────────────────────────────────────────────────────┘
```

---

## Anti-Patterns de Profiling

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **Profile só em dev** | Prod tem carga/dados diferentes | Continuous profiling em prod (< 2% overhead) |
| **Profile sem carga** | Sistema idle não mostra gargalos | Profile durante load test ou em prod |
| **Otimizar sem profile** | Otimiza parte errada | Sempre profile → identify hotspot → otimize |
| **Ignorar tail latency** | "Média é boa" = p99 pode ser 10s | Profile p99 requests (tag-based profiling) |
| **Heap dump em pico** | Full GC + IO stall = downtime | Configure HeapDumpOnOutOfMemoryError; dump off-peak |
| **Um thread dump só** | Pode pegar estado transitório | 3 dumps com 10s intervalo, comparar padrão |
| **Profile sem baseline** | "Isso é normal ou não?" | Sempre ter baseline para comparar |
| **Instrumentação em prod** | 10-100x overhead | Sampling profiler, nunca instrumentation em prod |
| **Ignorar GC logs** | GC pauses = tail latency oculta | GC logs SEMPRE habilitados (-Xlog:gc*) |
| **Flame graph sem context** | "A função X é 30% do CPU" — e daí? | Correlacionar com request rate e latência |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e configurações de profiling/diagnóstico, verifique:

1. **GC logs desabilitados em produção** — `-Xlog:gc*` (JVM) ou equivalente DEVE estar ativo sempre
2. **Heap dump não configurado para OOM** — `-XX:+HeapDumpOnOutOfMemoryError` é obrigatório
3. **Profiling endpoint exposto publicamente** — `/debug/pprof` (Go) deve ser localhost-only ou autenticado
4. **Connection pool sem métricas** — Monitore active, idle, wait time, timeouts do pool
5. **Query sem EXPLAIN em PR de schema change** — ALTER TABLE deve vir com EXPLAIN de queries afetadas
6. **Timeout ausente em chamadas externas** — Sem timeout = thread starvation potencial
7. **Logging síncrono em hot path** — Logger async ou reduzir verbosity em paths de alta vazão
8. **Cache sem métricas de hit ratio** — Cache sem observabilidade = cache inútil
9. **Thread pool sem métricas** — Monitore: active, queue size, rejected count
10. **Otimização sem benchmark before/after** — Toda otimização requer evidência mensurável
11. **JFR/Flight Recorder não habilitado** — Low overhead, alto valor para diagnóstico retroativo
12. **Continuous profiler ausente em serviços críticos** — Pyroscope/Phlare com < 2% overhead, diagnóstico retroativo

---

## Referências

- **Systems Performance** — Brendan Gregg (Addison-Wesley, 2020)
- **BPF Performance Tools** — Brendan Gregg (Addison-Wesley, 2019)
- **Flame Graphs** — Brendan Gregg — https://www.brendangregg.com/flamegraphs.html
- **async-profiler** — https://github.com/async-profiler/async-profiler
- **Pyroscope** — https://pyroscope.io/
- **Java Flight Recorder** — Oracle — JDK Mission Control
- **Go pprof** — https://pkg.go.dev/runtime/pprof
- **py-spy** — https://github.com/benfred/py-spy
- **Eclipse MAT** — https://eclipse.dev/mat/
- **GCEasy** — https://gceasy.io/
- **speedscope** — https://www.speedscope.app/
- **Gil Tene** — "How NOT to Measure Latency" — coordinated omission
- **Java Performance** — Scott Oaks (O'Reilly, 2020)
- **100 Go Mistakes** — Teiva Harsanyi — Profiling chapters
