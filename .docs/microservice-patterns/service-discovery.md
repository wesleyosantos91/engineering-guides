# Service Discovery

> **Categoria:** Infraestrutura e Deploy
> **Complementa:** API Gateway/BFF, Health Checks, Sidecar
> **Keywords:** service discovery, registro de serviços, client-side discovery, server-side discovery, service registry, DNS

---

## Problema

Em ambientes dinâmicos (containers, cloud), os serviços:

- **Mudam de IP** a cada deploy, restart ou scaling
- **Escalam** horizontalmente — múltiplas instâncias do mesmo serviço
- São **criados e destruídos** continuamente (autoscaling)

Hardcodar IPs e portas é inviável. Como um serviço **descobre** onde estão os outros serviços?

---

## Solução

Um mecanismo de **Service Discovery** que mantém um **registro** (registry) de todos os serviços e suas instâncias disponíveis, permitindo que serviços se encontrem dinamicamente.

---

## Tipos de Service Discovery

### 1. Client-Side Discovery

O **client** consulta o service registry e escolhe qual instância chamar:

```
┌──────────┐    1. Consulta registry    ┌──────────────────┐
│  Service │───────────────────────────▶│ Service Registry │
│  Client  │◀──────────────────────────│ (Consul, Eureka) │
│          │    2. Retorna lista:       └──────────────────┘
│          │       - svc-a:8080 (healthy)
│          │       - svc-a:8081 (healthy)
│          │       - svc-a:8082 (unhealthy)
│          │
│          │    3. Escolhe instância (load balance no client)
│          │───────────────────────────▶ svc-a:8081
└──────────┘
```

| Vantagem | Desvantagem |
|----------|------------|
| Menos hops (sem proxy intermediário) | Cada client precisa ter lógica de discovery |
| Client controla load balancing | Lógica acoplada ao client |
| Menos latência | Cada linguagem precisa de SDK |

### 2. Server-Side Discovery

Um **proxy/load balancer** consulta o registry e roteia o request:

```
┌──────────┐    1. Request     ┌──────────────┐    2. Roteia    ┌──────────┐
│  Service │──────────────────▶│ Load Balancer│───────────────▶│ Service A│
│  Client  │                   │ / Proxy      │                │ (instância)│
└──────────┘                   └──────┬───────┘                └──────────┘
                                      │
                               3. Consulta registry
                                      │
                               ┌──────▼───────┐
                               │   Service    │
                               │   Registry   │
                               └──────────────┘
```

| Vantagem | Desvantagem |
|----------|------------|
| Client simples (só faz request) | Proxy adiciona latência (hop extra) |
| Lógica centralizada no proxy | Proxy é ponto central (single point of failure) |
| Agnóstico de linguagem | Mais infra para gerenciar |

### 3. DNS-Based Discovery

Serviços registrados como **registros DNS** (A records, SRV records):

```
Resolução DNS:
  order-service.myapp.local → [10.0.1.5, 10.0.1.6, 10.0.1.7]

Client:
  http://order-service.myapp.local:8080/orders
  → DNS resolve para um dos IPs
```

| Vantagem | Desvantagem |
|----------|------------|
| Simples — todo sistema entende DNS | TTL do cache DNS causa stale entries |
| Sem SDK especial | Sem health checking granular |
| Funciona cross-linguagem | Load balancing limitado (round-robin) |

---

## Service Registry

O **Service Registry** é o banco de dados central de serviços:

| Item | Descrição |
|------|-----------|
| **Nome do serviço** | `order-service` |
| **Instâncias** | `10.0.1.5:8080`, `10.0.1.6:8080` |
| **Status** | `UP`, `DOWN`, `STARTING` |
| **Metadados** | Versão, região, zona, tags |
| **Health check** | URL de health, intervalo, threshold |

### Registro: Self-Registration vs. Third-Party

| Abordagem | Como funciona |
|-----------|--------------|
| **Self-Registration** | O próprio serviço se registra no registry ao iniciar e se desregistra ao parar |
| **Third-Party Registration** | Um agente externo (registrar, orchestrator) detecta serviços e registra no registry |

