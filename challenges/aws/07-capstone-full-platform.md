# Level 7 — Capstone: OrderFlow Full Platform

> **Objetivo:** Integrar todos os níveis (L0-L6) em uma plataforma production-ready completa,
> adicionando CI/CD pipeline, cost optimization, multi-environment strategy e Well-Architected
> Review. O resultado é a aplicação OrderFlow end-to-end funcionando localmente com LocalStack.

---

## Objetivo de Aprendizado

- Integrar Services (DynamoDB, S3, SQS, SNS, EventBridge, Lambda, Step Functions)
- Implementar CI/CD pipeline (GitHub Actions + OIDC + deploy to ECS)
- Aplicar tagging strategy para cost allocation e resource governance
- Implementar multi-environment configuration (dev, staging, prod)
- Conduzir Well-Architected Review nos 6 pilares
- Orquestrar full order lifecycle end-to-end
- Produzir documentação de arquitetura final

---

## Escopo Funcional

```
Capstone — OrderFlow Full Platform Architecture
════════════════════════════════════════════════

                            ┌──────────────┐
                            │   Client     │
                            │  (HTTP/REST) │
                            └──────┬───────┘
                                   │
                            ┌──────▼───────┐
                            │  Route 53    │
                            │  (DNS)       │
                            └──────┬───────┘
                                   │
                            ┌──────▼───────┐
                            │  CloudFront  │
                            │  (CDN)       │
                            └──────┬───────┘
                                   │
  ┌────────────────────────────────▼─────────────────────────────────┐
  │  VPC (10.0.0.0/16)                                              │
  │                                                                  │
  │  ┌─ Public ──────────────────────────────────────┐              │
  │  │  ALB (:443 HTTPS)                              │              │
  │  │  /api/v1/orders → order-api                    │              │
  │  │  /api/v1/payments → payment-api                │              │
  │  └──────────────┬─────────────────────────────────┘              │
  │                 │                                                 │
  │  ┌─ Private ────▼──────────────────────────────────────────────┐ │
  │  │  ECS Fargate Cluster                                        │ │
  │  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐    │ │
  │  │  │  Order API   │  │  Payment API │  │ Notification  │    │ │
  │  │  │  (Go/Java)   │  │  (Go/Java)   │  │ Worker        │    │ │
  │  │  │  :8080       │  │  :8081       │  │ (SQS Consumer)│    │ │
  │  │  └──────┬───────┘  └──────┬───────┘  └───────┬───────┘    │ │
  │  │         │                 │                   │             │ │
  │  │  ┌──────▼─────────────────▼───────────────────▼──────┐     │ │
  │  │  │              AWS Services (via SDK)                │     │ │
  │  │  │                                                    │     │ │
  │  │  │   DynamoDB ←→ S3 ←→ SQS (FIFO) ←→ SNS           │     │ │
  │  │  │       │              │                │            │     │ │
  │  │  │       │        ┌─────▼──────┐   ┌─────▼──────┐   │     │ │
  │  │  │   ElastiCache  │  Lambda    │   │EventBridge │   │     │ │
  │  │  │   (Redis)      │ (Payment)  │   │ (Router)   │   │     │ │
  │  │  │                └─────┬──────┘   └────────────┘   │     │ │
  │  │  │                      │                            │     │ │
  │  │  │               ┌──────▼──────┐                    │     │ │
  │  │  │               │Step Function│                    │     │ │
  │  │  │               │(Saga: Pay)  │                    │     │ │
  │  │  │               └─────────────┘                    │     │ │
  │  │  └────────────────────────────────────────────────────┘     │ │
  │  └─────────────────────────────────────────────────────────────┘ │
  │                                                                   │
  │  Security: IAM │ KMS │ Secrets Manager │ VPC Endpoints           │
  │  Observe:  CloudWatch │ X-Ray │ Alarms │ Dashboard               │
  └───────────────────────────────────────────────────────────────────┘

  CI/CD Pipeline:
  ═══════════════
    GitHub → Actions → Build (Go/Java) → Test → ECR Push → ECS Deploy
         └── OIDC (no long-lived keys) ──┘       (Blue/Green)

  Environments:
  ═════════════
    dev (LocalStack) → staging (AWS) → prod (AWS)
```

---

## Tarefas

### Tarefa 7.1 — Full Order Lifecycle (Go)

Integrar todos os serviços em um fluxo completo de pedido.

