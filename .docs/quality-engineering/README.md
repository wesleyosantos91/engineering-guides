# Quality Engineering — Guia Completo

> **Objetivo:** Servir como referência completa de **Quality Engineering**, cobrindo estratégia de testes, automação, performance, segurança, code review e observabilidade aplicada à qualidade.
> Otimizada para uso como **base de conhecimento para assistentes de código (Copilot/AI)** e consulta humana.
>
> **Público-alvo:** Engenheiros de software, QAs e tech leads que buscam elevar a maturidade de qualidade dos seus projetos com práticas modernas e integradas ao ciclo de desenvolvimento.

---

## Quick Reference — Pilares de Quality Engineering

| Pilar | Foco | Pergunta-chave |
|-------|------|----------------|
| **Estratégia de Testes** | Pirâmide, camadas, tipos | *Qual a estratégia de testes adequada para cada componente?* |
| **Automação** | Padrões reutilizáveis, receitas | *Como estruturar e manter testes automatizados?* |
| **Performance** | Carga, stress, SLOs | *O sistema atende os requisitos de performance?* |
| **Segurança** | SAST, DAST, OWASP | *O sistema está protegido contra vulnerabilidades conhecidas?* |
| **Code Review** | Revisão, checklists, feedback | *O código atende padrões de qualidade antes do merge?* |
| **Observabilidade** | Métricas, logs, traces, alertas | *Como monitorar qualidade em produção?* |

---

## Documentos

### 1. [Estratégia de Testes](01-testing-strategy.md)

Documento normativo que define a estratégia completa de testes — pirâmide de testes, padrões de escrita, estratégia por camada/componente, testes assíncronos e de resiliência, test doubles, quality gates no CI/CD e métricas.

- Pirâmide de Testes — proporção e escopo de cada nível
- Estratégia por camada — unit, integration, contract, e2e
- Testes de resiliência e assíncronos
- Test Doubles — mocks, stubs, fakes, spies
- Quality Gates no CI/CD e métricas de qualidade
- Definition of Done (DoD) com checklists

### 2. [Padrões de Automação de Testes](02-test-automation-patterns.md)

Receitas e padrões reutilizáveis para automação de testes — organização, criação de dados, setup/cleanup, asserções, testes de integração, API, assíncronos, banco de dados e refatoração.

- Organização e estrutura de testes (AAA, Given-When-Then)
- Test Data Builders e Object Mothers
- Padrões de setup e cleanup
- Asserções expressivas e custom matchers
- Testes de integração, API e banco de dados
- Receitas por linguagem (Java, Go, TypeScript)

### 3. [Performance Testing](03-performance-testing.md)

Guia completo de testes de performance — tipos de teste, metodologia, métricas e SLOs, ferramentas, padrões, análise de resultados e configuração de ambientes.

- Tipos de teste — load, stress, soak, spike, capacity
- Metodologia e planejamento de testes
- Métricas e SLOs (latência, throughput, error rate)
- Ferramentas (k6, Gatling, JMeter, Locust)
- Análise de resultados e gargalos
- Anti-patterns de performance testing

### 4. [Security Testing](04-security-testing.md)

Guia completo de testes de segurança — OWASP Top 10, SAST, DAST, SCA, segurança no CI/CD, APIs, containers, threat modeling e anti-patterns.

- OWASP Top 10 e como testar cada vulnerabilidade
- SAST — análise estática de segurança no código
- DAST — testes dinâmicos em runtime
- SCA — análise de dependências e CVEs
- Segurança de APIs e containers
- Threat Modeling e anti-patterns

### 5. [Code Review & Quality](05-code-review-quality.md)

Guia de revisão de código — princípios, checklists por área, padrões de feedback, processo de review, métricas e anti-patterns.

- Princípios de code review efetivo
- Checklist por área (lógica, segurança, performance, testes)
- Padrões de feedback construtivo
- Processo e fluxo de review
- Métricas de review e anti-patterns

### 6. [Observabilidade & Qualidade](06-observability-quality.md)

Guia de monitoramento aplicado a Quality Engineering — três pilares da observabilidade, SLIs/SLOs/SLAs, logging, métricas, tracing distribuído, alertas e dashboards.

- Três Pilares — logs, métricas, traces
- SLIs, SLOs e SLAs — definição e monitoramento
- Logging best practices e structured logging
- Métricas de aplicação e infraestrutura
- Distributed Tracing e correlação
- Estratégia de alertas e dashboards
- Observabilidade no CI/CD e anti-patterns

---

## Mapa de Decisão

```
                    ┌────────────────────────────────┐
                    │   O que você quer garantir?     │
                    └──────┬──────┬──────┬──────┬────┘
                           │      │      │      │
                  ┌────────▼─┐ ┌──▼────┐ │  ┌───▼──────────┐
                  │Corretude │ │Perf.  │ │  │ Segurança    │
                  │Funcional │ │SLOs   │ │  │ Compliance   │
                  └────┬─────┘ └──┬────┘ │  └───┬──────────┘
                       │          │      │      │
                  ┌────▼─────┐ ┌──▼─────────┐ ┌▼──────────────┐
                  │testing-  │ │performance-│ │security-      │
                  │strategy  │ │testing     │ │testing        │
                  └──────────┘ └────────────┘ └───────────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
   ┌──────▼──────┐ ┌───▼────────┐ ┌▼──────────────┐
   │test-        │ │code-review-│ │observability-  │
   │automation-  │ │quality     │ │quality         │
   │patterns     │ └────────────┘ └────────────────┘
   └─────────────┘
```

---

## Referências

- [Google Testing Blog](https://testing.googleblog.com/)
- [Martin Fowler — Testing](https://martinfowler.com/testing/)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [Google SRE Book — Testing](https://sre.google/sre-book/testing-reliability/)
- [Microsoft — Shift Left Testing](https://learn.microsoft.com/en-us/devops/develop/shift-left-make-testing-fast-reliable)
