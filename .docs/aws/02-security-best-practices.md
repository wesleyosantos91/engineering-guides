# AWS Security — Boas Práticas

Guia completo de segurança na AWS seguindo o modelo de responsabilidade compartilhada, defense in depth e zero trust.

---

## Índice

1. [IAM — Identity & Access Management](#iam)
2. [Criptografia & Gerenciamento de Secrets](#criptografia)
3. [Segurança de Rede](#segurança-de-rede)
4. [Detecção & Monitoramento](#detecção--monitoramento)
5. [Proteção de Dados](#proteção-de-dados)
6. [Incident Response](#incident-response)

---

## IAM

### Princípios Fundamentais

1. **Menor privilégio** — Apenas permissões necessárias
2. **Sem credenciais de longa duração** — Usar IAM Roles, nunca access keys
3. **MFA obrigatório** — Para todos os usuários humanos
4. **Centralizar identidade** — AWS IAM Identity Center (SSO)
5. **Conditions** — Restringir por IP, região, tempo, tags

### Hierarquia de Controles

```
┌─────────────────────────────────────────────────────────┐
│                   Service Control Policies (SCPs)        │
│              (OrganizationsBarreira mais alta)           │
│  ┌───────────────────────────────────────────────────┐  │
│  │            Permission Boundaries                    │  │
│  │         (Limite máximo por role/user)               │  │
│  │  ┌─────────────────────────────────────────────┐   │  │
│  │  │       Identity-Based Policies                │   │  │
│  │  │    (Permissões do role/user)                 │   │  │
│  │  │  ┌───────────────────────────────────────┐   │   │  │
│  │  │  │    Resource-Based Policies            │   │   │  │
│  │  │  │  (Permissões no recurso)              │   │   │  │
│  │  │  └───────────────────────────────────────┘   │   │  │
│  │  └─────────────────────────────────────────────┘   │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
  Permissão efetiva = interseção de TODOS os níveis
```

### Service Control Policies (SCPs)

```json
// SCP: Bloquear regiões não autorizadas
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonApprovedRegions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "sa-east-1",
            "us-east-1"
          ]
        },
        "ArnNotLike": {
          "aws:PrincipalARN": [
            "arn:aws:iam::*:role/OrganizationAdmin"
          ]
        }
      }
    },
    {
      "Sid": "DenyLeaveOrganization",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    },
    {
      "Sid": "DenyDisableSecurityServices",
      "Effect": "Deny",
      "Action": [
        "guardduty:DeleteDetector",
        "guardduty:DisassociateFromMasterAccount",
        "securityhub:DisableSecurityHub",
        "config:DeleteConfigRule",
        "config:StopConfigurationRecorder",
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail"
      ],
      "Resource": "*"
    }
  ]
}
```

### IAM Identity Center (SSO)

```hcl
# IAM Identity Center — acesso centralizado multi-account
resource "aws_ssoadmin_permission_set" "admin" {
  name             = "AdministratorAccess"
  description      = "Full admin access"
  instance_arn     = tolist(data.aws_ssoadmin_instances.main.arns)[0]
  session_duration = "PT4H"  # 4 horas

  tags = var.tags
}

resource "aws_ssoadmin_managed_policy_attachment" "admin" {
  instance_arn       = tolist(data.aws_ssoadmin_instances.main.arns)[0]
  permission_set_arn = aws_ssoadmin_permission_set.admin.arn
  managed_policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}

# Permission Set com política customizada (developer)
resource "aws_ssoadmin_permission_set" "developer" {
  name             = "DeveloperAccess"
  description      = "Developer access with boundaries"
  instance_arn     = tolist(data.aws_ssoadmin_instances.main.arns)[0]
  session_duration = "PT8H"

  tags = var.tags
}

resource "aws_ssoadmin_permission_set_inline_policy" "developer" {
  instance_arn       = tolist(data.aws_ssoadmin_instances.main.arns)[0]
  permission_set_arn = aws_ssoadmin_permission_set.developer.arn

  inline_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowDeveloperServices"
        Effect = "Allow"
        Action = [
          "ecs:*", "ecr:*", "lambda:*",
          "logs:*", "cloudwatch:*", "xray:*",
          "s3:GetObject", "s3:PutObject", "s3:ListBucket",
          "dynamodb:*", "sqs:*", "sns:*",
          "secretsmanager:GetSecretValue",
          "ssm:GetParameter*"
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyDangerousActions"
        Effect = "Deny"
        Action = [
          "iam:CreateUser", "iam:CreateAccessKey",
          "organizations:*", "account:*",
          "ec2:*Host*", "ec2:*ReservedInstances*"
        ]
        Resource = "*"
      }
    ]
  })
}

# Associar grupo a conta
resource "aws_ssoadmin_account_assignment" "dev_team" {
  instance_arn       = tolist(data.aws_ssoadmin_instances.main.arns)[0]
  permission_set_arn = aws_ssoadmin_permission_set.developer.arn

  principal_id   = aws_identitystore_group.developers.group_id
  principal_type = "GROUP"

  target_id   = var.dev_account_id
  target_type = "AWS_ACCOUNT"
}

data "aws_ssoadmin_instances" "main" {}
```

**Permission Sets recomendados:**

| Permission Set | Público | Escopo |
|----------------|---------|--------|
| AdministratorAccess | Platform team | Acesso total (audit trail) |
| DeveloperAccess | Devs | Serviços de compute, storage, observability |
| ReadOnlyAccess | Todos | Leitura para debugging |
| DatabaseAdmin | DBAs | RDS, DynamoDB, ElastiCache |
| SecurityAudit | Security team | Leitura de configs de segurança |

### Permission Boundaries

```hcl
# Permission boundary — limita o máximo que qualquer role pode ter
resource "aws_iam_policy" "boundary" {
  name = "${var.project}-permission-boundary"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowedServices"
        Effect = "Allow"
        Action = [
          "s3:*",
          "dynamodb:*",
          "sqs:*",
          "sns:*",
          "lambda:*",
          "logs:*",
          "xray:*",
          "secretsmanager:GetSecretValue",
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyDangerousActions"
        Effect = "Deny"
        Action = [
          "iam:CreateUser",
          "iam:CreateAccessKey",
          "organizations:*",
          "account:*"
        ]
        Resource = "*"
      },
      {
        Sid    = "RestrictToRegion"
        Effect = "Deny"
        NotAction = [
          "iam:*",
          "sts:*",
          "s3:*",
          "cloudfront:*"
        ]
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "aws:RequestedRegion" = ["sa-east-1", "us-east-1"]
          }
        }
      }
    ]
  })
}

# Aplicar boundary a todas as roles do projeto
resource "aws_iam_role" "app" {
  name                 = "${var.project}-app-role"
  permissions_boundary = aws_iam_policy.boundary.arn
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}
```

### IAM Policies — Padrões

```hcl
# PADRÃO 1: Acesso a recursos específicos por tag
resource "aws_iam_role_policy" "tag_based" {
  role = aws_iam_role.app.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = "arn:aws:s3:::*/*"
      Condition = {
        StringEquals = {
          "s3:ExistingObjectTag/Project" = var.project_name
        }
      }
    }]
  })
}

# PADRÃO 2: Cross-account assume role
resource "aws_iam_role" "cross_account" {
  name = "${var.project}-cross-account"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        AWS = "arn:aws:iam::${var.trusted_account_id}:root"
      }
      Action = "sts:AssumeRole"
      Condition = {
        StringEquals = {
          "sts:ExternalId" = var.external_id
        }
        Bool = {
          "aws:MultiFactorAuthPresent" = "true"
        }
      }
    }]
  })
}

# PADRÃO 3: Session tags para controle granular
# A role assume com tags de sessão que restringem acesso
resource "aws_iam_role_policy" "session_tag_based" {
  role = aws_iam_role.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:Query"]
      Resource = aws_dynamodb_table.main.arn
      Condition = {
        "ForAllValues:StringEquals" = {
          "dynamodb:LeadingKeys" = ["$${aws:PrincipalTag/TenantId}"]
        }
      }
    }]
  })
}
```

---

## Criptografia

### KMS — Estratégia de Chaves

```
┌──────────────────────────────────────────────────────┐
│            KMS Key Strategy                           │
├──────────────────────────────────────────────────────┤
│                                                      │
│  AWS Managed Keys (aws/s3, aws/ebs, etc.)           │
│  └── Conveniência, sem controle de políticas         │
│      Usar para: dev, dados não-sensíveis             │
│                                                      │
│  Customer Managed Keys (CMK)                         │
│  └── Controle total: políticas, rotação, auditoria  │
│      Usar para: produção, dados sensíveis, PII      │
│                                                      │
│  Estratégia recomendada:                             │
│  • 1 KMS key por serviço/domínio (não por recurso)  │
│  • Rotação automática habilitada (90 dias)           │
│  • Key policy com menor privilégio                   │
│  • Separar key admin de key usage                    │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Secrets Manager

```hcl
# Secret com rotação automática
resource "aws_secretsmanager_secret" "db_credentials" {
  name        = "${var.project}/${var.environment}/db-credentials"
  description = "Database credentials for ${var.project}"
  kms_key_id  = aws_kms_key.secrets.arn

  # Replicar para DR
  dynamic "replica" {
    for_each = var.environment == "prod" ? ["us-east-1"] : []
    content {
      region     = replica.value
      kms_key_id = var.dr_kms_key_arn
    }
  }
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  secret_string = jsonencode({
    username = var.db_username
    password = var.db_password
    engine   = "postgres"
    host     = aws_db_instance.main.address
    port     = 5432
    dbname   = var.db_name
  })
}

# Rotação automática
resource "aws_secretsmanager_secret_rotation" "db_credentials" {
  secret_id           = aws_secretsmanager_secret.db_credentials.id
  rotation_lambda_arn = aws_lambda_function.rotate_secret.arn

  rotation_rules {
    automatically_after_days = 30
  }
}

# Parameter Store para configurações não-sensíveis
resource "aws_ssm_parameter" "config" {
  name  = "/${var.project}/${var.environment}/config"
  type  = "String"
  value = jsonencode({
    feature_flags = {
      new_checkout = true
      dark_mode    = false
    }
    api_endpoints = {
      payment = "https://payment.internal.example.com"
    }
  })

  tags = var.tags
}

# Parameter Store SecureString para valores sensíveis
resource "aws_ssm_parameter" "api_key" {
  name   = "/${var.project}/${var.environment}/api-key"
  type   = "SecureString"
  value  = var.api_key
  key_id = aws_kms_key.secrets.arn

  tags = var.tags
}
```

### Criptografia em Trânsito e Repouso

```hcl
# Forçar HTTPS em S3
resource "aws_s3_bucket_policy" "force_ssl" {
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
          "${aws_s3_bucket.data.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      },
      {
        Sid       = "DenyOutdatedTLS"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.data.arn,
          "${aws_s3_bucket.data.arn}/*"
        ]
        Condition = {
          NumericLessThan = {
            "s3:TlsVersion" = 1.2
          }
        }
      }
    ]
  })
}

# ALB com TLS 1.2+
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.main.arn
  }
}

