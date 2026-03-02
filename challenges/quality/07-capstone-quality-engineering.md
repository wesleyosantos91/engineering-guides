# Level 7 — Capstone: Quality Engineering Completo

> **Objetivo:** Integrar todos os pilares de Quality Engineering — testes, automação, performance,
> segurança, code review e observabilidade — em um pipeline end-to-end para a Digital Wallet,
> produzir um relatório de maturidade e criar um portfólio demonstrável para entrevistas Staff/Principal.

**Referência:** [.docs/QUALITY-ENGINEERING/](../../.docs/QUALITY-ENGINEERING/)

---

## Contexto do Domínio

Este é o desafio final. Você vai consolidar **tudo** o que aprendeu nos Levels 0–6 em um
sistema de qualidade completo e coeso. A Digital Wallet precisa de um pipeline que garanta
qualidade desde o commit até a produção — com feedback rápido, gates automatizados, métricas
de maturidade e observabilidade contínua.

**O capstone responde à pergunta:** *"Se eu fosse o Staff Engineer/SRE responsável por esta
aplicação, como eu garantiria qualidade em cada etapa do ciclo de vida?"*

---

## Desafios

### Desafio 7.1 — Pipeline de Qualidade End-to-End

**Contexto:** Um pipeline de qualidade completo não é apenas "rodar testes no CI". É uma
sequência orquestrada de gates que verifica qualidade em múltiplas dimensões — funcional,
estrutural, segurança, performance e observabilidade — com feedback rápido.

**Requisitos:**

- Implementar pipeline CI/CD completo com **todos** os quality gates dos níveis anteriores:

```yaml
# .github/workflows/quality-pipeline.yml
name: Digital Wallet — Quality Engineering Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  GO_VERSION: '1.26'
  JAVA_VERSION: '25'
  K6_VERSION: '0.55.0'

jobs:
  # ──────────────────────────────────────────────
  # STAGE 1: Fast Feedback (< 2 min)
  # ──────────────────────────────────────────────
  lint-format:
    name: "Stage 1 — Lint & Format"
    runs-on: ubuntu-latest
    strategy:
      matrix:
        stack: [go-gin, java-spring, java-quarkus, java-micronaut, java-jakarta]
    steps:
      - uses: actions/checkout@v4

      # Go
      - name: Go — golangci-lint
        if: matrix.stack == 'go-gin'
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest
          args: --timeout=5m

      - name: Go — gofmt check
        if: matrix.stack == 'go-gin'
        run: |
          test -z "$(gofmt -l .)" || (echo "gofmt check failed" && exit 1)

      # Java
      - name: Java — Spotless check
        if: startsWith(matrix.stack, 'java-')
        run: ./mvnw spotless:check -pl ${{ matrix.stack }}

      - name: Java — SpotBugs
        if: startsWith(matrix.stack, 'java-')
        run: ./mvnw spotbugs:check -pl ${{ matrix.stack }}

  compile:
    name: "Stage 1 — Compile"
    runs-on: ubuntu-latest
    strategy:
      matrix:
        stack: [go-gin, java-spring, java-quarkus, java-micronaut, java-jakarta]
    steps:
      - uses: actions/checkout@v4

      - name: Go — Build
        if: matrix.stack == 'go-gin'
        run: go build ./...

      - name: Java — Compile
        if: startsWith(matrix.stack, 'java-')
        run: ./mvnw compile -pl ${{ matrix.stack }} -DskipTests

  # ──────────────────────────────────────────────
  # STAGE 2: Unit Tests + Coverage (< 5 min)
  # ──────────────────────────────────────────────
  unit-tests:
    name: "Stage 2 — Unit Tests + Coverage"
    needs: [lint-format, compile]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        stack: [go-gin, java-spring, java-quarkus, java-micronaut, java-jakarta]
    steps:
      - uses: actions/checkout@v4

      - name: Go — Unit Tests + Coverage
        if: matrix.stack == 'go-gin'
        run: |
          go test ./... -short -race -coverprofile=coverage.out -covermode=atomic
          go tool cover -func=coverage.out | tail -1
          # Gate: coverage ≥ 80%
          COVERAGE=$(go tool cover -func=coverage.out | tail -1 | awk '{print $3}' | sed 's/%//')
          echo "Coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "FAIL: Coverage ${COVERAGE}% < 80% threshold"
            exit 1
          fi

      - name: Java — Unit Tests + JaCoCo
        if: startsWith(matrix.stack, 'java-')
        run: |
          ./mvnw verify -pl ${{ matrix.stack }} \
            -Dtest='*UnitTest,*Test' \
            -DfailIfNoTests=false \
            -Djacoco.check.lineRatio=0.80 \
            -Djacoco.check.branchRatio=0.70

  mutation-tests:
    name: "Stage 2 — Mutation Testing"
    needs: [lint-format, compile]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        stack: [java-spring, java-quarkus, java-micronaut, java-jakarta]
    steps:
      - uses: actions/checkout@v4

      - name: Java — PIT Mutation Testing
        run: |
          ./mvnw org.pitest:pitest-maven:mutationCoverage -pl ${{ matrix.stack }} \
            -DtargetClasses="com.wallet.domain.*" \
            -DmutationThreshold=70

  # ──────────────────────────────────────────────
  # STAGE 3: Security (< 5 min)
  # ──────────────────────────────────────────────
  security-sast:
    name: "Stage 3 — SAST"
    needs: [unit-tests]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        stack: [go-gin, java-spring, java-quarkus, java-micronaut, java-jakarta]
    steps:
      - uses: actions/checkout@v4

      - name: Go — gosec
        if: matrix.stack == 'go-gin'
        run: |
          go install github.com/securego/gosec/v2/cmd/gosec@latest
          gosec -severity medium -confidence medium -fmt sarif -out gosec.sarif ./...

      - name: Java — Semgrep
        if: startsWith(matrix.stack, 'java-')
        run: |
          pip install semgrep
          semgrep --config=p/java --sarif --output=semgrep.sarif ./${{ matrix.stack }}

  security-sca:
    name: "Stage 3 — SCA (Dependency Check)"
    needs: [unit-tests]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        stack: [go-gin, java-spring, java-quarkus, java-micronaut, java-jakarta]
    steps:
      - uses: actions/checkout@v4

      - name: Go — govulncheck
        if: matrix.stack == 'go-gin'
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

      - name: Java — OWASP Dependency-Check
        if: startsWith(matrix.stack, 'java-')
        run: |
          ./mvnw org.owasp:dependency-check-maven:check -pl ${{ matrix.stack }} \
            -DfailBuildOnCVSS=7

  secret-scan:
    name: "Stage 3 — Secret Scanning"
    needs: [unit-tests]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}

  # ──────────────────────────────────────────────
  # STAGE 4: Integration Tests (< 10 min)
  # ──────────────────────────────────────────────
  integration-tests:
    name: "Stage 4 — Integration Tests"
    needs: [security-sast, security-sca, secret-scan]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        stack: [go-gin, java-spring, java-quarkus, java-micronaut, java-jakarta]
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: wallet
          POSTGRES_PASSWORD: wallet
          POSTGRES_DB: wallet_test
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 10s
    steps:
      - uses: actions/checkout@v4

      - name: Go — Integration Tests
        if: matrix.stack == 'go-gin'
        run: |
          go test ./... -run Integration -tags=integration -race \
            -timeout 10m -v

      - name: Java — Integration Tests
        if: startsWith(matrix.stack, 'java-')
        run: |
          ./mvnw verify -pl ${{ matrix.stack }} \
            -Dtest='*IntegrationTest,*IT' \
            -DfailIfNoTests=false

  # ──────────────────────────────────────────────
  # STAGE 5: DAST (< 15 min)
  # ──────────────────────────────────────────────
  security-dast:
    name: "Stage 5 — DAST (OWASP ZAP)"
    needs: [integration-tests]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start application
        run: docker compose up -d --wait

      - name: OWASP ZAP API Scan
        uses: zaproxy/action-api-scan@v0.9.0
        with:
          target: 'http://localhost:8080/v3/api-docs'
          rules_file_name: 'zap-rules.tsv'
          fail_action: true

      - name: Stop application
        if: always()
        run: docker compose down

  # ──────────────────────────────────────────────
  # STAGE 6: Performance (< 15 min)
  # ──────────────────────────────────────────────
  performance-tests:
    name: "Stage 6 — Performance (k6)"
    needs: [integration-tests]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start application
        run: docker compose up -d --wait

      - name: Run k6 load test
        run: |
          docker run --network=host \
            -v $(pwd)/k6:/scripts \
            grafana/k6:${{ env.K6_VERSION }} run \
            --out json=k6-results.json \
            /scripts/scenarios/baseline.js

      - name: Verify SLOs
        run: |
          # P99 latency ≤ 300ms (SLO)
          P99=$(jq '.metrics.http_req_duration.values["p(99)"]' k6-results.json)
          echo "P99 Latency: ${P99}ms"
          if (( $(echo "$P99 > 300" | bc -l) )); then
            echo "FAIL: P99 ${P99}ms > 300ms SLO"
            exit 1
          fi

          # Error rate < 1%
          ERROR_RATE=$(jq '.metrics.http_req_failed.values.rate' k6-results.json)
          echo "Error rate: ${ERROR_RATE}"
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "FAIL: Error rate ${ERROR_RATE} > 1%"
            exit 1
          fi

      - name: Stop application
        if: always()
        run: docker compose down

  # ──────────────────────────────────────────────
  # STAGE 7: Deploy + Observability (production)
  # ──────────────────────────────────────────────
  deploy-canary:
    name: "Stage 7 — Canary Deploy"
    needs: [security-dast, performance-tests]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy canary (10% traffic)
        run: |
          kubectl apply -f k8s/canary-rollout.yaml
          # Esperar análise de canary
          kubectl argo rollouts status digital-wallet --timeout=15m

      - name: Verify SLOs post-deploy
        run: |
          # Consultar Prometheus para error rate do canary
          ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query?query=\
            sum(rate(http_requests_total{version='canary',status=~'5..'}[5m]))\
            /sum(rate(http_requests_total{version='canary'}[5m]))" | jq '.data.result[0].value[1]')
          echo "Canary error rate: ${ERROR_RATE}"

      - name: Create deploy annotation
        run: |
          curl -X POST http://grafana:3000/api/annotations \
            -H "Authorization: Bearer ${{ secrets.GRAFANA_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"text":"Deploy ${{ github.sha }}", "tags":["deploy","wallet"]}'

  # ──────────────────────────────────────────────
  # STAGE 8: Quality Report
  # ──────────────────────────────────────────────
  quality-report:
    name: "Stage 8 — Quality Report"
    needs: [unit-tests, mutation-tests, security-sast, security-sca, integration-tests, performance-tests]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate quality report
        run: |
          echo "# Quality Report — $(date -u +%Y-%m-%d)" > quality-report.md
          echo "" >> quality-report.md
          echo "## Pipeline Results" >> quality-report.md
          echo "| Stage | Status |" >> quality-report.md
          echo "|-------|--------|" >> quality-report.md
          echo "| Lint & Format | ${{ needs.lint-format.result }} |" >> quality-report.md
          echo "| Unit Tests | ${{ needs.unit-tests.result }} |" >> quality-report.md
          echo "| Mutation Tests | ${{ needs.mutation-tests.result }} |" >> quality-report.md
          echo "| SAST | ${{ needs.security-sast.result }} |" >> quality-report.md
          echo "| SCA | ${{ needs.security-sca.result }} |" >> quality-report.md
          echo "| Integration Tests | ${{ needs.integration-tests.result }} |" >> quality-report.md
          echo "| Performance | ${{ needs.performance-tests.result }} |" >> quality-report.md

      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: quality-report
          path: quality-report.md
```

