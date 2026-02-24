# 38. Twitter / X — Social Feed

> **Categoria:** Classic System Design  
> **Nível:** Uma das perguntas mais frequentes — Google, Meta, Amazon, Twitter  
> **Complexidade:** Alta

---

## Definição

Design de um **Social Feed System** como Twitter/X, focando no problema central: **como servir o feed (timeline) de milhões de usuários em tempo real**. O core challenge é o **fan-out** — quando um usuário posta, como entregar esse conteúdo para todos os seguidores eficientemente?

---

## Requisitos

### Funcionais

```
1. Postar tweets (texto até 280 chars + mídia)
2. Seguir/deixar de seguir outros usuários
3. Home timeline: ver tweets de quem você segue (cronológico + ranking)
4. User timeline: ver tweets de um usuário específico
5. Like, retweet, reply
6. Search (por texto, hashtag, user)
7. Trending topics
```

### Não-Funcionais

```
1. Baixa latência: timeline carrega em < 200ms
2. Alta disponibilidade: 99.99%
3. Eventual consistency é aceitável (feed pode ter 1-2s delay)
4. Read-heavy: 100:1 read:write ratio
5. Escala: 300M DAU, 500M tweets/dia
```

---

## Estimativas

```
Users:
  Total: 500M users
  DAU: 300M
  QPS Tweets: 500M / 86,400 ≈ 6,000 QPS (write)
  QPS Timeline reads: 300M × 10 reads/dia = 3B / 86,400 ≈ 35,000 QPS (read)
  Peak: ~100,000 read QPS

Storage:
  Tweet: ~280 chars + metadata ≈ 500 bytes
  500M × 500B = 250 GB/dia
  Com mídia: muito mais (fotos, vídeos → S3 + CDN)

Social Graph:
  Avg followers: 200
  Total follow relationships: 500M × 200 = 100B edges
  Storage: 100B × 16 bytes (2 IDs) = 1.6 TB
```

---

## O Problema Central: Fan-out

### Abordagem 1: Fan-out on Write (Push Model)

```
Quando user posta tweet → PRÉ-DISTRIBUI para timeline de CADA seguidor

  @elonmusk posta tweet (150M followers):
  
  ┌─────────────┐     ┌──────────────────────────────────────────────┐
  │ Elon posts   │────▶│  Fan-out Service                            │
  │ "Hello World"│     │                                              │
  └─────────────┘     │  Para CADA follower:                         │
                      │    INSERT INTO timeline_cache                │
                      │    (user_id=follower, tweet_id, timestamp)   │
                      │                                              │
                      │  Follower 1: [tweet_elon, ...]              │
                      │  Follower 2: [tweet_elon, ...]              │
                      │  Follower 3: [tweet_elon, ...]              │
                      │  ...                                         │
                      │  Follower 150M: [tweet_elon, ...]           │
                      └──────────────────────────────────────────────┘

  Read (quando seguidor abre timeline):
  GET timeline_cache[user_id] → pronto! Já está pré-computado ✅

  ✅ Read é EXTREMAMENTE rápido (pré-computado, O(1))
  ✅ Timeline load = 1 cache lookup
  ❌ Write é LENTO para usuários com muitos followers
  ❌ @elonmusk: 150M inserts POR TWEET! (minutos de delay)
  ❌ Desperdício: muitos followers nunca abrem o app
  ❌ Storage: N tweets × M followers = explosão
```

### Abordagem 2: Fan-out on Read (Pull Model)

```
Quando user abre timeline → BUSCA tweets de quem segue em real-time

  User abre app:
  
  ┌─────────────┐     ┌──────────────────────────────────────────────┐
  │ User opens   │────▶│  Timeline Service                           │
  │ home feed    │     │                                              │
  └─────────────┘     │  1. Buscar quem user segue:                  │
                      │     following = [elon, gates, cook, ...]     │
                      │  2. Para CADA following:                      │
                      │     tweets = GET recent_tweets(following_id) │
                      │  3. Merge + Sort by timestamp/ranking        │
                      │  4. Return top N tweets                      │
                      └──────────────────────────────────────────────┘

  ✅ Write é INSTANTÂNEO (apenas salva o tweet, sem fan-out)
  ✅ Sem desperdício (só busca quando user quer)
  ❌ Read é LENTO (precisa consultar N fontes + merge)
  ❌ Se user segue 500 pessoas → 500 queries + merge!
  ❌ Latência = f(num_following) → pode ser alto
```

### Abordagem 3: Hybrid (O que o Twitter REALMENTE usa)

