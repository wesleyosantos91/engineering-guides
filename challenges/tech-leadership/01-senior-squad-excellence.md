# Level 1 — Senior: Excelência no Squad

> **Objetivo:** Liderar pelo exemplo dentro do squad — mentoria, feedback eficaz, code review como
> ferramenta de desenvolvimento, facilitação de cerimônias e construção de confiança no time.

**Esfera de impacto:** Squad (5-8 pessoas)
**Referência:** Pré-requisito para [02-staff-cross-team-alignment.md](02-staff-cross-team-alignment.md)

---

## Contexto — TechNova (Série A)

A TechNova levantou sua Série A e cresceu de 8 para 25 engenheiros organizados em 4 squads. Você é **Senior Engineer** no squad de Payments — o squad mais crítico para o negócio. O squad tem 6 pessoas: você (senior), 2 plenos, 2 juniores e 1 tech lead (que acumula função com management). O tech lead está sobrecarregado e precisa que você assuma mais responsabilidades de **liderança técnica informal**.

---

## Desafios

### Desafio 1.1 — 1:1s Eficazes (Mentoria Estruturada)

**Cenário:**
O tech lead pediu que você faça 1:1s semanais com os 2 devs juniores do squad. Um deles (Ana) é talentosa mas quieta — raramente fala nas dailies e nunca questiona decisões técnicas. O outro (Bruno) é comunicativo mas disperso — começa muitas coisas e não termina, e frequenta as discussões de arquitetura de outros squads sem entregar no próprio.

**Requisitos:**
- Estruturar **agendas de 1:1** adaptadas para cada perfil (Ana vs Bruno)
- Praticar perguntas do **modelo GROW** (Goal, Reality, Options, Will)
- Desenvolver habilidade de **coaching** (fazer perguntas) vs **mentoring** (dar respostas)
- Diferenciar quando usar coaching vs mentoring para cada pessoa
- Criar **tracking de desenvolvimento** para acompanhar evolução ao longo do tempo

**Entregáveis:**

1. **1:1 Template** — documento com estrutura base:
   - Check-in pessoal (5 min) — "Como você está?"
   - Agenda do mentorado (15 min) — tópicos que ELE quer discutir
   - Agenda do mentor (10 min) — feedback, observações, desenvolvimento
   - Action items (5 min) — compromissos para próxima semana
   - Meta-feedback (2 min) — "Este 1:1 está sendo útil? O que mudo?"

2. **Planos de Desenvolvimento Individuais:**
   - **Ana:** plano para desenvolver assertividade e voz técnica
     - Semana 1-2: Observar e documentar quando Ana tem opinião mas não fala
     - Semana 3-4: Criar espaços seguros (pedir opinião diretamente, elogiar contribuições)
     - Semana 5-8: Delegar apresentação de design para Ana com suporte prévio
   - **Bruno:** plano para desenvolver foco e follow-through
     - Semana 1-2: Co-criar WIP limits pessoais, visualizar em board pessoal
     - Semana 3-4: Feedback sobre impacto da dispersão (SBI)
     - Semana 5-8: Responsabilizar por entrega end-to-end de 1 feature

3. **Conversation Log** — registro de 4+ 1:1s simulados com Ana e Bruno:
   - Perguntas feitas | Tipo (coaching vs mentoring) | Resultado obtido

**Critérios de aceite:**
- [ ] 1:1 Template completo e aplicável (não genérico)
- [ ] Plano para Ana com ações progressivas de empowerment
- [ ] Plano para Bruno com ações de foco sem micromanagement
- [ ] Conversation Log com pelo menos 4 sessões simuladas por pessoa
- [ ] Uso balanceado de coaching (perguntas) vs mentoring (respostas)
- [ ] Reflexão: "Como adapto meu estilo de mentoria para perfis diferentes?"

**Framework de Referência:**
- GROW Model (John Whitmore — *Coaching for Performance*)
- Michael Bungay Stanier — *The Coaching Habit* (7 perguntas)
- Situational Leadership (Hersey & Blanchard) — D1→D4

---

### Desafio 1.2 — Feedback Eficaz (SBI + Radical Candor)

**Cenário:**
Três situações no squad precisam de feedback:

1. **Ana** fez um excelente refactoring no módulo de reconciliação — reduziu complexidade ciclomática em 40% e adicionou testes. Ninguém notou publicamente.
2. **Bruno** fez deploy de uma feature sem atualizar a documentação da API. É a terceira vez este mês. O tech lead está irritado mas não falou nada diretamente.
3. **Carlos** (dev pleno) discordou da sua decisão de arquitetura na daily e o fez de forma agressiva na frente do time: "Essa abordagem é claramente inferior, qualquer senior saberia disso."

**Requisitos:**
- Usar o framework **SBI** (Situation-Behavior-Impact) para estruturar os 3 feedbacks
- Mapear cada situação no quadrante do **Radical Candor** (Care Personally × Challenge Directly)
- Praticar feedback **positivo** (Ana), **construtivo** (Bruno) e **em resposta a agressão** (Carlos)
- Desenvolver **timing** — quando dar feedback imediato vs agendar conversa

**Entregáveis:**

1. **3 Feedback Scripts** usando SBI:
   - **Ana (positivo):**
     - Situation: "No refactoring do módulo de reconciliação esta semana..."
     - Behavior: "Você reduziu a complexidade ciclomática em 40% e adicionou X testes..."
     - Impact: "Isso facilitou que o time inteiro entendesse o módulo e reduziu bugs em..."
   - **Bruno (construtivo):**
     - Situation: "Nas últimas 3 features deployadas..."
     - Behavior: "A documentação da API não foi atualizada..."
     - Impact: "Os consumers da API ficaram sem saber das mudanças, e [time X] integrou errado..."
     - + Ask: "O que está impedindo você de atualizar a doc? Como posso ajudar?"
   - **Carlos (confrontation):**
     - Resposta imediata (na daily): modelo de resposta assertiva sem escalar
     - Conversa posterior (1:1): SBI completo sobre o impacto da agressividade

2. **Radical Candor Map** — posicionar cada situação no quadrante:
   ```
                    Challenge Directly
                          │
        Obnoxious         │        Radical
        Aggression        │        Candor
                          │         ★
   ──────────────────────┼──────────────────
                          │
        Manipulative      │        Ruinous
        Insincerity       │        Empathy
                          │
                Care Personally
   ```
   - Análise: onde VOCÊ tipicamente cai? Qual seu anti-pattern?

3. **Feedback Timing Guide** — quando aplicar cada tipo:
   - Feedback positivo: timing ideal, frequência, público vs privado
   - Feedback construtivo: timing, ambiente, preparação
   - Feedback em resposta a conflito: escalation ladder

**Critérios de aceite:**
- [ ] 3 feedback scripts completos usando SBI com linguagem natural (não robótica)
- [ ] Radical Candor Map com auto-análise de onde você tipicamente cai
- [ ] Feedback Timing Guide com regras práticas
- [ ] Script para Carlos inclui resposta imediata + follow-up
- [ ] Reflexão: "Qual tipo de feedback é mais difícil para mim (positivo, construtivo, confrontation) e por quê?"

**Framework de Referência:**
- SBI Model (Center for Creative Leadership)
- Kim Scott — *Radical Candor*
- Douglas Stone & Sheila Heen — *Thanks for the Feedback*

---

### Desafio 1.3 — Code Review como Ferramenta de Desenvolvimento

**Cenário:**
O squad de Payments tem um problema: code reviews viram gargalo. Você e o tech lead são os únicos que revisam. Os reviews são superficiais ("LGTM") ou excessivamente detalhistas (nit-picking em estilo de código). Os juniores têm medo de submeter PRs porque sempre recebem 30+ comentários. Os plenos não revisam porque "não se sentem qualificados".

**Requisitos:**
- Redesenhar o processo de code review do squad com foco em **aprendizado**, não apenas **gatekeeping**
- Criar **guidelines de review** que equilibrem rigor e construtividade
- Desenvolver habilidade de fazer **comentários de review que ensinam** (não apenas corrigem)
- Criar mecanismo de **peer review** onde todos do squad participam

**Entregáveis:**

1. **Code Review Guidelines** para o squad:
   - Categorização de comentários: 🔴 Blocker | 🟡 Suggestion | 🟢 Nit | 📚 Learning
   - SLA de review: tempo máximo para primeira resposta
   - Quem revisa: distribuição rotativa (todos revisam, seniors mentoram)
   - Tamanho de PR: guidelines de tamanho ideal (max lines, max files)
   - Auto-review checklist: o que verificar ANTES de pedir review

