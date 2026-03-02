# Level 16 — Kubestronaut (Capstone)

> **Objetivo:** Consolidar todo o conhecimento adquirido nos 16 níveis anteriores,
> validando a plataforma CloudShop como production-grade e demonstrando maestria em
> todas as 5 certificações Kubernetes (KCNA + CKAD + CKA + CKS + KCSA).

**Referência:** [Kubestronaut Guide](../../.docs/k8s/16-kubestronaut.md)

---

## Sobre o Kubestronaut

| Campo | Detalhe |
|-------|---------|
| Requisito | 5 certificações ativas: KCNA, CKAD, CKA, CKS, KCSA |
| Benefícios | Título Kubestronaut, acesso exclusivo à comunidade, jacket CNCF |
| Dificuldade | Manter todas as 5 certificações dentro do prazo de validade |

### Certificações Necessárias

| Certificação | Tipo | Horas estimadas |
|-------------|------|-----------------|
| KCNA | Múltipla escolha | 40h |
| CKAD | Hands-on | 80h |
| CKA | Hands-on | 100h |
| CKS | Hands-on | 80h |
| KCSA | Múltipla escolha | 60h |

---

## Contexto do Domínio

Este capstone integra **todos os conceitos** dos níveis 00-15. A CloudShop Platform
será avaliada como um sistema de produção completo, cobrindo arquitetura, segurança,
observabilidade, CI/CD, alta disponibilidade e compliance.

**Arquitetura Final da CloudShop:**

```
                         ┌──────────────────────────────────────────┐
                         │              Internet                    │
                         └───────────────┬──────────────────────────┘
                                         │
                         ┌───────────────▼──────────────────────────┐
                         │     Ingress Controller (nginx/envoy)     │
                         │     + cert-manager (Let's Encrypt TLS)   │
                         └───────────────┬──────────────────────────┘
                                         │
     ┌───────────────────────────────────┼───────────────────────────────────┐
     │                    cloudshop-prod namespace                          │
     │                                                                       │
     │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
     │   │  order   │  │ product  │  │   user   │  │ payment  │            │
     │   │  svc     │  │  svc     │  │   svc    │  │  svc     │            │
     │   │ (3 pods) │  │ (3 pods) │  │ (3 pods) │  │ (3 pods) │            │
     │   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘            │
     │        │              │              │              │                  │
     │   ┌──────────┐  ┌──────────┐                                         │
     │   │inventory │  │ notific. │  ← async via Kafka                      │
     │   │  svc     │  │  svc     │                                         │
     │   │ (3 pods) │  │ (2 pods) │                                         │
     │   └──────────┘  └──────────┘                                         │
     └───────────────────────────────────────────────────────────────────────┘
                         │
     ┌───────────────────┼───────────────────────────────────────────────────┐
     │            cloudshop-data namespace                                  │
     │                                                                       │
     │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
     │   │  PostgreSQL  │  │    Redis     │  │    Kafka     │              │
     │   │ StatefulSet  │  │ StatefulSet  │  │ StatefulSet  │              │
     │   │  (3 replicas)│  │  (3 replicas)│  │  (3 brokers) │              │
     │   └──────────────┘  └──────────────┘  └──────────────┘              │
     └───────────────────────────────────────────────────────────────────────┘
                         │
     ┌───────────────────┼───────────────────────────────────────────────────┐
     │            monitoring / logging namespaces                           │
     │                                                                       │
     │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
     │   │Prometheus│  │ Grafana  │  │  Tempo   │  │ FluentBit│           │
     │   │ + Alert  │  │          │  │          │  │ DaemonSet│           │
     │   └──────────┘  └──────────┘  └──────────┘  └──────────┘           │
     └───────────────────────────────────────────────────────────────────────┘
```

---

## Desafios

### Desafio 16.1 — Platform Architecture Review (KCNA + CKA)

Valide a arquitetura completa da CloudShop como production-grade.

