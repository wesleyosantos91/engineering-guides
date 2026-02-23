# Padrões Criacionais (Creational Patterns)

> **Agnóstico a linguagem** — Padrões criacionais abstraem o processo de instanciação de objetos,
> tornando o sistema independente de como seus objetos são criados, compostos e representados.

---

## Sumário

- [Visão Geral](#visão-geral)
- [Singleton](#singleton)
- [Factory Method](#factory-method)
- [Abstract Factory](#abstract-factory)
- [Builder](#builder)
- [Prototype](#prototype)
- [Comparativo](#comparativo)
- [Quando usar qual](#quando-usar-qual)

---

## Visão Geral

Os padrões criacionais resolvem problemas relacionados à **criação de objetos**, oferecendo
mecanismos que aumentam a flexibilidade e a reutilização do código.

| Padrão | Intenção |
|--------|----------|
| **Singleton** | Garantir que uma classe tenha apenas **uma instância** e fornecer um ponto global de acesso |
| **Factory Method** | Definir uma interface para criar objetos, **delegando** a decisão de qual classe instanciar para subclasses |
| **Abstract Factory** | Criar **famílias** de objetos relacionados sem especificar suas classes concretas |
| **Builder** | Separar a **construção** de um objeto complexo da sua representação |
| **Prototype** | Criar novos objetos **clonando** uma instância existente |

---

## Singleton

### Intenção

Garantir que uma classe tenha **exatamente uma instância** em toda a aplicação
e fornecer um ponto de acesso global a ela.

### Problema que resolve

- Controlar acesso a um recurso compartilhado (pool de conexão, cache, configuração).
- Garantir que todos os consumidores usem a mesma instância.

### Estrutura

```
┌──────────────────────┐
│      Singleton        │
├──────────────────────┤
│ - instance: Singleton │
├──────────────────────┤
│ - Singleton()         │  ← construtor privado
│ + getInstance()       │  ← ponto de acesso único
│ + operação()          │
└──────────────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Singleton** | Armazena a instância única e fornece o método de acesso |

### Quando usar

- Deve existir exatamente **uma** instância de uma classe (ex: configuração, logger, pool).
- A instância deve ser acessível por múltiplos consumidores.
- A instância precisa ser inicializada de forma **lazy** (sob demanda).

### Variantes

| Variante | Descrição |
|----------|-----------|
| **Eager initialization** | Instância criada no carregamento da classe |
| **Lazy initialization** | Instância criada no primeiro acesso |
| **Thread-safe** | Usa sincronização para ambientes concorrentes |
| **Double-checked locking** | Otimização da versão thread-safe |
| **Bill Pugh / Holder** | Usa classe interna estática (específico de algumas linguagens) |

### ⚠️ Cuidados e Anti-patterns

| Problema | Descrição |
|----------|-----------|
| **Estado global** | Singleton é essencialmente uma variável global — dificulta rastreamento |
| **Testabilidade** | Difícil de substituir por mock em testes |
| **Acoplamento oculto** | Dependências não ficam explícitas no construtor |
| **Concorrência** | Deve ser thread-safe em ambientes multi-threaded |
| **Abuso** | Não use Singleton como desculpa para variáveis globais |

### Alternativas recomendadas

- **Injeção de Dependência** com escopo singleton (gerenciado pelo container DI).
- Na maioria dos frameworks modernos, o container IoC gerencia o ciclo de vida — prefira isso.

---

## Factory Method

### Intenção

Definir uma interface para a criação de objetos, mas **deixar as subclasses** decidirem
qual classe concreta instanciar.

### Problema que resolve

- O código precisa criar objetos, mas não deve conhecer a classe concreta.
- Diferentes contextos precisam de diferentes implementações do mesmo contrato.

### Estrutura

```
┌───────────────────┐         ┌──────────────────┐
│     Creator       │         │     Product      │
│ (classe abstrata) │         │   (interface)    │
├───────────────────┤         └────────▲─────────┘
│ + factoryMethod() │                  │
│ + someOperation() │         ┌────────┴─────────┐
└────────▲──────────┘         │ ConcreteProduct   │
         │                    └──────────────────┘
┌────────┴──────────┐
│ ConcreteCreator    │
├───────────────────┤
│ + factoryMethod() │ ── cria ──▶ ConcreteProduct
└───────────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Product** | Interface do objeto que o factory method cria |
| **ConcreteProduct** | Implementação concreta do Product |
| **Creator** | Declara o factory method; pode fornecer implementação padrão |
| **ConcreteCreator** | Sobrescreve o factory method para retornar o ConcreteProduct |

### Quando usar

- Uma classe não pode antecipar qual tipo de objeto precisa criar.
- Uma classe quer que suas subclasses especifiquem os objetos que cria.
- Você quer **isolar** o código de criação do código de uso.

### Benefícios

- Elimina o acoplamento entre o criador e os produtos concretos.
- Respeita **SRP**: a lógica de criação fica em um único lugar.
- Respeita **OCP**: novos produtos podem ser adicionados sem alterar código existente.

### Trade-offs

- Pode levar a muitas subclasses paralelas (Creator + Product).
- Para casos simples, um `if/switch` pode ser suficiente (Simple Factory).

---

## Abstract Factory

### Intenção

Fornecer uma interface para criar **famílias de objetos relacionados** ou dependentes
sem especificar suas classes concretas.

### Problema que resolve

- O sistema precisa trabalhar com múltiplas famílias de objetos (ex: UI para Windows vs macOS).
- Objetos de uma família devem ser usados **juntos** e são incompatíveis com outra família.

### Estrutura

```
┌──────────────────────┐
│   AbstractFactory     │
├──────────────────────┤
│ + createProductA()    │
│ + createProductB()    │
└──────────▲───────────┘
           │
  ┌────────┴────────┐
  │                 │
┌─▼──────────┐  ┌──▼─────────┐
│ Factory1   │  │ Factory2   │
│ createA()→A1  │ createA()→A2
│ createB()→B1  │ createB()→B2
└────────────┘  └────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **AbstractFactory** | Declara interface para criar cada tipo de produto |
| **ConcreteFactory** | Implementa a criação de produtos de uma família específica |
| **AbstractProduct** | Interface para um tipo de produto |
| **ConcreteProduct** | Implementação do produto para uma família |

### Quando usar

- O sistema deve ser independente de como os produtos são criados e compostos.
- O sistema precisa garantir que produtos de uma família sejam usados juntos.
- Você quer fornecer uma biblioteca de produtos revelando apenas interfaces.

### Factory Method vs Abstract Factory

| Aspecto | Factory Method | Abstract Factory |
|---------|---------------|-----------------|
| **Escopo** | Um produto | Família de produtos |
| **Mecanismo** | Herança | Composição |
| **Extensão** | Subclasses do Creator | Nova factory concreta |
| **Complexidade** | Menor | Maior |

---

## Builder

### Intenção

Separar a **construção** de um objeto complexo da sua **representação**, de modo que
o mesmo processo de construção possa criar diferentes representações.

### Problema que resolve

- Objetos com muitos parâmetros opcionais (evita construtores telescópicos).
- O processo de construção deve permitir diferentes representações do produto.
- A construção de um objeto envolve múltiplas etapas que devem ser encadeadas.

### Estrutura

```
┌──────────────┐      ┌──────────────────┐
│   Director    │─────▶│     Builder      │
├──────────────┤      │   (interface)    │
│ + construct() │      ├──────────────────┤
└──────────────┘      │ + buildPartA()   │
                      │ + buildPartB()   │
                      │ + getResult()    │
                      └───────▲──────────┘
                              │
                      ┌───────┴──────────┐
                      │ ConcreteBuilder   │
                      ├──────────────────┤
                      │ + buildPartA()   │
                      │ + buildPartB()   │
                      │ + getResult()    │
                      └──────────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Builder** | Interface que define os passos de construção |
| **ConcreteBuilder** | Implementa os passos e mantém o produto em construção |
| **Director** | Orquestra a ordem de construção (opcional) |
| **Product** | O objeto complexo sendo construído |

### Quando usar

- O algoritmo para criar um objeto complexo deve ser independente das partes que compõem o objeto.
- O objeto tem muitos parâmetros opcionais (constructor telescoping problem).
- Diferentes representações do produto devem ser criadas pelo mesmo processo.

### Variantes comuns

| Variante | Descrição |
|----------|-----------|
| **Fluent Builder** | Métodos retornam `this`/`self` para encadeamento: `builder.setA().setB().build()` |
| **Step Builder** | Interfaces sequenciais forçam a ordem de construção |
| **Builder com Director** | Padrão GoF completo com orquestrador |
| **Builder sem Director** | Mais comum na prática — o cliente orquestra diretamente |

### Benefícios

- Evita construtores com dezenas de parâmetros.
- Código de construção é legível e fluente.
- Permite construir objetos **imutáveis** passo a passo.
- Permite reusar o mesmo processo para diferentes produtos.

### Trade-offs

- Aumento no número de classes.
- Para objetos simples (poucos campos obrigatórios), um construtor normal é suficiente.

---

## Prototype

### Intenção

Criar novos objetos **copiando** (clonando) uma instância existente,
evitando o custo de criação do zero.

### Problema que resolve

- Criação de objetos é custosa (ex: envolve chamada a banco, parsing de arquivo complexo).
- O sistema precisa criar objetos que são variações de um protótipo.
- O código não deve depender das classes concretas dos objetos que precisa copiar.

### Estrutura

```
┌──────────────────┐
│    Prototype      │
│   (interface)    │
├──────────────────┤
│ + clone()         │
└───────▲──────────┘
        │
┌───────┴──────────┐
│ ConcretePrototype │
├──────────────────┤
│ + clone()         │ ── retorna cópia de si mesmo
└──────────────────┘
```

### Participantes

| Participante | Responsabilidade |
|--------------|------------------|
| **Prototype** | Declara a interface de clonagem |
| **ConcretePrototype** | Implementa a clonagem de si mesmo |
| **Client** | Cria novos objetos pedindo ao protótipo para clonar-se |

### Quando usar

- Quando a criação de um objeto é mais cara do que copiar um existente.
- Quando o sistema deve ser independente de como os produtos são criados.
- Quando se precisa criar objetos cuja classe é determinada apenas em tempo de execução.

### Cuidados

| Aspecto | Descrição |
|---------|-----------|
| **Shallow copy vs Deep copy** | Cópias rasas compartilham referências; cópias profundas duplicam tudo |
| **Referências circulares** | Clonagem profunda pode entrar em loop — precisa de tratamento |
| **Identidade** | O clone deve ter seu próprio ID? Depende do domínio |

---

## Comparativo

| Aspecto | Singleton | Factory Method | Abstract Factory | Builder | Prototype |
|---------|-----------|---------------|-----------------|---------|-----------|
| **Instâncias** | Uma | Várias | Famílias | Uma (complexa) | Clones |
| **Complexidade** | Baixa | Média | Alta | Média | Média |
| **Acoplamento** | Alto (global) | Baixo | Baixo | Baixo | Baixo |
| **Testabilidade** | Difícil | Boa | Boa | Boa | Boa |
| **Uso típico** | Config, cache | Frameworks | UI toolkits | DTOs, configs | Objetos custosos |

---

## Quando usar qual

```
Preciso criar objetos?
│
├── Apenas UMA instância global?
│   └── Singleton (ou DI com escopo singleton)
│
├── Objetos de uma FAMÍLIA devem ser usados juntos?
│   └── Abstract Factory
│
├── Quero delegar a decisão de QUAL tipo criar?
│   └── Factory Method
│
├── O objeto é COMPLEXO com muitos parâmetros opcionais?
│   └── Builder
│
└── Criar do zero é CUSTOSO e posso copiar um existente?
    └── Prototype
```
