# Level 4 — Distinguished: Capacidade Organizacional

> **Objetivo:** Construir a capacidade técnica da organização inteira — cultura de engenharia,
> engineering ladder, org design, diversidade e gestão de transformações de grande escala.

**Esfera de impacto:** Organização (200-500 engenheiros, 30-50 squads)
**Referência:** Pré-requisito para [05-fellow-technology-strategy.md](05-fellow-technology-strategy.md)

---

## Contexto — TechNova (Pré-IPO)

A TechNova está se preparando para IPO. São 350 engenheiros em 45 squads organizados em 8 áreas. Você é **Distinguished Engineer** — um dos 3 ICs mais seniores da empresa. Seu papel é garantir que a **organização de engenharia como um todo** é capaz de entregar com qualidade, escalar com crescimento e reter talento de elite. Você senta no Engineering Leadership Team junto com o CTO, VP of Engineering e os 8 Engineering Directors.

---

## Desafios

### Desafio 4.1 — Engineering Culture Document

**Cenário:**
Com 350 engenheiros, a cultura de engenharia diluiu. Os fundadores tinham valores claros, mas os novos contratados não viveram a evolução. Engenheiros de áreas diferentes reportam culturas conflitantes: um time prioriza move fast, outro prioriza zero bugs. O CTO pede que você **codifique a cultura de engenharia** da TechNova.

**Requisitos:**
- Codificar **engineering values** que sejam vividos, não apenas palavras na parede
- Conectar cada value com **comportamentos observáveis** (o que significa na prática)
- Criar processo de **evolução cultural** (cultura não é estática)
- Endereçar tensão entre **subcultures** legítimas (platform vs produto, por exemplo)

**Entregáveis:**

1. **Engineering Culture Document:**
   - **5-7 Engineering Values** com:
     - Statement (1 frase)
     - "Isso significa..." (3+ comportamentos concretos)
     - "Isso NÃO significa..." (anti-patterns)
     - Tension pairs: como resolver quando 2 values conflitam
   - Exemplo:
     ```
     Value: "Own the outcome, not the output"
     Isso significa: medir impacto, não linhas de código; fazer post-mortem, não apontar dedo
     Isso NÃO significa: trabalhar fim de semana; ignorar processo de review
     Tensão com "Move fast": quando velocidade compromete outcome,
     outcome vence — mas documentar o trade-off
     ```

2. **Culture Health Survey** (template):
   - 15-20 perguntas que medem se os values estão sendo vividos
   - Escala Likert + perguntas abertas
   - Segmentação por área, tenure, seniority
   - Análise de gaps (value declarado vs value vivido)

3. **Culture Evolution Process:**
   - Como propor mudanças nos values (RFC cultural?)
   - Quem participa da decisão (governance)
   - Cadência de revisão (anual?)
   - Como onboardar novos engineers nos values

**Critérios de aceite:**
- [ ] 5-7 values com comportamentos concretos e anti-patterns
- [ ] Tension pairs explícitos (como resolver conflitos entre values)
- [ ] Culture Health Survey com 15+ perguntas aplicáveis
- [ ] Culture Evolution Process com governance clara
- [ ] Reflexão: "Cultura é o que acontece quando ninguém está olhando. Como garantir que values se sustentam sem enforcement?"
- [ ] Análise: como lidar com subcultures legítimas (platform team tem cultura diferente de product team)

**Framework de Referência:**
- Westrum Organizational Culture (pathological → bureaucratic → generative)
- Netflix Culture Doc (*Freedom & Responsibility*)
- GitLab Values (transparência radical)

---

### Desafio 4.2 — Engineering Ladder Design

**Cenário:**
A TechNova tem um career ladder informal: Junior → Pleno → Senior → Staff → Principal. Mas não há definição clara do que cada nível espera. Promoções são inconsistentes entre áreas. Engenheiros reclamam: *"não sei o que fazer para ser promovido"* e *"na área X é mais fácil ser promovido que na área Y"*. O VP pede que você crie o **engineering ladder oficial**.

**Requisitos:**
- Definir **competências e expectativas** para cada nível (IC track)
- Garantir **calibração** consistente entre áreas
- Criar **rubrica de avaliação** que evite bias
- Definir **processo de promoção** transparente

**Entregáveis:**

