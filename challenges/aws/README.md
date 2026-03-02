# Zero to Hero — AWS Cloud Challenge (Go + Java)

> **Programa de especialização progressiva** em serviços AWS, arquitetura cloud e uso
> avançado de SDKs, com aplicações implementadas em **Go (Gin)** e **Java (Spring Boot)**
> usando **LocalStack** para desenvolvimento local com custo zero.

---

## 1. Resumo Executivo

Este programa transforma um desenvolvedor em **especialista em AWS Cloud**, focando nos **serviços**, **arquitetura**, **SDK** e **padrões de design cloud-native** — NÃO em IaC/Terraform (coberto pelo [track de Terraform](../terraform/README.md)).

**Domínio escolhido:** Sistema de **Processamento de Pedidos (OrderFlow)** — uma plataforma e-commerce que exercita todos os serviços AWS core através de código de aplicação.

**Por que OrderFlow?**
- Cobre **todos os serviços AWS core**: networking, compute, storage, messaging, serverless, observabilidade
- Exige **uso profundo de SDKs** (Go aws-sdk-go-v2 + Java AWS SDK v2) para interagir com serviços
- Exercita **decisões arquiteturais** reais: DynamoDB vs RDS, SQS vs EventBridge, Lambda vs ECS
- Relevante para **e-commerce, marketplaces, fintechs** (portfólio forte)
- Permite progressão natural: do primeiro `aws s3 ls` até plataforma production-ready
- Perguntas de entrevista Staff/Principal frequentemente envolvem design de sistemas na AWS
- **100% local com LocalStack** — custo zero, iteração rápida

**Foco deste track vs track Terraform:**

| Aspecto | Este Track (AWS Cloud) | Track Terraform |
|---------|----------------------|-----------------|
| **Objetivo** | Entender e usar serviços AWS | Provisionar infra como código |
| **Ferramenta principal** | AWS SDK (Go + Java) + AWS CLI | Terraform HCL |
| **Código predominante** | Go, Java, Shell | HCL (.tf) |
| **Decisões-chave** | Qual serviço usar e por quê | Como modularizar e testar IaC |
| **Testes** | Testes de aplicação com AWS services | terraform test, Terratest |
| **Segurança** | IAM policies, encryption via SDK | IAM resources em HCL |
| **Output** | Aplicação cloud-native funcionando | Infraestrutura provisionada |

---

## 2. Stack Tecnológica

### Aplicação — Go

| Categoria | Ferramenta |
|-----------|-----------|
| **Linguagem** | Go 1.24+ |
| **Framework Web** | Gin |
| **AWS SDK** | `aws-sdk-go-v2` (modular) |
| **Config** | `caarlos0/env` |
| **Testes** | `testing` stdlib + testify + testcontainers-go |
| **Logging** | `slog` (structured, JSON) |
| **Build** | Docker multi-stage |

### Aplicação — Java

| Categoria | Ferramenta |
|-----------|-----------|
| **Linguagem** | Java 25 (LTS) |
| **Framework Web** | Spring Boot 3.x + Spring Cloud AWS 3.x |
| **AWS SDK** | AWS SDK for Java v2 + Spring Cloud AWS |
| **Config** | `application.yml` + profiles |
| **Testes** | JUnit 5 + Testcontainers + LocalStack module |
| **Logging** | SLF4J + Logback (JSON) |
| **Build** | Maven + Docker multi-stage |

### Tooling

| Categoria | Ferramenta |
|-----------|-----------|
| **Emulador AWS** | LocalStack (Community Edition) |
| **CLI** | AWS CLI v2 |
| **Container** | Docker + Docker Compose |
| **Observabilidade** | CloudWatch (Embedded Metrics) + X-Ray + OpenTelemetry |

---

## 3. Domínio — OrderFlow (Processamento de Pedidos)

### 3.1 Descrição

Sistema de processamento de pedidos que:
- Recebe **pedidos** via API REST (Go/Java)
- Armazena pedidos no **DynamoDB** (com single-table design)
- Armazena **comprovantes/notas fiscais** no **S3** (com lifecycle policies)
- Publica eventos de pedido no **SQS** para processamento assíncrono
- Notifica stakeholders via **SNS** (fan-out)
- Orquestra fluxo de pagamento via **Step Functions**
- Processa pagamentos via **Lambda** (event-driven)
- Roteia eventos via **EventBridge** (event bus)
- Expõe dashboards via **CloudWatch** (métricas, logs, alarms)
- Protege dados com **KMS** (encryption at rest e in transit)
- Gerencia credenciais via **Secrets Manager**
- Roda em containers via **ECS Fargate** dentro de **VPC** isolada

