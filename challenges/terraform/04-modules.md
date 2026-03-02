# Level 4 — Módulos

> **Objetivo:** Criar módulos Terraform reutilizáveis, compostos e bem documentados — seguindo princípios de design, versionamento semântico e patterns de composição. Publicar e consumir módulos como um time de plataforma faria.

---

## Objetivo de Aprendizado

- Diferenciar módulos atômicos vs compostos e quando usar cada um
- Criar módulos com interfaces minimalistas e defaults seguros
- Implementar composição de módulos (módulos que chamam outros módulos)
- Aplicar versionamento semântico em módulos
- Gerar documentação automática com terraform-docs
- Criar exemplos de uso (`examples/`) para cada módulo
- Consumir módulos da comunidade (Terraform Registry) com segurança
- Entender e evitar anti-patterns em módulos
- Implementar testes completos em cada módulo

---

## Escopo Funcional

### ShopFlow — Catálogo de Módulos de Plataforma

Crie um catálogo completo de módulos para a plataforma ShopFlow, simulando o trabalho de um time de plataforma que oferece "building blocks" padronizados para times de produto:

| Categoria | Módulo | Tipo | Descrição |
|-----------|--------|------|-----------|
| **Networking** | `vpc` | Atômico | VPC + Subnets + IGW + Route Tables |
| **Networking** | `security-group` | Atômico | Security Group com regras dinâmicas |
| **Storage** | `s3-bucket` | Atômico | S3 com versionamento, encryption, public access block |
| **Data** | `dynamodb-table` | Atômico | DynamoDB com GSI, TTL, encryption |
| **Messaging** | `sqs-queue` | Atômico | SQS + DLQ opcional + encryption |
| **Messaging** | `sns-topic` | Atômico | SNS Topic + subscriptions |
| **Compute** | `lambda-function` | Atômico | Lambda + IAM Role + CloudWatch Logs |
| **Compute** | `api-gateway` | Atômico | API Gateway REST + stages |
| **Composed** | `api-backend` | Composto | Lambda + API Gateway + SQS + DynamoDB |
| **Composed** | `event-pipeline` | Composto | SNS → SQS → Lambda → DynamoDB |

---

## Escopo Técnico

### Anatomia de um Módulo Bem Construído

```
modules/compute/lambda-function/
├── main.tf                    # Recursos principais
├── iam.tf                     # IAM Role e policies da Lambda
├── variables.tf               # Interface de entrada (inputs)
├── outputs.tf                 # Interface de saída (outputs)
├── versions.tf                # Required providers
├── locals.tf                  # Computações internas
├── README.md                  # Gerado por terraform-docs
├── CHANGELOG.md               # Histórico de mudanças (semver)
├── examples/
│   ├── basic/
│   │   ├── main.tf            # Uso mínimo do módulo
│   │   └── README.md
│   └── complete/
│       ├── main.tf            # Uso com todas as opções
│       └── README.md
└── tests/
    ├── unit/
    │   ├── basic_test.tftest.hcl
    │   └── validation_test.tftest.hcl
    └── integration/
        └── full_deploy_test.tftest.hcl
```

### Princípios de Design de Módulos

| Princípio | Descrição |
|-----------|-----------|
| **Interface Mínima** | Poucas variáveis, com defaults sensatos |
| **Defaults Seguros** | Encryption, logging, access block habilitados por padrão |
| **Composição > Herança** | Módulos compostos chamam módulos atômicos |
| **Sem Provider/Backend** | O módulo NUNCA define `provider {}` nem `backend {}` |
| **Outputs Completos** | Exporte tudo que o caller pode precisar |
| **Documentação** | Gerada automaticamente por terraform-docs |
| **Testes** | Unitários + integração em cada módulo |
| **Exemplos** | `basic/` e `complete/` para cada módulo |

---

## Tarefas

### Tarefa 1 — Módulo Atômico: `lambda-function`

Crie o módulo completo para provisionamento de Lambda Functions:

**`main.tf`:**

```hcl
resource "aws_lambda_function" "this" {
  function_name = var.function_name
  runtime       = var.runtime
  handler       = var.handler
  role          = aws_iam_role.lambda.arn
  memory_size   = var.memory_size
  timeout       = var.timeout
  architectures = [var.architecture]

  filename         = var.zip_file
  source_code_hash = filebase64sha256(var.zip_file)

  environment {
    variables = var.environment_variables
  }

  dynamic "vpc_config" {
    for_each = var.vpc_config != null ? [var.vpc_config] : []
    content {
      subnet_ids         = vpc_config.value.subnet_ids
      security_group_ids = vpc_config.value.security_group_ids
    }
  }

  dynamic "dead_letter_config" {
    for_each = var.dead_letter_target_arn != null ? [1] : []
    content {
      target_arn = var.dead_letter_target_arn
    }
  }

  tags = merge(var.tags, {
    Name = var.function_name
  })
}

resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = var.log_retention_days

  tags = var.tags
}
```

