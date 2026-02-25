# AWS Multi-Account Strategy — Boas Práticas

Guia de estratégia multi-account: AWS Organizations, Control Tower, OU design, account vending e governança centralizada.

---

## Por que Multi-Account?

```
┌──────────────────────────────────────────────────────────────────┐
│                    Benefícios                                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  🔒 Isolamento de segurança    Blast radius limitado por conta  │
│  💰 Billing separado           Custo visível por projeto/time   │
│  🚦 Limites independentes      Service quotas por conta         │
│  📋 Compliance granular        Regras diferentes por ambiente   │
│  🔧 Autonomia dos times        Cada time administra sua conta   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## OU Structure — Organizational Units

```
Root
├── Security OU
│   ├── Log Archive Account        ← CloudTrail, Config logs centralizados
│   └── Security Tooling Account   ← GuardDuty, Security Hub delegado
│
├── Infrastructure OU
│   ├── Network Hub Account        ← Transit Gateway, DNS, VPN
│   └── Shared Services Account    ← CI/CD, ECR, ferramentas compartilhadas
│
├── Sandbox OU                     ← Contas de experimentação (budget limitado)
│   └── Developer Sandbox Account
│
├── Workloads OU
│   ├── Production OU
│   │   ├── App A Prod Account
│   │   └── App B Prod Account
│   │
│   ├── Staging OU
│   │   ├── App A Staging Account
│   │   └── App B Staging Account
│   │
│   └── Development OU
│       ├── App A Dev Account
│       └── App B Dev Account
│
└── Suspended OU                   ← Contas desativadas (quarentena)
```

---

## AWS Organizations

### Terraform

```hcl
resource "aws_organizations_organization" "main" {
  feature_set = "ALL"

  aws_service_access_principals = [
    "cloudtrail.amazonaws.com",
    "config.amazonaws.com",
    "guardduty.amazonaws.com",
    "securityhub.amazonaws.com",
    "sso.amazonaws.com",
    "backup.amazonaws.com",
    "ram.amazonaws.com",
    "access-analyzer.amazonaws.com",
  ]

  enabled_policy_types = [
    "SERVICE_CONTROL_POLICY",
    "TAG_POLICY",
    "BACKUP_POLICY",
  ]
}

# OUs
resource "aws_organizations_organizational_unit" "security" {
  name      = "Security"
  parent_id = aws_organizations_organization.main.roots[0].id
}

resource "aws_organizations_organizational_unit" "infrastructure" {
  name      = "Infrastructure"
  parent_id = aws_organizations_organization.main.roots[0].id
}

resource "aws_organizations_organizational_unit" "workloads" {
  name      = "Workloads"
  parent_id = aws_organizations_organization.main.roots[0].id
}

resource "aws_organizations_organizational_unit" "production" {
  name      = "Production"
  parent_id = aws_organizations_organizational_unit.workloads.id
}

resource "aws_organizations_organizational_unit" "staging" {
  name      = "Staging"
  parent_id = aws_organizations_organizational_unit.workloads.id
}

resource "aws_organizations_organizational_unit" "development" {
  name      = "Development"
  parent_id = aws_organizations_organizational_unit.workloads.id
}

resource "aws_organizations_organizational_unit" "sandbox" {
  name      = "Sandbox"
  parent_id = aws_organizations_organization.main.roots[0].id
}

resource "aws_organizations_organizational_unit" "suspended" {
  name      = "Suspended"
  parent_id = aws_organizations_organization.main.roots[0].id
}
```

### Service Control Policies (SCPs)

```hcl
# SCP: Deny regiões não autorizadas
resource "aws_organizations_policy" "deny_regions" {
  name        = "DenyUnauthorizedRegions"
  description = "Restringe uso a regiões autorizadas"
  type        = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "DenyUnauthorizedRegions"
      Effect    = "Deny"
      NotAction = [
        "a4b:*", "budgets:*", "ce:*", "chime:*",
        "cloudfront:*", "cur:*", "globalaccelerator:*",
        "health:*", "iam:*", "importexport:*",
        "organizations:*", "route53:*", "sts:*",
        "support:*", "trustedadvisor:*", "waf:*"
      ]
      Resource = "*"
      Condition = {
        StringNotEquals = {
          "aws:RequestedRegion" = var.allowed_regions  # ["sa-east-1", "us-east-1"]
        }
      }
    }]
  })
}

