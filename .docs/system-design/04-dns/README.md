# 4. DNS (Domain Name System)

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial — base de toda comunicação na internet  
> **Complexidade:** Média

---

## Definição

**DNS (Domain Name System)** é o sistema hierárquico e distribuído que traduz nomes de domínio legíveis por humanos (ex: `google.com`) em endereços IP (ex: `142.250.80.46`) usados por computadores para se comunicarem na rede.

> DNS é frequentemente chamado de **"a lista telefônica da internet"**.

---

## Por Que é Importante em System Design?

| Aspecto | Relevância |
|---------|-----------|
| **Disponibilidade** | Se DNS falha, nada funciona (single point of failure da internet) |
| **Performance** | DNS lookup adiciona latência a toda primeira request |
| **Load Balancing** | DNS Round Robin distribui tráfego entre servidores |
| **Disaster Recovery** | DNS failover redireciona tráfego em caso de falha |
| **Global Routing** | GeoDNS direciona usuários ao datacenter mais próximo |

---

## Hierarquia DNS

```
                    ┌──────────────┐
                    │  Root DNS    │  (13 logical root servers)
                    │  Servers (.) │  "Quem gerencia .com?"
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  TLD DNS     │  (.com, .org, .net, .br)
                    │  Servers     │  "Quem gerencia google.com?"
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ Authoritative│  (ns1.google.com)
                    │ DNS Servers  │  "google.com = 142.250.80.46"
                    └──────────────┘
```

| Nível | Descrição | Exemplo | Quantidade |
|-------|-----------|---------|------------|
| **Root** | Topo da hierarquia | `.` | 13 root server clusters |
| **TLD** | Top-Level Domain | `.com`, `.org`, `.br` | ~1500 TLDs |
| **Authoritative** | DNS definitivo do domínio | `ns1.google.com` | Milhões |
| **Recursive Resolver** | Intermediário que faz as queries | ISP DNS, 8.8.8.8 | Milhares |

---

## Fluxo de Resolução DNS

```
┌────────┐   ┌────────────┐   ┌──────┐   ┌──────┐   ┌──────────────┐
│Browser │──▶│ Recursive  │──▶│ Root │──▶│ TLD  │──▶│Authoritative │
│        │   │ Resolver   │   │ DNS  │   │ DNS  │   │   DNS        │
└────────┘   └────────────┘   └──────┘   └──────┘   └──────────────┘
    │              │              │          │              │
    │ 1. "api.    │              │          │              │
    │  example.   │              │          │              │
    │  com = ?"   │              │          │              │
    │────────────▶│              │          │              │
    │             │ 2. Verifica  │          │              │
    │             │    cache     │          │              │
    │             │              │          │              │
    │             │ Se MISS:     │          │              │
    │             │ 3. "Quem     │          │              │
    │             │  sabe .com?" │          │              │
    │             │─────────────▶│          │              │
    │             │ 4. "Pergunte │          │              │
    │             │  ao TLD .com"│          │              │
    │             │◀─────────────│          │              │
    │             │              │          │              │
    │             │ 5. "Quem é   │          │              │
    │             │  example.com?"          │              │
    │             │────────────────────────▶│              │
    │             │ 6. "Pergunte ao        │              │
    │             │  ns1.example.com"      │              │
    │             │◀────────────────────────│              │
    │             │              │          │              │
    │             │ 7. "IP de api.example.com?"            │
    │             │───────────────────────────────────────▶│
    │             │ 8. "142.250.80.46, TTL=300"           │
    │             │◀───────────────────────────────────────│
    │             │              │          │              │
    │ 9. Cacheia  │              │          │              │
    │   + retorna │              │          │              │
    │◀────────────│              │          │              │
```

**Tempo total sem cache:** ~100-200ms (4 hops de rede)  
**Com cache:** ~0-1ms (resultado cacheado localmente)

---

## Tipos de DNS Records

| Record | Tipo | Descrição | Exemplo |
|--------|------|-----------|---------|
| **A** | Address | Mapeia domínio → IPv4 | `example.com → 93.184.216.34` |
| **AAAA** | IPv6 Address | Mapeia domínio → IPv6 | `example.com → 2606:2800:220:1:...` |
| **CNAME** | Canonical Name | Alias para outro domínio | `www.example.com → example.com` |
| **MX** | Mail Exchange | Servidor de email | `example.com → mail.example.com (priority 10)` |
| **NS** | Name Server | DNS autoritativo | `example.com → ns1.example.com` |
| **TXT** | Text | Metadados (SPF, DKIM, verificação) | `"v=spf1 include:_spf.google.com"` |
| **SRV** | Service | Localiza serviços (host + porta) | `_sip._tcp.example.com → 5060 sip.example.com` |
| **PTR** | Pointer | Reverse DNS (IP → domínio) | `34.216.184.93 → example.com` |
| **SOA** | Start of Authority | Metadados da zone | Serial, refresh, retry, expire, TTL |
| **CAA** | Cert Authority Auth | Quais CAs podem emitir certificados | `example.com CAA 0 issue "letsencrypt.org"` |

