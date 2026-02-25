# 1. Load Balancing

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial para qualquer entrevista de System Design  
> **Complexidade:** Média

---

## Definição

**Load Balancing** é o processo de distribuir tráfego de rede entre múltiplos servidores (ou instâncias) para garantir que nenhum servidor fique sobrecarregado, aumentando a **disponibilidade**, **confiabilidade** e **performance** do sistema.

O Load Balancer atua como um "policial de trânsito" — recebe todas as requests dos clients e decide para qual backend enviá-las.

---

## Por Que é Importante?

- **Elimina single point of failure** — se um server cai, o LB redireciona para outros
- **Escalabilidade horizontal** — adicione mais servers conforme a demanda cresce
- **Melhora latência** — distribui carga uniformemente, evitando gargalos
- **Manutenção sem downtime** — drene um server, faça deploy, volte ao pool

---

## Diagrama de Arquitetura

```
                     ┌──────────────┐
                     │   Internet   │
                     │   (Clients)  │
                     └──────┬───────┘
                            │
                     ┌──────▼───────┐
                     │     DNS      │
                     │ (Route 53)   │
                     └──────┬───────┘
                            │
                ┌───────────▼───────────┐
                │     Load Balancer     │
                │  (NGINX / ALB / NLB)  │
                │                       │
                │  Health Checks: ✓     │
                │  Algorithm: Round     │
                │  Robin / Least Conn   │
                └───┬───────┬───────┬───┘
                    │       │       │
             ┌──────▼─┐ ┌──▼────┐ ┌▼──────┐
             │Server 1│ │Server2│ │Server3│
             │ (app)  │ │ (app) │ │ (app) │
             │ :8080  │ │ :8080 │ │ :8080 │
             └────────┘ └───────┘ └───────┘
```

---

## Camadas de Load Balancing

### L4 — Transport Layer (TCP/UDP)

Opera na camada 4 do modelo OSI. Decisões baseadas em **IP de origem/destino** e **porta**.

| Aspecto | Detalhe |
|---------|---------|
| **Protocolo** | TCP, UDP |
| **Informação** | IP + Porta |
| **Performance** | Ultra-rápido (não inspeciona payload) |
| **Funcionalidades** | Limitadas (não vê HTTP headers) |
| **Caso de Uso** | Alta performance, gaming, streaming |
| **Exemplo** | AWS NLB, Maglev (Google), Katran (Meta) |

```
Client 192.168.1.5:54321
       │
       ▼
  L4 Load Balancer  ──▶  Backend 10.0.0.1:8080  (based on src IP hash)
  (TCP connection forwarded)
```

### L7 — Application Layer (HTTP/HTTPS)

Opera na camada 7. Pode inspecionar **conteúdo HTTP**: URL, headers, cookies, body.

| Aspecto | Detalhe |
|---------|---------|
| **Protocolo** | HTTP, HTTPS, WebSocket, gRPC |
| **Informação** | URL path, headers, cookies, body |
| **Performance** | Mais lento que L4 (inspeção de conteúdo) |
| **Funcionalidades** | Rich: routing, rewriting, A/B testing |
| **Caso de Uso** | Web apps, APIs, microservices |
| **Exemplo** | AWS ALB, NGINX, Envoy, HAProxy |

```
GET /api/users HTTP/1.1
Host: myapp.com
Cookie: session=abc123
       │
       ▼
  L7 Load Balancer
       │
       ├── /api/users   → Users Service Cluster
       ├── /api/orders  → Orders Service Cluster
       └── /static/*    → CDN / Static Server
```

### Comparativo L4 vs L7

| Critério | L4 | L7 |
|----------|----|----|
| Velocidade | Mais rápido | Mais lento |
| Flexibilidade | Baixa | Alta |
| SSL Termination | Não | Sim |
| Content-based routing | Não | Sim |
| WebSocket support | Pass-through | Nativo |
| Custo | Menor | Maior |

---

## Algoritmos de Load Balancing

