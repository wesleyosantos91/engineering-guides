# Level 0 — Observability Foundations

> **Objetivo:** Implementar os três pilares de observabilidade (logs, métricas, traces básicos) num serviço, com structured logging, métricas RED/USE, health checks e dashboards Grafana.

---

## Objetivo de Aprendizado

- Entender a diferença entre monitoramento e observabilidade
- Implementar structured logging (JSON) com correlation ID
- Expor métricas no formato Prometheus (counters, histograms, gauges)
- Aplicar modelos RED (Request, Error, Duration) e USE (Utilization, Saturation, Errors)
- Configurar health checks (liveness + readiness)
- Criar dashboards Grafana com Golden Signals
- Implementar correlation: mesmo `request_id` em logs e response headers
- Subir stack de observabilidade local via Docker Compose

---

## Escopo Funcional

### Serviço: Order Service (base)

```
── Health ──
GET  /health/live                       → Liveness probe (200 OK)
GET  /health/ready                      → Readiness probe (200 / 503)

── CRUD de Orders ──
POST /api/v1/orders                     → Criar pedido
GET  /api/v1/orders/{id}                → Buscar pedido por ID
GET  /api/v1/orders                     → Listar pedidos (paginado)
PATCH /api/v1/orders/{id}/status        → Atualizar status do pedido

── Métricas ──
GET  /metrics                           → Prometheus metrics endpoint
```

### Regras de Negócio

1. Pedido nasce com status `CREATED`
2. Status válidos: `CREATED → PROCESSING → PAID → SHIPPED → DELIVERED → CANCELLED`
3. Transições inválidas retornam 422 com ProblemDetail (RFC 9457)
4. Todo log deve ser JSON com campos: `timestamp`, `level`, `msg`, `request_id`, `service`
5. Todo response deve incluir header `X-Request-Id`

---

## Escopo Técnico

### Structured Logging

| Campo | Tipo | Obrigatório | Exemplo |
|-------|------|-------------|---------|
| `timestamp` | ISO 8601 UTC | ✅ | `2026-03-01T10:15:30.123Z` |
| `level` | string | ✅ | `INFO`, `ERROR`, `WARN` |
| `msg` | string | ✅ | `Order created successfully` |
| `service` | string | ✅ | `order-service` |
| `request_id` | UUID | ✅ | `550e8400-e29b-41d4-a716-446655440000` |
| `method` | string | ✅ (em HTTP) | `POST` |
| `path` | string | ✅ (em HTTP) | `/api/v1/orders` |
| `status_code` | int | ✅ (em HTTP) | `201` |
| `duration_ms` | float | ✅ (em HTTP) | `45.3` |
| `error` | string | Condicional | `insufficient inventory` |
| `user_id` | string | Condicional | `user-42` |

```json
{
  "timestamp": "2026-03-01T10:15:30.123Z",
  "level": "INFO",
  "msg": "Order created",
  "service": "order-service",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "method": "POST",
  "path": "/api/v1/orders",
  "status_code": 201,
  "duration_ms": 45.3,
  "order_id": "ord-999",
  "items_count": 3
}
```

### Métricas (Modelo RED + USE)

| Métrica | Tipo | Labels | Modelo |
|---|---|---|---|
| `http_requests_total` | Counter | `method`, `path`, `status` | RED — Rate |
| `http_request_duration_seconds` | Histogram | `method`, `path` | RED — Duration |
| `http_request_errors_total` | Counter | `method`, `path`, `error_type` | RED — Errors |
| `orders_created_total` | Counter | `status` | Business |
| `orders_status_transitions_total` | Counter | `from_status`, `to_status` | Business |
| `db_connections_active` | Gauge | `pool_name` | USE — Utilization |
| `db_connections_max` | Gauge | `pool_name` | USE — Saturation |
| `db_query_duration_seconds` | Histogram | `operation`, `table` | USE — Duration |
| `app_info` | Gauge (constante 1) | `version`, `go_version`/`jvm_version` | Info |

### Health Checks

| Endpoint | Tipo | Verifica | Response |
|----------|------|----------|----------|
| `/health/live` | Liveness | App está rodando | Always 200 |
| `/health/ready` | Readiness | DB conectado, dependencies OK | 200 ou 503 |

