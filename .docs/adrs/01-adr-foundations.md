# Architecture Decision Records — Fundamentos

> **Objetivo deste documento:** Servir como referência completa sobre **Architecture Decision Records (ADRs)** — o que são, por que existem e quando usar, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: conceitos fundamentais, anatomia de um ADR, tipos de decisões, ciclo de vida, e o papel do ADR na carreira de Principal Engineer.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **ADR** | Toda decisão arquitetural significativa é documentada | "A gente decidiu em uma call" — sem registro | Escrever ADR antes ou imediatamente após a decisão |
| **Imutabilidade** | ADRs são imutáveis — superseda, não edita | Editar ADR antigo para "corrigir" a decisão | Criar novo ADR que supersede o anterior |
| **Contexto** | Contexto é mais importante que a decisão | ADR com título e decisão sem explicar o porquê | Documentar constraints, trade-offs, alternativas |
| **Status** | Todo ADR tem status explícito | ADR sem status — ninguém sabe se está valendo | Proposed → Accepted → Superseded/Deprecated |
| **Escopo** | Uma decisão por ADR | ADR com 5 decisões misturadas | Separar em ADRs individuais |
| **Consequências** | Listar trade-offs explicitamente | Apenas consequências positivas | Documentar positivas E negativas |

---

## Sumário

