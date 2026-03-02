# Level 5 — Event-Driven: Outbox, CDC, Backpressure & Sharding

> **Objetivo:** Garantir publicação confiável de eventos com Transactional Outbox, propagar mudanças via CDC,
> controlar fluxo com Backpressure, e escalar dados com Sharding/Partitioning.

**Pré-requisito:** [Level 4 — Data Patterns](04-data-patterns-composition-cqrs-eventsourcing.md)

**Referências:**
- [13-outbox-pattern.md](../../.docs/microservice-patterns/13-outbox-pattern.md)
- [14-cdc.md](../../.docs/microservice-patterns/14-cdc.md)
- [16-backpressure.md](../../.docs/microservice-patterns/16-backpressure.md)
- [15-sharding-partitioning.md](../../.docs/microservice-patterns/15-sharding-partitioning.md)

---

## Contexto do Domínio

No RideFlow, a comunicação entre serviços gera desafios de consistência e escala:

- **Outbox** — Ao criar uma corrida, salvar no banco E publicar evento precisa ser atômico (sem dual write)
- **CDC** — Sincronizar write DB → read DB (CQRS do Level 4) em tempo real via WAL/binlog
- **Backpressure** — Em horário de pico, o GPS envia 50K atualizações/s mas o Location Service processa 10K/s
- **Sharding** — Corridas particionadas por cidade/região para escalar horizontalmente

---

## Desafios

### Desafio 5.1 — Transactional Outbox: Publicação Confiável de Eventos

Elimine o problema de dual write implementando o Transactional Outbox Pattern.

**Cenário:** O Ride Service cria uma corrida (INSERT no banco) e precisa notificar outros serviços via Kafka. Se publicar diretamente no Kafka pode falhar (dual write). A solução é salvar o evento na **mesma transação** do dado, e um publisher separado ler e enviar.

**Requisitos:**
- Tabela `outbox_events` com: `id`, `aggregate_type`, `aggregate_id`, `event_type`, `payload` (JSONB), `destination` (tópico), `status` (PENDING/SENT/FAILED), `created_at`, `sent_at`, `retry_count`
- Save de negócio + evento na **mesma transação** (atômica):
  ```sql
  BEGIN;
    INSERT INTO rides (...) VALUES (...);
    INSERT INTO outbox_events (...) VALUES (...);
  COMMIT;
  ```
- **Outbox Publisher** (polling): processo que roda em loop
  1. SELECT PENDING events ORDER BY created_at LIMIT 100
  2. Publica no Kafka com `aggregate_id` como partition key (garantir ordem por corrida)
  3. Marca como SENT
  4. Retry com backoff para falhas transitórias; mark FAILED após 5 tentativas
- **Limpeza:** job periódico que deleta registros SENT com > 7 dias
- Consumer idempotente (at-least-once delivery da outbox)

