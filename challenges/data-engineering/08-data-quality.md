# Level 08 — Data Quality

> **Objetivo:** Implementar validação de qualidade de dados com Great Expectations,
> data contracts, circuit breaker pattern e observabilidade de pipelines.

**Referência:** [05 — Data Governance](../../.docs/data-engineering/05-data-governance.md)

**Pré-requisito:** Level 07 concluído.

---

## Contexto

Dados sem qualidade são piores que dados inexistentes — levam a decisões erradas.
**Data quality** envolve validação de schema, completude, unicidade, freshness,
integridade referencial e regras de negócio. O **circuit breaker pattern** impede
que dados ruins contaminem downstream. **Data contracts** formalizam expectativas
entre producers e consumers.

---

## Parte 1 — ADRs (2 obrigatórios)

### ADR-019: Data Quality Framework

**Arquivo:** `docs/adrs/ADR-019-data-quality-framework.md`

**Context:** Precisamos validar qualidade de dados em cada zona do data lake
e antes de carregar no warehouse.

**Options:**
1. **Great Expectations** — Python, expectation suites, docs automáticas
2. **dbt tests** — SQL-based, integrado ao dbt, simple assertions
3. **Soda Core** — SQL + YAML checks, SodaCL language
4. **AWS Glue Data Quality** — serverless, regras no Glue Catalog
5. **Custom validators** — Python scripts com pandas/PyArrow

**Decision Drivers:**
- Expressividade das regras (SQL vs Python vs YAML)
- Integração com data lake e warehouse
- Profiling automático
- Alerting e reporting
- Custo de manutenção

---

### ADR-020: Data Contract Strategy

**Arquivo:** `docs/adrs/ADR-020-data-contracts.md`

**Context:** Producers e consumers de dados precisam concordar em schema, qualidade
e SLA.

**Options:**
1. **Schema + Quality + SLA** — contrato formal com schema (Avro), expectations, SLA
2. **API Contract** — OpenAPI-style contract para data APIs
3. **dbt contract** — column constraints + tests no dbt model
4. **DataHub contract** — managed contracts no DataHub metadata platform

**Decision Drivers:**
- Enforcement (build-time vs runtime)
- Evolução sem breaking changes
- Discovery e documentação
- Governança cross-team

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

### DrawIO-015: Data Quality Architecture

**Arquivo:** `docs/diagrams/08-data-quality.drawio`

```
┌────────────┐    ┌──────────────┐    ┌──────────────┐
│  Ingestion │───▶│  QUALITY     │───▶│   Load       │
│            │    │  GATE        │    │              │
│  raw data  │    │              │    │  curated     │
└────────────┘    │  ┌────────┐  │    │  zone        │
                  │  │  GE    │  │    └──────────────┘
                  │  │ Suite  │  │
                  │  └───┬────┘  │
                  │      │       │
                  │  ┌───▼────┐  │
                  │  │ Pass?  │  │
                  │  └───┬────┘  │
                  │   Y  │  N   │
                  └──────┘  │   │
                            │   │
                  ┌─────────▼───┘
                  │  CIRCUIT
                  │  BREAKER
                  │
                  ├──▶ Quarantine Zone (S3)
                  ├──▶ Alert (SNS/Slack)
                  └──▶ Pipeline Halt
```

### DrawIO-016: Data Contract Flow

**Arquivo:** `docs/diagrams/08-data-contracts.drawio`

```
┌──────────────┐              ┌──────────────┐
│   Producer   │              │   Consumer   │
│   (Team A)   │              │   (Team B)   │
└──────┬───────┘              └──────┬───────┘
       │                             │
       │  publishes                  │  expects
       ▼                             ▼
┌─────────────────────────────────────────────┐
│              DATA CONTRACT                  │
│                                             │
│  Schema:    Avro v3 (backward compatible)   │
│  Quality:   completeness > 99%              │
│             uniqueness(id) = 100%           │
│             freshness < 1 hour              │
│  SLA:       available by 07:00 UTC daily    │
│  Owner:     team-a@company.com              │
│  Version:   2.1.0                           │
└─────────────────────────────────────────────┘
       │
       ▼
  CI/CD validates contract on every change
```

---

## Parte 3 — Implementação

### 3.1 — Great Expectations Setup

```python
# python/quality/ge_setup.py
import great_expectations as gx

def create_context():
    """Cria Great Expectations context."""
    context = gx.get_context()
    
    # Add S3 datasource (via LocalStack)
    datasource = context.sources.add_pandas_s3(
        name="data_lake",
        bucket="data-lake-curated",
        boto3_options={
            "endpoint_url": "http://localhost:4566",
            "aws_access_key_id": "test",
            "aws_secret_access_key": "test",
        },
    )
    
    return context
```

### 3.2 — Expectation Suite (Quality Rules)

