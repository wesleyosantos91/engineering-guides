# Level 3 — Testes

> **Objetivo:** Implementar uma estratégia completa de testes para infraestrutura — testes unitários nativos (`terraform test`), testes de integração com Terratest, policy testing com OPA/Conftest, e security scanning com tfsec/Checkov. Todos rodando contra LocalStack.

---

## Objetivo de Aprendizado

- Implementar testes unitários com `terraform test` (plan-only e mocks)
- Implementar testes de integração com `terraform test` (apply) contra LocalStack
- Escrever testes de integração em Go com Terratest + LocalStack
- Aplicar security scanning com tfsec e Checkov
- Escrever policies com OPA/Conftest para compliance organizacional
- Configurar validação estática: `terraform validate` + TFLint
- Criar contract tests para interfaces de módulos
- Montar pipeline de testes em CI (GitHub Actions)

---

## Escopo Funcional

### Pirâmide de Testes para ShopFlow

```
         ▲
        / \        Testes E2E / Smoke
       /   \       (apply completo + validação via AWS CLI)
      /     \
     /───────\     Testes de Integração
    /         \    (Terratest / terraform test apply — LocalStack)
   /           \
  /─────────────\  Testes Unitários
 /               \ (terraform test plan-only + mocks)
/─────────────────\ Análise Estática
                    (fmt, validate, tflint, tfsec, checkov, conftest)
```

| Nível | Ferramenta | Tempo | Custo | Frequência |
|-------|-----------|-------|-------|------------|
| Análise Estática | fmt, validate, tflint, tfsec, checkov | Segundos | $0 | Todo commit |
| Unitário | `terraform test` (plan-only + mocks) | Segundos | $0 | Todo commit |
| Integração | Terratest / `terraform test` (apply) + LocalStack | Segundos-Minutos | $0 | PR / merge |
| E2E | Full apply + validação CLI | Minutos | $0 (LocalStack) | Nightly |

---

## Escopo Técnico

### Estrutura de Testes por Módulo

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
    ├── integration/
    │   └── full_deploy_test.tftest.hcl
    └── terratest/
        ├── vpc_test.go
        └── go.mod
```

---

## Tarefas

### Tarefa 1 — Testes Unitários com `terraform test` (Plan-Only)

Crie testes para o módulo VPC:

```hcl
# modules/networking/vpc/tests/unit/basic_test.tftest.hcl

variables {
  name                 = "test-vpc"
  cidr                 = "10.0.0.0/16"
  azs                  = ["us-east-1a", "us-east-1b"]
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]
}

# Teste: VPC é criada com CIDR correto
run "vpc_cidr_is_correct" {
  command = plan

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block should be 10.0.0.0/16"
  }
}

