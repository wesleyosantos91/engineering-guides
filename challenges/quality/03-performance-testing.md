# Level 3 — Performance Testing

> **Objetivo:** Implementar testes de performance completos para a Digital Wallet — load, stress,
> spike, soak — definindo SLOs, usando k6/Gatling, analisando resultados e integrando ao CI/CD.

**Referência:** [.docs/QUALITY-ENGINEERING/03-performance-testing.md](../../.docs/QUALITY-ENGINEERING/03-performance-testing.md)

---

## Contexto do Domínio

A Digital Wallet processa transações financeiras que exigem **baixa latência** e **alto throughput**.
Neste nível, você definirá SLOs para a API, implementará testes de performance com diferentes perfis
de carga e analisará gargalos usando métricas e profiling.

---

## Desafios

### Desafio 3.1 — Definição de SLOs e Métricas de Performance

**Contexto:** Antes de testar performance, a equipe precisa definir **o que é "rápido o suficiente"**
para a Digital Wallet. SLOs (Service Level Objectives) são os targets mensuráveis.

**Requisitos:**

- Definir SLOs para cada endpoint da Digital Wallet:

```yaml
slos:
  - name: "Criar Usuário"
    endpoint: "POST /api/v1/users"
    metrics:
      latency_p50: "≤ 50ms"
      latency_p95: "≤ 200ms"
      latency_p99: "≤ 500ms"
      error_rate: "≤ 0.1%"
      throughput: "≥ 500 rps"

  - name: "Depósito"
    endpoint: "POST /api/v1/wallets/{id}/transactions/deposit"
    metrics:
      latency_p50: "≤ 100ms"
      latency_p95: "≤ 300ms"
      latency_p99: "≤ 1s"
      error_rate: "≤ 0.1%"
      throughput: "≥ 200 rps"

  - name: "Transferência"
    endpoint: "POST /api/v1/wallets/{id}/transactions/transfer"
    metrics:
      latency_p50: "≤ 150ms"
      latency_p95: "≤ 500ms"
      latency_p99: "≤ 1.5s"
      error_rate: "≤ 0.1%"
      throughput: "≥ 100 rps"

  - name: "Extrato (listagem)"
    endpoint: "GET /api/v1/wallets/{id}/transactions"
    metrics:
      latency_p50: "≤ 50ms"
      latency_p95: "≤ 200ms"
      latency_p99: "≤ 500ms"
      error_rate: "≤ 0.01%"
      throughput: "≥ 1000 rps"

  - name: "Health Check"
    endpoint: "GET /api/v1/health"
    metrics:
      latency_p99: "≤ 10ms"
      error_rate: "0%"
```

- Definir métricas de infraestrutura:

| Métrica | Target sob carga normal | Target sob stress |
|---------|------------------------|-------------------|
| CPU utilization | ≤ 60% | ≤ 85% |
| Memory utilization | ≤ 70% (estável) | ≤ 85% |
| DB connection pool | ≤ 70% do max | ≤ 90% |
| GC pause time (JVM) | ≤ 20ms P99 | ≤ 50ms P99 |
| Goroutine count (Go) | ≤ 10K | ≤ 50K |

- Documentar **Apdex** target: ≥ 0.9 (satisfying threshold: 200ms)

**Critérios de aceite:**

- [ ] SLOs definidos para todos os endpoints (≥ 5)
- [ ] Métricas de infraestrutura com targets
- [ ] Apdex target definido
- [ ] Documento: `decisions/03-slo-definitions.yml`

---

### Desafio 3.2 — Baseline Test com k6

**Contexto:** Antes de qualquer otimização, é necessário medir o **estado atual** da aplicação
sob condições normais. O baseline serve como referência para comparações futuras.

**Requisitos:**

- Implementar baseline test com k6:
  - 10 VUs (Virtual Users) constantes por 2 minutos
  - Cenário: criar usuário → criar wallet → depositar → consultar saldo
  - Coletar: latência P50/P95/P99, throughput, error rate
- Gerar relatório de baseline:

