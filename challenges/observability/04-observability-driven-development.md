# Level 4 — Observability-Driven Development (ODD)

> **Objetivo:** Aplicar Observability-Driven Development integrando feature flags com métricas, canary deployments com SLO gates, chaos engineering com observabilidade, e processos de incident management (blameless post-mortem, runbooks, on-call).

---

## Objetivo de Aprendizado

- Entender ODD: observabilidade como parte do ciclo de desenvolvimento, não pós-deploy
- Integrar feature flags com métricas (flag habilitada → medir impacto)
- Implementar canary deployment com SLO gate (rollback automático se SLO viola)
- Projetar chaos experiments com hipóteses mensuráveis via observabilidade
- Conduzir incident management com severity levels e roles (IC, Comms, Ops)
- Escrever blameless post-mortems com timeline e action items
- Criar runbooks operacionais acionáveis
- Estabelecer on-call practices com escalation policies

---

## Escopo Funcional

### Feature Flags + Observabilidade

```
┌──────────────────────────────────────────────────────────────────┐
│                 FEATURE FLAGS + OBSERVABILITY                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  CENÁRIO: Novo algoritmo de cálculo de frete no Order Service.   │
│                                                                   │
│  ┌─────────────┐     ┌───────────────────────────┐               │
│  │ Feature Flag │     │ Observability              │               │
│  │ Provider     │     │                            │               │
│  │              │     │  Metric: feature_flag_     │               │
│  │ Flag:        │────>│    evaluation_total{       │               │
│  │ new-shipping-│     │    flag="new-shipping",    │               │
│  │ algorithm    │     │    variant="on|off"}       │               │
│  │              │     │                            │               │
│  │ Variants:    │     │  Span attribute:           │               │
│  │  on  (30%)   │     │    feature_flag.key=       │               │
│  │  off (70%)   │     │    "new-shipping-algorithm"│               │
│  │              │     │    feature_flag.variant=    │               │
│  └─────────────┘     │    "on"                     │               │
│                       │                            │               │
│                       │  Dashboard: compare         │               │
│                       │   - p99 latency on vs off   │               │
│                       │   - error rate on vs off    │               │
│                       │   - shipping calc time      │               │
│                       └───────────────────────────┘               │
│                                                                   │
│  DECISÃO: se error rate "on" > 2x "off" → disable flag.         │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Canary Deployment com SLO Gate

```
┌──────────────────────────────────────────────────────────────────┐
│                 CANARY + SLO GATE                                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────┐   ┌──────────────┐   ┌──────────────────┐         │
│  │  Load     │   │  Canary (10%)│   │  Stable (90%)     │         │
│  │  Balancer │──>│  v2.0.0      │   │  v1.9.0           │         │
│  │           │──>│              │   │                    │         │
│  └──────────┘   └──────┬───────┘   └──────────┬─────────┘         │
│                         │                      │                   │
│                         ▼                      ▼                   │
│              ┌──────────────────────────────────────────┐         │
│              │  Prometheus: SLI por deployment version  │         │
│              │                                           │         │
│              │  sli:order:availability{version="2.0.0"}  │         │
│              │  sli:order:availability{version="1.9.0"}  │         │
│              └────────────────┬──────────────────────────┘         │
│                               │                                    │
│                               ▼                                    │
│              ┌──────────────────────────────────────────┐         │
│              │  SLO Gate (script automático a cada 5m)  │         │
│              │                                           │         │
│              │  IF canary_error_rate > 2x stable:       │         │
│              │    → ROLLBACK canary                      │         │
│              │    → Alert: "Canary failed SLO gate"      │         │
│              │                                           │         │
│              │  IF canary_p99 > 1.5x stable:            │         │
│              │    → ROLLBACK canary                      │         │
│              │                                           │         │
│              │  IF 30min passed AND metrics OK:          │         │
│              │    → PROMOTE canary to 50% → 100%         │         │
│              └──────────────────────────────────────────┘         │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Chaos Engineering com Observabilidade

