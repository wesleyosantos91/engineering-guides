# Level 6 — Infrastructure: Service Discovery, External Config, Health Checks, API Gateway/BFF & Sidecar

> **Objetivo:** Implementar padrões de infraestrutura essenciais — Service Discovery para localizar serviços dinamicamente,
> Configuração Externa para separar config do código, Health Checks para auto-healing, API Gateway/BFF como ponto de entrada
> unificado, e Sidecar para cross-cutting concerns desacoplados.

**Pré-requisito:** [Level 5 — Event-Driven](05-event-driven-outbox-cdc-backpressure-sharding.md)

**Referências:**
- [17-service-discovery.md](../../.docs/microservice-patterns/17-service-discovery.md)
- [18-configuracao-externa.md](../../.docs/microservice-patterns/18-configuracao-externa.md)
- [19-health-checks.md](../../.docs/microservice-patterns/19-health-checks.md)
- [20-api-gateway-bff.md](../../.docs/microservice-patterns/20-api-gateway-bff.md)
- [21-sidecar.md](../../.docs/microservice-patterns/21-sidecar.md)

---

## Contexto do Domínio

O RideFlow precisa de infraestrutura robusta para operar em produção:

- **Service Discovery** — driver-service escala para 10 instâncias em pico; ride-service precisa encontrá-las
- **External Config** — timeouts, pool sizes, Kafka brokers mudam entre dev/staging/prod sem rebuild
- **Health Checks** — instância com pool de conexões esgotado é removida do load balancer automaticamente
- **API Gateway/BFF** — app do passageiro e app do motorista precisam de APIs diferentes (BFF)
- **Sidecar** — mTLS, métricas e tracing transparentes sem código nos serviços

---

## Desafios

### Desafio 6.1 — Service Discovery: Registro Dinâmico com Consul/Eureka

Implemente service discovery para que serviços se encontrem dinamicamente sem IPs hardcoded.

**Cenário:** O driver-service escala de 2 para 10 instâncias em horário de pico. O ride-service precisa encontrar todas as instâncias disponíveis para distribuir requests de matching. IPs mudam a cada scaling/deploy.

**Requisitos:**
- **Service Registry:** Consul (infraestrutura compartilhada entre Java e Go)
- **Self-Registration:** cada instância se registra ao iniciar com:
  - Service name, host, port, health check URL
  - Metadata: version, region, zone
- **Self-Deregistration:** ao receber SIGTERM, desregistrar antes de encerrar
- **Client-Side Discovery:**
  - Consultar registry para obter lista de instâncias saudáveis
  - Load balancing round-robin no client
  - Cache local com TTL de 30s (reduzir calls ao registry)
- **Health Check integration:** Consul chama `/health/ready` a cada 10s; instância unhealthy é removida
- Fallback: se registry indisponível, usar cache local da última lista conhecida

**Java 25 (Spring Cloud Consul):**
```java
// application.yml
spring:
  cloud:
    consul:
      host: consul
      port: 8500
      discovery:
        service-name: ride-service
        health-check-path: /health/ready
        health-check-interval: 10s
        instance-id: ${spring.application.name}:${random.value}
        metadata:
          version: "2.1.0"
          region: "sa-east-1"
        deregister: true

// Client-Side Discovery com load balancing
@Service
public class DriverServiceClient {
    private final DiscoveryClient discoveryClient;
    private final WebClient.Builder webClientBuilder;
    
    // Cache local de instâncias
    private volatile List<ServiceInstance> cachedInstances = List.of();
    private final AtomicInteger roundRobinIndex = new AtomicInteger(0);
    
    @Scheduled(fixedRate = 30_000) // refresh a cada 30s
    public void refreshInstances() {
        try {
            var instances = discoveryClient.getInstances("driver-service");
            if (!instances.isEmpty()) {
                cachedInstances = instances;
            }
        } catch (Exception e) {
            log.warn("Registry unavailable, using cached instances ({})", 
                cachedInstances.size());
        }
    }
    
    public DriverDTO getDriver(UUID driverId) {
        var instance = nextInstance();
        var url = instance.getUri() + "/api/drivers/" + driverId;
        
        return webClientBuilder.build()
            .get().uri(url)
            .retrieve()
            .bodyToMono(DriverDTO.class)
            .timeout(Duration.ofSeconds(3))
            .block();
    }
    
    private ServiceInstance nextInstance() {
        var instances = cachedInstances;
        if (instances.isEmpty()) {
            throw new ServiceUnavailableException("No driver-service instances available");
        }
        int idx = roundRobinIndex.getAndIncrement() % instances.size();
        return instances.get(idx);
    }
}

// Graceful shutdown: deregister
@PreDestroy
public void onShutdown() {
    log.info("Deregistering from Consul...");
    consulClient.agentServiceDeregister(serviceId);
}
```

