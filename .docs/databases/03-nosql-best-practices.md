# Databases — NoSQL — Boas Práticas

> **Objetivo deste documento:** Servir como referência completa sobre **boas práticas para bancos de dados NoSQL**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Cobre Document Stores, Key-Value, Wide-Column, Graph e bancos especializados — modelagem, access patterns, consistency tuning e anti-patterns.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **Access Patterns First** | Modele pelos padrões de acesso, não pela entidade | Modelar document DB como relacional | Listar queries antes de modelar |
| **Denormalization** | Dados redundantes são aceitos se reduzem I/O | Normalizar documents com referências excessivas | Embeddar dados frequentemente lidos juntos |
| **Partition Key** | Escolha uma partition key com alta cardinalidade | Partition key com poucos valores (hot partition) | Adicionar sufixo de sharding ou compor chave |
| **Consistency Tuning** | Ajustar consistência por operação, não globalmente | Usar eventual consistency para dados financeiros | Strong read para dados críticos |
| **Schema Validation** | Definir schema explícito mesmo em schemaless DBs | Confiar que "schemaless" dispensa modelagem | JSON Schema, Mongoose schema, validators |
| **TTL** | Definir TTL para dados temporários (sessões, cache) | Dados expirados crescem indefinidamente | TTL nativo do banco ou job de cleanup |

---

## Sumário

