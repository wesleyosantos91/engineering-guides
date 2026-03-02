# Level 5 — NoSQL: Wide-Column & Graph (Cassandra, Neo4j)

> **Objetivo:** Dominar modelagem query-driven para Wide-Column Stores (Cassandra) e
> Graph Databases (Neo4j), incluindo CQL, Cypher, e patterns de time-series e recomendação.

**Referência:** [.docs/databases/03-nosql-best-practices.md](../../.docs/databases/03-nosql-best-practices.md)

---

## Contexto do Domínio

A StreamX gera bilhões de watch events por mês (play, pause, seek, stop, quality_change).
Esses eventos precisam de um banco otimizado para alta ingestão e queries por time range
— Cassandra é ideal. Paralelamente, o sistema de recomendações precisa navegar relações
"quem assistiu X também assistiu Y" e "conteúdos similares" — Neo4j resolve naturalmente.

---

## Desafios

### Desafio 5.1 — Cassandra: Modelagem Query-Driven

**Contexto:** Em Cassandra, a modelagem começa pelas queries, não pelas entidades.
Desnormalização é esperada — dados são duplicados entre tables para otimizar leitura.

**Requisitos:**

- Listar queries necessárias para watch events:

| # | Query | Partition Key | Clustering Column |
|---|-------|---------------|-------------------|
| Q1 | Watch events de um user em um dia | `user_id, event_date` | `event_timestamp DESC` |
| Q2 | Watch events de um conteúdo em uma hora | `content_id, event_hour` | `event_timestamp DESC` |
| Q3 | Métricas diárias de um conteúdo | `content_id` | `metric_date DESC` |
| Q4 | Último progresso de um user em conteúdo | `user_id, content_id` | — (sobrescreve) |

- Criar tabela para cada query (desnormalização):

```sql
-- Q1: Watch events por usuário por dia
CREATE TABLE watch_events_by_user (
    user_id       UUID,
    event_date    DATE,
    event_timestamp TIMESTAMP,
    content_id    UUID,
    content_title TEXT,       -- desnormalizado (evita JOIN)
    event_type    TEXT,       -- PLAY, PAUSE, SEEK, STOP, QUALITY_CHANGE
    position_ms   BIGINT,
    duration_ms   BIGINT,
    device_type   TEXT,       -- MOBILE, TV, WEB, TABLET
    quality       TEXT,       -- SD, HD, FHD, 4K
    PRIMARY KEY ((user_id, event_date), event_timestamp)
) WITH CLUSTERING ORDER BY (event_timestamp DESC)
  AND default_time_to_live = 7776000  -- 90 dias
  AND compaction = {'class': 'TimeWindowCompactionStrategy',
                    'compaction_window_size': 1,
                    'compaction_window_unit': 'DAYS'};

-- Q2: Watch events por conteúdo por hora (analytics)
CREATE TABLE watch_events_by_content (
    content_id    UUID,
    event_hour    TIMESTAMP,  -- truncado para hora
    event_timestamp TIMESTAMP,
    user_id       UUID,
    event_type    TEXT,
    position_ms   BIGINT,
    device_type   TEXT,
    PRIMARY KEY ((content_id, event_hour), event_timestamp)
) WITH CLUSTERING ORDER BY (event_timestamp DESC)
  AND default_time_to_live = 2592000;  -- 30 dias

-- Q3: Métricas diárias por conteúdo (counter pattern)
CREATE TABLE content_daily_metrics (
    content_id     UUID,
    metric_date    DATE,
    total_views    COUNTER,
    unique_viewers COUNTER,
    total_watch_ms COUNTER,
    PRIMARY KEY (content_id, metric_date)
) WITH CLUSTERING ORDER BY (metric_date DESC);

-- Q4: Último progresso do usuário (sobrescreve sempre)
CREATE TABLE user_content_progress (
    user_id       UUID,
    content_id    UUID,
    position_ms   BIGINT,
    duration_ms   BIGINT,
    percentage    FLOAT,
    updated_at    TIMESTAMP,
    PRIMARY KEY (user_id, content_id)
);
```

**Critérios de aceite:**

- [ ] 4+ queries documentadas com partition key e clustering columns
- [ ] Tabela dedicada para cada query (query-driven design)
- [ ] Dados desnormalizados (`content_title` em `watch_events_by_user`)
- [ ] TTL adequado por tabela (90 dias para events, 30 dias para analytics)
- [ ] TimeWindowCompactionStrategy para dados temporais
- [ ] Counter table para métricas (tipo especial do Cassandra)