**Java 25 (Spring Boot + Spring Kafka):**
```java
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id private UUID id;
    private String aggregateType;
    private UUID aggregateId;
    private String eventType;
    
    @Column(columnDefinition = "jsonb")
    private String payload;
    
    private String destination;
    
    @Enumerated(EnumType.STRING)
    private OutboxStatus status; // PENDING, SENT, FAILED
    
    private Instant createdAt;
    private Instant sentAt;
    private int retryCount;
}

@Service
public class RideService {
    
    @Transactional // Mesma transação!
    public UUID createRide(CreateRideCommand cmd) {
        var ride = new Ride(cmd.passengerId(), cmd.origin(), cmd.destination());
        rideRepo.save(ride);
        
        var outbox = OutboxEvent.builder()
            .id(UUID.randomUUID())
            .aggregateType("Ride")
            .aggregateId(ride.getId())
            .eventType("RideCreated")
            .payload(objectMapper.writeValueAsString(
                new RideCreatedPayload(ride.getId(), ride.getPassengerId(), 
                    ride.getStatus().name())))
            .destination("ride-events")
            .status(OutboxStatus.PENDING)
            .createdAt(Instant.now())
            .retryCount(0)
            .build();
        outboxRepo.save(outbox);
        
        return ride.getId();
    }
}

// Outbox Publisher — processo separado (scheduled)
@Component
public class OutboxPublisher {
    private static final int MAX_RETRIES = 5;
    
    @Scheduled(fixedDelay = 500) // poll a cada 500ms
    @Transactional
    public void pollAndPublish() {
        var events = outboxRepo.findByStatusOrderByCreatedAt(
            OutboxStatus.PENDING, Limit.of(100));
        
        for (var event : events) {
            try {
                kafkaTemplate.send(
                    event.getDestination(),
                    event.getAggregateId().toString(), // partition key
                    event.getPayload()
                ).get(3, TimeUnit.SECONDS); // síncrono para confirmar envio
                
                event.setStatus(OutboxStatus.SENT);
                event.setSentAt(Instant.now());
                
            } catch (Exception e) {
                event.setRetryCount(event.getRetryCount() + 1);
                if (event.getRetryCount() >= MAX_RETRIES) {
                    event.setStatus(OutboxStatus.FAILED);
                    log.error("Outbox event {} failed after {} retries",
                        event.getId(), MAX_RETRIES);
                }
            }
            outboxRepo.save(event);
        }
    }
}

// Job de limpeza
@Scheduled(cron = "0 0 3 * * *") // 3am todo dia
public void cleanupSentEvents() {
    var cutoff = Instant.now().minus(7, ChronoUnit.DAYS);
    int deleted = outboxRepo.deleteByStatusAndSentAtBefore(OutboxStatus.SENT, cutoff);
    log.info("Cleaned up {} sent outbox events", deleted);
}
```

**Go 1.26 (Watermill + pgx):**
```go
type OutboxEvent struct {
    ID            uuid.UUID      `db:"id"`
    AggregateType string         `db:"aggregate_type"`
    AggregateID   uuid.UUID      `db:"aggregate_id"`
    EventType     string         `db:"event_type"`
    Payload       json.RawMessage `db:"payload"`
    Destination   string         `db:"destination"`
    Status        string         `db:"status"` // PENDING, SENT, FAILED
    CreatedAt     time.Time      `db:"created_at"`
    SentAt        *time.Time     `db:"sent_at"`
    RetryCount    int            `db:"retry_count"`
}

func (s *RideService) CreateRide(ctx context.Context, cmd CreateRideCmd) (uuid.UUID, error) {
    tx, err := s.db.BeginTxx(ctx, nil)
    if err != nil {
        return uuid.Nil, err
    }
    defer tx.Rollback()
    
    ride := newRide(cmd)
    _, err = tx.NamedExecContext(ctx,
        `INSERT INTO rides (id, passenger_id, origin_lat, origin_lng, dest_lat, dest_lng, status)
         VALUES (:id, :passenger_id, :origin_lat, :origin_lng, :dest_lat, :dest_lng, :status)`, ride)
    if err != nil {
        return uuid.Nil, fmt.Errorf("insert ride: %w", err)
    }
    
    payload, _ := json.Marshal(RideCreatedPayload{
        RideID: ride.ID, PassengerID: ride.PassengerID, Status: string(ride.Status),
    })
    
    _, err = tx.ExecContext(ctx,
        `INSERT INTO outbox_events (id, aggregate_type, aggregate_id, event_type, payload, destination, status, created_at)
         VALUES ($1, 'Ride', $2, 'RideCreated', $3, 'ride-events', 'PENDING', NOW())`,
        uuid.New(), ride.ID, payload)
    if err != nil {
        return uuid.Nil, fmt.Errorf("insert outbox: %w", err)
    }
    
    return ride.ID, tx.Commit()
}

// Outbox Publisher goroutine
func (p *OutboxPublisher) Start(ctx context.Context) {
    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            p.pollAndPublish(ctx)
        }
    }
}

func (p *OutboxPublisher) pollAndPublish(ctx context.Context) {
    var events []OutboxEvent
    p.db.SelectContext(ctx, &events,
        `SELECT * FROM outbox_events WHERE status = 'PENDING' 
         ORDER BY created_at LIMIT 100`)
    
    for _, event := range events {
        msg := message.NewMessage(event.ID.String(), event.Payload)
        msg.Metadata.Set("aggregate_id", event.AggregateID.String())
        
        if err := p.publisher.Publish(event.Destination, msg); err != nil {
            event.RetryCount++
            if event.RetryCount >= 5 {
                p.db.ExecContext(ctx,
                    `UPDATE outbox_events SET status = 'FAILED', retry_count = $1 WHERE id = $2`,
                    event.RetryCount, event.ID)
            } else {
                p.db.ExecContext(ctx,
                    `UPDATE outbox_events SET retry_count = $1 WHERE id = $2`,
                    event.RetryCount, event.ID)
            }
            continue
        }
        
        p.db.ExecContext(ctx,
            `UPDATE outbox_events SET status = 'SENT', sent_at = NOW() WHERE id = $1`,
            event.ID)
    }
}
```

