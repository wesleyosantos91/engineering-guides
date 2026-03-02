# Level 7 — Deployment: Strangler Fig, Blue-Green, Canary, Feature Flags & Shadow Traffic

> **Objetivo:** Dominar estratégias de deploy e modernização — migrar legado incrementalmente com Strangler Fig,
> fazer deploys com zero downtime via Blue-Green, releases progressivas com Canary, controlar features
> com Feature Flags, e validar com Shadow Traffic.

**Pré-requisito:** [Level 6 — Infrastructure](06-infrastructure-discovery-config-health-gateway-sidecar.md)

**Referências:**
- [22-strangler-fig.md](../../.docs/microservice-patterns/22-strangler-fig.md)
- [23-blue-green.md](../../.docs/microservice-patterns/23-blue-green.md)
- [24-canary-release.md](../../.docs/microservice-patterns/24-canary-release.md)
- [25-feature-flags.md](../../.docs/microservice-patterns/25-feature-flags.md)
- [26-shadow-traffic.md](../../.docs/microservice-patterns/26-shadow-traffic.md)

---

## Contexto do Domínio

O RideFlow opera em produção com milhares de corridas ativas. Deploys precisam ser seguros:

- **Strangler Fig** — Migrar o módulo de pricing de um monolito legado para o novo pricing-service
- **Blue-Green** — Deploy do ride-service v2 com zero downtime e rollback instantâneo
- **Canary** — Novo algoritmo de matching enviado para 5% → 25% → 100% dos requests
- **Feature Flags** — Ativar "surge pricing" apenas em SP, apenas para 10% dos passageiros
- **Shadow Traffic** — Testar novo payment gateway replicando tráfego real sem impactar usuários

---

## Desafios

### Desafio 7.1 — Strangler Fig: Migrar Pricing do Monolito

Implemente migração incremental do módulo de pricing do monolito para o novo pricing-service.

**Cenário:** O RideFlow começou como monolito. O módulo de pricing (cálculo de tarifa, surge pricing, promoções) precisa ser extraído para um microsserviço. A migração deve ser gradual — funcionalidade por funcionalidade — sem interromper o sistema.

**Requisitos:**
- **Fase 1 — Proxy:** API Gateway intercepta todas as chamadas `/pricing/**`:
  - Inicialmente 100% → monolito (transparente)
  - Gateway como façade: clientes nunca sabem de qual backend vem a resposta
- **Fase 2 — Migração incremental:**
  - `POST /pricing/estimate` → migrado para pricing-service (novo)
  - `POST /pricing/surge-rules` → ainda no monolito
  - `POST /pricing/promotions` → ainda no monolito
  - Routing por feature flag: `pricing.estimate.useNewService=true`
- **Fase 3 — Completar:**
  - Todas as funcionalidades migradas → desligar módulo no monolito
- **Verificação:** comparar respostas do monolito e novo serviço (shadow mode) antes de switchover
- **Rollback:** se erro rate > 5%, voltar para monolito automaticamente
- **Data sync:** CDC do banco do monolito para o pricing-service (pricing rules, promotions)

