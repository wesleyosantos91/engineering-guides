````markdown
# Segurança — Terraform AWS

> **Objetivo:** Guia de segurança para infraestrutura como código na AWS,
> abrangendo IAM, secrets management, encryption, scanning, compliance
> e práticas do AWS Well-Architected Framework (pilar de segurança).

---

## Sumário

- [Princípios de Segurança](#princípios-de-segurança)
- [IAM — Identity and Access Management](#iam--identity-and-access-management)
  - [Roles para Terraform](#roles-para-terraform)
  - [Least Privilege na Prática](#least-privilege-na-prática)
  - [Service-Linked Roles](#service-linked-roles)
  - [Cross-Account Access](#cross-account-access)
- [Secrets Management](#secrets-management)
- [Encryption](#encryption)
- [Networking Seguro](#networking-seguro)
- [Security Scanning Pipeline](#security-scanning-pipeline)
- [Compliance as Code](#compliance-as-code)
- [AWS Config Rules](#aws-config-rules)
- [GuardDuty e Security Hub](#guardduty-e-security-hub)
- [Checklist de Segurança por Serviço](#checklist-de-segurança-por-serviço)
- [Incident Response — Terraform](#incident-response--terraform)

---

## Princípios de Segurança

1. **Least Privilege** — conceda apenas as permissões necessárias
2. **Defense in Depth** — múltiplas camadas de segurança
3. **Encryption Everywhere** — at rest e in transit, sempre
4. **Zero Trust** — não confie em nada, verifique tudo
5. **Auditoria** — log tudo, monitore tudo
6. **Shift Left** — detecte problemas o mais cedo possível (no PR, não em produção)
7. **Immutable Infrastructure** — não faça patch, substitua
8. **Secrets não são código** — nunca no repositório

---

## IAM — Identity and Access Management

### Roles para Terraform

```hcl
# Role para CI/CD executar Terraform
resource "aws_iam_role" "terraform_ci" {
  name = "terraform-ci-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        # GitHub Actions OIDC
        Effect = "Allow"
        Principal = {
          Federated = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/token.actions.githubusercontent.com"
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          }
          StringLike = {
            "token.actions.githubusercontent.com:sub" = "repo:myorg/infra:*"
          }
        }
      }
    ]
  })

  # Limite máximo de permissões
  permissions_boundary = aws_iam_policy.terraform_boundary.arn

  tags = local.common_tags
}
```

### Least Privilege na Prática

```hcl
# ❌ RUIM — Admin access
resource "aws_iam_role_policy_attachment" "terrible" {
  role       = aws_iam_role.terraform_ci.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}

# ✅ BOM — Permissões específicas por serviço
resource "aws_iam_policy" "terraform_networking" {
  name = "terraform-networking-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "VPCManagement"
        Effect = "Allow"
        Action = [
          "ec2:CreateVpc",
          "ec2:DeleteVpc",
          "ec2:DescribeVpcs",
          "ec2:ModifyVpcAttribute",
          "ec2:CreateSubnet",
          "ec2:DeleteSubnet",
          "ec2:DescribeSubnets",
          "ec2:CreateInternetGateway",
          "ec2:DeleteInternetGateway",
          "ec2:AttachInternetGateway",
          "ec2:DetachInternetGateway",
          "ec2:DescribeInternetGateways",
          "ec2:CreateNatGateway",
          "ec2:DeleteNatGateway",
          "ec2:DescribeNatGateways",
          "ec2:AllocateAddress",
          "ec2:ReleaseAddress",
          "ec2:DescribeAddresses",
          "ec2:CreateRouteTable",
          "ec2:DeleteRouteTable",
          "ec2:CreateRoute",
          "ec2:DeleteRoute",
          "ec2:AssociateRouteTable",
          "ec2:DisassociateRouteTable",
          "ec2:DescribeRouteTables",
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "aws:RequestedRegion" = ["us-east-1", "sa-east-1"]
          }
        }
      },
      {
        Sid    = "TagManagement"
        Effect = "Allow"
        Action = [
          "ec2:CreateTags",
          "ec2:DeleteTags",
          "ec2:DescribeTags",
        ]
        Resource = "*"
      }
    ]
  })
}
```

### Permissions Boundary

```hcl
# Limite máximo — mesmo com Admin, não pode exceder
resource "aws_iam_policy" "terraform_boundary" {
  name = "terraform-permissions-boundary"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowMostActions"
        Effect = "Allow"
        Action = "*"
        Resource = "*"
      },
      {
        # Nunca pode criar users ou access keys
        Sid    = "DenyIAMUserCreation"
        Effect = "Deny"
        Action = [
          "iam:CreateUser",
          "iam:CreateAccessKey",
          "iam:CreateLoginProfile",
        ]
        Resource = "*"
      },
      {
        # Nunca pode alterar billing
        Sid    = "DenyBillingActions"
        Effect = "Deny"
        Action = [
          "aws-portal:*",
          "budgets:*",
        ]
        Resource = "*"
      },
      {
        # Nunca pode sair da organização
        Sid    = "DenyOrgActions"
        Effect = "Deny"
        Action = [
          "organizations:LeaveOrganization",
          "organizations:DeleteOrganization",
        ]
        Resource = "*"
      }
    ]
  })
}
```

### Cross-Account Access

```hcl
# Conta de Management → assume role na conta de Workload
provider "aws" {
  alias  = "prod"
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::PROD_ACCOUNT_ID:role/TerraformExecutionRole"
    session_name = "terraform-ci"
    external_id  = "terraform-prod-access"
  }
}

# Use o provider aliased para recursos na conta prod
resource "aws_vpc" "prod" {
  provider   = aws.prod
  cidr_block = "10.0.0.0/16"
}
```

---

## Secrets Management

### ❌ Nunca Faça Isso

```hcl
# RUIM — senha no código
resource "aws_db_instance" "main" {
  password = "SuperSecret123!"  # ❌ JAMAIS
}

# RUIM — senha em terraform.tfvars commitado
password = "SuperSecret123!"  # ❌ Se está no Git, não é secret
```

### ✅ AWS Secrets Manager

```hcl
# Crie o secret no Secrets Manager
resource "aws_secretsmanager_secret" "db_password" {
  name                    = "/${var.environment}/database/master-password"
  description             = "RDS master password"
  recovery_window_in_days = 7

  tags = local.common_tags
}

# Gere senha aleatória
resource "random_password" "db_password" {
  length           = 32
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db_password.result
}

# Use no RDS
resource "aws_db_instance" "main" {
  # ...
  password = random_password.db_password.result

  lifecycle {
    ignore_changes = [password]  # Rotação externa
  }
}
```

### ✅ SSM Parameter Store (SecureString)

```hcl
resource "aws_ssm_parameter" "api_key" {
  name        = "/${var.environment}/api/external-key"
  description = "API key for external service"
  type        = "SecureString"
  value       = var.api_key  # Passado via CI/CD envvar
  key_id      = aws_kms_key.secrets.arn

  tags = local.common_tags
}
```

### ✅ Referência Dinâmica (sem expor no state)

```hcl
# Leia secrets em runtime, não no Terraform
resource "aws_ecs_task_definition" "app" {
  container_definitions = jsonencode([
    {
      name = "app"
      secrets = [
        {
          name      = "DB_PASSWORD"
          valueFrom = aws_secretsmanager_secret.db_password.arn
        },
        {
          name      = "API_KEY"
          valueFrom = aws_ssm_parameter.api_key.arn
        }
      ]
    }
  ])
}
```

---

## Encryption

### S3

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "${local.name_prefix}-data"
}

# Encryption obrigatória
resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.data.arn
    }
    bucket_key_enabled = true
  }
}

# Bloquear acesso público
resource "aws_s3_bucket_public_access_block" "data" {
  bucket = aws_s3_bucket.data.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Forçar SSL
resource "aws_s3_bucket_policy" "enforce_ssl" {
  bucket = aws_s3_bucket.data.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "ForceSSLOnly"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.data.arn,
          "${aws_s3_bucket.data.arn}/*",
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}
```

### RDS

```hcl
resource "aws_db_instance" "main" {
  # ...
  storage_encrypted = true
  kms_key_id        = aws_kms_key.data.arn

  # Encryption in transit
  parameter_group_name = aws_db_parameter_group.enforce_ssl.name
}

resource "aws_db_parameter_group" "enforce_ssl" {
  family = "postgres16"
  name   = "${local.name_prefix}-enforce-ssl"

  parameter {
    name  = "rds.force_ssl"
    value = "1"
  }
}
```

### KMS Key Pattern

```hcl
resource "aws_kms_key" "data" {
  description             = "KMS key for ${var.project} data encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true  # ✅ Rotação automática

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "RootAccountFullAccess"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "ServiceAccess"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.app.arn
        }
        Action = [
          "kms:Decrypt",
          "kms:DescribeKey",
          "kms:GenerateDataKey*",
        ]
        Resource = "*"
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_kms_alias" "data" {
  name          = "alias/${var.project}-${var.environment}-data"
  target_key_id = aws_kms_key.data.key_id
}
```

---

## Networking Seguro

### VPC Flow Logs

```hcl
resource "aws_flow_log" "vpc" {
  vpc_id                   = aws_vpc.main.id
  traffic_type             = "ALL"
  log_destination_type     = "cloud-watch-logs"
  log_destination          = aws_cloudwatch_log_group.flow_logs.arn
  iam_role_arn             = aws_iam_role.flow_logs.arn
  max_aggregation_interval = 60

  tags = local.common_tags
}

resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/flow-logs/${local.name_prefix}"
  retention_in_days = 90
  kms_key_id        = aws_kms_key.logs.arn

  tags = local.common_tags
}
```

### Security Groups — Regras Rígidas

```hcl
# ✅ Específico — apenas portas e CIDRs necessários
resource "aws_security_group" "app" {
  name        = "${local.name_prefix}-app-sg"
  description = "Security group for application containers"
  vpc_id      = aws_vpc.main.id

  # Apenas ALB pode acessar a aplicação
  ingress {
    from_port       = var.app_port
    to_port         = var.app_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "Allow inbound from ALB only"
  }

  # Saída controlada
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow HTTPS outbound"
  }

  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.database.id]
    description     = "Allow PostgreSQL to database"
  }

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-app-sg"
  })
}

# ❌ NUNCA faça isso
resource "aws_security_group" "terrible" {
  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]  # ❌ Tudo aberto
  }
}
```

### NACLs (Network ACLs)

```hcl
resource "aws_network_acl" "private" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.private[*].id

  # Bloqueia SSH de qualquer lugar nas subnets privadas
  ingress {
    rule_no    = 50
    protocol   = "tcp"
    action     = "deny"
    cidr_block = "0.0.0.0/0"
    from_port  = 22
    to_port    = 22
  }

  # Permite tráfego interno da VPC
  ingress {
    rule_no    = 100
    protocol   = "-1"
    action     = "allow"
    cidr_block = var.vpc_cidr
    from_port  = 0
    to_port    = 0
  }

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-nacl"
  })
}
```

---

## Security Scanning Pipeline

### Pipeline Completa

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  pull_request:
    paths: ['**.tf']

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # 1. Checkov — CIS Benchmarks, compliance
      - name: Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: .
          framework: terraform
          output_format: cli,sarif
          output_file_path: console,results.sarif
          soft_fail: false
          skip_check: CKV_AWS_144  # Skip específico com justificativa

      # 2. tfsec — Análise de segurança
      - name: tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          sarif_file: tfsec.sarif

      # 3. Trivy — Vulnerabilidades em config
      - name: Trivy
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: .
          severity: HIGH,CRITICAL
          exit-code: 1

      # 4. Conftest — Policies customizadas (OPA)
      - name: Policy Check
        run: |
          terraform plan -out=tfplan.binary
          terraform show -json tfplan.binary > tfplan.json
          conftest test tfplan.json -p policy/ --fail-on-warn

      # 5. Upload SARIF para GitHub Security tab
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: results.sarif
```

