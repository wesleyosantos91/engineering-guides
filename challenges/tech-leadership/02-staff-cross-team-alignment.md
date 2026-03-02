# Level 2 — Staff: Alinhamento Cross-Team

> **Objetivo:** Expandir influência além do squad — alinhar decisões técnicas entre times,
> liderar sem autoridade formal, escalar práticas e criar pontes organizacionais.

**Esfera de impacto:** Multi-team (2-5 squads)
**Referência:** Pré-requisito para [03-principal-area-direction.md](03-principal-area-direction.md)

---

## Contexto — TechNova (Série B)

A TechNova levantou Série B e cresceu de 25 para 70 engenheiros. Há 10 squads organizados em 3 áreas (Payments, Marketplace, Platform). Você foi promovido a **Staff Engineer** e trabalha na área de Payments (3 squads). O VP de Engineering espera que você garanta **alinhamento técnico** entre os squads sem ser chefe de ninguém. Cada squad tem seu tech lead, e nem todos concordam com suas recomendações.

---

## Desafios

### Desafio 2.1 — Influência sem Autoridade (Leading Without Title)

**Cenário:**
Três squads da área de Payments precisam migrar de REST para gRPC nos serviços internos. Você projetou a estratégia de migração, mas encontra resistências:
- **Squad Checkout:** "Não temos bandwidth, estamos focados em features"
- **Squad Billing:** "Preferimos GraphQL, por que gRPC?"
- **Squad Anti-Fraud:** "Concordamos mas só se alguém fizer o boilerplate pra gente"

Nenhum deles reporta a você. O VP disse: "Alinhe com eles, não quero impor top-down."

**Requisitos:**
- Mapear os **stakeholders** e suas motivações/resistências usando Stakeholder Map
- Desenvolver **estratégias de influência** adaptadas para cada perfil de resistência
- Criar um **proposal document** persuasivo (não é um mandato, é uma proposta)
- Praticar a arte de **build coalition** — encontrar aliados antes de propor publicamente

**Entregáveis:**

1. **Stakeholder Map** com análise de cada squad:
   | Squad | Tech Lead | Posição | Motivação Real | Estratégia de Influência |
   |-------|-----------|---------|---------------|-------------------------|
   | Checkout | [nome] | Contra | Medo de atraso em OKRs | Mostrar impacto em latência (dados) |
   | Billing | [nome] | Alternativa | Preferência técnica legítima | Sessão técnica de trade-offs |
   | Anti-Fraud | [nome] | Condicional | Falta de capacity/skill | Oferecer par + template |

2. **Influence Strategy Document:**
   - **Why now:** business case com dados (latência, custo, developer experience)
   - **Why gRPC:** análise comparativa honesta (gRPC vs GraphQL vs REST para o use case)
   - **Migration path:** incremental, não big bang (reduz risco para cada squad)
   - **What's in it for them:** benefícios específicos para CADA squad
   - **Ask:** o que você precisa de cada um (específico e pequeno para começar)

3. **Coalition Building Plan:**
   - Quem abordar primeiro (early adopter)
   - Como converter critic em ally (sessão técnica, POC conjunto, dados)
   - Como lidar com veto (escalation path, mas como ÚLTIMO recurso)
   - Timeline de conversas (1:1s antes de reunião coletiva)

**Critérios de aceite:**
- [ ] Stakeholder Map com análise de motivação real (não superficial)
- [ ] Influence Strategy com business case baseado em dados
- [ ] Análise honesta de trade-offs (não advocacy cega)
- [ ] Coalition Building Plan com sequência tática de conversas
- [ ] Reflexão: "Quando me pego querendo 'impor' em vez de 'influenciar'? O que dispara isso?"
- [ ] Plano B: o que fazer se a proposta for rejeitada (como aceitar e seguir em frente)

**Framework de Referência:**
- Tanya Reilly — *The Staff Engineer's Path* (Cap. 5: Leading Big Projects)
- Robert Cialdini — *Influence: The Psychology of Persuasion* (6 princípios)
- Will Larson — *Staff Engineer* (4 archetypes: Tech Lead, Architect, Solver, Right Hand)

---

### Desafio 2.2 — Technical Alignment (Tech Specs e RFCs)

