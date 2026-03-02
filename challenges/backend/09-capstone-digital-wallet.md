# Level 9 — Capstone: Digital Wallet — Production Launch

> **Objetivo:** Integrar **todos** os conceitos dos levels 0–8 em um cenário de go-live real,
> validando a Digital Wallet como sistema production-grade em todas as 5 stacks,
> exercitando resiliência, consistência, observabilidade e deploy sob pressão.

**Pré-requisito:** Todos os levels anteriores (0–8) completos em pelo menos Go + Spring Boot.

---

## Contexto

A startup **NovaPay** escolheu sua implementação da Digital Wallet para ser a plataforma de carteira
digital do produto. Após 8 sprints de desenvolvimento (Levels 0–8), chegou o dia do **go-live**.

O Capstone simula as **72 horas críticas** em torno do lançamento:
- **Dia -1:** Production Readiness Review (PRR)
- **Dia 0:** Go-Live — deploy, smoke tests, abertura de tráfego
- **Dia +1:** Primeiro dia em produção — incidentes, escalas, chaos engineering

**Escala alvo do go-live:**
- **10K usuários** ativos no primeiro dia (crescimento para 100K em 30 dias)
- **50 TPS** (transações por segundo) sustentado, picos de **200 TPS**
- **Uptime:** ≥ 99.9% (≤ 43 min downtime/mês)
- **Latência:** p95 < 200ms para leitura, p95 < 500ms para escrita
- **Zero perda de transação** — toda operação financeira rastreável

---

## Parte 1 — Production Readiness Review (PRR)

### 1.1 — Checklist de Produção por Stack

Valide que **cada uma das 5 stacks** atende todos os critérios de produção definidos no README.
Monte uma **matriz de conformidade** cruzando requisitos × stacks.

```
                          Go(Gin)  Spring  Quarkus  Micronaut  Jakarta
─────────────────────────────────────────────────────────────────────
CRUD completo             [  ]     [  ]    [  ]     [  ]       [  ]
Validação (RFC 9457)      [  ]     [  ]    [  ]     [  ]       [  ]
Paginação/Ordenação       [  ]     [  ]    [  ]     [  ]       [  ]
Migrations versionadas    [  ]     [  ]    [  ]     [  ]       [  ]
Testes unit >80%          [  ]     [  ]    [  ]     [  ]       [  ]
Testes integração (TC)    [  ]     [  ]    [  ]     [  ]       [  ]
Health (liveness+ready)   [  ]     [  ]    [  ]     [  ]       [  ]
Métricas (Prometheus)     [  ]     [  ]    [  ]     [  ]       [  ]
Tracing (OpenTelemetry)   [  ]     [  ]    [  ]     [  ]       [  ]
Logging estruturado       [  ]     [  ]    [  ]     [  ]       [  ]
JWT + RBAC                [  ]     [  ]    [  ]     [  ]       [  ]
Rate limiting             [  ]     [  ]    [  ]     [  ]       [  ]
Circuit breaker           [  ]     [  ]    [  ]     [  ]       [  ]
Idempotência              [  ]     [  ]    [  ]     [  ]       [  ]
Kafka producer/consumer   [  ]     [  ]    [  ]     [  ]       [  ]
Graceful shutdown         [  ]     [  ]    [  ]     [  ]       [  ]
Docker multi-stage        [  ]     [  ]    [  ]     [  ]       [  ]
OpenAPI/Swagger           [  ]     [  ]    [  ]     [  ]       [  ]
Feature flags             [  ]     [  ]    [  ]     [  ]       [  ]
─────────────────────────────────────────────────────────────────────
Score                     __/19    __/19   __/19    __/19      __/19
```

### 1.2 — Documento de PRR

Produza um documento formal de Production Readiness Review (template abaixo):

```markdown
# Production Readiness Review — NovaPay Digital Wallet

## 1. Visão Geral
- Data de go-live prevista: ___
- Stacks em produção: ___
- Owner técnico: ___

## 2. Arquitetura
- Diagrama de arquitetura (C4 Container)
- Fluxo de dados: request → API → DB → Kafka → consumer
- Dependências externas: Kafka, PostgreSQL/MySQL, API de câmbio

## 3. Capacidade
- Throughput estimado: ___ TPS
- Carga máxima testada: ___ TPS (resultado do Level 8)
- Headroom: ___% acima do pico estimado

## 4. Resiliência
- Timeout configurado: ___ ms
- Retry: ___ tentativas com backoff de ___ ms
- Circuit breaker: threshold ___, window ___
- Fallback definido para: ___

## 5. Observabilidade
- Dashboards: ___ (listar)
- Alertas configurados: ___ (listar regras)
- Runbook de on-call: link

## 6. Segurança
- Autenticação: JWT RS256/ES256
- Autorização: RBAC (roles: ADMIN, USER, AUDITOR)
- Rate limiting: ___ req/s por usuário
- Secrets management: ___

## 7. Rollback Plan
- Estratégia: ___ (blue-green / canary / rollback manual)
- Tempo estimado de rollback: ___ minutos
- Procedimento: ___

## 8. Riscos e Mitigações
| Risco | Probabilidade | Impacto | Mitigação |
|-------|:---:|:---:|-----------|
| DB connection pool esgotado | Média | Alto | Pool sizing + circuit breaker |
| Kafka consumer lag | Baixa | Médio | Monitorar lag, scale consumers |
| Race condition em transferência | Alta | Crítico | Optimistic lock + idempotência |
| API de câmbio indisponível | Média | Médio | Circuit breaker + fallback cache |

## 9. Go/No-Go Decision
- [ ] PRR checklist ≥ 95% green
- [ ] Load test sustentando 2× pico estimado
- [ ] Rollback testado e documentado
- [ ] On-call escalation definido
- [ ] Stakeholders assinaram
```

