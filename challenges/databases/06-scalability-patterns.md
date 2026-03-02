# Level 6 — Escalabilidade: Sharding, Caching, CQRS & Event Sourcing

> **Objetivo:** Dominar patterns de escalabilidade de dados — estratégias de sharding,
> caching distribuído, CQRS, Event Sourcing, e Database per Service.

**Referência:** [.docs/databases/04-scalability-patterns.md](../../.docs/databases/04-scalability-patterns.md)

---

## Contexto do Domínio

A StreamX cresceu para 10M de assinantes e 1B de watch events/mês. O PostgreSQL
single-node não aguenta mais. É hora de escalar horizontalmente: sharding para
distribuir dados, caching multi-camada, CQRS para separar leitura de escrita,
e Event Sourcing para histórico completo de ações do usuário.

---

## Desafios

### Desafio 6.1 — Estratégias de Sharding

**Contexto:** Os dados da StreamX precisam ser distribuídos em múltiplos nós.
A escolha da shard key e da estratégia de sharding determina tudo.

**Requisitos:**

- Avaliar 4 estratégias para o domínio StreamX:

```
┌─────────────────┬──────────────────┬──────────────┬─────────────────────────┐
│ Estratégia      │ Shard Key        │ Distribuição │ Use Case StreamX        │
├─────────────────┼──────────────────┼──────────────┼─────────────────────────┤
│ Range Sharding  │ created_at       │ Temporal     │ Watch events por mês    │
│                 │                  │              │ ⚠ Hot shard (mês atual) │
├─────────────────┼──────────────────┼──────────────┼─────────────────────────┤
│ Hash Sharding   │ hash(user_id)    │ Uniforme     │ Users & subscriptions   │
│                 │                  │              │ ✅ Distribuição uniforme │
├─────────────────┼──────────────────┼──────────────┼─────────────────────────┤
│ Consistent Hash │ hash(user_id)    │ Ring-based   │ Content cache (Redis)   │
│                 │ mod virtual nodes│              │ ✅ Min reshuffling       │
├─────────────────┼──────────────────┼──────────────┼─────────────────────────┤
│ Geo Sharding    │ user_region      │ Geográfica   │ Multi-region            │
│                 │                  │              │ ✅ Latência local        │
└─────────────────┴──────────────────┴──────────────┴─────────────────────────┘
```

- Implementar hash sharding para `users` (application-level):

**Java 25:**
```java
public class ShardRouter {
    private static final int SHARD_COUNT = 4;
    private final Map<Integer, DataSource> shards;

    public ShardRouter(Map<Integer, DataSource> shards) {
        this.shards = Map.copyOf(shards);
    }

    public int getShardId(UUID userId) {
        // Murmur3 hash para distribuição uniforme
        int hash = Hashing.murmur3_32_fixed().hashBytes(userId.toString().getBytes()).asInt();
        return Math.abs(hash % SHARD_COUNT);
    }

    public DataSource getDataSource(UUID userId) {
        return shards.get(getShardId(userId));
    }

    // Scatter-gather para queries cross-shard
    public <T> List<T> queryAllShards(String sql, RowMapper<T> mapper) {
        return shards.values().parallelStream()
            .flatMap(ds -> queryDataSource(ds, sql, mapper).stream())
            .toList();
    }
}
```

