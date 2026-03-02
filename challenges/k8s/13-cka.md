# Level 13 — CKA (Certified Kubernetes Administrator)

> **Objetivo:** Preparar-se para a certificação CKA — prova prática que valida
> habilidades de administração e operação de clusters Kubernetes.

**Referência:** [CKA Certification Guide](../../.docs/k8s/13-cka.md)

---

## Sobre a Certificação

| Campo | Detalhe |
|-------|---------|
| Tipo | Performance-based (hands-on CLI) |
| Duração | 120 minutos |
| Aprovação | 66% |
| Validade | 2 anos |
| Custo | $395 USD (inclui 1 retake) |
| Ambiente | Browser com terminal + clusters reais |

### Domínios

| Domínio | Peso |
|---------|------|
| Cluster Architecture, Installation & Configuration | 25% |
| Workloads & Scheduling | 15% |
| Services & Networking | 20% |
| Storage | 10% |
| Troubleshooting | 30% |

---

## Contexto do Domínio

A prova CKA foca em administração de cluster (não desenvolvimento). Você precisa configurar clusters, gerenciar nodes, troubleshooting de componentes core, networking e storage. A CloudShop serve como cenário para praticar, mas o foco é infra.

---

## Desafios

### Desafio 13.1 — Cluster Architecture (25%)

Pratique instalação, configuração e gestão de clusters.

**Tópicos da prova:**
- kubeadm: init, join, upgrade
- etcd: backup e restore
- RBAC: Roles, ClusterRoles, Bindings
- Certificados e kubeconfig
- HA control plane

**Exercícios no estilo da prova:**
```bash
# Ex 1: Inicializar cluster com kubeadm
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<ip>

# Ex 2: Join worker node
kubeadm token create --print-join-command
# No worker:
kubeadm join <control-plane>:6443 --token <token> --discovery-token-ca-cert-hash <hash>

# Ex 3: Upgrade de cluster (1.29 → 1.30)
kubeadm upgrade plan
apt-get update && apt-get install -y kubeadm=1.30.0-00
kubeadm upgrade apply v1.30.0
apt-get install -y kubelet=1.30.0-00 kubectl=1.30.0-00
systemctl daemon-reload && systemctl restart kubelet

# Ex 4: etcd backup
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Ex 5: RBAC — criar Role e RoleBinding
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  --namespace=cloudshop-prod

kubectl create rolebinding dev-pod-reader \
  --role=pod-reader \
  --user=developer \
  --namespace=cloudshop-prod

# Verify
kubectl auth can-i get pods --as=developer -n cloudshop-prod  # yes
kubectl auth can-i delete pods --as=developer -n cloudshop-prod  # no

# Ex 6: Kubeconfig
kubectl config view
kubectl config get-contexts
kubectl config use-context <context-name>
kubectl config set-context --current --namespace=cloudshop-prod
```

**Critérios de aceite:**
- [ ] Cluster init com kubeadm (ou entender o processo)
- [ ] Worker join executado
- [ ] Cluster upgrade documentado
- [ ] etcd backup e restore
- [ ] RBAC Role + RoleBinding criados (< 3 min)
- [ ] Kubeconfig manipulado

---

### Desafio 13.2 — Workloads & Scheduling (15%)

Pratique scheduling e gestão de workloads.

**Tópicos da prova:**
- Deployments, DaemonSets, StatefulSets
- Scheduling: taints, tolerations, nodeSelector, affinity
- Resource limits e LimitRanges
- Static Pods

