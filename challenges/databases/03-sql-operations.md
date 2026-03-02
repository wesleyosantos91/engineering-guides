# Level 3 — SQL Operacional: Particionamento, Replicação & Migrations

> **Objetivo:** Dominar operações de produção em PostgreSQL — particionamento de tabelas,
> replicação para escalar leitura, migrations zero-downtime e monitoramento.

**Referência:** [.docs/databases/02-sql-best-practices.md](../../.docs/databases/02-sql-best-practices.md)

---

## Contexto do Domínio

A StreamX cresceu: a tabela `payments` tem 500M+ rows, os relatórios de billing degradam
o primary, e atualizações de schema precisam ser feitas sem downtime. É hora de operar
o PostgreSQL como um sistema de produção real.

---

## Desafios

### Desafio 3.1 — Particionamento de Tabelas

**Contexto:** A tabela `watch_events` da StreamX recebe 50M de INSERTs por dia.
Queries filtram sempre por data. Sem particionamento, scans ficam cada vez mais lentos.

**Requisitos:**

- Implementar **Range Partitioning** por mês na tabela `watch_events`:

```sql
CREATE TABLE watch_events (
    id          BIGINT GENERATED ALWAYS AS IDENTITY,
    user_id     BIGINT NOT NULL,
    content_id  BIGINT NOT NULL,
    event_type  VARCHAR(20) NOT NULL,  -- PLAY, PAUSE, SEEK, STOP, COMPLETE
    duration_ms BIGINT,
    metadata    JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Criar partições mensais
CREATE TABLE watch_events_2025_01 PARTITION OF watch_events
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE watch_events_2025_02 PARTITION OF watch_events
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
-- ... demais meses

-- Índices são criados automaticamente por partição
CREATE INDEX idx_watch_events_user_created
    ON watch_events(user_id, created_at DESC);
```

- Demonstrar **partition pruning**: query com `WHERE created_at >= '2025-01-15'`
  deve escanear apenas a partição de janeiro
- Implementar **List Partitioning** para `subscriptions` por status:

```sql
CREATE TABLE subscriptions_partitioned (
    -- mesmas colunas...
) PARTITION BY LIST (status);

CREATE TABLE subscriptions_active PARTITION OF subscriptions_partitioned
    FOR VALUES IN ('ACTIVE', 'PENDING');
CREATE TABLE subscriptions_inactive PARTITION OF subscriptions_partitioned
    FOR VALUES IN ('EXPIRED', 'CANCELLED', 'SUSPENDED');
```

- Implementar rotina de criação automática de partições futuras:

```sql
-- Criar partição para o próximo mês se não existir
DO $$
DECLARE
    next_month DATE := DATE_TRUNC('month', NOW() + INTERVAL '1 month');
    partition_name TEXT;
BEGIN
    partition_name := 'watch_events_' || TO_CHAR(next_month, 'YYYY_MM');
    IF NOT EXISTS (SELECT 1 FROM pg_class WHERE relname = partition_name) THEN
        EXECUTE FORMAT(
            'CREATE TABLE %I PARTITION OF watch_events FOR VALUES FROM (%L) TO (%L)',
            partition_name,
            next_month,
            next_month + INTERVAL '1 month'
        );
    END IF;
END $$;
```

- Demonstrar `DROP PARTITION` vs `DELETE` (performance e sem VACUUM)

**Critérios de aceite:**

- [ ] Range partitioning funcional em `watch_events` (pelo menos 3 meses)
- [ ] `EXPLAIN` demonstrando partition pruning
- [ ] List partitioning em `subscriptions` por status
- [ ] Script de criação automática de partições futuras
- [ ] Benchmark: `DELETE` 1M rows vs `DROP PARTITION` (tempo e VACUUM)
- [ ] Tabela de decisão: quando particionar (critérios de tamanho, query pattern)

---

### Desafio 3.2 — Replicação Primary-Replica

**Contexto:** Os dashboards e relatórios analíticos da StreamX degradam o primary database.
Configure replicação para direcionar reads analíticos para réplicas.

**Requisitos:**

- Configurar replicação **Primary-Replica** com Docker Compose:
  - 1 Primary (R/W) + 2 Replicas (R/O)
  - Streaming replication (WAL shipping)
  - Réplicas em modo hot-standby
- Implementar read/write splitting na aplicação:

**Java 25:**
```java
public class RoutingDataSource {
    private final DataSource primary;
    private final DataSource replica;

    public DataSource getDataSource(boolean readOnly) {
        return readOnly ? replica : primary;
    }

    // Para queries de leitura (dashboards, relatórios)
    public List<DashboardRow> getMonthlyRevenue() {
        try (var conn = getDataSource(true).getConnection()) {
            // executa na réplica
        }
    }

    // Para writes (criar assinatura, processar pagamento)
    public void createSubscription(Subscription sub) {
        try (var conn = getDataSource(false).getConnection()) {
            // executa no primary
        }
    }
}
```

**Go 1.26:**
```go
type DBRouter struct {
    primary *sql.DB
    replica *sql.DB
}

func (r *DBRouter) DB(readOnly bool) *sql.DB {
    if readOnly {
        return r.replica
    }
    return r.primary
}

// Read-after-write: após criar, ler do primary para evitar replication lag
func (r *DBRouter) CreateAndRead(ctx context.Context, sub *Subscription) (*Subscription, error) {
    // write no primary
    if err := r.insertSubscription(ctx, r.primary, sub); err != nil {
        return nil, err
    }
    // leitura imediata do primary (não da réplica)
    return r.findByID(ctx, r.primary, sub.ID)
}
```

- Documentar os 3 problemas de replication lag:
  1. **Read-after-write** — usuário salva e não vê (solução: ler do primary após write)
  2. **Monotonic reads** — refresh mostra dado antigo (solução: sticky session)
  3. **Stale reads** — réplica atrasada serve dado velho (solução: tolerar ou strong read)
- Monitorar replication lag entre primary e réplica

**Critérios de aceite:**

- [ ] Docker Compose com 1 primary + 2 replicas funcional
- [ ] Read/write splitting implementado em Java e Go
- [ ] Read-after-write pattern: após create, lê do primary
- [ ] Documentação dos 3 problemas de replication lag com soluções
- [ ] Query de monitoramento de lag: `SELECT pg_last_wal_replay_lsn()` etc.
- [ ] Teste: assinatura criada no primary é lida corretamente na réplica (eventual)

---

### Desafio 3.3 — Migrations Zero-Downtime

**Contexto:** A StreamX roda 24/7. Schema changes precisam ser aplicadas **sem parar a aplicação**.

**Requisitos:**

- Implementar migrations com Flyway (Java) e golang-migrate (Go)
- Demonstrar operações **safe** (sem lock prolongado):

```sql
-- ✅ ADD COLUMN — sem lock (PostgreSQL)
ALTER TABLE subscriptions ADD COLUMN trial_ends_at TIMESTAMPTZ;

-- ✅ ADD COLUMN NOT NULL com DEFAULT — lock breve (PG 11+)
ALTER TABLE users ADD COLUMN preferred_language VARCHAR(5) NOT NULL DEFAULT 'pt-BR';

-- ✅ CREATE INDEX CONCURRENTLY — sem bloquear writes
CREATE INDEX CONCURRENTLY idx_subscriptions_trial
    ON subscriptions(trial_ends_at) WHERE trial_ends_at IS NOT NULL;
```

- Demonstrar operação **perigosa** e sua solução com **Expand-and-Contract**:

```
Cenário: Renomear coluna "status" para "subscription_status"

Deploy 1 (Expand): Adicionar nova coluna
  ALTER TABLE subscriptions ADD COLUMN subscription_status VARCHAR(20);

Deploy 2 (Dual-Write): Escrever em ambas as colunas
  UPDATE subscriptions SET subscription_status = status WHERE subscription_status IS NULL;
  -- Trigger: novos writes populam ambas

Deploy 3 (Backfill): Migrar dados antigos
  -- Batch update em background

Deploy 4 (Switch Read): Aplicação lê da nova coluna

Deploy 5 (Contract): Remover coluna antiga
  ALTER TABLE subscriptions DROP COLUMN status;
```

- Criar migrations versionadas e reversíveis:

```
migrations/
├── V001__create_users.sql
├── V001__create_users_down.sql
├── V002__create_plans.sql
├── V003__create_subscriptions.sql
├── V004__add_trial_column.sql
└── V005__rename_status_expand.sql
```

**Critérios de aceite:**

- [ ] Flyway (Java) ou golang-migrate (Go) configurado e funcional
- [ ] 5+ migrations versionadas com UP e DOWN
- [ ] `CREATE INDEX CONCURRENTLY` demonstrado (sem lock)
- [ ] Expand-and-Contract pattern implementado em 3+ deploys
- [ ] Nenhum `ALTER TABLE RENAME COLUMN` direto (demonstrar por que é perigoso)
- [ ] Documento: `decisions/03-migration-strategy.md`