**Go 1.26:**
```go
type ShardRouter struct {
    shards []*sql.DB
    count  int
}

func NewShardRouter(dsns []string) (*ShardRouter, error) {
    shards := make([]*sql.DB, len(dsns))
    for i, dsn := range dsns {
        db, err := sql.Open("postgres", dsn)
        if err != nil {
            return nil, fmt.Errorf("shard %d: %w", i, err)
        }
        shards[i] = db
    }
    return &ShardRouter{shards: shards, count: len(dsns)}, nil
}

func (r *ShardRouter) GetShard(userID uuid.UUID) *sql.DB {
    h := fnv.New32a()
    h.Write(userID.Bytes())
    shardID := int(h.Sum32()) % r.count
    return r.shards[shardID]
}

// Scatter-gather
func (r *ShardRouter) QueryAllShards(ctx context.Context, query string, args ...any) ([]Row, error) {
    var (
        mu      sync.Mutex
        results []Row
        g       errgroup.Group
    )
    for _, shard := range r.shards {
        shard := shard
        g.Go(func() error {
            rows, err := shard.QueryContext(ctx, query, args...)
            if err != nil { return err }
            defer rows.Close()
            for rows.Next() {
                var row Row
                if err := rows.Scan(&row); err != nil { return err }
                mu.Lock()
                results = append(results, row)
                mu.Unlock()
            }
            return rows.Err()
        })
    }
    return results, g.Wait()
}
```

- Documentar shard key selection criteria:

```
Critérios para shard key:
✅ Alta cardinalidade (user_id: bilhões de valores únicos)
✅ Distribuição uniforme (hash elimina hotspots)
✅ Query isolation (maioria das queries usa user_id)
✅ Estável (user_id não muda)

❌ Evitar:
- created_at como shard key (hot shard no mês atual)
- country (distribuição desigual → US/BR com 80% dos dados)
- content_id para user-centric queries (cross-shard sempre)
```

**Critérios de aceite:**

- [ ] 4 estratégias de sharding comparadas com trade-offs
- [ ] Shard key selection com 4+ critérios documentados
- [ ] Hash sharding implementado em Java e Go (application-level)
- [ ] Scatter-gather implementado para cross-shard queries
- [ ] Anti-pattern: hot shard com Range Sharding temporal
- [ ] Diagrama: distribuição de dados por shard

---

### Desafio 6.2 — Caching Multi-Camada

**Contexto:** A StreamX precisa de cache em múltiplas camadas para suportar 10M de
usuários com latência < 50ms.

**Requisitos:**

- Implementar cache de 3 camadas:

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Cliente    │───▶│   CDN/Edge   │───▶│   App Cache  │───▶│   Database   │
│   (Browser)  │    │   (L1)       │    │   (L2 Redis) │    │   (Source)   │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
     ~0ms              < 10ms             < 5ms              < 50ms
```

- L1: In-Memory Cache (Caffeine/Ristretto) — hot data local:

**Java 25:**
```java
public class MultiLayerCache<T> {
    private final Cache<String, T> l1Cache; // Caffeine (in-process)
    private final RedisTemplate<String, T> l2Cache; // Redis (distributed)
    private final Function<String, T> dbLoader;

    public MultiLayerCache(int l1MaxSize, Duration l1Ttl, 
                           RedisTemplate<String, T> redis,
                           Function<String, T> dbLoader) {
        this.l1Cache = Caffeine.newBuilder()
            .maximumSize(l1MaxSize)
            .expireAfterWrite(l1Ttl)
            .recordStats()
            .build();
        this.l2Cache = redis;
        this.dbLoader = dbLoader;
    }

    public T get(String key) {
        // L1: In-memory (< 1ms)
        var l1 = l1Cache.getIfPresent(key);
        if (l1 != null) return l1;

        // L2: Redis (< 5ms)
        var l2 = l2Cache.opsForValue().get("cache:" + key);
        if (l2 != null) {
            l1Cache.put(key, l2); // promote to L1
            return l2;
        }

        // L3: Database (< 50ms)
        var value = dbLoader.apply(key);
        if (value != null) {
            l2Cache.opsForValue().set("cache:" + key, value, Duration.ofHours(1));
            l1Cache.put(key, value);
        }
        return value;
    }

    public void invalidate(String key) {
        l1Cache.invalidate(key);
        l2Cache.delete("cache:" + key);
    }
}
```

**Go 1.26:**
```go
type MultiLayerCache[T any] struct {
    l1    *ristretto.Cache[T]
    l2    *redis.Client
    load  func(ctx context.Context, key string) (T, error)
    l2TTL time.Duration
}

