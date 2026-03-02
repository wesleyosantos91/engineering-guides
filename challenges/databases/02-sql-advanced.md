# Level 2 — SQL Avançado: Indexação, Queries & Transações

> **Objetivo:** Dominar indexação estratégica (B-Tree, GIN, parcial, composto), otimização
> de queries com EXPLAIN ANALYZE, locking strategies e padrões avançados de SQL.

**Referência:** [.docs/databases/02-sql-best-practices.md](../../.docs/databases/02-sql-best-practices.md)

---

## Contexto do Domínio

Com o schema da StreamX criado no Level 1, agora você vai **otimizar** o acesso aos dados.
Cenários reais: listar assinaturas ativas, processar pagamentos concorrentes, buscar
usuários com full-text search, gerar relatórios de billing.

---

## Desafios

### Desafio 2.1 — Estratégia de Indexação com ESR Rule

**Contexto:** A StreamX tem queries lentas em produção. Aplique a regra ESR
(Equality → Sort → Range) para criar índices eficientes.

**Requisitos:**

- Analisar e indexar as seguintes queries:

```sql
-- Query 1: Listar assinaturas ativas de um usuário, mais recentes primeiro
SELECT * FROM subscriptions
WHERE user_id = :user_id AND status = 'ACTIVE'
ORDER BY created_at DESC;
-- ESR: Equality(user_id, status) → Sort(created_at DESC) → Range(nenhum)

-- Query 2: Buscar pagamentos pendentes dentro de um período
SELECT * FROM payments
WHERE status = 'PENDING' AND created_at BETWEEN :start AND :end
ORDER BY created_at;
-- ESR: Equality(status) → Sort(created_at) → Range(created_at)

-- Query 3: Buscar usuários por email (case-insensitive)
SELECT * FROM users
WHERE LOWER(email) = LOWER(:email) AND deleted_at IS NULL;

-- Query 4: Listar top conteúdos por view_count
SELECT id, title, view_count FROM content
WHERE is_published = TRUE
ORDER BY view_count DESC
LIMIT 100;
```

- Para cada query:
  - Criar o índice seguindo a regra ESR
  - Executar `EXPLAIN ANALYZE` antes e depois do índice
  - Documentar: custo estimado, actual time, rows examinados, index scan vs seq scan

```sql
-- Índice ESR para Query 1
CREATE INDEX idx_subscriptions_user_status_created
    ON subscriptions(user_id, status, created_at DESC);

-- Índice funcional + partial para Query 3
CREATE INDEX idx_users_email_lower_active
    ON users(LOWER(email)) WHERE deleted_at IS NULL;

-- Covering index para Query 4 (evita table lookup)
CREATE INDEX idx_content_published_views
    ON content(is_published, view_count DESC) INCLUDE (id, title);
```

**Critérios de aceite:**

- [ ] 4 índices criados seguindo regra ESR
- [ ] `EXPLAIN ANALYZE` antes/depois para cada query (screenshot ou output colado)
- [ ] Demonstrar transformação de Seq Scan → Index Scan
- [ ] Covering index demonstrado (Index Only Scan)
- [ ] Partial index funcional (filtro `WHERE deleted_at IS NULL`)
- [ ] Documento comparativo: custo antes vs depois

---

### Desafio 2.2 — Tipos de Índice Especializados

**Contexto:** Nem toda query é resolvida com B-Tree. A StreamX precisa de
full-text search, queries JSONB e range queries em dados temporais.

**Requisitos:**

- Implementar 4 tipos de índice diferentes:

```sql
-- 1. GIN para Full-Text Search — buscar conteúdo por título/descrição
ALTER TABLE content ADD COLUMN search_vector TSVECTOR;
UPDATE content SET search_vector = to_tsvector('portuguese', title || ' ' || description);

CREATE INDEX idx_content_search ON content USING GIN(search_vector);

-- Query: buscar "ficção científica espaço"
SELECT id, title, ts_rank(search_vector, query) AS rank
FROM content, to_tsquery('portuguese', 'ficção & científica & espaço') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;

-- 2. GIN para JSONB — content metadata flexível
ALTER TABLE content ADD COLUMN metadata JSONB NOT NULL DEFAULT '{}';
CREATE INDEX idx_content_metadata ON content USING GIN(metadata jsonb_path_ops);

-- Query: conteúdos com audio em português
SELECT * FROM content WHERE metadata @> '{"audio_languages": ["pt-BR"]}';

-- 3. BRIN para dados temporais ordenados — tabela de watch events
CREATE INDEX idx_watch_events_created_brin ON watch_events USING BRIN(created_at);
-- BRIN é ideal para dados inseridos em ordem cronológica (storage mínimo)

-- 4. Hash para equality-only — busca por idempotency_key
CREATE INDEX idx_payments_idempotency_hash ON payments USING HASH(idempotency_key);
```