### Ferramentas de Scanning

| Ferramenta | Foco | Gratuito |
|-----------|------|----------|
| **Checkov** | CIS, compliance, 1000+ checks | ✅ |
| **tfsec** | Segurança AWS/Azure/GCP | ✅ |
| **Trivy** | Vulnerabilidades, misconfig | ✅ |
| **Conftest** | Policies customizadas (OPA) | ✅ |
| **Sentinel** | Policy as Code (HashiCorp) | Terraform Cloud |
| **Snyk IaC** | Vulnerabilidades | Freemium |

---

## Compliance as Code

### AWS CIS Benchmark via Checkov

```bash
# Verificar conformidade com CIS AWS Foundations
checkov -d . --check CKV_AWS_*

# Checks importantes:
# CKV_AWS_18  - S3 access logging
# CKV_AWS_19  - S3 server-side encryption
# CKV_AWS_21  - S3 versioning
# CKV_AWS_145 - S3 KMS encryption
# CKV_AWS_23  - Security group has description
# CKV_AWS_24  - No SSH from 0.0.0.0/0
# CKV_AWS_25  - No RDP from 0.0.0.0/0
# CKV_AWS_16  - RDS encryption enabled
# CKV_AWS_17  - RDS logging enabled
# CKV_AWS_26  - SNS topic encrypted
# CKV_AWS_27  - SQS queue encrypted
# CKV_AWS_35  - CloudTrail log file validation
# CKV_AWS_36  - CloudTrail logging enabled
# CKV_AWS_40  - IAM policy no wildcard actions
```

