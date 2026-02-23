# 9. Database Sharding

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Avançado — necessário quando o banco não escala verticalmente mais  
> **Complexidade:** Alta (impacta toda a aplicação)

---

## Definição

**Sharding (Particionamento Horizontal)** é a técnica de dividir uma tabela grande em múltiplos bancos de dados menores (shards), cada um contendo um **subconjunto das linhas**. Cada shard é um banco de dados independente.

> É o "último recurso" de escalabilidade — adiciona complexidade significativa mas permite escalar escritas horizontalmente.

---

## Por Que é Importante?

| Sem Sharding | Com Sharding |
|-------------|-------------|
| 1 banco com 10TB | 10 bancos de 1TB cada |
| 1 server: CPU/RAM limitados | 10 servers: 10x CPU/RAM |
| Write bottleneck (1 master) | Writes distribuídos em N shards |
| Backup de 10TB = horas | Backup de 1TB = minutos (paralelo) |
| Índices enormes em memória | Índices menores, cabem no RAM |

**Quando shardar:**
- Dados > 1TB no banco
- Write throughput > capacidade de 1 server
- Latência de query degradando com crescimento
- Vertical scaling (mais CPU/RAM) chegou ao limite

---

## Conceito Visual

```
Tabela orders (antes):
┌──────────────────────────────────────┐
│ orders (100M rows, 10TB)             │
│ ┌────┬─────────┬────────┬──────────┐ │
│ │ id │ user_id │ amount │ created  │ │
│ ├────┼─────────┼────────┼──────────┤ │
│ │ 1  │ 101     │ 50.00  │ 2024-01  │ │
│ │ 2  │ 205     │ 30.00  │ 2024-01  │ │
│ │... │ ...     │ ...    │ ...      │ │
│ │100M│ 999     │ 75.00  │ 2024-12  │ │
│ └────┴─────────┴────────┴──────────┘ │
└──────────────────────────────────────┘

Tabela orders (depois — sharded por user_id):
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│  Shard 0       │  │  Shard 1       │  │  Shard 2       │
│  (user 0-333K) │  │  (user 333K-   │  │  (user 666K-   │
│                │  │   666K)        │  │   1M)          │
│ ┌────┬────────┐│  │ ┌────┬────────┐│  │ ┌────┬────────┐│
│ │ id │user_id ││  │ │ id │user_id ││  │ │ id │user_id ││
│ ├────┼────────┤│  │ ├────┼────────┤│  │ ├────┼────────┤│
│ │ 1  │ 101    ││  │ │ 2  │ 500100 ││  │ │ 5  │ 800001 ││
│ │ 3  │ 205    ││  │ │ 4  │ 400200 ││  │ │ 6  │ 999000 ││
│ │... │ ...    ││  │ │... │ ...    ││  │ │... │ ...    ││
│ └────┴────────┘│  │ └────┴────────┘│  │ └────┴────────┘│
│  DB: pg-shard0 │  │  DB: pg-shard1 │  │  DB: pg-shard2 │
│  Host: db-0    │  │  Host: db-1    │  │  Host: db-2    │
└────────────────┘  └────────────────┘  └────────────────┘
```

---

## Shard Key (Chave de Particionamento)

A **shard key** determina em qual shard cada row vai. É a decisão **mais crítica** do sharding.

### Critérios para Escolha

```
✓ Alta cardinalidade (muitos valores distintos)
✓ Distribuição uniforme (evita hotspots)
✓ Alinhada com padrão de query (queries tocam 1 shard)
✓ Imutável (não muda após inserção)
✓ Presente em queries mais frequentes
```

### Exemplos de Shard Keys

| Tabela | Boa Shard Key | Ruim Shard Key |
|--------|--------------|----------------|
| **orders** | user_id (queries por user) | status (poucos valores → hotspot) |
| **messages** | conversation_id | created_at (shard recente recebe todo write) |
| **products** | product_id | category (poucas categorias) |
| **logs** | timestamp (range) | level (3 valores: INFO, WARN, ERROR) |

---

## Estratégias de Sharding

### 1. Hash-Based Sharding