# Redirect HTTP → HTTPS
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

---

## Segurança de Rede

### VPC Security Design

```
┌─────────────────────────── VPC 10.0.0.0/16 ─────────────────────────────┐
│                                                                          │
│  ┌──── Public Subnet (10.0.100.0/24) ─────┐                            │
│  │  ALB, NAT Gateway, Bastion (se needed)  │                            │
│  │  NACL: Allow 80,443 inbound from 0.0.0.0│                           │
│  └───────────────────────────────────────────┘                           │
│                        │                                                  │
│  ┌──── Private Subnet (10.0.0.0/24) ──────┐                            │
│  │  ECS Tasks, Lambda, EC2                  │                            │
│  │  SG: Allow from ALB SG only             │                            │
│  │  NACL: Allow outbound to NAT Gateway    │                            │
│  └───────────────────────────────────────────┘                           │
│                        │                                                  │
│  ┌──── Data Subnet (10.0.200.0/24) ───────┐                            │
│  │  RDS, ElastiCache, OpenSearch            │                            │
│  │  SG: Allow from Private Subnet SG only  │                            │
│  │  NACL: Deny all inbound from public     │                            │
│  │  No internet access (nenhum NAT)        │                            │
│  └───────────────────────────────────────────┘                           │
│                                                                          │
│  VPC Endpoints: S3, DynamoDB, Secrets Manager, ECR, CloudWatch, STS     │
│  (Tráfego nunca sai da rede AWS)                                        │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Security Groups — Padrões

```hcl
# SG do ALB — aceita tráfego público
resource "aws_security_group" "alb" {
  name_prefix = "${var.project}-alb-"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTPS from Internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description     = "To ECS tasks"
    from_port       = var.container_port
    to_port         = var.container_port
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs.id]
  }

  tags = { Name = "${var.project}-alb-sg" }

  lifecycle {
    create_before_destroy = true
  }
}

