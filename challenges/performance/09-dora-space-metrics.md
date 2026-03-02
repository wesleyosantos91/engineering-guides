# Level 9 — DORA Metrics & SPACE Framework

> **Objetivo:** Implementar coleta automatizada das 4 métricas DORA (Deployment Frequency, Lead Time for Changes, Change Failure Rate, Mean Time to Recovery), construir dashboards de classificação (Elite/High/Medium/Low), instrumentar dimensões do SPACE Framework (Satisfaction, Performance, Activity, Communication, Efficiency) e integrar tudo em um sistema de melhoria contínua de engineering performance.

---

## Objetivo de Aprendizado

- Entender as 4+1 métricas DORA e como se relacionam com capabilities técnicas
- Implementar coleta automatizada de deployment events a partir do pipeline CI/CD
- Calcular Lead Time for Changes com breakdown (coding → review → merge → deploy)
- Correlacionar deploys com incidentes para derivar Change Failure Rate
- Medir MTTR com decomposição (MTTD, MTTE, MTTF, MTTV)
- Classificar equipe nos benchmarks DORA (Elite, High, Medium, Low)
- Aplicar o SPACE Framework para capturar dimensões além do delivery pipeline
- Instrumentar Efficiency & Flow (build time, review wait, focus time)
- Construir dashboards Grafana combinando DORA + SPACE
- Evitar anti-patterns de métricas (Goodhart's Law, gaming, individual ranking)
- Integrar DORA metrics com SLOs e error budgets (conexão com Level 7)

---

## Escopo Funcional

### Arquitetura do Sistema de Métricas

```
┌──────────────────────────────────────────────────────────────────┐
│                    DORA & SPACE METRICS SYSTEM                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────┐                                              │
│  │   CI/CD Pipeline │──deploy events──┐                            │
│  │ (GitHub Actions / │                 │                            │
│  │  GitLab CI)       │                 ▼                            │
│  └──────────────────┘      ┌────────────────────┐                  │
│                            │   DORA Collector    │                  │
│  ┌──────────────────┐      │     Service         │                  │
│  │   Git / VCS      │──PR──│                     │                  │
│  │ (GitHub API)     │ data │  POST /events/deploy│                  │
│  └──────────────────┘      │  POST /events/pr    │                  │
│                            │  POST /events/       │                  │
│  ┌──────────────────┐      │       incident      │                  │
│  │ Incident Tracker │──inc─│                     │                  │
│  │ (PagerDuty/      │ data │  GET /metrics/dora  │                  │
│  │  manual/label)   │      │  GET /metrics/space │                  │
│  └──────────────────┘      └────────┬───────────┘                  │
│                                     │                              │
│                            ┌────────▼───────────┐                  │
│                            │    Prometheus       │                  │
│                            │  (scrape /metrics)  │                  │
│                            └────────┬───────────┘                  │
│                                     │                              │
│                            ┌────────▼───────────┐                  │
│                            │     Grafana         │                  │
│                            │  DORA Dashboard     │                  │
│                            │  SPACE Dashboard    │                  │
│                            │  Trend Analysis     │                  │
│                            └────────────────────┘                  │
│                                                                    │
│  Endpoints do DORA Collector:                                      │
│  POST /api/v1/events/deployments     (CI/CD webhook)              │
│  POST /api/v1/events/pull-requests   (VCS webhook)                │
│  POST /api/v1/events/incidents       (Incident webhook)           │
│  GET  /api/v1/metrics/dora           (DORA summary)               │
│  GET  /api/v1/metrics/dora/trend     (Historical trend)           │
│  GET  /api/v1/metrics/space          (SPACE dimensions)           │
│  GET  /metrics                       (Prometheus scrape)          │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    DIGITAL WALLET — SYSTEM UNDER TEST             │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │  API     │  │  Order   │  │  Payment │  │ Inventory│          │
│  │ Gateway  │→ │ Service  │→ │ Service  │→ │ Service  │          │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘          │
│                                                                    │
│  Cada serviço tem seu CI/CD pipeline → webhook → DORA Collector   │
│  SLOs definidos no Level 7 alimentam a 5ª métrica (Reliability)   │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Fluxo de Coleta — Deployment Frequency

```
CI/CD Pipeline (deploy to production)
  │
  ├── Pipeline completa com sucesso
  │   └── Webhook dispara para DORA Collector
  │       POST /api/v1/events/deployments
  │       {
  │         "service": "order-service",
  │         "version": "v1.5.0",
  │         "sha": "abc123",
  │         "environment": "production",
  │         "deployed_at": "2026-01-15T14:30:00Z",
  │         "status": "success",
  │         "pipeline_id": "run-12345",
  │         "triggered_by": "merge_to_main"
  │       }
  │
  └── DORA Collector:
      ├── Persiste evento no banco
      ├── Incrementa counter Prometheus: dora_deployments_total{service, status}
      └── Recalcula DF (rolling 7d e 30d)
```

### Fluxo de Coleta — Lead Time for Changes

```
PR merged (webhook)
  │
  ├── GitHub/GitLab webhook dispara
  │   POST /api/v1/events/pull-requests
  │   {
  │     "service": "order-service",
  │     "pr_number": 127,
  │     "first_commit_at": "2026-01-14T09:00:00Z",
  │     "pr_opened_at": "2026-01-14T16:00:00Z",
  │     "pr_approved_at": "2026-01-15T10:00:00Z",
  │     "merged_at": "2026-01-15T10:30:00Z"
  │   }
  │
  └── Quando deploy acontece (match por SHA/version):
      ├── Lead Time = deployed_at - first_commit_at
      ├── Breakdown:
      │   ├── coding_time  = pr_opened_at - first_commit_at
      │   ├── review_time  = pr_approved_at - pr_opened_at
      │   ├── merge_time   = merged_at - pr_approved_at
      │   └── pipeline_time = deployed_at - merged_at
      └── Observa histogram: dora_lead_time_seconds{service}
```

### Fluxo de Coleta — Change Failure Rate

```
Incidente detectado
  │
  ├── Webhook ou label "incident" adicionado
  │   POST /api/v1/events/incidents
  │   {
  │     "service": "order-service",
  │     "incident_id": "INC-042",
  │     "severity": "sev2",
  │     "detected_at": "2026-01-15T15:00:00Z",
  │     "cause": "deployment",
  │     "related_deploy_sha": "abc123"
  │   }
  │
  └── Correlação:
      ├── Buscar deploy correspondente (same service + SHA + janela 24h)
      ├── Marcar deploy como "caused_failure"
      ├── Incrementar: dora_change_failures_total{service, failure_type}
      └── CFR = failures / total_deploys (rolling 30d)
```

### Fluxo de Coleta — MTTR

```
Incidente lifecycle
  │
  ├── OPENED:  detected_at
  │   └── dora_incident_opened_total++
  │
  ├── ACKNOWLEDGED: acknowledged_at
  │   └── MTTE = acknowledged_at - detected_at
  │
  ├── RESOLVED: resolved_at
  │   └── MTTF = resolved_at - acknowledged_at
  │
  └── RECOVERED: sli_recovered_at
      ├── MTTV = sli_recovered_at - resolved_at
      ├── MTTR = sli_recovered_at - detected_at
      └── Observa histogram: dora_incident_recovery_seconds{service, severity}
```

---

## Escopo Técnico

### Modelo de Dados

```sql
-- Tabela de eventos de deploy
CREATE TABLE deployment_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service         VARCHAR(100) NOT NULL,
    version         VARCHAR(50),
    sha             VARCHAR(40) NOT NULL,
    environment     VARCHAR(20) NOT NULL DEFAULT 'production',
    deployed_at     TIMESTAMPTZ NOT NULL,
    status          VARCHAR(20) NOT NULL, -- success, failure, rollback
    pipeline_id     VARCHAR(100),
    triggered_by    VARCHAR(50),
    caused_failure  BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Tabela de eventos de PR / Lead Time
CREATE TABLE pull_request_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service         VARCHAR(100) NOT NULL,
    pr_number       INTEGER NOT NULL,
    first_commit_at TIMESTAMPTZ NOT NULL,
    pr_opened_at    TIMESTAMPTZ,
    pr_approved_at  TIMESTAMPTZ,
    merged_at       TIMESTAMPTZ NOT NULL,
    deployed_at     TIMESTAMPTZ,     -- preenchido quando deploy acontece
    deploy_sha      VARCHAR(40),
    lead_time_secs  BIGINT,          -- calculado: deployed_at - first_commit_at
    coding_time_secs BIGINT,
    review_time_secs BIGINT,
    merge_time_secs  BIGINT,
    pipeline_time_secs BIGINT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Tabela de incidentes
CREATE TABLE incident_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    service         VARCHAR(100) NOT NULL,
    incident_id     VARCHAR(50) UNIQUE NOT NULL,
    severity        VARCHAR(10) NOT NULL, -- sev1, sev2, sev3
    cause           VARCHAR(50),          -- deployment, infra, external
    detected_at     TIMESTAMPTZ NOT NULL,
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    sli_recovered_at TIMESTAMPTZ,
    related_deploy_id UUID REFERENCES deployment_events(id),
    mttr_seconds    BIGINT,               -- calculado
    mttd_seconds    BIGINT,
    mtte_seconds    BIGINT,
    mttf_seconds    BIGINT,
    mttv_seconds    BIGINT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Tabela de SPACE survey responses (Satisfaction + Communication)
CREATE TABLE space_survey_responses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team            VARCHAR(100) NOT NULL,
    quarter         VARCHAR(7) NOT NULL,  -- 2026-Q1
    dimension       VARCHAR(20) NOT NULL,  -- satisfaction, communication
    question_key    VARCHAR(100) NOT NULL,
    score           NUMERIC(3,1) NOT NULL, -- 1.0 - 5.0
    respondent_hash VARCHAR(64),           -- anonym hash
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Índices para queries de métricas
CREATE INDEX idx_deploy_service_time ON deployment_events(service, deployed_at);
CREATE INDEX idx_deploy_sha ON deployment_events(sha);
CREATE INDEX idx_pr_service_merged ON pull_request_events(service, merged_at);
CREATE INDEX idx_incident_service_detected ON incident_events(service, detected_at);
CREATE INDEX idx_survey_team_quarter ON space_survey_responses(team, quarter);
```

### Prometheus Metrics Expostas pelo DORA Collector

| Metric Name | Type | Labels | Descrição |
|---|---|---|---|
| `dora_deployments_total` | Counter | `service`, `status` | Total de deploys (success/failure/rollback) |
| `dora_lead_time_seconds` | Histogram | `service` | Lead time do commit ao deploy (buckets: 1h, 4h, 8h, 1d, 3d, 7d, 30d) |
| `dora_lead_time_coding_seconds` | Histogram | `service` | Coding phase do lead time |
| `dora_lead_time_review_seconds` | Histogram | `service` | Review phase do lead time |
| `dora_lead_time_pipeline_seconds` | Histogram | `service` | Pipeline phase do lead time |
| `dora_change_failures_total` | Counter | `service`, `failure_type` | Deploys que causaram falha |
| `dora_incident_recovery_seconds` | Histogram | `service`, `severity` | MTTR (detection → recovery) |
| `dora_incident_engage_seconds` | Histogram | `service`, `severity` | MTTE (detection → acknowledge) |
| `space_survey_score` | Gauge | `team`, `dimension`, `quarter` | Score médio por dimensão SPACE |
| `space_review_turnaround_seconds` | Histogram | `service` | Tempo entre PR aberta e first review |
| `space_ci_build_seconds` | Histogram | `service` | Tempo de build no CI |
| `space_test_suite_seconds` | Histogram | `service` | Tempo de execução dos testes no CI |

### Recording Rules (Prometheus / Grafana)

```yaml
# prometheus/rules/dora-rules.yml
groups:
  - name: dora.metrics
    interval: 5m
    rules:
      # Deployment Frequency — deploys/day (rolling 7d)
      - record: dora:deployment_frequency:rate7d
        expr: |
          sum by (service) (
            increase(dora_deployments_total{status="success"}[7d])
          ) / 7

      # Lead Time p50 (rolling 30d) in hours
      - record: dora:lead_time:p50:30d
        expr: |
          histogram_quantile(0.50,
            sum by (service, le) (
              rate(dora_lead_time_seconds_bucket[30d])
            )
          ) / 3600

      # Lead Time p90 in hours
      - record: dora:lead_time:p90:30d
        expr: |
          histogram_quantile(0.90,
            sum by (service, le) (
              rate(dora_lead_time_seconds_bucket[30d])
            )
          ) / 3600

      # Change Failure Rate (rolling 30d)
      - record: dora:change_failure_rate:30d
        expr: |
          sum by (service) (increase(dora_change_failures_total[30d]))
          /
          sum by (service) (increase(dora_deployments_total{status="success"}[30d]))

      # MTTR p50 in minutes (rolling 30d)
      - record: dora:mttr:p50:30d
        expr: |
          histogram_quantile(0.50,
            sum by (service, le) (
              rate(dora_incident_recovery_seconds_bucket[30d])
            )
          ) / 60

      # DORA Classification — DF
      - record: dora:classification:deployment_frequency
        expr: |
          (dora:deployment_frequency:rate7d >= 1) * 4
          + (dora:deployment_frequency:rate7d >= 0.143 < 1) * 3
          + (dora:deployment_frequency:rate7d >= 0.033 < 0.143) * 2
          + (dora:deployment_frequency:rate7d < 0.033) * 1
```

### Ferramentas por Stack

| Aspecto | Go (Gin) | Spring Boot | Quarkus | Micronaut | Jakarta EE |
|---|---|---|---|---|---|
| **HTTP Server** | Gin + handlers | Spring Web MVC | RESTEasy Reactive | Micronaut HTTP | JAX-RS |
| **Database** | pgx + sqlc | Spring Data JPA | Hibernate Reactive | Micronaut Data | JPA |
| **Prometheus** | prometheus/client_golang | Micrometer + Counter/Histogram | SmallRye Metrics | Micrometer (built-in) | MicroProfile Metrics |
| **Webhook receiver** | Gin handler + HMAC validation | Spring Web + `@PostMapping` | `@POST` + Bean Validation | `@Post` + Validation | `@POST` + Bean Validation |
| **Scheduler** | `time.Ticker` / cron lib | `@Scheduled` | `@Scheduled` | `@Scheduled` | `@Schedule` (EJB Timer) |
| **JSON parsing** | encoding/json / sonic | Jackson | Jackson / JSON-B | Jackson / Serde | JSON-B |
| **Config** | envconfig / viper | application.yml | application.properties | application.yml | microprofile-config |
| **Testing** | testing + testcontainers-go | JUnit 5 + Testcontainers | JUnit 5 + Testcontainers | JUnit 5 + Testcontainers | JUnit 5 + Testcontainers |

---

## Critérios de Aceite

### DORA Metrics Collection

- [ ] DORA Collector Service recebe webhooks de deploy, PR e incident via REST API
- [ ] Deployment Frequency calculada: deploys/dia com rolling window (7d e 30d)
- [ ] Lead Time calculado: commit → deploy com breakdown (coding, review, merge, pipeline)
- [ ] Lead Time reportado como p50 e p90 (não média aritmética)
- [ ] Change Failure Rate calculado: correlação deploy ↔ incidente dentro de janela de 24h
- [ ] MTTR calculado com decomposição: MTTD + MTTE + MTTF + MTTV
- [ ] Classificação DORA (Elite/High/Medium/Low) derivada para cada métrica
- [ ] Classificação overall baseada na pior métrica individual

### SPACE Dimensions

- [ ] **Activity**: DF e PRs merged coletados automaticamente (overlap com DORA)
- [ ] **Efficiency**: CI build time e test suite time coletados do pipeline
- [ ] **Efficiency**: PR review turnaround time calculado (PR opened → first review)
- [ ] **Communication**: Cross-team PR reviews contabilizados
- [ ] **Satisfaction**: API para receber survey responses (POST /api/v1/surveys)
- [ ] **Performance**: SLO compliance integrado (reuso do Level 7 ou cálculo próprio)

### Dashboards

- [ ] Dashboard DORA no Grafana: 4 métricas com gauge (valor atual) + sparkline (trend 90d)
- [ ] Dashboard DORA: classification matrix (Elite/High/Medium/Low) para cada métrica
- [ ] Dashboard DORA: Lead Time breakdown (bar chart horizontal — coding vs review vs pipeline)
- [ ] Dashboard SPACE: radar chart (5 dimensões) por equipe
- [ ] Dashboard SPACE: trend trimestral (Satisfaction + Efficiency)
- [ ] Drill-down: serviço individual com histórico de deploys e incidentes

### API & Data

- [ ] REST API com validação de input (request body schemas)
- [ ] HMAC signature validation nos webhooks (segurança)
- [ ] Endpoint GET /api/v1/metrics/dora retorna JSON com as 4 métricas + classification
- [ ] Endpoint GET /api/v1/metrics/dora/trend retorna dados históricos (por semana/mês)
- [ ] Endpoint /metrics expõe Prometheus counters, histograms e gauges
- [ ] Dados persistidos em PostgreSQL com migrations

### Anti-Patterns Awareness

- [ ] README documenta: "DORA metrics são para equipes, não indivíduos"
- [ ] README documenta: relação com Goodhart's Law e como evitar gaming
- [ ] Nenhuma métrica individual (por dev) é coletada ou exibida

---

## Definição de Pronto (DoD)

- [ ] Todos os critérios de aceite ✅
- [ ] DORA Collector rodando com webhooks simulados (scripts de seed data)
- [ ] 4 métricas DORA calculadas e classificadas corretamente
- [ ] Pelo menos 2 dimensões SPACE instrumentadas automaticamente (Activity + Efficiency)
- [ ] Pelo menos 1 dimensão SPACE via survey (Satisfaction)
- [ ] Dashboards Grafana funcionais com dados de pelo menos 30 dias simulados
- [ ] Testes de integração para cálculo de cada métrica
- [ ] Script de seed que gera dados realistas (mix de elite e medianos)
- [ ] Documentação: decisões de design, modelo de dados, diagrama de arquitetura
- [ ] Commit: `feat(level-9): implement DORA metrics & SPACE framework collector`

---

## Checklist

### Go (Gin)

- [ ] `cmd/dora-collector/main.go` — Server setup, routes, graceful shutdown
- [ ] `internal/handler/deployment.go` — POST /api/v1/events/deployments handler
- [ ] `internal/handler/pullrequest.go` — POST /api/v1/events/pull-requests handler
- [ ] `internal/handler/incident.go` — POST /api/v1/events/incidents handler
- [ ] `internal/handler/metrics.go` — GET /api/v1/metrics/dora, /dora/trend, /space
- [ ] `internal/handler/survey.go` — POST /api/v1/surveys handler
- [ ] `internal/service/dora_calculator.go` — Lógica de cálculo DF, LT, CFR, MTTR
- [ ] `internal/service/dora_classifier.go` — Classificação Elite/High/Medium/Low
- [ ] `internal/service/space_calculator.go` — Cálculo das dimensões SPACE
- [ ] `internal/service/lead_time.go` — Breakdown de Lead Time (coding, review, pipeline)
- [ ] `internal/service/correlator.go` — Correlação deploy ↔ incidente
- [ ] `internal/repository/deployment_repo.go` — CRUD + queries de DF
- [ ] `internal/repository/pr_repo.go` — CRUD + queries de Lead Time
- [ ] `internal/repository/incident_repo.go` — CRUD + queries de MTTR
- [ ] `internal/repository/survey_repo.go` — CRUD de survey responses
- [ ] `internal/middleware/hmac.go` — HMAC signature validation para webhooks
- [ ] `internal/platform/prometheus/metrics.go` — Registro de counters e histograms
- [ ] `migrations/` — SQL migrations (schema acima)
- [ ] `scripts/seed_data.go` — Gerador de dados realistas (30-90 dias)
- [ ] `test/integration/dora_test.go` — Testes de integração com Testcontainers

### Spring Boot

- [ ] `pom.xml` — deps: spring-web, spring-data-jpa, micrometer-registry-prometheus
- [ ] `application.yml` — datasource, server port, HMAC secret
- [ ] `web/controller/DeploymentEventController.java` — `@PostMapping("/api/v1/events/deployments")`
- [ ] `web/controller/PullRequestEventController.java` — `@PostMapping("/api/v1/events/pull-requests")`
- [ ] `web/controller/IncidentEventController.java` — `@PostMapping("/api/v1/events/incidents")`
- [ ] `web/controller/DoraMetricsController.java` — `@GetMapping("/api/v1/metrics/dora")`
- [ ] `web/controller/SpaceMetricsController.java` — `@GetMapping("/api/v1/metrics/space")`
- [ ] `web/controller/SurveyController.java` — `@PostMapping("/api/v1/surveys")`
- [ ] `domain/service/DoraCalculatorService.java` — Cálculo DF, LT, CFR, MTTR
- [ ] `domain/service/DoraClassifierService.java` — Classificação Elite/High/Medium/Low
- [ ] `domain/service/SpaceCalculatorService.java` — Cálculo dimensões SPACE
- [ ] `domain/service/LeadTimeService.java` — Breakdown de Lead Time
- [ ] `domain/service/DeployIncidentCorrelatorService.java` — Correlação deploy-incidente
- [ ] `domain/model/DeploymentEvent.java` — JPA Entity
- [ ] `domain/model/PullRequestEvent.java` — JPA Entity
- [ ] `domain/model/IncidentEvent.java` — JPA Entity
- [ ] `domain/model/SpaceSurveyResponse.java` — JPA Entity
- [ ] `domain/model/DoraMetrics.java` — Value Object (DF, LT, CFR, MTTR + classification)
- [ ] `infrastructure/repository/DeploymentEventRepository.java` — Spring Data JPA
- [ ] `infrastructure/security/HmacFilter.java` — OncePerRequestFilter para HMAC validation
- [ ] `infrastructure/config/PrometheusConfig.java` — Custom Micrometer metrics
- [ ] `src/main/resources/db/migration/V1__create_dora_schema.sql` — Flyway migration
- [ ] `src/test/java/.../DoraCalculatorServiceTest.java` — Unit tests
- [ ] `src/test/java/.../DoraIntegrationTest.java` — Integration tests com Testcontainers

### Quarkus

- [ ] `pom.xml` — deps: quarkus-rest, quarkus-hibernate-orm-panache, quarkus-micrometer-registry-prometheus
- [ ] `application.properties` — datasource, HMAC config
- [ ] `@Path("/api/v1/events")` — Deployment, PR, Incident resources
- [ ] `@Path("/api/v1/metrics")` — DORA, SPACE resources
- [ ] Panache entities + repositories
- [ ] `@ContainerRequestFilter` para HMAC validation
- [ ] `@QuarkusTest` com Testcontainers

### Micronaut

- [ ] `pom.xml` — deps: micronaut-http-server, micronaut-data-jdbc, micronaut-micrometer-registry-prometheus
- [ ] `application.yml` — datasource, HMAC config
- [ ] `@Controller("/api/v1/events")` — Deployment, PR, Incident controllers
- [ ] `@Controller("/api/v1/metrics")` — DORA, SPACE controllers
- [ ] `@MappedEntity` + `@JdbcRepository` para persistência
- [ ] `HttpServerFilter` para HMAC validation
- [ ] `@MicronautTest` com Testcontainers

### Jakarta EE

- [ ] `pom.xml` — deps: jakarta.ws.rs, jakarta.persistence, microprofile-metrics
- [ ] `@Path("/api/v1/events")` — JAX-RS resources
- [ ] `@Path("/api/v1/metrics")` — DORA, SPACE resources
- [ ] JPA entities + DAO pattern
- [ ] `ContainerRequestFilter` para HMAC validation
- [ ] `@RegistryType` para MicroProfile Metrics counters/histograms
- [ ] Arquillian / Testcontainers integration tests

---

## Tarefas Sugeridas por Stack

### Go — DORA Calculator

```go
// internal/service/dora_calculator.go
package service

import (
    "context"
    "time"
)

// DoraClassification represents DORA benchmark levels
type DoraClassification string

const (
    ClassElite  DoraClassification = "elite"
    ClassHigh   DoraClassification = "high"
    ClassMedium DoraClassification = "medium"
    ClassLow    DoraClassification = "low"
)

// DoraMetrics holds the 4 key metrics with classifications
type DoraMetrics struct {
    DeploymentFrequency DeploymentFrequencyMetric `json:"deployment_frequency"`
    LeadTime            LeadTimeMetric            `json:"lead_time"`
    ChangeFailureRate   ChangeFailureRateMetric   `json:"change_failure_rate"`
    MeanTimeToRecovery  MTTRMetric                `json:"mean_time_to_recovery"`
    OverallClass        DoraClassification        `json:"overall_classification"`
    Period              string                    `json:"period"`
}

type DeploymentFrequencyMetric struct {
    DeploysPerDay   float64            `json:"deploys_per_day"`
    TotalDeploys    int                `json:"total_deploys"`
    Classification  DoraClassification `json:"classification"`
}

type LeadTimeMetric struct {
    P50Hours        float64            `json:"p50_hours"`
    P90Hours        float64            `json:"p90_hours"`
    Breakdown       LeadTimeBreakdown  `json:"breakdown"`
    Classification  DoraClassification `json:"classification"`
}

type LeadTimeBreakdown struct {
    CodingP50Hours  float64 `json:"coding_p50_hours"`
    ReviewP50Hours  float64 `json:"review_p50_hours"`
    MergeP50Hours   float64 `json:"merge_p50_hours"`
    PipelineP50Hours float64 `json:"pipeline_p50_hours"`
}

type ChangeFailureRateMetric struct {
    Rate            float64            `json:"rate"`
    FailedDeploys   int                `json:"failed_deploys"`
    TotalDeploys    int                `json:"total_deploys"`
    Classification  DoraClassification `json:"classification"`
}

type MTTRMetric struct {
    P50Minutes      float64            `json:"p50_minutes"`
    P90Minutes      float64            `json:"p90_minutes"`
    Breakdown       MTTRBreakdown      `json:"breakdown"`
    Classification  DoraClassification `json:"classification"`
}

type MTTRBreakdown struct {
    MTTDP50Minutes float64 `json:"mttd_p50_minutes"`
    MTTEP50Minutes float64 `json:"mtte_p50_minutes"`
    MTTFP50Minutes float64 `json:"mttf_p50_minutes"`
    MTTVP50Minutes float64 `json:"mttv_p50_minutes"`
}

type DoraCalculator struct {
    deployRepo  DeploymentRepository
    prRepo      PullRequestRepository
    incidentRepo IncidentRepository
}

func NewDoraCalculator(
    deployRepo DeploymentRepository,
    prRepo PullRequestRepository,
    incidentRepo IncidentRepository,
) *DoraCalculator {
    return &DoraCalculator{
        deployRepo:   deployRepo,
        prRepo:       prRepo,
        incidentRepo: incidentRepo,
    }
}

func (c *DoraCalculator) Calculate(ctx context.Context, service string, window time.Duration) (*DoraMetrics, error) {
    now := time.Now()
    from := now.Add(-window)

    // 1. Deployment Frequency
    deploys, err := c.deployRepo.CountSuccessful(ctx, service, from, now)
    if err != nil {
        return nil, err
    }
    days := window.Hours() / 24
    deploysPerDay := float64(deploys) / days
    dfClass := classifyDF(deploysPerDay)

    // 2. Lead Time
    ltP50, ltP90, err := c.prRepo.LeadTimePercentiles(ctx, service, from, now)
    if err != nil {
        return nil, err
    }
    ltBreakdown, err := c.prRepo.LeadTimeBreakdown(ctx, service, from, now)
    if err != nil {
        return nil, err
    }
    ltClass := classifyLT(ltP50)

    // 3. Change Failure Rate
    failedDeploys, err := c.deployRepo.CountCausedFailure(ctx, service, from, now)
    if err != nil {
        return nil, err
    }
    cfr := 0.0
    if deploys > 0 {
        cfr = float64(failedDeploys) / float64(deploys)
    }
    cfrClass := classifyCFR(cfr)

    // 4. MTTR
    mttrP50, mttrP90, err := c.incidentRepo.MTTRPercentiles(ctx, service, from, now)
    if err != nil {
        return nil, err
    }
    mttrBreakdown, err := c.incidentRepo.MTTRBreakdown(ctx, service, from, now)
    if err != nil {
        return nil, err
    }
    mttrClass := classifyMTTR(mttrP50)

    // Overall: worst classification wins
    overall := worstClassification(dfClass, ltClass, cfrClass, mttrClass)

    return &DoraMetrics{
        DeploymentFrequency: DeploymentFrequencyMetric{
            DeploysPerDay: deploysPerDay, TotalDeploys: deploys, Classification: dfClass,
        },
        LeadTime: LeadTimeMetric{
            P50Hours: ltP50.Hours(), P90Hours: ltP90.Hours(),
            Breakdown: *ltBreakdown, Classification: ltClass,
        },
        ChangeFailureRate: ChangeFailureRateMetric{
            Rate: cfr, FailedDeploys: failedDeploys, TotalDeploys: deploys,
            Classification: cfrClass,
        },
        MeanTimeToRecovery: MTTRMetric{
            P50Minutes: mttrP50.Minutes(), P90Minutes: mttrP90.Minutes(),
            Breakdown: *mttrBreakdown, Classification: mttrClass,
        },
        OverallClass: overall,
        Period:       window.String(),
    }, nil
}

// classifyDF classifies Deployment Frequency per DORA benchmarks
func classifyDF(deploysPerDay float64) DoraClassification {
    switch {
    case deploysPerDay >= 1.0:   // multiple times per day or daily
        return ClassElite
    case deploysPerDay >= 1.0/7: // weekly to monthly
        return ClassHigh
    case deploysPerDay >= 1.0/30: // monthly to every 6 months
        return ClassMedium
    default:
        return ClassLow
    }
}

// classifyLT classifies Lead Time per DORA benchmarks
func classifyLT(p50 time.Duration) DoraClassification {
    switch {
    case p50 <= 24*time.Hour:       // less than one day
        return ClassElite
    case p50 <= 7*24*time.Hour:     // less than one week
        return ClassHigh
    case p50 <= 30*24*time.Hour:    // less than one month
        return ClassMedium
    default:
        return ClassLow
    }
}

// classifyCFR classifies Change Failure Rate per DORA benchmarks
func classifyCFR(rate float64) DoraClassification {
    switch {
    case rate <= 0.05:  // 0-5%
        return ClassElite
    case rate <= 0.10:  // 5-10%
        return ClassHigh
    case rate <= 0.15:  // 10-15%
        return ClassMedium
    default:
        return ClassLow
    }
}

// classifyMTTR classifies MTTR per DORA benchmarks
func classifyMTTR(p50 time.Duration) DoraClassification {
    switch {
    case p50 <= 1*time.Hour:        // less than one hour
        return ClassElite
    case p50 <= 24*time.Hour:       // less than one day
        return ClassHigh
    case p50 <= 7*24*time.Hour:     // less than one week
        return ClassMedium
    default:
        return ClassLow
    }
}

func worstClassification(classes ...DoraClassification) DoraClassification {
    rank := map[DoraClassification]int{
        ClassElite: 4, ClassHigh: 3, ClassMedium: 2, ClassLow: 1,
    }
    worst := ClassElite
    for _, c := range classes {
        if rank[c] < rank[worst] {
            worst = c
        }
    }
    return worst
}
```

### Go — Prometheus Metrics Registration

```go
// internal/platform/prometheus/metrics.go
package prometheus

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    DeploymentsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "dora_deployments_total",
            Help: "Total deployments to production",
        },
        []string{"service", "status"},
    )

    LeadTimeSeconds = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "dora_lead_time_seconds",
            Help:    "Lead time from first commit to production deploy",
            Buckets: []float64{3600, 14400, 28800, 86400, 259200, 604800, 2592000},
        },
        []string{"service"},
    )

    LeadTimeCodingSeconds = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "dora_lead_time_coding_seconds",
            Help:    "Coding phase of lead time",
            Buckets: []float64{1800, 3600, 7200, 14400, 28800, 86400},
        },
        []string{"service"},
    )

    LeadTimeReviewSeconds = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "dora_lead_time_review_seconds",
            Help:    "Review phase of lead time",
            Buckets: []float64{900, 1800, 3600, 7200, 14400, 28800, 86400},
        },
        []string{"service"},
    )

    ChangeFailuresTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "dora_change_failures_total",
            Help: "Deployments that resulted in failure",
        },
        []string{"service", "failure_type"},
    )

    IncidentRecoverySeconds = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "dora_incident_recovery_seconds",
            Help:    "Time from incident detection to service recovery",
            Buckets: []float64{300, 900, 1800, 3600, 7200, 14400, 86400, 604800},
        },
        []string{"service", "severity"},
    )

    SpaceSurveyScore = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "space_survey_score",
            Help: "Average SPACE survey score by dimension",
        },
        []string{"team", "dimension", "quarter"},
    )

    SpaceReviewTurnaroundSeconds = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "space_review_turnaround_seconds",
            Help:    "Time from PR opened to first review",
            Buckets: []float64{900, 1800, 3600, 7200, 14400, 28800, 86400},
        },
        []string{"service"},
    )

    SpaceCIBuildSeconds = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "space_ci_build_seconds",
            Help:    "CI build duration",
            Buckets: []float64{30, 60, 120, 300, 600, 900, 1800},
        },
        []string{"service"},
    )
)
```

### Spring Boot — DORA Calculator Service

```java
// domain/service/DoraCalculatorService.java

