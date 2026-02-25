# Tech Leadership — Fundamentos e Career Ladder

> **Objetivo deste documento:** Servir como referência completa sobre **liderança técnica** — papéis, expectativas, competências por nível e modelo de evolução de carreira — otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: distinção entre os papéis (Tech Lead, Staff Engineer, Principal Engineer, Engineering Manager), career ladder, competências por nível, impacto esperado, modelo de avaliação.

---

## Quick Reference — Cheat Sheet

| Papel | Escopo de Impacto | Foco Principal | Código:Liderança |
|-------|-------------------|----------------|-------------------|
| **Senior Engineer** | 1 time | Ownership técnico do time | 80:20 |
| **Tech Lead** | 1 time | Delivery + people + tech | 50:50 |
| **Staff Engineer** | 2-3 times / domínio | Direção técnica de domínio | 30:70 |
| **Principal Engineer** | Organização | Estratégia técnica org-wide | 15:85 |
| **Engineering Manager** | 1 time | Pessoas + delivery | 10:90 |
| **Tech Manager (Sr. EM)** | 2-4 times / departamento | Org design + tech strategy | 5:95 |

---

## Sumário

- [Tech Leadership — Fundamentos e Career Ladder](#tech-leadership--fundamentos-e-career-ladder)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Os Quatro Papéis](#os-quatro-papéis)
  - [Career Ladder Detalhado](#career-ladder-detalhado)
  - [Competências por Nível](#competências-por-nível)
  - [IC Track vs Manager Track](#ic-track-vs-manager-track)
  - [Transições de Carreira](#transições-de-carreira)
  - [Modelo de Avaliação e Crescimento](#modelo-de-avaliação-e-crescimento)
  - [O que Muda em Cada Nível](#o-que-muda-em-cada-nível)
  - [Anti-Patterns de Liderança Técnica](#anti-patterns-de-liderança-técnica)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Os Quatro Papéis

```
OS 4 PAPÉIS DE LIDERANÇA TÉCNICA:

  ══════════════════════════════════════════════════════════════
  1. TECH LEAD
  ══════════════════════════════════════════════════════════════

  ESCOPO: 1 time (5-8 engenheiros)
  
  RESPONSABILIDADES:
  ┌────────────────────────────────────────────────────────────┐
  │ TECH                        │ PEOPLE                       │
  │ → Design reviews            │ → 1:1s com time              │
  │ → Code review (gatekeeper)  │ → Coaching técnico           │
  │ → Architectural decisions   │ → Pair programming           │
  │ → Tech debt prioritization  │ → Unblocking engineers       │
  │ → Build/buy decisions      │ → Feedback contínuo          │
  ├─────────────────────────────┼──────────────────────────────┤
  │ DELIVERY                    │ COMMUNICATION                │
  │ → Sprint planning técnico   │ → Traduzir requisitos → tech │
  │ → Estimativas               │ → Status para stakeholders   │
  │ → Risk identification       │ → Colaborar com PM/Designer  │
  │ → Quality gates             │ → Escalar bloqueios          │
  └─────────────────────────────┴──────────────────────────────┘

  O QUE DIFERENCIA UM BOM TECH LEAD:
  → Escreve código (30-50% do tempo) mas não em critical path
  → Otimiza para throughput do TIME, não individual
  → Diz "não" para scope creep com argumentos técnicos
  → Toma decisões rápido, documenta via ADRs
  → Constrói confiança: time busca Tech Lead para conselho

  ══════════════════════════════════════════════════════════════
  2. STAFF ENGINEER
  ══════════════════════════════════════════════════════════════

  ESCOPO: 2-3 times / 1 domínio técnico
  
  QUATRO ARCHETYPES (Will Larson):

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  TECH LEAD         Guia um time em entregas complexas   │
  │  ├── High context, deep team embed                      │
  │  └── Similar a Tech Lead mas com scope maior            │
  │                                                         │
  │  ARCHITECT          Define direção técnica cross-team   │
  │  ├── Low context per team, broad org context            │
  │  └── Design reviews, standards, frameworks              │
  │                                                         │
  │  SOLVER             Resolve problemas deep/ambíguos     │
  │  ├── Parachutes into complex problems                   │
  │  └── Moves between teams as needed                      │
  │                                                         │
  │  RIGHT HAND         Extends an exec's bandwidth         │
  │  ├── Operates with VP/CTO authority on topics           │
  │  └── Org-wide initiatives, special projects             │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  O QUE DIFERENCIA UM BOM STAFF:
  → Escreve código estratégico (protótipos, migrations, frameworks)
  → Multiplica: 1 Staff deve fazer 3 Seniors mais eficazes
  → Identifica o problema CERTO para resolver (problem selection)
  → Navega ambiguidade (não precisa de spec detalhada)
  → Influencia sem autoridade formal
  → Documenta decisões que duram (RFCs, ADRs, design docs)

  ══════════════════════════════════════════════════════════════
  3. PRINCIPAL ENGINEER
  ══════════════════════════════════════════════════════════════

  ESCOPO: Organização inteira / múltiplos domínios
  
  RESPONSABILIDADES:
  ┌────────────────────────────────────────────────────────────┐
  │ ESTRATÉGIA                    │ INFLUÊNCIA                  │
  │ → Visão técnica 2-3 anos      │ → Shape engineering culture │
  │ → Tech Radar da organização   │ → Mentor de Staff Engineers │
  │ → Major architectural decisions│ → Align 10+ teams          │
  │ → Build vs buy org-scale      │ → Trusted advisor for VPs   │
  │ → Migration strategies         │ → External representation   │
  ├──────────────────────────────┼─────────────────────────────┤
  │ GOVERNANCE                    │ EXECUÇÃO SELETIVA           │
  │ → Engineering principles      │ → Protótipos de alto risco  │
  │ → Padrões cross-org           │ → Debugging de crise        │
  │ → RFC/ADR review final        │ → Code em áreas críticas    │
  │ → Hiring bar (tech)           │ → Performance optimization  │
  └──────────────────────────────┴─────────────────────────────┘

  O QUE DIFERENCIA UM BOM PRINCIPAL:
  → Pensa em anos, não sprints
  → Traduz necessidade de negócio em estratégia técnica
  → Diz "não" mais do que "sim" — foco é poder
  → 1 Principal deve elevar 5 Staff Engineers
  → Faz o trabalho que ninguém pediu mas todos precisam
  → Consegue simplificar problemas complexos para qualquer audiência
  → Mantém credibilidade técnica mesmo codando pouco

  ══════════════════════════════════════════════════════════════
  4. ENGINEERING MANAGER / TECH MANAGER
  ══════════════════════════════════════════════════════════════

  EM (Engineering Manager):
  ESCOPO: 1 time (5-10 engenheiros diretos)
  
  Tech Manager (Senior EM / Director of Engineering):
  ESCOPO: 2-4 times, managers reportando a você

  ┌────────────────────────────────────────────────────────────┐
  │ EM                             │ TECH MANAGER              │
  │                                │                            │
  │ → Hiring/firing               │ → Hiring/firing managers   │
  │ → 1:1s, career growth         │ → Org design (team topology)│
  │ → Delivery management          │ → Cross-team processes     │
  │ → Sprint ceremonies            │ → Budget and headcount     │
  │ → Performance reviews          │ → Tech strategy com CTO    │
  │ → Team health                  │ → Engineering culture      │
  │ → Shield from politics         │ → Stakeholder management   │
  │ → Tech decisions (com TL)      │ → Metrics (DORA, NPS, etc)│
  │ → Unblock engineers            │ → Unblock managers         │
  └────────────────────────────────┴────────────────────────────┘

  O QUE DIFERENCIA UM BOM MANAGER:
  → Mede sucesso pelo sucesso do time, não pelo próprio
  → Faz 1:1 de verdade (não status update)
  → Dá feedback difícil com empatia mas sem evitar
  → Protege o time de interrupções desnecessárias
  → Não micro-manage: define outcomes, confia no como
  → Sabe quando intervir técnicamente e quando delegar
```

---

## Career Ladder Detalhado

```
CAREER LADDER — IC TRACK + MANAGER TRACK:

  IC TRACK                                   MANAGER TRACK
  ═════════                                  ═════════════

  Junior Engineer                            
  │ → Tarefas definidas                      
  │ → Aprende com pair                       
  │ → Code review receivido                  
  │                                          
  ▼                                          
  Mid-Level Engineer                         
  │ → Ownership de features                  
  │ → Estima com precisão razoável           
  │ → Dá code reviews úteis                  
  │                                          
  ▼                                          
  Senior Engineer ──────────────┐            
  │ → Ownership de subsistema   │            
  │ → Mentora juniors/mids      │── FORK ──→ Engineering Manager
  │ → Propõe soluções técnicas  │            │ → 1 time
  │ → Design reviews            │            │ → People management
  ▼                              │            │ → Delivery + hiring
  Tech Lead                      │            ▼
  │ → Ownership técnico do time  │            Senior EM / Tech Manager
  │ → 50% código, 50% liderança│            │ → 2-4 times
  │ → People influence           │            │ → Org design
  ▼                              │            │ → Tech strategy
  Staff Engineer ◄──────────────┘            ▼
  │ → 2-3 times / domínio                   Director of Engineering
  │ → Direção técnica                        │ → Departamento
  │ → Resolve ambiguidade                    │ → Budget decisions
  ▼                                          ▼
  Principal Engineer                         VP of Engineering
  │ → Organização inteira                    │ → Engineering org inteira
  │ → Estratégia 2-3 anos                   │ → Business partner
  ▼                                          ▼
  Distinguished / Fellow                     CTO / SVP Engineering
  │ → Indústria inteira                      │ → Company-level strategy
  │ → State-of-the-art research              │ → Board-level decisions

  NOTA: Os tracks não são one-way. 
  → Manager → IC (volta) é saudável e comum
  → Staff → EM é possível (broadening)
  → Principal → VP é raro mas existe (pendulum)
```

---

## Competências por Nível

```
COMPETENCY MATRIX — 7 DIMENSÕES × 4 PAPÉIS:

  ┌───────────────────┬──────────┬──────────┬──────────┬──────────┐
  │ Competência       │Tech Lead │ Staff    │Principal │Tech Mgr  │
  ├───────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ Technical Vision  │ Team-    │ Domain-  │ Org-wide │ Dept-    │
  │                   │ level    │ level    │ 2-3 yrs  │ level    │
  │                   │ quarter  │ year     │          │ year     │
  ├───────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ Org Influence     │ Own team │ 2-3 teams│ Exec +   │ Multiple │
  │                   │ + PM     │ + mgmt   │ org-wide │ teams +  │
  │                   │          │          │          │ product  │
  ├───────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ Cross-team        │ Collab   │ Lead     │ Align    │ Org      │
  │ Leadership        │ com      │ working  │ 5-10     │ design   │
  │                   │ outros TL│ groups   │ teams    │ + process│
  ├───────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ Mentoring         │ Juniors  │ Seniors  │ Staff    │ EMs +    │
  │                   │ + Mids   │ → Staff  │ → Princ  │ Tech Leads│
  ├───────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ Business Acumen   │ Feature  │ Product  │ P&L +    │ Budget + │
  │                   │ context  │ strategy │ market   │ headcount│
  │                   │          │          │ dynamics │ + ROI    │
  ├───────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ Written Comm      │ ADRs,    │ RFCs,    │ Vision   │ OKRs,    │
  │                   │ tech     │ design   │ docs,    │ strategy │
  │                   │ specs    │ docs     │ Tech Radar│ docs     │
  ├───────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ Failure           │ Team     │ Domain   │ Org-wide │ Process  │
  │ Leadership        │ retros,  │ incident │ post-    │ design,  │
  │                   │ blameless│ review   │ mortem   │ systemic │
  │                   │ within   │ improve  │ culture  │ improve  │
  └───────────────────┴──────────┴──────────┴──────────┴──────────┘

  COMO LER: cada célula mostra o ESCOPO esperado, 
  não se a competência existe ou não.
  
  EXEMPLO: "Mentoring"
  → Tech Lead mentora Juniors e Mids (foco técnico)
  → Staff mentora Seniors e os eleva para Staff (foco em autonomia)
  → Principal mentora Staff e os eleva para Principal (foco em estratégia)
  → Tech Manager mentora EMs e Tech Leads (foco em liderança)
```

---

## IC Track vs Manager Track

```
IC vs MANAGER — O QUE REALMENTE MUDA:

  ┌──────────────────────────────────────────────────────────┐
  │ DIMENSÃO          │ IC TRACK          │ MANAGER TRACK    │
  ├───────────────────┼───────────────────┼──────────────────┤
  │ Sucesso medido por│ Impacto técnico   │ Sucesso do time  │
  │ Output principal  │ Código, docs,     │ Pessoas,         │
  │                   │ arquitetura       │ processos, hiring│
  │ Feedback loop     │ Código funciona   │ People grow      │
  │                   │ (rápido)          │ (lento, meses)   │
  │ Dopamine source   │ Resolver problema │ Ver alguém que   │
  │                   │ técnico complexo  │ você mentorou     │
  │                   │                   │ sendo promovido   │
  │ Pior dia          │ Reunião o dia todo│ Demitir alguém   │
  │ Meeting load      │ 30-50% do tempo   │ 60-80% do tempo  │
  │ Career risk       │ Irrelevância      │ Burnout           │
  │                   │ técnica           │ emocional         │
  │ Escalation tool   │ Documento + dados │ Conversa + empatia│
  │ Autoridade        │ Expertise-based   │ Position-based    │
  │                   │ (earned)          │ (granted, must    │
  │                   │                   │ still be earned)  │
  └───────────────────┴───────────────────┴──────────────────┘

  SINAIS QUE IC TRACK É PARA VOCÊ:
  ✅ Quer resolver problemas técnicos deep
  ✅ Energia vem de prototipar + construir
  ✅ Documentar ideias > reuniões
  ✅ Prefere influenciar por exemplo/expertise
  ✅ 1:1s drenam sua energia se são frequentes

  SINAIS QUE MANAGER TRACK É PARA VOCÊ:
  ✅ Energia vem de ver OUTROS succeeding
  ✅ Gosta de dar feedback (inclusive difícil)
  ✅ Orquestra é mais satisfatório que solo
  ✅ Pensa em processos e sistemas de pessoas
  ✅ Hiring e develop people não parece "distração"

  PENDULUM (Charity Majors):
  → Alternar entre IC e Manager é SAUDÁVEL
  → Staff → EM → Staff: ganho de perspectiva em ambos
  → "Manager com passado técnico deep" = poderoso
  → "IC com passado em management" = empático
```

---

## Transições de Carreira

```
TRANSIÇÕES CRÍTICAS:

  ══════════════════════════════════════════════════════════════
  SENIOR → TECH LEAD (mais comum e mais difícil)
  ══════════════════════════════════════════════════════════════

  O QUE MUDA:
  
  ANTES (Senior):                  DEPOIS (Tech Lead):
  ┌────────────────────┐           ┌────────────────────────┐
  │ "Eu resolvo"       │    →      │ "O time resolve"       │
  │ "Meu código"       │    →      │ "Nosso sistema"        │
  │ "Qualidade do      │    →      │ "Qualidade do          │
  │  meu output"       │           │  output do TIME"       │
  │ "Individ. velocity"│    →      │ "Team throughput"      │
  │ "Tecnicamente      │    →      │ "Melhor decisão para   │
  │  elegante"         │           │  o contexto"           │
  └────────────────────┘           └────────────────────────┘

  ARMADILHAS COMUNS:
  1. "Super-coder" → faz todo trabalho difícil sozinho
     → FIX: delegar e mentorar, mesmo que demore 3x
  2. "Conflict avoider" → não dá feedback duro
     → FIX: pratique SBI (Situation-Behavior-Impact)
  3. "Context hoarder" → informação fica na cabeça do TL
     → FIX: documentation-first, runbooks, ADRs
  4. "Perfect code gatekeeper" → bloqueia PRs por estilo
     → FIX: linters automatizados + foco em correctness/design

  ══════════════════════════════════════════════════════════════
  TECH LEAD → STAFF ENGINEER
  ══════════════════════════════════════════════════════════════

  O QUE MUDA:

  ANTES (Tech Lead):               DEPOIS (Staff):
  ┌────────────────────┐           ┌────────────────────────┐
  │ 1 time              │    →      │ 2-3 times / domínio    │
  │ Decisões do time    │    →      │ Decisões cross-team    │
  │ Delivery focus      │    →      │ Direction focus        │
  │ "O que construir    │    →      │ "O que NÃO construir   │
  │  e como"            │           │  e por quê"            │
  │ Tactical (quarter)  │    →      │ Strategic (year)       │
  │ Assigned problems   │    →      │ Find the right problem │
  └────────────────────┘           └────────────────────────┘

  A "PROMOTION" MAIS DIFÍCIL EM TECH:
  → Staff não é "Senior com mais experiência"
  → Staff resolve problemas que ninguém atribui
  → Staff navega ambiguidade organizacional
  → Staff produz DOCUMENTOS que mudam direções
  → Test: "se essa pessoa sair, que DIREÇÃO se perde?"

  ══════════════════════════════════════════════════════════════
  STAFF → PRINCIPAL
  ══════════════════════════════════════════════════════════════

  O QUE MUDA:

  ANTES (Staff):                   DEPOIS (Principal):
  ┌────────────────────┐           ┌────────────────────────┐
  │ Domínio técnico     │    →      │ Organização inteira    │
  │ 1-year horizon      │    →      │ 2-3 year horizon       │
  │ Lead working groups │    →      │ Shape engineering      │
  │                     │           │ culture                │
  │ "Here's the plan"   │    →      │ "Here's WHY this       │
  │                     │           │  matters"              │
  │ Trusted by engineers│    →      │ Trusted by VPs/CTO     │
  │ Technical depth     │    →      │ Technical breadth +    │
  │                     │           │ business fluency       │
  └────────────────────┘           └────────────────────────┘

  ══════════════════════════════════════════════════════════════
  SENIOR → ENGINEERING MANAGER
  ══════════════════════════════════════════════════════════════

  O QUE MUDA:

  ANTES (Senior):                  DEPOIS (EM):
  ┌────────────────────┐           ┌────────────────────────┐
  │ Código = output     │    →      │ Pessoas = output       │
  │ Feedback = PR review│    →      │ Feedback = 1:1, difícil│
  │ Success = ship      │    →      │ Success = team grows   │
  │ Decisão = técnica   │    →      │ Decisão = prioridade + │
  │                     │           │  politica + técnica    │
  │ Identity = engineer │    →      │ Identity = enabler     │
  └────────────────────┘           └────────────────────────┘

  CONSELHO #1 PARA NOVOS EMs:
  "Sua identidade NÃO é mais 'engenheiro brilhante'.
   Sua identidade é 'pessoa que torna outros brilhantes'."

  ══════════════════════════════════════════════════════════════
  EM → TECH MANAGER (Sr. EM / Director)
  ══════════════════════════════════════════════════════════════

  O QUE MUDA:
  → Manage individual contributors → Manage MANAGERS
  → Team delivery → Cross-team PROCESS
  → 1:1 com engineers → 1:1 com EMs (coaching them to coach)
  → Sprint planning → Org design + team topology
  → Shield from politics → NAVIGATE politics
  → Budget consumer → Budget OWNER
```

---

## Modelo de Avaliação e Crescimento

```
FRAMEWORK DE AVALIAÇÃO — IMPACT × SCOPE × AUTONOMIA:

  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  PARA CADA NÍVEL, AVALIAR 3 DIMENSÕES:                  │
  │                                                          │
  │  1. SCOPE (amplitude do impacto)                        │
  │     1 = task │ 2 = feature │ 3 = team │ 4 = domain │    │
  │     5 = org │ 6 = industry                               │
  │                                                          │
  │  2. AUTONOMY (quanto de direção precisa?)               │
  │     1 = step-by-step │ 2 = goal defined │                │
  │     3 = problem defined │ 4 = problem space defined │    │
  │     5 = find the problem │ 6 = define the strategy       │
  │                                                          │
  │  3. IMPACT (resultado produzido)                        │
  │     1 = completes tasks │ 2 = ships features │            │
  │     3 = elevates team │ 4 = elevates domain │             │
  │     5 = elevates org │ 6 = elevates industry              │
  │                                                          │
  └──────────────────────────────────────────────────────────┘

  EXPECTATIVAS POR NÍVEL:

  │ Nível              │ Scope │ Autonomy │ Impact │ Total │
  │ Junior             │  1-2  │   1-2    │  1-2   │  3-6  │
  │ Mid-Level          │  2-3  │   2-3    │  2-3   │  6-9  │
  │ Senior             │  3    │   3-4    │  3     │  9-10 │
  │ Tech Lead          │  3-4  │   3-4    │  3-4   │ 10-12 │
  │ Staff              │  4-5  │   4-5    │  4-5   │ 12-15 │
  │ Principal          │  5-6  │   5-6    │  5-6   │ 15-18 │

  EXEMPLO CONCRETO — STAFF ENGINEER:

  Scope: 4 (domain-level)
  → "Redesenhou o pipeline de dados para 3 times, 
     eliminando duplicação de 70% nos ETL jobs"

  Autonomy: 5 (find the problem)
  → "Identificou que latency problem era causado por 
     N+1 queries cross-service, ninguém tinha mapeado"

  Impact: 4 (elevates domain)
  → "Após mudança, todos os 3 times do domínio de 
     pagamentos ganharam 40% performance sem esforço"

  COMO USAR PARA CRESCIMENTO:
  1. Avaliar estado atual (onde estou em cada dimensão?)
  2. Identificar gap (onde preciso chegar para promoção?)
  3. Criar plano com projetos que demonstrem o nível acima
  4. Buscar SPONSORS (não só mentors) que confirmem o impacto
```

---

## O que Muda em Cada Nível

```
EVOLUÇÃO DE MENTALIDADE:

  JUNIOR → MID:
  "Eu consigo completar tarefas"          
  → "Eu consigo entregar features end-to-end"

  MID → SENIOR:
  "Eu entrego features bem"               
  → "Eu faço escolhas técnicas que consideram trade-offs"

  SENIOR → TECH LEAD:
  "Eu sou o melhor engenheiro do time"    
  → "Eu faço o TIME melhor"

  TECH LEAD → STAFF:
  "Eu cuido da tech do meu time"          
  → "Eu defino a direção técnica de um domínio"

  STAFF → PRINCIPAL:
  "Eu resolvo problemas técnicos complexos"
  → "Eu defino QUAIS problemas a organização deve resolver"

  EM → TECH MANAGER:
  "Eu faço meu time entregar"             
  → "Eu faço TIMES entregarem, liderando managers"


  COMO O TEMPO É GASTO:

  ┌─────────────┬─────┬─────┬─────┬──────┬────────┬────────┐
  │ Atividade   │ Sr  │ TL  │Staff│Princ │  EM    │Tech Mgr│
  ├─────────────┼─────┼─────┼─────┼──────┼────────┼────────┤
  │ Coding      │ 70% │ 35% │ 20% │ 10%  │  5%    │  0%    │
  │ Code Review │ 15% │ 15% │ 10% │  5%  │  5%    │  0%    │
  │ Design/Arch │  5% │ 15% │ 25% │ 25%  │  5%    │ 10%    │
  │ Writing docs│  5% │ 10% │ 20% │ 25%  │  5%    │ 15%    │
  │ Meetings    │  5% │ 15% │ 15% │ 20%  │ 40%    │ 50%    │
  │ 1:1/Mentor  │  0% │ 10% │ 10% │ 15%  │ 25%    │ 20%    │
  │ Strategy    │  0% │  0% │  5% │ 10%  │  5%    │ 15%    │
  │ Politics/Nav│  0% │  0% │  5% │ 10%  │ 10%    │ 20%    │
  └─────────────┴─────┴─────┴─────┴──────┴────────┴────────┘
  
  Nota: "politics" não é pejorativo aqui.
  Significa: navegar stakeholders, alinhar prioridades, 
  construir alianças para direção técnica.
```

---

## Anti-Patterns de Liderança Técnica

| Anti-pattern | Papel | Problema | Correção |
|-------------|-------|---------|----------|
| **Hero coder** | Tech Lead | Resolve tudo sozinho, time não cresce | Delegar 80%, mentorar durante |
| **Architecture astronaut** | Staff/Principal | Só faz design, nunca valida na prática | Prototipar antes de propor |
| **Ivory tower** | Principal | Decisão top-down sem input dos times | RFC process com feedback genuíno |
| **Conflict avoider** | All | Evita feedback difícil, problemas festejam | SBI framework, radical candor |
| **Title-driven** | All | Quer promoção sem demonstrar impacto no nível acima | Demonstrar impacto FIRST, título follows |
| **Seagull architect** | Staff/Principal | "Voa, faz barulho, suja tudo, vai embora" | Ownership duradouro, follow-through |
| **Micro-manager** | EM/Tech Mgr | Controla como, não o quê; time perde autonomia | Define outcomes, confia no como |
| **Tech-nostalgic manager** | EM/Tech Mgr | Quer continuar codando em vez de gerenciar | Aceitar nova identidade, coach não coder |
| **Promotion gatekeeper** | EM/Tech Mgr | Não promove Staff porque "preciso deles aqui" | Sucesso = people crescem, mesmo que saiam |
| **Lone wolf staff** | Staff | Impacto individual sem multiplicação | Medir: "quantos engenheiros melhoraram por mim?" |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e decisões, considere o nível de liderança técnica:

1. **Ownership scope** — PR de Staff/Principal deveria ter impacto cross-team, não apenas local; avaliar se scope é compatível com o nível
2. **Documentation quality** — Leaders técnicos devem produzir ADRs de qualidade; se decisão arquitetural grande sem ADR, solicitar
3. **Multiplicação** — Se code review de Staff/Principal, verificar se há oportunidade de mentorar o autor em vez de só aprovar/rejeitar
4. **Delegation signal** — Tech Lead fazendo todo código complexo sozinho é anti-pattern; verificar se há pair programming ou delegação
5. **Design review first** — Mudança arquitetural sem design doc/RFC anterior → questionar processo
6. **Cross-team impact** — Se PR afeta outros times, verificar se foi comunicado e alinhado
7. **Decision reversibility** — Decisões Type 1 (irreversíveis) precisam de mais escrutínio e ADR
8. **Mentoring in PR** — Review comments que ensinam (explicam o "porquê") são mais valiosos que os que só apontam erros
9. **Simplification** — Leaders técnicos devem simplificar, não complicar; código over-engineered de Staff é red flag
10. **Vision alignment** — Mudanças devem estar alinhadas com a direção técnica documentada; desvio precisa de justificativa

---

## Referências

- **Staff Engineer** — Will Larson (2021) — https://staffeng.com/
- **The Staff Engineer's Path** — Tanya Reilly (O'Reilly, 2022)
- **An Elegant Puzzle** — Will Larson (Stripe Press, 2019)
- **The Manager's Path** — Camille Fournier (O'Reilly, 2017)
- **Becoming a Technical Leader** — Gerald Weinberg (Dorset House, 1986)
- **Radical Candor** — Kim Scott (St. Martin's Press, 2017)
- **The Making of a Manager** — Julie Zhuo (Portfolio, 2019)
- **High Output Management** — Andy Grove (Vintage, 1995)
- **Team Topologies** — Matthew Skelton & Manuel Pais (IT Revolution, 2019)
- **Accelerate** — Nicole Forsgren et al. (IT Revolution, 2018)
- **The Engineer/Manager Pendulum** — Charity Majors — https://charity.wtf/2017/05/11/the-engineer-manager-pendulum/
- **StaffEng Stories** — https://staffeng.com/stories/
- **LeadDev** — https://leaddev.com/
- **Will Larson's Blog** — https://lethain.com/