```
shard_number = hash(shard_key) % num_shards

Exemplo: 4 shards, shard_key = user_id
  hash(user_101) % 4 = 1  → Shard 1
  hash(user_205) % 4 = 3  → Shard 3
  hash(user_500) % 4 = 0  → Shard 0
  hash(user_999) % 4 = 2  → Shard 2
```

```
          user_id
             │
        hash(user_id) % 4
             │
    ┌────────┼────────┬────────┐
    ▼        ▼        ▼        ▼
 Shard 0  Shard 1  Shard 2  Shard 3
```

| Prós | Contras |
|------|---------|
| Distribuição uniforme | Range queries impossíveis |
| Simples de implementar | Resharding requer rehash de tudo |
| Sem hotspots | Adicionar shard = migração massiva |

### 2. Range-Based Sharding

```
Shard por faixa de valores:
  Shard 0: user_id 1 - 1.000.000
  Shard 1: user_id 1.000.001 - 2.000.000
  Shard 2: user_id 2.000.001 - 3.000.000

Ou por data:
  Shard 0: orders de Jan-Mar 2024
  Shard 1: orders de Apr-Jun 2024
  Shard 2: orders de Jul-Sep 2024
```

| Prós | Contras |
|------|---------|
| Range queries eficientes | Hotspot no shard mais recente |
| Fácil de entender | Distribuição pode ser desigual |
| Simples de adicionar shards | |

### 3. Directory-Based Sharding

Um **lookup service** mantém mapeamento key → shard.

```
┌──────────────────────────────┐
│    Directory Service         │
│  ┌──────────┬──────────────┐ │
│  │ user_id  │ shard        │ │
│  ├──────────┼──────────────┤ │
│  │ 101      │ shard-0      │ │
│  │ 205      │ shard-2      │ │
│  │ 500      │ shard-1      │ │     ← Flexível, qualquer mapeamento
│  │ 999      │ shard-0      │ │
│  └──────────┴──────────────┘ │
└──────────────────────────────┘
```

| Prós | Contras |
|------|---------|
| Máxima flexibilidade | Directory = single point of failure |
| Fácil resharding | Overhead de lookup em toda query |
| Qualquer distribuição | Precisa ser cacheado (Redis) |

### 4. Consistent Hashing

Resolve o problema de **rehash total** quando shards são adicionados/removidos.

```
Hash Ring (0 - 2^32):

            0
          ╱   ╲
       S0       S1        S = Shard (node no ring)
      ╱           ╲       K = Key  
     ╱             ╲
   K1    K4     K2
     ╲             ╱
      ╲           ╱
       S3       S2
          ╲   ╱
           K3

Cada key vai para o próximo Shard no sentido horário:
  K1 → S0
  K2 → S2
  K3 → S3
  K4 → S1

Adicionar S4 entre S0 e S1:
  - Apenas K4 se move (de S1 para S4)
  - K1, K2, K3 ficam no mesmo lugar
  - Mínima redistribuição!
```

| Prós | Contras |
|------|---------|
| Adicionar/remover shard = move poucos keys | Distribuição pode ser desigual |
| Usado por todos os sistemas distribuídos | Precisa de virtual nodes para uniformidade |

### 5. Geographic Sharding

```
Shard por região geográfica:
  Shard BR: dados de usuários brasileiros (São Paulo)
  Shard US: dados de usuários americanos (Virginia)
  Shard EU: dados de usuários europeus (Frankfurt)

User request:
  user.country = "BR" → Shard BR (baixa latência no Brasil)
```

| Prós | Contras |
|------|---------|
| Data locality (latência mínima) | Cross-region queries são lentas |
| Compliance (LGPD, GDPR) | Distribuição desigual entre regiões |

### Comparativo

```
              Hash      Range     Directory    Consistent Hash    Geo
Distribuição: ✓ Uniforme ✗ Pode    ✓ Flexível   ✓ Uniforme        ✗ Desigual
Range query:  ✗ Não      ✓ Sim     Depende      ✗ Não             Depende
Resharding:   ✗ Difícil  ✓ Fácil   ✓ Fácil      ✓ Fácil           ✗ Difícil
Hotspot:      ✓ Não      ✗ Sim     ✓ Não        ✓ Não             ✗ Possível
Complexidade: Baixa      Baixa     Média        Média             Alta
```