@Service
@RequiredArgsConstructor
public class DoraCalculatorService {

    private final DeploymentEventRepository deployRepo;
    private final PullRequestEventRepository prRepo;
    private final IncidentEventRepository incidentRepo;

    public DoraMetrics calculate(String service, Duration window) {
        var now = Instant.now();
        var from = now.minus(window);
        var days = window.toDays();

        // 1. Deployment Frequency
        long successfulDeploys = deployRepo
            .countByServiceAndStatusAndDeployedAtBetween(service, "success", from, now);
        double deploysPerDay = (double) successfulDeploys / days;
        var dfClass = classifyDF(deploysPerDay);

        // 2. Lead Time
        var leadTimes = prRepo
            .findByServiceAndDeployedAtBetween(service, from, now)
            .stream()
            .filter(pr -> pr.getLeadTimeSecs() != null)
            .map(PullRequestEvent::getLeadTimeSecs)
            .sorted()
            .toList();

        double ltP50Hours = percentile(leadTimes, 0.50) / 3600.0;
        double ltP90Hours = percentile(leadTimes, 0.90) / 3600.0;
        var ltClass = classifyLT(Duration.ofHours((long) ltP50Hours));

        // 3. Change Failure Rate
        long failedDeploys = deployRepo
            .countByServiceAndCausedFailureAndDeployedAtBetween(service, true, from, now);
        double cfr = successfulDeploys > 0
            ? (double) failedDeploys / successfulDeploys : 0.0;
        var cfrClass = classifyCFR(cfr);

        // 4. MTTR
        var mttrValues = incidentRepo
            .findByServiceAndDetectedAtBetween(service, from, now)
            .stream()
            .filter(inc -> inc.getMttrSeconds() != null)
            .map(IncidentEvent::getMttrSeconds)
            .sorted()
            .toList();

        double mttrP50Min = percentile(mttrValues, 0.50) / 60.0;
        double mttrP90Min = percentile(mttrValues, 0.90) / 60.0;
        var mttrClass = classifyMTTR(Duration.ofMinutes((long) mttrP50Min));

        var overall = worstOf(dfClass, ltClass, cfrClass, mttrClass);

        return DoraMetrics.builder()
            .deploymentFrequency(new DeploymentFrequencyMetric(
                deploysPerDay, (int) successfulDeploys, dfClass))
            .leadTime(new LeadTimeMetric(ltP50Hours, ltP90Hours, ltClass))
            .changeFailureRate(new ChangeFailureRateMetric(
                cfr, (int) failedDeploys, (int) successfulDeploys, cfrClass))
            .meanTimeToRecovery(new MTTRMetric(mttrP50Min, mttrP90Min, mttrClass))
            .overallClassification(overall)
            .period(window.toString())
            .build();
    }