2. **Comment Templates** — exemplos de como transformar comentários:
   | ❌ Antes | ✅ Depois |
   |----------|----------|
   | "Isso está errado" | "🔴 Esse approach pode causar [problema X]. Uma alternativa seria [Y] porque [razão]. [Link para doc]" |
   | "LGTM" | "✅ Gostei de como você [aspecto específico]. Uma sugestão para próximo PR: [growth opportunity]" |
   | "Use map ao invés de for" | "🟢 Nit: `map` aqui deixaria mais declarativo. Mas o `for` funciona — fica ao seu critério" |

3. **Peer Review Onboarding** — plano para engajar juniores e plenos como reviewers:
   - Semana 1: Pair review (junior + senior revisam juntos)
   - Semana 2: Junior revisa com checklist, senior valida o review
   - Semana 3: Junior revisa independente em PRs menores
   - Semana 4+: Review rotativo normal

**Critérios de aceite:**
- [ ] Code Review Guidelines completo e adotável pelo squad
- [ ] Comment Templates com pelo menos 10 exemplos antes/depois
- [ ] Peer Review Onboarding plan progressivo e não intimidante
- [ ] Categorização de comentários clara (Blocker / Suggestion / Nit / Learning)
- [ ] Reflexão: "Como code reviews podem ser a ferramenta de mentoria mais eficaz do squad?"
- [ ] Análise de 3 code reviews reais (ou simulados) aplicando as guidelines

**Framework de Referência:**
- Google Engineering Practices — *How to do a code review*
- Conventional Comments (conventionalcomments.org)
- Chelsea Troy — *Code Review Mindset*

---

### Desafio 1.4 — Facilitação de Rituais do Squad

**Cenário:**
As cerimônias do squad estão improdutivas:
- **Daily standup** — vira report de status, 30+ minutos, ninguém se ouve
- **Planning** — debatemos forever, não estimamos, não commitamos
- **Retrospectiva** — mesmos problemas toda sprint, nenhum action item executado
- **Design review** — só você e o tech lead falam, o resto observa passivamente

O tech lead pediu que você assuma a **facilitação** de pelo menos 2 cerimônias.

**Requisitos:**
- Diagnosticar os **anti-patterns** de cada cerimônia
- Redesenhar o **formato e facilitação** de no mínimo 2 cerimônias
- Praticar técnicas de **facilitação** (time-boxing, round-robin, dot voting, silent brainstorming)
- Garantir **participação equilibrada** (especialmente de Ana, que é quieta)

**Entregáveis:**

1. **Diagnóstico de Anti-Patterns** — para cada cerimônia:
   - O que está acontecendo? (observação factual)
   - Por que está acontecendo? (root cause)
   - Qual o impacto? (no time, na entrega, na moral)

2. **Playbook de Facilitação** para 2+ cerimônias, contendo:
   - **Objetivo** da cerimônia (em 1 frase)
   - **Formato** com timebox por bloco
   - **Regras** de participação (ex: "round-robin, 30 seg cada")
   - **Ferramentas** de facilitação (dot voting, 1-2-4-all, silent writing)
   - **Anti-patterns** a evitar e como intervir
   - **Inclusão** — técnicas específicas para incluir participantes quietos

3. **Facilitation Script** — roteiro detalhado da primeira sessão no novo formato:
   - Abertura: como explicar ao time por que mudou o formato
   - Transições: frases exatas para redirecionar quando sair do foco
   - Encerramento: como fechar com action items e owners
   - Intervenções: como lidar com quem domina, quem desvia, quem não participa

**Critérios de aceite:**
- [ ] Diagnóstico de anti-patterns para pelo menos 3 cerimônias
- [ ] Playbook de Facilitação completo para 2+ cerimônias
- [ ] Facilitation Script com roteiro realista e linguagem natural
- [ ] Pelo menos 3 técnicas de inclusão para participantes quietos
- [ ] Reflexão: "Qual a diferença entre facilitar e controlar uma reunião?"
- [ ] Métricas de sucesso definidas para cada cerimônia (como saber se melhorou)

**Framework de Referência:**
- Liberating Structures — *1-2-4-All*, *Troika Consulting*, *TRIZ*
- Sam Kaner — *Facilitator's Guide to Participatory Decision-Making*
- Esther Derby & Diana Larsen — *Agile Retrospectives*

