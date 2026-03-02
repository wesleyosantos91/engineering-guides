# Level 3 — Storage & Databases

> **Objetivo:** Dominar S3 para storage de objetos (lifecycle, encryption, presigned URLs,
> event notifications) e DynamoDB para dados operacionais (single-table design, GSI/LSI,
> DynamoDB Streams, DAX caching). Introduzir RDS/Aurora e ElastiCache.

---

## Objetivo de Aprendizado

- Implementar operações avançadas de S3 via SDK (multipart upload, presigned URLs, lifecycle)
- Projetar DynamoDB single-table design com access patterns definidos
- Usar GSI/LSI para query patterns secundários
- Implementar DynamoDB Streams para event-driven processing
- Configurar DAX como cache transparente para DynamoDB
- Entender quando usar RDS/Aurora vs DynamoDB
- Configurar ElastiCache (Redis) para caching de reads

---

## Escopo Funcional

```
Level 3 — Storage & Database Architecture
════════════════════════════════════════════════

  ┌─────────────────────────────────────────────────────────────┐
  │                     ECS Services (Level 2)                  │
  │                                                             │
  │  ┌──────────────────┐         ┌──────────────────┐          │
  │  │   Order API (Go) │         │  Order API (Java) │          │
  │  └────┬──────┬──────┘         └────┬──────┬──────┘          │
  │       │      │                     │      │                 │
  └───────┼──────┼─────────────────────┼──────┼─────────────────┘
          │      │                     │      │
    ┌─────▼──┐ ┌─▼──────┐      ┌──────▼─┐ ┌──▼────────┐
    │   S3   │ │DynamoDB │      │  DAX   │ │ElastiCache│
    │        │ │        │      │(cache) │ │  (Redis)  │
    └───┬────┘ └───┬────┘      └───┬────┘ └───────────┘
        │          │               │
        │          │          ┌────▼────┐
        │          │          │DynamoDB │
        │          │          │ (real)  │
        │          │          └────┬────┘
        │          │               │
   ┌────▼────┐ ┌──▼────────┐     │
   │ S3 Event│ │ DynamoDB   │     │
   │Notific. │ │ Streams    │     │
   └────┬────┘ └──┬────────┘     │
        │          │              │
   ┌────▼────┐ ┌──▼────────┐    │
   │ Lambda  │ │  Lambda    │    │
   │(process)│ │ (process)  │    │
   └─────────┘ └────────────┘    │
                                  │
                            ┌─────▼─────┐
                            │ RDS/Aurora │
                            │(analytics)│
                            └───────────┘

  Access Patterns:
  ═══════════════
  1. Get order by ID         → DynamoDB (PK=ORDER#id, SK=METADATA)
  2. List orders by customer → DynamoDB (GSI1: CUSTOMER#id)
  3. List orders by status   → DynamoDB (GSI2: STATUS#status)
  4. Upload receipt           → S3 (receipts/order-id/timestamp.pdf)
  5. Get receipt URL          → S3 Presigned URL (15 min TTL)
  6. Cache hot orders         → ElastiCache (Redis, 5 min TTL)
  7. Stream order changes     → DynamoDB Streams → Lambda
  8. Analytical queries       → RDS Aurora (aggregations)
```

---

## Tarefas

### Tarefa 3.1 — S3 Advanced Operations (Go)

