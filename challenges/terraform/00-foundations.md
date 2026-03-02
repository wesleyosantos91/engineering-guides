# Level 0 — Fundamentos Terraform + LocalStack

> **Objetivo:** Dominar os fundamentos do Terraform (HCL, CLI, workflow) e configurar o ambiente LocalStack para simulação de serviços AWS. Criar os primeiros recursos reais.

---

## Objetivo de Aprendizado

- Instalar e configurar Terraform 1.10+ e LocalStack
- Entender o workflow Terraform: `init → plan → apply → destroy`
- Dominar a sintaxe HCL: resources, variables, outputs, locals, data sources
- Compreender o conceito de state e seu papel no Terraform
- Criar recursos básicos na AWS (via LocalStack): S3, SQS, DynamoDB
- Usar `terraform fmt`, `terraform validate` e `terraform console`
- Entender providers, versioning e lock files

---

## Escopo Funcional

### Infraestrutura ShopFlow — Fundação

Neste nível, crie os seguintes recursos para a plataforma ShopFlow:

1. **S3 Bucket** — Armazenamento de assets estáticos (`shopflow-dev-assets`)
2. **S3 Bucket** — Armazenamento de logs (`shopflow-dev-logs`)
3. **SQS Queue** — Fila de processamento de pedidos (`shopflow-dev-orders`)
4. **SQS Dead Letter Queue** — DLQ para pedidos com falha (`shopflow-dev-orders-dlq`)
5. **DynamoDB Table** — Catálogo de produtos (`shopflow-dev-products`)
6. **DynamoDB Table** — Sessões de usuário (`shopflow-dev-sessions`)

---

## Escopo Técnico

### Setup do Ambiente

| Ferramenta | Versão | Propósito |
|-----------|--------|-----------|
| **Terraform** | 1.10+ | IaC engine |
| **Docker** | 24+ | Runtime para LocalStack |
| **LocalStack** | latest | Simulação AWS local |
| **AWS CLI** | 2.x | Validação de recursos (opcional) |
| **tfenv** (opcional) | latest | Gerenciador de versões do Terraform |

### Conceitos a Dominar

| Conceito | O que aprender |
|----------|---------------|
| **HCL** | Blocos `resource`, `variable`, `output`, `locals`, `data`, `terraform` |
| **Workflow** | `terraform init` → `plan` → `apply` → `destroy` |
| **State** | `terraform.tfstate`, `terraform show`, `terraform state list` |
| **Providers** | Declaração, versioning, endpoints customizados (LocalStack) |
| **Variables** | `type`, `default`, `description`, `sensitive`, `validation` |
| **Outputs** | `value`, `description`, `sensitive` |
| **Locals** | Computações, name prefixes, tag merging |
| **Data Sources** | `aws_caller_identity`, `aws_region`, `aws_availability_zones` |
| **CLI** | `fmt`, `validate`, `console`, `show`, `output`, `graph` |
| **Lock File** | `.terraform.lock.hcl` — reproduzibilidade de builds |

---

## Tarefas

### Tarefa 1 — Setup do Ambiente

```bash
# 1. Instalar Terraform
# macOS
brew install terraform

# Windows
choco install terraform

# Linux
sudo apt-get install -y terraform

# Verificar
terraform version
# Deve ser >= 1.10.0

# 2. Subir LocalStack
docker compose up -d

# 3. Verificar LocalStack
curl http://localhost:4566/_localstack/health
# Deve retornar JSON com serviços disponíveis

# 4. (Opcional) Configurar AWS CLI para LocalStack
aws configure --profile localstack
# AWS Access Key ID: test
# AWS Secret Access Key: test
# Default region: us-east-1
# Default output: json

# Testar
aws --endpoint-url=http://localhost:4566 s3 ls --profile localstack
```

### Tarefa 2 — Primeiro Recurso: S3 Bucket

Crie a estrutura de arquivos:

```
shopflow-infra/
├── docker-compose.yml       # LocalStack
├── main.tf                  # Recursos
├── variables.tf             # Variáveis
├── outputs.tf               # Outputs
├── locals.tf                # Locals
├── versions.tf              # Provider e versões
└── terraform.tfvars         # Valores das variáveis
```

**versions.tf:**

```hcl
terraform {
  required_version = ">= 1.10.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region                      = var.aws_region
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
  s3_use_path_style           = true

  endpoints {
    s3       = var.localstack_endpoint
    sqs      = var.localstack_endpoint
    dynamodb = var.localstack_endpoint
    iam      = var.localstack_endpoint
    sts      = var.localstack_endpoint
  }

  default_tags {
    tags = local.common_tags
  }
}
```

**Exercícios para a Tarefa 2:**

1. Crie o S3 bucket `shopflow-dev-assets` com versionamento habilitado
2. Crie o S3 bucket `shopflow-dev-logs` com lifecycle rule (expirar objetos após 30 dias)
3. Adicione `variables.tf` com variáveis `project`, `environment`, `aws_region`, `localstack_endpoint`
4. Crie `locals.tf` com `name_prefix` e `common_tags`
5. Exporte os ARNs e nomes dos buckets em `outputs.tf`
6. Execute o workflow completo: `init → fmt → validate → plan → apply`
7. Verifique os recursos criados: `terraform state list` e `terraform output`

