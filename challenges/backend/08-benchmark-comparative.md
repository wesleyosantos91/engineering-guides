# Level 8 — Benchmark e Análise Comparativa

> **Objetivo:** Executar benchmarks de carga e performance em todas as 5 stacks, comparar resultados quantitativamente e produzir um relatório técnico de portfólio com análise profunda.

---

## Objetivo de Aprendizado

- Executar load testing com k6
- Medir e comparar: latência (p50/p95/p99), throughput (RPS), memory, CPU, startup time
- Analisar GC behavior (Java stacks) vs Go runtime
- Comparar native image (Quarkus/Micronaut) vs JVM (Spring/Jakarta) vs Go
- Produzir relatório técnico com dados, gráficos e conclusões
- Documentar trade-offs reais baseados em evidência
- Criar portfolio piece publicável

---

## Escopo Funcional

### Cenários de Benchmark

```
── Cenário 1: Throughput de Leitura ──
GET /api/v1/users/{id}
  → 100 VUs, 2 minutos, ramp-up 30s

── Cenário 2: Escrita + Leitura Mista ──
POST /api/v1/wallets/{id}/transactions/deposit (70%)
GET  /api/v1/wallets/{id}/transactions (30%)
  → 50 VUs, 3 minutos

── Cenário 3: Pico de Carga ──
POST /api/v1/users (100%)
  → Ramp: 10→200→10 VUs, 5 minutos

── Cenário 4: Transferência Concorrente ──
POST /api/v1/wallets/{id}/transactions/transfer
  → 20 VUs, 2 minutos, mesma wallet de origem (stress no optimistic lock)

── Cenário 5: Startup e First Request ──
Tempo desde `docker run` até primeiro 200 OK em /health/ready
  → 10 iterações, média
```

### Métricas Coletadas

| Métrica | Unidade | Como Coletar |
|---|---|---|
| **Latência p50** | ms | k6 `http_req_duration{p(50)}` |
| **Latência p95** | ms | k6 `http_req_duration{p(95)}` |
| **Latência p99** | ms | k6 `http_req_duration{p(99)}` |
| **Throughput** | req/s | k6 `http_reqs` / duration |
| **Error rate** | % | k6 `http_req_failed` |
| **Memory RSS** | MB | `docker stats` / Prometheus `process_resident_memory_bytes` |
| **CPU usage** | % | `docker stats` / Prometheus `process_cpu_seconds_total` |
| **Startup time** | ms | `time docker run ... --health-cmd` ou log timestamp diff |
| **Docker image size** | MB | `docker images --format '{{.Size}}'` |
| **GC pauses** | ms | JVM: `-Xlog:gc*` / Go: `GODEBUG=gctrace=1` |

---

## Escopo Técnico

### Ferramenta de Load Test: k6

```javascript
// k6/scenarios/read-throughput.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const latency = new Trend('custom_latency', true);

export const options = {
  scenarios: {
    read_throughput: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 100 },  // ramp-up
        { duration: '2m', target: 100 },   // sustain
        { duration: '30s', target: 0 },    // ramp-down
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],
    errors: ['rate<0.01'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';
const TOKEN = __ENV.TOKEN;

export function setup() {
  // Seed de dados: criar 100 usuários + wallets para leitura
  const users = [];
  for (let i = 0; i < 100; i++) {
    const res = http.post(`${BASE_URL}/api/v1/users`, JSON.stringify({
      name: `User ${i}`,
      email: `user${i}@bench.com`,
      document: `000.000.000-${String(i).padStart(2, '0')}`,
    }), {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${TOKEN}`,
      },
    });
    if (res.status === 201) {
      users.push(JSON.parse(res.body).id);
    }
  }
  return { userIds: users };
}

