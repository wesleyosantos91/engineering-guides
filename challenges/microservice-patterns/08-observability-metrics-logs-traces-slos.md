# Level 8 — Observability: Metrics, Logs, Traces & SLOs

> **Objetivo:** Implementar os 3 pilares da observabilidade — Métricas (RED/USE), Logs Estruturados e
> Traces Distribuídos — conectados por Correlation ID. Definir SLOs/SLIs para garantir qualidade de serviço.

**Pré-requisito:** [Level 7 — Deployment](07-deployment-strangler-bluegreen-canary-flags-shadow.md)

**Referências:**
- [27-observabilidade.md](../../.docs/microservice-patterns/27-observabilidade.md)

---

## Contexto do Domínio

O RideFlow tem múltiplos serviços distribuídos. Quando a latência de matching sobe de 2s para 15s em horário de pico:
**Qual serviço falhou? Onde está o gargalo? É transiente ou persistente? Estamos dentro dos SLOs?**

Observabilidade responde essas perguntas correlacionando métricas, logs e traces com o mesmo traceId.

---

## Desafios

### Desafio 8.1 — Métricas: RED Method & USE Method

Instrumente todos os serviços com métricas seguindo os métodos RED (serviços) e USE (infraestrutura).

**Cenário:** O SRE precisa de dashboards para responder: "Qual é o throughput? Qual é o error rate? Qual é a latência P99? O banco está saturado?"

**Requisitos:**
- **RED Method** (para cada endpoint de cada serviço):
  - **Rate:** requests/segundo (`http_requests_total` counter)
  - **Errors:** taxa de erro (`http_errors_total` counter, segmentado por status code)
  - **Duration:** latência (`http_request_duration_seconds` histogram, buckets: 10ms, 50ms, 100ms, 250ms, 500ms, 1s, 5s)
- **USE Method** (para infraestrutura):
  - **Utilization:** `db_connections_active / db_connections_max` (gauge)
  - **Saturation:** `kafka_consumer_lag` (gauge)
  - **Errors:** `db_query_errors_total` (counter)
- **Labels/tags:** `service`, `method`, `endpoint`, `status_code`, `city_code`
- **Prometheus endpoint:** `GET /metrics` expondo métricas no formato Prometheus
- **Custom business metrics:**
  - `ride_matching_duration_seconds` — tempo para encontrar motorista
  - `rides_created_total` — corridas criadas (por cidade, status)
  - `active_rides_gauge` — corridas ativas (gauge)
  - `surge_pricing_multiplier` — multiplicador de surge (por cidade)

**Java 25 (Micrometer + Prometheus):**
```java
@Component
public class RideMetrics {
    private final Counter ridesCreated;
    private final Timer matchingDuration;
    private final Gauge activeRides;
    private final DistributionSummary surgeMultiplier;
    
    public RideMetrics(MeterRegistry registry) {
        this.ridesCreated = Counter.builder("rides_created_total")
            .description("Total rides created")
            .tag("city", "unknown") // será override por city
            .register(registry);
        
        this.matchingDuration = Timer.builder("ride_matching_duration_seconds")
            .description("Time to find a driver")
            .publishPercentiles(0.5, 0.95, 0.99)
            .sla(Duration.ofSeconds(2), Duration.ofSeconds(5), Duration.ofSeconds(10))
            .register(registry);
        
        this.activeRides = Gauge.builder("active_rides_gauge",
                rideRepository, repo -> repo.countByStatusIn(
                    List.of(RideStatus.MATCHED, RideStatus.IN_PROGRESS)))
            .description("Currently active rides")
            .register(registry);
    }
    
    public void recordRideCreated(String city) {
        Counter.builder("rides_created_total")
            .tag("city", city)
            .register(meterRegistry)
            .increment();
    }
    
    public Timer.Sample startMatchingTimer() {
        return Timer.start(meterRegistry);
    }
    
    public void stopMatchingTimer(Timer.Sample sample, String city) {
        sample.stop(Timer.builder("ride_matching_duration_seconds")
            .tag("city", city)
            .register(meterRegistry));
    }
}

// HTTP interceptor para RED method automático
@Component
public class HttpMetricsFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) throws ServletException, IOException {
        var sample = Timer.start(meterRegistry);
        
        try {
            chain.doFilter(request, response);
        } finally {
            sample.stop(Timer.builder("http_request_duration_seconds")
                .tag("method", request.getMethod())
                .tag("endpoint", normalizedPath(request))
                .tag("status", String.valueOf(response.getStatus()))
                .register(meterRegistry));
            
            Counter.builder("http_requests_total")
                .tag("method", request.getMethod())
                .tag("endpoint", normalizedPath(request))
                .tag("status", String.valueOf(response.getStatus()))
                .register(meterRegistry)
                .increment();
            
            if (response.getStatus() >= 500) {
                Counter.builder("http_errors_total")
                    .tag("method", request.getMethod())
                    .tag("endpoint", normalizedPath(request))
                    .tag("status", String.valueOf(response.getStatus()))
                    .register(meterRegistry)
                    .increment();
            }
        }
    }
}
```

