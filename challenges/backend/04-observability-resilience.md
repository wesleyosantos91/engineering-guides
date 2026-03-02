# Level 4 — Observabilidade e Resiliência

> **Objetivo:** Instrumentar a aplicação com métricas, tracing distribuído e logging estruturado, além de implementar padrões de resiliência (circuit breaker, retry, timeout) para integração com serviços externos.

---

## Objetivo de Aprendizado

- Implementar os 3 pilares de observabilidade: metrics, traces, logs
- Configurar Prometheus + Grafana para métricas
- Implementar tracing distribuído com OpenTelemetry + Jaeger/Tempo
- Configurar logging estruturado (JSON) correlacionado com trace ID
- Implementar health checks (liveness + readiness)
- Integrar com API externa (ex: exchange rates) com circuit breaker
- Aplicar retry com exponential backoff
- Implementar timeout e bulkhead

---

## Escopo Funcional

### Novos Endpoints / Funcionalidades

```
── Health ──
GET /health/live                       → Liveness probe (200)
GET /health/ready                      → Readiness probe (200 / 503)

── Funcionalidade Nova ──
GET /api/v1/exchange-rates/{currency}  → Consulta cotação (API externa)
POST /api/v1/wallets/{id}/convert      → Converter saldo para outra moeda (usa exchange rate)
```

### Regras de Negócio Adicionais

1. **Exchange rate** obtido de API externa (simular com WireMock/mock server)
2. Se API externa falhar → circuit breaker abre → retorna 503
3. Retry automático com backoff (máx. 3 tentativas) para erros transientes
4. Timeout de 2s para chamadas externas
5. Todas as requisições devem gerar trace com `traceId` propagado nos logs

---

## Escopo Técnico

### Métricas

| Métrica | Tipo | Tags |
|---|---|---|
| `http_requests_total` | Counter | `method`, `path`, `status` |
| `http_request_duration_seconds` | Histogram | `method`, `path` |
| `wallet_transactions_total` | Counter | `type` (DEPOSIT/WITHDRAWAL/TRANSFER) |
| `wallet_balance_current` | Gauge | `wallet_id`, `currency` |
| `external_api_calls_total` | Counter | `target`, `status` |
| `circuit_breaker_state` | Gauge | `name`, `state` (CLOSED/OPEN/HALF_OPEN) |

### Tracing

- Cada request gera um **span** com `traceId` + `spanId`
- Propagar context entre service calls
- Spans para: HTTP handler, DB query, external API call
- Exportar para Jaeger/Tempo via OTLP

### Logging

- Formato JSON em produção
- Campos obrigatórios: `timestamp`, `level`, `msg`, `traceId`, `spanId`, `requestId`
- Correlation: trace ID presente em log E no response header

### Frameworks de Métricas — RED / USE / VALET / Golden Signals

Organize suas métricas em frameworks consagrados para entender a saúde do sistema de forma estruturada:

**RED Method** (focado em serviços/microserviços — Tom Wilkie):

| Dimensão | Métrica Digital Wallet | PromQL Exemplo |
|---|---|---|
| **R**ate | Requests por segundo | `sum(rate(http_requests_total[5m]))` |
| **E**rrors | Taxa de erro (%) | `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100` |
| **D**uration | Latência p50/p95/p99 | `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))` |

**USE Method** (focado em recursos de infraestrutura — Brendan Gregg):

| Dimensão | Recurso | Métrica |
|---|---|---|
| **U**tilization | CPU | `rate(process_cpu_seconds_total[5m]) * 100` |
| **S**aturation | DB Connection Pool | `hikaricp_connections_pending` / `db_pool_pending_connections` |
| **E**rrors | Disco, rede, memória | `node_filesystem_errors_total`, OOM events |

**VALET Method** (visão de negócio):

| Dimensão | Aplicação no Wallet |
|---|---|
| **V**olume | Transações/min, Volume financeiro (R$) |
| **A**vailability | Uptime %, readiness check success rate |
| **L**atency | p50/p95/p99 por endpoint e por tipo de transação |
| **E**rrors | Error rate por tipo (4xx, 5xx), falhas de negócio (saldo insuficiente) |
| **T**ickets | Incidentes abertos, alertas disparados |

**Golden Signals** (Google SRE Book):

| Sinal | Métrica wallet | Alerta |
|---|---|---|
| **Latency** | `http_request_duration_seconds` | p99 > 500ms por 5min |
| **Traffic** | `http_requests_total` rate | Queda > 50% em 5min |
| **Errors** | `http_requests_total{status=~"5.."}` | Error rate > 0.1% |
| **Saturation** | CPU, memory, DB pool, Kafka lag | Pool > 80%, lag > 1000 |

