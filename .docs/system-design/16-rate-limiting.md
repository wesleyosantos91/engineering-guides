# 16. Rate Limiting e Throttling

> **Categoria:** Resiliência e Proteção de Sistemas  
> **Nível:** Essencial para qualquer entrevista de System Design  
> **Complexidade:** Média

---

## Definição

**Rate Limiting** é o controle da quantidade de requests que um client pode fazer em um intervalo de tempo. **Throttling** é o mecanismo de enforcement — reduzir ou bloquear requests quando o limite é excedido. Juntos, protegem contra abuso, DDoS, e garantem **fair usage** dos recursos.

---

## Por Que é Importante?

- **Proteção contra DDoS** — limita impacto de ataques
- **Fair usage** — evita que um client monopolize recursos
- **Estabilidade do sistema** — previne sobrecarga de backends
- **Controle de custos** — limita uso de APIs pagas (LLMs, SMS, etc.)
- **Compliance** — SLAs e contratos de API exigem rate limits

---

## Diagrama de Arquitetura

```
  ┌────────────┐        ┌──────────────────┐        ┌──────────────┐
  │   Client   │──req──▶│   Rate Limiter   │──req──▶│   Backend    │
  │            │        │                  │        │   Service    │
  │            │◀─resp──│  ┌────────────┐  │◀─resp──│              │
  └────────────┘        │  │ Counter /  │  │        └──────────────┘
                        │  │ Token Store│  │
                        │  │  (Redis)   │  │
                        │  └────────────┘  │
                        └──────────────────┘
                               │
                        Se limite excedido:
                        HTTP 429 Too Many Requests
```

### Onde Colocar o Rate Limiter

```
  Opção 1: API Gateway (mais comum)
  
  Client ──▶ [API Gateway + Rate Limiter] ──▶ Services
              (Kong, AWS API GW, Envoy)
  
  Opção 2: Middleware na aplicação
  
  Client ──▶ [Service] ──▶ [Rate Limit Middleware] ──▶ [Handler]
  
  Opção 3: Service Mesh (sidecar)
  
  Client ──▶ [Envoy Sidecar] ──▶ [Service]
              (rate limit filter)
  
  Opção 4: Load Balancer
  
  Client ──▶ [NGINX/HAProxy] ──▶ Services
              (limit_req module)
```

---

## Algoritmos de Rate Limiting

### 1. Token Bucket

```
  Bucket capacity: 10 tokens
  Refill rate: 2 tokens/second
  
  ┌──────────────────────┐
  │   Token Bucket       │
  │                      │
  │   ● ● ● ● ● ● ●    │  7 tokens disponíveis
  │   (capacity: 10)     │
  │                      │
  │   Refill: +2/sec     │
  └──────────────────────┘
  
  Cada request consome 1 token:
  
  t=0:  [●●●●●●●●●●] 10 tokens → request OK (9 tokens)
  t=0:  [●●●●●●●●●·] 9 tokens  → request OK (8 tokens)
  ...
  t=0:  [··········] 0 tokens   → request REJECTED (429)
  t=1:  [●●········] 2 tokens   → refilled! request OK
  
  Propriedades:
  ✅ Permite bursts (até capacity)
  ✅ Taxa média controlada (refill rate)
  ✅ Simples de implementar
  ✅ Usado por: AWS, Stripe, GitHub
```

### 2. Leaky Bucket

```
  Bucket capacity: 10 requests
  Leak rate: 2 requests/second (processamento constante)
  
  Requests entram no topo, saem pelo fundo a taxa constante:
  
        ▼ ▼ ▼ (requests chegam em burst)
  ┌──────────────┐
  │   req5       │
  │   req4       │  Fila FIFO
  │   req3       │
  │   req2       │
  │   req1       │
  └──────┬───────┘
         │ (leak: 2 req/sec, constante)
         ▼
     Processing
  
  Se bucket está cheio → request REJECTED
  
  Propriedades:
  ✅ Output suave e constante (sem bursts)
  ✅ Bom para APIs que precisam de taxa estável
  ❌ Não permite bursts legítimos
  ❌ Requests em burst ficam na fila (maior latência)
```

### 3. Fixed Window Counter

```
  Limite: 100 requests por minuto
  
  Janela 1 (00:00 - 00:59)    Janela 2 (01:00 - 01:59)
  ┌─────────────────────┐     ┌─────────────────────┐
  │ count: 87           │     │ count: 0 (reset)    │
  │ limit: 100          │     │ limit: 100          │
  │ ✅ OK               │     │                     │
  └─────────────────────┘     └─────────────────────┘
  
  Problema de borda (boundary burst):
  
  00:30 - 00:59: 100 requests (✅ dentro do limite)
  01:00 - 01:29: 100 requests (✅ dentro do limite)
  
  Mas: 200 requests em 1 minuto (00:30 - 01:29)! → DOBRO do limite
  
  ─────┤════════════════════════════════════════├─────────
       00:00        00:30   01:00        01:30
                    ╰─── 200 requests em 60s ───╯
```

