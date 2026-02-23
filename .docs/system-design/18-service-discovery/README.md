# 18. Service Discovery

> **Categoria:** Comunicação e Coordenação de Microservices  
> **Nível:** Essencial para entrevistas de System Design  
> **Complexidade:** Média

---

## Definição

**Service Discovery** é o mecanismo automático pelo qual microservices **encontram endereços de rede** (IP + porta) uns dos outros, sem configuração hardcoded. Em ambientes dinâmicos (cloud, containers, auto-scaling), instâncias de serviço sobem e descem constantemente — service discovery mantém um **registro atualizado** de quais instâncias estão disponíveis.

---

## Por Que é Importante?

- **Ambientes dinâmicos** — IPs mudam constantemente (containers, auto-scaling, deploys)
- **Elimina configuração hardcoded** — sem manter listas de IPs manualmente
- **Habilita auto-scaling** — novas instâncias se registram automaticamente
- **Load balancing inteligente** — distribuir tráfego entre instâncias saudáveis
- **Pré-requisito para microservices** — sem service discovery, microservices não funcionam em escala
- **Base para** health checks, circuit breakers, canary deployments

---

## O Problema

```
  Monolito: tudo no mesmo processo
  
  OrderService.process() → InventoryService.check() → (mesma JVM)
  
  ────────────────────────────────────────────────
  
  Microservices: serviços em processos/máquinas diferentes
  
  ┌──────────────┐      ┌────────────────────┐
  │ Order Service│─────▶│ Inventory Service  │
  │ (10.0.1.5)   │      │ (???.???.???.???)  │  ← QUAL IP?
  └──────────────┘      └────────────────────┘
  
  Problemas:
  → IP muda a cada deploy/restart
  → Auto-scaling adiciona/remove instâncias
  → Containers recebem IPs dinâmicos
  → Multi-region: instâncias em diferentes data centers
  
  ❌ Hardcoded: inventory-service.url=http://10.0.2.15:8080
  → Funciona até o próximo deploy/scaling event
```

---

## Componentes do Service Discovery

```
  ┌──────────────────────────────────────────────────────────┐
  │                SERVICE DISCOVERY SYSTEM                   │
  │                                                          │
  │  ┌─────────────────────┐                                 │
  │  │   Service Registry  │ ← Banco de dados de serviços   │
  │  │                     │   {service → [instances]}       │
  │  │  payment-svc:       │                                 │
  │  │   - 10.0.1.5:8080  │                                 │
  │  │   - 10.0.1.6:8080  │                                 │
  │  │   - 10.0.2.3:8080  │                                 │
  │  │                     │                                 │
  │  │  inventory-svc:     │                                 │
  │  │   - 10.0.3.1:9090  │                                 │
  │  │   - 10.0.3.2:9090  │                                 │
  │  └─────────────────────┘                                 │
  │         ▲         ▲         │                            │
  │         │         │         │                            │
  │    Register   Heartbeat   Query                          │
  │         │         │         │                            │
  │         │         │         ▼                            │
  │  ┌──────┴─────┐  │  ┌──────────────┐                    │
  │  │  Service   │  │  │   Client     │                    │
  │  │ Instance   │──┘  │  Service     │                    │
  │  └────────────┘     └──────────────┘                    │
  │                                                          │
  │  Fluxo:                                                  │
  │  1. Instância REGISTRA no registry (IP + porta + meta)   │
  │  2. Instância envia HEARTBEATS periódicos                │
  │  3. Client CONSULTA registry para encontrar instâncias   │
  │  4. Se heartbeat falha → instância REMOVIDA do registry  │
  └──────────────────────────────────────────────────────────┘
```

---

## Modelos de Service Discovery

### 1. Client-Side Discovery

