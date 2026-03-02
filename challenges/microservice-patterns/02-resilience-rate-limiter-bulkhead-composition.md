# Level 2 — Resiliência II: Rate Limiter, Bulkhead & Composição

> **Objetivo:** Completar o toolkit de resiliência com Rate Limiter (controle de taxa),
> Bulkhead (isolamento de recursos) e dominar a composição ordenada de todos os padrões de resiliência.

**Pré-requisito:** [Level 1 — Resiliência I](01-resilience-circuit-breaker-retry-timeout.md)

**Referências:**
- [04-rate-limiter.md](../../.docs/microservice-patterns/04-rate-limiter.md)
- [05-bulkhead.md](../../.docs/microservice-patterns/05-bulkhead.md)
- [06-composicao-resiliencia.md](../../.docs/microservice-patterns/06-composicao-resiliencia.md)

---

## Contexto do Domínio

No RideFlow, precisamos proteger os serviços de abuso e garantir isolamento:

- **Rate Limiter** — Limitar requisições de estimativa de preço por passageiro (anti-abuse) e no API Gateway
- **Bulkhead** — Isolar chamadas ao Payment Service (crítico) das chamadas ao Notification Service (best-effort)
- **Composição** — Orquestrar Retry → Circuit Breaker → Rate Limiter → Timeout → Bulkhead em cada chamada

---

## Desafios

### Desafio 2.1 — Rate Limiter: Proteção do Pricing Service

Implemente Rate Limiter para controlar a taxa de requisições ao Pricing Service.

**Cenário:** Um passageiro pode solicitar estimativas de preço repetidamente (abrindo o app, mudando destino). Sem rate limiting, um único usuário poderia sobrecarregar o Pricing Service. Além disso, o API Gateway precisa de rate limiting global.

**Requisitos:**
- **Por passageiro:** máximo 10 requisições por minuto ao Pricing Service (Token Bucket)
- **Global no Gateway:** máximo 1000 req/s por instância (Sliding Window Counter)
- Resposta HTTP 429 (Too Many Requests) quando limite atingido
- Headers na resposta: `Retry-After`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- Rate limiter **distribuído** usando Redis (para múltiplas instâncias)
- Dimensões: por IP, por userId, por endpoint
- Logs com informação de quem foi rate-limited e por quê

**Java 25 (Resilience4j):**
```java
@Configuration
public class RateLimiterConfig {
    
    @Bean
    public io.github.resilience4j.ratelimiter.RateLimiterConfig pricingRateLimiterConfig() {
        return io.github.resilience4j.ratelimiter.RateLimiterConfig.custom()
            .limitForPeriod(10)                    // 10 chamadas
            .limitRefreshPeriod(Duration.ofMinutes(1))  // por minuto
            .timeoutDuration(Duration.ZERO)        // fail-fast (não esperar)
            .build();
    }
}

@Service
public class PricingRateLimitedClient {
    
    private final RateLimiterRegistry registry;
    private final PricingServiceClient delegate;
    
    public PriceEstimate estimatePrice(UUID passengerId, GeoPoint origin, GeoPoint dest) {
        // Rate limiter por passageiro
        var rateLimiter = registry.rateLimiter(
            "pricing-" + passengerId,
            "pricing-per-user");
        
        return RateLimiter.decorateSupplier(rateLimiter,
            () -> delegate.estimatePrice(origin, dest))
            .get();
    }
}

// Filtro no Gateway para rate limiting global
@Component
public class GlobalRateLimitFilter implements WebFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        var response = exchange.getResponse();
        
        if (!rateLimiter.acquirePermission()) {
            response.setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            response.getHeaders().set("Retry-After", "1");
            response.getHeaders().set("X-RateLimit-Limit", "1000");
            response.getHeaders().set("X-RateLimit-Remaining", "0");
            return response.setComplete();
        }
        
        return chain.filter(exchange);
    }
}
```

