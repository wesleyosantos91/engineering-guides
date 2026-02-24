# Microservice Patterns

> **Objetivo:** Referência conceitual sobre **padrões de microsserviços**.
> Abrange resiliência, persistência, event-driven, infraestrutura, deploy e observabilidade.
> **Agnóstico de linguagem e framework** — foca em conceitos, princípios e decisões arquiteturais.

---

## Como usar este guia

Este repositório é uma **base de conhecimento** para assistentes de IA (Copilot, etc.) e engenheiros.

- **Cada arquivo** na pasta `patterns/` é autossuficiente: contém problema, solução, conceitos, pseudocódigo, antipadrões e boas práticas.
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
| **Circuit Breaker** | Interrompe chamadas a dependências com falha para evitar degradação em cascata | [circuit-breaker.md](patterns/circuit-breaker.md) |
| **Retry com Backoff + Jitter** | Reexecuta chamadas que falharam de forma transitória com intervalos crescentes e aleatórios | [retry.md](patterns/retry.md) |
| **Timeout** | Define tempo máximo para uma chamada, evitando bloqueio indefinido de recursos | [timeout.md](patterns/timeout.md) |
| **Rate Limiter** | Limita a quantidade de chamadas em uma janela de tempo para proteger contra sobrecarga | [rate-limiter.md](patterns/rate-limiter.md) |
| **Bulkhead** | Isola recursos por dependência para que uma falha não afete o sistema inteiro | [bulkhead.md](patterns/bulkhead.md) |
| **Composição de Resiliência** | Combina múltiplos padrões de resiliência em ordem adequada | [composicao-resiliencia.md](patterns/composicao-resiliencia.md) |

---

## 2. Persistência e Processamento de Dados

> Padrões para garantir consistência, idempotência e processamento confiável em arquiteturas distribuídas.

| Padrão | Descrição | Arquivo |
|--------|-----------|---------|
| **Idempotência** | Garante que processar a mesma operação N vezes produz o mesmo resultado | [idempotencia.md](patterns/idempotencia.md) |
| **DLQ (Dead Letter Queue)** | Direciona mensagens que falharam repetidamente para análise e reprocessamento | [dlq.md](patterns/dlq.md) |
| **Saga Pattern** | Coordena transações distribuídas por compensação (orquestração ou coreografia) | [saga.md](patterns/saga.md) |
| **API Composition** | Agrega dados de múltiplos serviços em uma única resposta | [api-composition.md](patterns/api-composition.md) |
| **CQRS** | Separa modelos de leitura e escrita para otimização independente | [cqrs.md](patterns/cqrs.md) |

---

## 3. Event-Driven e Escalabilidade

> Padrões para comunicação assíncrona, processamento de eventos e escalabilidade horizontal.

| Padrão | Descrição | Arquivo |
|--------|-----------|---------|
| **Event Sourcing** | Armazena todos os eventos que modificaram o estado em vez do estado final | [event-sourcing.md](patterns/event-sourcing.md) |
| **Outbox Pattern** | Garante consistência entre banco de dados e sistema de mensageria | [outbox-pattern.md](patterns/outbox-pattern.md) |
| **CDC (Change Data Capture)** | Captura mudanças no banco via log de transações e publica como eventos | [cdc.md](patterns/cdc.md) |
| **Sharding e Partitioning** | Distribui dados entre múltiplas partições ou instâncias de banco | [sharding-partitioning.md](patterns/sharding-partitioning.md) |
| **Backpressure** | Controla a taxa de consumo quando o produtor é mais rápido que o consumidor | [backpressure.md](patterns/backpressure.md) |

---

## 4. Infraestrutura e Deploy

> Padrões para service discovery, configuração, health checks, gateway e sidecars.

| Padrão | Descrição | Arquivo |
|--------|-----------|---------|
| **Service Discovery** | Registro e localização dinâmica de serviços em ambientes distribuídos | [service-discovery.md](patterns/service-discovery.md) |
| **Configuração Externa** | Externaliza configurações para que não sejam acopladas ao código | [configuracao-externa.md](patterns/configuracao-externa.md) |
| **Health Checks** | Endpoints padronizados que reportam saúde do serviço e dependências | [health-checks.md](patterns/health-checks.md) |
| **API Gateway / BFF** | Centraliza cross-cutting concerns e adapta APIs para diferentes clientes | [api-gateway-bff.md](patterns/api-gateway-bff.md) |
| **Sidecar Pattern** | Processo auxiliar que adiciona funcionalidades sem modificar a aplicação | [sidecar.md](patterns/sidecar.md) |

---

## 5. Estratégias de Modernização e Deploy

> Padrões para migração gradual de sistemas legados e deploys seguros.