```
  Client é responsável por:
  1. Consultar o registry
  2. Escolher instância (load balancing)
  3. Fazer a chamada diretamente
  
  ┌────────────┐  1.query  ┌──────────────────┐
  │   Client   │──────────▶│ Service Registry │
  │  Service   │◀──────────│ (Eureka/Consul)  │
  └──────┬─────┘  2.return │                  │
         │        instances└──────────────────┘
         │
         │ 3. Choose instance (round-robin, random, etc.)
         │
         ├──────────▶ Instance A (10.0.1.5:8080) ← escolhido
         ├──────────▶ Instance B (10.0.1.6:8080)
         └──────────▶ Instance C (10.0.2.3:8080)
```

| Prós | Contras |
|------|---------|
| Client controla LB strategy | Lógica de discovery em cada client |
| Menos hop de rede | Acoplamento com registry |
| Client-side caching | Cada linguagem precisa de SDK |

**Exemplos:** Netflix Eureka + Ribbon, Spring Cloud LoadBalancer

### 2. Server-Side Discovery

```
  Load Balancer / Router é responsável por:
  1. Consultar o registry
  2. Escolher instância
  3. Rotear o request
  
  ┌────────────┐          ┌───────────────┐          ┌────────────┐
  │   Client   │────req──▶│ Load Balancer │────req──▶│ Instance A │
  │  Service   │◀───resp──│ / Router      │          ├────────────┤
  └────────────┘          │               │          │ Instance B │
                          │  Consulta     │          ├────────────┤
                          │  registry     │          │ Instance C │
                          └───────┬───────┘          └────────────┘
                                  │
                          ┌───────▼───────┐
                          │   Service     │
                          │   Registry    │
                          └───────────────┘
```

| Prós | Contras |
|------|---------|
| Client simples (só faz request) | LB é ponto extra de latência |
| Linguagem agnóstica | LB pode ser SPOF (precisa HA) |
| Centralizado, fácil de gerenciar | Mais infraestrutura para manter |

**Exemplos:** AWS ALB + ECS, Kubernetes Service + kube-proxy, NGINX + Consul

### 3. DNS-Based Discovery

```
  Resolução de nomes via DNS:
  
  Client: "Preciso falar com payment-service"
  
  DNS query: payment-service.default.svc.cluster.local
  DNS response: 10.0.1.5, 10.0.1.6, 10.0.2.3  (A records)
  
  ┌────────────┐  DNS query   ┌──────────┐
  │   Client   │─────────────▶│   DNS    │
  │  Service   │◀─────────────│  Server  │
  └──────┬─────┘  IP list     └──────────┘
         │
         ├──▶ 10.0.1.5 (payment-svc instance 1)
         ├──▶ 10.0.1.6 (payment-svc instance 2)
         └──▶ 10.0.2.3 (payment-svc instance 3)
```

| Prós | Contras |
|------|---------|
| Universal (todo client suporta DNS) | DNS caching → stale entries |
| Sem SDK especial | TTL limita velocidade de atualização |
| Simples de entender | Sem health checking integrado |

**Exemplos:** Kubernetes DNS (CoreDNS), AWS Cloud Map, Consul DNS interface

### 4. Service Mesh (Sidecar Proxy)

```
  Sidecar proxy gerencia discovery transparentemente:
  
  ┌──────────────────────────────────┐
  │ Pod A                            │
  │ ┌────────────┐  ┌─────────────┐ │
  │ │   Client   │──│ Envoy Proxy │─┼───────▶ Pod B (Envoy → Service B)
  │ │  Service A │  │  (sidecar)  │ │
  │ └────────────┘  └─────────────┘ │
  └──────────────────────────────────┘
  
  O Envoy sidecar:
  1. Intercepta TODA comunicação de saída
  2. Consulta service registry (control plane)
  3. Faz load balancing, retry, circuit breaking
  4. Tudo TRANSPARENTE para a aplicação
  
  Aplicação chama: http://payment-service:8080/pay
  Envoy resolve, balanceia e roteia automaticamente
```

| Prós | Contras |
|------|---------|
| Transparente para a aplicação | Complexidade operacional |
| Linguagem agnóstica | Latência extra (sidecar hop) |
| Features avançadas (mTLS, tracing) | Mais recursos (CPU/RAM por sidecar) |
| Gerenciamento centralizado (control plane) | Curva de aprendizado alta |