1. **Engineering Ladder** (IC Track):
   | Nível | Scope | Technical Skill | Leadership | Autonomy |
   |-------|-------|----------------|-----------|----------|
   | **Junior (L1)** | Task | Aprende e aplica | Aprende com o time | Dirigida |
   | **Pleno (L2)** | Feature | Executa independente | Contribui para o time | Guiada |
   | **Senior (L3)** | Squad | Resolve problemas complexos | Mentor no squad | Autônoma |
   | **Staff (L4)** | Multi-team | Define approach técnico | Influencia entre times | Auto-dirigida |
   | **Principal (L5)** | Area/Platform | Direção técnica estratégica | Líder na área | Estratégica |
   | **Distinguished (L6)** | Organization | Capacidade organizacional | Líder na organização | Transformacional |
   | **Fellow (L7)** | Industry | Inovação e visão | Líder na indústria | Visionária |

   Para cada nível, documentar:
   - **5 competências** com descrição detalhada
   - **Exemplos de impacto** para evidenciar o nível
   - **Anti-patterns** comuns (senior que atua como pleno, staff que atua como senior)

2. **Promotion Criteria & Process:**
   - Quem propõe: manager + sponsor (IC senior que atesta impact)
   - Evidence packet: o que preparar (brag doc, impact examples, peer feedback)
   - Calibration committee: quem participa, como evitar bias
   - Decision communication: como comunicar approved e denied
   - Timeline: quando e com que frequência

3. **Calibration Guide:**
   - Como conduzir sessão de calibration (passo a passo)
   - Checklist anti-bias (recency, halo, similarity, contrast, anchor)
   - Exercícios de calibração: cases fictícios para alinhar os calibradores
   - Template de calibration notes (o que documentar e o que NÃO documentar)

**Critérios de aceite:**
- [ ] Engineering Ladder com 7 níveis detalhados em 5+ competências
- [ ] Exemplos de impacto e anti-patterns para cada nível
- [ ] Promotion process transparente com evidence packet
- [ ] Calibration guide com anti-bias checklist
- [ ] Reflexão: "Como garantir que o ladder não se torne burocracia que desanima em vez de motivar?"
- [ ] Análise: IC track vs Management track — como garantir paridade real (não apenas nominal)?

**Framework de Referência:**
- Progression.fyi (collection de career ladders públicos)
- Carta / Buffer / Spotify engineering ladders
- Will Larson — *Staff Engineer* (defining Staff scope)

---

### Desafio 4.3 — Organizational Design for Engineering

**Cenário:**
Com 350 engenheiros, a TechNova está sentindo dores de crescimento. Problemas:
- Squads muito acoplados — mudanças de API quebram 4 times
- Platform team sobrecarregado — fila de 3 meses para qualquer pedido
- Ownership unclear — "de quem é esse serviço?" é pergunta frequente
- Communication overhead — meetings demais, alignment meetings sobre alignment meetings

O CTO quer **reorganizar engineering**. Você precisa propor a nova estrutura.

**Requisitos:**
- Aplicar **Team Topologies** para propor organização eficaz
- Definir **ownership model** claro (quem é dono de quê)
- Reduzir **cognitive load** e **coordination overhead**
- Planejar **transition** da org atual para a nova estrutura (change management)

**Entregáveis:**