**Cenário:**
A área de Payments precisa implementar um **sistema de retry distribuído** que será usado pelos 3 squads. Cada squad hoje tem sua própria implementação ad-hoc de retry, com comportamentos inconsistentes (alguns fazem exponential backoff, outros não; alguns respeitam circuit breaker, outros não). O VP quer uma **solução compartilhada**.

Você precisa escrever um **RFC (Request for Comments)** que será revisado pelos tech leads dos 3 squads e aprovado em Design Review.

**Requisitos:**
- Escrever RFC completo e persuasivo
- Antecipar e endereçar **objeções** de cada squad
- Incluir **alternatives considered** com análise honesta
- Definir processo de **RFC review** para a área

**Entregáveis:**

1. **RFC Template** reutilizável para a área de Payments:
   ```
   # RFC-YYYY-NNN: [Título]
   
   ## Metadata
   - Author(s):
   - Status: Draft | In Review | Accepted | Rejected | Superseded
   - Reviewers:
   - Created:
   - Decision deadline:
   
   ## Summary (TL;DR)
   [2-3 frases]
   
   ## Motivation
   [Por que resolver isso agora? Dados, impacto, customer pain]
   
   ## Proposal
   [Solução proposta em detalhe técnico]
   
   ## Alternatives Considered
   [Cada alternativa com pros/cons honestos]
   
   ## Migration Plan
   [Como adotar incrementalmente]
   
   ## Risks & Mitigations
   [O que pode dar errado e como mitigar]
   
   ## Open Questions
   [Perguntas que precisam de input dos reviewers]
   
   ## Decision Record
   [Preenchido APÓS review — decisão + rationale]
   ```

2. **RFC Preenchido: Distributed Retry System**
   - RFC completo para o retry system com:
     - Análise do estado atual (3 implementações ad-hoc)
     - Proposta: shared library vs sidecar vs platform service
     - Alternatives: 3 abordagens com trade-offs reais
     - Migration plan squad-by-squad
     - Riscos: backward compatibility, performance overhead

3. **RFC Review Process** para a área:
   - Quem deve participar (RACI para RFCs)
   - Timeline: Draft → Review Period (1 semana) → Decision Meeting → Record
   - Regras de review: como comentar construtivamente
   - Quando um RFC é necessário vs quando decidir no squad

**Critérios de aceite:**
- [ ] RFC Template completo e reutilizável
- [ ] RFC do retry system com pelo menos 3 alternativas e trade-offs honestos
- [ ] Migration plan incremental (não exige migração simultânea)
- [ ] RFC Review Process definido com RACI e timeline
- [ ] Reflexão: "Como equilibrar proposals detalhadas o suficiente com a velocidade necessária?"
- [ ] Simulação: lista de 5+ objeções prováveis dos tech leads e respostas preparadas

**Framework de Referência:**
- Uber/Google/Spotify RFC Processes
- Joel Spolsky — *Painless Functional Specifications*
- Architecture Decision Records (ADR) — Michael Nygard

---

### Desafio 2.3 — Mediação Técnica (Resolving Cross-Team Disputes)

**Cenário:**
Conflito entre squads: o squad **Checkout** reclama que o squad **Billing** mudou o contrato de uma API interna sem aviso, quebrando a integração em staging. O tech lead de Checkout mandou mensagem irritada no canal público: *"Billing quebrou nosso pipeline AGAIN. Quando vamos ter contratos estáveis?"*. O tech lead de Billing respondeu: *"Nosso changelog está no Confluence. Se vocês lessem a documentação..."*

O VP de Engineering pediu que você **medeie o conflito** e proponha um processo para evitar recorrência.

**Requisitos:**
- Fazer **fact-finding** neutro antes de propor solução (ouvir os dois lados)
- Separar **problema técnico** (breaking changes) do **conflito interpessoal** (comunicação pública)
- Mediar conversa entre os dois tech leads com postura **neutra e construtiva**
- Propor **solução sistêmica** (processo/tooling) além de resolver o conflito imediato

**Entregáveis:**

1. **Fact-Finding Protocol:**
   - 1:1 com tech lead Checkout: perguntas preparadas (fatos, não julgamentos)
   - 1:1 com tech lead Billing: perguntas preparadas (perspectiva, contexto)
   - Dados técnicos: timeline do incidente, changelog, comunicação prévia
   - Assessment: o que falhou? (comunicação? processo? tooling? cultura?)

2. **Mediation Script** para sessão conjunta:
   - **Abertura:** ground rules (fatos > julgamentos, "eu" > "você", foco em futuro > blame)
   - **Ouvir:** cada um tem 5 min para descrever o impacto do SEU lado
   - **Common ground:** "Vocês dois querem [X]. A divergência é em [Y]"
   - **Solutions:** brainstorm conjunto de soluções (não chegar com solução pronta)
   - **Agreement:** compromisso mútuo documentado
   - **Follow-up:** check-in em 2 semanas

3. **API Contract Management Proposal:**
   - Consumer-driven contract testing (Pact ou similar)
   - Breaking change policy (semver, deprecation notice, migration window)
   - Communication protocol para mudanças de API (PR review by consumers)
   - Tooling integration (CI check de backward compatibility)

**Critérios de aceite:**
- [ ] Fact-Finding Protocol com perguntas neutras para ambos os lados
- [ ] Mediation Script com etapas claras e linguagem facilitadora
- [ ] Separação explícita de problema técnico vs conflito interpessoal
- [ ] API Contract Management Proposal com solução sistêmica
- [ ] Reflexão: "Quando mediando, como evito tomar partido inconscientemente?"
- [ ] Endereçamento do comportamento público (mensagem irritada no canal) — como abordar sem blame

**Framework de Referência:**
- Thomas-Kilmann Conflict Mode Instrument (5 estilos de conflito)
- Harvard Negotiation Project — *Getting to Yes* (Fisher & Ury)
- Consumer-Driven Contracts (Ian Robinson)

---

### Desafio 2.4 — Scaling Engineering Practices (Guilds & Standards)

**Cenário:**
Com 70 engenheiros, cada squad desenvolveu suas próprias práticas: diferentes frameworks de teste, diferentes padrões de logging, diferentes estruturas de projeto. Uma engenheira que mudou de squad reporta: *"Levei 3 semanas para entender o código do novo squad. Parece outra empresa."*

O VP de Engineering quer **standards sem burocracia** — consistência onde importa, autonomia onde não importa. Você precisa criar um programa de **guilds** (communities of practice) e definir quais práticas padronizar.

**Requisitos:**
- Distinguir entre **standards obrigatórios** (segurança, observability) e **guidelines opcionais** (style, patterns)
- Criar estrutura de **guilds** (quem participa, cadência, output, governance)
- Facilitar a criação de **Golden Paths** (caminhos recomendados, não impostos)
- Garantir que standards são **propostos bottom-up** (não impostos top-down)

**Entregáveis:**

1. **Standards vs Guidelines Matrix:**
   | Categoria | Obrigatório (Standard) | Recomendado (Guideline) | Livre (Squad Decision) |
   |-----------|----------------------|------------------------|----------------------|
   | Segurança | Auth patterns, secret management | OWASP checklist | Auth library choice |
   | Observability | Structured logging, tracing | Dashboarding approach | Alert thresholds |
   | Testing | Integration test em CI | Coverage threshold | Test framework |
   | API | Contract documentation | Versioning strategy | Internal tool choice |
   | Code | Linting + formatting CI | Naming conventions | Architecture patterns |

2. **Guild Charter Template:**
   - **Missão:** por que esta guild existe (1 frase)
   - **Escopo:** o que está IN e OUT
   - **Membros:** voluntários, rotativo, pelo menos 1 rep por squad
   - **Cadência:** reuniões quinzenais de 45 min (time-boxed)
   - **Outputs:** standards, guidelines, golden paths, blog posts internos
   - **Governance:** como propor, revisar e aprovar standards (RFC-like)
   - **Success metrics:** adoption rate, developer satisfaction, onboarding time

3. **First Guild Launch Plan** — plano para lançar a primeira guild (ex: Observability Guild):
   - Kick-off: comunicação, convite, expectativa
   - Primeiro output: 1 standard proposto em 4 semanas
   - Feedback loop: como validar com squads antes de oficializar
   - Iteration: como evoluir o standard baseado em uso real

