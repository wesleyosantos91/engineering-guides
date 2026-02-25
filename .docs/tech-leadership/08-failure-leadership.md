# Failure Leadership — Liderando Através de Falhas

> **Objetivo deste documento:** Servir como referência completa sobre **failure leadership** — como liderar blameless post-mortems, transformar falhas em melhorias sistêmicas, e construir cultura de aprendizado — otimizado para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Escopo: blameless culture, incident response leadership, post-mortem facilitation, systemic improvements, learning organizations, resilience engineering.

---

## Quick Reference — Cheat Sheet

| Conceito | Definição | Quando usar |
|----------|----------|-------------|
| **Blameless Post-mortem** | Análise de incidente sem buscar culpados | Após todo incidente significativo |
| **5 Whys** | Técnica de perguntar "por quê?" 5x para achar root cause | Root cause analysis |
| **Swiss Cheese Model** | Falhas passam por múltiplas camadas de defesa | Entender defesas sistêmicas |
| **Just Culture** | Accountability sem punição por erros honestos | Definir cultura de segurança |
| **Chaos Engineering** | Injetar falhas intencionais para aprender | Validar resiliência proativamente |
| **Error Budget** | SLO inversion: quanto downtime é "aceitável" | Balancear velocity e reliability |
| **Learning Review** | Reunião focada em learnings (não blame) | Após incidentes e near-misses |

---

## Sumário

