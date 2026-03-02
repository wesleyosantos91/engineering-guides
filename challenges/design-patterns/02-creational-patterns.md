# Level 2 — Padrões Criacionais (GoF)

> **Objetivo:** Implementar os 5 padrões criacionais do GoF no domínio Payment Processor,
> controlando **como** objetos são criados de forma flexível, desacoplada e testável.

**Referência:** [02-creational-patterns.md](../../../.docs/DESIGN-PATTERNS/02-creational-patterns.md)

---

## Pré-requisito

[Level 1 — Princípios SOLID](01-solid-principles.md) completo.
Os padrões criacionais implementam DIP (factories retornam abstrações) e OCP (novos produtos sem modificar factories).

---

## Desafios

### Desafio 2.1 — Singleton: Configuration Registry

> *"Garantir que uma classe tenha apenas uma instância e fornecer acesso global a ela."*

**Cenário:** O Payment Processor precisa de um `PaymentConfig` global que armazena: taxas padrão por tipo de pagamento, limites de valor (mínimo/máximo), configurações de retry, timeout de gateway. Este config é carregado uma vez no startup e consultado por todo o sistema.

**Requisitos:**
- `PaymentConfig` como Singleton thread-safe
- Carregamento lazy (inicializado no primeiro acesso)
- Imutável após inicialização (apenas leitura)
- Configurável em testes (resetável para testes — anti-pattern consciente documentado)

**Variantes a implementar:**

| Variante | Java 25 | Go 1.26 |
|----------|---------|---------|
| **Eager** | Campo `static final` | `init()` function |
| **Lazy Thread-Safe** | `Holder idiom` (lazy + thread-safe sem lock) | `sync.Once` |
| **Enum Singleton** | `enum` com instância única | N/A (usar `sync.Once`) |

**Java 25:**
```java
// Holder Idiom — lazy + thread-safe, zero overhead
public final class PaymentConfig {
    private PaymentConfig() { /* load config */ }
    
    private static final class Holder {
        static final PaymentConfig INSTANCE = new PaymentConfig();
    }
    
    public static PaymentConfig getInstance() {
        return Holder.INSTANCE;
    }
}
```

**Go 1.26:**
```go
var (
    instance *PaymentConfig
    once     sync.Once
)

func GetPaymentConfig() *PaymentConfig {
    once.Do(func() {
        instance = loadConfig()
    })
    return instance
}
```

**Anti-pattern awareness:**
- Documentar por que Singleton dificulta testabilidade
- Implementar alternativa com DI: `PaymentConfig` injetado como dependência normal
- Comparar trade-offs: Singleton global vs DI com escopo singleton

**Critérios de aceite:**
- [ ] 3 variantes implementadas (eager, lazy thread-safe, enum/sync.Once)
- [ ] Thread-safety comprovada com teste de concorrência
- [ ] Configuração imutável após inicialização
- [ ] `DECISIONS.md`: trade-off Singleton vs DI con escopo singleton
- [ ] Java: Holder idiom + Enum singleton demonstrados
- [ ] Go: `sync.Once` como mecanismo principal
- [ ] Testes de concorrência: 100 goroutines/threads acessando simultaneamente

---

### Desafio 2.2 — Factory Method: PaymentMethod Creation

> *"Definir uma interface para criar um objeto, mas deixar as subclasses decidirem qual classe instanciar."*

**Cenário:** O sistema recebe dados de pagamento de diferentes fontes (API, arquivo batch, webhook). Cada fonte fornece dados em formatos diferentes, mas o resultado deve ser um `PaymentMethod` do domínio. Use Factory Method para desacoplar a criação.

**Requisitos:**
- Interface `PaymentMethodFactory` com `create(PaymentMethodRequest): PaymentMethod`
- Factories concretas por tipo: `CreditCardFactory`, `PixFactory`, `BankSlipFactory`
- Validação de dados durante a criação (factory é responsável por criar objetos válidos)
- Factory resolve a implementação correta sem `if/else` no código cliente

**Estrutura:**

