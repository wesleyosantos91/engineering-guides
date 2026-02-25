# Migration Strategies — Padrões e Práticas

> **Objetivo deste documento:** Servir como referência completa sobre **estratégias de migração tecnológica** — padrões comprovados para migrar sistemas, databases, cloud, linguagens e frameworks com risco controlado — otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: Strangler Fig, Big Bang, Parallel Run, Branch by Abstraction, Cloud Migration (6Rs), database migrations, framework/language migrations, rollback strategies.

---

## Quick Reference — Cheat Sheet

| Padrão | Risco | Velocidade | Quando usar |
|--------|-------|-----------|-------------|
| **Strangler Fig** | Baixo | Lento | Monolito → microservices, substituição gradual |
| **Big Bang** | Alto | Rápido | Sistemas pequenos, downtime aceitável |
| **Parallel Run** | Baixo | Médio | Sistemas financeiros, zero-error tolerance |
| **Branch by Abstraction** | Baixo | Médio | Substituir componente interno, sem downtime |
| **Blue-Green** | Médio | Rápido | Infra migration, zero-downtime deploy |
| **Canary** | Baixo | Médio | Validação gradual, rollback rápido |
| **Feature Flag** | Baixo | Variável | Migração user-by-user, A/B comparison |

---

## Sumário

