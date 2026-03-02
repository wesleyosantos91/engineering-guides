# Level 6 — Capstone: Portfolio de Comunicação Estratégica

> **Nível anterior:** [05-fellow-thought-leadership.md](05-fellow-thought-leadership.md)

---

## O Cenário: "The Communication Crucible"

### A Situação

A TechNova acaba de viver as **48 horas mais intensas da sua história**. Uma sequência de eventos encadeados que testa TODAS as competências de comunicação que você desenvolveu ao longo desta trilha:

**Hora 0 (Segunda-feira, 8:00):**
Email do Wall Street Journal: "Temos fontes indicando que a TechNova está negociando aquisição da FinPay (concorrente). Vocês confirmam?" O CTO te liga: "Não era para vazar. O Board vai decidir essa semana. Precisamos de uma estratégia de comunicação AGORA."

**Hora 3 (Segunda-feira, 11:00):**
Enquanto você gerencia a situação do vazamento, o time de Platform detecta uma vulnerabilidade 0-day em uma biblioteca open-source usada em produção. Impacto potencial: dados de 2M de clientes expostos. CISO ativa protocolo de incidente.

**Hora 8 (Segunda-feira, 16:00):**
O artigo do WSJ é publicado (sem confirmação da TechNova, mas com detalhes suficientes para causar reação). O stock price cai 8%. Funcionários da TechNova e da FinPay começam a postar no LinkedIn e Twitter com perguntas, medos e especulações.

**Hora 14 (Segunda-feira, 22:00):**
A vulnerabilidade é pior do que imaginávamos. Evidência de que dados FORAM acessados (não apenas expostos). CISO escala para breach notification obrigatória. Legal diz: 72 horas para notificar reguladores (LGPD/GDPR).

**Hora 24 (Terça-feira, 8:00):**
Board meeting de emergência convocado para Quarta às 8h. Três staff engineers pedem demissão citando "incerteza sobre a aquisição". O All-Hands de Engineering, previamente agendado para Terça às 14h, precisa endereçar TUDO — aquisição, breach, saídas — sem criar pânico ou violar obrigações legais.

**Hora 36 (Terça-feira, 20:00):**
A FinPay publica nota dizendo que "conversas exploratórias" existem, mas "nenhum acordo final". Reguladores enviam notificação formal pedindo relatório detalhado do breach em 48 horas. Imprensa internacional (Reuters, Bloomberg) começa a cobrir a história.

**Hora 48 (Quarta-feira, 8:00):**
Board meeting. Você apresenta. O destino da TechNova — aquisição, resposta ao breach, confiança dos engenheiros, reputação de mercado — depende em grande parte da **comunicação** das próximas 72 horas.

---

## Parte 1 — Diagnóstico de Comunicação (Hora 0-3)

### Desafio

Nos primeiros minutos de uma crise multi-camada, a decisão mais importante é: **o que comunicar, para quem, em que ordem, e o que NÃO comunicar**.

### Entregáveis

1. **Stakeholder Communication Priority Matrix**
   - Mapeamento de TODAS as audiências afetadas (pelo menos 12):
     - Internal: Engineers, Managers, C-suite, Board, Legal, HR, PR
     - External: Clientes, Reguladores, Imprensa, Investidores, FinPay, Partners
   - Para cada: o que sabem, o que pensam, o que temem, o que precisam ouvir
   - Priorização: quem recebe comunicação primeiro e por quê
   - Interdependência: o que comunicar para A depende do que já comunicamos para B

2. **Communication Triage Protocol**
   - Classificação de informações: Confidencial (board only) / Restrito (leadership) / Interno (all employees) / Público (external)
   - O que pode ser dito agora vs o que precisa aguardar (legal/compliance)
   - Quem autoriza cada nível de comunicação (approval chain)
   - Holding statements: o que dizer quando não podemos dizer nada ainda
   - Risk of silence: quando NÃO comunicar é mais perigoso que comunicar imperfeitamente

3. **"First 3 Hours" Communication Plan**
   - Hora 0-1: Contenção (quem sabe? Como garantir que não vaze mais?)
   - Hora 1-2: Assessment (o que realmente sabemos vs especulação?)
   - Hora 2-3: First communication (internal leadership brief — 5 bullets, facts only)
   - Scripts para: "Eu não posso comentar sobre isso agora" (legal) e por que isso é diferente de "sem comentários"
   - War room communication: como coordenar 4 crises simultâneas sem confusão

