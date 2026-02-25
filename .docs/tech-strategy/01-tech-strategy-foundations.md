# Technical Strategy — Fundamentos e Tech Radar

> **Objetivo deste documento:** Servir como referência completa sobre **estratégia técnica, Tech Radar, ciclo de vida de tecnologias e frameworks de decisão estratégica** — como pensar, avaliar e comunicar decisões tecnológicas de alto impacto, otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: tech strategy como disciplina, Technology Radar, technology lifecycle, strategic alignment, governança tecnológica.

---

## Quick Reference — Cheat Sheet

| Conceito | Definição | Aplicação |
|----------|----------|-----------|
| **Tech Strategy** | Plano de longo prazo que alinha tecnologia com objetivos de negócio | Decisões de plataforma, linguagem, cloud |
| **Tech Radar** | Mapa visual de adoção de tecnologias (Adopt/Trial/Assess/Hold) | Governança de adoção technológica |
| **Technology Lifecycle** | Curva de maturidade: Emerging → Growth → Mature → Decline | Timing de adoção |
| **Strategic Fitness** | Adequação da tecnologia ao contexto específico da organização | Evita cargo-culting |
| **Reversibility** | Quão fácil/custoso é reverter uma decisão tecnológica | Risk assessment |
| **Blast Radius** | Número de sistemas/equipes afetados por uma decisão | Governança tier |

---

## Sumário

