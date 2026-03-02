# Level 7 — Capstone: ShopFlow Platform

> **Objetivo:** Integrar todos os conhecimentos dos níveis anteriores em um deploy completo da plataforma ShopFlow multi-ambiente. Este capstone exige a criação de uma infraestrutura enterprise-grade com CI/CD, observabilidade, segurança, testes automatizados e documentação profissional — tudo simulado no LocalStack.

---

## Objetivo de Aprendizado

- Integrar todos os módulos em uma arquitetura coesa multi-ambiente
- Implementar pipeline CI/CD completo para infraestrutura
- Aplicar deploy progressivo (dev → staging → prod)
- Garantir compliance e segurança em toda a stack
- Criar documentação profissional de infraestrutura
- Demonstrar maestria em Terraform para cenários enterprise
- Resolver trade-offs reais de arquitetura de IaC

---

## Escopo Funcional

### ShopFlow — Arquitetura Final

```
                        ShopFlow E-Commerce Platform
                        ═══════════════════════════

                    ┌─────── ENVIRONMENTS ───────┐
                    │   dev │ staging │   prod    │
                    └───────┴─────────┴───────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                  │
    ┌─────────▼──────┐ ┌──────▼───────┐ ┌────────▼────────┐
    │   NETWORKING   │ │   COMPUTE    │ │   DATA LAYER    │
    │                │ │              │ │                  │
    │ • VPC          │ │ • Lambda     │ │ • DynamoDB       │
    │ • Subnets (6)  │ │   - orders   │ │   - products     │
    │ • Security GrP │ │   - catalog  │ │   - orders       │
    │ • Route Tables │ │   - events   │ │   - sessions     │
    │                │ │ • API GW     │ │   - events       │
    └────────────────┘ │   - REST API │ │                  │
                       └──────────────┘ └──────────────────┘
              │                 │                  │
    ┌─────────▼──────┐ ┌──────▼───────┐ ┌────────▼────────┐
    │   MESSAGING    │ │   STORAGE    │ │   SECURITY      │
    │                │ │              │ │                  │
    │ • SQS          │ │ • S3         │ │ • IAM Roles (5) │
    │   - orders     │ │   - assets   │ │ • KMS Keys (4)  │
    │   - events     │ │   - logs     │ │ • Secrets (3)   │
    │   - DLQ (2)    │ │   - backups  │ │ • SSM Params    │
    │ • SNS          │ │   - tf-state │ │ • Perm Boundary │
    │   - order-evts │ │              │ │                  │
    │   - alerts     │ │              │ │                  │
    └────────────────┘ └──────────────┘ └──────────────────┘
              │                                    │
              └───────────► STATE ◄────────────────┘
                         • S3 Backend
                         • Native Locking
                         • Per-domain isolation
                         • Cross-state references
```

### Inventário de Recursos (por ambiente)

| Domínio | Recurso | Quantidade | Módulo |
|---------|---------|-----------|--------|
| Networking | VPC | 1 | `modules/networking/vpc` |
| Networking | Subnets | 6 (3 public + 3 private) | `modules/networking/vpc` |
| Networking | Security Groups | 3 (lambda, api-gw, internal) | `modules/networking/vpc` |
| Storage | S3 Buckets | 3 (assets, logs, backups) | `modules/storage/s3-bucket` |
| Data | DynamoDB Tables | 4 (products, orders, sessions, events) | `modules/data/dynamodb-table` |
| Messaging | SQS Queues | 4 (orders, events, 2x DLQ) | `modules/messaging/sqs-queue` |
| Messaging | SNS Topics | 2 (order-events, alerts) | `modules/messaging/sns-topic` |
| Compute | Lambda Functions | 3 (orders, catalog, events) | `modules/compute/lambda-function` |
| Compute | API Gateway | 1 REST API | `modules/compute/api-gateway` |
| Security | IAM Roles | 5 | `modules/security/iam-role` |
| Security | KMS Keys | 4 | `modules/security/kms-key` |
| Security | Secrets | 3 | `modules/security/secret` |
| Security | SSM Parameters | 5+ | inline |
| **Total** | | **~40 recursos** | **12 módulos** |

---

## Escopo Técnico

### Estrutura Final do Repositório