**`iam.tf`:**

```hcl
resource "aws_iam_role" "lambda" {
  name = "${var.function_name}-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })

  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy" "custom" {
  count = var.custom_policy_json != null ? 1 : 0

  name   = "${var.function_name}-custom-policy"
  role   = aws_iam_role.lambda.id
  policy = var.custom_policy_json
}
```

**`variables.tf`:**

```hcl
variable "function_name" {
  description = "Name of the Lambda function"
  type        = string

  validation {
    condition     = can(regex("^[a-zA-Z0-9-_]+$", var.function_name))
    error_message = "Function name can only contain alphanumeric characters, hyphens and underscores."
  }
}

variable "runtime" {
  description = "Lambda runtime"
  type        = string
  default     = "python3.12"

  validation {
    condition     = contains(["python3.12", "python3.11", "nodejs20.x", "nodejs22.x", "java21", "go1.x", "provided.al2023"], var.runtime)
    error_message = "Runtime must be a supported Lambda runtime."
  }
}

variable "handler" {
  description = "Lambda handler (e.g., handler.main)"
  type        = string
}

variable "zip_file" {
  description = "Path to the Lambda deployment package (ZIP file)"
  type        = string
}

variable "memory_size" {
  description = "Lambda memory in MB"
  type        = number
  default     = 128

  validation {
    condition     = var.memory_size >= 128 && var.memory_size <= 10240 && var.memory_size % 64 == 0
    error_message = "Memory must be between 128-10240 MB and a multiple of 64."
  }
}

variable "timeout" {
  description = "Lambda timeout in seconds"
  type        = number
  default     = 30

  validation {
    condition     = var.timeout >= 1 && var.timeout <= 900
    error_message = "Timeout must be between 1 and 900 seconds."
  }
}

variable "architecture" {
  description = "Lambda architecture"
  type        = string
  default     = "x86_64"

  validation {
    condition     = contains(["x86_64", "arm64"], var.architecture)
    error_message = "Architecture must be x86_64 or arm64."
  }
}

variable "environment_variables" {
  description = "Environment variables for the Lambda function"
  type        = map(string)
  default     = {}
}

variable "vpc_config" {
  description = "VPC configuration for the Lambda function"
  type = object({
    subnet_ids         = list(string)
    security_group_ids = list(string)
  })
  default = null
}

variable "dead_letter_target_arn" {
  description = "ARN of the SQS queue or SNS topic for dead letter delivery"
  type        = string
  default     = null
}

variable "custom_policy_json" {
  description = "Custom IAM policy JSON for the Lambda role"
  type        = string
  default     = null
}

variable "log_retention_days" {
  description = "CloudWatch Log Group retention in days"
  type        = number
  default     = 14

  validation {
    condition     = contains([1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, 400, 545, 731, 1827, 3653], var.log_retention_days)
    error_message = "Retention days must be a valid CloudWatch retention value."
  }
}

variable "tags" {
  description = "Additional tags for all resources"
  type        = map(string)
  default     = {}
}
```

**`outputs.tf`:**

```hcl
output "function_name" {
  description = "Name of the Lambda function"
  value       = aws_lambda_function.this.function_name
}

output "function_arn" {
  description = "ARN of the Lambda function"
  value       = aws_lambda_function.this.arn
}

output "invoke_arn" {
  description = "Invoke ARN of the Lambda function"
  value       = aws_lambda_function.this.invoke_arn
}

output "role_arn" {
  description = "ARN of the Lambda execution role"
  value       = aws_iam_role.lambda.arn
}

output "role_name" {
  description = "Name of the Lambda execution role"
  value       = aws_iam_role.lambda.name
}

output "log_group_name" {
  description = "Name of the CloudWatch Log Group"
  value       = aws_cloudwatch_log_group.lambda.name
}

output "log_group_arn" {
  description = "ARN of the CloudWatch Log Group"
  value       = aws_cloudwatch_log_group.lambda.arn
}
```

**Exemplo de uso (`examples/basic/main.tf`):**

```hcl
module "order_processor" {
  source = "../../"

  function_name = "shopflow-dev-process-order"
  runtime       = "python3.12"
  handler       = "handler.process_order"
  zip_file      = "${path.module}/lambda.zip"

  environment_variables = {
    TABLE_NAME = "shopflow-dev-orders"
    QUEUE_URL  = "http://localhost:4566/000000000000/shopflow-dev-orders"
  }

  tags = {
    Environment = "dev"
    Project     = "shopflow"
  }
}
```

