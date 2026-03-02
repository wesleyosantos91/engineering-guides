# Level 1 — Senior: Comunicação Técnica Eficaz

> **Nível anterior:** [00-foundations.md](00-foundations.md)
> **Próximo nível:** [02-staff-influence-persuasion.md](02-staff-influence-persuasion.md)

---

## Contexto na TechNova

A TechNova acaba de fechar a **Série A** e cresceu para **25 engenheiros** em **4 squads**. Você é **Senior Engineer** no squad de **Payments**, responsável pelo processamento de pagamentos em tempo real. Seu squad tem 5 pessoas: você, 2 juniors (Ana e Bruno), 1 pleno (Carlos) e o tech lead (Diana).

O crescimento traz novos desafios de comunicação: mais gente → mais contexto perdido → mais retrabalho. Diana te pediu para ajudar a elevar a barra de comunicação técnica do squad, começando pelo seu próprio exemplo.

**Seu escopo de comunicação como Senior:**
- Audiência primária: seu squad + stakeholders diretos (PM, designer)
- Canais: PRs, documentação, Slack, dailies, design discussions, retros
- Decisões: trade-offs técnicos no escopo do squad

---

## Desafio 1 — Pull Request como Ferramenta de Comunicação

### Cenário

Revisando os PRs do squad, você encontra padrões problemáticos:
- PRs com título "fix stuff" e descrição vazia (Bruno)
- PRs com 47 arquivos e 2000 linhas (Carlos — "é tudo relacionado")
- PRs que ficam 5 dias em review porque ninguém entende o contexto (você, às vezes)
- Ana faz PRs bem descritos mas não recebe feedback — o squad dá approve sem comentar

Diana comenta: "Nossos PRs não são comunicação — são dumps de código."

### Desafio

Transformar Pull Requests em **artefatos de comunicação de alta qualidade** que minimizem back-and-forth e sirvam como documentação viva.

### Entregáveis

1. **PR Description Template**
   - Seções obrigatórias: What (o que mudou), Why (por que mudou), How (decisões técnicas), Testing (como testar), Risks (o que pode quebrar)
   - Seções opcionais: Screenshots, Performance impact, Migration notes
   - Guidelines de tamanho (alvo: <400 linhas, quando quebrar)
   - Checklist de self-review antes de abrir PR

2. **PR Communication Playbook**
   - Como escrever títulos que contam uma história (padrão: `type(scope): imperative description`)
   - Como usar commit messages como narrativa (conventional commits + body significativo)
   - Como responder a comentários de review (agreement, pushback, discussion)
   - Como fazer inline comments no próprio PR para guiar o reviewer
   - Guia de quando e como usar draft PRs

3. **3 PRs Exemplares** (reais ou simulados)
   - PR pequeno (bug fix): descrição concisa com root cause analysis
   - PR médio (feature): descrição com contexto de negócio, decisões técnicas, alternativas consideradas
   - PR grande (refactor): descrição com motivação, approach, risks, rollback plan, links para ADR/RFC

### Critérios de Aceite

- [ ] Template é prático e pode ser copiado/utilizado imediatamente
- [ ] Playbook cobre títulos, commits, review responses e inline comments
- [ ] Os 3 PRs exemplares demostram diferentes níveis de complexidade
- [ ] Cada PR exemplar teria zero perguntas de contexto do reviewer
- [ ] Guidelines abordam como lidar com PRs grandes (stacked PRs, feature flags)
- [ ] Incluiu anti-patterns de PR com exemplos de antes/depois

### Framework de Referência

- **Conventional Commits**: feat, fix, refactor, docs, test, chore
- **Ship/Show/Ask**: Quando to merge direto, quando mostrar, quando pedir review
- **Stacked PRs**: Técnica para quebrar mudanças grandes em PRs incrementais

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Descrição** | Vazia ou uma linha | Template preenchido | Narrativa que guia | Reviewer não precisa perguntar nada |
| **Contexto** | Só o que (what) | What + why | What + why + alternatives | Story completa: why, what, how, risks |
| **Tamanho** | PRs gigantes | <500 linhas | <300 linhas, coesas | Stacked PRs quando necessário |
| **Review** | Ignora comentários | Responde OK/done | Discute substantivamente | Transforma review em learning |

