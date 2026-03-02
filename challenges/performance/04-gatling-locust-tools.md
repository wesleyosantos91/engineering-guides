# Level 4 — Load Testing: Gatling, Locust e Ferramentas Complementares

> **Objetivo:** Dominar o ecossistema completo de ferramentas de load testing — Gatling (Scala/Java DSL), Locust (Python), wrk2, Artillery e Vegeta — compreendendo coordinated omission, escolha de ferramentas e comparação prática.

---

## Objetivo de Aprendizado

- Criar simulações Gatling com Java DSL para cenários complexos
- Implementar load tests com Locust usando Python para cenários programáticos
- Usar wrk2 para benchmark rápido com correção de coordinated omission
- Configurar Artillery (YAML) para testes declarativos
- Usar Vegeta para load testing em Go
- Entender coordinated omission e como cada ferramenta lida com isso
- Comparar ferramentas e escolher a certa para cada cenário
- Integrar cada ferramenta com CI/CD e dashboards

---

## Escopo Funcional

### API sob teste — Digital Wallet

```
Endpoints (mesmos em todas as ferramentas):

GET  /api/v1/wallets/{id}                    → Consultar saldo
GET  /api/v1/wallets/{id}/transactions       → Listar transações (paginado)
POST /api/v1/wallets/{id}/transactions/deposit   → Depósito
POST /api/v1/wallets/{id}/transactions/transfer  → Transferência (2 wallets)
POST /api/v1/users                           → Criar usuário
GET  /api/v1/users/{id}                      → Consultar perfil

SLOs:
  GET  /wallets/{id}          → p95 < 50ms,  p99 < 200ms
  GET  /wallets/{id}/txns     → p95 < 100ms, p99 < 300ms
  POST /wallets/{id}/deposit  → p95 < 200ms, p99 < 500ms
  POST /wallets/{id}/transfer → p95 < 200ms, p99 < 500ms
  Error rate                  → < 0.1%
```

---

## Escopo Técnico

### 1. Gatling (Java DSL)

```java
// WalletSimulation.java — Gatling 3.10+ (Java DSL)
import io.gatling.javaapi.core.*;
import io.gatling.javaapi.http.*;
import static io.gatling.javaapi.core.CoreDsl.*;
import static io.gatling.javaapi.http.HttpDsl.*;

public class WalletSimulation extends Simulation {

    HttpProtocolBuilder httpProtocol = http
        .baseUrl("http://localhost:8080/api/v1")
        .acceptHeader("application/json")
        .contentTypeHeader("application/json")
        .shareConnections();

    // Feeder: dados parametrizados
    FeederBuilder<String> walletFeeder = csv("wallet_ids.csv").circular();

    // Cenários
    ScenarioBuilder readScenario = scenario("Read Operations")
        .feed(walletFeeder)
        .exec(
            http("Get Wallet")
                .get("/wallets/#{walletId}")
                .check(status().is(200))
                .check(jsonPath("$.balance").saveAs("currentBalance"))
        )
        .pause(1, 3)
        .exec(
            http("Get Transactions")
                .get("/wallets/#{walletId}/transactions?page=0&size=20")
                .check(status().is(200))
                .check(jsonPath("$.content").exists())
        )
        .pause(1, 3);

    ScenarioBuilder writeScenario = scenario("Write Operations")
        .feed(walletFeeder)
        .exec(
            http("Deposit")
                .post("/wallets/#{walletId}/transactions/deposit")
                .body(StringBody("""
                    {
                        "amount": #{amount},
                        "description": "Load test deposit",
                        "idempotency_key": "#{randomUuid}"
                    }
                    """))
                .check(status().is(201))
                .check(jsonPath("$.id").exists())
        )
        .pause(2, 5);

    ScenarioBuilder correlationScenario = scenario("Full User Journey")
        .exec(
            http("Create User")
                .post("/users")
                .body(StringBody("""
                    {"name": "User #{counter}", "email": "user#{counter}@loadtest.com"}
                    """))
                .check(status().is(201))
                .check(jsonPath("$.id").saveAs("userId"))
        )
        .pause(1)
        .exec(
            http("Get Created User")
                .get("/users/#{userId}")
                .check(status().is(200))
        );

    // Perfil de carga
    {
        setUp(
            readScenario.injectOpen(
                rampUsersPerSec(10).to(100).during(300),  // 5 min ramp
                constantUsersPerSec(100).during(1800)     // 30 min hold
            ),
            writeScenario.injectOpen(
                rampUsersPerSec(5).to(30).during(300),
                constantUsersPerSec(30).during(1800)
            )
        ).protocols(httpProtocol)
         .assertions(
            global().responseTime().percentile3().lt(200),  // p95 < 200ms
            global().responseTime().percentile4().lt(500),  // p99 < 500ms
            global().successfulRequests().percent().gt(99.9) // error < 0.1%
         );
    }
}
```

