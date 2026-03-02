# Level 1 — Distributed Tracing

> **Objetivo:** Implementar tracing distribuído com OpenTelemetry entre múltiplos serviços, com context propagation (W3C TraceContext), spans customizados, tracing assíncrono e visualização em Jaeger/Tempo.

---

## Objetivo de Aprendizado

- Entender anatomia de um trace (trace_id, span_id, parent_span_id)
- Configurar OpenTelemetry SDK para traces em Go e Java
- Implementar context propagation via W3C TraceContext (HTTP headers)
- Criar spans customizados com attributes de negócio
- Aplicar span naming conventions (`<verb> <noun>`)
- Implementar tracing assíncrono com SpanLinks (producer → consumer)
- Correlacionar logs com trace_id e span_id
- Visualizar traces em Jaeger ou Grafana Tempo
- Analisar traces para identificar bottlenecks (waterfall view)

---

## Escopo Funcional

### Serviços envolvidos

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    Order     │ ──▶ │   Payment    │     │ Notification │
│   Service    │ ──▶ │   Service    │     │   Service    │
│   (HTTP)     │     │   (HTTP)     │     │   (Async)    │
└──────┬───────┘     └──────────────┘     └──────────────┘
       │                                         ▲
       │              ┌──────────────┐           │
       └────────────▶ │  Inventory   │     Kafka/SQS
                      │   Service    │ ─────────┘
                      │   (HTTP)     │
                      └──────────────┘
```

### Fluxo de Criação de Pedido (traceável end-to-end)

```
POST /api/v1/orders
  │
  ├── [SPAN] order.create (Order Service)
  │   ├── [SPAN] order.validate_items
  │   │
  │   ├── [SPAN] HTTP CLIENT → Inventory Service
  │   │   └── [SPAN] inventory.reserve_items (Inventory Service)
  │   │       └── [SPAN] db.query SELECT + UPDATE
  │   │
  │   ├── [SPAN] HTTP CLIENT → Payment Service
  │   │   └── [SPAN] payment.process (Payment Service)
  │   │       ├── [SPAN] payment.validate_method
  │   │       └── [SPAN] db.query INSERT payment
  │   │
  │   ├── [SPAN] db.query INSERT order
  │   │
  │   └── [SPAN] PRODUCER → order.events topic
  │       └── [LINK] Notification Service consume
  │           └── [SPAN] notification.send_confirmation
  │               └── [SPAN] email.send
```

### Novos Endpoints

```
── Order Service (porta 8080) ──
POST /api/v1/orders                    → Criar pedido (orquestra chamadas)
GET  /api/v1/orders/{id}               → Buscar pedido

── Payment Service (porta 8081) ──
POST /api/v1/payments                  → Processar pagamento
GET  /api/v1/payments/{id}             → Buscar pagamento

── Inventory Service (porta 8082) ──
POST /api/v1/inventory/reserve         → Reservar itens
GET  /api/v1/inventory/{sku}           → Consultar estoque

── Notification Service (consumer assíncrono) ──
Consome do tópico "order.events"       → Envia email de confirmação
```

---

## Escopo Técnico

### OpenTelemetry SDK Setup

| Componente | Configuração |
|---|---|
| **TracerProvider** | Resource: `service.name`, `service.version`, `deployment.environment` |
| **SpanProcessor** | `BatchSpanProcessor` (nunca `SimpleSpanProcessor` em prod) |
| **SpanExporter** | OTLP gRPC → OTel Collector (ou direto para Jaeger) |
| **Propagator** | W3C TraceContext (`traceparent` + `tracestate`) |
| **Sampler** | `AlwaysOn` para development, `TraceIDRatioBased(0.1)` para prod |

### Span Design — Regras

| Regra | Bom ✅ | Ruim ❌ |
|-------|--------|---------|
| **Naming** | `GET /api/v1/orders/{id}` | `/api/v1/orders/550e8400` |
| **Naming** | `process_payment` | `span-1`, `my-span` |
| **Attributes** | `order.id=ord-999`, `order.total_cents=5000` | Span sem attributes |
| **Status** | `StatusCode.ERROR` + description | Exceção engolida sem marcar span |
| **Events** | `span.add_event("inventory.partial")` | Log fora do span sem correlação |
| **Kind** | `CLIENT` para outgoing, `SERVER` para incoming | Todos como `INTERNAL` |

### Span Attributes Obrigatórios

| Span | Attributes | Kind |
|------|-----------|------|
| `order.create` | `order.id`, `order.total_cents`, `order.items_count` | `SERVER` |
| `inventory.reserve_items` | `inventory.sku`, `inventory.quantity`, `inventory.available` | `SERVER` |
| `payment.process` | `payment.method`, `payment.amount_cents`, `payment.currency` | `SERVER` |
| `notification.send` | `notification.type` (email/push), `notification.recipient` | `CONSUMER` |
| HTTP client calls | `http.request.method`, `http.response.status_code`, `server.address` | `CLIENT` |
| DB queries | `db.system`, `db.operation.name`, `db.collection.name` | `CLIENT` |

### Context Propagation

```
HTTP (síncrono):
  Order Service ──traceparent──▶ Payment Service
  Order Service ──traceparent──▶ Inventory Service

  Header: traceparent: 00-{traceId}-{spanId}-01
  Hierarchy: Order (parent) → Payment (child)

