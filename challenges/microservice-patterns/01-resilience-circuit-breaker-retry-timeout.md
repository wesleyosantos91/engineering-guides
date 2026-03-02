# Level 1 — Resiliência I: Circuit Breaker, Retry & Timeout

> **Objetivo:** Proteger a comunicação entre microsserviços contra falhas em cascata usando Circuit Breaker,
> Retry com backoff exponencial e Timeout em camadas — os três padrões fundamentais de resiliência.

**Pré-requisito:** [Level 0 — Fundações de Microsserviços](00-microservice-foundations.md)

**Referências:**
- [01-circuit-breaker.md](../../.docs/microservice-patterns/01-circuit-breaker.md)
- [02-retry.md](../../.docs/microservice-patterns/02-retry.md)
- [03-timeout.md](../../.docs/microservice-patterns/03-timeout.md)

---

## Contexto do Domínio

No Level 0, você observou os problemas de chamadas sem proteção. Agora, vamos corrigir:

- **Circuit Breaker** no Ride Service → Payment Service (evitar cascading failure quando pagamento está fora)
- **Retry** no Ride Service → Location Service (GPS pode ter falhas transitórias)
- **Timeout** em todas as chamadas inter-serviço (nenhuma chamada pode bloquear indefinidamente)

---

## Desafios

### Desafio 1.1 — Circuit Breaker: Protegendo o Payment Service

Implemente Circuit Breaker no Ride Service para proteger chamadas ao Payment Service.

**Cenário:** O Payment Service processa pagamentos de corridas finalizadas. Quando ele fica instável (alta latência ou erros 5xx), o Ride Service não pode ficar bloqueado esperando — precisa abrir o circuito e usar fallback.

**Requisitos:**
- Circuit Breaker com sliding window **COUNT_BASED** de 10 chamadas
- Threshold de falha: 50% (5 de 10 falham → abrir circuito)
- Slow call rate threshold: 80% (chamadas > 3s são consideradas lentas)
- Wait duration em OPEN: 30 segundos antes de tentar HALF_OPEN
- Permitted calls em HALF_OPEN: 3 chamadas de teste
- Fallback: quando circuito aberto, enfileirar pagamento para processamento posterior
- Registrar transições de estado do circuito via logs

**Java 25 (Resilience4j):**
```java
@Configuration
public class ResilienceConfig {
    
    @Bean
    public CircuitBreakerConfig paymentCircuitBreakerConfig() {
        return CircuitBreakerConfig.custom()
            .slidingWindowType(SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(10)
            .failureRateThreshold(50)
            .slowCallRateThreshold(80)
            .slowCallDurationThreshold(Duration.ofSeconds(3))
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(3)
            .recordExceptions(IOException.class, TimeoutException.class)
            .ignoreExceptions(BusinessException.class)
            .build();
    }
}

@Service
public class PaymentServiceClient {
    
    private final CircuitBreaker circuitBreaker;
    private final WebClient webClient;
    private final PaymentQueueService queueService;
    
    public PaymentResult processPayment(UUID rideId, BigDecimal amount) {
        return CircuitBreaker.decorateSupplier(circuitBreaker,
            () -> callPaymentService(rideId, amount))
            .recover(CallNotPermittedException.class,
                ex -> fallbackQueuePayment(rideId, amount))
            .get();
    }
    
    private PaymentResult fallbackQueuePayment(UUID rideId, BigDecimal amount) {
        queueService.enqueue(rideId, amount);
        return PaymentResult.queued("Payment queued - circuit breaker open");
    }
}
```

**Go 1.26 (gobreaker):**
```go
type PaymentServiceClient struct {
    cb         *gobreaker.CircuitBreaker
    httpClient *http.Client
    baseURL    string
    queue      *PaymentQueue
}

func NewPaymentServiceClient(baseURL string, queue *PaymentQueue) *PaymentServiceClient {
    settings := gobreaker.Settings{
        Name:        "payment-service",
        MaxRequests: 3, // permitted calls in half-open
        Interval:    0, // não reseta contadores no CLOSED (sliding window manual)
        Timeout:     30 * time.Second, // wait duration in OPEN
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
            return counts.Requests >= 10 && failureRatio >= 0.5
        },
        OnStateChange: func(name string, from, to gobreaker.State) {
            slog.Info("circuit breaker state change",
                "name", name, "from", from.String(), "to", to.String())
        },
    }
    
    return &PaymentServiceClient{
        cb:         gobreaker.NewCircuitBreaker(settings),
        httpClient: &http.Client{Timeout: 5 * time.Second},
        baseURL:    baseURL,
        queue:      queue,
    }
}

func (c *PaymentServiceClient) ProcessPayment(
    ctx context.Context, rideID uuid.UUID, amount float64,
) (*PaymentResult, error) {
    result, err := c.cb.Execute(func() (interface{}, error) {
        return c.callPaymentService(ctx, rideID, amount)
    })
    
    if errors.Is(err, gobreaker.ErrOpenState) || errors.Is(err, gobreaker.ErrTooManyRequests) {
        // Fallback: enfileirar pagamento
        c.queue.Enqueue(rideID, amount)
        return &PaymentResult{Status: "QUEUED", Message: "Circuit breaker open"}, nil
    }
    
    if err != nil {
        return nil, fmt.Errorf("payment processing failed: %w", err)
    }
    return result.(*PaymentResult), nil
}
```

