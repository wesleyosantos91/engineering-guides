# Level 0 — AWS Cloud Foundations

> **Objetivo:** Entender os conceitos fundamentais da AWS, configurar o ambiente de
> desenvolvimento com LocalStack, e fazer as primeiras interações programáticas com
> serviços AWS usando AWS CLI, Go SDK e Java SDK.

---

## Objetivo de Aprendizado

- Entender o modelo de responsabilidade compartilhada da AWS
- Conhecer regiões, AZs e o modelo de pricing
- Configurar AWS CLI com LocalStack
- Configurar AWS SDK v2 para Go e Java com suporte a LocalStack
- Entender a credential provider chain
- Fazer operações básicas em S3, DynamoDB e SQS via SDK
- Compreender os 6 pilares do Well-Architected Framework

---

## Escopo Funcional

```
Level 0 — O que vamos construir
═══════════════════════════════

  Developer Machine
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │   ┌─────────────┐     ┌──────────────────────────┐  │
  │   │  AWS CLI v2  │────►│                          │  │
  │   └─────────────┘     │                          │  │
  │                        │      LocalStack         │  │
  │   ┌─────────────┐     │    (Docker Container)    │  │
  │   │  Go App     │────►│                          │  │
  │   │  (SDK v2)   │     │   ┌───┐ ┌────┐ ┌───┐   │  │
  │   └─────────────┘     │   │S3 │ │DDB │ │SQS│   │  │
  │                        │   └───┘ └────┘ └───┘   │  │
  │   ┌─────────────┐     │                          │  │
  │   │  Java App   │────►│   ┌───┐ ┌────┐ ┌───┐   │  │
  │   │  (SDK v2)   │     │   │SNS│ │IAM │ │KMS│   │  │
  │   └─────────────┘     │   └───┘ └────┘ └───┘   │  │
  │                        │                          │  │
  │                        └──────────────────────────┘  │
  │                                :4566                  │
  └──────────────────────────────────────────────────────┘
```

---

## Conceitos Fundamentais

### AWS Global Infrastructure

```
┌─ AWS Cloud ────────────────────────────────────────────────────────┐
│                                                                     │
│  Region: us-east-1 (N. Virginia)                                   │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │                                                            │    │
│  │  ┌─ AZ: us-east-1a ──┐  ┌─ AZ: us-east-1b ──┐           │    │
│  │  │  Data Center(s)    │  │  Data Center(s)    │  ...      │    │
│  │  │  Subnet: 10.0.1.0  │  │  Subnet: 10.0.2.0  │           │    │
│  │  └────────────────────┘  └────────────────────┘           │    │
│  │                                                            │    │
│  │  Serviços Regionais: S3, DynamoDB, SQS, SNS, Lambda, ECS │    │
│  │  Serviços Globais: IAM, Route 53, CloudFront, WAF        │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  Region: eu-west-1 (Ireland)                                       │
│  ┌──────────────────────────────────────────────────┐              │
│  │  AZs: eu-west-1a, eu-west-1b, eu-west-1c        │              │
│  └──────────────────────────────────────────────────┘              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Shared Responsibility Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                RESPONSABILIDADE DO CLIENTE                          │
│                                                                     │
│  Dados do cliente │ Platform & Application Management              │
│  Identity & Access Management │ OS, Network, Firewall Config       │
│  Client-side encryption │ Server-side encryption                    │
│  Network traffic protection │ Customer data encryption             │
├─────────────────────────────────────────────────────────────────────┤
│                RESPONSABILIDADE DA AWS                              │
│                                                                     │
│  Software: Compute │ Storage │ Database │ Networking               │
│  Hardware/Infrastructure: Regions │ AZs │ Edge Locations           │
│  Global infrastructure │ Physical security                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Tarefas

### Tarefa 0.1 — Configurar LocalStack com Docker Compose

```yaml
# docker-compose.yml
services:
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      - SERVICES=s3,sqs,sns,dynamodb,lambda,iam,kms,sts,secretsmanager,cloudwatch,logs,events,stepfunctions
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

volumes:
  localstack-data:
```

**Comandos de verificação:**

```bash
# Iniciar LocalStack
docker compose up -d

# Verificar serviços disponíveis
curl http://localhost:4566/_localstack/health | python3 -m json.tool

# Configurar AWS CLI para LocalStack
aws configure set aws_access_key_id test --profile localstack
aws configure set aws_secret_access_key test --profile localstack
aws configure set region us-east-1 --profile localstack

# Alias para facilitar uso
alias awslocal='aws --endpoint-url=http://localhost:4566'

# Teste rápido: listar buckets S3
awslocal s3 ls

# Teste: listar tabelas DynamoDB
awslocal dynamodb list-tables

