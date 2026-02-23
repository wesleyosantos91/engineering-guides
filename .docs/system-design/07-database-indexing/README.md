# 7. Database Indexing

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial — fundamental para performance de qualquer aplicação  
> **Complexidade:** Média-Alta (entender internals é o diferencial)

---

## Definição

**Index (Índice)** é uma estrutura de dados auxiliar que acelera operações de busca no banco de dados, reduzindo a quantidade de dados que precisam ser lidos do disco. Sem índice, o banco faz **full table scan** — lê todas as linhas.

> Analogia: Índice de um livro. Sem ele, você precisa ler página por página para encontrar um assunto. Com ele, vai direto à página.

---

## Por Que é Importante?

| Sem Índice | Com Índice |
|------------|-----------|
| Full table scan (O(n)) | Busca direta (O(log n)) |
| 1M rows → lê 1M rows | 1M rows → lê ~20 nodes da árvore |
| Query de 500ms | Query de 1ms |
| Lock de leitura em muitas páginas | Lock mínimo |

**Na prática:** A diferença entre um sistema que aguenta 100 req/s e um que aguenta 10.000 req/s muitas vezes está nos índices.

---

## Como Funciona Internamente

### Sem Índice — Full Table Scan

```
SELECT * FROM users WHERE email = 'joao@example.com';

Tabela users (1M rows, armazenada em páginas de 8KB):
┌──────────────────┐
│ Page 1: rows 1-100│  ← Lê do disco
│ Page 2: rows 101-200│  ← Lê do disco
│ Page 3: rows 201-300│  ← Lê do disco
│ ...                  │  ← Lê TUDO
│ Page 10000: rows...  │  ← Lê do disco
└──────────────────┘
Tempo: ~500ms (lê 10K páginas do disco)
```

### Com Índice — B-Tree Lookup

```
CREATE INDEX idx_users_email ON users(email);

SELECT * FROM users WHERE email = 'joao@example.com';

B-Tree Index:
                    ┌─────────────────────┐
         Level 0    │  [M]                │    ← Root (1 leitura)
                    └────┬─────────┬──────┘
                         │         │
              ┌──────────▼┐   ┌───▼──────────┐
   Level 1    │ [D, H]    │   │ [R, V]       │    ← 1 leitura
              └─┬───┬──┬──┘   └──┬───┬───┬───┘
                │   │  │         │   │   │
            ┌───▼┐ ┌▼─┐ ┌──▼┐  ┌▼──┐ ... 
   Level 2  │a-c │ │d-g│ │h-k│  │m-q│          ← 1 leitura
   (Leaf)   │    │ │   │ │   │  │   │
            └────┘ └───┘ └───┘  └───┘
                                  │
                           joao@example.com → row_id: 54321
                           
→ Aponta para: Page 543, Offset 21
→ 3-4 leituras de disco total
→ Tempo: ~1ms
```

---

## Tipos de Índice

### 1. B-Tree Index (Default)

A estrutura de dados mais comum em bancos de dados.

```
Características:
- Árvore balanceada (todas as folhas na mesma profundidade)
- Cada node = uma página de disco (~8-16KB)
- Suporta: =, <, >, <=, >=, BETWEEN, LIKE 'prefix%'
- NÃO suporta: LIKE '%suffix', funções no campo
```

| Profundidade | Rows que indexa | Leituras no pior caso |
|-------------|-----------------|----------------------|
| 1 | ~500 | 1 |
| 2 | ~250K | 2 |
| 3 | ~125M | 3 |
| 4 | ~62B | 4 |

> Com 3 níveis de B-Tree, você busca em 125 milhões de rows com apenas 3 leituras de disco!