**Go 1.26 (x/time/rate + Redis):**
```go
type RateLimiterMiddleware struct {
    limiters sync.Map // per-user limiters
    global   *rate.Limiter
}

func NewRateLimiterMiddleware() *RateLimiterMiddleware {
    return &RateLimiterMiddleware{
        global: rate.NewLimiter(1000, 100), // 1000/s, burst 100
    }
}

func (m *RateLimiterMiddleware) PerUser(limit rate.Limit, burst int) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            userID := r.Header.Get("X-User-ID")
            if userID == "" {
                userID = r.RemoteAddr
            }
            
            limiter := m.getLimiter(userID, limit, burst)
            
            if !limiter.Allow() {
                w.Header().Set("Retry-After", "60")
                w.Header().Set("X-RateLimit-Limit", fmt.Sprintf("%d", burst))
                w.Header().Set("X-RateLimit-Remaining", "0")
                http.Error(w, `{"error": "rate limit exceeded"}`,
                    http.StatusTooManyRequests)
                slog.Warn("rate limited", "userID", userID, "endpoint", r.URL.Path)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

func (m *RateLimiterMiddleware) getLimiter(key string, limit rate.Limit, burst int) *rate.Limiter {
    if v, ok := m.limiters.Load(key); ok {
        return v.(*rate.Limiter)
    }
    limiter := rate.NewLimiter(limit, burst)
    m.limiters.Store(key, limiter)
    return limiter
}
```

**Critérios de aceite:**
- [ ] Rate limiter por passageiro: 10 req/min ao Pricing Service
- [ ] Rate limiter global: 1000 req/s no API Gateway
- [ ] HTTP 429 com headers corretos (Retry-After, X-RateLimit-*)
- [ ] Múltiplas dimensões: por IP, por userId, por endpoint
- [ ] Rate limiter distribuído com Redis (funciona com múltiplas instâncias)
- [ ] Fail-fast: não enfileirar, retornar 429 imediatamente
- [ ] Testes: enviar 15 requisições em 1 minuto → primeiras 10 OK, últimas 5 recebem 429

---

### Desafio 2.2 — Bulkhead: Isolamento de Recursos

Implemente Bulkhead para isolar chamadas a dependências com diferentes criticidades.

**Cenário:** O Ride Service chama Payment Service (crítico — sem pagamento não há corrida) e Notification Service (best-effort — push notification pode atrasar). Se Notification Service ficar lento, NÃO pode afetar a capacidade do Payment Service.

**Requisitos:**
- **Semaphore Bulkhead** para Payment Service: max 20 chamadas concorrentes
- **Thread Pool Bulkhead** para Notification Service: max 5 threads dedicadas, queue de 10
- Quando bulkhead está cheio: fail-fast (não esperar na fila para Payment, esperar 500ms para Notification)
- Dimensionamento: throughput × latência média × 1.5 margem
- Métricas: concurrent calls, available permissions, waiting queue size

**Java 25 (Resilience4j Bulkhead):**
```java
@Configuration
public class BulkheadConfig {
    
    @Bean
    public io.github.resilience4j.bulkhead.BulkheadConfig paymentBulkheadConfig() {
        return io.github.resilience4j.bulkhead.BulkheadConfig.custom()
            .maxConcurrentCalls(20)
            .maxWaitDuration(Duration.ZERO) // fail-fast para pagamento
            .build();
    }
    
    @Bean
    public ThreadPoolBulkheadConfig notificationBulkheadConfig() {
        return ThreadPoolBulkheadConfig.custom()
            .maxThreadPoolSize(5)
            .coreThreadPoolSize(3)
            .queueCapacity(10)
            .keepAliveDuration(Duration.ofSeconds(30))
            .build();
    }
}

@Service
public class ResilientPaymentClient {
    
    private final Bulkhead bulkhead;
    private final PaymentServiceClient delegate;
    
    public PaymentResult processPayment(UUID rideId, BigDecimal amount) {
        return Bulkhead.decorateSupplier(bulkhead,
            () -> delegate.processPayment(rideId, amount))
            .recover(BulkheadFullException.class,
                ex -> {
                    log.error("Payment bulkhead full - system overloaded");
                    throw new ServiceOverloadedException("Payment service overloaded");
                })
            .get();
    }
}

@Service
public class ResilientNotificationClient {
    
    private final ThreadPoolBulkhead bulkhead;
    
    public CompletableFuture<Void> sendPushNotification(UUID userId, String message) {
        return bulkhead.executeSupplier(() -> {
            notificationClient.send(userId, message);
            return null;
        }).recover(BulkheadFullException.class, ex -> {
            log.warn("Notification bulkhead full - dropping notification");
            return null; // Best-effort: não falhar a corrida
        }).toCompletableFuture();
    }
}
```

