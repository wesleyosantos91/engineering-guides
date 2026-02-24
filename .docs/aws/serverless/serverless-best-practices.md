# AWS Serverless — Boas Práticas

Guia completo de boas práticas para arquiteturas serverless na AWS, cobrindo Lambda, API Gateway, Step Functions, EventBridge, SQS e SNS.

---

## Índice

1. [AWS Lambda](#aws-lambda)
2. [API Gateway](#api-gateway)
3. [Step Functions](#step-functions)
4. [EventBridge](#eventbridge)
5. [SQS & SNS](#sqs--sns)
6. [Padrões de Arquitetura](#padrões-de-arquitetura)

---

## AWS Lambda

### Princípios Fundamentais

- **Uma função, uma responsabilidade** — Single Responsibility Principle
- **Stateless** — Nunca armazenar estado na instância
- **Idempotente** — Re-execuções não devem causar side effects
- **Minimalista** — Menos código = menos cold start

### Configuração Terraform

```hcl
# Lambda Function — Configuração otimizada
resource "aws_lambda_function" "main" {
  function_name = "${var.project}-${var.function_name}"
  role          = aws_iam_role.lambda.arn
  handler       = "bootstrap"
  runtime       = "provided.al2023"  # Custom runtime para melhor performance
  architectures = ["arm64"]           # Graviton: 20% mais barato, melhor performance
  
  filename         = var.deployment_package
  source_code_hash = filebase64sha256(var.deployment_package)

  memory_size = 512   # Ajustar via Power Tuning
  timeout     = 30    # Nunca usar timeout máximo (900s) sem necessidade
  
  # Reservar concorrência para funções críticas
  reserved_concurrent_executions = var.reserved_concurrency

  environment {
    variables = {
      ENVIRONMENT    = var.environment
      LOG_LEVEL      = var.environment == "prod" ? "INFO" : "DEBUG"
      POWERTOOLS_SERVICE_NAME = var.function_name
      POWERTOOLS_LOG_LEVEL    = "INFO"
    }
  }

  # VPC apenas se necessário (adiciona cold start)
  dynamic "vpc_config" {
    for_each = var.needs_vpc ? [1] : []
    content {
      subnet_ids         = var.private_subnet_ids
      security_group_ids = [aws_security_group.lambda.id]
    }
  }

  tracing_config {
    mode = "Active"  # X-Ray tracing
  }

  dead_letter_config {
    target_arn = aws_sqs_queue.dlq.arn
  }

  tags = var.tags
}

# Log Group com retenção definida (evitar custos infinitos)
resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${aws_lambda_function.main.function_name}"
  retention_in_days = var.environment == "prod" ? 90 : 14
  kms_key_id        = var.kms_key_arn
}
```

### Cold Start — Estratégias de Mitigação

```
┌────────────────────────────────────────────────────────────────┐
│                  Cold Start Decision Tree                       │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Latência P99 < 1s é crítico?                                 │
│  ├── NÃO → Cold start padrão é aceitável                     │
│  │                                                             │
│  └── SIM → Usar Provisioned Concurrency                       │
│       │                                                        │
│       ├── Tráfego previsível?                                 │
│       │   └── SIM → Provisioned Concurrency fixo              │
│       │                                                        │
│       └── Tráfego variável?                                   │
│           └── SIM → Application Auto Scaling                   │
│                     no Provisioned Concurrency                  │
│                                                                │
│  Otimizações adicionais:                                       │
│  • Usar runtime com menor init (provided.al2023 / Go / Rust) │
│  • Minimizar dependências                                      │
│  • Inicializar SDK clients fora do handler                     │
│  • Usar ARM64 (Graviton)                                      │
│  • Usar SnapStart (Java)                                       │
│  • Evitar VPC quando possível                                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

```hcl
# Provisioned Concurrency com Auto Scaling
resource "aws_lambda_provisioned_concurrency_config" "main" {
  function_name                  = aws_lambda_function.main.function_name
  provisioned_concurrent_executions = var.min_provisioned_concurrency
  qualifier                      = aws_lambda_alias.live.name
}

resource "aws_appautoscaling_target" "lambda" {
  max_capacity       = var.max_provisioned_concurrency
  min_capacity       = var.min_provisioned_concurrency
  resource_id        = "function:${aws_lambda_function.main.function_name}:${aws_lambda_alias.live.name}"
  scalable_dimension = "lambda:function:ProvisionedConcurrency"
  service_namespace  = "lambda"
}

resource "aws_appautoscaling_policy" "lambda" {
  name               = "${var.project}-lambda-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.lambda.resource_id
  scalable_dimension = aws_appautoscaling_target.lambda.scalable_dimension
  service_namespace  = aws_appautoscaling_target.lambda.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "LambdaProvisionedConcurrencyUtilization"
    }
    target_value = 0.7  # Scale quando atingir 70% do provisionado
  }
}
```

### Estrutura do Handler (Java com Powertools)

```java
// Handler Lambda com AWS Lambda Powertools
@Logging(logEvent = true)
@Tracing
@Metrics(captureColdStart = true)
public class OrderHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    // Inicializar FORA do handler — reutilizado entre invocações
    private static final DynamoDbClient dynamoDb = DynamoDbClient.builder()
            .region(Region.of(System.getenv("AWS_REGION")))
            .build();
    
    private static final ObjectMapper mapper = new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

    private final OrderService orderService;

    public OrderHandler() {
        this.orderService = new OrderService(dynamoDb);
    }

    @Override
    public APIGatewayProxyResponseEvent handleRequest(
            APIGatewayProxyRequestEvent event, Context context) {
        
        try {
            // Idempotency key do header
            String idempotencyKey = event.getHeaders()
                    .getOrDefault("X-Idempotency-Key", UUID.randomUUID().toString());
            
            CreateOrderRequest request = mapper.readValue(event.getBody(), CreateOrderRequest.class);
            Order order = orderService.createOrder(request, idempotencyKey);
            
            MetricsLogger metricsLogger = MetricsUtils.metricsLogger();
            metricsLogger.putMetric("OrderCreated", 1, Unit.COUNT);
            
            return new APIGatewayProxyResponseEvent()
                    .withStatusCode(201)
                    .withBody(mapper.writeValueAsString(order))
                    .withHeaders(Map.of("Content-Type", "application/json"));
                    
        } catch (DuplicateOrderException e) {
            return new APIGatewayProxyResponseEvent()
                    .withStatusCode(409)
                    .withBody("{\"error\":\"Order already exists\"}");
        } catch (Exception e) {
            Logger.error("Failed to create order", e);
            return new APIGatewayProxyResponseEvent()
                    .withStatusCode(500)
                    .withBody("{\"error\":\"Internal server error\"}");
        }
    }
}
```

### Lambda Power Tuning

```hcl
# Instalar e executar o AWS Lambda Power Tuning
# https://github.com/alexcasalboni/aws-lambda-power-tuning

# Configuração recomendada para o Power Tuning:
# {
#   "lambdaARN": "arn:aws:lambda:sa-east-1:123456789:function:my-function",
#   "powerValues": [128, 256, 512, 1024, 1536, 2048, 3008],
#   "num": 50,
#   "payload": {"key": "value"},
#   "parallelInvocation": true,
#   "strategy": "balanced"  # cost, speed, balanced
# }
```

### Lambda SnapStart (Java)

Reduz cold start de Java de **~5-10s** para **~200ms** tirando snapshot da JVM inicializada.

```hcl
# Habilitar SnapStart para Lambda Java
resource "aws_lambda_function" "java_function" {
  function_name = "${var.project}-orders"
  role          = aws_iam_role.lambda.arn
  handler       = "com.example.OrderHandler::handleRequest"
  runtime       = "java21"
  architectures = ["arm64"]
  memory_size   = 1024
  timeout       = 30

  filename         = var.deployment_package
  source_code_hash = filebase64sha256(var.deployment_package)

  snap_start {
    apply_on = "PublishedVersions"  # SnapStart ativo
  }

  environment {
    variables = {
      ENVIRONMENT = var.environment
    }
  }

  tags = var.tags
}

# OBRIGATÓRIO: SnapStart requer versão publicada + alias
resource "aws_lambda_alias" "live" {
  name             = "live"
  function_name    = aws_lambda_function.java_function.function_name
  function_version = aws_lambda_function.java_function.version
}
```

**Limitações do SnapStart:**
- Apenas Java 11, 17 e 21 (managed runtimes)
- Não suporta Provisioned Concurrency simultaneamente
- Não suporta EFS, `arm64` com custom runtime, ou tamanho > 250MB (descompactado)
- Uniqueness: evitar caching de valores randômicos no init (ex: `UUID.randomUUID()`)
- Usar `CRaC` (Coordinated Restore at Checkpoint) para hooks de restore

```java
// Hook de restore — regenerar valores que devem ser únicos pós-restore
import org.crac.Context;
import org.crac.Core;
import org.crac.Resource;

public class UniqueIdGenerator implements Resource {
    private String instanceId;

    public UniqueIdGenerator() {
        Core.getGlobalContext().register(this);
        this.instanceId = UUID.randomUUID().toString();
    }

    @Override
    public void afterRestore(Context<? extends Resource> context) {
        // Regenerar após restore do snapshot
        this.instanceId = UUID.randomUUID().toString();
    }
}
```

### Lambda Layers

```hcl
# Layer para dependências compartilhadas (AWS SDK, Powertools, etc)
resource "aws_lambda_layer_version" "shared_deps" {
  layer_name          = "${var.project}-shared-deps"
  filename            = "layers/shared-deps.zip"
  source_code_hash    = filebase64sha256("layers/shared-deps.zip")
  compatible_runtimes = ["java21"]
  description         = "Shared dependencies: AWS SDK v2, Powertools, Jackson"
}

# Aplicar layer à função
resource "aws_lambda_function" "with_layer" {
  function_name = "${var.project}-processor"
  # ...

  layers = [
    aws_lambda_layer_version.shared_deps.arn,
    # Layer gerenciado da AWS (Powertools)
    "arn:aws:lambda:${var.region}:094274105915:layer:AWSLambdaPowertoolsJavaV2:51"
  ]
}
```

**Boas práticas para Layers:**
- Máximo 5 layers por função, total ≤ 250MB (descompactado)
- Usar para: dependências compartilhadas, extensions, custom runtimes
- Estrutura do ZIP: `java/lib/` para jars em Java
- Versionar layers (cada deploy cria nova versão)

### Lambda@Edge & CloudFront Functions

```hcl
# Lambda@Edge — executa no edge (CloudFront)
resource "aws_lambda_function" "at_edge" {
  provider = aws.us_east_1  # OBRIGATÓRIO: Lambda@Edge deve estar em us-east-1

  function_name = "${var.project}-auth-at-edge"
  role          = aws_iam_role.lambda_edge.arn
  handler       = "index.handler"
  runtime       = "nodejs20.x"  # Lambda@Edge: Node.js ou Python
  timeout       = 5             # Max 5s para viewer events, 30s para origin events
  memory_size   = 128           # Max 128MB para viewer events

  filename         = "edge/auth.zip"
  source_code_hash = filebase64sha256("edge/auth.zip")
  publish          = true  # Required for Lambda@Edge
}

# Associar ao CloudFront
resource "aws_cloudfront_distribution" "main" {
  # ...
  default_cache_behavior {
    # ...
    lambda_function_association {
      event_type   = "viewer-request"           # Antes de chegar ao cache
      lambda_arn   = aws_lambda_function.at_edge.qualified_arn
      include_body = false
    }
  }
}
```

```
Lambda@Edge vs CloudFront Functions:
┌──────────────────────┬──────────────────────┬──────────────────┐
│                      │ Lambda@Edge          │ CloudFront Func  │
├──────────────────────┼──────────────────────┼──────────────────┤
│ Runtime              │ Node.js, Python      │ JavaScript only  │
│ Execution time       │ 5s (viewer) / 30s    │ < 1ms            │
│ Memory               │ 128MB-10GB           │ 2MB              │
│ Network access       │ Sim                  │ Não              │
│ Preço                │ ~$0.60 / 1M requests │ ~$0.10 / 1M      │
│ Use cases            │ Auth, A/B test,      │ Header/URL       │
│                      │ origin selection     │ manipulation     │
└──────────────────────┴──────────────────────┴──────────────────┘
```

---

## API Gateway

### REST API vs HTTP API

```
┌──────────────────────────────────────────────────────────────┐
│              REST API vs HTTP API — Quando usar               │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  HTTP API (recomendado para maioria dos casos)               │
│  ├── Até 71% mais barato que REST API                       │
│  ├── Latência menor (~10ms vs ~29ms p50)                    │
│  ├── JWT Authorizer nativo                                   │
│  ├── CORS simplificado                                       │
│  └── Ideal para: CRUD APIs, webhooks, SPAs                  │
│                                                              │
│  REST API (quando precisa de recursos avançados)             │
│  ├── API Keys e Usage Plans                                  │
│  ├── Request/Response transformation                         │
│  ├── Request validation                                      │
│  ├── WAF integration                                         │
│  ├── Caching nativo                                          │
│  ├── Private endpoints                                       │
│  └── Ideal para: APIs públicas, marketplace, B2B             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### HTTP API Gateway com Lambda

```hcl
# HTTP API (v2) — mais barato e performático
resource "aws_apigatewayv2_api" "main" {
  name          = "${var.project}-api"
  protocol_type = "HTTP"
  
  cors_configuration {
    allow_headers = ["Content-Type", "Authorization", "X-Amz-Date"]
    allow_methods = ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
    allow_origins = var.allowed_origins
    max_age       = 3600
  }
}

resource "aws_apigatewayv2_stage" "main" {
  api_id      = aws_apigatewayv2_api.main.id
  name        = var.environment
  auto_deploy = true

  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api.arn
    format = jsonencode({
      requestId      = "$context.requestId"
      ip             = "$context.identity.sourceIp"
      requestTime    = "$context.requestTime"
      httpMethod     = "$context.httpMethod"
      routeKey       = "$context.routeKey"
      status         = "$context.status"
      protocol       = "$context.protocol"
      responseLength = "$context.responseLength"
      integrationLatency = "$context.integrationLatency"
      errorMessage   = "$context.error.message"
    })
  }

  default_route_settings {
    throttling_burst_limit = 100
    throttling_rate_limit  = 50
  }
}

# JWT Authorizer (Cognito ou qualquer OIDC provider)
resource "aws_apigatewayv2_authorizer" "jwt" {
  api_id           = aws_apigatewayv2_api.main.id
  authorizer_type  = "JWT"
  identity_sources = ["$request.header.Authorization"]
  name             = "jwt-authorizer"

  jwt_configuration {
    audience = [var.cognito_client_id]
    issuer   = "https://cognito-idp.${var.region}.amazonaws.com/${var.cognito_user_pool_id}"
  }
}

# Route com authorizer
resource "aws_apigatewayv2_route" "create_order" {
  api_id    = aws_apigatewayv2_api.main.id
  route_key = "POST /orders"
  target    = "integrations/${aws_apigatewayv2_integration.create_order.id}"
  
  authorization_type = "JWT"
  authorizer_id      = aws_apigatewayv2_authorizer.jwt.id
}

resource "aws_apigatewayv2_integration" "create_order" {
  api_id                 = aws_apigatewayv2_api.main.id
  integration_type       = "AWS_PROXY"
  integration_uri        = aws_lambda_function.create_order.invoke_arn
  payload_format_version = "2.0"
}
```

### Throttling e Rate Limiting

```hcl
# REST API com Usage Plans
resource "aws_api_gateway_usage_plan" "main" {
  name = "${var.project}-usage-plan"

  api_stages {
    api_id = aws_api_gateway_rest_api.main.id
    stage  = aws_api_gateway_stage.main.stage_name

    throttle {
      path        = "/orders/POST"
      burst_limit = 50
      rate_limit  = 25
    }
  }

  throttle_settings {
    burst_limit = 200
    rate_limit  = 100
  }

  quota_settings {
    limit  = 10000
    period = "DAY"
  }
}
```

---

## Step Functions

### Quando Usar

- Orquestração de múltiplos serviços
- Workflows de longa duração
- Processos com decisões condicionais
- Retry pattern complexo
- Human-in-the-loop workflows

### Standard vs Express

```
┌────────────────────────────────────────────────────────────────┐
│           Step Functions: Standard vs Express                   │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Standard Workflows                                            │
│  ├── Duração: até 1 ano                                       │
│  ├── Preço: por transição de estado                           │
│  ├── Execução: exatamente uma vez                             │
│  ├── Histórico: 90 dias                                       │
│  └── Ideal para: orquestração longa, human approval           │
│                                                                │
│  Express Workflows                                              │
│  ├── Duração: até 5 minutos                                   │
│  ├── Preço: por execução + duração (muito mais barato)        │
│  ├── Execução: pelo menos uma vez (async) / exatamente (sync) │
│  ├── Histórico: CloudWatch Logs                               │
│  └── Ideal para: alta volumetria, data processing, IoT        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Exemplo: Order Processing Workflow

```hcl
resource "aws_sfn_state_machine" "order_processing" {
  name     = "${var.project}-order-processing"
  role_arn = aws_iam_role.sfn.arn
  type     = "STANDARD"

  definition = jsonencode({
    Comment = "Order Processing Workflow"
    StartAt = "ValidateOrder"
    States = {
      ValidateOrder = {
        Type     = "Task"
        Resource = aws_lambda_function.validate_order.arn
        Next     = "CheckInventory"
        Retry = [
          {
            ErrorEquals     = ["Lambda.ServiceException", "Lambda.TooManyRequestsException"]
            IntervalSeconds = 2
            MaxAttempts     = 3
            BackoffRate     = 2
          }
        ]
        Catch = [
          {
            ErrorEquals = ["ValidationError"]
            Next        = "OrderFailed"
            ResultPath  = "$.error"
          }
        ]
      }

      CheckInventory = {
        Type     = "Task"
        Resource = aws_lambda_function.check_inventory.arn
        Next     = "HasInventory"
        Retry = [
          {
            ErrorEquals     = ["States.TaskFailed"]
            IntervalSeconds = 5
            MaxAttempts     = 2
            BackoffRate     = 2
          }
        ]
      }

      HasInventory = {
        Type = "Choice"
        Choices = [
          {
            Variable     = "$.inventoryAvailable"
            BooleanEquals = true
            Next         = "ProcessPayment"
          }
        ]
        Default = "WaitForInventory"
      }

      WaitForInventory = {
        Type    = "Wait"
        Seconds = 300
        Next    = "CheckInventory"
      }

      ProcessPayment = {
        Type     = "Task"
        Resource = aws_lambda_function.process_payment.arn
        Next     = "ParallelFulfillment"
        Retry = [
          {
            ErrorEquals     = ["PaymentRetryableError"]
            IntervalSeconds = 5
            MaxAttempts     = 3
            BackoffRate     = 2
          }
        ]
        Catch = [
          {
            ErrorEquals = ["PaymentDeclinedError"]
            Next        = "OrderFailed"
            ResultPath  = "$.error"
          }
        ]
      }

      ParallelFulfillment = {
        Type = "Parallel"
        Branches = [
          {
            StartAt = "SendConfirmation"
            States = {
              SendConfirmation = {
                Type     = "Task"
                Resource = "arn:aws:states:::sns:publish"
                Parameters = {
                  TopicArn = aws_sns_topic.order_notifications.arn
                  "Message.$" = "$.orderConfirmation"
                }
                End = true
              }
            }
          },
          {
            StartAt = "UpdateInventory"
            States = {
              UpdateInventory = {
                Type     = "Task"
                Resource = aws_lambda_function.update_inventory.arn
                End      = true
              }
            }
          },
          {
            StartAt = "InitiateShipping"
            States = {
              InitiateShipping = {
                Type     = "Task"
                Resource = aws_lambda_function.initiate_shipping.arn
                End      = true
              }
            }
          }
        ]
        Next = "OrderCompleted"
      }

      OrderCompleted = {
        Type = "Succeed"
      }

      OrderFailed = {
        Type  = "Task"
        Resource = aws_lambda_function.handle_failure.arn
        Next  = "FailState"
      }

      FailState = {
        Type  = "Fail"
        Error = "OrderProcessingFailed"
        Cause = "Order processing failed — check error details"
      }
    }
  })

  logging_configuration {
    log_destination        = "${aws_cloudwatch_log_group.sfn.arn}:*"
    include_execution_data = true
    level                  = "ALL"
  }

  tracing_configuration {
    enabled = true
  }
}
```

---

## EventBridge

### Boas Práticas

- **Usar schema registry** — Documentar e versionar schemas de eventos
- **Nomenclatura consistente** — `{company}.{domain}.{event-type}.{version}`
- **Dead Letter Queue** — Sempre configurar DLQ para rules
- **Archive** — Habilitar replay de eventos para debugging
- **Cross-account** — Usar event bus dedicado por domínio

### Configuração

```hcl
# Event Bus customizado
resource "aws_cloudwatch_event_bus" "orders" {
  name = "${var.project}-orders"
}

# Archive para replay
resource "aws_cloudwatch_event_archive" "orders" {
  name             = "${var.project}-orders-archive"
  event_source_arn = aws_cloudwatch_event_bus.orders.arn
  retention_days   = 30
}

# Schema Registry
resource "aws_schemas_registry" "main" {
  name        = "${var.project}-events"
  description = "Event schemas for ${var.project}"
}

resource "aws_schemas_schema" "order_created" {
  name          = "OrderCreated"
  registry_name = aws_schemas_registry.main.name
  type          = "OpenApi3"
  description   = "Event emitted when a new order is created"

  content = jsonencode({
    openapi = "3.0.0"
    info = {
      title   = "OrderCreated"
      version = "1.0.0"
    }
    paths = {}
    components = {
      schemas = {
        OrderCreated = {
          type = "object"
          required = ["orderId", "customerId", "amount", "timestamp"]
          properties = {
            orderId    = { type = "string", format = "uuid" }
            customerId = { type = "string", format = "uuid" }
            amount     = { type = "number" }
            currency   = { type = "string", enum = ["BRL", "USD"] }
            items      = {
              type  = "array"
              items = {
                type = "object"
                properties = {
                  productId = { type = "string" }
                  quantity  = { type = "integer" }
                  price     = { type = "number" }
                }
              }
            }
            timestamp = { type = "string", format = "date-time" }
          }
        }
      }
    }
  })
}

# Rule com padrão de evento
resource "aws_cloudwatch_event_rule" "order_created" {
  name           = "${var.project}-order-created"
  event_bus_name = aws_cloudwatch_event_bus.orders.name
  description    = "Capture order created events"

  event_pattern = jsonencode({
    source      = ["${var.project}.orders"]
    detail-type = ["OrderCreated"]
    detail = {
      amount = [{
        numeric = [">", 0]
      }]
    }
  })
}

# Target com DLQ e retry policy
resource "aws_cloudwatch_event_target" "process_order" {
  rule           = aws_cloudwatch_event_rule.order_created.name
  event_bus_name = aws_cloudwatch_event_bus.orders.name
  target_id      = "process-order-lambda"
  arn            = aws_lambda_function.process_order.arn

  retry_policy {
    maximum_event_age_in_seconds = 3600  # 1 hora
    maximum_retry_attempts       = 3
  }

  dead_letter_config {
    arn = aws_sqs_queue.events_dlq.arn
  }
}
```

### Publicar Eventos (Java)

```java
// Publicar evento no EventBridge
@Service
public class EventPublisher {
    
    private final EventBridgeClient eventBridge;
    private final ObjectMapper mapper;
    private final String eventBusName;

    public void publishOrderCreated(Order order) {
        var event = OrderCreatedEvent.builder()
                .orderId(order.getId())
                .customerId(order.getCustomerId())
                .amount(order.getAmount())
                .currency(order.getCurrency())
                .items(order.getItems())
                .timestamp(Instant.now())
                .build();

        var entry = PutEventsRequestEntry.builder()
                .eventBusName(eventBusName)
                .source("myapp.orders")
                .detailType("OrderCreated")
                .detail(mapper.writeValueAsString(event))
                .build();

        var response = eventBridge.putEvents(
                PutEventsRequest.builder().entries(entry).build());

        if (response.failedEntryCount() > 0) {
            log.error("Failed to publish event: {}", response.entries());
            // Retry ou fallback para SQS
        }
    }
}
```

---

## SQS & SNS

### SQS — Boas Práticas

```hcl
# SQS Queue com DLQ e criptografia
resource "aws_sqs_queue" "main" {
  name                       = "${var.project}-${var.queue_name}"
  visibility_timeout_seconds = 300  # 6x o timeout da Lambda consumer
  message_retention_seconds  = 1209600  # 14 dias
  receive_wait_time_seconds  = 20       # Long polling (economiza custos)
  max_message_size           = 262144   # 256 KB
  
  sqs_managed_sse_enabled = true  # Ou KMS para compliance

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 3  # Após 3 falhas, vai para DLQ
  })

  tags = var.tags
}

# Dead Letter Queue
resource "aws_sqs_queue" "dlq" {
  name                      = "${var.project}-${var.queue_name}-dlq"
  message_retention_seconds = 1209600  # 14 dias

  sqs_managed_sse_enabled = true

  tags = var.tags
}

# Alarme para mensagens na DLQ
resource "aws_cloudwatch_metric_alarm" "dlq_messages" {
  alarm_name          = "${var.project}-${var.queue_name}-dlq-not-empty"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "Messages in DLQ — requires investigation"
  alarm_actions       = [var.sns_alerts_arn]

  dimensions = {
    QueueName = aws_sqs_queue.dlq.name
  }
}

# Lambda Event Source Mapping
resource "aws_lambda_event_source_mapping" "sqs" {
  event_source_arn                   = aws_sqs_queue.main.arn
  function_name                      = aws_lambda_function.consumer.arn
  batch_size                         = 10
  maximum_batching_window_in_seconds = 5  # Batch para reduzir invocações
  
  function_response_types = ["ReportBatchItemFailures"]  # Partial batch failure

  scaling_configuration {
    maximum_concurrency = 10  # Limitar concorrência
  }
}
```

### FIFO Queue — Quando Usar

```hcl
# FIFO Queue — garantia de ordenação e exatamente uma vez
resource "aws_sqs_queue" "fifo" {
  name                        = "${var.project}-orders.fifo"
  fifo_queue                  = true
  content_based_deduplication = true   # Deduplica baseado no hash do body
  deduplication_scope         = "messageGroup"
  fifo_throughput_limit       = "perMessageGroupId"  # High throughput mode
  
  visibility_timeout_seconds = 300
  
  sqs_managed_sse_enabled = true

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.fifo_dlq.arn
    maxReceiveCount     = 3
  })
}
```

### SNS — Fan-Out Pattern

```hcl
# SNS Topic com criptografia
resource "aws_sns_topic" "order_events" {
  name              = "${var.project}-order-events"
  kms_master_key_id = aws_kms_key.main.id
  
  # Política de delivery retry
  delivery_policy = jsonencode({
    http = {
      defaultHealthyRetryPolicy = {
        minDelayTarget     = 20
        maxDelayTarget     = 20
        numRetries         = 3
        numNoDelayRetries  = 0
        backoffFunction    = "linear"
      }
      disableSubscriptionOverrides = false
    }
  })
}

