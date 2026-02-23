# 19. Heartbeat e Health Checks

> **Categoria:** Monitoramento e Resiliência  
> **Nível:** Essencial para entrevistas de System Design  
> **Complexidade:** Média

---

## Definição

**Heartbeat** é um sinal periódico enviado por um componente para indicar que está vivo e funcionando. **Health Check** é uma verificação ativa do estado de saúde de um serviço — pode ser feita pelo próprio serviço (self-report) ou por um componente externo (prober). Juntos, formam a base para **detecção de falhas**, **remoção automática** de instâncias doentes e **recuperação automática** em sistemas distribuídos.

---

## Por Que é Importante?

- **Detecção de falhas** — identifica instâncias que crasharam, travaram ou estão degradadas
- **Remoção automática** — service discovery remove instâncias não saudáveis do balanceamento
- **Auto-healing** — orquestradores (K8s) reiniciam containers com falha automaticamente
- **Prevenção de cascading failures** — não enviar tráfego para instâncias doentes
- **Observabilidade** — dashboards e alertas baseados em status de saúde
- **Complementa** service discovery, load balancing e circuit breakers

---

## Heartbeat Pattern

```
  ┌─────────────┐                    ┌───────────────────┐
  │  Service     │   "estou vivo"    │   Monitor /       │
  │  Instance    │──────────────────▶│   Coordinator     │
  │              │   (heartbeat)     │                   │
  │              │   a cada Δt       │   Se não receber  │
  │              │                   │   heartbeat por   │
  │              │                   │   N × Δt:         │
  │              │                   │   → marca DEAD    │
  └─────────────┘                   └───────────────────┘
  
  Exemplo concreto:
  ─────── Timeline ──────────────────────────────────────
  
  t=0s    HB ✓
  t=30s   HB ✓
  t=60s   HB ✓
  t=90s   (nada)  ← missed heartbeat
  t=120s  (nada)  ← missed heartbeat  
  t=150s  (nada)  ← timeout! → instância marcada como DEAD
  
  Configuração típica:
  • Interval: 30s (envia heartbeat a cada 30s)
  • Timeout: 90s (3 × interval → 3 heartbeats perdidos = dead)
```

---

## Tipos de Health Check

### 1. Liveness Check

```
  "O processo está rodando e responsivo?"
  
  Verifica: processo não travou / não está em deadlock
  Ação se falha: REINICIAR o container/processo
  
  ┌──────────────┐    GET /health/live    ┌──────────┐
  │  Orchestrator │──────────────────────▶│ Service  │
  │  (K8s kubelet)│◀──────────────────────│          │
  └──────────────┘    200 OK / timeout    └──────────┘
  
  Exemplos:
  ✓ HTTP endpoint retorna 200
  ✓ TCP connection aceita
  ✓ Processo responde a sinal
  ✗ Deadlock (thread travada) → timeout → restart
  ✗ OOM mas processo vivo → health check passa mas não funciona
```

### 2. Readiness Check

```
  "O serviço está PRONTO para receber tráfego?"
  
  Verifica: dependências conectadas, warmup concluído, cache carregado
  Ação se falha: REMOVER do load balancer (não mata, só para de enviar tráfego)
  
  ┌──────────────┐   GET /health/ready   ┌──────────────────────┐
  │  Orchestrator │──────────────────────▶│ Service              │
  │              │◀──────────────────────│                      │
  └──────────────┘   503 Not Ready       │ Checks:              │
                                          │ ✅ DB connection     │
                                          │ ❌ Cache warmup: 60% │
                                          │ ✅ Config loaded     │
                                          │ → NOT READY          │
                                          └──────────────────────┘
  
  Cenários:
  → Startup: DB ainda não conectou → not ready → sem tráfego
  → Runtime: DB connection pool esgotado → not ready → retira do LB
  → Recovery: DB reconecta → ready → volta ao LB
```

### 3. Startup Check

```
  "A aplicação terminou de inicializar?"
  
  Para apps com startup lento (carregar ML model, popular cache)
  Enquanto startup check não passa → liveness/readiness NÃO são verificados
  
  Timeline:
  ┌────────────────────────────────────────────────────────────┐
  │  [Container Start]                                         │
  │  │                                                         │
  │  ├── Startup probe ativo (failureThreshold: 30, period: 10s)│
  │  │   → Permite até 300s para inicializar                   │
  │  │                                                         │
  │  ├── t=0s:   startup probe → fail (loading ML model...)    │
  │  ├── t=10s:  startup probe → fail                          │
  │  ├── t=20s:  startup probe → fail                          │
  │  ├── ...                                                   │
  │  ├── t=120s: startup probe → SUCCESS ✓                     │
  │  │                                                         │
  │  ├── Liveness + Readiness probes ATIVADOS                  │
  │  ├── t=150s: liveness ✓, readiness ✓ → tráfego começa     │
  └────────────────────────────────────────────────────────────┘
```

