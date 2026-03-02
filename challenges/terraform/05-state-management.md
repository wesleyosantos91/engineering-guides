# Level 5 — State Management

> **Objetivo:** Dominar o gerenciamento do Terraform state — backends remotos, locking, isolamento por domínio e ambiente, migração de state, operações de emergência, disaster recovery e compartilhamento de dados entre states via remote state data source.

---

## Objetivo de Aprendizado

- Configurar backend S3 para state remoto (simulado no LocalStack)
- Implementar state locking nativo (Terraform 1.10+ `use_lockfile`)
- Entender e aplicar isolamento de state por domínio e ambiente
- Compartilhar dados entre states via `terraform_remote_state` e SSM Parameter Store
- Executar operações de state: `mv`, `rm`, `import`, `replace-provider`
- Implementar estratégia de disaster recovery para state
- Migrar state entre backends
- Resolver problemas de state travado (locked state)
- Entender anti-patterns de state management

---

## Escopo Funcional

### ShopFlow — State Isolation Architecture

```
                    ┌─────────────────────────────────────┐
                    │      S3: shopflow-terraform-state    │
                    │                                       │
                    │  ├── _global/iam/terraform.tfstate    │
                    │  ├── _global/kms/terraform.tfstate    │
                    │  │                                     │
                    │  ├── dev/                               │
                    │  │   ├── networking/terraform.tfstate   │
                    │  │   ├── storage/terraform.tfstate      │
                    │  │   ├── data/terraform.tfstate         │
                    │  │   ├── messaging/terraform.tfstate    │
                    │  │   └── compute/terraform.tfstate      │
                    │  │                                     │
                    │  ├── staging/                           │
                    │  │   ├── networking/terraform.tfstate   │
                    │  │   ├── storage/terraform.tfstate      │
                    │  │   └── ...                            │
                    │  │                                     │
                    │  └── prod/                              │
                    │      ├── networking/terraform.tfstate   │
                    │      ├── storage/terraform.tfstate      │
                    │      └── ...                            │
                    └─────────────────────────────────────┘
```

### Fluxo de Dados entre States

```
   _global/iam                 dev/networking              dev/compute
┌──────────────┐         ┌──────────────────┐        ┌──────────────────┐
│ IAM Roles    │         │ VPC, Subnets     │        │ Lambdas, API GW  │
│ KMS Keys     │         │ Security Groups  │        │                  │
│              │         │                  │        │                  │
│ Outputs:     │────────►│ Inputs:          │───────►│ Inputs:          │
│ - role_arns  │         │ - kms_key_arn    │        │ - vpc_id         │
│ - key_arns   │         │                  │        │ - subnet_ids     │
│              │         │ Outputs:         │        │ - sg_ids         │
└──────────────┘         │ - vpc_id         │        │ - role_arns      │
                         │ - subnet_ids     │        └──────────────────┘
                         │ - sg_ids         │
                         └──────────────────┘
```

---

## Escopo Técnico

### Bootstrap — Criar Backend de State

O backend S3 para state precisa existir **antes** de ser usado. No LocalStack:

```hcl
# bootstrap/main.tf — Execute uma única vez (state local)
terraform {
  required_version = ">= 1.10.0"
}

provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
  s3_use_path_style           = true

  endpoints {
    s3       = "http://localhost:4566"
    dynamodb = "http://localhost:4566"
    iam      = "http://localhost:4566"
    sts      = "http://localhost:4566"
  }
}

# Bucket para state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "shopflow-terraform-state"

  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name      = "Terraform State"
    ManagedBy = "manual-bootstrap"
    Project   = "shopflow"
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
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# (Legado) DynamoDB table para locking — apenas para exercício de migração
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "shopflow-terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name      = "Terraform Locks"
    ManagedBy = "manual-bootstrap"
  }
}
```

---

## Tarefas

### Tarefa 1 — Bootstrap do Backend

```bash
# 1. Criar diretório de bootstrap
mkdir -p bootstrap && cd bootstrap

# 2. Aplicar (state local — é o bootstrap)
terraform init
terraform apply -auto-approve

# 3. Verificar bucket criado
aws --endpoint-url=http://localhost:4566 s3 ls

# 4. Verificar versionamento
aws --endpoint-url=http://localhost:4566 s3api get-bucket-versioning \
  --bucket shopflow-terraform-state
```

### Tarefa 2 — Migrar State Local → S3

Migre todos os componentes de `live/dev/` do state local para o backend S3:

```hcl
# live/dev/networking/backend.tf
terraform {
  backend "s3" {
    bucket                      = "shopflow-terraform-state"
    key                         = "dev/networking/terraform.tfstate"
    region                      = "us-east-1"
    encrypt                     = true
    use_lockfile                = true
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true

    endpoints = {
      s3       = "http://localhost:4566"
      dynamodb = "http://localhost:4566"
      iam      = "http://localhost:4566"
      sts      = "http://localhost:4566"
    }

    access_key = "test"
    secret_key = "test"
  }
}
```