### 4. Sliding Window Log

```
  Limite: 5 requests por minuto
  Mantém timestamp de cada request:
  
  Log: [10:00:01, 10:00:15, 10:00:30, 10:00:45, 10:00:58]
  
  Nova request em 10:01:10:
  1. Remove timestamps > 1 min atrás (antes de 10:00:10)
     → Remove 10:00:01
     → Log: [10:00:15, 10:00:30, 10:00:45, 10:00:58]
  2. Count = 4 < 5 → ✅ ALLOWED
  3. Adiciona: [10:00:15, 10:00:30, 10:00:45, 10:00:58, 10:01:10]
  
  Propriedades:
  ✅ Precisão perfeita (sem boundary burst problem)
  ❌ Usa mais memória: O(n) onde n = requests na janela
  ❌ Mais lento que fixed window
```

### 5. Sliding Window Counter

```
  Combina Fixed Window + peso proporcional
  
  Limite: 100 requests/minuto
  
  Janela anterior (00:00-00:59): 84 requests
  Janela atual (01:00-01:59): 36 requests
  
  Request chega em 01:15 (25% da janela atual)
  
  Estimated count = 84 × 75% + 36 × 25%
                  = 84 × 0.75 + 36 × 0.25
                  = 63 + 9 = 72
  
  72 < 100 → ✅ ALLOWED
  
  ──────┤═══════════════╪════════════════├──────
       00:00         01:00  01:15     01:59
                      ╰───────╯
                      75% da janela anterior
                           ╰──╯
                           25% da janela atual
  
  Propriedades:
  ✅ Bom equilíbrio precisão/memória
  ✅ Resolve boundary burst (aproximação)
  ✅ Memória constante: O(1) — apenas 2 contadores
  ⚠️ Não é 100% preciso (aproximação)
```

### Comparativo de Algoritmos

| Algoritmo | Memória | Precisão | Burst | Complexidade |
|-----------|---------|----------|-------|--------------|
| **Token Bucket** | O(1) | Boa | Permite | Simples |
| **Leaky Bucket** | O(bucket size) | Boa | Não permite | Simples |
| **Fixed Window** | O(1) | Boundary issue | Na borda | Muito simples |
| **Sliding Window Log** | O(n) | Perfeita | Não | Moderada |
| **Sliding Window Counter** | O(1) | Boa (aprox.) | Controlado | Simples |

---

## Implementação com Redis

### Token Bucket (Lua Script — Atômico)

```lua
-- rate_limit.lua (executado atomicamente no Redis)
local key = KEYS[1]
local rate = tonumber(ARGV[1])      -- tokens per second
local capacity = tonumber(ARGV[2])  -- max tokens (burst size)
local now = tonumber(ARGV[3])       -- current timestamp (seconds)
local requested = tonumber(ARGV[4]) -- tokens requested (default 1)

-- Obter estado atual
local data = redis.call('HMGET', key, 'tokens', 'timestamp')
local tokens = tonumber(data[1]) or capacity
local last = tonumber(data[2]) or now

-- Calcular tokens acumulados desde última request
local elapsed = math.max(0, now - last)
tokens = math.min(capacity, tokens + elapsed * rate)

-- Verificar se tem tokens suficientes
if tokens < requested then
    -- Calcular quanto tempo até ter tokens suficientes
    local wait = (requested - tokens) / rate
    redis.call('HMSET', key, 'tokens', tokens, 'timestamp', now)
    redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)
    return {0, tokens, wait}  -- REJECTED, remaining, retry_after
end

-- Consumir tokens
tokens = tokens - requested
redis.call('HMSET', key, 'tokens', tokens, 'timestamp', now)
redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)
return {1, tokens, 0}  -- ALLOWED, remaining, no wait
```

### Sliding Window Counter (Redis)

```python
import redis
import time

def is_allowed(redis_client, key, limit, window_seconds):
    """Sliding window counter rate limiter"""
    now = time.time()
    window_start = now - window_seconds
    pipe = redis_client.pipeline()
    
    # Remove entries fora da janela
    pipe.zremrangebyscore(key, 0, window_start)
    # Conta entries na janela
    pipe.zcard(key)
    # Adiciona nova entry
    pipe.zadd(key, {f"{now}:{id(now)}": now})
    # Set TTL
    pipe.expire(key, window_seconds)
    
    results = pipe.execute()
    current_count = results[1]
    
    if current_count >= limit:
        # Remove a entry que acabamos de adicionar
        redis_client.zremrangebyscore(key, now, now)
        return False  # REJECTED
    
    return True  # ALLOWED

# Uso: 100 requests por minuto por user
allowed = is_allowed(redis, f"rate:{user_id}", limit=100, window_seconds=60)
```

