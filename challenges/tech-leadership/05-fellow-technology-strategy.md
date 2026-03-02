# Level 5 — Fellow: Estratégia Tecnológica

> **Objetivo:** Definir estratégia tecnológica de longo prazo, influenciar a indústria,
> construir parcerias estratégicas e deixar legacy que transcende a empresa atual.

**Esfera de impacto:** Indústria / Ecossistema
**Referência:** Pré-requisito para [06-capstone-leadership-portfolio.md](06-capstone-leadership-portfolio.md)

---

## Contexto — TechNova (Pós-IPO)

A TechNova fez IPO e tem 600+ engenheiros. É agora uma referência no mercado de payments. Você é **Fellow** — o IC mais senior da empresa, com acesso direto ao CTO e ocasionalmente ao board. Seu papel é garantir que a TechNova **continue relevante nos próximos 5-10 anos** e que a engenharia seja reconhecida como **world-class**. Você não tem time direto, não lidera projetos — você **define o jogo**.

---

## Desafios

### Desafio 5.1 — Technology Radar (Portfolio Tecnológico de Longo Prazo)

**Cenário:**
O CTO pergunta: *"Quais tecnologias devemos adotar, avaliar, manter e abandonar nos próximos 3 anos? Como garantimos que não ficamos presos a decisões de 5 anos atrás?"*

Hoje, decisões de tecnologia são emergentes — cada area/squad escolhe o que quer. Resultado: 4 linguagens, 6 databases, 3 message brokers, cada um com defensores apaixonados. Precisa de **curadoria estratégica**.

**Requisitos:**
- Criar **Technology Radar** da TechNova inspirado no ThoughtWorks model
- Balancear **standardization** (eficiência) com **experimentation** (inovação)
- Incluir **horizon planning** (H1: otimizar atual, H2: scale next, H3: create future)
- Definir governance: como entram e saem tecnologias do radar

**Entregáveis:**

1. **Technology Radar** com 4 rings e 4 quadrants:
   - **Rings:** Adopt | Trial | Assess | Hold
   - **Quadrants:** Languages & Frameworks | Tools | Platforms | Techniques
   - 30-40 itens posicionados com justificativa:
     - Adopt: tecnologias padrão, todos devem usar
     - Trial: usadas em 1-2 projetos controlados
     - Assess: investigar, nenhum compromisso ainda
     - Hold: parar de adotar, migrar quando possível

2. **Three Horizons Strategy:**
   | Horizonte | Timeframe | Foco | Exemplos |
   |-----------|-----------|------|----------|
   | **H1: Operate & Optimize** | 0-12 meses | Extrair valor do que temos | Performance tuning, tech debt payment |
   | **H2: Scale & Extend** | 12-36 meses | Escalar para próxima fase | Multi-region, platform-as-product |
   | **H3: Create Future** | 36-60 meses | Apostar no futuro | AI/ML, edge computing, Web3 payments |

3. **Technology Governance Model:**
   - Como propor new technology (RFC + POC + review)
   - Quem aprova (Technology Advisory Board — TAB)
   - Criteria de avaliação: ecosystem maturity, talent availability, strategic alignment
   - Exit criteria: quando mover de Trial → Adopt ou Trial → Hold
   - Sunset process: como deprecar tecnologia de forma controlada

**Critérios de aceite:**
- [ ] Technology Radar com 30+ itens nos 4 rings × 4 quadrants
- [ ] Three Horizons com exemplos concretos para TechNova
- [ ] Governance Model com processo claro de entrada e saída
- [ ] Reflexão: "Como balancear minha expertise (bias de experiência) com openness a novas tecnologias?"
- [ ] Análise de risk: quais itens no radar são apostas e quais são seguros? Como gerenciar risco?
- [ ] Stakeholder communication: como apresentar o radar para o board (non-technical audience)

**Framework de Referência:**
- ThoughtWorks — Technology Radar (modelo original)
- McKinsey — Three Horizons of Growth
- Wardley Mapping — strategic positioning of technology

---

### Desafio 5.2 — R&D Strategy & Innovation Program

**Cenário:**
O board pergunta ao CTO: *"O que nos diferencia tecnicamente? Quais são nossas vantagens competitivas de longo prazo? Outras empresas já copiaram nosso modelo — o que vem a seguir?"*

