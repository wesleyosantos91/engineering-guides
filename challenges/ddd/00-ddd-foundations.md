# Level 0 — Fundações DDD & Linguagem Ubíqua

> **Objetivo:** Construir os alicerces conceituais de Domain-Driven Design e criar os Value Objects
> e tipos fundamentais do domínio MerchantHub, estabelecendo a Ubiquitous Language desde o início.

**Referência:** [README.md (DDD Overview)](../../.docs/ddd/README.md) · [01-strategic-design.md](../../.docs/ddd/01-strategic-design.md) (seção Ubiquitous Language)

---

## Contexto do Domínio

Neste nível, você criará o **vocabulário base** e os **tipos fundamentais** do MerchantHub — a plataforma de e-commerce multi-tenant. O foco é entender que DDD começa pela **linguagem**, não pelo código. Antes de implementar qualquer entidade, você precisa de Value Objects sólidos e um glossário claro.

Tipos a construir:
- `Money` — Value Object para valores monetários com moeda e precisão
- `Currency` — Enumeração de moedas suportadas
- `EmailAddress`, `CPF`, `PhoneNumber` — Value Objects tipados (cura para Primitive Obsession)
- `Address` — Value Object composto
- `Quantity`, `SKU`, `Percentage` — Value Objects do domínio de e-commerce
- `DateRange` — Value Object temporal para promoções, garantias, etc.
- Glossário da Ubiquitous Language por Bounded Context

---

## Desafios

### Desafio 0.1 — Glossário da Ubiquitous Language

Antes de qualquer código, crie o documento `GLOSSARY.md` que define a Ubiquitous Language do MerchantHub.

**Requisitos:**
- Definir termos para **cada Bounded Context** separadamente
- Mostrar como o **mesmo conceito** tem significados diferentes em contextos diferentes
- Identificar sinônimos proibidos (termos que geram ambiguidade)
- Incluir termos de negócio, não técnicos

**Exemplo de estrutura:**

```
## Contexto: Catalog
- **Product**: Item disponível para venda com nome, descrição, imagens e preço de lista
- **Category**: Agrupamento hierárquico de produtos por tipo
- **SKU**: Stock Keeping Unit — identificador único de uma variante de produto

## Contexto: Sales
- **Order**: Pedido confirmado com itens, endereço de entrega e forma de pagamento
- **Cart**: Carrinho de compras em andamento — não é um pedido até ser confirmado
- **Discount**: Redução de preço aplicada a um pedido por regra de negócio

## Contexto: Inventory
- **InventoryItem**: Registro de quantidade disponível de um produto em um warehouse
  (NÃO é um Product — contém apenas productId, quantidade e localização)

## Sinônimos proibidos
- Product ≠ InventoryItem ≠ ShippableItem (cada contexto tem seu modelo)
- Customer ≠ Account (Sales vs Identity)
```

**Critérios de aceite:**
- [ ] Glossário com **pelo menos 6 termos** por contexto (Catalog, Sales, Inventory, Shipping, Billing, Identity)
- [ ] Demonstração de **termos homônimos** — mesmo nome, significados diferentes por contexto (ex: "Product")
- [ ] Lista de **sinônimos proibidos** com justificativa
- [ ] Linguagem usada é de **negócio**, não técnica (sem "Handler", "Manager", "DTO")

---

### Desafio 0.2 — Value Object `Money` (Imutabilidade & Encapsulamento)

Implemente um Value Object `Money` que represente valores monetários de forma segura.

**Requisitos:**
- Armazenar `amount` (precisão decimal) e `currency`
- Ser **imutável** — operações retornam novas instâncias
- Implementar operações: `add(Money)`, `subtract(Money)`, `multiply(factor)`, `percentage(rate)`
- Lançar erro ao operar com moedas diferentes
- Implementar `equals`/`hashCode` baseado em valor (não em referência)
- Implementar `toString()` com formatação localizada (ex: `"R$ 150,00"`, `"$ 150.00"`)
- Não pode ser negativo (invariante protegida no construtor)

**Java 25:**
```java
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        Objects.requireNonNull(currency, "Currency is required");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
    }

    public Money add(Money other) {
        requireSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }
    // ...
}
```

