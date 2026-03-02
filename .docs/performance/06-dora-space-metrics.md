# Performance Engineering — DORA Metrics & SPACE Framework

> **Objetivo deste documento:** Servir como referência completa sobre **DORA Metrics** e **SPACE Framework** — métricas de performance de engenharia de software, como medir, instrumentar, interpretar e usar dados para melhorar delivery e developer experience, otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: DORA 4 key metrics + reliability metric, SPACE framework (5 dimensões), instrumentação prática, dashboards, benchmarks, anti-patterns e integração com cultura de engenharia.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **Deployment Frequency** | Mais frequente = menor risco por deploy | "Deploy semanal é seguro" | Batches grandes = mais risco. Daily+ é o target |
| **Lead Time for Changes** | Commit → produção deve ser < 1 dia (elite) | "Nosso lead time é 3 semanas" | Automatizar pipeline, reduzir reviews pendentes |
| **Change Failure Rate** | < 5% das mudanças causam falhas | "15% de rollback é normal" | Testes, canary, feature flags, deploy incremental |
| **Mean Time to Recovery** | < 1 hora para restaurar serviço | "Demorou 6h para resolver" | Runbooks, observabilidade, rollback automatizado |
| **Reliability** | SLO como contrato, não "100% uptime" | "Nunca tivemos downtime" | Defina SLOs, meça SLIs, gerencie error budgets |
| **SPACE** | Nenhuma dimensão sozinha conta a história | "Lines of code = produtividade" | Combine múltiplas dimensões, sempre |
| **Satisfaction** | Developer experience importa para retenção | Ignorar surveys, focar só em métricas | Pesquisas regulares + ação nos resultados |
| **Flow** | Interrupções matam deep work | "Sempre disponível no Slack" | Focus time blocks, async communication |
| **Goodhart's Law** | Quando métrica vira target, deixa de ser boa métrica | Gamificar commits para subir DF | Métricas para entender, não para cobrar |

---

## Sumário

