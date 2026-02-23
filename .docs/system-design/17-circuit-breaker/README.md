# 17. Circuit Breaker

> **Categoria:** Resiliência e Tolerância a Falhas  
> **Nível:** Essencial para qualquer entrevista de System Design  
> **Complexidade:** Média

---

## Definição

**Circuit Breaker** é um padrão de resiliência que impede chamadas a serviços que estão falhando, permitindo que se recuperem e **evitando cascading failures** (efeito dominó). Inspirado no disjuntor elétrico: quando detecta falhas excessivas, "abre" o circuito e **rejeita requests imediatamente** (fail-fast) em vez de esperar timeouts.

---

## Por Que é Importante?

- **Previne cascading failures** — falha em um serviço não derruba toda a cadeia
- **Fail-fast** — retorna erro imediatamente em vez de esperar timeout (libera threads/conexões)
- **Dá tempo ao serviço** — permite que o serviço degradado se recupere sem carga
- **Graceful degradation** — permite retornar fallbacks em vez de erros puros
- **Economiza recursos** — não gasta threads/conexões em requests que vão falhar
- **Padrão obrigatório** em arquiteturas de microservices

---

## Diagrama — Os 3 Estados

```
  ┌──────────────────────────────────────────────────────────┐
  │                CIRCUIT BREAKER STATE MACHINE             │
  │                                                          │
  │     ┌────────────┐                                       │
  │     │   CLOSED   │◀──────────── success ─────────┐      │
  │     │            │                                │      │
  │     │ (normal)   │  requests passam normalmente   │      │
  │     │ conta      │  falhas são contadas           │      │
  │     │ falhas     │                                │      │
  │     └─────┬──────┘                                │      │
  │           │                                       │      │
  │           │ failure rate > threshold               │      │
  │           │ (ex: 50% de falha em 10 calls)        │      │
  │           ▼                                       │      │
  │     ┌────────────┐                          ┌─────┴────┐ │
  │     │    OPEN    │───── wait timeout ──────▶│HALF-OPEN │ │
  │     │            │     (ex: 30 seconds)     │          │ │
  │     │ (proteção) │                          │ (teste)  │ │
  │     │ rejeita    │◀──── failure ────────────│ permite  │ │
  │     │ tudo       │                          │ poucos   │ │
  │     │ fail-fast  │                          │ requests │ │
  │     └────────────┘                          └──────────┘ │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

### Estado: CLOSED (Normal)

```
  Requests passam normalmente.
  O circuit breaker monitora falhas em sliding window.
  
  Client ──▶ [Circuit Breaker: CLOSED] ──▶ Service
  Client ◀── response ◀──────────────────── Service
  
  Tracking:
  ┌────────────────────────────────────────┐
  │ Sliding Window (últimas 10 calls):     │
  │ ✅ ✅ ✅ ❌ ✅ ✅ ❌ ✅ ✅ ✅         │
  │ Failure rate: 2/10 = 20%               │
  │ Threshold: 50%                         │
  │ → 20% < 50% → permanece CLOSED        │
  └────────────────────────────────────────┘
```

### Estado: OPEN (Proteção)

```
  Todos os requests são rejeitados IMEDIATAMENTE (fail-fast).
  Nenhum request chega ao serviço degradado.
  
  Client ──▶ [Circuit Breaker: OPEN] ──╳──▶ Service (nem chega)
  Client ◀── fallback / error 503
  
  ┌──────────────────────────────────────────────┐
  │ Failure rate excedeu threshold!               │
  │ ❌ ✅ ❌ ❌ ✅ ❌ ❌ ❌ ✅ ❌              │
  │ Failure rate: 7/10 = 70% > 50% threshold     │
  │ → CIRCUIT OPEN!                               │
  │ → Timer: aguardar 30s antes de tentar de novo │
  └──────────────────────────────────────────────┘
  
  Neste estado:
  → Libera threads imediatamente
  → Retorna fallback (cache, default, error amigável)
  → Serviço tem tempo para se recuperar
```

### Estado: HALF-OPEN (Teste)

```
  Após wait timeout, permite POUCOS requests de teste.
  Se sucesso → volta para CLOSED.
  Se falha → volta para OPEN.
  
  Client ──▶ [Circuit Breaker: HALF-OPEN] ──▶ Service
  
  Permite N requests de teste (ex: 3):
  
  Test 1: ✅ success
  Test 2: ✅ success
  Test 3: ✅ success → All passed! → Estado volta para CLOSED ✅
  
  ── ou ──
  
  Test 1: ✅ success
  Test 2: ❌ failure → Falhou! → Estado volta para OPEN ❌
                       → Novo wait timeout de 30s