**Critérios de aceite:**

- [ ] GIN com tsvector funcional: full-text search em português retorna resultados rankeados
- [ ] GIN com JSONB funcional: query por path dentro de JSON
- [ ] BRIN demonstrado em tabela de dados temporais (comparar tamanho vs B-Tree)
- [ ] Hash index demonstrado para equality lookup
- [ ] Tabela comparativa: quando usar cada tipo de índice (B-Tree vs GIN vs GiST vs BRIN vs Hash)

---

### Desafio 2.3 — Query Optimization: Eliminar Anti-patterns

**Contexto:** Code review encontrou queries problemáticas no codebase da StreamX.
Refatore-as.

**Requisitos:**

- Refatorar cada anti-pattern:

| # | Anti-pattern | Query Problemática | Solução |
|---|-------------|-------------------|---------|
| 1 | `SELECT *` | `SELECT * FROM users WHERE id = ?` | Listar apenas colunas necessárias |
| 2 | N+1 queries | Loop: `SELECT * FROM payments WHERE sub_id = ?` | JOIN ou batch `WHERE sub_id IN (...)` |
| 3 | Function on index | `WHERE YEAR(created_at) = 2025` | `WHERE created_at >= '2025-01-01' AND ...` |
| 4 | OFFSET pagination | `LIMIT 20 OFFSET 100000` | Keyset pagination com cursor |
| 5 | Leading wildcard | `WHERE title LIKE '%matrix%'` | Full-text search com GIN |
| 6 | `NOT IN` with NULLs | `WHERE id NOT IN (SELECT ...)` | `NOT EXISTS` |
| 7 | Correlated subquery | Subquery executada N vezes | JOIN ou CTE |

- Para cada refatoração, mostrar `EXPLAIN ANALYZE` antes e depois

**Implementar keyset pagination em Java/Go:**

**Java 25:**
```java
public record PageCursor(long lastId, Instant lastCreatedAt) {}

public List<Subscription> findActive(PageCursor cursor, int size) {
    var sql = """
        SELECT id, external_id, user_id, plan_id, status, created_at
        FROM subscriptions
        WHERE status = 'ACTIVE'
          AND (created_at, id) < (?, ?)
        ORDER BY created_at DESC, id DESC
        LIMIT ?
        """;
    try (var conn = dataSource.getConnection();
         var stmt = conn.prepareStatement(sql)) {
        stmt.setTimestamp(1, Timestamp.from(cursor.lastCreatedAt()));
        stmt.setLong(2, cursor.lastId());
        stmt.setInt(3, size);
        // ... map results
    }
}
```

**Go 1.26:**
```go
type PageCursor struct {
    LastID        int64
    LastCreatedAt time.Time
}

func (r *SubscriptionRepo) FindActive(ctx context.Context, cursor PageCursor, size int) ([]Subscription, error) {
    rows, err := r.db.QueryContext(ctx, `
        SELECT id, external_id, user_id, plan_id, status, created_at
        FROM subscriptions
        WHERE status = 'ACTIVE'
          AND (created_at, id) < ($1, $2)
        ORDER BY created_at DESC, id DESC
        LIMIT $3`, cursor.LastCreatedAt, cursor.LastID, size)
    if err != nil {
        return nil, fmt.Errorf("find active subs: %w", err)
    }
    defer rows.Close()
    // ... scan results
}
```

**Critérios de aceite:**

- [ ] 7 anti-patterns refatorados com antes/depois
- [ ] Keyset pagination implementada em Java e Go
- [ ] `EXPLAIN ANALYZE` para pelo menos 3 refatorações
- [ ] Redução mensurável de custo/tempo em todas as queries
- [ ] Nenhum `SELECT *` remanescente no codebase

---

### Desafio 2.4 — Transações & Locking Strategies

**Contexto:** A StreamX processa pagamentos de assinatura. Concorrência incorreta pode
causar cobranças duplicadas ou race conditions no saldo.

**Requisitos:**

- Implementar **Optimistic Locking** para atualizar status de assinatura:

```sql
-- Optimistic Locking com versão
ALTER TABLE subscriptions ADD COLUMN version INT NOT NULL DEFAULT 0;

-- Na aplicação: ler versão atual, tentar update com WHERE version = ?
UPDATE subscriptions
SET status = 'ACTIVE', version = version + 1, updated_at = NOW()
WHERE id = :id AND version = :expected_version;
-- Se affected_rows = 0 → conflito → retry
```

