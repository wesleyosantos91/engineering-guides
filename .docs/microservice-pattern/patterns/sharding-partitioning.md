# Sharding & Partitioning

> **Categoria:** Event-Driven e Mensageria / Persistência
> **Complementa:** CQRS, Backpressure, Rate Limiter

---

## Problema

Quando o volume de dados ou de mensagens excede a capacidade de um **único nó**:

- Uma tabela com bilhões de registros: queries ficam lentas, índices enormes, backups demorados
- Um tópico de mensagens com milhões de mensagens/segundo: um único consumidor não dá conta
- Uma API com milhões de requests: um único servidor não suporta

**Escalar verticalmente** (mais CPU, RAM) tem limite. A solução é **distribuir** dados e carga entre múltiplos nós.

---

## Conceitos

| Conceito | Definição |
|---------|-----------|
| **Partitioning** | Dividir dados de uma entidade em pedaços menores (partições) dentro do mesmo sistema |
| **Sharding** | Distribuir partições entre múltiplos nós/servidores independentes |

Na prática, os termos são frequentemente usados de forma intercambiável. A diferença sutil:

```
Partitioning:  Uma tabela dividida em partições (mesmo banco, mesmo servidor)
Sharding:      Partições distribuídas entre servidores/bancos diferentes
```

---

## Tipos de Particionamento

### 1. Particionamento por Range (Faixa)

Dados distribuídos com base em um **intervalo** de valores:

```
Partition 1: orders WHERE created_at < '2026-01-01'
Partition 2: orders WHERE created_at BETWEEN '2026-01-01' AND '2026-06-30'
Partition 3: orders WHERE created_at >= '2026-07-01'
```

| Vantagem | Desvantagem |
|----------|------------|
| Range queries eficientes | Hotspot: partições recentes recebem mais escrita |
| Fácil de entender | Pode desbalancear (dados não uniformes) |
| Ideal para dados temporais | Novas partições devem ser criadas periodicamente |

### 2. Particionamento por Hash

Aplica função hash na **partition key** e distribui uniformemente:

```
partition = hash(order_id) % num_partitions

hash("order-001") % 4 = 2  → Partition 2
hash("order-002") % 4 = 0  → Partition 0
hash("order-003") % 4 = 3  → Partition 3
```

| Vantagem | Desvantagem |
|----------|------------|
| Distribuição uniforme | Range queries ineficientes (dados espalhados) |
| Sem hotspots (geralmente) | Rebalanceamento ao mudar nº de partições |
| Simples de implementar | Precisa de boa partition key |

### 3. Particionamento por Lista

Dados distribuídos por **valores específicos**:

```
Partition BR: orders WHERE country = 'BR'
Partition US: orders WHERE country = 'US'
Partition EU: orders WHERE country IN ('DE', 'FR', 'ES', 'IT')
```

| Vantagem | Desvantagem |
|----------|------------|
| Fácil de entender | Desbalanceia se um valor dominar (ex: 80% BR) |
| Queries por valor = single partition | Limitado a valores predefinidos |

### 4. Particionamento Composto

Combina dois tipos:

```
Primeiro nível: Hash por user_id → determina o shard
Segundo nível: Range por created_at → dentro do shard, partições temporais
```

---

## Escolha da Partition Key (Shard Key)

A **partition key** é a decisão mais crítica do sharding. Ela determina como os dados são distribuídos.

### Propriedades de uma boa Partition Key

| Propriedade | Por quê |
|------------|---------|
| **Alta cardinalidade** | Muitos valores distintos → distribuição uniforme |
| **Distribuição uniforme** | Sem hotspots (evitar que uma partição tenha muito mais dados) |
| **Usada nas queries** | Queries que filtram pela key acessam uma única partição |
| **Imutável** | Mudar a key exige mover dados entre partições |

### Exemplos

| Domínio | Boa Partition Key | Partition Key ruim |
|---------|------------------|--------------------|
| E-commerce | `order_id` | `status` (poucos valores) |
| Multi-tenant | `tenant_id` | `created_at` (hotspot em datas recentes) |
| Chat | `conversation_id` | `user_id` de celebridade (hotspot) |
| IoT | `device_id` | `region` (poucos valores, desbalanceia) |
| Logs | `service_name + date` | `level` (INFO domina) |

---

## Sharding em Bancos de Dados

### Abordagens

```
┌── Sharding no nível da aplicação ──┐
│                                    │
│  App decide qual shard consultar   │
│  baseado na partition key          │
│                                    │
│  App → shard_1 (users A-M)        │
│      → shard_2 (users N-Z)        │
└────────────────────────────────────┘

┌── Sharding no nível do proxy ──────┐
│                                    │
│  Proxy/Router roteia a query para  │
│  o shard correto                   │
│                                    │
│  App → Proxy → shard_1             │
│              → shard_2             │
└────────────────────────────────────┘

┌── Sharding nativo do banco ────────┐
│                                    │
│  O banco gerencia sharding         │
│  automaticamente (ex: CockroachDB, │
│  Vitess for MySQL, Citus for PG)   │
│                                    │
└────────────────────────────────────┘
```

---

## Particionamento em Message Brokers

