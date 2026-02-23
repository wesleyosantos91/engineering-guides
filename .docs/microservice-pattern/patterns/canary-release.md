# Canary Release

> **Categoria:** Modernização e Deploy Progressivo
> **Complementa:** Blue-Green Deploy, Feature Flags, Health Checks, API Gateway

---

## Problema

Mesmo com testes automatizados, bugs **só aparecem em produção** com tráfego real:

- Edge cases que testes não cobrem
- Problemas de escala (funciona com 10 users, falha com 10.000)
- Incompatibilidade com dados reais
- Problemas de infraestrutura (latência de rede, DNS, etc.)

Deployar para **100% dos usuários** de uma vez amplifica o risco: se a versão tem bug, **todos** são afetados.

---

## Solução

Deployar a nova versão para uma **pequena parcela** do tráfego (canary) e monitorar. Se as métricas forem boas, aumentar gradualmente até 100%. Se houver problemas, fazer rollback rápido — apenas a parcela pequena foi afetada.

O nome vem do **canário nas minas de carvão**: o canário detecta perigo antes dos mineradores.

---

## Fluxo

```
Fase 1 — Deploy Canary (5% do tráfego):
  ┌─────────┐         ┌──────────────┐
  │         │── 95% ─▶│   v1 Stable  │  (3 instâncias)
  │ Clients │         └──────────────┘
  │         │── 5% ──▶┌──────────────┐
  │         │         │   v2 Canary  │  (1 instância)
  └─────────┘         └──────────────┘

Fase 2 — Aumentar para 25%:
  ┌─────────┐         ┌──────────────┐
  │         │── 75% ─▶│   v1 Stable  │
  │ Clients │         └──────────────┘
  │         │── 25% ─▶┌──────────────┐
  │         │         │   v2 Canary  │
  └─────────┘         └──────────────┘

Fase 3 — Aumentar para 50%, depois 100%:
  ... gradual até ...
  
  ┌─────────┐         ┌──────────────┐
  │ Clients │── 100% ▶│   v2 (new)   │  ← Canary promovido!
  └─────────┘         └──────────────┘
```

---

## Fases Típicas

| Fase | % Tráfego para Canary | Duração | Verificação |
|------|-----------------------|---------|-------------|
| 1 | 1–5% | 15–30 min | Error rate, latência, logs |
| 2 | 10–25% | 30–60 min | Métricas de negócio |
| 3 | 50% | 1–2 horas | Comparação completa |
| 4 | 100% | — | Canary promovido, stable removido |

**Rollback:** Em qualquer fase, se métricas degradarem → rollback = 0% para canary.

---

## Métricas para Decisão

| Métrica | Threshold para Rollback |
|---------|------------------------|
| **Error rate** | Canary > 2× error rate do stable |
| **Latência p99** | Canary p99 > 2× stable p99 |
| **Success rate** | Canary < 99% (ou threshold do SLO) |
| **Memory/CPU** | Canary consumindo muito mais recursos |
| **Métricas de negócio** | Conversão, checkout, signup visivelmente menor |

### Análise Automatizada

```
function evaluateCanary(canaryMetrics, stableMetrics):
    // Error rate comparison
    if canaryMetrics.errorRate > stableMetrics.errorRate × 2:
        return ROLLBACK
    
    // Latência comparison
    if canaryMetrics.p99Latency > stableMetrics.p99Latency × 1.5:
        return ROLLBACK
    
    // Success rate
    if canaryMetrics.successRate < SLO_TARGET:
        return ROLLBACK
    
    return PROMOTE  // canary está saudável, avançar para próxima fase
```

---

## Estratégias de Roteamento

### 1. Porcentagem de Tráfego

O load balancer/gateway envia X% para canary:

```
Gateway:
  route: /api/**
    - weight: 95% → v1-stable
    - weight: 5%  → v2-canary
```

### 2. Por Usuários Específicos

Canary apenas para usuários internos, beta testers ou por user ID:

```
if user.id IN canaryUsers OR user.role == "BETA_TESTER":
    route → v2-canary
else:
    route → v1-stable
```

### 3. Por Região/Segmento

```
if user.region == "us-east-1":
    route → v2-canary (região piloto)
else:
    route → v1-stable
```

### 4. Por Header

```
Request com header X-Canary: true → v2-canary
Request sem header → v1-stable
```

---

## Canary Automatizado (Progressive Delivery)

Ferramentas de **progressive delivery** automatizam todo o ciclo:

```
1. Deploy canary (5%)
2. Espera 10 min
3. Coleta métricas (Prometheus, Datadog)
4. Compara canary vs. stable
5. Se OK → aumenta para 25%
6. Espera 10 min
7. Compara novamente
8. Se OK → 50% → 100% (promote)
9. Se NOT OK em qualquer etapa → Rollback automático
```

---

## Canary vs. Blue-Green

| Aspecto | Canary | Blue-Green |
|---------|--------|-----------|
| **Exposição ao risco** | Gradual (1% → 5% → 25% → 100%) | Tudo ou nada (0% → 100%) |
| **Rollback** | Remove canary (1-5% afetados) | Switch de volta (100% afetados temporariamente) |
| **Validação** | Com tráfego real, em produção | Com testes no ambiente inativo |
| **Coexistência de versões** | Sim (duas versões simultâneas) | Não (uma por vez) |
| **Complexidade** | Maior (roteamento por peso) | Menor (switch de ponteiro) |
| **Banco de dados** | Ambas versões acessam mesmo banco | Pode ter banco separado |

---

## Exemplo Conceitual (Pseudocódigo)

```
class CanaryDeployment:
    gateway
    metricsCollector
    phases = [
        Phase(weight: 5,  duration: 15min),
        Phase(weight: 25, duration: 30min),
        Phase(weight: 50, duration: 60min),
        Phase(weight: 100, duration: 0)     // promote
    ]
    
    function deploy(newVersion):
        deployCanaryInstances(newVersion)
        
        for phase in phases:
            // Ajusta peso no gateway
            gateway.setCanaryWeight(phase.weight)
            log.info("Canary em {}% do tráfego", phase.weight)
            
            // Espera período de observação
            wait(phase.duration)
            
            // Coleta e compara métricas
            canaryMetrics = metricsCollector.get("v2-canary")
            stableMetrics = metricsCollector.get("v1-stable")
            
            decision = evaluate(canaryMetrics, stableMetrics)
            
            if decision == ROLLBACK:
                gateway.setCanaryWeight(0)
                removeCanaryInstances()
                alert("Canary rollback! Métricas degradaram.")
                return FAILED
        
        // Todas as fases OK → promote
        promoteCanaryToStable()
        return SUCCESS
    
    function evaluate(canary, stable):
        if canary.errorRate > stable.errorRate × 2:
            return ROLLBACK
        if canary.p99 > stable.p99 × 1.5:
            return ROLLBACK
        return CONTINUE
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Canary sem monitoramento | Não sabe se canary está saudável | Métricas automatizadas + alertas |
| Canary com 50% de cara | Metade dos usuários afetados se houver bug | Comece com 1-5% |
| Sem rollback automatizado | Demora para reverter quando problemas surgem | Rollback automático baseado em métricas |
| Canary muito curto | Bugs que aparecem após 30min não são detectados | Mínimo 15-30 min por fase |
| Ignorar métricas de negócio | Canary saudável tecnicamente mas conversão caiu | Monitore métricas de negócio também |
| Sessões sticky no canary | Mesmo usuário sempre no canary — masking de problemas | Distribuição aleatória por request |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Blue-Green** | Blue-Green: switch total. Canary: gradual. Podem ser combinados. |
| **Feature Flags** | Flags podem ser o mecanismo de roteamento do canary |
| **Health Checks** | Health do canary é monitorado continuamente |
| **API Gateway** | Gateway implementa roteamento por peso |
| **Shadow Deploy** | Shadow valida sem afetar users; Canary afeta % dos users |

---

## Boas Práticas

1. Comece com **1-5%** do tráfego; nunca mais que 10% na primeira fase.
2. Defina **métricas de sucesso** antes do deploy (error rate, latência, negócio).
3. Implemente **rollback automático** quando métricas degradarem.
4. Cada fase deve ter duração suficiente (**mínimo 15 min**) para detectar problemas.
5. Monitore **métricas de negócio** além das técnicas.
6. Use **progressive delivery tools** para automação completa.
7. O banco de dados deve ser **compatível** com ambas as versões simultaneamente.
8. **Documente** o critério de promote/rollback para cada serviço.

---

## Referências

- Danilo Sato — *Canary Releases* (martinfowler.com)
- Microsoft — [Canary deployment strategy](https://learn.microsoft.com/en-us/azure/architecture/framework/mission-critical/mission-critical-deployment-testing#canary-deployment)
- Flagger — [Progressive Delivery](https://flagger.app/)
- Argo Rollouts — [Canary Strategy](https://argo-rollouts.readthedocs.io/en/stable/features/canary/)
