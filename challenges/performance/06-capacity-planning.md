# Level 6 — Capacity Planning

> **Objetivo:** Dominar capacity planning — modelagem de carga, forecasting de crescimento, right-sizing de recursos, estratégias de escalabilidade e otimização de custos — aplicando Little's Law, USL (Universal Scalability Law) e load modeling para prever a capacidade necessária antes de problemas em produção.

---

## Objetivo de Aprendizado

- Caracterizar workloads reais (padrões de uso, sazonalidade, picos)
- Aplicar Little's Law para dimensionar pools, filas e workers
- Usar USL (Universal Scalability Law) para prever limites de escalabilidade
- Criar modelos de carga que traduzem demanda de negócio em recursos técnicos
- Implementar forecasting de crescimento (linear, exponencial, sazonal)
- Right-sizing: otimizar CPU, memória, disco e réplicas
- Planejar escalabilidade vertical vs horizontal
- Integrar capacity planning com observability (Prometheus/Grafana)

---

## Escopo Funcional

### Digital Wallet — Cenário de Capacity Planning

```
CENÁRIO DE NEGÓCIO:

A Digital Wallet API tem atualmente:
  - 50.000 usuários ativos diários (DAU)
  - 200.000 transações/dia
  - Pico: 15x a média durante promoções (Black Friday)
  - Crescimento: 20% mês a mês
  - SLA: 99.9% availability, p99 < 500ms

PERGUNTAS QUE CAPACITY PLANNING RESPONDE:
  1. Quantos pods/instâncias preciso HOJE para o SLA?
  2. Quantos AMANHÃ (próximos 3, 6, 12 meses)?
  3. Qual o LIMITE do meu design atual antes de re-arquiteturar?
  4. Quanto CUSTA escalar para o crescimento projetado?
  5. Qual configuração de database (CPU, RAM, connections) é necessária?
```

---

## Escopo Técnico

### 1. Workload Characterization

```
── Passo 1: Coletar métricas de produção ──

FONTES DE DADOS:
  - Prometheus/Grafana: RPS, latência, error rate, CPU, RAM
  - APM (Datadog, New Relic): traces, throughput por endpoint
  - Load balancer logs: distribuição de requests
  - Database: QPS, connection count, slow queries
  - Business metrics: DAU, MAU, transações/dia, receita/hora

── Passo 2: Identificar padrões ──

MÉTRICAS-CHAVE PARA CAPACITY:
  - Peak-to-average ratio (PAR): pico / média diária
  - Requests per active user (RPU): requests / DAU
  - Growth rate: variação MoM (month-over-month)
  - Sazonalidade: hora do dia, dia da semana, feriados
  - Correlação: quais métricas de negócio correlacionam com carga?

EXEMPLO DIGITAL WALLET:
  DAU atual:          50.000
  Requests/user/dia:  12 (média)
  Total requests/dia: 600.000
  RPS médio:          600K / 86400 = ~7 RPS
  RPS pico (2x):      ~14 RPS
  RPS pico absoluto:  ~105 RPS (15x durante Black Friday)
  
  DISTRIBUIÇÃO DE ENDPOINTS:
  GET  /wallets/{id}              35%   (~2.5 RPS médio)
  GET  /wallets/{id}/transactions 25%   (~1.75 RPS médio)
  POST /transactions/deposit      20%   (~1.4 RPS médio)
  POST /transactions/transfer     10%   (~0.7 RPS médio)
  POST /users                     5%    (~0.35 RPS médio)
  GET  /users/{id}                5%    (~0.35 RPS médio)
```

### 2. Little's Law Aplicado

