````markdown
# Boas Práticas — Terraform AWS

> **Objetivo:** Compilar as melhores práticas do mercado para Terraform com foco em AWS,
> baseado no AWS Well-Architected Framework, HashiCorp Best Practices e experiências de
> empresas de grande porte.

---

## Sumário

- [Regras de Ouro](#regras-de-ouro)
- [Versionamento e Pinning](#versionamento-e-pinning)
- [Variáveis e Validações](#variáveis-e-validações)
- [Locals — Quando e Como Usar](#locals--quando-e-como-usar)
- [Data Sources](#data-sources)
- [Tags — Obrigatórias](#tags--obrigatórias)
- [Lifecycle Rules](#lifecycle-rules)
- [Depends On — Use com Parcimônia](#depends-on--use-com-parcimônia)
- [Count vs for_each](#count-vs-for_each)
- [Dynamic Blocks](#dynamic-blocks)
- [Moved Blocks — Refatoração Segura](#moved-blocks--refatoração-segura)
- [Import — Recursos Existentes](#import--recursos-existentes)
- [Formatação e Lint](#formatação-e-lint)
- [Documentação Automatizada](#documentação-automatizada)
- [Custo — Infracost](#custo--infracost)
- [Anti-Patterns a Evitar](#anti-patterns-a-evitar)

---

## Regras de Ouro

1. **Um recurso, uma responsabilidade** — cada resource block deve representar exatamente um recurso
2. **Nunca hardcode** — use variáveis, locals e data sources
3. **Pin versions** — provider, módulos e Terraform CLI
4. **Sempre use `terraform plan`** antes de `apply`
5. **State remoto com lock** — S3 + DynamoDB, sempre
6. **Code review** — toda mudança de infra passa por PR
7. **Testes automatizados** — unit + integration em CI
8. **Não misture responsabilidades** — separe networking, compute, data em states diferentes
9. **Faça `terraform fmt`** antes de cada commit
10. **Documente** — `description` em todas as variáveis e outputs

---

## Versionamento e Pinning

### Versão do Terraform

```hcl
terraform {
  required_version = ">= 1.6.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"    # Permite 5.x, bloqueia 6.x
    }
  }
}
```

### Versão de Módulos

```hcl
# ✅ Registry — pin major version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}

# ✅ Git — pin por tag
module "custom_module" {
  source = "git::https://github.com/myorg/terraform-modules.git//vpc?ref=v1.2.0"
}

# ❌ Nunca use sem versão
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  # Sem version = PERIGO
}
```

### Lock File

```bash
# Gere e commite o .terraform.lock.hcl
terraform init
git add .terraform.lock.hcl
git commit -m "chore: update terraform lock file"
```

> **Regra:** Sempre commite o `.terraform.lock.hcl` — garante builds reproduzíveis.

---

## Variáveis e Validações

### Tipos Explícitos

```hcl
variable "instance_type" {
  description = "EC2 instance type for the application server"
  type        = string
  default     = "t3.medium"
}

variable "subnets" {
  description = "List of subnet IDs for the ALB"
  type        = list(string)
}

variable "tags" {
  description = "Additional tags to apply to all resources"
  type        = map(string)
  default     = {}
}

variable "ecs_service_config" {
  description = "Configuration for the ECS service"
  type = object({
    cpu           = number
    memory        = number
    desired_count = number
    port          = number
  })
}
```

### Validações Customizadas

```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "cidr_block" {
  description = "VPC CIDR block"
  type        = string

  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR notation (e.g., 10.0.0.0/16)."
  }
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string

  validation {
    condition     = can(regex("^t3\\.|^m5\\.|^c5\\.", var.instance_type))
    error_message = "Only t3, m5, or c5 instance families are allowed."
  }
}
```

### Sensitive Variables

```hcl
variable "db_password" {
  description = "Password for the RDS master user"
  type        = string
  sensitive   = true  # Oculta do output e do plan
}
```

---

## Locals — Quando e Como Usar

```hcl
locals {
  # Prefixo para naming
  name_prefix = "${var.project}-${var.environment}"

  # Merge de tags
  common_tags = merge(var.tags, {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
    UpdatedAt   = timestamp()
  })

  # Computações complexas
  private_subnet_cidrs = [
    for i in range(var.az_count) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]

  public_subnet_cidrs = [
    for i in range(var.az_count) :
    cidrsubnet(var.vpc_cidr, 8, i + 100)
  ]

  # Condicionais
  enable_nat_gateway = var.environment == "prod" ? true : false
}
```

> **Regra:** Use locals para eliminar duplicação e tornar o código mais legível.
> Evite `locals` que simplesmente renomeiam uma variável sem agregar valor.

---

## Data Sources

```hcl
# ✅ Buscar AZs disponíveis dinamicamente
data "aws_availability_zones" "available" {
  state = "available"
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}

# ✅ Buscar AMI mais recente
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

# ✅ Buscar dados de outro state (Remote State)
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "${var.environment}/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# ✅ Buscar valores do SSM Parameter Store
data "aws_ssm_parameter" "db_endpoint" {
  name = "/${var.environment}/database/endpoint"
}

# ✅ Account ID atual
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
```

---

## Tags — Obrigatórias

### via Default Tags (Recomendado)

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project
      Team        = var.team
      ManagedBy   = "terraform"
      CostCenter  = var.cost_center
    }
  }
}
```

### Tags Específicas por Recurso

```hcl
resource "aws_instance" "app" {
  # ... config ...

  tags = {
    Name        = "${local.name_prefix}-app"
    Application = "payment-service"
    # Default tags são herdadas automaticamente
  }
}
```

### Padrão Mínimo de Tags

| Tag | Obrigatória | Descrição |
|-----|-------------|-----------|
| `Environment` | ✅ | dev, staging, prod |
| `Project` | ✅ | Nome do projeto |
| `Team` | ✅ | Time responsável |
| `ManagedBy` | ✅ | terraform |
| `CostCenter` | ✅ | Centro de custo |
| `Name` | ✅ | Nome descritivo |
| `Application` | Recomendada | Nome da aplicação |
| `Owner` | Recomendada | Email do responsável |

---

## Lifecycle Rules

```hcl
# Prevenir destruição acidental de recursos críticos
resource "aws_rds_instance" "main" {
  # ... config ...

  lifecycle {
    prevent_destroy = true  # Terraform recusa destruir
  }
}

# Ignorar mudanças feitas fora do Terraform
resource "aws_ecs_service" "app" {
  # ... config ...

  lifecycle {
    ignore_changes = [
      desired_count,  # Auto-scaling muda isso
      task_definition, # Deployment pipeline atualiza
    ]
  }
}

# Criar novo antes de destruir o antigo (zero downtime)
resource "aws_instance" "app" {
  # ... config ...

  lifecycle {
    create_before_destroy = true
  }
}
```

---

## Depends On — Use com Parcimônia

```hcl
# ✅ Necessário — dependência implícita não é suficiente
resource "aws_ecs_service" "app" {
  # ... config ...

  depends_on = [
    aws_lb_listener.https,  # Listener precisa existir antes do service
  ]
}

# ❌ Desnecessário — Terraform infere automaticamente
resource "aws_instance" "app" {
  subnet_id = aws_subnet.private.id  # Dependência implícita

  depends_on = [aws_subnet.private]  # REDUNDANTE
}
```

> **Regra:** Use `depends_on` apenas quando a dependência NÃO pode ser inferida
> por referência de atributo.

---

## Count vs for_each

```hcl
# ❌ count — problemas ao remover itens do meio da lista
resource "aws_subnet" "private" {
  count = length(var.private_subnets)
  # Se remover o item [1], o item [2] vira [1] e é recriado!
}

# ✅ for_each — estável, baseado em chave
resource "aws_subnet" "private" {
  for_each = tomap({
    for idx, cidr in var.private_subnets :
    "private-${idx}" => {
      cidr = cidr
      az   = data.aws_availability_zones.available.names[idx]
    }
  })

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az

  tags = {
    Name = "${local.name_prefix}-${each.key}"
  }
}

# ✅ count — OK para toggle simples (0 ou 1)
resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? 1 : 0
  # ...
}
```

> **Regra:** Use `for_each` para coleções que podem mudar. Use `count` apenas para
> condicionais (0 ou 1).

---

## Dynamic Blocks

```hcl
# ✅ Uso adequado — regras dinâmicas de security group
resource "aws_security_group" "app" {
  name        = "${local.name_prefix}-app-sg"
  description = "Security group for application"
  vpc_id      = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound traffic"
  }
}
```

> **Regra:** Use dynamic blocks com moderação. Se o bloco é fixo, escreva explicitamente.
> Dynamic blocks reduzem a legibilidade.

---

## Moved Blocks — Refatoração Segura

```hcl
# Renomear recurso sem destruir e recriar
moved {
  from = aws_s3_bucket.logs
  to   = aws_s3_bucket.application_logs
}

# Mover recurso para dentro de um módulo
moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.main
}

# Mover de um index (count) para for_each
moved {
  from = aws_subnet.private[0]
  to   = aws_subnet.private["us-east-1a"]
}
```

> **Regra:** Sempre use `moved` blocks ao refatorar. Nunca faça rename que resulte em
> destroy + create de recursos em produção.

---

## Import — Recursos Existentes

### Import Block (Terraform 1.5+)

```hcl
# Declarativo — no código
import {
  to = aws_s3_bucket.legacy_bucket
  id = "my-existing-bucket-name"
}

resource "aws_s3_bucket" "legacy_bucket" {
  bucket = "my-existing-bucket-name"
  # ... demais configurações
}
```

```bash
# Gerar código automaticamente a partir do import
terraform plan -generate-config-out=generated.tf
```

### Import CLI (Legacy)

```bash
terraform import aws_s3_bucket.legacy_bucket my-existing-bucket-name
```

---

## Formatação e Lint

### terraform fmt

```bash
# Formatar todos os arquivos
terraform fmt -recursive

# Verificar sem alterar (CI)
terraform fmt -check -recursive -diff
```

### TFLint

```hcl
# .tflint.hcl
plugin "terraform" {
  enabled = true
  preset  = "recommended"
}

plugin "aws" {
  enabled = true
  version = "0.31.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_naming_convention" {
  enabled = true
}

rule "terraform_documented_variables" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}
```

```bash
tflint --init
tflint --recursive
```

---

## Documentação Automatizada

### terraform-docs

```yaml
# .terraform-docs.yml
formatter: markdown table

output:
  file: README.md
  mode: inject

sort:
  enabled: true
  by: required

settings:
  indent: 3
  escape: false
  hide-empty: true
```

```bash
# Gerar docs para todos os módulos
terraform-docs markdown table --output-file README.md ./modules/networking/vpc/
```

### Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.88.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
        args:
          - --args=--config=__GIT_WORKING_DIR__/.tflint.hcl
      - id: terraform_docs
        args:
          - --hook-config=--path-to-file=README.md
          - --hook-config=--add-to-existing-file=true
      - id: terraform_tfsec
      - id: infracost_breakdown
        args:
          - --args=--path=.
```

---

## Custo — Infracost

```bash
# Verificar custo estimado
infracost breakdown --path=.

# Comparar custo entre branches (CI)
infracost diff --path=. --compare-to=infracost-base.json
```

```yaml
# GitHub Actions — custo como comentário no PR
- name: Infracost
  uses: infracost/actions/setup@v3
  with:
    api-key: ${{ secrets.INFRACOST_API_KEY }}
```

---

## Anti-Patterns a Evitar

### ❌ State Monolítico

```
# RUIM — tudo em um único state
terraform/
└── main.tf  # VPC + ECS + RDS + S3 + IAM + ...
```

### ❌ Hardcoded Values

```hcl
# RUIM
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  subnet_id     = "subnet-0bb1c79de3EXAMPLE"
}
```

### ❌ Provisioners

```hcl
# RUIM — use user_data ou ferramentas específicas (Ansible, SSM)
resource "aws_instance" "app" {
  provisioner "remote-exec" {
    inline = ["sudo apt-get update"]
  }
}
```

### ❌ Variáveis sem Tipo e Descrição

```hcl
# RUIM
variable "name" {}
variable "env" {}
```

### ❌ Outputs sem Descrição

```hcl
# RUIM
output "id" {
  value = aws_vpc.main.id
}
```

### ❌ Uso Excessivo de `null_resource`

```hcl
# RUIM — geralmente indica design problem
resource "null_resource" "deploy" {
  provisioner "local-exec" {
    command = "deploy.sh"
  }
}
```

> Prefira `terraform_data` (Terraform 1.4+) quando precisar de triggers
> sem recurso real.

````
