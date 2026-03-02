# Level 4 — Padrões Comportamentais (GoF)

> **Objetivo:** Implementar os 10 padrões comportamentais do GoF no domínio Payment Processor,
> organizando **algoritmos**, **responsabilidades** e **comunicação** entre objetos.

**Referência:** [04-behavioral-patterns.md](../../../.docs/DESIGN-PATTERNS/04-behavioral-patterns.md)

---

## Pré-requisito

[Level 3 — Padrões Estruturais](03-structural-patterns.md) completo.

---

## Desafios

### Desafio 4.1 — Strategy: Fee Calculation & Fraud Detection

> *"Definir uma família de algoritmos, encapsulá-los e torná-los intercambiáveis."*

**Cenário:** Evolua o `FeeStrategy` do Level 1 e adicione `FraudDetectionStrategy`. O sistema precisa de múltiplas estratégias de detecção de fraude que podem ser trocadas em runtime baseado no nível de risco do cliente.

**Estratégias de fraude a implementar:**

| Strategy | Lógica | Risco |
|----------|--------|-------|
| `BasicFraudCheck` | Verifica valor máximo e velocity (transações/hora) | Baixo |
| `MLFraudCheck` | Simulação de score ML (baseado em features como valor, horário, localização) | Médio |
| `ManualReviewCheck` | Marca para revisão manual se score acima do threshold | Alto |
| `CompositeFraudCheck` | Executa múltiplas strategies em sequência (compõe strategies) | Configurável |

**Requisitos:**
- Interface `FraudDetectionStrategy` com `analyze(Transaction): FraudResult`
- `FraudResult` contém: score (0-100), decision (APPROVE/REVIEW/REJECT), reasons
- Strategies intercambiáveis em runtime por perfil de cliente
- `CompositeFraudCheck` combina N strategies (Strategy + Composite)

**Java 25 — alternativa funcional:**
```java
// Strategy pode ser uma Function em Java moderno
@FunctionalInterface
public interface FraudDetectionStrategy {
    FraudResult analyze(Transaction tx);
}

// Lambda como strategy
FraudDetectionStrategy basic = tx -> {
    if (tx.amount().isGreaterThan(Money.of(10000_00, BRL))) {
        return FraudResult.reject("Valor acima do limite");
    }
    return FraudResult.approve();
};
```

**Go 1.26 — strategy como função:**
```go
type FraudDetectionStrategy func(tx *Transaction) FraudResult

// Strategy como função simples
var BasicFraudCheck FraudDetectionStrategy = func(tx *Transaction) FraudResult {
    if tx.Amount.GreaterThan(NewMoney(1000000, BRL)) {
        return FraudResult{Decision: Reject, Reason: "Valor acima do limite"}
    }
    return FraudResult{Decision: Approve}
}
```

**Critérios de aceite:**
- [ ] Interface `FraudDetectionStrategy` com 4 implementações
- [ ] `CompositeFraudCheck` combina múltiplas strategies
- [ ] Strategies trocáveis em runtime sem alterar consumidor
- [ ] Java: demonstrar strategy como interface E como `@FunctionalInterface` / lambda
- [ ] Go: demonstrar strategy como interface E como `func` type
- [ ] Testes: cada strategy + composite + troca em runtime

---

### Desafio 4.2 — Observer: Transaction Event Notifications

> *"Definir dependência um-para-muitos para notificação automática de mudanças."*

**Cenário:** Quando o estado de uma transação muda, múltiplos sistemas devem ser notificados: envio de e-mail ao cliente, webhook para o merchant, log no audit trail, atualização de métricas, disparo de regras de compliance. O `Transaction` (Subject) não deve conhecer nenhum observer concreto.

**Requisitos:**
- Interface `TransactionObserver` com `onStatusChange(event: TransactionEvent)`
- `TransactionEvent` contém: transactionId, previousStatus, newStatus, timestamp, metadata
- Subject: `Transaction` ou `TransactionEventPublisher` separado (decisão a documentar)
- Observers concretos: `EmailNotifier`, `WebhookNotifier`, `AuditLogger`, `MetricsCollector`, `ComplianceChecker`
- Suporte a attach/detach dinâmico de observers

