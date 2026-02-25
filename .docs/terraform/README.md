# Terraform — Guia Completo

> **Objetivo:** Guia centralizado de boas práticas, padrões e convenções para projetos **Terraform**, organizado por cloud provider.
> Baseado em práticas da comunidade, documentação oficial da HashiCorp e experiências de empresas de referência no mercado.
>
> **Uso como Knowledge Base:** Esta documentação serve como base de conhecimento para o GitHub Copilot.
> Ao gerar código Terraform, siga rigorosamente os padrões descritos nos guias específicos de cada provider —
> naming conventions, estrutura de arquivos, validações, segurança e testes.

---

## Guias por Cloud Provider

| Provider | Guia | Descrição |
|----------|------|-----------|
| **AWS** | [Terraform — AWS](aws/README.md) | Estrutura de projeto, boas práticas, módulos, state management, segurança e testes para AWS |

---

## Princípios Universais

Independente do cloud provider, todo projeto Terraform deve seguir estes princípios:

1. **Infrastructure as Code é código** — aplique as mesmas práticas de engenharia de software (review, testes, CI/CD)
2. **Imutabilidade** — prefira destruir e recriar a modificar in-place quando possível
3. **Idempotência** — `terraform apply` deve produzir o mesmo resultado independente de quantas vezes for executado
4. **DRY (Don't Repeat Yourself)** — use módulos para evitar duplicação
5. **Least Privilege** — roles com permissões mínimas necessárias
6. **State é sagrado** — proteja, versione e faça backup do state
7. **Plan antes de Apply** — sempre revise o plan antes de aplicar mudanças
8. **Blast Radius** — minimize o impacto de mudanças isolando state por domínio
9. **Fail Fast** — use validações e `check` blocks para detectar problemas antes do apply
10. **Segurança por padrão** — encryption, logging e acesso restrito habilitados por default

---

## Convenções Gerais

| Convenção | Padrão |
|-----------|--------|
| **Naming** | `snake_case` para recursos, variáveis e outputs |
| **Variáveis** | Sempre com `description`, `type` explícito e `validation` quando aplicável |
| **Outputs** | Sempre com `description` |
| **Loops** | `for_each` para coleções, `count` apenas para toggles 0/1 |
| **Locals** | Para computações e prefixos de naming |
| **Tags** | `default_tags` no provider para tags padrão |
| **Lifecycle** | `prevent_destroy` em recursos críticos |
| **Refatoração** | `moved` blocks em vez de destroy+create |
| **Formatação** | `terraform fmt` obrigatório |
| **Documentação** | `terraform-docs` para geração automática |

---

## Versões Mínimas Requeridas

```hcl
terraform {
  required_version = ">= 1.10.0, < 2.0.0"
}
```

---

## Referências

- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [HashiCorp Learn](https://developer.hashicorp.com/terraform/tutorials)
- [Terraform Registry](https://registry.terraform.io/)