**Critérios de aceite:**
- [ ] Matriz de conformidade preenchida para todas as 5 stacks
- [ ] Documento PRR completo (todas as seções)
- [ ] Diagrama de arquitetura C4 (ao menos Container level)
- [ ] Riscos identificados com mitigações documentadas
- [ ] Go/No-Go checklist verificado

---

## Parte 2 — Go-Live: Deploy e Validação End-to-End

### 2.1 — Infraestrutura de Produção

Monte o ambiente de produção completo via Docker Compose com todas as dependências.

```yaml
# docker-compose.production.yml
services:
  # === Infraestrutura ===
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: novapay_wallet
      POSTGRES_USER: wallet_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U wallet_user -d novapay_wallet"]
      interval: 5s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 1G

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: "1@kafka:9093"
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
      KAFKA_LOG_RETENTION_HOURS: 168
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 10s
      retries: 10

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 3

  # === Observability ===
  prometheus:
    image: prom/prometheus:v2.50.0
    volumes:
      - ./infra/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./infra/prometheus/alerts.yml:/etc/prometheus/alerts.yml
    ports: ["9090:9090"]

  grafana:
    image: grafana/grafana:11.0.0
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - ./infra/grafana/provisioning:/etc/grafana/provisioning
      - ./infra/grafana/dashboards:/var/lib/grafana/dashboards
    ports: ["3000:3000"]

  jaeger:
    image: jaegertracing/all-in-one:1.54
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
    ports: ["16686:16686", "4317:4317"]

  # === API Stacks (deploy 1 por vez ou todas para benchmark) ===
  api-go:
    build: ./go-wallet
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
      redis: { condition: service_healthy }
    environment:
      DATABASE_URL: postgres://wallet_user:${DB_PASSWORD}@postgres:5432/novapay_wallet?sslmode=disable
      KAFKA_BROKERS: kafka:9092
      REDIS_URL: redis://redis:6379
      OTEL_EXPORTER_OTLP_ENDPOINT: http://jaeger:4317
      JWT_SECRET: ${JWT_SECRET}
    ports: ["8081:8080"]
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 512M

  api-spring:
    build: ./spring-wallet
    depends_on:
      postgres: { condition: service_healthy }
      kafka: { condition: service_healthy }
      redis: { condition: service_healthy }
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/novapay_wallet
      SPRING_DATASOURCE_USERNAME: wallet_user
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      OTEL_EXPORTER_OTLP_ENDPOINT: http://jaeger:4317
      JWT_SECRET: ${JWT_SECRET}
    ports: ["8082:8080"]
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 512M

  # api-quarkus, api-micronaut, api-jakarta — mesma estrutura
  # Ports: 8083, 8084, 8085

volumes:
  pgdata:
```

### 2.2 — Jornada Completa de Usuário (End-to-End)

Execute a jornada completa que exercita **todos os endpoints e patterns**:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    JORNADA DO GO-LIVE                                │
│                                                                      │
│  1. Criar Usuário (João)                                             │
│     POST /api/v1/users                                               │
│     → Validação, persistência, audit log                             │
│                                                                      │
│  2. Criar Wallet BRL                                                 │
│     POST /api/v1/users/{id}/wallets                                  │
│     → currency=BRL, saldo=0                                          │
│                                                                      │
│  3. Depositar R$ 1.000,00 (com idempotency_key)                     │
│     POST /api/v1/wallets/{id}/transactions/deposit                   │
│     → Saldo atualizado, evento Kafka emitido, notificação async      │
│                                                                      │
│  4. Depositar novamente com MESMA idempotency_key                    │
│     → Retorna 200 com transação existente (NÃO duplica)              │
│                                                                      │
│  5. Criar Usuário (Maria)                                            │
│     POST /api/v1/users                                               │
│                                                                      │
│  6. Criar Wallet BRL para Maria                                      │
│     POST /api/v1/users/{id}/wallets                                  │
│                                                                      │
│  7. Transferir R$ 250,00 de João para Maria                          │
│     POST /api/v1/wallets/{joao}/transactions/transfer                │
│     → Atômico: débito João + crédito Maria na mesma tx DB            │
│     → Evento Kafka com ambos os wallet_ids                           │
│     → Notificação para João e Maria                                  │
│                                                                      │
│  8. Sacar R$ 100,00 da wallet de João                                │
│     POST /api/v1/wallets/{joao}/transactions/withdrawal              │
│     → Saldo: 1000 - 250 - 100 = R$ 650,00                           │
│                                                                      │
│  9. Tentar sacar R$ 1.000,00 (saldo insuficiente)                    │
│     → 422 ProblemDetail: INSUFFICIENT_BALANCE                        │
│                                                                      │
│  10. Consultar extrato paginado                                      │
│      GET /api/v1/wallets/{joao}/transactions?page=0&size=10&sort=... │
│      → 3 transações na ordem correta                                 │
│                                                                      │
│  11. Consultar transação específica                                  │
│      GET /api/v1/transactions/{id}                                   │
│      → Detalhes da transferência                                     │
│                                                                      │
│  12. Listar wallets do usuário                                       │
│      GET /api/v1/users/{joao}/wallets                                │
│      → 1 wallet, saldo = R$ 650,00                                   │
│                                                                      │
│  13. Bloquear wallet de João                                         │
│      POST /api/v1/wallets/{joao}/block                               │
│      → status = BLOCKED                                              │
│                                                                      │
│  14. Tentar depositar em wallet bloqueada                            │
│      → 422 ProblemDetail: WALLET_BLOCKED                             │
│                                                                      │
│  15. Verificar observabilidade                                       │
│      → Trace end-to-end no Jaeger                                    │
│      → Métricas no Prometheus/Grafana                                │
│      → Logs estruturados com correlation_id                          │
│                                                                      │
│  16. Verificar Kafka consumer lag                                    │
│      → Tópicos: wallet-transactions, wallet-notifications            │
│      → Consumer lag = 0 (tudo processado)                            │
└──────────────────────────────────────────────────────────────────────┘
```

**Script de validação E2E:**

```bash
#!/bin/bash
set -euo pipefail