**k6 Script:**
```javascript
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const depositLatency = new Trend('deposit_latency', true);

export const options = {
    stages: [
        { duration: '30s', target: 10 },   // ramp up
        { duration: '2m', target: 10 },     // steady state
        { duration: '30s', target: 0 },     // ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<300', 'p(99)<1000'],
        errors: ['rate<0.01'],
        deposit_latency: ['p(95)<300'],
    },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080/api/v1';

export default function () {
    group('Baseline: Create → Deposit → Check Balance', () => {
        // 1. Criar usuário
        const userRes = http.post(`${BASE_URL}/users`, JSON.stringify({
            name: `User ${Date.now()}`,
            email: `user-${Date.now()}-${__VU}@test.com`,
            document: generateCPF(),
        }), { headers: { 'Content-Type': 'application/json' } });

        check(userRes, { 'user created (201)': (r) => r.status === 201 });
        errorRate.add(userRes.status !== 201);
        const userId = userRes.json('id');

        // 2. Criar wallet
        const walletRes = http.post(
            `${BASE_URL}/users/${userId}/wallets`,
            JSON.stringify({ currency: 'BRL' }),
            { headers: { 'Content-Type': 'application/json' } }
        );
        check(walletRes, { 'wallet created (201)': (r) => r.status === 201 });
        const walletId = walletRes.json('id');

        // 3. Depositar
        const depositStart = Date.now();
        const depositRes = http.post(
            `${BASE_URL}/wallets/${walletId}/transactions/deposit`,
            JSON.stringify({
                amount: 100.50,
                description: 'Load test deposit',
                idempotency_key: `${Date.now()}-${__VU}-${__ITER}`,
            }),
            { headers: { 'Content-Type': 'application/json' } }
        );
        depositLatency.add(Date.now() - depositStart);
        check(depositRes, { 'deposit success (201)': (r) => r.status === 201 });

        // 4. Verificar saldo
        const balanceRes = http.get(`${BASE_URL}/wallets/${walletId}`);
        check(balanceRes, {
            'balance correct': (r) => r.json('balance') === 100.50,
        });

        sleep(1);
    });
}

function generateCPF() {
    return `${Math.floor(Math.random() * 999999999).toString().padStart(9, '0')}00`;
}
```

- Executar e documentar resultados:

```
k6 run --out json=results/baseline.json load-tests/baseline.js
```

**Critérios de aceite:**

- [ ] Script k6 de baseline funcional
- [ ] Teste roda por pelo menos 2 minutos
- [ ] Métricas coletadas: P50, P95, P99, error rate, throughput
- [ ] Relatório de baseline salvo em `results/baseline.json`
- [ ] Thresholds definidos no script
- [ ] Documento: `results/baseline-report.md`

---

### Desafio 3.3 — Load Test (Carga Normal)

**Contexto:** O load test valida que a Digital Wallet sustenta a **carga esperada** sem degradação.
Simula padrão de uso normal com múltiplos tipos de operação.

**Requisitos:**

- Implementar load test com cenário realista:
  - 50 VUs constantes por 5 minutos
  - Mix de operações (simulando uso real):
    - 40% consultas (GET extrato, GET wallet)
    - 30% depósitos
    - 20% transferências
    - 10% criação de wallet/usuário
  - Validar thresholds contra SLOs definidos

**k6 Script:**
```javascript
export const options = {
    scenarios: {
        read_heavy: {
            executor: 'constant-vus',
            vus: 20,
            duration: '5m',
            exec: 'readOperations',
        },
        deposits: {
            executor: 'constant-vus',
            vus: 15,
            duration: '5m',
            exec: 'depositOperations',
        },
        transfers: {
            executor: 'constant-vus',
            vus: 10,
            duration: '5m',
            exec: 'transferOperations',
        },
        creates: {
            executor: 'constant-vus',
            vus: 5,
            duration: '5m',
            exec: 'createOperations',
        },
    },
    thresholds: {
        'http_req_duration{scenario:read_heavy}': ['p(95)<200'],
        'http_req_duration{scenario:deposits}': ['p(95)<300'],
        'http_req_duration{scenario:transfers}': ['p(95)<500'],
        errors: ['rate<0.001'],  // < 0.1% error rate
    },
};

export function readOperations() {
    // GET extrato paginado, GET wallet
}

export function depositOperations() {
    // POST deposit com dados random
}

export function transferOperations() {
    // POST transfer entre 2 wallets pré-criadas
}

export function createOperations() {
    // POST user + POST wallet
}
```

- Comparar resultados com baseline: P95 não pode degradar mais que 20%