Async (Kafka/SQS):
  Order Service ──▶ Kafka topic ──▶ Notification Service

  Producer: inject traceparent nos Kafka headers / message attributes
  Consumer: cria NOVO trace com SpanLink → producer span
  
  Trace 1: Order → publish (Producer)
  Trace 2: Notification consume (SpanLink → Trace 1)
```

### Ferramentas por Stack

| Aspecto | Go (Gin) | Spring Boot | Quarkus | Micronaut | Jakarta EE |
|---|---|---|---|---|---|
| **OTel SDK** | `go.opentelemetry.io/otel` | `io.opentelemetry:opentelemetry-sdk` | SmallRye OpenTelemetry | `io.opentelemetry:opentelemetry-sdk` | `io.opentelemetry:opentelemetry-sdk` |
| **HTTP instr.** | `otelgin` middleware | Micrometer Tracing bridge | SmallRye auto | Micronaut Tracing | javaagent |
| **HTTP client instr.** | `otelhttp` transport | RestClient + OTel | REST Client auto-traced | `@Client` auto-traced | JAX-RS Client + interceptor |
| **DB instr.** | `otelsql` / `otelgorm` | Hibernate auto-traced (agent) | Hibernate auto-traced | JPA auto-traced | JPA + agent |
| **Kafka instr.** | `otelkafka` wrapper | Spring Kafka + OTel bridge | SmallRye Reactive Messaging | Micronaut Kafka | Kafka client + agent |
| **Exporter** | `otlptracegrpc` | `opentelemetry-exporter-otlp` | OTLP (SmallRye config) | `opentelemetry-exporter-otlp` | `opentelemetry-exporter-otlp` |

---

## Critérios de Aceite

### Tracing
- [ ] Cada request HTTP gera um trace com `trace_id` único
- [ ] Trace end-to-end visível: Order → Inventory → Payment (waterfall)
- [ ] Context propagation via `traceparent` header entre serviços
- [ ] Spans para: HTTP handler (SERVER), HTTP client call (CLIENT), DB query (CLIENT)
- [ ] Span names seguem convenção `<verb> <noun>` ou route template
- [ ] Span attributes de negócio presentes (order.id, payment.method, etc.)
- [ ] Erros marcados com `StatusCode.ERROR` e exception gravada via `record_exception`
- [ ] Tracing async: Notification Service com SpanLink para o producer span

### Correlation
- [ ] `trace_id` e `span_id` presentes em todos os logs (structured JSON)
- [ ] No Grafana/Jaeger: clicar no trace → ver logs correlacionados
- [ ] Header `X-Trace-Id` retornado no response HTTP

### Infra
- [ ] `docker-compose.yml` inclui: 3+ serviços, PostgreSQL, Kafka, OTel Collector, Jaeger/Tempo, Grafana
- [ ] OTel Collector configurado com receiver OTLP e exporter para Jaeger/Tempo
- [ ] Traces visíveis no Jaeger/Tempo via Grafana Explore

---

## Definição de Pronto (DoD)

- [ ] Todos os critérios de aceite ✅
- [ ] Pelo menos 3 serviços comunicando com traces end-to-end
- [ ] Producer/Consumer async com SpanLink funcionando
- [ ] Screenshot/recording do waterfall view mostrando trace completo
- [ ] Testes de integração validando propagation (trace_id consistente entre serviços)
- [ ] Documentação do span design (quais spans, attributes, kinds)
- [ ] Commit: `feat(level-1): implement distributed tracing with OpenTelemetry`

---

## Checklist

### Go (Gin)

- [ ] `internal/platform/tracing/provider.go` — TracerProvider + BatchSpanProcessor + OTLP exporter
- [ ] `internal/middleware/tracing.go` — otelgin middleware
- [ ] `internal/platform/tracing/propagation.go` — W3C TraceContext propagator
- [ ] `internal/platform/httpclient/traced.go` — HTTP client com otelhttp transport
- [ ] `internal/middleware/logging.go` — (atualizar) injetar trace_id e span_id nos logs Zap
- [ ] `internal/service/order.go` — Spans customizados: `order.create`, `order.validate_items`
- [ ] `internal/platform/kafka/producer.go` — Inject traceparent nos Kafka headers
- [ ] `internal/platform/kafka/consumer.go` — Extract traceparent + criar SpanLink
- [ ] `cmd/payment/main.go` — Payment Service com OTel setup
- [ ] `cmd/inventory/main.go` — Inventory Service com OTel setup
- [ ] `cmd/notification/main.go` — Notification Service (consumer) com SpanLink

### Spring Boot

- [ ] `pom.xml` — deps: `micrometer-tracing-bridge-otel`, `opentelemetry-exporter-otlp`
- [ ] `application.yml` — `management.tracing.sampling.probability=1.0` + OTLP endpoint
- [ ] `infrastructure/config/TracingConfig.java` — Propagator W3C, Resource attributes
- [ ] `infrastructure/client/InventoryClient.java` — RestClient com tracing auto-propagated
- [ ] `infrastructure/client/PaymentClient.java` — RestClient com tracing auto-propagated
- [ ] `domain/service/OrderService.java` — `@Observed` ou custom spans com Tracer
- [ ] `infrastructure/kafka/OrderEventProducer.java` — Produce com trace context
- [ ] `infrastructure/kafka/OrderEventConsumer.java` — Consume com SpanLink
- [ ] `logback-spring.xml` — MDC com `trace_id`, `span_id`

### Quarkus

- [ ] `pom.xml` — deps: `quarkus-opentelemetry`
- [ ] `application.properties` — `quarkus.otel.exporter.otlp.traces.endpoint`, `quarkus.otel.service.name`
- [ ] `@WithSpan` annotations para custom spans
- [ ] SmallRye Reactive Messaging com tracing auto-propagated

### Micronaut

- [ ] `pom.xml` — deps: `micronaut-tracing-opentelemetry-http`
- [ ] `application.yml` — `tracing.opentelemetry.*` config
- [ ] `@NewSpan`, `@ContinueSpan` annotations
- [ ] `@KafkaClient` / `@KafkaListener` com tracing headers

### Jakarta EE

- [ ] MicroProfile Telemetry 2.0 — `@WithSpan`, `@SpanAttribute`
- [ ] javaagent para auto-instrumentação
- [ ] Kafka client com interceptor manual para propagation

---

## Tarefas Sugeridas por Stack

### Go — OTel SDK Setup

```go
// internal/platform/tracing/provider.go
package tracing