```
┌──────────────────────────────────────────────────────────────────┐
│                 CHAOS EXPERIMENTS                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  EXPERIMENT 1: "Payment Service Down"                             │
│  ├── Hipótese: "Quando Payment Service está indisponível,        │
│  │    Order Service retorna 503 em < 5s com circuit breaker       │
│  │    e error rate sobe, mas SLO não é violado graças ao          │
│  │    error budget."                                              │
│  ├── Ação: docker stop payment-service por 5 minutos             │
│  ├── Observar:                                                    │
│  │    ├── Traces: spans com status=ERROR no payment call          │
│  │    ├── Metrics: circuit breaker state (open/closed/half-open)  │
│  │    ├── Metrics: error rate order-service                       │
│  │    ├── Logs: "payment service unreachable, circuit open"       │
│  │    └── SLO: error budget consumption rate                      │
│  └── Critério de sucesso:                                         │
│       ├── Circuit breaker abre em < 10s                           │
│       ├── Timeout configureado < 5s                               │
│       ├── Error budget consumption < 5% do total                  │
│       └── Recovery automático quando payment volta                │
│                                                                   │
│  EXPERIMENT 2: "Database Latency Spike"                           │
│  ├── Hipótese: "Latência de 2s no PostgreSQL causa degradação    │
│  │    controlada, não cascading failure."                          │
│  ├── Ação: tc qdisc add dev eth0 root netem delay 2000ms          │
│  ├── Observar:                                                    │
│  │    ├── Traces: db spans com duration > 2s                      │
│  │    ├── Metrics: p99 latency order-service                      │
│  │    ├── SLO: latency SLI violation                              │
│  │    └── Alerts: P2 burn rate alert triggered?                   │
│  └── Critério de sucesso:                                         │
│       ├── Connection pool não esgota                              │
│       ├── Timeout no DB configureado (< 5s)                       │
│       └── Serviço degrada gracefully, não crash                   │
│                                                                   │
│  EXPERIMENT 3: "Kafka Partition Leader Election"                  │
│  ├── Hipótese: "Rebalance do Kafka causa delay < 30s             │
│  │    nas notificações, mas freshness SLO se mantém."            │
│  ├── Ação: docker stop kafka-broker-1 (em cluster de 3)          │
│  ├── Observar:                                                    │
│  │    ├── Metrics: kafka consumer lag                              │
│  │    ├── Traces: notification delivery time                      │
│  │    ├── SLO: freshness SLI                                      │
│  │    └── Logs: consumer rebalance messages                       │
│  └── Critério de sucesso:                                         │
│       ├── Consumer rebalance completa em < 30s                    │
│       └── Zero mensagens perdidas                                 │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Incident Management

```
┌──────────────────────────────────────────────────────────────────┐
│                 INCIDENT MANAGEMENT                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  SEVERITY LEVELS:                                                 │
│  ┌──────────┬──────────────────────────────────────────────┐     │
│  │ Severity │ Definição                                     │     │
│  │ SEV1     │ Serviço principal (Order) indisponível.       │     │
│  │          │ Impacto em > 50% dos usuários. Perda de       │     │
│  │          │ receita ativa. Error budget esgotando         │     │
│  │          │ em < 2h (burn rate > 14.4x).                  │     │
│  │ SEV2     │ Degradação significativa. p99 > 2x normal.   │     │
│  │          │ Funcionalidade parcial comprometida.          │     │
│  │          │ Burn rate > 6x.                               │     │
│  │ SEV3     │ Degradação menor. Impacto limitado.          │     │
│  │          │ Error budget sendo consumido acima do normal. │     │
│  │ SEV4     │ Cosmético ou não urgente. Sem impacto em      │     │
│  │          │ SLOs. Pode ser tratado no próximo sprint.     │     │
│  └──────────┴──────────────────────────────────────────────┘     │
│                                                                   │
│  ROLES DURANTE INCIDENTE:                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Role                │ Responsabilidade                    │   │
│  │ Incident Commander  │ Coordena resposta, toma decisões    │   │
│  │ (IC)                │ de escopo, comunica status           │   │
│  │ Communications Lead │ Atualiza stakeholders, status page  │   │
│  │ Operations Lead     │ Executa ações técnicas, debug        │   │
│  │ Subject Matter      │ Especialistas consultados conforme  │   │
│  │ Experts (SMEs)      │ necessário                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Escopo Técnico

### Feature Flag Provider por Stack

| Stack | Feature Flag Provider | Configuração |
|-------|----------------------|-------------|
| **Go (Gin)** | OpenFeature Go SDK + flagd | `go.openfeature.dev/sdk` |
| **Spring Boot** | OpenFeature Java SDK + Togglz ou Unleash | `dev.openfeature:sdk` |
| **Quarkus** | OpenFeature Java SDK + Extension | `quarkus-openfeature` |
| **Micronaut** | OpenFeature Java SDK | `dev.openfeature:sdk` |
| **Jakarta EE** | OpenFeature Java SDK + CDI | `dev.openfeature:sdk` |

