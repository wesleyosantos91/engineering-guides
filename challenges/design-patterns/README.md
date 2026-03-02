# Zero to Hero — Design Patterns Challenge (Java 25 + Go 1.26)

> **Programa de especialização progressiva** em Design Patterns, implementado em **linguagem pura**:
> **Java 25** (sem Spring, Quarkus, Micronaut, Jakarta EE) · **Go 1.26** (sem Gin, Echo, Fiber)

---

## 1. Resumo Executivo

Este programa transforma um desenvolvedor em **especialista em design de software** através de desafios práticos implementados em **Java 25 e Go 1.26 puros** — sem nenhum framework. O foco é dominar os fundamentos de design orientado a objetos, princípios SOLID, padrões GoF (criacionais, estruturais, comportamentais), padrões arquiteturais e boas práticas de engenharia de software.

**Domínio escolhido:** Sistema de **Processador de Pagamentos (Payment Processor)** — um domínio rico que permite aplicar todos os padrões de design de forma natural e realista.

**Por que Payment Processor?**
- Domínio naturalmente complexo — envolve **múltiplas estratégias** (métodos de pagamento), **estados** (ciclo de vida da transação), **cadeias de validação** (fraude, limites), **notificações** (observers), **construção complexa** (builders de transações)
- Cada padrão GoF encontra **aplicação orgânica** no domínio — não são exercícios artificiais
- Relevante para **qualquer empresa** — payments é infraestrutura crítica em fintechs, e-commerce, marketplaces
- Permite exercitar **composição de padrões** — Strategy + Factory + Observer em cenários reais
- Perguntas de entrevista Senior/Staff frequentemente envolvem design de sistemas de pagamento

**Por que linguagem pura (sem frameworks)?**
- Frameworks **escondem** padrões — você usa Strategy via `@Bean`, mas não sabe implementar
- Dominar padrões em linguagem pura dá **fundamento real** para entender os frameworks depois
- Java 25 oferece **records**, **sealed classes**, **pattern matching**, **virtual threads** — substitutos modernos para muitos padrões tradicionais
- Go 1.26 força **composição sobre herança**, **interfaces implícitas**, **goroutines** — uma visão idiomática e pragmática dos padrões
- O contraste Java vs Go revela **quando um padrão é essencial** e quando é artefato da linguagem

---

## 2. Mapeamento Padrão → Domínio

| Padrão | Aplicação no Payment Processor |
|--------|-------------------------------|
| **SRP** | Separar cálculo de taxa, validação, persistência, notificação |
| **OCP** | Novos métodos de pagamento sem alterar o processador |
| **LSP** | Todo `PaymentMethod` é substituível no pipeline de processamento |
| **ISP** | Interfaces segregadas: `Chargeable`, `Refundable`, `Recurring` |
| **DIP** | Core depende de abstrações (ports), não de implementações concretas |
| **Singleton** | Configuração global, connection pool, registry de métodos de pagamento |
| **Factory Method** | Criar o `PaymentMethod` correto baseado no tipo da transação |
| **Abstract Factory** | Famílias de processadores por região/bandeira (Visa BR, Visa US, etc.) |
| **Builder** | Construção de `Transaction` com dados opcionais (metadata, retry policy, etc.) |
| **Prototype** | Template de transações recorrentes clonadas a cada cobrança |
| **Adapter** | Integrar gateways externos com interface interna unificada |
| **Bridge** | Desacoplar abstração (TransactionProcessor) da implementação (Gateway) |
| **Composite** | Pagamentos compostos (split payment: parte cartão + parte saldo) |
| **Decorator** | Adicionar logging, retry, cache, métricas ao processamento |
| **Facade** | API simplificada do Payment Service para clientes externos |
| **Flyweight** | Cache de metadados de bandeiras/moedas/configurações compartilhadas |
| **Proxy** | Lazy loading de dados de gateway, protection proxy para autorização |
| **Strategy** | Estratégias de cálculo de taxa, roteamento de gateway, anti-fraude |
| **Observer** | Notificar sistemas (e-mail, webhook, audit log) após mudança de estado |
| **Command** | Encapsular operações (charge, refund, void) com undo e fila |
| **Template Method** | Esqueleto do fluxo de processamento (validate → authorize → capture → notify) |
| **State** | Ciclo de vida da transação (created → authorized → captured → settled → refunded) |
| **Chain of Responsibility** | Pipeline de validação (fraude → limites → saldo → compliance) |
| **Mediator** | Coordenar interação entre processador, gateway, notificador, auditor |
| **Iterator** | Percorrer histórico de transações com filtros customizados |
| **Memento** | Snapshots de transação para undo/restore (estorno parcial) |
| **Visitor** | Relatórios sobre transações: cálculo de impostos, reconciliação, analytics |
| **Hexagonal** | Domínio isolado com ports (interfaces) e adapters (implementações) |
| **Clean Architecture** | Layers: Entity → Use Case → Interface Adapter → Infrastructure |
| **CQRS** | Separar command (processar pagamento) de query (consultar extrato) |
| **Event Sourcing** | Histórico completo de eventos da transação como fonte de verdade |