```go
// internal/service/order_lifecycle.go
package service

import (
	"context"
	"fmt"
	"log/slog"
	"time"

	"github.com/google/uuid"
)

// OrderLifecycleService orchestrates the full order lifecycle across all AWS services.
type OrderLifecycleService struct {
	dynamo      *DynamoOrderRepo
	s3          *S3Store
	sqs         *SQSClient
	sns         *SNSPublisher
	eventBridge *EventBridgeClient
	saga        *SagaClient
	kms         *KMSClient
	cache       *OrderCache
	metrics     *MetricsClient
}

type CreateOrderRequest struct {
	CustomerID string      `json:"customer_id"`
	Items      []OrderItem `json:"items"`
}

type OrderItem struct {
	ProductID string  `json:"product_id"`
	Name      string  `json:"name"`
	Quantity  int     `json:"quantity"`
	Price     float64 `json:"price"`
}

type OrderResponse struct {
	OrderID    string  `json:"order_id"`
	Status     string  `json:"status"`
	Total      float64 `json:"total"`
	CreatedAt  string  `json:"created_at"`
}

// CreateOrder — Full lifecycle:
// 1. Generate order ID + compute total
// 2. Encrypt sensitive data (KMS)
// 3. Save to DynamoDB (order + items atomically)
// 4. Publish to SQS (for async payment processing)
// 5. Publish to SNS (fan-out to notification + analytics)
// 6. Route event via EventBridge
// 7. Start payment saga (Step Functions)
// 8. Record metrics (CloudWatch)
func (s *OrderLifecycleService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*OrderResponse, error) {
	start := time.Now()
	orderID := uuid.New().String()

	slog.InfoContext(ctx, "Creating order",
		"order_id", orderID,
		"customer_id", req.CustomerID,
		"items", len(req.Items),
	)

	// 1. Compute total
	var total float64
	for _, item := range req.Items {
		total += item.Price * float64(item.Quantity)
	}

	// 2. Encrypt customer data
	encryptedCustomerID, err := s.kms.EnvelopeEncrypt(ctx, []byte(req.CustomerID), map[string]string{
		"order_id": orderID,
		"purpose":  "customer_pii",
	})
	if err != nil {
		return nil, fmt.Errorf("encrypt customer data: %w", err)
	}

	// 3. Save to DynamoDB atomically (order + items)
	order := Order{
		OrderID:    orderID,
		CustomerID: req.CustomerID,
		Status:     "PENDING",
		Total:      total,
		CreatedAt:  time.Now().UTC().Format(time.RFC3339),
		EncryptedRef: encryptedCustomerID,
	}

	if err := s.dynamo.TransactWriteOrderWithItems(ctx, order, req.Items); err != nil {
		return nil, fmt.Errorf("save order: %w", err)
	}

	// 4. Publish to SQS for async processing
	if err := s.sqs.Publish(ctx, QueueMessage{
		OrderID:     orderID,
		CustomerID:  req.CustomerID,
		EventType:   "ORDER_CREATED",
		Total:       total,
	}); err != nil {
		slog.ErrorContext(ctx, "Failed to publish to SQS (non-blocking)", "error", err)
		// Non-blocking: order is saved, SQS failure will be retried
	}

	// 5. Fan-out via SNS
	if err := s.sns.PublishOrderEvent(ctx, SNSOrderEvent{
		OrderID:    orderID,
		EventType:  "ORDER_CREATED",
		CustomerID: req.CustomerID,
		Total:      total,
	}); err != nil {
		slog.ErrorContext(ctx, "Failed to fan-out via SNS (non-blocking)", "error", err)
	}

	// 6. Route via EventBridge
	if err := s.eventBridge.PutEvent(ctx, EventBridgeEvent{
		Source:     "orderflow.order-api",
		DetailType: "OrderCreated",
		Detail: map[string]interface{}{
			"order_id":    orderID,
			"customer_id": req.CustomerID,
			"total":       total,
			"items_count": len(req.Items),
		},
	}); err != nil {
		slog.ErrorContext(ctx, "Failed to route via EventBridge (non-blocking)", "error", err)
	}

	// 7. Start payment saga
	executionID, err := s.saga.StartSaga(ctx, orderID, map[string]interface{}{
		"order_id":    orderID,
		"customer_id": req.CustomerID,
		"total":       total,
	})
	if err != nil {
		slog.ErrorContext(ctx, "Failed to start payment saga", "error", err)
		// Mark order for manual review if saga fails to start
	} else {
		slog.InfoContext(ctx, "Payment saga started", "execution_id", executionID)
	}

	// 8. Record business metrics
	duration := time.Since(start)
	_ = s.metrics.PutOrderMetric(ctx, "OrdersCreated", 1, map[string]string{
		"Service": "OrderAPI",
	})
	_ = s.metrics.PutLatencyMetric(ctx, "CreateOrder", float64(duration.Milliseconds()))

	// 9. Invalidate cache
	s.cache.InvalidateOrder(ctx, orderID)

	slog.InfoContext(ctx, "Order created successfully",
		"order_id", orderID,
		"total", total,
		"latency_ms", duration.Milliseconds(),
	)

	return &OrderResponse{
		OrderID:   orderID,
		Status:    "PENDING",
		Total:     total,
		CreatedAt: order.CreatedAt,
	}, nil
}

// GetOrder — cache-aside with Redis → DynamoDB fallback.
func (s *OrderLifecycleService) GetOrder(ctx context.Context, orderID string) (*Order, error) {
	// Try cache first
	order, err := s.cache.GetOrder(ctx, orderID)
	if err == nil && order != nil {
		return order, nil
	}

	// Fallback to DynamoDB
	order, err = s.dynamo.GetOrderWithItems(ctx, orderID)
	if err != nil {
		return nil, fmt.Errorf("get order: %w", err)
	}

	return order, nil
}

// CancelOrder — idempotent cancellation with compensation.
func (s *OrderLifecycleService) CancelOrder(ctx context.Context, orderID string) error {
	// Conditional update: only cancel if PENDING or PROCESSING
	if err := s.dynamo.UpdateOrderStatus(ctx, orderID, "CANCELLED"); err != nil {
		return fmt.Errorf("cancel order: %w", err)
	}

	// Publish cancellation event
	_ = s.eventBridge.PutEvent(ctx, EventBridgeEvent{
		Source:     "orderflow.order-api",
		DetailType: "OrderCancelled",
		Detail: map[string]interface{}{
			"order_id": orderID,
			"reason":   "customer_request",
		},
	})

	_ = s.metrics.PutOrderMetric(ctx, "OrdersCancelled", 1, map[string]string{
		"Service": "OrderAPI",
	})

	s.cache.InvalidateOrder(ctx, orderID)
	return nil
}
```

### Tarefa 7.2 — Full Order Lifecycle (Java)

