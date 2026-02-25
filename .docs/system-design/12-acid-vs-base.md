# 12. ACID vs BASE

> **Categoria:** Fundamentos de Banco de Dados Distribuídos  
> **Nível:** Essencial para qualquer entrevista de System Design  
> **Complexidade:** Média

---

## Definição

**ACID** e **BASE** são dois modelos de consistência para sistemas de banco de dados que representam extremos opostos de um espectro:

- **ACID** — Foco em **corretude e consistência forte** (bancos SQL/transacionais)
- **BASE** — Foco em **disponibilidade e performance** (bancos NoSQL/distribuídos)

A escolha entre eles define como seu sistema se comporta sob falhas, alta carga e em cenários distribuídos.

---

## Por Que é Importante?

- **Define o modelo mental** de como dados se comportam no sistema
- **Impacta diretamente** a confiabilidade de operações financeiras, inventário, etc.
- **Trade-off fundamental** entre performance/escala e corretude/simplicidade
- **Pergunta clássica** em entrevistas de system design e backend

---

## Diagrama Comparativo

```
    ┌─────────────────────────────────────────────────────┐
    │                SPECTRUM DE CONSISTÊNCIA              │
    │                                                     │
    │  ACID ◄──────────────────────────────────► BASE     │
    │                                                     │
    │  ● Strong consistency    ● Eventual consistency     │
    │  ● Pessimistic locking   ● Optimistic approach      │
    │  ● Vertical scaling      ● Horizontal scaling       │
    │  ● Single-node friendly  ● Distributed friendly     │
    │  ● Lower throughput      ● Higher throughput        │
    │  ● Higher latency        ● Lower latency           │
    └─────────────────────────────────────────────────────┘
```

---

## ACID (SQL / Transacional)

### Propriedades Detalhadas

| Propriedade | Significado | Exemplo |
|-------------|-------------|---------|
| **Atomicity** | Transação é "tudo ou nada" — se qualquer parte falha, tudo é revertido (rollback) | Transferência: debita + credita ou nenhum dos dois |
| **Consistency** | Dados sempre em estado válido antes e depois da transação (constraints, triggers) | Saldo nunca fica negativo se há constraint CHECK |
| **Isolation** | Transações concorrentes não interferem entre si | Dois saques simultâneos não causam saldo incorreto |
| **Durability** | Dados commitados sobrevivem a crash do sistema (write-ahead log) | Após COMMIT, dados persistem mesmo com queda de energia |

### Atomicity em Detalhe

```
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 'A';  -- ✓
  UPDATE accounts SET balance = balance + 100 WHERE id = 'B';  -- ✗ (erro!)
ROLLBACK;  -- Ambas operações são revertidas

-- Conta A e Conta B voltam ao estado original
-- Nenhum dinheiro é perdido
```

```
Cenário: Transferência de R$100 de A para B

  ┌──────────────────────────────────────┐
  │          Sucesso (COMMIT)            │
  │                                      │
  │  A: 1000 → 900   B: 500 → 600      │
  │  (ambos mudam)                       │
  └──────────────────────────────────────┘

  ┌──────────────────────────────────────┐
  │          Falha (ROLLBACK)            │
  │                                      │
  │  A: 1000 → 1000  B: 500 → 500      │
  │  (nenhum muda)                       │
  └──────────────────────────────────────┘
```

### Isolation Levels

| Nível | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------|------------|---------------------|--------------|-------------|
| **Read Uncommitted** | ✗ Permite | ✗ Permite | ✗ Permite | Mais rápido |
| **Read Committed** | ✓ Previne | ✗ Permite | ✗ Permite | Rápido |
| **Repeatable Read** | ✓ Previne | ✓ Previne | ✗ Permite | Médio |
| **Serializable** | ✓ Previne | ✓ Previne | ✓ Previne | Mais lento |

```
Dirty Read:           Tx1 escreve x=5, Tx2 lê x=5, Tx1 faz rollback
                      → Tx2 leu dado que nunca existiu!

Non-Repeatable Read:  Tx1 lê x=5, Tx2 atualiza x=10 e commita, Tx1 lê x=10
                      → Tx1 leu valores diferentes na mesma transação!

Phantom Read:         Tx1 lê "SELECT * WHERE age > 25" → 5 rows
                      Tx2 insere row com age=30 e commita
                      Tx1 repete query → 6 rows (row fantasma!)
```