func (c *MultiLayerCache[T]) Get(ctx context.Context, key string) (T, error) {
    // L1: in-process
    if val, found := c.l1.Get(key); found {
        return val, nil
    }

    // L2: Redis
    data, err := c.l2.Get(ctx, "cache:"+key).Bytes()
    if err == nil {
        var val T
        if err := json.Unmarshal(data, &val); err == nil {
            c.l1.Set(key, val, 1) // promote
            return val, nil
        }
    }

    // L3: Database
    val, err := c.load(ctx, key)
    if err != nil {
        var zero T
        return zero, err
    }
    serialized, _ := json.Marshal(val)
    c.l2.Set(ctx, "cache:"+key, serialized, c.l2TTL)
    c.l1.Set(key, val, 1)
    return val, nil
}
```

- TTL com jitter para evitar Cache Avalanche:

```java
private Duration ttlWithJitter(Duration baseTtl) {
    var jitter = ThreadLocalRandom.current().nextLong(0, baseTtl.toSeconds() / 10);
    return baseTtl.plusSeconds(jitter);
}
```

**Critérios de aceite:**

- [ ] Cache 3 camadas: in-memory → Redis → DB
- [ ] L1 com size limit e eviction (Caffeine/Ristretto)
- [ ] L2 com TTL e jitter
- [ ] `get()` com fallback chain demonstrado
- [ ] `invalidate()` em todas as camadas
- [ ] Métricas: hit ratio de L1 e L2 logadas

---

### Desafio 6.3 — Cache Invalidation Strategies

**Contexto:** "There are only two hard things in Computer Science: cache invalidation
and naming things." A StreamX precisa de invalidação consistente.

**Requisitos:**

- Implementar 3 estratégias de invalidação:

**1. TTL-based (Time to Live):**
```
Use case: Catálogo de conteúdo (muda raramente)
TTL: 1 hora + jitter
Stale data: aceitável (conteúdo não muda frequentemente)
```

**2. Event-based (via mensageria):**
```java
// Quando conteúdo é atualizado → publicar evento
@Transactional
public void updateContent(String contentId, ContentUpdate update) {
    // 1. Atualizar no DB
    contentRepository.update(contentId, update);

    // 2. Invalidar cache
    cacheInvalidator.invalidate("content:" + contentId);

    // 3. Publicar evento para outras instâncias
    eventPublisher.publish(new CacheInvalidationEvent("content:" + contentId));
}

// Listener em todas as instâncias
@EventListener
public void onCacheInvalidation(CacheInvalidationEvent event) {
    multiLayerCache.invalidate(event.key());
}
```

**3. Write-Through (atualizar cache junto com DB):**
```go
func (s *ContentService) Update(ctx context.Context, id string, update ContentUpdate) error {
    // 1. Atualizar DB
    content, err := s.repo.Update(ctx, id, update)
    if err != nil { return err }

    // 2. Atualizar cache imediatamente (write-through)
    serialized, _ := json.Marshal(content)
    s.redis.Set(ctx, "content:"+id, serialized, 1*time.Hour)
    s.l1Cache.Set(id, content, 1)

    return nil
}
```

- Tabela comparativa:

```
┌───────────────────┬─────────────┬──────────────┬───────────────────┐
│ Estratégia        │ Consistência│ Complexidade │ Use Case StreamX  │
├───────────────────┼─────────────┼──────────────┼───────────────────┤
│ TTL-only          │ Eventual    │ Baixa        │ Catálogo público  │
│ Event-based       │ Near-real   │ Média        │ User profiles     │
│ Write-through     │ Forte       │ Média        │ Subscription data │
│ Write-behind      │ Eventual    │ Alta         │ View counters     │
└───────────────────┴─────────────┴──────────────┴───────────────────┘
```

**Critérios de aceite:**

- [ ] 3 estratégias implementadas (TTL, Event-based, Write-through)
- [ ] Tabela comparativa com consistência, complexidade, use case
- [ ] Event-based com pub/sub para multi-instance invalidation
- [ ] Write-through com atualização atômica (DB + cache)
- [ ] Trade-off documentado: consistência vs complexidade vs latência

---

### Desafio 6.4 — CQRS: Command Query Responsibility Segregation

**Contexto:** A StreamX tem padrões de acesso muito diferentes para escrita
(gravar eventos) e leitura (dashboards, listagens). CQRS separa os dois.

**Requisitos:**

- Implementar arquitetura CQRS:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Commands    │────▶│   Write DB   │────▶│  Event Bus   │
│  (mutations) │     │  (PostgreSQL)│     │  (Kafka)     │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                                                  ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Queries     │◀────│   Read DB    │◀────│  Projector   │
│  (reads)     │     │  (MongoDB/   │     │  (consumer)  │
└──────────────┘     │   Redis)     │     └──────────────┘
                     └──────────────┘
```

