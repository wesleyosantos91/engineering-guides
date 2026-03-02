# Level 0 — Fundamentos de Banco de Dados

> **Objetivo:** Dominar os teoremas fundamentais (CAP, PACELC, PIE), compreender ACID vs BASE,
> e saber classificar e escolher o modelo de dados correto para cada cenário.

**Referência:** [.docs/databases/01-database-foundations.md](../../.docs/databases/01-database-foundations.md)

---

## Contexto do Domínio

Antes de modelar a **StreamX**, você precisa entender os fundamentos que guiam **toda decisão de banco de dados**.
Neste nível, você analisará os trade-offs teóricos e aplicará no domínio real de streaming:

- Quais dados exigem **consistência forte** (assinaturas, billing)?
- Quais dados podem ser **eventualmente consistentes** (visualizações, recomendações)?
- Qual **modelo de dados** é ideal para cada entidade?
- Como classificar cada banco usando **CAP, PACELC e PIE**?

---

## Desafios

### Desafio 0.1 — Classificação CAP do Ecossistema StreamX

**Contexto:** A StreamX usa múltiplos bancos de dados. Antes de modelar qualquer schema,
você precisa entender os trade-offs de cada banco no contexto do Teorema CAP.

**Requisitos:**

- Classificar os seguintes bancos de dados no Teorema CAP:
  - PostgreSQL, MongoDB, Redis, Cassandra, Neo4j, DynamoDB
- Para cada banco, indicar:
  - Classificação (CP, AP, ou CA com ressalvas)
  - O que acontece durante uma **partição de rede** (qual propriedade é sacrificada)
  - Cenário na StreamX onde esse trade-off é aceitável/inaceitável
- Criar um diagrama (ASCII ou Mermaid) posicionando cada banco no triângulo CAP
- Documentar: por que **CA puro não existe** em sistemas distribuídos?

**Critérios de aceite:**

- [ ] 6 bancos classificados com justificativa técnica
- [ ] Diagrama CAP com posicionamento visual de cada banco
- [ ] Pelo menos 1 cenário StreamX para cada classificação (CP e AP)
- [ ] Explicação documentada de por que CA não é viável com partições de rede
- [ ] Documento entregue em Markdown (arquivo `decisions/00-cap-classification.md`)

---

### Desafio 0.2 — Análise PACELC e PIE para Decisões de Design

**Contexto:** O CAP é insuficiente para decisões práticas porque só descreve comportamento
**durante partições**. O PACELC estende para operação normal, e o PIE adiciona a dimensão
de propósito do banco.

**Requisitos:**

- Classificar os 6 bancos (PostgreSQL, MongoDB, Redis, Cassandra, Neo4j, DynamoDB) nos 3 teoremas:
  - **CAP:** CP ou AP
  - **PACELC:** PA/EL, PA/EC, PC/EL, ou PC/EC
  - **PIE:** PE (Pattern + Efficiency), IE (Infinite Scale + Efficiency), ou PI (Pattern + Infinite Scale)
- Criar uma tabela comparativa com os 3 teoremas lado a lado
- Responder: "Para o sistema de **billing da StreamX** (assinaturas, cobranças), qual combinação
  CAP/PACELC/PIE é obrigatória? Justifique."
- Responder: "Para o sistema de **watch events** (bilhões de eventos de reprodução), qual
  combinação é ideal? Justifique."

**Critérios de aceite:**

- [ ] Tabela com 3 colunas de teoremas para cada um dos 6 bancos
- [ ] Justificativa técnica para billing (deve ser PC/EC ou equivalente)
- [ ] Justificativa técnica para watch events (deve priorizar escala e throughput)
- [ ] Comparação explícita: "PIE complementa CAP porque..."
- [ ] Documento: `decisions/00-pacelc-pie-analysis.md`

---

### Desafio 0.3 — ACID vs BASE: Implementação com Trade-offs

**Contexto:** A StreamX precisa de ACID para billing e BASE para streaming analytics.
Implemente ambos os modelos e documente quando usar cada um.

