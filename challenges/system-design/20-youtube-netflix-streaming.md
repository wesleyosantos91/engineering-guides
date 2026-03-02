# Level 20 — YouTube / Netflix — Video Streaming

> **Objetivo:** Projetar e implementar uma plataforma de streaming de vídeo com upload,
> transcoding, adaptive bitrate streaming e recomendações.

**Referência:** [41-youtube-netflix-streaming.md](../../.docs/SYSTEM-DESIGN/41-youtube-netflix-streaming.md)

**Pré-requisito:** Level 19 completo.

---

## Contexto

Sistema para **upload, processamento e streaming de vídeo** sob demanda. O desafio
é o **pipeline de transcoding** (converter para múltiplas resoluções/codecs) e a
**entrega adaptativa** (ABR — Adaptive Bitrate) via CDN.

**Escala alvo:**
- **2B usuários** mensais
- **500h de vídeo** enviados por minuto
- **1B vídeos** assistidos por dia
- **Resoluções:** 240p, 360p, 480p, 720p, 1080p, 4K
- **Storage:** ~PBs de conteúdo

---

## Parte 1 — ADRs (4 obrigatórios)

### ADR-001: Video Transcoding Pipeline

**Arquivo:** `docs/adrs/ADR-001-transcoding-pipeline.md`

**Options:**
1. **Single monolithic transcoder** — um serviço faz tudo
2. **DAG Pipeline** — orchestrator + step functions (split, transcode, merge)
3. **Map-reduce transcoding** — divide vídeo em chunks, transcodifica em paralelo
4. **Managed service** — AWS Elemental MediaConvert / similar

### ADR-002: Streaming Protocol

**Options:**
1. **HLS (HTTP Live Streaming)** — Apple standard, amplo suporte
2. **DASH (Dynamic Adaptive Streaming over HTTP)** — standard aberto
3. **HLS + DASH** — gerar ambos para máxima compatibilidade
4. **Progressive download** — simples, sem adaptive bitrate

### ADR-003: Content Delivery & Caching

**Options:**
1. **Pull-through CDN** — CDN puxa conteúdo do origin quando requisitado
2. **Push CDN** — pré-push conteúdo popular para edge servers
3. **Multi-tier** — hot cache (L1 edge) + warm cache (L2 origin shield) + cold (S3)

### ADR-004: Video Metadata & Recommendation

**Options:**
1. **Content-based filtering** — tags, categorias, descrição
2. **Collaborative filtering** — users who watched this also watched
3. **Hybrid** — combinação de content + collaborative
4. **Watch history weighted** — ponderado por tempo assistido vs skipped

**Critérios de aceite:**
- [ ] 4 ADRs completos
- [ ] Bandwidth estimation (custo CDN mensal)
- [ ] Storage estimation (growth rate)
- [ ] Transcoding time estimation

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

**Arquivo 1:** `docs/diagrams/20-streaming-hld.drawio`

```
┌──────────┐    ┌────────┐    ┌────────────────────────────────┐
│  Client  │───▶│  LB    │───▶│         API Gateway            │
│(browser/ │    └────────┘    └──┬──────────┬──────────┬───────┘
│ mobile)  │                     │          │          │
└────┬─────┘               ┌────▼──┐ ┌─────▼──┐ ┌────▼─────┐
     │                     │Upload │ │ Video  │ │  Search  │
     │                     │Service│ │Catalog │ │ /Recomm. │
     │                     └───┬───┘ └────┬───┘ └──────────┘
     │                         │          │
     │                    ┌────▼────────┐  │
     │                    │ Transcoding │  │
     │                    │  Pipeline   │  │
     │                    │  (Workers)  │  │
     │                    └──────┬──────┘  │
     │                           │         │
     │              ┌────────────▼─────────▼──────┐
     │              │      Object Storage (S3)     │
     │              │  originals / HLS segments /  │
     │              │  thumbnails / manifests      │
     │              └──────────────┬───────────────┘
     │                             │
     │                        ┌────▼────┐
     └────── stream ──────────│   CDN   │
              (HLS/DASH)      └─────────┘
```

**Arquivo 2:** `docs/diagrams/20-streaming-transcode-sequence.drawio`
- Upload flow (client → presigned URL → S3 → trigger pipeline)
- Transcoding pipeline (split → parallel transcode → merge → manifest)
- Playback flow (client → CDN → adaptive bitrate selection)