**Critérios de aceite:**
- [ ] Standards vs Guidelines Matrix com categorização clara e justificada
- [ ] Guild Charter Template reutilizável para qualquer guild
- [ ] First Guild Launch Plan com timeline realista
- [ ] Governança que garante proposals bottom-up (não imposição top-down)
- [ ] Reflexão: "Como garantir que standards não virem burocracia que mata velocidade?"
- [ ] Análise de risk: o que acontece se uma guild morrer? Como manter engajamento?

**Framework de Referência:**
- Spotify Model — Squads, Tribes, Chapters, Guilds
- Evan Bottcher — *Internal Platform Teams* (Golden Paths)
- Team Topologies — Platform Team enabling practices

---

### Desafio 2.5 — Onboarding em Escala (Ramp-Up de Novos Engineers)

**Cenário:**
A TechNova contratou 15 engenheiros em 2 meses (hypergrowth). O onboarding atual é caótico: cada squad faz diferente, alguns devs ficam 3 semanas sem conseguir subir o ambiente local, outros não entendem o domínio de negócio, e managers reclamam que "os novos não estão rendendo". 

Você precisa criar um **programa de onboarding escalável** para engenheiros.

**Requisitos:**
- Estruturar onboarding em **fases** (pre-boarding, semana 1, mês 1, mês 3)
- Equilibrar **conteúdo técnico** (ferramentas, codebase, arquitetura) e **relacional** (pessoas, cultura, squad dynamics)
- Criar sistema de **buddies** rotativo
- Definir **métricas de sucesso** do onboarding (time-to-first-PR, time-to-productive, eNPS)

**Entregáveis:**

1. **Onboarding Playbook** com fases:
   - **Pre-boarding** (antes do dia 1): equipamento, acessos, leitura preparatória, welcome message
   - **Semana 1:** meet the team, arch overview, first PR (definido e achievable), buddy assignment
   - **Mês 1:** domínio de negócio, pair programming rotation, first feature delivery, 30-day check-in
   - **Mês 3:** ownership de componente, contribuição para guild, 90-day review
   - Cada fase com: checklist, responsible, deliverables, success criteria

2. **Buddy Program:**
   - Quem é elegível para ser buddy (criteria)
   - O que se espera do buddy (responsibility matrix)
   - Duration: quanto tempo dura a relação de buddy
   - Training: como preparar buddies (buddy-of-buddy model)
   - Rotation: como evitar buddy burnout

3. **Onboarding Metrics Dashboard:**
   - Time-to-first-PR (target: < 3 dias)
   - Time-to-first-feature (target: < 2 semanas)
   - 30-day eNPS score
   - 90-day retention rate
   - Self-assessed confidence score (1-10) por semana

**Critérios de aceite:**
- [ ] Onboarding Playbook com 4 fases detalhadas e checklists
- [ ] Buddy Program com critérios, responsabilidades e anti-burnout
- [ ] Onboarding Metrics com targets quantitativos
- [ ] Balanceamento entre conteúdo técnico e relacional
- [ ] Reflexão: "Como medir se o onboarding é bom vs 'o novo é bom'?"
- [ ] Template de "30-day check-in" e "90-day review" para novos engineers

**Framework de Referência:**
- Lara Hogan — *Resilient Management* (cap. 2: Grow Your Teammates)
- Stripe — *How Stripe onboards new engineers*
- GitLab Handbook — Onboarding (público e open source)

---

### Desafio 2.6 — Staff Engineer Archetype (Self-Positioning)

**Cenário:**
Você tem 6 meses como Staff Engineer e seu manager pergunta: *"Qual tipo de Staff Engineer você quer ser? Tech Lead? Architect? Solver? Right Hand?"* Você não havia pensado nisso conscientemente. Ele explica: *"Preciso que você reflita sobre isso porque vai determinar como alocamos seu tempo e como medimos seu impacto."*

**Requisitos:**
- Estudar os **4 archetypes de Staff Engineer** (Will Larson)
- Fazer auto-análise honesta de qual archetype é seu **fit natural** vs qual a **organização mais precisa**
- Criar um **60-day plan** como Staff, baseado no archetype escolhido
- Definir como **medir impacto** de um Staff Engineer (não é linhas de código)

