# AWS Observability — Boas Práticas

Guia completo de observabilidade na AWS: os três pilares (Métricas, Logs, Traces) e como implementá-los.

---

## Três Pilares da Observabilidade

```
┌─────────────────────────────────────────────────────────────────┐
│                    Observability Pillars                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MÉTRICAS (CloudWatch Metrics)                                  │
│  "O quê está acontecendo?"                                     │
│  • CPU, Memory, Latency, Error Rate, Throughput                │
│  • Alarmes e dashboards                                         │
│  • Tendências e anomalias                                       │
│                                                                 │
│  LOGS (CloudWatch Logs)                                         │
│  "Por quê está acontecendo?"                                   │
│  • Application logs estruturados (JSON)                         │
│  • Access logs, Audit logs                                      │
│  • CloudTrail para API calls                                    │
│                                                                 │
│  TRACES (X-Ray / OpenTelemetry)                                │
│  "Onde está acontecendo?"                                      │
│  • Distributed tracing entre serviços                          │
│  • Latency breakdown por componente                            │
│  • Error propagation tracking                                   │
│                                                                 │
│  ═══════════════════════════════════════                        │
│  Correlação: Métricas → Logs → Traces                          │
│  Use Request ID / Trace ID para navegar entre os 3 pilares     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Métricas — CloudWatch

### Custom Metrics com EMF (Embedded Metric Format)

```java
// AWS Lambda Powertools — métricas automáticas
@Metrics(namespace = "MyApp", service = "OrderService")
@Tracing
@Logging
public class OrderHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    private final MetricsLogger metricsLogger = MetricsUtils.metricsLogger();

    @Override
    public APIGatewayProxyResponseEvent handleRequest(
            APIGatewayProxyRequestEvent event, Context context) {
        
        // Métricas de negócio
        metricsLogger.putMetric("OrderCreated", 1, Unit.COUNT);
        metricsLogger.putMetric("OrderAmount", order.getAmount(), Unit.NONE);
        
        // Dimensões
        metricsLogger.putDimensions(
            DimensionSet.of("Environment", System.getenv("ENVIRONMENT")),
            DimensionSet.of("Region", System.getenv("AWS_REGION"))
        );

        // ...
    }
}
```

### Métricas Essenciais por Serviço

```hcl
# Dashboard consolidado
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "${var.project}-${var.environment}"

  dashboard_body = jsonencode({
    widgets = [
      # --- ROW 1: Overview ---
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 8
        height = 6
        properties = {
          title   = "API Latency (p50, p90, p99)"
          metrics = [
            ["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", var.alb_arn_suffix, { stat = "p50", label = "p50" }],
            ["...", { stat = "p90", label = "p90" }],
            ["...", { stat = "p99", label = "p99" }]
          ]
          period = 60
          region = var.region
          yAxis  = { left = { min = 0, label = "seconds" } }
        }
      },
      {
        type   = "metric"
        x      = 8
        y      = 0
        width  = 8
        height = 6
        properties = {
          title   = "Request Count & Errors"
          metrics = [
            ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", var.alb_arn_suffix, { stat = "Sum", label = "Requests" }],
            ["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count", "LoadBalancer", var.alb_arn_suffix, { stat = "Sum", label = "5XX", color = "#d62728" }],
            ["AWS/ApplicationELB", "HTTPCode_Target_4XX_Count", "LoadBalancer", var.alb_arn_suffix, { stat = "Sum", label = "4XX", color = "#ff7f0e" }]
          ]
          period = 60
          region = var.region
        }
      },
      {
        type   = "metric"
        x      = 16
        y      = 0
        width  = 8
        height = 6
        properties = {
          title   = "Error Rate %"
          metrics = [
            [{
              expression = "(m2 / m1) * 100"
              label      = "Error Rate %"
              id         = "error_rate"
            }],
            ["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count", "LoadBalancer", var.alb_arn_suffix, { stat = "Sum", id = "m2", visible = false }],
            ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", var.alb_arn_suffix, { stat = "Sum", id = "m1", visible = false }]
          ]
          period = 60
          region = var.region
          yAxis  = { left = { min = 0, max = 100 } }
        }
      },

      # --- ROW 2: Compute ---
      {
        type   = "metric"
        x      = 0
        y      = 6
        width  = 12
        height = 6
        properties = {
          title   = "ECS - CPU & Memory"
          metrics = [
            ["AWS/ECS", "CPUUtilization", "ClusterName", var.cluster_name, "ServiceName", var.service_name, { stat = "Average" }],
            ["AWS/ECS", "MemoryUtilization", "ClusterName", var.cluster_name, "ServiceName", var.service_name, { stat = "Average" }]
          ]
          period = 60
          region = var.region
          yAxis  = { left = { min = 0, max = 100 } }
        }
      },

      # --- ROW 3: Database ---
      {
        type   = "metric"
        x      = 0
        y      = 12
        width  = 8
        height = 6
        properties = {
          title   = "RDS - CPU & Connections"
          metrics = [
            ["AWS/RDS", "CPUUtilization", "DBInstanceIdentifier", var.db_identifier, { stat = "Average" }],
            ["AWS/RDS", "DatabaseConnections", "DBInstanceIdentifier", var.db_identifier, { stat = "Average", yAxis = "right" }]
          ]
          period = 60
          region = var.region
        }
      },
      {
        type   = "metric"
        x      = 8
        y      = 12
        width  = 8
        height = 6
        properties = {
          title   = "DynamoDB - Read/Write Capacity"
          metrics = [
            ["AWS/DynamoDB", "ConsumedReadCapacityUnits", "TableName", var.dynamodb_table, { stat = "Sum" }],
            ["AWS/DynamoDB", "ConsumedWriteCapacityUnits", "TableName", var.dynamodb_table, { stat = "Sum" }]
          ]
          period = 60
          region = var.region
        }
      },
      {
        type   = "metric"
        x      = 16
        y      = 12
        width  = 8
        height = 6
        properties = {
          title   = "Redis - Memory & Cache Hit Rate"
          metrics = [
            ["AWS/ElastiCache", "DatabaseMemoryUsagePercentage", "CacheClusterId", var.redis_cluster_id],
            ["AWS/ElastiCache", "CacheHitRate", "CacheClusterId", var.redis_cluster_id, { yAxis = "right" }]
          ]
          period = 300
          region = var.region
        }
      }
    ]
  })
}
```

### Alarmes — SLI/SLO Based

```hcl
# SLO: 99.9% availability (43.8 min downtime/month)
# SLI: Error Rate < 0.1%

resource "aws_cloudwatch_metric_alarm" "slo_availability" {
  alarm_name          = "${var.project}-slo-availability"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 5
  threshold           = 0.1  # 0.1% error rate

  metric_query {
    id          = "error_rate"
    expression  = "(m2 / m1) * 100"
    label       = "Error Rate %"
    return_data = true
  }

  metric_query {
    id = "m2"
    metric {
      metric_name = "HTTPCode_Target_5XX_Count"
      namespace   = "AWS/ApplicationELB"
      period      = 60
      stat        = "Sum"
      dimensions  = { LoadBalancer = var.alb_arn_suffix }
    }
  }

  metric_query {
    id = "m1"
    metric {
      metric_name = "RequestCount"
      namespace   = "AWS/ApplicationELB"
      period      = 60
      stat        = "Sum"
      dimensions  = { LoadBalancer = var.alb_arn_suffix }
    }
  }

  alarm_actions = [aws_sns_topic.pager.arn]
  ok_actions    = [aws_sns_topic.pager.arn]
}

# SLO: P99 latency < 500ms
resource "aws_cloudwatch_metric_alarm" "slo_latency" {
  alarm_name          = "${var.project}-slo-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "TargetResponseTime"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  extended_statistic  = "p99"
  threshold           = 0.5  # 500ms
  alarm_actions       = [aws_sns_topic.pager.arn]

  dimensions = {
    LoadBalancer = var.alb_arn_suffix
  }
}
```

---

## Logs — Structured Logging

### Formato de Log Padrão (JSON)

```json
{
  "timestamp": "2026-02-24T10:30:00.000Z",
  "level": "INFO",
  "service": "order-service",
  "environment": "prod",
  "traceId": "1-abc123-def456",
  "spanId": "span-789",
  "requestId": "req-uuid-123",
  "correlationId": "corr-uuid-456",
  "message": "Order created successfully",
  "orderId": "ORD-2024-001",
  "customerId": "CUST-123",
  "amount": 150.00,
  "duration_ms": 45,
  "httpMethod": "POST",
  "httpPath": "/api/orders",
  "httpStatus": 201
}
```

### Log Configuration (Java — Logback)

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>
                {"service":"${SERVICE_NAME:-unknown}","environment":"${ENVIRONMENT:-unknown}"}
            </customFields>
            <fieldNames>
                <timestamp>timestamp</timestamp>
                <version>[ignore]</version>
                <levelValue>[ignore]</levelValue>
            </fieldNames>
        </encoder>
    </appender>

    <!-- Incluir MDC fields automaticamente -->
    <root level="${LOG_LEVEL:-INFO}">
        <appender-ref ref="CONSOLE" />
    </root>
</configuration>
```

```java
// MDC Filter — adicionar traceId e requestId em todas as mensagens
@Component
public class ObservabilityFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) throws Exception {
        
        String traceId = Optional.ofNullable(request.getHeader("X-Amzn-Trace-Id"))
                .orElse(UUID.randomUUID().toString());
        String requestId = Optional.ofNullable(request.getHeader("X-Request-Id"))
                .orElse(UUID.randomUUID().toString());

        MDC.put("traceId", traceId);
        MDC.put("requestId", requestId);
        MDC.put("httpMethod", request.getMethod());
        MDC.put("httpPath", request.getRequestURI());

        try {
            long start = System.currentTimeMillis();
            chain.doFilter(request, response);
            long duration = System.currentTimeMillis() - start;
            
            MDC.put("httpStatus", String.valueOf(response.getStatus()));
            MDC.put("duration_ms", String.valueOf(duration));
            
            log.info("Request completed");
        } finally {
            MDC.clear();
        }
    }
}
```

### CloudWatch Logs — Insights Queries

```sql
-- Top 10 endpoints mais lentos
fields @timestamp, httpMethod, httpPath, duration_ms
| filter level = "INFO" and ispresent(duration_ms)
| stats avg(duration_ms) as avg_ms, p99(duration_ms) as p99_ms, count(*) as requests by httpMethod, httpPath
| sort p99_ms desc
| limit 10

-- Error rate por endpoint
fields @timestamp, httpPath, httpStatus
| filter httpStatus >= 500
| stats count(*) as errors by httpPath
| sort errors desc

-- Buscar por correlationId (rastrear request entre serviços)
fields @timestamp, @message, service
| filter correlationId = "corr-uuid-456"
| sort @timestamp asc

-- Slow queries (acima de 1s)
fields @timestamp, httpMethod, httpPath, duration_ms
| filter duration_ms > 1000
| sort duration_ms desc
| limit 20

-- Exceptions com stack trace
fields @timestamp, @message
| filter level = "ERROR"
| parse @message "\"exception\":\"*\"" as exception_type
| stats count(*) as occurrences by exception_type
| sort occurrences desc
```

### Log Groups e Retenção

```hcl
# Log groups com retenção e criptografia
locals {
  log_groups = {
    app    = { retention = var.environment == "prod" ? 90 : 14, prefix = "/ecs/${var.project}" }
    api_gw = { retention = var.environment == "prod" ? 90 : 14, prefix = "/aws/apigateway/${var.project}" }
    lambda = { retention = var.environment == "prod" ? 90 : 14, prefix = "/aws/lambda/${var.project}" }
    vpc    = { retention = var.environment == "prod" ? 365 : 30, prefix = "/aws/vpc/${var.project}" }
    audit  = { retention = 365, prefix = "/aws/cloudtrail/${var.project}" }
  }
}

resource "aws_cloudwatch_log_group" "main" {
  for_each = local.log_groups

  name              = each.value.prefix
  retention_in_days = each.value.retention
  kms_key_id        = aws_kms_key.logs.arn

  tags = var.tags
}

# Subscription filter para centralizar logs
resource "aws_cloudwatch_log_subscription_filter" "to_opensearch" {
  name            = "${var.project}-to-opensearch"
  log_group_name  = aws_cloudwatch_log_group.main["app"].name
  filter_pattern  = ""  # All logs
  destination_arn = aws_lambda_function.log_forwarder.arn
}
```

---

## Traces — Distributed Tracing

### OpenTelemetry com ADOT

```hcl
# ADOT Collector como sidecar no ECS
# Adicionado no task definition como container sidecar

# X-Ray Sampling Rules customizadas
resource "aws_xray_sampling_rule" "main" {
  rule_name      = "${var.project}-default"
  priority       = 1000
  version        = 1
  reservoir_size = 1     # 1 request/segundo garantido
  fixed_rate     = 0.05  # 5% do restante
  url_path       = "*"
  host           = "*"
  http_method    = "*"
  service_type   = "*"
  service_name   = "${var.project}-*"
  resource_arn   = "*"
}

# Sampling rule para health checks (não tracear)
resource "aws_xray_sampling_rule" "health" {
  rule_name      = "${var.project}-health"
  priority       = 100
  version        = 1
  reservoir_size = 0
  fixed_rate     = 0    # 0% — não tracear health checks
  url_path       = "/health*"
  host           = "*"
  http_method    = "GET"
  service_type   = "*"
  service_name   = "${var.project}-*"
  resource_arn   = "*"
}

# Aumentar sampling para errors
resource "aws_xray_sampling_rule" "errors" {
  rule_name      = "${var.project}-errors"
  priority       = 50
  version        = 1
  reservoir_size = 10
  fixed_rate     = 1.0  # 100% — tracear todos os errors
  url_path       = "*"
  host           = "*"
  http_method    = "*"
  service_type   = "*"
  service_name   = "${var.project}-*"
  resource_arn   = "*"
  # Nota: filtrar por status code requer instrumentação no código
}
```

### Instrumentação (Java — Spring Boot)

```java
// OpenTelemetry Auto-instrumentation via Java Agent
// Adicionar ao Dockerfile ou task definition:
// -javaagent:/opt/opentelemetry-javaagent.jar

// Para instrumentação manual:
@Service
public class PaymentService {
    
    private final Tracer tracer;

    public PaymentResult processPayment(PaymentRequest request) {
        Span span = tracer.spanBuilder("processPayment")
                .setAttribute("payment.amount", request.getAmount())
                .setAttribute("payment.currency", request.getCurrency())
                .setAttribute("payment.method", request.getMethod().name())
                .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // Chamar gateway de pagamento
            var result = paymentGateway.charge(request);
            
            span.setAttribute("payment.transactionId", result.getTransactionId());
            span.setAttribute("payment.status", result.getStatus().name());
            span.setStatus(StatusCode.OK);
            
            return result;
        } catch (PaymentDeclinedException e) {
            span.setStatus(StatusCode.ERROR, "Payment declined");
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

---

## Alerting Strategy

### Severidade de Alarmes

| Severidade | Critério | Ação | Destino |
|:----------:|----------|------|---------|
| **SEV-1** | Service down, data loss | Page imediato, war room | PagerDuty/Opsgenie |
| **SEV-2** | Degradação significativa | Page em horário comercial | Slack + PagerDuty |
| **SEV-3** | Anomalia, threshold warning | Notificação | Slack channel |
| **SEV-4** | Informativo | Log apenas | Dashboard |

### SNS Topics por Severidade

```hcl
resource "aws_sns_topic" "alerts" {
  for_each = {
    critical    = "SEV-1: Service down"
    high        = "SEV-2: Degradation"
    medium      = "SEV-3: Warning"
    info        = "SEV-4: Info"
  }

  name              = "${var.project}-alerts-${each.key}"
  kms_master_key_id = aws_kms_key.sns.arn

  tags = merge(var.tags, { Severity = each.key })
}

# Subscriptions
resource "aws_sns_topic_subscription" "pagerduty" {
  topic_arn = aws_sns_topic.alerts["critical"].arn
  protocol  = "https"
  endpoint  = var.pagerduty_endpoint
}

resource "aws_sns_topic_subscription" "slack_high" {
  topic_arn = aws_sns_topic.alerts["high"].arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.slack_notifier.arn
}
```

---

## Synthetic Monitoring & Composite Alarms

### CloudWatch Synthetics (Canaries)

```hcl
# Canary — teste sintético de disponibilidade
resource "aws_synthetics_canary" "api_health" {
  name                 = "${var.project}-api-health"
  artifact_s3_location = "s3://${aws_s3_bucket.canary_artifacts.id}/canary/"
  execution_role_arn   = aws_iam_role.canary.arn
  handler              = "apiCanaryBlueprint.handler"
  runtime_version      = "syn-nodejs-puppeteer-9.1"
  zip_file             = "canary/api-health.zip"

  schedule {
    expression = "rate(5 minutes)"
  }

  run_config {
    timeout_in_seconds = 60
    memory_in_mb       = 960
    active_tracing     = true  # X-Ray integration
  }

  vpc_config {
    subnet_ids         = var.private_subnet_ids
    security_group_ids = [aws_security_group.canary.id]
  }

  tags = var.tags
}

# Script do Canary (API check)
# canary/nodejs/node_modules/apiCanaryBlueprint.js:
# const synthetics = require('Synthetics');
# exports.handler = async () => {
#   const response = await synthetics.executeHttpStep(
#     'Verify API',
#     { hostname: 'api.example.com', path: '/health', port: 443, protocol: 'https:' },
#     (res) => { if (res.statusCode !== 200) throw 'API unhealthy'; }
#   );
# };

# Alarme para Canary falhando
resource "aws_cloudwatch_metric_alarm" "canary_failed" {
  alarm_name          = "${var.project}-canary-failed"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  metric_name         = "SuccessPercent"
  namespace           = "CloudWatchSynthetics"
  period              = 300
  statistic           = "Average"
  threshold           = 90
  alarm_actions       = [aws_sns_topic.alerts["high"].arn]

  dimensions = {
    CanaryName = aws_synthetics_canary.api_health.name
  }
}
```

### Composite Alarms

```hcl
# Composite Alarm — combina múltiplos alarmes para reduzir ruído
resource "aws_cloudwatch_composite_alarm" "service_degraded" {
  alarm_name = "${var.project}-service-degraded"

  alarm_rule = "ALARM(${aws_cloudwatch_metric_alarm.slo_availability.alarm_name}) OR (ALARM(${aws_cloudwatch_metric_alarm.slo_latency.alarm_name}) AND ALARM(${aws_cloudwatch_metric_alarm.rds_cpu.alarm_name}))"

  alarm_actions = [aws_sns_topic.alerts["critical"].arn]
  ok_actions    = [aws_sns_topic.alerts["critical"].arn]

  actions_suppressor                    = aws_cloudwatch_composite_alarm.maintenance_window.alarm_name
  actions_suppressor_wait_period        = 120
  actions_suppressor_extension_period   = 60
}

# Maintenance window suppressor — silencia alarmes durante deploy
resource "aws_cloudwatch_composite_alarm" "maintenance_window" {
  alarm_name = "${var.project}-maintenance-window"
  alarm_rule = "FALSE"  # Ativado manualmente durante deploys
}
```

### CloudWatch Application Signals

Application Signals (preview) auto-descobre serviços e gera SLOs automaticamente:

```hcl
# Habilitar Application Signals no ECS
# Requer ADOT collector com configuração específica

# SLO automático via Application Signals:
# - Availability SLO: % de requests sem 5xx
# - Latency SLO: % de requests abaixo do threshold

# Ativar via ECS Task Definition (environment variable):
# OTEL_AWS_APPLICATION_SIGNALS_ENABLED=true
# OTEL_AWS_APPLICATION_SIGNALS_EXPORTER_ENDPOINT=http://localhost:4316/v1/metrics
```

---

## Observability Checklist

### Métricas

- [ ] Dashboard com RED metrics (Rate, Errors, Duration)
- [ ] Custom metrics para KPIs de negócio (EMF)
- [ ] Alarmes para SLO (availability, latency)
- [ ] Alarmes para recursos (CPU, memory, storage, connections)
- [ ] Alarmes para DLQ não vazia
- [ ] Anomaly detection para métricas-chave

### Logs

- [ ] Structured logging (JSON) em todos os serviços
- [ ] Log levels configuráveis por ambiente
- [ ] Correlation ID propagado entre serviços
- [ ] Retenção definida por tipo de log
- [ ] Logs criptografados (KMS)
- [ ] Queries salvas no CloudWatch Insights
- [ ] Metric filters para erros e padrões importantes

### Traces

- [ ] Distributed tracing habilitado (X-Ray/OTel)
- [ ] Sampling rules customizadas (mais traces para erros)
- [ ] Health checks excluídos do tracing
- [ ] Service map visível
- [ ] Trace ID propagado nos logs (correlação)

### Alerting

- [ ] Severidades definidas e documentadas
- [ ] Runbooks vinculados aos alarmes
- [ ] Escalation policy definida
- [ ] Canais de notificação por severidade
- [ ] Alarmes testados periodicamente

---

## Referências

- [AWS Observability Best Practices](https://aws-observability.github.io/observability-best-practices/)
- [AWS Distro for OpenTelemetry](https://aws-otel.github.io/)
- [CloudWatch Logs Insights Query Syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)
- [AWS X-Ray Documentation](https://docs.aws.amazon.com/xray/)