- **Stage gating rules:**

| Stage | Tempo máximo | Gate de falha |
|---|---|---|
| 1 — Lint & Format | < 2 min | Qualquer lint error bloqueia |
| 2 — Unit + Coverage + Mutation | < 5 min | Coverage < 80% ou mutation < 70% |
| 3 — Security (SAST + SCA + Secrets) | < 5 min | High/Critical vuln ou secret detectado |
| 4 — Integration Tests | < 10 min | Qualquer teste falhando |
| 5 — DAST | < 15 min | High risk finding |
| 6 — Performance | < 15 min | SLO violado (P99 > 300ms ou error > 1%) |
| 7 — Deploy + Canary | < 20 min | Canary error rate > baseline + 0.5% |
| 8 — Quality Report | < 2 min | Informativo (sempre gera) |

**Critérios de aceite:**

- [ ] Pipeline YAML funcional com 8 stages
- [ ] Cada stage tem gate de falha explícito
- [ ] Pipeline roda para todas as 5 stacks (Go + 4 Java)
- [ ] Tempo total < 30 minutos (com paralelismo)
- [ ] Quality report gerado como artefato
- [ ] Pipeline testado com pelo menos 1 stack completa

---

### Desafio 7.2 — Quality Maturity Assessment

**Contexto:** Para evoluir a qualidade de um sistema, primeiro é preciso saber onde estamos.
Um Quality Maturity Assessment mede a maturidade atual contra um modelo de referência e
identifica gaps prioritários.

**Requisitos:**

- Criar **Quality Maturity Model** baseado em 6 pilares:

```yaml
# quality-maturity-model.yaml
model:
  name: "Digital Wallet Quality Maturity"
  version: "1.0"
  date: "2026-03-01"

pillars:
  - name: "Testing Strategy"
    levels:
      1-Initial:
        description: "Testes manuais, sem estratégia definida"
        indicators:
          - "Sem testes automatizados"
          - "Coverage desconhecido"
          - "Bugs encontrados em produção"
      2-Managed:
        description: "Testes unitários básicos, CI simples"
        indicators:
          - "Testes unitários existem (coverage < 50%)"
          - "CI roda testes automaticamente"
          - "Bugs encontrados em staging"
      3-Defined:
        description: "Pirâmide de testes implementada, coverage > 70%"
        indicators:
          - "Unit > Integration > E2E (proporção correta)"
          - "Coverage > 70% com metas por camada"
          - "Contract tests entre serviços"
      4-Quantitatively Managed:
        description: "Métricas de qualidade rastreadas, mutation testing"
        indicators:
          - "Coverage > 80% + mutation score > 70%"
          - "Métricas de flaky tests rastreadas"
          - "Test execution time otimizado (< 5min unit)"
      5-Optimizing:
        description: "Melhoria contínua, risk-based testing, AI-assisted"
        indicators:
          - "Risk-based test selection"
          - "Test impact analysis"
          - "Continuous testing em produção (synthetic monitors)"

  - name: "Security Testing"
    levels:
      1-Initial:
        description: "Sem varredura de segurança automatizada"
        indicators:
          - "Sem SAST/DAST/SCA"
          - "Dependências nunca auditadas"
          - "Sem threat modeling"
      2-Managed:
        description: "SAST básico no CI"
        indicators:
          - "SAST roda no CI (gosec/SpotBugs)"
          - "Dependências verificadas manualmente"
      3-Defined:
        description: "SAST + SCA + DAST + Secret scanning"
        indicators:
          - "SAST + SCA + DAST integrados ao CI"
          - "Secret scanning com gitleaks"
          - "OWASP Top 10 coberto em testes"
      4-Quantitatively Managed:
        description: "Threat model completo, métricas de segurança"
        indicators:
          - "STRIDE threat model documentado"
          - "Mean time to remediate CVE rastreado"
          - "Security headers verificados automaticamente"
      5-Optimizing:
        description: "Security champions, bug bounty, penetration testing regular"
        indicators:
          - "Penetration testing trimestral"
          - "Security champions por squad"
          - "Vulnerabilidades zero-day < 24h para patch"

  - name: "Performance Testing"
    levels:
      1-Initial:
        description: "Sem testes de performance"
        indicators:
          - "Performance testada manualmente"
          - "Sem SLOs definidos"
      2-Managed:
        description: "Load tests básicos antes de release"
        indicators:
          - "k6/Gatling roda antes de major releases"
          - "SLOs informais"
      3-Defined:
        description: "Load/stress/spike tests no CI, SLOs definidos"
        indicators:
          - "Testes de performance no pipeline"
          - "SLOs com targets numéricos"
          - "Baseline comparisons automatizados"
      4-Quantitatively Managed:
        description: "Profiling contínuo, SLO compliance tracked"
        indicators:
          - "Continuous profiling (pprof/async-profiler)"
          - "SLO compliance dashboard"
          - "Performance regression detection"
      5-Optimizing:
        description: "Chaos engineering, adaptive capacity"
        indicators:
          - "Chaos experiments regulares"
          - "Auto-scaling baseado em métricas de negócio"
          - "Capacity planning data-driven"

  - name: "Code Review & Quality Gates"
    levels:
      1-Initial:
        description: "Reviews informais, sem checklists"
        indicators:
          - "Reviews não são obrigatórios"
          - "Sem quality gates no CI"
      2-Managed:
        description: "Reviews obrigatórios, gates básicos"
        indicators:
          - "PR reviews obrigatórios (1 approver)"
          - "CI roda testes antes do merge"
      3-Defined:
        description: "Checklists, CODEOWNERS, múltiplos gates"
        indicators:
          - "Checklists de review domain-specific"
          - "CODEOWNERS configurado"
          - "5+ quality gates no pipeline"
      4-Quantitatively Managed:
        description: "Review metrics (cycle time, throughput)"
        indicators:
          - "PR cycle time < 24h"
          - "Review throughput rastreado"
          - "Mutation testing no pipeline"
      5-Optimizing:
        description: "AI-assisted review, auto-fix"
        indicators:
          - "Auto-fix para formatting/linting"
          - "Code suggestions automatizados"
          - "Knowledge sharing via review comments"

  - name: "Observability"
    levels:
      1-Initial:
        description: "printf debugging, logs em texto plano"
        indicators:
          - "Logs não estruturados"
          - "Sem métricas de aplicação"
          - "Sem tracing"
      2-Managed:
        description: "Logging estruturado, métricas básicas"
        indicators:
          - "JSON logging configurado"
          - "Health check endpoint"
          - "CPU/memory monitorados"
      3-Defined:
        description: "3 pilares implementados + SLOs"
        indicators:
          - "Structured logging + metrics + tracing"
          - "SLIs/SLOs definidos"
          - "Dashboards por serviço"
      4-Quantitatively Managed:
        description: "Error budgets, alertas burn-rate, DORA"
        indicators:
          - "Error budget tracking"
          - "Multi-window burn rate alerts"
          - "DORA metrics rastreados"
      5-Optimizing:
        description: "AIOps, anomaly detection, self-healing"
        indicators:
          - "Anomaly detection ML-based"
          - "Auto-remediation para cenários conhecidos"
          - "Canary analysis automatizado"

  - name: "CI/CD Pipeline"
    levels:
      1-Initial:
        description: "Deploy manual, sem CI"
        indicators:
          - "Build manual"
          - "Deploy manual para produção"
      2-Managed:
        description: "CI básico, deploy semi-automatizado"
        indicators:
          - "CI compila e testa"
          - "Deploy com script/aprovação manual"
      3-Defined:
        description: "CD pipeline completo, quality gates"
        indicators:
          - "Pipeline com múltiplos stages"
          - "Quality gates automatizados"
          - "Deploy automatizado para staging"
      4-Quantitatively Managed:
        description: "Pipeline metrics, deployment frequency"
        indicators:
          - "Pipeline duration rastreado"
          - "Deployment frequency > 1/dia"
          - "Change failure rate < 15%"
      5-Optimizing:
        description: "Canary deploys, feature flags, zero-downtime"
        indicators:
          - "Canary deploy automatizado"
          - "Feature flags para progressive delivery"
          - "Zero-downtime deployments"
```

- Criar **script de self-assessment** que avalia a Digital Wallet:

**Java 25:**
```java
public record QualityAssessment(
        String pillar,
        int level,
        String levelName,
        List<String> evidences,
        List<String> gaps,
        String recommendation
) {}

public class QualityMaturityAssessor {

    public List<QualityAssessment> assess(ProjectContext project) {
        var assessments = new ArrayList<QualityAssessment>();

        // Testing Strategy
        assessments.add(assessTestingStrategy(project));
        assessments.add(assessSecurityTesting(project));
        assessments.add(assessPerformanceTesting(project));
        assessments.add(assessCodeReview(project));
        assessments.add(assessObservability(project));
        assessments.add(assessCiCd(project));

        return assessments;
    }

    private QualityAssessment assessTestingStrategy(ProjectContext project) {
        var evidences = new ArrayList<String>();
        var gaps = new ArrayList<String>();

        int level = 1;

        if (project.hasUnitTests()) {
            level = 2;
            evidences.add("Unit tests present");
        }

        if (project.getCoverage() >= 70 && project.hasContractTests()) {
            level = 3;
            evidences.add("Coverage: " + project.getCoverage() + "%");
            evidences.add("Contract tests: present");
        }

        if (project.getCoverage() >= 80 && project.getMutationScore() >= 70) {
            level = 4;
            evidences.add("Mutation score: " + project.getMutationScore() + "%");
        } else {
            gaps.add("Mutation score below 70% (current: "
                    + project.getMutationScore() + "%)");
        }

        return new QualityAssessment(
                "Testing Strategy", level, levelName(level),
                evidences, gaps,
                level < 4 ? "Adicionar mutation testing e elevar coverage para > 80%"
                        : "Explorar risk-based test selection"
        );
    }
}
```