```
── Perfis de injeção do Gatling ──

OPEN MODEL (mais realista):
  rampUsersPerSec(10).to(100).during(300)  → ramp de 10 a 100 users/sec em 5min
  constantUsersPerSec(100).during(1800)    → 100 users/sec por 30 min
  stressPeakUsers(1000).during(30)         → spike de 1000 users em 30s
  nothingFor(60)                           → pausa de 1 min

CLOSED MODEL (pool fixo de VUs):
  constantConcurrentUsers(100).during(1800)
  rampConcurrentUsers(10).to(500).during(600)

⚠️  OPEN model é preferível — não sofre de coordinated omission
```

### 2. Locust (Python)

```python
# locustfile.py — Locust load test para Digital Wallet
from locust import HttpUser, task, between, tag, events
import random
import uuid
import json
import logging

class WalletUser(HttpUser):
    """Simula um usuário da Digital Wallet API."""
    
    wait_time = between(1, 3)  # Think time: 1-3 segundos
    
    # Dados compartilhados
    wallet_ids = list(range(1, 101))  # IDs de 1 a 100
    
    def on_start(self):
        """Setup inicial por VU — executado ao criar cada user."""
        self.wallet_id = random.choice(self.wallet_ids)
        self.user_id = None
    
    @tag("read")
    @task(30)  # 30% do peso total
    def get_wallet(self):
        """Consultar saldo da wallet."""
        with self.client.get(
            f"/api/v1/wallets/{self.wallet_id}",
            name="/api/v1/wallets/[id]",  # agrupar por padrão, não por ID
            catch_response=True
        ) as response:
            if response.status_code == 200:
                data = response.json()
                if "balance" not in data:
                    response.failure("Missing balance field")
            else:
                response.failure(f"Status {response.status_code}")
    
    @tag("read")
    @task(25)
    def get_transactions(self):
        """Listar transações da wallet."""
        page = random.randint(0, 5)
        with self.client.get(
            f"/api/v1/wallets/{self.wallet_id}/transactions?page={page}&size=20",
            name="/api/v1/wallets/[id]/transactions",
            catch_response=True
        ) as response:
            if response.status_code != 200:
                response.failure(f"Status {response.status_code}")
    
    @tag("write")
    @task(20)
    def deposit(self):
        """Realizar depósito."""
        amount = round(random.uniform(10, 1000), 2)
        payload = {
            "amount": amount,
            "description": "Load test deposit",
            "idempotency_key": str(uuid.uuid4())
        }
        with self.client.post(
            f"/api/v1/wallets/{self.wallet_id}/transactions/deposit",
            json=payload,
            name="/api/v1/wallets/[id]/transactions/deposit",
            catch_response=True
        ) as response:
            if response.status_code not in (200, 201):
                response.failure(f"Status {response.status_code}")
    
    @tag("write")
    @task(10)
    def transfer(self):
        """Realizar transferência entre wallets."""
        target = random.choice([w for w in self.wallet_ids if w != self.wallet_id])
        payload = {
            "amount": round(random.uniform(1, 100), 2),
            "target_wallet_id": target,
            "description": "Load test transfer",
            "idempotency_key": str(uuid.uuid4())
        }
        with self.client.post(
            f"/api/v1/wallets/{self.wallet_id}/transactions/transfer",
            json=payload,
            name="/api/v1/wallets/[id]/transactions/transfer",
            catch_response=True
        ) as response:
            if response.status_code not in (200, 201):
                response.failure(f"Status {response.status_code}")
    
    @tag("write")
    @task(10)
    def create_user(self):
        """Criar novo usuário."""
        payload = {
            "name": f"Locust User {uuid.uuid4().hex[:8]}",
            "email": f"locust_{uuid.uuid4().hex[:8]}@test.com"
        }
        with self.client.post(
            "/api/v1/users",
            json=payload,
            name="/api/v1/users",
            catch_response=True
        ) as response:
            if response.status_code == 201:
                self.user_id = response.json().get("id")
            else:
                response.failure(f"Status {response.status_code}")
    
    @tag("read")
    @task(5)
    def get_user(self):
        """Consultar perfil do usuário."""
        uid = self.user_id or random.randint(1, 100)
        self.client.get(
            f"/api/v1/users/{uid}",
            name="/api/v1/users/[id]"
        )


# Custom event: exportar métricas para CSV ao final
@events.quitting.add_listener
def on_quitting(environment, **kwargs):
    stats = environment.runner.stats
    logging.info(f"Total requests: {stats.total.num_requests}")
    logging.info(f"Failure rate: {stats.total.fail_ratio:.2%}")
    logging.info(f"Avg response time: {stats.total.avg_response_time:.0f}ms")
    logging.info(f"p95: {stats.total.get_response_time_percentile(0.95):.0f}ms")
    logging.info(f"p99: {stats.total.get_response_time_percentile(0.99):.0f}ms")
```