**Variantes a implementar:**

| Variante | Descrição |
|----------|-----------|
| **Push** | Evento contém todos os dados necessários |
| **Pull** | Evento contém apenas ID; observer busca os dados que precisa |
| **Event Bus** | Publisher e observers desacoplados via mediador (Event Bus in-memory) |

**Critérios de aceite:**
- [ ] Interface `TransactionObserver` + 5 observers concretos
- [ ] `TransactionEvent` como objeto imutável com dados da mudança
- [ ] Attach/detach dinâmico de observers
- [ ] Variante Push implementada (evento completo)
- [ ] Variante Event Bus implementada (com `InMemoryEventBus`)
- [ ] Observer não remove → memory leak documentado e mitigado
- [ ] Testes: múltiplos observers notificados, attach/detach, order não garantida

---

### Desafio 4.3 — Command: Payment Operations with Undo/Redo

> *"Encapsular uma requisição como um objeto, permitindo undo/redo e enfileiramento."*

**Cenário:** Cada operação de pagamento (charge, refund, void, capture) é um Command que pode ser enfileirado, logado, e em alguns casos desfeito. O sistema mantém um histórico de commands executados.

**Commands a implementar:**

| Command | Execute | Undo |
|---------|---------|------|
| `ChargeCommand` | Processa cobrança | Cancela/estorna |
| `RefundCommand` | Processa estorno | Re-captura (se permitido) |
| `VoidCommand` | Anula transação autorizada | Re-autoriza |
| `CaptureCommand` | Captura transação autorizada | Libera autorização |

**Requisitos:**
- Interface `PaymentCommand` com `execute()`, `undo()`, `getDescription()`
- `CommandHistory` que armazena commands executados
- `CommandInvoker` que executa e gerencia undo/redo
- Suporte a undo sequencial (último executado é o primeiro desfeito)
- Log de todas as operações com timestamp

**Estrutura:**

```
Client → CommandInvoker.execute(new ChargeCommand(tx, gateway))
                │
                ├── command.execute() → processa
                ├── history.push(command) → salva para undo
                └── log(command.description + timestamp)

Client → CommandInvoker.undo()
                │
                ├── command = history.pop()
                └── command.undo() → desfaz
```

**Critérios de aceite:**
- [ ] Interface `PaymentCommand` + 4 commands concretos
- [ ] Cada command encapsula transação + gateway + dados necessários
- [ ] `execute()` processa e modifica estado da transação
- [ ] `undo()` reverte a operação (nem todo command suporta — documentar)
- [ ] `CommandHistory` com stack de commands executados
- [ ] `CommandInvoker` com `execute(cmd)`, `undo()`, `redo()`
- [ ] Testes: execute, undo, redo, undo de command não-reversível → erro

---

### Desafio 4.4 — Template Method: Payment Processing Flow

> *"Definir o esqueleto de um algoritmo, delegando passos específicos para subclasses."*

**Cenário:** Todo processamento de pagamento segue o mesmo fluxo: validate → calculate fees → authorize → capture → notify. Porém, cada tipo de pagamento tem passos específicos. Cartão de crédito precisa de autorização no gateway; PIX é instantâneo; Boleto gera PDF de cobrança.

**Requisitos:**
- Classe abstrata `PaymentProcessingTemplate` com o template method `process(tx)`
- Passos fixos (não sobrescrevíveis): `validate()`, `logStart()`, `logEnd()`
- Passos abstratos (obrigatórios): `authorize(tx)`, `capture(tx)`
- Hooks opcionais: `beforeAuthorize(tx)`, `afterCapture(tx)`, `onError(tx, error)`
- Implementações: `CreditCardProcessing`, `PixProcessing`, `BankSlipProcessing`

**Template Method:**
```
process(tx):
    logStart(tx)                    ← fixo
    validate(tx)                    ← fixo
    fees = calculateFees(tx)        ← fixo (usa FeeStrategy internamente)
    beforeAuthorize(tx)             ← hook (opcional)
    result = authorize(tx)          ← abstrato — subclasse implementa
    capture(tx, result)             ← abstrato — subclasse implementa
    afterCapture(tx)                ← hook (opcional)
    notify(tx, result)              ← fixo
    logEnd(tx, result)              ← fixo
```

