# Level 5 — Fellow: Thought Leadership

> **Nível anterior:** [04-distinguished-negotiation-change.md](04-distinguished-negotiation-change.md)
> **Próximo nível:** [06-capstone-communication-portfolio.md](06-capstone-communication-portfolio.md)

---

## Contexto na TechNova

A TechNova é agora uma **empresa pública** (post-IPO) com **600+ engenheiros**. Você é **Fellow** — o nível mais alto do engineering ladder, equivalente em influência ao CTO. Seu impacto vai além da TechNova: você influencia **a indústria** e é reconhecido como **thought leader** em pagamentos digitais e engenharia de software.

**A transformação do Fellow:**
Sua comunicação agora é pública, amplificada e permanente. O que você escreve em blog posts é citado, o que você diz em conferências é filmado, e suas posições técnicas influenciam outras empresas. Errare é humano, mas neste nível, erros de comunicação têm consequências na reputação da empresa e na sua pessoal.

**Novos desafios:**
- Falar para audiências de centenas/milhares de pessoas
- Publicar pensamento técnico que represente a TechNova e a indústria
- Comunicar com o Board of Directors (foco: strategy, risk, investment)
- Posicionar-se em debates técnicos da indústria (opinionated, but balanced)
- Mentorar a próxima geração de líderes em comunicação

---

## Desafio 1 — Conference Talks: Da Ideia ao Palco

### Cenário

Você foi convidado para ser keynote speaker em uma conferência de engenharia (3.000 participantes). O tema é: **"Building Payment Systems at Scale: What We Got Wrong."** Você tem 45 minutos de palco, incluindo Q&A.

Complicadores:
- A TechNova é empresa pública — tudo que você diz é escrutinado
- Legal review é necessário (não pode revelar dados financeiros ou security details)
- A audiência é mista: CTOs, Staff engineers, juniors, e jornalistas de tech
- Você NUNCA fez apresentação para mais de 100 pessoas
- O talk será gravado e publicado no YouTube

### Desafio

Criar e apresentar uma **conference talk memorável** que posicione você e a TechNova como thought leaders.

### Entregáveis

1. **Talk Design Document**
   - **Thesis**: 1 frase que resume tudo (ex: "We only learn from failures we're willing to admit")
   - **Audience analysis**: quem está na plateia, o que sabem, o que querem
   - **Story arc** (Duarte Resonate): alternando "o que é" com "o que poderia ser"
   - **3 key takeaways**: se a audiência lembrar de 3 coisas, quais são?
   - **Opening hook**: primeiros 60 segundos que capturam atenção (story, data, provocação)
   - **Closing CTA**: o que a audiência deveria FAZER diferente após ouvir
   - **Anti-patterns de keynote**: bullet-point-itis, humblebragging, hero story, no vulnerability
   - **Legal review checklist**: o que NÃO dizer (dados financeiros, security details, customer names)

2. **Talk Outline** (45 minutos)
   - **0-3 min**: Opening hook — o pior bug que já causamos (vulnerability, humor, humanity)
   - **3-12 min**: "What we built" — architecture journey, scale numbers (the setup)
   - **12-25 min**: "What we got wrong" — 3 failures com learnings (the meat)
     - Failure 1: Over-engineering (built for 10M users Day 1, had 1K)
     - Failure 2: Communication failure masked as technical failure
     - Failure 3: Not listening to the junior engineer who was right
   - **25-35 min**: "What we learned" — principles destilled from failures
   - **35-40 min**: "What's next" — open questions for the industry
   - **40-45 min**: Q&A — 5 prepared backup questions + genuine audience Q&A
   - Transitions entre seções: como manter flow narrativo

3. **Speaker Preparation Plan**
   - Practice roadmap: 8 rehearsals (mirror → friend → team → video → stage)
   - Slide design principles: max 7 words/slide, 1 idea/slide, visuals > text
   - Stage presence: where to stand, how to move, eye contact with 3K people
   - Nervousness management: techniques that work (breathing, power posing, preparation)
   - Q&A strategy: how to handle hostile questions, "I don't know", and redirects
   - Post-talk: slides sharing, blog post follow-up, social media engagement

