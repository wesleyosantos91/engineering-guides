# Trilha API — Zero to Hero

> **Objetivo:** Dominar os 3 paradigmas de API do mercado — REST, GraphQL e gRPC —
> projetando, implementando e operando APIs production-ready com Java 25 e Go 1.26.

**Referência:** [.docs/API/](../../.docs/API/)

---

## Domínio: Online Marketplace

Todos os desafios usam o domínio de um **Marketplace Online** (inspirado em Mercado Livre, Amazon, Shopify).
Entidades principais:

| Entidade | Descrição |
|----------|-----------|
| `Product` | Produto à venda (nome, preço, estoque, categoria, seller) |
| `Order` | Pedido com itens, status, pagamento, frete |
| `User` | Comprador ou vendedor com perfil e endereço |
| `Payment` | Pagamento associado a um pedido |
| `Review` | Avaliação de produto com nota e comentário |
| `Category` | Categoria hierárquica de produtos |
| `Inventory` | Controle de estoque com reservas |
| `Notification` | Notificação de eventos (pedido, pagamento, entrega) |

**Por que Marketplace?**
- REST: API pública para catálogo, pedidos, usuários (CRUD clássico)
- GraphQL: Storefront flexível para web/mobile (BFF, queries personalizadas)
- gRPC: Comunicação interna entre serviços (inventário, pagamentos, notificações)

---

## Paradigmas e Quando Usar

```
                    ┌─────────────────────────────────────────────┐
                    │              Sua Necessidade                 │
                    └─────────┬───────────┬───────────┬───────────┘
                              │           │           │
                    ┌─────────▼──┐  ┌─────▼──────┐  ┌▼───────────┐
                    │ API Pública │  │ Flexível   │  │ Microserv. │
                    │ Web/Mobile  │  │ Multi-client│  │ Alta Perf. │
                    │ CRUD        │  │ BFF        │  │ Streaming  │
                    └─────────┬──┘  └─────┬──────┘  └┬───────────┘
                              │           │           │
                         ┌────▼──┐   ┌────▼────┐  ┌──▼───┐
                         │ REST  │   │ GraphQL │  │ gRPC │
                         └───────┘   └─────────┘  └──────┘
```

| Critério | REST | GraphQL | gRPC |
|----------|:----:|:-------:|:----:|
| Curva de aprendizado | Baixa | Média | Alta |
| Browser support | Nativo | Nativo | Via proxy |
| Over/Under-fetching | Possível | Eliminado | N/A (tipado) |
| Caching HTTP | Nativo | Complexo | Não nativo |
| Performance | Boa | Boa | Excelente |
| Streaming | Limitado (SSE) | Subscriptions | Bidirecional nativo |
| Contratos | OpenAPI (opcional) | Schema (forte) | Protobuf (forte) |

---

## Equivalências Java 25 ↔ Go 1.26

| Conceito | Java 25 | Go 1.26 |
|----------|---------|---------|
| HTTP Server | `com.sun.net.httpserver.HttpServer` | `net/http` + `http.ServeMux` |
| HTTP Client | `java.net.http.HttpClient` | `net/http` + `http.Client` |
| JSON | Jackson (`com.fasterxml.jackson`) | `encoding/json` |
| Protobuf | `com.google.protobuf` | `google.golang.org/protobuf` |
| gRPC | `io.grpc` | `google.golang.org/grpc` |
| GraphQL | `graphql-java` | `github.com/99designs/gqlgen` |
| Concorrência | Virtual Threads | Goroutines + Channels |
| Validação | Records + compact constructors | Factory functions + error returns |
| Testes | JUnit 5 + AssertJ + Mockito | `testing` + `testify` + `httptest` |
| OpenAPI | Swagger Codegen / manual | `swag` / manual |

---

## Dependências Permitidas

| Paradigma | Java 25 | Go 1.26 |
|-----------|---------|---------|
| **REST** | Apenas stdlib (`java.net.http`, `com.sun.net.httpserver`) + Jackson | Apenas stdlib (`net/http`, `encoding/json`) |
| **GraphQL** | `graphql-java` (engine) + Jackson | `gqlgen` (codegen) |
| **gRPC** | `io.grpc` + `com.google.protobuf` + `protoc` | `google.golang.org/grpc` + `google.golang.org/protobuf` + `protoc` |
| **Testes** | JUnit 5, AssertJ, Mockito | `testing`, `testify`, `httptest`, `bufconn` |
| **Build** | Gradle ou Maven | `go` toolchain |

> **Proibido:** Frameworks web (Spring, Gin, Echo, Fiber, Quarkus, etc.), ORMs (Hibernate, GORM, etc.), servidores embedded de terceiros.

---

## Mapa de Níveis

| Level | Tema | Paradigma | Desafios |
|-------|------|-----------|----------|
| [0](00-http-foundations.md) | Fundamentos HTTP & API Design | Transversal | 6 |
| [1](01-rest-api.md) | REST API — Design & Implementação | REST | 8 |
| [2](02-rest-advanced.md) | REST API — Produção & Operação | REST | 8 |
| [3](03-graphql.md) | GraphQL — Schema & Operações | GraphQL | 8 |
| [4](04-graphql-advanced.md) | GraphQL — Performance & Escala | GraphQL | 7 |
| [5](05-grpc.md) | gRPC — Protobuf & Comunicação | gRPC | 8 |
| [6](06-grpc-advanced.md) | gRPC — Produção & Operação | gRPC | 7 |
| [7](07-capstone-multi-paradigm.md) | Capstone — API Platform Multi-Paradigma | Todos | 8 |

---

## Estrutura de Cada Desafio

```
level-N-tema/
└── README.md          ← Contexto, requisitos, exemplos Java/Go, critérios de aceite
```

Cada desafio inclui:
- **Contexto** — cenário de negócio no domínio Marketplace
- **Requisitos** — o que implementar com critérios objetivos
- **Exemplos** — snippets em Java 25 e Go 1.26
- **Critérios de aceite** — checklist verificável
- **Anti-patterns** — o que evitar

---

## Critérios de Qualidade Globais

- [ ] Código compila e executa em ambas as linguagens
- [ ] Testes automatizados para cada endpoint/operação
- [ ] Error handling consistente com padrões do paradigma
- [ ] HTTP status codes corretos (REST), error codes (GraphQL), status codes (gRPC)
- [ ] Documentação da API (OpenAPI, GraphQL Schema, `.proto` files)
- [ ] Zero dados sensíveis em logs ou URLs
- [ ] Idempotência em operações com side-effects

---

## Próxima Trilha

→ [Voltar ao índice](../README.md)