---

## 3. Equivalências Java 25 ↔ Go 1.26

| Conceito | Java 25 | Go 1.26 |
|----------|---------|---------|
| **Interface** | `interface` (pode ter default methods) | `interface` (implícitas, duck typing) |
| **Classe abstrata** | `abstract class` | Não existe — usar composição + interface |
| **Herança** | `extends` | Não existe — usar embedding |
| **Composição** | Campo + delegação | Embedding de struct |
| **Enum** | `enum` com métodos | `const` + `iota` + tipo customizado |
| **Record/VO** | `record` (imutável) | `struct` (mutável por default) |
| **Sealed types** | `sealed interface permits` | Não nativo — convenção por pacote |
| **Pattern matching** | `switch` com patterns, `instanceof` | Type switch (`switch v := x.(type)`) |
| **Generics** | `<T extends Comparable<T>>` | `[T comparable]` |
| **Optional/Null safety** | `Optional<T>` | Retorno múltiplo `(T, error)` |
| **Closures / Lambdas** | `Function<T,R>`, `->` | Funções de primeira classe |
| **Concorrência** | Virtual Threads (`Thread.ofVirtual()`) | Goroutines + channels |
| **Imutabilidade** | `record`, `final`, `Collections.unmodifiable*` | Convenção (sem suporte nativo) |
| **Encapsulamento** | `private/protected/public` + modules | Exportação por letra maiúscula |
| **Coleções** | `List`, `Set`, `Map`, `Stream API` | Slices, Maps + `slices`/`maps` packages |
| **Iteração** | `Iterator`, `Stream`, `for-each` | `for range`, iterators (1.23+) |
| **Testes** | JUnit 5 + AssertJ + Mockito | `testing` package + `testify` |

---

## 4. Mapa de Níveis

```
Level 0 — Fundações OOP & Linguagem
  │       (Interfaces, composição, polimorfismo, tipos, testes)
  ▼
Level 1 — Princípios SOLID
  │       (SRP, OCP, LSP, ISP, DIP aplicados ao domínio Payment)
  ▼
Level 2 — Padrões Criacionais (GoF)
  │       (Singleton, Factory Method, Abstract Factory, Builder, Prototype)
  ▼
Level 3 — Padrões Estruturais (GoF)
  │       (Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy)
  ▼
Level 4 — Padrões Comportamentais (GoF)
  │       (Strategy, Observer, Command, Template Method, State, Chain, Mediator, Iterator, Memento, Visitor)
  ▼
Level 5 — Padrões Arquiteturais
  │       (Layered, Hexagonal, Clean, MVC, CQRS, Event Sourcing, Event-Driven)
  ▼
Level 6 — Boas Práticas & Integração de Padrões
  │       (Composição vs Herança, DRY, KISS, YAGNI, Demeter, Anti-patterns, Refactoring)
  ▼
Level 7 — Projeto Capstone: Payment Processor Completo
          (Todos os padrões integrados em um sistema coeso, production-grade)
```

---

## 5. Estrutura de Cada Desafio

Cada nível segue a mesma estrutura:

```
level-N-nome/
├── README.md                ← Desafio detalhado com instruções
├── java/                    ← Implementação Java 25 pura
│   ├── src/
│   │   ├── main/java/
│   │   └── test/java/
│   └── build.gradle         ← ou pom.xml (sem dependência de framework)
└── go/                      ← Implementação Go 1.26 pura
    ├── cmd/
    ├── internal/
    ├── go.mod
    └── *_test.go
```

### Dependências Permitidas

| Categoria | Java 25 | Go 1.26 |
|-----------|---------|---------|
| **Linguagem** | JDK 25 (standard library) | Go 1.26 (standard library) |
| **Testes** | JUnit 5, AssertJ, Mockito | `testing`, `testify` |
| **Build** | Gradle 8.x ou Maven 3.9+ | `go build` / `go test` |
| **Serialização** | Jackson (apenas para I/O) | `encoding/json` (stdlib) |
| **Logging** | `java.util.logging` ou SLF4J + Logback | `log/slog` (stdlib) |
| **HTTP** | `java.net.http.HttpClient` (stdlib) | `net/http` (stdlib) |
| **Frameworks** | **PROIBIDO** (sem Spring, Quarkus, Micronaut, Jakarta EE) | **PROIBIDO** (sem Gin, Echo, Fiber, Chi) |
| **DI containers** | **PROIBIDO** — DI manual via construtor | **PROIBIDO** — DI manual via construtor/funções |

