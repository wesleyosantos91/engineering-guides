# Level 3 — OpenTelemetry Deep Dive

> **Objetivo:** Configurar pipelines completas do OpenTelemetry Collector (receivers → processors → exporters), implementar auto-instrumentação, aplicar semantic conventions, configurar sampling strategies (head-based e tail-based) e integrar spans, metrics e logs em um pipeline unificado.

---

## Objetivo de Aprendizado

- Entender a arquitetura do OTel Collector (Agent vs Gateway)
- Configurar receivers, processors e exporters declarativamente
- Implementar auto-instrumentação (Go otelgin/otelhttp, Java Agent)
- Mapear semantic conventions (HTTP, RPC, Database, Messaging)
- Implementar head-based sampling (TraceIDRatioBased) no SDK
- Implementar tail-based sampling no Collector Gateway (por latência, erro, atributo)
- Gerar métricas a partir de spans (spanmetrics connector)
- Processar e enriquecer telemetria no Collector (attributes, resource, batch)
- Multi-tenant pipeline com routing

---

## Escopo Funcional

### Arquitetura OTel Collector (Agent + Gateway)

```
┌─────────────────────────────────────────────────────────────────────┐
│                 OTEL COLLECTOR ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐        │
│  │  Order     │  │  Payment  │  │ Inventory │  │  Notif    │        │
│  │  Service   │  │  Service  │  │  Service  │  │  Service  │        │
│  │ (OTel SDK) │  │ (OTel SDK)│  │(OTel SDK) │  │(OTel SDK) │        │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘        │
│        │               │               │               │             │
│        ▼               ▼               ▼               ▼             │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │           OTel Collector — AGENT (DaemonSet)             │        │
│  │                                                          │        │
│  │  Receivers:                                              │        │
│  │   ├── otlp (gRPC :4317, HTTP :4318)                     │        │
│  │   └── prometheus (:8888 self-metrics)                    │        │
│  │                                                          │        │
│  │  Processors:                                             │        │
│  │   ├── memory_limiter (limit: 512MiB, spike: 128MiB)     │        │
│  │   ├── resourcedetection (env, system, docker)            │        │
│  │   ├── attributes/add-env (environment=staging)           │        │
│  │   └── batch (size: 1024, timeout: 5s)                    │        │
│  │                                                          │        │
│  │  Exporters:                                              │        │
│  │   └── otlp → Gateway (:4317)                            │        │
│  └──────────────────────────────────────────────────────────┘        │
│                              │                                       │
│                              ▼                                       │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │           OTel Collector — GATEWAY (Deployment)          │        │
│  │                                                          │        │
│  │  Receivers:                                              │        │
│  │   └── otlp (gRPC :4317)                                 │        │
│  │                                                          │        │
│  │  Processors:                                             │        │
│  │   ├── tail_sampling                                      │        │
│  │   │    ├── policy: always_sample errors (status=ERROR)   │        │
│  │   │    ├── policy: sample slow traces (latency > 1s)     │        │
│  │   │    └── policy: probabilistic 10% for rest            │        │
│  │   ├── transform (enrich attributes)                      │        │
│  │   └── batch                                              │        │
│  │                                                          │        │
│  │  Connectors:                                             │        │
│  │   └── spanmetrics → Prometheus (RED from traces)         │        │
│  │                                                          │        │
│  │  Exporters:                                              │        │
│  │   ├── otlp → Jaeger/Tempo (traces)                      │        │
│  │   ├── prometheusremotewrite → Prometheus (metrics)       │        │
│  │   └── loki → Loki (logs)                                 │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Semantic Conventions

```
┌──────────────────────────────────────────────────────────────┐
│           SEMANTIC CONVENTIONS OBRIGATÓRIAS                   │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  HTTP Server:                                                 │
│  ├── http.request.method = GET | POST | PUT | DELETE          │
│  ├── url.path = /api/v1/orders                                │
│  ├── http.response.status_code = 200                          │
│  ├── server.address = order-service                           │
│  ├── server.port = 8080                                       │
│  └── http.route = /api/v1/orders/{id}                         │
│                                                               │
│  HTTP Client:                                                 │
│  ├── http.request.method = POST                               │
│  ├── url.full = http://payment-service:8081/api/v1/payments   │
│  └── server.address = payment-service                         │
│                                                               │
│  Database:                                                    │
│  ├── db.system = postgresql                                   │
│  ├── db.namespace = orders                                    │
│  ├── db.operation.name = SELECT                               │
│  └── db.query.text = SELECT * FROM orders WHERE id = $1       │
│                                                               │
│  Messaging (Kafka):                                           │
│  ├── messaging.system = kafka                                 │
│  ├── messaging.operation.type = publish | process             │
│  ├── messaging.destination.name = order-events                │
│  ├── messaging.kafka.consumer.group = notification-group      │
│  └── messaging.message.id = <uuid>                            │
│                                                               │
│  Resource:                                                    │
│  ├── service.name = order-service                             │
│  ├── service.version = 1.2.0                                  │
│  ├── service.namespace = order-platform                       │
│  ├── deployment.environment.name = staging                    │
│  └── telemetry.sdk.language = go | java                       │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Escopo Técnico