**SQL:**
```sql
-- Index B-Tree padrão
CREATE INDEX idx_users_email ON users(email);

-- Queries que usam:
SELECT * FROM users WHERE email = 'joao@example.com';      -- ✓ Index Scan
SELECT * FROM users WHERE email LIKE 'joao%';               -- ✓ Index Scan
SELECT * FROM users WHERE email > 'j' AND email < 'k';      -- ✓ Range Scan
SELECT * FROM users ORDER BY email LIMIT 10;                 -- ✓ Index Scan

-- Queries que NÃO usam:
SELECT * FROM users WHERE UPPER(email) = 'JOAO@EXAMPLE.COM'; -- ✗ Full Scan
SELECT * FROM users WHERE email LIKE '%example.com';          -- ✗ Full Scan
```

### 2. Hash Index

Usa hash table para lookup O(1).

```
hash('joao@example.com') → bucket 42 → row_id 54321

Suporta:      = (equality only)
NÃO suporta:  <, >, range queries, ORDER BY
```

| Aspecto | B-Tree | Hash |
|---------|--------|------|
| Equality (=) | O(log n) | O(1) |
| Range (<, >) | Sim | Não |
| ORDER BY | Sim | Não |
| Uso de disco | Mais | Menos |
| Uso | Padrão | Memória (Redis, Memcached) |

### 3. Composite Index (Multi-Column)

```sql
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);
```

```
B-Tree ordenado por (user_id, created_at):

               ┌───────────────────────┐
               │ (user:500, 2024-01-01)│
               └───────┬───────────────┘
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│(100, 2024-01)│ │(500, 2024-01)│ │(800, 2024-01)│
│(100, 2024-02)│ │(500, 2024-02)│ │(800, 2024-02)│
│(200, 2024-01)│ │(500, 2024-03)│ │(900, 2024-01)│
│(200, 2024-03)│ │(600, 2024-01)│ │(999, 2024-05)│
└──────────────┘ └──────────────┘ └──────────────┘
```

**Regra do "Leftmost Prefix":**
```sql
-- Index: (user_id, created_at, status)

-- ✓ Usa index (prefixo completo)
WHERE user_id = 123 AND created_at > '2024-01-01' AND status = 'active'

-- ✓ Usa index (prefixo parcial)
WHERE user_id = 123 AND created_at > '2024-01-01'

-- ✓ Usa index (primeira coluna)
WHERE user_id = 123

-- ✗ NÃO usa index (pula user_id)
WHERE created_at > '2024-01-01'

-- ✗ NÃO usa index (pula user_id e created_at)
WHERE status = 'active'
```

### 4. Covering Index (Index-Only Scan)

O índice contém **todas** as colunas necessárias para a query — não precisa ir à tabela.

```sql
-- Index covering
CREATE INDEX idx_orders_cover ON orders(user_id, created_at, total_amount);

-- Esta query é resolvida INTEIRAMENTE pelo índice (index-only scan)
SELECT user_id, created_at, total_amount
FROM orders
WHERE user_id = 123
ORDER BY created_at DESC;

-- Sem ir à tabela! Ultra rápido.
```

```
Fluxo normal:     Index → encontra row_id → Table → lê row (2 I/O's)
Covering index:   Index → já tem os dados → retorna (1 I/O)
```

### 5. Partial Index (Conditional)

Indexa apenas um **subconjunto** dos dados.

```sql
-- Apenas orders ativas (90% são completed/cancelled)
CREATE INDEX idx_active_orders ON orders(user_id, created_at)
WHERE status = 'active';

-- Index menor, mais eficiente, mais rápido para updates
-- Muito útil quando só uma fração dos dados é consultada frequentemente
```

### 6. Expression/Functional Index

```sql
-- PostgreSQL: index em expressão
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Agora funciona:
SELECT * FROM users WHERE LOWER(email) = 'joao@example.com';  -- ✓ Index Scan
```

### 7. Full-Text Index (GIN/GiST)

Para busca textual eficiente.

```sql
-- PostgreSQL GIN (Generalized Inverted Index)
CREATE INDEX idx_articles_search ON articles USING GIN(to_tsvector('portuguese', content));

-- Busca full-text
SELECT * FROM articles
WHERE to_tsvector('portuguese', content) @@ to_tsquery('postgresql & indexação');
```

