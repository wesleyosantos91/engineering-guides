# Level 00 — Foundations & Environment Setup

> **Objetivo:** Configurar o ambiente completo de data engineering local, entender
> os conceitos fundamentais e construir o primeiro pipeline mínimo.

**Referência:** [Data Engineering — README](../../.docs/data-engineering/README.md)

**Pré-requisito:** Nenhum (ponto de partida da trilha).

---

## Contexto

Antes de construir pipelines de dados, é preciso entender o **ecossistema de data engineering**,
configurar o ambiente local com **LocalStack** (simulação de serviços AWS) e validar
que tudo funciona end-to-end com um pipeline mínimo.

---

## Parte 1 — ADR (1 obrigatório)

### ADR-001: Local Development Stack

**Arquivo:** `docs/adrs/ADR-001-local-dev-stack.md`

**Context:** Precisamos de um ambiente de desenvolvimento que simule AWS sem custos.

**Options:**
1. **LocalStack** — simula 70+ serviços AWS localmente
2. **MinIO + Kafka + PostgreSQL** — componentes open-source individuais
3. **AWS Free Tier** — usar AWS real com limites gratuitos
4. **Docker Compose puro** — sem simulação de cloud

**Decision Drivers:**
- Custo zero para desenvolvimento
- Paridade máxima com AWS production
- Simplicidade de setup
- Suporte a S3, Kinesis, DynamoDB, SQS, Lambda, Step Functions

**Critérios de aceite:**
- [ ] ADR completo com consequences documentadas
- [ ] Custo: $0 para desenvolvimento local

---

## Parte 2 — Diagrama DrawIO (1 obrigatório)

**Arquivo:** `docs/diagrams/00-dev-environment.drawio`

Diagrama do ambiente local:

```
┌──────────────────────────────────────────────────────────┐
│                    Docker Compose                        │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  LocalStack  │  │  PostgreSQL  │  │    Kafka     │   │
│  │              │  │    16        │  │   (KRaft)    │   │
│  │  S3          │  │              │  │              │   │
│  │  Kinesis     │  │  warehouse   │  │  CDC events  │   │
│  │  DynamoDB    │  │  + sources   │  │  streaming   │   │
│  │  SQS / SNS  │  │              │  │              │   │
│  │  Lambda      │  │              │  │              │   │
│  │  Step Func.  │  │              │  │              │   │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │   Trino /    │  │   Jupyter    │  │   MinIO      │   │
│  │   DuckDB     │  │  Notebook    │  │  (alt. S3)   │   │
│  │              │  │              │  │              │   │
│  │  SQL on S3   │  │  exploration │  │  object store│   │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
└──────────────────────────────────────────────────────────┘
```

**Critérios de aceite:**
- [ ] Todos os componentes do ambiente local representados
- [ ] Portas e endpoints documentados

---

## Parte 3 — Implementação

### 3.1 — Docker Compose Base

**Arquivo:** `infra/docker-compose.yml`

```yaml
services:
  localstack:
    image: localstack/localstack:3
    ports:
      - "4566:4566"          # Gateway
      - "4510-4559:4510-4559" # Service ports
    environment:
      - SERVICES=s3,kinesis,dynamodb,sqs,sns,lambda,stepfunctions,iam
      - DEFAULT_REGION=us-east-1
    volumes:
      - "./localstack/init-aws.sh:/etc/localstack/init/ready.d/init-aws.sh"
      - localstack_data:/var/lib/localstack

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: datawarehouse
      POSTGRES_USER: dataeng
      POSTGRES_PASSWORD: dataeng
    ports:
      - "5432:5432"

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      CLUSTER_ID: data-eng-cluster-001
    ports:
      - "9092:9092"

volumes:
  localstack_data:
```

### 3.2 — Init Script (provisionar recursos AWS)

**Arquivo:** `infra/localstack/init-aws.sh`