# Teste: Número correto de subnets
run "correct_subnet_count" {
  command = plan

  assert {
    condition     = length(aws_subnet.public) == 2
    error_message = "Should create 2 public subnets"
  }

  assert {
    condition     = length(aws_subnet.private) == 2
    error_message = "Should create 2 private subnets"
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

# Teste: Internet Gateway criado
run "internet_gateway_created" {
  command = plan

  assert {
    condition     = aws_internet_gateway.main.vpc_id == aws_vpc.main.id
    error_message = "Internet Gateway must be attached to the VPC"
  }
}
```

```hcl
# modules/networking/vpc/tests/unit/validation_test.tftest.hcl

# Teste: CIDR inválido é rejeitado
run "invalid_cidr_rejected" {
  command = plan

  variables {
    name                 = "test"
    cidr                 = "not-a-cidr"
    azs                  = ["us-east-1a"]
    public_subnet_cidrs  = ["10.0.1.0/24"]
    private_subnet_cidrs = ["10.0.10.0/24"]
  }

  expect_failures = [
    var.cidr,
  ]
}

# Teste: AZs vazio é rejeitado
run "empty_azs_rejected" {
  command = plan

  variables {
    name                 = "test"
    cidr                 = "10.0.0.0/16"
    azs                  = []
    public_subnet_cidrs  = []
    private_subnet_cidrs = []
  }

  expect_failures = [
    var.azs,
  ]
}
```

**Testes para os outros módulos:**

| Módulo | Arquivo de Teste | Cenários |
|--------|-----------------|----------|
| **S3** | `s3_basic_test.tftest.hcl` | Bucket name correto, versionamento habilitado, public access bloqueado |
| **S3** | `s3_validation_test.tftest.hcl` | Bucket name inválido rejeitado (maiúsculas, caracteres especiais) |
| **DynamoDB** | `dynamodb_basic_test.tftest.hcl` | Table criada, hash key correto, billing mode correto |
| **DynamoDB** | `dynamodb_gsi_test.tftest.hcl` | GSI criado com atributos corretos |
| **SQS** | `sqs_basic_test.tftest.hcl` | Fila criada, DLQ criada, redrive policy configurada |
| **SQS** | `sqs_validation_test.tftest.hcl` | Visibility timeout fora do range rejeitado |
| **Lambda** | `lambda_basic_test.tftest.hcl` | Função criada, runtime correto, memory correto |

### Tarefa 2 — Mocking de Providers

```hcl
# modules/networking/vpc/tests/unit/mock_test.tftest.hcl

# Mock do provider AWS — não precisa de LocalStack!
mock_provider "aws" {
  mock_data "aws_availability_zones" {
    defaults = {
      names = ["us-east-1a", "us-east-1b", "us-east-1c"]
    }
  }
}

variables {
  name                 = "test-vpc"
  cidr                 = "10.0.0.0/16"
  azs                  = ["us-east-1a", "us-east-1b"]
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]
}

run "vpc_with_mocked_provider" {
  command = plan

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR should match input"
  }
}
```

### Tarefa 3 — Testes de Integração com `terraform test` (Apply + LocalStack)

```hcl
# modules/storage/s3-bucket/tests/integration/full_deploy_test.tftest.hcl

# Provider real apontando para LocalStack
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
  s3_use_path_style           = true

  endpoints {
    s3  = "http://localhost:4566"
    iam = "http://localhost:4566"
    sts = "http://localhost:4566"
  }
}

variables {
  name              = "test-integration-bucket"
  enable_versioning = true
  tags = {
    Test = "integration"
  }
}

# Apply real — cria e destrói bucket no LocalStack
run "bucket_created_successfully" {
  command = apply

  assert {
    condition     = output.bucket_name == "test-integration-bucket"
    error_message = "Bucket name should match"
  }

  assert {
    condition     = output.bucket_arn != ""
    error_message = "Bucket ARN should not be empty"
  }
}
```

### Tarefa 4 — Terratest com Go + LocalStack

```go
// modules/networking/vpc/tests/terratest/vpc_test.go
package test

import (
	"testing"

	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestVpcModule(t *testing.T) {
	t.Parallel()

	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../../",
		Vars: map[string]interface{}{
			"name":                 "test-vpc",
			"cidr":                 "10.0.0.0/16",
			"azs":                  []string{"us-east-1a", "us-east-1b"},
			"public_subnet_cidrs":  []string{"10.0.1.0/24", "10.0.2.0/24"},
			"private_subnet_cidrs": []string{"10.0.10.0/24", "10.0.11.0/24"},
		},
		// Usa provider já configurado para LocalStack
		NoColor: true,
	})

	defer terraform.Destroy(t, terraformOptions)
	terraform.InitAndApply(t, terraformOptions)

	// Validar outputs
	vpcId := terraform.Output(t, terraformOptions, "vpc_id")
	assert.NotEmpty(t, vpcId, "VPC ID should not be empty")

	publicSubnetIds := terraform.OutputList(t, terraformOptions, "public_subnet_ids")
	assert.Equal(t, 2, len(publicSubnetIds), "Should have 2 public subnets")

	privateSubnetIds := terraform.OutputList(t, terraformOptions, "private_subnet_ids")
	assert.Equal(t, 2, len(privateSubnetIds), "Should have 2 private subnets")

	vpcCidr := terraform.Output(t, terraformOptions, "vpc_cidr")
	assert.Equal(t, "10.0.0.0/16", vpcCidr, "VPC CIDR should match")
}

