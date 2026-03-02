# Level 4 — NoSQL: Document & Key-Value (MongoDB, Redis, DynamoDB)

> **Objetivo:** Dominar modelagem NoSQL orientada a access patterns — Document Stores (MongoDB),
> Key-Value Stores (Redis, DynamoDB), embedding vs referencing, e caching patterns.

**Referência:** [.docs/databases/03-nosql-best-practices.md](../../.docs/databases/03-nosql-best-practices.md)

---

## Contexto do Domínio

O catálogo de conteúdo da StreamX tem metadata flexível: filmes têm `director`, `cast`, `duration`;
séries têm `seasons`, `episodes`; músicas têm `artist`, `album`, `track_number`; podcasts têm `host`, `guests`.
Um schema relacional rígido não funciona. Além disso, sessões de reprodução, cache de conteúdo popular
e rate limiting precisam de Key-Value com latência sub-millisecond.

---

## Desafios

### Desafio 4.1 — Access Patterns First: Modelagem do Catálogo

**Contexto:** Antes de modelar, liste todos os access patterns do catálogo StreamX.

**Requisitos:**

- Listar pelo menos 8 access patterns do catálogo:

| # | Access Pattern | Frequência | Latência Alvo |
|---|---------------|------------|---------------|
| 1 | Buscar conteúdo por ID | Muito Alta | < 10ms |
| 2 | Listar conteúdos por gênero (recentes primeiro) | Alta | < 50ms |
| 3 | Buscar conteúdo por título (full-text) | Alta | < 100ms |
| 4 | Listar episódios de uma série | Alta | < 20ms |
| 5 | Top 10 conteúdos mais assistidos (por semana) | Muito Alta | < 10ms |
| 6 | Conteúdos adicionados recentemente | Alta | < 50ms |
| 7 | Conteúdos por content creator | Média | < 50ms |
| 8 | Buscar review de um conteúdo | Média | < 50ms |

- Para cada access pattern, indicar:
  - Dados retornados (quais campos)
  - Frequência de acesso (leituras/segundo estimadas)
  - Se é um read ou write pattern
- Agrupar dados acessados juntos (base para embedding)

**Critérios de aceite:**

- [ ] 8+ access patterns documentados com latência alvo
- [ ] Cada pattern tem campos de retorno especificados
- [ ] Agrupamento de dados para decisão de embedding vs referencing
- [ ] Documento: `decisions/04-access-patterns.md`

---

### Desafio 4.2 — Modelagem MongoDB: Embedding Inteligente

**Contexto:** Modele o catálogo da StreamX em MongoDB aplicando os patterns de embedding corretos.

**Requisitos:**

- Implementar o document model para `content`:

```javascript
// ✅ BOM: Content document com embedding inteligente
{
  _id: ObjectId("..."),
  externalId: "content-uuid-001",
  title: "Interstellar",
  contentType: "MOVIE",  // MOVIE, SERIES, MUSIC, PODCAST

  // Extended Reference — dados do creator lidos com o conteúdo
  creators: [
    { id: ObjectId("creator-nolan"), name: "Christopher Nolan", role: "Director" },
    { id: ObjectId("creator-mcconaughey"), name: "Matthew McConaughey", role: "Actor" }
  ],

  // Embedded Array — gêneros são poucos e lidos sempre
  genres: ["Sci-Fi", "Drama", "Adventure"],

  // Metadata flexível por tipo (benefício do Document Store)
  metadata: {
    duration_minutes: 169,
    release_year: 2014,
    rating: "PG-13",
    audio_languages: ["en", "pt-BR", "es"],
    subtitle_languages: ["pt-BR", "en", "es", "fr"],
    resolution: "4K",
    hdr: true
  },

  // Computed Pattern — pré-calculados para evitar aggregation
  stats: {
    view_count: 15420000,
    avg_rating: NumberDecimal("4.7"),
    review_count: 89500
  },

  // Subset Pattern — top 3 reviews embeddados, demais em collection separada
  top_reviews: [
    {
      userId: ObjectId("user-1"),
      userName: "João",
      rating: 5,
      comment: "Obra-prima da ficção científica",
      createdAt: ISODate("2025-01-10")
    }
  ],

  isPublished: true,
  publishedAt: ISODate("2014-11-07"),
  createdAt: ISODate("2025-01-01"),
  updatedAt: ISODate("2025-01-15")
}
```