```java
// application/service/OrderLifecycleService.java
package com.orderflow.application.service;

import com.orderflow.domain.model.Order;
import com.orderflow.domain.model.OrderItem;
import com.orderflow.infrastructure.aws.*;
import com.orderflow.infrastructure.observability.CloudWatchMetricsService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

@Service
public class OrderLifecycleService {

    private static final Logger log = LoggerFactory.getLogger(OrderLifecycleService.class);

    private final DynamoOrderRepository orderRepo;
    private final S3StorageService s3;
    private final SqsMessageService sqs;
    private final SnsNotificationService sns;
    private final EventBridgeService eventBridge;
    private final StepFunctionsSagaService saga;
    private final KmsEncryptionService kms;
    private final CloudWatchMetricsService metrics;

    public OrderLifecycleService(
            DynamoOrderRepository orderRepo,
            S3StorageService s3,
            SqsMessageService sqs,
            SnsNotificationService sns,
            EventBridgeService eventBridge,
            StepFunctionsSagaService saga,
            KmsEncryptionService kms,
            CloudWatchMetricsService metrics) {
        this.orderRepo = orderRepo;
        this.s3 = s3;
        this.sqs = sqs;
        this.sns = sns;
        this.eventBridge = eventBridge;
        this.saga = saga;
        this.kms = kms;
        this.metrics = metrics;
    }

    public record CreateOrderRequest(String customerId, List<OrderItemRequest> items) {}
    public record OrderItemRequest(String productId, String name, int quantity, double price) {}
    public record OrderResponse(String orderId, String status, double total, String createdAt) {}

    public OrderResponse createOrder(CreateOrderRequest request) {
        var start = Instant.now();
        var orderId = UUID.randomUUID().toString();

        log.info("Creating order: orderId={}, customerId={}, items={}",
                orderId, request.customerId(), request.items().size());

        // 1. Compute total
        double total = request.items().stream()
                .mapToDouble(i -> i.price() * i.quantity())
                .sum();

        // 2. Encrypt customer PII
        byte[] encryptedCustomerId = kms.envelopeEncrypt(
                request.customerId().getBytes(),
                Map.of("order_id", orderId, "purpose", "customer_pii"));

        // 3. Save to DynamoDB atomically
        var order = Order.builder()
                .orderId(orderId)
                .customerId(request.customerId())
                .status("PENDING")
                .total(total)
                .createdAt(Instant.now().toString())
                .build();

        var items = request.items().stream()
                .map(i -> OrderItem.builder()
                        .orderId(orderId)
                        .productId(i.productId())
                        .name(i.name())
                        .quantity(i.quantity())
                        .price(i.price())
                        .build())
                .toList();

        orderRepo.transactWriteOrderWithItems(order, items);

        // 4. Publish to SQS
        try {
            sqs.publish(Map.of(
                    "order_id", orderId,
                    "customer_id", request.customerId(),
                    "event_type", "ORDER_CREATED",
                    "total", String.valueOf(total)));
        } catch (Exception e) {
            log.error("Failed to publish to SQS (non-blocking): {}", e.getMessage());
        }

        // 5. Fan-out via SNS
        try {
            sns.publishOrderEvent(orderId, "ORDER_CREATED", request.customerId(), total);
        } catch (Exception e) {
            log.error("Failed to fan-out via SNS (non-blocking): {}", e.getMessage());
        }

        // 6. Route via EventBridge
        try {
            eventBridge.publishEvent("orderflow.order-api", "OrderCreated",
                    Map.of("order_id", orderId, "customer_id", request.customerId(),
                            "total", total, "items_count", request.items().size()));
        } catch (Exception e) {
            log.error("Failed to route via EventBridge (non-blocking): {}", e.getMessage());
        }

        // 7. Start payment saga
        try {
            String executionId = saga.startSaga(orderId, Map.of(
                    "order_id", orderId,
                    "customer_id", request.customerId(),
                    "total", total));
            log.info("Payment saga started: executionId={}", executionId);
        } catch (Exception e) {
            log.error("Failed to start saga: {}", e.getMessage());
        }

        // 8. Metrics
        var duration = Duration.between(start, Instant.now());
        metrics.putOrderMetric("OrdersCreated", 1, Map.of("Service", "OrderAPI"));
        metrics.putLatencyMetric("CreateOrder", duration.toMillis());

        log.info("Order created: orderId={}, total={}, latencyMs={}",
                orderId, total, duration.toMillis());

        return new OrderResponse(orderId, "PENDING", total, order.getCreatedAt());
    }

    public Order getOrder(String orderId) {
        return orderRepo.getOrderWithItems(orderId);
    }

    public void cancelOrder(String orderId) {
        orderRepo.updateOrderStatus(orderId, "CANCELLED");
        eventBridge.publishEvent("orderflow.order-api", "OrderCancelled",
                Map.of("order_id", orderId, "reason", "customer_request"));
        metrics.putOrderMetric("OrdersCancelled", 1, Map.of("Service", "OrderAPI"));
    }
}
```

### Tarefa 7.3 — CI/CD Pipeline (GitHub Actions + AWS OIDC)

```yaml
# .github/workflows/deploy.yml
name: OrderFlow CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: orderflow
  ECS_CLUSTER: orderflow-cluster
  GO_VERSION: "1.24"
  JAVA_VERSION: "25"

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  # ──────────────────────────────────────────────────
  # Job 1: Build & Test Go
  # ──────────────────────────────────────────────────
  build-go:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./orderflow-go

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}

      - name: Download dependencies
        run: go mod download

      - name: Lint
        uses: golangci/golangci-lint-action@v6
        with:
          working-directory: ./orderflow-go

      - name: Unit tests
        run: go test ./... -v -count=1 -race -coverprofile=coverage.out

      - name: Integration tests (LocalStack)
        run: go test ./... -v -tags=integration -count=1
        env:
          AWS_ENDPOINT: http://localhost:4566
          AWS_REGION: us-east-1
          AWS_ACCESS_KEY_ID: test
          AWS_SECRET_ACCESS_KEY: test

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: go-coverage
          path: orderflow-go/coverage.out

  # ──────────────────────────────────────────────────
  # Job 2: Build & Test Java
  # ──────────────────────────────────────────────────
  build-java:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./orderflow-java

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: corretto
          java-version: ${{ env.JAVA_VERSION }}
          cache: maven

      - name: Build & Unit Tests
        run: mvn verify -B -Dspring.profiles.active=test

      - name: Integration Tests (LocalStack + Testcontainers)
        run: mvn verify -B -P integration-tests

      - name: Upload test reports
        uses: actions/upload-artifact@v4
        with:
          name: java-test-reports
          path: orderflow-java/target/surefire-reports/

  # ──────────────────────────────────────────────────
  # Job 3: Docker Build + Push to ECR
  # ──────────────────────────────────────────────────
  docker:
    needs: [build-go, build-java]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-deploy
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build & Push Go image
        env:
          ECR_REGISTRY: ${{ steps.ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY-go:$IMAGE_TAG \
                        -t $ECR_REGISTRY/$ECR_REPOSITORY-go:latest \
                        -f orderflow-go/Dockerfile \
                        orderflow-go/
          docker push $ECR_REGISTRY/$ECR_REPOSITORY-go:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY-go:latest

      - name: Build & Push Java image
        env:
          ECR_REGISTRY: ${{ steps.ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY-java:$IMAGE_TAG \
                        -t $ECR_REGISTRY/$ECR_REPOSITORY-java:latest \
                        -f orderflow-java/Dockerfile \
                        orderflow-java/
          docker push $ECR_REGISTRY/$ECR_REPOSITORY-java:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY-java:latest

  # ──────────────────────────────────────────────────
  # Job 4: Deploy to ECS (Blue/Green)
  # ──────────────────────────────────────────────────
  deploy:
    needs: docker
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-deploy
          aws-region: ${{ env.AWS_REGION }}

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: deploy/taskdef.json
          service: orderflow-api
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
          codedeploy-appspec: deploy/appspec.yaml
          codedeploy-application: orderflow
          codedeploy-deployment-group: orderflow-prod
```