**Critérios de aceite:**
- [ ] Circuit Breaker configurado com COUNT_BASED sliding window de 10
- [ ] Circuito abre quando failure rate ≥ 50%
- [ ] Circuito transita: CLOSED → OPEN → HALF_OPEN → CLOSED (ou volta a OPEN)
- [ ] Fallback funcional: pagamentos enfileirados quando circuito aberto
- [ ] Logs registram cada transição de estado (CLOSED→OPEN, OPEN→HALF_OPEN, etc.)
- [ ] Exceções de negócio (4xx) NÃO contam como falha do circuito
- [ ] Testes: simular 10 falhas → circuito abre → fallback ativado → após 30s → half-open → recuperação

---

### Desafio 1.2 — Retry com Backoff Exponencial: Location Service

Implemente Retry com backoff exponencial e jitter para chamadas ao Location Service.

**Cenário:** O Location Service fornece coordenadas GPS dos motoristas. A comunicação com GPS é instável — falhas transitórias são comuns (timeout, connection reset). Retry com backoff exponencial resolve 90% desses casos.

**Requisitos:**
- Máximo de 3 tentativas (1 original + 2 retries)
- Backoff exponencial: 500ms → 1000ms → 2000ms
- **Full jitter** para evitar thundering herd: `random(0, backoff)`
- Retryable: `IOException`, `TimeoutException`, HTTP 503/502/429
- **Não** retryable: HTTP 400, 401, 403, 404 (erros de cliente)
- Cada tentativa logada com número da tentativa e tempo de espera
- Retry **exige** que a operação do Location Service seja idempotente (GET)

**Java 25 (Resilience4j):**
```java
@Configuration
public class RetryConfig {
    
    @Bean
    public io.github.resilience4j.retry.RetryConfig locationRetryConfig() {
        return io.github.resilience4j.retry.RetryConfig.custom()
            .maxAttempts(3)
            .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
                Duration.ofMillis(500),  // initial interval
                2.0,                      // multiplier
                Duration.ofSeconds(5)     // max interval
            ))
            .retryOnException(ex -> ex instanceof IOException 
                || ex instanceof TimeoutException)
            .retryOnResult(response -> {
                if (response instanceof ResponseEntity<?> r) {
                    int status = r.getStatusCode().value();
                    return status == 502 || status == 503 || status == 429;
                }
                return false;
            })
            .ignoreExceptions(BusinessValidationException.class)
            .build();
    }
}

@Service
public class LocationServiceClient {
    
    private final Retry retry;
    private final WebClient webClient;
    
    public DriverLocation getDriverLocation(UUID driverId) {
        return Retry.decorateSupplier(retry,
            () -> fetchLocation(driverId))
            .get();
    }
}
```

**Go 1.26 (retry-go):**
```go
func (c *LocationServiceClient) GetDriverLocation(
    ctx context.Context, driverID uuid.UUID,
) (*DriverLocation, error) {
    var location *DriverLocation
    
    err := retry.Do(
        func() error {
            loc, err := c.fetchLocation(ctx, driverID)
            if err != nil {
                return err
            }
            location = loc
            return nil
        },
        retry.Attempts(3),
        retry.Delay(500*time.Millisecond),
        retry.DelayType(retry.BackOffDelay),
        retry.MaxJitter(500*time.Millisecond),
        retry.RetryIf(func(err error) bool {
            // Retry apenas erros transitórios
            var httpErr *HTTPError
            if errors.As(err, &httpErr) {
                return httpErr.StatusCode == 502 || 
                       httpErr.StatusCode == 503 || 
                       httpErr.StatusCode == 429
            }
            // Retry em erros de rede
            return !errors.Is(err, ErrClientError) // não retry em 4xx
        }),
        retry.OnRetry(func(n uint, err error) {
            slog.Warn("retrying location service",
                "attempt", n+1, "error", err.Error())
        }),
        retry.Context(ctx),
    )
    
    return location, err
}
```

