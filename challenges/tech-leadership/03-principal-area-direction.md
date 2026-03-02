# Level 3 — Principal: Direção de Área

> **Objetivo:** Dirigir a estratégia técnica de uma área/plataforma, tomar decisões de alto impacto,
> gerenciar stakeholders seniores e construir pipeline de talento técnico.

**Esfera de impacto:** Área / Plataforma (5-10 squads, 30-60 engenheiros)
**Referência:** Pré-requisito para [04-distinguished-org-capability.md](04-distinguished-org-capability.md)

---

## Contexto — TechNova (Série C)

A TechNova levantou Série C e tem 150 engenheiros em 20 squads organizados em 5 áreas (Payments, Marketplace, Core Platform, Data, Mobile). Você é **Principal Engineer** da área de Payments — a área de maior receita da empresa (3 squads de produto + 1 squad de plataforma, ~30 engenheiros). Você reporta tecnicamente ao VP de Engineering e pontilha com o CPO (Chief Product Officer) para alinhamento de roadmap. Além dos 3 Staff Engineers na área, há 8 Tech Leads e 4 Engineering Managers que você precisa influenciar.

---

## Desafios

### Desafio 3.1 — Technical Vision & Strategy Document

**Cenário:**
O CEO anunciou que a TechNova vai expandir para 3 novos países nos próximos 18 meses. Isso impacta massivamente a área de Payments: multi-currency, regulações locais (PCI-DSS, PSD2, LGPD equivalentes), gateways de pagamento regionais, compliance, latência cross-region. O VP de Engineering precisa de um **Technical Vision** da área de Payments para os próximos 2 anos.

**Requisitos:**
- Criar documento de **Technical Vision** que conecta estratégia de negócio com direção técnica
- Definir **princípios arquiteturais** que guiem decisões dos squads sem micromanagement
- Articular **bets técnicos** (apostas) com horizonte de investimento
- Balancear **inovação** (tech debt payoff, modernização) com **delivery** (features de produto)

**Entregáveis:**

1. **Technical Vision Document** (5-8 páginas):
   - **Business Context:** o que a TechNova quer nos próximos 2 anos (simples, sem jargão)
   - **Current State Assessment:** onde estamos tecnicamente (brutalmente honesto)
   - **Target Architecture:** para onde queremos ir (diagrama + narrativa)
   - **Architectural Principles:** 5-7 princípios que guiam decisões (ex: "Multi-tenancy first", "API-first", "Observability by default")
   - **Tech Bets:** 3-5 apostas técnicas com horizonte (6m/12m/24m), investimento e expected outcome
   - **Anti-Goals:** o que NÃO vamos fazer e por quê (tão importante quanto os goals)
   - **Success Metrics:** como saberemos que a visão está se concretizando

2. **Princípios Arquiteturais Detalhados** (1 página cada):
   - Cada princípio com: statement, rationale, implications, trade-offs
   - Exemplo: *"Multi-tenancy first: todo novo serviço deve suportar múltiplos merchants desde o design. Implication: database isolation por tenant. Trade-off: complexidade adicional no início vs custo de retrofit depois."*

3. **Investment Portfolio:** distribuição de engenharia proposta
   - Features de produto: X%
   - Tech debt / modernização: Y%
   - Platform / infrastructure: Z%
   - Exploração / R&D: W%
   - Justificativa para cada % baseada no momento da empresa

**Critérios de aceite:**
- [ ] Technical Vision conecta estratégia de negócio → direção técnica de forma clara
- [ ] Current State Assessment honesto (não wishful thinking)
- [ ] 5-7 princípios arquiteturais com trade-offs explícitos
- [ ] 3-5 tech bets com horizonte e expected outcome
- [ ] Anti-goals definidos (o que NÃO fazer é tão importante)
- [ ] Reflexão: "Como comunicar a visão para que os 30 engenheiros da área entendam e comprem?"

**Framework de Referência:**
- Will Larson — *An Elegant Puzzle* (Cap. 5: Culture)
- ThoughtWorks — Technology Radar approach
- Amazon 6-pager format (narrative over slides)

---

### Desafio 3.2 — Stakeholder Management (Navigating the Matrix)

**Cenário:**
Você precisa navegar múltiplos stakeholders com interesses frequentemente conflitantes:
- **VP Engineering:** quer reduzir incidents e melhorar reliability (investir em platform)
- **CPO:** quer features para expansão internacional (investir em produto)
- **CFO:** quer reduzir custo de infraestrutura em 30% (investir em otimização)
- **CISO:** quer compliance PCI-DSS Level 1 para IPO (investir em segurança)
- **Staff Engineers:** querem modernizar stack (investir em tech debt)
- **Engineering Managers:** querem reter talentos (investir em developer experience)

