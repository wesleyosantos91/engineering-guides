# Circuit Breaker

> **Categoria:** Resiliência
> **Origem:** Michael Nygard — *Release It!* (2007)

---

## Problema

Uma dependência externa (outro microsserviço, banco de dados, API de terceiros) está falhando. Cada chamada gera timeout ou erro, consumindo threads, conexões e memória. O acúmulo de chamadas com falha degrada **todo o sistema** — não apenas a funcionalidade que usa essa dependência.

Sem proteção, uma única dependência lenta pode causar uma **falha em cascata**: o serviço A espera o serviço B (que está lento), esgota seu pool de threads, e passa a não atender nem mesmo requisições que não dependem de B.

---

## Solução

O **Circuit Breaker** atua como um disjuntor elétrico. Ele monitora as chamadas a uma dependência e, quando detecta que a taxa de falhas ultrapassou um limiar, **abre o circuito** — rejeitando imediatamente toda nova chamada sem nem tentar executá-la (*fail-fast*).

Após um período de espera, o circuito entra em modo de teste (*half-open*), permitindo um número limitado de chamadas para verificar se a dependência se recuperou. Se as chamadas de teste forem bem-sucedidas, o circuito fecha e o tráfego normal é restaurado. Se falharem, o circuito reabre.

---

## Estados do Circuit Breaker

```
CLOSED ──(falhas > threshold)──▶ OPEN ──(wait-duration)──▶ HALF_OPEN
   ▲                                                          │
   └──────────(chamadas de teste OK)───────────────────────────┘
                                                               │
   OPEN ◀──────(chamadas de teste FAIL)────────────────────────┘
```

| Estado | Comportamento |
|--------|---------------|
| **CLOSED** | Chamadas passam normalmente. O circuit breaker monitora e contabiliza falhas dentro de uma janela (por contagem ou por tempo). |
| **OPEN** | Todas as chamadas são **rejeitadas imediatamente** (fail-fast). Nenhuma tentativa é feita. Um fallback pode ser retornado. |
| **HALF_OPEN** | Permite N chamadas de teste. Se a maioria tiver sucesso → volta para CLOSED. Se falhar → volta para OPEN. |

---

## Conceitos Fundamentais

### Sliding Window (Janela Deslizante)

O circuit breaker avalia falhas dentro de uma **janela deslizante**:

- **COUNT_BASED:** Avalia as últimas N chamadas. Ex: das últimas 10 chamadas, se 5 falharam (50%), abre o circuito.
- **TIME_BASED:** Avalia chamadas nos últimos N segundos. Ex: nos últimos 60 segundos, se 50% das chamadas falharam, abre o circuito.

### Failure Rate Threshold

Percentual de falhas que dispara a abertura do circuito. Exemplo: **50%** significa que se metade das chamadas dentro da janela falharem, o circuito abre.

### Slow Call Rate Threshold

Percentual de chamadas **lentas** (que excederam um tempo definido) que também pode abrir o circuito. Útil para detectar degradação de performance antes que se transforme em falha total.

### Wait Duration in Open State

Tempo que o circuito permanece aberto antes de permitir chamadas de teste (transição para HALF_OPEN). Valor típico: 15–60 segundos.

### Permitted Calls in Half-Open

Quantidade de chamadas de teste permitidas em HALF_OPEN para avaliar se a dependência se recuperou. Valor típico: 3–5 chamadas.

### Minimum Number of Calls

Número mínimo de chamadas necessárias antes de calcular a taxa de falhas. Evita abrir o circuito com uma amostra estatisticamente insignificante.

---

## Fallback

Quando o circuito está aberto, as chamadas são rejeitadas. O **fallback** define o que fazer nessa situação:

| Estratégia de Fallback | Descrição |
|------------------------|-----------|
| **Valor padrão** | Retorna um valor default ou cache (ex: "lista de produtos vazia" em vez de erro) |
| **Serviço alternativo** | Chama uma dependência secundária (ex: cache local, outro provedor) |
| **Exceção customizada** | Lança uma exceção de negócio clara (ex: "Serviço temporariamente indisponível") |
| **Degradação graceful** | Desativa a funcionalidade que depende do serviço falho mas mantem o restante operando |

O fallback **não deve ser silencioso** — ele deve sempre logar o evento e, idealmente, emitir uma métrica para que a equipe saiba que o circuito abriu.

---

## Quais exceções devem abrir o circuito?

| Deve abrir o circuito | NÃO deve abrir o circuito |
|----------------------|---------------------------|
| IOException (falha de rede) | Exceções de validação (400) |
| TimeoutException | Recurso não encontrado (404) |
| Erros HTTP 5xx (502, 503, 504) | Erros de negócio (regra violada) |
| Connection refused / reset | Exceções de autorização (401, 403) |

