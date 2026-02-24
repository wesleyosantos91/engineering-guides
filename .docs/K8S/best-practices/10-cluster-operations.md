# Cluster Operations — Best Practices

## 1. Cluster Upgrades

### 1.1 Estratégia de Upgrade
```
1. Ler Release Notes e Breaking Changes
2. Testar upgrade em ambiente dev/staging
3. Backup etcd
4. Upgrade Control Plane (um node por vez)
5. Upgrade Node Groups (rolling)
6. Validar workloads e health checks
7. Monitorar por 24-48h
```

### 1.2 Managed Kubernetes Upgrade
```bash
# EKS
aws eks update-cluster-version --name my-cluster --kubernetes-version 1.31
# Depois upgrade node groups
aws eks update-nodegroup-version --cluster-name my-cluster --nodegroup-name workers

# GKE
gcloud container clusters upgrade my-cluster --master --cluster-version 1.31
gcloud container clusters upgrade my-cluster --node-pool default-pool

# AKS
az aks upgrade --resource-group myRG --name my-cluster --kubernetes-version 1.31
```

### 1.3 Pre-Upgrade Checklist
- [ ] Backup etcd realizado
- [ ] APIs deprecadas removidas (`kubectl deprecations`)
- [ ] PDBs configurados para todos os workloads
- [ ] Node drain testado
- [ ] Rollback plan documentado
- [ ] Monitoring/alerting configurado
- [ ] Communication plan (equipes informadas)
- [ ] Admission webhooks compatíveis
- [ ] CRDs atualizados se necessário

### 1.4 Deprecation Check
```bash
# Pluto — detectar APIs deprecadas
pluto detect-all-in-cluster

# kubent — detect deprecated APIs
kubent

# kubectl — verificar API removidas
kubectl api-resources | grep -i deprecated
```

## 2. Node Maintenance

### 2.1 Node Drain
```bash
# Drain com segurança
kubectl drain node-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=120 \
  --timeout=300s \
  --pod-selector='app!=critical-db'

# Uncordon após manutenção
kubectl uncordon node-1
```

### 2.2 Node Rotation
```
# Processo de rotation (managed K8s)
1. Criar novo node group com versão atualizada
2. Cordon nodes antigos
3. Drain nodes antigos (respeitando PDB)
4. Verificar workloads nos novos nodes
5. Deletar node group antigo

# Com Karpenter — automático
spec:
  disruption:
    expireAfter: 720h  # nodes reciclados a cada 30 dias
```

## 3. Troubleshooting

### 3.1 Pod Debugging
```bash
# Status do Pod
kubectl get pod myapp-xxx -o wide
kubectl describe pod myapp-xxx

# Logs
kubectl logs myapp-xxx -c container-name
kubectl logs myapp-xxx --previous  # logs do container anterior (crash)
kubectl logs myapp-xxx --since=1h
kubectl logs -l app=myapp --all-containers

# Exec into pod
kubectl exec -it myapp-xxx -- /bin/sh

# Ephemeral debug container (container distroless)
kubectl debug pod/myapp-xxx -it --image=nicolaka/netshoot --target=myapp

# Debug de node
kubectl debug node/node-1 -it --image=ubuntu
```

### 3.2 Problemas Comuns

#### CrashLoopBackOff
```bash
# Diagnóstico
kubectl describe pod myapp-xxx  # ver Events
kubectl logs myapp-xxx --previous  # ver logs antes do crash

# Causas comuns:
# - Aplicação não inicia (missing config, bad CMD)
# - OOMKilled (memory limit muito baixo)
# - Liveness probe falhando
# - Dependência indisponível (sem readiness/startup probe)
```

#### ImagePullBackOff
```bash
# Diagnóstico
kubectl describe pod myapp-xxx | grep -A5 Events

# Causas comuns:
# - Imagem não existe (typo no nome/tag)
# - Registry privado sem imagePullSecrets
# - Rate limit do Docker Hub
# - Network issue no node
```

#### Pending Pods
```bash
# Diagnóstico
kubectl describe pod myapp-xxx | grep -A10 Events

# Causas comuns:
# - Insufficient resources (CPU/Memory)
# - Node selector/affinity sem match
# - PVC não bound
# - Taints sem toleration
# - ResourceQuota excedida

# Verificar capacidade
kubectl describe nodes | grep -A5 "Allocated resources"
kubectl top nodes
```

#### Evicted Pods
```bash
# Causas:
# - DiskPressure no node
# - MemoryPressure no node
# - BestEffort pods evictados primeiro

# Verificar
kubectl get pods --field-selector=status.phase=Failed
kubectl describe node node-1 | grep Conditions -A10
```

