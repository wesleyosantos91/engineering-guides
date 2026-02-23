# Outbox Pattern

> **Categoria:** Event-Driven e Mensageria
> **Complementa:** Event Sourcing, Saga, CDC, Idempotência

---

## Problema

Ao salvar dados no banco **e** publicar um evento no message broker, temos um problema de **dual write** (escrita dupla):

```
function createOrder(request):
    database.save(order)          // 1. Salva no banco  ✅
    messageBroker.publish(event)  // 2. Publica evento  ❌ (broker caiu)
```

Se o passo 2 falhar, o dado está no banco mas o evento não foi publicado. Outros serviços nunca saberão da mudança. **O sistema fica inconsistente.**

O inverso também é problemático:

```
function createOrder(request):
    messageBroker.publish(event)  // 1. Publica evento  ✅
    database.save(order)          // 2. Salva no banco  ❌ (banco caiu)
```

Evento publicado, dado nunca salvo. **Pior cenário.**

---

## Solução

Salvar o evento em uma **tabela outbox** no mesmo banco de dados e na **mesma transação** que a operação de negócio. Um processo separado (Outbox Publisher) lê a tabela outbox e publica os eventos no message broker.

```
┌─── Mesma Transação ───────────────────────┐
│                                           │
│  1. INSERT INTO orders (...)              │
│  2. INSERT INTO outbox (event_payload)    │
│                                           │
└── COMMIT (atômico) ──────────────────────┘

                    │
            (processo separado)
                    │
                    ▼
        ┌── Outbox Publisher ──┐
        │ Lê outbox            │
        │ Publica no broker    │
        │ Marca como enviado   │
        └──────────────────────┘
```

**Garantia:** Se a operação de negócio foi salva, o evento **também** foi salvo (mesma transação). A publicação pode atrasar, mas nunca será perdida.

---

## Tabela Outbox

```
Tabela: outbox_events
┌──────────────────────────────────────────────────────────┐
│ id               │ UUID (PK)                            │
│ aggregate_type   │ "Order"                              │
│ aggregate_id     │ "order-123"                          │
│ event_type       │ "OrderCreated"                       │
│ payload          │ {"orderId": "123", "total": 1500}    │
│ destination      │ "order-events" (tópico/fila destino) │
│ status           │ PENDING / SENT / FAILED              │
│ created_at       │ 2026-02-23T10:30:00Z                 │
│ sent_at          │ null                                 │
│ retry_count      │ 0                                    │
└──────────────────────────────────────────────────────────┘
```

---

## Fluxo Detalhado

```
1. Serviço recebe comando (ex: criar pedido)

2. MESMA TRANSAÇÃO:
   BEGIN TRANSACTION
     INSERT INTO orders (id, ...) VALUES (...)
     INSERT INTO outbox_events (id, event_type, payload, ...) VALUES (...)
   COMMIT

3. Outbox Publisher (processo separado):
   Loop contínuo:
     a. SELECT * FROM outbox_events WHERE status = 'PENDING' ORDER BY created_at
     b. Para cada evento:
        - Publica no message broker
        - UPDATE outbox_events SET status = 'SENT', sent_at = now()
     c. Sleep(intervalo)
```

---

## Estratégias do Outbox Publisher

### 1. Polling (Pull)

O publisher **consulta** a tabela outbox periodicamente:

```
Outbox Publisher:
  every 500ms:
    events = SELECT * FROM outbox_events 
             WHERE status = 'PENDING' 
             ORDER BY created_at 
             LIMIT 100
    
    for event in events:
        broker.publish(event.destination, event.payload)
        UPDATE outbox_events SET status = 'SENT' WHERE id = event.id
```

| Vantagem | Desvantagem |
|----------|------------|
| Simples de implementar | Latência (depende do intervalo de polling) |
| Funciona com qualquer banco | Carga no banco (queries periódicas) |
| Sem dependências extras | Não escala facilmente |

### 2. CDC (Change Data Capture)

Um sistema de CDC (ex: Debezium) **captura mudanças** na tabela outbox em tempo real lendo o transaction log do banco:

```
Banco (binlog/WAL) ──▶ CDC Connector ──▶ Message Broker

Fluxo:
  1. INSERT na tabela outbox → binlog registra
  2. CDC lê o binlog em tempo real
  3. CDC publica o evento no message broker
  4. (opcional) remove ou marca o registro na outbox
```

| Vantagem | Desvantagem |
|----------|------------|
| Latência muito baixa (sub-segundo) | Requer infraestrutura de CDC |
| Sem polling no banco | Mais complexo para operar |
| Escalável | Dependente do binlog do banco |

---

## Limpeza da Tabela Outbox

A tabela outbox cresce continuamente. Estratégias de limpeza:

| Estratégia | Descrição |
|-----------|-----------|
| **Delete após SENT** | Deleta imediatamente após publicar. Simples, sem histórico. |
| **Delete periódico** | Job que deleta registros SENT com mais de X horas/dias. |
| **Particionamento** | Particiona por data — drop de partições antigas. |
| **Retenção** | Mantém por N dias para debugging/auditoria, depois limpa. |

```
// Job de limpeza periódico
DELETE FROM outbox_events 
WHERE status = 'SENT' 
  AND sent_at < now() - INTERVAL 7 DAY
```

---

## Ordering (Garantia de Ordem)

Para garantir que eventos do **mesmo agregado** sejam publicados na ordem correta:

```
1. Na tabela outbox, use created_at + sequence por aggregate_id
2. O publisher deve processar em ordem por aggregate_id
3. No message broker, use o aggregate_id como partition key

Publisher:
  events = SELECT * FROM outbox_events 
           WHERE status = 'PENDING'
           ORDER BY aggregate_id, created_at  // ordena por agregado
```

---

## Exemplo Conceitual (Pseudocódigo)

```
// Serviço que cria pedido
class OrderService:
    orderRepository
    outboxRepository
    
    function createOrder(request):
        order = Order.create(request)
        
        event = OutboxEvent(
            aggregateType: "Order",
            aggregateId: order.id,
            eventType: "OrderCreated",
            payload: serialize(OrderCreatedPayload(order)),
            destination: "order-events"
        )
        
        // TRANSAÇÃO ATÔMICA
        transaction.begin()
        orderRepository.save(order)
        outboxRepository.save(event)
        transaction.commit()

// Outbox Publisher (processo separado)
class OutboxPublisher:
    outboxRepository
    messageBroker
    
    function pollAndPublish():
        events = outboxRepository.findPending(limit: 100)
        
        for event in events:
            try:
                messageBroker.publish(
                    topic: event.destination,
                    key: event.aggregateId,
                    payload: event.payload
                )
                outboxRepository.markAsSent(event.id)
            catch PublishException:
                outboxRepository.incrementRetry(event.id)
                if event.retryCount >= MAX_RETRIES:
                    outboxRepository.markAsFailed(event.id)
                    alertOps("Outbox event falhou após {} retries", event.id)
```

---

## Outbox + Idempotência

O consumer que recebe eventos da outbox **deve ser idempotente**, porque:
- O publisher pode enviar o mesmo evento mais de uma vez (at-least-once)
- Se o publisher cair entre publicar e marcar como SENT, republicará

```
Consumer:
  function onOrderCreated(event):
      if alreadyProcessed(event.id):
          return  // skip duplicata
      
      processEvent(event)
      markAsProcessed(event.id)
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Publish + Save separados | Dual write — inconsistência | Use transação atômica (outbox) |
| Outbox sem limpeza | Tabela cresce indefinidamente | Job de limpeza periódico |
| Publisher sem retry | Eventos perdidos se broker falhar | Implemente retry no publisher |
| Sem ordering | Eventos de um agregado chegam fora de ordem | Use aggregate_id como partition key |
| Consumer sem idempotência | Eventos duplicados processados 2x | Consumer deve ser idempotente |
| Polling muito frequente | Carga desnecessária no banco | Use CDC ou ajuste o intervalo |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **CDC** | CDC pode substituir o polling — lê a outbox via binlog em tempo real |
| **Event Sourcing** | Event Sourcing pode usar outbox para publicar eventos para projeções |
| **Saga** | Cada step da saga usa outbox para garantir consistência estado+evento |
| **Idempotência** | Consumers devem ser idempotentes (at-least-once delivery) |
| **DLQ** | Eventos da outbox que falham repetidamente podem ir para DLQ |

---

## Boas Práticas

1. Salve o evento na outbox na **mesma transação** da operação de negócio.
2. Use **CDC** para latência sub-segundo; use **polling** para simplicidade.
3. Implemente **limpeza periódica** da tabela outbox (delete registros SENT antigos).
4. Use `aggregate_id` como **partition key** para garantir ordem por agregado.
5. O consumer **deve ser idempotente** — o publisher garante at-least-once.
6. Monitore o **lag** da outbox (tempo entre INSERT e publicação).
7. Implemente **retry** no publisher para falhas transitórias.
8. Mantenha o **payload no evento** — não force o consumer a buscar dados adicionais.

---

## Referências

- Chris Richardson — *Microservices Patterns* (Manning) — Transactional Outbox Pattern
- Debezium — [Outbox Event Router](https://debezium.io/documentation/reference/transformations/outbox-event-router.html)
- Microsoft — [Transactional Outbox Pattern](https://learn.microsoft.com/en-us/azure/architecture/best-practices/transactional-outbox-cosmos)