export default function (data) {
  const userId = data.userIds[Math.floor(Math.random() * data.userIds.length)];

  const res = http.get(`${BASE_URL}/api/v1/users/${userId}`, {
    headers: { 'Authorization': `Bearer ${TOKEN}` },
  });

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response has id': (r) => JSON.parse(r.body).id !== undefined,
  });

  errorRate.add(res.status !== 200);
  latency.add(res.timings.duration);

  sleep(0.1);
}
```

### Scripts k6 por Cenário

| Script | Arquivo |
|---|---|
| Read Throughput | `k6/scenarios/read-throughput.js` |
| Mixed Workload | `k6/scenarios/mixed-workload.js` |
| Spike Test | `k6/scenarios/spike-test.js` |
| Concurrent Transfer | `k6/scenarios/concurrent-transfer.js` |
| Startup Time | `k6/scenarios/startup-time.sh` (shell script) |

### Ambiente de Benchmark

```yaml
# Todos os testes rodados no MESMO hardware/VM:
# - Docker Desktop / Podman com limites definidos
# - CPU: 2 cores por container
# - Memory: 512 MB por container
# - PostgreSQL: compartilhado (mesma instância)
# - Kafka: compartilhado
# - OS: mesma máquina

services:
  api-go:
    build: ./go-wallet
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 512M
    ports: ["8081:8080"]

  api-spring:
    build: ./spring-wallet
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 512M
    ports: ["8082:8080"]

  api-quarkus-native:
    build:
      context: ./quarkus-wallet
      dockerfile: Dockerfile.native
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 512M
    ports: ["8083:8080"]

  api-micronaut-native:
    build:
      context: ./micronaut-wallet
      dockerfile: Dockerfile.native
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 512M
    ports: ["8084:8080"]

  api-jakarta:
    build: ./jakarta-wallet
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 512M
    ports: ["8085:8080"]
```

---

## Critérios de Aceite

- [ ] k6 scripts rodam com sucesso contra todas as 5 stacks
- [ ] Resultados de cada cenário coletados em JSON/CSV
- [ ] Tabela comparativa com métricas de todas as stacks
- [ ] Gráficos de latência (histograma) para cada stack
- [ ] Gráfico de throughput ao longo do tempo para cada stack
- [ ] Startup time medido (10 iterações, média + desvio padrão)
- [ ] Memory usage durante teste documentado
- [ ] Docker image size comparado
- [ ] Relatório técnico ≥ 2000 palavras com análise e conclusões
- [ ] Nenhuma stack com error rate > 1% nos cenários normais

---

## Definição de Pronto (DoD)

- [ ] 5 scripts k6 funcionais
- [ ] Shell script `run-all-benchmarks.sh` que executa tudo automaticamente
- [ ] Resultados salvos em `results/` por stack e cenário
- [ ] Relatório em Markdown: `BENCHMARK-REPORT.md`
- [ ] Gráficos gerados (pode ser Excel, Python matplotlib, ou k6 cloud)
- [ ] Análise de trade-offs: quando usar cada stack
- [ ] Publicável como artigo técnico ou post de blog
- [ ] Commit: `feat(level-8): add comprehensive benchmark report across all 5 stacks`

---

## Checklist

### Scripts de Benchmark

- [ ] `k6/scenarios/read-throughput.js`
- [ ] `k6/scenarios/mixed-workload.js`
- [ ] `k6/scenarios/spike-test.js`
- [ ] `k6/scenarios/concurrent-transfer.js`
- [ ] `k6/scripts/startup-time.sh`
- [ ] `k6/scripts/collect-docker-stats.sh` — coletar CPU/memory via `docker stats`
- [ ] `k6/scripts/run-all.sh` — orquestrador de todos os testes
- [ ] `k6/scripts/generate-report.py` — gerar tabelas/gráficos (opcional)

### Infraestrutura

- [ ] `docker-compose.benchmark.yml` — todas as stacks com resource limits
- [ ] Scripts de seed para dados iniciais
- [ ] `Makefile` — `benchmark-go`, `benchmark-spring`, `benchmark-all`

### Relatório

- [ ] `BENCHMARK-REPORT.md` — relatório completo
- [ ] Seção: Metodologia (hardware, limites, cenários, repetições)
- [ ] Seção: Resultados (tabelas, gráficos)
- [ ] Seção: Análise (por que X é mais rápido em Y cenário?)
- [ ] Seção: Trade-offs (quando usar cada stack)
- [ ] Seção: Conclusões e recomendações
- [ ] Seção: Limitações da análise

---

## Template de Resultado Esperado

### Tabela Comparativa (exemplo fictício)

| Métrica | Go (Gin) | Spring Boot | Quarkus Native | Micronaut Native | Jakarta EE |
|---|---|---|---|---|---|
| **Startup** | 50ms | 3.2s (CDS: 1.8s) | 80ms | 90ms | 8.5s |
| **Image Size** | 18 MB | 180 MB | 65 MB | 75 MB | 450 MB |
| **Memory Idle** | 12 MB | 180 MB | 25 MB | 35 MB | 250 MB |
| **Memory Load** | 45 MB | 350 MB | 80 MB | 95 MB | 400 MB |
| **RPS (read)** | 15,200 | 8,500 | 12,000 | 11,500 | 5,200 |
| **p50 (read)** | 3ms | 8ms | 5ms | 6ms | 12ms |
| **p95 (read)** | 12ms | 35ms | 18ms | 20ms | 55ms |
| **p99 (read)** | 45ms | 120ms | 65ms | 70ms | 180ms |
| **RPS (mixed)** | 8,500 | 5,200 | 7,100 | 6,800 | 3,100 |
| **Error rate** | 0.02% | 0.05% | 0.03% | 0.04% | 0.08% |

> ⚠️ Valores fictícios — os seus resultados reais variarão conforme hardware e configuração.

### Análise Esperada

```markdown
## Análise

