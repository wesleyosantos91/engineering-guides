# Level 0 — Performance Foundations

> **Objetivo:** Dominar os fundamentos de Performance Engineering — métricas, percentis, modelos mentais (USE/RED), leis fundamentais (Amdahl, Little, queuing theory), taxonomia de gargalos e cultura de performance como disciplina contínua.

---

## Objetivo de Aprendizado

- Entender a diferença entre Performance Testing e Performance Engineering
- Dominar as 4 métricas fundamentais: latência, throughput, utilização e saturação
- Medir e interpretar percentis (p50, p95, p99, p99.9) — nunca confiar em média
- Aplicar Amdahl's Law para estimar limites de paralelismo
- Usar Little's Law para dimensionar pools e conexões
- Entender queuing theory básica e por que utilização > 70% é perigosa
- Aplicar modelos USE e RED para diagnóstico sistemático
- Identificar e classificar gargalos (CPU-bound, I/O-bound, memory-bound, contention)
- Internalizar "Profile First, Optimize Second"

---

## Escopo Funcional

### Serviço Alvo: Digital Wallet API

Usar a API já implementada nos challenges de backend:

```
── Endpoints para medição ──
POST /api/v1/users                           → CRUD (baseline simples)
POST /api/v1/wallets/{walletId}/deposit      → Transacional
POST /api/v1/wallets/{walletId}/transfer     → Atômico + concorrência
GET  /api/v1/wallets/{walletId}/transactions → Paginado, I/O-bound
GET  /metrics                                → Prometheus metrics
GET  /health/ready                           → Readiness
```

### Exercícios Práticos

1. **Medir as 4 métricas** de cada endpoint sob carga leve (10 VUs) e moderada (100 VUs)
2. **Montar dashboard USE/RED** no Grafana com os Golden Signals
3. **Calcular Little's Law** para dimensionar connection pool do banco de dados
4. **Demonstrar Amdahl's Law** com exercício prático de paralelização
5. **Provar que média mente** — gerar workload com distribuição bimodal e comparar média vs percentis

---

## Escopo Técnico

### Métricas Fundamentais — O que medir

| Métrica | O que é | Como medir | Perigo |
|---------|---------|------------|--------|
| **Latência** | Tempo para completar operação (end-to-end) | Percentis: p50, p95, p99 | Média esconde tail latency |
| **Throughput** | Operações por segundo (req/s) | Goodput = req/s bem-sucedidas | req/s com erros é vaidade |
| **Utilização** | Fração do tempo que recurso está ocupado | CPU%, mem%, disk I/O% | > 70% = filas explodem |
| **Saturação** | Trabalho na fila (não atendido) | Queue depth, pool wait count | Leading indicator de degradação |

### Modelos USE e RED

```
USE (por RECURSO — CPU, Memory, Disk, Network):
  U — Utilization: % do tempo que o recurso está ocupado
  S — Saturation: quantidade de trabalho na fila
  E — Errors: erros do recurso

RED (por SERVIÇO — cada endpoint/service):
  R — Rate: requests por segundo
  E — Errors: taxa de erros
  D — Duration: latência (distribuição / percentis)

  USE + RED JUNTOS:
  → RED detecta QUAL serviço está com problema
  → USE detecta QUAL recurso é o gargalo
```

### Leis Fundamentais

```
AMDAHL'S LAW — Limite do Paralelismo:

              1
  S(N) = ─────────────
          f + (1-f)/N

  f = fração serial (não-paralelizável)
  Se f = 5% → speedup máximo = 20x (mesmo com ∞ CPUs)

LITTLE'S LAW — Dimensionamento de Pools:

  L = λ × W

  L = itens em sistema (concurrent connections)
  λ = arrival rate (req/s)
  W = tempo no sistema (latência)

  Exemplo: 500 req/s × 0.020s (20ms) = 10 conexões necessárias
  Com headroom: 10 × 1.5 = 15 conexões no pool

QUEUING THEORY — Por que 100% utilização é impossível:

  Latência ∝ 1/(1 - ρ)     (ρ = utilização)

  ρ = 50%  → latência = 2x o tempo de processamento
  ρ = 70%  → latência = 3.3x
  ρ = 90%  → latência = 10x
  ρ = 99%  → latência = 100x
  
  → NUNCA planeje para utilização > 70%
```

