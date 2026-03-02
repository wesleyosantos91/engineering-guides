# Zero to Hero — Terraform AWS Challenge (LocalStack)

> **Programa de especialização progressiva** em Terraform com foco em AWS,
> usando **LocalStack** para simulação local de serviços AWS.
> Todo o aprendizado acontece sem custos de cloud e sem necessidade de conta AWS.

---

## 1. Resumo Executivo

Este programa transforma um desenvolvedor em **especialista em Terraform para AWS**, através de desafios progressivos que cobrem desde fundamentos até segurança avançada e multi-environment deploys — tudo simulado localmente com **LocalStack**.

**Domínio escolhido:** Infraestrutura de uma **plataforma de e-commerce multi-ambiente** (ShopFlow) — um domínio realista que exige networking, compute, storage, messaging, observabilidade e segurança.

**Por que ShopFlow?**
- Cobre **todos os serviços AWS essenciais**: VPC, EC2, S3, RDS, SQS, SNS, Lambda, DynamoDB, IAM
- Exige **multi-environment** (dev, staging, prod) — cenário real de empresas
- Envolve **módulos reutilizáveis**, state isolation, segurança e testes
- Permite crescer de infraestrutura trivial até enterprise-grade IaC
- Perguntas de entrevista Staff/Principal frequentemente envolvem IaC e design de infra

**Por que LocalStack?**
- **$0 de custo** — simula AWS localmente via Docker
- **Seguro para aprender** — impossível causar billing acidental ou expor recursos
- **Rápido** — sem latência de rede, provisioning em segundos
- **Reproduzível** — mesmo ambiente em qualquer máquina
- **CI-friendly** — testes de infraestrutura em pipeline sem credenciais reais

---

## 2. Stack Técnica

| Categoria | Ferramenta |
|-----------|-----------|
| **IaC** | Terraform 1.10+ |
| **Provider** | AWS Provider (`hashicorp/aws`) 5.x+ |
| **Simulação AWS** | LocalStack (via Docker) |
| **Provider LocalStack** | `localstack/aws` ou AWS provider com custom endpoints |
| **Testes Unitários** | `terraform test` (nativo 1.6+, mocks 1.7+) |
| **Testes Integração** | Terratest (`gruntwork-io/terratest`) + LocalStack |
| **Policy as Code** | tfsec, Checkov, OPA/Conftest |
| **Lint** | TFLint (`terraform-linters/tflint`) |
| **Formatação** | `terraform fmt` |
| **Docs** | terraform-docs (`terraform-docs/terraform-docs`) |
| **Containers** | Docker / Docker Compose |

---

## 3. Índice de Desafios

| Level | Título | Foco |
|-------|--------|------|
| [**Level 0**](00-foundations.md) | Fundamentos | Terraform CLI, HCL, LocalStack setup, primeiros recursos |
| [**Level 1**](01-project-structure.md) | Estrutura de Projeto | Organização mono-repo, naming, workspaces vs directories |
| [**Level 2**](02-best-practices.md) | Boas Práticas | Variáveis, validações, lifecycle, check blocks, ephemeral values |
| [**Level 3**](03-testing.md) | Testes | terraform test, Terratest, tfsec, Checkov, policy testing |
| [**Level 4**](04-modules.md) | Módulos | Módulos reutilizáveis, versionamento, composição, registry |
| [**Level 5**](05-state-management.md) | State Management | Backends, locking, isolation, migração, disaster recovery |
| [**Level 6**](06-security.md) | Segurança | IAM, encryption, secrets, compliance, scanning pipeline |
| [**Level 7**](07-capstone-shopflow.md) | Capstone — ShopFlow | Plataforma completa multi-env com CI/CD e testes end-to-end |

---

## 4. Domínio: ShopFlow — E-Commerce Platform

### Visão Geral da Infraestrutura

```
                    ┌─────────────────────────────────────────────────┐
                    │              ShopFlow Infrastructure            │
                    │                                                  │
                    │  ┌──────────┐    ┌──────────┐    ┌──────────┐  │
                    │  │   VPC    │    │   VPC    │    │   VPC    │  │
                    │  │   DEV    │    │ STAGING  │    │   PROD   │  │
                    │  └────┬─────┘    └────┬─────┘    └────┬─────┘  │
                    │       │               │               │        │
                    │  ┌────┴─────┐    ┌────┴─────┐    ┌────┴─────┐  │
                    │  │ Public   │    │ Public   │    │ Public   │  │
                    │  │ Subnets  │    │ Subnets  │    │ Subnets  │  │
                    │  │ (ALB)    │    │ (ALB)    │    │ (ALB)    │  │
                    │  └────┬─────┘    └────┬─────┘    └────┬─────┘  │
                    │       │               │               │        │
                    │  ┌────┴─────┐    ┌────┴─────┐    ┌────┴─────┐  │
                    │  │ Private  │    │ Private  │    │ Private  │  │
                    │  │ Subnets  │    │ Subnets  │    │ Subnets  │  │
                    │  │(App+DB)  │    │(App+DB)  │    │(App+DB)  │  │
                    │  └──────────┘    └──────────┘    └──────────┘  │
                    │                                                  │
                    │  ┌────────────────────────────────────────────┐  │
                    │  │          Serviços Compartilhados           │  │
                    │  │  S3 · SQS · SNS · Lambda · DynamoDB · KMS │  │
                    │  └────────────────────────────────────────────┘  │
                    └─────────────────────────────────────────────────┘
```