BASE_URL="${1:-http://localhost:8081}"
TOKEN="${2:-$(cat .token)}"
PASS=0
FAIL=0

assert_status() {
  local expected=$1 actual=$2 step=$3
  if [[ "$actual" == "$expected" ]]; then
    echo "✅ Step $step: HTTP $actual"
    ((PASS++))
  else
    echo "❌ Step $step: expected $expected, got $actual"
    ((FAIL++))
  fi
}

echo "=== NovaPay E2E Journey — $BASE_URL ==="

# Step 1: Create user João
JOAO=$(curl -s -w "\n%{http_code}" -X POST "$BASE_URL/api/v1/users" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"João Silva","email":"joao@novapay.com","document":"123.456.789-00"}')
STATUS=$(echo "$JOAO" | tail -1)
JOAO_ID=$(echo "$JOAO" | head -1 | jq -r '.id')
assert_status 201 "$STATUS" "1-create-user-joao"

# Step 2: Create wallet BRL
WALLET_JOAO=$(curl -s -w "\n%{http_code}" -X POST "$BASE_URL/api/v1/users/$JOAO_ID/wallets" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"currency":"BRL"}')
STATUS=$(echo "$WALLET_JOAO" | tail -1)
WALLET_JOAO_ID=$(echo "$WALLET_JOAO" | head -1 | jq -r '.id')
assert_status 201 "$STATUS" "2-create-wallet-joao"

# Step 3: Deposit R$ 1000
IDEMP_KEY=$(uuidgen)
DEP=$(curl -s -w "\n%{http_code}" -X POST \
  "$BASE_URL/api/v1/wallets/$WALLET_JOAO_ID/transactions/deposit" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"amount\":1000.00,\"description\":\"Depósito inicial\",\"idempotency_key\":\"$IDEMP_KEY\"}")
STATUS=$(echo "$DEP" | tail -1)
assert_status 201 "$STATUS" "3-deposit-1000"

# Step 4: Duplicate deposit (idempotency)
DEP2=$(curl -s -w "\n%{http_code}" -X POST \
  "$BASE_URL/api/v1/wallets/$WALLET_JOAO_ID/transactions/deposit" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"amount\":1000.00,\"description\":\"Depósito inicial\",\"idempotency_key\":\"$IDEMP_KEY\"}")
STATUS=$(echo "$DEP2" | tail -1)
assert_status 200 "$STATUS" "4-idempotency-check"

# Step 5-6: Create Maria + wallet
MARIA=$(curl -s -w "\n%{http_code}" -X POST "$BASE_URL/api/v1/users" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"Maria Santos","email":"maria@novapay.com","document":"987.654.321-00"}')
STATUS=$(echo "$MARIA" | tail -1)
MARIA_ID=$(echo "$MARIA" | head -1 | jq -r '.id')
assert_status 201 "$STATUS" "5-create-user-maria"

WALLET_MARIA=$(curl -s -w "\n%{http_code}" -X POST "$BASE_URL/api/v1/users/$MARIA_ID/wallets" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"currency":"BRL"}')
STATUS=$(echo "$WALLET_MARIA" | tail -1)
WALLET_MARIA_ID=$(echo "$WALLET_MARIA" | head -1 | jq -r '.id')
assert_status 201 "$STATUS" "6-create-wallet-maria"

# Step 7: Transfer R$ 250
TRANSFER=$(curl -s -w "\n%{http_code}" -X POST \
  "$BASE_URL/api/v1/wallets/$WALLET_JOAO_ID/transactions/transfer" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"target_wallet_id\":\"$WALLET_MARIA_ID\",\"amount\":250.00,\"description\":\"Pagamento\",\"idempotency_key\":\"$(uuidgen)\"}")
STATUS=$(echo "$TRANSFER" | tail -1)
assert_status 201 "$STATUS" "7-transfer-250"

# Step 8: Withdraw R$ 100
WITHDRAWAL=$(curl -s -w "\n%{http_code}" -X POST \
  "$BASE_URL/api/v1/wallets/$WALLET_JOAO_ID/transactions/withdrawal" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"amount\":100.00,\"description\":\"Saque\",\"idempotency_key\":\"$(uuidgen)\"}")
STATUS=$(echo "$WITHDRAWAL" | tail -1)
assert_status 201 "$STATUS" "8-withdrawal-100"

# Step 9: Insufficient balance
FAIL_WITHDRAW=$(curl -s -w "\n%{http_code}" -X POST \
  "$BASE_URL/api/v1/wallets/$WALLET_JOAO_ID/transactions/withdrawal" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"amount\":1000.00,\"description\":\"Saque grande\",\"idempotency_key\":\"$(uuidgen)\"}")
STATUS=$(echo "$FAIL_WITHDRAW" | tail -1)
assert_status 422 "$STATUS" "9-insufficient-balance"

# Step 10: Statement
STMT=$(curl -s -w "\n%{http_code}" \
  "$BASE_URL/api/v1/wallets/$WALLET_JOAO_ID/transactions?page=0&size=10&sort=created_at,desc" \
  -H "Authorization: Bearer $TOKEN")