# Teste: listar filas SQS
awslocal sqs list-queues
```

### Tarefa 0.2 — Primeiras Operações com AWS CLI

```bash
#!/bin/bash
# scripts/setup-localstack.sh — Cria recursos iniciais do OrderFlow

set -euo pipefail

ENDPOINT="http://localhost:4566"
REGION="us-east-1"
AWS="aws --endpoint-url=$ENDPOINT --region=$REGION"

echo "=== Creating S3 Bucket ==="
$AWS s3 mb s3://orderflow-local-receipts
$AWS s3api put-bucket-versioning \
  --bucket orderflow-local-receipts \
  --versioning-configuration Status=Enabled

echo "=== Creating DynamoDB Table ==="
$AWS dynamodb create-table \
  --table-name orderflow-local-orders \
  --attribute-definitions \
    AttributeName=PK,AttributeType=S \
    AttributeName=SK,AttributeType=S \
    AttributeName=GSI1PK,AttributeType=S \
    AttributeName=GSI1SK,AttributeType=S \
  --key-schema \
    AttributeName=PK,KeyType=HASH \
    AttributeName=SK,KeyType=RANGE \
  --global-secondary-indexes \
    '[{
      "IndexName": "GSI1",
      "KeySchema": [
        {"AttributeName": "GSI1PK", "KeyType": "HASH"},
        {"AttributeName": "GSI1SK", "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "ALL"}
    }]' \
  --billing-mode PAY_PER_REQUEST

echo "=== Creating SQS Queue ==="
$AWS sqs create-queue \
  --queue-name orderflow-local-order-events-dlq

DLQ_ARN=$($AWS sqs get-queue-attributes \
  --queue-url http://localhost:4566/000000000000/orderflow-local-order-events-dlq \
  --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)

$AWS sqs create-queue \
  --queue-name orderflow-local-order-events \
  --attributes "{
    \"RedrivePolicy\": \"{\\\"deadLetterTargetArn\\\":\\\"$DLQ_ARN\\\",\\\"maxReceiveCount\\\":\\\"3\\\"}\"
  }"

echo "=== Creating SNS Topic ==="
$AWS sns create-topic --name orderflow-local-order-notifications

echo "=== Verifying resources ==="
echo "S3 Buckets:"
$AWS s3 ls
echo "DynamoDB Tables:"
$AWS dynamodb list-tables
echo "SQS Queues:"
$AWS sqs list-queues
echo "SNS Topics:"
$AWS sns list-topics

echo "=== Setup complete! ==="
```

### Tarefa 0.3 — Credential Provider Chain e AWS Config (Go)

```go
// internal/aws/config.go
package aws

import (
	"context"
	"fmt"
	"log/slog"
	"os"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/credentials"
)

// NewAWSConfig creates an AWS config with LocalStack support.
//
// Credential Provider Chain (ordem de resolução):
// 1. Environment variables (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)
// 2. Shared credentials file (~/.aws/credentials)
// 3. IAM role for EC2/ECS (Instance Metadata / Task Role)
// 4. SSO credentials
//
// Em LocalStack, usamos static credentials (test/test).
func NewAWSConfig(ctx context.Context) (aws.Config, error) {
	opts := []func(*config.LoadOptions) error{
		config.WithRegion(getEnvOrDefault("AWS_REGION", "us-east-1")),
	}

	// Se tiver endpoint customizado (LocalStack), configura
	endpoint := os.Getenv("AWS_ENDPOINT")
	if endpoint != "" {
		slog.Info("Using custom AWS endpoint", "endpoint", endpoint)
		opts = append(opts,
			config.WithCredentialsProvider(
				credentials.NewStaticCredentialsProvider("test", "test", ""),
			),
		)
	}

	cfg, err := config.LoadDefaultConfig(ctx, opts...)
	if err != nil {
		return aws.Config{}, fmt.Errorf("unable to load AWS config: %w", err)
	}

	return cfg, nil
}

// EndpointResolver returns a custom endpoint for LocalStack.
// Use with service-specific clients.
func EndpointResolver() *string {
	if endpoint := os.Getenv("AWS_ENDPOINT"); endpoint != "" {
		return &endpoint
	}
	return nil
}

func getEnvOrDefault(key, defaultValue string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return defaultValue
}
```

```go
// internal/aws/clients.go
package aws

import (
	"context"
	"log/slog"

	awsconfig "github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
	"github.com/aws/aws-sdk-go-v2/service/s3"
	"github.com/aws/aws-sdk-go-v2/service/sqs"
	"github.com/aws/aws-sdk-go-v2/service/sns"
)

// Clients holds all AWS service clients.
type Clients struct {
	DynamoDB *dynamodb.Client
	S3       *s3.Client
	SQS      *sqs.Client
	SNS      *sns.Client
}