### Tarefa 7.4 — OIDC Provider Setup (CLI)

```bash
#!/bin/bash
# scripts/setup-cicd.sh — Configure GitHub Actions OIDC for AWS

set -euo pipefail

AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
GITHUB_ORG="your-org"
GITHUB_REPO="orderflow"

echo "=== Creating OIDC Provider for GitHub Actions ==="

# Create OIDC identity provider (one-time per account)
OIDC_ARN=$(aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1 \
  --query 'OpenIDConnectProviderArn' --output text 2>/dev/null || \
  aws iam list-open-id-connect-providers --query \
  "OpenIDConnectProviderList[?ends_with(Arn, 'token.actions.githubusercontent.com')].Arn" \
  --output text)

echo "OIDC Provider: $OIDC_ARN"

echo "=== Creating GitHub Actions Deploy Role ==="

# Trust policy: only allow GitHub Actions from specific repo
cat > /tmp/trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "$OIDC_ARN"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:${GITHUB_ORG}/${GITHUB_REPO}:*"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name github-actions-deploy \
  --assume-role-policy-document file:///tmp/trust-policy.json \
  --description "GitHub Actions deploy role (OIDC)" \
  --tags Key=Project,Value=orderflow Key=ManagedBy,Value=cli

echo "=== Attaching Deploy Permissions ==="

# Permission policy: ECR + ECS + CodeDeploy (least privilege)
cat > /tmp/deploy-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRLogin",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ECRPushPull",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "arn:aws:ecr:*:${AWS_ACCOUNT_ID}:repository/orderflow-*"
    },
    {
      "Sid": "ECSDeploy",
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeServices",
        "ecs:UpdateService",
        "ecs:DescribeTaskDefinition",
        "ecs:RegisterTaskDefinition"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Project": "orderflow"
        }
      }
    },
    {
      "Sid": "PassRole",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::${AWS_ACCOUNT_ID}:role/orderflow-task-role",
        "arn:aws:iam::${AWS_ACCOUNT_ID}:role/orderflow-execution-role"
      ]
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name github-actions-deploy \
  --policy-name deploy-permissions \
  --policy-document file:///tmp/deploy-policy.json

echo "=== CI/CD Setup Complete ==="
echo "OIDC Provider: $OIDC_ARN"
echo "Deploy Role: arn:aws:iam::${AWS_ACCOUNT_ID}:role/github-actions-deploy"
echo ""
echo "Add to GitHub Secrets:"
echo "  AWS_ACCOUNT_ID = ${AWS_ACCOUNT_ID}"
```

### Tarefa 7.5 — Tagging Strategy & Cost Tracking (Go)

```go
// internal/cost/tagging.go
package cost

import (
	"context"
	"fmt"
	"log/slog"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/costexplorer"
	cetypes "github.com/aws/aws-sdk-go-v2/service/costexplorer/types"
	"github.com/aws/aws-sdk-go-v2/service/resourcegroupstaggingapi"
	tagtypes "github.com/aws/aws-sdk-go-v2/service/resourcegroupstaggingapi/types"
)

// RequiredTags defines mandatory tags for all OrderFlow resources.
var RequiredTags = map[string]string{
	"Project":     "orderflow",
	"Environment": "", // set per environment
	"Owner":       "", // set per team
	"CostCenter":  "", // set per org
	"ManagedBy":   "sdk",
}

type TaggingClient struct {
	tagging      *resourcegroupstaggingapi.Client
	costExplorer *costexplorer.Client
}

func NewTaggingClient(
	tagging *resourcegroupstaggingapi.Client,
	costExplorer *costexplorer.Client,
) *TaggingClient {
	return &TaggingClient{tagging: tagging, costExplorer: costExplorer}
}

// AuditUntaggedResources finds resources missing required tags.
func (t *TaggingClient) AuditUntaggedResources(ctx context.Context) ([]UntaggedResource, error) {
	// Find resources tagged with Project=orderflow
	output, err := t.tagging.GetResources(ctx, &resourcegroupstaggingapi.GetResourcesInput{
		TagFilters: []tagtypes.TagFilter{
			{Key: aws.String("Project"), Values: []string{"orderflow"}},
		},
	})
	if err != nil {
		return nil, fmt.Errorf("get resources: %w", err)
	}

	var untagged []UntaggedResource
	requiredKeys := []string{"Project", "Environment", "Owner", "CostCenter", "ManagedBy"}

	for _, resource := range output.ResourceTagMappingList {
		tagMap := make(map[string]bool)
		for _, tag := range resource.Tags {
			tagMap[aws.ToString(tag.Key)] = true
		}

		var missingTags []string
		for _, key := range requiredKeys {
			if !tagMap[key] {
				missingTags = append(missingTags, key)
			}
		}

		if len(missingTags) > 0 {
			untagged = append(untagged, UntaggedResource{
				ARN:         aws.ToString(resource.ResourceARN),
				MissingTags: missingTags,
			})
		}
	}

	slog.Info("Tag audit complete",
		"total_resources", len(output.ResourceTagMappingList),
		"untagged", len(untagged),
	)

	return untagged, nil
}

// GetCostByService returns cost breakdown by AWS service for OrderFlow.
func (t *TaggingClient) GetCostByService(ctx context.Context, startDate, endDate string) ([]ServiceCost, error) {
	output, err := t.costExplorer.GetCostAndUsage(ctx, &costexplorer.GetCostAndUsageInput{
		TimePeriod: &cetypes.DateInterval{
			Start: aws.String(startDate),
			End:   aws.String(endDate),
		},
		Granularity: cetypes.GranularityMonthly,
		Metrics:     []string{"BlendedCost", "UnblendedCost", "UsageQuantity"},
		GroupBy: []cetypes.GroupDefinition{
			{Type: cetypes.GroupDefinitionTypeDimension, Key: aws.String("SERVICE")},
		},
		Filter: &cetypes.Expression{
			Tags: &cetypes.TagValues{
				Key:    aws.String("Project"),
				Values: []string{"orderflow"},
			},
		},
	})
	if err != nil {
		return nil, fmt.Errorf("get cost: %w", err)
	}

	var costs []ServiceCost
	for _, result := range output.ResultsByTime {
		for _, group := range result.Groups {
			costs = append(costs, ServiceCost{
				Service:       group.Keys[0],
				BlendedCost:   aws.ToString(group.Metrics["BlendedCost"].Amount),
				UnblendedCost: aws.ToString(group.Metrics["UnblendedCost"].Amount),
				Currency:      aws.ToString(group.Metrics["BlendedCost"].Unit),
			})
		}
	}

	return costs, nil
}

type UntaggedResource struct {
	ARN         string   `json:"arn"`
	MissingTags []string `json:"missing_tags"`
}

type ServiceCost struct {
	Service       string `json:"service"`
	BlendedCost   string `json:"blended_cost"`
	UnblendedCost string `json:"unblended_cost"`
	Currency      string `json:"currency"`
}
```

