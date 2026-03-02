# Level 1 — Profiling & Diagnostics

> **Objetivo:** Dominar técnicas de profiling (CPU, memory, I/O), flame graphs, GC analysis, thread/goroutine dumps e continuous profiling em produção — identificando WHERE o sistema gasta tempo e recursos.

---

## Objetivo de Aprendizado

- Entender a filosofia "Profile First, Optimize Second"
- Diferenciar sampling profiler vs instrumentation profiler
- Gerar e interpretar flame graphs (CPU, allocation, wall-clock)
- Identificar hotspots de CPU com async-profiler (Java) e pprof (Go)
- Diagnosticar memory leaks e allocation pressure
- Analisar GC logs e tunar garbage collector (G1, ZGC, Go GC)
- Capturar e interpretar thread dumps (Java) e goroutine dumps (Go)
- Configurar continuous profiling em produção com Pyroscope (< 2% overhead)
- Aplicar workflow de diagnóstico: Observe → Profile → Hypothesize → Optimize → Validate

---

## Escopo Funcional

### Cenários de Diagnóstico (deliberadamente com problemas)

Criar ou injetar os seguintes gargalos na Digital Wallet API para praticar profiling:

```
── Cenário 1: CPU Hotspot ──
  Endpoint: POST /api/v1/wallets/{id}/deposit
  Problema injetado: serialização JSON ineficiente em loop
  Diagnóstico: CPU profiler → flame graph → hotspot em serialize()
  Fix: pool de ObjectMapper / reutilizar encoder

── Cenário 2: Memory Leak ──
  Endpoint: GET /api/v1/wallets/{id}/transactions
  Problema injetado: cache unbounded (sem eviction)
  Diagnóstico: heap dump → growing map → leak identificado
  Fix: LRU cache com TTL e max size

── Cenário 3: N+1 Query ──
  Endpoint: GET /api/v1/users (com wallets embedded)
  Problema injetado: carrega wallets em loop (1 query por user)
  Diagnóstico: I/O profiler → tempo em DB wait → SQL log mostra N+1
  Fix: JOIN FETCH ou batch loading

── Cenário 4: Lock Contention ──
  Endpoint: POST /api/v1/wallets/{id}/transfer
  Problema injetado: synchronized/mutex global em processamento
  Diagnóstico: lock profiler → contention em transferLock
  Fix: lock por wallet ID (fine-grained locking)

── Cenário 5: GC Pressure ──
  Endpoint: POST /api/v1/orders (bulk)
  Problema injetado: criação de objetos temporários em loop
  Diagnóstico: GC logs → frequent young GC → allocation profiler
  Fix: object pooling, reduzir allocations
```

---

## Escopo Técnico

### CPU Profiling

| Aspecto | Go (pprof) | Java (async-profiler) |
|---------|-----------|----------------------|
| **Modo** | Sampling (built-in) | Sampling (sem safepoint bias) |
| **Ativar** | `import _ "net/http/pprof"` | `-javaagent:./profiler.jar` |
| **Coletar** | `go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30` | `./profiler.sh -d 30 -f /tmp/flame.html <PID>` |
| **Eventos** | CPU | `cpu`, `wall`, `alloc`, `lock` |
| **Output** | pprof binary (web, top, list) | HTML flame graph, JFR |
| **Overhead** | 1-5% | 1-5% |

### Memory Profiling

| Aspecto | Go | Java |
|---------|------|------|
| **Heap snapshot** | `pprof /debug/pprof/heap` | `jmap -dump:format=b,file=heap.hprof <PID>` |
| **Allocation rate** | `pprof /debug/pprof/allocs` | async-profiler `-e alloc` |
| **Análise** | `go tool pprof` → `top`, `web` | Eclipse MAT, VisualVM |
| **Leak detection** | Compare dois heap profiles | MAT: Leak Suspects report |

### GC Analysis