**Go 1.26 (semaphore pattern):**
```go
// Semaphore Bulkhead para Payment
type SemaphoreBulkhead struct {
    name    string
    sem     chan struct{}
    metrics *BulkheadMetrics
}

func NewSemaphoreBulkhead(name string, maxConcurrent int) *SemaphoreBulkhead {
    return &SemaphoreBulkhead{
        name:    name,
        sem:     make(chan struct{}, maxConcurrent),
        metrics: NewBulkheadMetrics(name),
    }
}

func (b *SemaphoreBulkhead) Execute(ctx context.Context, fn func() (any, error)) (any, error) {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
        b.metrics.RecordPermitted()
        return fn()
    default:
        b.metrics.RecordRejected()
        return nil, &BulkheadFullError{Name: b.name}
    }
}

// Thread Pool Bulkhead para Notification (worker pool)
type WorkerPoolBulkhead struct {
    name    string
    jobs    chan func()
    metrics *BulkheadMetrics
}

func NewWorkerPoolBulkhead(name string, workers, queueSize int) *WorkerPoolBulkhead {
    bp := &WorkerPoolBulkhead{
        name:    name,
        jobs:    make(chan func(), queueSize),
        metrics: NewBulkheadMetrics(name),
    }
    
    for i := 0; i < workers; i++ {
        go func() {
            for job := range bp.jobs {
                job()
            }
        }()
    }
    return bp
}

func (b *WorkerPoolBulkhead) Submit(fn func()) error {
    select {
    case b.jobs <- fn:
        b.metrics.RecordPermitted()
        return nil
    default:
        b.metrics.RecordRejected()
        slog.Warn("notification bulkhead full - dropping", "name", b.name)
        return &BulkheadFullError{Name: b.name}
    }
}
```

**Critérios de aceite:**
- [ ] Semaphore Bulkhead para Payment: max 20 concorrentes, fail-fast (maxWait = 0)
- [ ] Worker Pool Bulkhead para Notification: 5 workers, queue 10, best-effort
- [ ] Payment: quando bulkhead full → erro 503 (serviço sobrecarregado)
- [ ] Notification: quando bulkhead full → drop silencioso (log warning, não falhar corrida)
- [ ] Métricas: concurrent calls, available permissions, rejected count
- [ ] Dimensionamento documentado (throughput × latência × 1.5)
- [ ] Testes: 25 chamadas concorrentes ao Payment → 20 processadas, 5 rejeitadas

---

### Desafio 2.3 — Composição de Padrões de Resiliência

Orquestre todos os padrões de resiliência na ordem correta para cada dependência do Ride Service.

**Cenário:** Cada chamada a um serviço externo deve passar por uma cadeia ordenada de padrões de resiliência. A ordem importa: **Retry → Circuit Breaker → Rate Limiter → Timeout → Bulkhead → chamada real**.

**Requisitos:**
- Composição ordenada para **Payment Service** (crítico):
  - Retry: 2 tentativas, backoff 1s
  - Circuit Breaker: sliding window 10, threshold 50%
  - Rate Limiter: não (pagamento não tem rate limit interno)
  - Timeout: 5s
  - Bulkhead: semaphore 20
- Composição para **Notification Service** (best-effort):
  - Retry: 1 tentativa (sem retry adicional)
  - Circuit Breaker: sliding window 5, threshold 80% (mais permissivo)
  - Rate Limiter: não
  - Timeout: 2s
  - Bulkhead: thread pool 5
- Composição para **Pricing Service** (moderado):
  - Retry: 3 tentativas, backoff 500ms
  - Circuit Breaker: sliding window 20, threshold 60%
  - Rate Limiter: 10/min por passageiro
  - Timeout: 3s
  - Bulkhead: semaphore 30
- Documentar a justificativa para a configuração de cada serviço
- Tabela de decisão: qual padrão aplicar baseado nas características do serviço-alvo