**Critérios de aceite:**
- [ ] Negócio + evento salvos na mesma transação (verificar: crash após INSERT ride → outbox também presente)
- [ ] Publisher poll a cada 500ms (configurável)
- [ ] Partition key = `aggregate_id` (ordem garantida por corrida)
- [ ] Retry até 5x com backoff; status FAILED após max retries
- [ ] Limpeza automática de SENT > 7 dias
- [ ] Consumer idempotente (processar mesmo evento 2x sem efeito colateral)
- [ ] Testes: dual write scenario → outbox garante consistência; publisher crash → reprocessa ao reiniciar

---

### Desafio 5.2 — CDC: Sincronização Write → Read com Debezium

Configure CDC para capturar mudanças no banco do Ride Service e sincronizar automaticamente o query model (CQRS).

**Cenário:** No Level 4, a sincronização write → read era via eventos internos. Agora, substitua pelo CDC: Debezium lê o WAL do PostgreSQL e publica mudanças no Kafka. O query model é atualizado automaticamente, sem código no write path.

**Requisitos:**
- **Debezium PostgreSQL Connector** (via Docker) lendo o WAL:
  - Capturar tabelas `rides`, `ride_status_history`
  - Filtrar colunas sensíveis (ex: passenger_phone, driver_cpf)
  - Tópico: `cdc.rideflow.rides`, `cdc.rideflow.ride_status_history`
- Formato do evento CDC com `op` (c/u/d/r), `before`, `after`, `source`
- **Snapshot inicial:** ao conectar, publicar todos os dados existentes (op: r)
- **Consumer CDC** que atualiza o `ride_details_view`:
  - INSERT (op=c) → inserir na view
  - UPDATE (op=u) → atualizar campos alterados
  - DELETE (op=d) → marcar como deletado (soft delete)
- **Outbox via CDC** (bonus): substituir o polling publisher por CDC na tabela `outbox_events`
  - Debezium lê INSERTs na outbox → publica no tópico correto
  - Sem necessidade de polling — latência sub-segundo