**Go 1.26 (Prometheus client):**
```go
type Metrics struct {
    HTTPRequestsTotal   *prometheus.CounterVec
    HTTPRequestDuration *prometheus.HistogramVec
    HTTPErrorsTotal     *prometheus.CounterVec
    RidesCreatedTotal   *prometheus.CounterVec
    MatchingDuration    *prometheus.HistogramVec
    ActiveRidesGauge    prometheus.Gauge
    DBConnectionsActive prometheus.Gauge
    KafkaConsumerLag    *prometheus.GaugeVec
}

func NewMetrics(reg prometheus.Registerer) *Metrics {
    m := &Metrics{
        HTTPRequestsTotal: prometheus.NewCounterVec(
            prometheus.CounterOpts{
                Name: "http_requests_total",
                Help: "Total HTTP requests",
            }, []string{"method", "endpoint", "status"}),
        
        HTTPRequestDuration: prometheus.NewHistogramVec(
            prometheus.HistogramOpts{
                Name:    "http_request_duration_seconds",
                Help:    "HTTP request latency",
                Buckets: []float64{0.01, 0.05, 0.1, 0.25, 0.5, 1, 5},
            }, []string{"method", "endpoint"}),
        
        RidesCreatedTotal: prometheus.NewCounterVec(
            prometheus.CounterOpts{
                Name: "rides_created_total",
                Help: "Total rides created",
            }, []string{"city"}),
        
        MatchingDuration: prometheus.NewHistogramVec(
            prometheus.HistogramOpts{
                Name:    "ride_matching_duration_seconds",
                Help:    "Time to find a driver",
                Buckets: []float64{1, 2, 5, 10, 30},
            }, []string{"city"}),
        
        ActiveRidesGauge: prometheus.NewGauge(
            prometheus.GaugeOpts{
                Name: "active_rides_gauge",
                Help: "Currently active rides",
            }),
    }
    
    reg.MustRegister(m.HTTPRequestsTotal, m.HTTPRequestDuration,
        m.RidesCreatedTotal, m.MatchingDuration, m.ActiveRidesGauge)
    
    return m
}

// Middleware para RED automático
func MetricsMiddleware(metrics *Metrics) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            ww := middleware.NewWrapResponseWriter(w, r.ProtoMajor)
            
            next.ServeHTTP(ww, r)
            
            duration := time.Since(start).Seconds()
            status := strconv.Itoa(ww.Status())
            endpoint := chi.RouteContext(r.Context()).RoutePattern()
            
            metrics.HTTPRequestsTotal.WithLabelValues(r.Method, endpoint, status).Inc()
            metrics.HTTPRequestDuration.WithLabelValues(r.Method, endpoint).Observe(duration)
            
            if ww.Status() >= 500 {
                metrics.HTTPErrorsTotal.WithLabelValues(r.Method, endpoint, status).Inc()
            }
        })
    }
}
```