```
JAVA — GC Logs:
  JVM flags:
    -Xlog:gc*:file=gc.log:time,level,tags
    -XX:+UseG1GC (ou -XX:+UseZGC)
  
  Análise: upload gc.log para GCEasy.io ou GCViewer
  
  MÉTRICAS CHAVE:
  │ Métrica           │ Target          │ Ação se violado           │
  │ GC pause (p99)    │ < 100ms (G1)    │ Tune heap, regions       │
  │ GC pause (p99)    │ < 10ms (ZGC)    │ Verificar allocation rate│
  │ GC throughput     │ > 95%           │ Mais heap ou tune GC     │
  │ Allocation rate   │ < 1GB/s         │ Reduzir objetos efêmeros │
  │ Promotion rate    │ < 100MB/s       │ Objetos vivem demais     │

GO — GOGC e GC metrics:
  GOGC=100 (default: GC quando heap dobra)
  GOMEMLIMIT=512MiB (Go 1.19+: soft memory limit)
  
  runtime.ReadMemStats(&m) → HeapAlloc, NumGC, PauseTotalNs
  debug.SetGCPercent(200) → menos GC, mais memória
```

### Thread/Goroutine Dumps

```
JAVA — Thread Dump:
  jstack <PID>
  kill -3 <PID>          (Unix)
  jcmd <PID> Thread.print
  
  ESTADOS A PROCURAR:
  RUNNABLE       → threads trabalhando (OK)
  BLOCKED        → esperando lock (contention!)
  WAITING        → esperando notify (pode ser OK)
  TIMED_WAITING  → sleep/wait com timeout

GO — Goroutine Dump:
  curl localhost:6060/debug/pprof/goroutine?debug=2
  
  SINAIS DE PROBLEMA:
  → Muitas goroutines em "select" sem activity → possível goroutine leak
  → Goroutines em "semacquire" → mutex contention
  → Goroutine count crescendo continuamente → leak
```

### Flame Graphs — Como ler

```
ANATOMIA DE UM FLAME GRAPH:

  ┌─────────────────────────────────────────────────────────┐
  │                     main()                               │
  │  ┌─────────────────────────────────┐ ┌─────────────────┐│
  │  │        handleRequest()          │ │   gcWorker()     ││
  │  │  ┌──────────────┐ ┌──────────┐  │ │                  ││
  │  │  │ queryDB()    │ │serialize()│  │ │                  ││
  │  │  │  ┌─────────┐ │ │ ┌──────┐ │  │ │                  ││
  │  │  │  │ pgQuery │ │ │ │encode│ │  │ │                  ││
  │  └──└──└─────────┘─┘ └─└──────┘─┘──┘ └──────────────────┘│
  └─────────────────────────────────────────────────────────┘

  COMO LER:
  → LARGURA = tempo de CPU (ou amostras). Mais largo = mais tempo
  → ALTURA = profundidade do stack. Não importa para análise
  → NÃO é timeline. Eixo X é ALFABÉTICO (não temporal)
  → FOCO: funções LARGAS no TOPO = hotspots

  TIPOS:
  → CPU flame graph: onde CPU gasta tempo
  → Allocation flame graph: onde memória é alocada
  → Wall-clock flame graph: tempo total (inclui I/O wait)
  → Off-CPU flame graph: onde threads ficam bloqueadas
```

---

## Critérios de Aceite

### Todos os Stacks

- [ ] Gerar flame graph de CPU para endpoint `/deposit` sob carga (100 VUs)
- [ ] Gerar flame graph de allocation para endpoint `/transactions` (paginado)
- [ ] Identificar hotspot de CPU em pelo menos 1 cenário → fix → re-profile → confirmar melhoria
- [ ] Identificar memory leak no cenário de cache unbounded → fix → validar com heap profile
- [ ] Diagnosticar N+1 query → fix com JOIN FETCH/batch → re-medir latência
- [ ] Capturar thread/goroutine dump durante carga → analisar estados
- [ ] Configurar Pyroscope para continuous profiling (Docker Compose)
- [ ] Documento `PROFILING_REPORT.md` com flame graphs e análise before/after

### Go

- [ ] pprof endpoint exposto (`localhost:6060/debug/pprof/`)
- [ ] CPU profile de 30s coletado e analisado (`top`, `web`, `list`)
- [ ] Heap profile com diff entre dois snapshots (antes/depois de carga)
- [ ] Goroutine profile identificando goroutine leak no cenário de cache
- [ ] Benchmark com `-cpuprofile` para hot path

### Java

- [ ] async-profiler configurado no container Docker
- [ ] Flame graph gerado para eventos: `cpu`, `wall`, `alloc`, `lock`
- [ ] GC logs habilitados com análise em GCEasy ou GCViewer
- [ ] Thread dump capturado com `jstack` e analisado (BLOCKED threads)
- [ ] JFR (Java Flight Recorder) com gravação de 5 minutos
- [ ] Comparação G1 vs ZGC: GC pause p99 e throughput sob carga

---