> **Diretriz:** Use RED para monitorar seus **serviços**, USE para os **recursos** (CPU, DB, Kafka), VALET para reportar **métricas de negócio** para stakeholders, e Golden Signals para compor **SLIs/SLOs**.

---

### SLIs / SLOs — Service Level Indicators & Objectives

Defina SLIs e SLOs para o Digital Wallet como prática de SRE:

| SLI (Indicador) | Métrica | SLO (Objetivo) |
|---|---|---|
| **Disponibilidade** | `1 - (requests 5xx / total requests)` | ≥ 99.9% (30d rolling) |
| **Latência de leitura** | p95 de `GET /wallets`, `GET /transactions` | < 200ms |
| **Latência de escrita** | p95 de `POST /transactions/*` | < 500ms |
| **Durabilidade de tx** | Transações persistidas / transações aceitas | 100% (zero loss) |
| **Freshness** | Lag máximo do consumer Kafka | < 30 segundos |

**Error Budget:**
```
Error Budget (30d) = 1 − SLO = 0.1% = ~43 min de indisponibilidade/mês

Se error budget esgotado → congelar deploys, focar em confiabilidade
Se error budget saudável → pode assumir mais risco (features, refactoring)
```

**Burn Rate Alert (Multi-Window):**
```yaml
# Alerta se queimando error budget 14.4x mais rápido que o normal (1h window)
- alert: SLOBurnRateHigh
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[1h]))
      / sum(rate(http_requests_total[1h]))
    ) > (14.4 * 0.001)
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Queimando error budget 14.4x acima do esperado"
```

> **Referência:** Google SRE Book — Cap. 4 (Service Level Objectives), Cap. 5 (Eliminating Toil)

---

### Ferramentas por Stack

| Aspecto | Go (Gin) | Spring Boot | Quarkus | Micronaut | Jakarta EE |
|---|---|---|---|---|---|
| **Métricas** | prometheus/client_golang | Micrometer + Actuator | Micrometer (SmallRye) | Micrometer (built-in) | MicroProfile Metrics |
| **Tracing** | go.opentelemetry.io/otel | Micrometer Tracing + OTel | OpenTelemetry (SmallRye) | Micronaut Tracing + OTel | MicroProfile Telemetry |
| **Logging** | uber-go/zap | Logback + Logstash encoder | JBoss Logging / SLF4J | Logback | JBoss Logging |
| **Log Aggregation** | Loki + Promtail | Loki + Logback appender | Loki + Quarkus logging | Loki + Logback appender | Loki + JBoss Log handler |
| **Health** | custom `/health/*` | Spring Actuator | SmallRye Health | Micronaut Management | MicroProfile Health |
| **Circuit Breaker** | sony/gobreaker | Resilience4j | SmallRye Fault Tolerance | Micronaut Retry + CB | MicroProfile Fault Tolerance |
| **Retry** | cenkalti/backoff | Resilience4j `@Retry` | `@Retry` | `@Retryable` | `@Retry` |
| **Timeout** | `context.WithTimeout` | Resilience4j `@TimeLimiter` | `@Timeout` | `@Timeout` | `@Timeout` |
| **Bulkhead** | semaphore pattern | Resilience4j `@Bulkhead` | `@Bulkhead` | — | `@Bulkhead` |
| **HTTP Client** | net/http + retryable | WebClient / RestClient | REST Client Reactive | Micronaut HTTP Client | JAX-RS Client (MicroProfile Rest Client) |

---

## Critérios de Aceite

- [ ] `/health/live` retorna 200 (sempre)
- [ ] `/health/ready` retorna 503 se banco indisponível
- [ ] Métricas expostas em `/metrics` (formato Prometheus)
- [ ] Grafana dashboard visualiza: throughput, latência p50/p95/p99, error rate
- [ ] Cada request gera trace visível no Jaeger/Tempo
- [ ] Logs contêm `traceId` correlacionado com o trace
- [ ] Circuit breaker abre após N falhas consecutivas
- [ ] Circuit breaker OPEN retorna 503 (Service Unavailable)
- [ ] Circuit breaker HALF_OPEN permite 1 tentativa de recuperação
- [ ] Retry funciona para erros transitórios (500, timeout, connection refused)
- [ ] Retry NÃO acontece para erros de negócio (400, 404, 422)
- [ ] Timeout de 2s cancela chamada externa lenta

---

## Definição de Pronto (DoD)

