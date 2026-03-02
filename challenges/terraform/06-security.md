# Level 6 — Security

> **Objetivo:** Implementar segurança enterprise-grade na infraestrutura ShopFlow — IAM com least privilege, KMS para criptografia, Secrets Manager e SSM Parameter Store para gestão de segredos, encryption at rest/in transit, compliance scanning automatizado e permissions boundaries.

---

## Objetivo de Aprendizado

- Criar IAM roles e policies com princípio de menor privilégio
- Implementar KMS keys para criptografia de dados
- Gerenciar segredos com Secrets Manager e SSM Parameter Store
- Criptografar dados em repouso (S3, DynamoDB, SQS) e em trânsito
- Configurar pipeline de compliance scanning (tfsec + Checkov + OPA)
- Implementar permissions boundaries para limitar escopo
- Gerar relatórios de conformidade automatizados
- Aplicar políticas de segurança como código

---

## Escopo Funcional

### ShopFlow — Camadas de Segurança

```
┌──────────────────────────────────────────────────────────┐
│                    COMPLIANCE LAYER                       │
│  ┌─────────┐  ┌──────────┐  ┌───────────┐  ┌─────────┐ │
│  │  tfsec  │  │ Checkov  │  │ Conftest  │  │ Infracost│ │
│  └─────────┘  └──────────┘  └───────────┘  └─────────┘ │
├──────────────────────────────────────────────────────────┤
│                    IDENTITY LAYER                         │
│                                                           │
│  ┌────────────────────────────────────────────────────┐  │
│  │              Permissions Boundary                  │  │
│  │   ┌───────────────┐    ┌──────────────────────┐    │  │
│  │   │ Service Roles  │    │   CI/CD Roles        │    │  │
│  │   │ - lambda-exec  │    │ - terraform-deploy   │    │  │
│  │   │ - api-gw-exec  │    │ - terraform-plan     │    │  │
│  │   │ - event-proc   │    │ - terraform-reader   │    │  │
│  │   └───────────────┘    └──────────────────────┘    │  │
│  └────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────┤
│                    ENCRYPTION LAYER                       │
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐ │
│  │   KMS Keys   │  │   S3 SSE-KMS │  │  DynamoDB KMS  │ │
│  │ - storage    │  │   Versioning │  │  Encryption    │ │
│  │ - messaging  │  │   Lifecycle  │  │                │ │
│  │ - compute    │  │              │  │                │ │
│  └──────────────┘  └──────────────┘  └────────────────┘ │
├──────────────────────────────────────────────────────────┤
│                    SECRETS LAYER                          │
│                                                           │
│  ┌──────────────────┐  ┌──────────────────────────────┐ │
│  │ Secrets Manager  │  │   SSM Parameter Store        │ │
│  │ - db-credentials │  │   - /shopflow/dev/api_url    │ │
│  │ - api-keys       │  │   - /shopflow/dev/vpc_id     │ │
│  │ - jwt-secrets    │  │   - /shopflow/dev/db_host    │ │
│  └──────────────────┘  └──────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

## Escopo Técnico

### Estrutura de Diretórios

```
live/
├── _global/
│   ├── iam/
│   │   ├── main.tf
│   │   ├── policies/
│   │   │   ├── lambda-exec.json
│   │   │   ├── api-gateway.json
│   │   │   ├── ci-deploy.json
│   │   │   └── permissions-boundary.json
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── backend.tf
│   └── kms/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── backend.tf
│
└── dev/
    └── secrets/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── backend.tf

modules/
├── security/
│   ├── iam-role/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── kms-key/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   └── secret/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── README.md
```

---

## Tarefas

### Tarefa 1 — Módulo IAM Role (Least Privilege)

```hcl
# modules/security/iam-role/variables.tf
variable "role_name" {
  description = "Name of the IAM role"
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]+$", var.role_name))
    error_message = "Role name must be lowercase alphanumeric with hyphens."
  }
}

variable "assume_role_principals" {
  description = "Map of principal types to identifiers for assume role"
  type = object({
    service    = optional(list(string), [])
    aws        = optional(list(string), [])
    federated  = optional(list(string), [])
  })

  validation {
    condition = (
      length(var.assume_role_principals.service) > 0 ||
      length(var.assume_role_principals.aws) > 0 ||
      length(var.assume_role_principals.federated) > 0
    )
    error_message = "At least one principal must be specified."
  }
}

