# API Engineering Guide

> Guia centralizado de boas práticas para design, implementação e operação de APIs, cobrindo os três principais paradigmas do mercado.

---

## Paradigmas de API

| Paradigma | Protocolo | Formato | Melhor Para |
|-----------|-----------|---------|-------------|
| [**REST**](rest/rest-best-practices.md) | HTTP/1.1, HTTP/2 | JSON | APIs públicas, CRUD, web apps |
| [**GraphQL**](graphql/graphql-best-practices.md) | HTTP/1.1, HTTP/2 | JSON | APIs flexíveis, BFF, múltiplos clients |
| [**gRPC**](grpc/grpc-best-practices.md) | HTTP/2 | Protobuf (binário) | Microserviços internos, alta performance, streaming |

---

## Quando Usar Cada Paradigma

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

### Comparação Detalhada

| Critério | REST | GraphQL | gRPC |
|----------|:----:|:-------:|:----:|
| Curva de aprendizado | Baixa | Média | Alta |
| Suporte em browsers | Nativo | Nativo | Via proxy (gRPC-Web) |
| Over-fetching | Possível | Eliminado | N/A (tipado) |
| Under-fetching | Possível | Eliminado | N/A (tipado) |
| Real-time / Streaming | Limitado (SSE/WS) | Subscriptions | Nativo bidirecional |
| Caching HTTP | Nativo | Complexo | Não nativo |
| Code generation | Opcional | Opcional | Obrigatório |
| Ecossistema / Tooling | Vasto | Grande | Crescente |
| Performance (latência) | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Tamanho do payload | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Contratos/Tipagem | Fraca (OpenAPI) | Forte (Schema) | Forte (Protobuf) |

---

## Boas Práticas Transversais (Aplicáveis a Todos)

### Segurança
- TLS/HTTPS obrigatório em produção
- Autenticação e autorização em todas as operações
- Rate limiting para proteção contra abuso
- Validação rigorosa de inputs
- Princípio do menor privilégio

### Observabilidade
- Distributed tracing (OpenTelemetry)
- Logging estruturado com correlation ID
- Métricas de latência, throughput e error rate
- Health checks padronizados
- Alertas configurados para SLOs

### Resiliência
- Timeouts em todas as chamadas externas
- Circuit breaker para dependências
- Retry com exponential backoff e jitter
- Graceful degradation
- Idempotência em operações críticas

### Evolução
- Versionamento com estratégia clara
- Backward compatibility como regra
- Deprecation com prazos comunicados
- Changelog público e atualizado
- Contract testing automatizado

---

## Documentos

| Documento | Descrição |
|-----------|-----------|
| [REST Best Practices](rest/rest-best-practices.md) | URIs, métodos HTTP, status codes, paginação, caching, segurança, OpenAPI |
| [GraphQL Best Practices](graphql/graphql-best-practices.md) | Schema design, queries, mutations, pagination, federation, caching |
| [gRPC Best Practices](grpc/grpc-best-practices.md) | Protobuf design, streaming, deadlines, interceptors, load balancing |