**Java 25 (Gateway routing):**
```java
// Gateway — Dynamic routing com Feature Flag
@Component
public class StranglerFigRouter implements RouteLocator {
    private final FeatureFlagService flags;
    
    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()
            // Pricing estimate: routed baseado em feature flag
            .route("pricing-estimate", r -> r
                .path("/pricing/estimate")
                .and()
                .predicate(exchange -> flags.isEnabled("pricing.estimate.useNewService"))
                .uri("lb://pricing-service"))
            
            // Fallback: pricing estimate para monolito
            .route("pricing-estimate-legacy", r -> r
                .path("/pricing/estimate")
                .uri("http://monolith:8090"))
            
            // Ainda no monolito
            .route("pricing-legacy", r -> r
                .path("/pricing/surge-rules", "/pricing/promotions")
                .uri("http://monolith:8090"))
            .build();
    }
}

// Shadow comparison: validar antes do switchover
@Service
public class PricingShadowComparator {
    
    public void compareResponses(PricingRequest request) {
        var legacyF = CompletableFuture.supplyAsync(
            () -> monolithClient.estimate(request));
        var newF = CompletableFuture.supplyAsync(
            () -> pricingServiceClient.estimate(request));
        
        try {
            var legacy = legacyF.get(5, TimeUnit.SECONDS);
            var newResult = newF.get(5, TimeUnit.SECONDS);
            
            if (!legacy.price().equals(newResult.price())) {
                log.warn("Pricing mismatch! legacy={}, new={}, request={}",
                    legacy.price(), newResult.price(), request);
                mismatchCounter.increment();
            }
        } catch (Exception e) {
            log.error("Shadow comparison failed", e);
        }
    }
}

// Automatic rollback: error rate > 5%
@Component
public class StranglerFigHealthMonitor {
    
    @Scheduled(fixedRate = 30_000) // a cada 30s
    public void checkNewServiceHealth() {
        double errorRate = meterRegistry.get("pricing.new.errors")
            .counter().count() / 
            meterRegistry.get("pricing.new.total")
            .counter().count();
        
        if (errorRate > 0.05) {
            log.error("Error rate {}% > 5%, rolling back to monolith", errorRate * 100);
            featureFlagService.disable("pricing.estimate.useNewService");
        }
    }
}
```

**Go 1.26 (Reverse proxy routing):**
```go
func NewStranglerFigProxy(monolithURL, pricingServiceURL string, flags *FeatureFlagService) http.Handler {
    monolith, _ := url.Parse(monolithURL)
    pricing, _ := url.Parse(pricingServiceURL)
    
    monolithProxy := httputil.NewSingleHostReverseProxy(monolith)
    pricingProxy := httputil.NewSingleHostReverseProxy(pricing)
    
    r := chi.NewRouter()
    
    r.Post("/pricing/estimate", func(w http.ResponseWriter, r *http.Request) {
        if flags.IsEnabled("pricing.estimate.useNewService") {
            pricingProxy.ServeHTTP(w, r)
        } else {
            monolithProxy.ServeHTTP(w, r)
        }
    })
    
    // Ainda no monolito
    r.Handle("/pricing/surge-rules", monolithProxy)
    r.Handle("/pricing/promotions", monolithProxy)
    
    return r
}

// Shadow comparison
func (c *ShadowComparator) Compare(ctx context.Context, req PricingRequest) {
    g, ctx := errgroup.WithContext(ctx)
    var legacy, newResult PricingResponse
    
    g.Go(func() error {
        var err error
        legacy, err = c.monolithClient.Estimate(ctx, req)
        return err
    })
    g.Go(func() error {
        var err error
        newResult, err = c.pricingClient.Estimate(ctx, req)
        return err
    })
    
    if err := g.Wait(); err != nil {
        slog.Error("shadow comparison failed", "error", err)
        return
    }
    
    if legacy.Price != newResult.Price {
        slog.Warn("pricing mismatch",
            "legacy", legacy.Price, "new", newResult.Price)
        c.metrics.MismatchCount.Inc()
    }
}
```

**Critérios de aceite:**
- [ ] Gateway como façade: clientes não sabem se resposta vem do monolito ou novo serviço
- [ ] Feature flag controla routing (`pricing.estimate.useNewService`)
- [ ] Shadow comparison: respostas monolito vs novo comparadas; mismatches logados
- [ ] Rollback automático: error rate > 5% → flag desabilitada → tráfego volta ao monolito
- [ ] Migração incremental: estimate migrado ✓; surge-rules e promotions ainda no monolito
- [ ] CDC sincroniza dados do monolito → pricing-service
- [ ] Testes: flag on → novo serviço responde; flag off → monolito responde; erro → rollback funciona

---

### Desafio 7.2 — Blue-Green Deploy: Zero Downtime para Ride Service

Implemente deploy blue-green para o ride-service com switch instantâneo e rollback.

**Cenário:** O ride-service v2 tem novo algoritmo de state machine. O deploy deve ter zero downtime e rollback instantâneo em caso de problema.

