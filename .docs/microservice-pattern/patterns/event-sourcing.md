# Event Sourcing

> **Categoria:** Event-Driven e Mensageria
> **Origem:** Martin Fowler, Greg Young
> **Complementa:** CQRS, Outbox Pattern, Saga

---

## Problema

Em sistemas tradicionais, armazenamos o **estado atual** de uma entidade:

```
tb_account: id=1, balance=1500
```

Perdemos **como** chegamos a esse estado. Quem depositou? Quando? Quanto? Houve estorno? Não sabemos — apenas o saldo final.

Isso é problemático quando:
- Precisamos de **auditoria completa** (compliance, regulação)
- Precisamos **reconstruir o estado** em um ponto no passado (debugging, investigação)
- Precisamos **reprocessar eventos** (migração, correção de bug)
- Múltiplos serviços precisam **reagir** a mudanças de estado

---

## Solução

Armazenar **todos os eventos** (fatos) que aconteceram com uma entidade, em ordem. O estado atual é **derivado** da sequência de eventos, não armazenado diretamente.

```
Tradicional:  tb_account → balance = 1500  (apenas o estado final)

Event Sourcing:
  Event 1: AccountCreated(id=1, balance=0)
  Event 2: MoneyDeposited(id=1, amount=2000)
  Event 3: MoneyWithdrawn(id=1, amount=500)
  Event 4: MoneyDeposited(id=1, amount=200)
  Event 5: FeeCharged(id=1, amount=200)
  
  Estado atual: 0 + 2000 - 500 + 200 - 200 = 1500 ✅
```

---

## Conceitos Fundamentais

### Event Store

O **Event Store** é o banco de dados (append-only) que armazena todos os eventos:

```
Tabela: event_store
┌──────────────────────────────────────────────────────────────┐
│ event_id        │ UUID (PK)                                 │
│ aggregate_id    │ UUID do agregado (ex: account-123)        │
│ aggregate_type  │ "Account"                                 │
│ event_type      │ "MoneyDeposited"                          │
│ event_data      │ {"amount": 2000, "source": "transfer"}    │
│ metadata        │ {"userId": "u-456", "correlationId": ...} │
│ version         │ 3 (sequencial por aggregate)              │
│ created_at      │ 2026-02-23T10:30:00Z                      │
└──────────────────────────────────────────────────────────────┘
```

**Regras do Event Store:**
- **Append-only** — eventos nunca são alterados ou deletados
- **Ordenado** — cada evento tem uma versão (sequência) dentro do seu agregado
- **Imutável** — uma vez persistido, o evento é um fato histórico

### Aggregate (Agregado)

O **agregado** é a entidade de domínio cujo estado é construído a partir dos eventos:

```
class Account:
    id
    balance
    status
    
    function apply(event):
        match event.type:
            "AccountCreated":   balance = 0, status = "ACTIVE"
            "MoneyDeposited":   balance += event.amount
            "MoneyWithdrawn":   balance -= event.amount
            "AccountClosed":    status = "CLOSED"
```

### Reconstrução de Estado (Replay)

Para obter o estado atual, **carregamos todos os eventos** do agregado e aplicamos na ordem:

```
function loadAccount(accountId):
    events = eventStore.findByAggregateId(accountId)  // todos os eventos
    account = new Account()
    
    for event in events:
        account.apply(event)  // aplica cada evento em ordem
    
    return account  // estado atual reconstruído
```

---

## Fluxo Completo

```
1. Command chega:  DepositMoney(accountId=1, amount=200)
2. Carrega estado:  replay dos eventos → balance = 1300
3. Valida:          balance >= 0? ✅ (regras de negócio)
4. Cria evento:     MoneyDeposited(id=1, amount=200)
5. Persiste:        INSERT INTO event_store (...)
6. Aplica:          balance = 1300 + 200 = 1500
7. Publica:         event broker → consumers reagem
```

---

## Snapshots

Se um agregado tem **milhares de eventos**, reconstruir do zero a cada operação é lento. **Snapshots** resolvem isso:

```
Sem snapshot:
  Replay: Event1 → Event2 → ... → Event10000 → Estado atual
  (10.000 eventos para processar)

Com snapshot:
  Snapshot@version=9990: { balance: 14500, status: ACTIVE }
  Replay: Snapshot + Event9991 → ... → Event10000
  (apenas 10 eventos para processar)
```

### Quando criar Snapshot

| Estratégia | Descrição |
|-----------|-----------|
| **A cada N eventos** | Snapshot a cada 100 ou 1000 eventos |
| **Periódico** | Snapshot a cada hora/dia |
| **Sob demanda** | Quando a reconstrução ultrapassar threshold de latência |

```
Tabela: event_snapshots
┌──────────────────────────────────────────────────┐
│ aggregate_id  │ UUID                             │
│ version       │ 9990 (versão do último evento)   │
│ state         │ {"balance": 14500, "status": ...}│
│ created_at    │ 2026-02-23T10:30:00Z             │
└──────────────────────────────────────────────────┘
```