# SG das ECS Tasks — aceita apenas do ALB
resource "aws_security_group" "ecs" {
  name_prefix = "${var.project}-ecs-"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "From ALB"
    from_port       = var.container_port
    to_port         = var.container_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    description = "To anywhere (NAT Gateway)"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project}-ecs-sg" }

  lifecycle {
    create_before_destroy = true
  }
}

# SG do Database — aceita apenas das ECS Tasks
resource "aws_security_group" "db" {
  name_prefix = "${var.project}-db-"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "PostgreSQL from ECS"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs.id]
  }

  # Sem egress — database não precisa de acesso externo
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = []
  }

  tags = { Name = "${var.project}-db-sg" }

  lifecycle {
    create_before_destroy = true
  }
}
```

### VPC Endpoints (PrivateLink)

```hcl
# VPC Endpoints — tráfego interno sem NAT Gateway
locals {
  interface_endpoints = [
    "com.amazonaws.${var.region}.ecr.api",
    "com.amazonaws.${var.region}.ecr.dkr",
    "com.amazonaws.${var.region}.ecs",
    "com.amazonaws.${var.region}.ecs-agent",
    "com.amazonaws.${var.region}.ecs-telemetry",
    "com.amazonaws.${var.region}.logs",
    "com.amazonaws.${var.region}.secretsmanager",
    "com.amazonaws.${var.region}.ssm",
    "com.amazonaws.${var.region}.sts",
    "com.amazonaws.${var.region}.monitoring",
    "com.amazonaws.${var.region}.xray",
  ]
}

