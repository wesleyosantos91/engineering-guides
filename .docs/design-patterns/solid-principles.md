# Princípios SOLID

> **Agnóstico a linguagem** — Os princípios SOLID são diretrizes fundamentais para escrever
> código orientado a objetos que seja fácil de manter, estender e testar.
> Foram formulados por Robert C. Martin (Uncle Bob) e são a base para entender Design Patterns.

---

## Sumário

- [Visão Geral](#visão-geral)
- [S — Single Responsibility Principle (SRP)](#s--single-responsibility-principle-srp)
- [O — Open/Closed Principle (OCP)](#o--openclosed-principle-ocp)
- [L — Liskov Substitution Principle (LSP)](#l--liskov-substitution-principle-lsp)
- [I — Interface Segregation Principle (ISP)](#i--interface-segregation-principle-isp)
- [D — Dependency Inversion Principle (DIP)](#d--dependency-inversion-principle-dip)
- [Relação entre SOLID e Design Patterns](#relação-entre-solid-e-design-patterns)

---

## Visão Geral

| Princípio | Resumo em uma frase |
|-----------|---------------------|
| **SRP** | Uma classe deve ter **um e apenas um** motivo para mudar |
| **OCP** | Entidades devem ser **abertas para extensão** e **fechadas para modificação** |
| **LSP** | Subtipos devem ser **substituíveis** por seus tipos base |
| **ISP** | Clientes não devem depender de interfaces que **não utilizam** |
| **DIP** | Módulos de alto nível não devem depender de módulos de baixo nível; ambos devem depender de **abstrações** |

---

## S — Single Responsibility Principle (SRP)

### Definição

> *"Uma classe deve ter um, e apenas um, motivo para mudar."*

Uma classe (ou módulo/função) deve encapsular **uma única responsabilidade**. Se existem dois motivos
diferentes que poderiam causar alteração naquela classe, ela está fazendo coisas demais.

### Por que é importante

- **Manutenibilidade:** alterações em uma responsabilidade não impactam outra.
- **Testabilidade:** classes menores e focadas são mais fáceis de testar.
- **Reutilização:** componentes coesos podem ser combinados em diferentes contextos.
- **Legibilidade:** é simples entender o propósito de uma classe com responsabilidade única.

### Sinais de violação

| Sinal | Descrição |
|-------|-----------|
| Classe "God Object" | Uma classe que faz tudo: valida, persiste, formata, envia e-mail... |
| Nome genérico | Classes chamadas `Manager`, `Helper`, `Utils` com muitos métodos não relacionados |
| Muitos imports | Se a classe precisa importar dezenas de dependências, provavelmente faz coisas demais |
| Mudanças frequentes | Se toda feature nova altera a mesma classe, ela possui responsabilidades demais |

### Como aplicar

1. **Identifique os atores:** quem são os stakeholders que poderiam pedir mudanças?
2. **Separe por ator:** cada ator deve ter sua própria classe/módulo.
3. **Extraia responsabilidades:** mova lógica não-relacionada para classes especializadas.
4. **Composição:** conecte as partes através de injeção de dependência.

### Trade-offs

- **Excesso de classes:** aplicar SRP ao extremo pode gerar muitas classes pequenas demais.
- **Indireção:** mais classes significam mais navegação no código.
- **Bom senso:** o princípio é uma **diretriz**, não uma lei absoluta. Use julgamento.

---

## O — Open/Closed Principle (OCP)

### Definição

> *"Entidades de software devem ser abertas para extensão, mas fechadas para modificação."*

Você deve conseguir adicionar novos comportamentos **sem alterar** o código existente que já funciona.

### Por que é importante

- **Estabilidade:** código já testado e em produção não precisa ser alterado.
- **Extensibilidade:** novos requisitos são atendidos por adição, não por edição.
- **Redução de risco:** menos chance de introduzir bugs em funcionalidades existentes.

### Mecanismos de extensão

| Mecanismo | Quando usar |
|-----------|-------------|
| **Herança / Polimorfismo** | Quando existe uma família de tipos com comportamento variável |
| **Composição + Interfaces** | Preferido na maioria dos casos; mais flexível que herança |
| **Strategy Pattern** | Quando o algoritmo pode variar em tempo de execução |
| **Decorator Pattern** | Quando se deseja adicionar comportamento sem modificar a classe original |
| **Plugin / Extension Points** | Para sistemas que precisam de extensibilidade externa |

### Sinais de violação

- Cadeias de `if/else` ou `switch/case` que crescem a cada novo tipo.
- Necessidade de alterar múltiplas classes para adicionar um novo comportamento simples.
- Flag parameters que alteram o comportamento interno da classe.

### Trade-offs

- **Over-engineering:** nem toda classe precisa ser extensível desde o primeiro dia.
- **Abstrações prematuras:** crie abstrações quando houver **variação real**, não especulativa.
- **Regra dos 3:** considere abstrair quando o padrão se repetir pela terceira vez.

---

## L — Liskov Substitution Principle (LSP)

### Definição

> *"Objetos de uma superclasse devem poder ser substituídos por objetos de suas subclasses
> sem quebrar o comportamento correto do programa."*

Se `S` é subtipo de `T`, então objetos do tipo `T` podem ser substituídos por objetos do tipo `S`
sem alterar as propriedades desejáveis do programa.

### Por que é importante

- **Confiabilidade:** garante que hierarquias de herança fazem sentido semântico.
- **Polimorfismo correto:** código genérico funciona igualmente com qualquer subtipo.
- **Contratos claros:** pré-condições não são fortalecidas, pós-condições não são enfraquecidas.

### Regras formais

| Regra | Descrição |
|-------|-----------|
| **Pré-condições** | O subtipo NÃO pode exigir **mais** que o supertipo |
| **Pós-condições** | O subtipo NÃO pode garantir **menos** que o supertipo |
| **Invariantes** | Propriedades preservadas pelo supertipo devem ser preservadas pelo subtipo |
| **Exceções** | O subtipo NÃO deve lançar exceções inesperadas que o supertipo não lança |

### Exemplo clássico de violação

O famoso problema **Retângulo-Quadrado**: um Quadrado *não deveria* herdar de Retângulo se o
Retângulo permite alterar largura e altura de forma independente, pois o Quadrado viola essa
expectativa ao forçar ambas dimensões a serem iguais.

### Como verificar

1. **Test de substituição:** use o subtipo em todos os lugares onde o supertipo é usado. Funciona?
2. **Contratos:** o subtipo respeita pré-condições, pós-condições e invariantes?
3. **Princípio da menor surpresa:** o subtipo se comporta como o consumidor espera?

### Trade-offs

- **Herança nem sempre é a resposta:** quando LSP é difícil de garantir, prefira composição.
- **Design by Contract:** considere documentar contratos explicitamente (pré, pós, invariantes).

---

## I — Interface Segregation Principle (ISP)

### Definição

> *"Clientes não devem ser forçados a depender de interfaces que não utilizam."*

É melhor ter **muitas interfaces pequenas e específicas** do que poucas interfaces grandes e genéricas.

### Por que é importante

- **Baixo acoplamento:** classes dependem apenas do que realmente usam.
- **Facilidade de mock/stub:** interfaces menores são mais fáceis de simular em testes.
- **Coesão:** cada interface representa um papel claro e bem definido.

### Sinais de violação

| Sinal | Descrição |
|-------|-----------|
| "Fat interfaces" | Interfaces com dezenas de métodos onde implementadores deixam vários vazios |
| Implementações que lançam `NotImplemented` | O implementador não precisa de todos os métodos |
| Mudanças em cascata | Alterar a interface força mudanças em implementadores que não se importam |

### Como aplicar

1. **Identifique os papéis:** cada grupo de métodos que serve um propósito diferente é uma interface.
2. **Segregue:** divida a interface grande em interfaces menores por papel.
3. **Composição de interfaces:** uma classe pode implementar múltiplas interfaces.
4. **Dependa do papel:** consumidores declaram dependência apenas na interface que usam.

### Trade-offs

- **Muitas interfaces:** excesso de granularidade pode dificultar a navegação.
- **Equilíbrio:** agrupe métodos que são **sempre usados juntos** na mesma interface.

---

## D — Dependency Inversion Principle (DIP)

### Definição

> *"Módulos de alto nível não devem depender de módulos de baixo nível.
> Ambos devem depender de abstrações."*
>
> *"Abstrações não devem depender de detalhes.
> Detalhes devem depender de abstrações."*

### Por que é importante

- **Desacoplamento:** a lógica de negócio não conhece detalhes de infraestrutura.
- **Testabilidade:** é possível substituir dependências reais por mocks/stubs.
- **Flexibilidade:** trocar banco de dados, API externa ou framework sem alterar regras de negócio.

### Conceitos-chave

| Conceito | Descrição |
|----------|-----------|
| **Inversão de Controle (IoC)** | O framework/container controla o ciclo de vida dos objetos |
| **Injeção de Dependência (DI)** | As dependências são fornecidas externamente (construtor, setter, interface) |
| **Abstração (Interface/Contrato)** | O ponto de conexão entre alto e baixo nível |

### Formas de Injeção

| Forma | Quando usar |
|-------|-------------|
| **Construtor** | Preferida na maioria dos casos — dependência obrigatória e imutável |
| **Setter/Property** | Dependências opcionais ou configuráveis após criação |
| **Interface/Method** | Quando a dependência pode variar por chamada de método |

### Relação com a arquitetura

```
┌─────────────────────────────────────┐
│         Alto Nível (Negócio)        │
│         depende de Abstrações       │
└──────────────┬──────────────────────┘
               │ (interface)
┌──────────────▼──────────────────────┐
│       Baixo Nível (Infra)           │
│       implementa Abstrações         │
└─────────────────────────────────────┘
```

### Trade-offs

- **Complexidade:** adicionar abstrações tem custo cognitivo.
- **Não abstraia tudo:** nem toda dependência precisa ser invertida (ex: utilidades de string).
- **Pragmatismo:** aplique DIP nas fronteiras do sistema (banco, APIs externas, mensageria).

---

## Relação entre SOLID e Design Patterns

| Princípio | Padrões que ajudam a aplicá-lo |
|-----------|-------------------------------|
| **SRP** | Facade, Mediator, Command |
| **OCP** | Strategy, Decorator, Observer, Template Method |
| **LSP** | Factory Method, Abstract Factory (garantem subtipos corretos) |
| **ISP** | Adapter, Facade (expõem interfaces segregadas) |
| **DIP** | Abstract Factory, Strategy, Repository, Dependency Injection |

> **Dica:** Design Patterns são **implementações concretas** dos princípios SOLID.
> Entender SOLID primeiro torna o aprendizado de patterns muito mais natural.