## Definição de Pronto (DoD)

- [ ] 5 cenários de gargalo implementados (injetar → diagnosticar → fix)
- [ ] Flame graphs gerados para cada cenário (antes e depois do fix)
- [ ] Pyroscope rodando em Docker Compose com profiling contínuo
- [ ] GC analysis completa (Java): logs + relatório + tuning recommendation
- [ ] Relatório `PROFILING_REPORT.md` com:
  - Flame graph de cada cenário (screenshot ou link)
  - Análise textual: "80% do CPU está em X"
  - Before/after: latência antes e depois do fix
  - Recomendações de tuning
- [ ] Commit: `feat(perf-level-1): add profiling diagnostics and flame graph analysis`

---

## Checklist

### Setup de Profiling

- [ ] Docker Compose atualizado com Pyroscope
- [ ] pprof endpoint configurado (Go)
- [ ] async-profiler disponível no container Java
- [ ] GC logs habilitados com flags JVM
- [ ] Script para capturar thread dump / goroutine dump

### Cenário 1: CPU Hotspot

- [ ] Injetar problema: serialização ineficiente em loop
- [ ] Profile: gerar flame graph CPU
- [ ] Identificar: hotspot no flame graph (função mais larga no topo)
- [ ] Fix: pool de encoder / evitar serialização repetida
- [ ] Validar: re-profile → hotspot removido + medir p99 before/after

### Cenário 2: Memory Leak

- [ ] Injetar problema: cache sem eviction (Map<> crescente)
- [ ] Profile: heap snapshot T0 e T1 (após 5 min de carga)
- [ ] Identificar: diff de heap mostra Map crescendo
- [ ] Fix: LRU cache com TTL (Caffeine em Java, lru em Go)
- [ ] Validar: heap estável após fix

### Cenário 3: N+1 Query

- [ ] Injetar problema: loop de queries em listagem
- [ ] Profile: wall-clock flame graph → tempo em DB driver
- [ ] Identificar: SQL logs mostram N+1 queries
- [ ] Fix: JOIN FETCH (Java) / preload (Go GORM)
- [ ] Validar: contagem de queries reduzida, latência melhor

### Cenário 4: Lock Contention

- [ ] Injetar problema: mutex/synchronized global em transfer
- [ ] Profile: lock profiler (async-profiler `-e lock` / goroutine dump)
- [ ] Identificar: threads BLOCKED em transferLock
- [ ] Fix: lock por wallet ID (ConcurrentHashMap / sync.Map)
- [ ] Validar: throughput de transfer aumenta com VUs concorrentes

### Cenário 5: GC Pressure

- [ ] Injetar problema: criação de objetos temporários em bulk processing
- [ ] Profile: allocation flame graph + GC logs
- [ ] Identificar: high allocation rate + frequent young GC
- [ ] Fix: object pooling / reduzir allocations por request
- [ ] Validar: allocation rate reduzido + GC pauses menores

---

## Tarefas Sugeridas por Stack

### Go (Gin)

```go
// 1. Setup pprof
import (
    "net/http"
    _ "net/http/pprof"
)

func main() {
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    // ... start Gin server
}
```

```bash
# 2. Coletar profiles
# CPU (30 segundos)
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap (snapshot)
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutines
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Allocs (total allocations desde o início)
go tool pprof http://localhost:6060/debug/pprof/allocs

# No pprof interactive:
# (pprof) top 20           → top 20 funções por CPU
# (pprof) web              → abre flame graph no browser
# (pprof) list funcName    → código-fonte anotado
# (pprof) png > profile.png → exportar imagem
```

```go
// 3. Injetar CPU hotspot (exemplo)
func (s *TransactionService) Deposit(ctx context.Context, req DepositRequest) error {
    // PROBLEMA: serializa JSON em loop para "auditoria"
    for i := 0; i < 100; i++ {
        json.Marshal(req)  // ← hotspot: 100 serializations por deposit!
    }
    // ... lógica real
}
// Profile mostrará json.Marshal dominando o flame graph
// FIX: serializar uma vez, reutilizar resultado
```

### Spring Boot / Quarkus / Micronaut