variable "inline_policies" {
  description = "Map of inline policy names to policy documents"
  type        = map(string)
  default     = {}
}

variable "managed_policy_arns" {
  description = "List of managed policy ARNs to attach"
  type        = list(string)
  default     = []

  validation {
    condition     = alltrue([for arn in var.managed_policy_arns : can(regex("^arn:aws:iam::", arn))])
    error_message = "All managed policy ARNs must be valid IAM ARNs."
  }
}

variable "permissions_boundary_arn" {
  description = "ARN of the permissions boundary to attach"
  type        = string
  default     = null
}

variable "max_session_duration" {
  description = "Maximum session duration in seconds"
  type        = number
  default     = 3600

  validation {
    condition     = var.max_session_duration >= 3600 && var.max_session_duration <= 43200
    error_message = "Session duration must be between 1 hour and 12 hours."
  }
}

variable "tags" {
  description = "Tags to apply to the role"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/security/iam-role/main.tf
data "aws_iam_policy_document" "assume_role" {
  statement {
    actions = ["sts:AssumeRole"]

    dynamic "principals" {
      for_each = {
        for type, ids in var.assume_role_principals : type => ids
        if length(ids) > 0
      }
      content {
        type        = title(principals.key)
        identifiers = principals.value
      }
    }
  }
}

resource "aws_iam_role" "this" {
  name                 = var.role_name
  assume_role_policy   = data.aws_iam_policy_document.assume_role.json
  max_session_duration = var.max_session_duration
  permissions_boundary = var.permissions_boundary_arn

  tags = merge(var.tags, {
    ManagedBy = "terraform"
  })
}

resource "aws_iam_role_policy" "inline" {
  for_each = var.inline_policies

  name   = each.key
  role   = aws_iam_role.this.id
  policy = each.value
}

resource "aws_iam_role_policy_attachment" "managed" {
  for_each = toset(var.managed_policy_arns)

  role       = aws_iam_role.this.name
  policy_arn = each.value
}
```

```hcl
# modules/security/iam-role/outputs.tf
output "role_arn" {
  description = "ARN of the IAM role"
  value       = aws_iam_role.this.arn
}

output "role_name" {
  description = "Name of the IAM role"
  value       = aws_iam_role.this.name
}

output "role_id" {
  description = "Unique ID of the IAM role"
  value       = aws_iam_role.this.unique_id
}
```

### Tarefa 2 — Roles de Serviço ShopFlow

Crie as roles para cada componente da plataforma:

```hcl
# live/_global/iam/main.tf

# Policy document para Lambda processar pedidos
data "aws_iam_policy_document" "lambda_order_processor" {
  # Ler/Escrever no DynamoDB (tabelas específicas)
  statement {
    sid    = "DynamoDBAccess"
    effect = "Allow"
    actions = [
      "dynamodb:GetItem",
      "dynamodb:PutItem",
      "dynamodb:UpdateItem",
      "dynamodb:Query",
    ]
    resources = [
      "arn:aws:dynamodb:us-east-1:*:table/shopflow-*-orders",
      "arn:aws:dynamodb:us-east-1:*:table/shopflow-*-orders/index/*",
    ]
  }

  # Enviar mensagens para SQS
  statement {
    sid    = "SQSSendMessage"
    effect = "Allow"
    actions = [
      "sqs:SendMessage",
      "sqs:GetQueueAttributes",
    ]
    resources = [
      "arn:aws:sqs:us-east-1:*:shopflow-*-notifications",
    ]
  }

  # Publicar em SNS
  statement {
    sid    = "SNSPublish"
    effect = "Allow"
    actions = [
      "sns:Publish",
    ]
    resources = [
      "arn:aws:sns:us-east-1:*:shopflow-*-order-events",
    ]
  }

  # Logs (obrigatório para Lambda)
  statement {
    sid    = "CloudWatchLogs"
    effect = "Allow"
    actions = [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents",
    ]
    resources = ["arn:aws:logs:us-east-1:*:*"]
  }

  # KMS para decrypt de segredos
  statement {
    sid    = "KMSDecrypt"
    effect = "Allow"
    actions = [
      "kms:Decrypt",
      "kms:DescribeKey",
    ]
    resources = [
      "arn:aws:kms:us-east-1:*:key/*",
    ]
    condition {
      test     = "StringEquals"
      variable = "kms:ViaService"
      values   = ["secretsmanager.us-east-1.amazonaws.com"]
    }
  }
}

module "lambda_order_processor_role" {
  source = "../../../modules/security/iam-role"

  role_name = "shopflow-lambda-order-processor"
  assume_role_principals = {
    service = ["lambda.amazonaws.com"]
  }
  inline_policies = {
    "order-processor" = data.aws_iam_policy_document.lambda_order_processor.json
  }
  permissions_boundary_arn = aws_iam_policy.permissions_boundary.arn

  tags = local.common_tags
}

# Role para API Gateway
module "api_gateway_role" {
  source = "../../../modules/security/iam-role"

  role_name = "shopflow-api-gateway"
  assume_role_principals = {
    service = ["apigateway.amazonaws.com"]
  }
  inline_policies = {
    "invoke-lambda" = data.aws_iam_policy_document.api_gw_invoke_lambda.json
  }
  permissions_boundary_arn = aws_iam_policy.permissions_boundary.arn

  tags = local.common_tags
}

# Role para Event Processor Lambda (SQS → Lambda)
module "lambda_event_processor_role" {
  source = "../../../modules/security/iam-role"

  role_name = "shopflow-lambda-event-processor"
  assume_role_principals = {
    service = ["lambda.amazonaws.com"]
  }
  inline_policies = {
    "event-processor" = data.aws_iam_policy_document.lambda_event_processor.json
  }
  managed_policy_arns = [
    "arn:aws:iam::policy/service-role/AWSLambdaSQSQueueExecutionRole"
  ]
  permissions_boundary_arn = aws_iam_policy.permissions_boundary.arn

  tags = local.common_tags
}
```

### Tarefa 3 — Permissions Boundary

```hcl
# live/_global/iam/boundary.tf

data "aws_iam_policy_document" "permissions_boundary" {
  # Permitir acesso apenas a recursos ShopFlow
  statement {
    sid    = "AllowShopFlowResources"
    effect = "Allow"
    actions = [
      "s3:*",
      "dynamodb:*",
      "sqs:*",
      "sns:*",
      "lambda:*",
      "logs:*",
      "ssm:GetParameter*",
      "secretsmanager:GetSecretValue",
      "kms:Decrypt",
      "kms:DescribeKey",
      "kms:GenerateDataKey*",
    ]
    resources = ["*"]

    condition {
      test     = "StringLike"
      variable = "aws:ResourceTag/Project"
      values   = ["shopflow"]
    }
  }

  # Negar acesso a recursos críticos
  statement {
    sid    = "DenyDestructiveActions"
    effect = "Deny"
    actions = [
      "iam:CreateUser",
      "iam:DeleteUser",
      "iam:CreateAccessKey",
      "organizations:*",
      "account:*",
    ]
    resources = ["*"]
  }

  # Negar alterar o próprio boundary
  statement {
    sid    = "DenyBoundaryModification"
    effect = "Deny"
    actions = [
      "iam:DeleteRolePermissionsBoundary",
      "iam:PutRolePermissionsBoundary",
    ]
    resources = ["*"]
    condition {
      test     = "StringEquals"
      variable = "iam:PermissionsBoundary"
      values   = ["arn:aws:iam::policy/shopflow-permissions-boundary"]
    }
  }
}

resource "aws_iam_policy" "permissions_boundary" {
  name        = "shopflow-permissions-boundary"
  description = "Permissions boundary for all ShopFlow roles"
  policy      = data.aws_iam_policy_document.permissions_boundary.json

  tags = local.common_tags
}
```

### Tarefa 4 — Módulo KMS Key

```hcl
# modules/security/kms-key/variables.tf
variable "alias" {
  description = "Alias for the KMS key (without alias/ prefix)"
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9/-]+$", var.alias))
    error_message = "KMS alias must be lowercase alphanumeric with hyphens and slashes."
  }
}