```
shopflow-infrastructure/
│
├── README.md                          # Documentação principal
├── ARCHITECTURE.md                    # Decisões de arquitetura (ADR)
├── CHANGELOG.md                       # Histórico de mudanças
├── Makefile                           # Automação de comandos
├── docker-compose.yml                 # LocalStack + ferramentas
├── .pre-commit-config.yaml            # Git hooks
├── .github/
│   └── workflows/
│       ├── terraform-ci.yml           # Pipeline CI principal
│       ├── security-scan.yml          # Compliance scanning
│       └── terraform-docs.yml         # Auto-geração de docs
│
├── bootstrap/                         # Setup inicial do backend
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
│
├── modules/                           # Módulos reutilizáveis
│   ├── networking/
│   │   └── vpc/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       ├── README.md              # terraform-docs
│   │       └── tests/
│   │           └── vpc_test.tftest.hcl
│   ├── storage/
│   │   └── s3-bucket/
│   ├── data/
│   │   └── dynamodb-table/
│   ├── messaging/
│   │   ├── sqs-queue/
│   │   └── sns-topic/
│   ├── compute/
│   │   ├── lambda-function/
│   │   └── api-gateway/
│   └── security/
│       ├── iam-role/
│       ├── kms-key/
│       └── secret/
│
├── composed/                          # Módulos compostos
│   ├── api-backend/                   # API GW + Lambda + DynamoDB
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   └── event-pipeline/               # SNS → SQS → Lambda → DynamoDB
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── README.md
│
├── live/                              # Ambientes reais
│   ├── _global/                       # Recursos compartilhados
│   │   ├── iam/
│   │   │   ├── main.tf
│   │   │   ├── boundary.tf
│   │   │   ├── policies/
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── backend.tf
│   │   └── kms/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       └── backend.tf
│   │
│   ├── dev/
│   │   ├── terragrunt.hcl             # (Opcional) Config Terragrunt
│   │   ├── networking/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── backend.tf
│   │   │   └── terraform.tfvars
│   │   ├── storage/
│   │   ├── data/
│   │   ├── messaging/
│   │   ├── compute/
│   │   └── secrets/
│   │
│   ├── staging/
│   │   ├── networking/
│   │   ├── storage/
│   │   ├── data/
│   │   ├── messaging/
│   │   ├── compute/
│   │   └── secrets/
│   │
│   └── prod/
│       ├── networking/
│       ├── storage/
│       ├── data/
│       ├── messaging/
│       ├── compute/
│       └── secrets/
│
├── policy/                            # Políticas OPA
│   ├── security/
│   │   ├── encryption.rego
│   │   ├── iam.rego
│   │   └── secrets.rego
│   └── tagging/
│       └── required_tags.rego
│
├── scripts/                           # Automação
│   ├── apply-all.sh
│   ├── destroy-all.sh
│   ├── backup-state.sh
│   └── compliance-report.sh
│
└── tests/                             # Testes globais
    ├── integration/
    │   └── shopflow_test.go           # Terratest
    └── e2e/
        └── full_platform_test.go      # Teste end-to-end
```

---

## Tarefas

### Tarefa 1 — Consolidação de Módulos

Garanta que todos os 12 módulos estão finalizados, documentados e testados:

```bash
# Checklist de cada módulo
for module in modules/*/*; do
  echo "=== $module ==="

  # Tem main.tf?
  [ -f "$module/main.tf" ] && echo "  ✓ main.tf" || echo "  ✗ main.tf MISSING"

  # Tem variables.tf com validations?
  [ -f "$module/variables.tf" ] && echo "  ✓ variables.tf" || echo "  ✗ variables.tf MISSING"
  grep -q "validation {" "$module/variables.tf" 2>/dev/null && echo "  ✓ validations" || echo "  ✗ validations MISSING"

  # Tem outputs.tf?
  [ -f "$module/outputs.tf" ] && echo "  ✓ outputs.tf" || echo "  ✗ outputs.tf MISSING"

  # Tem README.md (terraform-docs)?
  [ -f "$module/README.md" ] && echo "  ✓ README.md" || echo "  ✗ README.md MISSING"

  # Tem testes?
  find "$module" -name "*.tftest.hcl" | grep -q . && echo "  ✓ tests" || echo "  ✗ tests MISSING"

  echo ""
done
```

### Tarefa 2 — Ambiente DEV Completo

Deploy completo do ambiente de desenvolvimento:

```bash
# 1. Iniciar LocalStack
docker compose up -d

# 2. Bootstrap do backend
cd bootstrap
terraform init && terraform apply -auto-approve

# 3. Deploy global (IAM + KMS)
cd ../live/_global/iam
terraform init && terraform apply -auto-approve

cd ../kms
terraform init && terraform apply -auto-approve

# 4. Deploy dev (em ordem de dependência)
for component in networking storage data messaging compute secrets; do
  echo ">>> Deploying dev/$component"
  cd ../../dev/$component
  terraform init && terraform apply -auto-approve
done

# 5. Validação completa
cd ../..
echo "=== Dev Environment Deployed ==="
```

### Tarefa 3 — Ambiente STAGING (Diferenças)

Crie o ambiente staging com configurações distintas:

```hcl
# live/staging/networking/terraform.tfvars
environment = "staging"
project     = "shopflow"

vpc_cidr           = "10.1.0.0/16"  # Diferente de dev (10.0.0.0/16)
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

public_subnet_cidrs  = ["10.1.1.0/24", "10.1.2.0/24", "10.1.3.0/24"]
private_subnet_cidrs = ["10.1.11.0/24", "10.1.12.0/24", "10.1.13.0/24"]
```

