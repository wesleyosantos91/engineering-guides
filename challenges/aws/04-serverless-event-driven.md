# Level 4 — Serverless & Event-Driven

> **Objetivo:** Construir pipelines event-driven com Lambda (Go + Java), API Gateway,
> SQS/SNS (fan-out), EventBridge (routing), e Step Functions (saga orchestration).
> Aplicar padrões de messaging, idempotência e dead-letter queues.

---

## Objetivo de Aprendizado

- Criar Lambda functions otimizadas para Go (`provided.al2023`) e Java (`SnapStart`)
- Configurar API Gateway (HTTP API) com routes, throttling e JWT authorizer
- Implementar SQS consumers com batch processing, visibility timeout e DLQ
- Usar SNS para fan-out (um evento → múltiplos consumers)
- Configurar EventBridge para event routing com rules e patterns
- Orquestrar Saga pattern com Step Functions (create → reserve → pay → confirm)
- Garantir idempotência e tratamento de poison messages

---

## Escopo Funcional

```
Level 4 — Serverless & Event-Driven Architecture
══════════════════════════════════════════════════

   Client
     │
     ▼
  ┌───────────────┐
  │  API Gateway  │──── JWT Authorizer
  │  (HTTP API)   │
  └──────┬────────┘
         │
    ┌────▼────────────────────────────────────────────┐
    │              Lambda Functions                    │
    │                                                  │
    │  ┌──────────┐ ┌──────────┐ ┌──────────────┐    │
    │  │CreateOrder│ │ GetOrder │ │ProcessPayment│    │
    │  └─────┬────┘ └──────────┘ └──────┬───────┘    │
    │        │                          │             │
    └────────┼──────────────────────────┼─────────────┘
             │                          │
        ┌────▼────┐              ┌──────▼───────┐
        │   SQS   │              │  EventBridge  │
        │ (queue) │              │  (event bus)  │
        └────┬────┘              └──────┬───────┘
             │                          │
        ┌────▼────┐         ┌───────────┼───────────┐
        │ Lambda  │         │           │           │
        │Consumer │    ┌────▼───┐ ┌─────▼──┐ ┌─────▼────┐
        └─────────┘    │Notify  │ │Audit   │ │Analytics │
                       │(SNS)   │ │(S3)    │ │(Lambda)  │
                       └────────┘ └────────┘ └──────────┘

  Step Functions Saga:
  ═══════════════════
    CreateOrder ──→ ReserveInventory ──→ ProcessPayment ──→ ConfirmOrder
         │                │                    │                │
         ▼                ▼                    ▼                ▼
    (compensate)     ReleaseStock        RefundPayment     CancelOrder
                    (on failure)         (on failure)      (on failure)
```

---

## Tarefas

### Tarefa 4.1 — Lambda Function (Go)

```go
// Lambda handler para Go usando provided.al2023 runtime.
// cmd/lambda/create-order/main.go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"os"
	"time"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
	"github.com/aws/aws-sdk-go-v2/service/sqs"
	"github.com/google/uuid"
)

// Initialize SDK clients OUTSIDE the handler (reused across invocations).
var (
	dynamoClient *dynamodb.Client
	sqsClient    *sqs.Client
	tableName    string
	queueURL     string
)

func init() {
	cfg, err := config.LoadDefaultConfig(context.Background())
	if err != nil {
		slog.Error("Failed to load AWS config", "error", err)
		os.Exit(1)
	}

	dynamoClient = dynamodb.NewFromConfig(cfg)
	sqsClient = sqs.NewFromConfig(cfg)
	tableName = os.Getenv("TABLE_NAME")
	queueURL = os.Getenv("QUEUE_URL")
}

type CreateOrderRequest struct {
	CustomerID string      `json:"customer_id"`
	Items      []OrderItem `json:"items"`
}

type OrderItem struct {
	ProductID string  `json:"product_id"`
	Quantity  int     `json:"quantity"`
	Price     float64 `json:"price"`
}

type CreateOrderResponse struct {
	OrderID   string `json:"order_id"`
	Status    string `json:"status"`
	CreatedAt string `json:"created_at"`
}

func handler(ctx context.Context, event events.APIGatewayV2HTTPRequest) (events.APIGatewayV2HTTPResponse, error) {
	// Extract request ID for idempotency
	requestID := event.RequestContext.RequestID
	slog.Info("Processing request", "request_id", requestID, "path", event.RawPath)

	var req CreateOrderRequest
	if err := json.Unmarshal([]byte(event.Body), &req); err != nil {
		return apiResponse(400, map[string]string{"error": "invalid request body"}), nil
	}

	if req.CustomerID == "" || len(req.Items) == 0 {
		return apiResponse(400, map[string]string{"error": "customer_id and items required"}), nil
	}

	orderID := uuid.New().String()
	now := time.Now().UTC()

	// Store order in DynamoDB (idempotent: uses condition expression)
	if err := saveOrder(ctx, orderID, req, now); err != nil {
		slog.Error("Failed to save order", "error", err)
		return apiResponse(500, map[string]string{"error": "internal error"}), nil
	}

	// Send event to SQS for async processing
	if err := publishOrderCreated(ctx, orderID, req.CustomerID); err != nil {
		slog.Warn("Failed to publish event (order saved, processing will retry)", "error", err)
	}

	response := CreateOrderResponse{
		OrderID:   orderID,
		Status:    "PENDING",
		CreatedAt: now.Format(time.RFC3339),
	}

	return apiResponse(201, response), nil
}

func apiResponse(status int, body any) events.APIGatewayV2HTTPResponse {
	data, _ := json.Marshal(body)
	return events.APIGatewayV2HTTPResponse{
		StatusCode: status,
		Headers:    map[string]string{"Content-Type": "application/json"},
		Body:       string(data),
	}
}

func saveOrder(ctx context.Context, orderID string, req CreateOrderRequest, now time.Time) error {
	// Implementation uses TransactWriteItems (see Level 3)
	_ = ctx
	_ = orderID
	_ = req
	_ = now
	return nil // Placeholder — integrate with DynamoDB repo from Level 3
}

func publishOrderCreated(ctx context.Context, orderID, customerID string) error {
	msg := map[string]string{
		"event_type":  "OrderCreated",
		"order_id":    orderID,
		"customer_id": customerID,
	}
	data, _ := json.Marshal(msg)

	_, err := sqsClient.SendMessage(ctx, &sqs.SendMessageInput{
		QueueUrl:       &queueURL,
		MessageBody:    stringPtr(string(data)),
		MessageGroupId: stringPtr(fmt.Sprintf("order-%s", orderID)),
	})
	return err
}

func stringPtr(s string) *string { return &s }

func main() {
	lambda.Start(handler)
}
```