| Implementação | authorize() | capture() | Hooks usados |
|---------------|-------------|-----------|--------------|
| `CreditCardProcessing` | Envia ao gateway | Captura no gateway | `beforeAuthorize`: verifica 3DS |
| `PixProcessing` | Gera QR code | Transferência instantânea | `afterCapture`: confirma em 5s |
| `BankSlipProcessing` | Gera boleto PDF | Aguarda compensação | `onError`: notifica falha |

**Template Method vs Strategy (documentar):**

| Aspecto | Template Method (4.4) | Strategy (4.1) |
|---------|----------------------|-----------------|
| **Mecanismo** | Herança | Composição |
| **Variação** | Passos do algoritmo | Algoritmo inteiro |
| **Quando usar** | Estrutura fixa, passos variam | Algoritmo inteiro varia |

**Critérios de aceite:**
- [ ] Classe abstrata com template method `process()` marcado como `final` (Java) ou documentado (Go)
- [ ] 3 implementações concretas com lógica específica nos passos abstratos
- [ ] Hooks opcionais com implementação default (no-op)
- [ ] Java: `abstract class` com `final` template method + `abstract` passos + `protected` hooks
- [ ] Go: struct com campos de função para hooks + composição (Go não tem abstract classes)
- [ ] `DECISIONS.md`: trade-off Template Method vs Strategy para este caso
- [ ] Testes: cada implementação + hooks chamados corretamente + fluxo completo

---

### Desafio 4.5 — State: Transaction Lifecycle

> *"Permitir que um objeto altere seu comportamento quando seu estado interno muda."*

**Cenário:** Uma transação passa por vários estados (CREATED → AUTHORIZED → CAPTURED → SETTLED → REFUNDED). Em cada estado, o comportamento é diferente: em CREATED pode ser cancelada ou autorizada; em AUTHORIZED pode ser capturada ou cancelada; em CAPTURED pode ser estornada; em SETTLED é final. Elimine os `if/switch` sobre status.

**Requisitos:**
- Interface `TransactionState` com métodos: `authorize()`, `capture()`, `settle()`, `cancel()`, `refund()`
- States concretos: `CreatedState`, `AuthorizedState`, `CapturedState`, `SettledState`, `CancelledState`, `RefundedState`, `FailedState`
- Cada state sabe quais transições são permitidas — operações inválidas retornam erro
- O state faz a transição (muda o state do context)

**Máquina de estados:**
```
CREATED ──authorize──▶ AUTHORIZED ──capture──▶ CAPTURED ──settle──▶ SETTLED
  │                      │                      │
  └──cancel──▶ CANCELLED  └──cancel──▶ CANCELLED  └──refund──▶ REFUNDED
  │                                               
  └──fail──▶ FAILED                               
```

| State | authorize | capture | settle | cancel | refund |
|-------|-----------|---------|--------|--------|--------|
| `Created` | → Authorized | ERROR | ERROR | → Cancelled | ERROR |
| `Authorized` | ERROR | → Captured | ERROR | → Cancelled | ERROR |
| `Captured` | ERROR | ERROR | → Settled | ERROR | → Refunded |
| `Settled` | ERROR | ERROR | ERROR | ERROR | ERROR (final) |
| `Cancelled` | ERROR | ERROR | ERROR | ERROR | ERROR (final) |
| `Refunded` | ERROR | ERROR | ERROR | ERROR | ERROR (final) |

**Critérios de aceite:**
- [ ] Interface `TransactionState` + 7 states concretos
- [ ] Zero `if/switch` sobre status no `Transaction`
- [ ] Cada state retorna erro para operações inválidas
- [ ] Transições fazem `context.setState(nextState)` automaticamente
- [ ] Observer notificado em cada transição de estado (integra com 4.2)
- [ ] Java: `sealed interface` para states (ou classes sem sealed)
- [ ] Go: interface + structs por state
- [ ] Testes: todas as transições válidas + todas as inválidas

