# Health Checks

> **Categoria:** Infraestrutura e Deploy
> **Complementa:** Service Discovery, Circuit Breaker, Blue-Green Deploy
> **Keywords:** health checks, liveness, readiness, startup probe, monitoramento, disponibilidade, auto-healing

---

## Problema

Como saber se um serviço está **realmente funcionando** e **pronto para receber tráfego**?

Serviço "no ar" ≠ serviço saudável. Um processo pode estar rodando mas:
- Banco de dados inacessível
- Pool de conexões esgotado
- Memória insuficiente
- Dependência externa indisponível
- Aplicação em estado inconsistente

Sem health checks, o load balancer/orchestrator continua enviando tráfego para instâncias doentes.

---

## Solução

Endpoints dedicados que reportam o **estado de saúde** do serviço. Orquestradores, load balancers e service registries consultam esses endpoints para tomar decisões automatizadas (remover do pool, reiniciar, não enviar tráfego).

---

## Tipos de Health Check

### 1. Liveness ("Estou vivo?")

Verifica se o **processo** está funcionando e não está travado (deadlock, infinite loop):

```
GET /health/live

Se a aplicação responder qualquer coisa → ALIVE
Se não responder (timeout) → DEAD → reiniciar o processo
```

| O que verifica | O que NÃO verifica |
|---------------|-------------------|
| Processo está rodando | Dependências (banco, cache) |
| Não está em deadlock | Capacidade de processar requests |
| Thread principal responsiva | Corretude dos dados |

**Ação em caso de falha:** **Restart** do processo/container.

### 2. Readiness ("Estou pronto para receber tráfego?")

Verifica se o serviço está **pronto** para processar requests:

```
GET /health/ready

Verifica:
  ✅ Banco de dados acessível
  ✅ Cache conectado
  ✅ Migrations executadas
  ✅ Warm-up concluído
  ✅ Dependências críticas respondendo

NOT READY:
  ❌ Pool de conexões esgotado → NÃO recebe tráfego (mas NÃO reinicia)
```

| O que verifica | Ação em caso de falha |
|---------------|----------------------|
| Dependências acessíveis | **Remove do pool** (não envia tráfego) |
| Capacidade de processar | NÃO reinicia (o serviço pode se recuperar) |
| Recursos disponíveis | Reinsere quando voltar a responder |

### 3. Startup ("Já inicializei?")

Verifica se o **processo de inicialização** completou (migrations, warm-up, cache loading):

```
GET /health/startup

Durante inicialização:
  → 503 Service Unavailable (não está pronto ainda)
  → Kubernetes NÃO reinicia e NÃO envia tráfego

Após inicialização completa:
  → 200 OK
  → Kubernetes começa a verificar liveness e readiness
```

**Uso:** Serviços com startup lento (cache warm-up, model loading, migration).

---

## Diagrama de Decisão

```
┌── Startup Check ──┐
│ Inicializou?      │
│                   │
│ NÃO → Aguarda    │
│ SIM → Continua   │
└─────────┬─────────┘
          │
          ▼
┌── Liveness Check ─┐     ┌── Readiness Check ─┐
│ Processo vivo?     │     │ Pronto p/ tráfego? │
│                    │     │                    │
│ NÃO → RESTART     │     │ NÃO → Remove do   │
│ SIM → OK          │     │        pool        │
└────────────────────┘     │ SIM → Envia        │
                           │        tráfego     │
                           └────────────────────┘
```

---

## O que verificar em cada Health Check

### Liveness (mínimo possível)

```
GET /health/live → 200 OK

// NÃO verificar dependências aqui!
// Se o banco cair e o liveness falhar, Kubernetes reinicia o serviço.
// Reiniciar não resolve banco fora do ar → restart loop infinito.
```

### Readiness (dependências)

```
GET /health/ready

Response (saudável):
{
  "status": "UP",
  "checks": {
    "database": { "status": "UP", "responseTime": "12ms" },
    "cache": { "status": "UP", "responseTime": "2ms" },
    "payment-service": { "status": "UP", "responseTime": "45ms" }
  }
}

Response (degradado):
{
  "status": "DOWN",
  "checks": {
    "database": { "status": "UP", "responseTime": "12ms" },
    "cache": { "status": "DOWN", "error": "Connection refused" },
    "payment-service": { "status": "UP", "responseTime": "45ms" }
  }
}
```

---

## Verificações Comuns