```
  ┌────────────────────────────────────────────────────────────────┐
  │  HYBRID MODEL                                                   │
  │                                                                 │
  │  Usuários NORMAIS (< 10K followers):                           │
  │    → Fan-out on Write (push)                                    │
  │    → Tweet pré-distribuído para timeline cache dos followers    │
  │                                                                 │
  │  CELEBRIDADES (> 10K followers):                                │
  │    → Fan-out on Read (pull)                                     │
  │    → Tweets buscados em real-time quando follower abre feed    │
  │                                                                 │
  │  Timeline final:                                                │
  │    = Pre-computed cache (normal users)                          │
  │    + Real-time merge (celebrity tweets)                         │
  │    + Ranking algorithm                                          │
  └────────────────────────────────────────────────────────────────┘

  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐
  │ Normal Users  │    │ Celebrities  │    │  Timeline        │
  │ (push tweets │    │ (tweets stay │    │  Mixer/Ranker    │
  │  to followers│    │  in tweet DB)│    │                  │
  │  cache)      │    │              │    │  Merge + Sort +  │
  └──────┬───────┘    └──────┬───────┘    │  Rank + Filter   │
         │                   │            └────────┬─────────┘
         ▼                   ▼                     │
  ┌─────────────────────────────────┐              │
  │  User's Timeline Cache (Redis)  │◀─────────────┘
  │  pre-computed + celebrity merge  │
  └─────────────────────────────────┘
```

---

## Arquitetura Completa

```
  ┌────────────────────────────────────────────────────────────────────┐
  │                                                                    │
  │  ┌────────┐    ┌──────────────┐                                    │
  │  │ Client │───▶│ API Gateway  │───▶ Rate Limit, Auth              │
  │  └────────┘    └──────┬───────┘                                    │
  │                       │                                            │
  │  ┌────────────────────┼──────────────────────────────┐             │
  │  │                    │                              │             │
  │  ▼                    ▼                              ▼             │
  │ ┌──────────┐   ┌──────────────┐              ┌─────────────┐      │
  │ │ Tweet    │   │  Timeline    │              │ Social Graph│      │
  │ │ Service  │   │  Service     │              │ Service     │      │
  │ │          │   │              │              │             │      │
  │ │ Create   │   │ Home feed    │              │ Follow/     │      │
  │ │ Delete   │   │ User feed    │              │ Unfollow    │      │
  │ │ Like     │   │ Ranking      │              │ Followers   │      │
  │ └────┬─────┘   └──────┬───────┘              │ Following   │      │
  │      │                │                      └──────┬──────┘      │
  │      │                │                             │              │
  │      ▼                ▼                             ▼              │
  │ ┌──────────┐   ┌──────────────┐              ┌─────────────┐      │
  │ │Tweet DB  │   │Timeline Cache│              │Graph DB     │      │
  │ │(Cassandra│   │  (Redis)     │              │(FlockDB/    │      │
  │ │ /MySQL)  │   │              │              │ TAO/Redis)  │      │
  │ └──────────┘   └──────────────┘              └─────────────┘      │
  │                                                                    │
  │  ┌────────────────────────────────────────────────────────────┐    │
  │  │  Fan-out Service (async via Kafka)                         │    │
  │  │                                                            │    │
  │  │  1. Tweet created → event to Kafka                         │    │
  │  │  2. Fan-out workers consume event                          │    │
  │  │  3. Check: is author celebrity? (>10K followers)           │    │
  │  │     YES → skip fan-out (will be pulled at read time)       │    │
  │  │     NO  → push to timeline cache of EACH follower          │    │
  │  └────────────────────────────────────────────────────────────┘    │
  │                                                                    │
  │  ┌────────────────────────────────────┐                            │
  │  │  Search & Trends (Elasticsearch)   │                            │
  │  │  - Full-text search               │                            │
  │  │  - Hashtag trending               │                            │
  │  │  - Real-time indexing via Kafka    │                            │
  │  └────────────────────────────────────┘                            │
  │                                                                    │
  │  ┌────────────────────────────────────┐                            │
  │  │  Media Service                     │                            │
  │  │  - Image/Video upload → S3/CDN    │                            │
  │  │  - Thumbnail generation           │                            │
  │  │  - Video transcoding              │                            │
  │  └────────────────────────────────────┘                            │
  └────────────────────────────────────────────────────────────────────┘
```

---

## Componentes em Detalhe

### Tweet Storage

```
  Cassandra (ou MySQL sharded via Vitess):
  
  tweets table:
  ┌──────────┬──────────┬────────────┬───────────┬─────────┐
  │ tweet_id │ user_id  │ content    │ media_urls│ created │
  │ (snowflk)│          │ (280 chars)│ (JSON)    │         │
  ├──────────┼──────────┼────────────┼───────────┼─────────┤
  │ 17283... │ 12345    │ "Hello..." │ [img.jpg] │ ts      │
  └──────────┴──────────┴────────────┴───────────┴─────────┘
  
  Partition key: tweet_id (para lookup por ID)
  Secondary index: user_id (para user timeline)
  
  Tweet ID: Snowflake ID (64-bit, ~timestamp ordered)
  → Globally unique, roughly sortable by time
```

### Timeline Cache (Redis)

```
  Cada user tem uma sorted set (ZSET) no Redis:
  
  timeline:{user_id} → ZSET (score = timestamp)
  
  ZADD timeline:123 1700000001 tweet_id_1
  ZADD timeline:123 1700000050 tweet_id_2
  ZADD timeline:123 1700000100 tweet_id_3
  
  Home timeline = ZREVRANGE timeline:123 0 49  (top 50 tweets)
  
  Limit: manter apenas últimos 800 tweet_ids por timeline
  ZREMRANGEBYRANK timeline:123 0 -801  (remove oldest)
  
  Memory:
  300M DAU × 800 tweets × 8 bytes/id = 1.9 TB
  → Redis cluster com ~20-30 nodes (64GB each)
```

