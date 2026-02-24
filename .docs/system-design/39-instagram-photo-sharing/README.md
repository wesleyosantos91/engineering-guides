# 39. Instagram — Photo Sharing

> **Categoria:** Classic System Design  
> **Nível:** Pergunta frequente em entrevistas — Meta, Google, Amazon  
> **Complexidade:** Alta

---

## Definição

Design de um **sistema de compartilhamento de fotos** como Instagram, focando em: upload e processamento de imagens, feed com ranking, stories, e escala para **2 bilhões de usuários**. O desafio core é o **pipeline de mídia** (upload → processamento → storage → delivery via CDN) e o **feed ranking** para bilhões de posts.

---

## Requisitos

### Funcionais

```
1. Upload de fotos com caption e tags
2. Follow/unfollow outros usuários
3. Home feed: fotos de quem segue + recomendações
4. Like, comment, share
5. Stories (fotos/vídeos temporários — 24h)
6. Explore/Discover page
7. Search (users, hashtags, locations)
8. Direct Messages (DMs)
9. Notifications (likes, comments, follows)
```

### Não-Funcionais

```
1. Alta disponibilidade (99.99%)
2. Upload rápido (< 5s mesmo em redes lentas)
3. Feed load < 200ms
4. Escala: 500M DAU, 100M photos/dia
5. Consistência eventual é OK para feed
6. Strong consistency para likes/comments counts (eventual OK)
7. CDN para delivery global de imagens
```

---

## Estimativas

```
Photos:
  100M uploads/dia
  Cada photo → múltiplos tamanhos:
    Original: 3 MB
    Large (1080px): 500 KB
    Medium (640px): 200 KB
    Small (320px): 100 KB
    Thumbnail (150px): 15 KB
    Total: ~4 MB per photo

Storage:
  Diário: 100M × 4 MB = 400 TB/dia
  Anual: 400 TB × 365 = 146 PB
  Replicação 3x: ~438 PB

Bandwidth:
  Upload: 100M/dia → 1,157/s × 4MB = 4.6 GB/s
  Download (10:1): ~46 GB/s → CDN ESSENCIAL

QPS:
  Upload: ~1,200 QPS (peak ~3,600)
  Feed reads: 500M DAU × 10/dia = 5B → ~58,000 QPS
  Peak: ~170,000 QPS
```

---

## Arquitetura Geral

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                                                                     │
  │  ┌────────┐    ┌──────────────────┐    ┌─────────────────────────┐ │
  │  │ Mobile │───▶│  API Gateway     │───▶│  Load Balancer          │ │
  │  │ Client │    │  (Auth, Rate     │    │  (L7, by endpoint)      │ │
  │  └────────┘    │   Limit, Route)  │    └───┬────────┬────────┬───┘ │
  │                └──────────────────┘        │        │        │     │
  │                                            ▼        ▼        ▼     │
  │  ┌────────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
  │  │ Photo Upload   │  │ Feed Service │  │ Social Graph / User    │ │
  │  │ Service        │  │              │  │ Service                │ │
  │  │                │  │ Home feed    │  │                        │ │
  │  │ Upload → Queue │  │ Explore feed │  │ Follow, Profile,      │ │
  │  │ → Process      │  │ Stories feed │  │ Search                 │ │
  │  └────────┬───────┘  └──────┬───────┘  └────────────┬───────────┘ │
  │           │                 │                        │             │
  │           ▼                 ▼                        ▼             │
  │  ┌─────────────────────────────────────────────────────────────┐   │
  │  │  Data Layer                                                 │   │
  │  │                                                             │   │
  │  │  ┌──────────┐  ┌──────────┐  ┌─────────┐  ┌───────────┐   │   │
  │  │  │ Photo    │  │ Post     │  │Feed Cache│  │Social Graph│  │   │
  │  │  │ Storage  │  │ Metadata │  │ (Redis)  │  │(TAO/Redis) │  │   │
  │  │  │ (S3)     │  │ (Postgres│  │          │  │            │  │   │
  │  │  │          │  │  +Cassan │  │Pre-comput│  │followers/  │  │   │
  │  │  │ Original │  │  dra)    │  │ed feeds  │  │following   │  │   │
  │  │  │ + resize │  │          │  │          │  │            │  │   │
  │  │  └──────────┘  └──────────┘  └─────────┘  └───────────┘   │   │
  │  └─────────────────────────────────────────────────────────────┘   │
  │                                                                     │
  │  ┌────────────────────────────────────────────────────────────┐     │
  │  │  CDN (Content Delivery Network)                            │     │
  │  │  - Akamai, CloudFront, Meta CDN                           │     │
  │  │  - Serve 80%+ de todas as imagens                         │     │
  │  │  - Edge locations globais                                  │     │
  │  │  - Cache por URL (hash do image ID)                       │     │
  │  └────────────────────────────────────────────────────────────┘     │
  │                                                                     │
  │  ┌────────────────────────────────────────────────────────────┐     │
  │  │  Image Processing Pipeline (async)                         │     │
  │  │                                                            │     │
  │  │  Upload → Kafka → Workers:                                │     │
  │  │   1. Virus scan                                           │     │
  │  │   2. EXIF extraction (GPS, camera, date)                  │     │
  │  │   3. Resize: 1080p, 640p, 320p, 150p                     │     │
  │  │   4. Apply filters (se solicitado)                        │     │
  │  │   5. Upload variants para S3                              │     │
  │  │   6. Invalidar CDN cache                                  │     │
  │  │   7. Update metadata DB                                   │     │
  │  │   8. ML: object detection, NSFW check, alt-text           │     │
  │  └────────────────────────────────────────────────────────────┘     │
  └─────────────────────────────────────────────────────────────────────┘