---

### Desafio 1.5 — Construção de Confiança (Trust Building)

**Cenário:**
O squad recebeu 2 novos membros: Diana (transferida de outro squad, frustrada porque foi "movida sem consulta") e Eduardo (contratação recente, ansioso para provar valor mas inseguro sobre a cultura). A confiança no squad precisa ser reconstruída — Diana não confia na liderança, Eduardo não confia que pode errar sem consequência.

**Requisitos:**
- Entender os **5 disfunções de um time** (Lencioni) e identificar onde o squad está
- Mapear o **Trust Equation** (Credibility + Reliability + Intimacy / Self-Orientation)
- Criar plano de **onboarding relacional** (não só técnico) para novos membros
- Desenvolver **rituais de confiança** no squad

**Entregáveis:**

1. **Team Health Assessment** usando o modelo de Lencioni:
   - Ausência de Confiança: sinais no squad (0-10 + evidências)
   - Medo de Conflito: sinais no squad (0-10 + evidências)
   - Falta de Comprometimento: sinais no squad (0-10 + evidências)
   - Evitar Accountability: sinais no squad (0-10 + evidências)
   - Desatenção a Resultados: sinais no squad (0-10 + evidências)

2. **Trust Building Plan** para Diana e Eduardo:
   - **Diana:** plano para reconstruir confiança na liderança
     - Semana 1: 1:1 de escuta ativa — ouvir frustração sem "resolver"
     - Semana 2: Transparência — compartilhar contexto da decisão de transferência
     - Semana 3: Autonomia — dar ownership de decisão técnica relevante
     - Semana 4+: Follow-through — entregar o que prometeu, ser previsível
   - **Eduardo:** plano para criar psychological safety
     - Semana 1: Buddy system — parear com membro mais acolhedor
     - Semana 2: Normalizar erros — compartilhar seus próprios erros publicamente
     - Semana 3: Primeira contribuição — PR pequeno com review construtivo
     - Semana 4+: Celebrar aprendizado — reconhecer perguntas, não só respostas

3. **Rituais de Confiança** — práticas semanais/mensais:
   - Semana: "Failure Friday" — alguém compartilha algo que não sabia/errou e o que aprendeu
   - Mensal: "Brag Doc Review" — cada um compartilha suas realizações (anti-síndrome de impostor)
   - Contínuo: "Learning in Public" — channel do Slack onde time compartilha TILs sem julgamento

**Critérios de aceite:**
- [ ] Team Health Assessment completo com score e evidências para cada disfunção
- [ ] Trust Building Plan específico para Diana (confiança) e Eduardo (safety)
- [ ] 3+ rituais de confiança concretos e implementáveis
- [ ] Trust Equation aplicada a si mesmo (como aumentar seu trust score)
- [ ] Reflexão: "O que eu faço que DIMINUI confiança no time sem perceber?"
- [ ] Plano de onboarding relacional (primeiras 4 semanas de um novo membro)

**Framework de Referência:**
- Patrick Lencioni — *The Five Dysfunctions of a Team*
- Amy Edmondson — *The Fearless Organization* (psychological safety)
- Charles Feltman — *The Thin Book of Trust* (Trust Equation)

---

### Desafio 1.6 — Technical Ownership & Tomada de Decisão no Squad

**Cenário:**
O squad precisa decidir entre duas abordagens para o novo módulo de recurring payments:

**Opção A:** Event Sourcing — Ana e você defendem. Mais complexo, mas auditoria natural e time-travel de estados.
**Opção B:** CRUD com audit log — Bruno e Carlos defendem. Mais simples, entrega mais rápido, mas menos flexível.

O tech lead está de férias. A decisão precisa ser tomada esta semana. O squad nunca tomou uma decisão técnica sozinho — sempre esperava o tech lead decidir.