### Startup
Go e Quarkus native dominam o startup, ambos abaixo de 100ms...

### Throughput de Leitura
Go apresenta o maior throughput (~15K RPS) devido ao modelo de goroutines...
Spring Boot com virtual threads fecha a distância significativamente...

### Workload Misto (escrita + leitura)
A diferença entre stacks diminui em cenários de escrita, pois o bottleneck
se desloca para o banco de dados...

### Memory Footprint
Go tem o menor consumo de memória tanto em idle quanto sob carga...
Native images (Quarkus/Micronaut) ficam entre Go e JVM...

### Quando Usar Cada Stack
- **Go (Gin)**: Microserviços com alta demanda de throughput, low-latency, memory-constrained
- **Spring Boot**: Enterprise apps com ecossistema rico, equipe Java sênior, muitas integrações
- **Quarkus Native**: Serverless/FaaS, startup rápido, cloud-native com equipe Java
- **Micronaut Native**: Similar ao Quarkus, preferência por compile-time DI e type-safety
- **Jakarta EE**: Sistemas legados, portabilidade entre app servers, padrões corporativos
```

---

## Extensões Opcionais

- [ ] Adicionar benchmark de Quarkus JVM mode vs native mode
- [ ] Comparar Spring Boot com e sem CDS
- [ ] Testar com GraalVM JIT (não apenas native) para Spring Boot
- [ ] Adicionar benchmark de startup cold vs warm (JVM warmup)
- [ ] Comparar com virtual threads (Java 25) habilitados vs desabilitados
- [ ] Publicar resultados no GitHub Pages como site estático
- [ ] Adicionar benchmark de consumo Kafka (mensagens/segundo por consumer)
- [ ] Testar com diferentes GC tunings (ZGC, Shenandoah, G1)
- [ ] Comparar build time (compile time) entre stacks

---

## Script de Benchmark Automatizado

```bash
#!/bin/bash
# run-all-benchmarks.sh

set -euo pipefail

RESULTS_DIR="results/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$RESULTS_DIR"

STACKS=("go:8081" "spring:8082" "quarkus:8083" "micronaut:8084" "jakarta:8085")
SCENARIOS=("read-throughput" "mixed-workload" "spike-test" "concurrent-transfer")

echo "=== Starting benchmark suite ==="

# 1. Build & start all stacks
docker-compose -f docker-compose.benchmark.yml up -d --build
echo "Waiting for all stacks to be healthy..."
sleep 30

# 2. Measure startup times
echo "=== Startup Times ==="
for stack_port in "${STACKS[@]}"; do
  IFS=':' read -r stack port <<< "$stack_port"
  echo "Measuring startup for $stack..."

  TOTAL=0
  for i in $(seq 1 10); do
    docker-compose -f docker-compose.benchmark.yml restart "api-$stack"
    START=$(date +%s%N)
    until curl -sf "http://localhost:$port/health/ready" > /dev/null 2>&1; do
      sleep 0.1
    done
    END=$(date +%s%N)
    ELAPSED=$(( (END - START) / 1000000 ))
    TOTAL=$((TOTAL + ELAPSED))
    echo "  Run $i: ${ELAPSED}ms"
  done
  AVG=$((TOTAL / 10))
  echo "$stack: avg startup = ${AVG}ms" >> "$RESULTS_DIR/startup-times.txt"