```
L = λ × W

L = número de requests concorrentes (em execução)
λ = taxa de chegada (requests/segundo)
W = tempo de processamento (segundos/request)

── Aplicação 1: Connection Pool de Banco ──

λ (pico) = 105 RPS
W (query média) = 10ms = 0.01s
L = 105 × 0.01 = 1.05 conexões necessárias

Mas com p99 de 50ms:
L_p99 = 105 × 0.05 = 5.25 conexões

Com margem de segurança (2x):
Pool size recomendado = ceil(5.25 × 2) = 11 conexões

── Aplicação 2: Worker Pool / Thread Pool ──

λ (pico) = 105 RPS
W (handler médio) = 20ms = 0.02s (inclui DB + serialização)
L = 105 × 0.02 = 2.1 workers ocupados em média

Com p99 de 200ms:
L_p99 = 105 × 0.2 = 21 workers

Thread pool recomendado = ceil(21 × 1.5) = 32 threads

── Aplicação 3: Réplicas de API ──

Capacidade por pod (medida em load test): 500 RPS
Carga de pico: 105 RPS
Réplicas mínimas = ceil(105 / 500) = 1

Com headroom de 30%:
Réplicas = ceil(105 / (500 × 0.7)) = 1 (ainda 1, carga baixa)

Projeção 6 meses (20% MoM):
RPS_6m = 105 × (1.2)^6 = 105 × 2.99 = 314 RPS
Réplicas_6m = ceil(314 / (500 × 0.7)) = 1 (ainda 1)

Projeção 12 meses:
RPS_12m = 105 × (1.2)^12 = 105 × 8.92 = 937 RPS
Réplicas_12m = ceil(937 / (500 × 0.7)) = 3
```

### 3. USL — Universal Scalability Law

```
               λ(1)⋅N
C(N) = ─────────────────────────
       1 + σ(N−1) + κ⋅N⋅(N−1)

Onde:
  C(N) = throughput com N workers/réplicas/cores
  λ(1) = throughput com 1 worker
  N    = número de workers
  σ    = contention coefficient (serialização, locks)
  κ    = coherency coefficient (crosstalk, cache invalidation)

TIPOS DE SCALING:

Linear (σ≈0, κ≈0):
  C(N) = λ(1) × N
  → Ideal, nunca acontece na prática

Sub-linear (σ>0, κ≈0) — Amdahl's Law:
  → Throughput cresce mas com rendimento decrescente
  → Causado por serialized sections (locks, I/O sequencial)

Retrograde (σ>0, κ>0):
  → Throughput DIMINUI após certo ponto!
  → Causado por coherency (cache invalidation, distributed locks)
  → N* = ponto ótimo = √((1−σ)/(κ))
```

```go
// usl.go — Implementação da Universal Scalability Law
package capacity

import (
    "fmt"
    "math"
)

type USLModel struct {
    Lambda1 float64 // throughput com 1 worker
    Sigma   float64 // contention coefficient
    Kappa   float64 // coherency coefficient
}

// Throughput previsto para N workers
func (u USLModel) Throughput(n float64) float64 {
    return (u.Lambda1 * n) / (1 + u.Sigma*(n-1) + u.Kappa*n*(n-1))
}

// N ótimo (ponto de máximo throughput)
func (u USLModel) OptimalN() float64 {
    if u.Kappa <= 0 {
        return math.Inf(1) // sem retrograde scaling
    }
    return math.Sqrt((1 - u.Sigma) / u.Kappa)
}

// Máximo throughput teórico
func (u USLModel) MaxThroughput() float64 {
    nOpt := u.OptimalN()
    return u.Throughput(nOpt)
}

func Example() {
    // Valores obtidos por regressão em dados de load test
    model := USLModel{
        Lambda1: 500,   // 500 RPS com 1 réplica
        Sigma:   0.02,  // 2% contention
        Kappa:   0.001, // 0.1% coherency
    }

    fmt.Println("Réplicas | Throughput Previsto")
    fmt.Println("---------|-------------------")
    for n := 1.0; n <= 20; n++ {
        fmt.Printf("   %2.0f    | %8.1f RPS\n", n, model.Throughput(n))
    }

    fmt.Printf("\nN ótimo: %.1f réplicas\n", model.OptimalN())
    fmt.Printf("Max throughput: %.1f RPS\n", model.MaxThroughput())
}
```

### 4. Forecasting de Crescimento

