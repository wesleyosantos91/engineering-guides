# DLQ (Dead Letter Queue)

> **Categoria:** Persistência e Processamento de Dados
> **Complementa:** Retry, Idempotência, Saga
> **Keywords:** DLQ, Dead Letter Queue, fila de mensagens mortas, reprocessamento, tratamento de erros, mensagens com falha

---

## Problema

Um consumer de mensagens falha ao processar uma mensagem. Se a mensagem for reenfileirada indefinidamente, ela **bloqueia o processamento** das mensagens seguintes (poison message). Se for descartada, **perde-se dados** sem possibilidade de recuperação.

---

## Solução

Após N tentativas de processamento sem sucesso, enviar a mensagem para uma **fila de mensagens mortas (Dead Letter Queue — DLQ)**. A DLQ é uma fila separada onde mensagens problemáticas ficam armazenadas para **análise, correção e reprocessamento** posterior.

---

## Fluxo

```
┌──────────────┐       ┌──────────────┐       ┌──────────────────┐
│  Producer    │──────▶│  Tópico/Fila │──────▶│  Consumer        │
│              │       │  Principal   │       │                  │
└──────────────┘       └──────────────┘       └──────┬───────────┘
                                                      │
                                               ┌──────▼───────┐
                                               │ Processou OK?│
                                               └──────┬───────┘
                                                ┌─────┴─────┐
                                                │           │
                                               SIM         NÃO
                                                │           │
                                                ▼           ▼
                                            Acknowledge  Tentativa N?
                                            (remove)     ┌────┴────┐
                                                         │         │
                                                        NÃO       SIM
                                                         │         │
                                                         ▼         ▼
                                                     Reenfileira  Envia p/ DLQ
                                                     (retry)      ┌─────────┐
                                                                  │  DLQ    │
                                                                  │ Tópico  │
                                                                  └────┬────┘
                                                                       │
                                                                       ▼
                                                                  DLQ Consumer
                                                                  (análise /
                                                                  reprocess)
```

---

## Tipos de Erro e Tratamento

Nem todos os erros devem ir para a DLQ. A decisão depende da **natureza do erro**:

| Tipo de Erro | Exemplo | Ação | Vai para DLQ? |
|-------------|---------|------|---------------|
| **Transitório** | Timeout, 503, connection reset | Retry com backoff | Não (a princípio) |
| **Transitório persistente** | Erro transitório que não se resolve após N retries | DLQ | Sim |
| **Irrecuperável (dados)** | JSON inválido, schema incompatível | DLQ imediato (sem retry) | Sim |
| **Irrecuperável (lógica)** | Validação de negócio, dados inconsistentes | DLQ imediato (sem retry) | Sim |
| **Poison Message** | Mensagem que sempre causa crash no consumer | DLQ imediato | Sim |

**Regra:** Erros **irrecuperáveis** devem ir direto para a DLQ sem retry — retentar não vai resolver e bloqueia processamento das demais mensagens.

---

## Convenção de Nomenclatura

| Tópico/Fila Principal | DLQ |
|-----------------------|-----|
| `order-created` | `order-created.dlq` |
| `payment-processed` | `payment-processed.dlq` |
| `notification-send` | `notification-send.dlq` |

**Padrão:** `{nome-do-topico}.dlq`

---

## Metadados na DLQ

Quando uma mensagem vai para a DLQ, é fundamental incluir **metadados** que ajudem na análise:

```
Mensagem na DLQ:
┌──────────────────────────────────────────────┐
│ payload original (inalterado)                │
│                                              │
│ Headers / Metadados:                         │
│   original-topic: order-created              │
│   original-partition: 2                      │
│   original-offset: 15432                     │
│   original-timestamp: 2026-02-23T10:30:00Z   │
│   error-message: "NullPointerException..."   │
│   error-class: NullPointerException          │
│   retry-count: 3                             │
│   consumer-group: order-service              │
│   sent-to-dlq-at: 2026-02-23T10:30:15Z      │
└──────────────────────────────────────────────┘
```

---

## Estratégias de Reprocessamento

Mensagens na DLQ não devem ficar lá para sempre. Há estratégias para tratá-las:

### 1. Reprocessamento Manual

Um operador analisa a mensagem, corrige o problema (deploy de fix, dados corrigidos) e reenfileira manualmente.

```
DLQ ──(operador analisa)──▶ Corrige bug/dados ──▶ Reenfileira no tópico original
```

### 2. Reprocessamento Automatizado (com cooldown)

Um DLQ consumer tenta reprocessar automaticamente após um período de espera (cooldown):

```
DLQ ──(espera 30min)──▶ Tenta processar novamente
                             │
                        ┌────┴────┐
                        │         │
                      Sucesso    Falha
                        │         │
                        ▼         ▼
                    Remove     Incrementa contador
                    da DLQ     retry_count >= MAX?
                                  │         │
                                 NÃO       SIM
                                  │         │
                                  ▼         ▼
                             Cooldown    Persiste em banco
                             novamente   (análise humana)
```

