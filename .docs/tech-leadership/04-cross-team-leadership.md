# Cross-team Leadership — Alinhar Times em Direção Comum

> **Objetivo deste documento:** Servir como referência completa sobre **liderança cross-team** — como alinhar 5-10 times em uma direção técnica comum — otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: guilds, working groups, design reviews, Team Topologies, coordination patterns, governance sem burocracia.

---

## Quick Reference — Cheat Sheet

| Conceito | Definição | Quando usar |
|----------|----------|-------------|
| **Guild** | Comunidade de prática cross-team (opt-in) | Compartilhar conhecimento |
| **Working Group** | Grupo temporário para resolver problema específico | Decisão/projeto cross-team |
| **Design Review** | Sessão formal de revisão de arquitetura | Mudança significativa |
| **Team Topologies** | Framework para org design (stream/platform/enabling/complicated-subsystem) | Estruturar times |
| **Interaction Mode** | Como times interagem (collaboration/X-as-a-service/facilitating) | Definir boundaries |
| **Architecture Review Board** | Grupo que governa decisões arquiteturais | Governança org-wide |

---

## Sumário

- [Cross-team Leadership — Alinhar Times em Direção Comum](#cross-team-leadership--alinhar-times-em-direção-comum)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O Desafio da Coordenação Cross-team](#o-desafio-da-coordenação-cross-team)
  - [Team Topologies para Tech Leaders](#team-topologies-para-tech-leaders)
  - [Guilds e Comunidades de Prática](#guilds-e-comunidades-de-prática)
  - [Working Groups](#working-groups)
  - [Design Reviews Cross-team](#design-reviews-cross-team)
  - [Architecture Governance sem Burocracia](#architecture-governance-sem-burocracia)
  - [Coordination Patterns](#coordination-patterns)
  - [Papel de Cada Nível na Coordenação](#papel-de-cada-nível-na-coordenação)
  - [Anti-Patterns](#anti-patterns)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O Desafio da Coordenação Cross-team

```
POR QUE CROSS-TEAM É DIFÍCIL:

  LEI DE CONWAY:
  "Organizations design systems that mirror their 
   communication structure."

  IMPLICAÇÃO:
  → 5 times com comunicação ruim → 5 serviços com 
    integração ruim
  → Alinhar ARQUITETURA requer alinhar COMUNICAÇÃO
  → Mudar arquitetura sem mudar org = fracasso

  O CUSTO DA FALTA DE COORDENAÇÃO:

  ┌─────────────────────────────────────────────────────────┐
  │ SEM COORDENAÇÃO:                                        │
  │                                                         │
  │ Time A: "Vamos usar Protocol Buffers para APIs"         │
  │ Time B: "Vamos usar JSON + REST para tudo"              │
  │ Time C: "Vamos usar GraphQL"                            │
  │ Time D: "Vamos usar gRPC"                               │
  │ Time E: "Vamos misturar tudo"                           │
  │                                                         │
  │ RESULTADO:                                              │
  │ → 5 padrões diferentes de integração                    │
  │ → Zero reuse de libraries/tooling                       │
  │ → Onboarding leva 2x mais tempo                        │
  │ → Cada cross-team feature = negotiations complexas      │
  │ → "Quem mantém o API gateway? 🤷"                      │
  │                                                         │
  │ COM COORDENAÇÃO:                                        │
  │                                                         │
  │ Guild decision: "REST para sync, Kafka para async,      │
  │ gRPC para internal high-performance. Exceções via ADR." │
  │                                                         │
  │ RESULTADO:                                              │
  │ → Shared libraries, tooling, documentation              │
  │ → Onboarding: 1 padrão, apply everywhere                │
  │ → Cross-team features: known integration patterns       │
  │ → Ownership claro via Team Topologies                   │
  └─────────────────────────────────────────────────────────┘

  ESCALA DA COORDENAÇÃO:

  │ # Times │ Coordenação necessária         │ Quem lidera        │
  │ 1       │ Time-internal (sprint rituals)  │ Tech Lead          │
  │ 2-3     │ Domain alignment                │ Staff Engineer     │
  │ 4-10    │ Cross-domain + standards        │ Staff + Principal  │
  │ 10+     │ Governance + org design          │ Principal + CTO   │
```

---

## Team Topologies para Tech Leaders

```
TEAM TOPOLOGIES — FRAMEWORK PARA ORG DESIGN:

  ┌─────────────────────────────────────────────────────────┐
  │ 4 TIPOS FUNDAMENTAIS DE TIMES:                          │
  │                                                         │
  │ 1. STREAM-ALIGNED TEAM (maioria)                       │
  │    → Ownership de um fluxo de valor end-to-end          │
  │    → Ex: "Time de Pagamentos", "Time de Checkout"      │
  │    → Entrega features para o negócio                    │
  │    → Deve ser autônomo: design → build → deploy → run  │
  │                                                         │
  │ 2. PLATFORM TEAM                                        │
  │    → Fornece capabilities self-service para stream teams│
  │    → Ex: "Time de Plataforma" (CI/CD, K8s, observability)│
  │    → Reduz cognitive load dos stream teams              │
  │    → Sucesso: stream teams usam sem pedir ajuda         │
  │                                                         │
  │ 3. ENABLING TEAM                                        │
  │    → Ajuda stream teams a adotar novas capabilities     │
  │    → Ex: "Team de SRE practices", "Security Champions"│
  │    → Temporary embed: entra, ensina, sai                │
  │    → Sucesso: stream team faz sozinho depois            │
  │                                                         │
  │ 4. COMPLICATED-SUBSYSTEM TEAM                          │
  │    → Ownership de componente que requer expertise deep  │
  │    → Ex: "Time de ML/AI Engine", "Time de Video Codec" │
  │    → Reduz cognitive load dos stream teams              │
  │    → Expõe API well-defined para consumption            │
  └─────────────────────────────────────────────────────────┘

  3 MODOS DE INTERAÇÃO:

  ┌───────────────┬────────────────────────────────────────┐
  │ COLLABORATION │ 2 times trabalham juntos temporariamente│
  │               │ Alto bandwidth, high cognitive load     │
  │               │ Bom para: explorar novas soluções       │
  │               │ Duração: weeks, não meses               │
  ├───────────────┼────────────────────────────────────────┤
  │ X-AS-A-SERVICE│ 1 time consome API/platform de outro    │
  │               │ Baixo coupling, clear contract          │
  │               │ Bom para: autonomia de delivery         │
  │               │ Duração: permanently                    │
  ├───────────────┼────────────────────────────────────────┤
  │ FACILITATING  │ 1 time ajuda outro a aprender          │
  │               │ Enabling team pattern                   │
  │               │ Bom para: adoção de novas práticas      │
  │               │ Duração: weeks, then hand off           │
  └───────────────┴────────────────────────────────────────┘

  EXEMPLO DE ORG DESIGN:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  STREAM-ALIGNED:                                        │
  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
  │  │Payments  │ │Checkout  │ │Catalog   │ │Logistics │  │
  │  │Team (6)  │ │Team (5)  │ │Team (6)  │ │Team (7)  │  │
  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘  │
  │       │            │            │            │          │
  │  ─────┴────────────┴────────────┴────────────┴────────  │
  │                        │                                │
  │  PLATFORM:             │                                │
  │  ┌─────────────────────┴───────────────────────┐       │
  │  │ Platform Team (8)                            │       │
  │  │ CI/CD, K8s, Observability, Dev Tools         │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                         │
  │  ENABLING:                                              │
  │  ┌──────────────────────┐                               │
  │  │ SRE Practices (3)    │ ← roda entre stream teams    │
  │  └──────────────────────┘                               │
  │                                                         │
  │  COMPLICATED-SUBSYSTEM:                                 │
  │  ┌──────────────────────┐                               │
  │  │ Search/ML Engine (4) │ ← API para Catalog team      │
  │  └──────────────────────┘                               │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  COMO TECH LEADERS USAM TEAM TOPOLOGIES:

  │ Papel      │ Como usa                                    │
  │ Tech Lead  │ Entender boundaries do seu time, pedir      │
  │            │ APIs claras de platform, flag cognitive load │
  │ Staff      │ Propor team topology changes, otimizar       │
  │            │ interaction modes entre 2-3 times           │
  │ Principal  │ Design team topology org-wide, align com    │
  │            │ CTO e VP Eng, evolve conforme strategy       │
  │ Tech Mgr   │ Implement team changes, hire for gaps,      │
  │            │ measure team health and cognitive load       │
```

---

## Guilds e Comunidades de Prática

```
GUILDS — COMUNIDADES CROSS-TEAM:

  ┌─────────────────────────────────────────────────────────┐
  │ GUILD = grupo opt-in de pessoas com interesse comum     │
  │ CROSS-TEAM: membros de diferentes times                 │
  │ VOLUNTARY: participação não é obrigatória               │
  │ REGULAR: cadência definida (bi-weekly/monthly)          │
  └─────────────────────────────────────────────────────────┘

  EXEMPLOS DE GUILDS:

  │ Guild             │ Foco                        │ Cadência  │
  │ Backend Guild     │ Java patterns, APIs, perf   │ Bi-weekly │
  │ Frontend Guild    │ React, design system, a11y  │ Bi-weekly │
  │ Data Engineering  │ Pipelines, quality, tools   │ Monthly   │
  │ SRE/Reliability   │ Incidents, SLOs, chaos eng  │ Bi-weekly │
  │ Security Guild    │ OWASP, scanning, compliance │ Monthly   │
  │ Architecture Guild│ Cross-cutting arch decisions│ Bi-weekly │

  ESTRUTURA DE UMA GUILD EFICAZ:

  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │  CHAIR (rotativo): facilita sessões                   │
  │  → Staff ou Senior apaixonado pelo tema               │
  │  → Rota a cada quarter para evitar burnout             │
  │                                                        │
  │  SPONSOR: EM ou Principal que garante                  │
  │  → Tempo protegido para membros                       │
  │  → Budget se necessário (ferramentas, livros)         │
  │  → Visibilidade para management                       │
  │                                                        │
  │  FORMATO DA SESSÃO (60 min):                          │
  │  ├── 10 min: Updates e announcements                   │
  │  ├── 30 min: Apresentação/deep dive (rotativo)        │
  │  ├── 15 min: Discussion/debate                         │
  │  └── 5 min: Action items + próxima sessão             │
  │                                                        │
  │  ARTEFATOS:                                            │
  │  → Meeting notes no Confluence/Notion                  │
  │  → Decisões documentadas como ADRs                     │
  │  → Shared guidelines/playbooks                         │
  │  → Recordings para quem não participou                 │
  │                                                        │
  └────────────────────────────────────────────────────────┘

  GUILD ACTIVITIES:

  1. TECH TALK (30 min)
     → Membro apresenta tema para o grupo
     → Ex: "Como implementamos circuit breaker no Payments"
  
  2. RFC REVIEW (30 min)
     → Guild revisa RFC proposto por membro
     → Feedback cross-team antes de decisão formal
  
  3. LIGHTNING TALKS (5 min cada)
     → 5-6 quick demos/tips
     → Rotação: cada time apresenta 1
  
  4. BOOK CLUB (mensal)
     → Capítulo de livro técnico
     → Discussion guiada
  
  5. BROWN BAG (lunchtime)
     → Demo informal de nova tech/tool
     → Hands-on session

  SINAIS DE GUILD SAUDÁVEL:
  ✅ Attendance voluntária mas consistente (60%+ membros)
  ✅ Decisões influenciam práticas dos times
  ✅ Membros contribuem ativamente (não só consomem)
  ✅ ADRs nascem de discussões na guild
  ✅ New joiners onboard via guild materials

  SINAIS DE GUILD DOENTE:
  ❌ Attendance mandatória (forçada)
  ❌ Sempre os mesmos apresentam
  ❌ Discussões não geram ação
  ❌ "Meeting que poderia ser email"
  ❌ Ninguém se voluntaria para chair
```

---

## Working Groups

```
WORKING GROUP — GRUPO TEMPORÁRIO PARA PROBLEMA ESPECÍFICO:

  WORKING GROUP ≠ GUILD:
  
  │ Aspecto   │ Guild              │ Working Group          │
  │ Duração   │ Permanente         │ Temporária (4-12 weeks)│
  │ Scope     │ Broad (área)       │ Narrow (1 problema)    │
  │ Output    │ Knowledge sharing  │ Decision + plan        │
  │ Membership│ Opt-in, open       │ Selected, 5-8 people   │
  │ Cadência  │ Regular            │ Intensive (weekly+)    │
  │ End state │ Continues          │ Dissolves when done    │

  QUANDO CRIAR UM WORKING GROUP:

  ✅ Problema afeta 3+ times
  ✅ Precisa de decisão coordenada
  ✅ Expertise de múltiplas áreas necessária
  ✅ Tem deadline (não é ongoing discussion)
  
  EXEMPLOS:
  → "API Versioning Working Group" — definir padrão para toda org
  → "Monolith Decomposition Working Group" — planejar migration
  → "Observability Standards WG" — definir o que cada serviço deve ter
  → "On-call Improvement WG" — redesenhar processo de on-call

  ESTRUTURA:

  ┌────────────────────────────────────────────────────────┐
  │ WORKING GROUP CHARTER                                  │
  │                                                        │
  │ Name: API Versioning Standards                         │
  │ Sponsor: Ana (Principal Engineer)                      │
  │ Lead: Pedro (Staff Engineer, Platform)                 │
  │                                                        │
  │ Members:                                               │
  │ → Senior from Payments team                            │
  │ → Senior from Catalog team                             │
  │ → Tech Lead from Mobile team (API consumer)            │
  │ → Staff from Platform team                             │
  │ → PM representative (API products)                     │
  │                                                        │
  │ Problem statement:                                     │
  │ "Temos 3 estilos diferentes de API versioning,         │
  │  causando confusão para consumidores internos e        │
  │  externos. Precisamos de padrão único."                │
  │                                                        │
  │ Expected output:                                       │
  │ → ADR com decisão de versioning strategy               │
  │ → Playbook de migração para APIs existentes            │
  │ → Timeline de adoção                                   │
  │                                                        │
  │ Timeline: 6 weeks                                      │
  │ Cadência: Weekly 60min sessions                        │
  │ Success criteria: decisão aceita + migration backlog   │
  └────────────────────────────────────────────────────────┘

  LIFECYCLE:

  Week 1: Charter + problem definition
  Week 2-3: Research + options analysis
  Week 4: Draft proposal + feedback round
  Week 5: Finalize + stakeholder review
  Week 6: Publish decision + hand off execution
  → DISSOLVE working group
  → Execution owned by relevant teams

  QUEM LIDERA WORKING GROUPS:

  │ Scope                  │ Quem lidera            │
  │ 2-3 times, 1 domínio  │ Staff Engineer          │
  │ Cross-domain, org-wide│ Principal ou Staff sênior│
  │ Process/org change     │ Tech Manager + Staff    │
```

---

## Design Reviews Cross-team

```
DESIGN REVIEW — A PRÁTICA MAIS IMPACTFUL:

  QUANDO FAZER DESIGN REVIEW:
  ✅ Novo serviço / componente
  ✅ Mudança arquitetural em serviço existente
  ✅ Integração entre 2+ serviços
  ✅ Mudança que afeta dados (schema, storage)
  ✅ Qualquer item que geraria RFC

  FORMATO:

  ┌────────────────────────────────────────────────────────┐
  │ DESIGN REVIEW SESSION (60 min)                         │
  │                                                        │
  │ PRE-READ (enviado 48h antes):                         │
  │ → Design doc (2-5 páginas)                             │
  │ → Diagrams (system, sequence)                          │
  │ → Open questions (o que o author quer feedback)        │
  │                                                        │
  │ NA SESSÃO:                                             │
  │ ├── 10 min: Author walkthrough (brief, assume leitura)│
  │ ├── 40 min: Questions + discussion                     │
  │ ├── 10 min: Decision + action items                    │
  │                                                        │
  │ PÓS-SESSÃO:                                            │
  │ → Author atualiza design doc com feedback              │
  │ → ADR se decisão significativa                         │
  │ → Go/No-go decision                                    │
  └────────────────────────────────────────────────────────┘

  QUEM PARTICIPA:

  │ Role              │ Responsabilidade no review         │
  │ Author            │ Apresenta, responde perguntas      │
  │ Domain Staff/Princ│ Architectural consistency check     │
  │ Affected team TLs │ Integration concerns, impact        │
  │ Platform team     │ Infra/observability/deploy concerns│
  │ Security champion │ Threat model, compliance gaps      │
  │ Product (optional)│ Business context se complexo        │

  CHECKLIST DE DESIGN REVIEW:

  FUNCIONAL:
  □ Problema claramente definido?
  □ Alternativas avaliadas (mínimo 2)?
  □ Escolha justificada com trade-offs?

  ARQUITETURAL:
  □ Alinhado com vision doc e principles?
  □ APIs bem definidas (contracts)?
  □ Data model consistente?
  □ Dependency analysis feita?
  □ Backward compatibility preservada?

  OPERACIONAL:
  □ Observability (metrics, tracing, logs)?
  □ Alerting e runbooks?
  □ Rollback strategy?
  □ Performance requirements endereçados?
  □ Scale considerations?

  SEGURANÇA:
  □ Authn/authz corretos?
  □ Data classification e encryption?
  □ Input validation?
  □ Audit logging?

  DELIVERY:
  □ Migration plan (se existente → novo)?
  □ Feature flag strategy?
  □ Testing strategy?
  □ Timeline realista?

  COMO EVITAR QUE DESIGN REVIEW VIRE BOTTLENECK:

  1. TIER SYSTEM:
     Tier 1 (team-internal): TL aprova, sem review formal
     Tier 2 (cross-team): Staff review + 1 sessão
     Tier 3 (org-wide): Principal review + full session

  2. ASYNC-FIRST:
     → Maioria resolve via comments no design doc
     → Sessão sííncrona só quando necessário
     → Max 60 min, nunca 2h

  3. DECISION DEADLINE:
     → Proposta enviada → 1 semana para review
     → Se sem feedback: author pode prosseguir
     → "Silence is consent" (com aviso prévio claro)
```

---

## Architecture Governance sem Burocracia

```
GOVERNANCE — STANDARDS SEM SER "IVORY TOWER":

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  GOVERNANCE RUIM:                 GOVERNANCE BOA:       │
  │                                                         │
  │  "Architecture Review Board      "Guidelines + ADR +   │
  │   que bloqueia todo mundo"        automated checks"     │
  │                                                         │
  │  "Aprovação de 3 comitês         "Self-service +       │
  │   para qualquer mudança"          escalation para      │
  │                                   exceções"            │
  │                                                         │
  │  "Documento de 50 páginas        "Principles that      │
  │   que ninguém lê"                 teams reference       │
  │                                   in ADRs"             │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  MODELO DE GOVERNANCE EM 3 TIERS:

  ═══════════════════════════════════
  TIER 1: AUTOMATED (zero friction)
  ═══════════════════════════════════
  → Linters, static analysis, CI checks
  → "Se o CI passa, está OK para este nível"
  
  Exemplos:
  → Code style → formatter automático
  → Dependency versions → Dependabot
  → Security scanning → Snyk/Trivy
  → API schema validation → spectral
  → Test coverage threshold → CI gate

  ═══════════════════════════════════
  TIER 2: PEER REVIEW (low friction)
  ═══════════════════════════════════
  → Code review normal (2 approvals)
  → Design review assíncrona (comments no doc)
  → ADR para decisões significativas
  
  Owner: Tech Lead do time
  Escalation: Staff Engineer se cross-team

  ═══════════════════════════════════
  TIER 3: FORMAL REVIEW (justified)
  ═══════════════════════════════════
  → Apenas para:
    - Nova tecnologia no stack
    - Mudança que afeta 3+ times
    - Investimento > X person-months
    - Decisão irreversível (Type 1)
  
  Owner: Principal ou Staff + Architecture Guild
  Format: RFC + Design Review session
  Timeline: max 2 weeks to decision

  COMO CADA NÍVEL CONTRIBUI:

  │ Papel      │ Tier 1           │ Tier 2          │ Tier 3        │
  │ Tech Lead  │ Configura CI     │ Gatekeeper      │ Participante  │
  │            │ gates do time    │ de code review  │               │
  │ Staff      │ Define standards │ Design reviewer │ RFC author/   │
  │            │ cross-team       │ cross-team      │ reviewer      │
  │ Principal  │ Define org-wide  │ Spot check      │ Final decision│
  │            │ CI requirements  │ de ADR quality  │ maker         │
  │ Tech Mgr   │ Budget for tools │ Process design  │ Sponsor/      │
  │            │                  │                 │ enabler       │
```

---

## Coordination Patterns

```
PATTERNS PARA COORDENAR CROSS-TEAM:

  ══════════════════════════════════════════════════════════════
  PATTERN 1: AMBASSADOR MODEL
  ══════════════════════════════════════════════════════════════

  Cada time tem 1 "ambassador" que participa de fóruns cross-team

  ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ Time A   │   │ Time B   │   │ Time C   │
  │ ┌──┐     │   │ ┌──┐     │   │ ┌──┐     │
  │ │🅰│     │   │ │🅱│     │   │ │🅲│     │
  │ └──┘     │   │ └──┘     │   │ └──┘     │
  └──────────┘   └──────────┘   └──────────┘
       │              │              │
       └──────────────┼──────────────┘
                      │
              ┌───────┴────────┐
              │ Architecture   │
              │ Guild (weekly) │
              └────────────────┘

  PRO: Baixo overhead (1 pessoa por time)
  CON: Ambassadors precisam ser bons comunicadores

  ══════════════════════════════════════════════════════════════
  PATTERN 2: RFC-DRIVEN ALIGNMENT
  ══════════════════════════════════════════════════════════════

  Toda decisão cross-team passa por RFC escrito

  Author (Staff) → Draft RFC → Review (async) → Decision
                      ↓
              Stakeholders comentam
              no documento

  PRO: Escala para times remotos/async
  CON: Pode ser lento se não tiver deadline

  ══════════════════════════════════════════════════════════════
  PATTERN 3: ROTATION/EMBED
  ══════════════════════════════════════════════════════════════

  Staff/Principal rota entre times por sprints

  Sprint 1-2: embedded em Time A
  Sprint 3-4: embedded em Time B
  Sprint 5-6: embedded em Time C

  PRO: Deep understanding de cada time
  CON: Context switching, pode parecer "fiscalização"
  MITIGATION: frame como "helping", não "auditing"

  ══════════════════════════════════════════════════════════════
  PATTERN 4: WEEKLY TECH LEADS SYNC
  ══════════════════════════════════════════════════════════════

  Todos os Tech Leads + Staff/Principal (30 min weekly)

  AGENDA:
  → 5 min: cross-team blockers
  → 10 min: upcoming changes que afetam outros times
  → 10 min: standards/guidelines discussion
  → 5 min: action items

  PRO: Regular cadence, low cost
  CON: Pode virar status meeting (combater com agenda estrita)

  ══════════════════════════════════════════════════════════════
  PATTERN 5: INNER SOURCE
  ══════════════════════════════════════════════════════════════

  Times contribuem para repos de outros times via PRs

  Time A precisa de feature no SDK do Time B
  → Time A cria PR no repo do Time B
  → Time B faz review
  → PR merged

  PRO: Resolve dependency sem esperar na fila
  CON: Requer good documentation + contributor guides
  BEST FOR: shared libraries, SDKs, platform components
```

---

## Papel de Cada Nível na Coordenação

```
COMO CADA NÍVEL LIDERA CROSS-TEAM:

  ══════════════════════════════════════════════════════════════
  TECH LEAD — "O Conector do Time"
  ══════════════════════════════════════════════════════════════

  → Representa time em guilds e Tech Lead sync
  → Comunica: "mudança em nosso serviço afeta Time B"
  → Participa de design reviews de serviços vizinhos
  → Alerta Staff/Principal sobre inconsistências
  → Alinha: "nosso ADR é consistente com padrão org?"
  
  KEY SKILL: comunicação clara + awareness do ecossistema

  ══════════════════════════════════════════════════════════════
  STAFF ENGINEER — "O Conector de Times"
  ══════════════════════════════════════════════════════════════

  → Lidera working groups (charter → decision → dissolve)
  → Facilita design reviews cross-team
  → Propõe e escreve RFCs que afetam 2-3 times
  → Cria shared libraries/tools que beneficiam múltiplos times
  → Identify: "Times A e B estão resolvendo o mesmo problema
    de maneiras diferentes — vamos alinhar"
  → Mentora Tech Leads em seus times
  
  KEY SKILL: navigating ambiguity + building consensus

  ══════════════════════════════════════════════════════════════
  PRINCIPAL ENGINEER — "O Alinhador da Organização"
  ══════════════════════════════════════════════════════════════

  → Design team topology (com VP Eng)
  → Define governance model (3-tier)
  → Chair da Architecture Guild
  → Final decision em RFCs org-wide
  → Identifica padrões: "3 times independentemente
    estão construindo a mesma coisa — precisamos de platform"
  → Evolui standards e principles baseado em learnings
  → Talks/writings que criam cultural alignment
  
  KEY SKILL: systems thinking (org + tech) + influence at scale

  ══════════════════════════════════════════════════════════════
  TECH MANAGER — "O Enabler Organizacional"
  ══════════════════════════════════════════════════════════════

  → Implementa team topology changes (hire, reorg)
  → Protege tempo: "meu time tem capacity para guild X"
  → Budget para tooling/platforms que habilita coordination
  → Processo: design review cadence, RFC process
  → Metrics: measure team interaction health
  → Remove blockers: "Time B precisa de API de Time A,
    vou escalar com EM do Time A"
  
  KEY SKILL: org design + stakeholder management + removing friction
```

---

## Anti-Patterns

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **Committee-driven architecture** | Decisão por consenso de 20 pessoas = nada decide | Owner claro: Staff/Principal decide com input dos times |
| **Ivory tower governance** | "Architecture board" que nunca escreveu código | Governance liderada por quem faz (Staff+), não por título |
| **Over-coordination** | Reunião com 5 times para mudança que afeta 2 | Tier system: escalar governance conforme impacto |
| **Guild as mandatory meeting** | Participação obrigatória mata engajamento | Opt-in com conteúdo valioso → attendance natural |
| **Working group without charter** | Grupo que discute sem nunca decidir | Charter com deadline + expected output antes de começar |
| **Ignoring Conway's Law** | Arquitetura que conflita com org structure | Team Topologies: alinhar times com componentes |
| **One-way design review** | Senior "tells" juniors what's wrong | Discussion: reviewer aprende tanto quanto author |
| **Standards without migration** | "New standard!" mas legacy continua forever | Todo standard precisa de migration plan para existente |
| **Coordination as bottleneck** | Staff/Principal é gargalo de approvals | Empoderar Tech Leads: tier 1-2 não precisa de Staff |
| **Shadow governance** | Decisões tomadas em Slack DM, não documentadas | RFC-driven: decisão cross-team = documento público |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e decisões, considere o aspecto cross-team:

1. **Cross-team API changes** — Se PR altera API consumida por outros times, verificar se há comunicação e versioning
2. **Shared library updates** — Mudança em shared library precisa de compatibility check + changelog
3. **Pattern consistency** — Se time está usando padrão diferente do standard organizacional, questionar (ADR necessário para exceção)
4. **Design doc linkado** — Novo serviço/componente sem design doc = gap no processo de design review
5. **Platform team contracts** — Se PR depende de feature de platform team, verificar se está no roadmap do platform
6. **Team Topology awareness** — Se stream team está construindo infra que deveria ser platform, sinalizar
7. **Inner source opportunity** — Se PR reimplementa algo que existe em outro time, sugerir contribuição ao repo original
8. **Guild decision referenced** — Se decisão foi tomada em guild, o ADR deveria referenciar
9. **Migration compliance** — Se existe standard novo + migration plan, código novo deveria seguir o novo padrão
10. **Dependency documentation** — Se serviço depende de outro time, verificar se dependency está documentada

---

## Referências

- **Team Topologies** — Matthew Skelton & Manuel Pais (IT Revolution, 2019)
- **The Staff Engineer's Path** — Tanya Reilly (O'Reilly, 2022)
- **An Elegant Puzzle** — Will Larson (Stripe Press, 2019)
- **Accelerate** — Nicole Forsgren et al. (IT Revolution, 2018)
- **Project to Product** — Mik Kersten (IT Revolution, 2018)
- **Dynamic Reteaming** — Heidi Helfand (2020)
- **Spotify Engineering Culture** — Henrik Kniberg — https://engineering.atspotify.com/
- **Conway's Law** — Melvin Conway — http://www.melconway.com/Home/Conways_Law.html
- **Inner Source Commons** — https://innersourcecommons.org/
- **RFC Process at Oxide** — https://oxide.computer/blog/rfd-1-requests-for-discussion
- **Google Design Docs** — https://www.industrialempathy.com/posts/design-docs-at-google/
- **Architecture Guild at Scale** — ThoughtWorks — https://www.thoughtworks.com/insights
