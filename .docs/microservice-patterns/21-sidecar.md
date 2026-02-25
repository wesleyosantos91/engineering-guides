# Sidecar Pattern

> **Categoria:** Infraestrutura e Deploy
> **Complementa:** Service Discovery, API Gateway, Observabilidade
> **Keywords:** sidecar, service mesh, proxy, cross-cutting concerns, infraestrutura desacoplada, Envoy

---

## Problema

Cross-cutting concerns (logging, métricas, networking, segurança, service discovery) precisam estar presentes em **cada microsserviço**. Implementar em cada um:

- **Duplicação** de código/configuração
- Serviços em **linguagens diferentes** precisam re-implementar a mesma lógica
- Atualizar o componente de networking exige **redeploy de todos** os serviços
- Mistura de **responsabilidades** — lógica de negócio + infraestrutura no mesmo processo

---

## Solução

Deployar um **processo auxiliar** (sidecar) junto com cada microsserviço. O sidecar roda no mesmo host/pod mas em um container/processo separado, lidando com cross-cutting concerns de forma **transparente** para o serviço principal.

```
┌── Pod / Host ──────────────────────────────┐
│                                            │
│  ┌──────────────┐    ┌───────────────────┐ │
│  │  Serviço     │◀──▶│  Sidecar          │ │
│  │  (lógica de  │    │  (proxy, logging, │ │
│  │  negócio)    │    │  metrics, mTLS,   │ │
│  │              │    │  service discovery)│ │
│  └──────────────┘    └───────────────────┘ │
│                                            │
└────────────────────────────────────────────┘
```

O serviço principal se comunica com o sidecar via **localhost** (comunicação local, sem rede). O sidecar intercepta tráfego de entrada e saída.

---

## Como Funciona

```
Tráfego de ENTRADA:
  Request externo → Sidecar (porta 15006) → processa (mTLS, auth, metrics) → Serviço (porta 8080)

Tráfego de SAÍDA:
  Serviço chama http://order-service:8080 → Sidecar intercepta → resolve DNS → 
  mTLS → routing → Sidecar do destino → Serviço destino
```

### Fluxo entre dois serviços com Sidecar

```
┌── Pod A ────────────────┐           ┌── Pod B ────────────────┐
│                         │           │                         │
│ Service A → Sidecar A ──┼───mTLS───┼──▶ Sidecar B → Service B│
│                         │           │                         │
└─────────────────────────┘           └─────────────────────────┘
```

1. Service A faz request para Service B (via localhost/sidecar)
2. Sidecar A intercepta, adiciona mTLS, headers de tracing, metrics
3. Sidecar A resolve o endereço de Service B (service discovery)
4. Sidecar B recebe, valida mTLS, registra métricas
5. Sidecar B encaminha para Service B
6. Service B processa e responde

---

## Responsabilidades do Sidecar

| Responsabilidade | Descrição |
|-----------------|-----------|
| **Proxy de rede** | Intercepta todo tráfego in/out do serviço |
| **mTLS** | Criptografia entre serviços (mutual TLS) transparente |
| **Service Discovery** | Resolve nomes de serviço para IPs |
| **Load Balancing** | Distribui requests entre instâncias do destino |
| **Circuit Breaker** | Protege contra falhas cascata |
| **Retry** | Retry automático com backoff |
| **Timeout** | Timeout configurável por rota/destino |
| **Rate Limiting** | Limita requests por serviço/rota |
| **Observabilidade** | Coleta métricas, traces distribuídos, access logs |
| **Traffic Management** | Canary, blue-green, traffic splitting |

---

## Service Mesh

Quando **todos** os serviços têm sidecars, e há um **control plane** central que configura todos os sidecars, temos um **Service Mesh**:

```
┌── Control Plane ────────────────────────┐
│  (Configuração, Política, Certificados) │
│                                         │
│  Configura todos os sidecars            │
└──────────┬────────────┬─────────────────┘
           │            │
    ┌──────▼──────┐  ┌──▼──────────────┐
    │  Pod A      │  │  Pod B          │
    │             │  │                 │
    │ Svc ←→ Proxy│  │ Svc ←→ Proxy   │   ← Data Plane
    │             │  │                 │     (múltiplos sidecars)
    └─────────────┘  └────────────────┘
```

| Componente | Papel |
|-----------|-------|
| **Data Plane** | Conjunto de todos os sidecar proxies (processam tráfego) |
| **Control Plane** | Configura os proxies: policies, routing rules, certificates |

---

## Sidecar sem Service Mesh

Sidecars podem ser usados sem um service mesh completo, para cenários específicos:

```
Log Sidecar:
  ┌── Pod ────────────────────┐
  │ Service → logs → arquivo  │
  │                   ↓       │
  │ Log Sidecar → lê arquivo  │
  │            → envia para    │
  │              Elasticsearch │
  └───────────────────────────┘

Config Sidecar:
  ┌── Pod ────────────────────┐
  │ Service → lê config local │
  │                   ↑       │
  │ Config Sidecar →  │       │
  │   busca config no │       │
  │   Consul/Vault    │       │
  │   atualiza local  │       │
  └───────────────────┘
```

