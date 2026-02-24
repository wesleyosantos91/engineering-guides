````markdown
# Testes — Terraform AWS

> **Objetivo:** Guia completo para testes de infraestrutura com Terraform, incluindo testes unitários nativos,
> testes de integração com Terratest, policy testing e compliance.
> Baseado nas práticas da HashiCorp, Gruntwork e empresas que operam infraestrutura em escala.

---

## Sumário

- [Pirâmide de Testes para IaC](#pirâmide-de-testes-para-iac)
- [Terraform Test — Testes Unitários Nativos](#terraform-test--testes-unitários-nativos)
  - [Estrutura de Arquivos](#estrutura-de-arquivos)
  - [Plan-only Tests (Unit)](#plan-only-tests-unit)
  - [Apply Tests (Integration)](#apply-tests-integration)
  - [Mocking de Providers](#mocking-de-providers)
  - [Variables Override](#variables-override)
  - [Executando Testes](#executando-testes)
- [Terratest — Testes de Integração em Go](#terratest--testes-de-integração-em-go)
  - [Setup do Projeto](#setup-do-projeto)
  - [Teste Básico de Módulo](#teste-básico-de-módulo)
  - [Teste com Validação AWS](#teste-com-validação-aws)
  - [Testes em Paralelo](#testes-em-paralelo)
  - [Retry e Resiliência](#retry-e-resiliência)
  - [Cleanup e Nuke](#cleanup-e-nuke)
- [Policy Testing — OPA / Conftest](#policy-testing--opa--conftest)
- [Security Scanning — tfsec / Checkov](#security-scanning--tfsec--checkov)
- [Validação Estática — terraform validate + TFLint](#validação-estática--terraform-validate--tflint)
- [Contract Tests — Outputs e Interfaces](#contract-tests--outputs-e-interfaces)
- [Testes de Custo — Infracost](#testes-de-custo--infracost)
- [Estratégia de Testes em CI](#estratégia-de-testes-em-ci)
- [Dicas e Patterns](#dicas-e-patterns)

---

## Pirâmide de Testes para IaC

```
         ▲
        / \        Testes E2E / Smoke
       /   \       (Ambiente real, pós-deploy)
      /     \
     /───────\     Testes de Integração
    /         \    (Terratest — apply + validate + destroy)
   /           \
  /─────────────\  Testes Unitários
 /               \ (terraform test — plan-only, mocks)
/─────────────────\ Análise Estática
                    (fmt, validate, tflint, tfsec, checkov, conftest)
```

| Nível | Ferramenta | Tempo | Custo AWS | Frequência |
|-------|-----------|-------|-----------|------------|
| Análise Estática | fmt, validate, tflint, tfsec | Segundos | $0 | Todo commit |
| Unitário | `terraform test` (plan-only) | Segundos | $0 | Todo commit |
| Integração | Terratest / `terraform test` (apply) | Minutos | $ | PR / merge |
| E2E | Custom scripts | Minutos-Horas | $$ | Nightly / release |

---

## Terraform Test — Testes Unitários Nativos

> Disponível a partir do **Terraform 1.6+** (mocks de providers a partir do 1.7+). Recomendado como primeira escolha.

### Estrutura de Arquivos

```
modules/networking/vpc/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── README.md
└── tests/
    ├── unit/
    │   ├── basic_test.tftest.hcl
    │   ├── validation_test.tftest.hcl
    │   └── custom_config_test.tftest.hcl
    └── integration/
        └── full_deploy_test.tftest.hcl
```

### Plan-only Tests (Unit)

Testes que rodam apenas `terraform plan` — **não criam recursos reais**.

```hcl
# tests/unit/basic_test.tftest.hcl

# Variáveis para o teste
variables {
  project     = "test-project"
  environment = "dev"
  vpc_cidr    = "10.0.0.0/16"
  az_count    = 2
}

# Teste: VPC é criada com CIDR correto
run "vpc_cidr_is_correct" {
  command = plan    # ← Apenas plan, sem apply

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block should be 10.0.0.0/16"
  }
}

# Teste: Número correto de subnets privadas
run "private_subnets_count" {
  command = plan

  assert {
    condition     = length(aws_subnet.private) == 2
    error_message = "Should create 2 private subnets"
  }
}

# Teste: Tags obrigatórias estão presentes
run "mandatory_tags_present" {
  command = plan

  assert {
    condition     = aws_vpc.main.tags["Environment"] == "dev"
    error_message = "VPC must have Environment tag"
  }

  assert {
    condition     = aws_vpc.main.tags["ManagedBy"] == "terraform"
    error_message = "VPC must have ManagedBy tag"
  }
}

# Teste: DNS habilitado
run "dns_enabled" {
  command = plan

  assert {
    condition     = aws_vpc.main.enable_dns_hostnames == true
    error_message = "DNS hostnames must be enabled"
  }

  assert {
    condition     = aws_vpc.main.enable_dns_support == true
    error_message = "DNS support must be enabled"
  }
}
```

### Validação de Variáveis

```hcl
# tests/unit/validation_test.tftest.hcl

# Teste: Validação de environment rejeita valores inválidos
run "invalid_environment_rejected" {
  command = plan

  variables {
    environment = "banana"
  }

  expect_failures = [
    var.environment,    # Espera que a validação da variável falhe
  ]
}

# Teste: Validação de CIDR rejeita valores inválidos
run "invalid_cidr_rejected" {
  command = plan

  variables {
    vpc_cidr = "not-a-cidr"
  }

  expect_failures = [
    var.vpc_cidr,
  ]
}

# Teste: Valores default funcionam
run "default_values_work" {
  command = plan

  variables {
    project     = "test"
    environment = "dev"
  }

  # Se as variáveis com default funcionam, o plan deve passar
  assert {
    condition     = aws_vpc.main.cidr_block != ""
    error_message = "VPC should be created with default CIDR"
  }
}
```

### Apply Tests (Integration)

Testes que fazem `apply` real — **criam e destroem recursos na AWS**.

```hcl
# tests/integration/full_deploy_test.tftest.hcl

variables {
  project     = "tftest"
  environment = "test"
  vpc_cidr    = "10.99.0.0/16"
  az_count    = 2
}

# Step 1: Cria a VPC
run "create_vpc" {
  command = apply    # ← Cria recursos reais

  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC ID should not be empty"
  }

  assert {
    condition     = length(output.private_subnet_ids) == 2
    error_message = "Should output 2 private subnet IDs"
  }

  assert {
    condition     = length(output.public_subnet_ids) == 2
    error_message = "Should output 2 public subnet IDs"
  }
}

# Step 2: Verifica que outputs são consistentes
run "verify_outputs" {
  command = plan    # Não precisa re-apply

  assert {
    condition     = output.vpc_cidr == "10.99.0.0/16"
    error_message = "VPC CIDR output should match input"
  }
}

# Terraform automaticamente faz destroy ao final dos testes
```

### Mocking de Providers

> Terraform 1.7+ permite mock de providers para testes 100% offline.

```hcl
# tests/unit/with_mocks_test.tftest.hcl

# Mock do provider AWS — sem chamadas reais
mock_provider "aws" {
  mock_data "aws_availability_zones" {
    defaults = {
      names = ["us-east-1a", "us-east-1b", "us-east-1c"]
    }
  }

  mock_data "aws_caller_identity" {
    defaults = {
      account_id = "123456789012"
    }
  }

  mock_data "aws_region" {
    defaults = {
      name = "us-east-1"
    }
  }
}

variables {
  project     = "mock-test"
  environment = "dev"
  vpc_cidr    = "10.0.0.0/16"
}

run "vpc_with_mocked_provider" {
  command = plan

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR should match"
  }
}
```

### Override Files

```hcl
# tests/unit/setup/providers_override.tf
# Override para remover backend remoto durante testes

terraform {
  backend "local" {}
}
```

### Variables Override

```hcl
# tests/fixtures/dev.tfvars
project     = "test-project"
environment = "dev"
vpc_cidr    = "10.0.0.0/16"
az_count    = 2
```

```hcl
# Referenciando no teste
run "with_tfvars" {
  command = plan

  variables {
    # Pode sobrescrever individualmente
    environment = "staging"
  }
}
```

### Executando Testes

```bash
# Rodar todos os testes
terraform test

# Rodar testes específicos
terraform test -filter=tests/unit/basic_test.tftest.hcl

# Verbose output
terraform test -verbose

# Com variáveis
terraform test -var="environment=dev"

# Apenas testes de um diretório
cd modules/networking/vpc && terraform test
```

---

## Terratest — Testes de Integração em Go

> Use Terratest quando precisa de validações mais complexas, chamadas à API AWS,
> ou lógica de teste que vai além do que `terraform test` oferece.

### Setup do Projeto

```
modules/networking/vpc/
├── main.tf
├── variables.tf
├── outputs.tf
└── test/
    ├── go.mod
    ├── go.sum
    └── vpc_test.go
```

```bash
cd modules/networking/vpc/test
go mod init github.com/myorg/terraform-modules/vpc/test
go get github.com/gruntwork-io/terratest/modules/terraform
go get github.com/gruntwork-io/terratest/modules/aws
go get github.com/stretchr/testify/assert
```

### Teste Básico de Módulo

```go
// test/vpc_test.go
package test

import (
	"testing"

	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestVpcBasic(t *testing.T) {
	t.Parallel()

	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../",      // Diretório do módulo

		Vars: map[string]interface{}{
			"project":     "terratest",
			"environment": "test",
			"vpc_cidr":    "10.99.0.0/16",
			"az_count":    2,
		},

		// Variáveis de ambiente
		EnvVars: map[string]string{
			"AWS_DEFAULT_REGION": "us-east-1",
		},
	})

	// Garante cleanup — destroy ao final
	defer terraform.Destroy(t, terraformOptions)

	// Init + Apply
	terraform.InitAndApply(t, terraformOptions)

	// Verifica outputs
	vpcId := terraform.Output(t, terraformOptions, "vpc_id")
	assert.NotEmpty(t, vpcId)

	vpcCidr := terraform.Output(t, terraformOptions, "vpc_cidr")
	assert.Equal(t, "10.99.0.0/16", vpcCidr)

	privateSubnets := terraform.OutputList(t, terraformOptions, "private_subnet_ids")
	assert.Len(t, privateSubnets, 2)
}
```

### Teste com Validação AWS

```go
package test

import (
	"testing"

	"github.com/gruntwork-io/terratest/modules/aws"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

func TestVpcWithAwsValidation(t *testing.T) {
	t.Parallel()

	awsRegion := "us-east-1"

	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../",
		Vars: map[string]interface{}{
			"project":     "terratest",
			"environment": "test",
			"vpc_cidr":    "10.99.0.0/16",
			"az_count":    2,
			"aws_region":  awsRegion,
		},
	})

	defer terraform.Destroy(t, terraformOptions)
	terraform.InitAndApply(t, terraformOptions)

	// Busca VPC na AWS e valida propriedades
	vpcId := terraform.Output(t, terraformOptions, "vpc_id")
	vpc := aws.GetVpcById(t, vpcId, awsRegion)

	assert.Equal(t, "10.99.0.0/16", vpc.CidrBlock)
	require.NotNil(t, vpc.Tags["Environment"])
	assert.Equal(t, "test", vpc.Tags["Environment"])
	assert.Equal(t, "terraform", vpc.Tags["ManagedBy"])

	// Verifica subnets
	subnets := aws.GetSubnetsForVpc(t, vpcId, awsRegion)
	assert.Equal(t, 4, len(subnets))  // 2 private + 2 public

	// Verifica que tem pelo menos 2 AZs
	azs := map[string]bool{}
	for _, subnet := range subnets {
		azs[subnet.AvailabilityZone] = true
	}
	assert.GreaterOrEqual(t, len(azs), 2)
}
```

### Testes em Paralelo

```go
func TestVpcMultipleConfigs(t *testing.T) {
	t.Parallel()

	testCases := []struct {
		name     string
		cidr     string
		azCount  int
		expected int
	}{
		{"TwoAZ", "10.1.0.0/16", 2, 4},
		{"ThreeAZ", "10.2.0.0/16", 3, 6},
	}

	for _, tc := range testCases {
		// Captura variável para goroutine
		tc := tc

		t.Run(tc.name, func(t *testing.T) {
			t.Parallel()

			terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
				TerraformDir: "../",
				Vars: map[string]interface{}{
					"project":     fmt.Sprintf("test-%s", strings.ToLower(tc.name)),
					"environment": "test",
					"vpc_cidr":    tc.cidr,
					"az_count":    tc.azCount,
				},
			})

			defer terraform.Destroy(t, terraformOptions)
			terraform.InitAndApply(t, terraformOptions)

			subnets := terraform.OutputList(t, terraformOptions, "all_subnet_ids")
			assert.Len(t, subnets, tc.expected)
		})
	}
}
```

### Retry e Resiliência

```go
func TestWithRetry(t *testing.T) {
	t.Parallel()

	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../",
		Vars: map[string]interface{}{
			"project":     "retry-test",
			"environment": "test",
		},

		// Retry na criação (AWS APIs podem ser flaky)
		MaxRetries:         3,
		TimeBetweenRetries: 10 * time.Second,
		RetryableTerraformErrors: map[string]string{
			"RequestError":     "Temporary AWS API error",
			"TooManyRequests":  "Rate limit",
			"OptInRequired":    "Region not opted in",
		},
	})

	defer terraform.Destroy(t, terraformOptions)
	terraform.InitAndApply(t, terraformOptions)
}
```

### Cleanup e Nuke

```bash
# Limpar recursos órfãos de testes (execute periodicamente)
# aws-nuke - Remove TODOS os recursos de uma conta
# cloud-nuke - Remove recursos com filtros

# Instalar cloud-nuke
go install github.com/gruntwork-io/cloud-nuke@latest

# Listar recursos antigos (mais de 24h)
cloud-nuke aws --list-resource-types
cloud-nuke aws --older-than 24h --dry-run

# Limpar (CUIDADO - apenas em contas de teste!)
cloud-nuke aws --older-than 24h
```

---

## Policy Testing — OPA / Conftest

### Conftest com Rego

```bash
# Gerar plan em JSON
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary > tfplan.json

# Rodar testes de policy
conftest test tfplan.json -p policy/
```

```
policy/
├── main.rego
├── s3.rego
├── iam.rego
├── networking.rego
└── tags.rego
```

```rego
# policy/tags.rego
package main

import rego.v1

# Regra: Todos os recursos devem ter tags obrigatórias
required_tags := {"Environment", "Project", "Team", "ManagedBy"}

deny contains msg if {
    resource := input.resource_changes[_]
    resource.change.actions[_] == "create"

    tags := object.get(resource.change.after, "tags", {})
    missing := required_tags - {key | tags[key]}
    count(missing) > 0

    msg := sprintf(
        "Resource %s (%s) is missing required tags: %v",
        [resource.address, resource.type, missing]
    )
}
```

```rego
# policy/s3.rego
package main

import rego.v1

# Regra: S3 buckets devem ter encryption habilitado
deny contains msg if {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket"
    resource.change.actions[_] == "create"

    # Verifica se não existe server_side_encryption_configuration
    not resource.change.after.server_side_encryption_configuration
    msg := sprintf("S3 bucket %s must have encryption enabled", [resource.address])
}

# Regra: S3 buckets não devem ser públicos
deny contains msg if {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket_public_access_block"
    resource.change.actions[_] == "create"

    config := resource.change.after
    not config.block_public_acls
    msg := sprintf("S3 bucket %s must block public ACLs", [resource.address])
}
```

```rego
# policy/iam.rego
package main

import rego.v1

# Regra: IAM policies não devem usar wildcard "*" em Action
deny contains msg if {
    resource := input.resource_changes[_]
    resource.type == "aws_iam_policy"
    resource.change.actions[_] == "create"

    policy := json.unmarshal(resource.change.after.policy)
    statement := policy.Statement[_]
    statement.Effect == "Allow"
    statement.Action[_] == "*"

    msg := sprintf(
        "IAM policy %s must not use wildcard '*' in Action",
        [resource.address]
    )
}

# Regra: IAM policies não devem usar wildcard "*" em Resource
deny contains msg if {
    resource := input.resource_changes[_]
    resource.type == "aws_iam_policy"
    resource.change.actions[_] == "create"

    policy := json.unmarshal(resource.change.after.policy)
    statement := policy.Statement[_]
    statement.Effect == "Allow"
    statement.Resource == "*"

    msg := sprintf(
        "IAM policy %s must not use wildcard '*' in Resource with Allow effect",
        [resource.address]
    )
}
```

```rego
# policy/networking.rego
package main

import rego.v1

# Regra: Security groups não devem permitir ingress 0.0.0.0/0
deny contains msg if {
    resource := input.resource_changes[_]
    resource.type == "aws_security_group_rule"
    resource.change.actions[_] == "create"

    resource.change.after.type == "ingress"
    resource.change.after.cidr_blocks[_] == "0.0.0.0/0"
    resource.change.after.from_port != 443
    resource.change.after.from_port != 80

    msg := sprintf(
        "Security group rule %s allows ingress from 0.0.0.0/0 on non-HTTP(S) port %d",
        [resource.address, resource.change.after.from_port]
    )
}
```

---

## Security Scanning — tfsec / Checkov

### tfsec

```bash
# Scan básico
tfsec .

# Com output em JUnit (para CI)
tfsec . --format=junit --out=tfsec-results.xml

# Ignorar regras específicas (com justificativa no código)
tfsec . --exclude=aws-s3-encryption-customer-key
```

```hcl
# Ignorar regra específica inline (com justificativa)
resource "aws_s3_bucket" "logs" {
  bucket = "my-logs" #tfsec:ignore:aws-s3-encryption-customer-key reason: uses SSE-S3
}
```

### Checkov

```bash
# Scan completo
checkov -d .

# Apenas checks de AWS
checkov -d . --framework terraform --check CKV_AWS*

# Output em JUnit
checkov -d . --output junitxml > checkov-results.xml

# Com baseline (ignora issues conhecidos)
checkov -d . --baseline .checkov.baseline
```

### Trivy (Scanner unificado)

```bash
# Scan de configuração Terraform
trivy config .

# Com severidade mínima
trivy config . --severity HIGH,CRITICAL

# Output em JSON
trivy config . -f json -o trivy-results.json
```

---

## Validação Estática — terraform validate + TFLint

```bash
# Validação sintática e semântica
terraform init -backend=false
terraform validate

# Lint com regras AWS
tflint --init
tflint --recursive --force

# Verificar formatação
terraform fmt -check -recursive -diff

# Pipeline completo de validação estática
terraform fmt -check -recursive && \
terraform validate && \
tflint --recursive && \
tfsec . && \
checkov -d .
```

---

## Contract Tests — Outputs e Interfaces

Garanta que módulos expõem os outputs esperados:

```hcl
# tests/unit/contract_test.tftest.hcl

variables {
  project     = "contract-test"
  environment = "dev"
  vpc_cidr    = "10.0.0.0/16"
}

# Contract: módulo VPC deve expor vpc_id
run "outputs_vpc_id" {
  command = plan

  assert {
    condition     = output.vpc_id != null
    error_message = "Module must output vpc_id"
  }
}

# Contract: módulo VPC deve expor private_subnet_ids como lista
run "outputs_private_subnets" {
  command = plan

  assert {
    condition     = output.private_subnet_ids != null
    error_message = "Module must output private_subnet_ids"
  }
}

# Contract: módulo VPC deve expor vpc_cidr
run "outputs_vpc_cidr" {
  command = plan

  assert {
    condition     = output.vpc_cidr != null
    error_message = "Module must output vpc_cidr"
  }
}
```

---

## Testes de Custo — Infracost

```bash
# Verificar se custo está dentro do budget
infracost breakdown --path=. --format=json | \
  jq '.totalMonthlyCost | tonumber' | \
  awk '{ if ($1 > 1000) { print "FAIL: Monthly cost $"$1" exceeds $1000 budget"; exit 1 } }'
```

```yaml
# .infracost/infracost.yml — Thresholds automáticos
version: 0.1
projects:
  - path: live/dev
    name: dev
  - path: live/prod
    name: prod
```

---

## Estratégia de Testes em CI

```yaml
# .github/workflows/terraform-test.yml
name: Terraform Tests

on:
  pull_request:
    paths:
      - 'modules/**'
      - 'live/**'

jobs:
  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.10.0"

      - name: Format Check
        run: terraform fmt -check -recursive

      - name: Validate
        run: |
          find modules -name "*.tf" -exec dirname {} \; | sort -u | while read dir; do
            echo "Validating $dir"
            cd "$dir"
            terraform init -backend=false
            terraform validate
            cd -
          done

      - name: TFLint
        uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: latest
      - run: |
          tflint --init
          tflint --recursive

      - name: tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: false

      - name: Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: .
          framework: terraform

  unit-tests:
    runs-on: ubuntu-latest
    needs: static-analysis
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.10.0"

      - name: Run Unit Tests
        run: |
          find modules -name "tests" -type d | while read test_dir; do
            module_dir=$(dirname "$test_dir")
            echo "Testing $module_dir"
            cd "$module_dir"
            terraform init -backend=false
            terraform test -filter=tests/unit/
            cd -
          done

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    if: github.event.pull_request.base.ref == 'main'
    environment: test-aws
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-arn: ${{ secrets.AWS_TEST_ROLE_ARN }}
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.10.0"

      - name: Run Integration Tests
        run: |
          find modules -name "tests" -type d | while read test_dir; do
            module_dir=$(dirname "$test_dir")
            echo "Integration testing $module_dir"
            cd "$module_dir"
            terraform init
            terraform test -filter=tests/integration/
            cd -
          done
        timeout-minutes: 30

  cost-check:
    runs-on: ubuntu-latest
    needs: static-analysis
    steps:
      - uses: actions/checkout@v4

      - name: Infracost
        uses: infracost/actions/setup@v3
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Cost Diff
        run: |
          infracost diff --path=. \
            --compare-to=infracost-base.json \
            --format=json --out-file=/tmp/infracost.json

      - name: Post Cost Comment
        uses: infracost/actions/comment@v1
        with:
          path: /tmp/infracost.json
          behavior: update
```

---

## Dicas e Patterns

### 1. Use Contas AWS Separadas para Testes

```
Contas AWS:
├── myorg-dev        (development)
├── myorg-staging    (staging)
├── myorg-prod       (production)
└── myorg-test       (testes automatizados — limpa periodicamente)
```

### 2. Naming Convention para Recursos de Teste

```hcl
# Prefixe recursos de teste para fácil identificação e cleanup
locals {
  test_prefix = "tftest-${random_id.test.hex}"
}
```

### 3. Timeout em Testes

```go
// Terratest — configure timeout
func TestWithTimeout(t *testing.T) {
	if testing.Short() {
		t.Skip("Skipping integration test in short mode")
	}
	// timeout via `go test -timeout 30m`
}
```

### 4. Skip Testes Caros em Desenvolvimento Local

```go
func TestExpensiveResource(t *testing.T) {
	// Skip se não estiver em CI
	if os.Getenv("CI") == "" {
		t.Skip("Skipping expensive test outside CI")
	}
}
```

### 5. Makefile para Testes

```makefile
# Makefile
.PHONY: test test-unit test-integration test-static

test-static:
	terraform fmt -check -recursive
	terraform validate
	tflint --recursive
	tfsec .
	checkov -d .

test-unit:
	@find modules -name "tests" -type d -exec sh -c \
		'cd $$(dirname {}) && terraform init -backend=false && terraform test -filter=tests/unit/' \;

test-integration:
	@find modules -name "tests" -type d -exec sh -c \
		'cd $$(dirname {}) && terraform init && terraform test -filter=tests/integration/' \;

test: test-static test-unit

test-all: test-static test-unit test-integration
```

````