---

### Desafio 4.6 — Chain of Responsibility: Validation Pipeline

> *"Dar a mais de um objeto a chance de tratar uma requisição."*

**Cenário:** Antes de processar um pagamento, múltiplas validações devem ser executadas em sequência: validação de dados, verificação de fraude, verificação de limites, verificação de saldo, compliance check. Cada handler pode aprovar e passar adiante ou rejeitar.

**Handlers a implementar:**

| Handler | Responsabilidade | Rejeita se |
|---------|------------------|------------|
| `DataValidationHandler` | Verifica campos obrigatórios | Dados faltando ou inválidos |
| `FraudCheckHandler` | Verifica score de fraude | Score acima do threshold |
| `AmountLimitHandler` | Verifica limites por tipo de pagamento | Acima do limite |
| `BalanceCheckHandler` | Verifica saldo do cliente (para débito) | Saldo insuficiente |
| `ComplianceHandler` | Verifica regras regulatórias | Não compliance |

**Variantes:**

| Variante | Comportamento |
|----------|---------------|
| **Puro** | Primeiro handler que trata interrompe a cadeia |
| **Pipeline** | Todos os handlers executam (coleta todos os erros) |
| **Middleware** | Cada handler decide se passa para `next()` |

**Requisitos:**
- Interface `ValidationHandler` com `handle(request)` e `setNext(handler)`
- Implementar variante **Pipeline** (todos executam, coleta erros)
- Implementar variante **Middleware** (handler decide se chama `next()`)
- Cadeia configurável em runtime (adicionar/remover handlers)

**Critérios de aceite:**
- [ ] Interface `ValidationHandler` + 5 handlers concretos
- [ ] Variante Pipeline: todos executam, resultado agrega todos os erros
- [ ] Variante Middleware: cada handler decide se passa adiante
- [ ] Cadeia configurável: builder para montar a cadeia de handlers
- [ ] Handler pode ser adicionado/removido sem alterar outros
- [ ] Testes: cadeia completa, handler que rejeita, handler adicionado dinamicamente

---

### Desafio 4.7 — Mediator: Payment Orchestration

> *"Definir um objeto que encapsula como um conjunto de objetos interage."*

**Cenário:** O processamento de pagamento envolve múltiplos componentes que precisam interagir: `PaymentProcessor`, `FraudDetector`, `GatewayRouter`, `NotificationService`, `AuditService`, `BalanceManager`. Sem Mediator, cada um conheceria todos os outros. O `PaymentMediator` centraliza a coordenação.

**Requisitos:**
- Interface `PaymentMediator` com `notify(sender, event)`
- `ConcretePaymentMediator` que conhece e coordena todos os componentes
- Componentes (Colleagues) conhecem apenas o Mediator
- Fluxo: Processor notifica Mediator → Mediator coordena FraudDetector, Gateway, Notifier, etc.

**Fluxo mediado:**
```
PaymentProcessor.process(tx)
  └── mediator.notify(this, "PAYMENT_INITIATED", tx)
        ├── FraudDetector.analyze(tx) → resultado
        ├── if approved: GatewayRouter.route(tx) → gateway
        ├── gateway.process(tx) → response
        ├── BalanceManager.update(tx)
        ├── AuditService.log(tx, response)
        └── NotificationService.notify(tx, response)
```

**Mediator vs Observer (documentar):**

| Aspecto | Mediator (4.7) | Observer (4.2) |
|---------|---------------|----------------|
| **Direção** | Bidirecional, centralizada | Unidirecional, broadcast |
| **Conhecimento** | Mediator conhece todos | Subject não conhece detalhes |
| **Propósito** | Coordenar interação complexa | Notificar mudanças |

**Critérios de aceite:**
- [ ] Interface `PaymentMediator` + implementação concreta
- [ ] 5+ componentes (Colleagues) que se comunicam apenas via Mediator
- [ ] Zero referência direta entre Colleagues
- [ ] Fluxo completo: process → fraud → gateway → balance → audit → notify
- [ ] `DECISIONS.md`: Mediator vs Observer — quando usar cada um
- [ ] Testes: fluxo completo mediado, componente falha → mediator trata