**Critérios de aceite:**
- [ ] Máximo 3 tentativas com backoff exponencial (500ms → 1s → 2s)
- [ ] Full jitter aplicado (tempos de espera não são determinísticos)
- [ ] Retry apenas para erros transitórios (5xx, IOException, Timeout)
- [ ] Erros de cliente (4xx) falham imediatamente (sem retry)
- [ ] Cada tentativa logada com número e tempo de espera
- [ ] Contexto propagado (cancelamento do request cancela retries pendentes)
- [ ] Testes: simular 2 falhas transitórias → 3ª tentativa sucesso → resultado correto

---

### Desafio 1.3 — Timeout em Camadas: Proteção Contra Bloqueio

Implemente timeouts em múltiplas camadas para garantir que nenhuma operação bloqueia indefinidamente.

**Cenário:** O Ride Service chama Pricing Service (cálculo) e Driver Service (matching). Sem timeout, uma chamada lenta bloqueia a thread, que bloqueia a request, que bloqueia o usuário. Timeouts em camadas garantem fail-fast.

**Requisitos:**
- **Connection timeout:** 3 segundos (estabelecer conexão TCP)
- **Read timeout:** 5 segundos (aguardar resposta completa)
- **Request timeout (application-level):** 10 segundos (tempo total da operação)
- O timeout do Ride Service deve ser **menor** que o timeout do API Gateway
- Timeout configurável por dependência:
  - Pricing Service: 3s (cálculo deve ser rápido)
  - Driver Service: 5s (matching pode demorar mais)
  - Location Service: 2s (GPS é fast ou fail)
- Quando timeout ocorre: retornar erro específico (não genérico 500)
- **Timeout ≠ cancelamento**: garantir que o contexto é cancelado no caller

**Java 25 (WebClient + TimeLimiter):**
```java
@Service
public class PricingServiceClient {
    
    private final WebClient webClient;
    private final TimeLimiter timeLimiter;
    
    public PricingServiceClient(
            @Value("${services.pricing.url}") String url,
            WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl(url)
            .clientConnector(new ReactorClientHttpConnector(
                HttpClient.create()
                    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3_000)
                    .responseTimeout(Duration.ofSeconds(5))
            ))
            .build();
        
        this.timeLimiter = TimeLimiter.of(TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(3)) // pricing deve ser rápido
            .cancelRunningFuture(true)
            .build());
    }
    
    public PriceEstimate estimatePrice(GeoPoint origin, GeoPoint destination) {
        var future = CompletableFuture.supplyAsync(() ->
            webClient.post()
                .uri("/pricing/estimate")
                .bodyValue(new PriceRequest(origin, destination))
                .retrieve()
                .bodyToMono(PriceEstimate.class)
                .block()
        );
        
        try {
            return timeLimiter.executeFutureSupplier(() -> future);
        } catch (TimeoutException e) {
            throw new ServiceTimeoutException("Pricing service timeout after 3s", e);
        }
    }
}
```

**Go 1.26 (context.WithTimeout):**
```go
type PricingServiceClient struct {
    baseURL string
    client  *http.Client
}

func NewPricingServiceClient(baseURL string) *PricingServiceClient {
    transport := &http.Transport{
        DialContext: (&net.Dialer{
            Timeout: 3 * time.Second, // connection timeout
        }).DialContext,
        ResponseHeaderTimeout: 5 * time.Second, // read timeout
    }
    
    return &PricingServiceClient{
        baseURL: baseURL,
        client:  &http.Client{Transport: transport},
    }
}

func (c *PricingServiceClient) EstimatePrice(
    ctx context.Context, origin, destination GeoPoint,
) (*PriceEstimate, error) {
    // Application-level timeout (menor que gateway)
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()
    
    body, _ := json.Marshal(PriceRequest{Origin: origin, Destination: destination})
    req, err := http.NewRequestWithContext(ctx, http.MethodPost,
        c.baseURL+"/pricing/estimate", bytes.NewReader(body))
    if err != nil {
        return nil, fmt.Errorf("creating request: %w", err)
    }
    req.Header.Set("Content-Type", "application/json")
    
    resp, err := c.client.Do(req)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            return nil, &ServiceTimeoutError{
                Service: "pricing", Timeout: 3 * time.Second,
            }
        }
        return nil, fmt.Errorf("calling pricing service: %w", err)
    }
    defer resp.Body.Close()
    
    var estimate PriceEstimate
    if err := json.NewDecoder(resp.Body).Decode(&estimate); err != nil {
        return nil, fmt.Errorf("decoding response: %w", err)
    }
    return &estimate, nil
}
```