**Critérios de aceite:**
- [ ] RED metrics para cada endpoint: rate, errors, duration (histogram com percentiles)
- [ ] USE metrics para infra: DB connections, Kafka consumer lag
- [ ] Custom business metrics: rides created, matching duration, active rides, surge multiplier
- [ ] Labels: service, method, endpoint, status_code, city_code
- [ ] `/metrics` endpoint no formato Prometheus
- [ ] Testes: criar corrida → `rides_created_total` incrementa; request lento → histogram registra

---

### Desafio 8.2 — Logs Estruturados com Correlation ID

Implemente logging estruturado (JSON) com traceId propagado em todos os serviços.

**Cenário:** Um passageiro reporta que a corrida demorou. O suporte precisa encontrar TODOS os logs relacionados a esse request, em TODOS os serviços. Sem traceId, é impossível correlacionar.

**Requisitos:**
- **Log format:** JSON estruturado com campos padronizados:
  ```json
  {
    "timestamp": "2025-01-15T10:30:45.123Z",
    "level": "ERROR",
    "service": "ride-service",
    "traceId": "abc-123-def-456",
    "spanId": "span-789",
    "userId": "user-123",
    "rideId": "ride-456",
    "message": "Timeout ao encontrar motorista",
    "error": "TimeoutException",
    "dependency": "driver-service",
    "duration_ms": 5000,
    "context": {
      "city": "SP",
      "retry_attempt": 3,
      "circuit_breaker_state": "HALF_OPEN"
    }
  }
  ```
- **TraceId propagation:** gerado no API Gateway, propagado via header `traceparent` (formato W3C)
- **MDC/context:** traceId, userId, rideId automaticamente incluídos em todos os logs do request
- **Log levels:**
  - `INFO` — eventos de negócio (corrida criada, motorista encontrado)
  - `WARN` — retry, circuit breaker half-open, timeout recuperado
  - `ERROR` — falha não recuperável, dependência down
- **NUNCA loggar:** passwords, tokens, cartão de crédito, CPF
- **Log correlation:** dado um traceId, encontrar logs em todos os serviços

**Java 25 (SLF4J + Logback JSON + MDC):**
```java
// logback-spring.xml
<configuration>
  <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <includeMdcKeyName>traceId</includeMdcKeyName>
      <includeMdcKeyName>spanId</includeMdcKeyName>
      <includeMdcKeyName>userId</includeMdcKeyName>
      <includeMdcKeyName>rideId</includeMdcKeyName>
      <customFields>{"service":"ride-service"}</customFields>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="JSON"/>
  </root>
</configuration>

// Filter para propagar traceId via MDC
@Component
public class TraceContextFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) throws ServletException, IOException {
        
        var traceId = request.getHeader("X-Trace-Id");
        if (traceId == null) {
            traceId = UUID.randomUUID().toString();
        }
        
        MDC.put("traceId", traceId);
        MDC.put("userId", request.getHeader("X-User-Id"));
        
        response.setHeader("X-Trace-Id", traceId);
        
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}

// Uso nos serviços — traceId incluído automaticamente
@Service
public class RideService {
    
    @Transactional
    public Ride createRide(CreateRideCommand cmd) {
        MDC.put("rideId", ride.getId().toString());
        
        log.info("Ride created", kv("origin", cmd.origin()), 
            kv("destination", cmd.destination()));
        
        try {
            var driver = matchingService.findDriver(ride);
            log.info("Driver matched", kv("driverId", driver.id()),
                kv("matchingDurationMs", duration));
        } catch (TimeoutException e) {
            log.error("Timeout finding driver", kv("dependency", "driver-service"),
                kv("timeoutMs", 5000), kv("retryAttempt", attempt));
            throw e;
        }
        
        return ride;
    }
}

// Propagar traceId ao chamar outro serviço
@Component
public class TracingWebClientFilter implements ExchangeFilterFunction {
    @Override
    public Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next) {
        var traceId = MDC.get("traceId");
        if (traceId != null) {
            request = ClientRequest.from(request)
                .header("X-Trace-Id", traceId)
                .build();
        }
        return next.exchange(request);
    }
}
```