**Go 1.26 (HashiCorp Consul client):**
```go
type ServiceRegistry struct {
    client *consul.Client
    serviceID string
}

func (r *ServiceRegistry) Register(name string, port int) error {
    r.serviceID = fmt.Sprintf("%s-%s", name, uuid.New().String()[:8])
    
    return r.client.Agent().ServiceRegister(&consul.AgentServiceRegistration{
        ID:   r.serviceID,
        Name: name,
        Port: port,
        Tags: []string{"v2.1.0", "sa-east-1"},
        Meta: map[string]string{"version": "2.1.0", "region": "sa-east-1"},
        Check: &consul.AgentServiceCheck{
            HTTP:                           fmt.Sprintf("http://localhost:%d/health/ready", port),
            Interval:                       "10s",
            Timeout:                        "3s",
            DeregisterCriticalServiceAfter: "30s",
        },
    })
}

func (r *ServiceRegistry) Deregister() error {
    return r.client.Agent().ServiceDeregister(r.serviceID)
}

// Client-Side Discovery
type DriverServiceClient struct {
    consul       *consul.Client
    cache        []ServiceEntry
    cacheMu      sync.RWMutex
    rrIndex      atomic.Int64
}

func (c *DriverServiceClient) RefreshLoop(ctx context.Context) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    c.refresh() // initial
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            c.refresh()
        }
    }
}

func (c *DriverServiceClient) refresh() {
    entries, _, err := c.consul.Health().Service("driver-service", "", true, nil)
    if err != nil {
        slog.Warn("registry unavailable, using cached entries", "cached", len(c.cache))
        return
    }
    c.cacheMu.Lock()
    c.cache = entries
    c.cacheMu.Unlock()
}

func (c *DriverServiceClient) GetDriver(ctx context.Context, driverID uuid.UUID) (*Driver, error) {
    entry := c.nextInstance()
    url := fmt.Sprintf("http://%s:%d/api/drivers/%s",
        entry.Service.Address, entry.Service.Port, driverID)
    
    req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("get driver: %w", err)
    }
    defer resp.Body.Close()
    
    var driver Driver
    json.NewDecoder(resp.Body).Decode(&driver)
    return &driver, nil
}

func (c *DriverServiceClient) nextInstance() *consul.ServiceEntry {
    c.cacheMu.RLock()
    defer c.cacheMu.RUnlock()
    if len(c.cache) == 0 {
        panic("no driver-service instances available")
    }
    idx := c.rrIndex.Add(1) % int64(len(c.cache))
    return c.cache[idx]
}
```

**Critérios de aceite:**
- [ ] Self-registration no Consul ao iniciar (com metadata version/region)
- [ ] Self-deregistration ao receber SIGTERM (graceful shutdown)
- [ ] Health check a cada 10s pelo Consul (`/health/ready`)
- [ ] Client-side discovery com round-robin load balancing
- [ ] Cache local de instâncias (TTL 30s) — reduz calls ao registry
- [ ] Fallback: registry down → usa cache; cache vazio → erro claro
- [ ] Testes: 3 instâncias registradas → requests distribuídos; 1 unhealthy → removida

---

### Desafio 6.2 — External Config: Configuração Centralizada Multi-Ambiente

Implemente configuração externalizada com hierarquia de fontes e refresh dinâmico.

**Cenário:** O RideFlow roda em dev, staging e prod. Cada ambiente tem configurações diferentes (database URLs, timeouts, pool sizes, Kafka brokers). O mesmo artefato Docker deve rodar em qualquer ambiente — apenas a config muda.

**Requisitos:**
- **Hierarquia de configuração** (prioridade crescente):
  1. Defaults no código (fallback)
  2. Arquivo base (`application.yml` / `config.yaml`)
  3. Arquivo por ambiente (`application-{env}.yml`)
  4. Variáveis de ambiente (12-Factor App)
  5. Config server / Consul KV (centralizado)
