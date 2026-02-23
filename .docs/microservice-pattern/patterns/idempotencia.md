# Idempotência

> **Categoria:** Persistência e Processamento de Dados
> **Complementa:** Retry, DLQ, Saga

---

## Problema

Em sistemas distribuídos, a **mesma operação pode ser executada mais de uma vez**:

- O cliente fez retry após timeout (mas o servidor já processou a primeira chamada)
- O message broker entregou a mesma mensagem duas vezes (at-least-once delivery)
- O load balancer redirecionou e o request foi duplicado
- Um job agendado executou duas vezes por race condition

Sem idempotência, essas duplicações causam efeitos colaterais: pagamentos cobrados duas vezes, emails enviados em duplicata, registros duplicados no banco.

---

## Solução

Garantir que **processar a mesma operação N vezes** produz **exatamente o mesmo resultado** que processá-la 1 vez. A segunda (e terceira, e enésima) execução é detectada e tratada como no-op ou retorna o resultado da primeira execução.

---

## Idempotência Natural vs. Artificial

### Operações Naturalmente Idempotentes

Algumas operações são idempotentes **por definição** — não precisam de tratamento especial:

| Operação | Por que é idempotente |
|---------|----------------------|
| **GET** /users/123 | Leitura pura — não muda estado |
| **PUT** /users/123 `{name: "João"}` | Substitui o recurso inteiro pelo mesmo valor |
| **DELETE** /users/123 | Deletar algo que já foi deletado = no-op |
| **SET** key=value (cache) | Sobrescreve com o mesmo valor |

### Operações que PRECISAM de tratamento

| Operação | Por que NÃO é idempotente |
|---------|---------------------------|
| **POST** /payments `{amount: 100}` | Cada chamada cria um novo pagamento |
| **INCREMENT** balance += 100 | Cada chamada incrementa novamente |
| **SEND** email to user | Cada chamada envia outro email |
| **PUBLISH** event to queue | Cada chamada publica outra mensagem |

---

## Estratégia Principal: Idempotency Key

O **cliente** gera uma chave única (UUID) e envia junto com a requisição. O servidor verifica se essa chave já foi processada:

```
Primeira chamada:
  POST /v1/payments
  Header: Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
  → Servidor: chave nova → processa → salva chave + resultado → retorna 201

Segunda chamada (retry):
  POST /v1/payments
  Header: Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
  → Servidor: chave já existe → retorna resultado cacheado → retorna 201
```

---

## Fluxo Detalhado

```
Request com Idempotency-Key
          │
          ▼
┌── Chave já existe no store? ──┐
│                                │
│ SIM                            │ NÃO
│                                │
▼                                ▼
Retorna resultado              Processa a operação
cacheado da primeira           (dentro de uma transação)
execução                              │
                                      ▼
                               ┌── Sucesso? ──┐
                               │              │
                               │ SIM          │ NÃO
                               │              │
                               ▼              ▼
                         Salva chave +    Não salva chave
                         resultado no     (permite retry)
                         store (mesma
                         transação!)
                               │
                               ▼
                         Retorna resultado
```

---

## Armazenamento da Chave

A chave de idempotência deve ser armazenada em uma **tabela dedicada** ou em um **cache distribuído**:

### Tabela no Banco de Dados

```
Tabela: idempotency_keys
┌──────────────────────────────────────────────────┐
│ id           │ chave única (PK ou UNIQUE)        │
│ idempotency_key │ a chave enviada pelo client    │
│ http_method  │ POST, PUT                         │
│ request_path │ /v1/payments                      │
│ response_status │ 201                            │
│ response_body │ {"paymentId": "abc123", ...}     │
│ created_at   │ timestamp de criação              │
│ expires_at   │ TTL para limpeza automática       │
└──────────────────────────────────────────────────┘
```

**Ponto crítico:** A chave deve ser salva na **mesma transação** que a operação de negócio. Se a operação é salvar um pagamento e registrar a chave, ambos devem estar no mesmo COMMIT:

```
BEGIN TRANSACTION
  1. INSERT INTO payments (...) VALUES (...)
  2. INSERT INTO idempotency_keys (...) VALUES (...)
COMMIT
```

Se salvar em transações separadas, há uma janela onde a operação foi executada mas a chave não foi registrada — e um retry causaria duplicação.

