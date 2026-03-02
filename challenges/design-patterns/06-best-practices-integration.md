# Level 6 — Boas Práticas & Integração

> **Objetivo:** Aplicar princípios de boas práticas de engenharia de software ao Payment Processor,
> identificar e corrigir anti-patterns, e consolidar a capacidade de tomar decisões de design conscientes.

**Referência:** [06-best-practices.md](../../../.docs/DESIGN-PATTERNS/06-best-practices.md)

---

## Pré-requisito

[Level 5 — Padrões Arquiteturais](05-architectural-patterns.md) completo.

---

## Desafios

### Desafio 6.1 — Composition over Inheritance

> *"Favoreça composição de objetos sobre herança de classes."*

**Cenário:** Revise todo o Payment Processor e identifique onde herança foi usada nos Levels anteriores. Refatore para composição onde apropriado, mantendo herança apenas onde é genuinamente uma relação "é-um".

**Antes (herança problemática):**
```java
class CreditCardPayment extends Payment {
    // herda tudo de Payment + adiciona lógica de cartão
}
class PixPayment extends Payment {
    // herda tudo de Payment + adiciona lógica de PIX
}
// Problema: e se precisar de CreditCardPixPayment? Diamond problem.
```

**Depois (composição):**
```java
record Transaction(
    TransactionId id,
    Money amount,
    PaymentMethod method,       // composição: quem paga
    FeeStrategy feeStrategy,    // composição: como calcula fee
    TransactionState state      // composição: ciclo de vida
) { }
```

**Tarefas:**
1. Auditar todo o código por uso de herança
2. Classificar cada herança como legítima ("é-um") ou candidata a composição ("tem-um")
3. Refatorar herança → composição onde aplicável
4. Documentar cada decisão em `REFACTORING-LOG.md`

**Java vs Go:**

| Aspecto | Java 25 | Go 1.26 |
|---------|---------|---------|
| Herança | `extends` (classes), `implements` (interfaces) | **Não existe** — composição by design |
| Composição | Campos do tipo interface, records | Struct embedding, campos de interface |
| Trade-off | Escolha consciente entre herança e composição | Composição é o único caminho |

**Critérios de aceite:**
- [ ] Auditoria: lista completa de todos os usos de herança com classificação
- [ ] Refatoração de ≥ 3 heranças para composição
- [ ] Justificativa documentada para cada herança mantida
- [ ] Go: demonstrar struct embedding como alternativa a herança
- [ ] Zero impacto em funcionalidade — testes passando antes e depois
- [ ] `REFACTORING-LOG.md` com before/after de cada mudança

---

### Desafio 6.2 — DRY, KISS, YAGNI: Code Simplification

> *"Don't Repeat Yourself. Keep It Simple, Stupid. You Aren't Gonna Need It."*

**Cenário:** Após 5 levels de adição de padrões, o código pode ter complexidade acidental: abstrações desnecessárias, duplicação sutil, generalizações prematuras. Simplifique.

**Tarefas:**

**DRY — Identificar e eliminar duplicação:**
- Buscar lógica duplicada entre commands, handlers, validators
- Extrair código comum sem criar abstrações forçadas
- Atenção: DRY é sobre **conhecimento** duplicado, não código textualmente similar

**KISS — Simplificar complexidade desnecessária:**
- Padrão aplicado onde bastava uma função simples?
- Hierarquia de classes onde bastava uma `interface` + 2 implementações?
- Se um pattern não justifica sua complexidade, remova e documente por quê

**YAGNI — Remover código especulativo:**
- Features implementadas "para o futuro" que ninguém usa
- Interfaces com um único implementador (justificado por testabilidade? Ou YAGNI?)
- Generalizações que nunca foram necessárias

| Prática | Pergunta-chave | Ação se violado |
|---------|---------------|-----------------|
| **DRY** | "Este conhecimento está em dois lugares?" | Extrair para uma única fonte |
| **KISS** | "A solução mais simples resolveria?" | Simplificar sem perder funcionalidade |
| **YAGNI** | "Alguém usa isso hoje?" | Remover e documentar decisão |

**Critérios de aceite:**
- [ ] Auditoria DRY: ≥ 5 duplicações identificadas e resolvidas
- [ ] Auditoria KISS: ≥ 3 simplificações realizadas
- [ ] Auditoria YAGNI: ≥ 2 remoções de código especulativo
- [ ] Cada mudança documentada com justificativa
- [ ] Nenhuma funcionalidade dos testes foi quebrada
- [ ] `SIMPLIFICATION-LOG.md` com análise e decisões

---

### Desafio 6.3 — Law of Demeter & Tell, Don't Ask

> *"Fale apenas com seus amigos imediatos. Diga o que fazer, não pergunte o que aconteceu."*

**Cenário:** Identifique violações da Law of Demeter (train wrecks: `a.getB().getC().doSomething()`) e violações de Tell Don't Ask (procedural code que consulta estado e decide externamente).