**Requisitos:**
- **Dois ambientes idênticos:** Blue (v1, ativo) e Green (v2, standby)
- **Deploy flow:**
  1. Deploy v2 no ambiente Green
  2. Rodar smoke tests contra Green (health checks, endpoints principais)
  3. Validar Green com tráfego sintético (requests de teste)
  4. **Switch:** load balancer aponta para Green (instantâneo)
  5. Blue mantido como standby por 1h (rollback window)
  6. **Rollback:** se necessário, switch LB de volta para Blue (< 1 minuto)
- **Database migration:** backward-compatible (v1 e v2 funcionam com o mesmo schema)
- **Docker Compose** com perfis blue/green ou labels
- **Health check gate:** Green só recebe tráfego se `/health/ready` retornar 200

**Docker Compose:**
```yaml
services:
  # Blue (v1) - currently active
  ride-service-blue:
    image: rideflow/ride-service:v1.0.0
    environment:
      - DEPLOYMENT_COLOR=blue
    deploy:
      labels:
        traefik.enable: "true"
        traefik.http.routers.ride.rule: "PathPrefix(`/api/rides`)"
        traefik.http.services.ride.loadbalancer.server.port: "8081"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/health/ready"]
      interval: 10s
  
  # Green (v2) - standby, recebe tráfego após switch
  ride-service-green:
    image: rideflow/ride-service:v2.0.0
    environment:
      - DEPLOYMENT_COLOR=green
    deploy:
      labels:
        traefik.enable: "false"  # inativo até switch
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/health/ready"]
      interval: 10s
  
  traefik:
    image: traefik:v3.0
    command:
      - "--providers.docker=true"
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

**Java 25 (Script de deploy):**
```java
// DeploymentController — endpoint para switch blue↔green
@RestController
@RequestMapping("/admin/deploy")
public class DeploymentController {
    
    @PostMapping("/switch")
    @PreAuthorize("hasRole('ADMIN')")
    public DeploymentStatus switchEnvironment() {
        var current = deploymentService.getActiveEnvironment(); // "blue" ou "green"
        var target = current.equals("blue") ? "green" : "blue";
        
        // 1. Validar que target está saudável
        if (!healthChecker.isHealthy(target)) {
            throw new DeploymentException(target + " is not healthy, aborting switch");
        }
        
        // 2. Smoke test no target
        var smokeResult = smokeTestRunner.run(target);
        if (!smokeResult.passed()) {
            throw new DeploymentException("Smoke tests failed: " + smokeResult.failures());
        }
        
        // 3. Switch LB
        loadBalancer.routeTrafficTo(target);
        
        // 4. Log + métricas
        log.info("Switched traffic from {} to {}", current, target);
        deployEvents.publish(new DeploymentSwitchEvent(current, target, Instant.now()));
        
        return new DeploymentStatus(target, "active", Instant.now());
    }
    
    @PostMapping("/rollback")
    @PreAuthorize("hasRole('ADMIN')")
    public DeploymentStatus rollback() {
        var current = deploymentService.getActiveEnvironment();
        var previous = current.equals("blue") ? "green" : "blue";
        
        loadBalancer.routeTrafficTo(previous);
        log.warn("ROLLBACK: switched traffic from {} to {}", current, previous);
        
        return new DeploymentStatus(previous, "rollback", Instant.now());
    }
}
```

**Critérios de aceite:**
- [ ] Dois ambientes (blue/green) com versões diferentes rodando simultaneamente
- [ ] Health check gate: Green só recebe tráfego se `/health/ready` = 200
- [ ] Smoke tests rodam antes do switch
- [ ] Switch instantâneo via load balancer (Traefik labels ou API)
- [ ] Rollback em < 1 minuto (switch LB de volta)
- [ ] Database migration backward-compatible (ambas versões funcionam)
- [ ] Testes: deploy v2 Green → switch → requests servidos por v2; rollback → requests servidos por v1

---

### Desafio 7.3 — Canary Release: Matching Algorithm Progressivo

Implemente canary release para lançar um novo algoritmo de matching gradualmente.

**Cenário:** O time implementou um novo algoritmo de matching (v2) que promete 30% menos tempo de espera. Mas é arriscado enviar para 100% dos usuários. Solução: canary release 5% → 25% → 50% → 100%.

**Requisitos:**
- **Weight-based routing:** percentual do tráfego para canary
  - Stage 1: 5% → v2 (canary), 95% → v1 (stable)
  - Stage 2: 25% → v2, 75% → v1 (se métricas OK por 30min)
  - Stage 3: 50% → v2, 50% → v1
  - Stage 4: 100% → v2 (promoção)
- **Métricas de validação** (comparar canary vs stable):
  - Error rate: canary ≤ stable + 1%
  - P99 latency: canary ≤ stable + 200ms
  - Business metric: average matching time (canary deve ser melhor)
- **Rollback automático:** se error rate canary > 5%, reverter para 0% instantaneamente
- **Sticky sessions:** mesmo passageiro sempre vai para mesma versão (evitar inconsistência)

**Java 25 (Canary Router):**
```java
@Component
public class CanaryRouter {
    private final AtomicInteger canaryWeight = new AtomicInteger(5); // 5% inicial
    private final Map<UUID, String> stickyMap = new ConcurrentHashMap<>();
    
