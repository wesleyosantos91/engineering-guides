# Level 0 — Fundamentos de Quality Engineering

> **Objetivo:** Compreender os princípios fundamentais de Quality Engineering, a filosofia Shift-Left,
> a pirâmide de testes moderna, cultura de qualidade e como planejar uma estratégia de qualidade
> para o sistema Digital Wallet.

**Referência:** [.docs/QUALITY-ENGINEERING/README.md](../../.docs/QUALITY-ENGINEERING/README.md) · [.docs/QUALITY-ENGINEERING/01-testing-strategy.md](../../.docs/QUALITY-ENGINEERING/01-testing-strategy.md)

---

## Contexto do Domínio

Antes de escrever qualquer teste para a **Digital Wallet**, você precisa entender os fundamentos
que guiam **toda decisão de qualidade**. Neste nível, você analisará os princípios de Quality
Engineering e aplicará no domínio real de carteira digital:

- Onde investir mais em testes (transações financeiras vs. CRUD de usuário)?
- Qual o **custo de um bug** em cada camada (depósito duplicado vs. nome truncado)?
- Como mapear **risco de negócio** para **estratégia de testes**?
- Qual o nível de maturidade de qualidade do projeto?

---

## Desafios

### Desafio 0.1 — Mapeamento de Risco e Estratégia Shift-Left

**Contexto:** A Digital Wallet está sendo iniciada. Antes de escrever qualquer teste, a equipe precisa
mapear os riscos de negócio e definir onde investir em qualidade desde o design.

**Requisitos:**

- Mapear todas as funcionalidades da Digital Wallet em uma **matriz de risco**:
  - Eixo X: **Probabilidade** de bug (baixa/média/alta)
  - Eixo Y: **Impacto** de bug (baixo/médio/alto/crítico)
- Para cada funcionalidade, classificar o risco:
  - `CRUD de usuário` → impacto médio, probabilidade baixa
  - `Depósito` → impacto alto, probabilidade média
  - `Transferência entre wallets` → impacto crítico, probabilidade alta (concorrência)
  - `Saque com saldo insuficiente` → impacto alto, probabilidade média
  - `Idempotência de transação` → impacto crítico, probabilidade alta
  - `Consulta de extrato` → impacto baixo, probabilidade baixa
- Definir as atividades de Shift-Left para cada fase:

| Fase | Atividade de Qualidade (Digital Wallet) |
|------|----------------------------------------|
| **Design** | Revisão de contratos de API, threat modeling para pagamentos, critérios de aceitação |
| **Código** | TDD para regras de saldo, pair programming em transações, pré-commit hooks (lint, format) |
| **Build** | Unit tests, integration tests, cobertura ≥ 80%, SAST |
| **Deploy** | Smoke tests (depósito + saque), canary com métricas de latência |
| **Produção** | SLOs de latência/disponibilidade, alertas de error budget, synthetic monitoring |

- Documentar: "Qual o custo de um bug de **saldo duplicado** descoberto em cada fase?"

**Critérios de aceite:**

- [ ] Matriz de risco com pelo menos 8 funcionalidades mapeadas
- [ ] Classificação de risco para cada funcionalidade (probabilidade × impacto)
- [ ] Tabela Shift-Left com atividades por fase
- [ ] Análise de custo de bug por fase (design vs. código vs. produção)
- [ ] Documento: `decisions/00-risk-matrix.md`

---

### Desafio 0.2 — Pirâmide de Testes para Digital Wallet

**Contexto:** A equipe precisa definir a distribuição de testes por nível da pirâmide,
adaptada especificamente para o domínio financeiro.

**Requisitos:**

- Desenhar a pirâmide de testes da Digital Wallet com proporções:

```
          ╱  ╲
         ╱ E2E╲              ← 2-5%: Fluxo depósito → transferência → extrato
        ╱ Smoke ╲
       ╱─────────╲
      ╱ Contrato   ╲         ← 5-8%: Schema da API, eventos de transação
     ╱───────────────╲
    ╱   Integração     ╲     ← 20%: Repository + banco, service + Redis
   ╱─────────────────────╲
  ╱      Unitários         ╲  ← 70%: Regras de saldo, validações, cálculos
 ╱───────────────────────────╲
```

- Para cada nível, listar **pelo menos 3 cenários concretos** da Digital Wallet:
  - **Unitário:** validação de CPF, cálculo de saldo após saque, verificação de saldo insuficiente
  - **Integração:** persistência de transação no PostgreSQL, cache de wallet no Redis, migration de schema
  - **Contrato:** schema do endpoint `POST /transactions/deposit`, evento `TransactionCompleted`
  - **E2E:** fluxo completo de criar usuário → criar wallet → depositar → transferir → consultar extrato
- Definir **critérios de promoção**: quando um bug deve subir de nível na pirâmide?
- Documentar anti-patterns: "Por que não testar regras de saldo via E2E?"

**Critérios de aceite:**