```

---

## Photo Upload Flow

```
  Client → Upload photo → Server

  Optimized flow (Instagram real):

  1. Client seleciona foto
  2. Client faz upload DIRETO para S3 (pre-signed URL)
     → Server NÃO é intermediário para bytes!
     → Menos carga nos API servers
  
  3. Client notifica API server: "upload done, S3 key = xxx"
  4. API server enqueue processamento (Kafka)
  5. Workers processam (resize, scan, ML)
  6. Status: "post published" → push notification para followers

  ┌────────┐  1.Request  ┌───────────┐  2.Pre-signed URL
  │ Client │────────────▶│ API Server│─────────────────────┐
  └───┬────┘             └───────────┘                     │
      │                                                    ▼
      │  3. Upload directly                         ┌──────────┐
      │─────────────────────────────────────────────▶│   S3     │
      │                                              └──────────┘
      │  4. Notify "upload complete"                       │
      │──────────────▶ API Server ──▶ Kafka ──▶ Workers ──┘
                                                    │
                                              Process:
                                              - Resize
                                              - Scan
                                              - ML labels
                                              - CDN push
```

---

## Feed System

### Hybrid Fan-out (similar ao Twitter)

```
  Mesma abordagem hybrid:
  
  Normal users (< 10K followers):
    Post → fan-out workers → push to each follower's feed cache
    
  Celebrity/High-follower users:
    Post → store in post DB
    → Pulled at read time + merged with pre-computed feed
  
  Feed Ranking (Instagram's ML pipeline):
  
  ┌─────────────────────────────────────────────────────────────┐
  │  Candidate Generation (1000s of posts)                      │
  │    → Posts from following                                    │
  │    → Recommended posts (Explore)                            │
  │    → Ads                                                    │
  │                                                              │
  │  Ranking Model:                                              │
  │    Features:                                                 │
  │    - P(like)          → probability user will like           │
  │    - P(comment)       → probability user will comment        │
  │    - P(save)          → probability user will save           │
  │    - P(share)         → probability user will share          │
  │    - Recency          → newer posts scored higher            │
  │    - Author affinity  → how much user interacts with author  │
  │    - Content type     → photo vs carousel vs reel            │
  │                                                              │
  │  Final score = weighted combination of all P(action)         │
  │  Sort by score → top N → serve to client                    │
  └─────────────────────────────────────────────────────────────┘
```

---

## Stories

```
  Stories: conteúdo temporário que expira em 24 horas
  
  ┌──────────────────────────────────────────────────────┐
  │  Story Storage                                        │
  │                                                       │
  │  stories table:                                       │
  │  ┌──────────┬─────────┬───────────┬──────────────┐   │
  │  │ story_id │ user_id │ media_url │ expires_at   │   │
  │  │ (UUID)   │         │ (S3)      │ (created+24h)│   │
  │  └──────────┴─────────┴───────────┴──────────────┘   │
  │                                                       │
  │  Story Tray (horizontal scroll no topo do feed):     │
  │  - Ordered by: recency + user affinity               │
  │  - Client pre-fetches next 5 stories                 │
  │  - Viewed stories tracked (seen_stories set)         │
  │                                                       │
  │  Cleanup:                                             │
  │  - TTL no Redis para story metadata                  │
  │  - S3 lifecycle rule: delete objects after 48h       │
  │    (buffer de 24h extra para timezone/cache)          │
  │                                                       │
  │  Viewers:                                             │
  │  - Lista de quem viu a story                         │
  │  - Append-only list com TTL de 48h                   │
  │  - Privacidade: apenas o autor vê viewers            │
  └──────────────────────────────────────────────────────┘
```

---

## Infraestrutura Real do Instagram/Meta

### TAO (The Associations and Objects)

```
  TAO: Meta's graph-aware distributed data store
  
  Objects: Users, Posts, Comments, Pages
  Associations: follows, likes, tagged_in, has_comment
  
  ┌────────────────────────────────────────────────────┐
  │  TAO Cache Layer (memcache-based)                   │
  │                                                     │
  │  Object query:                                      │
  │    TAO.get(post:12345) → {user_id, caption, ts}    │
  │                                                     │
  │  Association query:                                 │
  │    TAO.assoc_get(user:123, FOLLOWS) → [456, 789]   │
  │    TAO.assoc_count(post:123, LIKED_BY) → 50000     │
  │                                                     │
  │  Backed by: MySQL (sharded by entity_id)            │
  │  Cache hit rate: 99.8%                              │
  └────────────────────────────────────────────────────┘
```

### Haystack (Photo Storage)

```
  Haystack: Meta's custom photo storage
  
  Problema com filesystems tradicionais:
    Cada foto = 1 file no disco
    100B fotos = 100B inodes → filesystem metadata overhead!
    Cada read = 3 disk IOs: inode, directory, data
  
  Solução Haystack:
    Múltiplas fotos em 1 GRANDE arquivo (volume):
    
    ┌─────────────────────────────────────────────┐
    │  Volume file (100 GB):                      │
    │  ┌────────┬────────┬────────┬────────┬──── │
    │  │ Foto 1 │ Foto 2 │ Foto 3 │ Foto 4 │ ... │
    │  │ offset │ offset │ offset │ offset │     │
    │  │ 0      │ 3MB    │ 6MB    │ 9MB    │     │
    │  └────────┴────────┴────────┴────────┴──── │
    │                                             │
    │  Index (in-memory):                         │
    │  photo_id → (volume_id, offset, size)       │
    │                                             │
    │  Read = 1 disk IO: seek to offset, read N   │
    └─────────────────────────────────────────────┘
  
  Resultado: 1 IO per photo (vs 3 com filesystem)
  Throughput: 4x mais fotos servidas por server
```

---

## Database Schema

```sql
-- Users
CREATE TABLE users (
    id          BIGINT PRIMARY KEY,
    username    VARCHAR(30) UNIQUE,
    display_name VARCHAR(100),
    bio         TEXT,
    avatar_url  VARCHAR(500),
    is_private  BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMP
);

-- Posts (Photos)
CREATE TABLE posts (
    id          BIGINT PRIMARY KEY,     -- Snowflake ID
    user_id     BIGINT NOT NULL,
    caption     TEXT,
    location    VARCHAR(255),
    like_count  BIGINT DEFAULT 0,       -- Denormalized
    comment_count BIGINT DEFAULT 0,     -- Denormalized
    created_at  TIMESTAMP,
    INDEX idx_user_time (user_id, created_at DESC)
);

-- Post Media (multiple photos per post = carousel)
CREATE TABLE post_media (
    id          BIGINT PRIMARY KEY,
    post_id     BIGINT NOT NULL,
    media_type  VARCHAR(10),    -- photo, video
    s3_key      VARCHAR(500),
    width       INT,
    height      INT,
    position    SMALLINT,       -- order in carousel
    INDEX idx_post (post_id)
);

-- Follows (Social Graph)
CREATE TABLE follows (
    follower_id BIGINT,
    followee_id BIGINT,
    created_at  TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),
    INDEX idx_followee (followee_id)
);