```

---

## Cascading Failure — O Problema

```
  Sem Circuit Breaker:
  
  ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
  │ Service │────▶│ Service │────▶│ Service │────▶│ Service │
  │    A    │     │    B    │     │    C    │     │  D (💥) │
  └─────────┘     └─────────┘     └─────────┘     └─────────┘
  
  1. Service D fica lento (ex: DB sobrecarregado)
  2. Service C espera timeout de D → threads de C se esgotam
  3. Service B espera timeout de C → threads de B se esgotam
  4. Service A espera timeout de B → threads de A se esgotam
  5. TUDO CAIU! 💥 (cascading failure)
  
  Tempo total: 4 × timeout = minutos de latência horrível
  
  ────────────────────────────────────────────────
  
  Com Circuit Breaker:
  
  ┌─────────┐     ┌─────────┐     ┌──────────┐     ┌─────────┐
  │ Service │────▶│ Service │────▶│ Service  │──╳──│ Service │
  │    A    │     │    B    │     │ C [CB:⚡]│     │  D (💥) │
  └─────────┘     └─────────┘     └──────────┘     └─────────┘
  
  1. Service D fica lento
  2. Circuit Breaker em C detecta falhas → ABRE → fail-fast
  3. C retorna fallback para B imediatamente (ms, não seconds)
  4. B responde A normalmente
  5. Apenas D está degradado, resto do sistema FUNCIONA ✅
```

---

## Configurações-Chave

| Parâmetro | Descrição | Valor Típico |
|-----------|-----------|--------------|
| `failureRateThreshold` | % de falhas para abrir circuito | 50% |
| `slowCallRateThreshold` | % de calls lentas para abrir | 80% |
| `slowCallDurationThreshold` | Duração para considerar "lenta" | 2000ms |
| `waitDurationInOpenState` | Tempo no OPEN antes de HALF-OPEN | 30s - 60s |
| `slidingWindowType` | COUNT_BASED ou TIME_BASED | COUNT_BASED |
| `slidingWindowSize` | Tamanho da janela de avaliação | 10-100 calls |
| `minimumNumberOfCalls` | Mínimo de calls antes de avaliar | 5-10 |
| `permittedNumberOfCallsInHalfOpenState` | Calls de teste em HALF-OPEN | 3-5 |
| `recordExceptions` | Exceções que contam como falha | IOException, TimeoutException |
| `ignoreExceptions` | Exceções ignoradas (não contam) | BusinessException |

---

## Sliding Window Types

### Count-Based

```
  Avalia as últimas N chamadas (ex: N=10):
  
  ┌─────────────────────────────┐
  │ Window: últimas 10 calls    │
  │                             │
  │ ✅ ❌ ✅ ✅ ❌ ❌ ❌ ✅ ❌ ❌ │
  │ Failures: 6/10 = 60%       │
  │ Threshold: 50%              │
  │ → 60% > 50% → OPEN!        │
  └─────────────────────────────┘
  
  ✅ Prós: Simples, previsível
  ❌ Contras: Pode demorar para reagir se tráfego é baixo
```

### Time-Based

```
  Avalia chamadas no último intervalo (ex: últimos 60s):
  
  ┌────────────────────────────────────┐
  │ Window: últimos 60 segundos        │
  │                                    │
  │ 14:00:05 ✅                        │
  │ 14:00:12 ❌                        │
  │ 14:00:18 ❌                        │
  │ 14:00:25 ❌                        │
  │ 14:00:33 ✅                        │
  │ 14:00:41 ❌                        │
  │ 14:00:50 ❌                        │
  │ 14:00:58 ❌                        │
  │                                    │
  │ Failures: 6/8 = 75%               │
  │ Threshold: 50%                     │
  │ → 75% > 50% → OPEN!               │
  └────────────────────────────────────┘
  
  ✅ Prós: Reage mesmo com tráfego baixo
  ❌ Contras: Pode ser sensível demais em low-traffic
