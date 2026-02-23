# Backpressure

> **Categoria:** Event-Driven e Mensageria
> **Complementa:** Rate Limiter, Bulkhead, DLQ

---

## Problema

Um **producer** gera mensagens/requests mais rápido do que o **consumer** consegue processar. Sem controle:

- Filas crescem indefinidamente → **memória esgotada**, sistema falha
- Latência aumenta progressivamente → requests que deveriam levar 100ms levam 30s
- Consumer sobrecarregado → cascata de falhas para serviços downstream
- Backlog irrecuperável → consumer nunca mais alcança o producer

```
Producer: 10.000 msg/s ─────▶ Fila ─────▶ Consumer: 2.000 msg/s

Déficit: 8.000 msg/s acumulando → Em 1 hora: 28.800.000 mensagens atrasadas
```

---

## Solução

**Backpressure** é um mecanismo de **feedback** do consumer para o producer, sinalizando que o consumer está sobrecarregado e que o producer deve **desacelerar, parar ou adaptar** a taxa de envio.

```
                    ┌─── "Estou sobrecarregado!" ───┐
                    │         (backpressure)          │
                    │                                 │
                    ▼                                 │
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Producer │───▶│  Buffer  │───▶│  Queue   │───▶│ Consumer │
│          │    │          │    │          │    │          │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
```

---

## Estratégias de Backpressure

### 1. Drop (Descartar)

Descarta mensagens quando o buffer está cheio:

```
Buffer: [msg1][msg2][msg3]...[msgN] ── CHEIO!
Nova mensagem chega → DESCARTADA

Variantes:
  - Drop mais antiga (head): remove msg1, insere nova no final
  - Drop mais nova (tail): rejeita a nova mensagem
  - Drop aleatória: remove uma mensagem qualquer
```

| Quando usar | Quando NÃO usar |
|-------------|-----------------|
| Métricas/telemetria (perder uma amostra é OK) | Transações financeiras |
| Atualizações de posição GPS (última é suficiente) | Eventos de negócio que não podem ser perdidos |
| Dados de sensor com alta frequência | Mensagens com garantia de entrega |

### 2. Buffer/Queue (Enfileirar)

Armazena mensagens em uma fila durável até o consumer estar pronto:

```
Producer ──▶ Kafka/RabbitMQ/SQS (fila durável) ──▶ Consumer

Fila absorve o pico:
  t=0:  Producer 10K/s, Consumer 2K/s → fila cresce 8K/s
  t=60: Producer volta para 1K/s → Consumer drena o backlog
```

| Quando usar | Quando NÃO usar |
|-------------|-----------------|
| Picos temporários | Déficit contínuo (fila nunca drena) |
| Mensagens que não podem ser perdidas | Latência é crítica (mensagem na fila = latência) |
| Event-driven architecture | Buffer ilimitado (memória esgota) |

**Atenção:** Buffer **adia** o problema mas não resolve se o déficit for contínuo.

### 3. Throttling (Limitar o Producer)

O consumer **sinaliza** ao producer para reduzir a taxa:

```
HTTP:
  Consumer responde: 429 Too Many Requests
  Header: Retry-After: 30
  → Producer desacelera

Messaging:
  Consumer para de consumir (stop polling)
  → Broker para de entregar (consumer pull model)

Streaming (Reactive):
  Consumer.request(100)  → "Me envie no máximo 100"
  (producer só envia 100, espera novo request)
```

### 4. Scaling (Escalar Consumer)

Adicionar mais consumers para igualar a taxa do producer:

```
Antes: 1 consumer  → 2.000 msg/s
Depois: 5 consumers → 10.000 msg/s (= taxa do producer)

Auto-scaling:
  if queue_depth > threshold:
      scale_up(consumers)
  if queue_depth < low_threshold:
      scale_down(consumers)
```

### 5. Load Shedding (Rejeitar Seletivamente)

Rejeitar requests de **baixa prioridade** quando sobrecarregado, preservando os de alta prioridade:

```
Capacidade: 5.000 req/s
Carga atual: 8.000 req/s

Classificação:
  Alta prioridade:  pagamentos, autenticação → PROCESSA
  Média prioridade: buscas, listagens        → PROCESSA (se houver capacidade)
  Baixa prioridade: analytics, logs          → REJEITA (503)
```

---

## Backpressure em Diferentes Contextos

### HTTP / APIs

```
1. Servidor sobrecarregado → retorna 429 ou 503
2. Client respeita Retry-After header
3. Load balancer detecta → para de enviar tráfego

Headers relevantes:
  429 Too Many Requests     → rate limit atingido
  503 Service Unavailable   → server sobrecarregado
  Retry-After: 30           → tente novamente em 30s
```

### Message Brokers

| Broker | Mecanismo de Backpressure |
|--------|--------------------------|
| **Kafka** | Consumer-pull: consumer controla a taxa de consumo. Se parar de poll, não recebe. |
| **RabbitMQ** | Prefetch count: consumer define quantas mensagens receber sem acknowledge. |
| **SQS** | Visibility timeout + max receive: controla taxa de processamento. |

