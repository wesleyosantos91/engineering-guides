# 32. Data Partitioning Strategies

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial para qualquer entrevista de System Design  
> **Complexidade:** Média-Alta

---

## Definição

**Data Partitioning** é a técnica de dividir um grande dataset em partições menores distribuídas em múltiplos nós, para melhorar **performance**, **scalability** e **manageability**. É imprescindível quando um único servidor não pode armazenar ou processar todos os dados.

---

## Por Que é Importante?

- **Escala horizontal** — distribui dados entre N servidores
- **Performance** — queries acessam menos dados (partition pruning)
- **Manageability** — backup, manutenção e rebalancing por partição
- **Pergunta central em system design** — "como você shardeia esses dados?"
- **Usado em every Big Tech** — sem exceção

---

## Tipos de Partitioning

### Visão Geral

```
┌─────────────────────────────────────────────────────────┐
│  Horizontal (Sharding)      Vertical        Functional  │
│                                                         │
│  ┌────┐ ┌────┐ ┌────┐    ┌──┬──┬──┐      ┌──────────┐│
│  │Row1│ │Row4│ │Row7│    │C1│C2│C3│      │ Users DB ││
│  │Row2│ │Row5│ │Row8│    │C1│C2│  │      │ Orders DB││
│  │Row3│ │Row6│ │Row9│    │  │  │C3│      │ Payment DB│
│  └────┘ └────┘ └────┘    └──┴──┴──┘      └──────────┘│
│  Shard1  Shard2  Shard3   T1  T2  T3     Serviço 1,2,3│
│                                                         │
│  Divide LINHAS           Divide COLUNAS  Divide DOMÍNIO│
└─────────────────────────────────────────────────────────┘
```

### 1. Horizontal Partitioning (Sharding)

```
Divide LINHAS da tabela entre múltiplos servidores.
Cada shard contém um subconjunto de rows com o MESMO schema.

  Users table (100M rows):
  
  Shard 1 (Server A):        Shard 2 (Server B):       Shard 3 (Server C):
  ┌────┬──────┬────────┐    ┌────┬──────┬────────┐    ┌────┬──────┬────────┐
  │ id │ name │ region │    │ id │ name │ region │    │ id │ name │ region │
  ├────┼──────┼────────┤    ├────┼──────┼────────┤    ├────┼──────┼────────┤
  │  1 │ Ana  │ BR     │    │ 34M│ John │ US     │    │ 67M│ Yuki │ JP     │
  │  2 │ Carlos│ BR    │    │ 34M│ Jane │ US     │    │ 67M│ Sakura│ JP    │
  │... │ ...  │ ...    │    │... │ ...  │ ...    │    │... │ ...  │ ...    │
  │33M │ Maria│ PT     │    │66M │ Bob  │ UK     │    │100M│ Wei  │ CN     │
  └────┴──────┴────────┘    └────┴──────┴────────┘    └────┴──────┴────────┘
  
  ✅ Cada shard é independente e escalável
  ✅ Queries filtradas por shard key → rápidas
  ❌ Cross-shard queries são caras
  ❌ Rebalancing é complexo
```

### 2. Vertical Partitioning

```
Divide COLUNAS de uma tabela em tabelas menores (geralmente por frequência de acesso).

  Tabela Original (wide):
  ┌────┬──────┬────────┬──────────┬───────┬──────────────┐
  │ id │ name │ email  │ avatar   │ bio   │ preferences  │
  │    │      │        │ (5MB)    │ (1KB) │ (JSON, 10KB) │
  └────┴──────┴────────┴──────────┴───────┴──────────────┘

  Vertical Split:
  
  users_core (accessed frequently):     users_profile (accessed rarely):
  ┌────┬──────┬────────┐               ┌────┬──────────┬───────┬──────────────┐
  │ id │ name │ email  │               │ id │ avatar   │ bio   │ preferences  │
  └────┴──────┴────────┘               └────┴──────────┴───────┴──────────────┘
  ~100 bytes/row                        ~5MB/row
  
  ✅ Core table cabe em memória (cacheable)
  ✅ Menos I/O per query
  ❌ JOINs entre tabelas
  ❌ Nem sempre é claro como dividir
```

### 3. Functional Partitioning (by Domain)