**Java 25 (Consumer Kafka para CDC):**
```java
// Consumer CDC → Atualiza Query Model
@Component
public class RideCdcConsumer {
    
    @KafkaListener(topics = "cdc.rideflow.rides")
    public void onRideCdcEvent(@Payload String payload) {
        var cdcEvent = objectMapper.readValue(payload, CdcEvent.class);
        
        switch (cdcEvent.op()) {
            case "c", "r" -> { // create ou snapshot
                var after = cdcEvent.after();
                rideViewRepo.upsert(new RideDetailsView(
                    UUID.fromString(after.get("id").asText()),
                    after.get("status").asText(),
                    Instant.ofEpochMilli(after.get("created_at").asLong())
                ));
            }
            case "u" -> { // update
                var after = cdcEvent.after();
                var rideId = UUID.fromString(after.get("id").asText());
                rideViewRepo.findById(rideId).ifPresent(view -> {
                    view.setStatus(after.get("status").asText());
                    if ("COMPLETED".equals(after.get("status").asText())) {
                        view.setCompletedAt(Instant.ofEpochMilli(
                            after.get("completed_at").asLong()));
                    }
                    rideViewRepo.save(view);
                });
            }
            case "d" -> { // delete
                var before = cdcEvent.before();
                var rideId = UUID.fromString(before.get("id").asText());
                rideViewRepo.softDelete(rideId);
            }
        }
    }
}

// DTO para evento CDC (formato Debezium)
public record CdcEvent(
    String op,                    // c, u, d, r
    JsonNode before,              // estado anterior
    JsonNode after,               // estado novo
    CdcSource source              // metadados
) {}

public record CdcSource(
    String table,
    String db,
    long ts_ms
) {}
```

**Go 1.26 (Consumer Watermill para CDC):**
```go
type CdcEvent struct {
    Op     string          `json:"op"`
    Before json.RawMessage `json:"before"`
    After  json.RawMessage `json:"after"`
    Source CdcSource       `json:"source"`
}

type CdcSource struct {
    Table string `json:"table"`
    DB    string `json:"db"`
    TsMs  int64  `json:"ts_ms"`
}

func (c *RideCdcConsumer) HandleCdcEvent(msg *message.Message) error {
    var event CdcEvent
    if err := json.Unmarshal(msg.Payload, &event); err != nil {
        return fmt.Errorf("unmarshal CDC event: %w", err)
    }
    
    switch event.Op {
    case "c", "r": // create ou snapshot
        var after RideCdcPayload
        json.Unmarshal(event.After, &after)
        _, err := c.db.ExecContext(msg.Context(),
            `INSERT INTO ride_details_view (ride_id, status, created_at)
             VALUES ($1, $2, $3)
             ON CONFLICT (ride_id) DO UPDATE SET status = $2`,
            after.ID, after.Status, time.UnixMilli(after.CreatedAt))
        return err
        
    case "u": // update
        var after RideCdcPayload
        json.Unmarshal(event.After, &after)
        _, err := c.db.ExecContext(msg.Context(),
            `UPDATE ride_details_view SET status = $1, completed_at = $2 WHERE ride_id = $3`,
            after.Status, nilIfZero(after.CompletedAt), after.ID)
        return err
        
    case "d": // delete (soft)
        var before RideCdcPayload
        json.Unmarshal(event.Before, &before)
        _, err := c.db.ExecContext(msg.Context(),
            `UPDATE ride_details_view SET deleted = true WHERE ride_id = $1`, before.ID)
        return err
    }
    
    return nil
}
```

**Docker Compose (Debezium):**
```yaml
services:
  debezium:
    image: debezium/connect:2.5
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: rideflow-cdc
      CONFIG_STORAGE_TOPIC: cdc-configs
      OFFSET_STORAGE_TOPIC: cdc-offsets
      STATUS_STORAGE_TOPIC: cdc-status

# Registrar connector via REST:
# POST http://debezium:8083/connectors
# {
#   "name": "rideflow-rides-connector",
#   "config": {
#     "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
#     "database.hostname": "ride-db",
#     "database.port": "5432",
#     "database.dbname": "rideflow",
#     "database.user": "cdc_user",
#     "table.include.list": "public.rides,public.ride_status_history",
#     "column.exclude.list": "public.rides.passenger_phone",
#     "topic.prefix": "cdc.rideflow",
#     "plugin.name": "pgoutput",
#     "slot.name": "rideflow_slot"
#   }
# }
```