STATUS=$(echo "$STMT" | tail -1)
TOTAL=$(echo "$STMT" | head -1 | jq '.page.totalElements')
assert_status 200 "$STATUS" "10-statement"
echo "   📊 Transações no extrato: $TOTAL (esperado: 3)"

# Step 11-12: Get transaction, list wallets
# ... (similar pattern)

echo ""
echo "=== Resultado: $PASS passed, $FAIL failed ==="
[[ $FAIL -eq 0 ]] && echo "🎉 ALL TESTS PASSED" || echo "💥 SOME TESTS FAILED"
exit $FAIL
```

**Critérios de aceite:**
- [ ] Jornada E2E completa (16 steps) passa em TODAS as 5 stacks
- [ ] Script `e2e-journey.sh` automatizado e reutilizável
- [ ] Idempotência funciona (step 4 retorna 200, não 201)
- [ ] Transferência atômica (saldo consistente em ambas wallets)
- [ ] Wallet bloqueada rejeita transações (422)
- [ ] Kafka: mensagens publicadas e consumidas (lag = 0)
- [ ] Traces end-to-end visíveis no Jaeger para todo o fluxo
- [ ] Métricas no Prometheus para request count, latency, error rate

---

## Parte 3 — Chaos Engineering: Primeiro Dia em Produção

### 3.1 — Fault Injection Scenarios

Simule falhas reais que acontecem em produção para validar a resiliência do sistema.

**Cenário 1: Database Connection Pool Exhaustion**

```bash
# Simular: pool de conexões esgotado (max_connections baixo)
# 1. Reduzir max_connections do PostgreSQL para 5
docker exec postgres psql -U wallet_user -c "ALTER SYSTEM SET max_connections = 5;"
docker restart postgres

# 2. Executar carga que requer > 5 conexões simultâneas
k6 run -e BASE_URL=http://localhost:8081 \
  --vus 20 --duration 30s k6/scenarios/mixed-workload.js

# Esperado:
# - Circuit breaker abre após falhas de conexão
# - Requests retornam 503 (Service Unavailable) com ProblemDetail
# - Métricas: db_connection_pool_active, circuit_breaker_state
# - Alerta dispara no Prometheus

# 3. Restaurar max_connections e verificar recovery
docker exec postgres psql -U wallet_user -c "ALTER SYSTEM SET max_connections = 100;"
docker restart postgres
# Circuit breaker: OPEN → HALF_OPEN → CLOSED
```

**Cenário 2: Kafka Broker Down**

```bash
# 1. Derrubar Kafka
docker stop kafka

# 2. Executar transações (depósito, transferência)
# Esperado:
# - Transação financeira (DB) é efetivada normalmente
# - Publicação no Kafka falha → evento armazenado em outbox table
# - Consumer notification para de processar (sem Kafka)
# - Métricas: kafka_producer_failed_total incrementa

# 3. Subir Kafka
docker start kafka

# 4. Verificar que outbox events são republicados
# - Outbox processor detecta registros pendentes
# - Eventos publicados no Kafka
# - Consumer retoma processamento
# - Consumer lag volta a 0
```

**Cenário 3: Concurrent Transfers (Race Condition)**

```bash
# Simular: 50 transfers simultâneas da mesma wallet
# A wallet tem R$ 1.000,00 de saldo

k6 run --vus 50 --iterations 50 \
  -e BASE_URL=http://localhost:8081 \
  -e WALLET_ID=$WALLET_JOAO_ID \
  -e TARGET_WALLET_ID=$WALLET_MARIA_ID \
  -e AMOUNT=30.00 \
  k6/scenarios/concurrent-transfer.js

# Verificar invariantes:
# 1. Saldo de João: NUNCA negativo
SALDO_JOAO=$(curl -s "$BASE_URL/api/v1/wallets/$WALLET_JOAO_ID" \
  -H "Authorization: Bearer $TOKEN" | jq '.balance')
echo "Saldo João: R$ $SALDO_JOAO (deve ser >= 0)"

# 2. Soma dos saldos: deve ser igual ao valor inicial total
SALDO_MARIA=$(curl -s "$BASE_URL/api/v1/wallets/$WALLET_MARIA_ID" \
  -H "Authorization: Bearer $TOKEN" | jq '.balance')
SOMA=$(echo "$SALDO_JOAO + $SALDO_MARIA" | bc)
echo "Soma saldos: R$ $SOMA (deve ser R$ 1000.00)"

# 3. Transfers com saldo insuficiente retornaram 422 (não 500)
```

**Cenário 4: Cascading Failure (API externa + rate limiting)**

```bash
# 1. API externa de câmbio fica lenta (simular com tc ou mock)
# Endpoint que consulta câmbio para transações multi-moeda
docker exec api-go tc qdisc add dev eth0 root netem delay 5000ms

# 2. Enviar requests que dependem da API de câmbio
# Esperado:
# - Timeout: 3s (request cortado antes dos 5s de latência)
# - Retry: 1x com backoff
# - Circuit breaker: OPEN após 5 falhas consecutivas
# - Fallback: usar última cotação cacheada no Redis
# - Rate limiter: requests excess retornam 429 Too Many Requests

# 3. Verificar que requests normais (sem câmbio) NÃO são afetados
# - Bulkhead isola o pool de threads/goroutines da API de câmbio
# - Depósitos em BRL continuam funcionando normalmente

# 4. Remover latência e verificar recovery
docker exec api-go tc qdisc del dev eth0 root
# CB: OPEN → HALF_OPEN (após wait time) → CLOSED (após N success)
```

**Cenário 5: Memory Pressure (OOM)**

```bash
# Simular carga alta por tempo prolongado para testar memory leaks
k6 run --vus 100 --duration 10m \
  -e BASE_URL=http://localhost:8081 \
  k6/scenarios/mixed-workload.js

