# Level 05 — Pipeline Orchestration

> **Objetivo:** Orquestrar pipelines de dados complexos com Apache Airflow, AWS
> Step Functions e ferramentas de scheduling para garantir execução confiável,
> retry e observabilidade.

**Referência:** [03 — Data Processing](../../.docs/data-engineering/03-data-processing.md)

**Pré-requisito:** Level 04 concluído.

---

## Contexto

Pipelines de dados envolvem dezenas de steps interdependentes: ingestão, validação,
transformação, qualidade, carga. Um **orquestrador** gerencia a execução, dependências
entre tasks, retries, alertas e backfills. Apache Airflow é o padrão de mercado;
AWS Step Functions oferece uma alternativa serverless.

---

## Parte 1 — ADRs (2 obrigatórios)

### ADR-012: Pipeline Orchestrator

**Arquivo:** `docs/adrs/ADR-012-pipeline-orchestrator.md`

**Context:** Precisamos orquestrar ~20 DAGs de pipelines de dados com dependências,
schedules e retries.

**Options:**
1. **Apache Airflow** — DAGs em Python, UI rica, ecossistema de operators
2. **AWS Step Functions** — serverless, state machine JSON/ASL, integração AWS
3. **Prefect** — Python-first, cloud-native, UI moderna
4. **Dagster** — software-defined assets, type-safe, testing nativo
5. **AWS MWAA** — Airflow gerenciado na AWS

**Decision Drivers:**
- Complexidade dos DAGs (branching, dynamic tasks)
- Backfill e replay capability
- Monitoramento e alertas
- Custo operacional
- Integração com Spark, dbt, Kafka

---

### ADR-013: Scheduling Strategy

**Arquivo:** `docs/adrs/ADR-013-scheduling-strategy.md`

**Context:** Precisamos definir frequência e estratégia de execução dos pipelines.

**Options:**
1. **Cron-based** — schedule fixo (hourly, daily)
2. **Event-driven** — trigger por chegada de dados (S3 events, SQS)
3. **Hybrid** — cron para batch, event para streaming/CDC
4. **Data-aware** — executa quando dados upstream estão prontos (sensors)

**Decision Drivers:**
- Latência aceitável por caso de uso
- Custo de compute (idle vs on-demand)
- Complexidade de dependências entre pipelines
- SLA de disponibilidade dos dados

---

## Parte 2 — Diagramas DrawIO (2 obrigatórios)

### DrawIO-009: Orchestration Architecture

**Arquivo:** `docs/diagrams/05-orchestration-architecture.drawio`

```
┌─────────────────────────────────────────────────────────────┐
│                    AIRFLOW CLUSTER                          │
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │  Web UI  │    │Scheduler │    │  Workers  │              │
│  │  :8082   │    │  (cron)  │    │  (Celery) │              │
│  └──────────┘    └────┬─────┘    └──────┬────┘              │
│                       │                 │                    │
│                  ┌────▼─────────────────▼────┐              │
│                  │      DAG Execution        │              │
│                  │                           │              │
│                  │  ingest → validate →      │              │
│                  │  transform → quality →    │              │
│                  │  load → notify            │              │
│                  └───────────────────────────┘              │
└──────────────────────────┬──────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │   S3     │    │PostgreSQL│    │  Spark   │
    │ (data)   │    │(metadata)│    │ (compute)│
    └──────────┘    └──────────┘    └──────────┘
```

### DrawIO-010: Step Functions State Machine

**Arquivo:** `docs/diagrams/05-step-functions.drawio`

```
Start ──▶ ExtractData ──▶ ValidateSchema
                                │
                        ┌───Success───┐───Failure──┐
                        ▼             ▼            ▼
                  TransformData   RetryExtract  AlertOps
                        │                          │
                  ┌─────▼──────┐              SendSNS
                  │QualityCheck│              
                  └─────┬──────┘
                        │
                ┌───Pass───┐───Fail──┐
                ▼                    ▼
           LoadWarehouse      QuarantineData
                │                    │
                ▼                    ▼
           UpdateCatalog      NotifyDataTeam
                │
                ▼
              End
```

---

## Parte 3 — Implementação

### 3.1 — Apache Airflow: DAG de ETL Completo

