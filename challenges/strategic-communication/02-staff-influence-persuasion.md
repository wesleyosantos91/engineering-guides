# Level 2 — Staff: Influência e Persuasão

> **Nível anterior:** [01-senior-technical-communication.md](01-senior-technical-communication.md)
> **Próximo nível:** [03-principal-executive-communication.md](03-principal-executive-communication.md)

---

## Contexto na TechNova

A TechNova cresceu para **70 engenheiros** em **10 squads** após a **Série B**. Você foi promovido a **Staff Engineer** e é responsável pela área de **Payments** (3 squads: Gateway, Processing, Risk). Você não tem autoridade formal sobre outros squads, mas precisa **influenciar decisões técnicas** que afetam múltiplos times.

**O dilema do Staff:**
Você tem responsabilidade sobre resultados técnicos que dependem de pessoas que **não se reportam a você**. Sua principal ferramenta é **comunicação** — documentos persuasivos, alinhamento cross-team, facilitação de decisões e influence without authority.

**Nova dinâmica:**
- Você escreve para audiências maiores e mais diversas (engenheiros, PMs, designers, EMs)
- Suas decisões afetam 3 squads (15+ pessoas) — precisam ser bem comunicadas
- Você precisa alinhar stakeholders com objetivos conflitantes
- Seus documentos (RFCs, ADRs) se tornam referência para toda a área

---

## Desafio 1 — Escrevendo RFCs que Geram Consenso

### Cenário

Os 3 squads de Payments usam abordagens diferentes para processar pagamentos internacionais:
- Gateway: REST direto para cada provider
- Processing: Adapter pattern com interface unificada
- Risk: Micro-serviço separado por país

Isso causa bugs em 40% das transações internacionais, latência inconsistente e dificuldade de adicionar novos países. Você precisa propor uma arquitetura unificada, mas cada squad defende sua abordagem e teme o retrabalho.

### Desafio

Escrever um **RFC que persuade** stakeholders com interesses conflitantes a convergir para uma solução unificada.

### Entregáveis

1. **RFC: Payment International Processing Unification**
   - **Abstract** (2 parágrafos): o que, por que, impacto — lido em 30 segundos
   - **Context & Motivation**: dados de bugs, latência, custo de manutenção — evidence-based
   - **Current State**: diagrama de como funciona hoje (3 approaches), problemas quantificados
   - **Proposed Design**: arquitetura proposta com diagramas claros
   - **Alternatives Considered**: por que cada alternativa não é melhor (inclui status quo)
   - **Migration Strategy**: como migrar sem big-bang (progressive rollout)
   - **Risks & Mitigations**: honestidade sobre riscos com plano de mitigação
   - **Decision Record**: como vamos decidir (DACI, deadline, voting)
   - **Appendix**: detalhes técnicos que não são essenciais na primeira leitura

2. **Stakeholder Communication Plan**
   - Stakeholder mapping: quem afeta, quem é afetado, quem decide
   - Pré-trabalho: 1:1s com cada stakeholder antes de publicar RFC (build coalition)
   - Preocupações antecipadas de cada stakeholder e como endereçar
   - Timeline: quando publicar, período de comentários, quando decidir
   - Plano de comunicação da decisão (para concordantes e discordantes)

3. **RFC Writing Guide** (meta-documento)
   - O que faz um bom RFC vs um RFC ruim (anti-patterns)
   - Como equilibrar detalhamento vs concisão
   - Como gerenciar período de comentários (respond to all, summarize discussion)
   - Quando NÃO usar RFC (over-engineering do processo)
   - Lifecycle: Draft → Review → Accepted/Rejected → Superseded

### Critérios de Aceite

- [ ] RFC tem abstract que pode ser lido em 30 segundos e comunica essência
- [ ] Context é evidence-based (dados, não opiniões)
- [ ] Alternatives incluem status quo e análise honesta de prós/contras
- [ ] Stakeholder plan identifica coalition building pré-publicação
- [ ] Migration strategy é incremental (não big-bang)
- [ ] Writing guide tem anti-patterns com exemplos
- [ ] RFC pode ser entendido por um engenheiro de outro squad sem contexto prévio

### Framework de Referência

