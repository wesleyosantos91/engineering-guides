# Architecture Decision Records — Governança em Escala

> **Objetivo deste documento:** Servir como referência prática sobre **como escalar ADRs em organizações** — de um time para centenas, tratando governança, discovery, métricas, evolução e cultura de documentação, otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: governança organizacional, cross-team ADRs, tooling, métricas, e o papel do Principal Engineer na cultura de decisões.

---

## Quick Reference — Cheat Sheet

| Desafio | Solução | Papel do Principal |
|---------|---------|-------------------|
| **Ninguém sabe que ADR existe** | ADR Index + search (Log4brains/site) | Manter index atualizado, socializar |
| **Inconsistência entre times** | Template padrão + linting CI | Definir org template, review exemplar |
| **ADRs ficam Proposed forever** | SLA: decisão ≤ 2 semanas | Escalar, facilitar, ser tie-breaker |
| **ADRs cross-team não são lidos** | RFC → ADR pipeline; comunicação ativa | Ser bridge entre times |
| **ADRs sem manutenção** | Review cadence trimestral | Liderar ADR review sessions |
| **Time novo ignora ADRs** | Onboarding inclui ADRs-chave | Curar "ADRs essenciais" para onboarding |
| **Decisões contraditórias** | ADR dependency graph | Manter grafo de relacionamentos |

---

## Sumário