### Tarefa 2 — Módulo Atômico: `sns-topic`

Crie o módulo `modules/messaging/sns-topic/` com:

- SNS Topic principal
- Subscriptions configuráveis (SQS, Lambda, email) via `for_each`
- Dead letter policy opcional
- Encryption via KMS (opcional)
- Policy de acesso configurável

**Interface esperada:**

```hcl
module "order_notifications" {
  source = "../../../modules/messaging/sns-topic"

  name = "${local.name_prefix}-order-notifications"

  subscriptions = {
    sqs_orders = {
      protocol = "sqs"
      endpoint = module.orders_queue.queue_arn
    }
    lambda_notify = {
      protocol = "lambda"
      endpoint = module.notify_function.function_arn
    }
  }

  tags = local.common_tags
}
```

### Tarefa 3 — Módulo Atômico: `api-gateway`

Crie o módulo `modules/compute/api-gateway/` para API Gateway REST:

- REST API com stages configuráveis
- Integração com Lambda via proxy
- Throttling configurável
- CORS configurável
- Deploy automático

### Tarefa 4 — Módulo Composto: `api-backend`

Crie o módulo `modules/composed/api-backend/` que compõe módulos atômicos:

```hcl
# modules/composed/api-backend/main.tf

# Compõe: API Gateway + Lambda + DynamoDB + SQS
module "api" {
  source = "../../compute/api-gateway"

  name        = var.name
  description = var.description
  stage_name  = var.stage_name
}

module "handler" {
  source = "../../compute/lambda-function"

  function_name         = "${var.name}-handler"
  runtime               = var.runtime
  handler               = var.handler
  zip_file              = var.zip_file
  memory_size           = var.lambda_memory
  timeout               = var.lambda_timeout
  environment_variables = merge(var.environment_variables, {
    TABLE_NAME = module.table.table_name
    QUEUE_URL  = module.queue.queue_url
  })

  tags = var.tags
}

module "table" {
  source = "../../data/dynamodb-table"

  name         = "${var.name}-data"
  hash_key     = var.table_hash_key
  range_key    = var.table_range_key
  billing_mode = "PAY_PER_REQUEST"

  tags = var.tags
}

module "queue" {
  source = "../../messaging/sqs-queue"

  name       = "${var.name}-events"
  enable_dlq = true

  tags = var.tags
}
```

**Uso do módulo composto:**

```hcl
# live/dev/compute/main.tf
module "orders_api" {
  source = "../../../modules/composed/api-backend"

  name           = "${local.name_prefix}-orders"
  description    = "ShopFlow Orders API"
  stage_name     = var.environment
  runtime        = "python3.12"
  handler        = "handler.main"
  zip_file       = "${path.module}/lambda/orders.zip"
  table_hash_key = "order_id"

  environment_variables = {
    ENVIRONMENT = var.environment
  }

  tags = local.common_tags
}

module "catalog_api" {
  source = "../../../modules/composed/api-backend"

  name           = "${local.name_prefix}-catalog"
  description    = "ShopFlow Catalog API"
  stage_name     = var.environment
  runtime        = "python3.12"
  handler        = "handler.main"
  zip_file       = "${path.module}/lambda/catalog.zip"
  table_hash_key = "product_id"
  table_range_key = "category"

  environment_variables = {
    ENVIRONMENT = var.environment
  }

  tags = local.common_tags
}
```

### Tarefa 5 — Módulo Composto: `event-pipeline`

Crie o módulo `modules/composed/event-pipeline/` para o pattern:

```
SNS Topic → SQS Queue → Lambda Function → DynamoDB Table
```

Este módulo deve:
- Criar o SNS Topic
- Criar a SQS Queue com subscription no Topic
- Criar a Lambda Function com trigger na Queue
- Criar a DynamoDB Table destino
- Configurar todas as permissões IAM entre os serviços
- Criar a DLQ para falhas de processamento

### Tarefa 6 — Consumir Módulo da Comunidade

Use um módulo do Terraform Registry para complementar a infraestrutura:

```hcl
# Exemplo: Módulo da comunidade para VPC (estudo comparativo)
module "vpc_community" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${local.name_prefix}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.10.0/24", "10.0.11.0/24"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]

  enable_nat_gateway = false  # LocalStack não suporta NAT Gateway
  enable_dns_hostnames = true

  tags = local.common_tags
}
```