---

## 6. Critérios Globais

### Qualidade de Código

| Critério | Java 25 | Go 1.26 |
|----------|---------|---------|
| **Idiomático** | Records, sealed, pattern matching, streams | Interfaces implícitas, embedding, error handling |
| **Testes** | ≥ 85% cobertura (JaCoCo) | ≥ 85% cobertura (`go test -cover`) |
| **Documentação** | Javadoc nos tipos públicos | GoDoc nos tipos exportados |
| **Linting** | Checkstyle + PMD ou SonarLint | `golangci-lint` |
| **Naming** | `PascalCase` tipos, `camelCase` membros | `PascalCase` exportado, `camelCase` interno |
| **Error handling** | Checked exceptions + `Optional<T>` | `(T, error)` pattern + `errors.Is/As` |

### Documentação Esperada

Cada nível deve conter:
- `README.md` — Descrição do desafio com critérios de aceite
- `DECISIONS.md` — Trade-offs e justificativas de design
- `PATTERNS.md` — Quais padrões foram aplicados, onde e por quê
- Diagramas UML (opcional mas recomendado) — Mermaid ou PlantUML

---

## 7. Referência Cruzada com Documentação

| Nível | Documento de Referência |
|-------|------------------------|
| Level 0 | Conceitos fundamentais OOP (pré-requisito) |
| Level 1 | [01-solid-principles.md](../../.docs/DESIGN-PATTERNS/01-solid-principles.md) |
| Level 2 | [02-creational-patterns.md](../../.docs/DESIGN-PATTERNS/02-creational-patterns.md) |
| Level 3 | [03-structural-patterns.md](../../.docs/DESIGN-PATTERNS/03-structural-patterns.md) |
| Level 4 | [04-behavioral-patterns.md](../../.docs/DESIGN-PATTERNS/04-behavioral-patterns.md) |
| Level 5 | [05-architectural-patterns.md](../../.docs/DESIGN-PATTERNS/05-architectural-patterns.md) |
| Level 6 | [06-best-practices.md](../../.docs/DESIGN-PATTERNS/06-best-practices.md) |
| Level 7 | Todos os documentos acima integrados |

---

## 8. Navegação

| Nível | Título | Padrões | Status |
|-------|--------|---------|--------|
| [Level 0](00-oop-foundations.md) | Fundações OOP & Linguagem | Interfaces, Composição, Polimorfismo | 🔲 |
| [Level 1](01-solid-principles.md) | Princípios SOLID | SRP, OCP, LSP, ISP, DIP | 🔲 |
| [Level 2](02-creational-patterns.md) | Padrões Criacionais | Singleton, Factory, Abstract Factory, Builder, Prototype | 🔲 |
| [Level 3](03-structural-patterns.md) | Padrões Estruturais | Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy | 🔲 |
| [Level 4](04-behavioral-patterns.md) | Padrões Comportamentais | Strategy, Observer, Command, Template Method, State, Chain, Mediator, Iterator, Memento, Visitor | 🔲 |
| [Level 5](05-architectural-patterns.md) | Padrões Arquiteturais | Layered, Hexagonal, Clean, MVC, CQRS, Event Sourcing, Event-Driven | 🔲 |
| [Level 6](06-best-practices-integration.md) | Boas Práticas & Integração | Composição, DRY, KISS, YAGNI, Demeter, Anti-patterns, Composição de Padrões | 🔲 |
| [Level 7](07-capstone-project.md) | Projeto Capstone | Todos os padrões em um sistema coeso | 🔲 |

---

## 9. Pré-requisitos

| Requisito | Java | Go |
|-----------|------|----|
| **Versão** | JDK 25 (Early Access ou GA) | Go 1.26 |
| **Build tool** | Gradle 8.x ou Maven 3.9+ | `go` toolchain |
| **IDE** | IntelliJ IDEA ou VS Code | GoLand ou VS Code (gopls) |
| **Git** | Git 2.x | Git 2.x |
| **Conhecimento prévio** | Sintaxe Java básica, OOP fundamental | Sintaxe Go básica, structs, interfaces |

---

> **Legenda de status:** 🔲 Não iniciado · 🔶 Em progresso · ✅ Completo