**Go 1.26:**
```go
type Money struct {
    amount   int64    // centavos para evitar floating point
    currency Currency
}

func NewMoney(amount int64, currency Currency) (Money, error) {
    if amount < 0 {
        return Money{}, errors.New("amount cannot be negative")
    }
    return Money{amount: amount, currency: currency}, nil
}

func (m Money) Add(other Money) (Money, error) { ... }
```

**Critérios de aceite:**
- [ ] Value Object imutável em ambas as linguagens
- [ ] Operações algébricas com validação de moeda
- [ ] Comparação por valor (`equals` em Java, comparação de struct em Go)
- [ ] Sem floating point — usar `BigDecimal` (Java) ou centavos `int64` (Go)
- [ ] Invariante: amount >= 0 protegida no construtor
- [ ] Testes cobrindo: criação válida, operações, moedas incompatíveis, edge cases (zero, negativo)

---

### Desafio 0.3 — Value Objects Tipados (Cura para Primitive Obsession)

Crie Value Objects para conceitos que normalmente são representados como primitivos.

**Requisitos:**

| Value Object | Validação | Comportamento |
|-------------|-----------|---------------|
| `EmailAddress` | Formato RFC 5322, max 254 chars | `normalize()`, `domain()` |
| `CPF` | Dígitos verificadores (algoritmo) | `formatted()` ("123.456.789-00"), `unformatted()` |
| `PhoneNumber` | DDD + número, formato brasileiro | `formatted()`, `areaCode()`, `isCell()` |
| `SKU` | Alfanumérico 8-12 chars, uppercase | `category()` (primeiros 3 chars) |
| `Quantity` | Inteiro > 0, max 9999 | `add(Quantity)`, `subtract(Quantity)`, `isZero()` |
| `Percentage` | 0-100 (ou 0.0-1.0 interno) | `applyTo(Money)`, `of(part, total)` |

**Java 25:**
```java
public record EmailAddress(String value) {
    private static final Pattern EMAIL_REGEX = Pattern.compile("...");

    public EmailAddress {
        Objects.requireNonNull(value, "Email is required");
        value = value.trim().toLowerCase();
        if (!EMAIL_REGEX.matcher(value).matches()) {
            throw new InvalidEmailException(value);
        }
    }

    public String domain() {
        return value.substring(value.indexOf('@') + 1);
    }
}
```

**Go 1.26:**
```go
type EmailAddress struct {
    value string
}

func NewEmailAddress(value string) (EmailAddress, error) {
    normalized := strings.TrimSpace(strings.ToLower(value))
    if !emailRegex.MatchString(normalized) {
        return EmailAddress{}, fmt.Errorf("invalid email: %s", value)
    }
    return EmailAddress{value: normalized}, nil
}

func (e EmailAddress) Domain() string { ... }
func (e EmailAddress) String() string { return e.value }
```

**Critérios de aceite:**
- [ ] 6 Value Objects implementados em ambas as linguagens
- [ ] Cada VO é imutável e auto-validante (nunca existe em estado inválido)
- [ ] Java: usar `record` para cada VO
- [ ] Go: usar struct com campos não-exportados + factory function
- [ ] Igualdade por valor em todos os VOs
- [ ] Testes unitários para: criação válida, criação inválida, igualdade, comportamento de domínio

---

### Desafio 0.4 — Value Object Composto `Address`

Implemente o Value Object `Address` composto por outros Value Objects.

**Requisitos:**
- Campos: `street` (String), `number` (String), `complement` (opcional), `neighborhood` (String), `city` (String), `state` (String), `zipCode` (ZipCode), `country` (Country)
- `ZipCode` é um Value Object com validação de formato (CEP brasileiro: 12345-678)
- `Country` é um Value Object com código ISO 3166-1 alpha-2
- Imutável — método `withZipCode(newZipCode)` retorna nova instância
- Validação completa no construtor (campos obrigatórios, formatos)