    public String route(UUID userId) {
        // Sticky: mesmo usuário = mesma versão
        return stickyMap.computeIfAbsent(userId, id -> {
            int hash = Math.abs(id.hashCode() % 100);
            return hash < canaryWeight.get() ? "canary" : "stable";
        });
    }
    
    public void promoteCanary(int newWeight) {
        canaryWeight.set(newWeight);
        stickyMap.clear(); // recalcular afinidade
        log.info("Canary weight updated to {}%", newWeight);
    }
    
    public void rollbackCanary() {
        canaryWeight.set(0);
        stickyMap.clear();
        log.warn("CANARY ROLLBACK: all traffic to stable");
    }
}

// Canary Health Monitor
@Component
public class CanaryHealthMonitor {
    
    @Scheduled(fixedRate = 30_000)
    public void evaluateCanary() {
        var canaryMetrics = metricsService.getMetrics("canary");
        var stableMetrics = metricsService.getMetrics("stable");
        
        // Error rate check
        if (canaryMetrics.errorRate() > stableMetrics.errorRate() + 0.01) {
            log.error("Canary error rate {}% exceeds stable {}% + 1%",
                canaryMetrics.errorRate() * 100, stableMetrics.errorRate() * 100);
            canaryRouter.rollbackCanary();
            return;
        }
        
        // Latency check
        if (canaryMetrics.p99Latency() > stableMetrics.p99Latency() + Duration.ofMillis(200)) {
            log.warn("Canary P99 {}ms exceeds stable {}ms + 200ms",
                canaryMetrics.p99Latency().toMillis(), stableMetrics.p99Latency().toMillis());
            // Warning, não rollback automático para latência
        }
        
        // Business metric: matching time
        log.info("Matching time — canary: {}s, stable: {}s",
            canaryMetrics.avgMatchingTime(), stableMetrics.avgMatchingTime());
    }
}
```

**Go 1.26:**
```go
type CanaryRouter struct {
    canaryWeight atomic.Int32
    stickyMap    sync.Map // userID → "canary" | "stable"
}

func (r *CanaryRouter) Route(userID uuid.UUID) string {
    if val, ok := r.stickyMap.Load(userID); ok {
        return val.(string)
    }
    
    hash := int(binary.BigEndian.Uint32(userID[:4])) % 100
    version := "stable"
    if hash < int(r.canaryWeight.Load()) {
        version = "canary"
    }
    r.stickyMap.Store(userID, version)
    return version
}

func (r *CanaryRouter) Promote(weight int) {
    r.canaryWeight.Store(int32(weight))
    r.stickyMap.Range(func(key, _ any) bool {
        r.stickyMap.Delete(key)
        return true
    })
    slog.Info("canary weight updated", "weight", weight)
}

func (r *CanaryRouter) Rollback() {
    r.canaryWeight.Store(0)
    r.stickyMap.Range(func(key, _ any) bool {
        r.stickyMap.Delete(key)
        return true
    })
    slog.Warn("CANARY ROLLBACK: all traffic to stable")
}

