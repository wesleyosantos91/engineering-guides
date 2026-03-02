# Level 3 — Load Testing com JMeter

> **Objetivo:** Dominar Apache JMeter para load testing enterprise — criando Test Plans com Thread Groups, Samplers, Assertions, Listeners, parametrização avançada, correlação, distributed testing e integração com pipelines CI/CD.

---

## Objetivo de Aprendizado

- Instalar e configurar Apache JMeter (GUI e CLI mode)
- Entender a arquitetura: Test Plan → Thread Group → Sampler → Assertion → Listener
- Criar Test Plans para todos os tipos de load test (load, stress, soak, spike)
- Parametrizar requests com CSV Data Set Config e variáveis
- Implementar correlação (extrair valores de responses e reutilizar)
- Configurar assertions para validação automática de respostas
- Rodar JMeter em modo CLI (non-GUI) para testes de carga reais
- Configurar distributed testing (master-slave) para alta carga
- Gerar relatórios HTML e integrar com CI/CD
- Entender limitações do JMeter (coordinated omission, overhead)

---

## Escopo Funcional

### Test Plan da Digital Wallet API

```
Test Plan: Digital Wallet Load Test
│
├── Thread Group: Read Operations (70% da carga)
│   ├── HTTP Request: GET /api/v1/wallets/${walletId}
│   ├── HTTP Request: GET /api/v1/wallets/${walletId}/transactions
│   ├── HTTP Request: GET /api/v1/users/${userId}
│   ├── Response Assertion: status = 200
│   └── JSON Assertion: $.content is not null
│
├── Thread Group: Write Operations (30% da carga)
│   ├── HTTP Request: POST /api/v1/wallets/${walletId}/transactions/deposit
│   ├── HTTP Request: POST /api/v1/wallets/${walletId}/transactions/transfer
│   ├── HTTP Request: POST /api/v1/users
│   ├── JSON Extractor: $.id → created_user_id
│   ├── Response Assertion: status = 201
│   └── JSON Assertion: $.id is not null
│
├── CSV Data Set Config: wallet_ids.csv, user_ids.csv
├── HTTP Header Manager: Content-Type, Authorization
├── HTTP Cookie Manager
├── View Results Tree (debug only)
├── Summary Report
└── Aggregate Report
```

---

## Escopo Técnico

### Arquitetura do JMeter

```
TEST PLAN (arquivo .jmx)
│
├── Thread Group (simula usuários)
│   ├── Número de threads (VUs)
│   ├── Ramp-up period (tempo para criar todas as threads)
│   ├── Loop count (iterações por thread)
│   └── Duration (tempo total)
│
├── Config Elements (configuração compartilhada)
│   ├── HTTP Request Defaults (base URL, headers default)
│   ├── CSV Data Set Config (dados parametrizados)
│   ├── HTTP Header Manager (headers comuns)
│   └── User Defined Variables (constantes)
│
├── Samplers (requests HTTP, JDBC, etc.)
│   ├── HTTP Request
│   ├── JDBC Request
│   ├── JSR223 Sampler (Groovy script)
│   └── Debug Sampler
│
├── Pre-Processors (antes do request)
│   ├── JSR223 PreProcessor (gerar dados dinâmicos)
│   └── User Parameters
│
├── Post-Processors (após o response)
│   ├── JSON Extractor (extrair campo do JSON)
│   ├── Regular Expression Extractor
│   └── JSR223 PostProcessor
│
├── Assertions (validação)
│   ├── Response Assertion (status code, body)
│   ├── JSON Assertion (JSONPath)
│   ├── Duration Assertion (timeout)
│   └── Size Assertion
│
├── Timers (think time)
│   ├── Constant Timer
│   ├── Gaussian Random Timer
│   └── Uniform Random Timer
│
└── Listeners (coleta de resultados)
    ├── View Results Tree (debug — NUNCA em carga real)
    ├── Summary Report
    ├── Aggregate Report
    ├── Backend Listener (InfluxDB/Grafana)
    └── Simple Data Writer (CSV para análise)
```

### JMeter GUI vs CLI

