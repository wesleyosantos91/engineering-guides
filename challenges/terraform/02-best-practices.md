# Level 2 — Boas Práticas

> **Objetivo:** Aplicar boas práticas avançadas do Terraform — validações customizadas, lifecycle rules, check blocks, ephemeral values, moved blocks, dynamic blocks, import e provider-defined functions.

---

## Objetivo de Aprendizado

- Implementar validações customizadas em variáveis com `validation` blocks
- Usar `check` blocks para validações pós-apply
- Entender e aplicar lifecycle rules (`prevent_destroy`, `create_before_destroy`, `ignore_changes`)
- Usar `moved` blocks para refatoração segura (sem destroy+create)
- Implementar dynamic blocks para configurações repetitivas
- Usar `terraform_data` como substituto do `null_resource`
- Usar `removed` blocks para remover recursos do gerenciamento
- Aplicar `for_each` vs `count` corretamente
- Usar `depends_on` com critério (apenas quando necessário)
- Configurar Infracost para estimativa de custos
- (Terraform 1.10+) Usar `ephemeral` values para secrets que não persistem no state

---

## Escopo Funcional

### Infraestrutura ShopFlow — Endurecimento

Evolua os módulos do Level 1 aplicando boas práticas avançadas:

1. **Adicionar validações** em todas as variáveis que aceitam valores restritos
2. **Check blocks** para validar que recursos criados estão saudáveis pós-apply
3. **Lifecycle rules** em recursos críticos (DynamoDB, S3)
4. **Novos recursos**: Lambda Function + API Gateway (endpoints da ShopFlow)
5. **Dynamic blocks** para Security Groups com regras dinâmicas
6. **Import** de recurso existente (criar manualmente e importar)

---

## Escopo Técnico

### Validações Customizadas

```hcl
# Exercício 1: Validar environment
variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

# Exercício 2: Validar CIDR
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid CIDR notation (e.g., 10.0.0.0/16)."
  }

  validation {
    condition     = tonumber(split("/", var.vpc_cidr)[1]) <= 24
    error_message = "CIDR prefix must be /24 or larger (e.g., /16, /20)."
  }
}

# Exercício 3: Validar instance type
variable "lambda_memory" {
  description = "Lambda function memory in MB"
  type        = number
  default     = 128

  validation {
    condition     = var.lambda_memory >= 128 && var.lambda_memory <= 10240
    error_message = "Lambda memory must be between 128 MB and 10240 MB."
  }

  validation {
    condition     = var.lambda_memory % 64 == 0
    error_message = "Lambda memory must be a multiple of 64 MB."
  }
}

# Exercício 4: Validar objeto complexo
variable "dynamodb_config" {
  description = "DynamoDB table configuration"
  type = object({
    hash_key     = string
    range_key    = optional(string)
    billing_mode = optional(string, "PAY_PER_REQUEST")
  })

  validation {
    condition     = contains(["PAY_PER_REQUEST", "PROVISIONED"], var.dynamodb_config.billing_mode)
    error_message = "Billing mode must be PAY_PER_REQUEST or PROVISIONED."
  }
}
```

### Check Blocks

```hcl
# Exercício 5: Validar que S3 bucket existe e tem versionamento
check "s3_assets_versioning" {
  data "aws_s3_bucket" "assets_check" {
    bucket = aws_s3_bucket.assets.id
  }

  assert {
    condition     = data.aws_s3_bucket.assets_check.id != ""
    error_message = "Assets bucket must exist after apply."
  }
}

# Exercício 6: Validar que SQS queue está ativa
check "sqs_orders_active" {
  data "aws_sqs_queue" "orders_check" {
    name = aws_sqs_queue.orders.name
  }

  assert {
    condition     = data.aws_sqs_queue.orders_check.url != ""
    error_message = "Orders queue must be accessible."
  }
}
```

---

## Tarefas

### Tarefa 1 — Validações em Todos os Módulos

Adicione `validation` blocks em todas as variáveis dos módulos criados no Level 1:

1. **VPC Module**: Validar CIDR, validar que `azs` não está vazio, validar que subnet CIDRs são válidos
2. **S3 Module**: Validar bucket name (apenas letras minúsculas, números e hífens)
3. **DynamoDB Module**: Validar billing mode, validar que hash_key não está vazio
4. **SQS Module**: Validar visibility timeout (0-43200), validar retention (60-1209600)