**Critérios de aceite:**
- [ ] 3 camadas de timeout: connection (3s), read (5s), application (variável)
- [ ] Timeout configurável por serviço-alvo (Pricing 3s, Driver 5s, Location 2s)
- [ ] Timeout do Ride Service < timeout do API Gateway (hierarquia respeitada)
- [ ] Exceção/erro específico para timeout (não genérico 500)
- [ ] Contexto cancelado quando timeout ocorre (não deixar goroutine/thread pendurada)
- [ ] Logs com duração real da chamada vs timeout configurado
- [ ] Testes: serviço lento → timeout após tempo configurado → erro específico retornado

---

### Desafio 1.4 — Fallback Strategies: Degradação Graciosa

Implemente estratégias de fallback para cada tipo de falha (circuit breaker aberto, timeout, erro).

**Cenário:** Quando o Pricing Service falha, o Ride Service não deve retornar erro ao passageiro. Em vez disso, deve usar uma estratégia de fallback (preço estimado com cache, preço default, ou informar que o preço será calculado depois).

**Requisitos:**
- **Pricing Service fallback:** usar último preço conhecido (cache local) ou preço base por categoria
- **Driver Service fallback:** retornar lista vazia + mensagem "buscando motoristas..." (async)
- **Location Service fallback:** usar última localização conhecida do motorista (TTL 30s)
- Cada fallback é logado como WARNING com motivo (CB open, timeout, error)
- Métricas: contar quantas vezes cada fallback é ativado
- O passageiro NÃO sabe que houve fallback — experiência transparente

**Java 25:**
```java
@Service
public class ResilientPricingClient {
    
    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final TimeLimiter timeLimiter;
    private final Cache<String, PriceEstimate> priceCache;
    private final PricingDefaults defaults;
    
    public PriceEstimate estimatePrice(GeoPoint origin, GeoPoint dest, VehicleCategory cat) {
        var cacheKey = "%s_%s_%s".formatted(origin.hash(), dest.hash(), cat);
        
        Supplier<PriceEstimate> decoratedCall = Decorators
            .ofSupplier(() -> callPricingService(origin, dest, cat))
            .withTimeLimiter(timeLimiter, Executors.newVirtualThreadPerTaskExecutor())
            .withRetry(retry)
            .withCircuitBreaker(circuitBreaker)
            .withFallback(List.of(
                CallNotPermittedException.class,
                TimeoutException.class,
                IOException.class
            ), ex -> fallbackPrice(cacheKey, cat, ex))
            .decorate();
        
        return decoratedCall.get();
    }
    
    private PriceEstimate fallbackPrice(String cacheKey, VehicleCategory cat, Throwable ex) {
        log.warn("Pricing fallback activated: {}", ex.getMessage());
        metricsService.incrementFallbackCounter("pricing");
        
        // 1. Tentar cache
        var cached = priceCache.get(cacheKey);
        if (cached != null) {
            return cached.withSource("CACHE");
        }
        
        // 2. Preço default por categoria
        return defaults.getDefaultPrice(cat).withSource("DEFAULT");
    }
}
```

**Go 1.26:**
```go
type ResilientPricingClient struct {
    client   *PricingServiceClient
    cb       *gobreaker.CircuitBreaker
    cache    *ttlcache.Cache[string, *PriceEstimate]
    defaults *PricingDefaults
    metrics  *FallbackMetrics
}

func (c *ResilientPricingClient) EstimatePrice(
    ctx context.Context, origin, dest GeoPoint, category VehicleCategory,
) (*PriceEstimate, error) {
    cacheKey := fmt.Sprintf("%s_%s_%s", origin.Hash(), dest.Hash(), category)
    
    result, err := c.cb.Execute(func() (interface{}, error) {
        ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
        defer cancel()
        return c.client.EstimatePrice(ctx, origin, dest, category)
    })
    
    if err != nil {
        // Fallback strategy
        slog.Warn("pricing fallback activated", "error", err.Error())
        c.metrics.IncrementFallback("pricing")
        
        // 1. Tentar cache
        if cached := c.cache.Get(cacheKey); cached != nil {
            estimate := cached.Value()
            estimate.Source = "CACHE"
            return estimate, nil
        }
        
        // 2. Preço default por categoria
        defaultPrice := c.defaults.GetDefaultPrice(category)
        defaultPrice.Source = "DEFAULT"
        return defaultPrice, nil
    }
    
    // Sucesso: atualizar cache
    estimate := result.(*PriceEstimate)
    estimate.Source = "LIVE"
    c.cache.Set(cacheKey, estimate, 5*time.Minute)
    return estimate, nil
}
```

