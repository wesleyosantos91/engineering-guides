# Architecture Decision Records — Templates e Exemplos

> **Objetivo deste documento:** Fornecer **templates prontos e exemplos completos** de ADRs para diferentes tipos de decisões arquiteturais — copie, adapte e use, otimizado como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: templates (mínimo, MADR, lightweight), exemplos realistas para technology choice, integration pattern, data architecture, infra, security e process.

---

## Quick Reference — Cheat Sheet

| Tipo de decisão | Template recomendado | Seções-chave | Exemplo neste doc |
|-----------------|---------------------|-------------|-------------------|
| **Tipo 2 simples** | Template Mínimo (Nygard) | Status, Context, Decision, Consequences | Exemplo 1 |
| **Tipo 2 significativa** | Template Standard | + Decision Drivers, Options, Pros/Cons | Exemplo 2 |
| **Tipo 1 single-team** | Template MADR Completo | + Links, Compliance, Migration | Exemplo 3 |
| **Tipo 1 cross-team** | Template MADR + RFC ref | Todas as seções + RFC reference | Exemplo 4 |

---

## Sumário

- [Architecture Decision Records — Templates e Exemplos](#architecture-decision-records--templates-e-exemplos)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Template 1 — Mínimo (Nygard Original)](#template-1--mínimo-nygard-original)
  - [Template 2 — Standard (Recomendado)](#template-2--standard-recomendado)
  - [Template 3 — MADR Completo](#template-3--madr-completo)
  - [Exemplo 1 — Technology Choice (Tipo 2)](#exemplo-1--technology-choice-tipo-2)
  - [Exemplo 2 — Integration Pattern (Tipo 2 Significativa)](#exemplo-2--integration-pattern-tipo-2-significativa)
  - [Exemplo 3 — Data Architecture (Tipo 1)](#exemplo-3--data-architecture-tipo-1)
  - [Exemplo 4 — Infrastructure / Platform (Tipo 1 Cross-team)](#exemplo-4--infrastructure--platform-tipo-1-cross-team)
  - [Exemplo 5 — Security Decision](#exemplo-5--security-decision)
  - [Exemplo 6 — Process Decision](#exemplo-6--process-decision)
  - [Como Adaptar Templates](#como-adaptar-templates)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Template 1 — Mínimo (Nygard Original)

```markdown
# NNNN. [Título curto descritivo da decisão]

Date: YYYY-MM-DD

## Status

Proposed | Accepted | Deprecated | Superseded by [ADR-NNNN](NNNN-slug.md)

## Context

[Descreva a situação, o problema e as forças em jogo.
O que motiva esta decisão? Quais constraints existem?]

## Decision

[Declare a decisão em voz ativa: "Iremos..." ou "Adotaremos..."]

## Consequences

[Liste as consequências — positivas E negativas.
O que muda? O que fica mais fácil? O que fica mais difícil?]
```

```
QUANDO USAR O TEMPLATE MÍNIMO:

✅ Decisões Tipo 2 (reversíveis) com escopo pequeno
✅ Decisões óbvias que precisam apenas de registro
✅ Quando velocidade é mais importante que detalhe
✅ Times pequenos com contexto compartilhado

❌ Decisões Tipo 1 (irreversíveis)
❌ Decisões cross-team
❌ Quando há alternativas controversas para documentar
```

---

## Template 2 — Standard (Recomendado)

```markdown
# NNNN. [Título: verbo + objeto da decisão]

Date: YYYY-MM-DD
Owner: [nome/papel do author]
Reviewers: [nomes/papéis]
Status: Proposed | Accepted | Deprecated | Superseded by [ADR-NNNN]

## Context and Problem Statement

[Descreva a situação atual com dados quantitativos.
Qual problema precisa ser resolvido? Por que agora?]

## Decision Drivers

* [Driver 1 — ex: latência p99 < 50ms]
* [Driver 2 — ex: custo ≤ $3,000/mês]
* [Driver 3 — ex: experiência do time]
* [Driver N]

## Considered Options

* [Opção 1 — nome descritivo]
* [Opção 2 — nome descritivo]
* [Opção 3 — nome descritivo (se aplicável)]

## Decision Outcome

Chosen option: "[Opção N]", because [justificativa vinculada aos drivers].

### Positive Consequences

* [Consequência positiva 1]
* [Consequência positiva 2]

### Negative Consequences

* [Trade-off aceito 1 — com mitigação se houver]
* [Trade-off aceito 2]

## Pros and Cons of the Options

### [Opção 1]

[Descrição breve]

* ✅ [Pro 1]
* ✅ [Pro 2]
* ❌ [Con 1]
* ❌ [Con 2]

### [Opção 2]

[Descrição breve]

* ✅ [Pro 1]
* ❌ [Con 1]

### [Opção 3]

[Descrição breve]

* ✅ [Pro 1]
* ❌ [Con 1]

## Links

* [Link para RFC, se houver]
* [Link para spike/PoC results]
* [Link para ADRs relacionados]
```

```
QUANDO USAR O TEMPLATE STANDARD:

✅ Maioria das decisões — este é o DEFAULT
✅ Decisões Tipo 2 significativas
✅ Decisões Tipo 1 intra-time
✅ Quando existem alternativas claras para comparar

Template suficientemente completo sem ser pesado.
Balanceia rigor com praticidade.
```

---

## Template 3 — MADR Completo

```markdown
---
adr: "NNNN"
title: "[Título descritivo]"
status: "proposed"
date: "YYYY-MM-DD"
decision-makers: ["@author"]
reviewers: ["@reviewer1", "@reviewer2"]
tags: [tag1, tag2]
domain: "[domain-name]"
tier: "team-level | cross-team | org-wide"
supersedes: ""
related: []
---

# NNNN. [Título: verbo + objeto da decisão]

## Status

Proposed

## Context and Problem Statement

[Situação atual com dados. Problema quantificado. 
Urgência e impacto de não decidir.]

## Decision Drivers

* [Driver 1 — quantificado se possível]
* [Driver 2]
* [Driver N]

## Considered Options

* [Opção 0: Do Nothing / Status Quo]
* [Opção 1]
* [Opção 2]
* [Opção 3 (se aplicável)]

## Decision Outcome

Chosen option: "[Opção N]", because [justificativa vinculada aos drivers].

### Scope

This decision applies to: [escopo explícito]
This decision does NOT cover: [exclusões]

### Positive Consequences

* [Consequência positiva vinculada ao driver]

### Negative Consequences

* [Trade-off aceito com mitigação]

## Pros and Cons of the Options

### Opção 0: Do Nothing / Status Quo

[Manter situação atual]

* ✅ Zero custo de mudança
* ✅ Sem risco de regression
* ❌ [Problema que motivou o ADR persiste]
* ❌ [Consequência de inação]

### Opção 1: [Nome]

[Descrição]

* ✅ [Pro]
* ❌ [Con]

### Opção 2: [Nome]

[Descrição]

* ✅ [Pro]
* ❌ [Con]

## Compliance and Security

[Impactos em compliance, security, GDPR, SOC2, etc.
Omitir se não aplicável.]

## Migration Plan

[Alto nível de como implementar.
Detalhes vão em Design Doc separado.
Omitir para decisões que não requerem migration.]

## Links

* [RFC-NNNN: Título](link) — RFC que originou este ADR
* [ADR-NNNN: Título](link) — Supersedes / Related
* [Spike Results](link) — Evidência coletada
* [Design Doc](link) — Detalhamento de implementação
```

```
QUANDO USAR O TEMPLATE MADR COMPLETO:

✅ Decisões Tipo 1 de alto impacto
✅ Decisões cross-team ou org-wide (Tier 2/3)
✅ Quando compliance/security é relevante
✅ Quando migration é complexa
✅ Quando origina de um RFC

❌ Decisões Tipo 2 simples — use Template Standard
❌ Se o time reclamar que é pesado demais
```

---

## Exemplo 1 — Technology Choice (Tipo 2)

```markdown
# 0015. Usar Redis como cache layer do catalog-service

Date: 2024-01-15
Owner: Maria Silva (Senior Engineer, Catalog Team)
Status: Accepted

## Context and Problem Statement

O catalog-service atende ~2,000 req/s de listagem de produtos.
Atualmente, toda request faz query ao PostgreSQL. 
Latência p99 cresceu de 35ms para 120ms nos últimos 3 meses 
conforme o catálogo cresceu de 50K para 200K produtos.

Target de SLO é p99 < 80ms. Estamos violando SLO desde novembro.

## Decision Drivers

* Latência p99 < 80ms para reads de catálogo
* Custo de infra adicional ≤ $500/mês
* Complexidade operacional baixa (time de 5 devs)
* Cache invalidation não pode ter stale data > 60s

## Considered Options

* Opção 1: Redis (ElastiCache)
* Opção 2: Application-level cache (Caffeine/Guava)
* Opção 3: CDN caching (CloudFront)

## Decision Outcome

Chosen option: "Redis (ElastiCache)", because atende o SLO 
de latência com cache distribuído, permite invalidação 
controlada via TTL + pub/sub, e a equipe já operou Redis 
em outros serviços.

### Positive Consequences

* Latência p99 esperada ~15ms para cache hits (vs 120ms hoje)
* Cache ratio estimado > 90% (catálogo muda ~50 vezes/dia)
* ElastiCache managed — low operational overhead
* Custo estimado: ~$200/mês (cache.r6g.large)

### Negative Consequences

* Mais um componente na stack — mais um ponto de falha
  (mitigação: fallback para DB em caso de Redis down)
* Cache invalidation complexity — precisa invalidar no 
  write path (mitigação: write-through com TTL de 60s)
* Cold start: primeira request após deploy é lenta
  (mitigação: cache warming no startup)

## Pros and Cons of the Options

### Redis (ElastiCache)

Cache distribuído, compartilhado entre instâncias.

* ✅ Cache compartilhado entre N pods
* ✅ Invalidação via TTL + pub/sub
* ✅ Managed (ElastiCache) — backups, failover
* ✅ Time já tem experiência
* ❌ Mais um componente de infra
* ❌ Custo adicional ~$200/mês
* ❌ Network hop adicional (~1-2ms)

### Application-level cache (Caffeine)

Cache in-memory no processo da aplicação.

* ✅ Zero infra adicional
* ✅ Latência ~0ms (in-process)
* ✅ Simples de implementar
* ❌ Cache duplicado em cada pod (N cópias)
* ❌ Invalidação: cada pod precisa invalidar independente
* ❌ Stale data entre pods (eventual consistency)
* ❌ Consome RAM do pod (200K items × ~2KB = ~400MB)

### CDN caching (CloudFront)

Cache na edge, antes da aplicação.

* ✅ Reduz load no serviço
* ✅ Latência mínima para clientes
* ❌ Invalidação complexa (propagation delay 5-10min)
* ❌ Stale data > 60s viola requirement
* ❌ Não resolve latência de requests não-cacheáveis

## Links

* [Spike: Redis cache performance test](link-to-spike-pr)
* Monitored SLO dashboard: [Grafana link]
```

---

## Exemplo 2 — Integration Pattern (Tipo 2 Significativa)

```markdown
# 0023. Adotar event-driven communication entre order e payment services

Date: 2024-02-20
Owner: Carlos Mendes (Tech Lead, Orders)
Reviewers: @ana-payment-lead, @pedro-platform
Status: Accepted

## Context and Problem Statement

Hoje order-service chama payment-service via REST síncrono para 
processar pagamentos. Este acoplamento causa:

* Cascading failures: payment-service down → orders falham (3 P2 incidents em 6 meses)
* Latência: payment processing leva 2-8s, bloqueando thread do order-service
* Retry complexity: retries síncronos causam duplicate payments (2 casos em produção)
* Deploy coupling: mudança em payment API requer deploy coordenado

Volume: ~500 orders/min no pico, 200 orders/min médio.
SLA payment: processamento em ≤ 30s (não precisa ser síncrono para o user).

## Decision Drivers

* Eliminar cascading failures entre order e payment
* Payment processing não precisa ser síncrono (user recebe "order placed")
* Garantir exactly-once processing (idempotência)
* Manter rastreabilidade de ponta a ponta (correlation ID)
* Time já opera AWS — preferência por serviços managed

## Considered Options

* Opção 0: Manter REST síncrono (status quo)
* Opção 1: Amazon SQS com dead-letter queue
* Opção 2: Amazon EventBridge
* Opção 3: Apache Kafka (MSK)

## Decision Outcome

Chosen option: "Amazon SQS com dead-letter queue", because 
desacopla os serviços com garantia de at-least-once delivery, 
DLQ para mensagens problemáticas, custo proporcional ao volume,
e operational burden mínimo (fully managed).

### Scope

* Aplica-se à comunicação order → payment para processamento de pagamento
* NÃO aplica-se a queries de status (payment-service mantém API REST para consulta)
* NÃO define event schema (será ADR separado)

### Positive Consequences

* Order-service não depende de payment-service estar up
* Payment processing assíncrono — user experience melhora (order acknowledged imediatamente)
* SQS managed — zero operational overhead de brokers
* DLQ captura mensagens problemáticas para investigação
* Custo: ~$15/mês para 500 msg/min

### Negative Consequences

* Complexidade de eventual consistency (order "placed" mas payment pode falhar)
  → Mitigação: saga pattern com compensation (cancel order se payment falha após 30s)
* Debugging mais difícil — mensagem no SQS não é tão visível quanto REST call
  → Mitigação: structured logging com correlation ID em todas as mensagens
* Novo pattern para o time — curva de aprendizado
  → Mitigação: pairing com platform team que já usa SQS

## Pros and Cons of the Options

### Opção 0: Manter REST síncrono

* ✅ Simples, time já conhece
* ✅ Debugging: request/response visível
* ❌ Cascading failures persist
* ❌ Latência bloqueante
* ❌ Retry → duplicate payments

### Opção 1: Amazon SQS

* ✅ At-least-once delivery garantido
* ✅ Dead-letter queue para erros
* ✅ Fully managed, ~$15/mês
* ✅ Simples de operar
* ❌ Sem ordering garantido (standard queue)
* ❌ Não é pub/sub (1 consumer por mensagem)

### Opção 2: Amazon EventBridge

* ✅ Event routing com rules
* ✅ Schema discovery e registry
* ✅ Pub/sub nativo (N subscribers)
* ❌ Mais complexo para caso simples point-to-point
* ❌ Custo maior para alto volume ($1/million events)
* ❌ Time não tem experiência

### Opção 3: Apache Kafka (MSK)

* ✅ Ordering garantido por partition
* ✅ Replay de eventos possível
* ✅ Alto throughput
* ❌ Operational overhead significativo (managed ajuda, mas não elimina)
* ❌ Custo mínimo ~$500/mês (overkill para 500 msg/min)
* ❌ Complexidade desproporcional ao caso de uso

## Links

* [P2 Incident: cascading failure 2024-01-05](link)
* [P2 Incident: duplicate payment 2024-02-01](link)
* [SQS vs EventBridge comparison spike](link)
```

---

## Exemplo 3 — Data Architecture (Tipo 1)

```markdown
---
adr: "0042"
title: "Migrar order-service de MySQL 5.7 para PostgreSQL 16"
status: "accepted"
date: "2024-02-10"
decision-makers: ["@joao-senior-dev"]
reviewers: ["@tech-lead-orders", "@dba-team", "@principal-eng"]
tags: [database, postgresql, mysql, migration, orders]
domain: "order-management"
tier: "team-level"
supersedes: "0001"
related: ["0035", "0038"]
---

# 0042. Migrar order-service de MySQL 5.7 para PostgreSQL 16

## Status

Accepted (2024-02-25)
Supersedes: [ADR-0001: Use MySQL for order data](0001-use-mysql-for-order-data.md)

## Context and Problem Statement

O order-service usa MySQL 5.7 em RDS (db.r5.2xlarge, $1,400/mês) 
desde 2022 (ADR-0001). Nos últimos 12 meses, o cenário mudou:

**Problema de performance:**
* p99 read latency: 280ms (era 45ms há 6 meses) — violando SLO de 100ms
* Lock contention em writes > 100 TPS causa timeouts frequentes
* 3 incidentes P2 nos últimos 2 meses por query performance

**Problema de funcionalidade:**
* Novo modelo de customização de pedidos requer campos semi-estruturados
* MySQL 5.7 não tem suporte robusto a JSON (não indexável como JSONB)
* Workaround atual: serializar JSON como TEXT → sem query por fields internas

**Problema de lifecycle:**
* MySQL 5.7 EOL: outubro 2025
* Upgrade para MySQL 8.0 requer trabalho similar a migração

**Contexto do time:**
* 6 desenvolvedores, 3 com experiência em PostgreSQL
* Budget de infra: ≤ $3,000/mês para database
* Timeline: migração completa antes de OUT/2025

## Decision Drivers

* p99 read latency ≤ 50ms
* Suporte nativo a JSONB com indexação
* Custo ≤ $3,000/mês
* Experiência do time (ramp-up mínimo)
* Write throughput ≥ 200 TPS sem contention
* Caminho de migration viável em ≤ 3 meses

## Considered Options

* Opção 0: Manter MySQL 5.7 (upgrade para 8.0)
* Opção 1: PostgreSQL 16 no Amazon RDS
* Opção 2: Amazon DynamoDB
* Opção 3: Amazon Aurora PostgreSQL-Compatible

## Decision Outcome

Chosen option: "PostgreSQL 16 no Amazon RDS", because atende 
todos os decision drivers: JSONB nativo com indexação, custo 
dentro do budget ($1,800/mês estimado), equipe com experiência,
e migration path mais direto (schema relacional → schema relacional).

### Scope

This decision applies to:
* Database do order-service (orders, order_items, order_customizations)

This decision does NOT cover:
* Databases de outros serviços (cada time decide independentemente)
* Strategy de migration detalhada (será Design Doc: DD-0042)
* Schema do novo modelo de customização (será PR específico)
* DynamoDB onde já é usado (catalog-service — ADR-0010 permanece válido)

### Positive Consequences

* p99 read latency estimada ~30ms (spike confirmou com dataset produção)
* JSONB com GIN index permite queries em campos de customização
* Custo estimado $1,800/mês (30% abaixo do budget)
* 3/6 devs já têm experiência — ramp-up estimado: 1 semana para os outros 3
* Ecossistema maduro: PostGIS, pg_stat_statements, logical replication

### Negative Consequences

* **Migration downtime estimado: 2-4h**
  → Mitigação: blue/green deployment com logical replication
  → Fallback: rollback para MySQL read-replica
* **Write throughput ceiling: ~5,000 TPS single-primary**
  → Risco aceito: crescimento atual 20% YoY → ~3-4 anos antes do limite
  → Se atingir: sharding ou DynamoDB para write-heavy paths (novo ADR)
* **Schema migration tooling necessário**
  → Mitigação: adotar Flyway (time já usou em projeto anterior)
* **Operational overhead (backups, upgrades, monitoring)**
  → Mitigação: RDS managed handles maioria; alertas via CloudWatch

## Compliance and Security

* Encryption at rest: AWS KMS (padrão RDS)
* Encryption in transit: SSL/TLS required
* Backup retention: 35 dias (compliance requirement)
* Audit logging: pgAudit extension habilitado
* Network: Private subnet, security group restricts access

## Migration Plan (alto nível)

1. Provisionar PostgreSQL 16 RDS em paralelo ao MySQL
2. Setup AWS DMS para continuous replication MySQL → PostgreSQL
3. Migrar reads gradualmente (feature flag: 10% → 50% → 100%)
4. Cutover de writes em maintenance window (estimado 2h)
5. Manter MySQL read-replica por 2 semanas como rollback
6. Decommission MySQL após confirmação

Detalhes: [Design Doc DD-0042](link)

## Pros and Cons of the Options

### Opção 0: MySQL 8.0 upgrade

* ✅ Menor custo de migração (upgrade in-place)
* ✅ Sem mudança de stack
* ⚠️ JSON support melhorou, mas não é JSONB com GIN index
* ❌ Não resolve lock contention fundamentalmente
* ❌ Upgrade para 8.0 requer testing significativo de compatibilidade
* ❌ Sem benefício de JSONB indexing

### Opção 1: PostgreSQL 16 (RDS)

* ✅ JSONB nativo com GIN index
* ✅ MVCC sem read locks → resolve contention
* ✅ Custo $1,800/mês
* ✅ 3/6 devs experientes
* ✅ Rich extension ecosystem
* ❌ Migration requer downtime (mitigável)
* ❌ Single-primary write ceiling ~5K TPS

### Opção 2: DynamoDB

* ✅ Practically unlimited write throughput
* ✅ Single-digit ms latency
* ✅ Serverless — zero operational overhead
* ❌ 0/6 devs tem experiência (ramp-up 4-6 semanas)
* ❌ Requires fundamental data model redesign (no JOINs)
* ❌ Custo on-demand: estimado $2,200/mês 
* ❌ Migration complexity: rewrite parcial do data layer

### Opção 3: Aurora PostgreSQL-Compatible

* ✅ PostgreSQL-compatible — migration path similar
* ✅ Better scaling (up to 128TB, read replicas)
* ✅ Managed backups, failover automático
* ⚠️ Custo ~$2,800/mês (dentro do budget, mas tight)
* ❌ Vendor lock-in maior que plain RDS PostgreSQL
* ❌ Pricing model complexo (ACUs, I/O)

## Links

* Supersedes: [ADR-0001: Use MySQL for order data](0001-use-mysql-for-order-data.md)
* Related: [ADR-0035: CQRS pattern for order queries](0035-cqrs-pattern.md)
* Related: [ADR-0038: Data migration strategy](0038-data-migration-strategy.md)
* [Spike: PostgreSQL performance test with production dataset](link-to-pr)
* [Design Doc: DD-0042 Migration Plan](link)
* [SLO Dashboard](link-to-grafana)
```

---

## Exemplo 4 — Infrastructure / Platform (Tipo 1 Cross-team)

```markdown
# 0055. Adotar OpenTelemetry como padrão de observabilidade org-wide

Date: 2024-03-01
Owner: Ana Oliveira (Principal Engineer)
Reviewers: @platform-lead, @order-tech-lead, @payment-tech-lead, 
           @catalog-tech-lead, @security-lead
Status: Accepted
Tier: org-wide
RFC: [RFC-012: Observability Stack Modernization](link)

## Status

Accepted (2024-03-20)

## Context and Problem Statement

A organização (15 times, ~80 engineers, ~40 serviços) atualmente 
usa instrumentação heterogênea:

* 12 serviços usam DataDog APM (custo: $45K/ano)
* 8 serviços usam Jaeger para tracing (self-hosted, 2h/semana de manutenção)
* 15 serviços usam StatsD + Graphite para métricas (self-hosted, legacy)
* 5 serviços sem instrumentação estruturada

Problemas:
* Cross-service tracing impossível (trace perde contexto entre stacks)
* Custo combinado: ~$80K/ano + 8h/semana de manutenção
* Onboarding: cada time usa ferramentas diferentes
* 6 P1/P2 incidents em 2023 com MTTR > 2h por falta de tracing cross-service

Esta decisão origina de RFC-012 (consulta de 45 dias com todos os times).

## Decision Drivers

* Distributed tracing end-to-end entre todos os serviços
* Vendor neutrality (não lock-in em APM vendor)
* Custo ≤ $60K/ano (redução de 25% vs atual)
* Suporte a linguagens usadas: Java, Go, Python, Node.js
* Adoção incremental (não precisa migrar todos de uma vez)

## Considered Options

* Opção 0: Manter status quo (heterogêneo)
* Opção 1: OpenTelemetry + Grafana Stack (Tempo, Mimir, Loki)
* Opção 2: Padronizar em DataDog para todos
* Opção 3: AWS X-Ray + CloudWatch

## Decision Outcome

Chosen option: "OpenTelemetry + Grafana Stack", because 
OpenTelemetry é vendor-neutral (CNCF graduated), suporta 
todas as linguagens, permite adoção incremental via auto-instrumentation,
e Grafana Stack reduz custo para ~$35K/ano (Grafana Cloud).

### Scope

* TODOS os serviços da organização devem adotar OpenTelemetry SDK
* Backend: Grafana Cloud (Tempo, Mimir, Loki)
* Timeline: adoção completa em 2 quarters (incremental)
* Platform team provê libraries compartilhadas e documentação

### Positive Consequences

* End-to-end tracing entre todos os 40 serviços
* Vendor neutrality: trocar backend sem mudar instrumentação
* Custo estimado $35K/ano (56% redução vs $80K atual)
* Auto-instrumentation reduz esforço de adoção
* CNCF standard — community, tooling, suporte de longo prazo

### Negative Consequences

* **Migration effort: ~2 sprints por time para full adoption**
  → Mitigação: auto-instrumentation cobre 80%; manual para 20%
  → Platform team provê shared libraries e pairing
* **Grafana Cloud dependency**
  → Mitigação: OpenTelemetry é vendor-neutral; trocar backend = reconfigure collector
* **Learning curve para times usando DataDog**
  → Mitigação: workshop de 2h + documentation + office hours
* **Perda temporária de DataDog features (APM, profiling)**
  → Mitigação: Grafana Pyroscope para profiling; feature parity em 6 meses

## Links

* RFC: [RFC-012: Observability Stack Modernization](link)
* Spike: [OTel auto-instrumentation test](link)
* Cost analysis: [Spreadsheet: DataDog vs Grafana Cloud](link)
```

---

## Exemplo 5 — Security Decision

```markdown
# 0030. Adotar JWT com rotation como padrão de autenticação inter-service

Date: 2024-01-20
Owner: Ricardo Souza (Security Engineer)
Reviewers: @principal-eng, @platform-lead, @compliance
Status: Accepted
Tier: org-wide

## Status

Accepted (2024-02-05)

## Context and Problem Statement

Atualmente, serviços se comunicam usando API keys estáticas 
armazenadas em environment variables:

* 23 API keys ativas, nenhuma com rotation policy
* 4 keys compartilhadas entre múltiplos serviços
* Último rotation: nunca (keys criadas na provisão do serviço)
* SOC2 audit finding: "Static credentials without rotation" (High risk)
* Se uma key vazar, blast radius = todos os serviços que a usam

## Decision Drivers

* SOC2 compliance: credential rotation obrigatória
* Blast radius minimizado (uma key comprometida ≠ acesso total)
* Overhead de implementação razoável para 15 times
* Não requerer mudança de infra significativa (ride on AWS)

## Considered Options

* Opção 1: JWT com AWS Secrets Manager rotation
* Opção 2: mTLS (mutual TLS)
* Opção 3: API keys com manual rotation policy

## Decision Outcome

Chosen option: "JWT com AWS Secrets Manager rotation", because
atende SOC2 requirement, limita blast radius com short-lived tokens (15min TTL),
e implementação é incremental usando shared library.

### Positive Consequences

* SOC2 compliance addressed
* Short-lived tokens (15min) — blast radius limitado
* Centralized em AWS Secrets Manager — audit trail automático
* Shared library abstrai complexidade para os times

### Negative Consequences

* **Complexidade de token refresh** em chamadas longas
  → Mitigação: retry com token refresh automático na shared library
* **Clock skew** pode causar token rejection
  → Mitigação: 30s de tolerance na validação
* **Single point of failure:** Secrets Manager + token issuer
  → Mitigação: cache de public keys, graceful degradation

## Links

* SOC2 Audit Finding: [JIRA-SEC-234](link)
* Shared library design: [Design Doc DD-030](link)
```

---

## Exemplo 6 — Process Decision

```markdown
# 0008. Adotar trunk-based development com feature flags

Date: 2024-01-10
Owner: Felipe Costa (Tech Lead, Platform)
Reviewers: @engineering-managers, @tech-leads
Status: Accepted
Tier: org-wide

## Status

Accepted (2024-01-25)

## Context and Problem Statement

A organização usa Git Flow com branches de longa duração:

* Feature branches duram 2-6 semanas em média
* Merge conflicts frequentes e dolorosos (4h+ para resolver)
* Deploy para produção: 1x por semana (release train semanal)
* 8 hotfixes nos últimos 3 meses (bugs de merge)
* Lead time for changes: ~3 semanas (DORA metric)
* Time gasto em merge/integration: estimado 15% do capacity

DORA metrics atuais:
* Lead time: 3 semanas (target: < 1 semana)
* Deploy frequency: 1x/semana (target: daily)
* Change failure rate: 18% (target: < 15%)

## Decision Drivers

* Reduzir lead time for changes para < 1 semana
* Aumentar deploy frequency para daily
* Eliminar merge conflicts de long-lived branches
* Manter safety (não sacrificar estabilidade por velocidade)

## Considered Options

* Opção 0: Manter Git Flow
* Opção 1: Trunk-based development com feature flags
* Opção 2: GitHub Flow (short-lived branches, no feature flags)

## Decision Outcome

Chosen option: "Trunk-based development com feature flags",
because combina velocidade de integration com safety de rollback
via flags, e LaunchDarkly (já contratado) provê infraestrutura.

### Positive Consequences

* Merge conflicts praticamente eliminados (branches < 1 dia)
* Deploy frequency: pode chegar a multiple per day
* Feature flags permitem dark launches e progressive rollout
* Rollback instantâneo via flag toggle (sem redeploy)

### Negative Consequences

* **Disciplina necessária:** PRs pequenos, build nunca quebra
  → Mitigação: CI blockers, PR size linting (< 400 lines)
* **Feature flag debt:** flags não removidas viram tech debt
  → Mitigação: flag expiry policy (30 dias após 100% rollout)
* **Mudança cultural significativa** para times acostumados com Git Flow
  → Mitigação: adoção por time piloto primeiro, workshops

## Links

* [DORA Metrics Dashboard](link)
* [LaunchDarkly Setup Guide](link)
* [Feature Flag Lifecycle Policy](link)
```

---

## Como Adaptar Templates

```
REGRAS DE ADAPTAÇÃO:

1. COMECE COM O TEMPLATE STANDARD
   Remove seções que não aplicam.
   Adicione seções que faltem.
   NÃO adicione seções vazias "para completar".

2. CALIBRE COM O TIPO DE DECISÃO

   Tipo 2 simples → Template Mínimo
     Status, Context, Decision, Consequences
     30 minutos para escrever

   Tipo 2 significativa → Template Standard
     + Decision Drivers, Options, Pros/Cons
     1-2 horas para escrever

   Tipo 1 → Template MADR Completo
     Todas as seções + Compliance + Migration Plan
     2-4 horas para escrever (+ research time)

3. NUNCA SACRIFIQUE ESTAS SEÇÕES:
   □ Context — SEMPRE. É a seção mais importante.
   □ Decision — SEMPRE. Uma frase clara.
   □ Consequences (negativas) — SEMPRE. Se não tem, está mentindo.

4. SEÇÕES OPCIONAIS (incluir se relevante):
   □ Compliance/Security — se afeta compliance
   □ Migration Plan — se requer mudança de sistema
   □ YAML metadata — se org tem cataloging
   □ "Do Nothing" option — se sistema atual funciona

5. SE O TIME RECLAMA QUE É PESADO:
   → Reduza para Template Mínimo
   → ADR curto e consistent > ADR longo que ninguém escreve
   → 4 seções mínimas é o floor absoluto
```

---

## Diretrizes para Code Review assistido por AI

Ao gerar ou revisar ADRs usando templates, verifique:

1. **Template sem Context** — Toda ADR deve ter contexto com dados quantitativos, não genéricos
2. **Decisão em voz passiva** — Use voz ativa: "Iremos adotar X porque Y", não "Foi decidido que..."
3. **Apenas consequências positivas** — Trade-offs negativos são obrigatórios e a parte mais valiosa
4. **Alternativas strawman** — Cada opção deve ser apresentada no seu melhor (steelman)
5. **"Do nothing" ausente** — Manter o status quo deve ser alternativa quando sistema atual funciona
6. **Scope implícito** — Definir explicitamente o que a decisão NÃO cobre
7. **Template MADR para decisão Tipo 2** — Calibrar peso do template com importância da decisão
8. **Decision drivers ausentes** — Critérios devem ser explícitos e idealmente quantificados
9. **ADR com múltiplas decisões** — Cada ADR deve conter exatamente UMA decisão
10. **Links ausentes** — Referenciar spikes, RFCs, ADRs relacionados, e Design Docs
11. **YAML metadata ausente em org com cataloging** — Se organização faz catalogação, metadata é necessário
12. **Exemplo copiado sem adaptar** — Templates são ponto de partida; adapte ao contexto real

---

## Referências

- **Michael Nygard** — "Documenting Architecture Decisions" (2011) — Template original
- **MADR** — Markdown Any Decision Records — https://adr.github.io/madr/
- **Joel Parker Henderson** — Architecture Decision Record collection — https://github.com/joelparkerhenderson/architecture-decision-record
- **ThoughtWorks Tech Radar** — Lightweight ADRs (Adopt)
- **Design It!** — Michael Keeling (Pragmatic Bookshelf) — Chapter 15: ADRs
- **The Staff Engineer's Path** — Tanya Reilly
- **Accelerate** — Forsgren, Humble, Kim — DORA metrics
- **Jeff Bezos** — Type 1 vs Type 2 decisions (2016 Letter to Shareholders)
- **adr-tools** — https://github.com/npryce/adr-tools
- **Log4brains** — https://github.com/thomvaill/log4brains