**Exercício:** Compare o módulo da comunidade com o seu módulo custom. Documente:
- Diferenças de interface (variáveis/outputs)
- Features que o módulo da comunidade oferece que o seu não tem
- Quando usar cada abordagem (custom vs community)

### Tarefa 7 — Documentação com terraform-docs

```bash
# Gerar README para cada módulo
find modules -name "versions.tf" -exec dirname {} \; | sort -u | while read dir; do
  echo "Generating docs for $dir"
  terraform-docs markdown table "$dir" --output-file README.md
done

# Verificar que a documentação está atualizada
terraform-docs markdown table modules/compute/lambda-function/ --output-check
```

### Tarefa 8 — Versionamento Semântico

Crie um `CHANGELOG.md` para o módulo `lambda-function`:

```markdown
# Changelog

All notable changes to this module will be documented in this file.

## [1.0.0] - 2026-03-01

### Added
- Initial release of the Lambda function module
- IAM role auto-creation with least privilege
- CloudWatch Log Group with configurable retention
- VPC configuration support
- Dead letter queue support
- Dynamic blocks for optional configurations
- Comprehensive validation on all inputs
- Unit and integration tests
- Documentation generated with terraform-docs

### Security
- Execution role follows least privilege principle
- Environment variables support for secrets (via SSM/Secrets Manager reference)
- Log group encryption support
```

---

## Critérios de Aceite

- [ ] Módulo `lambda-function` completo (main, iam, variables, outputs, versions, README, examples, tests)
- [ ] Módulo `sns-topic` completo com subscriptions via `for_each`
- [ ] Módulo `api-gateway` completo com integração Lambda
- [ ] Módulo composto `api-backend` que compõe 4 módulos atômicos
- [ ] Módulo composto `event-pipeline` (SNS → SQS → Lambda → DynamoDB)
- [ ] Todos os módulos com testes unitários e de integração
- [ ] Todos os módulos com `examples/basic/` e `examples/complete/`
- [ ] terraform-docs gerando README automático para todos os módulos
- [ ] Consumo de pelo menos 1 módulo da comunidade (Terraform Registry)
- [ ] CHANGELOG.md com versionamento semântico em pelo menos 2 módulos
- [ ] Todos os módulos provisionados com sucesso no LocalStack
- [ ] Módulos NÃO definem `provider {}` nem `backend {}`

---

## Definição de Pronto (DoD)

- [ ] Todos os módulos validam sem erros
- [ ] Todos os testes passam (unit + integration)
- [ ] Documentação gerada e atualizada
- [ ] Exemplos de uso funcionais
- [ ] CHANGELOG.md mantido
- [ ] Commit semântico: `feat(level-4): create reusable modules catalog with composition`

---

## Checklist

- [ ] Módulo `lambda-function` implementado e testado
- [ ] Módulo `sns-topic` implementado e testado
- [ ] Módulo `api-gateway` implementado e testado
- [ ] Módulo composto `api-backend` implementado e testado
- [ ] Módulo composto `event-pipeline` implementado e testado
- [ ] terraform-docs configurado e gerando READMEs
- [ ] Exemplos de uso criados para todos os módulos
- [ ] Módulo da comunidade consumido e comparado
- [ ] Anti-patterns documentados em DECISIONS.md

---

## Anti-Patterns a Evitar

| Anti-Pattern | Por que é ruim | Alternativa |
|-------------|----------------|-------------|
| **Módulo wrapper** — módulo com 1 recurso e mesmas variáveis | Não agrega valor, apenas adiciona indireção | Use o resource diretamente |
| **God module** — módulo com 50+ resources | Impossível de manter, testar e entender | Decomponha em módulos atômicos + composto |
| **Provider no módulo** | Impede o caller de configurar region, endpoints, etc. | Herde o provider do caller |
| **Backend no módulo** | State do módulo deve ser gerenciado pelo caller | Nunca coloque `backend {}` no módulo |
| **Variáveis sem validação** | Erros detectados apenas no apply | Adicione `validation {}` blocks |
| **Outputs sem description** | Inútil para quem consome o módulo | Sempre adicione `description` |
| **Hardcoded values** | Módulo inflexível | Use variáveis com defaults seguros |

---

## Extensões Opcionais

- [ ] Implementar um módulo `cloudwatch-alarm` e integrar com os módulos compostos
- [ ] Criar um módulo `event-rule` (EventBridge) para scheduling de Lambda
- [ ] Publicar um módulo em um registry privado (GitHub Packages ou S3)
- [ ] Implementar `precondition` e `postcondition` nos módulos
- [ ] Criar um módulo usando `for_each` com `tomap` para múltiplas instâncias
- [ ] Comparar 3 módulos da comunidade e documentar trade-offs
