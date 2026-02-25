# Performance Engineering — Capacity Planning e Load Modeling

> **Objetivo deste documento:** Servir como referência completa sobre **capacity planning, load modeling, forecasting e right-sizing** — como prever e planejar recursos para atender demanda presente e futura, otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: metodologia de capacity planning, workload characterization, tráfego e load models, forecasting, scaling strategies, cost optimization.

---

## Quick Reference — Cheat Sheet

| Conceito | Fórmula / Regra | Quando usar |
|----------|-----------------|-------------|
| **Little's Law** | L = λ × W | Sizing pools, threads, conexões |
| **Utilization target** | ≤ 70% sustained | Headroom para picos |
| **Throughput capacity** | recursos × (1/tempo_por_request) | Throughput máximo teórico |
| **Headroom** | Capacity − Peak Load ≥ 30% | Planejamento de margem |
| **Growth forecast** | Load_futuro = Load_atual × (1 + growth_rate)^n | Quando provisionar mais |
| **N+1 redundancy** | Capacidade sobrevive perda de 1 nó | Fault tolerance mínima |
| **Cost per request** | Custo_total / total_requests | Eficiência de infra |

---

## Sumário

- [Performance Engineering — Capacity Planning e Load Modeling](#performance-engineering--capacity-planning-e-load-modeling)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que é Capacity Planning](#o-que-é-capacity-planning)
  - [Workload Characterization](#workload-characterization)
  - [Load Modeling](#load-modeling)
  - [Forecasting](#forecasting)
  - [Right-Sizing](#right-sizing)
  - [Scaling Strategies](#scaling-strategies)
  - [Cloud Capacity Planning](#cloud-capacity-planning)
  - [Capacity Planning por Componente](#capacity-planning-por-componente)
  - [Processo de Capacity Review](#processo-de-capacity-review)
  - [Anti-Patterns de Capacity Planning](#anti-patterns-de-capacity-planning)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que é Capacity Planning

```
CAPACITY PLANNING = garantir que o sistema tem RECURSOS SUFICIENTES
para atender a DEMANDA ESPERADA com QUALIDADE (SLOs) e CUSTO aceitável.

  ┌─────────────────────────────────────────────────────────┐
  │                                                          │
  │    DEMANDA              CAPACIDADE                       │
  │    (Load)               (Resources)                      │
  │                                                          │
  │    Requests/s    ───►   CPU cores                        │
  │    QPS           ───►   Memory (GB)                      │
  │    Connections   ───►   Network bandwidth                │
  │    Storage ops   ───►   Disk IOPS/throughput             │
  │    Concurrent    ───►   Thread/connection pools           │
  │    users                                                 │
  │                                                          │
  │   CAPACITY PLANNING RESPONDE:                            │
  │   1. Quanto recurso PRECISO para a carga atual?          │
  │   2. Quanto recurso VOU PRECISAR daqui a 3/6/12 meses?  │
  │   3. Quando preciso ESCALAR?                             │
  │   4. Quanto vai CUSTAR?                                  │
  │   5. O que acontece se um componente FALHAR?             │
  │                                                          │
  └─────────────────────────────────────────────────────────┘

  CAPACITY PLANNING vs PERFORMANCE TUNING:
  
  Performance Tuning: "como fazer MAIS com os MESMOS recursos?"
  Capacity Planning:  "QUANTOS recursos para atender a demanda?"

  → São complementares: tune primeiro, depois planeje capacidade
  → Tuning reduz a necessidade de capacidade
  → Capacity planning sem tuning = desperdício de dinheiro
```

---

## Workload Characterization

### Entendendo o padrão de tráfego

```
WORKLOAD CHARACTERIZATION — CONHECER SUA CARGA:

  1. QUEM (clientes/consumidores)
     → Usuários (browser, mobile, API partners)
     → Serviços internos (batch jobs, cron, queues)
     → Proporção: 80% user, 15% internal, 5% partner?

  2. O QUÊ (operações)
     → Read:Write ratio (e.g., 95:5 para e-commerce catalog)
     → Tipos de request: CRUD, search, aggregation, report
     → Payload sizes: distribuição (p50=2KB, p99=500KB)

  3. QUANDO (padrão temporal)
     → Diurnal pattern (pico diário)
     → Weekly pattern (fim de semana diferente?)
     → Seasonal (Black Friday, volta às aulas)
     → Growth trend (MoM, YoY)

  4. QUANTO (volume)
     → Peak QPS
     → Sustained throughput
     → Burst factor = Peak / Average

  EXEMPLO DE CHARACTERIZATION:

  │ Métrica         │ Average      │ Peak         │ Burst │
  │ Requests/s      │ 2,000        │ 8,000        │ 4x    │
  │ Read:Write      │ 90:10        │ 85:15        │       │
  │ Payload (p50)   │ 5 KB         │ 5 KB         │       │
  │ Payload (p99)   │ 200 KB       │ 500 KB       │ 2.5x  │
  │ Concurrent conn │ 500          │ 2,000        │ 4x    │
  │ DB queries/req  │ 3            │ 5            │       │
  │ Cache hit ratio │ 85%          │ 75% (cold)   │       │
```

### Traffic patterns

```
PADRÕES COMUNS DE TRÁFEGO:

  1. DIURNAL (mais comum)
     
     QPS
     ▲       ╱╲
     │      ╱  ╲
     │     ╱    ╲
     │    ╱      ╲         ╱╲
     │   ╱        ╲       ╱  ╲
     │  ╱          ╲     ╱    ╲
     │ ╱            ╲   ╱      ╲
     │╱              ╲ ╱        ╲
     ├───────┬────────┬──────────► Hora
     0       12       18        24
     
     Pico: horário comercial (9-18h)
     Trough: madrugada
     Ratio peak/trough: 3-10x

  2. SPIKE EVENT (flash sales, marketing push)
     
     QPS
     ▲          ╱╲
     │         ╱  │
     │        ╱   │
     │       ╱    │
     │      ╱     │
     │─────╱       ╲─────────────
     ├─────┬────────┬────────────► Tempo
           t0     t0+5min
     
     Ramp: segundos a minutos
     Duration: minutos a horas
     Multiplier: 10-100x normal

  3. CRESCIMENTO GRADUAL (organic growth)
     
     QPS
     ▲                            ╱
     │                         ╱╱╱
     │                     ╱╱╱
     │                 ╱╱╱
     │             ╱╱╱
     │         ╱╱╱
     │     ╱╱╱
     │ ╱╱╱
     ├────────────────────────────► Meses
     J  F  M  A  M  J  J  A  S  O
     
     Growth: 5-20% MoM (startups)
     Growth: 2-5% MoM (established)
     Planning horizon: 3-6 meses
```

---

## Load Modeling

### Calculando capacidade necessária

```
LOAD MODEL — TRADUZIR DEMANDA EM RECURSOS:

  PASSO 1: Definir unidade de trabalho
  → "1 request para /checkout requer:"
    - 50ms CPU time
    - 3 DB queries (avg 10ms cada)
    - 1 cache lookup (1ms)
    - 1 external API call (50ms)
    - Total wall time: ~130ms
    - CPU time: ~60ms

  PASSO 2: Calcular capacidade de 1 unidade de compute
  → 1 vCPU com 1 thread = 1000ms/s de CPU time disponível
  → Se cada request usa 60ms CPU:
    throughput_por_vcpu = 1000 / 60 = ~16 req/s/vCPU
  
  → 1 Pod com 2 vCPU e 4 threads:
    throughput_por_pod = 16 × 2 = ~32 req/s/pod
    (limitado por CPU; I/O é parallelizado)

  PASSO 3: Calcular pods necessários para peak
  → Peak: 8,000 req/s
  → Pods necessários = 8,000 / 32 = 250 pods
  
  PASSO 4: Adicionar headroom
  → Utilization target: 70%
  → Pods com headroom = 250 / 0.70 = 358 pods
  → Fault tolerance (N+1 AZ): +33% = 476 pods

CUIDADO COM I/O-BOUND:
  → Se request é 70% I/O wait:
    - Mais threads por pod ajudam (concurrency)
    - Menos CPU por pod needed
    - MAS: connection pools, memory, file descriptors limitam
    - Little's Law: L = λ × W
      - λ = 32 req/s, W = 130ms → L = 4.16 concurrent
      - Preciso ≥ 5 threads/connections por pod
```

### Little's Law para capacity

```
LITTLE'S LAW APLICADA A CAPACITY PLANNING:

  L = λ × W

  L = número de itens no sistema (concurrent requests)
  λ = taxa de chegada (requests/second)
  W = tempo médio no sistema (latency)

  APLICAÇÃO 1: CONNECTION POOL SIZE
  
  Dados:
  → λ = 500 req/s por pod
  → W = 15ms (DB query time)
  → L = 500 × 0.015 = 7.5
  → Pool size mínimo: 8 conexões
  → Com headroom (2x): 16 conexões
  → Com pico (burst 3x): 24 conexões

  APLICAÇÃO 2: THREAD POOL SIZE
  
  Dados:
  → λ = 200 req/s por pod (target)
  → W = 130ms (request total time, wall-clock)
  → L = 200 × 0.130 = 26
  → Thread pool mínimo: 26 threads
  → Com headroom: 40 threads

  APLICAÇÃO 3: QUANTOS PODS/INSTÂNCIAS

  Dados:
  → Total system λ = 10,000 req/s
  → Cada pod suporta 200 req/s (medido, não teórico!)
  → Pods = 10,000 / 200 = 50 pods
  → Com 70% target: 72 pods
  → Com N+1 AZ (3 AZs): 72 × 1.5 = 108 pods
    (perder 1 AZ = perder 1/3, restante aguenta)

  ⚠️  REGRA: O "MEDIDO" DEVE VIR DE LOAD TEST
  → Não use cálculo teórico: MEÇA com load test
  → Use cálculo teórico apenas para estimativa inicial
  → Depois VALIDE com load test realista
```

---

## Forecasting

### Modelos de previsão

```
FORECASTING — PREVER DEMANDA FUTURA:

  1. LINEAR EXTRAPOLATION (simples, bom para 3-6 meses)
     
     load(t+n) = load(t) + growth_rate × n
     
     Exemplo:
     → Hoje: 5,000 QPS peak
     → Growth: +500 QPS/mês
     → Em 6 meses: 5,000 + 500 × 6 = 8,000 QPS
     
     ✅ Simples, bom para crescimento estável
     ❌ Não captura aceleração/sazonalidade

  2. EXPONENTIAL GROWTH (startups)
     
     load(t+n) = load(t) × (1 + rate)^n
     
     Exemplo:
     → Hoje: 1,000 QPS peak  
     → Growth: 15% MoM
     → Em 6 meses: 1,000 × 1.15^6 = 2,313 QPS
     → Em 12 meses: 1,000 × 1.15^12 = 5,350 QPS
     
     ✅ Captura crescimento acelerado
     ❌ Overestimates se growth desacelera

  3. SEASONALITY-ADJUSTED
     
     Exemplo (e-commerce):
     → Baseline: 5,000 QPS
     → Black Friday multiplier: 5x
     → Planning: 25,000 QPS para novembro
     → Janeiro (pós-natal): 0.6x baseline = 3,000 QPS

  │ Mês │ Multiplier │ QPS Peak │ Pods Needed │
  │ Jan │ 0.6        │ 3,000    │ 45          │
  │ Fev │ 0.7        │ 3,500    │ 53          │
  │ Mar │ 0.8        │ 4,000    │ 60          │
  │ ... │            │          │             │
  │ Nov │ 5.0        │ 25,000   │ 375         │
  │ Dez │ 3.0        │ 15,000   │ 225         │
```

### Capacity planning timeline

```
CAPACITY PLANNING HORIZONS:

  ┌──────────────────────────────────────────────────────────┐
  │ Horizonte │ Atividade              │ Precisão │ Frequência│
  │───────────┼────────────────────────┼──────────┼──────────│
  │ Real-time │ Auto-scaling           │ Alta     │ Contínuo  │
  │ 1-7 dias  │ Pre-scaling (events)   │ Alta     │ Sob demand│
  │ 1-3 meses │ Resource procurement   │ Média    │ Mensal    │
  │ 3-6 meses │ Capacity review        │ Média    │ Trimestral│
  │ 6-12 meses│ Budget planning        │ Baixa    │ Semestral │
  │ 1-3 anos  │ Strategy/architecture  │ Baixa    │ Anual     │
  └──────────────────────────────────────────────────────────┘

  ENTRADAS PARA FORECAST:
  
  │ Fonte              │ O que fornece                       │
  │ Product/PM         │ Feature roadmap, user growth target │
  │ Marketing          │ Campanhas, eventos, lançamentos     │
  │ Data/Analytics     │ Trends históricos, seasonality      │
  │ Engineering        │ Eficiência (req/pod), otimizações   │
  │ Finance            │ Budget constraints, cost targets    │
```

---

## Right-Sizing

### Como dimensionar recursos

```
RIGHT-SIZING — NEM OVER NEM UNDER-PROVISIONED:

  UNDER-PROVISIONED:
  → Saturação → degradação → downtime
  → Custo oculto: SLA breach, lost revenue, incident response

  OVER-PROVISIONED:
  → Desperdício de dinheiro
  → "Mas é seguro!" — não: custo é recurso finito também
  → Cloud: over-provisioning = burning cash

  RIGHT-SIZED:
  → Atende peak com headroom de 30%
  → Auto-scales para absorver acima do peak
  → Custo-eficiente

  PROCESSO DE RIGHT-SIZING:

  1. MEASURE ACTUAL USAGE
     → CPU: p95 utilization durante peak
     → Memory: max RSS + 20% headroom
     → Disk: atual + crescimento de 6 meses
     → Network: peak bandwidth + 30%

  2. IDENTIFIQUE WASTE
     → CPU request 4 cores, uso p95 = 0.8 cores → 80% waste
     → Memory request 8Gi, uso max = 3Gi → 62% waste
     → Pods: 10 replicas, load distribui apenas em 6 → 40% waste

  3. RESIZE INCREMENTALMENTE
     → Reduza 20-30% de cada vez (não 80% de uma vez)
     → Monitore p99 latency após cada resize
     → Se p99 degradou → rollback, investigar

  KUBERNETES RESOURCE SIZING:

  resources:
    requests:          # scheduling guarantee
      cpu: "500m"      # p95 usage + 20% headroom
      memory: "1Gi"    # max RSS + 20%
    limits:
      cpu: "2000m"     # burst: 4x request (CPU é compressible)
      memory: "1.5Gi"  # hard limit: request + 50% (OOMKill se exceder)

  REGRAS:
  → CPU request = p95 utilization × 1.2
  → CPU limit = 2-4x request (permite burst)
  → Memory request = max RSS × 1.2
  → Memory limit = request × 1.5
  → Nunca: memory limit >> request (node overcommit → OOM kill)
```

---

## Scaling Strategies

### Vertical vs Horizontal

```
SCALING STRATEGIES:

  VERTICAL SCALING (Scale Up)
  ┌──────────┐      ┌──────────────┐
  │  4 CPU   │  →   │   16 CPU     │
  │  8GB RAM │      │   64GB RAM   │
  │  Small   │      │   X-Large    │
  └──────────┘      └──────────────┘
  
  ✅ Simples (não muda arquitetura)
  ✅ Sem distributed system complexity
  ❌ Limite físico (maior instância disponível)
  ❌ Single point of failure
  ❌ Downtime para resize (geralmente)
  
  Quando usar:
  → Database (vertical + read replicas)
  → Serviço que não pode ser particionado
  → Starter approach antes de precisar horizontal

  HORIZONTAL SCALING (Scale Out)
  ┌──────────┐      ┌────┐ ┌────┐ ┌────┐ ┌────┐
  │  4 CPU   │  →   │ 4C │ │ 4C │ │ 4C │ │ 4C │
  │  8GB RAM │      │ 8G │ │ 8G │ │ 8G │ │ 8G │
  │  1 pod   │      │    │ │    │ │    │ │    │
  └──────────┘      └────┘ └────┘ └────┘ └────┘
  
  ✅ Teoricamente ilimitado
  ✅ Fault tolerant (perder 1 de N)
  ✅ Gradual (add/remove incrementalmente)
  ❌ Complexidade distribuída (state, routing)
  ❌ Nem tudo escala horizontolamente (state!)
  
  Quando usar:
  → Stateless services (APIs, web servers)
  → Read-heavy workloads (read replicas)
  → Queue consumers (parallel processing)
```

### Auto-scaling

```
AUTO-SCALING — REACTIVE E PREDICTIVE:

  1. REACTIVE AUTO-SCALING (HPA — Horizontal Pod Autoscaler)
     
     Triggers:
     → CPU utilization > 70%
     → Memory utilization > 80%
     → Custom metrics (queue depth, request rate)
     
     apiVersion: autoscaling/v2
     kind: HorizontalPodAutoscaler
     spec:
       scaleTargetRef:
         apiVersion: apps/v1
         kind: Deployment
         name: order-service
       minReplicas: 3
       maxReplicas: 50
       metrics:
       - type: Resource
         resource:
           name: cpu
           target:
             type: Utilization
             averageUtilization: 70
       behavior:
         scaleUp:
           stabilizationWindowSeconds: 60
           policies:
           - type: Percent
             value: 100        # pode dobrar
             periodSeconds: 60
         scaleDown:
           stabilizationWindowSeconds: 300  # espera 5min
           policies:
           - type: Percent
             value: 10         # reduz 10% por vez
             periodSeconds: 60

     CUIDADOS COM REACTIVE:
     → Lag: detect (30s) + provision (60s) + ready (30s) = 2min
     → Se spike é de 0 para 10x em 30s: já caiu antes de escalar
     → Solução: pre-scaling ou predictive

  2. PREDICTIVE AUTO-SCALING
     
     → Baseado em padrões históricos
     → "Todo dia às 9h, tráfego sobe 3x"
     → Escala 15 minutos ANTES do pico
     → AWS: Predictive Scaling (Auto Scaling Groups)
     → Keda + CronScaler (Kubernetes)

  3. PRE-SCALING (eventos conhecidos)
     
     → Black Friday: escalar 48h antes
     → Marketing push: escalar 1h antes
     → Requer comunicação com stakeholders
     → Runbook: "escalar order-service para 200 pods"
```

---

## Cloud Capacity Planning

### Cloud-specific considerations

```
CAPACITY PLANNING NA CLOUD:

  ON-PREMISE:
  → Procure hardware → meses de lead time
  → Custo fixo (CapEx)
  → Overprovision by default (não pode escalar rápido)

  CLOUD:
  → Provision em minutos
  → Custo variável (OpEx) — paga por uso
  → Pode right-size continuamente
  → MAS: sem planejamento = bill shock

  CLOUD CAPACITY FRAMEWORK:

  ┌─────────────────────────────────────────────────────────┐
  │ Layer              │ Capacity Consideration             │
  │────────────────────┼────────────────────────────────────│
  │ Compute            │ Instance type, vCPU, pods, HPA     │
  │ Memory             │ Instance type, pod limits           │
  │ Storage            │ IOPS, throughput, capacity (GB/TB) │
  │ Network            │ Bandwidth, cross-AZ, NAT gateway    │
  │ Database           │ Instance class, read replicas, IOPS│
  │ Cache              │ Node type, cluster size, eviction   │
  │ Queue              │ Partitions, throughput, retention   │
  │ Load Balancer      │ CU/LCU capacity, target groups     │
  │ API Gateway        │ Throttling limits, burst            │
  │ Service Limits     │ Account/region quotas               │
  └─────────────────────────────────────────────────────────┘

  AWS SERVICE LIMITS (exemplos comuns):

  │ Resource              │ Default Limit      │ Planning           │
  │ EC2 instances/region  │ Varies by type     │ Request increase   │
  │ ELB targets/group     │ 1,000              │ Sufficient usually │
  │ RDS max connections   │ Based on instance  │ Connection pooling │
  │ DynamoDB WCU          │ On-demand: 40K     │ Request increase   │
  │ Lambda concurrent     │ 1,000/region       │ Reserved concurrency│
  │ S3 requests           │ 5,500 GET/s/prefix │ Partition prefix   │
  │ SQS messages/s        │ Unlimited (std)    │ Partitioning       │
  │ NAT Gateway bandwidth │ 45 Gbps            │ Multiple NATGWs    │
```

### Cost optimization

```
COST-AWARE CAPACITY PLANNING:

  MÉTRICA-CHAVE: COST PER REQUEST (ou por transaction)
  
  cost_per_request = monthly_infra_cost / monthly_requests
  
  Exemplo:
  → Infra: $50,000/mês
  → Requests: 500M/mês
  → Custo: $0.0001/request
  → Se otimizar 2x eficiência: $0.00005/request → $25,000/mês

  ESTRATÉGIAS DE COST REDUCTION:

  1. RIGHT-SIZE (maior impacto)
     → Métricas de uso real → reduzir instances
     → Tool: AWS Compute Optimizer, Kubecost

  2. RESERVAS (committed use)
     → Reserved Instances/Savings Plans: 30-60% desconto
     → Para baseline load (always-on)
     → On-Demand para peak/variable
     
     Padrão:
     ┌──────────────────────────────────────┐
     │                                       │
     │  Peak     ╱╲     ON-DEMAND / SPOT     │
     │          ╱  ╲                          │
     │    ╱╲  ╱    ╲  ╱╲                     │
     │   ╱  ╲╱      ╲╱  ╲                    │
     │──╱──────────────────╲── RESERVED ──── │
     │ ╱                    ╲  (savings plan) │
     │                                       │
     └──────────────────────────────────────┘

  3. SPOT/PREEMPTIBLE (70-90% desconto)
     → Para workloads tolerant a interrupção
     → Batch processing, CI/CD, dev/test
     → Não para latency-sensitive production

  4. AUTO-SCALING (já discutido)
     → Scale down em off-peak
     → Night/weekend scaling policies

  5. ARCHITECTURE EFFICIENCY
     → Cache (evitar recomputation)
     → CDN (offload origin)
     → Compression (menos bandwidth)
     → Async processing (smooth peaks)
```

---

## Capacity Planning por Componente

### Database

```
DATABASE CAPACITY PLANNING:

  MÉTRICAS CRÍTICAS:
  → CPU utilization (target: ≤ 60% peak)
  → Connections (active, idle, max)
  → IOPS (read + write)
  → Disk throughput (MB/s)
  → Replication lag (replicas)
  → Storage growth (GB/mês)
  → Query latency (p50, p99)

  SIZING FRAMEWORK:

  1. CONNECTIONS
     → Max = (num_pods × pool_size_per_pod) + overhead
     → Exemplo: 50 pods × 10 conn/pod = 500 connections
     → RDS: memory-based (t3.large ≈ 700 max)
     → Use PgBouncer/ProxySQL para connection pooling

  2. IOPS
     → Profile queries: IOPS por tipo
     → Read-heavy: read replicas
     → Write-heavy: vertical scale ou sharding
     → gp3: 3,000 base + provision até 16,000
     → io2: provision até 64,000 IOPS

  3. STORAGE
     → Current size + monthly growth × 12
     → Exemplo: 500GB + 20GB/mês × 12 = 740GB
     → Provision 1TB (headroom)
     → Retention: archive old data

  4. READ REPLICAS
     → Read ratio alta (> 80%): read replicas
     → 1 replica por AZ (availability)
     → Scale reads horizontalmente
     → Lag: aceitável para use case?
```

### Cache

```
CACHE CAPACITY PLANNING:

  SIZING:
  → Tamanho = num_keys × avg_value_size × overhead_factor
  → overhead_factor ≈ 1.5-2x (Redis/Memcached metadata)
  
  Exemplo:
  → 10M keys × 2KB avg = 20GB
  → Com overhead: 30-40GB
  → Cluster: 3 nodes × 15GB = 45GB (com redundancy)

  KEY METRICS:
  → Hit ratio (target: > 90%)
  → Memory utilization (target: < 80%)
  → Eviction rate (target: 0 em estado normal)
  → Connection count
  → Latency (p99 < 5ms)

  CAPACITY TRIGGERS:
  → Hit ratio < 85%: aumentar memória (mais keys cabem)
  → Evictions > 0 sustained: memória insuficiente
  → CPU > 70%: cluster precisa mais nodes
  → Latency p99 > 10ms: possível contention, escalar reads
```

### Queue

```
QUEUE/STREAMING CAPACITY PLANNING:

  KAFKA:
  → Partitions = max(throughput / producer_throughput_per_partition,
                     throughput / consumer_throughput_per_partition)
  → Exemplo: 100K msg/s, cada partition suporta 10K msg/s
    → Min 10 partitions
    → Consumer groups: partitions ≥ max_consumers
  
  → Storage = msg_rate × msg_size × retention_period
    → 100K msg/s × 1KB × 7 days = ~60TB
  
  → Brokers: 3 min (replication factor 3)
    → Disk throughput: total_produce_rate / num_brokers
    → Network: replication multiplier (RF × produce rate)

  SQS:
  → Praticamente ilimitado (AWS managed)
  → Capacity planning = consumer throughput
    → Consumers = message_rate × process_time
    → Exemplo: 1000 msg/s × 200ms = 200 concurrent consumers
```

---

## Processo de Capacity Review

```
CAPACITY REVIEW — PROCESSO TRIMESTRAL:

  ┌──────────────────────────────────────────────────────────┐
  │ STEP 1: COLLECT (1 semana antes)                         │
  │ → Métricas de uso (p95 CPU, memory, IOPS) por serviço   │
  │ → Growth trends (MoM) last 3 months                     │
  │ → Incident history (capacity-related)                    │
  │ → Cost per service                                       │
  └──────────────┬───────────────────────────────────────────┘
                 │
  ┌──────────────▼───────────────────────────────────────────┐
  │ STEP 2: ANALYZE                                          │
  │ → Over-provisioned (< 30% utilization)? → right-size     │
  │ → Under-provisioned (> 70% utilization)? → scale up      │
  │ → Growth acceleration? → adjust forecast                 │
  │ → New features coming? → estimate impact                 │
  └──────────────┬───────────────────────────────────────────┘
                 │
  ┌──────────────▼───────────────────────────────────────────┐
  │ STEP 3: PLAN                                             │
  │ → Actions: resize, scale, optimize, reserve              │
  │ → Timeline: immediate, next sprint, next quarter         │
  │ → Budget impact: $X/month change                         │
  │ → Owner per action                                       │
  └──────────────┬───────────────────────────────────────────┘
                 │
  ┌──────────────▼───────────────────────────────────────────┐
  │ STEP 4: EXECUTE & VALIDATE                               │
  │ → Apply changes                                          │
  │ → Monitor: latency, errors, utilization post-change      │
  │ → Confirm savings or capacity gain                       │
  └──────────────────────────────────────────────────────────┘

  CAPACITY REVIEW TEMPLATE:

  │ Serviço        │ CPU p95 │ Mem p95 │ QPS  │ Trend │ Action   │
  │ order-service  │ 68%     │ 55%     │ 2K   │ +10%  │ Monitor  │
  │ payment-svc    │ 25%     │ 40%     │ 500  │ +5%   │ Downsize │
  │ catalog-svc    │ 82%     │ 70%     │ 8K   │ +15%  │ Scale up │
  │ search-svc     │ 45%     │ 85%     │ 3K   │ +20%  │ Mem ↑    │
  │ postgres-main  │ 72%     │ 60%     │ 5K   │ +12%  │ Replica  │
```

---

## Anti-Patterns de Capacity Planning

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **Hope-based capacity** | "Acho que aguenta" sem dados | Medir, calcular, load test |
| **Over-provision everything** | 5x headroom = 5x custo | Right-size baseado em métricas reais |
| **Ignore seasonality** | Black Friday sem planejamento | Mapa de eventos + planning calendar |
| **No load testing** | Capacidade teórica ≠ real | Load test mensal ou pré-release |
| **Plan for average** | Average é 2K QPS, peak é 10K | Sempre planejar para PEAK + headroom |
| **Forget dependencies** | App escala, DB não | Capacity planning end-to-end |
| **Static provisioning** | Same size 24/7 | Auto-scaling + scheduled scaling |
| **Ignore service limits** | Hit AWS quota em pico | Audit quotas trimestralmente |
| **No cost tracking** | Bill cresce sem controle | Cost per request, alerts, reviews |
| **Plan in isolation** | Eng sem input de Product/Mkt | Cross-functional capacity reviews |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código, configurações e infraestrutura, verifique:

1. **Auto-scaling não configurado** — HPA/KEDA deve estar presente para serviços stateless
2. **Resource requests/limits ausentes** — Pods sem requests causam scheduling ineficiente; sem limits = noisy neighbor
3. **Connection pool não dimensionado** — Use Little's Law: L = λ × W para calcular pool mínimo
4. **Cache sem sizing rationale** — Tamanho de cache deve ser justificado com hit ratio target
5. **Database sem read replicas para read-heavy** — Se read:write > 80:20, considere replicas
6. **HPA apenas com CPU** — Considere custom metrics (queue depth, request rate) para scaling mais preciso
7. **Scale-down agressivo** — stabilizationWindowSeconds deve ser ≥ 300s para evitar flapping
8. **Service limits não auditados** — Verificar quotas AWS/GCP/Azure para recursos críticos
9. **Sem monitoramento de custo por serviço** — Tags de cost allocation em todos os recursos
10. **Provisioned IOPS sem justificativa** — gp3 com 3,000 IOPS base é suficiente para a maioria dos casos
11. **Kafka partitions sem cálculo** — Partitions = max consumers desejados (mínimo)
12. **Thread pool com tamanho hardcoded mágico** — Deve ser derivado de Little's Law ou medição

---

## Referências

- **The Art of Capacity Planning** — John Allspaw (O'Reilly, 2008)
- **The Art of Capacity Planning (2nd Ed.)** — Arun Kejariwal & John Allspaw
- **Site Reliability Engineering** — Google (O'Reilly) — Chapter 18: Software Engineering in SRE
- **Little's Law** — John D.C. Little (1961) — L = λW
- **Kubernetes Resource Management** — https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- **AWS Well-Architected: Performance Efficiency** — https://docs.aws.amazon.com/wellarchitected/latest/performance-efficiency-pillar/
- **AWS Compute Optimizer** — https://aws.amazon.com/compute-optimizer/
- **Kubecost** — https://www.kubecost.com/
- **Systems Performance** — Brendan Gregg (Addison-Wesley, 2020)
- **Release It!** — Michael Nygard (Pragmatic Bookshelf, 2018)