**Entregáveis:**

1. **Archetype Self-Assessment:**
   | Archetype | Descrição | Fit Pessoal (1-5) | Necessidade TechNova (1-5) | Gap |
   |-----------|-----------|:-----------------:|:-------------------------:|:---:|
   | **Tech Lead** | Guia approach + execution de 1 time crítico | ? | ? | ? |
   | **Architect** | Direção técnica responsável cross-time | ? | ? | ? |
   | **Solver** | Deep-dive em problemas complexos específicos | ? | ? | ? |
   | **Right Hand** | Extensão do VP/CTO, atua em gaps organizacionais | ? | ? | ? |

2. **60-Day Plan** baseado no archetype escolhido:
   - Semana 1-2: Quick wins que demonstram o papel
   - Semana 3-4: Primeiro projeto cross-team de impacto
   - Mês 2: Entregável de referência (RFC, migration, standard, etc.)
   - Métricas de impacto: como saber se está no caminho certo

3. **Impact Measurement Framework:**
   - Direto: projetos liderados, standards adotados, incidents prevenidos
   - Indireto: enablement (quantos engineers são mais produtivos por causa do seu trabalho?)
   - Team health: engineer satisfaction, retention, ramp-up time
   - Velocity: cycle time, deployment frequency nos times que você influencia
   - Brag doc mensal: template para documentar impacto

**Critérios de aceite:**
- [ ] Archetype Self-Assessment honesto com scores justificados
- [ ] 60-Day Plan específico e actionable (não genérico)
- [ ] Impact Measurement Framework com métricas diretas e indiretas
- [ ] Brag doc template preenchido com primeiro mês simulado
- [ ] Reflexão: "Qual archetype me assusta mais e por quê? O que isso revela?"
- [ ] Análise de tensão: onde o archetype natural conflita com a necessidade organizacional?

**Framework de Referência:**
- Will Larson — *Staff Engineer* (4 archetypes)
- Tanya Reilly — *The Staff Engineer's Path*
- Julia Grace — *What does a Staff Engineer actually do?*

---

## Entregáveis do Nível

| Artefato | Descrição |
|----------|-----------|
| Stakeholder Map + Influence Strategy | Liderança sem autoridade formal |
| RFC Template + RFC do Retry System | Alinhamento técnico documentado |
| Mediation Script + API Contract Proposal | Resolução de conflito cross-team |
| Guild Charter + Standards Matrix | Práticas escaláveis |
| Onboarding Playbook + Buddy Program | Ramp-up de novos engenheiros |
| Staff Archetype Assessment + 60-Day Plan | Auto-posicionamento na career ladder |

---

## Rubrica de Auto-Avaliação

| Competência | Iniciante (1) | Praticante (2) | Avançado (3) | Expert (4) |
|-------------|---------------|----------------|--------------|------------|
| **Influência** | Não consegue alinhar sem autoridade | Convence 1:1 mas perde em grupo | Coalition building natural, proposals data-driven | Times buscam seu input, influência é orgânica |
| **Tech Specs/RFCs** | Não documenta decisões | Documentação informal, ad-hoc | RFCs estruturados com alternatives e risks | RFC process adotado na área, cultura de documentação |
| **Mediação** | Toma partido ou evita conflito | Ouve ambos mas não facilita solução | Mediação neutra com solução sistêmica | Squads resolvem conflitos usando framework que você criou |
| **Standards** | No standards ou standards impostos | Standards definidos mas não adotados | Standards bottom-up com adoption pathway | Golden Paths com alta adoção voluntária |
| **Onboarding** | Cada squad faz diferente, caótico | Playbook existe mas não é seguido | Onboarding consistente com métricas | Self-service, novos hires são produtivos rapidamente |

---

## Próximo Nível

Quando completar todos os desafios deste nível, avance para:
→ [Level 3 — Principal: Direção de Área](03-principal-area-direction.md)

No próximo nível, sua esfera expande de **multi-team** para **área/plataforma** — com stakeholders mais seniores, decisões de maior impacto e a necessidade de pensar em **talent** e **org design** além de código.
