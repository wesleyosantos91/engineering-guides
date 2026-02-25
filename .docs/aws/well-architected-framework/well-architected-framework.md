# AWS Well-Architected Framework

O Well-Architected Framework fornece orientações para projetar e operar sistemas confiáveis, seguros, eficientes, econômicos e sustentáveis na nuvem AWS. É composto por **6 pilares**.

---

## Visão Geral dos Pilares

| Pilar | Foco Principal | Serviços-Chave |
|-------|---------------|----------------|
| Excelência Operacional | Executar e monitorar sistemas | CloudFormation, Config, CloudTrail |
| Segurança | Proteger dados e sistemas | IAM, KMS, GuardDuty, Security Hub |
| Confiabilidade | Recuperação de falhas | Route 53, Auto Scaling, S3, RDS Multi-AZ |
| Eficiência de Performance | Usar recursos eficientemente | CloudWatch, Auto Scaling, CloudFront |
| Otimização de Custos | Evitar gastos desnecessários | Cost Explorer, Budgets, Savings Plans |
| Sustentabilidade | Minimizar impacto ambiental | Graviton, Auto Scaling, S3 Intelligent-Tiering |

---

## 1. Excelência Operacional (Operational Excellence)

### Princípios de Design

- **Executar operações como código** — Definir toda a carga de trabalho como código (IaC)
- **Fazer mudanças frequentes, pequenas e reversíveis** — Deploy incremental
- **Refinar procedimentos operacionais com frequência** — Revisar runbooks
- **Antecipar falhas** — Realizar exercícios de game day
- **Aprender com todas as falhas operacionais** — Post-mortems sem culpa

### Boas Práticas

#### Organização

```yaml
# Estrutura de contas AWS Organizations
organization:
  management-account:
    purpose: "Billing e governança"
  
  security-ou:
    - security-tooling:    "GuardDuty delegado, Security Hub"
    - log-archive:         "Centralização de logs"
  
  infrastructure-ou:
    - networking:           "Transit Gateway, DNS"
    - shared-services:      "CI/CD, Container Registry"
  
  workloads-ou:
    - development:          "Ambiente dev"
    - staging:              "Ambiente staging"
    - production:           "Ambiente produção"
  
  sandbox-ou:
    - sandbox:              "Experimentação"
```

#### Preparação

```hcl
# Runbook automatizado com SSM
resource "aws_ssm_document" "restart_service" {
  name            = "restart-ecs-service"
  document_type   = "Automation"
  document_format = "YAML"

  content = yamlencode({
    schemaVersion = "0.3"
    description   = "Restart ECS Service"
    parameters = {
      ClusterName = { type = "String" }
      ServiceName = { type = "String" }
    }
    mainSteps = [
      {
        name   = "restartService"
        action = "aws:executeAwsApi"
        inputs = {
          Service    = "ecs"
          Api        = "UpdateService"
          cluster    = "{{ ClusterName }}"
          service    = "{{ ServiceName }}"
          forceNewDeployment = true
        }
      }
    ]
  })
}
```

#### Operação — Observabilidade

```hcl
# Dashboard operacional CloudWatch
resource "aws_cloudwatch_dashboard" "operations" {
  dashboard_name = "${var.project}-operations"
  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["AWS/ECS", "CPUUtilization", "ClusterName", var.cluster_name],
            ["AWS/ECS", "MemoryUtilization", "ClusterName", var.cluster_name]
          ]
          period = 300
          stat   = "Average"
          region = var.region
          title  = "ECS Cluster - CPU & Memory"
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", var.alb_arn_suffix],
            ["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count", "LoadBalancer", var.alb_arn_suffix]
          ]
          period = 60
          stat   = "Sum"
          region = var.region
          title  = "ALB - Latency & Errors"
        }
      }
    ]
  })
}
```

#### Alarmes Essenciais