resource "aws_vpc_endpoint" "interface" {
  for_each = toset(local.interface_endpoints)

  vpc_id              = aws_vpc.main.id
  service_name        = each.value
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]

  tags = {
    Name = "${var.project}-${split(".", each.value)[length(split(".", each.value)) - 1]}"
  }
}

# Gateway Endpoints (S3 e DynamoDB — gratuitos)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = aws_route_table.private[*].id

  tags = { Name = "${var.project}-s3-endpoint" }
}

resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.dynamodb"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = aws_route_table.private[*].id

  tags = { Name = "${var.project}-dynamodb-endpoint" }
}

# SG para VPC Endpoints
resource "aws_security_group" "vpc_endpoints" {
  name_prefix = "${var.project}-vpce-"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTPS from VPC"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }

  tags = { Name = "${var.project}-vpce-sg" }
}
```

### WAF (Web Application Firewall)

```hcl
resource "aws_wafv2_web_acl" "main" {
  name        = "${var.project}-waf"
  description = "WAF for ${var.project}"
  scope       = "REGIONAL"

  default_action {
    allow {}
  }

  # Rate limiting
  rule {
    name     = "rate-limit"
    priority = 1
    action {
      block {}
    }
    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "rate-limit"
    }
  }

  # AWS Managed Rules — Core Rule Set
  rule {
    name     = "aws-managed-common"
    priority = 2
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "aws-managed-common"
    }
  }

  # SQL Injection Protection
  rule {
    name     = "aws-managed-sqli"
    priority = 3
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesSQLiRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "aws-managed-sqli"
    }
  }

  # Known Bad Inputs
  rule {
    name     = "aws-managed-bad-inputs"
    priority = 4
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "aws-managed-bad-inputs"
    }
  }

  # Geo-restriction (exemplo: bloquear países específicos)
  rule {
    name     = "geo-restriction"
    priority = 5
    action {
      block {}
    }
    statement {
      geo_match_statement {
        country_codes = var.blocked_countries
      }
    }
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "geo-restriction"
    }
  }

  visibility_config {
    sampled_requests_enabled   = true
    cloudwatch_metrics_enabled = true
    metric_name                = "${var.project}-waf"
  }
}

