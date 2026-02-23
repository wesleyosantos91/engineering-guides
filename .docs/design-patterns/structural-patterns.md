# Padrões Estruturais (Structural Patterns)

> **Agnóstico a linguagem** — Padrões estruturais se preocupam com a **composição** de classes
> e objetos para formar estruturas maiores e mais flexíveis, mantendo a eficiência.

---

## Sumário

- [Visão Geral](#visão-geral)
- [Adapter](#adapter)
- [Bridge](#bridge)
- [Composite](#composite)
- [Decorator](#decorator)
- [Facade](#facade)
- [Flyweight](#flyweight)
- [Proxy](#proxy)
- [Comparativo](#comparativo)
- [Quando usar qual](#quando-usar-qual)

---

## Visão Geral

| Padrão | Intenção |
|--------|----------|
| **Adapter** | Converter a interface de uma classe em outra interface que o cliente espera |
| **Bridge** | Separar uma abstração de sua implementação para que ambas possam variar independentemente |
| **Composite** | Compor objetos em estruturas de árvore para representar hierarquias parte-todo |
| **Decorator** | Adicionar responsabilidades a um objeto dinamicamente |
| **Facade** | Fornecer uma interface simplificada para um subsistema complexo |
| **Flyweight** | Compartilhar objetos para suportar grande quantidade de objetos granulares eficientemente |
| **Proxy** | Fornecer um substituto ou placeholder para controlar o acesso a outro objeto |

---

## Adapter

### Intenção

Converter a interface de uma classe em outra interface que os clientes esperam.
Permite que classes com interfaces incompatíveis trabalhem juntas.

### Problema que resolve

- Você quer usar uma classe existente, mas sua interface não é compatível com o que precisa.
- Integração com bibliotecas/APIs de terceiros que possuem interface diferente da esperada.
- Migração gradual de um sistema legado.

### Estrutura

```
┌──────────┐       ┌──────────────┐       ┌──────────────┐
│  Client   │──────▶│   Target     │       │   Adaptee    │
│           │       │ (interface)  │       │ (existente)  │
└──────────┘       └──────▲───────┘       └──────▲───────┘
                          │                      │
                   ┌──────┴───────┐              │
                   │   Adapter     │──────────────┘
                   │ implementa    │   usa/delega
                   │ Target        │
                   └──────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Target** | Interface que o cliente espera |
| **Adaptee** | Classe existente com interface incompatível |
| **Adapter** | Adapta a interface do Adaptee para Target |
| **Client** | Colabora com objetos através da interface Target |

### Variantes

| Variante | Mecanismo | Quando usar |
|----------|-----------|-------------|
| **Object Adapter** | Composição — o Adapter contém referência ao Adaptee | Preferido — mais flexível |
| **Class Adapter** | Herança múltipla — o Adapter herda de ambos | Quando a linguagem suporta herança múltipla |

### Quando usar

- Integração com APIs ou bibliotecas externas.
- Wrapping de código legado para se adaptar a interfaces modernas.
- Quando duas interfaces existentes não combinam e nenhuma pode ser modificada.

### Exemplos do mundo real

- Adaptadores de banco de dados (driver JDBC, database/sql).
- Wrappers de APIs REST que convertem para o formato interno do sistema.
- Serializers que adaptam formatos (XML → JSON).

### Trade-offs

- Adiciona uma camada de indireção.
- Se existem muitas adaptações, talvez o design precise ser repensado.

---

## Bridge

### Intenção

Desacoplar uma **abstração** de sua **implementação**, de modo que ambas
possam variar independentemente.

### Problema que resolve

- Uma classe tem duas dimensões de variação (ex: forma + renderizador, plataforma + funcionalidade).
- Herança criaria uma explosão combinatória de subclasses.
- Você quer trocar a implementação em tempo de execução.

### Estrutura

```
┌──────────────────┐           ┌────────────────────┐
│   Abstraction     │──────────▶│  Implementation    │
├──────────────────┤           │   (interface)      │
│ - impl: Impl      │           ├────────────────────┤
│ + operation()     │           │ + operationImpl()  │
└───────▲──────────┘           └────────▲───────────┘
        │                               │
┌───────┴──────────┐          ┌────────┴───────────┐
│ RefinedAbstraction│          │ ConcreteImplA/B    │
└──────────────────┘          └────────────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Abstraction** | Define a interface de alto nível; mantém referência para Implementation |
| **RefinedAbstraction** | Estende a interface da Abstraction |
| **Implementation** | Interface para as implementações concretas |
| **ConcreteImplementation** | Implementação específica |

### Quando usar

- Você quer evitar uma ligação permanente entre abstração e implementação.
- Ambas devem ser extensíveis por herança de forma independente.
- Mudanças na implementação não devem impactar o código cliente.

### Bridge vs Adapter

| Aspecto | Bridge | Adapter |
|---------|--------|---------|
| **Intenção** | Projetar para independência desde o início | Compatibilizar interfaces depois |
| **Timing** | Design time | Integração time |
| **Variação** | Duas dimensões independentes | Uma conversão direcional |

### Trade-offs

- Aumenta a complexidade do design com mais classes/interfaces.
- Pode ser over-engineering se a variação em duas dimensões não é real.

---

## Composite

### Intenção

Compor objetos em **estruturas de árvore** para representar hierarquias **parte-todo**.
Permite que clientes tratem objetos individuais e composições de forma uniforme.

### Problema que resolve

- O sistema precisa representar hierarquias recursivas (menus, diretórios, expressões).
- Clientes devem tratar objetos simples e compostos da mesma forma.

### Estrutura

```
                ┌──────────────────┐
   Client ─────▶│   Component      │
                │  (interface)     │
                ├──────────────────┤
                │ + operation()    │
                └───────▲──────────┘
                        │
           ┌────────────┼────────────┐
           │                         │
   ┌───────┴────────┐      ┌────────┴────────┐
   │     Leaf        │      │   Composite      │
   ├────────────────┤      ├─────────────────┤
   │ + operation()  │      │ - children[]    │
   └────────────────┘      │ + operation()   │ ← delega para children
                           │ + add(Component)│
                           │ + remove()      │
                           └─────────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Component** | Interface comum para Leaf e Composite |
| **Leaf** | Objeto primitivo sem filhos |
| **Composite** | Objeto que contém filhos; delega operações para eles |

### Quando usar

- Representar hierarquias parte-todo (árvore de arquivos, menus, organogramas).
- Clientes devem poder ignorar a diferença entre composições e objetos individuais.
- Operações recursivas sobre a estrutura (calcular total, renderizar, serializar).

### Trade-offs

- Dificulta restringir quais tipos podem ser filhos de um Composite.
- Pode ser difícil manter invariantes da árvore (ex: profundidade máxima).
- Componentes tornam-se mais genéricos — menos segurança de tipos.

---

## Decorator

### Intenção

**Adicionar responsabilidades** a um objeto de forma dinâmica.
Uma alternativa flexível à herança para estender funcionalidade.

### Problema que resolve

- Herança é estática — impossível adicionar/remover comportamento em tempo de execução.
- Explosão de subclasses para cada combinação de funcionalidades.
- Quer adicionar comportamento **sem alterar** a classe original (OCP).

### Estrutura

```
┌──────────────────┐
│   Component       │
│  (interface)     │
├──────────────────┤
│ + operation()    │
└───────▲──────────┘
        │
   ┌────┴──────────────┐
   │                    │
┌──┴───────────┐  ┌────┴──────────────┐
│ Concrete      │  │    Decorator       │
│ Component     │  ├──────────────────┤
└──────────────┘  │ - wrapped: Comp.  │
                  │ + operation()     │ ← delega + adiciona
                  └───────▲───────────┘
                          │
                 ┌────────┴────────┐
                 │ ConcreteDecorator │
                 │ + operation()    │
                 │ + extraBehavior()│
                 └─────────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Component** | Interface do objeto que pode receber decorators |
| **ConcreteComponent** | Objeto original que será decorado |
| **Decorator** | Mantém referência ao Component e implementa a mesma interface |
| **ConcreteDecorator** | Adiciona comportamento antes/depois de delegar ao wrapped |

### Quando usar

- Adicionar responsabilidades de forma **dinâmica** e **transparente**.
- Quando herança geraria explosão de subclasses.
- Quando se precisa combinar diversos comportamentos (logging + cache + retry).

### Exemplos do mundo real

- Streams de I/O (buffered, compressed, encrypted).
- Middlewares HTTP (logging, auth, CORS, rate limiting).
- Cache decorators em repositórios.

### Decorator vs Herança

| Aspecto | Decorator | Herança |
|---------|-----------|---------|
| **Binding** | Dinâmico (runtime) | Estático (compile time) |
| **Combinação** | Livre — empilha decorators | Explosão de subclasses |
| **Princípio** | Composição sobre herança | Herança direta |
| **Flexibilidade** | Alta | Baixa |

### Trade-offs

- Muitos decorators empilhados dificultam o debugging.
- A identidade do objeto muda (o objeto decorado é um wrapper, não o original).
- A ordem dos decorators pode importar.

---

## Facade

### Intenção

Fornecer uma **interface unificada e simplificada** para um conjunto de interfaces
em um subsistema. Facade define uma interface de alto nível.

### Problema que resolve

- Subsistemas complexos com muitas classes são difíceis de usar.
- Clientes não precisam conhecer detalhes internos.
- Quer reduzir o acoplamento entre clientes e o subsistema.

### Estrutura

```
┌──────────┐
│  Client   │
└─────┬────┘
      │ usa
┌─────▼────────────┐
│     Facade        │
├──────────────────┤
│ + operaçãoSimples│ ← orquestra subsistema
└──┬───┬───┬───────┘
   │   │   │
   ▼   ▼   ▼
┌────┐┌────┐┌────┐
│ A  ││ B  ││ C  │  ← classes do subsistema
└────┘└────┘└────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Facade** | Conhece quais classes do subsistema são responsáveis por cada operação; delega pedidos |
| **Subsystem classes** | Implementam funcionalidade; não conhecem o Facade |
| **Client** | Usa o Facade em vez de acessar o subsistema diretamente |

### Quando usar

- Fornecer uma interface simples para um subsistema complexo.
- Desacoplar clientes de um subsistema, permitindo que o subsistema evolua independentemente.
- Definir pontos de entrada para camadas de uma arquitetura em camadas.

### Facade vs Adapter vs Mediator

| Aspecto | Facade | Adapter | Mediator |
|---------|--------|---------|----------|
| **Propósito** | Simplificar | Compatibilizar | Coordenar |
| **Direção** | Unidirecional (cliente → subsistema) | Bidirecional | Multidirecional |
| **Conhecimento** | Conhece o subsistema | Conhece target e adaptee | Conhece todos os colegas |

### Trade-offs

- Pode se tornar um "God Object" se abarcar funcionalidades demais.
- Não impede acesso direto ao subsistema — é uma conveniência, não uma restrição.

---

## Flyweight

### Intenção

Usar **compartilhamento** para suportar eficientemente grandes quantidades
de objetos granulares.

### Problema que resolve

- A aplicação precisa criar milhares/milhões de objetos similares.
- O consumo de memória é proibitivo.
- Parte do estado do objeto pode ser **compartilhada** (intrínseco), enquanto parte é **contextual** (extrínseco).

### Estrutura

```
┌──────────────────┐         ┌──────────────────┐
│ FlyweightFactory  │────────▶│   Flyweight      │
├──────────────────┤         │  (interface)     │
│ - pool: Map       │         ├──────────────────┤
│ + getFlyweight()  │         │ + operation(     │
└──────────────────┘         │   extrinsicState)│
                             └───────▲──────────┘
                                     │
                             ┌───────┴──────────┐
                             │ConcreteFlyweight  │
                             ├──────────────────┤
                             │ - intrinsicState  │ ← compartilhado
                             │ + operation(ext)  │ ← ext = contextual
                             └──────────────────┘
```

### Conceitos-chave

| Conceito | Descrição |
|----------|-----------|
| **Estado intrínseco** | Dados compartilhados entre objetos — armazenados no Flyweight |
| **Estado extrínseco** | Dados únicos por contexto — passados pelo cliente no momento do uso |
| **Pool/Cache** | Fábrica mantém cache de flyweights existentes |

### Quando usar

- A aplicação usa grande número de objetos que consomem muita memória.
- A maioria do estado pode ser extraída e tornada extrínseca.
- Muitos grupos de objetos podem ser substituídos por poucos objetos compartilhados.

### Exemplos do mundo real

- Pool de caracteres em editores de texto.
- Cache de ícones/texturas em jogos.
- String interning em linguagens de programação.

### Trade-offs

- Complexidade: o código precisa gerenciar estado extrínseco separadamente.
- Trade-off: RAM vs CPU (calcular estado extrínseco pode ter custo).
- Nem sempre é necessário — meça antes de otimizar.

---

## Proxy

### Intenção

Fornecer um **substituto ou placeholder** para outro objeto, controlando o acesso a ele.

### Problema que resolve

- Controle de acesso, lazy loading, logging, caching ou acesso remoto a um objeto.
- O cliente não deve (ou não precisa) acessar o objeto real diretamente.

### Estrutura

```
┌──────────┐       ┌──────────────┐
│  Client   │──────▶│   Subject    │
└──────────┘       │ (interface)  │
                   └──────▲───────┘
                          │
              ┌───────────┼───────────┐
              │                       │
      ┌───────┴────────┐    ┌────────┴───────┐
      │   RealSubject   │    │     Proxy       │
      │ (objeto real)   │    ├────────────────┤
      └────────────────┘    │ - real: Subject │
                            │ + request()     │ ← controla acesso
                            └────────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Subject** | Interface comum para RealSubject e Proxy |
| **RealSubject** | O objeto real que o Proxy representa |
| **Proxy** | Mantém referência ao RealSubject; controla acesso |

### Tipos de Proxy

| Tipo | Propósito | Exemplo |
|------|-----------|---------|
| **Virtual Proxy** | Lazy loading — cria o objeto real sob demanda | Carregamento de imagens |
| **Remote Proxy** | Representa um objeto em outro espaço de endereço | RPC, gRPC stubs |
| **Protection Proxy** | Controla acesso baseado em permissões | Proxy de autorização |
| **Caching Proxy** | Cache de resultados de operações custosas | Cache de queries |
| **Logging Proxy** | Registra chamadas ao objeto real | Auditoria, debugging |
| **Smart Reference** | Ações adicionais ao acessar o objeto | Contagem de referências |

### Quando usar

- Lazy initialization de objetos pesados.
- Controle de acesso/permissões.
- Cache transparente.
- Logging/monitoramento sem alterar a classe original.
- Acesso a objetos remotos.

### Proxy vs Decorator

| Aspecto | Proxy | Decorator |
|---------|-------|-----------|
| **Propósito** | Controlar **acesso** | Adicionar **comportamento** |
| **Ciclo de vida** | Proxy geralmente gerencia o ciclo do real subject | Decorator recebe o componente pronto |
| **Foco** | Segurança, lazy load, cache | Funcionalidade adicional |
| **Empilhamento** | Geralmente único | Geralmente empilhado |

### Trade-offs

- Adiciona indireção e latência.
- Se usado em excesso, dificulta rastrear qual objeto está sendo acessado.

---

## Comparativo

| Aspecto | Adapter | Bridge | Composite | Decorator | Facade | Flyweight | Proxy |
|---------|---------|--------|-----------|-----------|--------|-----------|-------|
| **Foco** | Compatibilidade | Desacoplamento | Hierarquia | Extensão | Simplificação | Memória | Controle |
| **Composição** | 1:1 | N:M | Árvore | Cadeia | 1:N | Pool | 1:1 |
| **Transparência** | Interface diferente | Mesma abstração | Mesma interface | Mesma interface | Interface nova | Mesma interface | Mesma interface |
| **Complexidade** | Baixa | Média | Média | Baixa | Baixa | Alta | Baixa |

---

## Quando usar qual

```
Preciso compor objetos de forma flexível?
│
├── Interfaces INCOMPATÍVEIS entre componentes?
│   └── Adapter
│
├── Abstração e implementação devem VARIAR independentemente?
│   └── Bridge
│
├── Estrutura em ÁRVORE (parte-todo)?
│   └── Composite
│
├── Adicionar COMPORTAMENTO dinamicamente?
│   └── Decorator
│
├── SIMPLIFICAR acesso a subsistema complexo?
│   └── Facade
│
├── Muitos objetos SIMILARES consumindo memória?
│   └── Flyweight
│
└── Controlar ACESSO a um objeto (lazy load, segurança, cache)?
    └── Proxy
```