resource "aws_organizations_policy_attachment" "deny_regions" {
  policy_id = aws_organizations_policy.deny_regions.id
  target_id = aws_organizations_organization.main.roots[0].id
}

# SCP: Proteger Security OU
resource "aws_organizations_policy" "protect_security" {
  name = "ProtectSecurityControls"
  type = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyCloudTrailChanges"
        Effect = "Deny"
        Action = [
          "cloudtrail:StopLogging",
          "cloudtrail:DeleteTrail",
          "cloudtrail:UpdateTrail"
        ]
        Resource = "*"
        Condition = {
          StringNotLike = {
            "aws:PrincipalArn" = "arn:aws:iam::*:role/OrganizationAdmin"
          }
        }
      },
      {
        Sid    = "DenyGuardDutyChanges"
        Effect = "Deny"
        Action = [
          "guardduty:DeleteDetector",
          "guardduty:DisassociateFromMasterAccount",
          "guardduty:UpdateDetector"
        ]
        Resource = "*"
        Condition = {
          StringNotLike = {
            "aws:PrincipalArn" = "arn:aws:iam::*:role/OrganizationAdmin"
          }
        }
      },
      {
        Sid    = "DenyLeavingOrganization"
        Effect = "Deny"
        Action = "organizations:LeaveOrganization"
        Resource = "*"
      }
    ]
  })
}

# SCP: Limitar Sandbox (budget protection)
resource "aws_organizations_policy" "sandbox_guardrails" {
  name = "SandboxGuardrails"
  type = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyExpensiveServices"
        Effect = "Deny"
        Action = [
          "redshift:*",
          "emr:*",
          "es:*",
          "sagemaker:CreateNotebookInstance",
          "sagemaker:CreateTrainingJob",
          "ec2:RunInstances"
        ]
        Resource = "*"
        Condition = {
          "ForAnyValue:StringNotLike" = {
            "ec2:InstanceType" = ["t3.*", "t4g.*"]
          }
        }
      },
      {
        Sid    = "DenyLargeInstances"
        Effect = "Deny"
        Action = "ec2:RunInstances"
        Resource = "arn:aws:ec2:*:*:instance/*"
        Condition = {
          "ForAnyValue:StringLike" = {
            "ec2:InstanceType" = ["*.metal", "*.24xlarge", "*.16xlarge", "*.12xlarge", "p*", "g*"]
          }
        }
      }
    ]
  })
}

resource "aws_organizations_policy_attachment" "sandbox_guardrails" {
  policy_id = aws_organizations_policy.sandbox_guardrails.id
  target_id = aws_organizations_organizational_unit.sandbox.id
}
```

---

## Centralized Logging

```
┌──────────────────────────────────────────────────────────────────┐
│                 Centralized Logging                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Account A ──┐                                                  │
│  Account B ──┼── CloudTrail ──→ Log Archive Account             │
│  Account C ──┤               ──→ S3 (org trail)                 │
│              │                                                   │
│              ├── Config     ──→ S3 (aggregator)                 │
│              ├── VPC Flow   ──→ CloudWatch Logs (cross-acct)    │
│              └── GuardDuty  ──→ Security Tooling Account        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Organization CloudTrail

```hcl
# Na Management Account
resource "aws_cloudtrail" "org" {
  name                       = "${var.project}-org-trail"
  s3_bucket_name             = var.log_archive_bucket_name  # Na Log Archive Account
  is_organization_trail      = true
  is_multi_region_trail      = true
  enable_log_file_validation = true
  kms_key_id                 = aws_kms_key.cloudtrail.arn

  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail_cw.arn

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

  insight_selectors {
    insight_type = "ApiCallRateInsight"
  }

  insight_selectors {
    insight_type = "ApiErrorRateInsight"
  }
}
```

### Config Aggregator

```hcl
# Aggregator na Security Tooling Account
resource "aws_config_configuration_aggregator" "org" {
  name = "${var.project}-org-aggregator"

  organization_aggregation_source {
    all_regions = true
    role_arn    = aws_iam_role.config_aggregator.arn
  }
}
```

---

## Network Hub (Transit Gateway)

