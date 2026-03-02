# Level 11 — KCNA (Kubernetes and Cloud Native Associate)

> **Objetivo:** Preparar-se para a certificação KCNA — prova de múltipla escolha que valida
> conhecimentos fundamentais de Kubernetes e ecossistema Cloud Native.

**Referência:** [KCNA Certification Guide](../../.docs/k8s/11-kcna.md)

---

## Sobre a Certificação

| Campo | Detalhe |
|-------|---------|
| Tipo | Múltipla escolha (60 questões) |
| Duração | 90 minutos |
| Aprovação | 75% |
| Validade | 3 anos |
| Custo | $250 USD |
| Pré-requisito | Nenhum |

### Domínios

| Domínio | Peso |
|---------|------|
| Kubernetes Fundamentals | 46% |
| Container Orchestration | 22% |
| Cloud Native Architecture | 16% |
| Cloud Native Observability | 8% |
| Cloud Native Application Delivery | 8% |

---

## Contexto do Domínio

Use a plataforma CloudShop como base para responder questões conceituais. Os desafios mapeiam os domínios da prova para exercícios práticos que reforçam a teoria.

---

## Desafios

### Desafio 11.1 — Kubernetes Fundamentals (46%)

Domine os conceitos fundamentais do Kubernetes.

**Tópicos para estudo:**
- Arquitetura do cluster (control plane + worker nodes)
- Componentes: kube-apiserver, etcd, kube-scheduler, kube-controller-manager, kubelet, kube-proxy
- Objetos: Pod, Deployment, Service, Namespace, ConfigMap, Secret
- Kubectl: comandos imperativos e declarativos
- RBAC: Role, ClusterRole, RoleBinding, ClusterRoleBinding

**Exercícios práticos:**
```bash
# 1. Identificar componentes do control plane
kubectl get pods -n kube-system

# 2. Descrever o que cada componente faz
kubectl describe pod kube-apiserver-control-plane -n kube-system

# 3. Criar recursos de forma declarativa e imperativa
kubectl create deployment nginx --image=nginx:1.25 --replicas=3 --dry-run=client -o yaml
kubectl apply -f deployment.yaml

# 4. Explorar API resources
kubectl api-resources | head -20
kubectl api-versions
kubectl explain deployment.spec.strategy

# 5. RBAC
kubectl auth can-i create pods --as=system:serviceaccount:cloudshop-prod:order-service
kubectl auth can-i '*' '*' --as=system:serviceaccount:kube-system:default
```

**Critérios de aceite:**
- [ ] Explicar a função de cada componente do control plane
- [ ] Diferenciar Pod, Deployment, ReplicaSet, StatefulSet, DaemonSet
- [ ] Criar recursos via kubectl (imperativo e declarativo)
- [ ] Explicar RBAC (Role vs ClusterRole, Binding vs ClusterBinding)
- [ ] Navegar a API com `kubectl explain`

---

### Desafio 11.2 — Container Orchestration (22%)

Entenda os conceitos de orquestração de containers.

**Tópicos para estudo:**
- Container runtimes: containerd, CRI-O
- Container lifecycle e estados de Pod
- Scheduling: nodeSelector, affinity, taints/tolerations
- Scaling: HPA, VPA, manual
- Rolling updates e rollbacks

**Exercícios práticos:**
```bash
# 1. Container runtime
kubectl get nodes -o wide  # ver CONTAINER-RUNTIME

# 2. Pod lifecycle
kubectl run lifecycle-test --image=busybox --restart=Never -- sh -c "echo hello; sleep 30"
kubectl get pod lifecycle-test -w  # observar estados: Pending → Running → Completed

# 3. Scheduling
kubectl label node node-worker-1 tier=frontend
kubectl run scheduled --image=nginx --overrides='{"spec":{"nodeSelector":{"tier":"frontend"}}}'

# 4. Scaling
kubectl scale deployment order-service --replicas=5 -n cloudshop-prod
kubectl autoscale deployment order-service --min=2 --max=10 --cpu-percent=70

# 5. Rolling update e rollback
kubectl set image deployment/order-service order-service=registry.cloudshop.io/order-service:v1.3.0 -n cloudshop-prod
kubectl rollout status deployment/order-service -n cloudshop-prod
kubectl rollout history deployment/order-service -n cloudshop-prod
kubectl rollout undo deployment/order-service -n cloudshop-prod
```

**Critérios de aceite:**
- [ ] Explicar CRI e container runtimes
- [ ] Descrever lifecycle de um Pod (estados e transições)
- [ ] Demonstrar scheduling com nodeSelector e affinity
- [ ] Executar rolling update e rollback
- [ ] HPA configurado e testado

---

### Desafio 11.3 — Cloud Native Architecture (16%)

Compreenda padrões de arquitetura cloud native.

**Tópicos para estudo:**
- 12-Factor Apps
- Microservices vs Monolith
- CNCF landscape e projetos graduados
- Cloud Native patterns: sidecar, ambassador, adapter
- Serverless e FaaS