```
Kafka (pull model):
  Consumer.poll(maxRecords=100, timeout=1s)
  → Consumer só recebe 100 mensagens por vez
  → Se processar devagar, naturalmente consome menos

RabbitMQ (prefetch):
  channel.basicQos(prefetchCount=10)
  → Broker envia no máximo 10 mensagens sem ACK
  → Se consumer está lento, broker para de enviar
```

### Streaming / Reactive

```
Reactive Streams (conceito):
  Publisher.subscribe(Subscriber)
  Subscriber.onSubscribe(subscription)
  subscription.request(100)    → "me dê 100 itens"
  Publisher envia até 100 itens
  subscription.request(50)     → "me dê mais 50"
  ...

O consumer CONTROLA quanto recebe → backpressure natural
```

### Banco de Dados

```
Sem backpressure:
  1000 threads → 1000 queries simultâneas → banco cai

Com backpressure (connection pool):
  Pool size = 20 → máximo 20 queries simultâneas
  Thread 21+ → espera na fila do pool
  
  Connection pool é uma forma de backpressure no banco.
```

---

## Métricas de Monitoramento

| Métrica | O que indica |
|---------|-------------|
| **Queue depth** (profundidade da fila) | Acúmulo de mensagens — se crescendo, consumer não acompanha |
| **Consumer lag** | Diferença entre último evento produzido e último consumido |
| **Processing time** | Tempo de processamento por mensagem — se aumentando, consumer sobrecarregado |
| **Error rate** | Taxa de erros — pode indicar sobrecarga |
| **CPU/Memory do consumer** | Saturação de recursos |
| **Rejection rate** | Taxa de 429/503 — quantas requests sendo rejeitadas |

---

## Exemplo Conceitual (Pseudocódigo)

```
// Consumer com backpressure baseado em capacidade
class BackpressureConsumer:
    maxConcurrent = 100           // máximo de processamentos simultâneos
    currentActive = AtomicCounter(0)
    
    function onMessage(message):
        if currentActive.get() >= maxConcurrent:
            // Backpressure: não consome mais até liberar
            message.nack()        // devolve a mensagem (ou para de poll)
            metrics.increment("backpressure.activated")
            return
        
        currentActive.increment()
        try:
            process(message)
            message.ack()
        catch:
            message.nack()
        finally:
            currentActive.decrement()

// Auto-scaling baseado em queue depth
class AutoScaler:
    function evaluate():
        depth = queue.getDepth()
        consumers = getConsumerCount()
        
        if depth > HIGH_THRESHOLD:
            scaleUp(consumers + 2)
        elif depth < LOW_THRESHOLD and consumers > MIN_CONSUMERS:
            scaleDown(consumers - 1)
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Buffer ilimitado | Memória esgota; latência cresce infinitamente | Buffer com tamanho máximo + drop ou reject |
| Ignorar o lag | Consumer cada vez mais atrasado sem ação | Monitore lag e configure auto-scaling |
| Retry agressivo em 429/503 | Amplifica a carga no sistema sobrecarregado | Respeite Retry-After; use exponential backoff |
| Consumer sem limite de concorrência | Thread explosion; OOM | Use pool de threads/workers com tamanho fixo |
| Tratar tudo com mesma prioridade | Load shedding descarta requests importantes | Classifique por prioridade; proteja fluxos críticos |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Rate Limiter** | Rate Limiter aplica backpressure no producer (rejeita excesso) |
| **Bulkhead** | Bulkhead limita concorrência por recurso = backpressure localizado |
| **Circuit Breaker** | CB interrompe chamadas quando downstream está sobrecarregado |
| **DLQ** | Mensagens que não podem ser processadas durante backpressure → DLQ |
| **Sharding/Partitioning** | Mais partições = mais consumers = mais capacidade de absorção |

---

## Boas Práticas

1. **Monitore queue depth e consumer lag** — são os primeiros indicadores de sobrecarga.
2. Use **pull model** (consumer puxa mensagens) em vez de push (broker empurra).
3. Defina **limites de buffer** — buffers ilimitados são bomba-relógio.
4. Implemente **auto-scaling** baseado em métricas de backlog.
5. Use **load shedding** para proteger fluxos críticos durante picos.
6. **Respeite sinais** de backpressure (429, Retry-After, nack).
7. Em streaming, use **request(n)** para controlar a taxa.
8. Backpressure **temporário** é normal; backpressure **contínuo** exige mais consumers ou otimização.

---

## Referências

- Martin Kleppmann — *Designing Data-Intensive Applications* — Backpressure and Flow Control
- Reactive Streams — [Specification](https://www.reactive-streams.org/)
- Jay Kreps — [The Log: What every software engineer should know](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
- AWS — [Controlling concurrency in distributed systems](https://aws.amazon.com/builders-library/controlling-concurrency-in-distributed-systems/)
