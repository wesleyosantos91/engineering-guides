# AWS Cost Optimization — Guia Prático

Guia de otimização de custos na AWS: FinOps, modelos de preço, right-sizing e automação.

---

## FinOps Maturity Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    FinOps Maturity                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CRAWL (Nível 1)                                                │
│  • Tagging básico (project, environment, owner)                │
│  • Cost Explorer habilitado                                     │
│  • Budgets com alertas                                          │
│  • Billing centralizado (Organizations)                        │
│                                                                 │
│  WALK (Nível 2)                                                │
│  • Tagging enforcement (SCP)                                   │
│  • Savings Plans / Reserved Instances                          │
│  • Right-sizing automático (Compute Optimizer)                 │
│  • Cost anomaly detection                                       │
│  • Showback/Chargeback por time/produto                        │
│                                                                 │
│  RUN (Nível 3)                                                  │
│  • Automação de right-sizing                                   │
│  • Spot Instances para workloads tolerantes                    │
│  • Graviton migration                                           │
│  • S3 Intelligent-Tiering automático                           │
│  • Unit economics (custo por transação/cliente)                │
│  • FinOps dashboards em tempo real                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tagging Strategy para Cost Allocation

### Tags Obrigatórias

```hcl
# Módulo de tags padrão
locals {
  required_tags = {
    Project     = var.project             # Nome do projeto/produto
    Environment = var.environment          # prod, staging, dev
    Owner       = var.owner               # Time responsável
    CostCenter  = var.cost_center         # Centro de custo
    ManagedBy   = "terraform"             # Gerenciamento
    Repository  = var.repository          # Repositório no Git
  }
}

# Variáveis
variable "project" {
  type        = string
  description = "Nome do projeto para identificação de custos"
}

variable "environment" {
  type        = string
  description = "Ambiente (prod, staging, dev)"
  validation {
    condition     = contains(["prod", "staging", "dev"], var.environment)
    error_message = "Environment deve ser prod, staging ou dev."
  }
}

variable "cost_center" {
  type        = string
  description = "Centro de custo para chargeback"
}
```

### SCP para Enforce Tagging

```hcl
# Impedir criação de recursos sem tags obrigatórias
resource "aws_organizations_policy" "require_tags" {
  name        = "require-cost-tags"
  description = "Exige tags de custo em recursos"
  type        = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyCreateWithoutTags"
        Effect    = "Deny"
        Action    = [
          "ec2:RunInstances",
          "ecs:CreateService",
          "rds:CreateDBInstance",
          "lambda:CreateFunction",
          "elasticloadbalancing:CreateLoadBalancer",
          "s3:CreateBucket"
        ]
        Resource  = "*"
        Condition = {
          "Null" = {
            "aws:RequestTag/Project"     = "true"
            "aws:RequestTag/Environment" = "true"
            "aws:RequestTag/CostCenter"  = "true"
          }
        }
      }
    ]
  })
}
```

---

## Modelos de Preço — Árvore de Decisão

```
Workload é previsível e constante?
├── SIM → Roda 24/7 por 1+ ano?
│   ├── SIM → Savings Plans (Compute ou EC2 Instance)
│   │   ├── Flexibilidade entre famílias? → Compute Savings Plans
│   │   └── Família fixa, máximo desconto? → EC2 Instance Savings Plans
│   └── NÃO → On-Demand (considerar Savings Plans parcial)
│
├── NÃO → Tolerante a interrupções?
│   ├── SIM → Spot Instances (até 90% desconto)
│   │   ├── Batch processing → Spot Fleet
│   │   ├── CI/CD → Spot com fallback On-Demand
│   │   └── ECS → Capacity Provider FARGATE_SPOT
│   └── NÃO → On-Demand
│
└── NÃO SEI → Analyze com Cost Explorer + Compute Optimizer
```

### Savings Plans