// Health monitor goroutine
func (m *CanaryMonitor) Run(ctx context.Context) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            canary := m.metrics.Get("canary")
            stable := m.metrics.Get("stable")
            
            if canary.ErrorRate > stable.ErrorRate+0.01 {
                slog.Error("canary error rate exceeds threshold",
                    "canary", canary.ErrorRate, "stable", stable.ErrorRate)
                m.router.Rollback()
                continue
            }
            
            slog.Info("canary metrics",
                "canary_matching_time", canary.AvgMatchingTime,
                "stable_matching_time", stable.AvgMatchingTime)
        }
    }
}
```

**Critérios de aceite:**
- [ ] Weight-based routing: 5% → 25% → 50% → 100% (configurável)
- [ ] Sticky sessions: mesmo userId sempre vai para mesma versão
- [ ] Métricas comparativas: error rate, P99 latency, matching time (canary vs stable)
- [ ] Rollback automático: error rate canary > stable + 1% → rollback para 0%
- [ ] Promoção via API: `POST /admin/canary/promote { weight: 25 }`
- [ ] Testes: 5% → 5 de 100 requests vão para canary; error rate alta → rollback; promote 100% → all canary

---

### Desafio 7.4 — Feature Flags: Surge Pricing Controlado

Implemente feature flags para controlar o lançamento de surge pricing por região e percentual.

**Cenário:** O time implementou "surge pricing" (preço dinâmico baseado em demanda). Quer lançar primeiro em SP para 10% dos passageiros, depois expandir. Feature flags controlam sem deploy.

**Requisitos:**
- **5 tipos de flags:**
  1. **Boolean:** `surge-pricing.enabled` (on/off global)
  2. **Percentage rollout:** `surge-pricing.rollout-percentage` (10% → 50% → 100%)
  3. **Regional:** `surge-pricing.regions` (apenas `["SP", "RJ"]`)
  4. **User segment:** `surge-pricing.beta-users` (lista de user IDs)
  5. **Time-based:** `surge-pricing.peak-only` (ativo apenas 7-9h e 17-20h)
- **Evaluation context:** flag avaliada com (userId, region, time)
- **Fallback:** se o flag service estiver indisponível, usar valor padrão (off)
- **Audit log:** cada avaliação de flag logada para debugging
- **Admin API:** criar, atualizar, ativar/desativar flags sem deploy

**Java 25 (Togglz / ff4j):**
```java
@Service
public class FeatureFlagService {
    private final FeatureFlagStore store; // Consul KV ou banco
    
    public boolean isEnabled(String flagName, EvaluationContext ctx) {
        try {
            var flag = store.getFlag(flagName);
            if (flag == null || !flag.isEnabled()) return false;
            
            boolean result = evaluate(flag, ctx);
            
            // Audit log
            auditLog.log(flagName, ctx.userId(), result);
            
            return result;
        } catch (Exception e) {
            log.warn("Flag service unavailable, using default for {}", flagName);
            return flag != null && flag.getDefaultValue();
        }
    }
    
    private boolean evaluate(FeatureFlag flag, EvaluationContext ctx) {
        return switch (flag.getType()) {
            case BOOLEAN -> true; // se enabled, é true
            case PERCENTAGE -> {
                int hash = Math.abs(ctx.userId().hashCode() % 100);
                yield hash < flag.getPercentage();
            }
            case REGIONAL -> flag.getRegions().contains(ctx.region());
            case USER_SEGMENT -> flag.getBetaUsers().contains(ctx.userId());
            case TIME_BASED -> {
                var hour = LocalTime.now().getHour();
                yield flag.getActiveHours().stream()
                    .anyMatch(range -> hour >= range.start() && hour <= range.end());
            }
        };
    }
}

// Uso no PricingService
@Service
public class PricingService {
    
    public PriceEstimate calculatePrice(Ride ride, UUID passengerId) {
        var basePrice = calculateBasePrice(ride);
        
        var ctx = new EvaluationContext(passengerId, ride.getCityCode(), Instant.now());
        
        if (featureFlags.isEnabled("surge-pricing.enabled", ctx)) {
            var surgeMultiplier = surgeService.getMultiplier(ride.getCityCode());
            return basePrice.withSurge(surgeMultiplier);
        }
        
        return basePrice;
    }
}
```

**Go 1.26 (go-feature-flag ou custom):**
```go
type FlagType string