```go
// internal/repository/s3_store.go
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
	s3types "github.com/aws/aws-sdk-go-v2/service/s3/types"
	"github.com/aws/aws-sdk-go-v2/feature/s3/manager"
	"github.com/aws/aws-sdk-go-v2/service/s3/presign"
)

type S3Store struct {
	client    *s3.Client
	presigner *s3.PresignClient
	uploader  *manager.Uploader
	bucket    string
}

func NewS3Store(client *s3.Client, bucket string) *S3Store {
	return &S3Store{
		client:    client,
		presigner: s3.NewPresignClient(client),
		uploader:  manager.NewUploader(client),
		bucket:    bucket,
	}
}

// UploadWithSSE uploads a file with server-side encryption (AES256).
func (s *S3Store) UploadWithSSE(ctx context.Context, key string, data []byte, contentType string) error {
	_, err := s.client.PutObject(ctx, &s3.PutObjectInput{
		Bucket:               aws.String(s.bucket),
		Key:                  aws.String(key),
		Body:                 bytes.NewReader(data),
		ContentType:          aws.String(contentType),
		ServerSideEncryption: s3types.ServerSideEncryptionAes256,
		Metadata: map[string]string{
			"uploaded-at": time.Now().UTC().Format(time.RFC3339),
		},
	})
	if err != nil {
		return fmt.Errorf("upload with SSE: %w", err)
	}

	slog.Info("Uploaded with SSE", "bucket", s.bucket, "key", key)
	return nil
}

// MultipartUpload uploads large files (>5MB) using multipart upload.
func (s *S3Store) MultipartUpload(ctx context.Context, key string, reader io.Reader, contentType string) error {
	_, err := s.uploader.Upload(ctx, &s3.PutObjectInput{
		Bucket:               aws.String(s.bucket),
		Key:                  aws.String(key),
		Body:                 reader,
		ContentType:          aws.String(contentType),
		ServerSideEncryption: s3types.ServerSideEncryptionAes256,
	})
	if err != nil {
		return fmt.Errorf("multipart upload: %w", err)
	}

	slog.Info("Multipart upload complete", "key", key)
	return nil
}

// GeneratePresignedURL creates a temporary URL for downloading an object.
func (s *S3Store) GeneratePresignedURL(ctx context.Context, key string, ttl time.Duration) (string, error) {
	result, err := s.presigner.PresignGetObject(ctx, &s3.GetObjectInput{
		Bucket: aws.String(s.bucket),
		Key:    aws.String(key),
	}, func(opts *s3.PresignOptions) {
		opts.Expires = ttl
	})
	if err != nil {
		return "", fmt.Errorf("presign URL: %w", err)
	}

	slog.Info("Presigned URL generated", "key", key, "ttl", ttl)
	return result.URL, nil
}

// GeneratePresignedUploadURL creates a presigned URL for uploading.
func (s *S3Store) GeneratePresignedUploadURL(ctx context.Context, key, contentType string, ttl time.Duration) (string, error) {
	result, err := s.presigner.PresignPutObject(ctx, &s3.PutObjectInput{
		Bucket:      aws.String(s.bucket),
		Key:         aws.String(key),
		ContentType: aws.String(contentType),
	}, func(opts *s3.PresignOptions) {
		opts.Expires = ttl
	})
	if err != nil {
		return "", fmt.Errorf("presign upload URL: %w", err)
	}

	return result.URL, nil
}

// SetLifecyclePolicy configures lifecycle rules for the bucket.
func (s *S3Store) SetLifecyclePolicy(ctx context.Context) error {
	_, err := s.client.PutBucketLifecycleConfiguration(ctx, &s3.PutBucketLifecycleConfigurationInput{
		Bucket: aws.String(s.bucket),
		LifecycleConfiguration: &s3types.BucketLifecycleConfiguration{
			Rules: []s3types.LifecycleRule{
				{
					ID:     aws.String("archive-old-receipts"),
					Status: s3types.ExpirationStatusEnabled,
					Filter: &s3types.LifecycleRuleFilterMemberPrefix{Value: "receipts/"},
					Transitions: []s3types.Transition{
						{
							Days:         aws.Int32(90),
							StorageClass: s3types.TransitionStorageClassStandardIa,
						},
						{
							Days:         aws.Int32(365),
							StorageClass: s3types.TransitionStorageClassGlacier,
						},
					},
				},
				{
					ID:     aws.String("cleanup-temp"),
					Status: s3types.ExpirationStatusEnabled,
					Filter: &s3types.LifecycleRuleFilterMemberPrefix{Value: "temp/"},
					Expiration: &s3types.LifecycleExpiration{
						Days: aws.Int32(7),
					},
				},
			},
		},
	})
	if err != nil {
		return fmt.Errorf("set lifecycle: %w", err)
	}

	slog.Info("Lifecycle policy configured", "bucket", s.bucket)
	return nil
}

// EnableEventNotification configures S3 events to trigger SQS.
func (s *S3Store) EnableEventNotification(ctx context.Context, queueArn string) error {
	_, err := s.client.PutBucketNotificationConfiguration(ctx, &s3.PutBucketNotificationConfigurationInput{
		Bucket: aws.String(s.bucket),
		NotificationConfiguration: &s3types.NotificationConfiguration{
			QueueConfigurations: []s3types.QueueConfiguration{
				{
					QueueArn: aws.String(queueArn),
					Events: []s3types.Event{
						s3types.EventS3ObjectCreated,
					},
					Filter: &s3types.NotificationConfigurationFilter{
						Key: &s3types.S3KeyFilter{
							FilterRules: []s3types.FilterRule{
								{Name: s3types.FilterRuleNamePrefix, Value: aws.String("receipts/")},
								{Name: s3types.FilterRuleNameSuffix, Value: aws.String(".pdf")},
							},
						},
					},
				},
			},
		},
	})
	if err != nil {
		return fmt.Errorf("set notification: %w", err)
	}

	slog.Info("Event notification configured", "bucket", s.bucket, "queue", queueArn)
	return nil
}
```

### Tarefa 3.2 — S3 Advanced Operations (Java)

```java
// infrastructure/storage/S3StorageService.java
package com.orderflow.infrastructure.storage;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;
import software.amazon.awssdk.services.s3.presigner.S3Presigner;
import software.amazon.awssdk.services.s3.presigner.model.*;
import software.amazon.awssdk.transfer.s3.S3TransferManager;
import software.amazon.awssdk.transfer.s3.model.*;

import java.io.InputStream;
import java.net.URL;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;

@Service
public class S3StorageService {

    private final S3Client s3Client;
    private final S3Presigner presigner;
    private final String bucket;

    public S3StorageService(
            S3Client s3Client,
            S3Presigner presigner,
            @Value("${orderflow.s3.bucket-name}") String bucket) {
        this.s3Client = s3Client;
        this.presigner = presigner;
        this.bucket = bucket;
    }

    public void uploadWithEncryption(String key, byte[] data, String contentType) {
        s3Client.putObject(
                PutObjectRequest.builder()
                        .bucket(bucket)
                        .key(key)
                        .contentType(contentType)
                        .serverSideEncryption(ServerSideEncryption.AES256)
                        .metadata(Map.of("uploaded-at", Instant.now().toString()))
                        .build(),
                RequestBody.fromBytes(data)
        );
    }

    /**
     * Generate a presigned download URL with configurable TTL.
     */
    public URL generatePresignedDownloadUrl(String key, Duration ttl) {
        var presignedRequest = presigner.presignGetObject(
                GetObjectPresignRequest.builder()
                        .signatureDuration(ttl)
                        .getObjectRequest(GetObjectRequest.builder()
                                .bucket(bucket)
                                .key(key)
                                .build())
                        .build());

        return presignedRequest.url();
    }

    /**
     * Generate a presigned upload URL for client-side uploads.
     */
    public URL generatePresignedUploadUrl(String key, String contentType, Duration ttl) {
        var presignedRequest = presigner.presignPutObject(
                PutObjectPresignRequest.builder()
                        .signatureDuration(ttl)
                        .putObjectRequest(PutObjectRequest.builder()
                                .bucket(bucket)
                                .key(key)
                                .contentType(contentType)
                                .build())
                        .build());

        return presignedRequest.url();
    }

    /**
     * Configure lifecycle rules: archive to IA after 90 days, Glacier after 365.
     */
    public void configureLifecycle() {
        s3Client.putBucketLifecycleConfiguration(PutBucketLifecycleConfigurationRequest.builder()
                .bucket(bucket)
                .lifecycleConfiguration(BucketLifecycleConfiguration.builder()
                        .rules(
                                LifecycleRule.builder()
                                        .id("archive-old-receipts")
                                        .status(ExpirationStatus.ENABLED)
                                        .filter(LifecycleRuleFilter.builder()
                                                .prefix("receipts/")
                                                .build())
                                        .transitions(
                                                Transition.builder()
                                                        .days(90)
                                                        .storageClass(TransitionStorageClass.STANDARD_IA)
                                                        .build(),
                                                Transition.builder()
                                                        .days(365)
                                                        .storageClass(TransitionStorageClass.GLACIER)
                                                        .build()
                                        )
                                        .build(),
                                LifecycleRule.builder()
                                        .id("cleanup-temp")
                                        .status(ExpirationStatus.ENABLED)
                                        .filter(LifecycleRuleFilter.builder()
                                                .prefix("temp/")
                                                .build())
                                        .expiration(LifecycleExpiration.builder()
                                                .days(7)
                                                .build())
                                        .build()
                        )
                        .build())
                .build());
    }
}
```

### Tarefa 3.3 — DynamoDB Single-Table Design

```
DynamoDB Single-Table Design para OrderFlow
═══════════════════════════════════════════════

Table: orderflow-orders

  ┌─────────────────────┬──────────────────┬──────────────────────────────┐
  │ PK                  │ SK               │ Entity / Data                │
  ├─────────────────────┼──────────────────┼──────────────────────────────┤
  │ ORDER#uuid-1        │ METADATA         │ Order (status, total, etc)   │
  │ ORDER#uuid-1        │ ITEM#prod-1      │ OrderItem (qty, price)       │
  │ ORDER#uuid-1        │ ITEM#prod-2      │ OrderItem (qty, price)       │
  │ ORDER#uuid-1        │ EVENT#2024-01-01 │ OrderEvent (status change)   │
  │ ORDER#uuid-1        │ PAYMENT#pay-1    │ Payment (method, status)     │
  │ CUSTOMER#cust-1     │ PROFILE          │ Customer (name, email)       │
  │ CUSTOMER#cust-1     │ ORDER#2024-01-01 │ Customer order reference     │
  └─────────────────────┴──────────────────┴──────────────────────────────┘

  GSI1 (Global Secondary Index):
  ┌─────────────────────┬───────────────────────────┬──────────────────┐
  │ GSI1PK              │ GSI1SK                    │ Query Pattern    │
  ├─────────────────────┼───────────────────────────┼──────────────────┤
  │ CUSTOMER#cust-1     │ ORDER#2024-01-01#uuid-1   │ Orders by cust   │
  │ CUSTOMER#cust-1     │ ORDER#2024-01-02#uuid-2   │ (sorted by date) │
  └─────────────────────┴───────────────────────────┴──────────────────┘

  GSI2 (Global Secondary Index):
  ┌─────────────────────┬───────────────────────────┬──────────────────┐
  │ GSI2PK              │ GSI2SK                    │ Query Pattern    │
  ├─────────────────────┼───────────────────────────┼──────────────────┤
  │ STATUS#PENDING      │ ORDER#2024-01-01#uuid-1   │ Orders by status │
  │ STATUS#CONFIRMED    │ ORDER#2024-01-02#uuid-2   │ (sorted by date) │
  └─────────────────────┴───────────────────────────┴──────────────────┘

  Access Patterns:
  ────────────────
  1. Get order by ID         →  PK=ORDER#id, SK=METADATA
  2. Get order with items    →  PK=ORDER#id, SK begins_with("ITEM#")
  3. Get full order          →  PK=ORDER#id  (all SKs)
  4. Orders by customer      →  GSI1PK=CUSTOMER#id
  5. Orders by status        →  GSI2PK=STATUS#status
  6. Order events (history)  →  PK=ORDER#id, SK begins_with("EVENT#")
```

### Tarefa 3.4 — DynamoDB Advanced Queries (Go)

```go
// internal/repository/dynamo_order_repo.go
package repository

import (
	"context"
	"fmt"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/feature/dynamodb/attributevalue"
	"github.com/aws/aws-sdk-go-v2/feature/dynamodb/expression"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb/types"
)

type DynamoOrderRepo struct {
	client    *dynamodb.Client
	tableName string
}

func NewDynamoOrderRepo(client *dynamodb.Client, tableName string) *DynamoOrderRepo {
	return &DynamoOrderRepo{client: client, tableName: tableName}
}

// OrderWithItems represents a complete order with its items.
type OrderWithItems struct {
	Order Order
	Items []OrderItem
}

// GetOrderWithItems retrieves order metadata and all items in a single query.
func (r *DynamoOrderRepo) GetOrderWithItems(ctx context.Context, orderID string) (*OrderWithItems, error) {
	// Use Query to get all items for this order
	keyEx := expression.Key("PK").Equal(expression.Value(fmt.Sprintf("ORDER#%s", orderID)))

	expr, err := expression.NewBuilder().WithKeyCondition(keyEx).Build()
	if err != nil {
		return nil, fmt.Errorf("build expression: %w", err)
	}

	output, err := r.client.Query(ctx, &dynamodb.QueryInput{
		TableName:                 aws.String(r.tableName),
		KeyConditionExpression:    expr.KeyCondition(),
		ExpressionAttributeNames:  expr.Names(),
		ExpressionAttributeValues: expr.Values(),
	})
	if err != nil {
		return nil, fmt.Errorf("query order: %w", err)
	}

	result := &OrderWithItems{}
	for _, item := range output.Items {
		sk := item["SK"].(*types.AttributeValueMemberS).Value

		switch {
		case sk == "METADATA":
			if err := attributevalue.UnmarshalMap(item, &result.Order); err != nil {
				return nil, fmt.Errorf("unmarshal order: %w", err)
			}
		case len(sk) > 5 && sk[:5] == "ITEM#":
			var orderItem OrderItem
			if err := attributevalue.UnmarshalMap(item, &orderItem); err != nil {
				return nil, fmt.Errorf("unmarshal item: %w", err)
			}
			result.Items = append(result.Items, orderItem)
		}
	}

	if result.Order.ID == "" {
		return nil, fmt.Errorf("order not found: %s", orderID)
	}

	return result, nil
}

// ListOrdersByCustomer uses GSI1 to query orders by customer.
func (r *DynamoOrderRepo) ListOrdersByCustomer(ctx context.Context, customerID string, limit int32) ([]Order, error) {
	keyEx := expression.Key("GSI1PK").Equal(expression.Value(fmt.Sprintf("CUSTOMER#%s", customerID)))

	expr, err := expression.NewBuilder().WithKeyCondition(keyEx).Build()
	if err != nil {
		return nil, fmt.Errorf("build expression: %w", err)
	}

	output, err := r.client.Query(ctx, &dynamodb.QueryInput{
		TableName:                 aws.String(r.tableName),
		IndexName:                 aws.String("GSI1"),
		KeyConditionExpression:    expr.KeyCondition(),
		ExpressionAttributeNames:  expr.Names(),
		ExpressionAttributeValues: expr.Values(),
		ScanIndexForward:          aws.Bool(false), // Most recent first
		Limit:                     aws.Int32(limit),
	})
	if err != nil {
		return nil, fmt.Errorf("query by customer: %w", err)
	}

	orders := make([]Order, 0, len(output.Items))
	for _, item := range output.Items {
		var order Order
		if err := attributevalue.UnmarshalMap(item, &order); err != nil {
			return nil, fmt.Errorf("unmarshal order: %w", err)
		}
		orders = append(orders, order)
	}

	return orders, nil
}

// ListOrdersByStatus uses GSI2 to query orders by status.
func (r *DynamoOrderRepo) ListOrdersByStatus(ctx context.Context, status string, since time.Time) ([]Order, error) {
	keyEx := expression.Key("GSI2PK").Equal(expression.Value(fmt.Sprintf("STATUS#%s", status))).
		And(expression.Key("GSI2SK").GreaterThanEqual(expression.Value(fmt.Sprintf("ORDER#%s", since.Format(time.RFC3339)))))

	expr, err := expression.NewBuilder().WithKeyCondition(keyEx).Build()
	if err != nil {
		return nil, fmt.Errorf("build expression: %w", err)
	}

	output, err := r.client.Query(ctx, &dynamodb.QueryInput{
		TableName:                 aws.String(r.tableName),
		IndexName:                 aws.String("GSI2"),
		KeyConditionExpression:    expr.KeyCondition(),
		ExpressionAttributeNames:  expr.Names(),
		ExpressionAttributeValues: expr.Values(),
	})
	if err != nil {
		return nil, fmt.Errorf("query by status: %w", err)
	}

	orders := make([]Order, 0, len(output.Items))
	for _, item := range output.Items {
		var order Order
		if err := attributevalue.UnmarshalMap(item, &order); err != nil {
			return nil, fmt.Errorf("unmarshal order: %w", err)
		}
		orders = append(orders, order)
	}

	return orders, nil
}

// UpdateOrderStatus uses conditional update to change status.
func (r *DynamoOrderRepo) UpdateOrderStatus(ctx context.Context, orderID, fromStatus, toStatus string) error {
	update := expression.Set(
		expression.Name("status"), expression.Value(toStatus),
	).Set(
		expression.Name("updated_at"), expression.Value(time.Now().UTC().Format(time.RFC3339)),
	).Set(
		expression.Name("GSI2PK"), expression.Value(fmt.Sprintf("STATUS#%s", toStatus)),
	)

	condition := expression.Name("status").Equal(expression.Value(fromStatus))

	expr, err := expression.NewBuilder().
		WithUpdate(update).
		WithCondition(condition).
		Build()
	if err != nil {
		return fmt.Errorf("build expression: %w", err)
	}

	_, err = r.client.UpdateItem(ctx, &dynamodb.UpdateItemInput{
		TableName: aws.String(r.tableName),
		Key: map[string]types.AttributeValue{
			"PK": &types.AttributeValueMemberS{Value: fmt.Sprintf("ORDER#%s", orderID)},
			"SK": &types.AttributeValueMemberS{Value: "METADATA"},
		},
		UpdateExpression:          expr.Update(),
		ConditionExpression:       expr.Condition(),
		ExpressionAttributeNames:  expr.Names(),
		ExpressionAttributeValues: expr.Values(),
	})
	if err != nil {
		return fmt.Errorf("update status (concurrent conflict?): %w", err)
	}

	return nil
}

// TransactWriteOrderWithItems creates order + items in a single transaction.
func (r *DynamoOrderRepo) TransactWriteOrderWithItems(ctx context.Context, order *Order, items []OrderItem) error {
	orderItem, err := attributevalue.MarshalMap(order)
	if err != nil {
		return fmt.Errorf("marshal order: %w", err)
	}

	transactItems := []types.TransactWriteItem{
		{
			Put: &types.Put{
				TableName: aws.String(r.tableName),
				Item:      orderItem,
				ConditionExpression: aws.String("attribute_not_exists(PK)"),
			},
		},
	}

	for _, item := range items {
		itemMap, err := attributevalue.MarshalMap(item)
		if err != nil {
			return fmt.Errorf("marshal item: %w", err)
		}
		// Set composite keys for the item
		itemMap["PK"] = &types.AttributeValueMemberS{Value: order.PK}
		itemMap["SK"] = &types.AttributeValueMemberS{Value: fmt.Sprintf("ITEM#%s", item.ProductID)}

		transactItems = append(transactItems, types.TransactWriteItem{
			Put: &types.Put{
				TableName: aws.String(r.tableName),
				Item:      itemMap,
			},
		})
	}

	_, err = r.client.TransactWriteItems(ctx, &dynamodb.TransactWriteItemsInput{
		TransactItems: transactItems,
	})
	if err != nil {
		return fmt.Errorf("transact write: %w", err)
	}

	return nil
}
```

### Tarefa 3.5 — DynamoDB Advanced Operations (Java)

```java
// infrastructure/repository/DynamoOrderRepository.java
package com.orderflow.infrastructure.repository;

import com.orderflow.domain.entity.OrderEntity;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Repository;
import software.amazon.awssdk.enhanced.dynamodb.*;
import software.amazon.awssdk.enhanced.dynamodb.model.*;
import software.amazon.awssdk.services.dynamodb.DynamoDbClient;
import software.amazon.awssdk.services.dynamodb.model.*;

import java.time.Instant;
import java.util.*;

@Repository
public class DynamoOrderRepository {

    private final DynamoDbEnhancedClient enhancedClient;
    private final DynamoDbClient dynamoClient;
    private final DynamoDbTable<OrderEntity> table;
    private final DynamoDbIndex<OrderEntity> gsi1;
    private final DynamoDbIndex<OrderEntity> gsi2;

    public DynamoOrderRepository(
            DynamoDbEnhancedClient enhancedClient,
            DynamoDbClient dynamoClient,
            @Value("${orderflow.dynamodb.table-name}") String tableName) {
        this.enhancedClient = enhancedClient;
        this.dynamoClient = dynamoClient;
        this.table = enhancedClient.table(tableName, TableSchema.fromBean(OrderEntity.class));
        this.gsi1 = table.index("GSI1");
        this.gsi2 = table.index("GSI2");
    }

    /**
     * Query all items for an order using begins_with on SK.
     */
    public List<OrderEntity> getOrderWithItems(String orderId) {
        var queryConditional = QueryConditional.keyEqualTo(
                Key.builder().partitionValue("ORDER#" + orderId).build()
        );

        return table.query(QueryEnhancedRequest.builder()
                        .queryConditional(queryConditional)
                        .build())
                .items()
                .stream()
                .toList();
    }

    /**
     * List orders by customer using GSI1, most recent first.
     */
    public List<OrderEntity> listByCustomer(String customerId, int limit) {
        var queryConditional = QueryConditional.keyEqualTo(
                Key.builder().partitionValue("CUSTOMER#" + customerId).build()
        );

        return gsi1.query(QueryEnhancedRequest.builder()
                        .queryConditional(queryConditional)
                        .scanIndexForward(false)
                        .limit(limit)
                        .build())
                .stream()
                .flatMap(page -> page.items().stream())
                .toList();
    }

    /**
     * List orders by status using GSI2.
     */
    public List<OrderEntity> listByStatus(String status, Instant since) {
        var queryConditional = QueryConditional.sortGreaterThanOrEqualTo(
                Key.builder()
                        .partitionValue("STATUS#" + status)
                        .sortValue("ORDER#" + since.toString())
                        .build()
        );

        return gsi2.query(QueryEnhancedRequest.builder()
                        .queryConditional(queryConditional)
                        .build())
                .stream()
                .flatMap(page -> page.items().stream())
                .toList();
    }

    /**
     * Conditional update: change status only if current status matches.
     */
    public void updateStatus(String orderId, String fromStatus, String toStatus) {
        dynamoClient.updateItem(UpdateItemRequest.builder()
                .tableName(table.tableName())
                .key(Map.of(
                        "PK", AttributeValue.builder().s("ORDER#" + orderId).build(),
                        "SK", AttributeValue.builder().s("METADATA").build()
                ))
                .updateExpression("SET #status = :newStatus, #updated = :now, #gsi2pk = :gsi2pk")
                .conditionExpression("#status = :oldStatus")
                .expressionAttributeNames(Map.of(
                        "#status", "status",
                        "#updated", "updated_at",
                        "#gsi2pk", "GSI2PK"
                ))
                .expressionAttributeValues(Map.of(
                        ":newStatus", AttributeValue.builder().s(toStatus).build(),
                        ":oldStatus", AttributeValue.builder().s(fromStatus).build(),
                        ":now", AttributeValue.builder().s(Instant.now().toString()).build(),
                        ":gsi2pk", AttributeValue.builder().s("STATUS#" + toStatus).build()
                ))
                .build());
    }
}
```

### Tarefa 3.6 — ElastiCache (Redis) para Caching (Go)

```go
// internal/cache/order_cache.go
package cache

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"time"

	"github.com/redis/go-redis/v9"

	"orderflow/internal/repository"
)

type OrderCache struct {
	redis  *redis.Client
	repo   *repository.DynamoOrderRepo
	ttl    time.Duration
}

func NewOrderCache(redisClient *redis.Client, repo *repository.DynamoOrderRepo, ttl time.Duration) *OrderCache {
	return &OrderCache{redis: redisClient, repo: repo, ttl: ttl}
}

// GetOrder implements cache-aside pattern.
// 1. Check Redis
// 2. If miss, query DynamoDB
// 3. Cache result in Redis
func (c *OrderCache) GetOrder(ctx context.Context, orderID string) (*repository.Order, error) {
	key := fmt.Sprintf("order:%s", orderID)

	// 1. Try cache
	data, err := c.redis.Get(ctx, key).Bytes()
	if err == nil {
		var order repository.Order
		if err := json.Unmarshal(data, &order); err == nil {
			slog.Debug("Cache HIT", "order_id", orderID)
			return &order, nil
		}
	}

	slog.Debug("Cache MISS", "order_id", orderID)

	// 2. Query DynamoDB
	orderWithItems, err := c.repo.GetOrderWithItems(ctx, orderID)
	if err != nil {
		return nil, err
	}

	// 3. Cache result
	encoded, err := json.Marshal(orderWithItems.Order)
	if err == nil {
		c.redis.Set(ctx, key, encoded, c.ttl)
	}

	return &orderWithItems.Order, nil
}

// InvalidateOrder removes an order from cache after mutation.
func (c *OrderCache) InvalidateOrder(ctx context.Context, orderID string) error {
	key := fmt.Sprintf("order:%s", orderID)
	return c.redis.Del(ctx, key).Err()
}

// GetOrdersByCustomer with cache.
func (c *OrderCache) GetOrdersByCustomer(ctx context.Context, customerID string) ([]repository.Order, error) {
	key := fmt.Sprintf("customer-orders:%s", customerID)

	data, err := c.redis.Get(ctx, key).Bytes()
	if err == nil {
		var orders []repository.Order
		if err := json.Unmarshal(data, &orders); err == nil {
			return orders, nil
		}
	}

	orders, err := c.repo.ListOrdersByCustomer(ctx, customerID, 20)
	if err != nil {
		return nil, err
	}

	encoded, err := json.Marshal(orders)
	if err == nil {
		c.redis.Set(ctx, key, encoded, c.ttl)
	}

	return orders, nil
}
```

### Tarefa 3.7 — DynamoDB Streams Setup via CLI

```bash
#!/bin/bash
# scripts/setup-dynamodb-streams.sh — Enable DynamoDB Streams

set -euo pipefail

ENDPOINT="http://localhost:4566"
REGION="us-east-1"
AWS="aws --endpoint-url=$ENDPOINT --region=$REGION"

echo "=== Enabling DynamoDB Streams ==="
$AWS dynamodb update-table \
  --table-name orderflow-local-orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

STREAM_ARN=$($AWS dynamodb describe-table \
  --table-name orderflow-local-orders \
  --query 'Table.LatestStreamArn' --output text)

echo "Stream ARN: $STREAM_ARN"

echo "=== Creating Stream Processor Lambda ==="
# Lambda function that processes DynamoDB Stream events
# (See Level 4 for full Lambda implementation)

echo "=== DynamoDB Streams enabled ==="
echo "Events: INSERT, MODIFY, REMOVE with NEW_AND_OLD_IMAGES"
```

### Tarefa 3.8 — Receipt API com Presigned URLs (Go)

```go
// internal/handler/receipt_handler.go
package handler

import (
	"fmt"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"

	"orderflow/internal/repository"
)

type ReceiptHandler struct {
	store *repository.S3Store
}

func NewReceiptHandler(store *repository.S3Store) *ReceiptHandler {
	return &ReceiptHandler{store: store}
}

// UploadReceipt uploads a receipt for an order.
// POST /api/v1/orders/:id/receipts
func (h *ReceiptHandler) UploadReceipt(c *gin.Context) {
	orderID := c.Param("id")

	file, header, err := c.Request.FormFile("file")
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "file required"})
		return
	}
	defer file.Close()

	key := fmt.Sprintf("receipts/%s/%d-%s", orderID, time.Now().UnixMilli(), header.Filename)

	if err := h.store.MultipartUpload(c.Request.Context(), key, file, header.Header.Get("Content-Type")); err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusCreated, gin.H{
		"key":      key,
		"order_id": orderID,
		"filename": header.Filename,
		"size":     header.Size,
	})
}

// GetReceiptURL returns a presigned download URL.
// GET /api/v1/orders/:id/receipts/:key
func (h *ReceiptHandler) GetReceiptURL(c *gin.Context) {
	key := c.Param("key")

	url, err := h.store.GeneratePresignedURL(c.Request.Context(), key, 15*time.Minute)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusOK, gin.H{
		"url":        url,
		"expires_in": "15m",
	})
}

// GetUploadURL returns a presigned upload URL for client-side upload.
// POST /api/v1/orders/:id/receipts/upload-url
func (h *ReceiptHandler) GetUploadURL(c *gin.Context) {
	orderID := c.Param("id")

	var req struct {
		Filename    string `json:"filename" binding:"required"`
		ContentType string `json:"content_type" binding:"required"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	key := fmt.Sprintf("receipts/%s/%d-%s", orderID, time.Now().UnixMilli(), req.Filename)

	url, err := h.store.GeneratePresignedUploadURL(c.Request.Context(), key, req.ContentType, 15*time.Minute)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusOK, gin.H{
		"upload_url":   url,
		"key":          key,
		"expires_in":   "15m",
		"content_type": req.ContentType,
	})
}
```

---

## Database Decision Tree

```
Quando usar qual database?
═══════════════════════════

  Preciso de...
      │
      ├── Key-value access, <10ms latency?
      │     └── DynamoDB ✓
      │         (orders, sessions, events)
      │
      ├── Complex joins, aggregations, reports?
      │     └── RDS Aurora ✓
      │         (analytics, reconciliation)
      │
      ├── Caching, pub/sub, rate limiting?
      │     └── ElastiCache Redis ✓
      │         (hot data, sessions, leaderboards)
      │
      ├── File/object storage?
      │     └── S3 ✓
      │         (receipts, exports, backups)
      │
      └── Full-text search?
            └── OpenSearch ✓
                (order search, logs)

  DynamoDB vs RDS Aurora:
  ┌──────────────────────┬──────────────────────────────┐
  │  DynamoDB             │  RDS Aurora                  │
  ├──────────────────────┼──────────────────────────────┤
  │  Key-value / document │  Relational (SQL)            │
  │  Predictable latency  │  Complex queries             │
  │  Auto-scale infinito  │  Scale vertical + read       │
  │  Pay-per-request      │  Instance-based pricing      │
  │  Sem schema fixo      │  Schema rígido               │
  │  NoSQL patterns       │  ACID transactions           │
  │  Streams built-in     │  Change Data Capture         │
  └──────────────────────┴──────────────────────────────┘