import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

func InitProvider(ctx context.Context, serviceName, serviceVersion string) (func(), error) {
    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName(serviceName),
            semconv.ServiceVersion(serviceVersion),
            semconv.DeploymentEnvironment("development"),
        ),
    )
    if err != nil {
        return nil, err
    }

    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("otel-collector:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.AlwaysSample()),
    )

    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    shutdown := func() {
        _ = tp.Shutdown(ctx)
    }
    return shutdown, nil
}
```

### Go — Custom Spans

```go
// internal/service/order.go
package service

import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("order-service")

func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    ctx, span := tracer.Start(ctx, "order.create", trace.WithAttributes(
        attribute.Int("order.items_count", len(req.Items)),
        attribute.Int64("order.total_cents", req.TotalCents),
    ))
    defer span.End()

    // Validar items
    ctx, validateSpan := tracer.Start(ctx, "order.validate_items")
    if err := s.validateItems(ctx, req.Items); err != nil {
        validateSpan.RecordError(err)
        validateSpan.SetStatus(codes.Error, "validation failed")
        validateSpan.End()
        return nil, err
    }
    validateSpan.End()

    // Reservar inventário (HTTP call → Inventory Service)
    if err := s.inventoryClient.Reserve(ctx, req.Items); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "inventory reservation failed")
        return nil, err
    }

    // Processar pagamento (HTTP call → Payment Service)
    payment, err := s.paymentClient.Process(ctx, req.PaymentMethod, req.TotalCents)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "payment processing failed")
        return nil, err
    }

    // Salvar order no banco
    order, err := s.store.Create(ctx, req, payment.ID)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "order persistence failed")
        return nil, err
    }

    span.SetAttributes(attribute.String("order.id", order.ID))

    // Publicar evento async (com trace context para SpanLink)
    s.eventProducer.Publish(ctx, OrderCreatedEvent{OrderID: order.ID})

    return order, nil
}
```

### Go — Async Tracing com SpanLink

```go
// internal/platform/kafka/consumer.go
package kafka

import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("notification-service")