variable "description" {
  description = "Description of the KMS key"
  type        = string
}

variable "deletion_window_in_days" {
  description = "Waiting period before KMS key deletion (days)"
  type        = number
  default     = 30

  validation {
    condition     = var.deletion_window_in_days >= 7 && var.deletion_window_in_days <= 30
    error_message = "Deletion window must be between 7 and 30 days."
  }
}

variable "enable_key_rotation" {
  description = "Enable automatic key rotation"
  type        = bool
  default     = true
}

variable "policy" {
  description = "Custom key policy JSON (optional)"
  type        = string
  default     = null
}

variable "service_principals" {
  description = "AWS service principals allowed to use this key"
  type        = list(string)
  default     = []
}

variable "tags" {
  description = "Tags to apply to the key"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/security/kms-key/main.tf
data "aws_caller_identity" "current" {}

resource "aws_kms_key" "this" {
  description             = var.description
  deletion_window_in_days = var.deletion_window_in_days
  enable_key_rotation     = var.enable_key_rotation
  policy                  = var.policy

  tags = merge(var.tags, {
    ManagedBy = "terraform"
  })
}

resource "aws_kms_alias" "this" {
  name          = "alias/${var.alias}"
  target_key_id = aws_kms_key.this.key_id
}
```

```hcl
# modules/security/kms-key/outputs.tf
output "key_id" {
  description = "ID of the KMS key"
  value       = aws_kms_key.this.key_id
}

output "key_arn" {
  description = "ARN of the KMS key"
  value       = aws_kms_key.this.arn
}

output "alias_arn" {
  description = "ARN of the KMS key alias"
  value       = aws_kms_alias.this.arn
}

output "alias_name" {
  description = "Name of the KMS key alias"
  value       = aws_kms_alias.this.name
}
```

### Tarefa 5 — KMS Keys da ShopFlow

```hcl
# live/_global/kms/main.tf

# Key para dados de storage (S3)
module "storage_key" {
  source = "../../../modules/security/kms-key"

  alias       = "shopflow/storage"
  description = "KMS key for ShopFlow storage encryption (S3)"

  service_principals = [
    "s3.amazonaws.com",
  ]

  tags = merge(local.common_tags, {
    Domain = "storage"
  })
}

# Key para dados (DynamoDB)
module "data_key" {
  source = "../../../modules/security/kms-key"

  alias       = "shopflow/data"
  description = "KMS key for ShopFlow data encryption (DynamoDB)"

  service_principals = [
    "dynamodb.amazonaws.com",
  ]

  tags = merge(local.common_tags, {
    Domain = "data"
  })
}

# Key para messaging (SQS/SNS)
module "messaging_key" {
  source = "../../../modules/security/kms-key"

  alias       = "shopflow/messaging"
  description = "KMS key for ShopFlow messaging encryption (SQS/SNS)"

  service_principals = [
    "sqs.amazonaws.com",
    "sns.amazonaws.com",
  ]

  tags = merge(local.common_tags, {
    Domain = "messaging"
  })
}

# Key para secrets (Secrets Manager)
module "secrets_key" {
  source = "../../../modules/security/kms-key"

  alias       = "shopflow/secrets"
  description = "KMS key for ShopFlow secrets encryption"

  service_principals = [
    "secretsmanager.amazonaws.com",
  ]

  tags = merge(local.common_tags, {
    Domain = "secrets"
  })
}
```

### Tarefa 6 — Módulo de Secrets e Aplicação

```hcl
# modules/security/secret/main.tf
resource "aws_secretsmanager_secret" "this" {
  name        = var.name
  description = var.description
  kms_key_id  = var.kms_key_id

  recovery_window_in_days = var.recovery_window_in_days

  tags = merge(var.tags, {
    ManagedBy = "terraform"
  })
}

resource "aws_secretsmanager_secret_version" "this" {
  count = var.secret_string != null ? 1 : 0

  secret_id     = aws_secretsmanager_secret.this.id
  secret_string = var.secret_string
}
```

```hcl
# live/dev/secrets/main.tf

# Segredos da aplicação
module "db_credentials" {
  source = "../../../modules/security/secret"

  name        = "shopflow/dev/db-credentials"
  description = "Database credentials for ShopFlow dev environment"
  kms_key_id  = data.terraform_remote_state.kms.outputs.secrets_key_id

  secret_string = jsonencode({
    username = "shopflow_app"
    password = "dev-only-password-change-in-prod"
    host     = "shopflow-dev-db.local"
    port     = 5432
    dbname   = "shopflow"
  })

  tags = local.common_tags
}

module "api_keys" {
  source = "../../../modules/security/secret"

  name        = "shopflow/dev/api-keys"
  description = "External API keys for ShopFlow dev environment"
  kms_key_id  = data.terraform_remote_state.kms.outputs.secrets_key_id

  secret_string = jsonencode({
    payment_gateway_key = "pk_test_shopflow_dev"
    email_service_key   = "sg_test_shopflow_dev"
    analytics_key       = "ak_test_shopflow_dev"
  })

  tags = local.common_tags
}

module "jwt_secret" {
  source = "../../../modules/security/secret"

  name        = "shopflow/dev/jwt-secret"
  description = "JWT signing secret for ShopFlow dev environment"
  kms_key_id  = data.terraform_remote_state.kms.outputs.secrets_key_id

  secret_string = jsonencode({
    signing_key   = "dev-jwt-signing-key-256-bits-long-ok"
    refresh_key   = "dev-jwt-refresh-key-256-bits-long-ok"
    expiry_hours  = 24
  })

  tags = local.common_tags
}

# SSM Parameters para configuração não-sensível
resource "aws_ssm_parameter" "api_url" {
  name  = "/shopflow/dev/api_url"
  type  = "String"
  value = "http://localhost:4566/restapis"

  tags = local.common_tags
}

resource "aws_ssm_parameter" "feature_flags" {
  name  = "/shopflow/dev/feature_flags"
  type  = "String"
  value = jsonencode({
    enable_new_checkout  = true
    enable_loyalty       = false
    maintenance_mode     = false
  })

  tags = local.common_tags
}
```

### Tarefa 7 — Encryption at Rest em Todos os Recursos

Atualize os módulos existentes para usar KMS:

```hcl
# Módulo S3 — Encryption com KMS
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = var.kms_key_arn
      sse_algorithm     = var.kms_key_arn != null ? "aws:kms" : "AES256"
    }
    bucket_key_enabled = var.kms_key_arn != null
  }
}

