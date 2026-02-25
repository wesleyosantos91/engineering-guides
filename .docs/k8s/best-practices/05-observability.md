# Observability — Best Practices

## 1. Os Três Pilares

| Pilar | Descrição | Ferramentas |
|-------|-----------|-------------|
| **Logging** | Registro de eventos | Fluentbit, Loki, ELK |
| **Metrics** | Dados numéricos ao longo do tempo | Prometheus, Grafana, Datadog |
| **Tracing** | Rastreamento de requests distribuídos | Jaeger, Tempo, Zipkin |

## 2. Logging

### 2.1 Structured Logging
```json
{
  "timestamp": "2026-02-24T10:30:00Z",
  "level": "ERROR",
  "message": "Failed to process order",
  "service": "order-api",
  "trace_id": "abc123",
  "span_id": "def456",
  "order_id": "ORD-789",
  "error": "connection timeout",
  "duration_ms": 5023
}
```

**Regras de Logging:**
- **Sempre use structured logging** (JSON).
- Inclua **trace_id e span_id** para correlação.
- Log para **stdout/stderr** (nunca para arquivos no container).
- Use **log levels** consistentes: DEBUG, INFO, WARN, ERROR.
- **Não logue dados sensíveis** (PII, tokens, senhas).

### 2.2 Stack de Logging
```
┌─────────┐    ┌───────────┐    ┌──────────┐    ┌──────────┐
│  Pods   │───▶│ FluentBit │───▶│   Loki   │───▶│ Grafana  │
│ (stdout)│    │ (DaemonSet)│    │ (Storage)│    │  (UI)    │
└─────────┘    └───────────┘    └──────────┘    └──────────┘
```

### 2.3 FluentBit DaemonSet
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    spec:
      serviceAccountName: fluent-bit
      tolerations:
        - operator: Exists  # rodar em todos os nodes
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:3.0
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
```

### 2.4 Boas Práticas de Logging
- Use **FluentBit** (leve) como coletor, não Fluentd (pesado).
- Configure **buffer e retry** para evitar perda de logs.
- **Filtre e enriqueça** logs no FluentBit (adicione namespace, pod, node labels).
- Use **Loki** para logs cost-effective (indexa labels, não conteúdo).
- Configure **retention policies** (7d staging, 30d prod, 90d compliance).
- Use **log sampling** para reduzir volume em alta escala.

## 3. Metrics

### 3.1 Prometheus Stack
```yaml
# Usando kube-prometheus-stack (Helm)
# helm install prometheus prometheus-community/kube-prometheus-stack

# ServiceMonitor — para coletar métricas da sua aplicação
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-metrics
  labels:
    release: prometheus  # deve bater com o label do Prometheus
spec:
  selector:
    matchLabels:
      app: api
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
      scrapeTimeout: 10s
  namespaceSelector:
    matchNames:
      - production
```

### 3.2 Métricas Essenciais (USE/RED)

**USE Method (Infraestrutura):**
| Métrica | Descrição |
|---------|-----------|
| **U**tilization | % de uso do recurso |
| **S**aturation | Fila de trabalho pendente |
| **E**rrors | Taxa de erros |

**RED Method (Serviços):**
| Métrica | Descrição |
|---------|-----------|
| **R**ate | Requests por segundo |
| **E**rrors | Taxa de erros |
| **D**uration | Latência (histograma) |

### 3.3 Métricas da Aplicação
```go
// Exemplo em Go com prometheus/client_golang
var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )
    
    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
        },
        []string{"method", "path"},
    )
)
```

### 3.4 Métricas do Kubernetes
| Componente | Métricas Importantes |
|-----------|---------------------|
| **Nodes** | cpu_usage, memory_usage, disk_pressure, network |
| **Pods** | restart_count, cpu/mem usage vs requests/limits |
| **Containers** | OOMKilled, CrashLoopBackOff, throttling |
| **API Server** | request_latency, request_count, error_rate |
| **etcd** | leader_changes, db_size, fsync_duration |
| **Scheduler** | scheduling_latency, pending_pods |
| **CoreDNS** | request_count, cache_hits, latency |

### 3.5 Alerting Rules
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: critical-alerts
spec:
  groups:
    - name: kubernetes
      rules:
        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total[15m]) * 60 * 5 > 0
          for: 15m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"

        - alert: HighMemoryUsage
          expr: |
            container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Container {{ $labels.container }} using >90% memory limit"

        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m])) 
            / sum(rate(http_requests_total[5m])) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Error rate >5% for {{ $labels.service }}"

        - alert: HighLatency
          expr: |
            histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) 
            by (le, service)) > 2
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "P99 latency >2s for {{ $labels.service }}"
```

