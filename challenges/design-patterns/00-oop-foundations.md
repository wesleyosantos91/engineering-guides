# Level 0 — Fundações OOP & Linguagem

> **Objetivo:** Dominar os fundamentos de Orientação a Objetos e os recursos de linguagem de Java 25 e Go 1.26
> que serão base para todos os padrões de design nos próximos níveis.

**Referência:** Conceitos fundamentais (pré-requisito para [01-solid-principles.md](../../../.docs/DESIGN-PATTERNS/01-solid-principles.md))

---

## Contexto do Domínio

Neste nível, você criará as **entidades base** do sistema Payment Processor:

- `Money` — Value Object para representar valores monetários com moeda e precisão
- `Currency` — Enumeração de moedas suportadas (BRL, USD, EUR)
- `PaymentMethod` — Interface/contrato para métodos de pagamento
- `CreditCard`, `DebitCard`, `Pix`, `BankSlip` — Implementações concretas
- `Transaction` — Entidade principal que registra uma operação de pagamento
- `TransactionStatus` — Enumeração de estados possíveis
- `Customer` — Entidade do cliente que inicia pagamentos
- `PaymentResult` — Objeto de resultado do processamento

---

## Desafios

### Desafio 0.1 — Value Object `Money` (Imutabilidade & Encapsulamento)

Implemente um Value Object `Money` que represente valores monetários de forma segura.

**Requisitos:**
- Armazenar `amount` (precisão decimal) e `currency`
- Ser **imutável** — operações retornam novas instâncias
- Implementar operações: `add(Money)`, `subtract(Money)`, `multiply(factor)`, `percentage(rate)`
- Lançar erro ao operar com moedas diferentes
- Implementar `equals`/`hashCode` baseado em valor (não em referência)
- Implementar `toString()` com formatação (ex: `"R$ 150,00"`, `"$ 150.00"`)

**Java 25:**
```java
// Usar record para imutabilidade automática
public record Money(BigDecimal amount, Currency currency) {
    // Validação no compact constructor
    // Operações retornam novos records
}
```

**Go 1.26:**
```go
// Struct com campos não-exportados + factory function
type Money struct {
    amount   int64    // centavos para evitar floating point
    currency Currency
}

func NewMoney(amount int64, currency Currency) Money { ... }
func (m Money) Add(other Money) (Money, error) { ... }
```

**Critérios de aceite:**
- [ ] Value Object imutável em ambas as linguagens
- [ ] Operações algébricas com validação de moeda
- [ ] Comparação por valor (`equals` em Java, comparação de struct em Go)
- [ ] Sem floating point — usar `BigDecimal` (Java) ou centavos `int64` (Go)
- [ ] Testes cobrindo: criação válida, operações, moedas incompatíveis, edge cases (zero, negativo)

---

### Desafio 0.2 — Interfaces & Polimorfismo (`PaymentMethod`)

Defina a abstração `PaymentMethod` e implemente variantes concretas.

**Requisitos:**
- Interface `PaymentMethod` com métodos: `type()`, `validate()`, `maskedIdentifier()`
- Implementações: `CreditCard`, `DebitCard`, `Pix`, `BankSlip`
- Cada implementação tem dados específicos (número do cartão, chave PIX, etc.)
- `validate()` verifica integridade dos dados (Luhn para cartão, formato de chave PIX, etc.)
- `maskedIdentifier()` retorna representação segura (ex: `"**** **** **** 1234"`)

**Java 25:**
```java
// Sealed interface para controle de implementações
public sealed interface PaymentMethod
    permits CreditCard, DebitCard, Pix, BankSlip {
    
    PaymentType type();
    ValidationResult validate();
    String maskedIdentifier();
}

// Cada implementação é um record
public record CreditCard(String number, String holder, YearMonth expiry, String cvv)
    implements PaymentMethod { ... }
```

**Go 1.26:**
```go
type PaymentMethod interface {
    Type() PaymentType
    Validate() error
    MaskedIdentifier() string
}

type CreditCard struct {
    Number string
    Holder string
    Expiry time.Time
    CVV    string
}

func (c CreditCard) Type() PaymentType          { return CreditCardType }
func (c CreditCard) Validate() error            { ... }
func (c CreditCard) MaskedIdentifier() string   { ... }
```

**Critérios de aceite:**
- [ ] Interface definida com contrato claro
- [ ] 4 implementações concretas com dados específicos
- [ ] Java: usar `sealed interface` + `record`
- [ ] Go: interface implícita satisfeita por structs
- [ ] Validação de dados em cada implementação (Luhn, formato PIX, etc.)
- [ ] Mascaramento de dados sensíveis
- [ ] Testes unitários para cada implementação + testes polimórficos via interface

---

### Desafio 0.3 — Enumerações & Type Safety (`TransactionStatus`, `Currency`, `PaymentType`)

Crie enumerações seguras e ricas em comportamento.