- Command Side (Write Model):

**Java 25:**
```java
// Command: registrar inscrição
public sealed interface SubscriptionCommand {
    record Create(UUID userId, String planId) implements SubscriptionCommand {}
    record Upgrade(UUID subscriptionId, String newPlanId) implements SubscriptionCommand {}
    record Cancel(UUID subscriptionId, String reason) implements SubscriptionCommand {}
}

@Service
public class SubscriptionCommandHandler {
    private final SubscriptionRepository writeRepo;  // PostgreSQL
    private final EventPublisher eventPublisher;

    public void handle(SubscriptionCommand.Create cmd) {
        var subscription = Subscription.create(cmd.userId(), cmd.planId());
        writeRepo.save(subscription);

        eventPublisher.publish(new SubscriptionCreatedEvent(
            subscription.id(), cmd.userId(), cmd.planId(), Instant.now()
        ));
    }
}
```

- Query Side (Read Model):

```java
// Projector: consome eventos e atualiza read model
@Component
public class SubscriptionProjector {
    private final MongoTemplate readStore;  // MongoDB

    @EventListener
    public void on(SubscriptionCreatedEvent event) {
        var readModel = new SubscriptionReadModel(
            event.subscriptionId(),
            event.userId(),
            event.planId(),
            "ACTIVE",
            event.timestamp()
        );
        readStore.save(readModel, "subscriptions_view");
    }

    @EventListener
    public void on(SubscriptionCancelledEvent event) {
        var update = Update.update("status", "CANCELLED")
            .set("cancelledAt", event.timestamp())
            .set("cancelReason", event.reason());
        readStore.updateFirst(
            Query.query(Criteria.where("_id").is(event.subscriptionId())),
            update, "subscriptions_view"
        );
    }
}

// Query Service: lê do read model
@Service
public class SubscriptionQueryService {
    private final MongoTemplate readStore;

    public List<SubscriptionReadModel> getActiveByUser(UUID userId) {
        return readStore.find(
            Query.query(Criteria.where("userId").is(userId).and("status").is("ACTIVE")),
            SubscriptionReadModel.class,
            "subscriptions_view"
        );
    }
}
```

**Go 1.26:**
```go
// Command Handler
type SubscriptionCommandHandler struct {
    writeDB   *sql.DB       // PostgreSQL
    publisher EventPublisher // Kafka
}

func (h *SubscriptionCommandHandler) CreateSubscription(ctx context.Context, cmd CreateSubscriptionCmd) error {
    sub := NewSubscription(cmd.UserID, cmd.PlanID)

    // Write to PostgreSQL
    _, err := h.writeDB.ExecContext(ctx,
        `INSERT INTO subscriptions (id, user_id, plan_id, status, created_at)
         VALUES ($1, $2, $3, $4, $5)`,
        sub.ID, sub.UserID, sub.PlanID, sub.Status, sub.CreatedAt,
    )
    if err != nil { return err }

    // Publish event
    return h.publisher.Publish(ctx, SubscriptionCreatedEvent{
        SubscriptionID: sub.ID,
        UserID:         cmd.UserID,
        PlanID:         cmd.PlanID,
        Timestamp:      sub.CreatedAt,
    })
}

// Projector (Kafka consumer)
type SubscriptionProjector struct {
    readDB *mongo.Collection // MongoDB
}

func (p *SubscriptionProjector) OnSubscriptionCreated(ctx context.Context, event SubscriptionCreatedEvent) error {
    _, err := p.readDB.InsertOne(ctx, bson.M{
        "_id":       event.SubscriptionID,
        "userId":    event.UserID,
        "planId":    event.PlanID,
        "status":    "ACTIVE",
        "createdAt": event.Timestamp,
    })
    return err
}
```

