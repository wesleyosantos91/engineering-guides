# Level 8 — Capstone: Performance Engineering Platform

> **Objetivo:** Integrar TODOS os níveis anteriores (0-7) em uma plataforma completa de performance engineering — profiling contínuo, load tests automatizados, benchmarks no CI, capacity planning em tempo real, SLOs com error budgets e game day exercise — produzindo um relatório executivo de performance.

---

## Objetivo de Aprendizado

- Integrar profiling, load testing, benchmarking, capacity planning e SLOs em um workflow coeso
- Construir pipeline CI/CD com gates de performance em cada estágio
- Implementar continuous profiling em produção (Pyroscope/Parca)
- Automatizar load tests (k6 + Gatling) como parte do ciclo de release
- Criar capacity planning dashboard com projeções em tempo real
- Conduzir game day exercise simulando cenários de degradação de performance
- Produzir relatório executivo com recomendações baseadas em dados

---

## Escopo Funcional

### Digital Wallet — Performance Engineering Completa

```
CENÁRIO FINAL:

A Digital Wallet API vai para PRODUÇÃO com:

  1. ✅ Métricas USE/RED em todos os componentes (Level 0)
  2. ✅ Continuous profiling rodando 24/7 (Level 1)
  3. ✅ Load tests k6 automatizados no CI/CD (Level 2)
  4. ✅ JMeter test plans para cenários enterprise (Level 3)
  5. ✅ Múltiplas ferramentas disponíveis para debugging (Level 4)
  6. ✅ Microbenchmarks com regression gates (Level 5)
  7. ✅ Capacity plan com projeção de 12 meses (Level 6)
  8. ✅ SLOs definidos com error budgets e burn rate alerts (Level 7)
  9. 🎯 TUDO integrado em um workflow operacional (Level 8)

ENTREGÁVEIS DESTE LEVEL:
  → Pipeline CI/CD com performance gates
  → Infra de observabilidade de performance completa
  → Game Day Exercise executado
  → Relatório Executivo de Performance
```

---

## Escopo Técnico

### 1. Arquitetura da Performance Platform

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE ENGINEERING PLATFORM                  │
│                                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐  │
│  │   CODE   │───▶│    CI    │───▶│ STAGING  │───▶│  PRODUCTION  │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────────┘  │
│       │               │               │                │           │
│  ┌────▼────┐    ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼──────┐   │
│  │  JMH/   │    │ Benchmark │   │  k6 Load  │   │ Continuous │   │
│  │  Go     │    │ Regression│   │  Test +   │   │ Profiling  │   │
│  │  Bench  │    │  Gate     │   │  Gatling  │   │ (Pyroscope)│   │
│  └─────────┘    └───────────┘   └───────────┘   └────────────┘   │
│                       │               │                │           │
│                 ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼──────┐   │
│                 │ benchstat │   │ Grafana/  │   │ SLO/Error  │   │
│                 │ Compare   │   │ InfluxDB  │   │ Budget     │   │
│                 └───────────┘   └───────────┘   │ Dashboard  │   │
│                                                  └────────────┘   │
│                                                        │           │
│                                              ┌─────────▼────────┐ │
│                                              │  Capacity Plan   │ │
│                                              │  + Forecasting   │ │
│                                              └──────────────────┘ │
│                                                        │           │
│                                              ┌─────────▼────────┐ │
│                                              │  Alerting:       │ │
│                                              │  Burn Rate +     │ │
│                                              │  Budget Breach   │ │
│                                              └──────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. Pipeline CI/CD com Performance Gates

