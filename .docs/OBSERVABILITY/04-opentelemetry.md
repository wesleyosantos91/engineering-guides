# Observabilidade Avançada — OpenTelemetry

> **Objetivo deste documento:** Servir como referência completa sobre **OpenTelemetry (OTel)** — a plataforma padrão de observabilidade, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: arquitetura OTel, SDK, Collector, auto-instrumentação, semantic conventions, deployment patterns e integração com backends.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **OTel SDK** | Um SDK para traces + metrics + logs | SDK vendor-specific (Datadog, X-Ray) | Migrar para OTel SDK + exporters |
| **Collector** | Sempre use Collector entre app e backend | App exporta direto para backend | App → Collector → Backend |
| **Auto-instrumentation** | Comece com zero-code, adicione custom | Instrumentar tudo manualmente do zero | Auto-instrument primeiro, custom depois |
| **Semantic Conventions** | Use attribute names padronizados | `myapp.request.url` custom | `url.full`, `http.request.method` (OTel standard) |
| **Batch exporter** | Sempre batch, nunca per-request | Exporter síncrono bloqueando a app | BatchSpanProcessor com queue |
| **Resource attributes** | `service.name` + `service.version` + `deployment.environment` | Spans sem identificação do serviço | Configurar Resource no SDK startup |

---

## Sumário

