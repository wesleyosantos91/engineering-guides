# Level 1 — Princípios SOLID

> **Objetivo:** Refatorar o código do Level 0 aplicando cada princípio SOLID de forma deliberada,
> transformando o domínio Payment Processor em um design extensível, testável e coeso.

**Referência:** [01-solid-principles.md](../../../.docs/DESIGN-PATTERNS/01-solid-principles.md)

---

## Pré-requisito

Todos os desafios do [Level 0 — Fundações OOP](00-oop-foundations.md) completos.
Este nível parte do código existente e o refatora sistematicamente.

---

## Desafios

### Desafio 1.1 — SRP: Single Responsibility Principle

> *"Um módulo deve ter um, e apenas um, motivo para mudar."*

**Cenário:** O `Transaction` do Level 0 está acumulando responsabilidades. Ele valida dados, gerencia transições de estado, formata saídas e calcula taxas. Refatore separando por ator/responsabilidade.

**Antes (violação):**
```
Transaction:
    - transitionTo()         ← gerência de estado
    - calculateFee()         ← cálculo financeiro
    - toReceipt()            ← formatação de saída
    - validate()             ← validação de negócio
    - toJSON()               ← serialização
```

**Depois (SRP aplicado):**

| Classe | Ator/Responsabilidade | Métodos |
|--------|----------------------|---------|
| `Transaction` | Domínio (entidade pura) | Estado, dados, transições |
| `FeeCalculator` | Financeiro | `calculate(Transaction): Money` |
| `TransactionValidator` | Compliance/Negócio | `validate(Transaction): ValidationResult` |
| `ReceiptFormatter` | Apresentação | `format(Transaction): Receipt` |
| `TransactionSerializer` | Integração/I/O | `toJSON(Transaction): String` |

**Requisitos:**
- Extrair cada responsabilidade em classe/struct dedicada
- `Transaction` mantém apenas estado e comportamento de domínio intrínseco
- Cada nova classe/struct tem **um único motivo para mudar**
- Nenhuma classe com nome genérico (Manager, Handler, Utils)

**Critérios de aceite:**
- [ ] `Transaction` não contém lógica de cálculo, formatação ou serialização
- [ ] `FeeCalculator` — calcula taxa por tipo de pagamento e bandeira
- [ ] `TransactionValidator` — valida limites, dados obrigatórios, regras de negócio
- [ ] `ReceiptFormatter` — gera recibo formatado com dados da transação
- [ ] `TransactionSerializer` — serializa/deserializa para JSON
- [ ] Cada classe é testável isoladamente com zero dependência das outras
- [ ] Java: classes `final` ou records onde aplicável
- [ ] Go: structs com interface mínima, métodos com receiver correto (value vs pointer)

---

### Desafio 1.2 — OCP: Open/Closed Principle

> *"Aberto para extensão, fechado para modificação."*

**Cenário:** O `FeeCalculator` do desafio anterior usa `if/else` para determinar a taxa por tipo de pagamento. Cada novo método de pagamento exige modificação. Refatore para ser extensível sem alterar código existente.

**Antes (violação):**
```
class FeeCalculator:
    calculate(tx):
        if tx.paymentMethod.type == CREDIT_CARD:
            return tx.amount * 0.029
        else if tx.paymentMethod.type == PIX:
            return tx.amount * 0.001
        else if tx.paymentMethod.type == BANK_SLIP:
            return tx.amount * 0.015
        // Toda nova forma de pagamento → modificação aqui
```

**Depois (OCP aplicado):**

| Componente | Papel |
|------------|-------|
| `FeeStrategy` (interface) | Contrato para cálculo de taxa |
| `CreditCardFeeStrategy` | Taxa de cartão de crédito (2.9%) |
| `PixFeeStrategy` | Taxa de PIX (0.1%) |
| `BankSlipFeeStrategy` | Taxa de boleto (1.5%) |
| `FeeCalculator` | Recebe `FeeStrategy` e delega — **nunca precisa ser modificado** |
| `FeeStrategyRegistry` | Registra e resolve strategies por `PaymentType` |