**Go 1.26 (slog JSON + context):**
```go
// Logger setup
func NewLogger(serviceName string) *slog.Logger {
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    })
    return slog.New(handler).With("service", serviceName)
}

// Context keys
type ctxKey string
const (
    traceIDKey ctxKey = "traceId"
    userIDKey  ctxKey = "userId"
    rideIDKey  ctxKey = "rideId"
)

// Middleware: extrair/criar traceId
func TraceMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        traceID := r.Header.Get("X-Trace-Id")
        if traceID == "" {
            traceID = uuid.New().String()
        }
        
        ctx := context.WithValue(r.Context(), traceIDKey, traceID)
        if userID := r.Header.Get("X-User-Id"); userID != "" {
            ctx = context.WithValue(ctx, userIDKey, userID)
        }
        
        w.Header().Set("X-Trace-Id", traceID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Helper: log com context
func LogFromCtx(ctx context.Context) *slog.Logger {
    logger := slog.Default()
    if traceID, ok := ctx.Value(traceIDKey).(string); ok {
        logger = logger.With("traceId", traceID)
    }
    if userID, ok := ctx.Value(userIDKey).(string); ok {
        logger = logger.With("userId", userID)
    }
    if rideID, ok := ctx.Value(rideIDKey).(string); ok {
        logger = logger.With("rideId", rideID)
    }
    return logger
}

// Uso
func (s *RideService) CreateRide(ctx context.Context, cmd CreateRideCmd) (*Ride, error) {
    log := LogFromCtx(ctx)
    
    ride := newRide(cmd)
    ctx = context.WithValue(ctx, rideIDKey, ride.ID.String())
    log = LogFromCtx(ctx)
    
    log.Info("ride created",
        "origin", cmd.Origin,
        "destination", cmd.Destination)
    
    driver, err := s.matching.FindDriver(ctx, ride)
    if err != nil {
        log.Error("timeout finding driver",
            "dependency", "driver-service",
            "error", err.Error(),
            "retryAttempt", attempt)
        return nil, err
    }
    
    log.Info("driver matched",
        "driverId", driver.ID,
        "matchingDurationMs", duration.Milliseconds())
    
    return ride, nil
}

// Propagar traceId ao chamar outro serviço
func (c *DriverClient) GetDriver(ctx context.Context, id uuid.UUID) (*Driver, error) {
    req, _ := http.NewRequestWithContext(ctx, http.MethodGet,
        fmt.Sprintf("%s/api/drivers/%s", c.baseURL, id), nil)
    
    if traceID, ok := ctx.Value(traceIDKey).(string); ok {
        req.Header.Set("X-Trace-Id", traceID)
    }
    
    resp, err := c.httpClient.Do(req)
    // ...
}
```

**Critérios de aceite:**
- [ ] Logs em JSON estruturado com timestamp, level, service, traceId, message
- [ ] TraceId gerado no Gateway e propagado via header `X-Trace-Id` / `traceparent`
- [ ] MDC/context: traceId, userId, rideId automaticamente incluídos em todos os logs
- [ ] Dado um traceId, é possível filtrar logs de TODOS os serviços que participaram
- [ ] Secrets NUNCA logados (password, token, cartão)
- [ ] Testes: request com traceId → todos os logs do request contêm mesmo traceId

---

### Desafio 8.3 — Distributed Tracing com OpenTelemetry

Implemente tracing distribuído com OpenTelemetry para visualizar a jornada de um request.

**Cenário:** Um request `POST /passenger/rides` traversa: Gateway → BFF → Ride Service → Driver Service → Payment Service → Notification Service. Cada serviço gera um span. O trace completo é visualizado no Jaeger.

**Requisitos:**
- **OpenTelemetry SDK** em cada serviço (auto-instrumentation ou manual)
- **W3C Trace Context:** propagação via header `traceparent`
- **Spans** para cada operação:
  - HTTP server span (inbound request)
  - HTTP client span (outbound call para outro serviço)
  - Database span (query SQL)
  - Kafka producer/consumer span