O CTO repassa para você: *"Precisamos de um programa de inovação que gere vantagem competitiva sustentável. Não é hackathon — é investimento estratégico em R&D."*

**Requisitos:**
- Distinguir **inovação sustentada** (melhoria do existente) de **inovação disruptiva** (novo paradigma)
- Criar programa de **R&D** com budget, governance e medição de resultados
- Balancear **exploração** (risco alto, retorno incerto) com **exploitation** (retorno garantido)
- Conectar R&D com **competitive advantage** real

**Entregáveis:**

1. **Innovation Portfolio Strategy:**
   | Tipo | % Investimento | Horizon | Risk | Expected Return | Exemplos |
   |------|:--------------:|:-------:|:----:|:---------------:|---------|
   | Core (melhorar o existente) | 70% | H1 | Low | Alto, previsível | Performance, UX, cost reduction |
   | Adjacent (expandir) | 20% | H2 | Medium | Moderado | Novos mercados, novos payment methods |
   | Transformational (criar futuro) | 10% | H3 | High | Incerto | AI fraud detection, blockchain settlement |

2. **R&D Program Design:**
   - **Structure:** dedicated team vs 20% time vs rotation program
   - **Governance:** como selecionar projetos (pitch + review board)
   - **Funding:** how to budget R&D (% of engineering budget)
   - **Measurement:** como medir ROI de R&D (papers, patents, prototypes, shipped features)
   - **Kill criteria:** quando cancelar um R&D project (fail fast framework)
   - **Knowledge transfer:** como learnings de R&D fluem para squads de produto

3. **Competitive Technical Moat Analysis:**
   - Quais são as **vantagens técnicas** da TechNova hoje?
   - São defensáveis? Por quanto tempo?
   - O que os competitors estão fazendo que pode ameaçar?
   - Quais investimentos em R&D criam moat mais duradouro?

**Critérios de aceite:**
- [ ] Innovation Portfolio com distribuição 70/20/10 (ou justificar outra)
- [ ] R&D Program com structure, governance e kill criteria
- [ ] Competitive Moat Analysis com pelo menos 3 vantagens atuais analisadas
- [ ] Reflexão: "Como defender investimento em R&D quando o board quer resultados trimestrais?"
- [ ] Análise de failures: exemplos de inovação que falharam e o que se aprendeu
- [ ] Success metrics para R&D que não são apenas "shipped to production"

**Framework de Referência:**
- Clayton Christensen — *The Innovator's Dilemma*
- Amazon — *Working Backwards* (R&D como mechanism)
- Google — 70/20/10 innovation model

---

### Desafio 5.3 — Industry Influence & Thought Leadership

**Cenário:**
O CTO quer que a TechNova seja conhecida como **empresa de referência em engenharia**, não apenas em payments. *"Quero que engenheiros brilhantes queiram trabalhar aqui porque sabem que vão aprender. Quero que a indústria olhe para nós como referência."*

Você precisa criar uma **strategy de thought leadership** que posicione a TechNova como employer brand técnico premium.

**Requisitos:**
- Criar programa de **engineering blog / tech talks** estruturado
- Desenvolver práticas de **open source contribution** estratégica
- Construir **presença em conferências** (speaking + sponsoring)
- Fomentar **academic partnerships** e relação com comunidade

**Entregáveis:**

1. **Engineering Brand Strategy:**
   - **Mission:** por que publicar knowledge externamente (beyond marketing)
   - **Channels:** engineering blog, conference talks, podcasts, open source
   - **Content calendar:** cadência e tipos de conteúdo
   - **Voice & tone:** como a engenharia da TechNova se posiciona
   - **Metrics:** page views, talk acceptances, GitHub stars, hire referrals via content

2. **Open Source Strategy:**
   - **Tier 1:** projetos internos que vale a pena open-sourcer (criteria)
   - **Tier 2:** contribuições estratégicas a projetos existentes (quais e por quê)
   - **Governance:** open source policy (IP, approval, maintenance commitment)
   - **Community:** como construir community ao redor dos projetos
   - **Time allocation:** quanto tempo de engineering para OSS (1-5%?)

3. **Conference & Speaking Program:**
   - CFP (Call for Proposals) coaching — como ajudar engenheiros a submeter talks
   - Speaking pipeline: identification → coaching → submission → preparation → delivery
   - Target conferences por relevância estratégica
   - Track record: meta de talks aceitas por ano

