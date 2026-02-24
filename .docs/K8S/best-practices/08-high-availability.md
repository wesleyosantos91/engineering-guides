# High Availability & Disaster Recovery — Best Practices

## 1. Pod Disruption Budget (PDB)

### 1.1 PDB por Disponibilidade
```yaml
# ✅ Garante que sempre há Pods disponíveis durante disruptions
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: production
spec:
  minAvailable: 2          # mínimo de Pods disponíveis
  # ou
  # maxUnavailable: 1      # máximo de Pods indisponíveis
  selector:
    matchLabels:
      app: api
```

**Quando usar cada approach:**
| Cenário | Configuração |
|---------|:---:|
| 3 replicas, crítico | `minAvailable: 2` |
| 5+ replicas | `maxUnavailable: 1` ou `maxUnavailable: 25%` |
| StatefulSet (quórum) | `minAvailable: (N/2)+1` |
| Batch jobs | Sem PDB (ou `maxUnavailable: 50%`) |

**Regras:**
- **Todo workload de produção** deve ter PDB.
- PDB protege contra **voluntary disruptions** (node drain, upgrades).
- PDB **NÃO** protege contra involuntary disruptions (crash, hardware failure).
- Nunca `minAvailable` = replicas (bloqueia manutenção).

## 2. Topology Spread & Anti-Affinity

### 2.1 Multi-AZ Distribution
```yaml
spec:
  topologySpreadConstraints:
    # Distribuir entre AZs
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: api
    # Distribuir entre nodes
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: api
```

### 2.2 Pod Anti-Affinity
```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: ["api"]
          topologyKey: kubernetes.io/hostname
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values: ["api"]
            topologyKey: topology.kubernetes.io/zone
```

## 3. Control Plane HA

### 3.1 Componentes Críticos
| Componente | Replicas | Estratégia HA |
|-----------|:---:|------------|
| **API Server** | 3+ | Load balancer fronting |
| **etcd** | 3 ou 5 | Raft consensus (odd numbers) |
| **Controller Manager** | 2+ | Leader election |
| **Scheduler** | 2+ | Leader election |
| **CoreDNS** | 2+ | Horizontal scaling |
| **Ingress Controller** | 2+ | Multi-AZ |

### 3.2 etcd Best Practices
- **Sempre 3 ou 5 membros** (odd numbers para quórum).
- **SSD dedicado** para etcd (latência < 10ms).
- **Backup automático** a cada 1-4 horas.
- **Encriptação at rest** habilitada.
- Monitore: `etcd_server_leader_changes_seen_total`, `etcd_disk_wal_fsync_duration_seconds`.

```bash
# Backup do etcd
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db
```

## 4. Multi-Cluster Strategy

### 4.1 Topologias
```
Opção 1: Active-Active (Multi-Region)
┌─────────────┐     ┌─────────────┐
│  Cluster A  │◄───▶│  Cluster B  │
│ (us-east-1) │     │ (eu-west-1) │
└──────┬──────┘     └──────┬──────┘
       │                    │
       └────────┬───────────┘
           Global LB
           (Route53 / CloudFlare)

Opção 2: Active-Passive (DR)
┌─────────────┐     ┌─────────────┐
│  Cluster A  │────▶│  Cluster B  │
│  (Primary)  │     │  (Standby)  │
└─────────────┘     └─────────────┘
  Active              Warm/Cold Standby

Opção 3: Hub-Spoke
┌─────────────┐
│  Hub Cluster│
│ (Management)│
└──────┬──────┘
       │
   ┌───┼───┐
   ▼   ▼   ▼
  C1   C2  C3  (Spoke Clusters)
```

### 4.2 Ferramentas Multi-Cluster
| Ferramenta | Descrição |
|-----------|-----------|
| **Submariner** | Conectividade cross-cluster |
| **Skupper** | Application-layer multi-cluster |
| **Liqo** | Virtual Kubelet cross-cluster |
| **KubeFed** | Federation de recursos |
| **ArgoCD** | GitOps multi-cluster |
| **Istio Multi-Cluster** | Service mesh cross-cluster |
| **Cilium Cluster Mesh** | Networking cross-cluster |