func (c *Consumer) HandleMessage(msg *kafka.Message) {
    // Extrair context do producer
    carrier := NewKafkaHeaderCarrier(msg.Headers)
    producerCtx := otel.GetTextMapPropagator().Extract(context.Background(), carrier)
    producerSpanCtx := trace.SpanContextFromContext(producerCtx)

    // Criar novo span com LINK para o producer (não child)
    ctx, span := tracer.Start(context.Background(), "notification.process_order_event",
        trace.WithLinks(trace.Link{
            SpanContext: producerSpanCtx,
            Attributes: []attribute.KeyValue{
                attribute.String("messaging.system", "kafka"),
                attribute.String("messaging.operation", "process"),
            },
        }),
        trace.WithSpanKind(trace.SpanKindConsumer),
    )
    defer span.End()

    span.SetAttributes(
        attribute.String("messaging.destination.name", msg.Topic),
        attribute.String("messaging.message.id", string(msg.Key)),
    )

    // Processar notificação
    c.notificationService.SendConfirmation(ctx, msg.Value)
}
```

### Spring Boot — Tracing com Micrometer Bridge

```java
// domain/service/OrderService.java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final InventoryClient inventoryClient;
    private final PaymentClient paymentClient;
    private final OrderRepository orderRepository;
    private final OrderEventProducer eventProducer;
    private final Tracer tracer;

    public Order createOrder(CreateOrderRequest request) {
        Span span = tracer.nextSpan().name("order.create")
            .tag("order.items_count", String.valueOf(request.items().size()))
            .tag("order.total_cents", String.valueOf(request.totalCents()))
            .start();

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            // Reservar inventário
            inventoryClient.reserve(request.items());

            // Processar pagamento
            var payment = paymentClient.process(
                request.paymentMethod(), request.totalCents()
            );

            // Salvar order
            var order = orderRepository.save(Order.from(request, payment.id()));
            span.tag("order.id", order.getId().toString());

            // Publicar evento
            eventProducer.publish(new OrderCreatedEvent(order.getId()));

            return order;
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

```yaml
# application.yml — Tracing config
management:
  tracing:
    sampling:
      probability: 1.0
    propagation:
      type: w3c
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces

logging:
  pattern:
    level: "%5p [${spring.application.name},%X{traceId},%X{spanId}]"
```

### Docker Compose — Multi-service + Tracing Stack

```yaml
# docker-compose.yml
services:
  order-service:
    build: ./order-service
    ports: ["8080:8080"]
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      - OTEL_SERVICE_NAME=order-service
    depends_on: [postgres, kafka, otel-collector]

  payment-service:
    build: ./payment-service
    ports: ["8081:8080"]
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      - OTEL_SERVICE_NAME=payment-service
    depends_on: [postgres, otel-collector]

  inventory-service:
    build: ./inventory-service
    ports: ["8082:8080"]
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      - OTEL_SERVICE_NAME=inventory-service
    depends_on: [postgres, otel-collector]

  notification-service:
    build: ./notification-service
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      - OTEL_SERVICE_NAME=notification-service
    depends_on: [kafka, otel-collector]

  postgres:
    image: postgres:17
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass

  kafka:
    image: confluentinc/cp-kafka:7.7.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'

  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.115.0
    volumes:
      - ./infra/otel/otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP

  jaeger:
    image: jaegertracing/all-in-one:1.63
    ports:
      - "16686:16686" # Jaeger UI
      - "14268:14268" # Jaeger collector HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true

  prometheus:
    image: prom/prometheus:v3.2.0
    volumes:
      - ./infra/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports: ["9090:9090"]

  grafana:
    image: grafana/grafana:11.5.0
    ports: ["3000:3000"]
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - ./infra/grafana/provisioning:/etc/grafana/provisioning
```

```yaml
# infra/otel/otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    send_batch_size: 1024
    timeout: 5s
  memory_limiter:
    check_interval: 1s
    limit_mib: 512

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/jaeger]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

---

## Extensões Opcionais

- [ ] Implementar `baggage` para propagar `user.tier` entre serviços
- [ ] Adicionar span events para logging dentro de spans (em vez de logs separados)
- [ ] Implementar retry com exponential backoff no HTTP client (spans de retry visíveis)
- [ ] Configurar Grafana Tempo como backend (em vez de Jaeger) com correlação Logs↔Traces
- [ ] Adicionar `exemplars` nas métricas Prometheus (link para trace_id específico)
- [ ] Implementar tracing para database queries (otelsql / Hibernate auto-traced)
- [ ] Criar dashboard Grafana com "Service Graph" (mapa de dependências gerado de traces)
