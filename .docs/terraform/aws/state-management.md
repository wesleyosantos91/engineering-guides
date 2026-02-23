````markdown
# State Management — Terraform AWS

> **Objetivo:** Guia completo para gerenciamento do Terraform state na AWS,
> incluindo backends, locking, isolation, migração e disaster recovery.
> O state é a parte mais crítica de qualquer projeto Terraform.

---

## Sumário

- [O que é o State](#o-que-é-o-state)
- [Backend S3 + DynamoDB](#backend-s3--dynamodb)
- [Criação do Backend (Bootstrap)](#criação-do-backend-bootstrap)
- [Isolamento de State](#isolamento-de-state)
- [Remote State como Data Source](#remote-state-como-data-source)
- [State Locking](#state-locking)
- [Migração de State](#migração-de-state)
- [Comandos de State](#comandos-de-state)
- [Disaster Recovery](#disaster-recovery)
- [Segurança do State](#segurança-do-state)
- [Anti-Patterns](#anti-patterns)

---

## O que é o State

O state (`.tfstate`) é um arquivo JSON que mapeia recursos Terraform para recursos reais na cloud.

```
Terraform Code ──► terraform plan ──► Compara Code vs State ──► Gera diff
                                            │
                                      terraform.tfstate
                                            │
                               Mapeia para recursos reais na AWS
```

### Por que o State é Crítico

- **Sem state, Terraform não sabe o que já existe** — tentaria criar tudo de novo
- **Contém dados sensíveis** — senhas de RDS, chaves de API, endpoints
- **Corrupção = downtime** — state corrompido pode causar destroys indesejados
- **Concorrência = desastre** — dois applies simultâneos corrompem o state

---

## Backend S3 + DynamoDB

### Configuração Padrão

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/networking/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true

    # Opcional — assume role para acesso cross-account
    # role_arn = "arn:aws:iam::123456789012:role/TerraformStateAccess"
  }
}
```

### Por que S3 + DynamoDB?

| Requisito | S3 | DynamoDB |
|-----------|-----|----------|
| Armazenamento durável | ✅ 99.999999999% | - |
| Versionamento | ✅ Nativo | - |
| Encryption at rest | ✅ SSE-S3/KMS | - |
| Locking distribuído | - | ✅ Conditional writes |
| Baixo custo | ✅ ~$0.01/mês | ✅ ~$0.01/mês |

---

## Criação do Backend (Bootstrap)

O backend precisa existir **antes** de ser usado. Use um script de bootstrap:

```hcl
# bootstrap/main.tf — Execute manualmente uma única vez

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state"

  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name      = "Terraform State"
    ManagedBy = "manual"
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    id     = "cleanup-old-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Name      = "Terraform Locks"
    ManagedBy = "manual"
  }
}

resource "aws_kms_key" "terraform_state" {
  description             = "KMS key for Terraform state encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  tags = {
    Name      = "terraform-state-key"
    ManagedBy = "manual"
  }
}

resource "aws_kms_alias" "terraform_state" {
  name          = "alias/terraform-state"
  target_key_id = aws_kms_key.terraform_state.key_id
}
```

```bash
# Bootstrap — execute uma única vez
cd bootstrap
terraform init
terraform apply

# Depois, nunca mais toque nesses recursos manualmente
```

---

## Isolamento de State

### Estratégia: Um State por Componente por Ambiente

```
S3 Key Structure:
mycompany-terraform-state/
├── _global/
│   └── iam/terraform.tfstate
├── dev/
│   ├── networking/terraform.tfstate
│   ├── compute/terraform.tfstate
│   ├── data/terraform.tfstate
│   └── messaging/terraform.tfstate
├── staging/
│   ├── networking/terraform.tfstate
│   ├── compute/terraform.tfstate
│   ├── data/terraform.tfstate
│   └── messaging/terraform.tfstate
└── prod/
    ├── networking/terraform.tfstate
    ├── compute/terraform.tfstate
    ├── data/terraform.tfstate
    └── messaging/terraform.tfstate
```

### Benefícios do Isolamento

| Aspecto | State Monolítico | State Isolado |
|---------|-----------------|---------------|
| Blast radius | ALTO — qualquer erro afeta tudo | BAIXO — erro limitado ao componente |
| Tempo de plan | Lento (100+ recursos) | Rápido (10-30 recursos) |
| Permissões | Tudo ou nada | Granular por componente |
| Concorrência | Bloqueio total | Locks independentes |
| Rollback | Complexo | Simples por componente |

### Dependências entre States

```
networking/ ──► compute/ ──► monitoring/
     │              │
     └──► data/ ────┘
```

Use `terraform_remote_state` ou **outputs no SSM** para comunicação entre states.

---

## Remote State como Data Source

### Opção 1: terraform_remote_state

```hcl
# live/dev/compute/data.tf

data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "dev/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Uso
resource "aws_ecs_service" "app" {
  # ...
  network_configuration {
    subnets = data.terraform_remote_state.networking.outputs.private_subnet_ids
  }
}
```

### Opção 2: SSM Parameter Store (Recomendado)

```hcl
# No módulo de networking — publica valores no SSM
resource "aws_ssm_parameter" "vpc_id" {
  name  = "/${var.environment}/networking/vpc_id"
  type  = "String"
  value = aws_vpc.main.id
}

resource "aws_ssm_parameter" "private_subnet_ids" {
  name  = "/${var.environment}/networking/private_subnet_ids"
  type  = "StringList"
  value = join(",", aws_subnet.private[*].id)
}
```

```hcl
# No módulo de compute — lê do SSM
data "aws_ssm_parameter" "vpc_id" {
  name = "/${var.environment}/networking/vpc_id"
}

data "aws_ssm_parameter" "private_subnet_ids" {
  name = "/${var.environment}/networking/private_subnet_ids"
}

locals {
  private_subnet_ids = split(",", data.aws_ssm_parameter.private_subnet_ids.value)
}
```

> **Vantagem do SSM:** Desacopla completamente os states. Não precisa saber
> o bucket/key do state de outro componente.

---

## State Locking

### Como Funciona

```
Developer A: terraform apply
  ├── 1. Tenta adquirir lock no DynamoDB
  ├── 2. Lock adquirido ✅
  ├── 3. Executa apply
  ├── 4. Atualiza state no S3
  └── 5. Libera lock no DynamoDB

Developer B: terraform apply (ao mesmo tempo)
  ├── 1. Tenta adquirir lock no DynamoDB
  └── 2. LOCK FALHOU ❌ — "Error acquiring the state lock"
```

### Forçar Unlock (Emergência)

```bash
# ⚠️ CUIDADO — apenas quando o lock está orphaned
terraform force-unlock LOCK_ID

# O LOCK_ID aparece na mensagem de erro
# Error: Error acquiring the state lock
# Lock Info:
#   ID:        xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

---

## Migração de State

### Local para S3

```bash
# 1. Adicione o backend config
# backend.tf
# terraform { backend "s3" { ... } }

# 2. Re-init — Terraform pergunta se quer migrar
terraform init -migrate-state

# 3. Confirme a migração
# "Do you want to copy existing state to the new backend?" → yes
```

### Mover Recurso entre States

```bash
# Cenário: Mover VPC do state monolítico para o state de networking

# 1. Do state origem — remova (sem destruir o recurso)
cd live/dev/monolith
terraform state rm aws_vpc.main

# 2. No state destino — importe
cd live/dev/networking
terraform import aws_vpc.main vpc-0123456789abcdef0
```

### Renomear Recurso no State

```bash
terraform state mv aws_s3_bucket.old_name aws_s3_bucket.new_name

# Ou melhor — use moved block no código (declarativo)
moved {
  from = aws_s3_bucket.old_name
  to   = aws_s3_bucket.new_name
}
```

---

## Comandos de State

```bash
# Listar todos os recursos no state
terraform state list

# Ver detalhes de um recurso
terraform state show aws_vpc.main

# Remover recurso do state (sem destruir na AWS)
terraform state rm aws_s3_bucket.legacy

# Mover recurso (renomear)
terraform state mv aws_s3_bucket.old aws_s3_bucket.new

# Pull — download do state remoto
terraform state pull > state-backup.json

# Push — upload de state local para remoto (⚠️ PERIGOSO)
terraform state push state-backup.json

# Refresh — sincroniza state com a realidade da AWS
terraform refresh
# Ou melhor (Terraform 1.5+):
terraform apply -refresh-only
```

---

## Disaster Recovery

### Backup via S3 Versioning

```bash
# Listar versões do state
aws s3api list-object-versions \
  --bucket mycompany-terraform-state \
  --prefix "prod/networking/terraform.tfstate" \
  --max-items 5

# Restaurar versão anterior
aws s3api get-object \
  --bucket mycompany-terraform-state \
  --key "prod/networking/terraform.tfstate" \
  --version-id "VERSION_ID" \
  restored-state.json

# Push do state restaurado
terraform state push restored-state.json
```

### Cross-Region Replication

```hcl
# Bucket de backup em outra região
resource "aws_s3_bucket" "terraform_state_backup" {
  provider = aws.backup_region
  bucket   = "mycompany-terraform-state-backup"
}

resource "aws_s3_bucket_replication_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "replicate-state"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.terraform_state_backup.arn
      storage_class = "STANDARD_IA"
    }
  }
}
```

### Procedimento de Recovery

```
1. Identifique o problema (state corrompido, deleção acidental)
2. PARE — ninguém faz apply
3. Identifique a última versão boa do state (S3 versioning)
4. Faça download da versão boa
5. Valide o state: terraform state list
6. Push do state restaurado
7. Execute: terraform plan (deve mostrar "No changes")
8. Libere o acesso
```

---

## Segurança do State

### Encryption

```hcl
# S3 — encryption at rest
backend "s3" {
  encrypt = true  # SSE-S3 por padrão
  # Para KMS:
  # kms_key_id = "alias/terraform-state"
}
```

### Acesso

```hcl
# IAM Policy — acesso mínimo para CI/CD
data "aws_iam_policy_document" "terraform_state" {
  # Leitura e escrita do state
  statement {
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject",
    ]
    resources = [
      "arn:aws:s3:::mycompany-terraform-state/prod/*",
    ]
  }

  # Listar bucket
  statement {
    effect    = "Allow"
    actions   = ["s3:ListBucket"]
    resources = ["arn:aws:s3:::mycompany-terraform-state"]
  }

  # DynamoDB locking
  statement {
    effect = "Allow"
    actions = [
      "dynamodb:GetItem",
      "dynamodb:PutItem",
      "dynamodb:DeleteItem",
    ]
    resources = [
      "arn:aws:dynamodb:us-east-1:*:table/terraform-locks",
    ]
  }
}
```

### Segregação por Ambiente

```hcl
# Developers — apenas dev/staging
# CI/CD prod — apenas prod
# Platform team — todos

# Exemplo: Policy para developers
statement {
  effect = "Allow"
  actions = [
    "s3:GetObject",
    "s3:PutObject",
  ]
  resources = [
    "arn:aws:s3:::mycompany-terraform-state/dev/*",
    "arn:aws:s3:::mycompany-terraform-state/staging/*",
  ]
  # Sem acesso a prod/*
}
```

---

## Anti-Patterns

### ❌ State Local em Produção

```hcl
# NUNCA em produção
terraform {
  backend "local" {}
}
```

### ❌ State Monolítico

```
# RUIM — um state para toda a infra
terraform.tfstate → VPC + ECS + RDS + IAM + S3 + CloudWatch + ...
```

### ❌ Editar State Manualmente

```bash
# NUNCA faça isso
vim terraform.tfstate  # ❌ JAMAIS
```

### ❌ Compartilhar State sem Lock

```hcl
# RUIM — S3 sem DynamoDB
backend "s3" {
  bucket = "my-state"
  # Sem dynamodb_table = PERIGO
}
```

### ❌ State no Git

```bash
# .gitignore — OBRIGATÓRIO
*.tfstate
*.tfstate.backup
*.tfstate.*.backup
.terraform/
```

````