```hcl
# Budget para monitorar commitment utilization
resource "aws_budgets_budget" "savings_plans_utilization" {
  name         = "savings-plans-utilization"
  budget_type  = "SAVINGS_PLANS_UTILIZATION"
  limit_amount = "100"  # target: 100% utilization
  limit_unit   = "PERCENTAGE"
  time_unit    = "MONTHLY"

  cost_filter {
    name   = "Service"
    values = ["Amazon Elastic Compute Cloud - Compute"]
  }

  notification {
    comparison_operator       = "LESS_THAN"
    threshold                 = 80
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_email_addresses = var.finops_emails
  }
}
```

### Spot Instances (ECS)

```hcl
# Capacity providers com Spot para workloads não-críticos
resource "aws_ecs_cluster_capacity_providers" "mixed" {
  cluster_name = aws_ecs_cluster.main.name

  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
    base              = 2  # Mínimo 2 tasks em Fargate (estável)
  }

  default_capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 4  # 80% das tasks adicionais em Spot
  }
}

# Para workers/batch processing: 100% Spot
resource "aws_ecs_service" "worker" {
  name            = "${var.project}-worker"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.worker.arn
  desired_count   = var.worker_count

  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 1
  }
}
```

---

## Right-Sizing

### Compute Optimizer

```hcl
# Habilitar Compute Optimizer
resource "aws_computeoptimizer_enrollment_status" "main" {
  status = "Active"
  include_member_accounts = true
}
```

### Guia de Right-Sizing por Serviço

| Serviço | Métrica-Chave | Sinal de Over-Provisioned | Ação |
|---------|---------------|---------------------------|------|
| EC2/ECS | CPU < 40% avg | Consistentemente baixo | Reduzir vCPU ou mudar família |
| EC2/ECS | Memory < 40% avg | Subutilizado | Reduzir memory allocation |
| RDS | CPU < 25% | Instance muito grande | Downgrade class (db.r6g → db.r5) |
| RDS | FreeableMemory > 80% | Muito RAM | Reduzir instance class |
| Lambda | Duration < 10% timeout | Timeout alto demais | Reduzir timeout |
| Lambda | Memory high, CPU low | Test Power Tuning | Reduzir memory |
| DynamoDB | ConsumedRCU < 20% | Provisioned muito alto | Mudar para On-Demand ou reduzir |
| ElastiCache | Memory < 30% | Node muito grande | Downgrade node type |

### Alarmes de Waste Detection

```hcl
# Alarme: ECS CPU consistentemente baixo (over-provisioned)
resource "aws_cloudwatch_metric_alarm" "ecs_cpu_waste" {
  alarm_name          = "${var.project}-ecs-cpu-waste"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 168  # 7 dias de periodos de 1h
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 3600  # 1 hora
  statistic           = "Average"
  threshold           = 20    # Abaixo de 20% por 7 dias

  dimensions = {
    ClusterName = var.cluster_name
    ServiceName = var.service_name
  }

  alarm_actions = [aws_sns_topic.alerts["info"].arn]
}

# Alarme: RDS CPU consistentemente baixo
resource "aws_cloudwatch_metric_alarm" "rds_cpu_waste" {
  alarm_name          = "${var.project}-rds-cpu-waste"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 168
  metric_name         = "CPUUtilization"
  namespace           = "AWS/RDS"
  period              = 3600
  statistic           = "Average"
  threshold           = 15

  dimensions = {
    DBInstanceIdentifier = var.db_identifier
  }

  alarm_actions = [aws_sns_topic.alerts["info"].arn]
}
```

---

## Storage Optimization

### S3 Lifecycle & Storage Classes

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "optimized" {
  bucket = aws_s3_bucket.main.id

  # Regra para logs e dados temporários
  rule {
    id     = "logs-lifecycle"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER_IR"  # Instant Retrieval
    }

    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"
    }

    expiration {
      days = 730  # 2 anos
    }
  }

  # Regra para dados com acesso imprevisível
  rule {
    id     = "intelligent-tiering"
    status = "Enabled"

    filter {
      prefix = "data/"
    }

    transition {
      days          = 0
      storage_class = "INTELLIGENT_TIERING"
    }
  }

  # Cleanup de multipart uploads incompletos
  rule {
    id     = "abort-multipart"
    status = "Enabled"

    filter {}

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }

  # Cleanup de versões anteriores
  rule {
    id     = "version-cleanup"
    status = "Enabled"

    filter {}

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }

    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }
}

