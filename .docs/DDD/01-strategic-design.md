# Domain-Driven Design — Strategic Design

> **Objetivo deste documento:** Servir como referência teórica completa sobre **Design Estratégico em DDD**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Abordagem **agnóstica de linguagem** — foca em conceitos, motivações, heurísticas, boas práticas e **pseudocódigo ilustrativo** sem vínculo a tecnologia específica.

---

## Quick Reference — Cheat Sheet

| Conceito | Regra de ouro | Violação típica | Correção |
|----------|--------------|------------------|----------|
| **Ubiquitous Language** | Código fala a mesma língua do negócio | Termos técnicos substituem termos do domínio | Alinhar nomenclatura com domain experts |
| **Bounded Context** | Um modelo por contexto, sem ambiguidade | Modelo único monolítico compartilhado por tudo | Dividir em contextos com fronteiras claras |
| **Context Map** | Relações entre contextos são explícitas | Dependências implícitas entre módulos | Mapear e documentar integrações |
| **Subdomain** | Identificar core, supporting e generic | Tratar tudo como igualmente importante | Priorizar investimento no core domain |

---

## Sumário

- [Domain-Driven Design — Strategic Design](#domain-driven-design--strategic-design)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que é DDD?](#o-que-é-ddd)
  - [Por que DDD importa?](#por-que-ddd-importa)
  - [Ubiquitous Language](#ubiquitous-language)
    - [Definição](#definição)
    - [Motivação](#motivação)
    - [Boas Práticas](#boas-práticas)
    - [Heurísticas para identificar violações](#heurísticas-para-identificar-violações)
    - [Anti-patterns](#anti-patterns)
  - [Subdomains — Classificação do Domínio](#subdomains--classificação-do-domínio)
    - [Core Domain](#core-domain)
    - [Supporting Subdomain](#supporting-subdomain)
    - [Generic Subdomain](#generic-subdomain)
    - [Como identificar cada tipo](#como-identificar-cada-tipo)
    - [Decisão de investimento](#decisão-de-investimento)
  - [Bounded Context](#bounded-context)
    - [Definição](#definição-1)
    - [Motivação](#motivação-1)
    - [Relação com Ubiquitous Language](#relação-com-ubiquitous-language)
    - [Boas Práticas](#boas-práticas-1)
    - [Heurísticas para definir fronteiras](#heurísticas-para-definir-fronteiras)
    - [Anti-patterns](#anti-patterns-1)
    - [Exemplo: e-commerce](#exemplo-e-commerce)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que é DDD?

**Domain-Driven Design (DDD)** é uma abordagem de design de software proposta por **Eric Evans** no livro *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003). O DDD coloca o **domínio do negócio** no centro de todas as decisões de design e arquitetura.

DDD divide-se em dois pilares:

| Pilar | Foco | Elementos |
|-------|------|-----------|
| **Design Estratégico** | Macro-arquitetura e modelagem do problema | Ubiquitous Language, Bounded Contexts, Context Maps, Subdomains |
| **Design Tático** | Micro-design dentro de cada contexto | Entities, Value Objects, Aggregates, Domain Events, Repositories, Services, Factories |

> **Princípio fundamental:** A complexidade primária do software está no **domínio do negócio**, não na tecnologia. O código deve ser um reflexo fiel do modelo mental dos especialistas do domínio.

---

## Por que DDD importa?

Software que não reflete o domínio do negócio deteriora rapidamente:

| Sintoma | Descrição |
|---------|-----------|
| **Translation Gap** | Desenvolvedores usam termos diferentes dos especialistas de negócio, gerando mal-entendidos |
| **Big Ball of Mud** | Ausência de fronteiras claras resulta em código emaranhado e acoplado |
| **Anemic Domain Model** | Lógica de negócio espalhada em serviços procedurais, entities são meros DTOs |
| **Knowledge Crunching Failure** | Regras de negócio críticas são perdidas ou distorcidas na tradução para código |
| **Modelo Monolítico** | Um único modelo tenta representar tudo, gerando ambiguidade e conflito |

DDD ataca esses sintomas ao fornecer:
- **Linguagem compartilhada** entre negócio e tecnologia
- **Fronteiras explícitas** entre modelos diferentes
- **Foco no core domain** — investir onde há maior valor competitivo
- **Tactical patterns** que protegem invariantes e expressam regras de negócio

---

## Ubiquitous Language

### Definição

> *"Uma linguagem estruturada em torno do modelo de domínio, usada por todos os membros da equipe para conectar todas as atividades do software ao domínio."*
> — Eric Evans

A **Ubiquitous Language** é o vocabulário compartilhado entre desenvolvedores, especialistas de domínio e stakeholders. Ela deve estar presente em:

- Conversas e discussões
- Documentação
- **Código-fonte** (nomes de classes, métodos, variáveis, módulos)
- Testes automatizados
- APIs e contratos

### Motivação

Quando o código usa termos diferentes do negócio:
- Desenvolvedores interpretam regras incorretamente
- Especialistas de domínio não conseguem validar o modelo
- Novos membros da equipe sofrem para entender a relação código ↔ negócio
- Bugs semânticos (o código "funciona" mas faz a coisa errada) proliferam

### Boas Práticas

1. **O código É o modelo** — Nomes de classes, métodos e variáveis devem refletir exatamente os termos usados pelos especialistas de domínio.

2. **Glossário vivo** — Mantenha um glossário de termos do domínio atualizado e acessível. Atualize-o à medida que o entendimento evolui.

3. **Rejeite sinônimos** — Cada conceito tem **um** nome. Se o negócio diz "Pedido", o código diz `Order`, não `Request`, `Purchase`, `Transaction`.

4. **Termos técnicos ≠ termos de domínio** — Palavras como `Manager`, `Handler`, `Processor`, `Data` são termos técnicos, não de domínio. Evite-os na camada de domínio.

5. **Evolua a linguagem** — A linguagem muda à medida que o entendimento do domínio aprofunda. **Renomeie** sem medo quando descobrir que um termo é impreciso.

6. **Teste com a linguagem** — Testes devem ler como especificações de negócio:

```
// BOM — linguagem do domínio
test "pedido com mais de 3 itens recebe desconto de volume"

// RUIM — linguagem técnica
test "testOrderDiscountMethod returns true when list size > 3"
```

7. **Desafie ambiguidades** — Se um termo pode significar coisas diferentes em contextos diferentes, é sinal de que existem **Bounded Contexts** distintos.

### Heurísticas para identificar violações

| Sinal | O que indica |
|-------|-------------|
| Desenvolvedores usam termos que especialistas de domínio não reconhecem | Gap de linguagem |
| O mesmo conceito tem nomes diferentes em partes do código | Falta de padronização |
| Especialistas de domínio não entendem nomes de classes/métodos | Código não reflete o domínio |
| Reuniões exigem "tradução" entre negócio e dev | Linguagem não é ubíqua |
| Comentários no código explicam "o que isso significa no negócio" | Nome não é autoexplicativo |

### Anti-patterns

| Anti-pattern | Descrição | Correção |
|-------------|-----------|----------|
| **Tech-first naming** | Classes chamadas `OrderProcessor`, `DataManager`, `UserHandler` | Renomear para termos do domínio: `OrderFulfillment`, `Catalog`, `AccountRegistration` |
| **Synonym soup** | `Customer`, `Client`, `User`, `Account` usados intercambiavelmente | Definir um termo canônico por contexto |
| **Abbreviation overload** | `OrdSvc`, `PmtMgr`, `CustRepo` | Usar nomes completos e legíveis |
| **Database-driven naming** | Classes refletem tabelas (`tbl_order`, `order_record`) | Nomear a partir do domínio, não do schema |

---

## Subdomains — Classificação do Domínio

Todo sistema de software opera em um **domínio** — o espaço de conhecimento e atividade ao qual o software se aplica. Eric Evans classifica o domínio em três tipos de **subdomains**:

### Core Domain

O **Core Domain** é o que torna o negócio **único e competitivo**. É onde a maior complexidade de negócio reside e onde o investimento em modelagem rica deve ser máximo.

| Característica | Descrição |
|---------------|-----------|
| **Diferencial competitivo** | É o que o negócio faz melhor que a concorrência |
| **Complexidade alta** | Regras de negócio sofisticadas, muitos edge cases |
| **Mudança frequente** | Evolui constantemente com o negócio |
| **Investimento máximo** | Melhores desenvolvedores, modelagem mais rica |

**Exemplos:**
- Para um e-commerce: motor de pricing/promoções, recomendação de produtos
- Para um banco: engine de análise de crédito, gestão de risco
- Para uma logística: otimização de rotas, planejamento de entregas

### Supporting Subdomain

Subdomínios que **suportam** o core domain mas não são o diferencial competitivo. São específicos da empresa, mas não são a razão de ser do negócio.

| Característica | Descrição |
|---------------|-----------|
| **Necessário mas não diferencial** | O negócio precisa, mas não é o que o torna único |
| **Complexidade moderada** | Regras específicas, mas menos sofisticadas |
| **Mudança moderada** | Evolui, mas menos frequentemente |
| **Investimento moderado** | Modelagem adequada, sem over-engineering |

**Exemplos:**
- Para um e-commerce: gestão de estoque, cadastro de produtos
- Para um banco: gestão de contas correntes, extratos
- Para uma logística: gestão de frota, cadastro de motoristas

### Generic Subdomain

Subdomínios que são **comuns a muitos negócios** e não oferecem diferencial competitivo. São candidatos naturais a soluções prontas (bibliotecas, SaaS, frameworks).

| Característica | Descrição |
|---------------|-----------|
| **Genérico, não diferencial** | Qualquer empresa precisa, não é exclusivo do negócio |
| **Complexidade variável** | Pode ser complexo (autenticação), mas é resolvido |
| **Mudança rara** | Relativamente estável |
| **Investimento mínimo** | Preferir soluções prontas (buy over build) |

**Exemplos:**
- Autenticação/autorização (use OAuth2/OIDC providers)
- Envio de e-mails/notificações (use serviços externos)
- Geração de PDFs (use bibliotecas prontas)
- Pagamentos (use gateways de pagamento)

### Como identificar cada tipo

| Pergunta | Core | Supporting | Generic |
|----------|------|-----------|---------|
| "Isso nos diferencia da concorrência?" | Sim | Não | Não |
| "Precisamos construir do zero?" | Sim | Talvez | Não |
| "Compramos pronto no mercado?" | Não | Talvez | Sim |
| "Melhores devs devem trabalhar aqui?" | Sim | Depende | Não |
| "Complexidade de regras de negócio?" | Alta | Média | Baixa/resolvida |

### Decisão de investimento

```
[Identificar Subdomain]
    │
    ├── É diferencial competitivo?
    │   ├── SIM → Core Domain → Investimento MÁXIMO
    │   │                        • DDD tático completo
    │   │                        • Modelagem rica
    │   │                        • Melhores devs
    │   │                        • Testes extensivos
    │   │
    │   └── NÃO → É específico do negócio?
    │       ├── SIM → Supporting → Investimento MODERADO
    │       │                      • Modelagem adequada
    │       │                      • Pode simplificar patterns
    │       │                      • CRUD quando apropriado
    │       │
    │       └── NÃO → Generic → Investimento MÍNIMO
    │                            • Buy over build
    │                            • Bibliotecas/SaaS
    │                            • Integração simples
```

> **Decisão rápida:** Antes de aplicar DDD tático completo (Aggregates, Domain Events, etc.), pergunte: "Este subdomínio é Core?". Se não for, simplifique.

---

## Bounded Context

### Definição

> *"Um Bounded Context é a fronteira explícita dentro da qual um modelo de domínio particular é definido e aplicável."*
> — Eric Evans

Um **Bounded Context** é o **limite** dentro do qual:
- Um modelo de domínio específico é válido
- A Ubiquitous Language é consistente e sem ambiguidade
- Termos têm um **único significado**

### Motivação

Um modelo único para todo o sistema é uma ilusão que leva a:
- **Ambiguidade**: "Produto" significa uma coisa no catálogo, outra no estoque, outra na logística
- **Acoplamento**: Mudanças em um conceito propagam por todo o sistema
- **Conflito**: Equipes diferentes têm visões diferentes do mesmo conceito
- **Complexidade**: O modelo cresce sem limite e ninguém o entende completamente

### Relação com Ubiquitous Language

Cada Bounded Context tem sua **própria** Ubiquitous Language:

| Termo | Contexto: Vendas | Contexto: Logística | Contexto: Financeiro |
|-------|------------------|---------------------|----------------------|
| **Pedido** | Carrinho convertido, itens, desconto | Pacote para despacho, endereço, peso | Transação de cobrança, valor, impostos |
| **Cliente** | Perfil de compras, preferências | Endereço de entrega, disponibilidade | Status de crédito, limites, histórico |
| **Produto** | Catálogo, imagens, descrição | Dimensões, peso, fragilidade | SKU, custo, margem |

> **Regra fundamental:** O mesmo termo em contextos diferentes representa **modelos diferentes**. Não tente unificar.

### Boas Práticas

1. **Um contexto, um modelo** — Cada Bounded Context define seu próprio modelo de domínio. Não compartilhe entidades entre contextos.

2. **Fronteiras linguísticas** — Se duas equipes discordam sobre o significado de um termo, provavelmente existem dois Bounded Contexts.

3. **Autonomia** — Cada contexto deve poder evoluir, ser deployado e escalado independentemente.

4. **Comunicação via contratos** — Contextos se comunicam por interfaces públicas bem definidas (APIs, eventos, contratos), **nunca compartilhando banco de dados ou modelos internos**.

5. **Tamanho adequado** — Um Bounded Context não é necessariamente um microservice. Pode ser um módulo, um pacote, ou um serviço. O importante é a **fronteira semântica**.

6. **Anti-Corruption Layer** — Ao integrar com sistemas externos ou contextos legados, use uma camada tradutora para proteger seu modelo de domínio.

7. **Mapeie antes de codificar** — Use **Context Maps** para documentar relações entre contextos antes de definir integrações.

### Heurísticas para definir fronteiras

| Heurística | Descrição |
|-----------|-----------|
| **Linguagem diverge** | Quando o mesmo termo significa coisas diferentes para equipes diferentes |
| **Equipes diferentes** | Equipes distintas geralmente representam contextos diferentes |
| **Ciclo de vida diferente** | Partes do sistema que mudam em ritmos diferentes |
| **Requisitos de escalabilidade** | Partes com necessidades de performance/escalabilidade distintas |
| **Regras de negócio isoladas** | Conjuntos de regras que não se misturam |
| **Dados que não devem vazar** | Informações sensíveis que devem ser isoladas (LGPD, PCI-DSS) |

### Anti-patterns

| Anti-pattern | Descrição | Correção |
|-------------|-----------|----------|
| **God Context** | Um único contexto engloba tudo | Identificar linguagens distintas e separar |
| **Shared Kernel sem governança** | Múltiplos contextos dependem de um modelo compartilhado sem regras claras | Definir ownership, versionamento e contratos |
| **Leaking Context** | Modelos internos de um contexto são expostos a outros | Usar ACL e Translation Layer |
| **Shared Database** | Contextos diferentes leem/escrevem nas mesmas tabelas | Cada contexto com seu próprio storage |
| **Context per entity** | Criar um contexto para cada entidade | Agrupar por coerência semântica e linguística |

### Exemplo: e-commerce

```
┌─────────────────────────────────────────────────────────┐
│                    E-COMMERCE DOMAIN                     │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   CATALOG    │  │    SALES     │  │  SHIPPING    │  │
│  │              │  │              │  │              │  │
│  │ • Product    │  │ • Order      │  │ • Shipment   │  │
│  │ • Category   │  │ • Cart       │  │ • Package    │  │
│  │ • Pricing    │  │ • Discount   │  │ • Route      │  │
│  │ • Review     │  │ • Payment    │  │ • Tracking   │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                 │           │
│         └────── Events ───┴──── Events ─────┘           │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  INVENTORY   │  │   BILLING    │  │   IDENTITY   │  │
│  │              │  │              │  │              │  │
│  │ • Stock      │  │ • Invoice    │  │ • Account    │  │
│  │ • Warehouse  │  │ • Tax        │  │ • Auth       │  │
│  │ • Allocation │  │ • Refund     │  │ • Profile    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                         │
│  Core: Sales, Catalog    Supporting: Inventory, Billing │
│  Generic: Identity                                      │
└─────────────────────────────────────────────────────────┘
```

Note como **Product** aparece em Catalog (com descrição, imagens, preço) e em Inventory (com quantidade, localização). São **modelos diferentes** do mesmo conceito real.

---

## Diretrizes para Code Review assistido por AI

> Regras acionáveis para que um assistente de código identifique violações de Design Estratégico DDD.

| # | Regra de detecção | Conceito | Ação sugerida |
|---|-------------------|----------|---------------|
| 1 | Classe/entidade de domínio usa termos técnicos (`Manager`, `Handler`, `Data`) em vez de termos de negócio | Ubiquitous Language | Renomear usando terminologia do domínio |
| 2 | Mesmo nome de classe com significados diferentes em módulos distintos, sem separação explícita | Bounded Context | Separar em contextos com modelos próprios |
| 3 | Entidade compartilhada entre múltiplos módulos/serviços por import direto | Bounded Context | Criar modelos locais com tradução na fronteira |
| 4 | Múltiplos serviços acessam as mesmas tabelas do banco de dados | Bounded Context | Cada contexto com seu próprio schema/database |
| 5 | Aplicação de DDD tático completo (Aggregates, Events) em CRUD simples | Subdomain | Avaliar se é Generic/Supporting e simplificar |
| 6 | Ausência de glossário ou inconsistência de termos no código | Ubiquitous Language | Criar e manter glossário do domínio |

---

## Referências

| Recurso | Autor | Tipo |
|---------|-------|------|
| *Domain-Driven Design: Tackling Complexity in the Heart of Software* | Eric Evans | Livro |
| *Implementing Domain-Driven Design* | Vaughn Vernon | Livro |
| *Domain-Driven Design Distilled* | Vaughn Vernon | Livro |
| *Learning Domain-Driven Design* | Vlad Khononov | Livro |
| *Patterns, Principles, and Practices of Domain-Driven Design* | Scott Millett, Nick Tune | Livro |
| *Strategic Monoliths and Microservices* | Vaughn Vernon, Tomasz Jaskula | Livro |
