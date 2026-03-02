# Level 1 — Load Balancing & Reverse Proxy

> **Objetivo:** Implementar um Load Balancer e um Reverse Proxy funcionais do zero,
> documentando decisões em ADR e diagramando a arquitetura no DrawIO.

**Referência:**
- [01-load-balancing.md](../../.docs/SYSTEM-DESIGN/01-load-balancing.md)
- [05-reverse-proxy.md](../../.docs/SYSTEM-DESIGN/05-reverse-proxy.md)

**Pré-requisito:** Level 0 completo.

---

## Contexto

O **Load Balancer** distribui tráfego entre múltiplos servidores backend, garantindo alta disponibilidade e distribuição uniforme de carga. O **Reverse Proxy** atua como intermediário entre clientes e servidores, adicionando segurança, caching e roteamento.

Esses dois componentes são a **primeira linha** de qualquer arquitetura escalável. Neste desafio, você vai implementá-los do zero para entender profundamente como funcionam.

---

## Parte 1 — ADR: Estratégia de Load Balancing

### Requisitos do ADR

**Arquivo:** `docs/adrs/ADR-001-load-balancing-algorithm-selection.md`

**Decisão a documentar:** Qual algoritmo de load balancing usar para distribuir requisições.

**Options a considerar:**
1. **Round Robin** — distribuição sequencial simples
2. **Weighted Round Robin** — peso proporcional à capacidade
3. **Least Connections** — envia para o server com menos conexões ativas
4. **IP Hash** — hash do IP do cliente para sticky sessions
5. **Random** — seleção aleatória

**Decision Drivers:**
- Distribuição uniforme de carga
- Suporte a servidores heterogêneos
- Complexidade de implementação
- Suporte a health checks
- Session affinity (quando necessário)

**Critérios de aceite:**
- [ ] ADR segue formato MADR
- [ ] Mínimo 4 opções documentadas com prós/contras
- [ ] Tabela comparativa com critérios quantificáveis
- [ ] Decision Outcome justificado
- [ ] Consequences (boas e ruins) documentadas

---

## Parte 2 — Diagrama DrawIO

### Requisitos do Diagrama

**Arquivo:** `docs/diagrams/01-load-balancer-architecture.drawio`

**View 1 — System Context:**
```
┌──────────┐
│ Clients  │
│(browsers,│
│  apps)   │
└────┬─────┘
     │ HTTPS
┌────▼─────┐
│  Reverse │─── SSL Termination
│  Proxy   │─── Rate Limiting
└────┬─────┘─── Request Logging
     │ HTTP
┌────▼──────────────────────────────┐
│         Load Balancer             │
│  ┌─────┐  ┌─────┐  ┌─────┐      │
│  │Algo │  │Health│  │Conn │      │
│  │     │  │Check │  │Pool │      │
│  └─────┘  └─────┘  └─────┘      │
└────┬────────┬────────┬───────────┘
     │        │        │
┌────▼──┐ ┌──▼───┐ ┌──▼───┐
│Srv 1  │ │Srv 2 │ │Srv 3 │
│:8081  │ │:8082 │ │:8083 │
└───────┘ └──────┘ └──────┘
```

**View 2 — Sequence Diagram:** Fluxo de uma request passando por Reverse Proxy → Load Balancer → Backend → Response

**View 3 — Health Check Flow:** Como o LB detecta servidores down e os remove do pool

**Critérios de aceite:**
- [ ] 3 views no DrawIO (System Context, Sequence, Health Check)
- [ ] Componentes com cores padronizadas
- [ ] Protocolos indicados nas setas (HTTPS, HTTP, TCP)
- [ ] Health check flow com detecção de falha

---

## Parte 3 — Implementação

### 3.1 — Go: Load Balancer + Reverse Proxy

Implemente um Load Balancer HTTP com suporte a múltiplos algoritmos.

**Estrutura:**
```
go/
├── cmd/
│   ├── lb/main.go              ← Load Balancer server
│   └── backend/main.go         ← Backend server de teste
├── internal/
│   ├── balancer/
│   │   ├── balancer.go         ← Interface LoadBalancer
│   │   ├── roundrobin.go       ← Round Robin algorithm
│   │   ├── weighted.go         ← Weighted Round Robin
│   │   ├── leastconn.go        ← Least Connections
│   │   └── iphash.go           ← IP Hash
│   ├── health/
│   │   ├── checker.go          ← Health check goroutine
│   │   └── checker_test.go
│   ├── proxy/
│   │   ├── reverse.go          ← Reverse proxy handler
│   │   └── middleware.go       ← Logging, rate limiting
│   └── config/
│       └── config.go           ← Configuration management
├── go.mod
├── Makefile
└── config.yaml                 ← Configuração dos backends
```

**Interface principal:**
```go
type LoadBalancer interface {
    // NextServer retorna o próximo servidor disponível
    NextServer() (*Server, error)
    // AddServer adiciona um servidor ao pool
    AddServer(server *Server)
    // RemoveServer remove um servidor do pool
    RemoveServer(addr string)
    // Servers retorna todos os servidores
    Servers() []*Server
}

type Server struct {
    Addr       string
    Weight     int
    Alive      bool
    ActiveConn int64
    mu         sync.RWMutex
}
```