### Critérios de Aceite

- [ ] Stakeholder Matrix tem 12+ audiências com análise completa
- [ ] Priorização é fundamentada (não arbitrária)
- [ ] Triage protocol diferencia 4 níveis de confidencialidade
- [ ] Holding statements são profissionais e não criam problemas legais
- [ ] First 3 Hours plan é exequível sob pressão real
- [ ] Endereça a tensão entre transparência e obrigação legal/regulatória

---

## Parte 2 — Multi-Crisis Communication (Hora 3-24)

### Desafio

Gerenciar **comunicação simultânea de múltiplas crises** para múltiplas audiências, mantendo consistência, honestidade e compliance.

### Entregáveis

1. **Crisis 1: M&A Leak — Communication Package**
   - Internal communication (all-hands email): acknowledge rumores, separar fatos de especulação, reafirmar valores, próximos passos
   - Employee FAQ: 15+ perguntas sobre aquisição (muitas terão resposta "não podemos comentar neste momento" — como fazer isso sem frustrar)
   - External holding statement (para imprensa): 3 frases, aprovadas por Legal, que não confirmam nem negam
   - Social media monitoring: como responder a posts de funcionários (sem censura, com guidelines)
   - Investor relations statement: obrigatório para empresa pública (SEC compliance)

2. **Crisis 2: Security Breach — Communication Package**
   - Internal engineering communication: what happened, what we know, what we're doing, how to help
   - CISO briefing support: executive summary técnico (facts, impact, remediation, timeline)
   - Customer notification draft: empática, factual, actionable (what they should do)
   - Regulatory notification draft: formal, compliant (LGPD/GDPR formato)
   - Press statement preparation: if/when breach becomes public
   - Coordination: como comunicar breach sem interferir com M&A comm (e vice-versa)

3. **Integrated Communication Timeline**
   - Timeline visual: quem recebe o quê, quando, por qual canal, aprovado por quem
   - Dependency map: mensagem X não pode sair antes de mensagem Y
   - Conflict resolution: quando M&A comm e breach comm precisam de coisas opostas (transparência vs confidencialidade)
   - Escalation triggers: em que condições o plano muda (breach piora, deal cancela, etc.)
   - Communication log: registro de tudo que foi comunicado, para quem, quando (audit trail)

### Critérios de Aceite

- [ ] M&A package tem internal + FAQ + external + social media + IR
- [ ] Breach package tem 5 audiências com comunicações distintas
- [ ] Timeline integra ambas as crises sem contradições
- [ ] Dependency map identifica sequenciamento correto
- [ ] Todas as comunicações existem em versão "aprovada por Legal" (realista)
- [ ] Communication log tem audit trail completo
- [ ] Abordou o caso de funcionários postando nas redes sociais durante crise

---

## Parte 3 — Board Presentation e All-Hands (Hora 24-48)

### Desafio

Preparar e entregar as **duas comunicações mais importantes da sua carreira**: o All-Hands para 600+ engenheiros e o Board meeting de emergência.

### Entregáveis