### Comparativo

| Tipo | Pergunta | Ação se Falha | Quando Verificar |
|------|----------|---------------|------------------|
| **Liveness** | "Está vivo?" | Restart container | Sempre (após startup) |
| **Readiness** | "Está pronto?" | Remove do LB | Sempre (após startup) |
| **Startup** | "Terminou de inicializar?" | Espera / Restart | Somente durante boot |

---

## Mecanismos de Verificação

### HTTP Health Check

```
  GET /health HTTP/1.1
  Host: service.local
  
  Resposta saudável (200):
  {
    "status": "UP",
    "checks": {
      "database": { "status": "UP", "responseTime": "5ms" },
      "redis": { "status": "UP", "responseTime": "2ms" },
      "diskSpace": { "status": "UP", "free": "15GB" }
    },
    "uptime": "72h30m",
    "version": "2.1.0"
  }
  
  Resposta não saudável (503):
  {
    "status": "DOWN",
    "checks": {
      "database": { "status": "DOWN", "error": "Connection refused" },
      "redis": { "status": "UP", "responseTime": "2ms" },
      "diskSpace": { "status": "UP", "free": "15GB" }
    }
  }
```

### TCP Health Check

```
  Apenas verifica se a porta aceita conexão:
  
  Prober ──── TCP SYN ────▶ Service:8080
  Prober ◀─── TCP SYN-ACK ── Service:8080  → HEALTHY ✓
  
  Prober ──── TCP SYN ────▶ Service:8080
  (timeout, RST, ou connection refused)     → UNHEALTHY ✗
  
  Uso: serviços que não suportam HTTP (databases, message brokers)
```

### gRPC Health Check

```protobuf
// Standard gRPC Health Checking Protocol
syntax = "proto3";

package grpc.health.v1;

service Health {
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}

message HealthCheckRequest {
  string service = 1;  // nome do serviço específico
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;        // saudável
    NOT_SERVING = 2;    // não saudável
    SERVICE_UNKNOWN = 3;
  }
  ServingStatus status = 1;
}
```

### Command-Based Check

```
  Executa um comando dentro do container:
  
  exec: ["pg_isready", "-U", "postgres"]
  → exit code 0 = healthy
  → exit code != 0 = unhealthy
  
  exec: ["redis-cli", "ping"]
  → "PONG" (exit 0) = healthy
```

---

## Kubernetes Health Probes

### Configuração Completa

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: payment-service
        image: payment-service:2.1.0
        ports:
        - containerPort: 8080

        # Startup Probe — verifica se a app inicializou
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 30   # 30 × 10s = 300s max para startup
          successThreshold: 1

        # Liveness Probe — verifica se está vivo
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 0   # inicia logo após startup probe OK
          periodSeconds: 15
          failureThreshold: 3      # 3 falhas consecutivas → restart
          successThreshold: 1
          timeoutSeconds: 5

        # Readiness Probe — verifica se está pronto para tráfego
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 10
          failureThreshold: 3      # 3 falhas → remove do Service
          successThreshold: 2      # 2 sucessos → volta ao Service
          timeoutSeconds: 5
```

### Fluxo no Kubernetes

```
  ┌────────────────────────────────────────────────────────┐
  │ Pod Lifecycle com Health Probes                         │
  │                                                        │
  │ [Create Pod]                                           │
  │      │                                                 │
  │      ▼                                                 │
  │ ┌─────────────────┐                                    │
  │ │ Container Start │                                    │
  │ └────────┬────────┘                                    │
  │          │                                             │
  │          ▼                                             │
  │ ┌─────────────────────────┐                            │
  │ │ Startup Probe (looping) │                            │
  │ │ Fail → aguarda          │                            │
  │ │ Success → continua ▼    │                            │
  │ └─────────────────────────┘                            │
  │          │                                             │
  │          ├──────────────────────────┐                   │
  │          ▼                          ▼                   │
  │ ┌──────────────────┐    ┌───────────────────┐          │
  │ │  Liveness Probe  │    │ Readiness Probe   │          │
  │ │  (periódico)     │    │ (periódico)       │          │
  │ │                  │    │                   │          │
  │ │  Fail (3x) →     │    │  Fail (3x) →      │          │
  │ │  RESTART container│    │  Remove do Service│          │
  │ │                  │    │  (Endpoints)      │          │
  │ │  Success →       │    │                   │          │
  │ │  container OK    │    │  Success (2x) →   │          │
  │ └──────────────────┘    │  Adiciona de volta│          │
  │                         └───────────────────┘          │
  └────────────────────────────────────────────────────────┘
