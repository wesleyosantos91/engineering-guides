# Level 5 — Observability

> **Objetivo:** Implementar observabilidade completa na plataforma CloudShop — logging centralizado,
> métricas com Prometheus, tracing distribuído e alertas automatizados.

**Referência:** [Observability Best Practices](../../.docs/k8s/05-observability.md)

---

## Objetivo de Aprendizado

- Implementar logging centralizado com FluentBit + Loki
- Configurar métricas com Prometheus + Grafana
- Implementar tracing distribuído com OpenTelemetry + Tempo
- Criar alertas com PrometheusRules e Alertmanager
- Construir dashboards USE/RED para microsserviços
- Implementar health checks e SLIs

---

## Contexto do Domínio

A plataforma CloudShop em produção precisa de visibilidade total: logs estruturados de todos os microsserviços, métricas de negócio e infra, tracing de requisições cross-service, e alertas para detectar degradação antes que impacte o cliente.

---

## Desafios

### Desafio 5.1 — Logging Centralizado

Implemente stack de logging com FluentBit (DaemonSet) coletando logs de todos os Pods.

**Requisitos:**
- FluentBit como DaemonSet em todos os nodes
- Parser para logs JSON dos microsserviços
- Enriquecimento com metadata Kubernetes (namespace, pod, container)
- Output para stdout (ou Loki se disponível)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
  labels:
    app.kubernetes.io/name: fluent-bit
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: fluent-bit
  template:
    metadata:
      labels:
        app.kubernetes.io/name: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      tolerations:
        - operator: Exists          # rodar em todos os nodes
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:3.0
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluent-bit/etc/
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluent-bit-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Parsers_File  parsers.conf

    [INPUT]
        Name              tail
        Path              /var/log/containers/cloudshop-*.log
        Parser            cri
        Tag               kube.*
        Refresh_Interval  5

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Merge_Log           On
        K8S-Logging.Parser  On

    [OUTPUT]
        Name   stdout
        Match  *
        Format json
```

**Critérios de aceite:**
- [ ] FluentBit DaemonSet com 1 Pod por node
- [ ] ServiceAccount com RBAC para ler metadata
- [ ] Logs JSON parseados corretamente
- [ ] Metadata Kubernetes adicionado (namespace, pod)
- [ ] Tolerations para rodar em todos os nodes

---

### Desafio 5.2 — Métricas com Prometheus

Instale Prometheus stack e configure ServiceMonitors para os microsserviços.

**Requisitos:**
- kube-prometheus-stack via Helm
- ServiceMonitor para cada microsserviço
- PodMonitor para DaemonSets
- Métricas USE (Utilization, Saturation, Errors) para infra
- Métricas RED (Rate, Errors, Duration) para microsserviços

```yaml
# ServiceMonitor — order-service
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-service
  namespace: monitoring
  labels:
    release: prometheus  # match com o label do Prometheus Operator
spec:
  namespaceSelector:
    matchNames:
      - cloudshop-prod
  selector:
    matchLabels:
      app.kubernetes.io/name: order-service
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
      scrapeTimeout: 10s
---
# PodMonitor — fluent-bit
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: fluent-bit
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
      - logging
  selector:
    matchLabels:
      app.kubernetes.io/name: fluent-bit
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
```

```bash
# Instalar kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=cloudshop123

# Verificar targets
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090 -n monitoring
# Abrir: http://localhost:9090/targets
```

**Critérios de aceite:**
- [ ] kube-prometheus-stack instalado (Prometheus + Grafana + Alertmanager)
- [ ] ServiceMonitor para cada microsserviço
- [ ] Targets "UP" no Prometheus UI
- [ ] Métricas `container_cpu_usage_seconds_total` disponíveis
- [ ] Grafana acessível com dashboards padrão

---

### Desafio 5.3 — Tracing Distribuído

Configure tracing distribuído para rastrear requisições cross-service.

**Requisitos:**
- Instalar Tempo (ou Jaeger) para armazenar traces
- Instrumentar microsserviços com OpenTelemetry
- Propagar context (trace-id) entre serviços
- Visualizar traces no Grafana

```yaml
# OpenTelemetry Collector (sidecar pattern)
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  config.yaml: |
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
        send_batch_size: 1000
      resource:
        attributes:
          - key: service.namespace
            value: cloudshop
            action: upsert

    exporters:
      otlp/tempo:
        endpoint: tempo.monitoring:4317
        tls:
          insecure: true

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch, resource]
          exporters: [otlp/tempo]
```

```bash
# Instalar Tempo via Helm
helm repo add grafana https://grafana.github.io/helm-charts
helm install tempo grafana/tempo \
  --namespace monitoring

# Configurar Grafana datasource para Tempo
# Settings → Data Sources → Add → Tempo → http://tempo:3100
```

**Critérios de aceite:**
- [ ] Tempo (ou Jaeger) instalado
- [ ] OpenTelemetry Collector configurado
- [ ] Traces visíveis no Grafana
- [ ] Trace propagation funciona cross-service
- [ ] Span de order-service → product-service visível

---

### Desafio 5.4 — Alertas

Configure PrometheusRules e Alertmanager para detectar problemas.

**Requisitos:**
- Alertas para Pods com CrashLoopBackOff
- Alertas para alta utilização CPU/memória
- Alertas para latência P99 > threshold
- Alertas para alta taxa de erros (5xx)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cloudshop-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: cloudshop.pods
      rules:
        - alert: PodCrashLooping
          expr: |
            rate(kube_pod_container_status_restarts_total{namespace=~"cloudshop-.*"}[15m]) > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} está em CrashLoopBackOff"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} reiniciou {{ $value | humanize }} vezes nos últimos 15min"

        - alert: HighMemoryUsage
          expr: |
            container_memory_working_set_bytes{namespace=~"cloudshop-.*"}
            / container_spec_memory_limit_bytes{namespace=~"cloudshop-.*"} > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Container {{ $labels.container }} usando >90% memória"

    - name: cloudshop.slis
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{namespace="cloudshop-prod",status=~"5.."}[5m]))
            / sum(rate(http_requests_total{namespace="cloudshop-prod"}[5m])) > 0.01
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Taxa de erro >1% na CloudShop"

        - alert: HighLatencyP99
          expr: |
            histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{namespace="cloudshop-prod"}[5m])) by (le, service)) > 2
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Latência P99 >2s para {{ $labels.service }}"
```

**Critérios de aceite:**
- [ ] PrometheusRules criadas e carregadas pelo Prometheus
- [ ] Alertas visíveis no Prometheus UI (Alerts tab)
- [ ] Alertmanager recebendo alertas
- [ ] Severidades definidas (critical, warning)
- [ ] Annotations com informações úteis

---

### Desafio 5.5 — Dashboards

Crie dashboards Grafana para visualização da plataforma CloudShop.

**Requisitos:**
- Dashboard de infraestrutura (USE method: nodes, pods)
- Dashboard de microsserviços (RED method: rate, errors, duration)
- Dashboard de negócio (pedidos/min, receita, latência checkout)

```yaml
# Dashboard provisionado via ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudshop-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"  # auto-discovery pelo Grafana sidecar
data:
  cloudshop-overview.json: |
    {
      "dashboard": {
        "title": "CloudShop Overview",
        "panels": [
          {
            "title": "Request Rate (RED - Rate)",
            "type": "timeseries",
            "targets": [{
              "expr": "sum(rate(http_requests_total{namespace='cloudshop-prod'}[5m])) by (service)"
            }]
          },
          {
            "title": "Error Rate (RED - Errors)",
            "type": "stat",
            "targets": [{
              "expr": "sum(rate(http_requests_total{status=~'5..'}[5m])) / sum(rate(http_requests_total[5m]))"
            }]
          },
          {
            "title": "Latency P99 (RED - Duration)",
            "type": "timeseries",
            "targets": [{
              "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{namespace='cloudshop-prod'}[5m])) by (le, service))"
            }]
          }
        ]
      }
    }
```

**Critérios de aceite:**
- [ ] Dashboard de infra (CPU, memória, disco por node)
- [ ] Dashboard RED para microsserviços
- [ ] Dashboard provisionado via ConfigMap (não manual)
- [ ] Grafana sidecar detecta novos dashboards automaticamente
- [ ] Variáveis Grafana para filtrar namespace/service

---

### Desafio 5.6 — Health Checks e SLIs

Defina SLIs (Service Level Indicators) e valide health dos serviços.

**Requisitos:**
- SLI de disponibilidade: % requests com status < 500
- SLI de latência: % requests com latência < 500ms
- Recording rules para SLIs
- Probes de saúde nos endpoints `/health` e `/ready`

```yaml
# Recording rules para SLIs
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cloudshop-slis
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: cloudshop.slis.recording
      interval: 30s
      rules:
        # SLI: Availability
        - record: cloudshop:sli:availability
          expr: |
            sum(rate(http_requests_total{namespace="cloudshop-prod",status!~"5.."}[5m]))
            / sum(rate(http_requests_total{namespace="cloudshop-prod"}[5m]))

        # SLI: Latency (% requests < 500ms)
        - record: cloudshop:sli:latency
          expr: |
            sum(rate(http_request_duration_seconds_bucket{namespace="cloudshop-prod",le="0.5"}[5m]))
            / sum(rate(http_request_duration_seconds_count{namespace="cloudshop-prod"}[5m]))

        # Error budget remaining (SLO 99.9%)
        - record: cloudshop:error_budget:remaining
          expr: |
            1 - ((1 - cloudshop:sli:availability) / (1 - 0.999))
```

**Critérios de aceite:**
- [ ] Recording rules para SLIs criadas
- [ ] SLI de disponibilidade calculado (target: 99.9%)
- [ ] SLI de latência calculado (target: 95% < 500ms)
- [ ] Error budget visível em dashboard
- [ ] Health endpoints verificados via Probes

---

## Definição de Pronto (DoD)

- [ ] FluentBit coletando logs de todos os Pods
- [ ] Prometheus scraping métricas de todos os microsserviços
- [ ] Tracing cross-service funcional
- [ ] Alertas configurados para cenários críticos
- [ ] Dashboards provisionados via ConfigMap
- [ ] SLIs definidos e monitorados
- [ ] Commit: `feat(level-5): add observability stack`

---

## Checklist

- [ ] FluentBit DaemonSet em todos os nodes
- [ ] kube-prometheus-stack instalado
- [ ] ServiceMonitors para todos os microsserviços
- [ ] Tempo/Jaeger para tracing
- [ ] OpenTelemetry Collector configurado
- [ ] PrometheusRules para alertas
- [ ] Alertmanager funcional
- [ ] Dashboards Grafana (infra, RED, negócio)
- [ ] Recording rules para SLIs
- [ ] Error budget monitorado
