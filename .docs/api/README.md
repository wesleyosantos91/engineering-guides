# API Engineering Guide

> Guia centralizado de boas práticas para design, implementação e operação de APIs, cobrindo os três principais paradigmas do mercado.

## Como Usar Este Guia

Este guia serve como **base de conhecimento** para orientar decisões de design e implementação de APIs. Ao consultar:

1. **Identifique o paradigma** adequado usando a seção [Quando Usar Cada Paradigma](#quando-usar-cada-paradigma).
2. **Siga as boas práticas transversais** que se aplicam a qualquer tipo de API.
3. **Consulte o documento específico** do paradigma escolhido para detalhes de implementação.
4. **Priorize segurança e observabilidade** desde o início — não são itens opcionais.

> **Regra geral**: quando em dúvida entre paradigmas, comece com REST para APIs externas e gRPC para comunicação interna entre serviços.

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
| Performance (latência) | Boa | Boa | Excelente |
| Tamanho do payload | Médio (JSON) | Médio (JSON) | Compacto (binário) |
| Contratos/Tipagem | Fraca (OpenAPI) | Forte (Schema) | Forte (Protobuf) |

### Critérios de Decisão

Use estas perguntas para escolher o paradigma:

| Pergunta | Se Sim → |
|----------|----------|
| A API será consumida por browsers ou clientes externos? | REST |
| Múltiplos clientes precisam de dados diferentes do mesmo endpoint? | GraphQL |
| Comunicação entre microserviços internos com requisitos de latência? | gRPC |
| Precisa de streaming bidirecional em tempo real? | gRPC |
| O time tem pouca experiência com APIs? | REST (menor curva) |
| Precisa expor um grafo complexo de entidades para um frontend? | GraphQL |
| Comunicação com dispositivos IoT ou banda limitada? | gRPC |
| Necessidade de forte caching HTTP? | REST |

> **Nota**: É comum combinar paradigmas — por exemplo, gRPC internamente entre serviços e REST ou GraphQL na borda (API Gateway).

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

---

## API Gateway — Considerações Transversais

Independente do paradigma, considere um API Gateway para centralizar:

| Responsabilidade | Descrição |
|------------------|-----------|
| **Autenticação/Autorização** | Validação de tokens antes de chegar ao serviço |
| **Rate Limiting** | Controle de abuso centralizado |
| **Logging/Tracing** | Injeção de correlation IDs |
| **TLS Termination** | Gestão centralizada de certificados |
| **Routing** | Direcionamento para serviços corretos por path/header |
| **Transformação** | Tradução entre protocolos (ex: REST → gRPC via gRPC-Gateway) |
| **Circuit Breaking** | Proteção contra cascading failures |

### Exemplos de Soluções

| Solução | Tipo | Melhor Para |
|---------|------|-------------|
| **Kong** | Open-source | REST/GraphQL, plugins extensíveis |
| **Envoy** | Open-source | gRPC nativo, service mesh (Istio) |
| **AWS API Gateway** | Managed | REST/WebSocket na AWS |
| **Apollo Router** | Open-source | GraphQL Federation |
| **Apigee** | Managed | APIs públicas enterprise |