**Critérios de aceite:**

- [ ] Write model (PostgreSQL) e Read model (MongoDB) separados
- [ ] Commands como sealed interface / sum type
- [ ] Projector consome eventos e atualiza read model
- [ ] Query service lê exclusivamente do read model
- [ ] Diagrama CQRS com fluxo Command → Event → Projection → Query
- [ ] Trade-off documentado: consistência eventual entre write e read

---

### Desafio 6.5 — Event Sourcing para Watch History

**Contexto:** Em vez de armazenar apenas o estado atual do progresso do usuário,
armazene todos os eventos. O estado é derivado do replay dos eventos.

**Requisitos:**

- Implementar Event Store:

```sql
-- PostgreSQL: Event Store
CREATE TABLE watch_events_store (
    event_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id   TEXT NOT NULL,           -- "user:{userId}:content:{contentId}"
    aggregate_type TEXT NOT NULL DEFAULT 'WatchSession',
    event_type     TEXT NOT NULL,           -- PLAYBACK_STARTED, PAUSED, SEEKED, RESUMED, STOPPED
    event_data     JSONB NOT NULL,
    metadata       JSONB DEFAULT '{}',      -- device, ip, app_version
    version        INTEGER NOT NULL,        -- para ordering e concurrency
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (aggregate_id, version)          -- optimistic concurrency
);

CREATE INDEX idx_events_aggregate ON watch_events_store (aggregate_id, version);
CREATE INDEX idx_events_type ON watch_events_store (event_type, created_at);
```

- Definir eventos:

```java
// Eventos de watch session
public sealed interface WatchEvent {
    String aggregateId();
    Instant timestamp();

    record PlaybackStarted(String aggregateId, UUID contentId,
        long positionMs, String device, Instant timestamp) implements WatchEvent {}

    record Paused(String aggregateId,
        long positionMs, Instant timestamp) implements WatchEvent {}

    record Seeked(String aggregateId,
        long fromMs, long toMs, Instant timestamp) implements WatchEvent {}

    record Resumed(String aggregateId,
        long positionMs, Instant timestamp) implements WatchEvent {}

    record Stopped(String aggregateId,
        long positionMs, long totalWatchedMs, Instant timestamp) implements WatchEvent {}
}
```

- Reconstruir estado a partir de eventos:

```java
public class WatchSession {
    private UUID userId;
    private UUID contentId;
    private long currentPositionMs;
    private long totalWatchedMs;
    private String status; // PLAYING, PAUSED, STOPPED
    private int version;

    // Reconstruct from events
    public static WatchSession fromEvents(List<WatchEvent> events) {
        var session = new WatchSession();
        for (var event : events) {
            session.apply(event);
        }
        return session;
    }

    private void apply(WatchEvent event) {
        switch (event) {
            case WatchEvent.PlaybackStarted e -> {
                this.userId = extractUserId(e.aggregateId());
                this.contentId = e.contentId();
                this.currentPositionMs = e.positionMs();
                this.status = "PLAYING";
            }
            case WatchEvent.Paused e -> {
                this.currentPositionMs = e.positionMs();
                this.status = "PAUSED";
            }
            case WatchEvent.Seeked e -> {
                this.currentPositionMs = e.toMs();
            }
            case WatchEvent.Resumed e -> {
                this.currentPositionMs = e.positionMs();
                this.status = "PLAYING";
            }
            case WatchEvent.Stopped e -> {
                this.currentPositionMs = e.positionMs();
                this.totalWatchedMs = e.totalWatchedMs();
                this.status = "STOPPED";
            }
        }
        this.version++;
    }
}
```