---

## HTTP Headers e Response

### Request Aceito

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 73
X-RateLimit-Reset: 1672531200
```

### Request Rejeitado

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1672531200
Retry-After: 30
Content-Type: application/json

{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Please retry after 30 seconds.",
  "retry_after": 30
}
```

### Headers Padronizados (IETF Draft)

| Header | Significado |
|--------|-------------|
| `X-RateLimit-Limit` | Requests permitidos na janela |
| `X-RateLimit-Remaining` | Requests restantes na janela |
| `X-RateLimit-Reset` | Unix timestamp de quando a janela reseta |
| `Retry-After` | Segundos até poder tentar novamente |

---

## Rate Limiting Distribuído

### Desafios

```
  Cenário: 2 instâncias do API Gateway, limite = 100 req/min
  
  Sem coordenação:
  ┌──────────────────┐     ┌──────────────────┐
  │ Gateway A        │     │ Gateway B        │
  │ count: 80/100    │     │ count: 70/100    │
  │ (✅ OK local)    │     │ (✅ OK local)    │
  └──────────────────┘     └──────────────────┘
  
  Total real: 150 requests! → Excedeu o limite global de 100
```

### Soluções

| Abordagem | Como | Prós | Contras |
|-----------|------|------|---------|
| **Centralized (Redis)** | Todos consultam Redis | Preciso, global | Redis = SPOF, latência de rede |
| **Local + Sync** | Cada nó tem quota, synca periodicamente | Rápido localmente | Impreciso entre syncs |
| **Sticky Sessions** | Mesmo client → mesmo gateway | Simples | Distribuição desigual |
| **Token Bucket Global** | Redis-backed token bucket | Preciso com burst | Latência do Redis |

```
  Centralized (Redis) — Mais comum:
  
  Client ──▶ Gateway A ──▶ Redis: INCR rate:{user_id}:{window}
  Client ──▶ Gateway B ──▶ Redis: INCR rate:{user_id}:{window}
  
  Ambos consultam o MESMO counter no Redis
  → Contagem global precisa
```

---

## Níveis de Rate Limiting

```
  ┌─────────────────────────────────────────────────────┐
  │  Nível 1: Por IP                                    │
  │  → Proteção básica contra DDoS                      │
  │  → Exemplo: 1000 req/min por IP                     │
  ├─────────────────────────────────────────────────────┤
  │  Nível 2: Por API Key / Token                       │
  │  → Fair usage entre clientes                        │
  │  → Exemplo: Free=100/hr, Pro=10000/hr               │
  ├─────────────────────────────────────────────────────┤
  │  Nível 3: Por Endpoint                              │
  │  → Proteger endpoints caros (search, export, AI)    │
  │  → Exemplo: /api/search = 10 req/min                │
  ├─────────────────────────────────────────────────────┤
  │  Nível 4: Por User                                  │
  │  → Limites personalizados por plano                 │
  │  → Exemplo: user:123 = 5000 req/hr (Enterprise)     │
  ├─────────────────────────────────────────────────────┤
  │  Nível 5: Global / Service                          │
  │  → Proteger o backend como um todo                  │
  │  → Exemplo: Total 1M req/min para o serviço         │
  └─────────────────────────────────────────────────────┘
```

---

## Throttling Strategies

| Estratégia | Descrição | Quando Usar |
|------------|-----------|-------------|
| **Hard Throttle** | Rejeita imediatamente (429) | APIs públicas, proteção DDoS |
| **Soft Throttle** | Permite X% acima do limite | Bursty workloads, grace period |
| **Elastic/Dynamic** | Limite ajusta baseado em carga do backend | Auto-scaling, degradação graciosa |
| **Queueing** | Excedente vai para fila em vez de rejeitar | Batch processing, background jobs |

```
  Dynamic Rate Limiting:
  
  Backend CPU < 50%:  limit = 1000 req/sec  (normal)
  Backend CPU 50-80%: limit = 500 req/sec   (slowing down)
  Backend CPU > 80%:  limit = 100 req/sec   (protecting)
  
  ┌──────────────┐     health check     ┌───────────┐
  │ Rate Limiter │◀════════════════════▶│  Backend  │
  │              │  "CPU: 85%"          │           │
  │ limit: 100  │                       │           │
  └──────────────┘                      └───────────┘
```

---

## Client-Side Best Practices

### Exponential Backoff with Jitter

```python
import random
import time

def request_with_retry(url, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url)
        
        if response.status_code != 429:
            return response
        
        # Exponential backoff with jitter
        base_delay = min(2 ** attempt, 32)  # 1, 2, 4, 8, 16, 32
        jitter = random.uniform(0, base_delay)
        delay = base_delay + jitter
        
        # Prefer Retry-After header if available
        retry_after = response.headers.get('Retry-After')
        if retry_after:
            delay = float(retry_after)
        
        print(f"Rate limited. Retrying in {delay:.1f}s (attempt {attempt+1})")
        time.sleep(delay)
    
    raise Exception("Max retries exceeded")
```