```
GUI MODE — apenas para CRIAR e DEBUGAR test plans
  jmeter.bat (Windows) / jmeter.sh (Linux)
  → Arrastar e soltar elementos
  → Ver responses individualmente (View Results Tree)
  → NUNCA use GUI para load test real (overhead alto)

CLI MODE — para EXECUTAR load tests
  jmeter -n -t test-plan.jmx -l results.jtl -e -o report/
  
  Flags:
  -n          non-GUI mode
  -t          test plan file
  -l          log de resultados (JTL/CSV)
  -e -o       gerar relatório HTML
  -Jthreads=  override de property
  -Jduration= override de property

⚠️  SEMPRE executar load tests em CLI mode!
```

---

## Critérios de Aceite

- [ ] Test Plan `.jmx` com workload mix realista (70% read, 30% write)
- [ ] **CSV Data Set Config** com pelo menos 100 wallet IDs e user IDs
- [ ] **JSON Extractors** para correlação: criar user → extrair ID → criar wallet → usar wallet ID
- [ ] **Assertions** em todos os requests: status code + JSON body
- [ ] **Gaussian Random Timer** para think time realista (1-3 segundos)
- [ ] **Thread Group para Load Test**: 100 threads, 5min ramp-up, 30min duration
- [ ] **Thread Group para Stress Test**: stepping 50→500 threads com Concurrency Thread Group plugin
- [ ] **Backend Listener** configurado para exportar métricas ao InfluxDB/Grafana
- [ ] Execução em **CLI mode** com geração de relatório HTML
- [ ] **Distributed testing** configurado com pelo menos 2 slaves (Docker)
- [ ] Script bash para executar JMeter com propriedades parametrizáveis
- [ ] Relatório `JMETER_REPORT.md` com análise de resultados

---

## Definição de Pronto (DoD)

- [ ] Arquivo `.jmx` funcional em `perf/jmeter/wallet-load-test.jmx`
- [ ] CSV files de dados em `perf/jmeter/data/`
- [ ] Script `run-jmeter.sh` para execução CLI com parâmetros
- [ ] Docker Compose com JMeter master + slaves
- [ ] Relatório HTML gerado e analisável
- [ ] Dashboard Grafana com métricas do JMeter (via Backend Listener)
- [ ] Commit: `feat(perf-level-3): add JMeter test plans with distributed testing`

---

## Checklist

### Setup JMeter

- [ ] JMeter 5.6+ instalado (`jmeter --version`)
- [ ] Plugins Manager instalado (JMeter Plugins)
- [ ] Plugins essenciais:
  - [ ] Concurrency Thread Group
  - [ ] Throughput Shaping Timer
  - [ ] Custom Thread Groups
  - [ ] 3 Basic Graphs
  - [ ] PerfMon (Monitor de servidor)

### Criação do Test Plan

```
── Passo 1: Test Plan base ──

1. Abrir JMeter GUI
2. Adicionar → Thread Group
   - Number of Threads: ${__P(threads,100)}
   - Ramp-Up Period: ${__P(rampup,300)}
   - Duration: ${__P(duration,1800)}

3. Adicionar → Config → HTTP Request Defaults
   - Protocol: http
   - Server: ${__P(host,localhost)}
   - Port: ${__P(port,8080)}

4. Adicionar → Config → HTTP Header Manager
   - Content-Type: application/json
   - Accept: application/json
```

### Parametrização com CSV

```
── wallet_ids.csv ──
wallet_id,user_id,balance
1,1,10000.00
2,2,5000.00
3,3,8500.00
...

── Configurar CSV Data Set Config ──
Filename: wallet_ids.csv
Variable Names: wallet_id,user_id,balance
Delimiter: ,
Recycle on EOF: True
Stop thread on EOF: False
Sharing mode: All threads
```

### Requests com Correlação

```
── Fluxo de Correlação ──

1. POST /api/v1/users
   Body: {"name":"User ${__counter(,)}","email":"user${__counter(,)}@test.com","document":"${__Random(10000000000,99999999999)}"}
   
   → JSON Extractor:
     Variable: created_user_id
     JSON Path: $.id

2. POST /api/v1/users/${created_user_id}/wallets
   Body: {"currency":"BRL"}
   
   → JSON Extractor:
     Variable: created_wallet_id
     JSON Path: $.id

3. POST /api/v1/wallets/${created_wallet_id}/transactions/deposit
   Body: {"amount":1000.00,"description":"Initial deposit","idempotency_key":"${__UUID()}"}

4. GET /api/v1/wallets/${created_wallet_id}/transactions
   → Response Assertion: status = 200
   → JSON Assertion: $.content[0].amount = 1000.00
```