- [Architecture Decision Records — Governança em Escala](#architecture-decision-records--governança-em-escala)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [ADRs em Organizações Grandes](#adrs-em-organizações-grandes)
  - [Topologias de Organização de ADRs](#topologias-de-organização-de-adrs)
  - [Cross-Team ADRs](#cross-team-adrs)
  - [Governança Leve (Lightweight Governance)](#governança-leve-lightweight-governance)
  - [Discovery e Navegação](#discovery-e-navegação)
  - [Lifecycle Management](#lifecycle-management)
  - [Métricas de ADR](#métricas-de-adr)
  - [Cultura de Documentação de Decisões](#cultura-de-documentação-de-decisões)
  - [O Papel do Principal Engineer](#o-papel-do-principal-engineer)
  - [ADR Maturity Model](#adr-maturity-model)
  - [Automação e CI/CD](#automação-e-cicd)
  - [Anti-Patterns Organizacionais](#anti-patterns-organizacionais)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## ADRs em Organizações Grandes

### O problema de escala

```
1 time, 5 devs, 1 repo:
  → ADRs no repo, todos leem, funciona naturalmente.

10 times, 50 devs, 20 repos:
  → "Onde está aquela decisão sobre autenticação?"
  → "O time X já resolveu isso?"
  → "Quem decidiu usar Kafka? Posso mudar para SQS?"

50 times, 500 devs, 100+ repos:
  → Decisões se perdem em repos que ninguém lê
  → Times tomam decisões contraditórias
  → Novo engenheiro não sabe o que já foi decidido
  → Decisões cross-team não têm owner claro

PROBLEMA CENTRAL:
  ADRs são LOCAIS (por repo/time)
  mas decisões arquiteturais são frequentemente GLOBAIS
```

### Dimensões de escala

```
┌─────────────────────────────────────────────────────────┐
│           DIMENSÕES DE ESCALA PARA ADRs                   │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. VOLUME                                               │
│     → Centenas de ADRs, como encontrar o relevante?      │
│     → Solução: Index, search, categorização              │
│                                                          │
│  2. DISTRIBUIÇÃO                                         │
│     → ADRs espalhados em N repos                         │
│     → Solução: Repo central + links, ou aggregator       │
│                                                          │
│  3. AUDIÊNCIA                                            │
│     → Quem precisa ler quais ADRs?                       │
│     → Solução: Categorização por domínio/impacto         │
│                                                          │
│  4. LIFECYCLE                                            │
│     → ADRs ficam stale, desatualizados                   │
│     → Solução: Review cadence, ownership, expiry         │
│                                                          │
│  5. CONSISTÊNCIA                                         │
│     → Times usam formatos diferentes                     │
│     → Solução: Template org-wide, linting                │
│                                                          │
│  6. GOVERNANÇA                                           │
│     → Quem aprova decisões cross-team?                   │
│     → Solução: Governance tiers com approval matrix      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Topologias de Organização de ADRs

### Topologia 1: Descentralizado (per-repo)

```
REPOS:                    ADRs:

order-service/            order-service/docs/adrs/
  docs/adrs/                0001-postgresql.md
                            0002-cqrs-pattern.md

payment-service/          payment-service/docs/adrs/
  docs/adrs/                0001-stripe-integration.md
                            0002-idempotency.md

catalog-service/          catalog-service/docs/adrs/
  docs/adrs/                0001-dynamodb.md
                            0002-search-engine.md

✅ Prós:
  • ADRs próximos do código que afetam
  • Ownership natural (time do repo)
  • PR review integrado ao workflow do time

❌ Contras:
  • Decisões cross-team ficam "homeless"
  • Difícil buscar decisões globalmente
  • Numeração independente por repo (ADR-0001 existe N vezes)
  • Novos engenheiros não sabem onde buscar

QUANDO USAR:
  Orgs pequenas (< 10 times), repos independentes
```

### Topologia 2: Centralizado

```
REPO CENTRAL:

architecture-decisions/
  org-wide/
    0001-api-gateway-pattern.md
    0002-observability-stack.md
    0003-authentication-standard.md
  
  domains/
    payments/
      0001-stripe-integration.md
      0002-idempotency.md
    orders/
      0001-postgresql.md
      0002-cqrs-pattern.md
    catalog/
      0001-dynamodb.md

✅ Prós:
  • Um lugar para buscar todas as decisões
  • Global numbering possível
  • Fácil para onboarding
  • Cross-team decisions têm home

❌ Contras:
  • ADRs desconectados do código
  • PRs em repo que o time não acompanha
  • Tende a ficar desatualizado
  • Governance overhead

QUANDO USAR:
  Decisões org-wide, padrões transversais
```

### Topologia 3: Híbrida (recomendada)

```
MODELO HÍBRIDO:

  ┌─────────────────────────────────────────────────┐
  │           REPO CENTRAL (architecture)            │
  │                                                  │
  │  org-wide/                                       │
  │    0001-api-gateway-pattern.md                   │
  │    0002-observability-stack.md                   │
  │    0003-authentication-standard.md               │
  │                                                  │
  │  index.md  ← Índice GLOBAL com links para       │
  │              ADRs locais nos repos               │
  └──────────────────────┬──────────────────────────┘
                         │ Links
        ┌────────────────┼────────────────┐
        │                │                │
  ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
  │ order-svc │   │ payment   │   │ catalog   │
  │ docs/adrs/│   │ docs/adrs/│   │ docs/adrs/│
  │ (local)   │   │ (local)   │   │ (local)   │
  └───────────┘   └───────────┘   └───────────┘

REGRAS:
  • Decisões que afetam 1 time → repo local
  • Decisões que afetam 2+ times → repo central
  • Padrões/standards org-wide → repo central
  • Índice no repo central agrega TODOS (locais + centrais)

✅ PRÓS:
  • Decisões locais perto do código
  • Decisões globais em lugar único
  • Um índice para discovery
  • Ownership claro

QUANDO USAR:
  Orgs médias/grandes (10+ times)
```

---

## Cross-Team ADRs

### Quando uma decisão é cross-team

```
CRITÉRIOS:

1. IMPACTO
   A decisão afeta contratos entre serviços?
   → API contracts, event schemas, auth patterns

2. CONSTRAINT
   A decisão impõe constraint a outros times?
   → "Todos devem usar JWT para autenticação"
   → "Eventos devem usar CloudEvents spec"

3. DEPENDENCY
   Outro time precisa mudar algo por causa desta decisão?
   → "Payment team precisa adaptar webhook format"

4. PRECEDENT
   A decisão cria precedente que outros vão seguir?
   → "Se order-service usa PostgreSQL, catalog vai querer também"

SE ≥ 1 CRITÉRIO = CROSS-TEAM ADR
```

### Pipeline: RFC → ADR para decisões grandes

```
DECISÕES GRANDES (CROSS-ORG):

  ┌────────────────┐
  │   RFC (Request  │   Documento exploratório
  │   For Comments) │   • Múltiplas opções detalhadas
  │                 │   • Spike/PoC results
  │   2-10 páginas  │   • Ampla consulta (30-60 dias)
  └───────┬────────┘
          │ Após consenso
  ┌───────▼────────┐
  │   ADR           │   Captura a DECISÃO
  │                 │   • 1-2 páginas
  │                 │   • Referencia RFC
  │                 │   • Registro permanente
  └───────┬────────┘
          │ Implementação
  ┌───────▼────────┐
  │   Design Docs   │   Detalhamento técnico
  │                 │   • Como implementar
  │                 │   • Referencia ADR
  └────────────────┘

QUANDO USAR RFC → ADR:
  • Decisão afeta 5+ times
  • Investimento > 1 quarter
  • Mudança de plataforma / stack
  • Decisão irreversível (Tipo 1) de alto impacto

QUANDO PULAR RFC (só ADR):
  • Decisão de 1-2 times
  • Opções claras, não precisa consulta ampla
  • Time já tem dados suficientes
```

### Cross-team ADR ownership

```
QUEM É OWNER DE UM CROSS-TEAM ADR?

1. AUTOR
   Quem identificou a necessidade e escreveu o draft.
   Frequentemente: Principal/Staff Engineer

2. SPONSOR
   Engineering Manager ou VP que suporta a iniciativa.
   Garante prioridade e recursos para implementação.

3. REVIEWERS
   Tech Leads dos times impactados.
   TODOS os times impactados DEVEM review.

4. APPROVER
   Principal/Chief Architect (para org-wide)
   ou conjunto de Tech Leads (para cross-team)

LIFECYCLE OWNERSHIP:
  Após Accepted, quem mantém?
  → Não é o autor original (pode sair da empresa)
  → É o papel (Principal, Architecture Guild, Platform Team)
  → Ownership explícito no ADR: "Owner: Platform Team"
```

---

## Governança Leve (Lightweight Governance)

### Governance tiers

```
GOVERNANCE TIERS:

┌─────────────────────────────────────────────────────────────────┐
│  TIER 1: TEAM-LEVEL                                              │
│  Scope: Decisões internas de um time                             │
│  Approval: Tech Lead do time                                     │
│  Review: ≥ 2 engineers do time                                   │
│  SLA: ≤ 1 semana                                                 │
│  Exemplo: "Usar Redis para cache do catálogo"                    │
├─────────────────────────────────────────────────────────────────┤
│  TIER 2: CROSS-TEAM                                              │
│  Scope: Decisões que afetam 2-5 times                            │
│  Approval: Principal Engineer + Tech Leads dos times             │
│  Review: Tech Leads + seniors dos times impactados               │
│  SLA: ≤ 2 semanas                                                │
│  Exemplo: "Migrar events para CloudEvents spec"                  │
├─────────────────────────────────────────────────────────────────┤
│  TIER 3: ORG-WIDE                                                │
│  Scope: Standards / decisões que afetam toda a eng               │
│  Approval: Chief Architect / Architecture Board                  │
│  Review: Tech Leads + Principal Engineers + Compliance            │
│  SLA: ≤ 4 semanas                                                │
│  Exemplo: "Adotar OpenTelemetry como padrão de observabilidade"  │
└─────────────────────────────────────────────────────────────────┘

PRINCÍPIO: Governança mínima necessária.
  Não tratar decisão Tier 1 como Tier 3.
  → Burocracia excessiva = times param de escrever ADRs.
```

### Architecture Guild / Advisory Board

```
ARCHITECTURE GUILD:

Composição:
  • Principal Engineers (obrigatório)
  • Staff Engineers (representantes de domínio)
  • Tech Leads seniores (representantes de time)
  • Rotating seat (qualquer engineer pode participar)

Responsabilidades:
  • Review de ADRs Tier 2 e Tier 3
  • Manutenção do ADR Index global
  • Resolução de conflitos entre ADRs
  • Review cadence trimestral
  • Onboarding: curar "ADRs essenciais"

Cadência:
  • Semanal: 30min standup — novos ADRs, status
  • Mensal: 1h deep-dive — review de ADRs pendentes
  • Trimestral: 2h — revisão de ADRs existentes

ANTI-PATTERN: "Architecture Board de aprovação"
  ❌ Guild como gatekeeper que atrasa decisões
  ✅ Guild como facilitador que acelera boas decisões
  ✅ Guild como reviewer, não approver (approval é do owner)
```

---

## Discovery e Navegação

### Como tornar ADRs encontráveis

```
CAMADAS DE DISCOVERY:

1. ADR INDEX (arquivo Markdown)
   Tabela com todos os ADRs: número, título, status, data
   Gerado automaticamente ou mantido manualmente
   → Nível: Repo (local) ou Org (global)

2. SEARCH
   Full-text search nos ADRs
   → Log4brains: site estático com search
   → GitHub/GitLab search: básico mas funciona
   → Custom: Algolia, MeiliSearch em site de docs

3. TAGS / CATEGORIAS
   Tags nos ADRs: domain, technology, impact-area
   → Metadata no header YAML:
     ---
     tags: [database, postgresql, orders]
     domain: order-management
     impact: team-level
     ---

4. DEPENDENCY GRAPH
   Visualização de relacionamentos entre ADRs
   → ADR-0042 supersedes ADR-0001
   → ADR-0042 relates-to ADR-0035
   → Mermaid diagram no index
```

### ADR metadata template

```yaml
---
adr: "0042"
title: "PostgreSQL for order-service"
status: "accepted"       # proposed | accepted | deprecated | superseded | rejected
date: "2024-02-10"
decision-makers:
  - "@senior-dev"        # Recommend
  - "@tech-lead"         # Decide
  - "@principal-eng"     # Input
reviewers:
  - "@team-lead-payments"
  - "@dba-team"
tags:
  - database
  - postgresql
  - order-management
domain: "order-management"
tier: "team-level"       # team-level | cross-team | org-wide
supersedes: "0001"
related:
  - "0035"               # CQRS pattern
  - "0038"               # Data migration strategy
---
```

### Mermaid dependency graph

```
VISUALIZAÇÃO DE DEPENDÊNCIA:

graph LR
  ADR0001[ADR-0001<br>MySQL for orders]
  ADR0035[ADR-0035<br>CQRS pattern]
  ADR0038[ADR-0038<br>Data migration strategy]
  ADR0042[ADR-0042<br>PostgreSQL for orders]
  
  ADR0042 -->|supersedes| ADR0001
  ADR0042 -->|relates-to| ADR0035
  ADR0042 -->|relates-to| ADR0038

USO:
  No ADR Index (README.md), incluir Mermaid diagram
  que mostra relacionamentos entre ADRs-chave.
  Não é necessário para TODOS os ADRs — focar em
  ADRs Tier 2/3 com múltiplas dependências.
```

---

## Lifecycle Management

### ADR Review Cadence

```
REVIEW CADENCE:

┌──────────────────────────────────────────────────────────┐
│  TRIMESTRAL: ADR Health Check                             │
│                                                           │
│  Para cada ADR com status Accepted:                       │
│                                                           │
│  1. O contexto ainda é válido?                            │
│     → Constraints mudaram? Budget? Team size?             │
│     → Tecnologia evoluiu? Nova opção disponível?          │
│                                                           │
│  2. A decisão está sendo seguida?                         │
│     → Código implementa o que ADR decidiu?                │
│     → Exceções não documentadas?                          │
│                                                           │
│  3. As consequências previstas se confirmaram?            │
│     → Latência atingiu target? Custo dentro do budget?    │
│     → Consequências negativas piores que o esperado?      │
│                                                           │
│  RESULTADO:                                               │
│  → ADR ainda válido → nenhuma ação                        │
│  → Contexto mudou → novo ADR superseding                  │
│  → Decisão não seguida → enforce ou supersede             │
│  → Consequências piores → avaliar migração                │
│                                                           │
│  FORMATO: Sessão de 1-2h com Architecture Guild           │
│  Output: Lista de ações (novos ADRs, deprecations)        │
└──────────────────────────────────────────────────────────┘
```

### Superseding vs Deprecating

```
SUPERSEDE:
  "A decisão estava boa, mas agora temos uma MELHOR."
  
  Exemplo:
    ADR-0001 (MySQL): Status → Superseded by ADR-0042
    ADR-0042 (PostgreSQL): Status → Accepted
    ADR-0042 Context menciona: "Substitui ADR-0001 porque..."

  NUNCA delete ou edite o ADR original.
  ADRs são imutáveis. Adicione link no header:
  "Superseded by: [ADR-0042](0042-postgresql-for-orders.md)"

DEPRECATE:
  "A decisão era boa, mas a situação mudou e ela 
   não é mais relevante (sem substituto direto)."
  
  Exemplo:
    ADR-0015 (Redis Cluster for sessions): Status → Deprecated
    Reason: "Migramos para JWT stateless, sessions não existem mais."
  
  ADR-0015 header ganha:
  "Deprecated: reason — migração para JWT stateless (2024-06)"

REJECT:
  "A proposta foi avaliada e rejeitada."
  Útil manter para evitar que alguém proponha o mesmo depois.
  
  Exemplo:
    ADR-0020 (GraphQL Federation): Status → Rejected
    Reason: "Complexidade operacional desproporcional. Team size insuficiente."
```

---

## Métricas de ADR

### Métricas de saúde

| Métrica | Target | Como medir | Ação se fora |
|---------|--------|-----------|-------------|
| **ADRs por quarter** | 3-8 (por time de 5-8) | Contar novos ADRs | < 3: incentivar; > 8: review se necessário |
| **Lead time** (Proposed → Accepted) | ≤ 2 semanas | Data diff | > 2 sem: escalar, facilitar |
| **% ADR com review** | 100% | PR reviews | < 100%: reforçar processo |
| **% ADR stale** (Accepted > 2 anos, never reviewed) | ≤ 20% | Last touched date | > 20%: ADR health check |
| **% ADR com consequências negativas** | > 80% | Content check | < 80%: coaching — ADRs otimistas |
| **% ADR referenciado em PRs** | > 50% | Busca texto "ADR-" nos PRs | < 50%: linking culture |
| **ADRs Proposed > 2 semanas** | 0 | Status + date | > 0: resolve imediatamente |

### O que NÃO medir

```
❌ NÃO MEDIR:
  • "Número de ADRs" como KPI de produtividade
    → Incentiva criar ADRs desnecessários
  
  • "Todos os ADRs devem ser Accepted"
    → Rejected é um resultado válido
  
  • "Tempo de escrita do ADR"
    → Incentiva ADRs superficiais
  
  • "Coverage: % de decisões com ADR"
    → Impossível definir o denominador

PRINCÍPIO: 
  Métricas servem para detectar PROBLEMAS no processo,
  não para medir PRODUTIVIDADE de indivíduos.
```

---

## Cultura de Documentação de Decisões

### Como criar uma cultura de ADRs

```
ADOÇÃO EM FASES:

FASE 1: SEED (1 time, 1-2 meses)
  → Time piloto escreve 3-5 ADRs
  → Principal Engineer modela comportamento
  → Feedback: formato, processo, tooling
  Métrica: time piloto adota consistentemente

FASE 2: SCALE (3-5 times, 2-4 meses)
  → Outros times adotam, baseado em exemplo do piloto
  → Template padronizado org-wide
  → ADR Index no repo central
  → Workshop de 30min para novos times
  Métrica: ≥ 3 times escrevendo ADRs regularmente

FASE 3: EMBED (toda eng, 4-6 meses)
  → ADR é parte do workflow padrão
  → CI/CD checks (lint, template compliance)
  → Onboarding inclui ADRs-chave
  → Architecture Guild mantém index e review cadence
  Métrica: ADRs são referenciados em PRs e discussões

FASE 4: SUSTAIN (contínuo)
  → Review cadence trimestral
  → Métricas de saúde monitoradas
  → Novos engenheiros já encontram ADRs como prática padrão
  → Decisões sem ADR levantam flag naturalmente:
    "Onde está o ADR disso?"
```

### Incentivos positivos

```
O QUE FUNCIONA:

  ✅ "ADR Hall of Fame" 
     → Destacar ADR bem escrito no all-hands
     → Modelo para outros

  ✅ ADR como parte de tech talks
     → "Decisão do mês" — analisar um ADR com o time

  ✅ Onboarding: "Os 10 ADRs que você precisa ler"
     → Curadoria pelo Principal Engineer

  ✅ PRs que referenciam ADR → feedback positivo no review
     → "Boa prática linkar ADR-0042 aqui 👍"

  ✅ Principal Engineer escreve ADR junto com o time
     → Pairing em draft, não review top-down

O QUE NÃO FUNCIONA:

  ❌ "Todo PR requer ADR" — burocracia excessiva
  ❌ "ADR como KPI" — gaming inevitável
  ❌ Board de aprovação — times param de escrever
  ❌ Template super pesado — 30min para escrever = ninguém escreve
```

---

## O Papel do Principal Engineer

### Principal Engineer como steward de decisões

```
RESPONSABILIDADES DO PRINCIPAL ENGINEER:

1. MODELAR COMPORTAMENTO
   → Escrever ADRs próprios como exemplo
   → Padrão de qualidade que times podem replicar
   → "Faço o que digo" — não apenas review, mas autoria

2. FACILITAR DECISÕES CROSS-TEAM
   → Identificar quando decisão precisa ser elevada
   → Conectar times que estão resolvendo problemas similares
   → Ser bridge entre domínios: "Time X tem ADR sobre isso"

3. MANTER COERÊNCIA ARQUITETURAL
   → Review de ADRs Tier 2 e Tier 3
   → Identificar decisões contraditórias
   → Manter dependency graph atualizado

4. COACHING
   → Ajudar engineers a escrever primeiro ADR
   → Feedback construtivo em reviews
   → Ensinar técnicas de argumentação e trade-off analysis

5. CURAR KNOWLEDGE BASE
   → "Top 10 ADRs para onboarding"
   → ADR Index atualizado
   → Review cadence trimestral

6. ESCALAR DECISÕES TRAVADAS
   → ADR Proposed > 2 semanas → Principal intervém
   → Impasse entre times → facilitar sessão de decisão
   → Quando necessário → ser tie-breaker (consent-based)
```

### Principal Engineer anti-patterns

```
❌ "IVORY TOWER ARCHITECT"
   → Escreve ADRs sozinho, sem consultar times
   → Times não se sentem donos das decisões
   Fix: Escrever COM os times, não PARA os times

❌ "BOTTLENECK DE APROVAÇÃO"
   → Toda ADR precisa de approval do Principal
   → Grid-locks em época de muitas decisões
   Fix: Principal review de Tier 2/3, não de todas

❌ "REVIEWER QUE SÓ CRITICA"
   → Aponta problemas sem propor soluções
   → Times precisam de orientação, não de obstáculos
   Fix: Review construtivo com sugestões concretas

❌ "GHOST REVIEWER"
   → Assignado como reviewer, demora 3 semanas
   → Bloqueia decisão do time
   Fix: SLA de review: ≤ 3 dias úteis

❌ "ADR DOGMATIST"
   → Todo ADR deve ter 7 seções preenchidas perfeitamente
   → Para decisões Tipo 2, um ADR leve é suficiente
   Fix: Calibrar peso do ADR com importância da decisão
```

---

## ADR Maturity Model

```
┌──────────────────────────────────────────────────────────────┐
│              ADR MATURITY MODEL                                │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  LEVEL 0: AD HOC                                              │
│  → Decisões são tomadas em Slack/meetings                     │
│  → Nenhuma documentação formal                                │
│  → Novos membros: "por que fazemos assim?" → ninguém sabe     │
│                                                               │
│  LEVEL 1: EXPERIMENTAL                                        │
│  → 1-2 times experimentam ADRs                                │
│  → Formato inconsistente                                      │
│  → ADRs existem mas ninguém referencia                        │
│                                                               │
│  LEVEL 2: STANDARDIZED                                        │
│  → Template padronizado org-wide                              │
│  → Múltiplos times adotando                                   │
│  → ADR Index existe                                           │
│  → PR review process defined                                  │
│                                                               │
│  LEVEL 3: INTEGRATED                                          │
│  → ADRs referenciados em PRs consistentemente                 │
│  → CI checks para formato/lint                                │
│  → Cross-team ADRs em repo central                            │
│  → Architecture Guild mantém review cadence                   │
│  → Onboarding inclui ADRs-chave                               │
│                                                               │
│  LEVEL 4: OPTIMIZED                                           │
│  → Métricas de saúde monitoradas                              │
│  → ADR dependency graph atualizado                            │
│  → Cultura: "Onde está o ADR disso?" é natural                │
│  → Review trimestral com actions claras                       │
│  → ADRs informam roadmap e tech radar                         │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Automação e CI/CD

### Linting e validação automática

```yaml
# .github/workflows/adr-lint.yml
name: ADR Lint
on:
  pull_request:
    paths: ['docs/adrs/**']

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Validate ADR format
        run: |
          for file in docs/adrs/*.md; do
            echo "Checking $file..."
            
            # Verifica seções obrigatórias
            grep -q "## Status" "$file" || echo "WARN: Missing Status in $file"
            grep -q "## Context" "$file" || echo "WARN: Missing Context in $file"  
            grep -q "## Decision" "$file" || echo "WARN: Missing Decision in $file"
            grep -q "## Consequences" "$file" || echo "WARN: Missing Consequences in $file"
            
            # Verifica naming convention NNNN-slug.md
            basename "$file" | grep -qE '^[0-9]{4}-[a-z0-9-]+\.md$' || \
              echo "WARN: Invalid naming in $file (expected NNNN-slug.md)"
          done
      
      - name: Check for stale ADRs
        run: |
          # ADRs Accepted há mais de 2 anos sem review
          find docs/adrs/ -name "*.md" -mtime +730 | while read f; do
            echo "STALE: $f (not modified in 2+ years)"
          done
```

### Geração automática de index

```bash
#!/bin/bash
# scripts/generate-adr-index.sh

echo "# ADR Index" > docs/adrs/README.md
echo "" >> docs/adrs/README.md
echo "| # | Title | Status | Date |" >> docs/adrs/README.md
echo "|---|-------|--------|------|" >> docs/adrs/README.md

for file in docs/adrs/[0-9]*.md; do
  num=$(basename "$file" | grep -oE '^[0-9]+')
  title=$(head -1 "$file" | sed 's/^# //')
  status=$(grep -m1 "^## Status" -A1 "$file" | tail -1 | tr -d '[:space:]')
  date=$(git log --format="%ai" --diff-filter=A -- "$file" | cut -d' ' -f1)
  
  echo "| $num | [$title]($file) | $status | $date |" >> docs/adrs/README.md
done
```

---

## Anti-Patterns Organizacionais

| Anti-pattern | Sintoma | Consequência | Correção |
|-------------|---------|-------------|----------|
| **"Architecture by Slack"** | Decisões em threads que desaparecem | Ninguém lembra por que | Regra: se gerou debate > 10 msg, precisa de ADR |
| **"Review board bottleneck"** | Decisões demoram meses | Times desistem de propor | Governance tiers: Tier 1 não precisa de board |
| **"ADR-only-for-big-decisions"** | Apenas 2 ADRs por ano | Decisões médias se perdem | Calibrar: ADR leve para decisões menores |
| **"Template overload"** | Template de 15 seções | Times não preenchem | ADR mínimo: 4 seções. MADR para decisões maiores |
| **"Central repo nobody reads"** | Repo com 200 ADRs desatualizados | Zero confiança nos ADRs | Review cadence + limpeza + search tool |
| **"Conflicting ADRs"** | Time A decide X, Time B decide anti-X | Arquitetura inconsistente | Architecture Guild + dependency graph |
| **"ADR-as-post-mortem"** | ADRs escritos meses depois | Contexto impreciso, alternativas inventadas | SLA: ADR antes/durante decisão, nunca depois |
| **"Premature standardization"** | Org impõe ADR antes de times entenderem | Resistência e compliance vazio | Seed → Scale → Embed (fases) |

---

## Diretrizes para Code Review assistido por AI

Ao revisar governança de ADRs e práticas organizacionais, verifique:

1. **Decisão cross-team em repo local** — Decisões que afetam múltiplos times devem estar no repo central
2. **Ausência de ADR Index** — Toda organização com > 10 ADRs precisa de índice navegável
3. **Governance tier mismatch** — Decisão Tier 1 com burocracia Tier 3 ou vice-versa
4. **ADR Proposed por > 2 semanas sem ação** — Escalar ou facilitar decisão imediatamente
5. **ADR sem ownership explícito** — Cross-team ADRs precisam de owner (papel, não pessoa)
6. **ADR stale (> 2 anos, nunca revisado)** — Incluir em review cadence trimestral
7. **Decisões em Slack/meetings sem ADR** — Se debate > 10 mensagens, provavelmente precisa de ADR
8. **Architecture Guild como bottleneck** — Guild facilita, não bloqueia. SLA de review ≤ 3 dias
9. **Template excessivamente pesado** — Para Tipo 2, ADR mínimo (4 seções) é suficiente
10. **ADR sem metadata de categorização** — Tags, domain, tier ajudam discovery em escala
11. **Principal Engineer escreve todos os ADRs** — Deve facilitar e modelar, não centralizar autoria
12. **Sem review cadence trimestral** — ADRs acumulam staleness sem revisão periódica

---

## Referências

- **Michael Nygard** — "Documenting Architecture Decisions" (2011) — Original ADR blog post
- **MADR** — Markdown Any Decision Records — https://adr.github.io/madr/
- **ThoughtWorks Tech Radar** — Lightweight ADRs (Adopt)
- **Team Topologies** — Matthew Skelton, Manuel Pais — Team interaction patterns
- **The Staff Engineer's Path** — Tanya Reilly — Advocacy, influence, decision facilitation
- **Staff Engineer** — Will Larson — Organizational leverage and decision documentation
- **Sociocracy 3.0** — Consent-based decision making — https://sociocracy30.org/
- **Accelerate** — Forsgren, Humble, Kim — Lightweight change approval processes
- **RAPID Decision Making** — Bain & Company — Roles in decision making
- **Architecture Guild patterns** — ThoughtWorks, Spotify Engineering Culture
- **Log4brains** — https://github.com/thomvaill/log4brains
- **adr-tools** — https://github.com/npryce/adr-tools