---

## Tecnologias

| Tecnologia | Tipo | Rate Limiting | Algoritmo |
|------------|------|---------------|-----------|
| **Kong** | API Gateway | Plugin nativo | Sliding window, Redis-backed |
| **AWS API Gateway** | Managed Gateway | Nativo | Token bucket |
| **NGINX** | Reverse Proxy | `limit_req` module | Leaky bucket |
| **Envoy** | Service Proxy | Rate limit filter | Token bucket, external service |
| **Redis** | Data Store | Lua scripts / RedisCell | Qualquer algoritmo |
| **Cloudflare** | CDN/WAF | Rate Limiting Rules | Fixed/Sliding window |
| **Traefik** | Reverse Proxy | Rate limit middleware | Token bucket |
| **Resilience4j** | Library (Java) | In-process | Sliding window |

### NGINX Rate Limiting

```nginx
http {
    # Define rate limit zone
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    
    server {
        location /api/ {
            # burst=20: permite 20 requests extras na fila
            # nodelay: processa burst imediatamente (não enfileira)
            limit_req zone=api burst=20 nodelay;
            
            # Custom 429 response
            limit_req_status 429;
            
            proxy_pass http://backend;
        }
    }
}
```

---

## Uso em Big Techs

### GitHub — API Rate Limiting
- **Authenticated:** 5000 requests/hour por token
- **Unauthenticated:** 60 requests/hour por IP
- **Search API:** 30 requests/minute (mais restritivo)
- **Headers:** `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- **Secondary rate limits:** Limite de criação de conteúdo (anti-spam)

### Twitter/X — Multi-Level Limits
- **Rate limits por endpoint:** Cada endpoint tem seu próprio limite
- **App-level + User-level:** Limites separados por app e por user
- **Tiers:** Basic (free), Pro, Enterprise com limites crescentes
- **Rate limit windows:** 15 minutos para maioria dos endpoints

### Stripe — API Rate Limiting
- **Live mode:** 100 requests/second por key
- **Test mode:** 25 requests/second
- **Idempotency keys:** Retries seguros sem side effects
- **Graceful degradation:** Slow down antes de hard limit
- **Best practice:** Exponential backoff + jitter

### Amazon — API Gateway Throttling
- **Account-level:** 10,000 requests/second (default)
- **Per-method:** Configurável por API method
- **Usage Plans:** Diferentes limites por API key/plan
- **Burst:** Token bucket com burst capacity
- **WAF integration:** Rate-based rules para DDoS protection

### Google — API Quotas
- **Per-project quotas:** Limites por Google Cloud project
- **Per-user quotas:** Limites por usuário final
- **Billable APIs:** Quota baseada em billing account
- **Quota increase:** Processo de request para aumentar limites

---

## Perguntas Comuns em Entrevistas

1. **Design um rate limiter distribuído.**
   - Redis-backed sliding window counter, API gateway como enforcement point

2. **Qual algoritmo escolher para permitir bursts?**
   - Token bucket (burst até capacity, taxa média controlada)

3. **Como lidar com rate limiting em múltiplas instâncias?**
   - Centralized counter (Redis) ou local + periodic sync

4. **Como um client deve reagir a um 429?**
   - Exponential backoff com jitter, respeitar Retry-After header

5. **Qual a diferença entre rate limiting e throttling?**
   - Rate limiting = regra (100 req/min); Throttling = ação (rejeitar/enfileirar)

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Algoritmo** | Token Bucket (flexível, bursts) | Fixed Window (simples) |
| **Storage** | Redis centralized (preciso) | In-memory local (rápido) |
| **Enforcement** | Hard reject (429) | Soft throttle (degrade gracefully) |
| **Granulidade** | Per-user (fair) | Per-IP (simples, não requer auth) |
| **Response** | Reject imediato | Queue e process later |
| **Limite** | Fixo (previsível) | Dinâmico (adapta à carga) |

---

## Referências

- [Stripe — Rate Limiting Best Practices](https://stripe.com/docs/rate-limits)
- [GitHub — Rate Limiting](https://docs.github.com/en/rest/rate-limit)
- [Cloudflare — Rate Limiting](https://developers.cloudflare.com/waf/rate-limiting-rules/)
- [Kong Rate Limiting Plugin](https://docs.konghq.com/hub/kong-inc/rate-limiting/)
- [NGINX Rate Limiting](https://www.nginx.com/blog/rate-limiting-nginx/)
- System Design Interview — Alex Xu, Vol. 1, Ch. 4: Design a Rate Limiter