```hcl
# live/staging/data/terraform.tfvars
environment  = "staging"
project      = "shopflow"

# Staging tem mais capacidade que dev
dynamodb_billing_mode = "PROVISIONED"
dynamodb_read_capacity  = 10
dynamodb_write_capacity = 5

# PITR habilitado em staging
point_in_time_recovery = true
```

```hcl
# live/staging/compute/terraform.tfvars
environment = "staging"
project     = "shopflow"

# Staging tem mais memória
lambda_memory_size = 256  # dev = 128
lambda_timeout     = 30   # dev = 15

# API Gateway com throttling
api_throttle_rate_limit  = 100
api_throttle_burst_limit = 50
```

### Tarefa 4 — Ambiente PROD (Hardening)

Ambiente de produção com configurações de segurança máxima:

```hcl
# live/prod/storage/terraform.tfvars
environment = "prod"
project     = "shopflow"

# Produção: todas as proteções ativas
s3_versioning          = true
s3_lifecycle_enabled   = true
s3_lifecycle_days      = 90
s3_kms_encryption      = true
s3_access_logging      = true
s3_force_destroy       = false  # NUNCA em prod!

# Buckets adicionais para prod
create_backup_bucket = true
```

```hcl
# live/prod/data/terraform.tfvars
environment = "prod"
project     = "shopflow"

dynamodb_billing_mode    = "PROVISIONED"
dynamodb_read_capacity   = 50
dynamodb_write_capacity  = 25

point_in_time_recovery   = true
deletion_protection      = true  # Impede terraform destroy
```

```hcl
# live/prod/compute/terraform.tfvars
environment = "prod"
project     = "shopflow"

lambda_memory_size = 512
lambda_timeout     = 60

# Reservar concorrência em prod
lambda_reserved_concurrency = 100

api_throttle_rate_limit  = 1000
api_throttle_burst_limit = 500
```

### Tarefa 5 — Diferenças por Ambiente (Tabela Comparativa)

Documente as diferenças entre ambientes:

```markdown
# ENVIRONMENTS.md

| Configuração | Dev | Staging | Prod |
|-------------|-----|---------|------|
| **VPC CIDR** | 10.0.0.0/16 | 10.1.0.0/16 | 10.2.0.0/16 |
| **DynamoDB Mode** | PAY_PER_REQUEST | PROVISIONED (10/5) | PROVISIONED (50/25) |
| **DynamoDB PITR** | ❌ | ✅ | ✅ |
| **DynamoDB Delete Protect** | ❌ | ❌ | ✅ |
| **Lambda Memory** | 128 MB | 256 MB | 512 MB |
| **Lambda Timeout** | 15s | 30s | 60s |
| **Lambda Concurrency** | Unreserved | Unreserved | 100 reserved |
| **S3 Versioning** | ✅ | ✅ | ✅ |
| **S3 Lifecycle** | 30 days | 60 days | 90 days |
| **S3 Force Destroy** | ✅ | ❌ | ❌ |
| **KMS Encryption** | ✅ | ✅ | ✅ |
| **API Throttle Rate** | 50/s | 100/s | 1000/s |
| **Secrets Rotation** | ❌ | ❌ | ✅ (30 days) |
| **State Locking** | ✅ | ✅ | ✅ |
```

### Tarefa 6 — Pipeline CI/CD Completo

