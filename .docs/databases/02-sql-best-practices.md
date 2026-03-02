# Databases — SQL & Bancos Relacionais — Boas Práticas

> **Objetivo deste documento:** Servir como referência completa sobre **boas práticas para bancos de dados relacionais (SQL)**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Cobre modelagem, indexação, query optimization, normalização, particionamento e operações para bancos SQL.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **Normalização** | 3NF para OLTP; desnormalizar somente para leitura | Dados em 1NF ou 5NF+ sem justificativa | Aplicar 3NF; criar views materializadas para reports |
| **Índices** | Indexar colunas de WHERE, JOIN e ORDER BY | Índice em toda coluna ou nenhum índice | Analisar query plans; criar índices compostos alinhados |
| **N+1 Queries** | Usar JOINs ou batch loading | Loop que faz 1 query por iteração | Eager loading, JOINs, ou IN clause batching |
| **SELECT \*** | Selecionar apenas colunas necessárias | `SELECT * FROM orders` em queries de produção | Listar colunas explicitamente |
| **Transactions** | Manter transações curtas e focadas | Transação longa com lógica de negócio dentro | Separar leitura da escrita; commit rápido |
| **Connection Pool** | Reutilizar conexões via pool | Abrir/fechar conexão por request | Usar pool (HikariCP, PgBouncer) |

---

## Sumário