- **Categorias de config:**
  - Infra: `DB_URL`, `KAFKA_BROKERS`, `REDIS_HOST`
  - Comportamento: `RIDE_MATCHING_TIMEOUT_MS=5000`, `PAYMENT_RETRY_MAX=3`
  - Secrets: `DB_PASSWORD`, `JWT_SECRET` (via vault/secrets manager)
- **Refresh dinâmico:** alterar `RIDE_MATCHING_TIMEOUT_MS` no Consul KV → serviço aplica em até 30s sem restart
- **Validação:** ao iniciar, validar que todas as configs obrigatórias estão presentes; falhar fast se ausente
- **Secrets:** nunca loggar valores de secrets; mascarar em endpoints de diagnóstico

**Java 25 (Spring Boot + Spring Cloud Config + Consul KV):**
```java
// application.yml (defaults)
ride:
  matching:
    timeout-ms: 5000
    max-radius-km: 10
  payment:
    retry-max: 3
    retry-backoff-ms: 500

// application-prod.yml (override por ambiente)
ride:
  matching:
    timeout-ms: 3000  # prod é mais agressivo

// Configuration class com validação
@Configuration
@ConfigurationProperties(prefix = "ride.matching")
@Validated
public class MatchingConfig {
    @NotNull @Min(1000) @Max(30000)
    private Integer timeoutMs;
    
    @NotNull @Min(1) @Max(50)
    private Integer maxRadiusKm;
    
    // Refresh dinâmico via @RefreshScope
}

@RefreshScope // Atualiza quando /actuator/refresh é chamado
@Service
public class RideMatchingService {
    private final MatchingConfig config;
    
    public Driver findNearestDriver(GeoPoint origin) {
        return driverClient.findNearby(origin, config.getMaxRadiusKm())
            .timeout(Duration.ofMillis(config.getTimeoutMs()))
            .blockFirst();
    }
}

// Validação na inicialização
@Component
public class ConfigValidator implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        var required = List.of("DB_URL", "KAFKA_BROKERS", "JWT_SECRET");
        var missing = required.stream()
            .filter(key -> System.getenv(key) == null)
            .toList();
        if (!missing.isEmpty()) {
            throw new IllegalStateException(
                "Missing required config: " + String.join(", ", missing));
        }
    }
}
```

**Go 1.26 (Viper + Consul KV):**
```go
type AppConfig struct {
    DB       DatabaseConfig  `mapstructure:"db"`
    Kafka    KafkaConfig     `mapstructure:"kafka"`
    Matching MatchingConfig  `mapstructure:"matching"`
    Payment  PaymentConfig   `mapstructure:"payment"`
}

type MatchingConfig struct {
    TimeoutMs  int `mapstructure:"timeout_ms" validate:"required,min=1000,max=30000"`
    MaxRadiusKm int `mapstructure:"max_radius_km" validate:"required,min=1,max=50"`
}

func LoadConfig(env string) (*AppConfig, error) {
    v := viper.New()
    
    // 1. Defaults
    v.SetDefault("matching.timeout_ms", 5000)
    v.SetDefault("matching.max_radius_km", 10)
    
    // 2. Arquivo base
    v.SetConfigName("config")
    v.SetConfigType("yaml")
    v.AddConfigPath("./config")
    v.ReadInConfig()
    
    // 3. Arquivo por ambiente
    v.SetConfigName("config-" + env) // config-prod.yaml
    v.MergeInConfig()
    
    // 4. Variáveis de ambiente
    v.AutomaticEnv()
    v.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
    
    var cfg AppConfig
    if err := v.Unmarshal(&cfg); err != nil {
        return nil, fmt.Errorf("unmarshal config: %w", err)
    }
    
    // Validação
    validate := validator.New()
    if err := validate.Struct(cfg); err != nil {
        return nil, fmt.Errorf("config validation: %w", err)
    }
    
    return &cfg, nil
}

// Refresh dinâmico via Consul KV watch
func (c *ConfigWatcher) WatchConsulKV(ctx context.Context, key string) {
    var lastIndex uint64
    for {
        select {
        case <-ctx.Done():
            return
        default:
            pair, meta, err := c.consul.KV().Get(key, &consul.QueryOptions{
                WaitIndex: lastIndex,
                WaitTime:  30 * time.Second, // long poll
            })
            if err != nil {
                slog.Error("consul watch error", "error", err)
                time.Sleep(5 * time.Second)
                continue
            }
            if meta.LastIndex != lastIndex {
                lastIndex = meta.LastIndex
                var newConfig MatchingConfig
                json.Unmarshal(pair.Value, &newConfig)
                c.applyConfig(newConfig)
                slog.Info("config refreshed", "key", key, "timeout_ms", newConfig.TimeoutMs)
            }
        }
    }
}
```

