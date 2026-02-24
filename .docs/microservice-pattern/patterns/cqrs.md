# CQRS (Command Query Responsibility Segregation)

> **Categoria:** Persistência e Processamento de Dados
> **Origem:** Greg Young (2010), baseado no CQS de Bertrand Meyer
> **Complementa:** Event Sourcing, Outbox Pattern, API Composition
> **Keywords:** CQRS, Command Query Responsibility Segregation, separação leitura escrita, read model, write model, projeção

---

## Problema

O modelo de dados otimizado para **escrita** (normalizado, consistente, com constraints) é ineficiente para **leitura** (queries complexas, joins, agregações, ordenações). E vice-versa.

Em sistemas com volumes de leitura extremamente maiores que escrita (ou vice-versa), usar o mesmo modelo para ambas as operações cria **gargalos**:

- Queries complexas degradam a performance de escrita (locks, I/O)
- Índices necessários para leitura impactam a velocidade de escrita
- Escalar leitura e escrita juntas é ineficiente (proporções diferentes)

---

## Solução

Separar os modelos de **Command** (escrita) e **Query** (leitura), permitindo otimizar, escalar e evoluir cada um independentemente.

```
                 ┌── Command (escrita) ──▶ Write Model (normalizado)
                 │                                │
Client ──▶ API  │                                ▼ (evento de sincronização)
                 │                           Projeção/Sync
                 │                                │
                 └── Query (leitura) ──▶ Read Model (desnormalizado, otimizado)
```

---

## Níveis de Separação

CQRS não é "tudo ou nada". Há níveis crescentes de separação:

### Nível 1: Separação de Código (mesmo banco)

Modelos e services de leitura e escrita são separados no código, mas usam o **mesmo banco de dados**:

```
┌────────────────────────────────────────────────┐
│                Code Layer                      │
│                                                │
│  ┌─────────────────┐  ┌────────────────────┐   │
│  │ CommandService   │  │ QueryService       │   │
│  │ (create, update) │  │ (find, search)     │   │
│  └────────┬─────────┘  └─────────┬──────────┘   │
│           │                      │               │
│           └──────────┬───────────┘               │
│                      │                           │
│              ┌───────▼────────┐                  │
│              │  Mesmo Banco   │                  │
│              │  (MySQL)       │                  │
│              └────────────────┘                  │
└────────────────────────────────────────────────┘
```

**Vantagem:** Simples, sem sincronização. **Desvantagem:** Mesmas limitações de performance do modelo único.

### Nível 2: Modelos Diferentes (mesmo banco)

Tabelas diferentes para escrita (normalizada) e leitura (view materializada ou tabela desnormalizada):

```
┌────────────────────────────────────────────────┐
│  CommandService ──▶ tb_order (normalizada)      │
│                          │                      │
│                     sync (trigger, event)        │
│                          │                      │
│  QueryService ──▶ vw_order_summary (view/tabela)│
└────────────────────────────────────────────────┘
```

### Nível 3: Bancos Separados (full CQRS)

Banco de escrita e banco de leitura são **diferentes**, sincronizados via eventos:

```
CommandService ──▶ Write DB (MySQL normalizado)
                         │
                    evento (async)
                         │
                         ▼
                   Projeção/Sync
                         │
                         ▼
QueryService ──▶ Read DB (Elasticsearch / Redis / Materialized View)
```

**Vantagem:** Escala independente, modelos totalmente otimizados. **Desvantagem:** Consistência eventual, complexidade de sincronização.

---

## Command vs. Query

| Aspecto | Command (Escrita) | Query (Leitura) |
|---------|-------------------|-----------------|
| **Operações** | Create, Update, Delete | Find, Search, List, Aggregate |
| **Modelo** | Normalizado, com constraints e validações | Desnormalizado, otimizado para queries |
| **Consistência** | Forte (ACID) | Eventual (pode estar defasado por milissegundos) |
| **Banco** | Relacional (MySQL, PostgreSQL) | Relacional, Cache (Redis), Search (Elasticsearch), etc. |
| **Escala** | Vertical (ou read replicas) | Horizontal (múltiplas instâncias, sharding) |
| **Volume** | Geralmente menor (ex: 100 writes/s) | Geralmente muito maior (ex: 10.000 reads/s) |

---

## Sincronização do Modelo de Leitura

| Mecanismo | Latência | Confiabilidade | Complexidade |
|-----------|---------|---------------|-------------|
| **Event interno (in-process)** | Milissegundos | Alta (mesma transação) | Baixa |
| **Message broker (Kafka, RabbitMQ)** | Milissegundos a segundos | Alta (com at-least-once) | Média |
| **CDC (Change Data Capture)** | Sub-segundo | Alta (lê do binlog) | Média-Alta |
| **Polling** | Segundos | Média | Baixa |
| **Materialized View (banco)** | Tempo real | Alta | Baixa (quando suportado) |

### Fluxo com Eventos