### Cache Distribuído (Redis)

Para cenários de alta performance onde a tabela no banco seria gargalo:

```
SET idempotency:550e8400 "{response}" EX 86400   // TTL de 24h
```

**Trade-off:** Redis é mais rápido, mas se o Redis falhar, a proteção se perde. Banco é mais confiável.

---

## Idempotência em Mensageria

Para consumers de mensagens (Kafka, RabbitMQ, SQS), use o **message ID** ou **correlation ID** como chave de idempotência:

```
Mensagem Kafka:
  key: "order-123"
  headers:
    message-id: "msg-550e8400"
    correlation-id: "corr-abc123"

Consumer:
  1. Verifica se msg-550e8400 já foi processada
  2. Se sim → skip (acknowledge sem processar)
  3. Se não → processa + registra msg-550e8400
```

**Kafka Exactly-Once:** Kafka suporta exactly-once semantics (EOS) com transações, mas isso é complexo e tem overhead. Na prática, a maioria dos sistemas usa **at-least-once + idempotência no consumer**.

---

## TTL (Time-to-Live) da Chave

As chaves de idempotência devem ter **prazo de expiração**:

| Cenário | TTL Recomendado |
|---------|----------------|
| APIs HTTP (retry do client) | 24h–48h |
| Mensageria (retry automático) | 7 dias |
| Processamento batch | Até o próximo batch |
| Pagamentos (compliance) | 30 dias |

Após o TTL, a chave é removida. Se o mesmo request chegar **depois do TTL**, será processado como novo. Isso é aceitável na maioria dos cenários.

---

## Exemplo Conceitual (Pseudocódigo)

```
function processPayment(idempotencyKey, paymentRequest):
    // 1. Verifica se já processou
    existing = idempotencyStore.find(idempotencyKey)
    if existing:
        return existing.cachedResponse  // retorna resultado anterior
    
    // 2. Processa na mesma transação
    transaction.begin()
    try:
        payment = paymentService.charge(paymentRequest)
        
        idempotencyStore.save(
            key: idempotencyKey,
            response: payment.toResponse(),
            expiresAt: now + 24hours
        )
        
        transaction.commit()
        return payment.toResponse()
    catch:
        transaction.rollback()
        throw  // NÃO salva a chave — permite retry
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Servidor gera a chave | Cada chamada gera chave diferente — sem deduplicação | O **cliente** deve gerar a chave |
| Chave salva em transação separada | Janela de inconsistência entre operação e chave | Mesma transação |
| Sem TTL | Tabela cresce indefinidamente | Defina TTL e rotina de limpeza |
| Idempotência só no HTTP | Consumers Kafka/SQS não são protegidos | Aplique em todas as interfaces |
| Chave por request inteiro | Requests com payloads diferentes mas mesma chave retornam resultado errado | Valide que o payload é o mesmo |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Retry** | Retry **exige** idempotência — sem ela, retry causa duplicação |
| **DLQ** | Mensagens reprocessadas da DLQ devem ser idempotentes |
| **Saga** | Cada step da saga deve ser idempotente (pode ser re-executado) |
| **Outbox** | O publisher da outbox pode republicar — consumer deve ser idempotente |
| **Event Sourcing** | Eventos duplicados devem ser detectados (idempotência por event ID) |

---

## Boas Práticas

1. O **cliente** gera a `Idempotency-Key` (UUID v4 ou v7).
2. Armazene a chave na **mesma transação** que o processamento.
3. Defina **TTL** para limpeza automática das chaves.
4. Para mensageria, use o `message-id` ou `correlation-id` como chave.
5. Em `PUT` e `DELETE`, a idempotência é geralmente implícita (mesmo recurso, mesmo resultado).
6. **Valide o payload** — se a chave é a mesma mas o payload é diferente, retorne erro 409 (Conflict) ou 422.
7. Na resposta de uma chamada idempotente detectada, retorne o **mesmo status code e body** da primeira execução.
8. Implemente um **job de limpeza** periódico para remover chaves expiradas.

---

## Referências

- Stripe — [Idempotency](https://stripe.com/docs/api/idempotent_requests)
- AWS — [Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- Pat Helland — *Idempotence Is Not a Medical Condition* (ACM Queue)
