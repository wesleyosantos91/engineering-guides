# Domain-Driven Design (DDD) — Guia Completo

> **Objetivo deste guia:** Servir como referência teórica completa sobre **Domain-Driven Design**, otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
> Abordagem **agnóstica de linguagem** — foca em conceitos, motivações, heurísticas, boas práticas e pseudocódigo ilustrativo sem vínculo a tecnologia específica.
>
> **Público-alvo:** Engenheiros projetando sistemas complexos com domínios ricos, preparando-se para entrevistas de arquitetura ou adotando DDD em produção.

---

## Quick Reference — Pilares do DDD

| Pilar | Foco | Pergunta-chave |
|-------|------|----------------|
| **Strategic Design** | Fronteiras e linguagem do negócio | *Quais são os contextos e como se relacionam?* |
| **Tactical Design** | Building blocks dentro de um contexto | *Como modelar Entities, VOs, Aggregates?* |
| **Context Mapping** | Integrações entre contextos | *Como dois contextos conversam?* |
| **Anti-Patterns** | Diagnóstico e correção | *Estou violando algum princípio DDD?* |
| **Arquitetura & Integração** | DDD + padrões arquiteturais | *Qual arquitetura melhor suporta meu domínio?* |

---

## Documentos

### 1. [Strategic Design](01-strategic-design.md)

Conceitos fundamentais de DDD no nível estratégico — **Ubiquitous Language**, **Bounded Contexts**, **Subdomains** (Core, Supporting, Generic) e como desenhar fronteiras entre modelos.

- Ubiquitous Language — Código fala a mesma língua do negócio
- Bounded Context — Um modelo por contexto, sem ambiguidade
- Subdomains — Identificar e priorizar core domain
- Context Discovery — Heurísticas para encontrar fronteiras

### 2. [Tactical Design (Building Blocks)](02-tactical-design.md)

Os blocos de construção do DDD dentro de um Bounded Context — **Entities**, **Value Objects**, **Aggregates**, **Domain Events**, **Repositories**, **Domain Services** e **Factories**.

- Entity — Identidade persiste ao longo do tempo
- Value Object — Igualdade por valor, imutabilidade
- Aggregate — Fronteira de consistência transacional
- Domain Event — Registrar fatos que ocorreram no domínio
- Repository — Coleção de Aggregates com persistência
- Domain Service — Lógica cross-entity stateless
- Factory — Encapsular criação complexa

### 3. [Context Mapping](03-context-mapping.md)

Padrões de integração entre Bounded Contexts — como mapear, documentar e implementar relações entre contextos com diferentes modelos e equipes.

- Shared Kernel, Customer-Supplier, Conformist
- Anti-Corruption Layer (ACL)
- Open Host Service / Published Language
- Separate Ways, Partnership, Big Ball of Mud

### 4. [Anti-Patterns & Code Smells](04-anti-patterns.md)

Diagnóstico e correção dos erros mais comuns ao aplicar DDD — desde **Anemic Domain Model** até **Big Ball of Mud**, classificados por severidade com sintomas, impactos e correções.

- Anemic Domain Model — Lógica espalhada fora do domínio
- God Aggregate — Aggregate grande demais
- Primitive Obsession — Usar primitivos em vez de Value Objects
- Shared Database — Contextos acoplados via banco
- CRUD Masquerading as DDD — Over-engineering em domínio simples

### 5. [Arquitetura & Integração](05-architecture-integration.md)

Como DDD se integra com padrões arquiteturais modernos — **Hexagonal**, **Clean Architecture**, **CQRS**, **Event Sourcing**, **Microservices** e **Modular Monolith**.

- Layered Architecture — Separa domínio de infra
- Hexagonal (Ports & Adapters) — Domínio no centro
- Clean Architecture — Regra de dependência protege domínio
- CQRS — Separa escrita (domínio rico) de leitura
- Event Sourcing — Persiste eventos de domínio
- Microservices — Bounded Context ≈ service boundary
- Modular Monolith — Bounded Contexts como módulos

---

## Mapa de Decisão

```
Novo projeto / funcionalidade
│
├─ Domínio complexo com regras ricas?
│   ├─ SIM → DDD completo (Strategic + Tactical)
│   │         ├─ Múltiplos times? → Context Mapping
│   │         ├─ Leitura/escrita assimétrica? → CQRS
│   │         └─ Audit trail necessário? → Event Sourcing
│   │
│   └─ NÃO → CRUD simples, não force DDD
│
├─ Sistema legado a integrar?
│   └─ Anti-Corruption Layer (Context Mapping)
│
└─ Monolito crescendo?
    └─ Modular Monolith → Microservices (Architecture & Integration)
```

---

## Referências

- Evans, Eric. *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003)
- Vernon, Vaughn. *Implementing Domain-Driven Design* (2013)
- Vernon, Vaughn. *Domain-Driven Design Distilled* (2016)
- Brandolini, Alberto. *EventStorming* — [eventstorming.com](https://www.eventstorming.com/)
- Millett, Scott & Tune, Nick. *Patterns, Principles, and Practices of Domain-Driven Design* (2015)