```hcl
# Alarme de alta latência
resource "aws_cloudwatch_metric_alarm" "high_latency" {
  alarm_name          = "${var.project}-high-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "TargetResponseTime"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "p99"
  threshold           = 2.0 # 2 segundos
  alarm_description   = "P99 latency acima de 2s por 3 minutos"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    LoadBalancer = aws_lb.main.arn_suffix
  }
}

# Alarme de erro rate
resource "aws_cloudwatch_metric_alarm" "error_rate" {
  alarm_name          = "${var.project}-error-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  threshold           = 5 # 5%

  metric_query {
    id          = "error_rate"
    expression  = "(errors / requests) * 100"
    label       = "Error Rate %"
    return_data = true
  }

  metric_query {
    id = "errors"
    metric {
      metric_name = "HTTPCode_Target_5XX_Count"
      namespace   = "AWS/ApplicationELB"
      period      = 60
      stat        = "Sum"
      dimensions  = { LoadBalancer = aws_lb.main.arn_suffix }
    }
  }

  metric_query {
    id = "requests"
    metric {
      metric_name = "RequestCount"
      namespace   = "AWS/ApplicationELB"
      period      = 60
      stat        = "Sum"
      dimensions  = { LoadBalancer = aws_lb.main.arn_suffix }
    }
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

#### Evolução — Post-Mortem Template

```markdown
## Post-Mortem: [Título do Incidente]

**Data**: YYYY-MM-DD
**Severidade**: SEV-1 / SEV-2 / SEV-3
**Duração**: X minutos/horas
**Impacto**: Descrição do impacto ao usuário

### Timeline
| Hora | Evento |
|------|--------|
| HH:MM | Alarme disparado |
| HH:MM | Equipe acionada |
| HH:MM | Causa identificada |
| HH:MM | Mitigação aplicada |
| HH:MM | Serviço restaurado |

### Causa Raiz
Descrição técnica detalhada.

### Ações Corretivas
| Ação | Responsável | Prazo | Status |
|------|-------------|-------|--------|
| ... | ... | ... | ... |

### Lições Aprendidas
- O que funcionou bem
- O que pode melhorar
```

---

## 2. Segurança (Security)

### Princípios de Design

- **Implementar uma base de identidade forte** — Princípio do menor privilégio
- **Rastreabilidade** — Monitorar e auditar todas as ações
- **Segurança em todas as camadas** — Defense in depth
- **Automatizar boas práticas de segurança** — Segurança como código
- **Proteger dados em trânsito e em repouso** — Criptografia sempre
- **Manter pessoas longe dos dados** — Eliminar acesso manual
- **Preparar-se para eventos de segurança** — Incident response automatizado

### IAM — Princípio do Menor Privilégio

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSpecificS3Actions",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/app-data/*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "sa-east-1"
        },
        "IpAddress": {
          "aws:SourceIp": "10.0.0.0/8"
        }
      }
    }
  ]
}
```

#### IAM Roles para Serviços (nunca credenciais hardcoded)

```hcl
# Role para ECS Task
resource "aws_iam_role" "ecs_task_role" {
  name = "${var.project}-ecs-task-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

# Política mínima necessária
resource "aws_iam_role_policy" "ecs_task_policy" {
  name = "${var.project}-ecs-task-policy"
  role = aws_iam_role.ecs_task_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ReadSecrets"
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = [
          "arn:aws:secretsmanager:${var.region}:${var.account_id}:secret:${var.project}/*"
        ]
      },
      {
        Sid    = "WriteToSpecificBucket"
        Effect = "Allow"
        Action = [
          "s3:PutObject",
          "s3:GetObject"
        ]
        Resource = [
          "${aws_s3_bucket.app_data.arn}/*"
        ]
      }
    ]
  })
}
```

### Criptografia

