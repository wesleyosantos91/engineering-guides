# 31. Back-of-the-Envelope Estimation

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial — primeiro passo em toda entrevista de System Design  
> **Complexidade:** Média

---

## Definição

**Back-of-the-Envelope Estimation** é a habilidade de estimar rapidamente os recursos necessários (servidores, storage, bandwidth, QPS) para um sistema, usando aritmética simples e números de referência. É o **primeiro passo** em qualquer design de sistema — antes de desenhar caixas e setas, você precisa entender a **escala** do problema.

---

## Por Que é Importante?

- **Etapa obrigatória em entrevistas** — "Quantos servidores precisamos?" é a primeira pergunta
- **Evita over/under-engineering** — dimensiona o sistema corretamente
- **Demonstra maturidade** — mostra que você pensa em escala antes de codificar
- **Identifica bottlenecks** — storage vs compute vs network
- **Google, Meta, Amazon** — todos pedem estimativas em entrevistas

---

## Números que Todo Engenheiro Deve Saber

### Latência de Operações

```
┌─────────────────────────────────────────────────────────────┐
│  Operação                        │  Latência               │
├──────────────────────────────────┼─────────────────────────┤
│  L1 cache reference              │  ~0.5 ns                │
│  Branch mispredict               │  ~5 ns                  │
│  L2 cache reference              │  ~7 ns                  │
│  Mutex lock/unlock               │  ~25 ns                 │
│  Main memory reference           │  ~100 ns                │
│  Compress 1KB (Snappy)           │  ~3 μs                  │
│  Send 1KB over 1 Gbps network   │  ~10 μs                 │
│  Read 4KB random from SSD        │  ~150 μs                │
│  Read 1MB sequencial from memory │  ~250 μs                │
│  Round trip same datacenter      │  ~500 μs (0.5 ms)       │
│  Read 1MB sequencial from SSD    │  ~1 ms                  │
│  HDD disk seek                   │  ~10 ms                 │
│  Read 1MB sequencial from HDD    │  ~20 ms                 │
│  Send packet CA → Netherlands    │  ~150 ms                │
└──────────────────────────────────┴─────────────────────────┘

Conclusões:
  Memory é ~100x mais rápido que SSD
  SSD é ~100x mais rápido que HDD
  Network dentro do DC é ~300x mais rápido que cross-continent
  Leituras sequenciais sempre >> random reads
```

### Potências de 2

```
┌──────────┬──────────────────┬──────────────────┐
│ Potência │ Valor Exato      │ Aproximação      │
├──────────┼──────────────────┼──────────────────┤
│ 2^10     │ 1,024            │ ~1 Mil (1 KB)    │
│ 2^16     │ 65,536           │ ~65 Mil          │
│ 2^20     │ 1,048,576        │ ~1 Milhão (1 MB) │
│ 2^30     │ 1,073,741,824    │ ~1 Bilhão (1 GB) │
│ 2^32     │ 4,294,967,296    │ ~4 Bilhões       │
│ 2^40     │ ~1.1 × 10^12     │ ~1 Trilhão (1 TB)│
│ 2^50     │ ~1.1 × 10^15     │ ~1 Quadrilhão (1PB)│
└──────────┴──────────────────┴──────────────────┘
```

### Conversões de Tempo

```
1 segundo   = 1,000 ms = 1,000,000 μs = 1,000,000,000 ns
1 dia       = 86,400 s  ≈ ~100K s   (arredonde para facilitar)
1 mês       = ~2.5M s
1 ano       = ~30M s    ≈ ~31.5M s

Regra prática:
  1M requests/dia ≈ 12 QPS
  1B requests/dia ≈ 12,000 QPS
  QPS = requests_total / 86,400
  Peak QPS ≈ 2x ~ 3x average QPS
```

### Throughput de Sistemas

```
┌──────────────────────────────────┬──────────────────┐
│  Componente                      │  Throughput       │
├──────────────────────────────────┼──────────────────┤
│  SSD sequential read             │  ~1 GB/s          │
│  HDD sequential read             │  ~200 MB/s        │
│  1 Gbps network                  │  ~125 MB/s        │
│  10 Gbps network                 │  ~1.25 GB/s       │
│  Redis (single node)             │  ~100K ops/s      │
│  Kafka (single broker)           │  ~1M msgs/s       │
│  MySQL (single server, OLTP)     │  ~5K-10K QPS      │
│  PostgreSQL (single server)      │  ~5K-15K QPS      │
│  Nginx (static)                  │  ~50K req/s       │
│  Web application server          │  ~1K-5K req/s     │
│  Cassandra (single node)         │  ~10K-50K ops/s   │
│  Elasticsearch                   │  ~5K-20K queries/s│
└──────────────────────────────────┴──────────────────┘
```

### Tamanhos de Dados