- **SCQA** (McKinsey): Situation → Complication → Question → Answer
- **Minto Pyramid**: Conclusão primeiro, depois supporting arguments
- **DACI**: Driver, Approver, Contributors, Informed
- **Coalition Building**: Alinhar aliados antes de proposta pública

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Persuasão** | Lista opções sem opinião | Recomenda com razões | Antecipa objeções e endereça | Constrói consenso antes de publicar |
| **Estrutura** | Texto corrido | Seções organizadas | Pyramid principle (conclusão primeiro) | Leitura em múltiplos níveis de detalhe |
| **Evidence** | Opiniões e feelings | Alguns dados | Dados completos e relevantes | Data-driven com visualizações |
| **Stakeholders** | Publica e espera | Informa stakeholders | Consulta stakeholders | Co-cria com stakeholders |

---

## Desafio 2 — Framing de Problemas e Propostas

### Cenário

Três situações que chegam a você na mesma semana:

**Situação A:** O CEO pergunta: "Por que nosso deploy demora 45 minutos?" Você sabe que é por causa de testes lentos, pipeline não paralelizada e approval manual, mas como comunicar sem parecer excuses?

**Situação B:** Você quer propor investimento em observability (tracing distribuído). É a coisa certa, mas o CPO vai perguntar "como isso afeta o número de features entregues?"

**Situação C:** Um engenheiro senior de outro squad te pede para explicar por que vocês estão migrando de MongoDB para PostgreSQL. Ele acha um downgrade.

### Desafio

Dominar a arte de **framing** — como a forma que você apresenta um problema determina como as pessoas pensam sobre a solução.

### Entregáveis

1. **Framing Toolkit**
   - 6 técnicas de framing com exemplos em engineering:
     - **Loss vs Gain**: "Perderemos $500K/ano" vs "Economizaremos $500K/ano"
     - **Scope framing**: Zoom in (detalhes) vs Zoom out (big picture)
     - **Analogy framing**: "É como trocar o motor do avião em voo"
     - **Time framing**: "Agora vs 6 meses" vs "esta semana vs as próximas 200 sprints"
     - **Stakeholder framing**: "Para engenharia..." vs "Para o cliente..."
     - **Opportunity cost**: "Cada dia que não fazemos isso custa X"
   - Anti-framing: como detectar quando alguém está framing você

2. **3 Propostas Reenquadradas** (uma para cada situação do cenário)
   - **Situação A**: "Deploy demora 45 min" → reframed para 3 audiências (CEO, VP Eng, equipe)
   - **Situação B**: "Investimento em observability" → reframed como investimento de negócio, não tech debt
   - **Situação C**: "MongoDB → PostgreSQL" → reframed como evolução de necessidades, não downgrade
   - Cada proposta: versão antes do reframing → versão after → análise do que mudou

3. **Problem Statement Workshop Format**
   - Como facilitar uma sessão de framing de problema com equipe
   - Template: "How might we..." (Design Thinking)
   - Exercício: mesmo problema, 5 framings diferentes, impacto na solução
   - Output: problem statement claro, conciso e alinhado

### Critérios de Aceite

- [ ] Toolkit tem 6 técnicas com exemplos específicos de engineering
- [ ] Cada técnica tem cenário "antes do framing" e "depois do framing"
- [ ] 3 propostas demonstram claramente o poder do reframing
- [ ] Anti-framing tem exemplos de como você pode ser manipulado
- [ ] Workshop format é facilitável em 60 min com 5-8 pessoas
- [ ] Cada proposta reenquadrada é honesta (framing ≠ manipulação)

### Framework de Referência

- **Reframing** (Tversky & Kahneman): Mesma informação, percepção diferente
- **How Might We** (IDEO): Reframe problema como oportunidade
- **Issue Trees** (McKinsey): Decompor problema em sub-problemas
- **MECE** (Mutually Exclusive, Collectively Exhaustive): Categorização sem gaps nem sobreposições

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Consciência** | Não sabe que frameia | Reconhece framing | Escolhe framing deliberadamente | Adapta framing por audiência |
| **Técnica** | Um framing por problema | 2 framings | 3+ framings com trade-offs | Framing customizado para contexto |
| **Honestidade** | Manipulação | Omissão seletiva | Transparência com framing | Explicita o framing escolhido |
| **Impacto** | Nenhum | Gera discussão | Gera alinhamento | Gera ação |

---

## Desafio 3 — Apresentações Técnicas Internas

### Cenário

Você precisa fazer 3 apresentações nas próximas 2 semanas:
1. **Tech Talk** (30 min, 40 engenheiros): Lições aprendidas na migração do payment gateway
2. **Area Review** (15 min, VP Eng + EMs): Status e riscos da área de Payments Q3
3. **Design Review** (45 min, 8 engenheiros): Nova arquitetura de retry com event sourcing