-- Likes
CREATE TABLE likes (
    user_id     BIGINT,
    post_id     BIGINT,
    created_at  TIMESTAMP,
    PRIMARY KEY (user_id, post_id),
    INDEX idx_post (post_id)
);
```

---

## Trade-offs

| Decisão | Opção A | Opção B | Instagram usa |
|---------|---------|---------|:-------------:|
| **Photo storage** | S3 | Custom (Haystack) | Haystack (custom) |
| **Feed** | Push (fan-out write) | Hybrid | Hybrid |
| **Social graph** | SQL | Graph store (TAO) | TAO |
| **Feed ranking** | Chronological | ML ranking | ML ranking |
| **Upload** | Via API server | Direct to S3 | Direct (pre-signed) |
| **Processing** | Sync | Async (queue) | Async (Kafka) |

---

## Perguntas Frequentes em Entrevistas

1. **"Como servir fotos para 2B users?"**
   - CDN é obrigatório (serve 80%+ das requests)
   - Multiple sizes gerados no upload (responsive images)
   - Pre-signed URLs para upload direto ao S3

2. **"Como funciona o feed do Instagram?"**
   - Hybrid fan-out (push for normal, pull for celebrities)
   - ML ranking: P(like, comment, save, share)
   - Candidate generation → ranking → serving

3. **"Como armazenar 146 PB/ano de fotos?"**
   - Object storage (S3 / Haystack)
   - Multiple tiers: hot (SSD), warm (HDD), cold (archival)
   - Compression + multiple sizes (não serve original)

4. **"Como implementar Stories?"**
   - TTL de 24h (metadata) + 48h (media, buffer)
   - S3 lifecycle rules para auto-delete
   - Separate storage/cache from main feed

5. **"Instagram usa PostgreSQL?"**
   - Sim! Sharded PostgreSQL para dados transacionais
   - Cassandra para dados de alta escala (likes, views)
   - TAO (graph cache) para social graph
   - Redis para feed cache

---

## Referências

- Instagram Engineering — *"Sharding & IDs at Instagram"*
- Instagram Engineering — *"Saving a Third of Our Memory"*
- Meta Engineering — *"TAO: Facebook's Distributed Data Store for the Social Graph"*
- Meta Engineering — *"Finding a Needle in Haystack: Facebook's Photo Storage"*
- Alex Xu — *"System Design Interview"*, Cap. 11-12: News Feed & Instagram
- ByteByteGo — *"Design Instagram"*
- Grokking the System Design Interview — *"Instagram"*