### Mecanismos de Implementação ACID

| Mecanismo | Garante | Como |
|-----------|---------|------|
| **WAL (Write-Ahead Log)** | Durability + Atomicity | Grava log ANTES de modificar dados |
| **Undo Log** | Atomicity | Permite reverter operações incompletas |
| **Redo Log** | Durability | Replay de operações após crash |
| **MVCC** | Isolation | Múltiplas versões do mesmo dado para reads concorrentes |
| **2PL (Two-Phase Locking)** | Isolation | Locks de leitura/escrita em duas fases |
| **Checkpoints** | Recovery | Snapshots periódicos para recovery mais rápido |

```
WAL (Write-Ahead Log):

  Client: INSERT INTO orders (...)
       │
       ▼
  ┌────────────┐    1. Grava     ┌──────────┐
  │   Engine   │──────────────▶ │   WAL    │  (log sequencial, fsync)
  │            │                 │ (disco)  │
  │            │    2. Aplica    └──────────┘
  │            │──────────────▶ ┌──────────┐
  │            │                 │  Buffer  │  (memória, batch flush)
  └────────────┘                 │  Pool    │
                                 └──────────┘
  
  Se crash após 1, antes de 2 → Redo a partir do WAL ✓
```

---

## BASE (NoSQL / Distribuído)

### Propriedades Detalhadas

| Propriedade | Significado | Exemplo |
|-------------|-------------|---------|
| **Basically Available** | Sistema sempre responde (pode ser com dados stale) | Cassandra retorna dados mesmo durante partição |
| **Soft State** | Estado do sistema pode mudar ao longo do tempo sem input externo | Réplicas convergem em background |
| **Eventually Consistent** | Dado um tempo suficiente sem writes, todas as réplicas convergem | DynamoDB: read após write pode retornar dado antigo por ms |

### Eventually Consistent em Detalhe

```
  t=0: Write x=5 no Nó A
  
  Nó A: x=5 ───replicate───▶ Nó B: x=5 (propagando...)
                              Nó C: x=1 (ainda antigo)
  
  t=50ms: Propagação parcial
  
  Nó A: x=5
  Nó B: x=5
  Nó C: x=1  ← leitura aqui retorna dado stale!
  
  t=200ms: Convergência completa
  
  Nó A: x=5
  Nó B: x=5
  Nó C: x=5  ← agora todos consistentes ✓
```

### Mecanismos de Convergência

| Mecanismo | Descrição | Usado Em |
|-----------|-----------|----------|
| **Read Repair** | Detecta inconsistência durante leitura e corrige | Cassandra, DynamoDB |
| **Anti-Entropy (Merkle Trees)** | Background sync comparando hashes de dados | Cassandra, Riak |
| **Hinted Handoff** | Nó intermediário guarda escrita para nó indisponível | Cassandra, DynamoDB |
| **Vector Clocks** | Detecta conflitos usando relógios lógicos causais | DynamoDB, Riak |
| **CRDTs** | Tipos de dados que convergem automaticamente sem coordenação | Redis, Riak |
| **Last Writer Wins (LWW)** | Timestamp mais recente vence conflito | Cassandra |

```
Read Repair:

  Client: READ(key=user:123)
     │
     ├──▶ Nó A: {name: "João", v=3}  ← mais recente
     ├──▶ Nó B: {name: "João", v=3}
     └──▶ Nó C: {name: "John", v=2}  ← desatualizado!
     
  Coordinator detecta v=2 < v=3
     │
     └──▶ Nó C: UPDATE para v=3  ← read repair!
     
  Retorna ao client: {name: "João", v=3}
```

---

## Comparação Direta

| Aspecto | ACID | BASE |
|---------|------|------|
| **Foco** | Corretude | Disponibilidade |
| **Consistência** | Forte (imediata) | Eventual (convergente) |
| **Disponibilidade** | Pode recusar requests | Sempre responde |
| **Escala** | Vertical (scale-up) | Horizontal (scale-out) |
| **Performance** | Menor throughput | Maior throughput |
| **Latência** | Maior (locks, consensus) | Menor (no coordination) |
| **Conflitos** | Prevenidos (locking) | Detectados e resolvidos |
| **Implementação** | Mais simples de raciocinar | Mais complexa |
| **Tipo de Banco** | PostgreSQL, MySQL, Oracle | Cassandra, DynamoDB, MongoDB |
| **CAP** | CP (tipicamente) | AP (tipicamente) |