**Java 25 (Resilience4j Decorators):**
```java
@Service
public class ResilientServiceFactory {
    
    public <T> Supplier<T> decorateForPayment(Supplier<T> call) {
        return Decorators.ofSupplier(call)
            .withBulkhead(paymentBulkhead)        // 5. Bulkhead (mais externo, executa primeiro)
            .withTimeLimiter(paymentTimeLimiter,    // 4. Timeout
                Executors.newVirtualThreadPerTaskExecutor())
            .withCircuitBreaker(paymentCB)         // 2. Circuit Breaker
            .withRetry(paymentRetry)               // 1. Retry (mais interno, decide se retenta)
            .withFallback(List.of(Exception.class), 
                ex -> paymentFallback(ex))
            .decorate();
    }
    
    public <T> Supplier<T> decorateForNotification(Supplier<T> call) {
        return Decorators.ofSupplier(call)
            .withBulkhead(notificationBulkhead)
            .withTimeLimiter(notificationTimeLimiter,
                Executors.newVirtualThreadPerTaskExecutor())
            .withCircuitBreaker(notificationCB)
            .withFallback(List.of(Exception.class),
                ex -> {
                    log.warn("Notification failed - best effort: {}", ex.getMessage());
                    return null;
                })
            .decorate();
    }
    
    public <T> Supplier<T> decorateForPricing(UUID passengerId, Supplier<T> call) {
        var userRateLimiter = rateLimiterRegistry.rateLimiter(
            "pricing-" + passengerId, "pricing-per-user");
        
        return Decorators.ofSupplier(call)
            .withBulkhead(pricingBulkhead)
            .withTimeLimiter(pricingTimeLimiter,
                Executors.newVirtualThreadPerTaskExecutor())
            .withRateLimiter(userRateLimiter)
            .withCircuitBreaker(pricingCB)
            .withRetry(pricingRetry)
            .withFallback(List.of(Exception.class),
                ex -> pricingFallback(ex))
            .decorate();
    }
}
```

**Go 1.26 (middleware chain):**
```go
// Middleware pattern para composição
type ServiceCall func(ctx context.Context) (any, error)
type Middleware func(ServiceCall) ServiceCall

func ComposeMiddleware(call ServiceCall, middlewares ...Middleware) ServiceCall {
    // Aplica na ordem reversa para que a execução siga a ordem correta
    for i := len(middlewares) - 1; i >= 0; i-- {
        call = middlewares[i](call)
    }
    return call
}

// Composição para Payment Service
func NewPaymentCallChain(
    paymentClient *PaymentServiceClient,
    bulkhead *SemaphoreBulkhead,
    cb *gobreaker.CircuitBreaker,
) ServiceCall {
    rawCall := func(ctx context.Context) (any, error) {
        return paymentClient.ProcessPayment(ctx)
    }
    
    return ComposeMiddleware(rawCall,
        WithRetry(2, 1*time.Second),           // 1. Retry
        WithCircuitBreaker(cb),                 // 2. Circuit Breaker
        WithTimeout(5*time.Second),             // 4. Timeout
        WithBulkhead(bulkhead),                 // 5. Bulkhead
        WithFallback(paymentFallback),          // 6. Fallback
    )
}

// Tabela de decisão por serviço
var serviceConfigs = map[string]ResilienceProfile{
    "payment": {
        Retry: RetryConfig{MaxAttempts: 2, Delay: 1 * time.Second},
        CB:    CBConfig{WindowSize: 10, Threshold: 0.5},
        Timeout: 5 * time.Second,
        Bulkhead: BulkheadConfig{MaxConcurrent: 20, Type: "semaphore"},
        Criticality: "CRITICAL",
    },
    "notification": {
        Retry: RetryConfig{MaxAttempts: 1},
        CB:    CBConfig{WindowSize: 5, Threshold: 0.8},
        Timeout: 2 * time.Second,
        Bulkhead: BulkheadConfig{MaxConcurrent: 5, Type: "worker-pool"},
        Criticality: "BEST_EFFORT",
    },
    "pricing": {
        Retry: RetryConfig{MaxAttempts: 3, Delay: 500 * time.Millisecond},
        CB:    CBConfig{WindowSize: 20, Threshold: 0.6},
        RateLimit: &RateLimitConfig{Rate: 10, Period: time.Minute},
        Timeout: 3 * time.Second,
        Bulkhead: BulkheadConfig{MaxConcurrent: 30, Type: "semaphore"},
        Criticality: "MODERATE",
    },
}
```

