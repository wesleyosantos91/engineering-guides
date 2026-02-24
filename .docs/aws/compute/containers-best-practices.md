# AWS Containers — Boas Práticas

Guia de boas práticas para orquestração de containers na AWS com ECS, EKS e Fargate.

---

## Decisão: ECS vs EKS vs App Runner

```
┌──────────────────────────────────────────────────────────────────┐
│              Árvore de Decisão: Container Orchestration           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Precisa de Kubernetes nativo?                                   │
│  ├── SIM → Amazon EKS                                           │
│  │    ├── Quer gerenciar nodes? → EKS on EC2                   │
│  │    └── Serverless?           → EKS on Fargate               │
│  │                                                               │
│  └── NÃO → Simplicidade é prioridade?                           │
│       ├── SIM, web app simples → AWS App Runner                 │
│       │                          (zero config, auto scaling)    │
│       │                                                          │
│       └── NÃO → ECS                                             │
│            ├── Serverless?       → ECS on Fargate               │
│            │                      (recomendado para maioria)    │
│            └── GPU/custom AMI?   → ECS on EC2                   │
│                                                                  │
│  Comparação de custo (ordem decrescente):                       │
│  EC2 (Reserved/Spot) < Fargate (Spot) < Fargate < App Runner   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## ECS on Fargate (Recomendado)

### Cluster

```hcl
resource "aws_ecs_cluster" "main" {
  name = "${var.project}-${var.environment}"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  configuration {
    execute_command_configuration {
      kms_key_id = aws_kms_key.main.arn
      logging    = "OVERRIDE"

      log_configuration {
        cloud_watch_log_group_name = aws_cloudwatch_log_group.ecs_exec.name
      }
    }
  }

  tags = var.tags
}