### Tarefa 4.2 — Lambda Function (Java com SnapStart)

```java
// Java Lambda with SnapStart for reduced cold start.
// src/main/java/com/orderflow/lambda/CreateOrderHandler.java
package com.orderflow.lambda;

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.APIGatewayV2HTTPEvent;
import com.amazonaws.services.lambda.runtime.events.APIGatewayV2HTTPResponse;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.crac.Core;
import org.crac.Resource;
import software.amazon.awssdk.services.dynamodb.DynamoDbClient;
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.SendMessageRequest;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

/**
 * Lambda handler optimized for SnapStart.
 *
 * SnapStart: JVM snapshot after init → restore on invocation.
 * - No cold start penalty (~100ms vs ~3s)
 * - Initialize all SDK clients in constructor
 * - CRaC checkpoint: refresh connections on restore
 */
public class CreateOrderHandler
        implements RequestHandler<APIGatewayV2HTTPEvent, APIGatewayV2HTTPResponse>,
                   Resource {

    private final DynamoDbClient dynamoClient;
    private final SqsClient sqsClient;
    private final ObjectMapper mapper;
    private final String tableName;
    private final String queueUrl;

    public CreateOrderHandler() {
        // Initialize SDK clients during init phase (snapshotted by SnapStart).
        this.dynamoClient = DynamoDbClient.create();
        this.sqsClient = SqsClient.create();
        this.mapper = new ObjectMapper();
        this.tableName = System.getenv("TABLE_NAME");
        this.queueUrl = System.getenv("QUEUE_URL");

        // Register for CRaC lifecycle (refresh connections on restore)
        Core.getGlobalContext().register(this);
    }

    @Override
    public void beforeCheckpoint(org.crac.Context<? extends Resource> context) {
        // Called before snapshot — close connections
    }

    @Override
    public void afterRestore(org.crac.Context<? extends Resource> context) {
        // Called after restore — re-establish connections
        // SDK clients auto-reconnect, but warm up here if needed
    }

    @Override
    public APIGatewayV2HTTPResponse handleRequest(
            APIGatewayV2HTTPEvent event, Context context) {
        try {
            var request = mapper.readValue(event.getBody(), CreateOrderRequest.class);

            if (request.customerId() == null || request.items().isEmpty()) {
                return response(400, Map.of("error", "customer_id and items required"));
            }

            var orderId = UUID.randomUUID().toString();
            var now = Instant.now();

            // Save order to DynamoDB (see Level 3 repository)
            saveOrder(orderId, request, now);

            // Publish event to SQS
            publishEvent(orderId, request.customerId());

            return response(201, Map.of(
                    "order_id", orderId,
                    "status", "PENDING",
                    "created_at", now.toString()
            ));
        } catch (Exception e) {
            context.getLogger().log("Error: " + e.getMessage());
            return response(500, Map.of("error", "internal error"));
        }
    }

    private void publishEvent(String orderId, String customerId) {
        sqsClient.sendMessage(SendMessageRequest.builder()
                .queueUrl(queueUrl)
                .messageBody("{\"event_type\":\"OrderCreated\",\"order_id\":\"%s\",\"customer_id\":\"%s\"}"
                        .formatted(orderId, customerId))
                .messageGroupId("order-" + orderId)
                .build());
    }

    private void saveOrder(String orderId, CreateOrderRequest request, Instant now) {
        // Integrate with DynamoDB repo from Level 3
    }

    private APIGatewayV2HTTPResponse response(int status, Object body) {
        try {
            return APIGatewayV2HTTPResponse.builder()
                    .withStatusCode(status)
                    .withHeaders(Map.of("Content-Type", "application/json"))
                    .withBody(mapper.writeValueAsString(body))
                    .build();
        } catch (Exception e) {
            return APIGatewayV2HTTPResponse.builder()
                    .withStatusCode(500)
                    .withBody("{\"error\":\"serialization failed\"}")
                    .build();
        }
    }

    record CreateOrderRequest(String customerId, java.util.List<OrderItemRequest> items) {}
    record OrderItemRequest(String productId, int quantity, double price) {}
}
```