```json
// GET /health/ready — 200
{
  "status": "UP",
  "checks": {
    "database": { "status": "UP", "response_time_ms": 2 },
    "disk": { "status": "UP", "free_gb": 42.5 }
  }
}

// GET /health/ready — 503
{
  "status": "DOWN",
  "checks": {
    "database": { "status": "DOWN", "error": "connection refused" },
    "disk": { "status": "UP", "free_gb": 42.5 }
  }
}
```

### Ferramentas por Stack

| Aspecto | Go (Gin) | Spring Boot | Quarkus | Micronaut | Jakarta EE |
|---|---|---|---|---|---|
| **Logging** | uber-go/zap | Logback + logstash-encoder | JBoss Logging + JSON formatter | Logback + logstash-encoder | JBoss Logging |
| **Métricas** | prometheus/client_golang | Micrometer + Actuator `/actuator/prometheus` | SmallRye Metrics (Micrometer) | Micrometer built-in | MicroProfile Metrics 5.1 |
| **Health** | custom handlers | Spring Actuator `/actuator/health` | SmallRye Health (`@Liveness`, `@Readiness`) | Micronaut Management `/health` | MicroProfile Health 4.0 |
| **Request ID** | middleware custom | `Filter` / `HandlerInterceptor` | `ContainerRequestFilter` (JAX-RS) | `HttpServerFilter` | `ContainerRequestFilter` |
| **Config** | `caarlos0/env` | `application.yml` | `application.properties` | `application.yml` | `microprofile-config.properties` |

---

## Critérios de Aceite

### Logging
- [ ] Todos os logs em formato JSON (zero log texto livre em produção)
- [ ] `request_id` presente em todo log de request HTTP
- [ ] `request_id` retornado no header `X-Request-Id`
- [ ] Log levels corretos: INFO para operações normais, ERROR para falhas, WARN para retries
- [ ] Nenhum dado sensível (passwords, tokens) nos logs
- [ ] Log de cada request HTTP com: method, path, status, duration_ms

### Métricas
- [ ] Endpoint `/metrics` expõe métricas no formato Prometheus
- [ ] `http_requests_total` com labels `method`, `path`, `status`
- [ ] `http_request_duration_seconds` como histogram com buckets adequados
- [ ] Pelo menos 2 métricas de negócio (orders_created, status_transitions)
- [ ] Métricas de DB pool (connections active/max)
- [ ] Métrica `app_info` com version da aplicação

### Health
- [ ] `/health/live` retorna 200 (sempre, se o processo está rodando)
- [ ] `/health/ready` retorna 503 se banco de dados indisponível
- [ ] Response body JSON com detalhes dos checks

### Infra
- [ ] `docker-compose.yml` com: app, PostgreSQL, Prometheus, Grafana
- [ ] `prometheus.yml` configurado para scrape do app
- [ ] Grafana dashboard com pelo menos 6 painéis (throughput, latência p50/p95/p99, error rate, status codes, DB connections, business metrics)

---

## Definição de Pronto (DoD)

- [ ] Todos os critérios de aceite ✅
- [ ] `docker-compose up` sobe toda a stack sem erros
- [ ] Grafana dashboard importável (JSON exportado no repositório)
- [ ] Logs visíveis e parseáveis (structured JSON)
- [ ] Prometheus targets UP e métricas coletadas
- [ ] Testes de integração para health checks
- [ ] README com instruções de setup e screenshots dos dashboards
- [ ] Commit: `feat(level-0): implement observability foundations - structured logging, metrics, health`

---

## Checklist

### Go (Gin)

- [ ] `internal/platform/logger/logger.go` — Zap logger configurado (JSON, nível por env)
- [ ] `internal/middleware/request_id.go` — Middleware que gera/propaga `X-Request-Id`
- [ ] `internal/middleware/logging.go` — Middleware que loga cada request (method, path, status, duration)
- [ ] `internal/middleware/metrics.go` — Middleware que registra `http_requests_total` e `http_request_duration_seconds`
- [ ] `internal/handler/health.go` — Handlers para `/health/live` e `/health/ready`
- [ ] `internal/handler/order.go` — CRUD de orders
- [ ] `internal/platform/metrics/metrics.go` — Registry Prometheus + métricas de negócio
- [ ] `docker-compose.yml` — App + PostgreSQL + Prometheus + Grafana
- [ ] `prometheus.yml` — Scrape config
- [ ] `grafana/dashboards/order-service.json` — Dashboard importável