### Critérios de Aceite

- [ ] Thesis é clara, memorable e defensível
- [ ] Story arc tem tensão dramática (não é flat presentation)
- [ ] 3 key takeaways são acionáveis (não platitudes)
- [ ] Opening hook captura atenção em 60 segundos
- [ ] Legal checklist existe e é realista para empresa pública
- [ ] Preparation plan inclui 8+ rehearsals com progression
- [ ] Anti-patterns de keynote são identificados com exemplos
- [ ] Vulnerability é genuína, não performativa

### Framework de Referência

- **Resonate** (Nancy Duarte): Story arc com "what is" vs "what could be"
- **TED Talks Guide**: 1 idea worth spreading, 18 minutes, no bullet points
- **Made to Stick** (Heath): Simple, Unexpected, Concrete, Credible, Emotional, Stories (SUCCESS)
- **Presentation Zen** (Garr Reynolds): Simplicidade, restraint, design thinking

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Conteúdo** | Informação sem narrativa | Narrativa com informação | Story que ensina | Story que transforma percepção |
| **Design** | Bullets em slides | Slides limpos | Visuais memoráveis | Slides que amplificam (não competem) |
| **Delivery** | Lê notas | Apresenta com confiança | Natural e engaged | Conexão com 3K pessoas |
| **Impact** | Aplausos educados | Perguntas no Q&A | Citado em Twitter/blogs | "A talk que mudou como penso sobre X" |

---

## Desafio 2 — Technical Writing para Publicação

### Cenário

Após a conferência, várias pessoas pediram "onde posso ler mais sobre isso?" Seu CTO sugere que você publique artigos técnicos regularmente — não como marketing, mas como genuine thought leadership que:
- Posiciona a TechNova como engineering-led company
- Atrai candidatos senior (employer branding)
- Contribui para a comunidade de engineering (give back)
- Estabelece você como autoridade no domínio

Você decide começar com um artigo profundo sobre **payment resilience patterns** para publicar no engineering blog da TechNova e potencialmente submeter para uma publicação técnica.

### Desafio

Desenvolver competência em **technical writing for publication** — escrita técnica profunda destinada a audiência ampla.

### Entregáveis

1. **Technical Article: "Payment Resilience at Scale: Patterns and Anti-Patterns"**
   - Formato: 3.000-5.000 palavras, revista técnica quality
   - Estrutura:
     - Hook: "In March 2024, we processed $0 for 47 minutes." (real or realistic)
     - Problem: why payment systems are uniquely challenging
     - Patterns: 4-5 resilience patterns com architecture diagrams, code snippets, e production data
     - Anti-patterns: 3 coisas que parecem boas ideias mas não são
     - Lessons: principles destilling patterns into decision framework
     - Call to action: o que o leitor pode aplicar amanhã
   - Style: authoritative mas acessível, com humor e humility, técnico mas não jargão-heavy
   - Review: parecido com academic peer review — 2 reviewers, incorporate feedback