### Tarefa 4.3 — SQS Producer e Consumer (Go)

```go
// internal/messaging/sqs_client.go
package messaging

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/sqs"
	sqstypes "github.com/aws/aws-sdk-go-v2/service/sqs/types"
)

type SQSClient struct {
	client   *sqs.Client
	queueURL string
}

func NewSQSClient(client *sqs.Client, queueURL string) *SQSClient {
	return &SQSClient{client: client, queueURL: queueURL}
}

// OrderEvent represents a domain event published to SQS.
type OrderEvent struct {
	EventType  string    `json:"event_type"`
	OrderID    string    `json:"order_id"`
	CustomerID string    `json:"customer_id"`
	Status     string    `json:"status,omitempty"`
	Timestamp  time.Time `json:"timestamp"`
}

// Publish sends an event to SQS with deduplication ID (FIFO).
func (s *SQSClient) Publish(ctx context.Context, event OrderEvent) error {
	data, err := json.Marshal(event)
	if err != nil {
		return fmt.Errorf("marshal event: %w", err)
	}

	_, err = s.client.SendMessage(ctx, &sqs.SendMessageInput{
		QueueUrl:               aws.String(s.queueURL),
		MessageBody:            aws.String(string(data)),
		MessageGroupId:         aws.String(fmt.Sprintf("order-%s", event.OrderID)),
		MessageDeduplicationId: aws.String(fmt.Sprintf("%s-%s-%d", event.EventType, event.OrderID, event.Timestamp.UnixNano())),
		MessageAttributes: map[string]sqstypes.MessageAttributeValue{
			"event_type": {DataType: aws.String("String"), StringValue: aws.String(event.EventType)},
		},
	})
	if err != nil {
		return fmt.Errorf("send message: %w", err)
	}

	slog.Info("Event published", "event_type", event.EventType, "order_id", event.OrderID)
	return nil
}

// StartConsumer polls SQS and processes messages in batches.
func (s *SQSClient) StartConsumer(ctx context.Context, handler func(ctx context.Context, event OrderEvent) error) error {
	slog.Info("Starting SQS consumer", "queue", s.queueURL)

	for {
		select {
		case <-ctx.Done():
			slog.Info("Consumer stopped")
			return nil
		default:
		}

		output, err := s.client.ReceiveMessage(ctx, &sqs.ReceiveMessageInput{
			QueueUrl:              aws.String(s.queueURL),
			MaxNumberOfMessages:   10,
			WaitTimeSeconds:       20, // Long polling
			VisibilityTimeout:     30,
			MessageAttributeNames: []string{"All"},
		})
		if err != nil {
			slog.Error("Receive error", "error", err)
			time.Sleep(5 * time.Second)
			continue
		}

		for _, msg := range output.Messages {
			var event OrderEvent
			if err := json.Unmarshal([]byte(*msg.Body), &event); err != nil {
				slog.Error("Unmarshal error", "error", err, "message_id", *msg.MessageId)
				// Move to DLQ by not deleting (visibility timeout expires)
				continue
			}

			if err := handler(ctx, event); err != nil {
				slog.Error("Handler error", "error", err, "event", event.EventType, "order_id", event.OrderID)
				continue // Will retry after visibility timeout
			}

			// Delete message on success
			_, _ = s.client.DeleteMessage(ctx, &sqs.DeleteMessageInput{
				QueueUrl:      aws.String(s.queueURL),
				ReceiptHandle: msg.ReceiptHandle,
			})

			slog.Info("Message processed", "event_type", event.EventType, "order_id", event.OrderID)
		}
	}
}
```

### Tarefa 4.4 — SQS Producer e Consumer (Java)