### Tarefa 7.6 — Multi-Environment Configuration (Go)

```go
// internal/config/config.go
package config

import (
	"fmt"
	"os"
)

type Config struct {
	Environment string
	AWS         AWSConfig
	Server      ServerConfig
	Features    FeatureFlags
}

type AWSConfig struct {
	Region       string
	Endpoint     string // empty for real AWS, LocalStack URL for dev
	DynamoTable  string
	S3Bucket     string
	SQSQueueURL  string
	SNSTopicARN  string
	KMSKeyID     string
	EventBusName string
	SagaARN      string
}

type ServerConfig struct {
	Port string
}

type FeatureFlags struct {
	EnableEncryption bool
	EnableTracing    bool
	EnableMetrics    bool
}

// LoadConfig loads environment-specific configuration.
func LoadConfig() (*Config, error) {
	env := getEnv("ENVIRONMENT", "dev")

	cfg := &Config{
		Environment: env,
		Server: ServerConfig{
			Port: getEnv("PORT", "8080"),
		},
	}

	switch env {
	case "dev":
		cfg.AWS = AWSConfig{
			Region:       "us-east-1",
			Endpoint:     "http://localhost:4566", // LocalStack
			DynamoTable:  "orderflow-orders",
			S3Bucket:     "orderflow-receipts-dev",
			SQSQueueURL:  "http://localhost:4566/000000000000/orderflow-orders.fifo",
			SNSTopicARN:  "arn:aws:sns:us-east-1:000000000000:orderflow-events",
			KMSKeyID:     "alias/orderflow-key",
			EventBusName: "orderflow-events",
			SagaARN:      "arn:aws:states:us-east-1:000000000000:stateMachine:orderflow-payment-saga",
		}
		cfg.Features = FeatureFlags{
			EnableEncryption: false, // Simplify local dev
			EnableTracing:    false,
			EnableMetrics:    true,
		}

	case "staging":
		cfg.AWS = AWSConfig{
			Region:       getEnv("AWS_REGION", "us-east-1"),
			DynamoTable:  "orderflow-orders-staging",
			S3Bucket:     "orderflow-receipts-staging",
			SQSQueueURL:  getEnv("SQS_QUEUE_URL", ""),
			SNSTopicARN:  getEnv("SNS_TOPIC_ARN", ""),
			KMSKeyID:     getEnv("KMS_KEY_ID", ""),
			EventBusName: "orderflow-events-staging",
			SagaARN:      getEnv("SAGA_ARN", ""),
		}
		cfg.Features = FeatureFlags{
			EnableEncryption: true,
			EnableTracing:    true,
			EnableMetrics:    true,
		}

	case "prod":
		cfg.AWS = AWSConfig{
			Region:       getEnv("AWS_REGION", "us-east-1"),
			DynamoTable:  "orderflow-orders",
			S3Bucket:     "orderflow-receipts-prod",
			SQSQueueURL:  getEnv("SQS_QUEUE_URL", ""),
			SNSTopicARN:  getEnv("SNS_TOPIC_ARN", ""),
			KMSKeyID:     getEnv("KMS_KEY_ID", ""),
			EventBusName: "orderflow-events",
			SagaARN:      getEnv("SAGA_ARN", ""),
		}
		cfg.Features = FeatureFlags{
			EnableEncryption: true,
			EnableTracing:    true,
			EnableMetrics:    true,
		}

	default:
		return nil, fmt.Errorf("unknown environment: %s", env)
	}

	return cfg, nil
}

func getEnv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}
```

### Tarefa 7.7 — Full LocalStack Bootstrap (CLI)