```yaml
# .github/workflows/terraform-ci.yml
name: Terraform CI/CD

on:
  push:
    branches: [main, develop]
    paths: ['**.tf', '**.tfvars', 'modules/**']
  pull_request:
    branches: [main]
    paths: ['**.tf', '**.tfvars', 'modules/**']

env:
  TF_VERSION: "1.10.0"
  LOCALSTACK_VERSION: "latest"

jobs:
  # ════════════════════════════════════════
  # Stage 1: Validação Estática
  # ════════════════════════════════════════
  static-analysis:
    name: Static Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive -diff

      - name: TFLint
        uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: latest
      - run: |
          tflint --init
          tflint --recursive --format compact

      - name: tfsec
        uses: aquasecurity/tfsec-action@v1.0.3
        with:
          soft_fail: false

      - name: Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: .
          framework: terraform
          soft_fail: false
          skip_check: CKV_AWS_18

      - name: Validate all components
        run: |
          for dir in $(find live modules composed -name "*.tf" -exec dirname {} \; | sort -u); do
            echo ">>> Validating: $dir"
            cd "$dir"
            terraform init -backend=false
            terraform validate
            cd "$GITHUB_WORKSPACE"
          done

  # ════════════════════════════════════════
  # Stage 2: Testes Unitários
  # ════════════════════════════════════════
  unit-tests:
    name: Unit Tests
    needs: static-analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Run terraform test (plan-only)
        run: |
          for module in modules/*/; do
            for mod in "$module"*/; do
              if find "$mod" -name "*.tftest.hcl" | grep -q .; then
                echo ">>> Testing: $mod"
                cd "$mod"
                terraform init -backend=false
                terraform test
                cd "$GITHUB_WORKSPACE"
              fi
            done
          done

  # ════════════════════════════════════════
  # Stage 3: Testes de Integração (LocalStack)
  # ════════════════════════════════════════
  integration-tests:
    name: Integration Tests
    needs: unit-tests
    runs-on: ubuntu-latest
    services:
      localstack:
        image: localstack/localstack:latest
        ports:
          - 4566:4566
        env:
          SERVICES: s3,sqs,sns,dynamodb,lambda,apigateway,iam,kms,ssm,secretsmanager,sts
          DEFAULT_REGION: us-east-1
          EAGER_SERVICE_LOADING: 1

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Wait for LocalStack
        run: |
          timeout 60 bash -c 'until curl -s http://localhost:4566/_localstack/health | grep -q running; do sleep 2; done'
          echo "LocalStack is ready"

      - name: Run terraform test (apply mode)
        run: |
          for module in modules/*/; do
            for mod in "$module"*/; do
              if find "$mod" -name "*.tftest.hcl" | grep -q .; then
                echo ">>> Integration test: $mod"
                cd "$mod"
                terraform init
                terraform test -verbose
                cd "$GITHUB_WORKSPACE"
              fi
            done
          done

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'

      - name: Run Terratest
        run: |
          cd tests/integration
          go test -v -timeout 30m ./...
        env:
          AWS_ACCESS_KEY_ID: test
          AWS_SECRET_ACCESS_KEY: test
          AWS_DEFAULT_REGION: us-east-1
          LOCALSTACK_ENDPOINT: http://localhost:4566

  # ════════════════════════════════════════
  # Stage 4: Policy Tests
  # ════════════════════════════════════════
  policy-tests:
    name: Policy Tests
    needs: unit-tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Install Conftest
        run: |
          LATEST=$(curl -s https://api.github.com/repos/open-policy-agent/conftest/releases/latest | jq -r .tag_name | sed 's/v//')
          curl -sL "https://github.com/open-policy-agent/conftest/releases/download/v${LATEST}/conftest_${LATEST}_Linux_x86_64.tar.gz" | tar xz
          sudo mv conftest /usr/local/bin/

      - name: Run OPA policies
        run: |
          for dir in live/dev/*/; do
            if [ -f "$dir/main.tf" ]; then
              echo ">>> Policy check: $dir"
              cd "$dir"
              terraform init -backend=false
              terraform plan -out=plan.tfplan -input=false 2>/dev/null || true
              terraform show -json plan.tfplan > plan.json 2>/dev/null || continue
              conftest test plan.json --policy "$GITHUB_WORKSPACE/policy/" || true
              cd "$GITHUB_WORKSPACE"
            fi
          done

  # ════════════════════════════════════════
  # Stage 5: Plan (PR only)
  # ════════════════════════════════════════
  plan:
    name: Terraform Plan
    needs: [integration-tests, policy-tests]
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    services:
      localstack:
        image: localstack/localstack:latest
        ports:
          - 4566:4566
        env:
          SERVICES: s3,sqs,sns,dynamodb,lambda,apigateway,iam,kms,ssm,secretsmanager,sts
    strategy:
      matrix:
        environment: [dev, staging]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Bootstrap
        run: |
          cd bootstrap
          terraform init && terraform apply -auto-approve

      - name: Plan ${{ matrix.environment }}
        run: |
          for dir in live/${{ matrix.environment }}/*/; do
            if [ -f "$dir/main.tf" ]; then
              echo ">>> Plan: $dir"
              cd "$dir"
              terraform init -input=false
              terraform plan -input=false -no-color | tee plan.txt
              cd "$GITHUB_WORKSPACE"
            fi
          done

  # ════════════════════════════════════════
  # Stage 6: Apply (main branch only)
  # ════════════════════════════════════════
  apply-dev:
    name: Apply Dev
    needs: [integration-tests, policy-tests]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: dev
    services:
      localstack:
        image: localstack/localstack:latest
        ports:
          - 4566:4566
        env:
          SERVICES: s3,sqs,sns,dynamodb,lambda,apigateway,iam,kms,ssm,secretsmanager,sts
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Bootstrap & Apply Dev
        run: |
          cd bootstrap && terraform init && terraform apply -auto-approve && cd ..
          cd live/_global/iam && terraform init && terraform apply -auto-approve && cd ../../..
          cd live/_global/kms && terraform init && terraform apply -auto-approve && cd ../../..
          for comp in networking storage data messaging compute secrets; do
            cd "live/dev/$comp"
            terraform init -input=false
            terraform apply -auto-approve -input=false
            cd "$GITHUB_WORKSPACE"
          done

  apply-staging:
    name: Apply Staging
    needs: apply-dev
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    services:
      localstack:
        image: localstack/localstack:latest
        ports:
          - 4566:4566
        env:
          SERVICES: s3,sqs,sns,dynamodb,lambda,apigateway,iam,kms,ssm,secretsmanager,sts
    steps:
      - uses: actions/checkout@v4

      - name: Setup & Apply Staging
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Apply
        run: |
          cd bootstrap && terraform init && terraform apply -auto-approve && cd ..
          cd live/_global/iam && terraform init && terraform apply -auto-approve && cd ../../..
          cd live/_global/kms && terraform init && terraform apply -auto-approve && cd ../../..
          for comp in networking storage data messaging compute secrets; do
            cd "live/staging/$comp"
            terraform init -input=false
            terraform apply -auto-approve -input=false
            cd "$GITHUB_WORKSPACE"
          done
```