```

---

## Fallback Strategies

| Estratégia | Descrição | Quando Usar |
|------------|-----------|-------------|
| **Cache** | Retorna último valor cacheado | Dados que podem ser stale (catálogo, perfil) |
| **Default Value** | Retorna valor padrão | Recomendações, features opcionais |
| **Degraded Service** | Versão simplificada da funcionalidade | Busca → mostra populares |
| **Error Response** | Retorna erro amigável com retry info | Operações críticas sem fallback |
| **Queue for Later** | Salva request para processar depois | Escritas (async) |
| **Alternative Service** | Chama serviço backup/alternativo | Multi-provider (ex: SMS) |

```
  Exemplo: Product Recommendation Service

  ┌──────────┐   ┌──────────────┐   ┌─────────────────────┐
  │  Client  │──▶│ Product Page │──▶│ Recommendation Svc  │
  └──────────┘   └──────┬───────┘   └─────────────────────┘
                        │                     💥 DOWN!
                        │
                        ▼ Fallback:
                   ┌──────────────────────────────┐
                   │ Return "Most Popular Items"  │
                   │ from local cache             │
                   └──────────────────────────────┘
  
  → User vê recomendações genéricas em vez de erro
  → Experiência degradada mas funcional
```

---

## Implementação — Resilience4j (Java)

### Configuração

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)                      // 50% failure → OPEN
    .slowCallRateThreshold(80)                     // 80% slow → OPEN
    .slowCallDurationThreshold(Duration.ofSeconds(2))
    .waitDurationInOpenState(Duration.ofSeconds(30)) // 30s antes de HALF-OPEN
    .slidingWindowType(SlidingWindowType.COUNT_BASED)
    .slidingWindowSize(10)                          // últimas 10 calls
    .minimumNumberOfCalls(5)                        // mínimo para avaliar
    .permittedNumberOfCallsInHalfOpenState(3)       // 3 calls de teste
    .recordExceptions(IOException.class, TimeoutException.class)
    .ignoreExceptions(BusinessException.class)
    .build();

CircuitBreaker circuitBreaker = CircuitBreaker.of("paymentService", config);
```

### Uso com Decorators

```java
// Decorar supplier com circuit breaker
Supplier<PaymentResponse> decoratedSupplier = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> paymentService.process(request));

// Executar com fallback
Try<PaymentResponse> result = Try.ofSupplier(decoratedSupplier)
    .recover(CallNotPermittedException.class, e -> {
        // Circuit is OPEN → return fallback
        log.warn("Circuit OPEN for payment service, returning queued response");
        return new PaymentResponse("QUEUED", "Payment will be processed later");
    })
    .recover(Exception.class, e -> {
        log.error("Payment failed", e);
        return new PaymentResponse("FAILED", e.getMessage());
    });
```

### Spring Boot Integration

```java
@Service
public class OrderService {

    @CircuitBreaker(name = "inventoryService", fallbackMethod = "inventoryFallback")
    @Retry(name = "inventoryService")           // retry ANTES do circuit breaker
    @TimeLimiter(name = "inventoryService")     // timeout
    public CompletableFuture<InventoryResponse> checkInventory(String productId) {
        return CompletableFuture.supplyAsync(() ->
            inventoryClient.check(productId)
        );
    }

    // Fallback: chamado quando circuito está OPEN ou call falha
    public CompletableFuture<InventoryResponse> inventoryFallback(
            String productId, Throwable t) {
        log.warn("Inventory check failed for {}, using cached data", productId, t);
        return CompletableFuture.completedFuture(
            inventoryCache.getLastKnown(productId)
                .orElse(InventoryResponse.unknown())
        );
    }
}
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      inventoryService:
        register-health-indicator: true
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        slow-call-duration-threshold: 2s
        slow-call-rate-threshold: 80
  retry:
    instances:
      inventoryService:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
```

### Python — Circuit Breaker

