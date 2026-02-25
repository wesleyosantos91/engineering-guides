# 3. CDN (Content Delivery Network)

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial — presente em todo sistema distribuído com usuários globais  
> **Complexidade:** Média

---

## Definição

**CDN (Content Delivery Network)** é uma rede de servidores distribuídos geograficamente que cacheia e entrega conteúdo a partir do ponto mais próximo do usuário, reduzindo latência e aliviando carga do servidor de origem.

---

## Por Que é Importante?

| Sem CDN | Com CDN |
|---------|---------|
| Usuário no Brasil acessa server nos EUA (~150ms RTT) | Conteúdo servido de São Paulo (~5ms RTT) |
| Origin server recebe TODO o tráfego | 80-95% do tráfego resolvido no edge |
| Um pico de tráfego derruba o origin | CDN absorve picos (DDoS protection inclusa) |
| Custo alto em bandwidth no origin | Bandwidth distribuído |

---

## Arquitetura Geral

```
                         ┌─────────────────────┐
                         │   Origin Server     │
                         │  (App + DB)         │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
              ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐
              │  PoP São  │  │  PoP New  │  │  PoP      │
              │  Paulo    │  │  York     │  │  Tokyo    │
              │           │  │           │  │           │
              │ ┌───────┐ │  │ ┌───────┐ │  │ ┌───────┐ │
              │ │ Edge  │ │  │ │ Edge  │ │  │ │ Edge  │ │
              │ │Server │ │  │ │Server │ │  │ │Server │ │
              │ └───────┘ │  │ └───────┘ │  │ └───────┘ │
              └─────▲─────┘  └─────▲─────┘  └─────▲─────┘
                    │               │               │
              ┌─────┴─────┐  ┌─────┴─────┐  ┌─────┴─────┐
              │  Usuários │  │  Usuários │  │  Usuários │
              │  Brasil   │  │  USA      │  │  Japão    │
              └───────────┘  └───────────┘  └───────────┘
```

### Componentes

| Componente | Descrição |
|------------|-----------|
| **Origin Server** | Servidor original onde o conteúdo reside |
| **PoP (Point of Presence)** | Datacenter local com edge servers |
| **Edge Server** | Servidor dentro do PoP que cacheia e serve conteúdo |
| **DNS** | Resolve o domínio para o PoP mais próximo do usuário |

---

## Fluxo de Request

```
┌──────┐     ┌─────┐     ┌──────────┐     ┌────────────┐
│Client│────▶│ DNS │────▶│CDN Edge  │────▶│  Origin    │
│      │     │     │     │(nearest) │     │  Server    │
└──┬───┘     └─────┘     └────┬─────┘     └─────┬──────┘
   │                          │                  │
   │  1. DNS resolve          │                  │
   │  ──────────────▶         │                  │
   │  2. IP do edge           │                  │
   │  ◀──────────────         │                  │
   │                          │                  │
   │  3. Request ao edge      │                  │
   │  ───────────────────────▶│                  │
   │                          │                  │
   │     Cache HIT?           │                  │
   │     ┌─── SIM ───┐       │                  │
   │     │  Retorna   │       │                  │
   │     │  conteúdo  │       │                  │
   │  ◀──┘            │       │                  │
   │                          │                  │
   │     NÃO (MISS)          │                  │
   │                          │  4. Fetch origin │
   │                          │  ───────────────▶│
   │                          │  5. Response     │
   │                          │  ◀───────────────│
   │                          │  6. Cache + serve│
   │  ◀──────────────────────│                  │
   │  7. Response                                │
```

---

## Types: Push CDN vs Pull CDN

### Pull CDN (Lazy — mais comum)

O CDN busca conteúdo no origin **sob demanda** (quando recebe primeira request).

```
1. User request → CDN: "tenho essa imagem?"
2. CDN: MISS → busca no origin
3. CDN cacheia → retorna ao user
4. Próximos requests → servidos do cache
```

