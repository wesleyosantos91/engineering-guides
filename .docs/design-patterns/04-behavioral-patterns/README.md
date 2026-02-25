# Padrões Comportamentais (Behavioral Patterns)

> **Agnóstico a linguagem** — Padrões comportamentais se preocupam com **algoritmos**
> e a **atribuição de responsabilidades** entre objetos.
> Descrevem padrões de comunicação entre objetos e como o fluxo de controle é distribuído.

---

## Sumário

- [Visão Geral](#visão-geral)
- [Strategy](#strategy)
- [Observer](#observer)
- [Command](#command)
- [Template Method](#template-method)
- [State](#state)
- [Chain of Responsibility](#chain-of-responsibility)
- [Mediator](#mediator)
- [Iterator](#iterator)
- [Memento](#memento)
- [Visitor](#visitor)
- [Comparativo](#comparativo)
- [Quando usar qual](#quando-usar-qual)

---

## Visão Geral

| Padrão | Intenção |
|--------|----------|
| **Strategy** | Definir família de algoritmos, encapsulá-los e torná-los intercambiáveis |
| **Observer** | Definir dependência um-para-muitos para que quando um objeto muda, todos os dependentes sejam notificados |
| **Command** | Encapsular uma requisição como um objeto, permitindo parametrização e enfileiramento |
| **Template Method** | Definir o esqueleto de um algoritmo, delegando alguns passos para subclasses |
| **State** | Permitir que um objeto altere seu comportamento quando seu estado interno muda |
| **Chain of Responsibility** | Evitar acoplamento entre emissor e receptor, dando a múltiplos objetos chance de tratar a requisição |
| **Mediator** | Definir um objeto que encapsula como um conjunto de objetos interage |
| **Iterator** | Prover acesso sequencial aos elementos de um agregado sem expor sua representação |
| **Memento** | Capturar e externalizar o estado interno de um objeto para restaurá-lo depois |
| **Visitor** | Representar uma operação a ser executada sobre elementos de uma estrutura, sem modificar as classes |

---

## Strategy

### Intenção

Definir uma **família de algoritmos**, encapsular cada um deles e torná-los **intercambiáveis**.
O Strategy permite que o algoritmo varie independentemente dos clientes que o utilizam.

### Problema que resolve

- O sistema precisa de diferentes variantes de um algoritmo.
- Blocos `if/else` ou `switch/case` selecionam o algoritmo — difícil de estender.
- O algoritmo pode mudar em tempo de execução.

### Estrutura

```
┌──────────────────┐         ┌──────────────────┐
│     Context       │────────▶│    Strategy      │
├──────────────────┤         │   (interface)    │
│ - strategy: Stg   │         ├──────────────────┤
│ + setStrategy()   │         │ + execute()      │
│ + doWork()        │         └───────▲──────────┘
└──────────────────┘                 │
                           ┌─────────┼─────────┐
                           │                    │
                   ┌───────┴──────┐    ┌───────┴──────┐
                   │ ConcreteStgA  │    │ ConcreteStgB  │
                   │ + execute()   │    │ + execute()   │
                   └──────────────┘    └──────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Strategy** | Interface comum para todos os algoritmos |
| **ConcreteStrategy** | Implementação de um algoritmo específico |
| **Context** | Mantém referência para Strategy; delega para ela |

### Quando usar

- Múltiplos algoritmos relacionados diferem apenas no comportamento.
- Quer evitar condicionais para selecionar o algoritmo.
- O algoritmo pode ser trocado em runtime.

### Exemplos do mundo real

- Estratégias de ordenação (quicksort, mergesort, heapsort).
- Estratégias de validação (por tipo de documento, por país).
- Estratégias de cálculo de frete (PAC, Sedex, transportadora).
- Estratégias de autenticação (JWT, OAuth, API Key).

### Trade-offs

- Clientes precisam conhecer as strategies disponíveis para escolher.
- Mais classes para algoritmos simples.
- Pode ser substituído por funções de primeira classe (lambdas) em linguagens funcionais.

### Sinais no código (quando considerar Strategy)

- Blocos `if/else` ou `switch/case` que selecionam algoritmos.
- Código que muda frequentemente para adicionar novas variantes de um comportamento.
- Classes com múltiplos métodos que diferem apenas no algoritmo interno.
- Necessidade de trocar algoritmo em runtime (ex: estratégia de preço por perfil de cliente).

### Relação com SOLID

| Princípio | Relação |
|-----------|--------|
| **OCP** | Novas strategies podem ser adicionadas sem alterar Context |
| **SRP** | Cada strategy encapsula um único algoritmo |
| **DIP** | Context depende da interface Strategy, não de implementações concretas |
| **LSP** | Todas as strategies são substituíveis via interface |

> **Veja também:** [State](#state) (similar em estrutura), [Template Method](#template-method) (alternativa via herança), [Factory Method](../02-creational-patterns/README.md#factory-method) (para criar a Strategy adequada), [OCP em SOLID](../01-solid-principles/README.md#o--openclosed-principle-ocp)

---

## Observer

### Intenção

Definir uma dependência **um-para-muitos** entre objetos, de modo que quando um objeto
muda de estado, todos os seus dependentes são **notificados e atualizados** automaticamente.

### Problema que resolve

- Múltiplos objetos precisam reagir quando outro objeto muda.
- O objeto que muda não deve conhecer os objetos que dependem dele (acoplamento baixo).
- O número de dependentes pode mudar dinamicamente.

### Estrutura

```
┌──────────────────┐         ┌──────────────────┐
│    Subject        │────────▶│    Observer      │
├──────────────────┤  *      │   (interface)    │
│ - observers[]    │         ├──────────────────┤
│ + attach(obs)     │         │ + update()       │
│ + detach(obs)     │         └───────▲──────────┘
│ + notify()        │                 │
└──────────────────┘         ┌───────┴──────────┐
                             │ ConcreteObserver  │
                             │ + update()        │
                             └──────────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Subject** | Conhece seus observers; fornece attach/detach/notify |
| **Observer** | Interface de notificação |
| **ConcreteObserver** | Reage à mudança do Subject |

### Variantes

| Variante | Descrição |
|----------|-----------|
| **Push** | Subject envia dados detalhados na notificação |
| **Pull** | Subject notifica; Observer busca os dados que precisa |
| **Event Bus / Message Broker** | Desacoplamento total via intermediário |
| **Reactive Streams** | Observer assíncrono com backpressure |

### Quando usar

- Quando a mudança em um objeto deve refletir em outros, sem saber quantos são.
- Sistemas de eventos, notificações, pub/sub.
- UI reativa (data binding).

### Trade-offs

- **Memory leaks:** observers não removidos mantêm referências (lapsed listener).
- **Ordem de notificação:** geralmente não é garantida.
- **Cascata:** um observer pode provocar notificações em cadeia (tempestade de eventos).
- **Debugging:** rastrear fluxo de eventos pode ser complexo.

### Sinais no código (quando considerar Observer)

- Polling (verificação periódica) para detectar mudanças de estado.
- Acoplamento direto entre quem produz dados e quem consume (múltiplas dependências diretas).
- Código que notifica manualmente cada consumidor um a um.
- Necessidade de adicionar novos "ouvintes" sem alterar o emissor.

### Relação com SOLID

| Princípio | Relação |
|-----------|--------|
| **OCP** | Novos observers podem ser adicionados sem alterar o Subject |
| **SRP** | Subject se preocupa com notificação; Observer com reação |
| **DIP** | Subject depende da interface Observer, não de implementações concretas |
| **ISP** | Interface Observer é minimalista (apenas `update()`) |

> **Veja também:** [Mediator](#mediator) (centraliza comunicação), [Event-Driven Architecture](../05-architectural-patterns/README.md#event-driven-architecture), [Command](#command)

---

## Command

### Intenção

Encapsular uma **requisição como um objeto**, permitindo parametrizar clientes
com diferentes requisições, enfileirar, registrar log e suportar **undo/redo**.

### Problema que resolve

- Precisa desacoplar quem invoca a operação de quem a executa.
- Quer suportar undo/redo, filas de operações ou log de transações.
- Operações devem ser tratadas como objetos de primeira classe.

### Estrutura

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  Invoker  │────▶│   Command    │────▶│  Receiver    │
│           │     │ (interface)  │     │ (executa)    │
└──────────┘     ├──────────────┤     └──────────────┘
                 │ + execute()  │
                 │ + undo()     │
                 └──────▲───────┘
                        │
                ┌───────┴────────┐
                │ConcreteCommand  │
                │ - receiver     │
                │ + execute()    │ ← delega para receiver
                │ + undo()       │
                └────────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Command** | Interface com `execute()` (e opcionalmente `undo()`) |
| **ConcreteCommand** | Liga uma ação ao Receiver; implementa execute/undo |
| **Invoker** | Pede ao Command para executar a requisição |
| **Receiver** | Sabe como executar a operação de fato |

### Quando usar

- Parametrizar objetos por ação a ser executada.
- Especificar, enfileirar e executar requisições em diferentes momentos.
- Suportar **undo/redo**.
- Suportar **logging** de operações para recovery.
- Estruturar transações de alto nível a partir de operações primitivas.

### Exemplos do mundo real

- Botões de UI que executam ações.
- Filas de jobs/tasks.
- Sistemas de transações bancárias.
- Command handlers em CQRS.

### Trade-offs

- Aumenta o número de classes (um Command por operação).
- Para operações simples, pode ser over-engineering.
- Undo complexo pode requerer armazenar muito estado.

### Sinais no código (quando considerar Command)

- Lógica de execução misturada com lógica de UI ou controle.
- Necessidade de undo/redo e não há mecanismo estruturado.
- Operações que precisam ser enfileiradas, agendadas ou logadas.
- Muitos callbacks anônimos com lógica complexa.

### Relação com SOLID

| Princípio | Relação |
|-----------|--------|
| **SRP** | Cada Command encapsula uma única ação |
| **OCP** | Novos commands adicionados sem alterar Invoker ou Receiver |
| **DIP** | Invoker depende da interface Command, não de implementações |

> **Veja também:** [Memento](#memento) (para armazenar estado de undo), [Strategy](#strategy), [CQRS](../05-architectural-patterns/README.md#cqrs--command-query-responsibility-segregation)

---

## Template Method

### Intenção

Definir o **esqueleto de um algoritmo** em uma operação, **delegando** alguns passos
para subclasses. Permite que subclasses redefinam certos passos sem alterar a estrutura.

### Problema que resolve

- Vários algoritmos têm a mesma estrutura, mas diferem em etapas específicas.
- Quer evitar duplicação de código do algoritmo principal.
- Subclasses devem poder customizar partes do algoritmo, não o todo.

### Estrutura

```
┌──────────────────────────┐
│   AbstractClass           │
├──────────────────────────┤
│ + templateMethod()        │ ← define o esqueleto (final)
│ # step1()                │ ← pode ter implementação padrão
│ # step2()                │ ← abstrato — subclasse implementa
│ # hook()                 │ ← opcional — subclasse pode sobrescrever
└────────────▲─────────────┘
             │
┌────────────┴─────────────┐
│   ConcreteClass           │
├──────────────────────────┤
│ # step2()                │ ← implementação concreta
│ # hook()                 │ ← sobrescrita opcional
└──────────────────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **AbstractClass** | Define o template method e os passos abstratos/hook |
| **ConcreteClass** | Implementa os passos abstratos e opcionalmente os hooks |

### Conceitos

| Conceito | Descrição |
|----------|-----------|
| **Template Method** | O algoritmo principal — chama os passos na ordem correta |
| **Abstract Steps** | Passos que **devem** ser implementados pela subclasse |
| **Hooks** | Passos com implementação padrão que **podem** ser sobrescritos |
| **Hollywood Principle** | "Don't call us, we'll call you" — o framework chama o código do usuário |

### Template Method vs Strategy

| Aspecto | Template Method | Strategy |
|---------|----------------|----------|
| **Mecanismo** | Herança | Composição |
| **Variação** | Parte do algoritmo | Algoritmo inteiro |
| **Binding** | Compile time | Runtime |
| **Flexibilidade** | Menor | Maior |
| **Uso** | Quando a estrutura é fixa | Quando o algoritmo inteiro varia |

### Trade-offs

- Usa herança — subclasses ficam acopladas à classe abstrata.
- Quanto mais passos, mais complexo o contrato entre base e subclasses.
- Prefira Strategy quando composição for viável.

### Sinais no código (quando considerar Template Method)

- Múltiplas classes com algoritmo idêntico na estrutura, diferindo apenas em etapas específicas.
- Duplicação de código onde apenas partes internas do algoritmo variam.
- Necessidade de forçar uma sequência de passos (framework/hook).

### Relação com SOLID

| Princípio | Relação |
|-----------|--------|
| **OCP** | Pontos de extensão (hooks e métodos abstratos) permitem variação sem modificar o esqueleto |
| **DIP** | O framework (classe base) chama código do usuário (subclasse) — Hollywood Principle |
| **LSP** | Subclasses devem respeitar o contrato do template method |

> **Veja também:** [Strategy](#strategy) (alternativa via composição), [Factory Method](../02-creational-patterns/README.md#factory-method) (frequentemente usado junto), [Composição sobre Herança](../06-best-practices/README.md#composição-sobre-herança)

---

## State

### Intenção

Permitir que um objeto **altere seu comportamento** quando seu estado interno muda.
O objeto parecerá ter mudado de classe.

### Problema que resolve

- Objeto com comportamento que varia conforme o estado.
- Condicionais complexos (`if state == X`, `switch state`) espalhados pelo código.
- Transições de estado são difíceis de manter e estender.

### Estrutura

```
┌──────────────────┐         ┌──────────────────┐
│     Context       │────────▶│      State       │
├──────────────────┤         │   (interface)    │
│ - state: State    │         ├──────────────────┤
│ + request()       │         │ + handle()       │
│ + setState()      │         └───────▲──────────┘
└──────────────────┘                 │
                           ┌─────────┼─────────┐
                           │                    │
                   ┌───────┴──────┐    ┌───────┴──────┐
                   │ ConcreteStA   │    │ ConcreteStB   │
                   │ + handle()    │    │ + handle()    │
                   └──────────────┘    └──────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Context** | Mantém o estado atual; delega comportamento para o objeto State |
| **State** | Interface para comportamento dependente de estado |
| **ConcreteState** | Implementa comportamento para um estado específico |

### Quando usar

- O comportamento de um objeto depende do seu estado e muda em runtime.
- Operações têm grandes condicionais dependentes do estado.
- Transições entre estados precisam ser explícitas e organizadas.

### Exemplos do mundo real

- Máquina de estados de pedidos (criado → pago → enviado → entregue).
- Conexões de rede (conectado → desconectado → reconectando).
- Players de mídia (parado → reproduzindo → pausado).
- Workflows de aprovação.

### State vs Strategy

| Aspecto | State | Strategy |
|---------|-------|----------|
| **Propósito** | Comportamento muda com estado interno | Algoritmo é escolhido pelo cliente |
| **Transição** | States se conhecem e fazem transições | Strategies são independentes |
| **Consciência** | States sabem sobre outros states | Strategies não sabem umas sobre outras |

### Trade-offs

- Pode ser over-engineering para máquinas com poucos estados.
- Número de classes aumenta proporcionalmente aos estados.
- Transições complexas podem ser difíceis de rastrear.

### Sinais no código (quando considerar State)

- Múltiplos `if (state == X)` ou `switch(state)` espalhados por vários métodos.
- Cada novo estado requer modificação em vários pontos do código.
- Lógica de transição de estado misturada com lógica de negócio.
- Flags booleanas controlando comportamento (`isActive`, `isPaused`, `isReady`).

### Relação com SOLID

| Princípio | Relação |
|-----------|--------|
| **OCP** | Novos estados = novas classes, sem alterar o Context |
| **SRP** | Cada ConcreteState encapsula lógica de um único estado |
| **LSP** | Todos os states são substituíveis via interface State |

> **Veja também:** [Strategy](#strategy) (similar em estrutura), [Singleton](../02-creational-patterns/README.md#singleton) (states compartilhados como singletons)

---

## Chain of Responsibility

### Intenção

Evitar o acoplamento entre o **emissor** de uma requisição e seu **receptor**, dando
a **mais de um objeto** a chance de tratar a requisição.

### Problema que resolve

- Múltiplos handlers podem processar uma requisição, mas o emissor não sabe qual.
- A ordem de processamento pode variar.
- Quer adicionar novos handlers sem alterar o emissor ou handlers existentes.

### Estrutura

```
Client ──▶ Handler A ──▶ Handler B ──▶ Handler C ──▶ (fim)
               │               │              │
           (trata ou        (trata ou      (trata ou
            passa)           passa)          passa)
```

```
┌──────────────────────┐
│      Handler          │
│    (interface)        │
├──────────────────────┤
│ - next: Handler       │
│ + handle(request)     │
│ + setNext(handler)    │
└───────────▲──────────┘
            │
   ┌────────┴────────┐
   │ConcreteHandler   │
   │ + handle(req)    │ ← trata se puder; senão passa para next
   └─────────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Handler** | Interface + referência para o próximo handler na cadeia |
| **ConcreteHandler** | Trata requisições que sabe resolver; repassa as demais |
| **Client** | Inicia a requisição no primeiro handler da cadeia |

### Variantes

| Variante | Descrição |
|----------|-----------|
| **Puro** | Apenas um handler trata; os demais repassam |
| **Pipeline** | Todos os handlers processam (cada um faz sua parte) |
| **Middleware** | Padrão pipeline com next() para controle de fluxo |

### Quando usar

- Mais de um objeto pode tratar uma requisição e o handler não é conhecido a priori.
- O conjunto de handlers deve ser configurável dinamicamente.
- Quer aplicar filtros/transformações em cadeia (middlewares HTTP).

### Exemplos do mundo real

- Middleware HTTP (autenticação → autorização → validação → handler).
- Pipelines de processamento de dados.
- Handlers de log com diferentes níveis (debug → info → warn → error).
- Chains de validação.

### Trade-offs

- Não há garantia de que a requisição será tratada.
- Debugging pode ser complexo em cadeias longas.
- Performance: cada handler adiciona overhead.

### Sinais no código (quando considerar Chain of Responsibility)

- Cadeias de `if/else if/else if` para determinar quem processa uma requisição.
- Lógica de middleware duplicada ou espalhada.
- Necessidade de configurar dinamicamente quem processa o quê.
- Vários filtros/validações aplicados em sequência.

### Relação com SOLID

| Princípio | Relação |
|-----------|--------|
| **OCP** | Novos handlers adicionados sem alterar handlers existentes |
| **SRP** | Cada handler se responsabiliza por um único tipo de processamento |
| **DIP** | Handlers dependem da interface Handler, não de concretos |

> **Veja também:** [Decorator](../03-structural-patterns/README.md#decorator) (similar em encadeamento), [Composite](../03-structural-patterns/README.md#composite), [Command](#command)

---

## Mediator

### Intenção

Definir um objeto que **encapsula como um conjunto de objetos interage**.
Promove acoplamento fraco ao evitar que objetos se refiram uns aos outros diretamente.

### Problema que resolve

- Muitos objetos se comunicam entre si, criando uma teia de dependências.
- Adicionar um novo participante requer alterar vários existentes.
- Quer centralizar a lógica de coordenação.

### Estrutura

```
┌────────────┐                    ┌────────────┐
│ Colleague A │──┐            ┌──│ Colleague B │
└────────────┘   │            │  └────────────┘
                 ▼            ▼
            ┌──────────────────┐
            │     Mediator      │
            │ (centraliza a     │
            │  comunicação)     │
            └──────────────────┘
                 ▲            ▲
┌────────────┐   │            │  ┌────────────┐
│ Colleague C │──┘            └──│ Colleague D │
└────────────┘                    └────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Mediator** | Interface para comunicação entre colleagues |
| **ConcreteMediator** | Coordena a interação entre os objetos concretos |
| **Colleague** | Conhece apenas o Mediator; comunica-se através dele |

### Quando usar

- Um conjunto de objetos se comunica de forma complexa e bem definida.
- A reutilização de um objeto é difícil por causa de suas dependências com outros.
- Quer centralizar lógica que está distribuída entre vários objetos.

### Exemplos do mundo real

- Torre de controle de aeroporto (aviões não se comunicam diretamente).
- Dialog boxes em UIs (campos, botões, listas interagem via dialog controller).
- Message brokers (Kafka, RabbitMQ como mediadores entre serviços).
- Padrão MediatR em aplicações CQRS.

### Mediator vs Observer

| Aspecto | Mediator | Observer |
|---------|----------|----------|
| **Direção** | Bidirecional (centralizada) | Unidirecional (broadcast) |
| **Conhecimento** | Mediator conhece todos | Subject não conhece detalhes dos observers |
| **Propósito** | Coordenar interação complexa | Notificar mudanças |

### Trade-offs

- O Mediator pode se tornar um "God Object" se concentrar lógica demais.
- Centralização pode ser um ponto único de falha.
- Para poucos participantes, pode ser desnecessário.

### Sinais no código (quando considerar Mediator)

- Muitas classes com referências cruzadas (A conhece B, B conhece C, C conhece A).
- Adicionar um novo participante requer alterar vários existentes.
- Teia de dependências bidirecionais entre componentes.
- Lógica de coordenação espalhada entre os participantes.

### Relação com SOLID

| Princípio | Relação |
|-----------|--------|
| **SRP** | Centraliza lógica de coordenação em um único lugar |
| **OCP** | Novos colleagues podem ser adicionados sem alterar os existentes |
| **DIP** | Colleagues dependem da interface Mediator, não uns dos outros |

> **Veja também:** [Observer](#observer) (alternativa descentralizada), [Facade](../03-structural-patterns/README.md#facade) (simplificação unidirecional)

---

## Iterator

### Intenção

Prover uma forma de acessar os elementos de um **agregado sequencialmente**
sem expor sua representação interna.

### Problema que resolve

- O cliente precisa percorrer uma coleção sem conhecer sua implementação (lista, árvore, hash).
- Diferentes formas de percorrer a mesma coleção (em ordem, reverso, filtrado).
- Múltiplas travessias simultâneas sobre o mesmo agregado.

### Estrutura

```
┌──────────────────┐       ┌──────────────────┐
│   Aggregate       │──────▶│    Iterator      │
│  (interface)     │       │   (interface)    │
├──────────────────┤       ├──────────────────┤
│ + createIterator()│       │ + hasNext()      │
└──────────────────┘       │ + next()         │
                           │ + current()      │
                           └──────────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Iterator** | Interface para acessar e percorrer elementos |
| **ConcreteIterator** | Implementa a travessia; mantém posição atual |
| **Aggregate** | Interface para criar um Iterator |
| **ConcreteAggregate** | Retorna uma instância do ConcreteIterator |

### Quando usar

- Acessar conteúdo de um agregado sem expor representação interna.
- Suportar múltiplas formas de travessia.
- Prover interface uniforme para percorrer diferentes estruturas (polimorfismo de iteração).

### Considerações modernas

A maioria das linguagens modernas já fornece iterators nativamente:
- `for...of` / `for...in` em JavaScript/TypeScript
- `for range` em Go
- `Stream API` em Java
- Generators / `yield` em Python, C#, JavaScript
- Traits como `Iterator` em Rust

O padrão Iterator é mais relevante quando se implementa coleções customizadas.

### Trade-offs

- Em linguagens modernas, raramente precisa ser implementado manualmente.
- Iteradores externos dão mais controle; iteradores internos são mais simples.

### Relação com SOLID

| Princípio | Relação |
|-----------|--------|
| **SRP** | Separa a lógica de travessia da lógica da coleção |
| **OCP** | Novas formas de iteração sem alterar a coleção |
| **ISP** | Interface Iterator é minimalista (`hasNext`, `next`) |

> **Veja também:** [Composite](../03-structural-patterns/README.md#composite) (iteração sobre árvores), [Visitor](#visitor)

---

## Memento

### Intenção

Capturar e externalizar o **estado interno** de um objeto, sem violar encapsulamento,
de modo que o objeto possa ser **restaurado** a esse estado posteriormente.

### Problema que resolve

- Implementar undo/redo, checkpoints ou snapshots.
- Salvar e restaurar o estado de um objeto preservando encapsulamento.

### Estrutura

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Caretaker    │────▶│   Memento    │◀────│  Originator   │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ - mementos[] │     │ - state      │     │ - state      │
│              │     │ + getState() │     │ + save()     │ ← cria Memento
└──────────────┘     └──────────────┘     │ + restore()  │ ← restaura de Memento
                                          └──────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Originator** | Cria mementos contendo snapshot do seu estado |
| **Memento** | Armazena o estado interno do Originator (opaco para outros) |
| **Caretaker** | Guarda mementos; nunca examina ou modifica seu conteúdo |

### Quando usar

- Implementar undo/redo.
- Fazer snapshots/checkpoints (transações, jogos, editores).
- Preservar encapsulamento enquanto externaliza estado.

### Trade-offs

- Pode consumir muita memória se mementos forem grandes ou frequentes.
- O Caretaker precisa gerenciar o ciclo de vida dos mementos (quando descartar).
- Considere armazenar apenas **deltas** em vez de snapshots completos.

### Sinais no código (quando considerar Memento)

- Necessidade de undo/redo sem mecanismo estruturado.
- Estado do objeto sendo salvo/restaurado manualmente em vários pontos.
- Lógica de checkpoint/snapshot ad-hoc.

### Relação com SOLID

| Princípio | Relação |
|-----------|--------|
| **SRP** | Originator cuida do negócio; Memento cuida do snapshot; Caretaker cuida do histórico |
| **Encapsulamento** | Memento preserva encapsulamento do Originator — Caretaker não acessa internals |

> **Veja também:** [Command](#command) (frequentemente usado junto para undo), [Event Sourcing](../05-architectural-patterns/README.md#event-sourcing) (alternativa arquitetural para histórico)

---

## Visitor

### Intenção

Representar uma **operação** a ser executada sobre os elementos de uma estrutura de objetos.
Visitor permite definir novas operações **sem alterar** as classes dos elementos.

### Problema que resolve

- Uma estrutura de objetos (ex: AST, documento) precisa suportar muitas operações diferentes.
- Adicionar cada operação às classes da estrutura violaria SRP.
- Quer adicionar operações sem modificar a hierarquia de elementos (OCP).

### Estrutura

```
┌──────────────────┐         ┌──────────────────┐
│    Element        │         │    Visitor        │
│  (interface)     │         │   (interface)    │
├──────────────────┤         ├──────────────────┤
│ + accept(visitor) │         │ + visitA(elemA)  │
└───────▲──────────┘         │ + visitB(elemB)  │
        │                    └───────▲──────────┘
   ┌────┴────┐                      │
   │         │              ┌───────┴──────────┐
┌──┴──┐  ┌──┴──┐           │ConcreteVisitor    │
│ElemA│  │ElemB│           │ + visitA(elemA)   │
│     │  │     │           │ + visitB(elemB)   │
└─────┘  └─────┘           └──────────────────┘

ElemA.accept(v) → v.visitA(this)   (double dispatch)
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Visitor** | Declara uma operação `visit` para cada tipo de Element |
| **ConcreteVisitor** | Implementa cada operação para cada tipo de Element |
| **Element** | Declara `accept(visitor)` |
| **ConcreteElement** | Implementa `accept` chamando a operação de visita correspondente |

### Quando usar

- Estrutura de objetos contém **muitas classes diferentes** e quer operações que dependem do tipo.
- Novas operações são adicionadas frequentemente, mas novos tipos de elementos são raros.
- Quer evitar "poluir" as classes de elemento com operações variadas.

### Exemplos do mundo real

- Compiladores: visitors para type-checking, otimização, geração de código sobre uma AST.
- Serialização: visitor que converte uma estrutura para JSON, XML, etc.
- Análise estática de código.

### Trade-offs

- **Difícil adicionar novos tipos de Elements:** cada Visitor precisa ser atualizado.
- **Violação de encapsulamento:** Visitor pode precisar acessar detalhes internos dos elementos.
- **Double dispatch:** necessário porque a maioria das linguagens tem single dispatch.
- **Complexidade:** para estruturas simples, polimorfismo direto pode ser suficiente.

### Sinais no código (quando considerar Visitor)

- Vários `instanceof`/`type switch` para executar operações diferentes por tipo de elemento.
- Novas operações sobre uma hierarquia de classes são adicionadas frequentemente.
- Operações não-relacionadas acumulando-se nas classes de elemento (violando SRP).

### Relação com SOLID

| Princípio | Relação |
|-----------|--------|
| **OCP** | Novas operações (visitors) sem alterar os elementos |
| **SRP** | Cada visitor encapsula uma operação específica; elementos não acumulam lógica |
| **Conflito com OCP** | Adicionar novos tipos de elementos requer alterar todos os visitors |

> **Veja também:** [Composite](../03-structural-patterns/README.md#composite) (Visitor é frequentemente usado sobre árvores Composite), [Iterator](#iterator), [Strategy](#strategy)

---

## Comparativo

| Aspecto | Strategy | Observer | Command | Template M. | State | Chain | Mediator | Iterator | Memento | Visitor |
|---------|----------|----------|---------|-------------|-------|-------|----------|----------|---------|---------|
| **Foco** | Algoritmo | Notificação | Ação | Esqueleto | Estado | Cadeia | Coordenação | Travessia | Snapshot | Operação |
| **Mecanismo** | Composição | Pub/Sub | Encapsulação | Herança | Delegação | Cadeia | Centralização | Sequência | Snapshot | Double dispatch |
| **Acoplamento** | Baixo | Baixo | Baixo | Médio | Baixo | Baixo | Central | Baixo | Baixo | Médio |

---

## Quando usar qual

```
Preciso organizar comportamento entre objetos?
│
├── O ALGORITMO deve variar independentemente?
│   └── Strategy
│
├── Objetos devem ser NOTIFICADOS de mudanças?
│   └── Observer
│
├── Ações devem ser ENCAPSULADAS como objetos (undo, fila, log)?
│   └── Command
│
├── A ESTRUTURA do algoritmo é fixa, mas passos variam?
│   └── Template Method
│
├── O comportamento muda conforme o ESTADO?
│   └── State
│
├── Múltiplos handlers devem ter CHANCE de tratar uma requisição?
│   └── Chain of Responsibility
│
├── Muitos objetos se COMUNICAM de forma complexa?
│   └── Mediator
│
├── Preciso PERCORRER uma coleção sem expor implementação?
│   └── Iterator
│
├── Preciso SALVAR e RESTAURAR estado (undo/checkpoint)?
│   └── Memento
│
└── Preciso adicionar OPERAÇÕES a uma estrutura sem alterar as classes?
    └── Visitor
```