**Requisitos:**
- `Currency` — `BRL`, `USD`, `EUR` com símbolo, casas decimais, locale
- `PaymentType` — `CREDIT_CARD`, `DEBIT_CARD`, `PIX`, `BANK_SLIP`
- `TransactionStatus` — `CREATED`, `AUTHORIZED`, `CAPTURED`, `SETTLED`, `CANCELLED`, `REFUNDED`, `FAILED`
- `TransactionStatus` deve validar transições permitidas (máquina de estados básica)

**Java 25:**
```java
public enum TransactionStatus {
    CREATED(Set.of(AUTHORIZED, CANCELLED, FAILED)),
    AUTHORIZED(Set.of(CAPTURED, CANCELLED)),
    CAPTURED(Set.of(SETTLED, REFUNDED)),
    // ...
    ;
    private final Set<TransactionStatus> allowedTransitions;
    
    public boolean canTransitionTo(TransactionStatus next) {
        return allowedTransitions.contains(next);
    }
}
```

**Go 1.26:**
```go
type TransactionStatus int

const (
    StatusCreated TransactionStatus = iota
    StatusAuthorized
    StatusCaptured
    // ...
)

var allowedTransitions = map[TransactionStatus][]TransactionStatus{
    StatusCreated: {StatusAuthorized, StatusCancelled, StatusFailed},
    // ...
}

func (s TransactionStatus) CanTransitionTo(next TransactionStatus) bool { ... }
func (s TransactionStatus) String() string { ... }
```

**Critérios de aceite:**
- [ ] Enums type-safe em ambas as linguagens
- [ ] `Currency` com metadados (símbolo, casas decimais)
- [ ] `TransactionStatus` com validação de transições
- [ ] Java: usar `enum` com campos e métodos
- [ ] Go: usar `const` + `iota` + método `String()` (satisfazer `fmt.Stringer`)
- [ ] Testes: transições válidas e inválidas, conversão para string

---

### Desafio 0.4 — Composição & Entidades (`Transaction`, `Customer`)

Componha as entities anteriores em objetos mais complexos.

**Requisitos:**
- `Customer` — id, name, email, document, createdAt
- `Transaction` — id, customer, paymentMethod, amount (Money), status, timestamps, metadata
- `Transaction` deve ser criada no estado `CREATED`
- Transições de estado via método que valida a transição
- Histórico de mudanças de status (lista de eventos: status + timestamp + reason)
- Geração de ID com UUID

**Java 25:**
```java
public final class Transaction {
    private final UUID id;
    private final Customer customer;
    private final PaymentMethod paymentMethod;
    private final Money amount;
    private TransactionStatus status;
    private final List<StatusChange> history;
    // ...
    
    public void transitionTo(TransactionStatus newStatus, String reason) {
        if (!status.canTransitionTo(newStatus)) {
            throw new InvalidTransitionException(status, newStatus);
        }
        history.add(new StatusChange(status, newStatus, Instant.now(), reason));
        status = newStatus;
    }
}
```

**Go 1.26:**
```go
type Transaction struct {
    ID            uuid.UUID
    Customer      Customer
    PaymentMethod PaymentMethod
    Amount        Money
    Status        TransactionStatus
    History       []StatusChange
    CreatedAt     time.Time
    UpdatedAt     time.Time
}

func NewTransaction(customer Customer, method PaymentMethod, amount Money) *Transaction { ... }
func (t *Transaction) TransitionTo(newStatus TransactionStatus, reason string) error { ... }
```

**Critérios de aceite:**
- [ ] `Transaction` criada sempre no estado `CREATED`
- [ ] Transições de estado validadas — estados inválidos retornam erro
- [ ] Histórico de mudanças de status com timestamp e razão
- [ ] Composição de Value Objects (`Money`, `PaymentMethod`) e entidades (`Customer`)
- [ ] Java: encapsulamento com campos `final` + lista de histórico copiada defensivamente
- [ ] Go: factory function `NewTransaction()` + método pointer receiver `TransitionTo()`
- [ ] UUID gerado automaticamente na criação
- [ ] Testes: criação, transições válidas, transições inválidas, histórico

---

### Desafio 0.5 — Collections & Iteração (`TransactionRepository` in-memory)

Implemente um repositório in-memory usando coleções da standard library.

**Requisitos:**
- Interface `TransactionRepository` com: `save`, `findByID`, `findByCustomer`, `findByStatus`, `findAll`
- Implementação `InMemoryTransactionRepository` usando Map (Java) / map (Go)
- Suportar filtragem por múltiplos critérios (status + data range + valor mínimo)
- Paginação simples: `offset` + `limit`
- Ordenação por `createdAt` (ascendente/descendente)

**Java 25:**
```java
public interface TransactionRepository {
    void save(Transaction tx);
    Optional<Transaction> findById(UUID id);
    List<Transaction> findByCustomer(UUID customerId);
    List<Transaction> findByStatus(TransactionStatus status);
    Page<Transaction> findAll(TransactionQuery query);
}

// Uso de Stream API para filtragem + paginação
public class InMemoryTransactionRepository implements TransactionRepository {
    private final Map<UUID, Transaction> store = new ConcurrentHashMap<>();
    // ...
}
```