```
┌──────────────────────────────────┬──────────────────┐
│  Dado                            │  Tamanho          │
├──────────────────────────────────┼──────────────────┤
│  1 char (ASCII)                  │  1 byte           │
│  1 char (UTF-8, avg)             │  2 bytes          │
│  UUID                            │  16 bytes         │
│  Timestamp (unix epoch, 64-bit)  │  8 bytes          │
│  Integer (64-bit)                │  8 bytes          │
│  Tweet (280 chars, UTF-8)        │  ~560 bytes       │
│  Metadata típica (JSON)          │  ~500 bytes - 2KB │
│  Foto thumbnail (150x150)        │  ~15 KB           │
│  Foto HD (1920x1080, JPEG)       │  ~300 KB - 2 MB   │
│  Foto smartphone                 │  ~3-5 MB          │
│  1 min de vídeo (HD)             │  ~50-150 MB       │
│  1 min de áudio (MP3, 128kbps)   │  ~1 MB            │
└──────────────────────────────────┴──────────────────┘
```

---

## Framework de Estimativa

### Template em 5 Passos

```
PASSO 1: Definir métricas base
  - DAU (Daily Active Users)
  - Ações por user/dia
  
PASSO 2: Calcular volume de operações
  - Total requests/dia = DAU × ações/user
  - QPS = requests/dia ÷ 86,400
  - Peak QPS = QPS × 2 ou 3
  
PASSO 3: Estimar storage
  - Tamanho médio por registro
  - Storage/dia = registros/dia × tamanho
  - Storage/ano = storage/dia × 365 (+ replicação)
  
PASSO 4: Estimar bandwidth
  - Ingress = write_QPS × tamanho_médio_request
  - Egress = read_QPS × tamanho_médio_response
  
PASSO 5: Calcular recursos
  - Servers = Peak QPS ÷ QPS_por_server
  - Cache = hot data size (geralmente 20% do total)
  - DB nodes = total storage ÷ storage_por_node
```

---

## Exemplos Práticos

### Exemplo 1: Twitter-like Timeline

```
🎯 Problema: Quantos servidores para servir timelines?

PASSO 1: Métricas base
  DAU: 300M
  Timeline reads/user/dia: 10
  New tweets/dia: 500M

PASSO 2: QPS
  Timeline reads: 300M × 10 = 3B/dia
  Read QPS: 3B / 86,400 ≈ 35,000 QPS
  Peak Read QPS: 35,000 × 3 ≈ 100,000 QPS
  
  Write QPS: 500M / 86,400 ≈ 6,000 QPS
  Peak Write QPS: 6,000 × 3 ≈ 18,000 QPS

PASSO 3: Storage
  Tweet size: ~300 bytes texto + 200 bytes metadata = 500 bytes
  Storage/dia: 500M × 500B = 250 GB
  Storage/ano: 250 GB × 365 = 91 TB
  Com 3x replicação: ~273 TB

PASSO 4: Bandwidth
  Egress (read-heavy):
    100K QPS × 5 tweets/page × 500B = 250 MB/s ≈ 2 Gbps
  
PASSO 5: Servers
  Se cada server suporta 3,000 QPS (app server com cache):
  Servers para reads: 100,000 / 3,000 ≈ 34 servers
  Servers para writes: 18,000 / 3,000 ≈ 6 servers
  
  Cache (Redis):
    Hot tweets (últimas 24h): 500M × 500B = 250 GB
    20% hot: ~50 GB → redistribuído em Redis cluster

📊 Resumo:
  ~40 app servers
  ~273 TB storage/ano
  ~50 GB cache (Redis)
  ~2 Gbps egress
```

### Exemplo 2: Instagram-like Photo Storage

```
🎯 Problema: Quanto storage para fotos?

PASSO 1: Métricas base
  DAU: 500M
  Photos uploaded/dia: 100M
  
PASSO 2: Storage
  Cada foto gera múltiplos tamanhos:
    Original: 3 MB
    Large (1080p): 500 KB
    Medium (640p): 200 KB
    Thumbnail: 15 KB
    Total per photo: ~3.7 MB ≈ 4 MB
  
  Storage/dia: 100M × 4 MB = 400 TB/dia
  Storage/ano: 400 TB × 365 = 146 PB
  Com 3x replicação: ~438 PB 🤯

PASSO 3: Bandwidth
  Upload (ingress):
    100M fotos/dia → 1,157/s
    1,157 × 4 MB = 4.6 GB/s ≈ 37 Gbps ingress

  Reads (egress) — assumindo 10:1 read:write:
    ~46 GB/s ≈ 370 Gbps egress → CDN essencial!

PASSO 4: CDN
  Se CDN absorve 80% dos reads:
    Origem serve: 370 × 0.2 = 74 Gbps
    CDN serve: 370 × 0.8 = 296 Gbps

📊 Resumo:
  ~400 TB storage/dia
  ~146 PB storage/ano (antes de replicação)
  CDN é ABSOLUTAMENTE necessário
  Image processing pipeline é bottleneck
```

### Exemplo 3: URL Shortener

