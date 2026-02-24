````markdown
# Módulos — Terraform AWS

> **Objetivo:** Guia para criação, versionamento, publicação e consumo de módulos Terraform reutilizáveis,
> seguindo os padrões da HashiCorp e práticas adotadas por equipes de plataforma em empresas de grande porte.

---

## Sumário

- [Quando Criar um Módulo](#quando-criar-um-módulo)
- [Anatomia de um Módulo](#anatomia-de-um-módulo)
- [Convenções de Design](#convenções-de-design)
- [Módulos Compostos vs Atômicos](#módulos-compostos-vs-atômicos)
- [Versionamento Semântico](#versionamento-semântico)
- [Publicação — Registry Privado](#publicação--registry-privado)
- [Consumo de Módulos da Comunidade](#consumo-de-módulos-da-comunidade)
- [Exemplos Práticos — AWS](#exemplos-práticos--aws)
  - [Módulo VPC](#módulo-vpc)
  - [Módulo ECS Service](#módulo-ecs-service)
  - [Módulo RDS](#módulo-rds)
  - [Módulo Lambda](#módulo-lambda)
- [Anti-Patterns em Módulos](#anti-patterns-em-módulos)

---

## Quando Criar um Módulo

### ✅ Crie quando:

- O mesmo padrão de recursos se repete em 2+ lugares
- Precisa encapsular complexidade (VPC + subnets + NAT + route tables)
- Quer impor padrões da organização (tags, encryption, logging)
- O time de plataforma quer oferecer "building blocks" padronizados

### ❌ Não crie quando:

- É um recurso simples sem customização (um S3 bucket básico)
- A abstração esconde informações úteis sem simplificar
- Vai criar apenas para "organizar" um módulo com 1-2 resources
- O módulo tem mais variáveis que o código original

> **Regra de ouro:** Se o módulo não simplifica o uso NEM impõe padrões, ele não deveria existir.

---

## Anatomia de um Módulo

```
modules/networking/vpc/
├── main.tf                # Recursos principais
├── variables.tf           # Todas as variáveis (inputs)
├── outputs.tf             # Todos os outputs
├── versions.tf            # Required providers e version constraints
├── locals.tf              # Computações internas
├── data.tf                # Data sources
├── README.md              # Documentação (gerada por terraform-docs)
├── CHANGELOG.md           # Histórico de mudanças
├── examples/              # Exemplos de uso
│   ├── basic/
│   │   └── main.tf
│   └── complete/
│       └── main.tf
└── tests/                 # Testes
    ├── unit/
    │   └── basic_test.tftest.hcl
    └── integration/
        └── full_test.tftest.hcl
```

### versions.tf

```hcl
terraform {
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}
```

> **Regra:** Módulos NÃO definem `provider` blocks nem `backend`. Quem define é o caller.

---

## Convenções de Design

### 1. Variáveis — Minimize e Documente

```hcl
# ✅ Bom — interface clara e enxuta
variable "name" {
  description = "Name for the VPC and associated resources"
  type        = string
}

variable "cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrhost(var.cidr, 0))
    error_message = "Must be a valid CIDR block."
  }
}

variable "azs" {
  description = "List of availability zones"
  type        = list(string)
  default     = []
}

variable "enable_nat_gateway" {
  description = "Enable NAT Gateway for private subnets"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Additional tags for all resources"
  type        = map(string)
  default     = {}
}
```

### 2. Outputs — Exporte Tudo que o Caller Precisa

```hcl
# ✅ Bom — outputs completos com description
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "The CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "nat_gateway_ips" {
  description = "List of NAT Gateway public IPs"
  value       = aws_eip.nat[*].public_ip
}

output "private_route_table_ids" {
  description = "List of private route table IDs"
  value       = aws_route_table.private[*].id
}
```

### 3. Defaults Seguros

```hcl
# ✅ Módulo com defaults seguros por padrão
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"  # Versionamento habilitado por padrão
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"  # Encryption por padrão
    }
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true   # Bloqueia ACLs públicas
  block_public_policy     = true   # Bloqueia policies públicas
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### 4. Composição via Submodules

```hcl
# modules/compute/ecs-service/main.tf
# Módulo composto que encapsula múltiplos concerns

module "task_definition" {
  source = "./modules/task-definition"
  # ...
}

module "service" {
  source = "./modules/service"
  task_definition_arn = module.task_definition.arn
  # ...
}

module "autoscaling" {
  source  = "./modules/autoscaling"
  service = module.service.name
  cluster = module.service.cluster
  # ...
}
```

---

## Módulos Compostos vs Atômicos

| Tipo | Descrição | Exemplo |
|------|-----------|---------|
| **Atômico** | Um recurso ou grupo mínimo | `modules/networking/security-group` |
| **Composto** | Combina múltiplos atômicos | `modules/networking/vpc` (VPC + subnets + NAT + routes) |
| **Stack** | Infraestrutura completa de um serviço | `modules/stacks/microservice` (ECS + ALB + RDS + SG) |

### Quando usar cada tipo:

```
Stack (microservice)
├── Composto (networking)
│   ├── Atômico (vpc)
│   ├── Atômico (subnet)
│   └── Atômico (security-group)
├── Composto (compute)
│   ├── Atômico (ecs-cluster)
│   └── Atômico (ecs-service)
└── Composto (data)
    └── Atômico (rds)
```

- **Atômicos:** Time de plataforma, max flexibilidade
- **Compostos:** Times de produto, combinações comuns
- **Stacks:** Self-service, opinionado, rápido para começar

---

## Versionamento Semântico

```
v1.0.0 → v1.1.0 → v1.2.0 → v2.0.0
  │          │          │        │
  │          │          │        └── Breaking change (remove variável, renomeia output)
  │          │          └── Nova feature (novo output, nova variável opcional)
  │          └── Nova feature compatível
  └── Versão inicial estável
```

### O que é Breaking Change?

- Remover ou renomear uma variável
- Remover ou renomear um output
- Mudar o tipo de uma variável
- Remover um recurso que causa destroy
- Mudar o comportamento default

### CHANGELOG.md

```markdown
## [2.0.0] - 2025-06-01

### BREAKING
- Removida variável `enable_logs` — agora logging é sempre habilitado
- Output `bucket_name` renomeado para `bucket_id`

### Added
- Suporte a KMS encryption com chave customizada

## [1.2.0] - 2025-05-15

### Added
- Nova variável `lifecycle_rules` para configurar expiration policies
- Novo output `bucket_arn`

## [1.1.0] - 2025-05-01

### Added
- Suporte a replicação cross-region via variável `replication_config`
```

---

## Publicação — Registry Privado

### Terraform Cloud / Enterprise

```hcl
# Consumir módulo do registry privado
module "vpc" {
  source  = "app.terraform.io/myorg/vpc/aws"
  version = "~> 2.0"
  # ...
}
```

### Git com Tags

```bash
# Taggear release
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
```

```hcl
# Consumir do Git
module "vpc" {
  source = "git::https://github.com/myorg/terraform-aws-vpc.git?ref=v1.0.0"
}

# SSH
module "vpc" {
  source = "git::ssh://git@github.com/myorg/terraform-aws-vpc.git?ref=v1.0.0"
}
```

### S3 como Registry (Simples)

```hcl
module "vpc" {
  source = "s3::https://s3-us-east-1.amazonaws.com/myorg-tf-modules/vpc/v1.0.0.zip"
}
```

---

## Consumo de Módulos da Comunidade

### Módulos Recomendados (terraform-aws-modules)

| Módulo | Registry | Uso |
|--------|----------|-----|
| VPC | `terraform-aws-modules/vpc/aws` | Redes |
| EKS | `terraform-aws-modules/eks/aws` | Kubernetes |
| RDS | `terraform-aws-modules/rds/aws` | Banco de dados |
| S3 | `terraform-aws-modules/s3-bucket/aws` | Storage |
| Lambda | `terraform-aws-modules/lambda/aws` | Serverless |
| ALB | `terraform-aws-modules/alb/aws` | Load balancer |
| Security Group | `terraform-aws-modules/security-group/aws` | Firewall |
| IAM | `terraform-aws-modules/iam/aws` | Identity |

### Boas Práticas ao Consumir

```hcl
# ✅ Pin de versão major
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"   # Permite 5.x, bloqueia 6.x

  name = "${local.name_prefix}-vpc"
  cidr = var.vpc_cidr

  azs             = data.aws_availability_zones.available.names
  private_subnets = local.private_subnet_cidrs
  public_subnets  = local.public_subnet_cidrs

  enable_nat_gateway     = true
  single_nat_gateway     = var.environment != "prod"
  enable_dns_hostnames   = true
  enable_dns_support     = true

  tags = local.common_tags
}
```

---

## Exemplos Práticos — AWS

### Módulo VPC

```hcl
# modules/networking/vpc/main.tf

resource "aws_vpc" "main" {
  cidr_block           = var.cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.tags, {
    Name = "${var.name}-vpc"
  })
}

resource "aws_subnet" "private" {
  for_each = {
    for idx, az in local.azs :
    az => {
      cidr = cidrsubnet(var.cidr, 8, idx)
      az   = az
    }
  }

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az

  tags = merge(var.tags, {
    Name = "${var.name}-private-${each.key}"
    Tier = "private"
  })
}

resource "aws_subnet" "public" {
  for_each = {
    for idx, az in local.azs :
    az => {
      cidr = cidrsubnet(var.cidr, 8, idx + 100)
      az   = az
    }
  }

  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.name}-public-${each.key}"
    Tier = "public"
  })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(var.tags, {
    Name = "${var.name}-igw"
  })
}

resource "aws_eip" "nat" {
  for_each = var.enable_nat_gateway ? toset(local.azs) : toset([])
  domain   = "vpc"

  tags = merge(var.tags, {
    Name = "${var.name}-nat-eip-${each.key}"
  })
}

resource "aws_nat_gateway" "main" {
  for_each      = var.enable_nat_gateway ? toset(local.azs) : toset([])
  allocation_id = aws_eip.nat[each.key].id
  subnet_id     = aws_subnet.public[each.key].id

  tags = merge(var.tags, {
    Name = "${var.name}-nat-${each.key}"
  })

  depends_on = [aws_internet_gateway.main]
}
```

### Módulo Lambda

```hcl
# modules/compute/lambda/main.tf

resource "aws_lambda_function" "this" {
  function_name = var.name
  description   = var.description
  role          = aws_iam_role.lambda.arn
  handler       = var.handler
  runtime       = var.runtime
  timeout       = var.timeout
  memory_size   = var.memory_size

  filename         = var.filename
  source_code_hash = var.source_code_hash

  environment {
    variables = var.environment_variables
  }

  dynamic "vpc_config" {
    for_each = var.vpc_subnet_ids != null ? [1] : []
    content {
      subnet_ids         = var.vpc_subnet_ids
      security_group_ids = var.vpc_security_group_ids
    }
  }

  tracing_config {
    mode = "Active"  # X-Ray habilitado por padrão
  }

  dead_letter_config {
    target_arn = var.dlq_arn
  }

  tags = var.tags

  depends_on = [
    aws_iam_role_policy_attachment.lambda_basic,
    aws_cloudwatch_log_group.lambda,
  ]
}

resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${var.name}"
  retention_in_days = var.log_retention_days

  tags = var.tags
}

resource "aws_iam_role" "lambda" {
  name = "${var.name}-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })

  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

---

## Anti-Patterns em Módulos

### ❌ Módulo "God" — Faz Tudo

```hcl
# RUIM — módulo que cria toda a infraestrutura
module "everything" {
  source = "./modules/everything"
  # 50+ variáveis
  # VPC + ECS + RDS + S3 + IAM + CloudWatch + ...
}
```

### ❌ Módulo "Wrapper Fino" — Não Agrega Valor

```hcl
# RUIM — apenas repassa variáveis
module "bucket" {
  source = "./modules/s3"
  bucket = var.bucket  # Só repassa
  tags   = var.tags    # Só repassa
}
```

### ❌ Provider no Módulo

```hcl
# RUIM — módulo não deve definir provider
# modules/vpc/main.tf
provider "aws" {
  region = "us-east-1"  # ❌ Quem decide é o caller
}
```

### ❌ Backend no Módulo

```hcl
# RUIM — módulo não deve definir backend
# modules/vpc/backend.tf
terraform {
  backend "s3" { ... }  # ❌ Quem decide é o caller
}
```

### ❌ Variáveis Demais

```hcl
# RUIM — se o módulo tem 40+ variáveis, provavelmente precisa ser quebrado
variable "vpc_cidr" { ... }
variable "subnet_1_cidr" { ... }
variable "subnet_2_cidr" { ... }
# ... 37 mais variáveis
```

> **Regra:** Se o módulo tem mais variáveis do que o código que ele encapsula,
> provavelmente não deveria ser um módulo.

````