- **Span attributes:** `http.method`, `http.url`, `http.status_code`, `db.statement`, `messaging.destination`
- **Span events:** `ride.created`, `driver.matched`, `payment.processed`
- **Error recording:** exception registrada no span com stack trace
- **Exporter:** Jaeger (via OTLP)
- **Sampling:** 100% em dev, 10% em prod (head-based sampling)

**Java 25 (OpenTelemetry + Spring Boot):**
```java
// application.yml — OpenTelemetry config
otel:
  service:
    name: ride-service
  exporter:
    otlp:
      endpoint: http://jaeger:4317
  traces:
    sampler:
      type: parentbased_traceidratio
      arg: "0.1"  # 10% em prod

// Auto-instrumentation via spring-boot-starter-actuator + opentelemetry
// Spans automáticos para HTTP, JDBC, Kafka etc.

// Manual span para lógica de negócio
@Service
public class RideMatchingService {
    private final Tracer tracer;
    
    public Driver findNearestDriver(Ride ride) {
        Span span = tracer.spanBuilder("ride.matching.findNearest")
            .setAttribute("ride.id", ride.getId().toString())
            .setAttribute("ride.city", ride.getCityCode())
            .setAttribute("ride.origin.lat", ride.getOrigin().lat())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            var drivers = driverClient.findNearby(ride.getOrigin(), 10);
            
            span.addEvent("drivers.found", Attributes.of(
                AttributeKey.longKey("count"), (long) drivers.size()));
            
            var best = scoringAlgorithm.rank(drivers, ride);
            
            span.addEvent("driver.matched", Attributes.of(
                AttributeKey.stringKey("driver.id"), best.getId().toString(),
                AttributeKey.doubleKey("distance.km"), best.getDistanceKm()));
            
            span.setAttribute("matching.duration_ms",
                Duration.between(ride.getCreatedAt(), Instant.now()).toMillis());
            
            return best;
            
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

**Go 1.26 (OpenTelemetry SDK):**
```go
// Tracer setup
func InitTracer(serviceName string) (*sdktrace.TracerProvider, error) {
    exporter, err := otlptracegrpc.New(context.Background(),
        otlptracegrpc.WithEndpoint("jaeger:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }
    
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String(serviceName),
        )),
        sdktrace.WithSampler(sdktrace.ParentBased(
            sdktrace.TraceIDRatioBased(0.1))), // 10% em prod
    )
    
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{}, propagation.Baggage{}))
    
    return tp, nil
}

// Manual span
func (s *RideMatchingService) FindNearestDriver(ctx context.Context, ride *Ride) (*Driver, error) {
    ctx, span := otel.Tracer("ride-service").Start(ctx, "ride.matching.findNearest",
        trace.WithAttributes(
            attribute.String("ride.id", ride.ID.String()),
            attribute.String("ride.city", ride.CityCode),
        ))
    defer span.End()
    
    drivers, err := s.driverClient.FindNearby(ctx, ride.Origin, 10)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }
    
    span.AddEvent("drivers.found", trace.WithAttributes(
        attribute.Int("count", len(drivers))))
    
    best := s.scoring.Rank(drivers, ride)
    
    span.AddEvent("driver.matched", trace.WithAttributes(
        attribute.String("driver.id", best.ID.String()),
        attribute.Float64("distance.km", best.DistanceKm)))
    
    return best, nil
}

// HTTP middleware para propagação automática
func OtelMiddleware(next http.Handler) http.Handler {
    return otelhttp.NewHandler(next, "")
}