---

## Quando Usar Cada Um

### ACID — Quando Corretude é Crítica

```
✅ ACID é a escolha certa para:

  ┌────────────────────────────────────────────┐
  │  💰 Transações financeiras                  │
  │     - Transferências bancárias              │
  │     - Processamento de pagamentos           │
  │     - Débito/crédito de contas             │
  │                                             │
  │  📦 Gestão de inventário                    │
  │     - Controle de estoque                   │
  │     - Reserva de produtos                   │
  │                                             │
  │  🎫 Sistema de reservas                     │
  │     - Passagens aéreas                      │
  │     - Ingressos de eventos                  │
  │     - Agendamento médico                    │
  │                                             │
  │  👤 Dados de usuário                        │
  │     - Autenticação/autorização              │
  │     - Perfil e configurações                │
  └────────────────────────────────────────────┘
```

### BASE — Quando Disponibilidade e Escala São Prioridade

```
✅ BASE é a escolha certa para:

  ┌────────────────────────────────────────────┐
  │  📱 Social media feeds                      │
  │     - Timeline, posts, likes               │
  │     - Contadores de views                   │
  │                                             │
  │  📊 Analytics e métricas                    │
  │     - Logs de acesso                        │
  │     - Dados de telemetria                   │
  │     - Tracking de eventos                   │
  │                                             │
  │  🛒 Catálogo de produtos                    │
  │     - Listagens (reads pesados)            │
  │     - Reviews e ratings                     │
  │                                             │
  │  💬 Messaging e notificações               │
  │     - Chat (entrega at-least-once)         │
  │     - Push notifications                    │
  │     - Activity feeds                        │
  │                                             │
  │  🔍 Search indexes                          │
  │     - Full-text search                      │
  │     - Recomendações                         │
  └────────────────────────────────────────────┘
```

---

## Padrões Híbridos

Na prática, sistemas reais usam **ambos**:

### Saga Pattern (ACID distribuído via BASE)

```
Saga: Pedido de E-Commerce

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Create   │───▶│ Reserve  │───▶│ Process  │───▶│  Ship    │
  │ Order    │    │ Stock    │    │ Payment  │    │  Order   │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘
       │               │               │               │
       ▼               ▼               ▼               ▼
  (compensate)    (compensate)    (compensate)     (complete)
  Cancel Order    Release Stock   Refund Payment
  
  Cada step é ACID local, o workflow inteiro é BASE (eventual consistency)
```

### Event Sourcing + CQRS

```
  ┌────────────┐          ┌────────────────┐
  │  Commands  │─── ACID ─│ Event Store    │
  │  (writes)  │          │ (append-only)  │
  └────────────┘          └───────┬────────┘
                                  │ events
                           ┌──────▼───────┐
                           │  Projections │─── BASE ── Read Models
                           │  (async)     │            (eventually consistent)
                           └──────────────┘
```

### Transactional Outbox

```
  ┌─────────────────────────────────────┐
  │           DB Transaction (ACID)      │
  │                                      │
  │  1. UPDATE orders SET status='paid'  │
  │  2. INSERT INTO outbox (event_data)  │
  │                                      │
  └──────────────────────────────────────┘
               │
               ▼ (async polling / CDC)
  ┌──────────────────────────┐
  │   Message Broker (BASE)  │──▶ Consumers (eventually consistent)
  └──────────────────────────┘
```

---

## Tecnologias e Classificação

