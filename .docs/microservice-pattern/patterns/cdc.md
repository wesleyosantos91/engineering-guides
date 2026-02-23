# CDC (Change Data Capture)

> **Categoria:** Event-Driven e Mensageria
> **Complementa:** Outbox Pattern, CQRS, Event Sourcing

---

## Problema

Quando um sistema precisa reagir a mudanças de dados em um banco de dados:

- **Polling** é ineficiente (carga no banco, latência, difícil detectar deletes)
- **Dual write** (salvar + publicar evento) é inconsistente
- Sistemas **legados** não publicam eventos — apenas salvam no banco
- Sincronizar dados entre bancos (write → read, CQRS) requer mecanismo confiável

---

## Solução

**Capturar mudanças** diretamente do **transaction log** (binlog, WAL) do banco de dados e transformá-las em eventos que são publicados em um message broker. O banco de dados já registra cada mudança no log — CDC lê esse log em tempo real.

```
┌──────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────────┐
│ Aplicação│────▶│  Banco de    │────▶│  CDC        │────▶│  Message     │
│          │     │  Dados       │     │  Connector  │     │  Broker      │
│          │     │  (binlog/WAL)│     │             │     │  (Kafka etc.)│
└──────────┘     └──────────────┘     └─────────────┘     └──────┬───────┘
                                                                  │
                                                           ┌──────▼───────┐
                                                           │  Consumers   │
                                                           │  (sync, CQRS,│
                                                           │   analytics) │
                                                           └──────────────┘
```

---

## Como Funciona

### Transaction Log (Binlog / WAL)

Todo banco de dados relacional mantém um **log de transações** que registra cada mudança:

| Banco | Nome do Log | Formato |
|-------|------------|---------|
| MySQL | Binary Log (binlog) | ROW, STATEMENT, MIXED |
| PostgreSQL | Write-Ahead Log (WAL) | Logical Decoding |
| SQL Server | Transaction Log | CDC nativo |
| Oracle | Redo Log | LogMiner / GoldenGate |
| MongoDB | Oplog | Change Streams |

### Fluxo de Captura

```
1. Aplicação executa: INSERT INTO orders (id, total) VALUES ('o-1', 1500)

2. Banco grava no binlog/WAL:
   { table: "orders", op: "INSERT", data: {id: "o-1", total: 1500}, ts: ... }

3. CDC Connector lê o binlog em tempo real:
   → Transforma em evento estruturado

4. Publica no message broker:
   Topic: "db.schema.orders"
   Key: "o-1"
   Value: {
     "op": "c",         // c=create, u=update, d=delete, r=read(snapshot)
     "before": null,     // estado anterior (null para INSERT)
     "after": { "id": "o-1", "total": 1500 },
     "source": { "table": "orders", "db": "mydb", "ts_ms": 1708686600000 },
     "ts_ms": 1708686600050
   }
```

---

## Tipos de Operações Capturadas

| Operação | Código | before | after |
|----------|--------|--------|-------|
| **INSERT** | `c` (create) | null | novo registro |
| **UPDATE** | `u` (update) | registro antes | registro depois |
| **DELETE** | `d` (delete) | registro antes | null |
| **SNAPSHOT** | `r` (read) | null | registro atual (carga inicial) |

### Exemplo de UPDATE

```
UPDATE orders SET status = 'SHIPPED' WHERE id = 'o-1'

Evento CDC:
{
  "op": "u",
  "before": { "id": "o-1", "status": "PAID", "total": 1500 },
  "after":  { "id": "o-1", "status": "SHIPPED", "total": 1500 },
  "source": { "table": "orders", "ts_ms": 1708686600000 }
}
```

Consumers podem comparar `before` vs `after` para reagir a mudanças específicas.

---

## Casos de Uso

### 1. Sincronização Write DB → Read DB (CQRS)

```
Write DB (MySQL) ──CDC──▶ Kafka ──▶ Read DB (Elasticsearch)
```

Cada INSERT/UPDATE no MySQL é automaticamente replicado no Elasticsearch. Sem código adicional na aplicação.

### 2. Outbox Pattern via CDC

```
Tabela outbox ──CDC──▶ Kafka ──▶ Consumers

A aplicação salva na outbox (mesma transação).
CDC lê a outbox e publica no Kafka.
Não precisa de polling — CDC é em tempo real.
```

### 3. Sincronização entre Microsserviços

```
Serviço A (banco A) ──CDC──▶ Kafka ──▶ Serviço B (replica local)
```

Serviço B mantém uma cópia dos dados relevantes de A, atualizada em tempo real.

### 4. Analytics / Data Lake

```
Bancos OLTP ──CDC──▶ Kafka ──▶ Data Lake / Data Warehouse
```

Streaming de dados transacionais para analytics sem impactar o banco operacional.

### 5. Migração de Sistemas Legados

```
Sistema Legado (banco) ──CDC──▶ Kafka ──▶ Novo Sistema
```

O sistema legado não precisa ser modificado — CDC lê diretamente do banco.

---

## CDC vs. Outras Abordagens