```
┌──────────────────────────────────────────────────────────────────┐
│                    Network Hub Account                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│                     Transit Gateway                              │
│                    ┌─────┴─────┐                                │
│              ┌─────┤           ├─────┐                          │
│              │     │           │     │                          │
│         ┌────▼──┐ ┌▼────────┐ ┌▼────┐                          │
│         │Shared │ │Prod VPC │ │Dev  │                          │
│         │Svcs   │ │(App A)  │ │VPC  │                          │
│         │VPC    │ │         │ │     │                          │
│         └───────┘ └─────────┘ └─────┘                          │
│              │                                                   │
│         ┌────▼──────┐                                           │
│         │Inspection │  ← Network Firewall (IDS/IPS)            │
│         │VPC        │                                           │
│         └───────────┘                                           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Transit Gateway com RAM

```hcl
# Na Network Hub Account
resource "aws_ec2_transit_gateway" "main" {
  description                     = "${var.project} Transit Gateway"
  default_route_table_association = "disable"
  default_route_table_propagation = "disable"
  auto_accept_shared_attachments  = "enable"
  dns_support                     = "enable"

  tags = merge(var.tags, { Name = "${var.project}-tgw" })
}

# Compartilhar via RAM (Resource Access Manager)
resource "aws_ram_resource_share" "tgw" {
  name                      = "${var.project}-tgw-share"
  allow_external_principals = false  # Apenas contas da Organization
}

resource "aws_ram_resource_association" "tgw" {
  resource_arn       = aws_ec2_transit_gateway.main.arn
  resource_share_arn = aws_ram_resource_share.tgw.arn
}

resource "aws_ram_principal_association" "org" {
  principal          = aws_organizations_organization.main.arn
  resource_share_arn = aws_ram_resource_share.tgw.arn
}

# Route Tables segmentadas
resource "aws_ec2_transit_gateway_route_table" "prod" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  tags               = { Name = "prod-rt" }
}

resource "aws_ec2_transit_gateway_route_table" "nonprod" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  tags               = { Name = "nonprod-rt" }
}

resource "aws_ec2_transit_gateway_route_table" "shared" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  tags               = { Name = "shared-services-rt" }
}
```

---

## Account Vending (Automação)

### Account Factory com Terraform

```hcl
# Criar nova conta na Organization
resource "aws_organizations_account" "workload" {
  name      = var.account_name
  email     = var.account_email
  role_name = "OrganizationAccountAccessRole"
  parent_id = var.target_ou_id

  lifecycle {
    ignore_changes = [role_name]
  }

  tags = merge(var.tags, {
    Team        = var.team_name
    CostCenter  = var.cost_center
    Environment = var.environment
  })
}

# Baseline: aplicar configurações padrão na nova conta
module "account_baseline" {
  source = "./modules/account-baseline"
  providers = {
    aws = aws.new_account
  }

  account_id          = aws_organizations_account.workload.id
  account_name        = var.account_name
  environment         = var.environment
  vpc_cidr            = var.vpc_cidr
  transit_gateway_id  = var.transit_gateway_id
  log_archive_bucket  = var.log_archive_bucket
  security_account_id = var.security_account_id
}
```

### Account Baseline Module

```hcl
# modules/account-baseline/main.tf

# 1. VPC padrão
module "vpc" {
  source = "../vpc"

  name       = "${var.account_name}-vpc"
  cidr_block = var.vpc_cidr
  # ... configuração padrão de VPC
}

# 2. Attach ao Transit Gateway
resource "aws_ec2_transit_gateway_vpc_attachment" "main" {
  subnet_ids         = module.vpc.private_subnet_ids
  transit_gateway_id = var.transit_gateway_id
  vpc_id             = module.vpc.vpc_id
}

# 3. Config Rules padrão
resource "aws_config_configuration_recorder" "main" {
  role_arn = aws_iam_role.config.arn

  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }
}

# 4. GuardDuty (membro)
resource "aws_guardduty_detector" "main" {
  enable = true
}

# 5. IAM Password Policy
resource "aws_iam_account_password_policy" "strict" {
  minimum_password_length        = 14
  require_lowercase_characters   = true
  require_numbers                = true
  require_uppercase_characters   = true
  require_symbols                = true
  allow_users_to_change_password = true
  max_password_age               = 90
  password_reuse_prevention      = 24
}

# 6. Default EBS encryption
resource "aws_ebs_encryption_by_default" "main" {
  enabled = true
}