```java
// infrastructure/messaging/SqsMessageService.java
package com.orderflow.infrastructure.messaging;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.*;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

@Service
public class SqsMessageService {

    private final SqsClient sqsClient;
    private final ObjectMapper mapper;
    private final String queueUrl;
    private final OrderEventHandler eventHandler;

    public SqsMessageService(
            SqsClient sqsClient,
            ObjectMapper mapper,
            @Value("${orderflow.sqs.queue-url}") String queueUrl,
            OrderEventHandler eventHandler) {
        this.sqsClient = sqsClient;
        this.mapper = mapper;
        this.queueUrl = queueUrl;
        this.eventHandler = eventHandler;
    }

    /**
     * Publish event to SQS FIFO queue with deduplication.
     */
    public void publishEvent(String eventType, String orderId, String customerId) {
        try {
            var event = new OrderEvent(eventType, orderId, customerId, Instant.now());
            var body = mapper.writeValueAsString(event);

            sqsClient.sendMessage(SendMessageRequest.builder()
                    .queueUrl(queueUrl)
                    .messageBody(body)
                    .messageGroupId("order-" + orderId)
                    .messageDeduplicationId(eventType + "-" + orderId + "-" + UUID.randomUUID())
                    .messageAttributes(Map.of(
                            "event_type", MessageAttributeValue.builder()
                                    .dataType("String")
                                    .stringValue(eventType)
                                    .build()
                    ))
                    .build());
        } catch (Exception e) {
            throw new RuntimeException("Failed to publish event: " + e.getMessage(), e);
        }
    }

    /**
     * Poll SQS with long polling, process in batches.
     */
    @Scheduled(fixedDelay = 100)
    public void pollMessages() {
        var response = sqsClient.receiveMessage(ReceiveMessageRequest.builder()
                .queueUrl(queueUrl)
                .maxNumberOfMessages(10)
                .waitTimeSeconds(20)
                .visibilityTimeout(30)
                .messageSystemAttributeNames(MessageSystemAttributeName.ALL)
                .messageAttributeNames("All")
                .build());

        for (var message : response.messages()) {
            try {
                var event = mapper.readValue(message.body(), OrderEvent.class);
                eventHandler.handle(event);

                sqsClient.deleteMessage(DeleteMessageRequest.builder()
                        .queueUrl(queueUrl)
                        .receiptHandle(message.receiptHandle())
                        .build());
            } catch (Exception e) {
                // Message will be retried after visibility timeout.
                // Moves to DLQ after maxReceiveCount.
            }
        }
    }

    public record OrderEvent(String eventType, String orderId, String customerId, Instant timestamp) {}
}
```

### Tarefa 4.5 — SNS Fan-Out (Go)

```go
// internal/messaging/sns_publisher.go
package messaging

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/sns"
	snstypes "github.com/aws/aws-sdk-go-v2/service/sns/types"
)

type SNSPublisher struct {
	client   *sns.Client
	topicArn string
}

func NewSNSPublisher(client *sns.Client, topicArn string) *SNSPublisher {
	return &SNSPublisher{client: client, topicArn: topicArn}
}

// PublishOrderEvent publishes to SNS with message attributes for filtering.
// Subscribers can filter by event_type and status.
func (p *SNSPublisher) PublishOrderEvent(ctx context.Context, event OrderEvent) error {
	data, err := json.Marshal(event)
	if err != nil {
		return fmt.Errorf("marshal event: %w", err)
	}

	_, err = p.client.Publish(ctx, &sns.PublishInput{
		TopicArn: aws.String(p.topicArn),
		Message:  aws.String(string(data)),
		MessageAttributes: map[string]snstypes.MessageAttributeValue{
			"event_type": {DataType: aws.String("String"), StringValue: aws.String(event.EventType)},
			"status":     {DataType: aws.String("String"), StringValue: aws.String(event.Status)},
		},
		// Subject is used for email subscriptions
		Subject: aws.String(fmt.Sprintf("OrderFlow: %s", event.EventType)),
	})
	if err != nil {
		return fmt.Errorf("publish to SNS: %w", err)
	}

	slog.Info("Published to SNS", "event_type", event.EventType, "topic", p.topicArn)
	return nil
}

// SetupFanOut creates SNS topic → SQS subscriptions with filter policies.
func (p *SNSPublisher) SetupFanOut(ctx context.Context, subscriptions []FanOutSubscription) error {
	for _, sub := range subscriptions {
		filterPolicy, _ := json.Marshal(sub.FilterPolicy)

		_, err := p.client.Subscribe(ctx, &sns.SubscribeInput{
			TopicArn: aws.String(p.topicArn),
			Protocol: aws.String("sqs"),
			Endpoint: aws.String(sub.QueueArn),
			Attributes: map[string]string{
				"FilterPolicy":      string(filterPolicy),
				"RawMessageDelivery": "true",
			},
		})
		if err != nil {
			return fmt.Errorf("subscribe %s: %w", sub.Name, err)
		}

		slog.Info("Subscription created", "name", sub.Name, "queue", sub.QueueArn)
	}

	return nil
}

type FanOutSubscription struct {
	Name         string
	QueueArn     string
	FilterPolicy map[string]any
}
```

### Tarefa 4.6 — EventBridge Event Router (Go)