- [ ] Pirâmide desenhada com proporções e cenários por nível
- [ ] Pelo menos 3 cenários concretos por nível (12+ total)
- [ ] Critérios de promoção entre níveis documentados
- [ ] Anti-patterns identificados com justificativa
- [ ] Documento: `decisions/00-test-pyramid.md`

---

### Desafio 0.3 — Princípios de Qualidade Aplicados

**Contexto:** A equipe precisa adotar os 7 princípios fundamentais de teste e demonstrar
como cada um se aplica na Digital Wallet.

**Requisitos:**

- Para cada princípio, criar um **exemplo concreto** e um **contra-exemplo** na Digital Wallet:

| Princípio | Exemplo correto | Contra-exemplo |
|-----------|----------------|----------------|
| **P1 — Feedback rápido** | Testes unitários de saldo rodam em < 100ms | Testar validação de CPF via E2E (30s+) |
| **P2 — Determinismo** | Teste de saque usa `BigDecimal("100.00")` fixo | Teste usa `LocalDateTime.now()` e falha à meia-noite |
| **P3 — Isolamento** | Cada teste cria seu próprio usuário/wallet | Testes compartilham wallet e a ordem importa |
| **P4 — Observabilidade no teste** | Mensagem: `"Saldo esperado: 50.00, atual: 0.00 para wallet abc-123"` | Mensagem: `"assertion failed"` |
| **P5 — Custo/Benefício** | 100% cobertura em `TransactionService` | 100% cobertura em getters/setters de `User` |
| **P6 — Teste como documentação** | `transferencia_comSaldoInsuficiente_retornaErro422()` | `test1()`, `testTransfer()` |
| **P7 — Propriedade coletiva** | Testes são refatorados junto com o código | Testes ignorados com `@Disabled` há 6 meses |

- Implementar 1 teste em Java e 1 em Go que **demonstre** cada princípio P1 e P3

**Java 25 (JUnit 5):**
```java
// P1 — Feedback rápido: teste unitário de regra de negócio
@Test
void saque_comSaldoSuficiente_atualizaSaldo() {
    // Arrange
    var wallet = WalletBuilder.umaWallet().comSaldo("100.00").build();

    // Act
    wallet.debit(new BigDecimal("30.00"));

    // Assert — roda em < 1ms
    assertThat(wallet.getBalance()).isEqualByComparingTo("70.00");
}

// P3 — Isolamento: cada teste cria seus próprios dados
@Test
void deposito_emWalletNova_incrementaSaldo() {
    // Arrange — wallet independente, criada aqui
    var wallet = WalletBuilder.umaWallet().comSaldo("0.00").build();

    // Act
    wallet.credit(new BigDecimal("50.00"));

    // Assert
    assertThat(wallet.getBalance()).isEqualByComparingTo("50.00");
}
```

**Go 1.26:**
```go
// P1 — Feedback rápido: teste unitário de regra de negócio
func TestWallet_Debit_WithSufficientBalance_UpdatesBalance(t *testing.T) {
    // Arrange
    wallet := NewTestWallet(WithBalance(100.00))

    // Act
    err := wallet.Debit(30.00)

    // Assert — roda em < 1ms
    require.NoError(t, err)
    assert.InDelta(t, 70.00, wallet.Balance, 0.001)
}

// P3 — Isolamento: cada teste cria seus próprios dados
func TestWallet_Credit_EmptyWallet_IncrementsBalance(t *testing.T) {
    // Arrange — wallet independente, criada aqui
    wallet := NewTestWallet(WithBalance(0.00))

    // Act
    wallet.Credit(50.00)

    // Assert
    assert.InDelta(t, 50.00, wallet.Balance, 0.001)
}
```

**Critérios de aceite:**

- [ ] Tabela com 7 princípios, exemplos e contra-exemplos
- [ ] Testes implementados em Java e Go para P1 e P3
- [ ] Testes passando (`./mvnw test`, `go test ./...`)
- [ ] Documento: `decisions/00-quality-principles.md`

---

### Desafio 0.4 — Tooling e Infraestrutura de Qualidade

**Contexto:** Configurar toda a infraestrutura de qualidade necessária para o projeto.

**Requisitos:**

- Configurar ferramentas de qualidade para **Go**:
  - `golangci-lint` com `.golangci.yml` (habilitar: errcheck, gosec, govet, staticcheck, unused)
  - `go test -cover` com threshold mínimo de 80%
  - `govulncheck` para scanning de dependências
  - `Makefile` com targets: `lint`, `test`, `test-cover`, `test-integration`, `security-scan`

- Configurar ferramentas de qualidade para **Java** (Spring Boot / Quarkus / Micronaut):
  - JaCoCo com threshold de 80% de cobertura
  - PIT (pitest) para mutation testing
  - Checkstyle ou Spotless para formatação
  - SpotBugs ou ErrorProne para análise estática
  - OWASP Dependency-Check para SCA
  - `pom.xml` com todos os plugins configurados

- Configurar pipeline de CI mínimo (GitHub Actions ou equivalente):

