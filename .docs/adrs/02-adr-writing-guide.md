# Architecture Decision Records — Guia de Escrita

> **Objetivo deste documento:** Servir como referência prática sobre **como escrever ADRs eficazes** — o processo desde a identificação da necessidade até a aprovação, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: processo de escrita, técnicas de argumentação, review process, ferramentas e integração com workflow de engenharia.

---

## Quick Reference — Cheat Sheet

| Fase | Ação-chave | Erro comum | Correção |
|------|-----------|------------|----------|
| **Identificar** | Trigger: mudança que alguém vai perguntar "por que?" em 1 ano | "Não precisa de ADR" para decisão Tipo 1 | Decision matrix: ≥ 1 critério = ADR |
| **Pesquisar** | Spike/PoC antes de decidir | Decidir sem dados | "Não tome decisão Tipo 1 sem evidência" |
| **Escrever** | Context é a seção mais importante | Começar pela decisão, contexto genérico | Escrever contexto primeiro, com dados |
| **Revisar** | Review async (PR) + sync se necessário | Uma pessoa escreve e commita sozinha | ≥ 2 reviewers de times impactados |
| **Aprovar** | Consenso ou consent (não unanimidade) | Esperar que todos concordem 100% | "Alguém tem objeção forte?" (consent-based) |
| **Comunicar** | Socializar: Slack, all-hands, wiki | ADR no repo que ninguém lê | Comunicação ativa nos canais do time |

---

## Sumário