# Associar WAF ao ALB
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

---

## Detecção & Monitoramento

### CloudTrail

```hcl
# CloudTrail — auditoria de todas as chamadas de API
resource "aws_cloudtrail" "main" {
  name                       = "${var.project}-trail"
  s3_bucket_name             = aws_s3_bucket.cloudtrail.id
  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail.arn

  is_multi_region_trail         = true
  include_global_service_events = true
  enable_log_file_validation    = true
  kms_key_id                    = aws_kms_key.cloudtrail.arn

  # Data events — S3 e Lambda
  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3"]
    }

    data_resource {
      type   = "AWS::Lambda::Function"
      values = ["arn:aws:lambda"]
    }
  }

  insight_selector {
    insight_type = "ApiCallRateInsight"
  }

  insight_selector {
    insight_type = "ApiErrorRateInsight"
  }
}
```

### AWS Config Rules

```hcl
# Config Rules — compliance contínua
resource "aws_config_config_rule" "s3_encryption" {
  name = "s3-bucket-server-side-encryption-enabled"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
  }
}

resource "aws_config_config_rule" "rds_encryption" {
  name = "rds-storage-encrypted"

  source {
    owner             = "AWS"
    source_identifier = "RDS_STORAGE_ENCRYPTED"
  }
}

resource "aws_config_config_rule" "iam_no_inline" {
  name = "iam-no-inline-policy-check"

  source {
    owner             = "AWS"
    source_identifier = "IAM_NO_INLINE_POLICY_CHECK"
  }
}

resource "aws_config_config_rule" "root_mfa" {
  name = "root-account-mfa-enabled"

  source {
    owner             = "AWS"
    source_identifier = "ROOT_ACCOUNT_MFA_ENABLED"
  }
}

resource "aws_config_config_rule" "vpc_flow_logs" {
  name = "vpc-flow-logs-enabled"

  source {
    owner             = "AWS"
    source_identifier = "VPC_FLOW_LOGS_ENABLED"
  }
}

resource "aws_config_config_rule" "ebs_encryption" {
  name = "encrypted-volumes"

  source {
    owner             = "AWS"
    source_identifier = "ENCRYPTED_VOLUMES"
  }
}
```

---

## Proteção de Dados

### Classificação de Dados

| Classificação | Exemplo | Criptografia | Acesso |
|---------------|---------|-------------|--------|
| **Público** | Marketing assets | SSE-S3 | Público via CloudFront |
| **Interno** | Docs internos | SSE-KMS (AWS managed) | Employees only |
| **Confidencial** | Dados de negócio | SSE-KMS (CMK) | Need-to-know |
| **Restrito** | PII, financeiro | SSE-KMS (CMK) + audit log | Aprovação explícita |

### Data Protection Checklist

- [ ] Dados criptografados em repouso (S3, EBS, RDS, DynamoDB)
- [ ] Dados criptografados em trânsito (TLS 1.2+)
- [ ] KMS keys com rotação automática habilitada
- [ ] Secrets em Secrets Manager (nunca em código ou env vars expostas)
- [ ] S3 buckets com public access block
- [ ] S3 bucket policies forçando SSL
- [ ] Versioning habilitado em S3 buckets críticos
- [ ] MFA Delete em S3 buckets de compliance
- [ ] Lifecycle policies para dados temporários
- [ ] Backups criptografados e testados
- [ ] Cross-region replication para DR (dados críticos)

