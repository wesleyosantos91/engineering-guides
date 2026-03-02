# Level 4 — Data Patterns: API Composition, CQRS & Event Sourcing

> **Objetivo:** Dominar padrões de dados distribuídos — agregar informações de múltiplos serviços com API Composition,
> separar modelos de leitura e escrita com CQRS, e usar eventos como fonte de verdade com Event Sourcing.

**Pré-requisito:** [Level 3 — Consistência de Dados](03-data-consistency-idempotency-dlq-saga.md)

**Referências:**
- [10-api-composition.md](../../.docs/microservice-patterns/10-api-composition.md)
- [11-cqrs.md](../../.docs/microservice-patterns/11-cqrs.md)
- [12-event-sourcing.md](../../.docs/microservice-patterns/12-event-sourcing.md)

---

## Contexto do Domínio

No RideFlow, os dados estão distribuídos entre microsserviços:

- **API Composition** — Tela "Detalhes da Corrida" precisa agregar dados de Ride + Driver + Payment + Location
- **CQRS** — Write: processar novas corridas (otimizado para consistência) / Read: histórico e dashboard (otimizado para queries)
- **Event Sourcing** — O ciclo de vida da corrida (requested → matched → started → completed) é um stream de eventos imutáveis

---

## Desafios

### Desafio 4.1 — API Composition: Tela de Detalhes da Corrida

Implemente um compositor que agrega dados de 4 serviços para montar a tela de detalhes da corrida.

**Cenário:** O app do passageiro mostra uma tela "Detalhes da Corrida" com informações de múltiplos serviços. O compositor (BFF ou service dedicado) chama os serviços em paralelo e agrega as respostas.

**Requisitos:**
- Compositor centralizado (`RideDetailsCompositor`) chama 4 serviços:
  1. Ride Service → dados da corrida (origem, destino, status, timestamps)
  2. Driver Service → dados do motorista (nome, foto, rating, placa)
  3. Payment Service → dados do pagamento (valor, método, status)
  4. Location Service → posição atual do motorista (lat/lng em tempo real)
- Chamadas em **paralelo** (latência = max das chamadas, não soma)
- **Partial failure handling:** se Location Service falhar, retornar dados sem localização (graceful degradation)
- **Timeout total:** 5s para a composição inteira
- Evitar **N+1 problem:** ao listar corridas, usar batch endpoint
- Cache de dados estáveis (dados do motorista: TTL 5min) vs dados voláteis (localização: sem cache)

**Java 25 (CompletableFuture paralelo):**
```java
@Service
public class RideDetailsCompositor {
    
    public RideDetailsDTO compose(UUID rideId) {
        var rideF = CompletableFuture.supplyAsync(
            () -> rideClient.getRide(rideId));
        var driverF = CompletableFuture.supplyAsync(
            () -> driverClient.getDriver(ride.driverId()));
        var paymentF = CompletableFuture.supplyAsync(
            () -> paymentClient.getPayment(rideId));
        var locationF = CompletableFuture.supplyAsync(
            () -> locationClient.getDriverLocation(ride.driverId()));
        
        try {
            var ride = rideF.get(5, TimeUnit.SECONDS);
            
            // Precisamos do ride para saber o driverId
            var driverFuture = CompletableFuture.supplyAsync(
                () -> driverClient.getDriver(ride.driverId()));
            
            var builder = RideDetailsDTO.builder().ride(ride);
            
            // Agregar em paralelo com graceful degradation
            try {
                builder.driver(driverFuture.get(3, TimeUnit.SECONDS));
            } catch (Exception e) {
                log.warn("Driver data unavailable: {}", e.getMessage());
                builder.driver(DriverDTO.unavailable());
            }
            
            try {
                builder.payment(paymentF.get(3, TimeUnit.SECONDS));
            } catch (Exception e) {
                log.warn("Payment data unavailable: {}", e.getMessage());
                builder.payment(PaymentDTO.unavailable());
            }
            
            try {
                builder.location(locationF.get(2, TimeUnit.SECONDS));
            } catch (Exception e) {
                log.warn("Location data unavailable: {}", e.getMessage());
                // Localização é volátil — omitir silenciosamente
            }
            
            return builder.build();
            
        } catch (TimeoutException e) {
            throw new CompositionTimeoutException("Ride details composition timed out");
        }
    }
}
```