```python
# python/quality/expectations/orders_suite.py
import great_expectations as gx
from great_expectations.core.expectation_suite import ExpectationSuite

def create_orders_suite() -> ExpectationSuite:
    """Define quality expectations para orders dataset."""
    suite = ExpectationSuite(name="orders_quality")
    
    # Completeness
    suite.add_expectation(
        gx.expectations.ExpectColumnValuesToNotBeNull(column="order_id")
    )
    suite.add_expectation(
        gx.expectations.ExpectColumnValuesToNotBeNull(column="customer_id")
    )
    suite.add_expectation(
        gx.expectations.ExpectColumnValuesToNotBeNull(column="amount")
    )
    
    # Uniqueness
    suite.add_expectation(
        gx.expectations.ExpectColumnValuesToBeUnique(column="order_id")
    )
    
    # Range / Domain
    suite.add_expectation(
        gx.expectations.ExpectColumnValuesToBeBetween(
            column="amount", min_value=0.01, max_value=1_000_000
        )
    )
    suite.add_expectation(
        gx.expectations.ExpectColumnValuesToBeInSet(
            column="status",
            value_set=["pending", "completed", "cancelled", "refunded"],
        )
    )
    
    # Freshness (data should be from today)
    suite.add_expectation(
        gx.expectations.ExpectColumnMaxToBeBetween(
            column="created_at",
            min_value="2024-01-01",
        )
    )
    
    # Row count (at least 100 records per batch)
    suite.add_expectation(
        gx.expectations.ExpectTableRowCountToBeBetween(min_value=100)
    )
    
    # Column count (schema drift detection)
    suite.add_expectation(
        gx.expectations.ExpectTableColumnCountToEqual(value=8)
    )
    
    return suite
```

### 3.3 — Circuit Breaker Pattern

```python
# python/quality/circuit_breaker.py
import json
from datetime import datetime
from enum import Enum
from dataclasses import dataclass
import boto3

class CircuitState(Enum):
    CLOSED = "closed"     # Normal: data flows
    OPEN = "open"         # Blocked: quality too low
    HALF_OPEN = "half_open"  # Testing: partial flow

@dataclass
class QualityResult:
    passed: bool
    score: float  # 0.0 - 1.0
    failed_expectations: list[str]
    timestamp: str

class DataQualityCircuitBreaker:
    def __init__(self, threshold: float = 0.95, failure_count_limit: int = 3):
        self.threshold = threshold
        self.failure_count_limit = failure_count_limit
        self.failure_count = 0
        self.state = CircuitState.CLOSED
        self.s3 = boto3.client("s3", endpoint_url="http://localhost:4566",
                                aws_access_key_id="test",
                                aws_secret_access_key="test")
    
    def check(self, result: QualityResult) -> bool:
        """Decide se dados podem prosseguir ou devem ser quarantined."""
        if self.state == CircuitState.OPEN:
            print(f"⛔ Circuit OPEN — data quarantined")
            self._quarantine(result)
            return False
        
        if result.score >= self.threshold:
            self.failure_count = 0
            self.state = CircuitState.CLOSED
            print(f"✓ Quality passed ({result.score:.1%})")
            return True
        
        self.failure_count += 1
        print(f"⚠ Quality failed ({result.score:.1%}), "
              f"failures: {self.failure_count}/{self.failure_count_limit}")
        
        if self.failure_count >= self.failure_count_limit:
            self.state = CircuitState.OPEN
            print(f"⛔ Circuit OPENED — consecutive failures exceeded threshold")
            self._alert(result)
        
        self._quarantine(result)
        return False
    
    def _quarantine(self, result: QualityResult):
        """Move dados ruins para quarantine zone."""
        key = f"quarantine/{datetime.utcnow().strftime('%Y/%m/%d/%H%M%S')}/result.json"
        self.s3.put_object(
            Bucket="data-lake-raw",
            Key=key,
            Body=json.dumps({
                "score": result.score,
                "failed": result.failed_expectations,
                "timestamp": result.timestamp,
                "circuit_state": self.state.value,
            }),
        )
    
    def _alert(self, result: QualityResult):
        """Envia alerta quando circuit abre."""
        sns = boto3.client("sns", endpoint_url="http://localhost:4566",
                            aws_access_key_id="test", aws_secret_access_key="test",
                            region_name="us-east-1")
        sns.publish(
            TopicArn="arn:aws:sns:us-east-1:000000000000:pipeline-alerts",
            Subject="🚨 Data Quality Circuit Breaker OPENED",
            Message=json.dumps({
                "score": result.score,
                "failed_expectations": result.failed_expectations,
                "action": "Pipeline halted. Manual review required.",
            }),
        )
```

### 3.4 — Data Contract (YAML)