```bash
# Execução Locust

# Modo GUI (browser em http://localhost:8089)
locust -f locustfile.py --host=http://localhost:8080

# Modo headless (CLI — para CI/CD)
locust -f locustfile.py \
  --host=http://localhost:8080 \
  --headless \
  --users 200 \
  --spawn-rate 10 \
  --run-time 30m \
  --csv=results/locust \
  --html=results/report.html \
  --tags read write

# Distributed (master + workers)
locust -f locustfile.py --master --host=http://localhost:8080
locust -f locustfile.py --worker --master-host=192.168.1.100
```

### 3. wrk2 (Latência precisa)

```bash
# wrk2 — corrige coordinated omission nativamente

# Benchmark rápido: 200 req/s por 5 minutos
wrk2 -t4 -c100 -d300s -R200 --latency http://localhost:8080/api/v1/wallets/1

# Com script Lua para POST + body
wrk2 -t4 -c100 -d300s -R100 --latency \
  -s deposit.lua http://localhost:8080/api/v1/wallets/1/transactions/deposit
```

```lua
-- deposit.lua — wrk2 script para POST com body dinâmico
local counter = 0

request = function()
    counter = counter + 1
    local body = string.format(
        '{"amount":%.2f,"description":"wrk2 test","idempotency_key":"wrk2-%d-%d"}',
        math.random(10, 1000) + math.random(),
        os.time(),
        counter
    )
    return wrk.format("POST", nil, {
        ["Content-Type"] = "application/json"
    }, body)
end

response = function(status, headers, body)
    if status ~= 201 then
        wrk.thread:set("errors", (wrk.thread:get("errors") or 0) + 1)
    end
end

done = function(summary, latency, requests)
    io.write("------------------------------\n")
    io.write(string.format("Total requests: %d\n", summary.requests))
    io.write(string.format("Duration: %.2fs\n", summary.duration / 1e6))
    io.write(string.format("Requests/sec: %.2f\n", summary.requests / (summary.duration / 1e6)))
    io.write(string.format("Avg latency: %.2fms\n", latency.mean / 1000))
    io.write(string.format("p50: %.2fms\n", latency:percentile(50) / 1000))
    io.write(string.format("p90: %.2fms\n", latency:percentile(90) / 1000))
    io.write(string.format("p99: %.2fms\n", latency:percentile(99) / 1000))
    io.write(string.format("p99.9: %.2fms\n", latency:percentile(99.9) / 1000))
    io.write("------------------------------\n")
end
```

