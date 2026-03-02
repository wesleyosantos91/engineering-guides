# Level 7 — Messaging Systems: Message Queues & Pub/Sub

> **Objetivo:** Implementar um Message Broker com suporte a point-to-point queues e
> publish/subscribe topics, com garantias de entrega e ordenação.

**Referência:**
- [14-message-queues.md](../../.docs/SYSTEM-DESIGN/14-message-queues.md)
- [15-pub-sub.md](../../.docs/SYSTEM-DESIGN/15-pub-sub.md)

**Pré-requisito:** Level 6 completo.

---

## Parte 1 — ADR: Message Broker Design

**Arquivo:** `docs/adrs/ADR-001-message-broker-design.md`

**Decisão:** Arquitetura do message broker e garantias de entrega.

**Options:**
1. **At-most-once** — fire and forget (pode perder mensagens)
2. **At-least-once** — retry até ACK (pode duplicar)
3. **Exactly-once** — transactional (complexo, lento)

**Options — Modelo:**
1. **Point-to-Point (Queue)** — 1 producer → 1 consumer
2. **Pub/Sub (Topic)** — 1 producer → N consumers
3. **Hybrid** — suporta ambos os modelos

**Decision Drivers:**
- Garantia de entrega necessária
- Ordenação de mensagens
- Throughput vs latência
- Consumer group support
- Dead letter queue

**Critérios de aceite:**
- [ ] Trade-offs de garantia de entrega documentados
- [ ] Queue vs Topic use cases
- [ ] Consumer group design documentado
- [ ] Dead letter queue strategy

---

## Parte 2 — Diagrama DrawIO

**Arquivo:** `docs/diagrams/07-messaging-architecture.drawio`

**View 1 — Message Queue (Point-to-Point):**
```
┌──────────┐    ┌─────────────────────┐    ┌──────────┐
│Producer 1│──▶ │       Queue         │──▶ │Consumer 1│
│          │    │ [msg1][msg2][msg3]   │    │(competes)│
│Producer 2│──▶ │                     │──▶ │Consumer 2│
└──────────┘    └─────────────────────┘    └──────────┘
```

**View 2 — Pub/Sub (Topic):**
```
┌──────────┐    ┌─────────────────────┐    ┌──────────┐
│Publisher  │──▶ │       Topic         │──▶ │Subscriber│
│          │    │                     │──▶ │    A     │
│          │    │  ┌─────┐ ┌─────┐   │──▶ │Subscriber│
│          │    │  │Part0│ │Part1│   │    │    B     │
│          │    │  └─────┘ └─────┘   │    │Subscriber│
│          │    │                     │    │    C     │
└──────────┘    └─────────────────────┘    └──────────┘
```

**View 3 — Message Lifecycle:** Produce → Store → Deliver → ACK → Delete/DLQ

**View 4 — Consumer Groups:** Partition assignment e rebalancing

**Critérios de aceite:**
- [ ] Queue e Topic flows distintos
- [ ] Message lifecycle completo
- [ ] Consumer group com partitions
- [ ] DLQ flow

---

## Parte 3 — Implementação

### 3.1 — Go: Mini Message Broker

**Estrutura:**
```
go/
├── cmd/
│   ├── broker/main.go             ← Message broker server
│   ├── producer/main.go           ← Producer client
│   └── consumer/main.go           ← Consumer client
├── internal/
│   ├── broker/
│   │   ├── broker.go              ← Core broker engine
│   │   ├── queue.go               ← Point-to-point queue
│   │   ├── topic.go               ← Pub/Sub topic
│   │   ├── partition.go           ← Topic partition
│   │   ├── consumer_group.go      ← Consumer group management
│   │   └── broker_test.go
│   ├── storage/
│   │   ├── log.go                 ← Append-only log (Kafka-style)
│   │   ├── segment.go             ← Log segment
│   │   └── index.go               ← Offset index
│   ├── protocol/
│   │   ├── message.go             ← Message format
│   │   ├── codec.go               ← Serialization
│   │   └── wire.go                ← Wire protocol (TCP)
│   ├── delivery/
│   │   ├── ack.go                 ← ACK tracking
│   │   ├── retry.go               ← Retry with backoff
│   │   └── dlq.go                 ← Dead letter queue
│   └── client/
│       ├── producer.go            ← Producer SDK
│       └── consumer.go            ← Consumer SDK
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **Append-only Log** persistente (Kafka-style segments)
2. **Queue** com competing consumers (round-robin delivery)
3. **Topic** com partitions e subscriber fan-out
4. **Consumer Groups** com partition assignment (range assignor)
5. **ACK tracking** com timeout e retry
6. **Dead Letter Queue** para mensagens que falharam N vezes
7. **TCP Wire Protocol** para comunicação broker ↔ client
8. **Producer** com batching e compression
9. **Consumer** com offset tracking e auto-commit
10. **Backpressure** quando consumer está lento

**Critérios de aceite Go:**
- [ ] Queue: 1 msg entregue a exatamente 1 consumer
- [ ] Topic: 1 msg entregue a todos os subscribers
- [ ] Partitions: ordering garantida dentro de cada partition
- [ ] Consumer group: rebalancing quando consumer join/leave
- [ ] ACK: mensagem reenviada se não ACK em timeout
- [ ] DLQ: mensagem enviada para DLQ após N retries
- [ ] Persistence: mensagens sobrevivem restart do broker
- [ ] Throughput: ≥ 10K msg/s em benchmark local
- [ ] ≥ 20 testes (unit + integration)

---

### 3.2 — Java: Mini Message Broker

**Funcionalidades Java:**
1. **Broker** com Spring Boot + Netty para TCP
2. **Queue** e **Topic** como beans
3. **Append-only Log** com `FileChannel` e `MappedByteBuffer`
4. **Consumer Groups** com Virtual Threads
5. **ACK** tracking com `ConcurrentHashMap`
6. **DLQ** com retry policies
7. **Records** para message format
8. **Sealed interfaces** para delivery guarantees

**Critérios de aceite Java:**
- [ ] Queue + Topic funcionais
- [ ] Persistence com NIO
- [ ] Consumer groups
- [ ] ACK + DLQ
- [ ] Testes com JUnit 5
- [ ] JaCoCo ≥ 80%

---

## Parte 4 — Integração com Kafka/RabbitMQ

Compare sua implementação com Kafka e RabbitMQ reais:

| Feature | Seu Broker | Kafka | RabbitMQ |
|---------|-----------|-------|----------|
| Throughput | | | |
| Latency | | | |
| Ordering | | | |
| Delivery | | | |
| Persistence | | | |

---

## Definição de Pronto (DoD)

- [ ] ADR com design do broker e garantias
- [ ] DrawIO com 4 views
- [ ] Go e Java: broker com queue + topic + partitions + ACK + DLQ
- [ ] Comparativo com Kafka/RabbitMQ documentado
- [ ] Commit: `feat(system-design-07): message queues and pub sub`
