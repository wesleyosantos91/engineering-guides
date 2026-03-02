# Databases — Fundamentos de Banco de Dados

> **Objetivo deste documento:** Servir como referência completa sobre **fundamentos de bancos de dados**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Cobre teoremas fundamentais (CAP, PACELC), modelos de dados, garantias transacionais e critérios de decisão entre SQL e NoSQL.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **CAP Theorem** | Escolha 2 de 3: Consistency, Availability, Partition Tolerance | Assumir que o banco garante os 3 simultaneamente | Entender o trade-off do banco escolhido (CP vs AP) |
| **PACELC** | Em partição CAP se aplica; senão, trade-off Latency vs Consistency | Ignorar o comportamento do banco em condições normais | Avaliar se o sistema prioriza latência ou consistência no dia-a-dia |
| **ACID** | Transações atômicas, consistentes, isoladas e duráveis | Assumir ACID em bancos NoSQL por padrão | Verificar garantias do banco; usar Saga pattern quando necessário |
| **BASE** | Eventualmente consistente para alta disponibilidade | Usar eventual consistency onde strong consistency é necessário | Avaliar requisitos de negócio antes de aceitar BASE |
| **Normalização** | Eliminar redundância até 3NF para OLTP | Normalizar dados de analytics (OLAP) desnecessariamente | Desnormalizar para leitura; normalizar para escrita |
| **PIE Theorem** | Escolha 2 de 3: Pattern flexibility, Infinite scale, Efficiency | Assumir que um banco atende os 3 simultâneamente | Entender trade-off PIE e escolher banco conforme prioridades |
| **Data Model** | Escolher modelo de dados baseado no padrão de acesso | Forçar modelo relacional em dados hierárquicos | Usar document store ou graph DB conforme o caso |

---

## Sumário