**Go 1.26 (errgroup paralelo):**
```go
func (c *RideDetailsCompositor) Compose(
    ctx context.Context, rideID uuid.UUID,
) (*RideDetailsDTO, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    // Primeiro: buscar dados da corrida (precisamos do driverID)
    ride, err := c.rideClient.GetRide(ctx, rideID)
    if err != nil {
        return nil, fmt.Errorf("fetching ride: %w", err)
    }
    
    result := &RideDetailsDTO{Ride: ride}
    var mu sync.Mutex
    
    g, ctx := errgroup.WithContext(ctx)
    
    // Paralelo: driver, payment, location
    g.Go(func() error {
        driver, err := c.driverClient.GetDriver(ctx, *ride.DriverID)
        if err != nil {
            slog.Warn("driver data unavailable", "error", err.Error())
            return nil // graceful degradation
        }
        mu.Lock()
        result.Driver = driver
        mu.Unlock()
        return nil
    })
    
    g.Go(func() error {
        payment, err := c.paymentClient.GetPayment(ctx, rideID)
        if err != nil {
            slog.Warn("payment data unavailable", "error", err.Error())
            return nil // graceful degradation
        }
        mu.Lock()
        result.Payment = payment
        mu.Unlock()
        return nil
    })
    
    g.Go(func() error {
        if ride.DriverID == nil {
            return nil
        }
        loc, err := c.locationClient.GetDriverLocation(ctx, *ride.DriverID)
        if err != nil {
            slog.Warn("location unavailable", "error", err.Error())
            return nil // volátil — omitir
        }
        mu.Lock()
        result.Location = loc
        mu.Unlock()
        return nil
    })
    
    if err := g.Wait(); err != nil {
        return nil, fmt.Errorf("composing ride details: %w", err)
    }
    
    return result, nil
}
```

**Critérios de aceite:**
- [ ] Compositor agrega dados de 4 serviços em uma resposta
- [ ] Chamadas em paralelo (latência total ≈ max das individuais)
- [ ] Partial failure: Location falha → resposta sem campo location (não erro 500)
- [ ] Partial failure: Payment falha → resposta com `payment: { status: "unavailable" }`
- [ ] Timeout total de 5s para composição inteira
- [ ] Batch endpoint para listagem (evitar N+1)
- [ ] Cache para dados estáveis (driver: 5min) vs dados voláteis (location: sem cache)
- [ ] Testes: todos OK; 1 falha → degradação graciosa; timeout → falha rápida

---

### Desafio 4.2 — CQRS: Separação de Modelos de Leitura e Escrita

Implemente CQRS com modelos separados para comando (processar corridas) e query (consultar histórico).

**Cenário:** O Ride Service tem padrões de acesso muito diferentes:
- **Write (Command):** criar corrida, atualizar status — precisa de ACID, normalizado, integridade referencial
- **Read (Query):** listar corridas do passageiro, dashboard analytics — precisa de queries rápidas, desnormalizado

**Requisitos:**
- **Command Model** (PostgreSQL normalizado):
  - Tabelas: `rides`, `ride_status_history`, `ride_events`
  - Operações: `CreateRide`, `UpdateRideStatus`, `AssignDriver`
  - Validação rigorosa, transações ACID
- **Query Model** (desnormalizado, otimizado para leitura):
  - View materializada ou tabela: `ride_details_view`
  - Campos: ride_id, passenger_name, driver_name, driver_plate, origin_address, destination_address, price, status, created_at, completed_at, duration_minutes, distance_km
  - Índices otimizados para queries frequentes (por passageiro, por data, por status)
