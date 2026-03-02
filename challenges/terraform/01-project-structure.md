# Level 1 — Estrutura de Projeto

> **Objetivo:** Organizar o projeto Terraform seguindo padrões profissionais — mono-repo, separação por domínio, naming conventions, isolamento por ambiente e Terragrunt para DRY multi-environment.

---

## Objetivo de Aprendizado

- Estruturar um projeto Terraform mono-repo profissional
- Aplicar naming conventions consistentes (recursos Terraform e AWS)
- Separar infraestrutura por domínio (networking, compute, data, messaging)
- Implementar isolamento por ambiente usando directory-based isolation
- Entender Workspaces vs Directory-based isolation e seus trade-offs
- Configurar ferramentas auxiliares: TFLint, terraform-docs, pre-commit
- (Opcional) Introduzir Terragrunt para eliminar duplicação entre ambientes

---

## Escopo Funcional

### Infraestrutura ShopFlow — Organização Multi-Domínio

Reorganize a infraestrutura do Level 0 em uma estrutura profissional com os seguintes domínios:

1. **Networking** — VPC, Subnets (public + private), Internet Gateway, Route Tables, Security Groups
2. **Storage** — S3 Buckets (assets, logs)
3. **Data** — DynamoDB Tables (products, sessions)
4. **Messaging** — SQS Queues (orders, DLQ), SNS Topics
5. **Global** — IAM Roles base, KMS Keys

---

## Escopo Técnico

### Estrutura Alvo — Mono-Repo

```
shopflow-infra/
├── modules/                              # Módulos internos reutilizáveis
│   ├── networking/
│   │   └── vpc/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       ├── versions.tf
│   │       └── README.md
│   ├── storage/
│   │   └── s3-bucket/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       ├── versions.tf
│   │       └── README.md
│   ├── data/
│   │   └── dynamodb-table/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       ├── versions.tf
│   │       └── README.md
│   └── messaging/
│       └── sqs-queue/
│           ├── main.tf
│           ├── variables.tf
│           ├── outputs.tf
│           ├── versions.tf
│           └── README.md
│
├── live/                                  # Ambientes reais (state isolado)
│   ├── _global/                           # Recursos globais (IAM, KMS)
│   │   └── iam/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       ├── providers.tf
│   │       └── terraform.tfvars
│   │
│   ├── dev/
│   │   ├── networking/
│   │   │   ├── main.tf                    # Chama module "../../../modules/networking/vpc"
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── providers.tf
│   │   │   └── terraform.tfvars
│   │   ├── storage/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── providers.tf
│   │   │   └── terraform.tfvars
│   │   ├── data/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── providers.tf
│   │   │   └── terraform.tfvars
│   │   └── messaging/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       ├── providers.tf
│   │       └── terraform.tfvars
│   │
│   ├── staging/
│   │   ├── networking/
│   │   ├── storage/
│   │   ├── data/
│   │   └── messaging/
│   │
│   └── prod/
│       ├── networking/
│       ├── storage/
│       ├── data/
│       └── messaging/
│
├── docker-compose.yml                     # LocalStack
├── .tflint.hcl                            # TFLint config
├── .terraform-docs.yml                    # terraform-docs config
├── .pre-commit-config.yaml                # Pre-commit hooks
├── Makefile                               # Comandos utilitários
└── README.md
```

### Naming Conventions

| Tipo | Padrão | Exemplo |
|------|--------|---------|
| **Recurso Terraform** | `snake_case`, descritivo | `aws_s3_bucket.application_logs` |
| **Recurso AWS (Name)** | `{project}-{env}-{component}` | `shopflow-dev-application-logs` |
| **Variáveis** | `snake_case` com `description` | `var.vpc_cidr` |
| **Outputs** | `snake_case` com `description` | `output.vpc_id` |
| **Módulos** | `snake_case` | `module.vpc` |
| **Locals** | `snake_case` | `local.name_prefix` |
| **Arquivos** | Padrão: `main.tf`, `variables.tf`, `outputs.tf`, `locals.tf`, `providers.tf`, `versions.tf` | — |

### Ferramentas de Qualidade