---

## Desafios do Sharding

### 1. Cross-Shard Queries (Scatter-Gather)

```
SELECT COUNT(*) FROM orders WHERE amount > 100;

Sem sharding: 1 query no banco

Com sharding (3 shards):
  → Query Shard 0: SELECT COUNT(*) FROM orders WHERE amount > 100;  → 50000
  → Query Shard 1: SELECT COUNT(*) FROM orders WHERE amount > 100;  → 48000
  → Query Shard 2: SELECT COUNT(*) FROM orders WHERE amount > 100;  → 52000
  ← Application combina: 150000

Problemas:
  - Latência = max(latência de cada shard)
  - Se 1 shard falha, query inteira falha
  - ORDER BY + LIMIT precisa merge sort
  - JOINs cross-shard são extremamente caros
```

### 2. Cross-Shard JOINs

```sql
-- Join entre tabelas em shards diferentes: IMPOSSÍVEL no DB
SELECT u.name, o.amount
FROM users u JOIN orders o ON u.id = o.user_id
WHERE o.created_at > '2024-01-01';

-- Soluções:
-- 1. Co-locate: mesma shard key (user_id) para users e orders
-- 2. Application-level join: busca users, depois busca orders
-- 3. Denormalizar: duplicar user_name na tabela orders
```

### 3. Cross-Shard Transactions

```
PROBLEMA: User 101 (Shard 0) transfere dinheiro para User 500 (Shard 1)

BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE user_id = 101;  -- Shard 0
  UPDATE accounts SET balance = balance + 100 WHERE user_id = 500;  -- Shard 1
COMMIT;

-- Não pode ter transaction ACID cross-shard nativamente!

Soluções:
  1. Two-Phase Commit (2PC): coordenador garante atomicidade (lento)
  2. Saga Pattern: sequência de transações locais + compensação
  3. Evitar: design para que transações fiquem no mesmo shard
```

### 4. Resharding (Rebalancing)

```
Cenário: 4 shards → 8 shards (crescimento)

Hash-based: hash(key) % 4 → hash(key) % 8
  - Quase TODAS as keys mudam de shard!
  - Precisa migrar dados massivamente
  - Downtime ou complexidade enorme

Consistent Hashing: adiciona 4 nodes ao ring
  - Apenas ~50% das keys se movem
  - Muito melhor!
  
Melhor ainda: Virtual sharding
  - Cria 256 virtual shards desde o início
  - 4 servers hospedam 64 virtual shards cada
  - 8 servers: move virtual shards (sem rehash)
```

### 5. Hot Shards

```
Problema: Celebridade com 50M followers posta tweet
  - Shard contendo esse user recebe 10x mais reads
  - Outros shards estão idle

Soluções:
  - Detect + split: monitora e divide shard quente
  - Key-level splitting: replica hot key em múltiplos shards
  - Caching agressivo: Redis na frente do hot shard
  - Celebrity/VIP list: trata hot keys de forma especial
```

---

## Arquitetura de Sharding

### Application-Level Sharding

```
┌──────────────┐
│  Application │
│              │
│  Shard Logic │ ← App decide qual shard
│  hash(key)%N │
│              │
└──┬───┬───┬───┘
   │   │   │
   ▼   ▼   ▼
  S0  S1  S2
```

### Proxy-Based Sharding

```
┌──────────────┐
│  Application │
└──────┬───────┘
       │
┌──────▼───────┐
│ Shard Proxy  │ ← Proxy decide qual shard
│ (Vitess,     │
│  ProxySQL)   │
└──┬───┬───┬───┘
   │   │   │
   ▼   ▼   ▼
  S0  S1  S2
```

### Database-Native Sharding

```
┌──────────────┐
│  Application │
└──────┬───────┘
       │
┌──────▼───────────────────────┐
│     Distributed Database     │
│  (CockroachDB, YugabyteDB,  │
│   TiDB, Spanner)            │
│                              │
│  Auto-sharding + rebalancing │
│  Cross-shard transactions    │
│  SQL interface normal        │
└──────────────────────────────┘
```