**Funcionalidades obrigatórias:**
1. **Round Robin** com thread-safety (atomic counter)
2. **Weighted Round Robin** respeitando pesos configurados
3. **Least Connections** tracking ativo de conexões
4. **IP Hash** para session affinity
5. **Health Checks** periódicos (goroutine dedicada)
6. **Reverse Proxy** com `httputil.ReverseProxy`
7. **Graceful shutdown** com `context.Context`
8. **Middleware chain** (logging, rate limiting, request ID)
9. **Hot reload** de configuração (watch file changes)
10. **Métricas** (requests/s, latência, erros por backend)

**Critérios de aceite Go:**
- [ ] 4 algoritmos implementados e intercambiáveis
- [ ] Health check com goroutine periódica (configurable interval)
- [ ] Reverse proxy com SSL termination (self-signed cert para dev)
- [ ] Middleware: logging (structured, slog), rate limiting (token bucket)
- [ ] Table-driven tests para cada algoritmo (≥ 10 cenários)
- [ ] Benchmark tests para throughput (`go test -bench`)
- [ ] `Makefile` com: `build`, `test`, `lint`, `run`, `bench`
- [ ] Configuration via YAML ou env vars
- [ ] Graceful shutdown (SIGINT/SIGTERM)
- [ ] Métricas expostas em `/metrics` (formato Prometheus)

---

### 3.2 — Java: Load Balancer + Reverse Proxy

Implemente o mesmo Load Balancer usando Spring Boot.

**Estrutura:**
```
java/
├── src/main/java/com/challenge/lb/
│   ├── Application.java
│   ├── config/
│   │   └── LoadBalancerConfig.java
│   ├── balancer/
│   │   ├── LoadBalancer.java           ← Interface
│   │   ├── RoundRobinBalancer.java
│   │   ├── WeightedRoundRobinBalancer.java
│   │   ├── LeastConnectionsBalancer.java
│   │   └── IpHashBalancer.java
│   ├── health/
│   │   └── HealthChecker.java          ← @Scheduled health checks
│   ├── proxy/
│   │   ├── ReverseProxyHandler.java
│   │   └── ProxyFilter.java           ← Logging, rate limiting
│   ├── model/
│   │   ├── BackendServer.java          ← Record
│   │   └── ProxyRequest.java
│   └── metrics/
│       └── LoadBalancerMetrics.java    ← Micrometer metrics
├── src/test/java/com/challenge/lb/
│   ├── balancer/
│   │   ├── RoundRobinBalancerTest.java
│   │   ├── WeightedRoundRobinTest.java
│   │   └── LeastConnectionsTest.java
│   └── proxy/
│       └── ReverseProxyIntegrationTest.java
├── pom.xml
└── src/main/resources/application.yml
```

**Funcionalidades obrigatórias:**
1. **4 algoritmos** como `@Component` com `@Qualifier`
2. **Strategy Pattern** para trocar algoritmo em runtime
3. **Health checks** com `@Scheduled` e Virtual Threads
4. **Reverse proxy** usando `WebClient` ou `RestClient`
5. **Filters** para logging e rate limiting
6. **Métricas** com Micrometer (counter, timer, gauge)
7. **Configuration** via `application.yml` com `@ConfigurationProperties`
8. **Records** para DTOs (Java 25)

**Critérios de aceite Java:**
- [ ] 4 algoritmos implementados como Spring beans
- [ ] Health check com Virtual Threads (`Thread.ofVirtual()`)
- [ ] Rate limiting filter com token bucket
- [ ] Testes: JUnit 5 + Testcontainers para integration
- [ ] Métricas expostas via Actuator (`/actuator/prometheus`)
- [ ] `application.yml` com configuração de backends
- [ ] `./mvnw test` passa sem erros
- [ ] JaCoCo ≥ 80% coverage nos algoritmos

---

## Parte 4 — Testes de Carga

### Teste de validação

Crie um script que:
1. Inicia 3 backend servers (portas 8081, 8082, 8083)
2. Inicia o Load Balancer (porta 8080)
3. Envia 1000 requests concorrentes
4. Valida distribuição uniforme (±5% para Round Robin)
5. Kill um backend, valida que LB detecta e redistribui

**Ferramentas sugeridas:** `hey`, `wrk`, `k6`, ou script Go/Java

---

## Definição de Pronto (DoD)

- [ ] ADR-001 escrito e revisado
- [ ] DrawIO com 3 views
- [ ] Go: 4 algoritmos + health check + reverse proxy + tests + bench
- [ ] Java: 4 algoritmos + health check + reverse proxy + tests + metrics
- [ ] Teste de carga validando distribuição
- [ ] README com instruções de setup e execução
- [ ] Commit: `feat(system-design-01): load balancer and reverse proxy`