- [Technical Strategy — Fundamentos e Tech Radar](#technical-strategy--fundamentos-e-tech-radar)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que é Technical Strategy](#o-que-é-technical-strategy)
  - [Pensamento Estratégico vs Tático](#pensamento-estratégico-vs-tático)
  - [Technology Radar](#technology-radar)
  - [Technology Lifecycle](#technology-lifecycle)
  - [Princípios de Decisão Tecnológica](#princípios-de-decisão-tecnológica)
  - [Strategic Alignment](#strategic-alignment)
  - [Governança Tecnológica](#governança-tecnológica)
  - [Comunicação de Estratégia](#comunicação-de-estratégia)
  - [Anti-Patterns de Tech Strategy](#anti-patterns-de-tech-strategy)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que é Technical Strategy

```
TECHNICAL STRATEGY = PLANO DE LONGO PRAZO QUE CONECTA
                     DECISÕES TÉCNICAS A OBJETIVOS DE NEGÓCIO

  ┌─────────────────────────────────────────────────────────────┐
  │                                                              │
  │   BUSINESS STRATEGY                                          │
  │   "Expandir para 5 mercados LATAM em 18 meses"              │
  │                                                              │
  │          │                                                   │
  │          ▼                                                   │
  │                                                              │
  │   TECHNICAL STRATEGY                                         │
  │   ┌─────────────────────────────────────────────────┐       │
  │   │ 1. Multi-region architecture (latência < 100ms) │       │
  │   │ 2. Multi-currency / i18n support                │       │
  │   │ 3. Compliance (LGPD, regulações locais)         │       │
  │   │ 4. Escalar de 10K → 500K TPS                    │       │
  │   │ 5. Reduzir deploy cycle de 2 weeks → 1 day      │       │
  │   └─────────────────────────────────────────────────┘       │
  │                                                              │
  │          │                                                   │
  │          ▼                                                   │
  │                                                              │
  │   TECHNICAL ROADMAP                                          │
  │   Q1: Multi-region infra + feature flags                     │
  │   Q2: i18n refactor + compliance audit                       │
  │   Q3: Performance engineering + auto-scaling                 │
  │   Q4: CI/CD pipeline optimization                            │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘

  O QUE TECH STRATEGY NÃO É:
  
  ❌ Lista de tecnologias "legais" que queremos usar
  ❌ "Vamos reescrever tudo em Rust"
  ❌ Decisões técnicas sem contexto de negócio
  ❌ Uma vez criada, nunca revisitada
  
  O QUE TECH STRATEGY É:
  
  ✅ Resposta técnica a necessidades de negócio
  ✅ Framework para tomar decisões consistentes
  ✅ Documento vivo, revisado trimestralmente
  ✅ Comunicação clara de direção e trade-offs
  ✅ Base para tech roadmaps e priorização
```

### Os 4 pilares de tech strategy

```
OS 4 PILARES:

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │              │  │              │  │              │  │              │
  │  DIRECTION   │  │  PRINCIPLES  │  │  PORTFOLIO   │  │  GOVERNANCE  │
  │              │  │              │  │              │  │              │
  │  Para onde   │  │  Como        │  │  Com o que   │  │  Como        │
  │  vamos?      │  │  decidimos?  │  │  trabalhamos?│  │  mantemos?   │
  │              │  │              │  │              │  │              │
  │ • Visão      │  │ • Princípios │  │ • Tech Radar │  │ • ADRs       │
  │ • Horizonte  │  │   de design  │  │ • Build/Buy  │  │ • Reviews    │
  │ • Roadmap    │  │ • Trade-offs │  │ • Tech Debt  │  │ • Standards  │
  │ • KPIs       │  │   explícitos │  │ • Skills map │  │ • Compliance │
  │              │  │              │  │              │  │              │
  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘

  DIRECTION: define o destino
  → Visão técnica de 1-3 anos
  → Alinhada com business strategy
  → Traduzida em roadmap executável
  
  PRINCIPLES: define como tomamos decisões
  → "Preferimos boring technology" (Dan McKinley)
  → "Consistência sobre perfeição local"
  → "Reversibilidade sempre que possível"
  
  PORTFOLIO: define o que usamos
  → Tech Radar: que tecnologias adotamos/evitamos
  → Build vs Buy: quando construir vs comprar
  → Skills inventory: que skills temos/precisamos
  
  GOVERNANCE: define como mantemos a disciplina
  → ADRs para decisões significativas
  → Architecture reviews para mudanças cross-cutting
  → Standards e guidelines documentados
  → Compliance e security requirements
```

---

## Pensamento Estratégico vs Tático

```
ESTRATÉGICO vs TÁTICO — QUANDO USAR CADA:

  │ Dimensão      │ Tático                     │ Estratégico                │
  │ Horizonte     │ Dias/Semanas               │ Meses/Anos                 │
  │ Reversibilidade│ Alta (facilmente revertido)│ Baixa (custoso reverter)   │
  │ Blast radius  │ 1 serviço, 1 equipe        │ Múltiplos serviços/equipes │
  │ Quem decide   │ Tech Lead / equipe          │ Staff+ / Architecture      │
  │ Documentação  │ PR description              │ ADR + Tech Strategy doc    │
  │ Exemplo       │ Lib de logging              │ Plataforma de observability│
  │ Exemplo       │ Refactor de 1 classe       │ Migração de monolito       │
  │ Exemplo       │ Fix de performance          │ Mudança de database engine │

  JEFF BEZOS — TYPE 1 vs TYPE 2 DECISIONS:

  TYPE 1 (One-Way Door) → ESTRATÉGICO
  → Difícil/impossível de reverter
  → Alto custo de mudança
  → Exige: análise profunda, consensus, ADR
  → Exemplos: linguagem principal, cloud provider, 
    database engine, API pública

  TYPE 2 (Two-Way Door) → TÁTICO
  → Facilmente reversível
  → Baixo custo de mudança
  → Exige: decisão rápida, bias for action
  → Exemplos: lib interna, framework de testes,
    ferramenta de CI, linter

  REGRA:
  → Type 2 decisions devem ser tomadas RÁPIDO
     (não gaste 3 semanas decidindo entre Mockito e JMockit)
  → Type 1 decisions devem ser tomadas com CUIDADO
     (gaste tempo avaliando antes de migrar para novo DB)
  → A MAIORIA das decisões são Type 2 
     (mas muitas orgs tratam todas como Type 1 → paralisia)
```

---

## Technology Radar

### O que é Tech Radar

```
TECHNOLOGY RADAR (ThoughtWorks):

  Mapa visual que categoriza tecnologias em 4 RINGS + 4 QUADRANTS

  RINGS (nível de adoção):

  ┌───────────────────────────────────────────────────────┐
  │                                                        │
  │                      ┌─────────┐                       │
  │                  ┌───┤  ADOPT  ├───┐                   │
  │              ┌───┤   └─────────┘   ├───┐               │
  │          ┌───┤   │     TRIAL       │   ├───┐           │
  │      ┌───┤   │   └────────────────┘   │   ├───┐       │
  │      │   │   │        ASSESS          │   │   │       │
  │      │   │   └────────────────────────┘   │   │       │
  │      │   │            HOLD                │   │       │
  │      │   └────────────────────────────────┘   │       │
  │      └────────────────────────────────────────┘       │
  │                                                        │
  └───────────────────────────────────────────────────────┘

  ADOPT  — USE EM PRODUÇÃO com confiança
  → Experiência comprovada, riscos conhecidos
  → Default choice para novos projetos
  → Ex: Kubernetes, PostgreSQL, OpenTelemetry

  TRIAL  — EXPERIMENTE em projetos selecionados  
  → Promissor, precisa mais experiência
  → 1-2 equipes piloto, com acompanhamento
  → Ex: Dagger CI, Temporal, WASM

  ASSESS — AVALIE, pesquise, protótipo
  → Interessante, mas sem experiência prática
  → Proof of concept, não produção
  → Ex: CRDTs para sync, WebTransport

  HOLD   — NÃO ADOTE para novos projetos
  → Legacy aceitável, mas não iniciar novo uso
  → Pode significar "migrando para fora"
  → Ex: Jenkins (→ GitHub Actions), MongoDB para dados relacionais

  QUADRANTS (categorias de tecnologia):

  ┌────────────────────┬────────────────────┐
  │                    │                    │
  │    TECHNIQUES      │    PLATFORMS       │
  │                    │                    │
  │  Práticas, métodos │  Infra, cloud,     │
  │  processos         │  plataformas       │
  │                    │                    │
  │  Ex: Trunk-based   │  Ex: AWS EKS,      │
  │  development,      │  Terraform,        │
  │  Feature flags     │  Kafka             │
  │                    │                    │
  ├────────────────────┼────────────────────┤
  │                    │                    │
  │    TOOLS           │    LANGUAGES &     │
  │                    │    FRAMEWORKS      │
  │  Dev tools,        │                    │
  │  libraries,        │  Linguagens,       │
  │  utilities         │  frameworks        │
  │                    │                    │
  │  Ex: k6, Grafana,  │  Ex: Go, Kotlin,   │
  │  AsyncAPI          │  Spring Boot 3,    │
  │                    │  htmx              │
  └────────────────────┴────────────────────┘
```

### Como construir um Tech Radar interno

```
PROCESSO PARA CRIAR TECH RADAR DA SUA ORG:

  STEP 1: INVENTÁRIO
  → Listar TODAS as tecnologias em uso
  → Fonte: repos, Dockerfiles, package.json, build files
  → Tool: Backstage tech insights, manual survey
  
  Resultado: "Temos 5 bancos de dados, 3 linguagens,
              4 frameworks web, 2 message brokers"

  STEP 2: CLASSIFICAR POR RING
  → Para cada tecnologia, perguntar:
    - Experiência interna positiva? → ADOPT
    - Experimentando com sucesso? → TRIAL
    - Queremos explorar? → ASSESS
    - Queremos parar de usar? → HOLD
  
  → Quem classifica:
    - Architecture Guild / Staff+ engineers
    - Input de Tech Leads de cada equipe
    - Decisão por consenso (não votação)

  STEP 3: DOCUMENTAR RATIONALE
  → Cada blip (item no radar) tem:
    - Nome da tecnologia
    - Ring atual (ADOPT/TRIAL/ASSESS/HOLD)
    - Ring anterior (se mudou: "era TRIAL, agora ADOPT")
    - Descrição: por quê neste ring?
    - ADR associada (se houver)
    - Owner: quem é referência interna

  STEP 4: PUBLICAR E COMUNICAR
  → Formato visual (radar chart)
  → Documento companion com detalhes
  → All-hands ou tech blog post
  → Disponível para todos (não só architects)

  STEP 5: REVISAR REGULARMENTE
  → Frequência: trimestral ou semestral
  → Input: experiência acumulada, novidades de mercado
  → Mudanças: blips mudam de ring (TRIAL→ADOPT, etc.)
  → Cada mudança documentada (ADR se significativo)

  FERRAMENTAS PARA RADAR:
  → ThoughtWorks Build Your Own Radar (BYOR)
    https://www.thoughtworks.com/radar/byor
  → Backstage Tech Radar plugin
  → Miro/FigJam (manual, baixa manutenção)
  → Spreadsheet + template de radar
```

### Exemplo de Tech Radar

```
EXEMPLO — TECH RADAR PARA UM TIME DE BACKEND:

  ══════════════════════════════════════════════════════════
  LANGUAGES & FRAMEWORKS
  ══════════════════════════════════════════════════════════
  
  ADOPT
  → Java 21 (LTS) + Spring Boot 3.x
    "Linguagem principal, equipe experiente, ecossistema maduro"
  → Go 1.22+
    "Serviços de alta performance, CLIs, infra tooling"
  → Kotlin (JVM)
    "Interop com Java, produtividade superior, coroutines"
  
  TRIAL
  → Quarkus 3.x
    "Startup rápido para serverless/containers leves.
     1 equipe pilotando em serviço de notificação."
  → Rust (para componentes de infra)
    "Performance crítica sem GC. Avaliando para 
     componente de processamento de stream."
  
  ASSESS
  → Zig
    "Alternativa a C para low-level.
     Investigar para WASM use cases."
  
  HOLD
  → Java 11 (EOS)
    "Migrar para Java 21. Não iniciar projetos em 11."
  → Spring Boot 2.x
    "Migrar para 3.x. Security, performance, Jakarta EE."
  → Node.js para backends
    "Manter existentes, novos serviços em Java/Go."

  ══════════════════════════════════════════════════════════
  PLATFORMS
  ══════════════════════════════════════════════════════════
  
  ADOPT
  → Kubernetes (EKS)
  → PostgreSQL 16+
  → Redis 7+
  → Apache Kafka (MSK)
  → Terraform
  
  TRIAL
  → DynamoDB (para specific patterns)
  → Apache Flink (stream processing)
  
  ASSESS
  → CockroachDB (distributed SQL)
  → Temporal (workflow engine)
  
  HOLD
  → MongoDB (para dados relacionais — "usar PostgreSQL")
  → RabbitMQ (migrar para Kafka)
  → CloudFormation (migrar para Terraform)

  ══════════════════════════════════════════════════════════
  TECHNIQUES
  ══════════════════════════════════════════════════════════
  
  ADOPT
  → Trunk-Based Development
  → Feature Flags (LaunchDarkly / Flagsmith)
  → ADRs (Architecture Decision Records)
  → OpenTelemetry (observability)
  → GitOps (ArgoCD)
  
  TRIAL
  → Platform Engineering (Internal Developer Platform)
  → Service Mesh (Istio/Linkerd) — 1 cluster piloto
  
  ASSESS
  → eBPF-based networking
  → AI-assisted code review
  
  HOLD
  → Long-lived feature branches (usar trunk-based)
  → Manual deployment (usar GitOps)
```

---

## Technology Lifecycle

```
TECHNOLOGY LIFECYCLE — TIMING DE ADOÇÃO:

  Adoption
  ▲
  │                              ┌─────────────┐
  │                             ╱│  MATURE     │╲
  │                           ╱  │  (Late      │  ╲
  │                         ╱    │  Majority)  │    ╲
  │                       ╱      │             │      ╲
  │                     ╱        └─────────────┘        ╲
  │                   ╱                                    ╲
  │              ┌──╱──┐                              ┌─────╲──┐
  │             │GROWTH│                              │DECLINE │
  │             │(Early│                              │(Laggard│
  │             │Major)│                              │ phase) │
  │          ┌──┤      ├──┐                           └────────┘
  │       ┌──┤  └──────┘  │
  │    ┌──┤  │  Early     │
  │    │  │  │  Adopters  │
  │    │  │  └────────────┘
  │ ┌──┤  │
  │ │  │  │ Innovators
  │ │  └──┘
  ├─┴─────────────────────────────────────────────────────────► Tempo
  │
  │ ←─ ASSESS ─→←─ TRIAL ─→←──── ADOPT ────→←─── HOLD ───→

  REGRA DE OURO: "Choose Boring Technology" (Dan McKinley)

  → Cada organização tem um "innovation budget" limitado
  → Escolha 2-3 slots para tecnologia nova
  → Todo o resto: use boring technology (comprovada)
  → "PostgreSQL é entediante. É exatamente por isso 
     que você deveria usá-lo."

  INNOVATION TOKENS (Dan McKinley):

  ┌─────────────────────────────────────────────────────┐
  │ A organização tem ≈ 3 "innovation tokens"           │
  │                                                      │
  │ Token 1: [Kafka] ← stream platform                  │
  │ Token 2: [Go]    ← nova linguagem                   │
  │ Token 3: [_____] ← disponível                       │
  │                                                      │
  │ REGRA: quer adotar algo novo?                        │
  │ → Precisa devolver um token (migrar algo para boring)│
  │ → OU: justificar por que precisa de 4 tokens         │
  │ → Cada token tem custo: learning curve, tooling,     │
  │   debugging expertise, hiring pool                   │
  └─────────────────────────────────────────────────────┘

  CUSTO ESCONDIDO DE TECNOLOGIA "EXCITING":
  
  │ Custo                    │ Impacto                         │
  │ Curva de aprendizado     │ Meses até produtividade          │
  │ Tooling imaturo          │ Debugging doloroso               │
  │ Documentação escassa     │ Stack Overflow vazio             │
  │ Comunidade pequena       │ Menos libs, menos exemplos       │
  │ Hiring difícil           │ Poucos candidatos com exp        │
  │ Bug fixes lentos         │ Maintainers sobrecarregados      │
  │ Breaking changes         │ Upgrades constantes              │
  │ Falta de experts internos│ Bus factor = 1                   │
```

---

## Princípios de Decisão Tecnológica

```
PRINCÍPIOS PARA DECISÕES TECNOLÓGICAS:

  1. FIT FOR PURPOSE (não "best in class")
     → "A melhor tecnologia" não existe em abstrato
     → Existe "a melhor para ESTE contexto"
     → Context: equipe, escala, prazo, compliance, skills
     
     Exemplo: MongoDB é excelente para document workloads
     Mas na SUA org com DBA experts em PostgreSQL:
     → PostgreSQL JSONB pode ser melhor fit (boring, known)

  2. REVERSIBILITY FIRST
     → Prefira decisões reversíveis (Type 2)
     → Se irreversível (Type 1): invista mais em análise
     → Design para trocar (interfaces, abstractions)
     
     Exemplo: 
     → Abstrair database atrás de repository interface ✅
     → Usar ORM-specific annotations em todo lugar ❌
     → Interface permite trocar DB se necessário

  3. TOTAL COST OF OWNERSHIP (TCO)
     → Não olhe só o custo de aquisição
     → Inclua: learning, operation, migration, suporte
     → TCO = acquisition + operation + migration + opportunity
     → Ver documento de Build vs Buy para framework completo

  4. BIAS FOR STANDARDIZATION
     → 1 linguagem bem usada > 5 linguagens mal usadas
     → Padronizar reduce: cognitive load, tooling, hiring
     → Exceções: documentadas via ADR com justificativa
     
     Polyglot saudável:
     → Java/Kotlin para serviços de negócio
     → Go para infra tooling e alta performance
     → Python para data/ML
     → TypeScript para frontend
     
     Polyglot patológico:
     → 8 linguagens para 12 serviços
     → "Cada equipe escolheu o que queria"
     → Resultado: ninguém consegue debugar o serviço vizinho

  5. STRATEGIC vs DIFFERENTIATING
     → Tecnologias que não diferenciam: COMPRE (commodity)
     → Tecnologias que diferenciam: considere CONSTRUIR
     → Wardley Mapping ajuda a identificar
     
     │ Tipo           │ Ação          │ Exemplo                   │
     │ Commodity      │ Compre/SaaS   │ Email, CI/CD, monitoring  │
     │ Utility        │ Cloud managed │ Database, queue, cache     │
     │ Custom         │ Avalie        │ API Gateway, search       │
     │ Differentiator │ Construa      │ Core business logic       │
```

---

## Strategic Alignment

```
ALINHAMENTO STRATEGY ↔ TECHNOLOGY:

  ┌── Business Strategy ──────────────────────────────────┐
  │                                                        │
  │  "Dobrar revenue em 2 anos via expansão LATAM"        │
  │                                                        │
  └────────────────────┬───────────────────────────────────┘
                       │
  ┌────────────────────▼───────────────────────────────────┐
  │  Business Capabilities Needed                           │
  │                                                         │
  │  • Multi-currency processing                            │
  │  • Multi-language support (PT, ES, EN)                  │
  │  • Compliance (LGPD, data residency)                    │
  │  • 10x throughput growth                                │
  │  • Time-to-market < 2 weeks per country                 │
  └────────────────────┬───────────────────────────────────┘
                       │
  ┌────────────────────▼───────────────────────────────────┐
  │  Technical Strategy & Initiatives                       │
  │                                                         │
  │  Initiative 1: Multi-region architecture                │
  │    → AWS infra em São Paulo + Virgínia                  │
  │    → Data residency por região                          │
  │    → Latency < 100ms intra-região                      │
  │                                                         │
  │  Initiative 2: Platform modernization                   │
  │    → Feature flags para rollout por país                │
  │    → i18n/l10n framework                                │
  │    → Currency abstraction layer                         │
  │                                                         │
  │  Initiative 3: Performance at scale                     │
  │    → Auto-scaling + capacity planning                   │
  │    → DB sharding strategy                               │
  │    → CDN + edge caching                                │
  │                                                         │
  │  Initiative 4: Developer velocity                       │
  │    → CI/CD pipeline < 15 min                           │
  │    → Staging environment per country                    │
  │    → Internal developer platform                        │
  │                                                         │
  └────────────────────┬───────────────────────────────────┘
                       │
  ┌────────────────────▼───────────────────────────────────┐
  │  Tech Roadmap (ver doc 05)                              │
  │                                                         │
  │  Q1 2026: Multi-region infra + feature flags            │
  │  Q2 2026: i18n + currency + Brasil launch               │
  │  Q3 2026: Mexico + Colombia + performance engineering   │
  │  Q4 2026: Argentina + Chile + platform optimization     │
  └─────────────────────────────────────────────────────────┘

  FERRAMENTAS DE ALINHAMENTO:

  1. WARDLEY MAPPING
     → Mapear componentes por: valor ao usuário × evolução
     → Genesis → Custom → Product → Commodity
     → Decisão: build (Genesis/Custom) vs buy (Product/Commodity)

  2. OKRs TÉCNICOS
     → Derivados de OKRs de negócio
     → Objective: "Infraestrutura multi-region operacional"
     → KR1: "Latência p99 < 100ms em todas as regiões"
     → KR2: "Deploy automático em 3 regiões em < 30 min"
     → KR3: "99.99% availability cross-region"

  3. CAPABILITY MAPPING
     → What capabilities does the business need?
     → What technical capabilities support them?
     → Gap analysis: what we have vs what we need
```

---

## Governança Tecnológica

```
GOVERNANÇA — COMO MANTER ESTRATÉGIA NA PRÁTICA:

  ┌─────────────────────────────────────────────────────────────┐
  │ TIER DE GOVERNANÇA (baseado em blast radius + reversibility)│
  │                                                              │
  │  TIER 1 — TEAM DECISION                                     │
  │  Blast radius: 1 serviço, 1 equipe                          │
  │  Reversibilidade: alta                                       │
  │  Quem: Tech Lead + equipe                                   │
  │  Como: PR + justificativa no PR                              │
  │  Exemplo: test library, utility lib, internal refactor       │
  │                                                              │
  │  TIER 2 — DOMAIN DECISION                                   │
  │  Blast radius: múltiplos serviços no domínio                │
  │  Reversibilidade: média                                      │
  │  Quem: Staff Engineer / Domain Architect                     │
  │  Como: Lightweight ADR + review                              │
  │  Exemplo: novo framework no domínio, API contract change     │
  │                                                              │
  │  TIER 3 — ORG-WIDE DECISION                                 │
  │  Blast radius: múltiplas equipes/domínios                   │
  │  Reversibilidade: baixa                                      │
  │  Quem: Architecture Guild / Principal + VP Eng              │
  │  Como: Full ADR + Architecture Review + stakeholders         │
  │  Exemplo: nova linguagem, novo database engine, cloud migr.  │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘

  RITUAIS DE GOVERNANÇA:

  │ Ritual                │ Frequência │ Participantes           │ Output        │
  │ Architecture Review   │ Semanal    │ Staff+, leads           │ Approved/ask  │
  │ Tech Radar Review     │ Trimestral │ Guild + leads           │ Updated radar │
  │ Tech Debt Review      │ Mensal     │ Leads + PM              │ Debt backlog  │
  │ Roadmap Sync          │ Trimestral │ Eng leadership + Product│ Updated roadmap│
  │ Post-mortem Review    │ Mensal     │ SRE + on-call + leads   │ Action items  │
  │ Security Review       │ Per-feature│ Security champion + dev │ Approved/block│
```

---

## Comunicação de Estratégia

```
COMUNICAÇÃO — STAKEHOLDERS DIFERENTES, MENSAGENS DIFERENTES:

  ┌─────────────────────────────────────────────────────────┐
  │ AUDIÊNCIA: C-LEVEL / VP                                  │
  │                                                          │
  │ Linguagem: Business outcomes, risk, cost, timeline       │
  │ Formato: 1-pager executivo, quarterly review deck        │
  │ Conteúdo:                                                │
  │   • Investimento: $X de custo em infra/time              │
  │   • Retorno: Y% melhoria em time-to-market              │
  │   • Risco: se não investirmos, Z impacto                │
  │   • Timeline: Q1-Q2 investimento, Q3-Q4 retorno         │
  │                                                          │
  │ ❌ "Vamos migrar de PostgreSQL 11 para 16"               │
  │ ✅ "Vamos eliminar risco de vulnerabilidades em DB       │
  │     e ganhar 40% em performance de queries"              │
  └─────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────┐
  │ AUDIÊNCIA: ENGINEERING LEADERSHIP (Directors, Managers)   │
  │                                                          │
  │ Linguagem: Capacity, headcount, dependencies, timeline   │
  │ Formato: Strategy doc + roadmap + resource ask           │
  │ Conteúdo:                                                │
  │   • O que muda (e o que NÃO muda)                       │
  │   • Headcount/allocation needed                          │
  │   • Dependencies entre times                             │
  │   • Milestones e check-in points                         │
  │   • Risk mitigation plan                                 │
  └─────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────┐
  │ AUDIÊNCIA: ENGINEERS                                     │
  │                                                          │
  │ Linguagem: Técnica, honesta, detalhada                   │
  │ Formato: Tech Strategy doc + ADRs + Tech Radar          │
  │ Conteúdo:                                                │
  │   • POR QUE estamos mudando (context)                   │
  │   • O QUE muda na prática (their daily work)            │
  │   • COMO vai acontecer (implementation plan)            │
  │   • QUANDO (timeline, phasing)                           │
  │   • Trade-offs (o que estamos abrindo mão)              │
  │   • "E o que eu faço amanhã?" (immediate next steps)    │
  └─────────────────────────────────────────────────────────┘
```

---

## Anti-Patterns de Tech Strategy

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **Strategy by FOMO** | "Todo mundo está usando X" → adota sem contexto | Avaliar fit para SEU contexto (Tech Radar process) |
| **Ivory Tower Architect** | Strategy definida sem input de quem implementa | Include Tech Leads e IC seniors no processo |
| **Strategy sem execução** | Documento bonito, nenhuma ação | Roadmap com owners, milestones, deadlines |
| **Golden Hammer** | "Kubernetes resolve tudo" — tudo vira nail | Fit for purpose: tool right para problema certo |
| **Résumé-Driven Dev** | Escolher tech para aprender, não por fit | Innovation tokens — slots limitados para "novo" |
| **Analysis Paralysis** | 6 meses avaliando 5 frameworks, nenhuma decisão | Timeboxar decisões: 2 semanas para Type 2, 4-8 para Type 1 |
| **Big Rewrite Fantasy** | "Vamos reescrever tudo em [tech nova]" | Strangler Fig — migração incremental |
| **No Radar, No Standards** | Cada equipe escolhe stack diferente | Tech Radar + governance tiers |
| **Strategy desconectada** | Tech strategy não reflete business strategy | Trilha: Business → Capabilities → Tech → Roadmap |
| **Never Update** | Strategy de 2022 em vigor em 2026 | Revisão trimestral mínima |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código, arquitetura e decisões tecnológicas, verifique:

1. **Tecnologia fora do radar** — Se tecnologia não está em ADOPT ou TRIAL no radar, exigir ADR justificando exceção
2. **Sem ADR para decisão Type 1** — Mudança de database engine, linguagem, framework principal DEVE ter ADR
3. **Lock-in sem abstração** — Uso direto de APIs vendor-specific sem camada de abstração em code paths críticos
4. **Polyglot desnecessário** — Nova linguagem/framework sem justificativa clara e innovation token disponível
5. **Build quando deveria Buy** — Reimplementando commodity (auth, email, payments) em vez de usar solução madura
6. **Sem rollback plan** — Migração sem plano de rollback ou canary strategy
7. **Dependency sem avaliação** — Nova dependência sem avaliar: manutenção ativa? licença? security? bus factor?
8. **Tech stack inconsistente no domínio** — Serviços do mesmo domínio com stacks diferentes sem justificativa
9. **Decisão estratégica sem stakeholder alignment** — Mudança cross-team sem review do Architecture Guild
10. **Falta de princípios de design documentados** — Repositório sem ARCHITECTURE.md ou `docs/decisions/`
11. **Vendor lock-in em infrastructure as code** — CloudFormation hard-lock; preferir Terraform/OpenTofu para portabilidade
12. **Sem métricas de success para iniciativa técnica** — Toda iniciativa estratégica precisa de KPIs mensuráveis

---

## Referências

- **Choose Boring Technology** — Dan McKinley — https://mcfunley.com/choose-boring-technology
- **Technology Radar** — ThoughtWorks — https://www.thoughtworks.com/radar
- **Wardley Mapping** — Simon Wardley — https://learnwardleymapping.com/
- **Team Topologies** — Matthew Skelton & Manuel Pais (IT Revolution, 2019)
- **An Elegant Puzzle** — Will Larson (Stripe Press, 2019)
- **Staff Engineer** — Will Larson (2021) — https://staffeng.com/
- **The Staff Engineer's Path** — Tanya Reilly (O'Reilly, 2022)
- **Technology Strategy Patterns** — Eben Hewitt (O'Reilly, 2018)
- **Fundamentals of Software Architecture** — Mark Richards & Neal Ford (O'Reilly, 2020)
- **Software Architecture: The Hard Parts** — Neal Ford et al. (O'Reilly, 2021)
- **Building Evolutionary Architectures** — Neal Ford et al. (O'Reilly, 2017)
- **Type 1 vs Type 2 Decisions** — Jeff Bezos — Amazon Shareholder Letter (2015)