**Antes (Tell Don't Ask violado):**
```java
// ASK: obtém estado e decide fora do objeto
if (transaction.getStatus() == AUTHORIZED) {
    if (transaction.getAmount().getValue() > 0) {
        gateway.capture(transaction.getId(), transaction.getAmount());
        transaction.setStatus(CAPTURED);
    }
}
```

**Depois (Tell, Don't Ask — comportamento no objeto):**
```java
// TELL: diz ao objeto o que fazer; objeto decide internamente
transaction.capture(gateway);
// Transaction internamente verifica estado, valor, e delega ao gateway
```

**Antes (Law of Demeter violada):**
```java
customer.getWallet().getDefaultCard().getNumber(); // train wreck
```

**Depois (Law of Demeter respeitada):**
```java
customer.getDefaultCardNumber(); // delega internamente
```

**Tarefas:**
1. Buscar train wrecks (cadeias de chamadas > 1 nível)
2. Buscar código procedural que pergunta e decide externamente
3. Refatorar movendo comportamento para o objeto certo
4. Atenção: Lei de Demeter NÃO se aplica a fluent builders, streams, ou data structures

**Critérios de aceite:**
- [ ] ≥ 5 violações de Demeter identificadas e refatoradas
- [ ] ≥ 5 violações de Tell Don't Ask identificadas e refatoradas
- [ ] Exceções documentadas: builders, streams, DTOs (onde Demeter não se aplica)
- [ ] Código resultante com métodos mais coesos e menos acoplados
- [ ] Testes passando sem alteração de comportamento

---

### Desafio 6.4 — Fail Fast & Defensive Programming

> *"Falhe imediatamente quando algo está errado. Proteja-se de inputs inválidos."*

**Cenário:** Implemente validação completa no Payment Processor com a filosofia "fail fast" — detectar problemas o mais cedo possível, com mensagens claras.

**Requisitos:**

**Fail Fast — Preconditions em toda entrada de método:**
```java
// Java: validação no construtor do record
public record Money(long cents, Currency currency) {
    public Money {
        if (cents < 0) throw new IllegalArgumentException("Amount cannot be negative: " + cents);
        Objects.requireNonNull(currency, "Currency is required");
    }
}
```

```go
// Go: validação retornando erro
func NewMoney(cents int64, currency Currency) (Money, error) {
    if cents < 0 {
        return Money{}, fmt.Errorf("amount cannot be negative: %d", cents)
    }
    return Money{Cents: cents, Currency: currency}, nil
}
```

**Defensive Programming:**
- Validar TODOS os inputs na fronteira (constructors, public methods)
- Nunca confiar em dados externos (HTTP request, arquivos, config)
- Usar tipos para evitar estados inválidos (make illegal states unrepresentable)
- `null`/`nil` safety: usar `Optional<T>` (Java) ou zero values + error (Go)

| Técnica | Java 25 | Go 1.26 |
|---------|---------|---------|
| Null safety | `Optional<T>`, `Objects.requireNonNull` | Zero values, `nil` check |
| Validação | Record compact constructor | Factory function retornando `(T, error)` |
| Estados inválidos | `sealed interface` + `record` | Tipo específico por estado |
| Erros descritivos | Custom exceptions com contexto | `fmt.Errorf("context: %w", err)` |

**Critérios de aceite:**
- [ ] Todos os Value Objects validam no construtor/factory (fail fast)
- [ ] Zero possibilidade de criar `Money` negativo, `Transaction` sem ID, etc.
- [ ] Optional (Java) em todo retorno que pode ser ausente — zero `null`
- [ ] Go: todo erro tratado — zero `_` em `err` (exceto onde documentado)
- [ ] HTTP handlers validam request na entrada — mensagens de erro claras
- [ ] `VALIDATION-RULES.md` documentando todas as regras de validação
- [ ] Testes: inputs inválidos cobertos para todos os constructors/factories

---

### Desafio 6.5 — Separation of Concerns & Program to Interface

> *"Cada módulo trata de uma preocupação. Programe para interfaces, não implementações."*

**Cenário:** Auditoria final de separação de preocupações e programação para interfaces no Payment Processor.

**Separation of Concerns — Checklist:**

| Concern | Deve estar em | Não deve estar em |
|---------|--------------|-------------------|
| Regras de negócio | Domain Layer | Controller, Repository |
| Validação de entrada | Controller / boundary | Domain model |
| Persistência | Repository adapter | Use case, domain |
| Serialização/parsing | Adapter / View | Domain model |
| Logging | Cross-cutting (decorator) | Dentro de regras de negócio |
| Error handling | Cada camada trata o seu nível | Catch-all genérico |

**Program to Interface:**
- Todas as dependências injetadas como interfaces, não implementações concretas
- Variáveis declaradas com tipo interface quando possível
- Constructors recebem interfaces — facilitam teste e troca

**Tarefas:**
1. Auditar cada classe/struct: tem uma única responsabilidade?
2. Verificar: todos os constructors usam interfaces para dependências?
3. Verificar: algum domain model faz I/O, logging, ou serialização?
4. Corrigir violações encontradas

**Critérios de aceite:**
- [ ] Auditoria de concerns: cada módulo com responsabilidade única
- [ ] Zero I/O direto no domain model
- [ ] Zero logging dentro de regras de negócio (usar decorator)
- [ ] Todas as dependências injetadas como interfaces
- [ ] Java: verificar que variáveis usam tipo interface onde aplicável
- [ ] Go: verificar que interfaces são pequenas (1-3 métodos) e usadas no consumer
- [ ] `CONCERNS-AUDIT.md` com resultado da auditoria

---

### Desafio 6.6 — Anti-Patterns: Detection & Refactoring

> *"Reconhecer o que NÃO fazer é tão importante quanto saber o que fazer."*

**Cenário:** Identifique e corrija anti-patterns que possam ter sido inadvertidamente introduzidos.

**Anti-patterns a buscar:**

| Anti-pattern | Sintoma | Correção |
|--------------|---------|----------|
| **God Object** | Classe com 10+ métodos e múltiplas responsabilidades | Extrair classes com SRP |
| **Anemic Domain Model** | Entidades com apenas getters/setters, lógica em services | Mover comportamento para entidades |
| **Golden Hammer** | Mesmo padrão usado em todo lugar | Usar padrão adequado ao problema |
| **Premature Abstraction** | Interface com 1 implementador sem expectativa de expansão | Remover abstração, inline |
| **Cargo Cult** | Padrão aplicado sem entender o porquê | Documentar ou remover |
| **Speculative Generality** | Extensões "para o futuro" | YAGNI: remover |
| **Feature Envy** | Método que usa mais dados de outra classe que da própria | Mover método |
| **Shotgun Surgery** | Mudança exige alterar N arquivos | Consolidar responsabilidade |

**Tarefas:**
1. Executar Code Review Checklist do `06-best-practices.md` em todo o código
2. Identificar cada anti-pattern com localização precisa
3. Propor e executar correção
4. Documentar lição aprendida

**Critérios de aceite:**
- [ ] Code Review Checklist executado em todos os módulos
- [ ] ≥ 5 anti-patterns identificados com localização
- [ ] Cada anti-pattern corrigido com before/after
- [ ] `ANTI-PATTERNS-LOG.md` com descobertas, análises e correções
- [ ] Nenhum Anemic Domain Model: entidades têm comportamento real
- [ ] Nenhum God Object: nenhuma classe com > 200 linhas (guideline)

---

### Desafio 6.7 — Integração: Quality Gate Final

**Cenário:** Execute uma auditoria completa de qualidade no Payment Processor, verificando todas as boas práticas dos Levels 0-6.

**Quality Gate Checklist:**

| # | Critério | Referência |
|---|---------|------------|
| 1 | SOLID respeitado em todas as classes/structs | Level 1 |
| 2 | Padrões GoF aplicados corretamente (não forçados) | Levels 2-4 |
| 3 | Arquitetura com Regra de Dependência | Level 5 |
| 4 | Composition over Inheritance | 6.1 |
| 5 | DRY: zero duplicação de conhecimento | 6.2 |
| 6 | KISS: sem complexidade desnecessária | 6.2 |
| 7 | YAGNI: sem código especulativo | 6.2 |
| 8 | Law of Demeter: sem train wrecks | 6.3 |
| 9 | Tell Don't Ask: comportamento nos objetos | 6.3 |
| 10 | Fail Fast: validação em todas as fronteiras | 6.4 |
| 11 | Program to Interface: dependências como interfaces | 6.5 |
| 12 | Zero anti-patterns | 6.6 |
| 13 | Cobertura de testes ≥ 85% | Global |
| 14 | Documentação: DECISIONS + PATTERNS + ARCHITECTURE | Global |

**Entregáveis da auditoria:**
- `QUALITY-REPORT.md` com resultado de cada critério (PASS/FAIL/PARTIAL)
- Plano de ação para cada FAIL/PARTIAL
- Execução do plano de ação
- Segundo `QUALITY-REPORT.md` após correções

**Critérios de aceite:**
- [ ] 14/14 critérios PASS no Quality Gate
- [ ] `QUALITY-REPORT.md` com análise detalhada
- [ ] Métricas de código: complexidade ciclomática, fan-in/fan-out documentados
- [ ] Todos os testes passando (unitários + integração)
- [ ] Cobertura ≥ 85%
- [ ] Documentação completa e atualizada

---

## Entregáveis

| Artefato | Prática |
|----------|---------|
| `REFACTORING-LOG.md` | Composition over Inheritance |
| `SIMPLIFICATION-LOG.md` | DRY, KISS, YAGNI |
| Refatorações Law of Demeter + Tell Don't Ask | Clean Code |
| `VALIDATION-RULES.md` | Fail Fast & Defensive Programming |
| `CONCERNS-AUDIT.md` | Separation of Concerns |
| `ANTI-PATTERNS-LOG.md` | Anti-patterns Detection |
| `QUALITY-REPORT.md` | Quality Gate Final |

---

## Próximo Nível

→ [Level 7 — Projeto Capstone](07-capstone-project.md)