- Implementar document para `series` (com episodes embeddados):

```javascript
// Série com Bucket Pattern para episódios
{
  _id: ObjectId("..."),
  title: "Stranger Things",
  contentType: "SERIES",
  seasons: [
    {
      number: 1,
      title: "Season 1",
      releaseYear: 2016,
      episodes: [
        { number: 1, title: "The Vanishing of Will Byers", duration_minutes: 47 },
        { number: 2, title: "The Weirdo on Maple Street", duration_minutes: 55 }
        // ... máximo ~20 episódios por temporada (bounded)
      ]
    }
  ],
  // ... genres, metadata, stats
}
```

- Demonstrar anti-pattern: **Unbounded Array**

```javascript
// ❌ RUIM: Array que cresce infinitamente
{
  _id: ObjectId("user-1"),
  name: "João",
  allWatchedContent: [/* milhares de entries → 16MB limit */]
}

// ✅ CORREÇÃO: Coleção separada referenciada
// Collection: viewing_history
{ userId: ObjectId("user-1"), contentId: ObjectId("..."), watchedAt: ISODate("...") }
```

**Critérios de aceite:**

- [ ] Document model com pelo menos 3 embedding patterns (Extended Reference, Subset, Computed)
- [ ] Metadata flexível por content type
- [ ] `NumberDecimal` para ratings/valores (não `Double`)
- [ ] `ISODate` para timestamps (não strings)
- [ ] Demonstração do anti-pattern Unbounded Array com solução
- [ ] Schema validation definido na collection

---

### Desafio 4.3 — MongoDB: Indexação & Aggregation Pipeline

**Contexto:** O catálogo da StreamX tem 500K+ conteúdos. Queries precisam de índices
eficientes e aggregations otimizadas.

**Requisitos:**

- Criar estratégia de índices para MongoDB:

```javascript
// 1. Compound index (ESR) — buscar por gênero, ordenar por views
db.content.createIndex(
  { genres: 1, "stats.view_count": -1 },
  { name: "idx_genre_views" }
);

// 2. Partial index — apenas conteúdos publicados
db.content.createIndex(
  { contentType: 1, publishedAt: -1 },
  {
    name: "idx_published_content",
    partialFilterExpression: { isPublished: true }
  }
);

// 3. Text index para full-text search
db.content.createIndex(
  { title: "text", "metadata.description": "text" },
  { weights: { title: 10, "metadata.description": 5 }, name: "idx_search" }
);

// 4. TTL index para sessões temporárias
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600, name: "idx_session_ttl" }
);

// 5. Wildcard para metadata queries dinâmicas
db.content.createIndex({ "metadata.$**": 1 });
```

- Implementar Aggregation Pipeline otimizado:

```javascript
// Top 10 conteúdos por gênero com estatísticas
db.content.aggregate([
  // 1. $match PRIMEIRO (usa índice)
  { $match: { isPublished: true, genres: "Sci-Fi" } },

  // 2. $project apenas campos necessários
  { $project: {
    title: 1,
    contentType: 1,
    "stats.view_count": 1,
    "stats.avg_rating": 1,
    releaseYear: "$metadata.release_year"
  }},

  // 3. $sort por views
  { $sort: { "stats.view_count": -1 } },

  // 4. $limit
  { $limit: 10 },

  // 5. $addFields para computed
  { $addFields: {
    popularity_score: {
      $add: [
        { $multiply: ["$stats.view_count", 0.7] },
        { $multiply: ["$stats.avg_rating", 0.3] }
      ]
    }
  }}
]);
```

**Critérios de aceite:**

- [ ] 5 tipos de índice criados (compound, partial, text, TTL, wildcard)
- [ ] Aggregation pipeline com `$match` primeiro (demonstrar com `explain()`)
- [ ] Full-text search funcional com ranking
- [ ] TTL index demonstrado (documents expiram automaticamente)  
- [ ] Benchmark: query com vs sem índice

---

### Desafio 4.4 — DynamoDB: Single-Table Design

