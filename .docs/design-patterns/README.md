# Design Patterns — Guia Teórico (Agnóstico a Linguagem)

> **Objetivo deste documento:** Servir como referência teórica completa sobre Design Patterns,
> princípios SOLID e boas práticas de design orientado a objetos.
> Todos os conceitos são apresentados de forma agnóstica a linguagem de programação —
> focando em **quando usar**, **por que usar** e **quais trade-offs** considerar.

> **Uso como Knowledge Base:** Este conjunto de documentos é projetado para servir como
> base de conhecimento para assistentes de código (Copilot). Cada documento é autocontido
> mas interligado, com cross-references, árvores de decisão e relações explícitas entre
> padrões, princípios e práticas.

---

## Sumário Geral

| Documento | Descrição | Palavras-chave |
|-----------|-----------|----------------|
| [Princípios SOLID](solid-principles.md) | Os 5 princípios fundamentais de design OO | SRP, OCP, LSP, ISP, DIP, responsabilidade, extensão, substituição, segregação, inversão |
| [Padrões Criacionais](creational-patterns.md) | Factory, Abstract Factory, Builder, Prototype, Singleton | instanciação, criação de objetos, fábrica, construção, clonagem, instância única |
| [Padrões Estruturais](structural-patterns.md) | Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy | composição, adaptação, wrapper, hierarquia, simplificação, cache, controle de acesso |
| [Padrões Comportamentais](behavioral-patterns.md) | Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor | algoritmo, notificação, evento, estado, cadeia, undo, redo, travessia, coordenação |
| [Padrões Arquiteturais](architectural-patterns.md) | MVC, Hexagonal, Clean Architecture, CQRS, Event Sourcing, Layered | arquitetura, camadas, portas, adaptadores, domínio, leitura, escrita, eventos |
| [Boas Práticas de Design](best-practices.md) | Composição vs Herança, DRY, KISS, YAGNI, Lei de Demeter, Tell Don't Ask | composição, simplicidade, duplicação, encapsulamento, coesão, acoplamento, imutabilidade |

---

## Como navegar

1. **Se você é iniciante:** comece por [Princípios SOLID](solid-principles.md) e depois [Boas Práticas](best-practices.md).
2. **Se quer resolver um problema específico:** vá direto ao grupo de padrões (Criacional, Estrutural ou Comportamental).
3. **Se está projetando a arquitetura:** consulte [Padrões Arquiteturais](architectural-patterns.md).
4. **Se está fazendo code review:** use o [Checklist de Code Review](best-practices.md#checklist-de-code-review) em Boas Práticas.

---

## Guia Rápido de Decisão

```
Qual é o seu problema?
│
├── Preciso CRIAR objetos de forma flexível?
│   └── → Padrões Criacionais (creational-patterns.md)
│       ├── Apenas uma instância? → Singleton
│       ├── Família de objetos? → Abstract Factory
│       ├── Delegar qual tipo criar? → Factory Method
│       ├── Objeto complexo com parâmetros opcionais? → Builder
│       └── Copiar objeto existente? → Prototype
│
├── Preciso COMPOR objetos em estruturas maiores?
│   └── → Padrões Estruturais (structural-patterns.md)
│       ├── Interfaces incompatíveis? → Adapter
│       ├── Duas dimensões de variação? → Bridge
│       ├── Hierarquia parte-todo (árvore)? → Composite
│       ├── Adicionar comportamento dinamicamente? → Decorator
│       ├── Simplificar subsistema complexo? → Facade
│       ├── Muitos objetos similares em memória? → Flyweight
│       └── Controlar acesso (lazy load, segurança, cache)? → Proxy
│
├── Preciso organizar COMPORTAMENTO entre objetos?
│   └── → Padrões Comportamentais (behavioral-patterns.md)
│       ├── Algoritmo intercambiável? → Strategy
│       ├── Notificar múltiplos observers? → Observer
│       ├── Encapsular ação como objeto (undo, fila)? → Command
│       ├── Esqueleto fixo com passos variáveis? → Template Method
│       ├── Comportamento muda com estado? → State
│       ├── Cadeia de handlers/middlewares? → Chain of Responsibility
│       ├── Coordenar interação complexa? → Mediator
│       ├── Percorrer coleção sem expor implementação? → Iterator
│       ├── Salvar e restaurar estado (checkpoint)? → Memento
│       └── Operações sobre estrutura sem alterar classes? → Visitor
│
├── Preciso definir a ARQUITETURA do sistema?
│   └── → Padrões Arquiteturais (architectural-patterns.md)
│       ├── App simples/CRUD? → Layered
│       ├── Domínio rico com integrações? → Hexagonal
│       ├── Máxima separação de concerns? → Clean Architecture
│       ├── App com UI? → MVC/MVP/MVVM
│       ├── Leitura ≠ Escrita em perfil? → CQRS
│       ├── Auditoria/histórico completo? → Event Sourcing
│       └── Componentes distribuídos? → Event-Driven
│
└── Preciso melhorar a QUALIDADE do design?
    └── → Princípios SOLID (solid-principles.md) + Boas Práticas (best-practices.md)
```

---

## Mapa de Relações entre Padrões

| Padrão | Frequentemente usado com | Alternativa a |
|--------|--------------------------|---------------|
| **Factory Method** | Template Method, Abstract Factory | `new` direto, Prototype |
| **Abstract Factory** | Factory Method, Singleton | Builder (escopo diferente) |
| **Builder** | Composite (construir árvores) | Construtor telescópico |
| **Singleton** | Abstract Factory, Facade | DI com escopo singleton |
| **Adapter** | Facade, Bridge | Mudança da interface original |
| **Bridge** | Abstract Factory | Herança múltipla |
| **Composite** | Iterator, Visitor, Decorator | Estruturas planas |
| **Decorator** | Strategy, Composite, Chain of Resp. | Herança |
| **Facade** | Singleton, Abstract Factory | Acesso direto ao subsistema |
| **Proxy** | Decorator, Adapter | Acesso direto ao objeto |
| **Strategy** | Factory Method, State | if/else, switch/case |
| **Observer** | Mediator, Command | Polling, acoplamento direto |
| **Command** | Memento, Strategy | Chamada direta de método |
| **State** | Strategy, Singleton (states compartilhados) | switch/case em estado |
| **Chain of Responsibility** | Composite, Decorator | if/else em cascata |
| **Mediator** | Observer, Facade | Comunicação direta N:N |

---

## Referências Bibliográficas

| Livro | Autor(es) | Ano |
|-------|-----------|-----|
| *Design Patterns: Elements of Reusable Object-Oriented Software* | Gamma, Helm, Johnson, Vlissides (Gang of Four) | 1994 |
| *Head First Design Patterns* | Freeman & Robson | 2004 |
| *Clean Architecture* | Robert C. Martin | 2017 |
| *Patterns of Enterprise Application Architecture* | Martin Fowler | 2002 |
| *Domain-Driven Design* | Eric Evans | 2003 |
| *Refactoring: Improving the Design of Existing Code* | Martin Fowler | 2018 |
| *Implementing Domain-Driven Design* | Vaughn Vernon | 2013 |