# Fan-out: um evento → múltiplos consumers
resource "aws_sns_topic_subscription" "to_sqs_fulfillment" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.fulfillment.arn
  
  filter_policy = jsonencode({
    eventType = ["OrderCreated", "OrderUpdated"]
    amount    = [{ numeric = [">=", 100] }]
  })
  filter_policy_scope = "MessageBody"
  
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.sns_dlq.arn
  })
}

resource "aws_sns_topic_subscription" "to_sqs_analytics" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.analytics.arn
  
  # Sem filtro — recebe tudo
}

resource "aws_sns_topic_subscription" "to_lambda_notification" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.send_notification.arn
  
  filter_policy = jsonencode({
    eventType = ["OrderCreated"]
  })
  filter_policy_scope = "MessageBody"
}
```

---

## Padrões de Arquitetura

### 1. API Pattern (Synchronous)

```
Client → API Gateway → Lambda → DynamoDB
                    └→ (response)
```

### 2. Queue Processing (Asynchronous)

```
Producer → SQS Queue → Lambda Consumer → DynamoDB
                  └→ DLQ (falhas)
```

### 3. Fan-Out Pattern

```
Producer → SNS Topic ─→ SQS Queue A → Lambda A
                     ├→ SQS Queue B → Lambda B
                     └→ Lambda C