**Contexto:** O histórico de visualizações da StreamX precisa de escala massiva e
acesso por partition key. Modele em DynamoDB com Single-Table Design.

**Requisitos:**

- Implementar Single-Table Design para viewing history:

```
PK (Partition Key)       | SK (Sort Key)                     | Dados
─────────────────────────┼───────────────────────────────────┼─────────────
USER#user-123            | PROFILE                            | name, email, plan
USER#user-123            | HISTORY#2025-01-15#content-001    | contentTitle, duration, progress
USER#user-123            | HISTORY#2025-01-14#content-002    | contentTitle, duration, progress
USER#user-123            | WATCHLIST#content-003              | addedAt, priority
CONTENT#content-001      | DETAILS                            | title, type, duration
CONTENT#content-001      | STATS#DAILY#2025-01-15            | views, uniqueViewers, avgDuration
```

- Access patterns resolvidos:
  1. `PK = USER#123, SK = PROFILE` → dados do usuário
  2. `PK = USER#123, SK begins_with("HISTORY#")` → histórico de visualizações
  3. `PK = USER#123, SK begins_with("WATCHLIST#")` → lista de "assistir depois"
  4. `PK = CONTENT#001, SK = DETAILS` → detalhes do conteúdo
  5. GSI: `GSI1PK = CONTENT#001, GSI1SK = created_at` → últimas visualizações de um conteúdo

- Implementar em Java e Go:

**Java 25:**
```java
// DynamoDB — Buscar histórico do usuário
var request = QueryRequest.builder()
    .tableName("streamx")
    .keyConditionExpression("PK = :pk AND begins_with(SK, :sk_prefix)")
    .expressionAttributeValues(Map.of(
        ":pk", AttributeValue.builder().s("USER#" + userId).build(),
        ":sk_prefix", AttributeValue.builder().s("HISTORY#").build()
    ))
    .scanIndexForward(false)  // mais recentes primeiro
    .limit(20)
    .build();

var result = dynamoClient.query(request);
```

**Go 1.26:**
```go
input := &dynamodb.QueryInput{
    TableName:              aws.String("streamx"),
    KeyConditionExpression: aws.String("PK = :pk AND begins_with(SK, :sk_prefix)"),
    ExpressionAttributeValues: map[string]types.AttributeValue{
        ":pk":        &types.AttributeValueMemberS{Value: "USER#" + userID},
        ":sk_prefix": &types.AttributeValueMemberS{Value: "HISTORY#"},
    },
    ScanIndexForward: aws.Bool(false),
    Limit:            aws.Int32(20),
}
result, err := client.Query(ctx, input)
```

**Critérios de aceite:**

- [ ] Single-Table Design com pelo menos 3 entity types
- [ ] Partition key com alta cardinalidade (user_id)
- [ ] 5+ access patterns resolvidos com Query (não Scan)
- [ ] GSI para access pattern secundário
- [ ] Implementação em Java e Go funcional
- [ ] Demonstrar: `FilterExpression` NÃO reduz RCUs (documentar trade-off)

---

### Desafio 4.5 — Redis: Caching Patterns para StreamX

**Contexto:** A StreamX precisa de cache para conteúdo popular, sessões de reprodução,
rate limiting e leaderboards.

**Requisitos:**

- Implementar 4 patterns com Redis:

**1. Cache-Aside para catálogo de conteúdo:**

```python
# Pseudocode / aplicar em Java e Go

def get_content(content_id: str) -> dict:
    # 1. Tentar cache
    cached = redis.get(f"content:{content_id}")
    if cached:
        return json.loads(cached)

    # 2. Cache miss → buscar no MongoDB
    content = mongo.content.find_one({"_id": ObjectId(content_id)})

    # 3. Popular cache com TTL de 1 hora
    redis.setex(f"content:{content_id}", 3600, json.dumps(content))
    return content
```

**2. Session State para reprodução:**

```java
// Java — Salvar estado de reprodução
public void savePlaybackState(String userId, String contentId, PlaybackState state) {
    var key = "playback:%s:%s".formatted(userId, contentId);
    var hash = Map.of(
        "position_ms", String.valueOf(state.positionMs()),
        "speed", String.valueOf(state.speed()),
        "audio_track", state.audioTrack(),
        "subtitle", state.subtitle(),
        "updated_at", Instant.now().toString()
    );
    jedis.hset(key, hash);
    jedis.expire(key, 86400); // TTL: 24 horas
}
```