### Taxonomia de Gargalos

| Tipo | Sintomas | Diagnóstico | Exemplo |
|------|----------|-------------|---------|
| **CPU-bound** | CPU alta, latência linear com carga | Profiler: hotspot em computação | Serialização JSON em hot path |
| **I/O-bound** | CPU baixa, threads blocked/waiting | Thread dump: WAITING em I/O | Query lenta, API externa |
| **Memory-bound** | GC frequent, allocation rate alta | GC logs, memory profiler | Objetos efêmeros em loop |
| **Contention** | CPU moderada, throughput não cresce | Lock profiler, thread dump | Lock contention em shared state |
| **Network-bound** | Latência alta inter-serviço | Network metrics, traceroute | Cross-region calls |

---

## Critérios de Aceite

### Todos os Stacks

- [ ] Dashboard Grafana com métricas RED para cada endpoint (Rate, Errors, Duration)
- [ ] Dashboard Grafana com métricas USE para infraestrutura (CPU, Memory, Disk, Network)
- [ ] Teste prático: medir p50, p95, p99 de `/deposit` com 10, 50, 100 VUs
- [ ] Demonstração de que média mente: gerar cenário bimodal e mostrar diferença média vs p99
- [ ] Cálculo de Little's Law aplicado: dimensionar connection pool baseado em throughput e latência medidos
- [ ] Exercício Amdahl's Law: implementar versão serial e paralela, medir speedup, comparar com teórico
- [ ] Documento com tabela de gargalos identificados (tipo, sintoma, diagnóstico)
- [ ] Métricas Prometheus expostas: `http_requests_total`, `http_request_duration_seconds`, `db_connections_active`

### Go

- [ ] Métricas via `prometheus/client_golang` com histograma de latência
- [ ] Exercício de paralelismo: goroutines + channels, medir speedup vs serial
- [ ] `runtime.NumGoroutine()` exposto como métrica gauge
- [ ] Dashboard mostrando goroutine count, heap alloc, GC pause

### Java (Spring Boot / Quarkus / Micronaut / Jakarta EE)

- [ ] Métricas via Micrometer (Spring/Quarkus/Micronaut) ou MicroProfile Metrics (Jakarta)
- [ ] Exercício de paralelismo: Virtual Threads vs Platform Threads, medir speedup
- [ ] Métricas de JVM expostas: `jvm_memory_used_bytes`, `jvm_gc_pause_seconds`, `jvm_threads_states`
- [ ] Dashboard mostrando heap usage, GC frequency, thread count

---

## Definição de Pronto (DoD)

- [ ] Docker Compose com: app + database + Prometheus + Grafana
- [ ] Dashboards importáveis (JSON provisioning)
- [ ] Script de carga básico (k6 ou wrk2) para gerar tráfego nos endpoints
- [ ] Documento `PERFORMANCE_FOUNDATIONS.md` com:
  - Tabela de métricas medidas (p50/p95/p99) por endpoint e carga
  - Cálculo de Little's Law para o sistema
  - Análise USE/RED dos resultados
  - Classificação dos gargalos encontrados
- [ ] Commit: `feat(perf-level-0): add performance foundations metrics and dashboards`

---

## Checklist

### Setup Inicial

- [ ] Docker Compose configurado com Prometheus + Grafana
- [ ] `prometheus.yml` com scrape config para a aplicação
- [ ] Grafana com datasource Prometheus configurado
- [ ] Aplicação expondo endpoint `/metrics`

### Métricas RED (por endpoint)