**Critérios de aceite:**
- [ ] Hierarquia: defaults → arquivo base → arquivo env → env vars → config server
- [ ] Mesmo artefato Docker funciona em dev e prod (apenas config muda)
- [ ] Validação na inicialização: config obrigatória ausente → fail fast com mensagem clara
- [ ] Refresh dinâmico: alterar config no Consul → serviço aplica em ≤30s sem restart
- [ ] Secrets nunca aparecem em logs ou endpoints de diagnóstico
- [ ] Testes: default → 5000ms; env var override → 3000ms; Consul KV refresh → novo valor aplicado

---

### Desafio 6.3 — Health Checks: Liveness, Readiness & Startup Probes

Implemente health checks que permitem auto-healing e zero-downtime deploys.

**Cenário:** O Kubernetes (ou Docker Compose healthcheck) precisa saber: (1) o serviço está vivo? (2) está pronto para tráfego? (3) já terminou inicialização?

**Requisitos:**
- **3 endpoints:**
  - `GET /health/live` — liveness: processo responsivo? Responder HTTP 200 se sim, else timeout/5xx → restart
  - `GET /health/ready` — readiness: banco, cache, Kafka acessíveis? Responder 200 com detalhes; 503 se algum DOWN → remove do LB
  - `GET /health/startup` — startup: migrations rodaram, cache aquecido? 200 → pronto; 503 → aguardando
- **Formato de resposta:**
  ```json
  {
    "status": "UP",
    "checks": {
      "database": { "status": "UP", "responseTimeMs": 12 },
      "kafka": { "status": "UP", "responseTimeMs": 5 },
      "redis": { "status": "DOWN", "error": "Connection refused" }
    }
  }
  ```
- **Regras:**
  - Liveness: NÃO verificar dependências (para evitar restart cascata)
  - Readiness: verificar dependências, mas com timeout curto (2s)
  - Startup: verificar uma vez, mudar para UP quando pronto, nunca voltar para DOWN
- **Self-healing:** Readiness DOWN → load balancer para de enviar tráfego → serviço se recupera → Readiness UP → tráfego volta

**Java 25 (Spring Boot Actuator):**
```java
// application.yml
management:
  endpoint:
    health:
      show-details: always
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState, db, redis, kafka
        startup:
          include: startupState
  endpoints:
    web:
      exposure:
        include: health

// Custom Health Indicator
@Component
public class KafkaHealthIndicator implements HealthIndicator {
    private final KafkaTemplate<String, String> kafkaTemplate;
    
    @Override
    public Health health() {
        try {
            long start = System.currentTimeMillis();
            kafkaTemplate.partitionsFor("health-check-topic"); // lightweight check
            long elapsed = System.currentTimeMillis() - start;
            return Health.up()
                .withDetail("responseTimeMs", elapsed)
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

// Readiness gate: bloqueia até warm-up completar
@Component
public class WarmUpHealthIndicator implements HealthIndicator {
    private volatile boolean warmedUp = false;
    
    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        // Executar warm-up (carregar cache, etc.)
        cacheService.warmUp();
        warmedUp = true;
    }
    
    @Override
    public Health health() {
        return warmedUp ? Health.up().build() : 
            Health.down().withDetail("reason", "warm-up in progress").build();
    }
}
```