### 3. Persistência em Banco para Auditoria

Salva a mensagem da DLQ em uma tabela no banco para auditoria, dashboard e reprocessamento via admin UI:

```
Tabela: dlq_messages
┌────────────────────────────────────────────────┐
│ id              │ UUID                         │
│ original_topic  │ order-created                │
│ original_key    │ order-123                    │
│ payload         │ {json da mensagem}           │
│ error_message   │ NullPointerException at...   │
│ retry_count     │ 3                            │
│ status          │ PENDING / REPROCESSED / DEAD │
│ created_at      │ 2026-02-23T10:30:15Z         │
│ reprocessed_at  │ null                         │
└────────────────────────────────────────────────┘
```

---

## DLQ por Message Broker

| Message Broker | Mecanismo de DLQ |
|---------------|-----------------|
| **Apache Kafka** | Tópico separado (ex: `topic.dlq`). Consumer publica na DLQ manualmente ou via error handler. |
| **RabbitMQ** | Dead Letter Exchange (DLX). Configurável por fila com `x-dead-letter-exchange`. |
| **AWS SQS** | DLQ nativa. Configurável com `RedrivePolicy` (maxReceiveCount). |
| **Azure Service Bus** | DLQ nativa por fila/subscription. Mensagens movidas automaticamente. |
| **Google Pub/Sub** | Dead letter topic. Configurável por subscription. |

---

## Exemplo Conceitual (Pseudocódigo)

```
// Error Handler — decide retry ou DLQ
function handleConsumerError(message, exception, retryCount):
    if isNonRetryable(exception):
        // Erro irrecuperável — DLQ imediato
        sendToDlq(message, exception, retryCount)
        return
    
    if retryCount < MAX_RETRIES:
        // Retry com backoff
        delay = baseDelay × (2 ^ retryCount)
        scheduleRetry(message, delay)
    else:
        // Retries esgotados — DLQ
        sendToDlq(message, exception, retryCount)

function sendToDlq(message, exception, retryCount):
    dlqMessage = enrichWithMetadata(message, exception, retryCount)
    publish(message.topic + ".dlq", dlqMessage)
    log.error("Mensagem enviada para DLQ: topic={}, key={}", 
              message.topic, message.key)
    metrics.increment("dlq.messages.total", topic=message.topic)

function isNonRetryable(exception):
    return exception instanceof [
        DeserializationException,
        ValidationException,
        SchemaIncompatibleException
    ]
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| DLQ como lixeira | Mensagens acumulam sem análise | Monitore, alerte e processe |
| Sem metadados na DLQ | Impossível diagnosticar a causa | Enriqueça com error message, stack trace, metadata |
| Retry em erros irrecuperáveis | Retenta 3x uma mensagem com JSON malformado — nunca vai funcionar | Classifique erros: retriável vs. irrecuperável |
| DLQ sem monitoramento | DLQ acumula sem ninguém saber | Alerte quando DLQ tiver mensagens |
| Reprocessamento sem fix | Reenfileira da DLQ sem corrigir o bug — volta para DLQ | Corrija antes de reprocessar |
| Consumer da DLQ sem idempotência | Reprocessamento duplica efeitos | Garanta idempotência |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Retry** | Retry tenta antes da DLQ. DLQ é o "último recurso" quando retries esgotam. |
| **Idempotência** | Reprocessamento da DLQ deve ser idempotente. |
| **Circuit Breaker** | Se a DLQ está acumulando, pode indicar que o Circuit Breaker deveria estar atuando. |
| **Saga** | Falhas em steps da saga podem ser encaminhadas para DLQ para compensação posterior. |
| **Outbox** | Mensagens da outbox que falharam repetidamente podem ir para DLQ. |

---

## Boas Práticas

1. Nomeie a DLQ com sufixo `.dlq` (ex: `order-created.dlq`).
2. **Persista** mensagens da DLQ em banco para auditoria e reprocessamento.
3. Separe erros **retriáveis** (timeout) de **irrecuperáveis** (desserialização) — irrecuperáveis vão direto para DLQ.
4. **Monitore o lag** da DLQ e crie alertas quando houver mensagens acumuladas.
5. Implemente mecanismo de **reprocessamento** (manual ou automatizado com cooldown).
6. Inclua **metadados** detalhados (erro, stack trace, tentativas, timestamp original).
7. O consumer do reprocessamento deve ser **idempotente**.
8. Defina **responsável** pela DLQ — alguém precisa olhar quando alerta disparar.

---

## Referências

- Enterprise Integration Patterns — *Dead Letter Channel*
- AWS — [Amazon SQS Dead-Letter Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- Confluent — [Error Handling Patterns for Apache Kafka](https://www.confluent.io/blog/error-handling-patterns-in-kafka/)