### Tarefa 7 — Makefile de Automação

```makefile
# Makefile
.PHONY: help init plan apply destroy test lint docs

SHELL := /bin/bash
ENV ?= dev
COMPONENT ?= all
TF_FLAGS ?= -input=false

# ════════════════════════════════════════
# Help
# ════════════════════════════════════════
help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

# ════════════════════════════════════════
# LocalStack
# ════════════════════════════════════════
up: ## Start LocalStack
	docker compose up -d
	@echo "Waiting for LocalStack..."
	@timeout 60 bash -c 'until curl -s http://localhost:4566/_localstack/health | grep -q running; do sleep 2; done'
	@echo "LocalStack is ready!"

down: ## Stop LocalStack
	docker compose down -v

status: ## Check LocalStack status
	@curl -s http://localhost:4566/_localstack/health | jq .

# ════════════════════════════════════════
# Bootstrap
# ════════════════════════════════════════
bootstrap: ## Bootstrap state backend
	cd bootstrap && terraform init && terraform apply -auto-approve

# ════════════════════════════════════════
# Terraform Operations
# ════════════════════════════════════════
init: ## Initialize component (ENV=dev COMPONENT=networking)
ifeq ($(COMPONENT),all)
	@for dir in live/$(ENV)/*/; do \
		echo ">>> Init: $$dir"; \
		cd "$$dir" && terraform init $(TF_FLAGS) && cd "$(CURDIR)"; \
	done
else
	cd live/$(ENV)/$(COMPONENT) && terraform init $(TF_FLAGS)
endif

plan: ## Plan component (ENV=dev COMPONENT=networking)
ifeq ($(COMPONENT),all)
	@for dir in live/$(ENV)/*/; do \
		echo ">>> Plan: $$dir"; \
		cd "$$dir" && terraform plan $(TF_FLAGS) && cd "$(CURDIR)"; \
	done
else
	cd live/$(ENV)/$(COMPONENT) && terraform plan $(TF_FLAGS)
endif

apply: ## Apply component (ENV=dev COMPONENT=networking)
ifeq ($(COMPONENT),all)
	@echo "=== Applying ALL components for $(ENV) ==="
	@for comp in networking storage data messaging compute secrets; do \
		echo ">>> Apply: live/$(ENV)/$$comp"; \
		cd "live/$(ENV)/$$comp" && terraform apply -auto-approve $(TF_FLAGS) && cd "$(CURDIR)"; \
	done
else
	cd live/$(ENV)/$(COMPONENT) && terraform apply -auto-approve $(TF_FLAGS)
endif

destroy: ## Destroy component (ENV=dev COMPONENT=networking)
ifeq ($(COMPONENT),all)
	@echo "=== Destroying ALL components for $(ENV) (reverse order) ==="
	@for comp in secrets compute messaging data storage networking; do \
		echo ">>> Destroy: live/$(ENV)/$$comp"; \
		cd "live/$(ENV)/$$comp" && terraform destroy -auto-approve $(TF_FLAGS) && cd "$(CURDIR)"; \
	done
else
	cd live/$(ENV)/$(COMPONENT) && terraform destroy -auto-approve $(TF_FLAGS)
endif

# ════════════════════════════════════════
# Global Resources
# ════════════════════════════════════════
apply-global: ## Apply global resources (IAM + KMS)
	cd live/_global/iam && terraform init && terraform apply -auto-approve $(TF_FLAGS)
	cd live/_global/kms && terraform init && terraform apply -auto-approve $(TF_FLAGS)

# ════════════════════════════════════════
# Full Deploy
# ════════════════════════════════════════
deploy-all: bootstrap apply-global ## Deploy everything for ENV
	$(MAKE) apply ENV=$(ENV) COMPONENT=all

deploy-dev: ## Deploy complete dev environment
	$(MAKE) deploy-all ENV=dev

deploy-staging: ## Deploy complete staging environment
	$(MAKE) deploy-all ENV=staging

deploy-prod: ## Deploy complete prod environment
	$(MAKE) deploy-all ENV=prod

# ════════════════════════════════════════
# Testing
# ════════════════════════════════════════
test-unit: ## Run unit tests (terraform test, plan-only)
	@for mod in modules/*/*/; do \
		if find "$$mod" -name "*.tftest.hcl" | grep -q .; then \
			echo ">>> Unit test: $$mod"; \
			cd "$$mod" && terraform init -backend=false && terraform test && cd "$(CURDIR)"; \
		fi; \
	done

test-integration: ## Run integration tests (terraform test against LocalStack)
	@for mod in modules/*/*/; do \
		if find "$$mod" -name "*.tftest.hcl" | grep -q .; then \
			echo ">>> Integration test: $$mod"; \
			cd "$$mod" && terraform init && terraform test -verbose && cd "$(CURDIR)"; \
		fi; \
	done

test-terratest: ## Run Terratest tests
	cd tests/integration && go test -v -timeout 30m ./...

test-policy: ## Run OPA policy tests
	@for dir in live/dev/*/; do \
		if [ -f "$$dir/main.tf" ]; then \
			echo ">>> Policy: $$dir"; \
			cd "$$dir" && \
			terraform init -backend=false && \
			terraform plan -out=plan.tfplan $(TF_FLAGS) 2>/dev/null && \
			terraform show -json plan.tfplan > plan.json && \
			conftest test plan.json --policy "$(CURDIR)/policy/" && \
			cd "$(CURDIR)"; \
		fi; \
	done

test-all: test-unit test-integration test-policy ## Run all tests

# ════════════════════════════════════════
# Quality
# ════════════════════════════════════════
fmt: ## Format all Terraform files
	terraform fmt -recursive

lint: ## Run TFLint
	tflint --recursive --format compact

scan-tfsec: ## Run tfsec security scan
	tfsec . --format lovely

scan-checkov: ## Run Checkov compliance scan
	checkov -d . --framework terraform --compact

scan: scan-tfsec scan-checkov ## Run all security scans

validate: ## Validate all components
	@for dir in $$(find live modules composed -name "*.tf" -exec dirname {} \; | sort -u); do \
		echo ">>> Validate: $$dir"; \
		cd "$$dir" && terraform init -backend=false && terraform validate && cd "$(CURDIR)"; \
	done

# ════════════════════════════════════════
# Documentation
# ════════════════════════════════════════
docs: ## Generate terraform-docs for all modules
	@for mod in modules/*/*/ composed/*/; do \
		if [ -f "$$mod/main.tf" ]; then \
			echo ">>> Docs: $$mod"; \
			terraform-docs markdown table "$$mod" > "$$mod/README.md"; \
		fi; \
	done

# ════════════════════════════════════════
# State Operations
# ════════════════════════════════════════
state-backup: ## Backup state for ENV/COMPONENT
	@mkdir -p backups
	cd live/$(ENV)/$(COMPONENT) && \
		terraform state pull > "$(CURDIR)/backups/$(ENV)-$(COMPONENT)-$$(date +%Y%m%d-%H%M%S).tfstate"
	@echo "State backed up to backups/"

state-list: ## List resources in state for ENV/COMPONENT
	cd live/$(ENV)/$(COMPONENT) && terraform state list
```

