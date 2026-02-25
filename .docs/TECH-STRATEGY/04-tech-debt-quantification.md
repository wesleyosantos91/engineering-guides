# Tech Debt Quantification — Métricas e Priorização

> **Objetivo deste documento:** Servir como referência completa sobre **quantificação, classificação e priorização de dívida técnica** — como medir, comunicar para stakeholders e reduzir tech debt sistematicamente — otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: tipos de tech debt, modelos de quantificação, debt registers, frameworks de priorização, comunicação para não-técnicos, estratégias de redução.

---

## Quick Reference — Cheat Sheet

| Conceito | Definição | Uso |
|----------|----------|-----|
| **Tech Debt** | Custo futuro de atalhos técnicos tomados hoje | Tracking e priorização |
| **Deliberate Debt** | Atalho consciente para entregar rápido | Aceitável se documentado + plano de pagamento |
| **Inadvertent Debt** | Debt por desconhecimento ou erro | Descoberto após o fato, requer investment |
| **Interest** | Custo recorrente de manter o debt | Slowdown, bugs, incidents, attrition |
| **Principal** | Custo de eliminar o debt | Eng-weeks para refactor/rewrite |
| **Debt Register** | Inventário vivo de toda dívida técnica | Visibilidade e priorização |
| **DORA Metrics** | Deployment freq, lead time, MTTR, change failure | Proxy para medir health vs debt |

---

## Sumário