- Snapshot para performance:

```java
// Snapshot a cada 100 eventos
public class WatchSessionRepository {
    private static final int SNAPSHOT_INTERVAL = 100;

    public WatchSession load(String aggregateId) {
        // 1. Buscar snapshot mais recente
        var snapshot = snapshotStore.findLatest(aggregateId);

        // 2. Buscar eventos após snapshot
        int fromVersion = snapshot != null ? snapshot.version() + 1 : 0;
        var events = eventStore.findEvents(aggregateId, fromVersion);

        // 3. Reconstruir
        var session = snapshot != null
            ? WatchSession.fromSnapshot(snapshot)
            : new WatchSession();
        events.forEach(session::apply);

        // 4. Criar snapshot se necessário
        if (session.version() % SNAPSHOT_INTERVAL == 0) {
            snapshotStore.save(session.toSnapshot());
        }

        return session;
    }
}
```

**Critérios de aceite:**

- [ ] Event Store com append-only (nunca UPDATE/DELETE)
- [ ] 5 event types definidos como sealed interface
- [ ] State reconstruction via replay de eventos
- [ ] Optimistic concurrency com version field
- [ ] Snapshot pattern para performance (a cada N eventos)
- [ ] Trade-off: storage vs queryability vs auditabilidade

---

### Desafio 6.6 — Database per Service Pattern

**Contexto:** Em microservices, cada serviço tem seu banco. A StreamX tem 4 serviços
com bancos distintos.

**Requisitos:**

- Definir a arquitetura polyglot por serviço:

```
┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐
│   User Service     │  │  Content Service   │  │  Streaming Service │
│   ┌──────────┐     │  │  ┌──────────┐      │  │  ┌──────────┐     │
│   │PostgreSQL│     │  │  │ MongoDB  │      │  │  │Cassandra │     │
│   │(users,   │     │  │  │(catalog, │      │  │  │(events,  │     │
│   │ subs,    │     │  │  │ reviews) │      │  │  │ metrics) │     │
│   │ billing) │     │  │  └──────────┘      │  │  └──────────┘     │
│   └──────────┘     │  └────────────────────┘  └────────────────────┘
└────────────────────┘
           │                      │                       │
           ▼                      ▼                       ▼
     ┌───────────────────────────────────────────────────────┐
     │                   Event Bus (Kafka)                   │
     └───────────────────────────────────────────────────────┘
           │                      │                       │
           ▼                      ▼                       ▼
┌────────────────────┐  ┌────────────────────┐
│  Rec. Service      │  │ Analytics Service  │
│  ┌──────────┐      │  │  ┌──────────┐      │
│  │  Neo4j   │      │  │  │TimescaleDB│     │
│  │(graph,   │      │  │  │(time-     │     │
│  │ recs)    │      │  │  │ series)   │     │
│  └──────────┘      │  │  └──────────┘      │
└────────────────────┘  └────────────────────┘
```

- Tratar problema de queries cross-service:

**API Composition:**
```java
// Compor dados de múltiplos serviços
public record ContentDetailView(
    ContentInfo content,       // do Content Service
    List<Review> topReviews,   // do Content Service
    UserProgress progress,     // do Streaming Service
    List<Recommendation> recs  // do Recommendation Service
) {}

@Service
public class ContentDetailComposer {
    private final ContentServiceClient contentClient;
    private final StreamingServiceClient streamingClient;
    private final RecommendationServiceClient recClient;

    public ContentDetailView compose(String contentId, String userId) {
        // Parallelizar chamadas independentes
        var contentFuture = CompletableFuture.supplyAsync(
            () -> contentClient.getContent(contentId));
        var progressFuture = CompletableFuture.supplyAsync(
            () -> streamingClient.getProgress(userId, contentId));
        var recsFuture = CompletableFuture.supplyAsync(
            () -> recClient.getSimilar(contentId));

        return new ContentDetailView(
            contentFuture.join(),
            contentClient.getTopReviews(contentId),
            progressFuture.join(),
            recsFuture.join()
        );
    }
}
```

- Tratar consistência entre serviços com Saga:

```java
// Saga: criar assinatura → provisionar acesso → ativar streaming
public class SubscriptionSaga {
    public void execute(CreateSubscriptionCmd cmd) {
        try {
            // Step 1: criar subscription (User Service)
            var sub = userService.createSubscription(cmd);

            // Step 2: configurar perfil streaming (Streaming Service)
            streamingService.provisionAccess(sub.userId(), sub.planId());

            // Step 3: atualizar grafo (Recommendation Service)
            recService.updateUserPlan(sub.userId(), sub.planId());

        } catch (Exception e) {
            // Compensating transactions
            userService.cancelSubscription(sub.id());
            streamingService.revokeAccess(sub.userId());
            throw new SagaFailedException("Subscription creation failed", e);
        }
    }
}
```

**Critérios de aceite:**

- [ ] 5 serviços com bancos distintos mapeados
- [ ] Event Bus (Kafka) como backbone de comunicação
- [ ] API Composition para queries cross-service
- [ ] Saga pattern para transações distribuídas
- [ ] Diagrama de arquitetura com fluxos de dados
- [ ] Trade-off: autonomia vs consistência vs complexidade

---

### Desafio 6.7 — Connection Patterns & Performance

**Contexto:** Com múltiplos bancos, gerenciar conexões é crítico.
Pool sizing errado pode derrubar o sistema.

**Requisitos:**

- Calcular pool size para StreamX:

```
Fórmula HikariCP:
  connections = ((core_count * 2) + effective_spindle_count)

StreamX PostgreSQL (API pods):
  - 8 cores por pod, SSD (spindle_count = 0)
  - connections = (8 * 2) + 0 = 16
  - 10 pods = 160 connections
  - PostgreSQL max_connections = 200 (40 reserva)

StreamX MongoDB (Content Service):
  - Connection pool per pod: minPoolSize=5, maxPoolSize=20
  - 5 pods = 100 max connections
  - mongos maxIncomingConnections = 150

StreamX Redis (Cache):
  - Pool size: 10 per pod (Redis é single-threaded, não precisa de muitas)
  - 10 pods = 100 connections
  - Redis maxclients = 128
```

- Implementar health check e circuit breaker:

**Java 25:**
```java
public record DatabaseHealth(
    String name,
    boolean healthy,
    Duration latency,
    int activeConnections,
    int idleConnections
) {}

public DatabaseHealth checkPostgres(HikariDataSource ds) {
    var start = Instant.now();
    try (var conn = ds.getConnection()) {
        conn.createStatement().execute("SELECT 1");
        return new DatabaseHealth(
            "postgresql",
            true,
            Duration.between(start, Instant.now()),
            ds.getHikariPoolMXBean().getActiveConnections(),
            ds.getHikariPoolMXBean().getIdleConnections()
        );
    } catch (SQLException e) {
        return new DatabaseHealth("postgresql", false, Duration.ZERO, 0, 0);
    }
}
```

- Implementar materialized view refresh pattern:

```sql
-- Materialized View: dashboard de conteúdo popular
CREATE MATERIALIZED VIEW mv_popular_content AS
SELECT
    c.id, c.title, c.content_type,
    count(DISTINCT w.user_id) AS unique_viewers,
    count(*) AS total_views,
    avg(w.watch_duration_ms) AS avg_watch_ms
FROM content c
JOIN watch_events_summary w ON c.id = w.content_id
WHERE w.event_date >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY c.id, c.title, c.content_type
ORDER BY total_views DESC
LIMIT 100;

-- Refresh sem bloquear reads
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_popular_content;

-- Schedule: a cada 15 minutos
-- pg_cron ou job scheduler externo
```

