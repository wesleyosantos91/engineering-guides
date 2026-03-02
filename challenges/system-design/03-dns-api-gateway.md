# Level 3 — DNS & API Gateway

> **Objetivo:** Implementar um DNS resolver simplificado e um API Gateway completo com
> roteamento, autenticação, rate limiting e transformação de requests.

**Referência:**
- [04-dns.md](../../.docs/SYSTEM-DESIGN/04-dns.md)
- [06-api-gateway.md](../../.docs/SYSTEM-DESIGN/06-api-gateway.md)

**Pré-requisito:** Levels 0-2 completos.

---

## Contexto

O **DNS** é a "agenda telefônica da internet" — traduz nomes de domínio para endereços IP. Um **API Gateway** é o ponto de entrada unificado para uma arquitetura de microserviços, gerenciando roteamento, autenticação, rate limiting, circuit breaking e observabilidade.

---

## Parte 1 — ADR: Design do API Gateway

**Arquivo:** `docs/adrs/ADR-001-api-gateway-design.md`

**Decisão:** Arquitetura e responsabilidades do API Gateway.

**Options:**
1. **Single API Gateway centralizado** — um gateway para todos os serviços
2. **BFF (Backend for Frontend)** — um gateway por tipo de cliente (web, mobile, IoT)
3. **Gateway per-domain** — um gateway por bounded context
4. **Service Mesh (sidecar)** — proxy sidecar em cada serviço (Envoy/Istio)

**Decision Drivers:**
- Complexidade operacional
- Latência adicionada (hop extra)
- Flexibilidade de roteamento
- Team autonomy
- Single point of failure

**Critérios de aceite:**
- [ ] 4 opções documentadas
- [ ] Comparativo: Gateway centralizado vs BFF vs Service Mesh
- [ ] Responsabilidades do gateway definidas (o que é e o que não é do gateway)
- [ ] Estratégia de versionamento de API documentada

---

## Parte 2 — Diagrama DrawIO

**Arquivo:** `docs/diagrams/03-dns-api-gateway-architecture.drawio`

**View 1 — DNS Resolution Flow:**
```
┌────────┐    ┌───────────┐    ┌────────────┐    ┌──────────────┐
│Browser │───▶│ Recursive │───▶│   Root     │───▶│   TLD        │
│        │    │ Resolver  │    │ Nameserver │    │ Nameserver   │
└────────┘    └───────────┘    └────────────┘    │ (.com, .io)  │
                    │                             └──────┬───────┘
                    │                                    │
                    │          ┌────────────────┐ ◀──────┘
                    └─────────▶│ Authoritative  │
                               │  Nameserver    │
                               └────────────────┘
```

**View 2 — API Gateway Architecture:**
```
┌──────────┐  ┌──────────┐  ┌──────────┐
│   Web    │  │  Mobile  │  │   IoT    │
│  Client  │  │  Client  │  │  Device  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │              │
     └─────────────┼──────────────┘
                   │
           ┌───────▼────────┐
           │   API Gateway  │
           │  ┌───────────┐ │
           │  │Auth/JWT   │ │
           │  │Rate Limit │ │
           │  │Routing    │ │
           │  │Transform  │ │
           │  │Circuit Brk│ │
           │  │Logging    │ │
           │  └───────────┘ │
           └───┬───┬───┬───┘
               │   │   │
         ┌─────▼┐ ┌▼──┐ ┌▼─────┐
         │User  │ │Ord│ │Paymt │
         │Svc   │ │Svc│ │Svc   │
         └──────┘ └───┘ └──────┘
```

**View 3 — Request Lifecycle:** Fluxo completo de uma request através do gateway

**Critérios de aceite:**
- [ ] 3 views distintas
- [ ] DNS com todos os níveis de resolução
- [ ] Gateway com pipeline de middleware claro
- [ ] Request lifecycle com timing em cada etapa

---

## Parte 3 — Implementação

### 3.1 — Go: DNS Resolver + API Gateway

