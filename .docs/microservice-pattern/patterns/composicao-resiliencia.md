# Composição de Padrões de Resiliência

> **Categoria:** Resiliência
> **Pré-requisitos:** Circuit Breaker, Retry, Timeout, Rate Limiter, Bulkhead
> **Keywords:** resiliência, composição de padrões, ordenação, camadas de proteção, combinação de resiliência

---

## Problema

Cada padrão de resiliência resolve um problema específico. Nenhum deles sozinho protege completamente uma chamada remota. A questão é: **como combiná-los?** Em qual **ordem** devem ser aplicados?

---

## Solução

Compor os padrões em camadas, cada um atuando em um nível diferente de proteção. A **ordem de execução** importa — padrões externos "envolvem" padrões internos.

---

## Ordem de Composição (de fora para dentro)

```
┌─────────────────────────────────────────────────────┐
│  1. Retry                                           │
│  ┌──────────────────────────────────────────────┐   │
│  │  2. Circuit Breaker                          │   │
│  │  ┌───────────────────────────────────────┐   │   │
│  │  │  3. Rate Limiter                      │   │   │
│  │  │  ┌────────────────────────────────┐   │   │   │
│  │  │  │  4. Timeout (Time Limiter)     │   │   │   │
│  │  │  │  ┌─────────────────────────┐   │   │   │   │
│  │  │  │  │  5. Bulkhead            │   │   │   │   │
│  │  │  │  │  ┌──────────────────┐   │   │   │   │   │
│  │  │  │  │  │  Chamada real    │   │   │   │   │   │
│  │  │  │  │  └──────────────────┘   │   │   │   │   │
│  │  │  │  └─────────────────────────┘   │   │   │   │
│  │  │  └────────────────────────────────┘   │   │   │
│  │  └───────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

**Ordem de execução:**

```
Retry → Circuit Breaker → Rate Limiter → Timeout → Bulkhead → chamada real
```

---

## Por que essa ordem?

| Posição | Padrão | Justificativa |
|---------|--------|--------------|
| **1. Retry** (mais externo) | Retry envolve tudo. Se a chamada inteira falhar (incluindo circuit breaker abrir), o retry pode tentar novamente. |
| **2. Circuit Breaker** | Antes de cada tentativa (incluindo retries), verifica se o circuito está aberto. Se está, falha imediato sem consumir slots do bulkhead. |
| **3. Rate Limiter** | Controla a taxa de chamadas que passam para a dependência. Se o limite foi atingido, rejeita sem consumir timeout ou bulkhead. |
| **4. Timeout** | Garante que a chamada não demore indefinidamente. Se estourar, libera o slot do bulkhead mais rápido. |
| **5. Bulkhead** (mais interno) | Limita concorrência real na dependência. É o último portão antes da chamada. |

---

## Fluxo Completo — Caminho Feliz

```
Request
  │
  ├── Retry: tentativa 1
  │     │
  │     ├── Circuit Breaker: circuito CLOSED → permite
  │     │     │
  │     │     ├── Rate Limiter: dentro do limite → permite
  │     │     │     │
  │     │     │     ├── Timeout: inicia timer (5s)
  │     │     │     │     │
  │     │     │     │     ├── Bulkhead: slot disponível → adquire
  │     │     │     │     │     │
  │     │     │     │     │     └── Chamada real → SUCESSO (200ms)
  │     │     │     │     │          │
  │     │     │     │     │     Bulkhead: libera slot
  │     │     │     │     │
  │     │     │     │     Timeout: OK (200ms < 5s)
  │     │     │     │
  │     │     │     Rate Limiter: decrementa contador
  │     │     │
  │     │     Circuit Breaker: registra SUCESSO
  │     │
  │     Retry: não precisa retentar
  │
  └── Retorna resposta
```

---

## Fluxo Completo — Falha com Recovery

```
Request
  │
  ├── Retry: tentativa 1
  │     │
  │     ├── Circuit Breaker: CLOSED → permite
  │     │     │
  │     │     ├── Rate Limiter: OK
  │     │     │     │
  │     │     │     ├── Timeout: inicia timer
  │     │     │     │     │
  │     │     │     │     ├── Bulkhead: slot OK
  │     │     │     │     │     │
  │     │     │     │     │     └── Chamada real → TIMEOUT (5s)
  │     │     │     │     │
  │     │     │     │     Timeout: ESTOURO → TimeoutException
  │     │     │     │     Bulkhead: libera slot
  │     │     │
  │     │     Circuit Breaker: registra FALHA (5/10 = 50%)
  │     │
  │     Retry: espera backoff + jitter → tentativa 2
  │
  ├── Retry: tentativa 2
  │     │
  │     ├── Circuit Breaker: 50% falhas → OPEN!
  │     │     │
  │     │     └── Reject imediato (CallNotPermittedException)
  │     │
  │     Retry: espera backoff + jitter → tentativa 3
  │
  ├── Retry: tentativa 3
  │     │
  │     ├── Circuit Breaker: ainda OPEN
  │     │     │
  │     │     └── Reject imediato
  │     │
  │     Retry: tentativas esgotadas
  │
  └── Fallback → "Serviço indisponível"
```

---

## Decisão: Quais padrões usar?

Nem toda chamada precisa de todos os padrões. Use a tabela para decidir:

| Cenário | Retry | Circuit Breaker | Rate Limiter | Timeout | Bulkhead |
|---------|-------|----------------|-------------|---------|----------|
| Chamada HTTP a outro microsserviço | ✅ | ✅ | ❌ (no Gateway) | ✅ | ✅ |
| Endpoint público exposto | ❌ | ❌ | ✅ | ✅ | ❌ |
| Chamada a API de terceiros (pagamento) | ✅ | ✅ | ❌ | ✅ | ✅ |
| Processamento de mensagem (Kafka) | ✅ (via DLQ) | ❌ | ❌ | ✅ | ❌ |
| Query ao banco de dados | ❌ (ou 1 retry) | ❌ | ❌ | ✅ | ❌ |
| Chamada a serviço de relatórios (lento) | ✅ | ✅ | ❌ | ✅ (longo) | ✅ (restrito) |

---

## Configuração Conceitual por Dependência

### Serviço de Pagamento (crítico, baixa latência)

| Padrão | Configuração | Justificativa |
|--------|-------------|--------------|
| Retry | 3 tentativas, backoff 500ms×2 | Falhas transitórias são comuns |
| Circuit Breaker | 50% failure rate, janela de 10 | Agressivo — pagamento é crítico |
| Timeout | 3s | Pagamento deve ser rápido |
| Bulkhead | 10 chamadas concorrentes | Isola de outras dependências |

### Serviço de Notificação (não-crítico, tolerante)

| Padrão | Configuração | Justificativa |
|--------|-------------|--------------|
| Retry | 5 tentativas, backoff 1s×2 | Mais tolerante — pode esperar |
| Circuit Breaker | 70% failure rate, janela de 20 | Mais permissivo |
| Timeout | 10s | Envio de email pode ser lento |
| Bulkhead | 15 chamadas concorrentes | Não é gargalo |

### Serviço de Relatórios (lento, pesado)

| Padrão | Configuração | Justificativa |
|--------|-------------|--------------|
| Retry | 2 tentativas, backoff 2s×2 | Poucos retries (operação pesada) |
| Circuit Breaker | 50% failure rate | Padrão |
| Timeout | 30s | Relatórios são lentos por natureza |
| Bulkhead | 5 chamadas concorrentes | **Muito restrito** — protege recursos |

---

## Antipadrões de Composição

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Retry **sem respeitar** o Circuit Breaker | Retry continua tentando mesmo quando o CB está aberto — desperdício de recursos | Retry envolve o CB (Retry → CB); quando o CB rejeita fast, o Retry conta como tentativa falhada e respeita o maxRetries |
| Timeout **fora** do Bulkhead | Timeout não libera o slot do bulkhead quando estoura | Timeout deve ser externo ao Bulkhead |
| Rate Limiter dentro do Retry | Cada retry conta como nova requisição no rate limiter | Rate Limiter deve ficar entre CB e Timeout |
| Todos os padrões em tudo | Overhead e complexidade desnecessários | Aplique apenas os padrões relevantes p/ cada caso |
| Mesma config para toda dependência | Pagamento com mesma tolerância que notificação | Customize por dependência — cada uma tem SLA diferente |

---

## Boas Práticas

1. **Ordem importa:** Retry → Circuit Breaker → Rate Limiter → Timeout → Bulkhead → chamada.
2. **Menos é mais:** Nem toda chamada precisa de todos os 5 padrões.
3. **Customize por dependência:** Cada serviço externo tem SLA e criticidade diferentes.
4. **Teste a composição:** Simule falhas em cascata e valide que os padrões interagem corretamente.
5. **Monitore cada camada:** Exponha métricas de cada padrão individualmente.
6. **Documente a configuração:** Explique por que cada parâmetro foi escolhido para cada dependência.

---

## Referências

- Resilience4j — [Getting Started: Ordering of Decorators](https://resilience4j.readme.io/docs/getting-started-3)
- Microsoft — [Resiliency Patterns](https://learn.microsoft.com/en-us/azure/architecture/patterns/category/resiliency)