**Go 1.26:**
```go
type QualityAssessment struct {
    Pillar         string   `json:"pillar"`
    Level          int      `json:"level"`
    LevelName      string   `json:"level_name"`
    Evidences      []string `json:"evidences"`
    Gaps           []string `json:"gaps"`
    Recommendation string   `json:"recommendation"`
}

func AssessTestingStrategy(project ProjectContext) QualityAssessment {
    var evidences, gaps []string
    level := 1

    if project.HasUnitTests {
        level = 2
        evidences = append(evidences, "Unit tests present")
    }

    if project.Coverage >= 70 && project.HasContractTests {
        level = 3
        evidences = append(evidences, fmt.Sprintf("Coverage: %.1f%%", project.Coverage))
    }

    if project.Coverage >= 80 && project.MutationScore >= 70 {
        level = 4
        evidences = append(evidences,
            fmt.Sprintf("Mutation score: %.1f%%", project.MutationScore))
    } else {
        gaps = append(gaps,
            fmt.Sprintf("Mutation score below 70%% (current: %.1f%%)", project.MutationScore))
    }

    recommendation := "Adicionar mutation testing e elevar coverage para > 80%"
    if level >= 4 {
        recommendation = "Explorar risk-based test selection"
    }

    return QualityAssessment{
        Pillar: "Testing Strategy", Level: level, LevelName: levelName(level),
        Evidences: evidences, Gaps: gaps, Recommendation: recommendation,
    }
}
```

- Gerar **radar chart** ou relatório visual (JSON para frontend/CLI):

```json
{
  "service": "digital-wallet",
  "date": "2026-03-01",
  "overall_score": 3.5,
  "assessments": [
    { "pillar": "Testing Strategy", "level": 4, "max": 5 },
    { "pillar": "Security Testing", "level": 3, "max": 5 },
    { "pillar": "Performance Testing", "level": 3, "max": 5 },
    { "pillar": "Code Review", "level": 4, "max": 5 },
    { "pillar": "Observability", "level": 4, "max": 5 },
    { "pillar": "CI/CD Pipeline", "level": 3, "max": 5 }
  ],
  "top_gaps": [
    "DAST not integrated in pipeline",
    "No chaos engineering experiments",
    "DORA metrics not tracked automatically"
  ],
  "action_plan": [
    { "priority": 1, "action": "Integrate OWASP ZAP in CI", "effort": "2 sprints" },
    { "priority": 2, "action": "Implement DORA metrics tracking", "effort": "1 sprint" },
    { "priority": 3, "action": "Add chaos experiments", "effort": "3 sprints" }
  ]
}
```

**Critérios de aceite:**

- [ ] Quality Maturity Model com 6 pilares × 5 níveis
- [ ] Cada nível tem 3+ indicadores mensuráveis
- [ ] Script de self-assessment funcional (Go + Java)
- [ ] Relatório JSON com score por pilar e overall
- [ ] Top gaps identificados com action plan priorizado
- [ ] Documento: `quality-maturity-assessment.yaml`

---

### Desafio 7.3 — Production Readiness Checklist

**Contexto:** Antes de um serviço ir para produção, precisa passar por uma
Production Readiness Review (PRR). Este checklist integra todos os pilares
e garante que a Digital Wallet está pronta.

**Requisitos:**

- Criar **Production Readiness Checklist** automatizado:

```yaml
# production-readiness-checklist.yaml
service: digital-wallet
reviewer: "@team-wallet"
date: "2026-03-01"

categories:
  # ── FUNCIONALIDADE ──
  functionality:
    - id: FUNC-01
      name: "Unit test coverage ≥ 80%"
      automated: true
      check: "JaCoCo / go tool cover"
      status: null  # preenchido pelo script

    - id: FUNC-02
      name: "Integration tests passando (Testcontainers)"
      automated: true
      check: "CI Stage 4"
      status: null

    - id: FUNC-03
      name: "Contract tests passando"
      automated: true
      check: "Pact verification"
      status: null

    - id: FUNC-04
      name: "Mutation testing score ≥ 70%"
      automated: true
      check: "PIT report"
      status: null

  # ── SEGURANÇA ──
  security:
    - id: SEC-01
      name: "SAST: zero high/critical findings"
      automated: true
      check: "gosec / SpotBugs / Semgrep"
      status: null

    - id: SEC-02
      name: "SCA: zero high/critical CVEs"
      automated: true
      check: "govulncheck / OWASP Dependency-Check"
      status: null

    - id: SEC-03
      name: "DAST: zero high risk findings"
      automated: true
      check: "OWASP ZAP"
      status: null

    - id: SEC-04
      name: "Secret scanning: zero findings"
      automated: true
      check: "gitleaks"
      status: null

    - id: SEC-05
      name: "Threat model documentado (STRIDE)"
      automated: false
      check: "Manual review"
      status: null

    - id: SEC-06
      name: "Security headers configurados"
      automated: true
      check: "DAST headers check"
      status: null

  # ── PERFORMANCE ──
  performance:
    - id: PERF-01
      name: "Load test baseline existe"
      automated: true
      check: "k6 baseline.js"
      status: null

    - id: PERF-02
      name: "SLOs definidos e validados"
      automated: true
      check: "k6 com thresholds"
      status: null

    - id: PERF-03
      name: "P99 latência ≤ 300ms (deposit)"
      automated: true
      check: "k6 threshold"
      status: null

    - id: PERF-04
      name: "Stress test executado sem crash"
      automated: true
      check: "k6 stress scenario"
      status: null

  # ── OBSERVABILIDADE ──
  observability:
    - id: OBS-01
      name: "Logging estruturado (JSON)"
      automated: true
      check: "Log format validation"
      status: null

    - id: OBS-02
      name: "Métricas RED expostas via /metrics"
      automated: true
      check: "Prometheus scrape test"
      status: null

    - id: OBS-03
      name: "Distributed tracing com OpenTelemetry"
      automated: true
      check: "Trace propagation test"
      status: null

    - id: OBS-04
      name: "SLIs/SLOs definidos com Prometheus rules"
      automated: false
      check: "SLO YAML review"
      status: null

    - id: OBS-05
      name: "Alertas configurados (burn rate)"
      automated: false
      check: "Alert rules review"
      status: null

    - id: OBS-06
      name: "Dashboard de serviço criado"
      automated: true
      check: "Grafana API check"
      status: null

    - id: OBS-07
      name: "Health check endpoint funcional"
      automated: true
      check: "GET /health returns 200"
      status: null

  # ── OPERACIONAL ──
  operational:
    - id: OPS-01
      name: "Runbooks para todos os alertas"
      automated: false
      check: "Manual review"
      status: null

    - id: OPS-02
      name: "Rollback procedure documentado"
      automated: false
      check: "Manual review"
      status: null

    - id: OPS-03
      name: "Graceful shutdown implementado"
      automated: true
      check: "Integration test"
      status: null

    - id: OPS-04
      name: "Resource limits (CPU/memory) definidos"
      automated: true
      check: "k8s manifest check"
      status: null

    - id: OPS-05
      name: "CODEOWNERS configurado"
      automated: true
      check: "File exists check"
      status: null
```