Todos têm razão. O budget é limitado. Você precisa **alinhar expectativas e priorizar**.

**Requisitos:**
- Mapear **stakeholders** com interesse, poder e relação atual (Support/Neutral/Oppose)
- Desenvolver **communication cadence** personalizada para cada stakeholder
- Criar **one-pager** que endereça múltiplos interesses sem prometer tudo
- Praticar **managing up** — como influenciar quem tem mais poder que você

**Entregáveis:**

1. **Stakeholder Matrix:**
   | Stakeholder | Interesse | Poder | Relação Atual | Relação Desejada | Estratégia |
   |-------------|-----------|:-----:|:-------------:|:----------------:|-----------|
   | VP Eng | Reliability | Alto | Support | Champion | Data de incidents, SLO dashboard |
   | CPO | Features | Alto | Neutral | Support | Roadmap com dates, trade-offs explícitos |
   | CFO | Custo | Alto | Oppose | Neutral | ROI de cada investimento técnico |
   | CISO | Compliance | Médio | Neutral | Support | Compliance roadmap com milestones |

2. **Communication Cadence Plan:**
   | Stakeholder | Formato | Frequência | Conteúdo | Canal |
   |-------------|---------|-----------|----------|-------|
   | VP Eng | 1:1 | Semanal | Status, risks, decisions needed | Presencial |
   | CPO | Status update | Quinzenal | Feature progress, dependencies | Escrito + meeting |
   | CFO | Dashboard | Mensal | Custo, trend, savings | Email + quarterly review |
   | CISO | Compliance review | Mensal | Gap analysis, remediation progress | Meeting formal |

3. **Priority One-Pager** (para apresentar ao leadership team):
   - Framework de priorização usado (RICE, ICE, ou custom)
   - Trade-offs explícitos: "Se priorizarmos X, Y fica para Q3"
   - Proposta de alocação por quarter
   - Ask claro: "Preciso de decisão sobre [A vs B]"
   - Riscos de cada caminho com probabilidade e impacto

**Critérios de aceite:**
- [ ] Stakeholder Matrix com todos os 6+ stakeholders mapeados
- [ ] Communication Cadence com formato e frequência por stakeholder
- [ ] Priority One-Pager claro e acessível para audiência não-técnica
- [ ] Trade-offs explícitos (não "vamos fazer tudo")
- [ ] Reflexão: "Qual stakeholder eu evito e por quê? Como superar isso?"
- [ ] Script de "managing up" — como levar bad news ao VP de forma construtiva

**Framework de Referência:**
- Colin Powell — "Tell me what you know, what you don't know, and what you think"
- Mendelow's Stakeholder Matrix (Power × Interest)
- The Minto Pyramid — *Pyramid Principle* (comunicação top-down)

---

### Desafio 3.3 — Hiring & Technical Interview (Raising the Bar)

**Cenário:**
A área de Payments precisa contratar 8 engenheiros em 6 meses para suportar a expansão. O processo de hiring atual é inconsistente: cada entrevistador avalia como quer, sem rubrica. Candidatos reportam experiências negativas ("cada entrevista perguntava a mesma coisa"). O VP pediu que você **redesenhe o processo de hiring técnico** e seja **bar raiser** (garante qualidade das contratações).

**Requisitos:**
- Definir **perfil ideal** por nível (Jr, Pleno, Senior) com competências técnicas e comportamentais
- Desenhar **pipeline de entrevistas** com etapas claras e avaliação complementar
- Criar **rubrica de avaliação** objetiva e calibrável
- Treinar entrevistadores para **reduzir bias** e melhorar candidate experience

**Entregáveis:**

1. **Job Profile Matrix:**
   | Competência | Junior | Pleno | Senior |
   |-------------|--------|-------|--------|
   | **Coding** | Resolve problemas simples, código limpo | Resolve problemas complexos, testes | Design de solução, mentoria de cod style |
   | **System Design** | Entende componentes | Projeta serviço | Projeta sistema distribuído |
   | **Collaboration** | Trabalha no squad | Facilita no squad | Influencia cross-team |
   | **Problem Solving** | Segue approach dado | Define approach | Define approach + alternativas |