```hcl
# KMS Key com rotação automática
resource "aws_kms_key" "main" {
  description             = "KMS key for ${var.project}"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  rotation_period_in_days = 90

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "KeyAdministration"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.account_id}:role/admin"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "KeyUsage"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.ecs_task_role.arn
        }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_kms_alias" "main" {
  name          = "alias/${var.project}"
  target_key_id = aws_kms_key.main.key_id
}

# S3 Bucket com criptografia obrigatória
resource "aws_s3_bucket" "data" {
  bucket = "${var.project}-data-${var.environment}"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.main.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "data" {
  bucket                  = aws_s3_bucket.data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Detecção e Resposta

```hcl
# GuardDuty habilitado
resource "aws_guardduty_detector" "main" {
  enable = true

  datasources {
    s3_logs {
      enable = true
    }
    kubernetes {
      audit_logs {
        enable = true
      }
    }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes {
          enable = true
        }
      }
    }
  }
}

# Security Hub
resource "aws_securityhub_account" "main" {}

resource "aws_securityhub_standards_subscription" "aws_foundational" {
  standards_arn = "arn:aws:securityhub:${var.region}::standards/aws-foundational-security-best-practices/v/1.0.0"
  depends_on    = [aws_securityhub_account.main]
}

resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.4.0"
  depends_on    = [aws_securityhub_account.main]
}
```

---

## 3. Confiabilidade (Reliability)

### Princípios de Design

- **Recuperar-se automaticamente de falhas** — Auto healing
- **Testar procedimentos de recuperação** — Chaos Engineering
- **Escalar horizontalmente** — Distribuir carga
- **Parar de adivinhar capacidade** — Auto Scaling
- **Gerenciar mudanças com automação** — CI/CD

### Multi-AZ Architecture

```hcl
# VPC Multi-AZ
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = { Name = "${var.project}-vpc" }
}

# Subnets em múltiplas AZs
resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.project}-private-${var.availability_zones[count.index]}"
    Tier = "private"
  }
}

resource "aws_subnet" "public" {
  count                   = length(var.availability_zones)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 100)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project}-public-${var.availability_zones[count.index]}"
    Tier = "public"
  }
}

# RDS Multi-AZ
resource "aws_db_instance" "main" {
  identifier     = "${var.project}-db"
  engine         = "postgres"
  engine_version = "16.4"
  instance_class = "db.r6g.large"

  multi_az               = true
  storage_encrypted      = true
  kms_key_id             = aws_kms_key.main.arn
  deletion_protection    = true
  backup_retention_period = 7
  
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]

  performance_insights_enabled    = true
  monitoring_interval             = 60
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
}
```

### Auto Scaling

```hcl
# ECS Auto Scaling
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

# Scale by Request Count
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
    target_value       = 1000
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

### Health Checks e Circuit Breaker

```hcl
# ECS Service com circuit breaker
resource "aws_ecs_service" "main" {
  name            = "${var.project}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.main.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 100
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.main.arn
    container_name   = var.container_name
    container_port   = var.container_port
  }

  network_configuration {
    security_groups  = [aws_security_group.ecs.id]
    subnets          = aws_subnet.private[*].id
    assign_public_ip = false
  }
}

# Health check no ALB
resource "aws_lb_target_group" "main" {
  name        = "${var.project}-tg"
  port        = var.container_port
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 15
    matcher             = "200"
  }

  deregistration_delay = 30
}
```

---

## 4. Eficiência de Performance (Performance Efficiency)

### Princípios de Design

- **Democratizar tecnologias avançadas** — Usar managed services
- **Ter alcance global em minutos** — Multi-region, CloudFront
- **Usar arquiteturas serverless** — Eliminar gerenciamento de servidores
- **Experimentar mais frequentemente** — A/B testing
- **Ter empatia mecânica** — Entender como os serviços funcionam

### Caching Strategy