```go
// internal/messaging/eventbridge_client.go
package messaging

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/eventbridge"
	ebtypes "github.com/aws/aws-sdk-go-v2/service/eventbridge/types"
)

type EventBridgeClient struct {
	client  *eventbridge.Client
	busName string
	source  string
}

func NewEventBridgeClient(client *eventbridge.Client, busName, source string) *EventBridgeClient {
	return &EventBridgeClient{client: client, busName: busName, source: source}
}

// OrderDomainEvent represents an event for EventBridge.
type OrderDomainEvent struct {
	EventType  string         `json:"event_type"`
	OrderID    string         `json:"order_id"`
	CustomerID string         `json:"customer_id"`
	Status     string         `json:"status"`
	Amount     float64        `json:"amount,omitempty"`
	Metadata   map[string]any `json:"metadata,omitempty"`
}

// PutEvent sends a domain event to EventBridge.
func (e *EventBridgeClient) PutEvent(ctx context.Context, detailType string, event OrderDomainEvent) error {
	detail, err := json.Marshal(event)
	if err != nil {
		return fmt.Errorf("marshal event: %w", err)
	}

	output, err := e.client.PutEvents(ctx, &eventbridge.PutEventsInput{
		Entries: []ebtypes.PutEventsRequestEntry{
			{
				EventBusName: aws.String(e.busName),
				Source:       aws.String(e.source),
				DetailType:   aws.String(detailType),
				Detail:       aws.String(string(detail)),
				Time:         aws.Time(time.Now().UTC()),
			},
		},
	})
	if err != nil {
		return fmt.Errorf("put events: %w", err)
	}

	if output.FailedEntryCount > 0 {
		return fmt.Errorf("failed to put %d entries", output.FailedEntryCount)
	}

	slog.Info("Event published to EventBridge",
		"detail_type", detailType,
		"order_id", event.OrderID,
		"bus", e.busName)
	return nil
}

// CreateRule creates an EventBridge rule with event pattern.
func (e *EventBridgeClient) CreateRule(ctx context.Context, ruleName string, pattern EventPattern) error {
	patternJSON, err := json.Marshal(pattern)
	if err != nil {
		return fmt.Errorf("marshal pattern: %w", err)
	}

	_, err = e.client.PutRule(ctx, &eventbridge.PutRuleInput{
		Name:         aws.String(ruleName),
		EventBusName: aws.String(e.busName),
		EventPattern: aws.String(string(patternJSON)),
		State:        ebtypes.RuleStateEnabled,
		Description:  aws.String(fmt.Sprintf("Route %s events", ruleName)),
	})
	if err != nil {
		return fmt.Errorf("create rule %s: %w", ruleName, err)
	}

	slog.Info("EventBridge rule created", "rule", ruleName)
	return nil
}

// AddTarget adds a target (Lambda, SQS, etc.) to an EventBridge rule.
func (e *EventBridgeClient) AddTarget(ctx context.Context, ruleName, targetArn, targetID string) error {
	_, err := e.client.PutTargets(ctx, &eventbridge.PutTargetsInput{
		Rule:         aws.String(ruleName),
		EventBusName: aws.String(e.busName),
		Targets: []ebtypes.Target{
			{
				Arn: aws.String(targetArn),
				Id:  aws.String(targetID),
			},
		},
	})
	if err != nil {
		return fmt.Errorf("add target to %s: %w", ruleName, err)
	}

	return nil
}

// EventPattern defines an EventBridge event pattern JSON.
type EventPattern struct {
	Source     []string               `json:"source,omitempty"`
	DetailType []string              `json:"detail-type,omitempty"`
	Detail     map[string]any        `json:"detail,omitempty"`
}
```

### Tarefa 4.7 — EventBridge Event Router (Java)

```java
// infrastructure/messaging/EventBridgeService.java
package com.orderflow.infrastructure.messaging;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.eventbridge.EventBridgeClient;
import software.amazon.awssdk.services.eventbridge.model.*;

import java.time.Instant;
import java.util.Map;

@Service
public class EventBridgeService {

    private final EventBridgeClient ebClient;
    private final ObjectMapper mapper;
    private final String busName;
    private final String source;

    public EventBridgeService(
            EventBridgeClient ebClient,
            ObjectMapper mapper,
            @Value("${orderflow.eventbridge.bus-name}") String busName,
            @Value("${orderflow.eventbridge.source:com.orderflow}") String source) {
        this.ebClient = ebClient;
        this.mapper = mapper;
        this.busName = busName;
        this.source = source;
    }

    /**
     * Publish a domain event to EventBridge.
     */
    public void publishEvent(String detailType, Object event) {
        try {
            var detail = mapper.writeValueAsString(event);

            var response = ebClient.putEvents(PutEventsRequest.builder()
                    .entries(PutEventsRequestEntry.builder()
                            .eventBusName(busName)
                            .source(source)
                            .detailType(detailType)
                            .detail(detail)
                            .time(Instant.now())
                            .build())
                    .build());

            if (response.failedEntryCount() > 0) {
                throw new RuntimeException("Failed to put %d entries".formatted(response.failedEntryCount()));
            }
        } catch (Exception e) {
            throw new RuntimeException("Failed to publish event: " + e.getMessage(), e);
        }
    }

    /**
     * Create an EventBridge rule with pattern matching.
     */
    public void createRule(String ruleName, Map<String, Object> eventPattern) {
        try {
            var patternJson = mapper.writeValueAsString(eventPattern);

            ebClient.putRule(PutRuleRequest.builder()
                    .name(ruleName)
                    .eventBusName(busName)
                    .eventPattern(patternJson)
                    .state(RuleState.ENABLED)
                    .description("Route events for " + ruleName)
                    .build());
        } catch (Exception e) {
            throw new RuntimeException("Failed to create rule: " + e.getMessage(), e);
        }
    }

    public void addTarget(String ruleName, String targetArn, String targetId) {
        ebClient.putTargets(PutTargetsRequest.builder()
                .rule(ruleName)
                .eventBusName(busName)
                .targets(Target.builder()
                        .arn(targetArn)
                        .id(targetId)
                        .build())
                .build());
    }
}
```

### Tarefa 4.8 — Step Functions Saga (CLI + SDK)