Cada apresentação requer uma abordagem completamente diferente. Você historicamente usa o mesmo estilo para todas e percebe que funciona bem só na tech talk.

### Desafio

Desenvolver competência em **apresentações adaptadas à audiência** e criar templates reutilizáveis.

### Entregáveis

1. **Presentation Adaptation Matrix**
   - Para cada tipo (tech talk, area review, design review):
     - Audiência: quem, o que sabem, o que precisam, o que temem
     - Objetivo: informar, persuadir, decidir, ensinar
     - Estrutura: abertura, corpo, fechamento, Q&A
     - Visuais: tipo de slides, data visualization, live demo
     - Tom: storytelling, executive, collaborative
     - Timing: quanto por seção, where to spend most time
     - Anti-pattern mais comum deste tipo

2. **3 Apresentações Preparadas** (outlines ou slides simplificados)
   - **Tech Talk**: Story arc (problema → jornada → lições), com hooks e takeaways
   - **Area Review**: Executive format (status → risks → asks → timeline), data-heavy
   - **Design Review**: Interactive (present → pause → discuss → iterate), com diagrams

3. **Presenter Self-Development Plan**
   - Auto-avaliação honesta: pontos fortes e fracos como apresentador
   - Gravação de 1 apresentação (pode ser para espelho) com auto-feedback
   - 5 técnicas de melhoria priorizadas com plano de prática
   - Gestão de nervosismo: técnicas que funcionam para você

### Critérios de Aceite

- [ ] Matrix cobre pelo menos 3 tipos de apresentação com variáveis claras
- [ ] Cada apresentação preparada tem estrutura, timing e visuais definidos
- [ ] Tech talk tem story arc identificável com hook e call-to-action
- [ ] Area review começa com bottom line (conclusão primeiro)
- [ ] Design review tem pontos de pausa para discussão
- [ ] Self-development plan é honesto e tem ações implementáveis

### Framework de Referência

- **Presentation Zen** (Reynolds): Simplicidade, restraint, naturalidade
- **Resonate** (Duarte): Story arc alternando entre "o que é" e "o que poderia ser"
- **Situation-Task-Action-Result** (STAR): Para narrativas de projetos
- **Bottom Line Up Front** (BLUF): Conclusão primeiro em contexto executive

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Adaptação** | Mesmo estilo sempre | Ajusta conteúdo | Ajusta estilo e conteúdo | Customiza por audiência e contexto |
| **Estrutura** | Stream of consciousness | Tem início/meio/fim | Story arc com tensão e resolução | Audiência engajada do início ao fim |
| **Visuais** | Slide de bullets | Slides limpos | Visuais que amplificam mensagem | Visuais que a audiência lembra |
| **Q&A** | Pego de surpresa | Responde adequadamente | Antecipa perguntas | Usa Q&A para reforçar mensagem |

---

## Desafio 4 — Influence Without Authority

### Cenário

Você identificou que 3 squads têm implementações diferentes de circuit breaker: Hystrix (deprecated), Resilience4j, e implementação própria. Isso causa inconsistência, dificuldade de troubleshooting e duplicação de esforço. Você quer convergir para uma solução única.

O problema: você NÃO é manager de nenhum desses squads. Os tech leads são:
- **Felipe** (Gateway): "Hystrix funciona para nós, não quero mexer"
- **Marina** (Processing): "Nosso custom é mais performante, tenho benchmarks"
- **Rafael** (Risk): "Resilience4j é o padrão da indústria, os outros deviam adotar"

Cada um tem razões válidas. Ninguém quer ceder. Nenhum gerente vai "mandar" a unificação.

### Desafio

Desenvolver e executar uma **estratégia de influência** para alinhar equipes sem autoridade formal.

### Entregáveis

1. **Stakeholder Influence Map**
   - Mapeamento de todos os envolvidos: posição, influência, interesse, posição provável
   - Power/Interest grid: quem gerenciar de perto, quem manter informado
   - Influence network: quem influencia quem (ex: Marina respeita a opinião do VP Eng)
   - Coalition plan: quem converter primeiro para criar momentum
   - Resistência map: quem vai resistir, por que, como endereçar

2. **Influence Playbook** (6 estratégias)
   - **Reciprocity**: "Eu ajudo você com X, preciso da sua abertura para Y"
   - **Social proof**: "Times A e B já adotaram" (mas sem peer pressure)
   - **Authority**: Trazer dados de mercado, SRE best practices, benchmarks
   - **Consistency**: Conectar com princípios que a pessoa já expressou
   - **Common ground**: Encontrar o que todos concordam como ponto de partida
   - **Pilot/prototype**: Propor teste pequeno antes de comprometimento total
   - Para cada: quando usar, quando NÃO usar, exemplo no cenário circuit breaker