# Monitorar durante o teste:
watch -n 5 'docker stats --no-stream api-go api-spring api-quarkus api-micronaut api-jakarta'

# Verificar:
# - Nenhum serviço recebeu OOMKilled (exit code 137)
# - Memory usage estabiliza (sem leak)
# - GC pauses (Java): < 100ms p99
# - Go: RSS estabiliza após GC
```

**Critérios de aceite:**
- [ ] Cenário 1 (DB Pool): circuit breaker abre e fecha corretamente
- [ ] Cenário 2 (Kafka Down): transações não são perdidas (outbox pattern)
- [ ] Cenário 3 (Race Condition): saldo NUNCA negativo, soma invariante
- [ ] Cenário 4 (Cascading): bulkhead isola falha, fallback funciona
- [ ] Cenário 5 (Memory): nenhum OOM em 10 minutos de carga sustentada
- [ ] Todos os cenários documentados com evidências (screenshots, métricas, logs)

### 3.2 — Chaos Engineering Estruturado com LitmusChaos

Além dos cenários manuais acima, use **LitmusChaos** para executar experimentos de caos de forma declarativa e reprodutível no Kubernetes.

**Setup LitmusChaos:**

```bash
# Instalar LitmusChaos no cluster
kubectl apply -f https://litmuschaos.github.io/litmus/3.0.0/litmus-3.0.0.yaml

# Instalar Chaos Hub (catálogo de experimentos)
kubectl apply -f https://hub.litmuschaos.io/api/chaos/3.0.0/charts/generic/experiments.yaml
```

**Experimento 1: Pod Kill**

```yaml
# chaos/pod-kill.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: wallet-pod-kill
  namespace: digital-wallet
spec:
  appinfo:
    appns: digital-wallet
    applabel: app=wallet-api
    appkind: deployment
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: CHAOS_INTERVAL
              value: "10"
            - name: FORCE
              value: "true"
  # Validação: durante o experimento, verificar que:
  # - Kubernetes recria o pod automaticamente
  # - Requests são redistribuídos para pods saudáveis
  # - Zero downtime (load balancer remove pod unhealthy)
  # - Métricas: error rate não excede 0.5% durante o experimento
```

**Experimento 2: Network Latency**

```yaml
# chaos/network-latency.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: wallet-network-delay
  namespace: digital-wallet
spec:
  appinfo:
    appns: digital-wallet
    applabel: app=wallet-api
    appkind: deployment
  experiments:
    - name: pod-network-latency
      spec:
        components:
          env:
            - name: NETWORK_INTERFACE
              value: "eth0"
            - name: NETWORK_LATENCY
              value: "2000"   # 2000ms de latência
            - name: TOTAL_CHAOS_DURATION
              value: "120"
            - name: DESTINATION_IPS
              value: "postgres-svc"  # latência apenas para o DB
```

**Experimento 3: Disk Fill (via Service Mesh)**

Se estiver usando Istio/Linkerd (Level 7), combine fault injection do mesh com LitmusChaos:

```yaml
# Istio VirtualService — inject 503 em 20% dos requests para exchange-rate-api
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: chaos-exchange-rate
spec:
  hosts:
    - exchange-rate-api
  http:
    - fault:
        abort:
          percentage:
            value: 20
          httpStatus: 503
      route:
        - destination:
            host: exchange-rate-api
```

**Critérios de aceite (LitmusChaos):**
- [ ] LitmusChaos instalado e funcional no cluster K8s
- [ ] Experimento `pod-delete`: app se recupera em < 30s, zero tx loss
- [ ] Experimento `network-latency`: circuit breaker ativa, timeout funciona
- [ ] Experimentos declarativos (YAML) e reprodutíveis via `kubectl apply`
- [ ] Resultados do experimento documentados com métricas do Grafana

### 3.3 — SRE Runbooks e Incident Response

Documente runbooks operacionais para os cenários de falha mais comuns. Estes runbooks são referenciados nos alertas do Prometheus (campo `runbook`).

**Runbook 1: High Error Rate (> 0.1%)**

```markdown
# Runbook: High Error Rate

## Alerta
- Nome: `HighErrorRate`
- Threshold: error rate > 0.1% por 5 minutos
- Severidade: CRITICAL

## Diagnóstico
1. Verificar Grafana dashboard RED — qual endpoint está falhando?
2. Verificar logs (Loki/stdout): `{app="wallet-api"} |= "ERROR" | json`
3. Verificar últimos deploys: `helm history wallet -n digital-wallet`
4. Verificar dependências (DB, Kafka, external APIs):
   - DB: `kubectl exec -it postgres -- pg_isready`
   - Kafka: verificar consumer lag no dashboard
   - Circuit breakers: verificar estado no dashboard

## Ações
- Se causado por deploy recente: `helm rollback wallet <revision>`
- Se causado por DB: verificar connection pool, restart se necessário
- Se causado por Kafka: verificar broker health, consumer lag
- Se causa desconhecida: escalar para on-call senior

## Comunicação
- Atualizar status page
- Notificar canais #incidents no Slack
- Após resolução: criar post-mortem em 48h
```

**Runbook 2: SLO Burn Rate Alto**

```markdown
# Runbook: SLO Burn Rate Alto

## Alerta
- Nome: `SLOBurnRateHigh`
- Threshold: queimando error budget 14.4x mais rápido que o normal (1h window)
- Severidade: CRITICAL

## Diagnóstico
1. Verificar error budget restante (SLO dashboard)
2. Identificar top 5 endpoints por error count
3. Verificar se há deploy em andamento

## Ações
- Se error budget < 20%: congelar deploys não-críticos
- Se error budget < 5%: rollback última mudança, focar 100% em reliability
- Implementar fix → validar em staging → deploy com canary (10% tráfego)