**Critérios de aceite:**

- [ ] Load test com 50 VUs por 5 minutos
- [ ] Mix de operações realista (read/write ratio)
- [ ] Thresholds baseados nos SLOs
- [ ] Comparação com baseline documentada
- [ ] Error rate < 0.1%
- [ ] Relatório: `results/load-test-report.md`

---

### Desafio 3.4 — Stress Test (Encontrar Ponto de Quebra)

**Contexto:** O stress test descobre o **ponto de quebra** da Digital Wallet — a carga
máxima antes que erros ou degradação inaceitável comecem.

**Requisitos:**

- Implementar stress test com rampa progressiva:
  - Fase 1: 10 VUs (warm up) — 1 minuto
  - Fase 2: 50 VUs — 2 minutos
  - Fase 3: 100 VUs — 2 minutos
  - Fase 4: 200 VUs — 2 minutos
  - Fase 5: 500 VUs — 2 minutos (potencial ponto de quebra)
  - Fase 6: Ramp down para 0 — 1 minuto

```javascript
export const options = {
    stages: [
        { duration: '1m', target: 10 },    // warm up
        { duration: '2m', target: 50 },    // normal load
        { duration: '2m', target: 100 },   // above normal
        { duration: '2m', target: 200 },   // stress
        { duration: '2m', target: 500 },   // breaking point?
        { duration: '1m', target: 0 },     // recovery
    ],
    thresholds: {
        http_req_failed: ['rate<0.05'],     // até 5% erro aceitável em stress
        http_req_duration: ['p(99)<3000'],  // P99 < 3s mesmo em stress
    },
};
```

- Identificar e documentar:
  - Em quantos VUs o P95 ultrapassa o SLO?
  - Em quantos VUs o error rate ultrapassa 1%?
  - A aplicação **se recupera** após redução de carga (ramp down)?
  - Qual recurso satura primeiro (CPU, memória, conexões de banco, goroutines/threads)?

**Critérios de aceite:**

- [ ] Stress test com pelo menos 5 estágios de carga
- [ ] Ponto de quebra identificado (VUs onde SLO é violado)
- [ ] Comportamento de recuperação documentado
- [ ] Recurso gargalo identificado (CPU/memória/DB connections)
- [ ] Relatório: `results/stress-test-report.md`

---

### Desafio 3.5 — Spike Test e Soak Test

**Contexto:** Spike test simula picos súbitos (Black Friday, campanha viral). Soak test
detecta memory leaks e degradação temporal.

**Requisitos:**

- **Spike Test:**
  - Carga normal (20 VUs) → pico súbito (200 VUs em 10s) → retorno ao normal → novo pico
  - Validar: tempo de recuperação após spike, error rate durante spike

```javascript
// Spike test
export const options = {
    stages: [
        { duration: '1m', target: 20 },    // normal
        { duration: '10s', target: 200 },   // SPIKE!
        { duration: '1m', target: 20 },     // recovery
        { duration: '10s', target: 200 },   // SPIKE again!
        { duration: '1m', target: 20 },     // recovery
        { duration: '30s', target: 0 },     // cool down
    ],
};
```

- **Soak Test:**
  - Carga moderada (30 VUs) por 30 minutos (ou mais)
  - Monitorar: memory usage (deve ser estável), GC pause time, connection pool, goroutine count
  - Se memória cresce linearmente → **memory leak detectado**

```javascript
// Soak test
export const options = {
    stages: [
        { duration: '2m', target: 30 },     // ramp up
        { duration: '30m', target: 30 },    // steady state - longa duração
        { duration: '2m', target: 0 },      // ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<300'],
        errors: ['rate<0.001'],
    },
};
```

- Documentar:
  - Memory footprint no início vs final do soak
  - GC behavior durante o teste (Java: GC logs, Go: runtime.MemStats)
  - Connection pool utilization timeline

**Critérios de aceite:**

- [ ] Spike test com pelo menos 2 picos
- [ ] Soak test de pelo menos 30 minutos
- [ ] Tempo de recuperação pós-spike medido e documentado
- [ ] Análise de memory leak (se existe ou não)
- [ ] GC behavior documentado
- [ ] Relatório: `results/spike-soak-report.md`

---

### Desafio 3.6 — Profiling e Análise de Gargalos