### 4. Artillery (YAML declarativo)

```yaml
# artillery-config.yml — Load test declarativo
config:
  target: "http://localhost:8080"
  phases:
    - name: "Warm up"
      duration: 60
      arrivalRate: 10
    - name: "Ramp up"
      duration: 300
      arrivalRate: 10
      rampTo: 100
    - name: "Sustained load"
      duration: 1800
      arrivalRate: 100
    - name: "Ramp down"
      duration: 60
      arrivalRate: 100
      rampTo: 0
  defaults:
    headers:
      Content-Type: "application/json"
      Accept: "application/json"
  ensure:
    thresholds:
      - http.response_time.p95: 200
      - http.response_time.p99: 500
      - http.request_rate: 100
  plugins:
    metrics-by-endpoint:
      useOnlyRequestNames: true
  payload:
    path: "wallet_ids.csv"
    fields:
      - "walletId"
      - "userId"
    order: sequence

scenarios:
  - name: "Read wallet"
    weight: 30
    flow:
      - get:
          url: "/api/v1/wallets/{{ walletId }}"
          capture:
            - json: "$.balance"
              as: "balance"
          expect:
            - statusCode: 200
            - hasProperty: "balance"
      - think: 2

  - name: "List transactions"
    weight: 25
    flow:
      - get:
          url: "/api/v1/wallets/{{ walletId }}/transactions?page=0&size=20"
          expect:
            - statusCode: 200
      - think: 2

  - name: "Deposit"
    weight: 20
    flow:
      - post:
          url: "/api/v1/wallets/{{ walletId }}/transactions/deposit"
          json:
            amount: "{{ $randomNumber(10, 1000) }}"
            description: "Artillery deposit"
            idempotency_key: "{{ $uuid }}"
          expect:
            - statusCode: 201
      - think: 3

  - name: "Transfer"
    weight: 10
    flow:
      - post:
          url: "/api/v1/wallets/{{ walletId }}/transactions/transfer"
          json:
            amount: "{{ $randomNumber(1, 100) }}"
            target_wallet_id: "{{ $randomNumber(1, 100) }}"
            description: "Artillery transfer"
            idempotency_key: "{{ $uuid }}"
          expect:
            - statusCode: 201
      - think: 3
```

```bash
# Executar Artillery
artillery run artillery-config.yml --output results.json
artillery report results.json --output report.html
```

### 5. Vegeta (Go)

```bash
# Vegeta — HTTP load testing tool em Go

# Ataque constante: 200 req/s por 5 minutos
echo "GET http://localhost:8080/api/v1/wallets/1" | \
  vegeta attack -rate=200/s -duration=300s | \
  vegeta report

# Múltiplos endpoints com pesos
cat <<EOF > targets.txt
GET http://localhost:8080/api/v1/wallets/1
GET http://localhost:8080/api/v1/wallets/2
GET http://localhost:8080/api/v1/wallets/1/transactions
POST http://localhost:8080/api/v1/wallets/1/transactions/deposit
Content-Type: application/json
@deposit.json
EOF

vegeta attack -targets=targets.txt -rate=100/s -duration=300s | \
  vegeta report -type=text

# Relatório em formato HDR histogram
vegeta attack -rate=200/s -duration=300s < targets.txt | \
  vegeta report -type=hdrplot > latency.hdr

# Plot (requer gnuplot)
vegeta attack -rate=200/s -duration=300s < targets.txt | \
  vegeta plot > plot.html

# Encode/decode para análise posterior
vegeta attack -rate=200/s -duration=300s < targets.txt | \
  vegeta encode > results.gob

cat results.gob | vegeta report -type=json
```

---

## Coordinated Omission