**Exercícios no estilo da prova:**
```bash
# Ex 1: Taint e Toleration
kubectl taint nodes node-worker-1 dedicated=database:NoSchedule

# Pod com toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
  namespace: ckad-practice
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "database"
      effect: "NoSchedule"
  nodeSelector:
    kubernetes.io/hostname: node-worker-1
  containers:
    - name: postgres
      image: postgres:16
      env:
        - name: POSTGRES_PASSWORD
          value: test123
EOF

# Ex 2: Node Affinity
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-affinity
  namespace: ckad-practice
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-affinity
  template:
    metadata:
      labels:
        app: web-affinity
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: tier
                    operator: In
                    values:
                      - frontend
                      - general
      containers:
        - name: nginx
          image: nginx:1.25
EOF

# Ex 3: Static Pod
# Criar manifest em /etc/kubernetes/manifests/
ssh node-worker-1
cat <<EOF > /etc/kubernetes/manifests/static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
spec:
  containers:
    - name: web
      image: nginx:1.25
      ports:
        - containerPort: 80
EOF
# kubelet detecta automaticamente

# Ex 4: DaemonSet
kubectl create deployment monitor-agent --image=busybox:1.36 \
  --dry-run=client -o yaml | \
  sed 's/kind: Deployment/kind: DaemonSet/' | \
  sed '/replicas/d' | \
  sed '/strategy/d' | \
  kubectl apply -f -

# Ex 5: LimitRange
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: ckad-practice
spec:
  limits:
    - type: Container
      default:
        cpu: 250m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
EOF
```

**Critérios de aceite:**
- [ ] Taint aplicado + Pod com toleration (< 3 min)
- [ ] Node affinity configurada (< 3 min)
- [ ] Static Pod criado em /etc/kubernetes/manifests (< 2 min)
- [ ] DaemonSet criado (< 3 min)
- [ ] LimitRange aplicado e verificado (< 2 min)

---

### Desafio 13.3 — Services & Networking (20%)

Pratique networking de cluster e exposição de serviços.

**Tópicos da prova:**
- Services (ClusterIP, NodePort, LoadBalancer)
- Ingress controllers
- CoreDNS
- Network Policies
- CNI basics

**Exercícios no estilo da prova:**
```bash
# Ex 1: Troubleshoot DNS
kubectl run dns-debug --rm -it --image=busybox:1.36 -- nslookup kubernetes.default
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Ex 2: CoreDNS ConfigMap
kubectl get configmap coredns -n kube-system -o yaml

# Ex 3: Ingress com múltiplos paths
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-path
  namespace: ckad-practice
spec:
  ingressClassName: nginx
  rules:
    - host: app.cka.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
EOF

# Ex 4: Network Policy — isolar namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: ckad-practice
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}  # apenas pods do mesmo namespace
EOF

# Ex 5: Verificar kube-proxy mode
kubectl logs -n kube-system -l k8s-app=kube-proxy | grep "Using"
# iptables ou ipvs
```

**Critérios de aceite:**
- [ ] DNS troubleshooting realizado (< 3 min)
- [ ] CoreDNS ConfigMap entendido
- [ ] Ingress multi-path criado (< 3 min)
- [ ] Network Policy para isolamento de namespace (< 3 min)
- [ ] kube-proxy mode identificado

---

### Desafio 13.4 — Storage (10%)

Pratique gerenciamento de storage persistente.

**Tópicos da prova:**
- PV, PVC, StorageClass
- Volume modes e access modes
- Expansion de volumes
- Storage provisioning

**Exercícios no estilo da prova:**
```bash
# Ex 1: Criar PV e PVC manualmente
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-manual
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/pv-manual
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-manual
  namespace: ckad-practice
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  # Sem storageClassName → match manual com PV
EOF

# Verificar binding
kubectl get pv,pvc -n ckad-practice

# Ex 2: Usar PVC em Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
  namespace: ckad-practice
spec:
  containers:
    - name: app
      image: nginx:1.25
      volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc-manual
EOF

# Ex 3: StorageClass com provisioner
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF

# Ex 4: PVC com StorageClass (dynamic provisioning)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
  namespace: ckad-practice
spec:
  storageClassName: fast
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
EOF
```

**Critérios de aceite:**
- [ ] PV e PVC manuais criados e bound (< 3 min)
- [ ] PVC montado em Pod (< 2 min)
- [ ] StorageClass criada (< 2 min)
- [ ] Dynamic provisioning funciona
- [ ] Access modes explicados (RWO, ROX, RWX)

---

### Desafio 13.5 — Troubleshooting (30%)

Pratique troubleshooting — o domínio de maior peso na CKA.

**Tópicos da prova:**
- Cluster component failures
- Node troubleshooting
- Networking troubleshooting
- Application troubleshooting

