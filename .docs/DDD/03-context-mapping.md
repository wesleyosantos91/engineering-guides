# Domain-Driven Design — Context Mapping

> **Objetivo deste documento:** Servir como referência teórica completa sobre **Context Mapping em DDD**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Abordagem **agnóstica de linguagem** — foca em conceitos, motivações, heurísticas, boas práticas e **diagramas ilustrativos** sem vínculo a tecnologia específica.

---

## Quick Reference — Cheat Sheet

| Padrão | Relação de poder | Quando usar | Risco |
|--------|-----------------|-------------|-------|
| **Shared Kernel** | Simétrico (parceria) | Dois contextos com forte overlap controlado | Acoplamento mútuo |
| **Customer-Supplier** | Assimétrico (upstream/downstream) | Supplier entrega o que Customer precisa | Dependência do supplier |
| **Conformist** | Assimétrico (downstream se adapta) | Sem poder de influenciar o upstream | Modelo importado sem controle |
| **Anti-Corruption Layer** | Proteção ativa | Integrar com legado ou contexto externo | Custo de manutenção da camada |
| **Open Host Service** | Upstream serve muitos downstreams | API pública com protocolo bem definido | Rigidez na evolução |
| **Published Language** | Formato compartilhado | Padrão de dados aberto (JSON Schema, Protobuf) | Governança do schema |
| **Separate Ways** | Sem relação | Contextos não precisam se integrar | Duplicação possível |
| **Partnership** | Simétrico (cooperação ativa) | Dois times que coordenam juntos | Exige comunicação constante |
| **Big Ball of Mud** | Caótico | Reconhecer sistemas sem fronteiras claras | Não é um alvo; é um diagnóstico |

---

## Sumário