- [ ] `http_requests_total{method, path, status}` — Counter
- [ ] `http_request_duration_seconds{method, path}` — Histogram (buckets: 5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s)
- [ ] `http_request_errors_total{method, path, error_type}` — Counter

### Métricas USE (infraestrutura)

- [ ] CPU utilization (via node_exporter ou métricas de runtime)
- [ ] Memory utilization (heap, non-heap / Go memstats)
- [ ] DB connection pool: active, idle, max, wait count
- [ ] Goroutine count (Go) / Thread count (Java)

### Dashboards Grafana

- [ ] **RED Dashboard** — 1 row por endpoint com Rate, Error Rate, Duration (p50/p95/p99)
- [ ] **USE Dashboard** — CPU, Memory, DB Pool, Threads/Goroutines
- [ ] **Overview Dashboard** — Apdex score, total throughput, error budget

### Exercícios Práticos

- [ ] **Little's Law** — Medir throughput (λ) e latência (W) → calcular pool size ideal (L)
- [ ] **Amdahl's Law** — Implementar processamento serial → paralelo → medir speedup real vs teórico
- [ ] **Média vs Percentis** — Gerar workload bimodal → provar que média esconde problemas
- [ ] **Saturação** — Aumentar carga gradualmente → identificar ponto de saturação no gráfico

---

## Tarefas Sugeridas por Stack

### Go (Gin)

```go
// 1. Middleware de métricas Prometheus
func PrometheusMiddleware() gin.HandlerFunc {
    httpRequestsTotal := promauto.NewCounterVec(
        prometheus.CounterOpts{Name: "http_requests_total"},
        []string{"method", "path", "status"},
    )
    httpRequestDuration := promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5},
        },
        []string{"method", "path"},
    )
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        duration := time.Since(start).Seconds()
        status := strconv.Itoa(c.Writer.Status())
        httpRequestsTotal.WithLabelValues(c.Request.Method, c.FullPath(), status).Inc()
        httpRequestDuration.WithLabelValues(c.Request.Method, c.FullPath()).Observe(duration)
    }
}
```

```go
// 2. Exercício Amdahl's Law — serial vs parallel
func ProcessOrdersSerial(orders []Order) []Result {
    results := make([]Result, len(orders))
    for i, o := range orders {
        results[i] = processOrder(o) // simula 10ms de CPU work
    }
    return results
}

func ProcessOrdersParallel(orders []Order, workers int) []Result {
    results := make([]Result, len(orders))
    sem := make(chan struct{}, workers)
    var wg sync.WaitGroup
    for i, o := range orders {
        wg.Add(1)
        sem <- struct{}{}
        go func(idx int, order Order) {
            defer wg.Done()
            defer func() { <-sem }()
            results[idx] = processOrder(order)
        }(i, o)
    }
    wg.Wait()
    return results
}
// Medir: speedup real para workers=1,2,4,8,16
// Comparar com Amdahl teórico (estimar fração serial)
```

```go
// 3. Little's Law — calcular e validar pool size
func CalculatePoolSize(targetThroughput float64, avgLatencyMs float64) int {
    L := targetThroughput * (avgLatencyMs / 1000.0)
    withHeadroom := L * 1.5
    return int(math.Ceil(withHeadroom))
}
// Medir: throughput real vs pool sizes diferentes
// Validar: pool = Little's Law result → throughput atinge target?
```

### Spring Boot

