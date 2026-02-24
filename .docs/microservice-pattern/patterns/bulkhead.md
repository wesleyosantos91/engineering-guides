# Bulkhead

> **Categoria:** Resiliência
> **Origem:** Analogia com os compartimentos estanques de um navio (bulkheads)
> **Complementa:** Circuit Breaker, Timeout, Rate Limiter
> **Keywords:** resiliência, isolamento, thread pool, semáforo, contenção de falhas, compartimentalização

---

## Problema

Um serviço faz chamadas a múltiplas dependências externas (Payment, Notification, Report). Se o serviço de **Report** ficar lento, todas as threads do pool são consumidas esperando respostas dele — e as funcionalidades de Payment e Notification, que estão saudáveis, também param de funcionar.

**Uma dependência degradada consome todos os recursos compartilhados e derruba funcionalidades que não têm nada a ver com ela.**

---

## Solução

Isolar as chamadas em **pools de recursos separados** (compartimentos), cada um com seu próprio limite de concorrência. Se o compartimento do Report encher, apenas chamadas ao Report são afetadas — Payment e Notification continuam operando normalmente.

Assim como os compartimentos estanques de um navio impedem que uma brecha afunde o navio inteiro, o Bulkhead impede que uma dependência degrade todo o sistema.

---

## Tipos de Bulkhead

### 1. Semaphore Bulkhead

Usa um **semáforo** para limitar o número de chamadas concorrentes. Não cria threads separadas — usa as threads existentes.

```
┌──────────────────────────────────────────┐
│              Thread Pool Geral           │
│                                          │
│  ┌─ Semáforo: Payment (max=10) ──┐      │
│  │  ████████░░  (8/10 em uso)    │      │
│  └───────────────────────────────┘      │
│                                          │
│  ┌─ Semáforo: Report (max=5) ────┐      │
│  │  █████  (5/5 em uso → CHEIO)  │      │
│  └───────────────────────────────┘      │
│                                          │
│  ┌─ Semáforo: Notification (max=15) ─┐  │
│  │  ███░░░░░░░░░░░░  (3/15 em uso)  │  │
│  └───────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

| Característica | Detalhe |
|---------------|---------|
| **Isolamento** | Limita concorrência, mas usa threads compartilhadas |
| **Overhead** | Baixo — apenas um semáforo por compartimento |
| **Quando usar** | Maioria dos cenários; chamadas síncronas |

### 2. Thread Pool Bulkhead

Cria um **pool de threads dedicado** para cada dependência. Isolamento total — threads de um pool não são compartilhadas.

```
┌──────────────────────────────────────────────┐
│  ┌─ Thread Pool: Payment ──────────────┐    │
│  │  Threads: 5 core / 10 max          │    │
│  │  Queue: 20                          │    │
│  │  ████████░░  (8/10 em uso)          │    │
│  └─────────────────────────────────────┘    │
│                                              │
│  ┌─ Thread Pool: Report ───────────────┐    │
│  │  Threads: 2 core / 5 max           │    │
│  │  Queue: 10                          │    │
│  │  █████  (5/5 em uso → usa queue)    │    │
│  └─────────────────────────────────────┘    │
│                                              │
│  ┌─ Thread Pool: Notification ─────────┐    │
│  │  Threads: 3 core / 8 max           │    │
│  │  Queue: 15                          │    │
│  │  ███░░░░░  (3/8 em uso)            │    │
│  └─────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

| Característica | Detalhe |
|---------------|---------|
| **Isolamento** | Total — cada dependência tem suas próprias threads |
| **Overhead** | Maior — threads dedicadas consomem memória |
| **Quando usar** | Quando isolamento total é necessário; dependências com latências muito diferentes |

---

## Comparação

| Aspecto | Semaphore | Thread Pool |
|---------|-----------|-------------|
| Isolamento | Parcial (limita concorrência) | Total (threads dedicadas) |
| Overhead | Baixo | Médio-Alto |
| Context switch | Não (mesma thread) | Sim (thread diferente) |
| Complexidade | Simples | Maior (dimensionar pools) |
| Quando usar | Padrão para maioria dos casos | Dependências com latência muito diferente |

---

## Como dimensionar

