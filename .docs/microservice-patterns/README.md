# Microservice Patterns

> **Objetivo:** Referência conceitual sobre **padrões de microsserviços**.
> Abrange resiliência, persistência, event-driven, infraestrutura, deploy e observabilidade.
> **Agnóstico de linguagem e framework** — foca em conceitos, princípios e decisões arquiteturais.

---

## Como usar este guia

Este repositório é uma **base de conhecimento** para assistentes de IA (Copilot, etc.) e engenheiros.

- **Cada arquivo** é autossuficiente: contém problema, solução, conceitos, pseudocódigo, antipadrões e boas práticas.
- **Para escolher um padrão:** consulte a tabela [Referência Rápida](#referência-rápida--quando-usar-cada-padrão) abaixo.
- **Para entender a relação entre padrões:** cada arquivo contém uma seção "Relação com Outros Padrões" que conecta ao contexto mais amplo.
- **Para implementar:** use os pseudocódigos como ponto de partida — são agnósticos de linguagem e focam na lógica essencial.

---

## Sumário

- [Microservice Patterns](#microservice-patterns)
  - [Como usar este guia](#como-usar-este-guia)
  - [Sumário](#sumário)
  - [1. Resiliência](#1-resiliência)
  - [2. Persistência e Processamento de Dados](#2-persistência-e-processamento-de-dados)
  - [3. Event-Driven e Escalabilidade](#3-event-driven-e-escalabilidade)
  - [4. Infraestrutura e Deploy](#4-infraestrutura-e-deploy)
  - [5. Estratégias de Modernização e Deploy](#5-estratégias-de-modernização-e-deploy)
  - [6. Observabilidade](#6-observabilidade)
  - [Referência Rápida — Quando usar cada padrão](#referência-rápida--quando-usar-cada-padrão)

---

## 1. Resiliência

> Padrões que protegem o sistema contra falhas em cascata, sobrecarga e indisponibilidade de dependências externas.

| Padrão | Descrição | Arquivo |
|--------|-----------|---------|
| **Circuit Breaker** | Interrompe chamadas a dependências com falha para evitar degradação em cascata | [01-circuit-breaker.md](01-circuit-breaker.md) |
| **Retry com Backoff + Jitter** | Reexecuta chamadas que falharam de forma transitória com intervalos crescentes e aleatórios | [02-retry.md](02-retry.md) |
| **Timeout** | Define tempo máximo para uma chamada, evitando bloqueio indefinido de recursos | [03-timeout.md](03-timeout.md) |
| **Rate Limiter** | Limita a quantidade de chamadas em uma janela de tempo para proteger contra sobrecarga | [04-rate-limiter.md](04-rate-limiter.md) |
| **Bulkhead** | Isola recursos por dependência para que uma falha não afete o sistema inteiro | [05-bulkhead.md](05-bulkhead.md) |
| **Composição de Resiliência** | Combina múltiplos padrões de resiliência em ordem adequada | [06-composicao-resiliencia.md](06-composicao-resiliencia.md) |

---

## 2. Persistência e Processamento de Dados

> Padrões para garantir consistência, idempotência e processamento confiável em arquiteturas distribuídas.

| Padrão | Descrição | Arquivo |
|--------|-----------|---------|
| **Idempotência** | Garante que processar a mesma operação N vezes produz o mesmo resultado | [07-idempotencia.md](07-idempotencia.md) |
| **DLQ (Dead Letter Queue)** | Direciona mensagens que falharam repetidamente para análise e reprocessamento | [08-dlq.md](08-dlq.md) |
| **Saga Pattern** | Coordena transações distribuídas por compensação (orquestração ou coreografia) | [09-saga.md](09-saga.md) |
| **API Composition** | Agrega dados de múltiplos serviços em uma única resposta | [10-api-composition.md](10-api-composition.md) |
| **CQRS** | Separa modelos de leitura e escrita para otimização independente | [11-cqrs.md](11-cqrs.md) |

---

## 3. Event-Driven e Escalabilidade

> Padrões para comunicação assíncrona, processamento de eventos e escalabilidade horizontal.

| Padrão | Descrição | Arquivo |
|--------|-----------|---------|
| **Event Sourcing** | Armazena todos os eventos que modificaram o estado em vez do estado final | [12-event-sourcing.md](12-event-sourcing.md) |
| **Outbox Pattern** | Garante consistência entre banco de dados e sistema de mensageria | [13-outbox-pattern.md](13-outbox-pattern.md) |
| **CDC (Change Data Capture)** | Captura mudanças no banco via log de transações e publica como eventos | [14-cdc.md](14-cdc.md) |
| **Sharding e Partitioning** | Distribui dados entre múltiplas partições ou instâncias de banco | [15-sharding-partitioning.md](15-sharding-partitioning.md) |
| **Backpressure** | Controla a taxa de consumo quando o produtor é mais rápido que o consumidor | [16-backpressure.md](16-backpressure.md) |

---

## 4. Infraestrutura e Deploy

> Padrões para service discovery, configuração, health checks, gateway e sidecars.

| Padrão | Descrição | Arquivo |
|--------|-----------|---------|
| **Service Discovery** | Registro e localização dinâmica de serviços em ambientes distribuídos | [17-service-discovery.md](17-service-discovery.md) |
| **Configuração Externa** | Externaliza configurações para que não sejam acopladas ao código | [18-configuracao-externa.md](18-configuracao-externa.md) |
| **Health Checks** | Endpoints padronizados que reportam saúde do serviço e dependências | [19-health-checks.md](19-health-checks.md) |
| **API Gateway / BFF** | Centraliza cross-cutting concerns e adapta APIs para diferentes clientes | [20-api-gateway-bff.md](20-api-gateway-bff.md) |
| **Sidecar Pattern** | Processo auxiliar que adiciona funcionalidades sem modificar a aplicação | [21-sidecar.md](21-sidecar.md) |

---

## 5. Estratégias de Modernização e Deploy

> Padrões para migração gradual de sistemas legados e deploys seguros.

| Padrão | Descrição | Arquivo |
|--------|-----------|---------|
| **Strangler Fig** | Substitui gradualmente um monolito construindo o novo sistema ao redor do legado | [22-strangler-fig.md](22-strangler-fig.md) |
| **Blue-Green Deployment** | Dois ambientes idênticos com chaveamento instantâneo de tráfego | [23-blue-green.md](23-blue-green.md) |
| **Canary Release** | Envia pequena porcentagem de tráfego para a nova versão antes do rollout completo | [24-canary-release.md](24-canary-release.md) |
| **Feature Flags** | Separa deploy de release; ativa funcionalidades gradualmente sem novo deploy | [25-feature-flags.md](25-feature-flags.md) |
| **Shadow Deployment** | Valida nova versão com tráfego real de produção sem impactar usuários | [26-shadow-deployment.md](26-shadow-deployment.md) |

---

## 6. Observabilidade

> Monitoramento, alertas e dashboards para validar que os padrões estão funcionando.

| Padrão | Descrição | Arquivo |
|--------|-----------|---------|
| **Observabilidade Aplicada** | Pilares (métricas, logs, traces), alertas e dashboards para microsserviços | [27-observabilidade.md](27-observabilidade.md) |

---

## Referência Rápida — Quando usar cada padrão

| Padrão | Use quando... | Não use quando... |
|--------|---------------|-------------------|
| **[Circuit Breaker](01-circuit-breaker.md)** | Dependência externa pode ficar indisponível por períodos | Dependência é local/in-process |
| **[Retry](02-retry.md)** | Falhas são transitórias (timeout, 503) | A falha é determinística (400, 404) |
| **[Timeout](03-timeout.md)** | Chamadas podem demorar indefinidamente | O tempo de resposta é previsível e curto |
| **[Rate Limiter](04-rate-limiter.md)** | Proteger contra sobrecarga ou abuso | Tráfego é estável e controlado |
| **[Bulkhead](05-bulkhead.md)** | Uma dependência lenta pode afetar outras funcionalidades | Todas as dependências têm latência similar |
| **[Idempotência](07-idempotencia.md)** | Retries, duplicação de mensagens, at-least-once delivery | Operações são naturalmente idempotentes (GET, DELETE) |
| **[DLQ](08-dlq.md)** | Mensagens podem falhar e precisam ser reprocessadas | Mensagens descartáveis (métricas, logs) |
| **[Saga](09-saga.md)** | Transação envolve múltiplos serviços | A operação é local a um serviço |
| **[API Composition](10-api-composition.md)** | Client precisa dados de múltiplos serviços agregados | Dados vêm de um único serviço |
| **[CQRS](11-cqrs.md)** | Leitura e escrita têm volumes ou modelos muito diferentes | CRUD simples com volume baixo |
| **[Event Sourcing](12-event-sourcing.md)** | Precisa de auditoria completa e reconstrução de estado | Dados são simples e não requerem histórico |
| **[Outbox Pattern](13-outbox-pattern.md)** | Precisa garantir consistência entre banco e mensageria | Eventual inconsistency é aceitável |
| **[CDC](14-cdc.md)** | Sincronizar dados entre serviços sem acoplamento | Volume de mudanças é muito baixo |
| **[Sharding](15-sharding-partitioning.md)** | Volume de dados excede capacidade de um banco | Dados cabem confortavelmente em um banco |
| **[Backpressure](16-backpressure.md)** | Producer pode ser mais rápido que consumer | Throughput é estável e equilibrado |
| **[Service Discovery](17-service-discovery.md)** | Endpoints mudam dinamicamente (containers, auto-scaling) | Infraestrutura é estática e poucas instâncias |
| **[Config Externa](18-configuracao-externa.md)** | Configs variam por ambiente e precisam mudar sem deploy | Configuração é fixa e simples |
| **[Health Checks](19-health-checks.md)** | Sempre — em qualquer microsserviço | Nunca (sempre use) |
| **[API Gateway](20-api-gateway-bff.md)** | Múltiplos serviços expostos externamente | Monolito com uma única API |
| **[BFF](20-api-gateway-bff.md)** | Frontends diferentes precisam de APIs distintas | Um único tipo de client |
| **[Sidecar](21-sidecar.md)** | Cross-cutting concerns em ambiente containerizado | Aplicação monolítica fora de containers |
| **[Strangler Fig](22-strangler-fig.md)** | Migração gradual de monolito para microsserviços | Sistema é greenfield (novo) |
| **[Blue-Green](23-blue-green.md)** | Precisa de zero-downtime deploy com rollback instantâneo | Infra não comporta dois ambientes completos |
| **[Canary Release](24-canary-release.md)** | Quer validar com tráfego real antes de 100% rollout | Não tem métricas para avaliar o canary |
| **[Feature Flags](25-feature-flags.md)** | Separar deploy de release; kill switch; A/B testing | Funcionalidade é trivial e de baixo risco |
| **[Shadow Deployment](26-shadow-deployment.md)** | Validar nova versão com tráfego real sem impacto | Testar write operations com side effects |
| **[Observabilidade](27-observabilidade.md)** | Sempre — em qualquer microsserviço | Nunca (sempre implemente) |
