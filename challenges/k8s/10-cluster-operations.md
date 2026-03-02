# Level 10 — Cluster Operations

> **Objetivo:** Dominar operações de cluster — upgrades, manutenção de nodes, troubleshooting,
> etcd operations, cost optimization e compliance — para operar a plataforma CloudShop em produção.

**Referência:** [Cluster Operations Best Practices](../../.docs/k8s/10-cluster-operations.md)

---

## Objetivo de Aprendizado

- Executar upgrades de cluster (kubeadm e managed K8s)
- Gerenciar nodes (drain, cordon, rotation)
- Diagnosticar problemas comuns (CrashLoopBackOff, Pending, Evicted)
- Operar e fazer backup do etcd
- Otimizar custos (right-sizing, spot instances, Kubecost)
- Implementar compliance com Kyverno

---

## Contexto do Domínio

A plataforma CloudShop em produção precisa de operações planificadas: upgrades sem downtime, manutenção de nodes, troubleshooting rápido de incidentes, backup do etcd, otimização de custos e compliance com políticas de segurança da empresa.

---

## Desafios

### Desafio 10.1 — Cluster Upgrades

Execute upgrade do cluster Kubernetes de forma segura.

**Requisitos:**
- Upgrade do control plane primeiro, depois workers
- Um minor version por vez (1.29 → 1.30, nunca 1.29 → 1.31)
- Testar com PDBs e draining de nodes
- Documentar runbook de upgrade

```bash
# === UPGRADE DO CONTROL PLANE ===

# 1. Verificar versões disponíveis
kubeadm upgrade plan

# 2. Atualizar kubeadm
apt-get update && apt-get install -y kubeadm=1.30.0-00

# 3. Aplicar upgrade no control plane
kubeadm upgrade apply v1.30.0

# 4. Atualizar kubelet e kubectl
apt-get install -y kubelet=1.30.0-00 kubectl=1.30.0-00
systemctl daemon-reload && systemctl restart kubelet

# === UPGRADE DOS WORKERS (um por vez) ===

# 5. Drain worker node
kubectl drain node-worker-1 --ignore-daemonsets --delete-emptydir-data

# 6. SSH no worker e atualizar
ssh node-worker-1
apt-get update && apt-get install -y kubeadm=1.30.0-00
kubeadm upgrade node
apt-get install -y kubelet=1.30.0-00
systemctl daemon-reload && systemctl restart kubelet
exit

# 7. Uncordon
kubectl uncordon node-worker-1

# 8. Verificar
kubectl get nodes
# Todos devem mostrar v1.30.0 e Ready

# 9. Repetir para cada worker node
```

**Critérios de aceite:**
- [ ] Control plane atualizado primeiro
- [ ] Workers atualizados um por vez
- [ ] PDBs respeitados durante drain
- [ ] Zero downtime para workloads
- [ ] `kubectl get nodes` mostra nova versão
- [ ] Runbook documentado (passo a passo)

---

### Desafio 10.2 — Node Maintenance

Execute manutenção de nodes com zero downtime.

**Requisitos:**
- Cordon/drain/uncordon workflow
- Rotação de nodes (substituir node antigo por novo)
- Gerenciar Pods com local storage durante drain

```bash
# Cordon — impedir novos Pods
kubectl cordon node-worker-2

# Drain — evacuar Pods existentes (respeita PDB)
kubectl drain node-worker-2 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=300s

# Verificar que Pods foram movidos
kubectl get pods -o wide -A | grep node-worker-2
# Devem restar apenas DaemonSets

# Executar manutenção (kernel update, reboot, etc.)
ssh node-worker-2
sudo apt-get update && sudo apt-get upgrade -y
sudo reboot

# Uncordon — permitir novos Pods
kubectl uncordon node-worker-2

# Verificar node Ready
kubectl get nodes
```

```bash
# Node rotation — substituir node antigo
# 1. Adicionar novo node ao cluster
kubeadm join <control-plane>:6443 --token <token> --discovery-token-ca-cert-hash <hash>

# 2. Verificar novo node
kubectl get nodes  # node-worker-4 Ready

# 3. Drain node antigo
kubectl drain node-worker-1 --ignore-daemonsets --delete-emptydir-data

# 4. Remover node antigo
kubectl delete node node-worker-1

# 5. Cleanup no host removido
ssh node-worker-1
kubeadm reset
```