```
GIN Index (Inverted Index):
  "postgresql" → [doc_1, doc_5, doc_42]
  "indexação"  → [doc_5, doc_17, doc_42]
  "query"      → [doc_1, doc_3, doc_5]
  
  @@ 'postgresql & indexação' → intersect → [doc_5, doc_42]
```

### 8. Bitmap Index

```
Cada valor distinto tem um bitmap (array de bits):

status = 'active':     [1, 0, 1, 1, 0, 0, 1, 0, ...]
status = 'inactive':   [0, 1, 0, 0, 1, 0, 0, 1, ...]
status = 'pending':    [0, 0, 0, 0, 0, 1, 0, 0, ...]

WHERE status = 'active' AND region = 'BR':
  active:  [1, 0, 1, 1, 0, 0, 1, 0]
  BR:      [1, 1, 0, 1, 0, 0, 0, 1]
  AND =    [1, 0, 0, 1, 0, 0, 0, 0]  ← rows 1 e 4
```

- **Bom para:** Colunas com poucos valores distintos (low cardinality), data warehousing
- **Ruim para:** Colunas com muitos valores distintos, muitos writes (lock contention)

### Resumo de Tipos

| Tipo | Melhor Para | Operações | DB |
|------|-------------|-----------|-----|
| **B-Tree** | Default, range queries | =, <, >, BETWEEN, ORDER BY | Todos |
| **Hash** | Equality lookups | = only | PostgreSQL, memory engines |
| **GIN** | Full-text, JSONB, arrays | @>, @@, ? | PostgreSQL |
| **GiST** | Geográfico, range types | &&, @>, <-> | PostgreSQL |
| **BRIN** | Data time-series, muito grandes | Range scans em dados ordered | PostgreSQL |
| **Bitmap** | Low cardinality, OLAP | AND, OR de múltiplos | Oracle, PostgreSQL (runtime) |

---

## Analyze & Explain

### EXPLAIN (PostgreSQL)

```sql
-- Ver plano de execução
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;

-- Output:
Index Scan using idx_orders_user_id on orders  (cost=0.43..8.45 rows=5 width=120)
  Index Cond: (user_id = 123)
  Planning Time: 0.1 ms
  Execution Time: 0.05 ms    ← 0.05ms! Excelente.

-- Sem index:
Seq Scan on orders  (cost=0.00..25000.00 rows=5 width=120)
  Filter: (user_id = 123)
  Rows Removed by Filter: 999995
  Planning Time: 0.1 ms
  Execution Time: 450.0 ms   ← 450ms! Precisa de index.
```

### Key Metrics no EXPLAIN

| Métrica | Bom | Ruim |
|---------|-----|------|
| **Scan Type** | Index Scan, Index Only Scan | Seq Scan (em tabela grande) |
| **Rows Removed by Filter** | Baixo | Alto (index não está ajudando) |
| **Execution Time** | < 10ms | > 100ms |
| **Actual Rows vs Estimated** | Próximos | Muito diferentes (ANALYZE needed) |

```sql
-- Atualizar estatísticas (optimizer decide melhor)
ANALYZE orders;

-- Ver estatísticas de uso de índices
SELECT
    indexrelname AS index_name,
    idx_scan AS times_used,
    idx_tup_read AS rows_read,
    idx_tup_fetch AS rows_fetched
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;
```

---

## Custos de Índices

| Operação | Sem Index | Com Index |
|----------|-----------|-----------|
| **SELECT (lookup)** | O(n) — full scan | O(log n) — B-Tree |
| **INSERT** | O(1) — append | O(log n) — atualiza B-Tree |
| **UPDATE** | O(n) + write | O(log n) + reindex |
| **DELETE** | O(n) + write | O(log n) + reindex |
| **Disk space** | Só dados | Dados + índice (20-30% extra) |

> **Trade-off fundamental:** Índices aceleram reads mas tornam writes mais lentos.