**Go 1.26 (Health endpoints manuais):**
```go
type HealthChecker struct {
    db     *sqlx.DB
    redis  *redis.Client
    kafka  sarama.Client
    ready  atomic.Bool
}

type HealthResponse struct {
    Status string                     `json:"status"`
    Checks map[string]ComponentHealth `json:"checks,omitempty"`
}

type ComponentHealth struct {
    Status         string `json:"status"`
    ResponseTimeMs int64  `json:"responseTimeMs,omitempty"`
    Error          string `json:"error,omitempty"`
}

// Liveness: apenas "estou vivo?"
func (h *HealthChecker) LivenessHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(HealthResponse{Status: "UP"})
}

// Readiness: verificar dependências com timeout
func (h *HealthChecker) ReadinessHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()
    
    checks := make(map[string]ComponentHealth)
    overallStatus := "UP"
    
    // Database
    start := time.Now()
    err := h.db.PingContext(ctx)
    checks["database"] = ComponentHealth{
        Status:         boolToStatus(err == nil),
        ResponseTimeMs: time.Since(start).Milliseconds(),
        Error:          errString(err),
    }
    if err != nil {
        overallStatus = "DOWN"
    }
    
    // Redis
    start = time.Now()
    _, err = h.redis.Ping(ctx).Result()
    checks["redis"] = ComponentHealth{
        Status:         boolToStatus(err == nil),
        ResponseTimeMs: time.Since(start).Milliseconds(),
        Error:          errString(err),
    }
    if err != nil {
        overallStatus = "DOWN"
    }
    
    // Kafka
    start = time.Now()
    _, err = h.kafka.Topics()
    checks["kafka"] = ComponentHealth{
        Status:         boolToStatus(err == nil),
        ResponseTimeMs: time.Since(start).Milliseconds(),
        Error:          errString(err),
    }
    if err != nil {
        overallStatus = "DOWN"
    }
    
    w.Header().Set("Content-Type", "application/json")
    if overallStatus == "DOWN" {
        w.WriteHeader(http.StatusServiceUnavailable)
    }
    json.NewEncoder(w).Encode(HealthResponse{Status: overallStatus, Checks: checks})
}

// Startup: marca pronto após warm-up
func (h *HealthChecker) StartupHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    if h.ready.Load() {
        json.NewEncoder(w).Encode(HealthResponse{Status: "UP"})
    } else {
        w.WriteHeader(http.StatusServiceUnavailable)
        json.NewEncoder(w).Encode(HealthResponse{Status: "DOWN"})
    }
}

func (h *HealthChecker) MarkReady() {
    h.ready.Store(true)
}
```

**Docker Compose healthcheck:**
```yaml
services:
  ride-service:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/health/ready"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 30s
```

**Critérios de aceite:**
- [ ] 3 endpoints: `/health/live`, `/health/ready`, `/health/startup`
- [ ] Liveness: NÃO verifica dependências (evita restart cascata)
- [ ] Readiness: verifica DB + Redis + Kafka com timeout 2s; retorna 503 se algum DOWN
- [ ] Startup: retorna 503 até warm-up completar; 200 após
- [ ] Formato JSON com status, checks, responseTimeMs, error
- [ ] Docker Compose healthcheck configurado com interval/timeout/retries/start_period
- [ ] Testes: DB down → readiness 503 → tráfego parado; DB volta → readiness 200 → tráfego retomado

---

### Desafio 6.4 — API Gateway & BFF: Ponto de Entrada Unificado

Implemente um API Gateway com BFFs dedicados para passageiro e motorista.

**Cenário:** O app do passageiro e o app do motorista precisam de APIs diferentes. O passageiro quer detalhes da corrida com preço e ETA; o motorista quer corridas disponíveis na sua região e earnings. Um gateway genérico não serve ambos eficientemente.

**Requisitos:**
- **API Gateway** (Spring Cloud Gateway / Go reverse proxy):
  - Roteamento por path: `/passenger/**` → BFF Passenger, `/driver/**` → BFF Driver
  - Cross-cutting concerns centralizados: JWT validation, rate limiting, CORS, request logging, traceId propagation
  - TLS termination
- **BFF Passenger** — otimizado para app do passageiro:
  - `GET /passenger/rides/{id}` → agrega Ride + Driver + Price + ETA (API Composition)
  - `POST /passenger/rides` → criar corrida
  - Campos: dados do motorista simplificados (nome, foto, rating, placa)
- **BFF Driver** — otimizado para app do motorista:
  - `GET /driver/available-rides` → corridas REQUESTED na região do motorista
  - `POST /driver/rides/{id}/accept` → aceitar corrida
  - `GET /driver/earnings?period=today` → resumo de ganhos
  - Campos: dados de ganhos, métricas de performance
- Rate limiting no gateway: passageiro 60 req/min, motorista 120 req/min