**Java 25:**
```java
public record Address(
    String street,
    String number,
    Optional<String> complement,
    String neighborhood,
    String city,
    String state,
    ZipCode zipCode,
    Country country
) {
    public Address {
        requireNonBlank(street, "Street");
        requireNonBlank(number, "Number");
        // ...
    }

    public Address withZipCode(ZipCode newZipCode) {
        return new Address(street, number, complement, neighborhood,
                          city, state, newZipCode, country);
    }
}
```

**Go 1.26:**
```go
type Address struct {
    street       string
    number       string
    complement   *string   // nil = sem complemento
    neighborhood string
    city         string
    state        string
    zipCode      ZipCode
    country      Country
}

func NewAddress(street, number string, complement *string,
    neighborhood, city, state string, zipCode ZipCode, country Country) (Address, error) { ... }
```

**Critérios de aceite:**
- [ ] `Address` composto por VOs (`ZipCode`, `Country`) e primitivos validados
- [ ] `ZipCode` com validação de formato CEP ("12345-678")
- [ ] `Country` com validação ISO 3166-1 alpha-2
- [ ] Imutável com método `withXxx` para "alteração"
- [ ] Campo opcional (`complement`) tratado idiomaticamente: `Optional` (Java), ponteiro (Go)
- [ ] Testes: criação válida, campos obrigatórios ausentes, formato inválido, igualdade

---

### Desafio 0.5 — Value Object `DateRange` & Enumerações de Domínio

Crie tipos temporais e enumerações ricas para o domínio.

**Requisitos:**

**DateRange:**
- `start` e `end` (datas)
- Invariante: `end >= start`
- Métodos: `contains(date)`, `overlaps(other)`, `durationInDays()`, `isActive()`
- Imutável

**Enumerações de domínio:**
- `OrderStatus` — `CREATED`, `CONFIRMED`, `PAID`, `SHIPPED`, `DELIVERED`, `CANCELLED`, `REFUNDED`
  - Com validação de transições (máquina de estados)
  - `canTransitionTo(next)` retorna boolean
- `PaymentType` — `CREDIT_CARD`, `DEBIT_CARD`, `PIX`, `BANK_SLIP`, `WALLET`
- `ProductStatus` — `DRAFT`, `ACTIVE`, `INACTIVE`, `DISCONTINUED`

**Java 25:**
```java
public record DateRange(LocalDate start, LocalDate end) {
    public DateRange {
        if (end.isBefore(start)) {
            throw new IllegalArgumentException("End date must be after start date");
        }
    }

    public boolean contains(LocalDate date) {
        return !date.isBefore(start) && !date.isAfter(end);
    }

    public boolean overlaps(DateRange other) {
        return !this.start.isAfter(other.end) && !this.end.isBefore(other.start);
    }
}

public enum OrderStatus {
    CREATED(Set.of(CONFIRMED, CANCELLED)),
    CONFIRMED(Set.of(PAID, CANCELLED)),
    PAID(Set.of(SHIPPED, REFUNDED)),
    // ...
    ;
    private final Set<OrderStatus> allowedTransitions;

    public boolean canTransitionTo(OrderStatus next) {
        return allowedTransitions.contains(next);
    }
}
```

**Go 1.26:**
```go
type DateRange struct {
    start time.Time
    end   time.Time
}

func NewDateRange(start, end time.Time) (DateRange, error) { ... }
func (d DateRange) Contains(date time.Time) bool { ... }
func (d DateRange) Overlaps(other DateRange) bool { ... }

type OrderStatus int
const (
    OrderStatusCreated OrderStatus = iota
    OrderStatusConfirmed
    // ...
)
func (s OrderStatus) CanTransitionTo(next OrderStatus) bool { ... }
```

**Critérios de aceite:**
- [ ] `DateRange` imutável com validação de invariante
- [ ] `DateRange` com métodos `contains`, `overlaps`, `durationInDays`, `isActive`
- [ ] `OrderStatus` com máquina de estados — transições válidas e inválidas
- [ ] `PaymentType` e `ProductStatus` como enumerações type-safe
- [ ] Java: usar `record` para DateRange, `enum` com campos e métodos para status
- [ ] Go: usar struct para DateRange, `const` + `iota` + método para status
- [ ] Testes: DateRange (sobreposição, contenção), OrderStatus (transições válidas/inválidas)

