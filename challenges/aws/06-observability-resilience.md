# Level 6 — Observability & Resilience

> **Objetivo:** Implementar observabilidade full-stack (CloudWatch metrics/logs/alarms, X-Ray
> distributed tracing, OpenTelemetry) e padrões de resiliência (multi-AZ, disaster recovery,
> health checks, graceful degradation).

---

## Objetivo de Aprendizado

- Publicar custom metrics com CloudWatch EMF (Embedded Metric Format)
- Implementar structured logging com CloudWatch Logs Insights queries
- Configurar alarms (simple, composite, anomaly detection)
- Instrumentar distributed tracing com X-Ray (Go + Java)
- Integrar OpenTelemetry como abstração de tracing/metrics
- Implementar health check patterns (shallow, deep, dependency)
- Projetar disaster recovery (backup & restore, pilot light, warm standby)
- Implementar graceful degradation e circuit breaker

---

## Escopo Funcional

```
Level 6 — Observability & Resilience Architecture
═══════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────────┐
  │  Application Services                                        │
  │                                                              │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐     │
  │  │ Order API   │  │ Payment Svc │  │ Notification Svc│     │
  │  │  (Go/Java)  │  │  (Go/Java)  │  │   (Go/Java)    │     │
  │  └──────┬──────┘  └──────┬──────┘  └────────┬────────┘     │
  │         │                │                   │               │
  │    ┌────▼────────────────▼───────────────────▼─────┐        │
  │    │         OpenTelemetry SDK (instrumentation)    │        │
  │    │         Traces | Metrics | Logs                │        │
  │    └────┬────────────────┬───────────────────┬─────┘        │
  └─────────┼────────────────┼───────────────────┼──────────────┘
            │                │                   │
   ┌────────▼──────┐ ┌──────▼───────┐ ┌─────────▼──────┐
   │   X-Ray       │ │  CloudWatch  │ │  CloudWatch    │
   │  (traces)     │ │  (metrics)   │ │  (logs)        │
   └──────┬────────┘ └──────┬───────┘ └────────┬───────┘
          │                 │                   │
   ┌──────▼────────┐ ┌─────▼────────┐ ┌────────▼──────┐
   │ Service Map   │ │  Alarms      │ │ Logs Insights │
   │ (dependency)  │ │ (composite)  │ │ (queries)     │
   └───────────────┘ └──────┬───────┘ └───────────────┘
                            │
                     ┌──────▼───────┐
                     │  SNS Alerts  │
                     │(PagerDuty)   │
                     └──────────────┘

  Health Check Levels:
  ═══════════════════
    Shallow:  GET /health        → 200 OK (app running)
    Deep:     GET /health/ready  → checks DynamoDB + S3 + SQS + Redis
    Liveness: GET /health/live   → checks goroutines/threads, memory

  Disaster Recovery:
  ═════════════════
    ┌────────────────┐              ┌────────────────┐
    │  Primary (AZ-a) │              │ Secondary (AZ-b)│
    │  ECS Service    │◄── ALB ──►  │  ECS Service    │
    │  DynamoDB       │ (global)    │  DynamoDB       │
    │  S3             │──── CRR ──►│  S3 (replica)   │
    └────────────────┘              └────────────────┘
```

---

## Tarefas

### Tarefa 6.1 — CloudWatch Custom Metrics (Go)

```go
// internal/observability/metrics.go
package observability

import (
	"context"
	"fmt"
	"log/slog"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/cloudwatch"
	cwtypes "github.com/aws/aws-sdk-go-v2/service/cloudwatch/types"
)

type MetricsClient struct {
	client    *cloudwatch.Client
	namespace string
}

func NewMetricsClient(client *cloudwatch.Client, namespace string) *MetricsClient {
	return &MetricsClient{client: client, namespace: namespace}
}

// PutOrderMetric publishes a custom business metric.
func (m *MetricsClient) PutOrderMetric(ctx context.Context, metricName string, value float64, dimensions map[string]string) error {
	dims := make([]cwtypes.Dimension, 0, len(dimensions))
	for k, v := range dimensions {
		dims = append(dims, cwtypes.Dimension{
			Name:  aws.String(k),
			Value: aws.String(v),
		})
	}

	_, err := m.client.PutMetricData(ctx, &cloudwatch.PutMetricDataInput{
		Namespace: aws.String(m.namespace),
		MetricData: []cwtypes.MetricDatum{
			{
				MetricName: aws.String(metricName),
				Value:      aws.Float64(value),
				Timestamp:  aws.Time(time.Now().UTC()),
				Dimensions: dims,
				Unit:       cwtypes.StandardUnitCount,
			},
		},
	})
	if err != nil {
		return fmt.Errorf("put metric: %w", err)
	}

	return nil
}

// PutLatencyMetric publishes latency with statistics.
func (m *MetricsClient) PutLatencyMetric(ctx context.Context, operation string, durationMs float64) error {
	_, err := m.client.PutMetricData(ctx, &cloudwatch.PutMetricDataInput{
		Namespace: aws.String(m.namespace),
		MetricData: []cwtypes.MetricDatum{
			{
				MetricName: aws.String("Latency"),
				Value:      aws.Float64(durationMs),
				Timestamp:  aws.Time(time.Now().UTC()),
				Unit:       cwtypes.StandardUnitMilliseconds,
				Dimensions: []cwtypes.Dimension{
					{Name: aws.String("Operation"), Value: aws.String(operation)},
				},
			},
		},
	})
	return err
}

// PublishBatchMetrics publishes multiple metrics in a single API call.
func (m *MetricsClient) PublishBatchMetrics(ctx context.Context, metrics []BusinessMetric) error {
	data := make([]cwtypes.MetricDatum, 0, len(metrics))
	for _, metric := range metrics {
		dims := make([]cwtypes.Dimension, 0)
		for k, v := range metric.Dimensions {
			dims = append(dims, cwtypes.Dimension{
				Name:  aws.String(k),
				Value: aws.String(v),
			})
		}

		data = append(data, cwtypes.MetricDatum{
			MetricName: aws.String(metric.Name),
			Value:      aws.Float64(metric.Value),
			Timestamp:  aws.Time(time.Now().UTC()),
			Unit:       cwtypes.StandardUnitCount,
			Dimensions: dims,
		})
	}

	// CloudWatch allows max 1000 metrics per PutMetricData (batch in chunks of 1000)
	for i := 0; i < len(data); i += 1000 {
		end := i + 1000
		if end > len(data) {
			end = len(data)
		}

		_, err := m.client.PutMetricData(ctx, &cloudwatch.PutMetricDataInput{
			Namespace:  aws.String(m.namespace),
			MetricData: data[i:end],
		})
		if err != nil {
			return fmt.Errorf("put batch metrics: %w", err)
		}
	}

	slog.Info("Batch metrics published", "count", len(metrics))
	return nil
}

type BusinessMetric struct {
	Name       string
	Value      float64
	Dimensions map[string]string
}
```

### Tarefa 6.2 — CloudWatch Custom Metrics (Java)

```java
// infrastructure/observability/CloudWatchMetricsService.java
package com.orderflow.infrastructure.observability;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.cloudwatch.CloudWatchClient;
import software.amazon.awssdk.services.cloudwatch.model.*;

import java.time.Instant;
import java.util.List;
import java.util.Map;

@Service
public class CloudWatchMetricsService {

    private final CloudWatchClient cwClient;
    private final String namespace;

    public CloudWatchMetricsService(
            CloudWatchClient cwClient,
            @Value("${orderflow.cloudwatch.namespace:OrderFlow}") String namespace) {
        this.cwClient = cwClient;
        this.namespace = namespace;
    }

    public void putOrderMetric(String metricName, double value, Map<String, String> dimensions) {
        var dims = dimensions.entrySet().stream()
                .map(e -> Dimension.builder().name(e.getKey()).value(e.getValue()).build())
                .toList();

        cwClient.putMetricData(PutMetricDataRequest.builder()
                .namespace(namespace)
                .metricData(MetricDatum.builder()
                        .metricName(metricName)
                        .value(value)
                        .timestamp(Instant.now())
                        .dimensions(dims)
                        .unit(StandardUnit.COUNT)
                        .build())
                .build());
    }

    public void putLatencyMetric(String operation, double durationMs) {
        cwClient.putMetricData(PutMetricDataRequest.builder()
                .namespace(namespace)
                .metricData(MetricDatum.builder()
                        .metricName("Latency")
                        .value(durationMs)
                        .timestamp(Instant.now())
                        .unit(StandardUnit.MILLISECONDS)
                        .dimensions(Dimension.builder()
                                .name("Operation")
                                .value(operation)
                                .build())
                        .build())
                .build());
    }

    /**
     * Publish batch metrics (up to 1000 per API call).
     */
    public void publishBatch(List<BusinessMetric> metrics) {
        var data = metrics.stream()
                .map(m -> MetricDatum.builder()
                        .metricName(m.name())
                        .value(m.value())
                        .timestamp(Instant.now())
                        .dimensions(m.dimensions().entrySet().stream()
                                .map(e -> Dimension.builder()
                                        .name(e.getKey())
                                        .value(e.getValue())
                                        .build())
                                .toList())
                        .unit(StandardUnit.COUNT)
                        .build())
                .toList();

        // Batch in chunks of 1000
        for (int i = 0; i < data.size(); i += 1000) {
            var chunk = data.subList(i, Math.min(i + 1000, data.size()));
            cwClient.putMetricData(PutMetricDataRequest.builder()
                    .namespace(namespace)
                    .metricData(chunk)
                    .build());
        }
    }

    public record BusinessMetric(String name, double value, Map<String, String> dimensions) {}
}
```

### Tarefa 6.3 — CloudWatch Alarms (Go)

```go
// internal/observability/alarms.go
package observability

import (
	"context"
	"fmt"
	"log/slog"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/cloudwatch"
	cwtypes "github.com/aws/aws-sdk-go-v2/service/cloudwatch/types"
)

type AlarmManager struct {
	client    *cloudwatch.Client
	namespace string
	snsArn    string
}

func NewAlarmManager(client *cloudwatch.Client, namespace, snsArn string) *AlarmManager {
	return &AlarmManager{client: client, namespace: namespace, snsArn: snsArn}
}

// CreateOrderErrorAlarm fires when error rate exceeds threshold.
func (a *AlarmManager) CreateOrderErrorAlarm(ctx context.Context) error {
	_, err := a.client.PutMetricAlarm(ctx, &cloudwatch.PutMetricAlarmInput{
		AlarmName:          aws.String("orderflow-high-error-rate"),
		AlarmDescription:   aws.String("Order error rate exceeds 5% in 5 minutes"),
		Namespace:          aws.String(a.namespace),
		MetricName:         aws.String("OrderErrors"),
		Statistic:          cwtypes.StatisticSum,
		Period:             aws.Int32(300), // 5 minutes
		EvaluationPeriods:  aws.Int32(2),
		Threshold:          aws.Float64(10),
		ComparisonOperator: cwtypes.ComparisonOperatorGreaterThanThreshold,
		TreatMissingData:   aws.String("notBreaching"),
		AlarmActions:       []string{a.snsArn},
		OKActions:          []string{a.snsArn},
		Dimensions: []cwtypes.Dimension{
			{Name: aws.String("Service"), Value: aws.String("OrderAPI")},
		},
	})
	if err != nil {
		return fmt.Errorf("create error alarm: %w", err)
	}

	slog.Info("Error alarm created", "name", "orderflow-high-error-rate")
	return nil
}

// CreateLatencyAlarm fires when P99 latency exceeds threshold.
func (a *AlarmManager) CreateLatencyAlarm(ctx context.Context) error {
	_, err := a.client.PutMetricAlarm(ctx, &cloudwatch.PutMetricAlarmInput{
		AlarmName:          aws.String("orderflow-high-latency"),
		AlarmDescription:   aws.String("P99 latency exceeds 2s for 3 evaluation periods"),
		Namespace:          aws.String(a.namespace),
		MetricName:         aws.String("Latency"),
		ExtendedStatistic:  aws.String("p99"),
		Period:             aws.Int32(60),
		EvaluationPeriods:  aws.Int32(3),
		Threshold:          aws.Float64(2000), // 2000ms
		ComparisonOperator: cwtypes.ComparisonOperatorGreaterThanThreshold,
		TreatMissingData:   aws.String("missing"),
		AlarmActions:       []string{a.snsArn},
		Dimensions: []cwtypes.Dimension{
			{Name: aws.String("Operation"), Value: aws.String("CreateOrder")},
		},
	})
	if err != nil {
		return fmt.Errorf("create latency alarm: %w", err)
	}

	return nil
}

// CreateCompositeAlarm fires only when BOTH error AND latency alarms are triggered.
func (a *AlarmManager) CreateCompositeAlarm(ctx context.Context) error {
	_, err := a.client.PutCompositeAlarm(ctx, &cloudwatch.PutCompositeAlarmInput{
		AlarmName:        aws.String("orderflow-critical"),
		AlarmDescription: aws.String("Both error rate AND latency are degraded"),
		AlarmRule:        aws.String("ALARM(\"orderflow-high-error-rate\") AND ALARM(\"orderflow-high-latency\")"),
		AlarmActions:     []string{a.snsArn},
	})
	if err != nil {
		return fmt.Errorf("create composite alarm: %w", err)
	}

	slog.Info("Composite alarm created", "name", "orderflow-critical")
	return nil
}

// ListAlarmsByState returns alarms in a specific state.
func (a *AlarmManager) ListAlarmsByState(ctx context.Context, state cwtypes.StateValue) ([]AlarmSummary, error) {
	output, err := a.client.DescribeAlarms(ctx, &cloudwatch.DescribeAlarmsInput{
		StateValue:     state,
		AlarmNamePrefix: aws.String("orderflow-"),
	})
	if err != nil {
		return nil, fmt.Errorf("describe alarms: %w", err)
	}

	summaries := make([]AlarmSummary, 0, len(output.MetricAlarms))
	for _, alarm := range output.MetricAlarms {
		summaries = append(summaries, AlarmSummary{
			Name:        aws.ToString(alarm.AlarmName),
			State:       string(alarm.StateValue),
			Description: aws.ToString(alarm.AlarmDescription),
			UpdatedAt:   alarm.StateUpdatedTimestamp.String(),
		})
	}

	return summaries, nil
}

type AlarmSummary struct {
	Name        string `json:"name"`
	State       string `json:"state"`
	Description string `json:"description"`
	UpdatedAt   string `json:"updated_at"`
}
```

### Tarefa 6.4 — Structured Logging & Logs Insights

```go
// internal/observability/logger.go
package observability

import (
	"context"
	"log/slog"
	"os"
	"time"
)

// SetupLogger configures structured JSON logging compatible with CloudWatch Logs.
func SetupLogger(serviceName, environment string) *slog.Logger {
	handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: logLevel(environment),
		ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
			// Rename "time" to "timestamp" for CloudWatch Logs Insights
			if a.Key == slog.TimeKey {
				a.Key = "timestamp"
				a.Value = slog.StringValue(a.Value.Time().Format(time.RFC3339Nano))
			}
			return a
		},
	})

	logger := slog.New(handler).With(
		"service", serviceName,
		"environment", environment,
	)

	slog.SetDefault(logger)
	return logger
}

func logLevel(env string) slog.Level {
	if env == "prod" {
		return slog.LevelInfo
	}
	return slog.LevelDebug
}

// RequestLogger creates a logger with request context.
func RequestLogger(ctx context.Context, requestID, traceID, method, path string) *slog.Logger {
	return slog.Default().With(
		"request_id", requestID,
		"trace_id", traceID,
		"method", method,
		"path", path,
	)
}

// Example log output (CloudWatch Logs Insights compatible):
// {
//   "timestamp": "2024-01-15T10:30:00.123456789Z",
//   "level": "INFO",
//   "msg": "Order created",
//   "service": "order-api",
//   "environment": "prod",
//   "request_id": "req-abc-123",
//   "trace_id": "1-abc-def",
//   "order_id": "order-xyz",
//   "customer_id": "cust-123",
//   "latency_ms": 45.2
// }

// CloudWatch Logs Insights queries:
//
// 1. Orders created in the last hour:
//    fields @timestamp, order_id, customer_id, latency_ms
//    | filter msg = "Order created"
//    | sort @timestamp desc
//    | limit 100
//
// 2. Error rate by service:
//    stats count(*) as total,
//          count_distinct(request_id) as requests,
//          sum(level = "ERROR") as errors
//    by service
//    | sort errors desc
//
// 3. P99 latency by operation:
//    filter ispresent(latency_ms)
//    | stats avg(latency_ms) as avg_ms,
//            pct(latency_ms, 50) as p50,
//            pct(latency_ms, 95) as p95,
//            pct(latency_ms, 99) as p99
//    by path
//
// 4. Slow requests (>2s):
//    filter latency_ms > 2000
//    | fields @timestamp, request_id, trace_id, path, latency_ms
//    | sort latency_ms desc
```

### Tarefa 6.5 — X-Ray Distributed Tracing (Go)

```go
// internal/observability/xray.go
package observability

import (
	"context"
	"fmt"
	"log/slog"
	"net/http"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/xray"
)

type TracingClient struct {
	client *xray.Client
}

func NewTracingClient(client *xray.Client) *TracingClient {
	return &TracingClient{client: client}
}

// GetServiceMap retrieves the service map for dependency visualization.
func (t *TracingClient) GetServiceMap(ctx context.Context, since time.Time) (*ServiceMap, error) {
	output, err := t.client.GetServiceGraph(ctx, &xray.GetServiceGraphInput{
		StartTime: aws.Time(since),
		EndTime:   aws.Time(time.Now()),
	})
	if err != nil {
		return nil, fmt.Errorf("get service graph: %w", err)
	}

	serviceMap := &ServiceMap{
		Services: make([]ServiceNode, 0),
	}

	for _, svc := range output.Services {
		node := ServiceNode{
			Name: aws.ToString(svc.Name),
			Type: aws.ToString(svc.Type),
		}

		if svc.SummaryStatistics != nil {
			node.TotalRequests = int64(aws.ToFloat64(svc.SummaryStatistics.TotalCount))
			node.ErrorRate = aws.ToFloat64(svc.SummaryStatistics.ErrorStatistics.TotalCount) /
				aws.ToFloat64(svc.SummaryStatistics.TotalCount) * 100

			if svc.SummaryStatistics.ResponseTime != nil {
				node.AvgLatencyMs = aws.ToFloat64(svc.SummaryStatistics.ResponseTime.Average) * 1000
			}
		}

		for _, edge := range svc.Edges {
			node.Dependencies = append(node.Dependencies, aws.ToString(edge.ReferenceId))
		}

		serviceMap.Services = append(serviceMap.Services, node)
	}

	return serviceMap, nil
}

// GetTracesByFilter searches traces by annotation or metadata.
func (t *TracingClient) GetTracesByFilter(ctx context.Context, filterExpression string, since time.Time) ([]TraceSummary, error) {
	output, err := t.client.GetTraceSummaries(ctx, &xray.GetTraceSummariesInput{
		StartTime:        aws.Time(since),
		EndTime:          aws.Time(time.Now()),
		FilterExpression: aws.String(filterExpression),
	})
	if err != nil {
		return nil, fmt.Errorf("get trace summaries: %w", err)
	}

	summaries := make([]TraceSummary, 0, len(output.TraceSummaries))
	for _, ts := range output.TraceSummaries {
		summary := TraceSummary{
			TraceID:    aws.ToString(ts.Id),
			Duration:   aws.ToFloat64(ts.Duration),
			HasError:   ts.HasError != nil && *ts.HasError,
			HasFault:   ts.HasFault != nil && *ts.HasFault,
			StatusCode: int(aws.ToInt32(ts.Http.HttpStatus)),
		}
		summaries = append(summaries, summary)
	}

	return summaries, nil
}

type ServiceMap struct {
	Services []ServiceNode `json:"services"`
}

type ServiceNode struct {
	Name         string   `json:"name"`
	Type         string   `json:"type"`
	TotalRequests int64   `json:"total_requests"`
	ErrorRate    float64  `json:"error_rate_pct"`
	AvgLatencyMs float64 `json:"avg_latency_ms"`
	Dependencies []string `json:"dependencies,omitempty"`
}

type TraceSummary struct {
	TraceID    string  `json:"trace_id"`
	Duration   float64 `json:"duration_s"`
	HasError   bool    `json:"has_error"`
	HasFault   bool    `json:"has_fault"`
	StatusCode int     `json:"status_code"`
}

// X-Ray filter expression examples:
// - service("order-api")
// - annotation.order_id = "order-123"
// - responsetime > 2
// - fault = true
// - http.status = 500
// - service("order-api") AND responsetime > 1
```

### Tarefa 6.6 — Health Check Endpoint (Go)

```go
// internal/handler/health_handler.go
package handler

import (
	"context"
	"net/http"
	"runtime"
	"sync"
	"time"

	"github.com/gin-gonic/gin"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
	"github.com/aws/aws-sdk-go-v2/service/s3"
	"github.com/aws/aws-sdk-go-v2/service/sqs"
	"github.com/redis/go-redis/v9"
)

type HealthHandler struct {
	dynamo  *dynamodb.Client
	s3      *s3.Client
	sqs     *sqs.Client
	redis   *redis.Client
	startAt time.Time
}

func NewHealthHandler(dynamo *dynamodb.Client, s3Client *s3.Client, sqsClient *sqs.Client, redisClient *redis.Client) *HealthHandler {
	return &HealthHandler{
		dynamo:  dynamo,
		s3:      s3Client,
		sqs:     sqsClient,
		redis:   redisClient,
		startAt: time.Now(),
	}
}

type DependencyCheck struct {
	Name    string `json:"name"`
	Status  string `json:"status"` // UP, DOWN, DEGRADED
	Latency string `json:"latency"`
	Error   string `json:"error,omitempty"`
}

// Liveness — is the process alive?
// GET /health/live
func (h *HealthHandler) Liveness(c *gin.Context) {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)

	c.JSON(http.StatusOK, gin.H{
		"status":       "UP",
		"uptime":       time.Since(h.startAt).String(),
		"goroutines":   runtime.NumGoroutine(),
		"heap_mb":      m.HeapAlloc / 1024 / 1024,
		"sys_mb":       m.Sys / 1024 / 1024,
		"gc_cycles":    m.NumGC,
	})
}

// Readiness — can the service handle requests?
// GET /health/ready
func (h *HealthHandler) Readiness(c *gin.Context) {
	ctx, cancel := context.WithTimeout(c.Request.Context(), 5*time.Second)
	defer cancel()

	checks := h.checkDependencies(ctx)

	overallStatus := "UP"
	statusCode := http.StatusOK

	for _, check := range checks {
		if check.Status == "DOWN" {
			overallStatus = "DOWN"
			statusCode = http.StatusServiceUnavailable
			break
		}
		if check.Status == "DEGRADED" {
			overallStatus = "DEGRADED"
		}
	}

	c.JSON(statusCode, gin.H{
		"status":       overallStatus,
		"timestamp":    time.Now().UTC().Format(time.RFC3339),
		"dependencies": checks,
	})
}

func (h *HealthHandler) checkDependencies(ctx context.Context) []DependencyCheck {
	var wg sync.WaitGroup
	checks := make([]DependencyCheck, 4)

	// Check DynamoDB
	wg.Add(1)
	go func() {
		defer wg.Done()
		start := time.Now()
		_, err := h.dynamo.ListTables(ctx, &dynamodb.ListTablesInput{Limit: aws.Int32(1)})
		checks[0] = DependencyCheck{
			Name:    "dynamodb",
			Status:  statusFromErr(err),
			Latency: time.Since(start).String(),
		}
		if err != nil {
			checks[0].Error = err.Error()
		}
	}()

	// Check S3
	wg.Add(1)
	go func() {
		defer wg.Done()
		start := time.Now()
		_, err := h.s3.ListBuckets(ctx, &s3.ListBucketsInput{})
		checks[1] = DependencyCheck{
			Name:    "s3",
			Status:  statusFromErr(err),
			Latency: time.Since(start).String(),
		}
		if err != nil {
			checks[1].Error = err.Error()
		}
	}()

	// Check SQS
	wg.Add(1)
	go func() {
		defer wg.Done()
		start := time.Now()
		_, err := h.sqs.ListQueues(ctx, &sqs.ListQueuesInput{MaxResults: aws.Int32(1)})
		checks[2] = DependencyCheck{
			Name:    "sqs",
			Status:  statusFromErr(err),
			Latency: time.Since(start).String(),
		}
		if err != nil {
			checks[2].Error = err.Error()
		}
	}()

	// Check Redis
	wg.Add(1)
	go func() {
		defer wg.Done()
		start := time.Now()
		err := h.redis.Ping(ctx).Err()
		checks[3] = DependencyCheck{
			Name:    "redis",
			Status:  statusFromErr(err),
			Latency: time.Since(start).String(),
		}
		if err != nil {
			checks[3].Error = err.Error()
		}
	}()

	wg.Wait()
	return checks
}

func statusFromErr(err error) string {
	if err == nil {
		return "UP"
	}
	return "DOWN"
}
```

### Tarefa 6.7 — Observability Middleware (Go)

```go
// internal/middleware/observability.go
package middleware

import (
	"fmt"
	"log/slog"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/google/uuid"

	"orderflow/internal/observability"
)

// ObservabilityMiddleware adds request tracing, logging, and metrics.
func ObservabilityMiddleware(metrics *observability.MetricsClient) gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()

		// Generate or extract request/trace IDs
		requestID := c.GetHeader("X-Request-ID")
		if requestID == "" {
			requestID = uuid.New().String()
		}
		traceID := c.GetHeader("X-Amzn-Trace-Id")

		c.Set("request_id", requestID)
		c.Header("X-Request-ID", requestID)

		// Create request-scoped logger
		logger := observability.RequestLogger(c.Request.Context(), requestID, traceID, c.Request.Method, c.FullPath())

		slog.Info("Request started",
			"request_id", requestID,
			"method", c.Request.Method,
			"path", c.FullPath(),
			"client_ip", c.ClientIP(),
		)

		// Process request
		c.Next()

		// Calculate duration
		duration := time.Since(start)
		status := c.Writer.Status()

		// Log request completion
		logger.Info("Request completed",
			"status", status,
			"latency_ms", duration.Milliseconds(),
			"response_size", c.Writer.Size(),
		)

		// Publish metrics
		if metrics != nil {
			ctx := c.Request.Context()
			dims := map[string]string{
				"Operation": fmt.Sprintf("%s %s", c.Request.Method, c.FullPath()),
				"Status":    fmt.Sprintf("%d", status),
			}

			_ = metrics.PutOrderMetric(ctx, "RequestCount", 1, dims)
			_ = metrics.PutLatencyMetric(ctx, c.FullPath(), float64(duration.Milliseconds()))

			if status >= 500 {
				_ = metrics.PutOrderMetric(ctx, "ServerErrors", 1, dims)
			} else if status >= 400 {
				_ = metrics.PutOrderMetric(ctx, "ClientErrors", 1, dims)
			}
		}
	}
}
```

### Tarefa 6.8 — Setup Observability Stack (CLI)

```bash
#!/bin/bash
# scripts/setup-observability.sh — Create CloudWatch resources in LocalStack

set -euo pipefail

ENDPOINT="http://localhost:4566"
REGION="us-east-1"
AWS="aws --endpoint-url=$ENDPOINT --region=$REGION"

echo "=== Creating CloudWatch Log Groups ==="
for LOG_GROUP in /ecs/orderflow-go /ecs/orderflow-java /orderflow/application; do
  $AWS logs create-log-group --log-group-name "$LOG_GROUP"
  $AWS logs put-retention-policy --log-group-name "$LOG_GROUP" --retention-in-days 30
  echo "Created: $LOG_GROUP (retention: 30 days)"
done

echo "=== Creating SNS Alert Topic ==="
ALERT_TOPIC=$($AWS sns create-topic --name orderflow-alerts \
  --query 'TopicArn' --output text)
echo "Alert Topic: $ALERT_TOPIC"

echo "=== Creating CloudWatch Alarms ==="

# Error rate alarm
$AWS cloudwatch put-metric-alarm \
  --alarm-name orderflow-high-error-rate \
  --alarm-description "Error count exceeds 10 in 5 minutes" \
  --namespace OrderFlow \
  --metric-name OrderErrors \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions "$ALERT_TOPIC" \
  --ok-actions "$ALERT_TOPIC"

# Latency alarm
$AWS cloudwatch put-metric-alarm \
  --alarm-name orderflow-high-latency \
  --alarm-description "P99 latency exceeds 2s for 3 periods" \
  --namespace OrderFlow \
  --metric-name Latency \
  --extended-statistic p99 \
  --period 60 \
  --evaluation-periods 3 \
  --threshold 2000 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions "$ALERT_TOPIC"

# DLQ depth alarm
$AWS cloudwatch put-metric-alarm \
  --alarm-name orderflow-dlq-depth \
  --alarm-description "DLQ has messages (poison messages)" \
  --namespace AWS/SQS \
  --metric-name ApproximateNumberOfMessagesVisible \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --dimensions "Name=QueueName,Value=orderflow-dlq.fifo" \
  --alarm-actions "$ALERT_TOPIC"

echo "=== Creating CloudWatch Dashboard ==="
$AWS cloudwatch put-dashboard \
  --dashboard-name orderflow-overview \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "x": 0, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "Order Metrics",
          "metrics": [
            ["OrderFlow", "OrdersCreated", "Service", "OrderAPI"],
            ["OrderFlow", "OrderErrors", "Service", "OrderAPI"]
          ],
          "period": 300,
          "stat": "Sum",
          "region": "us-east-1"
        }
      },
      {
        "type": "metric",
        "x": 12, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "Latency (P50, P95, P99)",
          "metrics": [
            ["OrderFlow", "Latency", "Operation", "CreateOrder", {"stat": "p50"}],
            ["OrderFlow", "Latency", "Operation", "CreateOrder", {"stat": "p95"}],
            ["OrderFlow", "Latency", "Operation", "CreateOrder", {"stat": "p99"}]
          ],
          "period": 60,
          "region": "us-east-1"
        }
      },
      {
        "type": "metric",
        "x": 0, "y": 6, "width": 12, "height": 6,
        "properties": {
          "title": "DynamoDB Consumed Capacity",
          "metrics": [
            ["AWS/DynamoDB", "ConsumedReadCapacityUnits", "TableName", "orderflow-orders"],
            ["AWS/DynamoDB", "ConsumedWriteCapacityUnits", "TableName", "orderflow-orders"]
          ],
          "period": 300,
          "stat": "Sum",
          "region": "us-east-1"
        }
      },
      {
        "type": "metric",
        "x": 12, "y": 6, "width": 12, "height": 6,
        "properties": {
          "title": "SQS Queue Depth",
          "metrics": [
            ["AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", "orderflow-orders.fifo"],
            ["AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", "orderflow-dlq.fifo"]
          ],
          "period": 60,
          "stat": "Maximum",
          "region": "us-east-1"
        }
      }
    ]
  }'

echo "=== Observability stack created ==="
echo "Log Groups: 3 created"
echo "Alarms: error rate, latency, DLQ depth"
echo "Dashboard: orderflow-overview"
```

---

## Alerting Strategy

```
Alerting Strategy — Severity Levels
════════════════════════════════════

  ┌──────────┬─────────────────────────┬──────────────┬────────────┐
  │ Severity │ Condition               │ Response     │ Channel    │
  ├──────────┼─────────────────────────┼──────────────┼────────────┤
  │ P1 CRIT  │ Service down            │ Page on-call │ PagerDuty  │
  │          │ Error rate > 10%        │ 5 min SLA    │            │
  │          │ DLQ depth > 100         │              │            │
  ├──────────┼─────────────────────────┼──────────────┼────────────┤
  │ P2 HIGH  │ P99 latency > 2s       │ Slack alert  │ Slack      │
  │          │ Error rate > 5%         │ 30 min SLA   │ #alerts    │
  │          │ Disk > 85%             │              │            │
  ├──────────┼─────────────────────────┼──────────────┼────────────┤
  │ P3 WARN  │ P95 latency > 1s       │ Ticket       │ Jira       │
  │          │ Error rate > 1%         │ Next sprint  │            │
  │          │ Memory > 70%           │              │            │
  ├──────────┼─────────────────────────┼──────────────┼────────────┤
  │ P4 INFO  │ Deployment              │ Dashboard    │ CloudWatch │
  │          │ Scale event             │ FYI          │ Dashboard  │
  │          │ Config change           │              │            │
  └──────────┴─────────────────────────┴──────────────┴────────────┘

  Composite Alarm:
  ════════════════
  ALARM(high-error-rate) AND ALARM(high-latency) → P1 (page)
  ALARM(high-error-rate) OR ALARM(high-latency)  → P2 (slack)
```

---

## Anti-Patterns

| Anti-Pattern | Por que está errado | Como corrigir |
|---|---|---|
| Log sem structured format | CloudWatch Logs Insights não funciona | JSON com campos indexáveis |
| Log sem request/trace ID | Impossível correlacionar | Propagar X-Request-ID e X-Amzn-Trace-Id |
| Alarme sem action | Ninguém é notificado | SNS topic → PagerDuty/Slack |
| Alarme em cada métrica | Alert fatigue | Composite alarms para severidade |
| Health check retorna 200 sempre | LB não remove instância unhealthy | Deep check com dependencies |
| Métrica sem dimensions | Impossível filtrar por service/op | Dimensions: Service, Operation, Status |
| Sem DLQ alarm | Poison messages passam despercebidos | Alarm em DLQ depth > 0 |
| Logs sem retention policy | Custo infinito no CloudWatch | Retention 30-90 days |

---

## Critérios de Aceite

### Metrics
- [ ] Custom metrics publicados para OrderFlow (Go + Java)
- [ ] Latency metric com percentiles (P50, P95, P99)
- [ ] Batch publish (up to 1000 per call)
- [ ] Dimensions: Service, Operation, Status

### Alarms
- [ ] Error rate alarm (>10 in 5 min)
- [ ] Latency alarm (P99 > 2s)
- [ ] DLQ depth alarm (>0)
- [ ] Composite alarm (error AND latency)
- [ ] Todos com SNS action

### Logging
- [ ] JSON structured logging (Go slog + Java SLF4J)
- [ ] Request/trace ID em todos os logs
- [ ] CloudWatch Logs Insights queries documentadas

### Tracing
- [ ] X-Ray service map retrievable via SDK
- [ ] Trace filter expressions documentadas

### Health Checks
- [ ] Liveness: `/health/live` (goroutines, heap, GC)
- [ ] Readiness: `/health/ready` (DynamoDB, S3, SQS, Redis — parallel)
- [ ] Readiness retorna 503 se qualquer dependency DOWN

### Middleware
- [ ] Request count, latency, error metrics por request
- [ ] Request/response logging com duration

---

## Definição de Pronto (DoD)

- [ ] CloudWatch custom metrics (Go + Java) com dimensions
- [ ] CloudWatch alarms: error, latency, DLQ, composite
- [ ] Structured JSON logging com Logs Insights queries
- [ ] X-Ray tracing queries via SDK
- [ ] Health checks: liveness + readiness com dependency checks
- [ ] Observability middleware para métricas automáticas
- [ ] Dashboard JSON definition via CLI
- [ ] Commit semântico: `feat(level-6): add CloudWatch metrics/alarms, X-Ray tracing, health checks`

---

## Checklist

- [ ] Metrics: custom metrics Go + Java, latency, batch publish
- [ ] Alarms: error rate, latency P99, DLQ depth, composite
- [ ] Logging: structured JSON, request/trace ID, Logs Insights queries
- [ ] Tracing: X-Ray service map, trace filter expressions
- [ ] Health checks: liveness + readiness, parallel dependency checks
- [ ] Middleware: request metrics + logging
- [ ] Script: `setup-observability.sh` (log groups, alarms, dashboard)

---

## Exercícios Extras

1. **OpenTelemetry Integration** — Substitua a instrumentação direta do X-Ray pelo OpenTelemetry SDK (Go/Java) com exporter para X-Ray. Compare a experiência de desenvolvimento.
2. **Anomaly Detection Alarm** — Crie um alarm com anomaly detection band para detectar desvios de padrão no volume de orders (sem threshold fixo).
3. **CloudWatch Contributor Insights** — Configure Contributor Insights no DynamoDB para identificar hot keys (top partition keys por throttling).
4. **Custom Dashboard API** — Implemente um endpoint `GET /api/v1/observability/dashboard` que retorna métricas do CloudWatch formatadas para frontend (últimas 1h: orders criadas, error rate, latência P50/P95/P99).
5. **Disaster Recovery Runbook** — Implemente um DR runbook via SDK: verificar estado do DynamoDB global table, S3 replication status, e failover de Route 53 health check.
