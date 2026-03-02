# Level 3 — Padrões Estruturais (GoF)

> **Objetivo:** Implementar os 7 padrões estruturais do GoF no domínio Payment Processor,
> compondo objetos e classes em **estruturas maiores** mantendo flexibilidade e eficiência.

**Referência:** [03-structural-patterns.md](../../../.docs/DESIGN-PATTERNS/03-structural-patterns.md)

---

## Pré-requisito

[Level 2 — Padrões Criacionais](02-creational-patterns.md) completo.

---

## Desafios

### Desafio 3.1 — Adapter: Gateway Integration

> *"Converter a interface de uma classe em outra interface esperada pelos clientes."*

**Cenário:** O Payment Processor precisa integrar com gateways externos que têm APIs completamente diferentes. O gateway Stripe usa `charge(amount_cents, currency, source)`, o PagSeguro usa `criarPagamento(valor, moeda, metodo)`, e o PayPal usa `createPayment(money, paymentSource)`. O core do sistema não deve conhecer essas APIs — apenas usa `Gateway.process(Transaction)`.

**Requisitos:**
- Interface interna `Gateway` com `process(Transaction): GatewayResponse`
- 3 classes externas simuladas (Stripe, PagSeguro, PayPal) com APIs incompatíveis
- 3 Adapters que traduzem `Gateway.process()` para a API do gateway externo
- Object Adapter (composição) — não Class Adapter (herança)

**Estrutura:**

```
Core:  Gateway.process(tx) ─────────┐
                                     │ (Port — interface interna)
                                     ▼
Adapter: StripeGatewayAdapter ──── implements Gateway
         │
         └── delega para StripeSDK.charge(amount_cents, currency, source)
```

| Gateway Externo | Método Original | Adapter Traduz Para |
|-----------------|----------------|---------------------|
| `StripeSDK` | `charge(amountCents, currency, source)` | `Gateway.process(tx)` |
| `PagSeguroSDK` | `criarPagamento(valor, moeda, metodo)` | `Gateway.process(tx)` |
| `PayPalSDK` | `createPayment(money, paymentSource)` | `Gateway.process(tx)` |

**Critérios de aceite:**
- [ ] Interface `Gateway` no core — zero referência a SDKs externos
- [ ] 3 SDKs simulados com APIs diferentes (classes/structs com métodos próprios)
- [ ] 3 Adapters (Object Adapter) que implementam `Gateway` e delegam para o SDK
- [ ] Conversão de dados bidirecional (Transaction → formato do SDK, response do SDK → GatewayResponse)
- [ ] `PaymentProcessor` usa `Gateway` sem saber qual adapter/SDK está por trás
- [ ] Testes: cada adapter individualmente + substituição transparente no processor

---

### Desafio 3.2 — Bridge: Transaction Processing Abstraction

> *"Desacoplar uma abstração da sua implementação para que ambas possam variar independentemente."*

**Cenário:** O processamento de transações tem duas dimensões de variação independentes: **tipo de operação** (charge, refund, void) e **canal de processamento** (online real-time, batch, scheduled). Sem Bridge, teríamos uma explosão combinatória: `OnlineCharge`, `BatchCharge`, `ScheduledCharge`, `OnlineRefund`, `BatchRefund`...

**Requisitos:**
- **Abstração:** `TransactionOperation` com variantes `ChargeOperation`, `RefundOperation`, `VoidOperation`
- **Implementação:** `ProcessingChannel` com variantes `OnlineChannel`, `BatchChannel`, `ScheduledChannel`
- Bridge conecta operação ao canal — 3 operações × 3 canais = 9 combinações sem 9 classes

**Estrutura:**

```
┌────────────────────────┐         ┌────────────────────────┐
│  TransactionOperation   │────────▶│   ProcessingChannel    │
│  (Abstraction)          │         │   (Implementation)     │
├────────────────────────┤         ├────────────────────────┤
│ + execute(tx)           │         │ + send(payload)        │
│ - channel: Channel      │         │ + getStatus(id)        │
└────────▲───────────────┘         └─────────▲──────────────┘
         │                                    │
    ┌────┼────┐                    ┌──────────┼─────────┐
    │    │    │                    │          │         │
  Charge Refund Void          Online    Batch    Scheduled
```