- [Architecture Decision Records — Guia de Escrita](#architecture-decision-records--guia-de-escrita)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Processo de Criação de um ADR](#processo-de-criação-de-um-adr)
  - [Como Escrever Contexto Eficaz](#como-escrever-contexto-eficaz)
  - [Como Estruturar Alternativas](#como-estruturar-alternativas)
  - [Como Escrever a Decisão](#como-escrever-a-decisão)
  - [Como Documentar Consequências](#como-documentar-consequências)
  - [Técnicas de Argumentação](#técnicas-de-argumentação)
  - [Review Process](#review-process)
  - [Modelos de Decisão](#modelos-de-decisão)
  - [Integração com Workflow de Engenharia](#integração-com-workflow-de-engenharia)
  - [Ferramentas e Tooling](#ferramentas-e-tooling)
  - [Escrita Inclusiva e Async-First](#escrita-inclusiva-e-async-first)
  - [Anti-Patterns de Escrita](#anti-patterns-de-escrita)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Processo de Criação de um ADR

```
┌─────────────────────────────────────────────────────────────────┐
│             PROCESSO DE CRIAÇÃO DE UM ADR                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐                                                │
│  │ 1. TRIGGER   │  Alguém identifica que uma decisão             │
│  │              │  arquitetural precisa ser tomada.               │
│  │              │  → Pergunta: "Alguém vai querer saber           │
│  │              │    POR QUE daqui 1 ano?"                        │
│  └──────┬───────┘                                                │
│         │ Sim                                                    │
│  ┌──────▼───────┐                                                │
│  │ 2. RESEARCH  │  Spike, PoC, benchmarks.                       │
│  │              │  → Coletar dados antes de propor.               │
│  │              │  → NUNCA propor Tipo 1 sem evidência.           │
│  └──────┬───────┘                                                │
│         │                                                        │
│  ┌──────▼───────┐                                                │
│  │ 3. DRAFT     │  Autor escreve ADR-NNNN.md                    │
│  │              │  → Status: Proposed                             │
│  │              │  → Foco: Context > Decision > Consequences     │
│  │              │  → Branch: `adr/NNNN-slug-descritivo`          │
│  └──────┬───────┘                                                │
│         │                                                        │
│  ┌──────▼───────┐                                                │
│  │ 4. REVIEW    │  Pull Request com reviewers relevantes.        │
│  │              │  → Async: comentários no PR (2-5 dias)         │
│  │              │  → Sync (se impasse): meeting de 30-60min      │
│  │              │  → Foco do review: contexto completo?          │
│  │              │    alternativas fair? trade-offs honest?        │
│  └──────┬───────┘                                                │
│         │                                                        │
│  ┌──────▼───────┐                                                │
│  │ 5. DECIDE    │  Modelo de decisão: consent ou consenso.       │
│  │              │  → "Alguém tem objeção bloqueante?"            │
│  │              │  → Se não → Status: Accepted. Merge PR.        │
│  │              │  → Se sim → Iterar draft, back to step 4.      │
│  └──────┬───────┘                                                │
│         │                                                        │
│  ┌──────▼───────┐                                                │
│  │ 6. COMMUNICATE│  Socializar a decisão.                        │
│  │              │  → Slack: canal de engenharia + times afetados │
│  │              │  → All-hands / tech talk (decisões grandes)    │
│  │              │  → Wiki: link no hub de decisões               │
│  └──────┬───────┘                                                │
│         │                                                        │
│  ┌──────▼───────┐                                                │
│  │ 7. IMPLEMENT │  PRs de implementação referenciam ADR.         │
│  │              │  → PR description: "Implements ADR-0042"       │
│  │              │  → Code comments: "// See ADR-0042"            │
│  └──────────────┘                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Lead time por tipo de decisão

| Tipo | Research | Draft | Review | Total típico |
|------|---------|-------|--------|-------------|
| **Tipo 1 — cross-team** | 1-4 semanas (spike/PoC) | 2-4 horas | 1-2 semanas (async) | 3-6 semanas |
| **Tipo 1 — single-team** | 1-2 semanas | 1-2 horas | 3-5 dias (async) | 2-3 semanas |
| **Tipo 2 — significativa** | 1-3 dias | 1 hora | 2-3 dias | 1 semana |
| **Tipo 2 — simples** | Horas | 30 min | 1-2 dias | 2-3 dias |

---

## Como Escrever Contexto Eficaz

### O Context é a seção mais importante

```
POR QUE CONTEXT > DECISION?

Se alguém lê daqui 2 anos:
  • A decisão em si é UMA FRASE ("usaremos PostgreSQL")
  • O contexto é o que permite AVALIAR se a decisão ainda vale
  • Se as constraints mudarem, o contexto mostra o que reassess

FRAMEWORK para escrever contexto:

  1. SITUAÇÃO ATUAL
     "Hoje temos X funcionando de Y maneira..."
     Dados: latência, throughput, custo, pain points
  
  2. PROBLEMA / NECESSIDADE
     "Precisamos mudar porque..."
     Drivers: crescimento, custo, complexidade, compliance
  
  3. CONSTRAINTS
     "As restrições são..."
     Budget, timeline, team size, skills, compliance, tech debt
  
  4. URGÊNCIA
     "Se não decidirmos agora..."
     Deadline, custo de delay, risco de não agir
```

### Exemplos de contexto

```
❌ CONTEXTO RUIM:
"Precisamos de um novo banco de dados para o serviço de pedidos."

→ Problema: Não diz por quê. Não diz o que há de errado. Não dá dados.

✅ CONTEXTO BOM:
"O serviço de pedidos atualmente usa MySQL 5.7 (EOL em outubro 2025)
em uma instância RDS db.r5.2xlarge ($1,400/mês). Nos últimos 6 meses:

  - Latência p99 de reads cresceu de 45ms para 280ms (6x)
  - Lock contention em writes > 100 TPS causa timeouts
  - Necessitamos de JSONB para o novo modelo de customização de pedidos
  - 3 incidentes P2 nos últimos 2 meses por query performance

A equipe (6 devs) tem experiência com PostgreSQL (3 projetos anteriores)
mas pouca experiência com DynamoDB. Budget de infraestrutura: $3,000/mês.
Timeline: migração precisa estar completa antes do EOL do MySQL 5.7."

→ Bom: Dados quantitativos, constraints claras, urgência explícita.
```

### Checklist de contexto

```
□ Descreve situação atual com dados?
□ Explica o problema ou necessidade?
□ Lista constraints (budget, timeline, skills)?
□ Quantifica o impacto do problema?
□ Explica o que acontece se NÃO decidir?
□ Menciona tentativas anteriores, se houver?
□ Cita ADRs relacionados ou precedentes?
```

---

## Como Estruturar Alternativas

### Framework para comparação justa

```
REGRAS PARA ALTERNATIVAS HONESTAS:

1. MÍNIMO 2 OPÇÕES (incluindo "do nothing" se aplicável)
   Decidir sem comparar não é decidir — é impor.

2. STEELMAN, NÃO STRAWMAN
   Apresente cada alternativa na sua melhor versão.
   Não monte alternativas fracas para a sua parecer melhor.
   ❌ Strawman: "Opção B é legada e ninguém usa"
   ✅ Steelman: "Opção B tem ecossistema maduro e custo menor"

3. CRITÉRIOS EXPLÍCITOS
   Defina os critérios ANTES de avaliar (evita viés de confirmação).
   Critérios derivados dos Decision Drivers.

4. TRADE-OFF MATRIX
   Mesmos critérios aplicados a todas as opções.

5. "DO NOTHING" É OPÇÃO VÁLIDA
   Às vezes a melhor decisão é não mudar.
   Custo de inação vs custo de mudança.
```

### Trade-off matrix template

```markdown
### Critérios de avaliação (derivados dos Decision Drivers)

| Critério | Peso | Opção A: PostgreSQL | Opção B: DynamoDB | Opção C: Aurora |
|----------|------|--------------------|--------------------|-----------------|
| Latência read p99 < 50ms | Alta | ✅ ~30ms | ✅ ~10ms | ✅ ~20ms |
| Suporte JSONB nativo | Alta | ✅ Nativo | ⚠️ Document model | ✅ Nativo (PG compat) |
| Custo ≤ $3,000/mês | Média | ✅ ~$1,800 | ✅ ~$2,200 (on-demand) | ⚠️ ~$2,800 |
| Experiência do time | Alta | ✅ 3 projetos | ❌ Nenhuma | ✅ PG-compatible |
| Write throughput > 500 TPS | Média | ⚠️ Requer tuning | ✅ Praticamente ilimitado | ✅ Otimizado |
| Migration complexity | Média | ⚠️ Schema migration | ❌ Rewrite parcial | ✅ Minimal (PG compat) |
| Operational overhead | Baixa | ⚠️ Managed (RDS) | ✅ Serverless | ✅ Managed |

**Resultado:** Opção A (PostgreSQL) atende melhor aos critérios de peso Alto,
especialmente experiência do time e suporte JSONB nativo. Opção C (Aurora)
é close second mas custo está no limite do budget.
```

### "Do Nothing" — Quando incluir

```
QUANDO "DO NOTHING" É ALTERNATIVA OBRIGATÓRIA:

✅ Quando o sistema atual funciona (mesmo que com problemas)
✅ Quando o custo de mudança é alto
✅ Quando o time tem outras prioridades urgentes
✅ Quando o problema pode se resolver sozinho (ex: tech debt que será reescrito)

COMO DOCUMENTAR:
  "Opção 0: Manter MySQL 5.7
   ✅ Zero custo de migração
   ✅ Zero risco de regression
   ❌ EOL em outubro 2025 (sem security patches)
   ❌ Latência continuará degradando
   ❌ Sem JSONB para novo modelo de customização"
```

---

## Como Escrever a Decisão

### A decisão em si é surpreendentemente simples

```
FORMATO:

"Iremos [verbo] [objeto] porque [razão vinculada a drivers]."

EXEMPLOS:

✅ "Iremos adotar PostgreSQL 16 no Amazon RDS como database 
    do order-service, porque atende aos requisitos de JSONB, 
    está dentro do budget, e o time já tem experiência."

✅ "Iremos usar CQRS com event sourcing para o domínio de 
    pagamentos, porque precisamos de audit trail completo 
    para compliance e a capacidade de reconstruir estado."

✅ "Iremos manter o monolito modular ao invés de migrar 
    para microsserviços, porque o time de 8 não tem a 
    capacidade operacional para gerenciar N deploys."

ANTI-PATTERNS:

❌ "Talvez devêssemos considerar PostgreSQL."
   → Ambíguo, não é uma decisão.

❌ "PostgreSQL foi escolhido."
   → Voz passiva, esconde quem decidiu e por quê.

❌ "Iremos adotar PostgreSQL."
   → Sem razão vinculada a drivers. Por quê?
```

### Decisão com scope explícito

```
DECISÃO COM SCOPE:

"Iremos adotar PostgreSQL 16 no Amazon RDS como database 
do order-service."

SCOPE EXPLÍCITO — o que esta decisão NÃO cobre:
• Não se aplica a outros serviços (cada time decide seu DB)
• Não define migration strategy (será Design Doc separado)
• Não define schema do novo modelo (será em PR específico)
• Não substitui DynamoDB onde já é usado (catalog-service)

→ Scope explícito evita interpretações expandidas da decisão.
```

---

## Como Documentar Consequências

### Framework: Positivas + Negativas + Neutras

```
REGRA DE OURO:
Se não tem consequências negativas, você está mentindo.
Toda decisão que vale um ADR tem trade-offs.

CONSEQUÊNCIAS POSITIVAS:
  Vinculadas diretamente aos Decision Drivers.
  "Resolve o problema X que motivou este ADR."

CONSEQUÊNCIAS NEGATIVAS:
  Trade-offs aceitos conscientemente.
  "Aceitamos X porque o benefício Y compensa."
  → Esta é a parte mais valiosa do ADR.
  → É o que permite reavaliar no futuro.

CONSEQUÊNCIAS NEUTRAS:
  Mudanças que não são boas nem ruins, mas que times
  precisam saber.
  "Precisaremos mudar o pipeline de CI para incluir step de migration."
```

### Exemplo completo de consequências

```markdown
### Positive Consequences

* Latência p99 esperada ~30ms (hoje: 280ms) — resolve o problema principal
* JSONB nativo permite o novo modelo de customização sem workarounds
* Time já tem experiência com PostgreSQL — ramp-up mínimo
* Custo estimado $1,800/mês — 40% abaixo do budget
* Community e tooling maduros (PostGIS, pg_stat, extensions)

### Negative Consequences

* **Migration downtime:** Estimamos 2-4h de downtime para migração
  (mitigável com blue/green, mas complexidade adicional)
* **Write throughput ceiling:** PostgreSQL single-primary tem limite
  prático de ~5,000 TPS. Se crescermos 10x, precisaremos sharding
  ou DynamoDB para write-heavy paths. (Risco aceito: crescimento
  atual é 20% YoY, temos ~3-4 anos antes de atingir o limite)
* **Operational burden:** RDS managed reduz, mas não elimina — 
  precisamos de monitoring, backup verification, upgrade cycles
* **Schema migrations:** Precisaremos de migration tooling (Flyway/Liquibase)
  e disciplina de zero-downtime migrations

### Neutral Consequences

* CI pipeline precisa de step de migration (Flyway)
* Runbooks de on-call precisam ser atualizados para PostgreSQL
* Dashboards de observabilidade precisam adaptar queries
```

---

## Técnicas de Argumentação

### Pyramid Principle (Barbara Minto)

```
PYRAMID PRINCIPLE APLICADO A ADRs:

  Comece pela conclusão (Decision), depois justifique.
  Leitor ocupado lê o topo e parou? Já entendeu a decisão.
  Leitor curioso? Vai mergulhar no contexto e alternativas.

                     ┌─────────────────┐
                     │  DECISÃO        │
                     │  (Conclusão)    │
                     └────────┬────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
        ┌─────▼─────┐  ┌─────▼─────┐  ┌──────▼────┐
        │ Razão 1   │  │ Razão 2   │  │ Razão 3   │
        │ (Driver)  │  │ (Driver)  │  │ (Driver)  │
        └─────┬─────┘  └─────┬─────┘  └──────┬────┘
              │              │               │
        ┌─────▼─────┐  ┌────▼──────┐  ┌─────▼─────┐
        │ Evidência │  │ Evidência │  │ Evidência │
        │ Dados     │  │ Benchmark │  │ PoC result│
        └───────────┘  └───────────┘  └───────────┘
```

### Quando a decisão é controversa

```
TÉCNICA: ACKNOWLEDGE → ADDRESS → ADVANCE

1. ACKNOWLEDGE
   "Entendemos que a opção B (DynamoDB) tem vantagens 
    significativas em write throughput e é totalmente serverless."

2. ADDRESS
   "Porém, no nosso contexto: a equipe não tem experiência 
    com DynamoDB (0 projetos), o modelo de dados é relacional 
    com JOINs complexos, e temos 3 meses para migrar."

3. ADVANCE
   "Portanto, propomos PostgreSQL agora, com reavaliação 
    em 12 meses se write throughput se tornar gargalo.
    ADR pode ser superseded se o contexto mudar."

RESULTADO:
  Quem defende DynamoDB se sente ouvido.
  Argumentos contra são endereçados com dados.
  Porta fica aberta para revisão futura.
```

### Dados > Opiniões

```
HIERARQUIA DE ARGUMENTOS (mais forte → mais fraco):

1. DADOS DE PRODUÇÃO
   "Em produção, p99 é 280ms e crescendo 15% MoM"
   → Incontestável

2. BENCHMARK / POC
   "Spike de 3 dias mostrou que PostgreSQL atingiu 30ms p99
    com nosso dataset de produção"
   → Forte evidência

3. CASE STUDY / EXPERIENCE
   "Time X usou PostgreSQL para caso similar e atingiu targets"
   → Relevante mas contextual

4. DOCUMENTAÇÃO / SPECS
   "Segundo a documentação, PostgreSQL suporta até 10K TPS"
   → Teórico, precisa validar no contexto

5. OPINIÃO DE EXPERT
   "Fulano (Staff at BigCorp) recomenda PostgreSQL"
   → Fraco — autoridade não é argumento

6. OPINIÃO PESSOAL
   "Eu acho que PostgreSQL é melhor"
   → Fraquíssimo — não deveria aparecer em ADR
```

---

## Review Process

### Quem deve revisar

```
REVIEWER MATRIX:

┌────────────────────┬───────────────────────────────────┐
│ Tipo de decisão    │ Reviewers recomendados             │
├────────────────────┼───────────────────────────────────┤
│ Intra-time         │ • Tech Lead do time               │
│ (afeta 1 serviço)  │ • Senior engineer do time         │
│                    │ • (Opcional) Principal/Architect   │
├────────────────────┼───────────────────────────────────┤
│ Cross-team         │ • Tech Leads dos times impactados │
│ (afeta N serviços) │ • Principal/Staff Engineer        │
│                    │ • Platform team (se infra)         │
├────────────────────┼───────────────────────────────────┤
│ Org-wide           │ • Principal/Chief Architect       │
│ (afeta a organização│• Engineering Managers impactados  │
│  inteira)          │ • VP Eng / CTO (para consciência) │
│                    │ • Security (se aplicável)          │
│                    │ • Compliance (se aplicável)        │
└────────────────────┴───────────────────────────────────┘
```

### O que o reviewer deve verificar

```
CHECKLIST DO REVIEWER:

CONTEXTO:
  □ O problema está claro e quantificado?
  □ Constraints estão explícitas (budget, time, skills)?
  □ Entendo por que esta decisão é necessária AGORA?

ALTERNATIVAS:
  □ Existem ≥ 2 alternativas reais (não strawmen)?
  □ "Do nothing" foi considerado?
  □ Alternativas rejeitadas receberam tratamento justo?
  □ Critérios de avaliação são razoáveis?

DECISÃO:
  □ A decisão é clara e afirmativa?
  □ Está vinculada aos decision drivers?
  □ O scope está explícito (o que NÃO é coberto)?

CONSEQUÊNCIAS:
  □ Existem consequências negativas honestas?
  □ Os trade-offs aceitos são razoáveis?
  □ Impactos em outros times estão documentados?

META:
  □ É escopo de ADR? (não deveria ser RFC/Design Doc?)
  □ É uma decisão ou múltiplas misturadas?
  □ Está conciso (1-2 páginas)?
```

### Review async vs sync

```
ASYNC (DEFAULT):
  PR aberto → reviewers analisam em 2-5 dias
  Comentários inline no Markdown
  Discussão threaded no PR

QUANDO FAZER SYNC:
  • Impasse em 2+ rounds de review async
  • Decisão cross-team com visões conflitantes
  • Timeline urgente (< 1 semana para decidir)

MEETING DE DECISÃO (se necessário):
  Duration: 30-60 min MAX
  Formato:
    5 min  — Autor apresenta ADR (leitura antes é dever de casa)
    15 min — Perguntas e objeções
    10 min — Discussão dos pontos controversos
    5 min  — Decisão: Accept / Iterate / Reject
  
  Regra: TODOS lêem o ADR ANTES da meeting.
  Se não leu → não pode reclamar na meeting.
```

---

## Modelos de Decisão

### Consent-based (recomendado)

```
CONSENT-BASED DECISION MAKING:

Pergunta: "Alguém tem objeção BLOQUEANTE?"
(Não "todos concordam?" — mas "alguém tem razão forte CONTRA?")

         ┌──────────────────────────┐
         │ Proposta apresentada     │
         └──────────┬───────────────┘
                    │
         ┌──────────▼───────────────┐
         │ "Alguém tem objeção      │
         │  bloqueante?"            │
         └──────────┬───────────────┘
                    │
          ┌─────────┴─────────┐
          │                   │
   ┌──────▼──────┐    ┌──────▼──────┐
   │  Ninguém    │    │  Objeção    │
   │  objeta     │    │  levantada  │
   └──────┬──────┘    └──────┬──────┘
          │                  │
   ┌──────▼──────┐    ┌─────▼───────┐
   │  ACCEPTED   │    │ Endereçar   │
   │             │    │ objeção     │
   └─────────────┘    │ → Iterar    │
                      │ → Voltar    │
                      └─────────────┘

VANTAGEM:
  Não espera unanimidade (impossível em decisões técnicas).
  Espera "safe enough to try, good enough for now."
  Principal Engineer é tie-breaker se impasse.
```

### RAPID model (para decisões complexas)

```
RAPID (Bain & Company):

  R — RECOMMEND: Quem propõe (autor do ADR)
  A — AGREE:     Quem tem poder de veto (compliance, security)
  P — PERFORM:   Quem implementa (engineering team)
  I — INPUT:     Quem consultar (stakeholders)
  D — DECIDE:    Quem toma a decisão final (1 pessoa)

EXEMPLO:
  ADR-0042: Migrar para PostgreSQL
  
  R (Recommend): Senior Dev do order-service team
  A (Agree):     Security team (compliance check), DBA
  P (Perform):   Order-service team
  I (Input):     Platform team, Payment team (dependency)
  D (Decide):    Principal Engineer / Tech Lead

QUANDO USAR:
  Decisões cross-org com muitos stakeholders.
  Quando não fica claro quem decide.
```

---

## Integração com Workflow de Engenharia

### ADR no fluxo de desenvolvimento

```
┌─────────────────────────────────────────────────────────────────┐
│        ADR INTEGRADO NO WORKFLOW                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. TICKET / EPIC                                                │
│     Jira/Linear Epic: "Migrate order-service to PostgreSQL"      │
│     ↳ First story: "Write ADR for database migration"            │
│                                                                  │
│  2. ADR PR                                                       │
│     Branch: adr/0042-postgresql-for-orders                       │
│     File: docs/adrs/0042-postgresql-for-orders.md                │
│     PR: Review by stakeholders                                   │
│     ↳ Status: Proposed → Accepted                                │
│                                                                  │
│  3. DESIGN DOC (se necessário)                                   │
│     Detalhamento da implementação                                │
│     Referencia: "Per ADR-0042"                                   │
│                                                                  │
│  4. IMPLEMENTATION PRs                                           │
│     Cada PR de implementação referencia o ADR:                   │
│     "Implements ADR-0042: PostgreSQL migration"                  │
│     Code comments onde relevante:                                │
│     // Per ADR-0042: using JSONB for order customization         │
│                                                                  │
│  5. DEPRECATED ADR                                               │
│     Quando código antigo é removido completamente:               │
│     ADR-0001 (MySQL) → Status: Superseded by ADR-0042           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Git conventions

```
COMMIT MESSAGES:
  docs(adr): add ADR-0042 PostgreSQL for order-service
  docs(adr): update ADR-0042 status to Accepted
  docs(adr): ADR-0001 superseded by ADR-0042

PR DESCRIPTION:
  ## Architecture Decision: ADR-0042
  
  Proposing PostgreSQL 16 as the database for order-service.
  See full ADR in docs/adrs/0042-postgresql-for-orders.md
  
  Reviewers: @tech-lead @principal-engineer @dba-team

BRANCH NAMING:
  adr/NNNN-slug-descritivo
  adr/0042-postgresql-for-orders
```

### ADR Index (README.md do diretório)

```markdown
# Architecture Decision Records

## Status Legend
- 🟢 Accepted — Decision in effect
- 🟡 Proposed — Under discussion
- 🔴 Deprecated — No longer relevant
- 🔵 Superseded — Replaced by newer ADR

## Index

| # | Title | Status | Date | Superseded by |
|---|-------|--------|------|---------------|
| 0001 | Use MySQL for order data | 🔵 Superseded | 2022-03-15 | ADR-0042 |
| 0002 | Adopt gRPC for internal comms | 🟢 Accepted | 2022-04-20 | |
| 0003 | Use event sourcing for payments | 🟢 Accepted | 2022-06-01 | |
| ... | | | | |
| 0041 | Adopt OpenTelemetry org-wide | 🟢 Accepted | 2024-01-10 | |
| 0042 | PostgreSQL for order-service | 🟡 Proposed | 2024-02-10 | |
```

---

## Ferramentas e Tooling

### Onde armazenar ADRs

| Opção | Prós | Contras | Quando usar |
|-------|------|---------|------------|
| **Monorepo: `docs/adrs/`** | Versionado com código, PR workflow | Difícil para cross-repo decisions | Default para single-repo projects |
| **Repo dedicado: `architecture-decisions`** | Centralizado, cross-team | Desconectado do código | Orgs com muitos repos |
| **Wiki (Confluence, Notion)** | Fácil de editar e buscar | Sem versioning, sem PR review | ❌ NÃO recomendado |
| **Repo + Wiki link** | Best of both worlds | Manter dois lugares em sync | Repo como source of truth, Wiki como discovery |

### Tooling de ADR

```
FERRAMENTAS:

1. adr-tools (Nygard) — CLI para gerenciar ADRs
   $ adr new "Use PostgreSQL for order-service"
   → Cria docs/adrs/0042-use-postgresql-for-order-service.md
   → Template pré-preenchido
   $ adr list
   $ adr link 42 "Supersedes" 1

2. Log4brains — ADR management + static site
   $ log4brains adr new
   → Gera site estático navegável dos ADRs
   → Preview no PR
   → Search integrado

3. MADR template — Markdown ADR template (GitHub)
   → Template estruturado com todas as seções
   → GitHub template repository

4. Custom: Copilot / AI
   → Use este documento como base de conhecimento
   → AI pode gerar draft de ADR a partir de discussão
   → AI pode review ADR contra checklist de qualidade
```

---

## Escrita Inclusiva e Async-First

### Princípios de escrita para times distribuídos

```
ADRs EM TIMES DISTRIBUÍDOS / ASYNC-FIRST:

1. SELF-CONTAINED
   Leitor não deveria precisar de meeting para entender.
   Se precisa de contexto verbal → ADR está incompleto.

2. LINGUAGEM CLARA
   • Evite jargão específico de um time
   • Defina acrônimos na primeira menção
   • Prefira exemplos concretos a abstrações

3. CONSIDERE FUSOS HORÁRIOS
   • Review period: mínimo 3 business days
   • Não declare "aceito" sem dar tempo para todos reviewers

4. CONSIDERE NÍVEIS DE EXPERIÊNCIA
   • Junior lendo ADR cross-team deve entender o contexto
   • Não assuma que leitor sabe sua stack

5. IDIOMA
   • Se o time é internacional: ADR em inglês
   • Se o time é local: ADR no idioma do time
   • Consistência > perfeição — escolha um e mantenha
```

---

## Anti-Patterns de Escrita

| Anti-pattern | Exemplo | Correção |
|-------------|---------|----------|
| **O ADR "obvious"** | Contexto: "Precisamos de um banco". Decisão: "PostgreSQL" | Explique POR QUE PostgreSQL e não DynamoDB/Aurora/etc |
| **O ADR "novel"** | 10 páginas com diagramas de sequência | ADR é 1-2 pg. Detalhes vão em Design Doc |
| **O ADR "político"** | Alternativas strawman para justificar decisão pré-tomada | Steelman: apresente cada opção na melhor versão |
| **O ADR "otimista"** | Apenas consequências positivas | Liste trade-offs honestamente |
| **O ADR "passivo"** | "Foi decidido que..." / "Pode-se considerar..." | Voz ativa: "Iremos adotar X porque Y" |
| **O ADR "tardio"** | Escrito 6 meses depois, memória vaga | Escreva durante ou imediatamente após a decisão |
| **O ADR "solo"** | Autor commita sem PR review | Sempre via PR com ≥ 2 reviewers relevantes |
| **O ADR "acumulador"** | 3 decisões em 1 ADR | Uma decisão por ADR |
| **O ADR "eterno Proposed"** | Proposed há 4 meses, ninguém decidiu | SLA: decisão em ≤ 2 semanas ou escalar |
| **O ADR "jargonado"** | Linguagem que só 2 pessoas entendem | Escreva para um engenheiro que entrou ontem |

---

## Diretrizes para Code Review assistido por AI

Ao revisar ADRs e processos de decisão, verifique:

1. **ADR sem processo de review** — Toda ADR deve passar por PR com reviewers explícitos
2. **Review sem checklist** — Reviewers devem validar: contexto, alternativas, trade-offs, scope
3. **Decisão Tipo 1 sem dados** — Decisões irreversíveis precisam de spike/PoC/benchmark antes do ADR
4. **Alternativas strawman** — Cada opção deve ser apresentada na melhor versão (steelman)
5. **Argumentação por opinião** — Priorizar: dados produção > benchmark > case study > documentação > opinião
6. **ADR sem socialização** — Após Accepted, comunicar ativamente (Slack, all-hands, wiki)
7. **PR sem referência a ADR** — Implementação de decisão arquitetural deve linkar ADR no PR
8. **Code comment sem contexto** — Se design choice precisa de explicação, referir ADR: `// Per ADR-NNNN`
9. **Consent vs consenso** — Não esperar unanimidade; "objeção bloqueante?" é suficiente
10. **ADR Proposed por > 2 semanas** — Escalar ou decidir; ADR em limbo perde relevância
11. **ADR sem scope explícito** — Definir o que a decisão NÃO cobre para evitar interpretação expandida
12. **ADR em wiki em vez de repo** — Source of truth deve ser versionado no Git com PR review

---

## Referências

- **Michael Nygard** — "Documenting Architecture Decisions" (2011)
- **MADR** — Markdown Any Decision Records — https://adr.github.io/madr/
- **adr-tools** — https://github.com/npryce/adr-tools
- **Log4brains** — https://github.com/thomvaill/log4brains
- **ThoughtWorks Tech Radar** — Lightweight ADRs (Adopt)
- **Pyramid Principle** — Barbara Minto (McKinsey)
- **RAPID Decision Making** — Bain & Company
- **The Staff Engineer's Path** — Tanya Reilly — Chapter on Communication
- **Staff Engineer** — Will Larson — Decision documentation
- **Design It!** — Michael Keeling (Pragmatic Bookshelf)
- **Sociocracy 3.0** — Consent-based decision making — https://sociocracy30.org/