- [Architecture Decision Records — Fundamentos](#architecture-decision-records--fundamentos)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que são Architecture Decision Records](#o-que-são-architecture-decision-records)
  - [Por que documentar decisões](#por-que-documentar-decisões)
  - [Anatomia de um ADR](#anatomia-de-um-adr)
  - [Tipos de Decisões Arquiteturais](#tipos-de-decisões-arquiteturais)
  - [Quando escrever um ADR](#quando-escrever-um-adr)
  - [Quando NÃO escrever um ADR](#quando-não-escrever-um-adr)
  - [Ciclo de Vida de um ADR](#ciclo-de-vida-de-um-adr)
  - [ADR e o Papel de Principal Engineer](#adr-e-o-papel-de-principal-engineer)
  - [ADR vs Outros Documentos](#adr-vs-outros-documentos)
  - [Qualidade de um ADR — Rubric](#qualidade-de-um-adr--rubric)
  - [Anti-Patterns de ADRs](#anti-patterns-de-adrs)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que são Architecture Decision Records

```
┌─────────────────────────────────────────────────────────────────┐
│            ARCHITECTURE DECISION RECORDS (ADRs)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DEFINIÇÃO:                                                      │
│  Um ADR é um documento curto que captura uma decisão             │
│  arquitetural significativa, junto com seu contexto,             │
│  alternativas consideradas e consequências.                      │
│                                                                  │
│  ORIGEM:                                                         │
│  Michael Nygard (2011) — "Documenting Architecture Decisions"    │
│  ThoughtWorks Technology Radar — "Adopt" desde 2016              │
│                                                                  │
│  FORMATO FUNDAMENTAL:                                            │
│  ┌──────────────────────────────────────┐                       │
│  │  # ADR-NNNN: Título da Decisão       │                       │
│  │                                       │                       │
│  │  Status: [Proposed|Accepted|...]      │                       │
│  │  Date: YYYY-MM-DD                     │                       │
│  │                                       │                       │
│  │  ## Context                           │                       │
│  │  Por que essa decisão é necessária?   │                       │
│  │                                       │                       │
│  │  ## Decision                          │                       │
│  │  O que foi decidido?                  │                       │
│  │                                       │                       │
│  │  ## Consequences                      │                       │
│  │  O que muda por causa dessa decisão?  │                       │
│  └──────────────────────────────────────┘                       │
│                                                                  │
│  PRINCÍPIOS:                                                     │
│  • Leve (1-2 páginas, não 50)                                    │
│  • Imutável (nunca editar — supersede com novo ADR)             │
│  • Numerado (sequencial: ADR-0001, ADR-0002, ...)               │
│  • Versionado (no Git, junto com o código)                       │
│  • Uma decisão por ADR                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### A metáfora do "Diário de bordo"

```
ADRs são como o diário de bordo de um navio:

  "Em 2023-06-15, decidimos migrar de monolito para microsserviços
   porque o time de 40 developers não conseguia deployar independentemente.
   Consideramos modular monolith mas rejeitamos porque precisávamos
   de deploys independentes com ciclos diferentes.
   Aceitamos o trade-off de complexidade operacional em troca de
   autonomia de times."

Sem o diário:
  2 anos depois: "Por que temos microsserviços? Ninguém lembra."
  Novo membro: "Não seria melhor um monolito? Por que não fizeram?"
  Resposta: *silêncio* — a decisão foi tomada mas o raciocínio se perdeu.

COM o diário (ADR):
  2 anos depois: Leia ADR-0042. Contexto, alternativas, trade-offs.
  Novo membro: "Entendi. As constraints mudaram? Se sim, podemos rever."
  Resposta: Decisão informada com contexto completo.
```

---

## Por que documentar decisões

### O problema real

```
┌─────────────────────────────────────────────────────────────────┐
│              O CUSTO DE NÃO DOCUMENTAR DECISÕES                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SEM ADRs:                                                       │
│                                                                  │
│  ┌─────────────────┐                                            │
│  │ Decisão tomada  │  → Contexto existe nas cabeças das pessoas │
│  │ em 2023-06      │  → Ninguém escreveu os trade-offs          │
│  └────────┬────────┘                                            │
│           │ 6 meses depois                                       │
│  ┌────────▼────────┐                                            │
│  │ Pessoas mudam   │  → 2 dos 4 decision makers saíram          │
│  │ de time/empresa │  → Contexto parcial, viés de memória       │
│  └────────┬────────┘                                            │
│           │ 12 meses depois                                      │
│  ┌────────▼────────┐                                            │
│  │ Nova pessoa      │  → "Por que foi feito assim?"             │
│  │ questiona        │  → Ninguém lembra. Tribal knowledge.      │
│  └────────┬────────┘                                            │
│           │ 18 meses depois                                      │
│  ┌────────▼────────┐                                            │
│  │ Decisão refeita │  → Mesma discussão de 18 meses atrás       │
│  │ sem contexto    │  → Mesmos erros, sem aprender com passado  │
│  └─────────────────┘                                            │
│                                                                  │
│  COM ADRs:                                                       │
│                                                                  │
│  ┌─────────────────┐                                            │
│  │ Decisão tomada  │  → ADR-0042 escrito com contexto completo  │
│  │ ADR-0042        │  → Git: quem escreveu, quem aprovou        │
│  └────────┬────────┘                                            │
│           │ 18 meses depois                                      │
│  ┌────────▼────────┐                                            │
│  │ Nova pessoa lê  │  → "Entendo o porquê. As constraints       │
│  │ ADR-0042        │     mudaram? Se sim, escrevo ADR-0089      │
│  │                 │     que supersede ADR-0042."                │
│  └─────────────────┘                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Os 7 motivos para documentar decisões

| # | Motivo | Impacto |
|---|--------|---------|
| 1 | **Memória organizacional** | Decisões sobrevivem à rotatividade de pessoas |
| 2 | **Onboarding acelerado** | Novo membro entende o "porquê" em horas, não meses |
| 3 | **Evitar re-discussão** | Decisão tomada + contexto = não repetir debate |
| 4 | **Accountability** | Quem decidiu, quando, com que informação |
| 5 | **Evolução informada** | Para mudar, primeiro entenda por que foi feito assim |
| 6 | **Comunicação assíncrona** | Times distribuídos tomam decisões com contexto escrito |
| 7 | **Rastreabilidade** | Auditoria: por que o sistema tem esta forma? ADR trail |

---

## Anatomia de um ADR

### Estrutura mínima (Nygard original)

```markdown
# ADR-NNNN: [Título curto e descritivo]

**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-XXXX
**Date:** YYYY-MM-DD
**Deciders:** [Quem participou da decisão]

## Context
[Qual é o problema ou necessidade que motivou esta decisão?
 Quais são as forças em jogo — técnicas, organizacionais, regulatórias?
 O que acontece se não decidirmos?]

## Decision
[O que decidimos fazer?
 Afirmativo: "Nós iremos..." / "Usaremos..." / "Adotaremos..."
 Direto e sem ambiguidade.]

## Consequences
[O que muda por causa dessa decisão?
 Positivas: ganhos, simplificações
 Negativas: trade-offs, custos, riscos aceitos
 Neutras: mudanças que não são boas nem ruins]
```

### Estrutura expandida (recomendada para decisões significativas)

```markdown
# ADR-NNNN: [Título curto e descritivo]

**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-XXXX
**Date:** YYYY-MM-DD
**Deciders:** [Nomes ou papéis]
**Technical Story:** [Link para ticket/RFC/issue que motivou]

## Context and Problem Statement

[Descreva o contexto e o problema. Por que esta decisão é necessária agora?
 Quais constraints existem (budget, time, team skills, compliance)?
 Inclua dados quantitativos quando possível.]

## Decision Drivers

* [driver 1: e.g., time-to-market de 3 meses]
* [driver 2: e.g., equipe de 5 sem experiência em Kafka]
* [driver 3: e.g., requisito LGPD de data residency]
* [driver 4: e.g., budget de $X/mês para infraestrutura]

## Considered Options

1. [Opção 1 — nome descritivo]
2. [Opção 2 — nome descritivo]
3. [Opção 3 — nome descritivo]

## Decision Outcome

**Chosen option:** "[Opção X]", because [justificativa resumida em 1-2 frases
vinculada aos decision drivers].

### Positive Consequences

* [Consequência positiva 1]
* [Consequência positiva 2]

### Negative Consequences

* [Trade-off aceito 1]
* [Trade-off aceito 2]

## Pros and Cons of the Options

### [Opção 1]

[Descrição breve da opção]

* ✅ [Pro 1]
* ✅ [Pro 2]
* ❌ [Con 1]
* ❌ [Con 2]

### [Opção 2]

[Descrição breve da opção]

* ✅ [Pro 1]
* ❌ [Con 1]
* ❌ [Con 2]

### [Opção 3]

[Descrição breve da opção]

* ✅ [Pro 1]
* ❌ [Con 1]

## Links

* [RFC/Issue que originou esta discussão]
* [ADR-XXXX que esta decisão supersede, se aplicável]
* [Spike/PoC realizado para validar]
* [Documentação externa relevante]
```

### Cada seção em detalhe

```
┌─────────────────────────────────────────────────────────────────┐
│                ANATOMIA DETALHADA DE UM ADR                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TITLE (Título)                                                  │
│  ─────────────                                                   │
│  Formato: "ADR-NNNN: [Verbo] + [Objeto] + [Contexto]"          │
│  ✅ "ADR-0042: Adotar PostgreSQL como database principal"        │
│  ✅ "ADR-0043: Usar event sourcing para domínio de pagamentos"  │
│  ❌ "ADR-0042: Database" (vago)                                 │
│  ❌ "ADR-0043: Mudanças no sistema" (genérico)                  │
│                                                                  │
│  STATUS                                                          │
│  ──────                                                          │
│  Proposed  → Decisão em discussão, aguardando review             │
│  Accepted  → Decisão aprovada, em vigor                          │
│  Deprecated → Decisão não mais relevante (contexto mudou)        │
│  Superseded → Substituída por ADR mais recente                   │
│  Rejected  → Proposta rejeitada (mantém para histórico)          │
│                                                                  │
│  CONTEXT                                                         │
│  ───────                                                         │
│  A SEÇÃO MAIS IMPORTANTE do ADR                                  │
│  • O que está acontecendo que exige uma decisão?                 │
│  • Quais são as forças/constraints em jogo?                      │
│  • Dados quantitativos (latência atual, custo, throughput)       │
│  • O que acontece se NÃO decidirmos?                             │
│                                                                  │
│  DECISION                                                        │
│  ────────                                                        │
│  • Afirmativo: "Nós iremos..."                                   │
│  • Uma decisão clara, sem ambiguidade                            │
│  • Não é uma discussão — é o resultado da discussão              │
│                                                                  │
│  CONSEQUENCES                                                    │
│  ────────────                                                    │
│  • POSITIVAS: ganhos diretos da decisão                          │
│  • NEGATIVAS: trade-offs aceitos conscientemente                 │
│  • O que precisamos fazer/mudar por causa dessa decisão          │
│  ⚠️ Se não tem consequências negativas, você está mentindo       │
│     — toda decisão tem trade-offs                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tipos de Decisões Arquiteturais

### Taxonomia de decisões

| Categoria | Exemplos | Impacto | Reversibilidade |
|-----------|---------|---------|-----------------|
| **Tecnologia** | Linguagem, framework, database, cloud provider | Alto | Baixa (anos para reverter) |
| **Estrutura** | Monolito vs microsserviços, modular monolith | Muito alto | Muito baixa |
| **Integração** | REST vs gRPC vs eventos, sync vs async | Alto | Média |
| **Dados** | Event sourcing vs CRUD, SQL vs NoSQL, partitioning | Muito alto | Muito baixa |
| **Infraestrutura** | Kubernetes vs ECS, serverless vs containers | Alto | Média |
| **Segurança** | Authn/authz strategy, encryption, zero-trust | Alto | Baixa |
| **Processo** | Trunk-based vs GitFlow, monorepo vs polyrepo | Médio | Média |
| **Organização** | Team boundaries, ownership model, platform team | Alto | Baixa |

### Decisões Tipo 1 vs Tipo 2 (Jeff Bezos)

```
┌─────────────────────────────────────────────────────────────────┐
│          DECISÕES TIPO 1 vs TIPO 2 (BEZOS)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TIPO 1 — PORTAS DE MÃO ÚNICA (One-way doors)                   │
│  ════════════════════════════════════════════                     │
│  • Irreversíveis ou extremamente custosas de reverter           │
│  • Requerem deliberação cuidadosa                               │
│  • ADR OBRIGATÓRIO                                               │
│  • Review por múltiplos stakeholders                             │
│                                                                  │
│  Exemplos:                                                       │
│  - Escolha de cloud provider                                     │
│  - Linguagem principal de backend                                │
│  - Monolito → microsserviços                                     │
│  - Event sourcing como source of truth                           │
│  - Modelo de multi-tenancy (shared vs isolated)                  │
│                                                                  │
│  TIPO 2 — PORTAS DE MÃO DUPLA (Two-way doors)                   │
│  ════════════════════════════════════════════                     │
│  • Reversíveis com custo razoável                               │
│  • Podem ser decididas rapidamente                               │
│  • ADR OPCIONAL (mas útil para comunicação)                      │
│  • Decisor pode ser time/squad                                   │
│                                                                  │
│  Exemplos:                                                       │
│  - Biblioteca de HTTP client                                     │
│  - Feature flag framework                                        │
│  - Formato de log (pode mudar com processor)                    │
│  - CI/CD pipeline tool                                           │
│  - Testing framework                                             │
│                                                                  │
│  ⚠️ ERRO COMUM:                                                 │
│  Tratar decisões Tipo 2 como Tipo 1 → análise paralysis         │
│  Tratar decisões Tipo 1 como Tipo 2 → arrependimento caro      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quando escrever um ADR

### Decision matrix

```
A decisão se encaixa em pelo menos um critério?

┌─────────────────────────────────────────────────┐
│  □ Afeta múltiplos times ou serviços            │
│  □ É difícil ou custosa de reverter             │
│  □ Tem trade-offs significativos                │
│  □ Alguém vai perguntar "por que?" daqui 1 ano │
│  □ Requer mudança de padrão/convenção existente │
│  □ Tem implicações de custo > $1000/mês         │
│  □ Afeta SLOs ou disponibilidade                │
│  □ Envolve escolha entre alternativas viáveis   │
│  □ Vai mudar como os times trabalham            │
│  □ Tem requisitos de compliance/segurança       │
└─────────────────────────────────────────────────┘

Se ≥ 1 checkbox = ✅  → Escreva um ADR
Se 0 checkboxes  = ❌ → Provavelmente não precisa de ADR
```

### Exemplos de decisões que MERECEM ADR

| Decisão | Por que merece ADR |
|---------|-------------------|
| Migrar de MySQL para PostgreSQL | Tipo 1, afeta todos os times, custosa de reverter |
| Adotar gRPC para comunicação interna | Afeta múltiplos times, muda padrão existente |
| Usar CQRS para domínio de orders | Trade-offs significativos, alguém vai perguntar |
| Kubernetes em vez de ECS | Tipo 1, custo, skills necessárias |
| Monorepo para frontend | Afeta como times trabalham, muda workflow |
| Adotar OpenTelemetry como padrão | Cross-team, muda stack de observabilidade |
| Multi-tenancy com schema isolation | Tipo 1, afeta performance e custo |

---

## Quando NÃO escrever um ADR

```
NÃO ESCREVA ADR PARA:

❌ Decisões triviais sem trade-offs significativos
   "Usar Prettier para formatação" → README do projeto

❌ Decisões temporárias / experimentos
   "Testar library X por 2 semanas" → Spike ticket

❌ Detalhes de implementação
   "Usar HashMap em vez de TreeMap" → Code review

❌ Bug fixes
   "Corrigir null pointer no handler" → Pull request

❌ Convenções de código
   "Tabs vs spaces" → .editorconfig / linter

❌ Decisões já universalmente aceitas na org
   "Usar Git" → Desnecessário

REGRA: Se a decisão não tem trade-offs significativos que 
       alguém possa questionar no futuro → não é um ADR.
```

---

## Ciclo de Vida de um ADR

```
┌─────────────────────────────────────────────────────────────────┐
│               CICLO DE VIDA DE UM ADR                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐                                                    │
│  │ PROPOSED │  Autor escreve ADR e abre PR/review                │
│  │          │  → Título, Context, Options claro                  │
│  │          │  → Decision pode ser tentativa                     │
│  │          │  → Pede review dos stakeholders                    │
│  └────┬─────┘                                                    │
│       │ Review & feedback (async ou sync meeting)                │
│       │                                                          │
│  ┌────▼─────┐    ┌───────────┐                                  │
│  │ ACCEPTED │    │ REJECTED  │                                   │
│  │          │    │           │                                    │
│  │ Decision │    │ Decision  │                                    │
│  │ aprovada │    │ negada    │                                    │
│  │ PR merged│    │ PR merged │  (mantém no repo para histórico)  │
│  └────┬─────┘    └───────────┘                                   │
│       │                                                          │
│       │  Tempo passa... contexto muda                            │
│       │                                                          │
│  ┌────▼──────────┐    ┌────────────────┐                        │
│  │ SUPERSEDED    │    │ DEPRECATED     │                         │
│  │               │    │                │                         │
│  │ Nova decisão  │    │ Contexto não   │                         │
│  │ substitui     │    │ mais relevante │                         │
│  │ (ADR-0089     │    │ (serviço       │                         │
│  │  supersedes   │    │  descontinuado)│                         │
│  │  ADR-0042)    │    │                │                         │
│  └───────────────┘    └────────────────┘                        │
│                                                                  │
│  REGRAS:                                                         │
│  • ADR NUNCA é editado após Accepted                             │
│  • Para mudar decisão → novo ADR que "supersedes ADR-NNNN"      │
│  • ADR Rejected mantém no repo (histórico: "já tentamos isso") │
│  • Status é atualizado apenas no campo Status (não no body)     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Numeração e organização

```
docs/adrs/                          # ou adr/ 
├── 0001-use-postgresql-as-primary-database.md
├── 0002-adopt-grpc-for-internal-communication.md
├── 0003-use-event-sourcing-for-payments.md
├── 0004-migrate-to-kubernetes.md
├── 0005-adopt-opentelemetry.md
├── 0006-use-trunk-based-development.md
├── ...
├── 0042-replace-postgresql-with-aurora.md   ← supersedes 0001
└── README.md                                ← índice com status

CONVENÇÕES DE NAMING:
  [NNNN]-[slug-descritivo].md
  
  Número: sequencial, 4 dígitos, zero-padded
  Slug: lowercase, hyphens, verbo + objeto
  Extensão: .md (Markdown)
  
NÃO RENUMERE. Gaps na sequência são OK.
Se ADR-0007 foi rejected, ADR-0008 segue normal.
```

---

## ADR e o Papel de Principal Engineer

### Por que ADRs são ferramenta fundamental de Principal

```
┌─────────────────────────────────────────────────────────────────┐
│       ADRs COMO FERRAMENTA DE PRINCIPAL ENGINEER                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PRINCIPAL ENGINEER opera em 3 dimensões:                        │
│                                                                  │
│  1. INFLUÊNCIA SEM AUTORIDADE                                    │
│     ─────────────────────────                                    │
│     Principal não é chefe. Influencia pela qualidade das         │
│     ideias e pela clareza da comunicação.                        │
│     ADRs são o veículo dessa influência:                         │
│     • Argumentos escritos > argumentos verbais                   │
│     • Alternativas explícitas mostram profundidade               │
│     • Trade-offs honestos criam confiança                        │
│                                                                  │
│  2. ESCALA ATRAVÉS DE DOCUMENTAÇÃO                               │
│     ─────────────────────────────                                │
│     Principal não pode estar em toda reunião.                    │
│     ADRs escalam o raciocínio do Principal:                      │
│     • Decisão documentada = decisão que não precisa             │
│       ser explicada pessoalmente 50 vezes                        │
│     • Outros times podem consumir e aplicar                      │
│     • Novos membros absorvem o "why" sem meeting                │
│                                                                  │
│  3. LEGADO E CONTINUIDADE                                        │
│     ────────────────────                                         │
│     Se o Principal sair, o raciocínio fica.                      │
│     ADRs são o legado intelectual:                               │
│     • Decisões sobrevivem a reorganizações                       │
│     • História arquitetural preservada                           │
│     • Evolução intencional ao invés de acidental                 │
│                                                                  │
│  RESPONSABILIDADES DO PRINCIPAL:                                 │
│  ✅ Escrever ADRs para decisões cross-team                      │
│  ✅ Revisar ADRs de outros times (quality gate)                  │
│  ✅ Ensinar e mentorear a escrita de ADRs                        │
│  ✅ Manter a cultura de documentar decisões                      │
│  ✅ Garantir que ADRs são referenciados em PRs                   │
│  ✅ Conduzir revisão periódica de ADRs ativos                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### ADR como ferramenta de comunicação

| Audiência | O que o ADR comunica | Formato preferido |
|-----------|---------------------|-------------------|
| **Engineers no time** | Decisão técnica detalhada com trade-offs | Full ADR (todas as seções) |
| **Engineers de outros times** | Decisão cross-team que impacta sua API/contrato | ADR + seção "Impact on teams" |
| **Engineering Managers** | Riscos, custos, timeline implications | Executive summary no topo |
| **Product Managers** | Feature implications, tempo necessário | "Consequences" em linguagem de negócio |
| **Future selves** | Por que o sistema tem esta forma | Context detalhado com dados |

---

## ADR vs Outros Documentos

```
┌─────────────────────────────────────────────────────────────────┐
│            ADR vs OUTROS DOCUMENTOS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ADR vs RFC (Request for Comments)                               │
│  ────────────────────────────────                                │
│  RFC: Proposta detalhada ANTES da decisão (5-20 páginas)         │
│  ADR: Registro da decisão DEPOIS (1-2 páginas)                   │
│  Relação: RFC → discussão → ADR captura o resultado              │
│                                                                  │
│  ADR vs Design Doc                                               │
│  ─────────────────                                               │
│  Design Doc: Como implementar (detalhes, diagramas, APIs)       │
│  ADR: POR QUE este approach (trade-offs, alternativas)           │
│  Relação: ADR decide "what/why", Design Doc detalha "how"       │
│                                                                  │
│  ADR vs README                                                   │
│  ────────────                                                    │
│  README: Estado atual (como rodar, como contribuir)              │
│  ADR: Decisão pontual (snapshot no tempo)                        │
│  Relação: README referencia ADRs, ADRs não substituem README    │
│                                                                  │
│  ADR vs Meeting Notes                                            │
│  ───────────────────                                             │
│  Meeting Notes: Discussão crua, não estruturada                  │
│  ADR: Resultado destilado, com alternativas e trade-offs         │
│  Relação: Meeting pode gerar ADR, ADR nunca é meeting notes     │
│                                                                  │
│  ADR vs Tech Radar                                               │
│  ────────────────                                                │
│  Tech Radar: Avaliação de tecnologias (Adopt/Trial/Hold)        │
│  ADR: Decisão específica de uso para um contexto                │
│  Relação: Tech Radar informa ADR (usamos X porque está Adopt)  │
│                                                                  │
│  COMPLEMENTARIDADE:                                              │
│  RFC → ADR → Design Doc → PR → Code                             │
│  (proposta → decisão → detalhamento → implementação → código)   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| Documento | Foco | Tamanho | Mutabilidade | Audiência |
|-----------|------|---------|-------------|-----------|
| **ADR** | Decisão + trade-offs | 1-2 páginas | Imutável | Engineering |
| **RFC** | Proposta detalhada | 5-20 páginas | Mutável (draft) | Engineering + PMs |
| **Design Doc** | Como implementar | 5-15 páginas | Mutável | Engineering |
| **README** | Estado atual do projeto | 1-5 páginas | Mutável | Everyone |
| **Tech Radar** | Avaliação de tecnologias | 1 página/item | Atualizado periódico | Engineering Org |
| **Runbook** | Como operar/resolver | 1-3 páginas | Mutável | On-call |

---

## Qualidade de um ADR — Rubric

### Checklist de qualidade

| Critério | ✅ Bom | ❌ Ruim |
|----------|--------|--------|
| **Título** | Verbo + objeto: "Adotar PostgreSQL como primary DB" | "Database" / "Mudanças" |
| **Status** | Explícito: Proposed / Accepted | Sem status ou ambíguo |
| **Contexto** | Dados quantitativos, constraints explícitas | "Precisamos de um banco de dados" |
| **Decision drivers** | Listados: time, budget, skills, compliance | Não mencionados |
| **Alternativas** | ≥ 2 opções com pros/cons | Apenas a opção escolhida |
| **Decisão** | Afirmativa: "Iremos usar X porque Y" | "Talvez usar X" / passiva |
| **Consequências +** | Benefícios concretos vinculados a drivers | Genérico: "vai ser melhor" |
| **Consequências −** | Trade-offs honestos: "Aceitamos X porque Y" | Nenhuma (red flag 🚩) |
| **Escopo** | Uma decisão por ADR | Múltiplas decisões misturadas |
| **Tamanho** | 1-2 páginas (focado) | 10+ páginas (deveria ser RFC/Design Doc) |
| **Linguagem** | Clara, direta, sem jargão desnecessário | Vago, passivo, corporativo |

### Scoring

```
SCORE                AÇÃO
─────────────────────────────────
9-11 critérios ✅    Excelente — merge
7-8 critérios ✅     Bom — merge com minor fixes
5-6 critérios ✅     Refinar — reescrever seções fracas
< 5 critérios ✅     Reescrever — não atingiu o propósito
```

---

## Anti-Patterns de ADRs

| Anti-pattern | Problema | Solução |
|-------------|---------|---------|
| **ADR retro-ativo fake** | Escrever ADR meses depois para "cobrir" decisão já implementada, sem contexto real | Escrever ADR no momento da decisão ou logo após |
| **ADR sem alternativas** | Apenas a opção escolhida — sem prova de que houve deliberação | Pelo menos 2 alternativas com pros/cons |
| **ADR sem consequências negativas** | Toda decisão tem trade-offs — se não listou, está omitindo | Listar explicitamente o que é sacrificado |
| **ADR editado** | Mudar decisão passada distorce o histórico | Nunca editar body — criar novo ADR que supersede |
| **ADR como RFC** | ADR de 15 páginas com diagramas detalhados | ADR é curto (1-2 pg). Detalhes vão em RFC/Design Doc |
| **ADR como meeting notes** | Transcrição de discussão sem estrutura | ADR é o resultado destilado, não o processo |
| **ADR sem review** | Uma pessoa decide e commita sozinha | PR review com stakeholders impactados |
| **ADR orphan** | ADR não referenciado em PRs/code | Linkar ADR no PR que implementa |
| **ADR graveyard** | 200 ADRs, ninguém sabe quais valem | README com índice + status. Revisão periódica |
| **ADR premature** | ADR antes de ter informação suficiente | Spike/PoC primeiro, depois ADR com dados |

---

## Diretrizes para Code Review assistido por AI

Ao revisar ADRs ou código relacionado a decisões arquiteturais, verifique:

1. **Decisão sem ADR** — Mudança arquitetural significativa (nova tech, novo pattern) sem ADR? Solicitar
2. **ADR sem contexto** — Context vago ou ausente → peça dados quantitativos e constraints explícitas
3. **ADR sem alternativas** — Apenas uma opção listada → peça pelo menos 2 alternativas com pros/cons
4. **ADR sem consequências negativas** — Todo trade-off tem custo → solicite honestidade sobre o que se perde
5. **ADR muito longo** — > 3 páginas → sugerir separar em ADR (decisão) + Design Doc (implementação)
6. **ADR com múltiplas decisões** — Separar em ADRs individuais para rastreabilidade
7. **PR sem referência a ADR** — Se PR implementa decisão arquitetural, deve linkar o ADR
8. **ADR com linguagem ambígua** — "Podemos considerar..." → deveria ser "Iremos adotar..."
9. **ADR sem status** — Todo ADR precisa de status explícito (Proposed/Accepted/etc)
10. **ADR editando body de ADR existente** — Nunca editar → criar novo que supersede
11. **Decisão Tipo 1 sem deliberação** — Decisão irreversível decidida sem revisão ampla
12. **Título vago** — "ADR: Database" → precisa ser "ADR-NNNN: Adotar PostgreSQL como primary database"

---

## Referências

- **Michael Nygard** — "Documenting Architecture Decisions" (2011) — https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
- **ADR GitHub Organization** — https://adr.github.io/
- **MADR (Markdown ADR)** — https://adr.github.io/madr/
- **Joel Parker Henderson — ADR collection** — https://github.com/joelparkerhenderson/architecture-decision-record
- **ThoughtWorks Technology Radar** — Lightweight Architecture Decision Records (Adopt)
- **Documenting Software Architectures** — Paul Clements et al. (SEI/CMU)
- **Design It!** — Michael Keeling (Pragmatic Bookshelf)
- **Fundamentals of Software Architecture** — Mark Richards, Neal Ford (O'Reilly)
- **Staff Engineer** — Will Larson — Decision-making visibility
- **The Staff Engineer's Path** — Tanya Reilly — Communication and documentation