**Critérios de aceite:**
- [ ] Abstração (`TransactionOperation`) e implementação (`ProcessingChannel`) variam independentemente
- [ ] 3 operações × 3 canais compostos em runtime sem classes combinatórias
- [ ] Adicionar novo canal (ex: `WebhookChannel`) sem alterar operações
- [ ] Adicionar nova operação (ex: `PartialCaptureOperation`) sem alterar canais
- [ ] Java: abstração com `protected` reference ao implementor
- [ ] Go: struct com campo interface para o channel
- [ ] Testes: todas as 9 combinações funcionam corretamente

---

### Desafio 3.3 — Composite: Split Payments

> *"Compor objetos em estruturas de árvore para representar hierarquias parte-todo."*

**Cenário:** O sistema suporta **split payment**: um pagamento pode ser dividido entre múltiplos destinos (marketplace → vendedor A + vendedor B + plataforma). Cada parte pode ser um pagamento simples ou uma composição de partes menores. O total das partes deve ser igual ao valor original.

**Requisitos:**
- Interface `PaymentComponent` com `amount(): Money` e `process(): PaymentResult`
- `SinglePayment` (Leaf) — pagamento unitário para um destinatário
- `SplitPayment` (Composite) — contém múltiplos `PaymentComponent` filhos
- Validação: soma dos filhos == amount total do composite
- Operação recursiva `process()` em toda a árvore

**Estrutura:**
```
SplitPayment (R$ 1000,00)
├── SinglePayment → Vendedor A (R$ 700,00)
├── SplitPayment → Taxas (R$ 300,00)
│   ├── SinglePayment → Plataforma (R$ 250,00)
│   └── SinglePayment → Gateway Fee (R$ 50,00)
```

**Critérios de aceite:**
- [ ] Interface `PaymentComponent` implementada por `SinglePayment` e `SplitPayment`
- [ ] `SplitPayment` contém filhos `PaymentComponent` (recursivo)
- [ ] Validação: soma dos filhos == total do split
- [ ] `process()` recursivo — processa todos os nós folha
- [ ] `amount()` recursivo — soma todos os filhos
- [ ] Adição/remoção de nós na árvore
- [ ] Testes: árvore simples, árvore aninhada, validação de soma, processamento recursivo

---

### Desafio 3.4 — Decorator: Processing Pipeline Enhancement

> *"Adicionar responsabilidades a um objeto dinamicamente, como alternativa flexível à herança."*

**Cenário:** O processamento de pagamento precisa de cross-cutting concerns que podem ser combinados livremente: logging, métricas de tempo, retry em caso de falha, cache de idempotência. Cada concern é um Decorator que envolve o `Gateway` e adiciona comportamento antes/depois de delegar.

**Requisitos:**
- Interface `Gateway` (já existente do desafio 3.1) como Component
- Decorators empilháveis: `LoggingGateway`, `MetricsGateway`, `RetryGateway`, `IdempotencyGateway`
- Ordem dos decorators importa (logging antes de retry mostra tentativas; logging depois de retry mostra apenas resultado)
- Cada decorator é independente — pode ser adicionado/removido sem afetar outros

**Composição:**
```java
// Java
Gateway gateway = new LoggingGateway(
    new MetricsGateway(
        new RetryGateway(
            new IdempotencyGateway(
                new StripeGatewayAdapter(stripeSDK)
            ), maxRetries: 3
        )
    )
);
```

```go
// Go
var gw Gateway = NewStripeAdapter(stripeSDK)
gw = NewIdempotencyGateway(gw, cache)
gw = NewRetryGateway(gw, 3)
gw = NewMetricsGateway(gw, metrics)
gw = NewLoggingGateway(gw, logger)
```

| Decorator | Comportamento |
|-----------|---------------|
| `LoggingGateway` | Log request/response com detalhes da transação |
| `MetricsGateway` | Mede tempo de processamento, conta sucesso/falha |
| `RetryGateway` | Retry com backoff exponencial em caso de erro retryable |
| `IdempotencyGateway` | Cache de resultado por idempotency key — evita processamento duplicado |