### Go — Feature Flag com Observabilidade

```go
// internal/platform/featureflags/provider.go
package featureflags

import (
    "context"

    "github.com/open-feature/go-sdk/openfeature"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/metric"
)

var (
    tracer           = otel.Tracer("feature-flags")
    meter            = otel.Meter("feature-flags")
    flagEvalCounter  metric.Int64Counter
)

func init() {
    var err error
    flagEvalCounter, err = meter.Int64Counter("feature_flag.evaluation_total",
        metric.WithDescription("Number of feature flag evaluations"),
    )
    if err != nil {
        panic(err)
    }
}

// EvalBool avalia uma flag e registra métricas + span attributes
func EvalBool(ctx context.Context, flagKey string, defaultValue bool) bool {
    client := openfeature.NewClient("order-service")

    result, err := client.BooleanValue(ctx, flagKey, defaultValue, openfeature.EvaluationContext{})
    if err != nil {
        flagEvalCounter.Add(ctx, 1,
            metric.WithAttributes(
                attribute.String("feature_flag.key", flagKey),
                attribute.String("feature_flag.variant", "error"),
            ),
        )
        return defaultValue
    }

    variant := "off"
    if result {
        variant = "on"
    }

    // Métrica de avaliação
    flagEvalCounter.Add(ctx, 1,
        metric.WithAttributes(
            attribute.String("feature_flag.key", flagKey),
            attribute.String("feature_flag.variant", variant),
        ),
    )

    // Span attribute para correlação em traces
    span := otel.SpanFromContext(ctx)
    if span.IsRecording() {
        span.SetAttributes(
            attribute.String("feature_flag.key", flagKey),
            attribute.String("feature_flag.variant", variant),
        )
    }

    return result
}
```

```go
// internal/domain/service/order_service.go — uso da flag
func (s *OrderService) CalculateShipping(ctx context.Context, order *Order) (Money, error) {
    ctx, span := tracer.Start(ctx, "OrderService.CalculateShipping")
    defer span.End()

    useNewAlgorithm := featureflags.EvalBool(ctx, "new-shipping-algorithm", false)

    var cost Money
    var err error

    if useNewAlgorithm {
        cost, err = s.newShippingCalculator.Calculate(ctx, order)
    } else {
        cost, err = s.legacyShippingCalculator.Calculate(ctx, order)
    }

    span.SetAttributes(
        attribute.String("shipping.algorithm", algorithmName(useNewAlgorithm)),
        attribute.Float64("shipping.cost", cost.Amount),
    )

    return cost, err
}
```

### Spring Boot — Feature Flag com Observabilidade

```java
// infrastructure/featureflags/ObservableFeatureFlagService.java
@Service
@RequiredArgsConstructor
public class ObservableFeatureFlagService {

    private final Client openFeatureClient;
    private final MeterRegistry meterRegistry;
    private final Tracer tracer;

    public boolean evaluate(String flagKey, boolean defaultValue) {
        boolean result = openFeatureClient.getBooleanValue(flagKey, defaultValue);

        String variant = result ? "on" : "off";

        // Métrica
        meterRegistry.counter("feature_flag.evaluation.total",
            "feature_flag.key", flagKey,
            "feature_flag.variant", variant
        ).increment();

        // Span attribute
        Span currentSpan = Span.current();
        if (currentSpan.isRecording()) {
            currentSpan.setAttribute("feature_flag.key", flagKey);
            currentSpan.setAttribute("feature_flag.variant", variant);
        }

        return result;
    }
}
```

### Canary SLO Gate — Script de Verificação