4. **Academic Partnerships:**
   - Universidades/bootcamps target
   - Collaboration models: guest lectures, research partnerships, internships
   - Recruitment pipeline: intern → junior engineer path
   - Research collaboration: problemas internos que beneficiam de pesquisa acadêmica

**Critérios de aceite:**
- [ ] Engineering Brand Strategy com mission, channels e cadência
- [ ] Open Source Strategy com governance e IP considerations
- [ ] Conference Program com CFP coaching process
- [ ] Academic Partnerships com pelo menos 3 modelos de colaboração
- [ ] Reflexão: "Como separar o que é genuíno thought leadership de marketing disfarçado?"
- [ ] Risk analysis: como lidar com publição de informação competitivamente sensível

**Framework de Referência:**
- Stripe — Engineering Blog strategy
- Netflix — Open Source Program (OSS strategy)
- Google — Academic research collaboration model

---

### Desafio 5.4 — Board-Level Technology Communication

**Cenário:**
Você é convidado a apresentar ao **Board of Directors** da TechNova sobre o estado da tecnologia e riscos técnicos. O board tem 5 membros: 2 da venture capital (financial background), 1 founder (product background), 1 independent (ex-CEO de outra tech company), 1 academic (professor de CS). Tempo: 30 minutos + 15 minutos de Q&A.

**Requisitos:**
- Traduzir **complexidade técnica** para **linguagem de negócio**
- Comunicar **riscos técnicos** sem alarmar e sem minimizar
- Propor **investimentos** com business case claro
- Antecipar **perguntas do board** e preparar respostas

**Entregáveis:**

1. **Board Presentation Outline** (30 min, 10-12 slides max):
   - Slide 1: Executive Summary (1 paragraph)
   - Slides 2-3: Technology Health Dashboard (red/yellow/green)
   - Slides 4-5: Key Wins (impacto em linguagem de business)
   - Slides 6-7: Technical Risks (probabilidade × impacto × mitigação)
   - Slides 8-9: Investment Asks (custo × ROI × timeline)
   - Slides 10-11: Technology Strategy Summary (3-year view)
   - Slide 12: Q&A topics prepped

2. **Translation Guide** (Technical → Business):
   | Technical Concept | Board Translation |
   |------------------|-------------------|
   | "Tech debt" | "Accumulated maintenance cost that slows feature delivery" |
   | "Microservices migration" | "Restructuring for faster innovation and reliability" |
   | "SLA 99.99%" | "Less than 52 minutes of downtime per year" |
   | "CI/CD pipeline" | "Automated quality checks that prevent customer-facing bugs" |
   | "Multi-region deployment" | "Serving customers from local datacenters for speed and compliance" |

3. **Q&A Preparation** — 15+ likely questions with prepared answers:
   - "Why should we invest $X in tech debt instead of new features?"
   - "What happens if we DON'T invest in [technology Y]?"
   - "How do we compare technically to [competitor Z]?"
   - "What's the biggest technical risk to IPO?"
   - "How do we know our engineers are productive?"

**Critérios de aceite:**
- [ ] Board Presentation com 10-12 slides (narrativa, não data dump)
- [ ] Translation Guide com 10+ conceitos técnicos traduzidos
- [ ] Q&A Preparation com 15+ perguntas e respostas preparadas
- [ ] Comunicação ajustada para audiência com mixed background (finance + product + tech + academic)
- [ ] Reflexão: "Qual a diferença entre comunicar com engenheiros e comunicar com board? O que me é mais difícil?"
- [ ] Risk communication: como comunicar riscos sem criar pânico nem complacência

**Framework de Referência:**
- Barbara Minto — *The Pyramid Principle* (structured communication)
- Nancy Duarte — *Resonate* (effective presentations)
- Andy Grove — *High Output Management* (communicating with executives)

---

### Desafio 5.5 — Legacy Building & Succession

**Cenário:**
Você está há 3 anos como Fellow. O CTO pergunta: *"Se você saísse amanhã, a organização continuaria executando a estratégia? Ou tudo depende de você estar aqui?"*

Você percebe que, ironicamente, quanto mais senior, maior o risco de se tornar single point of failure — não de código, mas de **knowledge, relationships e judgment**.