2. **Content Strategy Framework**
   - Content pillars: 3-4 temas nos quais a TechNova quer ser autoridade
   - Content calendar: 1 artigo/mês, com responsáveis e temas
   - Content types: deep-dive técnico, lessons learned, how-we-built, opinion/perspective
   - Distribution: blog, Medium, LinkedIn, HackerNews, Reddit, Twitter
   - Writing process: idea → outline → draft → review → publish → promote → measure
   - Editorial guidelines: tom, style, confidentiality, legal review process
   - How to encourage others to write (most engineers DON'T want to)

3. **Writing Craft Self-Study**
   - Analysis de 3 artigos técnicos que você admira: por que funcionam?
   - Elementos de craft identificados: hook, structure, pacing, conclusion, visuals
   - Your writing voice: como você soa vs como quer soar
   - Editing checklist: 15 itens para self-review antes de publicar
   - Metric of success: views, engagement, citations, candidate mentions

### Critérios de Aceite

- [ ] Artigo tem 3.000-5.000 palavras com profundidade técnica real
- [ ] Hook captura atenção nas primeiras 2 frases
- [ ] Patterns são ilustrados com diagramas E snippets de código
- [ ] Anti-patterns são tão úteis quanto os patterns
- [ ] Content strategy é sustentável (1/mês é realista com workload)
- [ ] Editorial guidelines incluem confidentiality e legal review
- [ ] Writing craft inclui auto-análise honesta
- [ ] Artigo passou por 2 reviewers e incorporou feedback

### Framework de Referência

- **On Writing Well** (Zinsser): Princípios de escrita não-ficção
- **Technical Writing Style Guides**: Google, Microsoft, AWS
- **Content Strategy** (Halvorson): Substância, estrutura, workflow, governance
- **Write-Review-Iterate**: Nenhum bom texto nasce no primeiro draft

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Profundidade** | Superficial | Technically sound | Novel insights | Original thinking |
| **Acessibilidade** | Expert-only | Accessible to senior | Accessible to mid-level | Read by diverse audiences |
| **Voice** | Corporate | Professional | Personal + professional | Recognizable (people know it's you) |
| **Impact** | Published | Shared | Discussed | Cited and referenced |

---

## Desafio 3 — Board-Level Communication

### Cenário

O Board of Directors da TechNova (post-IPO) tem reunião trimestral. O CTO normalmente apresenta a seção de Engineering, mas pediu que você o ajude a preparar — e que você esteja presente para Q&A técnico.

O Board é composto por:
- **Maria** (Chair): ex-CEO de banco, foco em risk e compliance
- **João** (VC representative): foco em growth metrics e valuation
- **Sandra** (Independent): ex-CTO de FAANG, foco em engineering excellence
- **Roberto** (CFO observer): foco em unit economics e cost

Pauta da seção de Engineering:
1. Engineering metrics update (DORA, headcount, cost)
2. Technology strategy update (major bets, risks)
3. Security posture update (incidents, compliance status)
4. Ask: $5M additional investment in platform modernization

### Desafio

Preparar material e **comunicar-se efetivamente com Board of Directors** — a audience mais exigente e de maior impacto.

### Entregáveis

1. **Board Presentation Package**
   - **Deck** (8-10 slides, max):
     - Slide 1: TL;DR (1 slide que resume tudo — lido em 30 seconds)
     - Slides 2-3: Metrics dashboard (RAG status, trends, benchmarks vs industry)
     - Slides 4-5: Technology strategy (major bets, progress, pivots)
     - Slide 6: Security & compliance (status, incidents since last board, actions)
     - Slides 7-8: Investment proposal ($5M ask com ROI projection)
     - Slide 9: Risks & mitigations (honest, not sanitized)
     - Slide 10: Asks (what do you need from the board?)
   - **Appendix** (10-20 slides): deep-dive técnico para Q&A (não presented, referenced)
   - **Pre-read document** (2 páginas): executive narrative que conta a história por trás dos slides

2. **Board Q&A Preparation**
   - 20 perguntas antecipadas por board member (customizado por perfil):
     - Maria (risk): "What's our exposure if [major provider] has outage?"
     - João (growth): "How does engineering velocity compare to industry?"
     - Sandra (engineering): "What's your approach to technical debt?"
     - Roberto (cost): "Why is cloud cost growing faster than revenue?"
   - Para cada: resposta prepared (30 seconds), deep-dive prepared (2 minutes)
   - "Safe to say" vs "Must not say" guidelines (public company constraints)
   - How to handle: "I don't know" (with commitment to follow up)

3. **Board Communication Translation Guide**
   - Vocabulary: DORA metrics → business language
   - Conceptual: what boards care about (risk, growth, competitive advantage, efficiency)
   - Framing: engineering updates as business updates (not technology updates)
   - Time horizon: boards think in quarters and years (not sprints)
   - Benchmarking: how to compare with industry without cherry-picking
   - Granularity: what level of detail for board vs. what saves for appendix

### Critérios de Aceite

- [ ] Deck tem máximo 10 slides (appendix separado)
- [ ] TL;DR slide é lido e entendido em 30 segundos
- [ ] Pre-read é 2 páginas narrativas (não bullets)
- [ ] 20 Q&As são customizadas por perfil de board member
- [ ] Respostas têm versão 30-second e versão 2-minute
- [ ] Investment proposal tem ROI projection com assumptions claras
- [ ] Translation guide ajuda qualquer engineer a preparar board material
- [ ] Material respeita constraints de empresa pública

### Framework de Referência

- **Board-Level Communication** (HBR): Strategy, risk, governance
- **McKinsey Presentation Standards**: Pyramid structure, MECE, data-driven
- **Public Company Constraints**: What can/cannot be said (SEC guidelines)
- **Investment Proposal**: ROI, payback period, risk-adjusted return

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Concisão** | 30 slides | 15 slides | 10 slides | 10 slides onde cada um é essential |
| **Tradução** | Engineering language | Business-friendly | Business-native | Board member language |
| **Q&A** | Surprised | Prepared | Anticipated | Redirects Q&A to strengthen message |
| **Impact** | Board tolerates | Board approves | Board asks questions (engaged) | Board champions engineering investment |

---

## Desafio 4 — Open Source Communication & Community Building

### Cenário

A TechNova decidiu open-source o framework de payment abstraction layer que você ajudou a construir. O CTO quer que isso se torne um projeto "tier 1" — adotado por outras empresas, com comunidade ativa.

Sua responsabilidade como Fellow: liderar o **communication layer** do open source project:
- Project positioning e branding
- Documentation que atrai contribuidores
- Community communication (issues, discussions, governance)
- Conference talks e blog posts sobre o projeto
- Internal communication (convencer engenheiros da TechNova a contribuir)

Nome do projeto: **PayFlow** — A universal payment abstraction framework.

### Desafio

Construir a **communication infrastructure** de um open source project que atrai, retém e capacita contribuidores.

### Entregáveis

1. **Open Source Communication Package**
   - **README.md** (A+ quality):
     - Hook: 1 frase que explica o que é e por que importa
     - Quick start: de zero a "hello world" em 5 minutos
     - Architecture overview: diagram que explica o design
     - Contributing: link para CONTRIBUTING.md
     - Roadmap: para onde o projeto vai
     - Badges: CI, coverage, version, license
   - **CONTRIBUTING.md**: como contribuir (issues, PRs, code style, review process)
   - **CODE_OF_CONDUCT.md**: padrões de comportamento na comunidade
   - **GOVERNANCE.md**: como decisões são tomadas (meritocracy, consensus, BDFL?)
   - **CHANGELOG.md strategy**: como comunicar mudanças para usuários

2. **Community Communication Strategy**
   - How to handle issues: triage, labels, response time SLA, templates
   - How to review PRs: welcoming, educational, timely
   - How to say no: declining PRs/features without discouraging contributors
   - How to recognize contributors: public praise, contributor spotlight, swag
   - Communication channels: GitHub Discussions, Discord, mailing list — when to use each
   - Release communication: changelog, blog post, social media, migration guide
   - Conflict resolution: what to do when community members disagree or misbehave

3. **OSS Thought Leadership Plan**
   - Blog series: 4 posts introducing PayFlow (why we built it, architecture, adoption guide, roadmap)
   - Conference submissions: 3 CFP proposals for different conferences
   - Metrics: GitHub stars, contributors, adopters, PRs, time-to-first-contribution
   - Internal evangelism: how to convince TechNova engineers to contribute "on company time"
   - Sustainability: how to avoid OSS maintainer burnout (rotation, corporate sponsorship, governance)

### Critérios de Aceite

- [ ] README é A+ quality (quick start em 5 min, diagram, clear value prop)
- [ ] CONTRIBUTING.md torna fácil para novatos contribuírem
- [ ] Governance model é explícito e fair
- [ ] Community strategy tem response time SLAs e conflict resolution
- [ ] Thought leadership plan tem blog series + 3 CFP proposals
- [ ] Internal evangelism plan convence leadership E engineers
- [ ] Sustainability plan endereça burnout e long-term maintenance

### Framework de Referência

- **Open Source Guides** (GitHub): Starting, contributing, maintaining, leadership
- **CNCF Graduation Criteria**: O que faz um projeto OSS "maduro"
- **InnerSource Patterns**: Comunicação em projetos open-source internos
- **Maintainer's Guide to Staying Sane**: Burnout prevention em OSS

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **README** | Basics | Good | Excellent ("A+ README") | Viral (people share it) |
| **Community** | Reactive | Responsive | Welcoming | Self-sustaining |
| **Governance** | Implied | Documented | Democratic | Trusted |
| **Sustainability** | Solo maintainer | Small team | Distributed | Self-governing community |

---

## Desafio 5 — Legacy Communication: Mentoria e Succession

### Cenário

Após 8 anos na TechNova (desde os primeiros 15 engenheiros até 600+), você está pensando em seu **legacy**. Não em ego — mas em:
- O que acontece quando você não estiver mais para responder perguntas?
- Como documentar conhecimento tácito que só existe na sua cabeça?
- Como desenvolver a próxima geração de communicators e thought leaders?
- Como transferir não apenas conhecimento técnico, mas julgamento e sabedoria?

O CTO te pergunta: "Se você fosse embora amanhã, o que quebraria? Não tecnicamente — organizacionalmente."

### Desafio

Criar um **legacy de comunicação** que transcenda sua presença — mentoria, documentação de decisões, e transferência de sabedoria.

### Entregáveis

1. **Knowledge Transfer Strategy**
   - Inventário de conhecimento tácito: o que você sabe que não está documentado em lugar nenhum
   - Categorização: (a) pode ser documentado, (b) precisa ser ensinado, (c) só vem com experiência
   - Documentação dos "(a)": ADRs retroativos, architecture docs, decision frameworks, playbooks
   - Ensino dos "(b)": mentoria structured, pairing, shadowing programs
   - Experiência dos "(c)": criar situações para outros praticarem (delegate high-stakes comm)
   - Timeline: 12-month succession plan para key communication responsibilities

2. **Mentoring Communication Leaders Program**
   - Perfis de mentees: 3-4 pessoas em diferentes estágios de carreira
   - Currículo personalizado por mentee: o que cada um precisa desenvolver
   - Hands-on assignments: situações reais onde mentee lidera comunicação (com safety net)
   - Feedback format: observation → reflection → coaching → practice
   - Graduation criteria: como saber quando o mentee está ready
   - Multiplier effect: cada mentee deve mentorar 2 outros (cascade)

3. **Communication Wisdom Document**
   - O que você aprendeu sobre comunicação em 8 anos de carreira (personal essay)
   - 10 princípios que você gostaria de ter sabido no início
   - 5 erros de comunicação que mais te ensinaram (com narrativa completa)
   - "If I had to give one piece of advice...": o que diria para cada nível da carreira
   - Anti-advice: o que "todo mundo diz" sobre comunicação que você discorda
   - Reading list: os 10 recursos (livros, talks, artigos) que mais te influenciaram

### Critérios de Aceite

- [ ] Knowledge inventory identifica gaps de documentação reais
- [ ] Succession plan tem 12-month timeline com milestones concretos
- [ ] Mentoring program tem 3-4 mentees com curricula individualizados
- [ ] Hands-on assignments são situações reais (não exercícios teóricos)
- [ ] Wisdom document é genuinamente pessoal (não generic advice)
- [ ] Anti-advice desafia pelo menos 3 "verdades" aceitas sobre comunicação
- [ ] Abordou como medir se a transferência de conhecimento está funcionando

### Framework de Referência

- **Knowledge Management**: Explicit vs Tacit knowledge (Nonaka & Takeuchi)
- **Mentoring Models**: GROW (coaching), Apprenticeship (shadowing), Sponsorship (advocacy)
- **Succession Planning**: Key-person risk mitigation
- **Teaching as Learning**: Ensinar para aprender mais profundamente

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Documentação** | Na sua cabeça | Docs ad-hoc | Structured knowledge base | Living, consultable, findable |
| **Mentoria** | Responde perguntas | Sessões regulares | Currículo customizado | Mentees que mentoram |
| **Transferência** | Indispensável | Conhecimento compartilhado | Skills transferidas | Julgamento transferido |
| **Legacy** | Pessoa-dependente | Documentado | Ensinado | Auto-sustentável sem você |