```bash
#!/bin/bash
# scripts/create-step-function.sh — Create OrderFlow Saga state machine

set -euo pipefail

ENDPOINT="http://localhost:4566"
REGION="us-east-1"
AWS="aws --endpoint-url=$ENDPOINT --region=$REGION"

DEFINITION='{
  "Comment": "OrderFlow Saga — Orchestration Pattern",
  "StartAt": "CreateOrder",
  "States": {
    "CreateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-create-order",
      "ResultPath": "$.order",
      "Next": "ReserveInventory",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "CancelOrder",
        "ResultPath": "$.error"
      }]
    },
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-reserve-inventory",
      "ResultPath": "$.inventory",
      "Next": "ProcessPayment",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "CancelOrder",
        "ResultPath": "$.error"
      }],
      "Retry": [{
        "ErrorEquals": ["States.TaskFailed"],
        "IntervalSeconds": 2,
        "MaxAttempts": 3,
        "BackoffRate": 2.0
      }]
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-process-payment",
      "ResultPath": "$.payment",
      "Next": "ConfirmOrder",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "RefundPayment",
        "ResultPath": "$.error"
      }]
    },
    "ConfirmOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-confirm-order",
      "End": true
    },
    "RefundPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-refund-payment",
      "ResultPath": "$.refund",
      "Next": "ReleaseInventory"
    },
    "ReleaseInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-release-inventory",
      "ResultPath": "$.release",
      "Next": "CancelOrder"
    },
    "CancelOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:000000000000:function:orderflow-cancel-order",
      "End": true
    }
  }
}'

echo "=== Creating Step Functions State Machine ==="
STATE_MACHINE_ARN=$($AWS stepfunctions create-state-machine \
  --name orderflow-saga \
  --definition "$DEFINITION" \
  --role-arn "arn:aws:iam::000000000000:role/step-functions-role" \
  --type STANDARD \
  --query 'stateMachineArn' --output text)

echo "State Machine ARN: $STATE_MACHINE_ARN"

echo "=== Starting Test Execution ==="
EXECUTION_ARN=$($AWS stepfunctions start-execution \
  --state-machine-arn "$STATE_MACHINE_ARN" \
  --name "test-$(date +%s)" \
  --input '{"customer_id":"cust-1","items":[{"product_id":"prod-1","qty":2,"price":29.99}]}' \
  --query 'executionArn' --output text)

echo "Execution ARN: $EXECUTION_ARN"
```

### Tarefa 4.9 — Step Functions SDK (Go)

```go
// internal/workflow/saga_client.go
package workflow

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/sfn"
	sfntypes "github.com/aws/aws-sdk-go-v2/service/sfn/types"
)

type SagaClient struct {
	client          *sfn.Client
	stateMachineArn string
}

func NewSagaClient(client *sfn.Client, stateMachineArn string) *SagaClient {
	return &SagaClient{client: client, stateMachineArn: stateMachineArn}
}

type SagaInput struct {
	CustomerID string     `json:"customer_id"`
	Items      []SagaItem `json:"items"`
}

type SagaItem struct {
	ProductID string  `json:"product_id"`
	Quantity  int     `json:"qty"`
	Price     float64 `json:"price"`
}

type ExecutionResult struct {
	ExecutionArn string `json:"execution_arn"`
	Status       string `json:"status"`
	StartDate    string `json:"start_date"`
	Output       string `json:"output,omitempty"`
}

// StartSaga initiates the order processing saga.
func (s *SagaClient) StartSaga(ctx context.Context, input SagaInput) (*ExecutionResult, error) {
	inputJSON, err := json.Marshal(input)
	if err != nil {
		return nil, fmt.Errorf("marshal input: %w", err)
	}

	execName := fmt.Sprintf("order-%d", time.Now().UnixNano())

	output, err := s.client.StartExecution(ctx, &sfn.StartExecutionInput{
		StateMachineArn: aws.String(s.stateMachineArn),
		Name:            aws.String(execName),
		Input:           aws.String(string(inputJSON)),
	})
	if err != nil {
		return nil, fmt.Errorf("start execution: %w", err)
	}

	slog.Info("Saga started",
		"execution_arn", *output.ExecutionArn,
		"start_date", output.StartDate)

	return &ExecutionResult{
		ExecutionArn: *output.ExecutionArn,
		Status:       "RUNNING",
		StartDate:    output.StartDate.Format(time.RFC3339),
	}, nil
}

// GetExecution returns the current status of a saga execution.
func (s *SagaClient) GetExecution(ctx context.Context, executionArn string) (*ExecutionResult, error) {
	output, err := s.client.DescribeExecution(ctx, &sfn.DescribeExecutionInput{
		ExecutionArn: aws.String(executionArn),
	})
	if err != nil {
		return nil, fmt.Errorf("describe execution: %w", err)
	}

	result := &ExecutionResult{
		ExecutionArn: executionArn,
		Status:       string(output.Status),
		StartDate:    output.StartDate.Format(time.RFC3339),
	}

	if output.Output != nil {
		result.Output = *output.Output
	}

	return result, nil
}

// ListExecutions returns recent saga executions filtered by status.
func (s *SagaClient) ListExecutions(ctx context.Context, statusFilter sfntypes.ExecutionStatus) ([]ExecutionResult, error) {
	output, err := s.client.ListExecutions(ctx, &sfn.ListExecutionsInput{
		StateMachineArn: aws.String(s.stateMachineArn),
		StatusFilter:    statusFilter,
		MaxResults:      20,
	})
	if err != nil {
		return nil, fmt.Errorf("list executions: %w", err)
	}

	results := make([]ExecutionResult, 0, len(output.Executions))
	for _, exec := range output.Executions {
		results = append(results, ExecutionResult{
			ExecutionArn: *exec.ExecutionArn,
			Status:       string(exec.Status),
			StartDate:    exec.StartDate.Format(time.RFC3339),
		})
	}

	return results, nil
}
```