### OTel Collector Agent — Configuração Completa

```yaml
# otel-collector/agent-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  # Protege contra OOM
  memory_limiter:
    check_interval: 5s
    limit_mib: 512
    spike_limit_mib: 128

  # Detecta resource attributes automaticamente
  resourcedetection:
    detectors: [env, system, docker]
    timeout: 5s
    override: false

  # Adiciona atributos de ambiente
  attributes/add-env:
    actions:
      - key: deployment.environment.name
        value: "${DEPLOYMENT_ENV}"
        action: upsert

  # Agrupa em batches para eficiência
  batch:
    send_batch_size: 1024
    send_batch_max_size: 2048
    timeout: 5s

exporters:
  otlp/gateway:
    endpoint: otel-gateway:4317
    tls:
      insecure: true
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s

  # Self-monitoring
  debug:
    verbosity: basic

service:
  telemetry:
    metrics:
      address: 0.0.0.0:8888

  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, attributes/add-env, batch]
      exporters: [otlp/gateway]

    metrics:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, attributes/add-env, batch]
      exporters: [otlp/gateway]

    logs:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, attributes/add-env, batch]
      exporters: [otlp/gateway]
```

### OTel Collector Gateway — Tail Sampling + SpanMetrics

```yaml
# otel-collector/gateway-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  memory_limiter:
    check_interval: 5s
    limit_mib: 2048
    spike_limit_mib: 512

  # ── Tail-Based Sampling ──
  # Decide se mantém trace COMPLETO após coletar todos os spans
  tail_sampling:
    decision_wait: 10s                   # aguarda 10s para spans chegarem
    num_traces: 100000                   # traces em memória
    expected_new_traces_per_sec: 1000
    policies:
      # 1. Sempre mantém traces com erro
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]

      # 2. Sempre mantém traces lentos (> 1s)
      - name: latency-policy
        type: latency
        latency:
          threshold_ms: 1000

      # 3. Sempre mantém traces com atributo "important"
      - name: important-policy
        type: string_attribute
        string_attribute:
          key: sampling.priority
          values: [important, debug]

      # 4. Amostra probabilística para o resto (10%)
      - name: probabilistic-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

  # ── Transform: enriquecer dados ──
  transform:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - set(attributes["processed_by"], "gateway")

  batch:
    send_batch_size: 2048
    timeout: 10s

# ── Connector: span → metrics ──
connectors:
  spanmetrics:
    histogram:
      explicit:
        buckets: [10ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s, 10s]
    dimensions:
      - name: http.request.method
      - name: http.response.status_code
      - name: service.name
    exemplars:
      enabled: true

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true

  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
    tls:
      insecure: true

  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, tail_sampling, transform, batch]
      exporters: [otlp/jaeger, spanmetrics]

    # Pipeline de métricas geradas a partir dos spans
    metrics/spanmetrics:
      receivers: [spanmetrics]
      processors: [batch]
      exporters: [prometheusremotewrite]

    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite]

    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [loki]
```

### Stack-Specific Configuration