```hcl
# ElastiCache Redis
resource "aws_elasticache_replication_group" "main" {
  replication_group_id = "${var.project}-redis"
  description          = "Redis cluster for ${var.project}"
  
  node_type            = "cache.r6g.large"
  num_cache_clusters   = 2  # Primary + Replica
  
  engine               = "redis"
  engine_version       = "7.1"
  port                 = 6379
  parameter_group_name = "default.redis7"
  
  automatic_failover_enabled = true
  multi_az_enabled           = true
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token
  
  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]

  snapshot_retention_limit = 7
  snapshot_window          = "03:00-05:00"
  maintenance_window       = "sun:05:00-sun:07:00"
}

# CloudFront para conteúdo estático
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  price_class         = "PriceClass_200"

  origin {
    domain_name              = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id                = "s3-static"
    origin_access_control_id = aws_cloudfront_origin_access_control.main.id
  }

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-static"

    cache_policy_id          = aws_cloudfront_cache_policy.optimized.id
    origin_request_policy_id = aws_cloudfront_origin_request_policy.cors.id

    viewer_protocol_policy = "redirect-to-https"
    compress               = true
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
}
```

### Seleção de Compute

```
┌─────────────────────────────────────────────────────────────────┐
│                    Árvore de Decisão: Compute                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Execução < 15 min e event-driven?                             │
│  ├── SIM → AWS Lambda                                          │
│  │         (Pague por invocação, zero gerenciamento)           │
│  │                                                              │
│  └── NÃO → Precisa de containers?                              │
│       ├── SIM → Quer gerenciar cluster Kubernetes?              │
│       │    ├── SIM → Amazon EKS                                 │
│       │    │         (Controle total do K8s)                    │
│       │    └── NÃO → Precisa de orquestração avançada?          │
│       │         ├── SIM → Amazon ECS on Fargate                 │
│       │         │         (Serverless containers)               │
│       │         └── NÃO → AWS App Runner                        │
│       │                   (Deploy direto do código/imagem)      │
│       │                                                         │
│       └── NÃO → Precisa de controle do OS?                     │
│            ├── SIM → Amazon EC2                                 │
│            │         (Graviton para melhor preço/performance)   │
│            └── NÃO → AWS App Runner ou Elastic Beanstalk       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Otimização de Custos (Cost Optimization)

### Princípios de Design

- **Implementar gestão financeira na nuvem** — FinOps
- **Adotar modelo de consumo** — Pagar apenas pelo que usa
- **Medir a eficiência geral** — Custo por transação
- **Parar de gastar em trabalho pesado indiferenciado** — Managed services
- **Analisar e atribuir gastos** — Tags e Cost Allocation

### Estratégias de Economia

```
┌───────────────────────────────────────────────────────────┐
│            Modelo de Pricing — Quando usar                │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  On-Demand                                                │
│  └── Workloads imprevisíveis, spikes, experimentação     │
│                                                           │
│  Reserved Instances / Savings Plans (até 72% desconto)    │
│  └── Workloads estáveis e previsíveis (1 ou 3 anos)     │
│       Compute SP: qualquer instância/região              │
│       EC2 Instance SP: instância específica              │
│                                                           │
│  Spot Instances (até 90% desconto)                        │
│  └── Workloads tolerantes a interrupção                  │
│       Batch processing, CI/CD workers, ML training        │
│                                                           │
│  Graviton (até 40% melhor preço/performance)             │
│  └── Qualquer workload Linux (recomendado como default)  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### Budget Alerts

```hcl
resource "aws_budgets_budget" "monthly" {
  name         = "${var.project}-monthly-budget"
  budget_type  = "COST"
  limit_amount = var.monthly_budget_limit
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  cost_filter {
    name   = "TagKeyValue"
    values = ["user:Project$${var.project_name}"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = var.budget_alert_emails
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = var.budget_alert_emails
    subscriber_sns_topic_arns  = [aws_sns_topic.billing.arn]
  }
}
```

### Right-Sizing Checklist

- [ ] Analisar CPU/Memory utilization via CloudWatch (mín. 2 semanas)
- [ ] Considerar Graviton (arm64) como tipo de instância default
- [ ] Usar AWS Compute Optimizer para recomendações
- [ ] Implementar Auto Scaling para escalar para baixo em baixa demanda
- [ ] Avaliar Savings Plans para workloads estáveis
- [ ] Usar S3 Intelligent-Tiering para storage com padrões de acesso variáveis
- [ ] Implementar lifecycle policies para S3 e ECR
- [ ] Desligar recursos não-produção fora do horário comercial