    private DoraClassification classifyDF(double deploysPerDay) {
        if (deploysPerDay >= 1.0) return DoraClassification.ELITE;
        if (deploysPerDay >= 1.0 / 7) return DoraClassification.HIGH;
        if (deploysPerDay >= 1.0 / 30) return DoraClassification.MEDIUM;
        return DoraClassification.LOW;
    }

    private DoraClassification classifyLT(Duration p50) {
        if (p50.compareTo(Duration.ofDays(1)) <= 0) return DoraClassification.ELITE;
        if (p50.compareTo(Duration.ofDays(7)) <= 0) return DoraClassification.HIGH;
        if (p50.compareTo(Duration.ofDays(30)) <= 0) return DoraClassification.MEDIUM;
        return DoraClassification.LOW;
    }

    private DoraClassification classifyCFR(double rate) {
        if (rate <= 0.05) return DoraClassification.ELITE;
        if (rate <= 0.10) return DoraClassification.HIGH;
        if (rate <= 0.15) return DoraClassification.MEDIUM;
        return DoraClassification.LOW;
    }

    private DoraClassification classifyMTTR(Duration p50) {
        if (p50.compareTo(Duration.ofHours(1)) <= 0) return DoraClassification.ELITE;
        if (p50.compareTo(Duration.ofDays(1)) <= 0) return DoraClassification.HIGH;
        if (p50.compareTo(Duration.ofDays(7)) <= 0) return DoraClassification.MEDIUM;
        return DoraClassification.LOW;
    }

