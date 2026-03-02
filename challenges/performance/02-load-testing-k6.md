# Level 2 — Load Testing com k6

> **Objetivo:** Dominar load testing com k6 (Grafana) — criando scripts JavaScript para todos os tipos de teste (load, stress, soak, spike, breakpoint), com thresholds, custom metrics, workload realista e integração CI/CD.

---

## Objetivo de Aprendizado

- Instalar e configurar k6 para load testing
- Escrever scripts k6 em JavaScript com cenários realistas
- Implementar todos os tipos de load test: load, stress, soak, spike, breakpoint
- Configurar thresholds (pass/fail) e custom metrics
- Modelar workload realista com mix de endpoints e think time
- Usar executors k6: `constant-arrival-rate`, `ramping-arrival-rate`, `ramping-vus`
- Exportar resultados para Grafana/InfluxDB/Prometheus
- Integrar k6 no CI/CD com quality gates de performance
- Entender e evitar coordinated omission

---

## Escopo Funcional

### Endpoints da Digital Wallet API para Load Testing

```
── Workload Mix (proporcional ao tráfego real) ──
GET  /api/v1/wallets/{id}                        → 30% (consulta rápida)
GET  /api/v1/wallets/{id}/transactions?page=0    → 25% (paginação, I/O)
POST /api/v1/wallets/{id}/deposit                → 20% (escrita simples)
POST /api/v1/wallets/{id}/transfer               → 10% (escrita atômica)
POST /api/v1/users                               → 10% (CRUD)
GET  /api/v1/users/{id}                          → 5%  (consulta)

── SLOs (targets de performance) ──
│ Endpoint          │ p95 target │ p99 target │ Error rate │
│ GET  /wallets     │ < 50ms     │ < 200ms    │ < 0.1%     │
│ GET  /transactions│ < 100ms    │ < 500ms    │ < 0.1%     │
│ POST /deposit     │ < 100ms    │ < 300ms    │ < 0.1%     │
│ POST /transfer    │ < 200ms    │ < 500ms    │ < 0.5%     │
│ POST /users       │ < 100ms    │ < 300ms    │ < 0.1%     │
```

---

## Escopo Técnico

### Tipos de Load Test com k6

```
1. LOAD TEST — Carga normal esperada
   Executor: constant-arrival-rate ou ramping-vus
   Duração: 30-60 min steady state
   Purpose: Validar SLOs sob carga esperada

2. STRESS TEST — Além do limite
   Executor: ramping-arrival-rate (crescente)
   Duração: ramps de 5min + holds de 10min
   Purpose: Encontrar breaking point

3. SOAK TEST — Estabilidade prolongada
   Executor: constant-arrival-rate
   Duração: 4-24 horas
   Purpose: Detectar memory leaks, connection leaks

4. SPIKE TEST — Burst súbito
   Executor: ramping-vus (ramp em segundos)
   Duração: spikes de 1-2 min
   Purpose: Testar auto-scaling e recovery

5. BREAKPOINT TEST — Ponto de ruptura
   Executor: ramping-arrival-rate (crescente indefinido)
   Duração: até falha
   Purpose: Determinar capacidade máxima
```

### Executors do k6

| Executor | Controla | Quando usar |
|----------|----------|-------------|
| `shared-iterations` | Total de iterações | Teste rápido, fixo |
| `per-vu-iterations` | Iterações por VU | Garantir N iterações/VU |
| `constant-vus` | VUs constantes | Baseline simples |
| `ramping-vus` | VUs com ramp-up/down | Load test clásico |
| `constant-arrival-rate` | Req/s constante | **Recomendado para load test** |
| `ramping-arrival-rate` | Req/s variável | **Recomendado para stress test** |
| `externally-controlled` | Via API | Teste interativo |

---

## Critérios de Aceite

- [ ] Script de **load test** com workload mix realista (70% read, 30% write) e think time
- [ ] Script de **stress test** com ramp-up progressivo (1x → 2x → 5x → 10x peak)
- [ ] Script de **soak test** com duração de pelo menos 1 hora (detectar leaks)
- [ ] Script de **spike test** com ramp de 0 → 10x em 10 segundos
- [ ] Script de **breakpoint test** com incremento de 10% a cada 5 minutos
- [ ] **Thresholds** configurados para todos os SLOs (p95, p99, error rate)
- [ ] **Custom metrics**: `wallet_deposit_duration`, `wallet_transfer_duration`, `wallet_errors`
- [ ] **Checks** em cada response (status code, body assertions)
- [ ] Resultados exportados para **Grafana** (InfluxDB ou Prometheus Remote Write)
- [ ] Integração **CI/CD**: script que roda k6 e falha se thresholds violados
- [ ] Relatório `K6_LOAD_TEST_REPORT.md` com análise de resultados