```yaml
# .github/workflows/performance-pipeline.yml
name: Performance Engineering Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # ── Stage 1: Unit + Microbenchmark (em cada PR) ──
  benchmark-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.23'

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      # Go Benchmarks
      - name: Run Go benchmarks
        run: |
          go test -bench=. -benchmem -count=10 ./... | tee bench_new.txt
          git checkout ${{ github.event.pull_request.base.sha }} -- .
          go test -bench=. -benchmem -count=10 ./... | tee bench_old.txt
          go install golang.org/x/perf/cmd/benchstat@latest
          benchstat bench_old.txt bench_new.txt | tee benchstat.txt

      # JMH Benchmarks
      - name: Run JMH benchmarks
        run: |
          cd benchmark/jmh
          mvn clean package -q
          java -jar target/benchmarks.jar -rf json -rff results.json -wi 3 -i 5 -f 2

      # Gate: falha se regressão > 10%
      - name: Check for regressions
        run: |
          if grep -E '\+[1-9][0-9]\.[0-9]+%' benchstat.txt; then
            echo "❌ BENCHMARK REGRESSION DETECTED"
            exit 1
          fi
          python3 scripts/check_jmh_regression.py \
            --current benchmark/jmh/results.json \
            --baseline benchmark/baseline/jmh-baseline.json \
            --threshold 10

      - name: Post benchmark results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const benchstat = fs.readFileSync('benchstat.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 📊 Benchmark Results\n\`\`\`\n${benchstat}\n\`\`\``
            });

  # ── Stage 2: Load Test (em merge para main) ──
  load-test:
    needs: benchmark-gate
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: wallet
          POSTGRES_USER: wallet
          POSTGRES_PASSWORD: wallet
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4

      - name: Start application
        run: |
          docker compose up -d app
          sleep 10
          curl -f http://localhost:8080/health || exit 1

      - name: Seed test data
        run: |
          ./scripts/seed-load-test-data.sh

      - name: Run k6 load test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: perf/k6/wallet-load-test.js
          flags: --out influxdb=http://localhost:8086/k6

      - name: Run k6 SLO validation
        run: |
          k6 run perf/k6/wallet-slo-check.js \
            --summary-export=k6-summary.json
          
          # Verificar thresholds
          python3 scripts/validate-k6-thresholds.py k6-summary.json

      - name: Archive results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: load-test-results
          path: |
            k6-summary.json
            perf/results/

  # ── Stage 3: Profiling durante load test ──
  profile-under-load:
    needs: benchmark-gate
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start app with profiling
        run: |
          docker compose -f docker-compose.profiling.yml up -d
          sleep 15

      - name: Run load + capture profiles
        run: |
          # Iniciar load test em background
          k6 run perf/k6/wallet-load-test.js --duration=5m &
          K6_PID=$!
          
          # Capturar profiles Go
          sleep 30
          curl -o cpu.prof http://localhost:6060/debug/pprof/profile?seconds=30
          curl -o heap.prof http://localhost:6060/debug/pprof/heap
          curl -o goroutine.prof http://localhost:6060/debug/pprof/goroutine
          
          wait $K6_PID

      - name: Analyze profiles
        run: |
          go tool pprof -top -cum cpu.prof > cpu-top.txt
          go tool pprof -top heap.prof > heap-top.txt
          echo "=== CPU Top Functions ===" && head -30 cpu-top.txt
          echo "=== Heap Top Allocators ===" && head -30 heap-top.txt

      - name: Archive profiles
        uses: actions/upload-artifact@v4
        with:
          name: profiles
          path: "*.prof"
```

### 3. Docker Compose — Infra Completa

```yaml
# docker-compose.performance.yml — Stack completa de performance engineering
services:
  # ── Application ──
  wallet-api:
    build: .
    ports:
      - "8080:8080"
      - "6060:6060"   # pprof (Go) / JFR (Java)
    environment:
      - DATABASE_URL=postgres://wallet:wallet@postgres:5432/wallet
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      - PYROSCOPE_SERVER_ADDRESS=http://pyroscope:4040
    depends_on:
      - postgres
      - otel-collector
      - pyroscope
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 512M

  # ── Database ──
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: wallet
      POSTGRES_USER: wallet
      POSTGRES_PASSWORD: wallet
    ports:
      - "5432:5432"
    volumes:
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
      - pgdata:/var/lib/postgresql/data
    command: >
      postgres
        -c shared_buffers=256MB
        -c max_connections=100
        -c log_min_duration_statement=100

  # ── Observability ──
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.96.0
    volumes:
      - ./otel/collector-config.yaml:/etc/otel/config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Metrics

  prometheus:
    image: prom/prometheus:v2.50.0
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules/:/etc/prometheus/rules/
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:10.3.0
    ports:
      - "3000:3000"
    volumes:
      - ./grafana/provisioning/:/etc/grafana/provisioning/
      - ./grafana/dashboards/:/var/lib/grafana/dashboards/
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

  # ── Continuous Profiling ──
  pyroscope:
    image: grafana/pyroscope:1.4.0
    ports:
      - "4040:4040"

  # ── Load Test Infrastructure ──
  influxdb:
    image: influxdb:1.8
    ports:
      - "8086:8086"
    environment:
      - INFLUXDB_DB=k6
      - INFLUXDB_HTTP_MAX_BODY_SIZE=0

  # ── Tracing ──
  tempo:
    image: grafana/tempo:2.3.0
    ports:
      - "3200:3200"
      - "9411:9411"  # Zipkin
    volumes:
      - ./tempo/tempo.yaml:/etc/tempo.yaml
    command: ["-config.file=/etc/tempo.yaml"]