```yaml
# .github/workflows/quality.yml
jobs:
  quality:
    steps:
      - lint           # formatação e análise estática
      - test-unit      # testes unitários
      - test-integration  # testes de integração (Testcontainers)
      - coverage       # relatório de cobertura
      - security-scan  # SAST + SCA
```

**Critérios de aceite:**

- [ ] `golangci-lint` configurado e passando
- [ ] JaCoCo configurado com relatório HTML
- [ ] PIT configurado (pelo menos 1 classe target)
- [ ] Pipeline CI definido em YAML com 5+ steps
- [ ] `Makefile` (Go) e `pom.xml` (Java) com plugins de qualidade
- [ ] Todas as ferramentas executam sem erro

---

### Desafio 0.5 — Definition of Done (DoD) para Digital Wallet

**Contexto:** A equipe precisa estabelecer um DoD que garanta qualidade consistente em cada entrega.

**Requisitos:**

- Criar um DoD com itens verificáveis organizados por categoria:

```markdown
## Definition of Done — Digital Wallet

### Código
- [ ] Compila sem warnings
- [ ] Linter passa sem erros (golangci-lint / Checkstyle)
- [ ] Formatação aplicada (gofmt / google-java-format)
- [ ] Sem TODOs ou código comentado

### Testes
- [ ] Testes unitários para toda nova lógica de negócio
- [ ] Testes de integração para toda nova interação com I/O
- [ ] Cobertura ≥ 80% no service layer
- [ ] Cobertura geral ≥ 70%
- [ ] Nenhum teste usa sleep/wait fixo
- [ ] Testes são independentes (sem dependência de ordem)

### Segurança
- [ ] SAST sem vulnerabilidades HIGH/CRITICAL
- [ ] SCA sem CVEs HIGH/CRITICAL
- [ ] Inputs validados na borda
- [ ] Secrets não hardcoded

### Review
- [ ] PR revisado por pelo menos 1 peer
- [ ] Comentários de review endereçados
- [ ] PR tem descrição com contexto e impacto

### Documentação
- [ ] README atualizado (se mudança significativa)
- [ ] API docs (OpenAPI) atualizados
- [ ] ADR criado para decisões arquiteturais
```

- Criar **quality gates** para o CI/CD:

| Gate | Critério | Bloqueia merge? |
|------|----------|-----------------|
| Lint | 0 erros | ✅ Sim |
| Unit Tests | 100% passando | ✅ Sim |
| Coverage | ≥ 80% service, ≥ 70% geral | ✅ Sim |
| Integration Tests | 100% passando | ✅ Sim |
| SAST | 0 HIGH/CRITICAL | ✅ Sim |
| SCA | 0 HIGH/CRITICAL | ✅ Sim |
| Mutation | ≥ 60% mutation score | ⚠️ Warning |
| Performance | P95 ≤ 500ms | ⚠️ Warning |

**Critérios de aceite:**

- [ ] DoD documentado com checklist verificável
- [ ] Quality gates definidos com critérios e enforcement
- [ ] Pipeline CI implementa pelo menos 4 gates
- [ ] Documento: `decisions/00-definition-of-done.md`

---

### Desafio 0.6 — Avaliação de Maturidade de Quality Engineering

**Contexto:** Avaliar o nível de maturidade atual do projeto e criar um roadmap de evolução.

**Requisitos:**

- Avaliar o projeto nos 5 níveis de maturidade:

| Nível | Maturidade | Características |
|-------|-----------|-----------------|
| 0 | **Ad-hoc** | Sem testes, "funciona na minha máquina" |
| 1 | **Básico** | Testes unitários, CI básico, cobertura medida |
| 2 | **Estruturado** | Pirâmide de testes, integration tests, SAST |
| 3 | **Avançado** | Mutation testing, performance testing, SLOs |
| 4 | **Excelência** | Observability-driven, chaos engineering, quality-as-code |

- Para o projeto Digital Wallet, definir:
  - Nível atual (provavelmente 0-1 no início)
  - Nível alvo ao final deste programa (3-4)
  - Roadmap com milestones por nível
- Criar uma **tabela de métricas** que serão acompanhadas:

| Métrica | Nível 1 | Nível 2 | Nível 3 | Nível 4 |
|---------|---------|---------|---------|---------|
| Cobertura de código | ≥ 50% | ≥ 70% | ≥ 80% | ≥ 85% |
| Mutation score | — | — | ≥ 60% | ≥ 75% |
| Flaky test rate | < 10% | < 5% | < 1% | 0% |
| MTTR (Mean Time to Repair) | < 24h | < 4h | < 1h | < 15min |
| Deploy frequency | Semanal | Diário | Multi/dia | On-demand |
| Test execution time (CI) | < 30min | < 15min | < 10min | < 5min |

**Critérios de aceite:**

- [ ] Avaliação de maturidade com nível atual e alvo
- [ ] Roadmap com milestones e critérios de promoção
- [ ] Tabela de métricas com targets por nível
- [ ] Documento: `decisions/00-maturity-assessment.md`