| Prós | Contras |
|------|---------|
| Zero configuração de upload | Primeiro request = lento (cache MISS) |
| Só cacheia o que é acessado | Origin precisa estar sempre disponível |
| Simples de operar | |

**Quando usar:** Maioria dos casos. Websites, APIs, assets dinâmicos.

### Push CDN

O desenvolvedor faz **upload direto** do conteúdo para o CDN.

```
1. Deploy: upload assets para CDN
2. CDN distribui para todos os PoPs
3. User request → CDN: já tem → serve
```

| Prós | Contras |
|------|---------|
| Zero cache MISS (tudo pré-populado) | Precisa gerenciar upload/invalidação |
| Origin pode ficar offline | Storage no CDN = custo |
| Controle total | Mais complexo de operar |

**Quando usar:** Conteúdo estático que muda raramente (vídeos, binários, assets de build).

---

## Cache Invalidation no CDN

| Método | Descrição | Velocidade |
|--------|-----------|------------|
| **TTL-based** | Conteúdo expira após tempo definido | Automático |
| **Purge** | Remover conteúdo específico manualmente | Imediato |
| **Versioning** | URL com hash/versão (`style.abc123.css`) | Imediato |
| **Soft Purge** | Marca como stale, serve enquanto revalida | Rápido |
| **Tag-based** | Purge por tag/grupo (`purge category:news`) | Imediato |

### Versioning (Mais recomendado para assets)

```html
<!-- Ruim: mesma URL, cache antigo pode persistir -->
<link rel="stylesheet" href="/style.css">

<!-- Bom: hash no filename (webpack, vite) -->
<link rel="stylesheet" href="/style.d8f3a2b1.css">

<!-- Alternativa: query string -->
<link rel="stylesheet" href="/style.css?v=20240125">
```

### Purge via API

```bash
# CloudFront
aws cloudfront create-invalidation \
  --distribution-id E1234 \
  --paths "/images/*" "/api/v1/products"

# Cloudflare
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone}/purge_cache" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"files":["https://example.com/style.css"]}'

# Fastly (Surrogate-Key based)
curl -X POST "https://api.fastly.com/service/{id}/purge/category:news" \
  -H "Fastly-Key: TOKEN"
```

---

## Edge Computing

CDNs modernos vão além de cache — executam **lógica no edge**:

| Serviço | Provider | Runtime |
|---------|----------|---------|
| **CloudFront Functions** | AWS | JavaScript (limitado) |
| **Lambda@Edge** | AWS | Node.js, Python |
| **Cloudflare Workers** | Cloudflare | V8 (JS, Wasm) |
| **Fastly Compute** | Fastly | Wasm |
| **Deno Deploy** | Deno | Deno/V8 |

**Casos de uso:**
- A/B testing no edge
- Geolocation-based routing
- Auth token validation
- Request/response transformation
- Personalization sem ir ao origin

```javascript
// Cloudflare Worker — Exemplo: geolocation redirect
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const country = request.headers.get('CF-IPCountry');
  
  if (country === 'BR') {
    return Response.redirect('https://br.example.com' + new URL(request.url).pathname);
  }
  
  return fetch(request);
}
```

---

## Métricas Importantes

| Métrica | Descrição | Target |
|---------|-----------|--------|
| **Cache Hit Ratio** | % requests servidas do cache | > 90% |
| **TTFB (Time To First Byte)** | Tempo até primeiro byte | < 100ms |
| **Bandwidth Offload** | % bandwidth economizado do origin | > 80% |
| **Origin Shield Hit Ratio** | % requests filtradas pelo shield | > 70% |
| **Purge Propagation Time** | Tempo para invalidar globalmente | < 10s |

---

## Tecnologias