**Regra:** Exceções que indicam **problema na dependência** devem ser contabilizadas. Exceções que indicam **problema no request do chamador** devem ser ignoradas.

---

## Configurações Típicas

| Parâmetro | Valor Típico | Descrição |
|-----------|-------------|-----------|
| Sliding window type | COUNT_BASED | Avalia as últimas N chamadas |
| Sliding window size | 10–20 | Tamanho da janela |
| Failure rate threshold | 50% | Percentual de falhas para abrir |
| Slow call duration threshold | 2–5s | O que é considerado "chamada lenta" |
| Slow call rate threshold | 80% | Percentual de chamadas lentas para abrir |
| Wait duration in open state | 15–60s | Tempo no estado OPEN |
| Permitted calls in half-open | 3–5 | Chamadas de teste |
| Minimum number of calls | 5–10 | Mínimo para avaliar taxa de falhas |

---

## Diagrama de Decisão

```
Chamada ao serviço externo
       │
       ▼
┌── Circuito está OPEN? ──┐
│                         │
│ SIM                     │ NÃO
│                         │
▼                         ▼
Reject imediatamente    Executa a chamada
(fallback)                   │
                             ▼
                    ┌── Sucesso? ──┐
                    │              │
                    │ SIM          │ NÃO
                    │              │
                    ▼              ▼
               Retorna OK    Contabiliza falha
                                   │
                                   ▼
                          Threshold atingido?
                          │              │
                          │ NÃO          │ SIM
                          │              │
                          ▼              ▼
                     Continua       Abre o circuito
                     CLOSED         → OPEN
```

---

## Exemplo Conceitual (Pseudocódigo)

```
class CircuitBreaker:
    state = CLOSED
    failureCount = 0
    successCount = 0
    lastFailureTime = null
    
    function call(operation):
        if state == OPEN:
            if (now - lastFailureTime) > waitDuration:
                state = HALF_OPEN
            else:
                return fallback()  // fail-fast
        
        try:
            result = operation.execute()
            onSuccess()
            return result
        catch exception:
            onFailure()
            return fallback()
    
    function onSuccess():
        if state == HALF_OPEN:
            successCount++
            if successCount >= permittedCalls:
                state = CLOSED   // recuperou!
                resetCounters()
    
    function onFailure():
        failureCount++
        lastFailureTime = now
        if failureRate() >= threshold:
            state = OPEN
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Circuit breaker para tudo | Aplicar em chamadas locais/in-process | Use apenas para chamadas remotas/externas |
| Fallback silencioso | Engolir o erro sem logar | Sempre logue e emita métrica no fallback |
| Threshold muito baixo | Abre o circuito com poucas falhas normais | Defina minimum-number-of-calls adequado |
| Wait duration muito curto | Circuito fica "piscando" entre OPEN e CLOSED | Aumente waitDuration para dar tempo de recuperação |
| Exceções de negócio contabilizadas | Erros 400/404 abrem o circuito indevidamente | Configure ignore-exceptions para erros de negócio |
| Sem métricas | Não sabe quando o circuito está abrindo | Exponha estado do CB como métrica e alerte em OPEN |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Retry** | Retry fica **dentro** do Circuit Breaker. O CB contabiliza as falhas finais (após retries). |
| **Timeout** | Timeout garante que chamadas lentas falhem rápido; o CB contabiliza esses timeouts. |
| **Bulkhead** | Bulkhead isola os recursos; o CB protege contra falhas em cascata. Complementares. |
| **Fallback / Degradação Graceful** | O fallback do CB é a implementação da degradação graceful. |
| **Health Check** | O estado do CB é um indicador de saúde das dependências. |

---

## Boas Práticas

1. **Nomeie por dependência** — cada serviço externo deve ter sua própria instância de circuit breaker (`payment-service`, `notification-service`).
2. **Ignore exceções de negócio** — exceções que indicam erro do chamador (validação, 404) não devem abrir o circuito.
3. **Sempre tenha fallback** — mesmo que o fallback lance uma exceção customizada, ele deve logar e dar contexto.
4. **Monitore o estado** — exponha métricas do circuit breaker (estado, taxa de falhas, chamadas rejeitadas) e crie alertas para quando abrir.
5. **Ajuste thresholds por dependência** — um serviço de pagamento pode ter thresholds mais agressivos que um serviço de notificação.
6. **Combine com Retry e Timeout** — Circuit Breaker sozinho não resolve tudo; a composição é o padrão ideal.
7. **Teste o circuito aberto** — simule falhas em testes para validar que o fallback funciona corretamente.

---

## Referências

- Michael Nygard — *Release It!* (Pragmatic Bookshelf)
- Martin Fowler — [Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- Microsoft — [Circuit Breaker Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