```java
// 1. Métricas automáticas via Micrometer (já incluídas com Actuator)
// application.yml
// management.endpoints.web.exposure.include: health, metrics, prometheus
// management.metrics.tags.application: wallet-service

// 2. Exercício Amdahl's Law — Virtual Threads vs Platform Threads
@Service
public class AmdahlDemoService {
    
    public Duration processSerial(List<Order> orders) {
        var start = Instant.now();
        orders.forEach(this::processOrder); // 10ms CPU cada
        return Duration.between(start, Instant.now());
    }
    
    public Duration processParallel(List<Order> orders, int threads) {
        var start = Instant.now();
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            var futures = orders.stream()
                .map(o -> executor.submit(() -> processOrder(o)))
                .toList();
            futures.forEach(f -> { try { f.get(); } catch (Exception e) { throw new RuntimeException(e); } });
        }
        return Duration.between(start, Instant.now());
    }
    
    private Result processOrder(Order order) {
        // Simula CPU work de 10ms
        Blackhole.consumeCPU(tokens);
        return new Result(order.id(), "processed");
    }
}

// 3. Custom métricas de negócio
@Component
public class WalletMetrics {
    private final Counter depositsTotal;
    private final Counter transfersTotal;
    private final DistributionSummary transferAmount;
    
    public WalletMetrics(MeterRegistry registry) {
        this.depositsTotal = Counter.builder("wallet.deposits.total").register(registry);
        this.transfersTotal = Counter.builder("wallet.transfers.total").register(registry);
        this.transferAmount = DistributionSummary.builder("wallet.transfer.amount")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
}
```

### Docker Compose (todos os stacks)

```yaml
# docker-compose.perf.yml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: wallet
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"

  prometheus:
    image: prom/prometheus:v2.53.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:11.0.0
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

### Script de carga básico (k6)

```javascript
// load-basic.js — script mínimo para gerar tráfego
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 10 },   // ramp to 10 VUs
    { duration: '5m', target: 10 },   // hold
    { duration: '2m', target: 50 },   // ramp to 50
    { duration: '5m', target: 50 },   // hold
    { duration: '2m', target: 100 },  // ramp to 100
    { duration: '5m', target: 100 },  // hold
    { duration: '2m', target: 0 },    // ramp down
  ],
};

const BASE = __ENV.BASE_URL || 'http://localhost:8080';

export default function () {
  const res = http.get(`${BASE}/api/v1/wallets/1/transactions?page=0&size=10`);
  check(res, { 'status 200': (r) => r.status === 200 });
  sleep(Math.random() * 2 + 1);
}
```

---

## Extensões Opcionais

- [ ] Implementar Apdex score e expor como métrica Prometheus
- [ ] Criar alerta Grafana para p99 > 500ms
- [ ] Implementar coordinated omission demo: comparar resultados wrk vs wrk2
- [ ] Calcular custo por request (cloud cost / total requests)
- [ ] Implementar Universal Scalability Law (USL) e comparar com Amdahl
- [ ] Criar flame chart de latência breakdown por componente (network + processing + DB + serialization)

---

## Erros Comuns

| Erro | Consequência | Correção |
|------|-------------|----------|
| Medir latência com média | Esconde tail latency, problemas invisíveis | Sempre p50, p95, p99 |
| Usar `time.Now()` manual para métricas | Impreciso, não padronizado | Use biblioteca de métricas (Micrometer/Prometheus client) |
| Pool de conexões = número de CPUs | Subdimensionado para I/O-bound | Little's Law: L = λ × W |
| Planejar para 100% utilização | Latência explode (queuing theory) | Target ≤ 70% sustained |
| Benchmark em laptop | Resultados não reproduzíveis | Ambiente controlado (Docker, CI) |
| Otimizar sem profiling | Otimiza a parte errada do sistema | Profile primeiro, dados > intuição |
| RED sem USE (ou vice-versa) | Diagnóstico incompleto | USE + RED juntos: serviço + recurso |

---

## Como Isso Aparece em Entrevistas

- "Qual a diferença entre latência p50, p95 e p99? Quando cada um importa?"
- "Como você dimensionaria um connection pool para 1000 req/s com latência média de 50ms?"
- "O que é Amdahl's Law e como ela limita o ganho de adicionar mais CPUs?"
- "Explique a diferença entre saturação e utilização"
- "Como você identificaria se um sistema é CPU-bound ou I/O-bound?"
- "O que são os modelos USE e RED? Quando usar cada um?"
- "Por que utilização de 90% é problemática? Explique com queuing theory"
- "O que é coordinated omission e como isso afeta load testing?"