**Requisitos:**

- Implementar uma transação **ACID** para o cenário:
  - Usuário assina plano → criar `Subscription`, debitar `Payment`, atualizar `User.status`
  - Tudo em uma única transação PostgreSQL com rollback em caso de falha
- Implementar uma operação **BASE** para o cenário:
  - Registrar um watch event → gravar no Cassandra com eventual consistency
  - Aceitar que a contagem de views pode divergir momentaneamente entre réplicas
- Documentar os 4 níveis de isolamento SQL (Read Uncommitted → Serializable)
- Implementar um teste que demonstra **dirty read** (Read Uncommitted) vs **prevenção** (Read Committed)

**Java 25:**
```java
// ACID — Transação de assinatura
try (var conn = dataSource.getConnection()) {
    conn.setAutoCommit(false);
    conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
    
    try {
        // 1. Criar subscription
        try (var stmt = conn.prepareStatement(
                "INSERT INTO subscriptions (id, user_id, plan_id, status, started_at) VALUES (?, ?, ?, 'ACTIVE', NOW())")) {
            stmt.setObject(1, subscriptionId);
            stmt.setObject(2, userId);
            stmt.setObject(3, planId);
            stmt.executeUpdate();
        }
        // 2. Registrar payment
        try (var stmt = conn.prepareStatement(
                "INSERT INTO payments (id, subscription_id, amount, currency, status) VALUES (?, ?, ?, 'BRL', 'COMPLETED')")) {
            stmt.setObject(1, paymentId);
            stmt.setObject(2, subscriptionId);
            stmt.setBigDecimal(3, plan.price());
            stmt.executeUpdate();
        }
        // 3. Atualizar status do user
        try (var stmt = conn.prepareStatement(
                "UPDATE users SET status = 'SUBSCRIBER', updated_at = NOW() WHERE id = ?")) {
            stmt.setObject(1, userId);
            stmt.executeUpdate();
        }
        conn.commit();
    } catch (Exception e) {
        conn.rollback();
        throw e;
    }
}
```

**Go 1.26:**
```go
// ACID — Transação de assinatura
tx, err := db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelReadCommitted})
if err != nil {
    return fmt.Errorf("begin tx: %w", err)
}
defer tx.Rollback()

if _, err = tx.ExecContext(ctx,
    "INSERT INTO subscriptions (id, user_id, plan_id, status, started_at) VALUES ($1, $2, $3, 'ACTIVE', NOW())",
    subscriptionID, userID, planID); err != nil {
    return fmt.Errorf("insert subscription: %w", err)
}

if _, err = tx.ExecContext(ctx,
    "INSERT INTO payments (id, subscription_id, amount, currency, status) VALUES ($1, $2, $3, 'BRL', 'COMPLETED')",
    paymentID, subscriptionID, plan.Price); err != nil {
    return fmt.Errorf("insert payment: %w", err)
}

if _, err = tx.ExecContext(ctx,
    "UPDATE users SET status = 'SUBSCRIBER', updated_at = NOW() WHERE id = $1",
    userID); err != nil {
    return fmt.Errorf("update user: %w", err)
}

return tx.Commit()
```

**Critérios de aceite:**

- [ ] Transação ACID funcional em PostgreSQL (Java e Go)
- [ ] Rollback automático em caso de falha em qualquer passo
- [ ] Operação BASE funcional com eventual consistency (Cassandra ou simulação)
- [ ] Tabela comparativa ACID vs BASE com 5+ critérios (consistência, disponibilidade, latência, etc.)
- [ ] Teste demonstrando diferença entre níveis de isolamento
- [ ] Documento: `decisions/00-acid-vs-base.md`

---

### Desafio 0.4 — Mapeamento de Modelos de Dados

**Contexto:** Cada entidade da StreamX se encaixa naturalmente em um modelo de dados diferente.
Mapeie todas as entidades para o modelo ideal.

**Requisitos:**