### Kafka: Partições por Tópico

```
Tópico: order-events (6 partições)

┌── Partition 0 ── Consumer A ──┐
│── Partition 1 ── Consumer A ──│
│── Partition 2 ── Consumer B ──│  Consumer Group
│── Partition 3 ── Consumer B ──│
│── Partition 4 ── Consumer C ──│
└── Partition 5 ── Consumer C ──┘

Mensagem chega:
  key = "order-123"
  partition = hash("order-123") % 6 = 2  → Partition 2 → Consumer B
```

**Garantia:** Mensagens com a **mesma key** vão para a **mesma partição** → ordem garantida para esse key.

### Regras de Dimensionamento

| Fator | Consideração |
|-------|-------------|
| **Nº de partições ≥ nº de consumers** | Cada consumer consome de pelo menos 1 partição |
| **Mais partições = mais paralelismo** | Mas mais overhead de coordenação |
| **Partição é a unidade de ordenação** | Ordem garantida DENTRO de uma partição apenas |
| **Rebalanceamento** | Ao adicionar/remover consumers, partições são redistribuídas |

---

## Problemas Conhecidos

### 1. Cross-Shard Queries

Queries que precisam de dados de múltiplas partições/shards:

```
// Query simples (single shard):
SELECT * FROM orders WHERE user_id = '123'  → vai para shard do user 123

// Query cross-shard:
SELECT COUNT(*) FROM orders WHERE status = 'PENDING'  → precisa consultar TODOS os shards
```

**Soluções:**
- **Scatter-Gather:** Consulta todos os shards em paralelo e agrega
- **Tabela agregada:** Mantenha uma tabela de contagens pré-calculada
- **CQRS:** Read model desnormalizado não é shardado (ou shardado diferente)

### 2. Hotspots

Uma partição recebe muito mais carga que as outras:

```
Sharding por user_id → Usuário com 1 milhão de seguidores causa hotspot

Soluções:
- Salting: user_id + random_suffix → espalha dados do mesmo user
- Sub-particionamento: divide o hotspot em sub-partições
```

### 3. Rebalanceamento

Ao mudar o número de partições, dados precisam ser redistribuídos:

```
Antes: 4 partições → hash(key) % 4
Depois: 6 partições → hash(key) % 6

Quase TODOS os dados mudam de partição! → Migração massiva
```

**Solução: Consistent Hashing** — minimiza a quantidade de dados que muda ao adicionar/remover nós. Apenas ~1/N dos dados migra (onde N = número de partições).

---

## Exemplo Conceitual (Pseudocódigo)

```
// Sharding no nível da aplicação
class ShardRouter:
    shards = [
        Shard(id=0, connection="db-shard-0"),
        Shard(id=1, connection="db-shard-1"),
        Shard(id=2, connection="db-shard-2"),
        Shard(id=3, connection="db-shard-3"),
    ]
    
    function getShardFor(partitionKey):
        shardIndex = hash(partitionKey) % shards.size
        return shards[shardIndex]
    
    function executeQuery(partitionKey, query):
        shard = getShardFor(partitionKey)
        return shard.connection.execute(query)
    
    function executeGlobalQuery(query):
        // Scatter-gather: consulta todos os shards em paralelo
        results = parallel(shards.map(s -> s.connection.execute(query)))
        return merge(results)
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Partition key com baixa cardinalidade | Poucas partições, distribuição desigual | Use key com alta cardinalidade |
| Sharding prematuro | Complexidade sem necessidade | Comece com um banco, otimize queries, escale verticalmente, só depois sharde |
| Cross-shard joins frequentes | Performance pior que sem sharding | Redesenhe o modelo ou use CQRS |
| Sem consistent hashing | Rebalanceamento move dados demais | Use consistent hashing |
| Mensageria: poucas partições | Limita o paralelismo de consumers | Dimensione partições para o máximo de consumers esperado |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **CQRS** | Read model pode ter particionamento diferente do write model |
| **Backpressure** | Partições permitem paralelismo que absorve picos (com backpressure) |
| **Rate Limiter** | Rate limiting por partição/shard para balancear carga |
| **Event Sourcing** | Event store shardado por aggregate_id |
| **CDC** | CDC captura mudanças de cada shard independentemente |

---

## Boas Práticas

1. Escolha a **partition key** com cuidado — é a decisão mais impactante.
2. **Não sharde prematuramente** — comece simples, sharde quando necessário.
3. Use **consistent hashing** para minimizar rebalanceamento.
4. Em mensageria, dimensione partições para o **máximo de consumers** esperado.
5. Monitore **distribuição** entre partições/shards para detectar hotspots.
6. Planeje para **cross-shard queries** — elas serão necessárias eventualmente.
7. Use **partition key = aggregate key** em mensageria para garantir ordenação por entidade.
8. Automatize o **rebalanceamento** com ferramentas do banco ou broker.

---

## Referências

- Martin Kleppmann — *Designing Data-Intensive Applications* — Chapter 6: Partitioning
- Confluent — [How to Choose the Number of Topics/Partitions in a Kafka Cluster](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster/)
- AWS — [Sharding Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/modernization-data-persistence/sharding.html)