## 5. Disaster Recovery

### 5.1 RPO e RTO
| Tier | RPO | RTO | Estratégia |
|------|-----|-----|------------|
| **Tier 1** | 0 | < 15min | Active-Active, replicação síncrona |
| **Tier 2** | < 1h | < 1h | Active-Passive, replicação assíncrona |
| **Tier 3** | < 24h | < 4h | Backup/Restore |
| **Tier 4** | < 24h | < 24h | Rebuild from scratch (IaC + GitOps) |

### 5.2 DR Checklist
```
Infraestrutura:
  ✅ IaC (Terraform) para recriar cluster rapidamente
  ✅ GitOps para redeployar todas as aplicações
  ✅ DNS com failover automático

Dados:
  ✅ etcd backup automático (testado)
  ✅ Database backup automático (testado)
  ✅ Volume snapshots automáticos
  ✅ Cross-region replication para dados críticos

Configuração:
  ✅ Secrets backup/replicação
  ✅ Certificates backup
  ✅ DNS records documentados

Processo:
  ✅ Runbook de DR documentado
  ✅ DR drills trimestrais
  ✅ Roles e responsabilidades definidos
  ✅ Comunicação (status page, escalation)
```

### 5.3 Velero para DR
```yaml
# Backup schedule
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: disaster-recovery
  namespace: velero
spec:
  schedule: "0 */4 * * *"  # a cada 4 horas
  template:
    includedNamespaces:
      - production
      - staging
    includedResources:
      - '*'
    storageLocation: aws-backup
    snapshotVolumes: true
    ttl: 720h  # 30 dias
  useOwnerReferencesInBackup: false
```

## 6. Graceful Shutdown

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: api
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 15 && kill -SIGTERM 1"]
```

**Sequência de shutdown:**
```
1. Pod marcado para termination
2. Pod removido dos endpoints (Service)
3. preStop hook executado ← aguarda conexões drenarem
4. SIGTERM enviado ao processo
5. Aplicação faz graceful shutdown
6. Após terminationGracePeriodSeconds → SIGKILL
```

**Regras:**
- `preStop` com **sleep 10-15s** para aguardar remoção dos endpoints.
- Aplicação deve tratar **SIGTERM** e parar de aceitar novas conexões.
- Configure `terminationGracePeriodSeconds` > (preStop sleep + app shutdown time).

## 7. Node Management

### 7.1 Node Groups Strategy
```
# EKS / GKE / AKS — separar node groups por tipo
┌──────────────────┐
│ System Node Group │  ← CoreDNS, monitoring, ingress
│ (m5.large, 3 AZs)│
├──────────────────┤
│ App Node Group    │  ← Application workloads
│ (m5.xlarge, 3 AZs)│
├──────────────────┤
│ GPU Node Group    │  ← ML/AI workloads
│ (p3.2xlarge)      │
├──────────────────┤
│ Spot Node Group   │  ← Batch, dev, non-critical
│ (m5.xlarge, spot) │
└──────────────────┘
```

### 7.2 Cluster Autoscaler / Karpenter
```yaml
# Karpenter (AWS) — mais eficiente que Cluster Autoscaler
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "spot"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.xlarge", "m5.2xlarge", "m6i.xlarge", "m6i.2xlarge"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h  # 30 dias — force rotation
```

## 8. Boas Práticas Resumo

- **PDB** em todos os workloads de produção.
- **Multi-AZ** distribution obrigatório (topologySpreadConstraints).
- **Mínimo 3 replicas** para workloads críticos.
- **Graceful shutdown** implementado em todas as aplicações.
- **DR drills** trimestrais (teste backup + restore).
- **GitOps** para rebuild rápido de todo o ambiente.
- **IaC** para recriar infraestrutura em minutos.
- **Monitoramento** de saúde do cluster (etcd, API server, nodes).