- Implementar **Pessimistic Locking** com `SELECT FOR UPDATE` para processamento de pagamento:

```sql
-- Worker processa um pagamento pendente de cada vez
BEGIN;
SELECT * FROM payments
WHERE status = 'PENDING'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;  -- não bloqueia se outro worker já pegou

-- processar pagamento...
UPDATE payments SET status = 'COMPLETED', paid_at = NOW() WHERE id = :id;
COMMIT;
```

- Implementar **Advisory Lock** para job exclusivo:

```sql
-- Apenas um cron job de billing pode rodar por vez
SELECT pg_try_advisory_lock(42);  -- 42 = hash do job "billing"
-- Se retornou true: rodar job
-- Se retornou false: outra instância já está rodando
SELECT pg_advisory_unlock(42);
```

**Java 25:**
```java
// Optimistic Locking
public boolean updateStatus(long id, String newStatus, int expectedVersion) {
    try (var conn = dataSource.getConnection();
         var stmt = conn.prepareStatement(
             "UPDATE subscriptions SET status = ?, version = version + 1, updated_at = NOW() WHERE id = ? AND version = ?")) {
        stmt.setString(1, newStatus);
        stmt.setLong(2, id);
        stmt.setInt(3, expectedVersion);
        return stmt.executeUpdate() > 0; // false = conflict
    }
}

// Pessimistic Locking — Worker queue
public Optional<Payment> claimNextPending() {
    try (var conn = dataSource.getConnection()) {
        conn.setAutoCommit(false);
        try (var stmt = conn.prepareStatement(
                "SELECT id, amount, subscription_id FROM payments WHERE status = 'PENDING' ORDER BY created_at LIMIT 1 FOR UPDATE SKIP LOCKED")) {
            var rs = stmt.executeQuery();
            if (!rs.next()) return Optional.empty();
            var payment = mapPayment(rs);
            // process...
            try (var update = conn.prepareStatement(
                    "UPDATE payments SET status = 'COMPLETED', paid_at = NOW() WHERE id = ?")) {
                update.setLong(1, payment.id());
                update.executeUpdate();
            }
            conn.commit();
            return Optional.of(payment);
        } catch (Exception e) {
            conn.rollback();
            throw e;
        }
    }
}
```

**Go 1.26:**
```go
// Optimistic Locking
func (r *SubscriptionRepo) UpdateStatus(ctx context.Context, id int64, newStatus string, expectedVersion int) (bool, error) {
    res, err := r.db.ExecContext(ctx,
        "UPDATE subscriptions SET status = $1, version = version + 1, updated_at = NOW() WHERE id = $2 AND version = $3",
        newStatus, id, expectedVersion)
    if err != nil {
        return false, fmt.Errorf("update status: %w", err)
    }
    rows, _ := res.RowsAffected()
    return rows > 0, nil // false = conflict → retry
}
```

**Critérios de aceite:**

- [ ] Optimistic locking funcional com coluna `version` (Java e Go)
- [ ] Teste de conflito: 2 threads/goroutines tentam atualizar a mesma row
- [ ] Pessimistic locking com `FOR UPDATE SKIP LOCKED` funcional
- [ ] Advisory lock demonstrado para job exclusivo
- [ ] Documento: `decisions/02-locking-strategy.md` justificando quando usar cada abordagem

---

### Desafio 2.5 — CTEs, Window Functions & Queries Analíticas

**Contexto:** A product team da StreamX precisa de dashboards. Implemente queries
analíticas com CTEs e window functions.

**Requisitos:**