```bash
#!/bin/bash
echo "=== Initializing LocalStack resources ==="

# S3 Buckets (data lake zones)
awslocal s3 mb s3://data-lake-raw
awslocal s3 mb s3://data-lake-curated
awslocal s3 mb s3://data-lake-refined

# Kinesis Streams
awslocal kinesis create-stream --stream-name events-stream --shard-count 2

# DynamoDB (metadata)
awslocal dynamodb create-table \
  --table-name pipeline-state \
  --attribute-definitions AttributeName=pipeline_id,AttributeType=S \
  --key-schema AttributeName=pipeline_id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# SQS (dead letter queue)
awslocal sqs create-queue --queue-name data-pipeline-dlq

echo "=== LocalStack initialization complete ==="
```

### 3.3 — Python: Primeiro Pipeline

**Objetivo:** Ler um CSV, transformar para Parquet, upload para S3 (LocalStack).

```python
# python/etl/hello_pipeline.py
import boto3
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
from io import BytesIO

def create_s3_client():
    return boto3.client(
        "s3",
        endpoint_url="http://localhost:4566",
        aws_access_key_id="test",
        aws_secret_access_key="test",
        region_name="us-east-1",
    )

def ingest_csv_to_parquet(csv_path: str, bucket: str, key: str):
    """Ingestion: CSV → Parquet → S3 raw zone."""
    df = pd.read_csv(csv_path)
    table = pa.Table.from_pandas(df)
    
    buffer = BytesIO()
    pq.write_table(table, buffer, compression="snappy")
    buffer.seek(0)
    
    s3 = create_s3_client()
    s3.put_object(Bucket=bucket, Key=key, Body=buffer.getvalue())
    
    print(f"✓ Uploaded {len(df)} rows to s3://{bucket}/{key}")
    return len(df)

if __name__ == "__main__":
    ingest_csv_to_parquet(
        csv_path="data/sample.csv",
        bucket="data-lake-raw",
        key="hello/sample.parquet",
    )
```

### 3.4 — Go: S3 Client & Health Check

**Objetivo:** Verificar conectividade com LocalStack e listar buckets.

```go
// go/cmd/healthcheck/main.go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

func main() {
    cfg, err := config.LoadDefaultConfig(context.TODO(),
        config.WithRegion("us-east-1"),
    )
    if err != nil {
        log.Fatal(err)
    }

    client := s3.NewFromConfig(cfg, func(o *s3.Options) {
        o.BaseEndpoint = aws.String("http://localhost:4566")
        o.UsePathStyle = true
    })

    result, err := client.ListBuckets(context.TODO(), &s3.ListBucketsInput{})
    if err != nil {
        log.Fatal(err)
    }

    fmt.Println("=== Data Lake Buckets ===")
    for _, bucket := range result.Buckets {
        fmt.Printf("  📦 %s\n", *bucket.Name)
    }
}
```

### 3.5 — Java: S3 Client (Spring Boot)

**Objetivo:** Spring Boot app que lista buckets e faz upload.

```java
// java/src/.../S3HealthCheck.java
@Component
public class S3HealthCheck implements CommandLineRunner {

    private final S3Client s3;

    public S3HealthCheck() {
        this.s3 = S3Client.builder()
            .endpointOverride(URI.create("http://localhost:4566"))
            .region(Region.US_EAST_1)
            .forcePathStyle(true)
            .build();
    }

    @Override
    public void run(String... args) {
        var buckets = s3.listBuckets().buckets();
        System.out.println("=== Data Lake Buckets ===");
        buckets.forEach(b -> 
            System.out.printf("  📦 %s%n", b.name()));
    }
}
```

---

## Critérios de Aceite

- [ ] Docker Compose sobe: LocalStack + PostgreSQL + Kafka
- [ ] S3 buckets criados (raw, curated, refined)
- [ ] Kinesis stream criado
- [ ] DynamoDB table criada
- [ ] Python: CSV → Parquet → S3 funcional
- [ ] Go: lista buckets do LocalStack
- [ ] Java: lista buckets do LocalStack
- [ ] ADR-001 completo
- [ ] DrawIO do ambiente local

---

## Definição de Pronto (DoD)

- [ ] 1 ADR (local dev stack)
- [ ] 1 DrawIO (dev environment)
- [ ] Docker Compose funcional
- [ ] Hello pipeline Python end-to-end
- [ ] Health check Go + Java
- [ ] Commit: `feat(data-engineering-00): foundations & setup`