func TestVpcModuleWithCustomCidr(t *testing.T) {
	t.Parallel()

	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../../",
		Vars: map[string]interface{}{
			"name":                 "test-vpc-custom",
			"cidr":                 "172.16.0.0/16",
			"azs":                  []string{"us-east-1a"},
			"public_subnet_cidrs":  []string{"172.16.1.0/24"},
			"private_subnet_cidrs": []string{"172.16.10.0/24"},
		},
		NoColor: true,
	})

	defer terraform.Destroy(t, terraformOptions)
	terraform.InitAndApply(t, terraformOptions)

	vpcCidr := terraform.Output(t, terraformOptions, "vpc_cidr")
	assert.Equal(t, "172.16.0.0/16", vpcCidr)
}
```

```go
// modules/storage/s3-bucket/tests/terratest/s3_test.go
package test

import (
	"fmt"
	"testing"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/credentials"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/s3"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

func TestS3BucketModule(t *testing.T) {
	t.Parallel()

	bucketName := fmt.Sprintf("test-bucket-%d", time.Now().UnixNano())

	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../../",
		Vars: map[string]interface{}{
			"name":              bucketName,
			"enable_versioning": true,
		},
		NoColor: true,
	})

	defer terraform.Destroy(t, terraformOptions)
	terraform.InitAndApply(t, terraformOptions)

	// Validar via AWS SDK apontando para LocalStack
	sess := session.Must(session.NewSession(&aws.Config{
		Region:           aws.String("us-east-1"),
		Endpoint:         aws.String("http://localhost:4566"),
		Credentials:      credentials.NewStaticCredentials("test", "test", ""),
		S3ForcePathStyle: aws.Bool(true),
	}))

	s3Client := s3.New(sess)

	// Verificar que bucket existe
	_, err := s3Client.HeadBucket(&s3.HeadBucketInput{
		Bucket: aws.String(bucketName),
	})
	require.NoError(t, err, "Bucket should exist")

	// Verificar versionamento
	versioning, err := s3Client.GetBucketVersioning(&s3.GetBucketVersioningInput{
		Bucket: aws.String(bucketName),
	})
	require.NoError(t, err)
	assert.Equal(t, "Enabled", *versioning.Status, "Versioning should be enabled")
}
```

### Tarefa 5 — Security Scanning com tfsec

```bash
# Escanear módulos
tfsec modules/

# Escanear um componente específico
tfsec live/dev/storage/

# Gerar relatório em formato JUnit (para CI)
tfsec live/dev/ --format junit --out tfsec-report.xml

# Escanear com severity mínima
tfsec live/dev/ --minimum-severity HIGH

# Ignorar regra específica (inline)
# resource "aws_s3_bucket" "logs" {
#   #tfsec:ignore:aws-s3-enable-bucket-logging
#   bucket = "..."
# }
```

**Regras a verificar:**

| Regra | Descrição | Severidade |
|-------|-----------|-----------|
| `aws-s3-enable-bucket-logging` | S3 bucket logging habilitado | MEDIUM |
| `aws-s3-enable-versioning` | S3 bucket versionamento habilitado | MEDIUM |
| `aws-s3-block-public-acls` | Block public ACLs | HIGH |
| `aws-dynamodb-enable-at-rest-encryption` | DynamoDB encryption at rest | HIGH |
| `aws-sqs-enable-queue-encryption` | SQS encryption | HIGH |
| `aws-vpc-no-public-ingress-sgr` | SG sem ingress público irrestrito | CRITICAL |

### Tarefa 6 — Security Scanning com Checkov

```bash
# Escanear diretório
checkov -d modules/

# Escanear com framework Terraform
checkov -d live/dev/storage/ --framework terraform

# Gerar relatório JSON
checkov -d live/dev/ --output json > checkov-report.json

# Verificar apenas checks específicos
checkov -d live/dev/ --check CKV_AWS_18,CKV_AWS_19,CKV_AWS_21

# Listar checks disponíveis
checkov --list --framework terraform | grep aws
```

### Tarefa 7 — Policy Testing com OPA/Conftest

```bash
# Instalar Conftest
brew install conftest    # macOS
choco install conftest   # Windows
```

```rego
# policy/terraform.rego
package main

