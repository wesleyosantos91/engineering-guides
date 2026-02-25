# Organizational Influence — Influenciar sem Autoridade

> **Objetivo deste documento:** Servir como referência completa sobre **influência organizacional** — como convencer stakeholders, alinhar direções e promover mudanças sem autoridade direta — otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: escrita persuasiva, apresentações para liderança, navegação política, construção de alianças, sponsor vs mentor, influência por nível.

---

## Quick Reference — Cheat Sheet

| Conceito | Definição | Uso |
|----------|----------|-----|
| **Influence without authority** | Convencer sem ser chefe | Core skill de Staff+ |
| **Sponsor** | Pessoa que advoga por você em salas onde você não está | Acelerador de carreira |
| **Mentor** | Pessoa que aconselha e compartilha experiência | Crescimento técnico |
| **Stakeholder mapping** | Identificar quem decide, quem influencia, quem bloqueia | Antes de qualquer proposta |
| **Pre-wire** | Alinhar decisão antes da reunião formal | Garantir aprovação |
| **SCQA** | Situation-Complication-Question-Answer framework | Estruturar comunicação persuasiva |

---

## Sumário

- [Organizational Influence — Influenciar sem Autoridade](#organizational-influence--influenciar-sem-autoridade)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Por que Influência é a Skill #1](#por-que-influência-é-a-skill-1)
  - [Stakeholder Mapping](#stakeholder-mapping)
  - [Frameworks de Persuasão](#frameworks-de-persuasão)
  - [Pre-wiring — Alinhamento Antes da Reunião](#pre-wiring--alinhamento-antes-da-reunião)
  - [Escrita Persuasiva](#escrita-persuasiva)
  - [Apresentações para Liderança](#apresentações-para-liderança)
  - [Navegando Resistência e Conflito](#navegando-resistência-e-conflito)
  - [Sponsors vs Mentors](#sponsors-vs-mentors)
  - [Influência por Nível](#influência-por-nível)
  - [Anti-Patterns](#anti-patterns)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Por que Influência é a Skill #1

```
POR QUE INFLUÊNCIA IMPORTA MAIS QUE HABILIDADE TÉCNICA:

  ┌─────────────────────────────────────────────────────────────┐
  │                                                              │
  │  CENÁRIO 1: Principal brilhante, sem influência             │
  │  → Escreve RFC perfeito                                     │
  │  → Ninguém lê                                                │
  │  → Ninguém implementa                                       │
  │  → Impacto = ZERO                                           │
  │                                                              │
  │  CENÁRIO 2: Staff bom, com influência                       │
  │  → Escreve RFC "bom o suficiente"                           │
  │  → Pre-alinha com 3 stakeholders                            │
  │  → 5 times implementam                                      │
  │  → Impacto = ENORME                                         │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘

  A EQUAÇÃO DO IMPACTO:

  Impacto = Qualidade da Ideia × Adoção
  
  Onde:
  → Qualidade da Ideia: 0-10 (design, análise, viabilidade)
  → Adoção: 0-10 (quantas pessoas/times implementam?)
  
  Ideia nota 10 × adoção 0 = 0 (gaveta)
  Ideia nota 7 × adoção 8 = 56 (alto impacto)

  POR QUE É DIFÍCIL PARA ENGINEERS:
  → Treinados para "o melhor argumento técnico vence"
  → Na realidade: política, timing, relações, e contexto vencem
  → Influência não é manipulação — é COMUNICAÇÃO EFICAZ
  → "Sendo técnicamente correto" ≠ "convencendo as pessoas"
```

---

## Stakeholder Mapping

```
STAKEHOLDER MAPPING — SABER COM QUEM FALAR:

  ANTES DE QUALQUER PROPOSTA SIGNIFICATIVA:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │          ALTO PODER                                     │
  │          │                                              │
  │  ┌───────┼──────────────────────────────────────┐      │
  │  │       │                                      │      │
  │  │  KEEP │               MANAGE                 │      │
  │  │ SATIS-│              CLOSELY                 │      │
  │  │ FIED  │           (key stakeholders)         │      │
  │  │       │                                      │      │
  │  │ VP Eng│     CTO    Principal do domínio      │      │
  │  │       │     EM do time afetado               │      │
  │  │       │                                      │      │
  │  ├───────┼──────────────────────────────────────┤      │
  │  │       │                                      │      │
  │  │MONITOR│               KEEP                   │      │
  │  │       │             INFORMED                  │      │
  │  │       │          (supporters)                │      │
  │  │ Legal │    Staff Engineers aliados            │      │
  │  │ Finance│   Tech Leads dos times              │      │
  │  │       │    Product Managers                   │      │
  │  │       │                                      │      │
  │  └───────┼──────────────────────────────────────┘      │
  │     BAIXO│INTERESSE ─────────────── ALTO INTERESSE     │
  │          │                                              │
  └──────────┴──────────────────────────────────────────────┘

  PARA CADA KEY STAKEHOLDER, MAPEAR:

  │ Nome   │ Papel      │ Interesse │ Posição │ O que quer         │
  │ Maria  │ CTO        │ Alto      │ Neutro  │ ROI, risk reduction│
  │ Pedro  │ VP Product │ Alto      │ Contra  │ Feature velocity   │
  │ Ana    │ Principal  │ Alto      │ A favor │ Tech excellence    │
  │ Carlos │ EM Payments│ Médio     │ Neutro  │ Team capacity      │

  Posições:
  → CHAMPION: vai defender por você (cultive!)
  → SUPPORTER: concorda, mas não vai defender ativamente
  → NEUTRAL: não formou opinião
  → SKEPTIC: tem dúvidas, pode ser convencido
  → BLOCKER: contra, precisa de atenção especial

  ESTRATÉGIA POR POSIÇÃO:
  
  CHAMPION: 
  → Mantenha informado, peça para advocar em reuniões
  → Dê crédito público: "Como a Ana sugeriu..."
  
  SUPPORTER:
  → Transforme em champion: envolva no design
  → Peça review do RFC: ownership cria advocacy
  
  NEUTRAL:
  → 1:1 para entender concerns
  → Mostre "what's in it for them"
  → Dados e exemplos concretos
  
  SKEPTIC:
  → Ouça PRIMEIRO (realmente ouça)
  → Acknowledge concerns genuinamente
  → Adapte proposta para endereçar concerns
  → Encontre common ground
  
  BLOCKER:
  → Entenda a motivação REAL (geralmente não é o que dizem)
  → 1:1 privado, nunca confronte em grupo
  → Se não converter: escalate ou go around (último recurso)
  → Às vezes: incorpore feedback e reconheça publicamente
```

---

## Frameworks de Persuasão

```
FRAMEWORKS PARA ESTRUTURAR ARGUMENTOS:

  ══════════════════════════════════════════════════════════════
  1. SCQA (Situation-Complication-Question-Answer)
  ══════════════════════════════════════════════════════════════

  Melhor para: executive communication, business cases

  SITUATION: Fato aceito por todos (common ground)
  "Nossa plataforma roda em single-region (us-east-1) 
   e atende clientes no Brasil, México e Colômbia."

  COMPLICATION: Problema ou mudança que cria tensão
  "Temos P99 latency de 350ms para LATAM (vs 80ms US), 
   e regulação BR exige dados de clientes em território 
   nacional até 2027."

  QUESTION: A questão que a audiência deveria fazer
  "Como podemos servir LATAM com latency aceitável 
   e compliance regulatória?"

  ANSWER: Sua proposta
  "Proposta: deploy multi-region com primary em sa-east-1 
   (São Paulo), replicação active-active para tier-1 
   services. Investment: 6 meses, 3 engenheiros."

  ══════════════════════════════════════════════════════════════
  2. MINTO PYRAMID (Top-Down)
  ══════════════════════════════════════════════════════════════

  Melhor para: written communication, RFCs, decision docs

  ESTRUTURA: conclusão primeiro, depois supporting arguments

                    ┌─────────────────┐
                    │ RECOMENDAÇÃO    │ ← Start here
                    │ "Migrar para    │
                    │  Kafka"         │
                    └──────┬──────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
  ┌───────┴───────┐ ┌─────┴──────┐ ┌──────┴───────┐
  │ Argumento 1   │ │Argumento 2 │ │ Argumento 3  │
  │ "Scale:       │ │"Replay:    │ │ "Ecosystem:  │
  │  200K msg/s"  │ │ audit req" │ │  Schema Reg" │
  └───────┬───────┘ └─────┬──────┘ └──────┬───────┘
          │               │               │
     ┌────┴────┐     ┌────┴────┐     ┌────┴────┐
     │Evidence │     │Evidence │     │Evidence │
     │Benchmark│     │Compliance│    │Industry │
     │data     │     │requirement│   │adoption │
     └─────────┘     └──────────┘    └─────────┘

  POR QUE TOP-DOWN:
  → Liderança tem tempo limitado
  → Se concordam: não precisam ler os detalhes
  → Se discordam: vão ler os argumentos
  → Bottom-up ("vou contar a história toda") = ninguém lê

  ══════════════════════════════════════════════════════════════
  3. DISAGREE AND COMMIT (Bezos)
  ══════════════════════════════════════════════════════════════

  Quando usar: decisão tomada, nem todos concordam

  REGRA: 
  → Debata abertamente até a decisão
  → Uma vez decidido: TODOS commitam, mesmo quem discordou
  → "I disagree, but I'll commit fully to making this work"
  
  COMO LÍDER:
  → "Eu ouvi os argumentos de todos"
  → "A decisão é X, por estas razões"
  → "Se você discorda, por favor registre no ADR 
     e commite na execução"
  → "Vamos medir em 3 meses e ajustar se necessário"

  ══════════════════════════════════════════════════════════════
  4. OPTIONS FRAMEWORK
  ══════════════════════════════════════════════════════════════

  Melhor para: propostas a leadership (nunca 1 opção só)

  SEMPRE APRESENTE 3 OPÇÕES:
  
  │ Dimensão    │ Option A     │ Option B      │ Option C     │
  │             │ (conservador)│ (RECOMENDADO)  │ (agressivo)  │
  │ Scope       │ Minimal      │ Balanced       │ Full scope   │
  │ tempo       │ 4 weeks      │ 8 weeks        │ 16 weeks     │
  │ Custo       │ $50K         │ $120K          │ $300K        │
  │ Risco       │ Baixo        │ Médio          │ Alto         │
  │ Outcome     │ Partial fix  │ Solves 80%     │ Solves 100%  │
  │ Trade-off   │ Debt remains │ Sustainable    │ Over-invest? │
  
  POR QUE 3 OPÇÕES:
  → 1 opção = "posso aprovar ou negar" (binário)
  → 3 opções = "qual eu prefiro?" (engajamento)
  → Ancora no meio = Option B costuma ser aceita
  → Mostra que você pensou em trade-offs
```

---

## Pre-wiring — Alinhamento Antes da Reunião

```
PRE-WIRING — A TÉCNICA MAIS PODEROSA:

  "Se você está surpreso com o resultado de uma reunião,
   o problema aconteceu ANTES da reunião."

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  ERRADO:                                                │
  │  ┌──────────────────────────────────┐                  │
  │  │ 1. Escrever RFC                  │                  │
  │  │ 2. Marcar reunião com 15 pessoas │                  │
  │  │ 3. Apresentar                    │                  │
  │  │ 4. Receber 15 opiniões conflitantes│                │
  │  │ 5. Reunião acaba sem decisão     │                  │
  │  │ 6. Repetir semana que vem        │                  │
  │  └──────────────────────────────────┘                  │
  │                                                         │
  │  CERTO (pre-wiring):                                    │
  │  ┌──────────────────────────────────┐                  │
  │  │ 1. Escrever RFC draft            │                  │
  │  │ 2. 1:1 com CTO — get input      │                  │
  │  │ 3. 1:1 com Principal — refine    │                  │
  │  │ 4. 1:1 com EM skeptic — address  │                  │
  │  │    concerns                      │                  │
  │  │ 5. Update RFC com todos os inputs│                  │
  │  │ 6. Marcar reunião formal         │                  │
  │  │ 7. Reunião confirma decisão      │                  │
  │  │    (já alinhada)                 │                  │
  │  └──────────────────────────────────┘                  │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  SCRIPT PARA 1:1 DE PRE-WIRING:

  "Oi [Stakeholder], estou trabalhando em [proposta X].
   Antes de formalizar, queria sua perspectiva sobre:
   1. O problema faz sentido da sua visão?
   2. A solução proposta tem gaps óbvios?
   3. O que preciso endereçar para ter seu suporte?
   Posso compartilhar o draft para você olhar com calma
   antes da reunião formal dia [data]."

  POR QUE FUNCIONA:
  → Stakeholder se sente ouvido (não "surpreendido")
  → Você incorpora feedback cedo (RFC fica melhor)
  → Evita surpresas em reuniões grandes
  → Build champions antes da decisão formal
  → Se alguém vai bloquear, você sabe antes

  REGRA: investir 60% do tempo em pre-wiring,
  30% em escrever o documento, 10% na reunião formal.
```

---

## Escrita Persuasiva

```
ESCRITA PERSUASIVA — A ARMA DO IC LEADER:

  "Staff+ Engineers influence primarily through writing."
  — Tanya Reilly

  ══════════════════════════════════════════════════════════════
  PRINCÍPIOS DA ESCRITA PERSUASIVA TÉCNICA:
  ══════════════════════════════════════════════════════════════

  1. LEAD WITH THE CONCLUSION
  ┌────────────────────────────────────────────────────────┐
  │ ❌ "Analisamos 5 message brokers ao longo de 3        │
  │     semanas, considerando latency, throughput,         │
  │     cost... [2 páginas depois] ...por isso escolhemos│
  │     Kafka."                                            │
  │                                                        │
  │ ✅ "Recomendamos migrar para Apache Kafka como event  │
  │     backbone. Este documento apresenta o business      │
  │     case, alternativas avaliadas e plano de migração." │
  └────────────────────────────────────────────────────────┘

  2. QUANTIFY EVERYTHING
  ┌────────────────────────────────────────────────────────┐
  │ ❌ "O build é lento"                                   │
  │ ✅ "Build P95 é 45 min. Meta: < 15 min. Engineers     │
  │     perdem ~2h/dia esperando CI. Equivale a $380K/ano │
  │     em productivity loss (estimativa: 30 eng × $190K   │
  │     × 5% tempo)."                                      │
  └────────────────────────────────────────────────────────┘

  3. ACKNOWLEDGE TRADE-OFFS
  ┌────────────────────────────────────────────────────────┐
  │ ❌ "Kafka é claramente a melhor escolha."              │
  │ ✅ "Kafka adds operational complexity (3 ZK nodes +   │
  │     brokers). Mitigação: usar MSK managed.             │
  │     Trade-off: $400/mês a mais vs self-hosted."       │
  └────────────────────────────────────────────────────────┘

  4. USE CONCRETE EXAMPLES
  ┌────────────────────────────────────────────────────────┐
  │ ❌ "Event-driven architecture vai melhorar o sistema." │
  │ ✅ "Hoje payment-service chama notification-service    │
  │     sincrono. Se notification está offline, payment    │
  │     retorna 500. Com eventos: payment publica no       │
  │     Kafka, notification consome quando disponível.     │
  │     Payment nunca falha por notification."             │
  └────────────────────────────────────────────────────────┘

  5. ADDRESS THE READER'S CONCERNS
  ┌────────────────────────────────────────────────────────┐
  │ Para ENGINEERS: "como isto muda meu dia a dia?"       │
  │ Para EMs: "quanto capacity precisa?" e "quem?"        │
  │ Para VPs: "quanto custa?" e "qual o risco?"           │
  │ Para CTO: "como se alinha com tech strategy?"         │
  └────────────────────────────────────────────────────────┘

  ══════════════════════════════════════════════════════════════
  TEMPLATES DE ESCRITA POR TIPO:
  ══════════════════════════════════════════════════════════════

  PROPOSAL (1-2 páginas):
  → TLDR (3 linhas)
  → Problem (com números)
  → Proposed Solution
  → Options Considered (table)
  → Recommendation + Why
  → Next Steps

  STRATEGY DOC (3-5 páginas):
  → Executive Summary
  → Context (business + technical)
  → Current State (with metrics)
  → Target State (with vision)
  → Gap Analysis
  → Strategic Initiatives
  → Timeline + Milestones
  → Risks + Mitigations

  DECISION DOC (1 página):
  → Decision: [1 sentence]
  → Context: [3 sentences]
  → Options: [table]
  → Decision Rationale
  → Consequences + Trade-offs
  → Review Date
```

---

## Apresentações para Liderança

```
APRESENTANDO PARA C-LEVEL / VP:

  ══════════════════════════════════════════════════════════════
  REGRAS DE OURO:
  ══════════════════════════════════════════════════════════════

  1. TEMPO: você tem 5 minutos (mesmo que marcou 30)
     → Slide 1: recomendação + ask
     → Slide 2: business impact ($$$ / risk)
     → Slide 3: options (A, B, C)
     → Slides 4+: backup (só se perguntarem)

  2. LINGUAGEM EXECUTIVA:
     ❌ "O sistema usa REST síncrono que causa acoplamento
         temporal com circuit breaker timeout de 30s e 
         retry exponencial."
     ✅ "Quando o sistema de notificações falha, os 
         pagamentos param — afetando $2M/dia em GMV.
         Proposta: desacoplar em 6 semanas, cost: $50K."

  3. FRAMEWORK: WHAT → SO WHAT → NOW WHAT
     WHAT: "Build time é 45 min"
     SO WHAT: "Engineers perdem 2h/dia, equivale a $380K/ano"
     NOW WHAT: "Investir 8 weeks, reduce para 15 min, ROI 3.2x"

  ══════════════════════════════════════════════════════════════
  FORMATO EXECUTIVO:
  ══════════════════════════════════════════════════════════════

  SLIDE 1: HEADLINE + ASK
  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │  "RECOMENDAÇÃO: Investir 6 semanas em                  │
  │   CI/CD optimization para reduzir build time           │
  │   de 45min → 15min"                                    │
  │                                                        │
  │  ASK: Aprovar dedicação de 2 engineers por 6 semanas   │
  │  ROI: $380K/ano em productivity + 4x deploy frequency  │
  │                                                        │
  └────────────────────────────────────────────────────────┘

  SLIDE 2: BUSINESS IMPACT
  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │  │ Métrica         │ Atual    │ Target   │ Impacto     │
  │  │ Build time P95  │ 45 min   │ 15 min   │ -67%        │
  │  │ Deploy freq     │ 0.5/dia  │ 4/dia    │ +700%       │
  │  │ Eng time wasted │ 2h/dia   │ 20min/dia│ -83%        │
  │  │ Annual savings  │ --       │ $380K    │ ROI 3.2x    │
  │                                                        │
  └────────────────────────────────────────────────────────┘

  SLIDE 3: OPTIONS
  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │  │          │ A: Minimal │ B: RECOMENDADO │ C: Full   │
  │  │ Scope    │ Cache only │ Cache+parallel │ + runners │
  │  │ Effort   │ 2 weeks    │ 6 weeks        │ 12 weeks  │
  │  │ Result   │ 30 min     │ 15 min         │ 8 min     │
  │  │ Risk     │ Low        │ Medium         │ High      │
  │                                                        │
  └────────────────────────────────────────────────────────┘

  COMO LIDAR COM PERGUNTAS:

  "Não sei": → "Boa pergunta. Vou pesquisar e voltar 
               em 24h com resposta."
  
  "E se...": → "Ótimo cenário. No worst case [X], 
               mitigação seria [Y]."
  
  "Quanto custa?":→ Sempre ter números prontos 
                    (mesmo que estimativa rough)
  
  "Por que agora?": → Conecte com business urgência
                      "Porque em Q3 temos [evento] e 
                       precisamos estar ready"
```

---

## Navegando Resistência e Conflito

```
LIDANDO COM RESISTÊNCIA — TÉCNICAS POR TIPO:

  ══════════════════════════════════════════════════════════════
  TIPO 1: "Não concordo tecnicamente"
  ══════════════════════════════════════════════════════════════

  → OUÇA a objeção completa (não interrompa)
  → REPITA para confirmar entendimento: 
    "Se entendi, sua concern é [X]. Correto?"
  → DADOS: traga benchmarks, provas de conceito, exemplos
  → Se legítima: ADAPTE a proposta
  → Se divergência de opinião: escale para decision maker 
    com os dois pontos de vista documentados

  ══════════════════════════════════════════════════════════════
  TIPO 2: "Não temos tempo/capacity"
  ══════════════════════════════════════════════════════════════

  → Quantifique custo de NÃO fazer
  → Mostre o custo CRESCENTE (debt compounds)
  → Proponha versão menor (MVP da melhoria)
  → Identifique overlap: "já vamos mexer nesse código 
    para feature X — podemos incluir melhoria Y"

  ══════════════════════════════════════════════════════════════
  TIPO 3: "Já tentamos e não funcionou"
  ══════════════════════════════════════════════════════════════

  → Investigue: o que exatamente tentaram? O que falhou?
  → Mostre o que mudou desde então (nova tech, novo contexto)
  → Proponha abordagem diferente com mitigação específica
    para o que falhou antes
  → Piloto pequeno: "deixe-me provar em 2 semanas"

  ══════════════════════════════════════════════════════════════
  TIPO 4: "Político" (resistência por territorialidade/ego)
  ══════════════════════════════════════════════════════════════

  SINAIS:
  → Objeções vagas que mudam quando você resolve
  → "Concordo mas não agora" repetidamente
  → Bloqueio sem razão técnica clara

  TÁTICAS:
  → Dê ownership parcial: "co-author do RFC"
  → Dê crédito: "building on [Person]'s earlier work"
  → 1:1 privado: "estou sentindo resistência, pode me
    ajudar a entender o que posso melhorar?"
  → Se persistir: sponsor/VP para destravar
  → NUNCA: confrontar em público

  ══════════════════════════════════════════════════════════════
  CONFLITO PRODUTIVO vs DESTRUTIVO
  ══════════════════════════════════════════════════════════════

  PRODUTIVO:                         DESTRUTIVO:
  ┌─────────────────────┐           ┌─────────────────────┐
  │ Debate a ideia      │           │ Ataca a pessoa       │
  │ "Essa abordagem     │           │ "Você não entende    │
  │  tem risco X"       │           │  distributed systems"│
  │                     │           │                      │
  │ Foco em dados       │           │ Foco em opinião      │
  │ "Benchmark mostra"  │           │ "Eu acho que"        │
  │                     │           │                      │
  │ Propõe alternativa  │           │ Só critica           │
  │ "E se fizermos..."  │           │ "Isso não funciona"  │
  │                     │           │                      │
  │ Busca resolução     │           │ Busca "vencer"       │
  │ "Como chegamos a    │           │ "Eu estava certo"    │
  │  um acordo?"        │           │                      │
  └─────────────────────┘           └─────────────────────┘
```

---

## Sponsors vs Mentors

```
SPONSOR vs MENTOR — A DIFERENÇA MAIS IMPORTANTE:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  MENTOR                          SPONSOR                │
  │                                                         │
  │  "Aqui está o que eu faria"      "Vou te colocar       │
  │                                   nesse projeto"        │
  │  Conversa 1:1 com você           Fala sobre você em    │
  │                                   salas que você não    │
  │                                   está                  │
  │  Dá conselho                     Dá oportunidade       │
  │                                                         │
  │  Você procura o mentor           Sponsor procura VOCÊ  │
  │                                   (ou você conquista)  │
  │                                                         │
  │  Qualquer seniority              Precisa ser 2+ níveis │
  │                                   acima (poder real)   │
  │                                                         │
  │  Nice to have                    ESSENTIAL para Staff+  │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  COMO CONSEGUIR SPONSORS:

  1. VISIBILIDADE
  → Faça trabalho impactful e VISÍVEL
  → Share wins em all-hands, newsletters
  → Write docs que circulam na organização
  → Apresente em tech talks, design reviews

  2. CONFIABILIDADE
  → Entregue o que prometeu, no prazo
  → Se vai atrasar, comunique cedo
  → Faça follow-through (não abandone projetos)
  → Build track record de execução

  3. SOLVE THEIR PROBLEMS
  → Entenda o que seu VP/Director está tentando resolver
  → Proponha soluções (mesmo que não pediu)
  → "Vi que estamos com challenge X. Tenho uma proposta."
  → Make their life easier

  4. ASK
  → "Gostaria de operar em scope de [Staff/Principal].
     Que projeto posso liderar para demonstrar isso?"
  → "Preciso de visibilidade em [área]. Pode me incluir?"

  SPONSOR POR NÍVEL:

  │ Você é      │ Sponsor ideal         │ Como conquistar              │
  │ Senior      │ Director / Sr Staff   │ Destaque em delivery + RFCs  │
  │ Tech Lead   │ VP Eng / Sr Staff     │ Impact metrics do time       │
  │ Staff       │ VP Eng / CTO          │ Cross-team impact visible    │
  │ Principal   │ CTO / CEO             │ Business-level impact        │
  │ EM          │ VP Eng / Director     │ Team health + delivery       │
  │ Tech Manager│ CTO / VP Product      │ Org design + strategy results│
```

---

## Influência por Nível

```
COMO INFLUENCIAR — TÁTICAS POR NÍVEL:

  ══════════════════════════════════════════════════════════════
  TECH LEAD
  ══════════════════════════════════════════════════════════════

  AUDIÊNCIA: time (5-8 eng) + PM + designer
  TOOL: decisões técnicas + code review + daily interaction

  TÁTICAS:
  → Code review como teaching moment (não só approve/reject)
  → Pair programming para alinhar padrões
  → Sprint retros: propor melhorias com dados
  → ADRs: documentar decisões para criar precedente
  → 1:1 com PM: educar sobre trade-offs técnicos
  → Demo day: mostrar impacto técnico para stakeholders

  ARMADILHA: usar autoridade em vez de influência
  → ❌ "Vai ser assim porque eu sou o Tech Lead"
  → ✅ "Recomendo X porque [dados]. O que vocês acham?"

  ══════════════════════════════════════════════════════════════
  STAFF ENGINEER
  ══════════════════════════════════════════════════════════════

  AUDIÊNCIA: 2-3 times + engineering management
  TOOL: RFCs + design reviews + working groups

  TÁTICAS:
  → RFC process: propor e iterar com feedback
  → Design reviews: ser o reviewer que agrega valor
  → Guilds/chapters: criar espaço de discussão cross-team
  → Writing: blog posts internos, tech talks
  → Protótipos: provar que funciona antes de propor
  → Pre-wiring: 1:1s com key stakeholders antes de propostas

  SUPERPOWER: Technical credibility + organizational awareness
  → Sabe o que é tecnicamente certo E o que é organizacionalmente viável

  ══════════════════════════════════════════════════════════════
  PRINCIPAL ENGINEER
  ══════════════════════════════════════════════════════════════

  AUDIÊNCIA: org inteira + C-level
  TOOL: vision docs + Tech Radar + strategic RFCs

  TÁTICAS:
  → Storytelling: narrativa que conecta tech a business
  → Data-driven: DORA metrics, incident trends, cost analysis
  → Coalition building: alinhar Staff Engineers como advocates
  → External credibility: conf talks, blog, open source
  → 1:1 com CTO/VP: trusted advisor relationship
  → Selective coding: protótipos que provam feasibility

  SUPERPOWER: Trusted voice at every level
  → C-level confia no julgamento técnico
  → Engineers respeitam pelo track record

  ══════════════════════════════════════════════════════════════
  TECH MANAGER
  ══════════════════════════════════════════════════════════════

  AUDIÊNCIA: EMs + VP + product leadership
  TOOL: org design + process + OKRs + budget

  TÁTICAS:
  → Data storytelling: metrics que mostram impacto
  → Budget proposals: ROI-based investment cases
  → Org design: reshape teams para habilitar strategy
  → Process improvement: DORA metrics como evidence
  → Cross-functional: alianças com Product, Design, Data
  → Hiring: build teams que execute a vision

  SUPERPOWER: Organizational leverage
  → Controla budget, headcount, team topology
  → Can remove blockers that ICs cannot
```

---

## Anti-Patterns

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **Technically right, organizationally wrong** | Proposta perfeita que ninguém adota | Pre-wire, build coalition, adapt to context |
| **Email warrior** | Só escreve, nunca faz 1:1 | Balance written + face-to-face communication |
| **Hero complex** | "Só eu posso convencer" | Build advocates, give credit, share ownership |
| **Surprise RFC** | Proposta sem pre-wiring | Always pre-wire with key stakeholders |
| **Data without story** | Apresenta números sem narrativa | SCQA framework, connect to business impact |
| **Ignore the skeptic** | Evita quem discorda | Engage 1:1, incorporate feedback, acknowledge concerns |
| **Title-reliant** | Usa título como argumento | Influence through expertise, data, and relationships |
| **Give up too early** | Proposta rejeitada = desiste | Adapt, retry with better data, find another angle |
| **Publicly confronts** | Debate agressivo em meetings | Move contentious discussions to 1:1 |
| **All talk, no execution** | Convence mas não entrega | Credibility = influence history × follow-through |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e decisões, considere o aspecto de influência organizacional:

1. **ADR presente** — Decisão significativa sem ADR = oportunidade perdida de influência; sugerir documentação
2. **Stakeholder acknowledgment** — Se PR afeta outros times, verificar se houve comunicação prévia
3. **Options documented** — Mudança com 1 opção avaliada = pouca persuasão; sugerir documentar alternativas
4. **Business connection** — PRs de tech investment sem conexão com business value = difícil defender; sugerir justificativa
5. **Quantificação** — "Melhora performance" sem números = fraco; exigir benchmark before/after
6. **RFC linkado** — Mudança cross-team sem RFC = processo incompleto
7. **Pre-wiring evidence** — RFC que já incorpora feedback de stakeholders é mais provável ser aprovado
8. **Credit sharing** — Docs que reconhecem contribuições de outros times = build goodwill
9. **Trade-offs explícitos** — Proposta que ignora trade-offs = menos credível; exigir honest assessment
10. **Follow-through tracking** — Se RFC foi aprovado, verificar que métricas de sucesso estão sendo tracked

---

## Referências

- **Staff Engineer** — Will Larson (2021) — https://staffeng.com/
- **The Staff Engineer's Path** — Tanya Reilly (O'Reilly, 2022)
- **Radical Candor** — Kim Scott (St. Martin's Press, 2017)
- **Crucial Conversations** — Kerry Patterson et al. (McGraw-Hill, 2011)
- **The Minto Pyramid Principle** — Barbara Minto (Pearson, 2008)
- **Influence Without Authority** — Allan R. Cohen & David L. Bradford (Wiley, 2005)
- **The Hard Thing About Hard Things** — Ben Horowitz (HarperBusiness, 2014)
- **Thinking in Bets** — Annie Duke (Portfolio, 2018)
- **Thanks for the Feedback** — Douglas Stone & Sheila Heen (Penguin, 2015)
- **Disagreeing and Committing** — Jeff Bezos (2016 Letter to Shareholders)
- **Writing Technical Design Documents** — https://www.industrialempathy.com/posts/design-docs-at-google/
- **Being Glue** — Tanya Reilly — https://noidea.dog/glue
- **LeadDev** — https://leaddev.com/