```
┌───────────────────────────┐
│   PaymentMethodFactory    │ ← interface
│  + create(request): PM    │
└───────────▲───────────────┘
            │
  ┌─────────┼──────────┐
  │         │          │
┌─┴──────┐┌─┴──────┐┌──┴─────┐
│CreditF  ││PixF    ││SlipF   │
│create()→ ││create()→││create()│
│CreditC.  ││Pix      ││BankSlip│
└─────────┘└─────────┘└────────┘
```

**Java 25:**
```java
public sealed interface PaymentMethodFactory
    permits CreditCardFactory, PixFactory, BankSlipFactory {
    
    PaymentMethod create(PaymentMethodRequest request);
}

// Registry para resolver factory por tipo
public final class PaymentMethodFactoryRegistry {
    private final Map<PaymentType, PaymentMethodFactory> factories;
    
    public PaymentMethod create(PaymentType type, PaymentMethodRequest request) {
        return factories.getOrDefault(type, t -> { throw ...; })
                        .create(request);
    }
}
```

**Go 1.26:**
```go
type PaymentMethodFactory interface {
    Create(request PaymentMethodRequest) (PaymentMethod, error)
}

type PaymentMethodFactoryRegistry struct {
    factories map[PaymentType]PaymentMethodFactory
}

func (r *PaymentMethodFactoryRegistry) Register(pt PaymentType, f PaymentMethodFactory) { ... }
func (r *PaymentMethodFactoryRegistry) Create(pt PaymentType, req PaymentMethodRequest) (PaymentMethod, error) { ... }
```

**Critérios de aceite:**
- [ ] Interface `PaymentMethodFactory` + 3+ factories concretas
- [ ] Registry que resolve factory por `PaymentType`
- [ ] Validação na factory — objetos inválidos **nunca** são criados
- [ ] Zero `if/else` no código cliente para decidir qual factory usar
- [ ] Adicionar nova factory **não altera** código existente (OCP)
- [ ] Testes: cada factory individualmente + registry resolve correto + tipo desconhecido → erro

---

### Desafio 2.3 — Abstract Factory: Gateway Families por Região

> *"Fornecer uma interface para criar famílias de objetos relacionados sem especificar classes concretas."*

**Cenário:** O Payment Processor opera em múltiplas regiões (BR, US, EU). Cada região tem uma **família** de componentes que devem ser usados **juntos**: gateway de processamento, calculadora de impostos, formatador de moeda, gerador de recibo. Misturar componentes de famílias diferentes causa bugs sutis.

**Requisitos:**
- `RegionalPaymentFactory` (Abstract Factory) com métodos para criar cada componente
- Famílias concretas: `BrazilPaymentFactory`, `USPaymentFactory`, `EUPaymentFactory`
- Componentes da família: `Gateway`, `TaxCalculator`, `CurrencyFormatter`, `ReceiptGenerator`
- O sistema seleciona a família correta baseado na região da transação

**Estrutura:**

```
┌──────────────────────────────┐
│    RegionalPaymentFactory    │ ← Abstract Factory
│  + createGateway()           │
│  + createTaxCalculator()     │
│  + createCurrencyFormatter() │
│  + createReceiptGenerator()  │
└──────────────▲───────────────┘
               │
    ┌──────────┼──────────┐
    │          │          │
┌───┴────┐┌───┴────┐┌────┴───┐
│Brazil  ││  US    ││  EU    │
│Factory ││Factory ││Factory │
│→BRGw   ││→USGw   ││→EUGw   │
│→BRTax  ││→USTax  ││→EUTax  │
│→BRFmt  ││→USFmt  ││→EUFmt  │
│→BRRcpt ││→USRcpt ││→EURcpt │
└────────┘└────────┘└────────┘
```

**Requisitos específicos por família:**

| Componente | Brasil | EUA | Europa |
|------------|--------|-----|--------|
| **Gateway** | PIX, Boleto, Cartão | ACH, Wire, Card | SEPA, Card |
| **TaxCalculator** | IOF, ISS | Sales Tax (por estado) | VAT |
| **CurrencyFormatter** | `R$ 1.000,00` | `$1,000.00` | `€1.000,00` |
| **ReceiptGenerator** | NF-e simplificada | US Receipt | EU Invoice |