    private DoraClassification worstOf(DoraClassification... classes) {
        return Arrays.stream(classes)
            .min(Comparator.comparingInt(DoraClassification::rank))
            .orElse(DoraClassification.LOW);
    }

    private double percentile(List<Long> sorted, double p) {
        if (sorted.isEmpty()) return 0;
        int idx = (int) Math.ceil(p * sorted.size()) - 1;
        return sorted.get(Math.max(0, idx));
    }
}

public enum DoraClassification {
    ELITE(4), HIGH(3), MEDIUM(2), LOW(1);

    private final int rank;

    DoraClassification(int rank) { this.rank = rank; }
    public int rank() { return rank; }
}
```

### Spring Boot — Webhook Controller

```java
// web/controller/DeploymentEventController.java

@RestController
@RequestMapping("/api/v1/events/deployments")
@RequiredArgsConstructor
@Validated
public class DeploymentEventController {

    private final DeploymentEventService deployService;
    private final MeterRegistry meterRegistry;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public DeploymentEventResponse receiveDeployment(
            @Valid @RequestBody DeploymentEventRequest request) {

        var event = deployService.record(request.toEvent());

        // Increment Prometheus counter
        meterRegistry.counter("dora.deployments.total",
            "service", event.getService(),
            "status", event.getStatus()
        ).increment();

        // Observe lead time if PR data available
        if (event.getLeadTimeSeconds() != null) {
            meterRegistry.timer("dora.lead.time",
                "service", event.getService()
            ).record(Duration.ofSeconds(event.getLeadTimeSeconds()));
        }

        return DeploymentEventResponse.from(event);
    }
}

