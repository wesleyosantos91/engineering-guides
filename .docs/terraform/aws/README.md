````markdown
# Terraform — AWS

> **Objetivo:** Guia completo de boas práticas, padrões e convenções para projetos Terraform focados em AWS.
> Baseado em práticas da comunidade, documentação oficial da HashiCorp, AWS Well-Architected Framework
> e experiências de empresas como Netflix, Airbnb, Nubank, Mercado Livre e iFood.

---

## Índice de Documentos

| Documento | Descrição |
|-----------|-----------|
| [project-structure.md](project-structure.md) | Estrutura de diretórios, organização de módulos e workspaces |
| [best-practices.md](best-practices.md) | Boas práticas gerais de Terraform com foco em AWS |
| [testing.md](testing.md) | Testes unitários, de integração, contract tests e compliance |
| [modules.md](modules.md) | Criação, versionamento e consumo de módulos reutilizáveis |
| [state-management.md](state-management.md) | Gerenciamento de state — backends, locking, isolation |
| [security.md](security.md) | Segurança, IAM, secrets, scanning e compliance na AWS |
| [ci-cd.md](ci-cd.md) | Pipelines de CI/CD, automação e GitOps com Terraform |
| [copilot-instructions.md](copilot-instructions.md) | Instruções para GitHub Copilot gerar código Terraform idiomático |

---

## Stack Recomendada

| Categoria | Ferramenta |
|-----------|-----------|
| **IaC** | Terraform 1.6+ (OpenTofu como alternativa) |
| **Provider** | AWS Provider (`hashicorp/aws`) 5.x+ |
| **State Backend** | S3 + DynamoDB (locking) |
| **Módulos** | Terraform Registry + módulos internos |
| **Testes Unitários** | `terraform test` (nativo 1.6+) |
| **Testes Integração** | Terratest (`gruntwork-io/terratest`) |
| **Policy as Code** | OPA/Conftest, Sentinel, tfsec, Checkov |
| **Lint** | TFLint (`terraform-linters/tflint`) |
| **Formatação** | `terraform fmt` |
| **Docs** | terraform-docs (`terraform-docs/terraform-docs`) |
| **Segurança** | tfsec, Checkov, Trivy, AWS Config |
| **CI/CD** | GitHub Actions, GitLab CI, AWS CodePipeline |
| **Secrets** | AWS Secrets Manager, SSM Parameter Store |
| **Cost** | Infracost (`infracost/infracost`) |

---

## Princípios Fundamentais

1. **Infrastructure as Code é código** — aplique as mesmas práticas de engenharia de software
2. **Imutabilidade** — prefira destruir e recriar a modificar in-place quando possível
3. **Idempotência** — `terraform apply` deve produzir o mesmo resultado independente de quantas vezes for executado
4. **DRY (Don't Repeat Yourself)** — use módulos para evitar duplicação
5. **Least Privilege** — IAM roles com permissões mínimas necessárias
6. **State é sagrado** — proteja, versione e faça backup do state
7. **Plan antes de Apply** — sempre revise o plan antes de aplicar mudanças
8. **Blast Radius** — minimize o impacto de mudanças isolando state por domínio

````