```
O QUE É COORDINATED OMISSION?

Quando o sistema sob teste fica lento, muitas ferramentas param de enviar
requests (aguardam o response) e depois reportam MENOS amostras no período
de alta latência. Isso ESCONDE os piores tempos de resposta.

Exemplo:
  SUT normalmente responde em 1ms
  Durante 1 segundo, SUT responde em 1000ms (problem)
  
  Sem correção:
    → 999 requests de 1ms + 1 request de 1000ms
    → p99 = 1ms (ERRADO! escondeu o problema)
  
  Com correção:
    → 999 requests de 1ms + ~999 requests "faltantes" de ~1000ms
    → p99 = ~1000ms (CORRETO! mostra o impacto real)

COMO CADA FERRAMENTA LIDA:

| Ferramenta | Coordinated Omission |
|-----------|----------------------|
| k6         | ✅ Corrige (arrival-rate executors) |
| JMeter     | ❌ Não corrige (Thread Groups bloqueiam) |
| Gatling    | ✅ Corrige (open model injection) |
| Locust     | ❌ Não corrige (event loop baseado) |
| wrk2       | ✅ Corrige nativamente (-R flag) |
| Artillery  | ✅ Corrige (arrival rate based) |
| Vegeta     | ✅ Corrige (constant rate attack) |

⚠️  Para medição PRECISA de latência, use ferramentas que corrigem CO.
    Para testes funcionais/exploratórios, CO importa menos.
```

---

## Critérios de Aceite

- [ ] **Gatling**: simulação Java com open model injection, feeders CSV, assertions
- [ ] **Locust**: locustfile.py com tasks weighted, custom assertions, modo headless
- [ ] **wrk2**: script Lua com POST + body dinâmico, corrigindo coordinated omission
- [ ] **Artillery**: config YAML com fases, thresholds, e payload CSV
- [ ] **Vegeta**: script bash com múltiplos targets, report HDR
- [ ] Mesmos endpoints testados por todas as 5 ferramentas
- [ ] Script `compare-tools.sh` executando cada ferramenta e gerando tabela comparativa
- [ ] Relatório `TOOL_COMPARISON.md` com:
  - p50, p95, p99, p99.9 de cada ferramenta
  - RPS atingido
  - Recurso consumido pela ferramenta (CPU/RAM)
  - Facilidade de uso (1-5)
  - Análise de coordinated omission
- [ ] Identificar discrepâncias de latência entre ferramentas com e sem correção de CO
- [ ] Docker Compose com todas as ferramentas pré-configuradas

---

## Definição de Pronto (DoD)

- [ ] 5 scripts/configs de load test (um por ferramenta) em `perf/tools/`
- [ ] `docker-compose.tools.yml` com containers para cada ferramenta
- [ ] Script `compare-tools.sh` que executa todas e gera tabela
- [ ] Relatório `TOOL_COMPARISON.md` com análise quantitativa
- [ ] README com instruções de setup e execução de cada ferramenta
- [ ] Commit: `feat(perf-level-4): add Gatling, Locust, wrk2, Artillery, Vegeta load tests`

---

## Checklist

### Setup

- [ ] Gatling 3.10+ instalado (Maven/Gradle plugin ou standalone)
- [ ] Locust 2.x instalado (`pip install locust`)
- [ ] wrk2 compilado (`git clone` + `make`)
- [ ] Artillery instalado (`npm install -g artillery`)
- [ ] Vegeta instalado (`go install github.com/tsenart/vegeta@latest`)
- [ ] Docker images disponíveis para cada ferramenta

### Exercícios

- [ ] Executar cada ferramenta contra a mesma API com ~200 req/s por 10 min
- [ ] Coletar p50, p95, p99, p99.9 de cada ferramenta
- [ ] Comparar resultados entre ferramentas com e sem CO (ex: wrk2 vs Locust)
- [ ] Medir overhead de cada ferramenta (CPU/RAM do client)
- [ ] Criar cenário de "slow endpoint" (sleep 1s) para evidenciar CO

---

## Tarefas Sugeridas por Stack

### Go (Gin)

- [ ] Vegeta como ferramenta primária (mesmo ecossistema Go)
- [ ] wrk2 para validar latência com correção de CO
- [ ] Comparar resultados Vegeta vs wrk2 para o mesmo endpoint

