# Observabilidade Avançada — SLOs, SLIs & Error Budgets

> **Objetivo deste documento:** Servir como referência completa sobre **SLOs (Service Level Objectives), SLIs (Service Level Indicators) e Error Budgets**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: definições, como definir SLIs/SLOs, error budget policies, multi-window alerting, implementação prática e cultura SRE.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **SLI** | Mede experiência do USUÁRIO, não infra | SLI = CPU usage | SLI = success rate de requests do user |
| **SLO** | Target realista, não 100% | SLO = 100% availability | SLO = 99.9% (permite 43min downtime/mês) |
| **SLA** | SLA < SLO (margem de segurança) | SLA = SLO | SLO = 99.95%, SLA = 99.9% (buffer) |
| **Error Budget** | Budget governa velocidade de feature releases | "Vamos deployar mesmo com budget zerado" | Freeze deploys quando budget < threshold |
| **Burn Rate** | Multi-window para evitar false positives | Alerta em janela única de 5min | Multi-window: 1h + 5min para P1 |
| **Alerting** | Alerte sobre burn rate, não sobre valor instantâneo | "Error rate > 1% agora" | "Error budget queimando 14.4x mais rápido" |

---

## Sumário

- [Observabilidade Avançada — SLOs, SLIs \& Error Budgets](#observabilidade-avançada--slos-slis--error-budgets)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O Framework SLI/SLO/SLA](#o-framework-slislosla)
  - [Definindo SLIs — O que medir](#definindo-slis--o-que-medir)
  - [Definindo SLOs — O target](#definindo-slos--o-target)
  - [Error Budgets — A matemática](#error-budgets--a-matemática)
  - [Error Budget Policies — Governança](#error-budget-policies--governança)
  - [Multi-Window, Multi-Burn-Rate Alerting](#multi-window-multi-burn-rate-alerting)
  - [SLO-Based Alerting — Implementação](#slo-based-alerting--implementação)
  - [SLO para Diferentes Tipos de Sistema](#slo-para-diferentes-tipos-de-sistema)
  - [SLO Dashboard Design](#slo-dashboard-design)
  - [SLO Review Process](#slo-review-process)
  - [Anti-Patterns de SLO](#anti-patterns-de-slo)
  - [Implementação Prática — Passo a Passo](#implementação-prática--passo-a-passo)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O Framework SLI/SLO/SLA

```
┌─────────────────────────────────────────────────────────────────┐
│                    SLI / SLO / SLA                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SLI (Service Level INDICATOR)                                   │
│  ═════════════════════════════                                   │
│  O QUE: Métrica quantitativa da experiência do usuário           │
│  TIPO: Ratio (0% a 100%)                                        │
│  EXEMPLO: "Proporção de requests HTTP que retornam < 300ms       │
│            e status 2xx no último período"                       │
│  FÓRMULA:                                                        │
│    SLI = (good events / total events) × 100%                    │
│    SLI = (requests < 300ms AND 2xx) / (total requests) × 100%  │
│                                                                  │
│  SLO (Service Level OBJECTIVE)                                   │
│  ═════════════════════════════                                   │
│  O QUE: Target para o SLI num período de tempo                   │
│  DEFINE: "Estamos bons o suficiente?"                            │
│  EXEMPLO: "99.9% dos requests devem ser good events em 30 dias" │
│  FORMULA:                                                        │
│    SLI ≥ SLO target → ✅ Dentro do objetivo                     │
│    SLI < SLO target → ❌ Violação do SLO                        │
│                                                                  │
│  SLA (Service Level AGREEMENT)                                   │
│  ═════════════════════════════                                   │
│  O QUE: Contrato com consequências (financeiras/legais)          │
│  REGRA: SLA < SLO (sempre mais relaxado que o SLO interno)      │
│  EXEMPLO: "SLA para clientes: 99.9% uptime, ou créditos"        │
│           "SLO interno: 99.95% (margem de segurança)"            │
│                                                                  │
│  HIERARQUIA:                                                     │
│                                                                  │
│  ┌────────────────────────────────────────┐                     │
│  │  SLA = 99.9%  (contrato, externo)      │                     │
│  │  ┌────────────────────────────────┐    │                     │
│  │  │  SLO = 99.95% (target, interno)│    │                     │
│  │  │  ┌────────────────────────┐    │    │                     │
│  │  │  │  SLI = 99.97% (atual)  │    │    │                     │
│  │  │  └────────────────────────┘    │    │                     │
│  │  └────────────────────────────────┘    │                     │
│  └────────────────────────────────────────┘                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Definindo SLIs — O que medir

### Tipos de SLI por categoria de serviço

| Tipo de serviço | SLI primário | SLI secundário | Fórmula |
|----------------|-------------|----------------|---------|
| **Request-driven** (API, web) | Availability + Latency | Throughput | `good_requests / total_requests` |
| **Pipeline** (ETL, batch) | Freshness + Correctness | Throughput | `fresh_outputs / expected_outputs` |
| **Storage** (DB, cache) | Durability + Availability | Latency | `successful_ops / total_ops` |
| **Streaming** (Kafka, Kinesis) | Freshness + Throughput | Error rate | `msgs_delivered_on_time / msgs_produced` |

### SLI Specification Template

```
PARA CADA SLI, DEFINA:

1. SLI Specification (o que queremos medir, linguagem natural):
   "A proporção de requests HTTP que completam com sucesso 
    em menos de 300ms, medido no load balancer"

2. SLI Implementation (como medir, fonte de dados):
   Source: ALB access logs
   Good: status_code IN (200..299) AND response_time < 0.3
   Valid: todos os HTTP requests (excluindo health checks)
   Formula: count(good) / count(valid)

3. SLO Target:
   99.9% over 30-day rolling window

4. Owner:
   Team: order-platform
   Slack: #order-platform-oncall
```

### Onde medir o SLI

```
┌─────────────────────────────────────────────────────────────┐
│              ONDE MEDIR O SLI                                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  User ──▶ CDN ──▶ LB ──▶ API GW ──▶ Service ──▶ DB         │
│                                                              │
│  OPÇÃO 1: No CLIENT (Real User Monitoring - RUM)             │
│  ✅ Perspectiva real do usuário                               │
│  ❌ Difícil de coletar, noise de rede do usuário              │
│                                                              │
│  OPÇÃO 2: No LOAD BALANCER ← RECOMENDADO para APIs          │
│  ✅ Mais próximo do usuário que temos server-side             │
│  ✅ Fácil de coletar (ALB access logs)                        │
│  ✅ Inclui latência do serviço completo                       │
│  ❌ Não inclui latência de rede do usuário                    │
│                                                              │
│  OPÇÃO 3: No SERVIÇO (application metrics)                   │
│  ✅ Controle total, custom metrics                            │
│  ❌ Não inclui overhead do LB/API Gateway                     │
│  ❌ Se o serviço está down, não há métrica                    │
│                                                              │
│  OPÇÃO 4: SYNTHETIC (probes)                                 │
│  ✅ Baseline consistente                                      │
│  ✅ Detecta problemas antes dos usuários                      │
│  ❌ Não representa tráfego real                               │
│                                                              │
│  RECOMENDAÇÃO:                                                │
│  • SLI primário: Load Balancer logs/metrics                  │
│  • SLI complementar: Synthetic monitoring (canary)            │
│  • SLI de negócio: Application metrics                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Exemplos concretos de SLIs

#### API / Microserviço

```
AVAILABILITY SLI:
  good_events: HTTP responses with status < 500
  valid_events: all HTTP requests (excluding health checks)
  SLI = count(status < 500) / count(all - health_checks)

LATENCY SLI:
  good_events: HTTP responses with duration < 300ms
  valid_events: all successful HTTP requests (status < 500)
  SLI = count(duration < 300ms AND status < 500) / count(status < 500)

NOTA: Latency SLI exclui errors porque request que falhou
      em 1ms não é "rápida" — é uma falha rápida.
```

#### Data Pipeline (ETL)

```
FRESHNESS SLI:
  good_events: pipeline runs where data_age < 2h
  valid_events: all scheduled pipeline runs
  SLI = count(data_age < 2h) / count(scheduled_runs)

CORRECTNESS SLI:
  good_events: pipeline outputs que passam quality checks
  valid_events: all pipeline outputs
  SLI = count(quality_passed) / count(outputs)

COVERAGE SLI:
  good_events: expected datasets que foram gerados
  valid_events: total expected datasets
  SLI = count(generated) / count(expected)
```

#### Streaming (Kafka/Kinesis)

```
FRESHNESS SLI:
  good_events: messages delivered with lag < 30s
  valid_events: all messages produced
  SLI = count(lag < 30s) / count(produced)

THROUGHPUT SLI:
  good_events: messages successfully consumed
  valid_events: all messages produced
  SLI = count(consumed) / count(produced)
```

---

## Definindo SLOs — O target

### A tabela dos noves

| SLO | Downtime/mês | Downtime/ano | Nível |
|-----|-------------|-------------|-------|
| 99% | 7h 18min | 3.65 dias | Batch jobs, ferramentas internas |
| 99.5% | 3h 39min | 1.83 dias | Serviços internos não-críticos |
| 99.9% | 43min 50s | 8h 46min | A maioria dos serviços de produção |
| 99.95% | 21min 55s | 4h 23min | Serviços críticos de negócio |
| 99.99% | 4min 23s | 52min 36s | Infraestrutura core (DNS, auth) |
| 99.999% | 26s | 5min 16s | Quase impossível sem redundância massiva |

### Como escolher o SLO certo

```
┌─────────────────────────────────────────────────────────────┐
│           DECISION FRAMEWORK — QUAL SLO?                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PERGUNTE:                                                   │
│                                                              │
│  1. "O que acontece se este serviço fica fora por 1h?"      │
│     → Ninguém nota     → 99%                                │
│     → Time reclama     → 99.5%                              │
│     → Clientes ligam   → 99.9%                              │
│     → Dinheiro perdido → 99.95%                             │
│     → Regulatório/safety → 99.99%                           │
│                                                              │
│  2. "Qual a capacidade real do sistema hoje?"                │
│     → Se hoje é 99.7%, não coloque SLO de 99.99%            │
│     → SLO = algo acima do histórico, mas alcançável          │
│     → SLO agressivo demais = violado sempre = ignorado       │
│                                                              │
│  3. "Estamos dispostos a investir para atingir?"             │
│     → 99% → 99.9% = effort moderado                         │
│     → 99.9% → 99.99% = effort 10x (redundância, DR, etc.)  │
│     → 99.99% → 99.999% = effort 100x                        │
│                                                              │
│  REGRA PRÁTICA:                                              │
│  • Comece com 99.9% para a maioria dos serviços              │
│  • Meça 4 semanas antes de definir SLO                       │
│  • SLO = P10 do histórico (melhor 10% das semanas)           │
│  • Revise trimestralmente                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### SLO Composite — Dependências

```
Se Service A depende de B e C (em série):

Service A SLO = min(SLO_A_own, SLO_B, SLO_C)

EXEMPLO:
  A (own logic):     99.99%
  B (payment):       99.9%
  C (database):      99.95%
  
  A effective SLO ≤ 99.9% (limitado pelo componente mais fraco)

IMPLICAÇÃO:
  Se você precisa de 99.99% end-to-end, CADA componente precisa de 99.999%+
  
  Ou: Design para tolerar falhas (retries, fallbacks, circuit breakers)
      para que SLO_effective > min(SLOs)
```

---

## Error Budgets — A matemática

### O que é Error Budget

```
┌─────────────────────────────────────────────────────────────┐
│                     ERROR BUDGET                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Error Budget = 1 - SLO target                               │
│                                                              │
│  Se SLO = 99.9%:                                             │
│    Error Budget = 0.1%                                       │
│                                                              │
│  EM 30 DIAS:                                                 │
│    Total minutos = 30 × 24 × 60 = 43,200 minutos            │
│    Error Budget = 43,200 × 0.1% = 43.2 minutos              │
│                                                              │
│  EM REQUESTS (se 1M req/dia):                                │
│    Total requests/mês = 30M                                  │
│    Error Budget = 30M × 0.1% = 30,000 failed requests       │
│                                                              │
│  SIGNIFICADO:                                                │
│  "Podemos ter até 43 minutos de downtime OU                  │
│   30,000 requests falhando no mês e ainda estar              │
│   dentro do nosso SLO."                                      │
│                                                              │
│  ┌──────────────────────────────────────────┐               │
│  │ Budget 100%  ████████████████████████████ │               │
│  │ Incident -5  ██████████████████████░░░░░░ │               │
│  │ Deploy -2    ████████████████████░░░░░░░░ │               │
│  │ Incident -10 ██████████░░░░░░░░░░░░░░░░░░ │               │
│  │ Remaining    ██████████ 40% left          │               │
│  └──────────────────────────────────────────┘               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Error Budget como ferramenta de decisão

```
ERROR BUDGET = VELOCIDADE DE INOVAÇÃO

Budget > 50% restante:
  → Deploy com confiança
  → Experimentos permitidos
  → Risk-tolerant changes
  
Budget 20-50% restante:
  → Deploy com cautela
  → Canary releases obrigatórios
  → Rollback automático configurado
  
Budget < 20% restante:
  → Apenas deploys de bug fix
  → Feature freeze
  → Foco em reliability
  
Budget = 0% (esgotado):
  → FREEZE completo de deploys
  → Apenas emergency fixes
  → Post-mortem obrigatório
  → Reliability sprints (próximo sprint = 100% resiliência)
```

### Cálculos úteis

```
BURN RATE = velocidade de consumo do budget

Burn Rate = (error_rate_current / error_budget_rate_threshold)

Se SLO = 99.9% (budget = 0.1%):
  Budget rate "normal" = 0.1% / 30 dias = 0.00333%/dia

  Error rate atual = 0.5%
  Burn Rate = 0.5% / 0.00333% = 150x ← consumindo 150x mais rápido!
  
  Time to exhaust = 30 / 150 = 0.2 dias = ~4.8 horas

─────────────────────────────────────

REMAINING BUDGET:

  remaining_budget = error_budget - consumed_budget
  
  consumed_budget = Σ(bad_minutes) ao longo do período
  
  remaining_pct = remaining_budget / error_budget × 100%
```

---

## Error Budget Policies — Governança

### Template de política

```
┌─────────────────────────────────────────────────────────────┐
│              ERROR BUDGET POLICY                             │
│              (aprovada por Eng + Product + SRE)              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  SERVIÇO: order-service                                      │
│  SLO: 99.9% availability, 99.5% latency < 300ms             │
│  PERÍODO: 30-day rolling window                              │
│  REVISÃO: Trimestral                                         │
│                                                              │
│  QUANDO BUDGET > 50%:                                        │
│  • Feature development normal                                │
│  • Deploys diários permitidos                                │
│  • Experimentation permitida                                 │
│                                                              │
│  QUANDO BUDGET 20-50%:                                       │
│  • Canary deploy obrigatório (10% → 50% → 100%)             │
│  • Rollback automático em SLI drop > 0.5%                    │
│  • On-call reviewes cada deploy                              │
│  • Post-mortem para incidents que queimaram > 10% budget     │
│                                                              │
│  QUANDO BUDGET < 20%:                                        │
│  • Feature freeze                                            │
│  • Apenas bug fixes e reliability improvements               │
│  • VP Eng approval para qualquer deploy não-fix              │
│  • Reliability sprint obrigatório                            │
│                                                              │
│  QUANDO BUDGET = 0%:                                         │
│  • Deploy freeze total                                       │
│  • Apenas P1 emergency fixes                                 │
│  • Blameless post-mortem obrigatório em 48h                  │
│  • Improvement plan com timeline                             │
│  • Escalar para Engineering Leadership                       │
│                                                              │
│  EXCEÇÕES:                                                   │
│  • Security patches: sempre permitidos                       │
│  • Regulatory requirements: sempre permitidos                │
│  • Revenue-critical launches: VP Eng + SRE Director approval │
│                                                              │
│  ASSINADO POR:                                               │
│  [ ] Engineering Manager                                     │
│  [ ] Product Manager                                         │
│  [ ] SRE Team Lead                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Multi-Window, Multi-Burn-Rate Alerting

### O problema com alerting simples

```
ALERTA SIMPLES: "error_rate > 0.1% por 5 minutos"

PROBLEMAS:
1. False positive: blip de 6min com 0.2% → alerta → resolve sozinho
2. False negative: 0.05% constante por 30 dias → queima todo budget silenciosamente
3. Sem contexto de urgência: 0.2% e 5% geram o mesmo alerta

─────────────────────────────────────────────────

SOLUÇÃO: Multi-Window, Multi-Burn-Rate Alerting (Google SRE)
Baseado em: "A qual VELOCIDADE estamos consumindo o error budget?"
```

### A tabela mágica

```
┌─────────────────────────────────────────────────────────────┐
│       MULTI-WINDOW MULTI-BURN-RATE ALERTING                  │
│       (para SLO de 99.9% sobre 30 dias)                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Burn   │ Long    │ Short   │ Budget  │ Time to  │ Sev     │
│  Rate   │ Window  │ Window  │ consumed│ exhaust  │         │
│  ───────┼─────────┼─────────┼─────────┼──────────┼─────────│
│  14.4x  │ 1h      │ 5min    │ 2%      │ ~2h      │ P1 PAGE│
│  6x     │ 6h      │ 30min   │ 5%      │ ~5d      │ P2 PAGE│
│  3x     │ 3d      │ 6h      │ 10%     │ ~10d     │ P3 TICK│
│  1x     │ 30d     │ 3d      │ 100%    │ ~30d     │ P4 LOG │
│                                                              │
│  LÓGICA:                                                     │
│  ALERTA dispara quando:                                      │
│    burn_rate(long_window) > threshold                        │
│    AND                                                       │
│    burn_rate(short_window) > threshold                       │
│                                                              │
│  POR QUÊ DUAS JANELAS?                                      │
│  • Long window: confirma que não é spike momentâneo          │
│  • Short window: confirma que o problema é AGORA             │
│  • Juntas: reduzem false positives drasticamente             │
│                                                              │
│  EXEMPLO — P1 (14.4x burn rate):                             │
│  Trigger: error_rate > 14.4 × (0.1%/30d) na última 1h      │
│           AND error_rate > 14.4 × (0.1%/30d) nos últimos 5m │
│  = error_rate > 0.48% sustentado por 1h E ativo agora        │
│  = "Em ~2h vamos queimar 100% do budget → PAGE!"            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Por que esses números?

```
BURN RATE 14.4x:
  Budget consumido em 1h window: 14.4 × (1h/720h) = 2%
  → 2% do budget em 1h é sério → PAGE imediato
  → Time to exhaust: 720h / 14.4 = 50h (~2 dias)

BURN RATE 6x:
  Budget consumido em 6h window: 6 × (6h/720h) = 5%
  → 5% em 6h é preocupante → PAGE para on-call
  
BURN RATE 3x:
  Budget consumido em 3d window: 3 × (72h/720h) = 30%
  → 30% em 3 dias → ticket para próximo sprint

BURN RATE 1x:
  → Consumo normal do budget ao longo do mês
  → Apenas tracking, não alerta
```

---

## SLO-Based Alerting — Implementação

### Pseudocode — Prometheus/Grafana

```yaml
# Prometheus alerting rules — SLO-based
groups:
  - name: slo-order-service
    rules:
      # ─── SLI: Availability ───
      # Good events: non-5xx responses
      # Total events: all HTTP requests
      
      # Recording rules para SLI
      - record: sli:http_requests:availability
        expr: |
          sum(rate(http_requests_total{job="order-service", status!~"5.."}[5m]))
          /
          sum(rate(http_requests_total{job="order-service"}[5m]))

      # ─── P1: 14.4x burn rate ───
      # Long window (1h) AND Short window (5m)
      - alert: SLOBurnRateCritical
        expr: |
          (
            1 - (sum(rate(http_requests_total{job="order-service", status!~"5.."}[1h]))
                 / sum(rate(http_requests_total{job="order-service"}[1h])))
          ) > (14.4 * 0.001)
          AND
          (
            1 - (sum(rate(http_requests_total{job="order-service", status!~"5.."}[5m]))
                 / sum(rate(http_requests_total{job="order-service"}[5m])))
          ) > (14.4 * 0.001)
        for: 2m
        labels:
          severity: critical
          slo: order-service-availability
        annotations:
          summary: "Order Service SLO burn rate is 14.4x (P1)"
          description: "Error budget will be exhausted in ~2h at current rate"
          runbook: "https://wiki.internal/runbooks/order-service-slo"

      # ─── P2: 6x burn rate ───
      - alert: SLOBurnRateHigh
        expr: |
          (
            1 - (sum(rate(http_requests_total{job="order-service", status!~"5.."}[6h]))
                 / sum(rate(http_requests_total{job="order-service"}[6h])))
          ) > (6 * 0.001)
          AND
          (
            1 - (sum(rate(http_requests_total{job="order-service", status!~"5.."}[30m]))
                 / sum(rate(http_requests_total{job="order-service"}[30m])))
          ) > (6 * 0.001)
        for: 5m
        labels:
          severity: warning
          slo: order-service-availability
        annotations:
          summary: "Order Service SLO burn rate is 6x (P2)"
          description: "Error budget will be exhausted in ~5d at current rate"
```

### Pseudocode — CloudWatch (AWS-native)

```python
# Pseudocode — SLO monitoring com CloudWatch
import boto3

cloudwatch = boto3.client('cloudwatch')

# Publicar custom metric: SLI availability
def publish_sli_metric(good_count, total_count):
    sli = good_count / total_count if total_count > 0 else 1.0
    
    cloudwatch.put_metric_data(
        Namespace='SLO/OrderService',
        MetricData=[
            {
                'MetricName': 'AvailabilitySLI',
                'Value': sli * 100,  # Percentual
                'Unit': 'Percent',
                'Dimensions': [
                    {'Name': 'Service', 'Value': 'order-service'},
                    {'Name': 'Environment', 'Value': 'production'}
                ]
            },
            {
                'MetricName': 'ErrorBudgetRemaining',
                'Value': calculate_remaining_budget(sli),
                'Unit': 'Percent',
                'Dimensions': [
                    {'Name': 'Service', 'Value': 'order-service'},
                    {'Name': 'Environment', 'Value': 'production'}
                ]
            }
        ]
    )

# CloudWatch Alarm — SLO burn rate
cloudwatch.put_metric_alarm(
    AlarmName='OrderService-SLO-BurnRate-P1',
    MetricName='AvailabilitySLI',
    Namespace='SLO/OrderService',
    Statistic='Average',
    Period=300,  # 5 min
    EvaluationPeriods=12,  # 1h (12 × 5min)
    Threshold=99.9 - (14.4 * 0.1),  # 99.9 - 1.44 = 98.46%
    ComparisonOperator='LessThanThreshold',
    AlarmActions=['arn:aws:sns:region:account:pagerduty-critical'],
    TreatMissingData='breaching'
)
```

---

## SLO para Diferentes Tipos de Sistema

### SLOs recomendados por tipo

| Tipo de sistema | SLI primário | SLO recomendado | SLI secundário | SLO |
|----------------|-------------|-----------------|----------------|-----|
| **API pública** | Availability | 99.9% | Latency P99 < 500ms | 99.5% |
| **API interna** | Availability | 99.5% | Latency P99 < 1s | 99% |
| **Payment service** | Availability | 99.95% | Latency P99 < 2s | 99.9% |
| **Auth service** | Availability | 99.99% | Latency P99 < 100ms | 99.9% |
| **Data pipeline** | Freshness < 2h | 99% | Correctness | 99.9% |
| **Streaming** | Freshness < 30s | 99.5% | Throughput | 99% |
| **Search API** | Availability | 99.9% | Latency P50 < 100ms | 99% |
| **Notification** | Delivery | 99.5% | Latency < 5min | 95% |
| **Background jobs** | Completion | 99% | Freshness < 1h | 95% |
| **CDN / Static** | Availability | 99.99% | Latency P50 < 50ms | 99.9% |

### SLOs compostos — Exemplo e-commerce

```
┌─────────────────────────────────────────────────────────────┐
│            E-COMMERCE — SLO MAP                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  User Journey: "Comprar um produto"                          │
│                                                              │
│  1. Browse catalog     → Catalog API SLO: 99.9% / 200ms    │
│  2. Search products    → Search API SLO: 99.9% / 100ms     │
│  3. View product       → Product API SLO: 99.9% / 150ms    │
│  4. Add to cart        → Cart API SLO: 99.95% / 300ms      │
│  5. Checkout           → Checkout SLO: 99.95% / 2s         │
│  6. Payment            → Payment SLO: 99.99% / 5s          │
│  7. Order confirmation → Order API SLO: 99.9% / 500ms      │
│  8. Email notification → Notif SLO: 99.5% / 5min           │
│                                                              │
│  Journey SLO (end-to-end):                                   │
│  availability = min(99.9, 99.9, 99.9, 99.95, 99.95,        │
│                     99.99, 99.9, 99.5)                       │
│             ≈ 99.5% (sem redundância)                       │
│             ≈ 99.9% (com retries + fallbacks)                │
│                                                              │
│  Bottleneck: Notification (99.5%) — mas é async,             │
│  não impacta o checkout completion SLO                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## SLO Dashboard Design

### Elementos obrigatórios

```
┌─────────────────────────────────────────────────────────────┐
│           ORDER SERVICE — SLO DASHBOARD                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │ AVAILABILITY    │  │ LATENCY P99     │  │ ERROR BUDGET│ │
│  │ SLI: 99.97%     │  │ SLI: 99.2%      │  │ Remaining:  │ │
│  │ SLO: 99.9%  ✅  │  │ SLO: 99.5%  ❌  │  │ 62%  ████░░│ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  ERROR BUDGET BURN (30-day rolling)                   │   │
│  │  100%│████████████████████████████                    │   │
│  │      │                         ╲                      │   │
│  │   50%│                           ╲  (incident)        │   │
│  │      │                             ╲                  │   │
│  │    0%│────────────────────────────────────────  Day   │   │
│  │      1    5    10   15   20   25   30                 │   │
│  │                                                       │   │
│  │  ── Ideal burn (linear)   ╲╲ Actual burn             │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  SLI OVER TIME (7d)                                   │   │
│  │  100%│─────────────── SLO=99.9% ──────────────────   │   │
│  │      │  ╱╲  ╱╲                      ╱╲               │   │
│  │ 99.9%│╱    ╲╱  ╲────────────────╱╲╱  ╲──────        │   │
│  │      │           ╲            ╱                       │   │
│  │ 99.8%│            ╲─── incident ──╱                   │   │
│  │      │─────────────────────────────────────── Time    │   │
│  │       Mon  Tue  Wed  Thu  Fri  Sat  Sun               │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  BURN RATE ALERTS (active):                                  │
│  🟡 P2: 6x burn rate detected (last 6h)                    │
│                                                              │
│  RECENT CHANGES:                                             │
│  📌 Deploy v2.3.1 — 2h ago                                  │
│  📌 Config: pool_size 10→50 — 6h ago                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## SLO Review Process

### Cadência trimestral

```
QUARTERLY SLO REVIEW AGENDA:

1. SLO STATUS (10 min)
   • Quantas vezes SLO foi violado no trimestre?
   • Error budget consumido vs disponível?
   • Tendência: melhorando ou piorando?

2. SLO CALIBRAÇÃO (15 min)
   • SLO muito fácil? (nunca violado) → Apertar
   • SLO muito agressivo? (sempre violado) → Relaxar
   • Novos SLIs necessários? (journey SLIs?)

3. ERROR BUDGET POLICY REVIEW (10 min)
   • Policy foi seguida?
   • Freezes foram efetivos?
   • Exceções usadas? Justificadas?

4. INCIDENT REVIEW (10 min)
   • Top 3 incidents que mais queimaram budget
   • Action items dos post-mortems: concluídos?

5. ACTION ITEMS (15 min)
   • Ajustes no SLO target
   • Novos alertas necessários
   • Reliability improvements a priorizar

PARTICIPANTES: Service owner, SRE, Product, Engineering Manager
```

---

## Anti-Patterns de SLO

| Anti-pattern | Problema | Solução |
|-------------|---------|---------|
| **SLO = 100%** | Impossível de atingir, equipe desmoralizada | Target realista: 99.9% ou 99.95% |
| **SLO sem medição** | "Nosso SLO é 99.9%" mas ninguém mede | Implementar SLI metrics + dashboards primeiro |
| **SLO = SLA** | Sem margem de segurança | SLO > SLA (ex: SLO=99.95%, SLA=99.9%) |
| **SLO sem policy** | SLO violou e nada acontece | Error budget policy com ações concretas |
| **SLO de infra** | "CPU < 80%" como SLO | SLI = user-facing metric (availability, latency) |
| **Muitos SLOs** | 20 SLOs por serviço | 2-3 SLOs por serviço (availability + latency + um de negócio) |
| **SLO não revisado** | Definiu uma vez, nunca ajustou | Revisão trimestral obrigatória |
| **Budget ignorado** | Deploys mesmo com budget zerado | Policy enforced, VP approval para exceções |
| **Latency only P50** | "P50 < 100ms" → 50% dos users lentos | Use P99 (ou pelo menos P95) para SLO |
| **Window mismatch** | SLO de 30d, alerta de 5min | Multi-window alerting |

---

## Implementação Prática — Passo a Passo

### Semana 1-2: Foundation

```
STEP 1: Identificar SLIs
  • Liste as 3-5 operações mais importantes do serviço
  • Para cada uma: qual é o "good event" e o "total event"?
  
STEP 2: Instrumentar SLIs
  • Adicionar métricas (counters/histograms) para good/total
  • Ou: configurar log-based SLIs (ALB logs, CloudWatch Logs Insights)

STEP 3: Medir baseline
  • Rodar por 2-4 semanas sem SLO definido
  • Calcular: P50, P90, P99 do SLI atual
```

### Semana 3-4: Define & Alert

```
STEP 4: Definir SLO
  • Target = algo acima do P10 histórico mas alcançável
  • Período = 30-day rolling window
  • Documentar SLO specification

STEP 5: Configurar alertas
  • Multi-window multi-burn-rate (P1 + P2 + P3)
  • Vincular alertas a runbooks

STEP 6: Dashboard
  • SLI atual vs SLO target
  • Error budget remaining
  • Burn rate over time
```

### Mês 2-3: Policy & Culture

```
STEP 7: Error budget policy
  • Escrever policy com Product + Eng + SRE
  • Definir ações para cada faixa de budget

STEP 8: Integrar no processo
  • Deploy pipeline checa error budget antes de deploy
  • Sprint planning considera reliability debt
  • Incident post-mortems referenciam impacto no SLO

STEP 9: Review
  • Primeira revisão após 1 mês
  • Ajustar SLOs e policies baseado em dados reais
  • Cadência trimestral estabelecida
```

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e configurações relacionadas a SLOs/SLIs, verifique:

1. **SLI mede infra, não user experience** — CPU, memory não são SLIs; availability, latency sim
2. **SLO = 100%** — Nunca; sugira 99.9% ou 99.95% conforme criticidade
3. **Sem error budget calculation** — Toda config de SLO deve ter error budget derivado
4. **Alerta em valor instantâneo** — `error_rate > 0.1%` → substitua por burn rate multi-window
5. **Apenas P50 para latency SLI** — Exija P95 ou P99 como SLI (P50 esconde outliers)
6. **SLA = SLO** — SLO deve ser mais restritivo que SLA (margem de segurança)
7. **SLO sem dashboard** — Toda SLO definition deve ter dashboard associado
8. **Alerta sem runbook** — Todo alerta de SLO deve ter link para runbook
9. **SLI no serviço, não no LB** — Prefira medir SLI no load balancer (mais próximo do user)
10. **Error budget sem policy** — Budget sem policy = número sem ação
11. **Window inconsistente** — Se SLO é 30d, alertas devem usar multi-window (1h+5m, 6h+30m, etc.)
12. **Health check contabilizado** — Exclua health check endpoints do cálculo de SLI

---

## Referências

- **Site Reliability Engineering** — Google (Chapter 4: Service Level Objectives)
- **The Site Reliability Workbook** — Google (Chapter 2: Implementing SLOs)
- **Implementing Service Level Objectives** — Alex Hidalgo (O'Reilly)
- **SLO Alerting for Mortals** — Google Cloud Blog
- **Multi-Window Multi-Burn-Rate Alerts** — Google SRE Workbook, Chapter 5
- **OpenSLO Specification** — https://openslo.com/
- **Sloth** — SLO generator for Prometheus — https://github.com/slok/sloth
- **Google Cloud SLO Monitoring** — https://cloud.google.com/stackdriver/docs/solutions/slo-monitoring