2. **Interview Pipeline:**
   | Etapa | Duração | Avaliador | Foco | Ferramenta |
   |-------|---------|-----------|------|-----------|
   | 1. Screening | 30 min | Recruiter + Eng | Fit básico, motivação | Checklist |
   | 2. Coding | 60 min | 2 engineers | Problem solving, code quality | Pair programming |
   | 3. System Design | 60 min | Staff/Principal | Architecture, trade-offs | Whiteboard |
   | 4. Behavioral | 45 min | EM + Senior | Culture fit, collaboration | STAR questions |
   | 5. Bar Raiser | 30 min | Principal | Cross-check, veto power | Rubrica final |

3. **Evaluation Rubric** com escala 1-4 por competência:
   - 1 (No Hire): descrição do que é inaceitável
   - 2 (Lean No): gaps significativos em competências core
   - 3 (Lean Yes): atende expectations com potencial de crescimento
   - 4 (Strong Yes): supera expectations, eleva o time

4. **Interviewer Training Guide:**
   - Bias awareness: 5 tipos de bias em entrevistas (halo, similarity, anchoring, etc.)
   - Técnicas anti-bias: structured interview, rubrica antes do debrief, independent scoring
   - Candidate experience: como conduzir entrevista mesmo quando o candidato vai mal
   - Legal considerations: perguntas que NUNCA fazer

**Critérios de aceite:**
- [ ] Job Profile Matrix para 3 níveis com competências técnicas e comportamentais
- [ ] Interview Pipeline com 5 etapas complementares (sem repetição)
- [ ] Evaluation Rubric com escala descritiva (não apenas numérica)
- [ ] Interviewer Training Guide com bias awareness
- [ ] Reflexão: "Quais biases eu tenho como entrevistador e como mitigar?"
- [ ] Candidate experience consideration: plano para feedback pós-entrevista