```
── Modelos de Forecast ──

LINEAR:
  y(t) = a + b⋅t
  Bom para: crescimento estável, curto prazo

EXPONENCIAL:
  y(t) = a ⋅ e^(b⋅t)
  Bom para: startups, crescimento rápido

LOGÍSTICO (S-curve):
  y(t) = K / (1 + e^(-b⋅(t−t₀)))
  K = carrying capacity (teto)
  Bom para: mercados com saturação

SAZONAL (Holt-Winters):
  y(t) = (Level + Trend⋅t) × Seasonality(t)
  Bom para: padrões cíclicos (hora, dia, semana, mês)
```

```python
# forecast.py — Projeção de capacidade
import numpy as np
from datetime import datetime, timedelta

def exponential_forecast(current_rps, growth_rate_monthly, months):
    """Projetar RPS futuro com crescimento exponencial."""
    projections = []
    for m in range(1, months + 1):
        future_rps = current_rps * (1 + growth_rate_monthly) ** m
        date = datetime.now() + timedelta(days=30 * m)
        projections.append({
            "month": m,
            "date": date.strftime("%Y-%m"),
            "rps_avg": round(future_rps, 1),
            "rps_peak": round(future_rps * 15, 1),  # 15x PAR
        })
    return projections

def calculate_resources(rps_peak, rps_per_pod, headroom=0.3):
    """Calcular pods necessários."""
    effective_capacity = rps_per_pod * (1 - headroom)
    pods = int(np.ceil(rps_peak / effective_capacity))
    return max(pods, 2)  # mínimo 2 para HA

def calculate_db_connections(rps_peak, query_time_ms, safety_factor=2):
    """Little's Law para pool de banco."""
    concurrent = rps_peak * (query_time_ms / 1000)
    pool_size = int(np.ceil(concurrent * safety_factor))
    return max(pool_size, 5)  # mínimo 5

# ── Projeção 12 meses ──
projections = exponential_forecast(
    current_rps=7,         # 7 RPS médio atual
    growth_rate_monthly=0.20,  # 20% MoM
    months=12
)

print("Month | Date    | RPS Avg | RPS Peak | Pods | DB Pool")
print("------|---------|---------|----------|------|--------")
for p in projections:
    pods = calculate_resources(p["rps_peak"], rps_per_pod=500)
    pool = calculate_db_connections(p["rps_peak"], query_time_ms=10)
    print(f"  {p['month']:2d}  | {p['date']} | {p['rps_avg']:7.1f} | {p['rps_peak']:8.1f} | {pods:4d} | {pool:6d}")
```

### 5. Right-Sizing

```
── CPU ──

Coletar: cpu_usage média e p95 por pod (últimos 7 dias)
Regra:
  p95 CPU < 70% → pod over-provisioned (reduzir)
  p95 CPU > 70% → scale horizontalmente
  avg CPU < 20% → desperdício significativo

── Memória ──

Coletar: memory_usage média e max por pod
Regra:
  max memory < 50% do limit → reduzir limit
  max memory > 80% do limit → aumentar ou scale
  
  Go: memória estável (sem GC tuning necessário)
  JVM: -Xmx = 75% do container memory limit
       Restante: metaspace + thread stacks + native memory

── Database ──

Connection pool: Little's Law (ver seção 2)
  pool_size = ceil(RPS_peak × query_time_p99 × safety_factor)

Storage: 
  crescimento_diario = transações/dia × tamanho_médio_row
  projeção_12m = crescimento_diario × 365 × safety_factor

Réplicas de leitura:
  Se read_ratio > 80% → 1-2 read replicas
  Se read_ratio > 90% → 2-3 read replicas
```

### 6. Kubernetes Resource Planning