| Padrão | Descrição | Arquivo |
|--------|-----------|---------|
| **Strangler Fig** | Substitui gradualmente um monolito construindo o novo sistema ao redor do legado | [strangler-fig.md](patterns/strangler-fig.md) |
| **Blue-Green Deployment** | Dois ambientes idênticos com chaveamento instantâneo de tráfego | [blue-green.md](patterns/blue-green.md) |
| **Canary Release** | Envia pequena porcentagem de tráfego para a nova versão antes do rollout completo | [canary-release.md](patterns/canary-release.md) |
| **Feature Flags** | Separa deploy de release; ativa funcionalidades gradualmente sem novo deploy | [feature-flags.md](patterns/feature-flags.md) |
| **Shadow Deployment** | Valida nova versão com tráfego real de produção sem impactar usuários | [shadow-deployment.md](patterns/shadow-deployment.md) |

---

## 6. Observabilidade

> Monitoramento, alertas e dashboards para validar que os padrões estão funcionando.

| Padrão | Descrição | Arquivo |
|--------|-----------|---------|
| **Observabilidade Aplicada** | Pilares (métricas, logs, traces), alertas e dashboards para microsserviços | [observabilidade.md](patterns/observabilidade.md) |

---

## Referência Rápida — Quando usar cada padrão

| Padrão | Use quando... | Não use quando... |
|--------|---------------|-------------------|
| **[Circuit Breaker](patterns/circuit-breaker.md)** | Dependência externa pode ficar indisponível por períodos | Dependência é local/in-process |
| **[Retry](patterns/retry.md)** | Falhas são transitórias (timeout, 503) | A falha é determinística (400, 404) |
| **[Timeout](patterns/timeout.md)** | Chamadas podem demorar indefinidamente | O tempo de resposta é previsível e curto |
| **[Rate Limiter](patterns/rate-limiter.md)** | Proteger contra sobrecarga ou abuso | Tráfego é estável e controlado |
| **[Bulkhead](patterns/bulkhead.md)** | Uma dependência lenta pode afetar outras funcionalidades | Todas as dependências têm latência similar |
| **[Idempotência](patterns/idempotencia.md)** | Retries, duplicação de mensagens, at-least-once delivery | Operações são naturalmente idempotentes (GET, DELETE) |
| **[DLQ](patterns/dlq.md)** | Mensagens podem falhar e precisam ser reprocessadas | Mensagens descartáveis (métricas, logs) |
| **[Saga](patterns/saga.md)** | Transação envolve múltiplos serviços | A operação é local a um serviço |
| **[API Composition](patterns/api-composition.md)** | Client precisa dados de múltiplos serviços agregados | Dados vêm de um único serviço |
| **[CQRS](patterns/cqrs.md)** | Leitura e escrita têm volumes ou modelos muito diferentes | CRUD simples com volume baixo |
| **[Event Sourcing](patterns/event-sourcing.md)** | Precisa de auditoria completa e reconstrução de estado | Dados são simples e não requerem histórico |
| **[Outbox Pattern](patterns/outbox-pattern.md)** | Precisa garantir consistência entre banco e mensageria | Eventual inconsistency é aceitável |
| **[CDC](patterns/cdc.md)** | Sincronizar dados entre serviços sem acoplamento | Volume de mudanças é muito baixo |
| **[Sharding](patterns/sharding-partitioning.md)** | Volume de dados excede capacidade de um banco | Dados cabem confortavelmente em um banco |
| **[Backpressure](patterns/backpressure.md)** | Producer pode ser mais rápido que consumer | Throughput é estável e equilibrado |
| **[Service Discovery](patterns/service-discovery.md)** | Endpoints mudam dinamicamente (containers, auto-scaling) | Infraestrutura é estática e poucas instâncias |
| **[Config Externa](patterns/configuracao-externa.md)** | Configs variam por ambiente e precisam mudar sem deploy | Configuração é fixa e simples |
| **[Health Checks](patterns/health-checks.md)** | Sempre — em qualquer microsserviço | Nunca (sempre use) |
| **[API Gateway](patterns/api-gateway-bff.md)** | Múltiplos serviços expostos externamente | Monolito com uma única API |
| **[BFF](patterns/api-gateway-bff.md)** | Frontends diferentes precisam de APIs distintas | Um único tipo de client |
| **[Sidecar](patterns/sidecar.md)** | Cross-cutting concerns em ambiente containerizado | Aplicação monolítica fora de containers |
| **[Strangler Fig](patterns/strangler-fig.md)** | Migração gradual de monolito para microsserviços | Sistema é greenfield (novo) |
| **[Blue-Green](patterns/blue-green.md)** | Precisa de zero-downtime deploy com rollback instantâneo | Infra não comporta dois ambientes completos |
| **[Canary Release](patterns/canary-release.md)** | Quer validar com tráfego real antes de 100% rollout | Não tem métricas para avaliar o canary |
| **[Feature Flags](patterns/feature-flags.md)** | Separar deploy de release; kill switch; A/B testing | Funcionalidade é trivial e de baixo risco |
| **[Shadow Deployment](patterns/shadow-deployment.md)** | Validar nova versão com tráfego real sem impacto | Testar write operations com side effects |
| **[Observabilidade](patterns/observabilidade.md)** | Sempre — em qualquer microsserviço | Nunca (sempre implemente) |