```
Self-Registration:
  Serviço inicia → POST /register { name: "order-service", host: "10.0.1.5", port: 8080 }
  Serviço para  → DELETE /deregister { ... }
  Heartbeat     → PUT /heartbeat (a cada 30s)

Third-Party (Kubernetes):
  Kubernetes cria Pod → registra automaticamente no DNS interno (kube-dns)
  Kubernetes remove Pod → remove do DNS
  → Serviço NÃO precisa saber sobre discovery
```

---

## Service Discovery por Plataforma

| Plataforma | Mecanismo |
|-----------|-----------|
| **Kubernetes** | DNS interno (kube-dns/CoreDNS). Cada Service tem um nome DNS. Pod IPs gerenciados pelo cluster. |
| **Consul (HashiCorp)** | Registry com health checking, DNS interface, Key-Value store |
| **AWS ECS** | Service Discovery via Cloud Map (DNS + HTTP) |
| **AWS ELB** | Server-side discovery via Load Balancer + Target Groups |
| **Docker Compose** | DNS interno por nome do serviço (ex: `http://order-service:8080`) |
| **Istio (Service Mesh)** | Sidecar proxy (Envoy) descobre serviços via control plane |

---

## Health Checking no Discovery

O registry precisa saber se uma instância está **saudável** para não rotear tráfego para ela:

```
Registry monitora:
  order-service:8080 → GET /health → 200 OK     → HEALTHY
  order-service:8081 → GET /health → 503         → UNHEALTHY → remove do pool
  order-service:8082 → GET /health → timeout     → UNHEALTHY → remove do pool
```

| Tipo | Quem verifica |
|------|-------------|
| **Active (push)** | Registry verifica periodicamente (pull health) |
| **Passive (pull)** | Serviço envia heartbeat periodicamente (push health) |
| **TTL** | Se o heartbeat não chegar em X segundos, marca como unhealthy |

---

## Exemplo Conceitual (Pseudocódigo)

```
// Self-registration
class ServiceRegistration:
    registry
    
    function onStartup():
        registry.register(
            name: "order-service",
            host: getLocalIp(),
            port: 8080,
            healthUrl: "/health",
            metadata: { version: "2.1.0", region: "us-east-1" }
        )
        
        // Heartbeat periódico
        scheduler.every(30s, () -> registry.heartbeat(instanceId))
    
    function onShutdown():
        registry.deregister(instanceId)

// Client-side discovery
class OrderServiceClient:
    registry
    loadBalancer  // round-robin, random, least-connections
    
    function callOrderService(request):
        instances = registry.getHealthyInstances("order-service")
        instance = loadBalancer.choose(instances)
        return httpClient.call(instance.url + "/orders", request)
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| IPs hardcoded | Qualquer mudança de infra quebra | Use service discovery |
| Sem health check | Tráfego roteado para instâncias mortas | Implemente health + deregistration |
| Sem deregistration no shutdown | Instância fantasma no registry | Graceful shutdown com deregister |
| Cache de DNS sem TTL curto | Client usa IP stale por horas | TTL de 5-30s para DNS de serviços |
| Registry como single point of failure | Registry cai = discovery para | Use registry com replicação (cluster) |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Health Checks** | Alimentam o registry — instâncias unhealthy são removidas |
| **API Gateway** | Gateway usa discovery para encontrar backends |
| **Load Balancing** | Discovery fornece a lista de instâncias; LB escolhe qual chamar |
| **Sidecar** | Sidecar proxy pode incluir discovery transparente para o serviço |
| **Circuit Breaker** | CB por instância — se uma falha, remove do pool temporariamente |

---

## Boas Práticas

1. Em **Kubernetes**, use o DNS interno — discovery é nativo e transparente.
2. Fora do Kubernetes, use ferramentas como Consul, eureka ou cloud-native (AWS Cloud Map).
3. Implemente **health checking** rigoroso — instâncias unhealthy devem sair do pool rapidamente.
4. Use **graceful shutdown** — deregistre a instância antes de parar.
5. Use **TTL curto** em cache de DNS (5-30s).
6. O registry deve ser **altamente disponível** (cluster, replicação).
7. Prefira **server-side discovery** (proxy/mesh) para simplicidade dos clients.
8. Inclua **metadados** no registro (versão, região, canary) para roteamento inteligente.

---

## Referências

- Chris Richardson — *Microservices Patterns* (Manning) — Service Discovery
- Consul — [Documentation](https://www.consul.io/docs)
- Kubernetes — [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- Sam Newman — *Building Microservices* — Finding Things