volumes:
  pgdata:
```

### 4. Grafana Dashboards

```
── Dashboards necessários: ──

1. USE/RED Dashboard (Level 0)
   - Utilization, Saturation, Errors por componente
   - Rate, Error, Duration por endpoint
   - Importar: grafana/dashboards/use-red.json

2. Profiling Dashboard (Level 1)
   - Link para Pyroscope (continuous profiling)
   - Flame graph embed
   - Top CPU/Memory consumers

3. Load Test Results (Level 2-4)
   - k6 results via InfluxDB
   - RPS, latência, error rate, VUs
   - Comparação entre load tests

4. Benchmark Trends (Level 5)
   - Histórico de benchmarks (time series)
   - Regression detection visual

5. Capacity Planning (Level 6)
   - Projeção de throughput
   - Headroom gauge
   - Cost estimation

6. SLO & Error Budget (Level 7)
   - Error budget remaining
   - Burn rate
   - Budget violations table
   - Latency budget breakdown
```

### 5. Game Day Exercise

```
── GAME DAY: Performance Chaos Exercise ──

OBJETIVO:
  Validar que a plataforma de performance é capaz de detectar,
  diagnosticar e responder a problemas de performance em produção.

PRÉ-REQUISITOS:
  - Toda a infra de observabilidade rodando
  - Load test rodando em background (simula tráfego)
  - SLO alerts configurados
  - Error budget dashboards visíveis
  - Equipe com laptops abertos

── CENÁRIO 1: CPU Hotspot (15 min) ──

  INJETAR: Deploy de versão com loop ineficiente na serialização JSON
  
  OBSERVAR:
    ✓ CPU gauge sobe no USE dashboard?
    ✓ Latência p99 sobe no RED dashboard?
    ✓ Burn rate alert disparou?
    ✓ Pyroscope mostra o hotspot na flame graph?
  
  REMEDIAR:
    → Identificar função no flame graph
    → Rollback para versão anterior
    → Verificar que métricas voltam ao normal
  
  CAPTURAR:
    → Screenshot dos dashboards
    → Tempo para detecção (alert fired)
    → Tempo para diagnóstico (root cause)
    → Tempo para remediação (rollback)

── CENÁRIO 2: Memory Leak (20 min) ──

  INJETAR: Deploy com cache que não faz eviction (cresce indefinidamente)
  
  OBSERVAR:
    ✓ Memory gauge cresce linearmente?
    ✓ GC time aumenta?
    ✓ Eventualmente OOM kill?
    ✓ Heap profile mostra o leak?
  
  REMEDIAR:
    → Capturar heap dump
    → Identificar objeto leaking
    → Rollback
  
  CAPTURAR:
    → Heap profile antes e durante leak
    → GC pause time graph

── CENÁRIO 3: Database Saturation (20 min) ──

  INJETAR: N+1 query em endpoint de listagem de transações
  
  OBSERVAR:
    ✓ DB connection pool esgota?
    ✓ Query latência sobe?
    ✓ Connection wait time aparece no APM?
    ✓ Little's Law prediz corretamente?
  
  REMEDIAR:
    → Identificar query no trace (span DB)
    → Fix: eager fetch / batch query
    → Rollback ou hotfix

── CENÁRIO 4: Fan-out Degradation (15 min) ──

  INJETAR: Adicionar sleep de 500ms em 5% dos requests de um serviço downstream
  
  OBSERVAR:
    ✓ p99 do gateway sobe significativamente?
    ✓ Fan-out probability calcula corretamente?
    ✓ Hedged requests mitigam?
  
  REMEDIAR:
    → Ativar hedged requests
    → Adicionar timeout per-component
    → Implementar degradação graceful

── CENÁRIO 5: Capacity Breach (20 min) ──

  INJETAR: Spike de load test (10x carga normal por 5 minutos)
  
  OBSERVAR:
    ✓ HPA escala pods?
    ✓ Tempo de scale-up < 2 min?
    ✓ Error rate durante scale-up?
    ✓ Capacity headroom gauge vai para vermelho?
  
  REMEDIAR:
    → Verificar HPA behavior
    → Ajustar minReplicas se scale-up muito lento
    → Implementar load shedding para picos extremos