- [Performance Engineering — DORA Metrics \& SPACE Framework](#performance-engineering--dora-metrics--space-framework)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que são DORA Metrics](#o-que-são-dora-metrics)
    - [DORA vs Métricas Tradicionais](#dora-vs-métricas-tradicionais)
  - [As 4+1 Métricas DORA](#as-41-métricas-dora)
    - [1. Deployment Frequency (DF)](#1-deployment-frequency-df)
    - [2. Lead Time for Changes (LT)](#2-lead-time-for-changes-lt)
    - [3. Change Failure Rate (CFR)](#3-change-failure-rate-cfr)
    - [4. Mean Time to Recovery (MTTR)](#4-mean-time-to-recovery-mttr)
    - [5. Reliability (5ª métrica DORA)](#5-reliability-5ª-métrica-dora)
  - [Benchmarks e Classificação DORA](#benchmarks-e-classificação-dora)
    - [Correlações Importante (da Pesquisa)](#correlações-importante-da-pesquisa)
  - [Instrumentação DORA — Como Medir](#instrumentação-dora--como-medir)
    - [Deployment Frequency — Automação](#deployment-frequency--automação)
    - [Lead Time — Cálculo Automatizado](#lead-time--cálculo-automatizado)
    - [Change Failure Rate — Correlação Deploy × Incidente](#change-failure-rate--correlação-deploy--incidente)
    - [MTTR — Timeline de Incidente](#mttr--timeline-de-incidente)
  - [SPACE Framework](#space-framework)
  - [As 5 Dimensões do SPACE](#as-5-dimensões-do-space)
    - [S — Satisfaction \& Well-being](#s--satisfaction--well-being)
    - [P — Performance](#p--performance)
    - [A — Activity](#a--activity)
    - [C — Communication \& Collaboration](#c--communication--collaboration)
    - [E — Efficiency \& Flow](#e--efficiency--flow)
  - [Combinando DORA + SPACE](#combinando-dora--space)
  - [Dashboards e Visualização](#dashboards-e-visualização)
  - [Anti-Patterns de Métricas](#anti-patterns-de-métricas)
  - [DORA e Cultura de Engenharia](#dora-e-cultura-de-engenharia)
  - [Instrumentação Prática — Automação](#instrumentação-prática--automação)
    - [GitHub Actions — Coletando DORA Events](#github-actions--coletando-dora-events)
    - [Prometheus Metrics — Exposição de DORA](#prometheus-metrics--exposição-de-dora)
    - [Grafana Recording Rules — DORA Calculations](#grafana-recording-rules--dora-calculations)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que são DORA Metrics

```
DORA = DevOps Research and Assessment

  Origem:
  ├── Pesquisa de Nicole Forsgren, Jez Humble e Gene Kim
  ├── 7+ anos de pesquisa com 36.000+ profissionais
  ├── Publicado em "Accelerate" (2018)
  └── Mantido pelo Google Cloud DORA Team (dora.dev)

  O QUE as DORA metrics NÃO são:
  ├── NÃO são KPIs para individual performance
  ├── NÃO são targets para cobrar de equipes
  ├── NÃO são substitutas de todas as métricas
  └── NÃO funcionam sem contexto (tamanho, domínio, maturidade)

  O QUE as DORA metrics SÃO:
  ├── Indicadores de saúde do proceso de delivery
  ├── Proxies para capabilities técnicas e organizacionais
  ├── Ferramenta de diagnóstico (onde melhorar)
  └── Linguagem comum entre engenharia e negócio

  PRINCÍPIO FUNDAMENTAL:
  ┌────────────────────────────────────────────────────────────────┐
  │  "Velocidade e estabilidade NÃO são trade-offs."              │
  │  Equipes elite são SIMULTANEAMENTE rápidas e estáveis.        │
  │  Investir em CI/CD, testing e observabilidade melhora AMBOS.  │
  └────────────────────────────────────────────────────────────────┘
```

### DORA vs Métricas Tradicionais

```
MÉTRICAS TRADICIONAIS (problemáticas):

  ┌──────────────────────────────────────────────────────────────┐
  │ Métrica                  │ Problema                          │
  │ Lines of code            │ Incentiva verbosidade             │
  │ Story points delivered   │ Inflação, não mede valor          │
  │ Bugs found               │ Incentiva reportar mais, não fix  │
  │ Hours worked             │ Burnout, presenteísmo             │
  │ Commits per day          │ Commits pequenos sem valor        │
  │ PRs merged               │ PRs minúsculas para gaming       │
  └──────────────────────────────────────────────────────────────┘

DORA METRICS (baseadas em outcomes):

  ┌──────────────────────────────────────────────────────────────┐
  │ Métrica                  │ Mede                              │
  │ Deployment Frequency     │ Capacidade de entregar valor      │
  │ Lead Time for Changes    │ Eficiência do pipeline            │
  │ Change Failure Rate      │ Qualidade do que entregamos       │
  │ Mean Time to Recovery    │ Capacidade de se recuperar        │
  └──────────────────────────────────────────────────────────────┘

  VELOCIDADE:  Deployment Frequency + Lead Time
  ESTABILIDADE: Change Failure Rate + MTTR
```

---

## As 4+1 Métricas DORA

### 1. Deployment Frequency (DF)

```
DEFINIÇÃO:
  Frequência com que a organização faz deploys para produção
  com sucesso (para o serviço/aplicação principal).

COMO MEDIR:
  ┌────────────────────────────────────────────────────────────────┐
  │                                                                │
  │  Fonte: CI/CD pipeline (GitHub Actions, GitLab CI, Jenkins)    │
  │                                                                │
  │  Evento: Deploy bem-sucedido para produção                     │
  │                                                                │
  │  Fórmula:                                                      │
  │    DF = count(successful_deploys) / período                    │
  │                                                                │
  │  Granularidade:                                                │
  │    └── Per service/application (não aggregate)                 │
  │                                                                │
  │  Filtros:                                                      │
  │    ├── Apenas deploys para PRODUÇÃO (não staging)              │
  │    ├── Apenas deploys SUCCESSFUL (não rollback)                │
  │    └── Excluir deploys automatizados de config-only            │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘

POR QUE IMPORTA:
  ├── DF alta = batches menores = menos risco por deploy
  ├── DF alta = feedback loop mais rápido
  ├── DF baixa geralmente indica: testes frágeis, processo manual,
  │   medo de deploy, merge hell, falta de automação
  └── DF é leading indicator: se cair, outros metrics vão seguir

ARMADILHAS:
  ├── "Deploy" de hotfix de emergência inflaciona DF artificialmente
  ├── Monorepo com deploys acoplados distorce se contar como 1
  └── Config changes (feature flags) devem ser contados separadamente
```

### 2. Lead Time for Changes (LT)

```
DEFINIÇÃO:
  Tempo entre o primeiro commit de uma mudança e quando essa 
  mudança está rodando em produção com sucesso.

COMO MEDIR:
  ┌────────────────────────────────────────────────────────────────┐
  │                                                                │
  │  Fonte: VCS (Git) + CI/CD pipeline                             │
  │                                                                │
  │  Início: primeiro commit no branch da mudança                  │
  │  Fim: deploy bem-sucedido em produção                          │
  │                                                                │
  │  Fórmula:                                                      │
  │    LT = timestamp(deploy_prod) - timestamp(first_commit)       │
  │                                                                │
  │  Breakdown (onde o tempo é gasto):                             │
  │    ├── Coding time: first commit → PR opened                   │
  │    ├── Review time: PR opened → PR approved                    │
  │    ├── Merge time: PR approved → merged to main                │
  │    ├── Build time: merge → artifact built                      │
  │    ├── Test time: artifact → all tests passed                  │
  │    └── Deploy time: tests passed → running in prod             │
  │                                                                │
  │  Reportar:                                                     │
  │    ├── Mediana (p50) — typical experience                      │
  │    └── p90 — worst common case                                 │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘

POR QUE IMPORTA:
  ├── LT longo = feedback loop lento = bugs vivem mais tempo
  ├── LT longo = mudanças acumulam = deploys maiores = mais risco
  ├── LT é proxy para a saúde do pipeline inteiro
  └── Breakdown revela onde investir (review? build? tests?)

ARMADILHAS:
  ├── "First commit" pode ser antes de o dev começar realmente
  │    → Alguns medem "PR opened" como início alternativo
  ├── Hotfixes distorcem mediana para baixo
  └── Rebase/squash pode perder o timestamp original do commit
```

### 3. Change Failure Rate (CFR)

```
DEFINIÇÃO:
  Percentual de deploys para produção que resultam em falha
  (incidente, rollback, hotfix, ou fix-forward).

COMO MEDIR:
  ┌────────────────────────────────────────────────────────────────┐
  │                                                                │
  │  Fonte: Incident tracker + CI/CD pipeline                      │
  │                                                                │
  │  Fórmula:                                                      │
  │    CFR = count(deploys_que_causaram_falha) / count(all_deploys)│
  │                                                                │
  │  O que conta como "falha":                                     │
  │    ├── Incidente causado pelo deploy (qualquer severidade)     │
  │    ├── Rollback necessário                                     │
  │    ├── Hotfix necessário dentro de 24h                         │
  │    ├── Fix-forward feito para corrigir o deploy                │
  │    └── Degradação de SLO causada pelo deploy                   │
  │                                                                │
  │  O que NÃO conta como "falha":                                 │
  │    ├── Incidente não relacionado ao deploy                     │
  │    ├── Feature flag desabilitada proativamente                 │
  │    └── Degradação pré-existente                                │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘

POR QUE IMPORTA:
  ├── CFR alto = qualidade baixa no que entregamos
  ├── CFR alto + DF alta = muitos deploys, muitos problemas
  ├── CFR baixo + DF alta = sweet spot (elite)
  └── CFR é trailing indicator: reflete qualidade de testes, 
      reviews, e processo de release

DECOMPOSIÇÃO:
  ├── CFR por tipo: bug vs performance vs config vs infra
  ├── CFR por severidade: P1 vs P2 vs P3
  └── CFR por componente: frontend vs backend vs data
```

### 4. Mean Time to Recovery (MTTR)

```
DEFINIÇÃO:
  Tempo médio para restaurar o serviço quando ocorre uma falha
  ou degradação causada por uma mudança.

COMO MEDIR:
  ┌────────────────────────────────────────────────────────────────┐
  │                                                                │
  │  Fonte: Incident tracker + Alerting system                     │
  │                                                                │
  │  Início: incidente detectado (alert fired ou report recebido)  │
  │  Fim: serviço restaurado (SLI está dentro do SLO novamente)    │
  │                                                                │
  │  Fórmula:                                                      │
  │    MTTR = mean(recovery_time - detection_time)                 │
  │                                                                │
  │  Breakdown:                                                    │
  │    ├── MTTD (Time to Detect): falha → alert/report             │
  │    ├── MTTE (Time to Engage): alert → pessoa olhando           │
  │    ├── MTTF (Time to Fix): investigando → fix deployed         │
  │    └── MTTV (Time to Verify): fix deployed → SLI recovered     │
  │                                                                │
  │    MTTR = MTTD + MTTE + MTTF + MTTV                           │
  │                                                                │
  │  Reportar:                                                     │
  │    ├── Mediana (p50) — typical recovery                        │
  │    ├── p90 — worst common case                                 │
  │    └── Por severidade: MTTR-P1, MTTR-P2, MTTR-P3              │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘

POR QUE IMPORTA:
  ├── MTTR baixo = impacto limitado mesmo quando falha acontece
  ├── MTTR é mais controlável que MTBF (prevenir toda falha é impossível)
  ├── MTTR reflete: observabilidade, runbooks, on-call, deploy speed
  └── Investir em MTTR > investir em "zero downtime" (mais realista)

RELAÇÃO COM OUTRAS MÉTRICAS:
  ├── MTTR baixo compensa CFR moderado (falha rápido, recupera rápido)
  ├── DF alta ajuda MTTR: se pipeline é rápido, fix-forward é viável
  └── Observabilidade (alertas, dashboards, traces) reduz MTTD e MTTF
```

### 5. Reliability (5ª métrica DORA)

```
ADIÇÃO DO STATE OF DEVOPS 2022:

  ┌────────────────────────────────────────────────────────────────┐
  │  A 5ª métrica reconhece que delivery rápido sem              │
  │  confiabilidade não gera valor.                                │
  │                                                                │
  │  Reliability = O serviço atende suas SLOs?                    │
  │                                                                │
  │  Como medir:                                                   │
  │    ├── SLO compliance: % do tempo que SLIs estão dentro do SLO│
  │    ├── Error budget remaining: quanto budget resta             │
  │    └── Uptime ≠ reliability (disponível mas degradado ≠ OK)   │
  │                                                                │
  │  Integração com DORA:                                          │
  │    ├── DF + LT altos MAS reliability baixa = problema         │
  │    ├── CFR baixo MAS reliability baixa = falhas não-deploy     │
  │    └── Reliability como guardrail: deploy rápido SEM degradar │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘
```

---

## Benchmarks e Classificação DORA

```
STATE OF DEVOPS REPORT — CLASSIFICAÇÃO

  ┌──────────────────────────────────────────────────────────────────┐
  │ Métrica              │ Elite        │ High         │ Medium      │ Low           │
  ├──────────────────────┼──────────────┼──────────────┼─────────────┼───────────────┤
  │ Deployment Frequency │ On-demand    │ 1/semana -   │ 1/mês -     │ < 1/mês       │
  │                      │ (múltiplos/  │ 1/mês        │ 1/6 meses   │ (< 1 cada     │
  │                      │ dia)         │              │             │  6 meses)     │
  ├──────────────────────┼──────────────┼──────────────┼─────────────┼───────────────┤
  │ Lead Time for        │ < 1 dia      │ 1 dia -      │ 1 semana -  │ > 6 meses     │
  │ Changes              │              │ 1 semana     │ 1 mês       │               │
  ├──────────────────────┼──────────────┼──────────────┼─────────────┼───────────────┤
  │ Change Failure Rate  │ 0-5%         │ 5-10%        │ 10-15%      │ > 15%         │
  ├──────────────────────┼──────────────┼──────────────┼─────────────┼───────────────┤
  │ Mean Time to         │ < 1 hora     │ < 1 dia      │ 1 dia -     │ > 6 meses     │
  │ Recovery             │              │              │ 1 semana    │               │
  └──────────────────────┴──────────────┴──────────────┴─────────────┴───────────────┘

  NOTA: Benchmarks são referência, não targets absolutos.
  Contexto importa: startup vs enterprise, greenfield vs legacy.
```

### Correlações Importante (da Pesquisa)

```
ACHADOS ESTATISTICAMENTE SIGNIFICATIVOS:

  1. VELOCIDADE + ESTABILIDADE:
     Equipes elite são 973x mais rápidas em DF e 6570x mais 
     rápidas em LT que equipes low, MAS também têm 3x menos 
     CFR e 6570x MTTR mais rápido.
     → Velocidade e estabilidade NÃO são trade-offs.

  2. CAPABILITIES QUE DRIVE DORA:
     ├── Technical:
     │    ├── CI/CD automation (strongest predictor)
     │    ├── Trunk-based development
     │    ├── Automated testing
     │    ├── Loosely coupled architecture
     │    ├── Continuous monitoring/observability
     │    └── Database change management
     ├── Process:
     │    ├── Work in small batches
     │    ├── Visibility of work in progress
     │    ├── Lightweight change approval
     │    └── Working in small, autonomous teams
     └── Cultural:
          ├── Generative culture (Westrum)
          ├── Learning from failures
          └── Psychological safety

  3. BUSINESS IMPACT:
     Equipes elite performers têm:
     ├── 2x probabilidade de atingir metas de negócio
     ├── 50% mais probabilidade de exceder metas de rentabilidade
     └── Menor burnout e maior satisfação no trabalho
```

---

## Instrumentação DORA — Como Medir

### Deployment Frequency — Automação

```
FONTES DE DADOS:

  GitHub Actions / GitLab CI / Jenkins:
  ┌────────────────────────────────────────────────────────────────┐
  │  Pipeline event: status = "success"                            │
  │  Environment: "production"                                     │
  │  Timestamp: completion_time                                    │
  │                                                                │
  │  Extrair:                                                      │
  │    ├── GitHub: workflow_run completed + environment = prod      │
  │    ├── GitLab: deployment event via webhook                     │
  │    └── Jenkins: build result = SUCCESS + deploy stage           │
  │                                                                │
  │  Armazenar:                                                    │
  │    {                                                           │
  │      "service": "order-service",                               │
  │      "version": "1.5.0",                                       │
  │      "environment": "production",                              │
  │      "deployed_at": "2026-01-15T14:30:00Z",                   │
  │      "status": "success",                                      │
  │      "triggered_by": "merge_to_main",                          │
  │      "pipeline_id": "abc123"                                   │
  │    }                                                           │
  └────────────────────────────────────────────────────────────────┘
```

### Lead Time — Cálculo Automatizado

```
PIPELINE DE CÁLCULO:

  ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐
  │  Commit   │──>│  PR Open  │──>│  Merge    │──>│  Deploy   │
  │  criado   │   │           │   │  to main  │   │  to prod  │
  │           │   │           │   │           │   │           │
  │ t₁        │   │ t₂        │   │ t₃        │   │ t₄        │
  └───────────┘   └───────────┘   └───────────┘   └───────────┘

  Lead Time Total = t₄ - t₁

  Breakdown:
    Coding Time  = t₂ - t₁ (commit → PR)
    Review Time  = t₃ - t₂ (PR → merge)
    Pipeline Time = t₄ - t₃ (merge → deploy)

  FONTES:
    ├── t₁: Git log (first commit in branch)
    ├── t₂: GitHub API (PR created_at)
    ├── t₃: GitHub API (PR merged_at)
    └── t₄: CI/CD (deployment completed_at)
```

### Change Failure Rate — Correlação Deploy × Incidente

```
CORRELAÇÃO:

  Para cada deploy D:
    ├── Verificar se houve incidente I onde:
    │    I.start_time > D.deploy_time
    │    AND I.start_time < D.deploy_time + 24h  (janela)
    │    AND I.related_service = D.service
    │    AND I.cause = "change" (não infra externa)
    │
    ├── OU houve rollback R onde:
    │    R.deploy_time > D.deploy_time
    │    AND R.deploy_time < D.deploy_time + 24h
    │    AND R.service = D.service
    │    AND R.type = "rollback"
    │
    └── Marcar D como "failure" se qualquer condição = true

  CFR = count(D where failure=true) / count(all D) × 100%
```

### MTTR — Timeline de Incidente

```
FONTES:

  Incident Tracker (PagerDuty, OpsGenie, Grafana OnCall):
    ├── alert_fired_at: timestamp do alerta
    ├── acknowledged_at: engenheiro aceitou
    ├── resolved_at: incidente marcado resolvido
    └── sli_recovered_at: SLI voltou ao SLO (automático)

  Cálculo:
    MTTR = sli_recovered_at - alert_fired_at

  Breakdown:
    MTTD = alert_fired_at - actual_failure_start
           (difícil de medir — usar SLI threshold crossing)
    MTTE = acknowledged_at - alert_fired_at
    MTTF = resolved_at - acknowledged_at
    MTTV = sli_recovered_at - resolved_at
```

---

## SPACE Framework

```
SPACE = Satisfaction, Performance, Activity, Communication, Efficiency

  Origem:
  ├── Paper acadêmico de Nicole Forsgren, Margaret-Anne Storey, 
  │   Chandra Maddila, Thomas Zimmermann, Brian Houck, Jenna Butler
  ├── Publicado em ACM Queue (2021)
  └── Resposta a: "como medir produtividade de desenvolvedores?"

  PRINCÍPIO CENTRAL:
  ┌────────────────────────────────────────────────────────────────┐
  │  "Developer productivity cannot be reduced to a single         │
  │   dimension or metric. You MUST capture multiple dimensions   │
  │   to get a meaningful picture."                                │
  │                                                                │
  │  → NUNCA use uma única métrica para avaliar produtividade.     │
  │  → Selecione pelo menos 3 dimensões do SPACE.                 │
  │  → Capture em múltiplos níveis: individual, team, system.     │
  └────────────────────────────────────────────────────────────────┘

  POR QUE SPACE (e não apenas DORA)?
  ├── DORA foca em delivery pipeline (output)
  ├── SPACE inclui experiência humana (satisfaction, flow)
  ├── DORA é necessário mas insuficiente para "produtividade"
  └── SPACE complementa DORA com dimensões que DORA não cobre
```

---

## As 5 Dimensões do SPACE

### S — Satisfaction & Well-being

```
DEFINIÇÃO:
  Quão satisfeitos os desenvolvedores estão com seu trabalho,
  ferramentas, processos e cultura.

COMO MEDIR:
  ┌────────────────────────────────────────────────────────────────┐
  │ Nível       │ Métrica                    │ Instrumento         │
  ├─────────────┼────────────────────────────┼─────────────────────┤
  │ Individual  │ Job satisfaction (1-5)     │ Survey trimestral   │
  │ Individual  │ Tool satisfaction (1-5)    │ Survey trimestral   │
  │ Individual  │ Burnout risk (MBI scale)   │ Survey semestral    │
  │ Team        │ Team health (1-5)          │ Squad Health Check  │
  │ Team        │ eNPS (employee NPS)        │ Survey trimestral   │
  │ System      │ Developer NPS              │ Survey trimestral   │
  │ System      │ Retention rate             │ HR data             │
  │ System      │ Regrettable attrition      │ HR data             │
  └─────────────┴────────────────────────────┴─────────────────────┘

POR QUE IMPORTA:
  ├── Devs satisfeitos produzem melhor código e ficam mais tempo
  ├── Burnout é leading indicator de turnover
  ├── Satisfação com ferramentas correlaciona com DORA metrics
  └── Inversamente proporcional a tech debt percebido

COMO COLETAR:
  ├── Surveys anônimos (DX, Pulse, custom)
  ├── 1:1 regulares com perguntas estruturadas
  ├── Retrospectivas com health check
  └── Exit interviews para patterns

EXEMPLO DE SURVEY:
  "De 1 a 5, quão fácil é para você entregar código em produção?"
  "De 1 a 5, os testes me dão confiança para fazer mudanças?"
  "De 1 a 5, eu consigo encontrar a documentação que preciso?"
  "De 1 a 5, eu tenho tempo focado suficiente para deep work?"
```

### P — Performance

```
DEFINIÇÃO:
  Outcomes do trabalho — impacto real das entregas no negócio
  e nos usuários. NÃO é "quanto código produziu."

COMO MEDIR:
  ┌────────────────────────────────────────────────────────────────┐
  │ Nível       │ Métrica                    │ Instrumento         │
  ├─────────────┼────────────────────────────┼─────────────────────┤
  │ Individual  │ Peer review quality (1-5)  │ 360 review          │
  │ Individual  │ Code review effectiveness  │ PR data + survey    │
  │ Team        │ Feature adoption rate      │ Product analytics   │
  │ Team        │ Customer impact metrics    │ Business metrics    │
  │ Team        │ SLO compliance             │ Monitoring          │
  │ System      │ Reliability (SLO)          │ DORA 5th metric     │
  │ System      │ Revenue impact of features │ Business analytics  │
  │ System      │ Quality (defect rate)      │ Bug tracker         │
  └─────────────┴────────────────────────────┴─────────────────────┘

NOTA IMPORTANTE:
  ├── Performance é a dimensão MAIS difícil de medir
  ├── Não confundir com "performance de sistema" (latência, throughput)
  ├── Foco em OUTCOMES, não OUTPUTS
  └── "Shipping features that nobody uses" = low performance
```

### A — Activity

```
DEFINIÇÃO:
  Contagem de ações e outputs — design docs, commits, code reviews,
  deploys, PRs. É o mais fácil de medir, mas o mais perigoso
  se usado isoladamente.

COMO MEDIR:
  ┌────────────────────────────────────────────────────────────────┐
  │ Nível       │ Métrica                    │ Instrumento         │
  ├─────────────┼────────────────────────────┼─────────────────────┤
  │ Individual  │ PRs authored per week      │ GitHub API          │
  │ Individual  │ Code reviews completed     │ GitHub API          │
  │ Individual  │ Commits per week           │ Git data            │
  │ Team        │ Deploy frequency           │ CI/CD (DORA!)       │
  │ Team        │ PRs merged per sprint      │ GitHub API          │
  │ Team        │ Incidents responded        │ Incident tracker    │
  │ System      │ Total deployments          │ CI/CD               │
  │ System      │ Incident volume            │ Incident tracker    │
  └─────────────┴────────────────────────────┴─────────────────────┘

  ⚠️  CUIDADO:
  ┌────────────────────────────────────────────────────────────────┐
  │  Activity metrics são PROXIES, não medidas de produtividade.  │
  │  NUNCA use activity sozinha para avaliar performance.          │
  │  "Mais PRs" pode significar PRs menores e melhores            │
  │  (correlação com DF) OU gaming (commits sem valor).           │
  │  SEMPRE combine com Satisfaction + Performance.               │
  └────────────────────────────────────────────────────────────────┘
```

### C — Communication & Collaboration

```
DEFINIÇÃO:
  Como pessoas e equipes se comunicam, colaboram e integram
  seu trabalho.

COMO MEDIR:
  ┌────────────────────────────────────────────────────────────────┐
  │ Nível       │ Métrica                    │ Instrumento         │
  ├─────────────┼────────────────────────────┼─────────────────────┤
  │ Individual  │ Review turnaround time     │ GitHub API          │
  │ Individual  │ Knowledge sharing (1-5)    │ Survey + data       │
  │ Team        │ PR review depth            │ GitHub: comments/PR │
  │ Team        │ Cross-team PRs             │ GitHub API          │
  │ Team        │ Documentation contributions│ Wiki/docs metrics   │
  │ System      │ Knowledge silos index      │ Bus factor analysis │
  │ System      │ Onboarding time            │ HR/survey data      │
  │ System      │ Meeting hours per dev/week │ Calendar data       │
  └─────────────┴────────────────────────────┴─────────────────────┘

INDICADORES SAUDÁVEIS:
  ├── Review turnaround < 4h (business hours)
  ├── PRs com ≥ 2 reviewers (knowledge sharing)
  ├── Cross-team contributions existem (não silos)
  ├── Documentation é atualizada como parte do trabalho
  └── Meeting load < 30% do tempo (preserva deep work)
```

### E — Efficiency & Flow

```
DEFINIÇÃO:
  Capacidade de completar trabalho com esforço mínimo de 
  fricção. Inclui flow state, interrupções e delays.

COMO MEDIR:
  ┌────────────────────────────────────────────────────────────────┐
  │ Nível       │ Métrica                    │ Instrumento         │
  ├─────────────┼────────────────────────────┼─────────────────────┤
  │ Individual  │ Flow state frequency (1-5) │ Survey              │
  │ Individual  │ Context switches per day   │ Survey/tool data    │
  │ Individual  │ % focus time (uninterrupt) │ Calendar analysis   │
  │ Team        │ Lead Time for Changes      │ DORA (LT!)          │
  │ Team        │ Handoff count per feature  │ Process analysis    │
  │ Team        │ Wait time (in queue)       │ Issue tracker       │
  │ System      │ Build time (CI)            │ CI/CD data          │
  │ System      │ Test suite time            │ CI/CD data          │
  │ System      │ Dev environment setup time │ Survey/measurement  │
  │ System      │ PR review wait time        │ GitHub API          │
  └─────────────┴────────────────────────────┴─────────────────────┘

FRICTION POINTS COMUNS:
  ├── CI build leva > 15 min → devs param de confiar em CI
  ├── Code review espera > 1 dia → batches maiores, LT cresce
  ├── Dev environment leva > 30 min para setup → onboarding lento
  ├── Reuniões > 10h/semana → zero deep work
  └── Context switching > 5x/dia → produtividade cai 40%
```

---

## Combinando DORA + SPACE

```
MAPEAMENTO DORA → SPACE:

  ┌───────────────────────────────────────────────────────────────┐
  │ DORA Metric              │ SPACE Dimension                   │
  ├──────────────────────────┼───────────────────────────────────┤
  │ Deployment Frequency     │ Activity (A) + Efficiency (E)     │
  │ Lead Time for Changes    │ Efficiency (E)                    │
  │ Change Failure Rate      │ Performance (P)                   │
  │ Mean Time to Recovery    │ Efficiency (E) + Performance (P)  │
  │ Reliability              │ Performance (P)                   │
  └──────────────────────────┴───────────────────────────────────┘

  O QUE DORA NÃO COBRE (e SPACE sim):
  ├── Satisfaction: devs estão felizes? burnout?
  ├── Communication: colaboração funciona? silos?
  └── Activity: métricas de volume com contexto

  RECOMENDAÇÃO: Use DORA como A + E + P metrics quantitativas,
                SPACE como framework para incluir S + C qualitativas.

EXEMPLO DE COMPOSIÇÃO PARA UM TIME:

  ┌────────────────────────────────────────────────────────────────┐
  │ Dimensão      │ Métricas escolhidas                          │
  ├───────────────┼──────────────────────────────────────────────┤
  │ Satisfaction  │ Developer survey (quarterly), eNPS            │
  │ Performance   │ SLO compliance, CFR, feature adoption rate   │
  │ Activity      │ DF, PRs merged, code reviews completed       │
  │ Communication │ Review turnaround, cross-team PRs, meeting % │
  │ Efficiency    │ Lead Time, CI build time, focus time %        │
  └───────────────┴──────────────────────────────────────────────┘

  NÍVEIS:
  ├── Individual: Satisfaction + Efficiency (NÃO Activity)
  ├── Team: Todas as dimensões
  └── Organization: DORA + Satisfaction + Efficiency aggregated
```

---

## Dashboards e Visualização

```
DASHBOARD DORA — DESIGN:

  ┌──────────────────────────────────────────────────────────────┐
  │                  DORA METRICS DASHBOARD                       │
  ├──────────────────────────────────────────────────────────────┤
  │                                                               │
  │  ┌─────────────────┐  ┌─────────────────┐                    │
  │  │ Deploy Frequency │  │ Lead Time       │                    │
  │  │                  │  │                  │                    │
  │  │  ████████  3.2/d │  │  ████████ 4.5h  │                    │
  │  │  Target: daily+  │  │  Target: < 1d   │                    │
  │  │  Status: ✅ Elite │  │  Status: ✅ Elite│                    │
  │  └─────────────────┘  └─────────────────┘                    │
  │                                                               │
  │  ┌─────────────────┐  ┌─────────────────┐                    │
  │  │ Change Failure % │  │ MTTR            │                    │
  │  │                  │  │                  │                    │
  │  │  ████████  4.2%  │  │  ████████ 45min │                    │
  │  │  Target: < 5%    │  │  Target: < 1h   │                    │
  │  │  Status: ✅ Elite │  │  Status: ✅ Elite│                    │
  │  └─────────────────┘  └─────────────────┘                    │
  │                                                               │
  │  ┌──────────────────────────────────────────────┐             │
  │  │ Trend (últimos 90 dias)                      │             │
  │  │                                               │             │
  │  │  DF: ───────────────────── ↗ improving        │             │
  │  │  LT: ─────────╲──────── ↘ improving          │             │
  │  │  CFR: ────────────────── → stable              │             │
  │  │  MTTR:──────────╲──── ↘ improving             │             │
  │  └──────────────────────────────────────────────┘             │
  │                                                               │
  │  ┌──────────────────────────────────────────────┐             │
  │  │ Classification Matrix                         │             │
  │  │                                               │             │
  │  │  DF: Elite  │ LT: Elite  │ CFR: Elite │ MTTR: Elite       │
  │  │  Overall: ELITE PERFORMER ★                   │             │
  │  └──────────────────────────────────────────────┘             │
  │                                                               │
  │  ┌──────────────────────────────────────────────┐             │
  │  │ Lead Time Breakdown                           │             │
  │  │                                               │             │
  │  │  Coding:  ██████ 2.1h (47%)                   │             │
  │  │  Review:  ████ 1.5h (33%)  ← bottleneck       │             │
  │  │  Build:   █ 0.3h (7%)                          │             │
  │  │  Deploy:  █ 0.6h (13%)                         │             │
  │  └──────────────────────────────────────────────┘             │
  │                                                               │
  └──────────────────────────────────────────────────────────────┘

DASHBOARD SPACE — DESIGN:

  ┌──────────────────────────────────────────────────────────────┐
  │                 SPACE FRAMEWORK DASHBOARD                     │
  ├──────────────────────────────────────────────────────────────┤
  │                                                               │
  │  ┌──────────────────────────────────────────────┐             │
  │  │ SPACE Radar (equipe)                          │             │
  │  │                                               │             │
  │  │              Satisfaction                      │             │
  │  │                  ★ 4.2                          │             │
  │  │                 ╱    ╲                          │             │
  │  │   Efficiency ★ ╱      ╲ ★ Performance           │             │
  │  │   3.8       ╱          ╲     4.0                │             │
  │  │            ╱            ╲                       │             │
  │  │           ★───────────── ★                      │             │
  │  │   Communication    Activity                     │             │
  │  │   3.5              4.5                          │             │
  │  └──────────────────────────────────────────────┘             │
  │                                                               │
  │  ┌─────────────────┐  ┌─────────────────┐                    │
  │  │ Developer NPS   │  │ Focus Time %    │                    │
  │  │                  │  │                  │                    │
  │  │  eNPS: +32       │  │  65% focus time │                    │
  │  │  (prev: +28) ↗   │  │  Target: > 60%  │                    │
  │  └─────────────────┘  └─────────────────┘                    │
  │                                                               │
  │  ┌──────────────────────────────────────────────┐             │
  │  │ Team Health Trends (quarterly)                │             │
  │  │                                               │             │
  │  │  Satisfaction: 3.8 → 4.0 → 4.2 ↗             │             │
  │  │  Efficiency:   3.5 → 3.5 → 3.8 ↗             │             │
  │  │  Communication:3.2 → 3.4 → 3.5 ↗             │             │
  │  └──────────────────────────────────────────────┘             │
  │                                                               │
  └──────────────────────────────────────────────────────────────┘
```

---

## Anti-Patterns de Métricas

```
GOODHART'S LAW:
  "When a measure becomes a target, it ceases to be a good measure."

  ┌──────────────────────────────────────────────────────────────┐
  │ Anti-Pattern              │ O que acontece                   │
  ├───────────────────────────┼──────────────────────────────────┤
  │ "Aumentar DF a qualquer   │ Deploys de whitespace, config   │
  │ custo"                    │ trivial, features não testadas   │
  │                           │                                  │
  │ "Reduzir LT as any cost" │ Reviews superficiais, skip tests │
  │                           │ CFR sobe como consequência       │
  │                           │                                  │
  │ "CFR zero"                │ Freeze de deploys, medo de      │
  │                           │ mudar, velocity cai              │
  │                           │                                  │
  │ "MTTR como KPI de dev"    │ Devs escondem incidentes,       │
  │                           │ não reportam, não investigam     │
  │                           │                                  │
  │ "Activity as              │ Gaming: commits pequenos, PRs    │
  │ productivity"             │ triviais, reviews sem conteúdo   │
  │                           │                                  │
  │ "Individual DORA"         │ DORA é para EQUIPE, não          │
  │                           │ indivíduo. Usar para 1:1 é       │
  │                           │ desmotivador e impreciso         │
  │                           │                                  │
  │ "Comparar equipes"        │ Contextos diferentes (legacy vs  │
  │                           │ greenfield) tornam comparação    │
  │                           │ injusta e desmotivadora          │
  └───────────────────────────┴──────────────────────────────────┘

COMO EVITAR:
  ├── Use métricas para ENTENDER, não para COBRAR
  ├── Sempre apresente com contexto (narrativa + número)
  ├── Combine múltiplas dimensões (SPACE)
  ├── Foque em TENDÊNCIA, não em valor absoluto
  ├── Deixe as equipes escolherem como melhorar (autonomia)
  └── Revise métricas regularmente: elas ainda fazem sentido?
```

---

## DORA e Cultura de Engenharia

```
WESTRUM ORGANIZATIONAL CULTURE MODEL:

  Usado na pesquisa DORA como predictor de performance:

  ┌──────────────────────────────────────────────────────────────┐
  │ Pathological          │ Bureaucratic        │ Generative     │
  │ (Power-oriented)      │ (Rule-oriented)     │ (Performance)  │
  ├───────────────────────┼─────────────────────┼────────────────┤
  │ Info is hidden        │ Info may travel      │ Info is shared │
  │ Messengers shot       │ Messengers tolerated │ Messengers     │
  │                       │                      │ trained        │
  │ Failures punished     │ Failures lead to     │ Failures lead  │
  │                       │ justice              │ to inquiry     │
  │ New ideas crushed     │ New ideas create     │ New ideas      │
  │                       │ problems             │ welcomed       │
  │ No cooperation        │ Modest cooperation   │ Deep           │
  │                       │                      │ cooperation    │
  │ Blame                 │ Procedures           │ Learning       │
  └───────────────────────┴─────────────────────┴────────────────┘

  CORRELAÇÃO:
  ├── Cultura Generativa → melhor DORA metrics
  ├── Psychological safety → mais experimentação → mais learning
  ├── Blameless post-mortems → MTTR melhora
  └── Info sharing → menos silos → DF melhora

  COMO FOMENTAR CULTURA GENERATIVA:
  ├── Blameless post-mortems (sempre, sem exceção)
  ├── Failure Fridays / Game Days
  ├── Transparência em métricas (dashboard público)
  ├── Celebrar aprendizado, não só sucesso
  ├── Autonomia de equipe sobre processo
  └── Feedback loops curtos (CI/CD, observability)
```

---

## Instrumentação Prática — Automação

### GitHub Actions — Coletando DORA Events

```yaml
# .github/workflows/dora-events.yml
name: DORA Event Collector

on:
  workflow_run:
    workflows: ["Deploy to Production"]
    types: [completed]

  pull_request:
    types: [opened, closed]

  issues:
    types: [opened, closed, labeled]

jobs:
  collect-deploy-event:
    if: github.event_name == 'workflow_run' && github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    steps:
      - name: Record deployment event
        run: |
          curl -X POST "${{ secrets.DORA_COLLECTOR_URL }}/events/deployment" \
            -H "Content-Type: application/json" \
            -d '{
              "service": "${{ github.repository }}",
              "version": "${{ github.event.workflow_run.head_sha }}",
              "environment": "production",
              "deployed_at": "${{ github.event.workflow_run.updated_at }}",
              "status": "success",
              "triggered_by": "${{ github.event.workflow_run.event }}"
            }'

  collect-lead-time:
    if: github.event_name == 'pull_request' && github.event.action == 'closed' && github.event.pull_request.merged
    runs-on: ubuntu-latest
    steps:
      - name: Calculate and record lead time
        uses: actions/github-script@v7
        with:
          script: |
            const pr = context.payload.pull_request;
            
            // Get first commit in the PR
            const commits = await github.rest.pulls.listCommits({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: pr.number,
              per_page: 1
            });
            
            const firstCommitDate = new Date(commits.data[0].commit.author.date);
            const mergedAt = new Date(pr.merged_at);
            const leadTimeMs = mergedAt - firstCommitDate;
            const leadTimeHours = (leadTimeMs / (1000 * 60 * 60)).toFixed(2);
            
            console.log(`Lead Time: ${leadTimeHours} hours`);
            console.log(`First commit: ${firstCommitDate.toISOString()}`);
            console.log(`Merged at: ${mergedAt.toISOString()}`);
            
            // Post to collector
            await fetch(process.env.DORA_COLLECTOR_URL + '/events/lead-time', {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({
                service: context.repo.repo,
                pr_number: pr.number,
                first_commit_at: firstCommitDate.toISOString(),
                merged_at: mergedAt.toISOString(),
                lead_time_hours: parseFloat(leadTimeHours),
                coding_time_hours: 0, // enrich later
                review_time_hours: 0  // enrich later
              })
            });

  collect-incident:
    if: |
      github.event_name == 'issues' && 
      contains(github.event.issue.labels.*.name, 'incident')
    runs-on: ubuntu-latest
    steps:
      - name: Record incident event
        run: |
          ACTION="${{ github.event.action }}"
          if [ "$ACTION" = "labeled" ] || [ "$ACTION" = "opened" ]; then
            EVENT_TYPE="incident_opened"
          elif [ "$ACTION" = "closed" ]; then
            EVENT_TYPE="incident_resolved"
          fi
          
          curl -X POST "${{ secrets.DORA_COLLECTOR_URL }}/events/incident" \
            -H "Content-Type: application/json" \
            -d "{
              \"service\": \"${{ github.repository }}\",
              \"issue_number\": ${{ github.event.issue.number }},
              \"event_type\": \"${EVENT_TYPE}\",
              \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
              \"severity\": \"$(echo '${{ toJSON(github.event.issue.labels.*.name) }}' | jq -r '.[] | select(startswith(\"sev\"))')\"
            }"
```

### Prometheus Metrics — Exposição de DORA

```go
// Go — Exposing DORA as Prometheus metrics
package dora

import (
    "github.com/prometheus/client_golang/prometheus"
)

var (
    DeploymentsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "dora_deployments_total",
            Help: "Total deployments to production",
        },
        []string{"service", "status"}, // status: success, failure, rollback
    )

    LeadTimeSeconds = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "dora_lead_time_seconds",
            Help:    "Lead time from first commit to production deploy",
            Buckets: []float64{
                3600,       // 1h
                14400,      // 4h
                28800,      // 8h (1 business day)
                86400,      // 1 day
                259200,     // 3 days
                604800,     // 1 week
                2592000,    // 30 days
            },
        },
        []string{"service"},
    )

    ChangeFailureTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "dora_change_failures_total",
            Help: "Deployments that resulted in failure (incident, rollback)",
        },
        []string{"service", "failure_type"}, // rollback, incident, hotfix
    )

    IncidentRecoverySeconds = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "dora_incident_recovery_seconds",
            Help:    "Time from incident detection to service recovery",
            Buckets: []float64{
                300,        // 5 min
                900,        // 15 min
                1800,       // 30 min
                3600,       // 1h
                7200,       // 2h
                14400,      // 4h
                86400,      // 1 day
                604800,     // 1 week
            },
        },
        []string{"service", "severity"},
    )
)

func init() {
    prometheus.MustRegister(
        DeploymentsTotal,
        LeadTimeSeconds,
        ChangeFailureTotal,
        IncidentRecoverySeconds,
    )
}
```

```java
// Java — Spring Boot Micrometer DORA metrics
@Configuration
public class DoraMetricsConfig {

    @Bean
    public MeterBinder doraMetrics() {
        return registry -> {
            // Deployment Frequency counter
            Counter.builder("dora.deployments.total")
                .description("Total deployments to production")
                .tags("service", "order-service")
                .register(registry);

            // Lead Time histogram
            Timer.builder("dora.lead.time")
                .description("Lead time from first commit to production")
                .publishPercentileHistogram()
                .sla(
                    Duration.ofHours(1),
                    Duration.ofHours(4),
                    Duration.ofDays(1),
                    Duration.ofDays(7)
                )
                .register(registry);

            // Change Failure Rate counter
            Counter.builder("dora.change.failures.total")
                .description("Deployments resulting in failure")
                .tags("service", "order-service", "type", "rollback")
                .register(registry);

            // MTTR histogram
            Timer.builder("dora.incident.recovery.time")
                .description("Time from detection to recovery")
                .publishPercentileHistogram()
                .sla(
                    Duration.ofMinutes(15),
                    Duration.ofHours(1),
                    Duration.ofHours(4),
                    Duration.ofDays(1)
                )
                .register(registry);
        };
    }
}
```

### Grafana Recording Rules — DORA Calculations

```yaml
# prometheus/rules/dora-rules.yml
groups:
  - name: dora.metrics
    interval: 5m
    rules:
      # Deployment Frequency (deploys per day, rolling 7d)
      - record: dora:deployment_frequency:rate7d
        expr: |
          sum by (service) (
            increase(dora_deployments_total{status="success"}[7d])
          ) / 7

      # Lead Time p50 (rolling 30d)
      - record: dora:lead_time:p50:30d
        expr: |
          histogram_quantile(0.50,
            sum by (service, le) (
              rate(dora_lead_time_seconds_bucket[30d])
            )
          ) / 3600  # convert to hours

      # Lead Time p90
      - record: dora:lead_time:p90:30d
        expr: |
          histogram_quantile(0.90,
            sum by (service, le) (
              rate(dora_lead_time_seconds_bucket[30d])
            )
          ) / 3600

      # Change Failure Rate (rolling 30d)
      - record: dora:change_failure_rate:30d
        expr: |
          sum by (service) (increase(dora_change_failures_total[30d]))
          /
          sum by (service) (increase(dora_deployments_total{status="success"}[30d]))

      # MTTR p50 (rolling 30d)
      - record: dora:mttr:p50:30d
        expr: |
          histogram_quantile(0.50,
            sum by (service, le) (
              rate(dora_incident_recovery_seconds_bucket[30d])
            )
          ) / 60  # convert to minutes

      # DORA Classification (1=low, 2=medium, 3=high, 4=elite)
      - record: dora:classification:deployment_frequency
        expr: |
          clamp(
            (dora:deployment_frequency:rate7d >= 1) * 4    # daily+ = elite
            + (dora:deployment_frequency:rate7d >= 0.143 and dora:deployment_frequency:rate7d < 1) * 3  # weekly = high
            + (dora:deployment_frequency:rate7d >= 0.033 and dora:deployment_frequency:rate7d < 0.143) * 2  # monthly = medium
            + (dora:deployment_frequency:rate7d < 0.033) * 1,  # < monthly = low
            1, 4
          )
```

---

## Diretrizes para Code Review assistido por AI

```
QUANDO UM ASSISTENTE AI REVISAR CÓDIGO OU DISCUTIR DORA/SPACE:

  1. DORA É PARA EQUIPES, NÃO INDIVÍDUOS:
     ├── Nunca sugira DORA metrics para avaliar performance individual
     ├── Recomende análise no nível de equipe ou serviço
     └── Se pedirem individual metrics, redirecione para SPACE (Satisfaction)

  2. CONTEXTO SEMPRE:
     ├── Startup vs Enterprise têm benchmarks diferentes
     ├── Greenfield vs Legacy afeta drasticamente LT e DF
     ├── Team size afeta granularidade útil
     └── Não existe "one size fits all" benchmark

  3. ANTI-GAMING:
     ├── Se detectar padrão de gaming (commits vazios, PRs triviais),
     │   alerte que métricas estão sendo gamificadas
     ├── Sugira combinar Activity com Performance e Satisfaction
     └── Lembre: Goodhart's Law

  4. INSTRUMENTAÇÃO:
     ├── Sugira automação via CI/CD (webhook events)
     ├── Evite coleta manual (imprecisa e insustentável)
     ├── Recomende dashboards com tendências, não snapshots
     └── Sugira breakdown de Lead Time para encontrar bottlenecks

  5. CULTURA:
     ├── DORA sem cultura Generativa = números sem impacto
     ├── Blameless post-mortems são pré-requisito
     ├── Métricas para entender e melhorar, não para punir
     └── Celebrar melhoria gradual, não targets absolutos
```

---

## Referências

```
LEITURA OBRIGATÓRIA:
├── Accelerate — Nicole Forsgren, Jez Humble, Gene Kim (2018)
├── "The SPACE of Developer Productivity" — ACM Queue (2021)
│   https://queue.acm.org/detail.cfm?id=3454124
├── State of DevOps Report — dora.dev (anual)
│   https://dora.dev/research/
└── "Are you an Elite DevOps performer?" — dora.dev
    https://dora.dev/quickcheck/

LEITURA RECOMENDADA:
├── Team Topologies — Matthew Skelton, Manuel Pais
├── An Elegant Puzzle — Will Larson
├── The Phoenix Project — Gene Kim et al.
├── A Typology of Organisational Cultures — Ron Westrum (2004)
└── DevEx: What Actually Drives Productivity — ACM (2023)
    https://queue.acm.org/detail.cfm?id=3595878

FERRAMENTAS:
├── DORA Quick Check: https://dora.dev/quickcheck/
├── Four Keys (Google OSS): https://github.com/dora-team/fourkeys
├── Apache DevLake: https://devlake.apache.org/
├── Faros AI: https://www.faros.ai/
├── LinearB: https://linearb.io/
├── Sleuth: https://www.sleuth.io/
├── Jellyfish: https://jellyfish.co/
└── Swarmia: https://www.swarmia.com/
```