# Módulo DynamoDB — Encryption com KMS
resource "aws_dynamodb_table" "this" {
  name         = var.table_name
  billing_mode = var.billing_mode
  hash_key     = var.hash_key
  range_key    = var.range_key

  server_side_encryption {
    enabled     = true
    kms_key_arn = var.kms_key_arn
  }

  point_in_time_recovery {
    enabled = true
  }

  # ... attributes, GSI, etc.
}

# Módulo SQS — Encryption com KMS
resource "aws_sqs_queue" "this" {
  name = var.queue_name

  kms_master_key_id                 = var.kms_key_id
  kms_data_key_reuse_period_seconds = 300

  # ... DLQ, visibility timeout, etc.
}

# Módulo SNS — Encryption com KMS
resource "aws_sns_topic" "this" {
  name = var.topic_name

  kms_master_key_id = var.kms_key_id

  # ... subscriptions, etc.
}
```

### Tarefa 8 — Pipeline de Compliance Scanning

```yaml
# .github/workflows/security-scan.yml
name: Security & Compliance Scan

on:
  pull_request:
    paths: ['**.tf', '**.tfvars']

env:
  TERRAFORM_VERSION: "1.10.0"

jobs:
  tfsec-scan:
    name: tfsec Static Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: tfsec scan
        uses: aquasecurity/tfsec-action@v1.0.3
        with:
          working_directory: .
          soft_fail: false
          additional_args: "--format json --out tfsec-results.json"

      - name: Upload tfsec results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: tfsec-results
          path: tfsec-results.json

  checkov-scan:
    name: Checkov Compliance Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Checkov scan
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: .
          framework: terraform
          output_format: json
          output_file_path: checkov-results.json
          soft_fail: false
          skip_check: CKV_AWS_18  # S3 access logging (LocalStack não suporta)

      - name: Upload Checkov results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: checkov-results
          path: checkov-results.json

  opa-policy-check:
    name: OPA Policy Validation
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TERRAFORM_VERSION }}

      - name: Generate plan JSON
        run: |
          cd live/dev/storage
          terraform init -backend=false
          terraform plan -out=plan.tfplan -input=false || true
          terraform show -json plan.tfplan > plan.json

      - name: Install Conftest
        run: |
          LATEST=$(curl -s https://api.github.com/repos/open-policy-agent/conftest/releases/latest | jq -r .tag_name | sed 's/v//')
          curl -sL "https://github.com/open-policy-agent/conftest/releases/download/v${LATEST}/conftest_${LATEST}_Linux_x86_64.tar.gz" | tar xz
          sudo mv conftest /usr/local/bin/

      - name: Run OPA policies
        run: |
          conftest test live/dev/storage/plan.json \
            --policy policy/ \
            --output json > opa-results.json

      - name: Upload OPA results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: opa-results
          path: opa-results.json

  compliance-report:
    name: Generate Compliance Report
    needs: [tfsec-scan, checkov-scan, opa-policy-check]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4

      - name: Generate report
        run: |
          echo "# ShopFlow Security Compliance Report" > report.md
          echo "Generated: $(date -u)" >> report.md
          echo "" >> report.md

          echo "## tfsec Results" >> report.md
          if [ -f tfsec-results/tfsec-results.json ]; then
            CRITICAL=$(jq '[.results[] | select(.severity == "CRITICAL")] | length' tfsec-results/tfsec-results.json)
            HIGH=$(jq '[.results[] | select(.severity == "HIGH")] | length' tfsec-results/tfsec-results.json)
            echo "- Critical: $CRITICAL" >> report.md
            echo "- High: $HIGH" >> report.md
          fi

          echo "" >> report.md
          echo "## Checkov Results" >> report.md
          if [ -f checkov-results/checkov-results.json ]; then
            PASSED=$(jq '.results.passed_checks | length' checkov-results/checkov-results.json)
            FAILED=$(jq '.results.failed_checks | length' checkov-results/checkov-results.json)
            echo "- Passed: $PASSED" >> report.md
            echo "- Failed: $FAILED" >> report.md
          fi

          cat report.md

      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: compliance-report
          path: report.md
```

### Tarefa 9 — Políticas OPA de Segurança

```rego
# policy/security/encryption.rego
package security.encryption

import rego.v1

# S3 buckets devem ter encryption com KMS
deny contains msg if {
  some resource in input.resource_changes
  resource.type == "aws_s3_bucket_server_side_encryption_configuration"
  rule := resource.change.after.rule[_]
  sse := rule.apply_server_side_encryption_by_default[_]
  sse.sse_algorithm != "aws:kms"
  msg := sprintf("S3 bucket '%s' must use KMS encryption (aws:kms), got '%s'", [resource.address, sse.sse_algorithm])
}

# DynamoDB tables devem ter encryption
deny contains msg if {
  some resource in input.resource_changes
  resource.type == "aws_dynamodb_table"
  not has_encryption(resource)
  msg := sprintf("DynamoDB table '%s' must have server-side encryption enabled", [resource.address])
}

has_encryption(resource) if {
  resource.change.after.server_side_encryption[_].enabled == true
}

# SQS queues devem ter KMS encryption
deny contains msg if {
  some resource in input.resource_changes
  resource.type == "aws_sqs_queue"
  not resource.change.after.kms_master_key_id
  msg := sprintf("SQS queue '%s' must have KMS encryption configured", [resource.address])
}
```

```rego
# policy/security/iam.rego
package security.iam

import rego.v1

# IAM policies não devem usar wildcard actions
deny contains msg if {
  some resource in input.resource_changes
  resource.type == "aws_iam_role_policy"
  policy := json.unmarshal(resource.change.after.policy)
  statement := policy.Statement[_]
  statement.Effect == "Allow"
  action := statement.Action[_]
  action == "*"
  msg := sprintf("IAM policy '%s' must not use wildcard (*) actions", [resource.address])
}

# IAM policies não devem usar wildcard resources (exceto logs)
deny contains msg if {
  some resource in input.resource_changes
  resource.type == "aws_iam_role_policy"
  policy := json.unmarshal(resource.change.after.policy)
  statement := policy.Statement[_]
  statement.Effect == "Allow"
  resource_arn := statement.Resource[_]
  resource_arn == "*"
  not is_log_action(statement)
  msg := sprintf("IAM policy '%s' uses wildcard resource. Restrict to specific ARNs.", [resource.address])
}

is_log_action(statement) if {
  action := statement.Action[_]
  startswith(action, "logs:")
}

# Roles devem ter permissions boundary
deny contains msg if {
  some resource in input.resource_changes
  resource.type == "aws_iam_role"
  not resource.change.after.permissions_boundary
  msg := sprintf("IAM role '%s' must have a permissions boundary attached", [resource.address])
}
```

```rego
# policy/security/secrets.rego
package security.secrets

import rego.v1

# Secrets Manager deve usar KMS
deny contains msg if {
  some resource in input.resource_changes
  resource.type == "aws_secretsmanager_secret"
  not resource.change.after.kms_key_id
  msg := sprintf("Secret '%s' must use a customer-managed KMS key", [resource.address])
}

# Recovery window deve ser >= 7 dias
deny contains msg if {
  some resource in input.resource_changes
  resource.type == "aws_secretsmanager_secret"
  resource.change.after.recovery_window_in_days < 7
  msg := sprintf("Secret '%s' recovery window must be >= 7 days", [resource.address])
}

# SSM SecureString parameters devem usar KMS
warn contains msg if {
  some resource in input.resource_changes
  resource.type == "aws_ssm_parameter"
  resource.change.after.type == "SecureString"
  not resource.change.after.key_id
  msg := sprintf("SSM parameter '%s' (SecureString) should use a customer-managed KMS key", [resource.address])
}
```

---

## Critérios de Aceite

- [ ] Módulo `iam-role` criado e testado com pelo menos 3 roles
- [ ] Roles de serviço para Lambda, API Gateway e Event Processor criadas
- [ ] Permissions boundary implementada e aplicada a todas as roles
- [ ] Módulo `kms-key` criado com pelo menos 4 keys (storage, data, messaging, secrets)
- [ ] Módulo `secret` criado com Secrets Manager e SSM Parameter Store
- [ ] Segredos da ShopFlow criados (db-credentials, api-keys, jwt-secret)
- [ ] Encryption at rest habilitada em todos os recursos (S3, DynamoDB, SQS, SNS)
- [ ] Pipeline de compliance scanning configurado (tfsec + Checkov + OPA)
- [ ] Políticas OPA para encryption, IAM e secrets implementadas
- [ ] Zero findings Critical/High no tfsec e Checkov
- [ ] Todos os recursos com tags de segurança

---

## Definição de Pronto (DoD)

- [ ] Todos os módulos de segurança com `terraform-docs` gerado
- [ ] Testes para módulos de segurança (terraform test ou Terratest)
- [ ] Pipeline de compliance funcional
- [ ] Políticas OPA cobrindo encryption, IAM e secrets
- [ ] Documentação de segurança (README com decisões e justificativas)
- [ ] Commit semântico: `feat(level-6): implement security with IAM, KMS, secrets and compliance`

---

## Checklist

- [ ] Módulo `iam-role` funcional
- [ ] Roles de serviço criadas com least privilege
- [ ] Permissions boundary aplicada
- [ ] Módulo `kms-key` funcional
- [ ] Keys KMS provisionadas
- [ ] Módulo `secret` funcional
- [ ] Segredos ShopFlow criados
- [ ] SSM Parameters configurados
- [ ] Encryption at rest em todos os serviços
- [ ] tfsec scan passando
- [ ] Checkov scan passando
- [ ] Políticas OPA implementadas
- [ ] Pipeline CI de compliance

---

## Anti-Patterns de Segurança

| Anti-Pattern | Risco | Alternativa |
|-------------|-------|-------------|
| **IAM `*` actions** | Acesso irrestrito | Actions específicas por operação |
| **IAM `*` resources** | Blast radius máximo | ARNs específicos com wildcards controlados |
| **Secrets em `.tf` ou `.tfvars`** | Exposição em VCS | Secrets Manager ou SSM SecureString |
| **KMS key sem rotation** | Comprometimento crescente | `enable_key_rotation = true` |
| **Sem permissions boundary** | Privilege escalation | Boundary em todas as roles |
| **Encryption com AES256** | Sem controle de chaves | SSE-KMS com CMK |
| **Security scanning manual** | Inconsistente, esquecido | Pipeline CI automatizado |
| **Hardcoded credentials** | Exposição total | Provider assume role ou env vars |

---

## Extensões Opcionais

- [ ] Implementar rotação automática de secrets com Lambda
- [ ] Criar policies de S3 bucket com condition keys
- [ ] Implementar VPC endpoints para acesso privado a S3, DynamoDB, SQS
- [ ] Configurar CloudTrail para auditoria de KMS e IAM
- [ ] Criar dashboard de compliance com métricas agregadas
- [ ] Implementar cross-account access com assume role
- [ ] Adicionar SCPs (Service Control Policies) simuladas
- [ ] Configurar alertas SNS para violações de compliance