| Abordagem | Latência | Impacto no Banco | Detecta Deletes | Complexidade |
|-----------|---------|------------------|-----------------|-------------|
| **CDC (binlog/WAL)** | Sub-segundo | Nenhum (lê o log) | Sim | Média-Alta |
| **Polling (SELECT)** | Segundos | Alto (queries constantes) | Difícil | Baixa |
| **Triggers** | Tempo real | Alto (executa no write path) | Sim | Média |
| **Dual Write (app)** | Tempo real | Nenhum | Sim | Baixa, mas inconsistente |
| **Outbox + Polling** | Segundos | Médio | N/A | Média |

---

## Snapshot Inicial

Quando o CDC é conectado pela primeira vez, ele faz um **snapshot** de todos os dados existentes antes de começar a capturar mudanças incrementais:

```
Fase 1 — Snapshot:
  Lê todos os registros existentes → publica como eventos (op: "r")

Fase 2 — Streaming:
  Captura mudanças incrementais em tempo real (INSERT, UPDATE, DELETE)
```

---

## Transformações

Eventos CDC podem ser **transformados** antes de chegar ao consumer:

| Transformação | Descrição |
|--------------|-----------|
| **Filtro de tabelas** | Capturar apenas tabelas específicas |
| **Filtro de colunas** | Excluir colunas sensíveis (ex: password) |
| **Roteamento** | Direcionar para tópicos diferentes baseado na tabela |
| **Outbox Router** | Extrair payload da outbox e publicar como evento de negócio |
| **Flatten** | Extrair apenas `after` (simplificar formato) |
| **Masking** | Mascarar dados sensíveis (PII) |

---

## Exemplo Conceitual (Pseudocódigo)

```
// Configuração conceitual de CDC (baseado em connector)
CdcConnector:
    source:
        type: mysql
        host: db-host
        port: 3306
        database: order_db
        tables: ["orders", "order_items"]
    
    sink:
        type: kafka
        bootstrap_servers: kafka:9092
        topic_prefix: "cdc.order_db"
    
    transforms:
        - type: outbox_router     // se usando outbox pattern
          table: "outbox_events"
          route_by: "destination"
        
        - type: column_filter
          exclude: ["password", "ssn"]  // remove dados sensíveis

// Consumer do CDC
class OrderSyncConsumer:
    readDB
    
    function onCdcEvent(event):
        match event.op:
            "c": readDB.insert(event.after)
            "u": readDB.update(event.after)
            "d": readDB.delete(event.before.id)
            "r": readDB.upsert(event.after)  // snapshot
```

---

## Problemas e Soluções

| Problema | Descrição | Solução |
|---------|-----------|---------|
| **Schema Evolution** | Tabela muda (nova coluna, tipo alterado) | Use schema registry; CDC propaga mudanças automaticamente |
| **Ordering** | Eventos de tabelas diferentes podem chegar fora de ordem | Use table-level topic + aggregate key como partition |
| **Volume** | Tabela com alta escrita gera muitos eventos | Filtre tabelas/colunas; use compaction no Kafka |
| **Permissões** | CDC precisa ler binlog — requer permissões especiais | Configure usuário com REPLICATION SLAVE (MySQL) ou replication role (PG) |
| **Latência do snapshot** | Snapshot inicial de tabela grande pode demorar horas | Execute em horário de baixo uso; use snapshot parallel |

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| CDC sem schema registry | Mudanças no schema quebram consumers | Use schema registry para versionamento |
| Capturar todas as tabelas | Volume excessivo, broker sobrecarregado | Capture apenas tabelas relevantes |
| Consumer sem idempotência | Reprocessamento (após falha do consumer) duplica dados | Consumer deve ser idempotente |
| Depender de CDC para lógica de negócio | Acoplamento ao schema do banco | Use Outbox Pattern para eventos de negócio + CDC para transporte |
| Sem monitoramento do lag | Consumers defasados sem visibilidade | Monitore lag do connector e consumers |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Outbox Pattern** | CDC é o mecanismo ideal para ler a tabela outbox sem polling |
| **CQRS** | CDC sincroniza o Write DB → Read DB em tempo real |
| **Event Sourcing** | CDC captura mudanças no banco; Event Sourcing armazena eventos explicitamente |
| **Strangler Fig** | CDC permite que o sistema novo receba dados do legado sem modificá-lo |
| **Idempotência** | Consumers de CDC devem ser idempotentes |

---

## Boas Práticas

1. Use **CDC via transaction log** (binlog/WAL) — nunca triggers ou polling para alta carga.
2. Filtre **apenas tabelas e colunas necessárias** para reduzir volume.
3. Use **schema registry** para gerenciar evolução de schema.
4. Para eventos de **negócio**, combine **Outbox + CDC** — outbox define o evento, CDC transporta.
5. Monitore o **lag do connector** — atraso indica problema de capacidade.
6. Consumers devem ser **idempotentes**.
7. Planeje o **snapshot inicial** para horários de baixo uso.
8. **Mascare dados sensíveis** (PII) antes de publicar no broker.

---

## Referências

- Debezium — [Documentation](https://debezium.io/documentation/)
- Martin Kleppmann — *Designing Data-Intensive Applications* — Chapter 11: Stream Processing
- Confluent — [Change Data Capture](https://www.confluent.io/learn/change-data-capture/)
- Microsoft — [Change Data Capture](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server)
