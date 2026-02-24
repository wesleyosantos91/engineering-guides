# Strangler Fig Pattern

> **Categoria:** Estratégias de Modernização e Deploy
> **Origem:** Martin Fowler (inspirado na Figueira Estranguladora)
> **Complementa:** CDC, Blue-Green Deploy, API Gateway, Feature Flags
> **Keywords:** strangler fig, migração incremental, modernização de legado, decomposição de monolito, routing facade

---

## Problema

Substituir um **sistema legado** de uma vez (big-bang rewrite) é extremamente arriscado:

- O legado funciona (negócio depende dele)
- Reescrever tudo leva meses/anos — sem valor entregue até o final
- Bugs e edge cases do legado não são conhecidos até serem encontrados
- Risco de o novo sistema ter problemas que o legado não tinha
- Se o novo sistema falhar, não há como voltar atrás

---

## Solução

Migrar **incrementalmente**, funcionalidade por funcionalidade. O novo sistema "cresce ao redor" do legado, absorvendo cada parte gradualmente. Quando todas as funcionalidades estiverem migradas, o legado é desligado.

A metáfora vem da **Figueira Estranguladora**: uma planta que cresce ao redor de uma árvore, absorvendo luz e nutrientes, até que a árvore original morre e a figueira se sustenta sozinha.

---

## Fases

### Fase 1: Proxy (Interceptar)

Introduzir um **proxy/facade** na frente do legado. Inicialmente, 100% do tráfego é encaminhado para o legado:

```
                    ┌──────────────┐
 Clients ──────────▶│    Proxy     │──────────▶ Legado (100%)
                    │  (Gateway)   │
                    └──────────────┘
```

### Fase 2: Migrar Funcionalidade por Funcionalidade

Implementar cada funcionalidade no novo sistema. O proxy roteia o tráfego para o novo sistema para funcionalidades já migradas:

```
                    ┌──────────────┐     ┌──────────────┐
 Clients ──────────▶│    Proxy     │────▶│ Novo Sistema │  (funcionalidades migradas)
                    │  (Gateway)   │     └──────────────┘
                    │              │
                    │              │────▶ Legado          (funcionalidades não migradas)
                    └──────────────┘
```

### Fase 3: Concluir a Migração

Todas as funcionalidades migradas. Legado desligado:

```
                    ┌──────────────┐     ┌──────────────┐
 Clients ──────────▶│    Proxy     │────▶│ Novo Sistema │  (100%)
                    │  (Gateway)   │     └──────────────┘
                    └──────────────┘
                    
                    Legado: DESLIGADO 🎉
```

---

## Fluxo Detalhado

```
Etapa 1: Identificar uma funcionalidade isolada
  Ex: Módulo de "Relatórios" no monolito

Etapa 2: Implementar no novo sistema
  Novo microsserviço: "report-service"
  Testar independentemente

Etapa 3: Roteamento no proxy
  /api/reports/** → report-service (novo)
  /** (tudo mais) → legado

Etapa 4: Validar em produção
  Comparar resultados do novo vs. legado (shadow mode)
  Monitorar erros, latência, corretude

Etapa 5: Migrar próxima funcionalidade
  Repetir para o próximo módulo

Etapa 6: Após todas as funcionalidades:
  Desligar o legado
```

---

## Estratégias de Migração de Dados

Um dos maiores desafios é **migrar dados** sem perder consistência:

### Opção 1: Banco Compartilhado (temporário)

```
Novo sistema ──┐
               ├──▶ Mesmo Banco (do legado)
Legado ────────┘

Vantagem: Sem migração de dados
Desvantagem: Acoplamento ao schema do legado
```

### Opção 2: Banco Separado + Sync

```
Legado ──▶ Banco Legado ──CDC──▶ Banco Novo ◀── Novo Sistema
```

CDC sincroniza dados do legado para o novo banco em tempo real.

### Opção 3: Migração Gradual