```bash
#!/bin/bash
# scripts/bootstrap-full.sh — Create ALL AWS resources in LocalStack for OrderFlow

set -euo pipefail

ENDPOINT="http://localhost:4566"
REGION="us-east-1"
AWS="aws --endpoint-url=$ENDPOINT --region=$REGION"

echo "╔══════════════════════════════════════════════════╗"
echo "║  OrderFlow — Full Platform Bootstrap (LocalStack) ║"
echo "╚══════════════════════════════════════════════════╝"

# ─────────────────────────────────────────────────────
# L0: Foundation — DynamoDB + S3
# ─────────────────────────────────────────────────────
echo ""
echo "=== [L0] Creating DynamoDB Table ==="
$AWS dynamodb create-table \
  --table-name orderflow-orders \
  --attribute-definitions \
    AttributeName=PK,AttributeType=S \
    AttributeName=SK,AttributeType=S \
    AttributeName=GSI1PK,AttributeType=S \
    AttributeName=GSI1SK,AttributeType=S \
    AttributeName=GSI2PK,AttributeType=S \
    AttributeName=GSI2SK,AttributeType=S \
  --key-schema \
    AttributeName=PK,KeyType=HASH \
    AttributeName=SK,KeyType=RANGE \
  --global-secondary-indexes \
    '[{"IndexName":"GSI1","KeySchema":[{"AttributeName":"GSI1PK","KeyType":"HASH"},{"AttributeName":"GSI1SK","KeyType":"RANGE"}],"Projection":{"ProjectionType":"ALL"},"ProvisionedThroughput":{"ReadCapacityUnits":5,"WriteCapacityUnits":5}},
      {"IndexName":"GSI2","KeySchema":[{"AttributeName":"GSI2PK","KeyType":"HASH"},{"AttributeName":"GSI2SK","KeyType":"RANGE"}],"Projection":{"ProjectionType":"ALL"},"ProvisionedThroughput":{"ReadCapacityUnits":5,"WriteCapacityUnits":5}}]' \
  --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10 \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
  --tags Key=Project,Value=orderflow Key=Environment,Value=dev

echo "=== [L0] Creating S3 Buckets ==="
$AWS s3 mb s3://orderflow-receipts-dev
$AWS s3api put-bucket-versioning --bucket orderflow-receipts-dev \
  --versioning-configuration Status=Enabled
$AWS s3api put-bucket-lifecycle-configuration --bucket orderflow-receipts-dev \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "archive-old-receipts",
        "Status": "Enabled",
        "Transitions": [
          {"Days": 90, "StorageClass": "STANDARD_IA"},
          {"Days": 365, "StorageClass": "GLACIER"}
        ]
      },
      {
        "ID": "cleanup-temp",
        "Status": "Enabled",
        "Filter": {"Prefix": "temp/"},
        "Expiration": {"Days": 7}
      }
    ]
  }'

# ─────────────────────────────────────────────────────
# L4: Messaging — SQS + SNS + EventBridge
# ─────────────────────────────────────────────────────
echo ""
echo "=== [L4] Creating SQS Queues ==="

# DLQ first
DLQ_URL=$($AWS sqs create-queue \
  --queue-name orderflow-dlq.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=false \
  --query 'QueueUrl' --output text)

DLQ_ARN=$($AWS sqs get-queue-attributes \
  --queue-url "$DLQ_URL" \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)

# Main queue with DLQ redrive
$AWS sqs create-queue \
  --queue-name orderflow-orders.fifo \
  --attributes '{
    "FifoQueue": "true",
    "ContentBasedDeduplication": "false",
    "VisibilityTimeout": "60",
    "MessageRetentionPeriod": "1209600",
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"'"$DLQ_ARN"'\",\"maxReceiveCount\":\"3\"}"
  }'

echo "=== [L4] Creating SNS Topic ==="
TOPIC_ARN=$($AWS sns create-topic --name orderflow-events --query 'TopicArn' --output text)

# Subscribe SQS queues to SNS (fan-out)
NOTIFICATION_QUEUE=$($AWS sqs create-queue --queue-name orderflow-notifications \
  --query 'QueueUrl' --output text)
NOTIFICATION_ARN=$($AWS sqs get-queue-attributes \
  --queue-url "$NOTIFICATION_QUEUE" \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)

$AWS sns subscribe \
  --topic-arn "$TOPIC_ARN" \
  --protocol sqs \
  --notification-endpoint "$NOTIFICATION_ARN" \
  --attributes '{"FilterPolicy":"{\"event_type\":[\"ORDER_CREATED\",\"ORDER_PAID\"]}","RawMessageDelivery":"true"}'

echo "=== [L4] Creating EventBridge ==="
$AWS events create-event-bus --name orderflow-events

$AWS events put-rule \
  --name orderflow-order-created \
  --event-bus-name orderflow-events \
  --event-pattern '{
    "source": ["orderflow.order-api"],
    "detail-type": ["OrderCreated"]
  }'

echo "=== [L4] Creating Step Functions (Payment Saga) ==="
$AWS stepfunctions create-state-machine \
  --name orderflow-payment-saga \
  --role-arn "arn:aws:iam::000000000000:role/orderflow-saga-role" \
  --definition '{
    "Comment": "OrderFlow Payment Saga",
    "StartAt": "CreateOrder",
    "States": {
      "CreateOrder": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-create-order",
        "Next": "ReserveInventory",
        "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "CancelOrder"}]
      },
      "ReserveInventory": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-reserve-inventory",
        "Next": "ProcessPayment",
        "Retry": [{"ErrorEquals": ["States.TaskFailed"], "MaxAttempts": 2, "IntervalSeconds": 3, "BackoffRate": 2.0}],
        "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "CancelOrder"}]
      },
      "ProcessPayment": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-process-payment",
        "Next": "ConfirmOrder",
        "Retry": [{"ErrorEquals": ["States.TaskFailed"], "MaxAttempts": 3, "IntervalSeconds": 5, "BackoffRate": 2.0}],
        "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "RefundPayment"}]
      },
      "ConfirmOrder": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-confirm-order",
        "End": true
      },
      "RefundPayment": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-refund-payment",
        "Next": "ReleaseInventory"
      },
      "ReleaseInventory": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-release-inventory",
        "Next": "CancelOrder"
      },
      "CancelOrder": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-cancel-order",
        "End": true
      }
    }
  }'

# ─────────────────────────────────────────────────────
# L5: Security — KMS + Secrets Manager
# ─────────────────────────────────────────────────────
echo ""
echo "=== [L5] Creating KMS Key ==="
KMS_KEY_ID=$($AWS kms create-key \
  --description "OrderFlow encryption key" \
  --tags TagKey=Project,TagValue=orderflow \
  --query 'KeyMetadata.KeyId' --output text)

$AWS kms create-alias \
  --alias-name alias/orderflow-key \
  --target-key-id "$KMS_KEY_ID"

echo "=== [L5] Creating Secrets ==="
$AWS secretsmanager create-secret \
  --name orderflow/database \
  --secret-string '{"host":"localhost","port":5432,"username":"orderflow","password":"dev-password-change-me","dbname":"orderflow"}' \
  --tags Key=Project,Value=orderflow

$AWS secretsmanager create-secret \
  --name orderflow/api-keys \
  --secret-string '{"payment_gateway":"pk_test_dev123","notification_service":"ns_test_dev456"}' \
  --tags Key=Project,Value=orderflow

# ─────────────────────────────────────────────────────
# L6: Observability — CloudWatch
# ─────────────────────────────────────────────────────
echo ""
echo "=== [L6] Creating CloudWatch Resources ==="
for LG in /ecs/orderflow-go /ecs/orderflow-java /orderflow/application; do
  $AWS logs create-log-group --log-group-name "$LG"
  $AWS logs put-retention-policy --log-group-name "$LG" --retention-in-days 30
done

ALERT_TOPIC=$($AWS sns create-topic --name orderflow-alerts --query 'TopicArn' --output text)

$AWS cloudwatch put-metric-alarm \
  --alarm-name orderflow-high-error-rate \
  --namespace OrderFlow --metric-name OrderErrors \
  --statistic Sum --period 300 --evaluation-periods 2 \
  --threshold 10 --comparison-operator GreaterThanThreshold \
  --alarm-actions "$ALERT_TOPIC"

$AWS cloudwatch put-metric-alarm \
  --alarm-name orderflow-dlq-depth \
  --namespace AWS/SQS --metric-name ApproximateNumberOfMessagesVisible \
  --statistic Sum --period 60 --evaluation-periods 1 \
  --threshold 0 --comparison-operator GreaterThanThreshold \
  --dimensions "Name=QueueName,Value=orderflow-dlq.fifo" \
  --alarm-actions "$ALERT_TOPIC"

echo ""
echo "╔══════════════════════════════════════════════════╗"
echo "║  Bootstrap Complete!                              ║"
echo "╠══════════════════════════════════════════════════╣"
echo "║  DynamoDB: orderflow-orders (+ GSI1, GSI2)       ║"
echo "║  S3:       orderflow-receipts-dev (versioned)    ║"
echo "║  SQS:      orderflow-orders.fifo + DLQ           ║"
echo "║  SNS:      orderflow-events (fan-out)            ║"
echo "║  EventBridge: orderflow-events (bus + rules)     ║"
echo "║  Step Functions: payment-saga (7 states)         ║"
echo "║  KMS:      alias/orderflow-key                   ║"
echo "║  Secrets:  database + api-keys                   ║"
echo "║  CloudWatch: log groups + alarms                 ║"
echo "╚══════════════════════════════════════════════════╝"
```

