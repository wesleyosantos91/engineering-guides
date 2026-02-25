# Technical Vision — Definir a Direção Técnica

> **Objetivo deste documento:** Servir como referência completa sobre **Technical Vision** — como definir, documentar e comunicar a direção técnica de 2-3 anos para uma organização — otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: Tech Radar, RFCs, documentos de visão, como criar e manter direção técnica, comunicação da visão para diferentes audiências.

---

## Quick Reference — Cheat Sheet

| Conceito | Definição | Quando usar |
|----------|----------|-------------|
| **Technical Vision** | Northstar de 2-3 anos para a plataforma | Alinhar decisões diárias |
| **Tech Radar** | Mapa do portfólio tecnológico (Adopt/Trial/Assess/Hold) | Governança de tecnologias |
| **RFC** | Request for Comments — proposta formal de mudança | Decisão técnica significativa |
| **ADR** | Architecture Decision Record — registro de decisão | Documentar o "porquê" |
| **Vision Doc** | Documento narrativo da direção técnica futura | Comunicar e alinhar stakeholders |
| **Principles** | Guideline de alto nível para decisões técnicas | Empoderar decisões descentralizadas |
| **Wardley Map** | Mapa de evolução de componentes no eixo valor × maturidade | Estratégia de portfolio |

---

## Sumário

