# Level 16 — Pastebin

> **Objetivo:** Projetar e implementar um Pastebin — sistema de compartilhamento de texto
> com links únicos, syntax highlighting, expiração e limites de tamanho.

**Referência:** [37-pastebin.md](../../.docs/SYSTEM-DESIGN/37-pastebin.md)

**Pré-requisito:** Level 15 completo.

---

## Contexto

Sistema que permite criar, armazenar e compartilhar trechos de texto/código com links curtos.
Diferente do URL Shortener, foca em **armazenamento de conteúdo** (potencialmente grande).

**Escala alvo:**
- **5M pastes/dia** criados
- **Read:Write ratio** = 5:1
- **Tamanho máximo:** 10 MB por paste
- **Retenção:** configurable (default: 30 dias)

---

## Parte 1 — ADRs (3 obrigatórios)

### ADR-001: Content Storage Strategy

**Arquivo:** `docs/adrs/ADR-001-content-storage.md`

**Options:**
1. **Database BLOB** — conteúdo no banco relacional
2. **Object Storage (S3/MinIO)** — conteúdo em blob store
3. **Filesystem** — conteúdo em disco local
4. **Database + Object Storage** — metadata no DB, conteúdo no S3

### ADR-002: Content Deduplication

**Options:**
1. **Content Hash (SHA256)** — detecta duplicatas, economiza storage
2. **No deduplication** — simples, sem overhead
3. **Bloom Filter pre-check + hash** — rápido check, confirm com hash

### ADR-003: Expiration & Cleanup Strategy

**Options:**
1. **TTL no database** + cron job de limpeza
2. **Lazy expiration** — verifica no acesso, remove se expirado
3. **Object lifecycle policies** (S3 lifecycle rules)

**Critérios de aceite:**
- [ ] 3 ADRs completos
- [ ] Storage cost analysis (estimativa de custo por mês)
- [ ] Deduplication savings estimados

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

**Arquivo 1:** `docs/diagrams/16-pastebin-hld.drawio`

```
┌─────────┐    ┌──────────┐    ┌──────────┐
│ Clients │───▶│   Load   │───▶│   API    │
│         │    │ Balancer │    │ Servers  │
└─────────┘    └──────────┘    └────┬─────┘
                                    │
                          ┌─────────┼─────────┐
                          │         │         │
                     ┌────▼──┐ ┌───▼───┐ ┌───▼─────┐
                     │ Redis │ │  DB   │ │ Object  │
                     │ Cache │ │(meta) │ │ Store   │
                     │(hot   │ │       │ │(S3/MinIO│
                     │pastes)│ │       │ │content) │
                     └───────┘ └───────┘ └─────────┘
```

**Arquivo 2:** `docs/diagrams/16-pastebin-sequence.drawio`
- Create paste (text upload → storage → short link)
- Read paste (link → cache → storage → render)
- Expiration cleanup flow

**Critérios de aceite:**
- [ ] HLD com metadata DB + content storage separados
- [ ] Sequence para create, read e cleanup
- [ ] CDN integration para pastes populares

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/api/main.go
├── internal/
│   ├── domain/
│   │   ├── paste.go               ← Paste entity
│   │   └── errors.go
│   ├── paste/
│   │   ├── service.go             ← Paste service
│   │   ├── generator.go           ← ID generation
│   │   ├── dedup.go               ← Content deduplication (SHA256)
│   │   └── service_test.go
│   ├── storage/
│   │   ├── storage.go             ← Interface ContentStore
│   │   ├── minio.go               ← MinIO/S3 adapter
│   │   ├── filesystem.go          ← Filesystem adapter
│   │   └── storage_test.go
│   ├── repository/
│   │   ├── repository.go          ← Interface (metadata)
│   │   ├── postgres.go
│   │   └── redis_cache.go
│   ├── handler/
│   │   ├── create.go              ← POST /api/v1/pastes
│   │   ├── read.go                ← GET /api/v1/pastes/:id
│   │   ├── raw.go                 ← GET /raw/:id (plain text)
│   │   └── handler_test.go
│   ├── cleanup/
│   │   ├── expirer.go             ← Background expiration worker
│   │   └── expirer_test.go
│   └── syntax/
│       └── highlighter.go         ← Syntax detection
├── docker-compose.yml
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **Create paste** com syntax detection automática
2. **Content deduplication** via SHA256 hash
3. **Object storage** (MinIO) para conteúdo
4. **Metadata** em PostgreSQL
5. **Cache** em Redis para hot pastes
6. **Expiration worker** (goroutine + ticker)
7. **Raw text endpoint** para CLI tools (`curl`)
8. **Paste visibility** (public, unlisted, private)
9. **Rate limiting** por IP
10. **Size limit** enforcement (10MB max)

**Critérios de aceite Go:**
- [ ] Create/read paste end-to-end
- [ ] Content stored in MinIO, metadata in Postgres
- [ ] Deduplication: same content → same storage key
- [ ] Expiration: worker remove expired pastes
- [ ] Cache: hot pastes servidos do Redis
- [ ] Raw endpoint: `curl http://host/raw/abc123`
- [ ] Size limit enforced
- [ ] ≥ 15 testes
- [ ] Docker Compose: app + postgres + redis + minio

---

### 3.2 — Java (Spring Boot)

**Funcionalidades Java:**
1. **Spring Boot** com REST API
2. **MinIO SDK** para object storage
3. **Spring Data JPA** para metadata
4. **Spring Data Redis** para cache
5. **`@Scheduled`** para expiration cleanup
6. **Records** para DTOs
7. **Multipart upload** para pastes grandes

**Critérios de aceite Java:**
- [ ] Mesmas funcionalidades do Go
- [ ] MinIO integration
- [ ] Scheduled cleanup
- [ ] Testes com Testcontainers
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] 3 ADRs (storage, deduplication, expiration)
- [ ] 2 DrawIO (HLD + Sequence)
- [ ] Go e Java: implementação completa + tests
- [ ] Docker Compose funcional (app + postgres + redis + minio)
- [ ] Commit: `feat(system-design-16): pastebin`