### 3.2 Arquitetura AWS

```
┌─ Internet ─────────────────────────────────────────────────────────────┐
│                                                                         │
│   Client → Route 53 (DNS) → CloudFront (CDN) → ALB (HTTPS)            │
│                                                                         │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │
┌─ VPC (10.0.0.0/16) ────────────┼───────────────────────────────────────┐
│                                 │                                       │
│  ┌─ Public Subnets ────────────┼─────────┐                             │
│  │   ALB listeners    NAT Gateway         │                             │
│  └─────────────────────────────┬─────────┘                             │
│                                │                                        │
│  ┌─ Private Subnets ──────────┼──────────────────────────────────────┐ │
│  │                             │                                      │ │
│  │  ┌─────────────────────────────────────────────┐                  │ │
│  │  │         ECS Cluster (Fargate)                │                  │ │
│  │  │                                              │                  │ │
│  │  │  ┌──────────────┐   ┌──────────────┐        │                  │ │
│  │  │  │ OrderFlow Go │   │ OrderFlow    │        │                  │ │
│  │  │  │ (Gin API)    │   │ Java (Spring)│        │                  │ │
│  │  │  │ :8080        │   │ :8080        │        │                  │ │
│  │  │  └──────┬───────┘   └──────┬───────┘        │                  │ │
│  │  └─────────┼──────────────────┼─────────────────┘                  │ │
│  │            │                  │                                     │ │
│  │  ┌─────────┴──────────────────┴──────────────────────────────┐    │ │
│  │  │                  AWS Services via SDK                      │    │ │
│  │  │                                                            │    │ │
│  │  │  DynamoDB ←→ S3 ←→ SQS ←→ SNS ←→ EventBridge            │    │ │
│  │  │       │                │                                   │    │ │
│  │  │       │          ┌─────┴──────┐                           │    │ │
│  │  │       │          │  Lambda    │                           │    │ │
│  │  │       │          │ (Payment)  │                           │    │ │
│  │  │       │          └─────┬──────┘                           │    │ │
│  │  │       │                │                                   │    │ │
│  │  │  ┌────┴────┐    ┌─────┴────────┐                         │    │ │
│  │  │  │  DAX    │    │Step Functions│                         │    │ │
│  │  │  │ (cache) │    │(orchestrate) │                         │    │ │
│  │  │  └─────────┘    └──────────────┘                         │    │ │
│  │  └────────────────────────────────────────────────────────────┘    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌─ Segurança ──────────────────────────────────────────────────────┐  │
│  │ IAM Roles │ KMS │ Secrets Manager │ VPC Endpoints │ Flow Logs    │  │
│  │ CloudTrail │ GuardDuty │ Security Hub │ AWS Config               │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌─ Observabilidade ───────────────────────────────────────────────┐  │
│  │ CloudWatch Logs │ CloudWatch Metrics (EMF) │ CloudWatch Alarms  │  │
│  │ X-Ray Tracing │ CloudWatch Dashboards │ CloudWatch Insights     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Serviços AWS Exercitados

| Serviço | Uso no OrderFlow | Nível |
|---------|-----------------|-------|
| **VPC** | Rede isolada, subnets públicas/privadas, SGs, NACLs | L1 |
| **ALB** | Load balancer para APIs, health checks, path routing | L1 |
| **ECS Fargate** | Container runtime para Go/Java APIs | L2 |
| **DynamoDB** | Storage de pedidos (single-table, GSI, Streams) | L3 |
| **S3** | Armazenamento de comprovantes, lifecycle, presigned URLs | L3 |
| **SQS** | Fila de processamento (standard/FIFO, DLQ) | L4 |
| **SNS** | Notificações fan-out, filtering | L4 |
| **Lambda** | Payment processor (event-driven, triggered by SQS) | L4 |
| **EventBridge** | Event bus, regras de roteamento, transformação | L4 |
| **Step Functions** | Orquestração do fluxo de pagamento (saga pattern) | L4 |
| **API Gateway** | API HTTP/REST com Lambda backend (alternativa a ALB) | L4 |
| **IAM** | Roles, policies, ABAC, permission boundaries | L5 |
| **KMS** | Envelope encryption, key rotation, grants | L5 |
| **Secrets Manager** | Credenciais, API keys, rotação automática | L5 |
| **CloudTrail** | Audit trail de ações | L5 |
| **GuardDuty** | Detecção de ameaças | L5 |
| **CloudWatch** | Logs, métricas (EMF), alarms, dashboards, Insights | L6 |
| **X-Ray** | Distributed tracing, service map | L6 |
| **Route 53** | DNS, health checks, roteamento (failover, weighted) | L7 |
| **CloudFront** | CDN para assets estáticos, presigned URLs | L7 |

### 3.4 Entidades do Domínio

```
Customer (1) ──── (N) Order (1) ──── (N) OrderItem
                        │
                        ├── status: PENDING → PROCESSING → PAID → SHIPPED → DELIVERED
                        │                                        └→ CANCELLED
                        └── payment_status: AWAITING → PROCESSING → APPROVED
                                                                 └→ REJECTED