## Prevenção
- Aumentar cobertura de testes de integração
- Adicionar chaos testing no CI pipeline (smoke chaos)
- Review de PRR antes de deploys
```

**Incident Response Drill (Game Day):**

```bash
# Simular incident response completo (1h exercise)
# 1. Injetar falha (sem avisar o time)
kubectl apply -f chaos/pod-kill.yaml

# 2. Time detecta via alertas (< 5 min para detectar)

# 3. Seguir runbook correspondente

# 4. Resolver (< 15 min para mitigar)

# 5. Post-mortem com timeline:
#    - T+0: falha injetada
#    - T+N: alerta disparou
#    - T+N: runbook iniciado
#    - T+N: falha mitigada
#    - T+N: causa raiz confirmada
#    - Total MTTR: X minutos
```

**Critérios de aceite (Runbooks):**
- [ ] ≥ 3 runbooks documentados (cobrindo os alertas mais críticos)
- [ ] Cada runbook referenciado no campo `runbook` do alerta Prometheus
- [ ] Game day executado: incident response drill com timeline documentado
- [ ] MTTR (Mean Time to Recovery) medido: target < 15 min para P1

---

## Parte 4 — Observabilidade em Ação

### 4.1 — Dashboards de Produção

Configure dashboards Grafana que um time de produção usaria no dia-a-dia.

**Dashboard 1: RED Metrics (Request / Error / Duration)**

| Painel | Query PromQL |
|--------|-------------|
| Request Rate | `sum(rate(http_server_requests_seconds_count[5m])) by (method, uri)` |
| Error Rate (%) | `sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) / sum(rate(http_server_requests_seconds_count[5m])) * 100` |
| Latency p50 | `histogram_quantile(0.50, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri))` |
| Latency p95 | `histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri))` |
| Latency p99 | `histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri))` |

**Dashboard 2: USE Metrics (Utilization / Saturation / Errors)**

| Painel | Query PromQL |
|--------|-------------|
| CPU Usage | `rate(process_cpu_seconds_total[5m]) * 100` |
| Memory RSS | `process_resident_memory_bytes / 1024 / 1024` |
| DB Connection Pool Active | `hikaricp_connections_active` / `db_pool_active_connections` |
| DB Connection Pool Pending | `hikaricp_connections_pending` / `db_pool_pending_connections` |
| Kafka Consumer Lag | `kafka_consumer_lag` |
| GC Pause Duration | `jvm_gc_pause_seconds_sum` / `go_gc_duration_seconds` |

**Dashboard 3: Business Metrics**

| Painel | Query PromQL |
|--------|-------------|
| Transações/min por tipo | `sum(rate(wallet_transactions_total[5m])) by (type) * 60` |
| Volume financeiro (R$) | `sum(increase(wallet_transaction_amount_sum[1h])) by (type)` |
| Transações falhadas | `sum(rate(wallet_transactions_total{status="FAILED"}[5m])) by (reason)` |
| Usuários criados/hora | `sum(increase(wallet_users_created_total[1h]))` |

### 4.2 — Alerting Rules

```yaml
# infra/prometheus/alerts.yml
groups:
  - name: wallet-slos
    rules:
      # SLO: 99.9% de requests com sucesso
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          / sum(rate(http_server_requests_seconds_count[5m]))
          > 0.001
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate acima de 0.1% por 5 minutos"
          runbook: "https://wiki.novapay.com/runbooks/high-error-rate"

      # SLO: p95 < 200ms para leituras
      - alert: HighReadLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_server_requests_seconds_bucket{method="GET"}[5m])) by (le)
          ) > 0.200
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "p95 de leitura acima de 200ms"

      # Kafka consumer lag alto
      - alert: KafkaConsumerLag
        expr: kafka_consumer_lag > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Consumer lag acima de 1000 mensagens"

      # Circuit breaker aberto
      - alert: CircuitBreakerOpen
        expr: resilience4j_circuitbreaker_state{state="open"} == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Circuit breaker {{ $labels.name }} está OPEN"

      # DB connection pool saturação
      - alert: DBPoolSaturation
        expr: |
          hikaricp_connections_active / hikaricp_connections_max > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Connection pool > 80% utilizado"
```

### 4.3 — Tracing End-to-End

Valide que o tracing cobre toda a jornada de uma transação.

```
Trace de uma transferência:
┌─────────────────────────────────────────────────────────────────────┐
│ HTTP POST /api/v1/wallets/{id}/transactions/transfer                │
│ ├── span: jwt.validation (0.2ms)                                    │
│ ├── span: rate.limiter.check (0.1ms)                                │
│ ├── span: service.transfer (45ms)                                   │
│ │   ├── span: db.begin_transaction (0.5ms)                          │
│ │   ├── span: db.select wallet_source (1.2ms)                       │
│ │   ├── span: db.select wallet_target (1.0ms)                       │
│ │   ├── span: service.validate_balance (0.1ms)                      │
│ │   ├── span: db.update wallet_source (2.3ms)                       │
│ │   ├── span: db.update wallet_target (2.1ms)                       │
│ │   ├── span: db.insert transaction (1.8ms)                         │
│ │   ├── span: db.insert outbox_event (1.5ms)                        │
│ │   └── span: db.commit (5.2ms)                                     │
│ └── span: kafka.produce transfer_event (3.5ms)                      │
│                                                                      │
│ [Async — Kafka Consumer]                                             │
│ ├── span: kafka.consume transfer_event (0.8ms)                      │
│ └── span: notification.send (12ms)                                   │
│     ├── span: template.render (1.2ms)                                │
│     └── span: email.send (10ms)                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Critérios de aceite:**
- [ ] 3 dashboards Grafana importados e funcionais
- [ ] RED metrics cobrindo todos os endpoints
- [ ] USE metrics cobrindo CPU, memory, DB pool, Kafka lag
- [ ] Business metrics cobrindo transações por tipo e volume
- [ ] AlertRules configuradas e testadas (verificar que disparam)
- [ ] Tracing end-to-end: trace de transferência visível no Jaeger com spans de DB, Kafka, e notification
- [ ] Correlation ID propagado entre request HTTP → Kafka → consumer