### Spring Boot

- [ ] `pom.xml` — deps: `spring-boot-starter-actuator`, `micrometer-registry-prometheus`
- [ ] `application.yml` — Actuator: health, prometheus, info endpoints
- [ ] `logback-spring.xml` — JSON format com logstash-encoder
- [ ] `infrastructure/filter/RequestIdFilter.java` — Filter para `X-Request-Id`
- [ ] `infrastructure/config/MetricsConfig.java` — Custom MeterBinder para métricas de negócio
- [ ] `api/rest/v1/controller/OrderController.java` — CRUD
- [ ] `api/rest/v1/controller/HealthController.java` — Custom health (se necessário além do Actuator)

### Quarkus

- [ ] `pom.xml` — deps: `quarkus-smallrye-health`, `quarkus-micrometer-registry-prometheus`
- [ ] `application.properties` — Health, metrics config
- [ ] `infrastructure/filter/RequestIdFilter.java` — `ContainerRequestFilter` + `ContainerResponseFilter`
- [ ] `api/rest/v1/resource/OrderResource.java` — CRUD (JAX-RS)
- [ ] `infrastructure/health/DatabaseHealthCheck.java` — `@Readiness` custom

### Micronaut

- [ ] `build.gradle` / `pom.xml` — deps: `micronaut-management`, `micronaut-micrometer`
- [ ] `application.yml` — Health, metrics endpoints
- [ ] `infrastructure/filter/RequestIdFilter.java` — `HttpServerFilter`
- [ ] `api/rest/v1/controller/OrderController.java` — CRUD
- [ ] `infrastructure/health/DatabaseHealthIndicator.java` — Custom `HealthIndicator`

### Jakarta EE

- [ ] `pom.xml` — deps: MicroProfile Health 4.0, MicroProfile Metrics 5.1
- [ ] `infrastructure/filter/RequestIdFilter.java` — `ContainerRequestFilter`
- [ ] `api/rest/v1/resource/OrderResource.java` — CRUD (JAX-RS 4.0)
- [ ] `infrastructure/health/DatabaseHealth.java` — `@Readiness` com `HealthCheck`
- [ ] `infrastructure/metrics/OrderMetrics.java` — `@Counted`, `@Timed`

---

## Tarefas Sugeridas por Stack

### Go — Structured Logging com Zap

```go
// internal/platform/logger/logger.go
package logger

import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
    "os"
)

func New() *zap.Logger {
    env := os.Getenv("APP_ENV")

    var config zap.Config
    if env == "production" {
        config = zap.NewProductionConfig()
        config.EncoderConfig.TimeKey = "timestamp"
        config.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder
    } else {
        config = zap.NewDevelopmentConfig()
    }

    logger, _ := config.Build()
    return logger
}
```

```go
// internal/middleware/request_id.go
package middleware

import (
    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
)

const RequestIDKey = "request_id"
const RequestIDHeader = "X-Request-Id"

func RequestID() gin.HandlerFunc {
    return func(c *gin.Context) {
        id := c.GetHeader(RequestIDHeader)
        if id == "" {
            id = uuid.New().String()
        }
        c.Set(RequestIDKey, id)
        c.Header(RequestIDHeader, id)
        c.Next()
    }
}
```

```go
// internal/middleware/logging.go
package middleware

import (
    "time"
    "github.com/gin-gonic/gin"
    "go.uber.org/zap"
)

func Logging(logger *zap.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        duration := time.Since(start)

        reqID, _ := c.Get(RequestIDKey)

        logger.Info("HTTP request",
            zap.String("request_id", reqID.(string)),
            zap.String("method", c.Request.Method),
            zap.String("path", c.Request.URL.Path),
            zap.Int("status", c.Writer.Status()),
            zap.Float64("duration_ms", float64(duration.Milliseconds())),
            zap.String("client_ip", c.ClientIP()),
        )
    }
}
```