- [ ] `docker-compose.yml` inclui: PostgreSQL, Prometheus, Grafana, Jaeger/Tempo
- [ ] Grafana dashboard importável (JSON) com ≥ 4 painéis
- [ ] Traces visíveis no Jaeger com spans de handler, service, DB, external API
- [ ] Logs em JSON com traceId em produção
- [ ] Testes de integração para circuit breaker (WireMock simulando falhas)
- [ ] Documentação dos endpoints de health/metrics
- [ ] Commit: `feat(level-4): add observability and resilience patterns`

---

## Checklist

### Go (Gin)

- [ ] `internal/platform/observability/metrics.go` — registro de métricas Prometheus
- [ ] `internal/platform/observability/tracing.go` — setup OpenTelemetry (OTLP exporter)
- [ ] `internal/platform/observability/logger.go` — Zap logger com traceId em fields
- [ ] `internal/middleware/metrics.go` — middleware que registra `http_requests_total`, `http_request_duration`
- [ ] `internal/middleware/tracing.go` — middleware que cria span + propaga context
- [ ] `internal/handler/health.go` — `/health/live`, `/health/ready`
- [ ] `internal/platform/httpclient/exchange.go` — HTTP client com retry + timeout
- [ ] `internal/platform/resilience/circuitbreaker.go` — gobreaker wrapper
- [ ] `internal/service/exchange.go` — `ExchangeRateService` com circuit breaker
- [ ] `internal/handler/exchange.go` — handler para consulta de cotação

### Java — Spring Boot

- [ ] `pom.xml` — deps: `spring-boot-starter-actuator`, `micrometer-tracing-bridge-otel`, `opentelemetry-exporter-otlp`
- [ ] `application.yml` — Actuator endpoints: health, metrics, prometheus
- [ ] `application.yml` — Tracing: `management.tracing.sampling.probability=1.0`
- [ ] `infrastructure/client/ExchangeRateClient.java` — `RestClient` para API externa
- [ ] `infrastructure/resilience/ExchangeRateClientResilient.java` — `@CircuitBreaker`, `@Retry`, `@TimeLimiter`
- [ ] `api/rest/v1/controller/ExchangeRateController.java`
- [ ] `domain/service/ExchangeRateService.java`
- [ ] `core/config/ResilienceConfig.java` — configuração Resilience4j
- [ ] `logback-spring.xml` — formato JSON com traceId/spanId

### Quarkus

- [ ] `application.properties` — `quarkus.smallrye-health.*`, `quarkus.micrometer.*`
- [ ] `@RegisterRestClient` para exchange rate API
- [ ] `@CircuitBreaker`, `@Retry`, `@Timeout` do SmallRye Fault Tolerance
- [ ] `@Liveness` / `@Readiness` health check procedures

### Micronaut

- [ ] `application.yml` — `micronaut.metrics.*`, `tracing.opentelemetry.*`
- [ ] `@Client` declarativo para exchange rate API
- [ ] `@CircuitBreaker`, `@Retryable`, `@Timeout` built-in
- [ ] `HealthIndicator` para readiness customizado

### Jakarta EE

- [ ] `@Liveness` / `@Readiness` — MicroProfile Health
- [ ] `@CircuitBreaker`, `@Retry`, `@Timeout`, `@Bulkhead` — MicroProfile Fault Tolerance
- [ ] `@RegisterRestClient` — MicroProfile Rest Client
- [ ] `@Metric`, `@Counted`, `@Timed` — MicroProfile Metrics

---

## Tarefas Sugeridas por Stack

### Go — Circuit Breaker com gobreaker

```go
// internal/platform/resilience/circuitbreaker.go
func NewCircuitBreaker(name string) *gobreaker.CircuitBreaker {
    return gobreaker.NewCircuitBreaker(gobreaker.Settings{
        Name:        name,
        MaxRequests: 1,                         // em HALF_OPEN
        Interval:    10 * time.Second,          // janela de contagem
        Timeout:     30 * time.Second,          // tempo em OPEN antes de HALF_OPEN
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            return counts.ConsecutiveFailures >= 5
        },
        OnStateChange: func(name string, from, to gobreaker.State) {
            logger.Warn("circuit breaker state change",
                zap.String("name", name),
                zap.String("from", from.String()),
                zap.String("to", to.String()),
            )
        },
    })
}
```

### Go — HTTP Client com Retry

```go
// internal/platform/httpclient/retryable.go
func DoWithRetry(ctx context.Context, cb *gobreaker.CircuitBreaker, fn func() (*http.Response, error)) (*http.Response, error) {
    var resp *http.Response
    operation := func() error {
        result, err := cb.Execute(func() (interface{}, error) {
            ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
            defer cancel()
            _ = ctx
            return fn()
        })
        if err != nil {
            return err
        }
        resp = result.(*http.Response)
        if resp.StatusCode >= 500 {
            return fmt.Errorf("server error: %d", resp.StatusCode)
        }
        return nil
    }

    b := backoff.WithMaxRetries(backoff.NewExponentialBackOff(), 3)
    err := backoff.Retry(operation, b)
    return resp, err
}
```