---

## Vantagens do Sidecar

| Vantagem | Explicação |
|----------|-----------|
| **Separação de concerns** | Serviço foca em negócio; sidecar em infra |
| **Polyglot** | Serviço em qualquer linguagem — sidecar é o mesmo para todos |
| **Atualização independente** | Atualiza sidecar sem recompilar o serviço |
| **Consistência** | Mesma política de segurança/retry/metrics em todos os serviços |
| **Transparência** | Serviço não precisa saber sobre mTLS, retry, tracing |

---

## Desvantagens / Trade-offs

| Desvantagem | Impacto |
|-------------|---------|
| **Latência** | Hop adicional (microsegundos a milissegundos por request) |
| **Recursos** | Cada sidecar consome CPU e memória (multiplicado por Nº de pods) |
| **Complexidade operacional** | Mais um componente para monitorar, versionar, debugar |
| **Debugging** | Problemas de rede podem estar no sidecar, não no serviço |
| **Overhead em scale** | 100 pods = 100 sidecars = 100x recursos adicionais |

---

## Quando usar vs. Quando NÃO usar

| Usar | NÃO usar |
|------|----------|
| Múltiplos serviços em linguagens diferentes | Poucos serviços (1-3) — latência do sidecar não compensa |
| Precisa de mTLS entre todos os serviços | Monolito modular — não precisa de service mesh |
| Precisa de observabilidade uniforme | Ambiente com recursos muito limitados |
| Traffic management complexo (canary, blue-green) | Overhead do sidecar é inaceitável |
| Equipe tem capacidade de operar service mesh | Equipe pequena sem experiência em mesh |

---

## Exemplo Conceitual (Pseudocódigo)

```
// Sem sidecar — serviço implementa tudo
class OrderService:
    function createOrder(request):
        // Autenticação
        jwt = validateJWT(request.headers.authorization)
        
        // Service Discovery
        paymentUrl = serviceRegistry.resolve("payment-service")
        
        // Circuit Breaker + Retry + Timeout
        response = circuitBreaker.execute(() ->
            retry(3, () ->
                httpClient.post(paymentUrl + "/charge", payload, timeout: 3s)
            )
        )
        
        // Métricas
        metrics.record("payment.call.duration", response.time)
        metrics.increment("payment.call.total")
        
        // Logging
        log.info("Payment charged for order {}", orderId)
        
        // Tracing
        tracer.addSpan("payment.charge", response.time)
        
        // ...lógica de negócio...

// COM sidecar — serviço só faz negócio
class OrderService:
    function createOrder(request):
        // Chama payment-service via localhost (sidecar intercepta)
        response = httpClient.post(
            "http://payment-service:8080/charge",   // sidecar resolve
            payload
        )
        
        // Sidecar cuida de: mTLS, retry, CB, timeout, discovery, 
        //                    metrics, tracing, logging
        
        // ...lógica de negócio...
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Sidecar com lógica de negócio | Sidecar vira parte do serviço — acoplamento | Sidecar: só cross-cutting concerns |
| Sidecar para tudo (overkill) | 3 serviços não justificam mesh | Use sidecar quando tiver múltiplos serviços |
| Ignorar overhead de recursos | 200 sidecars consomem CPU/memória significativos | Dimensione e monitore recursos dos sidecars |
| Não monitorar o sidecar | Problema no sidecar parece problema no serviço | Monitore o sidecar separadamente |
| Deploying sidecar e serviço em versões incompatíveis | Comportamento inesperado | Versione e teste sidecar + serviço juntos |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **API Gateway** | Gateway: ponto de entrada externo. Sidecar: comunicação interna (East-West). |
| **Service Discovery** | Sidecar inclui discovery transparente |
| **Circuit Breaker** | Sidecar implementa CB sem código no serviço |
| **Observabilidade** | Sidecar coleta métricas e traces automaticamente |
| **mTLS/Segurança** | Sidecar gerencia certificados e criptografia |
| **Canary Release** | Service mesh roteia % de tráfego via sidecars |

---

## Boas Práticas

1. Use sidecar para **cross-cutting concerns** — nunca para lógica de negócio.
2. Considere **service mesh** quando tiver >5-10 serviços com requisitos de segurança/observabilidade.
3. Monitore o **overhead** dos sidecars (CPU, memória, latência adicionada).
4. Mantenha sidecars **atualizados** — são parte da infraestrutura de segurança.
5. Em **Kubernetes**, use init-containers ou injection automática para deploy de sidecars.
6. Comece **sem** service mesh; adicione quando a complexidade justificar.
7. O serviço deve funcionar **sem** o sidecar (fallback) — pelo menos para debugging local.
8. Dimensione o **resource request/limit** dos sidecars separadamente.

---

## Referências

- Microsoft — [Sidecar Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/sidecar)
- Istio — [Architecture](https://istio.io/latest/docs/ops/deployment/architecture/)
- Envoy Proxy — [What is Envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/what_is_envoy)
- William Morgan — [What is a Service Mesh?](https://linkerd.io/what-is-a-service-mesh/)