- **Sincronização** via eventos internos:
  - Command publica evento → handler atualiza query model
  - Consistência eventual (aceitar delay de 1-2 segundos)
- **Nível de CQRS:** modelos separados, mesmo banco de dados (nível intermediário)

**Java 25 (Spring CQRS):**
```java
// Command Model
@Entity
@Table(name = "rides")
public class Ride {
    @Id private UUID id;
    private UUID passengerId;
    private UUID driverId;
    @Enumerated(EnumType.STRING)
    private RideStatus status;
    private BigDecimal price;
    // Normalizado, sem dados redundantes
}

// Command Handler
@Service
public class RideCommandService {
    
    @Transactional
    public UUID createRide(CreateRideCommand cmd) {
        var ride = new Ride(cmd.passengerId(), cmd.origin(), cmd.destination());
        rideRepo.save(ride);
        
        // Publicar evento para sincronizar query model
        eventPublisher.publish(new RideCreatedEvent(
            ride.getId(), cmd.passengerId(), ride.getStatus()));
        
        return ride.getId();
    }
    
    @Transactional
    public void updateStatus(UpdateStatusCommand cmd) {
        var ride = rideRepo.findById(cmd.rideId()).orElseThrow();
        ride.transitionTo(cmd.newStatus());
        rideRepo.save(ride);
        
        eventPublisher.publish(new RideStatusChangedEvent(
            ride.getId(), cmd.newStatus(), Instant.now()));
    }
}

// Query Model (desnormalizado)
@Entity
@Table(name = "ride_details_view")
public class RideDetailsView {
    @Id private UUID rideId;
    private String passengerName;
    private String driverName;
    private String driverPlate;
    private String originAddress;
    private String destinationAddress;
    private BigDecimal price;
    private String status;
    private Instant createdAt;
    private Instant completedAt;
    private Integer durationMinutes;
    private Double distanceKm;
}

// Query Handler
@Service
public class RideQueryService {
    
    public Page<RideDetailsView> getPassengerHistory(UUID passengerId, Pageable pageable) {
        return rideViewRepo.findByPassengerIdOrderByCreatedAtDesc(passengerId, pageable);
    }
    
    public DashboardDTO getDashboard(LocalDate from, LocalDate to) {
        return new DashboardDTO(
            rideViewRepo.countByDateRange(from, to),
            rideViewRepo.avgDurationByDateRange(from, to),
            rideViewRepo.totalRevenueByDateRange(from, to)
        );
    }
}

// Synchronizer: Event → Query Model
@Component
public class RideViewProjection {
    
    @EventListener
    public void onRideCreated(RideCreatedEvent event) {
        var passenger = passengerClient.getPassenger(event.passengerId());
        var view = new RideDetailsView();
        view.setRideId(event.rideId());
        view.setPassengerName(passenger.name());
        view.setStatus(event.status().name());
        view.setCreatedAt(event.occurredAt());
        rideViewRepo.save(view);
    }
    
    @EventListener
    public void onRideStatusChanged(RideStatusChangedEvent event) {
        var view = rideViewRepo.findById(event.rideId()).orElseThrow();
        view.setStatus(event.newStatus().name());
        if (event.newStatus() == RideStatus.COMPLETED) {
            view.setCompletedAt(event.occurredAt());
            view.setDurationMinutes(calculateDuration(view));
        }
        rideViewRepo.save(view);
    }
}
```

**Go 1.26:**
```go
// Command Model
type RideCommandService struct {
    db        *sqlx.DB
    publisher EventPublisher
}

func (s *RideCommandService) CreateRide(ctx context.Context, cmd CreateRideCommand) (uuid.UUID, error) {
    tx, err := s.db.BeginTxx(ctx, nil)
    if err != nil {
        return uuid.Nil, err
    }
    defer tx.Rollback()
    
    ride := &Ride{
        ID:          uuid.New(),
        PassengerID: cmd.PassengerID,
        Origin:      cmd.Origin,
        Destination: cmd.Destination,
        Status:      RideStatusRequested,
        CreatedAt:   time.Now(),
    }
    
    _, err = tx.NamedExecContext(ctx,
        `INSERT INTO rides (id, passenger_id, origin_lat, origin_lng, dest_lat, dest_lng, status, created_at)
         VALUES (:id, :passenger_id, :origin_lat, :origin_lng, :dest_lat, :dest_lng, :status, :created_at)`, ride)
    if err != nil {
        return uuid.Nil, fmt.Errorf("insert ride: %w", err)
    }
    
    if err := tx.Commit(); err != nil {
        return uuid.Nil, err
    }
    
    // Publicar evento (fora da transação neste nível; outbox no nível 5)
    s.publisher.Publish("ride-events", &RideCreatedEvent{
        RideID:      ride.ID,
        PassengerID: ride.PassengerID,
        Status:      string(ride.Status),
        OccurredAt:  ride.CreatedAt,
    })
    
    return ride.ID, nil
}

// Query Model (desnormalizado)
type RideQueryService struct {
    db *sqlx.DB
}

func (s *RideQueryService) GetPassengerHistory(
    ctx context.Context, passengerID uuid.UUID, page, size int,
) ([]RideDetailsView, error) {
    var views []RideDetailsView
    err := s.db.SelectContext(ctx, &views,
        `SELECT * FROM ride_details_view 
         WHERE passenger_id = $1 
         ORDER BY created_at DESC 
         LIMIT $2 OFFSET $3`,
        passengerID, size, page*size)
    return views, err
}

// Projection: Event → Query Model
type RideViewProjection struct {
    db             *sqlx.DB
    passengerClient *PassengerServiceClient
}

func (p *RideViewProjection) HandleRideCreated(msg *message.Message) error {
    var event RideCreatedEvent
    json.Unmarshal(msg.Payload, &event)
    
    passenger, _ := p.passengerClient.GetPassenger(msg.Context(), event.PassengerID)
    
    _, err := p.db.ExecContext(msg.Context(),
        `INSERT INTO ride_details_view (ride_id, passenger_name, status, created_at)
         VALUES ($1, $2, $3, $4)
         ON CONFLICT (ride_id) DO UPDATE SET status = $3`,
        event.RideID, passenger.Name, event.Status, event.OccurredAt)
    
    return err
}
```

**Critérios de aceite:**
- [ ] Command model normalizado com validação/transações ACID
- [ ] Query model desnormalizado otimizado para leitura (view materializada)
- [ ] Sincronização via eventos internos (eventual consistency)
- [ ] Queries do dashboard usam query model (não fazem JOINs no command model)
- [ ] Índices otimizados por passengerId, por data, por status
- [ ] Testes: criar corrida (command) → verificar query model atualizado
- [ ] Documentar trade-off: consistência eventual vs queries rápidas

---

### Desafio 4.3 — Event Sourcing: Ciclo de Vida da Corrida como Stream de Eventos

Implemente Event Sourcing para armazenar o ciclo de vida da corrida como sequência de eventos imutáveis.

**Cenário:** Em vez de armazenar apenas o estado atual da corrida (COMPLETED), armazenamos TODOS os eventos que levaram a esse estado. Isso permite auditar, reconstruir e fazer consultas temporais.

**Requisitos:**
- **Event Store** (tabela append-only) com eventos:
  - `RideRequested`, `DriverMatched`, `DriverEnRoute`, `RideStarted`, `RideCompleted`, `RideCancelled`
