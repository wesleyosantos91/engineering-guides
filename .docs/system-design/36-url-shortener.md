# 36. URL Shortener (TinyURL)

> **Categoria:** Classic System Design  
> **Nível:** Pergunta clássica em entrevistas — Google, Meta, Amazon  
> **Complexidade:** Média

---

## Definição

Um **URL Shortener** (como TinyURL, Bitly, short.io) converte URLs longas em URLs curtas únicas que redirecionam para a URL original. É um dos **problemas mais clássicos** de system design por combinar hashing, encoding, database design, caching e escalabilidade.

---

## Requisitos

### Funcionais

```
1. Dado uma long URL → gerar short URL única
2. Dado uma short URL → redirecionar para a long URL
3. Links expiram após X tempo (configurable, default: não expira)
4. Custom alias opcional (ex: short.url/meu-link)
5. Analytics: contagem de cliques, geolocation, referrer
```

### Não-Funcionais

```
1. Baixa latência de redirect (< 50ms com cache)
2. Alta disponibilidade (99.99%)
3. URLs devem ser o mais curtas possível
4. Não previsíveis (segurança)
5. Escala: 100M URLs criadas/dia, read-heavy (100:1 ratio)
```

---

## Estimativas (Back-of-the-Envelope)

```
Write (URL creation):
  100M URLs/dia
  QPS: 100M / 86,400 ≈ 1,200 QPS
  Peak: ~3,600 QPS

Read (redirects):
  Read:Write ratio = 100:1
  100 × 1,200 = 120,000 QPS
  Peak: ~360,000 QPS

Storage:
  Cada registro: ~500 bytes (short_url + long_url + metadata)
  Por dia: 100M × 500B = 50 GB
  5 anos: 50 GB × 365 × 5 = 91 TB

Cache:
  80/20 rule: 20% das URLs → 80% do tráfego
  Hot URLs/dia: 100M × 0.2 × 500B = 10 GB
  → Cabe facilmente em Redis

Bandwidth:
  Egress: 360K QPS × 500B = 180 MB/s ≈ 1.5 Gbps
```

---

## Short URL Design

### De onde vem o ID curto?

```
Base62 encoding: [a-z, A-Z, 0-9] = 62 caracteres

  Tamanho 6: 62^6 = 56.8 bilhões de combinações
  Tamanho 7: 62^7 = 3.5 trilhões de combinações ✅

  Com 100M URLs/dia × 365 × 10 anos = 365B URLs
  3.5T >> 365B → 7 caracteres é suficiente

  Exemplo: https://short.url/aB3x7Kp → 7 chars
```

### Métodos de Geração

#### 1. Hash + Truncate

```
  long_url = "https://example.com/very/long/path?param=value"
  
  hash = MD5(long_url) = "5d41402abc4b2a76b9719d911017c592"
                           ├────────┤
                           Pegar primeiros 7 chars
  
  short = base62_encode(first_43_bits)
  
  ✅ Determinístico (mesma URL → mesmo hash)
  ❌ Colisões possíveis → precisa verificar + retry
  ❌ Mesma URL sempre gera o mesmo short (pode ser feature ou bug)
```

#### 2. Counter-Based (Auto-increment)

```
  Distributed Counter:
  
  ┌────────────────┐     ┌────────────────┐
  │ Counter Service │     │ Counter Service │
  │ Range: 1-1M    │     │ Range: 1M-2M   │
  └───────┬────────┘     └───────┬────────┘
          │                      │
    next_id = 1               next_id = 1,000,001
    short = base62(1)         short = base62(1000001)
    = "1"                     = "4c92"
  
  id=12345 → base62(12345) = "3d7"
  
  ✅ Zero colisões (IDs são únicos)
  ✅ Simples e eficiente
  ❌ Previsível (IDs sequenciais → segurança)
  ❌ Counter é SPOF (solved: range-based distributed counter)
```

#### 3. Pre-generated Keys (KGS — Key Generation Service)