**Critérios de aceite:**
- [ ] 4 decorators independentes, empilháveis em qualquer ordem
- [ ] Cada decorator implementa `Gateway` e delega para o wrapped
- [ ] `LoggingGateway` — loga entrada e saída (ou erro)
- [ ] `MetricsGateway` — registra duração e resultado
- [ ] `RetryGateway` — retry N vezes com backoff se erro retryable
- [ ] `IdempotencyGateway` — retorna cached result se mesma idempotency key
- [ ] Testes: cada decorator isolado + composição de 4 decorators + ordem importa

---

### Desafio 3.5 — Facade: Payment Service API

> *"Fornecer uma interface unificada e simplificada para um subsistema complexo."*

**Cenário:** O Payment Processor internamente tem muitas partes: factory registry, regional factory, builder, processor, repository, notifier. Clientes externos (outros módulos do sistema) não devem conhecer essa complexidade. Crie um `PaymentService` (Facade) que expõe operações de alto nível.

**Requisitos:**
- `PaymentService` como Facade do subsistema de pagamento
- Operações: `charge(request)`, `refund(txId, amount)`, `getTransaction(txId)`, `getStatement(customerId, dateRange)`
- Facade orquestra: validação → factory → builder → processor → repository → notifier
- Clientes usam apenas `PaymentService` — zero acesso a componentes internos

**API do Facade:**

| Operação | Entrada | Saída | Orquestra |
|----------|---------|-------|-----------|
| `charge(ChargeRequest)` | Customer + PaymentData + Amount | `PaymentResult` | Factory → Builder → Processor → Repo → Notifier |
| `refund(txId, amount)` | Transaction ID + valor | `RefundResult` | Repo.find → Processor.refund → Repo.save → Notifier |
| `getTransaction(txId)` | Transaction ID | `Transaction` | Repo.findById |
| `getStatement(customerId, range)` | Customer + date range | `List<Transaction>` | Repo.findByCustomer + filtro |

**Critérios de aceite:**
- [ ] `PaymentService` expõe 4 operações de alto nível
- [ ] Clientes não conhecem Factory, Builder, Processor, Repository internos
- [ ] Facade não adiciona lógica de negócio — apenas orquestra
- [ ] Cada operação do Facade delega para componentes do subsistema
- [ ] Testes: charge end-to-end, refund, consulta, statement com filtros

---

### Desafio 3.6 — Flyweight: Currency & Card Brand Metadata

> *"Usar compartilhamento para suportar eficientemente grandes quantidades de objetos granulares."*

**Cenário:** O sistema processa milhares de transações por segundo. Cada transação referencia metadados de `Currency` (símbolo, casas decimais, locale) e `CardBrand` (nome, BIN ranges, taxas padrão). Esses metadados são idênticos para todas as transações da mesma moeda/bandeira. Compartilhar esses objetos reduz uso de memória significativamente.

**Requisitos:**
- `CurrencyInfo` e `CardBrandInfo` como Flyweights (estado intrínseco compartilhado)
- `CurrencyInfoFactory` e `CardBrandInfoFactory` com pool/cache
- Estado extrínseco (específico da transação) passado como parâmetro nos métodos
- Benchmark mostrando economia de memória: N transações com compartilhamento vs sem

| Estado | Intrínseco (compartilhado) | Extrínseco (por transação) |
|--------|---------------------------|---------------------------|
| `CurrencyInfo` | Código, símbolo, casas decimais, locale | Valor da transação |
| `CardBrandInfo` | Nome, BIN ranges, taxa padrão | Número do cartão específico |

**Critérios de aceite:**
- [ ] Flyweight objects imutáveis e compartilhados
- [ ] Factory com cache (`Map`/`map`) — mesma chave retorna mesma instância
- [ ] Estado intrínseco no Flyweight, extrínseco passado por parâmetro
- [ ] Identidade: `factory.get("BRL") === factory.get("BRL")` (mesma referência)
- [ ] Benchmark: criar 100.000 transações com e sem Flyweight, medir memória
- [ ] Testes: compartilhamento verificado por identidade de referência