# Regra: Todos os S3 buckets devem ter versionamento
deny[msg] {
  resource := input.resource.aws_s3_bucket_versioning[name]
  resource.versioning_configuration.status != "Enabled"
  msg := sprintf("S3 bucket '%s' must have versioning enabled", [name])
}

# Regra: DynamoDB deve ter encryption
deny[msg] {
  resource := input.resource.aws_dynamodb_table[name]
  not resource.server_side_encryption
  msg := sprintf("DynamoDB table '%s' must have server-side encryption", [name])
}

# Regra: Tags obrigatórias
deny[msg] {
  resource := input.resource[resource_type][name]
  required_tags := {"Environment", "Project", "ManagedBy"}
  tags := {tag | resource.tags[tag]}
  missing := required_tags - tags
  count(missing) > 0
  msg := sprintf("Resource '%s.%s' is missing required tags: %v", [resource_type, name, missing])
}

# Regra: CIDR não pode ser /0 (aberto para o mundo) em security groups
deny[msg] {
  resource := input.resource.aws_security_group[name]
  some i
  resource.ingress[i].cidr_blocks[_] == "0.0.0.0/0"
  resource.ingress[i].from_port != 443
  resource.ingress[i].from_port != 80
  msg := sprintf("Security group '%s' has unrestricted ingress on port %d", [name, resource.ingress[i].from_port])
}
```

```bash
# Gerar plan JSON e testar
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json

# Executar policies
conftest test tfplan.json --policy policy/

# Resultado esperado:
# FAIL - policy/terraform.rego - S3 bucket 'logs' must have versioning enabled
# PASS - 3 tests passed
```

### Tarefa 8 — Contract Tests (Interfaces de Módulos)

```hcl
# modules/networking/vpc/tests/unit/contract_test.tftest.hcl
# Testa que o módulo expõe todos os outputs necessários

variables {
  name                 = "contract-test"
  cidr                 = "10.0.0.0/16"
  azs                  = ["us-east-1a"]
  public_subnet_cidrs  = ["10.0.1.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24"]
}

# Contrato: vpc_id deve ser exportado
run "output_vpc_id_exists" {
  command = plan

  assert {
    condition     = output.vpc_id != null
    error_message = "Module must export vpc_id output"
  }
}

# Contrato: public_subnet_ids deve ser uma lista
run "output_public_subnets_is_list" {
  command = plan

  assert {
    condition     = length(output.public_subnet_ids) > 0
    error_message = "Module must export non-empty public_subnet_ids"
  }
}

# Contrato: private_subnet_ids deve ser uma lista
run "output_private_subnets_is_list" {
  command = plan

  assert {
    condition     = length(output.private_subnet_ids) > 0
    error_message = "Module must export non-empty private_subnet_ids"
  }
}

# Contrato: vpc_cidr deve ser exportado
run "output_vpc_cidr_matches_input" {
  command = plan

  assert {
    condition     = output.vpc_cidr == "10.0.0.0/16"
    error_message = "vpc_cidr output must match input CIDR"
  }
}
```

### Tarefa 9 — Pipeline de Testes em CI (GitHub Actions)

```yaml
# .github/workflows/terraform-ci.yml
name: Terraform CI

on:
  pull_request:
    paths: ['modules/**', 'live/**']
  push:
    branches: [main]
    paths: ['modules/**', 'live/**']