---

## Definição de Pronto (DoD)

- [ ] 5 scripts k6 (um por tipo de teste) no diretório `perf/k6/`
- [ ] Docker Compose com k6 + InfluxDB + Grafana para visualização
- [ ] Dashboard Grafana importável com resultados do k6
- [ ] CI pipeline (GitHub Actions) rodando load test com threshold gate
- [ ] Relatório com tabela de resultados: throughput, p50/p95/p99, error rate
- [ ] Commit: `feat(perf-level-2): add k6 load test scripts with CI integration`

---

## Checklist

### Setup k6

- [ ] k6 instalado localmente (`k6 version`)
- [ ] Aplicação rodando em Docker Compose
- [ ] Dados de seed: pelo menos 100 users, 100 wallets com saldo
- [ ] Script de seed para popular dados antes do teste

### Script 1: Load Test

```javascript
// perf/k6/load-test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// Custom metrics
const depositDuration = new Trend('wallet_deposit_duration', true);
const transferDuration = new Trend('wallet_transfer_duration', true);
const errorRate = new Rate('wallet_errors');
const successfulDeposits = new Counter('successful_deposits');

export const options = {
  scenarios: {
    load_test: {
      executor: 'constant-arrival-rate',
      rate: 200,                // 200 req/s
      timeUnit: '1s',
      duration: '30m',
      preAllocatedVUs: 50,
      maxVUs: 200,
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],
    wallet_deposit_duration: ['p(95)<100', 'p(99)<300'],
    wallet_transfer_duration: ['p(95)<200', 'p(99)<500'],
    wallet_errors: ['rate<0.01'],
    http_req_failed: ['rate<0.01'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';
const WALLET_COUNT = parseInt(__ENV.WALLET_COUNT || '100');

function getRandomWalletId() {
  return Math.floor(Math.random() * WALLET_COUNT) + 1;
}

export default function () {
  const rand = Math.random();

  if (rand < 0.30) {
    // 30% — GET wallet
    group('GET wallet', () => {
      const id = getRandomWalletId();
      const res = http.get(`${BASE_URL}/api/v1/wallets/${id}`);
      check(res, { 'wallet 200': (r) => r.status === 200 });
      errorRate.add(res.status !== 200);
    });

  } else if (rand < 0.55) {
    // 25% — GET transactions (paginado)
    group('GET transactions', () => {
      const id = getRandomWalletId();
      const page = Math.floor(Math.random() * 5);
      const res = http.get(`${BASE_URL}/api/v1/wallets/${id}/transactions?page=${page}&size=20`);
      check(res, { 'transactions 200': (r) => r.status === 200 });
      errorRate.add(res.status !== 200);
    });

  } else if (rand < 0.75) {
    // 20% — POST deposit
    group('POST deposit', () => {
      const id = getRandomWalletId();
      const payload = JSON.stringify({
        amount: (Math.random() * 1000 + 1).toFixed(2),
        description: 'k6 load test deposit',
        idempotency_key: `k6-${Date.now()}-${Math.random().toString(36).substr(2)}`,
      });
      const params = { headers: { 'Content-Type': 'application/json' } };
      const start = Date.now();
      const res = http.post(`${BASE_URL}/api/v1/wallets/${id}/transactions/deposit`, payload, params);
      depositDuration.add(Date.now() - start);
      check(res, { 'deposit 201': (r) => r.status === 201 });
      errorRate.add(res.status !== 201);
      if (res.status === 201) successfulDeposits.add(1);
    });

  } else if (rand < 0.85) {
    // 10% — POST transfer
    group('POST transfer', () => {
      const sourceId = getRandomWalletId();
      let targetId = getRandomWalletId();
      while (targetId === sourceId) targetId = getRandomWalletId();

      const payload = JSON.stringify({
        target_wallet_id: targetId.toString(),
        amount: (Math.random() * 100 + 1).toFixed(2),
        description: 'k6 load test transfer',
        idempotency_key: `k6-${Date.now()}-${Math.random().toString(36).substr(2)}`,
      });
      const params = { headers: { 'Content-Type': 'application/json' } };
      const start = Date.now();
      const res = http.post(`${BASE_URL}/api/v1/wallets/${sourceId}/transactions/transfer`, payload, params);
      transferDuration.add(Date.now() - start);
      check(res, { 'transfer 2xx': (r) => r.status >= 200 && r.status < 300 });
      errorRate.add(res.status >= 400);
    });

  } else if (rand < 0.95) {
    // 10% — POST user
    group('POST user', () => {
      const payload = JSON.stringify({
        name: `User k6-${Date.now()}`,
        email: `k6-${Date.now()}-${Math.random().toString(36).substr(2)}@test.com`,
        document: `${Math.floor(Math.random() * 99999999999).toString().padStart(11, '0')}`,
      });
      const params = { headers: { 'Content-Type': 'application/json' } };
      const res = http.post(`${BASE_URL}/api/v1/users`, payload, params);
      check(res, { 'user 201': (r) => r.status === 201 });
      errorRate.add(res.status !== 201);
    });

  } else {
    // 5% — GET user
    group('GET user', () => {
      const id = Math.floor(Math.random() * 100) + 1;
      const res = http.get(`${BASE_URL}/api/v1/users/${id}`);
      check(res, { 'user 200': (r) => r.status === 200 });
      errorRate.add(res.status !== 200);
    });
  }

  // Think time: simula pausa do usuário (1-3s)
  sleep(Math.random() * 2 + 1);
}
```