```
🎯 Problema: Quanto espaço para 100M URLs/dia?

PASSO 1: Métricas base
  URLs created/dia: 100M
  Read:Write ratio: 10:1

PASSO 2: QPS
  Write: 100M / 86,400 ≈ 1,200 QPS
  Read: 1,200 × 10 = 12,000 QPS
  Peak: 12,000 × 3 = 36,000 QPS

PASSO 3: Storage
  Cada registro:
    short_url (7 chars): 7 bytes
    long_url (avg 200 chars): 200 bytes
    metadata: ~100 bytes
    Total: ~307 bytes ≈ 500 bytes (com overhead)
  
  Storage/dia: 100M × 500B = 50 GB
  Storage/5 anos: 50 GB × 365 × 5 = 91 TB

PASSO 4: Short URL space
  7 chars base62 (a-z, A-Z, 0-9): 62^7 = 3.5 trilhões
  100M/dia × 365 × 10 anos = 365B URLs
  3.5T >> 365B → espaço suficiente ✅

PASSO 5: Cache
  80/20 rule: 20% das URLs recebem 80% do tráfego
  Hot URLs/dia: 100M × 0.2 = 20M
  Cache: 20M × 500B = 10 GB → cabe em um Redis ✅

📊 Resumo:
  ~36K peak QPS (servível com cache + poucos servers)
  ~91 TB em 5 anos
  ~10 GB cache
  62^7 é mais que suficiente para IDs
```

---

## Dicas para Entrevistas

```
1. ARREDONDE agressivamente:
   ✅ 86,400 ≈ 100,000
   ✅ 2.5M ≈ 3M
   ✅ Usar potências de 10

2. MOSTRE o raciocínio, não só o resultado:
   ❌ "Precisamos de 50 servers"
   ✅ "300M DAU × 10 req/user = 3B/dia ÷ 100K = 30K QPS..."

3. Identifique o BOTTLENECK:
   - É storage? → Sharding, compression, tiered storage
   - É compute? → Mais servers, caching
   - É bandwidth? → CDN, compression, caching
   - É latência? → Caching, edge, data locality

4. Use analogias com números conhecidos:
   "Redis faz 100K ops/s, precisamos de 300K QPS,
    então ~3 Redis nodes para o hot path"

5. Sempre calcule PEAK:
   Average QPS × 2-3 = Peak QPS
   Design para peak, não para average

6. Pergunte ao entrevistador:
   "Podemos assumir DAU de 100M?"
   "Read:write ratio de 10:1 é razoável?"
```

---

## Cheat Sheet de Estimativa Rápida

```
┌───────────────────────────────────────────────────────┐
│  Escala                         │  QPS (1 req/day)    │
├─────────────────────────────────┼─────────────────────┤
│  1M DAU                        │  ~12 QPS             │
│  10M DAU                       │  ~120 QPS            │
│  100M DAU                      │  ~1,200 QPS          │
│  1B DAU                        │  ~12,000 QPS         │
└─────────────────────────────────┴─────────────────────┘

  Multiplique por requests/user para QPS real
  Multiplique por 2-3x para Peak QPS

┌───────────────────────────────────────────────────────┐
│  Atalho                         │  Valor              │
├─────────────────────────────────┼─────────────────────┤
│  1 server handles               │  ~1K-5K QPS (app)   │
│  Redis handles                  │  ~100K ops/s         │
│  Kafka handles                  │  ~1M msgs/s          │
│  1 DB server handles            │  ~5K-15K QPS         │
│  1 GB network = 125 MB/s       │                       │
│  80/20 rule for caching         │  20% data = 80% hits │
│  Rule of 3 for replication      │  3x storage           │
└─────────────────────────────────┴─────────────────────┘
```

---

## Uso em Big Techs

| Empresa | Cenário | Escala |
|---------|---------|--------|
| **Google** | Search: 8.5B queries/dia | ~100K QPS |
| **Meta** | Facebook: 2B DAU, 300PB+ data | Exabyte-scale storage |
| **Twitter/X** | 500M tweets/dia | ~6K write QPS |
| **YouTube** | 500h vídeo uploaded/min | ~700TB/dia raw |
| **Netflix** | 15% de toda bandwidth da internet | ~Tbps egress |
| **Uber** | 20M rides/dia | ~250 QPS |
| **WhatsApp** | 100B messages/dia | ~1.2M QPS |

---

## Perguntas Frequentes em Entrevistas

1. **"Como você estimaria quantos servidores precisamos?"**
   - Calcule QPS (DAU × requests/user / 86400)
   - Peak QPS = 2-3x average
   - Servers = Peak QPS / capacidade por server

2. **"Quanto storage precisamos para 5 anos?"**
   - Data/dia = volume × tamanho_médio
   - × 365 × 5 anos × replicação (3x)

3. **"Qual é o bottleneck do sistema?"**
   - Compare: QPS vs server capacity, storage growth vs budget,
     bandwidth vs network capacity

4. **"Por que 62^7 é suficiente para URL shortener?"**
   - 62^7 = 3.5 trilhões combinações
   - Mesmo a 1B URLs/ano, dura 3.500 anos

---

## Referências

- Jeff Dean — *"Numbers Everyone Should Know"* — Google
- Alex Xu — *"System Design Interview"*, Cap. 2: Back-of-the-Envelope Estimation
- ByteByteGo — *"Latency Numbers Every Programmer Should Know"*
- Colin Scott — *"Latency Numbers"* — interactive (colin-scott.github.io)
- Werner Vogels — *"All Things Distributed"* — Amazon CTO blog