// web/dto/DeploymentEventRequest.java
public record DeploymentEventRequest(
    @NotBlank String service,
    String version,
    @NotBlank @Size(min = 7, max = 40) String sha,
    @NotBlank String environment,
    @NotNull Instant deployedAt,
    @NotBlank @Pattern(regexp = "success|failure|rollback") String status,
    String pipelineId,
    String triggeredBy
) {
    public DeploymentEvent toEvent() {
        return DeploymentEvent.builder()
            .service(service)
            .version(version)
            .sha(sha)
            .environment(environment)
            .deployedAt(deployedAt)
            .status(status)
            .pipelineId(pipelineId)
            .triggeredBy(triggeredBy)
            .build();
    }
}
```

### Go — HMAC Webhook Validation Middleware

```go
// internal/middleware/hmac.go
package middleware

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "io"
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
)

func HMACValidation(secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        signature := c.GetHeader("X-Hub-Signature-256")
        if signature == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "missing X-Hub-Signature-256 header",
            })
            return
        }

        body, err := io.ReadAll(c.Request.Body)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{
                "error": "failed to read request body",
            })
            return
        }
        // Restore body for downstream handlers
        c.Request.Body = io.NopCloser(strings.NewReader(string(body)))

        mac := hmac.New(sha256.New, []byte(secret))
        mac.Write(body)
        expectedSig := "sha256=" + hex.EncodeToString(mac.Sum(nil))

        if !hmac.Equal([]byte(signature), []byte(expectedSig)) {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
                "error": "invalid webhook signature",
            })
            return
        }

        c.Next()
    }
}
```

### Seed Data Script (Go)

```go
// scripts/seed_data.go
package main