**Critérios de aceite:**
- [ ] Cordon impede novos Pods no node
- [ ] Drain evacua Pods respeitando PDB
- [ ] DaemonSets permanecem durante drain
- [ ] Node maintenance executada com zero downtime
- [ ] Node rotation: adicionar novo, remover antigo
- [ ] Workflow documentado

---

### Desafio 10.3 — Troubleshooting

Diagnostique e resolva problemas comuns de Kubernetes.

**Requisitos:**
- Diagnosticar: CrashLoopBackOff, ImagePullBackOff, Pending, Evicted, OOMKilled
- Usar: describe, logs, events, debug containers
- Documentar runbook de troubleshooting

```bash
# === CrashLoopBackOff ===
kubectl get pods -n cloudshop-prod | grep CrashLoop
kubectl describe pod <pod-name> -n cloudshop-prod
kubectl logs <pod-name> -n cloudshop-prod --previous  # logs da instância anterior
# Causas comuns: config errada, missing env var, port conflict, permission denied

# === ImagePullBackOff ===
kubectl describe pod <pod-name> | grep -A 5 "Events"
# Causas: imagem não existe, registry privado sem imagePullSecrets, tag errada

# === Pending ===
kubectl describe pod <pod-name> | grep -A 10 "Events"
kubectl get events --field-selector reason=FailedScheduling
# Causas: resources insuficientes, nodeSelector sem match, taint sem toleration

# === OOMKilled ===
kubectl get pods -n cloudshop-prod -o json | \
  jq '.items[] | select(.status.containerStatuses[]?.lastState.terminated.reason == "OOMKilled") | .metadata.name'
kubectl describe pod <pod-name> | grep -A 3 "Last State"
# Solução: aumentar memory limit

# === Evicted ===
kubectl get pods -A --field-selector status.phase=Failed | grep Evicted
# Causas: disk pressure, memory pressure
kubectl describe node <node> | grep -A 5 "Conditions"

# === Debug container ===
kubectl debug -it <pod-name> -n cloudshop-prod \
  --image=nicolaka/netshoot \
  --target=order-service

# === Node debug ===
kubectl debug node/node-worker-1 -it --image=ubuntu
# chroot /host (acesso ao filesystem do node)
```

**Critérios de aceite:**
- [ ] CrashLoopBackOff diagnosticado e resolvido
- [ ] ImagePullBackOff diagnosticado e resolvido
- [ ] Pending (scheduling) diagnosticado e resolvido
- [ ] OOMKilled diagnosticado e resolvido
- [ ] Ephemeral debug container usado
- [ ] Runbook de troubleshooting documentado

---

### Desafio 10.4 — etcd Operations

Execute operações de backup e restore do etcd.

**Requisitos:**
- Backup do etcd (snapshot)
- Verificar integridade do snapshot
- Restore do etcd a partir do snapshot
- Monitorar saúde do etcd

```bash
# Verificar saúde do etcd
kubectl exec -n kube-system etcd-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Snapshot (backup)
kubectl exec -n kube-system etcd-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup.db

# Copiar backup para local
kubectl cp kube-system/etcd-control-plane:/tmp/etcd-backup.db ./etcd-backup.db

# Verificar integridade
etcdctl snapshot status ./etcd-backup.db --write-table

# Restore (⚠️ apenas em emergência!)
# 1. Parar kube-apiserver
# 2. Restore etcd data directory
etcdctl snapshot restore ./etcd-backup.db \
  --data-dir=/var/lib/etcd-restored
# 3. Atualizar etcd manifest para usar /var/lib/etcd-restored
# 4. Reiniciar etcd e kube-apiserver
```

**Critérios de aceite:**
- [ ] etcd health check funciona
- [ ] Snapshot criado com sucesso
- [ ] Snapshot copiado para fora do cluster
- [ ] Integridade verificada (status ok)
- [ ] Processo de restore documentado
- [ ] CronJob (ou schedule) para backup periódico do etcd

---

### Desafio 10.5 — Cost Optimization

Otimize custos do cluster identificando waste e right-sizing.

**Requisitos:**
- Identificar Pods over-provisioned (request >> utilização real)
- Identificar namespaces sem quotas
- Avaliar uso de spot instances para workloads tolerant
- Calcular custo estimado por namespace