**Exemplos:** Istio + Envoy, Linkerd, AWS App Mesh

---

## Registration Patterns

### Self-Registration

```
  Serviço registra A SI MESMO no registry:
  
  ┌──────────────────┐        ┌──────────────────┐
  │ Service Instance │──reg──▶│ Service Registry │
  │                  │◀──ack──│                  │
  │ Responsável por: │        │                  │
  │ • Register       │        │                  │
  │ • Heartbeat      │        │                  │
  │ • Deregister     │        │                  │
  └──────────────────┘        └──────────────────┘
  
  Startup: POST /registry { name: "payment-svc", host: "10.0.1.5", port: 8080 }
  Running: PUT /registry/heartbeat (a cada 30s)
  Shutdown: DELETE /registry/payment-svc/10.0.1.5
```

**Exemplos:** Eureka client, Consul agent (service registration)

### Third-Party Registration

```
  Componente externo observa e registra serviços:
  
  ┌──────────────────┐        ┌──────────────────┐
  │ Service Instance │        │ Service Registry │
  │ (não sabe do     │        │                  │
  │  registry)       │        │                  │
  └──────────────────┘        └──────────────────┘
          ▲                           ▲
          │ monitor                   │ register
          │                           │
  ┌───────┴───────────────────────────┴──────────┐
  │            Service Registrar                  │
  │ (Kubernetes, AWS ECS, Netflix Prana)          │
  │ Monitora containers e registra automaticamente │
  └──────────────────────────────────────────────┘
```

**Exemplos:** Kubernetes (kubelet + endpoints controller), AWS ECS (task registration)

---

## Tecnologias

### Comparativo

| Tecnologia | Tipo | Consistência | Health Check | DNS Interface | Key-Value Store |
|------------|------|-------------|--------------|---------------|-----------------|
| **Eureka** | AP (eventual) | Eventual | Client heartbeat | Não | Não |
| **Consul** | CP (Raft) | Strong | TCP/HTTP/gRPC | Sim | Sim |
| **Zookeeper** | CP (ZAB) | Strong | Session/Ephemeral | Não | Sim |
| **etcd** | CP (Raft) | Strong | Lease/TTL | Não | Sim |
| **Kubernetes DNS** | — | Eventual | Probes (liveness/readiness) | Sim (nativo) | — |
| **AWS Cloud Map** | Managed | Eventual | Health checks | Sim | Não |

### Netflix Eureka

```
  ┌─────────────────────────────────────────────────────┐
  │                   Eureka Architecture               │
  │                                                     │
  │  ┌──────────┐     ┌──────────┐                     │
  │  │ Eureka   │◀═══▶│ Eureka   │  Peer replication   │
  │  │ Server 1 │     │ Server 2 │  (AP: eventual)     │
  │  └────┬─────┘     └────┬─────┘                     │
  │       │                │                            │
  │       │ register       │ register                   │
  │       │ heartbeat      │ heartbeat                  │
  │       │ fetch registry │ fetch registry             │
  │       │                │                            │
  │  ┌────▼─────┐    ┌────▼─────┐    ┌──────────┐     │
  │  │Service A │    │Service B │    │Service C │     │
  │  │(client)  │    │(client)  │    │(client)  │     │
  │  └──────────┘    └──────────┘    └──────────┘     │
  │                                                     │
  │  Eureka Client:                                     │
  │  • Registra no startup                              │
  │  • Heartbeat a cada 30s                             │
  │  • Fetches registry a cada 30s (cache local)       │
  │  • Self-preservation: se muitos heartbeats falham   │
  │    simultaneamente → Eureka mantém registry         │
  │    (assume falha de rede, não de serviços)          │
  └─────────────────────────────────────────────────────┘
```

### HashiCorp Consul