## 4. Tracing

### 4.1 OpenTelemetry (Padrão da Indústria)
```yaml
# OpenTelemetry Collector
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
spec:
  mode: daemonset
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      batch:
        timeout: 5s
        send_batch_size: 1024
      memory_limiter:
        check_interval: 1s
        limit_mib: 512
    exporters:
      otlp:
        endpoint: tempo.monitoring:4317
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp]
```

### 4.2 Instrumentação Automática
```yaml
# OpenTelemetry Auto-Instrumentation
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: auto-instrumentation
spec:
  exporter:
    endpoint: http://otel-collector:4317
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "0.1"  # 10% sampling
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:latest
---
# Anotar o Pod para auto-instrumentação
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-java: "true"
        # ou inject-python, inject-nodejs, inject-dotnet
```

### 4.3 Sampling Strategies
| Estratégia | Descrição | Quando Usar |
|-----------|-----------|-------------|
| **Head-based** | Decide no início do trace | Simples, previsível |
| **Tail-based** | Decide após trace completo | Captura erros e latência alta |
| **Rate limiting** | N traces por segundo | Alto throughput |
| **Always on** | 100% sampling | Dev/staging |

## 5. Dashboards (Grafana)

### 5.1 Dashboards Essenciais
| Dashboard | Descrição |
|-----------|-----------|
| **Cluster Overview** | CPU, Memory, Pods, Nodes |
| **Node Exporter** | Métricas detalhadas de nodes |
| **Namespace Overview** | Resources por namespace |
| **Pod Details** | CPU, Memory, Network, Restarts |
| **API Server** | Latência, throughput, errors |
| **CoreDNS** | DNS queries, cache, errors |
| **Application RED** | Rate, Errors, Duration por serviço |

### 5.2 SLI/SLO Dashboards
```
SLI (Service Level Indicator):
  - Availability: % requests com status 2xx/3xx
  - Latency: P99 response time < 500ms
  - Throughput: requests/s

SLO (Service Level Objective):
  - Availability: 99.9% (8.76h downtime/ano)
  - Latency: P99 < 500ms para 99% do tempo
  
Error Budget:
  - 100% - 99.9% = 0.1% error budget
  - 0.1% de 30 dias = 43.2 minutos
```

## 6. Stack Recomendada

### Open Source
```
Métricas:  Prometheus + Thanos/Mimir (long-term storage) + Grafana
Logs:      FluentBit + Loki + Grafana
Traces:    OpenTelemetry + Tempo + Grafana
Alerting:  Alertmanager + PagerDuty/OpsGenie
```

### Comercial
```
Datadog:     All-in-one (metrics, logs, traces, APM)
New Relic:   All-in-one com foco em APM
Dynatrace:   Auto-instrumentation, AI ops
Elastic:     ELK Stack + APM
```

## 7. Boas Práticas Gerais

- Use **OpenTelemetry** como padrão de instrumentação (vendor-neutral).
- Implemente **correlation IDs** (trace_id) em todos os serviços.
- Configure **alertas actionáveis** — cada alerta deve ter runbook.
- Dashboards devem responder: **"O sistema está saudável?"**
- Use **Grafana como single pane of glass** (metrics + logs + traces).
- Implemente **SLOs** e monitore **error budgets**.
- **Cost-optimize**: sampling, retention policies, aggregation.
- Separe **observability stack** em namespace/cluster dedicado.