```bash
# Over-provisioned: request muito acima da utilização real
kubectl top pods -n cloudshop-prod --sort-by=cpu
kubectl top pods -n cloudshop-prod --sort-by=memory

# Comparar request vs utilização
for pod in $(kubectl get pods -n cloudshop-prod -o name); do
  echo "--- $pod ---"
  echo "Requests:"
  kubectl get $pod -n cloudshop-prod -o jsonpath='{.spec.containers[0].resources.requests}'
  echo ""
  echo "Usage:"
  kubectl top $pod -n cloudshop-prod --no-headers
  echo ""
done

# Namespaces sem ResourceQuota
for ns in $(kubectl get ns -o name); do
  quotas=$(kubectl get resourcequota -n ${ns##*/} 2>/dev/null | wc -l)
  if [ "$quotas" -le 1 ]; then
    echo "⚠️ ${ns} — sem ResourceQuota"
  fi
done

# Idle pods (CPU < 5m por mais de 24h)
# Usar VPA recommendations como baseline

# Spot instances — workloads tolerant
# Microsserviços stateless → OK para spot
# Databases (StatefulSets) → NÃO usar spot
```

```yaml
# Node affinity para spot instances (microsserviços)
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: node.kubernetes.io/lifecycle
                operator: In
                values:
                  - spot
  tolerations:
    - key: "spot"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
```

**Critérios de aceite:**
- [ ] Over-provisioned pods identificados (tabela: request vs usage)
- [ ] Namespaces sem quotas identificados
- [ ] VPA recomendações analisadas para right-sizing
- [ ] Spot instances avaliados (quais workloads são tolerant)
- [ ] Relatório de otimização gerado

---

### Desafio 10.6 — Compliance com Kyverno

Implemente políticas de compliance para governança do cluster.

**Requisitos:**
- Instalar Kyverno
- Policies em modo Audit (logar violações) e Enforce (bloquear)
- Políticas: require labels, deny latest tag, require resources, deny privileged

```yaml
# Kyverno — require labels
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-app-label
      match:
        any:
          - resources:
              kinds:
                - Deployment
                - StatefulSet
                - DaemonSet
      validate:
        message: "Labels 'app.kubernetes.io/name' e 'app.kubernetes.io/version' são obrigatórios"
        pattern:
          metadata:
            labels:
              app.kubernetes.io/name: "?*"
              app.kubernetes.io/version: "?*"
---
# Kyverno — deny latest tag
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-image-tag
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Imagens com tag 'latest' ou sem tag são proibidas"
        pattern:
          spec:
            containers:
              - image: "*:*"
            initContainers:
              - image: "*:*"
---
# Kyverno — require resource limits
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resources
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-limits
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Containers devem ter requests e limits definidos"
        pattern:
          spec:
            containers:
              - resources:
                  requests:
                    cpu: "?*"
                    memory: "?*"
                  limits:
                    cpu: "?*"
                    memory: "?*"
```

```bash
# Instalar Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno -n kyverno --create-namespace

# Verificar policies
kubectl get clusterpolicies
kubectl get policyreport -A

# Testar: deploy sem labels → rejeitado
kubectl run test-no-labels --image=nginx:1.25 -n cloudshop-dev
# Deve ser rejeitado pela policy require-labels

# Testar: deploy com latest → rejeitado
kubectl run test-latest --image=nginx -n cloudshop-dev
# Deve ser rejeitado pela policy deny-latest-tag

# Ver report de violações
kubectl get policyreport -A -o wide
```

**Critérios de aceite:**
- [ ] Kyverno instalado
- [ ] Policy require-labels (Enforce)
- [ ] Policy deny-latest-tag (Enforce)
- [ ] Policy require-resources (Enforce)
- [ ] Deploy sem labels é rejeitado
- [ ] Deploy com latest tag é rejeitado
- [ ] PolicyReport mostra violações existentes

---

## Definição de Pronto (DoD)

- [ ] Cluster upgrade executado (ou simulado) com zero downtime
- [ ] Node maintenance demonstrada (drain/cordon/uncordon)
- [ ] 5 cenários de troubleshooting diagnosticados
- [ ] etcd backup e restore documentados
- [ ] Cost optimization relatório gerado
- [ ] Kyverno policies implementadas
- [ ] Commit: `feat(level-10): add cluster operations`

---

## Checklist

- [ ] Upgrade runbook documentado
- [ ] Node drain/cordon/uncordon praticado
- [ ] Troubleshooting: CrashLoopBackOff, ImagePullBackOff, Pending, OOMKilled
- [ ] Ephemeral debug containers usados
- [ ] etcd snapshot e verify
- [ ] Over-provisioned pods identificados
- [ ] Spot instances avaliados
- [ ] Kyverno instalado com 3+ policies
- [ ] PolicyReport verificado