**Critérios de aceite:**
- [ ] Debezium conectado ao PostgreSQL via WAL (não polling)
- [ ] Snapshot inicial: dados existentes publicados com op=r
- [ ] INSERT/UPDATE/DELETE capturados e publicados nos tópicos corretos
- [ ] Colunas sensíveis filtradas (`column.exclude.list`)
- [ ] Consumer atualiza `ride_details_view` para cada operação CDC
- [ ] Consumer idempotente (replay do snapshot não duplica dados)
- [ ] Testes: INSERT ride → CDC event no Kafka → query model atualizado
- [ ] (Bonus) Outbox via CDC: substituir polling publisher por Debezium na tabela outbox

---

### Desafio 5.3 — Backpressure: Controle de Fluxo no Location Service

Implemente mecanismos de backpressure para lidar com picos de atualizações de GPS.

**Cenário:** Em horário de pico, 50.000 motoristas enviam atualizações de GPS a cada 5 segundos (10K msg/s). O Location Service processa no máximo 5K/s. Sem backpressure, a fila cresce indefinidamente.

**Requisitos:**
- Implementar **3 estratégias** de backpressure:
  1. **Buffer com limite** — fila bounded (max 10.000 mensagens), após isso rejeitar novas
  2. **Drop oldest** — quando buffer cheio, descartar a posição mais antiga do mesmo motorista (manter mais recente)
  3. **Load Shedding por prioridade** — corridas ativas (IN_PROGRESS) = alta prioridade; motoristas disponíveis (IDLE) = baixa prioridade; em pico, descartar updates de motoristas IDLE
- **Métricas de backpressure:**
  - `location.buffer.size` — tamanho atual do buffer
  - `location.messages.dropped` — mensagens descartadas (por estratégia)
  - `location.consumer.lag` — lag do consumer
- **Auto-scaling trigger:** quando buffer > 80% por 2 minutos, sinalizar para scaling
- **Feedback ao producer:** retornar HTTP 429 quando buffer > 90%

**Java 25 (Bounded Queue + Load Shedding):**
```java
@Service
public class LocationIngestionService {
    private static final int MAX_BUFFER = 10_000;
    private final BlockingQueue<LocationUpdate> buffer = 
        new LinkedBlockingQueue<>(MAX_BUFFER);
    
    private final Counter droppedMessages;
    private final Gauge bufferSize;
    
    public LocationIngestionResult ingest(LocationUpdate update) {
        bufferSize.set(buffer.size());
        
        // Feedback: rejeitar se buffer > 90%
        if (buffer.size() > MAX_BUFFER * 0.9) {
            droppedMessages.increment();
            return LocationIngestionResult.rejected("Backpressure: buffer at 90%");
        }
        
        // Load shedding: em pico, priorizar corridas ativas
        if (buffer.size() > MAX_BUFFER * 0.7 && update.rideStatus() == RideStatus.IDLE) {
            droppedMessages.increment();
            return LocationIngestionResult.shed("Low priority: driver idle");
        }
        
        // Drop oldest do mesmo motorista
        if (!buffer.offer(update)) {
            buffer.stream()
                .filter(u -> u.driverId().equals(update.driverId()))
                .findFirst()
                .ifPresent(buffer::remove);
            buffer.offer(update);
        }
        
        return LocationIngestionResult.accepted();
    }
    
    // Consumer: processa em batch
    @Scheduled(fixedRate = 100) // a cada 100ms
    public void processBuffer() {
        var batch = new ArrayList<LocationUpdate>(500);
        buffer.drainTo(batch, 500); // drena até 500 de uma vez
        
        if (!batch.isEmpty()) {
            locationRepo.batchUpsert(batch); // bulk write
        }
    }
}
```