```python
# airflow/dags/etl_orders_dag.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.operators.bash import BashOperator
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from airflow.providers.amazon.aws.operators.s3 import S3CopyObjectOperator
from airflow.utils.trigger_rule import TriggerRule

default_args = {
    "owner": "data-engineering",
    "depends_on_past": False,
    "email_on_failure": True,
    "email": ["data-team@example.com"],
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
    "retry_exponential_backoff": True,
    "max_retry_delay": timedelta(minutes=30),
}

with DAG(
    dag_id="etl_orders_daily",
    default_args=default_args,
    description="Daily ETL pipeline for orders data",
    schedule="0 6 * * *",  # 06:00 UTC daily
    start_date=datetime(2024, 1, 1),
    catchup=False,
    max_active_runs=1,
    tags=["orders", "etl", "daily"],
) as dag:

    # 1. Sensor: wait for source data
    wait_for_data = S3KeySensor(
        task_id="wait_for_source_data",
        bucket_name="data-lake-raw",
        bucket_key="orders/{{ ds }}/",
        aws_conn_id="localstack",
        poke_interval=60,
        timeout=3600,
    )

    # 2. Extract: copy raw data to processing area
    def extract(**context):
        execution_date = context["ds"]
        # Read from raw zone
        import boto3
        s3 = boto3.client("s3", endpoint_url="http://localstack:4566",
                           aws_access_key_id="test", aws_secret_access_key="test")
        response = s3.list_objects_v2(
            Bucket="data-lake-raw",
            Prefix=f"orders/{execution_date}/",
        )
        files = [obj["Key"] for obj in response.get("Contents", [])]
        context["ti"].xcom_push(key="source_files", value=files)
        return len(files)

    extract_task = PythonOperator(
        task_id="extract_data",
        python_callable=extract,
    )

    # 3. Validate: check schema and data quality
    def validate(**context):
        files = context["ti"].xcom_pull(task_ids="extract_data", key="source_files")
        # Schema validation + row count check
        if not files:
            return "no_data_branch"
        return "transform_data"

    validate_task = BranchPythonOperator(
        task_id="validate_schema",
        python_callable=validate,
    )

    # 4. Transform: PySpark or dbt
    def transform(**context):
        execution_date = context["ds"]
        # Run PySpark transformation
        from etl.batch.spark_etl import run_pipeline
        run_pipeline(
            source=f"orders/{execution_date}/",
            target=f"orders_curated/{execution_date}/",
        )

    transform_task = PythonOperator(
        task_id="transform_data",
        python_callable=transform,
    )

    # 5. dbt run
    dbt_run = BashOperator(
        task_id="dbt_transform",
        bash_command="cd /opt/dbt && dbt run --select marts.mart_sales_summary --vars '{execution_date: {{ ds }}}'",
    )

    # 6. Quality check
    def quality_check(**context):
        # Run Great Expectations validation
        pass

    quality_task = PythonOperator(
        task_id="quality_check",
        python_callable=quality_check,
    )

    # 7. Load to warehouse
    def load_warehouse(**context):
        pass

    load_task = PythonOperator(
        task_id="load_warehouse",
        python_callable=load_warehouse,
    )

    # 8. No data branch
    def handle_no_data(**context):
        print("No data found for execution date")

    no_data = PythonOperator(
        task_id="no_data_branch",
        python_callable=handle_no_data,
    )

    # 9. Notify
    notify_success = BashOperator(
        task_id="notify_success",
        bash_command='echo "Pipeline completed for {{ ds }}"',
        trigger_rule=TriggerRule.NONE_FAILED_MIN_ONE_SUCCESS,
    )

    # DAG dependencies
    wait_for_data >> extract_task >> validate_task
    validate_task >> transform_task >> dbt_run >> quality_task >> load_task >> notify_success
    validate_task >> no_data >> notify_success
```

### 3.2 — Airflow: DAG Factory Pattern

```python
# airflow/dags/dag_factory.py
"""Factory para criar DAGs parametrizadas por configuração YAML."""
import yaml
from pathlib import Path
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

def create_etl_dag(config: dict) -> DAG:
    dag = DAG(
        dag_id=config["dag_id"],
        schedule=config.get("schedule", "@daily"),
        start_date=datetime(2024, 1, 1),
        max_active_runs=1,
        tags=config.get("tags", []),
        default_args={
            "retries": config.get("retries", 3),
            "retry_delay": timedelta(minutes=config.get("retry_delay_min", 5)),
        },
    )
    
    with dag:
        for step in config["steps"]:
            PythonOperator(
                task_id=step["name"],
                python_callable=globals()[step["callable"]],
                op_kwargs=step.get("kwargs", {}),
            )
    
    return dag

# Load all DAG configs
config_dir = Path(__file__).parent / "configs"
for config_file in config_dir.glob("*.yml"):
    config = yaml.safe_load(config_file.read_text())
    globals()[config["dag_id"]] = create_etl_dag(config)
```

### 3.3 — AWS Step Functions (LocalStack)

