# Performance Engineering — Benchmarking e Load Testing

> **Objetivo deste documento:** Servir como referência completa sobre **benchmarking, microbenchmarks, load testing, stress testing e análise estatística de resultados** — como medir performance de forma confiável e reproduzível, otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: metodologia de benchmarking, micro vs macrobenchmarks, load testing tools, tipos de load test, rigor estatístico, CI/CD integration, regression detection.

---

## Quick Reference — Cheat Sheet

| Tipo de teste | O que mede | Duração | Quando |
|--------------|-----------|---------|--------|
| **Microbenchmark** | Função/método isolado | Segundos-minutos | Otimização de hot path |
| **Load test** | Sistema sob carga normal | 10-60 min | Pré-release, mensal |
| **Stress test** | Sistema além do limite | 30-60 min | Trimestral, pré-evento |
| **Soak test** | Estabilidade prolongada | 4-24 horas | Pré-release, mensal |
| **Spike test** | Resposta a burst súbito | 15-30 min | Pré-evento |
| **Breakpoint test** | Ponto de ruptura | Variável | Capacity planning |
| **Baseline benchmark** | Performance de referência | 30-60 min | A cada release |

---

## Sumário

- [Performance Engineering — Benchmarking e Load Testing](#performance-engineering--benchmarking-e-load-testing)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Filosofia de Benchmarking](#filosofia-de-benchmarking)
  - [Microbenchmarks](#microbenchmarks)
  - [Load Testing](#load-testing)
  - [Tipos de Load Test](#tipos-de-load-test)
  - [Ferramentas de Load Testing](#ferramentas-de-load-testing)
  - [Rigor Estatístico](#rigor-estatístico)
  - [Ambiente de Teste](#ambiente-de-teste)
  - [Análise de Resultados](#análise-de-resultados)
  - [Regression Detection em CI/CD](#regression-detection-em-cicd)
  - [Anti-Patterns de Benchmarking](#anti-patterns-de-benchmarking)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Filosofia de Benchmarking

```
BENCHMARKING = MEDIR PERFORMANCE DE FORMA 
               CONFIÁVEL, REPRODUZÍVEL E SIGNIFICATIVA

  REGRAS FUNDAMENTAIS:

  1. BENCHMARK É CIÊNCIA
     → Hipótese → Experimento → Medição → Conclusão
     → Sem hipótese: "estamos medindo por quê?"
     → Sem controle: "o que mudou entre A e B?"

  2. MEDIR O QUE IMPORTA
     → Latência percebida pelo USUÁRIO (end-to-end)
     → Throughput sob carga REALISTA
     → Comportamento sob STRESS (degradation graceful?)

  3. REPRODUZÍVEL
     → Mesmas condições → mesmo resultado (±margem)
     → Se não é reproduzível, não é benchmark
     → Documenta: hardware, OS, versão, config, data

  4. COMPARÁVEL
     → Sempre compare com BASELINE
     → Before/after de otimização
     → Release N vs Release N+1
     → "Mais rápido" sem baseline = clickbait

  5. RELEVANTE
     → Benchmark sintético ≠ performance real
     → Micro-benchmark de JSON parsing ≠ performance do sistema
     → O mais próximo de produção, melhor
```

---

## Microbenchmarks

### Quando usar microbenchmarks

```
MICROBENCHMARK: medir uma FUNÇÃO/OPERAÇÃO ISOLADA

  QUANDO USAR:
  → Comparar algoritmos/estruturas de dados
  → Validar otimização de hot path (confirmado por profiler)
  → Escolher entre bibliotecas (JSON parsers, etc.)
  → Reproduzir paper/claim: "library X é 5x mais rápida"

  QUANDO NÃO USAR:
  → Prever performance do sistema inteiro
  → "A média de 10ns dessa função melhorou 15%" 
    (se não é hot path, irrelevante)
  → Comparar linguagens ("Go vs Java no Fibonacci")
    (não mede nada real)

  PERIGOS:
  → Dead Code Elimination (compilador remove código sem efeito)
  → Constant Folding (compilador resolve em compile-time)
  → Loop Optimizations (JIT otimiza loops do benchmark)
  → Cache Effects (dados cabem em L1 no benchmark, não em prod)
  → Branch Prediction (predição perfeita em loop, não em prod)
```

### JMH (Java Microbenchmark Harness)

```java
// JMH — O PADRÃO OURO para microbenchmarks JVM

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(3)  // 3 JVM forks (evita JIT bias)
@State(Scope.Benchmark)
public class SerializationBenchmark {

    private Order order;

    @Setup
    public void setup() {
        order = createRealisticOrder();  // dados realistas!
    }

    @Benchmark
    public byte[] jacksonSerialize() {
        return objectMapper.writeValueAsBytes(order);
    }

    @Benchmark
    public byte[] gsonSerialize() {
        return gson.toJson(order).getBytes();
    }

    @Benchmark
    public byte[] protobufSerialize() {
        return order.toProto().toByteArray();
    }
}

// RESULTADO:
// Benchmark                        Mode  Cnt    Score   Error  Units
// jacksonSerialize                 avgt   30   850.3 ± 12.4   ns/op
// gsonSerialize                    avgt   30  1240.5 ± 18.7   ns/op
// protobufSerialize                avgt   30   340.2 ±  5.1   ns/op
//
// INTERPRETAÇÃO:
// Protobuf é ~2.5x mais rápido que Jackson para este payload
// MAS: importa? Se serialização é 0.1% do request time, NÃO.
```

### Go benchmarks

```go
// Go — testing.B built-in

func BenchmarkJSONMarshal(b *testing.B) {
    order := createRealisticOrder()
    b.ResetTimer()  // não conta setup no tempo
    
    for i := 0; i < b.N; i++ {
        json.Marshal(order)
    }
}

func BenchmarkJSONMarshal_Parallel(b *testing.B) {
    order := createRealisticOrder()
    b.ResetTimer()
    
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            json.Marshal(order)
        }
    })
}

// Executar:
// go test -bench=BenchmarkJSON -benchmem -count=5 ./...
//
// BenchmarkJSONMarshal-8     500000   2341 ns/op   1024 B/op   12 allocs/op
// BenchmarkJSONMarshal_Parallel-8  2000000  621 ns/op  1024 B/op  12 allocs/op
//
// FLAGS IMPORTANTES:
// -benchmem          → mostra allocations (B/op, allocs/op)
// -count=5           → repete 5x (para calcular variance)
// -benchtime=5s      → roda por 5s ao invés de 1s default
// -cpuprofile=cpu.out → gera CPU profile do benchmark
```

### Benchstat (análise estatística)

```bash
# Go — benchstat para comparar resultados

# Benchmark versão A
go test -bench=. -count=10 ./... > old.txt

# Aplica otimização, benchmark versão B
go test -bench=. -count=10 ./... > new.txt

# Comparar com análise estatística
benchstat old.txt new.txt

# name              old time/op  new time/op  delta
# JSONMarshal-8     2.34µs ± 2%  1.87µs ± 1%  -20.09%  (p=0.000 n=10+10)
# JSONUnmarshal-8   5.67µs ± 3%  5.71µs ± 2%     ~     (p=0.089 n=10+10)
#
# INTERPRETAÇÃO:
# Marshal: 20% mais rápido, p=0.000 (estatisticamente significativo ✅)
# Unmarshal: ~ (não significativo), p=0.089 > 0.05 (não mudou ❌)
```

---

## Load Testing

### Princípios de load testing

```
LOAD TESTING = SIMULAR CARGA REALISTA NO SISTEMA

  OBJETIVO:
  → Validar que o sistema atende SLOs sob carga esperada
  → Encontrar limites antes que produção encontre
  → Detectar regressões de performance
  → Input para capacity planning

  PRINCÍPIOS:

  1. WORKLOAD REALISTA
     → Mix de operações proporcional ao real
     → "70% reads, 20% writes, 10% search"
     → Think time entre requests (usuários reais pausam)
     → Dados realistas (tamanhos variados, volume real)

  2. RAMP-UP GRADUAL
     → Não começar com 100% da carga
     → Ramp 0 → target em 5-10 minutos
     → Permite warmup (JIT, caches, connection pools)
     → Permite observar degradação progressiva

  3. DURAÇÃO SUFICIENTE
     → Mínimo: 10 minutos em steady state
     → Recomendado: 30-60 minutos
     → Soak: 4-24 horas (memory leaks, GC degradation)

  4. MEDIR DO LADO DO CLIENTE
     → Latência: registrar timestamp de envio e resposta
     → Não confiar em server-side metrics para latência
     → Coordinated Omission: usar ferramentas que compensam
       (wrk2, Gatling, k6 — NÃO wrk, ab)

  5. MONITORAMENTO DURANTE TESTE
     → Dashboards: CPU, memory, GC, connections, errors
     → Distributed tracing: sample durante o teste
     → GC logs: pausas durante o teste
     → Network: retransmits, drops
```

---

## Tipos de Load Test

```
TIPOS DE LOAD TEST:

  1. LOAD TEST (carga normal)
     
     VUs
     ▲
     │    ┌──────────────────────┐
     │   ╱│                      │╲
     │  ╱ │   Steady State       │ ╲
     │ ╱  │   (30-60 min)        │  ╲
     │╱   │                      │   ╲
     ├────┼──────────────────────┼────► Tempo
      Ramp-up                   Ramp-down
     
     Propósito: Validar SLOs sob carga ESPERADA
     Users: Peak traffic projetado
     Duração: 30-60 min steady state
     Success: Todos SLOs met (latência, error rate)

  2. STRESS TEST (além do limite)
     
     VUs
     ▲           ╱
     │         ╱╱
     │       ╱╱    ← Degradação graceful?
     │     ╱╱        Ou colapso?
     │   ╱╱
     │ ╱╱
     │╱
     ├────────────────────────────► Tempo
     
     Propósito: Encontrar breaking point e degradation behavior
     Users: Progressivo até ruptura (2x, 3x, 5x peak)
     Duração: Incrementos de 5-10 min cada step
     Success: Degradação graceful, recovery após redução

  3. SOAK TEST (stability test)
     
     VUs
     ▲
     │ ┌────────────────────────────────────────┐
     │ │                                         │
     │ │        4-24 horas steady state           │
     │ │                                         │
     │ └────────────────────────────────────────┘
     ├──────────────────────────────────────────► Tempo
     
     Propósito: Detectar problemas de long-run
     → Memory leaks (heap cresce sem parar)
     → Connection leaks (pool se esgota)
     → GC degradation (pauses crescem)
     → File descriptor leaks
     → Thread leaks
     Duration: 4-24 horas
     Success: Métricas estáveis, sem degradação

  4. SPIKE TEST
     
     VUs
     ▲     ╱╲
     │    ╱  ╲
     │   ╱    ╲     ╱╲
     │──╱      ╲   ╱  ╲──
     │          ╲ ╱
     │           ╲╱
     ├────────────────────────────► Tempo
     
     Propósito: Resposta a burst súbito de tráfego
     Ramp: 0 → 10x em segundos
     Success: Recovery rápido, sem cascade failure
     → Auto-scaling respondeu adequadamente?
     → Circuit breakers ativaram?
     → Queue não saturou?

  5. BREAKPOINT TEST
     
     VUs              Error rate
     ▲                ▲
     │     ╱╱         │        ╱╱
     │   ╱╱           │      ╱╱
     │ ╱╱             │    ╱╱
     │╱               │  ╱╱
     ├──────► Tempo   ├──────► VUs
     
     Propósito: Encontrar capacidade máxima exata
     Users: Incrementar 10% a cada 5min até falha
     Métrica: throughput satura, latência explode, errors > threshold
     Output: "O sistema suporta máx X req/s com Y pods"
```

---

## Ferramentas de Load Testing

### Comparação de ferramentas

```
FERRAMENTAS DE LOAD TESTING:

  │ Ferramenta │ Linguagem    │ Scripting   │ Coordinated │ Distributed │
  │            │ do script    │ Complexity  │ Omission    │ Support     │
  │────────────┼──────────────┼─────────────┼─────────────┼─────────────│
  │ k6         │ JavaScript   │ Baixa       │ ✅ Sim      │ ✅ (k6 Cloud)│
  │ Gatling    │ Scala/Java   │ Média       │ ✅ Sim      │ ✅ Enterprise│
  │ Locust     │ Python       │ Baixa       │ ✅ Sim      │ ✅ Built-in  │
  │ wrk2       │ Lua          │ Baixa       │ ✅ Sim      │ ❌ Single    │
  │ JMeter     │ XML/GUI      │ Alta        │ ❌ Não      │ ✅ Built-in  │
  │ Artillery  │ YAML/JS      │ Baixa       │ ✅ Sim      │ ✅ Cloud     │
  │ Vegeta     │ CLI/Go       │ Baixa       │ ✅ Sim      │ ❌ Single    │
  │ wrk        │ Lua          │ Baixa       │ ❌ NÃO      │ ❌ Single    │
  │ ab         │ CLI          │ N/A         │ ❌ NÃO      │ ❌ Single    │

  ⚠️  NÃO USE wrk ou ab para medir latência!
  → wrk e ab sofrem de Coordinated Omission
  → Latências reportadas são MENORES que a realidade
  → Use wrk2, k6, Gatling, ou Locust

  RECOMENDAÇÕES:
  → Times iniciantes: k6 (JS, simples, boa docs)
  → Times Java: Gatling (DSL poderosa, relatórios belos)
  → Times Python: Locust (familiar, extensível)
  → Quick test: wrk2 (CLI, one-liner)
  → Enterprise: Gatling Enterprise ou k6 Cloud
```

### k6 — exemplo completo

```javascript
// k6 — Exemplo de load test completo

import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const orderLatency = new Trend('order_latency', true);

// Cenários (workload model)
export const options = {
  scenarios: {
    // Cenário 1: carga constante
    steady_load: {
      executor: 'constant-arrival-rate',
      rate: 500,               // 500 req/s
      timeUnit: '1s',
      duration: '30m',
      preAllocatedVUs: 100,
      maxVUs: 500,
    },
    // Cenário 2: ramp-up para stress
    stress: {
      executor: 'ramping-arrival-rate',
      startRate: 100,
      timeUnit: '1s',
      stages: [
        { duration: '5m',  target: 500 },   // ramp to 500/s
        { duration: '10m', target: 500 },   // hold
        { duration: '5m',  target: 1000 },  // ramp to 1000/s
        { duration: '10m', target: 1000 },  // hold
        { duration: '5m',  target: 2000 },  // ramp to 2000/s
        { duration: '10m', target: 2000 },  // hold
        { duration: '5m',  target: 0 },     // ramp down
      ],
      preAllocatedVUs: 200,
      maxVUs: 2000,
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],  // SLO
    errors: ['rate<0.01'],                            // < 1% errors
    order_latency: ['p(99)<1000'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

export default function () {
  // Simular workload realista: 70% read, 30% write
  const rand = Math.random();
  
  if (rand < 0.5) {
    // GET catalog (50% das requests)
    const res = http.get(`${BASE_URL}/api/catalog/products`, {
      tags: { name: 'GET /catalog' },
    });
    check(res, { 'catalog 200': (r) => r.status === 200 });
    errorRate.add(res.status !== 200);
    
  } else if (rand < 0.8) {
    // GET product detail (30%)
    const productId = Math.floor(Math.random() * 10000) + 1;
    const res = http.get(`${BASE_URL}/api/catalog/products/${productId}`, {
      tags: { name: 'GET /product/:id' },
    });
    check(res, { 'product 200': (r) => r.status === 200 });
    errorRate.add(res.status !== 200);
    
  } else {
    // POST order (20%)
    const payload = JSON.stringify({
      productId: Math.floor(Math.random() * 10000) + 1,
      quantity: Math.floor(Math.random() * 5) + 1,
    });
    const params = { headers: { 'Content-Type': 'application/json' } };
    const start = Date.now();
    const res = http.post(`${BASE_URL}/api/orders`, payload, params);
    orderLatency.add(Date.now() - start);
    check(res, { 'order 201': (r) => r.status === 201 });
    errorRate.add(res.status !== 201);
  }
  
  // Think time: simula pausa do usuário (1-3s)
  sleep(Math.random() * 2 + 1);
}
```

```bash
# Executar k6
k6 run --out json=results.json load-test.js

# Com variáveis de ambiente
k6 run -e BASE_URL=https://staging.example.com load-test.js

# Exportar para Grafana/InfluxDB
k6 run --out influxdb=http://localhost:8086/k6 load-test.js
```

### Gatling — exemplo

```scala
// Gatling — Exemplo de simulation

class OrderSimulation extends Simulation {
  
  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .shareConnections  // connection pooling realista
  
  val catalog = scenario("Browse Catalog")
    .exec(
      http("list_products")
        .get("/api/catalog/products")
        .check(status.is(200))
    )
    .pause(1, 3)  // think time 1-3s
  
  val order = scenario("Place Order")
    .exec(
      http("create_order")
        .post("/api/orders")
        .body(StringBody("""{"productId": ${productId}, "quantity": 1}"""))
        .check(status.is(201))
    )
    .pause(2, 5)
  
  setUp(
    catalog.inject(
      rampUsersPerSec(10).to(500).during(5.minutes),
      constantUsersPerSec(500).during(30.minutes),
    ),
    order.inject(
      rampUsersPerSec(5).to(100).during(5.minutes),
      constantUsersPerSec(100).during(30.minutes),
    ),
  ).protocols(httpProtocol)
   .assertions(
     global.responseTime.percentile3.lt(500),  // p95 < 500ms
     global.successfulRequests.percent.gt(99),  // > 99% success
   )
}
```

---

## Rigor Estatístico

### Warmup e steady state

```
POR QUE WARMUP É ESSENCIAL:

  Sem warmup:
  Latency
  ▲
  │ ██
  │ ██                             ← Cold start: JIT, caches frias, 
  │ ██                               connection pools frios
  │ ██ ██
  │ ██ ██ ██
  │ ██ ██ ██ ██ ██ ██ ██ ██ ██    ← Steady state
  ├──────────────────────────────► Tempo
   0    2    4    6    8   10 min

  Se INCLUIR warmup nos resultados → latência artificialmente ALTA
  
  REGRAS:
  → Warmup: 2-5 minutos (descartar dados)
  → JVM: JIT precisa ~10,000 invocações para compilar
  → Caches: precisam preencher
  → Connection pools: precisam abrir conexões
  → DB query planner: precisam aquecer estatísticas
  
  COMO GARANTIR:
  → k6/Gatling: use ramp-up phase (não meça)
  → JMH: @Warmup(iterations = 5)
  → Go: b.ResetTimer() após setup
  → Manual: descarte primeiros 5 min de dados
```

### Variance e confidence intervals

```
RIGOR ESTATÍSTICO — NÃO CONFIE EM UMA ÚNICA MEDIÇÃO:

  PROBLEMA:
  Run 1: 850ns/op
  Run 2: 920ns/op  
  Run 3: 780ns/op
  → Qual é o "resultado"? Range: 780-920 (18% variação!)

  SOLUÇÃO: MÚLTIPLAS RUNS + ESTATÍSTICA

  1. REPETIR (mínimo 5x, recomendado 10x)
     → JMH: @Fork(3) + @Measurement(iterations=10)
     → Go: -count=10
     → k6: rodar 3 vezes, comparar distribuições

  2. REPORTAR DISTRIBUIÇÃO (não só média)
     → p50 (mediana), p95, p99, p99.9
     → Mean ± StdDev
     → Confidence interval (95% CI)

  3. TESTE DE SIGNIFICÂNCIA
     → "É 5% mais rápido" → significativo ou ruído?
     → benchstat (Go): reporta p-value
     → Se p > 0.05: NÃO é significativo (noise)
     → Se p < 0.05: provavelmente significativo

  EXEMPLO:
  
  │ Métrica     │ Versão A      │ Versão B      │ Significativo? │
  │ p50         │ 12ms          │ 11ms          │               │
  │ p95         │ 45ms ± 3ms    │ 38ms ± 2ms    │ ✅ p=0.002    │
  │ p99         │ 120ms ± 15ms  │ 110ms ± 12ms  │ ❌ p=0.12     │
  │ throughput  │ 5,000 ± 200   │ 5,800 ± 150   │ ✅ p=0.001    │

  INTERPRETAÇÃO:
  → p95 melhorou significativamente (15% reduction)
  → p99 NÃO podemos afirmar que melhorou (variance muito alta)
  → Throughput: melhoria real de ~16%
```

### Histograma de latência

```
POR QUE HISTOGRAMA > PERCENTILES:

  Percentiles escondem FORMA da distribuição:

  BIMODAL (dois picos — cache hit vs miss):
  
  Count
  ▲ ██
  │ ██
  │ ██ ██                     ██
  │ ██ ██ ██               ██ ██
  │ ██ ██ ██ ██         ██ ██ ██
  │ ██ ██ ██ ██ ██   ██ ██ ██ ██
  ├──────────────────────────────► Latency (ms)
    1  5  10 20 50  200 500 1000
     Cache Hit       Cache Miss

  p50=10ms, p99=800ms ← percentiles não mostram que é BIMODAL
  Histograma: MOSTRA claramente os dois modos
  → Ação: melhorar cache hit ratio (não otimizar código)

  FERRAMENTA: HdrHistogram (Gil Tene)
  → Resolução de nano/microsegundos
  → Compressão logarítmica
  → Merge de histogramas distribuídos
  → Suporte em: Java, Go, Python, C, JS
```

---

## Ambiente de Teste

```
AMBIENTE DE LOAD TEST — CONSIDERAÇÕES:

  1. PRODUÇÃO (ideal, mas nem sempre possível)
     ✅ Dados reais, infraestrutura real, latência real
     ❌ Risco de impacto em usuários
     → Se possível: canary/shadow traffic
     → Dark launch: processa em paralelo, não retorna resultado

  2. STAGING (recomendado como padrão)
     ✅ Ambiente controlado
     ✅ Sem risco para produção
     ❌ Precisa ser proporcional a produção
     → Scale: staging = 1/10 de prod → multiplique resultados
     → OU staging = prod (custo alto, mais preciso)

  3. CONSIDERAÇÕES CRÍTICAS:
     
     │ Fator            │ Impacto se diferente de prod   │
     │ Data volume      │ Query plans mudam com volume   │
     │ Network latency  │ Staging pode ter 0ms inter-svc│
     │ Hardware          │ CPU/memory diferente = dados  │
     │                  │   não comparáveis               │
     │ External deps    │ Mocks ≠ real latency/behavior  │
     │ Cache state      │ Cache fria vs quente            │
     │ TLS              │ TLS termination adiciona ~1ms  │
     │ Load balancer    │ Diferente = routing diferente  │

  4. ISOLAMENTO:
     → Load test NÃO deve rodar no mesmo hardware que o sistema
     → Load generators em máquinas separadas
     → Monitor load generator CPU (se > 80%, resultados invalidos)
     → Network entre loader e sistema: medir e reportar
```

---

## Análise de Resultados

### Como analisar resultados de load test

```
ANÁLISE DE RESULTADOS:

  1. VERIFICAR VALIDADE DO TESTE
     □ Load generator não estava saturado? (CPU < 80%)
     □ Network não era bottleneck? (sem retransmits excessivos)
     □ Warmup foi descartado?
     □ Duração suficiente para steady state?
     □ Erro rate < threshold?

  2. MÉTRICAS PRIMÁRIAS
     
     │ Métrica          │ Target (exemplo)  │ Resultado │ Status │
     │ Throughput        │ ≥ 5,000 req/s     │ 5,200     │ ✅     │
     │ Latency p50       │ < 50ms            │ 35ms      │ ✅     │
     │ Latency p95       │ < 200ms           │ 180ms     │ ✅     │
     │ Latency p99       │ < 500ms           │ 620ms     │ ❌     │
     │ Error rate        │ < 0.1%            │ 0.05%     │ ✅     │
     │ CPU utilization   │ < 70%             │ 65%       │ ✅     │
     │ Memory            │ < 80%             │ 72%       │ ✅     │

  3. PADRÕES A OBSERVAR

     Throughput satura enquanto latência cresce:
     → Gargalo encontrado, precisa escalar
     
     Throughput
     ▲         ┌───────────────    Latency
     │        ╱│                   ▲           ╱
     │      ╱  │                   │         ╱
     │    ╱    │ ← saturation      │       ╱
     │  ╱      │   point           │     ╱
     │╱        │                   │   ╱
     ├─────────┼──────► VUs        ├──────────► VUs
     
     Latência cresce linearmente com VUs:
     → Queuing: recurso limitado (pool, CPU, disk)
     
     Latência tem spikes periódicos:
     → GC pauses! Verificar GC logs

  4. COMPARAR COM BASELINE (obrigatório)
     → Delta vs último release
     → Regressão > 10%: investigate before merge
     → Relatório: tabela before/after com % change
```

---

## Regression Detection em CI/CD

```
PERFORMANCE REGRESSION DETECTION:

  PIPELINE:

  ┌──────────┐    ┌──────────┐    ┌──────────────┐
  │  Build   │───►│  Unit    │───►│  Performance │
  │          │    │  Tests   │    │  Benchmark   │
  └──────────┘    └──────────┘    └──────┬───────┘
                                         │
                                  ┌──────▼───────┐
                                  │  Compare vs  │
                                  │  Baseline    │
                                  └──────┬───────┘
                                         │
                              ┌──────────┴──────────┐
                              │                     │
                       ┌──────▼──────┐       ┌──────▼──────┐
                       │  Regression │       │  No         │
                       │  Detected   │       │  Regression │
                       │  → FAIL     │       │  → PASS     │
                       └─────────────┘       └─────────────┘

  IMPLEMENTAÇÃO:

  1. MICROBENCHMARK EM CI (rápido, ~2 min)
     → JMH / Go bench em cada PR
     → Compare com main branch: go test -bench . | benchstat
     → Threshold: regression > 10% AND p < 0.05 → fail

  2. LOAD TEST EM CD (demorado, ~30 min)
     → Após merge em main / pré-deploy
     → k6 / Gatling contra staging
     → Compare com último run estável
     → Threshold: p99 regression > 15% → block deploy

  3. PRODUÇÃO (continuous)
     → Canary deploy: compare canary vs stable
     → p99 do canary > 20% pior → auto-rollback
     → Continuous profiling: diff flame graphs

  ARMAZENAMENTO DE RESULTADOS:
  → Banco de benchmarks históricos
  → Time series: throughput, latency por commit
  → Dashboard: trend ao longo de releases
  → Alerting: degradação significativa
```

### Exemplo de CI pipeline

```yaml
# GitHub Actions — Performance Benchmark em CI

name: Performance Benchmark
on:
  pull_request:
    branches: [main]

jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      
      # Benchmark do PR
      - name: Run benchmarks (PR)
        run: go test -bench=. -count=10 -benchmem ./... > new.txt
      
      # Benchmark do main (baseline)
      - name: Checkout main
        uses: actions/checkout@v4
        with:
          ref: main
          path: main-branch
      
      - name: Run benchmarks (main)
        run: |
          cd main-branch
          go test -bench=. -count=10 -benchmem ./... > ../old.txt
      
      # Comparar
      - name: Compare benchmarks
        run: |
          go install golang.org/x/perf/cmd/benchstat@latest
          benchstat old.txt new.txt | tee benchmark-results.txt
          
          # Fail se regression > 10%
          if grep -E '\+[1-9][0-9]\.[0-9]+%' benchmark-results.txt; then
            echo "::error::Performance regression detected!"
            exit 1
          fi
      
      - name: Comment PR with results
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const results = fs.readFileSync('benchmark-results.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Benchmark Results\n\`\`\`\n${results}\n\`\`\``
            });
```

---

## Anti-Patterns de Benchmarking

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **Benchmark sem warmup** | Inclui cold start nos resultados | Descartar primeiros 2-5 min; usar warmup explícito |
| **Uma única medição** | Não captura variance, não é significativo | Mínimo 5 runs, idealmente 10 |
| **Reportar só média** | Esconde tail latency e bimodal | Percentiles (p50, p95, p99) + histograma |
| **Coordinated omission** | Latência underreported | Usar wrk2, k6, Gatling — NÃO wrk ou ab |
| **Dev machine benchmark** | Resultados não reproduzíveis | CI/CD environment consistente ou staging |
| **Benchmark sem baseline** | "É rápido" comparado a quê? | Sempre comparar before/after |
| **Workload irrealista** | 100% writes, 0 think time | Mix realista, think time, dados variados |
| **Load generator saturado** | Client-side bottleneck nos resultados | Monitorar CPU do load generator (< 80%) |
| **Ignorar variability** | "5% mais rápido" sem significância | benchstat, p-value < 0.05 |
| **Micro-benchmark generalizado** | "Protobuf é sempre melhor que JSON" | Benchmark no SEU use case com SEUS dados |
| **Não versionar benchmarks** | Não consegue comparar versions | Benchmarks no repositório, resultados históricos |
| **Ignorar GC durante benchmark** | GC pauses mascarados pela média | Forçar GC antes, monitorar durante |

---

## Diretrizes para Code Review assistido por AI

Ao revisar benchmarks, load tests e análises de performance, verifique:

1. **Benchmark sem warmup** — JMH: @Warmup presente; Go: b.ResetTimer() após setup; Load test: ramp-up phase
2. **Coordinated Omission** — Se usa wrk ou ab para medir latência: REJEITAR; exigir wrk2, k6, Gatling
3. **Uma única run** — Benchmarks devem ter ≥5 iterations/forks para validade estatística
4. **Só média reportada** — Exigir p50, p95, p99 no mínimo; histograma para análise completa
5. **Microbenchmark sem JMH/testing.B** — Benchmark manual com System.nanoTime() em loop: REJEITAR
6. **Dead code elimination** — JMH: resultado deve ser consumido (@Benchmark return ou Blackhole); Go: result global var
7. **Load test sem think time** — Sem pausa entre requests = não simula usuário real
8. **Threshold ausente** — Load test sem criterio de pass/fail definido = inútil para CI/CD
9. **Sem comparação com baseline** — Resultado absoluto sem delta vs versão anterior
10. **Load generator no mesmo host** — Generator e system under test devem ser em hosts separados
11. **Benchmark de operação não-hot-path** — Se profiler não identificou como hotspot, micro-benchmark é premature optimization
12. **Falta de análise de variância** — Resultados sem confidence interval ou p-value não permitem conclusões

---

## Referências

- **Systems Performance** — Brendan Gregg (Addison-Wesley, 2020) — Chapter: Benchmarking
- **Java Performance** — Scott Oaks (O'Reilly, 2020) — JMH chapters
- **JMH** — OpenJDK — https://openjdk.org/projects/code-tools/jmh/
- **k6** — Grafana — https://k6.io/docs/
- **Gatling** — https://gatling.io/docs/
- **Locust** — https://docs.locust.io/
- **wrk2** — Gil Tene — https://github.com/giltene/wrk2
- **benchstat** — Go — https://pkg.go.dev/golang.org/x/perf/cmd/benchstat
- **HdrHistogram** — Gil Tene — https://github.com/HdrHistogram/HdrHistogram
- **"How NOT to Measure Latency"** — Gil Tene — coordinated omission
- **Release It!** — Michael Nygard (Pragmatic Bookshelf, 2018) — Stability patterns
- **Continuous Profiling** — Pyroscope — https://pyroscope.io/