- [Observabilidade Avançada — OpenTelemetry](#observabilidade-avançada--opentelemetry)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que é OpenTelemetry](#o-que-é-opentelemetry)
  - [Arquitetura OpenTelemetry](#arquitetura-opentelemetry)
  - [OTel SDK — Instrumentação](#otel-sdk--instrumentação)
  - [OTel Collector — O Hub Central](#otel-collector--o-hub-central)
  - [Auto-Instrumentação — Zero Code](#auto-instrumentação--zero-code)
  - [Semantic Conventions](#semantic-conventions)
  - [Context Propagation](#context-propagation)
  - [Sampling no OTel](#sampling-no-otel)
  - [Deployment Patterns](#deployment-patterns)
  - [OTel na AWS — ADOT](#otel-na-aws--adot)
  - [OTel + Backends — Integração](#otel--backends--integração)
  - [Migration Path — Vendor SDK → OTel](#migration-path--vendor-sdk--otel)
  - [Anti-Patterns OTel](#anti-patterns-otel)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que é OpenTelemetry

```
┌─────────────────────────────────────────────────────────────────┐
│                     OPENTELEMETRY                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  O QUE: Framework de observabilidade open-source, vendor-neutral │
│  PROJETO: CNCF (2nd most active after Kubernetes)                │
│  MERGE DE: OpenTracing + OpenCensus (2019)                       │
│  STATUS: Traces (GA), Metrics (GA), Logs (GA)                    │
│                                                                  │
│  O QUE OTEL É:                                                   │
│  ✅ SDK para instrumentação (traces, metrics, logs)              │
│  ✅ APIs padronizadas (language-specific)                        │
│  ✅ Collector para processamento e roteamento                    │
│  ✅ Semantic conventions (nomes padronizados)                    │
│  ✅ Context propagation (W3C TraceContext)                       │
│  ✅ Auto-instrumentação (zero-code)                              │
│  ✅ OTLP protocol (wire format padrão)                           │
│                                                                  │
│  O QUE OTEL NÃO É:                                              │
│  ❌ Backend de armazenamento (não é Prometheus/Jaeger/Tempo)     │
│  ❌ Visualization (não é Grafana)                                │
│  ❌ Alerting engine (não é Alertmanager)                         │
│                                                                  │
│  ANALOGIA:                                                       │
│  OTel = JDBC/ODBC da observabilidade                             │
│  "Um padrão para conectar qualquer app a qualquer backend"       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Linguagens suportadas (SDKs GA)

| Language | Traces | Metrics | Logs | Auto-instrumentation |
|----------|--------|---------|------|---------------------|
| **Java** | GA | GA | GA | ✅ Agent (javaagent) |
| **Python** | GA | GA | GA | ✅ (pip packages) |
| **Go** | GA | GA | GA | ⚠️ (eBPF experimental) |
| **.NET** | GA | GA | GA | ✅ (NuGet packages) |
| **JavaScript/Node** | GA | GA | GA | ✅ (npm packages) |
| **Rust** | GA | Alpha | Alpha | ❌ |
| **C++** | GA | GA | Experimental | ❌ |
| **Ruby** | GA | Experimental | Experimental | ✅ Parcial |

---

## Arquitetura OpenTelemetry

```
┌─────────────────────────────────────────────────────────────────┐
│                  OTEL ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────┐               │
│  │            APPLICATION CODE                    │               │
│  │                                                │               │
│  │  ┌──────────────────────────────────────────┐ │               │
│  │  │         OTel API (interfaces)             │ │               │
│  │  │  Tracer, Meter, Logger                    │ │               │
│  │  │  (dependency leve, sem implementação)      │ │               │
│  │  └──────────────────────────────────────────┘ │               │
│  │                    │                           │               │
│  │  ┌──────────────────────────────────────────┐ │               │
│  │  │         OTel SDK (implementation)         │ │               │
│  │  │                                           │ │               │
│  │  │  TracerProvider → SpanProcessor →         │ │               │
│  │  │                   SpanExporter            │ │               │
│  │  │  MeterProvider  → MetricReader →          │ │               │
│  │  │                   MetricExporter          │ │               │
│  │  │  LoggerProvider → LogProcessor →          │ │               │
│  │  │                   LogExporter             │ │               │
│  │  └──────────────────────────────────────────┘ │               │
│  │                    │                           │               │
│  │  ┌──────────────────────────────────────────┐ │               │
│  │  │     Auto-instrumentation libraries        │ │               │
│  │  │  HTTP client/server, DB drivers,          │ │               │
│  │  │  gRPC, message queues (zero-code)         │ │               │
│  │  └──────────────────────────────────────────┘ │               │
│  └──────────────────────┬───────────────────────┘               │
│                         │ OTLP (gRPC/HTTP)                       │
│                         ▼                                        │
│  ┌──────────────────────────────────────────────┐               │
│  │          OTel COLLECTOR                        │               │
│  │                                                │               │
│  │  Receivers → Processors → Exporters            │               │
│  │                                                │               │
│  │  Receivers:  OTLP, Jaeger, Zipkin, Prometheus  │               │
│  │  Processors: Batch, Filter, Attributes,        │               │
│  │              Sampling, Transform                │               │
│  │  Exporters:  OTLP, Prometheus, Jaeger, X-Ray,  │               │
│  │              Datadog, Loki, CloudWatch          │               │
│  └──────────────────────┬───────────────────────┘               │
│                         │                                        │
│              ┌──────────┼──────────┐                            │
│              ▼          ▼          ▼                            │
│         ┌────────┐ ┌────────┐ ┌────────┐                       │
│         │ Traces │ │Metrics │ │  Logs  │                       │
│         │ Tempo  │ │Promethe│ │  Loki  │                       │
│         │ Jaeger │ │Thanos  │ │  ES    │                       │
│         │ X-Ray  │ │CW      │ │  CW    │                       │
│         └────────┘ └────────┘ └────────┘                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Separação API vs SDK — Por quê?

```
API (interfaces):
  • Dependência leve para libraries/frameworks
  • Não faz nada por default (no-op)
  • Libraries instrumentadas dependem APENAS da API
  
SDK (implementation):
  • Apenas na APPLICATION (entry point)
  • Configura providers, exporters, processors
  • Aplicação "liga" a instrumentação no startup

BENEFÍCIO:
  Uma library (e.g., HTTP client) pode ser instrumentada com OTel API
  sem forçar o usuário da library a usar OTel SDK.
  Se o usuário não configura SDK → instrumentação é no-op (zero overhead).
```

---

## OTel SDK — Instrumentação

### Setup básico (pseudocode)

```python
# Pseudocode — OTel SDK setup (Python)
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

# ─── Step 1: Define Resource (identidade do serviço) ───
resource = Resource.create({
    "service.name": "order-service",
    "service.version": "2.3.1",
    "deployment.environment": "production",
    "service.namespace": "ecommerce",
    "cloud.provider": "aws",
    "cloud.region": "us-east-1"
})

# ─── Step 2: Configure Traces ───
tracer_provider = TracerProvider(resource=resource)

# Exporter: envia spans via OTLP para o Collector
otlp_exporter = OTLPSpanExporter(
    endpoint="http://otel-collector:4317",  # gRPC
    timeout=10  # seconds
)

# Processor: batch spans antes de exportar
span_processor = BatchSpanProcessor(
    otlp_exporter,
    max_queue_size=2048,
    max_export_batch_size=512,
    export_timeout_millis=30000,
    schedule_delay_millis=5000
)

tracer_provider.add_span_processor(span_processor)
trace.set_tracer_provider(tracer_provider)

# ─── Step 3: Configure Metrics ───
meter_provider = MeterProvider(
    resource=resource,
    metric_readers=[
        PeriodicExportingMetricReader(
            OTLPMetricExporter(endpoint="http://otel-collector:4317"),
            export_interval_millis=60000  # export a cada 60s
        )
    ]
)
metrics.set_meter_provider(meter_provider)

# ─── Step 4: Usar no código ───
tracer = trace.get_tracer("order-service", "2.3.1")
meter = metrics.get_meter("order-service", "2.3.1")
```

### Instrumentação manual — Traces

```python
# Pseudocode — Spans customizados
tracer = trace.get_tracer("order-service")

# Span simples
with tracer.start_as_current_span("process_order") as span:
    span.set_attribute("order.id", order_id)
    span.set_attribute("order.total_cents", total_cents)
    span.set_attribute("user.tier", "premium")
    
    # Sub-span (child automático)
    with tracer.start_as_current_span("validate_inventory") as child:
        child.set_attribute("inventory.items_checked", len(items))
        result = check_inventory(items)
        
        if not result.all_available:
            child.add_event("inventory.partial_availability", {
                "unavailable_items": result.unavailable
            })
    
    # Span com error handling
    with tracer.start_as_current_span("charge_payment") as payment_span:
        payment_span.set_attribute("payment.provider", "stripe")
        payment_span.set_attribute("payment.method", "credit_card")
        try:
            charge_result = stripe.charge(total_cents)
            payment_span.set_attribute("payment.charge_id", charge_result.id)
        except TimeoutError as e:
            payment_span.record_exception(e)
            payment_span.set_status(StatusCode.ERROR, "Payment timeout")
            raise
```

### Instrumentação manual — Metrics

```python
# Pseudocode — Custom metrics
meter = metrics.get_meter("order-service")

# Counter: total de orders criadas (monotônico)
order_counter = meter.create_counter(
    name="orders.created.total",
    description="Total number of orders created",
    unit="{order}"
)

# Histogram: distribuição de valores de order
order_value_histogram = meter.create_histogram(
    name="orders.value",
    description="Distribution of order values",
    unit="USD"
)

# UpDownCounter: orders em processamento (sobe/desce)
orders_in_progress = meter.create_up_down_counter(
    name="orders.in_progress",
    description="Number of orders currently being processed",
    unit="{order}"
)

# Observable Gauge: tamanho da fila (callback-based)
def get_queue_size(options):
    size = sqs_client.get_queue_attributes(queue_url)["ApproximateNumberOfMessages"]
    options.observe(size, {"queue.name": "order-events"})

meter.create_observable_gauge(
    name="queue.size",
    callbacks=[get_queue_size],
    description="Current queue depth",
    unit="{message}"
)

# ─── Uso no código ───
def create_order(order):
    orders_in_progress.add(1, {"payment.method": order.payment_method})
    try:
        result = process(order)
        order_counter.add(1, {
            "payment.method": order.payment_method,
            "channel": order.channel,
            "status": "success"
        })
        order_value_histogram.record(order.total_usd, {
            "category": order.category
        })
    except Exception:
        order_counter.add(1, {
            "payment.method": order.payment_method,
            "channel": order.channel,
            "status": "failure"
        })
        raise
    finally:
        orders_in_progress.add(-1, {"payment.method": order.payment_method})
```

---

## OTel Collector — O Hub Central

### Arquitetura do Collector

```
┌─────────────────────────────────────────────────────────────────┐
│                   OTEL COLLECTOR                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────┐    ┌────────────┐    ┌────────────┐              │
│  │ RECEIVERS │ →  │ PROCESSORS │ →  │ EXPORTERS  │              │
│  │           │    │            │    │            │              │
│  │ • otlp    │    │ • batch    │    │ • otlp     │              │
│  │ • jaeger  │    │ • memory   │    │ • prometheus│             │
│  │ • zipkin  │    │   _limiter │    │ • jaeger   │              │
│  │ • prometh │    │ • filter   │    │ • zipkin   │              │
│  │   eus     │    │ • attribut │    │ • awsxray  │              │
│  │ • kafka   │    │   es       │    │ • awscw    │              │
│  │ • filelog │    │ • resource │    │ • datadog  │              │
│  │ • hostm.  │    │ • tail_    │    │ • loki     │              │
│  │ • awsxray │    │   sampling │    │ • kafka    │              │
│  │           │    │ • transform│    │ • file     │              │
│  │           │    │ • spanmetr │    │            │              │
│  │           │    │   ics      │    │            │              │
│  └───────────┘    └────────────┘    └────────────┘              │
│                                                                  │
│  PIPELINES (definidas no config):                                │
│                                                                  │
│  traces:                                                         │
│    receivers: [otlp]                                             │
│    processors: [memory_limiter, batch, tail_sampling]            │
│    exporters: [otlp/tempo, awsxray]                              │
│                                                                  │
│  metrics:                                                        │
│    receivers: [otlp, prometheus]                                 │
│    processors: [memory_limiter, batch]                           │
│    exporters: [prometheusremotewrite, awscloudwatch]             │
│                                                                  │
│  logs:                                                           │
│    receivers: [otlp, filelog]                                    │
│    processors: [memory_limiter, batch, attributes]               │
│    exporters: [loki, awscloudwatchlogs]                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Collector config — Exemplo completo

```yaml
# otel-collector-config.yaml — Exemplo production-ready

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  # Scrape Prometheus metrics de apps que já expõem /metrics
  prometheus:
    config:
      scrape_configs:
        - job_name: 'kubernetes-pods'
          kubernetes_sd_configs:
            - role: pod
          relabel_configs:
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
              action: keep
              regex: true

processors:
  # Prevenir OOM do collector
  memory_limiter:
    check_interval: 1s
    limit_mib: 1500       # 75% do container limit
    spike_limit_mib: 500

  # Agrupar telemetria antes de exportar (reduz overhead de rede)
  batch:
    send_batch_size: 1024
    send_batch_max_size: 2048
    timeout: 5s

  # Adicionar attributes a toda telemetria
  resource:
    attributes:
      - key: cloud.provider
        value: aws
        action: upsert
      - key: cloud.region
        value: us-east-1
        action: upsert

  # Filtrar telemetria indesejada
  filter:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.route"] == "/health"'
        - 'attributes["http.route"] == "/ready"'

  # Gerar métricas RED a partir de spans (spanmetrics)
  spanmetrics:
    metrics_exporter: prometheusremotewrite
    dimensions:
      - name: http.method
      - name: http.status_code
      - name: http.route

  # Redact PII de attributes
  transform:
    trace_statements:
      - context: span
        statements:
          - replace_pattern(attributes["http.url"], "email=[^&]+", "email=REDACTED")
          - delete_key(attributes, "user.email")

exporters:
  # Traces → Grafana Tempo (via OTLP)
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

  # Traces → AWS X-Ray
  awsxray:
    region: us-east-1

  # Metrics → Prometheus Remote Write
  prometheusremotewrite:
    endpoint: "http://prometheus:9090/api/v1/write"

  # Metrics → CloudWatch
  awscloudwatch:
    region: us-east-1
    namespace: "MyApp/Production"

  # Logs → Loki
  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"

extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  zpages:
    endpoint: 0.0.0.0:55679

service:
  extensions: [health_check, zpages]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, filter, resource, batch]
      exporters: [otlp/tempo, awsxray]
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, resource, batch]
      exporters: [prometheusremotewrite, awscloudwatch]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, resource, transform, batch]
      exporters: [loki]
```

### Processors mais importantes

| Processor | Função | Quando usar |
|-----------|--------|-------------|
| **memory_limiter** | Previne OOM do collector | SEMPRE — obrigatório |
| **batch** | Agrupa telemetria antes de exportar | SEMPRE — reduz overhead |
| **filter** | Remove spans/metrics/logs indesejados | Health checks, noise |
| **attributes** | Adiciona/modifica/remove attributes | Enriquecer contexto |
| **resource** | Modifica resource attributes | cloud.region, env |
| **tail_sampling** | Amostragem inteligente (pós-trace) | Gateway collector, high traffic |
| **spanmetrics** | Gera RED metrics de spans | Quando não tem metrics separados |
| **transform** | Transformações complexas (OTTL) | Redact PII, rename, compute |
| **probabilistic_sampler** | Head-based sampling simples | Agent collector |
| **groupbytrace** | Agrupa spans por trace_id | Pre-req para tail_sampling |

---

## Auto-Instrumentação — Zero Code

### Java (javaagent)

```bash
# Download do agent
# curl -L https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar -o opentelemetry-javaagent.jar

# Executar com agent
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=order-service \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
     -Dotel.metrics.exporter=otlp \
     -Dotel.logs.exporter=otlp \
     -jar myapp.jar

# O QUE É INSTRUMENTADO AUTOMATICAMENTE:
# • HTTP server (Spring MVC, JAX-RS, Servlet)
# • HTTP client (HttpClient, OkHttp, RestTemplate, WebClient)
# • Database (JDBC, Hibernate, JPA, R2DBC)
# • gRPC client/server
# • Kafka producer/consumer
# • AWS SDK calls (S3, SQS, DynamoDB, etc.)
# • Redis (Jedis, Lettuce)
# • MongoDB
# • Elasticsearch
```

### Python

```bash
# Install
# pip install opentelemetry-distro opentelemetry-exporter-otlp
# opentelemetry-bootstrap -a install  # instala instrumentações detectadas

# Environment variables
export OTEL_SERVICE_NAME=order-service
export OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
export OTEL_TRACES_EXPORTER=otlp
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp

# Executar com auto-instrumentation
opentelemetry-instrument python myapp.py

# O QUE É INSTRUMENTADO:
# • Flask, Django, FastAPI
# • requests, httpx, aiohttp
# • psycopg2, SQLAlchemy, pymongo
# • boto3 (AWS SDK)
# • Redis, Celery
# • gRPC
```

### Kubernetes — OTel Operator

```yaml
# Pseudocode — OTel Operator auto-instrumentation via annotation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        # O Operator injeta o agent automaticamente!
        instrumentation.opentelemetry.io/inject-java: "true"
        # Ou para Python:
        # instrumentation.opentelemetry.io/inject-python: "true"
    spec:
      containers:
        - name: order-service
          image: order-service:2.3.1
          env:
            - name: OTEL_SERVICE_NAME
              value: "order-service"
            # Collector endpoint configurado pelo Operator
```

---

## Semantic Conventions

### O que são

```
Semantic Conventions = nomes padronizados para attributes

SEM conventions:          COM conventions:
  myapp.http.method         http.request.method
  request_url               url.full
  response_code             http.response.status_code
  db_query                  db.statement
  msg_queue                 messaging.system
  
BENEFÍCIO:
• Dashboards/queries portáveis entre serviços
• Auto-instrumentation usa conventions automaticamente
• Backends otimizam para conventions conhecidas
```

### Conventions mais importantes

#### HTTP

| Attribute | Exemplo | Obrigatório |
|-----------|---------|-------------|
| `http.request.method` | `GET`, `POST` | ✅ |
| `http.response.status_code` | `200`, `500` | ✅ |
| `http.route` | `/api/v1/orders/{id}` | ✅ (se disponível) |
| `url.scheme` | `https` | ✅ |
| `url.full` | `https://api.com/orders?page=1` | ⚠️ (cuidado com PII) |
| `server.address` | `api.example.com` | ✅ |
| `server.port` | `443` | Condicional |

#### Database

| Attribute | Exemplo | Obrigatório |
|-----------|---------|-------------|
| `db.system` | `postgresql`, `dynamodb`, `redis` | ✅ |
| `db.operation.name` | `SELECT`, `GetItem`, `GET` | ✅ |
| `db.collection.name` | `orders`, `users` | ✅ (se aplicável) |
| `db.namespace` | `ecommerce` | Condicional |
| `db.query.text` | `SELECT * FROM orders WHERE id = ?` | ⚠️ (sanitize!) |

#### Messaging

| Attribute | Exemplo | Obrigatório |
|-----------|---------|-------------|
| `messaging.system` | `aws_sqs`, `kafka`, `rabbitmq` | ✅ |
| `messaging.destination.name` | `order-events` | ✅ |
| `messaging.operation` | `publish`, `receive`, `process` | ✅ |
| `messaging.message.id` | `msg-abc123` | Condicional |
| `messaging.batch.message_count` | `10` | Condicional |

#### Cloud / AWS

| Attribute | Exemplo |
|-----------|---------|
| `cloud.provider` | `aws` |
| `cloud.region` | `us-east-1` |
| `cloud.account.id` | `123456789012` |
| `cloud.platform` | `aws_ecs`, `aws_lambda`, `aws_eks` |
| `aws.ecs.task.arn` | `arn:aws:ecs:...` |
| `aws.lambda.invoked_arn` | `arn:aws:lambda:...` |
| `faas.trigger` | `http`, `pubsub`, `datasource` |

---

## Context Propagation

### Propagators no OTel

```
PROPAGATORS = como o contexto viaja entre serviços

┌────────────────────────────────────────────────┐
│  W3C TraceContext (DEFAULT, RECOMENDADO)         │
│  Headers: traceparent, tracestate                │
│  Example: traceparent: 00-traceId-spanId-01      │
├────────────────────────────────────────────────┤
│  B3 (Zipkin legacy)                              │
│  Headers: X-B3-TraceId, X-B3-SpanId, etc.       │
│  ou: b3: {traceId}-{spanId}-{sampled}           │
├────────────────────────────────────────────────┤
│  AWS X-Ray                                       │
│  Header: X-Amzn-Trace-Id                        │
│  Example: Root=1-abc-def;Parent=ghj;Sampled=1   │
├────────────────────────────────────────────────┤
│  Jaeger (legacy)                                 │
│  Header: uber-trace-id                           │
│  Example: traceId:spanId:parentId:flags          │
└────────────────────────────────────────────────┘

CONFIG:
  OTEL_PROPAGATORS=tracecontext,baggage  (default)
  OTEL_PROPAGATORS=tracecontext,baggage,xray  (AWS hybrid)
  OTEL_PROPAGATORS=b3  (Zipkin migration)
```

### Baggage — Propagação de contexto customizado

```python
# Pseudocode — Baggage (context propagation customizado)
from opentelemetry import baggage, context

# Serviço A: define baggage
ctx = baggage.set_baggage("user.tier", "premium")
ctx = baggage.set_baggage("experiment.group", "variant-b", ctx=ctx)

# Baggage viaja automaticamente nos headers HTTP:
# baggage: user.tier=premium,experiment.group=variant-b

# Serviço B: lê baggage
tier = baggage.get_baggage("user.tier")
# tier = "premium"

# CUIDADO:
# ⚠️ Baggage viaja para TODOS os downstream services
# ⚠️ Não coloque PII, credentials, ou dados grandes
# ⚠️ Baggage NÃO é automaticamente adicionado a spans
#    (você precisa copiar manualmente se quiser nos attributes)
```

---

## Sampling no OTel

### Configuração de sampling

```
HEAD-BASED (no SDK):
  OTEL_TRACES_SAMPLER=parentbased_traceidratio
  OTEL_TRACES_SAMPLER_ARG=0.1   # 10%

  Samplers disponíveis:
  • always_on          → 100% (development)
  • always_off         → 0% (desativar tracing)
  • traceidratio       → X% probabilístico
  • parentbased_*      → Respeita decisão do parent

TAIL-BASED (no Collector):
  → Configurar tail_sampling processor no Collector Gateway
  → Ver doc 02-distributed-tracing.md para detalhes

RECOMENDAÇÃO:
  SDK (agent):     parentbased_always_on (sem filtrar)
  Collector agent: memory_limiter + batch (sem sampling)
  Collector GW:    tail_sampling (decisão inteligente)
```

---

## Deployment Patterns

### Pattern 1: DaemonSet + Gateway (Recomendado para Kubernetes)

```
┌─────────────────────────────────────────────────────────────┐
│          DAEMONSET + GATEWAY PATTERN                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Node 1                          Node 2                      │
│  ┌────────────────────┐         ┌────────────────────┐      │
│  │ Pod A → Collector  │         │ Pod C → Collector  │      │
│  │ Pod B → (DaemonSet)│         │ Pod D → (DaemonSet)│      │
│  └────────┬───────────┘         └────────┬───────────┘      │
│           │                              │                   │
│           └──────────┬───────────────────┘                   │
│                      │                                       │
│               ┌──────▼──────┐                                │
│               │  Collector  │  ← Gateway (Deployment)        │
│               │  (Gateway)  │     2+ replicas, HPA           │
│               │             │     Tail sampling               │
│               │  Processors:│     Rate limiting               │
│               │  • tail_samp│     Multi-backend routing       │
│               │  • transform│                                 │
│               └──────┬──────┘                                │
│                      │                                       │
│            ┌─────────┼─────────┐                             │
│            ▼         ▼         ▼                             │
│         Tempo    Prometheus   Loki                           │
│                                                              │
│  DaemonSet: 1 por node, lightweight (batch + memory_limiter)│
│  Gateway: centralizado, heavy (tail_sampling + transform)    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Pattern 2: Sidecar (para isolamento máximo)

```
Cada pod tem seu próprio collector sidecar.

Pod:
┌────────────────────────────────────┐
│  ┌────────────┐  ┌──────────────┐ │
│  │    App      │→│  Collector   │ │
│  │  Container  │  │  (Sidecar)  │ │
│  └────────────┘  └──────┬───────┘ │
└─────────────────────────┼─────────┘
                          │
                     Backend(s)

QUANDO USAR:
• Multi-tenant (isolamento entre services)
• Diferentes configs por serviço
• Compliance requirements
• Service mesh já usa sidecar pattern
```

### Pattern 3: Lambda (AWS serverless)

```
┌─────────────────────────────────────────────────────────────┐
│              LAMBDA OTEL PATTERN                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  OPÇÃO A: ADOT Lambda Layer                                  │
│  Lambda Function                                             │
│  ┌──────────────────────────────────┐                       │
│  │  ┌───────────┐  ┌────────────┐  │                       │
│  │  │ Your Code │  │ ADOT Layer │  │                       │
│  │  │           │→│ (collector  │  │                       │
│  │  │           │  │  embedded) │  │                       │
│  │  └───────────┘  └──────┬─────┘  │                       │
│  └─────────────────────────┼────────┘                       │
│                            │ OTLP                            │
│                            ▼                                 │
│                    Collector Gateway                          │
│                    ou X-Ray / CloudWatch                      │
│                                                              │
│  OPÇÃO B: OTel SDK inline (flush before return)              │
│  • SDK configurado no handler                                │
│  • force_flush() antes de retornar                           │
│  • Menor cold start que Layer                                │
│  • Mais controle, mais código                                │
│                                                              │
│  COLD START IMPACT:                                          │
│  • ADOT Layer Java: +300-500ms                               │
│  • ADOT Layer Python: +100-200ms                             │
│  • ADOT Layer Node: +100-150ms                               │
│  • Mitigação: Provisioned Concurrency                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## OTel na AWS — ADOT

### AWS Distro for OpenTelemetry (ADOT)

```
ADOT = AWS-supported distribution of OTel

INCLUI:
• OTel SDK (Java, Python, Node, .NET, Go)
• OTel Collector (com exporters AWS)
• Lambda Layers
• ECS sidecar container
• EKS add-on

EXPORTERS AWS INCLUÍDOS:
• awsxray         → Traces para X-Ray
• awscloudwatch   → Metrics para CloudWatch
• awscloudwatchlogs → Logs para CloudWatch Logs
• awsemf          → Embedded Metric Format para CW
• awsprometheusremotewrite → Amazon Managed Prometheus

POR QUE ADOT (vs upstream OTel):
• Suportado pela AWS (patches de segurança)
• Testado com serviços AWS
• Pré-configurado com exporters AWS
• Lambda Layers prontas
• EKS add-on managed
```

### ADOT Collector na AWS — Opções de deploy

| Ambiente | Método | Como |
|----------|--------|------|
| **EKS** | EKS Add-on | `aws eks create-addon --addon-name adot` |
| **EKS** | Helm chart | `helm install adot-collector ...` |
| **ECS** | Sidecar | Container ADOT no Task Definition |
| **EC2** | Systemd service | Install ADOT collector como daemon |
| **Lambda** | Layer | ARN da ADOT Lambda Layer |

---

## OTel + Backends — Integração

### Grafana Stack (LGTM)

```
┌─────────────────────────────────────────────────────────────┐
│              GRAFANA LGTM STACK                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  OTel Collector                                              │
│       │                                                      │
│       ├──▶ Loki      (Logs)     ── Grafana Explore          │
│       ├──▶ Grafana   (Dashboards)                            │
│       ├──▶ Tempo     (Traces)   ── Grafana Explore          │
│       └──▶ Mimir     (Metrics)  ── Grafana Dashboards       │
│            (ou Prometheus)                                    │
│                                                              │
│  CORRELATION no Grafana:                                     │
│  • Log line → click trace_id → abre Trace no Tempo          │
│  • Metric exemplar → click → abre Trace no Tempo            │
│  • Trace span → click → mostra Logs do período              │
│                                                              │
│  AWS Managed:                                                │
│  • Amazon Managed Grafana                                    │
│  • Amazon Managed Prometheus (Mimir-like)                    │
│  • Logs: CloudWatch Logs (ou Loki self-hosted)               │
│  • Traces: X-Ray ou Tempo                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Decisão: AWS-native vs OSS Stack

| Critério | AWS Native | Grafana/OSS Stack |
|----------|-----------|-------------------|
| **Traces** | X-Ray | Tempo / Jaeger |
| **Metrics** | CloudWatch / AMP | Prometheus / Mimir |
| **Logs** | CloudWatch Logs | Loki / Elasticsearch |
| **Dashboards** | CloudWatch / AMG | Grafana |
| **Setup** | Simples (managed) | Mais trabalho (mas mais flexível) |
| **Custo (alto vol.)** | $$$$$ (CW é caro em high vol.) | $$ (self-hosted) ou $$$ (managed) |
| **Lock-in** | Alto | Baixo |
| **Correlation** | Parcial (X-Ray ↔ CW) | Excelente (Grafana explore) |
| **Community** | AWS docs | Enorme (CNCF + Grafana Labs) |

---

## Migration Path — Vendor SDK → OTel

### Estratégia de migração

```
FASE 1: Collector como proxy (Semana 1-2)
═════════════════════════════════════════
  App (vendor SDK) → Vendor agent → OTel Collector → Vendor backend
  
  Mudança: Inserir OTel Collector entre app/agent e backend
  Benefício: Visibilidade do fluxo, capacidade de routing
  Risco: Zero (apenas proxy)

FASE 2: Dual-write (Semana 3-4)
════════════════════════════════
  App (vendor SDK) → OTel Collector → Vendor backend
                                    → New backend (Tempo/Prometheus)
  
  Mudança: Collector exporta para dois backends
  Benefício: Validar que dados chegam corretos no novo backend
  Risco: Baixo (dados duplicados temporariamente)

FASE 3: SDK migration (Mês 2-3)
════════════════════════════════
  App (OTel SDK) → OTel Collector → Old vendor backend (mantido)
                                  → New backend (primary)
  
  Mudança: Substituir vendor SDK por OTel SDK (serviço por serviço)
  Benefício: Portabilidade completa
  Risco: Médio (testar instrumentação)

FASE 4: Cutover (Mês 4)
═══════════════════════
  App (OTel SDK) → OTel Collector → New backend (primary)
  
  Mudança: Remover export para vendor antigo
  Benefício: Custo reduzido, stack simplificado
```

---

## Anti-Patterns OTel

| Anti-pattern | Problema | Solução |
|-------------|---------|---------|
| **App → Backend direto** | Sem buffering, sem processing, app coupled ao backend | Sempre use Collector entre app e backend |
| **Sem memory_limiter** | Collector OOM em pico de tráfego | SEMPRE configurar `memory_limiter` processor |
| **Sem batch processor** | Uma request de export por span | SEMPRE configurar `batch` processor |
| **Sync exporter** | Export bloqueia thread da app | Use `BatchSpanProcessor` (async) |
| **Agent faz tail sampling** | Decisão incompleta (não vê trace inteiro) | Tail sampling só no Gateway collector |
| **SDK em library** | Library força OTel SDK como dependency | Library deve depender apenas da API (não SDK) |
| **Ignore Resource** | Spans sem `service.name` → impossível identificar | Configurar Resource com service.name, version, env |
| **Custom attribute names** | `myapp.url` em vez de `url.full` | Usar Semantic Conventions |
| **Baggage com PII** | PII propagado para todos os downstream services | Nunca colocar dados sensíveis em Baggage |
| **Flush on every request** | `force_flush()` a cada request → overhead | Apenas flush no shutdown (ou Lambda return) |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código relacionado a OpenTelemetry, verifique:

1. **App exportando direto para backend** — Exija OTel Collector entre app e backend
2. **Sem Resource configuration** — `service.name`, `service.version`, `deployment.environment` obrigatórios
3. **Sync exporter** — Use `BatchSpanProcessor`, nunca `SimpleSpanProcessor` em produção
4. **Sem memory_limiter no Collector** — Obrigatório para prevenir OOM
5. **Sem batch processor no Collector** — Obrigatório para reduzir overhead de rede
6. **Custom attribute names** — Use Semantic Conventions: `http.request.method`, não `my_method`
7. **PII em attributes/baggage** — Nunca email, CPF, password; use `transform` processor para redact
8. **Tail sampling no agent** — Tail sampling só funciona corretamente no Gateway collector
9. **SDK como dependency de library** — Libraries devem depender da API, não do SDK
10. **force_flush() em hot path** — Apenas no shutdown ou Lambda pre-return, nunca por request
11. **Sem propagation configurada** — Verificar que `OTEL_PROPAGATORS` inclui `tracecontext,baggage`
12. **Health checks em SLI** — Filtrar `/health`, `/ready` no Collector (`filter` processor)
13. **Collector sem health_check extension** — Obrigatório para K8s liveness/readiness probes
14. **High cardinality em span names** — Span name com IDs → use route template

---

## Referências

- **OpenTelemetry Documentation** — https://opentelemetry.io/docs/
- **OpenTelemetry Specification** — https://opentelemetry.io/docs/specs/otel/
- **OpenTelemetry Semantic Conventions** — https://opentelemetry.io/docs/specs/semconv/
- **OTel Collector Contrib** — https://github.com/open-telemetry/opentelemetry-collector-contrib
- **AWS Distro for OpenTelemetry** — https://aws-otel.github.io/
- **ADOT Lambda Layer** — https://aws-otel.github.io/docs/getting-started/lambda
- **Grafana Labs — OTel Integration** — https://grafana.com/docs/opentelemetry/
- **Observability Engineering** — Charity Majors, Liz Fong-Jones, George Miranda (O'Reilly)