**Requisitos:**
- Interface `FeeStrategy` com `calculate(Transaction): Money`
- Uma implementação por tipo de pagamento
- `FeeCalculator` recebe a strategy via construtor ou registry
- Adicionar um novo tipo de pagamento **não altera nenhum arquivo existente**
- `FeeStrategyRegistry` permite registrar novas strategies em runtime

**Critérios de aceite:**
- [ ] Zero `if/else` ou `switch` sobre tipo de pagamento no cálculo de taxa
- [ ] Nova strategy `DebitCardFeeStrategy` adicionada **sem alterar** classes existentes
- [ ] `FeeStrategyRegistry` com `register(PaymentType, FeeStrategy)` e `resolve(PaymentType)`
- [ ] Java: usar `Map<PaymentType, FeeStrategy>` no registry
- [ ] Go: usar `map[PaymentType]FeeStrategy` no registry
- [ ] Testes: cada strategy individualmente + registry resolve corretamente + strategy ausente → erro

---

### Desafio 1.3 — LSP: Liskov Substitution Principle

> *"Subtipos devem ser substituíveis por seus tipos base sem alterar a corretude."*

**Cenário:** Nem todo `PaymentMethod` suporta as mesmas operações. `CreditCard` suporta estorno (`refund`), mas `Pix` não (após 24h). `BankSlip` não suporta captura parcial. Se todas implementam a mesma interface com operações que lançam `UnsupportedOperationException`, estamos violando LSP.

**Requisitos:**
- Identificar onde o contrato `PaymentMethod` do Level 0 pode violar LSP
- Segregar interfaces para que cada método implemente **apenas o que suporta**
- Nenhuma implementação deve lançar `UnsupportedOperationException` ou retornar erro por "não suportado"
- Todo implementador de uma interface **cumpre 100% do contrato**

**Refatoração proposta:**

| Interface | Métodos | Implementadores |
|-----------|---------|-----------------|
| `PaymentMethod` | `type()`, `validate()`, `maskedIdentifier()` | Todos |
| `Chargeable` | `authorize(amount)`, `capture(amount)` | CreditCard, DebitCard |
| `Refundable` | `refund(amount)`, `partialRefund(amount)` | CreditCard |
| `Instantaneous` | `transfer(amount)` | Pix |
| `Expirable` | `expiresAt()`, `isExpired()` | BankSlip |

**Verificação LSP (Design by Contract):**
- **Pré-condições:** subtipos não devem exigir mais que o tipo base
- **Pós-condições:** subtipos não devem entregar menos que o tipo base
- **Invariantes:** subtipos devem manter todas as invariantes do tipo base

**Critérios de aceite:**
- [ ] Zero `UnsupportedOperationException` / `errors.New("not supported")`
- [ ] Cada implementação satisfaz 100% das interfaces que declara
- [ ] Código cliente usa interfaces específicas, não cast/type-check
- [ ] Java: `sealed interface` com `permits` para cada sub-contrato
- [ ] Go: interfaces pequenas compostas (`Chargeable`, `Refundable` separadas)
- [ ] Teste de substituição: função que aceita `PaymentMethod` funciona com **qualquer** implementação
- [ ] Teste de contrato: cada interface testada com todos os implementadores

---

### Desafio 1.4 — ISP: Interface Segregation Principle

> *"Clientes não devem ser forçados a depender de interfaces que não utilizam."*

**Cenário:** O `TransactionRepository` do Level 0 tem uma interface ampla (`save`, `findById`, `findByCustomer`, `findByStatus`, `findAll`). Nem todo consumidor precisa de tudo — o módulo de relatórios precisa apenas de queries, o módulo de processamento precisa apenas de `save` e `findById`.