1. **Current State Assessment:**
   - Mapa de dependencies entre times (quem depende de quem)
   - Cognitive Load Assessment por time (quantos domínios, serviços, tecnologias)
   - Communication patterns: quem fala com quem (Conway's Law analysis)
   - Pain points documentados com evidência (tickets, delays, incident data)

2. **Proposed Org Design** usando Team Topologies:
   - **Stream-aligned teams:** quais, qual domínio, qual product surface
   - **Platform teams:** quais, quais capabilities, qual self-service
   - **Enabling teams:** quais, qual missão, quando se dissolvem
   - **Complicated-subsystem teams:** quais, por que isolar
   - Team interaction modes: collaboration, X-as-a-Service, facilitating
   - Mapa visual da nova organização

3. **Transition Plan:**
   - Phases: como migrar da org atual para a proposta (não big bang!)
   - Communication plan: como anunciar, como lidar com medo e resistência
   - People impact: quem muda de time, como minimizar disruption
   - Success metrics: como saber se a reorg funcionou (6 meses after)
   - Risk mitigation: o que fazer se não funcionar

**Critérios de aceite:**
- [ ] Current State Assessment baseado em dados (não intuição)
- [ ] Org Design proposto usando Team Topologies vocabulary
- [ ] Ownership model claro (cada serviço/domínio tem exatamente 1 time dono)
- [ ] Transition Plan gradual com change management
- [ ] Reflexão: "Como garantir que a reorg resolve problemas em vez de criar novos?"
- [ ] Análise de Conway's Law: a nova org design reflete a arquitetura desejada?

**Framework de Referência:**
- Matthew Skelton & Manuel Pais — *Team Topologies*
- Mel Conway — *Conway's Law* (org structure → system architecture)
- Heidi Helfand — *Dynamic Reteaming* (como times mudam)

---

### Desafio 4.4 — Diversity, Equity & Inclusion in Engineering

**Cenário:**
Uma análise de dados revela problemas preocupantes na engenharia da TechNova:
- Apenas 18% dos engenheiros são mulheres (mercado benchmark: 25%)
- 0% dos Distinguished/Fellow são de grupos sub-representados
- Turnover de mulheres é 2x o de homens nos primeiros 12 meses
- Feedback 360 revela: *"É difícil ser ouvida nas design reviews"*
- Pipeline de hiring: 40% mulheres no topo do funil → 15% nas contratações

O CTO declara D&I como prioridade. Você precisa propor um **plano de ação concreto** (não apenas declaração de intenções).

**Requisitos:**
- Analisar dados com rigor (não assumptions)
- Propor ações em **hiring, retention e progression** (pipeline completo)
- Incluir **métricas mensuráveis** (não vague)
- Endereçar aspectos **culturais e sistêmicos** (não apenas números)

**Entregáveis:**

1. **Data Analysis & Diagnosis:**
   - Funil de hiring: onde as candidatas desistem ou são eliminadas?
   - Retention analysis: por que o turnover é 2x? (exit interview patterns)
   - Progression analysis: rate de promoção por gênero/grupo
   - Culture analysis: temas recorrentes em feedback de grupos sub-representados
   - Benchmark: comparação com mercado e best practices

2. **Action Plan (Hiring → Retention → Progression):**
   - **Hiring:**
     - Job description review (linguagem inclusiva, requisitos essenciais vs nice-to-have)
     - Diverse interview panels (regra de composição)
     - Pipeline diversity (parcerias com comunidades, bootcamps, universidades)
     - Structured interviews (reduz bias conforme Desafio 3.3)
   - **Retention:**
     - Mentorship program para grupos sub-representados
     - Employee Resource Groups (ERGs)
     - Management training em inclusive leadership
     - Salary equity audit (transparência salarial)
   - **Progression:**
     - Sponsorship (não apenas mentorship) para promoção
     - Calibration bias review (análise de padrões por grupo)
     - Visibility opportunities (conferências, tech talks, projects de alto impacto)

3. **Métricas & Accountability:**
   | Métrica | Atual | Target (12m) | Target (24m) | Owner |
   |---------|:-----:|:------------:|:------------:|-------|
   | % mulheres eng | 18% | 22% | 28% | Hiring + D&I |
   | Turnover mulheres | 2x | 1.5x | 1x | EM + HR |
   | % sub-rep em L4+ | 0% | 5% | 10% | Calibration committee |
   | Inclusion score | ? | +15% | +30% | Culture survey |

**Critérios de aceite:**
- [ ] Data analysis rigorosa (fatos > assumptions)
- [ ] Action plan cobrindo hiring, retention E progression
- [ ] Métricas mensuráveis com targets e owners
- [ ] Ações sistêmicas (processo, cultura) além de ações pontuais
- [ ] Reflexão: "Quais dos meus próprios biases contribuem invisvelmente para o problema?"
- [ ] Análise: como evitar tokenism e garantir que diversidade gere pertencimento real

**Framework de Referência:**
- Project Include — *Comprehensive guide to D&I in tech*
- McKinsey — *Diversity Wins* (business case for diversity)
- Iris Bohnet — *What Works: Gender Equality by Design*

---

### Desafio 4.5 — Engineering Effectiveness & Developer Experience

**Cenário:**
O CTO está preocupado: *"Estamos com 350 engenheiros mas entregamos como se fossemos 200. Onde está o atrito?"* Ele pede que você investigue e proponha melhorias na **engineering effectiveness** sem "fazer todo mundo trabalhar mais".

**Requisitos:**
- Medir **developer experience** atual (não apenas velocity de JIRA tickets)
- Identificar **toil** (trabalho repetitivo que poderia ser automatizado)
- Propor melhorias em **tooling, process e culture**
- Definir **DORA metrics** e framework de medição contínua

**Entregáveis:**

1. **Developer Experience Survey:**
   - 20-25 perguntas cobrindo:
     - Environment setup (quanto tempo para rodar o projeto?)
     - CI/CD experience (quanto tempo de pipeline? quantos breaks?)
     - Code review turnaround (tempo médio para first review?)
     - Documentation quality (consegue entender código de outro time?)
     - Tooling satisfaction (IDEs, infra, testing tools)
     - Meeting load (% do tempo em meetings vs coding)
     - Cognitive load (quantos contextos diferentes na semana?)
   - Mix de escala Likert e perguntas abertas
   - Segmentação por área, seniority, tenure

2. **Toil Inventory:**
   | Categoria | Toil Identificado | Impacto (h/sem/time) | Solução Proposta | Effort | Impact |
   |-----------|------------------|:--------------------:|-----------------|:------:|:------:|
   | Env Setup | Setup manual de 2 dias | 16h/quarter | Dev containers | M | H |
   | Testing | Testes flaky bloqueiam PR | 8h/sem/time | Quarantine + fix | S | H |
   | Deploy | Deploy manual com 12 steps | 4h/sem/time | Automated pipeline | L | H |
   | Oncall | Alert fatigue (30+ alerts/dia) | 10h/sem/oncall | Tune thresholds | M | M |

3. **DORA Metrics Baseline + Targets:**
   | Metric | Current | Target (6m) | Target (12m) | Elite Benchmark |
   |--------|:-------:|:-----------:|:------------:|:---------------:|
   | Deployment Frequency | Weekly | Daily | Multiple/day | On-demand |
   | Lead Time for Changes | 2 weeks | 1 week | < 1 day | < 1 hour |
   | Change Failure Rate | 25% | 15% | < 10% | < 5% |
   | Time to Restore | 4 hours | 1 hour | < 30 min | < 10 min |

4. **Improvement Roadmap** (prioritized):
   - Quick wins (< 2 semanas, alto impacto)
   - Medium-term (1-3 meses, investimento moderado)
   - Long-term (3-6 meses, transformacional)
   - Para cada: sponsor, team, expected ROI

**Critérios de aceite:**
- [ ] Developer Experience Survey com 20+ perguntas relevantes
- [ ] Toil Inventory com pelo menos 10 itens quantificados
- [ ] DORA Metrics baseline e targets realistas
- [ ] Improvement Roadmap prioritizado com quick wins identificados
- [ ] Reflexão: "Como medir productivity sem criar culture de surveillance?"
- [ ] Análise: DORA metrics medem o que importa? O que elas NÃO capturam?

**Framework de Referência:**
- DORA (DevOps Research & Assessment) — *Accelerate* (Forsgren, Humble, Kim)
- DX Framework (Developer Experience)
- SPACE Framework (Satisfaction, Performance, Activity, Communication, Efficiency)

---

### Desafio 4.6 — Managing Large-Scale Technical Transformation

**Cenário:**
A TechNova decide migrar de monolith para microservices. Essa decisão já foi tomada pelo CTO. Você precisa **liderar a transformação** que vai afetar 350 engenheiros por 2-3 anos. Desafios:
- 60% dos engenheiros nunca trabalharam com microservices
- A arquitetura atual tem 5 anos de acoplamento
- Features não podem parar de ser entregues durante a migração
- Stakeholders querem resultados visíveis em 6 meses

**Requisitos:**
- Criar **transformation strategy** que balanceia migração com delivery de features
- Planejar **skills transformation** (como upskill 350 engenheiros)
- Gerenciar **change resistance** (não todos vão apoiar a mudança)
- Comunicar progresso de forma transparente e honesta

**Entregáveis:**

1. **Transformation Strategy:**
   - **Approach:** Strangler Fig Pattern (incremental, não big bang)
   - **Phases:**
     - Phase 1 (0-6m): Foundation — platform, tooling, first 2-3 services extracted
     - Phase 2 (6-18m): Acceleration — stream-aligned teams extract domain services
     - Phase 3 (18-30m): Optimization — performance, observability, sunsetting monolith pieces
   - **Feature delivery model:** como entregar features DURANTE a migração
   - **Decision rights:** quem decide o quê durante a transformação

2. **Skills Transformation Plan:**
   - **Assessment:** skills gap analysis da organização
   - **Training program:** workshops, dojos, pair programming, guilds
   - **Champions network:** early adopters que ajudam seus times
   - **Learning paths:** por seniority (junior vs senior paths diferentes)
   - **Metrics:** skill assessment antes e depois, certification program

3. **Change Management Plan:**
   - **Communication cadence:** town halls, newsletters, demo days
   - **Resistance handling:** como identificar, como engajar, quando escalar
   - **Quick wins:** resultados visíveis em 3-6 meses que demonstram valor
   - **Feedback loops:** como coletar e atuar em feedback da transformação
   - **Success stories:** documentar e compartilhar vitórias

4. **Governance Model:**
   - Architecture Review Board (ou não? trade-offs)
   - Decision authority matrix (quem decide decomposition boundaries?)
   - Migration sequencing (quais domínios primeiro e por quê)
   - Quality gates (quando um microservice está "ready for production"?)

**Critérios de aceite:**
- [ ] Transformation Strategy incremental (Strangler Fig, não big bang)
- [ ] Skills Transformation Plan com assessment e training
- [ ] Change Management Plan com resistance handling
- [ ] Governance Model com decision rights claros
- [ ] Reflexão: "Como liderar transformação quando eu mesmo não tenho certeza de que vai dar certo?"
- [ ] Exit strategy: o que fazer se a migração não estiver funcionando em 12 meses

**Framework de Referência:**
- Martin Fowler — *Strangler Fig Application*
- John Kotter — *Leading Change* (8-step model)
- Gene Kim — *The Phoenix Project* (transformation narrative)

---

## Entregáveis do Nível

| Artefato | Descrição |
|----------|-----------|
| Engineering Culture Document | Values codificadas com comportamentos concretos |
| Engineering Ladder | 7 níveis detalhados com competências e promotion process |
| Org Design Proposal | Team Topologies applied com transition plan |
| D&I Action Plan | Hiring, retention, progression com métricas |
| Engineering Effectiveness | DevEx survey, toil inventory, DORA metrics |
| Transformation Strategy | Large-scale migration com change management |

---

## Rubrica de Auto-Avaliação

| Competência | Iniciante (1) | Praticante (2) | Avançado (3) | Expert (4) |
|-------------|---------------|----------------|--------------|------------|
| **Culture** | Não articula culture | Values genéricos sem comportamentos | Values com anti-patterns e tensions | Culture vivida, self-reinforcing |
| **Career Ladder** | Ladder ad-hoc ou inexistente | Ladder existe mas não calibrado | Ladder com rubrica, promotion process, calibration | Career conversations contínuas, ladder evolui com a org |
| **Org Design** | Org cresce organicamente sem design | Org design reativo (reorg quando dói) | Team Topologies applied, ownership claro | Org evolui proativamente, Conway's Law consciente |
| **D&I** | Não endereça D&I | Declaração sem ação / ações pontuais | Programa estruturado com métricas | D&I embedded em todos os processos |
| **Engineering Effectiveness** | Não mede, cada time se vira | DORA metrics exist mas não atuadas | DevEx measured, toil reduced, improvements tracked | Engineering effectiveness é cultura, não projeto |
| **Transformation** | Evita transformações grandes | Tenta mas sem change management | Incremental, skills + culture + governance | Organização abraça mudança como competência core |

---

## Próximo Nível

Quando completar todos os desafios deste nível, avance para:
→ [Level 5 — Fellow: Estratégia Tecnológica](05-fellow-technology-strategy.md)

No próximo nível, sua esfera expande de **organização** para **indústria** — definindo estratégia tecnológica de longo prazo, influenciando padrões da indústria e construindo legacy que transcende a empresa.