### Exemplo de Zone File

```dns
$TTL 3600
@       IN  SOA   ns1.example.com. admin.example.com. (
                   2024012501  ; Serial
                   3600        ; Refresh
                   900         ; Retry
                   604800      ; Expire
                   86400 )     ; Minimum TTL

; Name Servers
@       IN  NS    ns1.example.com.
@       IN  NS    ns2.example.com.

; A Records
@       IN  A     93.184.216.34
www     IN  A     93.184.216.34
api     IN  A     10.0.1.100
api     IN  A     10.0.1.101     ; Round Robin (2 IPs)

; CNAME
blog    IN  CNAME example.com.
store   IN  CNAME shop.shopify.com.

; MX
@       IN  MX    10 mail1.example.com.
@       IN  MX    20 mail2.example.com.

; TXT
@       IN  TXT   "v=spf1 include:_spf.google.com ~all"
```

---

## TTL (Time To Live)

O TTL define por quanto tempo um record pode ser cacheado.

| TTL | Valor | Uso |
|-----|-------|-----|
| **Muito curto** | 30-60s | Failover rápido, migração |
| **Curto** | 300s (5 min) | Serviços que mudam frequentemente |
| **Médio** | 3600s (1 hora) | Padrão razoável |
| **Longo** | 86400s (24h) | Records estáveis (MX, NS) |

### Estratégia de Migração

```
Situação: Migrar server de IP 10.0.1.100 para 10.0.2.200

Passo 1: Reduzir TTL (dias antes)
  api.example.com → 10.0.1.100 TTL=60

Passo 2: Atualizar record
  api.example.com → 10.0.2.200 TTL=60

Passo 3: Aguardar propagação (TTL antigo expira)
  Máximo 60s até todos verem novo IP

Passo 4: Aumentar TTL novamente
  api.example.com → 10.0.2.200 TTL=3600
```

---

## DNS para System Design

### 1. DNS Round Robin (Load Balancing simples)

```dns
api.example.com    A    10.0.1.100
api.example.com    A    10.0.1.101
api.example.com    A    10.0.1.102
```

- Cada resolução retorna IPs em ordem diferente
- **Prós:** Simples, zero custo
- **Contras:** Sem health check, sem weighted routing, cache afeta distribuição

### 2. GeoDNS (Geographic Routing)

Retorna IPs diferentes baseado na **localização do usuário**:

```
Usuário no Brasil  → api.example.com → 10.0.1.100 (São Paulo)
Usuário nos EUA    → api.example.com → 10.0.2.100 (Virginia)
Usuário na Europa  → api.example.com → 10.0.3.100 (Frankfurt)
```

```
                      ┌─────────┐
                      │ GeoDNS  │
                      └────┬────┘
                  ┌────────┼────────┐
                  ▼        ▼        ▼
            ┌─────────┐ ┌──────┐ ┌──────────┐
            │São Paulo│ │ USA  │ │Frankfurt │
            │ DC      │ │ DC   │ │  DC      │
            └─────────┘ └──────┘ └──────────┘
```

**Usado por:** Netflix, Google, Amazon — direciona para datacenter mais próximo.

### 3. Weighted DNS

Distribui tráfego por peso:

```
api.example.com → 10.0.1.100 (weight=70)   ← 70% do tráfego
api.example.com → 10.0.2.100 (weight=30)   ← 30% do tráfego
```

**Usado para:** Canary deployments, migração gradual.

### 4. DNS Failover

```
Primary:    api.example.com → 10.0.1.100 (health check: HTTP 200)
Secondary:  api.example.com → 10.0.2.100 (ativado se primary falhar)
```

```
                    ┌──────────┐
                    │   DNS    │
                    │ +Health  │
                    │  Check   │
                    └────┬─────┘
                         │
               ┌─────────┼─────────┐
               ▼                    ▼
         ┌──────────┐        ┌──────────┐
         │ Primary  │        │Secondary │
         │ (active) │        │(standby) │
         └──────────┘        └──────────┘
              ✓                   ✗
         serving traffic    waiting...

         Se Primary falha:
              ✗                   ✓
         down              serving traffic
```

### 5. Latency-Based Routing

Mede latência real dos PoPs e retorna o mais rápido:

```
Usuário em Recife:
  - São Paulo: 25ms ← retorna este
  - Virginia: 140ms
  - Frankfurt: 180ms
```

---

## DNS em Serviços Cloud

### AWS Route 53

