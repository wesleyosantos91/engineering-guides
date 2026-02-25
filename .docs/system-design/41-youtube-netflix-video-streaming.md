# 41. YouTube / Netflix — Video Streaming

## Índice

- [Visão Geral](#visão-geral)
- [Requisitos](#requisitos)
- [Estimativas (Back-of-the-Envelope)](#estimativas-back-of-the-envelope)
- [Arquitetura de Alto Nível](#arquitetura-de-alto-nível)
- [Upload Pipeline](#upload-pipeline)
- [Transcoding Pipeline](#transcoding-pipeline)
- [Adaptive Bitrate Streaming](#adaptive-bitrate-streaming)
- [Content Delivery Network (CDN)](#content-delivery-network-cdn)
- [Modelo de Dados](#modelo-de-dados)
- [Recommendation Engine](#recommendation-engine)
- [Search Service](#search-service)
- [Netflix Real Architecture](#netflix-real-architecture)
- [YouTube Real Architecture](#youtube-real-architecture)
- [Código Ilustrativo](#código-ilustrativo)
- [Trade-offs e Decisões](#trade-offs-e-decisões)
- [Perguntas Comuns em Entrevistas](#perguntas-comuns-em-entrevistas)
- [Referências](#referências)

---

## Visão Geral

Um sistema de streaming de vídeo permite que usuários façam **upload**, **processem** e **assistam** vídeos sob demanda, com qualidade adaptativa conforme a conexão do espectador. É um dos sistemas mais intensivos em **storage**, **bandwidth** e **compute** que existem.

**Big Techs que operam este tipo de sistema:**
- **YouTube** (Google) — maior plataforma de vídeo (2B+ MAU)
- **Netflix** — líder em streaming de entretenimento (260M+ assinantes)
- **Twitch** (Amazon) — live streaming
- **Disney+**, **HBO Max**, **Prime Video**

---

## Requisitos

### Funcionais
| Requisito | Descrição |
|-----------|-----------|
| Upload de vídeo | Upload, validação e processamento |
| Streaming | Playback com qualidade adaptativa |
| Search | Busca por título, descrição, tags |
| Recommendations | Sugestões personalizadas |
| Comments/Likes | Interação social |
| Watch history | Histórico e resume playback |
| Live streaming | (Opcional) Transmissão ao vivo |

### Não-Funcionais
| Requisito | Valor |
|-----------|-------|
| MAU | 2B (YouTube-scale) |
| Upload rate | 500 horas de vídeo/minuto |
| Latência de playback start | < 2s |
| Disponibilidade | 99.99% |
| Consistência | Eventual (metadata), strong (billing) |
| Suporte global | CDN em 190+ países |

---

## Estimativas (Back-of-the-Envelope)

```
Upload:
  500h/min × 1440 min/dia = 720,000 horas de vídeo/dia
  Avg processed video (múltiplas resoluções): ~500 MB
  Upload storage/dia: ~360 TB
  Upload storage/ano: ~131 PB

Streaming:
  DAU: 800M (YouTube)
  Avg watch time: 40 min/dia
  Avg bitrate: 5 Mbps
  Peak concurrent viewers: ~100M
  Peak bandwidth: 100M × 5 Mbps = 500 Tbps

Metadata:
  Videos no catálogo: ~800M (YouTube)
  Avg metadata/video: ~5 KB
  Total metadata: ~4 TB
```

---

## Arquitetura de Alto Nível

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         VIDEO STREAMING PLATFORM                         │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────┐         ┌───────────────┐         ┌──────────────┐        │
│  │ Uploader │────────▶│ Upload Service│────────▶│ Object Store │        │
│  │  Client  │ chunked │ (validation,  │  raw    │   (S3/GCS)   │        │
│  └──────────┘ upload  │  dedup, scan) │  video  └──────┬───────┘        │
│                        └───────────────┘                │                │
│                                                  ┌──────▼───────┐        │
│                                                  │  Transcoding │        │
│                                                  │   Pipeline   │        │
│                                                  │ (DAG Tasks)  │        │
│                                                  └──────┬───────┘        │
│                                                         │                │
│                                    ┌────────────────────┼────────┐       │
│                                    ▼          ▼         ▼        ▼       │
│                                 [240p]     [480p]    [720p]   [1080p]   │
│                                    │          │         │        │       │
│                                    └──────────┴────┬────┴────────┘       │
│                                                    ▼                     │
│                                            ┌──────────────┐              │
│                                            │  CDN Origin  │              │
│                                            └──────┬───────┘              │
│                                                   │                      │
│                                            ┌──────▼───────┐              │
│                                            │  CDN Edge    │              │
│                                            │  Servers     │              │
│                                            └──────┬───────┘              │
│                                                   │                      │
│  ┌──────────┐  HLS/DASH  ┌──────────────┐        │                      │
│  │ Viewer   │◀───────────│ Streaming    │◀───────┘                      │
│  │  Client  │ adaptive   │  Service     │                                │
│  └──────────┘ bitrate    └──────────────┘                                │
│                                                                          │
│  ┌──────────────────────────────────────────────────────┐                │
│  │               Supporting Services                     │                │
│  ├──────────┬──────────┬────────────┬───────────────────┤                │
│  │ Metadata │ Search   │ Recommend  │ Analytics         │                │
│  │ Service  │ (Elastic)│ Engine(ML) │ (views, engagement)│                │
│  └──────────┴──────────┴────────────┴───────────────────┘                │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Upload Pipeline

O processo de upload é complexo e deve lidar com arquivos grandes de forma resiliente:

```
┌──────────────────────────────────────────────────────────┐
│                    UPLOAD PIPELINE                         │
│                                                           │
│  1. Client-Side                                          │
│  ┌──────────────────────────────┐                        │
│  │ • Chunk file (5-10 MB parts) │                        │
│  │ • Compute checksum (MD5/SHA) │                        │
│  │ • Request upload session ID  │                        │
│  └──────────┬───────────────────┘                        │
│             │                                             │
│  2. Upload Service                                       │
│  ┌──────────▼───────────────────┐                        │
│  │ • Receive chunks in parallel │                        │
│  │ • Verify checksums           │                        │
│  │ • Resumable (track progress) │                        │
│  │ • Anti-virus scan            │                        │
│  │ • Content moderation (AI)    │                        │
│  │ • Deduplication (hash-based) │                        │
│  └──────────┬───────────────────┘                        │
│             │                                             │
│  3. Object Storage                                       │
│  ┌──────────▼───────────────────┐                        │
│  │ • Store raw video in S3/GCS  │                        │
│  │ • Generate signed URL        │                        │
│  │ • Trigger transcoding event  │                        │
│  └──────────────────────────────┘                        │
└──────────────────────────────────────────────────────────┘
```

### Resumable Upload Protocol

```python
# Exemplo: Resumable upload (similar ao Google's Resumable Upload API)

import hashlib
import requests

class ResumableUploader:
    def __init__(self, file_path: str, chunk_size: int = 10 * 1024 * 1024):
        self.file_path = file_path
        self.chunk_size = chunk_size  # 10 MB
        self.session_id = None
    
    def initiate(self, api_url: str, metadata: dict) -> str:
        """Inicia sessão de upload resumable."""
        response = requests.post(f"{api_url}/uploads", json={
            "filename": metadata["filename"],
            "content_type": metadata["content_type"],
            "total_size": metadata["total_size"],
        })
        self.session_id = response.json()["session_id"]
        return self.session_id
    
    def upload_chunks(self, api_url: str):
        """Envia chunks com retry automático."""
        offset = self._get_server_offset(api_url)  # Resume from last successful
        
        with open(self.file_path, "rb") as f:
            f.seek(offset)
            while True:
                chunk = f.read(self.chunk_size)
                if not chunk:
                    break
                
                checksum = hashlib.md5(chunk).hexdigest()
                headers = {
                    "Content-Range": f"bytes {offset}-{offset + len(chunk) - 1}",
                    "X-Upload-Checksum": checksum,
                    "X-Session-ID": self.session_id,
                }
                
                response = requests.put(
                    f"{api_url}/uploads/{self.session_id}",
                    data=chunk,
                    headers=headers,
                )
                
                if response.status_code == 200:
                    offset += len(chunk)
                elif response.status_code == 308:  # Resume Incomplete
                    offset = self._get_server_offset(api_url)
    
    def _get_server_offset(self, api_url: str) -> int:
        resp = requests.head(f"{api_url}/uploads/{self.session_id}")
        return int(resp.headers.get("X-Upload-Offset", 0))
```

---

## Transcoding Pipeline

O transcoding converte o vídeo raw em múltiplas resoluções e formatos para suportar todos os dispositivos e condições de rede:

```
┌───────────────────────────────────────────────────────────────┐
│                 TRANSCODING DAG (Directed Acyclic Graph)       │
│                                                               │
│  ┌──────────┐                                                 │
│  │ Raw Video│                                                 │
│  │  (Input) │                                                 │
│  └────┬─────┘                                                 │
│       │                                                       │
│       ▼                                                       │
│  ┌──────────┐     ┌───────────┐                               │
│  │ Inspect  │────▶│  Extract  │                               │
│  │ (probe)  │     │  Audio    │                               │
│  └────┬─────┘     └─────┬─────┘                               │
│       │                 │                                     │
│       ▼                 ▼                                     │
│  ┌──────────────────────────────────────┐                     │
│  │          Parallel Encode              │                     │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐│                     │
│  │  │ 240p │ │ 480p │ │ 720p │ │1080p ││                     │
│  │  │H.264 │ │H.264 │ │H.265 │ │H.265 ││                     │
│  │  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘│                     │
│  └─────│────────│────────│────────│─────┘                     │
│        ▼        ▼        ▼        ▼                           │
│  ┌──────────────────────────────────────┐                     │
│  │         Package (Muxing)              │                     │
│  │  ┌────────────┐  ┌────────────┐      │                     │
│  │  │ HLS (.m3u8 │  │ DASH (.mpd │      │                     │
│  │  │  + .ts)    │  │  + .m4s)   │      │                     │
│  │  └────────────┘  └────────────┘      │                     │
│  └──────────────────────┬───────────────┘                     │
│                         │                                     │
│                         ▼                                     │
│                  ┌──────────────┐                              │
│                  │ Thumbnails + │                              │
│                  │ Watermark    │                              │
│                  └──────────────┘                              │
└───────────────────────────────────────────────────────────────┘
```

### Codecs e Containers

| Codec | Compressão | Qualidade | Suporte | Uso |
|-------|-----------|-----------|---------|-----|
| **H.264 (AVC)** | Baseline | Boa | Universal | Mobile, low-end |
| **H.265 (HEVC)** | ~50% menor | Excelente | Patenteado | 4K, Apple |
| **VP9** | ~35% menor | Excelente | Open-source | YouTube |
| **AV1** | ~30% menor que VP9 | Superior | Open-source | Netflix, futuro |

### Per-Title Encoding (Netflix)

Netflix não usa bitrate fixo — otimiza por título:

```
Conteúdo simples (animação): 1080p pode precisar de apenas 1.5 Mbps
Conteúdo complexo (ação):    1080p pode precisar de 8+ Mbps

Processo:
1. Analisa complexidade visual de cada shot
2. Gera "encoding ladder" customizada por título
3. Resultado: mesmo VMAF quality score com menos bandwidth
```

---

## Adaptive Bitrate Streaming

O ABR permite que o player ajuste automaticamente a qualidade do vídeo:

```
┌─────────────────────────────────────────────────────┐
│           ADAPTIVE BITRATE STREAMING                 │
│                                                      │
│  Server-side:                                        │
│  ┌─────────────────────────────────────────┐        │
│  │ Manifest File (.m3u8 / .mpd)            │        │
│  │                                          │        │
│  │ #EXTM3U                                  │        │
│  │ #EXT-X-STREAM-INF:BANDWIDTH=800000       │        │
│  │ video_240p/playlist.m3u8                  │        │
│  │ #EXT-X-STREAM-INF:BANDWIDTH=2400000      │        │
│  │ video_480p/playlist.m3u8                  │        │
│  │ #EXT-X-STREAM-INF:BANDWIDTH=5000000      │        │
│  │ video_720p/playlist.m3u8                  │        │
│  │ #EXT-X-STREAM-INF:BANDWIDTH=8000000      │        │
│  │ video_1080p/playlist.m3u8                 │        │
│  └─────────────────────────────────────────┘        │
│                                                      │
│  Client-side (Player Logic):                         │
│                                                      │
│  ┌─────────┐    ┌──────────┐    ┌───────────┐       │
│  │Download  │──▶│ Measure  │──▶│  Select   │       │
│  │ Segment  │   │Bandwidth │   │ Quality   │       │
│  │ n (4s)   │   │& Buffer  │   │for n+1    │       │
│  └─────────┘   └──────────┘   └───────────┘       │
│                                                      │
│  Bandwidth alta → 1080p                              │
│  Bandwidth cai  → 480p (sem interrupção)            │
│  Buffer baixo   → 240p (evita buffering)            │
└─────────────────────────────────────────────────────┘
```

### HLS vs DASH

| Característica | HLS (Apple) | DASH (MPEG) |
|---------------|-------------|-------------|
| Formato manifest | .m3u8 | .mpd (XML) |
| Segmentos | .ts | .m4s |
| DRM | FairPlay | Widevine, PlayReady |
| Suporte browser | Safari nativo, outros via JS | Via JS (dash.js) |
| Adoção | iOS, Apple TV, maioria CDNs | Android, Smart TVs |
| Latência | ~30s (normal), ~2s (LL-HLS) | ~3-5s (LL-DASH) |

---

## Content Delivery Network (CDN)

```
┌───────────────────────────────────────────────────────┐
│                CDN ARCHITECTURE                        │
│                                                        │
│  ┌──────────────┐                                     │
│  │  CDN Origin   │  (backup — usado em cache miss)    │
│  │  (S3/GCS)     │                                     │
│  └───────┬───────┘                                     │
│          │                                             │
│     ┌────┼────────────────────────┐                    │
│     ▼              ▼              ▼                    │
│  ┌───────┐    ┌───────┐    ┌───────┐                  │
│  │Edge   │    │Edge   │    │Edge   │                  │
│  │US-East│    │EU-West│    │Asia   │                  │
│  │(cache)│    │(cache)│    │(cache)│                  │
│  └───┬───┘    └───┬───┘    └───┬───┘                  │
│      │            │            │                       │
│      ▼            ▼            ▼                       │
│   Viewers      Viewers      Viewers                    │
│   (low          (low         (low                      │
│    latency)     latency)     latency)                  │
└───────────────────────────────────────────────────────┘
```

### Netflix Open Connect

Netflix vai além de CDNs tradicionais:

```
Problema: CDN terceirizado é caro e não otimizado
Solução: Open Connect Appliances (OCA)

┌─────────────────────────────────────────────────┐
│  Netflix Open Connect                            │
│                                                  │
│  ┌───────────────┐    ┌──────────────────┐      │
│  │ ISP Data Center│    │ Internet Exchange │      │
│  │  ┌─────────┐  │    │   ┌─────────┐    │      │
│  │  │  OCA    │  │    │   │  OCA    │    │      │
│  │  │(100TB+) │  │    │   │(petabyts)│    │      │
│  │  └─────────┘  │    │   └─────────┘    │      │
│  └───────────────┘    └──────────────────┘      │
│                                                  │
│  Benefícios:                                     │
│  • Vídeo servido de dentro do ISP = latência ~0  │
│  • Netflix preenche OCAs durante off-peak hours  │
│  • 95%+ do tráfego servido de OCAs              │
│  • Custo: hardware fixo vs. pay-per-GB           │
└─────────────────────────────────────────────────┘
```

---

## Modelo de Dados

### Video Metadata (SQL/NoSQL)

```sql
-- Tabela principal de vídeos
CREATE TABLE videos (
    video_id        UUID PRIMARY KEY,
    uploader_id     UUID NOT NULL,
    title           VARCHAR(500),
    description     TEXT,
    duration_ms     BIGINT,
    upload_status   VARCHAR(20),  -- 'uploading', 'processing', 'ready', 'failed'
    visibility      VARCHAR(20),  -- 'public', 'private', 'unlisted'
    category_id     INT,
    tags            TEXT[],
    view_count      BIGINT DEFAULT 0,
    like_count      BIGINT DEFAULT 0,
    created_at      TIMESTAMP WITH TIME ZONE,
    updated_at      TIMESTAMP WITH TIME ZONE
);

-- Tabela de resoluções transcodificadas
CREATE TABLE video_encodings (
    video_id        UUID REFERENCES videos(video_id),
    resolution      VARCHAR(10),   -- '240p', '480p', '720p', '1080p', '4K'
    codec           VARCHAR(20),   -- 'h264', 'h265', 'vp9', 'av1'
    bitrate_kbps    INT,
    storage_url     VARCHAR(1000), -- S3/GCS path
    file_size_bytes BIGINT,
    PRIMARY KEY (video_id, resolution, codec)
);

-- Watch history
CREATE TABLE watch_history (
    user_id         UUID,
    video_id        UUID,
    watched_at      TIMESTAMP,
    progress_ms     BIGINT,        -- resume playback position
    PRIMARY KEY (user_id, watched_at)
) WITH CLUSTERING ORDER BY (watched_at DESC);  -- Cassandra-style
```

### View Counter (Redis + Batch)

```
Problema: 1M+ views/s no YouTube em vídeos virais
Solução: Counter aggregation

1. Client → API → Redis INCR video:{id}:views
2. Background job: flush Redis → DB every N seconds
3. Eventual consistency aceitável para view count

Redis:
  INCR video:abc123:views           → 42,001
  PFADD video:abc123:unique_viewers  user_789  → HyperLogLog (unique)
```

---

## Recommendation Engine

```
┌──────────────────────────────────────────────────────┐
│              RECOMMENDATION SYSTEM                     │
│                                                        │
│  ┌─────────────────────────────────────┐              │
│  │         Data Sources                 │              │
│  │  • Watch history                     │              │
│  │  • Likes/dislikes                    │              │
│  │  • Search queries                    │              │
│  │  • Impressions & click-through       │              │
│  │  • Device, time of day, location     │              │
│  └─────────────┬───────────────────────┘              │
│                │                                       │
│  ┌─────────────▼───────────────────────┐              │
│  │     Feature Pipeline (Spark/Flink)   │              │
│  └─────────────┬───────────────────────┘              │
│                │                                       │
│  ┌─────────────▼───────────────────────┐              │
│  │         ML Models                    │              │
│  │  ┌──────────┐  ┌──────────────────┐ │              │
│  │  │Candidate │  │ Ranking Model    │ │              │
│  │  │Generation│─▶│ (deep learning)  │ │              │
│  │  │(~1000)   │  │ → top 20-50     │ │              │
│  │  └──────────┘  └──────────────────┘ │              │
│  └─────────────────────────────────────┘              │
│                                                        │
│  YouTube: Two-tower model → candidate → ranking        │
│  Netflix: Matrix factorization + deep learning         │
└──────────────────────────────────────────────────────┘
```

---

## Search Service

```
Viewer ──query──▶ [API] ──▶ [Search Service]
                                    │
                             ┌──────▼──────┐
                             │Elasticsearch │
                             │ (title, desc,│
                             │  tags, captions│
                             │  auto-generated)│
                             └──────────────┘

Otimizações:
• Autocomplete (prefix matching)
• Spell correction (fuzzy search)
• Relevance ranking = text match + popularity + freshness + personalization
• Auto-generated captions (speech-to-text) indexados para busca
```

---

## Netflix Real Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  NETFLIX ARCHITECTURE                      │
│                                                           │
│  ┌─────────────────────────────────────────┐              │
│  │            Control Plane (AWS)           │              │
│  │                                          │              │
│  │  ┌──────┐  ┌────────┐  ┌─────────────┐ │              │
│  │  │ Zuul │  │ Eureka │  │   Hystrix   │ │              │
│  │  │(API  │  │(Service│  │ (Circuit    │ │              │
│  │  │ GW)  │  │ Disc.) │  │  Breaker)   │ │              │
│  │  └──────┘  └────────┘  └─────────────┘ │              │
│  │                                          │              │
│  │  ┌──────────┐  ┌───────────────────────┐│              │
│  │  │ EVCache  │  │  Cassandra / CockroachDB││              │
│  │  │(Memcached│  │  (metadata, user data)  ││              │
│  │  │ wrapper) │  └───────────────────────┘│              │
│  │  └──────────┘                           │              │
│  └─────────────────────────────────────────┘              │
│                                                           │
│  ┌─────────────────────────────────────────┐              │
│  │          Data Plane (Open Connect)       │              │
│  │                                          │              │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐ │              │
│  │  │  OCA    │  │  OCA    │  │  OCA    │ │              │
│  │  │(ISP BR) │  │(ISP US) │  │(ISP JP) │ │              │
│  │  └─────────┘  └─────────┘  └─────────┘ │              │
│  │                                          │              │
│  │  • Fill during off-peak (2-7 AM)        │              │
│  │  • 95%+ traffic served from OCAs        │              │
│  │  • Per-title encoding optimization      │              │
│  └─────────────────────────────────────────┘              │
└──────────────────────────────────────────────────────────┘
```

---

## YouTube Real Architecture

```
Componentes-chave:
• Vitess:    MySQL sharding middleware (open-sourced by YouTube)
• Bigtable:  Metadata storage (comentários, likes)
• Spanner:   Globally consistent data
• Colossus:  Distributed file system (video storage)
• Borg:      Container orchestration (predecessor do K8s)
• Mesa:      Near-realtime analytics (view counts, ads)

Fluxo de upload:
1. Upload → Google Front End (GFE)
2. GFE → Upload Server → Colossus (raw)
3. Trigger Borg job → Transcoding (FFmpeg-based)
4. Output → Colossus → replicated to CDN edge caches
5. Metadata → Vitess/Spanner
6. Index → Search (Caffeine)
```

---

## Código Ilustrativo

### HLS Manifest Generator (Python)

```python
from dataclasses import dataclass
from typing import List

@dataclass
class EncodingProfile:
    resolution: str
    bandwidth: int      # bps
    codec: str
    segment_duration: int = 4  # seconds

class HLSManifestGenerator:
    """Gera master playlist HLS (.m3u8)."""
    
    def generate_master_playlist(
        self, video_id: str, profiles: List[EncodingProfile]
    ) -> str:
        lines = ["#EXTM3U", "#EXT-X-VERSION:3", ""]
        
        for profile in sorted(profiles, key=lambda p: p.bandwidth):
            lines.append(
                f"#EXT-X-STREAM-INF:BANDWIDTH={profile.bandwidth},"
                f"RESOLUTION={profile.resolution},"
                f'CODECS="{profile.codec}"'
            )
            lines.append(
                f"/streams/{video_id}/{profile.resolution}/playlist.m3u8"
            )
            lines.append("")
        
        return "\n".join(lines)
    
    def generate_media_playlist(
        self, video_id: str, profile: EncodingProfile, total_segments: int
    ) -> str:
        lines = [
            "#EXTM3U",
            "#EXT-X-VERSION:3",
            f"#EXT-X-TARGETDURATION:{profile.segment_duration}",
            "#EXT-X-MEDIA-SEQUENCE:0",
            "",
        ]
        
        for i in range(total_segments):
            lines.append(f"#EXTINF:{profile.segment_duration}.0,")
            lines.append(
                f"/segments/{video_id}/{profile.resolution}/seg_{i:05d}.ts"
            )
        
        lines.append("#EXT-X-ENDLIST")
        return "\n".join(lines)


# Uso
profiles = [
    EncodingProfile("426x240",   800_000,  "avc1.42e00a"),
    EncodingProfile("854x480",  2_400_000, "avc1.4d401f"),
    EncodingProfile("1280x720", 5_000_000, "avc1.4d4020"),
    EncodingProfile("1920x1080",8_000_000, "avc1.640028"),
]

gen = HLSManifestGenerator()
print(gen.generate_master_playlist("vid_abc123", profiles))
```

### Adaptive Bitrate Selection (JavaScript)

```javascript
class ABRController {
  constructor(profiles) {
    this.profiles = profiles.sort((a, b) => a.bandwidth - b.bandwidth);
    this.bandwidthHistory = [];
    this.bufferLevel = 0; // seconds
  }

  /**
   * Seleciona o perfil de qualidade baseado em bandwidth e buffer.
   * Similar ao algoritmo usado em players como hls.js e dash.js
   */
  selectQuality(measuredBandwidth, bufferLevel) {
    this.bandwidthHistory.push(measuredBandwidth);
    this.bufferLevel = bufferLevel;

    // Média ponderada das últimas 5 medições (EWMA)
    const avgBandwidth = this._ewma(this.bandwidthHistory.slice(-5));

    // Safety factor (usar apenas 70% da bandwidth estimada)
    const safeBandwidth = avgBandwidth * 0.7;

    // Buffer-based adjustment
    let selected = this.profiles[0]; // fallback: lowest quality

    if (bufferLevel < 5) {
      // Buffer crítico: forçar qualidade mínima
      return selected;
    }

    for (const profile of this.profiles) {
      if (profile.bandwidth <= safeBandwidth) {
        selected = profile;
      } else {
        break;
      }
    }

    // Bonus: se buffer > 30s, permitir um nível acima
    if (bufferLevel > 30) {
      const idx = this.profiles.indexOf(selected);
      if (idx < this.profiles.length - 1) {
        selected = this.profiles[idx + 1];
      }
    }

    return selected;
  }

  _ewma(values) {
    const alpha = 0.3;
    return values.reduceRight(
      (acc, val) => alpha * val + (1 - alpha) * acc,
      values[values.length - 1]
    );
  }
}
```

---

## Trade-offs e Decisões

| Decisão | Opção A | Opção B | Escolha Recomendada |
|---------|---------|---------|--------------------|
| Storage | S3 standard | Custom DFS (Colossus) | S3 (exceto Google-scale) |
| Codec | H.264 (compatível) | AV1 (eficiente) | H.264 + AV1 para modernos |
| CDN | Terceirizado (CloudFront) | Próprio (Open Connect) | Terceirizado (exceto Netflix-scale) |
| Segment duration | 2s (low latency) | 10s (efficiency) | 4-6s (balanço) |
| Transcoding | On-demand | Pre-encode tudo | Pre-encode (batch) |
| View count | Real-time | Near-realtime | Near-realtime (Redis + batch flush) |
| Thumbnails | Static | Animated preview | Ambos (hover = animated) |
| DB metadata | SQL (PostgreSQL) | NoSQL (Cassandra) | SQL + cache (reads dominantes) |

---

## Perguntas Comuns em Entrevistas

1. **"Como você lida com um vídeo viral com milhões de views simultâneos?"**
   - CDN edge caching, segmentos pre-cacheados, escalonamento horizontal de edge servers, Redis para counters

2. **"Como funciona o adaptive bitrate streaming?"**
   - Manifest com múltiplas qualidades, player mede bandwidth e buffer, seleciona segmentos dinamicamente

3. **"Como o Netflix reduz custos de bandwidth?"**
   - Open Connect (OCAs dentro de ISPs), per-title encoding, AV1 codec, pre-fill off-peak

4. **"Como garantir que o transcoding é resiliente?"**
   - DAG de tasks, cada step idempotente, message queue entre steps, retry com backoff, checkpoint em storage

5. **"Como priorizar o transcoding?"**
   - Fila com prioridade: vídeos de creators grandes primeiro, resoluções mais populares (720p) primeiro

6. **"Como lidar com copyright/DMCA?"**
   - Content ID (YouTube): fingerprint de áudio/vídeo, match contra database de conteúdo protegido, ação automática

---

## Referências

| Recurso | Descrição |
|---------|-----------|
| **Netflix Tech Blog** | Arquitetura de encoding e delivery |
| **YouTube Engineering Blog** | Vitess, Borg, transcoding at scale |
| **Alex Xu — System Design Interview** | Capítulo YouTube/Netflix |
| **Netflix: Open Connect Overview** | Como funciona a CDN própria |
| **Adaptive Bitrate Streaming (RFC)** | HLS/DASH specifications |
| **Per-Title Encoding (Netflix)** | Otimização de bitrate por conteúdo |
| **Vitess.io** | MySQL sharding do YouTube (open-source) |
| **Designing Data-Intensive Applications** | Cap. sobre storage e batch processing |