# Capacity providers
resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name

  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = var.environment == "prod" ? 80 : 0
    base              = var.environment == "prod" ? 2 : 0
  }

  default_capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = var.environment == "prod" ? 20 : 100
  }
}
```

### Task Definition

```hcl
resource "aws_ecs_task_definition" "main" {
  family                   = "${var.project}-${var.service_name}"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  
  # Usar ARM64 (Graviton) — 20% mais barato, melhor performance
  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture        = "ARM64"
  }

  cpu    = var.cpu     # 256, 512, 1024, 2048, 4096
  memory = var.memory  # Depende do cpu escolhido

  execution_role_arn = aws_iam_role.ecs_execution.arn  # Pull image, logs
  task_role_arn      = aws_iam_role.ecs_task.arn        # App permissions

  container_definitions = jsonencode([
    {
      name  = var.service_name
      image = "${var.ecr_repository_url}:${var.image_tag}"
      
      essential = true
      
      portMappings = [
        {
          containerPort = var.container_port
          protocol      = "tcp"
        }
      ]

      # Health check no container
      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:${var.container_port}/health || exit 1"]
        interval    = 15
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }

      # Logs
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = var.region
          "awslogs-stream-prefix" = var.service_name
        }
      }

      # Environment
      environment = [
        { name = "ENVIRONMENT", value = var.environment },
        { name = "PORT", value = tostring(var.container_port) },
        { name = "OTEL_SERVICE_NAME", value = var.service_name },
        { name = "OTEL_EXPORTER_OTLP_ENDPOINT", value = "http://localhost:4317" }
      ]

      # Secrets do Secrets Manager / SSM
      secrets = [
        {
          name      = "DATABASE_URL"
          valueFrom = aws_secretsmanager_secret.db_url.arn
        },
        {
          name      = "API_KEY"
          valueFrom = "${aws_ssm_parameter.api_key.arn}"
        }
      ]

      # Resource limits
      ulimits = [
        {
          name      = "nofile"
          hardLimit = 65536
          softLimit = 65536
        }
      ]

      # Security: read-only filesystem
      readonlyRootFilesystem = true
      
      # Security: non-root user
      user = "1000:1000"

      # Volumes para paths que precisam de escrita
      mountPoints = [
        {
          sourceVolume  = "tmp"
          containerPath = "/tmp"
          readOnly      = false
        }
      ]
    },
    # Sidecar: ADOT Collector (OpenTelemetry)
    {
      name      = "otel-collector"
      image     = "public.ecr.aws/aws-observability/aws-otel-collector:latest"
      essential = false
      
      command = ["--config=/etc/ecs/ecs-default-config.yaml"]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.otel.name
          "awslogs-region"        = var.region
          "awslogs-stream-prefix" = "otel"
        }
      }

      healthCheck = {
        command     = ["/healthcheck"]
        interval    = 5
        timeout     = 6
        retries     = 5
        startPeriod = 1
      }
    }
  ])

  volume {
    name = "tmp"
  }

  tags = var.tags
}
```

### Service

```hcl
resource "aws_ecs_service" "main" {
  name            = "${var.project}-${var.service_name}"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.main.arn
  desired_count   = var.desired_count

  # Capacity provider strategy
  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = var.environment == "prod" ? 80 : 0
    base              = var.environment == "prod" ? 2 : 0
  }

  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = var.environment == "prod" ? 20 : 100
  }

  # Deployment
  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 100
  }

  # Enable ECS Exec (debug em prod)
  enable_execute_command = true

  # Load Balancer
  load_balancer {
    target_group_arn = aws_lb_target_group.main.arn
    container_name   = var.service_name
    container_port   = var.container_port
  }

  # Network
  network_configuration {
    security_groups  = [aws_security_group.ecs.id]
    subnets          = var.private_subnet_ids
    assign_public_ip = false
  }

  # Service Discovery (opcional — para comunicação interna)
  service_registries {
    registry_arn = aws_service_discovery_service.main.arn
  }

  # Aguardar stabilização
  wait_for_steady_state = false

  lifecycle {
    ignore_changes = [desired_count]  # Ignorar changes do auto scaling
  }

  depends_on = [aws_lb_listener_rule.main]
}
```

### Auto Scaling

```hcl
resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = var.max_capacity
  min_capacity       = var.min_capacity
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.main.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

# Scale by CPU
resource "aws_appautoscaling_policy" "cpu" {
  name               = "${var.project}-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 70.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}

# Scale by Memory
resource "aws_appautoscaling_policy" "memory" {
  name               = "${var.project}-memory-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageMemoryUtilization"
    }
    target_value       = 80.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}

# Scale by Request Count per Target
resource "aws_appautoscaling_policy" "requests" {
  name               = "${var.project}-request-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label         = "${aws_lb.main.arn_suffix}/${aws_lb_target_group.main.arn_suffix}"
    }
    target_value       = 500
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

---

## Container Image Best Practices

### Dockerfile Otimizado (Java)

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY . .
RUN ./gradlew bootJar --no-daemon

# Runtime stage — imagem mínima
FROM eclipse-temurin:21-jre-alpine

# Security: non-root user
RUN addgroup -g 1000 appgroup && \
    adduser -u 1000 -G appgroup -s /bin/sh -D appuser

WORKDIR /app