**Contexto:** Quando testes de performance revelam problemas, é necessário fazer **profiling**
para identificar o gargalo exato (CPU, memória, I/O, queries lentas).

**Requisitos:**

- Implementar profiling para Go:
  - Ativar `pprof` endpoint: `import _ "net/http/pprof"`
  - Coletar CPU profile durante load test: `go tool pprof http://localhost:6060/debug/pprof/profile`
  - Gerar **flame graph** e identificar hotspots
  - Analisar heap profile para detectar alocações excessivas
  - Monitorar goroutine count: `http://localhost:6060/debug/pprof/goroutine`

```go
// main.go — habilitar pprof
import _ "net/http/pprof"

func main() {
    // Separar pprof server do app server
    go func() {
        log.Println(http.ListenAndServe(":6060", nil))
    }()
    // ... app server na porta 8080
}
```

- Implementar profiling para Java:
  - Ativar JFR (Java Flight Recorder) durante load test
  - Analisar com JMC (Java Mission Control) ou VisualVM
  - Monitorar GC com flags: `-Xlog:gc*:file=gc.log:time,uptime,level,tags`
  - Coletar thread dump durante stress test

```bash
# Iniciar app com JFR
java -XX:+FlightRecorder \
     -XX:StartFlightRecording=duration=5m,filename=wallet-perf.jfr \
     -Xlog:gc*:file=gc.log:time,uptime,level,tags \
     -jar wallet-app.jar

# Analisar
jfr print --events jdk.CPULoad wallet-perf.jfr
jfr summary wallet-perf.jfr
```

- Analisar queries lentas:
  - Ativar `pg_stat_statements` no PostgreSQL
  - Identificar as top 5 queries mais lentas durante load test
  - Gerar `EXPLAIN ANALYZE` para cada query lenta

**Critérios de aceite:**

- [ ] pprof habilitado e flame graph gerado (Go)
- [ ] JFR recording gerado e analisado (Java)
- [ ] GC log analisado (pause times, throughput)
- [ ] Top 5 queries lentas identificadas com `EXPLAIN ANALYZE`
- [ ] Pelo menos 1 otimização aplicada baseada no profiling
- [ ] Relatório: `results/profiling-report.md`

---

### Desafio 3.7 — Performance Testing no CI/CD

**Contexto:** Testes de performance devem ser **automatizados** e rodar no pipeline
para detectar regressões antes do deploy.

**Requisitos:**

- Implementar pipeline de performance no CI:

```yaml
# .github/workflows/performance.yml
name: Performance Tests
on:
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 3 * * 1-5'  # Seg-Sex 3AM (nightly)

jobs:
  performance:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: wallet_perf
          POSTGRES_USER: perf
          POSTGRES_PASSWORD: perf
    steps:
      - uses: actions/checkout@v4

      - name: Build application
        run: # build app

      - name: Start application
        run: # start app in background

      - name: Run baseline test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: load-tests/baseline.js
          flags: --out json=results/baseline.json

      - name: Check thresholds
        run: |
          # Comparar P95 com threshold
          P95=$(jq '.metrics.http_req_duration.values.p95' results/baseline.json)
          if (( $(echo "$P95 > 300" | bc -l) )); then
            echo "::error::P95 latency ${P95}ms exceeds 300ms threshold"
            exit 1
          fi

      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: performance-results
          path: results/
```

- Implementar **comparação de regressão**:
  - Salvar baseline do `main` branch
  - Comparar PR results com baseline
  - Alertar se P95 degradar > 20%
  - Alertar se throughput cair > 15%
- Configurar thresholds como **quality gates**:

| Métrica | Gate | Ação |
|---------|------|------|
| P95 latência | > 300ms | ❌ Bloqueia merge |
| P99 latência | > 1000ms | ❌ Bloqueia merge |
| Error rate | > 0.1% | ❌ Bloqueia merge |
| P95 regression | > 20% vs baseline | ⚠️ Warning |
| Throughput regression | > 15% vs baseline | ⚠️ Warning |

**Critérios de aceite:**

- [ ] Pipeline CI com teste de performance automatizado
- [ ] k6 integrado ao GitHub Actions (ou equivalente)
- [ ] Thresholds definidos como quality gates
- [ ] Comparação com baseline implementada
- [ ] Resultados salvos como artifacts
- [ ] Relatório de performance gerado automaticamente