### Tarefa 3 — SQS Queues

1. Crie a fila SQS `shopflow-dev-orders` com:
   - `visibility_timeout_seconds = 30`
   - `message_retention_seconds = 86400` (1 dia)
   - Redrive policy apontando para a DLQ
2. Crie a DLQ `shopflow-dev-orders-dlq` com:
   - `message_retention_seconds = 1209600` (14 dias)
   - `max_receive_count = 3` na redrive policy da fila principal
3. Exporte URLs e ARNs das filas

### Tarefa 4 — DynamoDB Tables

1. Crie a tabela `shopflow-dev-products` com:
   - Hash key: `product_id` (S)
   - Range key: `category` (S)
   - Billing mode: `PAY_PER_REQUEST`
   - GSI: `category-price-index` (hash: `category`, range: `price`)
2. Crie a tabela `shopflow-dev-sessions` com:
   - Hash key: `session_id` (S)
   - TTL habilitado no atributo `expires_at`
   - Billing mode: `PAY_PER_REQUEST`
3. Exporte nomes e ARNs das tabelas

### Tarefa 5 — Terraform Console e Exploração

```bash
# 1. Explorar o state
terraform state list
terraform state show aws_s3_bucket.assets
terraform show -json | jq '.values.root_module.resources'

# 2. Usar terraform console
terraform console
> var.project
> local.name_prefix
> aws_s3_bucket.assets.arn
> aws_sqs_queue.orders.url
> [for t in aws_dynamodb_table.products.attribute : t.name]

# 3. Gerar grafo de dependências
terraform graph | dot -Tpng > graph.png

# 4. Destruir tudo
terraform destroy
```

### Tarefa 6 — Validação com AWS CLI

```bash
# Listar buckets S3
aws --endpoint-url=http://localhost:4566 s3 ls

# Listar filas SQS
aws --endpoint-url=http://localhost:4566 sqs list-queues

# Descrever tabela DynamoDB
aws --endpoint-url=http://localhost:4566 dynamodb describe-table \
  --table-name shopflow-dev-products

# Enviar mensagem para a fila
aws --endpoint-url=http://localhost:4566 sqs send-message \
  --queue-url http://localhost:4566/000000000000/shopflow-dev-orders \
  --message-body '{"order_id": "123", "amount": 99.90}'

# Receber mensagem
aws --endpoint-url=http://localhost:4566 sqs receive-message \
  --queue-url http://localhost:4566/000000000000/shopflow-dev-orders
```

---

## Critérios de Aceite

- [ ] Terraform 1.10+ instalado e funcionando (`terraform version`)
- [ ] LocalStack rodando via Docker Compose e saudável
- [ ] Provider AWS configurado com endpoints LocalStack
- [ ] `.terraform.lock.hcl` gerado e commitado
- [ ] 2 S3 buckets criados (assets + logs) com versionamento
- [ ] 2 SQS queues criadas (orders + DLQ) com redrive policy
- [ ] 2 DynamoDB tables criadas (products + sessions) com GSI e TTL
- [ ] Todas as variáveis com `description` e `type` explícito
- [ ] Todos os outputs com `description`
- [ ] `locals` usado para `name_prefix` e `common_tags`
- [ ] `default_tags` no provider
- [ ] `terraform fmt` passa sem mudanças
- [ ] `terraform validate` passa sem erros
- [ ] `terraform plan` mostra recursos a criar
- [ ] `terraform apply` cria todos os recursos com sucesso
- [ ] `terraform output` mostra ARNs e nomes
- [ ] `terraform destroy` remove tudo sem erros

---

## Definição de Pronto (DoD)

- [ ] Código compila sem erros (`terraform validate`)
- [ ] Formatação aplicada (`terraform fmt -check`)
- [ ] Todos os recursos provisionados no LocalStack
- [ ] Outputs verificados manualmente
- [ ] README.md do projeto criado com instruções de setup
- [ ] Commit semântico: `feat(level-0): setup terraform project with LocalStack and basic resources`

---

## Checklist

- [ ] Terraform instalado e versão verificada
- [ ] Docker instalado e rodando
- [ ] LocalStack subindo via `docker compose up`
- [ ] Estrutura de arquivos criada (`main.tf`, `variables.tf`, etc.)
- [ ] Provider configurado para LocalStack
- [ ] S3 buckets criados e verificados
- [ ] SQS queues criadas com redrive policy
- [ ] DynamoDB tables criadas com GSI e TTL
- [ ] `terraform console` usado para explorar state
- [ ] `terraform graph` gerado (opcional)
- [ ] Recursos verificados via AWS CLI
- [ ] `terraform destroy` executado com sucesso

---

## Extensões Opcionais

- [ ] Instalar `tfenv` para gerenciar múltiplas versões do Terraform
- [ ] Explorar `terraform import` — criar recurso manualmente via AWS CLI e importar no Terraform
- [ ] Usar `terraform console` para testar expressões HCL complexas (loops, condicionais)
- [ ] Criar um script `Makefile` com targets: `init`, `plan`, `apply`, `destroy`, `fmt`, `validate`
- [ ] Explorar `terraform providers` e `terraform version -json`
- [ ] Adicionar um SNS Topic e subscription para a fila SQS (fan-out pattern)