const (
    FlagBoolean    FlagType = "BOOLEAN"
    FlagPercentage FlagType = "PERCENTAGE"
    FlagRegional   FlagType = "REGIONAL"
    FlagUserSegment FlagType = "USER_SEGMENT"
    FlagTimeBased  FlagType = "TIME_BASED"
)

type FeatureFlag struct {
    Name        string    `json:"name"`
    Enabled     bool      `json:"enabled"`
    Type        FlagType  `json:"type"`
    Percentage  int       `json:"percentage,omitempty"`
    Regions     []string  `json:"regions,omitempty"`
    BetaUsers   []string  `json:"betaUsers,omitempty"`
    ActiveHours []TimeRange `json:"activeHours,omitempty"`
    Default     bool      `json:"default"`
}

type EvaluationContext struct {
    UserID string
    Region string
    Time   time.Time
}

func (s *FeatureFlagService) IsEnabled(flagName string, ctx EvaluationContext) bool {
    flag, err := s.store.GetFlag(flagName)
    if err != nil {
        slog.Warn("flag service unavailable, using default", "flag", flagName)
        if flag != nil {
            return flag.Default
        }
        return false
    }
    
    if !flag.Enabled {
        return false
    }
    
    result := s.evaluate(flag, ctx)
    s.auditLog.Log(flagName, ctx.UserID, result)
    return result
}

func (s *FeatureFlagService) evaluate(flag *FeatureFlag, ctx EvaluationContext) bool {
    switch flag.Type {
    case FlagBoolean:
        return true
    case FlagPercentage:
        h := fnv.New32a()
        h.Write([]byte(ctx.UserID))
        return int(h.Sum32()%100) < flag.Percentage
    case FlagRegional:
        return slices.Contains(flag.Regions, ctx.Region)
    case FlagUserSegment:
        return slices.Contains(flag.BetaUsers, ctx.UserID)
    case FlagTimeBased:
        hour := ctx.Time.Hour()
        for _, tr := range flag.ActiveHours {
            if hour >= tr.Start && hour <= tr.End {
                return true
            }
        }
        return false
    }
    return false
}
```

**Critérios de aceite:**
- [ ] 5 tipos de flags implementados (boolean, percentage, regional, user segment, time-based)
- [ ] Evaluation context com userId, region, time
- [ ] Percentage rollout determinístico (mesmo userId = mesmo resultado, baseado em hash)
- [ ] Fallback: flag service indisponível → valor default
- [ ] Audit log: cada avaliação registrada (flag, userId, result)
- [ ] Admin API: CRUD de flags sem deploy
- [ ] Testes: SP + 10% → 10% dos users SP veem surge; RJ → ninguém vê; flag off → ninguém vê

---

### Desafio 7.5 — Shadow Traffic: Validar Novo Payment Gateway

Implemente shadow traffic para testar um novo payment gateway sem impactar usuários.

**Cenário:** O RideFlow vai trocar de payment gateway (v1 → v2). Antes de migrar, queremos replicar tráfego de produção para o novo gateway (shadow), comparar resultados, e validar sem impactar usuários reais.

**Requisitos:**
- **Traffic mirroring:** cada request ao payment-service v1 é replicado para v2 (shadow)
- **V1 é authoritative:** resposta ao cliente SEMPRE vem de v1
- Shadow call é **fire-and-forget** (não bloqueia, não afeta latência)
- **Comparação** de respostas (assíncrona):
  - Campos: status code, amount, transaction_id matching rules
  - Divergências logadas com full context (request + response v1 + response v2)
- **Métricas de shadow:**
  - `shadow.match_rate` — percentual de respostas idênticas
  - `shadow.error_rate` — percentual de erros no v2
  - `shadow.latency_diff` — diferença de latência v1 vs v2
- **Safety:** shadow nunca debita dinheiro real (usar sandbox/dry-run mode no v2)
- Shadow ativado via feature flag: `payment.shadow.enabled`

**Java 25:**
```java
@Service
public class PaymentServiceWithShadow {
    private final PaymentGatewayV1 v1;
    private final PaymentGatewayV2 v2Shadow;
    private final FeatureFlagService flags;
    private final ExecutorService shadowExecutor;
    