- [Failure Leadership — Liderando Através de Falhas](#failure-leadership--liderando-através-de-falhas)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Por Que Failure Leadership Importa](#por-que-failure-leadership-importa)
  - [Blameless Culture — Os Fundamentos](#blameless-culture--os-fundamentos)
  - [Incident Response Leadership](#incident-response-leadership)
  - [Facilitando Post-mortems](#facilitando-post-mortems)
  - [De Incidentes a Melhorias Sistêmicas](#de-incidentes-a-melhorias-sistêmicas)
  - [Chaos Engineering e Proactive Failure](#chaos-engineering-e-proactive-failure)
  - [Construindo uma Learning Organization](#construindo-uma-learning-organization)
  - [Papel de Cada Nível em Failure Leadership](#papel-de-cada-nível-em-failure-leadership)
  - [Anti-Patterns](#anti-patterns)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## Por Que Failure Leadership Importa

```
A REALIDADE DOS SISTEMAS COMPLEXOS:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  "In complex systems, failure is not a matter of IF,   │
  │   but WHEN." — Sidney Dekker                           │
  │                                                         │
  │  → Microservices com 50 componentes                    │
  │  → Cada um com 99.9% uptime individual                │
  │  → Probabilidade de TUDO UP = 0.999^50 = 95.1%        │
  │  → = ~18 dias de algum componente down por ano         │
  │                                                         │
  │  FALHAS SÃO INEVITÁVEIS. A QUESTÃO:                    │
  │  → As falhas DESTROEM confiança e moral?               │
  │  → Ou falhas FORTALECEM o sistema e o time?            │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  DOIS MODELOS DE RESPOSTA A FALHA:

  ┌──────────────────────────┬───────────────────────────────┐
  │ BLAME CULTURE            │ LEARNING CULTURE              │
  ├──────────────────────────┼───────────────────────────────┤
  │ "Quem fez o deploy?"     │ "O que o sistema permitiu?"   │
  │ "Como isso passou       │ "Como nossas defesas          │
  │  no review?"             │  falharam?"                   │
  │ Engineers escondem erros  │ Engineers reportam proactive  │
  │ Incentivo: não mudar nada │ Incentivo: experimentar       │
  │ Post-mortem = tribunal   │ Post-mortem = aprendizado      │
  │ "Quem podemos demitir?"  │ "Que processo está quebrado?" │
  │ MTTR alto (medo de agir) │ MTTR baixo (autonomia)        │
  │ Mesmos bugs se repetem    │ Sistema evolui continuamente  │
  └──────────────────────────┴───────────────────────────────┘

  O CUSTO DA BLAME CULTURE:

  → Engineers param de fazer deploys na sexta
    (medo, não processo)
  → Ninguém oferece para ser on-call
  → Incidentes se repetem (ninguém reporta near-misses)
  → Innovation para (medo de falhar)
  → Bons engineers pedem demissão

  O VALOR DA LEARNING CULTURE:

  → Deploys são seguros a qualquer hora
    (porque defesas são robustas)
  → On-call é rotação normal (não punição)
  → Near-misses são reportados = prevenção
  → Innovation é segura (feature flags, canary, rollback)
  → Top talent quer trabalhar aqui
```

---

## Blameless Culture — Os Fundamentos

```
JUST CULTURE — O FRAMEWORK:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  JUST CULTURE não é "ninguém é responsável".           │
  │  É: "pessoas são responsáveis pelas DECISÕES,          │
  │   não pelo OUTCOME quando seguiram processo razoável." │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  3 CATEGORIAS DE COMPORTAMENTO:

  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │ 1. ERRO HUMANO (HUMAN ERROR)                          │
  │    → Pessoa tentou fazer a coisa certa                 │
  │    → E cometeu erro não intencional                    │
  │    → Ex: Deploy com bug que passou no code review      │
  │    → RESPOSTA: Console, aprenda, melhore processo     │
  │    → NÃO: Punição                                     │
  │                                                        │
  │ 2. COMPORTAMENTO DE RISCO (AT-RISK BEHAVIOR)          │
  │    → Pessoa cortou caminho conscientemente             │
  │    → Porque o sistema incentiva ou tolera              │
  │    → Ex: Skip staging porque "é uma mudança pequena"  │
  │    → RESPOSTA: Entenda por que. Fix incentives/process │
  │    → NÃO: Punição individual                          │
  │                                                        │
  │ 3. COMPORTAMENTO NEGLIGENTE (RECKLESS BEHAVIOR)       │
  │    → Pessoa conscientemente ignorou riscos conhecidos  │
  │    → Sem justificativa razoável                        │
  │    → Ex: Desativar todos os testes para "ir mais rápido"│
  │    → RESPOSTA: Accountability com consequências        │
  │    → SIM: Consequências proporcionais                  │
  │                                                        │
  └────────────────────────────────────────────────────────┘

  REGRA DE OURO:

  "Se a mesma pessoa, no mesmo contexto, com as mesmas
   informações, tivesse feito a mesma coisa 9 em 10 vezes,
   o SISTEMA precisa mudar, não a PESSOA."

  SWISS CHEESE MODEL — COMO INCIDENTES ACONTECEM:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │ INCIDENTE = FALHA que passa por TODAS as camadas       │
  │                                                         │
  │ ══════     ══════     ══════     ══════                │
  │ │ ○  │     │○   │     │    │     │  ○ │   ← Camadas   │
  │ │    │     │    │     │  ○ │     │    │     de defesa  │
  │ │    │     │    │     │    │     │    │                 │
  │ ══════     ══════     ══════     ══════                │
  │ Code       Code       Staging    Monitoring            │
  │ Review     CI/Tests   Test       + Alerting            │
  │                                                         │
  │ Quando os "buracos" se alinham, falha passa por tudo:  │
  │                                                         │
  │ ══════     ══════     ══════     ══════                │
  │ │    │     │    │     │    │     │    │                 │
  │ │ ●──┼─────┼──●─┼─────┼──●─┼─────┼──●─┼──→ INCIDENTE!│
  │ │    │     │    │     │    │     │    │                 │
  │ ══════     ══════     ══════     ══════                │
  │                                                         │
  │ LIÇÃO: Nenhuma camada é perfeita.                      │
  │ Resiliência = REDUNDÂNCIA de defesas.                  │
  │ Post-mortem = encontrar onde os buracos se alinharam.  │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  COMO CONSTRUIR BLAMELESS CULTURE:

  1. LINGUAGEM IMPORTA:
     ❌ "Quem fez o deploy que causou isso?"
     ✅ "Que condição permitiu que essa mudança 
         chegasse a produção com esse bug?"
     
     ❌ "Por que você não testou isso?"
     ✅ "Nosso processo de teste cobria esse cenário?"

  2. LEADERSHIP PRIMEIRO:
     → Quando VP/Director pune alguém por incidente,
       TODA a org recebe a mensagem: "esconda erros"
     → Quando VP reconhece publicamente: "Time, o incidente
       foi difícil, mas nosso post-mortem gerou 3 melhorias
       que já implementamos" → mensagem: "falhar é ok"

  3. CELEBRATE NEAR-MISSES:
     → Near-miss = quase-incidente que foi evitado
     → Reportar near-miss deveria ser CELEBRADO
     → "Obrigado por reportar! Isso preveniu um incidente!"

  4. REWARD LEARNING, NOT PERFECTION:
     → Promo criteria inclui "liderou post-mortem que gerou
       melhoria X"
     → Não inclui "zero incidentes" (incentiva esconder)
```

---

## Incident Response Leadership

```
COMO LIDERAR DURANTE UM INCIDENTE:

  ROLES DURANTE INCIDENTE:

  ┌────────────────────────────────────────────────────────┐
  │ ROLE            │ RESPONSABILIDADE                     │
  ├─────────────────┼──────────────────────────────────────┤
  │ Incident        │ Coordena resposta, toma decisões,    │
  │ Commander (IC)  │ mantém foco, comunica status         │
  │                 │ NÃO debugga — LIDERA                 │
  ├─────────────────┼──────────────────────────────────────┤
  │ Technical Lead  │ Diagnostica e resolve                │
  │                 │ Propõe mitigation strategies          │
  │                 │ Executa fix/rollback                  │
  ├─────────────────┼──────────────────────────────────────┤
  │ Communications  │ Atualiza stakeholders/clientes       │
  │ Lead            │ Traduz tech para business language    │
  │                 │ Mantém status page atualizada         │
  ├─────────────────┼──────────────────────────────────────┤
  │ Scribe          │ Documenta timeline em real-time      │
  │                 │ Registra decisões e ações             │
  │                 │ Matéria-prima para post-mortem        │
  └─────────────────┴──────────────────────────────────────┘

  COMPORTAMENTO DO INCIDENT COMMANDER:

  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │ ✅ FALAR DEVAGAR e claramente                         │
  │ ✅ Repetir decisões para confirmar entendimento        │
  │ ✅ "Vamos fazer X. Alguém discorda?" (Give room)     │
  │ ✅ Time-box investigação: "Temos 10 min. Se não       │
  │    acharmos root cause, fazemos rollback."             │
  │ ✅ Status updates a cada 15-30 min                    │
  │ ✅ Declarar claramente: "Estamos em incidente SEV1.   │
  │    Eu sou IC. [Nome] é Tech Lead. [Nome] é Comms."   │
  │                                                        │
  │ ❌ Gritar ou demonstrar pânico                        │
  │ ❌ Culpar alguém durante o incidente                  │
  │ ❌ Fazer debug E coordenar ao mesmo tempo              │
  │ ❌ Ficar em silêncio por > 15 min                     │
  │ ❌ Tomar decisão sem comunicar time                   │
  │                                                        │
  └────────────────────────────────────────────────────────┘

  FRAMEWORK DE RESPOSTA:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  1. DETECT (< 5 min)                                   │
  │     → Alarme dispara ou user reporta                    │
  │     → On-call acknowledges                              │
  │     → Severity assessment (SEV1/2/3)                    │
  │                                                         │
  │  2. MOBILIZE (< 15 min)                                │
  │     → IC designado (ou on-call assume)                  │
  │     → Roles assigned (Tech Lead, Comms, Scribe)        │
  │     → War room aberto (Slack channel, Zoom, etc.)      │
  │     → Stakeholders notificados                          │
  │                                                         │
  │  3. INVESTIGATE + MITIGATE (parallel)                   │
  │     → MITIGATE: parar o bleeding (rollback, feature    │
  │       flag off, scale up, traffic redirect)             │
  │     → INVESTIGATE: root cause (logs, metrics, traces)  │
  │     → Regra: MITIGATION > INVESTIGATION                │
  │       (stop impact first, understand later)             │
  │                                                         │
  │  4. RESOLVE                                             │
  │     → Fix deployed ou mitigated                        │
  │     → Métricas voltam ao normal                        │
  │     → Stakeholders informados: "resolved"              │
  │                                                         │
  │  5. FOLLOW-UP                                           │
  │     → Post-mortem scheduled (< 72 horas)               │
  │     → Timeline draft criado (Scribe → Author)          │
  │     → Action items tracked                              │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  DECISÕES DIFÍCEIS DURANTE INCIDENTE:

  │ Situação                           │ Decision framework     │
  │ "Rollback vs fix forward?"         │ Se fix < 15 min: fix   │
  │                                    │ Se não: rollback       │
  │ "Precisamos acordar alguém?"       │ Se SEV1: sim           │
  │                                    │ Se SEV2+: pode esperar │
  │ "Comunicar clientes?"              │ Se > 5 min impact: sim │
  │ "Escalar para management?"         │ Se SEV1 ou > 30 min   │
  │                                    │ de SEV2: sim           │
  │ "Qual time é responsável?"         │ IC decide, não debate  │
  │                                    │ agora. Ajuste depois.  │
```

---

## Facilitando Post-mortems

```
POST-MORTEM FACILITATION — A SKILL MAIS IMPORTANTE:

  TIMELINE:

  Incidente acontece → < 72h → Post-mortem escrito
                     → < 1 semana → Sessão de review (1h)
                     → < 2 semanas → Action items assigned
                     → < 30 dias → Action items completed

  PREPARAÇÃO (ANTES DA SESSÃO):

  ┌────────────────────────────────────────────────────────┐
  │ FACILITADOR (geralmente Staff+ ou Tech Lead):          │
  │                                                        │
  │ 1. Coletar dados:                                      │
  │    → Logs, métricas, deploy history                    │
  │    → Timeline do Scribe notes                          │
  │    → Slack/chat history                                │
  │                                                        │
  │ 2. Escrever draft (FACTUAL, não interpretativo):       │
  │    → Executive summary (o que aconteceu)               │
  │    → Timeline cronológico detalhado                    │
  │    → Impacto quantificado                              │
  │    → Contributing factors (não "root cause" ainda)     │
  │                                                        │
  │ 3. Enviar draft 24h antes da sessão:                   │
  │    → "Leiam antes. Na sessão vamos DISCUTIR, não       │
  │       apresentar."                                     │
  │                                                        │
  │ 4. Definir attendees:                                  │
  │    → Quem esteve envolvido no incidente                │
  │    → Quem pode contribuir para análise                 │
  │    → NÃO: management como "audiência" (intimidante)    │
  │                                                        │
  └────────────────────────────────────────────────────────┘

  NA SESSÃO (60 MINUTOS):

  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │ 0-5 MIN: GROUND RULES                                 │
  │ "Este é um post-mortem BLAMELESS.                      │
  │  Regras:                                               │
  │  → Falamos sobre SISTEMAS, não PESSOAS                │
  │  → Em vez de 'fulano errou', dizemos                  │
  │    'o sistema permitiu que X acontecesse'              │
  │  → Todos erramos. Estamos aqui para APRENDER.         │
  │  → Se alguém perceber blame, pode sinalizar."         │
  │                                                        │
  │ 5-15 MIN: TIMELINE WALKTHROUGH                        │
  │ → Facilitador walkthrough da timeline                  │
  │ → "Está faltando algo? Alguma correção?"              │
  │ → Ajustar baseado em input dos envolvidos              │
  │                                                        │
  │ 15-35 MIN: ANALYSIS (coração do post-mortem)          │
  │ → "Por que isso aconteceu?"                            │
  │ → 5 Whys ou Fault Tree Analysis                        │
  │ → Identificar CONTRIBUTING FACTORS (plural, não        │
  │   single root cause)                                   │
  │                                                        │
  │ PERGUNTAS DO FACILITADOR:                              │
  │ → "O que vocês viram que estava diferente do normal?"  │
  │ → "Se tivéssemos que reproduzir esse incidente de     │
  │    propósito, o que precisaríamos?"                    │
  │ → "Que informação teria ajudado a resolver mais rápido?"│
  │ → "Isso poderia ter acontecido antes? O que mudou?"   │
  │ → "Que defesa deveria ter pego isso?"                  │
  │                                                        │
  │ 35-50 MIN: IMPROVEMENTS                               │
  │ → "O que podemos fazer para prevenir?"                 │
  │ → "O que podemos fazer para detectar mais rápido?"     │
  │ → "O que podemos fazer para mitigar mais rápido?"      │
  │ → Para cada: viável? Quem? Quando?                     │
  │                                                        │
  │ 50-60 MIN: ACTION ITEMS                                │
  │ → Cada improvement → Action item                       │
  │ → DRI (Directly Responsible Individual) assignado      │
  │ → Deadline definido                                    │
  │ → Priorização: P0 (this sprint), P1 (next sprint),    │
  │   P2 (this quarter)                                    │
  │                                                        │
  └────────────────────────────────────────────────────────┘

  TÉCNICAS DE FACILITAÇÃO:

  SE ALGUÉM BLAMING:
  → "Entendo a frustração. Vamos focar: que condição do
     sistema fez com que essa ação tivesse esse resultado?"

  SE SILÊNCIO:
  → "Na timeline, às 14:55 houve uma decisão de rollback.
     Que informação vocês tinham nesse momento?"

  SE DEBATE TÉCNICO LONGO:
  → "Vamos capturar isso como open question e resolver
     offline. Quem pode investigar?"

  SE MANAGEMENT QUER "CULPADO":
  → (em privado) "Se punirmos, as pessoas escondem erros.
     Se aprendermos, prevenimos o próximo incidente."
```

---

## De Incidentes a Melhorias Sistêmicas

```
FRAMEWORK: INCIDENTES → PATTERNS → MELHORIAS SISTÊMICAS

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  1 incidente = 1 fix                                   │
  │  3 incidentes similares = 1 pattern                    │
  │  1 pattern = 1 melhoria sistêmica                      │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  COMO IDENTIFICAR PATTERNS:

  1. TRACK ACTION ITEMS:
     → Spreadsheet/dashboard de todos action items
     → Coluna: categoria (detection, prevention, mitigation)
     → Coluna: componente/serviço
     → Coluna: status (open, in-progress, done, won't do)

  2. REVIEW TRIMESTRAL:
     → "Quais categorias têm mais action items?"
     → "Quais serviços aparecem repetidamente?"
     → "Quantos action items foram realmente completados?"
     → Isso revela patterns sistêmicos

  EXEMPLO DE PATTERN → MELHORIA SISTÊMICA:

  ┌────────────────────────────────────────────────────────┐
  │ POST-MORTEMS DOS ÚLTIMOS 6 MESES:                     │
  │                                                        │
  │ PM-012: DB timeout → query sem index → deploy broke    │
  │ PM-015: OOM na API → memory leak → deploy broke        │
  │ PM-018: Config errada → env variable → deploy broke    │
  │ PM-021: Breaking API change → consumer failed → deploy │
  │                                                        │
  │ PATTERN: 4 incidentes ligados a DEPLOY de mudanças     │
  │ que não foram pegos antes de production.               │
  │                                                        │
  │ MELHORIA SISTÊMICA (não 4 fixes individuais):          │
  │ → Canary deployment obrigatório para todos serviços   │
  │ → Automated rollback se error rate > 1% em 5 min      │
  │ → Load test obrigatório em staging para mudanças DB   │
  │ → Contract tests entre consumer e provider             │
  │                                                        │
  │ INVESTIMENTO: 1 Staff + 2 Seniors × 2 meses           │
  │ IMPACTO: Elimina inteira CATEGORIA de incidentes       │
  └────────────────────────────────────────────────────────┘

  CATEGORIAS COMUNS DE MELHORIA:

  ═══════════════════════════════════════════════════
  DETECÇÃO (reduzir MTTD — Mean Time To Detect)
  ═══════════════════════════════════════════════════
  → Alertas mais sensíveis
  → SLO-based alerting (alert on user impact, not symptoms)
  → Anomaly detection automation
  → Health check endpoints
  → Synthetic monitoring (probes from user perspective)

  ═══════════════════════════════════════════════════
  PREVENÇÃO (reduzir FREQUÊNCIA de incidentes)
  ═══════════════════════════════════════════════════
  → Automated testing gates (CI blocks deploy)
  → Canary deployments (gradual rollout)
  → Feature flags (decouple deploy from release)
  → Load testing pre-deploy
  → Schema migration validation
  → Contract testing

  ═══════════════════════════════════════════════════
  MITIGAÇÃO (reduzir MTTR — Mean Time To Resolve)
  ═══════════════════════════════════════════════════
  → Automated rollback mechanisms
  → Runbooks (step-by-step para cenários conhecidos)
  → Circuit breakers (isolate failure)
  → Feature flags (quick disable)
  → Improved observability (faster root cause)
  → Incident response training/gamedays

  ERROR BUDGET — CONNECTING FAILURE TO VELOCITY:

  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │ SLO = 99.9% availability (target)                     │
  │ Error Budget = 100% - 99.9% = 0.1%                    │
  │ = 43.2 minutos de downtime por mês                    │
  │                                                        │
  │ SE budget > 0 (temos saldo):                           │
  │ → Deploy com confiança                                 │
  │ → Experimentar (feature flags)                         │
  │ → Velocidade > cautela                                 │
  │                                                        │
  │ SE budget ≈ 0 (consumido):                             │
  │ → Freeze de features                                   │
  │ → Foco em reliability                                  │
  │ → Apenas fixes e melhorias operacionais                │
  │                                                        │
  │ SE budget < 0 (violated):                              │
  │ → Incident review obrigatório                          │
  │ → Reliability sprint dedicado                          │
  │ → Prioridade: estabilizar antes de featurizar          │
  │                                                        │
  └────────────────────────────────────────────────────────┘
```

---

## Chaos Engineering e Proactive Failure

```
CHAOS ENGINEERING — FALHAR DE PROPÓSITO PARA NÃO FALHAR DE SURPRESA:

  PRINCÍPIO:
  "Em vez de esperar um incidente para aprender,
   INJETE falhas controladas e aprenda pro-ativamente."

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │ CHAOS MATURITY LEVELS:                                  │
  │                                                         │
  │ LEVEL 0: NENHUM                                         │
  │ → "Rezamos para não cair"                              │
  │                                                         │
  │ LEVEL 1: GAMEDAYS                                       │
  │ → Exercícios planejados de incident response            │
  │ → Manual: "E se o DB falhar agora?"                    │
  │ → Cadência: trimestral                                 │
  │                                                         │
  │ LEVEL 2: EXPERIMENTS                                    │
  │ → Testes controlados em staging/production              │
  │ → Kill container, spike latency, fill disk             │
  │ → Observar: sistema se recupera? Alerta dispara?       │
  │ → Cadência: mensal                                     │
  │                                                         │
  │ LEVEL 3: CONTINUOUS CHAOS                              │
  │ → Chaos experiments automatizados em CI/CD             │
  │ → Steady state → inject → verify → rollback            │
  │ → Cadência: continuous (every deploy)                   │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  GAMEDAY — FORMATO PRÁTICO:

  ┌────────────────────────────────────────────────────────┐
  │ GAMEDAY TEMPLATE                                       │
  │                                                        │
  │ Cenário: "DB primary falha e failover para replica"    │
  │ Objetivo: Validar que serviços resistem + alerta       │
  │           funciona + runbook está correto              │
  │                                                        │
  │ Pre-requisites:                                        │
  │ □ Stakeholders avisados                                │
  │ □ Rollback plan definido                               │
  │ □ Monitoring dashboards abertas                        │
  │ □ Time preparado (IC definido)                         │
  │                                                        │
  │ Execution (em staging ou prod com blast radius):       │
  │ 1. Capturar steady state metrics                       │
  │ 2. Injetar falha: kill DB primary                      │
  │ 3. Observar: failover acontece? Em quanto tempo?       │
  │ 4. Verificar: serviços degradam gracefully?            │
  │ 5. Verificar: alerta dispara? Em quanto tempo?         │
  │ 6. Testar runbook: seguir passos documentados         │
  │ 7. Restaurar e verificar recovery                      │
  │                                                        │
  │ Post-gameday:                                          │
  │ → O que funcionou como esperado?                       │
  │ → O que NÃO funcionou?                                │
  │ → Action items para gaps encontrados                   │
  │ → Next gameday scenario planning                       │
  └────────────────────────────────────────────────────────┘

  TIPOS DE FALHA PARA INJETAR:

  │ Tipo             │ Exemplo                │ O que testa        │
  │ Infrastructure   │ Kill instance/pod      │ Auto-scaling, HA   │
  │ Network          │ Add latency, drop pkts │ Timeouts, retries  │
  │ Dependency       │ Block external API     │ Circuit breaker    │
  │ Data             │ Corrupt/delay DB resp  │ Error handling     │
  │ Capacity         │ CPU/memory spike       │ Auto-scaling, OOM  │
  │ Configuration    │ Bad config deploy      │ Validation, rollback│
  │ Security         │ Expired cert           │ TLS handling       │
```

---

## Construindo uma Learning Organization

```
LEARNING ORGANIZATION — O OBJETIVO FINAL:

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │ LEARNING ORG = organização que SISTEMATICAMENTE         │
  │ transforma experiências (incluindo falhas) em           │
  │ conhecimento compartilhado e melhoria contínua.         │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  PRÁTICAS PARA CONSTRUIR:

  ══════════════════════════════════════════════════════════
  1. POST-MORTEM REVIEW MENSAL
  ══════════════════════════════════════════════════════════
  → 1x/mês: review de TODOS post-mortems do mês
  → Participantes: Staff+, Tech Leads, EMs
  → Buscar: patterns, action items atrasados, gaps
  → Output: quarterly reliability initiatives

  ══════════════════════════════════════════════════════════
  2. INCIDENT OF THE WEEK / LEARNING DIGEST
  ══════════════════════════════════════════════════════════
  → Weekly email/Slack: resumo de incidentes da semana
  → Incluir NEAR-MISSES (não só incidentes reais)
  → Formato: 3-5 linhas: o que, por que, o que aprendemos
  → Normalized: incidente ≠ vergonha, incidente = learning

  ══════════════════════════════════════════════════════════
  3. FAILURE FRIDAY / WHEEL OF MISFORTUNE
  ══════════════════════════════════════════════════════════
  → Simulação de incidente com role-play
  → IC rotativo (treinamento)
  → Cenário baseado em incidente real (do passado)
  → Time treina response sem pressão real
  → Google popularizou como "Wheel of Misfortune"

  ══════════════════════════════════════════════════════════
  4. NEAR-MISS REPORTING
  ══════════════════════════════════════════════════════════
  → Canal dedicado: #near-misses
  → Formato simples: "O que quase aconteceu e por quê"
  → Reward: reconhecimento público por reportar
  → Review: incluir em post-mortem review mensal

  ══════════════════════════════════════════════════════════
  5. RESILIENCE METRICS DASHBOARD
  ══════════════════════════════════════════════════════════

  │ Métrica                     │ Target     │ Trend     │
  │ # incidentes SEV1/mês       │ < 1        │ ↓         │
  │ # incidentes SEV2/mês       │ < 3        │ ↓         │
  │ MTTD (detect)               │ < 5 min    │ ↓         │
  │ MTTR (resolve)              │ < 30 min   │ ↓         │
  │ Post-mortem completion rate │ 100%       │ →         │
  │ Action item completion rate │ > 85%      │ ↑         │
  │ Error budget remaining      │ > 50%      │ →         │
  │ Gamedays completed/quarter  │ ≥ 2        │ ↑         │
  │ Near-misses reported/mês    │ ≥ 5        │ ↑ (bom!)  │
  │ Recurrence rate             │ < 10%      │ ↓         │

  MATURIDADE EM FAILURE LEADERSHIP:

  │ Nível             │ Características                    │
  │ 1. Reactivo       │ Incidentes acontecem, ninguém sabe │
  │                   │ por quê. Blame. Mesmos bugs.       │
  │ 2. Responsivo     │ Post-mortems existem mas ad-hoc.    │
  │                   │ Action items ficam open forever.    │
  │ 3. Proativo       │ Post-mortems blameless. Action items│
  │                   │ trackados. Patterns identificados.  │
  │ 4. Antecipativo   │ Gamedays regulares. Near-miss       │
  │                   │ reporting ativo. Error budgets.     │
  │ 5. Learning org   │ Continuous chaos. Failure como      │
  │                   │ input para design. Culture of       │
  │                   │ psychological safety.               │
```

---

## Papel de Cada Nível em Failure Leadership

```
COMO CADA NÍVEL LIDERA ATRAVÉS DE FALHAS:

  ══════════════════════════════════════════════════════════
  TECH LEAD — "O First Responder"
  ══════════════════════════════════════════════════════════

  → Primeiro IC (Incident Commander) do time
  → Lidera incident response dos serviços do time
  → Escreve post-mortems do time
  → Garante que action items são completados
  → Mantém runbooks atualizados
  → Treina time para on-call rotation
  
  KEY METRIC: MTTR dos serviços do time
  
  POST-MORTEM ROLE: Author ou Facilitator para 
  incidentes do time

  ══════════════════════════════════════════════════════════
  STAFF ENGINEER — "O Pattern Detector"
  ══════════════════════════════════════════════════════════

  → IC escalonado para incidentes cross-team
  → Facilita post-mortems de incidentes complexos
  → Identifica patterns entre incidentes:
    "Últimos 3 post-mortems mostram gap em contract testing"
  → Propõe melhorias sistêmicas (não apenas fixes)
  → Lidera implementação de defenses cross-team
  → Gameday designer + facilitator
  
  KEY METRIC: Recurrence rate (incidentes similares repetidos)
  
  POST-MORTEM ROLE: Facilitator + system analysis

  ══════════════════════════════════════════════════════════
  PRINCIPAL ENGINEER — "O Culture Builder"
  ══════════════════════════════════════════════════════════

  → Define incident response process org-wide
  → Review trimestral de post-mortems (patterns org-wide)
  → Propõe investimentos de reliability ao leadership
  → Error budget policy (quando freeze, quando release)
  → Chaos engineering strategy
  → Culture: fala publicamente sobre falhas próprias
    "Eu cometi erro X em 2019. Aprendemos Y."
  → Mentora Staff/TLs em facilitation skills
  
  KEY METRIC: Maturity level da learning organization
  
  POST-MORTEM ROLE: Reviewer + policy maker

  ══════════════════════════════════════════════════════════
  TECH MANAGER — "O Shield + Enabler"
  ══════════════════════════════════════════════════════════

  → Protege time de blame de management/stakeholders:
    "O time respondeu corretamente. O sistema tinha gap X.
     Aqui está nosso plano para corrigir."
  → Budget para reliability: gamedays, tools, training
  → Processo: on-call rotation justa e sustentável
  → Promo criteria inclui failure leadership
  → Comunicação: traduz post-mortem para leadership
  → Remove pressure: "Zero incidents" NÃO é goal
  → Garante psychological safety: engineers reportam 
    near-misses sem medo
  
  KEY METRIC: Psychological safety score + MTTR + action 
  item completion rate
  
  POST-MORTEM ROLE: Sponsor + stakeholder communicator
```

---

## Anti-Patterns

| Anti-pattern | Problema | Correção |
|-------------|---------|----------|
| **Blame the deployer** | "Quem fez o deploy?" como primeira pergunta | "Que condição permitiu isso?" — foco no sistema |
| **Post-mortem without actions** | "Aprendemos" mas nada muda | Todo finding → action item com DRI + deadline |
| **Action items que morrem** | 70% dos action items nunca são completados | Track em dashboard, review mensal, accountability |
| **Root cause singular** | "A causa raiz foi X" (sempre 1 coisa) | Contributing factors (plural): múltiplas defesas falharam |
| **Post-mortem só para SEV1** | Ignora SEV2/SEV3 e near-misses | Post-mortem para todo incidente, lightweight para SEV3 |
| **Zero incidents goal** | "Nossa meta é zero incidentes" | Incentiva esconder erros. Meta: rápida detecção + resolução |
| **Hero culture** | "João salvou a empresa às 3h da manhã!" | Se precisou de herói, o SISTEMA falhou |
| **Post-mortem > 2 semanas depois** | Memórias são falhas, contexto se perde | < 72h para draft, < 1 semana para sessão |
| **Management tourism** | VP assiste post-mortem como "audiência" | Se vai participar, siga as regras de blameless |
| **Chaos without safety** | Chaos experiment em prod sem rollback plan | Sempre: steady state → inject → verify → abort plan |

---

## Diretrizes para Code Review assistido por AI

Ao revisar código, considere o aspecto de failure leadership e resiliência:

1. **Error handling quality** — Erros devem ser tratados, logados com contexto, e gerar métricas; nunca silenciados
2. **Circuit breaker presence** — Chamadas a dependências externas devem ter circuit breaker ou timeout
3. **Graceful degradation** — Se dependência falha, serviço deveria degradar (funcionalidade parcial), não crash
4. **Retry com backoff** — Retries devem ter exponential backoff + jitter; retry infinito = DDoS interno
5. **Rollback capability** — Mudança deve ser reversível; se não é, precisa de flag ou blue-green strategy
6. **Feature flag** — Mudanças significativas devem ter feature flag para quick disable sem deploy
7. **Observability hooks** — Código novo deve emitir métricas, logs estruturados e traces; não só "funcionar"
8. **Runbook update** — Se comportamento do serviço muda, verificar se runbook está atualizado
9. **Health check** — Endpoints de health devem verificar dependências reais (DB, cache), não só return 200
10. **Blast radius** — Avaliar: se esse código falhar, qual o blast radius? Está limitado (timeout, isolation)?

---

## Referências

- **The Field Guide to Understanding Human Error** — Sidney Dekker (CRC Press, 2014)
- **Drift into Failure** — Sidney Dekker (Ashgate, 2011)
- **Chaos Engineering** — Casey Rosenthal & Nora Jones (O'Reilly, 2020)
- **Accelerate** — Nicole Forsgren et al. (IT Revolution, 2018) — DORA metrics
- **Site Reliability Engineering** — Google (O'Reilly, 2016) — Post-mortem culture
- **Just Culture** — Sidney Dekker (CRC Press, 2016)
- **The Fifth Discipline** — Peter Senge (Currency, 2006) — Learning organizations
- **Blameless Post-mortems at Google** — https://sre.google/sre-book/postmortem-culture/
- **Etsy: Blameless Post-mortems** — https://www.etsy.com/codeascraft/blameless-postmortems
- **Netflix Chaos Monkey** — https://netflix.github.io/chaosmonkey/
- **PagerDuty Incident Response** — https://response.pagerduty.com/
- **Learning from Incidents** — https://www.learningfromincidents.io/
- **Wheel of Misfortune** — Google — https://landing.google.com/sre/resources/practicesandprocesses/
