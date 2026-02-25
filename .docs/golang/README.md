# Go (Golang) — Guia de Estrutura de Projetos

> **Objetivo deste guia:** Servir como referência organizacional para projetos Go,
> cobrindo frameworks e convenções do ecossistema. Cada documento detalha **estrutura de diretórios,
> convenções de código, organização de pacotes e configurações** seguindo os padrões da comunidade Go.

> **Uso como Knowledge Base:** Este conjunto de documentos é projetado para servir como
> base de conhecimento para assistentes de código (Copilot). Cada documento é autocontido
> e alinhado com [Effective Go](https://go.dev/doc/effective_go),
> [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
> e [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments).

---

## Frameworks

| Framework | Descrição | Documento |
|-----------|-----------|-----------|
| [**Gin**](01-gin.md) | Framework HTTP de alta performance — o mais popular do ecossistema Go para APIs REST | Estrutura de pacotes, handlers, middlewares, configuração por environment |

---

## Princípios da Comunidade Go

Os guias seguem os princípios fundamentais do ecossistema Go:

1. **Simplicidade** — código claro e direto, sem abstrações desnecessárias
2. **Pacotes pequenos e coesos** — cada pacote com responsabilidade única
3. **Interfaces implícitas** — composição sobre herança, interfaces definidas pelo consumidor
4. **Erros como valores** — tratamento explícito de erros, sem exceções
5. **Convenções da stdlib** — nomes curtos, packages lowercase, sem underscores

---

## Estrutura Padrão Go

Referência: [golang-standards/project-layout](https://github.com/golang-standards/project-layout)

```
├── cmd/           → Entrypoints da aplicação
├── internal/      → Código privado (não importável externamente)
├── pkg/           → Código público reutilizável (opcional)
├── api/           → Specs OpenAPI, Protobuf, schemas
├── configs/       → Arquivos de configuração
├── migrations/    → Scripts de migração de banco
├── scripts/       → Scripts auxiliares (build, deploy)
├── test/          → Testes de integração e fixtures
├── docs/          → Documentação
├── go.mod         → Definição do módulo
└── go.sum         → Lock de dependências
```