// NewClients creates all AWS service clients from a shared config.
func NewClients(ctx context.Context) (*Clients, error) {
	cfg, err := NewAWSConfig(ctx)
	if err != nil {
		return nil, err
	}

	endpoint := EndpointResolver()
	opts := func(o *awsconfig.Config) {}
	_ = opts

	clients := &Clients{
		DynamoDB: newDynamoDBClient(cfg, endpoint),
		S3:       newS3Client(cfg, endpoint),
		SQS:      newSQSClient(cfg, endpoint),
		SNS:      newSNSClient(cfg, endpoint),
	}

	slog.Info("AWS clients initialized",
		"region", cfg.Region,
		"endpoint", endpoint,
	)

	return clients, nil
}

func newDynamoDBClient(cfg awsconfig.Config, endpoint *string) *dynamodb.Client {
	opts := []func(*dynamodb.Options){}
	if endpoint != nil {
		opts = append(opts, func(o *dynamodb.Options) {
			o.BaseEndpoint = endpoint
		})
	}
	return dynamodb.NewFromConfig(cfg, opts...)
}

func newS3Client(cfg awsconfig.Config, endpoint *string) *s3.Client {
	opts := []func(*s3.Options){}
	if endpoint != nil {
		opts = append(opts, func(o *s3.Options) {
			o.BaseEndpoint = endpoint
			o.UsePathStyle = true // Required for LocalStack
		})
	}
	return s3.NewFromConfig(cfg, opts...)
}

func newSQSClient(cfg awsconfig.Config, endpoint *string) *sqs.Client {
	opts := []func(*sqs.Options){}
	if endpoint != nil {
		opts = append(opts, func(o *sqs.Options) {
			o.BaseEndpoint = endpoint
		})
	}
	return sqs.NewFromConfig(cfg, opts...)
}

func newSNSClient(cfg awsconfig.Config, endpoint *string) *sns.Client {
	opts := []func(*sns.Options){}
	if endpoint != nil {
		opts = append(opts, func(o *sns.Options) {
			o.BaseEndpoint = endpoint
		})
	}
	return sns.NewFromConfig(cfg, opts...)
}
```

### Tarefa 0.4 — AWS Config e Clients (Java)

```java
// infrastructure/aws/AwsConfig.java
package com.orderflow.infrastructure.aws;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials;
import software.amazon.awssdk.auth.credentials.AwsCredentialsProvider;
import software.amazon.awssdk.auth.credentials.DefaultCredentialsProvider;
import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider;
import software.amazon.awssdk.regions.Region;

import java.net.URI;

/**
 * AWS SDK Configuration.
 *
 * Credential Provider Chain (ordem de resolução):
 * 1. Java System Properties (aws.accessKeyId, aws.secretAccessKey)
 * 2. Environment Variables (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)
 * 3. Web Identity Token (ECS/EKS)
 * 4. Shared credentials file (~/.aws/credentials)
 * 5. EC2 Instance Metadata / ECS Task Role
 */
@Configuration
public class AwsConfig {

    @Value("${aws.region:us-east-1}")
    private String region;

    @Bean
    public Region awsRegion() {
        return Region.of(region);
    }

    @Bean
    @Profile("!localstack")
    public AwsCredentialsProvider defaultCredentialsProvider() {
        // Uses the default credential provider chain (production)
        return DefaultCredentialsProvider.create();
    }

    @Bean
    @Profile("localstack")
    public AwsCredentialsProvider localstackCredentialsProvider() {
        return StaticCredentialsProvider.create(
            AwsBasicCredentials.create("test", "test")
        );
    }
}
```

```java
// infrastructure/aws/DynamoDbConfig.java
package com.orderflow.infrastructure.aws;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.auth.credentials.AwsCredentialsProvider;
import software.amazon.awssdk.enhanced.dynamodb.DynamoDbEnhancedClient;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.dynamodb.DynamoDbClient;

import java.net.URI;
import java.util.Optional;

@Configuration
public class DynamoDbConfig {

    @Value("${aws.endpoint:#{null}}")
    private String endpoint;

    @Bean
    public DynamoDbClient dynamoDbClient(Region region, AwsCredentialsProvider credentialsProvider) {
        var builder = DynamoDbClient.builder()
                .region(region)
                .credentialsProvider(credentialsProvider);

        Optional.ofNullable(endpoint)
                .map(URI::create)
                .ifPresent(builder::endpointOverride);

        return builder.build();
    }