```yaml
# deployment.yaml — Resource requests e limits baseados em capacity planning
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wallet-api
spec:
  replicas: 3  # Baseado em projeção de 6 meses
  template:
    spec:
      containers:
      - name: wallet-api
        resources:
          requests:          # Baseado em p50 de uso (scheduling)
            cpu: "250m"      # 0.25 cores
            memory: "256Mi"  # 256 MB
          limits:            # Baseado em p99 de uso (throttling)
            cpu: "1000m"     # 1 core
            memory: "512Mi"  # 512 MB (JVM: -Xmx384m)

---
# HPA baseado em capacity planning
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: wallet-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wallet-api
  minReplicas: 2       # HA mínimo
  maxReplicas: 10      # Projeção de 12 meses + headroom
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale at 70% CPU
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "350"  # 70% da capacidade do pod (500 RPS)
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100          # Dobrar pods se necessário
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Esperar 5 min antes de scale down
      policies:
      - type: Percent
        value: 10           # Reduzir max 10% por vez
        periodSeconds: 60
```

### 7. Dashboard de Capacity Planning

```
── Grafana Dashboard: Capacity Planning ──

PAINEL 1: Projeção de Throughput
  Query: rate(http_requests_total[5m])
  + Overlay: linha de projeção exponencial
  + Threshold: capacidade máxima do cluster

PAINEL 2: Utilização de Recursos
  CPU: avg(rate(container_cpu_usage_seconds_total[5m]))
  RAM: avg(container_memory_working_set_bytes)
  + Threshold: 70% (scale trigger)

PAINEL 3: Database Capacity
  Connections: pg_stat_activity_count
  QPS: rate(pg_stat_user_tables_idx_scan[5m])
  Storage: pg_database_size_bytes
  + Projeção de storage para 12 meses

PAINEL 4: Cost Estimation
  Pods × pod_cost + DB_size × storage_cost + network_cost
  + Projeção de custo mensal

PAINEL 5: Headroom
  (capacidade_max - uso_atual) / capacidade_max × 100
  VERDE: > 50% headroom
  AMARELO: 30-50% headroom
  VERMELHO: < 30% headroom → ação necessária

ALERTAS:
  - Headroom < 30% → warning (Slack)
  - Headroom < 15% → critical (PagerDuty)
  - Growth rate > projetado → info (review forecast)
```

---

## Critérios de Aceite

- [ ] **Workload characterization** documentada para a Digital Wallet API
- [ ] **Little's Law** aplicado para dimensionar: connection pool, thread pool, réplicas
- [ ] **USL model** implementado em Go (`usl.go`) com:
  - Curva de throughput previsto vs N réplicas
  - N ótimo identificado
  - Validação contra dados de load test
- [ ] **Forecasting**: projeção de 12 meses (RPS, pods, DB connections, storage)
- [ ] **Right-sizing**: análise de CPU/memória de cada pod com recomendações
- [ ] **Kubernetes manifests**: Deployment, HPA, resource requests/limits baseados em dados
- [ ] **Capacity Planning dashboard** no Grafana com projeções e alertas
- [ ] Script `forecast.py` gerando tabela de projeções
- [ ] Relatório `CAPACITY_PLAN.md` com:
  - Workload characterization
  - Modelo de crescimento (com dados)
  - Projeções 3/6/12 meses
  - Estimativa de custo
  - Recomendações de ação

---

## Definição de Pronto (DoD)

- [ ] `usl.go` com modelo USL funcional e testes
- [ ] `forecast.py` com projeção e tabela de recursos
- [ ] Kubernetes manifests em `k8s/capacity/`
- [ ] Dashboard Grafana exportado em `grafana/capacity-planning.json`
- [ ] Relatório `CAPACITY_PLAN.md` completo
- [ ] Commit: `feat(perf-level-6): add capacity planning with USL, forecasting, and K8s HPA`

---

## Checklist

### Workload Characterization

- [ ] Coletar métricas de 7+ dias (ou simular com load test de longa duração)
- [ ] Calcular: RPS médio, pico, PAR, RPU, distribuição por endpoint
- [ ] Identificar padrões sazonais (hora, dia, semana)
- [ ] Correlacionar métricas de negócio com carga técnica

### Little's Law

- [ ] Calcular pool de banco: `Pool = RPS_pico × query_p99 × safety_factor`
- [ ] Calcular thread pool: `Threads = RPS_pico × handler_p99 × safety_factor`
- [ ] Calcular réplicas: `Replicas = ceil(RPS_pico / (capacity_pod × utilization_target))`
- [ ] Validar com load test: pool very small → timeout; pool correto → OK

