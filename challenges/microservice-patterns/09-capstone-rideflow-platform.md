# Level 9 — Capstone: RideFlow Platform Completo

> **Objetivo:** Integrar TODOS os 27 patterns de microserviços em um sistema RideFlow funcional end-to-end,
> exercitando resiliência, consistência, observabilidade e deployment em cenário realista de produção.

**Pré-requisito:** [Level 8 — Observability](08-observability-metrics-logs-traces-slos.md) (todos os levels 0-8 concluídos)

**Referências:**
- Todos os source docs: [01 a 27](../../.docs/microservice-patterns/)
- Patterns aplicados: Circuit Breaker, Retry, Timeout, Rate Limiter, Bulkhead, Idempotência, DLQ, Saga (Orquestrada + Coreografada), API Composition, CQRS, Event Sourcing, Transactional Outbox, CDC, Backpressure, Sharding, Service Discovery, External Config, Health Checks, API Gateway, BFF, Sidecar, Strangler Fig, Blue-Green, Canary, Feature Flags, Shadow Traffic, Observabilidade (Métricas/Logs/Traces/SLOs)

---

## Contexto do Domínio

O RideFlow é uma plataforma de mobilidade urbana operando em **4 cidades** (SP, RJ, BH, CWB).
O Capstone simula um dia de operação real — do primeiro request matinal ao pico noturno — exercitando
todos os patterns sob falhas controladas e escalas variáveis.

### Arquitetura Final

```
                                    ┌──────────────┐
                                    │    Envoy      │
                                    │   Sidecar     │
                                    │   (mTLS)      │
                                    └──────┬───────┘
                                           │
┌──────────┐      ┌──────────────┐    ┌────▼─────┐    ┌──────────────┐
│ Passenger │──────│ API Gateway  │────│   BFF    │────│ Ride Service │
│   App     │      │  (SCG/Go)   │    │ Passenger│    │ (Event Store)│
└──────────┘      │ JWT+RateLimit│    └──────────┘    │  Saga Orch.  │
                   └──────────────┘                    └──────┬───────┘
┌──────────┐      ┌──────────────┐    ┌──────────┐           │ Events
│  Driver   │──────│ API Gateway  │────│   BFF    │    ┌──────▼───────┐
│   App     │      │  (SCG/Go)   │    │  Driver  │    │  Event Bus   │
└──────────┘      └──────────────┘    └──────────┘    │   (Kafka)    │
                                                       └──┬──┬──┬──┬─┘
                  ┌──────────┐  ┌──────────┐  ┌──────────▼┐│  │  │
                  │  Driver  │  │ Payment  │  │ Pricing   ││  │  │
                  │ Service  │  │ Service  │  │ Service   ││  │  │
                  │ (Consul) │  │ (CB+Retry│  │(Sharding) ││  │  │
                  └──────────┘  │  Saga)   │  └───────────┘│  │  │
                                └──────────┘     ┌─────────▼┐ │  │
                  ┌──────────┐                   │ Location  │ │  │
                  │Notificat.│                   │ Service   │ │  │
                  │ Service  │                   │(Backpress)│ │  │
                  │ (DLQ)    │                   └───────────┘ │  │
                  └──────────┘                                 │  │
                                        ┌─────────────────────▼┘  │
                                        │   Observability Stack   │
                                        │ Prometheus+Jaeger+Logs  │
                                        └─────────────────────────┘
```

### Serviços e Patterns por Serviço

| Serviço | Patterns Implementados |
|---|---|
| **API Gateway** | Rate Limiter, JWT, CORS, TraceId propagation, Routing |
| **BFF Passenger** | API Composition, Timeout, Fallback |
| **BFF Driver** | API Composition, Timeout, Fallback |
| **Ride Service** | Event Sourcing, Saga Orchestrator, CQRS, Idempotência, Outbox |
| **Driver Service** | Service Discovery, Health Checks, External Config |
| **Payment Service** | Circuit Breaker, Retry, Bulkhead, Saga Participant, DLQ |
| **Pricing Service** | Sharding (por cidade), Feature Flags (surge pricing) |
| **Location Service** | Backpressure, Rate Limiter, Batch Processing |
| **Notification Service** | DLQ, Retry, Bulkhead |
| **Envoy Sidecars** | mTLS, Retry, Metrics scraping |
| **Infra** | Consul (discovery + config), Kafka, PostgreSQL, Jaeger, Prometheus |