```bash
#!/bin/bash
# scripts/canary-slo-gate.sh
# Verifica se canary version atende SLO gates antes de promover

PROMETHEUS_URL="${PROMETHEUS_URL:-http://localhost:9090}"
CANARY_VERSION="${CANARY_VERSION:-2.0.0}"
STABLE_VERSION="${STABLE_VERSION:-1.9.0}"
MAX_ERROR_RATIO=2.0    # canary error rate max 2x do stable
MAX_LATENCY_RATIO=1.5  # canary p99 max 1.5x do stable

echo "=== SLO Gate Check ==="
echo "Canary: $CANARY_VERSION | Stable: $STABLE_VERSION"

# Error rate canary
canary_errors=$(curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=rate(http_requests_total{service_version=\"$CANARY_VERSION\",status=~\"5..\"}[5m])" \
  | jq -r '.data.result[0].value[1] // "0"')

# Error rate stable
stable_errors=$(curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=rate(http_requests_total{service_version=\"$STABLE_VERSION\",status=~\"5..\"}[5m])" \
  | jq -r '.data.result[0].value[1] // "0"')

# Ratio check
if (( $(echo "$canary_errors > $stable_errors * $MAX_ERROR_RATIO" | bc -l) )); then
    echo "❌ FAILED: Canary error rate ($canary_errors) > ${MAX_ERROR_RATIO}x stable ($stable_errors)"
    echo "ACTION: Rolling back canary..."
    # kubectl rollout undo deployment/order-service-canary
    exit 1
fi

# p99 latency canary
canary_p99=$(curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{service_version=\"$CANARY_VERSION\"}[5m]))" \
  | jq -r '.data.result[0].value[1] // "0"')

# p99 latency stable
stable_p99=$(curl -s "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{service_version=\"$STABLE_VERSION\"}[5m]))" \
  | jq -r '.data.result[0].value[1] // "0"')

if (( $(echo "$canary_p99 > $stable_p99 * $MAX_LATENCY_RATIO" | bc -l) )); then
    echo "❌ FAILED: Canary p99 ($canary_p99) > ${MAX_LATENCY_RATIO}x stable ($stable_p99)"
    echo "ACTION: Rolling back canary..."
    exit 1
fi

echo "✅ PASSED: Canary metrics within SLO gates"
echo "  Error rate: canary=$canary_errors stable=$stable_errors"
echo "  P99 latency: canary=$canary_p99 stable=$stable_p99"
echo "ACTION: Ready to promote canary"
```

### Chaos Experiment — Docker Compose

```yaml
# scripts/chaos/payment-down.sh
#!/bin/bash
# Chaos Experiment: Payment Service Down
# Hipótese: Circuit breaker abre em < 10s, error budget consumption < 5%

echo "=== CHAOS: Payment Service Down ==="
echo "Hipótese: Circuit breaker abre em < 10s, orders degradam gracefully"
echo ""

# Capturar baseline
echo "[1/4] Capturando baseline metrics..."
baseline_error_budget=$(curl -s "http://localhost:9090/api/v1/query" \
  --data-urlencode 'query=slo:order_service:availability:error_budget_remaining' \
  | jq -r '.data.result[0].value[1]')
echo "  Error budget remaining: ${baseline_error_budget}%"

# Iniciar chaos
echo "[2/4] Parando payment-service..."
docker stop payment-service
START_TIME=$(date +%s)

# Aguardar e observar
echo "[3/4] Aguardando 5 minutos. Observe em:"
echo "  Grafana: http://localhost:3000/d/slo-overview"
echo "  Jaeger:  http://localhost:16686 (procure spans com error)"
echo "  Logs:    docker logs -f order-service"
sleep 300

# Verificar impacto
echo "[4/4] Verificando impacto..."
final_error_budget=$(curl -s "http://localhost:9090/api/v1/query" \
  --data-urlencode 'query=slo:order_service:availability:error_budget_remaining' \
  | jq -r '.data.result[0].value[1]')
echo "  Error budget remaining: ${final_error_budget}%"
budget_consumed=$(echo "$baseline_error_budget - $final_error_budget" | bc)
echo "  Budget consumed during chaos: ${budget_consumed}%"

# Restaurar
echo "Restaurando payment-service..."
docker start payment-service

# Avaliar
if (( $(echo "$budget_consumed < 5" | bc -l) )); then
    echo "✅ HIPÓTESE CONFIRMADA: Budget consumption < 5%"
else
    echo "❌ HIPÓTESE REFUTADA: Budget consumption ${budget_consumed}% >= 5%"
    echo "  Action items:"
    echo "  - Revisar timeout do circuit breaker"
    echo "  - Verificar fallback do Payment Service"
fi
```

### Runbook Template

```markdown
## Runbook: Order Service High Error Rate

### Metadata
| Field | Value |
|-------|-------|
| Service | order-service |
| Alert | OrderServiceAvailabilityBudgetBurnP1 |
| Severity | SEV1/Page |
| On-call | order-platform team |
| Last updated | 2025-01-15 |
| Owner | @team-order-platform |

### Symptoms
- Alerta P1 de burn rate > 14.4x disparado
- Error budget esgotando em < 2 horas
- Dashboard: [SLO Order Service](http://grafana:3000/d/slo-order-service)

### Diagnóstico

#### Step 1: Identificar scope do problema
```bash
# Verificar error rate atual
curl -s "http://prometheus:9090/api/v1/query" \
  --data-urlencode 'query=rate(http_requests_total{service="order-service",status=~"5.."}[5m])'