done

# 3. Run k6 scenarios
for stack_port in "${STACKS[@]}"; do
  IFS=':' read -r stack port <<< "$stack_port"
  echo "=== Running benchmarks for $stack ==="

  for scenario in "${SCENARIOS[@]}"; do
    echo "  Scenario: $scenario"
    k6 run \
      --out json="$RESULTS_DIR/${stack}_${scenario}.json" \
      --summary-export="$RESULTS_DIR/${stack}_${scenario}_summary.json" \
      -e "BASE_URL=http://localhost:$port" \
      -e "TOKEN=$(cat .token)" \
      "k6/scenarios/${scenario}.js"
    echo "  Done."
    sleep 10  # cooldown entre cenários
  done
done

# 4. Collect Docker stats
echo "=== Docker Stats ==="
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}" \
  > "$RESULTS_DIR/docker-stats.txt"

# 5. Image sizes
echo "=== Image Sizes ==="
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | grep wallet \
  > "$RESULTS_DIR/image-sizes.txt"

echo "=== Benchmark complete ==="
echo "Results saved to $RESULTS_DIR/"
```

---

## Erros Comuns

| Erro | Como evitar |
|---|---|
| Comparar sem mesmos resource limits | Docker `deploy.resources.limits` iguais para todos |
| Não aquecer a JVM antes de medir | Ramp-up de 30s mínimo; ignorar primeiros 30s de dados |
| Banco como bottleneck → todos iguais | Verificar se PostgreSQL está limitando; considerar pool size |
| Rodar k6 na mesma máquina → interferência | Idealmente k6 em máquina separada; se não, documentar |
| Comparar native build sem incluir build time | Native build pode levar 5-10min — documentar trade-off |
| Poucos dados → conclusões precipitadas | ≥ 3 rodadas de cada cenário; calcular desvio padrão |
| Não documentar versões | Registrar: Go version, Java version, k6 version, Docker version, kernel |

---

## Como Isso Aparece em Entrevistas

- "Como você faria um benchmark justo entre Go e Java para uma API REST?"
- "Quais métricas são mais importantes em um load test?"
- "O que é o percentil 99 e por que é mais importante que a média?"
- "Qual a diferença de performance entre JVM e native image?"
- "Quando o startup time importa e quando não importa?"
- "Como o modelo de concorrência de Go difere das virtual threads do Java?"
- "O que é tail latency e como reduzir?"
- "Como você apresentaria resultados de benchmark para uma decisão de tecnologia?"
- "Quais são os trade-offs entre throughput e latência?"
- "Se todos tivessem throughput similar, como você escolheria a stack?"

---

## Comandos de Execução

```bash
# Instalar k6
# macOS: brew install k6
# Windows: choco install k6
# Linux: snap install k6

# Build todas as stacks
docker-compose -f docker-compose.benchmark.yml build

# Subir todas
docker-compose -f docker-compose.benchmark.yml up -d

# Rodar benchmark individual
k6 run -e BASE_URL=http://localhost:8081 k6/scenarios/read-throughput.js

# Rodar todos os benchmarks
./k6/scripts/run-all.sh

# Ver resultados
cat results/latest/startup-times.txt
cat results/latest/docker-stats.txt

# Gerar gráficos (se tiver script Python)
python3 k6/scripts/generate-report.py results/latest/

# Cleanup
docker-compose -f docker-compose.benchmark.yml down -v
```

---

## Portfolio Output

Ao concluir este level, você terá um artefato publicável que demonstra:

1. **Proficiência em 5 stacks backend** — mesmo domínio, implementação idiomática
2. **Capacidade de análise técnica** — benchmark com metodologia e dados
3. **Visão arquitetural** — sabe quando usar cada tecnologia
4. **Habilidade de comunicação** — relatório técnico publicável
5. **Mentalidade de engenharia** — decisões baseadas em evidência, não opinião

### Formato sugerido para publicação:

- GitHub: monorepo com 5 projetos + results + report
- Blog post (Dev.to / Medium): resumo de 2000 palavras com tabelas e gráficos
- LinkedIn: post curto com 3 insights chave + link para GitHub
- Apresentação: slide deck de 15-20 slides para meetup/conferência