```

| Entidade | Campos Principais | Storage |
|----------|------------------|---------|
| **Customer** | `id` (UUID), `name`, `email`, `phone`, `created_at` | DynamoDB |
| **Order** | `id` (UUID), `customer_id`, `status`, `payment_status`, `total`, `currency`, `items[]`, `created_at` | DynamoDB |
| **OrderItem** | `product_id`, `product_name`, `quantity`, `unit_price`, `subtotal` | Embedded in Order |
| **OrderEvent** | `order_id`, `event_type`, `timestamp`, `payload` | SQS / EventBridge |
| **Receipt** | `order_id`, `file_key`, `content_type`, `generated_at` | S3 |
| **Payment** | `order_id`, `amount`, `method`, `status`, `processed_at` | DynamoDB (via Lambda) |

---

## 4. Contrato Funcional (API REST — Go + Java)

### 4.1 Endpoints

```
Base path: /api/v1

── Orders ──
POST   /api/v1/orders                                → Criar pedido
GET    /api/v1/orders/{id}                            → Buscar por ID
GET    /api/v1/orders?customer_id=X&status=PENDING&page=0&size=20 → Listar com filtros
PUT    /api/v1/orders/{id}/status                     → Atualizar status
DELETE /api/v1/orders/{id}                            → Cancelar pedido

── Receipts ──
POST   /api/v1/orders/{orderId}/receipts              → Upload de comprovante
GET    /api/v1/orders/{orderId}/receipts               → Listar comprovantes
GET    /api/v1/orders/{orderId}/receipts/{key}/url     → Gerar presigned URL

── Notifications ──
POST   /api/v1/notifications/subscribe                → Inscrever em topic
GET    /api/v1/notifications/topics                    → Listar topics

── Payments ──
POST   /api/v1/orders/{orderId}/pay                   → Iniciar pagamento (envia para SQS → Lambda)
GET    /api/v1/orders/{orderId}/payment                → Consultar status do pagamento

