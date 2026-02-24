# CKA — Certified Kubernetes Administrator

## Informações do Exame

| Item | Detalhe |
|------|---------|
| **Formato** | Hands-on (terminal Linux) |
| **Duração** | 2 horas |
| **Questões** | 15-20 tarefas |
| **Passing Score** | 66% |
| **Validade** | 2 anos |
| **Pré-requisitos** | Nenhum |
| **Preço** | $395 USD (inclui 1 retake + killer.sh) |
| **Ambiente** | PSI Bridge, 1 aba extra para docs |

## Domínios do Exame

| Domínio | Peso | Tópicos |
|---------|:---:|---------|
| **Cluster Architecture, Installation & Configuration** | 25% | RBAC, kubeadm, upgrade, etcd backup/restore |
| **Workloads & Scheduling** | 15% | Deployments, scaling, scheduling, resource limits |
| **Services & Networking** | 20% | Services, Ingress, DNS, NetworkPolicy |
| **Storage** | 10% | PV, PVC, StorageClass |
| **Troubleshooting** | 30% | Cluster/node/app troubleshooting, networking |

## Setup Inicial
```bash
alias k=kubectl
alias kn='kubectl config set-context --current --namespace'
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
```

## Conteúdo por Domínio

### 1. Cluster Architecture, Installation & Configuration (25%)

#### kubeadm — Instalação do Cluster
```bash
# Inicializar control plane
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.1.10

# Configurar kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Instalar CNI (ex: Calico)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Join worker nodes
kubeadm join 192.168.1.10:6443 --token abc123 \
  --discovery-token-ca-cert-hash sha256:xxx
```

#### kubeadm — Upgrade do Cluster
```bash
# 1. Upgrade control plane
sudo apt update
sudo apt install -y kubeadm=1.31.0-*
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.31.0

# 2. Upgrade kubelet e kubectl no control plane
sudo apt install -y kubelet=1.31.0-* kubectl=1.31.0-*
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 3. Upgrade worker nodes (um por vez)
kubectl drain worker-1 --ignore-daemonsets
# No worker node:
sudo apt install -y kubeadm=1.31.0-*
sudo kubeadm upgrade node
sudo apt install -y kubelet=1.31.0-*
sudo systemctl daemon-reload
sudo systemctl restart kubelet
# No control plane:
kubectl uncordon worker-1
```

#### RBAC
```bash
# Criar Role
k create role pod-reader --verb=get,list,watch --resource=pods -n production

# Criar RoleBinding
k create rolebinding dev-pod-reader --role=pod-reader --user=dev-user -n production

# ClusterRole
k create clusterrole node-reader --verb=get,list,watch --resource=nodes

# ClusterRoleBinding
k create clusterrolebinding admin-nodes --clusterrole=node-reader --user=admin

# Verificar permissões
k auth can-i list pods --as dev-user -n production
k auth can-i '*' '*' --as system:serviceaccount:default:mysa
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-manager
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
```

#### etcd Backup e Restore
```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verificar backup
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-out=table

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd-from-backup

# Atualizar etcd para usar novo data-dir
# Editar /etc/kubernetes/manifests/etcd.yaml
# Mudar --data-dir e hostPath para /var/lib/etcd-from-backup
```

#### Certificates
```bash
# Verificar certificados
kubeadm certs check-expiration

# Renovar certificados
kubeadm certs renew all

# Gerar CSR para novo usuário
openssl genrsa -out dev-user.key 2048
openssl req -new -key dev-user.key -out dev-user.csr -subj "/CN=dev-user/O=developers"
```

```yaml
# CertificateSigningRequest
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: dev-user
spec:
  request: $(cat dev-user.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
    - client auth
```

```bash
k certificate approve dev-user
k get csr dev-user -o jsonpath='{.status.certificate}' | base64 -d > dev-user.crt
```

### 2. Workloads & Scheduling (15%)

#### Scheduling
```yaml
# nodeSelector — simples
spec:
  nodeSelector:
    disktype: ssd

# nodeAffinity — avançado
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]

# Taints e Tolerations
# kubectl taint nodes node1 key=value:NoSchedule
spec:
  tolerations:
    - key: "key"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"
```

#### Static Pods
```bash
# Static pod path (configurado no kubelet)
cat /var/lib/kubelet/config.yaml | grep staticPodPath
# Default: /etc/kubernetes/manifests/

# Criar static pod
kubectl run static-nginx --image=nginx $do > /etc/kubernetes/manifests/static-nginx.yaml
```

#### Resource Management
```yaml
# LimitRange
apiVersion: v1
kind: LimitRange
metadata:
  name: limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi

# ResourceQuota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
  namespace: production
spec:
  hard:
    pods: "50"
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
```

