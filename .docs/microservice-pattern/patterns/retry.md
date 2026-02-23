# Retry com Backoff Exponencial + Jitter

> **Categoria:** Resiliência
> **Complementa:** Circuit Breaker, Timeout

---

## Problema

Falhas **transitórias** são comuns em sistemas distribuídos: um timeout de rede que dura 200ms, um HTTP 503 porque a dependência estava reiniciando, um connection reset momentâneo. Essas falhas se resolvem sozinhas em segundos ou minutos.

Sem retry, cada falha transitória é tratada como erro definitivo, degradando desnecessariamente a experiência do usuário ou perdendo mensagens.

---

## Solução

Reexecutar automaticamente a operação que falhou, com intervalos crescentes entre as tentativas (**exponential backoff**) e variação aleatória (**jitter**) para evitar que múltiplas instâncias façam retry simultaneamente.

---

## Conceitos Fundamentais

### Exponential Backoff

Em vez de esperar um intervalo fixo entre retries, o tempo de espera **dobra** a cada tentativa:

```
Tentativa 1: falha → espera 500ms
Tentativa 2: falha → espera 1000ms  (500ms × 2)
Tentativa 3: falha → espera 2000ms  (1000ms × 2)
Tentativa 4: falha → espera 4000ms  (2000ms × 2)
```

**Por quê?** Se a dependência está sobrecarregada, retry imediato piora a situação. Backoff dá tempo para recuperação.

### Jitter (Variação Aleatória)

Adiciona um fator aleatório ao intervalo de espera. Sem jitter, se 100 instâncias falharem ao mesmo tempo, todas farão retry ao mesmo tempo, criando um **thundering herd** — uma avalanche de requisições que sobrecarrega a dependência novamente.

```
Sem jitter:  todas as 100 instâncias → retry em 500ms, 1000ms, 2000ms (picos)
Com jitter:  instância A → 350ms, 870ms, 1600ms
             instância B → 620ms, 1200ms, 2300ms
             instância C → 480ms, 960ms, 1800ms
             → carga distribuída no tempo
```

**Tipos de Jitter:**

| Tipo | Fórmula | Descrição |
|------|---------|-----------|
| **Full Jitter** | `random(0, baseDelay × 2^attempt)` | Máxima dispersão — recomendado na maioria dos casos |
| **Equal Jitter** | `baseDelay × 2^attempt / 2 + random(0, baseDelay × 2^attempt / 2)` | Metade fixa + metade aleatória |
| **Decorrelated Jitter** | `min(cap, random(baseDelay, previousDelay × 3))` | Baseado no delay anterior — boa distribuição |

### Max Attempts (Tentativas Máximas)

Número total de execuções (1 chamada original + N retries). **Nunca** retente infinitamente — defina um limite baseado no SLA da dependência.

| Cenário | Max Attempts Típico |
|---------|-------------------|
| Chamadas HTTP a outro microsserviço | 3–4 |
| Envio de email/notificação | 4–5 |
| Publicação em fila de mensagens | 3 |
| Escrita em banco replicado | 2–3 |

### Max Delay / Cap

Limite máximo do intervalo de espera. Sem cap, o backoff exponencial cresce indefinidamente (500ms → 1s → 2s → 4s → 8s → 16s → ...). O cap define o teto:

```
Com cap de 5s:  500ms → 1s → 2s → 4s → 5s → 5s → 5s
```

---

## Quais erros devem ser retentados?

| DEVE retentar (transitório) | NÃO DEVE retentar (permanente) |
|----------------------------|-------------------------------|
| IOException (falha de rede) | HTTP 400 (Bad Request) |
| HTTP 503 (Service Unavailable) | HTTP 404 (Not Found) |
| HTTP 502 (Bad Gateway) | HTTP 409 (Conflict — estado inconsistente) |
| HTTP 429 (Too Many Requests) | Erro de validação de negócio |
| Connection timeout / reset | Erro de desserialização |
| Database connection timeout | Erro de autenticação (401) |
| Deadlock no banco | Erro de autorização (403) |

**Regra:** Se repetir a mesma chamada com os mesmos dados **pode** produzir resultado diferente, é transitório → retente. Se vai **sempre** falhar da mesma forma, é permanente → não retente.

---

## Diagrama de Fluxo

```
Chamada ao serviço
       │
       ▼
  ┌── Sucesso? ──┐
  │              │
  │ SIM          │ NÃO
  │              │
  ▼              ▼
Retorna OK   Erro é transitório?
                  │              │
                  │ SIM          │ NÃO
                  │              │
                  ▼              ▼
           Tentativas           Propaga erro
           esgotadas?          (sem retry)
           │         │
           │ NÃO     │ SIM
           │         │
           ▼         ▼
   Espera (backoff  Fallback ou
   + jitter) e      propaga erro
   tenta novamente   final
```

---

## Exemplo Conceitual (Pseudocódigo)

```
function callWithRetry(operation, config):
    attempts = 0
    lastException = null
    
    while attempts < config.maxAttempts:
        try:
            return operation.execute()
        catch exception:
            if not isRetryable(exception):
                throw exception  // erro permanente — não retenta
            
            lastException = exception
            attempts++
            
            if attempts < config.maxAttempts:
                delay = calculateDelay(attempts, config)
                sleep(delay)
    
    // Todas as tentativas falharam
    throw RetryExhaustedException(lastException)

function calculateDelay(attempt, config):
    // Backoff exponencial
    exponentialDelay = config.baseDelay × (config.multiplier ^ attempt)
    
    // Cap no máximo
    cappedDelay = min(exponentialDelay, config.maxDelay)
    
    // Jitter (±50%)
    jitter = cappedDelay × random(0.5, 1.5)
    
    return jitter
```

---

## Retry + Idempotência

Retry **exige** que a operação seja **idempotente** ou que o chamador use uma **idempotency key**:

```
Cenário perigoso:
  POST /v1/payments → timeout (mas o pagamento FOI processado!)
  Retry POST /v1/payments → COBRA DUAS VEZES

Cenário seguro:
  POST /v1/payments (Idempotency-Key: abc-123) → timeout
  Retry POST /v1/payments (Idempotency-Key: abc-123) → servidor detecta duplicata → retorna resultado original
```

Se a operação **não é idempotente** e não suporta idempotency key, **não use retry automático** — o custo do efeito duplicado pode ser pior que a falha.

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Retry sem backoff | Intervalo fixo sobrecarrega a dependência | Use backoff exponencial |
| Retry sem jitter | Thundering herd — todos retentam ao mesmo tempo | Adicione jitter (±50% mínimo) |
| Retry em erros permanentes | Retenta 404, 400, validação — nunca vai funcionar | Configure lista de exceções retentáveis |
| Retry infinito | Consumer nunca avança; bloqueia processamento | Defina max-attempts e fallback (DLQ, log) |
| Retry sem idempotência | Operações são executadas múltiplas vezes (cobrança dupla) | Garanta idempotência antes de habilitar retry |
| Retry acima do circuit breaker | Retenta mesmo quando a dependência está claramente down | Retry deve ficar **dentro** do Circuit Breaker |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Circuit Breaker** | Retry fica **dentro** do CB. Se todas as tentativas falharem, conta como 1 falha no CB. |
| **Timeout** | Cada tentativa deve ter seu próprio timeout. O retry reexecuta após timeout. |
| **Idempotência** | Retry exige que operações sejam idempotentes para evitar duplicação. |
| **DLQ** | Quando todas as tentativas falham, a mensagem vai para a DLQ para análise. |
| **Backpressure** | Se a dependência retorna 429 (rate limited), respeite o Retry-After header. |

---

## Boas Práticas

1. **Sempre** use backoff exponencial — retry com intervalo fixo pode piorar a situação.
2. **Sempre** adicione jitter para evitar thundering herd.
3. Configure **ignore-exceptions** para erros **permanentes** (400, 404, validação).
4. Defina `max-attempts` de acordo com o SLA da dependência.
5. Combine com Circuit Breaker: o Retry fica **dentro** do Circuit Breaker.
6. Toda operação sujeita a retry deve ser **idempotente**.
7. Logue cada retry com nível WARN — para visibilidade e diagnóstico.
8. Após esgotar retries, envie para **DLQ** ou **fallback** — não perca a operação silenciosamente.

---

## Referências

- AWS Architecture Blog — [Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- Microsoft — [Retry Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry)
- Marc Brooker — [Jitter: Making Things Better With Randomness](https://brooker.co.za/blog/)