---

### Desafio 4.8 — Iterator: Transaction History Traversal

> *"Prover acesso sequencial aos elementos sem expor a estrutura interna."*

**Cenário:** O histórico de transações pode ser armazenado de diversas formas (lista, árvore por data, hash por status). Implementar iteradores customizados que permitem percorrer transações com diferentes critérios sem expor a estrutura interna.

**Iterators a implementar:**

| Iterator | Comportamento |
|----------|---------------|
| `ChronologicalIterator` | Percorre por data (ascendente ou descendente) |
| `StatusFilterIterator` | Percorre apenas transações com status específico |
| `AmountRangeIterator` | Percorre apenas transações em um range de valor |
| `CompositeIterator` | Combina múltiplos filtros (AND logic) |

**Java 25:**
```java
// Implementar java.util.Iterator<Transaction>
public class StatusFilterIterator implements Iterator<Transaction> {
    // ... filtra por status enquanto itera
}

// Stream API como alternativa funcional
transactions.stream()
    .filter(tx -> tx.status() == CAPTURED)
    .filter(tx -> tx.amount().isGreaterThan(minAmount))
    .sorted(Comparator.comparing(Transaction::createdAt))
    .forEach(System.out::println);
```

**Go 1.26:**
```go
// Implementar padrão iterator idiomático em Go (iter package 1.23+)
func FilterByStatus(txs []*Transaction, status TransactionStatus) iter.Seq[*Transaction] {
    return func(yield func(*Transaction) bool) {
        for _, tx := range txs {
            if tx.Status == status {
                if !yield(tx) { return }
            }
        }
    }
}
```

**Critérios de aceite:**
- [ ] 4 iterators com critérios diferentes
- [ ] `CompositeIterator` combina filtros (chaining)
- [ ] Java: implementar `Iterator<T>` + comparar com `Stream API`
- [ ] Go: implementar `iter.Seq[T]` (Go 1.23+) / `for range` pattern
- [ ] Iteração não expõe estrutura interna do repositório
- [ ] Testes: cada iterator + composição + coleção vazia

---

### Desafio 4.9 — Memento: Transaction Snapshots

> *"Capturar e externalizar o estado interno de um objeto para restaurá-lo depois."*

**Cenário:** O sistema precisa suportar checkpoints de transação: antes de cada operação crítica (authorize, capture), um snapshot é salvo. Se a operação falha, o estado é restaurado ao último checkpoint. Útil para estorno parcial (restaurar ao estado pré-captura).

**Requisitos:**
- `TransactionMemento` — snapshot imutável do estado da transação
- `Transaction` (Originator) — `createMemento()` e `restoreFrom(memento)`
- `TransactionCheckpointManager` (Caretaker) — gerencia stack de mementos
- Mementos preservam: status, amount, history, metadata, timestamp
- Caretaker não acessa internals do Memento

**Critérios de aceite:**
- [ ] `TransactionMemento` imutável (record/struct sem setters)
- [ ] `Transaction.createMemento()` retorna snapshot do estado atual
- [ ] `Transaction.restoreFrom(memento)` restaura ao estado do snapshot
- [ ] `TransactionCheckpointManager` com `save(tx)` e `restore(tx)`
- [ ] Caretaker não conhece conteúdo do Memento (encapsulamento preservado)
- [ ] Integração com Command (4.3): `undo()` usa Memento para restaurar
- [ ] Testes: save, restore, múltiplos checkpoints, restore ao ponto correto

---

### Desafio 4.10 — Visitor: Transaction Reports & Analytics

> *"Representar uma operação sobre elementos de uma estrutura sem alterar as classes."*

**Cenário:** O sistema precisa gerar diferentes relatórios sobre transações sem adicionar métodos de relatório nas classes de transação: cálculo de impostos, reconciliação bancária, analytics de fraude, extrato formatado. Cada relatório é um Visitor.

**Visitors a implementar:**