**Exercícios:**
```markdown
## Quiz — Cloud Native Architecture

1. Quais são os 12 fatores de uma Cloud Native App? Liste e explique cada um.

2. Compare microservices vs monolith:
   - Quando usar cada um?
   - Trade-offs de cada abordagem?
   - Como a CloudShop se beneficia de microservices?

3. Cite 5 projetos CNCF graduados e sua função:
   - Kubernetes: orquestração
   - Prometheus: monitoramento
   - Envoy: service proxy
   - containerd: container runtime
   - Helm: package manager

4. Explique os multi-container patterns:
   - Sidecar: FluentBit coletando logs
   - Ambassador: proxy para external services
   - Adapter: transformar output format

5. O que é GitOps e por que é importante?
```

**Critérios de aceite:**
- [ ] 12-Factor Apps explicados
- [ ] Microservices vs Monolith comparados
- [ ] 5+ projetos CNCF identificados com função
- [ ] Multi-container patterns explicados com exemplos da CloudShop
- [ ] GitOps definido e justificado

---

### Desafio 11.4 — Cloud Native Observability (8%)

Compreenda pilares de observabilidade cloud native.

**Tópicos para estudo:**
- Três pilares: Logs, Metrics, Traces
- Prometheus e modelo de métricas (pull-based)
- OpenTelemetry
- Fluentd/FluentBit para logging
- Grafana para visualização

**Exercícios:**
```bash
# 1. Verificar métricas no Prometheus
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090 -n monitoring
# Executar queries PromQL:
# rate(http_requests_total{namespace="cloudshop-prod"}[5m])
# container_memory_working_set_bytes{namespace="cloudshop-prod"}

# 2. Verificar logs
kubectl logs -l app.kubernetes.io/name=order-service -n cloudshop-prod --tail=50

# 3. Verificar traces (se Tempo/Jaeger instalado)
kubectl port-forward svc/tempo 3100 -n monitoring
```

**Critérios de aceite:**
- [ ] 3 pilares de observabilidade explicados
- [ ] Prometheus: pull vs push model explicado
- [ ] PromQL básico demonstrado
- [ ] OpenTelemetry: papel e componentes explicados
- [ ] CNCF observability stack identificado

---

### Desafio 11.5 — Cloud Native Application Delivery (8%)

Compreenda métodos de entrega de aplicações cloud native.

**Tópicos para estudo:**
- CI/CD pipelines
- GitOps (ArgoCD, Flux)
- Helm e package management
- Kustomize
- Canary, Blue/Green, Rolling deployments

**Exercícios:**
```markdown
## Quiz — Application Delivery

1. Diferencie CI vs CD vs CD (Continuous Integration vs Delivery vs Deployment)

2. Compare GitOps vs Traditional CI/CD:
   - Push-based vs Pull-based
   - Source of truth
   - Drift detection

3. Helm vs Kustomize:
   - Quando usar Helm? (parameterization complexa, library charts)
   - Quando usar Kustomize? (overlays simples, sem templating)
   - Podem ser usados juntos? Como?

4. Deployment strategies:
   - Rolling Update: como funciona no Kubernetes?
   - Blue/Green: como implementar?
   - Canary: como implementar com Gateway API?

5. O que é um Helm Chart? Quais são seus componentes?
```

**Critérios de aceite:**
- [ ] CI/CD pipeline explicado
- [ ] GitOps vs Traditional comparado
- [ ] Helm vs Kustomize: trade-offs documentados
- [ ] 3 deployment strategies explicadas
- [ ] ArgoCD ou Flux: arquitetura explicada

---

### Desafio 11.6 — Simulado KCNA

Execute um simulado completo com 60 questões no estilo da prova.

**Requisitos:**
- 60 questões de múltipla escolha
- 90 minutos de tempo
- Cobrir todos os domínios com os pesos corretos
- Alvo: 75% de acerto

**Recursos para simulado:**
```markdown
- https://killer.sh/kcna (simulado oficial)
- https://github.com/cncf/curriculum (currículo oficial)
- Revisar anotações dos desafios 11.1-11.5
- Revisar todos os levels 0-10

## Checklist pré-prova
- [ ] Completei todos os desafios 11.1-11.5
- [ ] Fiz pelo menos 2 simulados com >75%
- [ ] Revisei os conceitos que errei nos simulados
- [ ] Entendo a diferença entre recursos core e extended
- [ ] Conheço o CNCF landscape (projetos graduados)
```

**Critérios de aceite:**
- [ ] Simulado completo (60 questões, 90 min)
- [ ] Score ≥ 75%
- [ ] Questões erradas revisadas e estudadas
- [ ] Fraquezas identificadas e plano de revisão criado

---

## Definição de Pronto (DoD)

- [ ] Todos os domínios estudados com exercícios práticos
- [ ] Quiz de cada domínio completado
- [ ] Simulado com score ≥ 75%
- [ ] Questões erradas revisadas
- [ ] Pronto para agendar a prova

---

## Checklist

- [ ] Kubernetes Fundamentals — 46%
- [ ] Container Orchestration — 22%
- [ ] Cloud Native Architecture — 16%
- [ ] Cloud Native Observability — 8%
- [ ] Cloud Native App Delivery — 8%
- [ ] 2+ simulados com score ≥ 75%