# Verificar se é um endpoint específico
curl -s "http://prometheus:9090/api/v1/query" \
  --data-urlencode 'query=topk(5, rate(http_requests_total{service="order-service",status=~"5.."}[5m]))'
```

#### Step 2: Verificar dependências
```bash
# Payment service health
curl -s http://payment-service:8081/health/ready

# Database connectivity
curl -s http://order-service:8080/health/ready | jq '.checks.database'

# Kafka connectivity
curl -s http://order-service:8080/health/ready | jq '.checks.kafka'
```

#### Step 3: Verificar traces recentes com erro
- Abrir Jaeger: http://jaeger:16686
- Filtrar: service=order-service, tags=error=true
- Identificar span com erro → qual dependência falhou?

#### Step 4: Verificar logs
```bash
# Logs com erro nos últimos 5 minutos
docker logs order-service --since 5m 2>&1 | grep -i "error\|panic\|fatal"

# Ou no Grafana Explore → Loki:
# {service="order-service"} |= "error" | json
```

### Mitigação

#### Se dependência external (Payment, Inventory) está down:
1. Verificar se circuit breaker está ativo
2. Se não, considerar feature flag para desabilitar chamada
3. Notificar team da dependência

#### Se database está lenta/down:
1. Verificar connection pool: `pg_stat_activity`
2. Verificar slow queries: `pg_stat_statements`
3. Se necessário, failover para replica

#### Se é bug na versão atual:
1. Rollback: `docker-compose up -d --force-recreate order-service` (versão anterior)
2. Ou: `kubectl rollout undo deployment/order-service`

### Escalação
| Tempo | Ação |
|-------|------|
| 0-15min | On-call investiga |
| 15-30min | Escalar para IC (Incident Commander) |
| 30min+ | Escalar para Engineering Manager |
| 1h+ | Considerar SEV1 war room |
```

### Post-Mortem Template

```markdown
## Post-Mortem: [Título do Incidente]

### Metadata
| Field | Value |
|-------|-------|
| Date | YYYY-MM-DD |
| Duration | Xh Ym |
| Severity | SEV1/SEV2/SEV3 |
| Incident Commander | @name |
| Author | @name |
| Status | Draft / Reviewed / Complete |

### Summary
[1-2 parágrafos descrevendo o que aconteceu, impacto, e como foi resolvido]

### Impact
- **Duração**: XX minutos
- **Usuários impactados**: ~XX%
- **Orders perdidos/atrasados**: XX
- **Error budget consumed**: XX%
- **Revenue impact**: $XX (estimativa)

### Timeline (UTC)
| Time | Event |
|------|-------|
| 14:00 | Deploy v2.3.0 iniciado |
| 14:05 | P1 alert disparado: burn rate 14.4x |
| 14:07 | IC declarado. On-call acknowledge |
| 14:12 | Root cause identificado: query N+1 na nova feature |
| 14:15 | Rollback para v2.2.0 iniciado |
| 14:18 | Rollback concluído. Error rate normalizando |
| 14:25 | P1 alert resolvido. SLI recovery confirmado |
| 14:30 | Incidente encerrado |

### Root Cause
[Descrição técnica da causa raiz. Sem culpar pessoas.]

### Detection
- **Como detectamos**: Alerta P1 de burn rate automático
- **Tempo para detectar**: 5 minutos
- **Poderia ter sido mais rápido?**: Sim, canary com SLO gate teria detectado no staging

### Resolution
[O que foi feito para resolver. Rollback? Hotfix? Config change?]

### Lessons Learned
#### O que funcionou bem
- Alerta P1 detectou rapidamente
- Runbook estava atualizado
- Rollback foi rápido (3 minutos)

#### O que pode melhorar
- Canary deployment não estava habilitado
- Faltava load test na CI para queries pesadas
- Dashboard de SLO não tinha o novo endpoint

### Action Items
| # | Action | Owner | Priority | Due Date | Status |
|---|--------|-------|----------|----------|--------|
| 1 | Implementar canary com SLO gate no CI/CD | @name | P1 | YYYY-MM-DD | TODO |
| 2 | Adicionar load test para order listing | @name | P1 | YYYY-MM-DD | TODO |
| 3 | Incluir novo endpoint no SLO dashboard | @name | P2 | YYYY-MM-DD | TODO |
| 4 | Review de queries antes de merge | @name | P2 | YYYY-MM-DD | TODO |

### 5 Whys
1. **Por que houve alta error rate?** Porque a query de listing era N+1.
2. **Por que a query N+1 passou?** Porque não havia load test no CI.
3. **Por que não havia load test?** Porque não era mandatório no pipeline.
4. **Por que não era mandatório?** Porque não tínhamos definido quality gates.
5. **Por que não tínhamos quality gates?** → **Action Item**: Definir quality gates obrigatórios.
```