| Stack | Auto-instrumentação | Configuração |
|-------|-------------------|-------------|
| **Go (Gin)** | `otelgin.Middleware()`, `otelhttp.NewTransport()` | `TracerProvider` programático |
| **Spring Boot** | `opentelemetry-javaagent.jar` (-javaagent) | Env vars: `OTEL_*`, `application.yml` |
| **Quarkus** | `quarkus-opentelemetry` extension (built-in) | `application.properties` |
| **Micronaut** | `micronaut-tracing-opentelemetry-http` (built-in) | `application.yml` |
| **Jakarta EE** | MicroProfile Telemetry 2.0 + CDI interceptors | `microprofile-config.properties` |

---

## Critérios de Aceite

### OTel Collector
- [ ] Agent Collector configurado com receivers (otlp), processors (memory_limiter, resourcedetection, batch), exporters (otlp/gateway)
- [ ] Gateway Collector configurado com tail-based sampling (4 policies)
- [ ] SpanMetrics connector gerando métricas RED a partir de spans
- [ ] Memory limiter protegendo Agent (512MiB) e Gateway (2048MiB)
- [ ] Self-monitoring do Collector acessível em `:8888/metrics`

### Auto-instrumentação
- [ ] Go: `otelgin.Middleware()` gerando spans HTTP server automáticos
- [ ] Go: `otelhttp.NewTransport()` gerando spans HTTP client automáticos
- [ ] Java: `-javaagent:opentelemetry-javaagent.jar` auto-instrumentando HTTP, JDBC, Kafka
- [ ] Spans automáticos seguem semantic conventions (verificar atributos)

### Semantic Conventions
- [ ] HTTP spans com: `http.request.method`, `url.path`, `http.response.status_code`, `http.route`
- [ ] Database spans com: `db.system`, `db.namespace`, `db.operation.name`
- [ ] Messaging spans com: `messaging.system`, `messaging.operation.type`, `messaging.destination.name`
- [ ] Resource attributes: `service.name`, `service.version`, `deployment.environment.name`

### Sampling
- [ ] Head-based: SDK Go configurado com `TraceIDRatioBased(0.5)` em development
- [ ] Tail-based: Gateway mantém 100% de traces com erro
- [ ] Tail-based: Gateway mantém 100% de traces lentos (> 1s)
- [ ] Tail-based: Gateway amostra 10% dos traces normais

### Logs (OTel)
- [ ] Logs enviados via OTel SDK (não mais diretamente para Loki)
- [ ] `trace_id` e `span_id` presentes nos logs estruturados
- [ ] Correlação log ↔ trace funcional no Grafana (clicar no trace, ver logs)

---

## Definição de Pronto (DoD)

- [ ] Todos os critérios de aceite ✅
- [ ] Dois Collector configs (Agent + Gateway) funcionando
- [ ] Tail-based sampling testado: trace com erro aparece, trace normal amostrado em ~10%
- [ ] SpanMetrics gerando métricas visíveis no Prometheus
- [ ] Semantic conventions validados nos spans do Jaeger
- [ ] Documentação: decisão de sampling, pipeline diagram, semantic convention mapping
- [ ] Commit: `feat(level-3): opentelemetry collector pipelines, sampling, semantic conventions`

---

## Checklist

### OTel Collector
- [ ] `otel-collector/agent-config.yaml` — Agent config completo
- [ ] `otel-collector/gateway-config.yaml` — Gateway config com tail sampling + spanmetrics
- [ ] Docker Compose com `otel-agent` e `otel-gateway` como serviços separados
- [ ] Health check dos Collectors: `curl http://localhost:13133/`

### Go (Gin)
- [ ] `internal/platform/otel/provider.go` — TracerProvider, MeterProvider, LoggerProvider
- [ ] `otelgin.Middleware("order-service")` no router
- [ ] `otelhttp.NewTransport(http.DefaultTransport)` para HTTP clients
- [ ] `otelsql.Open()` para database connection
- [ ] Head-based sampler: `TraceIDRatioBased(0.5)` em dev, `AlwaysOn` em prod
- [ ] Custom spans com semantic conventions corretas nos atributos

### Spring Boot
- [ ] `Dockerfile` com `-javaagent:opentelemetry-javaagent.jar`
- [ ] Env vars no `docker-compose.yml`: `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_TRACES_SAMPLER`
- [ ] `application.yml`: custom resource attributes, `service.version`
- [ ] Verificar que JDBC, HTTP client, Kafka são auto-instrumentados

