# 5. Reverse Proxy

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial — presente em toda arquitetura de produção  
> **Complexidade:** Média

---

## Definição

**Reverse Proxy** é um servidor intermediário que fica na frente dos backend servers, recebendo requests dos clients e encaminhando-as para o servidor adequado. O client nunca se comunica diretamente com os backend servers.

> Diferente de um **forward proxy** (que fica na frente dos clients), o reverse proxy fica na frente dos **servers**.

---

## Forward Proxy vs Reverse Proxy

```
Forward Proxy:                         Reverse Proxy:
  Clients → [Proxy] → Internet          Internet → [Proxy] → Servers

┌────────┐                             ┌────────┐
│Client A│──┐                          │Client A│──┐
└────────┘  │  ┌─────────┐  Internet   └────────┘  │           ┌──────────┐
┌────────┐  ├─▶│ Forward │──────────▶  ┌────────┐  │  ┌──────┐ │ Server 1 │
│Client B│──┤  │ Proxy   │            │Client B│──┼─▶│Reverse├▶│ Server 2 │
└────────┘  │  └─────────┘            └────────┘  │  │Proxy  │ │ Server 3 │
┌────────┐  │                          ┌────────┐  │  └──────┘ └──────────┘
│Client C│──┘                          │Client C│──┘
└────────┘                             └────────┘

Protege os CLIENTS                     Protege os SERVERS
```

| Aspecto | Forward Proxy | Reverse Proxy |
|---------|--------------|---------------|
| **Posição** | Frente dos clients | Frente dos servers |
| **Quem protege** | Clients (anonimato) | Servers (segurança) |
| **Quem sabe** | Server não sabe quem é o client real | Client não sabe qual server real atende |
| **Uso** | VPN, censura bypass, cache corporativo | Load balancing, SSL, WAF, cache |

---

## Funções do Reverse Proxy

### Arquitetura Geral

```
                              ┌─────────────────────────────┐
                              │       Reverse Proxy         │
┌────────┐                    │                             │
│ Client │──── HTTPS ────────▶│  ┌────────────────────┐    │
└────────┘                    │  │ SSL Termination    │    │
                              │  │ Rate Limiting      │    │
                              │  │ Compression        │    │         ┌───────────┐
                              │  │ Load Balancing     │────HTTP──▶ │ Backend 1 │
                              │  │ Caching            │    │         ├───────────┤
                              │  │ Request Routing    │────HTTP──▶ │ Backend 2 │
                              │  │ WAF (Security)     │    │         ├───────────┤
                              │  │ Logging/Metrics    │────HTTP──▶ │ Backend 3 │
                              │  └────────────────────┘    │         └───────────┘
                              └─────────────────────────────┘
```

### 1. SSL/TLS Termination

O proxy decifra HTTPS e encaminha HTTP puro para o backend:

```
Client ──── HTTPS (TLS 1.3) ────▶ Reverse Proxy ──── HTTP ────▶ Backend
                                        │
                                  Certificado SSL
                                  gerenciado aqui
```