# 7. S3 Block Public Access (conta inteira)
resource "aws_s3_account_public_access_block" "main" {
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# 8. Budget padrão
resource "aws_budgets_budget" "account" {
  name         = "${var.account_name}-monthly"
  budget_type  = "COST"
  limit_amount = var.monthly_budget
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 80
    threshold_type            = "PERCENTAGE"
    notification_type         = "FORECASTED"
    subscriber_sns_topic_arns = [var.budget_sns_topic_arn]
  }

  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 100
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_sns_topic_arns = [var.budget_sns_topic_arn]
  }
}
```

---

## Delegated Administration

Serviços que suportam administração delegada (não precisa de management account):

| Serviço | Delegate To | Terraform |
|---------|-------------|-----------|
| **Security Hub** | Security Tooling | `aws_securityhub_organization_admin_account` |
| **GuardDuty** | Security Tooling | `aws_guardduty_organization_admin_account` |
| **IAM Access Analyzer** | Security Tooling | `aws_accessanalyzer_analyzer` (org scope) |
| **AWS Config** | Security Tooling | `aws_config_configuration_aggregator` |
| **Firewall Manager** | Security Tooling | `aws_fms_admin_account` |
| **AWS Backup** | Infrastructure | `aws_backup_global_settings` |
| **CloudFormation StackSets** | Infrastructure | `aws_cloudformation_stack_set` (service_managed) |

```hcl
# Delegar Security Hub para Security Tooling Account
resource "aws_securityhub_organization_admin_account" "main" {
  admin_account_id = var.security_tooling_account_id
}

# Delegar GuardDuty
resource "aws_guardduty_organization_admin_account" "main" {
  admin_account_id = var.security_tooling_account_id
}
```

---

## Tag Policies

```hcl
resource "aws_organizations_policy" "tag_policy" {
  name = "StandardTagPolicy"
  type = "TAG_POLICY"

  content = jsonencode({
    tags = {
      Environment = {
        tag_key = {
          "@@assign" = "Environment"
        }
        tag_value = {
          "@@assign" = ["production", "staging", "development", "sandbox"]
        }
        enforced_for = {
          "@@assign" = [
            "ec2:instance",
            "ec2:volume",
            "rds:db",
            "s3:bucket",
            "lambda:function",
            "ecs:service"
          ]
        }
      }
      Project = {
        tag_key = {
          "@@assign" = "Project"
        }
      }
      Team = {
        tag_key = {
          "@@assign" = "Team"
        }
      }
      CostCenter = {
        tag_key = {
          "@@assign" = "CostCenter"
        }
      }
    }
  })
}
```

---

## Multi-Account Checklist

### Organization Setup

- [ ] AWS Organizations com ALL features habilitado
- [ ] OU structure definida (Security, Infrastructure, Workloads, Sandbox)
- [ ] Management account apenas para billing e Organizations (sem workloads)
- [ ] SCPs aplicadas por OU (deny regions, protect security controls)
- [ ] Tag Policies configuradas e enforced

### Security

- [ ] CloudTrail organization trail (todas as contas)
- [ ] GuardDuty delegado para Security Tooling
- [ ] Security Hub delegado com standards habilitados
- [ ] IAM Identity Center configurado (SSO)
- [ ] Config Aggregator para compliance centralizado
- [ ] Access Analyzer scope organization

### Networking

- [ ] Transit Gateway na Network Hub Account
- [ ] RAM sharing para compartilhar TGW
- [ ] Route tables segmentadas (prod ↔ nonprod isolados)
- [ ] Centralized egress (NAT/Firewall na Network Account)
- [ ] DNS centralizado (Route 53 Resolver rules compartilhadas)

### Account Vending

- [ ] Processo automatizado de criação de contas
- [ ] Baseline aplicado automaticamente (VPC, Config, GuardDuty, Budgets)
- [ ] Default security controls (EBS encryption, S3 block public)
- [ ] Budget alerts por conta

---

## Referências

- [AWS Multi-Account Strategy](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/)
- [AWS Organizations User Guide](https://docs.aws.amazon.com/organizations/latest/userguide/)
- [AWS Control Tower](https://docs.aws.amazon.com/controltower/latest/userguide/)
- [SCP Examples](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_examples.html)
- [Transit Gateway Best Practices](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-best-design-practices.html)