| Ferramenta | Propósito | Arquivo de Config |
|-----------|-----------|------------------|
| **TFLint** | Lint de código Terraform | `.tflint.hcl` |
| **terraform-docs** | Geração automática de README | `.terraform-docs.yml` |
| **pre-commit** | Hooks de pré-commit | `.pre-commit-config.yaml` |
| **terraform fmt** | Formatação automática | — (built-in) |

---

## Tarefas

### Tarefa 1 — Reestruturar Projeto

Migre o código do Level 0 (tudo em um diretório) para a estrutura mono-repo:

1. Crie a árvore de diretórios `modules/` e `live/`
2. Extraia cada grupo de recursos para seu respectivo módulo em `modules/`
3. Crie os componentes em `live/dev/` chamando os módulos
4. Cada componente em `live/dev/` deve ter seus próprios `providers.tf` e `terraform.tfvars`

### Tarefa 2 — Módulo VPC (Networking)

Crie o módulo `modules/networking/vpc/` que provisiona:

```hcl
# Recursos a criar no módulo:
# - aws_vpc.main
# - aws_subnet.public (for_each sobre AZs)
# - aws_subnet.private (for_each sobre AZs)
# - aws_internet_gateway.main
# - aws_route_table.public
# - aws_route.public_internet
# - aws_route_table_association.public (for_each)
# - aws_route_table.private (for_each)
# - aws_route_table_association.private (for_each)
```

**Interface do módulo (`variables.tf`):**

```hcl
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
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets"
  type        = list(string)
}

variable "tags" {
  description = "Additional tags for all resources"
  type        = map(string)
  default     = {}
}
```

**Uso em `live/dev/networking/main.tf`:**

```hcl
module "vpc" {
  source = "../../../modules/networking/vpc"

  name                 = "${local.name_prefix}-vpc"
  cidr                 = "10.0.0.0/16"
  azs                  = ["us-east-1a", "us-east-1b"]
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]

  tags = {
    Component = "networking"
  }
}
```

### Tarefa 3 — Módulo S3 Bucket (Storage)

Crie o módulo `modules/storage/s3-bucket/` com:

- Versionamento configurável (`enable_versioning`)
- Lifecycle rules configuráveis
- Bloqueio de acesso público por padrão
- Encryption (SSE-S3) por padrão
- Tags herdadas + específicas

### Tarefa 4 — Módulo DynamoDB (Data)

Crie o módulo `modules/data/dynamodb-table/` com:

- Hash key configurável
- Range key opcional
- GSIs configuráveis via `for_each`
- TTL configurável
- Billing mode configurável (`PAY_PER_REQUEST` default)

### Tarefa 5 — Módulo SQS (Messaging)

Crie o módulo `modules/messaging/sqs-queue/` com:

- DLQ opcional (auto-criada quando `enable_dlq = true`)
- Redrive policy configurável
- Visibility timeout e retention configuráveis
- Encryption (SSE-SQS) por padrão

### Tarefa 6 — Conectar Ambientes

1. Crie `live/dev/` com todos os componentes chamando os módulos
2. Crie `live/staging/` com os mesmos módulos mas `terraform.tfvars` diferentes
3. Execute `terraform apply` em cada componente do ambiente `dev`
4. Verifique isolamento: cada componente tem seu próprio state

### Tarefa 7 — Configurar Ferramentas de Qualidade

**.tflint.hcl:**

```hcl
config {
  call_module_type = "local"
}

plugin "terraform" {
  enabled = true
  preset  = "recommended"
}

plugin "aws" {
  enabled = true
  version = "0.35.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}
```

**.terraform-docs.yml:**

```yaml
formatter: markdown table
output:
  file: README.md
  mode: inject

sort:
  enabled: true
  by: required

settings:
  hide-empty: true
  read-comments: true
```

**.pre-commit-config.yaml:**

```yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-tf
    rev: v1.96.1
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_docs
        args: ['--args=--config=.terraform-docs.yml']
      - id: terraform_tflint
        args: ['--args=--config=__GIT_WORKING_DIR__/.tflint.hcl']
```

**Makefile:**