---

## Boas Práticas

### Quando Criar Índice

```
✓ Colunas usadas em WHERE frequentemente
✓ Colunas usadas em JOIN (foreign keys)
✓ Colunas usadas em ORDER BY / GROUP BY
✓ Colunas com alta seletividade (muitos valores distintos)
✓ Tabelas grandes (>10K rows onde full scan é perceptível)
```

### Quando NÃO Criar

```
✗ Tabelas pequenas (<1K rows — full scan é rápido)
✗ Colunas com poucos valores distintos (exceto bitmap)
✗ Tabelas com muitos writes e poucos reads
✗ Colunas que raramente aparecem em WHERE
✗ Duplicar indexes que já existem como prefixo de composite
```

### Anti-Patterns

```sql
-- ✗ RUIM: Index em TODA coluna
CREATE INDEX idx1 ON users(name);
CREATE INDEX idx2 ON users(email);
CREATE INDEX idx3 ON users(age);
CREATE INDEX idx4 ON users(city);
CREATE INDEX idx5 ON users(created_at);
-- Problema: cada INSERT atualiza 5 B-Trees!

-- ✓ BOM: Index compostos estratégicos
CREATE INDEX idx_users_lookup ON users(email);  -- login
CREATE INDEX idx_users_search ON users(city, age);  -- busca
-- 2 indexes cobrem os casos de uso

-- ✗ RUIM: Index redundante
CREATE INDEX idx_a ON orders(user_id);
CREATE INDEX idx_b ON orders(user_id, created_at);
-- idx_a é redundante! idx_b cobre queries WHERE user_id = ?

-- ✗ RUIM: Função no campo indexado
WHERE YEAR(created_at) = 2024
-- ✓ BOM: Range query
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
```

---

## Uso em Big Techs

### Google — Bigtable / Spanner
- Bigtable: Dados ordenados por row key (LSM-tree based)
- Spanner: Índices secundários distribuídos globalmente
- Index interleaving para co-localizar child/parent data

### Amazon — DynamoDB
- Primary key (partition key + sort key) como index principal
- GSI (Global Secondary Index): index em qualquer atributo
- LSI (Local Secondary Index): index com mesma partition key
- Custo: GSI = tabela duplicada (dobra custo de storage e write)

### Uber — Schemaless (MySQL)
- Index no MySQL para dados acessados por key
- Write path otimizado com indexes mínimos
- Indexes denormalizados para padrões de query específicos

---

## Perguntas Comuns em Entrevistas

1. **O que é um B-Tree index?** → Árvore balanceada onde cada node é uma página de disco. Suporta range queries, O(log n) lookup
2. **Composite index: ordem das colunas importa?** → Sim! Leftmost prefix rule. Coloquue a coluna mais seletiva ou usada em equality primeiro
3. **Index acelera writes?** → Não, torna writes MAIS LENTOS (precisa atualizar B-Tree). Trade-off read vs write
4. **Covering index?** → Index que contém todas as colunas da query. Index-only scan sem ir à tabela
5. **Quando NÃO usar index?** → Tabelas pequenas, colunas low cardinality, tabelas write-heavy

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Read vs Write perf** | Mais indexes (reads rápidos) | Menos indexes (writes rápidos) |
| **Single vs Composite** | Indexes simples (flexível) | Composite (otimizado para query) |
| **B-Tree vs Hash** | B-Tree (versátil) | Hash (O(1) equality) |
| **Full index vs Partial** | Full (cobre tudo) | Partial (menor, mais eficiente) |
| **Space vs Speed** | Covering index (rápido) | Minimal index (menos espaço) |

---

## Referências

- [PostgreSQL Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- [Use The Index, Luke](https://use-the-index-luke.com/) — Guia completo de indexação SQL
- [MySQL Index Documentation](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html)
- Designing Data-Intensive Applications — Martin Kleppmann, Chapter 3
- Database Internals — Alex Petrov