**Go 1.26 (Buffered channel + Load Shedding):**
```go
type LocationIngestionService struct {
    buffer   chan LocationUpdate
    maxSize  int
    metrics  *IngestionMetrics
}

func NewLocationIngestion(maxSize int) *LocationIngestionService {
    return &LocationIngestionService{
        buffer:  make(chan LocationUpdate, maxSize),
        maxSize: maxSize,
    }
}

func (s *LocationIngestionService) Ingest(update LocationUpdate) IngestionResult {
    currentSize := len(s.buffer)
    s.metrics.BufferSize.Set(float64(currentSize))
    
    // Feedback: rejeitar se buffer > 90%
    if currentSize > int(float64(s.maxSize)*0.9) {
        s.metrics.DroppedMessages.Inc()
        return IngestionResult{Status: "rejected", Reason: "backpressure: buffer at 90%"}
    }
    
    // Load shedding: em pico, priorizar corridas ativas
    if currentSize > int(float64(s.maxSize)*0.7) && update.RideStatus == RideStatusIdle {
        s.metrics.DroppedMessages.Inc()
        return IngestionResult{Status: "shed", Reason: "low priority: driver idle"}
    }
    
    select {
    case s.buffer <- update:
        return IngestionResult{Status: "accepted"}
    default:
        // Buffer cheio — drop (channel cheio)
        s.metrics.DroppedMessages.Inc()
        return IngestionResult{Status: "dropped", Reason: "buffer full"}
    }
}

// Consumer goroutine: processa em batch
func (s *LocationIngestionService) StartConsumer(ctx context.Context) {
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            batch := make([]LocationUpdate, 0, 500)
            for range 500 {
                select {
                case update := <-s.buffer:
                    batch = append(batch, update)
                default:
                    break
                }
            }
            if len(batch) > 0 {
                s.locationRepo.BatchUpsert(ctx, batch)
            }
        }
    }
}
```

**Critérios de aceite:**
- [ ] Buffer bounded com limite de 10.000 mensagens
- [ ] Drop oldest: motorista com update antigo no buffer é substituído pelo novo
- [ ] Load shedding: em pico (>70%), descartar updates de motoristas IDLE
- [ ] HTTP 429 retornado quando buffer > 90%
- [ ] Métricas: buffer size, dropped messages (por estratégia), consumer lag
- [ ] Processamento em batch (500 por iteração) para eficiência
- [ ] Testes de carga: 10K msg/s → sistema permanece estável; drops registrados nas métricas

---

### Desafio 5.4 — Sharding: Corridas Particionadas por Região

Implemente sharding para distribuir corridas por região geográfica e escalar horizontalmente.

**Cenário:** O RideFlow opera em São Paulo, Rio de Janeiro, Belo Horizonte e Curitiba. Cada cidade gera milhares de corridas por minuto. Um banco único não suporta a carga. Solução: shard por cidade.

**Requisitos:**
- **Shard key:** `city_code` (SP, RJ, BH, CWB)
- **Shard Router:** componente que determina qual shard recebe a query/write
- **Estratégia:** particionamento por lista (cada cidade = 1 shard)
- **Cross-shard queries:** quando necessário (relatório nacional), consultar todas as shards e agregar
- **Tópicos Kafka particionados:** `ride-events-SP`, `ride-events-RJ` (ou partições por city no mesmo tópico)
- **Rebalanceamento:** ao adicionar nova cidade (ex: Brasília/BSB), criar novo shard sem downtime