---

## Desafio 2 — Documentação que as Pessoas Leem

### Cenário

O squad Payments tem um problema existencial com documentação:
- A wiki do Confluence tem 150 páginas, 80% desatualizadas
- O README do repo tem 3 linhas ("run npm install")
- Quando um novo engenheiro entra, gasta 2 semanas "descobrindo" como as coisas funcionam
- O único "documento vivo" é o código — e nem sempre é auto-explicativo

Diana pede que você lidere a reorganização da documentação do squad.

### Desafio

Criar uma **estratégia de documentação** que maximize valor e minimize overhead de manutenção.

### Entregáveis

1. **Documentation Strategy** (1 página)
   - Filosofia: o que documentar e o que não documentar
   - Tipos de doc: tutorials, how-to guides, reference, explanation (Diátaxis framework)
   - Onde vive cada tipo (README, wiki, código, ADR, runbook)
   - Quem mantém e quando atualizar (ownership model)
   - Definition of Done que inclui documentação

2. **Living Documentation Templates**
   - README do repositório (run, develop, test, deploy, architecture overview)
   - ADR template (context, decision, consequences, status)
   - Runbook template (when to use, steps, troubleshooting, escalation)
   - Onboarding guide (day 1, week 1, month 1 — learning journey)
   - API documentation guidelines (endpoints, contracts, examples, errors)

3. **Documentation Audit do Squad Payments**
   - Inventário do que existe vs. o que deveria existir
   - Classificar cada doc: accurate, outdated, missing, redundant
   - Plano de ação com prioridades (quick wins primeiro)
   - Métricas de sucesso: tempo de onboarding, perguntas repetidas, incidents causados por doc ruim

### Critérios de Aceite

- [ ] Strategy cabe em 1 página e é seguível
- [ ] Aplica Diátaxis framework com exemplos concretos
- [ ] Templates são copiáveis e práticos
- [ ] Audit tem inventário categorizado com priorização
- [ ] Incluiu critério claro de "quando é o suficiente" (evitar over-documenting)
- [ ] Abordou como manter docs atualizadas (docs-as-code, automated tests, review)

### Framework de Referência

- **Diátaxis** (Daniele Procida): Tutorials × How-to × Reference × Explanation
- **Docs as Code**: Documentação no mesmo repo, mesmo review process, mesmo CI
- **ADR** (Architecture Decision Records): Michael Nygard

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Cobertura** | Sem documentação | README básico | Diátaxis completo | Docs auto-atualizáves |
| **Clareza** | Confusa ou incompleta | Tecnicamente correta | Clara para target audience | Clara para qualquer leitor |
| **Manutenção** | Escreve e esquece | Atualiza quando lembra | Atualiza no PR | Testes automatizados de docs |
| **Valor** | Ninguém lê | Consultada ocasionalmente | Primeiro recurso para dúvidas | Reduz perguntas em 80% |

---

## Desafio 3 — Post-Mortems e Comunicação de Incidentes

### Cenário

Sexta-feira, 17h30. O gateway de pagamentos para de processar transações. O squad descobre depois de 47 minutos (via alerta do cliente, não do monitoramento). O fix leva mais 23 minutos. Impacto: $43K em transações perdidas, 2.300 clientes afetados.

Diana conduz o post-mortem na segunda-feira, mas o resultado é um documento burocrático que culpa o "deploy de sexta" e lista 12 action items que nunca serão completados. Pior: o CEO recebe um email sobre o incidente que começa com "O pod do Kubernetes crashou por causa de um OOM kill no sidecar do Envoy..."

### Desafio

Dominar a **comunicação de incidentes** em dois eixos: blameless post-mortem (interno) e incident communication (stakeholders).

### Entregáveis

1. **Blameless Post-Mortem Template**
   - Incident summary (5W: what, when, where, who, impact)
   - Timeline (minute by minute: detect, respond, mitigate, resolve)
   - Root cause analysis (5 Whys ou Fishbone diagram)
   - Contributing factors (technical, process, organizational)
   - Action items com owner, deadline, priority (máx 5 — focados)
   - Lessons learned (o que funcionou bem + o que melhorar)
   - Linguagem: "o sistema falhou" vs "João quebrou" — guia de framing blameless