### Assertions

```xml
<!-- Response Assertion -->
<ResponseAssertion>
  <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
  <stringProp name="Assertion.test_type">2</stringProp> <!-- equals -->
  <stringProp name="Assertion.test_string">200</stringProp>
</ResponseAssertion>

<!-- JSON Assertion -->
<JSONPathAssertion>
  <stringProp name="JSON_PATH">$.id</stringProp>
  <boolProp name="EXPECT_NULL">false</boolProp>
  <boolProp name="INVERT">false</boolProp>
</JSONPathAssertion>

<!-- Duration Assertion -->
<DurationAssertion>
  <stringProp name="DurationAssertion.duration">500</stringProp> <!-- max 500ms -->
</DurationAssertion>
```

### Think Time

```
── Gaussian Random Timer ──
Deviation: 1000 ms
Constant Delay: 2000 ms
→ Think time entre 1s e 3s (distribuição normal)

── Uniform Random Timer ──
Random Delay Max: 2000 ms
Constant Delay: 1000 ms
→ Think time entre 1s e 3s (distribuição uniforme)
```

### Backend Listener (InfluxDB)

```
── Configuração ──
Implementation: org.apache.jmeter.visualizers.backend.influxdb.InfluxdbBackendListenerClient
influxdbUrl: http://localhost:8086/write?db=jmeter
application: wallet-api
measurement: jmeter
summaryOnly: false
samplersRegex: .*
testTitle: Digital Wallet Load Test
```

### Execução CLI

```bash
#!/bin/bash
# run-jmeter.sh — Executar JMeter em modo CLI

JMETER_HOME=${JMETER_HOME:-/opt/apache-jmeter}
TEST_PLAN=${1:-perf/jmeter/wallet-load-test.jmx}
RESULTS_DIR=perf/jmeter/results/$(date +%Y%m%d_%H%M%S)

mkdir -p $RESULTS_DIR

$JMETER_HOME/bin/jmeter \
  -n \
  -t $TEST_PLAN \
  -l $RESULTS_DIR/results.jtl \
  -e -o $RESULTS_DIR/report/ \
  -Jthreads=${THREADS:-100} \
  -Jrampup=${RAMPUP:-300} \
  -Jduration=${DURATION:-1800} \
  -Jhost=${HOST:-localhost} \
  -Jport=${PORT:-8080} \
  -j $RESULTS_DIR/jmeter.log

echo "Relatório HTML: $RESULTS_DIR/report/index.html"
echo "Resultados JTL: $RESULTS_DIR/results.jtl"

# Verificar se houve erros acima do threshold
ERROR_RATE=$(awk -F',' 'NR>1{total++; if($8=="false") errors++} END{printf "%.2f", (errors/total)*100}' $RESULTS_DIR/results.jtl)
echo "Error rate: ${ERROR_RATE}%"

if (( $(echo "$ERROR_RATE > 1.0" | bc -l) )); then
  echo "❌ FAIL: Error rate ${ERROR_RATE}% exceeds 1% threshold"
  exit 1
fi

echo "✅ PASS: Error rate within threshold"
```

### Distributed Testing (Docker)

```yaml
# docker-compose.jmeter.yml
services:
  jmeter-master:
    image: justb4/jmeter:5.6
    volumes:
      - ./perf/jmeter:/tests
    command: >
      -n -t /tests/wallet-load-test.jmx
      -l /tests/results/results.jtl
      -e -o /tests/results/report
      -R jmeter-slave1,jmeter-slave2
      -Jthreads=50
      -Jduration=1800
    depends_on:
      - jmeter-slave1
      - jmeter-slave2

  jmeter-slave1:
    image: justb4/jmeter:5.6
    command: -s
    expose:
      - "1099"
      - "4000"

  jmeter-slave2:
    image: justb4/jmeter:5.6
    command: -s
    expose:
      - "1099"
      - "4000"
```

### Concurrency Thread Group (Stress Test)