    public PaymentResult processPayment(PaymentRequest request) {
        // V1 é authoritative — sempre retorna a resposta de v1
        var result = v1.process(request);
        
        // Shadow: fire-and-forget para v2
        if (flags.isEnabled("payment.shadow.enabled")) {
            shadowExecutor.submit(() -> {
                try {
                    var shadowRequest = request.withDryRun(true); // sandbox mode
                    long start = System.currentTimeMillis();
                    var shadowResult = v2Shadow.process(shadowRequest);
                    long shadowLatency = System.currentTimeMillis() - start;
                    
                    compareAndLog(request, result, shadowResult, shadowLatency);
                } catch (Exception e) {
                    shadowErrorCounter.increment();
                    log.warn("Shadow payment failed: {}", e.getMessage());
                }
            });
        }
        
        return result;
    }
    
    private void compareAndLog(PaymentRequest req, PaymentResult v1Result, 
                               PaymentResult v2Result, long shadowLatency) {
        boolean match = v1Result.statusCode() == v2Result.statusCode()
            && v1Result.amount().equals(v2Result.amount());
        
        if (match) {
            shadowMatchCounter.increment();
        } else {
            shadowMismatchCounter.increment();
            log.warn("Shadow mismatch: request={}, v1={}, v2={}", req, v1Result, v2Result);
        }
        
        shadowLatencyHistogram.record(shadowLatency);
    }
}
```

**Go 1.26:**
```go
type PaymentServiceWithShadow struct {
    v1       PaymentGateway
    v2Shadow PaymentGateway
    flags    *FeatureFlagService
    metrics  *ShadowMetrics
}

func (s *PaymentServiceWithShadow) ProcessPayment(
    ctx context.Context, req PaymentRequest,
) (*PaymentResult, error) {
    // V1 é authoritative
    result, err := s.v1.Process(ctx, req)
    if err != nil {
        return nil, err
    }
    
    // Shadow: fire-and-forget
    if s.flags.IsEnabled("payment.shadow.enabled", EvaluationContext{}) {
        go func() {
            shadowReq := req
            shadowReq.DryRun = true // sandbox mode
            
            start := time.Now()
            shadowResult, err := s.v2Shadow.Process(context.Background(), shadowReq)
            shadowLatency := time.Since(start)
            
            if err != nil {
                s.metrics.ShadowErrors.Inc()
                slog.Warn("shadow payment failed", "error", err.Error())
                return
            }
            
            s.compareAndLog(req, result, shadowResult, shadowLatency)
        }()
    }
    
    return result, nil
}

func (s *PaymentServiceWithShadow) compareAndLog(
    req PaymentRequest, v1, v2 *PaymentResult, latency time.Duration,
) {
    match := v1.StatusCode == v2.StatusCode && v1.Amount == v2.Amount
    
    if match {
        s.metrics.MatchCount.Inc()
    } else {
        s.metrics.MismatchCount.Inc()
        slog.Warn("shadow mismatch",
            "request", req.ID,
            "v1_status", v1.StatusCode, "v2_status", v2.StatusCode,
            "v1_amount", v1.Amount, "v2_amount", v2.Amount)
    }
    
    s.metrics.LatencyDiff.Observe(latency.Seconds())
}
```

**Critérios de aceite:**
- [ ] Cada request v1 replicado para v2 (shadow) quando flag ativo
- [ ] Resposta ao cliente SEMPRE de v1 (v2 nunca impacta usuário)
- [ ] Shadow é fire-and-forget (não aumenta latência da resposta v1)
- [ ] V2 em dry-run/sandbox (nunca debita dinheiro real)
- [ ] Comparação assíncrona: match rate, error rate, latency diff registrados
- [ ] Divergências logadas com full context (request, v1 response, v2 response)
- [ ] Feature flag controla ativação do shadow
- [ ] Testes: shadow habilitado → v2 chamado; desabilitado → v2 não chamado; mismatch → logado
