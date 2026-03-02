# Level 18 — Instagram / Photo Sharing

> **Objetivo:** Projetar e implementar uma plataforma de compartilhamento de fotos com
> upload de imagens, feed de fotos, stories, CDN e processamento de mídia.

**Referência:** [39-instagram-photo-sharing.md](../../.docs/SYSTEM-DESIGN/39-instagram-photo-sharing.md)

**Pré-requisito:** Level 17 completo.

---

## Contexto

Sistema centrado em **mídia visual** — upload de fotos, processamento de imagens
(resize, thumbnails, filters), armazenamento em object storage e distribuição via CDN.
Diferente do Twitter, o foco é no **conteúdo binário** e na **entrega de mídia**.

**Escala alvo:**
- **500M usuários** ativos mensais
- **100M fotos/dia** uploads
- **Average photo size:** 2MB (original), 200KB (display)
- **Storage:** ~200TB/dia novas fotos
- **Feed latency:** < 300ms

---

## Parte 1 — ADRs (4 obrigatórios)

### ADR-001: Image Storage & Processing Pipeline

**Arquivo:** `docs/adrs/ADR-001-image-storage-pipeline.md`

**Options:**
1. **Synchronous processing** — resize + upload + respond
2. **Async pipeline** — upload original → respond → process async via workers
3. **CDN-edge processing** — transformação on-the-fly no CDN edge

### ADR-002: Feed Generation Strategy

**Options:**
1. **Precomputed feed** (fan-out on write, similar ao Twitter)
2. **On-demand feed** (fan-out on read)
3. **Ranked feed** (ML scoring) + precomputed top candidates

### ADR-003: Media Delivery Architecture

**Options:**
1. **Single CDN** — todas imagens via um CDN
2. **Multi-tier CDN** — edge + origin shield
3. **Custom CDN** — edge servers proprietários + S3 origin
4. **Progressive loading** — placeholder blur/LQIP → full resolution

### ADR-004: Stories Ephemeral Content

**Options:**
1. **TTL-based storage** — conteúdo com 24h de vida
2. **Archived stories** — mantém mas esconde após 24h
3. **Separate storage tier** — stories em storage mais barato

**Critérios de aceite:**
- [ ] 4 ADRs com diagramas de fluxo
- [ ] Storage cost estimation (custo mensal projetado)
- [ ] Bandwidth estimation para CDN

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

**Arquivo 1:** `docs/diagrams/18-instagram-hld.drawio`

```
┌──────────┐    ┌───────┐    ┌─────────────────────────────────┐
│ Mobile / │───▶│  LB   │───▶│          API Gateway            │
│   Web    │    └───────┘    └──┬──────────┬──────────┬────────┘
└──────────┘                    │          │          │
                          ┌─────▼──┐ ┌─────▼──┐ ┌────▼────┐
                          │ Upload │ │  Feed  │ │ Story   │
                          │Service │ │Service │ │Service  │
                          └───┬────┘ └───┬────┘ └───┬─────┘
                              │          │          │
                    ┌─────────▼──┐   ┌───▼───┐     │
                    │  Processing│   │ Redis │     │
                    │  Pipeline  │   │(feeds)│     │
                    │  (Workers) │   └───────┘     │
                    └─────┬──────┘                  │
                          │                         │
                   ┌──────▼───────────────────┐     │
                   │     Object Storage       │◀────┘
                   │  (S3/MinIO — originals    │
                   │  + thumbnails + stories)  │
                   └──────────┬───────────────┘
                              │
                         ┌────▼────┐
                         │   CDN   │
                         └─────────┘
```

**Arquivo 2:** `docs/diagrams/18-instagram-upload-sequence.drawio`
- Photo upload (client → presigned URL → S3 → processing pipeline → thumbnails → CDN)
- Feed loading (merge + rank + CDN image URLs)
- Story creation + expiration