### 1. Round Robin

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  (volta ao início)
```

- **Prós:** Simples, previsível
- **Contras:** Não considera carga atual ou capacidade do server
- **Quando usar:** Servidores homogêneos com requests de duração similar

### 2. Weighted Round Robin

```
Weights: A=5, B=3, C=2 (total=10)

Requests 1-5  → Server A
Requests 6-8  → Server B
Requests 9-10 → Server C
```

- **Prós:** Respeita capacidade diferente dos servidores
- **Contras:** Não reage a carga dinâmica
- **Quando usar:** Servidores com capacidades diferentes (ex: m5.xlarge + m5.2xlarge)

### 3. Least Connections

```
Server A: 15 active connections ←
Server B: 42 active connections
Server C: 23 active connections

Próximo request → Server A (menos conexões)
```

- **Prós:** Adapta-se à carga real, bom para requests de duração variável
- **Contras:** Overhead de tracking, pode não considerar server capacity
- **Quando usar:** Requests com duração variável (ex: websockets, uploads)

### 4. Least Response Time

- Combina **least connections** com **menor tempo de resposta**
- Envia para o server que responde mais rápido E tem menos conexões
- **Quando usar:** Quando latência é prioridade

### 5. IP Hash

```
hash(client_ip) % num_servers = server_index

Client 192.168.1.5 → always goes to Server B
Client 192.168.1.9 → always goes to Server A
```

- **Prós:** Garante que mesmo client vai para mesmo server (sticky sessions)
- **Contras:** Distribuição pode ser desigual; server failure requer rehash
- **Quando usar:** Sticky sessions, cache local por user

### 6. Consistent Hashing

- Usa um ring hash para distribuição
- Adicionar/remover server redistribui apenas ~K/N keys
- **Quando usar:** Cache distribuído, CDN, sharding

### 7. Random

- Escolhe um server aleatoriamente
- Surpreendentemente eficaz com muitos servers (lei dos grandes números)
- **Variante:** "Power of Two Choices" — escolhe 2 aleatórios, envia para o com menos carga

### 8. Resource-Based (Adaptive)

- Server reporta métricas (CPU, memória, connections)
- LB usa essas métricas para decisão
- **Quando usar:** Workloads heterogêneos, servers com recursos limitados

---

## Padrões de Deployment

### Single Load Balancer

```
Clients → [LB] → Servers
```
- Simples, mas **single point of failure**

### Active-Passive (HA Pair)

```
             ┌───── [LB Active] ─────┐
             │                       │
Clients ─────┤   (heartbeat/VRRP)    ├───▶ Servers
             │                       │
             └───── [LB Passive] ────┘
                    (standby)
```
- LB Passive assume via **Virtual IP (VIP)** se Active falha
- Protocolo: VRRP (Virtual Router Redundancy Protocol)

### Active-Active

```
             ┌───── [LB 1] ─────┐
DNS RR ──────┤                   ├───▶ Servers
             └───── [LB 2] ─────┘
```
- Ambos processam tráfego
- DNS retorna ambos os IPs
- Melhor throughput, mas complex de manter state sync

### Global Server Load Balancing (GSLB)

```
🇧🇷 User (São Paulo)               🇺🇸 User (Virginia)
       │                                  │
       ▼                                  ▼
  [GeoDNS / Route 53]               [GeoDNS / Route 53]
       │                                  │
       ▼                                  ▼
  [LB São Paulo]                    [LB Virginia]
       │                                  │
       ▼                                  ▼
  [Servers SP]                      [Servers VA]
```
- Distribui tráfego entre data centers/regiões
- Baseado em geolocalização, latência ou health

---

## Health Checks

O LB precisa saber quais backends estão saudáveis:

```
Load Balancer
    │
    ├── GET /health → Server A → 200 OK ✓ (healthy)
    ├── GET /health → Server B → 200 OK ✓ (healthy)
    └── GET /health → Server C → timeout ✗ (unhealthy → remove do pool)