---

## Desafios

### Desafio 9.1 — Bootstrap: Infraestrutura e Serviços Integrados

Monte a infraestrutura completa com Docker Compose e valide que todos os serviços se registram, comunicam e são observáveis.

**Cenário:** Primeiro dia do time SRE. Subir toda a plataforma, validar que os serviços se encontram, que o config está distribuído e que health checks funcionam.

**Requisitos:**
- **Docker Compose** com todos os serviços + infra:
  - 8 serviços aplicação (Gateway, 2 BFFs, Ride, Driver, Payment, Pricing, Location, Notification)
  - Envoy sidecar para cada serviço (network_mode sharing)
  - Kafka (3 brokers) + Zookeeper
  - PostgreSQL (4 instâncias: ride-db, driver-db, payment-db, pricing-db com sharding)
  - Redis (rate limiter distribuído, cache)
  - Consul (service discovery + config KV)
  - Jaeger (traces)
  - Prometheus (metrics)
  - Debezium Connect (CDC)
- **Startup order** via healthchecks:
  1. Infra (Postgres, Kafka, Redis, Consul) → healthy
  2. Debezium → Kafka connected
  3. Serviços aplicação → DB migrated, registered in Consul
  4. Gateway → all backends discovered
- **Validação pós-startup:**
  - Todos os serviços visíveis no Consul UI
  - Health checks: `GET /health/ready` retorna 200 para cada serviço
  - Jaeger UI mostra traces (pelo menos startup traces)
  - Prometheus targets all UP
  - Kafka topics criados (ride-events, payment-events, location-updates, notifications, outbox-events, dlq-*)

**Java 25 (docker-compose.yml excerpt):**
```yaml
services:
  # --- Infra ---
  postgres-ride:
    image: postgres:16
    environment:
      POSTGRES_DB: rideflow_rides
      POSTGRES_USER: ride_user
      POSTGRES_PASSWORD: ride_pass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ride_user -d rideflow_rides"]
      interval: 5s
      retries: 5
    volumes:
      - ./ride-service/src/main/resources/db/migration:/docker-entrypoint-initdb.d

  consul:
    image: hashicorp/consul:1.18
    ports: ["8500:8500"]
    command: agent -server -bootstrap-expect=1 -ui -client=0.0.0.0
    healthcheck:
      test: ["CMD", "consul", "members"]
      interval: 5s
      retries: 5

  kafka-1:
    image: confluentinc/cp-kafka:7.6.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: "1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093"
      # ... KRaft config
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 10s
      retries: 10

  jaeger:
    image: jaegertracing/all-in-one:1.54
    ports: ["16686:16686", "4317:4317"]
    environment:
      COLLECTOR_OTLP_ENABLED: "true"

  prometheus:
    image: prom/prometheus:v2.50.0
    volumes:
      - ./infra/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports: ["9090:9090"]

  # --- Application Services ---
  ride-service:
    build: ./ride-service
    depends_on:
      postgres-ride: { condition: service_healthy }
      kafka-1: { condition: service_healthy }
      consul: { condition: service_healthy }
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-ride:5432/rideflow_rides
      SPRING_CLOUD_CONSUL_HOST: consul
      OTEL_EXPORTER_OTLP_ENDPOINT: http://jaeger:4317
      OTEL_SERVICE_NAME: ride-service
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/health/ready"]
      interval: 10s
      retries: 10

  ride-service-envoy:
    image: envoyproxy/envoy:v1.29
    network_mode: "service:ride-service"
    volumes:
      - ./infra/envoy/ride-service.yaml:/etc/envoy/envoy.yaml

  api-gateway:
    build: ./api-gateway
    depends_on:
      ride-service: { condition: service_healthy }
      driver-service: { condition: service_healthy }
      payment-service: { condition: service_healthy }
    ports: ["8080:8080"]
    environment:
      SPRING_CLOUD_CONSUL_HOST: consul
```