### Tarefa 2 — Lifecycle Rules

```hcl
# 1. Proteger DynamoDB tables de destruição acidental
resource "aws_dynamodb_table" "products" {
  # ... config ...

  lifecycle {
    prevent_destroy = true
  }
}

# 2. Proteger S3 buckets
resource "aws_s3_bucket" "assets" {
  # ... config ...

  lifecycle {
    prevent_destroy = true
  }
}

# 3. Ignorar mudanças externas em tags aplicadas por automação
resource "aws_dynamodb_table" "sessions" {
  # ... config ...

  lifecycle {
    ignore_changes = [
      tags["UpdatedAt"],
      tags["LastModifiedBy"],
    ]
  }
}
```

Teste: Execute `terraform destroy` e verifique que recursos protegidos são recusados.

### Tarefa 3 — Lambda Function para ShopFlow

Crie um módulo `modules/compute/lambda/` e provisione funções Lambda:

```hcl
# lambda/main.tf
resource "aws_lambda_function" "this" {
  function_name = var.function_name
  runtime       = var.runtime
  handler       = var.handler
  role          = var.role_arn
  memory_size   = var.memory_size
  timeout       = var.timeout

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

  tags = var.tags
}
```

**Funções a criar:**

| Função | Handler | Propósito |
|--------|---------|-----------|
| `shopflow-dev-process-order` | `handler.process_order` | Processa pedidos da fila SQS |
| `shopflow-dev-update-catalog` | `handler.update_catalog` | Atualiza catálogo no DynamoDB |
| `shopflow-dev-notify-customer` | `handler.notify_customer` | Envia notificações via SNS |

**Código Lambda dummy (Python):**

```python
# lambda/src/handler.py
import json

def process_order(event, context):
    print(f"Processing order: {json.dumps(event)}")
    return {"statusCode": 200, "body": json.dumps({"message": "Order processed"})}

def update_catalog(event, context):
    print(f"Updating catalog: {json.dumps(event)}")
    return {"statusCode": 200, "body": json.dumps({"message": "Catalog updated"})}

def notify_customer(event, context):
    print(f"Notifying customer: {json.dumps(event)}")
    return {"statusCode": 200, "body": json.dumps({"message": "Customer notified"})}
```

### Tarefa 4 — Dynamic Blocks para Security Groups

```hcl
# Exercício: Security Group com regras dinâmicas
variable "ingress_rules" {
  description = "List of ingress rules"
  type = list(object({
    description = string
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    {
      description = "HTTP"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      description = "HTTPS"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
  ]
}

resource "aws_security_group" "alb" {
  name        = "${local.name_prefix}-alb-sg"
  description = "Security group for ALB"
  vpc_id      = module.vpc.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      description = ingress.value.description
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Tarefa 5 — Moved Blocks (Refatoração Segura)

Simule uma refatoração: renomear um recurso sem destruí-lo.

```hcl
# Antes: recurso com nome ruim
resource "aws_s3_bucket" "bucket1" {
  bucket = "${local.name_prefix}-assets"
}

# Depois: renomear para nome descritivo
resource "aws_s3_bucket" "application_assets" {
  bucket = "${local.name_prefix}-assets"
}

# moved block para migrar state
moved {
  from = aws_s3_bucket.bucket1
  to   = aws_s3_bucket.application_assets
}
```

1. Crie um recurso com nome genérico
2. Execute `terraform apply`
3. Renomeie o recurso e adicione o `moved` block
4. Execute `terraform plan` — verifique que NÃO há destroy
5. Execute `terraform apply`
6. Remova o `moved` block (após apply, não é mais necessário)

### Tarefa 6 — Import de Recurso Existente

```bash
# 1. Criar recurso manualmente via AWS CLI
aws --endpoint-url=http://localhost:4566 s3 mb s3://shopflow-dev-manual-bucket

# 2. Criar o resource block correspondente
resource "aws_s3_bucket" "imported_bucket" {
  bucket = "shopflow-dev-manual-bucket"
}

# 3. Importar
terraform import aws_s3_bucket.imported_bucket shopflow-dev-manual-bucket

