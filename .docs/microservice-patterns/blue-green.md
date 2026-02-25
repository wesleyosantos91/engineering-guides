# Blue-Green Deploy

> **Categoria:** Estratégias de Modernização e Deploy
> **Complementa:** Health Checks, Canary Release, Feature Flags
> **Keywords:** blue-green deploy, zero downtime, rollback instantâneo, ambiente duplicado, cutover, release segura

---

## Problema

Deploys tradicionais criam **downtime** ou **risco** durante a transição:

- **Rolling deploy** atualiza instâncias gradualmente, mas durante a transição coexistem duas versões
- Se a nova versão tiver bugs, o **rollback** é lento (precisa redesployar a versão anterior)
- Não há como **testar em produção** antes de expor ao tráfego real

---

## Solução

Manter **dois ambientes idênticos** (Blue e Green). Apenas um recebe tráfego por vez. O deploy é feito no ambiente inativo, validado, e o tráfego é trocado instantaneamente:

```
Estado inicial:
  ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
  │   Clients   │────────▶│  BLUE (v1)  │         │  GREEN (v1) │
  │             │   LB    │  ▪ ATIVO    │         │  ▪ INATIVO  │
  └─────────────┘         └─────────────┘         └─────────────┘

Deploy v2 no GREEN:
  ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
  │   Clients   │────────▶│  BLUE (v1)  │         │  GREEN (v2) │
  │             │   LB    │  ▪ ATIVO    │         │  ▪ Deploy   │
  └─────────────┘         └─────────────┘         │  ▪ Validar  │
                                                   └─────────────┘

Switch (instantâneo):
  ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
  │   Clients   │─────────────────────────────────▶│  GREEN (v2) │
  │             │   LB    │  BLUE (v1)  │         │  ▪ ATIVO    │
  └─────────────┘         │  ▪ INATIVO  │         └─────────────┘
                          │  (standby)  │
                          └─────────────┘
```

---

## Fluxo

```
1. Estado atual: BLUE ativo (v1), tráfego → BLUE

2. Deploy no GREEN:
   - Deploy v2 no ambiente GREEN
   - GREEN NÃO recebe tráfego real

3. Validação no GREEN:
   - Smoke tests no GREEN
   - Health checks verificam se GREEN está saudável
   - Testes de integração no GREEN

4. Switch:
   - Load balancer/DNS aponta para GREEN
   - Cliente agora atende pelo GREEN (v2)
   - Switch é instantâneo (mudança de ponteiro no LB)

5. Monitoramento:
   - Monitorar erros, latência, métricas no GREEN
   - Se OK → deploy completo. BLUE fica como standby.
   - Se problemas → Rollback: switch de volta para BLUE (instantâneo)

6. Próximo deploy:
   - Deploy v3 no BLUE (agora inativo)
   - Validar → Switch para BLUE
   - (alternância contínua entre BLUE ↔ GREEN)
```

---

## Vantagens

| Vantagem | Descrição |
|----------|-----------|
| **Zero downtime** | Tráfego muda instantaneamente — sem período de transição |
| **Rollback instantâneo** | Se a nova versão falhar, volta para o ambiente anterior em segundos |
| **Teste em produção** | Novo ambiente pode ser testado com infraestrutura real antes do switch |
| **Simplicidade** | Conceito simples — dois ambientes, troca de ponteiro |
| **Sem coexistência de versões** | Apenas uma versão atende por vez (evita incompatibilidades) |

---

## Desvantagens / Trade-offs

| Desvantagem | Impacto |
|-------------|---------|
| **Custo dobrado** | Dois ambientes completos rodando (pode otimizar com scale down do inativo) |
| **Banco de dados** | Se o schema mudou, ambas versões precisam ser compatíveis |
| **Sessões** | Sessões in-memory se perdem no switch |
| **Warmup** | Novo ambiente pode ter cold start (cache frio, connections não pool) |

---

## Migração de Banco de Dados

O maior desafio é quando a nova versão requer **mudança de schema**:

### Estratégia: Backward-Compatible Migrations

```
Versão v1: Usa coluna "name" (VARCHAR)
Versão v2: Quer dividir em "first_name" e "last_name"

Abordagem em 3 deploys:
  Deploy 1: Adiciona colunas novas (first_name, last_name) — v1 ignora
  Deploy 2: Switch para v2 que usa novas colunas (popula ambas)
  Deploy 3: Remove coluna antiga "name" (após confirmar que v2 é estável)
```