── DEBRIEF (30 min) ──

  Para cada cenário, documentar:
  1. Tempo de detecção (MTTD)
  2. Tempo de diagnóstico (MTTI — mean time to identify)
  3. Tempo de remediação (MTTR)
  4. O que funcionou bem nos dashboards/alerts?
  5. O que faltou ou foi difícil de diagnosticar?
  6. Action items para melhorar
```

### 6. Relatório Executivo de Performance

```markdown
# Performance Engineering Report — Digital Wallet API
## Executive Summary
Data: YYYY-MM-DD | Autor: [nome] | Stack: [Go/Spring Boot/...]

### Key Findings

| Métrica | Valor Atual | SLO Target | Status |
|---------|-------------|------------|--------|
| Availability (30d) | 99.97% | 99.9% | ✅ |
| GET /wallets p99 | 42ms | <50ms | ✅ |
| POST /transfer p99 | 380ms | <500ms | ✅ |
| Error rate (30d) | 0.03% | <0.1% | ✅ |
| Error Budget remaining | 72% | >25% | ✅ |

### Capacity Projections

| Timeframe | RPS Peak | Pods | DB Pool | Est. Cost |
|-----------|----------|------|---------|-----------|
| Current | 105 | 2 | 11 | $X/mo |
| +3 months | 182 | 2 | 15 | $Y/mo |
| +6 months | 314 | 3 | 20 | $Z/mo |
| +12 months | 937 | 5 | 35 | $W/mo |

### Performance Risks

1. **Database connection pool** — Current pool_size=20, projecting need for 35 at +12mo
   - Recommendation: Implement read replicas at +6mo
   
2. **Transfer endpoint latency** — p99=380ms vs budget=500ms (24% headroom)
   - Recommendation: Optimize DB queries, add caching for balance reads

3. **Fan-out risk** — Dashboard endpoint fans out to 3 services
   - Recommendation: Implement hedged requests, target P(all<p99) > 99%

### Benchmark Trends (last 30 days)

| Benchmark | Baseline | Current | Delta |
|-----------|----------|---------|-------|
| JSON serialization | 452ns | 445ns | -1.5% |
| Balance calculation (10k) | 125µs | 89µs | -28.8% ✅ |
| Password hashing (bcrypt-12) | 215ms | 218ms | +1.4% |

### Game Day Results

| Scenario | MTTD | MTTI | MTTR |
|----------|------|------|------|
| CPU Hotspot | 2min | 5min | 8min |
| Memory Leak | 5min | 12min | 15min |
| DB Saturation | 1min | 8min | 10min |
| Fan-out Degradation | 3min | 10min | 12min |
| Capacity Breach | 1min | 3min | 5min |

### Recommendations