# 4. Verificar
terraform state show aws_s3_bucket.imported_bucket
terraform plan  # Deve mostrar "No changes"
```

**Alternativa — import block (Terraform 1.5+):**

```hcl
import {
  to = aws_s3_bucket.imported_bucket
  id = "shopflow-dev-manual-bucket"
}

resource "aws_s3_bucket" "imported_bucket" {
  bucket = "shopflow-dev-manual-bucket"
}
```

### Tarefa 7 — terraform_data (Substituto do null_resource)

```hcl
# Executar script após criação de recursos
resource "terraform_data" "seed_products" {
  triggers_replace = [
    aws_dynamodb_table.products.arn,
  ]

  provisioner "local-exec" {
    command = <<-EOT
      aws --endpoint-url=http://localhost:4566 dynamodb put-item \
        --table-name ${aws_dynamodb_table.products.name} \
        --item '{"product_id": {"S": "PROD-001"}, "category": {"S": "electronics"}, "name": {"S": "Laptop"}, "price": {"N": "2999.90"}}'
    EOT
  }
}
```

### Tarefa 8 — Removed Blocks

```hcl
# Remover recurso do gerenciamento Terraform sem destruí-lo na AWS
removed {
  from = aws_s3_bucket.imported_bucket

  lifecycle {
    destroy = false  # Não destruir o recurso real
  }
}
```

### Tarefa 9 — Infracost (Estimativa de Custo)

```bash
# Instalar Infracost
brew install infracost    # macOS
choco install infracost   # Windows

# Registrar (free tier)
infracost auth login

# Estimar custo
cd live/dev/data
infracost breakdown --path .

# Comparar custos entre environments
infracost diff --path live/prod/data --compare-to live/dev/data
```

---

## Critérios de Aceite

- [ ] Validações customizadas em todas as variáveis dos módulos (pelo menos 10 validações)
- [ ] Check blocks implementados (pelo menos 3) verificando recursos pós-apply
- [ ] `lifecycle.prevent_destroy` em DynamoDB tables e S3 buckets
- [ ] `lifecycle.ignore_changes` em pelo menos 1 recurso
- [ ] Módulo Lambda criado e 3 funções provisionadas no LocalStack
- [ ] Dynamic block implementado em Security Group
- [ ] Moved block exercitado: renomear recurso sem destroy
- [ ] Import exercitado: recurso criado via CLI e importado no Terraform
- [ ] `terraform_data` usado para seed de dados
- [ ] `removed` block exercitado
- [ ] Infracost executado e output documentado
- [ ] `terraform fmt -check` passa em todos os arquivos
- [ ] `terraform validate` passa em todos os componentes
- [ ] Todos os módulos aplicados com sucesso no LocalStack

---

## Definição de Pronto (DoD)

- [ ] Código valida sem erros
- [ ] Formatação aplicada
- [ ] Validações cobrem todos os inputs críticos
- [ ] Check blocks passam pós-apply
- [ ] Lifecycle rules testadas (tentar destroy e verificar que é bloqueado)
- [ ] Documento `DECISIONS.md` com justificativa de cada lifecycle rule
- [ ] Commit semântico: `feat(level-2): apply best practices - validations, lifecycle, check blocks`

---

## Checklist

- [ ] Validações adicionadas em todos os módulos
- [ ] Check blocks implementados e passando
- [ ] Lifecycle rules configuradas em recursos críticos
- [ ] Lambda module criado e funções provisionadas
- [ ] Dynamic blocks em Security Groups
- [ ] Moved block exercitado com sucesso
- [ ] Import (CLI e import block) exercitado
- [ ] terraform_data usado para provisioning auxiliar
- [ ] Removed block exercitado
- [ ] Infracost executado

---

## Extensões Opcionais

- [ ] Implementar `precondition` e `postcondition` em outputs dos módulos
- [ ] Usar `provider-defined functions` (Terraform 1.10+ / AWS Provider 5.40+): `provider::aws::arn_parse`
- [ ] Criar um `check` block que verifica DNS resolution de um endpoint
- [ ] Implementar `ephemeral` variables para secrets (Terraform 1.10+)
- [ ] Criar policy de Infracost que falha se custo estimado > threshold
- [ ] Implementar `replace_triggered_by` em lifecycle para forçar recreação