---

## Critérios de Aceite

### Feature Flags
- [ ] Feature flag provider configurado (OpenFeature + flagd/provider)
- [ ] Flag `new-shipping-algorithm` criada com variantes on/off
- [ ] Métrica `feature_flag.evaluation_total{flag, variant}` sendo emitida
- [ ] Span attribute `feature_flag.key` e `feature_flag.variant` nos traces
- [ ] Dashboard: comparação de métricas entre variantes on vs off

### Canary Deployment
- [ ] Script/pipeline de canary SLO gate implementado
- [ ] Canary verifica error rate e p99 latency contra stable
- [ ] Rollback automático se canary excede thresholds
- [ ] Promoção automática se canary OK por 30 min

### Chaos Engineering
- [ ] Pelo menos 2 chaos experiments documentados com hipóteses
- [ ] Cada experiment tem: hipótese, ação, métricas observadas, critério de sucesso
- [ ] Scripts de chaos reproduzíveis (bash ou docker compose)
- [ ] Resultados documentados: hipótese confirmada ou refutada + action items

### Incident Management
- [ ] Severity levels definidos (SEV1-SEV4) com critérios claros
- [ ] Roles documentados (IC, Comms, Ops, SME)
- [ ] Runbook escrito para pelo menos 2 alertas (P1 error rate, P2 latency)
- [ ] Post-mortem template com: timeline, root cause, 5 whys, action items

---

## Definição de Pronto (DoD)

- [ ] Todos os critérios de aceite ✅
- [ ] Feature flag funcional com métricas no dashboard
- [ ] Canary SLO gate testado com deploy simulado
- [ ] Chaos experiment executado e resultado documentado
- [ ] Runbook validado: outra pessoa seguiu e resolveu cenário simulado
- [ ] Post-mortem escrito para um incidente (real ou simulado)
- [ ] Commit: `feat(level-4): observability-driven development with feature flags, canary, chaos`

---

## Checklist

### Feature Flags
- [ ] OpenFeature SDK configurado no serviço principal
- [ ] flagd ou provider configurado no Docker Compose
- [ ] Métricas de flag evaluation exportadas para Prometheus
- [ ] Grafana dashboard: comparison panel (on vs off)

### Canary
- [ ] `scripts/canary-slo-gate.sh` — Script de verificação
- [ ] Docker Compose com 2 versões do mesmo serviço (canary + stable)
- [ ] Prometheus rules para SLI por `service_version`

### Chaos
- [ ] `scripts/chaos/payment-down.sh` — Experiment 1
- [ ] `scripts/chaos/db-latency.sh` — Experiment 2
- [ ] `docs/chaos-results.md` — Resultados documentados

### Incident Management
- [ ] `docs/runbooks/order-service-high-error-rate.md` — Runbook P1
- [ ] `docs/runbooks/order-service-high-latency.md` — Runbook P2
- [ ] `docs/incident-management.md` — Severity levels, roles, escalation
- [ ] `docs/post-mortem-template.md` — Template reutilizável

### Go
- [ ] `internal/platform/featureflags/` — OpenFeature provider com métricas
- [ ] Feature flag usage no domain service com span attributes

### Spring Boot / Quarkus / Micronaut / Jakarta EE
- [ ] OpenFeature Java SDK configurado
- [ ] Feature flag evaluation com Micrometer counter
- [ ] Span attributes para feature flag key/variant

---

## Extensões Opcionais

- [ ] Implementar progressive rollout: 10% → 25% → 50% → 100% com SLO gate em cada step
- [ ] Integrar Chaos Mesh (Kubernetes) para chaos experiments automatizados
- [ ] Implementar automated rollback via Prometheus webhook receiver
- [ ] Criar status page com Cachet ou Statuspage exibindo SLO compliance
- [ ] Implementar on-call rotation com PagerDuty/Opsgenie/Grafana OnCall
- [ ] Criar game day completo com cenário de incidente multi-serviço
- [ ] Integrar post-mortem com sistema de tracking para action items