- [Tech Debt Quantification — Métricas e Priorização](#tech-debt-quantification--métricas-e-priorização)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que é Tech Debt](#o-que-é-tech-debt)
  - [Taxonomia de Tech Debt](#taxonomia-de-tech-debt)
  - [Quantificação de Tech Debt](#quantificação-de-tech-debt)
  - [Métricas Proxy para Tech Debt](#métricas-proxy-para-tech-debt)
  - [Tech Debt Register](#tech-debt-register)
  - [Frameworks de Priorização](#frameworks-de-priorização)
  - [Comunicação para Stakeholders](#comunicação-para-stakeholders)
  - [Estratégias de Redução](#estratégias-de-redução)
  - [Tech Debt Budget](#tech-debt-budget)
  - [Anti-Patterns](#anti-patterns)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que é Tech Debt

```
TECH DEBT — A METÁFORA FINANCEIRA (Ward Cunningham, 1992):

  ┌─────────────────────────────────────────────────────────────┐
  │                                                              │
  │  DÍVIDA FINANCEIRA:                                          │
  │  → Você toma empréstimo (principal)                          │
  │  → Paga juros mensais (interest)                             │
  │  → Enquanto não pagar: juros acumulam                        │
  │  → Eventualmente: paga o principal + juros                   │
  │                                                              │
  │  DÍVIDA TÉCNICA:                                             │
  │  → Você toma um atalho técnico (principal)                   │
  │  → Paga "juros" em velocidade: cada feature demora mais     │
  │  → Enquanto não refatorar: produtividade cai progressivamente│
  │  → Eventualmente: refatora (paga principal) ou sistema morre │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘

  ANALOGIA VISUAL:

  Velocidade de
  entrega
  ▲
  │ ████                                        
  │ ████ ████                                   ← SEM debt
  │ ████ ████ ████ ████ ████ ████ ████ ████     (velocity constante)
  │ ────────────────────────────────────────
  │ ████ 
  │ ████ ████
  │ ████ ████ ███                               ← COM debt não-pago
  │ ████ ████ ███ ██                             (velocity decrescente)
  │ ████ ████ ███ ██ █ · · ·                    
  │──────────────────────────────────────────► Tempo
  │      Q1    Q2   Q3  Q4  Q5

  A "velocidade" cai porque cada mudança:
  → Exige entender código mais complexo
  → Tem mais efeitos colaterais inesperados
  → Quebra coisas não relacionadas
  → Exige mais testing manual (testes automatizados insuficientes)
  → Demora mais para deploy (pipeline frágil)

  TECH DEBT NÃO É:
  ❌ Qualquer código que você não gosta
  ❌ Código legado (pode ser legacy saudável)
  ❌ Desculpa para não entregar features
  ❌ Perfeccionismo técnico disfarçado
  
  TECH DEBT É:
  ✅ Atalho CONSCIENTE com trade-off documentado
  ✅ Código que ATIVAMENTE desacelera o time
  ✅ Arquitetura que IMPEDE evolução do sistema
  ✅ Falta de automação que CAUSA erros recorrentes
  ✅ Dependências DESATUALIZADAS com vulnerabilidades
```

---

## Taxonomia de Tech Debt

```
QUADRANTE DE TECH DEBT (Martin Fowler, baseado em Cunningham):

                    DELIBERADA                   INADVERTIDA
                    (consciente)                 (inconsciente)
                    
  ┌─────────────────────────────┬──────────────────────────────┐
  │                             │                              │
  │  PRUDENTE + DELIBERADA      │  PRUDENTE + INADVERTIDA      │
  │                             │                              │
  │  "Sabemos que é atalho,     │  "Agora entendemos como      │
  │   mas precisamos entregar   │   deveríamos ter feito"       │
  │   até sexta. Vamos pagar    │                              │
  │   na sprint seguinte."      │  Só percebemos depois de     │
  │                             │  aprender mais sobre          │
  │  ✅ Aceitável se:            │  o domínio                   │
PRUD│  - Documentado (ADR)       │                              │
ENTE│  - Plan de pagamento       │  ✅ Normal no                │
  │  - Tracking no backlog      │  desenvolvimento              │
  │  - Prazo definido            │  - Refatorar quando aprender │
  │                             │  - Não é "erro", é evolução  │
  │                             │                              │
  ├─────────────────────────────┼──────────────────────────────┤
  │                             │                              │
  │  IMPRUDENTE + DELIBERADA    │  IMPRUDENTE + INADVERTIDA    │
  │                             │                              │
  │  "Não temos tempo para      │  "O que é design patterns?"  │
  │   design, just ship it"     │                              │
  │                             │  Debt por falta de            │
  │  ❌ Perigosa:                │  conhecimento/skill          │
IMPR│  - Sem documentação        │                              │
UDNT│  - Sem plano de pagamento  │  ❌ Mais perigosa:            │
  E │  - "Vamos arrumar depois"  │  - Não sabe que tem debt     │
  │  - "Depois" nunca chega     │  - Não sabe como resolver    │
  │                             │  - Precisa: treinamento,      │
  │                             │    mentoria, pairing          │
  │                             │                              │
  └─────────────────────────────┴──────────────────────────────┘

  TIPOS DE TECH DEBT POR CATEGORIA:

  ┌──────────────────────────────────────────────────────────────┐
  │ CATEGORIA          │ EXEMPLOS                                │
  ├────────────────────┼──────────────────────────────────────────┤
  │ Code Debt          │ - Código duplicado                      │
  │                    │ - God classes / functions                │
  │                    │ - Dead code                              │
  │                    │ - Magic numbers / hardcoded values       │
  │                    │ - Missing error handling                 │
  ├────────────────────┼──────────────────────────────────────────┤
  │ Architecture Debt  │ - Monolito difícil de escalar           │
  │                    │ - Circular dependencies                 │
  │                    │ - Shared database entre serviços         │
  │                    │ - Missing abstractions                  │
  │                    │ - Tight coupling                        │
  ├────────────────────┼──────────────────────────────────────────┤
  │ Test Debt          │ - Baixa coverage em paths críticos      │
  │                    │ - Testes frágeis (flaky tests)          │
  │                    │ - Missing integration tests             │
  │                    │ - No contract tests                     │
  │                    │ - Manual QA como único gate              │
  ├────────────────────┼──────────────────────────────────────────┤
  │ Infrastructure Debt│ - Infra manual (no IaC)                 │
  │                    │ - Snowflake servers                     │
  │                    │ - Missing auto-scaling                  │
  │                    │ - No disaster recovery testado          │
  │                    │ - CI/CD pipeline lento/frágil           │
  ├────────────────────┼──────────────────────────────────────────┤
  │ Dependency Debt    │ - Libraries desatualizadas              │
  │                    │ - CVEs não resolvidos                   │
  │                    │ - EOL runtime (Java 8, Python 2)        │
  │                    │ - Framework sem suporte                 │
  ├────────────────────┼──────────────────────────────────────────┤
  │ Documentation Debt │ - APIs sem docs                         │
  │                    │ - Runbooks desatualizados               │
  │                    │ - Missing ADRs para decisões            │
  │                    │ - Onboarding docs inexistentes          │
  ├────────────────────┼──────────────────────────────────────────┤
  │ Process Debt       │ - Deploy manual                         │
  │                    │ - No code review process                │
  │                    │ - Missing monitoring/alerting           │
  │                    │ - On-call sem runbooks                  │
  └────────────────────┴──────────────────────────────────────────┘
```

---

## Quantificação de Tech Debt

```
COMO QUANTIFICAR — 3 MODELOS COMPLEMENTARES:

  ═══════════════════════════════════════════════════════════════
  MODELO 1: CUSTO DE ATRASO (Cost of Delay)
  "Quanto custa NÃO pagar esse debt?"
  ═══════════════════════════════════════════════════════════════

  Interest (custo recorrente por sprint):

  │ Tipo de custo          │ Como medir                         │
  │ Velocity tax           │ % do tempo gasto trabalhando       │
  │                        │ "ao redor" do debt                  │
  │ Incident cost          │ Incidents causados × custo/incident│
  │ Opportunity cost       │ Features não entregues por causa   │
  │                        │ do debt                             │
  │ Onboarding cost        │ Semanas extras para novo dev       │
  │                        │ compreender código com debt        │
  │ Attrition cost         │ Engenheiros que saem por frustração│

  EXEMPLO CONCRETO:

  Tech Debt: "Serviço de Orders com 50K linhas, 0 testes, 
              deploy manual de 4 horas"

  Interest por mês:
  → 2 dias/sprint de developer time trabalhando ao redor = $4,800
  → 1 incident/mês (P2) = $5,000 (MTTR × custo eng × opportunidade)
  → Deploy manual 4h × 4 deploys/mês × 2 eng = $3,200
  → Feature delay: 1 semana extra / feature = variable
  
  Total interest: ~$13,000/mês = $156,000/ano

  Principal (custo de resolver):
  → Adicionar testes: 3 eng × 3 semanas = $36,000
  → Automatizar CI/CD: 1 eng × 2 semanas = $8,000
  → Refactor major modules: 2 eng × 4 semanas = $32,000
  → Total principal: ~$76,000

  ROI = Interest anual / Principal = $156K / $76K = 2.05x
  Payback period = Principal / (Interest mensal) = $76K / $13K ≈ 6 meses

  CONCLUSÃO: ROI de 2x e payback de 6 meses → PRIORIZAR

  ═══════════════════════════════════════════════════════════════
  MODELO 2: SCORING QUALITATIVO
  Para decisões rápidas sem cálculo de custo preciso
  ═══════════════════════════════════════════════════════════════

  Para cada item de debt, pontuar (1-5):

  │ Dimensão               │ 1 (baixo)     │ 5 (alto)          │
  │ IMPACTO na velocity    │ Nenhum        │ Bloqueia progresso│
  │ FREQUÊNCIA do impacto  │ Raro          │ Toda sprint       │
  │ BLAST RADIUS           │ 1 componente  │ Toda a plataforma │
  │ RISCO de segurança     │ Nenhum        │ CVE crítica       │
  │ COMPLEXIDADE de fix    │ Trivial (1d)  │ Meses de trabalho │
  │ TENDÊNCIA              │ Estável       │ Piorando rápido   │

  Priority Score = (Impacto × Frequência × Blast × Risco) / Complexidade

  INTERPRETAÇÃO:
  → Score > 50: URGENT — resolver este quarter
  → Score 20-50: HIGH — planejar para próximo quarter
  → Score 5-20: MEDIUM — incluir em roadmap
  → Score < 5: LOW — monitorar, resolver oportunisticamente

  ═══════════════════════════════════════════════════════════════
  MODELO 3: DORA METRICS COMO PROXY
  Correlacionar DORA metrics com tech debt
  ═══════════════════════════════════════════════════════════════

  │ DORA Metric              │ Healthy     │ Debt Indicator      │
  │ Deployment Frequency     │ Multiple/day│ < 1/week = debt      │
  │ Lead Time for Changes    │ < 1 day     │ > 1 week = debt      │
  │ Mean Time to Recovery    │ < 1 hour    │ > 1 day = debt       │
  │ Change Failure Rate      │ < 5%        │ > 15% = debt         │

  Se DORA metrics estão degradando:
  → É EVIDÊNCIA de tech debt acumulando
  → Use como argumento quantitativo para stakeholders
  → Track DORA antes/depois de pagar debt = proof de valor
```

---

## Métricas Proxy para Tech Debt

```
MÉTRICAS DE CÓDIGO:

  │ Métrica                │ Ferramenta              │ Threshold Danger │
  │ Cyclomatic Complexity  │ SonarQube, CodeClimate  │ > 20 per method  │
  │ Cognitive Complexity   │ SonarQube               │ > 15 per method  │
  │ Code Duplication       │ SonarQube, jscpd        │ > 5% duplicated  │
  │ Code Coverage          │ JaCoCo, Istanbul, cover │ < 60% critical   │
  │ Dependency Freshness   │ Dependabot, Renovate    │ > 2 major behind │
  │ CVE Count              │ Snyk, Trivy, Grype      │ Any high/critical│
  │ Dead Code              │ Coverage + profiling     │ > 10% unreachable│
  │ Lines per File         │ Custom                   │ > 500 lines      │
  │ Coupling (afferent/    │ ArchUnit, jdepend        │ Highly coupled   │ 
  │  efferent)             │                         │ modules           │

  MÉTRICAS DE PROCESSO:

  │ Métrica                  │ Como medir                │ Debt Signal     │
  │ PR review time           │ GitHub/GitLab metrics     │ > 48h average   │
  │ PR size (lines changed)  │ GitHub stats              │ > 500 lines avg │
  │ Build time               │ CI/CD metrics             │ > 30 min        │
  │ Flaky test rate          │ CI failures / total runs  │ > 5%            │
  │ Hotspot frequency        │ Git log analysis          │ 20% files, 80%  │
  │                          │                           │ of changes      │
  │ Time to onboard          │ Survey                    │ > 3 months      │

  CODE HOTSPOT ANALYSIS:

  Correlacionar: frequência de mudança × complexidade

  Complexidade
  alta  │  ┌─────────────────┐  ┌─────────────────┐
        │  │ REFACTOR         │  │ ⚠️ CRITICAL DEBT │
        │  │ (complex, stable)│  │ (complex + high  │
        │  │                  │  │  churn = pain)   │
        │  │ Prioridade: Média│  │ Prioridade: ALTA │
        │  └─────────────────┘  └─────────────────┘
        │  ┌─────────────────┐  ┌─────────────────┐
  baixa │  │ LEAVE ALONE      │  │ SIMPLIFY         │
        │  │ (simple, stable) │  │ (simple but high │
        │  │                  │  │  churn = easy win)│
        │  │ Prioridade: Baixa│  │ Prioridade: Média│
        │  └─────────────────┘  └─────────────────┘
        └─────────────────────────────────────────────
         baixa                                   alta
                    Frequência de mudança (churn)

  FERRAMENTA: Adam Tornhill's CodeScene / code-maat
  → Analisa git history para identificar hotspots
  → Files changed frequently + high complexity = worst debt
```

---

## Tech Debt Register

```
TECH DEBT REGISTER — INVENTÁRIO VIVO:

  Um registro centralizado de toda dívida técnica conhecida.
  
  TEMPLATE DE DEBT REGISTER:

  ┌──────────────────────────────────────────────────────────────────┐
  │ ID      │ DEBT-042                                               │
  │ Título  │ Orders Service: zero testes automatizados              │
  │ Tipo    │ Test Debt                                              │
  │ Quadrante│ Prudente + Deliberada                                  │
  │ Criada   │ 2024-Q2 (decisão de ship rápido)                      │
  │ Owner   │ Orders Team (Tech Lead: João)                          │
  │ Serviço │ orders-service                                          │
  │ Priority │ CRITICAL (score: 64)                                  │
  ├──────────┼────────────────────────────────────────────────────────┤
  │ Contexto │ Serviço de orders foi escrito em 6 weeks para        │
  │          │ launch. Decisão consciente de pular testes para       │
  │          │ atingir deadline. Agora com 50K LoC e 0 tests.       │
  ├──────────┼────────────────────────────────────────────────────────┤
  │ Impacto  │ - 2 incidents/mês (P2) causados por regressões       │
  │ (Interest)│ - Feature velocity 40% mais lenta que serviços com   │
  │          │   bom coverage                                        │
  │          │ - Deploy: manual, 4h, high-stress                     │
  │          │ - Custo estimado: $13K/mês                            │
  ├──────────┼────────────────────────────────────────────────────────┤
  │ Plano de │ - Sprint 1: Critical path tests (checkout flow)       │
  │ Pagamento│ - Sprint 2: Integration tests (DB + messaging)       │
  │ (Principal)│ - Sprint 3: CI/CD pipeline automation               │
  │          │ - Sprint 4: Remaining unit tests                      │
  │          │ - Custo estimado: $76K (3 eng × 6 weeks)             │
  ├──────────┼────────────────────────────────────────────────────────┤
  │ ROI      │ 2.05x (payback em ~6 meses)                          │
  │ ADR      │ ADR-2024-015                                          │
  │ Status   │ 🟡 IN PROGRESS (Sprint 1 completa)                   │
  └──────────┴────────────────────────────────────────────────────────┘

  ONDE MANTER O DEBT REGISTER:

  │ Ferramenta        │ Pros                   │ Cons                 │
  │ Jira (epic/board) │ Já usamos, tracking    │ Pode "sumir" no      │
  │                   │ integrado              │ backlog              │
  │ Spreadsheet       │ Simples, overview      │ Sem tracking, manual │
  │ Wiki (Notion/     │ Visível, detalhado     │ Desatualiza fácil    │
  │  Confluence)      │                        │                      │
  │ GitHub Issues     │ Próximo do código,     │ Se mistura com bugs  │
  │  (com labels)     │ PR linkável            │                      │
  │ Custom dashboard  │ Metrics-driven,        │ Esforço de build     │
  │  (ex: Backstage)  │ automatizado           │                      │

  RECOMENDADO: 
  → Source of truth: GitHub Issues com label "tech-debt" + wiki page
  → Dashboard: Backstage ou custom (agregar, visualizar)
  → Review: mensal com leads + PM
```

---

## Frameworks de Priorização

```
FRAMEWORK 1: WSJF (Weighted Shortest Job First)

  Do SAFe framework, adaptado para tech debt:

  WSJF = Cost of Delay / Job Size

  Cost of Delay = User-Business Value + Time Criticality + Risk Reduction

  │ Componente         │ Pergunta                            │
  │ User-Business Value│ Quanto impacta entrega de features? │
  │ Time Criticality   │ Piora com o tempo? (deadline?)      │
  │ Risk Reduction     │ Reduz risco de incident/security?   │
  │ Job Size           │ Quanto esforço requer?              │

  Pontuar cada (1-10):

  │ Debt Item          │ Value │ Time │ Risk │ CoD │ Size │ WSJF │
  │ Orders: zero tests │  8   │  7   │  6   │ 21  │  5   │ 4.2  │
  │ Java 11 → 21       │  3   │  9   │  8   │ 20  │  8   │ 2.5  │
  │ Replace MongoDB     │  6   │  3   │  4   │ 13  │  13  │ 1.0  │
  │ Fix flaky tests     │  4   │  5   │  2   │ 11  │  2   │ 5.5  │ ← FIRST

  Ordem: Fix flaky tests → Orders tests → Java upgrade → MongoDB

  ═══════════════════════════════════════════════════════════════
  FRAMEWORK 2: RICE SCORING (Reach, Impact, Confidence, Effort)
  ═══════════════════════════════════════════════════════════════

  RICE = (Reach × Impact × Confidence) / Effort

  │ Fator       │ Escala    │ Descrição                         │
  │ Reach       │ # de eng  │ Quantos engenheiros são afetados? │
  │ Impact      │ 0.25-3    │ Quanto melhora produtividade?      │
  │ Confidence  │ 0-100%    │ Quão certos estamos do impacto?   │
  │ Effort      │ eng-weeks │ Quanto work para resolver?         │

  │ Debt Item          │ Reach │ Impact│ Conf │ Effort │ RICE  │
  │ Fix CI pipeline    │  30   │  2    │ 90%  │  2w    │ 27.0  │ ← FIRST
  │ Orders: zero tests │  5    │  3    │ 80%  │  6w    │  2.0  │
  │ Java 11 → 21       │  30   │  1    │ 70%  │  8w    │  2.6  │
  │ Fix flaky tests    │  20   │  1    │ 95%  │  1w    │ 19.0  │ ← SECOND

  ═══════════════════════════════════════════════════════════════
  FRAMEWORK 3: QUADRANTE URGÊNCIA × IMPACTO 
  ═══════════════════════════════════════════════════════════════

  IMPACTO ALTO │  PLANEJAR          │  AGORA!              │
               │  (importante, não  │  (importante +       │
               │   urgente)         │   urgente)           │
               │                     │                     │
               │  Ex: migrar DB,    │  Ex: CVE crítica,    │
               │  melhorar arch     │  incident-causing    │
               │                     │  debt               │
  ─────────────┼─────────────────────┼─────────────────────│
  IMPACTO BAIXO│  IGNORAR /         │  QUICK FIX           │
               │  MONITOR           │  (urgente, mas       │
               │                     │  pequeno impacto)   │
               │  Ex: código feio   │                     │
               │  mas funcional     │  Ex: dependency      │
               │                     │  update simples     │
               └─────────────────────┴─────────────────────│
                 URGÊNCIA BAIXA       URGÊNCIA ALTA
```

---

## Comunicação para Stakeholders

```
COMO COMUNICAR TECH DEBT PARA NÃO-TÉCNICOS:

  ═══════════════════════════════════════════════════════════
  REGRA #1: FALE EM TERMOS DE NEGÓCIO, NÃO TÉCNICOS
  ═══════════════════════════════════════════════════════════

  ❌ "Precisamos refatorar o Orders Service porque tem 
     acoplamento temporal, zero testes, e complexidade 
     ciclomática > 40 em 12 métodos"

  ✅ "O serviço de pedidos está nos custando $13K/mês 
     em incidentes e lentidão de entrega. Com investimento
     de $76K (6 semanas, 3 engenheiros), economizamos 
     $156K/ano e reduzimos incidentes de 2/mês para 0."

  ═══════════════════════════════════════════════════════════
  REGRA #2: USE ANALOGIAS QUE NÃO-TÉCNICOS ENTENDEM
  ═══════════════════════════════════════════════════════════

  ANALOGIA 1 — "Manutenção de fábrica":
  "Imagine uma fábrica que nunca para para manutenção. 
   As máquinas vão quebrando, produção fica mais lenta,
   defeitos aumentam. Tech debt é a manutenção adiada."

  ANALOGIA 2 — "Juros de cartão de crédito":
  "Tomamos um 'empréstimo' técnico para entregar rápido.
   O juros é: cada feature nova demora 40% a mais.
   Se pagarmos a dívida agora, voltamos à velocidade normal."

  ANALOGIA 3 — "Reforma de casa":
  "Construímos rápido no início (apartamento). Agora somos 
   uma família grande (empresa growing). Precisamos reformar
   para caber. Se não reformarmos, teremos problemas de 
   encanamento (incidents) e não conseguimos adicionar quartos 
   (features)."

  ═══════════════════════════════════════════════════════════
  REGRA #3: MOSTRE O ROI COM NÚMEROS
  ═══════════════════════════════════════════════════════════

  TEMPLATE DE BUSINESS CASE:

  ┌──────────────────────────────────────────────────────────┐
  │ TECH DEBT: Orders Service Modernization                   │
  │                                                          │
  │ PROBLEMA:                                                 │
  │ → 2 incidents/mês afetando clientes                      │
  │ → Features de pedidos demoram 3x mais que outros times   │
  │ → Deploy manual de 4h → risco alto                       │
  │                                                          │
  │ CUSTO DE NÃO AGIR (por ano):                             │
  │ → Incidents: $60K (resolução + impacto)                  │
  │ → Velocity loss: $57K (tempo extra de desenvolvimento)   │
  │ → Manual deploys: $38K (tempo de engenharia)             │
  │ → Attrition risk: $150K+ (se perder 1 engineer sênior)  │
  │ TOTAL: $305K+/ano                                        │
  │                                                          │
  │ INVESTIMENTO SOLICITADO:                                  │
  │ → 3 engineers × 6 weeks = $76K                           │
  │                                                          │
  │ RETORNO ESPERADO:                                        │
  │ → Redução de incidents: 90% (2/mês → 0.2/mês)          │
  │ → Feature velocity: +40% para Orders team               │
  │ → Deploy time: 4 hours → 15 minutes                     │
  │ → Payback: 3 meses                                       │
  │ → ROI ano 1: 4x                                          │
  │                                                          │
  │ Se não investirmos:                                       │
  │ → Custo cresce ~20% ao ano (debt acumula juros)          │
  │ → Risco de incident P1 (major customer impact)           │
  │ → Team morale decreases → turnover risk                  │
  └──────────────────────────────────────────────────────────┘

  ═══════════════════════════════════════════════════════════
  REGRA #4: PROPONHA OPÇÕES (não ultimatos)
  ═══════════════════════════════════════════════════════════

  OPÇÃO A — FULL INVESTMENT (recommended)
  → 3 eng × 6 weeks
  → Resolve 100% do debt no serviço
  → ROI: 4x no primeiro ano

  OPÇÃO B — PARTIAL INVESTMENT  
  → 2 eng × 3 weeks
  → Resolve testes + CI/CD (70% do benefício)
  → ROI: 3x no primeiro ano

  OPÇÃO C — MINIMUM VIABLE
  → 1 eng × 2 weeks (20% allocation)
  → Resolve CI/CD apenas (30% do benefício)
  → ROI: 2x no primeiro ano

  OPÇÃO D — DO NOTHING
  → Custo: $305K+/ano (crescendo)
  → Risco: incident P1, attrition
```

---

## Estratégias de Redução

```
ESTRATÉGIAS PARA PAGAR TECH DEBT:

  ═══════════════════════════════════════════════════════════
  STRATEGY 1: DEDICATED CAPACITY (% of sprints)
  ═══════════════════════════════════════════════════════════

  Alocar % fixo de capacity para debt:

  │ Nível de debt │ Alocação recomendada │ Descrição         │
  │ Low           │ 10-15%               │ Maintenance mode  │
  │ Medium        │ 20-25%               │ Active reduction  │
  │ High          │ 30-40%               │ Urgent recovery   │
  │ Critical      │ 50-80%               │ Full stop mode    │

  COMO IMPLEMENTAR:
  → "Em cada sprint de 10 days, 2 days são para tech debt"
  → Tech Lead escolhe o que atacar (do Debt Register)
  → Track velocity de debt reduction separadamente
  → Reavalia % trimestralmente

  ═══════════════════════════════════════════════════════════
  STRATEGY 2: BOY SCOUT RULE
  "Leave the code better than you found it"
  ═══════════════════════════════════════════════════════════

  → Cada PR que toca código com debt: melhora um pouco
  → Não resolve tudo, mas previne que piore
  → Exemplos: rename variável, extract method, add 1 test
  → Zero overhead de planejamento
  → Funciona para debt SMALL/MEDIUM

  ⚠️ Não funciona para:
  → Architecture debt (precisa de projeto dedicado)
  → Major dependency upgrades (precisa de sprint)
  → Debt que exige coordenação cross-team

  ═══════════════════════════════════════════════════════════
  STRATEGY 3: DEBT SPRINTS (Tech Debt Weeks)
  ═══════════════════════════════════════════════════════════

  → 1 semana dedicada a tech debt por quarter
  → Equipe inteira focada (sem features)
  → Debt items escolhidos previamente (do Register)
  → Demo no final: "O que melhoramos?"
  → Métricas: DORA antes vs depois

  ═══════════════════════════════════════════════════════════
  STRATEGY 4: EMBED IN FEATURE WORK
  ═══════════════════════════════════════════════════════════

  → Quando feature toca área com debt: inclua fix no escopo
  → "Para implementar Feature X, precisamos antes refatorar Y"
  → Debt fix é PRÉ-REQUISITO da feature (não opcional)
  → Funciona quando debt e feature overlap

  ═══════════════════════════════════════════════════════════
  STRATEGY 5: MIGRATION PROJECT (para debt grande)
  ═══════════════════════════════════════════════════════════

  → Para debt tipo: "migrar de Java 11 para 21" ou 
    "quebrar monolito em microservices"
  → Projeto com: scope, timeline, milestones, owner
  → Tracking separado (ver doc 03 - Migration Strategies)
  → Comunicação como projeto (não como "cleanup")
```

---

## Tech Debt Budget

```
TECH DEBT BUDGET — QUANTO INVESTIR:

  ┌────────────────────────────────────────────────────────────┐
  │                                                             │
  │  REGRA GERAL:                                               │
  │                                                             │
  │  ┌───────────────────────────────────────────────────────┐  │
  │  │                     70-80%                            │  │
  │  │              FEATURE DEVELOPMENT                      │  │
  │  │                                                       │  │
  │  ├───────────────────────────────────────────────────────┤  │
  │  │  15-20%  TECH DEBT / PLATFORM INVESTMENT              │  │
  │  ├───────────────────────────────────────────────────────┤  │
  │  │ 5-10% INNOVATION / EXPLORATION                       │  │
  │  └───────────────────────────────────────────────────────┘  │
  │                                                             │
  │  Ajustar por estado atual:                                  │
  │                                                             │
  │  Startup mode:    85% features / 10% debt / 5% innovation  │
  │  Healthy growth:  70% features / 20% debt / 10% innovation │
  │  Debt overload:   50% features / 40% debt / 10% innovation │
  │  Recovery mode:   30% features / 60% debt / 10% innovation │
  │                                                             │
  └────────────────────────────────────────────────────────────┘

  COMO SABER EM QUAL ESTADO VOCÊ ESTÁ:

  │ Indicador               │ Healthy    │ Overload     │ Recovery    │
  │ Deployment frequency    │ Daily+     │ Weekly       │ Monthly     │
  │ Lead time for changes   │ < 1 day    │ 1-2 weeks    │ > 1 month  │
  │ Change failure rate     │ < 5%       │ 15-30%       │ > 30%      │
  │ MTTR                    │ < 1 hour   │ 1-24 hours   │ > 24 hours │
  │ Developer satisfaction  │ > 7/10     │ 5-6/10       │ < 5/10     │
  │ Time on new features    │ > 60%      │ 40-60%       │ < 40%      │
  │ Incidents per sprint    │ < 1        │ 2-5          │ > 5        │
```

---

## Anti-Patterns

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **"We'll fix it later"** | Later nunca chega; debt acumula | Debt Register + sprint allocation |
| **Invisible debt** | Liderança não sabe que existe | Debt Register + business case mensal |
| **Perfeccionismo** | "Tech debt" = tudo que não é perfeito | Debt é o que IMPEDE progresso (não código feio porém funcional) |
| **All-or-nothing** | "Precisamos de 3 meses full stop" | Incremental: 20% capacity ongoing > 100% 1 vez |
| **No tracking** | Debt existe mas ninguém sabe quanto | Debt Register + DORA correlation |
| **Feature vs Debt war** | PM vs Eng em conflito constante | Budget agreement: 80/20 rule |
| **Gold plating** | Usar "debt" para fazer over-engineering | Debt fix with measurable before/after metrics |
| **Blame game** | "Quem criou isso?" | Debt é sistêmico, não individual. Focus on fix. |
| **Debt normalization** | "É assim que é" — aceitar como normal | Benchmark DORA metrics vs industry |
| **No celebration** | Pagar debt sem reconhecimento | Demo debt fixes, celebrate improvements |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código, identifique potencial tech debt:

1. **Código duplicado significativo** — Duplicação > 20 linhas que deveria ser extraída para função/módulo compartilhado
2. **Complexidade ciclomática alta** — Métodos com complexidade > 15 que dificultam testes e manutenção
3. **Dead code** — Código comentado, métodos não chamados, imports não usados — remover, não manter "caso precise"
4. **Missing error handling** — Happy path sem tratamento de falhas, especialmente em I/O, rede, parse
5. **Hardcoded values** — Strings mágicas, números mágicos, URLs hardcoded que deveriam ser config/constants
6. **Missing tests para código modificado** — Se o PR modifica lógica sem adicionar/atualizar testes → sinalize
7. **Dependency desatualizada** — Library com 2+ major versions atrás ou CVE conhecida
8. **Schema migration destrutiva sem expand/contract** — DROP/ALTER irreversível sem fase de transição
9. **God class / God function** — Arquivo > 500 linhas ou função > 50 linhas: sinal de que precisa decomposição
10. **Missing abstraction** — Uso direto de vendor SDK em business logic sem adapter/interface
11. **Debt criada sem documentação** — Se PR cria atalho consciente, exigir comentário TODO + issue no debt register
12. **Flaky test adicionado** — Test que depende de timing, external service, ou ordem de execução

---

## Referências

- **Technical Debt — Ward Cunningham** (1992) — http://wiki.c2.com/?WardExplainsDebtMetaphor
- **Managing Technical Debt** — Philippe Kruchten et al. (SEI/CMU, 2019)
- **Software Design X-Rays** — Adam Tornhill (Pragmatic Bookshelf, 2018)
- **Your Code as a Crime Scene** — Adam Tornhill (Pragmatic Bookshelf, 2015)
- **Accelerate** — Nicole Forsgren et al. (IT Revolution, 2018)
- **Refactoring** — Martin Fowler (Addison-Wesley, 2018)
- **A Philosophy of Software Design** — John Ousterhout (Yaknyam Press, 2018)
- **An Elegant Puzzle** — Will Larson (Stripe Press, 2019)
- **The Staff Engineer's Path** — Tanya Reilly (O'Reilly, 2022)
- **SonarQube Documentation** — https://docs.sonarqube.org/
- **DORA Metrics** — https://dora.dev/
- **CodeScene** — Adam Tornhill — https://codescene.com/
- **Weighted Shortest Job First (WSJF)** — SAFe Framework — https://scaledagileframework.com/wsjf/