### Tarefa 4.10 — Setup LocalStack Serverless Stack (CLI)

```bash
#!/bin/bash
# scripts/setup-serverless.sh — Create all serverless resources in LocalStack

set -euo pipefail

ENDPOINT="http://localhost:4566"
REGION="us-east-1"
AWS="aws --endpoint-url=$ENDPOINT --region=$REGION"

echo "=== Creating SQS Queues ==="

# DLQ first
$AWS sqs create-queue --queue-name orderflow-dlq.fifo \
  --attributes '{
    "FifoQueue": "true",
    "MessageRetentionPeriod": "1209600"
  }'

DLQ_ARN=$($AWS sqs get-queue-attributes \
  --queue-url "$ENDPOINT/000000000000/orderflow-dlq.fifo" \
  --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)

# Main queue with DLQ redrive
$AWS sqs create-queue --queue-name orderflow-orders.fifo \
  --attributes "{
    \"FifoQueue\": \"true\",
    \"ContentBasedDeduplication\": \"false\",
    \"VisibilityTimeout\": \"30\",
    \"RedrivePolicy\": \"{\\\"deadLetterTargetArn\\\":\\\"$DLQ_ARN\\\",\\\"maxReceiveCount\\\":\\\"3\\\"}\"
  }"

echo "=== Creating SNS Topic ==="

TOPIC_ARN=$($AWS sns create-topic --name orderflow-events \
  --query 'TopicArn' --output text)

echo "Topic ARN: $TOPIC_ARN"

# Fan-out: SNS → multiple SQS queues with filter policies
# Queue for notifications
$AWS sqs create-queue --queue-name orderflow-notifications
NOTIFY_QUEUE_ARN=$($AWS sqs get-queue-attributes \
  --queue-url "$ENDPOINT/000000000000/orderflow-notifications" \
  --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)

$AWS sns subscribe --topic-arn "$TOPIC_ARN" \
  --protocol sqs --notification-endpoint "$NOTIFY_QUEUE_ARN" \
  --attributes '{"FilterPolicy":"{\"event_type\":[\"OrderConfirmed\",\"OrderCancelled\"]}","RawMessageDelivery":"true"}'

# Queue for analytics
$AWS sqs create-queue --queue-name orderflow-analytics
ANALYTICS_ARN=$($AWS sqs get-queue-attributes \
  --queue-url "$ENDPOINT/000000000000/orderflow-analytics" \
  --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)

$AWS sns subscribe --topic-arn "$TOPIC_ARN" \
  --protocol sqs --notification-endpoint "$ANALYTICS_ARN" \
  --attributes '{"RawMessageDelivery":"true"}'

echo "=== Creating EventBridge Event Bus ==="

$AWS events create-event-bus --name orderflow-bus

# Rule: route OrderCreated events to SQS
$AWS events put-rule \
  --name order-created-rule \
  --event-bus-name orderflow-bus \
  --event-pattern '{
    "source": ["com.orderflow"],
    "detail-type": ["OrderCreated"]
  }' \
  --state ENABLED

echo "=== Creating Lambda Functions ==="

# Create a simple Lambda for testing
cat > /tmp/lambda-handler.py << 'EOF'
import json
def handler(event, context):
    print(json.dumps(event))
    return {"statusCode": 200, "body": json.dumps({"message": "processed"})}
EOF

cd /tmp && zip -j lambda.zip lambda-handler.py

for FUNC in create-order reserve-inventory process-payment confirm-order \
            refund-payment release-inventory cancel-order; do
  $AWS lambda create-function \
    --function-name "orderflow-$FUNC" \
    --runtime python3.12 \
    --handler lambda-handler.handler \
    --zip-file fileb:///tmp/lambda.zip \
    --role "arn:aws:iam::000000000000:role/lambda-role" \
    --timeout 30 \
    --memory-size 256

  echo "Created Lambda: orderflow-$FUNC"
done

echo "=== Serverless stack created ==="
echo "SQS FIFO Queue: orderflow-orders.fifo"
echo "SNS Topic: $TOPIC_ARN"
echo "EventBridge Bus: orderflow-bus"
echo "Lambda Functions: 7 created"
```

---

## Messaging Decision Tree