| Visitor | Propósito | Saída |
|---------|-----------|-------|
| `TaxCalculatorVisitor` | Calcula impostos por tipo de transação | `TaxReport` |
| `ReconciliationVisitor` | Reconcilia com extrato bancário | `ReconciliationReport` |
| `FraudAnalyticsVisitor` | Analisa padrões de fraude | `FraudReport` |
| `StatementVisitor` | Gera extrato formatado | `Statement` (texto) |

**Requisitos:**
- Interface `TransactionVisitor` com `visit(CreditCardTransaction)`, `visit(PixTransaction)`, etc.
- Interface `TransactionElement` com `accept(visitor)`
- Double dispatch: `element.accept(visitor)` → `visitor.visit(this)`
- Visitantes operam sobre o histórico de transações (lista)

**Java 25 — alternativa com pattern matching:**
```java
// Visitor tradicional (double dispatch)
interface TransactionVisitor<R> {
    R visit(CreditCardTransaction tx);
    R visit(PixTransaction tx);
    R visit(BankSlipTransaction tx);
}

// Alternativa moderna: sealed + pattern matching (sem Visitor!)
sealed interface TypedTransaction permits CreditCardTransaction, PixTransaction, BankSlipTransaction {}

TaxReport report = switch (tx) {
    case CreditCardTransaction cc -> calculateCardTax(cc);
    case PixTransaction pix -> calculatePixTax(pix);
    case BankSlipTransaction bs -> calculateSlipTax(bs);
};
// Documentar trade-off: Visitor clássico vs pattern matching
```

**Critérios de aceite:**
- [ ] Interface `TransactionVisitor` + 4 visitors concretos
- [ ] Double dispatch implementado (`accept` → `visit`)
- [ ] Visitors operam sobre lista de transações
- [ ] Java: implementar Visitor clássico E alternativa com `sealed` + pattern matching
- [ ] Go: implementar Visitor E alternativa com type switch
- [ ] `DECISIONS.md`: quando usar Visitor clássico vs pattern matching/type switch
- [ ] Testes: cada visitor + aplicação sobre conjunto de transações misto

---

### Desafio 4.11 — Integração: Behavioral Composition

**Cenário final:** Combine todos os 10 padrões comportamentais no fluxo de processamento:

```
1. TransactionBuilder (Builder L2) constrói a transação
2. ValidationPipeline (Chain) valida em cadeia
3. FraudDetectionStrategy (Strategy) analisa fraude
4. TransactionState (State) gerencia ciclo de vida
5. PaymentCommand (Command) encapsula operação com undo
6. PaymentProcessingTemplate (Template Method) executa fluxo
7. PaymentMediator (Mediator) coordena componentes
8. TransactionObservers (Observer) notificam interessados
9. TransactionCheckpoint (Memento) salva snapshots
10. TransactionReports (Visitor) geram analytics
11. TransactionIterators (Iterator) percorrem histórico
```

**Critérios de aceite:**
- [ ] Todos os 10 padrões integrados em um fluxo coeso
- [ ] `PATTERNS.md` com mapa completo de cada padrão no fluxo
- [ ] Testes end-to-end: charge com all validations, state transitions, notifications, undo, reports
- [ ] Cobertura ≥ 85%

---

## Entregáveis

| Artefato | Padrão |
|----------|--------|
| `FraudDetectionStrategy` + 4 strategies | Strategy |
| `TransactionObserver` + 5 observers + Event Bus | Observer |
| `PaymentCommand` + 4 commands + History + Invoker | Command |
| `PaymentProcessingTemplate` + 3 implementações | Template Method |
| `TransactionState` + 7 states | State |
| `ValidationHandler` + 5 handlers + Pipeline/Middleware | Chain of Responsibility |
| `PaymentMediator` + 5 colleagues | Mediator |
| 4 iterators + Composite Iterator | Iterator |
| `TransactionMemento` + Checkpoint Manager | Memento |
| `TransactionVisitor` + 4 visitors | Visitor |
| Fluxo integrado | Todos os 10 padrões |

---

## Próximo Nível

→ [Level 5 — Padrões Arquiteturais](05-architectural-patterns.md)