**Benefícios:**
- Backend não precisa gerenciar certificados
- Offload de criptografia (CPU-intensive)
- Centraliza renovação de certificados (Let's Encrypt)

### 2. Load Balancing

Distribui requests entre múltiplos backends:

```
Request 1 ──▶ [Proxy] ──▶ Backend A
Request 2 ──▶ [Proxy] ──▶ Backend B
Request 3 ──▶ [Proxy] ──▶ Backend C
Request 4 ──▶ [Proxy] ──▶ Backend A  (Round Robin)
```

### 3. Caching

```
Request ──▶ [Proxy Cache] ──HIT──▶ Response (sem ir ao backend)
                          ──MISS──▶ Backend ──▶ Cache + Response
```

### 4. Compression

```
Backend responde JSON (100KB)
  ──▶ Proxy aplica gzip/brotli
  ──▶ Client recebe (15KB)
```

### 5. Rate Limiting

```
Client envia 1000 req/s
  ──▶ Proxy permite 100 req/s
  ──▶ 900 requests recebem 429 Too Many Requests
  ──▶ Backend recebe apenas 100 req/s seguramente
```

### 6. Request Routing

```
/api/*        ──▶ API Server (port 8080)
/static/*     ──▶ File Server (port 8081)
/ws/*         ──▶ WebSocket Server (port 8082)
/admin/*      ──▶ Admin Server (port 8083) + IP whitelist
```

### 7. Security (WAF)

```
Request com SQL injection
  ──▶ Proxy detecta padrão malicioso
  ──▶ Bloqueia (403 Forbidden)
  ──▶ Backend nunca vê a request
```

### 8. Observability

```
Toda request passa pelo proxy:
  ──▶ Access logs centralizados
  ──▶ Métricas (latência, status codes, throughput)
  ──▶ Request ID injection para tracing
  ──▶ Headers de auditoria (X-Request-Id, X-Forwarded-For)
```

---

## Tecnologias

### NGINX

O reverse proxy mais popular do mundo.

```nginx
# /etc/nginx/conf.d/app.conf

upstream backend {
    least_conn;
    server 10.0.1.100:8080 weight=5;
    server 10.0.1.101:8080 weight=3;
    server 10.0.1.102:8080 weight=2;
    
    keepalive 32;  # Connection pooling
}

server {
    listen 443 ssl http2;
    server_name api.example.com;
    
    # SSL Termination
    ssl_certificate     /etc/ssl/certs/api.example.com.pem;
    ssl_certificate_key /etc/ssl/private/api.example.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    
    # Compression
    gzip on;
    gzip_types application/json text/plain text/css;
    gzip_min_length 1024;
    
    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;
    
    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Strict-Transport-Security "max-age=31536000" always;
    
    # API Proxy
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-ID $request_id;
        
        # Timeouts
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
        proxy_send_timeout 10s;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    # Static files (servidos diretamente pelo NGINX)
    location /static/ {
        root /var/www;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    
    # WebSocket
    location /ws/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    
    # Health Check endpoint
    location /health {
        access_log off;
        return 200 'OK';
    }
}
```

### Envoy Proxy

Service mesh proxy de alto desempenho criado pelo Lyft (agora CNCF).

```yaml
# envoy.yaml
static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 8080
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/api"
                          route:
                            cluster: api_cluster
                        - match:
                            prefix: "/static"
                          route:
                            cluster: static_cluster
                http_filters:
                  - name: envoy.filters.http.router

  clusters:
    - name: api_cluster
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: api_cluster
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: api-service
                      port_value: 8080
      health_checks:
        - timeout: 5s
          interval: 10s
          unhealthy_threshold: 3
          healthy_threshold: 2
          http_health_check:
            path: /health
```

**Diferenciais do Envoy:**
- gRPC nativo
- Circuit breaking built-in
- Distributed tracing (Zipkin, Jaeger)
- Hot restart (zero downtime config reload)
- xDS API (configuração dinâmica via control plane)

### HAProxy

```
                  Performance-focused
                  Muito usado em bare-metal / VM
                  Suporta L4 (TCP) e L7 (HTTP)
```

```haproxy
# /etc/haproxy/haproxy.cfg

global
    maxconn 50000
    log stdout format raw local0

defaults
    mode http
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    
frontend http_front
    bind *:443 ssl crt /etc/ssl/server.pem
    
    # ACL-based routing
    acl is_api path_beg /api
    acl is_static path_beg /static
    
    use_backend api_servers if is_api
    use_backend static_servers if is_static
    default_backend api_servers

backend api_servers
    balance leastconn
    option httpchk GET /health
    
    server api1 10.0.1.100:8080 check weight 5
    server api2 10.0.1.101:8080 check weight 3
    server api3 10.0.1.102:8080 check weight 2

backend static_servers
    balance roundrobin
    server static1 10.0.2.100:8081 check
```

### Comparativo

| Feature | NGINX | Envoy | HAProxy | Traefik |
|---------|-------|-------|---------|---------|
| **Tipo** | Web server + proxy | Service proxy | LB + proxy | Cloud-native proxy |
| **Config** | Static files | xDS API (dinâmico) | Static files | Auto-discovery |
| **Protocol** | HTTP/1.1, HTTP/2 | HTTP/1.1, HTTP/2, gRPC | HTTP, TCP | HTTP, TCP, gRPC |
| **Service Mesh** | Limitado | Sidecar do Istio | Não | Limitado |
| **Performance** | Excelente | Excelente | Excelente (L4) | Bom |
| **Config Reload** | `nginx -s reload` | Hot restart | Reload | Auto |
| **Observability** | Access logs | Métricas ricas, tracing | Stats page | Dashboard |
| **Caso de uso** | Web server, reverse proxy | Service mesh, microservices | High-performance LB | Kubernetes ingress |

---

## Padrões de Deploy

### 1. Proxy Único (Simples)

```
Internet ──▶ [NGINX] ──▶ App Servers
```

### 2. Proxy + CDN

```
Internet ──▶ [CDN] ──▶ [NGINX] ──▶ App Servers
```

### 3. Sidecar Proxy (Service Mesh)

```
┌──────────────────────────┐
│ Pod                      │
│ ┌──────────┐ ┌─────────┐│
│ │   App    │ │  Envoy  ││   Cada pod tem seu proxy
│ │ Container│◀▶│ Sidecar ││   mTLS entre todos os proxies
│ └──────────┘ └─────────┘│
└──────────────────────────┘
```

### 4. Multi-Layer

```
Internet ──▶ [CDN] ──▶ [L7 LB (NGINX)] ──▶ [API Gateway] ──▶ [Sidecar Proxies] ──▶ Services
```

---

## Uso em Big Techs

### Google — GFE (Google Front End)
- Todo request para Google passa pelo GFE
- Reverse proxy global
- DDoS protection, SSL termination, routing
- Serve Search, YouTube, Gmail, etc.

### Uber — Envoy
- Migrou de HAProxy para Envoy
- Sidecar proxy em todos os microservices
- Service mesh com mutual TLS
- Observability integrada (metrics, tracing)

### Netflix — Zuul → Zuul 2
- Zuul: Reverse proxy/gateway Java (Spring ecosystem)
- Zuul 2: Async, non-blocking (Netty-based)
- Funções: routing, canary deployments, load shedding
- Processa bilhões de requests por dia

### Cloudflare — NGINX customizado
- Core da infraestrutura baseada em NGINX extensivamente customizado
- Processa ~30M requests por segundo
- Recentemente migrando partes para Rust (Pingora)

### Meta — Proxygen
- HTTP proxy framework próprio em C++
- Suporta HTTP/1.1, HTTP/2, HTTP/3 (QUIC)
- Extremamente otimizado para o scale do Facebook

---

## Configuração de Segurança

### Headers de Segurança (NGINX)

```nginx
# Previne clickjacking
add_header X-Frame-Options "SAMEORIGIN" always;

# Previne MIME sniffing
add_header X-Content-Type-Options "nosniff" always;

# XSS Protection
add_header X-XSS-Protection "1; mode=block" always;

# HSTS (força HTTPS)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# Content Security Policy
add_header Content-Security-Policy "default-src 'self'; script-src 'self'" always;

# Referrer Policy
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Esconde versão do server
server_tokens off;

# Limita tamanho do body (previne abuse)
client_max_body_size 10m;
```

---

## Perguntas Comuns em Entrevistas

1. **Reverse proxy vs Load Balancer?** → LB é uma função do reverse proxy. Reverse proxy faz LB + SSL + cache + routing + security
2. **Reverse proxy vs API Gateway?** → API Gateway adiciona auth, rate limiting, protocol transformation. Reverse proxy é mais genérico
3. **NGINX vs Envoy?** → NGINX para web serving + proxy; Envoy para service mesh + microservices
4. **Por que não acessar o backend diretamente?** → Segurança (oculta IPs), SSL termination, caching, observability centralizada
5. **O que é sidecar proxy?** → Proxy co-located com cada service instance (Envoy no Istio/Kubernetes)

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **NGINX vs Envoy** | NGINX (maduro, simples) | Envoy (service mesh, gRPC) |
| **Single proxy vs Mesh** | Um proxy centralizado | Sidecar por service |
| **SSL at proxy vs end-to-end** | Terminar no proxy (performance) | mTLS end-to-end (segurança) |
| **Static config vs Dynamic** | NGINX (arquivo) | Envoy xDS (API) |
| **Managed vs Self-hosted** | ALB/CloudFront | NGINX/Envoy self-hosted |

---

## Referências

- [NGINX Documentation](https://nginx.org/en/docs/)
- [Envoy Proxy Documentation](https://www.envoyproxy.io/docs/)
- [HAProxy Documentation](https://www.haproxy.org/#docs)
- [Cloudflare - What is a Reverse Proxy?](https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/)
- [Netflix Zuul](https://github.com/Netflix/zuul)