1. **Engineering All-Hands Script** (30 min + 30 min Q&A)
   - Abertura: acknowledge a ansiedade (don't pretend everything is fine)
   - M&A update: o que podemos dizer, o que não podemos, por que não podemos
   - Security incident: what happened, what we did, what we're doing
   - People update: acknowledge saídas, reafirmar commitment com quem fica
   - Q&A preparation: 30 perguntas difíceis com respostas (algumas serão "I can't answer")
   - Tone: honest, empathetic, confident (not arrogant), human (not corporate)
   - Red lines: o que NÃO dizer no all-hands (legal constraints + respeito às pessoas que saíram)

2. **Board Emergency Presentation** (20 min + 40 min discussion)
   - Slide 1: Situation summary (2 crises + people impact in 1 view)
   - Slides 2-3: M&A status (deal viability, employee sentiment, market reaction)
   - Slides 4-5: Security breach (impact, root cause, remediation, regulatory status)
   - Slides 6-7: Engineering stability (attrition risk, retention plan, critical roles)
   - Slide 8: Communication executed (what we said, to whom, reactions)
   - Slides 9-10: Recommendations (3 options with trade-offs for board decision)
   - Pre-read memo: 3 pages narrative + appendix com details

3. **Post-Meeting Communication Cascade**
   - Board decisions → CTO → VP of Eng → Directors → Managers → Engineers (cascade)
   - What changed after board meeting: how to communicate CHANGES to plan
   - Individual conversations: for the 3 engineers who resigned (exit interview as data)
   - Partner/customer communication: update com new information and commitment
   - Media response: updated statement if press coverage intensifies

### Critérios de Aceite

- [ ] All-Hands script é apresentável para 600 pessoas ansiosas
- [ ] Script balances transparência com obrigação legal
- [ ] Q&A tem 30 perguntas realistas (inclui as que mais doem)
- [ ] Board presentation tem máximo 10 slides + 3-page pre-read
- [ ] 3 opções de recommendation com trade-offs claros para board decision
- [ ] Post-meeting cascade não quebra informação nem cria rumor
- [ ] Endereçou como responder a "por que 3 pessoas pediram demissão?"

---

## Parte 4 — Portfolio de Comunicação e Reflexão

### Desafio

Após a crise, criar um **portfolio consolidado de comunicação** que demonstre sua evolução ao longo de toda a trilha de Strategic Communication.

### Entregáveis

1. **Communication Portfolio Completo**
   - **Index**: todos os artefatos criados nos níveis 0-5 + capstone, organizados por competência
   - **Greatest Hits**: os 5 melhores artefatos que melhor demonstram suas competências
   - **Lessons Learned**: análise do que cada nível te ensinou sobre comunicação
   - **Evolution**: compare um entregável do Nível 0 com um do Nível 5 — como sua comunicação evoluiu?
   - **Self-Assessment**: score honesto para cada competência da trilha (com evidências)
   - **Blind Spots**: o que você AINDA não sabe fazer bem em comunicação

2. **Communication Philosophy Document** (ensaio pessoal)
   - Seus 7 princípios de comunicação (destilados da experiência)
   - Sua definição de "excelência em comunicação para engenheiros"
   - O que você acredita sobre: writing, speaking, listening, deciding, negotiating
   - What the world gets wrong about technical communication
   - O papel da vulnerabilidade na comunicação profissional
   - Comunicação e ética: onde você traça a linha entre persuasão e manipulação

3. **Impact Reflection**
   - Mapear 3 situações reais na sua carreira onde comunicação fez a diferença
   - Para cada: contexto, o que você fez, o que deu certo, o que faria diferente
   - Quantificar impacto: decisão influenciada, conflito resolvido, time alinhado
   - Feedback recebido de peers, managers, subordinados sobre sua comunicação (360°)
   - **"Letter to My Past Self"**: o que você diria sobre comunicação para o(a) engenheiro(a) que começou esta trilha

4. **Peer Teaching Session**
   - Preparar e entregar uma sessão de 30 min para pares sobre "Communication for Engineers"
   - Conteúdo: os 3 insights mais valiosos que você aprendeu nesta trilha
   - Formato: interactive (não lecture) — exercícios, discussão, prática
   - Feedback: coletar feedback dos participantes sobre a sessão
   - Meta-reflexão: ensinar é o teste final de compreensão

### Critérios de Aceite

- [ ] Portfolio tem index completo com todos os artefatos dos 7 níveis
- [ ] Greatest Hits tem 5 artefatos selecionados com justificativa
- [ ] Evolution mostra progresso claro com evidências
- [ ] Self-Assessment é honesto (nem 10 em tudo, nem self-deprecating)
- [ ] Philosophy document é genuinamente pessoal (não generic)
- [ ] Impact reflection tem 3 situações com análise profunda
- [ ] Letter to past self é emocional E prática
- [ ] Peer teaching session foi efetivamente realizada com pelo menos 3 participantes

---

## Meta-Rubrica do Capstone

| Dimensão | Nível 1 | Nível 2 | Nível 3 | Nível 4 |
|----------|---------|---------|---------|---------|
| **Complexidade** | Endereça 1 crise | Endereça 2 crises | Integra múltiplas crises | Navega complexidade com clareza |
| **Multi-Audiência** | 2 audiências | 4 audiências | 8 audiências | 12+ audiências customizadas |
| **Consistência** | Contradições entre mensagens | Alinhamento parcial | Consistente across audiências | Framework integrado, múltiplas manifestações |
| **Ética** | Omissões problemáticas | Honesto mas desajeitado | Transparente com tato | Balanceia transparência, empatia e legal |
| **Reflexão** | Descritiva | Analítica | Insights genuínos | Transformação pessoal documentada |
| **Legacy** | Individual | Compartilhável | Ensinável | Inspirador e replicável |