**Critérios de aceite:**
- [ ] Abstract Factory com 4 métodos de criação
- [ ] 3 famílias concretas (BR, US, EU) — cada uma produzindo 4 componentes
- [ ] Componentes de uma família são **compatíveis** entre si
- [ ] Impossível misturar componentes de famílias diferentes (compilação ou validação)
- [ ] Factory selecionada por região no wiring — `PaymentProcessor` não sabe qual família usa
- [ ] Java: interfaces para cada produto + factory interface
- [ ] Go: interfaces para cada produto + factory interface
- [ ] Testes: cada família produz componentes coerentes + factory correta por região

---

### Desafio 2.4 — Builder: Transaction Builder

> *"Separar a construção de um objeto complexo da sua representação."*

**Cenário:** `Transaction` tem muitos campos: obrigatórios (customer, paymentMethod, amount) e opcionais (metadata, retryPolicy, idempotencyKey, description, expiresAt, splitRules). Construir com construtor telescópico é ilegível. Use Builder para construção fluente e segura.

**Requisitos:**
- `TransactionBuilder` com API fluente
- Campos obrigatórios validados — `build()` falha se ausentes
- Campos opcionais com valores default razoáveis
- `Transaction` resultante é **imutável**
- (Bônus) Step Builder: forçar ordem de campos obrigatórios via interfaces

**Java 25:**
```java
Transaction tx = Transaction.builder()
    .customer(customer)
    .paymentMethod(creditCard)
    .amount(Money.of(150_00, Currency.BRL))
    .description("Compra online")
    .idempotencyKey(UUID.randomUUID())
    .retryPolicy(RetryPolicy.exponential(3))
    .build();  // ← valida e retorna objeto imutável
```

**Go 1.26:**
```go
// Functional Options Pattern (idiomático em Go)
tx, err := NewTransaction(
    customer,
    creditCard,
    NewMoney(15000, BRL),
    WithDescription("Compra online"),
    WithIdempotencyKey(uuid.New()),
    WithRetryPolicy(ExponentialRetry(3)),
)
```

**Variantes a implementar:**

| Variante | Descrição |
|----------|-----------|
| **Fluent Builder** | Java: `builder().x().y().build()` |
| **Step Builder** | Java: interfaces encadeadas forçam campos obrigatórios na ordem |
| **Functional Options** | Go: opções como funções `WithX(value)` |

**Critérios de aceite:**
- [ ] Builder com API fluente (Java) e Functional Options (Go)
- [ ] Campos obrigatórios validados em `build()` — ausência gera erro
- [ ] Campos opcionais com defaults (retryPolicy = noRetry, description = "")
- [ ] `Transaction` imutável após construção
- [ ] Java: Step Builder (bônus) que força `customer` → `paymentMethod` → `amount` → opcionais → `build()`
- [ ] Go: Functional Options pattern com `type TransactionOption func(*Transaction)`
- [ ] Testes: construção válida, campos obrigatórios ausentes, defaults aplicados, imutabilidade

---

### Desafio 2.5 — Prototype: Transaction Templates

> *"Criar novos objetos copiando um protótipo existente."*

**Cenário:** O sistema suporta **pagamentos recorrentes** (assinaturas). Uma transação-template é criada uma vez e clonada a cada cobrança mensal, alterando apenas a data e o ID. Clonar é mais barato que reconstruir do zero (especialmente se envolve resolução de gateway, cálculo de taxa, etc.).

**Requisitos:**
- Interface `Cloneable<T>` (Java) / método `Clone()` (Go)
- `Transaction` implementa clonagem profunda (deep copy)
- Clone gera novo ID e timestamp, mas mantém customer, paymentMethod, amount iguais
- `TransactionTemplate` armazena protótipos por nome (subscription plan)
- Registry de templates: `register(name, template)`, `create(name): Transaction`