- Implementar **script de validação** que executa os checks automatizados:

**Go 1.26:**
```go
type CheckResult struct {
    ID        string `json:"id"`
    Name      string `json:"name"`
    Status    string `json:"status"` // PASS, FAIL, SKIP
    Details   string `json:"details,omitempty"`
    Automated bool   `json:"automated"`
}

type ReadinessReport struct {
    Service   string          `json:"service"`
    Date      string          `json:"date"`
    Results   []CheckResult   `json:"results"`
    Summary   ReadinessSummary `json:"summary"`
    Ready     bool            `json:"ready"`
}

type ReadinessSummary struct {
    Total     int `json:"total"`
    Passed    int `json:"passed"`
    Failed    int `json:"failed"`
    Skipped   int `json:"skipped"`
    PassRate  float64 `json:"pass_rate"`
}

func RunReadinessChecks(ctx context.Context, config ChecklistConfig) (*ReadinessReport, error) {
    var results []CheckResult

    // FUNC-01: Coverage check
    coverage, err := getCoveragePercent(ctx)
    if err != nil {
        results = append(results, CheckResult{
            ID: "FUNC-01", Name: "Unit test coverage ≥ 80%",
            Status: "FAIL", Details: err.Error(), Automated: true,
        })
    } else {
        status := "PASS"
        if coverage < 80 {
            status = "FAIL"
        }
        results = append(results, CheckResult{
            ID: "FUNC-01", Name: "Unit test coverage ≥ 80%",
            Status: status, Details: fmt.Sprintf("%.1f%%", coverage), Automated: true,
        })
    }

    // SEC-04: Secret scanning
    secrets, err := runGitleaks(ctx)
    if err != nil || secrets > 0 {
        results = append(results, CheckResult{
            ID: "SEC-04", Name: "Secret scanning: zero findings",
            Status: "FAIL", Details: fmt.Sprintf("%d secrets found", secrets), Automated: true,
        })
    } else {
        results = append(results, CheckResult{
            ID: "SEC-04", Name: "Secret scanning: zero findings",
            Status: "PASS", Automated: true,
        })
    }

    // OBS-07: Health check
    resp, err := http.Get(config.BaseURL + "/health")
    if err != nil || resp.StatusCode != 200 {
        results = append(results, CheckResult{
            ID: "OBS-07", Name: "Health check endpoint funcional",
            Status: "FAIL", Details: "Health check failed", Automated: true,
        })
    } else {
        results = append(results, CheckResult{
            ID: "OBS-07", Name: "Health check endpoint funcional",
            Status: "PASS", Automated: true,
        })
    }

    // ... remaining checks

    summary := calculateSummary(results)
    return &ReadinessReport{
        Service: "digital-wallet",
        Date:    time.Now().Format(time.DateOnly),
        Results: results,
        Summary: summary,
        Ready:   summary.Failed == 0,
    }, nil
}
```

**Critérios de aceite:**

- [ ] Production Readiness Checklist com 5+ categorias e 20+ checks
- [ ] Checks automatizados identificados (≥ 70% automatizáveis)
- [ ] Script de validação que roda checks automatizados
- [ ] Relatório JSON com status e summary
- [ ] Flag `Ready: true/false` baseado em zero failures
- [ ] Documento: `production-readiness-checklist.yaml`

---

### Desafio 7.4 — Quality Engineering Decision Records

**Contexto:** Decisões sobre qualidade devem ser documentadas como ADRs (Architecture Decision Records)
para garantir rastreabilidade e onboarding de novos membros.

**Requisitos:**

- Criar **Quality Engineering Decision Records (QEDRs)** para as decisões-chave:

```markdown
## QEDR-001: Estratégia de Testes — Pirâmide vs Troféu

### Status
Aceito

### Contexto
A Digital Wallet precisa de uma estratégia de testes escalável. O debate
pirâmide-de-testes vs troféu-de-testes (Kent C. Dodds) precisa de uma decisão
para guiar o time na alocação de esforço em testes.

### Decisão
Adotar a **Pirâmide de Testes clássica** com adaptação para contexto financeiro:

| Camada | Proporção | Justificativa |
|--------|-----------|---------------|
| Unit | ~60% | Lógica de domínio financeiro é complexa e pode ser isolada |
| Integration | ~25% | Persistência e APIs precisam de verificação real (Testcontainers) |
| Contract | ~10% | Comunicação entre serviços deve ser validada |
| E2E | ~5% | Smoke tests para fluxos críticos (happy path) |

### Consequências
- **Positiva:** Feedback rápido (suite unit < 30s)
- **Positiva:** Cobertura financeira profunda
- **Negativa:** E2E limitados podem perder bugs de integração entre UI e API
- **Mitigação:** Contract tests compensam o gap entre integration e E2E
```

```markdown
## QEDR-002: SLO Target — 99.9% Availability vs 99.99%

### Status
Aceito

### Contexto
Definir o target de disponibilidade impacta diretamente o error budget
e a velocidade de desenvolvimento.

### Decisão
Adotar **99.9% (three nines)** para a primeira versão:
- Error budget: ~43 min/mês de indisponibilidade permitida
- Permite ritmo de deploy razoável (>= 1x/dia)
- Revisitar após 6 meses com dados reais

### Alternativa Rejeitada
99.99% (four nines) — apenas ~4.3 min/mês. Exigiria:
- Redundância total multi-região
- Feature freeze frequente
- Investimento em infra incompatível com fase atual do produto

### Consequências
- **Positiva:** Error budget suficiente para experimentação
- **Positiva:** Alinhado com a fase do produto (MVP → growth)
- **Negativa:** Clientes enterprise podem exigir mais disponibilidade
- **Mitigação:** Oferecer tier premium com SLA mais restrito no futuro
```

```markdown
## QEDR-003: Ferramenta de Performance — k6 vs Gatling vs JMeter

### Status
Aceito

### Contexto
Precisamos de uma ferramenta para testes de carga que funcione para
ambas as stacks (Go e Java) e se integre ao CI/CD.

### Decisão
Adotar **k6** como ferramenta primária:
- Scripts em JavaScript (linguagem conhecida pelo time)
- Excelente integração com CI/CD (Docker, GitHub Actions)
- Output para Prometheus/Grafana (stack já em uso)
- Lightweight (binário Go, baixo consumo de recursos)

### Alternativas
- **Gatling:** Mais poderoso para cenários complexos, mas scripts em Scala
  adicionam barreira de entrada
- **JMeter:** Amplamente usado, mas GUI-based e XML dificultam versionamento
- **vegeta:** Simples, mas limitado para cenários avançados

### Consequências
- **Positiva:** Time unified em uma ferramenta
- **Positiva:** CI/CD integration nativa
- **Negativa:** k6 não suporta protocolos como gRPC nativamente (requer extensão)
```

- Cada QEDR deve conter: **Status, Contexto, Decisão, Alternativas, Consequências**
- Mínimo **5 QEDRs** cobrindo decisões dos Levels 0–6

**Critérios de aceite:**

- [ ] ≥ 5 QEDRs documentados
- [ ] Cada QEDR tem Status, Contexto, Decisão, Consequências
- [ ] QEDRs cobrem decisões de pelo menos 4 pilares diferentes
- [ ] Alternativas rejeitadas documentadas com justificativa
- [ ] Formato consistente (Markdown)
- [ ] Diretório: `decisions/`

---

### Desafio 7.5 — Relatório Final de Portfólio

**Contexto:** O objetivo final do programa é produzir uma peça de portfólio
que demonstre expertise em Quality Engineering para entrevistas Staff/Principal.

**Requisitos:**

- Produzir **relatório técnico final** que consolida toda a jornada:

```markdown
# Digital Wallet — Quality Engineering Report

## Executive Summary
Implementação completa de Quality Engineering para um sistema financeiro
(Digital Wallet) em 5 stacks: Go (Gin), Spring Boot, Quarkus, Micronaut, Jakarta EE.

### Resultados-chave
| Métrica | Valor |
|---------|-------|
| Unit Test Coverage | 87% (Go) / 85% (Java avg) |
| Mutation Score | 73% (PIT) |
| SAST Findings (High) | 0 |
| SCA CVEs (Critical) | 0 |
| P99 Latency (Deposit) | 142ms |
| SLO Compliance (30d) | 99.95% |
| Pipeline Duration | 22 min |
| DORA — Deployment Frequency | 3x/day |
| DORA — Lead Time | 45 min |
| DORA — Change Failure Rate | 4.2% |
| DORA — MTTR | 18 min |

## 1. Testing Strategy
[Resumo das decisões e implementações do Level 1]
- Pirâmide de testes com proporção 60/25/10/5
- Testcontainers para todos os integration tests
- Contract tests via Pact

## 2. Test Automation Patterns
[Resumo do Level 2]
- Test Data Builders para entidades financeiras
- Custom Assertions para Money/Transaction/Wallet
- API test helpers com RestAssured/httptest

## 3. Performance Testing
[Resumo do Level 3]
- k6 com cenários: baseline, load, stress, spike, soak
- SLOs: P99 ≤ 300ms (deposit), P99 ≤ 500ms (transfer)
- Profiling com pprof (Go) / async-profiler (Java)

## 4. Security Testing
[Resumo do Level 4]
- STRIDE threat model para operações financeiras
- SAST: gosec + SpotBugs + Semgrep
- SCA: govulncheck + OWASP Dependency-Check
- DAST: OWASP ZAP integrado ao CI
- Secret scanning: gitleaks em pre-commit e CI

## 5. Code Review & Quality Gates
[Resumo do Level 5]
- Checklist de review domain-specific (P0–P4)
- 7 quality gates no pipeline CI/CD
- Mutation testing com PIT (threshold 70%)
- Review SLAs: P0 < 4h, P1 < 8h, P2 < 24h

## 6. Observability
[Resumo do Level 6]
- Structured logging (slog/SLF4J) com mascaramento de PII
- RED metrics + business KPIs via Micrometer/Prometheus
- Distributed tracing com OpenTelemetry
- SLOs com error budget e multi-window burn rate alerts
- Service Overview dashboard (Grafana)

## 7. Pipeline Quality
[Resumo do Level 7]
- 8-stage pipeline (lint → unit → security → integration → DAST → perf → deploy → report)
- Tempo total: 22 min com paralelismo
- Quality Maturity: 3.5/5.0 (overall)
- Production Readiness: 23/25 checks passed

## Comparative Analysis (Go vs Java)
| Aspecto | Go (Gin) | Java (Spring Boot) | Java (Quarkus) |
|---------|----------|-------------------|----------------|
| Test Suite Speed | 8s | 45s | 22s |
| Coverage Tool | go cover | JaCoCo | JaCoCo |
| SAST | gosec | SpotBugs + Semgrep | SpotBugs + Semgrep |
| Startup Time | 50ms | 3.2s | 0.8s (native) |
| Docker Image | 18MB | 280MB | 85MB (native) |
| Memory (idle) | 12MB | 185MB | 45MB (native) |

## Lessons Learned
1. **Shift-left funciona** — bugs encontrados em unit/integration são 10x mais baratos
2. **Mutation testing revela gaps** — coverage 80% com mutation 50% = falsa segurança
3. **SLOs alinham o time** — error budget remove debates subjetivos sobre "rápido o suficiente"
4. **Observabilidade > monitoramento** — structured logging + tracing + métricas = debug em minutos
5. **Security is a feature** — SAST/SCA/DAST no CI previne > 90% das vulns conhecidas

## Architecture Decisions
- [QEDR-001: Pirâmide vs Troféu](decisions/QEDR-001-test-strategy.md)
- [QEDR-002: SLO Target 99.9%](decisions/QEDR-002-slo-target.md)
- [QEDR-003: k6 vs Gatling vs JMeter](decisions/QEDR-003-perf-tool.md)
- [QEDR-004: Structured Logging Strategy](decisions/QEDR-004-logging.md)
- [QEDR-005: Security Scanning Pipeline](decisions/QEDR-005-security-pipeline.md)
```

- O relatório deve incluir:
  - **Dados reais** (coverage, mutation scores, latências, SLO compliance)
  - **Gráficos/tabelas** comparativos entre stacks
  - **Lessons learned** baseados na experiência
  - **QEDRs** referenciados
  - **Reprodutibilidade:** instruções para reproduzir os resultados

```bash
# Instruções de reprodução
git clone https://github.com/<user>/digital-wallet-quality.git
cd digital-wallet-quality

# Rodar todo o pipeline
make quality-all  # ou: ./scripts/run-quality-pipeline.sh

# Gerar relatório
make quality-report

# Resultados em:
# - reports/quality-report.md
# - reports/quality-maturity.json
# - reports/production-readiness.json
# - reports/benchmark-comparison.json
```

**Critérios de aceite:**

- [ ] Relatório cobre todos os 6 pilares de qualidade
- [ ] Dados comparativos entre pelo menos Go + 2 Java stacks
- [ ] Lessons learned baseados em experiência real
- [ ] ≥ 5 QEDRs referenciados
- [ ] Instruções de reprodução (Makefile ou script)
- [ ] Relatório publicável (portfólio GitHub, blog, ou PDF)

---

## Definição de Pronto (DoD) — Capstone

- [ ] Pipeline CI/CD com 8 stages e quality gates automatizados
- [ ] Quality Maturity Assessment com 6 pilares × 5 níveis
- [ ] Production Readiness Checklist com 20+ checks automatizados
- [ ] ≥ 5 Quality Engineering Decision Records
- [ ] Relatório final com dados reais e análise comparativa
- [ ] Todo o código versionado e reprodutível
- [ ] Commit semântico: `feat(level-7): implement capstone quality engineering pipeline`

---

## Checklist

- [ ] Pipeline: 8 stages com gates de falha
- [ ] Pipeline: roda para Go + 4 Java stacks
- [ ] Pipeline: tempo total < 30 min com paralelismo
- [ ] Maturity: modelo com 6 pilares × 5 níveis
- [ ] Maturity: script de self-assessment funcional
- [ ] Maturity: relatório JSON com scores e gaps
- [ ] PRR: checklist 5+ categorias, 20+ checks
- [ ] PRR: script de validação automatizado
- [ ] PRR: flag Ready true/false
- [ ] QEDRs: ≥ 5 documentados (4+ pilares)
- [ ] Relatório: cobre todos os pilares
- [ ] Relatório: dados comparativos (Go vs Java)
- [ ] Relatório: lessons learned reais
- [ ] Relatório: instruções de reprodução
- [ ] Relatório: publicável como portfólio