2. **Incident Communication Templates** (3 audiências)
   - Para engenheiros: técnico, detalhado, com logs e diagramas
   - Para gestão (VP, CTO): impacto de negócio, root cause em plain language, ações
   - Para clientes (se necessário): o que aconteceu, impacto, compensação, prevenção
   - Cada template: linguagem adequada, nível de detalhe, tom, o que NÃO dizer

3. **Incident Narrative** (caso Payments Gateway)
   - Escrever o post-mortem completo do incidente descrito no cenário
   - Escrever o email para CEO (máx 5 parágrafos, plain language)
   - Escrever o communication para clientes (se a TechNova decidir comunicar)
   - Comparar os 3 textos: como mesma informação é adaptada para audiências diferentes

### Critérios de Aceite

- [ ] Post-mortem template é genuinamente blameless (linguagem de sistema, não de pessoa)
- [ ] Timeline é granular (minute-by-minute) com decisions e actions
- [ ] Action items são máximo 5, cada um com owner e deadline
- [ ] 3 communication templates demonstram adaptação de audiência clara
- [ ] Incident narrative do caso Payments é completa e profissional
- [ ] Incluiu "red flags" de post-mortems ruins (blame, vagos, 20 action items)

### Framework de Referência

- **Blameless Post-Mortem** (Google SRE): Cultura de aprendizado, não de culpa
- **5 Whys** (Toyota): Investigar causa raiz em cascata
- **Incident Severity Levels**: SEV1-SEV4 com critérios claros
- **Communication Protocol**: Quem comunica o quê, quando, para quem

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Blameless** | Culpa pessoas | Linguagem neutra | Foco em sistema | Cria ambiente seguro para verdade |
| **Root Cause** | Symptom treatment | 1 nível de why | 5 whys completo | Contributing factors sistêmicos |
| **Communication** | Mesmo texto para todos | Ajusta nível técnico | Adapta completamente por audiência | Proativo e empático |
| **Action Items** | Lista infinita genérica | Priorizados | Priorizados com owner + deadline | Track até conclusão |

---

## Desafio 4 — Comunicação Assíncrona Eficaz

### Cenário

Com a TechNova crescendo, o squad Payments agora tem membros em **3 fusos horários** (São Paulo, Lisboa, Austin). As meetings se tornaram difíceis de agendar e a maioria das discussões técnicas acontece de forma assíncrona no Slack/GitHub.

Problemas emergentes:
- Threads de Slack de 200+ mensagens onde a decisão se perde
- Alguém faz uma pergunta em São Paulo, Austin responde 6h depois, Lisboa clarifica 3h depois — e a conversa se arrastou 24h para algo que levaria 15 min síncrono
- Contexto importante perdido em DMs que deveriam ser em canais públicos
- Reuniões marcadas "só para sincronizar" — que poderiam ser documentos

### Desafio

Projetar um **sistema de comunicação assíncrona** para equipes distribuídas em engenharia.

### Entregáveis

1. **Async Communication Handbook**
   - Princípios de async-first communication (por que padrão deveria ser async)
   - Decision tree: quando síncrono vs assíncrono
   - Como escrever mensagens que não geram back-and-forth ("front-load context")
   - Formato: Contexto → Problema → Opções → Pergunta → Deadline para resposta
   - Working agreements: response time expectation, blocked protocols, escalation