- [Domain-Driven Design — Context Mapping](#domain-driven-design--context-mapping)
  - [Quick Reference — Cheat Sheet](#quick-reference--cheat-sheet)
  - [Sumário](#sumário)
  - [O que é Context Mapping?](#o-que-é-context-mapping)
  - [Por que mapear contextos?](#por-que-mapear-contextos)
  - [Padrões de Relacionamento](#padrões-de-relacionamento)
    - [Partnership](#partnership)
    - [Shared Kernel](#shared-kernel)
    - [Customer-Supplier](#customer-supplier)
    - [Conformist](#conformist)
    - [Anti-Corruption Layer (ACL)](#anti-corruption-layer-acl)
    - [Open Host Service (OHS)](#open-host-service-ohs)
    - [Published Language (PL)](#published-language-pl)
    - [Separate Ways](#separate-ways)
    - [Big Ball of Mud](#big-ball-of-mud)
  - [Como criar um Context Map](#como-criar-um-context-map)
    - [Passo a passo](#passo-a-passo)
    - [Template de documentação](#template-de-documentação)
  - [Exemplo completo: e-commerce](#exemplo-completo-e-commerce)
    - [Detalhamento das relações](#detalhamento-das-relações)
  - [Padrões de integração técnica](#padrões-de-integração-técnica)
  - [Diretrizes para Code Review assistido por AI](#diretrizes-para-code-review-assistido-por-ai)
  - [Referências](#referências)

---

## O que é Context Mapping?

**Context Mapping** é a prática de documentar e visualizar as **relações entre Bounded Contexts**. Um **Context Map** mostra:

- Quais contextos existem no sistema
- Como eles se relacionam (quem fornece, quem consome)
- Qual é a **dinâmica de poder** entre eles (quem influencia quem)
- Quais padrões de integração são usados

> *"A Context Map is not a diagram of the architecture. It's a diagram of the political landscape of the project."*
> — Eric Evans

---

## Por que mapear contextos?

| Benefício | Descrição |
|----------|-----------|
| **Visibilidade** | Torna explícitas dependências que são frequentemente implícitas |
| **Comunicação** | Facilita discussão sobre integrações entre equipes |
| **Decisão informada** | Ajuda a escolher o padrão de integração correto |
| **Prevenção de acoplamento** | Identifica pontos de acoplamento perigoso antes que causem problemas |
| **Planejamento** | Informa decisões de arquitetura, deploy e evolução |

Sem um Context Map, equipes frequentemente:
- Criam dependências ocultas entre módulos
- Compartilham bancos de dados sem saber
- Duplicam modelos sem consciência
- Quebram módulos de outras equipes acidentalmente

---

## Padrões de Relacionamento

### Partnership

**Dois times que cooperam ativamente**, sincronizando mudanças em seus modelos. Nenhum dos dois é "server" do outro.

```
┌──────────────┐     Partnership     ┌──────────────┐
│  Context A   │◄══════════════════►│  Context B   │
│  (Time Alpha)│                     │  (Time Beta) │
└──────────────┘                     └──────────────┘
```

| Aspecto | Descrição |
|---------|-----------|
| **Quando usar** | Dois times trabalham juntos, com comunicação frequente |
| **Dinâmica** | Mudanças são coordenadas; ambos cedem quando necessário |
| **Risco** | Exige overhead de comunicação; escalabilidade limitada |
| **Exemplo** | Time de Orders e time de Payments em startup pequena |

**Boas práticas:**
- Definir cadência de sincronização (ex: reunião semanal)
- Testes de integração compartilhados
- CI/CD que valida ambos os contextos

---

### Shared Kernel

Dois contextos **compartilham um subconjunto** pequeno e explícito do modelo. Qualquer mudança no kernel afeta ambos.

```
┌──────────────┐                     ┌──────────────┐
│  Context A   │    ┌──────────┐     │  Context B   │
│              │───►│  Shared  │◄────│              │
│              │    │  Kernel  │     │              │
└──────────────┘    └──────────┘     └──────────────┘
```

| Aspecto | Descrição |
|---------|-----------|
| **Quando usar** | Overlap genuíno entre contextos; custo de duplicação é alto |
| **Dinâmica** | Mudanças no kernel requerem acordo de ambos os times |
| **Risco** | Acoplamento; uma mudança pode quebrar ambos |
| **Exemplo** | Shared `Money` value object, `UserId` type |

**Boas práticas:**
- Manter o kernel o **menor possível**
- Governança clara (quem pode alterar, como)
- Testes compartilhados que validam o kernel
- Versionamento semântico
- **Se o kernel cresce demais** → sinal de que os contextos devem ser fundidos ou o kernel deve ser extraído para um contexto próprio

---

### Customer-Supplier

Relação **upstream-downstream** onde o upstream (supplier) fornece dados/serviços e o downstream (customer) consome.

```
┌──────────────┐                     ┌──────────────┐
│  Upstream    │ ════════════════►   │  Downstream  │
│  (Supplier)  │                     │  (Customer)  │
└──────────────┘                     └──────────────┘
```

| Aspecto | Descrição |
|---------|-----------|
| **Quando usar** | Um contexto depende de dados/funcionalidades de outro |
| **Dinâmica** | Customer expressa necessidades; Supplier prioriza atendê-las |
| **Risco** | Supplier pode não priorizar necessidades do Customer |
| **Exemplo** | Catalog (supplier) → Sales (customer) |

**Boas práticas:**
- Customer define o que precisa via **testes de contrato** (consumer-driven contracts)
- Supplier não muda API sem notificar Customers
- Definir SLAs e tempos de resposta para mudanças
- Versioning da API com deprecation policy

---

### Conformist

O downstream **aceita o modelo do upstream sem adaptação**. Cede completamente ao modelo alheio.

```
┌──────────────┐                     ┌──────────────┐
│  Upstream    │ ════════════════►   │  Downstream  │
│  (Dictator)  │     Conformist      │  (Conforma)  │
└──────────────┘                     └──────────────┘
```

| Aspecto | Descrição |
|---------|-----------|
| **Quando usar** | O upstream não vai mudar por você (ex: API de terceiro, legado intocável) |
| **Dinâmica** | Downstream importa modelo do upstream diretamente |
| **Risco** | Modelo externo contamina seu domínio; mudanças no upstream quebram você |
| **Exemplo** | Consumir API de pagamentos de terceiros usando DTOs deles diretamente |

**Boas práticas:**
- **Evite** este padrão quando possível — prefira ACL
- Use somente quando: (a) o custo de tradução não compensa, ou (b) o modelo externo é adequado
- Documente claramente que o modelo é importado e não controlado

---

### Anti-Corruption Layer (ACL)

Uma **camada de tradução** que protege seu modelo de domínio contra modelos externos, legados ou corruptos.

```
┌──────────────┐     ┌─────────────┐     ┌──────────────┐
│  External /  │────►│    ACL      │────►│  My Domain   │
│  Legacy      │     │ (Translate) │     │  (Protected) │
└──────────────┘     └─────────────┘     └──────────────┘
```

| Aspecto | Descrição |
|---------|-----------|
| **Quando usar** | Integrar com sistema legado, API externa, ou contexto com modelo "sujo" |
| **Dinâmica** | ACL traduz entre o modelo externo e o modelo interno |
| **Risco** | Custo de manutenção da camada de tradução |
| **Exemplo** | Integrar com ERP legado, APIs de terceiros, sistemas SOAP |

**Boas práticas:**

1. **Traduza na fronteira** — ACL converte DTOs/schemas externos em objetos do seu domínio:

```
// ACL traduz modelo externo → modelo interno
class LegacyPaymentACL implements PaymentGateway:
    legacyClient: LegacyPaymentClient

    processPayment(order: Order): PaymentResult
        // Traduz do SEU domínio → formato do legado
        legacyRequest = LegacyPaymentRequest(
            orderRef = order.id.toString(),
            amt = order.totalAmount().toCents(),
            ccy = order.totalAmount().currency.isoCode()
        )

        // Chama o legado
        legacyResponse = legacyClient.pay(legacyRequest)

        // Traduz do legado → SEU domínio
        if legacyResponse.code == "00":
            return PaymentResult.success(
                transactionId = TransactionId(legacyResponse.txnRef))
        else:
            return PaymentResult.failed(
                reason = translateErrorCode(legacyResponse.code))
```

2. **Interface no domínio** — A interface que a ACL implementa pertence à camada de domínio.

3. **ACL é descartável** — Quando o sistema legado for substituído, troque a implementação da ACL sem afetar o domínio.

4. **Teste a tradução** — Testes unitários para a ACL garantem que a tradução está correta.

---

### Open Host Service (OHS)

O upstream define um **protocolo/API aberta** que múltiplos downstreams podem consumir.

```
                                ┌──────────────┐
                       ┌───────│  Consumer A   │
┌──────────────┐       │       └──────────────┘
│  Open Host   │───────┤       ┌──────────────┐
│  Service     │───────┼───────│  Consumer B   │
│  (API Pública)│──────┤       └──────────────┘
└──────────────┘       │       ┌──────────────┐
                       └───────│  Consumer C   │
                               └──────────────┘
```

| Aspecto | Descrição |
|---------|-----------|
| **Quando usar** | Contexto serve múltiplos consumidores com contrato padronizado |
| **Dinâmica** | API pública estável; consumidores se adaptam |
| **Risco** | Evolução da API é lenta (breaking changes afetam muitos) |
| **Exemplo** | Serviço de autenticação consumido por todos os outros contextos |

**Boas práticas:**
- Versionamento de API (path ou header)
- Documentação (OpenAPI/AsyncAPI)
- Backward compatibility
- Deprecation policy com timeline
- Testes de contrato

---

### Published Language (PL)

Um **formato de dados padronizado** usado para comunicação entre contextos. Frequentemente combinado com OHS.

| Aspecto | Descrição |
|---------|-----------|
| **Quando usar** | Múltiplos contextos precisam trocar dados em formato comum |
| **Formatos comuns** | JSON Schema, Protocol Buffers, Avro, XML Schema, AsyncAPI |
| **Exemplo** | Schema Registry para eventos Kafka; OpenAPI spec para REST |

**Boas práticas:**
- Schema versionado e registrado em Schema Registry
- Evolução compatível (adicionar campos opcionais, não remover)
- Validação automática contra schema
- Documentação gerada a partir do schema

---

### Separate Ways

Dois contextos **não se integram**. Cada um resolve o problema independentemente.

| Aspecto | Descrição |
|---------|-----------|
| **Quando usar** | Custo de integração é maior que o custo de duplicação |
| **Dinâmica** | Sem dependência; total autonomia |
| **Risco** | Duplicação de dados/lógica; pode divergir |
| **Exemplo** | Cada contexto mantém seu próprio catálogo de produtos simplificado |

**Boas práticas:**
- Documente a decisão e o motivo
- Revise periodicamente se a integração passou a valer a pena
- Aceite a duplicação consciente

---

### Big Ball of Mud

Um sistema (ou parte dele) **sem fronteiras claras** nem modelo explícito. NÃO é um padrão desejável — é um diagnóstico.

| Aspecto | Descrição |
|---------|-----------|
| **Quando reconhecer** | Sistema legado emaranhado, sem separação de concerns |
| **Dinâmica** | Tudo acessa tudo; não há modelo coerente |
| **Ação** | Isolar com ACL; extrair contextos progressivamente |

**Estratégia de migração:**
1. **Identifique seams** — pontos de divisão natural
2. **ACL na fronteira** — proteja novos contextos do mud
3. **Strangler Fig** — substitua pedaços progressivamente
4. **Bubble Context** — crie "bolhas" de DDD limpo dentro do legado

---

## Como criar um Context Map

### Passo a passo

1. **Liste todos os Bounded Contexts** do sistema
2. **Identifique integrações** — quais contextos se comunicam?
3. **Determine a direção** — quem é upstream, quem é downstream?
4. **Classifique a relação** — qual padrão se aplica?
5. **Documente** — diagrama + descrição textual
6. **Revise periodicamente** — o mapa evolui com o sistema

### Template de documentação

Para cada relação, documente:

```
## [Context A] ←→ [Context B]

- **Padrão:** Customer-Supplier
- **Upstream:** Context A (fornece dados de produto)
- **Downstream:** Context B (consome catálogo)
- **Mecanismo:** REST API com OHS + Published Language (OpenAPI)
- **ACL:** Context B implementa ProductCatalogACL
- **Dados trocados:** ProductId, Name, Price, Availability
- **SLA:** Latência < 200ms, disponibilidade 99.9%
- **Owner:** Team Alpha (upstream), Team Beta (downstream)
- **Observações:** Contract tests executados no CI de ambos
```

---

## Exemplo completo: e-commerce

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CONTEXT MAP                                  │
│                                                                     │
│                    ┌──────────────┐                                  │
│           ┌──────►│   CATALOG    │◄──────┐                          │
│           │  OHS  │  (Core)      │  ACL  │                          │
│           │  +PL  └──────┬───────┘       │                          │
│           │              │               │                          │
│    ┌──────┴──────┐       │ C/S    ┌──────┴──────┐                   │
│    │   SEARCH    │       │        │  INVENTORY  │                   │
│    │ (Supporting)│       │        │ (Supporting) │                  │
│    └─────────────┘       │        └──────┬──────┘                   │
│                          │               │                          │
│                   ┌──────▼───────┐       │ Domain Events            │
│                   │    SALES     │◄──────┘                          │
│                   │   (Core)     │                                   │
│                   └──────┬───────┘                                   │
│                          │                                          │
│              ┌───────────┼───────────┐                              │
│              │           │           │                              │
│        Domain Events  Domain Events  Domain Events                  │
│              │           │           │                              │
│       ┌──────▼──┐  ┌─────▼────┐  ┌──▼───────────┐                  │
│       │SHIPPING │  │ BILLING  │  │ NOTIFICATION │                  │
│       │(Support)│  │(Support) │  │  (Generic)   │                  │
│       └─────────┘  └─────┬────┘  └──────────────┘                  │
│                          │                                          │
│                     ACL  │                                          │
│                          │                                          │
│                   ┌──────▼───────┐                                   │
│                   │   PAYMENT    │   ◄── Conformist / ACL com       │
│                   │  (Generic)   │       gateway externo            │
│                   │  [External]  │                                   │
│                   └──────────────┘                                   │
│                                                                     │
│  Legenda:                                                          │
│  ═══ = Shared Kernel    ──► = Customer/Supplier                    │
│  C/S = Customer/Supplier  OHS = Open Host Service                  │
│  ACL = Anti-Corruption Layer  PL = Published Language              │
└─────────────────────────────────────────────────────────────────────┘
```

### Detalhamento das relações

| Upstream | Downstream | Padrão | Mecanismo |
|----------|-----------|--------|-----------|
| Catalog | Sales | Customer-Supplier | REST API (OHS + PL) |
| Catalog | Search | OHS + Published Language | Event stream (produto criado/atualizado) |
| Catalog | Inventory | ACL | Events + query API |
| Sales | Shipping | Domain Events | Async messaging |
| Sales | Billing | Domain Events | Async messaging |
| Sales | Notification | Domain Events | Async messaging |
| Billing | Payment (ext.) | ACL / Conformist | REST API do gateway externo |
| Inventory | Sales | Domain Events | Async (estoque reservado/liberado) |

---

## Padrões de integração técnica

| Padrão DDD | Implementação técnica comum |
|-----------|---------------------------|
| **OHS + PL** | REST API com OpenAPI spec / gRPC com Protobuf |
| **Domain Events** | Message broker (Kafka, RabbitMQ, SQS/SNS) |
| **ACL** | Adapter/Translator class na fronteira |
| **Shared Kernel** | Biblioteca compartilhada (package/module) com versionamento semântico |
| **Customer-Supplier** | Consumer-Driven Contract Tests (Pact, Spring Cloud Contract) |
| **Conformist** | Client library do fornecedor usado diretamente |

---

## Diretrizes para Code Review assistido por AI

> Regras acionáveis para identificar violações de Context Mapping.

| # | Regra de detecção | Padrão | Ação sugerida |
|---|-------------------|--------|---------------|
| 1 | Módulo importa diretamente classes/tipos de outro módulo de domínio | Bounded Context | Criar interface ou ACL na fronteira |
| 2 | Dois módulos acessam as mesmas tabelas do banco | Shared Database (anti-pattern) | Cada contexto com storage próprio |
| 3 | DTOs de API externa usados diretamente na camada de domínio | Conformist indesejado | Implementar ACL para traduzir |
| 4 | Chamada síncrona direta entre módulos para manter consistência | Cross-context transaction | Usar Domain Events e eventual consistency |
| 5 | Modelo compartilhado grande e crescente entre módulos | Shared Kernel descontrolado | Minimizar kernel ou extrair para contexto próprio |
| 6 | Ausência de Contract Tests entre producer e consumer | Customer-Supplier frágil | Implementar consumer-driven contract tests |
| 7 | Ausência de Context Map documentado | Context Mapping | Criar e manter documentação de relações |

---

## Referências

| Recurso | Autor | Tipo |
|---------|-------|------|
| *Domain-Driven Design: Tackling Complexity in the Heart of Software* (Ch. 14) | Eric Evans | Livro |
| *Implementing Domain-Driven Design* (Ch. 3, 13) | Vaughn Vernon | Livro |
| *Learning Domain-Driven Design* (Ch. 4) | Vlad Khononov | Livro |
| *Domain-Driven Design Distilled* (Ch. 4) | Vaughn Vernon | Livro |
| *Context Mapping* | Alberto Brandolini | Workshop/Talk |
| *Team Topologies* | Skelton & Pais | Livro |