### Tarefa 7.8 — Docker Compose (Full Stack)

```yaml
# docker-compose.yml — OrderFlow Full Platform
version: "3.9"

services:
  # ─── AWS Emulator ────────────────────────
  localstack:
    image: localstack/localstack:3
    ports:
      - "4566:4566"
    environment:
      SERVICES: "dynamodb,s3,sqs,sns,events,stepfunctions,lambda,kms,secretsmanager,sts,logs,cloudwatch,iam"
      DEFAULT_REGION: us-east-1
      DEBUG: 0
    volumes:
      - localstack-data:/var/lib/localstack
      - /var/run/docker.sock:/var/run/docker.sock
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4566/_localstack/health"]
      interval: 10s
      retries: 5

  # ─── Bootstrap Script ───────────────────
  bootstrap:
    image: amazon/aws-cli:latest
    depends_on:
      localstack:
        condition: service_healthy
    environment:
      AWS_ACCESS_KEY_ID: test
      AWS_SECRET_ACCESS_KEY: test
      AWS_DEFAULT_REGION: us-east-1
    volumes:
      - ./scripts/bootstrap-full.sh:/bootstrap.sh:ro
    entrypoint: ["/bin/bash", "/bootstrap.sh"]

  # ─── Redis (Cache) ──────────────────────
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 3

  # ─── Order API (Go) ─────────────────────
  order-api-go:
    build:
      context: ./orderflow-go
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      ENVIRONMENT: dev
      PORT: "8080"
      AWS_ENDPOINT: http://localstack:4566
      AWS_REGION: us-east-1
      AWS_ACCESS_KEY_ID: test
      AWS_SECRET_ACCESS_KEY: test
      REDIS_URL: redis://redis:6379
    depends_on:
      bootstrap:
        condition: service_completed_successfully
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/health/live"]
      interval: 10s
      retries: 3

  # ─── Order API (Java) ───────────────────
  order-api-java:
    build:
      context: ./orderflow-java
      dockerfile: Dockerfile
    ports:
      - "8081:8080"
    environment:
      SPRING_PROFILES_ACTIVE: localstack
      AWS_ENDPOINT: http://localstack:4566
      AWS_REGION: us-east-1
      AWS_ACCESS_KEY_ID: test
      AWS_SECRET_ACCESS_KEY: test
      SPRING_DATA_REDIS_HOST: redis
    depends_on:
      bootstrap:
        condition: service_completed_successfully
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 15s
      retries: 5

volumes:
  localstack-data:
```

---

## Well-Architected Review

```
Well-Architected Framework — OrderFlow Assessment
══════════════════════════════════════════════════

  ┌──────────────────────┬──────────────────────────────────────────────┐
  │ Pilar                │ Implementação no OrderFlow                   │
  ├──────────────────────┼──────────────────────────────────────────────┤
  │ Operational          │ CloudWatch Logs/Metrics/Alarms (L6)         │
  │ Excellence           │ X-Ray distributed tracing (L6)              │
  │                      │ Health checks: liveness + readiness (L6)    │
  │                      │ CI/CD pipeline automatizado (L7)            │
  │                      │ Infrastructure as code (CLI scripts)        │
  ├──────────────────────┼──────────────────────────────────────────────┤
  │ Security             │ IAM least-privilege roles (L5)              │
  │                      │ KMS envelope encryption (L5)                │
  │                      │ Secrets Manager + rotation (L5)             │
  │                      │ VPC + private subnets (L1)                  │
  │                      │ OIDC for CI/CD (no long-lived keys) (L7)   │
  │                      │ CloudTrail audit (L5)                       │
  ├──────────────────────┼──────────────────────────────────────────────┤
  │ Reliability          │ Multi-AZ ECS (L2)                           │
  │                      │ DLQ for poison messages (L4)                │
  │                      │ Step Functions saga (compensating tx) (L4)  │
  │                      │ Health check auto-recovery (L6)             │
  │                      │ S3 versioning + lifecycle (L3)              │
  │                      │ Retry + exponential backoff (L4)            │
  ├──────────────────────┼──────────────────────────────────────────────┤
  │ Performance          │ DynamoDB single-table design + GSI (L3)     │
  │ Efficiency           │ ElastiCache Redis cache-aside (L3)          │
  │                      │ SQS FIFO for ordered processing (L4)       │
  │                      │ Lambda SnapStart (Java) (L4)                │
  │                      │ CloudFront CDN (L1)                         │
  │                      │ Go scratch images (<20MB) (L2)              │
  ├──────────────────────┼──────────────────────────────────────────────┤
  │ Cost                 │ Fargate Spot for workers (L7)               │
  │ Optimization         │ S3 lifecycle (Standard→IA→Glacier) (L3)    │
  │                      │ DynamoDB on-demand for variable load (L3)   │
  │                      │ Tagging strategy + Cost Explorer (L7)       │
  │                      │ Lambda pay-per-invocation (L4)              │
  ├──────────────────────┼──────────────────────────────────────────────┤
  │ Sustainability       │ Graviton (ARM64) instances (L2)             │
  │                      │ Right-sized containers (L2)                 │
  │                      │ S3 Intelligent-Tiering (L3)                 │
  │                      │ Efficient data formats (JSON→compact) (L3)  │
  │                      │ Lambda ephemeral compute (L4)               │
  └──────────────────────┴──────────────────────────────────────────────┘
```

---

## Cost Optimization Checklist