| Componente | O que verificar | Onde incluir |
|-----------|----------------|-------------|
| **Banco de dados** | Conexão ativa, query simples (`SELECT 1`) | Readiness |
| **Cache (Redis)** | Conexão ativa, `PING` | Readiness |
| **Message broker** | Conexão ativa, consumer group ativo | Readiness |
| **Disco** | Espaço disponível acima do threshold | Readiness |
| **Memória** | Heap/memory usage abaixo do limite | Liveness ou Readiness |
| **Dependência externa** | Responde dentro do timeout | Readiness |
| **Thread pool** | Threads disponíveis | Readiness |
| **Migrations** | Todas executadas | Startup |

---

## Health Check em Kubernetes

```
Configuração no Pod:

livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10      # espera antes de começar
  periodSeconds: 15             # verifica a cada 15s
  timeoutSeconds: 3             # timeout por verificação
  failureThreshold: 3           # 3 falhas → restart

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 2           # 2 falhas → remove do Service

startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  periodSeconds: 5
  failureThreshold: 30          # permite até 150s de startup
```

---

## Health Check e Load Balancer

```
Load Balancer verifica saúde:

  LB → GET /health → instance-1 → 200 ✅ (envia tráfego)
  LB → GET /health → instance-2 → 503 ❌ (remove do pool)
  LB → GET /health → instance-3 → timeout ❌ (remove do pool)
  
  ... após recuperação ...
  
  LB → GET /health → instance-2 → 200 ✅ (reinsere no pool)
```

---

## Exemplo Conceitual (Pseudocódigo)

```
class HealthCheckEndpoints:
    database
    cache
    startupComplete = false
    
    // Liveness — mínimo humano, sem verificar dependências
    function liveness():
        return Response(200, { "status": "UP" })
    
    // Startup — verifica se inicialização completou
    function startup():
        if startupComplete:
            return Response(200, { "status": "UP" })
        else:
            return Response(503, { "status": "STARTING" })
    
    // Readiness — verifica dependências
    function readiness():
        checks = {}
        allUp = true
        
        // Database
        try:
            database.execute("SELECT 1")
            checks["database"] = { status: "UP" }
        catch:
            checks["database"] = { status: "DOWN" }
            allUp = false
        
        // Cache
        try:
            cache.ping()
            checks["cache"] = { status: "UP" }
        catch:
            checks["cache"] = { status: "DOWN" }
            allUp = false
        
        status = allUp ? 200 : 503
        return Response(status, { 
            status: allUp ? "UP" : "DOWN",
            checks: checks
        })
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Verificar dependências no liveness | Banco cai → liveness falha → restart → banco ainda fora → restart loop | Liveness: só processo. Readiness: dependências. |
| Health check lento (query pesada) | Timeout no health check = instância marked as dead | Use verificações leves (`SELECT 1`, `PING`) |
| Sem health check | Tráfego enviado para instâncias mortas | Implemente liveness + readiness |
| Health check que sempre retorna 200 | Mascarado — serviço doente parece saudável | Verifique status real das dependências |
| Expor dados sensíveis no health | Detalhes internos expostos publicamente | Health público: status apenas. Health interno: detalhes. |
| Sem threshold de falha | Uma falha transitória marca como unhealthy | Use failureThreshold (2-3 falhas consecutivas) |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Service Discovery** | Registry usa health checks para saber quais instâncias estão UP |
| **Circuit Breaker** | CB e health check se complementam — ambos detectam falhas |
| **Blue-Green Deploy** | Health check valida que a nova versão está saudável antes de rotear tráfego |
| **Canary Release** | Health da canary é monitorado — rollback automático se unhealthy |
| **API Gateway** | Gateway usa health check para rotear apenas para backends saudáveis |

---

## Boas Práticas

1. **Liveness = mínimo** — só verificar se o processo responde. **NÃO** verificar dependências.
2. **Readiness = dependências** — verificar banco, cache, serviços críticos.
3. Use **startup probe** para serviços com inicialização lenta.
4. Health checks devem ser **rápidos** (<100ms). Queries leves, sem lógica pesada.
5. Use **failureThreshold** — não marque como unhealthy na primeira falha.
6. **Não exponha detalhes** internos em endpoints públicos (só status UP/DOWN).
7. Health checks **não devem ter side effects** — são leitura pura.
8. Em deploys (blue-green, canary), valide health antes de rotear tráfego para a nova versão.

---

## Referências

- Kubernetes — [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- Microsoft — [Health Endpoint Monitoring Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/health-endpoint-monitoring)
- 12-Factor App — [IX. Disposability](https://12factor.net/disposability)