```
  ┌─────────────────────────────────────────────────────────┐
  │  Key Generation Service (KGS)                           │
  │                                                         │
  │  1. Pré-gera bilhões de short keys únicas              │
  │  2. Armazena em DB com status: used/unused              │
  │  3. Quando app precisa: pega next unused key            │
  │                                                         │
  │  ┌─────────────────┐    ┌─────────────────────────┐    │
  │  │  Unused Keys DB  │───▶│  Used Keys DB           │    │
  │  │  aB3x7Kp ✅      │    │  zT9mNwQ (→ google.com) │    │
  │  │  mN2pQr5 ✅      │    │  xY8kLp3 (→ github.com) │    │
  │  │  zK8wVn1 ✅      │    │  ...                     │    │
  │  └─────────────────┘    └─────────────────────────┘    │
  │                                                         │
  │  App Server carrega batch (ex: 1000 keys) em memória   │
  │  → Sem contention no DB para cada request              │
  └─────────────────────────────────────────────────────────┘
  
  ✅ Zero colisões (pré-validado)
  ✅ Sem vulnerabilidade de previsão (random)
  ✅ Extremamente rápido (keys já prontas)
  ❌ Complexidade extra (KGS service)
  ❌ Key waste se server crash com batch em memória
```

---

## Arquitetura

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                                                                 │
  │  Client                                                         │
  │    │                                                            │
  │    ▼                                                            │
  │  ┌──────────────┐     ┌───────────────┐                         │
  │  │ Load Balancer │────▶│  API Servers  │                         │
  │  └──────────────┘     │  (Stateless)  │                         │
  │                       └───────┬───────┘                         │
  │                               │                                 │
  │                   ┌───────────┼───────────┐                     │
  │                   ▼           ▼           ▼                     │
  │          ┌────────────┐ ┌──────────┐ ┌─────────────────┐       │
  │          │   Cache    │ │ URL DB   │ │  Key Generation │       │
  │          │  (Redis)   │ │(Cassandra│ │  Service (KGS)  │       │
  │          │            │ │ /DynamoDB)│ │                 │       │
  │          │  short_url │ │          │ │  Pre-generated  │       │
  │          │  → long_url│ │ short_url│ │  unique keys    │       │
  │          └────────────┘ │ long_url │ └─────────────────┘       │
  │                         │ created  │                            │
  │                         │ expires  │                            │
  │                         │ user_id  │                            │
  │                         └──────────┘                            │
  │                                                                 │
  │  Analytics (async):                                             │
  │  ┌──────────┐    ┌────────────┐    ┌──────────────────┐        │
  │  │  Kafka   │───▶│  Consumer  │───▶│  Analytics DB    │        │
  │  │ (clicks) │    │  Workers   │    │  (ClickHouse/    │        │
  │  └──────────┘    └────────────┘    │   TimescaleDB)   │        │
  │                                    └──────────────────┘        │
  └─────────────────────────────────────────────────────────────────┘
```

### Flow: Create Short URL

```
  1. Client → POST /api/shorten { long_url: "..." }
  2. API Server:
     a. Validar URL (format, length, not blocked)
     b. Check custom alias (se fornecido) → disponível?
     c. Obter short key:
        - KGS: pegar próximo key do batch local
        - Hash: hash(long_url), check colisão
        - Counter: next_id → base62
     d. Salvar no DB: short_url → long_url
     e. Salvar no Cache: short_url → long_url
  3. Retornar: { short_url: "https://short.url/aB3x7Kp" }
```

### Flow: Redirect

```
  1. Client → GET https://short.url/aB3x7Kp
  2. API Server:
     a. Check Cache (Redis): aB3x7Kp → ?
        - HIT → long_url encontrada
        - MISS → Query DB → salvar no cache
     b. Verificar expiração
     c. Enviar evento para Kafka (analytics, async)
     d. Retornar HTTP 301 ou 302 redirect

  301 (Moved Permanently):
    Browser cacheia o redirect → requests futuros NÃO passam pelo server
    ✅ Menos carga no server
    ❌ Analytics incompleto (browser bypassa)

  302 (Found / Temporary):
    Browser NÃO cacheia → TODA request passa pelo server
    ✅ Analytics completo (cada clique registrado)
    ❌ Mais carga no server
    
  → A maioria usa 302 para ter analytics precisos
```

---

## Schema do Banco de Dados

```sql
-- Main URL table
CREATE TABLE urls (
    short_key   VARCHAR(7) PRIMARY KEY,    -- Base62 key
    long_url    TEXT NOT NULL,             -- Original URL
    user_id     BIGINT,                    -- Creator (nullable)
    created_at  TIMESTAMP DEFAULT NOW(),
    expires_at  TIMESTAMP,                 -- NULL = never expires
    click_count BIGINT DEFAULT 0           -- Denormalized counter
);

-- Index para busca por long_url (dedup)
CREATE INDEX idx_long_url ON urls (long_url);

-- Analytics table (append-only)
CREATE TABLE clicks (
    id          BIGINT AUTO_INCREMENT,
    short_key   VARCHAR(7),
    clicked_at  TIMESTAMP DEFAULT NOW(),
    ip_address  INET,
    user_agent  TEXT,
    referrer    TEXT,
    country     VARCHAR(2)
);