**Critérios de aceite:**
- [ ] Upload pipeline com async processing
- [ ] CDN distribution com cache strategy
- [ ] Story lifecycle (create → expire → cleanup)

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/api/main.go
├── internal/
│   ├── domain/
│   │   ├── photo.go               ← Photo entity (metadata)
│   │   ├── user.go
│   │   ├── story.go               ← Story (ephemeral content)
│   │   ├── feed.go                ← Feed item model
│   │   └── like.go                ← Like/comment
│   ├── upload/
│   │   ├── service.go             ← Upload orchestration
│   │   ├── presigner.go           ← Pre-signed URL generation
│   │   ├── validator.go           ← Image type/size validation
│   │   └── service_test.go
│   ├── processing/
│   │   ├── pipeline.go            ← Image processing pipeline
│   │   ├── resizer.go             ← Resize to multiple dimensions
│   │   ├── thumbnail.go           ← Thumbnail generation
│   │   ├── blur_hash.go           ← BlurHash/LQIP generation
│   │   └── pipeline_test.go
│   ├── feed/
│   │   ├── service.go             ← Feed construction
│   │   ├── ranker.go              ← Simple ranking (chronological + engagement)
│   │   └── service_test.go
│   ├── story/
│   │   ├── service.go             ← Story CRUD + expiration
│   │   ├── ring.go                ← Story ring (ordered by user)
│   │   └── service_test.go
│   ├── social/
│   │   ├── like_service.go        ← Like/unlike
│   │   ├── comment_service.go     ← Comment CRUD
│   │   └── follow_service.go      ← Follow/unfollow
│   ├── handler/
│   │   ├── upload.go              ← POST /api/v1/photos (presigned URL)
│   │   ├── feed.go                ← GET /api/v1/feed
│   │   ├── photo.go               ← GET /api/v1/photos/:id
│   │   ├── story.go               ← POST/GET /api/v1/stories
│   │   ├── like.go                ← POST /api/v1/photos/:id/like
│   │   └── comment.go             ← POST /api/v1/photos/:id/comments
│   ├── storage/
│   │   ├── minio.go               ← MinIO/S3 client
│   │   └── cdn.go                 ← CDN URL builder
│   ├── repository/
│   │   ├── photo_repo.go          ← PostgreSQL
│   │   ├── feed_repo.go           ← Redis Sorted Set
│   │   └── story_repo.go          ← Redis with TTL
│   └── worker/
│       ├── image_processor.go     ← Kafka consumer → process images
│       └── story_expirer.go       ← Cleanup expired stories
├── docker-compose.yml
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **Pre-signed URL upload** — client faz upload direto ao MinIO
2. **Image processing pipeline** — goroutines: resize 4 tamanhos (thumb, small, medium, original)
3. **BlurHash generation** — placeholder de baixa resolução para loading progressivo
4. **Feed construction** — Redis Sorted Sets + chronological ranking
5. **Stories** — Redis com TTL de 24h
6. **Likes/Comments** — counter denormalized para performance
7. **CDN URL builder** — compõe URLs de CDN com image transforms
8. **Pagination** — cursor-based no feed

**Critérios de aceite Go:**
- [ ] Upload → processing → 4 tamanhos gerados
- [ ] BlurHash para loading progressivo
- [ ] Feed com cursor-based pagination
- [ ] Stories: criação + expiração (24h TTL)
- [ ] Likes/comments com counters
- [ ] Pre-signed URLs (MinIO)
- [ ] ≥ 18 testes
- [ ] Docker Compose: app + postgres + redis + minio + kafka

---

### 3.2 — Java (Spring Boot)

**Funcionalidades Java:**
1. **Spring Boot** REST API
2. **MinIO SDK** + pre-signed URLs
3. **`javax.imageio`** / Thumbnailator para image processing
4. **Spring Data JPA** para metadata
5. **Spring Data Redis** para feeds + stories
6. **Spring Kafka** para processing pipeline
7. **`@Async`** para operações não-bloqueantes

**Critérios de aceite Java:**
- [ ] Mesmas funcionalidades do Go
- [ ] Thumbnailator para resize
- [ ] Kafka processing pipeline
- [ ] Testes com Testcontainers
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] 4 ADRs (image pipeline, feed, CDN, stories)
- [ ] 2 DrawIO (HLD + Upload Sequence)
- [ ] Go e Java: implementação completa + tests
- [ ] Image processing: upload → 4 resolutions + blurhash
- [ ] Docker Compose: app + postgres + redis + minio + kafka
- [ ] Commit: `feat(system-design-18): instagram photo sharing`