**Java 25 (Spring Cloud Gateway + BFF):**
```java
// API Gateway — application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: passenger-bff
          uri: lb://passenger-bff
          predicates:
            - Path=/passenger/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 1
                redis-rate-limiter.burstCapacity: 60
        - id: driver-bff
          uri: lb://driver-bff
          predicates:
            - Path=/driver/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 2
                redis-rate-limiter.burstCapacity: 120
      default-filters:
        - name: AddRequestHeader
          args:
            name: X-Trace-Id
            value: "#{T(java.util.UUID).randomUUID().toString()}"

// JWT Validation Filter (Gateway)
@Component
public class JwtAuthFilter implements GatewayFilter {
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (token == null || !token.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        try {
            var claims = jwtVerifier.verify(token.substring(7));
            exchange.getRequest().mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-User-Role", claims.get("role", String.class));
            return chain.filter(exchange);
        } catch (JwtException e) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
    }
}

// BFF Passenger Controller
@RestController
@RequestMapping("/rides")
public class PassengerBffController {
    
    @GetMapping("/{rideId}")
    public RideDetailsForPassenger getRideDetails(@PathVariable UUID rideId) {
        var ride = rideClient.getRide(rideId);
        var driver = driverClient.getDriver(ride.driverId());
        var eta = locationClient.getETA(ride.driverId(), ride.origin());
        
        return new RideDetailsForPassenger(
            ride.id(), ride.status(),
            ride.origin(), ride.destination(),
            ride.price(),
            new DriverSummary(driver.name(), driver.photo(), driver.rating(), driver.plate()),
            eta.minutes()
        );
    }
}
```

**Go 1.26 (Reverse proxy + Chi BFF):**
```go
// API Gateway (reverse proxy)
func NewGateway(passengerBFF, driverBFF string) http.Handler {
    r := chi.NewRouter()
    
    // Cross-cutting middleware
    r.Use(middleware.Logger)
    r.Use(middleware.Recoverer)
    r.Use(corsMiddleware)
    r.Use(traceIDMiddleware)
    r.Use(jwtAuthMiddleware)
    
    // Route to BFFs
    passengerURL, _ := url.Parse(passengerBFF)
    driverURL, _ := url.Parse(driverBFF)
    
    r.Route("/passenger", func(r chi.Router) {
        r.Use(rateLimitMiddleware(60, time.Minute)) // 60 req/min
        r.Handle("/*", httputil.NewSingleHostReverseProxy(passengerURL))
    })
    
    r.Route("/driver", func(r chi.Router) {
        r.Use(rateLimitMiddleware(120, time.Minute)) // 120 req/min
        r.Handle("/*", httputil.NewSingleHostReverseProxy(driverURL))
    })
    
    return r
}

func jwtAuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if !strings.HasPrefix(token, "Bearer ") {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        claims, err := verifyJWT(token[7:])
        if err != nil {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }
        r.Header.Set("X-User-Id", claims.Subject)
        r.Header.Set("X-User-Role", claims.Role)
        next.ServeHTTP(w, r)
    })
}

// BFF Passenger
func NewPassengerBFF() chi.Router {
    r := chi.NewRouter()
    
    r.Get("/rides/{rideID}", func(w http.ResponseWriter, r *http.Request) {
        rideID := chi.URLParam(r, "rideID")
        
        g, ctx := errgroup.WithContext(r.Context())
        var ride Ride
        var driver Driver
        var eta ETAResponse
        
        g.Go(func() error {
            var err error
            ride, err = rideClient.GetRide(ctx, rideID)
            return err
        })
        // driver e eta dependem de ride, então sequencial após ride
        
        if err := g.Wait(); err != nil {
            http.Error(w, "ride not found", http.StatusNotFound)
            return
        }
        
        g2, ctx2 := errgroup.WithContext(ctx)
        g2.Go(func() error {
            var err error
            driver, err = driverClient.GetDriver(ctx2, *ride.DriverID)
            return err
        })
        g2.Go(func() error {
            var err error
            eta, err = locationClient.GetETA(ctx2, *ride.DriverID, ride.Origin)
            return err
        })
        g2.Wait() // graceful degradation se falhar
        
        json.NewEncoder(w).Encode(RideDetailsForPassenger{
            RideID:      ride.ID,
            Status:      ride.Status,
            Origin:      ride.Origin,
            Destination: ride.Destination,
            Price:       ride.Price,
            Driver:      DriverSummary{Name: driver.Name, Photo: driver.Photo, Rating: driver.Rating, Plate: driver.Plate},
            ETAMinutes:  eta.Minutes,
        })
    })
    
    return r
}
```