── Health ──
GET    /api/v1/health                                  → Liveness
GET    /api/v1/health/ready                            → Readiness (DynamoDB + SQS + S3)
```

### 4.2 Payloads

**CreateOrderRequest:**
```json
{
  "customer_id": "uuid",
  "currency": "BRL",
  "items": [
    {
      "product_id": "uuid",
      "product_name": "Wireless Mouse",
      "quantity": 2,
      "unit_price": 49.90
    }
  ]
}
```

**OrderResponse:**
```json
{
  "id": "uuid",
  "customer_id": "uuid",
  "status": "PENDING",
  "payment_status": "AWAITING",
  "total": 99.80,
  "currency": "BRL",
  "items": [...],
  "created_at": "2026-03-01T10:30:00Z",
  "updated_at": "2026-03-01T10:30:00Z"
}
```

**Erro (RFC 9457 ProblemDetail):**
```json
{
  "type": "https://orderflow.example.com/errors/order-not-found",
  "title": "Pedido não encontrado",
  "status": 404,
  "detail": "Pedido com ID xyz não existe",
  "instance": "/api/v1/orders/xyz"
}
```

---

## 5. Trilha Zero to Hero (Nível 0 ao Nível 7)

| Nível | Foco | Semana | Serviços AWS |
|---|---|---|---|
| [Level 0](00-foundations.md) | AWS Foundations + SDK + LocalStack | Semana 1-2 | IAM, STS, S3, DynamoDB (básico) |
| [Level 1](01-networking-content-delivery.md) | Networking + Content Delivery | Semana 3-4 | VPC, ALB/NLB, Route 53, CloudFront, SGs |
| [Level 2](02-compute-containers.md) | Compute + Containers | Semana 5-6 | ECS Fargate, ECR, App Runner, Auto Scaling |
| [Level 3](03-storage-databases.md) | Storage + Databases | Semana 7-9 | S3, DynamoDB, RDS/Aurora, ElastiCache, DAX |
| [Level 4](04-serverless-event-driven.md) | Serverless + Event-Driven | Semana 10-12 | Lambda, API GW, SQS, SNS, EventBridge, Step Functions |
| [Level 5](05-security-compliance.md) | Security + Compliance | Semana 13-14 | IAM, KMS, Secrets Manager, CloudTrail, GuardDuty |
| [Level 6](06-observability-resilience.md) | Observability + Resilience | Semana 15-16 | CloudWatch, X-Ray, OpenTelemetry, DR patterns |
| [Level 7](07-capstone-full-platform.md) | Capstone — Plataforma Completa | Semana 17-20 | Todos os serviços integrados |

---

## 6. Estrutura do Projeto

```
orderflow/
├── app-go/                            # Aplicação Go (Gin + AWS SDK v2)
│   ├── cmd/api/main.go
│   ├── internal/
│   │   ├── config/                    # Config via env vars
│   │   ├── domain/                    # Entities, business rules
│   │   ├── handler/                   # HTTP handlers (Gin)
│   │   ├── service/                   # Business logic
│   │   ├── repository/               # DynamoDB, S3 implementations
│   │   ├── messaging/                # SQS producer, SNS publisher
│   │   ├── middleware/               # Auth, logging, tracing
│   │   └── aws/                      # AWS client factories, helpers
│   ├── Dockerfile
│   ├── go.mod
│   └── Makefile
│
├── app-java/                          # Aplicação Java (Spring Boot + Spring Cloud AWS)
│   ├── src/main/java/.../orderflow/
│   │   ├── api/rest/v1/              # Controllers
│   │   ├── domain/
│   │   │   ├── entity/               # DynamoDB entities (@DynamoDbBean)
│   │   │   ├── service/              # Business logic
│   │   │   └── repository/           # DynamoDB, S3
│   │   ├── infrastructure/
│   │   │   ├── aws/                  # AWS client configs
│   │   │   ├── messaging/            # SQS listener, SNS publisher
│   │   │   └── config/               # Spring configs
│   │   └── OrderFlowApplication.java
│   ├── src/main/resources/
│   │   ├── application.yml
│   │   └── application-localstack.yml
│   ├── Dockerfile
│   └── pom.xml
│
├── lambda/                            # Lambda functions
│   ├── payment-processor/
│   │   ├── go/main.go                 # Lambda em Go
│   │   └── java/Handler.java         # Lambda em Java
│   └── notification-sender/
│       ├── go/main.go
│       └── java/Handler.java
│
├── scripts/
│   ├── setup-localstack.sh           # Cria recursos via AWS CLI
│   ├── seed-data.sh                  # Popula dados de teste
│   └── test-e2e.sh                   # Testes end-to-end
│
├── docker-compose.yml                # LocalStack + Apps
├── Makefile                          # Comandos raiz
└── README.md
```

---

## 7. Equivalências SDK — Go vs Java

| Conceito | Go (`aws-sdk-go-v2`) | Java (AWS SDK v2 + Spring Cloud AWS) |
|----------|---------------------|--------------------------------------|
| **Config** | `config.LoadDefaultConfig(ctx)` | `AwsCredentialsProvider` + `Region` |
| **LocalStack** | `aws.EndpointResolverWithOptionsFunc` | `spring.cloud.aws.endpoint` |
| **Credentials** | `credentials.NewStaticCredentialsProvider` | `StaticCredentialsProvider.create()` |
| **DynamoDB** | `dynamodb.NewFromConfig(cfg)` | `DynamoDbEnhancedClient` + `@DynamoDbBean` |
| **S3 Upload** | `s3.PutObject()` | `S3Client.putObject()` / `S3TransferManager` |
| **S3 Presigned** | `s3.NewPresignClient(client)` | `S3Presigner` |
| **SQS Send** | `sqs.SendMessage()` | `SqsTemplate.send()` |
| **SQS Receive** | goroutine + `ReceiveMessage` | `@SqsListener` |
| **SNS Publish** | `sns.Publish()` | `SnsTemplate.sendNotification()` |
| **Lambda Handler** | `lambda.Start(handler)` | `RequestHandler<SQSEvent, Void>` |
| **EventBridge** | `eventbridge.PutEvents()` | `EventBridgeClient.putEvents()` |
| **Step Functions** | `sfn.StartExecution()` | `SfnClient.startExecution()` |
| **KMS Encrypt** | `kms.Encrypt()` / `kms.GenerateDataKey()` | `KmsClient.encrypt()` |
| **Secrets** | `secretsmanager.GetSecretValue()` | `SecretsManagerClient` / Spring integration |
| **X-Ray** | `xray.Configure()` | Spring Cloud AWS + `aws-xray-recorder-sdk` |
| **CloudWatch** | `cloudwatch.PutMetricData()` | `CloudWatchClient` / Micrometer |

---

## 8. Docker Compose — LocalStack

```yaml
# docker-compose.yml
services:
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      - SERVICES=s3,sqs,sns,dynamodb,lambda,ecs,iam,kms,ssm,secretsmanager,cloudwatch,logs,sts,events,stepfunctions,route53,cloudfront
      - DEFAULT_REGION=us-east-1
      - DEBUG=0
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "localstack-data:/var/lib/localstack"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4566/_localstack/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  orderflow-go:
    build: ./app-go
    ports:
      - "8080:8080"
    environment:
      - ENVIRONMENT=local
      - AWS_ENDPOINT=http://localstack:4566
      - AWS_REGION=us-east-1
      - AWS_ACCESS_KEY_ID=test
      - AWS_SECRET_ACCESS_KEY=test
    depends_on:
      localstack:
        condition: service_healthy

  orderflow-java:
    build: ./app-java
    ports:
      - "8081:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=localstack
      - AWS_ENDPOINT=http://localstack:4566
      - AWS_REGION=us-east-1
    depends_on:
      localstack:
        condition: service_healthy