**Estrutura:**
```
go/
├── cmd/
│   ├── dns/main.go                 ← DNS resolver server
│   └── gateway/main.go             ← API Gateway server
├── internal/
│   ├── dns/
│   │   ├── resolver.go             ← DNS resolver (recursive)
│   │   ├── cache.go                ← DNS cache com TTL
│   │   ├── records.go              ← A, AAAA, CNAME, MX records
│   │   └── server.go               ← UDP DNS server
│   ├── gateway/
│   │   ├── gateway.go              ← Core gateway router
│   │   ├── router.go               ← Route matching (path, method, header)
│   │   ├── middleware/
│   │   │   ├── auth.go             ← JWT validation
│   │   │   ├── ratelimit.go        ← Rate limiting per client/route
│   │   │   ├── circuitbreaker.go   ← Circuit breaker per upstream
│   │   │   ├── logging.go          ← Structured request logging
│   │   │   ├── cors.go             ← CORS handling
│   │   │   ├── transform.go        ← Request/response transformation
│   │   │   └── requestid.go        ← Request ID propagation
│   │   ├── upstream/
│   │   │   ├── pool.go             ← Upstream service pool
│   │   │   └── health.go           ← Upstream health monitoring
│   │   └── config/
│   │       └── routes.go           ← Route configuration (YAML)
│   └── shared/
│       └── middleware.go           ← Middleware chain builder
├── configs/
│   └── routes.yaml                 ← Route definitions
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **DNS Resolver** que resolve recursivamente (root → TLD → authoritative)
2. **DNS Cache** com TTL respeitando o TTL do record
3. **API Gateway** com route matching por path pattern, method, headers
4. **JWT Authentication** middleware com validação de claims
5. **Rate Limiting** per-client (sliding window) e per-route (token bucket)
6. **Circuit Breaker** per-upstream com states (closed, open, half-open)
7. **Request/Response Transformation** (header injection, body mapping)
8. **Route Configuration** via YAML com hot reload
9. **Health monitoring** de upstreams
10. **Request ID** propagation (X-Request-ID)

**Critérios de aceite Go:**
- [ ] DNS resolver funcional (resolve domínios reais via UDP)
- [ ] DNS cache com TTL e eviction
- [ ] Gateway roteando para 3+ serviços upstream
- [ ] JWT auth validando tokens RS256
- [ ] Rate limiting: sliding window (por IP) + token bucket (por rota)
- [ ] Circuit breaker com transição de estados automática
- [ ] Testes: ≥ 20 cenários (unit + integration)
- [ ] Hot reload de configuração de rotas

---

### 3.2 — Java: DNS Resolver + API Gateway

**Estrutura:**
```
java/
├── src/main/java/com/challenge/gateway/
│   ├── Application.java
│   ├── dns/
│   │   ├── DnsResolver.java
│   │   ├── DnsCache.java
│   │   └── DnsRecord.java              ← sealed interface
│   ├── gateway/
│   │   ├── GatewayRouter.java
│   │   ├── RouteDefinition.java         ← Record
│   │   └── RouteMatcher.java
│   ├── filter/
│   │   ├── AuthenticationFilter.java    ← JWT validation
│   │   ├── RateLimitFilter.java
│   │   ├── CircuitBreakerFilter.java
│   │   ├── LoggingFilter.java
│   │   ├── CorsFilter.java
│   │   └── TransformFilter.java
│   ├── upstream/
│   │   ├── UpstreamPool.java
│   │   └── HealthMonitor.java
│   └── config/
│       ├── GatewayProperties.java       ← @ConfigurationProperties
│       └── FilterChainConfig.java
├── src/test/java/com/challenge/gateway/
│   ├── GatewayRouterTest.java
│   ├── filter/
│   │   ├── AuthenticationFilterTest.java
│   │   └── RateLimitFilterTest.java
│   └── GatewayIntegrationTest.java
└── pom.xml
```

**Funcionalidades Java:**
1. **DNS Resolver** com `java.net.InetAddress` + cache custom
2. **Spring Cloud Gateway** integration (ou implementação manual)
3. **JWT Filter** com `spring-security-oauth2-jose`
4. **Rate Limiting** com Resilience4j ou implementação manual
5. **Circuit Breaker** com Resilience4j
6. **Sealed interfaces** para DNS record types
7. **Records** para route definitions e DTOs
8. **Virtual Threads** para I/O operations

**Critérios de aceite Java:**
- [ ] DNS resolver com cache
- [ ] Gateway roteando para 3+ serviços
- [ ] JWT auth com Spring Security
- [ ] Rate limiting por IP e por rota
- [ ] Circuit breaker com Resilience4j
- [ ] Testes com MockServer ou WireMock
- [ ] JaCoCo ≥ 80%

---

## Definição de Pronto (DoD)

- [ ] ADR documentando design do API Gateway
- [ ] DrawIO com 3 views (DNS, Gateway Architecture, Request Lifecycle)
- [ ] Go: DNS resolver + API Gateway com todos os middlewares + testes
- [ ] Java: DNS resolver + API Gateway com Spring + testes
- [ ] Demo: request passando por DNS → Gateway → Service → Response
- [ ] Commit: `feat(system-design-03): dns resolver and api gateway`