    @Bean
    public DynamoDbEnhancedClient dynamoDbEnhancedClient(DynamoDbClient dynamoDbClient) {
        return DynamoDbEnhancedClient.builder()
                .dynamoDbClient(dynamoDbClient)
                .build();
    }
}
```

```java
// infrastructure/aws/S3Config.java
package com.orderflow.infrastructure.aws;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.auth.credentials.AwsCredentialsProvider;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;

import java.net.URI;
import java.util.Optional;

@Configuration
public class S3Config {

    @Value("${aws.endpoint:#{null}}")
    private String endpoint;

    @Bean
    public S3Client s3Client(Region region, AwsCredentialsProvider credentialsProvider) {
        var builder = S3Client.builder()
                .region(region)
                .credentialsProvider(credentialsProvider)
                .forcePathStyle(true); // Required for LocalStack

        Optional.ofNullable(endpoint)
                .map(URI::create)
                .ifPresent(builder::endpointOverride);

        return builder.build();
    }

    @Bean
    public S3Presigner s3Presigner(Region region, AwsCredentialsProvider credentialsProvider) {
        var builder = S3Presigner.builder()
                .region(region)
                .credentialsProvider(credentialsProvider);

        Optional.ofNullable(endpoint)
                .map(URI::create)
                .ifPresent(builder::endpointOverride);

        return builder.build();
    }
}
```

```yaml
# application-localstack.yml
aws:
  region: us-east-1
  endpoint: http://localhost:4566

orderflow:
  dynamodb:
    table-name: orderflow-local-orders
  s3:
    bucket-name: orderflow-local-receipts
  sqs:
    queue-url: http://localhost:4566/000000000000/orderflow-local-order-events
  sns:
    topic-arn: arn:aws:sns:us-east-1:000000000000:orderflow-local-order-notifications
```

### Tarefa 0.5 — Primeira Operação: S3 Upload/Download (Go)

```go
// internal/repository/receipt_store.go
package repository

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"log/slog"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/s3"
)

type ReceiptStore struct {
	client *s3.Client
	bucket string
}

func NewReceiptStore(client *s3.Client, bucket string) *ReceiptStore {
	return &ReceiptStore{client: client, bucket: bucket}
}

// Upload stores a receipt file in S3.
func (s *ReceiptStore) Upload(ctx context.Context, orderID string, data []byte, contentType string) (string, error) {
	key := fmt.Sprintf("receipts/%s/%d.pdf", orderID, time.Now().UnixMilli())

	_, err := s.client.PutObject(ctx, &s3.PutObjectInput{
		Bucket:      aws.String(s.bucket),
		Key:         aws.String(key),
		Body:        bytes.NewReader(data),
		ContentType: aws.String(contentType),
		Metadata: map[string]string{
			"order-id":   orderID,
			"uploaded-at": time.Now().UTC().Format(time.RFC3339),
		},
	})
	if err != nil {
		return "", fmt.Errorf("failed to upload receipt: %w", err)
	}

	slog.Info("Receipt uploaded", "bucket", s.bucket, "key", key, "size", len(data))
	return key, nil
}

// Download retrieves a receipt from S3.
func (s *ReceiptStore) Download(ctx context.Context, key string) ([]byte, error) {
	output, err := s.client.GetObject(ctx, &s3.GetObjectInput{
		Bucket: aws.String(s.bucket),
		Key:    aws.String(key),
	})
	if err != nil {
		return nil, fmt.Errorf("failed to download receipt: %w", err)
	}
	defer output.Body.Close()

	return io.ReadAll(output.Body)
}

// ListByOrder lists all receipts for a given order.
func (s *ReceiptStore) ListByOrder(ctx context.Context, orderID string) ([]string, error) {
	prefix := fmt.Sprintf("receipts/%s/", orderID)

	output, err := s.client.ListObjectsV2(ctx, &s3.ListObjectsV2Input{
		Bucket: aws.String(s.bucket),
		Prefix: aws.String(prefix),
	})
	if err != nil {
		return nil, fmt.Errorf("failed to list receipts: %w", err)
	}

	keys := make([]string, 0, len(output.Contents))
	for _, obj := range output.Contents {
		keys = append(keys, *obj.Key)
	}
	return keys, nil
}
```

### Tarefa 0.6 — Primeira Operação: S3 Upload/Download (Java)

```java
// domain/repository/ReceiptRepository.java
package com.orderflow.domain.repository;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Repository;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;

import java.time.Instant;
import java.util.List;
import java.util.Map;

@Repository
public class ReceiptRepository {

    private final S3Client s3Client;
    private final String bucketName;

    public ReceiptRepository(
            S3Client s3Client,
            @Value("${orderflow.s3.bucket-name}") String bucketName) {
        this.s3Client = s3Client;
        this.bucketName = bucketName;
    }