// HTTP client com propagação
func NewTracedHTTPClient() *http.Client {
    return &http.Client{
        Transport: otelhttp.NewTransport(http.DefaultTransport),
    }
}
```

**Docker Compose (Jaeger):**
```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:1.54
    ports:
      - "16686:16686"  # UI
      - "4317:4317"    # OTLP gRPC
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
```

**Critérios de aceite:**
- [ ] Cada serviço gera spans (HTTP server, HTTP client, DB, Kafka)
- [ ] W3C `traceparent` header propagado entre serviços
- [ ] Span attributes: http.method, http.url, http.status_code, db.statement
- [ ] Span events para lógica de negócio (ride.created, driver.matched)
- [ ] Exceptions gravadas no span com stack trace
- [ ] Traces visíveis no Jaeger UI com waterfall view
- [ ] Sampling: 10% em prod (configurável)
- [ ] Testes: criar corrida → trace no Jaeger mostra jornada completa (Gateway→BFF→Ride→Driver→Payment)

---

### Desafio 8.4 — SLOs, SLIs & Error Budget

Defina SLOs para o RideFlow e implemente monitoramento de error budget.

**Cenário:** O time de produto quer garantias: "99.9% das corridas devem ser criadas em menos de 2 segundos" e "99.5% dos matchings devem ocorrer em menos de 30 segundos". Isso é um SLO.

**Requisitos:**
- **SLIs (Service Level Indicators):**
  - Availability SLI: `successful_requests / total_requests` (requests com status < 500)
  - Latency SLI: `requests_under_threshold / total_requests` (requests < 2s para criação, < 30s para matching)
  - Matching SLI: `rides_matched / rides_requested` (corridas que encontraram motorista)
- **SLOs (Service Level Objectives):**
  - Availability: 99.9% (mensal) → error budget = 0.1% = ~43 min/mês
  - Ride creation latency: 99.5% < 2s
  - Matching success: 99.0% das corridas matched em < 30s
- **Error Budget:**
  - Calcular: total_budget - consumed_budget = remaining_budget
  - Dashboard: "Error budget remaining: 75% (32 min de 43 min restantes)"
  - **Alerta** quando error budget < 25% restante
  - **Freeze deploys** quando error budget = 0% (política)
- **Burn rate alert:** alertar quando error budget está sendo consumido X vezes mais rápido que o esperado
  - Fast burn: 14.4x em 1h (gastaria budget em ~3 dias)
  - Slow burn: 6x em 6h (gastaria budget em ~5 dias)

**Java 25 (SLO Calculator):**
```java
@Service
public class SloMonitor {
    
    // SLI: Availability
    public double calculateAvailabilitySLI(Duration window) {
        var total = meterRegistry.get("http_requests_total").counters().stream()
            .mapToDouble(Counter::count).sum();
        var errors = meterRegistry.get("http_errors_total").counters().stream()
            .mapToDouble(Counter::count).sum();
        
        return total > 0 ? (total - errors) / total : 1.0;
    }
    
    // SLI: Latency
    public double calculateLatencySLI(Duration window, Duration threshold) {
        var histogram = meterRegistry.get("http_request_duration_seconds")
            .timer();
        var totalCount = histogram.count();
        // Requests under threshold via percentile query
        var underThreshold = histogram.takeSnapshot()
            .percentileValues().length; // simplified
        
        return totalCount > 0 ? (double) underThreshold / totalCount : 1.0;
    }
    
    // Error Budget
    public ErrorBudgetStatus calculateErrorBudget(double sloTarget, Duration period) {
        double currentSLI = calculateAvailabilitySLI(period);
        double totalBudget = 1.0 - sloTarget; // 0.001 para 99.9%
        double consumed = 1.0 - currentSLI;
        double remaining = Math.max(0, totalBudget - consumed);
        double remainingPercent = remaining / totalBudget * 100;
        
        double remainingMinutes = remaining * period.toMinutes();
        
        return new ErrorBudgetStatus(
            sloTarget, currentSLI,
            totalBudget, consumed, remaining,
            remainingPercent, remainingMinutes
        );
    }
    
    // Burn rate alert
    @Scheduled(fixedRate = 60_000) // a cada 1 min
    public void checkBurnRate() {
        // Fast burn: 14.4x em 1h
        double sli1h = calculateAvailabilitySLI(Duration.ofHours(1));
        double burnRate1h = (1 - sli1h) / (1 - 0.999); // SLO 99.9%
        
        if (burnRate1h > 14.4) {
            alertService.critical("FAST BURN: error budget being consumed at %.1fx rate (1h window)"
                .formatted(burnRate1h));
        }
        
        // Slow burn: 6x em 6h  
        double sli6h = calculateAvailabilitySLI(Duration.ofHours(6));
        double burnRate6h = (1 - sli6h) / (1 - 0.999);
        
        if (burnRate6h > 6.0) {
            alertService.warning("SLOW BURN: error budget being consumed at %.1fx rate (6h window)"
                .formatted(burnRate6h));
        }
    }
}
```

**Go 1.26:**
```go
type SLOMonitor struct {
    metrics     *Metrics
    alerter     Alerter
    sloTargets  map[string]float64
}