2. **Channel Architecture**
   - Proposta de organização de canais (Slack/Teams) para o squad e inter-squads
   - Naming conventions (#squad-payments, #incidents, #rfc, #random)
   - O que vai em cada canal vs. em cada ferramenta (Slack vs GitHub vs Wiki vs Email)
   - Thread hygiene: quando usar threads, quando nova mensagem, quando doc
   - Archival policy: como não perder contexto importante em Slack

3. **Async Decision-Making Protocol**
   - Template para propor decisão assíncrona
   - Voting/commenting protocol (thumbs up? formal "approve"? silent = agree?)
   - Timeboxing: prazo para input, o que acontece se ninguém responde
   - Escalation: quando converter async → sync (meeting)
   - Como registrar a decisão final e comunicar para ausentes

### Critérios de Aceite

- [ ] Handbook tem princípios claros com exemplos de antes/depois
- [ ] Decision tree é prático (pode ser poster na wiki)
- [ ] Channel architecture cobre squad + inter-squad + incidents
- [ ] Protocol de decisão async tem timebox, voting e escalation
- [ ] Working agreements são negociáveis e razoáveis (não autoritários)
- [ ] Abordou o desafio de fusos horários explicitamente

### Framework de Referência

- **Async-First Communication** (GitLab Handbook): Princípios de remote-first
- **DACI** (Driver, Approver, Contributor, Informed): Papéis em decisão
- **Working Agreements**: Contrato social aceito pela equipe

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Contexto** | Manda "oi, está aí?" | Manda pergunta direta | Front-loads todo contexto | Antecipa follow-up questions |
| **Canal** | DM para tudo | Canal público | Canal certo para conteúdo certo | Organiza informação para futuro |
| **Decisão** | Espera meeting | Propõe em thread | Propõe com deadline e opções | Facilita decisão sem meeting |
| **Documentação** | Contexto morre no Slack | Copia para wiki | Escreve na fonte de verdade | Sistema onde Slack é efêmero, docs são permanentes |

---

## Desafio 5 — Design Discussions e Comunicação de Trade-offs

### Cenário

O squad Payments precisa decidir entre 3 abordagens para implementar retry de pagamentos falhados:
- **Opção A**: Retry síncrono com exponential backoff (simples, acoplado)
- **Opção B**: Fila de retry com dead letter queue (desacoplado, complexo)
- **Opção C**: Saga pattern com compensação (resiliente, muito complexo)

A discussão na meeting durou 90 minutos sem conclusão. Carlos defende A ("KISS"), você e Ana preferem B ("escalável"), Bruno googou C e acha que é "o certo". Diana encerra com "vamos pensar mais e voltar na próxima sprint."

### Desafio

Transformar **design discussions improdutivas** em processos de decisão estruturados com comunicação clara de trade-offs.

### Entregáveis

1. **Design Discussion Facilitation Guide**
   - Como preparar uma design discussion (pre-read obrigatório)
   - Formato: Present → Clarify → Challenge → Decide
   - Técnicas de facilitação: timeboxing, silent brainstorming, dot voting
   - Como prevenir HiPPO (Highest-Paid Person's Opinion)
   - Como lidar com impasses (decision-maker protocol, time-bound default)

2. **Trade-off Communication Template**
   - Comparison matrix template: critérios × opções × scores
   - Como escolher critérios (must-have vs nice-to-have, pesos)
   - Como comunicar trade-offs visualmente (radar chart, comparison table)
   - Exemplo completo: caso retry de pagamentos com 3 opções avaliadas
   - "Recommendation + Rationale" vs "Options deck" — quando usar cada um

3. **Mini-RFC: Payment Retry Strategy** (caso prático)
   - Context: por que estamos decidindo isso agora
   - Problem: o que estamos resolvendo (com dados/evidências)
   - Options: 3 opções com prós, contras, e riscos de cada uma
   - Recommendation: opção preferida com rationale claro
   - Decision: como decidimos, quem decide, deadline
   - Consequences: o que muda após a decisão

### Critérios de Aceite

- [ ] Facilitation guide cobre preparação, execução e follow-up
- [ ] Template de trade-offs é reutilizável para qualquer decisão técnica
- [ ] Trade-offs são apresentados de forma visual (matrix ou chart)
- [ ] Mini-RFC do caso retry é completo e tomaria <15 min para ler
- [ ] Critérios de avaliação incluem dimensões técnicas E de negócio
- [ ] Abordou como lidar com "analysis paralysis" e "decision debt"

### Framework de Referência

- **RFC** (Request for Comments): Proposta estruturada para decisão
- **ADR** (Architecture Decision Record): Registro de decisão com contexto
- **Weighted Decision Matrix**: Critérios ponderados para comparação
- **Reversibility** (Jeff Bezos): Type 1 (irreversível, cuidado) vs Type 2 (reversível, velocidade)

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Preparação** | Discute no improviso | Agenda preparada | Pre-read distribuído | Options pré-analisadas |
| **Trade-offs** | "X é melhor que Y" | Lista prós e contras | Matrix com critérios ponderados | Trade-offs quantificados |
| **Facilitação** | Monólogo do sênior | Todos falam | Todos contribuem igualmente | Dissent é buscado ativamente |
| **Decisão** | Empurra para próxima sprint | Decide por votação | Decide com critérios claros | Documenta decisão e rationale |

---

## Desafio 6 — Feedforward: Receber e Dar Feedback Técnico

### Cenário

Situações recentes no squad Payments:
1. Você fez um code review detalhado do PR do Carlos (27 comentários). Ele pareceu defensivo e disse: "Se você não gosta, faz você."
2. Ana te mandou DM pedindo feedback sobre a carreira dela. Você disse "tá indo bem" e ela pareceu insatisfeita.
3. Diana te deu feedback na 1:1: "Você precisa comunicar melhor." Você não entendeu o que especificamente precisa melhorar.
4. Na retro, ninguém levanta problemas reais. Brad Pitt (PM) comentou: "Parece que tudo está sempre 'ok' neste squad."

### Desafio

Desenvolver competência para **dar e receber feedback** de forma que gere aprendizado (não defensividade) — especialmente no contexto de code review e desenvolvimento técnico.

### Entregáveis

1. **Code Review Communication Guide**
   - Como dar feedback no código sem avaliar a pessoa
   - Taxonomy de comentários: blocking, non-blocking, nitpick, question, praise
   - Técnicas: "What if...", "I wonder...", "Have you considered..." vs "This is wrong"
   - Como receber feedback: ouvir, agradecer, considerar, decidir
   - Template para PR comment que é aprendizado, não julgamento
   - Volume guidelines: quantos comentários por PR? Quando é demais?

2. **Feedback Scripts** (6 cenários de engineering)
   - Dar feedback positivo específico e acionável (SBI: Situation-Behavior-Impact)
   - Dar feedback construtivo sem gerar defensividade (SBI + feedforward)
   - Receber feedback negativo com graça (ouvir → agradecer → refletir → agir)
   - Pedir feedback proativamente ("O que posso melhorar no próximo RFC?")
   - Feedback em grupo (retro) — como criar segurança psicológica
   - Feedback para alguém mais senior (managing up feedback)

3. **Feedback Culture Assessment do Squad**
   - Diagnóstico: o squad Payments tem cultura de feedback? (survey/observação)
   - Safety score: as pessoas se sentem seguras para dar feedback honesto?
   - Frequency: feedback acontece regularmente ou só em performance review?
   - Quality: feedback é específico e acionável ou vago e genérico?
   - Plano de ação para melhorar feedback culture (3 ações concretas)

### Critérios de Aceite

- [ ] Code review guide diferencia tipos de comentários com exemplos
- [ ] Scripts cobrem as 6 situações com frases específicas e realistas
- [ ] Cada script tem versão "ruim" (como NÃO fazer) e versão "boa"
- [ ] Assessment tem pelo menos 3 dimensões analisadas com evidências
- [ ] Plano de ação é concreto com 3 ações implementáveis no próximo mês
- [ ] Abordou feedforward (foco no futuro) além de feedback (foco no passado)

### Framework de Referência

- **SBI** (Center for Creative Leadership): Situation → Behavior → Impact
- **Radical Candor** (Kim Scott): Care Personally + Challenge Directly
- **Psychological Safety** (Amy Edmondson): Segurança para ser vulnerável no grupo
- **Feedforward** (Marshall Goldsmith): Sugestões para o futuro vs. avaliação do passado

### Rubrica de Auto-Avaliação

| Dimensão | Iniciante | Praticante | Avançado | Expert |
|----------|-----------|------------|----------|--------|
| **Dar** | Não dá ou é vago | Dá quando pedido | Dá proativamente com SBI | Feedback é natural e contínuo |
| **Receber** | Defensivo | Aceita silenciosamente | Agradece e reflete | Busca ativamente e age sobre |
| **Code Review** | Comentários vagos/rudes | Respeitoso mas genérico | Específico e educativo | Transforma review em mentoria |
| **Cultura** | Feedback só annual review | Feedback em 1:1s | Feedback entre pares regular | Squad com cultura de feedback forte |