**Critérios de aceite:**
- [ ] Composição na ordem correta: Retry → CB → RateLimiter → Timeout → Bulkhead → call
- [ ] Configuração por serviço-alvo (Payment vs Notification vs Pricing)
- [ ] Payment (crítico): retry 2x, CB agressivo, timeout 5s, bulkhead 20
- [ ] Notification (best-effort): sem retry, CB permissivo, timeout 2s, bulkhead 5
- [ ] Pricing (moderado): retry 3x, CB moderado, rate limit 10/min, timeout 3s, bulkhead 30
- [ ] Tabela de decisão documentada no DECISIONS.md
- [ ] Justificativa para cada configuração baseada em criticidade do serviço
- [ ] Testes de integração: simular falhas e verificar que a cadeia inteira funciona

---

### Desafio 2.4 — Anti-Patterns de Resiliência: O Que NÃO Fazer

Identifique e corrija anti-patterns comuns de resiliência em código legado do RideFlow.

**Cenário:** Você recebeu um código legado do RideFlow v1 com vários anti-patterns de resiliência. Refatore cada um aplicando a prática correta.

**Requisitos:**
Corrigir os seguintes anti-patterns:

1. **Retry sem backoff** — retry imediato causa thundering herd
2. **Circuit Breaker com threshold muito baixo** — 2 falhas abrem o circuito (false positive)
3. **Timeout maior que o timeout do caller** — chamada lenta bloqueia a cascata
4. **Rate limiter sem Redis** em ambiente multi-instância — cada instância tem seu próprio limite
5. **Bulkhead com maxWait infinito** — threads ficam paradas esperando permissão
6. **Retry em operação não-idempotente** — pagamento cobrado 3x
7. **Retry em erro de negócio (400)** — retentar nunca vai resolver "saldo insuficiente"

**Java 25 — Anti-pattern → Correção:**
```java
// ❌ ANTI-PATTERN: Retry sem backoff (thundering herd)
RetryConfig.custom()
    .maxAttempts(5)
    .waitDuration(Duration.ZERO) // retry imediato!
    .build();

// ✅ CORREÇÃO: Exponential backoff com jitter
RetryConfig.custom()
    .maxAttempts(3)
    .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
        Duration.ofMillis(500), 2.0, Duration.ofSeconds(5)))
    .build();

// ❌ ANTI-PATTERN: Retry em operação não-idempotente
@Retry(name = "payment")
public PaymentResult chargeCustomer(UUID customerId, BigDecimal amount) {
    return paymentGateway.charge(customerId, amount); // cobrado N vezes!
}

// ✅ CORREÇÃO: Idempotency key + retry apenas no GET do resultado
@Retry(name = "payment")
public PaymentResult chargeCustomer(UUID customerId, BigDecimal amount) {
    var idempotencyKey = UUID.randomUUID().toString();
    return paymentGateway.chargeIdempotent(customerId, amount, idempotencyKey);
}
```

**Go 1.26 — Anti-pattern → Correção:**
```go
// ❌ ANTI-PATTERN: Timeout maior que o caller
func (c *Client) Call(ctx context.Context) (*Result, error) {
    // Caller tem timeout de 5s, mas este timeout é 30s — inútil
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    return c.httpClient.Do(req.WithContext(ctx))
}

// ✅ CORREÇÃO: Timeout menor que o caller
func (c *Client) Call(ctx context.Context) (*Result, error) {
    // Caller tem timeout de 5s, este é 3s — fail-fast
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()
    return c.httpClient.Do(req.WithContext(ctx))
}

// ❌ ANTI-PATTERN: Bulkhead com espera infinita
sem := make(chan struct{}, 10)
sem <- struct{}{} // bloqueia para sempre se cheio

// ✅ CORREÇÃO: Fail-fast com select + default
select {
case sem <- struct{}{}:
    defer func() { <-sem }()
    return doWork()
default:
    return nil, ErrBulkheadFull
}
```

**Critérios de aceite:**
- [ ] 7 anti-patterns identificados, documentados e corrigidos
- [ ] Cada anti-pattern tem: código incorreto, explicação do problema, código correto
- [ ] `ANTI-PATTERNS.md` com catálogo completo e referências
- [ ] Testes que demonstram o problema (antes) e a correção (depois)
- [ ] Revisão de todo o código do Level 0-2 para garantir ausência de anti-patterns
- [ ] Checklist de review para futuras implementações de resiliência