**3. Rate Limiting (Sliding Window):**

```go
// Go — Rate limiter com Sorted Set
func (rl *RateLimiter) IsAllowed(ctx context.Context, userID string, maxReqs int, windowSecs int) (bool, error) {
    key := fmt.Sprintf("ratelimit:%s", userID)
    now := float64(time.Now().UnixMilli())
    windowStart := now - float64(windowSecs*1000)

    pipe := rl.redis.Pipeline()
    pipe.ZRemRangeByScore(ctx, key, "0", fmt.Sprintf("%f", windowStart))
    pipe.ZAdd(ctx, key, redis.Z{Score: now, Member: fmt.Sprintf("%f", now)})
    countCmd := pipe.ZCard(ctx, key)
    pipe.Expire(ctx, key, time.Duration(windowSecs)*time.Second)
    if _, err := pipe.Exec(ctx); err != nil {
        return false, err
    }
    return countCmd.Val() <= int64(maxReqs), nil
}
```

**4. Leaderboard (Top Conteúdos):**

```go
// Atualizar views (incrementar score)
func (lb *Leaderboard) IncrementViews(ctx context.Context, contentID string) error {
    return lb.redis.ZIncrBy(ctx, "leaderboard:weekly", 1, contentID).Err()
}

// Top 10 da semana
func (lb *Leaderboard) TopContent(ctx context.Context, n int64) ([]redis.Z, error) {
    return lb.redis.ZRevRangeWithScores(ctx, "leaderboard:weekly", 0, n-1).Result()
}
```

**Critérios de aceite:**

- [ ] Cache-Aside implementado com TTL (Java e Go)
- [ ] Session state com Hash (`HSET/HGET`) e TTL
- [ ] Rate limiter com Sliding Window (Sorted Set)
- [ ] Leaderboard com Sorted Set (`ZINCRBY`, `ZREVRANGE`)
- [ ] Demonstrar cache miss → DB → populate → cache hit
- [ ] Demonstrar rate limiting bloqueando excesso de requests

---

### Desafio 4.6 — Redis: Problemas de Cache e Soluções

**Contexto:** Caching parece simples, mas tem problemas clássicos que podem derrubar a StreamX.

**Requisitos:**

- Implementar soluções para 4 problemas:

**1. Cache Stampede (Thundering Herd):**
```java
// Problema: TTL expira → 1000 requests simultâneos ao MongoDB
// Solução: Lock — apenas 1 request reconstrói o cache
public Content getContentWithLock(String contentId) {
    var key = "content:" + contentId;
    var cached = redis.get(key);
    if (cached != null) return deserialize(cached);

    var lockKey = "lock:content:" + contentId;
    var lockAcquired = redis.set(lockKey, "1", SetParams.setParams().nx().ex(10));

    if (lockAcquired) {
        try {
            var content = mongodb.findContent(contentId);
            redis.setex(key, 3600, serialize(content));
            return content;
        } finally {
            redis.del(lockKey);
        }
    } else {
        // Outro thread está reconstruindo — aguardar ou retornar stale
        Thread.sleep(100);
        return getContentWithLock(contentId); // retry
    }
}
```

**2. Cache Penetration:**
```go
// Problema: requests para IDs inexistentes sempre dão cache miss → DB hit
// Solução: cachear null results com TTL curto
func (c *ContentCache) Get(ctx context.Context, id string) (*Content, error) {
    cached, err := c.redis.Get(ctx, "content:"+id).Result()
    if err == nil {
        if cached == "NULL" { return nil, nil } // cached null
        return deserialize(cached), nil
    }
    content, err := c.mongo.FindContent(ctx, id)
    if err != nil { return nil, err }
    if content == nil {
        c.redis.Set(ctx, "content:"+id, "NULL", 5*time.Minute) // cache null com TTL curto
        return nil, nil
    }
    c.redis.Set(ctx, "content:"+id, serialize(content), 1*time.Hour)
    return content, nil
}
```

**3. Cache Avalanche:**
- TTL com jitter: `ttl = baseTTL + random(0, jitterRange)`