**Requisitos:**
- Garantir que seu **impact é sistêmico** (embedded em processos) e não **pessoal** (depende de você)
- Criar **succession pipeline** para próximos Distinguished/Fellow
- Documentar **institutional knowledge** que está apenas na sua cabeça
- Definir **transition plan** para quando (não se) você sair

**Entregáveis:**

1. **Legacy Audit:**
   - O que funciona sem você? (sistemas, processos, culture que se auto-sustentam)
   - O que depende de você? (knowledge, relationships, judgment calls)
   - O que morre sem você? (isso é o mais perigoso)
   - Action plan: para cada item que depende de você, como transferir

2. **Institutional Knowledge Documentation:**
   - **Decision history:** por que foram feitas as decisões técnicas mais importantes (ADR retrospective)
   - **Relationship map:** stakeholders, como se relacionam, o que cada um valoriza
   - **Judgment heuristics:** as "regras de bolso" que você usa para decisões rápidas
   - **Failure stories:** o que deu errado e por quê (conhecimento mais valioso)
   - **Unwritten rules:** coisas que "todo mundo sabe" mas ninguém documentou

3. **Successor Development Program:**
   - Identificar 2-3 potenciais successores (Distinguished → Fellow path)
   - Para cada: assessment, gaps, development plan, stretch assignments
   - Shadow program: successor acompanha board meetings, strategy sessions
   - Gradually transfer: delegar decisões progressivamente
   - Anti-pattern: não criar "mini-me" — successor deve trazer perspectiva diferente

4. **Transition Plan:**
   - 30-60-90 day plan para saída ordenada
   - Knowledge transfer sessions (structured, scheduled, documented)
   - Stakeholder relationship handover
   - Safety net: pós-transição, período de advisory (consultable, not accountable)

**Critérios de aceite:**
- [ ] Legacy Audit honesto com ações para cada item que depende de você
- [ ] Institutional Knowledge documentada (pelo menos 10 itens críticos)
- [ ] Successor Development com 2-3 candidatos e planos individuais
- [ ] Transition Plan de 30-60-90 dias
- [ ] Reflexão: "Meu ego está ligado ao meu papel? Como separar identidade pessoal de posição profissional?"
- [ ] Anti-pattern analysis: como evitar "indispensable syndrome" (se sentir necessário demais)

**Framework de Referência:**
- Marshall Goldsmith — *What Got You Here Won't Get You There*
- Jim Collins — *Level 5 Leadership* (leaders who build lasting organizations)
- Ray Dalio — *Principles* (systematizing decision-making)

---

## Entregáveis do Nível

| Artefato | Descrição |
|----------|-----------|
| Technology Radar | Portfolio tecnológico curado com governance |
| R&D Program | Inovação estruturada com 70/20/10 |
| Engineering Brand Strategy | Thought leadership, OSS, conferences |
| Board Presentation Package | Comunicação executiva com Q&A |
| Legacy Audit + Succession Plan | Sustentabilidade organizacional |

---

## Rubrica de Auto-Avaliação

| Competência | Iniciante (1) | Praticante (2) | Avançado (3) | Expert (4) |
|-------------|---------------|----------------|--------------|------------|
| **Technology Strategy** | Reativo, cada time escolhe | Radar existe, governance fraca | Radar ativo, three horizons, governance clara | Organização navega mudanças tecnológicas com fluidez |
| **R&D/Innovation** | Sem investimento em R&D | Hackathons esporádicos | Programa estruturado com portfolio e kill criteria | Inovação é competência core, competitive moat claro |
| **Industry Influence** | Sem presença externa | Blog posts esporádicos | Engineering brand reconhecida, OSS strategy ativa | Referência da indústria, talent magnet |
| **Executive Communication** | Não consegue traduzir tech → business | Comunica quando perguntado | Proativo, dashboard, translation natural | Board trusted advisor, influencia estratégia via dados |
| **Legacy & Succession** | Tudo depende de você | Documenta mas não transfere | Successor identified, knowledge transferida | Organização funciona sem você, isso é o maior sucesso |

---

## Próximo Nível

Quando completar todos os desafios deste nível, avance para:
→ [Level 6 — Capstone: Portfolio de Liderança](06-capstone-leadership-portfolio.md)

O capstone integra todas as competências em um cenário complexo que exige exercitar simultaneamente todas as habilidades desenvolvidas ao longo da jornada.