**Java 25:**
```java
public interface Prototype<T> {
    T deepClone();
}

public final class Transaction implements Prototype<Transaction> {
    @Override
    public Transaction deepClone() {
        return new Transaction(
            UUID.randomUUID(),          // novo ID
            this.customer,              // cópia do mesmo
            this.paymentMethod,         // cópia do mesmo
            this.amount,                // imutável — compartilha
            TransactionStatus.CREATED,  // reset do estado
            Instant.now(),              // novo timestamp
            // ...
        );
    }
}
```

**Go 1.26:**
```go
type Prototype[T any] interface {
    DeepClone() T
}

func (t *Transaction) DeepClone() *Transaction {
    clone := *t                    // shallow copy
    clone.ID = uuid.New()          // novo ID
    clone.Status = StatusCreated   // reset estado
    clone.CreatedAt = time.Now()   // novo timestamp
    clone.History = nil            // reset histórico
    // deep copy de campos reference-type
    clone.Metadata = maps.Clone(t.Metadata)
    return &clone
}
```

**Critérios de aceite:**
- [ ] Interface `Prototype<T>` / `Prototype[T]` genérica
- [ ] Deep clone correto — alteração no clone **não afeta** o original
- [ ] Clone gera novo ID, timestamp e reseta estado para `CREATED`
- [ ] `TransactionTemplateRegistry` com `register(name, template)` e `createFrom(name)`
- [ ] Testes: clone é independente do original, clone tem novo ID, template registry funciona
- [ ] Teste de deep copy: alterar campo do clone não altera original (referências não compartilhadas)
- [ ] Performance: clonar N transações vs criar do zero (benchmark básico)

---

### Desafio 2.6 — Integração: Factory Composition

**Cenário:** Combine os padrões criacionais em um fluxo real:

1. `PaymentConfig` (Singleton) fornece configuração global
2. `PaymentMethodFactoryRegistry` (Factory Method) cria o `PaymentMethod` correto
3. `RegionalPaymentFactory` (Abstract Factory) cria componentes da região certa
4. `TransactionBuilder` (Builder) constrói a transação com dados obrigatórios e opcionais
5. `TransactionTemplateRegistry` (Prototype) clona templates para recorrências

**Fluxo:**
```
PaymentRequest recebido
  │
  ├── PaymentConfig.getInstance() → resolve configurações
  │
  ├── PaymentMethodFactoryRegistry.create(type, data) → PaymentMethod
  │
  ├── RegionalPaymentFactory.create(region) → Gateway + TaxCalc + Formatter
  │
  ├── TransactionBuilder
  │     .customer(...)
  │     .paymentMethod(...)
  │     .amount(...)
  │     .build() → Transaction
  │
  └── (se recorrente) TemplateRegistry.createFrom("premium-monthly") → Transaction clone
```

**Critérios de aceite:**
- [ ] Fluxo completo de criação de pagamento usando os 5 padrões
- [ ] Cada padrão em sua responsabilidade — sem "God factory"
- [ ] Wiring no `main` — composition root
- [ ] `PATTERNS.md` documentando onde cada padrão criacional é usado e por quê
- [ ] Testes end-to-end: request → payment method → factory regional → transaction → resultado

---

## Entregáveis

| Artefato | Padrão | Descrição |
|----------|--------|-----------|
| `PaymentConfig` | Singleton | Configuração global thread-safe |
| `PaymentMethodFactory` + registry | Factory Method | Criação desacoplada por tipo |
| `RegionalPaymentFactory` + 3 famílias | Abstract Factory | Componentes por região |
| `TransactionBuilder` | Builder | Construção fluente e validada |
| `TransactionTemplate` + registry | Prototype | Clonagem para recorrências |
| Fluxo integrado | Todos | Composição dos 5 padrões |
| `PATTERNS.md` | — | Documentação de padrões aplicados |
| Testes | — | ≥ 85% cobertura |

---

## Próximo Nível

→ [Level 3 — Padrões Estruturais](03-structural-patterns.md)