---

### Desafio 5.2 — Cassandra: Bucket Pattern & Partition Sizing

**Contexto:** Partitions com muitas rows ficam lentas. O Bucket Pattern limita o
tamanho de cada partition distribuindo dados por "buckets" temporais.

**Requisitos:**

- Analisar o problema de partitions grandes:

```
// ❌ Partition key apenas por content_id
// Conteúdo popular: 10M events/dia → partition gigante (> 100MB)
PRIMARY KEY (content_id, event_timestamp)

// ✅ Bucket Pattern: particionar por hora
// Máximo ~250K events/hora → partition controlada (~10MB)
PRIMARY KEY ((content_id, event_hour), event_timestamp)
```

- Calcular tamanho de partition:

```
Partition Size = Nrows × (CKsize + ColumnSize)
Target: < 100MB por partition, < 100K rows idealmente

Exemplo StreamX (conteúdo popular):
- 500K plays/dia = ~21K plays/hora
- Row size: ~200 bytes
- Partition/hora: 21K × 200B = ~4.2MB ✅ (bem abaixo de 100MB)
```

- Implementar insert com bucket em Java e Go:

**Java 25:**
```java
// Cassandra — Inserir watch event com bucket por hora
public void recordWatchEvent(WatchEvent event) {
    var eventHour = event.timestamp().truncatedTo(ChronoUnit.HOURS);

    session.execute(
        QueryBuilder.insertInto("watch_events_by_content")
            .value("content_id", literal(event.contentId()))
            .value("event_hour", literal(eventHour))
            .value("event_timestamp", literal(event.timestamp()))
            .value("user_id", literal(event.userId()))
            .value("event_type", literal(event.eventType().name()))
            .value("position_ms", literal(event.positionMs()))
            .value("device_type", literal(event.deviceType().name()))
            .build()
    );

    // Counter: incrementar métricas diárias
    session.execute(
        QueryBuilder.update("content_daily_metrics")
            .increment("total_views", literal(1L))
            .increment("total_watch_ms", literal(event.durationMs()))
            .whereColumn("content_id").isEqualTo(literal(event.contentId()))
            .whereColumn("metric_date").isEqualTo(literal(event.timestamp().toLocalDate()))
            .build()
    );
}
```

**Go 1.26:**
```go
func (r *WatchEventRepo) Record(ctx context.Context, event WatchEvent) error {
    eventHour := event.Timestamp.Truncate(time.Hour)

    // Insert event
    if err := r.session.Query(`
        INSERT INTO watch_events_by_content
            (content_id, event_hour, event_timestamp, user_id, event_type, position_ms, device_type)
        VALUES (?, ?, ?, ?, ?, ?, ?)`,
        event.ContentID, eventHour, event.Timestamp,
        event.UserID, event.EventType, event.PositionMs, event.DeviceType,
    ).WithContext(ctx).Exec(); err != nil {
        return fmt.Errorf("insert watch event: %w", err)
    }

    // Increment counter
    return r.session.Query(`
        UPDATE content_daily_metrics
        SET total_views = total_views + 1, total_watch_ms = total_watch_ms + ?
        WHERE content_id = ? AND metric_date = ?`,
        event.DurationMs, event.ContentID, event.Timestamp.Format("2006-01-02"),
    ).WithContext(ctx).Exec()
}
```

**Critérios de aceite:**

- [ ] Bucket Pattern implementado (particionar por hora)
- [ ] Cálculo de partition size documentado
- [ ] Insert com dual-write (event + counter) em Java e Go
- [ ] Time truncation correta (`truncatedTo(HOURS)` / `Truncate(time.Hour)`)
- [ ] Demonstrar: query dentro de 1 bucket vs cross-bucket

---

### Desafio 5.3 — Cassandra: Consistency Tuning & Tombstones

**Contexto:** Cassandra oferece consistency tuning por query.
A StreamX precisa de consistência diferente para eventos vs progresso do usuário.

**Requisitos:**

- Configurar consistency levels por use case:

```
┌────────────────────┬─────────────────────┬──────────────────┬──────────────┐
│ Use Case           │ Write CL            │ Read CL          │ Justificativa│
├────────────────────┼─────────────────────┼──────────────────┼──────────────┤
│ Watch Events       │ ONE                 │ ONE              │ Perda aceitá │
│ (analytics)        │                     │                  │ vel, volume  │
│                    │                     │                  │ altíssimo    │
├────────────────────┼─────────────────────┼──────────────────┼──────────────┤
│ User Progress      │ LOCAL_QUORUM        │ LOCAL_QUORUM     │ Consistência │
│ (posição video)    │                     │                  │ forte, UX    │
│                    │                     │                  │ importa      │
├────────────────────┼─────────────────────┼──────────────────┼──────────────┤
│ Content Metrics    │ ANY (counter)       │ ONE              │ Aproximado é │
│ (view_count)       │                     │                  │ suficiente   │
└────────────────────┴─────────────────────┴──────────────────┴──────────────┘

Regra: W + R > RF → strong consistency
RF=3: QUORUM(2) + QUORUM(2) = 4 > 3 → ✅ strong
RF=3: ONE(1) + ONE(1) = 2 < 3 → ✗ eventual
```

- Lidar com tombstones (anti-pattern de DELETEs frequentes):

```sql
-- ❌ RUIM: Delete individualizado em Cassandra cria tombstones
DELETE FROM watch_events_by_user WHERE user_id = ? AND event_date = ? AND event_timestamp = ?;
-- → Milhões de tombstones → read performance degrada

-- ✅ BOM: TTL para expiração automática (sem tombstones)
INSERT INTO watch_events_by_user (...) VALUES (...) USING TTL 7776000;
-- → Dados expiram automaticamente após 90 dias

-- ✅ BOM: Partition-level delete (1 tombstone por partition)
DELETE FROM watch_events_by_user WHERE user_id = ? AND event_date = ?;
-- → Remove partition inteira do dia (aceitável após ETL)
```

- Monitorar tombstones:

```sql
-- Verificar tombstone warnings no log
-- nodetool cfstats streamx.watch_events_by_user
-- → droppable_tombstone_ratio: deve ser baixo
```

**Critérios de aceite:**

- [ ] Consistency level configurado por use case (3 cenários)
- [ ] Demonstrar: `W + R > RF` para strong consistency
- [ ] TTL como alternativa a DELETE (sem tombstones)
- [ ] Partition-level delete quando necessário
- [ ] Monitoramento de tombstones documentado
- [ ] Compaction strategy justificada (TWCS para time-series)

---

### Desafio 5.4 — Neo4j: Modelagem do Grafo de Recomendações

**Contexto:** O sistema de recomendações da StreamX precisa de um grafo conectando
Users, Content, Genres, Creators com relações ricas.

**Requisitos:**

- Criar modelo de grafo:

```
Nodes:
  (:User {id, name, plan, createdAt})
  (:Content {id, title, contentType, releaseYear})
  (:Genre {name})
  (:Creator {id, name, type})  // Director, Actor, Artist, Host

Relationships:
  (:User)-[:WATCHED {watchedAt, rating, completionPct}]->(:Content)
  (:User)-[:ADDED_TO_WATCHLIST {addedAt}]->(:Content)
  (:User)-[:FOLLOWS]->(:User)
  (:Content)-[:IN_GENRE]->(:Genre)
  (:Content)-[:CREATED_BY {role}]->(:Creator)
  (:Content)-[:SIMILAR_TO {score}]->(:Content)
```

- Criar dados com Cypher:

```cypher
// Criar nodes
CREATE (:User {id: 'user-001', name: 'Alice', plan: 'PREMIUM', createdAt: datetime()})
CREATE (:User {id: 'user-002', name: 'Bob', plan: 'STANDARD', createdAt: datetime()})

CREATE (:Content {id: 'content-001', title: 'Interstellar', contentType: 'MOVIE', releaseYear: 2014})
CREATE (:Content {id: 'content-002', title: 'The Martian', contentType: 'MOVIE', releaseYear: 2015})
CREATE (:Content {id: 'content-003', title: 'Arrival', contentType: 'MOVIE', releaseYear: 2016})

CREATE (:Genre {name: 'Sci-Fi'})
CREATE (:Genre {name: 'Drama'})

CREATE (:Creator {id: 'creator-nolan', name: 'Christopher Nolan', type: 'Director'})

// Criar relationships
MATCH (u:User {id: 'user-001'}), (c:Content {id: 'content-001'})
CREATE (u)-[:WATCHED {watchedAt: datetime(), rating: 5, completionPct: 100}]->(c)

MATCH (c:Content {id: 'content-001'}), (g:Genre {name: 'Sci-Fi'})
CREATE (c)-[:IN_GENRE]->(g)

MATCH (c:Content {id: 'content-001'}), (cr:Creator {id: 'creator-nolan'})
CREATE (c)-[:CREATED_BY {role: 'Director'}]->(cr)
```