**Critérios de aceite:**
- [ ] Gateway roteia `/passenger/**` → BFF Passenger, `/driver/**` → BFF Driver
- [ ] JWT validado no gateway (não nos BFFs) — `X-User-Id` propagado como header
- [ ] Rate limiting: passageiro 60/min, motorista 120/min (HTTP 429)
- [ ] TraceId gerado no gateway e propagado para downstream
- [ ] BFF Passenger: endpoint otimizado com dados agregados para passageiro
- [ ] BFF Driver: endpoint otimizado com dados específicos para motorista
- [ ] CORS configurado no gateway
- [ ] Testes: request sem JWT → 401; com JWT → routed + X-User-Id propagado; rate limit exceeded → 429

---

### Desafio 6.5 — Sidecar: Cross-Cutting Concerns Desacoplados

Implemente o Sidecar Pattern usando Envoy como proxy para mTLS, métricas e retry automático.

**Cenário:** Cada serviço do RideFlow precisa de mTLS, métricas de tráfego e retry automático. Em vez de implementar isso em cada serviço (Java E Go), usar um sidecar Envoy que intercepta todo o tráfego.

**Requisitos:**
- **Envoy sidecar** configurado via Docker Compose (1 Envoy por serviço)
- Serviço fala com `localhost:envoy_port`; Envoy roteia para o serviço destino
- **mTLS** entre sidecars: tráfego serviço→serviço sempre criptografado
- **Métricas** coletadas pelo sidecar: request count, latency histogram, error rate (sem código no serviço)
- **Retry automático** no sidecar: 2 retries com backoff para 5xx (sem código no serviço)
- Serviço principal **não sabe** que existe sidecar — transparente

**Docker Compose:**
```yaml
services:
  ride-service:
    build: ./ride-service
    # Serviço não expõe porta diretamente — tráfego vai pelo sidecar
    
  ride-service-sidecar:
    image: envoyproxy/envoy:v1.31
    volumes:
      - ./envoy/ride-service.yaml:/etc/envoy/envoy.yaml
      - ./certs:/etc/certs
    ports:
      - "8081:8081"   # inbound (clientes chamam aqui)
    network_mode: "service:ride-service"  # compartilha network namespace

  driver-service:
    build: ./driver-service
    
  driver-service-sidecar:
    image: envoyproxy/envoy:v1.31
    volumes:
      - ./envoy/driver-service.yaml:/etc/envoy/envoy.yaml
      - ./certs:/etc/certs
    ports:
      - "8082:8082"
    network_mode: "service:driver-service"
```

**Envoy config (`ride-service.yaml`):**
```yaml
static_resources:
  listeners:
    - name: inbound
      address:
        socket_address: { address: 0.0.0.0, port_value: 8081 }
      filter_chains:
        - transport_socket:
            name: envoy.transport_sockets.tls
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
              common_tls_context:
                tls_certificates:
                  - certificate_chain: { filename: /etc/certs/ride-service.crt }
                    private_key: { filename: /etc/certs/ride-service.key }
                validation_context:
                  trusted_ca: { filename: /etc/certs/ca.crt }
                  # mTLS: exigir certificado do client
          filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                route_config:
                  virtual_hosts:
                    - name: local_service
                      routes:
                        - match: { prefix: "/" }
                          route: { cluster: local_app }
  clusters:
    - name: local_app
      load_assignment:
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address: { address: 127.0.0.1, port_value: 8080 }  # app real
    - name: driver-service
      load_assignment:
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address: { address: driver-service-sidecar, port_value: 8082 }
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          common_tls_context:
            tls_certificates:
              - certificate_chain: { filename: /etc/certs/ride-service.crt }
                private_key: { filename: /etc/certs/ride-service.key }
      # Retry policy
      retry_policy:
        retry_on: "5xx"
        num_retries: 2
        retry_back_off:
          base_interval: 0.5s
          max_interval: 5s
```

**Critérios de aceite:**
- [ ] Envoy sidecar para cada serviço (ride, driver, payment)
- [ ] mTLS entre sidecars (tráfego criptografado inter-service)
- [ ] Métricas de tráfego coletadas pelo sidecar (request count, latency, errors) via `/stats/prometheus`
- [ ] Retry automático 2x com backoff para 5xx (sem código no serviço)
- [ ] Serviço principal não tem código de retry/mTLS — totalmente transparente
- [ ] Testes: request ride→driver via sidecars → mTLS verificado nos logs; driver retorna 500 → sidecar retenta automaticamente