### Script 2: Stress Test

```javascript
// perf/k6/stress-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    stress: {
      executor: 'ramping-arrival-rate',
      startRate: 50,
      timeUnit: '1s',
      stages: [
        { duration: '5m',  target: 200 },   // 1x peak
        { duration: '10m', target: 200 },   // hold
        { duration: '5m',  target: 400 },   // 2x peak
        { duration: '10m', target: 400 },   // hold
        { duration: '5m',  target: 1000 },  // 5x peak
        { duration: '10m', target: 1000 },  // hold
        { duration: '5m',  target: 2000 },  // 10x peak
        { duration: '10m', target: 2000 },  // hold
        { duration: '5m',  target: 0 },     // ramp down
      ],
      preAllocatedVUs: 100,
      maxVUs: 3000,
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500'],       // relaxed for stress
    http_req_failed: ['rate<0.05'],         // allow 5% errors under extreme load
  },
};

// ... same workload function as load-test.js
```

### Script 3: Soak Test

```javascript
// perf/k6/soak-test.js
export const options = {
  scenarios: {
    soak: {
      executor: 'constant-arrival-rate',
      rate: 100,               // moderate sustained load
      timeUnit: '1s',
      duration: '4h',          // 4 horas
      preAllocatedVUs: 30,
      maxVUs: 150,
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

// OBSERVAR DURANTE O SOAK:
// → Memory cresce sem parar? (memory leak)
// → Connection count cresce? (connection leak)
// → GC pauses aumentam? (GC degradation)
// → Latência aumenta gradualmente? (resource exhaustion)
// → Error rate cresce ao longo do tempo? (file descriptor leak)
```

### Script 4: Spike Test

```javascript
// perf/k6/spike-test.js
export const options = {
  scenarios: {
    spike: {
      executor: 'ramping-vus',
      stages: [
        { duration: '2m', target: 10 },      // baseline
        { duration: '10s', target: 1000 },    // SPIKE! 0→1000 em 10s
        { duration: '2m', target: 1000 },     // hold spike
        { duration: '10s', target: 10 },      // drop back
        { duration: '5m', target: 10 },       // recovery period
        { duration: '10s', target: 1000 },    // second spike
        { duration: '2m', target: 1000 },     // hold
        { duration: '1m', target: 0 },        // ramp down
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<1000'],      // allow higher latency during spike
    http_req_failed: ['rate<0.10'],          // allow 10% during spike
  },
};
```

### Script 5: Breakpoint Test

```javascript
// perf/k6/breakpoint-test.js
export const options = {
  scenarios: {
    breakpoint: {
      executor: 'ramping-arrival-rate',
      startRate: 50,
      timeUnit: '1s',
      stages: [
        // Incrementar 10% a cada 5 minutos até falha
        { duration: '5m', target: 100 },
        { duration: '5m', target: 200 },
        { duration: '5m', target: 400 },
        { duration: '5m', target: 600 },
        { duration: '5m', target: 800 },
        { duration: '5m', target: 1000 },
        { duration: '5m', target: 1500 },
        { duration: '5m', target: 2000 },
        { duration: '5m', target: 3000 },
        { duration: '5m', target: 5000 },
      ],
      preAllocatedVUs: 100,
      maxVUs: 5000,
    },
  },
  thresholds: {
    // Sem threshold de pass/fail — objetivo é encontrar o ponto de ruptura
    // Analisar manualmente: onde throughput satura? Onde error rate sobe?
  },
};

// RESULTADO ESPERADO:
// Encontrar o ponto onde:
// 1. Throughput para de crescer (saturação)
// 2. Latência cresce exponencialmente
// 3. Error rate sobe acima de 1%
// → "O sistema suporta máximo de X req/s com Y pods"
```