**Validation script:**
```bash
#!/bin/bash
set -e

echo "=== RideFlow Platform Validation ==="

# 1. Consul: all services registered
SERVICES=$(curl -s http://localhost:8500/v1/catalog/services | jq 'keys | length')
echo "Services in Consul: $SERVICES (expected >= 8)"

# 2. Health checks
for svc in ride-service driver-service payment-service pricing-service \
           location-service notification-service bff-passenger bff-driver; do
  PORT=$(curl -s "http://localhost:8500/v1/health/service/$svc?passing" | jq -r '.[0].Service.Port')
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:$PORT/health/ready")
  echo "$svc: health=$STATUS"
done

# 3. Jaeger: traces present
TRACE_COUNT=$(curl -s "http://localhost:16686/api/traces?service=api-gateway&limit=1" | jq '.data | length')
echo "Traces in Jaeger: $TRACE_COUNT"

# 4. Prometheus: targets up
TARGETS_UP=$(curl -s http://localhost:9090/api/v1/targets | jq '[.data.activeTargets[] | select(.health=="up")] | length')
echo "Prometheus targets UP: $TARGETS_UP"

# 5. Kafka: topics
TOPICS=$(docker exec kafka-1 kafka-topics --list --bootstrap-server localhost:9092 | wc -l)
echo "Kafka topics: $TOPICS"

echo "=== Validation Complete ==="
```

**Critérios de aceite:**
- [ ] `docker compose up` sobe toda a plataforma (ordem correta via healthchecks)
- [ ] Todos os serviços registrados no Consul e passando health checks
- [ ] Jaeger, Prometheus, Consul UI acessíveis
- [ ] Kafka topics criados automaticamente
- [ ] Validation script passa 100%
- [ ] Tempo total de startup < 3 minutos

---

### Desafio 9.2 — Jornada Completa: Ride Lifecycle com Falhas

Execute uma jornada completa de corrida que exercita todos os patterns, incluindo falhas controladas.

**Cenário:** Um passageiro solicita uma corrida em SP. O fluxo percorre TODOS os serviços, com falhas injetadas no meio: Payment fica lento (timeout/retry/CB), Location tem backpressure, Notification falha (DLQ).

**Requisitos:**

**Fase 1 — Happy Path (todos os patterns funcionando):**
1. `POST /passenger/rides` (Gateway → BFF → Ride Service)
   - Gateway: rate limit check, JWT validation, traceId generation
   - BFF: API Composition (ride details + pricing estimate)
   - Ride Service: Event Sourcing `RideRequested` event, idempotency check
   - Outbox: evento publicado via transactional outbox
2. Saga orquestrada inicia:
   - Step 1: Reserve driver (Driver Service via Service Discovery/Consul)
   - Step 2: Calculate price (Pricing Service, shard por cidade SP)
   - Step 3: Pre-authorize payment (Payment Service com CB + Retry)
   - Step 4: Send notification (Notification Service)
3. Driver aceita: `POST /driver/rides/{id}/accept`
   - Event Sourcing: `DriverMatched` event
   - CDCSink: ride_details_view atualizado via Debezium
4. Location updates durante a corrida:
   - GPS events (50/segundo por motorista) via Kafka
   - Location Service com backpressure (bounded buffer)
   - Batch processing para persistência
5. Corrida finalizada: `POST /driver/rides/{id}/complete`
   - Event Sourcing: `RideCompleted` event
   - Saga: charge payment, update driver stats, send receipt
   - CQRS: query model atualizado

**Fase 2 — Fault Injection (resiliência sob stress):**
6. **Payment timeout:** Simular payment-service respondendo em 10s
   - Esperado: timeout 5s → retry 3x → circuit breaker OPEN → fallback (queue for later)
   - Corrida NÃO deve ser cancelada (payment será processado depois)
7. **Driver Service down:** Derrubar driver-service container
   - Circuit breaker OPEN após 5 falhas
   - Health check: Consul marca como unhealthy (10s)
   - Requests redirecionados para fallback (cache de motoristas disponíveis)
   - Subir driver-service → CB transition HALF_OPEN → CLOSED
8. **Notification failure:** Notification service rejeita mensagem
   - Mensagem vai para DLQ (dlq-notifications)
   - Admin API: `GET /admin/dlq/notifications` lista mensagens
   - Reprocessar: `POST /admin/dlq/notifications/{id}/retry`
9. **Duplicate request:** Mesmo Idempotency-Key enviado 2x
   - Segunda request retorna resultado da primeira (sem criar nova corrida)