**Requisitos:**
- Segregar `TransactionRepository` em interfaces menores e focadas
- Cada consumidor depende apenas da interface mínima que precisa
- Implementação `InMemoryTransactionRepository` implementa todas

| Interface | Métodos | Consumidor típico |
|-----------|---------|-------------------|
| `TransactionWriter` | `save(tx)` | Processador de pagamento |
| `TransactionReader` | `findById(id)`, `findByCustomer(customerId)` | Serviço de consulta |
| `TransactionQuery` | `findByStatus(status)`, `findAll(query)` | Módulo de relatórios |
| `TransactionRepository` | Compõe Writer + Reader + Query | Poucos consumidores que precisam de tudo |

**Critérios de aceite:**
- [ ] 3+ interfaces segregadas derivadas do repository original
- [ ] `TransactionRepository` composta pelas interfaces menores (extends/embedding)
- [ ] Processador depende **apenas** de `TransactionWriter` + `TransactionReader`
- [ ] Módulo de relatório depende **apenas** de `TransactionQuery`
- [ ] Java: interfaces com `extends` para compor — `TransactionRepository extends TransactionWriter, TransactionReader, TransactionQuery`
- [ ] Go: interfaces compostas via embedding — `type TransactionRepository interface { TransactionWriter; TransactionReader; TransactionQuery }`
- [ ] Implementação `InMemoryTransactionRepository` satisfaz `TransactionRepository` (e portanto todas)
- [ ] Testes: cada consumidor testado com mock da interface mínima (não da interface gigante)

---

### Desafio 1.5 — DIP: Dependency Inversion Principle

> *"Dependa de abstrações, não de implementações."*

**Cenário:** Crie o `PaymentProcessor` — o componente central que orquestra o processamento de pagamento. Ele precisa de: repositório (persistência), validador, calculador de taxas, notificador. **Todas as dependências devem ser abstrações injetadas via construtor.**

**Requisitos:**
- `PaymentProcessor` não instancia nenhuma dependência internamente (`new`)
- Todas as dependências são interfaces injetadas via construtor
- Pode ser testado com mocks/stubs de todas as dependências
- Inversão: módulo de alto nível (`PaymentProcessor`) e módulo de baixo nível (`InMemoryRepository`, `ConsoleNotifier`) **ambos dependem de abstrações**

**Estrutura:**

```
┌──────────────────────────┐
│     PaymentProcessor     │ ← módulo de alto nível
│  - repository: Writer    │
│  - validator: Validator  │
│  - feeCalc: FeeStrategy  │
│  - notifier: Notifier    │
│  + process(tx): Result   │
└──────────┬───────────────┘
           │ depende de abstrações (interfaces)
           ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │ <<interface>> │  │ <<interface>> │  │ <<interface>> │
    │ Writer        │  │ FeeStrategy  │  │ Notifier      │
    └──────▲───────┘  └──────▲───────┘  └──────▲───────┘
           │                 │                 │
    ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐
    │ InMemoryRepo  │  │CreditCardFee │  │ConsoleNotify  │
    └──────────────┘  └──────────────┘  └──────────────┘
```

**Fluxo `process(transaction)`:**
1. Validar a transação (`TransactionValidator.validate()`)
2. Calcular taxa (`FeeStrategy.calculate()`)
3. Processar pagamento (transição de estado)
4. Persistir (`TransactionWriter.save()`)
5. Notificar (`Notifier.notify()`)
6. Retornar `PaymentResult`

**Critérios de aceite:**
- [ ] `PaymentProcessor` tem **zero imports** de implementações concretas
- [ ] Todas as dependências injetadas via construtor
- [ ] Java: construtor com `final` fields para dependências
- [ ] Go: `NewPaymentProcessor(writer, validator, feeCalc, notifier)` function
- [ ] Teste com mocks: `InMemoryWriter`, `StubFeeStrategy`, `SpyNotifier`
- [ ] Teste prova que `PaymentProcessor` funciona com **qualquer** implementação das interfaces
- [ ] Nenhum `new ConcreteClass()` dentro do PaymentProcessor
- [ ] Composição root (wiring) em `main`/`Main` — único lugar que conhece as implementações concretas

