# AWS Disaster Recovery — Boas Práticas

Guia de Disaster Recovery na AWS: estratégias por RPO/RTO, AWS Backup, replicação cross-region e runbooks.

---

## Estratégias de DR

```
┌────────────────────────────────────────────────────────────────────────┐
│            DR Strategies — Custo vs Velocidade de Recovery            │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Custo ↑                                          Recovery Time ↓    │
│                                                                        │
│  ████████  Multi-Site Active/Active   RPO: ~0    RTO: ~0             │
│  ██████    Warm Standby               RPO: seg   RTO: min            │
│  ████      Pilot Light                RPO: min   RTO: 10-30min      │
│  ██        Backup & Restore           RPO: horas RTO: horas         │
│                                                                        │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  RPO = Recovery Point Objective (quanto dado posso perder?)           │
│  RTO = Recovery Time Objective (quanto tempo posso ficar fora?)       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Quando usar cada estratégia

| Estratégia | RPO | RTO | Custo | Use Case |
|-----------|-----|-----|-------|----------|
| **Backup & Restore** | Horas | Horas | $ | Dev/staging, dados históricos |
| **Pilot Light** | Minutos | 10-30 min | $$ | Apps com tolerância a downtime curto |
| **Warm Standby** | Segundos | Minutos | $$$ | Apps de negócio críticas |
| **Multi-Site Active** | ~0 | ~0 | $$$$ | Sistemas financeiros, saúde, e-commerce tier-1 |

---

## Backup & Restore

### AWS Backup — Configuração Centralizada

```hcl
# Backup Vault com criptografia
resource "aws_backup_vault" "main" {
  name        = "${var.project}-backup-vault"
  kms_key_arn = aws_kms_key.backup.arn

  tags = var.tags
}

# Vault Lock — imutabilidade (proteção contra ransomware)
resource "aws_backup_vault_lock_configuration" "main" {
  backup_vault_name   = aws_backup_vault.main.name
  min_retention_days  = 7
  max_retention_days  = 365
  changeable_for_days = 3  # Grace period antes de lock permanente
}

# Backup Plan
resource "aws_backup_plan" "main" {
  name = "${var.project}-backup-plan"

  # Backup diário — retenção 35 dias
  rule {
    rule_name         = "daily"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 3 * * ? *)"  # 3:00 UTC

    lifecycle {
      cold_storage_after = 14
      delete_after       = 35
    }

    # Cross-region copy para DR
    copy_action {
      destination_vault_arn = aws_backup_vault.dr_region.arn

      lifecycle {
        delete_after = 35
      }
    }
  }

  # Backup semanal — retenção 90 dias
  rule {
    rule_name         = "weekly"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 3 ? * SUN *)"  # Domingo 3:00 UTC

    lifecycle {
      cold_storage_after = 30
      delete_after       = 90
    }

    copy_action {
      destination_vault_arn = aws_backup_vault.dr_region.arn

      lifecycle {
        delete_after = 90
      }
    }
  }

  # Backup mensal — retenção 365 dias
  rule {
    rule_name         = "monthly"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 3 1 * ? *)"  # Dia 1 de cada mês

    lifecycle {
      cold_storage_after = 90
      delete_after       = 365
    }

    copy_action {
      destination_vault_arn = aws_backup_vault.dr_region.arn

      lifecycle {
        delete_after = 365
      }
    }
  }
}

# Seleção de recursos (tag-based)
resource "aws_backup_selection" "main" {
  name         = "${var.project}-backup-selection"
  iam_role_arn = aws_iam_role.backup.arn
  plan_id      = aws_backup_plan.main.id

  selection_tag {
    type  = "STRINGEQUALS"
    key   = "Backup"
    value = "true"
  }
}