### Social Graph

```
  Follower relationships:
  
  Redis:
    followers:{user_id} → SET of follower IDs
    following:{user_id} → SET of following IDs
    
    SMEMBERS followers:elon → [user1, user2, ..., user150M]
    SCARD followers:elon → 150000000
    SISMEMBER followers:elon user123 → true/false

  Ou Graph DB (FlockDB — Twitter's original):
    (user_a) -[FOLLOWS]-> (user_b)
    
  Meta usa TAO (graph-aware cache over MySQL)
```

### Feed Ranking

```
  Chronological (original Twitter):
    ORDER BY created_at DESC
  
  Algorithmic (modern Twitter/X):
    score = f(relevance, recency, engagement, user_affinity)
    
    Fatores:
    - Recency: tweets recentes > antigos
    - Engagement: likes, retweets, replies
    - User affinity: interações passadas com o autor
    - Content type: vídeos, imagens, text-only
    - Author authority: verified, follower count
    
    ML Model: ranking model treinado em click/engagement data
    → Feature store + real-time inference
```

---

## Snowflake ID

```
  Twitter's distributed unique ID generator:
  
  64 bits:
  ┌─────────────────────────────────────────────────────────────┐
  │ 0 │ 41 bits: timestamp (ms) │ 10 bits: machine │ 12 bits:  │
  │   │ since epoch              │ ID                │ sequence  │
  └─────────────────────────────────────────────────────────────┘
  
  - Timestamp: ~69 years from custom epoch
  - Machine ID: 1024 machines
  - Sequence: 4096 IDs per millisecond per machine
  
  Total: ~4M IDs/second globally
  Sortable by time (timestamp is MSB)
  No coordination needed (each machine generates independently)
```

---

## Trade-offs

| Aspecto | Fan-out on Write | Fan-out on Read | Hybrid |
|---------|:----------------:|:---------------:|:------:|
| **Write latency** | 🔴 Alta (fan-out) | 🟢 Baixa | 🟡 Varia |
| **Read latency** | 🟢 Baixa (cached) | 🔴 Alta (compute) | 🟢 Baixa |
| **Storage** | 🔴 Alto (duplicação) | 🟢 Baixo | 🟡 Moderado |
| **Celebrity handling** | 🔴 150M inserts! | 🟢 On-demand | 🟢 Otimizado |
| **Complexidade** | 🟢 Simples | 🟡 Moderado | 🔴 Complexo |
| **Freshness** | 🟡 Lag possível | 🟢 Sempre fresh | 🟡 Hybrid |

---

## Uso em Big Techs

| Empresa | Arquitetura | Detalhes |
|---------|-----------|----------|
| **Twitter/X** | Hybrid fan-out + Snowflake IDs | Manhattan (KV store), Redis timeline cache |
| **Meta (News Feed)** | Pull-based + TAO graph cache | Ranking ML pipeline, News Feed aggregator |
| **Instagram** | Hybrid (similar Twitter) | Cassandra + Redis, Explore feed com ML |
| **LinkedIn** | Pull + precomputed feed | Feed AI ranking, real-time spark processing |
| **TikTok** | Recommendation-first (não follow-based) | Content-based vs social-based ranking |

---

## Perguntas Frequentes em Entrevistas

1. **"Fan-out on Write vs Fan-out on Read?"**
   - Write: rápido para ler, lento para postar (celebrities = death)
   - Read: rápido para postar, lento para ler (muitas queries)
   - Hybrid: melhor dos dois mundos (Twitter's approach)

2. **"Como lidar com celebrities/VIPs?"**
   - Não fazer fan-out on write para users com >N followers
   - Pull tweets de celebrities em real-time no read
   - Cache agressivo de celebrity tweets

3. **"Como gerar IDs únicos distribuídos?"**
   - Snowflake: timestamp + machine_id + sequence
   - ULID, KSUID: alternativas modernas
   - Sortable by time é crucial para timelines

4. **"Como funciona o ranking do feed?"**
   - ML model: engagement prediction (P(click), P(like), P(RT))
   - Features: recency, author affinity, content type, engagement
   - Balance: relevance vs chronology vs diversity

5. **"Como escalar o timeline cache?"**
   - Redis cluster: sharded por user_id
   - Manter apenas top 800 tweet_ids
   - Evict inactive user timelines (< 30 days)

---

## Referências

- Twitter Engineering — *"The Infrastructure Behind Twitter: Scale"*
- Raffi Krikorian — *"Timelines at Scale"* (QCon talk, 2012)
- Twitter Snowflake — github.com/twitter-archive/snowflake
- Alex Xu — *"System Design Interview"*, Cap. 11: Design a News Feed System
- Meta Engineering — *"Serving Facebook Multifeed"*
- ByteByteGo — *"Design a News Feed System"*
