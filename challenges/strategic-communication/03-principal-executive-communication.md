# Level 3 — Principal: Comunicação Executiva

> **Nível anterior:** [02-staff-influence-persuasion.md](02-staff-influence-persuasion.md)
> **Próximo nível:** [04-distinguished-negotiation-change.md](04-distinguished-negotiation-change.md)

---

## Contexto na TechNova

A TechNova levantou **Série C** e tem **150 engenheiros** em **20 squads**. Você é **Principal Engineer** da área de **Payments** (~30 engenheiros, 4 squads). Sua audiência expandiu dramaticamente: além de engenheiros, você agora comunica regularmente com **VP de Engenharia**, **CPO**, **CFO** e **CISO**.

**A mudança fundamental no Principal:**
No Senior, você comunicava para seu squad. No Staff, para squads adjacentes. Agora no Principal, sua comunicação precisa funcionar em **múltiplos registros** — do detalhe técnico mais profundo ao executive summary mais conciso — frequentemente na mesma semana, sobre o mesmo tema.

**Novos desafios de comunicação:**
- Traduzir complexidade técnica para executivos em 5 minutos
- Apresentar para C-level com dados e sem jargão
- Negociar prioridades entre engineering, product e business
- Comunicar bad news (delays, incidents, budget overruns) de forma construtiva
- Managing up: ajudar seu VP de Eng a ter sucesso comunicando para cima

---

## Desafio 1 — Executive Communication: One-Pagers e Executive Summaries

### Cenário

O CFO da TechNova convocou uma reunião com engenharia. Pauta: "O custo de infraestrutura cresceu 180% no último ano enquanto a receita cresceu 120%. Preciso entender por quê e ter um plano."

O VP de Eng te pediu para preparar o material. Você tem 15 minutos de apresentação, e o CFO toma decisões baseadas em "documents de uma página que explicam tudo o que preciso saber."

Dados disponíveis:
- Cloud cost: $120K/mês → $336K/mês em 12 meses
- 40% do custo é o novo serviço de anti-fraud (com ML) — regulatory requirement
- 25% é crescimento orgânico (mais clientes, mais transações)
- 20% é tech debt (serviços ineficientes, over-provisioning, sem auto-scaling)
- 15% é experimentação (3 POCs que não foram descontinuadas)

### Desafio

Comunicar informação técnica e financeira complexa para um **executivo C-level** de forma que gere decisão — não mais perguntas.

### Entregáveis

1. **One-Pager: Infrastructure Cost Analysis & Plan**
   - Formato: 1 página, printável, standalone (entendível sem apresentação)
   - Estrutura: Situation (fatos) → Analysis (why) → Recommendation (what to do) → Impact (results expected)
   - Visualização: 1 gráfico que conta toda a história (waterfall, stacked bar, ou treemap)
   - Action items: 3 recomendações concretas com impacto estimado em $ e timeline
   - Bottom line: "$Xk/mês de redução possível em Y meses, investindo Z"

2. **Executive Summary Writing Guide**
   - 10 regras de escrita para executivos:
     1. Lead with the ask (o que você precisa de mim?)
     2. Quantify everything (nada de "muito" ou "pouco")
     3. So what? (cada fato deve ter implicação)
     4. Options, not open questions (ofereça opções, não perguntas)
     5. One page, one decision (escopo claro)
     6-10. [Completar com suas descobertas]
   - Antes/depois: 5 exemplos de comunicação para executivos
   - Template universal para one-pagers executivos

3. **Presentation Script** (15 minutos para CFO)
   - Minuto 0-2: Headline + contexto (resposta curta à preocupação do CFO)
   - Minuto 2-5: Análise visual (o gráfico que explica)
   - Minuto 5-8: Recomendações com trade-offs
   - Minuto 8-10: Plano de ação com milestones
   - Minuto 10-15: Q&A preparado (10 perguntas + respostas)
   - "Parking lot" de detalhes técnicos (para perguntas deep-dive)

### Critérios de Aceite

- [ ] One-pager cabe em 1 página e é auto-suficiente
- [ ] Um não-engenheiro consegue entender e tomar decisão lendo o one-pager
- [ ] Visualização conta a história sem explicação verbal
- [ ] Recomendações são acionáveis com $ e timeline
- [ ] Writing guide tem 10 regras com exemplos antes/depois
- [ ] Script de apresentação respeita timebox com Q&A preparado
- [ ] Testou o one-pager com alguém não-técnico e incorporou feedback

### Framework de Referência

- **Amazon 6-Pager** (adaptado para 1-pager): Narrativa → Data → Decision
- **BLUF** (Bottom Line Up Front): Militar — conclusão primeiro, detalhes depois
- **So What? Test**: Cada frase passa no teste "e daí?"
- **Minto Pyramid**: Conclusão → Key arguments → Supporting data

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Concisão** | Documento de 5 páginas | 2 páginas | 1 página | 1 página que antecipa todas as perguntas |
| **Tradução** | Fala em terms técnicos | Simplifica termos | Fala em linguagem de negócio | CFO esquece que é engineering |
| **Visualização** | Tabela de dados | Gráfico genérico | Gráfico que conta a história | 1 imagem = decisão |
| **Decisão** | Gera mais perguntas | Gera discussão | Gera decisão | Gera decisão rápida e confiante |

---

## Desafio 2 — Comunicação de Bad News

### Cenário

Três bad news na sua semana:

**Bad News 1:** O projeto de migração para nova plataforma de pagamentos vai atrasar 6 semanas (de 8 planejadas). Motivo: subestimou complexidade de integração com 3 providers legacy. O CPO conta com essa migração para lançar em 2 novos países.

**Bad News 2:** Uma auditoria de segurança encontrou vulnerabilidade crítica (dados de cartão em log). Não há evidência de exploit, mas regulação (PCI-DSS) exige notificação ao board em 72h. CISO precisa de um briefing técnico em 24h.

**Bad News 3:** O engenheiro mais sênior do squad (Staff Engineer com 4 anos de empresa) aceitou proposta de concorrente. A equipe está demoralizadora e o PM está preocupado com o roadmap de Q4.

### Desafio

Comunicar **bad news** de forma que mantenha confiança, demonstre ownership e proponha caminho forward.

### Entregáveis

1. **Bad News Communication Framework**
   - Timing: quando comunicar (early, não perfect — "bad news doesn't age well")
   - Structure: Fato → Impacto → Causa → Ação → Ask
   - Framing: honestidade + ownership + solução (não excuses)
   - Escalonamento: quando escalar quais bad news para quem
   - Anti-patterns: minimizar, blame, esconder, dump sem proposta
   - Channel: quando bad news vai por escrito vs presencial vs ambos

2. **3 Bad News Communications** (uma para cada cenário)
   - **Atraso do projeto**: Communication para CPO (email + talking points para 1:1)
     - Reconhecer impacto no lançamento dos 2 países
     - Explicar root cause sem blame
     - Propor plano alternativo (scope reduction? parallel workstreams? interim solution?)
   - **Vulnerabilidade security**: Briefing para CISO (documento técnico + executive summary)
     - Fatos sem especulação
     - Impact assessment (blast radius)
     - Remediation plan com timeline
     - Compliance timeline (PCI-DSS 72h notification)
   - **Saída do Staff Engineer**: Communication para PM + equipe (separadas)
     - Para PM: impacto no roadmap + mitigation plan
     - Para equipe: honestidade + continuidade + valorização de quem fica

3. **"When to Escalate" Decision Matrix**
   - Categorize bad news: impacto × urgência × audiência
   - Para cada categoria: quem comunicar, quando, como, por quem
   - Escalation path: PM → VP Eng → CTO → CEO → Board (quando cada nível)
   - Template de escalation email (5 linhas: what, impact, action, ask, deadline)

### Critérios de Aceite

- [ ] Framework tem timing, structure e framing claros
- [ ] 3 communications são realistas e profissionais
- [ ] Cada communication tem fato, impacto, causa, ação e ask
- [ ] Nenhuma communication tem blame direto ou excuses
- [ ] Todas propõem caminho forward (não apenas o problema)
- [ ] Decision matrix é prática e referenciável
- [ ] Incluiu diferença entre bad news operacional vs estratégica vs pessoal