```python
# python/orchestration/step_functions.py
import json
import boto3

sfn = boto3.client("stepfunctions", endpoint_url="http://localhost:4566",
                    aws_access_key_id="test", aws_secret_access_key="test",
                    region_name="us-east-1")

DEFINITION = {
    "Comment": "Data Pipeline Orchestration",
    "StartAt": "ExtractData",
    "States": {
        "ExtractData": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:000000000000:function:extract",
            "Next": "ValidateSchema",
            "Retry": [
                {
                    "ErrorEquals": ["States.TaskFailed"],
                    "IntervalSeconds": 60,
                    "MaxAttempts": 3,
                    "BackoffRate": 2.0,
                }
            ],
        },
        "ValidateSchema": {
            "Type": "Choice",
            "Choices": [
                {
                    "Variable": "$.validation.passed",
                    "BooleanEquals": True,
                    "Next": "TransformData",
                }
            ],
            "Default": "QuarantineData",
        },
        "TransformData": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:000000000000:function:transform",
            "Next": "QualityCheck",
        },
        "QualityCheck": {
            "Type": "Choice",
            "Choices": [
                {
                    "Variable": "$.quality.score",
                    "NumericGreaterThanEquals": 0.95,
                    "Next": "LoadWarehouse",
                }
            ],
            "Default": "QuarantineData",
        },
        "LoadWarehouse": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:000000000000:function:load",
            "Next": "UpdateCatalog",
        },
        "UpdateCatalog": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:000000000000:function:catalog",
            "End": True,
        },
        "QuarantineData": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:000000000000:function:quarantine",
            "Next": "NotifyFailure",
        },
        "NotifyFailure": {
            "Type": "Task",
            "Resource": "arn:aws:states:::sns:publish",
            "Parameters": {
                "TopicArn": "arn:aws:sns:us-east-1:000000000000:pipeline-alerts",
                "Message.$": "$.error",
            },
            "End": True,
        },
    },
}

def create_state_machine():
    response = sfn.create_state_machine(
        name="data-pipeline",
        definition=json.dumps(DEFINITION),
        roleArn="arn:aws:iam::000000000000:role/step-functions-role",
    )
    print(f"✓ State Machine ARN: {response['stateMachineArn']}")
    return response["stateMachineArn"]

def execute_pipeline(arn: str, input_data: dict):
    response = sfn.start_execution(
        stateMachineArn=arn,
        input=json.dumps(input_data),
    )
    print(f"✓ Execution ARN: {response['executionArn']}")
    return response["executionArn"]
```

### 3.4 — Go: Workflow Scheduler

```go
// go/cmd/scheduler/main.go
package main

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"
)

type Task struct {
    Name         string
    DependsOn    []string
    Execute      func(ctx context.Context) error
    Retries      int
    RetryDelay   time.Duration
}

type DAG struct {
    Tasks map[string]*Task
}

func (d *DAG) Run(ctx context.Context) error {
    completed := make(map[string]bool)
    var mu sync.Mutex

    for len(completed) < len(d.Tasks) {
        var wg sync.WaitGroup
        for name, task := range d.Tasks {
            if completed[name] {
                continue
            }
            // Check dependencies
            ready := true
            for _, dep := range task.DependsOn {
                if !completed[dep] {
                    ready = false
                    break
                }
            }
            if !ready {
                continue
            }

            wg.Add(1)
            go func(name string, task *Task) {
                defer wg.Done()
                
                var err error
                for attempt := 0; attempt <= task.Retries; attempt++ {
                    err = task.Execute(ctx)
                    if err == nil {
                        break
                    }
                    log.Printf("[%s] attempt %d failed: %v", name, attempt+1, err)
                    time.Sleep(task.RetryDelay)
                }

                mu.Lock()
                if err == nil {
                    completed[name] = true
                    fmt.Printf("✓ %s completed\n", name)
                } else {
                    fmt.Printf("✗ %s failed after %d retries\n", name, task.Retries+1)
                }
                mu.Unlock()
            }(name, task)
        }
        wg.Wait()
    }
    return nil
}
```

### 3.5 — Docker Compose: Airflow Stack

```yaml
# infra/docker-compose.airflow.yml
services:
  airflow-webserver:
    image: apache/airflow:2.8.1
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@airflow-db:5432/airflow
      AIRFLOW__CORE__LOAD_EXAMPLES: "false"
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - ./airflow/plugins:/opt/airflow/plugins
    ports:
      - "8082:8080"
    depends_on:
      - airflow-db
    command: webserver

  airflow-scheduler:
    image: apache/airflow:2.8.1
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@airflow-db:5432/airflow
    volumes:
      - ./airflow/dags:/opt/airflow/dags
    depends_on:
      - airflow-db
    command: scheduler

  airflow-db:
    image: postgres:16
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - airflow_db_data:/var/lib/postgresql/data

volumes:
  airflow_db_data:
```

---

## Critérios de Aceite

- [ ] Airflow DAG com ≥ 6 tasks e dependências
- [ ] Branching (BranchPythonOperator) funcional
- [ ] S3KeySensor aguardando dados
- [ ] Retries com exponential backoff configurados
- [ ] DAG factory pattern implementado
- [ ] Step Functions state machine criada no LocalStack
- [ ] Step Functions execution com branching (Choice state)
- [ ] Go DAG scheduler com dependências e retries
- [ ] Airflow UI acessível (:8082)
- [ ] ADR-012 e ADR-013 completos
- [ ] 2 DrawIOs (orchestration arch + step functions)

---

## Definição de Pronto (DoD)

- [ ] 2 ADRs (orchestrator + scheduling)
- [ ] 2 DrawIOs (orchestration architecture + step functions)
- [ ] Airflow DAG completo com sensor → extract → validate → transform → dbt → quality → load
- [ ] DAG factory pattern
- [ ] Step Functions state machine funcional
- [ ] Go scheduler com DAG execution
- [ ] Docker Compose Airflow stack
- [ ] Commit: `feat(data-engineering-05): pipeline orchestration`