```dockerfile
# Dockerfile com async-profiler
FROM eclipse-temurin:25-jdk AS builder
# ... build

FROM eclipse-temurin:25-jre
# Instalar async-profiler
RUN apt-get update && apt-get install -y wget && \
    wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz && \
    tar xzf async-profiler-3.0-linux-x64.tar.gz -C /opt/ && \
    rm async-profiler-3.0-linux-x64.tar.gz

# JVM flags para profiling e GC logs
ENV JAVA_OPTS="-XX:+UseG1GC \
    -Xlog:gc*:file=/tmp/gc.log:time,level,tags \
    -XX:+UnlockDiagnosticVMOptions \
    -XX:+DebugNonSafepoints"
    
COPY --from=builder /app/target/*.jar /app/app.jar
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

```bash
# Profiling com async-profiler dentro do container
docker exec -it wallet-app /opt/async-profiler-3.0/bin/asprof -d 30 -f /tmp/cpu.html 1
docker cp wallet-app:/tmp/cpu.html ./reports/

# Eventos disponíveis:
# asprof -e cpu    → CPU profiling
# asprof -e wall   → Wall-clock (inclui I/O wait)
# asprof -e alloc  → Allocation profiling
# asprof -e lock   → Lock contention profiling

# Heap dump
docker exec wallet-app jcmd 1 GC.heap_dump /tmp/heap.hprof
docker cp wallet-app:/tmp/heap.hprof ./reports/

# Thread dump
docker exec wallet-app jstack 1

# JFR recording (5 minutos)
docker exec wallet-app jcmd 1 JFR.start duration=5m filename=/tmp/recording.jfr
docker cp wallet-app:/tmp/recording.jfr ./reports/
```

```java
// Injetar N+1 Query (exemplo para fix)
// PROBLEMA:
@Query("SELECT u FROM User u")
List<User> findAll();  // depois faz user.getWallets() em loop → N+1

// FIX:
@Query("SELECT u FROM User u LEFT JOIN FETCH u.wallets")
List<User> findAllWithWallets();  // single query com JOIN
```

### Docker Compose — Pyroscope

```yaml
# Adicionar ao docker-compose.perf.yml
services:
  pyroscope:
    image: grafana/pyroscope:1.7.0
    ports:
      - "4040:4040"
    
  app-go:
    environment:
      - PYROSCOPE_SERVER_ADDRESS=http://pyroscope:4040
    # SDK Go: import github.com/grafana/pyroscope-go

  app-java:
    environment:
      - PYROSCOPE_SERVER_ADDRESS=http://pyroscope:4040
      - JAVA_OPTS=-javaagent:/opt/pyroscope.jar
    volumes:
      - ./pyroscope.jar:/opt/pyroscope.jar
```

---

## Extensões Opcionais

- [ ] Implementar off-CPU flame graph (threads bloqueadas em I/O)
- [ ] Usar eBPF para profiling de sistema (ferramentas do Brendan Gregg)
- [ ] Comparar safepoint-biased profiler vs async-profiler (mostrar diferença)
- [ ] Implementar custom JFR event para métricas de negócio
- [ ] Diff de flame graphs entre duas versões (Pyroscope diff view)
- [ ] Profiling de native image Quarkus/Micronaut vs JVM mode

---

## Erros Comuns

| Erro | Consequência | Correção |
|------|-------------|----------|
| Otimizar sem profiling | Otimiza a parte errada (3% do tempo) | Profile primeiro: dados > intuição |
| Usar instrumentation profiler em produção | 10-100x slowdown, Heisenberg effect | Sampling profiler: override < 5% |
| Ignorar wall-clock profiling | Perde tempo em I/O wait (DB, HTTP) | Use `wall` event além de `cpu` |
| Heap dump em horário de pico | Full GC + pausa da aplicação | Heap dump em low traffic ou replica |
| GC logs desabilitados em produção | Sem dados para diagnosticar pauses | Sempre habilitar, overhead < 1% |
| Goroutine dump sem baseline | Não sabe se 1000 goroutines é normal | Medir goroutine count continuamente |
| Profiling por tempo insuficiente | Amostra não representativa | Mínimo 30s CPU, 5min para steady state |

---

## Como Isso Aparece em Entrevistas

- "Como você diagnosticaria um endpoint que está com p99 de 5 segundos?"
- "Explique como funciona um sampling profiler vs instrumentation profiler"
- "O que é um flame graph e como você lê?"
- "Como detectar um memory leak em Java/Go?"
- "Qual a diferença entre profiling de CPU e wall-clock?"
- "Como funciona o garbage collector G1 do Java? Quando usaria ZGC?"
- "O que é continuous profiling e qual o overhead típico?"
- "Descreva seu workflow de diagnóstico de performance"