### 3.3 Networking Debug
```bash
# DNS
kubectl run debug --rm -it --image=nicolaka/netshoot -- nslookup api.production
kubectl run debug --rm -it --image=nicolaka/netshoot -- dig +search api

# Connectivity
kubectl run debug --rm -it --image=nicolaka/netshoot -- curl -v http://api:8080/health

# Network policies
kubectl get networkpolicies -A
kubectl describe networkpolicy default-deny -n production

# Service endpoints
kubectl get endpoints api -n production
kubectl get endpointslices -n production -l kubernetes.io/service-name=api
```

### 3.4 Performance Debug
```bash
# Resource usage
kubectl top pods -n production --sort-by=cpu
kubectl top pods -n production --sort-by=memory
kubectl top nodes

# Check throttling (via cAdvisor/Prometheus)
# container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total

# Check OOMKill
kubectl get events -n production --field-selector reason=OOMKilling
dmesg | grep -i "oom\|kill"
```

## 4. etcd Operations

### 4.1 Backup
```bash
# Snapshot
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verificar snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
```

### 4.2 Restore
```bash
# Restore
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir /var/lib/etcd-restored \
  --name etcd-0 \
  --initial-cluster etcd-0=https://etcd-0:2380 \
  --initial-advertise-peer-urls https://etcd-0:2380
```

### 4.3 Health Check
```bash
# Health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://etcd-0:2379,https://etcd-1:2379,https://etcd-2:2379

# Performance
ETCDCTL_API=3 etcdctl check perf --load="s"

# Defragment (quando db size cresce)
ETCDCTL_API=3 etcdctl defrag --endpoints=https://etcd-0:2379
```

## 5. Cost Optimization

### 5.1 Estratégias
| Estratégia | Economia Estimada | Esforço |
|-----------|:-:|:-:|
| **Rightsizing** (ajustar requests/limits) | 20-40% | Médio |
| **Spot/Preemptible instances** | 50-90% | Médio |
| **Cluster Autoscaler / Karpenter** | 15-30% | Baixo |
| **Desligar ambientes non-prod** | 30-60% | Baixo |
| **Namespace quotas** | Previne waste | Baixo |
| **Reserved Instances** | 30-60% | Baixo |

### 5.2 Ferramentas de Custo
| Ferramenta | Tipo | Descrição |
|-----------|------|-----------|
| **Kubecost** | Monitoring | Cost allocation per workload |
| **OpenCost** | Open Source | CNCF cost monitoring |
| **Goldilocks** | Recommendations | VPA-based rightsizing |
| **Robusta KRR** | CLI | Resource recommendations |
| **Kube-green** | Scheduler | Desligar workloads non-prod |

### 5.3 Spot Instances
```yaml
# Karpenter — mix on-demand + spot
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot-workers
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.xlarge", "m5a.xlarge", "m6i.xlarge", "m5.2xlarge"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 500
```

**Regras para Spot:**
- Use para **workloads tolerantes a interrupção** (batch, dev, stateless).
- **Diversifique instance types** (>4 tipos) para melhor disponibilidade.
- Implemente **graceful shutdown** (2 min warning em AWS).
- **Nunca** use spot para databases ou workloads stateful críticos.

## 6. Compliance e Governance

### 6.1 Policy Enforcement
```yaml
# Kyverno — múltiplas políticas essenciais
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-team-label
      match:
        any:
          - resources:
              kinds: ["Deployment", "StatefulSet", "DaemonSet"]
      validate:
        message: "Label 'team' is required"
        pattern:
          metadata:
            labels:
              team: "?*"

---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-limits
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Resource limits are required"
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

## 7. Useful kubectl Commands

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes -o wide
kubectl top nodes

# Quick overview
kubectl get all -n production
kubectl get events -n production --sort-by='.lastTimestamp'

# Resource usage
kubectl resource-capacity --pods --util  # (kubectl-resource-capacity plugin)

# Broad search
kubectl get pods -A --field-selector=status.phase!=Running

# JSON path
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u

# Rollout
kubectl rollout status deployment/myapp -n production
kubectl rollout history deployment/myapp -n production
kubectl rollout undo deployment/myapp -n production

# Port forward
kubectl port-forward svc/myapp 8080:80 -n production

# Copy files
kubectl cp production/myapp-xxx:/tmp/debug.log ./debug.log
```

## 8. Essential kubectl Plugins

| Plugin | Descrição |
|--------|-----------|
| **kubectx/kubens** | Switch contexto/namespace rápido |
| **kubectl-neat** | Limpa output YAML |
| **kubectl-tree** | Visualiza hierarquia de recursos |
| **kubectl-who-can** | RBAC analysis |
| **kubectl-np-viewer** | Network policy viewer |
| **kubecost** | Cost analysis |
| **kubectl-sniff** | Packet capture |
| **krew** | Plugin manager |