---

### Desafio 3.4 — OLTP vs OLAP: Modelagem Star Schema

**Contexto:** A StreamX precisa de analytics: "receita por plano por mês", "churn rate",
"top conteúdos por região". O schema OLTP (3NF) não é otimizado para isso.

**Requisitos:**

- Criar um Star Schema para analytics da StreamX:

```sql
-- Tabelas de Dimensão
CREATE TABLE dim_time (
    id          INT PRIMARY KEY,
    full_date   DATE NOT NULL,
    year        INT NOT NULL,
    quarter     INT NOT NULL,
    month       INT NOT NULL,
    month_name  VARCHAR(20) NOT NULL,
    day         INT NOT NULL,
    day_of_week INT NOT NULL,
    is_weekend  BOOLEAN NOT NULL
);

CREATE TABLE dim_plan (
    id              INT PRIMARY KEY,
    plan_name       VARCHAR(100) NOT NULL,
    max_resolution  VARCHAR(10) NOT NULL,
    price_tier      VARCHAR(20) NOT NULL  -- 'Basic', 'Standard', 'Premium'
);

CREATE TABLE dim_user (
    id              INT PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    age_group       VARCHAR(20),
    country         VARCHAR(50),
    signup_cohort   VARCHAR(20)  -- 'Q1-2025', 'Q2-2025'
);

-- Tabela Fato
CREATE TABLE fact_subscriptions (
    id              BIGINT PRIMARY KEY,
    time_id         INT REFERENCES dim_time(id),
    plan_id         INT REFERENCES dim_plan(id),
    user_id         INT REFERENCES dim_user(id),
    event_type      VARCHAR(20) NOT NULL,  -- 'NEW', 'RENEWAL', 'UPGRADE', 'DOWNGRADE', 'CANCEL'
    amount          DECIMAL(19,4) NOT NULL,
    currency        VARCHAR(3) NOT NULL
);
```

- Implementar ETL que popula o star schema a partir do OLTP
- Criar Materialized View para dashboard:

```sql
CREATE MATERIALIZED VIEW mv_monthly_revenue AS
SELECT
    dt.year, dt.month, dt.month_name,
    dp.plan_name,
    COUNT(*) AS subscription_count,
    SUM(fs.amount) AS revenue
FROM fact_subscriptions fs
JOIN dim_time dt ON dt.id = fs.time_id
JOIN dim_plan dp ON dp.id = fs.plan_id
WHERE fs.event_type IN ('NEW', 'RENEWAL')
GROUP BY dt.year, dt.month, dt.month_name, dp.plan_name
WITH DATA;

-- Refresh sem bloquear reads
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_monthly_revenue;
```

**Critérios de aceite:**

- [ ] Star Schema com 3 dimensões e 1 fato
- [ ] ETL script funcional (OLTP → OLAP)
- [ ] Materialized View para dashboard com refresh concorrente
- [ ] Comparar: mesma query no OLTP (3NF) vs OLAP (Star Schema) — performance
- [ ] Diagrama Star Schema em Mermaid

---

### Desafio 3.5 — Monitoramento de Produção

**Contexto:** Sem monitoramento, problemas em produção são descobertos pelo usuário.
Implemente observabilidade do PostgreSQL.

**Requisitos:**

- Implementar queries de monitoramento:

```sql
-- 1. Top 10 queries mais lentas (requer pg_stat_statements)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT query, calls, mean_exec_time, total_exec_time, rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 10;

-- 2. Índices não utilizados (desperdiçando espaço e write performance)
SELECT schemaname, tablename, indexname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexname NOT LIKE 'pg_%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- 3. Tabelas precisando de VACUUM
SELECT relname, n_dead_tup, n_live_tup,
       ROUND(n_dead_tup::NUMERIC / NULLIF(n_live_tup, 0) * 100, 2) AS dead_pct,
       last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- 4. Conexões ativas por estado
SELECT state, COUNT(*) FROM pg_stat_activity GROUP BY state;

-- 5. Queries bloqueadas (locks)
SELECT blocked.pid, blocked.query AS blocked_query,
       blocking.pid AS blocking_pid, blocking.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_stat_activity blocked ON blocked.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.relation = blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_stat_activity blocking ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- 6. Cache hit ratio (deve ser > 99%)
SELECT
    SUM(heap_blks_hit) AS hits,
    SUM(heap_blks_read) AS reads,
    ROUND(SUM(heap_blks_hit)::NUMERIC / NULLIF(SUM(heap_blks_hit + heap_blks_read), 0) * 100, 2) AS hit_ratio
FROM pg_statio_user_tables;
```