**Framework de Referência:**
- Amazon Bar Raiser Program
- Laszlo Bock — *Work Rules!* (Google's hiring approach)
- STAR Method (Situation, Task, Action, Result) para behavioral interviews

---

### Desafio 3.4 — Decisões Técnicas de Alto Impacto (Build vs Buy vs Open Source)

**Cenário:**
A TechNova precisa de um sistema de **fraud detection** para a expansão internacional. Três opções:
- **Build:** construir internamente com ML team (12 meses, 4 engenheiros dedicados, total control)
- **Buy:** contratar SaaS de fraud detection (Stripe Radar/Sift, 2 meses de integração, custo mensal alto)
- **Open Source:** usar Apache Flink + ML pipeline (6 meses, customizável, mas precisa de expertise)

Cada opção tem defensores apaixonados. A decisão impacta roadmap de 2+ anos. CTO e CPO querem recomendação da área de Payments.

**Requisitos:**
- Estruturar análise com framework **rigoroso e reproduzível** (não gut feeling)
- Envolver stakeholders corretos no processo decisório
- Documentar decisão de forma que futuros engenheiros entendam o rationale
- Planejar **exit strategy** (o que fazer se a decisão estiver errada)

**Entregáveis:**

1. **Decision Framework (Build vs Buy vs OSS):**
   | Critério | Peso | Build | Buy | Open Source |
   |----------|:----:|:-----:|:---:|:-----------:|
   | Time-to-market | 25% | 2 | 5 | 3 |
   | Total Cost (3 anos) | 20% | 3 | 2 | 4 |
   | Customizability | 20% | 5 | 2 | 4 |
   | Maintenance burden | 15% | 2 | 5 | 3 |
   | Talent dependency | 10% | 3 | 5 | 2 |
   | Strategic value | 10% | 5 | 1 | 3 |
   | **Weighted Score** | | ? | ? | ? |
   
   - Escala 1-5 com descrição do que cada score significa
   - Pesos justificados pela necessidade da TechNova neste momento

2. **Recommendation Document** (formato one-pager executive):
   - Recomendação (1 frase)
   - Por que agora (urgency)
   - Framework usado (transparência)
   - Trade-offs aceitos (honestidade)
   - Riscos e mitigações
   - Ask: aprovação + recursos necessários

3. **Exit Strategy:**
   - Para cada opção: "e se precisarmos mudar em 18 meses?"
   - Sunk cost analysis: quando pivotar vs insistir
   - Reversibility assessment: quão reversível é cada decisão
   - Trigger points: métricas que indicam que a decisão precisa ser revisitada

**Critérios de aceite:**
- [ ] Decision Framework com critérios pesados e scores justificados
- [ ] Recommendation Document objetivo e non-technical-friendly
- [ ] Exit Strategy para cada opção (demonstra maturidade de pensamento)
- [ ] Processo decisório inclui input de stakeholders e não é unilateral
- [ ] Reflexão: "Como separo minha preferência técnica pessoal da melhor decisão para a empresa?"
- [ ] Análise de reversibility: classificar a decisão como Type 1 (irreversível) ou Type 2 (reversível)

**Framework de Referência:**
- Amazon — Type 1 vs Type 2 decisions
- Martin Fowler — *Build vs Buy* (bliki)
- McKinsey Decision Framework — (relevance, achievability, risk)

---

### Desafio 3.5 — Talent Development & Succession Planning

**Cenário:**
A área de Payments tem 30 engenheiros. O VP pergunta: *"Se você sair amanhã, quem segura a área? Se o Staff Engineer X sair, quem sobe? Temos bus factor 1 em componentes críticos?"*

Você percebe que nunca pensou em **succession planning** para um IC track. Também percebe que não tem um processo de **talent development** — as promoções acontecem ad-hoc, e vários engenheiros competentes não sabem o que fazer para crescer.

**Requisitos:**
- Criar **talent map** da área com assessment de performance e potencial
- Identificar **succession risks** (bus factor, single points of failure)
- Desenvolver **growth paths** personalizados para 3-5 engenheiros-chave
- Estabelecer processo de **calibration** (como avaliar consistentemente)

**Entregáveis:**

1. **Talent Map (9-Box Grid):**
   ```
              Low Performance    Medium Performance    High Performance
   High      ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   Potential  │   Enigma     │   │   Growth     │   │   Star       │
              │              │   │   Candidate  │   │              │
              └──────────────┘   └──────────────┘   └──────────────┘
   Medium    ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   Potential  │   Risk       │   │   Core       │   │   High       │
              │              │   │   Performer  │   │   Professional│
              └──────────────┘   └──────────────┘   └──────────────┘
   Low       ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   Potential  │   Action     │   │   Consistent │   │   Expert     │
              │   Needed     │   │   Performer  │   │   Specialist │
              └──────────────┘   └──────────────┘   └──────────────┘
   ```
   - Posicionar 10+ engenheiros fictícios (simulando a área de 30)
   - Justificativa para cada posicionamento
   - Action plan por quadrante (não apenas para Stars)

2. **Succession Risk Matrix:**
   | Componente/Sistema | Owner Atual | Bus Factor | Backup | Risk Level | Mitigation |
   |-------------------|-------------|:----------:|--------|:----------:|-----------|
   | Payment Gateway | Staff Eng X | 1 | Ninguém | 🔴 Critical | Pair + doc |
   | Reconciliation | Senior Y | 2 | Junior Z | 🟡 Medium | Cross-train |

3. **Individual Growth Plans** (3 exemplos):
   - Engenheiro **Senior → Staff**: o que falta, milestones, timeline, support needed
   - Engenheiro **Pleno → Senior**: competências a desenvolver, projetos de stretch
   - Engenheiro **Expert mas sem promoção**: carreira lateral? especialização? novo desafio?

4. **Calibration Process:**
   - Como conduzir sessão de calibration (quem participa, formato, duração)
   - Como evitar bias (recency, halo, similarity)
   - Como comunicar resultado para o engenheiro (feedback construtivo)
   - Frequência: semestral com check-ins trimestrais

**Critérios de aceite:**
- [ ] Talent Map com 10+ posicionamentos justificados
- [ ] Succession Risk Matrix com pelo menos 5 componentes críticos
- [ ] 3 Individual Growth Plans específicos e actionable
- [ ] Calibration Process documentado com anti-bias practices
- [ ] Reflexão: "Como avaliar potencial sem confundir com performance? E sem confundir com 'parece comigo'?"
- [ ] Análise: como succession planning no IC track difere do management track?

**Framework de Referência:**
- 9-Box Grid (McKinsey talent management)
- Bus Factor analysis
- Will Larson — *An Elegant Puzzle* (Cap. 6: Careers)

---

### Desafio 3.6 — Crisis Leadership (Production Incident de Grande Escala)

**Cenário:**
Sexta-feira, 16h. O sistema de Payments para de processar transações. O impacto financeiro é estimado em R$ 50.000/minuto perdido. O CEO está ligando para o VP de Engineering. A imprensa começa a notar (tweets de merchants reclamando). Dois squads estão tentando debugar simultaneamente sem coordenação, criando mais confusão. Você é o **IC mais senior disponível** — o VP está em call com o CEO. Precisa liderar o incident response.

**Requisitos:**
- Estruturar **incident response** com roles claros (Incident Commander, Comms Lead, etc.)
- Tomar **decisões sob pressão** com informação incompleta
- Gerenciar **comunicação durante crise** (interna e externa)
- Conduzir **blameless post-mortem** e converter em melhorias sistêmicas

**Entregáveis:**

1. **Incident Response Playbook:**
   - **Roles & Responsibilities:**
     - Incident Commander (IC): coordena, não debugga
     - Tech Lead: lidera investigação técnica
     - Comms Lead: atualiza stakeholders e externos
     - Scribe: documenta timeline em tempo real
   - **Severity Levels:** S1-S4 com criteria e response time
   - **Communication Templates:**
     - Internal (Slack): status update a cada 15 min
     - Customer (status page): linguagem cuidadosa, sem blame
     - Executive (email): impacto, ETA, actions in progress
   - **Escalation Path:** quando e como escalar

2. **Incident Simulation** — narração passo-a-passo do incidente:
   - 16:00 — Alerta disparado. O que você faz primeiro?
   - 16:10 — CEO liga. VP não atende. O que comunica?
   - 16:30 — Time A acha que é database, Time B acha que é gateway. Como coordena?
   - 17:00 — Fix parcial disponível mas com risco. Deploy agora ou esperar?
   - 17:30 — Sistema volta. Comunicação post-incident?
   - Decisões documentadas com rationale em cada momento

3. **Blameless Post-Mortem Template:**
   - Executive Summary (2-3 frases)
   - Timeline com fatos (não julgamentos)
   - Root Cause Analysis (5 Whys ou Fishbone)
   - Impact Assessment (financeiro, reputacional, técnico)
   - Action Items com owner, deadline, priority (P0/P1/P2)
   - Learnings (sistêmicos, não pessoais)
   - Follow-up process (quem verifica que AIs foram executados)

**Critérios de aceite:**
- [ ] Incident Response Playbook com roles, severity levels e templates
- [ ] Incident Simulation com decisões documentadas e rationale
- [ ] Blameless Post-Mortem Template completo e reutilizável
- [ ] Communication templates para 3 audiências (internal, customer, executive)
- [ ] Reflexão: "Como tomo decisões com 60% da informação? E como lido com o desconforto?"
- [ ] Análise: 3 melhorias sistêmicas que preveniriam o incidente (foco em sistema, não em pessoas)

**Framework de Referência:**
- Google SRE Book — Cap. 14-15 (Incident Management, Postmortems)
- PagerDuty Incident Response Guide
- Sidney Dekker — *The Field Guide to Understanding Human Error*

---

## Entregáveis do Nível

| Artefato | Descrição |
|----------|-----------|
| Technical Vision Document | Visão técnica de 2 anos para área de Payments |
| Stakeholder Matrix + Communication Cadence | Navegação de múltiplos stakeholders |
| Hiring Pipeline + Evaluation Rubric | Processo de hiring técnico consistente |
| Decision Framework (Build/Buy/OSS) | Decisões de alto impacto estruturadas |
| Talent Map + Succession Plan | Desenvolvimento e retenção de talento |
| Incident Response Playbook | Liderança em crise com comunicação estruturada |

---

## Rubrica de Auto-Avaliação

| Competência | Iniciante (1) | Praticante (2) | Avançado (3) | Expert (4) |
|-------------|---------------|----------------|--------------|------------|
| **Vision & Strategy** | Não articula visão técnica | Visão existe mas desconectada do business | Visão clara conectando business → tech com princípios | Visão inspira e guia decisões autônomas dos squads |
| **Stakeholder Mgmt** | Evita stakeholders difíceis | Reativo, responde quando demandado | Proativo, cadência adaptada, manages up | Trusted advisor, stakeholders buscam seu input |
| **Hiring** | Entrevista sem rubrica, gut feeling | Rubrica básica, processo inconsistente | Pipeline estruturado, bias awareness, bar raiser | Pipeline reproduzível, candidate experience excelente |
| **Decisão de Alto Impacto** | Paralisia ou decisão por gut feeling | Framework básico, mas evita trade-offs difíceis | Framework rigoroso, exit strategy, reversibility | Decisões bem documentadas que resistem ao tempo |
| **Talent Development** | Não pensa em succession | Identifica gaps mas não atua | Growth plans ativos, calibration regular | Talent pipeline saudável, promoções previsíveis |
| **Crisis Leadership** | Pânico ou paralisia | Consegue liderar mas comunicação falha | IC eficaz, comunicação clara, post-mortem útil | Cultura de incident response, equipe auto-organizada |

---

## Próximo Nível

Quando completar todos os desafios deste nível, avance para:
→ [Level 4 — Distinguished: Capacidade Organizacional](04-distinguished-org-capability.md)

No próximo nível, sua esfera expande de **área** para **organização inteira** — definindo cultura de engenharia, engineering ladder, governance e capacidade técnica que sustentam centenas de engenheiros.
