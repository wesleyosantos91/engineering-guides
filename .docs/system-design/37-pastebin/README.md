# 37. Pastebin

> **Categoria:** Classic System Design  
> **Nível:** Problema clássico em entrevistas — variação simples de URL Shortener  
> **Complexidade:** Média

---

## Definição

**Pastebin** (como pastebin.com, GitHub Gist, hastebin) é um serviço que permite usuários armazenar e compartilhar **texto/código** através de uma URL única. É uma variação do URL Shortener focada em **armazenamento e retrieval de conteúdo textual** com features como syntax highlighting, expiração e controle de acesso.

---

## Requisitos

### Funcionais

```
1. Upload de texto/código (até 10MB) → receber URL única
2. Acessar paste via URL → ver conteúdo
3. Expiração configurável (10min, 1h, 1d, 1w, 1m, nunca)
4. Pastes podem ser públicos ou privados (unlisted)
5. Syntax highlighting por linguagem
6. API para criação programática
7. Raw text endpoint (/raw/abc123)
```

### Não-Funcionais

```
1. Alta disponibilidade (99.9%+)
2. Baixa latência de leitura (< 200ms)
3. Durabilidade dos dados (não perder pastes)
4. Read-heavy (5:1 read:write ratio)
5. Escala: 5M pastes/dia (similarmente ao Pastebin real)
```

---

## Estimativas

```
Write:
  5M pastes/dia
  QPS: 5M / 86,400 ≈ 60 QPS
  Peak: ~180 QPS

Read:
  Read:Write = 5:1
  5 × 60 = 300 QPS
  Peak: ~900 QPS

Storage:
  Tamanho médio de paste: 10 KB
  Storage/dia: 5M × 10KB = 50 GB
  Storage/ano: 50 GB × 365 = 18 TB
  5 anos: 91 TB
  Com replicação 3x: ~273 TB

  ⚡ Diferença do URL Shortener:
  URL Shortener armazena ~500 bytes (apenas URL)
  Pastebin armazena ~10KB (conteúdo inteiro!)
  → Storage é o bottleneck principal

Bandwidth:
  Read: 900 QPS × 10KB = 9 MB/s ≈ 72 Mbps (modesto)
```

---

## Arquitetura

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  Client (Browser / API)                                          │
  │    │                                                             │
  │    ▼                                                             │
  │  ┌──────────────┐                                                │
  │  │ Load Balancer │                                                │
  │  └──────┬───────┘                                                │
  │         │                                                        │
  │  ┌──────▼──────────┐                                             │
  │  │   API Servers    │ (Stateless)                                 │
  │  │                  │                                             │
  │  │  POST /pastes    │ → Create                                   │
  │  │  GET  /pastes/:id│ → Read                                     │
  │  │  GET  /raw/:id   │ → Raw text                                 │
  │  └──────┬──────────┘                                             │
  │         │                                                        │
  │  ┌──────┼────────────────────────────┐                           │
  │  │      │                            │                           │
  │  ▼      ▼                            ▼                           │
  │ ┌──────────┐   ┌──────────────┐   ┌───────────────────────┐     │
  │ │  Cache   │   │ Metadata DB  │   │  Object Storage (S3)  │     │
  │ │ (Redis)  │   │ (PostgreSQL) │   │                       │     │
  │ │          │   │              │   │  Paste content stored  │     │
  │ │ Hot      │   │ paste_id     │   │  as objects:           │     │
  │ │ pastes   │   │ user_id      │   │  s3://pastes/abc123    │     │
  │ │          │   │ s3_key       │   │  s3://pastes/def456    │     │
  │ │          │   │ language     │   │                       │     │
  │ │          │   │ expires_at   │   │  ✅ Cheap storage      │     │
  │ │          │   │ visibility   │   │  ✅ Durable (11 nines) │     │
  │ │          │   │ created_at   │   │  ✅ Scales infinitely  │     │
  │ └──────────┘   └──────────────┘   └───────────────────────┘     │
  │                                                                  │
  │  Cleanup:                                                        │
  │  ┌──────────────────────────────────┐                            │
  │  │  Expiration Worker (CRON)        │                            │
  │  │  - Scan expired pastes           │                            │
  │  │  - Delete from S3 + DB           │                            │
  │  │  - Run every hour                │                            │
  │  └──────────────────────────────────┘                            │
  └──────────────────────────────────────────────────────────────────┘

  Por que S3 e não DB para conteúdo?
  ┌──────────────────────────────────────────────────────────────────┐
  │  DB (PostgreSQL): Ótimo para metadados pequenos (<1KB/row)       │
  │    → Mas 10KB+ por paste SOBRECARREGA o DB                      │
  │    → Backups ficam enormes                                       │
  │    → Query performance degrada com BLOBs                         │
  │                                                                  │
  │  S3/Object Storage: Perfeito para conteúdo grande                │
  │    → $0.023/GB/mês (muito barato)                                │
  │    → 11 nines durability (99.999999999%)                         │
  │    → Escala infinitamente                                        │
  │    → CDN-friendly (servir direto do S3)                          │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Flows

### Create Paste

```
  1. Client → POST /api/pastes
     Body: { content: "print('hello')", language: "python", expiry: "1d" }
  
  2. API Server:
     a. Validar: tamanho ≤ 10MB, content não vazio
     b. Gerar paste_id: KGS ou base62(counter)
     c. Upload content para S3:
        PUT s3://pastes/{paste_id}
        Content-Type: text/plain
     d. Salvar metadata no DB:
        INSERT INTO pastes (id, s3_key, language, expires_at, ...)
     e. Salvar no Cache (Redis):
        SET paste:{paste_id} → content (TTL: 1h)
  
  3. Retornar: { url: "https://paste.io/abc123" }
```

### Read Paste

```
  1. Client → GET /pastes/abc123
  
  2. API Server:
     a. Check Cache (Redis): paste:abc123 → ?
        HIT → retorna content
        MISS ↓
     b. Query metadata DB:
        SELECT * FROM pastes WHERE id = 'abc123'
        AND (expires_at IS NULL OR expires_at > NOW())
     c. Se encontrado: GET s3://pastes/abc123
     d. Salvar no Cache (TTL: 1h)
     e. Retornar content com syntax highlighting
     
     Se não encontrado ou expirado → 404
```

### Expiration Cleanup

```
  CRON Job (hourly):
  
  1. Query: SELECT id, s3_key FROM pastes 
            WHERE expires_at IS NOT NULL AND expires_at < NOW()
            LIMIT 1000  -- batch processing
  
  2. Para cada batch:
     a. DELETE from S3: DELETE s3://pastes/{paste_id}
     b. DELETE from DB: DELETE FROM pastes WHERE id IN (...)
     c. DELETE from Cache: DEL paste:{paste_id}
  
  3. Repeat até nenhum resultado
  
  Lazy delete também:
  → No read, se expires_at < NOW() → retorna 404 + schedule delete
```

---

## Schema

```sql
CREATE TABLE pastes (
    id          VARCHAR(8) PRIMARY KEY,     -- Base62 unique key
    s3_key      VARCHAR(255) NOT NULL,      -- S3 object path
    title       VARCHAR(255),               -- Optional title
    language    VARCHAR(50) DEFAULT 'text', -- For syntax highlighting
    visibility  VARCHAR(10) DEFAULT 'public', -- public, unlisted, private
    user_id     BIGINT,                     -- NULL for anonymous pastes
    size_bytes  INTEGER NOT NULL,           -- Content size
    expires_at  TIMESTAMP,                  -- NULL = never
    created_at  TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_expires (expires_at) WHERE expires_at IS NOT NULL,
    INDEX idx_user (user_id) WHERE user_id IS NOT NULL
);

-- Para buscar pastes recentes de um user:
-- SELECT * FROM pastes WHERE user_id = ? ORDER BY created_at DESC LIMIT 20;
```

---

## Diferenças vs URL Shortener

```
┌────────────────────┬──────────────────────┬──────────────────────┐
│  Aspecto            │  URL Shortener       │  Pastebin            │
├────────────────────┼──────────────────────┼──────────────────────┤
│  Conteúdo          │  URL (~200 bytes)     │  Texto (~10KB+)      │
│  Storage           │  DB é suficiente      │  Object Storage (S3) │
│  Response          │  Redirect (301/302)   │  Render content      │
│  Read pattern      │  Pure redirect        │  Content delivery    │
│  Traffic ratio     │  100:1 read:write     │  5:1 read:write      │
│  CDN               │  Menos importante     │  Muito importante    │
│  Syntax highlight  │  N/A                  │  Feature principal   │
│  Size limit        │  URL length (~2KB)    │  Até 10MB            │
└────────────────────┴──────────────────────┴──────────────────────┘
```

---

## Otimizações

### 1. Compression

```
  Texto comprime MUITO bem (60-80% redução):
  
  Paste original: 10 KB
  Gzip compressed: ~3 KB
  
  Storage economizado: 70%
  
  Pipeline:
  Upload: content → gzip → S3 (Content-Encoding: gzip)
  Download: S3 → decompress → client (Accept-Encoding: gzip)
  
  Economia em 5 anos:
  Sem compression: 91 TB
  Com compression: ~27 TB (70% saving!)
```

### 2. CDN para Raw Content

```
  Pastes populares (hot content):
  
  Client → CDN → S3 (cache miss) → CDN Cache → Client (cache hit)
  
  CDN TTL: 5 min para pastes públicos
  Cache invalidation: quando paste é deletado/editado
  
  Reduz S3 reads em ~80%
```

### 3. Rate Limiting

```
  Prevenir abuse (spam, crypto mining scripts, etc.):
  
  Anonymous: 10 pastes/hora
  Registered: 100 pastes/hora  
  API key: configurable
  
  Size limit: 10 MB por paste
  Total storage per user: 100 MB (free), 1 GB (paid)
```

---

## Uso em Big Techs

| Empresa/Produto | Foco | Detalhes |
|----------------|------|----------|
| **GitHub Gist** | Code sharing | Git-backed, versionado, markdown |
| **Pastebin.com** | Text sharing | 17M+ pastes, rate limited |
| **Hastebin** | Minimal code sharing | Open-source, keyboard shortcuts |
| **GitLab Snippets** | Code sharing | Parte do GitLab, CI integration |
| **Rentry.co** | Markdown pastes | Markdown rendering nativo |
| **PrivateBin** | Encrypted pastes | Zero-knowledge, E2E encryption |

---

## Perguntas Frequentes em Entrevistas

1. **"Por que S3 ao invés do DB para conteúdo?"**
   - DB: ótimo para metadados (<1KB), ruim para BLOBs (10KB+)
   - S3: $0.023/GB, 11 nines durability, escala infinita
   - Separação: metadata no DB, content no S3

2. **"Como lidar com pastes muito grandes?"**
   - Limit: 10MB server-side validation
   - Compression: gzip reduz 60-80%
   - Streaming upload: multipart para files grandes

3. **"Como implementar expiração?"**
   - Lazy check: no read, verify expires_at
   - CRON cleanup: batch delete de expirados (hourly)
   - S3 lifecycle rules: auto-delete objetos expirados

4. **"Pastebin vs URL Shortener?"**
   - Shortener: redireciona, não armazena conteúdo
   - Pastebin: armazena e serve conteúdo
   - Pastebin precisa de object storage (S3)

5. **"Como escalar reads?"**
   - Cache (Redis) para hot pastes
   - CDN para raw content delivery
   - Read replicas no DB para metadata queries

---

## Referências

- Alex Xu — *"System Design Interview"*, Cap. 12 (similar patterns)
- Grokking the System Design Interview — *"Design Pastebin"*
- GitHub Gist Architecture — github.blog
- AWS S3 Best Practices — aws.amazon.com/s3
- System Design Primer — *"Design Pastebin.com"*