1. [ ] Add read replica before RPS exceeds 500
2. [ ] Implement hedged requests for fan-out endpoints
3. [ ] Reduce transfer latency with query optimization
4. [ ] Increase benchmark coverage for critical paths
5. [ ] Schedule quarterly game day exercises
```

---

## Critérios de Aceite

- [ ] **Pipeline CI/CD** com 3 stages: benchmark gate, load test, profiling under load
- [ ] **Docker Compose** com stack completa (app, DB, Prometheus, Grafana, Pyroscope, InfluxDB, OTel, Tempo)
- [ ] **6 Grafana dashboards** (USE/RED, profiling, load test, benchmark, capacity, SLO)
- [ ] **Continuous profiling** com Pyroscope rodando e acessível
- [ ] **Game Day Exercise** executado com 5 cenários documentados
- [ ] **Relatório Executivo** com:
  - SLO status (todas as métricas)
  - Capacity projections (3/6/12 meses)
  - Benchmark trends
  - Game Day results (MTTD, MTTI, MTTR)
  - Risk assessment e recomendações
- [ ] **Seed scripts** para popular dados de teste
- [ ] **Runbooks** para cada tipo de problema de performance

---

## Definição de Pronto (DoD)

- [ ] Pipeline CI funcional (benchmark gate bloqueia PR com regressão)
- [ ] Load test executa automaticamente em merge para main
- [ ] Todos os dashboards Grafana importáveis (JSON exportado)
- [ ] Game Day executado, documentado e com action items
- [ ] Relatório Executivo em `PERFORMANCE_REPORT.md`
- [ ] README atualizado com instruções para rodar toda a stack
- [ ] Commit: `feat(perf-level-8): complete performance engineering platform with game day`

---

## Checklist

### Infraestrutura

- [ ] Docker Compose com todos os serviços
- [ ] Prometheus com recording rules e alerting rules
- [ ] Grafana com dashboards provisionados
- [ ] Pyroscope recebendo profiles
- [ ] InfluxDB recebendo métricas do k6
- [ ] OTel Collector recebendo traces

### Pipeline

- [ ] Benchmark gate em PRs (Go benchstat + JMH)
- [ ] Load test em merge para main (k6)
- [ ] Profile capture durante load test
- [ ] Artifact upload (resultados, profiles)
- [ ] PR comments com resultados

### Game Day

- [ ] 5 cenários preparados (scripts de injeção)
- [ ] Dashboards abertos durante exercício
- [ ] Cronograma documentado (cenário, duração, intervalo)
- [ ] Debrief documentado com MTTD/MTTI/MTTR
- [ ] Action items priorizados

### Relatório

- [ ] Executive summary com SLO status
- [ ] Capacity projections com tabela
- [ ] Benchmark trends (últimos 30 dias)
- [ ] Game Day results
- [ ] Risk assessment com severity
- [ ] Recomendações priorizadas

---

## Tarefas Sugeridas por Stack

### Go (Gin)

- [ ] pprof endpoints expostos (`/debug/pprof/`)
- [ ] Pyroscope SDK integrado
- [ ] k6 load tests como primário
- [ ] Vegeta para smoke tests rápidos
- [ ] Go benchmarks com benchstat no CI

### Spring Boot

- [ ] Actuator endpoints + Micrometer
- [ ] JFR continuous recording
- [ ] Pyroscope Java agent
- [ ] Gatling como primary load test
- [ ] JMH benchmarks com regression gate
- [ ] Spring Boot startup time benchmark

### Quarkus

- [ ] Dev Services para desenvolvimento local
- [ ] Native image vs JIT benchmark comparison
- [ ] Quarkus continuous testing integration
- [ ] SmallRye Health para readiness durante scale-up

### Micronaut

- [ ] Micronaut Test para benchmark de startup
- [ ] Compile-time DI benchmark vs Spring runtime DI
- [ ] Netty tuning para throughput máximo

### Jakarta EE

- [ ] Application server tuning (thread pools, connection pools)
- [ ] JFR + JMC analysis
- [ ] EJB vs CDI performance comparison
- [ ] JMeter como primary (ecossistema enterprise)

---

## Extensões Opcionais

- [ ] Chaos Mesh / Litmus para injeção de falhas automatizada
- [ ] Continuous load testing (24/7 com carga baixa para detectar drift)
- [ ] Performance comparison: Go vs Spring Boot vs Quarkus (mesma API)
- [ ] Cost optimization report com cloud pricing
- [ ] Automated remediation: scale up automático baseado em burn rate
- [ ] Performance review como parte do code review checklist
- [ ] Monthly performance retrospective template

---

## Erros Comuns

| Erro | Consequência | Correção |
|------|-------------|----------|
| Game Day sem load de fundo | Problemas de perf não aparecem sem carga | Sempre rodar load test durante game day |
| Pipeline sem benchmark gate | Regressões passam para produção | Benchmark gate obrigatório em PRs |
| Dashboards sem alertas | Ninguém olha dashboards proativamente | Burn rate alerts → Slack/PagerDuty |
| Relatório sem ações | Documento que ninguém usa | Cada finding = recommendation + owner + deadline |
| Game Day sem debrief | Lições não são capturadas | Debrief obrigatório com action items |
| Capacity plan sem revisão | Projeções ficam desatualizadas | Revisão mensal do capacity plan |
| Load test só pré-release | Problemas descobertos tarde | Load test contínuo (canary) |
| Profiling só quando problema acontece | Baseline desconhecido | Continuous profiling 24/7 |

---

## Como Isso Aparece em Entrevistas

- "Descreva um pipeline de CI/CD com gates de performance"
- "Como você implementaria performance engineering em um projeto greenfield?"
- "O que é um game day exercise? Como conduzir um focado em performance?"
- "Como integrar continuous profiling, load testing e SLOs em um workflow coeso?"
- "Produza um relatório executivo de performance para stakeholders não-técnicos"
- "Qual a diferença entre testar performance em CI vs staging vs produção?"
- "Como você mediria o ROI de uma iniciativa de performance engineering?"
- "Descreva um incidente de performance que você diagnosticou e resolveu"