### Tarefa 8 — Teste End-to-End

```go
// tests/e2e/full_platform_test.go
package e2e

import (
	"encoding/json"
	"fmt"
	"os"
	"testing"
	"time"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/credentials"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/dynamodb"
	"github.com/aws/aws-sdk-go/service/s3"
	"github.com/aws/aws-sdk-go/service/sqs"
	"github.com/aws/aws-sdk-go/service/sns"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

const localstackEndpoint = "http://localhost:4566"

func getAWSSession(t *testing.T) *session.Session {
	sess, err := session.NewSession(&aws.Config{
		Region:      aws.String("us-east-1"),
		Endpoint:    aws.String(localstackEndpoint),
		Credentials: credentials.NewStaticCredentials("test", "test", ""),
		S3ForcePathStyle: aws.Bool(true),
	})
	require.NoError(t, err)
	return sess
}

func TestFullShopFlowPlatform(t *testing.T) {
	t.Parallel()

	sess := getAWSSession(t)

	// ─── Verificar S3 Buckets ───
	t.Run("S3_Buckets_Exist", func(t *testing.T) {
		s3Client := s3.New(sess)

		expectedBuckets := []string{
			"shopflow-dev-assets",
			"shopflow-dev-logs",
			"shopflow-dev-backups",
		}

		result, err := s3Client.ListBuckets(&s3.ListBucketsInput{})
		require.NoError(t, err)

		bucketNames := make([]string, len(result.Buckets))
		for i, b := range result.Buckets {
			bucketNames[i] = *b.Name
		}

		for _, expected := range expectedBuckets {
			assert.Contains(t, bucketNames, expected,
				"Expected bucket %s to exist", expected)
		}
	})

	// ─── Verificar DynamoDB Tables ───
	t.Run("DynamoDB_Tables_Exist", func(t *testing.T) {
		ddbClient := dynamodb.New(sess)

		expectedTables := []string{
			"shopflow-dev-products",
			"shopflow-dev-orders",
			"shopflow-dev-sessions",
			"shopflow-dev-events",
		}

		result, err := ddbClient.ListTables(&dynamodb.ListTablesInput{})
		require.NoError(t, err)

		tableNames := make([]string, len(result.TableNames))
		for i, name := range result.TableNames {
			tableNames[i] = *name
		}

		for _, expected := range expectedTables {
			assert.Contains(t, tableNames, expected,
				"Expected table %s to exist", expected)
		}
	})

	// ─── Verificar SQS Queues ───
	t.Run("SQS_Queues_Exist", func(t *testing.T) {
		sqsClient := sqs.New(sess)

		expectedQueues := []string{
			"shopflow-dev-orders",
			"shopflow-dev-orders-dlq",
			"shopflow-dev-events",
			"shopflow-dev-events-dlq",
		}

		result, err := sqsClient.ListQueues(&sqs.ListQueuesInput{})
		require.NoError(t, err)

		for _, expected := range expectedQueues {
			found := false
			for _, url := range result.QueueUrls {
				if url != nil && contains(*url, expected) {
					found = true
					break
				}
			}
			assert.True(t, found, "Expected queue %s to exist", expected)
		}
	})

	// ─── Verificar SNS Topics ───
	t.Run("SNS_Topics_Exist", func(t *testing.T) {
		snsClient := sns.New(sess)

		result, err := snsClient.ListTopics(&sns.ListTopicsInput{})
		require.NoError(t, err)

		expectedTopics := []string{
			"shopflow-dev-order-events",
			"shopflow-dev-alerts",
		}

		topicArns := make([]string, len(result.Topics))
		for i, topic := range result.Topics {
			topicArns[i] = *topic.TopicArn
		}

		for _, expected := range expectedTopics {
			found := false
			for _, arn := range topicArns {
				if contains(arn, expected) {
					found = true
					break
				}
			}
			assert.True(t, found, "Expected topic %s to exist", expected)
		}
	})

	// ─── Teste de Fluxo: Publicar Pedido ───
	t.Run("Order_Flow_Integration", func(t *testing.T) {
		sqsClient := sqs.New(sess)
		ddbClient := dynamodb.New(sess)

		// 1. Inserir produto no DynamoDB
		_, err := ddbClient.PutItem(&dynamodb.PutItemInput{
			TableName: aws.String("shopflow-dev-products"),
			Item: map[string]*dynamodb.AttributeValue{
				"product_id": {S: aws.String("PROD-001")},
				"name":       {S: aws.String("Test Product")},
				"price":      {N: aws.String("29.99")},
				"category":   {S: aws.String("electronics")},
			},
		})
		require.NoError(t, err)

		// 2. Verificar produto existe
		getResult, err := ddbClient.GetItem(&dynamodb.GetItemInput{
			TableName: aws.String("shopflow-dev-products"),
			Key: map[string]*dynamodb.AttributeValue{
				"product_id": {S: aws.String("PROD-001")},
			},
		})
		require.NoError(t, err)
		assert.NotNil(t, getResult.Item)

		// 3. Enviar pedido para SQS
		queueURL := fmt.Sprintf("%s/000000000000/shopflow-dev-orders", localstackEndpoint)

		order := map[string]interface{}{
			"order_id":   "ORD-001",
			"product_id": "PROD-001",
			"quantity":   2,
			"timestamp":  time.Now().UTC().Format(time.RFC3339),
		}
		orderJSON, _ := json.Marshal(order)

		_, err = sqsClient.SendMessage(&sqs.SendMessageInput{
			QueueUrl:    aws.String(queueURL),
			MessageBody: aws.String(string(orderJSON)),
		})
		require.NoError(t, err)

		// 4. Receber mensagem da fila
		receiveResult, err := sqsClient.ReceiveMessage(&sqs.ReceiveMessageInput{
			QueueUrl:            aws.String(queueURL),
			MaxNumberOfMessages: aws.Int64(1),
			WaitTimeSeconds:     aws.Int64(5),
		})
		require.NoError(t, err)
		require.Len(t, receiveResult.Messages, 1)

		var receivedOrder map[string]interface{}
		err = json.Unmarshal([]byte(*receiveResult.Messages[0].Body), &receivedOrder)
		require.NoError(t, err)
		assert.Equal(t, "ORD-001", receivedOrder["order_id"])
	})
}

func contains(s, substr string) bool {
	return len(s) >= len(substr) && (s == substr || len(s) > 0 && containsSubstring(s, substr))
}

func containsSubstring(s, substr string) bool {
	for i := 0; i <= len(s)-len(substr); i++ {
		if s[i:i+len(substr)] == substr {
			return true
		}
	}
	return false
}
```