```bash
# Migrar state
cd live/dev/networking
terraform init -migrate-state

# Verificar que state foi para S3
aws --endpoint-url=http://localhost:4566 s3 ls \
  s3://shopflow-terraform-state/dev/networking/

# Verificar que state local NÃO existe mais
ls terraform.tfstate  # Deve estar vazio ou não existir

# Repetir para todos os componentes
for comp in storage data messaging compute; do
  cd ../../../live/dev/$comp
  terraform init -migrate-state -input=false
done
```

### Tarefa 3 — State Locking (Native S3 Lock)

```bash
# 1. Simular lock: iniciar apply em background
cd live/dev/networking
terraform apply -auto-approve &

# 2. Tentar segundo apply enquanto o primeiro roda
terraform apply
# Deve falhar com: "Error locking state: ..."

# 3. Verificar lock file no S3
aws --endpoint-url=http://localhost:4566 s3 ls \
  s3://shopflow-terraform-state/dev/networking/ --recursive
# Deve mostrar .tflock

# 4. (Emergência) Forçar unlock
terraform force-unlock <LOCK_ID>
```

### Tarefa 4 — Exercício: Locking com DynamoDB (Legado)

Para entender a migração, configure temporariamente um componente com DynamoDB:

```hcl
# Antes (DynamoDB)
terraform {
  backend "s3" {
    bucket         = "shopflow-terraform-state"
    key            = "dev/data/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "shopflow-terraform-locks"
    encrypt        = true
    # ... endpoints LocalStack
  }
}
```

Depois migre para native lock:

```hcl
# Depois (Native Lock — 1.10+)
terraform {
  backend "s3" {
    bucket       = "shopflow-terraform-state"
    key          = "dev/data/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
    encrypt      = true
    # ... endpoints LocalStack
  }
}
```

```bash
# Migrar
terraform init -reconfigure
```

### Tarefa 5 — Remote State como Data Source

Configure compartilhamento de dados entre states:

```hcl
# live/dev/compute/data.tf — Lê outputs de networking
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket                      = "shopflow-terraform-state"
    key                         = "dev/networking/terraform.tfstate"
    region                      = "us-east-1"
    skip_credentials_validation = true
    skip_metadata_api_check     = true

    endpoints = {
      s3 = "http://localhost:4566"
    }

    access_key = "test"
    secret_key = "test"
  }
}

# Usar dados do networking
locals {
  vpc_id            = data.terraform_remote_state.networking.outputs.vpc_id
  private_subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
  public_subnet_ids  = data.terraform_remote_state.networking.outputs.public_subnet_ids
}

# Exemplo: Lambda em subnet privada
module "order_processor" {
  source = "../../../modules/compute/lambda-function"

  function_name = "${local.name_prefix}-process-order"
  runtime       = "python3.12"
  handler       = "handler.process_order"
  zip_file      = "${path.module}/lambda/orders.zip"

  vpc_config = {
    subnet_ids         = local.private_subnet_ids
    security_group_ids = [data.terraform_remote_state.networking.outputs.lambda_sg_id]
  }

  tags = local.common_tags
}
```

### Tarefa 6 — Alternativa: SSM Parameter Store (Acoplamento Menor)

```hcl
# Em live/dev/networking/outputs.tf — Publicar no SSM
resource "aws_ssm_parameter" "vpc_id" {
  name  = "/${var.environment}/networking/vpc_id"
  type  = "String"
  value = module.vpc.vpc_id

  tags = local.common_tags
}

resource "aws_ssm_parameter" "private_subnet_ids" {
  name  = "/${var.environment}/networking/private_subnet_ids"
  type  = "StringList"
  value = join(",", module.vpc.private_subnet_ids)

  tags = local.common_tags
}
```

```hcl
# Em live/dev/compute/data.tf — Ler do SSM
data "aws_ssm_parameter" "vpc_id" {
  name = "/${var.environment}/networking/vpc_id"
}

data "aws_ssm_parameter" "private_subnet_ids" {
  name = "/${var.environment}/networking/private_subnet_ids"
}

locals {
  vpc_id             = data.aws_ssm_parameter.vpc_id.value
  private_subnet_ids = split(",", data.aws_ssm_parameter.private_subnet_ids.value)
}
```

### Tarefa 7 — Operações de State

```bash
# 1. Listar todos os recursos no state
terraform state list

# 2. Mostrar detalhes de um recurso
terraform state show module.vpc.aws_vpc.main

# 3. Mover recurso (renomear no state)
terraform state mv \
  module.vpc.aws_subnet.public["public-0"] \
  module.vpc.aws_subnet.public["us-east-1a"]

# 4. Remover recurso do state (sem destruir)
terraform state rm aws_s3_bucket.temporary

# 5. Pull — baixar state remoto
terraform state pull > state_backup.json

# 6. Push — subir state (CUIDADO — operação destrutiva)
# terraform state push state_backup.json

# 7. Verificar tamanho do state
terraform state pull | wc -c
terraform state pull | jq '.resources | length'
```