    public String upload(String orderId, byte[] data, String contentType) {
        String key = "receipts/%s/%d.pdf".formatted(orderId, Instant.now().toEpochMilli());

        s3Client.putObject(
                PutObjectRequest.builder()
                        .bucket(bucketName)
                        .key(key)
                        .contentType(contentType)
                        .metadata(Map.of(
                                "order-id", orderId,
                                "uploaded-at", Instant.now().toString()
                        ))
                        .build(),
                RequestBody.fromBytes(data)
        );

        return key;
    }

    public byte[] download(String key) {
        return s3Client.getObjectAsBytes(
                GetObjectRequest.builder()
                        .bucket(bucketName)
                        .key(key)
                        .build()
        ).asByteArray();
    }

    public List<String> listByOrder(String orderId) {
        String prefix = "receipts/%s/".formatted(orderId);

        ListObjectsV2Response response = s3Client.listObjectsV2(
                ListObjectsV2Request.builder()
                        .bucket(bucketName)
                        .prefix(prefix)
                        .build()
        );

        return response.contents().stream()
                .map(S3Object::key)
                .toList();
    }
}
```

### Tarefa 0.7 — Primeira Operação: DynamoDB PutItem/GetItem (Go)

```go
// internal/repository/order_store.go
package repository

import (
	"context"
	"fmt"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/feature/dynamodb/attributevalue"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb/types"
	"github.com/google/uuid"
)

// Order represents an order in DynamoDB.
// DynamoDB key design:
//   PK: ORDER#<order-id>
//   SK: METADATA
//   GSI1PK: CUSTOMER#<customer-id>
//   GSI1SK: ORDER#<created-at>#<order-id>
type Order struct {
	PK             string      `dynamodbav:"PK"`
	SK             string      `dynamodbav:"SK"`
	GSI1PK         string      `dynamodbav:"GSI1PK"`
	GSI1SK         string      `dynamodbav:"GSI1SK"`
	ID             string      `dynamodbav:"id"`
	CustomerID     string      `dynamodbav:"customer_id"`
	Status         string      `dynamodbav:"status"`
	PaymentStatus  string      `dynamodbav:"payment_status"`
	Total          float64     `dynamodbav:"total"`
	Currency       string      `dynamodbav:"currency"`
	Items          []OrderItem `dynamodbav:"items"`
	CreatedAt      string      `dynamodbav:"created_at"`
	UpdatedAt      string      `dynamodbav:"updated_at"`
}

type OrderItem struct {
	ProductID   string  `dynamodbav:"product_id"`
	ProductName string  `dynamodbav:"product_name"`
	Quantity    int     `dynamodbav:"quantity"`
	UnitPrice   float64 `dynamodbav:"unit_price"`
	Subtotal    float64 `dynamodbav:"subtotal"`
}

type OrderStore struct {
	client    *dynamodb.Client
	tableName string
}

func NewOrderStore(client *dynamodb.Client, tableName string) *OrderStore {
	return &OrderStore{client: client, tableName: tableName}
}

// Create inserts a new order into DynamoDB.
func (s *OrderStore) Create(ctx context.Context, customerID string, items []OrderItem) (*Order, error) {
	now := time.Now().UTC().Format(time.RFC3339)
	id := uuid.New().String()

	var total float64
	for i := range items {
		items[i].Subtotal = float64(items[i].Quantity) * items[i].UnitPrice
		total += items[i].Subtotal
	}

	order := &Order{
		PK:            fmt.Sprintf("ORDER#%s", id),
		SK:            "METADATA",
		GSI1PK:        fmt.Sprintf("CUSTOMER#%s", customerID),
		GSI1SK:        fmt.Sprintf("ORDER#%s#%s", now, id),
		ID:            id,
		CustomerID:    customerID,
		Status:        "PENDING",
		PaymentStatus: "AWAITING",
		Total:         total,
		Currency:      "BRL",
		Items:         items,
		CreatedAt:     now,
		UpdatedAt:     now,
	}

	item, err := attributevalue.MarshalMap(order)
	if err != nil {
		return nil, fmt.Errorf("failed to marshal order: %w", err)
	}

	_, err = s.client.PutItem(ctx, &dynamodb.PutItemInput{
		TableName: aws.String(s.tableName),
		Item:      item,
		// Condition to prevent overwriting existing orders
		ConditionExpression: aws.String("attribute_not_exists(PK)"),
	})
	if err != nil {
		return nil, fmt.Errorf("failed to put order: %w", err)
	}

	return order, nil
}