```
Cada domínio / bounded context tem seu próprio banco de dados.

  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │ User Service │   │ Order Service│   │Payment Service│
  │              │   │              │   │              │
  │  ┌────────┐  │   │  ┌────────┐  │   │  ┌────────┐  │
  │  │Users DB│  │   │  │Orders DB│ │   │  │Pay DB  │  │
  │  │(Postgres)│ │   │  │(Cassandra)│  │  │(Postgres)│ │
  │  └────────┘  │   │  └────────┘  │   │  └────────┘  │
  └──────────────┘   └──────────────┘   └──────────────┘
  
  ✅ Cada serviço escala e evolui independentemente
  ✅ DB ideal por domínio (SQL para users, Cassandra para orders)
  ❌ Join cross-service impossível (precisa de API)
  ❌ Distributed transactions (Saga pattern)
```

---

## Estratégias de Sharding (Partition Keys)

### Range-Based Partitioning

```
Divide dados por FAIXAS de valores:

  Shard 1: user_id [1 — 1,000,000]
  Shard 2: user_id [1,000,001 — 2,000,000]
  Shard 3: user_id [2,000,001 — 3,000,000]

  Ou por data:
  Shard Jan: orders de 2024-01-01 a 2024-01-31
  Shard Feb: orders de 2024-02-01 a 2024-02-28

  ✅ Range queries eficientes (ex: "orders de janeiro")
  ✅ Fácil de entender e implementar
  ❌ HOTSPOT: se user_ids novos vão todos para o último shard
  ❌ Distribuição desigual se ranges não forem bem escolhidos
```

### Hash-Based Partitioning

```
Aplica hash function no shard key para distribuir uniformemente:

  shard_id = hash(user_id) % num_shards

  hash("user_1")   % 3 = 0  → Shard 0
  hash("user_2")   % 3 = 2  → Shard 2
  hash("user_3")   % 3 = 1  → Shard 1
  hash("user_123") % 3 = 0  → Shard 0

  ✅ Distribuição uniforme (sem hotspots)
  ✅ Simples de implementar
  ❌ Range queries impossíveis (dados espalhados)
  ❌ Add/remove shards → rehash quase tudo (solved by consistent hashing)
```

### Consistent Hashing

```
Resolve o problema de rehashing quando shards mudam:

  Hash Ring (0 — 2^32):
  
       Node A (pos: 100)
         ·
        · ·
       ·   ·
  Node D    · Node B (pos: 500)
  (pos:900) ·
       ·   ·
        · ·
         ·
       Node C (pos: 700)

  key_hash("user_1") = 350 → vai para Node B (próximo clockwise)
  key_hash("user_2") = 750 → vai para Node D (próximo clockwise)

  Adicionar Node E (pos: 600):
    Apenas keys entre C(700) e E(600) migram → mínimo rebalancing!

  Virtual nodes: cada nó tem múltiplas posições → distribuição mais uniforme

  (Ver tópico 13 - Consistent Hashing para detalhes completos)
```

### List-Based Partitioning

```
Mapeia valores específicos para partições:

  Shard Brasil:   region IN ('BR', 'PT', 'AO')
  Shard USA:      region IN ('US', 'CA', 'MX')
  Shard Europe:   region IN ('DE', 'FR', 'UK', 'ES')
  Shard Asia:     region IN ('JP', 'CN', 'KR', 'IN')

  ✅ Controle total sobre distribuição
  ✅ Data locality (users da mesma região → mesmo shard)
  ✅ Compliance (dados de EU ficam em servidor na EU — GDPR)
  ❌ Distribuição manual (pode ficar desbalanceado)
  ❌ Manutenção: novos países precisam ser mapeados
```

### Directory-Based Partitioning

```
Lookup service centralizado mapeia key → partition:

  ┌─────────────────────────────────┐
  │  Directory Service (Lookup)     │
  │                                 │
  │  user_id: 123  → Shard 3       │
  │  user_id: 456  → Shard 1       │
  │  user_id: 789  → Shard 2       │
  └─────────┬──────┬──────┬────────┘
            │      │      │
       Shard 1 Shard 2 Shard 3

  ✅ Máxima flexibilidade (qualquer esquema de routing)
  ✅ Rebalancing fácil (atualiza directory)
  ❌ SPOF: directory down → sistema down
  ❌ Latência extra: lookup antes de cada query
  ❌ Escalabilidade do directory é o bottleneck
```

---

## Comparativo