---

### Desafio 3.7 — Proxy: Gateway Protection & Lazy Loading

> *"Fornecer um substituto que controla acesso a outro objeto."*

**Cenário:** Implemente 3 tipos de Proxy sobre o `Gateway`:

| Tipo | Propósito | Comportamento |
|------|-----------|---------------|
| **Protection Proxy** | Autorização | Verifica se o caller tem permissão antes de processar |
| **Virtual Proxy** | Lazy Loading | Inicializa o gateway real (que é caro) apenas no primeiro uso |
| **Caching Proxy** | Cache | Retorna resultado cacheado para mesma idempotency key |

**Requisitos:**
- Todos implementam `Gateway` (mesma interface que o RealSubject)
- `ProtectionProxy` verifica role/permissions antes de delegar
- `VirtualProxy` atrasa `new RealGateway()` até o primeiro `process()` call
- `CachingProxy` armazena `Map<idempotencyKey, GatewayResponse>` (similar ao Decorator de idempotência, mas com foco diferente — Proxy controla **acesso**, Decorator adiciona **comportamento**)

**Proxy vs Decorator (documentar diferença):**

| Aspecto | Proxy (3.7) | Decorator (3.4) |
|---------|-------------|-----------------|
| **Propósito** | Controlar acesso | Adicionar funcionalidade |
| **Ciclo de vida** | Proxy gerencia a criação do real subject | Decorator recebe componente pronto |
| **Foco** | Segurança, lazy, cache | Logging, retry, métricas |

**Critérios de aceite:**
- [ ] 3 Proxies implementando `Gateway` interface
- [ ] `ProtectionProxy`: bloqueia se permissão insuficiente, delega se OK
- [ ] `VirtualProxy`: lazy init — gateway real criado apenas no primeiro `process()`
- [ ] `CachingProxy`: retorna cacheado se mesma key, senão delega e cacheia
- [ ] `DECISIONS.md`: diferença documentada entre Proxy e Decorator com exemplos concretos
- [ ] Testes: proteção bloqueia/permite, lazy init não cria até necessário, cache hit/miss

---

### Desafio 3.8 — Integração: Structural Composition

**Cenário:** Combine todos os padrões estruturais em um fluxo de processamento realista:

```
PaymentService (Facade)
  │
  ├── Gateway pipeline:
  │   VirtualProxy → ProtectionProxy → LoggingDecorator → RetryDecorator →
  │   StripeAdapter (Adapter) usando CurrencyInfo/CardBrandInfo (Flyweight)
  │
  ├── Split payment (Composite):
  │   SplitPayment
  │   ├── SinglePayment → Gateway A
  │   └── SinglePayment → Gateway B
  │
  └── Operation × Channel (Bridge):
      ChargeOperation → OnlineChannel
```

**Critérios de aceite:**
- [ ] Todos os 7 padrões estruturais integrados em um fluxo coeso
- [ ] Facade simplifica a complexidade para clientes externos
- [ ] Gateway pipeline com Adapter + Decorator + Proxy empilhados
- [ ] Split payments com Composite processados pelo pipeline
- [ ] `PATTERNS.md` com mapa de cada padrão → componente → motivação
- [ ] Testes end-to-end: charge simples, split payment, refund, consulta

---

## Entregáveis

| Artefato | Padrão |
|----------|--------|
| `StripeAdapter`, `PagSeguroAdapter`, `PayPalAdapter` | Adapter |
| `TransactionOperation` × `ProcessingChannel` | Bridge |
| `SplitPayment` + `SinglePayment` | Composite |
| `LoggingGateway`, `MetricsGateway`, `RetryGateway`, `IdempotencyGateway` | Decorator |
| `PaymentService` | Facade |
| `CurrencyInfoFactory`, `CardBrandInfoFactory` | Flyweight |
| `ProtectionProxy`, `VirtualProxy`, `CachingProxy` | Proxy |
| Pipeline integrado | Composição de todos |
| `PATTERNS.md`, `DECISIONS.md` | Documentação |

---

## Próximo Nível

→ [Level 4 — Padrões Comportamentais](04-behavioral-patterns.md)
