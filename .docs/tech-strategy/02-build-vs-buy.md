# Build vs Buy — Frameworks de Decisão

> **Objetivo deste documento:** Servir como referência completa sobre **frameworks de decisão Build vs Buy**, incluindo análise de TCO, avaliação de vendors, matrizes de decisão e critérios de open source — otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: Build vs Buy decision frameworks, TCO, vendor evaluation, Make/Buy/Partner, open source assessment, SaaS vs self-hosted.

---

## Quick Reference — Cheat Sheet

| Conceito | Definição | Quando usar |
|----------|----------|------------|
| **Build** | Desenvolver internamente a solução | Core competency, diferenciador, controle total |
| **Buy (SaaS/License)** | Adquirir solução pronta | Commodity, fora do core, time-to-market crítico |
| **Partner** | Integrar com parceiro/plataforma | Expertise externa, co-development |
| **Open Source + Operate** | Usar OSS e manter operação interna | Controle + comunidade, sem vendor lock-in |
| **TCO** | Total Cost of Ownership — custo completo da decisão | Toda decisão Build vs Buy |
| **Wardley Evolution** | Genesis → Custom → Product → Commodity | Identificar se build ou buy faz sentido |

---

## Sumário

- [Build vs Buy — Frameworks de Decisão](#build-vs-buy--frameworks-de-decisão)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Quando Build, quando Buy?](#quando-build-quando-buy)
  - [Framework de Decisão Completo](#framework-de-decisão-completo)
  - [Total Cost of Ownership (TCO)](#total-cost-of-ownership-tco)
  - [Vendor Evaluation](#vendor-evaluation)
  - [Open Source Assessment](#open-source-assessment)
  - [SaaS vs Self-Hosted](#saas-vs-self-hosted)
  - [Make / Buy / Partner Matrix](#make--buy--partner-matrix)
  - [Wardley Mapping para Build vs Buy](#wardley-mapping-para-build-vs-buy)
  - [Case Studies](#case-studies)
  - [Anti-Patterns](#anti-patterns)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Quando Build, quando Buy?

```
REGRA FUNDAMENTAL:

  BUILD quando a tecnologia É o diferencial competitivo
  BUY quando a tecnologia SUPORTA o diferencial (mas não é ele)

  ┌─────────────────────────────────────────────────────────────┐
  │                                                              │
  │  "Se seu negócio é e-commerce, COMPRE autenticação,         │
  │   COMPRE email, COMPRE processo de pagamento.               │
  │   CONSTRUA o motor de recomendação, a experiência           │
  │   de checkout, o sistema de pricing dinâmico."              │
  │                                                              │
  │  "Se seu negócio é cybersecurity, CONSTRUA seus             │
  │   engines de detecção. COMPRE o sistema de billing,         │
  │   COMPRE o CMS, COMPRE o email marketing."                  │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘

  HEURÍSTICAS RÁPIDAS:

  BUILD ✅                          BUY ✅
  ─────────                         ──────
  Core business logic               Autenticação (Auth0, Cognito)
  Proprietary algorithms            CI/CD (GitHub Actions, CircleCI)
  Competitive advantage             Email/Notifications (SendGrid)
  Unique workflow                   Payment processing (Stripe)
  Nenhuma solução fit no mercado    Monitoring (Datadog, Grafana)
  Controle total necessário         CMS (Contentful, Strapi)
  Data/IP sensitivity extreme       Feature flags (LaunchDarkly)
  
  ⚠️ BUILD COM CAUTELA              ⚠️ BUY COM CAUTELA
  ──────────────────                 ───────────────────
  Queue/Message broker               Custom ERP
  API Gateway                        Custom database engine
  Internal developer platform        Vendor lock-in heavy
  Custom observability               Solução 10x mais cara que build
```

---

## Framework de Decisão Completo

```
FRAMEWORK DE 7 DIMENSÕES PARA BUILD vs BUY:

  Para cada dimensão, pontuar de 1 (favorece BUY) a 5 (favorece BUILD):

  ┌───────────────────────────────────────────────────────────────────┐
  │                                                                    │
  │  DIMENSÃO 1: CORE vs CONTEXT                                      │
  │                                                                    │
  │  1 ──────────────────────────────────── 5                          │
  │  Commodity / Context                   Core Competency              │
  │  "Todos os competitors                 "Isto NOS diferencia        │
  │   resolvem igual"                       no mercado"                │
  │                                                                    │
  │  Email sending → 1                     Recommendation engine → 5   │
  │  User auth → 1-2                       Pricing engine → 5          │
  │  Log aggregation → 2                   Core domain logic → 5       │
  │  API Gateway → 3                       Custom workflow → 4         │
  │                                                                    │
  └───────────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────────┐
  │  DIMENSÃO 2: MARKET FIT                                            │
  │                                                                    │
  │  1 ──────────────────────────────────── 5                          │
  │  Solução perfeita existe               Nenhuma solução encaixa     │
  │  no mercado                            no mercado                  │
  │                                                                    │
  │  Se existe solução madura que resolve 90%+: BUY                    │
  │  Se nenhuma solução cobre seus requirements: BUILD                 │
  │  Se cobre 60-80%: avaliar custo de customização vs build           │
  │                                                                    │
  │  CUIDADO: "90% fit" significa que 10% restante pode ser            │
  │  exactamente o diferencial que você precisa                        │
  │                                                                    │
  └───────────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────────┐
  │  DIMENSÃO 3: TIME-TO-MARKET                                       │
  │                                                                    │
  │  1 ──────────────────────────────────── 5                          │
  │  Preciso AGORA (weeks)                 Posso investir (months)     │
  │                                                                    │
  │  BUY é quase sempre mais rápido no curto prazo                     │
  │  BUILD pode ser mais rápido no longo prazo para iterações          │
  │                                                                    │
  │  Pressão de mercado alta → BUY (para entrar rápido)               │
  │  Depois: avaliar BUILD para substituir se for core                 │
  │                                                                    │
  └───────────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────────┐
  │  DIMENSÃO 4: TEAM CAPABILITY                                       │
  │                                                                    │
  │  1 ──────────────────────────────────── 5                          │
  │  Sem expertise interna                 Expertise forte interna     │
  │                                                                    │
  │  Temos equipe que sabe construir E manter? → BUILD viável         │
  │  Precisamos contratar/treinar? → Adicionar ao TCO de BUILD        │
  │  Equipe vai crescer nessa área? → Investimento em build faz sentido │
  │  Equipe quer ownership dessa área? → Motivação conta               │
  │                                                                    │
  └───────────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────────┐
  │  DIMENSÃO 5: OPERAÇÃO E MANUTENÇÃO                                │
  │                                                                    │
  │  1 ──────────────────────────────────── 5                          │
  │  Quero SLA sem responsabilidade        Quero controle total        │
  │                                                                    │
  │  BUY: vendor cuida de uptime, patches, scaling                     │
  │  BUILD: VOCÊ cuida de tudo (24/7 on-call, patching, upgrades)     │
  │                                                                    │
  │  Perguntas-chave:                                                  │
  │  → Quem acorda às 3AM quando isso quebrar?                        │
  │  → Quem aplica security patches?                                   │
  │  → Quem escala quando crescer 10x?                                │
  │                                                                    │
  └───────────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────────┐
  │  DIMENSÃO 6: VENDOR RISK                                           │
  │                                                                    │
  │  1 ──────────────────────────────────── 5                          │
  │  Vendor estável, lock-in                Lock-in inaceitável,       │
  │  aceitável                              controle é crítico         │
  │                                                                    │
  │  Avaliar:                                                          │
  │  → Vendor pode ser adquirido/fechar?                               │
  │  → Pricing pode mudar dramaticamente? (ex: Twitter API, Redis)    │
  │  → Lock-in é reversível? Portabilidade dos dados?                 │
  │  → O que acontece se vendor viola compliance?                      │
  │                                                                    │
  └───────────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────────┐
  │  DIMENSÃO 7: COMPLIANCE & SECURITY                                 │
  │                                                                    │
  │  1 ──────────────────────────────────── 5                          │
  │  Vendor compliance é                    Dados sensíveis,           │
  │  suficiente                             regulação rígida           │
  │                                                                    │
  │  Se dados precisam ficar on-premises → BUILD ou self-hosted       │
  │  Se vendor tem SOC2/ISO27001 → BUY aceitável                      │
  │  Se regulação exige controle de dados → BUILD                     │
  │                                                                    │
  └───────────────────────────────────────────────────────────────────┘

  SCORING:
  
  │ Score Total (soma)│ Recomendação       │
  │ 7 — 15            │ BUY (forte)        │
  │ 16 — 22           │ Avaliar com mais profundidade │
  │ 23 — 28           │ BUILD (considere)  │
  │ 29 — 35           │ BUILD (forte)      │
  
  ⚠️ Scoring é GUIA, não regra absoluta.
  Uma dimensão pode ter peso desproporcional no contexto.
```

---

## Total Cost of Ownership (TCO)

```
TCO — CUSTO TOTAL DE PROPRIEDADE:

  TCO = Custo de Aquisição + Custo de Operação + Custo de Saída

  ═══════════════════════════════════════════════════════════════
  COMPONENTE 1: CUSTO DE AQUISIÇÃO (one-time + setup)
  ═══════════════════════════════════════════════════════════════

  BUY (SaaS/License):                 BUILD:
  ─────────────────                    ──────
  • Licença/subscription               • Desenvolvimento (sprints)
  • Setup fee                          • Design + Architecture
  • Integração                         • Code review + QA
  • Migração de dados                  • Testing + staging
  • Treinamento                        • Initial deployment
  • Customização inicial               • Documentação
  • POC/piloto                         • Treinamento interno

  ═══════════════════════════════════════════════════════════════
  COMPONENTE 2: CUSTO DE OPERAÇÃO (anual, recorrente)
  ═══════════════════════════════════════════════════════════════

  BUY:                                 BUILD:
  ────                                 ──────
  • Subscription anual                 • Infraestrutura (cloud)
  • Overage fees (escala)              • On-call + SRE
  • Professional services              • Bug fixes + patches
  • Upgrade fees                       • Security updates
  • Support tiers                      • Feature development
  • Compliance audit                   • Performance tuning
                                       • Scaling engineering
                                       • Dependency updates

  ═══════════════════════════════════════════════════════════════
  COMPONENTE 3: CUSTO DE OPORTUNIDADE
  ═══════════════════════════════════════════════════════════════

  BUILD:                               BUY:
  ──────                               ────
  • Engenheiros trabalhando nisso       • Vendor dependency
    NÃO trabalham em features core     • Roadmap controlado pelo vendor
  • Meses até primeiro valor            • Customização limitada
  • Custo de recrutamento para          • Menos aprendizado interno
    expertise específica                

  ═══════════════════════════════════════════════════════════════
  COMPONENTE 4: CUSTO DE SAÍDA (switching cost)
  ═══════════════════════════════════════════════════════════════

  BUY:                                 BUILD:
  ────                                 ──────
  • Data export/migration              • Sunk cost (investimento perdido)
  • Contrato de saída (penalties)      • Knowledge loss se equipe sair
  • Reescrita de integrações           • Legacy maintenance burden
  • Downtime durante migração          • Sem suporte externo
  • Retreinamento equipe               • Reescrever se mal arquitetado

  ═══════════════════════════════════════════════════════════════

  EXEMPLO COMPARATIVO — AUTENTICAÇÃO

  ┌──────────────────────────┬──────────────┬──────────────────┐
  │ Item                     │ BUILD        │ BUY (Auth0)      │
  ├──────────────────────────┼──────────────┼──────────────────┤
  │ Setup (month 0-3)        │ $120K        │ $15K             │
  │  Dev team (3 eng × 3mo)  │ $90K         │ -                │
  │  Security audit           │ $20K         │ Included         │
  │  Integration              │ $10K         │ $15K             │
  ├──────────────────────────┼──────────────┼──────────────────┤
  │ Year 1 Operation          │ $95K         │ $48K             │
  │  Maintenance (1 eng)      │ $60K         │ -                │
  │  Infrastructure           │ $15K         │ -                │
  │  Security patches         │ $20K         │ Included         │
  │  Subscription (1K MAU)    │ -            │ $48K             │
  ├──────────────────────────┼──────────────┼──────────────────┤
  │ Year 2 Operation          │ $95K         │ $72K             │
  │  (escala 10K MAU)         │              │                  │
  ├──────────────────────────┼──────────────┼──────────────────┤
  │ Year 3 Operation          │ $95K         │ $120K            │
  │  (escala 100K MAU)        │              │                  │
  ├──────────────────────────┼──────────────┼──────────────────┤
  │ 3-YEAR TCO                │ $405K        │ $255K            │
  │ 5-YEAR TCO                │ $595K        │ $495K            │
  ├──────────────────────────┼──────────────┼──────────────────┤
  │ Opportunity cost          │ 3 eng × 3mo │ -                │
  │ (features não entregues)  │ = 9 eng-mo   │                  │
  ├──────────────────────────┼──────────────┼──────────────────┤
  │ RESULTADO                 │ BUILD: mais  │ BUY: mais barato │
  │                           │ caro em 3yr  │ em 3yr, break-   │
  │                           │              │ even em ~5yr      │
  └──────────────────────────┴──────────────┴──────────────────┘

  CONCLUSÃO: para auth → BUY (commodity, não é diferenciador)
  Para pricing engine de fintech → BUILD (diferenciador competitivo)
```

---

## Vendor Evaluation

```
VENDOR EVALUATION SCORECARD:

  ┌─────────────────────────────────────────────────────────────────┐
  │ CRITÉRIO                │ Peso │ Vendor A │ Vendor B │ Build   │
  ├─────────────────────────┼──────┼──────────┼──────────┼─────────┤
  │ FUNCIONALIDADE                                                  │
  │ Feature completeness    │  15% │  4/5     │  3/5     │  5/5    │
  │ Customizability         │  10% │  2/5     │  4/5     │  5/5    │
  │ Integration APIs        │  10% │  4/5     │  3/5     │  5/5    │
  ├─────────────────────────┼──────┼──────────┼──────────┼─────────┤
  │ OPERAÇÃO                                                        │
  │ SLA/Uptime guarantee    │   8% │  4/5     │  5/5     │  3/5    │
  │ Support quality         │   5% │  3/5     │  4/5     │  N/A    │
  │ Documentation           │   5% │  4/5     │  3/5     │  3/5    │
  ├─────────────────────────┼──────┼──────────┼──────────┼─────────┤
  │ SEGURANÇA & COMPLIANCE                                          │
  │ SOC2 / ISO27001         │   8% │  5/5     │  5/5     │  3/5    │
  │ Data residency options  │   5% │  3/5     │  4/5     │  5/5    │
  │ Encryption (rest/transit)│  5% │  5/5     │  5/5     │  4/5    │
  ├─────────────────────────┼──────┼──────────┼──────────┼─────────┤
  │ CUSTO & RISCO                                                   │
  │ TCO (3 years)           │  12% │  3/5     │  4/5     │  2/5    │
  │ Vendor viability        │   5% │  4/5     │  3/5     │  N/A    │
  │ Switching cost          │   7% │  2/5     │  3/5     │  5/5    │
  │ Pricing predictability  │   5% │  3/5     │  4/5     │  5/5    │
  └─────────────────────────┴──────┴──────────┴──────────┴─────────┘

  CHECKLIST DE DUE DILIGENCE PARA VENDOR:

  □ Empresa é financeiramente saudável? (funding, revenue)
  □ Tem clientes do nosso tamanho/indústria?
  □ Roadmap alinhado com nossas necessidades futuras?
  □ SLA inclui compensation real? (não "best effort")
  □ Suporte: canais, SLA de resposta, dedicated CSM?
  □ Security: pen test reports? Bug bounty? SOC2?
  □ Data export: posso exportar TODOS os meus dados?
  □ API: REST/GraphQL documentada? Rate limits razoáveis?
  □ Pricing: previsível? Overage caps? Renegociação anual?
  □ Contrato: termination clause? Lock-in period?
  □ Referências: 3+ clientes similares satisfeitos?
  □ Compliance: LGPD, GDPR, HIPAA (se aplicável)?
  □ Integration: webhooks, SDK, SSO/SAML?
  □ Multi-tenancy ou single-tenant?
  □ Disaster recovery: RPO/RTO do vendor?
```

---

## Open Source Assessment

```
OPEN SOURCE EVALUATION FRAMEWORK:

  Nem todo open source é igual. Avaliar antes de adotar:

  ┌──────────────────────────────────────────────────────────────┐
  │ DIMENSÃO              │ SINAIS POSITIVOS    │ RED FLAGS       │
  ├───────────────────────┼─────────────────────┼─────────────────┤
  │ Manutenção            │ Commits regulares   │ Sem commit      │
  │                       │ Releases frequentes │ há 6+ meses     │
  │                       │ Maintainers ativos  │ 1 maintainer    │
  ├───────────────────────┼─────────────────────┼─────────────────┤
  │ Comunidade            │ 1000+ stars         │ < 100 stars     │
  │                       │ Issues respondidas  │ Issues ignoradas│
  │                       │ PRs aceitos         │ PRs pendentes   │
  │                       │                     │ há meses        │
  ├───────────────────────┼─────────────────────┼─────────────────┤
  │ Licença               │ Apache 2.0 / MIT    │ AGPL / SSPL     │
  │                       │ BSD                  │ Custom license  │
  │                       │ MPL 2.0             │ "Source available"│
  ├───────────────────────┼─────────────────────┼─────────────────┤
  │ Segurança             │ CVE response <30d   │ CVEs não fixados│
  │                       │ Security policy      │ Sem security    │
  │                       │ Signed releases      │ policy          │
  ├───────────────────────┼─────────────────────┼─────────────────┤
  │ Documentação          │ Getting started      │ README mínimo  │
  │                       │ API docs             │ Exemplos antigos│
  │                       │ Migration guides     │ Sem changelog   │
  ├───────────────────────┼─────────────────────┼─────────────────┤
  │ Estabilidade          │ Semver respeitado   │ Breaking changes│
  │                       │ LTS versions         │ frequentes      │
  │                       │ Deprecation notices  │ Sem migration   │
  │                       │                     │ guide            │
  ├───────────────────────┼─────────────────────┼─────────────────┤
  │ Backing               │ CNCF / Apache Found.│ Single-company  │
  │                       │ Linux Foundation     │ controlled      │
  │                       │ Multi-company        │ (bait-and-switch│
  │                       │ contributors         │ risk: Redis,    │
  │                       │                     │ Elasticsearch)  │
  └───────────────────────┴─────────────────────┴─────────────────┘

  LICENÇAS — QUICK GUIDE:

  │ Licença      │ Uso comercial │ Modificável │ Obriga share?    │
  │ MIT          │ ✅            │ ✅          │ ❌               │
  │ Apache 2.0   │ ✅            │ ✅          │ ❌ (+ patent)    │
  │ BSD 2/3      │ ✅            │ ✅          │ ❌               │
  │ MPL 2.0      │ ✅            │ ✅          │ ⚠️ Arquivo mod   │
  │ LGPL 2.1/3   │ ✅            │ ✅          │ ⚠️ Se modifica   │
  │ GPL 2/3      │ ⚠️ Condicional│ ✅          │ ✅ Derivative    │
  │ AGPL 3       │ ⚠️ Condicional│ ✅          │ ✅ Network use   │
  │ SSPL         │ ❌ Practical  │ ✅          │ ✅ Toda stack    │

  RISCO DE "BAIT AND SWITCH":
  → Projeto OSS popular → empresa controla → muda licença
  → Exemplos recentes: Redis (SSPL), Elasticsearch (SSPL→AGPL),
    HashiCorp Terraform (BUSL), MongoDB (SSPL)
  → Mitigação: preferir projetos com governance independente
    (CNCF, Apache Foundation, Linux Foundation)
  → Alternativa: forks abertos (OpenTofu, OpenSearch, Valkey)
```

---

## SaaS vs Self-Hosted

```
SaaS vs SELF-HOSTED — DECISION MATRIX:

  ┌─────────────────────────────────┬──────────────┬──────────────┐
  │ Dimensão                        │ SaaS         │ Self-Hosted  │
  ├─────────────────────────────────┼──────────────┼──────────────┤
  │ Responsabilidade operacional    │ Vendor       │ Sua equipe   │
  │ Time-to-value                   │ Dias/semanas │ Semanas/meses│
  │ Customização                    │ Limitada     │ Total        │
  │ Data residency / soberania      │ Depende      │ Total control│
  │ Custo em escala pequena         │ Menor        │ Maior        │
  │ Custo em escala grande          │ Maior        │ Menor        │
  │ Security patches                │ Vendor       │ Sua equipe   │
  │ Disaster recovery               │ Vendor       │ Sua equipe   │
  │ Latência (data proximity)       │ Depende      │ Controlável  │
  │ Compliance (SOC2, HIPAA)        │ Vendor provê │ Você provê   │
  │ Vendor lock-in risk             │ Alto         │ Baixo        │
  │ Equipe de infra necessária      │ Não          │ Sim          │
  └─────────────────────────────────┴──────────────┴──────────────┘

  DECISÃO POR CENÁRIO:

  STARTUP (< 50 eng)
  → SaaS quase sempre (não gaste eng-time com infra commodity)
  → Exceção: compliance requer self-hosted

  SCALE-UP (50-200 eng)
  → SaaS para commodity + self-hosted para core/cost-sensitive
  → Começar a internalizar onde custo de SaaS explode

  ENTERPRISE (200+ eng)
  → Mix: SaaS + self-hosted + managed cloud services
  → Platform team para gerenciar self-hosted
  → Negociar enterprise agreements com vendors

  HYBRID — BEST OF BOTH WORLDS:
  → Control Plane: SaaS (gestão, UI)
  → Data Plane: self-hosted (dados ficam com você)
  → Exemplos: Temporal Cloud, Confluent, Grafana Cloud
```

---

## Make / Buy / Partner Matrix

```
MAKE / BUY / PARTNER — DECISION FRAMEWORK:

  ┌────────────────────────────────────────────────────────────────┐
  │                                                                 │
  │                    STRATEGIC IMPORTANCE                          │
  │                    (para o negócio)                              │
  │                                                                 │
  │  Alto  │  PARTNER            │  MAKE (BUILD)                    │
  │        │                     │                                  │
  │        │  Strategic mas fora  │  Core competency,               │
  │        │  da competência core │  diferencial competitivo        │
  │        │                     │                                  │
  │        │  Ex: AI/ML service   │  Ex: Recommendation engine,     │
  │        │  com parceiro,       │  pricing engine,                │
  │        │  co-innovation       │  core domain logic              │
  │        │                     │                                  │
  │  ──────┼─────────────────────┼──────────────────────────────── │
  │        │                     │                                  │
  │  Baixo │  BUY (SaaS/License) │  BUY + CUSTOMIZE                │
  │        │                     │                                  │
  │        │  Commodity, solved   │  Não é core, mas precisa        │
  │        │  problem, low        │  de customização moderada       │
  │        │  differentiation     │                                  │
  │        │                     │  Ex: CRM configurado,            │
  │        │  Ex: Email, auth,    │  ERP com integrações,            │
  │        │  monitoring,         │  Analytics customizado           │
  │        │  CI/CD               │                                  │
  │        │                     │                                  │
  │        └─────────────────────┴──────────────────────────────── │
  │         Baixa                                     Alta          │
  │                    INTERNAL CAPABILITY                           │
  │                    (expertise da equipe)                         │
  │                                                                 │
  └────────────────────────────────────────────────────────────────┘

  PARTNER — QUANDO USAR:
  → Strategicamente importante + sem expertise interna
  → Co-development com empresa especializada
  → Joint ventures para tecnologia compartilhada
  → Consulting engagement para transferir conhecimento
  → Exemplo: startup de fintech faz partner com payment processor
    → Processor traz compliance/expertise
    → Startup traz inovação de UX
  
  EVOLUÇÃO NATURAL:
  PARTNER → BUILD (quando skill internalizado)
  BUY → BUILD (quando custo de vendor supera build)
  BUILD → BUY (quando não é mais diferenciador)
```

---

## Wardley Mapping para Build vs Buy

```
WARDLEY MAPPING — POSICIONAMENTO NA CADEIA DE VALOR:

  VISIBILIDADE                                        EVOLUÇÃO
  para o usuário                                      tecnológica
  
  Alta │                                              
       │  [User Experience]                            
       │       │                                       
       │  [Business Logic]                             
       │       │          │                            
       │  [API Gateway]   [Auth Service]               
       │       │          │                            
       │  [Messaging]     [Monitoring]                 
       │       │          │                            
       │  [Database]      [CI/CD]                      
       │       │          │                            
       │  [Compute]       [Network]                    
  Baixa│                                              
       └──────────────────────────────────────────────
        Genesis    Custom    Product    Commodity
         │           │          │           │
         │           │          │           │
       BUILD      BUILD     BUY+Custom    BUY
       (inovação) (vantagem)(configura)   (commodity)

  REGRA WARDLEY:
  → Genesis/Custom = BUILD (diferenciação, inovação)
  → Product = BUY + CUSTOMIZE (configurable solutions)
  → Commodity = BUY/SaaS/Managed (não gaste tempo)

  EXEMPLO APLICADO:

  │ Componente         │ Evolução   │ Ação          │ Exemplo         │
  │ Checkout UX        │ Custom     │ BUILD         │ Custom React    │
  │ Pricing Engine     │ Custom     │ BUILD         │ Internal service│
  │ Payment Processing │ Product    │ BUY (Stripe)  │ API integration │
  │ Auth/Identity      │ Product    │ BUY (Auth0)   │ SDK integration │
  │ Email sending      │ Commodity  │ BUY (SES)     │ API call        │
  │ Monitoring         │ Product    │ BUY (Datadog) │ Agent + config  │
  │ Compute            │ Commodity  │ BUY (AWS)     │ EKS / Lambda    │

  MOVIMENTO AO LONGO DO TEMPO:
  → Tecnologias evoluem: Custom → Product → Commodity
  → O que você BUILD hoje, pode BUY amanhã
  → O que é differentiator hoje, é commodity em 5 anos
  → Reavaliar periodicamente
```

---

## Case Studies

```
CASE STUDY 1 — OBSERVABILITY STACK

  OPÇÃO A: BUILD (ELK + Prometheus + Jaeger + Grafana)
  ─────
  TCO 3yr (equipe 100 eng): $380K
  • Infra: $120K/yr (ELK cluster, Prometheus, storage)
  • Eng: 1 FTE dedicado + 20% de 2 SREs = $140K/yr
  • Setup: $80K (3 meses de 2 eng)
  • Pros: controle total, sem vendor lock-in
  • Cons: manutenção pesada, escalar ELK é dor

  OPÇÃO B: BUY (Datadog)
  ─────
  TCO 3yr (equipe 100 eng): $540K
  • Subscription: $180K/yr
  • Setup: $20K (2 semanas)
  • Eng: 10% de 1 SRE = $15K/yr
  • Pros: setup rápido, UX excelente, correlaçãao
  • Cons: custo escala com volume, vendor lock-in

  OPÇÃO C: HYBRID (Grafana Cloud + self-hosted LGTM stack)
  ─────
  TCO 3yr (equipe 100 eng): $290K
  • Grafana Cloud: $60K/yr
  • Infra parcial: $30K/yr
  • Eng: 30% de 1 SRE = $45K/yr
  • Pros: flexibility, OSS-based, good TCO
  • Cons: mais complexo que pure SaaS

  DECISÃO: Para equipe < 50 eng → BUY (Datadog)
           Para equipe 50-200 eng → HYBRID (Grafana ecosystem)
           Para equipe 200+ eng → BUILD ou HYBRID

  ─────────────────────────────────────────────────────────

  CASE STUDY 2 — FEATURE FLAGS

  BUILD: Simple key-value store + SDK interno
  TCO 3yr: $95K (setup $40K + $18K/yr manutenção)
  Funcionalidade: 30% do LaunchDarkly

  BUY: LaunchDarkly
  TCO 3yr: $120K ($40K/yr)
  Funcionalidade: 100% (targeting, metrics, experiments)

  BUY: Flagsmith (self-hosted, open source)
  TCO 3yr: $45K (free + $15K/yr infra/manutenção)
  Funcionalidade: 80% do LaunchDarkly

  DECISÃO: Feature flags não é core → BUY (Flagsmith OSS)
  → Bom TCO, sem vendor lock-in, funcionalidade suficiente
```

---

## Anti-Patterns

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **NIH Syndrome** | "Not Invented Here" — rebuild tudo | Honestidade: isso é core business? Se não → BUY |
| **Vendor Worship** | "AWS has a service for that" → compra tudo | Avaliar TCO — managed service nem sempre é mais barato |
| **Lock-in Blindness** | Ignora vendor lock-in até ser tarde | Avaliar switching cost ANTES de comprar |
| **TCO Amnesia** | Compara só preço de setup (ignora operação) | Sempre calcular TCO de 3-5 anos |
| **Shiny Object** | "Lançaram ferramenta nova, vamos usar!" | Tech Radar assessment antes de adotar |
| **Build Everything** | Constrói auth, email, CI/CD internamente | Auth não é diferencial (a menos que seja seu negócio) |
| **Buy and Forget** | Compra e nunca reavalia | Revisão anual de vendors e contratos |
| **Open Source = Free** | "É open source, custo zero" | OSS tem custo: operate, patch, upgrade, debug |
| **Sunk Cost Trap** | "Já investimos $500K, não podemos abandonar" | Avaliar custo FUTURO, não passado |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código e decisões de build vs buy, verifique:

1. **Reimplementando commodity** — Auth, email, payments, feature flags custom quando soluções maduras existem → questionar
2. **Sem TCO analysis** — Decisão de build/buy sem análise de TCO de pelo menos 3 anos
3. **Vendor lock-in sem abstração** — SDK de vendor usado diretamente em business logic sem interface/adapter
4. **Open source sem avaliação** — Nova dependência OSS sem checar: licença, manutenção, segurança, community health
5. **Licença incompatível** — AGPL/SSPL em projeto comercial sem awareness das implicações
6. **Single-vendor dependency** — Componente crítico dependente de 1 vendor sem plano B
7. **Build em área sem expertise** — Equipe construindo sistema complexo (crypto, auth, payment) sem experiência → risco alto
8. **Middleware interno duplicando OSS** — Framework interno que replica funcionalidade de projeto OSS maduro
9. **Sem exit strategy** — Integração com vendor sem plano de migração caso vendor falhe/mude pricing
10. **SaaS sem data export strategy** — Dados em SaaS sem mecanismo testado de export/portabilidade

---

## Referências

- **Choose Boring Technology** — Dan McKinley — https://mcfunley.com/choose-boring-technology
- **Wardley Mapping** — Simon Wardley — https://learnwardleymapping.com/
- **Technology Strategy Patterns** — Eben Hewitt (O'Reilly, 2018)
- **The Build Trap** — Melissa Perri (O'Reilly, 2018)
- **Software Architecture: The Hard Parts** — Neal Ford et al. (O'Reilly, 2021)
- **Fundamentals of Software Architecture** — Mark Richards & Neal Ford (O'Reilly, 2020)
- **An Elegant Puzzle** — Will Larson (Stripe Press, 2019)
- **Team Topologies** — Matthew Skelton & Manuel Pais (IT Revolution, 2019)
- **The Phoenix Project** — Gene Kim et al. (IT Revolution, 2013)
- **Open Source Guides** — https://opensource.guide/
- **CNCF Landscape** — https://landscape.cncf.io/
- **Thoughtworks Technology Radar** — https://www.thoughtworks.com/radar