```

---

## Implementação — Spring Boot Actuator

### Configuração

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true   # habilita /health/liveness e /health/readiness
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

### Custom Health Indicator

```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final PaymentGatewayClient gatewayClient;

    public PaymentGatewayHealthIndicator(PaymentGatewayClient gatewayClient) {
        this.gatewayClient = gatewayClient;
    }

    @Override
    public Health health() {
        try {
            long start = System.currentTimeMillis();
            boolean reachable = gatewayClient.ping();
            long latency = System.currentTimeMillis() - start;

            if (reachable && latency < 5000) {
                return Health.up()
                    .withDetail("gateway", "reachable")
                    .withDetail("latencyMs", latency)
                    .build();
            } else if (reachable) {
                return Health.status("DEGRADED")
                    .withDetail("gateway", "slow")
                    .withDetail("latencyMs", latency)
                    .build();
            } else {
                return Health.down()
                    .withDetail("gateway", "unreachable")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("gateway", "error")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### Readiness State Management

```java
@Service
public class CacheWarmupService implements ApplicationListener<ApplicationReadyEvent> {

    private final ApplicationAvailability availability;
    private final ApplicationEventPublisher eventPublisher;

    public CacheWarmupService(ApplicationAvailability availability,
                               ApplicationEventPublisher eventPublisher) {
        this.availability = availability;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        // Marcar como NOT accepting traffic durante warmup
        AvailabilityChangeEvent.publish(eventPublisher, this,
            ReadinessState.REFUSING_TRAFFIC);

        try {
            warmupCache(); // Processo lento
            
            // Warmup completo → aceitar tráfego
            AvailabilityChangeEvent.publish(eventPublisher, this,
                ReadinessState.ACCEPTING_TRAFFIC);
        } catch (Exception e) {
            // Falha no warmup → manter not ready
            AvailabilityChangeEvent.publish(eventPublisher, this,
                ReadinessState.REFUSING_TRAFFIC);
        }
    }

    private void warmupCache() {
        // Carregar dados em cache...
    }
}
```

---

## Heartbeat em Sistemas Distribuídos

### Leader ↔ Follower Heartbeat

```
  ┌──────────┐   heartbeat + log entries   ┌──────────┐
  │  Leader   │───────────────────────────▶│ Follower │
  │          │◀───────────────────────────│          │
  └──────────┘   ack + log position       └──────────┘
  
  Se Follower não recebe heartbeat do Leader por T ms:
  → Inicia Leader Election
  
  Raft: heartbeat interval = 150ms, election timeout = 150-300ms
  
  Se Leader não recebe ack do Follower por T ms:
  → Marca Follower como unavailable
  → Não conta no quorum
```

### Gossip-Based Failure Detection

```
  Alternativa a heartbeat centralizado:
  
  Cada nó "fofoca" com N nós aleatórios periodicamente:
  
  Nó A: "Eu vi B (t=100), C (t=95), D (t=98)"
  Nó B: "Eu vi A (t=99), C (t=80), D (t=97)"
  
  Se timestamp de C for muito antigo em TODOS os nós:
  → C provavelmente está morto
  
  Vantagem: sem coordenador central para monitorar
  Usado em: Cassandra, DynamoDB, Consul (Serf)
```

---

## Padrões Avançados

### Phi Accrual Failure Detector

```
  Problema: heartbeat timeout fixo gera falsos positivos/negativos
  
  Solução: detector adaptativo baseado em estatísticas
  
  ┌──────────────────────────────────────────────────────────┐
  │ Phi (φ) Accrual Failure Detector                         │
  │                                                          │
  │ Monitora intervalos entre heartbeats:                    │
  │ [28ms, 32ms, 30ms, 31ms, 29ms, 33ms, ...]              │
  │                                                          │
  │ Calcula distribuição normal:                             │
  │ μ (mean) = 30.4ms                                       │
  │ σ (std dev) = 1.7ms                                     │
  │                                                          │
  │ Se último heartbeat foi há 45ms:                         │
  │ φ = -log₁₀(1 - F(45ms)) ≈ 8.2                         │
  │                                                          │
  │ Threshold: φ > 8 → provavelmente morto                  │
  │ Threshold: φ > 12 → definitivamente morto               │
  │                                                          │
  │ Benefício: adapta-se a variações de rede                │
  │ (latência maior → threshold se ajusta)                  │
  └──────────────────────────────────────────────────────────┘
  
  Usado em: Akka Cluster, Cassandra, Amazon (internal)
```

### Graceful Shutdown

```
  ┌────────────────────────────────────────────────────┐
  │ Graceful Shutdown Flow                              │
  │                                                    │
  │ 1. SIGTERM recebido                                │
  │    │                                               │
  │ 2. ├── Marcar readiness = NOT READY                │
  │    │   → Load balancer para de enviar tráfego      │
  │    │                                               │
  │ 3. ├── Aguardar drain period (ex: 30s)             │
  │    │   → Requests em andamento terminam            │
  │    │                                               │
  │ 4. ├── Deregister do service registry              │
  │    │   → Eureka/Consul remove instância            │
  │    │                                               │
  │ 5. ├── Fechar connections (DB, Redis, MQ)          │
  │    │                                               │
  │ 6. └── Exit process                                │
  │                                                    │
  │ Kubernetes: terminationGracePeriodSeconds: 60      │
  │ preStop hook → sleep 5 (dá tempo pro LB atualizar) │
  └────────────────────────────────────────────────────┘
```

---

## Uso em Big Techs

### Netflix
- Eureka heartbeat: instâncias enviam heartbeat a cada 30s
- Self-preservation: se >15% dos heartbeats falham de uma vez, Eureka assume falha de rede e NÃO remove instâncias
- Health check extensivo com custom indicators

### Google (Kubernetes)
- 3 tipos de probe (liveness, readiness, startup)
- Chap (internal): distributed health checking system
- gRPC health checking protocol (criado pelo Google)

### Amazon
- ELB health checks: HTTP/TCP em intervalos configuráveis
- Route 53 health checks: remove DNS records de instâncias unhealthy
- Phi Accrual Failure Detector em sistemas internos (DynamoDB)

### Uber
- Health check cascade: serviço reporta saúde das dependências
- Custom failure detector baseado em latência (p99)
- Tenacity framework para health-aware load balancing

---

## Perguntas Comuns em Entrevistas

1. **Qual a diferença entre Liveness e Readiness probe?**
   - Liveness: "está vivo?" — falha → restart. Readiness: "está pronto?" — falha → remove do LB.

2. **Como evitar falsos positivos em health checks?**
   - Múltiplas falhas antes de declarar unhealthy (failureThreshold). Phi Accrual detector. Timeouts realistas.

3. **O que é Graceful Shutdown e por que é importante?**
   - Processo que drena requests em andamento antes de morrer. Evita erros 5xx durante deploys.

4. **Heartbeat vs Active Health Check?**
   - Heartbeat: serviço ENVIA sinal (push). Active: monitor VERIFICA o serviço (pull). Active detecta mais tipos de falha.

5. **Como health checks se integram com service discovery?**
   - Registry só retorna instâncias saudáveis. Heartbeat falha → remove do registry → clients não recebem IP morto.

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Modelo** | Push heartbeat (serviço envia) | Pull health check (monitor consulta) |
| **Timeout** | Curto (detecta rápido) | Longo (menos falsos positivos) |
| **Granularidade** | Simples (UP/DOWN) | Detalhado (checks por dependência) |
| **Probe type** | HTTP (rico em info) | TCP (lightweight) |
| **Falha** | Restart (liveness) | Remove do LB (readiness) |
| **Detector** | Fixed timeout (simples) | Phi Accrual (adaptativo) |
| **Health exposure** | Expor detalhes (debug) | Esconder detalhes (segurança) |

---

## Referências

- [Kubernetes — Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Spring Boot — Health Indicators](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.endpoints.health)
- [gRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md)
- [The Phi Accrual Failure Detector — Hayashibara et al.](https://www.computer.org/csdl/proceedings-article/srds/2004/22390066/12OmNBiWOrY)
- [Netflix Eureka — Self-Preservation](https://github.com/Netflix/eureka/wiki/Server-Self-Preservation-Mode)