### Baseline — Ignorar Issues Conhecidos

```bash
# Gerar baseline (issues aceitos)
checkov -d . --create-baseline
# Gera .checkov.baseline

# Rodar ignorando issues do baseline
checkov -d . --baseline .checkov.baseline
```

---

## AWS Config Rules

```hcl
# Monitore compliance em runtime (pós-deploy)
resource "aws_config_config_rule" "s3_encryption" {
  name = "s3-bucket-server-side-encryption-enabled"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
  }

  tags = local.common_tags
}

resource "aws_config_config_rule" "rds_encryption" {
  name = "rds-storage-encrypted"

  source {
    owner             = "AWS"
    source_identifier = "RDS_STORAGE_ENCRYPTED"
  }
}

resource "aws_config_config_rule" "sg_open_to_world" {
  name = "restricted-ssh"

  source {
    owner             = "AWS"
    source_identifier = "INCOMING_SSH_DISABLED"
  }
}
```

---

## GuardDuty e Security Hub

```hcl
# GuardDuty — Detecção de ameaças
resource "aws_guardduty_detector" "main" {
  enable = true

  finding_publishing_frequency = "FIFTEEN_MINUTES"

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

  tags = local.common_tags
}

# Security Hub — Agregação de findings
resource "aws_securityhub_account" "main" {}

resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.4.0"
  depends_on    = [aws_securityhub_account.main]
}

resource "aws_securityhub_standards_subscription" "aws_best_practices" {
  standards_arn = "arn:aws:securityhub:${data.aws_region.current.name}::standards/aws-foundational-security-best-practices/v/1.0.0"
  depends_on    = [aws_securityhub_account.main]
}
```