- Criar índices:

```cypher
CREATE INDEX idx_user_id FOR (u:User) ON (u.id);
CREATE INDEX idx_content_id FOR (c:Content) ON (c.id);
CREATE INDEX idx_genre_name FOR (g:Genre) ON (g.name);
CREATE INDEX idx_creator_id FOR (cr:Creator) ON (cr.id);
```

**Critérios de aceite:**

- [ ] 4 node labels com propriedades relevantes
- [ ] 6+ relationship types definidos
- [ ] Relationships com propriedades (rating, score, role)
- [ ] Dados seed: 10+ nodes, 20+ relationships
- [ ] Índices em todas as propriedades usadas em MATCH
- [ ] Diagrama visual do grafo

---

### Desafio 5.5 — Neo4j: Queries Cypher para Recomendação

**Contexto:** Implemente as queries de recomendação que são impossíveis ou extremamente
ineficientes em SQL/NoSQL-document mas naturais em grafo.

**Requisitos:**

- Implementar 5 queries de recomendação:

**1. "Quem assistiu X também assistiu Y" (Collaborative Filtering):**
```cypher
MATCH (target:User {id: $userId})-[:WATCHED]->(c:Content)<-[:WATCHED]-(similar:User)
WHERE target <> similar
WITH similar, count(c) AS commonContent
WHERE commonContent >= 3
MATCH (similar)-[:WATCHED]->(rec:Content)
WHERE NOT (target)-[:WATCHED]->(rec)
RETURN rec.title, rec.contentType,
       count(DISTINCT similar) AS recommenders,
       avg(similar.rating) AS avgRating
ORDER BY recommenders DESC, avgRating DESC
LIMIT 10
```

**2. Conteúdos similares por gênero e creator:**
```cypher
MATCH (c:Content {id: $contentId})-[:IN_GENRE]->(g:Genre)<-[:IN_GENRE]-(similar:Content)
WHERE c <> similar
WITH similar, count(g) AS sharedGenres
OPTIONAL MATCH (c)-[:CREATED_BY]->(cr:Creator)<-[:CREATED_BY]-(similar)
RETURN similar.title, similar.contentType,
       sharedGenres,
       count(cr) AS sharedCreators,
       (sharedGenres * 2 + count(cr) * 3) AS similarityScore
ORDER BY similarityScore DESC
LIMIT 10
```

**3. Shortest Path entre dois usuários (grau de separação):**
```cypher
MATCH path = shortestPath(
  (a:User {id: $userId1})-[:WATCHED*..6]-(b:User {id: $userId2})
)
RETURN [n IN nodes(path) | 
  CASE WHEN n:User THEN n.name 
       WHEN n:Content THEN n.title 
  END
] AS connection,
length(path) AS degrees
```

**4. Trending: conteúdos mais assistidos na última semana:**
```cypher
MATCH (u:User)-[w:WATCHED]->(c:Content)
WHERE w.watchedAt > datetime() - duration('P7D')
WITH c, count(u) AS weeklyViews, avg(w.rating) AS avgRating
RETURN c.title, c.contentType, weeklyViews, avgRating,
       (weeklyViews * 0.7 + avgRating * 0.3 * 100) AS trendScore
ORDER BY trendScore DESC
LIMIT 20
```

**5. Community detection: gêneros favoritos do usuário:**
```cypher
MATCH (u:User {id: $userId})-[w:WATCHED]->(c:Content)-[:IN_GENRE]->(g:Genre)
RETURN g.name AS genre,
       count(c) AS contentWatched,
       avg(w.rating) AS avgRating,
       sum(w.completionPct) / count(c) AS avgCompletion
ORDER BY contentWatched DESC
```

- Implementar acesso ao Neo4j em Java e Go:

**Java 25 (Neo4j Driver):**
```java
public List<Recommendation> getCollaborativeRecommendations(String userId) {
    var query = """
        MATCH (target:User {id: $userId})-[:WATCHED]->(c:Content)<-[:WATCHED]-(similar:User)
        WHERE target <> similar
        WITH similar, count(c) AS commonContent WHERE commonContent >= 3
        MATCH (similar)-[w:WATCHED]->(rec:Content)
        WHERE NOT (target)-[:WATCHED]->(rec)
        RETURN rec.id AS id, rec.title AS title, rec.contentType AS type,
               count(DISTINCT similar) AS recommenders, avg(w.rating) AS avgRating
        ORDER BY recommenders DESC, avgRating DESC LIMIT 10
        """;

    try (var session = driver.session()) {
        return session.executeRead(tx -> {
            var result = tx.run(query, Map.of("userId", userId));
            return result.list(r -> new Recommendation(
                r.get("id").asString(),
                r.get("title").asString(),
                r.get("type").asString(),
                r.get("recommenders").asLong(),
                r.get("avgRating").asDouble()
            ));
        });
    }
}
```

**Go 1.26 (Neo4j Driver):**
```go
func (r *RecommendationRepo) GetCollaborative(ctx context.Context, userID string) ([]Recommendation, error) {
    query := `
        MATCH (target:User {id: $userId})-[:WATCHED]->(c:Content)<-[:WATCHED]-(similar:User)
        WHERE target <> similar
        WITH similar, count(c) AS commonContent WHERE commonContent >= 3
        MATCH (similar)-[w:WATCHED]->(rec:Content)
        WHERE NOT (target)-[:WATCHED]->(rec)
        RETURN rec.id AS id, rec.title AS title, rec.contentType AS type,
               count(DISTINCT similar) AS recommenders, avg(w.rating) AS avgRating
        ORDER BY recommenders DESC, avgRating DESC LIMIT 10`

    result, err := neo4j.ExecuteQuery(ctx, r.driver, query,
        map[string]any{"userId": userID},
        neo4j.EagerResultTransformer,
        neo4j.WithReadAccess(),
    )
    if err != nil {
        return nil, fmt.Errorf("collaborative filtering: %w", err)
    }

    recs := make([]Recommendation, 0, len(result.Records))
    for _, record := range result.Records {
        id, _ := record.Get("id")
        title, _ := record.Get("title")
        recs = append(recs, Recommendation{
            ID:    id.(string),
            Title: title.(string),
        })
    }
    return recs, nil
}
```

**Critérios de aceite:**

- [ ] 5 queries Cypher implementadas
- [ ] Collaborative filtering funcional
- [ ] `shortestPath` demonstrado
- [ ] Temporal filter com `duration()` em Cypher
- [ ] Driver Neo4j em Java e Go funcional
- [ ] Performance: queries retornam em < 100ms com dados seed

---

### Desafio 5.6 — NoSQL Anti-Patterns no Contexto StreamX

**Contexto:** Identifique anti-patterns NoSQL que a equipe StreamX pode cometer
e implemente as correções.

**Requisitos:**

- Corrigir 6 anti-patterns:

**1. Cassandra: Unbounded Partition**
```sql
-- ❌ Partition cresce indefinidamente
PRIMARY KEY (content_id, event_timestamp)

-- ✅ Bucket por hora
PRIMARY KEY ((content_id, event_hour), event_timestamp)
```

**2. MongoDB: Massive Array**
```javascript
// ❌ Array que cresce sem limite
{ userId: "...", allWatchedIds: [/* milhares */] }

// ✅ Coleção separada com referência
// Collection: viewing_history
{ userId: "...", contentId: "...", watchedAt: ISODate("...") }
```

**3. Neo4j: Dense Node (Supernode)**
```cypher
// ❌ Node com milhões de relationships
(:Content {title: "Top 10 Film"})-[:WATCHED]-> (10M users)

// ✅ Meta-nodes para fragmentar
(:Content {title: "Top 10 Film"})-[:HAS_BUCKET]->
  (:WatchBucket {month: "2025-01"})-[:WATCHED_BY]-> (subset of users)
```