3. **Campaign Plan: Circuit Breaker Unification**
   - Week 1-2: Discovery (entender cada perspectiva genuinamente)
   - Week 3-4: Coalition building (1:1s com aliados naturais)
   - Week 5: Shared analysis (document técnico com comparação honesta — incluindo benchmarks de Marina)
   - Week 6: Proposal com opção de pilot (não pedir commitment total)
   - Week 7-8: Pilot execution com early adopter
   - Week 9: Results sharing e decisão final
   - Plan B: o que fazer se não conseguir consenso (escalation? accept diversity?)

### Critérios de Aceite

- [ ] Influence map identifica todos os stakeholders com posição e motivação
- [ ] Playbook tem 6+ estratégias com exemplos concretos do cenário
- [ ] Campaign plan é faseado com milestones e checkpoints
- [ ] Incluiu genuíno respeito pelas posições dos outros (não é manipulação)
- [ ] Plan B é pragmático (nem sempre unificação é a resposta)
- [ ] Abordou ética de influência (persuasão vs manipulação)

### Framework de Referência

- **6 Princípios de Influência** (Cialdini): Reciprocity, Commitment, Social Proof, Authority, Liking, Scarcity
- **Power/Interest Grid**: Classificação de stakeholders
- **Kotter's Change Model**: 8 etapas para mudança
- **Disagree and Commit** (Amazon): Quando parar de discutir e executar

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Mapeamento** | Não mapeia stakeholders | Lista stakeholders | Mapeia posições e motivações | Mapeia rede de influência |
| **Estratégia** | Vai direto ao pedido | Prepara argumentos | Adapta abordagem por pessoa | Cria condições antes de propor |
| **Execução** | Uma conversa | Série de conversas | Campaign faseada | Movimento orgânico |
| **Ética** | Ignora | Evita manipulação | Transparente sobre intenção | Genuinamente busca best outcome |

---

## Desafio 5 — Facilitação de Decisões Cross-Team

### Cenário

A TechNova precisa decidir sobre a estratégia de caching para a plataforma. Multiple squads são impactados, e a decisão envolve trade-offs significativos:

| Opção | Prós | Contras | Quem defende |
|-------|------|---------|-------------|
| Redis centralizado | Simples, consistente | Single point of failure, custo | Squad Platform |
| Cache por serviço | Isolamento, performance | Invalidação complexa, inconsistência | Squads de domínio |
| CDN + edge caching | Performance extrema | Limitado a read-heavy, complexo | Squad Frontend |

O VP de Eng te pediu para **facilitar** a decisão (não para decidir). São 15 pessoas de 5 squads, com opiniões fortes e interesses diferentes.

### Desafio

Facilitar um **processo de decisão cross-team** que gere commitment independente do resultado.

### Entregáveis

1. **Decision Facilitation Framework**
   - Antes: preparação (define scope, success criteria, decision-making method)
   - Durante: estrutura da sessão (diverge → converge → decide)
   - Depois: comunicação, registro, follow-up
   - Roles: Facilitator (você), Decision-maker (VP Eng), Contributors, Informed
   - Ground rules: desacordo é esperado, dados > opiniões, decisão será respeitada
   - Técnicas anti-groupthink: anonymous voting, devil's advocate, pre-mortem

2. **Decision Session Design** (para o caso caching)
   - Pre-work: cada squad prepara 1-pager com perspectiva
   - Session 1 (60 min): Present (15 min/opção) → Clarify questions
   - Between sessions: Escrita de concerns e questions em documento compartilhado
   - Session 2 (90 min): Address concerns → Evaluation criteria → Score → Decide
   - Post-decision: Communication template para anunciar
   - Total: 4h investidas para decisão que afeta 15+ pessoas por meses

3. **Decision Communication Package**
   - Announcement template: decisão, rationale, impact, timeline, FAQ
   - Para quem concordou: "Obrigado, expectativas e next steps"
   - Para quem discordou: "Sua perspectiva foi ouvida, aqui está por quê decidimos diferente"
   - ADR registro: para referência futura (por que decidimos assim em [data])
   - Follow-up: como revisitar a decisão se premissas mudarem

### Critérios de Aceite

- [ ] Framework cobre antes/durante/depois com roles definidos
- [ ] Session design é prático e respeita o tempo (não fica em decision limbo)
- [ ] Inclui pré-trabalho assíncrono para otimizar tempo síncrono
- [ ] Communication package endereça tanto concordantes quanto discordantes
- [ ] ADR registra context, decision, consequences, status
- [ ] Abordou "disagree and commit" — como gerar commitment após decisão