**Critérios de aceite:**

- [ ] Pool sizing calculado para 3 databases (PG, Mongo, Redis)
- [ ] Fórmula de HikariCP aplicada
- [ ] Health check endpoint com latência e pool stats
- [ ] Materialized view com `REFRESH CONCURRENTLY`
- [ ] Connection leak detection configurado
- [ ] Alerta: pool exhaustion threshold definido

---

### Desafio 6.8 — Scalability Anti-Patterns

**Contexto:** Identifique anti-patterns de escalabilidade que a StreamX pode enfrentar
ao crescer de 1M para 10M usuários.

**Requisitos:**

- Documentar e corrigir 6 anti-patterns:

**1. Distributed Monolith:**
```
❌ Microservices que compartilham o mesmo banco
   → Qualquer migration afeta todos os serviços
   → Coupling > monolito

✅ Database per Service com comunicação via eventos
```

**2. N+1 Across Services:**
```java
// ❌ Para cada conteúdo, chamar User Service para pegar creator info
for (var content : contents) {
    var creator = userService.getUser(content.creatorId()); // N calls
}

// ✅ Batch call ou cache local
var creatorIds = contents.stream().map(Content::creatorId).toList();
var creators = userService.getUsersBatch(creatorIds); // 1 call
```

**3. Over-Caching (cache everything):**
```
❌ Cachear dados raramente acessados → desperdício de memória
❌ Cachear dados que mudam frequentemente → invalidação constante

✅ Cache seletivo:
  - Hot data (top 100 conteúdos): cache agressivo (1h TTL)
  - Warm data (catálogo): cache moderado (15min TTL)
  - Cold data (historico >1 ano): sem cache
```

**4. Improper Pagination:**
```sql
-- ❌ OFFSET pagination em tabelas grandes
SELECT * FROM watch_events ORDER BY created_at DESC OFFSET 100000 LIMIT 20;
-- → Scan + Sort de 100K+ rows

-- ✅ Keyset pagination
SELECT * FROM watch_events
WHERE created_at < $last_seen_timestamp
ORDER BY created_at DESC LIMIT 20;
```

**5. Missing Index on Foreign Keys:**
```sql
-- ❌ FK sem índice → table scan em JOINs e cascading deletes
ALTER TABLE subscriptions ADD FOREIGN KEY (user_id) REFERENCES users(id);
-- → SELECT * FROM subscriptions WHERE user_id = ? → seq scan

-- ✅ Índice na FK
CREATE INDEX idx_subscriptions_user_id ON subscriptions (user_id);
```

**6. Synchronous Cross-Database Writes:**
```java
// ❌ Escrita síncrona em PG + Mongo + Redis na mesma request
@Transactional
public void createContent(Content content) {
    postgresRepo.save(content);      // pode falhar
    mongoRepo.save(content);         // pode falhar → PG já commitou
    redisCache.set(content);         // pode falhar → inconsistência
}

// ✅ Outbox Pattern: commit PG + outbox → async para Mongo + Redis
@Transactional
public void createContent(Content content) {
    postgresRepo.save(content);
    outboxRepo.save(new OutboxEvent("ContentCreated", content));
    // → Worker consome outbox e propaga para Mongo/Redis
}
```

**Critérios de aceite:**

- [ ] 6 anti-patterns documentados com código "antes/depois"
- [ ] Cada anti-pattern aplicado ao domínio StreamX
- [ ] Impacto quantificado (latência, custo, risco)
- [ ] Outbox Pattern implementado para cross-DB consistency
- [ ] Documento: `decisions/06-scalability-anti-patterns.md`