# IAM Role para AWS Backup
resource "aws_iam_role" "backup" {
  name = "${var.project}-backup-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "backup.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "backup" {
  role       = aws_iam_role.backup.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup"
}

resource "aws_iam_role_policy_attachment" "restore" {
  role       = aws_iam_role.backup.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores"
}
```

### Backup Vault na Região DR

```hcl
provider "aws" {
  alias  = "dr"
  region = var.dr_region  # ex: us-east-1
}

resource "aws_backup_vault" "dr_region" {
  provider    = aws.dr
  name        = "${var.project}-backup-vault-dr"
  kms_key_arn = aws_kms_key.backup_dr.arn
}
```

---

## Pilot Light

Infraestrutura mínima na região DR, escalada em caso de desastre.

```hcl
# Região DR: apenas banco de dados replicado
# A infra de compute é provisionada sob demanda

# Aurora Global Database (replicação contínua)
resource "aws_rds_global_cluster" "main" {
  global_cluster_identifier = "${var.project}-global"
  engine                    = "aurora-postgresql"
  engine_version            = "16.4"
  storage_encrypted         = true
}

# Cluster primário (sa-east-1)
resource "aws_rds_cluster" "primary" {
  cluster_identifier        = "${var.project}-primary"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  engine                    = "aurora-postgresql"
  engine_version            = "16.4"
  master_username           = "admin"
  manage_master_user_password = true
  db_subnet_group_name      = aws_db_subnet_group.primary.name
  vpc_security_group_ids    = [aws_security_group.db_primary.id]
}

# Cluster secundário (us-east-1) — read replica
resource "aws_rds_cluster" "secondary" {
  provider                  = aws.dr
  cluster_identifier        = "${var.project}-secondary"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  engine                    = "aurora-postgresql"
  engine_version            = "16.4"
  db_subnet_group_name      = aws_db_subnet_group.secondary.name
  vpc_security_group_ids    = [aws_security_group.db_secondary.id]

  # Pilot Light: 1 instância mínima (escala durante failover)
  depends_on = [aws_rds_cluster.primary]
}

resource "aws_rds_cluster_instance" "secondary" {
  provider           = aws.dr
  count              = 1  # Mínimo para pilot light
  identifier         = "${var.project}-secondary-${count.index + 1}"
  cluster_identifier = aws_rds_cluster.secondary.id
  instance_class     = "db.r6g.large"  # Pode ser menor que produção
  engine             = "aurora-postgresql"
}
```

### DNS Failover (Route 53)

```hcl
# Health Check na região primária
resource "aws_route53_health_check" "primary" {
  fqdn              = var.primary_alb_dns
  port               = 443
  type               = "HTTPS"
  resource_path      = "/health"
  failure_threshold  = 3
  request_interval   = 10

  tags = {
    Name = "${var.project}-primary-health"
  }
}

# DNS Failover: primário → DR
resource "aws_route53_record" "app" {
  zone_id = var.hosted_zone_id
  name    = "app.${var.domain}"
  type    = "A"

  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier = "primary"
  health_check_id = aws_route53_health_check.primary.id

  alias {
    name                   = var.primary_alb_dns
    zone_id                = var.primary_alb_zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "app_dr" {
  zone_id = var.hosted_zone_id
  name    = "app.${var.domain}"
  type    = "A"

  failover_routing_policy {
    type = "SECONDARY"
  }

  set_identifier = "secondary"

  alias {
    name                   = var.dr_alb_dns
    zone_id                = var.dr_alb_zone_id
    evaluate_target_health = true
  }
}
```

---

## Warm Standby

Infraestrutura completa na região DR, mas em escala reduzida.

```hcl
# Região DR: mesma infra, menor escala
# ECS Service na região DR (25% da capacidade de produção)
resource "aws_ecs_service" "dr" {
  provider        = aws.dr
  name            = "${var.project}-service"
  cluster         = aws_ecs_cluster.dr.id
  task_definition = aws_ecs_task_definition.dr.arn
  desired_count   = max(1, floor(var.prod_desired_count * 0.25))  # 25% de prod

  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.dr.arn
    container_name   = "app"
    container_port   = 8080
  }
}

# Auto Scaling na região DR (escala para 100% durante failover)
resource "aws_appautoscaling_target" "dr" {
  provider           = aws.dr
  max_capacity       = var.prod_desired_count * 2  # Pode escalar até 2x prod
  min_capacity       = max(1, floor(var.prod_desired_count * 0.25))
  resource_id        = "service/${aws_ecs_cluster.dr.name}/${aws_ecs_service.dr.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "dr_cpu" {
  provider           = aws.dr
  name               = "${var.project}-dr-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.dr.resource_id
  scalable_dimension = aws_appautoscaling_target.dr.scalable_dimension
  service_namespace  = aws_appautoscaling_target.dr.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 60.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

---

## Multi-Site Active/Active

```
┌────────────────────────────────────────────────────────────────┐
│                 Multi-Site Active/Active                        │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│                    Route 53                                     │
│              (Weighted / Latency)                               │
│                 ┌────┴────┐                                    │
│                 │         │                                    │
│           ┌─────▼──┐  ┌──▼─────┐                              │
│           │sa-east │  │us-east │                              │
│           │  -1    │  │  -1    │                              │
│           ├────────┤  ├────────┤                              │
│           │ ALB    │  │ ALB    │                              │
│           │ ECS    │  │ ECS    │                              │
│           │ Aurora │  │ Aurora │  ← Global Database            │
│           │(write) │  │(write) │  ← Write Forwarding          │
│           └────────┘  └────────┘                              │
│                │            │                                  │
│                └──── DynamoDB Global Tables ────┘              │
│                      (multi-region write)                      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### DynamoDB Global Tables

```hcl
# DynamoDB Global Table — multi-region write
resource "aws_dynamodb_table" "main" {
  name         = "${var.project}-sessions"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "PK"
  range_key    = "SK"

  attribute {
    name = "PK"
    type = "S"
  }

  attribute {
    name = "SK"
    type = "S"
  }

  # Global Table replication
  replica {
    region_name    = "us-east-1"
    kms_key_arn    = var.dr_kms_key_arn
    point_in_time_recovery = true
  }

  point_in_time_recovery {
    enabled = true
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = var.primary_kms_key_arn
  }

  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"  # Obrigatório para Global Tables

  ttl {
    attribute_name = "ExpiresAt"
    enabled        = true
  }
}
```

### S3 Cross-Region Replication

```hcl
resource "aws_s3_bucket" "primary" {
  bucket = "${var.project}-data-${var.primary_region}"
}

resource "aws_s3_bucket_versioning" "primary" {
  bucket = aws_s3_bucket.primary.id
  versioning_configuration {
    status = "Enabled"  # Obrigatório para replicação
  }
}

resource "aws_s3_bucket_replication_configuration" "primary_to_dr" {
  depends_on = [aws_s3_bucket_versioning.primary]
  bucket     = aws_s3_bucket.primary.id
  role       = aws_iam_role.s3_replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    filter {}  # Tudo

    destination {
      bucket        = aws_s3_bucket.dr.arn
      storage_class = "STANDARD_IA"

      encryption_configuration {
        replica_kms_key_id = var.dr_kms_key_arn
      }

      metrics {
        status = "Enabled"
        event_threshold {
          minutes = 15
        }
      }

      replication_time {
        status = "Enabled"
        time {
          minutes = 15
        }
      }
    }

    source_selection_criteria {
      sse_kms_encrypted_objects {
        status = "Enabled"
      }
    }

    delete_marker_replication {
      status = "Enabled"
    }
  }
}
```

---

## DR Testing

### Runbook Template

```
# DR Runbook — [NOME DO SERVIÇO]

## Pre-Requisitos
- [ ] Acesso à região DR verificado
- [ ] DNS TTL baixo (60s) confirmado
- [ ] Backup recente validado (< 24h)
- [ ] Equipe de plantão notificada

## Failover Steps
1. **Verificar** o incidente na região primária
   - CloudWatch alarms
   - Route 53 health checks
   - AWS Health Dashboard

2. **Comunicar** status para stakeholders
   - Canal: #incident-response
   - Status page: atualizar

3. **Executar failover do banco**
   - Aurora: `aws rds failover-global-cluster --global-cluster-identifier <id> --target-db-cluster-identifier <dr-cluster>`
   - DynamoDB Global Tables: automático (nenhuma ação necessária)

4. **Verificar compute na região DR**
   - ECS: verificar tasks running, escalar se necessário
   - `aws ecs update-service --cluster <cluster> --service <service> --desired-count <prod-count>`

5. **DNS failover**
   - Se automático (Route 53 failover): verificar propagação
   - Se manual: `aws route53 change-resource-record-sets`

6. **Validação**
   - Health check endpoints
   - Smoke tests automatizados
   - Verificar métricas de negócio

## Failback Steps
1. Aguardar região primária estável por 30+ minutos
2. Resync dados da região DR → primária
3. Failback do banco
4. Validar região primária
5. Restaurar DNS para primário
6. Escalar DR de volta para standby

## Métricas
- Tempo de detecção: ____ min
- Tempo de decisão: ____ min
- Tempo de execução: ____ min
- RTO alcançado: ____ min (meta: ____ min)
```

### Game Day Schedule

| Frequência | Tipo de Teste | Escopo |
|-----------|---------------|--------|
| Mensal | Backup restore test | Restaurar 1 backup e validar integridade |
| Trimestral | Failover parcial | Failover de 1 serviço para região DR |
| Semestral | Full DR drill | Failover completo, operar da região DR por 2h |
| Anual | Chaos engineering | Simular falha de AZ ou região com Fault Injection Simulator |

---

## DR Checklist

### Infraestrutura

- [ ] Backup automatizado com AWS Backup (diário + cross-region)
- [ ] Vault Lock habilitado (proteção contra ransomware)
- [ ] Aurora Global Database ou replicação cross-region
- [ ] DynamoDB Global Tables para dados de sessão
- [ ] S3 Cross-Region Replication com RTC (Replication Time Control)
- [ ] DNS failover com Route 53 health checks
- [ ] IaC para provisionar a região DR (Terraform)

### Operacional

- [ ] RPO/RTO definidos por serviço
- [ ] Runbooks documentados e testados
- [ ] Game Days regulares (mínimo trimestral)
- [ ] Alertas de falha de replicação
- [ ] Monitoramento de lag de replicação
- [ ] Equipe treinada em procedimentos de failover

---

## Referências

- [AWS Disaster Recovery Workloads](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html)
- [AWS Backup Developer Guide](https://docs.aws.amazon.com/aws-backup/latest/devguide/)
- [Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [Route 53 Health Checks](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover.html)