**4. Using NoSQL as SQL (Normalização em Document Store)**
```javascript
// ❌ Normalizar em MongoDB como se fosse RDBMS
// Coleção: videos, coleção: categories, coleção: video_categories
// → Sem JOIN nativo, múltiplas queries

// ✅ Embed dados frequentemente lidos juntos
{ title: "...", categories: ["Sci-Fi", "Drama"] }
```

**5. Cassandra: Scan em vez de Query**
```sql
-- ❌ Full table scan (ALLOW FILTERING)
SELECT * FROM watch_events_by_user WHERE content_id = ?
  ALLOW FILTERING;

-- ✅ Tabela dedicada para esse access pattern
SELECT * FROM watch_events_by_content
  WHERE content_id = ? AND event_hour = ?;
```

**6. Redis: Sem TTL**
```
// ❌ Cache sem expiração → memória esgota
SET content:123 "{...}"

// ✅ Sempre com TTL
SETEX content:123 3600 "{...}"
```

**Critérios de aceite:**

- [ ] 6 anti-patterns identificados com causa raiz
- [ ] Código "antes" e "depois" para cada anti-pattern
- [ ] Cada correção aplicada ao domínio StreamX
- [ ] Impacto mensurável de cada anti-pattern (latência, storage, custo)
- [ ] Documento: `decisions/05-nosql-anti-patterns.md`

---

### Desafio 5.7 — Decisão Arquitetural: Cassandra vs TimescaleDB

**Contexto:** Para watch events de time-series, compare Cassandra vs TimescaleDB
(extensão PostgreSQL). A equipe StreamX precisa decidir.

**Requisitos:**

- Comparar para o caso de uso StreamX:

| Critério | Cassandra | TimescaleDB |
|----------|-----------|-------------|
| Modelo | Wide-Column | Relacional + hypertables |
| Ingestão | Excelente (TWCS) | Boa (compressão nativa) |
| Queries temporais | Limitadas (partition key) | SQL completo (window functions) |
| Joins | Não suportado | SQL nativo |
| Scaling | Linear, multi-DC | PostgreSQL replication |
| Compressão | Por SSTable (TWCS) | Colunar (até 95%) |
| Operacional | Complexo (ring, repairs) | Simples (extensão PG) |
| Downsampling | Manual (batch jobs) | Continuous aggregates |
| Ecossistema | CQL, drivers próprios | SQL padrão, pg ecosystem |

- Implementar watch events em TimescaleDB para comparação:

```sql
-- TimescaleDB: criar hypertable
CREATE TABLE watch_events (
    event_time    TIMESTAMPTZ NOT NULL,
    user_id       UUID NOT NULL,
    content_id    UUID NOT NULL,
    event_type    TEXT NOT NULL,
    position_ms   BIGINT,
    duration_ms   BIGINT,
    device_type   TEXT,
    quality       TEXT
);

SELECT create_hypertable('watch_events', 'event_time',
    chunk_time_interval => INTERVAL '1 day');

-- Continuous aggregate (downsampling automático)
CREATE MATERIALIZED VIEW hourly_content_metrics
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', event_time) AS bucket,
    content_id,
    count(*) AS total_events,
    count(DISTINCT user_id) AS unique_viewers,
    avg(duration_ms) AS avg_duration_ms
FROM watch_events
GROUP BY bucket, content_id;

-- Retention policy (auto-delete old data)
SELECT add_retention_policy('watch_events', INTERVAL '90 days');

-- Compression policy
ALTER TABLE watch_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'content_id',
    timescaledb.compress_orderby = 'event_time DESC'
);
SELECT add_compression_policy('watch_events', INTERVAL '7 days');
```

- Recomendação justificada:

```
Recomendação para StreamX:
├── Watch events (ingestão): Cassandra
│   └── Justificativa: 100K+ writes/segundo, multi-DC, tolerante a falhas
├── Analytics (queries): TimescaleDB
│   └── Justificativa: SQL window functions, continuous aggregates, JOINs
└── Alternativa: TimescaleDB-only para times menores
    └── Se < 50K writes/segundo, TimescaleDB é mais simples operacionalmente
```

**Critérios de aceite:**

- [ ] Tabela comparativa com 9+ critérios técnicos
- [ ] Mesma entidade implementada em ambos (DDL + insert + query)
- [ ] Continuous aggregate demonstrado no TimescaleDB
- [ ] Recomendação com threshold de decisão (volume de escrita)
- [ ] Documento: `decisions/05-cassandra-vs-timescaledb.md`