### Componentes por Nível

| Componente | Level Introduzido | Serviços AWS |
|-----------|-------------------|-------------|
| **Networking** | Level 0-1 | VPC, Subnets, Security Groups, Internet Gateway |
| **Storage** | Level 1-2 | S3 (assets, logs, state) |
| **Compute** | Level 2-4 | Lambda, EC2 (opcional) |
| **Database** | Level 2-4 | DynamoDB, RDS (PostgreSQL) |
| **Messaging** | Level 4-5 | SQS, SNS, EventBridge |
| **Security** | Level 6 | IAM, KMS, Secrets Manager, SSM Parameter Store |
| **Observability** | Level 5-7 | CloudWatch Logs/Metrics/Alarms |
| **DNS/CDN** | Level 7 | Route53, CloudFront |

---

## 5. Setup do Ambiente

### Pré-requisitos

```bash
# Terraform 1.10+
terraform version

# Docker (para LocalStack)
docker --version

# LocalStack CLI (opcional, mas recomendado)
pip install localstack
localstack --version

# Ferramentas auxiliares
# TFLint
brew install tflint           # macOS
choco install tflint          # Windows

# tfsec
brew install tfsec            # macOS
choco install tfsec           # Windows

# terraform-docs
brew install terraform-docs   # macOS
choco install terraform-docs  # Windows

# Checkov
pip install checkov
```

### LocalStack via Docker Compose

```yaml
# docker-compose.yml
services:
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"            # Gateway único para todos os serviços
      - "4510-4559:4510-4559"  # Range para serviços externos
    environment:
      - SERVICES=s3,sqs,sns,lambda,dynamodb,iam,sts,ec2,rds,kms,ssm,secretsmanager,cloudwatch,logs,events,route53,cloudfront
      - DEBUG=0
      - PERSISTENCE=1          # Persistir estado entre restarts
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - localstack_data:/var/lib/localstack
      - /var/run/docker.sock:/var/run/docker.sock

volumes:
  localstack_data:
```

### Provider Configuration para LocalStack

```hcl
# providers.tf — Configuração para LocalStack
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
  s3_use_path_style           = true

  endpoints {
    s3             = "http://localhost:4566"
    sqs            = "http://localhost:4566"
    sns            = "http://localhost:4566"
    lambda         = "http://localhost:4566"
    dynamodb       = "http://localhost:4566"
    iam            = "http://localhost:4566"
    sts            = "http://localhost:4566"
    ec2            = "http://localhost:4566"
    rds            = "http://localhost:4566"
    kms            = "http://localhost:4566"
    ssm            = "http://localhost:4566"
    secretsmanager = "http://localhost:4566"
    cloudwatch     = "http://localhost:4566"
    cloudwatchlogs = "http://localhost:4566"
    events         = "http://localhost:4566"
    route53        = "http://localhost:4566"
    cloudfront     = "http://localhost:4566"
  }

  default_tags {
    tags = {
      Environment = "localstack"
      ManagedBy   = "terraform"
      Project     = "shopflow"
    }
  }
}
```

---

## 6. Convenções do Projeto

| Convenção | Padrão |
|-----------|--------|
| **Naming** | `snake_case` para recursos, variáveis e outputs |
| **Variáveis** | Sempre com `description`, `type` explícito e `validation` quando aplicável |
| **Outputs** | Sempre com `description` |
| **Loops** | `for_each` para coleções, `count` apenas para toggles 0/1 |
| **Locals** | Para computações e prefixos de naming |
| **Tags** | `default_tags` no provider para tags padrão |
| **Lifecycle** | `prevent_destroy` em recursos críticos |
| **Formatação** | `terraform fmt` obrigatório antes de cada commit |
| **Documentação** | `terraform-docs` para geração automática de README |
| **Testes** | Todos os módulos devem ter testes unitários (`terraform test`) |
| **Commits** | Semânticos: `feat(level-N): descrição` |

---

## 7. Filosofia

1. **Domínio unificado** — ShopFlow é usado em todos os níveis, cada nível adiciona componentes
2. **Progressão incremental** — Cada nível constrói sobre o anterior
3. **Custo zero** — LocalStack simula tudo localmente
4. **Critérios de aceite claros** — Cada desafio tem checklist objetivo e verificável
5. **Portfólio real** — O projeto resultante demonstra domínio de IaC em entrevistas
6. **Segurança por padrão** — Encryption, least privilege e compliance desde o Level 0
7. **Testável** — Toda infra é testada automaticamente (unit + integration + policy)

---

## 8. Referências

- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [LocalStack Documentation](https://docs.localstack.cloud/)
- [LocalStack Terraform Guide](https://docs.localstack.cloud/user-guide/integrations/terraform/)
- [Terratest](https://terratest.gruntwork.io/)
- [tfsec](https://aquasecurity.github.io/tfsec/)
- [Checkov](https://www.checkov.io/)
- [TFLint](https://github.com/terraform-linters/tflint)