func NewSLOMonitor(metrics *Metrics, alerter Alerter) *SLOMonitor {
    return &SLOMonitor{
        metrics: metrics,
        alerter: alerter,
        sloTargets: map[string]float64{
            "availability":     0.999,  // 99.9%
            "ride_creation_p99": 0.995, // 99.5% < 2s
            "matching_success":  0.990, // 99.0%
        },
    }
}

type ErrorBudgetStatus struct {
    SLOTarget        float64 `json:"sloTarget"`
    CurrentSLI       float64 `json:"currentSLI"`
    TotalBudget      float64 `json:"totalBudget"`
    ConsumedBudget   float64 `json:"consumedBudget"`
    RemainingBudget  float64 `json:"remainingBudget"`
    RemainingPercent float64 `json:"remainingPercent"`
    RemainingMinutes float64 `json:"remainingMinutes"`
}

func (m *SLOMonitor) CalculateErrorBudget(sloTarget float64, period time.Duration) ErrorBudgetStatus {
    currentSLI := m.calculateAvailabilitySLI(period)
    totalBudget := 1.0 - sloTarget
    consumed := 1.0 - currentSLI
    remaining := max(0, totalBudget-consumed)
    remainingPercent := remaining / totalBudget * 100
    remainingMinutes := remaining * period.Minutes()
    
    return ErrorBudgetStatus{
        SLOTarget: sloTarget, CurrentSLI: currentSLI,
        TotalBudget: totalBudget, ConsumedBudget: consumed,
        RemainingBudget: remaining, RemainingPercent: remainingPercent,
        RemainingMinutes: remainingMinutes,
    }
}

// Burn rate monitor goroutine
func (m *SLOMonitor) MonitorBurnRate(ctx context.Context) {
    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            sli1h := m.calculateAvailabilitySLI(1 * time.Hour)
            burnRate1h := (1 - sli1h) / (1 - 0.999)
            
            if burnRate1h > 14.4 {
                m.alerter.Critical(fmt.Sprintf(
                    "FAST BURN: error budget consumed at %.1fx rate (1h)", burnRate1h))
            }
            
            sli6h := m.calculateAvailabilitySLI(6 * time.Hour)
            burnRate6h := (1 - sli6h) / (1 - 0.999)
            
            if burnRate6h > 6.0 {
                m.alerter.Warning(fmt.Sprintf(
                    "SLOW BURN: error budget consumed at %.1fx rate (6h)", burnRate6h))
            }
        }
    }
}

// API endpoint
func (m *SLOMonitor) StatusHandler(w http.ResponseWriter, r *http.Request) {
    status := map[string]ErrorBudgetStatus{
        "availability":    m.CalculateErrorBudget(0.999, 30*24*time.Hour),
        "ride_creation":   m.CalculateErrorBudget(0.995, 30*24*time.Hour),
        "matching_success": m.CalculateErrorBudget(0.990, 30*24*time.Hour),
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(status)
}
```

**Critérios de aceite:**
- [ ] SLIs calculados: availability, latency (< 2s), matching success rate
- [ ] SLOs definidos: 99.9% availability, 99.5% latency, 99.0% matching
- [ ] Error budget calculado: total, consumed, remaining (em % e minutos)
- [ ] API endpoint: `GET /admin/slo/status` retorna error budget de todos os SLOs
- [ ] Burn rate alerts: fast burn (14.4x/1h), slow burn (6x/6h)
- [ ] Alerta quando error budget < 25% restante
- [ ] Testes: simular 5 erros em 1000 requests → SLI = 99.5% → error budget de availability consumido