```sql
-- 1. CTE Recursiva: Categorias hierárquicas de conteúdo
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 AS depth, name::TEXT AS path
    FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.name, c.parent_id, ct.depth + 1, ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY path;

-- 2. Window Function: Ranking de conteúdos por gênero
SELECT
    g.name AS genre,
    c.title,
    c.view_count,
    ROW_NUMBER() OVER (PARTITION BY g.id ORDER BY c.view_count DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY c.view_count DESC) AS global_rank
FROM content c
JOIN content_genres cg ON cg.content_id = c.id
JOIN genres g ON g.id = cg.genre_id
WHERE c.is_published = TRUE;

-- 3. Billing report: receita mensal com running total
SELECT
    DATE_TRUNC('month', p.paid_at) AS month,
    COUNT(*) AS payment_count,
    SUM(p.amount) AS monthly_revenue,
    SUM(SUM(p.amount)) OVER (ORDER BY DATE_TRUNC('month', p.paid_at)) AS cumulative_revenue,
    LAG(SUM(p.amount)) OVER (ORDER BY DATE_TRUNC('month', p.paid_at)) AS prev_month_revenue,
    ROUND(
        (SUM(p.amount) - LAG(SUM(p.amount)) OVER (ORDER BY DATE_TRUNC('month', p.paid_at)))
        / NULLIF(LAG(SUM(p.amount)) OVER (ORDER BY DATE_TRUNC('month', p.paid_at)), 0) * 100, 2
    ) AS growth_pct
FROM payments p
WHERE p.status = 'COMPLETED'
GROUP BY DATE_TRUNC('month', p.paid_at)
ORDER BY month;

-- 4. Churn: usuários que cancelaram nos últimos 30 dias
WITH churned AS (
    SELECT user_id, cancelled_at,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY cancelled_at DESC) AS rn
    FROM subscriptions
    WHERE status = 'CANCELLED' AND cancelled_at >= NOW() - INTERVAL '30 days'
)
SELECT u.id, u.name, u.email, c.cancelled_at
FROM churned c
JOIN users u ON u.id = c.user_id
WHERE c.rn = 1;

-- 5. Upsert com ON CONFLICT — idempotent payment processing
INSERT INTO payments (id, subscription_id, amount, currency, status, idempotency_key, created_at)
VALUES (:id, :sub_id, :amount, :currency, 'COMPLETED', :idem_key, NOW())
ON CONFLICT (idempotency_key) DO NOTHING
RETURNING id, status;
```

**Critérios de aceite:**

- [ ] CTE recursiva para categorias hierárquicas funcional
- [ ] Window functions: `ROW_NUMBER`, `RANK`, `LAG`, `SUM() OVER` demonstrados
- [ ] Report de receita com running total e growth %
- [ ] Churn query identificando cancelamentos recentes
- [ ] Upsert idempotente com `ON CONFLICT` funcional
- [ ] Todas as queries executam em < 500ms com 100K+ rows (demonstrar com `EXPLAIN ANALYZE`)

---

### Desafio 2.6 — Connection Pooling & Management

**Contexto:** A StreamX tem 50 instâncias de aplicação conectando ao PostgreSQL.
Sem connection pooling, o banco atinge `max_connections` e novas conexões falham.

**Requisitos:**

- Implementar connection pool em Java e Go:

**Java 25 (HikariCP):**
```java
var config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/streamx");
config.setUsername("app");
config.setPassword("secret");
config.setMaximumPoolSize(16);           // cores * 2
config.setMinimumIdle(16);               // warm pool
config.setConnectionTimeout(30_000);      // 30s
config.setIdleTimeout(600_000);           // 10min
config.setMaxLifetime(1_800_000);         // 30min
config.setLeakDetectionThreshold(60_000); // 60s

var dataSource = new HikariDataSource(config);
```

**Go 1.26 (database/sql + pgx):**
```go
db, err := sql.Open("pgx", "postgres://app:secret@localhost:5432/streamx")
if err != nil {
    log.Fatal(err)
}

db.SetMaxOpenConns(16)              // cores * 2
db.SetMaxIdleConns(16)              // warm pool
db.SetConnMaxLifetime(30 * time.Minute)
db.SetConnMaxIdleTime(10 * time.Minute)
```

- Documentar a fórmula de pool sizing: `connections = (cores × 2) + effective_spindle_count`
- Implementar health check que monitora pool stats (active, idle, waiting)
- Simular connection leak e demonstrar detecção (Java: `leakDetectionThreshold`)

**Critérios de aceite:**

- [ ] Connection pool configurado em Java (HikariCP) e Go (`database/sql`)
- [ ] Pool sizing justificado com fórmula
- [ ] Health check endpoint que expõe pool stats
- [ ] Teste: simular 100 requests concorrentes, pool reutiliza conexões (não cria 100)
- [ ] Demonstrar leak detection em Java
- [ ] Documento: `decisions/02-connection-pool-sizing.md`

---

### Desafio 2.7 — Batch Operations & Bulk Loading

**Contexto:** A StreamX precisa importar catálogo de conteúdo (100K+ registros)
e processar pagamentos em batch.

**Requisitos:**

- Implementar bulk insert com `COPY` e batch insert com `UNNEST`:

```sql
-- COPY para carga massiva (10-100x mais rápido que INSERT individual)
COPY content (title, description, content_type, is_published, created_at)
FROM '/tmp/content_catalog.csv' WITH CSV HEADER;

-- Batch upsert com UNNEST
INSERT INTO content (external_id, title, content_type)
SELECT * FROM UNNEST(
    ARRAY['uuid-1', 'uuid-2', 'uuid-3']::UUID[],
    ARRAY['Matrix', 'Inception', 'Interstellar']::VARCHAR[],
    ARRAY['MOVIE', 'MOVIE', 'MOVIE']::VARCHAR[]
)
ON CONFLICT (external_id) DO UPDATE SET
    title = EXCLUDED.title,
    updated_at = NOW();
```

- Implementar batch update controlado (evitar lock prolongado):

```sql
-- Batch update com controle de tamanho
WITH batch AS (
    SELECT id FROM payments
    WHERE status = 'PENDING' AND created_at < NOW() - INTERVAL '7 days'
    LIMIT 5000
    FOR UPDATE SKIP LOCKED
)
UPDATE payments SET status = 'EXPIRED'
WHERE id IN (SELECT id FROM batch);
```

- Comparar performance: INSERT individual vs batch INSERT vs COPY

**Java 25:**
```java
// Batch insert com PreparedStatement
try (var conn = dataSource.getConnection()) {
    conn.setAutoCommit(false);
    try (var stmt = conn.prepareStatement(
            "INSERT INTO content (external_id, title, content_type) VALUES (?, ?, ?)")) {
        for (var content : contentList) {
            stmt.setObject(1, content.externalId());
            stmt.setString(2, content.title());
            stmt.setString(3, content.type());
            stmt.addBatch();
            if (++count % 5000 == 0) {
                stmt.executeBatch();
                conn.commit();
            }
        }
        stmt.executeBatch();
        conn.commit();
    }
}
```

**Critérios de aceite:**

- [ ] Bulk insert com `COPY` funcional para 100K+ registros
- [ ] Batch upsert com `UNNEST` funcional
- [ ] Batch update com `SKIP LOCKED` (não bloqueia tabela)
- [ ] Benchmark comparativo: individual vs batch vs COPY (ms, rows/s)
- [ ] Batch insert em Java e Go com commit a cada N registros
- [ ] Demonstrar que batch update não causa lock wait timeout

---

### Desafio 2.8 — Aggregations com FILTER, LATERAL & Advanced SQL

**Contexto:** Queries analíticas avançadas para dashboards e relatórios da StreamX.

**Requisitos:**

```sql
-- 1. FILTER — múltiplas aggregations condicionais em uma query
SELECT
    DATE_TRUNC('month', created_at) AS month,
    COUNT(*) FILTER (WHERE status = 'ACTIVE') AS active_subs,
    COUNT(*) FILTER (WHERE status = 'CANCELLED') AS cancelled_subs,
    SUM(price) FILTER (WHERE status = 'ACTIVE' AND billing_cycle = 'MONTHLY') AS mrr,
    SUM(price) FILTER (WHERE status = 'ACTIVE' AND billing_cycle = 'ANNUAL') / 12 AS arr_monthly
FROM subscriptions
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month;

-- 2. LATERAL JOIN — top 3 conteúdos por gênero
SELECT g.name AS genre, top.title, top.view_count
FROM genres g
CROSS JOIN LATERAL (
    SELECT c.title, c.view_count
    FROM content c
    JOIN content_genres cg ON cg.content_id = c.id
    WHERE cg.genre_id = g.id AND c.is_published = TRUE
    ORDER BY c.view_count DESC
    LIMIT 3
) top;

-- 3. GROUPING SETS — relatório multi-dimensão
SELECT
    COALESCE(pl.name, 'ALL PLANS') AS plan_name,
    COALESCE(s.billing_cycle, 'ALL CYCLES') AS cycle,
    COUNT(*) AS total_subs,
    SUM(s.price) AS total_revenue
FROM subscriptions s
JOIN plans pl ON pl.id = s.plan_id
WHERE s.status = 'ACTIVE'
GROUP BY GROUPING SETS (
    (pl.name, s.billing_cycle),    -- detail
    (pl.name),                      -- subtotal by plan
    (s.billing_cycle),              -- subtotal by cycle
    ()                              -- grand total
)
ORDER BY pl.name NULLS LAST, s.billing_cycle NULLS LAST;
```

**Critérios de aceite:**

- [ ] `FILTER` demonstrado com pelo menos 3 aggregations condicionais
- [ ] `LATERAL JOIN` funcional para top-N por grupo
- [ ] `GROUPING SETS` com subtotais e grand total
- [ ] Todas as queries executam com performance aceitável (< 1s em 100K+ rows)
- [ ] Java e Go executando as queries e mapeando resultados para DTOs/structs