| Parâmetro | Como calcular |
|-----------|--------------|
| **max-concurrent-calls** | `throughput_esperado × latência_média_da_dependência` |
| Exemplo | 100 req/s × 0.1s (100ms) = 10 chamadas concorrentes |
| Com margem | 10 × 1.5 = 15 (50% de margem) |

**Para Thread Pool:**

| Parâmetro | Consideração |
|-----------|-------------|
| **core-thread-pool-size** | Carga normal esperada |
| **max-thread-pool-size** | Picos de carga |
| **queue-capacity** | Buffer para absorver bursts (cuidado — queue grande mascara problemas) |

---

## Comportamento quando o Bulkhead está cheio

| Configuração | Comportamento |
|-------------|--------------|
| **max-wait-duration: 0** | Reject imediato (fail-fast) — **recomendado** |
| **max-wait-duration: 500ms** | Espera até 500ms por um slot; rejeita se não liberar |

**Recomendação:** Use `max-wait-duration: 0` — enfileirar consume memória e mascara problemas. Se o bulkhead está cheio, a dependência está sobrecarregada e esperar só piora.

---

## Diagrama de Decisão

```
Chamada ao serviço externo
           │
           ▼
Qual bulkhead? (por dependência)
           │
           ▼
┌── Há slot disponível? ──┐
│                          │
│ SIM                      │ NÃO
│                          │
▼                          ▼
Adquire slot          max-wait-duration > 0?
Executa chamada       │              │
Libera slot           │ SIM          │ NÃO
                      ▼              ▼
                 Espera timeout  Reject imediato
                      │          (BulkheadFullException)
                      ▼
                 Slot liberou?
                 │         │
                 │ SIM     │ NÃO
                 ▼         ▼
            Executa    Reject
                       (BulkheadFullException)
```

---

## Exemplo Conceitual (Pseudocódigo)

```
class SemaphoreBulkhead:
    semaphore = Semaphore(maxConcurrentCalls)
    
    function call(operation):
        acquired = semaphore.tryAcquire(maxWaitDuration)
        
        if not acquired:
            throw BulkheadFullException(
                "Bulkhead cheio: max concurrent calls atingido")
        
        try:
            return operation.execute()
        finally:
            semaphore.release()  // SEMPRE libera — mesmo se falhar
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Bulkhead único para tudo | Não isola nada — equivale a não ter bulkhead | Crie um bulkhead por dependência |
| max-concurrent-calls muito alto | Bulkhead nunca é acionado; não protege | Dimensione baseado no throughput real |
| max-wait-duration muito alto | Consome memória e mascara degradação | Use 0 (reject imediato) |
| Queue muito grande (thread pool) | Buffer infinito mascara que a dependência está lenta | Limite a queue; prefira rejeição |
| Sem fallback | Erro genérico quando bulkhead rejeita | Forneça fallback com contexto |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Circuit Breaker** | CB protege contra falhas; Bulkhead protege contra lentidão. Complementares. |
| **Timeout** | Timeout garante que chamadas lentas liberem o slot do bulkhead mais rápido. |
| **Rate Limiter** | Rate Limiter controla requisições por tempo; Bulkhead controla concorrência simultânea. |
| **Thread Pool** | Thread Pool Bulkhead é uma especialização do padrão Thread Pool. |

---

## Boas Práticas

1. Crie **um bulkhead por dependência** — `paymentBulkhead`, `reportBulkhead`, `notificationBulkhead`.
2. Use **Semaphore** (padrão) para maioria dos casos — menor overhead.
3. Use **Thread Pool** apenas quando precisar de isolamento total entre dependências.
4. Configure `max-wait-duration: 0` — rejeite imediatamente em vez de enfileirar.
5. Dimensione `max-concurrent-calls` baseado no throughput e latência esperados.
6. Combine com **Timeout** — se uma chamada está lenta, o timeout libera o slot mais rápido.
7. Monitore **utilização do bulkhead** — slots disponíveis vs. máximo é uma métrica importante.
8. Combine com **Circuit Breaker** — o CB detecta falhas em cascata; o Bulkhead limita o dano.

---

## Referências

- Michael Nygard — *Release It!* (Pragmatic Bookshelf)
- Microsoft — [Bulkhead Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead)
- Netflix — [Hystrix: Isolation](https://github.com/Netflix/Hystrix/wiki/How-it-Works#isolation)
