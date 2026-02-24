# Terraform — AWS

> **Objetivo:** Guia completo de boas práticas, padrões e convenções para projetos Terraform focados em AWS.
> Baseado em práticas da comunidade, documentação oficial da HashiCorp, AWS Well-Architected Framework
> e experiências de empresas como Netflix, Airbnb, Nubank, Mercado Livre e iFood.
>
> **Uso como Knowledge Base:** Esta documentação serve como base de conhecimento para o GitHub Copilot.
> Ao gerar código Terraform, siga rigorosamente os padrões aqui descritos — naming conventions,
> estrutura de arquivos, validações, segurança e testes.

---

## Índice de Documentos

| Documento | Descrição |
|-----------|-----------|
| [project-structure.md](project-structure.md) | Estrutura de diretórios, organização de módulos e workspaces |
| [best-practices.md](best-practices.md) | Boas práticas gerais, check blocks, ephemeral values, provider functions |
| [testing.md](testing.md) | Testes unitários, de integração, contract tests e compliance |
| [modules.md](modules.md) | Criação, versionamento e consumo de módulos reutilizáveis |
| [state-management.md](state-management.md) | Gerenciamento de state — backends, native S3 locking, isolation |
| [security.md](security.md) | Segurança, IAM, secrets, ephemeral values, scanning e compliance na AWS |

---

## Stack Recomendada

| Categoria | Ferramenta |
|-----------|-----------|
| **IaC** | Terraform 1.10+ (OpenTofu 1.8+ como alternativa) |
| **Provider** | AWS Provider (`hashicorp/aws`) 5.x+ (com provider-defined functions 5.40+) |
| **State Backend** | S3 com native locking (`use_lockfile = true`) — DynamoDB apenas para projetos legados |
| **Módulos** | Terraform Registry + módulos internos |
| **Testes Unitários** | `terraform test` (nativo 1.6+, mocks 1.7+, provider mocking 1.8+) |
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
9. **Fail Fast** — use `check` blocks e validações para detectar problemas antes do apply
10. **Segurança por padrão** — encryption, logging e acesso restrito habilitados por default em todos os módulos

---

## Convenções para Geração de Código

Ao gerar código Terraform para AWS, sempre:

- Use `snake_case` para nomes de recursos, variáveis e outputs
- Adicione `description` em **todas** as variáveis e outputs
- Adicione `type` explícito em **todas** as variáveis
- Adicione `validation` blocks em variáveis que aceitam valores restritos
- Use `for_each` ao invés de `count` para coleções (exceto toggles 0/1)
- Use `locals` para computações e prefixos de naming, evitando repetição
- Use `default_tags` no provider para tags padrão
- Inclua `lifecycle` blocks quando apropriado (`prevent_destroy` em recursos críticos)
- Prefira `moved` blocks em vez de destroy+create ao refatorar
- Use `terraform_data` ao invés de `null_resource`
- Use `removed` blocks para remover recursos do gerenciamento sem destruí-los
- Sempre use `check` blocks para validações pós-apply de aspectos críticos
- Use `ephemeral` values (1.10+) para secrets que não devem persistir no state
- Use provider-defined functions (`provider::aws::arn_parse`) para manipulação de ARNs
- Garanta encryption at rest e in transit por padrão em todos os serviços

---

## Versões Mínimas Requeridas

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
```
