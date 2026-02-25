# Written Communication — Documentos que Influenciam Decisões

> **Objetivo deste documento:** Servir como referência completa sobre **comunicação escrita para líderes técnicos** — como escrever RFCs, design docs, ADRs, post-mortems e blog posts que influenciam decisões e escalam conhecimento — otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: writing frameworks, design docs, RFCs, ADRs, post-mortems, blog posts internos, escrita para diferentes audiências.

---

## Quick Reference — Cheat Sheet

| Tipo de documento | Propósito | Audiência | Tamanho |
|-------------------|----------|-----------|---------|
| **Design Doc** | Propor solução técnica para problema | Time + stakeholders | 3-10 páginas |
| **RFC** | Propor decisão cross-team para discussão | Org engineering | 5-15 páginas |
| **ADR** | Registrar decisão tomada + contexto | Futuro-eu / time | 1-2 páginas |
| **Post-mortem** | Documentar incidente + aprendizados | Toda eng + management | 2-5 páginas |
| **Tech Blog Post** | Compartilhar conhecimento publicamente | Engenheiros (int/ext) | 1500-3000 palavras |
| **One-pager** | Resumir proposta para leadership | Executives | 1 página |

---

## Sumário

- [Written Communication — Documentos que Influenciam Decisões](#written-communication--documentos-que-influenciam-decisões)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Por Que Escrita É a Skill #1 para Staff+](#por-que-escrita-é-a-skill-1-para-staff)
  - [Princípios de Escrita Técnica](#princípios-de-escrita-técnica)
  - [Design Docs](#design-docs)
  - [RFCs](#rfcs)
  - [ADRs — Architecture Decision Records](#adrs--architecture-decision-records)
  - [Post-mortems](#post-mortems)
  - [One-pagers para Leadership](#one-pagers-para-leadership)
  - [Blog Posts Técnicos](#blog-posts-técnicos)
  - [Papel de Cada Nível na Escrita](#papel-de-cada-nível-na-escrita)
  - [Anti-Patterns](#anti-patterns)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Por Que Escrita É a Skill #1 para Staff+

```
O PODER DA ESCRITA EM ENGINEERING:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  CÓDIGO escala em MÁQUINAS.                             │
  │  FALA escala em 1 SALA.                                 │
  │  ESCRITA escala em TODA A ORGANIZAÇÃO.                  │
  │                                                         │
  │  ─────────────────────────────────────────────────────  │
  │                                                         │
  │  Senior: influencia com CÓDIGO                          │
  │  Staff:  influencia com DOCUMENTOS + CÓDIGO             │
  │  Principal: influencia com DOCUMENTOS + PRESENÇA        │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  POR QUE ISSO IMPORTA:

  1. INFLUÊNCIA ASSÍNCRONA
     → Você não precisa estar na sala para influenciar
     → Um RFC bem escrito convence 50 engineers sem meeting
     → Timezone-friendly (remote/distributed teams)

  2. PENSAMENTO RIGOROSO
     → Escrever FORÇA you a pensar com clareza
     → Se não consegue explicar por escrito, não entendeu
     → "Writing is thinking" — William Zinsser

  3. INSTITUCIONAL MEMORY
     → Pessoas saem, documentos ficam
     → ADR de 2021 explica decisão que ninguém lembra
     → Onboarding: novos engineers leem docs, não perguntam

  4. ACCOUNTABILITY
     → Documento escrito = compromisso registrado
     → "Dissemos no RFC que X" vs "alguém mencionou X"
     → Post-mortem registra que prometemos corrigir Y

  SINAIS DE BOA ESCRITA TÉCNICA:
  ✅ Leitor sabe o que fazer depois de ler o primeiro parágrafo
  ✅ Não-experts entendem o problema (mesmo que não a solução)
  ✅ Alternativas estão presentes com trade-offs claros
  ✅ Documento é referenciado meses depois
  ✅ Gera decisão, não apenas discussão

  SINAIS DE ESCRITA RUIM:
  ❌ Leitor não sabe a conclusão até a última página
  ❌ Jargão impenetrável sem glossário
  ❌ Só 1 opção apresentada (sem alternativas)
  ❌ "O que exatamente esse doc está pedindo?"
  ❌ Ninguém referencia depois de aprovado
```

---

## Princípios de Escrita Técnica

```
7 PRINCÍPIOS PARA ESCRITA QUE INFLUENCIA:

  ══════════════════════════════════════════════════════════
  1. CONCLUSÃO PRIMEIRO (INVERTED PYRAMID)
  ══════════════════════════════════════════════════════════

  ❌ NARRATIVO (academia):  Contexto → Análise → Conclusão
  ✅ INVERTIDO (business):  Conclusão → Evidência → Contexto

  EXEMPLO:
  ❌ "Analisamos 5 opções de message broker ao longo de 
     3 semanas. Avaliamos throughput, latency, custo...
     [3 páginas depois] ...recomendamos Kafka."

  ✅ "Recomendamos Kafka como message broker padrão.
     Custo: R$X/mês. Migração: 8 weeks.
     Alternativas avaliadas: RabbitMQ, SQS, Pulsar.
     [detalhes abaixo para quem quiser]"

  ══════════════════════════════════════════════════════════
  2. QUANTIFIQUE TUDO
  ══════════════════════════════════════════════════════════

  ❌ "Performance vai melhorar significativamente"
  ✅ "P95 latency reduz de 800ms para 200ms (75%)"

  ❌ "O projeto vai demorar um tempo"
  ✅ "Estimativa: 8 semanas com 2 engineers"

  ❌ "Vários clientes foram afetados"
  ✅ "1,247 clientes experienciaram errors (3.2% do total)"

  ══════════════════════════════════════════════════════════
  3. SEMPRE APRESENTE ALTERNATIVAS
  ══════════════════════════════════════════════════════════

  REGRA: mínimo 3 opções (incluindo "não fazer nada")

  │ Opção │ Custo │ Benefício │ Risco │ Recomendação │
  │ A: Do nothing │ R$0 │ None │ Status quo degrada │ ❌ │
  │ B: Quick fix │ R$30K │ 80% solve │ Temporary │ ─ │
  │ C: Full solution │ R$150K │ 100% solve │ 3 months │ ✅ │

  POR QUE: Decisores querem ESCOLHER, não ser TOLD.
  Dar opção cria buy-in.

  ══════════════════════════════════════════════════════════
  4. ESCREVA PARA O LEITOR MAIS OCUPADO
  ══════════════════════════════════════════════════════════

  → Executive summary no TOPO (3-5 linhas)
  → TL;DR para cada seção
  → Bold nas frases-chave
  → Hierarquia visual: headings, bullets, tabelas
  → Se cortasse 50%, o documento ainda faria sentido?

  ══════════════════════════════════════════════════════════
  5. MOSTRE OS TRADE-OFFS
  ══════════════════════════════════════════════════════════

  ❌ "Kafka é a melhor opção"
  ✅ "Kafka é a melhor opção para throughput alto, mas é
     operacionalmente complexo. Se não precisarmos de 
     >100K msg/s, SQS é mais simples e managed."

  TRADE-OFF TEMPLATE:
  "Escolhemos X porque [razão 1] e [razão 2].
   A desvantagem é [trade-off].
   Mitigamos isso com [medida].
   Rejeitamos Y porque [razão] apesar de [vantagem de Y]."

  ══════════════════════════════════════════════════════════
  6. USE DIAGRAMS COMO PRIMEIRA LINGUAGEM
  ══════════════════════════════════════════════════════════

  → Arquitetura: system context diagram SEMPRE
  → Fluxo: sequence diagram para integrações
  → Dados: ER diagram para models
  → Decisão: decision tree diagram

  1 diagrama = 1000 palavras de explicação
  Coloque diagrams ANTES do texto que explica

  ══════════════════════════════════════════════════════════
  7. RESPONDA AS PERGUNTAS ANTES DELAS SEREM FEITAS
  ══════════════════════════════════════════════════════════

  Antes de publicar, pergunte-se:
  → "Por que não usar [alternativa óbvia]?" — já respondi?
  → "Quanto custa?" — já está claro?
  → "E se der errado?" — rollback plan está descrito?
  → "Quanto tempo?" — timeline está definida?
  → "Quem é afetado?" — stakeholders estão identificados?
```

---

## Design Docs

```
DESIGN DOC — O DOCUMENTO CENTRAL DE ENGINEERING:

  QUANDO ESCREVER:
  ✅ Novo serviço ou componente
  ✅ Mudança arquitetural significativa
  ✅ Feature que dura mais que 2 sprints
  ✅ Qualquer coisa que precisa de design review
  
  QUANDO NÃO ESCREVER:
  ❌ Bug fix (use post-mortem se outage)
  ❌ Refactor pequeno (code review é suficiente)
  ❌ Feature simples de 1 sprint (ticket basta)

  TEMPLATE:

  ┌────────────────────────────────────────────────────────┐
  │ DESIGN DOC: [Título Descritivo]                        │
  │                                                        │
  │ Author: [nome]  |  Date: [data]  |  Status: [WIP/Review/Approved]
  │ Reviewers: [lista]                                     │
  │ Related: [links para tickets, RFCs, ADRs]              │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ TL;DR (3-5 linhas)                                     │
  │ ═══════════════════════════════════════════════════     │
  │ [Resumo da proposta, decisão principal, impacto]       │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ 1. CONTEXTO E PROBLEMA                                 │
  │ ═══════════════════════════════════════════════════     │
  │ → Qual problema estamos resolvendo?                    │
  │ → Por que agora? (business driver)                     │
  │ → Métricas atuais (estado AS-IS)                       │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ 2. GOALS E NON-GOALS                                   │
  │ ═══════════════════════════════════════════════════     │
  │ Goals:                                                 │
  │ → [O que essa proposta resolve]                        │
  │ Non-goals:                                             │
  │ → [O que explicitamente NÃO resolve / out of scope]   │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ 3. PROPOSTA (SOLUÇÃO)                                  │
  │ ═══════════════════════════════════════════════════     │
  │ [Descrição da solução com diagrams]                    │
  │ → System architecture diagram                          │
  │ → Sequence diagram (fluxos principais)                 │
  │ → Data model (se aplicável)                            │
  │ → API contracts (se aplicável)                         │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ 4. ALTERNATIVAS CONSIDERADAS                           │
  │ ═══════════════════════════════════════════════════     │
  │ │ Opção      │ Prós       │ Contras    │ Decisão   │  │
  │ │ A: [...]   │ [...]      │ [...]      │ Escolhida │  │
  │ │ B: [...]   │ [...]      │ [...]      │ Rejeitada │  │
  │ │ C: Não fazer│ [...]     │ [...]      │ Baseline  │  │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ 5. IMPACTO OPERACIONAL                                 │
  │ ═══════════════════════════════════════════════════     │
  │ → Observability: métricas, logs, traces                │
  │ → Alerting: que alarmes criar                          │
  │ → Runbooks: como operar/troubleshoot                   │
  │ → Rollback: como reverter se der errado               │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ 6. CUSTOS E TIMELINE                                   │
  │ ═══════════════════════════════════════════════════     │
  │ → Estimativa de custo (infra + eng effort)             │
  │ → Fases e milestones                                   │
  │ → Dependencies                                         │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ 7. OPEN QUESTIONS                                      │
  │ ═══════════════════════════════════════════════════     │
  │ → [Perguntas que ainda precisa responder]              │
  │ → [Onde espera feedback dos reviewers]                 │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ 8. REFERÊNCIAS                                         │
  │ ═══════════════════════════════════════════════════     │
  │ → [Links para docs, artigos, RFCs relacionados]        │
  └────────────────────────────────────────────────────────┘

  PROCESSO:

  1. Author escreve draft (1-3 dias)
  2. Self-review: "Se eu não soubesse nada, entenderia?"
  3. Envia para 2-3 reviewers (48h para review async)
  4. Coleta feedback, atualiza doc
  5. Design review session (se necessário)
  6. Marca como "Approved" e começa implementação
```

---

## RFCs

```
RFC — REQUEST FOR COMMENTS — DECISÃO ORG-WIDE:

  RFC ≠ DESIGN DOC:
  
  │ Aspecto    │ Design Doc           │ RFC                      │
  │ Scope      │ 1 time/projeto       │ Cross-team / org-wide    │
  │ Decisão    │ "Como implementar X" │ "Devemos adotar Y?"      │
  │ Audiência  │ Time + stakeholders  │ Toda engineering org     │
  │ Lifecycle  │ Approved → implement │ Draft→Review→Decision    │
  │ Outcome    │ Implementation guide │ Policy/Standard/Direction│

  QUANDO ESCREVER RFC:
  ✅ Nova tecnologia no stack da org
  ✅ Padrão que todos os times devem seguir
  ✅ Mudança que afeta 3+ times
  ✅ Investimento significativo (> 2 person-months)
  ✅ Decisão irreversível (Type 1)

  TEMPLATE:

  ┌────────────────────────────────────────────────────────┐
  │ RFC-[número]: [Título]                                 │
  │                                                        │
  │ Author: [nome]                                         │
  │ Status: DRAFT | PROPOSED | REVIEW | ACCEPTED | REJECTED│
  │ Created: [data]  Decision date: [target]               │
  │ Stakeholders: [quem precisa revisar]                   │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ EXECUTIVE SUMMARY (5 linhas max)                       │
  │ ═══════════════════════════════════════════════════     │
  │ [O que propõe, por que importa, impacto esperado]      │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ MOTIVAÇÃO                                              │
  │ ═══════════════════════════════════════════════════     │
  │ → Qual problema existe hoje?                           │
  │ → Quem é afetado e como?                               │
  │ → Dados que suportam o problema (métricas, incidents)  │
  │ → Por que agora? Call to action                        │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ PROPOSTA                                               │
  │ ═══════════════════════════════════════════════════     │
  │ → Descrição detalhada da proposta                      │
  │ → Como funciona na prática (exemplos concretos)        │
  │ → Diagrams de arquitetura / fluxo                      │
  │ → Migration plan para existing code/systems            │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ ALTERNATIVAS                                           │
  │ ═══════════════════════════════════════════════════     │
  │ → Opção A: [proposta principal]                        │
  │ → Opção B: [alternativa viável]                        │
  │ → Opção C: Não fazer nada (status quo)                 │
  │ → Comparação com trade-offs                            │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ IMPACTO                                                │
  │ ═══════════════════════════════════════════════════     │
  │ → Times afetados e como                                │
  │ → Custo estimado (effort + infra)                      │
  │ → Timeline de adoção                                   │
  │ → Riscos e mitigações                                  │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ OPEN QUESTIONS                                         │
  │ ═══════════════════════════════════════════════════     │
  │ → O que o author precisa de input                      │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ DECISION LOG                                           │
  │ ═══════════════════════════════════════════════════     │
  │ [Data] Decision: ACCEPTED / REJECTED                   │
  │ [Data] Rationale: [motivo]                             │
  │ [Data] Action items: [próximos passos]                 │
  └────────────────────────────────────────────────────────┘

  LIFECYCLE DO RFC:

  DRAFT (author escrevendo)
    ↓
  PROPOSED (publicado para comentários)
    ↓ 1-2 semanas de review
  REVIEW (discussão ativa)
    ↓ Pode ter sessão presencial
  DECISION (accepted ou rejected)
    ↓
  IMPLEMENTED (quando execução começa)

  REGRAS DE ENGAJAMENTO:

  1. Review deadline: max 2 semanas
  2. Silence = consent (com notificação prévia)
  3. Comments no doc, NÃO em Slack/email (rastreabilidade)
  4. Disagreement must be constructive (propor alternativa)
  5. Decision owner: author ou nominated decider
```

---

## ADRs — Architecture Decision Records

```
ADR — O REGISTRO MAIS SUBESTIMADO:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  "O valor de um ADR não é para quem escreve.           │
  │   É para quem lê DAQUI A 2 ANOS."                     │
  │                                                         │
  │  Sem ADR: "Por que usamos MongoDB aqui?"               │
  │  → "Acho que o João sabia... mas ele saiu da empresa"  │
  │                                                         │
  │  Com ADR: "ADR-007 explica: PostgreSQL não suportava   │
  │  o schema flexível que precisávamos na época. Hoje     │
  │  com PostgreSQL JSONB, podemos reconsiderar."          │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  TEMPLATE (CONCISO — 1-2 PÁGINAS):

  ┌────────────────────────────────────────────────────────┐
  │ ADR-[número]: [Título da decisão]                      │
  │                                                        │
  │ Status: PROPOSED | ACCEPTED | DEPRECATED | SUPERSEDED  │
  │ Date: [data]                                           │
  │ Deciders: [quem participou]                            │
  │ Supersedes: [ADR anterior, se houver]                  │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ CONTEXTO                                               │
  │ ═══════════════════════════════════════════════════     │
  │ [Qual situação levou a essa decisão?]                  │
  │ [Constraints técnicas e de negócio na época]           │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ DECISÃO                                                │
  │ ═══════════════════════════════════════════════════     │
  │ "Decidimos [ação] porque [razão principal]."           │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ ALTERNATIVAS REJEITADAS                                │
  │ ═══════════════════════════════════════════════════     │
  │ → [Opção B]: Rejeitada porque [razão]                  │
  │ → [Opção C]: Rejeitada porque [razão]                  │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ CONSEQUÊNCIAS                                          │
  │ ═══════════════════════════════════════════════════     │
  │ Positivas:                                             │
  │ → [o que ganhamos]                                     │
  │ Negativas:                                             │
  │ → [o que perdemos / trade-offs aceitos]                │
  │ → [o que precisamos monitorar]                         │
  └────────────────────────────────────────────────────────┘

  EXEMPLO:

  ADR-012: Uso de Kafka como message broker padrão

  Status: ACCEPTED
  Date: 2024-03-15
  Deciders: Ana (Staff), Pedro (Principal), Backend Guild

  CONTEXTO:
  Temos 3 message brokers (RabbitMQ, SQS, Redis pub/sub)
  em diferentes serviços. Manutenção é difícil, cada um
  requer expertise diferentes. Precisamos padronizar.

  DECISÃO:
  Kafka é o message broker padrão para comunicação
  assíncrona entre serviços. Exceções devem ter ADR
  próprio justificando.

  ALTERNATIVAS REJEITADAS:
  → SQS: Simples mas not enough para event sourcing needs
  → RabbitMQ: Bom mas equipe has more Kafka expertise
  → Manter status quo: Custo de manutenção crescente

  CONSEQUÊNCIAS:
  Positivas:
  → Stack única, 1 runbook, 1 set de métricas
  → Event replay capability (Kafka retains events)
  → Melhor hiring (Kafka skills amplamente disponíveis)
  Negativas:
  → Kafka é operacionalmente complexo
  → Need dedicated team/person para managed cluster
  → Overhead para use cases simples (ponto-a-ponto)

  BOAS PRÁTICAS DE ADR:

  1. IMUTÁVEIS: Nunca edite um ADR aceito. Crie novo
     que supersede o anterior.
  2. CONTEXTO: Sempre registre constraints da ÉPOCA.
     Condições mudam, decisão pode ser reavaliada.
  3. NUMBERING: Sequencial (ADR-001, ADR-002, ...)
  4. LOCATION: /docs/adrs/ no repo, versionado
  5. REVIEW: ADRs nascem de discussions (guild, design
     review, RFC), nunca surgem "do nada"
```

---

## Post-mortems

```
POST-MORTEM — TRANSFORMANDO INCIDENTES EM APRENDIZADO:

  PROPÓSITO: Documentar O QUE ACONTECEU, POR QUE aconteceu,
  e O QUE VAMOS FAZER para não repetir. SEM CULPA.

  TEMPLATE:

  ┌────────────────────────────────────────────────────────┐
  │ POST-MORTEM: [Nome do incidente]                       │
  │                                                        │
  │ Date: [data/hora início → fim]                         │
  │ Duration: [duração total]                              │
  │ Severity: SEV1 / SEV2 / SEV3                          │
  │ Author: [nome]                                         │
  │ Facilitator: [nome, se houve sessão]                   │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ EXECUTIVE SUMMARY (5 linhas)                           │
  │ ═══════════════════════════════════════════════════     │
  │ [O que aconteceu, impacto, como resolveu]              │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ IMPACTO                                                │
  │ ═══════════════════════════════════════════════════     │
  │ → Usuários afetados: [número]                          │
  │ → Revenue impactado: [R$]                              │
  │ → Duração do impacto: [horas/minutos]                  │
  │ → SLA burn: [quanto SLA budget consumiu]               │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ TIMELINE (cronológico)                                 │
  │ ═══════════════════════════════════════════════════     │
  │ 14:32 - Deploy da versão 2.3.1                         │
  │ 14:45 - Alarme de error rate > 5% dispara              │
  │ 14:48 - On-call acknowledges                           │
  │ 14:55 - Identificado: query N+1 em novo endpoint      │
  │ 15:02 - Rollback para versão 2.3.0                    │
  │ 15:10 - Métricas normalizam                           │
  │ 15:30 - All clear declarado                            │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ ROOT CAUSE ANALYSIS                                    │
  │ ═══════════════════════════════════════════════════     │
  │ → Causa raiz técnica: [ex: query sem index]            │
  │ → Causa sistêmica: [ex: load test não cobre esse path] │
  │ → Por que não detectamos antes: [ex: staging tem menos │
  │   dados que prod]                                      │
  │                                                        │
  │ 5 WHYS:                                                │
  │ 1. Por que caiu? → DB timeout por query lenta          │
  │ 2. Por que query lenta? → Scan completo sem index      │
  │ 3. Por que sem index? → Schema migration não incluiu   │
  │ 4. Por que não incluiu? → Review não checou query plan │
  │ 5. Por que review não checou? → Checklist não inclui   │
  │    query plan para novas queries                        │
  │                                                        │
  │ CAUSA RAIZ: Nosso code review checklist não inclui     │
  │ verificação de query plan para novas queries.           │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ O QUE DEU CERTO                                        │
  │ ═══════════════════════════════════════════════════     │
  │ → Alarme disparou em < 15 min                          │
  │ → Rollback foi rápido (8 min)                          │
  │ → Comunicação com stakeholders foi clara               │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ O QUE PODE MELHORAR                                    │
  │ ═══════════════════════════════════════════════════     │
  │ → Detection: alarme poderia disparar em < 5 min        │
  │ → Prevention: load test deveria cobrir esse path       │
  │ → Mitigation: circuit breaker teria limitado blast     │
  │                                                        │
  │ ═══════════════════════════════════════════════════     │
  │ ACTION ITEMS (cada um com DRI e deadline)              │
  │ ═══════════════════════════════════════════════════     │
  │ □ Adicionar query plan check ao review checklist       │
  │   DRI: Pedro  | Deadline: 2024-04-01                   │
  │ □ Adicionar load test para novo endpoint               │
  │   DRI: Ana   | Deadline: 2024-04-15                    │
  │ □ Implementar circuit breaker no DB layer              │
  │   DRI: Carlos | Deadline: 2024-04-30                   │
  └────────────────────────────────────────────────────────┘

  PRINCÍPIOS DE POST-MORTEM EFICAZ:

  1. BLAMELESS: "O sistema falhou" não "Maria errou"
  2. FACTUAL: Timeline baseada em logs/métricas, não memória
  3. ACTIONABLE: Todo finding → action item com DRI e date
  4. SYSTEMIC: Buscar causas sistêmicas, não individuais
  5. SHARED: Publicar para toda eng (learning org-wide)
```

---

## One-pagers para Leadership

```
ONE-PAGER — RESUMO EXECUTIVO PARA DECISÃO:

  QUANDO: Precisa de buy-in de VP, Director, C-level

  REGRA: 1 PÁGINA. Se não cabe em 1 página, 
  você não pensou o suficiente.

  TEMPLATE:

  ┌────────────────────────────────────────────────────────┐
  │ ONE-PAGER: [Título claro e direto]                     │
  │ Author: [nome] | Date: [data]                          │
  │                                                        │
  │ ─────────────────────────────────────────────────────  │
  │ PROBLEMA (2-3 linhas)                                  │
  │ [O que está ruim, em termos de negócio, com número]    │
  │                                                        │
  │ ─────────────────────────────────────────────────────  │
  │ PROPOSTA (2-3 linhas)                                  │
  │ [O que quer fazer, resultado esperado com número]      │
  │                                                        │
  │ ─────────────────────────────────────────────────────  │
  │ IMPACTO ESPERADO                                       │
  │ → Revenue/Cost: [R$]                                   │
  │ → Timeline: [weeks/months]                             │
  │ → Effort: [engineers × time]                           │
  │                                                        │
  │ ─────────────────────────────────────────────────────  │
  │ OPÇÕES                                                 │
  │ A: [rápido/barato/parcial]                             │
  │ B: [recomendado/balanceado]                            │
  │ C: [completo/caro/longo]                               │
  │                                                        │
  │ ─────────────────────────────────────────────────────  │
  │ ASK                                                    │
  │ [O que precisa: aprovação, budget, headcount, tempo]   │
  │                                                        │
  │ ─────────────────────────────────────────────────────  │
  │ RISCOS (1-2 linhas)                                    │
  │ [Maior risco + mitigação]                              │
  │                                                        │
  │ ─────────────────────────────────────────────────────  │
  │ NEXT STEPS                                             │
  │ [Se aprovado, o que acontece na segunda-feira]         │
  └────────────────────────────────────────────────────────┘

  DICAS:
  → Use a mesma linguagem que o executive usa
  → Números > palavras ("R$500K" > "significativo")
  → 1 decisão por one-pager (não 3 assuntos)
  → Teste: executive entende o "ask" em 30 segundos?
```

---

## Blog Posts Técnicos

```
BLOG POSTS — AMPLIANDO INFLUÊNCIA VIA ESCRITA:

  POR QUE ESCREVER:
  → Fortalece personal brand (interno e externo)
  → Demonstra expertise (visibilidade para promoção)
  → Beneficia toda a org (learning multiplicado)
  → Clarifica seu próprio pensamento
  → Atrai talento para a empresa

  INTERNAL VS EXTERNAL:

  │ Tipo     │ Público      │ Conteúdo                    │
  │ Internal │ Eng da empresa│ Decisões, learnings, how-tos │
  │ External │ Comunidade   │ Patterns, war stories, guides│
  │          │              │ (sem levar segredos)          │

  TIPOS DE POSTS QUE GERAM IMPACTO:

  1. "HOW WE..." (war story)
     "Como migramos de monolito para microservices em 18 meses"
     → Contexto real, desafios reais, resultados reais
     
  2. "X LESSONS FROM..." (insights)
     "7 lições de 3 anos operando Kafka em produção"
     → Lista digestível, cada item com exemplo concreto

  3. "GUIDE TO..." (tutorial/referência)
     "Guia completo para design docs na [empresa]"
     → Practical, reusable, referenciável

  4. "WHY WE CHOSE..." (thought leadership)
     "Por que escolhemos Go em vez de Java para nosso
      novo serviço de processamento"
     → Trade-offs, decision process, outcome

  5. "POST-MORTEM ANALYSIS" (learning)
     "O que aprendemos com o outage de 4 horas na 
      Black Friday"
     → Transparência, humildade, ação

  ESTRUTURA DE UM POST EFICAZ:

  ┌────────────────────────────────────────────────────────┐
  │ 1. TITLE: específico e searchable                      │
  │    ❌ "Nossa jornada com microsserviços"               │
  │    ✅ "Como reduzimos deployment time de 2h para 15min │
  │        com CI/CD pipeline redesign"                    │
  │                                                        │
  │ 2. HOOK (primeiro parágrafo): problem + resultado     │
  │    "Nosso deployment levava 2 horas e falhava 1 em 5  │
  │     vezes. Redesenhamos o pipeline e agora leva 15    │
  │     min com 99.8% success rate. Aqui está como."      │
  │                                                        │
  │ 3. BODY: contexto → problema → solução → resultados   │
  │    Use headers, code blocks, diagrams                  │
  │    Cada seção começa com TL;DR                        │
  │                                                        │
  │ 4. TAKEAWAYS: 3-5 bullet points acionáveis            │
  │    "Se você está enfrentando problema similar..."      │
  │                                                        │
  │ 5. LENGTH: 1500-3000 palavras (7-15 min de leitura)   │
  └────────────────────────────────────────────────────────┘

  CADÊNCIA RECOMENDADA:

  │ Nível     │ Frequência     │ Onde                       │
  │ Senior    │ 1/quarter      │ Blog interno               │
  │ Staff     │ 1-2/quarter    │ Blog interno + 1/ano ext   │
  │ Principal │ 1/mês          │ Interno + externo regular  │
  │ Tech Mgr  │ 1/quarter      │ Blog interno + eng brand   │
```

---

## Papel de Cada Nível na Escrita

```
COMO CADA NÍVEL CONTRIBUI VIA ESCRITA:

  ══════════════════════════════════════════════════════════
  TECH LEAD — "O Documentador do Time"
  ══════════════════════════════════════════════════════════

  ESCREVE:
  → Design docs para features do time
  → ADRs para decisões do time
  → Runbooks para serviços que o time mantém
  → Code review comments pedagógicos
  
  QUALITY GATE:
  → Garante que todo novo serviço tem design doc
  → Todo ADR tem alternativas documentadas
  → README dos repos está atualizado

  ══════════════════════════════════════════════════════════
  STAFF ENGINEER — "O Influenciador por Escrito"
  ══════════════════════════════════════════════════════════

  ESCREVE:
  → RFCs para mudanças cross-team
  → Design docs complexos (multi-service)
  → Blog posts internos sobre patterns/learnings
  → Post-mortems de incidentes significativos
  → Contribui para vision docs
  
  QUALITY GATE:
  → Review design docs de outros times
  → Feedback detalhado em RFCs (não só "LGTM")
  → Mentora Seniors em escrita (review de seus docs)

  ══════════════════════════════════════════════════════════
  PRINCIPAL ENGINEER — "O Autor de Referência"
  ══════════════════════════════════════════════════════════

  ESCREVE:
  → Vision docs (tech strategy, multi-year direction)
  → RFCs org-wide (standards, governance)
  → Engineering principles e guidelines
  → Blog posts externos (thought leadership)
  → Post-mortems de incidents críticos (framing)
  
  QUALITY GATE:
  → Final review de RFCs antes de decision
  → Define templates e standards de documentação
  → Encoraja cultura de escrita (writing guild, etc.)

  ══════════════════════════════════════════════════════════
  TECH MANAGER — "O Facilitador de Escrita"
  ══════════════════════════════════════════════════════════

  ESCREVE:
  → One-pagers para leadership
  → Quarterly reports / OKR updates
  → Promo packets (articula caso por escrito)
  → Status updates que contam história (não só bullets)
  
  QUALITY GATE:
  → Protege tempo para engineers escreverem
  → Inclui "writing" como expectation em career ladder
  → Review de docs do time (clarity, not just tech)
  → Budget para tools de documentação
```

---

## Anti-Patterns

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **Bottom-up writing** | Conclusão no final, leitor desiste antes | Conclusão no PRIMEIRO parágrafo, sempre |
| **Solution without alternatives** | Só 1 opção = "trust me" | Mínimo 3 opções com trade-offs |
| **Jargon overload** | Apenas experts entendem | Glossário + exemplos concretos |
| **Design doc after implementation** | Doc post-facto é formalidade morta | Doc ANTES de implementar, é a decisão |
| **ADR without context** | "Decidimos usar X" — mas por quê? | Contexto: constraints e razões DA ÉPOCA |
| **Post-mortem without actions** | "Aprendemos muito" mas nada muda | Todo finding → action item com DRI + date |
| **Perfection paralysis** | "Vou polir mais antes de publicar" | 80% draft > 0% published. Ship then iterate |
| **Writing in isolation** | Author escreve 20 páginas sem feedback | Share draft at 30% → iterate, pre-wire key reviewers |
| **LGTM reviews** | "Looks good to me" sem substância | Reviews devem adicionar valor ou questionar |
| **No documentation culture** | "Somos ágeis, não documentamos" | Documentação é produto minimamente viável também |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e documentação, considere o aspecto de comunicação escrita:

1. **PR description quality** — Se PR não descreve "o que", "por que", e "como testar", sugerir melhoria
2. **Commit message clarity** — Commits devem contar uma história; não "fix bug" mas "Fix N+1 query in checkout that caused 500ms P95"
3. **Code comments purpose** — Comments devem explicar "por que", não "o que"; código deve ser auto-explicativo no "o que"
4. **README completeness** — Se novo serviço/módulo não tem README com setup, arquitetura e troubleshooting, sinalizar
5. **ADR trigger** — Se PR introduce nova tecnologia, pattern ou deviation, perguntar "isso deveria ser um ADR?"
6. **Changelog entry** — Se mudança é user-facing ou API change, verificar se há changelog/release notes
7. **Error messages quality** — Error messages devem ser actionable: não "Error 500" mas "Payment processing failed: timeout connecting to gateway. Retry in 30s."
8. **API documentation** — Se endpoint novo não tem OpenAPI spec ou doc, sinalizar gap
9. **Runbook reference** — Se serviço crítico muda behavior, verificar se runbook está atualizado
10. **Design doc linkage** — Se PR implementa feature significativa, link para design doc deveria estar no PR description

---

## Referências

- **The Staff Engineer's Path** — Tanya Reilly (O'Reilly, 2022) — Chapter on communication
- **A Philosophy of Software Design** — John Ousterhout (2018) — "Comments should describe things that are not obvious from the code"
- **On Writing Well** — William Zinsser (Harper Perennial, 2006) — Princípios de escrita clara
- **The Pyramid Principle** — Barbara Minto (Pearson, 2010)
- **Google Design Docs** — https://www.industrialempathy.com/posts/design-docs-at-google/
- **ADR GitHub** — https://adr.github.io/ — Architecture Decision Records
- **RFC Process at Oxide** — https://oxide.computer/blog/rfd-1-requests-for-discussion
- **Blameless Post-mortems** — https://sre.google/sre-book/postmortem-culture/
- **Paul Graham: Write Like You Talk** — http://www.paulgraham.com/talk.html
- **Technical Writing Courses** — Google — https://developers.google.com/tech-writing
- **Gergely Orosz: Writing at Staff+ Level** — https://newsletter.pragmaticengineer.com/