```go
// internal/middleware/metrics.go
package middleware

import (
    "strconv"
    "time"
    "github.com/gin-gonic/gin"
    "github.com/prometheus/client_golang/prometheus"
)

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
        },
        []string{"method", "path"},
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal, httpRequestDuration)
}

func Metrics() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        duration := time.Since(start).Seconds()

        status := strconv.Itoa(c.Writer.Status())
        httpRequestsTotal.WithLabelValues(c.Request.Method, c.FullPath(), status).Inc()
        httpRequestDuration.WithLabelValues(c.Request.Method, c.FullPath()).Observe(duration)
    }
}
```

### Spring Boot — Structured Logging + Actuator

```xml
<!-- pom.xml — dependências essenciais -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>8.0</version>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus, info
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
  health:
    db:
      enabled: true
  metrics:
    tags:
      application: order-service
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 50ms, 100ms, 200ms, 500ms, 1s
```

```xml
<!-- logback-spring.xml -->
<configuration>
    <springProfile name="production">
        <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <fieldNames>
                    <timestamp>timestamp</timestamp>
                    <message>msg</message>
                    <levelValue>[ignore]</levelValue>
                </fieldNames>
                <customFields>{"service":"order-service"}</customFields>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="JSON" />
        </root>
    </springProfile>
</configuration>
```

```java
// infrastructure/filter/RequestIdFilter.java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestIdFilter extends OncePerRequestFilter {

    private static final String REQUEST_ID_HEADER = "X-Request-Id";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain) throws ServletException, IOException {
        String requestId = request.getHeader(REQUEST_ID_HEADER);
        if (requestId == null || requestId.isBlank()) {
            requestId = UUID.randomUUID().toString();
        }

        MDC.put("request_id", requestId);
        response.setHeader(REQUEST_ID_HEADER, requestId);

        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove("request_id");
        }
    }
}
```

### Quarkus — Health + Metrics

```java
// infrastructure/health/DatabaseHealthCheck.java
@Readiness
@ApplicationScoped
public class DatabaseHealthCheck implements HealthCheck {

    @Inject
    DataSource dataSource;

    @Override
    public HealthCheckResponse call() {
        try (Connection conn = dataSource.getConnection()) {
            conn.createStatement().execute("SELECT 1");
            return HealthCheckResponse.up("database");
        } catch (Exception e) {
            return HealthCheckResponse.down("database");
        }
    }
}
```

```properties
# application.properties
quarkus.smallrye-health.root-path=/health
quarkus.micrometer.export.prometheus.path=/metrics
quarkus.micrometer.binder.http-server.enabled=true
quarkus.log.console.json=true
quarkus.log.console.json.fields.timestamp.enabled=true
```

### Docker Compose (todos os stacks)

```yaml
# docker-compose.yml
services:
  order-service:
    build: .
    ports:
      - "8080:8080"
    environment:
      - APP_ENV=production
      - DATABASE_URL=postgres://user:pass@postgres:5432/orders
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:17
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d orders"]
      interval: 5s
      retries: 5
    ports:
      - "5432:5432"

  prometheus:
    image: prom/prometheus:v3.2.0
    volumes:
      - ./infra/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:11.5.0
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - ./infra/grafana/provisioning:/etc/grafana/provisioning
      - ./infra/grafana/dashboards:/var/lib/grafana/dashboards
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
```

```yaml
# infra/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'order-service'
    metrics_path: /metrics
    static_configs:
      - targets: ['order-service:8080']
        labels:
          service: order-service
          environment: local
```

---

## Extensões Opcionais

- [ ] Configurar log rotation e size limits
- [ ] Implementar `app_build_info` metric com git commit hash
- [ ] Adicionar métricas customizadas de negócio: `orders_total_value_cents` (histogram)
- [ ] Criar alerting rules no Prometheus para error rate > 5%
- [ ] Implementar `/metrics` com exemplars (link para trace_id — preparação para Level 1)
- [ ] Configurar Grafana alerting (Grafana Alerting 9+)
- [ ] Dashboard com variáveis (nome do serviço como template variable)
