# Zero to Hero — Quality Engineering Challenge (Go + Java Multi-Framework)

> **Programa de especialização progressiva** em Quality Engineering, cobrindo estratégia de testes,
> automação, performance, segurança, code review e observabilidade com implementações em
> **Go (Gin)** · **Spring Boot** · **Quarkus** · **Micronaut** · **Jakarta EE**

---

## 1. Resumo Executivo

Este programa transforma um desenvolvedor em **especialista em Quality Engineering** — da estratégia de testes (pirâmide, camadas, tipos) à observabilidade em produção (SLIs, SLOs, alertas). Todos os desafios são aplicados no **domínio Digital Wallet** (consistente com o backend challenge), com implementações em Go e Java (multi-framework).

**Domínio escolhido:** **Sistema de Gestão de Carteira Digital (Digital Wallet)** — mesmo domínio do backend challenge, garantindo continuidade de aprendizado.

**Por que Quality Engineering sobre Digital Wallet?**
- **Testes financeiros** — transações exigem cobertura rigorosa (saldos, concorrência, idempotência)
- **Performance crítica** — latência e throughput impactam diretamente a experiência do usuário
- **Segurança mandatória** — dados financeiros exigem OWASP, SAST, DAST, SCA
- **Observabilidade essencial** — detecção de anomalias, error budgets, SLOs financeiros
- Perguntas de entrevista Staff/Principal frequentemente envolvem qualidade de sistemas financeiros

**Referência:** [.docs/QUALITY-ENGINEERING/](../../.docs/QUALITY-ENGINEERING/)

---

## 2. Pilares de Quality Engineering

| Pilar | Foco | Pergunta-chave |
|-------|------|----------------|
| **Estratégia de Testes** | Pirâmide, camadas, tipos | *Qual a estratégia de testes adequada para cada componente?* |
| **Automação** | Padrões reutilizáveis, receitas | *Como estruturar e manter testes automatizados?* |
| **Performance** | Carga, stress, SLOs | *O sistema atende os requisitos de performance?* |
| **Segurança** | SAST, DAST, OWASP | *O sistema está protegido contra vulnerabilidades conhecidas?* |
| **Code Review** | Revisão, checklists, feedback | *O código atende padrões de qualidade antes do merge?* |
| **Observabilidade** | Métricas, logs, traces, alertas | *Como monitorar qualidade em produção?* |

---

## 3. Tecnologias e Ferramentas

### Ferramentas por Pilar

| Pilar | Go | Java (Spring/Quarkus/Micronaut/Jakarta) |
|-------|-----|------------------------------------------|
| **Unit Tests** | `testing` + testify | JUnit 5 + Mockito + AssertJ |
| **Integration** | testcontainers-go | Testcontainers + `@SpringBootTest` / `@QuarkusTest` / `@MicronautTest` |
| **Mocks** | mockery / gomock | Mockito / WireMock |
| **Coverage** | `go test -cover` | JaCoCo |
| **Mutation** | `go-mutesting` (opcional) | PIT (pitest) |
| **Contract** | pact-go | Pact JVM / Spring Cloud Contract |
| **Performance** | k6 / vegeta | k6 / Gatling / JMeter |
| **SAST** | gosec + golangci-lint | SonarQube / Semgrep / CodeQL |
| **SCA** | govulncheck | Snyk / OWASP Dependency-Check |
| **DAST** | OWASP ZAP | OWASP ZAP |
| **Observability** | OpenTelemetry Go SDK | OpenTelemetry Java Agent / Micrometer |
| **Logging** | `slog` / `zap` | SLF4J + Logback (JSON) |
| **Metrics** | Prometheus client_golang | Micrometer + Prometheus |
| **Tracing** | OpenTelemetry Go SDK | OpenTelemetry Java Agent |
| **Linting** | golangci-lint | Checkstyle / SpotBugs / ErrorProne |
| **Formatting** | `gofmt` / `goimports` | google-java-format / Spotless |

### Infraestrutura de Testes

| Ferramenta | Propósito |
|------------|-----------|
| **Testcontainers** | Containers efêmeros para PostgreSQL, Redis, Kafka, etc. |
| **WireMock** | Stub de APIs externas |
| **k6** | Testes de carga/performance (scripts em JavaScript) |
| **Gatling** | Testes de carga (scripts em Scala/Java) |
| **OWASP ZAP** | Scanner de segurança dinâmico |
| **SonarQube** | Análise estática de qualidade e segurança |
| **Prometheus + Grafana** | Métricas e dashboards |
| **Jaeger / Tempo** | Distributed tracing |
| **Loki** | Agregação de logs |