```python
import time
from enum import Enum
from threading import Lock
from collections import deque

class State(Enum):
    CLOSED = "CLOSED"
    OPEN = "OPEN"
    HALF_OPEN = "HALF_OPEN"

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: float = 0.5,   # 50%
        window_size: int = 10,
        wait_duration: float = 30.0,      # seconds
        half_open_max_calls: int = 3,
    ):
        self.failure_threshold = failure_threshold
        self.window_size = window_size
        self.wait_duration = wait_duration
        self.half_open_max_calls = half_open_max_calls
        
        self.state = State.CLOSED
        self.results = deque(maxlen=window_size)  # True=success, False=failure
        self.opened_at = 0.0
        self.half_open_calls = 0
        self.lock = Lock()
    
    def _failure_rate(self) -> float:
        if len(self.results) == 0:
            return 0.0
        failures = sum(1 for r in self.results if not r)
        return failures / len(self.results)
    
    def call(self, func, *args, fallback=None, **kwargs):
        with self.lock:
            if self.state == State.OPEN:
                if time.time() - self.opened_at >= self.wait_duration:
                    self.state = State.HALF_OPEN
                    self.half_open_calls = 0
                else:
                    if fallback:
                        return fallback(*args, **kwargs)
                    raise CircuitOpenError("Circuit is OPEN")
            
            if self.state == State.HALF_OPEN:
                if self.half_open_calls >= self.half_open_max_calls:
                    if fallback:
                        return fallback(*args, **kwargs)
                    raise CircuitOpenError("HALF_OPEN: max test calls reached")
                self.half_open_calls += 1
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            if fallback:
                return fallback(*args, **kwargs)
            raise
    
    def _on_success(self):
        with self.lock:
            self.results.append(True)
            if self.state == State.HALF_OPEN:
                if self.half_open_calls >= self.half_open_max_calls:
                    self.state = State.CLOSED
                    self.results.clear()
    
    def _on_failure(self):
        with self.lock:
            self.results.append(False)
            if self.state == State.HALF_OPEN:
                self.state = State.OPEN
                self.opened_at = time.time()
            elif self.state == State.CLOSED:
                if (len(self.results) >= self.window_size and
                    self._failure_rate() >= self.failure_threshold):
                    self.state = State.OPEN
                    self.opened_at = time.time()

class CircuitOpenError(Exception):
    pass


# Uso
cb = CircuitBreaker(failure_threshold=0.5, window_size=10, wait_duration=30)

def get_product(product_id):
    return requests.get(f"http://product-svc/products/{product_id}").json()

def get_product_fallback(product_id):
    return cache.get(f"product:{product_id}") or {"name": "Unknown", "price": 0}

result = cb.call(get_product, "123", fallback=get_product_fallback)
```

---

## Circuit Breaker + Retry + Timeout — Composição

```
  Ordem recomendada (de fora para dentro):
  
  Client Request
       │
       ▼
  ┌──────────────────┐   Aberto? → Fallback imediatamente
  │  Circuit Breaker │
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐   Máximo de tentativas?
  │      Retry       │   Ex: 3 tentativas com backoff
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐   Passou do tempo? → TimeoutException
  │    Timeout       │   Ex: 2 segundos
  └────────┬─────────┘
           │
           ▼
     Service Call
  
  Fluxo:
  1. Circuit Breaker verifica se pode chamar
  2. Retry tenta até 3x em caso de falha
  3. Cada tentativa tem timeout de 2s
  4. Se todas falharem → Circuit Breaker conta como falha
  5. Se failure rate > threshold → Circuit abre
```

---

## Circuit Breaker em Service Mesh (Envoy / Istio)

```yaml
# Istio DestinationRule — Circuit Breaker
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100        # max conexões TCP
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 50  # max pending requests
        http2MaxRequests: 100        # max concurrent requests
        maxRequestsPerConnection: 10
    outlierDetection:               # ← Circuit Breaker
      consecutive5xxErrors: 5       # 5 erros 5xx → eject
      interval: 10s                 # avaliar a cada 10s
      baseEjectionTime: 30s         # tempo fora do pool
      maxEjectionPercent: 50        # max 50% ejected
```

```
  Envoy Outlier Detection (Circuit Breaker no Mesh):
  
  ┌──────────────────────────────────────────────────────┐
  │ Service A → [Envoy Sidecar] → Service B instances:   │
  │                                                       │
  │ Instance B1: ✅ healthy → recebe tráfego             │
  │ Instance B2: ❌ 5xx × 5 → EJECTED (30s) → sem tráfego│
  │ Instance B3: ✅ healthy → recebe tráfego             │
  │                                                       │
  │ Após 30s: B2 volta ao pool (como HALF-OPEN)          │
  │ Se falhar de novo → ejection time dobra (60s, 120s...) │
  └──────────────────────────────────────────────────────┘
```

---

## Monitoramento e Observabilidade