-- Partitioned by time para facilitar cleanup
-- CREATE TABLE clicks ... PARTITION BY RANGE (clicked_at);
```

---

## Code: Implementação Simplificada

```python
import hashlib
import redis
import string

CHARSET = string.ascii_letters + string.digits  # a-zA-Z0-9 (62 chars)
SHORT_LENGTH = 7
cache = redis.Redis()

def base62_encode(num: int) -> str:
    """Convert integer to base62 string."""
    if num == 0:
        return CHARSET[0]
    result = []
    while num > 0:
        result.append(CHARSET[num % 62])
        num //= 62
    return ''.join(reversed(result))

def create_short_url(long_url: str, counter_service) -> str:
    """Generate short URL using counter-based approach."""
    next_id = counter_service.get_next_id()
    short_key = base62_encode(next_id).zfill(SHORT_LENGTH)
    
    # Save to DB
    db.execute(
        "INSERT INTO urls (short_key, long_url) VALUES (%s, %s)",
        (short_key, long_url)
    )
    
    # Save to cache
    cache.setex(f"url:{short_key}", 86400, long_url)
    
    return f"https://short.url/{short_key}"

def redirect(short_key: str) -> str:
    """Resolve short URL to long URL."""
    # 1. Check cache
    long_url = cache.get(f"url:{short_key}")
    if long_url:
        return long_url.decode()
    
    # 2. Check DB
    result = db.execute(
        "SELECT long_url FROM urls WHERE short_key = %s AND "
        "(expires_at IS NULL OR expires_at > NOW())",
        (short_key,)
    )
    if result:
        long_url = result[0]
        cache.setex(f"url:{short_key}", 86400, long_url)
        return long_url
    
    return None  # 404
```

---

## Trade-offs e Decisões

| Decisão | Opção A | Opção B | Recomendação |
|---------|---------|---------|:------------:|
| **ID Generation** | Hash | Counter/KGS | KGS (sem colisões) |
| **Redirect** | 301 Permanent | 302 Temporary | 302 (analytics) |
| **DB** | SQL (PostgreSQL) | NoSQL (Cassandra) | NoSQL para escala |
| **Cache** | Redis | Memcached | Redis (TTL nativo) |
| **Expiration** | Active cleanup | Lazy + TTL | Lazy + batch cleanup |
| **Analytics** | Sync (na request) | Async (Kafka) | Async (não bloqueia) |

---

## Uso em Big Techs

| Empresa | Serviço | Escala | Detalhes |
|---------|---------|--------|----------|
| **Bitly** | bit.ly | 500M clicks/mês | Counter-based IDs |
| **Google** | goo.gl (deprecated) | Bilhões de URLs | — |
| **Twitter/X** | t.co | Cada link no Twitter | Auto-shorten em posts |
| **Meta** | fb.me, m.me | Links de Messenger | Internal shortener |
| **YouTube** | youtu.be | Shortener de vídeos | Fixed format |
| **Amazon** | amzn.to | Links de produtos | Product shortener |
| **TinyURL** | tinyurl.com | Um dos primeiros | Simple hash-based |

---

## Perguntas Frequentes em Entrevistas

1. **"Como gerar short URLs únicas?"**
   - KGS (pré-gerados), counter + base62, hash + collision check
   - 7 chars base62 = 3.5T combinações (suficiente por décadas)

2. **"301 vs 302 redirect?"**
   - 301: browser cacheia, menos carga, analytics incompleto
   - 302: cada click passa pelo server, analytics completo
   - Use 302 se analytics é importante

3. **"Como escalar para 100K+ QPS?"**
   - Cache (Redis) para hot URLs
   - Stateless servers → horizontal scaling
   - DB sharding por short_key

4. **"Como lidar com URLs duplicadas?"**
   - Opção 1: mesma long URL → mesmo short URL (dedup)
   - Opção 2: cada request gera novo short URL
   - Trade-off: storage vs lookup complexity

5. **"Como implementar expiration?"**
   - Lazy: check expires_at no redirect (zero custo extra)
   - Background job: batch cleanup de URLs expiradas (libera storage)

---

## Referências

- Alex Xu — *"System Design Interview"*, Cap. 8: Design a URL Shortener
- Bitly Engineering — *"Building a Scalable URL Shortener"*
- System Design Primer — *"Design TinyURL"*
- Martin Kleppmann — *"Designing Data-Intensive Applications"*
- Grokking the System Design Interview — *"Designing a URL Shortening Service"*