---

## Tecnologias

| Tecnologia | Tipo | Sharding |
|------------|------|----------|
| **Vitess** | Proxy (MySQL) | Hash, range, custom. YouTube usa |
| **Citus** | Extension (PostgreSQL) | Hash, reference tables. Distributed PG |
| **ProxySQL** | Proxy (MySQL) | Query routing rules |
| **CockroachDB** | NewSQL | Auto-sharding + rebalancing |
| **TiDB** | NewSQL | Auto-sharding (MySQL compatible) |
| **YugabyteDB** | NewSQL | Auto-sharding (PG compatible) |
| **MongoDB** | Document DB | Hash + range sharding nativo |
| **Cassandra** | Wide-column | Consistent hashing nativo |
| **Spanner** | Google | Auto-sharding + strong consistency |

### Vitess (YouTube/Google)

```yaml
# Vitess VSchema (define sharding)
{
  "sharded": true,
  "vindexes": {
    "user_hash": {
      "type": "hash"
    }
  },
  "tables": {
    "orders": {
      "column_vindexes": [
        {
          "column": "user_id",
          "name": "user_hash"
        }
      ]
    },
    "users": {
      "column_vindexes": [
        {
          "column": "id",
          "name": "user_hash"
        }
      ]
    }
  }
}
```

---

## Uso em Big Techs

### YouTube — Vitess (MySQL Sharding)
- Milhares de MySQL shards gerenciados por Vitess
- Shard key: video_id ou channel_id
- Cross-shard queries via scatter-gather
- Resharding online sem downtime

### Meta (Facebook) — TAO + MySQL
- MySQL shardado por user_id
- TAO cache layer na frente
- Billions of queries/day
- Custom sharding middleware

### Uber — Schemaless → DocStore
- MySQL shardado por city + entity_id
- Schemaless: schema-agnostic storage
- Permite alteração de schema sem migration
- Geo-sharding por cidade

### Pinterest — MySQL + ZooKeeper
- Sharding manual com lookup table no ZooKeeper
- Shard key: user_id para pins, board_id para boards
- Co-location: user's pins no mesmo shard

### Stripe — MongoDB Sharding
- Payment data shardado para compliance e performance
- Sharding + encryption at rest
- Careful shard key selection para transaction locality

---

## Perguntas Comuns em Entrevistas

1. **O que é sharding?** → Dividir tabela em múltiplos bancos. Cada shard tem subconjunto das rows
2. **Quando shardar?** → Quando vertical scaling não é suficiente (>1TB, write bottleneck)
3. **Hash vs Range sharding?** → Hash = distribuição uniforme, sem range query. Range = range query ok, possível hotspot
4. **Maior desafio do sharding?** → Cross-shard queries, JOINs, transactions, resharding
5. **Como escolher shard key?** → Alta cardinalidade, distribuição uniforme, alinhada com queries, imutável

---

## Trade-offs

| Decisão | Opção A | Opção B |
|---------|---------|---------|
| **Shard vs No-shard** | Sharding (escala writes) | Vertical scaling / read replicas (simples) |
| **Hash vs Range** | Hash (uniforme, sem range) | Range (range queries, hotspots) |
| **App-level vs Proxy** | App decides (performance) | Proxy/Vitess (transparente) |
| **Manual vs Auto** | Manual sharding (controle) | NewSQL auto-shard (conveniência) |
| **Few big vs Many small** | Poucos shards grandes (simples) | Muitos shards pequenos (flexível) |

---

## Referências

- [Vitess Documentation](https://vitess.io/docs/)
- [Citus (PostgreSQL Sharding)](https://www.citusdata.com/documentation/)
- [MongoDB Sharding](https://www.mongodb.com/docs/manual/sharding/)
- [Uber Schemaless](https://eng.uber.com/schemaless-part-one-mysql-datastore/)
- Designing Data-Intensive Applications — Martin Kleppmann, Chapter 6
- Database Internals — Alex Petrov