volumes:
  localstack-data:
```

---

## 9. Referência — Well-Architected Framework (6 Pilares)

Este track está alinhado aos 6 pilares do AWS Well-Architected Framework:

| Pilar | Onde é exercitado |
|-------|-------------------|
| **Operational Excellence** | L0 (tooling), L6 (observability), L7 (runbooks) |
| **Security** | L5 (IAM, KMS, secrets), L1 (VPC, SGs) |
| **Reliability** | L4 (DLQ, retry), L6 (DR, multi-AZ), L7 (auto-scaling) |
| **Performance Efficiency** | L2 (right-sizing), L3 (DAX, caching), L4 (async) |
| **Cost Optimization** | L7 (Fargate Spot, right-sizing, lifecycle policies) |
| **Sustainability** | L2 (Graviton), L7 (efficient architectures) |

---

## 10. Rubrica de Avaliação (0–100)

| Critério | Peso | 0-25 | 26-50 | 51-75 | 76-100 |
|---|---|---|---|---|---|
| **AWS Services** | 25% | Usa 1-2 serviços | Usa 5+ serviços corretamente | Integra 10+ serviços com patterns | Domina trade-offs e escolhe serviços ideais |
| **SDK Usage** | 20% | Código funcional básico | Error handling + retries | Pagination, presigned URLs, streaming | Production-ready: caching, circuit breaker, metrics |
| **Architecture** | 20% | Monolítico, sem patterns | Event-driven básico | Saga, fan-out, CQRS | Well-Architected review completo |
| **Security** | 15% | Sem IAM | Roles básicas | Least privilege + KMS + secrets | Zero trust, ABAC, compliance |
| **Observability** | 10% | Sem logs | Structured logs | Métricas + tracing + alarms | Dashboards + Insights + SLI/SLO |
| **Resilience** | 10% | Sem DR | DLQ + retry | Multi-AZ + auto-scaling | DR strategy implementada |

---

## 11. Pré-requisitos

- [ ] Docker Desktop instalado
- [ ] Go 1.24+ instalado (`go version`)
- [ ] Java 25 (LTS) instalado (`java --version`)
- [ ] Maven wrapper configurado
- [ ] AWS CLI v2 instalado (`aws --version`)
- [ ] LocalStack CLI instalado (`pip install localstack` ou `brew install localstack`)
- [ ] Noções básicas de HTTP/REST
- [ ] Noções básicas de containers (Docker)

---

## 12. Próximos Passos

1. Comece pelo [Level 0 — AWS Foundations](00-foundations.md)
2. Siga a ordem dos níveis — cada nível depende do anterior
3. Implemente as aplicações Go + Java com uso profundo dos SDKs
4. Valide tudo com LocalStack antes de pensar em AWS real
5. Documente decisões arquiteturais em `DECISIONS.md`
6. Após este track, complemente com o [track de Terraform](../terraform/README.md) para IaC