---

## Event Sourcing + CQRS

Event Sourcing combina naturalmente com CQRS:

```
┌──────────────┐              ┌──────────────┐
│ Command Side │              │  Query Side  │
│              │              │              │
│ Commands ────▶ Event Store  │  Projections │
│              │ (write)      │  (read)      │
│              │      │       │      ▲       │
│              │      │evento │      │       │
│              │      └───────┼──────┘       │
│              │              │              │
└──────────────┘              └──────────────┘

Write: Event Store (append-only, todos os eventos)
Read:  Projection/Read DB (desnormalizado, derivado dos eventos)
```

- **Event Store = Write Model** — todos os fatos
- **Projections = Read Models** — visões materializadas derivadas dos eventos
- Múltiplas projections podem existir para diferentes necessidades de query

---

## Temporal Query

Com Event Sourcing, é possível **reconstruir o estado em qualquer ponto no tempo**:

```
// Estado em 2026-01-15:
events = eventStore.findByAggregateId(accountId)
                    .filter(e -> e.createdAt <= "2026-01-15")
                    
account = replay(events)  // estado como era em 15/jan
```

Isso é impossível em sistemas tradicionais que armazenam apenas o estado atual.

---

## Exemplo Conceitual (Pseudocódigo)

```
// COMMAND: Depositar dinheiro
class DepositMoneyHandler:
    eventStore
    
    function handle(command):
        // 1. Reconstuir estado
        events = eventStore.findByAggregateId(command.accountId)
        account = Account.replay(events)
        
        // 2. Validar regras
        if account.status != "ACTIVE":
            throw AccountNotActiveException()
        
        // 3. Criar evento
        event = MoneyDeposited(
            aggregateId: command.accountId,
            amount: command.amount,
            version: account.currentVersion + 1
        )
        
        // 4. Persistir (com controle de concorrência via version)
        eventStore.append(event, expectedVersion: account.currentVersion)
        
        // 5. Publicar para consumers/projections
        eventPublisher.publish(event)

// PROJEÇÃO: Atualizar read model
class AccountBalanceProjection:
    readDB
    
    function onMoneyDeposited(event):
        readDB.updateBalance(event.aggregateId, 
                            balance += event.amount)
    
    function onMoneyWithdrawn(event):
        readDB.updateBalance(event.aggregateId, 
                            balance -= event.amount)
```

---

## Controle de Concorrência

Quando dois commands tentam alterar o mesmo agregado simultaneamente:

```
Thread A: carrega Account (version=5) → cria evento (version=6)
Thread B: carrega Account (version=5) → cria evento (version=6) ← CONFLITO!
```

**Solução: Optimistic Concurrency**

```
function append(event, expectedVersion):
    currentVersion = getLatestVersion(event.aggregateId)
    if currentVersion != expectedVersion:
        throw ConcurrencyException("Agregado modificado por outra operação")
    
    // Persiste com a nova versão
    insert(event)
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Alterar/deletar eventos | Perde auditoria e integridade | Eventos são imutáveis — crie eventos compensatórios |
| Evento com dados insuficientes | Projeções precisam consultar outros serviços | Evento deve carregar todos os dados necessários |
| Sem snapshots (muitos eventos) | Reconstrução lenta | Implemente snapshots para agregados com muitos eventos |
| Event Sourcing em tudo | Desnecessariamente complexo para CRUD simples | Use apenas onde auditoria/temporal query é necessário |
| Versionar sem cuidado | Mudança no formato do evento quebra replay | Use versionamento de eventos (upcasting) |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **CQRS** | Event Sourcing + CQRS: eventos no write, projeções no read. Par natural. |
| **Outbox Pattern** | Garante que evento é persistido e publicado atomicamente |
| **Saga** | O estado da saga pode ser reconstruído a partir dos eventos |
| **CDC** | Alternativa ao Event Sourcing: captura mudanças no banco em vez de eventos explícitos |
| **Idempotência** | Consumers/projeções devem ser idempotentes (eventos podem ser reprocessados) |

---

## Boas Práticas

1. Eventos são **imutáveis** — nunca altere ou delete.
2. Nomeie eventos no **passado** (MoneyDeposited, OrderCreated — fatos que aconteceram).
3. Eventos devem ser **auto-contidos** — carregar dados suficientes para projeções.
4. Implemente **snapshots** quando agregados acumulam muitos eventos.
5. Use **versionamento** de eventos para evolução do schema.
6. Combine com **CQRS** — Event Store para escrita, projeções para leitura.
7. Monitore o **lag das projeções** (tempo entre evento e projeção atualizada).
8. Event Sourcing é ideal para domínios com **requisitos de auditoria, compliance ou temporal queries**.

---

## Referências

- Martin Fowler — [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- Greg Young — [Event Sourcing](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)
- Microsoft — [Event Sourcing Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- Vaughn Vernon — *Implementing Domain-Driven Design* (DDD + Event Sourcing)