---

### Desafio 0.6 — Error Handling de Domínio

Implemente um sistema de erros tipados e expressivos para o domínio MerchantHub.

**Requisitos:**
- Hierarquia de erros de domínio: `DomainError` (base)
  - `ValidationError` — campo inválido (com field + message)
  - `BusinessRuleViolation` — regra de negócio violada (com rule name + context)
  - `EntityNotFound` — entity/aggregate não encontrado (com type + id)
  - `InvalidStateTransition` — transição de estado inválida (com from + to)
  - `CurrencyMismatchError` — operação com moedas diferentes
- Cada erro com: código, mensagem, timestamp
- Erros devem ser identificáveis por tipo

**Java 25:**
```java
public sealed abstract class DomainError extends RuntimeException
    permits ValidationError, BusinessRuleViolation, EntityNotFound,
            InvalidStateTransition, CurrencyMismatchError {

    private final String code;
    private final Instant timestamp;

    protected DomainError(String code, String message) {
        super(message);
        this.code = code;
        this.timestamp = Instant.now();
    }
}

public final class ValidationError extends DomainError {
    private final String field;
    // ...
}

// Pattern matching no tratamento
switch (error) {
    case ValidationError v -> handleValidation(v);
    case EntityNotFound e -> return404(e);
    case BusinessRuleViolation b -> return422(b);
    default -> return500(error);
}
```

**Go 1.26:**
```go
type DomainError struct {
    Code      string
    Message   string
    Timestamp time.Time
}

func (e *DomainError) Error() string { return fmt.Sprintf("[%s] %s", e.Code, e.Message) }

type ValidationError struct {
    DomainError
    Field string
}

type EntityNotFound struct {
    DomainError
    EntityType string
    EntityID   string
}

// Uso: errors.Is / errors.As
var err error = &EntityNotFound{...}
var notFound *EntityNotFound
if errors.As(err, &notFound) {
    // handle
}
```

**Critérios de aceite:**
- [ ] Hierarquia de erros de domínio com 5 tipos especializados
- [ ] Java: `sealed` hierarchy + pattern matching no `switch`
- [ ] Go: custom error types + `errors.Is/As` + `Unwrap()` chain
- [ ] Cada erro com código, mensagem e timestamp
- [ ] Erros usam linguagem de domínio: `BusinessRuleViolation`, não `ApplicationException`
- [ ] Testes: criação, identificação por tipo, wrapping/unwrapping

---

## Entregáveis

| Artefato | Descrição |
|----------|-----------|
| `GLOSSARY.md` | Glossário da Ubiquitous Language por Bounded Context |
| `Money` + `Currency` | Value Object monetário imutável com operações |
| `EmailAddress`, `CPF`, `PhoneNumber`, `SKU`, `Quantity`, `Percentage` | Value Objects tipados |
| `Address` + `ZipCode` + `Country` | Value Object composto |
| `DateRange` | Value Object temporal |
| `OrderStatus`, `PaymentType`, `ProductStatus` | Enumerações ricas com comportamento |
| Hierarquia de erros de domínio | Erros tipados e expressivos |
| Testes | ≥ 85% cobertura em ambas as linguagens |

---

## O que você vai exercitar

| Conceito DDD | O que pratica |
|-------------|---------------|
| **Ubiquitous Language** | Criar glossário, nomear tipos com linguagem de domínio |
| **Value Object** | Imutabilidade, auto-validação, igualdade por valor |
| **Primitive Obsession (cure)** | Substituir primitivos por tipos ricos do domínio |
| **Domain Error** | Erros expressivos com linguagem de negócio |
| **Bounded Context awareness** | Entender que termos mudam de significado entre contextos |

---

## Próximo Nível

Quando completar todos os desafios deste nível, avance para:
→ [Level 1 — Strategic Design](01-strategic-design.md)

Os Value Objects e tipos criados aqui (`Money`, `Address`, `SKU`, `OrderStatus`, erros de domínio) serão a base para modelar Entities, Aggregates e Bounded Contexts nos próximos níveis.