- Cada evento: `event_id`, `aggregate_id` (ride_id), `aggregate_type`, `event_type`, `version`, `payload`, `occurred_at`
- **Replay:** reconstruir o estado atual da corrida a partir dos eventos
- **Versão otimista:** detectar conflitos de concorrência via version do aggregate
- **Snapshots:** a cada 10 eventos, salvar snapshot para performance de replay
- Eventos nomeados no **passado** (RideRequested, não RequestRide)
- Combinar com CQRS: event store é a fonte de verdade, query model é projeção

**Java 25:**
```java
// Event Store
@Entity
@Table(name = "ride_events")
public class StoredEvent {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long eventId;
    
    private UUID aggregateId;
    private String aggregateType;
    private String eventType;
    private int version;
    
    @Column(columnDefinition = "jsonb")
    private String payload;
    
    private Instant occurredAt;
}

// Aggregate
public sealed interface RideEvent permits
        RideRequested, DriverMatched, DriverEnRoute,
        RideStarted, RideCompleted, RideCancelled {
    UUID rideId();
    Instant occurredAt();
}

public record RideRequested(UUID rideId, UUID passengerId,
        GeoPoint origin, GeoPoint destination, Instant occurredAt) implements RideEvent {}

public record DriverMatched(UUID rideId, UUID driverId,
        String driverName, Instant occurredAt) implements RideEvent {}

// Aggregate Root
public class RideAggregate {
    private UUID id;
    private RideStatus status;
    private UUID passengerId;
    private UUID driverId;
    private BigDecimal price;
    private int version;
    
    private final List<RideEvent> uncommittedEvents = new ArrayList<>();
    
    // Factory: criar novo aggregate
    public static RideAggregate requestRide(UUID passengerId, GeoPoint origin, GeoPoint dest) {
        var agg = new RideAggregate();
        agg.apply(new RideRequested(UUID.randomUUID(), passengerId, origin, dest, Instant.now()));
        return agg;
    }
    
    // Command: aceitar motorista
    public void matchDriver(UUID driverId, String driverName) {
        if (status != RideStatus.REQUESTED) {
            throw new IllegalStateException("Cannot match driver in state " + status);
        }
        apply(new DriverMatched(id, driverId, driverName, Instant.now()));
    }
    
    // Aplicar evento (muda estado + marca como uncommitted)
    private void apply(RideEvent event) {
        when(event); // atualiza estado
        uncommittedEvents.add(event);
        version++;
    }
    
    // Replay: reconstruir estado a partir de eventos
    public static RideAggregate replay(List<RideEvent> events) {
        var agg = new RideAggregate();
        for (var event : events) {
            agg.when(event);
            agg.version++;
        }
        return agg;
    }
    
    // Event handler: atualiza estado sem side-effects
    private void when(RideEvent event) {
        switch (event) {
            case RideRequested e -> {
                this.id = e.rideId();
                this.passengerId = e.passengerId();
                this.status = RideStatus.REQUESTED;
            }
            case DriverMatched e -> {
                this.driverId = e.driverId();
                this.status = RideStatus.MATCHED;
            }
            case RideStarted e -> this.status = RideStatus.IN_PROGRESS;
            case RideCompleted e -> {
                this.status = RideStatus.COMPLETED;
                this.price = e.finalPrice();
            }
            case RideCancelled e -> this.status = RideStatus.CANCELLED;
            default -> throw new IllegalArgumentException("Unknown event: " + event);
        }
    }
}

// Event Store Repository
@Service
public class RideEventStore {
    
    @Transactional
    public void save(RideAggregate aggregate) {
        var events = aggregate.getUncommittedEvents();
        int expectedVersion = aggregate.getVersion() - events.size();
        
        // Optimistic concurrency check
        int currentVersion = eventRepo.getLatestVersion(aggregate.getId());
        if (currentVersion != expectedVersion) {
            throw new OptimisticConcurrencyException(
                "Expected version %d, found %d".formatted(expectedVersion, currentVersion));
        }
        
        for (var event : events) {
            eventRepo.save(new StoredEvent(
                aggregate.getId(), "Ride", event.getClass().getSimpleName(),
                ++expectedVersion, serialize(event), event.occurredAt()
            ));
        }
        
        // Snapshot a cada 10 eventos
        if (aggregate.getVersion() % 10 == 0) {
            snapshotRepo.save(new Snapshot(
                aggregate.getId(), aggregate.getVersion(), serialize(aggregate)));
        }
    }
    
    public RideAggregate load(UUID rideId) {
        // Tentar carregar do snapshot
        var snapshot = snapshotRepo.findLatest(rideId);
        
        List<StoredEvent> events;
        if (snapshot.isPresent()) {
            var agg = deserialize(snapshot.get());
            events = eventRepo.findByAggregateIdAndVersionGreaterThan(
                rideId, snapshot.get().getVersion());
            // Replay apenas eventos após o snapshot
            for (var event : events) {
                agg.when(deserialize(event));
            }
            return agg;
        }
        
        events = eventRepo.findByAggregateIdOrderByVersion(rideId);
        return RideAggregate.replay(events.stream()
            .map(this::deserialize)
            .toList());
    }
}
```