```

**Tipos:**
- **Active Health Checks:** LB faz requests periódicos ao backend
- **Passive Health Checks:** LB observa respostas reais (erros = unhealthy)

**Configuração típica (NGINX):**
```nginx
upstream backend {
    server 10.0.0.1:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.3:8080 max_fails=3 fail_timeout=30s;
}
```

**Configuração AWS ALB:**
```json
{
  "HealthCheckProtocol": "HTTP",
  "HealthCheckPath": "/health",
  "HealthCheckIntervalSeconds": 10,
  "HealthyThresholdCount": 3,
  "UnhealthyThresholdCount": 2,
  "HealthCheckTimeoutSeconds": 5
}
```

---

## Session Persistence (Sticky Sessions)

Quando o estado é mantido no server (session no memória), o client precisa ir sempre ao mesmo server.

**Métodos:**
1. **Cookie-based:** LB insere cookie com server ID
2. **IP Hash:** Mesmo IP → mesmo server
3. **Application Cookie:** App define cookie que o LB respeita

**Problemas:**
- Impede scaling/draining uniforme
- Server failure perde sessions de todos os users nele
- **Melhor solução:** Externalizar sessions (Redis/Memcached) e usar stateless servers

---

## Tecnologias Populares

| Tecnologia | Tipo | Camada | Destaques |
|------------|------|--------|-----------|
| **NGINX** | Software | L7 (+ L4) | Extremamente popular, reverse proxy + LB |
| **HAProxy** | Software | L4 + L7 | Alta performance, usado em produção massiva |
| **Envoy** | Software | L7 | Service mesh, gRPC nativo, observability |
| **Traefik** | Software | L7 | Auto-discovery (Docker/K8s), Let's Encrypt |
| **AWS ALB** | Cloud | L7 | Managed, integrado com ECS/EKS |
| **AWS NLB** | Cloud | L4 | Ultra-low latency, static IP |
| **GCP Cloud LB** | Cloud | L4 + L7 | Global, Anycast IP |
| **Azure Load Balancer** | Cloud | L4 | Regional, HA zones |
| **F5 BIG-IP** | Hardware | L4 + L7 | Enterprise, WAF, SSL offload |

---

## Uso em Big Techs

### Google — Maglev
- Load balancer L4 customizado
- Consistent hashing para distribuição de conexões
- Processa milhões de requests/sec por máquina
- Paper: "Maglev: A Fast and Reliable Software Network Load Balancer" (2016)

### Meta — Katran
- L4 load balancer baseado em **XDP/eBPF** (kernel bypass)
- Open-source
- Processa pacotes no kernel space sem copiar para user space
- Performance extremamente alta

### Netflix — Zuul / Spring Cloud Gateway
- L7 edge proxy + gateway
- Funções: authentication, routing, canary, metrics
- Migração: Zuul 1 (Servlet) → Zuul 2 (Async/Netty) → Spring Cloud Gateway

### Uber — Envoy
- Service mesh com Envoy como sidecar proxy
- gRPC load balancing nativo
- Circuit breaking + retry integrados

---

## Perguntas Comuns em Entrevistas

1. **Qual a diferença entre L4 e L7 load balancer?**
2. **Como garantir HA do próprio load balancer?** (Active-Passive com VIP)
3. **Como lidar com sticky sessions sem acoplar ao server?** (External session store)
4. **Qual algoritmo escolher para requests de duração variável?** (Least Connections)
5. **Como fazer load balancing global entre data centers?** (GSLB / GeoDNS)
6. **O que acontece se um backend fica lento mas não falha?** (Passive health checks, circuit breaker)

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Camada** | L4 (performance) | L7 (flexibilidade) |
| **HA Model** | Active-Passive (simples) | Active-Active (throughput) |
| **Sessions** | Sticky (simples) | Stateless + Redis (escalável) |
| **Managed vs Self** | AWS ALB (zero-ops) | NGINX/HAProxy (controle total) |
| **Algoritmo** | Round Robin (simples) | Least Connections (adaptativo) |

---

## Referências

- [NGINX Load Balancing Documentation](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
- [HAProxy Documentation](https://www.haproxy.com/documentation/)
- [Google Maglev Paper](https://research.google/pubs/pub44824/)
- [Meta Katran](https://github.com/facebookincubator/katran)
- [AWS Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/)
- System Design Interview — Alex Xu, Chapter: Design a Rate Limiter (LB context)