---

## 4. Mapa de Níveis

```
Level 0 — Fundamentos de Quality Engineering
  │       (Shift-Left, princípios, pirâmide, cultura de qualidade)
  ▼
Level 1 — Estratégia de Testes por Camada
  │       (Unitário, integração, contrato, E2E, test doubles, naming)
  ▼
Level 2 — Padrões de Automação de Testes
  │       (Builders, factories, fixtures, asserções, testes de API/DB)
  ▼
Level 3 — Performance Testing
  │       (Load, stress, spike, soak, SLOs, k6, Gatling, análise)
  ▼
Level 4 — Security Testing
  │       (OWASP Top 10, SAST, DAST, SCA, threat modeling)
  ▼
Level 5 — Code Review & Quality Gates
  │       (Checklists, feedback, métricas, CI/CD gates, mutation testing)
  ▼
Level 6 — Observabilidade & Qualidade em Produção
  │       (Logs, métricas, traces, SLIs/SLOs, alertas, dashboards)
  ▼
Level 7 — Capstone: Quality Engineering Completo
          (Pipeline end-to-end, quality gates, relatório de maturidade)
```

| Level | Tema | Foco | Desafios |
|-------|------|------|----------|
| [0](00-quality-foundations.md) | Fundamentos de Quality Engineering | Princípios, cultura, pirâmide | 6 |
| [1](01-testing-strategy.md) | Estratégia de Testes por Camada | Unit, integration, contract, E2E | 7 |
| [2](02-test-automation-patterns.md) | Padrões de Automação de Testes | Builders, fixtures, asserções, API tests | 7 |
| [3](03-performance-testing.md) | Performance Testing | Load, stress, SLOs, k6, Gatling | 7 |
| [4](04-security-testing.md) | Security Testing | OWASP, SAST, DAST, SCA | 7 |
| [5](05-code-review-quality.md) | Code Review & Quality Gates | Checklists, CI/CD, mutation testing | 6 |
| [6](06-observability-quality.md) | Observabilidade & Qualidade | Logs, métricas, traces, SLIs/SLOs | 7 |
| [7](07-capstone-quality-engineering.md) | Capstone: Quality Engineering | Pipeline completo, maturidade | 5 |

---

## 5. Entidades do Domínio (Digital Wallet)

```
User (1) ──── (N) Wallet (1) ──── (N) Transaction
                                        │
                                        └── type: DEPOSIT | WITHDRAWAL | TRANSFER
                                        └── status: PENDING | COMPLETED | FAILED | REVERSED
```

| Entidade | Campos Principais |
|----------|-------------------|
| **User** | `id` (UUID), `name`, `email` (unique), `document` (CPF/CNPJ), `status`, `created_at`, `updated_at` |
| **Wallet** | `id` (UUID), `user_id` (FK), `currency`, `balance` (decimal), `status`, `created_at`, `updated_at` |
| **Transaction** | `id` (UUID), `wallet_id` (FK), `target_wallet_id` (nullable FK), `type`, `amount`, `currency`, `status`, `description`, `idempotency_key` (unique), `created_at` |

---

## 6. Estrutura de Cada Desafio

Cada desafio inclui:
- **Contexto** — cenário de negócio no domínio Digital Wallet
- **Requisitos** — o que implementar com critérios objetivos
- **Código** — exemplos em Java 25 e Go 1.26 com frameworks relevantes
- **Critérios de aceite** — checklist verificável

---

## 7. Filosofia

1. **Shift-Left** — qualidade desde o design, não apenas no CI
2. **Automação primeiro** — se pode ser automatizado, não deveria ser manual
3. **Risk-based testing** — mais testes onde o risco é maior
4. **Feedback rápido** — testes em segundos, não minutos
5. **Observabilidade contínua** — qualidade não para no merge
6. **Security by default** — segurança integrada, não bolt-on
7. **Multi-linguagem** — mesmos princípios, implementações idiomáticas em Go e Java
8. **Portfólio real** — os projetos demonstram maturidade em entrevistas Staff/Principal