```
1. CommandService salva no Write DB + publica evento
2. Projection Service consome o evento
3. Projection Service atualiza o Read DB (modelo desnormalizado)
4. QueryService lê do Read DB

Timeline:
  t=0ms:   Command salva no Write DB
  t=5ms:   Evento publicado
  t=50ms:  Projection processa e atualiza Read DB
  t=51ms+: Query retorna dado atualizado
```

**Janela de inconsistência:** Entre t=0ms e t=50ms, o dado está no Write DB mas ainda não no Read DB. Isso é **consistência eventual** — aceitável na grande maioria dos cenários.

---

## Modelo de Leitura: Desnormalização

O modelo de leitura é intencionalmente **desnormalizado** — otimizado para as queries que o frontend precisa:

```
Write Model (normalizado):
  tb_order:    id, user_id, status, created_at
  tb_order_item: id, order_id, product_id, quantity, price
  tb_user:     id, name, email
  tb_product:  id, name, category

Read Model (desnormalizado):
  order_summary:
    order_id, user_name, user_email, status, 
    total_items, total_amount, product_names, 
    created_at
    
  → UMA query retorna tudo, ZERO joins
```

---

## Diagrama de Decisão: Preciso de CQRS?

```
Leitura e escrita têm volumes MUITO diferentes? ───────────── NÃO ── CRUD simples
           │
          SIM
           │
           ▼
Modelo de leitura precisa ser diferente do de escrita? ──── NÃO ── Read replicas
           │
          SIM
           │
           ▼
Consistência eventual é aceitável? ────────────────────── NÃO ── Nível 1 ou 2
           │                                                      (mesmo banco)
          SIM
           │
           ▼
Full CQRS (bancos separados, sync via eventos)
```

---

## Exemplo Conceitual (Pseudocódigo)

```
// COMMAND SIDE
class OrderCommandService:
    orderRepository      // write DB
    eventPublisher       // publica eventos
    
    function createOrder(request):
        order = Order.create(request)
        orderRepository.save(order)
        
        // Publica evento para sincronizar read model
        eventPublisher.publish(OrderCreatedEvent(
            orderId: order.id,
            userId: order.userId,
            items: order.items,
            total: order.total
        ))
        
        return order.id

// QUERY SIDE
class OrderQueryService:
    orderReadRepository  // read DB (desnormalizado)
    
    function findAll(pagination):
        return orderReadRepository.findAllSummaries(pagination)
    
    function findById(id):
        return orderReadRepository.findSummaryById(id)

// PROJECTION (sincronização)
class OrderProjectionUpdater:
    orderReadRepository
    
    function onOrderCreated(event):
        summary = OrderSummary(
            orderId: event.orderId,
            userName: event.userName,    // já desnormalizado no evento
            totalItems: event.items.size,
            totalAmount: event.total,
            status: "CREATED"
        )
        orderReadRepository.saveSummary(summary)
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| CQRS em CRUD simples | Complexidade desnecessária | Use CQRS quando leitura/escrita têm requisitos realmente diferentes |
| Consistência forte no Read Model | Anula o benefício de bancos separados | Aceite consistência eventual |
| Projeção que consulta Write DB | Acoplamento entre read e write | Projeção deve receber tudo no evento |
| Sem tratamento de inconsistência | UI mostra dado obsoleto sem aviso | Indique ao usuário que dados podem estar defasados |
| Modelo de leitura = cópia do write | Não otimiza nada; só aumenta complexidade | Desnormalize o read model para as queries reais |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Event Sourcing** | CQRS + Event Sourcing: eventos são o write model; projeções são o read model |
| **Outbox Pattern** | Garante que comando + evento são consistentes |
| **CDC** | Alternativa para sincronizar write→read sem publicar eventos explícitos |
| **API Composition** | API Composition consulta em tempo real; CQRS pré-materializa |
| **Saga** | Sagas geram eventos que podem atualizar read models |

---

## Boas Práticas

1. CQRS **não requer** bancos separados — comece separando **modelos e services** no código.
2. Use **consistência eventual** e aceite a janela de defasagem.
3. O modelo de leitura deve ser **desnormalizado** para as queries do frontend.
4. Combine com **Event Sourcing** quando precisar do histórico completo de mudanças.
5. Comece **simples** — Nível 1 (mesmo banco, services separados) → evolua conforme necessidade.
6. Monitore o **atraso da projeção** (lag entre write e read).
7. **Eventos devem carregar dados suficientes** — a projeção não deve precisar consultar o write DB.
8. Separe **endpoints de command** (POST, PUT, DELETE) de **endpoints de query** (GET).

---

## Referências

- Greg Young — [CQRS Documents](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)
- Martin Fowler — [CQRS](https://martinfowler.com/bliki/CQRS.html)
- Microsoft — [CQRS Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- Chris Richardson — *Microservices Patterns* (Manning) — Capítulo 7: Implementing Queries in a Microservice Architecture