```
  ┌─────────────────────────────────────────────────────┐
  │                  Consul Architecture                 │
  │                                                     │
  │  ┌────────────────────────────────────────────┐     │
  │  │          Consul Servers (Raft cluster)     │     │
  │  │  ┌────────┐  ┌────────┐  ┌────────┐      │     │
  │  │  │ Leader │  │Follower│  │Follower│      │     │
  │  │  └────────┘  └────────┘  └────────┘      │     │
  │  └─────────────────┬──────────────────────────┘     │
  │                    │                                 │
  │         ┌──────────┼──────────┐                     │
  │         │          │          │                     │
  │  ┌──────▼───┐ ┌────▼─────┐ ┌─▼──────────┐         │
  │  │ Consul   │ │ Consul   │ │ Consul     │         │
  │  │ Agent    │ │ Agent    │ │ Agent      │         │
  │  │ (client) │ │ (client) │ │ (client)   │         │
  │  │          │ │          │ │            │         │
  │  │ Service A│ │ Service B│ │ Service C  │         │
  │  └──────────┘ └──────────┘ └────────────┘         │
  │                                                     │
  │  Features:                                          │
  │  • Service discovery + health checking              │
  │  • Key-value store                                  │
  │  • Multi-datacenter                                 │
  │  • Service mesh (Consul Connect)                    │
  │  • DNS + HTTP API interfaces                        │
  └─────────────────────────────────────────────────────┘
```

### Kubernetes Service Discovery

```
  ┌─────────────────────────────────────────────────────┐
  │             Kubernetes Service Discovery             │
  │                                                     │
  │  ┌───────────────────────────────────────────┐      │
  │  │         kube-apiserver + etcd             │      │
  │  └─────────────────┬─────────────────────────┘      │
  │                    │                                 │
  │    ┌───────────────┼───────────────┐                │
  │    │               │               │                │
  │    ▼               ▼               ▼                │
  │ ┌────────┐    ┌────────┐    ┌────────┐             │
  │ │ CoreDNS│    │kube-   │    │Endpoints│             │
  │ │        │    │proxy   │    │Controller│            │
  │ └────────┘    └────────┘    └──────────┘            │
  │                                                     │
  │  Resolução:                                         │
  │  payment-svc.default.svc.cluster.local              │
  │  ├── ClusterIP: 10.96.0.15 (virtual IP)            │
  │  └── kube-proxy → iptables/IPVS → Pod IPs          │
  │      ├── 10.244.1.5:8080 (pod 1)                   │
  │      ├── 10.244.2.3:8080 (pod 2)                   │
  │      └── 10.244.3.7:8080 (pod 3)                   │
  │                                                     │
  │  Tipos de Service:                                  │
  │  • ClusterIP: interno ao cluster (default)          │
  │  • NodePort: expõe em porta do nó                   │
  │  • LoadBalancer: provisiona LB externo (cloud)      │
  │  • Headless: retorna Pod IPs diretamente (DNS)      │
  └─────────────────────────────────────────────────────┘
```

---

## Implementação — Spring Cloud + Eureka

### Eureka Server

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml (Eureka Server)
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    enable-self-preservation: true
    eviction-interval-timer-in-ms: 60000
```

### Service Registration (Client)

```java
@SpringBootApplication
@EnableDiscoveryClient
public class PaymentServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentServiceApplication.class, args);
    }
}
```

```yaml
# application.yml (Payment Service)
spring:
  application:
    name: payment-service

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
    registry-fetch-interval-seconds: 30
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90
    metadata-map:
      version: v2.1.0
      region: us-east-1
```

### Service Consumption (Client-Side LB)

```java
@Service
public class OrderService {

    @Autowired
    private WebClient.Builder webClientBuilder; // auto-configured with LB

    public Mono<InventoryResponse> checkInventory(String productId) {
        return webClientBuilder.build()
            .get()
            .uri("http://inventory-service/api/inventory/{id}", productId)
            //    ^^^^^^^^^^^^^^^^^ service name (NOT IP!)
            // Spring Cloud LoadBalancer resolve via Eureka
            .retrieve()
            .bodyToMono(InventoryResponse.class);
    }
}