```
  Métricas essenciais:
  
  ┌───────────────────────────────────────────────────┐
  │ Circuit Breaker Dashboard                         │
  │                                                   │
  │ Service: payment-service                          │
  │ State: 🔴 OPEN (since 14:23:45)                  │
  │ Failure Rate: 72% (threshold: 50%)                │
  │ Slow Call Rate: 45%                               │
  │                                                   │
  │ Last 100 calls:                                   │
  │ ████████████████████░░░░░░░░░░ 28% success        │
  │                                                   │
  │ State Transitions (last 1h):                      │
  │ CLOSED → OPEN (14:23:45)                          │
  │ OPEN → HALF_OPEN (14:24:15)                       │
  │ HALF_OPEN → OPEN (14:24:18)                       │
  │ OPEN → HALF_OPEN (14:24:48)                       │
  │ HALF_OPEN → CLOSED (14:24:51) ✅                  │
  └───────────────────────────────────────────────────┘
```

**Métricas a monitorar:**

| Métrica | Tipo | Alerta |
|---------|------|--------|
| `circuit_breaker_state` | Gauge (0/1/2) | State != CLOSED por > 5min |
| `circuit_breaker_failure_rate` | Gauge | > threshold warning |
| `circuit_breaker_calls_total` | Counter | Quick drop = circuit open |
| `circuit_breaker_not_permitted_calls` | Counter | Increasing = circuit open |
| `circuit_breaker_slow_call_rate` | Gauge | > slow threshold |

---

## Uso em Big Techs

### Netflix — Hystrix (pioneiros)
- Inventaram o Hystrix (2012) — primeiro circuit breaker library popular
- Thread pool isolation: cada dependência tem pool separado
- Hystrix Dashboard para visualização real-time
- **Deprecado em 2018** → migrado para Resilience4j
- Centenas de circuit breakers monitorando centenas de microservices

### Amazon — Client-Side Resilience
- Circuit breakers em todos os clientes SDK
- Exponential backoff + jitter como padrão
- Cell-based architecture: blast radius limitado
- Cada serviço define seus próprios limites de falha

### Uber — Service Resilience
- Circuit breakers em comunicação inter-service
- Integrado com service mesh (Envoy)
- Alertas automáticos quando circuito abre
- Runbooks vinculados a cada circuit breaker

### Google — Load Shedding + Circuit Breaking
- gRPC deadlines propagam timeouts pela cadeia
- Backend-driven flow control
- Client-side circuit breaking com gRPC interceptors
- Site Reliability Engineering (SRE) define thresholds

---

## Perguntas Comuns em Entrevistas

1. **O que é Circuit Breaker e por que é necessário?**
   - Padrão que previne cascading failures. Quando serviço falha acima de threshold, rejeita requests imediatamente (fail-fast).

2. **Quais são os 3 estados?**
   - CLOSED (normal, conta falhas), OPEN (rejeita tudo, fail-fast), HALF-OPEN (testa com poucos requests).

3. **Qual a diferença entre Circuit Breaker e Retry?**
   - Retry tenta de novo (mesma request). Circuit Breaker para de tentar quando detecta que não vai funcionar.

4. **Como evitar cascading failures sem circuit breaker?**
   - Timeouts agressivos, bulkhead (thread pool isolation), rate limiting, load shedding.

5. **Circuit Breaker no API Gateway vs no client?**
   - API Gateway: protege o backend de forma centralizada. Client: protege cada client individualmente, mais granular.

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Window** | Count-based (previsível) | Time-based (reativo) |
| **Threshold** | Baixo (sensível, abre rápido) | Alto (tolerante, abre tarde) |
| **Wait duration** | Curto (testa logo) | Longo (mais tempo para recuperar) |
| **Fallback** | Cache/default (UX boa) | Error response (honesto) |
| **Escopo** | Per-host (granular) | Per-service (simples) |
| **Implementação** | Library (Resilience4j) | Service Mesh (Istio/Envoy) |
| **Isolation** | Thread pool (Hystrix-style) | Semaphore (Resilience4j default) |

---

## Referências

- [Resilience4j — Circuit Breaker](https://resilience4j.readme.io/docs/circuitbreaker)
- [Martin Fowler — Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Netflix — Hystrix Wiki](https://github.com/Netflix/Hystrix/wiki)
- [Istio — Outlier Detection](https://istio.io/latest/docs/reference/config/networking/destination-rule/#OutlierDetection)
- [Michael Nygard — Release It!](https://pragprog.com/titles/mnee2/release-it-second-edition/) — Padrão original
- [Microsoft — Circuit Breaker Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