**4. Hot Key:**
- Local cache (L1) + Redis (L2): conteúdos populares cacheados em memória do app

**Critérios de aceite:**

- [ ] Cache Stampede: lock-based rebuild demonstrado
- [ ] Cache Penetration: null caching com TTL curto
- [ ] Cache Avalanche: TTL com jitter implementado
- [ ] Hot Key: cache L1 (in-memory) + L2 (Redis) demonstrado
- [ ] Teste: simular 100 requests simultâneos — apenas 1 chega ao DB (stampede)
- [ ] Documento: `decisions/04-caching-strategy.md`

---

### Desafio 4.7 — Schema Validation & Evolution em NoSQL

**Contexto:** "Schemaless" não significa "sem regras". A StreamX precisa de schema validation
em MongoDB e estratégias de evolução.

**Requisitos:**

- Implementar JSON Schema validation em MongoDB:

```javascript
db.createCollection("content", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["title", "contentType", "isPublished", "createdAt"],
      properties: {
        title: { bsonType: "string", minLength: 1, maxLength: 500 },
        contentType: { enum: ["MOVIE", "SERIES", "MUSIC", "PODCAST"] },
        isPublished: { bsonType: "bool" },
        "stats.avg_rating": { bsonType: "decimal", minimum: 0, maximum: 5 },
        genres: {
          bsonType: "array",
          items: { bsonType: "string" },
          maxItems: 10
        },
        createdAt: { bsonType: "date" }
      }
    }
  },
  validationLevel: "moderate",    // valida updates, não existing documents
  validationAction: "error"       // rejeita documentos inválidos
});
```

- Implementar schema evolution com versioning:

```javascript
// Schema v1
{ schemaVersion: 1, title: "Interstellar", genre: "Sci-Fi" }

// Schema v2 — genre virou array
{ schemaVersion: 2, title: "Interstellar", genres: ["Sci-Fi", "Drama"] }

// Aplicação trata ambas versões:
function normalizeGenres(doc) {
  if (doc.schemaVersion === 1) {
    return [doc.genre]; // migração lazy
  }
  return doc.genres;
}
```

- Implementar lazy migration: migrar documents on-read

**Critérios de aceite:**

- [ ] JSON Schema validation configurado na collection
- [ ] Teste: insert de document inválido é rejeitado
- [ ] Schema versioning implementado com campo `schemaVersion`
- [ ] Lazy migration: read de v1 converte para v2 e salva
- [ ] Documento: `decisions/04-schema-evolution.md`

---

### Desafio 4.8 — MongoDB vs DynamoDB: Trade-offs

**Contexto:** Justifique quando usar MongoDB vs DynamoDB no ecossistema StreamX.

**Requisitos:**

- Criar tabela comparativa:

| Critério | MongoDB | DynamoDB |
|----------|---------|----------|
| Modelo de dados | Document (BSON, 16MB max) | Key-Value/Document (400KB max) |
| Query flexibility | Rica (ad-hoc, aggregation) | Limitada (pk/sk, GSI) |
| Scaling | Sharding manual ou Atlas | Auto-scaling nativo |
| Consistency | Tunable (majority, linearizable) | Eventually/Strongly per-query |
| Pricing | Instance-based ou Atlas serverless | Pay-per-request ou provisioned |
| Indexação | Compound, text, geo, wildcard | LSI, GSI (max 20) |
| Transactions | ACID multi-document | ACID transactions (2x custo) |
| Operations | Self-managed ou Atlas | Fully managed (zero ops) |

- Justificar escolha para StreamX:
  - Catálogo → MongoDB (queries flexíveis, aggregation, full-text search)
  - Viewing History → DynamoDB (escala auto, access pattern fixo, partition key)
- Implementar a mesma entidade em ambos e comparar

**Critérios de aceite:**

- [ ] Tabela comparativa com 8+ critérios
- [ ] Justificativa de escolha para catálogo e viewing history
- [ ] Mesma entidade (`ViewingHistory`) implementada em ambos
- [ ] Trade-off explícito: flexibilidade (MongoDB) vs escala (DynamoDB)
- [ ] Documento: `decisions/04-mongodb-vs-dynamodb.md`