### Spring Boot — Resilience4j

```java
@Service
public class ExchangeRateService {

    private final ExchangeRateClient client;

    @CircuitBreaker(name = "exchangeRate", fallbackMethod = "fallbackRate")
    @Retry(name = "exchangeRate")
    @TimeLimiter(name = "exchangeRate")
    public CompletableFuture<ExchangeRate> getRate(String currency) {
        return CompletableFuture.supplyAsync(() -> client.getRate(currency));
    }

    private CompletableFuture<ExchangeRate> fallbackRate(String currency, Exception ex) {
        // Retornar última cotação conhecida ou lançar ServiceUnavailableException
        throw new ServiceUnavailableException("Exchange rate service unavailable", ex);
    }
}
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      exchangeRate:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
  retry:
    instances:
      exchangeRate:
        max-attempts: 3
        wait-duration: 500ms
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
  timelimiter:
    instances:
      exchangeRate:
        timeout-duration: 2s
```

---

## docker-compose.yml (observabilidade)

```yaml
services:
  postgres:
    image: postgres:16-alpine
    # ... (mesmo do Level 2)

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./infra/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - ./infra/grafana/dashboards:/var/lib/grafana/dashboards
      - ./infra/grafana/provisioning:/etc/grafana/provisioning

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"   # UI
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
```

```yaml
# infra/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'digital-wallet'
    metrics_path: '/metrics'     # ou '/actuator/prometheus' para Spring
    static_configs:
      - targets: ['host.docker.internal:8080']
```

---

## Extensões Opcionais

- [ ] Implementar rate limiting (token bucket) como middleware
- [ ] Adicionar alertas no Grafana (ex: error rate > 5%)
- [ ] Implementar custom span attributes (userId, walletId)
- [ ] Adicionar baggage propagation para correlation IDs
- [ ] Integrar com **Loki + Promtail/Alloy** para agregação de logs centralizada (LogQL queries)
- [ ] Implementar **SLI/SLO dashboard** completo com error budget tracking e burn rate alerts (multi-window)
- [ ] Configurar **Grafana Alerting** com notificações (Slack, email) para SLO violations
- [ ] Implementar **exemplars** no Prometheus para vincular métricas a traces específicos
- [ ] Adicionar **Grafana Tempo** como backend alternativo ao Jaeger para tracing

---

## Erros Comuns

| Erro | Stack | Como evitar |
|---|---|---|
| Não propagar context em chamadas downstream | Todos | Sempre passar `ctx` / `Span.current()` |
| Circuit breaker nunca abre | Todos | Verificar threshold e window size — testar com fault injection |
| Retry em erros de negócio (400, 422) | Todos | Retry apenas para erros transitórios (5xx, timeout) |
| Logs sem traceId | Todos | Configurar MDC bridge (Java) ou log fields (Go) |
| Métricas com cardinalidade alta | Todos | Não usar userId/email como tag — usar apenas categorias |
| Health check retorna 200 com banco fora | Todos | Readiness probe deve verificar conexão real |
| Prometheus scrape muito frequente (1s) | Todos | 15s é suficiente para maioria dos casos |

---

## Como Isso Aparece em Entrevistas

- "Quais são os 3 pilares da observabilidade? Como se complementam?"
- "O que é um circuit breaker? Desenhe o diagrama de estados"
- "Qual a diferença entre liveness e readiness probe?"
- "Como funciona o exponential backoff? Quando NÃO fazer retry?"
- "O que é cardinalidade em métricas e por que importa?"
- "Como você correlaciona logs com traces?"
- "O que é OpenTelemetry e qual a diferença para Prometheus?"
- "Explique a diferença entre Micrometer e Resilience4j no Spring"
- "Como funciona o SmallRye Fault Tolerance comparado ao Resilience4j?"

---

## Comandos de Execução

```bash
# Subir stack de observabilidade
docker-compose up -d postgres prometheus grafana jaeger

# Verificar targets no Prometheus
open http://localhost:9090/targets

# Grafana dashboards
open http://localhost:3000  # admin/admin

# Jaeger traces
open http://localhost:16686

# Gerar carga para visualizar métricas
for i in $(seq 1 100); do
  curl -s http://localhost:8080/api/v1/users > /dev/null
done

# Testar circuit breaker (simular falha da API externa)
docker-compose stop exchange-rate-mock
curl http://localhost:8080/api/v1/exchange-rates/USD  # deve retornar 503 após N falhas
```