### Tarefa 8 — Disaster Recovery

```bash
# 1. Backup manual do state
terraform state pull > backups/dev-networking-$(date +%Y%m%d-%H%M%S).tfstate

# 2. Listar versões do state no S3 (versionamento)
aws --endpoint-url=http://localhost:4566 s3api list-object-versions \
  --bucket shopflow-terraform-state \
  --prefix dev/networking/terraform.tfstate

# 3. Restaurar versão anterior
aws --endpoint-url=http://localhost:4566 s3api get-object \
  --bucket shopflow-terraform-state \
  --key dev/networking/terraform.tfstate \
  --version-id <VERSION_ID> \
  restored-state.tfstate

# 4. Verificar integridade
cat restored-state.tfstate | jq '.version, .serial, (.resources | length)'

# 5. Simular recovery: Importar de backup
cp restored-state.tfstate terraform.tfstate
terraform init -reconfigure
terraform plan  # Deve mostrar "No changes"
```

### Tarefa 9 — Script de Aplicação Ordenada

Crie um script que aplica componentes na ordem correta respeitando dependências:

```bash
#!/bin/bash
# scripts/apply-all.sh
set -euo pipefail

ENV="${1:-dev}"
BASE_DIR="live/${ENV}"

echo "=== Applying ShopFlow infrastructure for environment: ${ENV} ==="

# Ordem de aplicação (respeita dependências)
COMPONENTS=(
  "_global/iam"
  "${BASE_DIR}/networking"
  "${BASE_DIR}/storage"
  "${BASE_DIR}/data"
  "${BASE_DIR}/messaging"
  "${BASE_DIR}/compute"
)

for component in "${COMPONENTS[@]}"; do
  dir="live/${component}"
  if [ -d "$dir" ]; then
    echo ""
    echo ">>> Applying: ${component}"
    cd "$dir"
    terraform init -input=false
    terraform apply -auto-approve
    cd -
  else
    echo ">>> Skipping (not found): ${component}"
  fi
done

echo ""
echo "=== All components applied successfully ==="
```

---

## Critérios de Aceite

- [ ] Bootstrap do backend S3 executado com sucesso
- [ ] Todos os componentes migrados de state local para S3
- [ ] State locking nativo (`use_lockfile`) funcionando
- [ ] Exercício de DynamoDB locking → migração para native lock completado
- [ ] `terraform_remote_state` configurado entre pelo menos 2 componentes
- [ ] SSM Parameter Store como alternativa de compartilhamento implementado
- [ ] Operações de state exercitadas: `list`, `show`, `mv`, `rm`, `pull`
- [ ] Backup e restore de state exercitados
- [ ] Versionamento de state no S3 verificado
- [ ] Script de aplicação ordenada funcional
- [ ] Todos os componentes funcionando com backend remoto no LocalStack

---

## Definição de Pronto (DoD)

- [ ] Backend S3 provisionado e funcional
- [ ] Todos os states em S3 com locking nativo
- [ ] Remote state data sources configurados
- [ ] Operações de emergência documentadas no README
- [ ] Script de apply ordenado funcional
- [ ] Commit semântico: `feat(level-5): implement state management with S3 backend and isolation`

---

## Checklist

- [ ] Bootstrap executado
- [ ] State migrado para S3
- [ ] Locking nativo testado
- [ ] Remote state data sources configurados
- [ ] SSM parameters publicados e consumidos
- [ ] Operações de state exercitadas
- [ ] Backup/restore testado
- [ ] Script de apply ordenado criado
- [ ] Anti-patterns documentados

---

## Anti-Patterns de State Management

| Anti-Pattern | Risco | Alternativa |
|-------------|-------|-------------|
| **State local em produção** | Perda de dados, sem locking | Backend remoto (S3) |
| **State monolítico** | Blast radius enorme, apply lento | Isolamento por domínio/ambiente |
| **State sem versionamento** | Impossível fazer rollback | S3 versioning habilitado |
| **State sem encryption** | Dados sensíveis expostos | `encrypt = true` sempre |
| **State sem locking** | Corrupção por aplys concorrentes | `use_lockfile = true` |
| **`terraform state push` sem backup** | Sobrescrever state = desastre | Sempre backup antes |
| **DynamoDB para locking (novos projetos)** | Complexidade desnecessária | Native S3 lock (1.10+) |

---

## Extensões Opcionais

- [ ] Implementar cross-account state access com assume role
- [ ] Criar módulo de bootstrap reutilizável para novos projetos
- [ ] Implementar state cleanup: identificar recursos órfãos
- [ ] Configurar notificações quando state muda (S3 event → SNS)
- [ ] Explorar `terraform show -json` para análise programática do state
- [ ] Criar dashboard com métricas de state (tamanho, recursos, última modificação)