### USL

- [ ] Executar load test com 1, 2, 4, 8, 16 réplicas
- [ ] Coletar throughput para cada N
- [ ] Fit modelo USL (regressão) para obter σ e κ
- [ ] Prever N ótimo e throughput máximo

### Forecasting

- [ ] Definir growth rate baseado em dados históricos (ou premissa de negócio)
- [ ] Projetar RPS para 3, 6, 12 meses
- [ ] Calcular recursos (pods, DB, storage) para cada milestone
- [ ] Estimar custo (cloud provider pricing)

---

## Tarefas Sugeridas por Stack

### Go (Gin)

- [ ] `usl.go` com implementação USL + testes
- [ ] `capacity_test.go` validando Little's Law com benchmark
- [ ] Configurar `GOMAXPROCS` baseado em CPU request do container
- [ ] pgxpool: validar pool size com Little's Law

### Spring Boot

- [ ] HikariCP pool sizing com Little's Law
  ```yaml
  spring:
    datasource:
      hikari:
        maximum-pool-size: 11  # Little's Law: ceil(105 × 0.05 × 2)
        minimum-idle: 5
  ```
- [ ] JVM sizing: `-Xmx` = 75% do container memory limit
- [ ] Spring Boot Actuator + Micrometer para métricas de capacity

### Quarkus

- [ ] Agroal pool sizing com Little's Law
- [ ] Native image: memory footprint vs JIT
- [ ] Quarkus reactive: event loop sizing (`quarkus.vertx.worker-pool-size`)

### Micronaut

- [ ] HikariCP / connection pool sizing
- [ ] Netty event loop tuning
- [ ] Startup time benchmark para cold start scenarios (serverless)

### Jakarta EE

- [ ] DataSource pool sizing no application server
- [ ] Thread pool do application server (WildFly, Payara)
- [ ] JCA connection factory sizing

---

## Extensões Opcionais

- [ ] Simular Black Friday: 15x spike, verificar auto-scaling reage em tempo
- [ ] Cost modeling com AWS/GCP pricing calculator
- [ ] Capacity planning para banco de dados (RDS sizing, Aurora scaling)
- [ ] Load shedding: implementar adaptive throttling quando headroom < 10%
- [ ] Chaos experiment: remover 1 réplica durante pico, observar degradation
- [ ] Vertical Pod Autoscaler (VPA) vs HPA: comparar abordagens

---

## Erros Comuns

| Erro | Consequência | Correção |
|------|-------------|----------|
| Dimensionar pela média, não pelo pico | Under-provisioning em promoções | Sempre usar RPS_pico × PAR |
| Connection pool muito grande | Saturar DB com connections idle | Little's Law + safety factor |
| Ignorar sazonalidade | Surpresa durante picos previsíveis | Analisar 4+ semanas de dados |
| Forecast linear para crescimento exponencial | Sub-dimensionar em 6+ meses | Modelar growth rate correto |
| HPA sem stabilization window | Thrashing: pods subindo e descendo | scaleDown window ≥ 300s |
| CPU limit muito baixo | Throttling durante picos | Limit = 2-4x request |
| Não considerar headroom | Sem margem para picos inesperados | 30% headroom mínimo |
| Over-provisioning "just in case" | Custo desnecessário | Right-sizing baseado em dados |

---

## Como Isso Aparece em Entrevistas

- "Como você dimensionaria um connection pool de banco de dados?"
- "Explique Little's Law e dê um exemplo prático"
- "Sua API recebe 1000 req/s com tempo médio de 50ms. Quantas threads precisam?"
- "Como prever quantos servidores serão necessários em 6 meses?"
- "Explique a Universal Scalability Law. O que são σ e κ?"
- "Como configurar HPA no Kubernetes para uma API HTTP?"
- "Qual a diferença entre vertical e horizontal scaling? Quando usar cada um?"
- "Descreva seu processo de capacity planning para um sistema novo"