**Exercícios no estilo da prova:**
```bash
# === Cenário 1: Kubelet não está rodando ===
# Sintoma: node NotReady
kubectl get nodes  # node-worker-2 NotReady

ssh node-worker-2
systemctl status kubelet
journalctl -u kubelet -f
# Diagnóstico: kubelet parado ou crashando
systemctl start kubelet
systemctl enable kubelet

# === Cenário 2: Pod Pending — insufficient resources ===
kubectl describe pod <pending-pod> -n cloudshop-prod
# Events: "Insufficient cpu" ou "Insufficient memory"
# Solução: reduzir requests ou adicionar nodes

# === Cenário 3: Service não conecta ao Pod ===
kubectl get endpoints <service-name> -n cloudshop-prod
# Se ENDPOINTS está vazio → selector do Service não match com Pod labels
kubectl get svc <service> -o jsonpath='{.spec.selector}'
kubectl get pods -l <selector-key>=<selector-value> -n cloudshop-prod

# === Cenário 4: Control plane component down ===
kubectl get pods -n kube-system
# Se kube-apiserver, kube-scheduler, ou kube-controller-manager não está Running
kubectl describe pod kube-scheduler-control-plane -n kube-system
kubectl logs kube-scheduler-control-plane -n kube-system

# Verificar manifests (static pods)
ls /etc/kubernetes/manifests/
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# === Cenário 5: Certificado expirado ===
kubeadm certs check-expiration
kubeadm certs renew all

# === Cenário 6: Network not working ===
# Verificar CNI plugin
ls /etc/cni/net.d/
kubectl get pods -n kube-system | grep -i calico  # ou flannel, cilium
kubectl logs -n kube-system <cni-pod>

# === Cenário 7: Worker node join falha ===
# No control plane, gerar novo token
kubeadm token create --print-join-command
# No worker, executar o join command
```

**Critérios de aceite:**
- [ ] Node NotReady diagnosticado (kubelet) (< 5 min)
- [ ] Pod Pending diagnosticado (resources/scheduling) (< 3 min)
- [ ] Service sem endpoints diagnosticado (< 3 min)
- [ ] Control plane component failure diagnosticado (< 5 min)
- [ ] Certificados verificados
- [ ] CNI troubleshooting executado
- [ ] Runbook de troubleshooting documentado

---

### Desafio 13.6 — Simulado CKA

Execute simulado prático completo.

**Requisitos:**
- Ambiente com múltiplos clusters (kind)
- 15-20 questões hands-on
- 120 minutos
- Apenas kubernetes.io/docs

**Dicas específicas CKA:**
```bash
# CKA geralmente tem múltiplos clusters — troque contexto!
kubectl config get-contexts
kubectl config use-context <cluster-name>

# Essencial para CKA:
# 1. etcdctl com certificados
# 2. kubeadm upgrade process
# 3. Static pods em /etc/kubernetes/manifests
# 4. journalctl -u kubelet para troubleshooting
# 5. kubectl drain/cordon/uncordon
# 6. Certificados: /etc/kubernetes/pki

# Bookmark essenciais:
# - kubernetes.io/docs/reference/kubectl/cheatsheet
# - kubernetes.io/docs/tasks/administer-cluster
# - kubernetes.io/docs/setup/production-environment/tools/kubeadm
```

**Critérios de aceite:**
- [ ] Simulado completo (120 min)
- [ ] Score ≥ 66%
- [ ] Contextos de cluster alternados corretamente
- [ ] etcd backup/restore praticado
- [ ] Troubleshooting de 5+ cenários

---

## Definição de Pronto (DoD)

- [ ] 5 domínios praticados com exercícios hands-on
- [ ] Troubleshooting (30%) dominado
- [ ] etcd operations praticado
- [ ] Simulado com score ≥ 66%
- [ ] Pronto para agendar CKA

---

## Checklist

- [ ] Cluster Architecture — kubeadm, etcd, RBAC
- [ ] Workloads — taints, tolerations, affinity, static pods
- [ ] Services & Networking — DNS, Ingress, NetworkPolicy
- [ ] Storage — PV, PVC, StorageClass
- [ ] Troubleshooting — Node, Pod, Service, Control Plane
- [ ] 2+ simulados com score ≥ 66%