10. **Backpressure:** Enviar 50K location events/segundo
    - Location Service: bounded buffer (10K max), drop oldest per driver
    - Métricas: buffer utilization > 80% → log warning
    - HTTP 429 retornado quando buffer > 90%

**Java 25 / Go 1.26 — Integration Test:**
```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
class RideJourneyIntegrationTest {

    @Container static PostgreSQLContainer<?> postgres = ...;
    @Container static KafkaContainer kafka = ...;
    @Container static GenericContainer<?> consul = ...;
    
    @Test
    void shouldCompleteFullRideJourney() {
        // Phase 1: Happy path
        var createResponse = gateway.post("/passenger/rides",
            new CreateRideRequest("SP", origin, destination),
            headers("Idempotency-Key", idempotencyKey));
        
        assertThat(createResponse.statusCode()).isEqualTo(201);
        var rideId = createResponse.body().rideId();
        
        // Verify Event Sourcing
        var events = rideEventStore.getEvents(rideId);
        assertThat(events).hasSize(1);
        assertThat(events.getFirst()).isInstanceOf(RideRequested.class);
        
        // Verify Outbox
        await().atMost(5, SECONDS).until(() ->
            outboxRepository.findByAggregateId(rideId).stream()
                .anyMatch(e -> e.getStatus() == OutboxStatus.SENT));
        
        // Verify Saga started
        var saga = sagaRepository.findByRideId(rideId);
        assertThat(saga.getStatus()).isEqualTo(SagaStatus.STARTED);
        
        // Driver accepts
        var acceptResponse = gateway.post(
            "/driver/rides/" + rideId + "/accept",
            new AcceptRideRequest(driverId));
        assertThat(acceptResponse.statusCode()).isEqualTo(200);
        
        // Verify CQRS view updated
        await().atMost(10, SECONDS).until(() -> {
            var view = rideViewRepository.findById(rideId);
            return view.isPresent() && view.get().getDriverName() != null;
        });
        
        // Complete ride
        var completeResponse = gateway.post(
            "/driver/rides/" + rideId + "/complete",
            new CompleteRideRequest(driverId, distance, duration));
        assertThat(completeResponse.statusCode()).isEqualTo(200);
        
        // Verify final event stream
        var finalEvents = rideEventStore.getEvents(rideId);
        assertThat(finalEvents).extracting("class").containsExactly(
            RideRequested.class, DriverMatched.class,
            RideStarted.class, RideCompleted.class);
    }
    
    @Test
    void shouldHandlePaymentTimeoutWithCircuitBreaker() {
        // Inject fault: payment responds in 10s
        paymentWireMock.stubFor(post("/payments/authorize")
            .willReturn(ok().withFixedDelay(10_000)));
        
        var response = gateway.post("/passenger/rides", createRideRequest);
        
        // Ride created (payment queued for later)
        assertThat(response.statusCode()).isEqualTo(201);
        
        // Circuit breaker should be OPEN after retries
        await().atMost(30, SECONDS).until(() ->
            circuitBreakerRegistry.circuitBreaker("payment-service")
                .getState() == CircuitBreaker.State.OPEN);
        
        // Payment queued in DLQ or retry queue
        // Ride not cancelled
        var ride = rideRepository.findById(rideId);
        assertThat(ride.getStatus()).isNotEqualTo(RideStatus.CANCELLED);
    }
    
    @Test
    void shouldRejectDuplicateWithIdempotencyKey() {
        var key = UUID.randomUUID().toString();
        
        var first = gateway.post("/passenger/rides", createRideRequest,
            headers("Idempotency-Key", key));
        var second = gateway.post("/passenger/rides", createRideRequest,
            headers("Idempotency-Key", key));
        
        assertThat(first.statusCode()).isEqualTo(201);
        assertThat(second.statusCode()).isEqualTo(200); // cached result
        assertThat(first.body().rideId()).isEqualTo(second.body().rideId());
    }
}
```