**Java 25 (Shard Router + Multi-DataSource):**
```java
@Component
public class ShardRouter {
    private final Map<String, DataSource> shards;
    
    public ShardRouter(
        @Qualifier("shard-sp") DataSource sp,
        @Qualifier("shard-rj") DataSource rj,
        @Qualifier("shard-bh") DataSource bh,
        @Qualifier("shard-cwb") DataSource cwb
    ) {
        this.shards = Map.of("SP", sp, "RJ", rj, "BH", bh, "CWB", cwb);
    }
    
    public DataSource route(String cityCode) {
        var ds = shards.get(cityCode);
        if (ds == null) {
            throw new UnknownShardException("No shard for city: " + cityCode);
        }
        return ds;
    }
    
    public Collection<DataSource> allShards() {
        return shards.values();
    }
}

@Service
public class ShardedRideRepository {
    private final ShardRouter router;
    
    public void save(Ride ride) {
        var ds = router.route(ride.getCityCode());
        var jdbc = new JdbcTemplate(ds);
        jdbc.update("INSERT INTO rides (...) VALUES (...)", /* params */);
    }
    
    public Optional<Ride> findById(String cityCode, UUID rideId) {
        var ds = router.route(cityCode);
        var jdbc = new JdbcTemplate(ds);
        return jdbc.queryForOptional("SELECT * FROM rides WHERE id = ?",
            rideMapper, rideId);
    }
    
    // Cross-shard query (scatter-gather)
    public DashboardDTO nationalDashboard(LocalDate date) {
        return router.allShards().parallelStream()
            .map(ds -> {
                var jdbc = new JdbcTemplate(ds);
                return jdbc.queryForObject(
                    "SELECT count(*) as total, sum(price) as revenue FROM rides WHERE date = ?",
                    dashboardMapper, date);
            })
            .reduce(DashboardDTO.empty(), DashboardDTO::merge);
    }
}
```

**Go 1.26 (Shard Router):**
```go
type ShardRouter struct {
    shards map[string]*sqlx.DB
}

func NewShardRouter(configs map[string]string) (*ShardRouter, error) {
    shards := make(map[string]*sqlx.DB)
    for city, dsn := range configs {
        db, err := sqlx.Connect("postgres", dsn)
        if err != nil {
            return nil, fmt.Errorf("connect shard %s: %w", city, err)
        }
        shards[city] = db
    }
    return &ShardRouter{shards: shards}, nil
}

func (r *ShardRouter) Route(cityCode string) (*sqlx.DB, error) {
    db, ok := r.shards[cityCode]
    if !ok {
        return nil, fmt.Errorf("unknown shard: %s", cityCode)
    }
    return db, nil
}

type ShardedRideRepo struct {
    router *ShardRouter
}

func (repo *ShardedRideRepo) Save(ctx context.Context, ride *Ride) error {
    db, err := repo.router.Route(ride.CityCode)
    if err != nil {
        return err
    }
    _, err = db.NamedExecContext(ctx,
        `INSERT INTO rides (id, city_code, passenger_id, status, created_at)
         VALUES (:id, :city_code, :passenger_id, :status, :created_at)`, ride)
    return err
}

// Cross-shard: scatter-gather
func (repo *ShardedRideRepo) NationalDashboard(ctx context.Context, date time.Time) (*Dashboard, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make(chan DashboardPartial, len(repo.router.shards))
    
    for city, db := range repo.router.shards {
        g.Go(func() error {
            var partial DashboardPartial
            err := db.GetContext(ctx, &partial,
                `SELECT count(*) as total, COALESCE(sum(price), 0) as revenue
                 FROM rides WHERE DATE(created_at) = $1`, date)
            if err != nil {
                return fmt.Errorf("shard %s: %w", city, err)
            }
            results <- partial
            return nil
        })
    }
    
    if err := g.Wait(); err != nil {
        return nil, err
    }
    close(results)
    
    dashboard := &Dashboard{}
    for partial := range results {
        dashboard.TotalRides += partial.Total
        dashboard.TotalRevenue += partial.Revenue
    }
    return dashboard, nil
}
```

**Critérios de aceite:**
- [ ] Shard key = `city_code` com particionamento por lista
- [ ] Router direciona reads/writes para o shard correto
- [ ] Cada shard tem seu próprio banco (ou schema) PostgreSQL
- [ ] Cross-shard query (scatter-gather) para relatório nacional funciona
- [ ] Cross-shard query é paralela (errgroup/parallelStream)
- [ ] Adicionar nova cidade (BSB) sem downtime: criar shard + registrar no router
- [ ] Tópicos Kafka particionados por cidade (partition key = city_code)
- [ ] Testes: write em SP → lê de SP shard; write em RJ → lê de RJ shard; national dashboard agrega todos