| Tecnologia | Modelo | Tipo | Caso de Uso Ideal |
|------------|--------|------|-------------------|
| **PostgreSQL** | ACID | SQL Relacional | Dados transacionais, geoespaciais |
| **MySQL/InnoDB** | ACID | SQL Relacional | Web apps, e-commerce |
| **Oracle** | ACID | SQL Relacional | Enterprise, ERP |
| **SQL Server** | ACID | SQL Relacional | Enterprise, .NET ecosystem |
| **CockroachDB** | ACID | Distributed SQL | ACID distribuído + scale-out |
| **Google Spanner** | ACID | Distributed SQL | Global transactions, Google-scale |
| **Cassandra** | BASE | Wide-Column | Time-series, IoT, high-write |
| **DynamoDB** | BASE | Key-Value+Document | Serverless, variable traffic |
| **MongoDB** | ACID* | Document | Flexible schema (*multi-doc transactions v4.0+) |
| **Redis** | BASE | Key-Value | Cache, sessions, pub/sub |
| **Couchbase** | BASE | Document+KV | Mobile sync, caching |
| **Riak** | BASE | Key-Value | High availability, IoT |

---

## Uso em Big Techs

### Amazon — Estratégia Dual
- **Transações financeiras:** Aurora (MySQL/PostgreSQL compatível) — ACID completo
- **Carrinho de compras / catálogo:** DynamoDB — BASE, alta disponibilidade
- **Princípio:** "O custo de indisponibilidade é maior que inconsistência temporária para o carrinho"

### Google — Spanner (ACID + Escala Global)
- **Inovação:** Primeiro banco a oferecer ACID com escala horizontal global
- **TrueTime:** Relógios atômicos + GPS para timestamps globais consistentes
- **Uso:** AdWords, Google Play — tabelas com trilhões de rows
- **SLA:** 99.999% availability com serializable transactions

### Netflix — Abordagem por Domínio
- **Billing:** MySQL (ACID) — cobrança exige corretude absoluta
- **Catálogo/metadata:** Cassandra (BASE) — disponibilidade é prioridade
- **Viewing activity:** Cassandra (BASE) — high-write, eventual consistency OK
- **A/B testing:** Cassandra + custom merge logic
- **Princípio:** *"CAP trade-off é decidido por bounded context"*

### Uber — Event Sourcing Híbrido
- **Trip data:** Eventual consistency (aceita milissegundos de atraso)
- **Payments:** ACID local com Saga pattern para coordenação
- **Schemaless:** Banco interno que combina MySQL (ACID) com Cassandra-like API

### Stripe — ACID-First
- **Toda operação de pagamento:** PostgreSQL com ACID rigoroso
- **Idempotency keys:** Garante que retries não criam transações duplicadas
- **Double-entry bookkeeping:** Cada transação é um débito + crédito (always balanced)
- **Eventual consistency:** Apenas para webhooks e eventos assíncronos

---

## Perguntas Comuns em Entrevistas

1. **Explique a diferença entre ACID e BASE com exemplos.**
   - ACID = banco (transferência tudo-ou-nada); BASE = feed social (posts podem aparecer fora de ordem)

2. **Quando você escolheria BASE sobre ACID?**
   - Quando disponibilidade e escala são mais importantes que consistência imediata

3. **O que é eventual consistency e quais os riscos?**
   - Dados convergem ao longo do tempo; risco de leituras stale durante a janela de convergência

4. **Como implementar transações distribuídas sem ACID global?**
   - Saga pattern, event sourcing, transactional outbox

5. **MongoDB é ACID ou BASE?**
   - Ambos: single-document ACID nativo, multi-document ACID desde v4.0, mas sem as garantias de um banco relacional em escala

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Modelo** | ACID (corretude garantida) | BASE (escala e disponibilidade) |
| **Escala** | Vertical, limites físicos | Horizontal, teoricamente ilimitada |
| **Complexidade** | Simples de raciocinar | Conflict resolution necessário |
| **Latência** | Maior (locks, consensus) | Menor (no coordination) |
| **Throughput** | Limitado pelo lock contention | Alto (parallel writes) |
| **Recovery** | WAL + rollback automático | Read repair + anti-entropy |
| **Transações dist.** | 2PC (blocking, frágil) | Saga (eventually consistent) |

---

## Referências

- [Martin Kleppmann — Designing Data-Intensive Applications, Cap. 7: Transactions](https://dataintensive.net/)
- [Google Spanner Paper](https://research.google/pubs/pub39966/)
- [Amazon Dynamo Paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Saga Pattern — Chris Richardson](https://microservices.io/patterns/data/saga.html)
- [PostgreSQL Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- System Design Interview — Alex Xu, Vol. 1