```

---

## Anti-Patterns

| Anti-Pattern | Por que está errado | Como corrigir |
|---|---|---|
| Scan em DynamoDB | Full table scan, caro e lento | Usar Query com PK/SK design certo |
| GSI para cada query | Custo alto (write amplification) | Single-table design com GSIs planejados |
| Hot partition (PK = status) | Throttling em uma partição | Prefixar com shard (STATUS#PENDING#1) |
| S3 sem encryption | Dados em repouso não protegidos | SSE-S3 ou SSE-KMS em tudo |
| S3 sem versioning | Perda irreversível de dados | Habilitar versioning + lifecycle |
| Cache sem invalidação | Stale data | Invalidar cache em cada write |
| Redis sem TTL | Memória cresce indefinidamente | TTL em todas as chaves |
| Presigned URL com TTL longo | Risco de vazamento | Máximo 15 min para downloads |

---

## Critérios de Aceite

### S3
- [ ] Upload com SSE (server-side encryption AES256)
- [ ] Multipart upload para arquivos >5MB
- [ ] Presigned download URL com TTL de 15 min
- [ ] Presigned upload URL para client-side upload
- [ ] Lifecycle policy: Standard → IA (90d) → Glacier (365d)
- [ ] Event notification para SQS em criação de receipts

### DynamoDB
- [ ] Single-table design com PK/SK documentado
- [ ] GSI1 por customer, GSI2 por status
- [ ] Query por order ID (com items) — sem Scan
- [ ] Query por customer ID via GSI1
- [ ] Query por status via GSI2
- [ ] Conditional update de status (optimistic locking)
- [ ] TransactWriteItems para ordem + items atômico
- [ ] DynamoDB Streams habilitado (NEW_AND_OLD_IMAGES)

### ElastiCache
- [ ] Cache-aside pattern implementado (Go)
- [ ] Cache invalidation em mutations
- [ ] TTL configurado em todas as chaves

### API
- [ ] POST `/orders/:id/receipts` — upload com multipart
- [ ] GET `/orders/:id/receipts/:key` — presigned download URL
- [ ] POST `/orders/:id/receipts/upload-url` — presigned upload URL

---

## Definição de Pronto (DoD)

- [ ] S3 operations completas (Go + Java) com encryption + presigned URLs
- [ ] DynamoDB single-table design implementado e documentado
- [ ] ElastiCache (Redis) cache-aside para hot orders
- [ ] Receipt API funcional com presigned URLs
- [ ] Testes de integração com LocalStack
- [ ] Commit semântico: `feat(level-3): add S3 advanced ops, DynamoDB single-table, ElastiCache caching`

---

## Checklist

- [ ] S3: SSE, multipart, presigned URLs (download + upload)
- [ ] S3: lifecycle policy + event notifications
- [ ] DynamoDB: single-table design documentado
- [ ] DynamoDB: GSI1 (customer) + GSI2 (status)
- [ ] DynamoDB: conditional updates, TransactWriteItems
- [ ] DynamoDB Streams habilitado
- [ ] ElastiCache: cache-aside com invalidação
- [ ] Receipt API: upload, download URL, upload URL

---

## Exercícios Extras

1. **DynamoDB TTL** — Adicione um campo `ttl` nos eventos do order para auto-expiração após 90 dias. Configure TTL no DynamoDB.
2. **S3 Cross-Region Replication** — Configure CRR para o bucket de receipts (backup em outra região).
3. **DAX (DynamoDB Accelerator)** — Configure DAX como cache transparente para DynamoDB. Compare latência com e sem DAX via benchmarks.
4. **DynamoDB Batch Operations** — Implemente `BatchWriteItem` para import em massa de orders (até 25 items por batch, com exponential backoff para `UnprocessedItems`).
5. **S3 Select** — Use S3 Select para fazer queries SQL-like diretamente em arquivos JSON/CSV no S3 sem baixar o arquivo completo.