**Regra:** Migrations devem ser **backward-compatible** — a versão anterior deve continuar funcionando com o novo schema.

---

## Mecanismos de Switch

| Mecanismo | Como funciona | Velocidade |
|-----------|--------------|-----------|
| **Load Balancer** | Muda o target group/backend pool | Segundos |
| **DNS** | Muda o registro A/CNAME | Minutos (TTL) |
| **Kubernetes Service** | Muda o selector do Service | Segundos |
| **Feature Flag** | Flag controla qual ambiente atende | Instantâneo |

---

## Exemplo Conceitual (Pseudocódigo)

```
// Processo de deploy Blue-Green
class BlueGreenDeployment:
    loadBalancer
    environments = { "blue": Environment, "green": Environment }
    
    function deploy(newVersion):
        active = getActiveEnvironment()      // ex: "blue"
        inactive = getInactiveEnvironment()  // ex: "green"
        
        // 1. Deploy no ambiente inativo
        inactive.deploy(newVersion)
        
        // 2. Aguarda startup
        waitForHealthy(inactive, timeout: 5min)
        
        // 3. Smoke tests
        runSmokeTests(inactive.url)
        
        // 4. Switch
        loadBalancer.routeTrafficTo(inactive)
        log.info("Switch: {} → {}", active.name, inactive.name)
        
        // 5. Monitor
        monitor(inactive, duration: 15min)
        if errorsAboveThreshold():
            rollback(active)
    
    function rollback(previousEnvironment):
        loadBalancer.routeTrafficTo(previousEnvironment)
        log.warn("Rollback para {}", previousEnvironment.name)
        alert("Deploy rollback executado!")
```

---

## Blue-Green em Kubernetes

```
# Blue deployment (v1 — ativo)
Deployment: order-service-blue
  replicas: 3
  labels:
    app: order-service
    version: blue

# Green deployment (v2 — novo)
Deployment: order-service-green
  replicas: 3
  labels:
    app: order-service
    version: green

# Service aponta para BLUE
Service: order-service
  selector:
    app: order-service
    version: blue          ← SWITCH: trocar para "green"
```

Switch = mudar o `selector.version` no Service. Instantâneo.

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Schema migration não backward-compatible | Rollback impossível (schema incompatível) | Migrations devem ser backward-compatible |
| Sem health check antes do switch | Switch para ambiente com erros | Health check + smoke tests obrigatórios |
| Dois ambientes ativos ao mesmo tempo | Requests vão para ambos (inconsistência) | Apenas UM ambiente recebe tráfego |
| Sem monitoramento pós-switch | Problemas detectados tarde | Monitor ativo nos primeiros 15-30 min |
| Ambiente inativo sem recursos | Cold start lento, cache frio | Warm-up do ambiente antes do switch |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Canary Release** | Canary é mais gradual (10% → 50% → 100%); Blue-Green é tudo ou nada |
| **Health Checks** | Health check valida o ambiente antes do switch |
| **Feature Flags** | Flags podem ser o mecanismo de switch |
| **Strangler Fig** | Strangler migra funcionalidades; Blue-Green deploya versões |
| **API Gateway** | Gateway pode ser o ponto de switch (roteamento por versão) |

---

## Boas Práticas

1. **Health checks** no ambiente novo antes do switch.
2. **Smoke tests** automatizados no ambiente novo (CI/CD).
3. Migrations devem ser **backward-compatible** (expand-and-contract).
4. Monitore **métricas pós-switch** por pelo menos 15 minutos.
5. Tenha **rollback automatizado** se error rate ultrapassar threshold.
6. **Warm-up** do ambiente novo (cache, connection pool) antes do switch.
7. O **ambiente inativo** pode ter scale reduzido (economia), mas deve escalar rápido.
8. Em Kubernetes, use **labels** no selector do Service para switch instantâneo.

---

## Referências

- Martin Fowler — [Blue Green Deployment](https://martinfowler.com/bliki/BlueGreenDeployment.html)
- Microsoft — [Blue-Green Deployments](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/blue-green-spring/blue-green-spring)
- Jez Humble & David Farley — *Continuous Delivery* (Addison-Wesley)