- Para cada um dos 7 modelos de dados (Relational, Document, Key-Value, Wide-Column, Graph, Time-Series, Vector),
  listar pelo menos uma entidade StreamX que se encaixa e justificar
- Criar um diagrama de "decisão de modelo" tipo árvore de decisão:
  - Dados transacionais com relações fortes? → Relacional
  - Metadata flexível por tipo? → Document
  - Acesso por chave com TTL? → Key-Value
  - Escrita massiva com consulta por partição? → Wide-Column
  - Relações complexas N-hops? → Graph
  - Dados temporais com aggregation? → Time-Series
  - Embeddings e busca semântica? → Vector
- Para o modelo **Vector**, descrever como a StreamX usaria embeddings:
  - Gerar embedding de cada conteúdo (título + sinopse + gênero)
  - Busca semântica: "filmes parecidos com Interstellar" → similarity search

**Critérios de aceite:**

- [ ] 7 modelos mapeados com pelo menos 1 entidade StreamX cada
- [ ] Árvore de decisão em ASCII ou Mermaid com pelo menos 7 folhas
- [ ] Justificativa de trade-off para cada escolha (por que NÃO usar relacional para tudo?)
- [ ] Descrição do caso de uso Vector com exemplo de query similarity
- [ ] Documento: `decisions/00-data-model-mapping.md`

---

### Desafio 0.5 — Anti-patterns: O Que NÃO Fazer

**Contexto:** É comum cometer erros de escolha de banco de dados, especialmente por
desconhecimento dos trade-offs fundamentais.

**Requisitos:**

- Analisar 5 anti-patterns comuns e mapear para cenários StreamX:
  1. **One DB to Rule Them All** — usar PostgreSQL para tudo (inclusive cache, eventos, grafos)
  2. **Premature NoSQL** — adotar Cassandra com 1000 usuários por "escalabilidade futura"
  3. **Ignoring CAP** — usar eventual consistency para billing/pagamentos
  4. **NoSQL for Reports** — tentar fazer relatórios analíticos complexos no MongoDB
  5. **Schemaless = No Schema** — inserir dados sem validação em document store
- Para cada anti-pattern:
  - Descrever o **problema** concreto no contexto StreamX
  - Mostrar um **exemplo de código ruim** (SQL ou NoSQL) que demonstra o erro
  - Apresentar a **solução correta** com justificativa

**Critérios de aceite:**

- [ ] 5 anti-patterns documentados com exemplo de código
- [ ] Cada anti-pattern tem: cenário StreamX, código ruim, solução, justificativa
- [ ] Pelo menos 2 anti-patterns incluem queries SQL/NoSQL reais
- [ ] Documento: `decisions/00-anti-patterns.md`

---

### Desafio 0.6 — Design Decision Record: Escolha de Bancos

**Contexto:** Formalize a decisão de quais bancos de dados a StreamX usará, usando o formato
Architecture Decision Record (ADR).

**Requisitos:**

- Criar um ADR formal (seguindo template Nygard) com:
  - **Título:** Escolha de bancos de dados para plataforma StreamX
  - **Contexto:** Requisitos funcionais e não-funcionais do sistema
  - **Decisão:** Quais bancos para quais workloads
  - **Consequências:** Trade-offs aceitos
- O ADR deve justificar a escolha de **pelo menos 4 bancos diferentes**
- Incluir métricas esperadas:
  - Users: ~10M, Subscriptions: ~5M, Watch Events: ~1B/mês
  - Latência alvo: billing < 200ms, catalog < 50ms, events < 20ms
  - Disponibilidade: billing 99.99%, catalog 99.9%, events 99.5%

**Critérios de aceite:**

- [ ] ADR completo seguindo template Nygard (Status, Context, Decision, Consequences)
- [ ] Pelo menos 4 bancos justificados com métricas
- [ ] Trade-offs explícitos (o que se ganha e o que se perde com cada escolha)
- [ ] Requisitos não-funcionais quantificados (latência, throughput, disponibilidade)
- [ ] Documento: `decisions/00-adr-database-selection.md`