### Quarkus
- [ ] `quarkus-opentelemetry` dependency
- [ ] `quarkus.otel.exporter.otlp.traces.endpoint` configurado
- [ ] `quarkus.otel.resource.attributes` com service metadata

### Micronaut
- [ ] `micronaut-tracing-opentelemetry-http` dependency
- [ ] `otel.traces.exporter` configurado para otlp
- [ ] Resource attributes via `otel.resource.attributes`

### Jakarta EE
- [ ] MicroProfile Telemetry 2.0 dependency
- [ ] `microprofile-config.properties`: `otel.service.name`, `otel.exporter.otlp.endpoint`

---

## Tarefas Sugeridas por Stack

### Go — Provider Setup Completo

```go
// internal/platform/otel/provider.go
package otel

import (
    "context"
    "time"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc"
    "go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploggrpc"
    "go.opentelemetry.io/otel/log/global"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/log"
    "go.opentelemetry.io/otel/sdk/metric"
    "go.opentelemetry.io/otel/sdk/resource"
    "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

type ShutdownFunc func(ctx context.Context) error

func SetupOTelSDK(ctx context.Context, serviceName, serviceVersion, env string) (ShutdownFunc, error) {
    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName(serviceName),
            semconv.ServiceVersion(serviceVersion),
            semconv.DeploymentEnvironmentName(env),
            semconv.ServiceNamespace("order-platform"),
        ),
    )
    if err != nil {
        return nil, err
    }

    // ── Trace Provider ──
    traceExporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    var sampler trace.Sampler
    if env == "production" {
        sampler = trace.AlwaysSample() // tail sampling no Gateway
    } else {
        sampler = trace.TraceIDRatioBased(0.5) // 50% em dev
    }

    tp := trace.NewTracerProvider(
        trace.WithResource(res),
        trace.WithBatcher(traceExporter,
            trace.WithBatchTimeout(5*time.Second),
            trace.WithMaxExportBatchSize(512),
        ),
        trace.WithSampler(sampler),
    )
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    // ── Meter Provider ──
    metricExporter, err := otlpmetricgrpc.New(ctx,
        otlpmetricgrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    mp := metric.NewMeterProvider(
        metric.WithResource(res),
        metric.WithReader(metric.NewPeriodicReader(metricExporter,
            metric.WithInterval(30*time.Second),
        )),
    )
    otel.SetMeterProvider(mp)

    // ── Logger Provider (OTel Logs) ──
    logExporter, err := otlploggrpc.New(ctx,
        otlploggrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    lp := log.NewLoggerProvider(
        log.WithResource(res),
        log.WithProcessor(log.NewBatchProcessor(logExporter)),
    )
    global.SetLoggerProvider(lp)

    // Shutdown function
    shutdown := func(ctx context.Context) error {
        var errs []error
        if err := tp.Shutdown(ctx); err != nil {
            errs = append(errs, err)
        }
        if err := mp.Shutdown(ctx); err != nil {
            errs = append(errs, err)
        }
        if err := lp.Shutdown(ctx); err != nil {
            errs = append(errs, err)
        }
        if len(errs) > 0 {
            return errs[0]
        }
        return nil
    }

    return shutdown, nil
}
```

### Go — Auto-instrumentação (Gin + HTTP Client + SQL)

```go
// cmd/api/main.go
func main() {
    ctx := context.Background()

    shutdown, err := otel.SetupOTelSDK(ctx, "order-service", "1.0.0", os.Getenv("DEPLOY_ENV"))
    if err != nil {
        log.Fatal("failed to setup OTel", zap.Error(err))
    }
    defer shutdown(ctx)

    r := gin.New()

    // Auto-instrumentação HTTP server
    r.Use(otelgin.Middleware("order-service"))

    // HTTP client auto-instrumentado
    httpClient := &http.Client{
        Transport: otelhttp.NewTransport(http.DefaultTransport),
    }

    // Database auto-instrumentada
    db, err := otelsql.Open("pgx", os.Getenv("DATABASE_URL"),
        otelsql.WithAttributes(
            semconv.DBSystemPostgreSQL,
        ),
    )
    if err != nil {
        log.Fatal("failed to open database", zap.Error(err))
    }
    otelsql.RegisterDBStatsMetrics(db)

    // ... registrar rotas
}
```

