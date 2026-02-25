# SOLID Principles — Guia Teórico

> **Objetivo deste documento:** Servir como referência teórica completa sobre os princípios SOLID, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Abordagem **agnóstica de linguagem** — foca em conceitos, motivações, heurísticas, boas práticas e **pseudocódigo ilustrativo** sem vínculo a tecnologia específica.
>
> **Como usar este documento como knowledge base:** Cada seção é autocontida e pode ser referenciada individualmente. As tabelas de heurísticas, code smells e receitas de refatoração são otimizadas para consulta rápida. A seção ["Diretrizes para Code Review assistido por AI"](#diretrizes-para-code-review-assistido-por-ai) contém regras acionáveis para detecção automática de violações.

---

## Quick Reference — Cheat Sheet

| Princípio | Regra de ouro | Violação típica | Correção |
|-----------|--------------|------------------|----------|
| **SRP** | Um módulo, um motivo para mudar | God Class com múltiplas responsabilidades | Extrair classes por ator/eixo de mudança |
| **OCP** | Estender sem modificar código existente | Cadeia `if/else` sobre tipos | Polimorfismo via Strategy/Factory |
| **LSP** | Subtipo substitui tipo base sem surpresas | Subtipo lança exceção inesperada ou ignora método | Revisitar hierarquia; preferir composição |
| **ISP** | Cliente não depende do que não usa | Interface com 15+ métodos, implementações com stubs vazios | Segregar em role interfaces menores |
| **DIP** | Depender de abstrações, não de concretizações | Service instancia dependências com `new` | Injeção de dependência + interface no domínio |

**Resumo em uma frase (para referência rápida):**

- **SRP:** Separe o que muda por motivos diferentes — agrupe o que muda junto.
- **OCP:** Projete para que novos comportamentos sejam adicionados criando código novo, não modificando o existente.
- **LSP:** Todo subtipo deve cumprir 100% do contrato do tipo base — sem surpresas.
- **ISP:** Cada cliente deve depender apenas do mínimo que realmente usa.
- **DIP:** A lógica de negócio define os contratos (interfaces); a infraestrutura os implementa.

> **Decisão rápida:** Quando vir um code smell, consulte a tabela "Code smells que indicam violação de SOLID" nesta mesma seção para identificar qual princípio atacar primeiro.

---

## Glossário

> Termos técnicos utilizados ao longo deste documento, padronizados para evitar ambiguidade.

| Termo | Definição |
|-------|-----------|
| **Abstração** | Interface, tipo abstrato ou contrato que define *o quê* sem definir *como*. É o ponto de extensão central para OCP e DIP. |
| **Ator (stakeholder)** | Pessoa, grupo ou sistema que solicita mudanças em um módulo. Conceito central do SRP reformulado. |
| **Composição** | Técnica de construir comportamento complexo combinando objetos menores (em oposição à herança). Preferida quando a relação não é "é-um". |
| **Contrato** | Conjunto de pré-condições, pós-condições e invariantes que definem o comportamento esperado de uma abstração. Base do LSP. |
| **Coesão** | Grau em que os elementos de um módulo estão funcionalmente relacionados. Alta coesão → boa aplicação de SRP e ISP. |
| **Acoplamento** | Grau de dependência entre módulos. Baixo acoplamento → boa aplicação de DIP e ISP. |
| **Delegação** | Padrão onde um objeto repassa uma responsabilidade a outro objeto colaborador, em vez de executá-la diretamente. |
| **Design smell** | Sintoma no código que indica problemas de design (rigidez, fragilidade, imobilidade, viscosidade). |
| **God Class** | Anti-pattern onde uma classe acumula muitas responsabilidades, violando SRP. |
| **Injeção de dependência (DI)** | Técnica onde dependências são fornecidas externamente (via construtor, método ou framework) em vez de instanciadas internamente. Implementa DIP. |
| **Inversão de controle (IoC)** | Princípio genérico onde o controle do fluxo é delegado a um framework ou container externo. DI é uma forma de IoC. |
| **Módulo** | Unidade de organização de código — pode ser classe, pacote, namespace, arquivo ou microsserviço, dependendo do nível de abstração. |
| **Módulo de alto nível** | Contém lógica de negócio e orquestra operações. Não deve depender de detalhes de infraestrutura (DIP). |
| **Módulo de baixo nível** | Implementa detalhes técnicos (I/O, banco, rede, filesystem). Deve implementar abstrações definidas pelo alto nível (DIP). |
| **Polimorfismo** | Capacidade de tratar objetos de diferentes tipos através de uma interface comum. Mecanismo central para OCP e LSP. |
| **Role interface** | Interface projetada a partir da perspectiva do cliente/consumidor, contendo apenas os métodos que aquele papel requer. Oposto de *header interface*. |
| **Subtipo** | Tipo que estende ou implementa outro tipo (tipo base). Para respeitar LSP, deve ser substituível sem alterar o comportamento esperado. |
| **YAGNI** | *You Ain't Gonna Need It* — princípio que adverte contra abstrações prematuras. Complementar ao uso pragmático de SOLID. |

---

## Sumário

- [SOLID Principles — Guia Teórico](#solid-principles--guia-teórico)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Glossário](#glossário)
  - [Sumário](#sumário)
  - [O que é SOLID?](#o-que-é-solid)
  - [Por que SOLID importa?](#por-que-solid-importa)
  - [S — Single Responsibility Principle (SRP)](#s--single-responsibility-principle-srp)
    - [Definição](#definição)
    - [Motivação](#motivação)
    - [Heurísticas para identificar violações](#heurísticas-para-identificar-violações)
    - [Boas Práticas](#boas-práticas)
    - [Exemplo: antes e depois (pseudocódigo)](#exemplo-antes-e-depois-pseudocódigo)
    - [Equívocos comuns](#equívocos-comuns)
    - [Anti-patterns comuns](#anti-patterns-comuns)
    - [Relação com outros princípios](#relação-com-outros-princípios)
  - [O — Open/Closed Principle (OCP)](#o--openclosed-principle-ocp)
    - [Definição](#definição-1)
    - [Motivação](#motivação-1)
    - [Mecanismos de extensão](#mecanismos-de-extensão)
    - [Heurísticas para identificar violações](#heurísticas-para-identificar-violações-1)
    - [Boas Práticas](#boas-práticas-1)
    - [Exemplo: antes e depois (pseudocódigo)](#exemplo-antes-e-depois-pseudocódigo-1)
    - [Equívocos comuns](#equívocos-comuns-1)
    - [Anti-patterns comuns](#anti-patterns-comuns-1)
    - [Relação com outros princípios](#relação-com-outros-princípios-1)
  - [L — Liskov Substitution Principle (LSP)](#l--liskov-substitution-principle-lsp)
    - [Definição](#definição-2)
    - [Motivação](#motivação-2)
    - [Regras formais](#regras-formais)
    - [Heurísticas para identificar violações](#heurísticas-para-identificar-violações-2)
    - [Boas Práticas](#boas-práticas-2)
    - [Exemplo: antes e depois (pseudocódigo)](#exemplo-antes-e-depois-pseudocódigo-2)
    - [Equívocos comuns](#equívocos-comuns-2)
    - [Anti-patterns comuns](#anti-patterns-comuns-2)
    - [Relação com outros princípios](#relação-com-outros-princípios-2)
  - [I — Interface Segregation Principle (ISP)](#i--interface-segregation-principle-isp)
    - [Definição](#definição-3)
    - [Motivação](#motivação-3)
    - [Heurísticas para identificar violações](#heurísticas-para-identificar-violações-3)
    - [Boas Práticas](#boas-práticas-3)
    - [Exemplo: antes e depois (pseudocódigo)](#exemplo-antes-e-depois-pseudocódigo-3)
    - [Equívocos comuns](#equívocos-comuns-3)
    - [Anti-patterns comuns](#anti-patterns-comuns-3)
    - [Relação com outros princípios](#relação-com-outros-princípios-3)
  - [D — Dependency Inversion Principle (DIP)](#d--dependency-inversion-principle-dip)
    - [Definição](#definição-4)
    - [Motivação](#motivação-4)
    - [Conceitos fundamentais](#conceitos-fundamentais)
    - [Heurísticas para identificar violações](#heurísticas-para-identificar-violações-4)
    - [Boas Práticas](#boas-práticas-4)
    - [Exemplo: antes e depois (pseudocódigo)](#exemplo-antes-e-depois-pseudocódigo-4)
    - [Equívocos comuns](#equívocos-comuns-4)
    - [Anti-patterns comuns](#anti-patterns-comuns-4)
    - [Relação com outros princípios](#relação-com-outros-princípios-4)
  - [SOLID na Prática — Diretrizes Gerais](#solid-na-prática--diretrizes-gerais)
    - [Quando aplicar](#quando-aplicar)
    - [Quando não aplicar (pragmatismo)](#quando-não-aplicar-pragmatismo)
    - [Métricas e limiares de referência](#métricas-e-limiares-de-referência)
    - [Code smells que indicam violação de SOLID](#code-smells-que-indicam-violação-de-solid)
    - [Checklist de revisão](#checklist-de-revisão)
    - [Receitas de refatoração](#receitas-de-refatoração)
      - [Receita 1: God Class → SRP](#receita-1-god-class--srp)
      - [Receita 2: if/else sobre tipos → OCP](#receita-2-ifelse-sobre-tipos--ocp)
      - [Receita 3: Herança incorreta → LSP](#receita-3-herança-incorreta--lsp)
      - [Receita 4: Fat Interface → ISP](#receita-4-fat-interface--isp)
      - [Receita 5: Dependência direta → DIP](#receita-5-dependência-direta--dip)
  - [SOLID em Arquiteturas Modernas](#solid-em-arquiteturas-modernas)
    - [Hexagonal Architecture (Ports \& Adapters)](#hexagonal-architecture-ports--adapters)
    - [Microservices](#microservices)
    - [Clean Architecture / Onion Architecture](#clean-architecture--onion-architecture)
    - [Event-Driven Architecture](#event-driven-architecture)
  - [Relação entre os Princípios](#relação-entre-os-princípios)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
    - [Regras de detecção automática](#regras-de-detecção-automática)
    - [Níveis de severidade](#níveis-de-severidade)
    - [Formato de sugestão](#formato-de-sugestão)
    - [Priorização](#priorização)
  - [Referências](#referências)

---

## O que é SOLID?

SOLID é um acrônimo criado por **Michael Feathers** a partir de cinco princípios de design orientado a objetos identificados e promovidos por **Robert C. Martin (Uncle Bob)** no início dos anos 2000.

| Letra | Princípio                          | Autor original      | Ano de publicação formal |
|-------|------------------------------------|----------------------|--------------------------|
| **S** | Single Responsibility Principle    | Robert C. Martin     | 2003¹                    |
| **O** | Open/Closed Principle              | Bertrand Meyer       | 1988                     |
| **L** | Liskov Substitution Principle      | Barbara Liskov       | 1987²                    |
| **I** | Interface Segregation Principle    | Robert C. Martin     | 1996                     |
| **D** | Dependency Inversion Principle     | Robert C. Martin     | 1996                     |

> ¹ SRP foi articulado por Uncle Bob no paper *Design Principles and Design Patterns* (2000) e formalizado no livro *Agile Software Development* (2003). A reformulação "um ator" veio em *Clean Architecture* (2017).
>
> ² O paper formal de Liskov & Wing (*A Behavioral Notion of Subtyping*) é de 1994, mas o conceito foi apresentado por Barbara Liskov em sua keynote de 1987.

Esses princípios guiam o design de módulos, classes e interfaces para produzir software que seja:

- **Compreensível** — fácil de entender e navegar
- **Flexível** — fácil de estender sem efeitos colaterais
- **Manutenível** — fácil de modificar com confiança

---

## Por que SOLID importa?

Software sem princípios de design tende a deteriorar com o tempo, apresentando sintomas conhecidos como **design smells**:

| Sintoma         | Descrição                                                                 |
|-----------------|---------------------------------------------------------------------------|
| **Rigidez**     | Uma mudança simples exige alterações em cascata em vários módulos         |
| **Fragilidade**  | Mudanças em um módulo causam falhas inesperadas em partes não relacionadas |
| **Imobilidade**  | Componentes são tão acoplados que não podem ser reutilizados em outro contexto |
| **Viscosidade** | É mais fácil fazer a coisa errada do que a coisa certa dentro do design atual |
| **Complexidade desnecessária** | Abstrações prematuras e camadas que não resolvem problemas reais |

SOLID ataca diretamente esses sintomas ao fornecer **heurísticas concretas** para decisões de design.

> **Importante:** SOLID são *princípios*, não *regras absolutas*. Devem ser aplicados com **julgamento e contexto**. Seguir SOLID cegamente pode levar a over-engineering tão prejudicial quanto ignorá-los.

---

## S — Single Responsibility Principle (SRP)

### Definição

> *"Um módulo deve ter um, e apenas um, motivo para mudar."*
> — Robert C. Martin

Em termos mais precisos (reformulação de 2014):

> *"Um módulo deve ser responsável perante um, e apenas um, ator (stakeholder)."*

O SRP **não** diz que uma classe deve "fazer apenas uma coisa" — diz que ela deve **servir a apenas um ator** ou **motivo de mudança**.

### Motivação

Quando um módulo atende a múltiplos atores:
- Mudanças solicitadas por um ator podem quebrar funcionalidades usadas por outro
- Conflitos de merge tornam-se frequentes quando equipes diferentes alteram o mesmo arquivo
- O módulo cresce sem limite claro, tornando-se uma "God Class"

### Heurísticas para identificar violações

| Sinal                                | O que indica                                    |
|--------------------------------------|-------------------------------------------------|
| Classe com muitos métodos públicos   | Provavelmente atende a múltiplos atores          |
| Nome genérico (Manager, Handler, Utils, Helper) | Responsabilidade difusa                  |
| Mudanças frequentes por motivos diferentes | Múltiplos eixos de mudança              |
| Dificuldade de nomear a classe de forma precisa | Responsabilidade não está clara         |
| Muitas dependências injetadas (>4-5) | Classe faz coisas demais                        |
| Testes unitários excessivamente longos | Muitos cenários = muitas responsabilidades     |

### Boas Práticas

1. **Pergunte "quem pede mudanças neste módulo?"** — Se a resposta incluir mais de um ator (ex: "time de pagamentos" e "time de relatórios"), separe.

2. **Nomeie com precisão** — Se não consegue nomear a classe sem usar palavras genéricas (Manager, Processor, Handler), provavelmente ela tem muitas responsabilidades.

3. **Prefira composição** — Em vez de uma classe monolítica, componha comportamento a partir de colaboradores menores e focados.

4. **Separe leitura de escrita** — Operações de consulta e operações de modificação frequentemente atendem a atores diferentes (veja CQRS).

5. **Agrupe por coesão, não por tipo técnico** — Agrupe o que muda junto, não o que "parece" similar tecnicamente.

### Exemplo: antes e depois (pseudocódigo)

**ANTES — violação do SRP** (classe serve a múltiplos atores):

```
class Employee:
    calculatePay()        // ← Ator: Departamento Financeiro
    generateReport()      // ← Ator: Departamento de Relatórios
    saveToDatabase()      // ← Ator: DBA / Infraestrutura

    // Três motivos para mudar, três atores distintos
```

**DEPOIS — SRP aplicado** (cada classe serve a um ator):

```
class PayCalculator:
    calculatePay(employee)       // Ator: Financeiro

class EmployeeReporter:
    generateReport(employee)     // Ator: Relatórios

class EmployeeRepository:
    save(employee)               // Ator: Infraestrutura

// A classe Employee mantém apenas os dados e comportamento
// intrínsecos do domínio Employee
class Employee:
    name, role, salary
    promote(newRole)             // Comportamento de domínio puro
```

### Equívocos comuns

| Equívoco | Realidade |
|----------|-----------|
| "Uma classe deve fazer apenas uma coisa" | SRP fala de **um motivo para mudar** (um ator), não de um único método. Uma classe pode ter vários métodos desde que sirvam ao mesmo ator. |
| "Cada classe deve ter apenas um método" | Classes com um único método indicam over-decomposition. O critério é coesão por ator, não contagem de métodos. |
| "SRP = classes pequenas" | Tamanho é sintoma, não causa. Uma classe com 200 linhas pode ser SRP se todas servem ao mesmo ator. Uma com 50 linhas pode violar se serve a dois. |
| "Aplicar SRP desde o primeiro dia" | Em muitos casos, é melhor esperar a segunda mudança revelar o eixo de variação antes de separar prematuramente. |

### Anti-patterns comuns

| Anti-pattern       | Descrição                                                    |
|--------------------|--------------------------------------------------------------|
| **God Class**      | Uma classe que faz tudo — validação, persistência, formatação, envio de e-mail |
| **Util / Helper**  | Classes "lixeira" que acumulam métodos sem relação entre si  |
| **Smart Entity**   | Entidade de domínio que contém lógica de persistência, validação e formatação |

### Relação com outros princípios

- **ISP**: Classes com responsabilidade única naturalmente expõem interfaces coesas
- **OCP**: Módulos focados são mais fáceis de estender sem modificação

---

## O — Open/Closed Principle (OCP)

### Definição

> *"Entidades de software (classes, módulos, funções) devem ser abertas para extensão, mas fechadas para modificação."*
> — Bertrand Meyer (1988), refinado por Robert C. Martin

Isso significa que o **comportamento** de um módulo pode ser estendido **sem alterar seu código-fonte existente**.

### Motivação

- **Código testado não deve ser tocado**: Modificar código que já funciona introduz risco de regressão
- **Novos requisitos são inevitáveis**: O design deve acomodar mudanças futuras sem reescrever o que já existe
- **Deploy independente**: Módulos fechados para modificação podem ser deployados sem recompilar dependentes

### Mecanismos de extensão

O OCP se implementa através de abstrações. Os mecanismos mais comuns (independente de linguagem):

| Mecanismo                    | Como funciona                                              |
|------------------------------|------------------------------------------------------------|
| **Polimorfismo (herança/interface)** | Novos comportamentos via novas implementações de uma abstração |
| **Composição + Strategy**    | Injetar diferentes estratégias sem alterar o consumidor    |
| **Decorators / Wrappers**    | Adicionar comportamento ao redor de uma implementação existente |
| **Plugins / Registry**       | Registrar novos handlers em um registry sem tocar no core  |
| **Configuração / Dados**     | Novos comportamentos via configuração ou dados externos    |
| **Callbacks / Hooks**        | Pontos de extensão explícitos onde comportamento customizado pode ser injetado |

### Heurísticas para identificar violações

| Sinal                                          | O que indica                              |
|------------------------------------------------|-------------------------------------------|
| Cadeias de `if/else` ou `switch` sobre tipos   | Falta polimorfismo                        |
| Toda nova feature requer alterar código existente | Módulo não é extensível                 |
| Medo de mexer em código funcional              | Fragilidade por falta de abstrações       |
| "Shotgun surgery" — uma mudança toca muitos arquivos | Alto acoplamento, baixa extensibilidade |

### Boas Práticas

1. **Identifique os eixos de variação** — Pergunte "o que pode mudar?". Abstraia apenas o que *realmente* varia.

2. **Não abstraia prematuramente** — Espere pelo menos **dois exemplos concretos** de variação antes de criar uma abstração. Uma abstração prematura é pior que código duplicado.

3. **Use o padrão Strategy para variação de comportamento** — Quando um algoritmo pode variar, encapsule-o atrás de uma interface e injete a implementação.

4. **Use o padrão Template Method para variação de etapas** — Quando a estrutura é fixa mas passos individuais variam, defina o esqueleto e deixe subtipos implementar os passos.

5. **Prefira composição a herança** — Herança é um mecanismo forte de extensão, mas cria acoplamento rígido. Composição oferece mais flexibilidade.

### Exemplo: antes e depois (pseudocódigo)

**ANTES — violação do OCP** (toda nova forma de pagamento exige modificar código existente):

```
class PaymentProcessor:
    process(payment):
        if payment.type == "CREDIT_CARD":
            // lógica cartão de crédito
        else if payment.type == "BOLETO":
            // lógica boleto
        else if payment.type == "PIX":        // ← MUDANÇA: precisou abrir a classe
            // lógica pix
        // Cada novo método = modificação nesta classe
```

**DEPOIS — OCP aplicado** (novos métodos de pagamento sem tocar no código existente):

```
interface PaymentMethod:
    process(payment)

class CreditCardPayment implements PaymentMethod:
    process(payment):  // ...

class BoletoPayment implements PaymentMethod:
    process(payment):  // ...

class PixPayment implements PaymentMethod:     // ← EXTENSÃO: nova classe, sem tocar nas existentes
    process(payment):  // ...

class PaymentProcessor:
    constructor(method: PaymentMethod)         // Recebe a estratégia via injeção
    process(payment):
        method.process(payment)                // Delega — fechado para modificação
```

### Equívocos comuns

| Equívoco | Realidade |
|----------|-----------|
| "Nunca modifique código existente" | OCP é um **ideal direcional**, não uma proibição absoluta. Bug fixes e refatorações legítimas requerem modificação. O ponto é que **novos comportamentos** não devem exigir mudança no código estável. |
| "Toda classe precisa de uma interface" | Crie abstrações apenas quando houver **variação real ou provável**. Interface para uma única implementação é overhead (YAGNI). |
| "OCP = herança" | Herança é apenas um mecanismo. Composição, Strategy, Decorator e plugins também implementam OCP — frequentemente de forma superior. |
| "Código 100% OCP é possível" | Nenhum design é fechado para todas as mudanças. Escolha **quais eixos de variação** proteger com base nos requisitos do negócio. |

### Anti-patterns comuns

| Anti-pattern                   | Descrição                                                   |
|--------------------------------|-------------------------------------------------------------|
| **Type checking (if/switch)**  | Verificar o tipo concreto para decidir comportamento        |
| **Abstração prematura**        | Criar interfaces/abstrações sem dois exemplos reais de variação |
| **Framework-itis**             | Tornar tudo extensível "por precaução", criando complexidade desnecessária |

### Relação com outros princípios

- **LSP**: A extensão só funciona se subtipos respeitarem o contrato do tipo base
- **DIP**: Depender de abstrações (não de concretizações) é o que permite extensão
- **SRP**: Módulos com responsabilidade única são mais fáceis de estender pontualmente

---

## L — Liskov Substitution Principle (LSP)

### Definição

> *"Se S é um subtipo de T, então objetos do tipo T podem ser substituídos por objetos do tipo S sem alterar as propriedades desejáveis do programa."*
> — Barbara Liskov (1987)

Em termos práticos: **todo subtipo deve poder ser usado no lugar do tipo base sem surpresas**. O código que usa o tipo base não deve precisar saber qual subtipo está recebendo.

### Motivação

- **Polimorfismo confiável**: Se subtipos não respeitam o contrato do tipo base, o polimorfismo se torna imprevisível
- **Reutilização real**: Código genérico só funciona se todas as implementações se comportam de forma consistente
- **Testes válidos**: Se o subtipo quebra o contrato, testes que passam para o tipo base falham para o subtipo

### Regras formais

O LSP estabelece regras precisas sobre o que um subtipo pode e não pode fazer:

| Regra                         | Descrição                                                              |
|-------------------------------|------------------------------------------------------------------------|
| **Pré-condições**             | Um subtipo **não pode fortalecer** pré-condições (não pode exigir mais) |
| **Pós-condições**             | Um subtipo **não pode enfraquecer** pós-condições (não pode entregar menos) |
| **Invariantes**               | Um subtipo **deve preservar** todas as invariantes do tipo base        |
| **Regra do histórico**        | Um subtipo não pode introduzir mudanças de estado que o tipo base não permite |
| **Exceções**                  | Um subtipo não pode lançar exceções que o tipo base não especifica     |
| **Retorno (covariância)**     | O tipo de retorno pode ser mais específico (subtipo do retorno original) |
| **Parâmetros (contravariância)** | Os parâmetros podem ser mais genéricos (supertipo do parâmetro original) |

### Heurísticas para identificar violações

| Sinal                                          | O que indica                              |
|------------------------------------------------|-------------------------------------------|
| Subtipo lança exceção inesperada               | Pré-condição fortalecida                  |
| Subtipo ignora/esvazia um método herdado       | Pós-condição enfraquecida                 |
| Código cliente faz cast ou `instanceof` check  | Subtipo não é substituível transparente   |
| Subtipo rejeita inputs que o tipo base aceita  | Pré-condição fortalecida                  |
| Método retorna valor especial (null, -1) em vez de cumprir contrato | Contrato violado           |
| Herança usada para reutilizar código, não para modelar "é-um" | Relação semântica incorreta    |

### Boas Práticas

1. **Projete por contrato** — Defina explicitamente pré-condições, pós-condições e invariantes de cada abstração. Documente-as.

2. **Prefira composição quando a relação não é "é-um"** — Se o subtipo precisa desabilitar funcionalidades do tipo base, a hierarquia está errada. Use composição.

3. **Teste subtipos contra o contrato do tipo base** — Escreva testes genéricos que qualquer implementação da interface deve passar (contract tests).

4. **Evite herança para reuso de código** — Herança expressa uma relação semântica ("é-um"), não é um mecanismo de reuso. Para reuso, use composição ou delegação.

5. **Questione hierarquias profundas** — Quanto mais níveis de herança, mais difícil é garantir que todos os subtipos respeitem todos os contratos acumulados.

### Exemplo: antes e depois (pseudocódigo)

**ANTES — violação do LSP** (Square não é substituível por Rectangle):

```
class Rectangle:
    width, height

    setWidth(w):
        this.width = w

    setHeight(h):
        this.height = h

    area():
        return this.width * this.height

class Square extends Rectangle:
    setWidth(w):
        this.width = w
        this.height = w    // ← Efeito colateral: altera height!

    setHeight(h):
        this.width = h     // ← Efeito colateral: altera width!
        this.height = h

// Código cliente espera comportamento de Rectangle:
function resize(rect: Rectangle):
    rect.setWidth(5)
    rect.setHeight(10)
    assert rect.area() == 50   // ← FALHA com Square! area() == 100
```

**DEPOIS — LSP respeitado** (composição em vez de herança incorreta):

```
interface Shape:
    area(): number

class Rectangle implements Shape:
    constructor(width, height)
    area(): return this.width * this.height

class Square implements Shape:
    constructor(side)
    area(): return this.side * this.side

// Ambos implementam Shape sem promessas de comportamento mutável
// que não conseguem cumprir. O contrato é simples e respeitável.
```

### Equívocos comuns

| Equívoco | Realidade |
|----------|-----------|
| "LSP é só sobre herança de classes" | LSP se aplica a **qualquer relação de subtipagem**: interfaces, generics, duck typing, ou qualquer contrato que o cliente espera. |
| "Se compila, respeita LSP" | LSP é sobre **comportamento**, não sobre tipos. Um subtipo pode compilar perfeitamente e ainda violar contratos semânticos (ex: retornar null quando o contrato promete non-null). |
| "Herança é sempre 'é-um'" | A relação "é-um" do mundo real nem sempre mapeia para herança no software. Um quadrado "é um" retângulo na geometria, mas no código, Square não pode substituir Rectangle se os setters se comportam diferente. |
| "Basta não lançar exceção" | Não lançar exceção é necessário mas insuficiente. Pós-condições, invariantes e efeitos colaterais também devem ser preservados. |

### Anti-patterns comuns

| Anti-pattern                     | Descrição                                                       |
|----------------------------------|-----------------------------------------------------------------|
| **Refused Bequest**              | Subtipo herda métodos que não fazem sentido e os deixa vazios ou lança exceção |
| **Circle-Ellipse problem**       | Modelar um quadrado como subtipo de retângulo viola invariantes |
| **Herança por conveniência**     | Herdar para reaproveitar código sem relação semântica real      |
| **Exceções surpresa**            | Subtipo lança exceções que o tipo base nunca especificou        |

### Relação com outros princípios

- **OCP**: O OCP depende do LSP — extensão via polimorfismo só funciona se subtipos são substituíveis
- **ISP**: Interfaces segregadas reduzem a superfície de contrato, facilitando o cumprimento do LSP
- **DIP**: Depender de abstrações requer que implementações concretas respeitem o LSP

---

## I — Interface Segregation Principle (ISP)

### Definição

> *"Clientes não devem ser forçados a depender de interfaces que não utilizam."*
> — Robert C. Martin

Em termos práticos: **prefira várias interfaces específicas e coesas a uma interface grande e genérica**.

### Motivação

- **Acoplamento desnecessário**: Quando um cliente depende de uma interface "gorda", ele depende indiretamente de métodos que não usa
- **Recompilação em cascata**: Mudanças em métodos não utilizados pelo cliente ainda podem forçar recompilação
- **Implementações forçadas**: Classes que implementam interfaces grandes são forçadas a implementar métodos irrelevantes (violando potencialmente o LSP)
- **Testabilidade**: Interfaces menores são mais fáceis de mockar/stubar em testes

### Heurísticas para identificar violações

| Sinal                                                | O que indica                             |
|------------------------------------------------------|------------------------------------------|
| Interface com muitos métodos (>5-7)                  | Provavelmente atende a múltiplos clientes com necessidades diferentes |
| Implementações com métodos vazios ou que lançam `UnsupportedOperationException` | Interface gorda demais  |
| Clientes que usam apenas 2-3 métodos de uma interface com 10 | Acoplamento desnecessário       |
| Mudanças em métodos não relacionados afetam quem não os usa | Falta de segregação              |
| Dificuldade de nomear a interface sem palavras genéricas | Responsabilidades misturadas       |

### Boas Práticas

1. **Projete interfaces a partir do cliente** — Pergunte "o que este cliente precisa?" em vez de "o que esta implementação oferece?". Interfaces devem ser definidas pela **necessidade do consumidor**, não pela capacidade do fornecedor.

2. **Prefira role interfaces a header interfaces** — Uma *role interface* define um papel específico que o cliente espera. Uma *header interface* espelha todos os métodos de uma classe (anti-pattern).

3. **Composição de interfaces** — Uma classe pode implementar múltiplas interfaces pequenas. Clientes dependem apenas da interface que precisam.

4. **Aplique o princípio da menor interface** — Se o cliente precisa de apenas um método, crie uma interface com um método. Interfaces de um único método (functional interfaces / SAM) são perfeitamente válidas.

5. **Segregue por eixo de mudança** — Métodos que mudam juntos devem estar na mesma interface. Métodos que mudam por motivos diferentes devem ser separados.

### Exemplo: antes e depois (pseudocódigo)

**ANTES — violação do ISP** (interface gorda que força implementações desnecessárias):

```
interface Animal:
    eat()
    fly()
    swim()
    walk()

class Dog implements Animal:
    eat(): // ok
    fly(): throw UnsupportedOperationException  // ← Forçado a implementar!
    swim(): // ok
    walk(): // ok

class Fish implements Animal:
    eat(): // ok
    fly(): throw UnsupportedOperationException  // ← Forçado a implementar!
    swim(): // ok
    walk(): throw UnsupportedOperationException // ← Forçado a implementar!
```

**DEPOIS — ISP aplicado** (interfaces segregadas por capacidade):

```
interface Eater:
    eat()

interface Flyer:
    fly()

interface Swimmer:
    swim()

interface Walker:
    walk()

class Dog implements Eater, Swimmer, Walker:
    eat(): // ...
    swim(): // ...
    walk(): // ...

class Fish implements Eater, Swimmer:
    eat(): // ...
    swim(): // ...

class Eagle implements Eater, Flyer, Walker:
    eat(): // ...
    fly(): // ...
    walk(): // ...

// Cada classe implementa apenas o que faz sentido.
// Clientes dependem apenas da interface que precisam.
```

### Equívocos comuns

| Equívoco | Realidade |
|----------|-----------|
| "Cada interface deve ter um único método" | ISP não diz "um método por interface". Diz que **clientes não devem depender do que não usam**. Uma interface com 3-5 métodos coesos é perfeitamente aceitável. |
| "ISP só se aplica a interfaces explícitas" | Em linguagens com duck typing ou tipagem estrutural, o princípio se aplica a **qualquer contrato implícito**: parâmetros de função, structs, types, etc. |
| "Segregar tudo resulta em melhor design" | Segregação excessiva leva a uma explosão de interfaces minúsculas que dificultam a navegação e compreensão. O critério é **necessidade do cliente**, não atomicidade. |
| "ISP e SRP são a mesma coisa" | SRP fala sobre **motivos para mudar** (perspectiva do implementador). ISP fala sobre **dependências do cliente** (perspectiva do consumidor). São complementares, não idênticos. |

### Anti-patterns comuns

| Anti-pattern                | Descrição                                                          |
|-----------------------------|--------------------------------------------------------------------|
| **Fat Interface**           | Interface com dezenas de métodos que nenhum cliente usa por completo |
| **Header Interface**        | Interface que espelha 1:1 os métodos de uma classe concreta        |
| **God Interface**           | Uma interface para todas as operações do sistema                   |
| **Implementação parcial**   | Classes que implementam metade da interface com stubs vazios       |

### Relação com outros princípios

- **SRP**: Interfaces segregadas naturalmente refletem responsabilidades únicas
- **LSP**: Interfaces menores têm contratos mais simples, mais fáceis de cumprir por subtipos
- **DIP**: Clientes dependem de abstrações coesas (interfaces pequenas), não de detalhes

---

## D — Dependency Inversion Principle (DIP)

### Definição

> *"A. Módulos de alto nível não devem depender de módulos de baixo nível. Ambos devem depender de abstrações."*
>
> *"B. Abstrações não devem depender de detalhes. Detalhes devem depender de abstrações."*
> — Robert C. Martin

### Motivação

- **Módulos de alto nível contêm as regras de negócio** — são o coração do sistema e não devem ser afetados por mudanças em detalhes de infraestrutura (banco de dados, frameworks, I/O)
- **Testabilidade**: Quando a lógica de negócio depende de abstrações, é trivial substituir implementações reais por mocks/stubs nos testes
- **Flexibilidade de deploy**: Módulos podem ser deployados independentemente quando conectados via abstrações
- **Troca de tecnologia**: É possível trocar banco de dados, frameworks ou serviços externos sem tocar na lógica de negócio

### Conceitos fundamentais

| Conceito                         | Descrição                                                         |
|----------------------------------|-------------------------------------------------------------------|
| **Módulo de alto nível**         | Contém lógica de negócio, orquestra operações                    |
| **Módulo de baixo nível**        | Implementa detalhes técnicos (I/O, banco, rede, filesystem)     |
| **Abstração**                    | Interface ou tipo abstrato que define um contrato                |
| **Detalhe / Implementação**      | Classe concreta que implementa o contrato da abstração           |
| **Inversão**                     | Em vez de alto nível depender de baixo nível, ambos dependem de uma abstração que "pertence" ao alto nível |

**Direção de dependência (antes do DIP):**

```
[Regra de Negócio] ──depende──► [Banco de Dados]
```

**Direção de dependência (com DIP):**

```
[Regra de Negócio] ──depende──► [<<Abstração>> Repositório]
                                         ▲
[Implementação MySQL] ──implementa───────┘
```

A **abstração pertence ao módulo de alto nível** — é o negócio que define o que precisa, não a infraestrutura que define o que oferece.

### Heurísticas para identificar violações

| Sinal                                           | O que indica                              |
|-------------------------------------------------|-------------------------------------------|
| Lógica de negócio importa classes de framework  | Alto nível depende de baixo nível         |
| Instanciação direta de dependências com `new`   | Acoplamento a implementações concretas    |
| Impossível testar sem banco/rede/filesystem     | Dependência de detalhes concretos         |
| Trocar o banco exige alterar lógica de negócio  | Inversão de dependência não aplicada      |
| Módulos core importam módulos de infraestrutura | Direção de dependência invertida          |

### Boas Práticas

1. **Defina abstrações no módulo de alto nível** — A interface do repositório deve estar no pacote de domínio/negócio, não no pacote de infraestrutura. O módulo de alto nível **define** o contrato; o módulo de baixo nível **implementa**.

2. **Use injeção de dependência** — Não instancie dependências diretamente. Receba-as via construtor, método ou framework de DI. Isso permite substituição para testes e troca de implementação.

3. **Aplique a regra de dependência** — Dependências devem apontar **para dentro** (em direção ao domínio/negócio), nunca para fora (em direção a detalhes de infraestrutura).

4. **Dependa de abstrações estáveis** — Abstrações devem mudar com menos frequência que implementações concretas. Se a abstração muda o tempo todo, talvez não seja a abstração certa.

5. **Não abstraia tudo** — Classes concretas estáveis e que dificilmente mudarão (ex: tipos primitivos, estruturas de dados padrão) não precisam de abstração. O DIP se aplica a **componentes voláteis**.

### Exemplo: antes e depois (pseudocódigo)

**ANTES — violação do DIP** (alto nível depende diretamente do baixo nível):

```
class OrderService:
    constructor():
        this.repository = new MySQLOrderRepository()   // ← Dependência concreta!
        this.emailer = new SmtpEmailSender()           // ← Dependência concreta!
        this.logger = new FileLogger("/var/log/app")   // ← Dependência concreta!

    createOrder(order):
        this.repository.save(order)
        this.emailer.send(order.customer.email, "Pedido criado!")
        this.logger.log("Order created: " + order.id)

// Impossível testar sem MySQL, SMTP e filesystem reais
// Trocar MySQL por Postgres exige alterar lógica de negócio
```

**DEPOIS — DIP aplicado** (ambos dependem de abstrações):

```
// Abstrações definidas no módulo de DOMÍNIO (alto nível)
interface OrderRepository:
    save(order)

interface NotificationSender:
    notify(recipient, message)

interface Logger:
    log(message)

// Alto nível depende de abstrações
class OrderService:
    constructor(repo: OrderRepository, notifier: NotificationSender, logger: Logger):
        this.repository = repo
        this.notifier = notifier
        this.logger = logger

    createOrder(order):
        this.repository.save(order)
        this.notifier.notify(order.customer.email, "Pedido criado!")
        this.logger.log("Order created: " + order.id)

// Implementações em módulo de INFRAESTRUTURA (baixo nível)
class MySQLOrderRepository implements OrderRepository: // ...
class PostgresOrderRepository implements OrderRepository: // ...
class SmtpNotificationSender implements NotificationSender: // ...
class InMemoryOrderRepository implements OrderRepository: // Para testes!
```

### Equívocos comuns

| Equívoco | Realidade |
|----------|-----------|
| "DIP = Dependency Injection (DI)" | São conceitos **diferentes**. DIP é um **princípio de design** (depender de abstrações). DI é uma **técnica de implementação** (injetar dependências via construtor/método). DI ajuda a implementar DIP, mas não é sinônimo. |
| "Toda classe precisa de uma interface" | Abstrair classes estáveis (ex: `String`, `List`, utilitários puros) é overhead sem valor. DIP se aplica a **fronteiras voláteis** — onde implementações podem mudar. |
| "DIP é só para testes" | Testabilidade é um **benefício colateral**. O objetivo principal é proteger regras de negócio de mudanças em detalhes de infraestrutura e permitir evolução independente. |
| "Service Locator implementa DIP" | Service Locator **esconde** dependências em vez de declará-las explicitamente, dificultando a compreensão e testabilidade. Prefer a injeção via construtor. |

### Anti-patterns comuns

| Anti-pattern                       | Descrição                                                     |
|------------------------------------|---------------------------------------------------------------|
| **Dependência direta de framework**| Lógica de negócio usa classes de ORM, HTTP, etc. diretamente  |
| **Service Locator**                | Buscar dependências em runtime via registro global (esconde dependências) |
| **Abstração no pacote errado**     | Interface definida no módulo de infraestrutura (o oposto do DIP) |
| **"new" em lógica de negócio**     | Instanciar implementações concretas diretamente dentro de services |
| **Abstração espelho**              | Interface que 1:1 espelha a implementação concreta sem valor real |

### Relação com outros princípios

- **OCP**: O DIP é o **mecanismo** pelo qual se alcança o OCP — dependência de abstrações permite extensão sem modificação
- **LSP**: Só funciona se as implementações concretas respeitam o contrato da abstração
- **ISP**: Abstrações devem ser coesas e segregadas para que o DIP não crie acoplamentos desnecessários

---

## SOLID na Prática — Diretrizes Gerais

### Quando aplicar

| Situação                                        | Ação recomendada                                |
|-------------------------------------------------|-------------------------------------------------|
| Código novo com requisitos claros               | Aplique SRP e DIP desde o início                |
| Segundo caso de variação aparece                | Aplique OCP — crie a abstração agora            |
| Hierarquia de tipos sendo construída            | Valide LSP para cada subtipo                    |
| Interface crescendo muito                       | Aplique ISP — segregue                          |
| Refactoring de código legado                    | Aplique incrementalmente, priorizando SRP e DIP |
| Código descartável / protótipo                  | Seja pragmático — SOLID pode esperar            |

### Quando não aplicar (pragmatismo)

SOLID **não é dogma**. Existem situações onde o custo da abstração supera o benefício:

1. **Protótipos e MVPs** — Velocidade importa mais que design perfeito. Refatore depois.
2. **Código que não vai mudar** — Se o módulo é estável e não prevê variações, abstrações são overhead.
3. **Complexidade acidental** — Se aplicar um princípio torna o código mais difícil de entender, reconsidere. **Clareza supera pureza.**
4. **Equipes pequenas, escopo limitado** — Em projetos de escopo pequeno, o custo de manter abstrações pode ser maior que o benefício.
5. **Apenas um caso de uso** — Não crie interface para uma única implementação "por precaução" (YAGNI).

> **Regra de ouro:** *"Faça a coisa mais simples que funciona. Quando a simplicidade não for suficiente, aplique SOLID para gerenciar a complexidade."*

### Métricas e limiares de referência

Valores de referência para orientar decisões (não são regras rígidas):

| Métrica | Limiar saudável | Sinal de alerta | Princípio relacionado |
|---------|-----------------|------------------|-----------------------|
| Linhas por classe/módulo | < 200 | > 300 | SRP |
| Métodos públicos por classe | ≤ 7 | > 10 | SRP |
| Métodos por interface | ≤ 5 | > 7 | ISP |
| Dependências injetadas no construtor | ≤ 4 | > 5 | SRP |
| Profundidade de herança | ≤ 2 | > 3 | LSP |
| Branches em if/else sobre tipo | 0 (ideal) | ≥ 3 | OCP |
| Cobertura de testes unitários | > 80% | < 60% | DIP (testabilidade) |
| Imports de infraestrutura em domínio | 0 | > 0 | DIP |

> Esses valores são **heurísticas**, não dogmas. Contexto, linguagem e domínio afetam os limiares adequados.

### Code smells que indicam violação de SOLID

| Code Smell                     | Princípio violado | Ação                                          |
|-------------------------------|-------------------|-----------------------------------------------|
| God Class (classe com 500+ linhas) | SRP          | Extraia responsabilidades em classes focadas   |
| Cadeia de if/else sobre tipos  | OCP              | Substitua por polimorfismo (Strategy, Factory) |
| Subtipo com método vazio       | LSP              | Revise a hierarquia; use composição            |
| Interface com 15+ métodos      | ISP              | Segregue em interfaces menores por role        |
| Service instancia dependências | DIP              | Injete via construtor; defina interfaces       |
| Shotgun surgery                | SRP + OCP        | Agrupe o que muda junto                       |
| Feature envy                   | SRP              | Mova o comportamento para quem possui os dados |
| Refused bequest                | LSP              | Repense a hierarquia de herança               |
| Parallel class hierarchies     | OCP + DIP        | Unifique via composição e abstrações           |
| Long parameter list (>4-5 params) | SRP + ISP     | Extraia Parameter Object; verifique se o método faz coisas demais |
| Classe com nome genérico (Utils, Common, Base) | SRP | Identifique responsabilidades reais; renomeie ou separe |
| Variáveis globais/singletons mutáveis | DIP       | Substitua por injeção de dependência           |
| Testes com setup complexo (muitos mocks) | SRP + DIP | Classe tem dependências demais; separe responsabilidades |
| Código duplicado entre subtipos | LSP + OCP       | Extraia para classe base ou composição via delegação |

### Checklist de revisão

Use esta checklist em code reviews para validar aderência ao SOLID:

- [ ] **SRP** — Esta classe/módulo tem apenas um motivo para mudar?
- [ ] **SRP** — O nome da classe descreve precisamente sua responsabilidade?
- [ ] **OCP** — Para adicionar novo comportamento, preciso modificar código existente?
- [ ] **OCP** — Existe ponto de extensão claro para variações futuras?
- [ ] **LSP** — Todo subtipo pode substituir o tipo base sem surpresas?
- [ ] **LSP** — Algum subtipo ignora ou lança exceção em métodos herdados?
- [ ] **ISP** — O cliente usa todos os métodos da interface que consome?
- [ ] **ISP** — A interface pode ser dividida para atender melhor cada cliente?
- [ ] **DIP** — A lógica de negócio depende de abstrações ou de implementações concretas?
- [ ] **DIP** — A abstração pertence ao módulo de alto nível (domínio)?

### Receitas de refatoração

Passos concretos para corrigir as violações mais comuns:

#### Receita 1: God Class → SRP

1. **Identifique os atores** — Liste quem solicita mudanças nesta classe (ex: time financeiro, time de relatórios, DBA)
2. **Agrupe métodos por ator** — Coloque cada grupo em uma lista separada
3. **Extraia classe por grupo** — Crie uma nova classe para cada grupo de métodos com nome preciso
4. **Mova dependências** — Cada nova classe recebe apenas as dependências que seus métodos precisam
5. **Crie uma classe Facade (opcional)** — Se clientes existentes precisam de todos os métodos juntos, crie um facade que delega

#### Receita 2: if/else sobre tipos → OCP

1. **Identifique o eixo de variação** — Qual condição (`type`, `status`, `category`) determina o comportamento?
2. **Extraia interface** — Crie uma interface com o método que varia (ex: `PaymentMethod.process()`)
3. **Implemente por caso** — Cada branch do if/else vira uma classe que implementa a interface
4. **Injete a implementação** — O consumidor recebe a implementação via construtor ou factory
5. **Remova o if/else** — Substitua pela chamada polimórfica

#### Receita 3: Herança incorreta → LSP

1. **Identifique o contrato violado** — Qual pré-condição, pós-condição ou invariante o subtipo não respeita?
2. **Avalie a relação semântica** — O subtipo realmente "é-um" tipo base no contexto do software (não do mundo real)?
3. **Substitua por composição** — Se a relação não é "é-um", extraia uma interface comum e use composição/delegação
4. **Escreva contract tests** — Crie testes que qualquer implementação da interface deve passar

#### Receita 4: Fat Interface → ISP

1. **Liste os clientes** — Identifique quem consome a interface
2. **Mapeie uso por cliente** — Para cada cliente, liste quais métodos ele realmente usa
3. **Agrupe por padrão de uso** — Métodos usados pelos mesmos clientes formam uma role interface
4. **Segregue** — Crie as novas interfaces e faça a classe concreta implementar todas elas
5. **Atualize clientes** — Cada cliente passa a depender apenas da interface que precisa

#### Receita 5: Dependência direta → DIP

1. **Identifique a fronteira** — Onde está o limite entre lógica de negócio e infraestrutura?
2. **Extraia interface no domínio** — Crie a abstração no pacote/módulo de alto nível
3. **Mova a implementação para infraestrutura** — A classe concreta fica no módulo de baixo nível
4. **Injete via construtor** — O service de alto nível recebe a abstração no construtor
5. **Configure a composição** — Em um composition root (ou framework DI), conecte as implementações

---

## Tensões e Trade-offs entre Princípios

Na prática, os princípios SOLID podem criar **tensões** entre si. Reconhecer essas tensões é fundamental para tomar decisões pragmáticas.

| Tensão | Descrição | Como resolver |
|--------|-----------|---------------|
| **SRP vs. simplicidade** | Separar demais cria uma explosão de classes pequenas que dificulta navegação e compreensão do fluxo. | Aplique SRP quando houver **eixos de mudança reais**, não hipotéticos. Se dois conceitos sempre mudam juntos, podem ficar na mesma classe. |
| **OCP vs. YAGNI** | Criar pontos de extensão para variações que talvez nunca aconteçam gera complexidade acidental. | Espere pelo **segundo caso concreto** de variação antes de abstrair. Abstrair na primeira vez é prematuro. |
| **ISP vs. coesão** | Segregar interfaces ao extremo (1 método cada) pode fragmentar contratos que semanticamente pertencem juntos. | Agrupe métodos que o **mesmo cliente usa junto**. A granularidade deve refletir a necessidade real do consumidor, não atomicidade artificial. |
| **DIP vs. simplicidade** | Criar interfaces para toda dependência (incluindo estáveis) adiciona camadas de indireção sem valor. | Aplique DIP apenas nas **fronteiras voláteis** — onde a implementação pode mudar (banco, serviço externo, I/O). Dependências estáveis (String, List, utilitários puros) não precisam de abstração. |
| **LSP vs. reuso** | Herdar para reaproveitar código sem relação semântica "é-um" leva a subtipos que não respeitam o contrato. | Prefira **composição para reuso** e **herança apenas para polimorfismo** com contrato semântico real. |
| **SRP + ISP vs. DIP** | Segregar demais pode multiplicar interfaces a ponto de dificultar a configuração do container de DI. | Balance o número de abstrações com a complexidade de composição. Um facade pode unir role interfaces quando necessário. |

> **Princípio meta:** Quando princípios entram em conflito, priorize **clareza e testabilidade**. Se a aplicação de um princípio torna o código mais difícil de entender ou testar, reconsidere.

### Árvore de decisão rápida

```
Nova funcionalidade ou refatoração?
│
├── Código em classe com múltiplos atores?
│   └── SIM → Aplique SRP (separar por ator)
│
├── Nova variação de comportamento?
│   ├── É o 2º+ caso de variação → Aplique OCP (polimorfismo/Strategy)
│   └── É o 1º caso → Implemente direto (YAGNI); marque como candidato a abstração futura
│
├── Hierarquia de herança com subtipo problemático?
│   └── SIM → Valide LSP (contrato); considere composição
│
├── Interface grande com implementações parciais?
│   └── SIM → Aplique ISP (segregar por role/cliente)
│
├── Lógica de negócio depende de infraestrutura?
│   └── SIM → Aplique DIP (extrair interface no domínio + injetar)
│
└── Nenhum sinal claro → Mantenha simples; reavalie no próximo ciclo de mudança
```

---

## SOLID em Arquiteturas Modernas

Os princípios SOLID transcendem classes e se aplicam a **módulos, serviços e componentes** em arquiteturas modernas:

### Hexagonal Architecture (Ports & Adapters)

| Princípio | Aplicação na Hexagonal Architecture |
|-----------|--------------------------------------|
| **SRP** | Cada **port** (porta) é responsável por um tipo de interação (entrada/saída) específico |
| **OCP** | Novos **adapters** podem ser adicionados sem alterar o domínio (core) |
| **LSP** | Qualquer adapter deve respeitar o contrato definido pela port |
| **ISP** | Ports devem ser segregadas — uma port por tipo de interação, não uma "God Port" |
| **DIP** | O domínio define as ports (abstrações) → adapters (detalhes) as implementam. Dependência aponta para dentro |

### Microservices

| Princípio | Aplicação em Microservices |
|-----------|---------------------------|
| **SRP** | Cada serviço é responsável por um **bounded context** — um domínio de negócio coeso |
| **OCP** | Novos serviços estendem o sistema sem alterar os existentes; eventos permitem extensão desacoplada |
| **LSP** | Versionamento de APIs: v2 deve ser retro-compatível com clientes de v1 quando possível |
| **ISP** | APIs devem expor apenas o que cada consumidor precisa (BFF pattern, GraphQL) |
| **DIP** | Serviços dependem de contratos (schemas, protobuf, eventos), não de implementações internas de outros serviços |

### Clean Architecture / Onion Architecture

| Camada | Princípio dominante | Regra |
|--------|---------------------|-------|
| **Entities** | SRP + LSP | Entidades de domínio puras, sem dependências externas |
| **Use Cases** | SRP + OCP | Cada caso de uso é uma unidade coesa; novos comportamentos = novos use cases |
| **Interface Adapters** | DIP + ISP | Adaptam portas do domínio para frameworks/drivers externos |
| **Frameworks & Drivers** | DIP | Detalhes de implementação; dependem das abstrações definidas pelas camadas internas |

### Event-Driven Architecture

| Princípio | Aplicação |
|-----------|-----------|
| **SRP** | Cada handler de evento é responsável por uma reação a um evento específico |
| **OCP** | Novos handlers podem reagir a eventos existentes sem alterar produtores |
| **DIP** | Produtores não conhecem consumidores — ambos dependem do contrato do evento |

---

## SOLID além de OOP — Aplicação em Paradigma Funcional

SOLID foi concebido no contexto de OOP, mas os **princípios subjacentes** se traduzem para programação funcional. A tabela abaixo mapeia cada princípio para seu equivalente funcional:

| Princípio | Equivalente funcional | Explicação |
|-----------|----------------------|------------|
| **SRP** | **Funções puras e pequenas** | Cada função faz uma única transformação. Módulos/namespaces agrupam funções que mudam pelo mesmo motivo. |
| **OCP** | **Higher-order functions + composição** | Em vez de modificar uma função existente, passe comportamento novo via funções de alta ordem (map, filter, reduce customizados). Pipelines de composição permitem extensão sem modificação. |
| **LSP** | **Contratos de tipos / type classes** | Funções que operam sobre um tipo genérico (typeclass, protocol, trait) esperam que toda instância respeite as leis algébricas (identidade, associatividade, etc.). Equivalente direto do LSP. |
| **ISP** | **Parâmetros mínimos + tipagem estrutural** | Funções recebem apenas os dados que precisam (destructuring mínimo), não uma estrutura inteira. Em linguagens com row types ou structural typing, o consumidor define exatamente o shape que precisa. |
| **DIP** | **Injeção via parâmetros / Reader Monad** | Em vez de uma função importar diretamente um módulo de infraestrutura, ela recebe dependências como parâmetros (funções ou records de funções). O Reader Monad formaliza esse padrão. |

### Exemplo funcional: DIP via higher-order function

```
// ANTES — Acoplado: a função importa o módulo de banco diretamente
processOrder(order):
    db.save(order)              // ← dependência direta
    email.send(order.customer)  // ← dependência direta

// DEPOIS — Desacoplado: dependências injetadas como parâmetros
processOrder(order, saveFn, notifyFn):
    saveFn(order)
    notifyFn(order.customer)

// Uso em produção:
processOrder(order, db.save, email.send)

// Uso em testes:
processOrder(order, inMemorySave, noopNotify)
```

### Exemplo funcional: OCP via composição de funções

```
// Extensão sem modificação — adicionar validações via composição
validate = compose(
    validateEmail,
    validateAge,
    validateName
)

// Estender: adicionar nova regra sem alterar as existentes
validateExtended = compose(
    validatePhone,    // ← NOVA — sem tocar nas existentes
    validateEmail,
    validateAge,
    validateName
)
```

> **Insight:** O paradigma funcional naturalmente favorece muitos princípios SOLID — imutabilidade elimina problemas de estado compartilhado (LSP), funções pequenas promovem SRP, e higher-order functions viabilizam OCP e DIP sem necessidade de classes/interfaces.

---

## Relação entre os Princípios

Os cinco princípios não são independentes — eles se **reforçam mutuamente**:

```
         ┌──────────────────────────────────────────────────────┐
         │                                                      │
         │   SRP                                                │
         │   "Um motivo para mudar"                             │
         │       │                                              │
         │       ├── Promove ──► ISP (interfaces coesas)        │
         │       │                                              │
         │       └── Facilita ──► OCP (módulos focados          │
         │                              são mais extensíveis)   │
         │                                                      │
         │   OCP                                                │
         │   "Aberto para extensão, fechado para modificação"   │
         │       │                                              │
         │       ├── Depende de ──► LSP (subtipos confiáveis)   │
         │       │                                              │
         │       └── Implementado via ──► DIP (abstrações)      │
         │                                                      │
         │   LSP                                                │
         │   "Subtipos são substituíveis"                       │
         │       │                                              │
         │       └── Facilitado por ──► ISP (contratos menores) │
         │                                                      │
         │   ISP                                                │
         │   "Interfaces coesas e focadas"                      │
         │       │                                              │
         │       └── Reflete ──► SRP (responsabilidade única)   │
         │                                                      │
         │   DIP                                                │
         │   "Dependa de abstrações"                            │
         │       │                                              │
         │       ├── Viabiliza ──► OCP (extensão sem modificar) │
         │       │                                              │
         │       └── Requer ──► LSP (implementações confiáveis) │
         │                                                      │
         └──────────────────────────────────────────────────────┘
```

**Resumo da sinergia:**

| De → Para | Relação                                                           |
|-----------|-------------------------------------------------------------------|
| SRP → ISP | Responsabilidade única leva a interfaces coesas                   |
| SRP → OCP | Módulos focados são mais fáceis de estender                       |
| OCP → DIP | Extensão sem modificação requer dependência de abstrações         |
| OCP → LSP | Extensão via polimorfismo requer que subtipos cumpram o contrato  |
| LSP → ISP | Contratos menores são mais fáceis de cumprir por subtipos         |
| DIP → LSP | Implementações concretas devem respeitar a abstração do alto nível |
| ISP → SRP | Interfaces segregadas refletem responsabilidades únicas           |

---

## Diretrizes para Code Review assistido por AI

> Esta seção define **regras acionáveis** para que um assistente de código (Copilot, AI reviewer) identifique e sugira correções para violações SOLID.

### Regras de detecção automática

Ao revisar código, o assistente deve aplicar estas regras de detecção:

| # | Regra de detecção | Princípio | Ação sugerida |
|---|-------------------|-----------|---------------|
| 1 | Classe com **mais de 300 linhas** ou **mais de 8 métodos públicos** | SRP | Sugerir extração de responsabilidades por ator/eixo de mudança |
| 2 | Nome de classe contém **Manager, Handler, Processor, Utils, Helper, Service** genérico | SRP | Questionar se a classe serve a múltiplos atores; sugerir nome mais específico |
| 3 | **Cadeia if/else ou switch/case** que verifica tipo, status ou categoria para decidir comportamento | OCP | Sugerir substituição por polimorfismo (Strategy, Factory) |
| 4 | **Novo requisito requer modificar** classe/função existente em vez de adicionar nova | OCP | Sugerir ponto de extensão (interface + implementação) |
| 5 | Subtipo com **método vazio, throw UnsupportedOperationException ou retorno de valor sentinela** (null, -1) | LSP | Sugerir revisão da hierarquia; recomendar composição |
| 6 | Código cliente faz **type cast ou instanceof/type check** para decidir comportamento | LSP | Indicar que o subtipo não é substituível; sugerir refatoração |
| 7 | Interface com **mais de 7 métodos** | ISP | Sugerir segregação em role interfaces menores |
| 8 | Implementação de interface com **métodos stub (vazios ou exception)** | ISP + LSP | Sugerir segregação da interface |
| 9 | Classe de negócio **instancia dependência com new** (repositório, client HTTP, logger) | DIP | Sugerir injeção de dependência via construtor + interface |
| 10 | **Import de módulo de infraestrutura** dentro de módulo de domínio/negócio | DIP | Alertar inversão de dependência; abstração deve estar no domínio |
| 11 | **Mais de 5 dependências injetadas** no construtor | SRP | Sugerir que a classe tem responsabilidades demais |
| 12 | **Testes requerem infraestrutura real** (banco, rede, filesystem) para executar | DIP | Sugerir abstração de dependências externas e uso de mocks/stubs |
| 13 | **Classe estática / singleton mutável** usado como dependência global | DIP | Sugerir substituição por injeção de dependência; singletons mutáveis dificultam testes e escondem acoplamento |
| 14 | **Método com mais de 4-5 parâmetros** de tipos diferentes | SRP + ISP | Sugerir extração de Parameter Object ou verificar se o método faz coisas demais |
| 15 | **Herança com mais de 3 níveis** de profundidade | LSP | Alertar que hierarquias profundas aumentam risco de violação de contratos; sugerir achatar com composição |
| 16 | **Classe abstrata com lógica concreta** significativa (>50% do código) | SRP + LSP | Questionar se a motivação é reuso ou polimorfismo; sugerir extração para classe colaboradora |
| 17 | **Enum com comportamento** (switch em enum para decidir lógica) em vez de polimorfismo | OCP | Sugerir substituição por Strategy ou mapa de comportamentos |
| 18 | **Retorno de null** em método que deveria cumprir contrato | LSP | Sugerir Optional/Maybe, Null Object Pattern ou exceção explícita |

### Níveis de severidade

| Severidade | Quando usar | Exemplo |
|------------|-------------|---------|
| **Crítico** | Viola múltiplos princípios simultaneamente; impacta testabilidade e manutenibilidade | God Class com 1000+ linhas que instancia dependências diretamente |
| **Alto** | Viola um princípio de forma clara; dificuldade futura garantida | Cadeia de 8 if/else sobre tipos sem abstração |
| **Médio** | Viola parcialmente; impacto depende do contexto e da frequência de mudança | Interface com 10 métodos onde alguns clientes usam apenas 3 |
| **Baixo** | Code smell leve; pode ser válido no contexto (protótipo, módulo estável) | Classe com 250 linhas com leve tendência a múltiplas responsabilidades |

### Formato de sugestão

Ao identificar uma violação, o assistente deve estruturar a sugestão assim:

```
**Princípio violado:** [SRP | OCP | LSP | ISP | DIP]
**Severidade:** [Crítico | Alto | Médio | Baixo]
**Problema:** Descrição objetiva do que está errado
**Impacto:** Como isso prejudica manutenibilidade, testabilidade ou extensibilidade
**Sugestão:** Passos concretos para corrigir (referenciando as receitas de refatoração acima)
**Exemplo:** Pseudocódigo ou esboço do design corrigido
```

### Priorização

Quando múltiplas violações coexistem, priorize nesta ordem:

1. **DIP** — Corrigir direção de dependência primeiro (habilita testabilidade imediata)
2. **SRP** — Separar responsabilidades (reduz escopo de cada componente)
3. **OCP** — Introduzir pontos de extensão (previne futuras mudanças em cascata)
4. **ISP** — Segregar interfaces (reduz acoplamento desnecessário)
5. **LSP** — Corrigir hierarquias (garante polimorfismo confiável)

> **Princípio geral:** Ataque primeiro o que **desbloqueia testabilidade** e depois o que **previne deterioração futura**.

### Exemplos de feedback estruturado

Exemplos concretos de como o assistente deve formatar sugestões em code review:

**Exemplo 1 — God Class (SRP + DIP)**

```
**Princípio violado:** SRP + DIP
**Severidade:** Crítico
**Problema:** `OrderService` (480 linhas) acumula validação, cálculo de frete,
persistência e envio de notificação. Instancia `MySQLRepository` e `SmtpMailer` diretamente.
**Impacto:** Impossível testar lógica de negócio sem banco e SMTP reais.
Mudança no cálculo de frete pode quebrar notificação inadvertidamente.
**Sugestão:**
1. Extrair `FreightCalculator`, `OrderValidator`, `OrderNotifier` (SRP — Receita 1)
2. Criar interfaces `OrderRepository`, `Notifier` no domínio (DIP — Receita 5)
3. Injetar via construtor
**Exemplo:**
    class OrderService(repo: OrderRepository, freight: FreightCalculator, notifier: Notifier):
        createOrder(order):
            OrderValidator.validate(order)
            order.freight = freight.calculate(order)
            repo.save(order)
            notifier.notify(order.customer, "Pedido criado!")
```

**Exemplo 2 — Switch sobre tipo (OCP)**

```
**Princípio violado:** OCP
**Severidade:** Alto
**Problema:** `NotificationService.send()` contém switch com 6 cases (email, sms,
push, slack, teams, webhook) que cresce a cada novo canal.
**Impacto:** Todo novo canal exige modificar classe existente; risco de regressão alto.
**Sugestão:**
1. Extrair interface `NotificationChannel` com método `send(recipient, message)` (Receita 2)
2. Cada canal vira uma implementação: `EmailChannel`, `SmsChannel`, etc.
3. Injetar lista de canais ou usar Registry pattern
**Exemplo:**
    interface NotificationChannel:
        send(recipient, message)
        supports(channelType): boolean

    class NotificationService(channels: List<NotificationChannel>):
        send(recipient, message, channelType):
            channel = channels.find(c -> c.supports(channelType))
            channel.send(recipient, message)
```

---

## Perguntas Frequentes (FAQ)

> Perguntas recorrentes sobre SOLID, formatadas para consulta rápida por humanos e AI.

**P: SOLID se aplica apenas a orientação a objetos?**
R: Não. Os **princípios subjacentes** (coesão, desacoplamento, substituibilidade, contratos mínimos, inversão de dependência) transcendem paradigmas. Em FP, manifestam-se via funções puras, composição, higher-order functions e tipagem. Veja a seção [SOLID além de OOP](#solid-além-de-oop--aplicação-em-paradigma-funcional).

**P: Devo aplicar todos os 5 princípios simultaneamente em todo código?**
R: Não. SOLID são **heurísticas direcionais**, não regras absolutas. Aplique o princípio relevante quando houver indício claro de problema (code smell) ou quando o contexto exigir. Protótipos, scripts e código descartável podem ser pragmaticamente simples. Veja [Quando não aplicar](#quando-não-aplicar-pragmatismo).

**P: Qual é a diferença entre DIP e DI (Dependency Injection)?**
R: **DIP** é um *princípio de design*: "dependa de abstrações, não de concretizações". **DI** é uma *técnica de implementação*: fornecer dependências externamente via construtor, setter ou framework. DI é uma das formas de implementar DIP, mas não a única (higher-order functions em FP também implementam DIP).

**P: Como sei quando uma classe viola SRP vs. ISP?**
R: Perspectiva do **implementador** → SRP: "esta classe tem múltiplos motivos para mudar?" Perspectiva do **consumidor** → ISP: "o cliente usa todos os métodos desta interface?" Ambas podem coexistir: uma God Class (SRP) frequentemente expõe uma God Interface (ISP).

**P: Interface com uma única implementação viola YAGNI?**
R: Depende. Se a interface existe apenas "por precaução" sem variação real prevista, sim, é overhead (YAGNI). Se existe para **definir fronteira arquitetural** (ex: `OrderRepository` no domínio, implementado por `MySQLOrderRepository` na infra), é DIP legítimo — mesmo com uma única implementação. O critério é: a abstração **protege uma fronteira volátil**?

**P: Como SOLID se relaciona com Clean Architecture?**
R: Clean Architecture é uma **aplicação arquitetural** dos princípios SOLID em larga escala. A Regra de Dependência (dependências apontam para dentro) é DIP aplicado entre camadas. Cada camada respeita SRP. Use Cases são OCP (novos use cases estendem o sistema). Veja [Clean Architecture / Onion Architecture](#clean-architecture--onion-architecture).

**P: Posso aplicar SOLID em código legado sem reescrever tudo?**
R: Sim. A abordagem recomendada é **incremental**: identifique os pontos de maior dor (classes que mudam frequentemente, módulos impossíveis de testar), aplique as [receitas de refatoração](#receitas-de-refatoração) uma por vez, e escreva testes antes de refatorar. Priorize DIP primeiro (habilita testabilidade) e SRP em seguida (reduz escopo de risco).

**P: "Composition over inheritance" significa nunca usar herança?**
R: Não. Herança é apropriada quando existe relação semântica "é-um" genuína no contexto do software (não apenas no mundo real) E o subtipo respeita o contrato do tipo base (LSP). Use composição quando o objetivo for **reuso de comportamento** sem relação semântica de subtipagem.

---

---

## Relação com Design Patterns

> Os princípios SOLID são a **base teórica** sobre a qual os Design Patterns são construídos. A tabela abaixo mapeia cada princípio aos padrões que ajudam a implementá-lo.

| Princípio | Padrões que ajudam a aplicá-lo |
|-----------|-------------------------------|
| **SRP** | [Facade](03-structural-patterns.md#facade), [Mediator](04-behavioral-patterns.md#mediator), [Command](04-behavioral-patterns.md#command) |
| **OCP** | [Strategy](04-behavioral-patterns.md#strategy), [Decorator](03-structural-patterns.md#decorator), [Observer](04-behavioral-patterns.md#observer), [Template Method](04-behavioral-patterns.md#template-method) |
| **LSP** | [Factory Method](02-creational-patterns.md#factory-method), [Abstract Factory](02-creational-patterns.md#abstract-factory) (garantem subtipos corretos) |
| **ISP** | [Adapter](03-structural-patterns.md#adapter), [Facade](03-structural-patterns.md#facade) (expõem interfaces segregadas) |
| **DIP** | [Abstract Factory](02-creational-patterns.md#abstract-factory), [Strategy](04-behavioral-patterns.md#strategy), [Hexagonal](05-architectural-patterns.md#hexagonal-architecture-ports--adapters) |

### Guia de decisão: qual princípio aplicar primeiro?

```
Analisando código problemático?
│
├── Classe faz coisas demais / muitos motivos para mudar?
│   └── Aplicar SRP (dividir responsabilidades)
│
├── Precisa modificar código existente para cada nova feature?
│   └── Aplicar OCP (criar pontos de extensão)
│
├── Subclasse quebra comportamento esperado?
│   └── Verificar LSP (repensar hierarquia ou usar composição)
│
├── Classe forçada a implementar métodos que não usa?
│   └── Aplicar ISP (segregar interfaces)
│
└── Lógica de negócio depende de infraestrutura (banco, HTTP, framework)?
    └── Aplicar DIP (extrair interface + injeção de dependência)
```

> **Dica:** Design Patterns são **implementações concretas** dos princípios SOLID.
> Entender SOLID primeiro torna o aprendizado de patterns muito mais natural.
> **Veja também:** [Boas Práticas de Design](06-best-practices.md), [Padrões Arquiteturais](05-architectural-patterns.md)

---

## Referências

| Recurso                                                    | Autor                | Tipo      | Relevância SOLID |
|------------------------------------------------------------|----------------------|-----------|------------------|
| *Agile Software Development: Principles, Patterns, and Practices* | Robert C. Martin | Livro | Definição original dos 5 princípios |
| *Clean Architecture: A Craftsman's Guide to Software Structure and Design* | Robert C. Martin | Livro | SRP reformulado; SOLID em escala arquitetural |
| *Object-Oriented Software Construction*                    | Bertrand Meyer       | Livro     | OCP original; Design by Contract |
| *A Behavioral Notion of Subtyping* (1994)                  | Barbara Liskov, Jeannette Wing | Paper | Formalização do LSP |
| *Design Principles and Design Patterns* (2000)             | Robert C. Martin     | Paper     | Paper seminal que articula os 5 princípios |
| *The Pragmatic Programmer*                                 | Hunt & Thomas        | Livro     | Pragmatismo e trade-offs em design |
| *Refactoring: Improving the Design of Existing Code* (2ª ed.) | Martin Fowler     | Livro     | Receitas de refatoração aplicáveis |
| *Head First Design Patterns*                               | Freeman & Robson     | Livro     | Patterns que implementam OCP, DIP |
| *Effective Java* (3ª ed.)                                  | Joshua Bloch         | Livro     | ISP e LSP em contexto Java |
| *Domain-Driven Design: Tackling Complexity in the Heart of Software* | Eric Evans    | Livro     | SRP e DIP em design de domínio |
| *Working Effectively with Legacy Code*                     | Michael Feathers     | Livro     | SOLID incremental em código legado |
| *Implementing Domain-Driven Design*                        | Vaughn Vernon        | Livro     | DIP e SRP em arquitetura hexagonal |