```yaml
# contracts/orders_contract.yml
apiVersion: datacontract/v1
kind: DataContract

metadata:
  name: orders-contract
  version: "2.1.0"
  owner: team-orders
  contact: orders-team@company.com

schema:
  type: avro
  registry: http://localhost:8085
  subject: orders-value
  compatibility: BACKWARD

quality:
  completeness:
    - column: order_id
      threshold: 1.0
    - column: customer_id
      threshold: 1.0
    - column: amount
      threshold: 0.99

  uniqueness:
    - column: order_id
      threshold: 1.0

  validity:
    - column: amount
      min: 0.01
      max: 1000000
    - column: status
      allowed_values: [pending, completed, cancelled, refunded]

  freshness:
    max_age_hours: 1
    reference_column: created_at

sla:
  availability: "07:00 UTC daily"
  latency_p99: "< 30 minutes"
  retention_days: 365

lineage:
  sources:
    - system: orders-api
      table: orders
      method: CDC (Debezium)
  destinations:
    - system: data-lake
      zone: curated
      format: parquet
    - system: warehouse
      table: warehouse.fact_orders
```

### 3.5 — Go: Quality Metrics Collector

```go
// go/cmd/quality-collector/main.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/dynamodb"
    "github.com/aws/aws-sdk-go-v2/service/dynamodb/types"
)

type QualityMetric struct {
    PipelineID   string    `json:"pipeline_id"`
    RunID        string    `json:"run_id"`
    Score        float64   `json:"score"`
    TotalChecks  int       `json:"total_checks"`
    PassedChecks int       `json:"passed_checks"`
    FailedChecks []string  `json:"failed_checks"`
    Timestamp    time.Time `json:"timestamp"`
}

func recordMetric(ctx context.Context, client *dynamodb.Client, metric QualityMetric) error {
    failedJSON, _ := json.Marshal(metric.FailedChecks)
    
    _, err := client.PutItem(ctx, &dynamodb.PutItemInput{
        TableName: aws.String("pipeline-state"),
        Item: map[string]types.AttributeValue{
            "pipeline_id":   &types.AttributeValueMemberS{Value: metric.PipelineID},
            "run_id":        &types.AttributeValueMemberS{Value: metric.RunID},
            "score":         &types.AttributeValueMemberN{Value: fmt.Sprintf("%.4f", metric.Score)},
            "total_checks":  &types.AttributeValueMemberN{Value: fmt.Sprintf("%d", metric.TotalChecks)},
            "passed_checks": &types.AttributeValueMemberN{Value: fmt.Sprintf("%d", metric.PassedChecks)},
            "failed_checks": &types.AttributeValueMemberS{Value: string(failedJSON)},
            "timestamp":     &types.AttributeValueMemberS{Value: metric.Timestamp.Format(time.RFC3339)},
        },
    })
    return err
}
```

### 3.6 — Java: Quality Gate Service

```java
// java/src/.../quality/QualityGateService.java
@Service
public class QualityGateService {

    private final DataSource warehouseDs;
    
    public QualityReport validate(String table, QualityContract contract) {
        var report = new QualityReport(table);
        
        // Check completeness
        for (var rule : contract.getCompleteness()) {
            long nullCount = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM " + table + " WHERE " + rule.getColumn() + " IS NULL",
                Long.class
            );
            long totalCount = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM " + table, Long.class
            );
            double completeness = 1.0 - ((double) nullCount / totalCount);
            report.addCheck(rule.getColumn() + "_completeness", 
                completeness >= rule.getThreshold(), completeness);
        }
        
        // Check uniqueness
        for (var rule : contract.getUniqueness()) {
            long duplicates = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) - COUNT(DISTINCT " + rule.getColumn() + ") FROM " + table,
                Long.class
            );
            report.addCheck(rule.getColumn() + "_uniqueness",
                duplicates == 0, duplicates == 0 ? 1.0 : 0.0);
        }
        
        // Circuit breaker decision
        if (report.getScore() < contract.getMinScore()) {
            report.setAction(QualityAction.QUARANTINE);
        } else {
            report.setAction(QualityAction.PROCEED);
        }
        
        return report;
    }
}
```

---

## Critérios de Aceite

- [ ] Great Expectations suite com ≥ 8 expectations
- [ ] Validation executada contra dados no S3 (LocalStack)
- [ ] Circuit breaker: 3 falhas consecutivas → circuit OPEN
- [ ] Quarantine zone: dados ruins movidos para `s3://data-lake-raw/quarantine/`
- [ ] Alerta SNS quando circuit abre
- [ ] Data contract YAML com schema, quality, SLA
- [ ] Go metrics collector gravando no DynamoDB
- [ ] Java quality gate com completeness + uniqueness checks
- [ ] ADR-019 e ADR-020 completos
- [ ] 2 DrawIOs (quality architecture + contract flow)

---

## Definição de Pronto (DoD)

- [ ] 2 ADRs (quality framework + data contracts)
- [ ] 2 DrawIOs (quality architecture + contract flow)
- [ ] Great Expectations suite funcional
- [ ] Circuit breaker pattern implementado
- [ ] Data contract definido e validado
- [ ] Go quality metrics collector
- [ ] Java quality gate service
- [ ] Commit: `feat(data-engineering-08): data quality`