---

## Incident Response

### Preparação

```hcl
# EventBridge rule para detecções do GuardDuty
resource "aws_cloudwatch_event_rule" "guardduty_findings" {
  name        = "${var.project}-guardduty-findings"
  description = "Capture GuardDuty findings"

  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
    detail = {
      severity = [{ numeric = [">=", 7] }]  # HIGH e CRITICAL
    }
  })
}

resource "aws_cloudwatch_event_target" "guardduty_to_sns" {
  rule      = aws_cloudwatch_event_rule.guardduty_findings.name
  target_id = "send-to-sns"
  arn       = aws_sns_topic.security_alerts.arn

  input_transformer {
    input_paths = {
      severity    = "$.detail.severity"
      type        = "$.detail.type"
      description = "$.detail.description"
      region      = "$.detail.region"
      account     = "$.detail.accountId"
    }
    input_template = "\"[SECURITY ALERT] Severity: <severity> | Type: <type> | Account: <account> | Region: <region> | Description: <description>\""
  }
}

# Auto-remediação: isolar instância comprometida
resource "aws_cloudwatch_event_rule" "auto_remediate" {
  name = "${var.project}-auto-remediate-ec2"

  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
    detail = {
      type = [
        "UnauthorizedAccess:EC2/MaliciousIPCaller.Custom",
        "CryptoCurrency:EC2/BitcoinTool.B!DNS"
      ]
    }
  })
}

resource "aws_cloudwatch_event_target" "isolate_instance" {
  rule      = aws_cloudwatch_event_rule.auto_remediate.name
  target_id = "isolate-instance"
  arn       = aws_lambda_function.isolate_instance.arn
}
```

### Incident Response Runbook

```markdown
## Runbook: Suspicious Activity Detected

### Severity Assessment
1. Check GuardDuty finding severity (1-10)
2. Identify affected resources
3. Determine blast radius

### Containment (< 15 min)
1. Isolate affected instance (modify SG to deny all)
2. Revoke compromised IAM credentials
3. Block suspicious IPs via WAF/NACL
4. Preserve forensic evidence (snapshot EBS, export logs)

### Investigation
1. Review CloudTrail logs for suspicious API calls
2. Check VPC Flow Logs for anomalous traffic
3. Analyze CloudWatch logs for application-level indicators
4. Check for unauthorized IAM changes

### Recovery
1. Terminate compromised resources
2. Rotate all potentially exposed credentials
3. Deploy clean resources from IaC
4. Verify integrity of data stores

### Post-Incident
1. Update GuardDuty threat lists
2. Add new WAF rules if needed
3. Update SCPs/IAM policies
4. Conduct post-mortem
5. Update this runbook
```

---

## Security Checklist Geral

### Account Level

- [ ] AWS Organizations com SCPs
- [ ] CloudTrail habilitado (multi-region, data events)
- [ ] AWS Config habilitado com rules de compliance
- [ ] GuardDuty habilitado
- [ ] Security Hub habilitado
- [ ] IAM Access Analyzer habilitado
- [ ] Root account com MFA hardware
- [ ] Billing alerts configurados

### Workload Level

- [ ] IAM Roles (nunca access keys de longa duração)
- [ ] Menor privilégio em todas as policies
- [ ] Permission boundaries em roles criadas por automação
- [ ] Secrets no Secrets Manager com rotação
- [ ] Criptografia em repouso (KMS CMK para prod)
- [ ] Criptografia em trânsito (TLS 1.2+)
- [ ] WAF no ALB/API Gateway/CloudFront
- [ ] VPC Flow Logs habilitados
- [ ] Security Groups com menor privilégio
- [ ] VPC Endpoints para serviços AWS
- [ ] S3 public access block habilitado
- [ ] Logging habilitado em todos os serviços

---

## Referências

- [AWS Security Best Practices](https://docs.aws.amazon.com/security/)
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [CIS AWS Foundations Benchmark](https://www.cisecurity.org/benchmark/amazon_web_services)
- [AWS Security Hub](https://docs.aws.amazon.com/securityhub/)