```makefile
.PHONY: init plan apply destroy fmt validate lint docs

ENV ?= dev
COMPONENT ?= networking

DIR := live/$(ENV)/$(COMPONENT)

init:
	cd $(DIR) && terraform init

plan:
	cd $(DIR) && terraform plan

apply:
	cd $(DIR) && terraform apply -auto-approve

destroy:
	cd $(DIR) && terraform destroy -auto-approve

fmt:
	terraform fmt -recursive

validate:
	cd $(DIR) && terraform validate

lint:
	cd $(DIR) && tflint --config=../../../.tflint.hcl

docs:
	find modules -name "*.tf" -exec dirname {} \; | sort -u | \
		xargs -I {} terraform-docs markdown table {} --output-file README.md

all-plan:
	@for comp in networking storage data messaging; do \
		echo "=== Planning $(ENV)/$$comp ==="; \
		cd live/$(ENV)/$$comp && terraform init -input=false && terraform plan && cd ../../..; \
	done

all-apply:
	@for comp in networking storage data messaging; do \
		echo "=== Applying $(ENV)/$$comp ==="; \
		cd live/$(ENV)/$$comp && terraform init -input=false && terraform apply -auto-approve && cd ../../..; \
	done
```

### Tarefa 8 — Workspaces vs Directories (Exercício Comparativo)

Crie um mini-projeto separado (`workspace-demo/`) que use workspaces para entender os trade-offs:

```bash
# Criar workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Listar
terraform workspace list

# Selecionar
terraform workspace select dev

# Usar no código
locals {
  environment = terraform.workspace
}

# Problema: como diferenciar configurações?
# Resposta: ternários e maps — fica feio rapidamente

locals {
  instance_type = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.medium"
  }[terraform.workspace]
}
```

Após experimentar, documente os trade-offs e por que directory-based é preferível para times.

---

## Critérios de Aceite

- [ ] Estrutura mono-repo implementada (`modules/` + `live/`)
- [ ] Módulo VPC criado e funcional com `for_each` para subnets
- [ ] Módulo S3 criado com versionamento, encryption e public access block
- [ ] Módulo DynamoDB criado com GSI e TTL configuráveis
- [ ] Módulo SQS criado com DLQ opcional e redrive policy
- [ ] `live/dev/` com 4 componentes (networking, storage, data, messaging)
- [ ] `live/staging/` com pelo menos 2 componentes (networking, storage)
- [ ] Cada componente com seu próprio state isolado
- [ ] Naming conventions aplicadas consistentemente
- [ ] `default_tags` no provider de cada componente
- [ ] TFLint configurado e passando
- [ ] terraform-docs gerando README para módulos
- [ ] Makefile funcional com targets padrão
- [ ] `terraform fmt -check` passa em todos os arquivos
- [ ] `terraform validate` passa em todos os componentes

---

## Definição de Pronto (DoD)

- [ ] Código valida sem erros (`terraform validate`)
- [ ] Formatação aplicada (`terraform fmt -check`)
- [ ] Lint passa (`tflint`)
- [ ] Documentação gerada (`terraform-docs`)
- [ ] Ambientes dev e staging provisionados no LocalStack
- [ ] README.md atualizado com a nova estrutura
- [ ] Commit semântico: `feat(level-1): restructure project as mono-repo with modules`

---

## Checklist

- [ ] Árvore de diretórios criada
- [ ] Módulo VPC implementado e testado
- [ ] Módulo S3 implementado e testado
- [ ] Módulo DynamoDB implementado e testado
- [ ] Módulo SQS implementado e testado
- [ ] `live/dev/` todos os componentes aplicados
- [ ] `live/staging/` pelo menos networking e storage aplicados
- [ ] TFLint configurado e executado
- [ ] terraform-docs gerando READMEs
- [ ] Makefile implementado e funcional
- [ ] Pre-commit hooks configurados
- [ ] Exercício de workspaces realizado e trade-offs documentados

---

## Extensões Opcionais

- [ ] Adicionar Terragrunt para DRY multi-environment com `terragrunt.hcl` raiz
- [ ] Criar módulo `security-group` genérico e usar em networking
- [ ] Implementar `data.terraform_remote_state` entre componentes (ex: data lendo output de networking)
- [ ] Criar shell script de bootstrap que provisiona toda a infra em ordem
- [ ] Usar `terraform graph` para visualizar dependências entre componentes