- [Databases — SQL \& Bancos Relacionais — Boas Práticas](#databases--sql--bancos-relacionais--boas-práticas)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Normalização](#normalização)
    - [Formas Normais](#formas-normais)
    - [Quando desnormalizar](#quando-desnormalizar)
  - [Modelagem Relacional](#modelagem-relacional)
    - [Convenções de nomenclatura](#convenções-de-nomenclatura)
    - [Tipos de dados — Boas Práticas](#tipos-de-dados--boas-práticas)
    - [Constraints essenciais](#constraints-essenciais)
  - [Indexação](#indexação)
    - [Tipos de índices](#tipos-de-índices)
    - [Estratégia de indexação](#estratégia-de-indexação)
    - [Anti-patterns de indexação](#anti-patterns-de-indexação)
  - [Query Optimization](#query-optimization)
    - [EXPLAIN e Query Plans](#explain-e-query-plans)
    - [Padrões de queries eficientes](#padrões-de-queries-eficientes)
    - [Anti-patterns de queries](#anti-patterns-de-queries)
  - [Transactions e Concorrência](#transactions-e-concorrência)
    - [Boas práticas de transações](#boas-práticas-de-transações)
    - [Locking strategies](#locking-strategies)
  - [Connection Management](#connection-management)
  - [Particionamento](#particionamento)
  - [Replicação](#replicação)
  - [Migrations e Schema Evolution](#migrations-e-schema-evolution)
  - [Modelagem OLTP vs OLAP](#modelagem-oltp-vs-olap)
  - [Monitoramento e Operações](#monitoramento-e-operações)
  - [Anti-patterns SQL](#anti-patterns-sql)
  - [Referências](#referências)

---

## Normalização

### Formas Normais

```
┌──────────────────────────────────────────────────────────────────┐
│                   FORMAS NORMAIS                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1NF — Primeira Forma Normal                                    │
│  │  • Valores atômicos (sem listas, arrays, JSON em colunas)    │
│  │  • Cada linha é única (tem primary key)                      │
│  │  • Sem grupos repetitivos                                     │
│  │                                                               │
│  │  ❌ ERRADO: telefones = "11999,11888,11777"                  │
│  │  ✅ CERTO:  tabela separada user_phones (user_id, phone)     │
│  │                                                               │
│  2NF — Segunda Forma Normal                                     │
│  │  • Está em 1NF                                                │
│  │  • Todo atributo não-chave depende da chave inteira           │
│  │  • Sem dependências parciais (em chaves compostas)            │
│  │                                                               │
│  │  ❌ ERRADO: (pedido_id, produto_id) → nome_produto           │
│  │     nome_produto depende só de produto_id, não da chave toda │
│  │  ✅ CERTO: mover nome_produto para tabela produtos           │
│  │                                                               │
│  3NF — Terceira Forma Normal                                    │
│  │  • Está em 2NF                                                │
│  │  • Sem dependências transitivas                               │
│  │  • Atributos não-chave dependem SOMENTE da chave             │
│  │                                                               │
│  │  ❌ ERRADO: pedidos(pedido_id, cliente_id, cidade_cliente)   │
│  │     cidade_cliente depende de cliente_id, não de pedido_id   │
│  │  ✅ CERTO: cidade fica na tabela clientes                    │
│  │                                                               │
│  BCNF — Forma Normal de Boyce-Codd                              │
│  │  • Todo determinante é uma chave candidata                    │
│  │  • Elimina anomalias residuais da 3NF                        │
│  │                                                               │
│  4NF+ — Raramente necessário em sistemas comuns                 │
│                                                                  │
│  🎯 REGRA PRÁTICA: 3NF é suficiente para 95% dos sistemas OLTP │
└──────────────────────────────────────────────────────────────────┘
```

### Quando desnormalizar

| Cenário | Justificativa | Técnica |
|---------|--------------|---------|
| **Leitura pesada com JOINs caros** | Performance de leitura > custo de redundância | Views materializadas, colunas calculadas |
| **Reporting/Analytics** | Modelo estrela/snowflake para OLAP | Data warehouse com fatos e dimensões |
| **Caching de dados derivados** | Evitar recalcular aggregations em toda leitura | Colunas denormalizadas com triggers de atualização |
| **Alta escala de leitura** | Reduzir latência eliminando JOINs quentes | Tabelas de leitura separadas (CQRS) |

> **Regra:** Desnormalize **conscientemente** e **documente** o motivo. Cada desnormalização cria uma obrigação de manter a consistência manualmente.

---

## Modelagem Relacional

### Convenções de nomenclatura

```sql
-- ✅ Boas práticas de nomenclatura
CREATE TABLE orders (                    -- substantivo no plural, snake_case
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id      BIGINT NOT NULL,        -- FK com sufixo _id
    status       VARCHAR(20) NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,  -- nomes descritivos
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),  -- timestamps com _at
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    CONSTRAINT fk_orders_user 
        FOREIGN KEY (user_id) REFERENCES users(id),
    CONSTRAINT chk_orders_total_positive 
        CHECK (total_amount >= 0)
);

-- Índices: idx_{tabela}_{colunas}
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status_created ON orders(status, created_at);
```

| Regra | Exemplo bom | Exemplo ruim |
|-------|------------|-------------|
| Tabelas no plural, snake_case | `order_items` | `OrderItem`, `tbl_order_item` |
| PKs como `id` (BIGINT) | `id BIGINT` | `order_id INT` como PK |
| FKs com `_id` suffix | `user_id` | `user`, `fk_user` |
| Timestamps com `_at` | `created_at`, `deleted_at` | `creation_date`, `dt_criacao` |
| Booleans com `is_`/`has_` | `is_active`, `has_discount` | `active`, `discount` |
| Sem prefixos desnecessários | `users` | `tbl_users`, `tb_users` |
| Constraints nomeadas | `chk_orders_total_positive` | constraints anônimas |

### Tipos de dados — Boas Práticas

| Dado | Tipo recomendado | Evitar | Motivo |
|------|-----------------|--------|--------|
| **ID / PK** | `BIGINT GENERATED ALWAYS AS IDENTITY` ou `UUID` | `INT`, `SERIAL` | BIGINT evita overflow; UUID para distribuídos |
| **Dinheiro** | `DECIMAL(12,2)` ou `NUMERIC` | `FLOAT`, `DOUBLE` | Ponto flutuante tem erros de arredondamento |
| **Timestamp** | `TIMESTAMPTZ` | `TIMESTAMP` sem timezone | Sem timezone causa bugs em sistemas multi-região |
| **Status/Enum** | `VARCHAR(20)` com CHECK | `INT` mágico ou `ENUM` nativo | VARCHAR + CHECK é mais portável e legível |
| **Texto curto** | `VARCHAR(n)` com limite | `TEXT` sem validação | Limite previne abuso; TEXT para conteúdo livre |
| **JSON** | `JSONB` (PostgreSQL) | `JSON` ou `TEXT` com JSON | JSONB é indexável e mais eficiente |
| **IP Address** | `INET` (PostgreSQL) | `VARCHAR(45)` | Tipo nativo permite operações de rede |
| **Boolean** | `BOOLEAN` | `INT(1)`, `CHAR(1)` | Tipo nativo é mais claro e eficiente |

### Constraints essenciais

```sql
-- Sempre defina constraints explicitamente
CREATE TABLE products (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku         VARCHAR(50) NOT NULL UNIQUE,          -- unicidade de negócio
    name        VARCHAR(200) NOT NULL,
    price       DECIMAL(10,2) NOT NULL,
    stock       INT NOT NULL DEFAULT 0,
    category_id BIGINT NOT NULL,
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- Foreign keys explícitas
    CONSTRAINT fk_products_category 
        FOREIGN KEY (category_id) REFERENCES categories(id)
        ON DELETE RESTRICT                             -- impedir exclusão com dependentes
        ON UPDATE CASCADE,                             -- propagar atualização de PK
    
    -- Check constraints
    CONSTRAINT chk_products_price_positive 
        CHECK (price > 0),
    CONSTRAINT chk_products_stock_non_negative 
        CHECK (stock >= 0),
    CONSTRAINT chk_products_sku_format 
        CHECK (sku ~ '^[A-Z0-9\-]+$')                 -- regex para formato
);
```

---

## Indexação

### Tipos de índices

```
┌──────────────────────────────────────────────────────────────────┐
│                    TIPOS DE ÍNDICES                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  B-Tree (padrão)                                                │
│  │  • =, <, >, <=, >=, BETWEEN, IN, LIKE 'abc%'                │
│  │  • Ideal para: maioria dos casos, range queries               │
│  │  • CREATE INDEX idx_name ON table(column);                    │
│  │                                                               │
│  Hash                                                            │
│  │  • Apenas igualdade (=)                                       │
│  │  • Mais rápido que B-Tree para lookups exatos                │
│  │  • CREATE INDEX idx_name ON table USING hash(column);         │
│  │                                                               │
│  GIN (Generalized Inverted Index)                               │
│  │  • Full-text search, JSONB, arrays, tsvector                 │
│  │  • CREATE INDEX idx_name ON table USING gin(column);          │
│  │                                                               │
│  GiST (Generalized Search Tree)                                 │
│  │  • Dados geoespaciais, ranges, nearest neighbor               │
│  │  • CREATE INDEX idx_name ON table USING gist(column);         │
│  │                                                               │
│  BRIN (Block Range Index)                                        │
│  │  • Dados naturalmente ordenados (timestamps, IDs sequenciais)│
│  │  • Muito compacto, ideal para tabelas muito grandes           │
│  │  • CREATE INDEX idx_name ON table USING brin(column);         │
│  │                                                               │
│  Partial Index                                                   │
│  │  • Indexa subset de linhas                                    │
│  │  • CREATE INDEX idx_name ON table(col) WHERE condition;       │
│  │                                                               │
│  Composite Index                                                 │
│  │  • Múltiplas colunas — ordem importa!                        │
│  │  • Segue leftmost prefix rule                                 │
│  │  • CREATE INDEX idx_name ON table(col1, col2, col3);          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Estratégia de indexação

```sql
-- 1. Composite index — a ordem das colunas importa!
-- Queries: WHERE status = ? AND created_at > ?
-- O índice é útil para: (status), (status, created_at)
-- NÃO é útil para: (created_at) sozinho
CREATE INDEX idx_orders_status_created 
    ON orders(status, created_at);

-- 2. Covering index (INCLUDE) — evita "table lookup"
-- A query inteira é resolvida pelo índice
CREATE INDEX idx_orders_covering 
    ON orders(user_id, status) 
    INCLUDE (total_amount, created_at);

-- 3. Partial index — indexa apenas dados relevantes
-- Se 95% dos pedidos são 'completed', indexar só os ativos
CREATE INDEX idx_orders_active 
    ON orders(user_id, created_at) 
    WHERE status IN ('pending', 'processing');

-- 4. Expression index — indexa resultado de função
CREATE INDEX idx_users_email_lower 
    ON users(LOWER(email));

-- 5. JSONB index — busca em campos JSON
CREATE INDEX idx_users_preferences 
    ON users USING gin(preferences jsonb_path_ops);

-- 6. Unique index para business rules
CREATE UNIQUE INDEX idx_users_email_unique 
    ON users(LOWER(email)) 
    WHERE deleted_at IS NULL;  -- unique apenas para não-deletados
```

**Regra do composite index — Equality, Sort, Range (ESR):**

```
┌──────────────────────────────────────────────────────────────────┐
│           COMPOSITE INDEX — REGRA ESR                            │
│                                                                  │
│  Ordem ideal das colunas no índice composto:                    │
│                                                                  │
│  1. Equality  (=)     → Colunas com filtro de igualdade PRIMEIRO│
│  2. Sort      (ORDER BY) → Colunas de ordenação em SEGUNDO      │
│  3. Range     (>, <, BETWEEN) → Colunas de range por ÚLTIMO    │
│                                                                  │
│  Exemplo:                                                        │
│  SELECT * FROM orders                                            │
│  WHERE status = 'pending'          -- Equality                  │
│    AND region = 'us-east-1'        -- Equality                  │
│  ORDER BY created_at DESC          -- Sort                       │
│  WHERE total > 100;                -- Range                      │
│                                                                  │
│  ✅ Índice ideal: (status, region, created_at, total)            │
│  ❌ Índice ruim:  (created_at, status, total, region)            │
└──────────────────────────────────────────────────────────────────┘
```

### Anti-patterns de indexação

| Anti-pattern | Problema | Solução |
|-------------|----------|---------|
| **Index on every column** | Escritas lentas, storage alto | Indexar baseado em queries reais |
| **Missing composite index** | Múltiplos index scans em vez de um | Criar composite seguindo ESR |
| **Wrong column order** | Índice não é usado pela query | Analisar EXPLAIN, reordenar colunas |
| **Indexing low-cardinality** | B-tree em boolean/enum com 2-3 valores | Partial index ou sem índice |
| **Unused indexes** | Storage e write overhead sem benefício | `pg_stat_user_indexes` para identificar |
| **Over-indexing** | Degradação de INSERT/UPDATE/DELETE | Máximo 5-8 índices por tabela como guia |
| **Redundant indexes** | `(a)` e `(a, b)` — o primeiro é redundante | Remover `(a)`, o composto já cobre |

---

## Query Optimization

### EXPLAIN e Query Plans

```sql
-- Sempre use EXPLAIN ANALYZE para queries em produção
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) 
SELECT o.id, o.total_amount, u.name
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '7 days'
ORDER BY o.created_at DESC
LIMIT 50;
```

**O que observar no EXPLAIN:**

| Indicador | Bom | Ruim |
|-----------|-----|------|
| **Scan type** | Index Scan, Index Only Scan | Seq Scan em tabela grande |
| **Rows estimated vs actual** | Próximos | Ordem de grandeza diferente |
| **Buffers** | Shared hit (cache) | Shared read (disco) |
| **Sort method** | Top-N heapsort, quicksort (em memória) | External merge (disco) |
| **Join type** | Nested Loop (poucos), Hash Join (médio) | Nested Loop em tabelas grandes |

### Padrões de queries eficientes

```sql
-- ✅ Paginação eficiente com cursor (keyset pagination)
-- Muito mais rápido que OFFSET para páginas profundas
SELECT id, name, created_at
FROM products
WHERE created_at < '2025-01-15T10:30:00Z'  -- cursor da página anterior
ORDER BY created_at DESC
LIMIT 20;

-- ❌ EVITAR: OFFSET para paginação profunda
SELECT * FROM products ORDER BY created_at DESC LIMIT 20 OFFSET 100000;
-- ^ Precisa ler 100.020 linhas para retornar 20

-- ✅ Batch update com CTE
WITH batch AS (
    SELECT id FROM large_table
    WHERE status = 'pending'
    ORDER BY id
    LIMIT 1000
    FOR UPDATE SKIP LOCKED  -- pula linhas lockadas por outros processos
)
UPDATE large_table 
SET status = 'processing', updated_at = NOW()
WHERE id IN (SELECT id FROM batch);

-- ✅ Upsert (INSERT ON CONFLICT)
INSERT INTO products (sku, name, price, stock)
VALUES ('SKU-001', 'Widget', 29.99, 100)
ON CONFLICT (sku) 
DO UPDATE SET 
    price = EXCLUDED.price,
    stock = EXCLUDED.stock,
    updated_at = NOW()
WHERE products.price != EXCLUDED.price 
   OR products.stock != EXCLUDED.stock;  -- só atualiza se mudou

-- ✅ Aggregation com filtro (evita subqueries)
SELECT 
    status,
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE total_amount > 1000) AS high_value,
    AVG(total_amount) AS avg_amount
FROM orders
WHERE created_at > NOW() - INTERVAL '30 days'
GROUP BY status;

-- ✅ Lateral join para "top-N por grupo"
SELECT u.id, u.name, recent_orders.*
FROM users u
CROSS JOIN LATERAL (
    SELECT o.id AS order_id, o.total_amount, o.created_at
    FROM orders o
    WHERE o.user_id = u.id
    ORDER BY o.created_at DESC
    LIMIT 3
) recent_orders
WHERE u.is_active = true;
```

### Anti-patterns de queries

| Anti-pattern | Exemplo ruim | Correção |
|-------------|-------------|----------|
| **SELECT \*** | `SELECT * FROM orders` | Listar colunas necessárias |
| **N+1 queries** | Loop com query por iteração | JOIN ou batch loading |
| **Function on indexed column** | `WHERE YEAR(created_at) = 2025` | `WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01'` |
| **Leading wildcard LIKE** | `WHERE name LIKE '%laptop%'` | Full-text search (GIN + tsvector) |
| **Implicit type cast** | `WHERE id = '123'` (string vs int) | Usar tipo correto: `WHERE id = 123` |
| **OR in WHERE** | `WHERE a = 1 OR b = 2` | `UNION ALL` de duas queries indexadas |
| **NOT IN com NULL** | `WHERE id NOT IN (SELECT...)` | `NOT EXISTS` (seguro com NULLs) |
| **Correlated subquery** | Subquery executada por linha | JOIN ou CTE |
| **DISTINCT como band-aid** | `SELECT DISTINCT` para esconder duplicatas | Corrigir JOINs que causam duplicatas |
| **OFFSET paginação** | `LIMIT 20 OFFSET 100000` | Keyset pagination com cursor |

---

## Transactions e Concorrência

### Boas práticas de transações

```sql
-- ✅ Transação curta e focada
BEGIN;
    UPDATE accounts SET balance = balance - 100.00 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100.00 WHERE id = 2;
    INSERT INTO transfers (from_id, to_id, amount, created_at)
    VALUES (1, 2, 100.00, NOW());
COMMIT;

-- ✅ Retry on serialization failure (na aplicação)
-- PostgreSQL pode retornar ERROR 40001 com Serializable
-- A aplicação deve implementar retry com backoff
```

```
┌──────────────────────────────────────────────────────────────────┐
│              REGRAS PARA TRANSAÇÕES                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Mantenha transações CURTAS                                  │
│     • Sem chamadas HTTP dentro de transações                    │
│     • Sem processamento pesado dentro de transações             │
│     • Sem user input dentro de transações                       │
│                                                                  │
│  2. Use o nível de isolamento ADEQUADO                          │
│     • Read Committed = padrão para 90% dos casos               │
│     • Repeatable Read = quando leitura consistente é crítica    │
│     • Serializable = quando integridade absoluta é necessária   │
│                                                                  │
│  3. Ordene locks CONSISTENTEMENTE                               │
│     • Sempre adquira locks na mesma ordem (ex: por ID crescente)│
│     • Evita deadlocks                                            │
│                                                                  │
│  4. Use SAVEPOINT para rollback parcial                         │
│     • BEGIN; SAVEPOINT sp1; ...; ROLLBACK TO sp1; COMMIT;       │
│                                                                  │
│  5. Handle deadlocks na aplicação                               │
│     • Detect + Retry com exponential backoff                    │
│     • SQLSTATE 40P01 (PostgreSQL)                                │
│                                                                  │
│  6. Use advisory locks para coordenação                         │
│     • pg_advisory_lock(key) para processos exclusivos           │
│     • Cron jobs, migrations, batch processing                   │
└──────────────────────────────────────────────────────────────────┘
```

### Locking strategies

| Estratégia | Uso | Exemplo |
|-----------|-----|---------|
| **Optimistic Locking** | Conflitos raros; alta concorrência | Coluna `version` — check no UPDATE |
| **Pessimistic Locking** | Conflitos frequentes; dados críticos | `SELECT ... FOR UPDATE` |
| **SKIP LOCKED** | Worker queue pattern | `SELECT ... FOR UPDATE SKIP LOCKED` |
| **Advisory Lock** | Processos exclusivos (cron, migration) | `pg_advisory_lock(hash)` |

```sql
-- Optimistic Locking — na aplicação
UPDATE products 
SET stock = stock - 1, version = version + 1
WHERE id = 42 AND version = 5;  -- versão lida anteriormente
-- Se affected_rows = 0, outro processo alterou → retry

-- Pessimistic Locking — SELECT FOR UPDATE
BEGIN;
SELECT stock FROM products WHERE id = 42 FOR UPDATE;  -- lock na linha
-- processar...
UPDATE products SET stock = stock - 1 WHERE id = 42;
COMMIT;

-- Worker queue com SKIP LOCKED
BEGIN;
SELECT id, payload FROM job_queue
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;  -- pula jobs já em processamento

UPDATE job_queue SET status = 'processing' WHERE id = ?;
COMMIT;
```

---

## Connection Management

```
┌──────────────────────────────────────────────────────────────────┐
│              CONNECTION POOLING                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Application ──► Connection Pool ──► Database                   │
│                                                                  │
│  Por que usar pool:                                              │
│  • Criar conexão TCP + SSL + auth ≈ 50-200ms                   │
│  • Pool reutiliza conexões ≈ < 1ms                              │
│  • Banco tem limite de conexões (max_connections)               │
│                                                                  │
│  Sizing do pool:                                                │
│  • Fórmula: connections = (cores * 2) + disk_spindles           │
│  • Para SSD: connections ≈ cores * 2                            │
│  • Exemplo: 8 cores → pool size = 16-20                        │
│  • Mais não é melhor! Pool grande = contenção interna           │
│                                                                  │
│  Ferramentas:                                                    │
│  • Java: HikariCP (recomendado)                                 │
│  • PostgreSQL: PgBouncer (proxy externo)                        │
│  • MySQL: ProxySQL                                               │
│  • Cloud: RDS Proxy (AWS)                                        │
│                                                                  │
│  Configuração HikariCP recomendada:                              │
│  • maximumPoolSize: cores * 2 + 2                               │
│  • minimumIdle: mesmo que maximumPoolSize (warm pool)           │
│  • connectionTimeout: 30s                                        │
│  • idleTimeout: 600s (10min)                                     │
│  • maxLifetime: 1800s (30min) — antes do DB timeout             │
│  • leakDetectionThreshold: 60s                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Particionamento

```
┌──────────────────────────────────────────────────────────────────┐
│              PARTICIONAMENTO DE TABELAS                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Range Partitioning — por período/faixa                         │
│  │  Ideal para: Dados temporais (logs, eventos, métricas)       │
│  │  Partition pruning: query com WHERE date = ? só lê 1 partição│
│  │                                                               │
│  List Partitioning — por valor discreto                         │
│  │  Ideal para: Status, região, categoria                       │
│  │  Ex: Uma partição por região (US, EU, APAC)                  │
│  │                                                               │
│  Hash Partitioning — por hash da chave                          │
│  │  Ideal para: Distribuição uniforme, sem padrão natural       │
│  │  Ex: Particionar por hash(user_id) em 8 partições            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

```sql
-- Range partitioning por mês (PostgreSQL)
CREATE TABLE events (
    id          BIGINT GENERATED ALWAYS AS IDENTITY,
    event_type  VARCHAR(50) NOT NULL,
    payload     JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Criar partições mensais
CREATE TABLE events_2025_01 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE events_2025_02 PARTITION OF events
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Índices são criados por partição automaticamente
CREATE INDEX idx_events_type ON events(event_type, created_at);

-- Queries = automático! PostgreSQL faz partition pruning
SELECT * FROM events 
WHERE created_at >= '2025-01-15' AND created_at < '2025-02-01';
-- → Só escaneia events_2025_01
```

**Quando particionar:**

| Critério | Recomendação |
|----------|-------------|
| Tabela > 100 milhões de linhas | Considerar particionamento |
| Queries sempre filtram por um range (data, região) | Range ou List partitioning |
| Precisa deletar dados antigos periodicamente | `DROP PARTITION` em vez de `DELETE` (instantâneo) |
| Tabela < 10 milhões de linhas | Geralmente não precisa |

---

## Replicação

```
┌──────────────────────────────────────────────────────────────────┐
│               MODELOS DE REPLICAÇÃO                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Primary-Replica (mais comum)                                   │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐                   │
│  │ Primary │────►│Replica 1│     │Replica 2│                   │
│  │  (R/W)  │────►│  (R/O)  │     │  (R/O)  │                   │
│  └─────────┘     └─────────┘     └─────────┘                   │
│  • Writes → Primary only                                        │
│  • Reads → Replicas (read scaling)                              │
│  • Failover → Promote replica to primary                        │
│                                                                  │
│  Síncrona vs Assíncrona:                                        │
│  • Síncrona: Commit espera réplica confirmar → durável, lento   │
│  • Assíncrona: Commit imediato → rápido, risco de data loss     │
│  • Semi-síncrona: Pelo menos 1 réplica confirma                 │
│                                                                  │
│  Multi-Primary (Active-Active)                                  │
│  ┌─────────┐◄───►┌─────────┐                                   │
│  │Primary 1│     │Primary 2│                                   │
│  │  (R/W)  │     │  (R/W)  │                                   │
│  └─────────┘     └─────────┘                                   │
│  • Ambos aceitam writes                                          │
│  • Conflitos precisam ser resolvidos                             │
│  • Mais complexo, use com cautela                               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Replication lag e suas implicações:**

| Problema | Cenário | Solução |
|----------|---------|---------|
| **Read-after-write** | Usuário salva e não vê o dado | Read from primary após write |
| **Monotonic reads** | Refresh mostra dado antigo | Sticky session para mesma réplica |
| **Stale reads** | Réplica atrasada serve dado velho | Tolerar ou usar strong consistency |

---

## Migrations e Schema Evolution

```
┌──────────────────────────────────────────────────────────────────┐
│              BOAS PRÁTICAS DE MIGRATIONS                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Migrations são IMUTÁVEIS após aplicadas                     │
│     Nunca editar uma migration já em produção                   │
│                                                                  │
│  2. Cada migration deve ser REVERSÍVEL                          │
│     Sempre implementar UP e DOWN                                │
│                                                                  │
│  3. ZERO-DOWNTIME migrations                                    │
│     • ADD COLUMN → sem lock (PostgreSQL)                        │
│     • ADD COLUMN NOT NULL DEFAULT → com lock breve (PG 11+)    │
│     • DROP COLUMN → marcar como unused primeiro                 │
│     • RENAME COLUMN → criar novo, dual-write, migrar, remover  │
│     • ADD INDEX → usar CONCURRENTLY                             │
│                                                                  │
│  4. Expand-and-Contract pattern                                  │
│     Deploy 1: Adicionar nova coluna (expand)                    │
│     Deploy 2: Escrever em ambas as colunas                      │
│     Deploy 3: Migrar dados antigos                              │
│     Deploy 4: Ler da nova coluna                                │
│     Deploy 5: Remover coluna antiga (contract)                  │
│                                                                  │
│  Ferramentas: Flyway, Liquibase, Alembic, golang-migrate,      │
│               Prisma Migrate, TypeORM migrations                 │
└──────────────────────────────────────────────────────────────────┘
```

```sql
-- ✅ Adicionar índice sem lock
CREATE INDEX CONCURRENTLY idx_orders_user 
    ON orders(user_id);

-- ✅ Adicionar coluna NOT NULL (PostgreSQL 11+)
ALTER TABLE orders ADD COLUMN priority INT NOT NULL DEFAULT 0;

-- ❌ PERIGOSO: Renomear coluna diretamente
ALTER TABLE orders RENAME COLUMN status TO order_status;
-- ^ Quebra queries que usam "status"

-- ✅ SEGURO: Expand-and-Contract para renomear
-- Step 1: Adicionar nova coluna
ALTER TABLE orders ADD COLUMN order_status VARCHAR(20);
-- Step 2: Trigger de dual-write
-- Step 3: Backfill dados
UPDATE orders SET order_status = status WHERE order_status IS NULL;
-- Step 4: Aplicação lê de order_status
-- Step 5: Remover coluna antiga
ALTER TABLE orders DROP COLUMN status;
```

---

## Modelagem OLTP vs OLAP

```
┌──────────────────────────────────────────────────────────────────┐
│              OLTP vs OLAP                                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  OLTP (Online Transaction Processing)                           │
│  • Normalizado (3NF)                                             │
│  • Muitas tabelas, JOINs                                        │
│  • Queries simples e rápidas                                    │
│  • Alto volume de INSERT/UPDATE/DELETE                           │
│  • Modelo: ER (Entity-Relationship)                             │
│  • Ex: PostgreSQL, MySQL, Aurora                                │
│                                                                  │
│  OLAP (Online Analytical Processing)                            │
│  • Desnormalizado                                                │
│  • Poucas tabelas grandes (Star Schema)                         │
│  • Queries complexas com aggregations                           │
│  • Bulk loads, pouco UPDATE                                      │
│  • Modelo: Star Schema / Snowflake Schema                       │
│  • Ex: Redshift, BigQuery, Snowflake, ClickHouse                │
│                                                                  │
│  Star Schema:                                                    │
│        ┌──────────┐                                              │
│        │dim_time  │                                              │
│        └────┬─────┘                                              │
│  ┌─────────┐│┌──────────┐                                       │
│  │dim_user ├┼┤fact_sales│                                       │
│  └─────────┘│└────┬─────┘                                       │
│        ┌────┴─────┐                                              │
│        │dim_product│                                             │
│        └──────────┘                                              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Monitoramento e Operações

| Métrica | O que observar | Threshold de alerta |
|---------|---------------|-------------------|
| **Connections** | Active vs idle vs max | > 80% max_connections |
| **Query latency** | p50, p95, p99 | p99 > 500ms |
| **Slow queries** | Queries acima de threshold | > 1s (OLTP) |
| **Replication lag** | Delay entre primary e replica | > 10s |
| **Cache hit ratio** | Buffer cache effectiveness | < 99% (PostgreSQL) |
| **Dead tuples** | Rows que precisam de VACUUM | > 10% da tabela |
| **Lock waits** | Transações aguardando lock | > 5s |
| **Disk usage** | % de storage utilizado | > 80% |
| **IOPS** | Operações de I/O por segundo | Perto do limite provisioned |
| **TPS** | Transações por segundo | Baseline ± 30% |

```sql
-- PostgreSQL — Queries de monitoramento úteis

-- Top 10 queries mais lentas (pg_stat_statements extensão)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 10;

-- Índices não utilizados
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexname NOT LIKE 'pg_%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Tabelas precisando de VACUUM
SELECT schemaname, relname, n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Conexões ativas
SELECT state, count(*) 
FROM pg_stat_activity 
GROUP BY state;

-- Queries bloqueadas (locks)
SELECT blocked_locks.pid AS blocked_pid,
       blocked_activity.query AS blocked_query,
       blocking_locks.pid AS blocking_pid,
       blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity 
    ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.relation = blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity 
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

---

## Anti-patterns SQL

| # | Anti-pattern | Problema | Solução |
|---|-------------|----------|---------|
| 1 | **God Table** | Tabela com 100+ colunas | Decomposição em entidades menores |
| 2 | **EAV (Entity-Attribute-Value)** | Tabela genérica (entity, attribute, value) | JSONB para flexibilidade ou tabelas concretas |
| 3 | **Soft Delete sem indexar** | `WHERE deleted_at IS NULL` sem partial index | Partial index ou tabela de arquivo separada |
| 4 | **Storing files in DB** | BLOBs grandes no banco | Object storage (S3) com referência no banco |
| 5 | **No foreign keys** | Dados órfãos, integridade comprometida | FKs explícitas com ON DELETE adequado |
| 6 | **UUID v4 como PK clustered** | Fragmentação do índice, writes aleatórios | UUID v7 (time-ordered) ou BIGINT |
| 7 | **Business logic in triggers** | Debug impossível, surpresas | Lógica na aplicação, triggers para auditoria |
| 8 | **No prepared statements** | SQL Injection, parse overhead | Sempre usar parameterized queries |
| 9 | **Schema without constraints** | Dados inválidos, bugs silenciosos | CHECK, NOT NULL, FK, UNIQUE constraints |
| 10 | **No connection pooling** | Connection exhaustion, overhead | HikariCP, PgBouncer, RDS Proxy |

---

## Referências

- Kleppmann, M. (2017). *Designing Data-Intensive Applications* — O'Reilly
- Winand, M. *Use The Index, Luke* — https://use-the-index-luke.com/
- PostgreSQL. *Official Documentation* — https://www.postgresql.org/docs/
- Percona. *MySQL & PostgreSQL Performance Blog* — https://www.percona.com/blog/
- pganalyze. *PostgreSQL Performance* — https://pganalyze.com/docs
- Amazon. *Aurora Best Practices* — https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/
- HikariCP. *Pool Sizing* — https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing
