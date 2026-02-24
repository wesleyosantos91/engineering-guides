# Observabilidade & Qualidade — Guia de Monitoramento para Quality Engineering

> **Versão:** 1.0
> **Última atualização:** 2026-02-24
> **Público-alvo:** Engenheiros de software, SREs, líderes técnicos
> **Documentos relacionados:** [Testing Strategy](testing-strategy.md) · [Performance Testing](performance-testing.md) · [Security Testing](security-testing.md)

---

## Sumário

1. [Visão Geral](#1-visão-geral)
2. [Três Pilares da Observabilidade](#2-três-pilares-da-observabilidade)
3. [SLIs, SLOs e SLAs](#3-slis-slos-e-slas)
4. [Logging — Boas Práticas](#4-logging--boas-práticas)
5. [Métricas — O que Medir](#5-métricas--o-que-medir)
6. [Distributed Tracing](#6-distributed-tracing)
7. [Alerting — Estratégia de Alertas](#7-alerting--estratégia-de-alertas)
8. [Dashboards — Princípios de Design](#8-dashboards--princípios-de-design)
9. [Testes de Observabilidade](#9-testes-de-observabilidade)
10. [Observabilidade em CI/CD](#10-observabilidade-em-cicd)
11. [Anti-Patterns](#11-anti-patterns)
12. [Glossário](#12-glossário)

---

## 1. Visão Geral

### 1.1 Por que Observabilidade é parte de Quality Engineering?

Qualidade não termina no merge. Um sistema que não pode ser observado em produção é um sistema cuja qualidade **não pode ser verificada**.

| Aspecto                        | Testes (pré-produção)                    | Observabilidade (pós-deploy)             |
|-------------------------------|------------------------------------------|------------------------------------------|
| **Quando**                    | Antes do merge / no pipeline             | Após deploy, continuamente               |
| **O que detecta**             | Bugs conhecidos, regressões              | Bugs desconhecidos, degradações          |
| **Feedback loop**             | Minutos (CI)                             | Segundos a minutos (alertas)             |
| **Cobertura**                 | Cenários pré-definidos                   | Comportamento real do sistema            |
| **Complementaridade**         | Previne defeitos conhecidos              | Detecta defeitos desconhecidos           |

### 1.2 Maturidade de Observabilidade

| Nível | Maturidade          | Características                                                       |
|-------|---------------------|-----------------------------------------------------------------------|
| 0     | **Nenhuma**         | Sem logs estruturados, sem métricas, "funciona na minha máquina"      |
| 1     | **Reativa**         | Logs centralizados, alertas básicos, investigação manual              |
| 2     | **Proativa**        | SLIs/SLOs definidos, dashboards por serviço, alertas baseados em SLO  |
| 3     | **Avançada**        | Distributed tracing, correlation IDs, anomaly detection               |
| 4     | **Observability-driven** | Feature flags com métricas, canary analysis, auto-rollback       |

---

## 2. Três Pilares da Observabilidade

```
┌────────────────────────────────────────────────────────────┐
│                   OBSERVABILIDADE                          │
│                                                            │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  LOGS    │    │  MÉTRICAS    │    │  TRACES          │  │
│  │          │    │              │    │                  │  │
│  │ O que    │    │ Quanto /     │    │ Onde / como      │  │
│  │ aconteceu│    │ com que freq │    │ uma requisição   │  │
│  │          │    │              │    │ fluiu            │  │
│  │ Detalhado│    │ Agregado     │    │ Distribuído      │  │
│  │ Alto vol.│    │ Baixo custo  │    │ Custo médio      │  │
│  └──────────┘    └──────────────┘    └──────────────────┘  │
│                                                            │
│  Complementados por: Events, Profiles, Continuous Testing  │
└────────────────────────────────────────────────────────────┘
```

| Pilar       | Ferramenta(s) Comuns                                    | Formato                        |
|-------------|--------------------------------------------------------|--------------------------------|
| **Logs**    | ELK Stack, Loki + Grafana, CloudWatch Logs, Datadog    | JSON estruturado               |
| **Métricas**| Prometheus + Grafana, CloudWatch Metrics, Datadog      | Dimensional (labels/tags)      |
| **Traces**  | Jaeger, Zipkin, AWS X-Ray, Datadog APM, Tempo          | OpenTelemetry (OTLP)           |

---

## 3. SLIs, SLOs e SLAs

### 3.1 Definições

| Conceito | Definição                                                              | Exemplo                                      |
|----------|------------------------------------------------------------------------|----------------------------------------------|
| **SLI**  | Service Level Indicator — métrica que mede a qualidade do serviço      | Latência p99 das requests                    |
| **SLO**  | Service Level Objective — alvo interno para um SLI                     | p99 latência ≤ 200ms em 99.5% do tempo       |
| **SLA**  | Service Level Agreement — compromisso contratual com o cliente         | Disponibilidade ≥ 99.9% mensal               |
| **Error Budget** | 100% - SLO target. Quanto de "falha" é aceitável                | 99.9% SLO → 0.1% error budget → ~43 min/mês |

### 3.2 SLIs Recomendados por Tipo de Serviço

| Tipo de Serviço | SLIs Recomendados                                                          |
|-----------------|----------------------------------------------------------------------------|
| **API / HTTP**  | Availability, Latency (p50, p95, p99), Error rate (5xx), Throughput (rps)  |
| **Pipeline/ETL**| Freshness (idade dos dados), Correctness, Throughput (records/s)           |
| **Messaging**   | Publish latency, End-to-end latency, Message loss rate, Consumer lag       |
| **Storage**     | Availability, Durability, Read/Write latency                               |
| **Front-end**   | LCP, FID/INP, CLS (Core Web Vitals), Error rate JS, TTI                   |

### 3.3 Template de SLO

```yaml
# slo-definition.yaml
service: order-service
owner: team-checkout
slos:
  - name: availability
    description: "Proporção de requests bem-sucedidas"
    sli:
      type: availability
      good_events: "http_status < 500"
      total_events: "all http requests"
    target: 99.9
    window: 30d
    
  - name: latency
    description: "Latência p99 do endpoint de criação de pedido"
    sli:
      type: latency
      threshold_ms: 500
      percentile: 99
    target: 99.5
    window: 30d
    
  - name: correctness
    description: "Proporção de pedidos processados corretamente"
    sli:
      type: correctness
      good_events: "order_status != ERROR"
      total_events: "all orders created"
    target: 99.99
    window: 30d
```

### 3.4 Error Budget — Como Usar

| Error Budget Status | Ação                                                                  |
|--------------------|-----------------------------------------------------------------------|
| > 50% restante     | Desenvolvimento normal, features e experimentos                       |
| 25-50% restante    | Aumentar cautela; reviews mais rigorosos; reduzir mudanças arriscadas |
| < 25% restante     | Foco em estabilidade; só bug fixes e melhorias de confiabilidade      |
| 0% (esgotado)      | Feature freeze; foco total em confiabilidade até recuperar o budget   |

---

## 4. Logging — Boas Práticas

### 4.1 Formato Estruturado

**Sempre usar JSON estruturado.** Logs de texto livre dificultam busca e análise.

```json
{
  "timestamp": "2025-01-15T10:30:45.123Z",
  "level": "ERROR",
  "logger": "com.example.order.OrderService",
  "message": "Failed to create order",
  "traceId": "abc123def456",
  "spanId": "span789",
  "correlationId": "req-uuid-001",
  "userId": "user-42",
  "orderId": "order-999",
  "error": {
    "type": "PaymentGatewayException",
    "message": "Connection timeout after 5000ms",
    "stackTrace": "..."
  },
  "context": {
    "paymentMethod": "credit_card",
    "amount": 150.00,
    "currency": "BRL",
    "retryAttempt": 2
  }
}
```

### 4.2 O que Logar em Cada Nível

| Nível    | Quando usar                                                    | Exemplos                                                |
|----------|----------------------------------------------------------------|---------------------------------------------------------|
| **TRACE**| Debug detalhado (desligado em produção normalmente)            | Valores de variáveis internas, flow steps                |
| **DEBUG**| Informações úteis para debugging                                | Parâmetros de entrada, resultados intermediários         |
| **INFO** | Eventos de negócio importantes, ações do sistema                | Pedido criado, pagamento processado, usuário logou      |
| **WARN** | Situação inesperada mas recuperável                             | Retry necessário, cache miss, fallback ativado          |
| **ERROR**| Erro que afeta a operação atual                                 | Exception, falha de integração, timeout                 |
| **FATAL**| Sistema não pode continuar operando                             | Out of memory, configuração inválida, falha de conexão DB |

### 4.3 Regras de Logging

| Regra                                        | Razão                                                     |
|----------------------------------------------|-----------------------------------------------------------|
| **Sempre incluir correlationId/traceId**     | Permite rastrear request end-to-end                       |
| **Nunca logar dados sensíveis**              | PII, senhas, tokens, cartões — mascarar ou omitir         |
| **Logar na fronteira (entrada/saída)**       | Início e fim de operações importantes                     |
| **Incluir contexto de negócio**              | orderId, userId, amount — facilita diagnóstico            |
| **Logar decisões não óbvias**                | "Usando fallback porque serviço X indisponível"           |
| **Não logar em loops de alta frequência**    | Gera volume excessivo, impacta performance                |
| **Usar MDC/context para dados transversais** | Evita passar correlationId manualmente em toda chamada    |

### 4.4 Exemplo — Java (SLF4J + MDC)

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;

public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    
    public Order createOrder(CreateOrderRequest request) {
        MDC.put("correlationId", request.getCorrelationId());
        MDC.put("userId", request.getUserId());
        
        log.info("Creating order: items={}, total={}", 
                 request.getItems().size(), request.getTotal());
        
        try {
            Order order = orderRepository.save(buildOrder(request));
            
            log.info("Order created successfully: orderId={}, status={}", 
                     order.getId(), order.getStatus());
            
            return order;
        } catch (PaymentException e) {
            log.error("Payment failed for order: paymentMethod={}, amount={}", 
                      request.getPaymentMethod(), request.getTotal(), e);
            throw new OrderCreationException("Payment processing failed", e);
        } finally {
            MDC.clear();
        }
    }
}
```

### 4.5 Exemplo — Go (slog structured)

```go
package order

import (
    "context"
    "log/slog"
)

func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    logger := slog.With(
        "correlationId", req.CorrelationID,
        "userId", req.UserID,
    )
    
    logger.InfoContext(ctx, "creating order",
        "items", len(req.Items),
        "total", req.Total,
    )
    
    order, err := s.repo.Save(ctx, buildOrder(req))
    if err != nil {
        logger.ErrorContext(ctx, "failed to create order",
            "error", err,
            "paymentMethod", req.PaymentMethod,
        )
        return nil, fmt.Errorf("create order: %w", err)
    }
    
    logger.InfoContext(ctx, "order created successfully",
        "orderId", order.ID,
        "status", order.Status,
    )
    
    return order, nil
}
```

---

## 5. Métricas — O que Medir

### 5.1 RED Method (para serviços request-driven)

| Métrica          | O que mede                                          | Prometheus Example                             |
|------------------|-----------------------------------------------------|------------------------------------------------|
| **R**ate         | Throughput — requisições por segundo                 | `rate(http_requests_total[5m])`                |
| **E**rrors       | Taxa de erros — requisições com falha                | `rate(http_requests_total{status=~"5.."}[5m])` |
| **D**uration     | Latência — tempo de resposta                         | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` |

### 5.2 USE Method (para recursos/infraestrutura)

| Métrica            | O que mede                                        | Exemplos                                    |
|--------------------|----------------------------------------------------|---------------------------------------------|
| **U**tilization    | % do recurso em uso                                | CPU %, memória %, disco %                   |
| **S**aturation     | Trabalho que não pode ser atendido (fila)          | Queue depth, thread pool waiting            |
| **E**rrors         | Erros do recurso                                    | Disk errors, network drops, OOM kills       |

### 5.3 Four Golden Signals (Google SRE)

| Signal        | Descrição                              | Tipo de Métrica    |
|---------------|----------------------------------------|--------------------|
| **Latency**   | Tempo para processar uma request       | Histogram          |
| **Traffic**   | Volume de demanda                       | Counter (rate)     |
| **Errors**    | Taxa de requests com falha              | Counter (rate)     |
| **Saturation**| Quão "cheio" o serviço está            | Gauge              |

### 5.4 Métricas de Negócio (Business KPIs)

Métricas técnicas não bastam. **Monitorar indicadores de negócio** é essencial:

| Domínio            | Métricas de Negócio                                                |
|--------------------|----------------------------------------------------------------------|
| **E-commerce**     | Pedidos/min, valor médio, taxa de abandono, conversão                |
| **Pagamentos**     | Transações/min, taxa de aprovação, tempo de processamento            |
| **Autenticação**   | Logins/min, taxa de falha, MFA adoption                              |
| **Mensageria**     | Mensagens processadas/min, consumer lag, DLQ growth                  |

### 5.5 Exemplo — Métricas com Micrometer (Java/Spring Boot)

```java
import io.micrometer.core.instrument.*;

@Service
public class OrderService {
    private final Counter ordersCreated;
    private final Counter ordersFailed;
    private final Timer orderCreationTimer;
    private final DistributionSummary orderValue;
    
    public OrderService(MeterRegistry registry) {
        this.ordersCreated = Counter.builder("orders.created.total")
            .description("Total de pedidos criados com sucesso")
            .tag("payment_method", "all")
            .register(registry);
            
        this.ordersFailed = Counter.builder("orders.failed.total")
            .description("Total de pedidos que falharam")
            .tag("reason", "unknown")
            .register(registry);
            
        this.orderCreationTimer = Timer.builder("orders.creation.duration")
            .description("Tempo para criar um pedido")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
            
        this.orderValue = DistributionSummary.builder("orders.value")
            .description("Valor dos pedidos")
            .baseUnit("BRL")
            .publishPercentiles(0.5, 0.95)
            .register(registry);
    }
    
    public Order createOrder(CreateOrderRequest request) {
        return orderCreationTimer.record(() -> {
            try {
                Order order = processOrder(request);
                ordersCreated.increment();
                orderValue.record(order.getTotal().doubleValue());
                return order;
            } catch (Exception e) {
                Metrics.counter("orders.failed.total",
                    "reason", e.getClass().getSimpleName()
                ).increment();
                throw e;
            }
        });
    }
}
```

### 5.6 Exemplo — Métricas com Prometheus (Go)

```go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    OrdersCreated = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "orders_created_total",
        Help: "Total de pedidos criados",
    }, []string{"payment_method", "status"})
    
    OrderDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "order_creation_duration_seconds",
        Help:    "Tempo para criar um pedido",
        Buckets: []float64{0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5},
    }, []string{"status"})
    
    OrderValue = promauto.NewHistogram(prometheus.HistogramOpts{
        Name:    "order_value_brl",
        Help:    "Valor dos pedidos em BRL",
        Buckets: prometheus.ExponentialBuckets(10, 2, 10),
    })
)
```

---

## 6. Distributed Tracing

### 6.1 Conceitos

| Conceito        | Definição                                                                  |
|-----------------|----------------------------------------------------------------------------|
| **Trace**       | Representa o caminho completo de uma requisição pelo sistema               |
| **Span**        | Uma operação individual dentro de um trace (ex.: chamada HTTP, query DB)   |
| **Parent Span** | O span que iniciou a operação atual                                        |
| **Context Propagation** | Mecanismo para passar traceId/spanId entre serviços               |
| **Baggage**     | Metadados propagados junto ao trace context (ex.: userId, tenantId)        |

### 6.2 Fluxo de um Trace

```
[API Gateway]        [Order Service]       [Payment Service]      [Notification]
     │                     │                      │                     │
     ├──── span-1 ────────►│                      │                     │
     │  POST /orders       │                      │                     │
     │                     ├──── span-2 ─────────►│                     │
     │                     │  POST /payments      │                     │
     │                     │                      ├── span-3 ──►[Bank] │
     │                     │                      │  External call      │
     │                     │◄─── response ────────┤                     │
     │                     │                                            │
     │                     ├──────── span-4 ───────────────────────────►│
     │                     │  PUBLISH order.created                     │
     │◄─── response ───────┤                                            │
     │                     │                                            │
     
traceId: abc-123 (propagado em todos os spans)
```

### 6.3 OpenTelemetry — Instrumentação (Java)

```java
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;

@Service
public class OrderService {
    private final Tracer tracer;
    
    public OrderService(Tracer tracer) {
        this.tracer = tracer;
    }
    
    public Order createOrder(CreateOrderRequest request) {
        Span span = tracer.spanBuilder("order.create")
            .setAttribute("order.items.count", request.getItems().size())
            .setAttribute("order.total", request.getTotal().doubleValue())
            .setAttribute("user.id", request.getUserId())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            Order order = processOrder(request);
            
            span.setAttribute("order.id", order.getId());
            span.setAttribute("order.status", order.getStatus().name());
            span.setStatus(StatusCode.OK);
            
            return order;
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### 6.4 Regras de Tracing

| Regra                                           | Razão                                               |
|--------------------------------------------------|------------------------------------------------------|
| **Propagar trace context em TODA comunicação**   | HTTP headers, message headers, gRPC metadata          |
| **Adicionar atributos de negócio aos spans**     | orderId, userId, paymentMethod — facilita busca       |
| **Instrumentar chamadas externas**               | DB, HTTP, messaging — são os gargalos mais comuns     |
| **Definir sampling rate adequado**               | 100% em dev, 1-10% em produção (ajustar conforme custo) |
| **Não criar spans para operações triviais**      | Gets de cache, cálculos simples — geram ruído         |

---

## 7. Alerting — Estratégia de Alertas

### 7.1 Princípios de Alertas

| Princípio                                  | Descrição                                                  |
|--------------------------------------------|------------------------------------------------------------|
| **Alerta = ação necessária**               | Todo alerta deve exigir intervenção humana                 |
| **Baseado em sintomas, não causas**         | Alertar "latência alta" não "CPU alta" (CPU alta sem impacto não é urgente) |
| **Baseado em SLO**                         | Alertar quando error budget está sendo consumido rápido    |
| **Sem alert fatigue**                      | Se > 50% dos alertas são ignorados, o sistema está errado  |
| **Runbook obrigatório**                    | Todo alerta deve ter link para procedimento de resolução   |

### 7.2 Severidade de Alertas

| Severidade  | Critério                                             | Resposta Esperada     | Notificação          |
|-------------|------------------------------------------------------|-----------------------|----------------------|
| **P1 / SEV1** | Serviço down, perda de dados, segurança            | Imediata (< 15 min)  | PagerDuty/Opsgenie + telefone |
| **P2 / SEV2** | Degradação significativa, SLO ameaçado             | < 1 hora              | Slack + PagerDuty    |
| **P3 / SEV3** | Degradação menor, funcionalidade parcialmente afetada | < 4 horas           | Slack                |
| **P4 / SEV4** | Anomalia observada, sem impacto atual               | Próximo dia útil      | Ticket/Jira          |

### 7.3 Exemplo — Alertas Prometheus (AlertManager)

```yaml
groups:
  - name: order-service-slo
    rules:
      # Error rate SLO burn rate alert
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{service="order-service", status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{service="order-service"}[5m]))
          ) > 0.01
        for: 5m
        labels:
          severity: critical
          team: checkout
        annotations:
          summary: "Order Service error rate > 1%"
          description: "Error rate is {{ $value | humanizePercentage }} over last 5 min"
          runbook: "https://wiki.example.com/runbooks/order-service-errors"
          dashboard: "https://grafana.example.com/d/order-service"
      
      # Latency SLO alert
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, 
            sum(rate(http_request_duration_seconds_bucket{service="order-service"}[5m])) by (le)
          ) > 0.5
        for: 10m
        labels:
          severity: warning
          team: checkout
        annotations:
          summary: "Order Service p99 latency > 500ms"
          runbook: "https://wiki.example.com/runbooks/order-service-latency"
      
      # Business metric alert
      - alert: LowOrderRate
        expr: |
          sum(rate(orders_created_total[15m])) < 1
        for: 15m
        labels:
          severity: critical
          team: checkout
        annotations:
          summary: "Quase nenhum pedido nos últimos 15 minutos"
          description: "Rate atual: {{ $value }} orders/sec"
          runbook: "https://wiki.example.com/runbooks/low-order-rate"
```

### 7.4 Multi-Window, Multi-Burn-Rate

Para alertas baseados em SLO, usar a abordagem de múltiplas janelas:

| Alerta                      | Janela curta | Janela longa | Burn rate | Severidade |
|-----------------------------|-------------|-------------|-----------|------------|
| Consumo rápido de budget    | 5 min       | 1 hora      | 14.4x     | Critical   |
| Consumo moderado de budget  | 30 min      | 6 horas     | 6x        | Warning    |
| Consumo lento de budget     | 2 horas     | 24 horas    | 3x        | Ticket     |

---

## 8. Dashboards — Princípios de Design

### 8.1 Hierarquia de Dashboards

```
┌─────────────────────────────────┐
│     Executive / Business        │  ← KPIs de negócio, disponibilidade global
├─────────────────────────────────┤
│     Service Overview            │  ← RED metrics por serviço, SLO status
├─────────────────────────────────┤
│     Service Detail              │  ← Breakdown por endpoint, host, dependência
├─────────────────────────────────┤
│     Infrastructure              │  ← CPU, memória, disco, rede, pods
├─────────────────────────────────┤
│     Debug / Investigation       │  ← Traces, logs correlacionados, flame graphs
└─────────────────────────────────┘
```

### 8.2 Service Overview Dashboard — Template

| Painel                        | Query / Conteúdo                           | Tipo         |
|-------------------------------|--------------------------------------------|--------------|
| **SLO Status**                | Current SLO % vs target                    | Stat/Gauge   |
| **Error Budget Remaining**    | % do error budget restante no mês          | Stat         |
| **Request Rate**              | Rate(requests_total) por status code        | Time series  |
| **Error Rate**                | Rate(5xx) / Rate(total)                     | Time series  |
| **Latency (p50/p95/p99)**     | histogram_quantile por percentile           | Time series  |
| **Throughput vs. Errors**     | Requests e errors sobrepostos               | Time series  |
| **Dependency Health**         | Status de dependências (DB, cache, APIs)    | Status map   |
| **Recent Deploys**            | Annotations de deploy no timeline           | Annotations  |
| **Top 5 Slow Endpoints**      | Endpoints com maior latência p99            | Table        |
| **Top 5 Error Endpoints**     | Endpoints com maior taxa de erro            | Table        |

### 8.3 Boas Práticas de Dashboard

| Prática                                  | Razão                                                    |
|------------------------------------------|----------------------------------------------------------|
| **Usar USE/RED como estrutura**          | Framework comprovado, fácil de entender                  |
| **Marcar deploys como annotations**      | Correlacionar mudanças com degradações                   |
| **Consistência visual entre dashboards** | Mesmas cores e layouts para métricas similares            |
| **Mostrar período de comparação**        | "Hoje vs. semana passada" detecta anomalias              |
| **Incluir links para runbooks**          | De alerta → dashboard → runbook em 1 clique              |
| **Dashboard como código (Jsonnet/Terraform)** | Versionável, replicável, revisável                  |

---

## 9. Testes de Observabilidade

### 9.1 Verificar que Logs São Emitidos

```java
// Java — verificar logs com LogCaptor
import nl.altindag.log.LogCaptor;

@Test
void shouldLogOrderCreation() {
    try (LogCaptor logCaptor = LogCaptor.forClass(OrderService.class)) {
        orderService.createOrder(validRequest());
        
        assertThat(logCaptor.getInfoLogs())
            .anyMatch(log -> log.contains("Order created successfully"));
        
        assertThat(logCaptor.getInfoLogs())
            .anyMatch(log -> log.contains("orderId="));
    }
}

@Test
void shouldNotLogSensitiveData() {
    try (LogCaptor logCaptor = LogCaptor.forClass(OrderService.class)) {
        orderService.createOrder(requestWithCreditCard());
        
        String allLogs = String.join("\n", logCaptor.getLogs());
        
        assertThat(allLogs).doesNotContain("4111111111111111"); // card number
        assertThat(allLogs).doesNotContain(request.getCpf());    // CPF
    }
}
```

### 9.2 Verificar que Métricas São Registradas

```java
// Java — Spring Boot + Micrometer test
import io.micrometer.core.instrument.simple.SimpleMeterRegistry;

@Test
void shouldRecordOrderMetrics() {
    SimpleMeterRegistry registry = new SimpleMeterRegistry();
    OrderService service = new OrderService(registry);
    
    service.createOrder(validRequest());
    
    assertThat(registry.counter("orders.created.total").count())
        .isEqualTo(1.0);
    
    assertThat(registry.timer("orders.creation.duration").count())
        .isEqualTo(1);
}

@Test
void shouldRecordErrorMetrics() {
    SimpleMeterRegistry registry = new SimpleMeterRegistry();
    OrderService service = new OrderService(registry, failingPaymentGateway());
    
    assertThrows(OrderCreationException.class, 
        () -> service.createOrder(validRequest()));
    
    assertThat(registry.counter("orders.failed.total",
        "reason", "PaymentException").count())
        .isEqualTo(1.0);
}
```

### 9.3 Verificar Trace Context Propagation

```java
// Verificar que trace context é propagado entre serviços
@Test
void shouldPropagateTraceContext() {
    // Given
    MockWebServer paymentService = new MockWebServer();
    paymentService.enqueue(new MockResponse().setBody("{}").setResponseCode(200));
    
    // When
    orderService.createOrder(validRequest());
    
    // Then — verificar que header de tracing foi enviado
    RecordedRequest paymentRequest = paymentService.takeRequest();
    assertThat(paymentRequest.getHeader("traceparent")).isNotNull();
    assertThat(paymentRequest.getHeader("traceparent")).startsWith("00-");
}
```

### 9.4 Teste de Health Check

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class HealthCheckTest {
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void healthEndpointShouldReturnUp() {
        ResponseEntity<Map> response = restTemplate
            .getForEntity("/actuator/health", Map.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().get("status")).isEqualTo("UP");
    }
    
    @Test
    void healthEndpointShouldCheckDependencies() {
        ResponseEntity<Map> response = restTemplate
            .getForEntity("/actuator/health", Map.class);
        
        Map<String, Object> components = (Map) response.getBody().get("components");
        assertThat(components).containsKeys("db", "redis", "diskSpace");
    }
}
```

---

## 10. Observabilidade em CI/CD

### 10.1 Deploy Observability Checklist

```markdown
**Pré-deploy:**
- [ ] Métricas de baseline capturadas (latência, error rate, throughput)
- [ ] Alertas de SLO ativados
- [ ] Dashboard do serviço acessível

**Durante deploy:**
- [ ] Deploy annotation criada no Grafana/Datadog
- [ ] Monitorar error rate em tempo real por 15-30 minutos
- [ ] Comparar métricas com baseline

**Pós-deploy (canary/progressive):**
- [ ] Canary vs. baseline: error rate < threshold
- [ ] Canary vs. baseline: latência p99 < threshold
- [ ] Auto-rollback configurado se thresholds violados
- [ ] Logs sem novos tipos de erro
```

### 10.2 Pipeline Metrics

| Métrica do Pipeline               | O que mede                                         | Meta             |
|------------------------------------|----------------------------------------------------|------------------|
| **Deployment frequency**          | Quantas vezes deploy em produção por semana         | ≥ 1/dia (elite) |
| **Lead time for changes**         | Tempo de commit até produção                         | < 1 hora (elite)|
| **Change failure rate**           | % de deploys que causam incidente                    | < 5%             |
| **Mean time to recover (MTTR)**   | Tempo para restaurar serviço após incidente          | < 1 hora         |

> Estas são as **DORA Metrics** — indicadores-chave de performance de entrega de software.

### 10.3 Canary Analysis Automatizada

```yaml
# Exemplo: Argo Rollouts canary com analysis
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: order-service-canary
spec:
  metrics:
    - name: error-rate
      interval: 60s
      successCondition: result < 0.01
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{service="order-service",
                     version="{{args.canary-version}}", status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{service="order-service",
                     version="{{args.canary-version}}"}[5m]))
    
    - name: latency-p99
      interval: 60s
      successCondition: result < 0.5
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{
                service="order-service",
                version="{{args.canary-version}}"
              }[5m])) by (le))
```

---

## 11. Anti-Patterns

| Anti-pattern                              | Problema                                                 | Solução                                                  |
|-------------------------------------------|----------------------------------------------------------|----------------------------------------------------------|
| **Log sem estrutura**                     | Impossível fazer queries; grep manual                    | JSON estruturado com campos padrão                       |
| **Métricas de vaidade**                   | Medir tudo sem propósito; dashboards com 50 painéis      | Focar em RED/USE e métricas de negócio                   |
| **Alertas em tudo**                       | Alert fatigue; time ignora alertas                       | Alertar apenas o que requer ação; usar SLO burn rate     |
| **Sem correlationId**                     | Impossível rastrear request entre serviços               | Propagar traceId/correlationId em todas chamadas         |
| **Logar dados sensíveis**                 | Violação de LGPD/GDPR; risco de vazamento                | Mascarar PII; audit periódico de logs                    |
| **Observabilidade como afterthought**     | Adicionada só após incidente; cobertura parcial          | Incluir no Definition of Done: "é observável?"           |
| **Dashboards sem contexto**               | Painel com gráficos que ninguém entende                  | Título claro, descrição, links para runbooks             |
| **Monitorar apenas infraestrutura**       | Saber que CPU está ok não diz se o negócio funciona      | Monitorar métricas de negócio (pedidos/min, conversão)   |
| **Tracing com 100% sampling em produção** | Custo altíssimo e volume excessivo                       | Sampling 1-10%, head-based ou tail-based sampling        |
| **Sem runbooks nos alertas**              | Alerta dispara e ninguém sabe o que fazer                | Runbook obrigatório para todo alerta com ação            |

---

## 12. Glossário

| Termo                        | Definição                                                                          |
|------------------------------|------------------------------------------------------------------------------------|
| **Burn rate**                | Velocidade de consumo do error budget (1x = consumo linear)                        |
| **Cardinality**              | Número de combinações únicas de labels em métricas (alta = custo alto)             |
| **Context propagation**      | Técnica para passar traceId entre serviços via headers                             |
| **DORA Metrics**             | 4 métricas de DevOps: deploy freq, lead time, change failure rate, MTTR            |
| **Error budget**             | Quantidade de "falha" aceitável (100% - SLO target)                                |
| **Golden signals**           | Latência, tráfego, erros, saturação — as 4 métricas essenciais (Google SRE)        |
| **Head-based sampling**      | Decisão de sample no início do trace (simples, mas pode perder traces com erro)    |
| **MDC**                      | Mapped Diagnostic Context — armazenamento thread-local para dados de log           |
| **MTTR**                     | Mean Time To Recovery — tempo médio para restaurar serviço                         |
| **MTTD**                     | Mean Time To Detect — tempo médio para detectar um problema                        |
| **OpenTelemetry (OTel)**     | Framework unificado para logs, métricas e traces                                   |
| **RED method**               | Rate, Errors, Duration — framework de métricas para serviços                       |
| **Runbook**                  | Procedimento documentado para diagnosticar e resolver um alerta                    |
| **SLI**                      | Service Level Indicator — métrica que mede qualidade do serviço                    |
| **SLO**                      | Service Level Objective — alvo interno para um SLI                                 |
| **SLA**                      | Service Level Agreement — compromisso contratual de nível de serviço               |
| **Tail-based sampling**      | Decisão de sample no fim do trace (captura erros, mas requer buffering)            |
| **Trace**                    | Caminho completo de uma requisição distribuída pelo sistema                        |
| **USE method**               | Utilization, Saturation, Errors — framework de métricas para recursos              |