```go
func TestRideJourneyE2E(t *testing.T) {
    ctx := context.Background()
    
    // Setup testcontainers
    pgContainer, _ := postgres.RunContainer(ctx)
    kafkaContainer, _ := kafka.RunContainer(ctx)
    defer pgContainer.Terminate(ctx)
    defer kafkaContainer.Terminate(ctx)
    
    // Phase 1: Happy path
    resp, err := gateway.Post("/passenger/rides", CreateRideRequest{
        CityCode: "SP", Origin: origin, Destination: destination,
    }, WithHeader("Idempotency-Key", idempotencyKey))
    require.NoError(t, err)
    assert.Equal(t, http.StatusCreated, resp.StatusCode)
    
    var result CreateRideResponse
    json.NewDecoder(resp.Body).Decode(&result)
    rideID := result.RideID
    
    // Verify Event Sourcing
    events, _ := eventStore.GetEvents(ctx, rideID)
    assert.Len(t, events, 1)
    assert.IsType(t, &RideRequested{}, events[0])
    
    // Driver accepts
    resp, _ = gateway.Post(fmt.Sprintf("/driver/rides/%s/accept", rideID),
        AcceptRideRequest{DriverID: driverID})
    assert.Equal(t, http.StatusOK, resp.StatusCode)
    
    // Verify CQRS view
    assert.Eventually(t, func() bool {
        view, err := rideViewRepo.FindByID(ctx, rideID)
        return err == nil && view.DriverName != ""
    }, 10*time.Second, 500*time.Millisecond)
    
    // Complete ride
    resp, _ = gateway.Post(fmt.Sprintf("/driver/rides/%s/complete", rideID),
        CompleteRideRequest{DriverID: driverID, Distance: 5.2, Duration: 15})
    assert.Equal(t, http.StatusOK, resp.StatusCode)
    
    // Verify full event stream
    finalEvents, _ := eventStore.GetEvents(ctx, rideID)
    assert.Len(t, finalEvents, 4) // Requested, Matched, Started, Completed
}

func TestCircuitBreakerOnPaymentTimeout(t *testing.T) {
    // Mock: payment responds in 10s
    paymentMock := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(10 * time.Second)
        w.WriteHeader(http.StatusOK)
    }))
    defer paymentMock.Close()
    
    resp, _ := gateway.Post("/passenger/rides", createRideRequest)
    assert.Equal(t, http.StatusCreated, resp.StatusCode)
    
    // CB should be OPEN
    assert.Eventually(t, func() bool {
        return paymentCB.State() == gobreaker.StateOpen
    }, 30*time.Second, 1*time.Second)
}

func TestIdempotency(t *testing.T) {
    key := uuid.New().String()
    
    resp1, _ := gateway.Post("/passenger/rides", req, WithHeader("Idempotency-Key", key))
    resp2, _ := gateway.Post("/passenger/rides", req, WithHeader("Idempotency-Key", key))
    
    assert.Equal(t, http.StatusCreated, resp1.StatusCode)
    assert.Equal(t, http.StatusOK, resp2.StatusCode)
    
    var r1, r2 CreateRideResponse
    json.NewDecoder(resp1.Body).Decode(&r1)
    json.NewDecoder(resp2.Body).Decode(&r2)
    assert.Equal(t, r1.RideID, r2.RideID)
}
```

**Critérios de aceite:**
- [ ] Happy path: corrida criada → matched → started → completed (todos os events no event store)
- [ ] Saga orquestrada: 4 steps executados na ordem correta
- [ ] Outbox: eventos publicados via transactional outbox (nenhum dual write)
- [ ] CQRS: ride_details_view atualizado após cada evento
- [ ] Circuit Breaker: CLOSED → OPEN quando Payment falha, HALF_OPEN → CLOSED quando recupera
- [ ] Retry: 3 tentativas com exponential backoff para Payment
- [ ] DLQ: notificação falha → vai para DLQ → reprocessada com sucesso
- [ ] Idempotência: mesma Idempotency-Key retorna resultado idêntico sem duplicar
- [ ] Backpressure: 50K events/s → buffer cheio → drop oldest + HTTP 429
- [ ] Observabilidade: trace no Jaeger mostrando jornada completa (todos os serviços)

---

### Desafio 9.3 — Chaos Day: Resiliência e SLOs sob Stress

Simule um "chaos day" com falhas simultâneas e verifique que os SLOs são mantidos.

**Cenário:** O CTO quer prova de que a plataforma aguenta falhas de infra sem violar SLOs. O time SRE executa um script de chaos que injeta falhas durante 30 minutos enquanto tráfego real (simulado) continua.

**Requisitos:**