### 3. Services & Networking (20%)

#### Services
```bash
k expose deployment nginx --port=80 --type=ClusterIP --name=nginx-svc
k expose deployment nginx --port=80 --type=NodePort --name=nginx-np
```

```yaml
# ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080

# NodePort
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080  # 30000-32767
```

#### Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
```

#### CoreDNS & DNS
```bash
# Verificar DNS
k run dns-test --rm -it --image=busybox -- nslookup kubernetes
k run dns-test --rm -it --image=busybox -- nslookup myservice.production.svc.cluster.local

# DNS format
<service>.<namespace>.svc.cluster.local
<pod-ip-dashed>.<namespace>.pod.cluster.local
```

#### Network Policies
```yaml
# Deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress

# Allow specific
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
spec:
  podSelector:
    matchLabels:
      role: api
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
        - namespaceSelector:
            matchLabels:
              purpose: production
      ports:
        - port: 8080
```

### 4. Storage (10%)

```yaml
# PersistentVolume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/pv

---
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 5Gi

---
# Uso no Pod
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc-data
  containers:
    - name: app
      volumeMounts:
        - name: data
          mountPath: /data
```

```yaml
# StorageClass com dynamic provisioning
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/no-provisioner  # ou cloud provider
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

### 5. Troubleshooting (30%)

#### Cluster Troubleshooting
```bash
# Control plane components (static pods)
ls /etc/kubernetes/manifests/
# kube-apiserver.yaml, kube-controller-manager.yaml, kube-scheduler.yaml, etcd.yaml

# Verificar logs
sudo journalctl -u kubelet -f
k logs -n kube-system kube-apiserver-master
k logs -n kube-system kube-scheduler-master
k logs -n kube-system kube-controller-manager-master

# Verificar status
k get nodes
k describe node node-1
k get componentstatuses  # deprecated mas ainda informativo
```

#### Node Troubleshooting
```bash
# Node NotReady
k describe node worker-1 | grep -A5 Conditions
# Verificar kubelet
ssh worker-1
sudo systemctl status kubelet
sudo journalctl -u kubelet --no-pager | tail -50
sudo systemctl restart kubelet
```

#### Application Troubleshooting
```bash
# Fluxo de diagnóstico
k get pods -o wide                    # Status dos pods
k describe pod myapp-xxx              # Events e condições
k logs myapp-xxx                      # Logs da aplicação
k logs myapp-xxx --previous           # Logs do crash anterior
k exec -it myapp-xxx -- sh            # Entrar no container
k get endpoints myapp-svc             # Verificar Service endpoints
k get events --sort-by=.lastTimestamp  # Timeline de eventos
```

#### Common Issues Checklist
```
Pod Pending:
  ☐ Resources insuficientes → k describe node
  ☐ Node selector/affinity sem match
  ☐ PVC não bound
  ☐ Taints sem toleration

Pod CrashLoopBackOff:
  ☐ Verificar logs → k logs --previous
  ☐ Command/args errados
  ☐ Missing config (env vars, volumes)
  ☐ Liveness probe muito agressiva

Service sem conectividade:
  ☐ Labels do selector batem com Pod?
  ☐ Endpoints criados? → k get ep
  ☐ Porta do targetPort correta?
  ☐ NetworkPolicy bloqueando?

Node NotReady:
  ☐ kubelet rodando? → systemctl status kubelet
  ☐ Certificados válidos?
  ☐ Container runtime ok? → systemctl status containerd
  ☐ Disco cheio?
```

## Dicas Específicas para CKA

1. **Troubleshooting é 30%** — pratique muito cenários de problema.
2. **etcd backup/restore** — cai quase sempre, saiba de cor.
3. **kubeadm upgrade** — pratique o fluxo completo.
4. **RBAC** — Role, RoleBinding, ClusterRole, ClusterRoleBinding.
5. **Network Policies** — understand ingress AND egress.
6. **Static Pods** — saiba onde ficam e como criar.
7. **Certificates** — entenda o fluxo de CSR.
8. **journalctl** — essencial para debug de kubelet.
9. **Contextos** — sempre mude para o contexto correto da questão.
10. **killer.sh** — mais difícil que o exame real, ótima preparação.

## Recursos de Estudo

- [CKA Curriculum](https://github.com/cncf/curriculum)
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- KodeKloud CKA Course (Mumshad)
- Udemy CKA Course (Mumshad)
- killer.sh (simulador oficial)
- [CKA Exercises (walidshaari)](https://github.com/walidshaari/Kubernetes-Certified-Administrator)
- Killercoda Scenarios