- [Technical Vision — Definir a Direção Técnica](#technical-vision--definir-a-direção-técnica)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que é Technical Vision](#o-que-é-technical-vision)
  - [Papel de Cada Nível na Visão Técnica](#papel-de-cada-nível-na-visão-técnica)
  - [Criando o Documento de Visão Técnica](#criando-o-documento-de-visão-técnica)
  - [Tech Radar — Governança do Portfolio](#tech-radar--governança-do-portfolio)
  - [RFCs — Propondo Mudanças Técnicas](#rfcs--propondo-mudanças-técnicas)
  - [Engineering Principles](#engineering-principles)
  - [De Visão a Execução](#de-visão-a-execução)
  - [Comunicação da Visão](#comunicação-da-visão)
  - [Mantendo a Visão Viva](#mantendo-a-visão-viva)
  - [Anti-Patterns](#anti-patterns)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que é Technical Vision

```
TECHNICAL VISION — O NORTHSTAR DA ENGENHARIA:

  ┌─────────────────────────────────────────────────────────────┐
  │                                                              │
  │  TECHNICAL VISION responde:                                  │
  │                                                              │
  │  1. ONDE ESTAMOS?                                            │
  │     → Estado atual: monolito, 50 services, Java 11,         │
  │       single-region, 30min deploy, 99.9% uptime              │
  │                                                              │
  │  2. ONDE QUEREMOS ESTAR EM 2-3 ANOS?                         │
  │     → Estado futuro: microservices maduros, Java 21+,        │
  │       multi-region, < 10min deploy, 99.99% uptime            │
  │                                                              │
  │  3. POR QUE ESSE DESTINO?                                    │
  │     → Habilita LATAM expansion (business driver)             │
  │     → Suporta 50x scale (growth projection)                  │
  │     → Reduz incident rate em 80% (reliability need)          │
  │     → Atrai e retém talento (competitive hiring)             │
  │                                                              │
  │  4. QUAIS OS PRINCÍPIOS QUE GUIAM AS DECISÕES?              │
  │     → "Prefer boring technology"                              │
  │     → "API-first design"                                      │
  │     → "Build for failure"                                     │
  │     → "Observability as a first-class citizen"                │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘

  TECHNICAL VISION ≠:
  
  ❌ Lista de tecnologias que queremos usar
  ❌ Roadmap de projetos (isso é EXECUTION PLAN)
  ❌ Documento que um Architect escreveu sozinho
  ❌ PowerPoint bonito guardado no Google Drive
  
  TECHNICAL VISION =:
  
  ✅ Narrativa clara do estado atual → estado futuro
  ✅ Conectada a Business Strategy
  ✅ Informa milhares de micro-decisões diárias
  ✅ Referenciada em RFCs, ADRs, design reviews
  ✅ Viva: revisitada e atualizada a cada 6 meses

  POR QUE IMPORTA:

  SEM VISÃO:                        COM VISÃO:
  ┌──────────────────────┐          ┌──────────────────────────┐
  │ Time A escolhe Kafka │          │ "Nossa visão diz event-  │
  │ Time B escolhe RabbitMQ│        │  driven com Kafka. ADR   │
  │ Time C escolhe SQS   │          │  para exceções."         │
  │ → 3 soluções para    │          │ → 1 solução, 3 times    │
  │   1 problema          │          │   alinhados              │
  │ → 3x custo de maint  │          │ → 1x custo de maint     │
  │ → ninguém tem context│          │ → shared expertise       │
  └──────────────────────┘          └──────────────────────────┘
```

---

## Papel de Cada Nível na Visão Técnica

```
QUEM FAZ O QUÊ:

  ┌────────────────────────────────────────────────────────────┐
  │ PAPEL              │ CONTRIBUIÇÃO PARA VISÃO TÉCNICA       │
  ├────────────────────┼───────────────────────────────────────┤
  │                    │                                       │
  │ Tech Lead          │ INPUT PROVIDER + EXECUTOR             │
  │                    │ → Traz reality check do time          │
  │                    │ → Reviews RFCs com olhar prático      │
  │                    │ → Traduz visão em decisões diárias    │
  │                    │ → Mantém time alinhado com visão      │
  │                    │ → ADRs do time consistent com visão   │
  │                    │                                       │
  │ Staff Engineer     │ CO-AUTHOR + ADVOCATE                  │
  │                    │ → Co-escreve partes do vision doc     │
  │                    │ → Propõe RFCs que implementam visão   │
  │                    │ → Design reviews verificam alinhamento│
  │                    │ → Evangeliza visão nos 2-3 times      │
  │                    │ → Identifica gaps entre visão e       │
  │                    │   realidade                           │
  │                    │                                       │
  │ Principal Engineer │ PRIMARY AUTHOR + STEWARD              │
  │                    │ → Escreve/lidera o vision doc         │
  │                    │ → Define Tech Radar org-wide          │
  │                    │ → Garante consistency cross-org       │
  │                    │ → Apresenta para C-level              │
  │                    │ → Revisa e evolui a cada 6 meses     │
  │                    │ → Final approver de RFCs estruturais  │
  │                    │                                       │
  │ Engineering Manager│ ENABLER + COMMUNICATOR                │
  │                    │ → Aloca capacity para tech investment │
  │                    │ → Comunica prioridades para o time    │
  │                    │ → Traduz visão técnica em OKRs        │
  │                    │ → Escala bloqueios organizacionais    │
  │                    │                                       │
  │ Tech Manager       │ SPONSOR + STRATEGIST                  │
  │                    │ → Alinha visão com business strategy  │
  │                    │ → Aprova budget para investment       │
  │                    │ → Defende investment para VP/CTO      │
  │                    │ → Org design para executar visão      │
  │                    │                                       │
  └────────────────────┴───────────────────────────────────────┘
```

---

## Criando o Documento de Visão Técnica

```
TEMPLATE DE TECHNICAL VISION DOCUMENT:

  ══════════════════════════════════════════════════════════════
  SEÇÃO 1: EXECUTIVE SUMMARY (1 página)
  ══════════════════════════════════════════════════════════════

  → 1 parágrafo: contexto de negócio
  → 1 parágrafo: estado atual (com números)
  → 1 parágrafo: estado futuro desejado (com números)
  → 1 parágrafo: investimento necessário (high-level)
  → Bullet list: top 5 iniciativas

  EXEMPLO:
  ┌────────────────────────────────────────────────────────────┐
  │ "[Empresa] está planejando expansão para LATAM e           │
  │ crescimento de 10x em base de usuários em 24 meses.        │
  │                                                            │
  │ Atualmente operamos single-region (us-east-1) com 50       │
  │ microservices (80% Java 11, 20% Python), deploy médio de   │
  │ 45min, SLA de 99.9%, e 3.2 P1 incidents/mês.              │
  │                                                            │
  │ Em 24 meses, nossa plataforma será multi-region (3 AZs,    │
  │ 2 regions), 100% Java 21, deploy < 10min, SLA 99.99%,     │
  │ e < 0.5 P1 incidents/mês.                                  │
  │                                                            │
  │ Isso requer ~25% de capacity de engineering dedicado       │
  │ a tech investment ao longo de 8 quarters."                  │
  └────────────────────────────────────────────────────────────┘

  ══════════════════════════════════════════════════════════════
  SEÇÃO 2: STRATEGIC CONTEXT (2-3 páginas)
  ══════════════════════════════════════════════════════════════

  → Business drivers (por que agora?)
  → Growth projections (quanto precisamos escalar?)
  → Competitive landscape (o que concorrentes fazem?)
  → Talent market (o que atrai/retém engenheiros?)
  → Regulatory (compliance requirements)
  → Technology trends (o que amadureceu e é relevante?)

  ══════════════════════════════════════════════════════════════
  SEÇÃO 3: CURRENT STATE — AS-IS (3-5 páginas)
  ══════════════════════════════════════════════════════════════

  Com NÚMEROS e diagramas:

  ARCHITECTURE:
  ┌──────────────────────────────────────────────────────────┐
  │ 50 microservices (42 Java 11, 8 Python 3.8)              │
  │ 1 monolith (billing, legacy Java 8, 200K LOC)           │
  │ PostgreSQL (3 clusters), DynamoDB (12 tables)            │
  │ RabbitMQ (message broker, single instance)               │
  │ EKS (1 cluster, us-east-1, 80 nodes)                    │
  │ Single-region deployment                                  │
  │ CI/CD: GitHub Actions, avg build: 45 min                 │
  │ Observability: Datadog (logs + APM), minimal tracing     │
  └──────────────────────────────────────────────────────────┘

  HEALTH INDICATORS:
  │ Métrica                  │ Atual        │ Target      │
  │ Deploy frequency         │ 0.5/dia      │ 5/dia       │
  │ Lead time               │ 5 dias        │ 1 dia       │
  │ MTTR                    │ 45 min        │ 10 min      │
  │ Change failure rate     │ 15%           │ 5%          │
  │ P1 incidents/mês         │ 3.2           │ 0.5         │
  │ Developer satisfaction   │ 6.2/10        │ 8.5/10      │
  │ Uptime SLA              │ 99.9%         │ 99.99%      │

  TECH DEBT:
  → Top 5 tech debts com custo estimado
  → Diagrama mostrando pain points

  ══════════════════════════════════════════════════════════════
  SEÇÃO 4: FUTURE STATE — TO-BE (3-5 páginas)
  ══════════════════════════════════════════════════════════════

  TARGET ARCHITECTURE:
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  ┌────────┐    ┌────────┐    ┌────────┐                 │
  │  │Region 1│    │Region 2│    │Region 3│    Multi-Region│
  │  │us-east │    │sa-east │    │eu-west │                 │
  │  └───┬────┘    └───┬────┘    └───┬────┘                 │
  │      │             │             │                       │
  │      └─────────────┼─────────────┘                       │
  │                    │                                     │
  │           ┌────────┴────────┐                            │
  │           │ Global Load     │                            │
  │           │ Balancer (Route53)│                          │
  │           └────────┬────────┘                            │
  │                    │                                     │
  │      ┌─────────────┼─────────────┐                      │
  │      │             │             │                       │
  │  ┌───┴────┐   ┌────┴───┐   ┌────┴───┐                  │
  │  │API GW  │   │API GW  │   │API GW  │                  │
  │  └───┬────┘   └────┬───┘   └────┬───┘                  │
  │      │             │             │                       │
  │  ┌───┴─────────────┴─────────────┴───┐                  │
  │  │    EKS Cluster (per region)       │  Kubernetes     │
  │  │    60+ services (Java 21)         │                  │
  │  │    Auto-scaling (HPA + KEDA)      │                  │
  │  └──────────────────────────────────┘                  │
  │                    │                                     │
  │  ┌─────────────────┴──────────────────┐                 │
  │  │ Kafka (event backbone)              │  Event-Driven │
  │  │ Schema Registry                     │                 │
  │  └─────────────────┬──────────────────┘                 │
  │                    │                                     │
  │  ┌────────┐  ┌─────┴───┐  ┌──────────┐                 │
  │  │Postgres│  │DynamoDB  │  │S3 + Athena│  Data Layer   │
  │  │(Aurora │  │(Global   │  │(Analytics)│                 │
  │  │Global) │  │Tables)   │  │           │                 │
  │  └────────┘  └─────────┘  └──────────┘                 │
  │                                                          │
  │  ┌──────────────────────────────────────┐               │
  │  │ OpenTelemetry → Grafana Stack        │  Observability│
  │  │ (Traces + Metrics + Logs unified)    │                │
  │  └──────────────────────────────────────┘               │
  │                                                          │
  └──────────────────────────────────────────────────────────┘

  ══════════════════════════════════════════════════════════════
  SEÇÃO 5: ENGINEERING PRINCIPLES (1-2 páginas)
  ══════════════════════════════════════════════════════════════

  → 5-8 princípios que guiam todas as decisões
  → Cada princípio com: statement + rationale + example
  → (detalhado na seção Engineering Principles abaixo)

  ══════════════════════════════════════════════════════════════
  SEÇÃO 6: STRATEGIC INITIATIVES (3-5 páginas)
  ══════════════════════════════════════════════════════════════

  → Top 5-8 initiatives para chegar do AS-IS ao TO-BE
  → Para cada: justificativa, scope, effort estimate, dependencies
  → Sequencing: o que vem primeiro e por quê

  ══════════════════════════════════════════════════════════════
  SEÇÃO 7: RISKS AND MITIGATIONS (1-2 páginas)
  ══════════════════════════════════════════════════════════════

  │ Risco                    │ Impacto │ Probabilidade │ Mitigação         │
  │ Contratação de 5 SREs   │ Alto    │ Média         │ Upskill internos  │
  │ Kafka expertise gap      │ Alto    │ Alta          │ Consultoria + hire│
  │ Monolith mais complexo   │ Médio   │ Alta          │ Feature freeze    │
  │ Budget cortado 30%       │ Alto    │ Baixa         │ Priorizar 3 top   │

  ══════════════════════════════════════════════════════════════
  SEÇÃO 8: SUCCESS METRICS (1 página)
  ══════════════════════════════════════════════════════════════

  → Como saberemos que a visão está se materializando?
  → OKRs de 6 meses e de 2 anos
  → Leading indicators (não só lagging)
```

---

## Tech Radar — Governança do Portfolio

```
TECH RADAR — MAPA DO PORTFOLIO TECNOLÓGICO:

  ┌───────────────────────────────────────────────────┐
  │                                                   │
  │                    HOLD                            │
  │              ┌─────────────┐                      │
  │              │             │                      │
  │          ┌───┤   ASSESS    ├───┐                  │
  │          │   │             │   │                  │
  │       ┌──┤   ├─────────────┤   ├──┐               │
  │       │  │   │   TRIAL     │   │  │               │
  │       │  │   │             │   │  │               │
  │    ┌──┤  │   ├─────────────┤   │  ├──┐            │
  │    │  │  │   │   ADOPT     │   │  │  │            │
  │    │  │  │   │             │   │  │  │            │
  │    │  │  │   └─────────────┘   │  │  │            │
  │    │  └──┘                     └──┘  │            │
  │    └─────┘                     └─────┘            │
  │   Languages    Platforms   Tools   Techniques     │
  │                                                   │
  └───────────────────────────────────────────────────┘

  RINGS:
  ┌───────────┬───────────────────────────────────────────────┐
  │ ADOPT     │ Padrão da organização. Usar por default.      │
  │           │ "Se não tem motivo para diferente, use isto." │
  ├───────────┼───────────────────────────────────────────────┤
  │ TRIAL     │ Validamos que funciona. Pronto para adoção.   │
  │           │ "Pode usar em produção com acompanhamento."  │
  ├───────────┼───────────────────────────────────────────────┤
  │ ASSESS    │ Promissor. Fazendo spike/prova de conceito.   │
  │           │ "Investigando. Não usar em produção ainda."  │
  ├───────────┼───────────────────────────────────────────────┤
  │ HOLD      │ Não adotar mais. Migrar quando possível.      │
  │           │ "Existente OK, mas não crie nada novo."      │
  └───────────┴───────────────────────────────────────────────┘

  EXEMPLO — TECH RADAR DE UMA PLATAFORMA:

  LANGUAGES:
    ADOPT: Java 21, TypeScript 5.x, Python 3.12
    TRIAL: Kotlin (para novos services), Rust (para edge)
    ASSESS: Go (para CLIs e infra tooling)
    HOLD: Java 8, Java 11, Python 2.x, JavaScript (raw)

  PLATFORMS:
    ADOPT: EKS, Aurora PostgreSQL, DynamoDB, S3
    TRIAL: Kafka (MSK), OpenSearch
    ASSESS: Graviton (ARM), KEDA
    HOLD: EC2 bare (prefer containers), RabbitMQ

  TOOLS:
    ADOPT: GitHub Actions, Terraform, ArgoCD, Datadog
    TRIAL: OpenTelemetry, Grafana Stack
    ASSESS: Backstage (IDP), Pulumi
    HOLD: Jenkins, CloudFormation, manual deploys

  TECHNIQUES:
    ADOPT: ADRs, Feature Flags, Trunk-based dev, SLOs
    TRIAL: Event-Driven Architecture, Chaos Engineering
    ASSESS: Platform Engineering, eBPF observability
    HOLD: Long-lived branches, manual QA gates

  COMO CRIAR E MANTER O TECH RADAR:

  PASSO 1: Input (anual, com reviews semestrais)
  → Cada Staff/Principal propõe items com justificativa
  → EMs trazem input de developer satisfaction

  PASSO 2: Review Session
  → 2-4 horas com Staff+, Principals, CTO
  → Debate cada item controverso (consenso ou owner decide)
  → Documenta decisão + rationale para cada mudança

  PASSO 3: Publicação
  → Publicar radar visual + document descritivo
  → All-hands walkthrough (30 min)
  → Q&A session (30 min)

  PASSO 4: Enforcement
  → Novos serviços: devem usar stack ADOPT
  → Exceções: precisam de ADR + aprovação Staff/Principal
  → HOLD items: migration plan com timeline
  → Review: a cada 6 meses, revisitar o radar

  RESPONSABILIDADE POR NÍVEL:
  │ Papel      │ Responsabilidade no Radar                │
  │ Tech Lead  │ Propor items, compliance no time          │
  │ Staff      │ Propor + advocate, migration planning     │
  │ Principal  │ Curate radar, final decisions, publish    │
  │ Tech Mgr   │ Budget para migrations, enforce adoption │
```

---

## RFCs — Propondo Mudanças Técnicas

```
RFC (REQUEST FOR COMMENTS) — O VEÍCULO DA MUDANÇA:

  QUANDO ESCREVER UM RFC:
  ✅ Mudança afeta mais de 1 time
  ✅ Nova tecnologia no stack
  ✅ Mudança de padrão ou convenção
  ✅ Investimento > 2 person-months
  ✅ Decisão irreversível ou difícil de reverter (Type 1)
  
  QUANDO NÃO PRECISA:
  ❌ Mudança local no time (use ADR)
  ❌ Bug fix, feature normal
  ❌ Já coberto por princípio/padrão existente

  ══════════════════════════════════════════════════════════
  RFC TEMPLATE:
  ══════════════════════════════════════════════════════════

  ┌────────────────────────────────────────────────────────┐
  │ RFC-042: Migração de Message Broker para Apache Kafka  │
  │                                                        │
  │ Author: João Silva (Staff Engineer, Platform)          │
  │ Status: PROPOSED → REVIEW → ACCEPTED/REJECTED          │
  │ Date: 2026-02-24                                       │
  │ Reviewers: Maria (Principal), Pedro (Staff, Payments), │
  │            Ana (EM, Platform)                           │
  │ Decision deadline: 2026-03-10                          │
  │                                                        │
  │ ─────────────────────────────────────────────          │
  │ 1. CONTEXTO E PROBLEMA                                │
  │ ─────────────────────────────────────────────          │
  │ Atualmente usamos RabbitMQ como message broker.        │
  │ Com crescimento de 10x projetado e necessidade de      │
  │ event sourcing, RabbitMQ apresenta limitações:         │
  │ - Sem replay de mensagens                              │
  │ - Desempenho degrada > 50K msg/sec                     │
  │ - Single point of failure (não multi-AZ nativo)        │
  │                                                        │
  │ ─────────────────────────────────────────────          │
  │ 2. PROPOSTA                                            │
  │ ─────────────────────────────────────────────          │
  │ Migrar para Apache Kafka (AWS MSK) como event          │
  │ backbone da plataforma ao longo de 6 meses,            │
  │ usando Strangler Fig pattern.                          │
  │                                                        │
  │ ─────────────────────────────────────────────          │
  │ 3. ALTERNATIVAS CONSIDERADAS                          │
  │ ─────────────────────────────────────────────          │
  │ A. Manter RabbitMQ + escalar horizontalmente           │
  │    Pro: menor esforço. Con: não resolve replay.        │
  │ B. Amazon EventBridge                                  │
  │    Pro: serverless. Con: vendor lock-in, latency.      │
  │ C. Apache Kafka (RECOMENDADO)                          │
  │    Pro: replay, scale, ecosystem. Con: complexity.     │
  │                                                        │
  │ ─────────────────────────────────────────────          │
  │ 4. DESIGN DETALHADO                                   │
  │ ─────────────────────────────────────────────          │
  │ (diagramas, schemas, configurações, examples)         │
  │                                                        │
  │ ─────────────────────────────────────────────          │
  │ 5. IMPACTO E RISCOS                                   │
  │ ─────────────────────────────────────────────          │
  │ - 3 person-months de engineering                       │
  │ - Risco: expertise gap (mitigação: treinamento)        │
  │ - Risco: dual-write durante migração                   │
  │                                                        │
  │ ─────────────────────────────────────────────          │
  │ 6. PLANO DE MIGRAÇÃO                                  │
  │ ─────────────────────────────────────────────          │
  │ Phase 1: Setup MSK + 1 pilot service (4 weeks)        │
  │ Phase 2: Migrate tier-1 services (6 weeks)             │
  │ Phase 3: Migrate remaining + decommission Rabbit       │
  │                                                        │
  │ ─────────────────────────────────────────────          │
  │ 7. MÉTRICAS DE SUCESSO                                │
  │ ─────────────────────────────────────────────          │
  │ - 100% services migrados em 6 meses                    │
  │ - Zero data loss durante migração                      │
  │ - Throughput > 200K msg/sec sustentado                  │
  │                                                        │
  └────────────────────────────────────────────────────────┘

  RFC LIFECYCLE:

  DRAFT ──→ PROPOSED ──→ REVIEW ──→ DECISION
    │           │           │           │
    │    share com         2 weeks     ACCEPTED
    │    stakeholders      para        → execução
    │                      comments    REJECTED
    │                                  → archive + rationale
    │                                  DEFERRED
    │                                  → revisitar em N meses
    
  REGRAS DO PROCESSO:
  → Author: qualquer engineer, idealmente Staff+
  → Review period: mínimo 1 semana, máximo 2 semanas
  → Approvers: Principal + EM + affected teams
  → "Silence is not consent" — requer approval explícito
  → Após accepted: gera ADR + tasks no roadmap

  QUEM ESCREVE RFCs (por nível):

  │ Nível     │ Tipo de RFC                              │
  │ Tech Lead │ Decisão técnica do time (mais ADR scope) │
  │ Staff     │ Mudança que afeta 2-3 times / domínio    │
  │ Principal │ Mudança org-wide / nova tecnologia no stack│
  │ Tech Mgr  │ Processo / org-structure changes          │
```

---

## Engineering Principles

```
ENGINEERING PRINCIPLES — GUIAS PARA DECISÕES:

  POR QUE PRINCÍPIOS:
  → Você não pode estar em todas as decisões
  → Princípios são "você em escala"
  → Empoderam decisões descentralizadas
  → Criam consistência sem micromanagement

  COMO CRIAR BONS PRINCÍPIOS:

  ┌────────────────────────────────────────────────────────┐
  │ BOM PRINCÍPIO:                                        │
  │ → Opinionated (exclui alternativas)                   │
  │ → Actionable (guia decisão concreta)                  │
  │ → Testable ("esta decisão segue o princípio?")       │
  │ → Documented (rationale + example)                   │
  │                                                       │
  │ MAU PRINCÍPIO:                                        │
  │ → Vago ("quality first")                              │
  │ → Óbvio ("write good code")                           │
  │ → Sem trade-off explícito                             │
  │ → Impossível discordar                                │
  └────────────────────────────────────────────────────────┘

  TEMPLATE:

  PRINCÍPIO: [Nome curto]
  STATEMENT: [1 frase clara]
  RATIONALE: [Por que escolhemos isso, que trade-off aceitamos]
  EXAMPLE: [Caso concreto onde o princípio guia a decisão]
  COUNTER-EXAMPLE: [Caso onde NÃO seguir e por quê]

  EXEMPLO:

  PRINCÍPIO: API-First Design
  STATEMENT: "Todo serviço expõe uma API bem definida como 
  contrato, antes de implementar lógica."
  RATIONALE: Reduz acoplamento, habilita desenvolvimento 
  paralelo, força clareza de interface. Trade-off: mais 
  upfront design, pode parecer lento no início.
  EXAMPLE: Novo serviço de notificações — definir OpenAPI 
  spec primeiro, mock endpoints, times consumidores validam 
  contrato, depois implementar.
  COUNTER-EXAMPLE: Script interno one-off — overhead de 
  API formal não justifica.

  ── EXEMPLO DE SET DE PRINCÍPIOS ──────────────────────

  1. "Choose Boring Technology"
     → Inovação se paga com tokens limitados
     → Usar tech madura por default, inovar com justificativa
  
  2. "API-First Design"
     → Contratos antes de implementação
     → OpenAPI/AsyncAPI spec como primeiro artefato
  
  3. "Build for Failure"
     → Todo componente falha eventualmente
     → Circuit breakers, retries, fallbacks by default
  
  4. "Observe Everything"
     → Se não tem metric, não existe
     → Tracing, metrics, logs em todo serviço novo
  
  5. "Own Your Data"
     → Cada serviço é dono do seu data store
     → Zero database sharing entre serviços
  
  6. "Prefer Reversible Decisions"
     → Feature flags > big bang releases
     → Canary/blue-green > cutover
     → Decisions que podem ser undone com facilidade
  
  7. "Automate the Toil"
     → Se você fez manualmente 3x, automatize
     → CI/CD, infrastructure-as-code, automated testing
  
  8. "Security is Everyone's Job"
     → Não é "o time de security resolve"
     → Shift-left: security no design, no CI, no review
```

---

## De Visão a Execução

```
O GAP ENTRE VISÃO E EXECUÇÃO:

  "Vision without execution is hallucination." — Thomas Edison

  VISÃO                    BRIDGE                    EXECUÇÃO
  ┌─────────────┐         ┌─────────────┐          ┌─────────────┐
  │ Vision Doc  │ ──→     │ Tech Roadmap│ ──→      │ OKRs/Quarter│
  │ (2-3 anos)  │         │(Now/Next/   │          │ plans       │
  │             │         │ Later)      │          │             │
  │ Principles  │ ──→     │ Engineering │ ──→      │ ADRs no dia │
  │             │         │ Standards   │          │ a dia       │
  │             │         │             │          │             │
  │ Tech Radar  │ ──→     │ Migration   │ ──→      │ PRs e tasks │
  │             │         │ Plans       │          │ concretas   │
  └─────────────┘         └─────────────┘          └─────────────┘
       ↑                                                │
       └─────── FEEDBACK LOOP (metrics, retros) ────────┘

  PROCESSO CONCRETO:

  1. VISÃO (Principal + CTO):
     "Em 2 anos, multi-region, event-driven, 99.99%"
     
  2. ROADMAP (Principal + Staff):
     NOW: CI/CD optimization, OpenTelemetry
     NEXT: Kafka migration, Java 21
     LATER: Multi-region, auto-scaling
     
  3. OKRs (Staff + EMs):
     Q1: "Pipeline P95 < 15min, OTel em 50% services"
     
  4. TASKS (Tech Leads + Teams):
     Sprint tasks derivados dos OKRs
     ADRs para cada decisão significativa
     
  5. EXECUTION (Teams):
     PRs, code reviews, deployments
     → CADA PR deveria ser rastreável até um OKR
     
  6. FEEDBACK (All):
     DORA metrics, incident trends, dev satisfaction
     → Input para próximo cycle de planning

  RESPONSABILIDADE POR NÍVEL:
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  Principal    Staff       Tech Lead    Time              │
  │  ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐              │
  │  │VISÃO│───→│ROAD │───→│OKRs │───→│TASKS│              │
  │  │     │    │ MAP │    │     │    │     │              │
  │  └─────┘    └─────┘    └─────┘    └─────┘              │
  │  "Para onde  "O que     "Quanto    "Como               │
  │   vamos?"    fazer?"    ?"         fazer?"              │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

---

## Comunicação da Visão

```
COMUNICAÇÃO — O 80% DO TRABALHO DO LEADER:

  "Você não comunicou até que a pessoa consiga 
   explicar para outro." — Tanya Reilly

  AUDIÊNCIA E FORMATO:

  ┌─────────────────────────────────────────────────────────┐
  │ AUDIÊNCIA        │ FORMATO          │ FOCO              │
  ├──────────────────┼──────────────────┼───────────────────┤
  │ C-level / Board  │ 1-pager +        │ Business impact,  │
  │                  │ 15 min talk      │ ROI, risk         │
  │                  │                  │                   │
  │ VP / Directors   │ Vision doc +     │ Strategy, budget, │
  │                  │ 30 min talk      │ timeline, trade-  │
  │                  │                  │ offs              │
  │                  │                  │                   │
  │ Eng Managers     │ Roadmap +        │ Capacity, OKRs,   │
  │                  │ planning session │ team allocation   │
  │                  │                  │                   │
  │ Staff/Principals │ Full vision doc +│ Architecture,     │
  │                  │ working session  │ principles, gaps  │
  │                  │                  │                   │
  │ Tech Leads       │ Roadmap + OKRs + │ What changes for  │
  │                  │ Q&A session      │ their team, how   │
  │                  │                  │                   │
  │ All Engineers    │ All-hands +      │ Why, what's in it │
  │                  │ Tech Radar       │ for me, timeline  │
  │                  │                  │                   │
  │ Product/Design   │ Joint planning + │ How enables       │
  │                  │ dependency map   │ product roadmap   │
  └──────────────────┴──────────────────┴───────────────────┘

  TÁTICAS DE COMUNICAÇÃO POR NÍVEL:

  TECH LEAD:
  → Traduz visão em linguagem do time
  → Mostra como afeta o dia a dia
  → "Nosso próximo projeto usa o padrão X da visão"
  → Usa ADRs para referenciar princípios

  STAFF:
  → Escreve RFCs que implementam a visão
  → Apresenta em guilds e working groups
  → Faz workshops hands-on ("como usar Kafka")
  → Blog posts internos com deep dives

  PRINCIPAL:
  → Apresenta para C-level (business language)
  → Keynote no engineering all-hands
  → 1:1s com Staff Engineers para alignment
  → External talks, blog posts (employer branding)

  TECH MANAGER:
  → Alinha com product leadership
  → Budget conversations com VP/CFO
  → Org design conversations com HR
  → Monthly update para stakeholders

  REGRA DE OURO: REPETIÇÃO
  ┌────────────────────────────────────────────────────┐
  │ "Se você não está cansado de repetir a visão,     │
  │  as pessoas ainda não ouviram."                    │
  │                                                    │
  │ → Diga na all-hands                                │
  │ → Diga no RFC                                      │
  │ → Diga no design review                            │
  │ → Diga no 1:1                                      │
  │ → Diga no Slack                                    │
  │ → Diga na weekly update                            │
  │ → Diga no onboarding de novos engineers            │
  └────────────────────────────────────────────────────┘
```

---

## Mantendo a Visão Viva

```
VISÃO TÉCNICA NÃO É ESTÁTICA:

  CADÊNCIA DE REVISÃO:
  
  │ Atividade                   │ Frequência  │ Quem              │
  │ Full vision doc review       │ Anual       │ Principal + CTO   │
  │ Tech Radar update            │ Semestral   │ Principal + Staff │
  │ Principles review            │ Anual       │ Staff + community │
  │ Roadmap refresh              │ Trimestral  │ Staff + EMs       │
  │ OKR review                   │ Trimestral  │ EMs + Tech Leads  │
  │ Alignment check (are we on?) │ Mensal      │ Principal         │

  SINAIS QUE A VISÃO PRECISA DE UPDATE:

  1. EXTERNAL CHANGE
  → Nova regulação (compliance requirement)
  → Competitor lançou algo que muda o jogo
  → Vendor changed pricing/license (ex: Terraform BSL)
  → New tech matured (ex: WASM, AI code generation)

  2. INTERNAL CHANGE
  → Business pivot ou M&A
  → Hiring plan mudou significativamente
  → Budget cut/expansion
  → New leadership (CTO, VP)

  3. EXECUTION SIGNALS
  → Visão de 2 anos já está 80% done em 1 ano → expand
  → Key initiative bloqueada há 2 quarters → replan
  → Tech debt acumulou mais rápido que previsto → reprioritize
  → Developer satisfaction caiu → investigate cause

  COMO MANTER VISÃO RELEVANTE:

  ┌────────────────────────────────────────────────────────┐
  │ 1. ANCHOR em Business Outcomes                        │
  │    → Se business muda, visão muda                     │
  │    → "Vamos para multi-region PORQUE LATAM expansion" │
  │                                                       │
  │ 2. MEASURE Progress                                   │
  │    → DORA metrics como proxy de saúde                 │
  │    → KRs que mostram evolução                         │
  │                                                       │
  │ 3. CELEBRATE Milestones                               │
  │    → "Java 21 migration 100% done!" → all-hands demo  │
  │    → Visibilidade mantém momentum                     │
  │                                                       │
  │ 4. TELL STORIES                                       │
  │    → "Time de Payments migrou para Kafka e reduziu     │
  │       19 incidents/quarter para 2"                     │
  │    → Narrativa > número                                │
  │                                                       │
  │ 5. KILL Items Honestly                                │
  │    → "Decidimos não fazer multi-region porque          │
  │       não vamos expandir LATAM este ano"               │
  │    → Transparência > pretender que nada mudou          │
  │                                                       │
  └────────────────────────────────────────────────────────┘
```

---

## Anti-Patterns

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **Vision in the drawer** | Documento criado, ninguém consulta | Referenciar em ADRs, RFCs, reviews; repetir constantemente |
| **One-person vision** | Architect ou Principal cria sozinho | Input de Staff, TLs, EMs, Product; collaborative sessions |
| **No current state** | Só descreve futuro, ignora onde estamos | Assessment honesto do AS-IS com métricas |
| **Technology shopping** | Visão = "vamos usar K8s + Kafka + Rust" | Visão = outcomes e princípios, não stack |
| **Unchangeable vision** | "Definimos, não pode mudar" | Review semestral, adaptar a mudanças de contexto |
| **No connection to business** | Visão técnica desconectada de strategy | Cada initiative ligada a business driver |
| **All vision, no execution** | Documento lindo, nada implementado | Traduzir em roadmap → OKRs → tasks |
| **Principle without teeth** | "API-first" mas ninguém verifica | Enforcement em design reviews, RFCs |
| **Radar without migration** | HOLD items sem timeline de saída | Todo HOLD precisa de migration plan |
| **Vision scope creep** | Tenta resolver tudo de uma vez | 3-5 strategic themes MAX; foco é poder |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e propostas, considere a visão técnica:

1. **Alinhamento com princípios** — Verificar se decisões no código seguem os Engineering Principles documentados da organização
2. **Tech Radar compliance** — Se introduz nova tecnologia/library, verificar se está em ADOPT/TRIAL no radar; se não, exigir ADR/RFC
3. **Vision-connected RFC** — Mudança arquitetural grande precisa de RFC que referencia a visão técnica
4. **Future state direction** — Se código reforça estado ATUAL (legacy) quando já existe caminho para estado FUTURO, sugerir migração
5. **Principle enforcement** — "Choose Boring Technology" → bibliotecas exóticas precisam de justificativa forte
6. **API-first check** — Novo serviço sem contract/schema definido primeiro → questionar ordem
7. **Observability by default** — Código sem métricas/tracing em novo serviço → não está seguindo "Observe Everything"
8. **Documentation quality** — RFCs e ADRs devem ter contexto, alternativas e rationale; se superficial, pedir aprofundamento
9. **Cross-team impact** — Se mudança afeta outros times, verificar se foi comunicada (princípio de comunicação da visão)
10. **Reversibility preference** — Decisões irreversíveis sem feature flag ou rollback plan → questionar

---

## Referências

- **Staff Engineer** — Will Larson (2021) — https://staffeng.com/
- **The Staff Engineer's Path** — Tanya Reilly (O'Reilly, 2022)
- **An Elegant Puzzle** — Will Larson (Stripe Press, 2019)
- **Technology Strategy Patterns** — Eben Hewitt (O'Reilly, 2018)
- **Wardley Maps** — Simon Wardley — https://learnwardleymapping.com/
- **ThoughtWorks Technology Radar** — https://www.thoughtworks.com/radar
- **Build Your Own Radar** — https://www.thoughtworks.com/radar/byor
- **Choose Boring Technology** — Dan McKinley — https://boringtechnology.club/
- **Design Docs at Google** — https://www.industrialempathy.com/posts/design-docs-at-google/
- **RFC Process at Oxide** — https://oxide.computer/blog/rfd-1-requests-for-discussion
- **Writing Technical Vision** — Will Larson — https://lethain.com/writing-technical-visions/
- **Engineering Principles** — Monzo — https://monzo.com/blog/2018/06/29/engineering-principles
- **Technical Decision-Making** — Gergely Orosz — https://blog.pragmaticengineer.com/