**Load Generation (k6 ou script):**
- 100 requests/segundo de criação de corridas (distribuídas: SP 40%, RJ 30%, BH 20%, CWB 10%)
- 50K location updates/segundo (GPS de motoristas ativos)
- 20 requests/segundo de consulta de corridas ativas (API Composition)
- Duração: 30 minutos

**Chaos Script (falhas injetadas):**
```
Minuto 00-05: Warm-up (nenhuma falha)
Minuto 05-10: Payment Service lento (P99 = 8s) → esperar CB open
Minuto 10-15: Driver Service down (1 de 2 réplicas) → esperar load balance
Minuto 15-20: Kafka broker-2 down → esperar partition rebalance
Minuto 20-25: PostgreSQL ride-db slow queries (lock simulation)
Minuto 25-30: Recovery (todas as falhas removidas)
```

**SLO Dashboard durante chaos:**
- Availability SLO (99.9%): requests com status < 500 / total requests
- Latency SLO (99.5% < 2s): criação de corridas < 2s
- Matching SLO (99.0%): corridas matched em < 30s
- Error Budget: consumed vs remaining (burn rate)

**Relatório Final (gerado automaticamente):**
```markdown
## RideFlow Chaos Day Report — 2025-01-15

### SLO Compliance
| SLO | Target | Actual | Status |
|---|---|---|---|
| Availability | 99.9% | 99.85% | ⚠️ WARN |
| Ride Creation Latency | 99.5% < 2s | 99.2% | ⚠️ WARN |
| Matching Success | 99.0% | 99.5% | ✅ OK |

### Error Budget Impact
- Availability: consumed 15% of monthly budget in 30min
- Burn rate peak: 43x at minute 12 (driver-service down)

### Incident Timeline
| Time | Event | Impact | Recovery |
|---|---|---|---|
| 05:00 | Payment slow (8s P99) | CB opened in 15s | 3 rides queued |
| 10:00 | Driver instance down | 2s routing to healthy | Consul detected in 10s |
| 15:00 | Kafka broker-2 down | Consumer lag spike 5K | Rebalanced in 45s |
| 20:00 | PostgreSQL slow queries | P99 jumped 500ms→3s | Timeouts kicked in |
| 25:00 | All faults cleared | Full recovery in 60s | SLIs normalized |

### Pattern Effectiveness
| Pattern | Triggered | Behavior | Verdict |
|---|---|---|---|
| Circuit Breaker | ✅ | OPEN at 05:15, HALF_OPEN at 10:30 | Effective |
| Retry | ✅ | 3x with backoff, 85% recovered | Effective |
| Bulkhead | ✅ | Payment isolated, other services unaffected | Effective |
| DLQ | ✅ | 12 notifications queued, 12 reprocessed | Effective |
| Backpressure | ✅ | Buffer peaked 85%, 2% dropped | Effective |
| Fallback | ✅ | Cached driver data served for 5min | Effective |
| Service Discovery | ✅ | Unhealthy instance removed in 10s | Effective |
```

**Java 25 / Go 1.26 — Chaos Test Runner:**
```java
@Test
@Timeout(value = 35, unit = TimeUnit.MINUTES)
void chaosDaySimulation() {
    // Start load generation
    var loadGenerator = new LoadGenerator(gatewayUrl, 100, 50_000, 20);
    loadGenerator.startAsync();
    
    // Phase: Warm-up
    sleep(5, MINUTES);
    assertSLOs("warm-up");
    
    // Phase: Payment slow
    paymentProxy.addLatency(Duration.ofSeconds(8));
    sleep(5, MINUTES);
    assertCircuitBreakerOpen("payment-service");
    assertSLOsWithTolerance("payment-slow", 0.005); // allow 0.5% degradation
    paymentProxy.removeLatency();
    
    // Phase: Driver down
    driverContainer.stop();
    sleep(5, MINUTES);
    assertConsulHealthCheckFailed("driver-service");
    assertFallbackServing("driver-service");
    driverContainer.start();
    await().atMost(30, SECONDS).until(() -> consulHealthy("driver-service"));
    
    // Phase: Kafka broker down
    kafkaBroker2.stop();
    sleep(5, MINUTES);
    assertKafkaRebalanced();
    kafkaBroker2.start();
    
    // Phase: DB slow
    postgresRide.exec("SELECT pg_sleep(2)"); // simulate slow queries
    sleep(5, MINUTES);
    
    // Phase: Recovery
    sleep(5, MINUTES);
    
    // Generate report
    loadGenerator.stop();
    var report = sloReporter.generateReport(loadGenerator.getMetrics());
    
    // Assert SLOs
    assertThat(report.availabilitySLI()).isGreaterThan(0.998); // close to 99.9%
    assertThat(report.latencySLI()).isGreaterThan(0.990);
    assertThat(report.matchingSLI()).isGreaterThan(0.990);
    
    // Write report to file
    Files.writeString(Path.of("chaos-report.md"), report.toMarkdown());
}
```