| Estratégia | Range Queries | Distribuição | Hotspot Risk | Rebalancing |
|------------|:------------:|:------------:|:------------:|:-----------:|
| **Range** | ✅ Ótimo | ❌ Pode ser ruim | 🔴 Alto | 🟡 Médio |
| **Hash** | ❌ Impossível | ✅ Uniforme | 🟢 Baixo | 🔴 Rehash tudo |
| **Consistent Hash** | ❌ Impossível | ✅ Uniforme | 🟢 Baixo | 🟢 Mínimo |
| **List** | 🟡 Dentro da lista | 🟡 Manual | 🟡 Depende | 🟡 Manual |
| **Directory** | ✅ Flexível | ✅ Controlado | 🟢 Controlado | 🟢 Fácil |

---

## Problemas Comuns

### Hot Partitions (Hotspot)

```
Problema: Um shard recebe tráfego desproporcional.

  Exemplo: Twitter — @elonmusk tem 150M followers
  Se shardear por user_id: shard do Elon recebe 1000x mais reads

  Soluções:
  1. Consistent hashing com virtual nodes
  2. Celebrity/VIP sharding separado
  3. Fan-out read: copiar hot data para múltiplas replicas
  4. Caching layer antes dos shards (Redis)
  5. Salting: adicionar sufixo aleatório ao shard key
     user_id = "elon_" + random(0,9) → distribui entre 10 shards
```

### Cross-Shard Queries

```
Problema: Query precisa de dados de múltiplos shards.

  "SELECT * FROM orders WHERE status = 'pending' ORDER BY date LIMIT 20"
  → Precisa consultar TODOS os shards e merge!

  Soluções:
  1. Scatter-Gather: query em paralelo → merge results
  2. Denormalization: duplicar dados para evitar cross-shard
  3. Secondary index global (caro mas poderoso)
  4. CQRS: read model materializado sem sharding
  5. Evitar queries cross-shard no design da shard key
```

### Rebalancing

```
Problema: Precisa adicionar/remover shards sem downtime.

  Técnicas:
  1. Consistent hashing → minimal data movement
  2. Virtual shards: muitos shards lógicos (256) em poucos nós físicos
     → mover shard lógico inteiro é mais simples
  3. Double write: durante migração, write em ambos
  4. Backfill + cutover: copiar dados, trocar routing
```

---

## Uso em Big Techs

| Empresa | Estratégia | Shard Key | Detalhes |
|---------|-----------|-----------|----------|
| **Instagram** | Hash + range | user_id | PostgreSQL sharded via pgbouncer/Citus |
| **YouTube/Vitess** | Hash | video_id | Vitess — middleware de sharding para MySQL |
| **Uber** | Hash | city_id + trip_id | Sharding por cidade para locality |
| **Cassandra** | Consistent Hash | partition key | Murmur3 hash → token ring |
| **DynamoDB** | Hash | partition key | Automatic splitting de partitions |
| **MongoDB** | Hash ou Range | shard key | Configurable por collection |
| **Stripe** | Functional | by domain | Cada serviço com DB separado |
| **Slack** | Range (by workspace) | workspace_id | Workspace isolation |

---

## Perguntas Frequentes em Entrevistas

1. **"Como você shardeia essa tabela?"**
   - Identificar access pattern principal
   - Escolher shard key que minimize cross-shard queries
   - Hash se distribuição uniforme; range se range queries necessárias

2. **"Qual shard key para um chat system?"**
   - conversation_id (todas as mensagens de uma conversa no mesmo shard)
   - Não user_id (mensagens de grupo ficariam em múltiplos shards)

3. **"Como lidar com hotspots?"**
   - Caching, read replicas, salting, virtual nodes
   - Celebrity handling separado (fan-out)

4. **"Horizontal vs vertical partitioning?"**
   - Horizontal: mesmos columns, divide rows → escala dados
   - Vertical: mesmas rows, divide columns → otimiza I/O

5. **"E se precisar adicionar shards?"**
   - Consistent hashing: mínima migração
   - Virtual shards: mover shards lógicos entre nós
   - Planeje para 2-3x capacidade atual

---

## Referências

- Alex Xu — *"System Design Interview"*, Cap. 5: Design Consistent Hashing
- Martin Kleppmann — *"Designing Data-Intensive Applications"*, Cap. 6: Partitioning
- Vitess (YouTube) — vitess.io — Horizontal sharding middleware for MySQL
- Instagram Engineering — *"Sharding & IDs at Instagram"*
- Uber Engineering — *"Schemaless: Uber's Scalable Datastore"*
- Cassandra Architecture — *"How data is distributed across a cluster"*