---

## 6. Sustentabilidade (Sustainability)

### Princípios de Design

- **Entender seu impacto** — Medir consumo de recursos
- **Estabelecer metas de sustentabilidade** — KPIs de eficiência
- **Maximizar utilização** — Right-sizing e Auto Scaling
- **Adotar ofertas gerenciadas mais eficientes** — Serverless, Graviton
- **Reduzir impacto downstream** — Minimizar dados trafegados

### Boas Práticas

```hcl
# Usar Graviton (ARM) — mais eficiente energeticamente
resource "aws_ecs_task_definition" "sustainable" {
  family                   = "${var.project}-task"
  requires_compatibilities = ["FARGATE"]
  
  # Graviton: melhor preço/performance e menor consumo energético
  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture        = "ARM64"
  }

  cpu    = 512
  memory = 1024
  # ...
}

# S3 Intelligent-Tiering — mover dados automaticamente
resource "aws_s3_bucket_intelligent_tiering_configuration" "main" {
  bucket = aws_s3_bucket.data.id
  name   = "EntireBucket"

  tiering {
    access_tier = "DEEP_ARCHIVE_ACCESS"
    days        = 180
  }

  tiering {
    access_tier = "ARCHIVE_ACCESS"
    days        = 90
  }
}

# ECR Lifecycle — limpar imagens antigas
resource "aws_ecr_lifecycle_policy" "cleanup" {
  repository = aws_ecr_repository.main.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 10 images"
        selection = {
          tagStatus   = "any"
          countType   = "imageCountMoreThan"
          countNumber = 10
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

## Well-Architected Review Checklist

### Pré-Review

- [ ] Diagrama de arquitetura atualizado
- [ ] Lista de serviços AWS utilizados
- [ ] Métricas de baseline (latência, throughput, error rate)
- [ ] Custos atuais mapeados por serviço

### Durante o Review

| Pilar | Pergunta-Chave | Status |
|-------|----------------|--------|
| **Ops Excellence** | Infra é 100% IaC? | ☐ |
| **Ops Excellence** | Alarmes cobrem cenários críticos? | ☐ |
| **Ops Excellence** | Runbooks documentados e testados? | ☐ |
| **Segurança** | Princípio do menor privilégio aplicado? | ☐ |
| **Segurança** | Dados criptografados em repouso e trânsito? | ☐ |
| **Segurança** | Logs de auditoria habilitados (CloudTrail)? | ☐ |
| **Segurança** | Secrets gerenciados (Secrets Manager/SSM)? | ☐ |
| **Confiabilidade** | Arquitetura Multi-AZ? | ☐ |
| **Confiabilidade** | Auto Scaling configurado? | ☐ |
| **Confiabilidade** | Backups automáticos e testados? | ☐ |
| **Confiabilidade** | Health checks configurados? | ☐ |
| **Performance** | Caching implementado onde necessário? | ☐ |
| **Performance** | CDN para conteúdo estático? | ☐ |
| **Performance** | Tipo de instância adequado ao workload? | ☐ |
| **Custos** | Tags de cost allocation em todos os recursos? | ☐ |
| **Custos** | Budget alerts configurados? | ☐ |
| **Custos** | Savings Plans avaliados? | ☐ |
| **Sustentabilidade** | Graviton utilizado onde possível? | ☐ |
| **Sustentabilidade** | Lifecycle policies configurados? | ☐ |
| **Sustentabilidade** | Auto Scaling permite scale-down? | ☐ |

---

## Referências

- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/)
- [AWS Well-Architected Tool](https://aws.amazon.com/well-architected-tool/)
- [AWS Well-Architected Labs](https://www.wellarchitectedlabs.com/)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)