| CDN | Tipo | PoPs | Diferencial |
|-----|------|------|-------------|
| **CloudFront** | Managed (AWS) | 450+ | Integração AWS, Lambda@Edge |
| **Cloudflare** | SaaS | 300+ | Workers (compute), DDoS, DNS |
| **Akamai** | Enterprise | 4000+ | Maior rede, enterprise features |
| **Fastly** | Developer | 90+ | VCL config, instant purge, compute |
| **Google Cloud CDN** | Managed (GCP) | Global | Integração GCP, Anycast |
| **Azure CDN** | Managed (Azure) | Global | Múltiplos providers (Verizon, Akamai) |

---

## Uso em Big Techs

### Netflix — Open Connect

```
Netflix constrói e opera seu PRÓPRIO CDN chamado Open Connect.

- Open Connect Appliances (OCAs): boxes físicas instaladas dentro dos ISPs
- Conteúdo popular é pré-posicionado durante off-peak hours
- > 95% do tráfego de vídeo servido dos OCAs
- Cada OCA: ~100-200TB storage, ~100Gbps throughput
- Netflix representa ~15% de TODO o tráfego de internet mundial
```

**Fluxo:**
```
1. User clica play no filme
2. Netflix control plane (AWS) → decide qual OCA tem o arquivo
3. Client conecta diretamente ao OCA dentro do ISP
4. Streaming servido do ISP local (latência mínima)
```

### Google — Global Cache (GGC)
- Caches instalados dentro de ISPs
- Serve YouTube videos e Google static assets
- Similar ao Open Connect da Netflix

### Meta (Facebook)
- CDN interno para fotos, vídeos
- Haystack (photo storage) + CDN para entrega
- Bilhões de imagens servidas por dia

### Akamai (fundou o conceito de CDN)
- 4000+ PoPs em 130+ países
- Serve 15-30% do tráfego web mundial
- Clientes: Apple, Microsoft, major banks

---

## Padrão: Origin Shield

Camada extra entre edge e origin que reduz requests ao origin:

```
Users → Edge PoPs → Origin Shield (1 região) → Origin Server

Sem shield:  10 PoPs × cache miss = 10 requests ao origin
Com shield:  10 PoPs → 1 shield → 1 request ao origin
```

---

## CDN para Dynamic Content

CDN não é só para static assets. Para API responses:

| Técnica | Descrição |
|---------|-----------|
| **Micro-caching** | Cache por 1-5 segundos (absorve spikes) |
| **Vary header** | Cache diferente por Accept-Language, Cookie |
| **ESI (Edge Side Includes)** | Compõe página com partes cacheable + dynamic |
| **Stale-while-revalidate** | Serve stale, renova em background |

```http
# API response cacheável por 10s no CDN
Cache-Control: public, max-age=10, s-maxage=10, stale-while-revalidate=60
Vary: Accept-Language, Authorization
```

---

## Perguntas Comuns em Entrevistas

1. **Pull vs Push CDN?** → Pull é on-demand (mais comum); Push é pré-upload (vídeos, builds)
2. **Como invalidar cache no CDN?** → Versioning (melhor), TTL, purge API, surrogate keys
3. **CDN para APIs?** → Sim, com micro-caching, stale-while-revalidate, Vary headers
4. **O que é Origin Shield?** → Camada intermediária que consolida cache misses antes do origin
5. **Netflix CDN?** → Open Connect: boxes físicas nos ISPs, pre-positioned content

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Pull vs Push** | Pull (simples, on-demand) | Push (zero MISS, mais controle) |
| **TTL** | Curto (fresh data) | Longo (mais offload) |
| **Single CDN vs Multi** | Um provider (simples) | Multi-CDN (redundância, best perf) |
| **Versioning vs Purge** | Versioning (previsível) | Purge (mantém URLs) |
| **Managed vs Private** | CloudFront/CF (zero-ops) | Open Connect style (controle total) |

---

## Referências

- [Netflix Open Connect](https://openconnect.netflix.com/)
- [Cloudflare Learning Center](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/)
- [AWS CloudFront Docs](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/)
- [Web Almanac - CDN](https://almanac.httparchive.org/en/2022/cdn)
- System Design Interview — Alex Xu, Vol 1, Chapter on CDN