```
── Plugin: Concurrency Thread Group ──
Target Concurrency: 500
Ramp Up Time: 10 min
Ramp-Up Steps Count: 10
Hold Target Rate Time: 20 min

Resultado: sobe de 0 → 500 threads em 10 steps de 1 min cada,
depois mantém 500 threads por 20 min

── Plugin: Throughput Shaping Timer ──
| Start RPS | End RPS | Duration |
| 50        | 50      | 5m       |  ← baseline
| 50        | 200     | 5m       |  ← ramp
| 200       | 200     | 10m      |  ← hold
| 200       | 500     | 5m       |  ← ramp
| 500       | 500     | 10m      |  ← hold
| 500       | 0       | 5m       |  ← ramp down
```

---

## JMeter vs k6 — Comparação

| Aspecto | JMeter | k6 |
|---------|--------|-----|
| **Scripting** | XML (GUI) + Groovy | JavaScript |
| **Curva de aprendizado** | Alta (muitos elementos) | Baixa (código) |
| **Coordinated Omission** | ❌ Não corrige | ✅ Corrige |
| **Protocolos** | HTTP, JDBC, JMS, LDAP, FTP, SOAP, gRPC... | HTTP, WebSocket, gRPC |
| **Distributed** | Built-in (master/slave) | k6 Cloud ou xk6-distributed |
| **Overhead** | Alto (Java, GUI, XML parsing) | Baixo (Go binary) |
| **CI/CD** | CLI mode + scripts | Nativo, exit codes |
| **Relatórios** | HTML report built-in | JSON/CSV + Grafana |
| **Extensibilidade** | Plugins Java | Extensions Go (xk6) |
| **Comunidade** | Madura, enterprise | Crescente, moderna |
| **Melhor para** | Enterprise, multi-protocolo, equipes QA | DevOps, CI/CD, developers |

```
QUANDO USAR JMETER:
→ Testes multi-protocolo (JDBC, JMS, SOAP)
→ Equipe QA tradicional (GUI-friendly)
→ Enterprise com requirements de compliance
→ Distributed testing out-of-the-box
→ Correlação complexa (extractors, regex)

QUANDO USAR k6:
→ Developer-centric (código versionável)
→ CI/CD nativo (scripting + thresholds)
→ Medição precisa de latência (sem coordinated omission)
→ Leveza (Go binary vs JVM)
→ Integração nativa com Grafana ecosystem
```

---

## Extensões Opcionais

- [ ] JMeter com protocolo JDBC: load test direto no banco de dados
- [ ] Teste de WebSocket com JMeter WebSocket Sampler plugin
- [ ] Recording com JMeter HTTP(S) Test Script Recorder (proxy)
- [ ] Groovy scripting para lógica complexa (JSR223 Sampler)
- [ ] PerfMon plugin para monitorar CPU/RAM do servidor durante teste
- [ ] Comparar resultados JMeter vs k6 para os mesmos cenários
- [ ] JMeter + Maven plugin para integração no build Java

---

## Erros Comuns

| Erro | Consequência | Correção |
|------|-------------|----------|
| Usar GUI mode para load test | Overhead alto, resultados imprecisos | Sempre CLI mode (`-n`) |
| View Results Tree em carga | Consome toda a memória do JMeter | Desabilitar em load test |
| Sem Timer (think time) | Cada thread faz requests sem parar | Gaussian Random Timer |
| CSV sharing mode errado | Todos os threads usam os mesmos dados | Sharing mode: All threads |
| Não parametrizar threads/duration | Hardcoded no .jmx, não flexível | Properties: `${__P(threads,100)}` |
| JMeter single machine para alta carga | JMeter machine satura primeiro | Distributed testing (master/slave) |
| Coordinated omission não tratado | Latências reportadas são otimistas | Usar Throughput Shaping Timer plugin |
| Esquecer ramp-up | Cold start: threads criadas de uma vez | Ramp-up: mínimo 5 min |
| Assertions sem JSON Path | Validação superficial (só status code) | JSON Assertion com paths específicos |

---

## Como Isso Aparece em Entrevistas

- "Compare JMeter com k6. Quando usaria cada um?"
- "O que é coordinated omission e como o JMeter lida com isso?"
- "Como configurar distributed load testing com JMeter?"
- "Qual a diferença entre Thread Group e Concurrency Thread Group?"
- "Como parametrizar um test plan JMeter para reusar em ambientes diferentes?"
- "Descreva o workflow de criação de um test plan JMeter enterprise"
- "Como integrar JMeter no CI/CD pipeline?"