- [Migration Strategies — Padrões e Práticas](#migration-strategies--padrões-e-práticas)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Princípios de Migração](#princípios-de-migração)
  - [Strangler Fig Pattern](#strangler-fig-pattern)
  - [Big Bang Migration](#big-bang-migration)
  - [Parallel Run](#parallel-run)
  - [Branch by Abstraction](#branch-by-abstraction)
  - [Cloud Migration — 6Rs](#cloud-migration--6rs)
  - [Database Migration](#database-migration)
  - [Language e Framework Migration](#language-e-framework-migration)
  - [Rollback Strategies](#rollback-strategies)
  - [Migration Metrics e Tracking](#migration-metrics-e-tracking)
  - [Anti-Patterns de Migração](#anti-patterns-de-migração)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Princípios de Migração

```
PRINCÍPIOS FUNDAMENTAIS:

  1. MIGRE INCREMENTALMENTE
     → Pequenos passos verificáveis > um grande salto
     → Cada passo deve ser revertível
     → "If it hurts, do it more frequently" (Martin Fowler)
  
  2. NUNCA MIGRE SEM ROLLBACK PLAN
     → Antes de cada passo: "como voltamos atrás?"
     → Testar rollback ANTES de executar migração
     → Feature flags são seu melhor amigo
  
  3. DUAL-WRITE / DUAL-READ ANTES DE SWITCH
     → Escreva em ambos (old + new) temporariamente
     → Compare resultados (shadow testing)
     → Só corte old quando new está comprovado
  
  4. OBSERVE ANTES DE CORTAR
     → Métricas: latência, erros, throughput no novo sistema
     → Comparar com baseline do sistema antigo
     → Mínimo 2 semanas em produção antes de decommission
  
  5. ISOLE O BLAST RADIUS
     → Migre 1 feature/serviço/tabela por vez
     → Não migre tudo junto
     → Se falhar, falha é contida

  CUSTO DE NÃO MIGRAR:
  
  ┌─────────────────────────────────────────────────────────────┐
  │  Custo de não migrar cresce exponencialmente com o tempo:   │
  │                                                              │
  │  Custo │                                           ╱        │
  │        │                                         ╱          │
  │        │                                       ╱            │
  │        │                                    ╱               │
  │        │                                 ╱                  │
  │        │                             ╱                      │
  │        │                         ╱ ← Custo de manter legacy │
  │        │                     ╱                              │
  │        │                ╱                                   │
  │        │           ╱                                        │
  │        │ ──────╱──────── ← Custo de migrar (relativamente    │
  │        │ ╱                   constante ou crescente lento)   │
  │        └──────────────────────────────────────────► Tempo    │
  │                                                              │
  │  LESSON: "A melhor hora para migrar era há 2 anos.          │
  │           A segunda melhor hora é agora."                    │
  └─────────────────────────────────────────────────────────────┘
```

---

## Strangler Fig Pattern

```
STRANGLER FIG — MIGRAÇÃO GRADUAL:

  Nomeado pela figueira-estranguladora: planta que cresce ao redor
  de uma árvore existente, eventualmente substituindo-a.

  COMO FUNCIONA:

  ANTES (100% monolito):
  
  Client ──→ [MONOLITO ████████████████████]
              Feature A, B, C, D, E, F

  STEP 1 — Interceptar (Facade/Proxy):
  
  Client ──→ [PROXY/GATEWAY] ──→ [MONOLITO ████████████████████]
              Route all          Feature A, B, C, D, E, F

  STEP 2 — Extrair Feature A:
  
  Client ──→ [PROXY/GATEWAY] ──→ [New: Feature A] ✅
              │                   
              └──→ [MONOLITO ████████████████]
                   Feature B, C, D, E, F

  STEP 3 — Extrair Features B, C:
  
  Client ──→ [PROXY/GATEWAY] ──→ [New: Feature A] ✅
              │                ──→ [New: Feature B] ✅
              │                ──→ [New: Feature C] ✅
              └──→ [MONOLITO ████████]
                   Feature D, E, F

  STEP N — Monolito deprecated:
  
  Client ──→ [PROXY/GATEWAY] ──→ [Feature A] ✅
                               ──→ [Feature B] ✅
                               ──→ [Feature C] ✅
                               ──→ [Feature D] ✅
                               ──→ [Feature E] ✅
                               ──→ [Feature F] ✅
  
              [MONOLITO] ← decommissioned 🗑️

  IMPLEMENTAÇÃO COM API GATEWAY:

  // Exemplo com Kong/NGINX routing
  // Step 1: Tudo vai para monolito
  routes:
    - path: /api/v1/*
      upstream: monolith:8080

  // Step 2: Feature A vai para novo serviço
  routes:
    - path: /api/v1/orders/*           # Feature A
      upstream: orders-service:8080     # Novo serviço
    - path: /api/v1/*                   # Tudo mais
      upstream: monolith:8080           # Monolito

  // Step 3: Features B, C também
  routes:
    - path: /api/v1/orders/*
      upstream: orders-service:8080
    - path: /api/v1/payments/*          # Feature B
      upstream: payments-service:8080
    - path: /api/v1/inventory/*         # Feature C
      upstream: inventory-service:8080
    - path: /api/v1/*
      upstream: monolith:8080

  QUANDO USAR STRANGLER FIG:
  ✅ Monolito grande com múltiplos domínios
  ✅ Não pode parar produção
  ✅ Equipes querem otimizar gradualmente
  ✅ Budget permite timeline longa (meses/anos)
  ✅ APIs existentes podem ser preservadas no gateway
  
  QUANDO NÃO USAR:
  ❌ Sistema muito pequeno (refactor direto é mais simples)
  ❌ Acoplamento interno tão alto que extração é impossível
  ❌ Sem API clara de entrada (batch processing, por exemplo)

  RISCOS E MITIGAÇÕES:
  │ Risco                          │ Mitigação                    │
  │ Dados compartilhados           │ Database per service gradual │
  │ Transações distribuídas        │ Saga pattern durante transição│
  │ Migração incompleta ("stuck")  │ Set deadlines per extraction │
  │ Performance do proxy            │ Monitor latency overhead     │
  │ Feature parity temporária      │ Shadow testing + canary      │
```

---

## Big Bang Migration

```
BIG BANG — SUBSTITUIÇÃO COMPLETA DE UMA VEZ:

  ┌─────────────────────────────────────────────────────────┐
  │                                                          │
  │  Antes:  [Sistema Antigo] ──── operando ────────── X     │
  │                                                    │     │
  │  Switch: ─────────────── CUTOVER WINDOW ──────────┤     │
  │                          (downtime)                │     │
  │  Depois: ─────────────────────────── [Sistema Novo]      │
  │                                                          │
  └─────────────────────────────────────────────────────────┘

  QUANDO FIZER SENTIDO:
  → Sistema pequeno (< 5 serviços, < 50K linhas)
  → Downtime aceitável (ex: sistema interno, weekend window)
  → Sistema antigo impossível de strangler (sem API, batch-only)
  → Prazo firme (compliance deadline, EOL vendor)
  → Novo sistema já testado exaustivamente em staging

  CHECKLIST PARA BIG BANG SEGURO:

  PRÉ-MIGRATION:
  □ Data migration testada 3+ vezes em staging
  □ Rollback plan testado E praticado
  □ Communication plan para stakeholders
  □ Cutover window definida e comunicada
  □ Feature parity validada (acceptance testing)
  □ Performance benchmarks passam no novo sistema
  □ Runbook detalhado step-by-step escrito
  □ All-hands war room com responsáveis definidos
  
  DURANTE CUTOVER:
  □ Stop writes no sistema antigo
  □ Final data sync / migration
  □ Validation queries: contagens, checksums
  □ DNS / routing switch
  □ Smoke tests automatizados
  □ Sample manual testing
  □ Monitoring dashboards abertos
  
  PÓS-MIGRATION (primeiras 48h):
  □ Monitoring intensivo (error rate, latency)
  □ User reports channel aberto
  □ Rollback readiness mantido por 1 semana
  □ Data integrity checks rodando
  □ Performance comparison com baseline

  RISCOS:
  │ Risco             │ Probabilidade │ Impacto │ Mitigação          │
  │ Data corruption    │ Média         │ Alto    │ Checksums, backups │
  │ Feature gap        │ Alta          │ Médio   │ Acceptance tests   │
  │ Performance worse  │ Média         │ Alto    │ Load test antes    │
  │ Rollback falha     │ Baixa         │ Crítico │ Praticar rollback  │
  │ Extended downtime  │ Média         │ Alto    │ Runbook + timeboxes│
```

---

## Parallel Run

```
PARALLEL RUN — EXECUTAR AMBOS SISTEMAS SIMULTANEAMENTE:

  ┌─────────────────────────────────────────────────────────────┐
  │                                                              │
  │  Client ──→ [ROUTER / PROXY]                                │
  │              │              │                                │
  │              ▼              ▼                                │
  │        [OLD SYSTEM]   [NEW SYSTEM]                          │
  │        (primary)      (shadow)                               │
  │              │              │                                │
  │              ▼              ▼                                │
  │        [Response]     [Response]                             │
  │          USADO         COMPARADO                             │
  │              │              │                                │
  │              └──────┬───────┘                                │
  │                     ▼                                        │
  │              [COMPARATOR]                                    │
  │              Compare responses                               │
  │              Log differences                                 │
  │              Alert on divergence                              │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘

  FASES DO PARALLEL RUN:

  PHASE 1: SHADOW MODE (weeks 1-4)
  → Old system: primary (responde ao client)
  → New system: shadow (processa, resultado descartado)
  → Comparator: loga diferenças, não alerta
  → Goal: encontrar bugs no novo sistema

  PHASE 2: COMPARISON MODE (weeks 5-8)
  → Ambos processam, old ainda é primary
  → Comparator: alerta em divergências significativas
  → Equipe investiga CADA divergência
  → Goal: atingir 99.9%+ de paridade

  PHASE 3: CANARY MODE (weeks 9-12)
  → New system: primary para X% dos requests
  → Old system: primary para (100-X)% dos requests
  → Ramp up gradual: 1% → 5% → 25% → 50% → 100%
  → Rollback instant se error rate sobe

  PHASE 4: CUTOVER (week 13+)
  → New system: 100% primary
  → Old system: mantido read-only por 2-4 semanas
  → Monitoring comparativo
  → Decommission old system

  IMPLEMENTAÇÃO CONCEITUAL:

  // Pseudocode para parallel run
  class ParallelRunProxy:
      def handle_request(request):
          // Fork request para ambos
          old_response = old_system.process(request)      // primary
          new_response = new_system.process_async(request) // shadow
          
          // Log comparison
          comparator.compare(
              request, old_response, new_response,
              dimensions=["status", "body", "latency"]
          )
          
          // Retornar resultado do primary
          return old_response

  // Fase 3: Canary
  class CanaryRouter:
      def handle_request(request):
          if feature_flag.enabled("new-system", user=request.user):
              return new_system.process(request)
          else:
              return old_system.process(request)

  QUANDO USAR PARALLEL RUN:
  ✅ Sistemas financeiros (zero tolerance para erros de cálculo)
  ✅ Healthcare / compliance-heavy (auditoria de paridade)
  ✅ Sistemas de decisão (scoring, pricing, risk)
  ✅ Quando precisam de PROVA de que novo sistema é equivalente
  
  LIMITAÇÕES:
  ❌ Dobra custo de infra durante parallel run
  ❌ Write operations precisam de cuidado especial (idempotência)
  ❌ Difícil com side effects (email sending, payments)
  → Solução: shadow mode processa mas não executa side effects
```

---

## Branch by Abstraction

```
BRANCH BY ABSTRACTION — SUBSTITUIR COMPONENTE INTERNO:

  Para trocar implementação interna SEM feature branch longa.

  STEP 1: IDENTIFY (componente a substituir)
  
  [Service A] ──→ [Old Implementation] ──→ [Database]
  [Service B] ──→ [Old Implementation] ──→ [Database]

  STEP 2: CREATE ABSTRACTION (interface/adapter)
  
  [Service A] ──→ ┌─────────────┐ ──→ [Old Impl] ──→ [Database]
  [Service B] ──→ │ ABSTRACTION │ ──→ [Old Impl] ──→ [Database]
                   │ (interface) │
                   └─────────────┘

  STEP 3: BUILD NEW IMPLEMENTATION (behind abstraction)
  
  [Service A] ──→ ┌─────────────┐ ──→ [Old Impl] ──→ [Old DB]
  [Service B] ──→ │ ABSTRACTION │
                   │ (interface) │ ──→ [New Impl] ──→ [New DB]
                   └─────────────┘     (em construção)

  STEP 4: SWITCH (via config/feature flag)
  
  [Service A] ──→ ┌─────────────┐     [Old Impl]  (desativado)
  [Service B] ──→ │ ABSTRACTION │
                   │ (flag=new)  │ ──→ [New Impl] ──→ [New DB]
                   └─────────────┘

  STEP 5: CLEANUP (remover old implementation)
  
  [Service A] ──→ [New Impl] ──→ [New DB]
  [Service B] ──→ [New Impl] ──→ [New DB]

  EXEMPLO EM CÓDIGO:

  // Step 2: Criar abstração
  interface NotificationSender {
      void send(Notification notification);
      NotificationStatus getStatus(String id);
  }

  // Old implementation
  class EmailJsNotificationSender implements NotificationSender {
      // implementação atual com EmailJS
  }

  // Step 3: Nova implementação
  class SendGridNotificationSender implements NotificationSender {
      // nova implementação com SendGrid
  }

  // Step 4: Switch via feature flag
  @Configuration
  class NotificationConfig {
      @Bean
      NotificationSender notificationSender(FeatureFlags flags) {
          if (flags.isEnabled("use-sendgrid")) {
              return new SendGridNotificationSender();
          }
          return new EmailJsNotificationSender();
      }
  }

  QUANDO USAR:
  ✅ Substituir library/SDK (ex: trocar ORM, HTTP client)
  ✅ Substituir integração (ex: trocar payment provider)
  ✅ Refatorar componente interno (ex: trocar engine de busca)
  ✅ Trabalho é INTERNO ao serviço (não precisa de proxy externo)
  ✅ Quer manter trunk-based development (sem long-lived branches)
```

---

## Cloud Migration — 6Rs

```
6Rs da CLOUD MIGRATION (AWS):

  ┌─────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
  │  │          │  │          │  │          │                      │
  │  │ REHOST   │  │REPLATFORM│  │REFACTOR  │                      │
  │  │          │  │          │  │          │                      │
  │  │ Lift &   │  │ Lift &   │  │ Re-      │                      │
  │  │ Shift    │  │ Optimize │  │ architect│                      │
  │  │          │  │          │  │          │                      │
  │  │ Esforço: │  │ Esforço: │  │ Esforço: │                      │
  │  │   Baixo  │  │   Médio  │  │   Alto   │                      │
  │  │          │  │          │  │          │                      │
  │  │ Valor:   │  │ Valor:   │  │ Valor:   │                      │
  │  │   Baixo  │  │   Médio  │  │   Alto   │                      │
  │  │          │  │          │  │          │                      │
  │  └──────────┘  └──────────┘  └──────────┘                      │
  │                                                                  │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
  │  │          │  │          │  │          │                      │
  │  │REPURCHASE│  │ RETIRE   │  │ RETAIN   │                      │
  │  │          │  │          │  │          │                      │
  │  │ Replace  │  │ Shutdown │  │ Keep     │                      │
  │  │ with SaaS│  │ / Decom  │  │ as-is    │                      │
  │  │          │  │          │  │          │                      │
  │  │ Esforço: │  │ Esforço: │  │ Esforço: │                      │
  │  │   Médio  │  │   Baixo  │  │   Zero   │                      │
  │  │          │  │          │  │          │                      │
  │  │ Valor:   │  │ Valor:   │  │ Valor:   │                      │
  │  │   Alto   │  │   Saving │  │   Zero   │                      │
  │  │          │  │          │  │          │                      │
  │  └──────────┘  └──────────┘  └──────────┘                      │
  │                                                                  │
  └─────────────────────────────────────────────────────────────────┘

  DETALHAMENTO:

  1. REHOST (Lift & Shift)
     → Mover para cloud SEM mudanças
     → VM → EC2, DB → RDS (same engine)
     → Rápido, low risk, low benefit
     → Usado: primeira onda de migração, quick wins
     → Exemplo: mover VMs de datacenter para EC2

  2. REPLATFORM (Lift & Optimize)
     → Mover COM otimizações pontuais
     → Não reescreve, mas aproveita cloud services
     → Exemplo: self-managed MySQL → RDS MySQL
     → Exemplo: app on VM → containerized on ECS
     → Moderado: mais benefício que rehost

  3. REFACTOR / RE-ARCHITECT
     → Reescrever para aproveitar cloud-native
     → Monolito → microservices
     → Serverless, managed services, auto-scaling
     → Alto esforço, alto valor de longo prazo
     → Exemplo: reescrever batch job → Step Functions + Lambda

  4. REPURCHASE
     → Substituir por SaaS
     → Self-hosted CRM → Salesforce
     → Self-hosted email → SES/SendGrid
     → Elimina manutenção operacional

  5. RETIRE
     → Desligar (não migrar)
     → Sistema sem uso, duplicado, ou obsoleto
     → "O melhor código é o que você não precisa migrar"
     → Surpreendentemente: 10-30% dos apps em portfolio

  6. RETAIN
     → Manter onde está (on-prem) por agora
     → Razões: compliance, dependency, risk too high
     → Revisitar em próximo ciclo de migração

  ABORDAGEM TÍPICA DE MIGRAÇÃO CLOUD:

  ┌─────────────────────────────────────────────────────────────┐
  │                                                              │
  │  WAVE 1 (Quick Wins — Mês 1-3)                              │
  │  → RETIRE: identificar e desligar 10-30% dos apps           │
  │  → REHOST: migrar apps mais simples (low coupling)          │
  │  → REPURCHASE: substituir por SaaS onde fizer sentido       │
  │                                                              │
  │  WAVE 2 (Core Migration — Mês 3-9)                          │
  │  → REPLATFORM: otimizar apps sendo migrados                 │
  │  → REHOST restante que não precisa de refactor              │
  │                                                              │
  │  WAVE 3 (Modernization — Mês 9-18)                          │
  │  → REFACTOR: apps core/strategic → cloud-native             │
  │  → Microservices, serverless onde fizer sentido              │
  │                                                              │
  │  WAVE 4 (Optimization — Ongoing)                             │
  │  → Cost optimization (reserved instances, spot)              │
  │  → Performance tuning                                        │
  │  → RETAIN apps revisited                                     │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘
```

---

## Database Migration

```
DATABASE MIGRATION — TIPOS E ESTRATÉGIAS:

  ═══════════════════════════════════════════════════════════════
  TIPO 1: SCHEMA MIGRATION (mesmo engine)
  PostgreSQL 11 → PostgreSQL 16
  ═══════════════════════════════════════════════════════════════

  ESTRATÉGIA: Blue-Green com replicação

  [App] ──→ [PG 11 — Primary] ──replication──→ [PG 16 — Replica]
            (writes + reads)                    (promote quando pronto)

  Steps:
  1. Setup PG 16 como replica de PG 11
  2. Validar data sync (checksums por tabela)
  3. Testar app contra PG 16 (staging)
  4. Cutover: promote PG 16, redirect app
  5. Manter PG 11 read-only por 1 semana

  ═══════════════════════════════════════════════════════════════
  TIPO 2: ENGINE MIGRATION
  MongoDB → PostgreSQL, MySQL → PostgreSQL
  ═══════════════════════════════════════════════════════════════

  ESTRATÉGIA: Dual-Write + Gradual Cutover

  PHASE 1 — Dual Write:
  [App] ──write──→ [MongoDB]     ← primary
        └─write──→ [PostgreSQL]  ← shadow (via CDC ou app-level)

  PHASE 2 — Backfill:
  → Migrar dados históricos do MongoDB → PostgreSQL
  → Validation: count, checksums, sample comparison

  PHASE 3 — Dual Read:
  [App] ──read──→ [PostgreSQL]  ← primary reads
        └─read──→ [MongoDB]     ← comparação (shadow)

  PHASE 4 — Cutover:
  [App] ──→ [PostgreSQL]   ← primary (read + write)
  [MongoDB] ← decommission after 2-4 weeks

  FERRAMENTAS ÚTEIS:
  → CDC (Change Data Capture): Debezium, AWS DMS
  → Schema conversion: AWS SCT, pgLoader
  → Validation: data-diff, custom checksums

  ═══════════════════════════════════════════════════════════════
  TIPO 3: SCHEMA EVOLUTION (Expand/Contract Pattern)
  Mudar schema sem downtime
  ═══════════════════════════════════════════════════════════════

  ANTI-PATTERN: ALTER TABLE + deploy ao mesmo tempo
  → Risco: app nova espera coluna X, app velha não conhece X
  → Risco: migration lock durante ALTER TABLE em tabela grande

  CORRETO: EXPAND → MIGRATE → CONTRACT

  Step 1 EXPAND: adicionar nova coluna (nullable, com default)
  → ALTER TABLE users ADD COLUMN email_verified BOOLEAN DEFAULT false;
  → Deploy code que ESCREVE em nova coluna (mas não depende dela)
  → Zero downtime: app velha ignora coluna, app nova popula

  Step 2 MIGRATE: backfill dados existentes
  → UPDATE users SET email_verified = true WHERE verified_at IS NOT NULL;
  → Executar em batches (não lock table inteira)
  
  Step 3 CONTRACT: remover coluna/tabela antiga
  → Só depois de verificar que nova coluna está sendo usada
  → ALTER TABLE users DROP COLUMN old_verified_flag;
  → Deploy remove código que lia a coluna antiga

  TIMELINE:
  │ Step     │ Deploy         │ DB Change               │ Duration │
  │ Expand   │ v2.1 (reads+)│ ADD COLUMN               │ Day 1    │
  │ Migrate  │ v2.1          │ Backfill data            │ Day 2-3  │
  │ Contract │ v2.2 (clean) │ DROP old COLUMN          │ Day 7+   │
```

---

## Language e Framework Migration

```
LANGUAGE / FRAMEWORK MIGRATION:

  ═══════════════════════════════════════════════════════════════
  CENÁRIO 1: Migração de framework (mesmo language)
  Spring Boot 2.x → Spring Boot 3.x
  ═══════════════════════════════════════════════════════════════

  ABORDAGEM: Service by Service + Compatibility Layer

  1. ASSESS: inventário de breaking changes
     → javax → jakarta namespace
     → Deprecated APIs removidas
     → Java 17+ requirement
     → Security config changes
  
  2. PREPARE: criar compatibility checklist
     → Dependency matrix: quais libs suportam SB3?
     → Custom auto-configurations a adaptar
     → Test coverage suficiente para validar migração?
  
  3. PILOT: migrar 1 serviço de baixo risco
     → Serviço com poucos dependents
     → Boa test coverage
     → Equipe motivada/experiente
     → Documentar todos os issues encontrados
  
  4. SCALE: migrar restante com playbook do piloto
     → Template/playbook gerado no piloto
     → Sprint dedicado por serviço
     → CI pipeline valida ambas versões durante transição

  5. CLEANUP: remover compatibilidade com versão antiga
     → Remover branches de compatibilidade
     → Atualizar templates de serviço
     → Atualizar docs e runbooks

  ═══════════════════════════════════════════════════════════════
  CENÁRIO 2: Migração de linguagem
  Python → Go (para serviços de alta performance)
  ═══════════════════════════════════════════════════════════════

  ABORDAGEM: Strangler Fig + API-first

  PRINCÍPIO: NÃO reescrever tudo de uma vez.
  
  1. DEFINIR ESCOPO: quais serviços migram?
     → Critérios: performance-sensitive, CPU-bound, 
       alta concorrência, long-running
     → NÃO migrar: scripts, data pipelines, ML code
  
  2. API CONTRACT FIRST
     → Manter mesma API (REST/gRPC contracts)
     → Consumers não devem perceber mudança
     → Contract tests garantem paridade
  
  3. MIGRAR 1 SERVIÇO POR VEZ (Strangler)
     → Reescrever serviço em Go
     → Deploy lado a lado
     → Shadow traffic para validar
     → Cutover via gateway routing
  
  4. SKILL BUILDING
     → Training para equipe simultaneamente
     → Pair programming Go experts + Python devs
     → Code reviews intensivos nos primeiros serviços
     → Definir Go style guide da org

  DECISION MATRIX — TO MIGRATE OR NOT:

  │ Critério              │ Migra ✅         │ Não migra ❌      │
  │ Latency-critical      │ p99 > SLO        │ Dentro do SLO     │
  │ CPU-bound             │ > 70% CPU usage   │ < 30% CPU usage   │
  │ Concurrency-heavy     │ 1000+ goroutines  │ Low concurrency   │
  │ Equipe capacity       │ Engenheiros       │ Equipe saturada   │
  │                       │ disponíveis       │                   │
  │ Business priority     │ Core path         │ Internal tool     │
  │ Test coverage         │ > 80%             │ < 30%             │
  │ API well-defined      │ OpenAPI spec      │ Spaghetti routes  │
```

---

## Rollback Strategies

```
ROLLBACK — ESTRATÉGIAS POR TIPO DE MIGRAÇÃO:

  ┌──────────────────────────────────────────────────────────────┐
  │ TIPO DE MIGRAÇÃO          │ ROLLBACK STRATEGY               │
  ├───────────────────────────┼─────────────────────────────────┤
  │ Application deploy        │ Blue-Green: switch DNS/LB back  │
  │                           │ Canary: ramp down new version   │
  │                           │ Feature flag: disable flag      │
  ├───────────────────────────┼─────────────────────────────────┤
  │ Database schema           │ Expand/Contract: rollback é     │
  │ (additive change)         │ natural (old code ignora nova   │
  │                           │ coluna)                          │
  ├───────────────────────────┼─────────────────────────────────┤
  │ Database schema           │ ⚠️ HARD — precisa backup +      │
  │ (destructive change)      │ restore, ou manter dual schema  │
  │                           │ por período de bake-in           │
  ├───────────────────────────┼─────────────────────────────────┤
  │ Database engine           │ Manter old DB operacional       │
  │ migration                 │ por 2-4 semanas pós-cutover     │
  │                           │ Dual-write permite rollback     │
  ├───────────────────────────┼─────────────────────────────────┤
  │ Cloud migration           │ Hybrid: manter on-prem por      │
  │                           │ período de transição             │
  │                           │ DNS failback para on-prem       │
  ├───────────────────────────┼─────────────────────────────────┤
  │ Microservice extraction   │ Gateway routing: redirect back  │
  │ (Strangler Fig)           │ to monolith endpoint            │
  └───────────────────────────┴─────────────────────────────────┘

  ROLLBACK TESTING:

  → ANTES de executar migração, testar rollback:
  
  1. Execute migração em staging
  2. Valide que funciona
  3. Execute ROLLBACK em staging
  4. Valide que rollback funciona
  5. Execute migração novamente (idempotente?)
  6. Só então: executar em produção

  CRITÉRIOS DE ROLLBACK AUTOMÁTICO:

  // Exemplo de automated rollback triggers
  rollback_triggers:
    - metric: error_rate
      threshold: "> 1% (baseline + 5x)"
      window: "5 minutes"
      action: "auto-rollback"
    
    - metric: latency_p99
      threshold: "> 500ms (baseline 100ms)"
      window: "10 minutes"
      action: "alert + manual decision"
    
    - metric: success_rate
      threshold: "< 99%"
      window: "5 minutes"
      action: "auto-rollback"

  PONTO DE NÃO-RETORNO:
  → Identificar o ponto após o qual rollback é impossível/muito custoso
  → Antes desse ponto: investir em validação
  → No ponto: decisão explícita de GO / NO-GO
  → Exemplos de ponto de não-retorno:
    - Data migration com transformação irreversível
    - Publicação de nova API pública (clientes migrados)
    - Descommission de serviço com dados excluídos
```

---

## Migration Metrics e Tracking

```
MÉTRICAS DE ACOMPANHAMENTO DE MIGRAÇÃO:

  ═══════════════════════════════════════════════════════════
  PROGRESS METRICS (quanto já migramos)
  ═══════════════════════════════════════════════════════════

  │ Métrica                    │ Fórmula                      │
  │ % Services migrated        │ migrated / total × 100       │
  │ % Traffic on new system    │ new_requests / total × 100   │
  │ % Data migrated            │ migrated_rows / total × 100  │
  │ % Users on new system      │ users_new / users_total × 100│
  │ % Endpoints cutover        │ new_endpoints / total × 100  │

  ═══════════════════════════════════════════════════════════
  QUALITY METRICS (migração está saudável?)
  ═══════════════════════════════════════════════════════════

  │ Métrica                    │ Target                        │
  │ Error rate delta           │ New system ≤ old system       │ 
  │ Latency delta              │ New p99 ≤ 1.2x old p99       │
  │ Data parity                │ 100% (zero drift)             │
  │ Rollbacks executed         │ 0 (ideal) / track count       │
  │ Incidents caused           │ 0 (ideal) / track severity    │
  │ Feature parity gaps        │ 0 (before cutover)            │

  ═══════════════════════════════════════════════════════════
  EFFICIENCY METRICS
  ═══════════════════════════════════════════════════════════

  │ Métrica                    │ Tracking                      │
  │ Eng-weeks per service      │ Actual vs estimated           │
  │ Calendar time per service  │ Lead time of migration        │
  │ Cost (infra) during overlap│ Double-running cost           │
  │ Downtime attributed        │ Minutes of downtime from migr │

  DASHBOARD DE MIGRAÇÃO:

  ┌──────────────────────────────────────────────────────────┐
  │ MIGRATION: Monolith → Microservices                      │
  │ Start: Jan 2026 │ Target: Dec 2026 │ Status: ON TRACK   │
  │                                                          │
  │ Progress:  ████████████░░░░░░░░  62% services            │
  │ Traffic:   ██████████████░░░░░░  71% on new system       │
  │ Data:      ████████████████████  100% migrated           │
  │                                                          │
  │ Quality:                                                  │
  │   Error rate: 0.02% (baseline: 0.03%) ✅                 │
  │   Latency p99: 85ms (baseline: 92ms)  ✅                 │
  │   Incidents: 1 (P3, resolved)          ✅                 │
  │   Rollbacks: 0                         ✅                 │
  │                                                          │
  │ Next milestones:                                          │
  │   → Orders service cutover: Feb 15                       │
  │   → Payments service start: Mar 1                        │
  │   → Old monolith decommission: Dec 31                    │
  └──────────────────────────────────────────────────────────┘
```

---

## Anti-Patterns de Migração

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **Big Bang everything** | Migrar tudo ao mesmo tempo → risco catastrófico | Strangler Fig: migrar incrementalmente |
| **No rollback plan** | "Vai dar certo" → quando não dá, caos | Testar rollback ANTES de migrar |
| **Migração eterna** | Começou há 2 anos, nunca termina | Set deadlines, track progress, kill zombies |
| **Second system syndrome** | Novo sistema tenta resolver TUDO → overengineered | Feature parity first, enhancements DEPOIS |
| **Data migration afterthought** | Foca em app, esquece dados → corruption | Data migration é o CENTRO da migração |
| **No shadow testing** | Deploy new system sem comparar com old | Parallel Run com comparação automatizada |
| **Forget to decommission** | Old system rodando (e custando) indefinidamente | Deadline de decommission no plano |
| **Feature creep during migration** | "Já que estamos migrando, vamos adicionar..." | Migration ≠ feature development. Separar. |
| **Insufficient testing** | "Funciona em staging" → quebra em prod | Load test, shadow test, canary, gradual rollout |
| **No communication plan** | Equipes surpreendidas por mudanças | Communicate early, often, with context |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código em contexto de migração, verifique:

1. **Ausência de feature flag** — Migração sem feature flag para rollback rápido
2. **Backward compatibility quebrada** — API pública com breaking change sem versioning
3. **Schema migration destrutiva** — DROP COLUMN ou ALTER TYPE sem expand/contract pattern
4. **Dual-write incompleto** — Escrevendo apenas em novo sistema sem manter escrita no antigo durante transição
5. **Missing validation** — Sem checksums ou data validation pós-migração
6. **Hardcoded routing** — Routing para novo sistema hardcoded em vez de via feature flag/config
7. **Missing rollback path** — Código de migração sem mecanismo de reverter
8. **Sem métricas de comparação** — Novo sistema sem logging/metrics para comparar com baseline do antigo
9. **Migration acoplada a feature** — Migração e nova funcionalidade no mesmo PR/deploy
10. **Timeout/retry sem ajuste** — Chamar novo sistema com mesmos timeouts do antigo sem validar latência

---

## Referências

- **Strangler Fig Application** — Martin Fowler — https://martinfowler.com/bliki/StranglerFigApplication.html
- **Branch by Abstraction** — Martin Fowler — https://martinfowler.com/bliki/BranchByAbstraction.html
- **Parallel Change** — Danilo Sato — https://martinfowler.com/bliki/ParallelChange.html
- **Refactoring Databases** — Scott Ambler & Pramod Sadalage (Addison-Wesley, 2006)
- **Monolith to Microservices** — Sam Newman (O'Reilly, 2019)
- **Building Microservices, 2nd Ed** — Sam Newman (O'Reilly, 2021)
- **Database Reliability Engineering** — Laine Campbell & Charity Majors (O'Reilly, 2017)
- **6 Strategies for Migrating to the Cloud** — AWS — https://aws.amazon.com/blogs/enterprise-strategy/6-strategies-for-migrating-applications-to-the-cloud/
- **Expand and Contract Pattern** — Thoughtworks — https://www.thoughtworks.com/radar/techniques/expand-and-contract
- **Feature Toggles** — Martin Fowler — https://martinfowler.com/articles/feature-toggles.html
- **Release It!** — Michael Nygard (Pragmatic Bookshelf, 2018)
- **Accelerate** — Nicole Forsgren et al. (IT Revolution, 2018)