---

## Parte 5 — Load Test de Go-Live

### 5.1 — Produção Simulada (4 fases)

Execute um load test que simula o primeiro dia de produção, com tráfego progressivo:

```javascript
// k6/scenarios/go-live-simulation.js
export const options = {
  scenarios: {
    go_live: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        // Fase 1: Abertura gradual (10 min)
        { duration: '2m', target: 10 },   // soft launch
        { duration: '3m', target: 50 },   // ramping up
        { duration: '5m', target: 50 },   // sustain

        // Fase 2: Crescimento orgânico (10 min)
        { duration: '5m', target: 100 },  // growing
        { duration: '5m', target: 100 },  // sustain

        // Fase 3: Pico (campanha de marketing) (10 min)
        { duration: '2m', target: 200 },  // spike!
        { duration: '5m', target: 200 },  // sustain peak
        { duration: '3m', target: 100 },  // cool down

        // Fase 4: Estabilização (5 min)
        { duration: '5m', target: 50 },   // evening
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],
    'wallet_transfer_duration': ['p(95)<800'],
  },
};
```

**Workload distribution:**
- 40% — Consultas (GET users, wallets, transactions)
- 25% — Depósitos
- 20% — Transferências entre wallets
- 10% — Saques
- 5% — Criação de usuários + wallets

### 5.2 — Comparação Final entre Stacks

Execute o go-live simulation em **todas as 5 stacks** e produza a tabela comparativa final:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    GO-LIVE SIMULATION RESULTS                                │
│                                                                              │
│ Métrica              │ Go(Gin) │ Spring │ Quarkus │ Micronaut │ Jakarta     │
│──────────────────────┼─────────┼────────┼─────────┼───────────┼─────────────│
│ Total Requests       │         │        │         │           │             │
│ Failed Requests      │         │        │         │           │             │
│ Error Rate %         │         │        │         │           │             │
│ p50 (ms)             │         │        │         │           │             │
│ p95 (ms)             │         │        │         │           │             │
│ p99 (ms)             │         │        │         │           │             │
│ Max RPS sustained    │         │        │         │           │             │
│ Max RPS at spike     │         │        │         │           │             │
│ Memory peak (MB)     │         │        │         │           │             │
│ CPU peak (%)         │         │        │         │           │             │
│ Kafka lag max        │         │        │         │           │             │
│ CB trips             │         │        │         │           │             │
│ Zero tx loss?        │         │        │         │           │             │
│ SLO compliance (%)   │         │        │         │           │             │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Critérios de aceite:**
- [ ] Go-live simulation (35 min) executado em todas as 5 stacks
- [ ] Error rate < 1% em todas as stacks no sustained load (50 VUs)
- [ ] p95 < 500ms em todas as stacks no sustained load
- [ ] Zero perda de transação (saldo consistente pós-teste)
- [ ] Tabela comparativa completa com métricas reais
- [ ] Evidências: screenshots dos dashboards Grafana durante o teste

---

## Parte 6 — Deploy e Rollback

### 6.1 — Blue-Green Deploy Simulation

```bash
# === BLUE-GREEN DEPLOY ===

# 1. Estado atual: versão v1.0.0 (BLUE) rodando
docker-compose -f docker-compose.production.yml up -d api-go

# 2. Deploy v2.0.0 (GREEN) em paralelo
docker-compose -f docker-compose.production.yml up -d api-go-v2
# api-go-v2 roda na porta 8091

# 3. Smoke test na versão GREEN
./e2e-journey.sh http://localhost:8091
# Se falhar: DELETE green, manter blue

# 4. Switch tráfego: nginx proxy aponta para GREEN
# nginx.conf: upstream { server api-go-v2:8080; }
docker exec nginx nginx -s reload

# 5. Monitorar por 5 min
# Se métricas degradam: instant rollback

# 6. Teardown BLUE
docker stop api-go
```

### 6.2 — Rollback Procedure

```bash
# === ROLLBACK PROCEDURE ===
# Trigger: error rate > 1% OU p99 > 2000ms por > 2 min

# 1. Switch tráfego: proxy volta para BLUE
# nginx.conf: upstream { server api-go:8080; }
docker exec nginx nginx -s reload

# 2. Verificar health
curl -sf http://localhost:8081/api/v1/health/ready

# 3. Stop GREEN
docker stop api-go-v2

# 4. Post-mortem
# - O que causou a falha?
# - Quais métricas alertaram primeiro?
# - Quanto tempo levou o rollback?
```

### 6.3 — Database Migration Rollback

```bash
# Cenário: migration V3.0 introduz bug. Rollback para V2.0.

# Go: golang-migrate
migrate -path ./migrations -database "$DATABASE_URL" down 1

# Java: Flyway
./mvnw flyway:undo -Dflyway.target=2.0

# Verificar: schema está correto?
psql -U wallet_user -d novapay_wallet -c "\dt"

# Verificar: API funciona com schema V2.0?
./e2e-journey.sh http://localhost:8081
```

**Critérios de aceite:**
- [ ] Blue-green deploy executado sem downtime
- [ ] Smoke test automatizado validou GREEN antes de switch
- [ ] Rollback executado em < 2 minutos
- [ ] Database migration rollback funcional
- [ ] Procedimentos documentados em runbook