jobs:
  static-analysis:
    name: Static Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.10.0'

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Validate (modules)
        run: |
          for dir in modules/*/; do
            if [ -f "$dir/main.tf" ]; then
              echo "=== Validating $dir ==="
              cd "$dir" && terraform init -backend=false && terraform validate && cd -
            fi
          done

      - name: TFLint
        uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: latest

      - run: tflint --recursive --config .tflint.hcl

      - name: tfsec
        uses: aquasecurity/tfsec-action@v1.0.3
        with:
          soft_fail: false

      - name: Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: modules/
          framework: terraform

  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: static-analysis
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.10.0'

      - name: Run Unit Tests (Plan-only)
        run: |
          for module_dir in modules/*/*/; do
            if [ -d "$module_dir/tests" ]; then
              echo "=== Testing $module_dir ==="
              cd "$module_dir"
              terraform init -backend=false
              terraform test -filter=tests/unit/
              cd -
            fi
          done

  integration-tests:
    name: Integration Tests (LocalStack)
    runs-on: ubuntu-latest
    needs: unit-tests
    services:
      localstack:
        image: localstack/localstack:latest
        ports:
          - 4566:4566
        env:
          SERVICES: s3,sqs,sns,dynamodb,iam,sts,ec2,lambda,kms
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.10.0'

      - name: Run Integration Tests
        run: |
          for module_dir in modules/*/*/; do
            if [ -d "$module_dir/tests/integration" ]; then
              echo "=== Integration Testing $module_dir ==="
              cd "$module_dir"
              terraform init
              terraform test -filter=tests/integration/
              cd -
            fi
          done

      - name: Setup Go (for Terratest)
        uses: actions/setup-go@v5
        with:
          go-version: '1.23'

      - name: Run Terratest
        run: |
          for test_dir in modules/*/*/tests/terratest/; do
            if [ -d "$test_dir" ]; then
              echo "=== Terratest $test_dir ==="
              cd "$test_dir"
              go test -v -timeout 10m
              cd -
            fi
          done

  policy-tests:
    name: Policy Tests
    runs-on: ubuntu-latest
    needs: static-analysis
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.10.0'

      - name: Install Conftest
        run: |
          wget https://github.com/open-policy-agent/conftest/releases/latest/download/conftest_Linux_x86_64.tar.gz
          tar xzf conftest_Linux_x86_64.tar.gz
          sudo mv conftest /usr/local/bin/

      - name: Run Policy Tests
        run: |
          cd live/dev/storage
          terraform init
          terraform plan -out=tfplan
          terraform show -json tfplan > tfplan.json
          conftest test tfplan.json --policy ../../../policy/
```

---

## Critérios de Aceite

- [ ] Testes unitários (`terraform test` plan-only) em todos os módulos (pelo menos 20 assertions)
- [ ] Testes de validação (`expect_failures`) em todos os módulos
- [ ] Mock de provider exercitado em pelo menos 1 módulo
- [ ] Testes de integração (`terraform test` apply) contra LocalStack em pelo menos 2 módulos
- [ ] Terratest (Go) implementado para pelo menos 2 módulos com validação via AWS SDK
- [ ] tfsec executado em todos os módulos — zero findings HIGH/CRITICAL
- [ ] Checkov executado em todos os módulos — zero findings HIGH/CRITICAL
- [ ] Policies OPA/Conftest implementadas (pelo menos 4 regras)
- [ ] Contract tests verificando outputs de pelo menos 2 módulos
- [ ] Pipeline CI (GitHub Actions) configurado com todos os estágios
- [ ] Todos os testes passando com LocalStack

---

## Definição de Pronto (DoD)

- [ ] Testes unitários passando: `terraform test -filter=tests/unit/`
- [ ] Testes de integração passando contra LocalStack
- [ ] Security scanning sem findings críticos
- [ ] Pipeline CI funcional e documentado
- [ ] README.md do projeto atualizado com instruções de execução de testes
- [ ] Commit semântico: `feat(level-3): implement test strategy - unit, integration, security, policy`

---

## Checklist

- [ ] `terraform test` unitários em todos os módulos
- [ ] `terraform test` com mocks de provider
- [ ] `terraform test` integration (apply) contra LocalStack
- [ ] Terratest (Go) para VPC e S3 modules
- [ ] tfsec executado e findings resolvidos
- [ ] Checkov executado e findings resolvidos
- [ ] Conftest policies escritas e testadas
- [ ] Contract tests para interfaces de módulos
- [ ] GitHub Actions workflow criado
- [ ] Documentação de testes atualizada

---

## Extensões Opcionais

- [ ] Implementar testes de custo com Infracost em CI (falhar se custo > threshold)
- [ ] Adicionar Terratest com `test_structure.RunTestStage` para testes mais complexos
- [ ] Implementar custom TFLint rules
- [ ] Configurar Checkov custom policies
- [ ] Adicionar badge de CI no README
- [ ] Implementar mutation testing: alterar código e verificar que testes detectam
