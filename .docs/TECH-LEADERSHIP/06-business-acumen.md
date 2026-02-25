# Business Acumen — Conectando Tecnologia a Resultados de Negócio

> **Objetivo deste documento:** Servir como referência completa sobre **business acumen para líderes técnicos** — como conectar decisões de engenharia a impacto de negócio, falar a linguagem do produto, e participar de decisões estratégicas — otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: ROI thinking, P&L awareness, product planning, cost engineering, métricas de negócio, tradução tech↔business.

---

## Quick Reference — Cheat Sheet

| Conceito | Definição | Quando usar |
|----------|----------|-------------|
| **ROI** | Return on Investment: (Ganho - Custo) / Custo | Justificar projetos técnicos |
| **TCO** | Total Cost of Ownership: custo completo ao longo do tempo | Comparar soluções |
| **P&L** | Profit & Loss: receita - custos = lucro | Entender impacto financial |
| **OKR** | Objective + Key Results | Alinhar tech com business goals |
| **North Star Metric** | Métrica principal que indica sucesso do produto | Priorização |
| **Opportunity Cost** | O que deixa de ganhar ao escolher opção A | Trade-offs de investimento |

---

## Sumário

- [Business Acumen — Conectando Tecnologia a Resultados de Negócio](#business-acumen--conectando-tecnologia-a-resultados-de-negócio)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Por Que Engineers Precisam de Business Acumen](#por-que-engineers-precisam-de-business-acumen)
  - [Entendendo o Business Model](#entendendo-o-business-model)
  - [P&L para Engineers](#pl-para-engineers)
  - [ROI de Decisões Técnicas](#roi-de-decisões-técnicas)
  - [Cost Engineering](#cost-engineering)
  - [Participando de Product Planning](#participando-de-product-planning)
  - [Tradução Tech para Business](#tradução-tech-para-business)
  - [Papel de Cada Nível em Business Acumen](#papel-de-cada-nível-em-business-acumen)
  - [Anti-Patterns](#anti-patterns)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Por Que Engineers Precisam de Business Acumen

```
O GAP ENTRE TECH E BUSINESS:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  ENGENHEIRO SEM BUSINESS ACUMEN:                       │
  │  "Vamos reescrever o serviço em Rust porque é mais     │
  │   performante."                                        │
  │                                                         │
  │  PERGUNTAS QUE NÃO SABE RESPONDER:                    │
  │  → "Quanto custa a performance ruim hoje? ($)"         │
  │  → "Quantos clientes estamos perdendo por isso?"       │
  │  → "Quanto custa reescrever? (time × opportunity cost)"│
  │  → "Existe alternativa mais barata que resolve 80%?"   │
  │                                                         │
  │  ─────────────────────────────────────────────────────  │
  │                                                         │
  │  ENGENHEIRO COM BUSINESS ACUMEN:                       │
  │  "P95 latency > 500ms está causando 12% de abandono   │
  │   no checkout (≈ R$2.3M/mês). Proposta: otimizar hot  │
  │   paths em 4 semanas (1 engineer) vs rewrite completo  │
  │   em 6 meses (3 engineers). Opção 1 resolve 80% do    │
  │   problema com ROI de 180x."                           │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  POR QUE ISSO IMPORTA POR NÍVEL:

  │ Nível     │ Business acumen necessário                  │
  │ Senior    │ Entende impacto do que constrói no produto  │
  │ Tech Lead │ Prioriza backlog técnico por business value │
  │ Staff     │ Propõe investimentos com ROI quantificado   │
  │ Principal │ Participa de decisões de product strategy   │
  │ Tech Mgr  │ Budget planning, headcount justification    │

  A EQUAÇÃO DO IMPACTO:

  ┌─────────────────────────────────────────────────────┐
  │                                                     │
  │  IMPACTO = EXCELLENCE TÉCNICA × RELEVÂNCIA BUSINESS │
  │                                                     │
  │  Alta tech, zero business = "projeto legal que      │
  │  ninguém usa"                                       │
  │                                                     │
  │  Baixa tech, alta business = "gambiarra que funciona│
  │  até explodir"                                      │
  │                                                     │
  │  Alta tech × Alta business = STAFF+ territory       │
  │                                                     │
  └─────────────────────────────────────────────────────┘
```

---

## Entendendo o Business Model

```
ANTES DE PROPOR QUALQUER DECISÃO TÉCNICA, ENTENDA:

  ┌─────────────────────────────────────────────────────────┐
  │ BUSINESS MODEL CANVAS — VERSÃO SIMPLIFICADA P/ ENGINEER│
  │                                                         │
  │ 1. COMO A EMPRESA GANHA DINHEIRO?                      │
  │    → Subscription (SaaS)? Transação? Ads? Marketplace?│
  │    → Qual o revenue model? (MRR, ARR, GMV)             │
  │                                                         │
  │ 2. QUEM SÃO OS CLIENTES?                              │
  │    → B2B? B2C? B2B2C?                                  │
  │    → Segmentos (Enterprise, SMB, Consumer)?            │
  │    → Quem PAGA vs quem USA?                            │
  │                                                         │
  │ 3. QUAL O DIFERENCIAL COMPETITIVO?                     │
  │    → Speed? Preço? Experiência? Network effects?       │
  │    → Qual o "moat" (competitive advantage)?            │
  │                                                         │
  │ 4. QUAIS AS MÉTRICAS QUE IMPORTAM?                     │
  │    → North Star Metric (ex: DAU, GMV, MRR)            │
  │    → Métricas de crescimento (CAC, LTV, Churn)        │
  │    → Métricas financeiras (Revenue, Margin, Burn rate) │
  │                                                         │
  │ 5. QUAL A STRATEGY ATUAL?                              │
  │    → Crescer user base? Aumentar margem?               │
  │    → Lançar novo produto? Entrar em novo mercado?      │
  │    → Cortar custos? Melhorar retention?                │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  COMO DESCOBRIR ESSAS INFORMAÇÕES:

  │ Fonte                    │ O que aprender                  │
  │ All-hands / Town halls   │ Strategy, OKRs, highlights      │
  │ Product roadmap          │ Prioridades de negócio          │
  │ 1:1 com PM do seu time   │ Métricas do produto, constraints│
  │ 1:1 com EM               │ Budget, headcount decisions     │
  │ Earnings call (se public)│ Revenue, growth, investor view  │
  │ Customer calls (listen)  │ Pain points reais               │
  │ Finance reports          │ Custos de infra, cloud spend    │
  │ Competitive analysis     │ Onde perdemos e por quê         │

  MÉTRICAS DE NEGÓCIO QUE ENGINEER DEVE CONHECER:

  │ Métrica │ Significado                    │ Como tech impacta    │
  │ MRR/ARR │ Monthly/Annual Recurring Rev   │ Feature → conversion │
  │ Churn   │ % clientes que saem/mês        │ Reliability → retain │
  │ CAC     │ Customer Acquisition Cost      │ Performance → conv.  │
  │ LTV     │ Lifetime Value do cliente      │ Features → stickiness│
  │ NPS     │ Net Promoter Score             │ UX + reliability     │
  │ GMV     │ Gross Merchandise Volume       │ Platform scalability │
  │ COGS    │ Cost of Goods Sold             │ Infra optimization   │
  │ Margin  │ Revenue - COGS / Revenue       │ Cost engineering     │
  │ Burn    │ Cash consumed per month        │ Efficiency of eng    │
  │ DAU/MAU │ Daily/Monthly Active Users     │ Performance + UX     │
```

---

## P&L para Engineers

```
P&L (PROFIT & LOSS) — O QUE TECH LEADERS PRECISAM SABER:

  ┌─────────────────────────────────────────────────────────┐
  │ SIMPLIFIED P&L — EMPRESA DE SOFTWARE                    │
  │                                                         │
  │ RECEITA (Revenue)                                       │
  │ + Subscription                          R$ 10,000,000  │
  │ + Serviços profissionais                R$  1,500,000  │
  │ ──────────────────────────────────────────────────────  │
  │ = RECEITA TOTAL                         R$ 11,500,000  │
  │                                                         │
  │ COGS (Cost of Goods Sold)                               │
  │ - Cloud/Infra (AWS, GCP)                R$  1,200,000  │
  │ - Third-party APIs/licenses             R$    300,000  │
  │ - Customer support (L1)                 R$    400,000  │
  │ ──────────────────────────────────────────────────────  │
  │ = GROSS MARGIN                          R$  9,600,000  │
  │   (Margem bruta: 83%)                                  │
  │                                                         │
  │ OPEX (Operating Expenses)                               │
  │ - Engineering (salários + benefits)     R$  4,500,000  │
  │ - Product & Design                      R$    800,000  │
  │ - Sales & Marketing                     R$  2,000,000  │
  │ - G&A (General & Admin)                 R$    800,000  │
  │ ──────────────────────────────────────────────────────  │
  │ = EBITDA                                R$  1,500,000  │
  │   (Margem EBITDA: 13%)                                 │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  COMO TECH IMPACTA CADA LINHA:

  RECEITA:
  → Features que aumentam conversão (checkout optimization)
  → Reliability que reduz churn (99.9% uptime)
  → Performance que melhora NPS (page load < 1s)
  → New capabilities que abrem mercados (API pública)

  COGS (CUSTO DIRETO):
  → Cloud spend optimization (reserved instances, right-sizing)
  → Architecture efficiency (serverless vs always-on)
  → Reduzir dependências paid (build vs buy decisions)

  OPEX (CUSTO OPERACIONAL):
  ┌────────────────────────────────────────────────────────┐
  │ Engineering é geralmente 30-50% do OPEX em SaaS       │
  │                                                        │
  │ COMO REDUZIR CUSTO SEM CORTAR PEOPLE:                 │
  │ → Automação de toil (CI/CD, testing, deploys)          │
  │ → Developer Experience (reduzir tempo de onboarding)   │
  │ → Platform engineering (self-service infra)            │
  │ → Reduzir tech debt (costs compound over time)         │
  │ → Better estimation (menos overcommit/rework)          │
  └────────────────────────────────────────────────────────┘

  LINGUAGEM P&L PARA PROPOSALS:

  ❌ "Vamos migrar para Kubernetes"
  ✅ "Migrar para K8s reduz cloud spend em 30% (R$360K/ano)
      e reduz deployment time de 2h para 15min, acelerando
      time-to-market de features que geram R$X/mês"

  ❌ "Precisamos contratar mais 3 engineers"
  ✅ "3 engineers a R$X/mês custam R$Y/ano. Eles vão 
      entregar feature Z que tem projected revenue de R$W/ano.
      ROI: Zx em 18 meses. Alternativa: não contratar e
      feature fica para Q3 com opportunity cost de R$V."
```

---

## ROI de Decisões Técnicas

```
FRAMEWORK DE ROI PARA DECISÕES TÉCNICAS:

  FÓRMULA BÁSICA:

  ROI = (Ganho - Custo) / Custo × 100%

  FÓRMULA PARA ENGINEERING:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │ CUSTO DO PROJETO:                                       │
  │ = (# engineers × salário mensal × meses) +             │
  │   (infra costs) +                                      │
  │   (opportunity cost: o que esses engineers NÃO fariam) │
  │                                                         │
  │ GANHO DO PROJETO:                                       │
  │ = (receita adicional gerada) +                         │
  │   (custo evitado: incidents, cloud, etc.) +            │
  │   (tempo economizado × valor do tempo)                 │
  │                                                         │
  │ PAYBACK PERIOD:                                         │
  │ = Custo total / Ganho mensal                           │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  EXEMPLO CONCRETO — PROPOSTA DE OTIMIZAÇÃO:

  ┌────────────────────────────────────────────────────────┐
  │ PROPOSAL: Otimização de hot paths no checkout          │
  │                                                        │
  │ PROBLEMA:                                              │
  │ P95 latency = 800ms (target: 200ms)                   │
  │ Abandonment rate checkout: 15% (benchmark: 8%)        │
  │ Delta abandonment (latency-related): ~5%               │
  │ Monthly transactions: 100,000                          │
  │ Ticket médio: R$150                                    │
  │ Revenue lost/mês: 5% × 100K × R$150 = R$750,000      │
  │                                                        │
  │ CUSTO:                                                 │
  │ 2 engineers × 4 weeks = 2 person-months               │
  │ Cost: ~R$60,000 (salary + benefits + infra)            │
  │                                                        │
  │ GANHO ESPERADO:                                        │
  │ Recover 60-80% of latency-related abandonment          │
  │ Conservative: R$450,000/mês em revenue adicional      │
  │                                                        │
  │ ROI:                                                   │
  │ = (R$450K - R$60K) / R$60K = 650%                    │
  │ Payback: < 1 semana                                    │
  │                                                        │
  │ ALTERNATIVAS:                                          │
  │ A) Full rewrite (6 meses, 3 eng): custo R$540K,       │
  │    resolve 95% — payback 6 semanas                     │
  │ B) Quick optimization (acima): custo R$60K,            │
  │    resolve 80% — payback < 1 semana                    │
  │ C) Do nothing: -R$750K/mês (growing with traffic)     │
  │                                                        │
  │ RECOMENDAÇÃO: Opção B agora + avaliar opção A em Q3   │
  └────────────────────────────────────────────────────────┘

  ROI DE INVESTIMENTOS TÉCNICOS COMUNS:

  │ Investimento          │ Custo típico  │ Como calcular ganho        │
  │ Migração de DB        │ 3-6 eng-months│ Redução cloud cost +       │
  │                       │               │ performance improvement     │
  │ CI/CD improvement     │ 1-2 eng-months│ Tempo deploy × deploys/dia │
  │                       │               │ × valor do tempo            │
  │ Observability stack   │ 2-3 eng-months│ MTTR reduction × custo de  │
  │                       │               │ downtime/hora              │
  │ Tech debt cleanup     │ Ongoing       │ Development velocity gain  │
  │                       │               │ (cycle time reduction)     │
  │ Security hardening    │ 1-2 eng-months│ Risk reduction (expected   │
  │                       │               │ cost of breach)            │
  │ Platform/self-service │ 6+ eng-months │ Hours saved × teams ×      │
  │                       │               │ cost per hour              │
```

---

## Cost Engineering

```
COST ENGINEERING — OTIMIZANDO CUSTOS DE INFRAESTRUTURA:

  ┌─────────────────────────────────────────────────────────┐
  │ UNIT ECONOMICS EM TECH:                                 │
  │                                                         │
  │ Cost per transaction = Infra cost / # transactions     │
  │ Cost per user = Infrastructure / MAU                    │
  │ Gross margin = (Revenue - COGS) / Revenue              │
  │                                                         │
  │ REGRA: Se cost per unit sobe com crescimento,          │
  │ o negócio não escala. Tech PRECISA resolver isso.      │
  └─────────────────────────────────────────────────────────┘

  CLOUD COST OPTIMIZATION FRAMEWORK:

  ══════════════════════════════════════════════════════════
  TIER 1: QUICK WINS (1-2 semanas, 20-30% saving)
  ══════════════════════════════════════════════════════════
  → Right-sizing: instances oversized para workload atual
  → Reserved Instances / Savings Plans: 1-3 year commit
  → Cleanup: recursos orphaned (EBS, snapshots, IPs)
  → Scheduling: dev/staging off durante noite/weekend
  → Storage tiering: S3 lifecycle policies

  ══════════════════════════════════════════════════════════
  TIER 2: ARCHITECTURAL (1-3 meses, 30-50% saving)
  ══════════════════════════════════════════════════════════
  → Serverless para workloads intermitentes
  → Spot instances para batch processing
  → Cache strategy (Redis reduz DB queries/costs)
  → CDN para static content (reduz origin traffic)
  → Database optimization (query tuning, connection pool)

  ══════════════════════════════════════════════════════════
  TIER 3: STRATEGIC (3-6 meses, 50%+ saving)
  ══════════════════════════════════════════════════════════
  → Multi-cloud strategy para leverage pricing
  → Data architecture (cold storage, archival)
  → Build vs Buy re-evaluation
  → Self-hosted vs managed services tradeoff
  → Regional deployment optimization

  COST AWARENESS DASHBOARD:

  │ Métrica              │ Target    │ Alert se  │ Owner           │
  │ Total cloud spend/mês│ < R$120K │ > R$130K  │ Staff + EM      │
  │ Cost per transaction │ < R$0.02 │ > R$0.03  │ Tech Lead       │
  │ Cost per MAU         │ < R$0.50 │ > R$0.75  │ Principal       │
  │ Reserved utilization │ > 85%    │ < 70%     │ Platform team   │
  │ Waste (orphaned)     │ < 5%     │ > 10%     │ SRE/Platform    │

  FINOPS MINDSET:

  ┌────────────────────────────────────────────────────────┐
  │ PRINCÍPIOS:                                            │
  │                                                        │
  │ 1. COST É FEATURE                                     │
  │    → Todo design doc deve ter seção de custos          │
  │    → Todo service deve ter cost tags                    │
  │    → Cost é parte do definition of done                │
  │                                                        │
  │ 2. TEAMS OWN THEIR COSTS                              │
  │    → Cada time vê seus custos (cost allocation)       │
  │    → Cost review mensal (como sprint review, mas $)   │
  │    → Incentive: savings = budget para outras coisas   │
  │                                                        │
  │ 3. DECISIONS HAVE DOLLAR SIGNS                        │
  │    → "Vamos adicionar cache" = "vai custar R$X/mês    │
  │       mas salvar R$Y/mês em DB queries"               │
  │    → Trade-offs sempre incluem custos                  │
  │                                                        │
  │ 4. OBSERVE, NOT JUST ESTIMATE                         │
  │    → Estimativa antes de implementar                   │
  │    → Medição depois (estimate vs actual)              │
  │    → Ajustar modelos de estimativa com dados reais    │
  └────────────────────────────────────────────────────────┘
```

---

## Participando de Product Planning

```
COMO TECH LEADERS PARTICIPAM DE PRODUCT PLANNING:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  FALHA COMUM: Tech é RECEBEDOR do roadmap              │
  │  PM: "Construam X até data Y"                          │
  │  Eng: "Tá... vamos tentar"                             │
  │                                                         │
  │  IDEAL: Tech é CO-CRIADOR do roadmap                    │
  │  PM: "Precisamos resolver problema X"                   │
  │  Eng: "Podemos resolver com A, B, ou C. A é mais       │
  │        rápido mas não escala. B leva 2x mais mas       │
  │        habilita Y e Z no futuro. Recomendo B."         │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  ONDE ENGINEERS ADICIONAM VALOR NO PLANNING:

  │ Fase             │ Contribuição de tech                  │
  │ Discovery        │ "Isso é possível? Quanto custa?"     │
  │ Prioritization   │ "Feature A habilita B e C depois"    │
  │ Estimation       │ "Complexidade real vs percebida"     │
  │ Trade-offs       │ "Podemos fazer 80% em metade do tempo"│
  │ Sequencing       │ "Se fizermos X primeiro, Y fica fácil"│
  │ Risk assessment  │ "Maior risco técnico é Z"            │
  │ Tech enablers    │ "Investir em platform agora = 3x     │
  │                  │  faster features em Q3"               │

  FRAMEWORK: RICE SCORE COM INPUT TÉCNICO

  RICE = (Reach × Impact × Confidence) / Effort

  │ Fator      │ PM define          │ Eng contribui            │
  │ Reach      │ Quantos users?     │ Tech constraints         │
  │ Impact     │ Quanto muda behavior│ Performance/reliability  │
  │ Confidence │ Quão certos estamos │ Technical feasibility    │
  │ Effort     │ ─                  │ Eng effort (person-weeks)│

  COMO PROPOR INVESTIMENTOS TÉCNICOS NO ROADMAP:

  O PITCH PERFEITO PARA PM/LEADERSHIP:
  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │ 1. PAIN (em linguagem de negócio):                     │
  │    "Deploy leva 2h, e fazemos 3/semana.                │
  │    Cada deploy bloqueia 2 engineers.                   │
  │    = 12 engineer-hours/week perdidas."                 │
  │                                                        │
  │ 2. IMPACT (quantificado):                              │
  │    "Automatizar reduz para 15 min.                     │
  │    Recupera ~10 engineer-hours/week.                   │
  │    = R$X/mês em produtividade."                        │
  │                                                        │
  │ 3. COST (honesto):                                     │
  │    "2 engineers × 3 weeks = 6 person-weeks.            │
  │    Cost: R$Y."                                         │
  │                                                        │
  │ 4. PAYBACK (claro):                                    │
  │    "Payback em 6 semanas.                              │
  │    Net gain em 12 meses: R$Z."                         │
  │                                                        │
  │ 5. RISK DE NÃO FAZER:                                  │
  │    "A cada quarter, o custo aumenta porque teremos     │
  │    mais services para deployar."                       │
  │                                                        │
  └────────────────────────────────────────────────────────┘

  TECH ENABLERS — O CONCEITO QUE PMs ENTENDEM:

  "Investimento técnico agora que HABILITA
   features do negócio depois."

  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │  SEM ENABLER:                                          │
  │  Feature A: 8 weeks                                    │
  │  Feature B: 8 weeks                                    │
  │  Feature C: 8 weeks                                    │
  │  TOTAL: 24 weeks                                       │
  │                                                        │
  │  COM ENABLER (4 weeks de platform work):               │
  │  Enabler: 4 weeks                                      │
  │  Feature A: 3 weeks                                    │
  │  Feature B: 3 weeks                                    │
  │  Feature C: 3 weeks                                    │
  │  TOTAL: 13 weeks                                       │
  │                                                        │
  │  SAVING: 11 weeks (46% faster)                         │
  │  BREAK-EVEN: after Feature B                           │
  │                                                        │
  └────────────────────────────────────────────────────────┘
```

---

## Tradução Tech para Business

```
DICIONÁRIO DE TRADUÇÃO TECH ↔ BUSINESS:

  │ Tech diz               │ Business ouve            │ Tradução correta         │
  │ "Precisamos refatorar" │ "Gastar tempo sem feature"│ "Reduzir custo de        │
  │                        │                           │  desenvolvimento futuro" │
  │ "Tech debt"            │ "Vocês fizeram errado?"  │ "Manutenção necessária   │
  │                        │                           │  para velocidade"        │
  │ "Migrar para K8s"      │ "Tech por tech"          │ "Reduzir infra cost 30%  │
  │                        │                           │  + deploy 10x mais rápido"│
  │ "Precisamos de mais    │ "Gastar mais $"          │ "Para atingir meta X,    │
  │  engenheiros"          │                           │  precisamos de Y capacity"│
  │ "O sistema é legacy"   │ "Está quebrado?"         │ "Custo de manutenção     │
  │                        │                           │  cresce 20%/quarter"     │
  │ "Precisamos de testes" │ "Já não funciona?"       │ "Cada bug em prod custa  │
  │                        │                           │  R$X para corrigir"      │
  │ "Latency alta"         │ "Tá lento?"              │ "Perdemos R$X/mês em     │
  │                        │                           │  conversão por lentidão" │
  │ "SLA de 99.9%"         │ "Parece bom?"            │ "8.7 horas de downtime   │
  │                        │                           │  permitidas/ano"         │

  PRINCÍPIOS DE COMUNICAÇÃO TECH → BUSINESS:

  1. LIDERE COM IMPACTO, NÃO COM SOLUÇÃO
     ❌ "Vamos implementar circuit breaker pattern"
     ✅ "Vamos evitar que falha em pagamentos derrube
         todo o checkout (R$X/minuto de downtime)"

  2. USE DINHEIRO COMO UNIDADE
     ❌ "P95 melhorou de 800ms para 200ms"
     ✅ "Otimização reduziu abandonment em 3% = 
         R$225K/mês de revenue adicional"

  3. COMPARE COM ALTERNATIVAS
     ❌ "Precisamos de 3 meses para o projeto"
     ✅ "Opção A: 3 meses, resolve 100%. Opção B: 1 mês,
         resolve 80%. Opção C: não fazer, custa R$X/mês"

  4. FALE DE RISCO EM TERMOS DE PROBABILIDADE + IMPACTO
     ❌ "O sistema pode cair"
     ✅ "Probabilidade de 30% de outage na Black Friday.
         Impacto estimado: R$X em 4 horas de downtime."

  5. CONECTE A ESTRATÉGIA
     ❌ "Vamos adotar event-driven architecture"
     ✅ "Event-driven architecture habilita real-time 
         inventory sync que é pré-requisito para o
         produto marketplace do Q4"
```

---

## Papel de Cada Nível em Business Acumen

```
COMO CADA NÍVEL APLICA BUSINESS ACUMEN:

  ══════════════════════════════════════════════════════════
  TECH LEAD — "Business Awareness no Time"
  ══════════════════════════════════════════════════════════

  → Entende o produto que o time constrói
  → Participa de sprint planning com contexto de negócio
  → Questiona: "Por que essa feature importa para o user?"
  → Prioriza tech debt por business impact (não apenas
    "incomoda")
  → Métricas: conhece as métricas do produto do time
  
  ENTREGÁVEL: Backlog priorizado com justificativas de 
  business value, não apenas técnicas

  ══════════════════════════════════════════════════════════
  STAFF ENGINEER — "ROI Quantifier"
  ══════════════════════════════════════════════════════════

  → Propõe investimentos técnicos com ROI calculado
  → Participa de product planning como tech advisor
  → Traduz tech debt para linguagem de custo/risco
  → Compara: build vs buy com TCO de 2-3 anos
  → Propõe tech enablers que aceleram roadmap
  → Cost reviews: "nosso custo por transação está subindo,
    aqui está o porquê e como resolver"
  
  ENTREGÁVEL: Proposals com ROI, cost analysis, 
  risk quantification

  ══════════════════════════════════════════════════════════
  PRINCIPAL ENGINEER — "Tech Strategy ↔ Business Strategy"
  ══════════════════════════════════════════════════════════

  → Participa de strategy discussions com C-level
  → Conecta technical vision ao business model
  → Influencia product strategy com tech capabilities:
    "Se investirmos em X, habilitamos modelo de negócio Y"
  → P&L awareness: sabe quanto eng custa e quanto gera
  → Market awareness: entende competitive landscape
    e como tech é diferenciador
  → Propõe Build vs Buy decisions com visão de 3-5 anos
  
  ENTREGÁVEL: Tech strategy alinhada com business strategy,
  proposals que mudam product direction

  ══════════════════════════════════════════════════════════
  TECH MANAGER — "Budget Owner + Business Translator"
  ══════════════════════════════════════════════════════════

  → Budget planning: headcount + infra + tools
  → Justifica investimentos para VP Eng / CFO
  → ROI tracking: mede se investimento gerou resultado
  → Alinha eng capacity com business priorities
  → Negocia scope com product: "Temos X capacity, 
    podemos fazer A+B ou C, qual?" 
  → Hiring: "3 engineers geram R$X de impacto via 
    feature Y; sem eles, lost revenue = R$Z"
  
  ENTREGÁVEL: Budget proposals, capacity planning,
  ROI tracking reports
```

---

## Anti-Patterns

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **Tech for tech's sake** | Propor solução sem business justification | Sempre comece com "qual problema de negócio resolve?" |
| **Ignoring costs** | Design doc sem seção de custos | Cost é feature: todo design inclui estimate de custos |
| **Speaking only tech** | Apresentar para leadership em jargão técnico | Traduza: latency → revenue lost, uptime → customer trust |
| **Over-engineering ROI** | Inventar números para justificar projeto | Use ranges honestos (conservative/optimistic) e dados reais |
| **Ignoring opportunity cost** | "Custa só 2 engineers por 3 meses" | O que esses 2 engineers fariam em vez disso? Quanto vale? |
| **Binary thinking** | "Fazemos ou não fazemos" | Sempre 3 opções: do nothing, quick win, full solution |
| **P&L blindness** | Não saber quanto o time custa ou gera | Pergunte ao EM: quanto gastamos? Quanto geramos? |
| **Gold plating** | Solução perfeita quando 80% resolve o negócio | MVP thinking: qual o mínimo que gera 80% do valor? |
| **Sunk cost fallacy** | "Já investimos 6 meses, não podemos parar" | Avalie: partindo de hoje, vale continuar ou pivotar? |
| **Disconnected OKRs** | Tech OKRs sem conexão com business OKRs | Todo tech OKR deve mapear para business OKR |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e proposals, considere o aspecto de business acumen:

1. **Cost-aware architecture** — Se design cria recursos expensive (DB, compute), verificar se há estimativa de custo mensal
2. **Feature flag for gradual rollout** — Business features devem ter rollback strategy para proteger revenue
3. **Metrics instrumentation** — Se feature impacta business metric (conversion, churn), verificar se há instrumentação para medir
4. **API rate limiting** — Se API é consumida por clientes, verificar se há proteção contra uso abusivo (cost exposure)
5. **Cost tags** — Recursos de cloud devem ter tags de cost allocation por team/service/environment
6. **Efficiency check** — Se loop/query pode gerar custo proporcional a input size, avaliar bounds e costs
7. **Build vs Buy** — Se PR implementa algo que existe como serviço managed, questionar trade-off
8. **Data retention** — Se armazena dados indefinidamente, avaliar lifecycle policies (storage cost cresce)
9. **Caching strategy** — Se chamada expensive é feita repetidamente, cache pode reduzir custo e melhorar performance
10. **Observability of costs** — Se novo serviço não tem cost monitoring/alerting, sinalizar como gap

---

## Referências

- **The Staff Engineer's Path** — Tanya Reilly (O'Reilly, 2022) — Capítulo sobre business context
- **An Elegant Puzzle** — Will Larson (Stripe Press, 2019) — Sizing and budgeting
- **High Output Management** — Andy Grove (Vintage, 1995)
- **Measure What Matters** — John Doerr (Portfolio/Penguin, 2018) — OKRs
- **FinOps Foundation** — https://www.finops.org/
- **Cloud Financial Management** — AWS Well-Architected: https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/
- **Marty Cagan: Empowered** — Product teams and engineering collaboration
- **Software Engineering at Google** — Titus Winters et al. (O'Reilly, 2020) — Chapter on measuring engineering productivity
- **DORA Metrics** — https://dora.dev/ — Connecting engineering metrics to business outcomes
- **Gergely Orosz: The Pragmatic Engineer** — https://newsletter.pragmaticengineer.com/ — Business thinking for engineers