// GetByID retrieves an order by its ID.
func (s *OrderStore) GetByID(ctx context.Context, orderID string) (*Order, error) {
	output, err := s.client.GetItem(ctx, &dynamodb.GetItemInput{
		TableName: aws.String(s.tableName),
		Key: map[string]types.AttributeValue{
			"PK": &types.AttributeValueMemberS{Value: fmt.Sprintf("ORDER#%s", orderID)},
			"SK": &types.AttributeValueMemberS{Value: "METADATA"},
		},
	})
	if err != nil {
		return nil, fmt.Errorf("failed to get order: %w", err)
	}

	if output.Item == nil {
		return nil, fmt.Errorf("order not found: %s", orderID)
	}

	var order Order
	if err := attributevalue.UnmarshalMap(output.Item, &order); err != nil {
		return nil, fmt.Errorf("failed to unmarshal order: %w", err)
	}

	return &order, nil
}
```

### Tarefa 0.8 — Primeira Operação: DynamoDB (Java)

```java
// domain/entity/OrderEntity.java
package com.orderflow.domain.entity;

import software.amazon.awssdk.enhanced.dynamodb.mapper.annotations.*;

import java.time.Instant;
import java.util.List;

@DynamoDbBean
public class OrderEntity {

    private String pk;
    private String sk;
    private String gsi1pk;
    private String gsi1sk;
    private String id;
    private String customerId;
    private String status;
    private String paymentStatus;
    private double total;
    private String currency;
    private List<OrderItemEntity> items;
    private Instant createdAt;
    private Instant updatedAt;

    @DynamoDbPartitionKey
    @DynamoDbAttribute("PK")
    public String getPk() { return pk; }
    public void setPk(String pk) { this.pk = pk; }

    @DynamoDbSortKey
    @DynamoDbAttribute("SK")
    public String getSk() { return sk; }
    public void setSk(String sk) { this.sk = sk; }

    @DynamoDbSecondaryPartitionKey(indexNames = "GSI1")
    @DynamoDbAttribute("GSI1PK")
    public String getGsi1pk() { return gsi1pk; }
    public void setGsi1pk(String gsi1pk) { this.gsi1pk = gsi1pk; }

    @DynamoDbSecondarySortKey(indexNames = "GSI1")
    @DynamoDbAttribute("GSI1SK")
    public String getGsi1sk() { return gsi1sk; }
    public void setGsi1sk(String gsi1sk) { this.gsi1sk = gsi1sk; }

    // ... getters and setters for all fields
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public String getCustomerId() { return customerId; }
    public void setCustomerId(String customerId) { this.customerId = customerId; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public String getPaymentStatus() { return paymentStatus; }
    public void setPaymentStatus(String paymentStatus) { this.paymentStatus = paymentStatus; }
    public double getTotal() { return total; }
    public void setTotal(double total) { this.total = total; }
    public String getCurrency() { return currency; }
    public void setCurrency(String currency) { this.currency = currency; }
    public List<OrderItemEntity> getItems() { return items; }
    public void setItems(List<OrderItemEntity> items) { this.items = items; }
    public Instant getCreatedAt() { return createdAt; }
    public void setCreatedAt(Instant createdAt) { this.createdAt = createdAt; }
    public Instant getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(Instant updatedAt) { this.updatedAt = updatedAt; }
}
```

```java
// domain/repository/OrderRepository.java
package com.orderflow.domain.repository;

import com.orderflow.domain.entity.OrderEntity;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Repository;
import software.amazon.awssdk.enhanced.dynamodb.*;
import software.amazon.awssdk.enhanced.dynamodb.model.*;
import software.amazon.awssdk.services.dynamodb.model.ConditionalCheckFailedException;

import java.time.Instant;
import java.util.Optional;
import java.util.UUID;

@Repository
public class OrderRepository {

    private final DynamoDbTable<OrderEntity> table;

    public OrderRepository(
            DynamoDbEnhancedClient enhancedClient,
            @Value("${orderflow.dynamodb.table-name}") String tableName) {
        this.table = enhancedClient.table(tableName, TableSchema.fromBean(OrderEntity.class));
    }

    public OrderEntity create(String customerId, double total) {
        String id = UUID.randomUUID().toString();
        Instant now = Instant.now();

        var entity = new OrderEntity();
        entity.setPk("ORDER#" + id);
        entity.setSk("METADATA");
        entity.setGsi1pk("CUSTOMER#" + customerId);
        entity.setGsi1sk("ORDER#" + now + "#" + id);
        entity.setId(id);
        entity.setCustomerId(customerId);
        entity.setStatus("PENDING");
        entity.setPaymentStatus("AWAITING");
        entity.setTotal(total);
        entity.setCurrency("BRL");
        entity.setCreatedAt(now);
        entity.setUpdatedAt(now);

        table.putItem(PutItemEnhancedRequest.builder(OrderEntity.class)
                .item(entity)
                .conditionExpression(Expression.builder()
                        .expression("attribute_not_exists(PK)")
                        .build())
                .build());

        return entity;
    }