@Configuration
public class WebClientConfig {
    @Bean
    @LoadBalanced  // ← habilita client-side load balancing via Service Discovery
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}
```

---

## Health-Aware Discovery

```
  Service Registry deve só retornar instâncias SAUDÁVEIS:
  
  Registry:
  ┌──────────────────────────────────────────────┐
  │ payment-service:                              │
  │   10.0.1.5:8080  status: UP    ✅ → retorna  │
  │   10.0.1.6:8080  status: UP    ✅ → retorna  │
  │   10.0.2.3:8080  status: DOWN  ❌ → omite    │
  │   10.0.2.4:8080  status: UP    ✅ → retorna  │
  └──────────────────────────────────────────────┘
  
  Client query: "payment-service instances?"
  Response: [10.0.1.5, 10.0.1.6, 10.0.2.4]  (sem .2.3)
  
  Mecanismos:
  → Heartbeat timeout (Eureka: 90s sem heartbeat → remove)
  → Active health check (Consul: HTTP/TCP/gRPC probes)
  → Readiness probe (K8s: remove de Endpoints se not ready)
```

---

## Uso em Big Techs

### Netflix — Eureka (criadores)
- Inventaram o Eureka para service discovery
- AP model: prioriza availability (self-preservation mode)
- Client-side load balancing (Ribbon, agora Spring Cloud LB)
- Centenas de microservices se registram no Eureka
- Cache local no client (resiste a falha do Eureka)

### Uber — Custom Service Discovery
- Começaram com Hyperbahn (TChannel-based)
- Migraram para Ringpop (consistent hashing ring)
- Hoje usam combinação de service mesh + DNS
- Milhares de microservices em múltiplas regiões

### Google — Kubernetes + Istio
- Kubernetes DNS como base (CoreDNS)
- Istio service mesh para discovery avançado
- Traffic Director para multi-cluster discovery
- gRPC name resolution integrado

### Amazon — AWS Cloud Map + ECS
- Cloud Map: managed service registry (DNS + API)
- ECS: auto-registration de containers
- App Mesh: managed service mesh (Envoy)
- Route 53: DNS-based discovery para serviços AWS

---

## Perguntas Comuns em Entrevistas

1. **O que é Service Discovery e por que é necessário?**
   - Mecanismo para serviços encontrarem endereços uns dos outros em ambientes dinâmicos (cloud, containers, auto-scaling).

2. **Client-Side vs Server-Side Discovery?**
   - Client-side: client consulta registry e faz LB (Eureka). Server-side: LB consulta e roteia (K8s Service, ALB).

3. **O que acontece se o Service Registry cai?**
   - Client-side: usa cache local. Self-preservation (Eureka): mantém registry antigo. Redundância: múltiplas instâncias do registry.

4. **Como evitar que instâncias mortas recebam tráfego?**
   - Heartbeat + health checks. Remoção automática após timeout. Readiness probes em Kubernetes.

5. **Eureka vs Consul vs Kubernetes DNS?**
   - Eureka: AP, Java-centric. Consul: CP, multi-language, multi-DC. K8s DNS: nativo se já usa K8s.

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Modelo** | Client-side (Eureka, flexível) | Server-side (K8s, simples) |
| **Consistência** | AP (Eureka, sempre disponível) | CP (Consul/etcd, sempre correto) |
| **Registration** | Self-registration (simples) | Third-party (desacoplado) |
| **Interface** | DNS (universal) | HTTP API (rico, metadata) |
| **Cache** | Client cache (resiliente) | No cache (sempre atualizado) |
| **Health check** | Client heartbeat (pull) | Server active check (push) |
| **Plataforma** | Managed (Cloud Map, K8s) | Self-hosted (Eureka, Consul) |

---

## Referências

- [Netflix Eureka — GitHub](https://github.com/Netflix/eureka)
- [HashiCorp Consul — Service Discovery](https://developer.hashicorp.com/consul/docs/concepts/service-discovery)
- [Kubernetes — Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Chris Richardson — Microservices Patterns](https://microservices.io/patterns/service-discovery.html)
- [Spring Cloud Netflix — Eureka](https://spring.io/projects/spring-cloud-netflix)
- [Istio — Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)
