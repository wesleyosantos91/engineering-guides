# Observabilidade Avançada — Observability-Driven Development

> **Objetivo deste documento:** Servir como referência completa sobre **Observability-Driven Development (ODD)** — a prática de desenvolver software com observabilidade como cidadã de primeira classe, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: ODD principles, testing in production, progressive delivery, chaos engineering, incident management, on-call.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **ODD** | Observabilidade é parte do design, não afterthought | Instrumentar só depois que problemas aparecem | Definir SLIs/spans DURANTE o design da feature |
| **Feature flags** | Toda feature nova nasce com flag + métricas | Deploy = release (tudo junto) | Flag off → deploy → observe → flag on gradual |
| **Canary** | SLO como gate automático | Canary manual com "looks good to me" | SLO-gated canary com rollback automático |
| **Chaos** | Falhas devem validar hipóteses | "Vamos ver o que acontece" | Hipótese → experimento → observar → concluir |
| **Incidents** | Blameless, focado em sistema | "Quem causou?" | "Que condição permitiu?" |
| **Post-mortems** | Ações com owners e deadlines | Post-mortem sem follow-up | Template com action items + revisão em 30d |

---

## Sumário

- [Observabilidade Avançada — Observability-Driven Development](#observabilidade-avançada--observability-driven-development)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que é Observability-Driven Development](#o-que-é-observability-driven-development)
  - [ODD no Ciclo de Desenvolvimento](#odd-no-ciclo-de-desenvolvimento)
  - [Feature Flags + Observabilidade](#feature-flags--observabilidade)
  - [Progressive Delivery](#progressive-delivery)
  - [Testing in Production](#testing-in-production)
  - [Chaos Engineering + Observabilidade](#chaos-engineering--observabilidade)
  - [Deploy Strategies com Observability Gates](#deploy-strategies-com-observability-gates)
  - [Incident Management](#incident-management)
  - [Post-Mortem / Incident Review](#post-mortem--incident-review)
  - [On-Call Practices](#on-call-practices)
  - [Observability como Cultura](#observability-como-cultura)
  - [Anti-Patterns ODD](#anti-patterns-odd)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que é Observability-Driven Development

```
┌─────────────────────────────────────────────────────────────────┐
│           OBSERVABILITY-DRIVEN DEVELOPMENT (ODD)                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DEFINIÇÃO:                                                      │
│  Prática de projetar, desenvolver e operar software              │
│  com observabilidade como requisito de primeira classe —         │
│  não como afterthought.                                          │
│                                                                  │
│  PRINCÍPIOS:                                                     │
│                                                                  │
│  1. INSTRUMENTE DURANTE O DESIGN, NÃO DEPOIS                    │
│     • SLIs definidos na user story                               │
│     • Spans planejados junto com API design                      │
│     • Dashboards como "acceptance criteria"                      │
│                                                                  │
│  2. DEPLOY ≠ RELEASE                                             │
│     • Deploy: código no production                               │
│     • Release: feature visível para usuários                     │
│     • Feature flags controlam release, observabilidade valida    │
│                                                                  │
│  3. EACH FEATURE IS A HYPOTHESIS                                 │
│     • "Acreditamos que X vai melhorar Y"                         │
│     • Instrumentação mede o impacto                              │
│     • Dados validam ou invalidam                                 │
│                                                                  │
│  4. PRODUCTION É O ENVIRONMENT QUE IMPORTA                       │
│     • Staging nunca reproduz production fielmente                │
│     • Test in production com safety nets                         │
│     • Observabilidade é a safety net                             │
│                                                                  │
│  5. MEAN TIME TO UNDERSTANDING > MEAN TIME TO RESOLUTION         │
│     • Entender rápido é mais importante que fixar rápido         │
│     • Observabilidade reduz MTTU → reduz MTTR                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### ODD vs Desenvolvimento Tradicional

| Aspecto | Tradicional | ODD |
|---------|------------|-----|
| **Instrumentação** | Depois do deploy, quando problemas surgem | Durante o design da feature |
| **SLIs** | Definidos pela equipe de ops | Definidos pelo time de produto + eng |
| **Deploys** | Big bang, pray everything works | Canary → observe → expand → full |
| **Features** | Deploy = release | Deploy ≠ release (feature flags) |
| **Bugs** | Reproduzir localmente | Investigar em production com traces |
| **Testing** | 100% pré-produção (unit/integration) | Pre-prod + test-in-production |
| **Alertas** | Baseados em causa (CPU > 80%) | Baseados em sintoma (SLO breach) |
| **Falhas** | Evitar a todo custo | Esperadas, planejadas, praticadas |
| **Post-mortems** | Blame, buscar culpado | Blameless, buscar condições sistêmicas |

---

## ODD no Ciclo de Desenvolvimento

### Workflow por fase

```
┌─────────────────────────────────────────────────────────────────┐
│            ODD DEVELOPMENT LIFECYCLE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐                                                │
│  │   1. DESIGN  │  → Definir SLIs da feature                    │
│  │              │  → Planejar spans e attributes                │
│  │              │  → Definir success metrics                     │
│  │              │  → Dashboard mock/wireframe                    │
│  └──────┬───────┘                                                │
│         │                                                        │
│  ┌──────▼───────┐                                                │
│  │   2. BUILD   │  → Code + instrumentação juntos               │
│  │              │  → Spans custom para lógica de negócio        │
│  │              │  → Métricas de negócio (counters, histograms) │
│  │              │  → Feature flag wrapping                       │
│  └──────┬───────┘                                                │
│         │                                                        │
│  ┌──────▼───────┐                                                │
│  │  3. VERIFY   │  → Testes validam instrumentação              │
│  │              │  → Span assertions no integration test        │
│  │              │  → Metric assertions                          │
│  │              │  → "Consigo ver o que preciso?"               │
│  └──────┬───────┘                                                │
│         │                                                        │
│  ┌──────▼───────┐                                                │
│  │  4. DEPLOY   │  → Flag OFF, código em production            │
│  │              │  → Verificar: spans chegando? métricas ok?    │
│  │              │  → Deploy ≠ Release                           │
│  └──────┬───────┘                                                │
│         │                                                        │
│  ┌──────▼───────┐                                                │
│  │  5. RELEASE  │  → Flag ON gradual (1% → 10% → 50% → 100%) │
│  │              │  → Observar SLIs a cada step                  │
│  │              │  → Canary comparison: old vs new              │
│  │              │  → Rollback automático se SLO breach          │
│  └──────┬───────┘                                                │
│         │                                                        │
│  ┌──────▼───────┐                                                │
│  │  6. OBSERVE  │  → Dashboard mostra impacto real             │
│  │              │  → Hipótese validada? (feature metrics)       │
│  │              │  → SLIs impactados positiva/negativamente?     │
│  │              │  → Aprender, iterar                            │
│  └──────────────┘                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Observability Design Doc — Template

```markdown
## Feature: [Nome da Feature]

### Hipótese
Acreditamos que [mudança X] vai [melhorar métrica Y] para [segmento Z].

### SLIs impactados
| SLI | Baseline atual | Target esperado | Medição |
|-----|---------------|----------------|---------|
| Latency p99 do checkout | 450ms | ≤ 400ms | histogram `checkout.duration` |
| Error rate do payment | 0.3% | ≤ 0.2% | counter `payment.errors / payment.total` |

### Instrumentação planejada

**Spans:**
- `feature.new_payment_flow` (root)
  - `feature.validate_card` (attribute: `card.type`)
  - `feature.process_charge` (attribute: `provider`, `amount_cents`)

**Métricas:**
- `feature.new_flow.adoption_rate` (gauge: % de users no novo flow)
- `feature.new_flow.conversion_rate` (counter: completions / starts)
- `feature.new_flow.error_rate` (counter: errors por tipo)

**Feature Flag:**
- Flag: `enable-new-payment-flow`
- Rollout plan: 1% → 5% → 20% → 50% → 100% (1 semana cada)
- Rollback trigger: p99 > 500ms OR error_rate > 0.5%

### Dashboard
- [Link para dashboard mock/wireframe]
- Panels: adoption rate, latency comparison, error comparison, conversion
```

---

## Feature Flags + Observabilidade

### Anatomia de um feature flag observável

```python
# Pseudocode — Feature flag com observabilidade integrada

from opentelemetry import trace, metrics

tracer = trace.get_tracer("checkout-service")
meter = metrics.get_meter("checkout-service")

flag_counter = meter.create_counter("feature_flag.evaluation.total")
flag_duration = meter.create_histogram("feature_flag.variant.duration_ms")

def process_checkout(order, user):
    # Avaliar flag
    variant = feature_flags.evaluate("new-checkout-flow", user)
    
    flag_counter.add(1, {
        "flag.name": "new-checkout-flow",
        "flag.variant": variant,  # "control" ou "treatment"
        "user.tier": user.tier
    })
    
    with tracer.start_as_current_span("checkout.process") as span:
        span.set_attribute("feature_flag.name", "new-checkout-flow")
        span.set_attribute("feature_flag.variant", variant)
        span.set_attribute("user.tier", user.tier)
        
        start = time.monotonic()
        
        if variant == "treatment":
            result = new_checkout_flow(order, user)
        else:
            result = legacy_checkout_flow(order, user)
        
        duration_ms = (time.monotonic() - start) * 1000
        flag_duration.record(duration_ms, {
            "flag.name": "new-checkout-flow",
            "flag.variant": variant
        })
        
        span.set_attribute("checkout.result", result.status)
        return result
```

### Métricas essenciais por feature flag

```
PARA CADA FLAG ATIVA, MEÇA:

1. ADOPTION
   feature_flag.evaluation.total{variant="treatment"} / total

2. PERFORMANCE COMPARISON
   flag_duration{variant="treatment"} vs flag_duration{variant="control"}
   (p50, p95, p99)

3. ERROR COMPARISON  
   errors{variant="treatment"} / evaluations{variant="treatment"}
   vs
   errors{variant="control"} / evaluations{variant="control"}

4. BUSINESS METRICS
   conversion{variant="treatment"} vs conversion{variant="control"}
   revenue{variant="treatment"} vs revenue{variant="control"}

DASHBOARD LAYOUT:
┌─────────────────────┬─────────────────────┐
│ Adoption Rate (%)    │ Error Rate by Variant│
│  treatment: 20%      │  ██ control: 0.3%    │
│  control: 80%        │  ██ treatment: 0.2%  │
├─────────────────────┼─────────────────────┤
│ Latency by Variant   │ Conversion by Variant│
│  p50  p95  p99       │  control: 3.2%       │
│  C: 100 250 450ms    │  treatment: 3.8%     │
│  T:  80 200 380ms    │  (+18.7%)            │
├─────────────────────┴─────────────────────┤
│ Rollout Timeline                           │
│  ━━━━━━━━━━━━━━━━┯━━━┯━━━━━┯━━━━━━━━━━━  │
│  1%             5%  20%  50%        100%   │
│  ▲ (current)                                │
└────────────────────────────────────────────┘
```

---

## Progressive Delivery

### Canary Deployment com SLO Gates

```
┌─────────────────────────────────────────────────────────────────┐
│         CANARY DEPLOYMENT — SLO-GATED                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STEP 1: Deploy canary (5% traffic)                              │
│  ┌───────────────────────────────────────────────┐              │
│  │  Stable (v1) ██████████████████████████ 95%    │              │
│  │  Canary (v2) ██                          5%    │              │
│  └───────────────────────────────────────────────┘              │
│  Gate: Observar 10min. SLO p99 < 500ms? Error < 1%?            │
│  ✅ Pass → Step 2     ❌ Fail → Rollback automático             │
│                                                                  │
│  STEP 2: Expand (20% traffic)                                    │
│  ┌───────────────────────────────────────────────┐              │
│  │  Stable (v1) ████████████████████████   80%    │              │
│  │  Canary (v2) ██████                     20%    │              │
│  └───────────────────────────────────────────────┘              │
│  Gate: Observar 15min. SLOs mantidos?                            │
│  ✅ Pass → Step 3     ❌ Fail → Rollback automático             │
│                                                                  │
│  STEP 3: Expand (50% traffic)                                    │
│  ┌───────────────────────────────────────────────┐              │
│  │  Stable (v1) █████████████              50%    │              │
│  │  Canary (v2) █████████████              50%    │              │
│  └───────────────────────────────────────────────┘              │
│  Gate: Observar 30min. SLOs mantidos?                            │
│  ✅ Pass → Full rollout  ❌ Fail → Rollback                     │
│                                                                  │
│  STEP 4: Full rollout (100%)                                     │
│  ┌───────────────────────────────────────────────┐              │
│  │  New (v2) ███████████████████████████████ 100%  │              │
│  └───────────────────────────────────────────────┘              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### SLO Gate — Implementação

```yaml
# Pseudocode — Canary analysis config (Flagger/Argo Rollouts style)
apiVersion: rollout/v1
kind: CanaryAnalysis
spec:
  canary:
    steps:
      - setWeight: 5
      - pause: { duration: 10m }
      - analysis:
          templates:
            - name: slo-gate
          args:
            - name: service
              value: order-service
            - name: namespace
              value: production

      - setWeight: 20
      - pause: { duration: 15m }
      - analysis:
          templates:
            - name: slo-gate

      - setWeight: 50
      - pause: { duration: 30m }
      - analysis:
          templates:
            - name: slo-gate

      - setWeight: 100

  analysis:
    templates:
      - name: slo-gate
        metrics:
          - name: error-rate
            provider: prometheus
            query: |
              sum(rate(http_requests_total{
                service="{{ args.service }}",
                status=~"5..",
                canary="true"
              }[5m]))
              /
              sum(rate(http_requests_total{
                service="{{ args.service }}",
                canary="true"
              }[5m]))
            thresholdRange:
              max: 0.01  # < 1% error rate
            interval: 1m
            count: 5
            failureLimit: 2

          - name: latency-p99
            provider: prometheus
            query: |
              histogram_quantile(0.99,
                sum(rate(http_request_duration_seconds_bucket{
                  service="{{ args.service }}",
                  canary="true"
                }[5m])) by (le)
              )
            thresholdRange:
              max: 0.5  # < 500ms p99
            interval: 1m
            count: 5
            failureLimit: 2

        rollbackPolicy: automatic
```

---

## Testing in Production

### Categorias de teste em produção

```
┌─────────────────────────────────────────────────────────────────┐
│          TESTING IN PRODUCTION — SPECTRUM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RISCO BAIXO ◁──────────────────────────▷ RISCO ALTO            │
│                                                                  │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌──────────────┐     │
│  │ Observar │  │ Canary   │  │ Shadow  │  │ Chaos        │     │
│  │ & Alertar│  │ Deploy   │  │ Traffic │  │ Engineering  │     │
│  │          │  │          │  │         │  │              │     │
│  │ Read-only│  │ % users  │  │ Dual    │  │ Inject       │     │
│  │ monitoring│ │ expostos │  │ request │  │ failures     │     │
│  │ sintético │  │ gradual  │  │ compare │  │ in prod      │     │
│  └─────────┘  └──────────┘  └─────────┘  └──────────────┘     │
│                                                                  │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌──────────────┐     │
│  │Synthetic │  │ Feature  │  │Load Test│  │ Game Days    │     │
│  │Monitoring│  │ Flags    │  │in Prod  │  │              │     │
│  │          │  │          │  │         │  │ Team-wide    │     │
│  │Smoke test│  │Controlled│  │ Real    │  │ failure      │     │
│  │contínuo  │  │ rollout  │  │ infra   │  │ simulation   │     │
│  └─────────┘  └──────────┘  └─────────┘  └──────────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Synthetic Monitoring

```python
# Pseudocode — Synthetic monitoring com observabilidade

from opentelemetry import trace

tracer = trace.get_tracer("synthetic-monitor")

def synthetic_checkout_flow():
    """Executa a cada 1 minuto em produção."""
    with tracer.start_as_current_span("synthetic.checkout_flow") as span:
        span.set_attribute("synthetic", True)
        span.set_attribute("test.type", "smoke")
        
        # Step 1: Buscar produto
        with tracer.start_as_current_span("synthetic.get_product"):
            product = api_client.get("/api/v1/products/test-product-123")
            assert product.status_code == 200
        
        # Step 2: Adicionar ao carrinho
        with tracer.start_as_current_span("synthetic.add_to_cart"):
            cart = api_client.post("/api/v1/cart", {
                "product_id": "test-product-123",
                "quantity": 1,
                "synthetic": True  # Backend ignora em métricas de negócio
            })
            assert cart.status_code == 201
        
        # Step 3: Checkout (com payment sandbox)
        with tracer.start_as_current_span("synthetic.process_checkout"):
            result = api_client.post("/api/v1/checkout", {
                "cart_id": cart.json()["id"],
                "payment_method": "test_card",
                "synthetic": True
            })
            assert result.status_code == 200
            assert result.json()["status"] == "completed"
        
        span.set_attribute("synthetic.result", "pass")

# REGRAS PARA SYNTHETIC:
# ✅ Marcar com attribute "synthetic: true"
# ✅ Filtrar de SLIs reais (não conta como user traffic)
# ✅ Backend deve reconhecer e não cobrar/converter
# ✅ Cleanup automático dos dados de teste
# ❌ Nunca enviar email/SMS real
# ❌ Nunca cobrar payment real
```

### Shadow Traffic / Dark Launch

```
SHADOW TRAFFIC:
  Request real → Serviço estável (responde ao user)
                → Serviço novo (resposta descartada, apenas observada)

┌────────┐     ┌──────────────┐     ┌──────────┐
│ Client │ ──▶ │   Router /   │ ──▶ │ Stable   │ ──▶ Response
│        │     │   Load Bal.  │     │ (v1)     │     ao user
└────────┘     │              │     └──────────┘
               │              │     ┌──────────┐
               │              │ ──▶ │ Shadow   │ ──▶ Response
               │              │     │ (v2)     │     descartada
               └──────────────┘     └──────────┘
                                         │
                                    Comparar:
                                    • Latency
                                    • Errors
                                    • Response diff

QUANDO USAR:
• Migração de sistema grande (rewrite)
• Novo algoritmo de ranking/recomendação
• Database migration (dual-write + compare)
• Risco alto, precisa validar sem impactar users
```

---

## Chaos Engineering + Observabilidade

### Chaos Engineering Loop

```
┌─────────────────────────────────────────────────────────────────┐
│           CHAOS ENGINEERING LOOP                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. STEADY STATE                                                 │
│     Definir comportamento normal usando SLIs                     │
│     "Em condições normais, p99 < 300ms e error < 0.1%"          │
│                                                                  │
│  2. HYPOTHESIS                                                   │
│     Formular hipótese de resiliência                             │
│     "Se o database primário falhar, o failover para             │
│      read-replica deve manter p99 < 500ms"                       │
│                                                                  │
│  3. EXPERIMENT                                                   │
│     Injetar falha controlada                                     │
│     "Simulate database primary failure por 5 min"               │
│                                                                  │
│  4. OBSERVE                                                      │
│     Monitorar SLIs durante o experimento                         │
│     Traces mostram o caminho do failover?                        │
│     Alertas dispararam corretamente?                             │
│     Dashboard reflete o impacto?                                 │
│                                                                  │
│  5. CONCLUDE                                                     │
│     Hipótese validada ou invalidada?                             │
│     Se invalidada → fix → re-test                                │
│     Se validada → documentar resiliência comprovada              │
│                                                                  │
│            ┌──────────────┐                                      │
│            │ STEADY STATE │                                      │
│            └──────┬───────┘                                      │
│                   ▼                                              │
│            ┌──────────────┐                                      │
│            │  HYPOTHESIS  │                                      │
│            └──────┬───────┘                                      │
│                   ▼                                              │
│            ┌──────────────┐                                      │
│            │  EXPERIMENT  │                                      │
│            └──────┬───────┘                                      │
│                   ▼                                              │
│            ┌──────────────┐                                      │
│            │   OBSERVE    │                                      │
│            └──────┬───────┘                                      │
│                   ▼                                              │
│            ┌──────────────┐                                      │
│            │   CONCLUDE   │─── invalidated ─── FIX ───┐         │
│            └──────┬───────┘                            │         │
│                   │ validated                           │         │
│                   ▼                                     │         │
│            ┌──────────────┐                    ┌───────▼──┐     │
│            │  DOCUMENT    │                    │ RE-TEST  │     │
│            └──────────────┘                    └──────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Experimentos de Chaos por categoria

| Categoria | Experimento | Ferramenta | O que observar |
|-----------|-------------|-----------|---------------|
| **Network** | Latency injection (200ms extra) | Chaos Mesh, Litmus | p99 do serviço, timeouts, retries |
| **Network** | Packet loss 10% | tc netem, Chaos Mesh | Error rate, retry storms |
| **Network** | DNS failure | Chaos Mesh | Fallback path, caching |
| **Compute** | Kill pod/container | kubectl delete, Chaos Mesh | Recovery time, request errors |
| **Compute** | CPU stress 90% | stress-ng, Chaos Mesh | Latency, autoscaling trigger |
| **Compute** | Memory pressure | stress-ng | OOM kills, GC pauses |
| **Database** | Primary failover | RDS failover | Failover time, error spike duration |
| **Database** | Slow queries (inject delay) | Toxiproxy | Circuit breaker trigger, timeout |
| **External** | 3rd-party API timeout | Toxiproxy, Chaos Mesh | Fallback, degraded mode, circuit breaker |
| **External** | 3rd-party API 500 errors | WireMock, Toxiproxy | Retry logic, error budget impact |
| **Region** | AZ failure simulation | AWS FIS | Multi-AZ failover, cross-AZ latency |

### AWS Fault Injection Simulator (FIS)

```
AWS FIS — Managed chaos engineering

EXPERIMENTOS DISPONÍVEIS:
• EC2: stop/terminate instances, CPU/memory stress
• ECS: stop tasks, drain container instances
• EKS: terminate pods, node drain
• RDS: failover, reboot
• Network: disrupt connectivity between AZs
• SSM: inject latency/errors via SSM agent

INTEGRAÇÃO COM OBSERVABILIDADE:
  1. Antes: snapshot dos SLIs (baseline)
  2. Durante: dashboard em tempo real
  3. Trigger: FIS experiment + CloudWatch Alarm
  4. Stop condition: SLO breach → auto-stop experiment
  5. Depois: comparar SLIs antes/durante/depois

STOP CONDITION (OBRIGATÓRIO):
  "Se error_rate > 5% por 2 minutos, PARAR o experimento"
  Usa CloudWatch Alarm como stop condition
```

---

## Deploy Strategies com Observability Gates

### Comparação de estratégias

| Strategy | Rollback speed | Observability window | Risco | Complexidade |
|----------|---------------|---------------------|-------|-------------|
| **Rolling update** | Minutos | Limitado (mixed versions durante rollout) | Médio | Baixa |
| **Blue/Green** | Segundos (switch) | Curto (switch instantâneo) | Baixo | Média |
| **Canary** | Segundos (% → 0) | Longo + granular | Muito baixo | Alta |
| **Feature flag** | Milliseconds (flag off) | Excelente (A/B comparison) | Mínimo | Média |
| **Shadow/Dark** | N/A (users não impactados) | Excelente | Zero | Alta |

### Pipeline com Observability Gates

```
┌─────────────────────────────────────────────────────────────────┐
│         CI/CD PIPELINE — OBSERVABILITY-GATED                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Build → Test → Deploy Canary → OBSERVE → Promote → OBSERVE     │
│                                   │                    │         │
│                                   ▼                    ▼         │
│                            ┌────────────┐       ┌────────────┐  │
│                            │ Gate Check │       │ Gate Check │  │
│                            │            │       │            │  │
│                            │ • Error <1%│       │ • SLO met  │  │
│                            │ • p99 ok   │       │ • No new   │  │
│                            │ • No crash │       │   errors   │  │
│                            │ • SLO safe │       │ • Metrics  │  │
│                            │            │       │   stable   │  │
│                            ├────────────┤       ├────────────┤  │
│                            │ ✅ → next  │       │ ✅ → done  │  │
│                            │ ❌ → roll  │       │ ❌ → roll  │  │
│                            │     back   │       │     back   │  │
│                            └────────────┘       └────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Incident Management

### Incident Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│              INCIDENT LIFECYCLE                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐                │
│  │ DETECT   │ ──▶ │ TRIAGE   │ ──▶ │ RESPOND  │                │
│  │          │     │          │     │          │                │
│  │ • Alert  │     │ • Sev?   │     │ • IC     │                │
│  │ • User   │     │ • Impact?│     │ • Comms  │                │
│  │   report │     │ • Scope? │     │ • Fix    │                │
│  │ • Synth. │     │ • IC     │     │ • Mitigate│               │
│  │   monitor│     │   assign │     │          │                │
│  └──────────┘     └──────────┘     └──────────┘                │
│                                          │                       │
│                                          ▼                       │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐                │
│  │ IMPROVE  │ ◀── │ REVIEW   │ ◀── │ RESOLVE  │                │
│  │          │     │          │     │          │                │
│  │ • Action │     │ •Blamele-│     │ • Verify │                │
│  │   items  │     │  ss post │     │ • Commun.│                │
│  │ • SLO    │     │  mortem  │     │ • Monitor│                │
│  │   update │     │ • Root   │     │   period │                │
│  │ • Runbook│     │   cause  │     │          │                │
│  └──────────┘     └──────────┘     └──────────┘                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Severidade e Resposta

| Severidade | Definição | Response time | Comunicação | Exemplo |
|-----------|-----------|--------------|-------------|---------|
| **SEV1 / P1** | Serviço principal down, impacto total | < 5 min | Incident channel + StatusPage + Stakeholders | API retornando 500 para todos os users |
| **SEV2 / P2** | Degradação significativa | < 15 min | Incident channel + StatusPage | p99 > 2x baseline, 10% users impactados |
| **SEV3 / P3** | Impacto limitado, workaround existe | < 1 hora | Incident channel | Feature específica lenta, página lollect |
| **SEV4 / P4** | Impacto mínimo, cosmético | Próximo business day | Ticket | UI bug, log errors sem impacto |

### Incident Commander (IC) — Responsabilidades

```
INCIDENT COMMANDER:

FASE DETECT/TRIAGE:
  ✅ Declarar o incidente (severity)
  ✅ Criar canal de comunicação (#incident-YYYY-MM-DD-descricao)
  ✅ Designar roles (IC, tech lead, communicator)

FASE RESPOND:
  ✅ Coordenar (NÃO debugar) — IC não faz hands-on
  ✅ Manter timeline atualizado
  ✅ Decidir: mitigar primeiro ou fix root cause
  ✅ Escalar se necessário (mais pessoas, higher management)
  ✅ Comunicação regular (a cada 15-30 min para SEV1)

FASE RESOLVE:
  ✅ Confirmar resolução (SLIs voltaram ao normal)
  ✅ Período de observação (30min após fix)
  ✅ Comunicar resolução
  ✅ Agendar post-mortem (até 48h após resolução)

REGRA:
  "O IC coordena. O tech lead investiga. 
   O communicator atualiza stakeholders.
   NUNCA uma pessoa faz tudo."
```

---

## Post-Mortem / Incident Review

### Template de Post-Mortem Blameless

```markdown
## Post-Mortem: [Título do Incidente]

**Data:** YYYY-MM-DD
**Severidade:** SEV1/SEV2/SEV3
**Duração:** HH:MM (detect to resolve)
**Incident Commander:** [Nome]
**Autores:** [Nomes]

### Resumo Executivo
[2-3 frases: o que aconteceu, impacto, resolução]

### Impacto
- **Usuários impactados:** X (Y% do total)
- **Duração do impacto para users:** HH:MM
- **Error budget consumido:** Z% do budget mensal
- **Revenue impactado:** $X (se aplicável)
- **SLOs violados:**
  - Availability: 99.5% (target: 99.9%)
  - Latency p99: 2.5s (target: 500ms)

### Timeline (OBRIGATÓRIO — detalhado, em UTC)
| Horário (UTC) | Evento |
|--------------|--------|
| 14:00 | Deploy v2.3.1 inicia |
| 14:05 | Canary 5% — métricas normais |
| 14:15 | Canary 50% — p99 começa a subir |
| 14:22 | Alert P2: p99 > 500ms |
| 14:25 | IC declarado: [Nome] |
| 14:30 | Root cause identificado: query N+1 no novo endpoint |
| 14:32 | Rollback para v2.3.0 iniciado |
| 14:35 | Rollback concluído, SLIs normalizando |
| 14:45 | SLIs normais, incidente resolvido |

### Root Cause
[Descrição técnica detalhada da causa raiz]
Não "quem", mas "que condição no sistema".

### Detection
- **Como foi detectado:** Alert automático / User report / Synthetic
- **Tempo até detecção (TTD):** X minutos
- **O alerta foi suficiente?** Sim/Não — o que faltou?

### Diagrama de Contribuição (não "causa e efeito" linear)
[Fatores que contribuíram para o incidente]
- Deploy sem canary gate automático
- Query sem index para novo padrão de acesso  
- Load test não cobria esse cenário
- Circuit breaker não configurado para DB timeout

### O que funcionou bem
- Alert disparou em 7 minutos
- Runbook de rollback seguido corretamente
- Comunicação rápida no canal de incidente

### O que pode ser melhorado
- Canary gate automático teria detectado antes do 50%
- Query review no PR não pegou o N+1

### Action Items (OBRIGATÓRIO — com owner e deadline)
| # | Ação | Owner | Deadline | Status |
|---|------|-------|----------|--------|
| 1 | Implementar canary gate com SLO check | @eng-alice | 2024-02-15 | TODO |
| 2 | Adicionar index para query pattern X | @eng-bob | 2024-02-08 | DONE |
| 3 | Adicionar load test para cenário Y | @eng-carol | 2024-02-20 | TODO |
| 4 | Configurar circuit breaker para DB | @eng-dave | 2024-02-15 | TODO |
| 5 | Revisar alerta de latency (threshold) | @eng-alice | 2024-02-10 | TODO |

### Revisão dos Action Items
- **Data de revisão:** YYYY-MM-DD (+30 dias)
- **Responsável por follow-up:** [Nome do Engineering Manager / IC]
```

### Métricas de incident management

```
MÉTRICAS ESSENCIAIS:

MTTD (Mean Time to Detect)
  = Tempo entre início do impacto e primeiro alerta
  Target: < 5 min para SEV1

MTTE (Mean Time to Engage)  
  = Tempo entre alerta e IC respondendo
  Target: < 10 min para SEV1

MTTR (Mean Time to Resolve)
  = Tempo entre detecção e resolução
  Target: < 1 hora para SEV1

MTTU (Mean Time to Understand)
  = Tempo entre engage e entender root cause
  Mais difícil de medir, mais impactante
  Observabilidade REDUZ MTTU → reduz MTTR

FREQUÊNCIA DE INCIDENTES
  = Incidentes SEV1+SEV2 por mês
  Trend importa mais que valor absoluto

ERROR BUDGET CONSUMPTION POR INCIDENTE
  = % do error budget mensal consumido
  Incidentes que consomem > 50% do budget → SEV1 review obrigatório
```

---

## On-Call Practices

### On-Call Guidelines

```
┌─────────────────────────────────────────────────────────────────┐
│                  ON-CALL GUIDELINES                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PRINCÍPIOS:                                                     │
│                                                                  │
│  1. ON-CALL É RESPONSABILIDADE DO TIME, NÃO DE OPS              │
│     "You build it, you run it" — Werner Vogels                   │
│     Developers on-call para seus próprios serviços               │
│                                                                  │
│  2. ALERTAS ACIONÁVEIS                                           │
│     Cada alerta que acorda alguém DEVE requerer ação humana     │
│     Se o alerta não precisa de ação → eliminar/silent           │
│                                                                  │
│  3. RUNBOOKS PARA CADA ALERTA                                    │
│     Alerta sem runbook = alerta mal definido                     │
│     Runbook: o que verificar, como diagnosticar, como resolver  │
│                                                                  │
│  4. ROTAÇÃO JUSTA                                                │
│     Mínimo 2 pessoas por rotação                                 │
│     Rotação semanal (não mensal)                                 │
│     Compensação adequada (dia de folga, bônus)                   │
│                                                                  │
│  5. MELHORIA CONTÍNUA                                            │
│     Toda page que acorda → post-incident review                  │
│     "Por que o sistema não se auto-recuperou?"                   │
│     Objetivo: ZERO pages desnecessárias                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Runbook Template

```markdown
## Runbook: [Nome do Alerta]

**Alerta:** `slo:order_service:availability:burn_rate_1h > 14.4`
**Severidade:** P1
**Dashboard:** [link para dashboard]

### O que está acontecendo
O burn rate de 1h está > 14.4x, significando que o error budget de 30 dias
será consumido em ~2 horas se o problema continuar.

### Diagnóstico rápido (< 5 min)
1. Abrir dashboard: [link]
2. Verificar: Quais endpoints estão com errors?
3. Verificar: Latency está alta? (pode ser timeout → erro)
4. Verificar: Deploy recente? (`kubectl rollout history`)
5. Verificar: Dependency health (DB, cache, 3rd party)

### Causas comuns + resolução
| Causa | Como confirmar | Resolução |
|-------|---------------|-----------|
| Bad deploy | Errors começaram após deploy | `kubectl rollout undo` |
| DB overload | DB CPU/connections altos | Scale read replicas / Kill long queries |
| Memory leak | Pod memory crescendo | Restart pods (`kubectl rollout restart`) |
| 3rd party down | Errors no span da dependency | Ativar circuit breaker / Fallback mode |
| Traffic spike | QPS > 2x normal | Verificar autoscaling / Rate limiting |

### Escalação
- Se não resolver em 15 min → Chamar tech lead: [contato]
- Se impacto financeiro → Notificar management: [contato]
- Se data breach suspeitado → Security team: [contato]
```

---

## Observability como Cultura

### Maturity model cultural

| Nível | Cultura | Práticas | Indicador |
|-------|---------|----------|-----------|
| **1. Reactive** | "Ops cuida de production" | Alertas por email, investigação manual | MTTR > 4 horas |
| **2. Informed** | "Developers olham dashboards" | Dashboards básicas, some alerting | MTTR 1-4 horas |
| **3. Proactive** | "Instrumentamos antes de deployar" | ODD, SLOs definidos, canary deploys | MTTR 30min-1h |
| **4. Predictive** | "Prevemos problemas antes de acontecerem" | Anomaly detection, chaos eng regular, error budgets como velocity tool | MTTR < 30 min |
| **5. Autonomous** | "O sistema se auto-recupera, nós melhoramos" | Auto-remediation, ML-driven alerting, culture of learning | MTTR < 10 min |

### Como evoluir a cultura

```
DE NÍVEL 1 → 2:
  ✅ Dar acesso a dashboards para TODOS os devs
  ✅ Mostrar impacto de incidentes (error budget em dashboard)
  ✅ Incluir devs em post-mortems

DE NÍVEL 2 → 3:
  ✅ Definir SLOs para cada serviço
  ✅ Implementar ODD no ciclo de desenvolvimento
  ✅ Deploy ≠ Release (feature flags)
  ✅ "You build it, you run it" (devs on-call)

DE NÍVEL 3 → 4:
  ✅ Chaos engineering regular (game days)
  ✅ Error budget policies com teeth (parar features)
  ✅ Instrumentação como parte do Definition of Done
  ✅ SLO reviews trimestrais com data

DE NÍVEL 4 → 5:
  ✅ Auto-remediation (runbooks automatizados)
  ✅ AIOps (anomaly detection, correlation)
  ✅ Continuous chaos (automated experiments)
  ✅ Blame-aware → learn-focused culture
```

---

## Anti-Patterns ODD

| Anti-pattern | Problema | Solução |
|-------------|---------|---------|
| **Observability afterthought** | Instrumentar depois que bugs aparecerem | ODD: instrumentar durante design |
| **Deploy = Release** | Tudo junto, sem controle gradual | Feature flags, canary deployments |
| **Alert fatigue** | 100+ alertas, ninguém lê | Cada alerta = ação humana necessária. SLO-based alerting |
| **Blame culture** | Post-mortem busca culpado | Blameless: buscar condições sistêmicas |
| **Post-mortem sem follow-up** | Action items nunca executados | Owner + deadline + revisão em 30d |
| **Only pre-prod testing** | "Funciona no staging" | Test in production com safety nets |
| **Canary manual** | "Looks good to me" subjetivo | SLO-gated canary com rollback automático |
| **Chaos sem hipótese** | "Vamos ver o que acontece" | Hypothesis → experiment → observe → conclude |
| **On-call como punição** | Apenas ops/SRE on-call | Devs on-call para seus serviços |
| **Dashboard sem contexto** | Métricas sem explicação | Cada panel com description do que significa e thresholds |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e práticas relacionadas a Observability-Driven Development, verifique:

1. **Feature sem instrumentação** — Toda feature nova deve ter spans custom, métricas de negócio e dashboard
2. **Deploy sem canary/flag** — Features de risco devem usar feature flag ou canary com SLO gate
3. **Feature flag sem métricas** — Cada flag deve ter métricas de adoption, performance e errors por variant
4. **SLO sem definição formal** — Todo serviço em produção deve ter SLOs documentados e alertas configurados
5. **Alerta sem runbook** — Cada alerta que pode gerar page deve ter runbook linkado
6. **Alerta baseado em causa** — Alertar por sintomas (SLO breach), não por causas (CPU > 80%)
7. **Post-mortem sem action items** — Todo post-mortem deve ter action items com owner, deadline e status
8. **Chaos sem stop condition** — Todo experimento de chaos deve ter stop condition automática (SLO breach)
9. **Synthetic sem flag "synthetic: true"** — Testes sintéticos devem ser identificáveis e filtráveis dos SLIs
10. **Rollback manual sem automação** — Canary gates devem ter rollback automático em caso de SLO breach
11. **On-call sem rotação documentada** — Rotação, escalação e contact list devem estar documentados
12. **Error budget sem policy** — Error budget sem política de ação documentada é apenas um número

---

## Referências

- **Observability Engineering** — Charity Majors, Liz Fong-Jones, George Miranda (O'Reilly)
- **Implementing Service Level Objectives** — Alex Hidalgo (O'Reilly)
- **Site Reliability Engineering** — Betsy Beyer et al. (Google)
- **The Site Reliability Workbook** — Betsy Beyer et al. (Google)
- **Chaos Engineering** — Casey Rosenthal, Nora Jones (O'Reilly)
- **Accelerate** — Nicole Forsgren, Jez Humble, Gene Kim
- **Release It!** — Michael Nygard (Pragmatic Bookshelf)
- **AWS Fault Injection Simulator** — https://aws.amazon.com/fis/
- **OpenFeature** — https://openfeature.dev/ (standard para feature flags)
- **Argo Rollouts** — https://argoproj.github.io/rollouts/
- **Flagger** — https://flagger.app/
- **PagerDuty Incident Response** — https://response.pagerduty.com/