**Go 1.26:**
```go
type RideEvent interface {
    EventType() string
    RideID() uuid.UUID
    OccurredAt() time.Time
}

type RideRequested struct {
    ID          uuid.UUID `json:"rideId"`
    PassengerID uuid.UUID `json:"passengerId"`
    Origin      GeoPoint  `json:"origin"`
    Destination GeoPoint  `json:"destination"`
    Timestamp   time.Time `json:"occurredAt"`
}

func (e RideRequested) EventType() string    { return "RideRequested" }
func (e RideRequested) RideID() uuid.UUID    { return e.ID }
func (e RideRequested) OccurredAt() time.Time { return e.Timestamp }

type RideAggregate struct {
    ID          uuid.UUID
    Status      RideStatus
    PassengerID uuid.UUID
    DriverID    *uuid.UUID
    Price       *float64
    Version     int
    
    uncommitted []RideEvent
}

func (a *RideAggregate) Apply(event RideEvent) {
    a.When(event)
    a.uncommitted = append(a.uncommitted, event)
    a.Version++
}

func (a *RideAggregate) When(event RideEvent) {
    switch e := event.(type) {
    case RideRequested:
        a.ID = e.ID
        a.PassengerID = e.PassengerID
        a.Status = RideStatusRequested
    case DriverMatched:
        a.DriverID = &e.DriverID
        a.Status = RideStatusMatched
    case RideStarted:
        a.Status = RideStatusInProgress
    case RideCompleted:
        a.Status = RideStatusCompleted
        a.Price = &e.FinalPrice
    case RideCancelled:
        a.Status = RideStatusCancelled
    }
}

func ReplayAggregate(events []RideEvent) *RideAggregate {
    agg := &RideAggregate{}
    for _, event := range events {
        agg.When(event)
        agg.Version++
    }
    return agg
}

// Event Store
type EventStore struct {
    db *sqlx.DB
}

func (s *EventStore) Save(ctx context.Context, agg *RideAggregate) error {
    tx, err := s.db.BeginTxx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // Optimistic concurrency
    var currentVersion int
    tx.GetContext(ctx, &currentVersion,
        "SELECT COALESCE(MAX(version), 0) FROM ride_events WHERE aggregate_id = $1",
        agg.ID)
    
    expectedVersion := agg.Version - len(agg.uncommitted)
    if currentVersion != expectedVersion {
        return fmt.Errorf("optimistic concurrency: expected %d, got %d",
            expectedVersion, currentVersion)
    }
    
    version := expectedVersion
    for _, event := range agg.uncommitted {
        version++
        payload, _ := json.Marshal(event)
        _, err := tx.ExecContext(ctx,
            `INSERT INTO ride_events (aggregate_id, aggregate_type, event_type, version, payload, occurred_at)
             VALUES ($1, 'Ride', $2, $3, $4, $5)`,
            agg.ID, event.EventType(), version, payload, event.OccurredAt())
        if err != nil {
            return fmt.Errorf("insert event: %w", err)
        }
    }
    
    // Snapshot a cada 10 eventos
    if agg.Version%10 == 0 {
        snapshot, _ := json.Marshal(agg)
        tx.ExecContext(ctx,
            `INSERT INTO ride_snapshots (aggregate_id, version, data, created_at)
             VALUES ($1, $2, $3, NOW())
             ON CONFLICT (aggregate_id) DO UPDATE SET version = $2, data = $3`,
            agg.ID, agg.Version, snapshot)
    }
    
    return tx.Commit()
}
```