### Framework de Referência

- **SCARF** (David Rock): Status, Certainty, Autonomy, Relatedness, Fairness — o que bad news ameaça?
- **Radical Transparency** (Ray Dalio): Honestidade radical + ownership
- **Crisis Communication Principles**: Velocidade, transparência, empatia, ação

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Timing** | Espera até ser obrigado | Comunica com atraso | Comunica early | Comunica antes de ser bad news (risk flagging) |
| **Ownership** | Blame externo | Fatos sem blame | Ownership + root cause | Ownership + systemic fix |
| **Solução** | Só o problema | Problema + 1 opção | Problema + 2-3 opções | Recomendação + fallback plan |
| **Confiança** | Perde confiança | Mantém confiança | Ganha confiança (honestidade) | É buscado PORQUE dá bad news bem |

---

## Desafio 3 — Managing Up: Ajudando seu Líder a Ter Sucesso

### Cenário

Sua VP de Engenharia (Carla) é nova na empresa (3 meses) e tem dificuldade de comunicar a realidade de engenharia para o CEO. Nas reuniões de liderança, ela frequentemente é pega de surpresa por perguntas sobre a área de Payments. Resultado: o CEO forma impressões incorretas e toma decisões sem contexto técnico adequado.

Exemplos recentes:
- CEO perguntou "por que não usamos AI para detectar fraude?" — Carla não sabia que vocês já estão fazendo isso
- CEO propôs "contratar 20 engenheiros para acelerar" — Carla não tinha data para explicar por que isso não funciona (Brooks's Law)
- CEO comparou velocity de Payments com squad de Marketing — Carla não conseguiu explicar a diferença de complexidade

Carla te pediu: "Me ajude a nunca mais ser pega de surpresa."

### Desafio

Desenvolver um **sistema de managing up** que mantém seu líder informado, preparado e empoderado para representar engineering.

### Entregáveis

1. **Managing Up Playbook**
   - Princípios: "Make your boss successful" → "Engineering gets resources and support"
   - Frequência e formato de updates (weekly? bi-weekly? written? verbal?)
   - O que comunicar proativamente: status, risks, wins, asks, landscape
   - Como comunicar: executive brief format (5 bullets, 2 min para ler)
   - O que NÃO escalar (evitar "cry wolf") vs o que SEMPRE escalar
   - Como entender as prioridades e pressões do seu líder
   - Como pedir o que você precisa (budget, headcount, prioridade, ar cover)

2. **VP Engineering Weekly Brief Template**
   - Formato: máximo 1 página, leitura em 3 minutos
   - Seções: 🟢 Wins (2-3), 🟡 Risks (2-3), 🔴 Blockers (0-1), 📊 Metrics snapshot, 🎯 Asks
   - "Talking points if CEO asks...": 3 perguntas prováveis + sugestão de resposta
   - "Decisions needed from you": máximo 2 por semana, com recomendação e deadline
   - Exemplo preenchido para a semana descrita no cenário

3. **Communication Cadence Design**
   - Daily: nada (a menos que urgente — Slack DM com contexto)
   - Weekly: brief escrito (template acima) + 30 min 1:1 para discussion
   - Monthly: area review presentation (15 min, slides) com métricas e outlook
   - Quarterly: strategic update (vision, roadmap, people, risks)
   - Ad-hoc: framework para decidir quando interromper fora do cadence
   - Adaptação: como ajustar o cadence ao estilo de comunicação do líder

### Critérios de Aceite

- [ ] Playbook tem princípios claros com exemplos práticos
- [ ] Brief template é preenchível em 15 min e informativo em 3 min de leitura
- [ ] "Talking points" antecipam perguntas reais do CEO
- [ ] Cadence é prático e sustentável (não cria overhead excessivo)
- [ ] Incluiu como adaptar ao estilo de comunicação do líder (leitor? verbal? data-driven? story-driven?)
- [ ] Abordou cenário onde líder tem viewpoint diferente do seu

### Framework de Referência

- **Managing Up** (HBR): Alinhar expectativas, antecipar necessidades, adaptar ao estilo
- **Status Update Best Practices**: Red/Yellow/Green, metrics-driven, exception-based
- **RACI para Comunicação**: Quem é Responsible vs Informed em cada tema
- **One-on-One Best Practices**: Agenda compartilhada, seu 1:1 é SEU meeting

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Proatividade** | Responde quando perguntado | Updates periódicos | Antecipa necessidades do líder | Líder nunca é pego de surpresa |
| **Formato** | Dump de informação | Estruturado | Executive-ready | Customizado ao estilo do líder |
| **Context** | Fatos isolados | Fatos + implicações | Fatos + implicações + recomendação | Líder confia para representar eng |
| **Relacionamento** | Transacional | Profissional | Trusted advisor | Strategic partner |

---

## Desafio 4 — Comunicação de Visão Técnica

### Cenário

A TechNova planeja expandir para **5 novos países** nos próximos 18 meses. A área de Payments precisa de uma visão técnica de 2 anos que guie as decisões de arquitetura, contratação e investimento.

O VP de Eng te pediu para criar e **comunicar** essa visão para 3 audiências:
1. **Engenheiros da área** (30 pessoas): precisam de direção clara e motivação
2. **Liderança de produto** (PM Lead, Head of Product): precisam entender capabilities e constraints
3. **C-suite** (CEO, CTO, CFO): precisam de business case e investment thesis

A mesma visão. Três comunicações completamente diferentes.

### Desafio

Criar uma **Technical Vision** e comunicá-la efetivamente para **múltiplas audiências** com interesses, linguagem e preocupações distintas.

### Entregáveis

1. **Technical Vision Document: Payments 2025-2026**
   - Vision statement (1 frase, memorável, aspiracional)
   - Current state: onde estamos (honest assessment com dados)
   - Target state: onde queremos estar (com diagramas de arquitetura)
   - Gap analysis: o que precisa mudar (categorizado: people, process, technology)
   - Strategic bets: 3-4 grandes apostas com rationale
   - Non-goals: o que explicitamente NÃO vamos fazer (tão importante quanto goals)
   - Success metrics: como saberemos se chegamos lá (OKRs ou similar)
   - Timeline: roadmap de alto nível com milestones

2. **3 Versões da Mesma Visão**
   - **Para engenheiros** (3-5 páginas): técnico, com diagramas, frameworks, decisões de design, o que muda no dia-a-dia, oportunidades de crescimento, call-to-action
   - **Para produto** (1-2 páginas): capabilities unlocked, constraints, dependencies, timeline de entrega, trade-offs entre features e fundação
   - **Para C-suite** (1 página): investment thesis, ROI, risk/reward, competitive advantage, hiring plan

3. **Vision Communication Plan**
   - Para engenheiros: all-hands presentation (30 min) + Q&A (30 min) + follow-up doc
   - Para produto: workshop (2h) — co-criação de roadmap com prioridades mútuas
   - Para C-suite: 1-pager + 15 min presentation + Q&A preparado
   - Feedback loop: como incorporar input de cada audiência na visão
   - Refresh: quando e como atualizar a visão (quarterly review)

### Critérios de Aceite

- [ ] Vision document é claro, acionável e cabe em 5 páginas (engenheiros) ou 1 página (C-suite)
- [ ] 3 versões contam a mesma história com linguagem e profundidade diferentes
- [ ] Vision statement é memorável (cabe num tweet) e testável
- [ ] Non-goals são explícitos e fundamentados
- [ ] Communication plan é multi-channel e inclui feedback loop
- [ ] Testou versão C-suite com alguém não-técnico
- [ ] Incluiu como lidar com pushback ("muito ambicioso" ou "pouco ambicioso")

### Framework de Referência

- **Vision → Strategy → Roadmap**: Cascata de comunicação estratégica
- **Wardley Mapping**: Visualizar posição e evolução de componentes
- **OKR** (Objectives & Key Results): Fazer visão mensurável
- **North Star Framework**: Métrica única que guia decisões

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Clareza** | Vaga e genérica | Clara mas técnica | Clara para qualquer audiência | Memorável e inspiradora |
| **Adaptação** | Uma versão para todos | 2 versões (tech/business) | 3+ versões customizadas | Cada audiência sente que foi escrita para ela |
| **Execução** | Publica no wiki | Apresenta uma vez | Campaign multi-channel | Visão vira parte da cultura |
| **Feedback** | Ignora | Aceita | Solicita | Co-cria com stakeholders |

---

## Desafio 5 — Decisões Estratégicas sob Incerteza

### Cenário

Três decisões estratégicas que precisam ser tomadas na área de Payments. Nenhuma tem resposta "certa" — todas envolvem trade-offs significativos sob incerteza.

**Decisão 1 — Build vs Buy:** Um vendor oferece uma solução de payment orchestration por $200K/ano. Sua equipe pode construir algo similar em 6 meses com 3 engenheiros ($300K em salários). A solução do vendor cobre 80% do needed; custom cobre 100% mas leva 6 meses. Mercado muda rápido.

**Decisão 2 — Platform bet:** Apostar em real-time payments (PIX-like) como core capability da plataforma? Requer investimento de 8 engenheiros por 12 meses. Se funcionar, competitive moat significativo. Se não, $1.2M investidos em algo que pode não ter market demand.

**Decisão 3 — People:** Promover engenheiro interno a Staff Engineer (bom tecnicamente, comunicação fraca) ou contratar Staff externo (expensive, disruptive, mas traz perspectiva nova)? Time sente que o interno "merece". RH apoia contratação externa. Budget para apenas uma posição.

### Desafio

Estruturar **tomada de decisão** sob incerteza usando frameworks de pensamento estratégico.

### Entregáveis

1. **Decision-Making Under Uncertainty Toolkit**
   - Taxonomy de decisões: reversible vs irreversible, high-stakes vs low-stakes
   - Quando usar cada approach:
     - **High stakes + irreversible**: Full analysis, multiple stakeholders, documentation
     - **High stakes + reversible**: Fast analysis, bias toward action, define rollback
     - **Low stakes**: Delegate or decide quickly, don't overthink
   - Cognitive debiasing checklist (10 perguntas para cada decisão importante)
   - Pre-mortem protocol: "Imagine we chose X and it failed. Why?"
   - Post-mortem protocol: Revisitar decisão em 6 meses (was it right? what did we learn?)

2. **3 Decision Analyses** (uma para cada decisão)
   - **Build vs Buy**: Decision matrix com critérios ponderados + scenario analysis (best/worst/likely)
   - **Platform bet**: ROI analysis + reversibility assessment + optionality (can we start small?)
   - **People**: Stakeholder analysis + values alignment + risk assessment + succession plan
   - Cada análise: framework used, options, criteria, recommendation, dissenting view, decision log

3. **Strategic Thinking One-Pager** (meta-documento)
   - Como você pensa sobre decisões estratégicas (seu mental model)
   - First principles thinking: decompose em verdades fundamentais
   - Second-order thinking: "e depois, o que acontece?"
   - Inversion: "como garantir que isso DÊ ERRADO?"
   - Regret minimization: "aos 80 anos, do que me arrependeria?"
   - Quando parar de analisar e decidir (analysis paralysis antidote)

### Critérios de Aceite

- [ ] Toolkit categoriza decisões por stakes e reversibilidade
- [ ] 3 análises usam frameworks diferentes apropriados ao tipo de decisão
- [ ] Cada análise tem recommendation clara com rationale
- [ ] Dissenting views são incluídas genuinamente (não strawman)
- [ ] Pre-mortem identifica riscos não-óbvios
- [ ] Strategic thinking one-pager articula SEU mental model (não cópia de livro)
- [ ] Abordou decisões com informação incompleta (como decidir com 60% dos dados)

### Framework de Referência

- **Decision Matrix** (Eisenhower): Urgência × Importância
- **Pre-Mortem** (Gary Klein): Antecipação de falha
- **First Principles** (Musk/Aristotle): Decomposição em fundamentos
- **Type 1 / Type 2 Decisions** (Bezos): Irreversível (cautela) vs reversível (velocidade)
- **OODA Loop** (Boyd): Observe → Orient → Decide → Act (para decisões rápidas)

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Framework** | Gut feeling | 1 framework | Framework adequado ao tipo | Combina frameworks criativamente |
| **Análise** | Superficial | Prós e contras | Multi-critério + scenarios | Inclui second-order effects |
| **Vieses** | Ignora vieses | Consciente de vieses | Aplica debiasing | Busca dissent ativamente |
| **Velocidade** | Paralysis ou impulso | Decide em tempo razoável | Calibra velocidade ao tipo | Sabe quando 70% é suficiente |

---

## Desafio 6 — Comunicação Cross-Functional

### Cenário

O lançamento do TechNova Payments em **México e Colômbia** requer coordenação entre 6 departamentos: Engineering, Product, Legal, Compliance, Operations, e Finance. Cada departamento tem linguagem, prioridades e preocupações diferentes.

Você precisa comunicar o **mesmo plano técnico** para:
- **Legal**: "Quais dados pessoais vocês armazenam e onde?" (LGPD/privacy)
- **Compliance**: "Como garantem PCI-DSS em infra multi-região?"
- **Finance**: "Quanto custa a infra adicional e como isso afeta unit economics?"
- **Operations**: "Como fazemos deploy e rollback se algo der errado em production?"
- **Product**: "Quando posso prometer ao cliente que teremos cobertura full?"

### Desafio

Comunicar efetivamente **o mesmo conteúdo técnico** para **audiências cross-functional** com vocabulários, prioridades e concerns completamente diferentes.

### Entregáveis

1. **Cross-Functional Communication Map**
   - Para cada departamento: vocabulário, prioridades, medos, o que os motiva, como preferem receber informação
   - Translation table: termo técnico → termo do departamento
   - Example: "Latency p99" → Legal: N/A, Compliance: "SLA compliance", Finance: "cost of timeout = lost revenue", Ops: "alert threshold", Product: "user experience metric"
   - Common ground: interesses compartilhados entre todos os departamentos
   - Conflict zones: onde interesses departamentais conflitam

2. **5 Versões do Mesmo Plano** (1 pager cada)
   - Legal: foco em data flow, storage, processing, jurisdição, data retention
   - Compliance: foco em controls, certificações, audit trail, incident response
   - Finance: foco em costs, ROI, unit economics, scaling costs
   - Ops: foco em deploy, monitoring, runbooks, SLA, escalation
   - Product: foco em capabilities, timeline, limitations, feature flags
   - Cada versão: mesma empresa, mesmo projeto, linguagem completamente diferente

3. **Cross-Functional Meeting Facilitation Guide**
   - Como conduzir reunião com 6 departamentos sem caos
   - Agenda design: common ground primeiro, department-specific depois
   - Vocabulary agreement: glossário compartilhado (evitar ambiguidade)
   - Decision-making: quem decide o quê (RACI cross-functional)
   - Conflict resolution: protocol quando departamentos discordam
   - Follow-up: comunicar decisões em linguagem de cada departamento

### Critérios de Aceite

- [ ] Communication map cobre 5 departamentos com vocabulário e prioridades
- [ ] Translation table tem pelo menos 10 termos técnicos traduzidos
- [ ] 5 versões do plano são auto-suficientes (cada departamento entende a sua)
- [ ] Facilitation guide é prático para reunião de 6 departamentos
- [ ] Glossário compartilhado elimina ambiguidade nos termos mais críticos
- [ ] RACI cross-functional define claramente quem decide em cada tipo de decisão
- [ ] Abordou como lidar com "language police" (cada departamento corrigindo termos)

### Framework de Referência

- **RACI Matrix Cross-Functional**: Papéis em decisões que cruzam departamentos
- **Stakeholder Communication Plan**: Análise de audiência por departamento
- **Boundary Spanning** (Tushman): Papel do engenheiro como tradutor entre domínios
- **Common Language Framework**: Criar vocabulário compartilhado

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Tradução** | Mesma linguagem para todos | Simplifica jargão | Customiza por audiência | Cada audiência sente empatia |
| **Empatia** | Foca em eng priorities | Reconhece outras prioridades | Entende prioridades em profundidade | Antecipa concerns não expressos |
| **Conflito** | Evita | Escala para managers | Medeia com técnica | Transforma conflito em alinhamento |
| **Impacto** | Informou | Concordaram | Comprometeram-se | Viraram aliados |