### Exportar Resultados para Grafana

```bash
# Via InfluxDB
k6 run --out influxdb=http://localhost:8086/k6 perf/k6/load-test.js

# Via Prometheus Remote Write
k6 run --out experimental-prometheus-rw perf/k6/load-test.js

# Via JSON (para análise posterior)
k6 run --out json=results.json perf/k6/load-test.js

# Via CSV (para planilha)
k6 run --out csv=results.csv perf/k6/load-test.js
```

```yaml
# Docker Compose com InfluxDB para k6
services:
  influxdb:
    image: influxdb:1.8
    ports:
      - "8086:8086"
    environment:
      - INFLUXDB_DB=k6

  grafana:
    image: grafana/grafana:11.0.0
    ports:
      - "3000:3000"
    # Importar dashboard k6: https://grafana.com/grafana/dashboards/2587
```

### CI/CD Integration

```yaml
# .github/workflows/performance.yml
name: Performance Test
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  load-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: wallet
          POSTGRES_PASSWORD: secret
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4
      
      - name: Build and start application
        run: |
          # Build e start da aplicação (adaptar por stack)
          docker compose -f docker-compose.yml up -d --build
          sleep 30  # aguardar warm-up

      - name: Seed test data
        run: |
          # Popular dados para load test
          ./scripts/seed-load-test-data.sh

      - name: Install k6
        run: |
          curl https://github.com/grafana/k6/releases/download/v0.52.0/k6-v0.52.0-linux-amd64.tar.gz -L | tar xz
          sudo mv k6-v0.52.0-linux-amd64/k6 /usr/local/bin/

      - name: Run load test
        run: |
          k6 run \
            -e BASE_URL=http://localhost:8080 \
            -e WALLET_COUNT=100 \
            --out json=results.json \
            perf/k6/load-test.js
        # k6 faz exit code != 0 se thresholds violados

      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: k6-results
          path: results.json

      - name: Comment PR with results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            // parse results.json e postar no PR
            const fs = require('fs');
            // ... summarize and comment
```

---

## Extensões Opcionais

- [ ] Implementar k6 com browser module para testar frontend
- [ ] Usar k6 Cloud para teste distribuído
- [ ] Criar custom k6 extension em Go (`xk6`)
- [ ] Implementar test data management (lifecycle de dados para load test)
- [ ] Criar script de smoke test (validação rápida pré-deploy, 1 min)
- [ ] Implementar correlation entre k6 metrics e traces (traceID no header)
- [ ] Usar `handleSummary()` para gerar relatório HTML customizado

---

## Erros Comuns

| Erro | Consequência | Correção |
|------|-------------|----------|
| Sem think time | Carga irreal: cada VU faz requests sem parar | `sleep(Math.random() * 2 + 1)` entre requests |
| Workload 100% write | Não representa tráfego real (geralmente 80-90% reads) | Mix proporcional: 70-80% reads |
| Sem warmup/ramp-up | Inclui cold start nos resultados | Ramp-up de 2-5 min antes do steady state |
| `shared-iterations` para SLO validation | Não controla req/s, velocidade depende do server | Use `constant-arrival-rate` para req/s fixo |
| Sem thresholds | Teste roda mas nunca "fail" | Definir p95, p99, error rate targets |
| Dados fixos (mesmo wallet ID) | Cache hit 100%, DB hotspot, resultados irreais | IDs aleatórios dentro do dataset |
| Rodar k6 no mesmo host que o servidor | Client-side CPU compete com server | Hosts separados para loader e SUT |
| Sem seed data | API retorna 404 para tudo | Script de seed com dados realistas |

---

## Como Isso Aparece em Entrevistas

- "Como você validaria que um sistema aguenta a carga esperada?"
- "Qual a diferença entre load test, stress test e soak test?"
- "O que é coordinated omission e como o k6 lida com isso?"
- "Como integrar load testing no CI/CD?"
- "O que são thresholds em k6 e como definir SLOs como quality gates?"
- "Descreva um workload model realista para um e-commerce"
- "Como você encontraria a capacidade máxima de um sistema?"