```bash
# 1. Verificar todos os namespaces existem
kubectl get namespaces | grep -E "cloudshop|monitoring|logging|argocd|ingress"
# Esperado: cloudshop-dev, cloudshop-staging, cloudshop-prod,
#           cloudshop-data, monitoring, logging, argocd, ingress-nginx

# 2. Verificar cluster health
kubectl get nodes -o wide
kubectl get componentstatuses 2>/dev/null || kubectl get --raw='/readyz?verbose'
kubectl cluster-info

# 3. Verificar workloads por namespace
for ns in cloudshop-prod cloudshop-data monitoring logging; do
  echo "=== $ns ==="
  kubectl get deploy,sts,ds,job -n $ns
  echo ""
done

# 4. Verificar quotas e limits
for ns in cloudshop-dev cloudshop-staging cloudshop-prod; do
  echo "=== $ns ==="
  kubectl describe resourcequota -n $ns
  kubectl describe limitrange -n $ns
  echo ""
done

# 5. Verificar PDBs
kubectl get pdb -A

# 6. Topology Spread verificar distribuição
kubectl get pods -n cloudshop-prod -o wide --sort-by='{.spec.nodeName}'
```

**Critérios de aceite:**
- [ ] Todos os namespaces existem e estão organizados
- [ ] Todos os nodes estão Ready
- [ ] 6 Deployments em cloudshop-prod (order, product, user, payment, inventory, notification)
- [ ] 3 StatefulSets em cloudshop-data (PostgreSQL, Redis, Kafka)
- [ ] FluentBit DaemonSet em todos os nodes
- [ ] ResourceQuota e LimitRange em todos os cloudshop-* namespaces
- [ ] PDBs configurados para workloads críticos
- [ ] Pods distribuídos em múltiplos nodes (topology spread)

---

### Desafio 16.2 — Security Audit Completo (CKS + KCSA)

Execute uma auditoria de segurança completa na plataforma.

```bash
# 1. CIS Benchmark
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl wait --for=condition=complete job/kube-bench --timeout=120s
kubectl logs job/kube-bench | tail -20

# 2. Verificar RBAC
kubectl get clusterrolebindings -o json | jq '
  .items[] |
  select(.roleRef.name == "cluster-admin") |
  {name: .metadata.name, subjects: .subjects}
'
# Deve ter mínimo de cluster-admin bindings

# 3. Verificar Pod Security Standards
kubectl get namespaces -o json | jq '
  .items[] |
  select(.metadata.name | startswith("cloudshop")) |
  {name: .metadata.name, labels: .metadata.labels | 
   with_entries(select(.key | startswith("pod-security")))}
'
# cloudshop-prod DEVE ter: enforce=restricted

# 4. Verificar Network Policies
for ns in cloudshop-prod cloudshop-data; do
  echo "=== $ns ==="
  kubectl get networkpolicies -n $ns
done
# Deve ter default-deny + allow específicos

# 5. Verificar secrets encryption
kubectl get pods -n kube-system kube-apiserver-* -o yaml | \
  grep encryption-provider-config

# 6. Scan de imagens
for deploy in order product user payment inventory notification; do
  image=$(kubectl get deploy $deploy -n cloudshop-prod -o jsonpath='{.spec.template.spec.containers[0].image}')
  echo "=== $deploy: $image ==="
  trivy image --severity HIGH,CRITICAL "$image" 2>/dev/null | tail -5
done

# 7. Verificar SecurityContext em todos os pods
kubectl get pods -n cloudshop-prod -o json | jq '
  .items[] |
  {
    name: .metadata.name,
    runAsNonRoot: .spec.containers[0].securityContext.runAsNonRoot,
    readOnlyRootFS: .spec.containers[0].securityContext.readOnlyRootFilesystem,
    allowPrivEsc: .spec.containers[0].securityContext.allowPrivilegeEscalation,
    capabilities: .spec.containers[0].securityContext.capabilities
  }
'

# 8. Verificar audit logging
kubectl get pods -n kube-system kube-apiserver-* -o yaml | \
  grep -E "audit-log-path|audit-policy-file"

# 9. Falco runtime security
kubectl get pods -n falco
kubectl logs -n falco -l app=falco --tail=20
```

**Critérios de aceite:**
- [ ] CIS Benchmark executado — zero FAIL em categorias críticas
- [ ] cluster-admin bindings ≤ 2 (system + emergency)
- [ ] cloudshop-prod com PSS enforce=restricted
- [ ] Default-deny Network Policies em todos os namespaces de produção
- [ ] etcd encryption at rest configurado
- [ ] Todas as imagens sem vulnerabilidades CRITICAL
- [ ] Todos os pods: runAsNonRoot=true, readOnlyRootFS=true, drop ALL
- [ ] Audit logging habilitado
- [ ] Falco instalado e monitorando

---

### Desafio 16.3 — Observability Validation (CKA + CKAD)

Valide o stack de observabilidade de ponta a ponta.

```bash
# 1. Verificar Prometheus
kubectl get prometheus -n monitoring
kubectl get servicemonitor -n monitoring
# Deve ter ServiceMonitors para todos os microservices

# 2. Verificar alertas ativos
kubectl exec -n monitoring prometheus-k8s-0 -- \
  wget -qO- 'localhost:9090/api/v1/alerts' | jq '.data.alerts | length'

# 3. Verificar PrometheusRules
kubectl get prometheusrules -n monitoring -o custom-columns='NAME:.metadata.name,GROUPS:.spec.groups[*].name'
# Deve ter: high-error-rate, pod-restart, slo-breach, resource-saturation

# 4. Verificar Grafana dashboards
kubectl get configmaps -n monitoring -l grafana_dashboard=1

# 5. Verificar Tracing (Tempo + OpenTelemetry)
kubectl get pods -n monitoring -l app=tempo
kubectl get pods -n monitoring -l app.kubernetes.io/name=opentelemetry-collector

# 6. Verificar Logging (FluentBit)
kubectl get ds -n logging
# Pods FluentBit em todos os nodes
kubectl get pods -n logging -o wide
FLUENT_POD=$(kubectl get pods -n logging -l app=fluent-bit -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n logging $FLUENT_POD --tail=5

# 7. Testar tracing end-to-end
# Fazer um request e verificar o trace
curl -s https://cloudshop.local/api/orders \
  -H "X-Request-ID: test-trace-$(date +%s)" | jq .
# Verificar trace no Tempo

# 8. Verificar SLOs
# SLO: 99.9% availability para order-service
# SLO: p99 latência < 500ms para product-service
kubectl exec -n monitoring prometheus-k8s-0 -- \
  wget -qO- 'localhost:9090/api/v1/query?query=slo:availability:ratio{service="order"}' | jq .
```

**Critérios de aceite:**
- [ ] Prometheus coletando métricas de todos os microservices
- [ ] ServiceMonitors configurados para 6 microservices + 3 StatefulSets
- [ ] AlertRules: error-rate, pod-restart, slo-breach, resource-saturation
- [ ] Grafana: dashboards USE (resources) e RED (requests) importados
- [ ] Tracing: OpenTelemetry Collector → Tempo funcional
- [ ] Logging: FluentBit coletando logs de todos os nodes
- [ ] SLOs definidos e mensuráveis via PromQL

---

### Desafio 16.4 — CI/CD & GitOps Validation (CKAD + CKA)

Valide o pipeline completo de CI/CD com GitOps.

```bash
# 1. Verificar ArgoCD
kubectl get applications -n argocd
# Deve ter ApplicationSet gerando apps por microservice × ambiente

# 2. Verificar sync status
kubectl get applications -n argocd -o custom-columns=\
'NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'
# Todos devem estar: Synced + Healthy

# 3. Verificar Kustomize overlays
ls -la gitops/base/
ls -la gitops/overlays/dev/
ls -la gitops/overlays/staging/
ls -la gitops/overlays/prod/

# 4. Verificar Helm charts
helm list -A
# Deve ter releases para: prometheus, grafana, tempo, fluentbit, argocd,
# ingress-nginx, cert-manager, external-secrets

# 5. Testar promotion pipeline
# dev → staging → prod
# Alterar image tag e verificar propagação

# 6. Verificar ApplicationSet matrix
kubectl get applicationset -n argocd -o yaml | grep -A 20 'generators'
# Matrix: 6 microservices × 3 ambientes = 18 Applications

# 7. Verificar Image Updater
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-image-updater
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater --tail=10
```

**Critérios de aceite:**
- [ ] ArgoCD instalado e gerenciando todos os ambientes
- [ ] ApplicationSet Matrix gerando 18 Applications (6×3)
- [ ] Todos os Applications: Synced + Healthy
- [ ] Kustomize: base + 3 overlays (dev/staging/prod)
- [ ] Helm charts para infraestrutura (prometheus, grafana, etc.)
- [ ] Image Updater automático para dev
- [ ] Promotion pipeline documentado (dev → staging → prod)

---

### Desafio 16.5 — Disaster Recovery Drill (CKA + CKS)

Execute um DR drill completo na plataforma.

```bash
# ========================================
# FASE 1: PREPARAÇÃO
# ========================================

# 1. Documentar estado atual
echo "=== PRE-DR STATE ===" > /tmp/dr-report.txt
kubectl get nodes >> /tmp/dr-report.txt
kubectl get pods -A --field-selector=status.phase=Running | wc -l >> /tmp/dr-report.txt
kubectl get pv >> /tmp/dr-report.txt

# 2. Verificar backups existentes
velero backup get
velero backup describe cloudshop-daily-latest

# 3. Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-pre-dr.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-pre-dr.db --write-table

# ========================================
# FASE 2: SIMULAR DESASTRE
# ========================================

# 4. Cenário: namespace cloudshop-prod deletado acidentalmente
kubectl delete namespace cloudshop-prod --wait=false

# 5. Verificar impacto
kubectl get pods -n cloudshop-prod
# Esperado: namespace não encontrado

# ========================================
# FASE 3: RECOVERY
# ========================================

# 6. Restaurar via Velero
velero restore create cloudshop-dr-restore \
  --from-backup cloudshop-daily-latest \
  --include-namespaces cloudshop-prod

# 7. Monitorar restauração
velero restore describe cloudshop-dr-restore
kubectl get pods -n cloudshop-prod -w

# 8. Verificar integridade dos dados
kubectl exec -n cloudshop-data postgresql-0 -- \
  psql -U cloudshop -c "SELECT count(*) FROM orders;"

# ========================================
# FASE 4: VALIDAÇÃO
# ========================================

# 9. Smoke tests
curl -s https://cloudshop.local/health | jq .
curl -s https://cloudshop.local/api/products | jq '.items | length'

# 10. Calcular métricas
echo "=== DR METRICS ===" >> /tmp/dr-report.txt
echo "RPO: tempo desde último backup" >> /tmp/dr-report.txt
echo "RTO: tempo total de recovery" >> /tmp/dr-report.txt
echo "Data integrity: verificar contagem de registros" >> /tmp/dr-report.txt
```

| Métrica | Target | Resultado |
|---------|--------|-----------|
| RPO (Recovery Point Objective) | ≤ 1h | _____ |
| RTO (Recovery Time Objective) | ≤ 15min | _____ |
| Data Integrity | 100% | _____ |
| Services Recovered | 6/6 | _____ |
| StatefulSets Recovered | 3/3 | _____ |

**Critérios de aceite:**
- [ ] Backup Velero funcional com schedule diário
- [ ] etcd snapshot criado e verificado
- [ ] Namespace destruído e restaurado com sucesso
- [ ] RPO ≤ 1 hora
- [ ] RTO ≤ 15 minutos
- [ ] Data integrity 100% (contagem de registros confere)
- [ ] Todos os 6 services + 3 StatefulSets restaurados
- [ ] Smoke tests passando após recovery
- [ ] DR report documentado

---

### Desafio 16.6 — Kubestronaut Readiness Assessment

Auto-avaliação final para validar prontidão Kubestronaut.

```markdown
## Checklist de Competências — Kubestronaut

### KCNA (Conceitual)
- [ ] Explico a arquitetura K8s (control plane, data plane, etcd)
- [ ] Entendo CNCF landscape e graduated projects
- [ ] Sei a diferença entre containers, VMs e serverless
- [ ] Entendo GitOps, CI/CD e Cloud Native principles
- [ ] Conheço os 12 Factor App principles

### CKAD (Developer)
- [ ] Crio Deployments, Services, Ingress em < 2min
- [ ] Configuro probes (liveness, readiness, startup)
- [ ] Gerencio ConfigMaps e Secrets
- [ ] Crio Jobs, CronJobs e init containers
- [ ] Debugo pods com logs, exec, port-forward
- [ ] Uso kubectl eficientemente (aliases, dry-run, explain)

### CKA (Administrator)
- [ ] Instalo cluster com kubeadm
- [ ] Faço upgrade de cluster (control plane + workers)
- [ ] Backup/restore de etcd
- [ ] Configuro RBAC (users, groups, service accounts)
- [ ] Troubleshoot: CrashLoopBackOff, Pending, OOMKilled
- [ ] Gerencio networking (Services, DNS, Ingress, NetworkPolicy)
- [ ] Configuro StorageClass, PV, PVC

### CKS (Security)
- [ ] Executo CIS Benchmark com kube-bench
- [ ] Configuro Pod Security Standards (PSS)
- [ ] Implemento Network Policies (default deny + allow)
- [ ] Gerencio Secrets com External Secrets Operator
- [ ] Configuro audit logging
- [ ] Instalo e configuro Falco para runtime security
- [ ] Scan de imagens com Trivy
- [ ] Implemento supply chain security (Cosign)
- [ ] SecurityContext: non-root, readOnly, drop ALL, seccomp

### KCSA (Security Concepts)
- [ ] Explico 4 Cs (Cloud, Cluster, Container, Code)
- [ ] Aplico STRIDE threat model ao K8s
- [ ] Conheço MITRE ATT&CK for Containers
- [ ] Entendo SLSA framework
- [ ] Conheço CIS Benchmark, NIST 800-190, SOC 2
- [ ] Entendo Zero Trust Architecture
```

```bash
# Score Calculator
echo "=== KUBESTRONAUT READINESS ==="
echo ""
echo "Certificação  | Score  | Status"
echo "------------- | ------ | ------"
echo "KCNA          |   /5   | "
echo "CKAD          |   /6   | "
echo "CKA           |   /7   | "
echo "CKS           |   /9   | "
echo "KCSA          |   /6   | "
echo "------------- | ------ | ------"
echo "TOTAL         |   /33  | "
echo ""
echo "Aprovação: ≥ 28/33 (85%)"
echo ""
echo "Ordem recomendada de certificação:"
echo "1. KCNA (entry-level, conceitual)"
echo "2. CKAD (developer hands-on)"
echo "3. CKA  (admin hands-on)"
echo "4. KCSA (security conceitual)"
echo "5. CKS  (security hands-on, requer CKA)"
```

**Critérios de aceite:**
- [ ] Checklist preenchido com honestidade
- [ ] Score ≥ 28/33 (85%)
- [ ] Gaps identificados e plano de estudo criado
- [ ] CloudShop Platform 100% operacional
- [ ] DR drill executado com sucesso
- [ ] Security audit sem CRITICAL findings
- [ ] Observability stack completo e funcional

---

## Definição de Pronto (DoD)

### Platform Checklist
- [ ] 6 microservices running em cloudshop-prod (3+ replicas cada)
- [ ] 3 StatefulSets running em cloudshop-data (PostgreSQL, Redis, Kafka)
- [ ] HPA configurado para todos os microservices
- [ ] PDBs para workloads críticos
- [ ] Topology Spread em múltiplos nodes

### Security Checklist
- [ ] CIS Benchmark: zero FAIL crítico
- [ ] PSS enforce=restricted em prod
- [ ] Network Policies default-deny em todos os namespaces
- [ ] etcd encryption at rest
- [ ] Audit logging habilitado
- [ ] Falco monitorando runtime
- [ ] Image scanning sem CRITICAL CVEs

### Observability Checklist
- [ ] Prometheus + ServiceMonitors para todos os workloads
- [ ] Grafana dashboards (USE + RED)
- [ ] Alerting rules (error-rate, pod-restart, SLO breach)
- [ ] Distributed tracing (OTel → Tempo)
- [ ] Centralized logging (FluentBit)

### CI/CD Checklist
- [ ] ArgoCD com ApplicationSet (6×3 = 18 apps)
- [ ] Kustomize overlays (dev/staging/prod)
- [ ] Image Updater automático para dev
- [ ] Promotion pipeline documentado

### DR Checklist
- [ ] Velero backups diários
- [ ] etcd snapshots
- [ ] DR drill executado: RPO ≤ 1h, RTO ≤ 15min
- [ ] Runbook de DR documentado

### Certificações Checklist
- [ ] KCNA: simulado ≥ 75%
- [ ] CKAD: simulado ≥ 66%
- [ ] CKA: simulado ≥ 66%
- [ ] CKS: simulado ≥ 67%
- [ ] KCSA: simulado ≥ 75%

---

## Rubrica Final

| Nível | Critério |
|-------|----------|
| 0-20 | Cluster básico, poucos workloads |
| 21-40 | Workloads rodando, sem segurança/observability |
| 41-60 | Segurança parcial, observability básica |
| 61-80 | Plataforma funcional, falta DR ou compliance |
| 81-90 | Plataforma production-grade, DR testado |
| 91-100 | Kubestronaut ready — todas as checklists ✅ |