**Go 1.26:**
```go
type TransactionRepository interface {
    Save(tx *Transaction) error
    FindByID(id uuid.UUID) (*Transaction, error)
    FindByCustomer(customerID uuid.UUID) ([]*Transaction, error)
    FindByStatus(status TransactionStatus) ([]*Transaction, error)
    FindAll(query TransactionQuery) (*Page[*Transaction], error)
}

type InMemoryTransactionRepository struct {
    mu    sync.RWMutex
    store map[uuid.UUID]*Transaction
}
```

**Critérios de aceite:**
- [ ] Interface de repositório definida com contrato claro
- [ ] Implementação in-memory com thread-safety
- [ ] Java: `ConcurrentHashMap` + `Stream API` para queries
- [ ] Go: `sync.RWMutex` + iteração com filtros
- [ ] Paginação com `Page<T>` / `Page[T]` (offset, limit, total, items)
- [ ] Filtragem combinada (status AND date range AND min amount)
- [ ] Ordenação ascendente/descendente por data
- [ ] Testes: CRUD, queries com filtros, paginação, concorrência

---

### Desafio 0.6 — Error Handling & Tipos de Erro

Implemente um sistema de erros tipados e expressivos.

**Requisitos:**
- Hierarquia de erros do domínio: `PaymentError` (base), `ValidationError`, `InsufficientFundsError`, `InvalidTransitionError`, `GatewayError`, `NotFoundError`
- Cada erro deve conter: código, mensagem, causa (wrapped), timestamp
- Erros devem ser identificáveis por tipo (para decisão de retry, logging, etc.)

**Java 25:**
```java
// Sealed hierarchy para erros de domínio
public sealed abstract class PaymentError extends RuntimeException
    permits ValidationError, InsufficientFundsError, InvalidTransitionError,
            GatewayError, NotFoundError {
    
    private final String code;
    private final Instant timestamp;
    // ...
}

// Pattern matching para tratar erros
switch (error) {
    case ValidationError v -> handleValidation(v);
    case GatewayError g when g.isRetryable() -> retry(g);
    case GatewayError g -> fail(g);
    default -> log(error);
}
```

**Go 1.26:**
```go
type PaymentError struct {
    Code      string
    Message   string
    Cause     error
    Timestamp time.Time
}

func (e *PaymentError) Error() string { ... }
func (e *PaymentError) Unwrap() error { ... }

// Sentinel errors + custom types
var ErrNotFound = &PaymentError{Code: "NOT_FOUND", ...}

type ValidationError struct {
    PaymentError
    Field   string
    Details []string
}

// errors.Is / errors.As para identificar tipo
```

**Critérios de aceite:**
- [ ] Hierarquia de erros de domínio clara e tipada
- [ ] Java: `sealed` hierarchy + pattern matching no `switch`
- [ ] Go: custom error types + `errors.Is` / `errors.As` + `Unwrap()` chain
- [ ] Cada erro com código, mensagem e timestamp
- [ ] Erros wrappable (causa encadeada)
- [ ] Distinção entre erros retryable e non-retryable
- [ ] Testes: criação, wrapping, unwrapping, identificação por tipo

---

## Entregáveis

| Artefato | Descrição |
|----------|-----------|
| `Money` | Value Object imutável com operações algébricas |
| `Currency`, `PaymentType`, `TransactionStatus` | Enumerações type-safe com comportamento |
| `PaymentMethod` + 4 implementações | Interface + polimorfismo concreto |
| `Transaction`, `Customer` | Entidades compostas com validação de estado |
| `TransactionRepository` + `InMemoryTransactionRepository` | Interface + implementação com queries |
| Hierarquia de erros | Erros tipados e expressivos |
| Testes | ≥ 85% cobertura em ambas as linguagens |

---

## O que você vai exercitar

| Conceito | Java 25 | Go 1.26 |
|----------|---------|---------|
| **Imutabilidade** | `record`, `final`, defensive copies | Retorno por valor, sem ponteiros |
| **Interfaces** | `sealed interface` com tipo controlado | Interface implícita (duck typing) |
| **Polimorfismo** | Herança + interface, pattern matching | Interface + type switch |
| **Composição** | Campos de objetos, delegação | Struct embedding |
| **Encapsulamento** | `private` fields, API pública mínima | Campos não exportados (lowercase) |
| **Collections** | `List`, `Map`, `Stream`, `Optional` | Slices, maps, multiple returns |
| **Error handling** | Sealed exceptions, pattern matching | `error` interface, `errors.Is/As` |
| **Concorrência básica** | `ConcurrentHashMap` | `sync.RWMutex`, `sync.Map` |
| **Testes** | JUnit 5, AssertJ | `testing`, table-driven tests |

---

## Próximo Nível

Quando completar todos os desafios deste nível, avance para:
→ [Level 1 — Princípios SOLID](01-solid-principles.md)

Os tipos criados aqui (`Money`, `PaymentMethod`, `Transaction`, `TransactionRepository`, hierarquia de erros) serão a base para refatoração e aplicação de padrões nos próximos níveis.