# Intelligent Tiering configuration
resource "aws_s3_bucket_intelligent_tiering_configuration" "main" {
  bucket = aws_s3_bucket.main.id
  name   = "full-optimization"

  tiering {
    access_tier = "ARCHIVE_ACCESS"
    days        = 90
  }

  tiering {
    access_tier = "DEEP_ARCHIVE_ACCESS"
    days        = 180
  }
}
```

### EBS Optimization

```hcl
# Identificar volumes não utilizados (snapshot e deletar)
# Use AWS Config Rule para detectar automaticamente

resource "aws_config_config_rule" "unattached_ebs" {
  name = "ebs-unattached-volumes"

  source {
    owner             = "AWS"
    source_identifier = "EC2_VOLUME_INUSE_CHECK"
  }
}

# Migrar gp2 → gp3 (20% mais barato, melhor performance)
# Usar AWS Config para detectar volumes gp2
resource "aws_config_config_rule" "ebs_gp3_check" {
  name = "ebs-gp3-migration"

  source {
    owner             = "AWS"
    source_identifier = "EC2_VOLUME_INUSE_CHECK"
  }
}
```

---

## Graviton (ARM64) Migration

### Benefício: 20-40% melhor price-performance

```hcl
# EC2: migrar para instâncias Graviton
# c5.xlarge → c7g.xlarge (Graviton3)
# m5.large  → m7g.large (Graviton3)

# RDS: migrar para Graviton
resource "aws_db_instance" "graviton" {
  instance_class = "db.r7g.large"  # Graviton3 (antes: db.r6i.large)
  # ...
}

# ElastiCache: migrar para Graviton
resource "aws_elasticache_cluster" "graviton" {
  node_type = "cache.r7g.large"  # Graviton3 (antes: cache.r6g.large)
  # ...
}

# ECS Fargate: ARM64
resource "aws_ecs_task_definition" "graviton" {
  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture        = "ARM64"  # Fargate Graviton: ~20% mais barato
  }
  # ...
}

# Lambda: ARM64
resource "aws_lambda_function" "graviton" {
  architectures = ["arm64"]  # ~34% melhor price-performance
  # ...
}
```

---

## Budgets e Alertas

```hcl
# Budget mensal por projeto
resource "aws_budgets_budget" "monthly" {
  name         = "${var.project}-monthly-budget"
  budget_type  = "COST"
  limit_amount = var.monthly_budget
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  cost_filter {
    name   = "TagKeyValue"
    values = ["user:Project$${var.project}"]
  }

  # Alerta em 50%
  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 50
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_email_addresses = var.finops_emails
  }

  # Alerta em 80%
  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 80
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_email_addresses = var.finops_emails
  }

  # Alerta em 100%
  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 100
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_email_addresses = concat(var.finops_emails, var.engineering_leads)
  }

  # Forecast: alerta se projeção > 120%
  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 120
    threshold_type            = "PERCENTAGE"
    notification_type         = "FORECASTED"
    subscriber_email_addresses = concat(var.finops_emails, var.engineering_leads)
  }
}

# Cost Anomaly Detection
resource "aws_ce_anomaly_monitor" "main" {
  name              = "${var.project}-anomaly-monitor"
  monitor_type      = "DIMENSIONAL"
  monitor_dimension = "SERVICE"
}

resource "aws_ce_anomaly_subscription" "main" {
  name = "${var.project}-anomaly-alerts"

  monitor_arn_list = [aws_ce_anomaly_monitor.main.arn]

  subscriber {
    type    = "EMAIL"
    address = var.finops_email
  }

  subscriber {
    type    = "SNS"
    address = aws_sns_topic.alerts["medium"].arn
  }

  threshold_expression {
    dimension {
      key           = "ANOMALY_TOTAL_IMPACT_ABSOLUTE"
      values        = ["100"]  # Alertar se impacto > $100
      match_options = ["GREATER_THAN_OR_EQUAL"]
    }
  }

  frequency = "DAILY"
}
```

---

## Ambientes Não-Produção (Dev/Staging)

### Reduzir custos em 60-80%

```hcl
locals {
  is_prod = var.environment == "prod"
}