### Tarefa 9 — Documentação Profissional

Crie o `ARCHITECTURE.md` com Architecture Decision Records:

```markdown
# ShopFlow Infrastructure — Architecture Decisions

## ADR-001: LocalStack para Simulação AWS

**Status:** Aceito
**Data:** 2024-XX-XX

### Contexto
Precisamos de um ambiente de desenvolvimento que permita simular serviços AWS
sem custos e com execução local rápida.

### Decisão
Usar LocalStack como simulador de serviços AWS para todos os ambientes
de desenvolvimento e testes.

### Consequências
- ✅ Zero custo de cloud em dev/test
- ✅ Execução rápida (sem latência de rede)
- ✅ Portável (Docker em qualquer máquina)
- ⚠️ Nem todos os serviços AWS são suportados
- ⚠️ Comportamento pode diferir levemente do AWS real

---

## ADR-002: Isolamento de State por Domínio

**Status:** Aceito

### Contexto
Um state monolítico torna applies lentos e aumenta o blast radius.

### Decisão
Isolar state por domínio (networking, storage, data, messaging, compute, secrets)
e por ambiente (dev, staging, prod).

### Consequências
- ✅ Blast radius reduzido
- ✅ Apply paralelo por domínio
- ✅ Permissões granulares por equipe
- ⚠️ Requer coordenação entre states
- ⚠️ Remote state data sources adicionam coupling

---

## ADR-003: Native S3 Lock vs DynamoDB

**Status:** Aceito

### Decisão
Usar native S3 locking (Terraform 1.10+, `use_lockfile = true`)
em vez de DynamoDB table para state locking.

### Consequências
- ✅ Menos infraestrutura para gerenciar
- ✅ Menos custo (sem DynamoDB table)
- ✅ Configuração mais simples
- ⚠️ Requer Terraform 1.10+

---

## ADR-004: Módulos Atômicos + Compostos

**Status:** Aceito

### Decisão
Criar módulos atômicos (um recurso principal) e módulos compostos
(combinam vários atômicos para use cases comuns).

### Consequências
- ✅ Máxima reutilização
- ✅ Composição flexível
- ✅ Testabilidade individual
- ⚠️ Mais arquivos para manter
- ⚠️ Interface entre módulos precisa ser bem definida

---

## ADR-005: KMS Customer-Managed Keys

**Status:** Aceito

### Decisão
Usar KMS CMK para encryption de todos os serviços (S3, DynamoDB, SQS, SNS,
Secrets Manager), separado por domínio.

### Consequências
- ✅ Controle total das chaves
- ✅ Auditoria via CloudTrail
- ✅ Rotação automática
- ⚠️ Custo adicional por key/uso em prod
- ⚠️ Complexidade adicional em IAM policies
```