import (
    "encoding/json"
    "fmt"
    "math/rand"
    "net/http"
    "strings"
    "time"
)

func main() {
    baseURL := "http://localhost:8090/api/v1"
    services := []string{"order-service", "payment-service", "inventory-service"}
    now := time.Now()

    // Generate 90 days of deployment data
    for _, svc := range services {
        for day := 90; day >= 0; day-- {
            date := now.AddDate(0, 0, -day)

            // Simulate 1-5 deploys per day (elite-level)
            deploysToday := rand.Intn(5) + 1
            for d := 0; d < deploysToday; d++ {
                hour := 8 + rand.Intn(10) // 8am-6pm
                deployTime := time.Date(date.Year(), date.Month(), date.Day(),
                    hour, rand.Intn(60), 0, 0, time.UTC)

                status := "success"
                if rand.Float64() < 0.04 { // ~4% failure rate
                    status = "failure"
                }

                body, _ := json.Marshal(map[string]interface{}{
                    "service":      svc,
                    "version":      fmt.Sprintf("v1.%d.%d", 90-day, d),
                    "sha":          fmt.Sprintf("%040x", rand.Int63()),
                    "environment":  "production",
                    "deployed_at":  deployTime.Format(time.RFC3339),
                    "status":       status,
                    "pipeline_id":  fmt.Sprintf("run-%d-%d", day, d),
                    "triggered_by": "merge_to_main",
                })

                resp, err := http.Post(baseURL+"/events/deployments",
                    "application/json", strings.NewReader(string(body)))
                if err != nil {
                    fmt.Printf("Error: %v\n", err)
                    continue
                }
                resp.Body.Close()
            }
        }
    }

    fmt.Println("Seed data generated for 90 days across 3 services")
}
```

### docker-compose.yml — DORA Infrastructure

```yaml
# docker-compose.yml
services:
  dora-collector:
    build: .
    ports:
      - "8090:8090"
    environment:
      DATABASE_URL: "postgres://dora:dora@postgres:5432/dora_metrics?sslmode=disable"
      HMAC_SECRET: "${HMAC_SECRET:-dev-secret}"
      PORT: "8090"
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:17
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: dora_metrics
      POSTGRES_USER: dora
      POSTGRES_PASSWORD: dora
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dora"]
      interval: 5s
      timeout: 3s
      retries: 5

  prometheus:
    image: prom/prometheus:v2.53.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules:/etc/prometheus/rules
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=90d'

  grafana:
    image: grafana/grafana:11.0.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Viewer
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards

volumes:
  pgdata:
```

---

## Extensões Opcionais

- **GitHub App**: Criar uma GitHub App que coleta automaticamente deploy events, PR events e calcula lead time sem webhooks manuais
- **Multi-team view**: Dashboard com comparação entre equipes (sem ranking), mostrando tendências e contexto
- **DORA Quick Check automatizado**: Endpoint `/api/v1/metrics/dora/quickcheck` que retorna assessment no formato do dora.dev/quickcheck
- **SPACE Survey automação**: Slack Bot que envia surveys trimestrais e coleta respostas automaticamente
- **Anomaly detection**: Alertas quando uma métrica DORA degrada (ex: DF cai 50% em 7 dias)
- **Correlation analysis**: Correlacionar melhorias em DORA com ações específicas (ex: migrar para trunk-based → LT caiu 40%)
- **Cost correlation**: Integrar com cloud cost data para correlacionar deploy frequency com custo de infra
- **Tech Debt proxy**: Usar tendência de LT e CFR como proxy para Tech Debt (conforme doc TECH-STRATEGY)
- **Integration com Backstage/Port**: Plugin para exibir DORA metrics no developer portal
- **Apache DevLake integration**: Usar DevLake como alternativa open-source para coleta de dados de múltiplos VCS/CI

---

## Referências

- **Accelerate** — Nicole Forsgren, Jez Humble, Gene Kim (IT Revolution, 2018)
- **"The SPACE of Developer Productivity"** — Forsgren et al. (ACM Queue, 2021) — https://queue.acm.org/detail.cfm?id=3454124
- **State of DevOps Report** — https://dora.dev/research/ (anual)
- **DORA Quick Check** — https://dora.dev/quickcheck/
- **Four Keys (Google OSS)** — https://github.com/dora-team/fourkeys
- **Apache DevLake** — https://devlake.apache.org/
- **"DevEx: What Actually Drives Productivity"** — ACM (2023) — https://queue.acm.org/detail.cfm?id=3595878
- **A Typology of Organisational Cultures** — Ron Westrum (2004)
- **Docs de referência:** `../../.docs/performance/06-dora-space-metrics.md`