**Critérios de aceite:**
- [ ] Upload → transcoding pipeline DAG
- [ ] HLS segment delivery via CDN
- [ ] Adaptive bitrate switching diagram

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/
│   ├── api/main.go                ← REST API server
│   └── transcoder/main.go         ← Transcoding worker
├── internal/
│   ├── domain/
│   │   ├── video.go               ← Video entity (metadata)
│   │   ├── segment.go             ← HLS segment
│   │   ├── manifest.go            ← HLS/DASH manifest
│   │   └── user.go
│   ├── upload/
│   │   ├── service.go             ← Upload orchestration
│   │   ├── presigner.go           ← Pre-signed URL (MinIO)
│   │   ├── validator.go           ← Video format/size validation
│   │   └── service_test.go
│   ├── transcode/
│   │   ├── pipeline.go            ← DAG pipeline orchestrator
│   │   ├── splitter.go            ← Split video into chunks
│   │   ├── encoder.go             ← FFmpeg wrapper (exec.Command)
│   │   ├── merger.go              ← Merge transcoded chunks
│   │   ├── manifest_gen.go        ← HLS manifest (.m3u8) generator
│   │   ├── thumbnail.go           ← Extract thumbnails at intervals
│   │   └── pipeline_test.go
│   ├── catalog/
│   │   ├── service.go             ← Video catalog (CRUD, search)
│   │   └── service_test.go
│   ├── streaming/
│   │   ├── service.go             ← Serve manifest + segments
│   │   ├── abr.go                 ← Adaptive bitrate logic (quality selection)
│   │   └── progress.go            ← Watch progress tracking
│   ├── recommendation/
│   │   ├── engine.go              ← Simple content-based recommendation
│   │   ├── collaborative.go       ← Watch history correlation
│   │   └── engine_test.go
│   ├── handler/
│   │   ├── upload.go              ← POST /api/v1/videos (presigned URL)
│   │   ├── catalog.go             ← GET /api/v1/videos, search, detail
│   │   ├── stream.go              ← GET /stream/:id/manifest.m3u8
│   │   ├── recommend.go           ← GET /api/v1/recommendations
│   │   └── progress.go            ← POST /api/v1/progress
│   ├── repository/
│   │   ├── video_repo.go          ← PostgreSQL
│   │   ├── segment_repo.go        ← MinIO (or filesystem)
│   │   └── progress_repo.go       ← Redis (watch progress)
│   └── worker/
│       └── transcode_worker.go    ← Kafka consumer → transcode pipeline
├── docker-compose.yml
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **Video upload** — pre-signed URL to MinIO
2. **Transcoding pipeline** — FFmpeg via `exec.Command`
   - Split em chunks de 10s
   - Transcode para 3 resoluções: 360p, 720p, 1080p
   - Gera HLS segments (.ts) + manifest (.m3u8)
3. **HLS manifest** — master playlist + variant playlists
4. **Thumbnail extraction** — screenshots em intervalos
5. **Video catalog** — browse, search (PG FTS), detail
6. **Streaming endpoint** — serve manifest + segments via HTTP
7. **Watch progress** — redis-based progress tracker
8. **Simple recommendation** — content-based (tags) + watch history
9. **Async processing** — Kafka: upload event → transcoding worker

**Critérios de aceite Go:**
- [ ] Upload → transcoding → HLS segments gerados
- [ ] Master manifest com múltiplas resoluções
- [ ] Streaming endpoint: manifest + segments servidos via HTTP
- [ ] Thumbnails gerados em intervalos
- [ ] Video catalog com busca
- [ ] Watch progress tracking
- [ ] Recommendation simples funcional
- [ ] ≥ 18 testes
- [ ] Docker Compose: api + transcoder + postgres + redis + minio + kafka
- [ ] **Nota:** FFmpeg deve estar instalado no container

---

### 3.2 — Java (Spring Boot)

**Funcionalidades Java:**
1. **Spring Boot** REST API
2. **ProcessBuilder** para FFmpeg (transcoding)
3. **MinIO SDK** para object storage
4. **Spring Data JPA** para metadata
5. **Spring Data Redis** para watch progress
6. **Spring Kafka** para transcoding pipeline
7. **`@Async`** + `CompletableFuture` para parallel transcoding
8. **Virtual Threads** para streaming endpoints

**Critérios de aceite Java:**
- [ ] Mesmas funcionalidades do Go
- [ ] FFmpeg via ProcessBuilder
- [ ] CompletableFuture para parallel transcoding
- [ ] Testes com Testcontainers
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] 4 ADRs (transcoding, streaming protocol, CDN, recommendations)
- [ ] 2 DrawIO (HLD + Transcoding Sequence)
- [ ] Go e Java: implementação completa + tests
- [ ] Video: upload → transcode → 3 resoluções + HLS + thumbnails
- [ ] Docker Compose: api + transcoder + postgres + redis + minio + kafka
- [ ] Commit: `feat(system-design-20): youtube netflix streaming`