# RDS: Single-AZ em dev, Multi-AZ em prod
resource "aws_db_instance" "main" {
  multi_az       = local.is_prod
  instance_class = local.is_prod ? "db.r7g.xlarge" : "db.t4g.medium"
  
  # Desligar dev à noite
  # (usar EventBridge + Lambda para stop/start)
}

# ECS: menos tasks em dev
resource "aws_ecs_service" "main" {
  desired_count = local.is_prod ? 3 : 1
}

# ElastiCache: menor em dev
resource "aws_elasticache_cluster" "main" {
  node_type       = local.is_prod ? "cache.r7g.large" : "cache.t4g.micro"
  num_cache_nodes = local.is_prod ? 3 : 1
}

# NAT Gateway: usar NAT Instance em dev
# NAT Gateway: ~$32/mês
# NAT Instance (t4g.nano): ~$3/mês
```

### Scheduler para Dev/Staging

```hcl
# Desligar recursos não-prod fora do horário comercial
resource "aws_scheduler_schedule" "stop_dev" {
  count = local.is_prod ? 0 : 1

  name       = "${var.project}-stop-dev"
  group_name = "cost-optimization"

  schedule_expression          = "cron(0 22 ? * MON-FRI *)"  # 22:00 UTC weekdays
  schedule_expression_timezone = "America/Sao_Paulo"

  flexible_time_window {
    mode = "OFF"
  }

  target {
    arn      = "arn:aws:scheduler:::aws-sdk:ecs:updateService"
    role_arn = aws_iam_role.scheduler.arn

    input = jsonencode({
      Cluster      = var.cluster_name
      Service      = var.service_name
      DesiredCount = 0
    })
  }
}

resource "aws_scheduler_schedule" "start_dev" {
  count = local.is_prod ? 0 : 1

  name       = "${var.project}-start-dev"
  group_name = "cost-optimization"

  schedule_expression          = "cron(0 8 ? * MON-FRI *)"  # 08:00 UTC weekdays
  schedule_expression_timezone = "America/Sao_Paulo"

  flexible_time_window {
    mode = "OFF"
  }

  target {
    arn      = "arn:aws:scheduler:::aws-sdk:ecs:updateService"
    role_arn = aws_iam_role.scheduler.arn

    input = jsonencode({
      Cluster      = var.cluster_name
      Service      = var.service_name
      DesiredCount = var.desired_count
    })
  }
}
```

---

## Cost Optimization Checklist

### Quick Wins (Impacto imediato)

- [ ] Habilitar AWS Cost Explorer
- [ ] Criar budgets com alertas (50%, 80%, 100%)
- [ ] Habilitar Cost Anomaly Detection
- [ ] Deletar recursos não utilizados (EBS, EIP, snapshots antigos)
- [ ] Migrar EBS gp2 → gp3
- [ ] Habilitar S3 Intelligent-Tiering
- [ ] Configurar S3 lifecycle policies

### Médio Prazo (1-3 meses)

- [ ] Implementar tagging com enforcement (SCP)
- [ ] Analisar e comprar Savings Plans
- [ ] Right-sizing com Compute Optimizer
- [ ] Migrar para Graviton (ARM64)
- [ ] Desligar dev/staging fora do horário
- [ ] Consolidar ambientes dev/staging

### Longo Prazo (3-6 meses)

- [ ] Spot Instances para workloads tolerantes
- [ ] Serverless para workloads variáveis (Lambda, Fargate)
- [ ] Unit economics (custo por transação/cliente)
- [ ] FinOps dashboard com showback/chargeback
- [ ] Revisão trimestral de Savings Plans
- [ ] Automação de right-sizing contínuo

---

## Referências

- [AWS Cost Optimization Pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/)
- [AWS Compute Optimizer](https://docs.aws.amazon.com/compute-optimizer/)
- [FinOps Foundation](https://www.finops.org/)
- [AWS Pricing Calculator](https://calculator.aws/)
- [Graviton Transition Guide](https://github.com/aws/aws-graviton-getting-started)