- [Databases — NoSQL — Boas Práticas](#databases--nosql--boas-práticas)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Princípios de Modelagem NoSQL](#princípios-de-modelagem-nosql)
    - [Pensando em Access Patterns](#pensando-em-access-patterns)
    - [Embedding vs Referencing](#embedding-vs-referencing)
  - [Document Stores (MongoDB, DocumentDB)](#document-stores-mongodb-documentdb)
    - [Modelagem de Documents](#modelagem-de-documents)
    - [Indexação em MongoDB](#indexação-em-mongodb)
    - [Aggregation Pipeline](#aggregation-pipeline)
    - [Boas Práticas MongoDB](#boas-práticas-mongodb)
  - [Key-Value Stores (Redis, DynamoDB)](#key-value-stores-redis-dynamodb)
    - [DynamoDB — Modelagem Single-Table](#dynamodb--modelagem-single-table)
    - [DynamoDB — Boas Práticas](#dynamodb--boas-práticas)
    - [Redis — Patterns e Boas Práticas](#redis--patterns-e-boas-práticas)
  - [Wide-Column Stores (Cassandra, ScyllaDB)](#wide-column-stores-cassandra-scylladb)
    - [Modelagem Cassandra](#modelagem-cassandra)
    - [Boas Práticas Cassandra](#boas-práticas-cassandra)
  - [Graph Databases (Neo4j, Neptune)](#graph-databases-neo4j-neptune)
    - [Modelagem de Grafos](#modelagem-de-grafos)
    - [Boas Práticas Graph](#boas-práticas-graph)
  - [Consistency Tuning](#consistency-tuning)
  - [Schema Evolution em NoSQL](#schema-evolution-em-nosql)
  - [Anti-patterns NoSQL](#anti-patterns-nosql)
  - [Referências](#referências)

---

## Princípios de Modelagem NoSQL

### Pensando em Access Patterns

```
┌──────────────────────────────────────────────────────────────────┐
│        MODELAGEM NoSQL — ACCESS PATTERNS FIRST                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SQL (Entity-first):                                            │
│  "Quais são as entidades?" → Tabelas → Queries                  │
│                                                                  │
│  NoSQL (Query-first):                                           │
│  "Como os dados serão acessados?" → Modelo → Estrutura          │
│                                                                  │
│  Processo de modelagem NoSQL:                                    │
│                                                                  │
│  1. LISTAR todos os access patterns                             │
│     • Quais queries a aplicação fará?                           │
│     • Qual a frequência de cada uma?                            │
│     • Qual a latência aceitável?                                │
│                                                                  │
│  2. AGRUPAR dados acessados juntos                              │
│     • Dados lidos juntos → armazenar juntos                     │
│     • "Pre-join" no modelo                                      │
│                                                                  │
│  3. DEFINIR chaves de acesso                                    │
│     • Partition key: identifica a partição                      │
│     • Sort key: ordena dentro da partição                       │
│     • Secondary indexes: access patterns alternativos           │
│                                                                  │
│  4. DESNORMALIZAR conforme necessário                           │
│     • Duplicar dados é ACEITÁVEL em NoSQL                       │
│     • O custo de storage << custo de JOINs distribuídos         │
│     • Manter consistência via aplicação                         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Exemplo prático — E-commerce:**

| # | Access Pattern | Frequência | Latência |
|---|---------------|------------|----------|
| 1 | Buscar pedido por ID | Alta | < 10ms |
| 2 | Listar pedidos de um usuário (recentes primeiro) | Alta | < 50ms |
| 3 | Buscar produto por SKU | Alta | < 10ms |
| 4 | Listar pedidos por status | Média | < 100ms |
| 5 | Buscar todos os itens de um pedido | Alta | < 20ms |

### Embedding vs Referencing

```
┌──────────────────────────────────────────────────────────────────┐
│           EMBEDDING vs REFERENCING                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  EMBEDDING (desnormalizado)         REFERENCING (normalizado)    │
│  ┌──────────────────────┐          ┌────────────────┐           │
│  │ order: {             │          │ order: {       │           │
│  │   id: "ord-1",       │          │   id: "ord-1", │           │
│  │   items: [           │          │   items: [     │           │
│  │     {                │          │     "item-1",  │           │
│  │       name: "Laptop",│          │     "item-2"   │           │
│  │       price: 4500,   │          │   ]            │           │
│  │       qty: 1         │          │ }              │           │
│  │     }                │          └────────────────┘           │
│  │   ],                 │          + lookup em items collection  │
│  │   user: {            │                                        │
│  │     name: "João",    │                                        │
│  │     email: "..."     │                                        │
│  │   }                  │                                        │
│  │ }                    │                                        │
│  └──────────────────────┘                                        │
│                                                                  │
│  Quando EMBEDDAR:                  Quando REFERENCIAR:          │
│  • 1:1 ou 1:few relação           • 1:many (milhares)           │
│  • Dados lidos juntos sempre       • Dados atualizados          │
│  • Dados imutáveis/append-only     independentemente            │
│  • Subdocumento < 16MB             • Documento cresceria demais │
│  • Não precisa do sub              • Dados compartilhados entre │
│    documento sozinho               múltiplos documents          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Patterns de Embedding:**

| Pattern | Quando usar | Exemplo |
|---------|------------|---------|
| **Embedded Document** | 1:1 ou 1:few, dados lidos juntos | Endereço dentro de Usuário |
| **Embedded Array** | 1:few, itens < ~100 | Tags em um Post, itens de um pedido pequeno |
| **Subset Pattern** | Documento com dados lidos parcialmente | Top 10 reviews embedados, demais em coleção |
| **Extended Reference** | Referência + campos frequentemente usados | `author: { id, name }` em vez de só `author_id` |
| **Computed Pattern** | Dados derivados pré-calculados | `total_amount`, `items_count` no pedido |
| **Bucket Pattern** | Agrupamento temporal ou por batch | Medições IoT agrupadas por hora/minuto |

---

## Document Stores (MongoDB, DocumentDB)

### Modelagem de Documents

```javascript
// ✅ BOM: Document com embedding inteligente
// Access patterns: buscar pedido completo, listar por usuário
{
  _id: ObjectId("..."),
  orderNumber: "ORD-2025-0001",
  status: "delivered",
  
  // Extended Reference — dados do user que são lidos no pedido
  customer: {
    id: ObjectId("user-123"),
    name: "João Silva",
    email: "joao@email.com"
    // NÃO embedar: endereço completo, histórico, preferências
  },
  
  // Embedded Array — itens são sempre lidos com o pedido
  items: [
    {
      sku: "LAPTOP-001",
      name: "Laptop Pro 15",
      price: NumberDecimal("4500.00"),
      quantity: 1,
      subtotal: NumberDecimal("4500.00")
    }
  ],
  
  // Computed Pattern — pré-calculado para evitar aggregation
  summary: {
    itemsCount: 1,
    totalAmount: NumberDecimal("4500.00"),
    currency: "BRL"
  },
  
  // Embedded Document — sempre lido junto, 1:1
  shipping: {
    address: "Rua X, 123",
    city: "São Paulo",
    trackingCode: "BR123456789",
    deliveredAt: ISODate("2025-01-20")
  },

  createdAt: ISODate("2025-01-15"),
  updatedAt: ISODate("2025-01-20")
}
```

```javascript
// ❌ RUIM: Embedding excessivo
{
  _id: ObjectId("..."),
  name: "João Silva",
  
  // ❌ Array que pode crescer infinitamente
  allOrders: [/* centenas/milhares de pedidos */],
  
  // ❌ Dados que mudam frequentemente em múltiplos documents
  currentAddress: {/* muda e precisa atualizar em todos os pedidos */}
}
```

### Indexação em MongoDB

```javascript
// MongoDB — Estratégia de índices

// 1. Compound index seguindo ESR (Equality, Sort, Range)
db.orders.createIndex(
  { status: 1, createdAt: -1 },  // Equality first, Sort second
  { name: "idx_status_created" }
);

// 2. Partial index — indexa apenas subset relevante
db.orders.createIndex(
  { customerId: 1, createdAt: -1 },
  { 
    name: "idx_active_orders",
    partialFilterExpression: { 
      status: { $in: ["pending", "processing", "shipped"] }
    }
  }
);

// 3. TTL index — delete automático após expiração
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }  // Remove após 1 hora
);

// 4. Text index para full-text search
db.products.createIndex(
  { name: "text", description: "text" },
  { weights: { name: 10, description: 5 } }
);

// 5. Wildcard index para JSONB-like queries
db.events.createIndex({ "metadata.$**": 1 });

// 6. Unique sparse index
db.users.createIndex(
  { email: 1 },
  { unique: true, sparse: true }  // NULL emails não conflitam
);
```

### Aggregation Pipeline

```javascript
// MongoDB Aggregation — Boas práticas

// ✅ Pipeline otimizado: $match primeiro para reduzir documents
db.orders.aggregate([
  // 1. Filtrar PRIMEIRO (usa índice)
  { $match: { 
    status: "delivered",
    createdAt: { $gte: ISODate("2025-01-01") }
  }},
  
  // 2. Projetar apenas campos necessários
  { $project: {
    customerId: "$customer.id",
    totalAmount: "$summary.totalAmount",
    month: { $month: "$createdAt" }
  }},
  
  // 3. Agrupar
  { $group: {
    _id: { customerId: "$customerId", month: "$month" },
    totalSpent: { $sum: "$totalAmount" },
    orderCount: { $sum: 1 }
  }},
  
  // 4. Ordenar
  { $sort: { totalSpent: -1 } },
  
  // 5. Limitar resultado
  { $limit: 100 }
]);
```

### Boas Práticas MongoDB

```
┌──────────────────────────────────────────────────────────────────┐
│            MONGODB — BOAS PRÁTICAS                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Modelagem:                                                      │
│  • Tamanho máximo de um document: 16 MB                         │
│  • Arrays não devem crescer indefinidamente (< 100 elementos)   │
│  • Use NumberDecimal para valores monetários (não Double)       │
│  • Use ISODate para timestamps (não strings)                    │
│  • Defina schema validation no nível da collection              │
│                                                                  │
│  Performance:                                                    │
│  • $match e $sort no início do pipeline (usa índices)           │
│  • $project/fields filter para reduzir tráfego de rede          │
│  • Read concern "majority" para consistência                    │
│  • Write concern "majority" para durabilidade                   │
│  • Use change streams em vez de polling                         │
│                                                                  │
│  Operações:                                                      │
│  • Replica set mínimo: 3 membros (1 primary + 2 secondary)     │
│  • Sharding: shard key baseada em access patterns               │
│  • Backup: mongodump/mongorestore ou snapshots de storage       │
│  • Monitoring: mongostat, mongotop, Atlas metrics               │
│                                                                  │
│  Segurança:                                                      │
│  • Autenticação: SCRAM-SHA-256 (mínimo)                        │
│  • Autorização: RBAC com roles granulares                       │
│  • Encryption at rest: WiredTiger encrypted storage             │
│  • Encryption in transit: TLS obrigatório                       │
│  • Network: Bind a interface interna, firewall                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key-Value Stores (Redis, DynamoDB)

### DynamoDB — Modelagem Single-Table

```
┌──────────────────────────────────────────────────────────────────┐
│          DYNAMODB — SINGLE TABLE DESIGN                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Conceito: Uma tabela para TODAS as entidades                   │
│  Vantagem: Queries resolvidas em uma operação                   │
│  Trade-off: Complexidade de modelagem e manutenção              │
│                                                                  │
│  PK (Partition Key)  │ SK (Sort Key)           │ Dados...       │
│  ────────────────────┼─────────────────────────┼────────────    │
│  USER#user-123       │ PROFILE                  │ name, email   │
│  USER#user-123       │ ORDER#2025-01-15#ord-1   │ total, status │
│  USER#user-123       │ ORDER#2025-01-20#ord-2   │ total, status │
│  ORDER#ord-1         │ ITEM#sku-001             │ qty, price    │
│  ORDER#ord-1         │ ITEM#sku-002             │ qty, price    │
│  PRODUCT#sku-001     │ DETAILS                  │ name, price   │
│                                                                  │
│  Access Patterns:                                                │
│  1. Get user profile: PK = USER#123, SK = PROFILE               │
│  2. List user orders: PK = USER#123, SK begins_with("ORDER#")  │
│  3. Get order items:  PK = ORDER#1, SK begins_with("ITEM#")    │
│  4. Get product:      PK = PRODUCT#sku, SK = DETAILS            │
│                                                                  │
│  GSI (Global Secondary Index):                                   │
│  • GSI1PK: status, GSI1SK: created_at                           │
│    → List orders by status (access pattern 4)                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### DynamoDB — Boas Práticas

```
┌──────────────────────────────────────────────────────────────────┐
│          DYNAMODB — BOAS PRÁTICAS                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Partition Key:                                                  │
│  • Alta cardinalidade (muitos valores distintos)                 │
│  • Distribuição uniforme de carga                                │
│  • ❌ Evitar: data, status, região como PK sozinhas             │
│  • ✅ Usar: user_id, order_id, device_id                        │
│  • Hot partition? → Adicionar sufixo de sharding                │
│    Ex: PK = "sensor#001#" + random(0, 9) → 10 partições        │
│                                                                  │
│  Capacity:                                                       │
│  • On-Demand: workloads imprevisíveis, desenvolvimento          │
│  • Provisioned + Auto Scaling: workloads previsíveis, produção  │
│  • Reserved Capacity: workloads estáveis (desconto de até 77%)  │
│                                                                  │
│  Item Size:                                                      │
│  • Máximo 400 KB por item                                        │
│  • Dados grandes → S3 com referência no DynamoDB                │
│  • Compressão de atributos grandes (gzip antes de salvar)       │
│                                                                  │
│  Queries:                                                        │
│  • Query > Scan (SEMPRE)                                         │
│  • FilterExpression NÃO reduz RCUs (filtra após leitura)       │
│  • ProjectionExpression para reduzir dados transferidos          │
│  • Paginação: use LastEvaluatedKey (não offset)                 │
│                                                                  │
│  Consistency:                                                    │
│  • Eventually consistent (default, 0.5 RCU por 4KB)            │
│  • Strongly consistent (1 RCU por 4KB, 50% mais caro)          │
│  • Transactional (2 RCUs por 4KB, 4x mais caro)                │
│                                                                  │
│  Streams:                                                        │
│  • DynamoDB Streams para CDC (Change Data Capture)              │
│  • Lambda trigger para reações a mudanças                       │
│  • Kinesis Data Streams para alto volume                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Redis — Patterns e Boas Práticas

```
┌──────────────────────────────────────────────────────────────────┐
│            REDIS — PATTERNS E BOAS PRÁTICAS                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Data Structures e seus usos:                                   │
│  ┌─────────────┬──────────────────────────────────────────────┐ │
│  │ Estrutura   │ Uso ideal                                    │ │
│  ├─────────────┼──────────────────────────────────────────────┤ │
│  │ String      │ Cache, sessões, contadores, feature flags   │ │
│  │ Hash        │ Objetos (user profile), configurações       │ │
│  │ List        │ Filas, últimos N itens, timeline            │ │
│  │ Set         │ Tags, memberships, deduplicação             │ │
│  │ Sorted Set  │ Rankings, leaderboards, rate limiting       │ │
│  │ Stream      │ Event log, messaging, CDC                   │ │
│  │ HyperLogLog │ Contagem aproximada de únicos (UV)          │ │
│  │ Bitmap      │ Flags por usuário, presença diária          │ │
│  │ Geo         │ Proximidade geográfica                      │ │
│  └─────────────┴──────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

```python
# Redis Patterns — Exemplos práticos

# 1. Cache-Aside Pattern
def get_product(product_id: str) -> dict:
    # Tentar cache primeiro
    cached = redis.get(f"product:{product_id}")
    if cached:
        return json.loads(cached)
    
    # Cache miss → buscar no banco
    product = db.query("SELECT * FROM products WHERE id = %s", product_id)
    
    # Popular cache com TTL
    redis.setex(
        f"product:{product_id}",
        3600,  # TTL: 1 hora
        json.dumps(product)
    )
    return product

# 2. Distributed Lock (Redlock simplificado)
def acquire_lock(lock_name: str, ttl_ms: int = 5000) -> str | None:
    token = str(uuid.uuid4())
    acquired = redis.set(
        f"lock:{lock_name}", 
        token,
        nx=True,          # Só seta se não existe
        px=ttl_ms         # TTL em milliseconds
    )
    return token if acquired else None

def release_lock(lock_name: str, token: str):
    # Lua script atômico para evitar deletar lock de outro processo
    script = """
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    end
    return 0
    """
    redis.eval(script, 1, f"lock:{lock_name}", token)

# 3. Rate Limiter (Sliding Window)
def is_rate_limited(user_id: str, max_requests: int = 100, window_secs: int = 60) -> bool:
    key = f"ratelimit:{user_id}"
    now = time.time()
    
    pipe = redis.pipeline()
    pipe.zremrangebyscore(key, 0, now - window_secs)  # Remove expirados
    pipe.zadd(key, {str(now): now})                    # Adiciona atual
    pipe.zcard(key)                                     # Conta total
    pipe.expire(key, window_secs)                       # TTL no key
    _, _, count, _ = pipe.execute()
    
    return count > max_requests

# 4. Leaderboard com Sorted Set
def update_score(player_id: str, score: int):
    redis.zadd("leaderboard:global", {player_id: score})

def get_top_players(n: int = 10) -> list:
    return redis.zrevrange("leaderboard:global", 0, n - 1, withscores=True)

def get_player_rank(player_id: str) -> int:
    return redis.zrevrank("leaderboard:global", player_id) + 1
```

**Redis — Boas Práticas Operacionais:**

| Prática | Recomendação |
|---------|-------------|
| **Eviction policy** | `allkeys-lru` para cache; `noeviction` para dados persistentes |
| **Maxmemory** | Definir sempre; ~75% da RAM disponível |
| **Key naming** | Usar `:` como separador: `entity:id:field` |
| **Key expiration** | SEMPRE definir TTL para dados de cache |
| **Big keys** | Evitar keys > 1MB; fragmentar se necessário |
| **Pipelining** | Batch múltiplos comandos (reduz round-trips) |
| **Lua scripts** | Operações atômicas complexas (vs MULTI/EXEC) |
| **Persistence** | RDB + AOF para durabilidade; RDB-only para cache |
| **Cluster** | Mínimo 6 nós (3 masters + 3 replicas) |

---

## Wide-Column Stores (Cassandra, ScyllaDB)

### Modelagem Cassandra

```
┌──────────────────────────────────────────────────────────────────┐
│          CASSANDRA — MODELAGEM                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Princípios:                                                     │
│  • Uma tabela por query (query-driven design)                   │
│  • Desnormalização é a norma                                    │
│  • Sem JOINs, sem subqueries                                    │
│  • Partition key define onde o dado fica no cluster              │
│  • Clustering key define a ordenação dentro da partição          │
│                                                                  │
│  Regras de Partition Key:                                       │
│  • Alta cardinalidade (muitos valores distintos)                 │
│  • Distribuição uniforme de dados                                │
│  • Partição ideal: 10-100 MB                                    │
│  • ❌ MAX: 2 billion cells / 100 MB por partição                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

```cql
-- Cassandra CQL — Exemplo de modelagem query-driven

-- Access Pattern 1: Listar posts de um usuário (mais recentes primeiro)
CREATE TABLE posts_by_user (
    user_id     UUID,
    created_at  TIMESTAMP,
    post_id     UUID,
    title       TEXT,
    content     TEXT,
    tags        SET<TEXT>,
    PRIMARY KEY ((user_id), created_at, post_id)
) WITH CLUSTERING ORDER BY (created_at DESC, post_id ASC);
-- PK: user_id (partition) + created_at, post_id (clustering)
-- Query: SELECT * FROM posts_by_user WHERE user_id = ?

-- Access Pattern 2: Buscar posts por tag
CREATE TABLE posts_by_tag (
    tag         TEXT,
    created_at  TIMESTAMP,
    post_id     UUID,
    user_id     UUID,
    title       TEXT,
    PRIMARY KEY ((tag), created_at, post_id)
) WITH CLUSTERING ORDER BY (created_at DESC, post_id ASC);
-- Dados DUPLICADOS entre as duas tabelas — isso é normal em Cassandra

-- Access Pattern 3: Métricas por sensor por hora (bucket pattern)
CREATE TABLE sensor_readings (
    sensor_id   TEXT,
    bucket      TEXT,           -- "2025-01-15-14" (hora)
    reading_at  TIMESTAMP,
    temperature DOUBLE,
    humidity    DOUBLE,
    PRIMARY KEY ((sensor_id, bucket), reading_at)
) WITH CLUSTERING ORDER BY (reading_at DESC)
  AND default_time_to_live = 2592000;  -- 30 dias TTL
-- Bucket limita tamanho da partição
```

### Boas Práticas Cassandra

| Prática | Recomendação |
|---------|-------------|
| **Write consistency** | `LOCAL_QUORUM` para produção (durável + performático) |
| **Read consistency** | `LOCAL_QUORUM` para dados importantes; `ONE` para analytics |
| **Partição size** | Monitorar; alertar se > 100MB |
| **Tombstones** | Evitar DELETE massivo; usar TTL quando possível |
| **Batch** | Somente `UNLOGGED BATCH` para operações na mesma partição |
| **LWT (IF NOT EXISTS)** | Usar com parcimônia (Paxos = lento) |
| **Compaction strategy** | `LeveledCompactionStrategy` para read-heavy; `TimeWindowCompactionStrategy` para time-series |
| **Replication factor** | RF=3 mínimo em produção |
| **Repair** | Rodar `nodetool repair` semanalmente |

---

## Graph Databases (Neo4j, Neptune)

### Modelagem de Grafos

```
┌──────────────────────────────────────────────────────────────────┐
│           GRAPH DB — MODELAGEM                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Componentes:                                                    │
│  • Nodes (vértices): entidades com labels e properties          │
│  • Relationships (arestas): relações direcionadas com           │
│    tipo e properties                                             │
│  • Properties: key-value pairs em nodes e relationships         │
│                                                                  │
│  Regras de modelagem:                                            │
│  • Substantivos → Nodes   (Person, Product, Company)            │
│  • Verbos → Relationships (BOUGHT, WORKS_AT, FOLLOWS)           │
│  • Adjetivos → Properties (name, since, amount)                 │
│  • Relationships devem ser ESPECÍFICAS (LIKES vs LOVES)         │
│  • Direção importa: (A)-[:FOLLOWS]->(B) ≠ (B)-[:FOLLOWS]->(A) │
│                                                                  │
│  Quando usar Graph DB:                                           │
│  • Queries com profundidade variável (amigos de amigos de...)   │
│  • Detecção de padrões em relações                              │
│  • Shortest path, comunidades, centralidade                     │
│  • Muitos JOINs em relacional (> 4 tabelas) é sinal de graph  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

```cypher
// Neo4j Cypher — Exemplos

// Criar nós e relações
CREATE (u:User {id: 'user-1', name: 'João', since: date('2024-01-01')})
CREATE (p:Product {id: 'prod-1', name: 'Laptop', price: 4500})
CREATE (u)-[:PURCHASED {date: date('2025-01-15'), amount: 4500}]->(p);

// Recomendação: "Quem comprou X também comprou..."
MATCH (u:User)-[:PURCHASED]->(p:Product {id: 'prod-1'})
MATCH (u)-[:PURCHASED]->(other:Product)
WHERE other.id <> 'prod-1'
RETURN other.name, COUNT(u) AS buyers
ORDER BY buyers DESC
LIMIT 5;

// Shortest path entre dois usuários
MATCH path = shortestPath(
  (a:User {id: 'user-1'})-[:FOLLOWS*..6]-(b:User {id: 'user-99'})
)
RETURN path, length(path);

// Detecção de fraude: ciclos suspeitos
MATCH (a:Account)-[:TRANSFER]->(b:Account)-[:TRANSFER]->(c:Account)-[:TRANSFER]->(a)
WHERE a <> b AND b <> c
RETURN a, b, c;

// Comunidades (Louvain algorithm via GDS — Graph Data Science)
CALL gds.louvain.stream('social-graph')
YIELD nodeId, communityId
RETURN gds.util.asNode(nodeId).name AS name, communityId
ORDER BY communityId;
```

### Boas Práticas Graph

| Prática | Recomendação |
|---------|-------------|
| **Indexes** | Criar em propriedades usadas em MATCH/WHERE |
| **Labels** | Múltiplos labels para categorização (`:User:Premium`) |
| **Relationship types** | Específicos e em UPPER_CASE (`PURCHASED`, não `related`) |
| **Supernodes** | Evitar nós com milhões de relações; usar intermediários |
| **UNWIND** | Para bulk imports em vez de múltiplos CREATE |
| **APOC** | Biblioteca de procedures úteis (load, export, algorithms) |
| **Profiling** | `PROFILE` antes de `EXPLAIN` para ver plano real |
| **Transactions** | Neo4j suporta ACID — usar para consistência |

---

## Consistency Tuning

```
┌──────────────────────────────────────────────────────────────────┐
│          CONSISTENCY TUNING POR BANCO                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Cassandra (Tunable per-query):                                 │
│  ┌───────────────┬───────────────────────────────────────────┐  │
│  │ Level         │ Comportamento                             │  │
│  ├───────────────┼───────────────────────────────────────────┤  │
│  │ ONE           │ 1 réplica responde (mais rápido, AP)     │  │
│  │ QUORUM        │ Maioria responde (equilíbrio)            │  │
│  │ LOCAL_QUORUM  │ Maioria no datacenter local (prod)       │  │
│  │ ALL           │ Todas réplicas (mais lento, CP)          │  │
│  │ EACH_QUORUM   │ Quorum em cada datacenter                │  │
│  └───────────────┴───────────────────────────────────────────┘  │
│  Regra: W + R > RF → strong consistency                        │
│  Ex: RF=3, W=QUORUM(2), R=QUORUM(2) → 2+2 > 3 ✅             │
│                                                                  │
│  DynamoDB:                                                       │
│  • Eventually consistent read (default, 0.5 RCU/4KB)           │
│  • Strongly consistent read (1 RCU/4KB)                        │
│  • Transactional read/write (2x custo)                          │
│                                                                  │
│  MongoDB:                                                        │
│  • Read Concern: local, available, majority, linearizable       │
│  • Write Concern: 0, 1, majority, {w: N}                       │
│  • Read Preference: primary, primaryPreferred, secondary,       │
│    secondaryPreferred, nearest                                  │
│                                                                  │
│  Cosmos DB (5 níveis):                                          │
│  Strong → Bounded Staleness → Session → Consistent Prefix      │
│  → Eventual                                                     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Schema Evolution em NoSQL

```
┌──────────────────────────────────────────────────────────────────┐
│          SCHEMA EVOLUTION — ESTRATÉGIAS                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Schema Version Field                                        │
│     • Adicionar "schemaVersion" em cada document                │
│     • Aplicação trata múltiplas versões                         │
│     • Migração lazy (sob demanda quando lido)                   │
│     {                                                            │
│       "schemaVersion": 2,                                       │
│       "name": "João",                                           │
│       "address": { "street": "...", "city": "..." }            │
│       // v1 tinha "address" como string                        │
│     }                                                            │
│                                                                  │
│  2. Additive Changes Only                                       │
│     • Sempre adicionar campos novos (nunca remover/renomear)    │
│     • Campos antigos → deprecate com documentação               │
│     • Tratamento de null/missing na aplicação                   │
│                                                                  │
│  3. Dual-Write Migration                                         │
│     • Fase 1: Escrever em formato novo + antigo                 │
│     • Fase 2: Migrar dados históricos em background             │
│     • Fase 3: Ler do formato novo                               │
│     • Fase 4: Parar de escrever formato antigo                  │
│                                                                  │
│  4. Schema Validation (MongoDB)                                  │
│     • db.createCollection com $jsonSchema                       │
│     • validationLevel: "moderate" (valida updates, não existing)│
│     • Evita que dados inválidos entrem no futuro                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Anti-patterns NoSQL

| # | Anti-pattern | Problema | Solução |
|---|-------------|----------|---------|
| 1 | **Relational modeling in NoSQL** | JOINs, normalização excessiva, foreign keys | Modelar por access patterns, desnormalizar |
| 2 | **Unbounded arrays** | Array em document cresce sem limite → 16MB | Limitar com bucket pattern ou coleção separada |
| 3 | **Hot partition** | Uma partition key concentra todo o tráfego | Write sharding, partition key com alta cardinalidade |
| 4 | **Scan instead of Query** | Full table scan em DynamoDB/Cassandra | Criar GSI/tabela materializada para o access pattern |
| 5 | **No TTL on ephemeral data** | Sessões, cache, tokens crescem indefinidamente | Definir TTL nativo do banco |
| 6 | **Over-indexing** | Índices demais = writes mais lentas + storage | Indexar somente access patterns reais |
| 7 | **Large items/documents** | Documents de 10+ MB causam latência | Comprimir, fragmentar, ou mover para S3 |
| 8 | **Ignoring tombstones** | Cassandra com milhões de tombstones = reads lentas | TTL em vez de DELETE; ajustar gc_grace_seconds |
| 9 | **Schemaless = no validation** | Dados corrompidos entram sem controle | Schema validation, tipos fortes na aplicação |
| 10 | **One collection per entity** | MongoDB com 50+ collections = muitos $lookups | Single collection patterns, embedding inteligente |
| 11 | **Distributed transactions** | Multi-partition transactions = lento | Saga pattern, eventual consistency, idempotência |
| 12 | **Using NoSQL for reports** | Aggregations complexas ad-hoc | Export para OLAP (Redshift, BigQuery) via CDC |

---

## Referências

- MongoDB. *Data Modeling Patterns* — https://www.mongodb.com/docs/manual/data-modeling/
- DynamoDB. *Best Practices* — https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html
- Rick Houlihan. *DynamoDB Advanced Design Patterns* — re:Invent talks
- DataStax. *Cassandra Data Modeling* — https://docs.datastax.com/en/cassandra-oss/
- Neo4j. *Graph Database Concepts* — https://neo4j.com/docs/getting-started/
- Redis. *Data Types & Commands* — https://redis.io/docs/data-types/
- Fowler, M. (2012). *NoSQL Distilled* — Addison-Wesley
- Kleppmann, M. (2017). *Designing Data-Intensive Applications* — O'Reilly