**Critérios de aceite:**
- [ ] Fallback para Pricing: cache local → preço default por categoria
- [ ] Fallback para Driver: lista vazia com flag "procurando assíncronamente"
- [ ] Fallback para Location: última localização conhecida (TTL 30s máximo)
- [ ] Cada fallback distingue a fonte da resposta (LIVE, CACHE, DEFAULT)
- [ ] Contador de fallbacks por serviço (métrica exportável)
- [ ] Logs WARNING em cada ativação de fallback com motivo
- [ ] Testes: falha no serviço → fallback correto → resposta sem erro para o cliente

---

### Desafio 1.5 — Configuração e Métricas dos Padrões de Resiliência

Externalize toda a configuração de resiliência e exponha métricas para monitoramento.

**Cenário:** Os valores de timeout, threshold do circuit breaker e número de retries não devem ser hardcoded. Devem ser configuráveis por ambiente (dev: mais permissivo, prod: mais restritivo) e acessíveis via métricas para dashboards.

**Requisitos:**
- Toda configuração de resiliência via `application.yml` (Java) ou `config.yaml` (Go)
- Configuração diferente por perfil/ambiente (dev, staging, prod)
- **Métricas expostas:**
  - Circuit Breaker: estado atual, failure rate, slow call rate, calls totais
  - Retry: tentativas totais, sucessos após retry, falhas após todas tentativas
  - Timeout: chamadas com timeout, duração média, p99
- Endpoint `/actuator/metrics` (Java) ou `/metrics` (Go) no formato Prometheus
- Dashboard config (Grafana JSON ou descrição do dashboard desejado)

**Java 25 (application.yml):**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment-service:
        sliding-window-size: 10
        failure-rate-threshold: 50
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 3s
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        register-health-indicator: true
      pricing-service:
        sliding-window-size: 20
        failure-rate-threshold: 60
        wait-duration-in-open-state: 15s
  retry:
    instances:
      location-service:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
  timelimiter:
    instances:
      pricing-service:
        timeout-duration: 3s
        cancel-running-future: true
      driver-service:
        timeout-duration: 5s

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,circuitbreakers,retries
  metrics:
    export:
      prometheus:
        enabled: true
```

**Go 1.26 (config.yaml + Viper):**
```go
type ResilienceConfig struct {
    CircuitBreaker map[string]CBConfig `yaml:"circuit_breaker"`
    Retry          map[string]RetryConfig `yaml:"retry"`
    Timeout        map[string]TimeoutConfig `yaml:"timeout"`
}

type CBConfig struct {
    SlidingWindowSize   int           `yaml:"sliding_window_size"`
    FailureRateThreshold float64      `yaml:"failure_rate_threshold"`
    WaitDurationOpen    time.Duration `yaml:"wait_duration_open"`
    PermittedHalfOpen   int           `yaml:"permitted_half_open"`
}

func LoadResilienceConfig() (*ResilienceConfig, error) {
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath("./config")
    
    // Environment-specific override
    env := os.Getenv("APP_ENV")
    if env != "" {
        viper.SetConfigName("config." + env) // config.prod.yaml
    }
    
    if err := viper.ReadInConfig(); err != nil {
        return nil, fmt.Errorf("reading config: %w", err)
    }
    
    var cfg ResilienceConfig
    if err := viper.Unmarshal(&cfg); err != nil {
        return nil, fmt.Errorf("unmarshaling config: %w", err)
    }
    return &cfg, nil
}
```

**Critérios de aceite:**
- [ ] Toda configuração de resiliência externalizada (zero hardcode)
- [ ] Perfis de ambiente: dev (permissivo), prod (restritivo)
- [ ] Métricas de Circuit Breaker: estado, failure rate, slow call rate
- [ ] Métricas de Retry: tentativas, sucesso após retry, falhas finais
- [ ] Métricas de Timeout: total de timeouts, histograma de duração
- [ ] Endpoint de métricas no formato Prometheus
- [ ] Testes: alterar config → comportamento do CB/retry/timeout muda conforme esperado
