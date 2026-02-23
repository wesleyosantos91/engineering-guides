````markdown
# Estrutura de Projeto — Terraform AWS

> **Objetivo:** Definir a organização de diretórios, naming conventions e padrões de estrutura
> para projetos Terraform focados em AWS.

---

## Sumário

- [Estrutura Mono-Repo (Recomendada para times)](#estrutura-mono-repo-recomendada-para-times)
- [Estrutura Multi-Repo](#estrutura-multi-repo)
- [Estrutura por Ambiente](#estrutura-por-ambiente)
- [Naming Conventions](#naming-conventions)
- [Organização de Arquivos por Componente](#organização-de-arquivos-por-componente)
- [Terraform Workspaces vs Directory-based Isolation](#terraform-workspaces-vs-directory-based-isolation)
- [Terragrunt para DRY Multi-Environment](#terragrunt-para-dry-multi-environment)

---

## Estrutura Mono-Repo (Recomendada para times)

```
infrastructure/
├── modules/                          # Módulos internos reutilizáveis
│   ├── networking/
│   │   ├── vpc/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── versions.tf
│   │   │   ├── README.md
│   │   │   └── tests/
│   │   │       └── vpc_test.tftest.hcl
│   │   ├── security-group/
│   │   └── alb/
│   ├── compute/
│   │   ├── ecs-service/
│   │   ├── lambda/
│   │   └── ec2/
│   ├── data/
│   │   ├── rds/
│   │   ├── dynamodb/
│   │   ├── elasticache/
│   │   └── s3/
│   ├── messaging/
│   │   ├── sqs/
│   │   ├── sns/
│   │   └── eventbridge/
│   ├── observability/
│   │   ├── cloudwatch/
│   │   └── xray/
│   └── security/
│       ├── iam-role/
│       ├── kms/
│       └── waf/
│
├── live/                              # Ambientes reais (state isolado)
│   ├── _global/                       # Recursos globais (IAM, Route53, etc.)
│   │   ├── iam/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── backend.tf
│   │   │   └── terraform.tfvars
│   │   └── route53/
│   │
│   ├── dev/
│   │   ├── networking/
│   │   │   ├── main.tf               # Chama module "../../../modules/networking/vpc"
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── backend.tf
│   │   │   └── terraform.tfvars
│   │   ├── compute/
│   │   ├── data/
│   │   └── messaging/
│   │
│   ├── staging/
│   │   ├── networking/
│   │   ├── compute/
│   │   ├── data/
│   │   └── messaging/
│   │
│   └── prod/
│       ├── networking/
│       ├── compute/
│       ├── data/
│       └── messaging/
│
├── terragrunt.hcl                     # Config raiz (se usar Terragrunt)
├── .tflint.hcl                        # Configuração do TFLint
├── .terraform-docs.yml                # Configuração do terraform-docs
├── .pre-commit-config.yaml            # Hooks de pre-commit
└── Makefile                           # Comandos utilitários
```

---

## Estrutura Multi-Repo

Use quando diferentes times são responsáveis por diferentes domínios:

```
# Repo: infra-networking
modules/
└── vpc/
live/
├── dev/
├── staging/
└── prod/

# Repo: infra-compute
modules/
└── ecs-service/
live/
├── dev/
├── staging/
└── prod/

# Repo: terraform-modules (registry interno)
modules/
├── vpc/
├── ecs-service/
├── rds/
└── s3/
```

---

## Estrutura por Ambiente

Cada componente contém os seguintes arquivos padrão:

```
live/dev/networking/
├── main.tf                # Recursos e chamadas de módulo
├── variables.tf           # Declaração de variáveis
├── outputs.tf             # Outputs exportados
├── locals.tf              # Locals e computações
├── data.tf                # Data sources (remote state, SSM, etc.)
├── backend.tf             # Configuração do backend S3
├── providers.tf           # Provider configuration
├── versions.tf            # Required providers e versão do TF
└── terraform.tfvars       # Valores para o ambiente
```

### backend.tf — Exemplo

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "dev/networking/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

> **Regra:** Cada componente por ambiente tem seu próprio state file (`key` diferente).

### providers.tf — Exemplo

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
      Project     = var.project_name
      Team        = var.team
    }
  }
}
```

> **Regra:** Sempre use `default_tags` para aplicar tags padrão em todos os recursos.

---

## Naming Conventions

### Recursos Terraform

```hcl
# ✅ Correto — snake_case, descritivo
resource "aws_s3_bucket" "application_logs" { ... }
resource "aws_ecs_service" "payment_api" { ... }
resource "aws_security_group" "alb_ingress" { ... }

# ❌ Errado — genérico, camelCase, abreviações
resource "aws_s3_bucket" "bucket1" { ... }
resource "aws_ecs_service" "myService" { ... }
resource "aws_security_group" "sg" { ... }
```

### Recursos AWS (nome real na AWS)

```hcl
# Padrão: {project}-{environment}-{component}-{qualifier}
locals {
  name_prefix = "${var.project}-${var.environment}"
}

resource "aws_s3_bucket" "application_logs" {
  bucket = "${local.name_prefix}-application-logs"
  # Resultado: "myapp-dev-application-logs"
}

resource "aws_ecs_cluster" "main" {
  name = "${local.name_prefix}-cluster"
  # Resultado: "myapp-dev-cluster"
}
```

### Variáveis e Outputs

```hcl
# ✅ Variáveis — claras, com description e type
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid CIDR block."
  }
}

# ✅ Outputs — prefixo do recurso, description obrigatória
output "vpc_id" {
  description = "The ID of the VPC"
  value       = module.vpc.vpc_id
}
```

---

## Terraform Workspaces vs Directory-based Isolation

| Aspecto | Workspaces | Directory-based |
|---------|-----------|----------------|
| Complexidade | Baixa | Média |
| Isolamento de State | Mesmo backend, keys diferentes | Backends/keys completamente separados |
| Segurança | Menor — mesmo backend | Maior — permissões por diretório |
| Diferenças entre envs | Difícil (ternários) | Fácil (tfvars diferentes) |
| Visibilidade | `terraform workspace list` | Estrutura de pastas |
| **Recomendação** | Projetos pequenos, pessoais | **Produção, times** ✅ |

> **Recomendação:** Use **directory-based isolation** para produção. Workspaces introduzem
> complexidade desnecessária quando ambientes têm diferenças significativas.

---

## Terragrunt para DRY Multi-Environment

Quando a duplicação entre ambientes se torna excessiva, use Terragrunt:

```
infrastructure/
├── modules/                      # Módulos Terraform puros
│   └── networking/vpc/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── live/
│   ├── terragrunt.hcl            # Config raiz (remote state, provider)
│   │
│   ├── dev/
│   │   ├── env.hcl               # Variáveis do ambiente
│   │   ├── networking/
│   │   │   └── terragrunt.hcl    # Referencia o módulo + inputs
│   │   └── compute/
│   │       └── terragrunt.hcl
│   │
│   ├── staging/
│   │   ├── env.hcl
│   │   └── networking/
│   │       └── terragrunt.hcl
│   │
│   └── prod/
│       ├── env.hcl
│       └── networking/
│           └── terragrunt.hcl
```

### terragrunt.hcl (raiz)

```hcl
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "mycompany-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

generate "provider" {
  path      = "providers.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "us-east-1"
  default_tags {
    tags = {
      ManagedBy = "terraform"
    }
  }
}
EOF
}
```

### terragrunt.hcl (componente)

```hcl
include "root" {
  path = find_in_parent_folders()
}

locals {
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
}

terraform {
  source = "../../../modules/networking/vpc"
}

inputs = {
  environment = local.env_vars.locals.environment
  vpc_cidr    = "10.0.0.0/16"
  project     = "myapp"
}
```

````