---

## Parte 7 — Relatório Técnico de Portfólio

### 7.1 — Documento Final

Produza o relatório técnico consolidado **PRODUCTION-REPORT.md**:

```markdown
# NovaPay Digital Wallet — Production Launch Report

## 1. Executive Summary
- 5 stacks implementadas: Go (Gin), Spring Boot, Quarkus, Micronaut, Jakarta EE
- Mesmo domínio, mesmos contratos, mesmos critérios de aceite
- N levels completados em M semanas

## 2. Architecture
- Diagrama C4 (Context + Container)
- Technology decisions (ADRs)
- Trade-offs documentados

## 3. Production Readiness
- PRR checklist resultado
- Security review
- Capacity planning

## 4. Benchmark Results
- Tabela comparativa Level 8 (benchmarks puros)
- Tabela comparativa Level 9 (go-live simulation)
- Análise de diferenças e insights

## 5. Chaos Engineering
- 5 fault scenarios executados
- Resultados e lições aprendidas
- Gaps identificados e corrigidos

## 6. Observability
- Dashboards (screenshots)
- Alerting rules configuradas
- Tracing end-to-end

## 7. Deployment Strategy
- Blue-green deploy com rollback
- Database migration strategy
- CI/CD pipeline

## 8. Lessons Learned
- O que funcionou bem
- O que faria diferente
- Recomendações para próximo projeto

## 9. Stack Recommendation
- Quando usar cada stack (baseado em evidência)
- Matriz de decisão personalizada
- Contextos onde cada stack brilha

## 10. Next Steps
- O que falta para produção real
- Melhorias de performance identificadas
- Roadmap técnico sugerido
```

**Critérios de aceite:**
- [ ] Relatório ≥ 3000 palavras com dados e análise
- [ ] Diagramas de arquitetura incluídos
- [ ] Resultados de benchmark com tabelas e gráficos
- [ ] Análise de chaos engineering com evidências
- [ ] Screenshots de dashboards
- [ ] Recomendação de stack baseada em dados
- [ ] Publicável como artigo técnico/blog post

---

## Definição de Pronto (DoD)

### Infraestrutura
- [ ] `docker-compose.production.yml` com todas as dependências
- [ ] Prometheus + Grafana + Jaeger configurados
- [ ] 3 dashboards Grafana importados
- [ ] Alerting rules configuradas

### Validação E2E
- [ ] Script `e2e-journey.sh` automatizado
- [ ] 16 steps do fluxo validados em todas as 5 stacks
- [ ] Idempotência, atomicidade e invariantes verificados

### Chaos Engineering
- [ ] 5 cenários de fault injection documentados e executados
- [ ] Evidências (logs, métricas, screenshots) salvos
- [ ] Nenhum cenário causou perda de dados

### Load Test
- [ ] Go-live simulation (35 min) executado em 5 stacks
- [ ] Error rate < 1%, p95 < 500ms no sustained load
- [ ] Tabela comparativa final

### Deploy
- [ ] Blue-green deploy sem downtime
- [ ] Rollback em < 2 minutos
- [ ] Database migration rollback testado

### Relatório
- [ ] `PRODUCTION-REPORT.md` ≥ 3000 palavras
- [ ] Portfolio-ready (publicável)

---

## Rubrica de Avaliação (0–100)

| Critério | Peso | 0-25 | 26-50 | 51-75 | 76-100 |
|---|---|---|---|---|---|
| **E2E Journey** | 20% | Parcial, <50% steps | 50-80% steps | 100% steps, 3+ stacks | 100% steps, 5 stacks + automatizado |
| **Chaos Engineering** | 20% | 1 cenário | 2-3 cenários | 4-5 cenários com evidências | 5 cenários + métricas + auto-recovery |
| **Observabilidade** | 15% | Logs básicos | 1 dashboard | 3 dashboards + alertas | Dashboards + alertas + tracing E2E |
| **Load Test** | 15% | Script funcional | 1 stack testada | 3+ stacks testadas | 5 stacks + tabela comparativa |
| **Deploy** | 10% | Docker manual | Docker Compose | Blue-green | Blue-green + rollback + DB migration |
| **Relatório** | 10% | README básico | Relatório parcial | Relatório completo | Portfolio-ready + recomendações |
| **PRR** | 10% | Sem documento | Checklist parcial | PRR completo | PRR + riscos + decisão Go/No-Go |

---

## Como Isso Aparece em Entrevistas

- "Descreva o processo de go-live de um sistema de pagamentos que você construiu"
- "Como você garantiu zero perda de transação durante o deploy?"
- "Quais cenários de chaos engineering você executou e o que aprendeu?"
- "Como você monitoraria um sistema financeiro em produção?"
- "Descreva como você fez rollback de um deploy problemático"
- "Comparou diferentes tecnologias para resolver o mesmo problema? Como?"
- "Como você lida com race conditions em transações concorrentes?"
- "Qual sua estratégia de idempotência para sistemas financeiros?"
- "Se o Kafka ficar fora do ar, o que acontece com seus eventos?"
- "Como você valida que seu sistema é resiliente antes do go-live?"

---

## Commits Sugeridos

```
feat(level-9): add production docker-compose with full observability stack
feat(level-9): add E2E journey script (16 steps, 5 stacks)
feat(level-9): add chaos engineering scenarios (5 fault injection)
feat(level-9): add go-live load test simulation (35 min, 4 phases)
feat(level-9): add Grafana dashboards (RED, USE, Business)
feat(level-9): add Prometheus alerting rules
feat(level-9): add blue-green deploy with rollback procedure
docs(level-9): add Production Readiness Review document
docs(level-9): add PRODUCTION-REPORT.md (portfolio piece)
```
