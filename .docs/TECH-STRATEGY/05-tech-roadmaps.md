# Tech Roadmaps — Construção e Execução

> **Objetivo deste documento:** Servir como referência completa sobre **construção, comunicação e execução de tech roadmaps** — como planejar e comunicar a direção técnica de uma organização — otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: frameworks de roadmap (Now/Next/Later), alinhamento com product roadmap, OKRs técnicos, comunicação para stakeholders, tracking de execução, quarterly planning.

---

## Quick Reference — Cheat Sheet

| Conceito | Definição | Uso |
|----------|----------|-----|
| **Tech Roadmap** | Plano temporal de iniciativas técnicas | Comunicação e alinhamento |
| **Now/Next/Later** | Framework de horizonte temporal flexível | Roadmaps sem datas fixas |
| **Theme** | Agrupamento de iniciativas relacionadas | Organizar roadmap |
| **Tech OKR** | Objective + Key Results para tech | Medir progresso de roadmap |
| **Tech Investment** | % de capacity alocado para tech vs features | Budget de engineering |
| **Milestone** | Ponto verificável de progresso | Tracking e comunicação |

---

## Sumário

- [Tech Roadmaps — Construção e Execução](#tech-roadmaps--construção-e-execução)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que é um Tech Roadmap](#o-que-é-um-tech-roadmap)
  - [Framework Now / Next / Later](#framework-now--next--later)
  - [Construindo o Roadmap](#construindo-o-roadmap)
  - [Alinhamento com Product Roadmap](#alinhamento-com-product-roadmap)
  - [OKRs Técnicos](#okrs-técnicos)
  - [Comunicação do Roadmap](#comunicação-do-roadmap)
  - [Quarterly Planning](#quarterly-planning)
  - [Tracking e Execução](#tracking-e-execução)
  - [Balancing — Features vs Tech](#balancing--features-vs-tech)
  - [Anti-Patterns de Roadmap](#anti-patterns-de-roadmap)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que é um Tech Roadmap

```
TECH ROADMAP — O MAPA DA DIREÇÃO TÉCNICA:

  ┌─────────────────────────────────────────────────────────────┐
  │                                                              │
  │  TECH ROADMAP ≠ lista de tarefas técnicas                   │
  │  TECH ROADMAP ≠ backlog de tech debt                        │
  │  TECH ROADMAP ≠ lista de desejos de engenheiros             │
  │                                                              │
  │  TECH ROADMAP = plano comunicável que mostra:               │
  │  → ONDE estamos (estado atual)                              │
  │  → ONDE queremos chegar (visão técnica)                     │
  │  → COMO vamos chegar (sequência de iniciativas)             │
  │  → POR QUE essas iniciativas importam (business value)      │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘

  COMPONENTES DE UM BOM TECH ROADMAP:

  1. VISÃO TÉCNICA (Northstar)
     → "Em 18 meses, nossa plataforma será multi-region,
        cloud-native, com deploy em < 15 minutos para 
        qualquer serviço."
  
  2. THEMES (agrupamentos estratégicos)
     → Reliability: "de 99.9% para 99.99%"
     → Velocity: "deploy time de 2h para 15min"
     → Scale: "de 10K para 500K requests/sec"
     → Security: "zero trust architecture"
  
  3. INITIATIVES (projetos concretos dentro dos themes)
     → Theme Reliability: 
       - Multi-AZ deployment
       - Automated failover
       - Chaos engineering adoption
     → Theme Velocity:
       - CI/CD pipeline optimization
       - Internal developer platform
       - Feature flag infrastructure
  
  4. MILESTONES (pontos verificáveis)
     → "Multi-AZ deploy completo para tier-1 services"
     → "CI/CD pipeline < 15 min para 80% dos serviços"
  
  5. DEPENDENCIES e RISKS
     → "Platform team precisa terminar X antes de Y"
     → "Risco: hiring 2 SREs necessários para initiative Z"

  QUEM CRIA O TECH ROADMAP:
  
  │ Papel               │ Contribuição                        │
  │ CTO / VP Eng        │ Visão de alto nível, budget         │
  │ Principal/Staff Eng │ Traduzir visão em initiatives        │
  │ Engineering Managers│ Capacity planning, team allocation   │
  │ Tech Leads          │ Input técnico, estimativas, expertise│
  │ Product Management  │ Alignment com product roadmap        │
  │ SRE / Platform      │ Infra needs, reliability goals       │
```

---

## Framework Now / Next / Later

```
NOW / NEXT / LATER — ALTERNATIVA A ROADMAPS COM DATAS FIXAS:

  ┌─────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  NOW                    NEXT                   LATER             │
  │  (este quarter)         (próximo quarter)      (futuro)          │
  │  Alta confiança         Média confiança        Baixa confiança  │
  │  Comprometido           Planejado              Aspiracional     │
  │                                                                  │
  │  ┌─────────────────┐   ┌─────────────────┐   ┌────────────────┐│
  │  │ CI/CD Pipeline  │   │ Internal Dev    │   │ Multi-region   ││
  │  │ Optimization    │   │ Platform (IDP)  │   │ Architecture   ││
  │  │ [████████░░] 80%│   │ [██░░░░░░░░] 20%│   │ [░░░░░░░░░░] 0%││
  │  └─────────────────┘   └─────────────────┘   └────────────────┘│
  │  ┌─────────────────┐   ┌─────────────────┐   ┌────────────────┐│
  │  │ Java 11 → 21    │   │ Event-Driven    │   │ Edge Computing ││
  │  │ Migration        │   │ Architecture    │   │ for LATAM      ││
  │  │ [██████░░░░] 60%│   │ [░░░░░░░░░░] 0% │   │ [░░░░░░░░░░] 0%││
  │  └─────────────────┘   └─────────────────┘   └────────────────┘│
  │  ┌─────────────────┐   ┌─────────────────┐                     │
  │  │ Observability   │   │ Zero Trust      │                     │
  │  │ (OpenTelemetry) │   │ Network         │                     │
  │  │ [████████████]ok│   │ [░░░░░░░░░░] 0% │                     │
  │  └─────────────────┘   └─────────────────┘                     │
  │                                                                  │
  └─────────────────────────────────────────────────────────────────┘

  POR QUE NOW/NEXT/LATER vs GANTT CHART:

  │ Aspecto           │ Gantt / Timeline       │ Now/Next/Later       │
  │ Precisão de datas │ Falsa precisão         │ Honesto sobre incerteza │
  │ Flexibilidade     │ Rígido, cascading delay│ Fácil de reprioritizar │
  │ Comunicação       │ "Por que estamos atrasados?"│ "O que mudou de prioridade?" │
  │ Commitment level  │ Tudo parece "prometido"│ Claro: NOW=committed  │
  │ Planning horizon  │ Detalhado demais p/ futuro│ Detalha o próximo   │

  REGRAS:
  → NOW: máximo 3-5 initiatives (focus é poder)
  → NEXT: 3-5 initiatives (em refinamento)
  → LATER: qualquer quantidade (backlog priorizado)
  → Items movem: LATER → NEXT → NOW 
  → Items também podem sair: "não vamos mais fazer"
  → Review: a cada quarter, rever todas as colunas
```

---

## Construindo o Roadmap

```
PROCESSO PARA CONSTRUIR TECH ROADMAP:

  ═══════════════════════════════════════════════════════════
  STEP 1: DESCOBERTA (1-2 semanas)
  ═══════════════════════════════════════════════════════════

  INPUT GATHERING:
  
  ┌──────────────────────────────────────────────────────────┐
  │ FONTE                  │ O QUE EXTRAIR                   │
  ├────────────────────────┼─────────────────────────────────┤
  │ Business Strategy      │ Objetivos 1-3 anos              │
  │ Product Roadmap        │ Features que exigem tech invest │
  │ Tech Debt Register     │ Debt de alto impacto            │
  │ Incident retrospectives│ Padrões de falha recorrentes    │
  │ DORA Metrics           │ Onde estamos weak               │
  │ Team surveys           │ Developer pain points           │
  │ Security audits        │ Compliance gaps                 │
  │ Capacity planning      │ Scale needs (growth projections)│
  │ Competitive analysis   │ Tech capabilities dos competitors│
  │ Industry trends        │ Emerging tech relevante          │
  └────────────────────────┴─────────────────────────────────┘

  ═══════════════════════════════════════════════════════════
  STEP 2: THEMED GROUPING (1 semana)
  ═══════════════════════════════════════════════════════════

  Agrupar todas as necessidades em THEMES:

  EXEMPLO DE THEMES:

  Theme 1: RELIABILITY & RESILIENCE
  → Multi-AZ deployment
  → Automated failover
  → Chaos engineering
  → Disaster recovery testing
  
  Theme 2: DEVELOPER VELOCITY
  → CI/CD optimization (build time < 15 min)
  → Internal Developer Platform
  → Feature flags infrastructure
  → Dev environment parity (staging = prod)
  
  Theme 3: SCALE
  → Database sharding
  → Auto-scaling for all tier-1 services
  → CDN + edge caching
  → Async processing for heavy workloads
  
  Theme 4: MODERNIZATION
  → Java 11 → 21 migration
  → Strangler Fig: monolith decomposition
  → Event-driven architecture adoption
  → API versioning strategy
  
  Theme 5: SECURITY
  → Zero trust networking
  → Secret management (Vault)
  → Dependency vulnerability scanning
  → SOC2 Type II preparation

  ═══════════════════════════════════════════════════════════
  STEP 3: PRIORIZAÇÃO (1 semana)
  ═══════════════════════════════════════════════════════════

  Para cada initiative, avaliar:

  │ Critério              │ Weight │ Scale │ Descrição                 │
  │ Business Impact       │  30%   │ 1-5   │ Quanto habilita negócio? │
  │ Risk Reduction        │  25%   │ 1-5   │ Quanto reduz risco?       │
  │ Effort (inverse)      │  20%   │ 1-5   │ 5=fácil, 1=muito esforço │
  │ Dependencies          │  15%   │ 1-5   │ 5=independente, 1=blocked│
  │ Team Readiness        │  10%   │ 1-5   │ 5=skill exists, 1=needs hire│

  Resultado: ranked list de initiatives

  ═══════════════════════════════════════════════════════════
  STEP 4: SEQUENCIAMENTO (1 semana)
  ═══════════════════════════════════════════════════════════

  Considerar:
  → Dependencies: X precisa ser feito antes de Y
  → Capacity: quantas initiatives paralelas é viável?
  → Quick wins: 1-2 initiatives de alto impacto e baixo esforço
  → Foundation-first: infra básica antes de features avançadas

  Dependency Map:

  [CI/CD Optimization] ──→ [Feature Flags] ──→ [Canary Deploys]
         │
         └──→ [IDP Phase 1] ──→ [IDP Phase 2]
  
  [Java 21 Migration] ──→ [Spring Boot 3 Upgrade]
                               │
  [DB Sharding] ──→ [Auto-scaling] ──→ [Multi-region]

  ═══════════════════════════════════════════════════════════
  STEP 5: DOCUMENTAÇÃO (2-3 dias)
  ═══════════════════════════════════════════════════════════

  Produzir:
  → 1-pager executivo (para C-level)
  → Roadmap visual (Now/Next/Later format)
  → Detailed initiatives doc (para eng leadership)
  → OKRs derivados (para tracking)
  → Communication deck (para all-hands)
```

---

## Alinhamento com Product Roadmap

```
PRODUCT ROADMAP ↔ TECH ROADMAP — ALINHAMENTO:

  ┌─────────────────────────────────────────────────────────────┐
  │                                                              │
  │  PRODUCT ROADMAP          TECH ROADMAP                       │
  │  (o que entregar)         (como habilitar)                   │
  │                                                              │
  │  "LATAM expansion" ─────→ "Multi-region architecture"       │
  │  "Real-time analytics" ──→ "Event streaming (Kafka)"        │
  │  "Mobile app launch" ────→ "API optimization + CDN"         │
  │  "Enterprise tier" ──────→ "SOC2 + SSO + audit logging"    │
  │  "10x user growth" ─────→ "Auto-scaling + DB sharding"     │
  │                                                              │
  │  + TECH-ONLY initiatives:                                    │
  │  "--" ──────────────────→ "CI/CD optimization"              │
  │  "--" ──────────────────→ "Tech debt reduction"             │
  │  "--" ──────────────────→ "Java 21 migration"               │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘

  MODELO DE INTERAÇÃO COM PRODUCT:

  ┌────────────────────────────────────────────────────────────┐
  │                                                             │
  │ QUARTERLY PLANNING CYCLE:                                   │
  │                                                             │
  │ Week 1: Product shares upcoming priorities                  │
  │    → "Estes são os themes de produto para Q3"              │
  │                                                             │
  │ Week 2: Tech assesses implications                          │
  │    → "Para LATAM expansion, precisamos de multi-region     │
  │       infra. Isso leva 8 weeks."                           │
  │    → "Real-time analytics precisa de Kafka setup. 4 weeks." │
  │                                                             │
  │ Week 3: Joint prioritization                                │
  │    → Trade-offs explícitos:                                 │
  │      "Se priorizarmos LATAM, adiamos mobile API optimization"│
  │    → Acordo: 80% capacity para product-enabling tech,       │
  │              20% para tech-only (debt, modernization)        │
  │                                                             │
  │ Week 4: Roadmap finalizado e comunicado                     │
  │    → Publicar: product roadmap + tech roadmap side-by-side │
  │    → OKRs definidos e aceitos                               │
  │    → Dependencies mapeadas e acknowledged                   │
  │                                                             │
  └────────────────────────────────────────────────────────────┘

  REGRA DE OURO:
  → Tech roadmap deve ser ~60-70% PRODUCT-ENABLING
    (infra para product features)
  → ~20-30% TECH EXCELLENCE
    (debt, modernization, velocity improvements)
  → ~5-10% EXPLORATION/INNOVATION
    (tech spikes, prototypes, learning)
```

---

## OKRs Técnicos

```
OKRs TÉCNICOS — MEDIR O PROGRESSO:

  ESTRUTURA:
  
  Objective: QUALITATIVO, inspiracional, direção
  Key Results: QUANTITATIVOS, mensuráveis, time-bound

  ═══════════════════════════════════════════════════════════
  EXEMPLOS DE OKRs POR THEME:
  ═══════════════════════════════════════════════════════════

  THEME: RELIABILITY
  ─────────────────
  Objective: "Nossa plataforma é confiável e resiliente 
             para suportar crescimento de 10x"
  
  KR1: SLA de 99.95% → 99.99% para tier-1 services
  KR2: MTTR reduzido de 45 min para < 15 min (p50)
  KR3: Zero P1 incidents causados por falta de failover
  KR4: DR testado mensalmente com RTO < 30 min confirmado

  THEME: DEVELOPER VELOCITY
  ──────────────────────────
  Objective: "Engenheiros entregam features com velocidade 
             e confiança"
  
  KR1: CI/CD pipeline duration P95 < 15 min (atual: 45 min)
  KR2: Deployment frequency ≥ 2x/dia para todos os serviços
  KR3: Lead time for changes < 1 dia (atual: 5 dias)
  KR4: Developer satisfaction survey ≥ 8/10 (atual: 6/10)

  THEME: SCALE
  ────────────
  Objective: "Plataforma escala automaticamente para 
             suportar LATAM expansion"
  
  KR1: 100% dos tier-1 services com auto-scaling configurado
  KR2: Latency P99 < 100ms em região São Paulo
  KR3: Capacity para 500K req/s (atual: 50K)
  KR4: Custo de infra / request reduzido em 30%

  THEME: MODERNIZATION
  ────────────────────
  Objective: "Stack modernizada, sem legacy bloqueante"
  
  KR1: 100% dos serviços em Java 21 (atual: 40%)
  KR2: 3 serviços extraídos do monolito (Strangler Fig)
  KR3: Zero CVEs críticas em dependencies (atual: 12)
  KR4: Spring Boot 3.x em 100% dos serviços Java

  THEME: SECURITY
  ───────────────
  Objective: "Security é built-in, não bolted-on"
  
  KR1: SOC2 Type II audit passed
  KR2: 100% secrets em Vault (zero hardcoded)
  KR3: Dependency scanning em 100% dos repos CI
  KR4: Mean time to patch critical CVE < 48h

  ═══════════════════════════════════════════════════════════
  BOAS PRÁTICAS PARA OKRs TÉCNICOS:
  ═══════════════════════════════════════════════════════════

  ✅ Máximo 3-5 Objectives por quarter
  ✅ 2-4 KRs por Objective
  ✅ KRs devem ser MENSURÁVEIS (número, não "melhorar")
  ✅ Ambicioso: 70% achievement = good
  ✅ Owner claro para cada KR
  ✅ Check-in semanal ou bi-semanal no KR progress
  
  ❌ "Melhorar a qualidade do código" (vago)
  ❌ "Implementar microsserviços" (output, não outcome)
  ❌ 10 Objectives (impossível focar)
  ❌ KRs sem baseline ("reduzir latency" — de quanto?)
```

---

## Comunicação do Roadmap

```
COMUNICAÇÃO — FORMATOS POR AUDIÊNCIA:

  ═══════════════════════════════════════════════════════════
  FORMATO 1: EXECUTIVE SUMMARY (1-pager)
  Audiência: C-level, Board
  ═══════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────┐
  │ TECH ROADMAP 2026 — EXECUTIVE SUMMARY                    │
  │                                                          │
  │ VISÃO: "Plataforma cloud-native, multi-region, com       │
  │ deploy automatizado, suportando 10x do volume atual"     │
  │                                                          │
  │ INVESTIMENTO: 25% de capacity de engineering              │
  │ (equivalente a $X em 12 meses)                           │
  │                                                          │
  │ TOP 3 PRIORIDADES:                                       │
  │ 1. Multi-region (habilita LATAM expansion) — Q1-Q2       │
  │ 2. Auto-scaling (suporta 10x growth) — Q2-Q3            │
  │ 3. Security hardening (SOC2 compliance) — Q3-Q4          │
  │                                                          │
  │ RISCOS: hiring 3 SREs, Kafka expertise gap               │
  │ EXPECTED ROI: 40% faster feature delivery, 80%           │
  │ reduction in incidents                                    │
  └──────────────────────────────────────────────────────────┘

  ═══════════════════════════════════════════════════════════
  FORMATO 2: ROADMAP VISUAL
  Audiência: Engineering leadership, all-hands
  ═══════════════════════════════════════════════════════════

  2026         Q1              Q2              Q3              Q4
  ─────────────┼───────────────┼───────────────┼───────────────┼──
  RELIABILITY  │               │               │               │
               │████████████████               │               │
               │ Multi-AZ      │               │               │
               │               │███████████████████████████████│
               │               │ Chaos Engineering + DR Testing│
  ─────────────┼───────────────┼───────────────┼───────────────┼──
  VELOCITY     │               │               │               │
               │██████████████████████████████│               │
               │ CI/CD Pipeline Optimization   │               │
               │               │████████████████████████████████
               │               │ Internal Developer Platform   │
  ─────────────┼───────────────┼───────────────┼───────────────┼──
  SCALE        │               │               │               │
               │               │██████████████████████████████│
               │               │ Auto-scaling + DB Sharding    │
               │               │               │████████████████
               │               │               │ Multi-Region  │
  ─────────────┼───────────────┼───────────────┼───────────────┼──
  MODERNIZE    │               │               │               │
               │████████████████████████████████               │
               │ Java 21 + Spring Boot 3 Migration             │
               │██████████████████████████████████████████████│
               │ Monolith Decomposition (Strangler Fig)        │
  ─────────────┼───────────────┼───────────────┼───────────────┼──
  SECURITY     │               │               │               │
               │████████████████               │               │
               │ Secret Mgmt   │               │               │
               │               │               │████████████████
               │               │               │ SOC2 Type II  │
  ─────────────┼───────────────┼───────────────┼───────────────┼──

  ═══════════════════════════════════════════════════════════
  FORMATO 3: DETAILED INITIATIVE CARDS
  Audiência: Engineering teams
  ═══════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────┐
  │ INITIATIVE: CI/CD Pipeline Optimization                    │
  │ Theme: Developer Velocity                                 │
  │ Priority: NOW (Q1-Q2 2026)                                │
  │ Owner: Maria (Staff Engineer, Platform)                   │
  │                                                          │
  │ CONTEXT:                                                  │
  │ Pipeline atual leva 45 min (P95). Target: < 15 min.       │
  │ Bloqueia deploy frequency e developer experience.         │
  │                                                          │
  │ SCOPE:                                                    │
  │ 1. Build caching (Docker layer, Maven/Gradle cache)       │
  │ 2. Parallel test execution                                │
  │ 3. Incremental builds                                     │
  │ 4. Flaky test quarantine                                  │
  │ 5. Self-hosted runners (performance)                      │
  │                                                          │
  │ MILESTONES:                                               │
  │ □ Week 2: Build caching implemented → 30 min target       │
  │ □ Week 4: Parallel tests → 20 min target                  │
  │ □ Week 8: All optimizations → 15 min target               │
  │                                                          │
  │ RESOURCES: 2 engineers × 8 weeks                          │
  │ DEPENDENCIES: GitHub Actions upgrade, self-hosted runners │
  │ KEY RESULTS: Pipeline P95 < 15 min, Deploy freq ≥ 2x/dia│
  │ RISKS: Flaky test backlog larger than estimated           │
  └──────────────────────────────────────────────────────────┘
```

---

## Quarterly Planning

```
QUARTERLY PLANNING CYCLE:

  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  WEEK -4 (1 mês antes do quarter)                        │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ RETROSPECTIVA DO QUARTER ATUAL                 │      │
  │  │ → O que entregamos? (vs planejado)             │      │
  │  │ → OKR achievement (green/yellow/red)           │      │
  │  │ → O que NÃO entregamos e por quê?             │      │
  │  │ → Surpresas: o que não previmos?               │      │
  │  │ → Learnings para próximo quarter              │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                          │
  │  WEEK -3                                                  │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ INPUT GATHERING                                │      │
  │  │ → Product priorities para próximo quarter      │      │
  │  │ → Tech debt status (o que piorou/melhorou)    │      │
  │  │ → Incident trends (novos patterns)            │      │
  │  │ → Team feedback (surveys, 1:1s)               │      │
  │  │ → Capacity: quem está disponível?             │      │
  │  │ → External: vendor changes, compliance needs   │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                          │
  │  WEEK -2                                                  │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ DRAFT ROADMAP                                  │      │
  │  │ → Staff/Principal Engineers criam proposta     │      │
  │  │ → Priorizar initiatives (WSJF/RICE scoring)   │      │
  │  │ → Map dependencies                             │      │
  │  │ → Define OKRs                                  │      │
  │  │ → Capacity allocation (features vs tech)       │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                          │
  │  WEEK -1                                                  │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ REVIEW & ALIGNMENT                             │      │
  │  │ → Review with Eng leadership                   │      │
  │  │ → Alignment com Product leadership              │      │
  │  │ → Ajustes de prioridade baseados em feedback   │      │
  │  │ → Final OKRs committed                         │      │
  │  │ → Communication plan                           │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                          │
  │  WEEK 1 (início do quarter)                              │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ KICKOFF                                        │      │
  │  │ → Comunicar roadmap em all-hands                │      │
  │  │ → Publicar roadmap + OKRs                       │      │
  │  │ → Teams iniciam execution                       │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                          │
  │  WEEK 6 (mid-quarter checkpoint)                         │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ MID-QUARTER REVIEW                             │      │
  │  │ → OKR progress check (on-track / at-risk?)     │      │
  │  │ → Adjust scope se necessário                    │      │
  │  │ → Identify blockers + escalate                  │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

---

## Tracking e Execução

```
TRACKING DE ROADMAP — COMO SABER SE ESTAMOS NO CAMINHO:

  ═══════════════════════════════════════════════════════════
  DASHBOARD DE ROADMAP (semanal)
  ═══════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────┐
  │ TECH ROADMAP Q1 2026 — STATUS                            │
  │ Updated: 2026-02-15                                      │
  │                                                          │
  │ OVERALL: 🟢 ON TRACK (65% complete, target 58%)          │
  │                                                          │
  │ RELIABILITY:                                              │
  │   Multi-AZ Deploy       [████████████████░░░░] 80% 🟢    │
  │   → Milestone: tier-1 services done ✅                    │
  │   → Remaining: tier-2 services (3 of 8)                  │
  │                                                          │
  │ VELOCITY:                                                 │
  │   CI/CD Optimization    [████████████░░░░░░░░] 60% 🟡    │
  │   → Build cache done ✅, parallel tests in progress       │
  │   → At risk: flaky test backlog larger than estimated    │
  │   → Action: adding 1 engineer for 2 weeks                │
  │                                                          │
  │ MODERNIZATION:                                            │
  │   Java 21 Migration     [██████░░░░░░░░░░░░░░] 30% 🟡    │
  │   → 4/12 services migrated                               │
  │   → Blocker: spring-cloud-sleuth incompatible            │
  │   → Action: evaluate micrometer-tracing alternative      │
  │                                                          │
  │ SCALE:                                                    │
  │   Auto-scaling          [░░░░░░░░░░░░░░░░░░░░]  0% ⚪    │
  │   → Scheduled for Q2 (on plan)                           │
  │                                                          │
  │ OKR PROGRESS:                                             │
  │   KR1: Pipeline P95 < 15 min  Actual: 28 min  🟡 55%    │
  │   KR2: Deploy freq ≥ 2x/day   Actual: 1.2/day 🟡 60%    │
  │   KR3: Java 21 100%           Actual: 33%     🟡 33%    │
  │   KR4: CVE críticas = 0       Actual: 4       🟡 67%    │
  └──────────────────────────────────────────────────────────┘

  STATUS DEFINITIONS:
  🟢 ON TRACK: ≥ expected progress, no blockers
  🟡 AT RISK:  behind expected, actions identified
  🔴 BLOCKED:  significant blocker, escalation needed
  ⚪ NOT STARTED: planned for later (on schedule)

  ═══════════════════════════════════════════════════════════
  RITMO DE CHECK-INS
  ═══════════════════════════════════════════════════════════

  │ Ritual            │ Frequência │ Audiência          │ Duração │
  │ OKR standup       │ Semanal    │ Initiative owners   │ 15 min  │
  │ Roadmap review    │ Bi-semanal │ Eng leadership      │ 30 min  │
  │ Stakeholder update│ Mensal     │ Product + Eng mgmt │ 30 min  │
  │ Quarter review    │ Trimestral │ All eng + product   │ 60 min  │

  QUANDO RE-PRIORIZAR:

  → Incident pattern emerge que não estava no roadmap
  → Business priority changes (M&A, new competitor, regulation)
  → Initiative mais complexa que estimado (scope cut needed)
  → Opportunity: quick win discovered
  → Dependency blocked (team/vendor/external)
  
  REGRA: re-priorizar é OK e esperado. 
  O problema é re-priorizar SILENCIOSAMENTE.
  → Sempre comunicar o que mudou e por quê.
```

---

## Balancing — Features vs Tech

```
ALOCAÇÃO DE CAPACITY — A ETERNA TENSÃO:

  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  "Se só entregamos features, plataforma degrada"        │
  │  "Se só investimos em tech, negócio não cresce"          │
  │                                                          │
  │  SOLUÇÃO: agreement explícito de alocação               │
  │                                                          │
  │  ┌────────────────────────────────────────────────────┐ │
  │  │ HEALTHY ALLOCATION:                                │ │
  │  │                                                    │ │
  │  │ ████████████████████████████████████████████░░░░░░ │ │
  │  │ │←──── 70% Product Features ────→│ 20% │ 10% │    │ │
  │  │                                    Tech   Innov    │ │
  │  └────────────────────────────────────────────────────┘ │
  │                                                          │
  │  AJUSTES POR CONTEXTO:                                   │
  │                                                          │
  │  Startup (pre-PMF):                                      │
  │  ████████████████████████████████████████████████░░░░   │
  │  90% features / 8% tech / 2% innovation                  │
  │  → Foco em product-market fit                            │
  │                                                          │
  │  Growth (post-PMF, scaling):                              │
  │  ████████████████████████████░░░░░░░░░░░░░░░░░░░░░░   │
  │  60% features / 30% tech / 10% innovation                │
  │  → "Pay the tech debt from growth phase"                 │
  │                                                          │
  │  Mature (established product):                            │
  │  ████████████████████████████████████████░░░░░░░░░░░   │
  │  70% features / 20% tech / 10% innovation                │
  │  → Sustainable balance                                    │
  │                                                          │
  │  Crisis (major incidents, attrition):                     │
  │  ██████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   │
  │  30% features / 60% tech / 10% innovation                │
  │  → "Feature freeze" para estabilizar                     │
  │                                                          │
  └──────────────────────────────────────────────────────────┘

  COMO CONSEGUIR BUY-IN PARA TECH INVESTMENT:

  1. QUANTIFIQUE O CUSTO DE NÃO INVESTIR
     → "Sem auto-scaling, incident P1 a cada growth spike"
     → "Sem CI/CD fix, engineers gastam 2h/dia esperando build"

  2. CONECTE A BUSINESS OUTCOMES
     → "CI/CD fast → deploy 4x/dia → feature reach market faster"
     → "Auto-scaling → zero downtime → customer trust"

  3. PROPONHA COMO INVESTMENT, NÃO CUSTO
     → ❌ "Precisamos gastar 3 meses em tech debt"
     → ✅ "Investimento de 8 weeks que retorna 40% mais 
          velocity para os próximos 2 anos"

  4. SHOW, DON'T TELL
     → Mostre DORA metrics trends
     → Mostre incident frequency trends
     → Mostre custo de incidents ($)
     → Mostre developer satisfaction trends
```

---

## Anti-Patterns de Roadmap

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **Roadmap = wishlist** | Lista sem priorização → execução nenhuma | Max 3-5 NOW items, rigorously prioritized |
| **No business connection** | "Vamos adotar K8s" sem "para que" | Cada initiative ligada a business outcome |
| **Promise-driven** | Datas exatas para tudo → frustração | Now/Next/Later com commitment levels claros |
| **Unchangeable roadmap** | "Foi planejado, não pode mudar" | Roadmap é vivo — review a cada quarter minimum |
| **No tracking** | Roadmap criado, ninguém acompanha | Weekly OKR standup + bi-weekly roadmap review |
| **Secret roadmap** | Só leadership sabe | Publicar para todos os engineers |
| **All tech, no product** | Engenheiros planejam sem PM input | Joint planning: tech + product always aligned |
| **One-person roadmap** | Architect cria sozinho | Input de Tech Leads, PMs, SREs, todos os ICs |
| **No quick wins** | Só projetos de 6+ meses | Include 2-3 quick wins (< 4 weeks) per quarter |
| **No celebration** | Entregar milestone sem reconhecimento | Demo milestones, celebrate delivery, share impact |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e propostas de arquitetura, considere o roadmap:

1. **Alinhamento com roadmap** — Se mudança arquitetural grande, verificar se está alinhada com tech roadmap ou se é desvio não-planejado
2. **ADR para decisão significativa** — Mudança que afeta roadmap (nova tecnologia, mudança de direção) precisa de ADR
3. **Quick win opportunity** — Se PR revela melhoria de alto impacto e baixo esforço, sugerir adição ao roadmap como quick win
4. **Dependency awareness** — Se PR cria dependency cross-team, verificar se está mapeada no roadmap
5. **Tech debt criado** — Se PR cria shortcut consciente, exigir documentação e adição ao Debt Register
6. **OKR-connected** — Para initiatives do roadmap, verificar se PR menciona/linka o OKR/initiative relevante
7. **Migration consistency** — Se parte de migração (Java 21, Spring Boot 3, etc.), verificar que segue o playbook documentado
8. **Scope creep na migration** — PR de migração com features novas misturadas → separar: migration-only + feature
9. **Observability for new initiatives** — Code de initiative nova sem metrics/logging → como saberemos se está funcionando?
10. **Roadmap drift detection** — Se padrão de PRs sugere trabalho fora do roadmap, sinalizar para revisão de prioridades

---

## Referências

- **An Elegant Puzzle** — Will Larson (Stripe Press, 2019)
- **Staff Engineer** — Will Larson (2021) — https://staffeng.com/
- **The Staff Engineer's Path** — Tanya Reilly (O'Reilly, 2022)
- **Technology Strategy Patterns** — Eben Hewitt (O'Reilly, 2018)
- **Team Topologies** — Matthew Skelton & Manuel Pais (IT Revolution, 2019)
- **Measure What Matters** — John Doerr (Portfolio/Penguin, 2018)
- **Radical Focus** — Christina Wodtke (Cucina Media, 2016)
- **Product Roadmaps Relaunched** — C. Todd Lombardo et al. (O'Reilly, 2017)
- **Now Next Later** — Janna Bastow — https://www.prodpad.com/blog/invented-now-next-later-roadmap/
- **Accelerate** — Nicole Forsgren et al. (IT Revolution, 2018)
- **The Phoenix Project** — Gene Kim et al. (IT Revolution, 2013)
- **DORA Metrics** — https://dora.dev/
- **SAFe Framework — PI Planning** — https://scaledagileframework.com/pi-planning/
- **Thoughtworks Technology Radar** — https://www.thoughtworks.com/radar