```go
func TestChaosDay(t *testing.T) {
    if testing.Short() {
        t.Skip("chaos day requires full infrastructure")
    }
    
    ctx, cancel := context.WithTimeout(context.Background(), 35*time.Minute)
    defer cancel()
    
    // Start load
    lg := NewLoadGenerator(gatewayURL, LoadConfig{
        RideCreationRPS:  100,
        LocationEventsPS: 50_000,
        QueryRPS:         20,
    })
    go lg.Run(ctx)
    
    // Warm-up
    time.Sleep(5 * time.Minute)
    assertSLOs(t, lg.Metrics(), "warm-up")
    
    // Payment slow
    paymentProxy.AddLatency(8 * time.Second)
    time.Sleep(5 * time.Minute)
    assert.Equal(t, gobreaker.StateOpen, paymentCB.State())
    paymentProxy.RemoveLatency()
    
    // Driver down
    driverContainer.Stop(ctx)
    time.Sleep(5 * time.Minute)
    driverContainer.Start(ctx)
    
    // Kafka broker down
    kafkaBroker2.Stop(ctx)
    time.Sleep(5 * time.Minute)
    kafkaBroker2.Start(ctx)
    
    // DB slow
    _, _ = pgPool.Exec(ctx, "SELECT pg_sleep(2)")
    time.Sleep(5 * time.Minute)
    
    // Recovery
    time.Sleep(5 * time.Minute)
    
    // Report
    cancel()
    report := GenerateChaosReport(lg.Metrics())
    
    assert.Greater(t, report.AvailabilitySLI, 0.998)
    assert.Greater(t, report.LatencySLI, 0.990)
    assert.Greater(t, report.MatchingSLI, 0.990)
    
    os.WriteFile("chaos-report.md", []byte(report.ToMarkdown()), 0644)
}
```

**Critérios de aceite:**
- [ ] Load generation: 100 rides/s + 50K locations/s + 20 queries/s durante 30 min
- [ ] Chaos script: 5 fases de falha executadas na sequência correta
- [ ] SLOs mantidos: availability > 99.8%, latency > 99.0%, matching > 99.0%
- [ ] Cada pattern demonstrou efetividade (CB opened, retry recovered, DLQ captured, etc.)
- [ ] Relatório gerado automaticamente em Markdown
- [ ] Error budget burn rate calculado para cada fase
- [ ] Recovery: sistema volta ao normal em < 60s após falhas removidas
- [ ] Todos os traces no Jaeger correlacionam a jornada completa durante chaos
- [ ] Métricas no Prometheus refletem cada fase (spikes de latência, error rate, consumer lag)
- [ ] Zero dados perdidos: todas as corridas criadas durante chaos são recuperáveis

---

## Entrega Final — Portfolio

Ao completar o Capstone, o engenheiro terá:

| Artefato | Descrição |
|---|---|
| **Código fonte** | 8+ microserviços Java 25 ou Go 1.26 com 27 patterns implementados |
| **Docker Compose** | Infraestrutura completa (15+ containers) com startup ordering |
| **Testes** | Integration tests com Testcontainers cobrindo happy path + falhas |
| **Chaos Report** | Relatório markdown do chaos day com SLOs, timeline, pattern effectiveness |
| **Architecture Decision Records** | ADRs para decisões de cada pattern (por que saga orquestrada vs coreografada, etc.) |
| **Observability Dashboard** | Prometheus queries + Jaeger traces demonstrando os 3 pilares |

> **Parabéns!** 🎉 Ao completar todos os 10 levels, você domina os patterns fundamentais de arquitetura
> de microserviços — da teoria à implementação sob falhas reais.