### Spring Boot / Quarkus / Micronaut / Jakarta EE

- [ ] Gatling Java DSL como ferramenta primária (mesmo ecossistema JVM)
- [ ] Integrar Gatling no build Maven/Gradle:
  ```xml
  <!-- pom.xml — Gatling Maven Plugin -->
  <plugin>
      <groupId>io.gatling</groupId>
      <artifactId>gatling-maven-plugin</artifactId>
      <version>4.9.6</version>
  </plugin>
  ```
  ```bash
  mvn gatling:test -Dgatling.simulationClass=WalletSimulation
  ```
- [ ] Locust como ferramenta secundária para cenários exploratórios
- [ ] Artillery para smoke tests rápidos em CI/CD

---

## Matriz de Decisão de Ferramentas

```
┌─────────────────────────────────────────────────────────────────┐
│                 QUAL FERRAMENTA USAR?                           │
├─────────────┬──────┬────────┬───────┬──────────┬───────────────┤
│             │  k6  │ Gatling│ Locust│  wrk2    │  Artillery    │
├─────────────┼──────┼────────┼───────┼──────────┼───────────────┤
│ CI/CD       │ ★★★★★│ ★★★★  │ ★★★  │ ★★★★★   │ ★★★★         │
│ Scripting   │ JS   │ Java   │ Python│ Lua     │ YAML          │
│ Protocols   │ H,WS │ H,WS,J│ H,WS │ H       │ H,WS,Socket  │
│ GUI         │ ✗    │ ✗     │ ✓ web │ ✗       │ ✗             │
│ Distributed │ Cloud│ ✓     │ ✓    │ ✗       │ ✗             │
│ CO handled  │ ✓    │ ✓     │ ✗    │ ✓       │ ✓             │
│ Memory      │ Low  │ Med   │ Low  │ Minimal │ Low           │
│ Relatórios  │ JSON │ HTML  │ HTML │ Text    │ HTML/JSON     │
│ Melhor para │ Dev  │ JVM   │ Data │ Quick   │ Declarativo   │
│             │ team │ teams │ teams│ bench   │ simple tests  │
└─────────────┴──────┴────────┴───────┴──────────┴───────────────┘

H = HTTP, WS = WebSocket, J = JMS
```

---

## Extensões Opcionais

- [ ] Locust distributed com Docker Swarm (master + N workers)
- [ ] Gatling com protocolo WebSocket para teste de real-time features
- [ ] wrk2 com scripts Lua avançados (autenticação, correlação)
- [ ] Artillery com plugins customizados (métricas para Datadog)
- [ ] hey (Go) como alternativa simples ao wrk2
- [ ] Comparar coordinated omission: criar endpoint com `sleep(random 0-2s)` e rodar todas as ferramentas

---

## Erros Comuns

| Erro | Consequência | Correção |
|------|-------------|----------|
| Ignorar coordinated omission | Latências parecem melhores que a realidade | Usar open model / arrival-rate |
| Testar da mesma máquina do servidor | Client e server competem por CPU/rede | Máquinas separadas |
| Não agrupar URLs dinâmicas no Locust | Cada URL vira uma linha no relatório | `name="/wallets/[id]"` |
| Gatling closed model para load test | CO, não simula produção real | `injectOpen()` sempre |
| wrk2 sem `-R` flag | wrk2 sem rate = wrk1 (sem correção CO) | Sempre usar `-R <rate>` |
| Comparar ferramentas sem warm-up | Cold start distorce resultados | Phase de warm-up em todas |
| Artillery sem `ensure.thresholds` | Teste nunca falha mesmo com SLO violado | Definir thresholds |

---

## Como Isso Aparece em Entrevistas

- "Descreva 3 ferramentas de load testing e quando usaria cada uma"
- "O que é coordinated omission e por que é problemático?"
- "Compare open model vs closed model em load testing"
- "Como você escolheria uma ferramenta de load testing para um projeto novo?"
- "Implemente um load test distribuído para 10.000 req/s"
- "Qual a diferença na latência reportada por wrk2 vs Locust quando o servidor fica lento?"