```
Qual serviço de messaging usar?
═══════════════════════════════

  Preciso de...
      │
      ├── 1-to-1 async processing?
      │     └── SQS ✓
      │         (order processing, batch jobs)
      │         ├── Ordering matters? → FIFO Queue
      │         └── Just decouple?    → Standard Queue
      │
      ├── 1-to-N fan-out (broadcast)?
      │     └── SNS → SQS ✓
      │         (notifications, multi-consumer)
      │
      ├── Event routing with patterns?
      │     └── EventBridge ✓
      │         (cross-service events, rules, archive)
      │
      ├── Multi-step workflow with retries?
      │     └── Step Functions ✓
      │         (sagas, orchestration, approval flows)
      │         ├── <5 min, high volume? → Express
      │         └── Long-running?        → Standard
      │
      └── Real-time streaming?
            └── Kinesis ✓
                (clickstream, IoT, log aggregation)

  SQS Standard vs FIFO:
  ┌──────────────────┬───────────────────────────────┐
  │  Standard         │  FIFO                         │
  ├──────────────────┼───────────────────────────────┤
  │  At-least-once    │  Exactly-once                 │
  │  Best-effort order│  Strict ordering (per group)  │
  │  Unlimited TPS    │  300 msg/s (3000 with batch)  │
  │  Simpler          │  Requires group + dedup ID    │
  └──────────────────┴───────────────────────────────┘
```

---

## Anti-Patterns

| Anti-Pattern | Por que está errado | Como corrigir |
|---|---|---|
| Lambda em VPC sem necessidade | +10s cold start, ENI provisioning | VPC apenas quando acessa RDS/ElastiCache |
| SDK client criado dentro do handler | Re-cria TCP connection a cada invocação | Criar clients no `init()` / construtor |
| Lambda sem timeout definido | Execução infinita, custo descontrolado | Definir timeout conservador (30s default) |
| SQS sem DLQ | Poison messages travam a fila | DLQ com maxReceiveCount=3 |
| SQS sem visibility timeout adequado | Mensagem reprocessada antes de completar | VisibilityTimeout > tempo de processamento |
| SNS sem filter policy | Todos consumers recebem tudo | Filtrar por atributos (event_type) |
| EventBridge sem DLQ | Eventos dropped silenciosamente | DLQ nos targets |
| Step Functions com Lambda >15min | Timeout e custo desnecessário | Activity ou Map state paralelo |

---

## Critérios de Aceite

### Lambda
- [ ] Go Lambda compilado para `provided.al2023` + ARM64
- [ ] Java Lambda configurado com SnapStart
- [ ] SDK clients inicializados fora do handler (reuse)
- [ ] Environment variables para configuração (TABLE_NAME, QUEUE_URL)
- [ ] Structured logging com request ID

### SQS
- [ ] FIFO queue com DLQ (maxReceiveCount=3)
- [ ] Producer com MessageGroupId e DeduplicationId
- [ ] Consumer com long polling (WaitTimeSeconds=20)
- [ ] Message deletion após processamento bem-sucedido
- [ ] Batch receive (MaxNumberOfMessages=10)

### SNS
- [ ] Topic com fan-out para múltiplos SQS queues
- [ ] Filter policies por event_type
- [ ] RawMessageDelivery habilitado

### EventBridge
- [ ] Custom event bus criado
- [ ] Rules com event patterns (source, detail-type)
- [ ] Targets configurados (Lambda, SQS)

### Step Functions
- [ ] Saga state machine com 7 estados
- [ ] Catch blocks para compensação
- [ ] Retry com exponential backoff
- [ ] Execução testada via CLI

---

## Definição de Pronto (DoD)

- [ ] Lambda functions: Go (`provided.al2023`) + Java (SnapStart) implementados
- [ ] SQS FIFO: producer/consumer com DLQ, go + java
- [ ] SNS fan-out com filter policies
- [ ] EventBridge event bus com rules e targets
- [ ] Step Functions saga com compensação
- [ ] Setup script para LocalStack completo
- [ ] Testes de integração com LocalStack
- [ ] Commit semântico: `feat(level-4): add Lambda, SQS, SNS, EventBridge, Step Functions`

---

## Checklist

- [ ] Lambda Go: `provided.al2023`, ARM64, SDK clients no `init()`
- [ ] Lambda Java: SnapStart, CRaC lifecycle, SDK clients no construtor
- [ ] SQS: FIFO + DLQ, producer/consumer Go e Java
- [ ] SNS: fan-out, filter policies, RawMessageDelivery
- [ ] EventBridge: custom bus, rules, event patterns
- [ ] Step Functions: saga definition, retry, catch, compensate
- [ ] Scripts: `setup-serverless.sh` e `create-step-function.sh`

---

## Exercícios Extras

1. **Lambda Layers** — Crie um Lambda Layer com shared utilities (logging, error handling) reutilizável entre todas as funções.
2. **EventBridge Archive + Replay** — Configure archive no event bus e implemente replay de eventos para reprocessamento.
3. **SQS Batch Operations** — Implemente `SendMessageBatch` e `DeleteMessageBatch` para processar até 10 mensagens por chamada — meça throughput vs mensagem individual.
4. **Step Functions Map State** — Adicione um Map state paralelo para processar múltiplos items de um order simultaneamente.
5. **API Gateway HTTP API** — Configure um API Gateway HTTP API com routes → Lambda, JWT Authorizer, e throttling (rate limit + burst).
