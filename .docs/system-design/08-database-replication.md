# 8. Database Replication

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial — base de alta disponibilidade e escalabilidade de leitura  
> **Complexidade:** Alta (consistência distribuída é inerentemente complexa)

---

## Definição

**Database Replication** é o processo de copiar e manter dados idênticos em múltiplos servidores de banco de dados. O objetivo é aumentar **disponibilidade**, **durabilidade** e **performance de leitura**.

---

## Por Que é Importante?

| Sem Replicação | Com Replicação |
|---------------|---------------|
| Single point of failure | Se primary falha, replica assume |
| Todo read e write no mesmo server | Reads distribuídos entre replicas |
| Backup = downtime ou lag | Replica é backup em tempo real |
| Uma região = alta latência global | Replicas cross-region = baixa latência |

---

## Topologias de Replicação

### 1. Single-Leader (Master-Slave)

A topologia mais comum. Um **leader** (primary/master) recebe todas as escritas e replica para **followers** (replicas/slaves).

```
                    WRITES
                      │
                 ┌────▼────┐
                 │ Primary │  (Leader)
                 │ (R/W)   │
                 └────┬────┘
                      │ Replication Stream
           ┌──────────┼──────────┐
           ▼          ▼          ▼
     ┌──────────┐ ┌──────────┐ ┌──────────┐
     │ Replica 1│ │ Replica 2│ │ Replica 3│
     │ (Read)   │ │ (Read)   │ │ (Read)   │
     └──────────┘ └──────────┘ └──────────┘
           ▲          ▲          ▲
           └──────────┼──────────┘
                   READS
```

| Prós | Contras |
|------|---------|
| Simples de entender e operar | Leader é bottleneck de escrita |
| Sem conflitos de escrita | Failover pode causar perda de dados |
| Read scaling horizontal | Replication lag = stale reads |

### 2. Multi-Leader (Master-Master)

Múltiplos nodes aceitam escritas. Usado em **multi-datacenter** setups.

```
Datacenter A                    Datacenter B
┌──────────┐    Replication    ┌──────────┐
│ Leader A │◀──────────────────▶│ Leader B │
│ (R/W)    │                   │ (R/W)    │
└────┬─────┘                   └────┬─────┘
     │                              │
  ┌──▼───┐                      ┌──▼───┐
  │Rep A1│                      │Rep B1│
  └──────┘                      └──────┘
```

| Prós | Contras |
|------|---------|
| Writes em múltiplas regiões | **Conflitos de escrita** (mesmo dado, DCs diferentes) |
| Tolerante a falha de DC inteiro | Resolução de conflitos é complexa |
| Menor latência de escrita global | Eventual consistency |

**Resolução de Conflitos:**

| Estratégia | Descrição |
|------------|-----------|
| **Last Writer Wins (LWW)** | Timestamp mais recente ganha. Simples mas pode perder dados |
| **Custom merge** | Lógica de negócio decide (ex: merge de campos) |
| **CRDTs** | Estruturas de dados que convergem automaticamente |
| **Manual resolution** | Armazena conflito, humano decide |

### 3. Leaderless (Peer-to-Peer)

Todos os nodes aceitam reads e writes. Usado por **DynamoDB, Cassandra, Riak**.

```
          Write request
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│ Node 1 │ │ Node 2 │ │ Node 3 │
│ (R/W)  │ │ (R/W)  │ │ (R/W)  │
└────────┘ └────────┘ └────────┘

Write para N nodes, requer W acknowledgements
Read de N nodes, requer R acknowledgements

Quórum: W + R > N  (garante overlap entre writes e reads)
Exemplo: N=3, W=2, R=2 → sempre lê pelo menos uma cópia atualizada
```

| Prós | Contras |
|------|---------|
| Alta disponibilidade | Eventual consistency |
| Sem single point of failure | Conflitos de escrita possíveis |
| Tolerante a falhas parciais | Mais complexo de operar |

### Comparativo de Topologias

```
                    Single-Leader    Multi-Leader     Leaderless
Write nodes:        1                N (N DCs)        Todos
Conflitos:          Nenhum           Sim              Sim
Consistência:       Forte possível   Eventual         Eventual
Disponibilidade:    Média            Alta             Muito alta
Complexidade:       Baixa            Alta             Alta
Uso:                PostgreSQL,      MySQL Group,     Cassandra,
                    MySQL, MongoDB   CockroachDB      DynamoDB, Riak
```

---

## Modos de Replicação

### Synchronous Replication

```
Client ──▶ Primary ──write──▶ Replica ──ACK──▶ Primary ──ACK──▶ Client

Timeline:
Client ────── [write request] ──────▶ Primary
Primary ───── [replicate] ──────────▶ Replica
Replica ───── [write + ACK] ────────▶ Primary
Primary ───── [ACK to client] ──────▶ Client

Total: ~5-20ms (depende da rede entre primary e replica)
```

| Prós | Contras |
|------|---------|
| Zero data loss (RPO=0) | Latência de escrita mais alta |
| Strong consistency | Se replica falha, writes bloqueiam |
| Replica sempre up-to-date | |

### Asynchronous Replication

```
Client ──▶ Primary ──ACK──▶ Client
                    └──async──▶ Replica (eventualmente)

Timeline:
Client ────── [write request] ──────▶ Primary
Primary ───── [ACK to client] ──────▶ Client  (imediato!)
Primary ───── [replicate async] ────▶ Replica  (depois, background)

Total write latency: ~1-5ms (sem esperar replica)
Replication lag: 10ms - vários segundos
```

| Prós | Contras |
|------|---------|
| Baixa latência de escrita | Pode perder dados se primary falha |
| Primary não depende de replica | Replication lag → stale reads |
| Mais resiliente a falha de rede | |

### Semi-Synchronous

```
Primary replica para N replicas:
- 1 replica: synchronous (garante pelo menos 1 cópia)
- N-1 replicas: asynchronous

Se a sync replica falha, uma async é promovida a sync.
```

**Usado por:** MySQL (semi-sync replication), PostgreSQL (synchronous_standby_names)

---

## Replication Lag e Seus Problemas

### O Problema

```
T0: Client escreve no Primary: user.name = "João"
T1: Primary confirma ao Client
T2: Client lê da Replica: user.name = "Maria" ← STALE! Lag de 500ms

Cenário: E-commerce
1. Usuário atualiza endereço de entrega
2. Atualização vai para Primary
3. Usuário clica "Confirmar Pedido" → lê da Replica → endereço antigo!
```

### Soluções

| Solução | Como Funciona |
|---------|---------------|
| **Read-your-own-writes** | Após write, lê do Primary por X segundos |
| **Monotonic reads** | Sticky session → mesmo client lê sempre da mesma replica |
| **Consistent prefix reads** | Garante ordem causal de eventos relacionados |
| **Synchronous read** | Força read do Primary para dados críticos |

```
Read-your-own-writes:

1. Client writes → Primary (T0)
2. Client marca: "last_write_timestamp = T0"
3. Client reads:
   - Se now() - last_write < 10s → lê do Primary
   - Se now() - last_write > 10s → lê da Replica (já convergiu)
```

---

## Failover

### Automatic Failover

```
Normal operation:
  Client → Primary (R/W)
  Primary → Replica 1, Replica 2

Primary fails:
  1. Health check detecta Primary down (timeout ~10-30s)
  2. Election: Replica com dados mais recentes é promovida
  3. DNS/proxy atualizado para apontar para novo Primary
  4. Antiga Primary, quando voltar, vira Replica

Timeline:
  T0:    Primary falha
  T0+10s: Heartbeat timeout
  T0+15s: Election inicia
  T0+20s: Nova Replica promovida
  T0+25s: Clients redirecionados
  
  Downtime total: ~15-30 segundos
```

### Problemas de Failover

| Problema | Descrição | Mitigação |
|----------|-----------|-----------|
| **Data loss** | Async replica não tem últimos writes | Semi-sync, RPO acceptance |
| **Split brain** | Dois nodes acham que são Primary | Fencing (STONITH), quorum |
| **Stale reads during failover** | Clients leem de replica desatualizada | Connection drain + warmup |
| **Sequence gaps** | Auto-increment IDs podem ter gaps | UUID, ULID |

### Split Brain

```
Cenário perigoso:

           Network Partition
                  ║
  ┌───────────┐   ║   ┌───────────┐
  │ Primary   │   ║   │ Replica   │
  │ (thinks   │   ║   │ (promoted │
  │  it's     │   ║   │  to       │
  │  primary) │   ║   │  primary) │
  └───────────┘   ║   └───────────┘
  Aceita writes   ║   Aceita writes  ← CONFLITO!

Solução: Fencing
  - STONITH (Shoot The Other Node In The Head)
  - Quorum: precisa de maioria para ser Primary
  - Shared disk / distributed lock
```

---

## WAL (Write-Ahead Log) Replication

### Como Funciona (PostgreSQL)

```
1. Client: INSERT INTO orders (...)
2. Primary escreve no WAL (append-only log no disco)
3. Primary aplica no data files
4. WAL segments são enviados para Replicas
5. Replicas aplicam WAL no seus data files

WAL Segment:
┌────────────────────────────────────────────────┐
│ LSN: 0/3000028 | INSERT | Table: orders       │
│ Tuple: (id=123, user_id=456, total=99.90)      │
├────────────────────────────────────────────────┤
│ LSN: 0/3000100 | UPDATE | Table: users         │
│ Set: login_count = login_count + 1              │
├────────────────────────────────────────────────┤
│ LSN: 0/3000180 | DELETE | Table: sessions       │
│ Where: id = 789                                 │
└────────────────────────────────────────────────┘
```

### Tipos de Replicação por WAL

| Tipo | Descrição | Nível |
|------|-----------|-------|
| **Physical (Streaming)** | Replica WAL byte-a-byte | Bloco de disco |
| **Logical** | Replica operações SQL (INSERT, UPDATE, DELETE) | Linha/tabela |

```
Physical:
  + Simples, replica tudo
  + Mesma versão de DB
  - Não pode replicar seletivamente

Logical:
  + Replica tabelas específicas
  + Cross-version replication
  + Pode transformar dados durante replica
  - Mais complexo, DDL não replicado automaticamente
```

---

## Configuração Prática

### PostgreSQL — Streaming Replication

```ini
# postgresql.conf (Primary)
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB
synchronous_standby_names = 'replica1'  # semi-sync

# pg_hba.conf (Primary)
host replication replicator 10.0.1.0/24 scram-sha-256
```

```ini
# postgresql.conf (Replica)
primary_conninfo = 'host=10.0.1.100 port=5432 user=replicator password=xxx'
hot_standby = on  # permite reads na replica
```

```bash
# Inicializar replica a partir do Primary
pg_basebackup -h 10.0.1.100 -D /var/lib/postgresql/data -U replicator -v -P -R
```

### MySQL — GTID Replication

```sql
-- Primary
SET GLOBAL gtid_mode = ON;
SET GLOBAL enforce_gtid_consistency = ON;
SET GLOBAL server_id = 1;

-- Replica
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = '10.0.1.100',
    SOURCE_USER = 'replicator',
    SOURCE_PASSWORD = 'xxx',
    SOURCE_AUTO_POSITION = 1;  -- GTID-based
START REPLICA;
```

### Monitoramento de Lag

```sql
-- PostgreSQL: lag em bytes e tempo
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS byte_lag,
    replay_lag
FROM pg_stat_replication;

-- MySQL: lag em segundos
SHOW REPLICA STATUS\G
-- Seconds_Behind_Source: 0  ← ideal
```

---

## Read Replicas para Scaling

```
                 ┌──────────────┐
                 │   Primary    │ ← Writes only
                 │              │
                 └──────┬───────┘
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
      ┌──────────┐ ┌──────────┐ ┌──────────┐
      │ Replica 1│ │ Replica 2│ │ Replica 3│
      │ (reads)  │ │ (reads)  │ │ (reads)  │
      └──────────┘ └──────────┘ └──────────┘
      
App connection pool:
  - Write pool:  Primary (1 node)
  - Read pool:   Replica 1, 2, 3 (load balanced)
```

```java
// Spring Boot — Read/Write splitting
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary
    public DataSource routingDataSource() {
        Map<Object, Object> targetDataSources = Map.of(
            "primary", primaryDataSource(),
            "replica", replicaDataSource()
        );
        
        RoutingDataSource routing = new RoutingDataSource();
        routing.setTargetDataSources(targetDataSources);
        routing.setDefaultTargetDataSource(primaryDataSource());
        return routing;
    }
}

// Transaction annotation switches datasource
@Transactional(readOnly = true)  // → Replica
public User findById(Long id) { ... }

@Transactional  // → Primary
public User save(User user) { ... }
```

---

## Uso em Big Techs

### Meta (Facebook) — MySQL Replication
- Milhares de MySQL instances
- Primary + muitas replicas por shard
- Async replication (aceita eventual consistency para reads)
- Custom replication monitoring (MHA → orchestrator → custom)

### Netflix — Cassandra (Leaderless)
- Replicação multi-region com Cassandra
- Quorum reads/writes para consistência configurável
- `QUORUM`, `LOCAL_QUORUM`, `EACH_QUORUM` por operação

### Google — Spanner (Synchronous + Paxos)
- **Replicação síncrona global** usando Paxos consensus
- TrueTime: relógios atômicos para ordering global
- Forte consistência com latência aceitável (~10-15ms per write)

### Amazon — Aurora
- Storage-level replication (não DB-level)
- 6 cópias dos dados em 3 AZs
- Write quorum: 4/6, Read quorum: 3/6
- Replica lag < 20ms (storage-based)

---

## Perguntas Comuns em Entrevistas

1. **Sync vs Async replication?** → Sync = zero loss, high latency; Async = fast writes, possible data loss
2. **O que é replication lag?** → Delay entre write no primary e aplicação na replica. Causa stale reads
3. **Como lidar com lag?** → Read-your-own-writes, monotonic reads, lê do primary para dados críticos
4. **O que é split brain?** → Dois nodes acham que são primary. Resolução: quorum, fencing (STONITH)
5. **Single-leader vs Multi-leader?** → Single = simples, sem conflitos; Multi = multi-region writes, conflitos

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Sync vs Async** | Sync (zero data loss) | Async (melhor performance) |
| **Single vs Multi-leader** | Single (simples, consistente) | Multi-leader (multi-region) |
| **Read scaling** | Mais replicas (horizontal) | Caching layer (Redis) |
| **Failover** | Manual (controle) | Automático (speed, risco) |
| **Consistency** | Strong (lê do primary) | Eventual (lê de replicas) |

---

## Referências

- [PostgreSQL Replication](https://www.postgresql.org/docs/current/high-availability.html)
- [MySQL Replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- [AWS Aurora Docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/)
- Designing Data-Intensive Applications — Martin Kleppmann, Chapter 5
- Database Internals — Alex Petrov, Part II
