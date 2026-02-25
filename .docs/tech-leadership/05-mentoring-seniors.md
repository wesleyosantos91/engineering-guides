# Mentoring de Seniors — Elevando a Próxima Geração de Staff+

> **Objetivo deste documento:** Servir como referência completa sobre **mentoring de engenheiros senior** — como elevar Seniors a Staff, desenvolver Tech Leads, e construir pipeline de liderança técnica — otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: coaching vs mentoring vs sponsoring, architectural thinking development, delegação progressiva, frameworks de feedback, growth plans.

---

## Quick Reference — Cheat Sheet

| Conceito | Definição | Quando usar |
|----------|----------|-------------|
| **Mentoring** | Guiar com experiência e conselhos | Desenvolvimento contínuo |
| **Coaching** | Perguntas que levam a auto-descoberta | Destravar autonomia |
| **Sponsoring** | Advocacy + dar oportunidades visíveis | Promoção / crescimento acelerado |
| **Stretch Assignment** | Tarefa acima do nível atual | Provar readiness para next level |
| **Delegation Poker** | Gradação de delegação (1-7) | Clarificar ownership |
| **Architectural Pairing** | Pair em design, não só code | Desenvolver systems thinking |

---

## Sumário

- [Mentoring de Seniors — Elevando a Próxima Geração de Staff+](#mentoring-de-seniors--elevando-a-próxima-geração-de-staff)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Por Que Mentoring de Seniors é Diferente](#por-que-mentoring-de-seniors-é-diferente)
  - [Mentoring vs Coaching vs Sponsoring](#mentoring-vs-coaching-vs-sponsoring)
  - [Diagnóstico: Onde Estão os Gaps](#diagnóstico-onde-estão-os-gaps)
  - [Architectural Pair Programming](#architectural-pair-programming)
  - [Delegação Progressiva](#delegação-progressiva)
  - [Growth Plans Concretos](#growth-plans-concretos)
  - [Feedback que Transforma](#feedback-que-transforma)
  - [Papel de Cada Nível no Mentoring](#papel-de-cada-nível-no-mentoring)
  - [Anti-Patterns](#anti-patterns)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Por Que Mentoring de Seniors é Diferente

```
O PARADOXO DO SENIOR:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  JUNIOR → MID → SENIOR:                                │
  │  "Aprenda tecnicamente, resolva problemas cada vez     │
  │   mais complexos, entregue com menos supervisão."      │
  │                                                         │
  │  SENIOR → STAFF:                                        │
  │  "PARE de resolver tudo sozinho. Comece a multiplicar  │
  │   seu impacto ATRAVÉS dos outros."                     │
  │                                                         │
  │  É A TRANSIÇÃO MAIS DIFÍCIL DA CARREIRA DE IC.          │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  POR QUE É DIFÍCIL:

  ┌──────────────────────────────┬────────────────────────────┐
  │ O que fez Senior ter sucesso │ O que ATRAPALHA em Staff  │
  ├──────────────────────────────┼────────────────────────────┤
  │ "Eu sei a resposta"          │ Precisa ouvir antes de     │
  │                              │ falar                      │
  │ "Eu escrevo o código melhor" │ Precisa deixar outros      │
  │                              │ escreverem (e aprenderem)  │
  │ "Eu resolvo rápido"          │ Precisa pensar em trade-   │
  │                              │ offs de longo prazo         │
  │ "Meu time me conhece"        │ Precisa influenciar sem    │
  │                              │ authority em outros times  │
  │ "Entrego features"           │ Precisa gerar impacto      │
  │                              │ sem necessariamente codar  │
  └──────────────────────────────┴────────────────────────────┘

  TIPOS DE MENTEES (Seniors que você mentora):

  TYPE 1: "O IMPLEMENTADOR BRILHANTE"
  → Profundo em tech, escreve código excelente
  → GAP: não sabe influenciar, não pensa cross-team
  → APPROACH: stretches em design review, guild presentations

  TYPE 2: "O GENERALIST CONNECTOR"
  → Conhece um pouco de tudo, ótimo comunicador
  → GAP: falta depth técnica para Staff
  → APPROACH: ownership de problema técnico complexo end-to-end

  TYPE 3: "O EXPERT INTROVERTIDO"
  → Deep expertise em uma área (DB, performance, security)
  → GAP: ninguém sabe o que ele faz (invisibilidade)
  → APPROACH: sponsorship + lightning talks + writing

  TYPE 4: "O OVERACHIEVER ANSIOSO"
  → Quer promoção rápido, faz tudo mas sem profundidade
  → GAP: largura sem profundidade, scope sem ownership 
  → APPROACH: focar em 1-2 áreas de impacto sustentável
```

---

## Mentoring vs Coaching vs Sponsoring

```
3 MODOS DE DEVELOPMENT — QUANDO USAR CADA:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  MENTORING:                                             │
  │  "Quando eu enfrentei isso, eu fiz X porque..."         │
  │                                                         │
  │  → Compartilha experiência e conhecimento               │
  │  → Diz o que fazer (direcionado)                        │
  │  → Bom para: acelerar em área nova, evitar armadilhas   │
  │  → Contexto: Senior em área que o mentor domina         │
  │  → Risco: criar dependência, não desenvolve autonomia   │
  │                                                         │
  │  COACHING:                                              │
  │  "O que você acha que aconteceria se tentasse Y?"       │
  │                                                         │
  │  → Faz perguntas que levam à resposta                   │
  │  → NÃO DÁ a resposta                                   │
  │  → Bom para: desenvolver pensamento independente        │
  │  → Contexto: Senior que sabe mas precisa de confiança   │
  │  → Risco: frustrante se a pessoa realmente não sabe     │
  │                                                         │
  │  SPONSORING:                                            │
  │  "Eu recomendo Ana para liderar esse projeto porque..." │
  │                                                         │
  │  → Abre portas e dá oportunidades                       │
  │  → Advocacia em salas onde o mentee NÃO está            │
  │  → Bom para: promoção, visibilidade, stretch assignments│
  │  → Contexto: Senior pronto mas falta oportunidade/visi- │
  │    bilidade                                             │
  │  → Risco: sponsor reputação vinculada ao mentee         │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  QUANDO USAR CADA MODO:

  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │  ALTO                                                │
  │  │                                                   │
  │  │    MENTORING        COACHING                      │
  │  │    (ensina como)    (faz descobrir)               │
  │  │                                                   │
  │  │                                                   │
  │  │                     SPONSORING                    │
  │  │                     (abre portas)                 │
  │  │                                                   │
  │  │    DIRECTING        DELEGATING                    │
  │  │    (diz o que)      (confia e sai)               │
  │  │                                                   │
  │  BAIXO─────────────────────────────────── ALTO       │
  │            SKILL do mentee na área                   │
  │                                                      │
  │  Eixo vertical: ambiguidade/complexidade            │
  │  Eixo horizontal: skill do mentee                   │
  │                                                      │
  └──────────────────────────────────────────────────────┘

  FREQUÊNCIA RECOMENDADA:

  │ Modo       │ Cadência         │ Formato             │
  │ Mentoring  │ Weekly/bi-weekly │ 1:1 (30-45 min)     │
  │ Coaching   │ Weekly 1:1       │ Embedded in 1:1     │
  │ Sponsoring │ Sempre (async)   │ Advocacy em reuniões│
```

---

## Diagnóstico: Onde Estão os Gaps

```
FRAMEWORK DE DIAGNÓSTICO PARA SENIOR → STAFF:

  Use as 7 competências de liderança técnica para avaliar:

  ┌─────────────────────────────────────────────────────────┐
  │ COMPETÊNCIA            │ Assessment                     │
  │                        │ 1=iniciante  5=Staff-ready     │
  ├────────────────────────┼────────────────────────────────┤
  │ Technical Vision       │ ___                            │
  │ → Articula trade-offs de longo prazo?                  │
  │ → Pensa 2-3 anos à frente?                             │
  │ → Contribui para vision doc?                           │
  ├────────────────────────┼────────────────────────────────┤
  │ Organizational         │ ___                            │
  │ Influence              │                                │
  │ → Persuade fora do time?                               │
  │ → Escreve proposals convincentes?                      │
  │ → Stakeholders buscam sua opinião?                     │
  ├────────────────────────┼────────────────────────────────┤
  │ Cross-team Leadership  │ ___                            │
  │ → Participa de guilds/WGs ativamente?                  │
  │ → Ajuda times vizinhos com problemas?                  │
  │ → Propõe padrões que beneficiam org?                   │
  ├────────────────────────┼────────────────────────────────┤
  │ Mentoring de Seniors   │ ___                            │
  │ → Mentora pelo menos 1 pessoa?                        │
  │ → Faz pair programming com propósito?                  │
  │ → Delega com contexto, não com instruções?             │
  ├────────────────────────┼────────────────────────────────┤
  │ Business Acumen        │ ___                            │
  │ → Entende o problema de negócio?                       │
  │ → Fala a linguagem do produto?                         │
  │ → Conecta decisões técnicas a ROI?                     │
  ├────────────────────────┼────────────────────────────────┤
  │ Written Communication  │ ___                            │
  │ → Escreve design docs claros?                          │
  │ → ADRs com trade-offs bem articulados?                 │
  │ → Outros times leem e referenciam seus docs?           │
  ├────────────────────────┼────────────────────────────────┤
  │ Failure Leadership     │ ___                            │
  │ → Lidera post-mortems sem culpa?                       │
  │ → Transforma incidentes em melhorias reais?            │
  │ → Cria cultura de aprendizado com falhas?              │
  └────────────────────────┴────────────────────────────────┘

  COMO INTERPRETAR:

  │ Score │ Significa                │ Ação                   │
  │ 1-2   │ Awareness, sem prática   │ Mentoring direto + obs │
  │ 3     │ Pratica mas inconsistent │ Coaching + stretches    │
  │ 4     │ Consistente no time      │ Stretch cross-team     │
  │ 5     │ Consistente cross-team   │ Ready para Staff       │

  TEMPLATE DE 1:1 DIAGNÓSTICO:

  "Ana, quero entender onde você está em cada área para
   criar um growth plan. Não é avaliação — é para eu
   saber como te ajudar melhor."

  Perguntas:
  1. "Qual decisão técnica de longo prazo você fez/influenciou
      nos últimos 6 meses?"
  2. "Me conta uma vez que você precisou convencer alguém
      fora do time."
  3. "Quem você está ajudando a crescer? Como?"
  4. "Qual artigo/doc que você escreveu que outros referenciam?"
  5. "Como você conectou uma decisão técnica ao impacto no 
      negócio?"
```

---

## Architectural Pair Programming

```
ARCHITECTURAL PAIRING — ALÉM DO CODE PAIRING:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │ CODE PAIRING:                                           │
  │ "Vamos implementar esse handler juntos."                │
  │ → Desenvolve: coding skill, padrões do time             │
  │ → Bom para: Junior → Mid → Senior                      │
  │                                                         │
  │ ARCHITECTURAL PAIRING:                                  │
  │ "Vamos pensar juntos em como decompor esse feature em   │
  │  componentes e definir as interfaces."                  │
  │ → Desenvolve: systems thinking, trade-off analysis      │
  │ → Bom para: Senior → Staff                             │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  FORMATO DE ARCHITECTURAL PAIRING (90 min):

  ┌────────────────────────────────────────────────────────┐
  │ 1. SETUP (10 min)                                      │
  │    Mentor explica: "Hoje vamos fazer o design de X.    │
  │    Vou pensar em voz alta E quero que você faça        │
  │    o mesmo. Não tem resposta errada."                   │
  │                                                        │
  │ 2. PROBLEM FRAMING (15 min)                            │
  │    MENTEE lidera: definir o problema, requirements,    │
  │    constraints.                                        │
  │    MENTOR: faz perguntas, NÃO corrige (coaching mode)  │
  │                                                        │
  │ 3. OPTIONS EXPLORATION (20 min)                        │
  │    JUNTOS: whiteboard 2-3 opções de design             │
  │    MENTOR pensa em voz alta: "Minha reação imediata    │
  │    é X, porque em experiências anteriores..."          │
  │    MENTEE: propõe opções, não espera mentor             │
  │                                                        │
  │ 4. TRADE-OFF ANALYSIS (20 min)                         │
  │    MENTEE lidera: listar prós/contras de cada opção    │
  │    MENTOR: adiciona dimensões esquecidas               │
  │    (ex: "E sobre operabilidade? E sobre escala?")      │
  │                                                        │
  │ 5. DECISION (15 min)                                   │
  │    MENTEE decide e justifica                            │
  │    MENTOR: "Concordo? Se não, por quê?"                │
  │    Se mentor discorda: "Concordo em discordar e vamos  │
  │    com a sua decisão. Veja por quê depois."            │
  │                                                        │
  │ 6. DEBRIEF (10 min)                                    │
  │    MENTOR: "O que você aprendeu? O que faria           │
  │    diferente? Qual parte foi mais difícil?"            │
  └────────────────────────────────────────────────────────┘

  TEMAS PARA ARCHITECTURAL PAIRING:

  │ Tema                        │ Desenvolve              │
  │ Decomposição de monolito    │ Bounded context thinking│
  │ Design de API pública       │ Contract design          │
  │ Escolha de storage          │ Trade-off analysis       │
  │ Cache strategy              │ Performance thinking     │
  │ Event-driven design         │ Async reasoning          │
  │ Migration plan              │ Incremental thinking     │
  │ Observability design        │ Operational awareness    │
  │ Capacity planning           │ Business + tech thinking │

  PROGRESSÃO:

  Mês 1-2: Mentor guia 70%, mentee observa 30%
  Mês 3-4: 50/50 — mentee propõe, mentor ajusta
  Mês 5-6: Mentee guia 70%, mentor faz provocações 30%
  Mês 7+: Mentee faz sozinho, mentor faz spot review

  SINAIS DE PROGRESSO:
  ✅ Mentee começa considerando trade-offs sem prompt
  ✅ Mentee desenha diagrams antes de código
  ✅ Mentee pergunta "quem é afetado?" naturalmente
  ✅ Mentee referencia decisões anteriores como precedente
  ✅ Outros buscam o mentee para design questions
```

---

## Delegação Progressiva

```
DELEGATION POKER — 7 NÍVEIS DE DELEGAÇÃO:

  ┌─────────────────────────────────────────────────────────┐
  │ NÍVEL │ LABEL     │ SIGNIFICADO                        │
  ├───────┼───────────┼────────────────────────────────────┤
  │ 1     │ TELL      │ Eu decido e digo o que fazer       │
  │ 2     │ SELL      │ Eu decido mas explico o porquê     │
  │ 3     │ CONSULT   │ Eu peço input e depois decido      │
  │ 4     │ AGREE     │ Decidimos juntos                   │
  │ 5     │ ADVISE    │ Você decide, eu aconselho           │
  │ 6     │ INQUIRE   │ Você decide e me informa            │
  │ 7     │ DELEGATE  │ Você decide, não precisa me dizer  │
  └───────┴───────────┴────────────────────────────────────┘

  COMO USAR COM MENTEES:

  INÍCIO DO MENTORING (mês 1):
  → Decisão de tech stack: nível 2 (SELL)
  → Design de API: nível 3 (CONSULT)
  → Implementation: nível 5 (ADVISE)

  MEIO (mês 3-4):
  → Decisão de tech stack: nível 3 (CONSULT)
  → Design de API: nível 5 (ADVISE)
  → Implementation: nível 7 (DELEGATE)

  FINAL (mês 6+):
  → Decisão de tech stack: nível 5 (ADVISE)
  → Design de API: nível 6 (INQUIRE)
  → Implementation: nível 7 (DELEGATE)

  STRETCH ASSIGNMENTS — COMO DELEGAR PARA CRESCIMENTO:

  ┌────────────────────────────────────────────────────────┐
  │ STRETCH ASSIGNMENT TEMPLATE                            │
  │                                                        │
  │ Para: [mentee name]                                    │
  │ Assignment: [descrição]                                │
  │ Growth target: [qual competência desenvolve]           │
  │ Delegation level: [1-7, justificativa]                │
  │ Support: [como vai apoiar]                             │
  │ Check-in: [cadência]                                   │
  │ Success criteria: [como sabe que deu certo]            │
  │ Safety net: [o que acontece se não der certo]          │
  └────────────────────────────────────────────────────────┘

  EXEMPLO:

  Para: Ana (Senior, target Staff)
  Assignment: Liderar o design review do novo serviço 
              de notificações (cross-team)
  Growth target: Cross-team leadership + Written 
                 Communication
  Delegation level: 4 (AGREE) — decidimos juntos se o 
                    design está bom, Ana lidera a sessão
  Support: 1:1 semanal para preparar, estou presente
           na sessão mas não lidero
  Check-in: weekly até completion
  Success criteria: design doc aprovado, pelo menos 2
                    times concordam com a proposta
  Safety net: Se travar, eu posso intervir e fazer 
              co-facilitation

  MENU DE STRETCH ASSIGNMENTS POR COMPETÊNCIA:

  │ Competência            │ Stretch assignment             │
  │ Technical Vision       │ Escrever seção do Tech Radar   │
  │ Org Influence          │ Apresentar proposta para VP    │
  │ Cross-team Leadership  │ Liderar working group          │
  │ Mentoring              │ Mentore 1 mid-level engineer   │
  │ Business Acumen        │ Participar de product planning │
  │ Written Communication  │ Escrever RFC para guild        │
  │ Failure Leadership     │ Facilitar próximo post-mortem  │
```

---

## Growth Plans Concretos

```
GROWTH PLAN — TEMPLATE DE 6 MESES:

  ┌────────────────────────────────────────────────────────┐
  │ GROWTH PLAN: [nome] — Senior → Staff                   │
  │ Mentor: [seu nome]                                     │
  │ Início: [data]  Review: [trimestral]                   │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ DIAGNÓSTICO INICIAL (7 competências):                  │
  │ ─────────────────────────────────────                   │
  │ Technical Vision:       ███░░ (3/5)                    │
  │ Org Influence:          ██░░░ (2/5)                    │
  │ Cross-team Leadership:  ██░░░ (2/5)                    │
  │ Mentoring:              ████░ (4/5)                    │
  │ Business Acumen:        ██░░░ (2/5)                    │
  │ Written Communication:  ███░░ (3/5)                    │
  │ Failure Leadership:     ███░░ (3/5)                    │
  │                                                        │
  │ FOCO (máximo 2 áreas por quarter):                     │
  │ Q1: Org Influence + Written Communication              │
  │ Q2: Cross-team Leadership + Business Acumen            │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ Q1 — AÇÕES ESPECÍFICAS:                                │
  │ ─────────────────────                                   │
  │ Org Influence:                                         │
  │ □ Escrever 1 proposal/RFC e defender para leadership   │
  │ □ Praticar SCQA framework em 3 comunicações            │
  │ □ Identificar 1 sponsor e ter 2 conversas              │
  │                                                        │
  │ Written Communication:                                 │
  │ □ Escrever 2 ADRs com trade-offs claros                │
  │ □ Publicar 1 post no blog interno                      │
  │ □ Receber feedback de 3 pessoas em 1 design doc        │
  │                                                        │
  │ ARCHITECTURAL PAIRING (ongoing):                       │
  │ □ 2x/mês sessão de architectural pairing com mentor    │
  │ □ Foco: system decomposition e API design              │
  │                                                        │
  │ STRETCH ASSIGNMENTS:                                   │
  │ □ Liderar design review de [feature X]                 │
  │ □ Apresentar na Backend Guild sobre [topic Y]          │
  │                                                        │
  │ CHECK-IN: bi-weekly 1:1 dedicado ao growth plan       │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ Q1 REVIEW — EVIDÊNCIAS:                                │
  │ ─────────────────────                                   │
  │ [preenchido no final do quarter com outcomes reais]    │
  └────────────────────────────────────────────────────────┘

  RITUALS DO GROWTH PLAN:

  │ Ritual          │ Cadência  │ Duração  │ Conteúdo              │
  │ 1:1 Growth      │ Bi-weekly │ 30 min   │ Check progress, unblock│
  │ Arch Pairing    │ 2x/mês   │ 90 min   │ Design sessions        │
  │ Quarter Review  │ Trimestral│ 60 min   │ Assessment, re-plan    │
  │ 360 Feedback    │ Semi-anual│ Async    │ Peers + cross-team     │
  │ Promo Discussion│ Anual     │ 45 min   │ Com EM, packet review  │

  SINAIS DE READY PARA STAFF:

  ✅ Completou 2+ stretch assignments cross-team com sucesso
  ✅ Outros times pedem sua opinião em design reviews
  ✅ Escreveu 2+ RFCs que influenciaram decisões
  ✅ Mentora 1+ pessoa ativamente
  ✅ Manager de outro time disse "Ana impactou positivamente
     meu time"
  ✅ Articula trade-offs em termos de negócio (não só tech)
  ✅ Consegue discordar respeitosamente e commitar
  ✅ Lidera sem authority (não precisa de título para
     influenciar)
```

---

## Feedback que Transforma

```
FRAMEWORKS DE FEEDBACK PARA SENIORS:

  ══════════════════════════════════════════════════════════
  SBI — SITUATION / BEHAVIOR / IMPACT
  ══════════════════════════════════════════════════════════

  Mais eficaz para feedback específico (positivo ou negativo)

  SITUAÇÃO: "No design review de ontem..."
  BEHAVIOR: "...você propôs 3 opções com trade-offs e 
             deixou o time decidir..."
  IMPACT: "...isso acelerou a decisão e o Time B me disse
           que se sentiu incluído."

  ANTI-PATTERN: "Você é ótimo em design reviews"
  → Vago, não replicável, não desenvolve

  ══════════════════════════════════════════════════════════
  FEEDFORWARD — PARA CRESCIMENTO
  ══════════════════════════════════════════════════════════

  Em vez de: "Você deveria ter feito X no passado"
  Use: "Da próxima vez, tente X porque..."

  EXEMPLO:
  "Para o próximo RFC, tente adicionar uma seção de 
   'Impacto no Produto' — isso vai fazer o PM prestar
   atenção e te dar prática em Business Acumen."

  ══════════════════════════════════════════════════════════
  PERGUNTAS PODEROSAS (COACHING MODE)
  ══════════════════════════════════════════════════════════

  Em vez de DIZER, PERGUNTE:

  │ Área          │ Pergunta                              │
  │ Technical     │ "Se você tivesse que manter esse      │
  │ Vision        │ sistema por 3 anos, o que mudaria?"   │
  │ Org Influence │ "Quem precisa concordar com isso? O   │
  │               │ que eles se importam?"                 │
  │ Cross-team    │ "Qual time é mais afetado por essa    │
  │               │ decisão? Você falou com eles?"         │
  │ Business      │ "Quanto custa se isso falhar? Quanto  │
  │ Acumen        │ vale se isso funcionar?"               │
  │ Written Comm  │ "Se eu lesse só o primeiro parágrafo, │
  │               │ ela saberia o que fazer?"              │
  │ Failure       │ "O que o sistema nos disse que nós    │
  │ Leadership    │ não escutamos?"                        │

  ══════════════════════════════════════════════════════════
  RADICAL CANDOR — FRAMEWORK GLOBAL
  ══════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────┐
  │         CARE PERSONALLY                              │
  │         │                                            │
  │  HIGH   │  RUINOUS       RADICAL                     │
  │         │  EMPATHY       CANDOR ✅                    │
  │         │  (nice but     (caring +                   │
  │         │   unhelpful)    direct)                    │
  │         │                                            │
  │  LOW    │  MANIPULATIVE  OBNOXIOUS                   │
  │         │  INSINCERITY   AGGRESSION                  │
  │         │  (neither)     (direct but                 │
  │         │                 cold)                      │
  │         │                                            │
  │         └───────────────────────────────── HIGH       │
  │                CHALLENGE DIRECTLY                    │
  └──────────────────────────────────────────────────────┘

  TARGET: Radical Candor — "I care about your growth AND
  I'm telling you the truth."

  EXEMPLO BOM:
  "Ana, eu sei que você se dedicou muito ao RFC. Ele tem
   problemas que vão impedir aprovação: falta análise de
   alternativas e o custo não está claro. Vamos trabalhar
   nisso juntos na 1:1 de terça."

  EXEMPLO RUIM (ruinous empathy):
  "O RFC está ótimo, muito bom!" (sabendo que vai ser 
  rejeitado)
```

---

## Papel de Cada Nível no Mentoring

```
COMO CADA NÍVEL MENTORA:

  ══════════════════════════════════════════════════════════
  TECH LEAD — "Mentor de Mids + Coach de Seniors"
  ══════════════════════════════════════════════════════════

  → Mentor primário de Mid → Senior no time
  → Coach de Seniors: não dá resposta, faz perceber
  → Delegação: entrega ownership de features complexas
  → Code review como ensino (comments pedagógicos)
  → Pair programming regular (code + arch para Seniors)
  
  QUEM MENTORA: 1-3 pessoas do time (mids + seniors novos)
  CADÊNCIA: weekly 1:1 (30 min cada)

  ══════════════════════════════════════════════════════════
  STAFF ENGINEER — "Mentor de Seniors → Staff"
  ══════════════════════════════════════════════════════════

  → Architectural pairing (foco: systems thinking)
  → Sponsor: recomenda Seniors para stretch assignments
  → Growth plans formais com diagnóstico de competências
  → Modelo: Seniors observam como Staff opera cross-team
  → Guild leadership development: prepara mentees para
    liderar sessões
  
  QUEM MENTORA: 2-3 Seniors (do time ou cross-team)
  CADÊNCIA: bi-weekly 1:1 dedicado (45 min)

  ══════════════════════════════════════════════════════════
  PRINCIPAL ENGINEER — "Mentor de Staffs + Pipeline Builder"
  ══════════════════════════════════════════════════════════

  → Mentora Staff → Principal (scope expansion)
  → Cria oportunidades: "Preciso de alguém pra liderar
    esse working group — vou indicar Pedro (Staff)"
  → Systems thinking mentoring: org + tech interplay
  → Feedback em RFCs e design docs (não reescreve,
    faz perguntas)
  → Role model: como operar com ambiguidade em escala
  
  QUEM MENTORA: 1-2 Staffs + spot coaching qualquer nível
  CADÊNCIA: monthly deep dive (60 min) + ad-hoc

  ══════════════════════════════════════════════════════════
  TECH MANAGER — "Enabler + Career Architect"
  ══════════════════════════════════════════════════════════

  → Growth plans formais (objectives, evidence, promo case)
  → Protege tempo de staff ICs para mentoring
  → Conecta mentees com mentors certos
  → Promo packets: articula caso para promoção
  → Organizational sponsorship: advocacy em calibration
  → Budget para learning (conferências, cursos, books)
  
  QUEM GERENCIA: 5-8 reports (mix de seniority)
  CADÊNCIA: weekly 1:1 (30 min) + quarterly career discussion
```

---

## Anti-Patterns

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **Mini-me mentoring** | Só mentora quem pensa igual a você | Diversifique: mentore tipos diferentes de Senior |
| **Rescue mode** | Mentor "salva" mentee quando stuck | Deixe falhar (com safety net) — isso ensina |
| **Eternal mentoring** | Dependência: mentee nunca opera sozinho | Delegation poker: vá subindo os níveis progressivamente |
| **Mentoring without sponsoring** | Dá conselhos mas não abre portas | Sponsorship = advocacy + oportunidades + visibilidade |
| **One-size-fits-all** | Mesmo approach para todos os mentees | Diagnóstico primeiro: cada Senior tem gaps diferentes |
| **Title-obsessed mentoring** | "O que falta para promoção?" como único tema | Foco em growth, título é consequência |
| **Missing safety net** | Stretch assignment sem fallback plan | Sempre definir: "Se X acontecer, faremos Y" |
| **Feedback delay** | Guardar feedback por meses para annual review | SBI imediato (em 48h) — timing é tudo |
| **Mentoring in vacuum** | Mentor não alinha com EM do mentee | Triângulo: Mentor + EM + Mentee alinhados |
| **Skipping the diagnosis** | Pular direto para "faça X" sem entender gaps | Assessment das 7 competências primeiro |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código, considere o aspecto de mentoring e desenvolvimento:

1. **Pedagogic comments** — Code review comments devem ensinar, não só apontar erros; explicar o "porquê" junto com o "o quê"
2. **Level-appropriate feedback** — Ajustar profundidade do feedback ao nível: Mid recebe mais detalhes, Senior recebe provocações
3. **Trade-off discussion** — Se o review mostra decisão sem alternativas avaliadas, sugerir considerar options analysis
4. **Pattern recognition** — Se developer está repetindo erro, sinalizar como growth opportunity em vez de só corrigir
5. **Stretch identification** — Se review mostra que developer está enfrentando problema acima do nível, reconhecer e encorajar
6. **Documentation quality** — Avaliar se comments e docs são claros para outros: "Alguém de outro time entenderia?"
7. **Systems thinking check** — Se PR resolve problema local mas ignora impacto cross-team, questionar visão sistêmica
8. **Delegation evidence** — Se tech lead está fazendo trabalho que deveria delegar, sinalizar como oportunidade de mentoring
9. **Design doc reference** — Se PR implementa design significativo sem doc linkado, sugerir como exercício de escrita
10. **Growth tracking** — Se developer está aplicando feedback de reviews anteriores, reconhecer explicitamente a evolução

---

## Referências

- **The Staff Engineer's Path** — Tanya Reilly (O'Reilly, 2022)
- **An Elegant Puzzle** — Will Larson (Stripe Press, 2019)
- **Radical Candor** — Kim Scott (St. Martin's Press, 2017)
- **The Manager's Path** — Camille Fournier (O'Reilly, 2017)
- **The Coaching Habit** — Michael Bungay Stanier (Box of Crayons, 2016)
- **Thanks for the Feedback** — Douglas Stone & Sheila Heen (Penguin, 2014)
- **Management 3.0: Delegation Poker** — Jurgen Appelo — https://management30.com/practice/delegation-poker/
- **Lara Hogan: Sponsorship** — https://larahogan.me/blog/what-sponsorship-looks-like/
- **Will Larson: Staff Engineer Archetypes** — https://staffeng.com/guides/staff-archetypes
- **SBI Feedback Model** — Center for Creative Leadership — https://www.ccl.org/articles/leading-effectively-articles/closing-the-gap-between-intent-and-impact/