```

### 4. Event-Driven (EventBridge)

```
Service A → EventBridge ─→ Lambda B (via Rule)
                        ├→ Step Functions (via Rule)
                        ├→ SQS Queue (via Rule)
                        └→ Archive (replay)
```

### 5. Saga Pattern (Choreography)

```
Order Service → EventBridge → Payment Service → EventBridge → Shipping Service
                           └→ (PaymentFailed) → Compensate Order
```

### 6. Saga Pattern (Orchestration)

```
API Gateway → Step Functions → Lambda: Validate
                            → Lambda: Payment
                            → Lambda: Inventory
                            → Lambda: Shipping
                            → (Catch → Compensate)
```

---

## Anti-Patterns Serverless

| Anti-Pattern | Problema | Solução |
|-------------|----------|---------|
| Lambda Monolítica | Função faz tudo, difícil de escalar | Uma função por operação |
| Synchronous Chain | Lambda → Lambda → Lambda | Usar Step Functions ou EventBridge |
| Over-Orchestration | Step Functions para 1 step | Lambda direto |
| No Timeout | Lambda com timeout 900s | Timeout justo + 30% margem |
| VPC Desnecessária | Lambda em VPC sem necessidade | VPC apenas para acessar recursos privados |
| Fat Functions | Deploy com todas as deps | Tree-shaking, layers seletivos |
| No DLQ | Mensagens perdidas silenciosamente | Sempre configurar DLQ |
| Polling SQS sem Long Polling | Custo alto e latência | `receive_wait_time_seconds = 20` |
| Ignorar Cold Start | Surpresa em produção | Medir, otimizar ou Provisioned Concurrency |

---

## Referências

- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [Serverless Land — Patterns](https://serverlessland.com/patterns)
- [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning)
- [AWS Lambda Powertools](https://docs.powertools.aws.dev/lambda/java/)
- [EventBridge Best Practices](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-best-practices.html)