---

## Critérios de Aceite

- [ ] Todos os 12 módulos finalizados, documentados e testados
- [ ] Ambientes dev, staging e prod configurados com diferenças documentadas
- [ ] Bootstrap do backend executado com sucesso
- [ ] Deploy completo do ambiente dev funcional
- [ ] Deploy completo do ambiente staging funcional
- [ ] Pipeline CI/CD com 6 stages configurado
- [ ] Makefile com todos os targets funcionais
- [ ] Testes end-to-end passando (Terratest)
- [ ] Compliance scanning com zero findings Critical/High
- [ ] Documentação ARCHITECTURE.md com pelo menos 5 ADRs
- [ ] ENVIRONMENTS.md com tabela comparativa
- [ ] CHANGELOG.md com histórico
- [ ] Todos os recursos com tags obrigatórias (Project, Environment, ManagedBy)
- [ ] ~40 recursos por ambiente provisionados no LocalStack
- [ ] Remote state data sources funcionando entre domínios

---

## Definição de Pronto (DoD)

- [ ] Repositório com estrutura completa conforme template
- [ ] Todos os módulos com terraform-docs gerado
- [ ] Pipeline CI/CD funcional em GitHub Actions
- [ ] Testes em 3 níveis (unit, integration, policy)
- [ ] Documentação profissional (README, ARCHITECTURE, ENVIRONMENTS, CHANGELOG)
- [ ] Zero warnings em `terraform validate` e `tflint`
- [ ] Zero findings Critical/High em security scanning
- [ ] Commit semântico: `feat(level-7): complete ShopFlow platform capstone`
- [ ] Diagrama de arquitetura atualizado

---

## Checklist

- [ ] Módulos consolidados (12/12)
- [ ] Módulos compostos finalizados (2/2)
- [ ] Bootstrap funcional
- [ ] Deploy dev completo
- [ ] Deploy staging completo
- [ ] Deploy prod configurado
- [ ] Pipeline CI/CD funcional
- [ ] Makefile completo
- [ ] Terratest e2e passando
- [ ] Compliance scan passando
- [ ] ARCHITECTURE.md escrito
- [ ] ENVIRONMENTS.md escrito
- [ ] CHANGELOG.md criado
- [ ] terraform-docs gerado
- [ ] Pre-commit hooks configurados

---

## Extensões Opcionais

- [ ] Configurar Terragrunt para eliminar duplicação entre ambientes
- [ ] Implementar Infracost para estimativa de custos por PR
- [ ] Adicionar diagrama gerado automaticamente via `terraform graph`
- [ ] Implementar blue-green deployment pattern com Terraform
- [ ] Criar módulo de observabilidade (CloudWatch dashboards, alarms, SNS)
- [ ] Adicionar drift detection agendado via GitHub Actions cron
- [ ] Implementar import de recursos existentes (simulando adoção de IaC)
- [ ] Criar pull request template com checklist de segurança