```
Routing Policies:
- Simple: Round Robin
- Weighted: Tráfego proporcional por peso
- Latency: Menor latência medida
- Failover: Active-passive com health checks
- Geolocation: Por país/continente do usuário
- Geoproximity: Por proximidade geográfica com bias
- Multi-value: Retorna múltiplos IPs com health check
```

### Cloudflare DNS
- Anycast DNS (ultra-rápido, ~11ms global)
- Proxy mode: DDoS protection automático
- DNSSEC built-in

### Google Cloud DNS
- Anycast
- Integrado com GCP Load Balancer
- 100% SLA

---

## DNS Interno (Service Discovery)

Em microservices, DNS é usado para **descobrir serviços**:

```
# Kubernetes DNS (CoreDNS)
my-service.my-namespace.svc.cluster.local → 10.96.0.15

# Consul DNS
api.service.consul → 10.0.1.100, 10.0.1.101

# AWS Cloud Map
api.my-namespace → 10.0.1.100
```

```
┌──────────────────────────────────────────────┐
│              Kubernetes Cluster               │
│                                               │
│  ┌─────────┐  DNS query   ┌──────────┐      │
│  │ Service │ ────────────▶ │ CoreDNS  │      │
│  │   A     │               │          │      │
│  └─────────┘  ◀─────────── └──────────┘      │
│               10.96.0.15                      │
│                    │                          │
│              ┌─────▼─────┐                    │
│              │ Service B │                    │
│              │ Pod IPs   │                    │
│              └───────────┘                    │
└──────────────────────────────────────────────┘
```

---

## Segurança DNS

| Ameaça | Descrição | Solução |
|--------|-----------|---------|
| **DNS Spoofing/Poisoning** | Injeta records falsos no cache | DNSSEC |
| **DNS Hijacking** | Redireciona DNS para server malicioso | DNSSEC + DNS-over-HTTPS |
| **DDoS contra DNS** | Ataque volumétrico ao DNS server | Anycast, over-provisioning |
| **DNS Tunneling** | Exfiltra dados via queries DNS | Monitoring, rate limiting |

### DNSSEC

```
Cadeia de confiança criptográfica:

Root (.) assina → .com
.com assina     → example.com
example.com     → records individuais

Cada resposta DNS vem com assinatura digital verificável.
```

### DNS over HTTPS (DoH) / DNS over TLS (DoT)

```
Tradicional: DNS query em texto plano (porta 53) → ISP pode ver/interceptar
DoH:         DNS query encriptada via HTTPS (porta 443) → privado
DoT:         DNS query encriptada via TLS (porta 853) → privado
```

---

## Uso em Big Techs

### Google — Google Public DNS (8.8.8.8)
- Maior resolver público do mundo
- Anycast global
- Suporta DNSSEC, DoH, DoT
- Processa **trilhões** de queries por dia

### Cloudflare — 1.1.1.1
- Resolver público mais rápido (~11ms avg)
- Privacy-first (não vende dados DNS)
- Anycast em 300+ cidades

### Amazon — Route 53
- DNS autoritativo + resolver
- Integrado com AWS ecosystem
- Health checks + failover automático
- 100% SLA (único serviço AWS com esse SLA)

### Netflix — DNS para Multi-Region
- Route 53 com latency-based routing
- Failover automático entre regiões AWS
- Se us-east-1 falha → tráfego vai para eu-west-1

---

## Perguntas Comuns em Entrevistas

1. **O que acontece quando você digita google.com no browser?** → DNS recursive resolution (Root → TLD → Authoritative), TCP handshake, TLS, HTTP request
2. **Como usar DNS para alta disponibilidade?** → DNS failover com health checks, multi-region com latency-based routing
3. **DNS Round Robin vs Load Balancer?** → DNS é simples mas sem health check; LB é mais inteligente
4. **TTL baixo vs alto?** → Baixo = failover rápido mas mais queries; Alto = menos queries mas propagação lenta
5. **Como funciona GeoDNS?** → Resolve IP do recursive resolver → deduz localização → retorna IP do datacenter mais próximo

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **TTL** | Curto (failover rápido) | Longo (menos queries, mais cache) |
| **DNS LB vs Real LB** | DNS Round Robin (simples, grátis) | L4/L7 LB (health check, sticky) |
| **Single DNS vs Multi** | Um provider | Multi-DNS (redundância) |
| **Public DNS** | Google 8.8.8.8 (reliable) | Cloudflare 1.1.1.1 (mais rápido) |
| **GeoDNS vs Anycast** | GeoDNS (controle fino) | Anycast (automático por rede) |

---

## Referências

- [Cloudflare Learning - What is DNS?](https://www.cloudflare.com/learning/dns/what-is-dns/)
- [AWS Route 53 Documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/)
- [IETF RFC 1035 - DNS](https://tools.ietf.org/html/rfc1035)
- [Google Public DNS](https://developers.google.com/speed/public-dns)
- System Design Interview — Alex Xu, Vol 1