```
Cost Optimization — OrderFlow
═════════════════════════════

  Tagging:
  ├── [ ] Tag em TODOS os recursos: Project, Environment, Owner, CostCenter, ManagedBy
  ├── [ ] Tag audit automático (AuditUntaggedResources via SDK)
  └── [ ] Cost allocation tags ativados no Billing Console

  Compute:
  ├── [ ] ECS Fargate Spot para workers/batch (até 70% economia)
  ├── [ ] Go scratch images (<20MB vs ~200MB Java)
  ├── [ ] Lambda ARM64 (Graviton, 20% mais barato)
  └── [ ] Auto-scaling baseado em métricas reais (CPU/memory/request)

  Storage:
  ├── [ ] S3 lifecycle: Standard → IA (90d) → Glacier (365d)
  ├── [ ] DynamoDB on-demand para variável, provisioned para previsível
  ├── [ ] ElastiCache: eviction policy + TTL
  └── [ ] CloudWatch Logs retention: 30 dias (não infinito)

  Messaging:
  ├── [ ] SQS: batch send/receive (reduz API calls)
  ├── [ ] SNS: filter policies (reduz entregas desnecessárias)
  └── [ ] EventBridge: regras específicas (não processar tudo)

  Monitoring:
  ├── [ ] Budget alerts (50%, 80%, 100%)
  ├── [ ] Cost Explorer dashboard mensal
  └── [ ] Cost anomaly detection habilitado
```

---

## Anti-Patterns

| Anti-Pattern | Por que está errado | Como corrigir |
|---|---|---|
| Long-lived AWS credentials em CI | Exposição de credentials | OIDC federation (sem chaves) |
| Deploy direto para prod | Sem validação em staging | Pipeline: dev → staging → prod |
| Sem tags nos recursos | Impossível track cost/ownership | Required tags policy |
| Monolith em um container | Blast radius alto, scale acoplado | Serviços independentes por domínio |
| Tudo síncrono | Acoplamento temporal | Messaging async (SQS, EventBridge) |
| Sem DLQ | Mensagens perdidas silenciosamente | DLQ em toda fila + alarm |
| Code e infra no mesmo repo sem separação | Tudo muda junto | Repos separados ou paths claros |
| Sem health checks | ALB não remove instâncias quebradas | Liveness + Readiness endpoints |

---

## Critérios de Aceite

### Full Lifecycle
- [ ] CreateOrder executa: DynamoDB + SQS + SNS + EventBridge + Step Functions
- [ ] GetOrder com cache-aside (Redis → DynamoDB)
- [ ] CancelOrder com evento de compensação
- [ ] Go + Java implementações completas

### CI/CD
- [ ] GitHub Actions workflow com OIDC (sem long-lived credentials)
- [ ] Build paralelo Go + Java
- [ ] Lint + Unit tests + Integration tests (LocalStack/Testcontainers)
- [ ] Docker build + ECR push
- [ ] ECS deploy com rollback
- [ ] OIDC provider + IAM role via CLI script

### Cost
- [ ] Tagging strategy (5 tags obrigatórias)
- [ ] Tag audit via SDK (AuditUntaggedResources)
- [ ] Cost Explorer query por serviço (filtrado por Project tag)

### Multi-Environment
- [ ] Config diferenciada por env (dev/staging/prod)
- [ ] Feature flags (encryption, tracing, metrics)
- [ ] LocalStack endpoint para dev, real AWS para staging/prod

### Full Stack
- [ ] `docker-compose up` levanta todo o stack
- [ ] Bootstrap script cria todos os recursos AWS (L0-L6)
- [ ] Go API + Java API rodando simultaneamente
- [ ] LocalStack + Redis + Bootstrap + APIs

---

## Definição de Pronto (DoD)

- [ ] Full lifecycle Go (CreateOrder → 8 steps) + Java equivalente
- [ ] GitHub Actions CI/CD com OIDC (4 jobs: build-go, build-java, docker, deploy)
- [ ] OIDC provider + IAM deploy role via CLI script
- [ ] Tagging strategy + cost tracking via SDK (Cost Explorer)
- [ ] Multi-env config (dev/staging/prod) com feature flags
- [ ] Bootstrap script cria ALL resources (DynamoDB, S3, SQS, SNS, EventBridge, Step Functions, KMS, Secrets, CloudWatch)
- [ ] Docker Compose full stack (LocalStack + Redis + Go API + Java API)
- [ ] Well-Architected Review (6 pilares documentados)
- [ ] Cost optimization checklist validado
- [ ] `docker-compose up && curl localhost:8080/health/ready` retorna 200 OK
- [ ] Commit semântico: `feat(level-7): add full platform integration, CI/CD, cost optimization`

---

## Checklist

- [ ] Lifecycle: CreateOrder/GetOrder/CancelOrder Go + Java (all services)
- [ ] CI/CD: GitHub Actions + OIDC + ECR + ECS deploy
- [ ] Security: OIDC provider + IAM deploy role (CLI)
- [ ] Cost: tagging strategy + tag audit + Cost Explorer query
- [ ] Config: multi-environment (dev/staging/prod) + feature flags
- [ ] Bootstrap: `bootstrap-full.sh` (DynamoDB + S3 + SQS + SNS + EventBridge + Step Functions + KMS + Secrets + CloudWatch)
- [ ] Docker Compose: LocalStack + Redis + Go + Java
- [ ] WAF: 6-pillar review documented

---

## Exercícios Extras

1. **End-to-End Integration Test** — Implemente um teste que cria um pedido via API, verifica que o pedido chegou na SQS, que o SNS disparou notificação, que o EventBridge roteou o evento, e que a saga started no Step Functions. Tudo contra LocalStack via Testcontainers.
2. **Blue/Green Deploy Simulation** — Implemente um script que simula blue/green deploy: sobe nova versão, health check, shift traffic (weighted Route 53 ou ALB target groups), rollback se health check falhar.
3. **Chaos Engineering** — Implemente um endpoint `POST /chaos/{scenario}` que simula falhas: `dynamodb-latency` (sleep 5s antes do DynamoDB), `sqs-failure` (rejeita publish), `cache-miss` (limpa Redis). Verifique que o sistema degrada gracefully.
4. **Cost Dashboard API** — Implemente `GET /api/v1/cost/summary` que retorna custo por serviço AWS (via Cost Explorer SDK), top untagged resources, e projeção mensal.
5. **Architecture Decision Records (ADRs)** — Documente 5 ADRs do OrderFlow: (1) DynamoDB vs Aurora, (2) SQS vs EventBridge para order events, (3) ECS vs Lambda para API, (4) Go vs Java trade-offs, (5) Single-table vs multi-table DynamoDB.