---

## Checklist de Segurança por Serviço

### S3

- [ ] Encryption at rest (SSE-KMS)
- [ ] Block public access (todas as 4 flags)
- [ ] Bucket policy com enforce SSL
- [ ] Versioning habilitado
- [ ] Access logging habilitado
- [ ] Lifecycle rules configuradas
- [ ] CORS restritivo (se necessário)

### RDS

- [ ] storage_encrypted = true
- [ ] Multi-AZ (produção)
- [ ] Backup automático com retenção >= 7 dias
- [ ] Subnet privada (sem acesso público)
- [ ] Security group restritivo
- [ ] SSL obrigatório (force_ssl)
- [ ] Enhanced monitoring habilitado
- [ ] Performance Insights habilitado
- [ ] deletion_protection = true
- [ ] Senha no Secrets Manager (não no código)

### ECS

- [ ] Task role com least privilege
- [ ] Execution role separada
- [ ] Secrets via Secrets Manager (não env vars)
- [ ] Logging para CloudWatch
- [ ] Container insights habilitado
- [ ] Fargate (preferir sobre EC2 launch type)
- [ ] VPC privadas para tasks

### Lambda

- [ ] Execution role com least privilege
- [ ] VPC quando acessa recursos internos
- [ ] X-Ray tracing habilitado
- [ ] Dead letter queue configurada
- [ ] Concurrency limit definido
- [ ] Environment variables encrypted

### EC2 (quando inevitável)

- [ ] AMI hardened e atualizada
- [ ] IMDSv2 obrigatório
- [ ] EBS encryption habilitado
- [ ] Session Manager ao invés de SSH
- [ ] Security group restritivo
- [ ] IAM Instance Profile (não access keys)

```hcl
# IMDSv2 obrigatório
resource "aws_instance" "app" {
  # ...
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # ✅ Força IMDSv2
    http_put_response_hop_limit = 1
  }
}
```

---

## Incident Response — Terraform

### Revogar Acesso de Emergência

```bash
# 1. Identifique o recurso comprometido
terraform state list | grep aws_iam

# 2. Revogue diretamente via CLI (mais rápido que Terraform)
aws iam detach-role-policy \
  --role-name compromised-role \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# 3. Atualize o Terraform para refletir
terraform plan  # Deve mostrar as mudanças feitas via CLI
terraform apply # Sincroniza o state
```

### Lock de Conta

```hcl
# SCP para bloquear toda atividade em caso de incidente
resource "aws_organizations_policy" "lockdown" {
  name = "emergency-lockdown"
  type = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyAllExceptSecurityTeam"
        Effect    = "Deny"
        Action    = "*"
        Resource  = "*"
        Condition = {
          StringNotLike = {
            "aws:PrincipalArn" = [
              "arn:aws:iam::*:role/SecurityTeamRole",
              "arn:aws:iam::*:role/BreakGlassRole",
            ]
          }
        }
      }
    ]
  })
}
```

````