# Copiar apenas o JAR
COPY --from=builder /app/build/libs/*.jar app.jar

# Security: non-root
USER appuser:appgroup

# Health check
HEALTHCHECK --interval=15s --timeout=5s --start-period=60s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

EXPOSE 8080

ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:+UseG1GC", \
  "-Djava.security.egd=file:/dev/urandom", \
  "-jar", "app.jar"]
```

### ECR — Lifecycle e Scanning

```hcl
resource "aws_ecr_repository" "main" {
  name                 = "${var.project}/${var.service_name}"
  image_tag_mutability = "IMMUTABLE"  # Tags imutáveis para rastreabilidade

  image_scanning_configuration {
    scan_on_push = true  # Scan de vulnerabilidades automático
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = var.tags
}

# Lifecycle: manter últimas N imagens
resource "aws_ecr_lifecycle_policy" "main" {
  repository = aws_ecr_repository.main.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 20 tagged images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v"]
          countType     = "imageCountMoreThan"
          countNumber   = 20
        }
        action = {
          type = "expire"
        }
      },
      {
        rulePriority = 2
        description  = "Remove untagged images after 7 days"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 7
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}
```

---

## Service Discovery

```hcl
# Namespace para service discovery interno
resource "aws_service_discovery_private_dns_namespace" "main" {
  name        = "${var.project}.local"
  description = "Service Discovery for ${var.project}"
  vpc         = aws_vpc.main.id
}

# Registro de serviço
resource "aws_service_discovery_service" "main" {
  name = var.service_name

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.main.id

    dns_records {
      ttl  = 10
      type = "A"
    }

    routing_policy = "MULTIVALUE"
  }

  health_check_custom_config {
    failure_threshold = 1
  }
}

# Uso: outro serviço acessa via DNS
# http://order-service.myproject.local:8080/api/orders
```

---

## Deployment Strategies

### Blue/Green com CodeDeploy

```hcl
resource "aws_codedeploy_app" "main" {
  name             = "${var.project}-${var.service_name}"
  compute_platform = "ECS"
}

resource "aws_codedeploy_deployment_group" "main" {
  app_name               = aws_codedeploy_app.main.name
  deployment_group_name  = "${var.project}-${var.service_name}"
  deployment_config_name = "CodeDeployDefault.ECSLinear10PercentEvery1Minutes"
  service_role_arn       = aws_iam_role.codedeploy.arn

  auto_rollback_configuration {
    enabled = true
    events  = ["DEPLOYMENT_FAILURE", "DEPLOYMENT_STOP_ON_ALARM"]
  }

  blue_green_deployment_config {
    deployment_ready_option {
      action_on_timeout = "CONTINUE_DEPLOYMENT"
    }

    terminate_blue_instances_on_deployment_success {
      action                           = "TERMINATE"
      termination_wait_time_in_minutes = 5
    }
  }

  deployment_style {
    deployment_option = "WITH_TRAFFIC_CONTROL"
    deployment_type   = "BLUE_GREEN"
  }

  ecs_service {
    cluster_name = aws_ecs_cluster.main.name
    service_name = aws_ecs_service.main.name
  }

  load_balancer_info {
    target_group_pair_info {
      prod_traffic_route {
        listener_arns = [aws_lb_listener.https.arn]
      }

      test_traffic_route {
        listener_arns = [aws_lb_listener.test.arn]
      }

      target_group {
        name = aws_lb_target_group.blue.name
      }

      target_group {
        name = aws_lb_target_group.green.name
      }
    }
  }

  alarm_configuration {
    alarms  = [aws_cloudwatch_metric_alarm.error_rate.alarm_name]
    enabled = true
  }
}
```

---

## Container Security Checklist

- [ ] Imagens base oficiais e atualizadas
- [ ] Multi-stage build (não incluir build tools na imagem final)
- [ ] Non-root user no container
- [ ] Read-only root filesystem
- [ ] Scan de vulnerabilidades no push (ECR)
- [ ] Imutable tags (sem `latest` em produção)
- [ ] Secrets via Secrets Manager/SSM (nunca em env vars no task def)
- [ ] Task Role com menor privilégio (separado de execution role)
- [ ] Logs centralizados (CloudWatch ou OpenSearch)
- [ ] Health checks configurados (container + ALB)
- [ ] Resource limits definidos (CPU + Memory)
- [ ] Network policy (Security Groups restritivos)

---

## Referências

- [Amazon ECS Best Practices](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/)
- [Amazon EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [Fargate Documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html)
- [Docker Best Practices](https://docs.docker.com/build/building/best-practices/)