- Criar endpoint `/health/db` que retorna métricas:

```json
{
  "connections": { "active": 12, "idle": 4, "max": 100 },
  "cache_hit_ratio": 99.7,
  "replication_lag_ms": 150,
  "slow_queries_count": 3,
  "dead_tuples_pct": 2.1,
  "disk_usage_pct": 45
}
```

- Definir alertas (thresholds):

| Métrica | Warning | Critical |
|---------|---------|----------|
| Connections | > 60% max | > 80% max |
| Cache hit ratio | < 99% | < 95% |
| Replication lag | > 5s | > 30s |
| Slow queries (p99) | > 500ms | > 2s |
| Dead tuples | > 10% | > 30% |
| Disk usage | > 70% | > 85% |

**Critérios de aceite:**

- [ ] 6 queries de monitoramento funcional
- [ ] Endpoint `/health/db` implementado em Java e Go
- [ ] Tabela de alertas com thresholds justificados
- [ ] `pg_stat_statements` habilitado e funcionando
- [ ] Demonstração: identificar índice não utilizado e removê-lo
- [ ] Demonstração: identificar query lenta e otimizá-la

---

### Desafio 3.6 — SQL Anti-patterns de Operações

**Contexto:** Erros operacionais comuns em produção. Identifique e corrija.

**Requisitos:**

- Analisar e corrigir 5 anti-patterns operacionais:

| # | Anti-pattern | Problema | Correção |
|---|-------------|----------|----------|
| 1 | **God Table** | Tabela `events` com 100+ colunas | Decomposição em tabelas especializadas |
| 2 | **No connection pool** | Conexão por request = 50ms overhead | HikariCP/PgBouncer |
| 3 | **UUID v4 como PK clustered** | Fragmentação de B-Tree, writes aleatórios | UUID v7 (time-ordered) ou BIGINT |
| 4 | **Storing blobs in DB** | Thumbnails de conteúdo no PostgreSQL | S3 + referência no banco |
| 5 | **Schema sem constraints** | Dados inválidos entram sem controle | CHECK, NOT NULL, FK explícitas |

- Para cada anti-pattern:
  - Query ou código demonstrando o problema
  - Solução implementada
  - Benchmark quando aplicável (UUID v4 vs v7 insert performance)

**Critérios de aceite:**

- [ ] 5 anti-patterns documentados com exemplo de código
- [ ] UUID v7 implementado e comparado com v4 (insert performance)
- [ ] Solução para blob storage: S3 + referência
- [ ] Constraints adicionados em tabela sem validação
- [ ] Documento: `decisions/03-operational-anti-patterns.md`

---

### Desafio 3.7 — Disaster Recovery Plan

**Contexto:** O PostgreSQL primary da StreamX fica indisponível. Qual é o plano?

**Requisitos:**

- Definir RPO e RTO para cada workload:

| Workload | RPO | RTO | Estratégia |
|----------|-----|-----|------------|
| Billing (subscriptions, payments) | 0 (zero data loss) | < 5 min | Multi-AZ sync replication |
| User profiles | < 1 hora | < 15 min | Automated backups + failover |
| Watch events | < 24 horas | < 1 hora | Daily snapshots |

- Configurar backup automatizado:
  - `pg_dump` diário com retenção de 30 dias
  - WAL archiving para point-in-time recovery
- Implementar script de failover:
  - Promover réplica a primary
  - Atualizar connection string na aplicação
  - Validar integridade dos dados
- **Testar restore**: restaurar backup em novo container e validar dados

```bash
# Backup
pg_dump -h primary-host -U app -d streamx -F c -f /backups/streamx_$(date +%Y%m%d).dump

# Restore em novo container
docker run --name streamx-restore -e POSTGRES_DB=streamx_restore -d postgres:17
pg_restore -h localhost -U postgres -d streamx_restore /backups/streamx_20250115.dump

# Validar
psql -h localhost -d streamx_restore -c "SELECT COUNT(*) FROM users;"
psql -h localhost -d streamx_restore -c "SELECT SUM(amount) FROM payments WHERE status = 'COMPLETED';"
```

**Critérios de aceite:**

- [ ] RPO/RTO definidos para 3 workloads
- [ ] Script de backup funcional (pg_dump ou snapshot)
- [ ] Restore testado em container separado com validação
- [ ] Script de failover: promover réplica
- [ ] Runbook documentado: passo a passo para DR
- [ ] Documento: `decisions/03-disaster-recovery-plan.md`