- [Databases — Fundamentos de Banco de Dados](#databases--fundamentos-de-banco-de-dados)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [Teorema CAP](#teorema-cap)
    - [Os 3 Pilares](#os-3-pilares)
    - [Trade-offs na Prática](#trade-offs-na-prática)
    - [Classificação de Bancos pelo CAP](#classificação-de-bancos-pelo-cap)
  - [Teorema PACELC](#teorema-pacelc)
    - [Classificação PACELC de Bancos](#classificação-pacelc-de-bancos)
  - [Teorema PIE — The Iron Triangle of Purpose](#teorema-pie--the-iron-triangle-of-purpose)
    - [Os 3 Vértices](#os-3-vértices)
    - [Combinações — Pick Two](#combinações--pick-two)
    - [Por que sistemas PE falham em escala?](#por-que-sistemas-pe-falham-em-escala)
    - [Classificação de Bancos pelo PIE](#classificação-de-bancos-pelo-pie)
    - [PIE vs CAP — Diferenças Fundamentais](#pie-vs-cap--diferenças-fundamentais)
    - [Critérios para Escolha — Checklist PIE](#critérios-para-escolha--checklist-pie)
  - [ACID vs BASE](#acid-vs-base)
    - [ACID — Relacional Tradicional](#acid--relacional-tradicional)
    - [BASE — NoSQL e Sistemas Distribuídos](#base--nosql-e-sistemas-distribuídos)
    - [Quando usar cada um](#quando-usar-cada-um)
  - [Modelos de Dados](#modelos-de-dados)
    - [Relacional (Tabelas)](#relacional-tabelas)
    - [Document (JSON/BSON)](#document-jsonbson)
    - [Key-Value](#key-value)
    - [Column-Family (Wide-Column)](#column-family-wide-column)
    - [Graph](#graph)
    - [Time-Series](#time-series)
    - [Vector](#vector)
  - [Consistência — Níveis e Modelos](#consistência--níveis-e-modelos)
  - [Árvore de Decisão — Qual Banco Usar?](#árvore-de-decisão--qual-banco-usar)
  - [Anti-patterns Fundamentais](#anti-patterns-fundamentais)
  - [Referências](#referências)

---

## Teorema CAP

O **Teorema CAP** (Eric Brewer, 2000) estabelece que um sistema distribuído **não pode garantir simultaneamente** as três propriedades. Em caso de **partição de rede** (P), o sistema deve escolher entre **Consistência (C)** e **Disponibilidade (A)**.

### Os 3 Pilares

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TEOREMA CAP                                  │
│                                                                     │
│                          C                                          │
│                         ╱ ╲                                         │
│                        ╱   ╲                                        │
│                       ╱     ╲                                       │
│                      ╱  CP   ╲                                      │
│                     ╱         ╲                                     │
│                    ╱           ╲                                    │
│                   A ─────────── P                                   │
│                        AP                                           │
│                                                                     │
│  C (Consistency)  → Todos os nós veem os mesmos dados ao mesmo     │
│                     tempo. Toda leitura recebe o dado mais recente. │
│                                                                     │
│  A (Availability) → Toda requisição recebe uma resposta (sucesso   │
│                     ou falha), mesmo que nem todos os nós estejam   │
│                     atualizados.                                    │
│                                                                     │
│  P (Partition     → O sistema continua operando mesmo com falhas   │
│     Tolerance)      de comunicação entre nós.                       │
│                                                                     │
│  ⚠️  Em produção, partições ACONTECEM. Logo, a escolha real        │
│      é entre CP (forte consistência) e AP (alta disponibilidade).  │
└─────────────────────────────────────────────────────────────────────┘
```

### Trade-offs na Prática

| Tipo | Comportamento durante partição | Exemplo de uso | Bancos |
|------|-------------------------------|----------------|--------|
| **CP** | Recusa escritas/leituras para manter consistência | Transações financeiras, inventário | MongoDB (default), HBase, Redis Cluster, Zookeeper |
| **AP** | Aceita escritas em todos os nós, resolve conflitos depois | Redes sociais, catálogos de produtos | Cassandra, DynamoDB, CouchDB, Riak |
| **CA** | Não tolera partições (single-node ou rede perfeita) | Bancos tradicionais single-instance | PostgreSQL (single), MySQL (single), Oracle (single) |

> **Nota importante:** CA só é possível em ambientes **sem partições de rede**, o que na prática significa single-node. Em ambientes distribuídos, **P é inevitável**, tornando a escolha real entre CP e AP.

### Classificação de Bancos pelo CAP

```
┌──────────────────────────────────────────────────────────────────┐
│                 BANCOS NO ESPECTRO CAP                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CP (Consistência + Partição)                                   │
│  ├── MongoDB (w:majority, r:majority)                           │
│  ├── HBase                                                       │
│  ├── Redis Cluster                                               │
│  ├── etcd                                                        │
│  ├── Zookeeper                                                   │
│  └── Google Cloud Spanner                                        │
│                                                                  │
│  AP (Disponibilidade + Partição)                                │
│  ├── Cassandra (tunable consistency)                             │
│  ├── Amazon DynamoDB (eventual consistency default)              │
│  ├── CouchDB                                                     │
│  ├── Riak                                                        │
│  ├── ScyllaDB                                                    │
│  └── Couchbase                                                   │
│                                                                  │
│  CA (Consistência + Disponibilidade) — single-node only         │
│  ├── PostgreSQL (standalone)                                     │
│  ├── MySQL (standalone)                                          │
│  ├── Oracle (standalone)                                         │
│  └── SQL Server (standalone)                                     │
│                                                                  │
│  ⚠️  Muitos bancos são "tunable" — permitem ajustar o trade-off │
│     Ex: Cassandra com ALL = CP, Cassandra com ONE = AP          │
│     Ex: DynamoDB com strongly consistent reads = CP-like        │
└──────────────────────────────────────────────────────────────────┘
```

---

## Teorema PACELC

O **Teorema PACELC** (Daniel Abadi, 2012) é uma extensão do CAP. Ele afirma:

> **Se** há Partição (P), escolha entre **Availability (A)** e **Consistency (C)**.
> **Senão (Else)**, escolha entre **Latency (L)** e **Consistency (C)**.

O PACELC captura o trade-off que existe **mesmo quando não há partições** — o que o CAP ignora.

```
┌──────────────────────────────────────────────────────────────────┐
│                     TEOREMA PACELC                                │
│                                                                  │
│        ┌──── Partição? ────┐                                    │
│        │ SIM               │ NÃO                                │
│        ▼                   ▼                                    │
│   ┌─────────┐        ┌──────────┐                               │
│   │ P: A/C  │        │ E: L/C   │                               │
│   │         │        │          │                                │
│   │ A → Alta│        │ L → Baixa│                               │
│   │   disp. │        │   latência│                              │
│   │         │        │          │                                │
│   │ C → Dados│       │ C → Dados│                               │
│   │   corretos│      │   corretos│                              │
│   └─────────┘        └──────────┘                               │
│                                                                  │
│  Formato: P[A|C] / E[L|C]                                      │
│  Exemplo: PA/EL = Em partição prioriza Availability,            │
│           sem partição prioriza Latency                          │
└──────────────────────────────────────────────────────────────────┘
```

### Classificação PACELC de Bancos

| Banco | Partição (P) | Else (E) | Classificação | Descrição |
|-------|-------------|----------|---------------|-----------|
| **DynamoDB** | A (disponibilidade) | L (latência) | PA/EL | Prioriza disponibilidade e baixa latência |
| **Cassandra** | A (disponibilidade) | L (latência) | PA/EL | Baixa latência em operações normais |
| **Riak** | A (disponibilidade) | L (latência) | PA/EL | Foco em alta disponibilidade e performance |
| **MongoDB** | C (consistência) | C (consistência) | PC/EC | Consistência forte é a prioridade |
| **HBase** | C (consistência) | C (consistência) | PC/EC | Modelo CP forte, otimizado para consistência |
| **PostgreSQL** | C (consistência) | C (consistência) | PC/EC | ACID estrito, consistência acima de tudo |
| **Cosmos DB** | Tunable | Tunable | Configurável | 5 níveis de consistência (Strong → Eventual) |
| **CockroachDB** | C (consistência) | L (latência) | PC/EL | Consistência em partição, latência no normal |
| **ScyllaDB** | A (disponibilidade) | L (latência) | PA/EL | Fork C++ do Cassandra, foco em performance |
| **Google Spanner** | C (consistência) | C (consistência) | PC/EC | Consistência global forte com TrueTime |

> **Insight prático:** A maioria dos sistemas web modernos precisa de **PA/EL** — alta disponibilidade durante falhas e baixa latência no dia-a-dia. Sistemas financeiros e de inventário geralmente precisam de **PC/EC**.

---

## Teorema PIE — The Iron Triangle of Purpose

O **Teorema PIE** (também chamado de **Iron Triangle of Purpose**), apresentado pela AWS na sessão **re:Invent 2018: Match Your Workload to the Right Database**, é um framework para escolha de bancos de dados baseado em **propósito de uso e eficiência**.

Enquanto o **CAP** apresenta trade-offs de **disponibilidade vs consistência** em cenários de falha, o **PIE** apresenta trade-offs de **eficiência e otimização** — ajudando a evitar viéses que o CAP pode introduzir (como sempre escolher consistência sobre disponibilidade). Assim como o CAP, o PIE afirma que você só pode otimizar para **dois dos três** vértices simultaneamente.

> **"Pick Two"** — Nenhum banco de dados consegue ser excelente nos três ao mesmo tempo. A terceira característica não pode ser plenamente atendida em um único sistema.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                  THE IRON TRIANGLE OF PURPOSE                           │
│                    (Teorema PIE / Pick Two)                              │
│                                                                          │
│                              P                                           │
│                             ╱ ╲                                          │
│                            ╱   ╲                                         │
│                           ╱     ╲                                        │
│                     PI   ╱       ╲   PE                                  │
│                         ╱  Pick   ╲                                      │
│                        ╱   Two     ╲                                     │
│                       ╱             ╲                                    │
│                      I ───────────── E                                   │
│                             IE                                           │
│                                                                          │
│  P = Pattern Flexibility    Suporta padrões de acesso aleatórios         │
│                              e queries ad hoc                            │
│                                                                          │
│  I = Infinite Scale         Escala graciosamente em tamanho e            │
│                              throughput sem limites práticos              │
│                                                                          │
│  E = Efficiency             Entrega a latência necessária para           │
│                              o workload em todos os momentos             │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Os 3 Vértices

| Vértice | Nome | Definição | Exemplos de Capacidade |
|---------|------|-----------|------------------------|
| **P** | **Pattern Flexibility** | O banco suporta **padrões de acesso aleatórios** e **queries ad hoc** | JOINs complexos, agregações, filtros arbitrários, subqueries |
| **I** | **Infinite Scale** | O banco consegue **crescer graciosamente** em tamanho e throughput **sem limites práticos** | Auto-sharding, horizontal scaling, petabytes de dados |
| **E** | **Efficiency** | O banco entrega a **latência necessária** para o workload **em todos os momentos** | Latência single-digit ms, performance previsível, SLAs rígidos |

**Detalhamento dos vértices:**

- **Pattern Flexibility (P):** Quando não conhecemos de forma antecipada quais consultas serão executadas pela base de usuários. Aplicações de múltiplos propósitos têm maiores chances de precisarem de flexibilidade. Essa flexibilidade se dá por meio de **normalização** e **Ad Hoc Queries**.

- **Infinite Scale (I):** Para aplicações cujo volume de dados e throughput crescem de forma ilimitada sem indisponibilidade. Sistemas como Instagram, Reddit e Waze são exemplos onde a escala precisa ser "infinita".

- **Efficiency (E):** Para aplicações que necessitam atender centenas, milhares ou milhões de requisições com **pouca ou nenhuma oscilação no tempo de resposta**. Aplicações que toleram diferentes tempos de resposta para diferentes usuários possivelmente não necessitam dessa dimensão.

### Combinações — Pick Two

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     COMBINAÇÕES PIE                                      │
│                                                                          │
│  ┌─────────┐   P + E = PE  (Flexibilidade + Eficiência)                    │
│  │   PE    │   Bancos relacionais OLTP tradicionais                      │
│  │         │   ✅ Dados normalizados (3NF), Ad Hoc Queries                │
│  │         │   ✅ Workloads mistos (transacional + analítico)             │
│  │         │   ❌ SACRIFICA Infinite Scale (escala vertical)              │
│  │         │   📦 PostgreSQL, Oracle, SQL Server, RDS, Aurora             │
│  │         │   💡 Ideal para: OLTP, APIs, Aplicações de negócio         │
│  └─────────┘                                                             │
│                                                                          │
│  ┌─────────┐   I + E = IE  (Escala Infinita + Eficiência)                  │
│  │   IE    │   Distributed Hash Tables / NoSQL                          │
│  │         │   ✅ Denormalização, escala horizontal nativa               │
│  │         │   ✅ Latência previsível em qualquer escala                 │
│  │         │   ❌ SACRIFICA Pattern Flexibility (sem JOINs/Ad Hoc)       │
│  │         │   📦 DynamoDB, MongoDB, Cassandra, HBase, ScyllaDB           │
│  │         │   💡 Ideal para: Apps web de alta escala (Instagram,         │
│  │         │      Reddit, Waze), IoT, Gaming, Sessões                    │
│  └─────────┘                                                             │
│                                                                          │
│  ┌─────────┐   P + I = PI  (Flexibilidade + Escala Infinita)              │
│  │   PI    │   Data Warehouses / OLAP                                    │
│  │         │   ✅ Queries SQL flexíveis em dados massivos                │
│  │         │   ✅ Oscilações no tempo de resposta são aceitáveis        │
│  │         │   ❌ SACRIFICA Efficiency (latência variável)               │
│  │         │   📦 Redshift, Snowflake, Athena, BigQuery, Hive             │
│  │         │   💡 Ideal para: Analytics, Data Lakes, Data Warehouse      │
│  └─────────┘                                                             │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Por que sistemas PE falham em escala?

Sistemas **PE** são a escolha natural para a maioria das aplicações de negócio. Porém, quando a aplicação cresce como um sistema web de larga escala (Instagram, Reddit, Waze), sistemas PE se tornam ineficientes por dois motivos:

```
┌──────────────────────────────────────────────────────────────────────────┐
│         POR QUE PE FALHA EM ESCALA?                                     │
│                                                                          │
│  1. Escala Vertical tem limites físicos                                  │
│     │  Sistemas PE escalam verticalmente (mais CPU/RAM).                 │
│     │  Em certo ponto, o hardware atinge seu limite máximo.              │
│     └── Resultado: indisponibilidade ou degradação.                       │
│                                                                          │
│  2. JOINs em grande escala são ineficientes                              │
│     │  JOINs em grandes volumes apresentam latência logarítmica.         │
│     │  Normalização (3NF) = mais JOINs = mais latência em escala.       │
│     └── Resultado: queries cada vez mais lentas conforme dados crescem.  │
│                                                                          │
│  Solução: Migrar para IE                                                 │
│     Denormalizar, duplicar dados, armazenar em documentos.               │
│     Habilita escala horizontal nativa, abrindo mão de Ad Hoc Queries.   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Classificação de Bancos pelo PIE

| Banco | Classificação | O que ganha | O que sacrifica | Caso de uso típico |
|-------|:------------:|-------------|-----------------|--------------------|
| **PostgreSQL** | PE | Extensibilidade, SQL avançado | Escala horizontal difícil | APIs, dados complexos |
| **Oracle** | PE | SQL enterprise, ACID robusto | Escala vertical, custo | Aplicações corporativas |
| **SQL Server** | PE | Integração Microsoft, T-SQL | Escala limitada (vertical) | Aplicações .NET, enterprise |
| **Amazon RDS** | PE | SQL gerenciado, ACID, JOINs | Escala limitada (vertical) | OLTP, aplicações web |
| **Aurora** | PE | SQL + performance 5x MySQL | Escala limitada (128TB) | Aplicações web de alta performance |
| **Aurora Serverless** | PE | SQL com auto-scaling | Escala limitada vs DynamoDB | Workloads intermitentes |
| **Amazon Neptune** | PE | Queries de grafo flexíveis | Escala limitada | Knowledge graphs, recomendações |
| **Elasticsearch** | PE | Full-text search, agregações | Escala complexa de gerenciar | Search, log analytics |
| **Amazon DynamoDB** | IE | Single-digit ms em qualquer escala | Acesso apenas por PK/SK | IoT, gaming, sessions |
| **MongoDB** | IE | Documents flexíveis, escala horizontal | Queries ad hoc limitadas em escala | Apps web, catálogos, conteúdo |
| **Apache Cassandra** | IE | Escala linear, alta escrita | CQL limitado (sem JOINs) | Time-series, IoT massivo |
| **ScyllaDB** | IE | Cassandra-compatible, mais rápido | Mesmas limitações do Cassandra | Alta throughput, baixa latência |
| **HBase** | IE | Escala sobre HDFS | API limitada, operacional | Hadoop ecosystem, analytics em tempo real |
| **Amazon Redshift** | PI | Queries SQL flexíveis em petabytes | Latência (segundos a minutos) | Data Warehouse, Analytics |
| **Snowflake** | PI | SQL serverless, separação compute/storage | Latência variável por query | Data Warehouse, Data Lake |
| **Amazon Athena** | PI | SQL sobre S3, sem infra para gerenciar | Latência (serverless, cold start) | Queries ad hoc em Data Lake |
| **BigQuery** | PI | Analytics serverless em escala massiva | Latência variável | Analytics, ML workloads |
| **Apache Hive** | PI | Queries SQL sobre Hadoop/HDFS | Latência alta (batch) | ETL, processamento batch |

### PIE vs CAP — Diferenças Fundamentais

| Aspecto | CAP Theorem | PIE Theorem |
|---------|-------------|-------------|
| **Foco** | Comportamento em **falhas de rede** | Escolha de banco por **propósito/workload** |
| **Quando se aplica** | Sistemas distribuídos sob partição | **Sempre** — decisão de arquitetura |
| **Trade-off** | Consistency vs Availability | Flexibility vs Scale vs Efficiency |
| **Pergunta** | "O que acontece quando a rede falha?" | "Para que tipo de workload este banco é ideal?" |
| **Natureza** | Teórico / limitação física | Prático / guia de decisão |
| **Complementar?** | Sim — use CAP + PIE juntos | Sim — PIE ajuda a filtrar, CAP ajuda a validar |
| **Viés** | Tende a favorecer Consistency sobre Availability | Equilibrado — foca em eficiência do workload |

> **Insight prático:** Se basearmos nossas escolhas arquiteturais **unicamente no CAP**, possivelmente escolheremos Consistência sobre Disponibilidade. O PIE complementa o CAP ao apresentar uma perspectiva de **eficiência e otimização**, ajudando a evitar possíveis viéses.
>
> Use o **PIE** como **primeiro filtro** para escolher a categoria de banco com base no workload. Em seguida, use **CAP/PACELC** para validar o comportamento em cenários de falha.

### Critérios para Escolha — Checklist PIE

A escolha do sistema adequado depende dos requisitos específicos da aplicação:

| # | Critério | PE | IE | PI |
|---|----------|:--:|:--:|:--:|
| 1 | Padrões de acesso desconhecidos / Ad Hoc Queries | ✅ | ❌ | ✅ |
| 2 | Volume de dados tende ao infinito | ❌ | ✅ | ✅ |
| 3 | Latência previsível e baixa é mandatória | ✅ | ✅ | ❌ |
| 4 | Queries complexas com JOINs | ✅ | ❌ | ✅ |
| 5 | Escalabilidade horizontal nativa | ❌ | ✅ | ✅ |
| 6 | Workloads mistos (transacional + analítico) | ✅ | ❌ | ❌ |
| 7 | Oscilação no tempo de resposta é aceitável | ❌ | ❌ | ✅ |

```
┌──────────────────────────────────────────────────────────────────────────┐
│              WORKFLOW DE DECISÃO: PIE → CAP/PACELC                      │
│                                                                          │
│  1. Identifique seu workload principal                                   │
│     │                                                                    │
│  2. Aplique o PIE Theorem                                                │
│     ├── Analytics/OLAP? ──────────── → PI (Redshift, Athena)            │
│     ├── OLTP/API? ────────────────── → PE (RDS, Aurora, PostgreSQL)     │
│     └── Alta escala + baixa latência? → IE (DynamoDB, Cassandra)        │
│     │                                                                    │
│  3. Valide com CAP/PACELC                                                │
│     ├── Precisa de consistência forte em partição? → CP (PC/EC)         │
│     └── Prioriza disponibilidade? ──────────────── → AP (PA/EL)         │
│     │                                                                    │
│  4. Escolha o banco específico                                           │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## ACID vs BASE

### ACID — Relacional Tradicional

```
┌──────────────────────────────────────────────────────────────────┐
│                      ACID                                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  A — Atomicity (Atomicidade)                                    │
│  │   Transação é tudo-ou-nada.                                  │
│  │   Se qualquer parte falha, TUDO é revertido.                 │
│  │   Ex: Transferência bancária debita E credita, ou nenhum.    │
│  │                                                               │
│  C — Consistency (Consistência)                                  │
│  │   Transação leva o banco de um estado válido a outro.        │
│  │   Constraints, triggers e regras são respeitados.            │
│  │   Ex: Foreign keys, unique constraints, check constraints.   │
│  │                                                               │
│  I — Isolation (Isolamento)                                      │
│  │   Transações concorrentes não interferem entre si.           │
│  │   Nível configurável: Read Uncommitted → Serializable.       │
│  │   Ex: Duas compras simultâneas do último item em estoque.    │
│  │                                                               │
│  D — Durability (Durabilidade)                                   │
│  │   Uma vez commitado, o dado sobrevive a falhas.              │
│  │   WAL (Write-Ahead Log) garante persistência.                │
│  │   Ex: Crash após commit → dado está salvo.                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Níveis de Isolamento (do menos ao mais restritivo):**

| Nível | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------|-----------|-------------------|-------------|-------------|
| **Read Uncommitted** | ✅ Possível | ✅ Possível | ✅ Possível | Mais rápido |
| **Read Committed** | ❌ Bloqueado | ✅ Possível | ✅ Possível | Rápido |
| **Repeatable Read** | ❌ Bloqueado | ❌ Bloqueado | ✅ Possível | Moderado |
| **Serializable** | ❌ Bloqueado | ❌ Bloqueado | ❌ Bloqueado | Mais lento |

> **Padrão recomendado:** `Read Committed` para a maioria dos casos OLTP. Use `Serializable` apenas quando a integridade absoluta é crítica (ex: transações financeiras).

### BASE — NoSQL e Sistemas Distribuídos

```
┌──────────────────────────────────────────────────────────────────┐
│                      BASE                                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  BA — Basically Available (Basicamente Disponível)              │
│  │    O sistema garante disponibilidade, mesmo que               │
│  │    respostas possam estar desatualizadas.                     │
│  │                                                               │
│  S  — Soft State (Estado Flexível)                               │
│  │    O estado do sistema pode mudar ao longo do tempo,          │
│  │    mesmo sem input, devido à propagação eventual.             │
│  │                                                               │
│  E  — Eventually Consistent (Eventualmente Consistente)          │
│  │    O sistema eventualmente convergirá para um estado          │
│  │    consistente, dado tempo suficiente sem novas escritas.     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Quando usar cada um

| Critério | ACID | BASE |
|----------|------|------|
| **Transações financeiras** | ✅ Obrigatório | ❌ Risco |
| **Redes sociais (likes, posts)** | ❌ Over-engineering | ✅ Ideal |
| **Inventário/estoque** | ✅ Recomendado | ⚠️ Com compensação |
| **Catálogo de produtos** | ⚠️ Pode ser demais | ✅ Ideal |
| **Sessões de usuário** | ❌ Desnecessário | ✅ Ideal |
| **Auditoria/compliance** | ✅ Obrigatório | ❌ Insuficiente |
| **IoT / telemetria** | ❌ Não escala | ✅ Ideal |
| **E-commerce checkout** | ✅ Recomendado | ❌ Risco de inconsistência |

---

## Modelos de Dados

### Relacional (Tabelas)

```
┌────────────────────────────────────────────────────────────────────┐
│                    MODELO RELACIONAL                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Estrutura: Tabelas com linhas e colunas                          │
│  Relações: Foreign Keys, JOINs                                    │
│  Schema: Rígido (schema-on-write)                                 │
│  Linguagem: SQL                                                    │
│                                                                    │
│  ┌──────────┐     ┌──────────────┐     ┌───────────┐             │
│  │ users    │     │ orders       │     │ products  │             │
│  ├──────────┤     ├──────────────┤     ├───────────┤             │
│  │ id (PK)  │◄────│ user_id (FK) │     │ id (PK)   │             │
│  │ name     │     │ product_id───│────►│ name      │             │
│  │ email    │     │ quantity     │     │ price     │             │
│  │ created  │     │ total        │     │ category  │             │
│  └──────────┘     └──────────────┘     └───────────┘             │
│                                                                    │
│  ✅ Melhor para: Dados estruturados, relações complexas,          │
│     transações ACID, queries ad-hoc, reporting                    │
│  ❌ Não ideal para: Dados sem schema, escala horizontal massiva, │
│     hierarquias profundas, dados polimórficos                     │
│                                                                    │
│  Bancos: PostgreSQL, MySQL, Oracle, SQL Server, Aurora, MariaDB  │
└────────────────────────────────────────────────────────────────────┘
```

### Document (JSON/BSON)

```
┌────────────────────────────────────────────────────────────────────┐
│                    MODELO DOCUMENT                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Estrutura: Documentos JSON/BSON agrupados em coleções            │
│  Relações: Embedded documents ou referências                      │
│  Schema: Flexível (schema-on-read)                                │
│  Linguagem: Query API específica do banco                         │
│                                                                    │
│  {                                                                 │
│    "_id": "user-123",                                             │
│    "name": "João Silva",                                          │
│    "email": "joao@email.com",                                    │
│    "orders": [                                                     │
│      {                                                             │
│        "id": "ord-456",                                           │
│        "products": [                                               │
│          { "name": "Laptop", "price": 4500.00 }                   │
│        ],                                                          │
│        "total": 4500.00                                           │
│      }                                                             │
│    ],                                                              │
│    "preferences": {                                                │
│      "theme": "dark",                                             │
│      "notifications": true                                        │
│    }                                                               │
│  }                                                                 │
│                                                                    │
│  ✅ Melhor para: Dados semi-estruturados, schemas evolutivos,     │
│     content management, catálogos, perfis de usuário              │
│  ❌ Não ideal para: Relações complexas N:N, transações multi-doc, │
│     queries com muitos JOINs, strong consistency requerida        │
│                                                                    │
│  Bancos: MongoDB, CouchDB, DocumentDB, Firestore, Couchbase      │
└────────────────────────────────────────────────────────────────────┘
```

### Key-Value

```
┌────────────────────────────────────────────────────────────────────┐
│                    MODELO KEY-VALUE                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Estrutura: Pares chave → valor (valor pode ser qualquer coisa)  │
│  Relações: Nenhuma nativa                                         │
│  Schema: Schema-less                                               │
│  Operações: GET, PUT, DELETE — O(1)                               │
│                                                                    │
│  ┌───────────────────┬──────────────────────────────┐             │
│  │ Key               │ Value                         │             │
│  ├───────────────────┼──────────────────────────────┤             │
│  │ session:abc123    │ {user: "joao", ttl: 3600}    │             │
│  │ cache:product:42  │ {name: "Laptop", price: 4500}│             │
│  │ config:feature:x  │ true                         │             │
│  │ rate:ip:10.0.0.1  │ 150                          │             │
│  └───────────────────┴──────────────────────────────┘             │
│                                                                    │
│  ✅ Melhor para: Cache, sessões, feature flags, rate limiting,    │
│     filas simples, contadores, leaderboards                       │
│  ❌ Não ideal para: Queries por valor, relações, range queries    │
│     complexas, dados que precisam de índices secundários          │
│                                                                    │
│  Bancos: Redis, Memcached, DynamoDB, Riak, etcd                  │
└────────────────────────────────────────────────────────────────────┘
```

### Column-Family (Wide-Column)

```
┌────────────────────────────────────────────────────────────────────┐
│                MODELO COLUMN-FAMILY (WIDE-COLUMN)                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Estrutura: Linhas com famílias de colunas dinâmicas              │
│  Relações: Nenhuma nativa — dados desnormalizados                 │
│  Schema: Semi-estruturado (column families definidas)             │
│  Otimizado para: Escritas massivas, dados esparsos, time-series  │
│                                                                    │
│  Row Key      │ info:name  │ info:email       │ stats:logins     │
│  ─────────────┼────────────┼──────────────────┼──────────────────│
│  user:001     │ João       │ joao@email.com   │ 142              │
│  user:002     │ Maria      │ maria@email.com  │ (não definido)   │
│  user:003     │ Pedro      │ pedro@email.com  │ 89               │
│                                                                    │
│  ✅ Melhor para: Dados de IoT/telemetria, logs, time-series,     │
│     analytics pesado, write-heavy workloads, dados esparsos      │
│  ❌ Não ideal para: Transações ACID, queries ad-hoc complexas,   │
│     dados com muitas relações, atualizações frequentes           │
│                                                                    │
│  Bancos: Cassandra, HBase, ScyllaDB, Google Bigtable             │
└────────────────────────────────────────────────────────────────────┘
```

### Graph

```
┌────────────────────────────────────────────────────────────────────┐
│                    MODELO GRAPH                                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Estrutura: Nós (vértices) + Arestas (relações) + Propriedades   │
│  Relações: Cidadãs de primeira classe                             │
│  Schema: Flexível                                                  │
│  Linguagem: Cypher (Neo4j), Gremlin (TinkerPop), SPARQL (RDF)   │
│                                                                    │
│       ┌──────────┐  COMPROU   ┌──────────┐                        │
│       │  João    │───────────►│  Laptop  │                        │
│       │ (Person) │            │(Product) │                        │
│       └────┬─────┘            └──────────┘                        │
│            │                       ▲                               │
│       AMIGO_DE              VISUALIZOU                            │
│            │                       │                               │
│       ┌────▼─────┐            ┌────┴─────┐                        │
│       │  Maria   │            │  Pedro   │                        │
│       │ (Person) │            │ (Person) │                        │
│       └──────────┘            └──────────┘                        │
│                                                                    │
│  ✅ Melhor para: Redes sociais, recomendação, detecção de fraude, │
│     knowledge graphs, análise de impacto, dependency graphs      │
│  ❌ Não ideal para: Dados tabulares simples, heavy writes,       │
│     aggregations massivas, dados sem relações claras             │
│                                                                    │
│  Bancos: Neo4j, Amazon Neptune, JanusGraph, ArangoDB, TigerGraph │
└────────────────────────────────────────────────────────────────────┘
```

### Time-Series

```
┌────────────────────────────────────────────────────────────────────┐
│                    MODELO TIME-SERIES                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Estrutura: Séries temporais — timestamp como dimensão principal  │
│  Otimizado para: Escritas append-only, aggregations temporais    │
│  Compressão: Otimizada para dados sequenciais (delta encoding)   │
│                                                                    │
│  timestamp            │ sensor_id │ temperature │ humidity        │
│  ─────────────────────┼───────────┼─────────────┼─────────────── │
│  2025-01-01T00:00:00  │ s-001     │ 23.5        │ 65.2           │
│  2025-01-01T00:01:00  │ s-001     │ 23.6        │ 65.1           │
│  2025-01-01T00:02:00  │ s-001     │ 23.4        │ 65.3           │
│                                                                    │
│  ✅ Melhor para: IoT, métricas de infraestrutura, dados          │
│     financeiros (ticks), logs de eventos, analytics temporal     │
│  ❌ Não ideal para: Queries por relação, dados sem componente    │
│     temporal, CRUD genérico, transações complexas                │
│                                                                    │
│  Bancos: TimescaleDB, InfluxDB, Amazon Timestream, QuestDB,     │
│          Prometheus (métricas), ClickHouse                        │
└────────────────────────────────────────────────────────────────────┘
```

### Vector

```
┌────────────────────────────────────────────────────────────────────┐
│                    MODELO VECTOR                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Estrutura: Vetores de alta dimensão (embeddings)                 │
│  Operação chave: Similarity search (ANN — Approximate Nearest    │
│                  Neighbor)                                         │
│  Uso principal: Busca semântica, RAG, recomendação por embedding │
│                                                                    │
│  Documento               Embedding (simplificado)                 │
│  ───────────────────────  ────────────────────────                 │
│  "Banco de dados SQL"    [0.12, 0.85, 0.43, ...]                 │
│  "Database relacional"   [0.13, 0.84, 0.42, ...] ← similar!     │
│  "Receita de bolo"       [0.91, 0.02, 0.77, ...] ← diferente    │
│                                                                    │
│  ✅ Melhor para: RAG com LLMs, busca semântica, recomendação,    │
│     detecção de duplicatas, image similarity                     │
│  ❌ Não ideal para: Queries exatas, dados transacionais,         │
│     relações complexas, dados sem embeddings                     │
│                                                                    │
│  Bancos: Pinecone, Weaviate, Milvus, Qdrant, ChromaDB,          │
│          pgvector (PostgreSQL), OpenSearch (k-NN)                │
└────────────────────────────────────────────────────────────────────┘
```

---

## Consistência — Níveis e Modelos

```
┌──────────────────────────────────────────────────────────────────┐
│              ESPECTRO DE CONSISTÊNCIA                             │
│                                                                  │
│  Mais Forte ◄──────────────────────────────► Mais Fraca         │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐│
│  │Lineariz- │  │Sequential│  │ Causal   │  │  Eventual        ││
│  │ability   │  │Consistency│  │Consistency│  │  Consistency    ││
│  │          │  │          │  │          │  │                  ││
│  │ Toda     │  │ Ops em   │  │ Causally │  │ Dados convergem ││
│  │ leitura  │  │ ordem    │  │ related  │  │ eventualmente   ││
│  │ retorna  │  │ global   │  │ ops em   │  │ Sem garantia de ││
│  │ o último │  │ consistente│ │ ordem    │  │ quando          ││
│  │ write    │  │          │  │          │  │                  ││
│  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘│
│       ▲                                           ▲              │
│  + Consistência                          + Performance          │
│  - Performance                           - Consistência         │
│  - Disponibilidade                       + Disponibilidade      │
└──────────────────────────────────────────────────────────────────┘
```

| Modelo | Garantia | Latência | Uso típico |
|--------|----------|----------|------------|
| **Linearizability** | Leitura sempre retorna o último write | Alta | Locks distribuídos, leader election |
| **Sequential Consistency** | Todas operações em ordem global | Moderada-Alta | Replicated state machines |
| **Causal Consistency** | Operações causalmente relacionadas em ordem | Moderada | Collaborative editing, chat |
| **Eventual Consistency** | Dados convergem sem garantia de tempo | Baixa | DNS, caches, redes sociais |
| **Read-your-writes** | Usuário sempre vê seus próprios writes | Baixa-Moderada | User profiles, shopping carts |
| **Monotonic Reads** | Leituras nunca retrocedem no tempo | Baixa-Moderada | Feeds, dashboards |

---

## Árvore de Decisão — Qual Banco Usar?

```
┌──────────────────────────────────────────────────────────────────┐
│          ÁRVORE DE DECISÃO: Qual Tipo de Banco?                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Qual é o padrão principal de acesso?                           │
│  │                                                               │
│  ├── Relações complexas com JOINs frequentes?                   │
│  │   ├── SIM → Dados estruturados e schema estável?             │
│  │   │   ├── SIM → Relacional (PostgreSQL, MySQL, Aurora)       │
│  │   │   └── NÃO → Relações são o foco? → Graph (Neo4j,Neptune)│
│  │   └── NÃO → Continua ▼                                      │
│  │                                                               │
│  ├── Acesso por chave com latência < 10ms?                      │
│  │   ├── SIM → Dados simples (cache/session)?                   │
│  │   │   ├── SIM → Key-Value (Redis, DynamoDB)                  │
│  │   │   └── NÃO → Documentos JSON ricos?                      │
│  │   │       ├── SIM → Document (MongoDB, DocumentDB)           │
│  │   │       └── NÃO → Key-Value (DynamoDB)                     │
│  │   └── NÃO → Continua ▼                                      │
│  │                                                               │
│  ├── Dados temporais (métricas, IoT, logs)?                     │
│  │   └── SIM → Time-Series (TimescaleDB, InfluxDB, Timestream) │
│  │                                                               │
│  ├── Escritas massivas com dados esparsos?                      │
│  │   └── SIM → Wide-Column (Cassandra, ScyllaDB, HBase)        │
│  │                                                               │
│  ├── Busca full-text ou analytics de logs?                      │
│  │   └── SIM → Search Engine (Elasticsearch, OpenSearch)        │
│  │                                                               │
│  └── Busca semântica com embeddings?                            │
│      └── SIM → Vector DB (Pinecone, pgvector, Weaviate)        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Anti-patterns Fundamentais

| # | Anti-pattern | Problema | Solução |
|---|-------------|----------|---------|
| 1 | **One Database to Rule Them All** | Usar um único banco para todos os workloads | Polyglot persistence — banco certo para cada caso |
| 2 | **Premature NoSQL** | Migrar para NoSQL sem necessidade real | Começar com relacional; migrar quando limites forem atingidos |
| 3 | **Ignoring CAP** | Escolher banco sem entender trade-offs de consistência | Mapear requisitos de consistência antes de escolher |
| 4 | **Schemaless = No Schema** | Achar que document DB dispensa modelagem | Definir schemas implícitos, validar na aplicação ou com JSON Schema |
| 5 | **NoSQL with SQL Mindset** | Modelar NoSQL como relacional (normalizado demais) | Modelar por access patterns, desnormalizar quando necessário |
| 6 | **SQL with NoSQL Mindset** | Desnormalizar tudo em banco relacional | Normalizar adequadamente; usar views materializadas para leitura |
| 7 | **Over-normalization** | Normalizar até 5NF+ causando JOINs excessivos | 3NF para OLTP é suficiente; desnormalizar seletivamente |
| 8 | **Eventual Consistency Everywhere** | Aceitar eventual consistency sem avaliar impacto | Usar strong consistency para dados críticos (financeiro, estoque) |
| 9 | **Ignorar Access Patterns** | Escolher banco sem mapear como os dados serão acessados | Access patterns definem o modelo — comece por eles |
| 10 | **Vendor Lock-in Cego** | Acoplamento total a banco proprietário sem abstração | Camada de abstração; considerar portabilidade desde o início |

---

## Referências

- Brewer, E. (2000). *Towards Robust Distributed Systems* — CAP Theorem
- Abadi, D. (2012). *Consistency Tradeoffs in Modern Distributed Database System Design* — PACELC
- AWS re:Invent 2018. *The Iron Triangle of Purpose (PIE Theorem)* — Purpose-Built Databases
- Nomura, D. *Bancos de Dados Distribuídos — Teorema de PIE* — https://www.linkedin.com/pulse/bancos-de-dados-distribu%C3%ADdos-teorema-pie-diogo-nomura
- Kleppmann, M. (2017). *Designing Data-Intensive Applications* — O'Reilly
- Fowler, M. (2012). *NoSQL Distilled* — Addison-Wesley
- Amazon. *AWS Database Services Overview* — https://aws.amazon.com/products/databases/
- Google. *Cloud Spanner CAP Theorem* — https://cloud.google.com/spanner/docs/true-time-external-consistency
- MongoDB. *Consistency in MongoDB* — https://www.mongodb.com/docs/manual/core/read-isolation-consistency-recency/