    public Optional<OrderEntity> findById(String orderId) {
        Key key = Key.builder()
                .partitionValue("ORDER#" + orderId)
                .sortValue("METADATA")
                .build();

        return Optional.ofNullable(table.getItem(key));
    }
}
```

### Tarefa 0.9 — Primeiro Teste de Integração com LocalStack (Go)

```go
// internal/repository/order_store_test.go
package repository_test

import (
	"context"
	"testing"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/credentials"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb/types"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	"github.com/testcontainers/testcontainers-go/modules/localstack"

	"orderflow/internal/repository"
)

func setupLocalStack(t *testing.T) *dynamodb.Client {
	t.Helper()
	ctx := context.Background()

	// Start LocalStack container
	container, err := localstack.Run(ctx, "localstack/localstack:latest")
	require.NoError(t, err)
	t.Cleanup(func() { container.Terminate(ctx) })

	// Get endpoint
	endpoint, err := container.PortEndpoint(ctx, "4566/tcp", "http")
	require.NoError(t, err)

	// Create DynamoDB client
	cfg, err := config.LoadDefaultConfig(ctx,
		config.WithRegion("us-east-1"),
		config.WithCredentialsProvider(
			credentials.NewStaticCredentialsProvider("test", "test", ""),
		),
	)
	require.NoError(t, err)

	client := dynamodb.NewFromConfig(cfg, func(o *dynamodb.Options) {
		o.BaseEndpoint = aws.String(endpoint)
	})

	// Create table
	_, err = client.CreateTable(ctx, &dynamodb.CreateTableInput{
		TableName: aws.String("test-orders"),
		AttributeDefinitions: []types.AttributeDefinition{
			{AttributeName: aws.String("PK"), AttributeType: types.ScalarAttributeTypeS},
			{AttributeName: aws.String("SK"), AttributeType: types.ScalarAttributeTypeS},
		},
		KeySchema: []types.KeySchemaElement{
			{AttributeName: aws.String("PK"), KeyType: types.KeyTypeHash},
			{AttributeName: aws.String("SK"), KeyType: types.KeyTypeRange},
		},
		BillingMode: types.BillingModePayPerRequest,
	})
	require.NoError(t, err)

	return client
}

func TestOrderStore_CreateAndGet(t *testing.T) {
	client := setupLocalStack(t)
	store := repository.NewOrderStore(client, "test-orders")
	ctx := context.Background()

	// Create order
	items := []repository.OrderItem{
		{ProductID: "prod-1", ProductName: "Mouse", Quantity: 2, UnitPrice: 49.90},
	}
	order, err := store.Create(ctx, "customer-123", items)
	require.NoError(t, err)

	assert.NotEmpty(t, order.ID)
	assert.Equal(t, "PENDING", order.Status)
	assert.Equal(t, "AWAITING", order.PaymentStatus)
	assert.Equal(t, 99.80, order.Total)
	assert.Len(t, order.Items, 1)

	// Get order
	retrieved, err := store.GetByID(ctx, order.ID)
	require.NoError(t, err)

	assert.Equal(t, order.ID, retrieved.ID)
	assert.Equal(t, order.CustomerID, retrieved.CustomerID)
	assert.Equal(t, order.Total, retrieved.Total)
}
```

### Tarefa 0.10 — Primeiro Teste com Testcontainers (Java)

```java
// OrderRepositoryIntegrationTest.java
package com.orderflow.domain.repository;

import com.orderflow.domain.entity.OrderEntity;
import com.orderflow.infrastructure.aws.DynamoDbConfig;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.localstack.LocalStackContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;
import software.amazon.awssdk.services.dynamodb.DynamoDbClient;
import software.amazon.awssdk.services.dynamodb.model.*;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.testcontainers.containers.localstack.LocalStackContainer.Service.DYNAMODB;

@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {

    @Container
    static LocalStackContainer localStack = new LocalStackContainer(
            DockerImageName.parse("localstack/localstack:latest"))
            .withServices(DYNAMODB);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("aws.endpoint", () -> localStack.getEndpoint().toString());
        registry.add("aws.region", () -> localStack.getRegion());
        registry.add("orderflow.dynamodb.table-name", () -> "test-orders");
    }