### Spring Boot — Auto-instrumentação com Java Agent

```dockerfile
# Dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Download OTel Java Agent
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar /app/opentelemetry-javaagent.jar

COPY target/*.jar app.jar

ENTRYPOINT ["java", \
  "-javaagent:/app/opentelemetry-javaagent.jar", \
  "-jar", "app.jar"]
```

```yaml
# docker-compose.yml (excerpt)
services:
  order-service:
    build: ./order-service
    environment:
      OTEL_SERVICE_NAME: order-service
      OTEL_RESOURCE_ATTRIBUTES: "service.version=1.0.0,service.namespace=order-platform,deployment.environment.name=staging"
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-agent:4317
      OTEL_EXPORTER_OTLP_PROTOCOL: grpc
      OTEL_TRACES_SAMPLER: always_on    # tail sampling no Gateway
      OTEL_METRICS_EXPORTER: otlp
      OTEL_LOGS_EXPORTER: otlp
      OTEL_INSTRUMENTATION_COMMON_DB_STATEMENT_SANITIZER_ENABLED: true
```

### Quarkus — Configuração OTel

```properties
# src/main/resources/application.properties
quarkus.otel.enabled=true
quarkus.otel.exporter.otlp.traces.endpoint=http://otel-agent:4317
quarkus.otel.exporter.otlp.metrics.endpoint=http://otel-agent:4317
quarkus.otel.exporter.otlp.logs.endpoint=http://otel-agent:4317

quarkus.otel.resource.attributes=service.version=1.0.0,service.namespace=order-platform,deployment.environment.name=staging

# Logging com correlação de trace
quarkus.log.console.format=%d{yyyy-MM-dd HH:mm:ss} %-5p traceId=%X{traceId} spanId=%X{spanId} [%c{2.}] (%t) %s%e%n
```

### Docker Compose — Agent + Gateway + Stack

```yaml
# docker-compose.yml
services:
  # ── OTel Collector Agent ──
  otel-agent:
    image: otel/opentelemetry-collector-contrib:0.115.0
    command: ["--config=/etc/otelcol/agent-config.yaml"]
    volumes:
      - ./otel-collector/agent-config.yaml:/etc/otelcol/agent-config.yaml
    ports:
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
      - "8888:8888"     # Self-metrics
    environment:
      DEPLOYMENT_ENV: staging
    depends_on:
      - otel-gateway

  # ── OTel Collector Gateway ──
  otel-gateway:
    image: otel/opentelemetry-collector-contrib:0.115.0
    command: ["--config=/etc/otelcol/gateway-config.yaml"]
    volumes:
      - ./otel-collector/gateway-config.yaml:/etc/otelcol/gateway-config.yaml
    ports:
      - "14317:4317"    # OTLP gRPC (from agents)
      - "13133:13133"   # Health check
    depends_on:
      - jaeger
      - prometheus
      - loki

  # ── Jaeger ──
  jaeger:
    image: jaegertracing/jaeger:2.1.0
    ports:
      - "16686:16686"   # UI
      - "4327:4317"     # OTLP gRPC
    environment:
      COLLECTOR_OTLP_ENABLED: true

  # ── Prometheus ──
  prometheus:
    image: prom/prometheus:v2.54.0
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules:/etc/prometheus/rules
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--web.enable-remote-write-receiver'    # para spanmetrics
      - '--enable-feature=exemplar-storage'

  # ── Loki ──
  loki:
    image: grafana/loki:3.0.0
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml

  # ── Grafana ──
  grafana:
    image: grafana/grafana:11.0.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
```

---

## Extensões Opcionais

- [ ] Implementar `transform` processor para sanitizar PII dos spans (remover emails, IPs)
- [ ] Configurar ADOT Collector (AWS Distro) exportando para X-Ray e CloudWatch
- [ ] Implementar `routing` processor para multi-tenant (headers → exporters diferentes)
- [ ] Conectar `spanmetrics` com Grafana dashboard de RED metrics automático
- [ ] Implementar `filter` processor para dropar spans de health check antes de exportar
- [ ] Criar `span events` customizados em Go/Java para capturar business events
- [ ] Implementar Baggage propagation para passar `tenant_id` entre serviços
- [ ] Configurar Exemplars no Prometheus + Grafana (link metric → trace)
