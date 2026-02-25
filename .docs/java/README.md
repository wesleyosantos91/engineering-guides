# Java — Guia de Estrutura de Projetos

> **Objetivo deste guia:** Servir como referência organizacional para projetos Java,
> cobrindo os principais frameworks do ecossistema. Cada documento detalha **estrutura de diretórios,
> convenções de código, organização de pacotes e configurações** de forma prática e opinada.

> **Uso como Knowledge Base:** Este conjunto de documentos é projetado para servir como
> base de conhecimento para assistentes de código (Copilot). Cada documento é autocontido
> e focado em um framework específico, com convenções alinhadas às boas práticas da comunidade Java.

---

## Frameworks

| Framework | Descrição | Documento |
|-----------|-----------|-----------|
| [**Spring Boot**](01-spring.md) | Framework mais popular do ecossistema Java — convenção sobre configuração, vasto ecossistema de starters | Estrutura de pacotes, convenções, profiles, testes |
| [**Quarkus**](02-quarkus.md) | Framework cloud-native otimizado para GraalVM e containers — startup rápido, baixo consumo de memória | Estrutura JAX-RS, CDI, Panache, configuração por profile |
| [**Micronaut**](03-micronaut.md) | Framework com compile-time DI — sem reflection, ideal para microserviços e serverless | Estrutura de pacotes, DI em compile-time, configuração por environment |

---

## Qual framework escolher?

```
Qual é o seu cenário?
│
├── Ecossistema maduro, muitas libs, grande comunidade?
│   └── Spring Boot
│
├── Cloud-native, GraalVM, startup ultra-rápido?
│   └── Quarkus
│
├── Compile-time DI, serverless, baixa latência de cold start?
│   └── Micronaut
│
└── Não tem certeza?
    └── Spring Boot (maior comunidade e documentação)
```

---

## Comparativo Rápido

| Aspecto | Spring Boot | Quarkus | Micronaut |
|---------|-------------|---------|-----------|
| **DI** | Runtime (reflexão) | CDI (build-time otimizado) | Compile-time (sem reflexão) |
| **Startup** | Moderado | Muito rápido | Muito rápido |
| **Memória** | Moderado | Baixo | Baixo |
| **GraalVM Native** | Suportado | Otimizado | Otimizado |
| **Ecossistema** | Vastíssimo | Crescente | Crescente |
| **Curva de aprendizado** | Suave (muita documentação) | Moderada | Moderada |
| **Comunidade** | Muito grande | Grande | Média |

---

## Princípios Comuns

Independente do framework, todos os guias seguem os mesmos princípios de organização:

1. **Arquitetura em camadas** — separação clara entre API, domínio e infraestrutura
2. **Convenção de nomes** — sufixos consistentes (`Controller`, `Service`, `Repository`, `DTO`, etc.)
3. **Configuração por ambiente** — profiles/environments para dev, staging e produção
4. **Testabilidade** — estrutura que facilita testes unitários e de integração
5. **Extensibilidade** — pontos de extensão para mensageria, bancos de dados e protocolos de API