**Requisitos:**
- Facilitar o processo de **decisão coletiva** sem impor sua preferência
- Aplicar um framework estruturado de **tomada de decisão técnica**
- Documentar a decisão em formato **Lightweight ADR** (Architecture Decision Record)
- Gerenciar o risco de **groupthink** e **HIPPO** (Highest Paid Person's Opinion)

**Entregáveis:**

1. **Decision Facilitation Script** — como conduzir a sessão de decisão:
   - Abertura: definir o problema, não a solução (separar problema de proposta)
   - Exploração: cada opção apresentada por quem defende (5 min cada)
   - Critérios: time TOGETHER define critérios de avaliação ANTES de avaliar
   - Avaliação: matrix de decisão com pesos nos critérios
   - Decisão: método escolhido (consenso, consent, votação, consultative)
   - Commit: "disagree and commit" protocol

2. **Lightweight ADR** para a decisão tomada:
   ```
   # ADR-001: Abordagem para Recurring Payments
   
   ## Status: Accepted
   ## Context: [cenário]
   ## Decision: [qual opção e por quê]
   ## Consequences: [trade-offs aceitos]
   ## Participants: [quem participou da decisão]
   ## Dissent: [registro respeitoso de quem divergiu e o argumento]
   ```

3. **Anti-HIPPO Checklist** — práticas para evitar decisões enviesadas:
   - Pessoa mais senior fala por último
   - Critérios definidos ANTES de avaliar opções
   - Silent writing antes de discussão verbal
   - Devil's advocate designado para cada opção
   - Explicitação de "o que precisa ser verdade para essa opção funcionar?"

**Critérios de aceite:**
- [ ] Decision Facilitation Script detalhado e replicável
- [ ] Lightweight ADR completo incluindo dissent registrado
- [ ] Anti-HIPPO Checklist com pelo menos 5 práticas
- [ ] Análise de qual método de decisão (consenso/consent/consultative) é melhor para este caso
- [ ] Reflexão: "Como separar minha preferência técnica pessoal da facilitação neutra?"
- [ ] Framework de "disagree and commit" — como apoiar decisão que não é a sua preferida

**Framework de Referência:**
- Michael Nygard — *Documenting Architecture Decisions* (ADR format)
- Sociocracy — Consent-based decision making
- Amazon — *Disagree and Commit* (Leadership Principle)

---

## Entregáveis do Nível

| Artefato | Descrição |
|----------|-----------|
| 1:1 Templates + Development Plans | Mentoria estruturada para devs junior |
| 3 Feedback Scripts (SBI) | Feedback positivo, construtivo e em resposta a conflito |
| Code Review Guidelines | Processo de review focado em desenvolvimento |
| Facilitation Playbook | Redesign de 2+ cerimônias do squad |
| Team Health Assessment | Diagnóstico de confiança e safety |
| Trust Building Plans | Planos individualizados para novos membros |
| Lightweight ADR + Decision Process | Decisão técnica facilitada e documentada |

---

## Rubrica de Auto-Avaliação

| Competência | Iniciante (1) | Praticante (2) | Avançado (3) | Expert (4) |
|-------------|---------------|----------------|--------------|------------|
| **Mentoria/1:1** | 1:1s sem estrutura, "como posso ajudar?" genérico | Template básico, adapta por pessoa | Coaching vs mentoring situacional, tracking de progresso | Mentor de mentores, cria cultura de mentoria no squad |
| **Feedback** | Evita feedback ou dá de forma vaga | Usa SBI mas desconfortável com conflito | Feedback natural, positivo e construtivo regularmente | Cria cultura de feedback contínuo no time |
| **Code Review** | LGTM ou nit-picking, não ensina | Comentários categorizados, alguns pedagógicos | Reviews que ensinam, incluem alternativas e razões | Everyone reviews, reviews são o melhor momento de aprendizado |
| **Facilitação** | Reuniões sem estrutura, mesmas pessoas falam | Timeboxing básico, agenda definida | Técnicas de inclusão, output claro, action items | Time se auto-facilita, você não é mais necessário |
| **Trust Building** | Não investe em relações, foco só em código | Reconhece importância mas inconsistente | Rituais ativos, onboarding relacional, normaliza erros | Psychological safety é visível, time assume riscos |
| **Decisão Técnica** | Decide sozinho ou espera alguém decidir | Consulta opiniões mas decide por autoridade | Facilita decisão coletiva com framework | Time é autônomo, ADRs são prática natural |

---

## Próximo Nível

Quando completar todos os desafios deste nível, avance para:
→ [Level 2 — Staff: Alinhamento Cross-Team](02-staff-cross-team-alignment.md)

As habilidades deste nível (mentoria, feedback, facilitação, trust, decisão) são exercidas **dentro do squad**. No próximo nível, você expandirá sua esfera de influência para **múltiplos times** — onde liderar sem autoridade se torna essencial.