**Critérios de aceite:**
- [ ] Event Store append-only com eventos imutáveis (RideRequested, DriverMatched, etc.)
- [ ] Replay: reconstruir estado do aggregate a partir de eventos
- [ ] Optimistic concurrency: detectar conflitos via version
- [ ] Snapshots a cada 10 eventos (performance de replay)
- [ ] Eventos nomeados no passado (RideRequested, não RequestRide)
- [ ] Temporal queries: "qual era o estado da corrida às 14:30?"
- [ ] Integração ES + CQRS: eventos alimentam o query model
- [ ] Testes: criar corrida com 5 eventos → replay → estado correto; conflito de version → erro

---

### Desafio 4.4 — Projeções e Temporal Queries

Crie projeções especializadas a partir do event store e implemente consultas temporais.

**Cenário:** Com o event store como fonte de verdade, podemos criar múltiplas projeções para diferentes necessidades:
1. **Projeção de histórico** — timeline visual da corrida
2. **Projeção de analytics** — métricas agregadas (tempo médio, corridas por hora)
3. **Temporal query** — "qual era o estado da corrida X às 14:30?"

**Requisitos:**
- Projeção de **timeline** da corrida (todos os status com timestamps formatados)
- Projeção de **analytics diário** (total corridas, receita, tempo médio, por categoria)
- Temporal query: dado um `rideId` e `timestamp`, reconstruir o estado naquele momento
- Projeções são **rebuild-able**: deletar e reconstruir do zero a partir do event store
- Cada projeção roda como consumer independente (pode estar atrasada sem afetar outros)

**Java 25:**
```java
// Temporal Query: estado no tempo
public RideState getStateAt(UUID rideId, Instant pointInTime) {
    var events = eventStore.findByAggregateIdAndOccurredAtBefore(rideId, pointInTime);
    return RideAggregate.replay(events).toState();
}

// Projeção de Analytics
@Component
public class RideAnalyticsProjection {
    
    @EventListener
    public void onRideCompleted(RideCompletedEvent event) {
        var date = event.occurredAt().atZone(ZoneId.systemDefault()).toLocalDate();
        
        analyticsRepo.upsertDailyStats(date, stats -> {
            stats.incrementTotalRides();
            stats.addRevenue(event.finalPrice());
            stats.addDuration(event.durationMinutes());
            stats.incrementCategory(event.vehicleCategory());
        });
    }
    
    // Rebuild completo da projeção
    public void rebuild() {
        analyticsRepo.deleteAll();
        var allEvents = eventStore.findAllByType("RideCompleted");
        allEvents.forEach(this::onRideCompleted);
    }
}
```

**Critérios de aceite:**
- [ ] Projeção de timeline: lista ordenada de eventos com timestamps formatados
- [ ] Projeção de analytics: total corridas, receita, tempo médio por dia/categoria
- [ ] Temporal query funcional: `getStateAt(rideId, timestamp)` retorna estado correto
- [ ] Projeções são rebuild-able do zero (delete + replay all events)
- [ ] Cada projeção é independente (falha de uma não afeta outras)
- [ ] Testes: criar corrida com N eventos → temporal query no meio → estado intermediário correto
