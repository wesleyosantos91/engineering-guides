# Feature Flags (Feature Toggles)

> **Categoria:** Estratégias de Modernização e Deploy
> **Origem:** Martin Fowler, Pete Hodgson
> **Complementa:** Canary Release, Blue-Green Deploy, Configuração Externa
> **Keywords:** feature flags, feature toggles, release toggles, trunk-based development, deploy desacoplado, ativação gradual

---

## Problema

- **Deploy ≠ Release:** Toda funcionalidade deployada vai para produção instantaneamente
- **Branches de longa duração:** Feature branches conflitam com trunk-based development
- **Rollback = redeploy:** Se a feature tiver bug, precisa de novo deploy para reverter
- **Tudo ou nada:** Não é possível habilitar para um grupo de usuários e desabilitar para outros
- **Coordenação:** Marketing quer lançar na terça, ops quer deployar na segunda

---

## Solução

**Separar o deploy do release.** Uma feature flag é uma **condição no código** que habilita ou desabilita uma funcionalidade em runtime, sem redeploy:

```
if featureFlags.isEnabled("new-checkout"):
    return newCheckoutFlow(request)
else:
    return legacyCheckoutFlow(request)
```

O código é deployado com a flag **desabilitada**. Quando quisermos liberar, **habilitamos a flag** — sem deploy.

---

## Tipos de Feature Flags

| Tipo | Duração | Quem controla | Exemplo |
|------|---------|-------------|---------|
| **Release Toggle** | Temporário (semanas) | Dev/PM | Nova feature em desenvolvimento |
| **Experiment Toggle** | Temporário (dias-semanas) | PM/Data | A/B test: botão azul vs. verde |
| **Ops Toggle** | Permanente ou longo prazo | Ops/SRE | Desabilitar feature pesada em pico |
| **Permission Toggle** | Permanente | Produto | Feature apenas para plano Premium |
| **Kill Switch** | Permanente | Ops/SRE | Desabilitar dependência não-essencial em falha |

---

## Fluxo

```
1. Developer implementa feature com flag
2. Deploy para produção (flag DESABILITADA)
3. QA valida em produção com flag habilitada para ambiente de teste
4. PM decide habilitar para 10% dos usuários (canary)
5. Monitora métricas
6. PM habilita para 100% (release)
7. Developer remove a flag e o código antigo (cleanup)
```

---

## Estratégias de Avaliação

### 1. Boolean (Liga/Desliga)

```
featureFlags.isEnabled("dark-mode")  → true / false
```

### 2. Por Usuário

```
featureFlags.isEnabled("new-dashboard", userId: "user-123")

Regras:
  - user-123 → true (beta tester)
  - user-456 → false (não liberado)
```

### 3. Por Porcentagem (Gradual Rollout)

```
featureFlags.isEnabled("new-checkout", userId: user.id)

Regra: 20% dos usuários
  - hash(user.id) % 100 < 20 → true
  - hash(user.id) % 100 >= 20 → false

// O MESMO usuário sempre recebe a mesma decisão (consistent hashing)
```

### 4. Por Segmento

```
featureFlags.isEnabled("premium-feature", context: {
    plan: user.plan,         // "PRO"
    country: user.country,   // "BR"
    role: user.role           // "ADMIN"
})

Regras:
  - plan == "PRO" AND country == "BR" → true
  - plan == "FREE" → false
```

### 5. Por Ambiente

```
featureFlags.isEnabled("debug-panel", environment: "staging")

Regras:
  - environment == "production" → false
  - environment == "staging" → true
```

---

## Arquitetura

```
┌──────────────┐     ┌─────────────────────────┐
│   Serviço    │◀────│  Feature Flag Service    │
│              │     │  (LaunchDarkly, Unleash, │
│  if flag:    │     │   Flagsmith, custom)     │
│    newFlow() │     │                          │
│  else:       │     │  Dashboard:              │
│    oldFlow() │     │  [✓] new-checkout: ON    │
│              │     │  [ ] dark-mode: OFF      │
└──────────────┘     └─────────────────────────┘
                              │
                     ┌────────▼────────┐
                     │  Config Store   │
                     │  (DB, Redis,    │
                     │   API, file)    │
                     └─────────────────┘
```

### Avaliação: Server-side vs. Client-side

| Abordagem | Descrição |
|-----------|-----------|
| **Server-side** | Serviço consulta o flag service a cada request (ou com cache local) |
| **Client-side (SDK)** | SDK no client faz polling/streaming das flags; avaliação local |

```
Server-side:
  Serviço → flag-service.isEnabled("feat", userContext) → true/false

Client-side (SDK com cache):
  SDK.init() → carrega todas as flags em memória
  SDK.isEnabled("feat", userContext) → avaliação local (sem roundtrip)
  SDK → streaming/polling → atualiza cache quando flag muda
```

---

## Kill Switch

Um caso especial de feature flag: o **kill switch** permite desabilitar rapidamente uma funcionalidade que está causando problemas:

```
// Kill switch para dependência não-essencial
function getRecommendations(userId):
    if featureFlags.isEnabled("recommendations-enabled"):
        return recommendationService.getFor(userId)
    else:
        return []  // fallback — dependência desabilitada

// Em caso de incidente:
// Dashboard: recommendations-enabled → OFF
// → Instantaneamente, recomendações param sem redeploy
```

---

## Ciclo de Vida da Flag

```
┌── Criar ──▶ Código com flag ──▶ Deploy (flag OFF) ──▶ Testar ──┐
│                                                                  │
│  ┌── Gradual rollout (10% → 50% → 100%) ──▶ Flag ON para todos │
│  │                                                               │
│  └──── Monitorar ──▶ Se OK: REMOVER flag + código antigo ───────│
│                      Se NOK: Flag OFF (rollback instantâneo)     │
│                                                                  │
└── CLEANUP: Remover flag do código após release completo ─────────┘
```

**Atenção ao cleanup:** Flags não removidas acumulam **dívida técnica** (tech debt).

---

## Exemplo Conceitual (Pseudocódigo)

```
// Uso básico
class CheckoutController:
    featureFlags
    
    function checkout(request, user):
        if featureFlags.isEnabled("new-checkout-flow", context: user):
            return newCheckoutService.process(request)
        else:
            return legacyCheckoutService.process(request)

// Kill switch
class ProductController:
    featureFlags
    
    function getProduct(productId):
        product = productService.findById(productId)
        
        // Kill switch para recomendações
        if featureFlags.isEnabled("show-recommendations"):
            product.recommendations = recommendationService.getFor(productId)
        else:
            product.recommendations = []  // degradação graceful
        
        return product

// Gradual rollout
class FeatureFlagEvaluator:
    function isEnabled(flagName, context):
        flag = flagStore.get(flagName)
        
        if flag.type == "boolean":
            return flag.enabled
        
        if flag.type == "percentage":
            // Consistent: mesmo user sempre recebe mesmo resultado
            bucket = hash(flagName + context.userId) % 100
            return bucket < flag.percentage
        
        if flag.type == "segment":
            return flag.rules.any(rule -> rule.matches(context))
```

---

## Boas Práticas para Cleanup

| Prática | Descrição |
|---------|-----------|
| **TTL na flag** | Defina data de expiração ao criar a flag |
| **Alertas de stale flags** | Alerte quando flag ultrapassar o TTL sem ser removida |
| **Ticket de cleanup** | Ao criar flag, crie também um ticket para remover |
| **Dashboard de flags** | Visibilidade de todas as flags ativas, stales, por serviço |
| **Code review** | Não aprove PR que adiciona flag sem ticket de remoção |

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Flags que nunca são removidas | Código cheio de condicionais mortos — tech debt | TTL + ticket de cleanup ao criar |
| Flag dentro de flag | `if (A && B && !C)` — complexidade exponencial | Mantenha flags simples e independentes |
| Toda lógica em flags | Código vira "spaghetti de flags" | Flags para release/ops; não para lógica de negócio permanente |
| Sem dashboard/visibilidade | Ninguém sabe quais flags existem e seus estados | Dashboard centralizado |
| Flag com side effects | Habilitar/desabilitar causa inconsistência de dados | Flags: fluxo, não estado. Use migration para dados. |
| Teste só com flag ON | Bug aparece quando flag é OFF (código legado) | Teste ambos os caminhos |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Canary Release** | Feature flag pode ser o mecanismo do canary (gradual rollout %) |
| **Blue-Green** | Flag pode controlar o "switch" entre versões |
| **Circuit Breaker** | Kill switch é um CB manual para funcionalidades |
| **Configuração Externa** | Feature flags são uma forma de configuração dinâmica |
| **Strangler Fig** | Flags controlam quais funcionalidades usam novo vs. legado |
| **A/B Testing** | Experiment toggles habilitam A/B test nativo |

---

## Boas Práticas

1. **Deploy ≠ Release** — deploye com flag OFF, habilite quando quiser.
2. Defina **TTL** para cada flag — flag sem data de remoção vira tech debt.
3. Use **gradual rollout** (1% → 10% → 50% → 100%) para features de risco.
4. Implemente **kill switches** para dependências não-essenciais.
5. **Teste ambos os caminhos** (flag ON e OFF) nos testes automatizados.
6. Mantenha um **dashboard** centralizado de todas as flags.
7. Crie **ticket de cleanup** ao criar a flag — não a deixe permanente.
8. Use consistent hashing para que o **mesmo usuário sempre receba a mesma experiência**.

---

## Referências

- Pete Hodgson — [Feature Toggles (aka Feature Flags)](https://martinfowler.com/articles/feature-toggles.html) (martinfowler.com)
- Martin Fowler — [Feature Toggle](https://martinfowler.com/bliki/FeatureToggle.html)
- LaunchDarkly — [Feature Flag Best Practices](https://launchdarkly.com/blog/feature-flag-best-practices/)
- Unleash — [Feature Toggle Types](https://docs.getunleash.io/reference/feature-toggle-types)
