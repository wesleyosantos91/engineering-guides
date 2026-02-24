# Padrões Arquiteturais (Architectural Patterns)

> **Agnóstico a linguagem** — Padrões arquiteturais definem a **estrutura fundamental**
> de um sistema de software, especificando como componentes se organizam e interagem
> em nível macro. Diferem de Design Patterns por atuarem no nível da **aplicação inteira**,
> não de classes individuais.

---

## Sumário

- [Visão Geral](#visão-geral)
- [Layered Architecture](#layered-architecture)
- [Hexagonal Architecture (Ports & Adapters)](#hexagonal-architecture-ports--adapters)
- [Clean Architecture](#clean-architecture)
- [MVC — Model-View-Controller](#mvc--model-view-controller)
- [CQRS — Command Query Responsibility Segregation](#cqrs--command-query-responsibility-segregation)
- [Event Sourcing](#event-sourcing)
- [Event-Driven Architecture](#event-driven-architecture)
- [Comparativo](#comparativo)
- [Quando usar qual](#quando-usar-qual)

---

## Visão Geral

| Padrão | Intenção |
|--------|----------|
| **Layered** | Organizar o sistema em **camadas** horizontais com responsabilidades bem definidas |
| **Hexagonal** | Isolar o **domínio** de detalhes técnicos através de portas e adaptadores |
| **Clean Architecture** | Dependências apontam para dentro — o **domínio** é o centro |
| **MVC** | Separar **dados**, **apresentação** e **controle** de fluxo |
| **CQRS** | Separar modelos de **leitura** e **escrita** |
| **Event Sourcing** | Persistir o **histórico de eventos** em vez do estado atual |
| **Event-Driven** | Comunicação entre componentes via **eventos assíncronos** |

---

## Layered Architecture

### Intenção

Organizar o sistema em camadas horizontais, onde cada camada tem uma responsabilidade
específica e só pode depender de camadas **inferiores** (ou adjacentes).

### Estrutura

```
┌──────────────────────────────┐
│       Presentation Layer      │  UI, Controllers, APIs
├──────────────────────────────┤
│       Application Layer       │  Casos de uso, Orquestração
├──────────────────────────────┤
│         Domain Layer          │  Entidades, Regras de negócio
├──────────────────────────────┤
│      Infrastructure Layer     │  Banco, APIs externas, I/O
└──────────────────────────────┘
```

### Camadas típicas

| Camada | Responsabilidade | Depende de |
|--------|------------------|------------|
| **Presentation** | Interação com o usuário/cliente; serialização de entrada/saída | Application |
| **Application** | Orquestra casos de uso; não contém regras de negócio | Domain |
| **Domain** | Regras de negócio puras; entidades e value objects | Nada (centro) |
| **Infrastructure** | Implementações técnicas (DB, HTTP clients, filas) | Domain (implementa interfaces) |

### Regras

1. **Dependência unidirecional:** camada superior depende da inferior, nunca o contrário.
2. **Não pular camadas:** Presentation não deve acessar Infrastructure diretamente.
3. **Interfaces nas fronteiras:** use interfaces para desacoplar camadas.

### Benefícios

- Fácil de entender e comunicar para equipes.
- Separação clara de responsabilidades.
- Cada camada pode ser testada independentemente.

### Trade-offs

- Pode gerar "passthrough" desnecessário (a camada apenas delega).
- Rigidez: nem toda aplicação precisa de todas as camadas.
- Sem inversão de dependência, Infrastructure fica acoplada ao Domain.

### Sinais de que a arquitetura precisa evoluir

| Sinal | Ação sugerida |
|-------|---------------|
| Regras de negócio dependem de frameworks/banco | Migrar para Hexagonal ou Clean |
| Testes unitários precisam de banco/API real | Extrair Ports & Adapters |
| Camadas se tornam "passthrough" sem lógica | Simplificar ou colapsar camadas |
| Múltiplos serviços precisam se comunicar | Considerar Event-Driven |

> **Veja também:** [DIP em SOLID](solid-principles.md#d--dependency-inversion-principle-dip), [Separation of Concerns](best-practices.md#separation-of-concerns)

---

## Hexagonal Architecture (Ports & Adapters)

### Intenção

Permitir que o **domínio** (lógica de negócio) seja completamente isolado dos detalhes
técnicos (frameworks, banco de dados, APIs externas), usando **portas** (interfaces)
e **adaptadores** (implementações).

### Origem

Proposta por **Alistair Cockburn** (2005). Também conhecida como "Ports and Adapters".

### Estrutura

```
           ┌─────────────────────────────┐
           │      ADAPTERS (Driving)      │
           │  REST Controller, CLI, gRPC  │
           └──────────┬──────────────────┘
                      │ (Port In)
           ┌──────────▼──────────────────┐
           │        APPLICATION           │
           │     (Use Cases / Services)   │
           │                              │
           │  ┌──────────────────────┐   │
           │  │       DOMAIN          │   │
           │  │  Entities, VOs, Rules │   │
           │  └──────────────────────┘   │
           └──────────┬──────────────────┘
                      │ (Port Out)
           ┌──────────▼──────────────────┐
           │     ADAPTERS (Driven)        │
           │  DB Repository, HTTP Client  │
           │  Message Publisher, Cache     │
           └─────────────────────────────┘
```

### Conceitos-chave

| Conceito | Descrição |
|----------|-----------|
| **Port** | Interface que define um contrato (entrada ou saída do domínio) |
| **Adapter** | Implementação concreta de uma Port (traduz tecnologia ↔ domínio) |
| **Driving Adapter (Primary)** | Inicia a interação (ex: REST controller, CLI, test) |
| **Driven Adapter (Secondary)** | É chamado pelo domínio (ex: repository, HTTP client, publisher) |
| **Hexágono** | O núcleo — contém domínio + application services |

### Regras de dependência

1. **O domínio NÃO depende de nada externo** — ele define portas (interfaces).
2. **Adapters dependem do domínio** — implementam as portas.
3. **Frameworks ficam nos adapters** — nunca no domínio.

### Benefícios

- **Testabilidade:** o domínio é testável sem infraestrutura real.
- **Substituibilidade:** trocar banco de dados ou framework = trocar um adapter.
- **Foco no negócio:** o domínio é livre de preocupações técnicas.

### Trade-offs

- Mais interfaces e classes que uma arquitetura simples.
- Curva de aprendizado para equipes acostumadas com layered.
- Para aplicações muito simples (CRUD), pode ser over-engineering.

### Relação com Design Patterns

| Pattern | Uso na Hexagonal |
|---------|------------------|
| [Adapter](structural-patterns.md#adapter) | Driven/Driving adapters implementam as portas |
| [Strategy](behavioral-patterns.md#strategy) | Portas são essencialmente o padrão Strategy |
| [Factory](creational-patterns.md#factory-method) | Cria adapters específicos por ambiente (test, prod) |
| [Facade](structural-patterns.md#facade) | Application Service atua como Facade do domínio |
| [Decorator](structural-patterns.md#decorator) | Adiciona cross-cutting concerns nos adapters |

> **Veja também:** [DIP em SOLID](solid-principles.md#d--dependency-inversion-principle-dip), [Program to an Interface](best-practices.md#program-to-an-interface-not-an-implementation)

---

## Clean Architecture

### Intenção

Organizar o sistema em **camadas concêntricas** onde as dependências apontam
sempre para **dentro** (em direção ao domínio). Proposta por **Robert C. Martin** (2012).

### Estrutura

```
┌─────────────────────────────────────────────┐
│              Frameworks & Drivers            │  ← camada mais externa
│  (Web, UI, DB, Devices, External APIs)       │
├─────────────────────────────────────────────┤
│           Interface Adapters                 │
│  (Controllers, Gateways, Presenters)         │
├─────────────────────────────────────────────┤
│            Application Business              │
│         (Use Cases / Interactors)            │
├─────────────────────────────────────────────┤
│          Enterprise Business Rules           │  ← camada mais interna
│        (Entities / Domain Objects)            │
└─────────────────────────────────────────────┘

         Dependências apontam → para DENTRO
```

### The Dependency Rule

> *"Dependências de código-fonte devem apontar apenas para dentro,
> em direção a políticas de nível mais alto."*

| Camada | Pode depender de | NÃO pode depender de |
|--------|-------------------|----------------------|
| **Entities** | Nada | Application, Interface Adapters, Frameworks |
| **Use Cases** | Entities | Interface Adapters, Frameworks |
| **Interface Adapters** | Use Cases, Entities | Frameworks (idealmente) |
| **Frameworks** | Tudo acima | — |

### Componentes

| Componente | Responsabilidade |
|------------|------------------|
| **Entities** | Regras de negócio mais gerais e de alto nível (independem da aplicação) |
| **Use Cases** | Regras de negócio **específicas da aplicação**; orquestram entidades |
| **Interface Adapters** | Convertem dados entre formato do Use Case e formato externo |
| **Frameworks & Drivers** | Detalhes técnicos — banco, web framework, UI |

### Clean Architecture vs Hexagonal

| Aspecto | Clean Architecture | Hexagonal |
|---------|-------------------|-----------|
| **Camadas** | 4 anéis concêntricos | Hexágono + adapters |
| **Autor** | Robert C. Martin | Alistair Cockburn |
| **Foco** | Dependency Rule formal | Ports & Adapters |
| **Essência** | São muito similares na prática |

### Trade-offs

- Verbosidade: muitas camadas, muitas conversões de dados.
- Para CRUDs simples, o custo pode não compensar.
- Mappers entre camadas podem gerar muito código boilerplate.
- O benefício é proporcional à **complexidade do domínio**.

### Relação com Design Patterns

| Pattern | Uso na Clean Architecture |
|---------|---------------------------|
| [Strategy](behavioral-patterns.md#strategy) | Use Cases podem usar strategies para variações de regras |
| [Command](behavioral-patterns.md#command) | Use Cases frequentemente são modelados como Commands |
| [Observer](behavioral-patterns.md#observer) | Domain Events para notificação entre Use Cases |
| [Adapter](structural-patterns.md#adapter) | Interface Adapters convertem entre domínio e frameworks |
| [Factory](creational-patterns.md#factory-method) | Criação de entidades e value objects complexos |

> **Veja também:** [SOLID](solid-principles.md) (Clean Architecture é uma aplicação direta dos princípios SOLID em nível arquitetural)

---

## MVC — Model-View-Controller

### Intenção

Separar a aplicação em três componentes interconectados: **dados** (Model),
**apresentação** (View) e **lógica de controle** (Controller).

### Estrutura

```
┌──────────────┐
│    View       │ ← Apresentação (UI / Template / Response)
└──────┬───────┘
       │ atualiza
       │
┌──────▼───────┐     ┌──────────────┐
│  Controller   │────▶│    Model      │ ← Dados + Regras de negócio
│ (lógica de    │     │              │
│  controle)    │◀────│              │
└──────────────┘     └──────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Model** | Representa dados e regras de negócio; notifica sobre mudanças |
| **View** | Renderiza os dados do Model para o usuário |
| **Controller** | Recebe input do usuário; manipula o Model; seleciona a View |

### Variantes

| Variante | Descrição |
|----------|-----------|
| **MVC clássico** | View observa o Model diretamente |
| **MVC Web** | Controller recebe HTTP request, manipula Model, retorna View/JSON |
| **MVP (Model-View-Presenter)** | Presenter substitui Controller; View é passiva |
| **MVVM (Model-View-ViewModel)** | ViewModel expõe dados reativos; data binding bidirecional |

### Quando usar

- Aplicações com interface de usuário (web, desktop, mobile).
- Separação entre equipes de frontend e backend.
- Quando o mesmo Model deve ser apresentado de formas diferentes.

### Trade-offs

- Controllers podem crescer demais ("fat controller") — extraia lógica para services.
- MVC não define claramente onde ficam as regras de negócio complexas.
- Para APIs puras (sem UI), prefira arquiteturas baseadas em camadas ou hexagonal.

### Sinais de evolução

| Sinal | Ação sugerida |
|-------|---------------|
| Controllers com centenas de linhas | Extrair para Services (Application Layer) |
| Model contendo lógica de apresentação | Separar DTOs/ViewModels |
| Regras de negócio em Controllers | Migrar para domínio (Hexagonal/Clean) |

> **Veja também:** [Observer](behavioral-patterns.md#observer) (Model-View usa Observer), [Strategy](behavioral-patterns.md#strategy), [Facade](structural-patterns.md#facade)

---

## CQRS — Command Query Responsibility Segregation

### Intenção

Separar as operações de **leitura** (Query) das operações de **escrita** (Command)
em modelos distintos, permitindo otimizar cada lado independentemente.

### Origem

Proposto por **Greg Young**, baseado no princípio CQS (Command-Query Separation) de Bertrand Meyer.

### Estrutura

```
                    ┌──────────────────┐
           ┌───────│      Client       │───────┐
           │       └──────────────────┘       │
           ▼                                   ▼
   ┌───────────────┐                  ┌───────────────┐
   │   Command      │                  │    Query       │
   │   (Write)      │                  │    (Read)      │
   ├───────────────┤                  ├───────────────┤
   │ Command Handler│                  │ Query Handler  │
   │ Domain Logic   │                  │ Read Model     │
   │ Write Model    │                  │ (otimizado)    │
   └───────┬───────┘                  └───────┬───────┘
           │                                   │
           ▼                                   ▼
   ┌───────────────┐                  ┌───────────────┐
   │  Write Store   │ ──sync/event──▶ │  Read Store    │
   │ (normalizado)  │                  │(desnormalizado)│
   └───────────────┘                  └───────────────┘
```

### Conceitos-chave

| Conceito | Descrição |
|----------|-----------|
| **Command** | Intenção de alterar estado; não retorna dados |
| **Query** | Solicitação de dados; não altera estado |
| **Write Model** | Otimizado para consistência e regras de negócio |
| **Read Model** | Otimizado para performance de consulta (desnormalizado, projeções) |
| **Sincronização** | Read model pode ser eventual consistency via eventos |

### Níveis de CQRS

| Nível | Descrição |
|-------|-----------|
| **Lógico** | Separação em código (handlers diferentes) mas mesmo banco |
| **Banco separado** | Write DB e Read DB distintos; sincronização via eventos |
| **Com Event Sourcing** | Write side usa Event Store; Read side reconstrói a partir dos eventos |

### Quando usar

- A complexidade do domínio é alta (muitas regras de escrita).
- Perfis de leitura e escrita são muito diferentes (ex: muitas leituras, poucas escritas).
- Quer escalar leitura e escrita independentemente.
- Precisa de múltiplas projeções dos mesmos dados.

### Trade-offs

- **Complexidade operacional:** manter dois modelos sincronizados.
- **Eventual consistency:** o read model pode estar desatualizado.
- **Over-engineering:** para CRUDs simples, é excessivo.
- **CQRS não é um substituto para bom design de domínio.**

### Relação com Design Patterns

| Pattern | Uso no CQRS |
|---------|-------------|
| [Command](behavioral-patterns.md#command) | Commands encapsulam intenções de escrita |
| [Mediator](behavioral-patterns.md#mediator) | Roteia commands/queries para handlers (ex: MediatR) |
| [Observer](behavioral-patterns.md#observer) | Eventos sincronizam read/write models |
| [Strategy](behavioral-patterns.md#strategy) | Diferentes estratégias de leitura/projeção |

> **Veja também:** [Event Sourcing](#event-sourcing), [Command Pattern](behavioral-patterns.md#command)

---

## Event Sourcing

### Intenção

Persistir o **histórico completo de eventos** que levaram ao estado atual,
em vez de armazenar apenas o último estado.

### Estrutura

```
┌──────────────────────────────────────┐
│            Event Store                │
├──────────────────────────────────────┤
│ Event 1: AccountCreated(id, name)    │
│ Event 2: MoneyDeposited(id, 100)     │
│ Event 3: MoneyWithdrawn(id, 30)      │
│ Event 4: MoneyDeposited(id, 50)      │
├──────────────────────────────────────┤
│ Estado atual = replay dos eventos:    │
│ Saldo = 0 + 100 - 30 + 50 = 120     │
└──────────────────────────────────────┘
```

### Conceitos-chave

| Conceito | Descrição |
|----------|-----------|
| **Event** | Fato imutável que aconteceu (passado); ex: `OrderPlaced` |
| **Event Store** | Repositório append-only de eventos |
| **Aggregate** | Reconstrói seu estado a partir da sequência de eventos |
| **Snapshot** | Foto do estado em um ponto; otimiza o replay |
| **Projection** | Read model construído a partir dos eventos (materialização) |

### Quando usar

- Auditoria completa é requisito (financeiro, jurídico, compliance).
- Precisa responder "como chegamos neste estado?".
- Domínio é event-driven by nature (transações, workflows).
- Integração com CQRS para otimizar leituras.

### Trade-offs

- **Complexidade:** replay, snapshots, projeções, versionamento de eventos.
- **Eventual consistency:** projeções podem estar desatualizadas.
- **Evolução de schema:** eventos são imutáveis — upcasting/versioning é necessário.
- **Debugging:** entender o estado requer reproduzir o histórico.
- **Storage:** o volume de eventos pode crescer significativamente.

### Relação com Design Patterns

| Pattern | Uso no Event Sourcing |
|---------|-----------------------|
| [Memento](behavioral-patterns.md#memento) | Snapshots são essencialmente Mementos do aggregate |
| [Observer](behavioral-patterns.md#observer) | Projections são observers dos eventos |
| [Command](behavioral-patterns.md#command) | Commands geram eventos ao serem processados |

> **Veja também:** [CQRS](#cqrs--command-query-responsibility-segregation), [Imutabilidade](best-practices.md#imutabilidade)

---

## Event-Driven Architecture

### Intenção

Projetar sistemas onde componentes se comunicam através de **eventos assíncronos**,
promovendo desacoplamento e reatividade.

### Estrutura

```
┌──────────┐     ┌────────────────┐     ┌──────────────┐
│ Producer  │────▶│  Event Bus /    │────▶│  Consumer A   │
│ (Emite)   │     │  Message Broker │     └──────────────┘
└──────────┘     │                │     ┌──────────────┐
                 │ (Kafka, SQS,   │────▶│  Consumer B   │
                 │  RabbitMQ...)   │     └──────────────┘
                 └────────────────┘
```

### Tipos de eventos

| Tipo | Descrição | Exemplo |
|------|-----------|---------|
| **Domain Event** | Algo significativo que aconteceu no domínio | `OrderPlaced`, `UserRegistered` |
| **Integration Event** | Comunica mudanças entre bounded contexts/serviços | `PaymentConfirmed` |
| **Notification Event** | Apenas notifica (consumidor busca detalhes se quiser) | `OrderChanged(id)` |
| **Event-Carried State Transfer** | Carrega dados suficientes para o consumidor não precisar consultar a origem | `OrderPlaced(id, items, total, customer)` |

### Padrões de entrega

| Padrão | Garantia |
|--------|----------|
| **At-most-once** | Pode perder eventos; sem duplicatas |
| **At-least-once** | Não perde eventos; pode duplicar → consumidor deve ser **idempotente** |
| **Exactly-once** | Complexo de implementar; geralmente requer transações de infraestrutura |

### Quando usar

- Microserviços que precisam se comunicar sem acoplamento direto.
- Processamento assíncrono (filas de trabalho, pipelines).
- Sistemas reativos que respondem a mudanças de estado.
- Integração entre bounded contexts em DDD.

### Trade-offs

- **Complexidade operacional:** infraestrutura de messaging (Kafka, RabbitMQ, SQS).
- **Eventual consistency:** difícil de garantir consistência imediata.
- **Debugging:** rastrear fluxo de eventos distribuídos é desafiador.
- **Ordering:** garantir ordem de eventos pode ser complexo.
- **Idempotência:** consumidores devem tratar duplicatas.

### Relação com Design Patterns

| Pattern | Uso no Event-Driven |
|---------|---------------------|
| [Observer](behavioral-patterns.md#observer) | Pub/Sub é Observer distribuído |
| [Mediator](behavioral-patterns.md#mediator) | Message Broker atua como mediador |
| [Chain of Responsibility](behavioral-patterns.md#chain-of-responsibility) | Pipelines de processamento de eventos |
| [Command](behavioral-patterns.md#command) | Eventos podem carregar intenções de ação |

> **Veja também:** [Observer](behavioral-patterns.md#observer), [Defensive Programming](best-practices.md#defensive-programming) (idempotência é essencial)

---

## Combinações Práticas

Arquiteturas reais frequentemente combinam múltiplos padrões:

| Combinação | Quando usar | Complexidade |
|-----------|-------------|---------------|
| **Hexagonal + CQRS** | Domínio rico com perfis de leitura/escrita distintos | Alta |
| **Clean + Event Sourcing** | Domínio complexo com requisitos de auditoria | Muito alta |
| **Layered + MVC** | Aplicações web tradicionais | Baixa |
| **Hexagonal + Event-Driven** | Microserviços com domínio rico | Alta |
| **CQRS + Event Sourcing + Event-Driven** | Sistemas distribuídos com auditoria completa | Muito alta |
| **Layered + CQRS (lógico)** | CQRS simplificado sem banco separado | Média |

### Guia de evolução progressiva

```
Início: Layered Architecture simples
│
├── Domínio ficou complexo?
│   └── Evoluir para Hexagonal
│
├── Leitura e escrita com perfis muito diferentes?
│   └── Adicionar CQRS (primeiro lógico, depois físico)
│
├── Auditoria/histórico é requisito?
│   └── Adotar Event Sourcing (geralmente + CQRS)
│
└── Múltiplos serviços precisam se comunicar?
    └── Adotar Event-Driven Architecture
```

---

## Comparativo

| Aspecto | Layered | Hexagonal | Clean | MVC | CQRS | Event Sourcing | Event-Driven |
|---------|---------|-----------|-------|-----|------|----------------|--------------|
| **Escopo** | Aplicação | Aplicação | Aplicação | UI Layer | Read/Write | Persistência | Sistema |
| **Complexidade** | Baixa | Média | Alta | Baixa | Alta | Muito alta | Alta |
| **Testabilidade** | Boa | Excelente | Excelente | Boa | Boa | Boa | Média |
| **Quando** | Maioria | Domínio rico | Domínio complexo | Apps com UI | Alta escala R/W | Auditoria | Microserviços |
| **Curva** | Suave | Moderada | Íngreme | Suave | Moderada | Íngreme | Moderada |

---

## Quando usar qual

```
Qual é a natureza do sistema?
│
├── Aplicação simples / CRUD?
│   └── Layered Architecture (3-4 camadas)
│
├── Domínio rico com regras complexas?
│   ├── Muitas integrações externas?
│   │   └── Hexagonal (Ports & Adapters)
│   └── Máxima separação de concerns?
│       └── Clean Architecture
│
├── Aplicação com interface visual?
│   └── MVC / MVP / MVVM
│
├── Leituras e escritas têm perfis MUITO diferentes?
│   └── CQRS
│
├── Auditoria completa / histórico é requisito?
│   └── Event Sourcing (geralmente + CQRS)
│
└── Componentes distribuídos que precisam se comunicar?
    └── Event-Driven Architecture
```

> **Nota:** Esses padrões **não são mutuamente exclusivos**. É comum combinar:
> - Clean Architecture + CQRS
> - Hexagonal + Event Sourcing
> - Event-Driven + CQRS + Event Sourcing
> - Layered + MVC
