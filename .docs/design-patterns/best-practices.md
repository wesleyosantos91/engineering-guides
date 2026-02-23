# Boas Práticas de Design de Software

> **Agnóstico a linguagem** — Princípios, heurísticas e diretrizes que guiam decisões
> de design independentemente da linguagem ou framework escolhidos.
> Complementam os princípios SOLID e são a base para um código sustentável.

---

## Sumário

- [Composição sobre Herança](#composição-sobre-herança)
- [DRY — Don't Repeat Yourself](#dry--dont-repeat-yourself)
- [KISS — Keep It Simple, Stupid](#kiss--keep-it-simple-stupid)
- [YAGNI — You Ain't Gonna Need It](#yagni--you-aint-gonna-need-it)
- [Lei de Demeter (Principle of Least Knowledge)](#lei-de-demeter-principle-of-least-knowledge)
- [Tell, Don't Ask](#tell-dont-ask)
- [Fail Fast](#fail-fast)
- [Separation of Concerns](#separation-of-concerns)
- [Program to an Interface, Not an Implementation](#program-to-an-interface-not-an-implementation)
- [Princípio da Menor Surpresa](#princípio-da-menor-surpresa)
- [Encapsulamento](#encapsulamento)
- [Coesão e Acoplamento](#coesão-e-acoplamento)
- [Imutabilidade](#imutabilidade)
- [Defensive Programming](#defensive-programming)
- [Anti-Patterns Comuns](#anti-patterns-comuns)
- [Checklist de Code Review](#checklist-de-code-review)

---

## Composição sobre Herança

### Princípio

> *"Favoreça composição de objetos sobre herança de classes."* — Gang of Four

### Por que preferir composição

| Aspecto | Herança | Composição |
|---------|---------|------------|
| **Binding** | Compile time (estático) | Runtime (dinâmico) |
| **Acoplamento** | Alto (subclasse ↔ superclasse) | Baixo (via interface) |
| **Flexibilidade** | Uma hierarquia fixa | Combina comportamentos livremente |
| **Reutilização** | Limitada à hierarquia | Via delegação a qualquer objeto |
| **Testabilidade** | Difícil mockear superclasse | Fácil substituir dependência |

### Quando herança AINDA faz sentido

- Relação genuína **"é um"** (não "tem um" ou "usa um").
- Estrutura de tipos que segue LSP com confiança.
- Template Method Pattern (esqueleto fixo com passos variáveis).
- Frameworks que exigem herança (ex: classes base de teste).

### Regra prática

> Se pensou em herança, pergunte: "Estou estendendo **comportamento** ou **tipo**?"
> - Estendendo comportamento → **Composição** (Strategy, Decorator).
> - Estendendo tipo verdadeiro → **Herança** (com cuidado, respeitando LSP).

---

## DRY — Don't Repeat Yourself

### Princípio

> *"Every piece of knowledge must have a single, unambiguous, authoritative representation
> within a system."* — Andrew Hunt & David Thomas, *The Pragmatic Programmer*

### O que DRY **realmente** significa

DRY não é apenas sobre código duplicado. É sobre **conhecimento duplicado**:

| Tipo de duplicação | Exemplo | Solução |
|-------------------|---------|---------|
| **Código** | Mesma lógica copiada em dois lugares | Extrair função/método |
| **Dados** | Mesma informação em duas tabelas | Normalizar |
| **Lógica** | Regra de negócio em service + controller | Centralizar no service |
| **Documentação** | README desatualizado vs código | Gerar docs automaticamente |

### Quando NÃO aplicar DRY

- **Coincidência ≠ Duplicação:** dois trechos iguais hoje podem evoluir diferentemente.
- **Regra dos 3:** tolere duplicação até a terceira ocorrência antes de abstrair.
- **Acoplamento indesejado:** unificar pode criar dependência entre módulos não relacionados.
- **Microserviços:** alguma duplicação entre serviços é aceitável para manter autonomia.

### WET — Write Everything Twice

> É melhor ter código duplicado do que uma abstração errada.
> *"Duplication is far cheaper than the wrong abstraction."* — Sandi Metz

---

## KISS — Keep It Simple, Stupid

### Princípio

> *"A maioria dos sistemas funciona melhor quando são mantidos simples,
> em vez de tornados complexos."*

### Sinais de complexidade desnecessária

| Sinal | Descrição |
|-------|-----------|
| O dev precisa de 30 min para entender uma classe | Classe faz demais ou é abstrata demais |
| 5 camadas de indireção para salvar um registro | Over-engineering |
| Design patterns usados sem problema real | Pattern fever |
| Generalizações para casos que nunca aconteceram | Especulação |
| Nomes de classes/métodos incompreensíveis | Falta de clareza |

### Como manter simples

1. **Resolva o problema de hoje**, não o de amanhã (YAGNI).
2. **Nomeie bem:** se é difícil nomear, talvez a responsabilidade esteja errada.
3. **Prefira código óbvio:** "clever code" é inimigo de manutenibilidade.
4. **Menos indireção:** cada camada de abstração tem custo cognitivo.
5. **Refatore quando necessário:** simplicidade é um alvo em movimento.

---

## YAGNI — You Ain't Gonna Need It

### Princípio

> *"Não implemente algo até que seja realmente necessário."*

### Aplicação

| Situação | YAGNI diz |
|----------|-----------|
| "E se no futuro precisarmos de suporte a XML?" | Implemente quando (e SE) precisar |
| "Vou criar uma abstração para trocar o banco facilmente" | Você vai trocar o banco? Quando? |
| "Vou adicionar 3 interfaces 'por precaução'" | Interfaces sem variação real são ruído |
| "Vou construir um plugin system genérico" | Tem requisito concreto para plugins? |

### YAGNI vs Bom Design

YAGNI **NÃO** significa:
- Ignorar princípios de design.
- Escrever código sem estrutura.
- Nunca pensar no futuro.

YAGNI **SIGNIFICA:**
- Não implementar features especulativas.
- Construir a abstração certa quando a necessidade é real.
- Evitar over-engineering.

---

## Lei de Demeter (Principle of Least Knowledge)

### Princípio

> *"Fale apenas com seus amigos imediatos."*

Um método de um objeto deve chamar apenas métodos de:
1. Ele mesmo (`this`/`self`).
2. Seus parâmetros.
3. Objetos que ele cria.
4. Seus atributos diretos.

### Violação clássica

```
// RUIM: conhece a estrutura interna de vários objetos
order.getCustomer().getAddress().getCity().getName()

// BOM: pede diretamente o que precisa
order.getDeliveryCity()
```

### Por que importa

- **Acoplamento reduzido:** mudanças internas não propagam para clientes distantes.
- **Encapsulamento preservado:** objetos não expõem sua estrutura interna.
- **Manutenibilidade:** menos "efeito dominó" em refatorações.

### Trade-offs

- Pode gerar muitos métodos "wrapper" (delegação excessiva).
- Em DTOs/Value Objects simples (sem comportamento), acessar propriedades aninhadas pode ser ok.
- Use bom senso: a lei é uma **heurística**, não uma regra absoluta.

---

## Tell, Don't Ask

### Princípio

> *"Diga aos objetos o que fazer, não pergunte sobre seu estado para decidir."*

### Comparação

```
// ASK (ruim): pergunta o estado e decide externamente
if (account.getBalance() >= amount) {
    account.setBalance(account.getBalance() - amount)
}

// TELL (bom): diz o que fazer — o objeto decide internamente
account.withdraw(amount)  // lança exceção se saldo insuficiente
```

### Por que importa

- **Encapsulamento:** a lógica fica no objeto que possui os dados.
- **Coesão:** comportamento e dados ficam juntos.
- **SRP:** o chamador não precisa conhecer regras internas do objeto.

### Quando "Ask" é aceitável

- Consultas puras (queries) que não alteram estado.
- DTOs / View Models (objetos sem comportamento, apenas dados).
- Verificações em pontos de integração (ex: validação de input na borda).

---

## Fail Fast

### Princípio

> *"Se algo vai falhar, falhe o mais cedo possível com uma mensagem clara."*

### Aplicação

| Onde | Como |
|------|------|
| **Validação de input** | Valide parâmetros no início do método |
| **Pré-condições** | Asserte invariantes antes de processar |
| **Configuração** | Verifique configurações obrigatórias no startup |
| **Dependências** | Teste conectividade com serviços no boot |
| **Type system** | Use tipos fortes para evitar erros em runtime |

### Benefícios

- Bugs são **detectados mais perto da causa**, facilitando diagnóstico.
- Erros não propagam silenciosamente pelo sistema.
- Mensagens de erro são mais **claras e contextuais**.

### Trade-offs

- Em sistemas distribuídos, fail fast pode causar cascata de falhas → use circuit breakers.
- Nem toda falha deve derrubar o sistema → graceful degradation quando possível.

---

## Separation of Concerns

### Princípio

> *"Cada módulo/componente deve ser responsável por um aspecto do sistema."*

### Dimensões de separação

| Dimensão | Exemplo |
|----------|---------|
| **Funcional** | Pagamento vs Estoque vs Notificação |
| **Técnica** | Apresentação vs Lógica vs Persistência |
| **Cross-cutting** | Logging, Segurança, Transações (separar com AOP, middlewares, decorators) |
| **Temporal** | Build time vs Deploy time vs Runtime |

### Mecanismos

| Mecanismo | Quando usar |
|-----------|-------------|
| **Funções/Métodos** | Separar operações dentro de uma classe |
| **Classes** | Separar responsabilidades |
| **Módulos/Packages** | Separar domínios/funcionalidades |
| **Serviços** | Separar bounded contexts |
| **Layers** | Separar preocupações técnicas |

---

## Program to an Interface, Not an Implementation

### Princípio

> *"Variáveis devem ser declaradas como tipos abstratos (interfaces),
> não como tipos concretos."*

### Benefícios

- **Substituibilidade:** trocar implementação sem alterar consumidores.
- **Testabilidade:** injetar mocks/stubs facilmente.
- **Desacoplamento:** consumidores não conhecem detalhes de implementação.

### Quando usar implementações diretamente

- Quando não há variação (ex: `String`, `List`, objetos de valor simples).
- Em código de infraestrutura mais interno.
- Quando a abstração seria artificial e sem valor.

---

## Princípio da Menor Surpresa

### Princípio

> *"Um componente deve se comportar como a maioria dos usuários espera.
> O comportamento não deve surpreender."*

### Aplicação

| Área | Exemplo |
|------|---------|
| **Nomes** | `calculate()` não deve salvar no banco |
| **Retornos** | `find()` retorna resultado ou null/empty — não lança exceção |
| **Side effects** | Getters não devem modificar estado |
| **Convenções** | Siga os padrões da linguagem/framework |
| **APIs** | Use HTTP verbs com semântica correta (GET não altera dados) |

---

## Encapsulamento

### Princípio

> *"Esconda os detalhes internos e exponha apenas o que é necessário."*

### Níveis de encapsulamento

| Nível | Descrição |
|-------|-----------|
| **Dados** | Atributos são privados; acesso via métodos com comportamento |
| **Implementação** | O "como" está escondido; expõe apenas o "o quê" |
| **Tipo** | Consumidores trabalham com interfaces, não classes concretas |
| **Design** | Mudanças internas não propagam para consumidores |

### Regra de ouro

- **Minimize a superfície pública:** tudo que é público é um contrato.
- **Maximize a privacidade:** máxima restrição de acesso por padrão.
- **Exponha comportamento, não dados:** ofereça operações, não getters/setters.

---

## Coesão e Acoplamento

### Definições

| Conceito | Definição | Meta |
|----------|-----------|------|
| **Coesão** | Grau em que elementos de um módulo estão **relacionados entre si** | ALTA |
| **Acoplamento** | Grau de **dependência** entre módulos | BAIXO |

### Tipos de coesão (do pior para o melhor)

| Tipo | Descrição |
|------|-----------|
| **Coincidental** | Elementos sem relação (pior) |
| **Logical** | Elementos relacionados logicamente, mas não funcionalmente |
| **Temporal** | Elementos executados no mesmo tempo |
| **Procedural** | Elementos na mesma sequência de execução |
| **Communicational** | Elementos operam nos mesmos dados |
| **Sequential** | Output de um é input do próximo |
| **Functional** | Todos contribuem para uma única função bem definida (melhor) |

### Tipos de acoplamento (do pior para o melhor)

| Tipo | Descrição |
|------|-----------|
| **Content** | Módulo acessa internals de outro (pior) |
| **Common** | Compartilham dados globais |
| **Control** | Um envia flag que controla o fluxo do outro |
| **Stamp** | Compartilham estrutura de dados (mas usam partes diferentes) |
| **Data** | Comunicam-se por parâmetros simples (melhor) |
| **Message** | Comunicam-se por mensagens/eventos sem dependência direta (melhor ainda) |

---

## Imutabilidade

### Princípio

> *"Prefira objetos que não mudam de estado após a criação."*

### Benefícios

| Benefício | Descrição |
|-----------|-----------|
| **Thread safety** | Objetos imutáveis são seguros por definição |
| **Previsibilidade** | O estado não muda — sem surpresas |
| **Cache** | Imutáveis podem ser cacheados com segurança |
| **Debugging** | Mais fácil rastrear quando o estado nunca muda |
| **Hash keys** | Podem ser usados como chaves de mapa com segurança |

### Onde aplicar

| Contexto | Imutável? |
|----------|-----------|
| **Value Objects** (DDD) | Sim, sempre |
| **DTOs** | Sim, preferencialmente |
| **Configurações** | Sim (carregadas uma vez) |
| **Entidades** (DDD) | Parcialmente (identidade imutável, estado mutável controlado) |
| **Collections** | Prefira versões imutáveis (copie ao modificar) |

---

## Defensive Programming

### Princípio

> *"Assuma que qualquer coisa que pode dar errado, vai dar errado."*

### Técnicas

| Técnica | Descrição |
|---------|-----------|
| **Validação de input** | Nunca confie em dados externos |
| **Null checks** | Use Optional/Maybe ou retorne vazios em vez de null |
| **Assertions** | Verifique invariantes em pontos críticos |
| **Limites** | Defina e valide limites (tamanho, range, timeout) |
| **Error handling** | Trate erros explicitamente; nunca engula exceções silenciosamente |
| **Imutabilidade** | Previna modificações inesperadas |
| **Copies defensivas** | Copie dados recebidos e retornados quando necessário |

### Não confunda com paranoia

- Valide nas **fronteiras** (input do usuário, APIs externas, mensagens).
- **Dentro** do sistema, contratos entre módulos são suficientes (asserts em dev, logs em prod).
- Excesso de defensividade gera código ilegível e lento.

---

## Anti-Patterns Comuns

| Anti-pattern | Descrição | Solução |
|-------------|-----------|---------|
| **God Object** | Classe que sabe/faz tudo | Dividir em classes coesas (SRP) |
| **Spaghetti Code** | Código sem estrutura, fluxo entrelaçado | Extrair funções/classes, aplicar padrões |
| **Golden Hammer** | Usar a mesma solução/ferramenta para tudo | Escolher a ferramenta certa para cada problema |
| **Premature Optimization** | Otimizar sem medir e antes de funcionar | Faça funcionar → Faça correto → Faça rápido |
| **Cargo Cult** | Copiar padrões sem entender o porquê | Entenda o problema antes de aplicar o padrão |
| **Lava Flow** | Código morto/legado nunca removido | Remova código que não é usado |
| **Big Ball of Mud** | Sistema sem estrutura aparente | Refatorar com boundaries claros |
| **Primitive Obsession** | Usar tipos primitivos em vez de Value Objects | Crie tipos ricos para conceitos de domínio |
| **Feature Envy** | Método que usa mais dados de outra classe que da própria | Mova o método para a classe correta |
| **Shotgun Surgery** | Uma mudança requer edições em muitas classes | Consolidar a responsabilidade |

---

## Checklist de Code Review

Ao revisar ou criar código, verifique:

### Design

- [ ] Responsabilidade única clara?
- [ ] Depende de abstrações (interfaces), não de implementações?
- [ ] Favorece composição sobre herança?
- [ ] Encapsulamento adequado? (superfície pública mínima)
- [ ] Sem acoplamento desnecessário entre módulos?

### Simplicidade

- [ ] Solução mais simples que resolve o problema?
- [ ] Sem generalizações especulativas (YAGNI)?
- [ ] Nomes claros e autoexplicativos?
- [ ] Sem código morto ou comentários obsoletos?

### Robustez

- [ ] Inputs validados nas fronteiras?
- [ ] Erros tratados explicitamente (não engolidos)?
- [ ] Fail fast em condições inválidas?
- [ ] Sem null/nil retornado quando uma coleção vazia serve?

### Testabilidade

- [ ] Dependências são injetáveis (não hardcoded)?
- [ ] Efeitos colaterais são isoláveis?
- [ ] Estado global é evitado?
- [ ] Comportamento é verificável via testes unitários?