    @BeforeAll
    static void createTable(@Autowired DynamoDbClient client) {
        client.createTable(CreateTableRequest.builder()
                .tableName("test-orders")
                .attributeDefinitions(
                        AttributeDefinition.builder()
                                .attributeName("PK").attributeType(ScalarAttributeType.S).build(),
                        AttributeDefinition.builder()
                                .attributeName("SK").attributeType(ScalarAttributeType.S).build()
                )
                .keySchema(
                        KeySchemaElement.builder()
                                .attributeName("PK").keyType(KeyType.HASH).build(),
                        KeySchemaElement.builder()
                                .attributeName("SK").keyType(KeyType.RANGE).build()
                )
                .billingMode(BillingMode.PAY_PER_REQUEST)
                .build());
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldCreateAndRetrieveOrder() {
        // Create
        OrderEntity order = orderRepository.create("customer-123", 99.80);

        assertThat(order.getId()).isNotNull();
        assertThat(order.getStatus()).isEqualTo("PENDING");
        assertThat(order.getPaymentStatus()).isEqualTo("AWAITING");

        // Retrieve
        Optional<OrderEntity> retrieved = orderRepository.findById(order.getId());

        assertThat(retrieved).isPresent();
        assertThat(retrieved.get().getCustomerId()).isEqualTo("customer-123");
        assertThat(retrieved.get().getTotal()).isEqualTo(99.80);
    }
}
```

---

## Anti-Patterns

| Anti-Pattern | Por que está errado | Como corrigir |
|---|---|---|
| Hardcoded credentials no código | Falha de segurança, impossível rotacionar | Usar credential provider chain |
| Hardcoded endpoint URLs | Impossível trocar entre ambientes | Usar variáveis de ambiente / config |
| Ignorar erros do SDK | Perda de dados, comportamento indefinido | Always check `err`, wrap com contexto |
| Criar um client por request | Overhead de conexão, throttling | Criar uma vez, reutilizar (singleton) |
| Não usar `context.Context` | Impossível cancelar operações, leak | Propagar `ctx` em todas as chamadas |
| Usar `aws-sdk-go` v1 | Deprecated, sem suporte | Usar `aws-sdk-go-v2` |
| Field injection em Java | Dificulta testes | Usar constructor injection |

---

## Critérios de Aceite

### Setup
- [ ] LocalStack rodando via Docker Compose com health check
- [ ] AWS CLI configurado e consegue listar recursos no LocalStack
- [ ] Script `setup-localstack.sh` cria S3 bucket, DynamoDB table e SQS queue

### Go
- [ ] AWS config com suporte a LocalStack (endpoint customizado)
- [ ] Factory de clients (DynamoDB, S3, SQS, SNS) configuráveis
- [ ] ReceiptStore funcional: Upload, Download, ListByOrder
- [ ] OrderStore funcional: Create, GetByID
- [ ] Teste de integração com Testcontainers (LocalStack)

### Java
- [ ] AWS config com profiles (`localstack` vs default)
- [ ] DynamoDB Enhanced Client configurado com `@DynamoDbBean`
- [ ] ReceiptRepository funcional: upload, download, listByOrder
- [ ] OrderRepository funcional: create, findById
- [ ] Teste de integração com Testcontainers

### Conhecimento
- [ ] Consegue explicar o modelo de responsabilidade compartilhada
- [ ] Entende a diferença entre serviços regionais e globais
- [ ] Conhece a credential provider chain (ordem de resolução)
- [ ] Sabe por que path-style é necessário para S3 no LocalStack

---

## Definição de Pronto (DoD)

- [ ] LocalStack + setup script funcionam com `docker compose up && ./scripts/setup-localstack.sh`
- [ ] App Go e App Java conseguem CRUD básico em DynamoDB e S3 via SDK
- [ ] Pelo menos 1 teste de integração passando por stack (Go + Java)
- [ ] Commit semântico: `feat(level-0): add AWS foundations with SDK setup and LocalStack`

---

## Checklist

- [ ] docker-compose.yml com LocalStack (healthcheck configurado)
- [ ] setup-localstack.sh testado
- [ ] AWS config Go: credential chain + endpoint override
- [ ] AWS config Java: profiles + Spring Cloud AWS
- [ ] S3: Upload + Download + List (Go + Java)
- [ ] DynamoDB: PutItem + GetItem (Go + Java)
- [ ] Testes de integração com Testcontainers
- [ ] Nenhum credential hardcoded no código

---

## Exercícios Extras

1. **SQS First Message** — Envie uma mensagem para a fila `order-events` e consuma-a via long polling. Implemente em Go (goroutine) e Java (`@SqsListener`).
2. **SNS Publish** — Publique uma notificação no topic `order-notifications`. Crie uma subscription SQS para receber a mensagem.
3. **AWS CLI Scripting** — Crie um script que faz um health check completo: verifica S3, DynamoDB, SQS, SNS e retorna status de cada serviço.
4. **STS AssumeRole** — Use `sts.AssumeRole()` para simular troca de credenciais. Entenda como funciona no ECS Task Role.