---

### Desafio 1.6 — SOLID Integrado: Refatoração Completa

> Aplique **todos os 5 princípios** em conjunto e documente os trade-offs.

**Cenário:** Revise todo o código produzido nos desafios 1.1 a 1.5. Garanta que:

1. **SRP** — Cada classe tem um único motivo para mudar
2. **OCP** — Novos comportamentos via extensão (novas classes), não modificação
3. **LSP** — Toda substituição por subtipo é segura
4. **ISP** — Nenhum consumidor depende de métodos que não usa
5. **DIP** — `PaymentProcessor` depende apenas de abstrações

**Requisitos de documentação:**
- `PATTERNS.md` — Lista onde cada princípio SOLID está aplicado, com justificativa
- `DECISIONS.md` — Trade-offs documentados:
  - Onde OCP foi aplicado vs onde foi considerado YAGNI
  - Onde ISP gerou interfaces demais (over-segregation) e foi simplificado
  - Onde herança (Template Method) foi preferida sobre composição (Strategy)
  - Como DIP impactou a complexidade do wiring no `main`

**Diagramas recomendados (Mermaid):**
- Diagrama de classes mostrando as interfaces e implementações
- Diagrama de dependências mostrando a inversão (alto nível → abstrações ← baixo nível)

**Critérios de aceite:**
- [ ] Code review checklist do [06-best-practices.md](../../../.docs/DESIGN-PATTERNS/06-best-practices.md#checklist-de-code-review) satisfeito
- [ ] Todos os testes dos desafios 1.1-1.5 passando
- [ ] Nenhuma God Class, nenhum Util/Helper genérico
- [ ] `PATTERNS.md` com mapeamento princípio → classe → justificativa
- [ ] `DECISIONS.md` com pelo menos 4 trade-offs documentados
- [ ] Cobertura de testes ≥ 85% em ambas as linguagens
- [ ] Java e Go: código idiomático (records vs classes, sealed vs open, embedding vs interfaces)

---

## Entregáveis

| Artefato | Descrição |
|----------|-----------|
| `FeeCalculator` + `FeeStrategy` + estratégias | SRP + OCP — cálculo extensível |
| `TransactionValidator` | SRP — validação isolada |
| `ReceiptFormatter` | SRP — formatação isolada |
| Interfaces segregadas (`Chargeable`, `Refundable`, etc.) | LSP + ISP |
| `TransactionWriter/Reader/Query` + composição | ISP |
| `PaymentProcessor` com DI manual | DIP — zero dependência concreta |
| `Notifier` interface + `ConsoleNotifier` | DIP — abstração de notificação |
| `PATTERNS.md` | Documentação de padrões aplicados |
| `DECISIONS.md` | Trade-offs justificados |
| Testes com mocks/stubs | ≥ 85% cobertura |

---

## O que você vai exercitar

| Princípio | Java 25 | Go 1.26 |
|-----------|---------|---------|
| **SRP** | Classes `final` focadas, records para dados | Structs com interface mínima |
| **OCP** | Strategy pattern via interfaces | Interface + registry com map |
| **LSP** | Sealed interfaces, sem exceptions "forçadas" | Interfaces pequenas, sem panic |
| **ISP** | `extends` múltiplo em interfaces | Interface embedding |
| **DIP** | Construtor com interfaces `final` | Factory function com interfaces como parâmetros |

---

## Próximo Nível

Quando completar todos os desafios deste nível, avance para:
→ [Level 2 — Padrões Criacionais](02-creational-patterns.md)

Os princípios SOLID aplicados aqui serão a base para **todos** os padrões GoF — cada padrão implementa um ou mais princípios SOLID de forma concreta.