```
Fase 1: Novo sistema lê do banco legado + escreve no novo
Fase 2: Novo sistema lê e escreve apenas no novo
Fase 3: Desliga acesso ao banco legado
```

---

## Critérios para Escolher Funcionalidades

| Prioridade | Critério | Justificativa |
|-----------|----------|---------------|
| 1ª | **Menor dependência** com outras partes do legado | Mais fácil de extrair |
| 2ª | **Maior valor de negócio** | ROI rápido justifica o investimento |
| 3ª | **Mais bugs/problemas** no legado | Migrar resolve dores existentes |
| 4ª | **Mais mudanças solicitadas** | Área com mais changes = mais benefício |
| 5ª | **Melhor definida** (bounded context claro) | Menos ambiguidade na migração |

---

## Exemplo de Roteamento no Proxy (Pseudocódigo)

```
// Proxy/Gateway — roteamento por funcionalidade migrada
class StranglerProxy:
    migratedRoutes = {
        "/api/reports/**": "report-service",
        "/api/notifications/**": "notification-service",
        "/api/users/profile/**": "user-service"
    }
    legacyUrl = "http://legacy-monolith:8080"
    
    function route(request):
        for pattern, service in migratedRoutes:
            if request.path matches pattern:
                return forwardTo(service, request)
        
        // Não migrado → legado
        return forwardTo(legacyUrl, request)
```

---

## Validação: Shadow Mode / Dark Launch

Antes de rotear tráfego para o novo sistema, valide com **shadow mode**:

```
Client → Proxy → Legado (responde ao client)
                    │
                    └──▶ Novo Sistema (shadow — resultado descartado)

Comparação automática:
  Se legado.response != novo.response:
      log.warn("Divergência detectada!")
      metrics.increment("shadow.divergence")
```

Isso permite validar que o novo sistema produz os mesmos resultados antes de rotear tráfego real.

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Big-bang rewrite | Risco altíssimo; meses sem entregar valor | Migre incrementalmente |
| Migrar tudo de uma vez | Mesmo que big-bang rewrite | Uma funcionalidade por vez |
| Não validar em produção | Bugs descobertos muito tarde | Shadow mode + canary |
| Acoplamento de dados permanente | Novo sistema preso ao schema do legado | Planeje migração de dados desde o início |
| Sem proxy/gateway | Clients precisam saber se é legado ou novo | Proxy abstraí — client não sabe |
| Legado nunca é desligado | Dois sistemas rodando indefinidamente (custo) | Defina deadline para desligar legado |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **API Gateway** | O gateway é o proxy natural para o Strangler Fig |
| **CDC** | Sincroniza dados entre banco legado e novo banco |
| **Feature Flags** | Flags controlam qual sistema atende cada funcionalidade |
| **Blue-Green** | Pode ser combinado: legado=blue, novo=green (por funcionalidade) |
| **Canary Release** | Tráfego gradual para o novo sistema (10% → 50% → 100%) |
| **Shadow Deploy** | Valida o novo sistema sem afetar produção |

---

## Boas Práticas

1. **Comece pelo proxy/gateway** — intercepte o tráfego antes de migrar qualquer coisa.
2. Migre funcionalidades **isoladas e de alto valor** primeiro.
3. Use **shadow mode** para validar antes de rotear tráfego real.
4. Planeje a **migração de dados** desde o início.
5. Defina **métricas de sucesso** para cada funcionalidade migrada.
6. Tenha **rollback** rápido — se o novo falhar, o proxy volta para o legado.
7. Defina um **prazo para desligar** o legado — evite dois sistemas indefinidamente.
8. Trate o legado com respeito — ele funciona e o negócio depende dele.

---

## Referências

- Martin Fowler — [Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html)
- Microsoft — [Strangler Fig Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/strangler-fig)
- Sam Newman — *Monolith to Microservices* (O'Reilly) — Capítulo 3: Splitting the Monolith