### Framework de Referência

- **RAPID** (Bain): Recommend, Agree, Perform, Input, Decide — quem faz o quê
- **Disagree and Commit** (Amazon LP): Commit even if you disagree
- **Gradients of Agreement** (Kaner): 8 níveis de concordância (endorse → block)
- **Consent-based Decision Making**: Não precisa de concordância, precisa de não-objeção

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Preparação** | Marca meeting | Define agenda | Pre-work + criteria | Framework completo de decisão |
| **Facilitação** | Opina e participa | Neutro mas passivo | Neutro e ativo (dirige) | Neutro, ativo e adaptável |
| **Inclusão** | HiPPO decide | Votação simples | Todos contribuem | Vozes minoritárias são buscadas |
| **Commitment** | Decisão ignorada | Decisão aceita | Decisão comunicada | Decisão com commitment genuíno |

---

## Desafio 6 — Escrita Persuasiva para Investimento Técnico

### Cenário

O CPO (Chief Product Officer) anunciou que Q4 é "product velocity quarter" — zero tolerance para tech investment que não seja diretamente ligado a features. Porém, você identificou que:
- O sistema de pagamentos não tem idempotency keys → cobranças duplicadas (2-3/semana)
- A falta de circuit breakers causa cascading failures quando um provider cai
- Sem distributed tracing, debugging leva 3x mais tempo do que deveria

Cada um desses problemas **impacta** product velocity, mas você precisa **escrever** isso de forma que o CPO entenda.

### Desafio

Escrever **propostas de investimento técnico** que "vendem" engineering work para stakeholders de negócio.

### Entregáveis

1. **Technical Investment One-Pager Template**
   - Header: título, autor, data, decisão solicitada
   - Executive summary: 3 linhas com impacto de negócio (não técnico)
   - Problem: o que está acontecendo (com dados e examples)
   - Impact: custo de não fazer ($, horas, risco, customer impact)
   - Proposal: o que fazer (em plain language), investimento necessário
   - Expected outcomes: métricas mensuráveis de sucesso
   - Risks of not acting: cenário se não investir
   - Timeline: quando começar, quando terminar, checkpoints
   - Design: cabe em 1 página (frente) + 1 página técnica (verso, opcional)

2. **3 One-Pagers Escritos** (uma para cada problema do cenário)
   - Idempotency keys: foco em customer trust e revenue
   - Circuit breakers: foco em reliability e SLA compliance
   - Distributed tracing: foco em developer productivity e MTTR
   - Cada um: versão "nerd" (antes) → versão "business" (depois)

3. **Persuasion Techniques for Technical Writing**
   - Como traduzir "tech debt" para "business risk"
   - Como usar números efetivamente (avoid: "improve performance", use: "reduce p99 from 2s to 200ms, impacting 5K daily users")
   - Como criar urgência sem alarmismo
   - Como ancorar expectativas ("pedir 4 sprints para negociar 2")
   - Common objections e como responder por escrito
   - Gallery de "subject lines" que fazem CPO abrir seu email

### Critérios de Aceite

- [ ] Template cabe em 1 página (frente) com todos os elementos essenciais
- [ ] 3 one-pagers são auto-contidos e persuasivos para audiência não-técnica
- [ ] Cada one-pager tem antes/depois mostrando transformação da linguagem
- [ ] Impact é sempre quantificado ($, horas, %, clientes afetados)
- [ ] Persuasion techniques são éticas (transparência + honestidade)
- [ ] Subject lines são testáveis e não clickbait
- [ ] Incluiu como responder a "mas features são mais importantes" por escrito

### Framework de Referência

- **Amazon 6-Pager**: Narrativa estruturada para decisão
- **Pyramid Principle** (Minto): BLUF (Bottom Line Up Front)
- **ROI Communication**: Return on Investment em linguagem de negócio
- **Loss Aversion** (Kahneman): Perdas pesam mais que ganhos equivalentes

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Linguagem** | Fala em tech para business | Mistura tech e business | Business com tech como suporte | Audiência esquece que é tech |
| **Quantificação** | "Vai melhorar" | Algumas métricas | ROI completo | Cost of delay quantificado |
| **Concisão** | 5 páginas | 2 páginas | 1 página | 1 página que gera ação |
| **Persuasão** | Informa | Argumenta | Convence | CPO pede mais investimento tech |
