# Networking — Best Practices

## 1. Services

### 1.1 Tipos de Service
| Tipo | Uso | Quando Usar |
|------|-----|-------------|
| **ClusterIP** | Interno | Comunicação entre Pods no cluster |
| **NodePort** | Externo | Dev/test, ranges 30000-32767 |
| **LoadBalancer** | Externo | Produção com cloud provider |
| **ExternalName** | DNS CNAME | Referenciar serviços externos |
| **Headless** | DNS direto | StatefulSets, service discovery direto |

### 1.2 ClusterIP (Padrão)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
    - name: http  # ✅ sempre nomeie as portas
      port: 80
      targetPort: 8080
      protocol: TCP
```

### 1.3 Headless Service (StatefulSets)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None  # headless
  selector:
    app: postgres
  ports:
    - port: 5432
      name: postgres
# DNS: postgres-0.postgres.namespace.svc.cluster.local
```

### 1.4 Boas Práticas de Services
- **Sempre nomeie as portas** (`name: http`, `name: grpc`).
- Use **`targetPort` referenciando nome** em vez de número para flexibilidade.
- Configure **`sessionAffinity: ClientIP`** quando necessário para sticky sessions.
- Use **`externalTrafficPolicy: Local`** para preservar IP do cliente.
- Prefira **Headless Services** para StatefulSets.

## 2. Ingress

### 2.1 Ingress com NGINX
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/rate-limit-connections: "10"
    nginx.ingress.kubernetes.io/rate-limit-rps: "100"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: api-v1
                port:
                  number: 80
          - path: /api/v2
            pathType: Prefix
            backend:
              service:
                name: api-v2
                port:
                  number: 80
```

### 2.2 Gateway API (Evolução do Ingress)
```yaml
# Gateway API — mais expressivo e extensível
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
spec:
  gatewayClassName: istio  # ou nginx, envoy, etc.
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: api-tls
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway-access: "true"
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
spec:
  parentRefs:
    - name: main-gateway
  hostnames:
    - "api.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-service
          port: 80
          weight: 90
        - name: api-service-canary
          port: 80
          weight: 10  # canary 10%
```

**Gateway API vs Ingress:**
- Gateway API é o **futuro** — mais expressivo, role-oriented.
- Suporte a **traffic splitting** nativo (canary/blue-green).
- Melhor **separação de responsabilidades** (infra vs app teams).
- Use Gateway API para novos projetos quando possível.

### 2.3 Boas Práticas de Ingress/Gateway
- **Sempre TLS** — use cert-manager com Let's Encrypt.
- Configure **rate limiting** para proteger contra DDoS.
- Use **annotations de segurança** (CORS, HSTS, CSP).
- Separe **Ingress Controllers** por ambiente (prod vs non-prod).
- Monitore latência e erros no Ingress Controller.

## 3. DNS

### 3.1 Service Discovery
```
# Formato do DNS interno
<service>.<namespace>.svc.cluster.local

# Exemplos
api.production.svc.cluster.local
postgres-0.postgres.production.svc.cluster.local  # StatefulSet
```

### 3.2 DNS Policies
```yaml
spec:
  dnsPolicy: ClusterFirst  # padrão — usa CoreDNS do cluster
  # ou
  dnsPolicy: None
  dnsConfig:
    nameservers:
      - 8.8.8.8
    searches:
      - production.svc.cluster.local
      - svc.cluster.local
    options:
      - name: ndots
        value: "2"  # ✅ reduzir de 5 (padrão) para 2 melhora performance DNS
```

### 3.3 CoreDNS Optimization
```yaml
# ConfigMap do CoreDNS
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
            max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

**Boas Práticas DNS:**
- Reduza **ndots para 2** (padrão é 5, causa queries desnecessárias).
- Use **NodeLocal DNSCache** para alto throughput DNS.
- Monitore métricas DNS do CoreDNS via Prometheus.
- Use **FQDN com ponto final** (`api.production.svc.cluster.local.`) para evitar search domains.

## 4. Service Mesh

### 4.1 Quando Usar Service Mesh
| Cenário | Service Mesh? |
|---------|:---:|
| Poucos serviços (<10) | Provavelmente não |
| mTLS obrigatório | ✅ |
| Traffic management avançado | ✅ |
| Observabilidade L7 entre serviços | ✅ |
| Requisitos regulatórios (PCI, HIPAA) | ✅ |
| >50 microservices | Considere fortemente |

### 4.2 Opções de Service Mesh
| Mesh | Descrição | Proxy |
|------|-----------|-------|
| **Istio** | Mais completo, maior complexidade | Envoy |
| **Linkerd** | Leve, fácil de operar | linkerd2-proxy (Rust) |
| **Cilium** | eBPF-based, sem sidecar | eBPF |
| **Consul Connect** | Multi-platform | Envoy |

### 4.3 Istio — mTLS
```yaml
# PeerAuthentication — mTLS strict
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

---
# DestinationRule — mTLS para serviço específico
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: api
spec:
  host: api.production.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

## 5. Load Balancing

### 5.1 Estratégias
```yaml
# Kubernetes Service — Round Robin (padrão)
# Para algoritmos avançados, use Service Mesh

# Istio — Consistent Hash (sticky sessions)
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: api
spec:
  host: api
  trafficPolicy:
    loadBalancer:
      consistentHash:
        httpHeaderName: x-user-id
```

### 5.2 External Load Balancer
- Use **annotations do cloud provider** para configurar.
- Configure **health checks** no LB externo.
- Use **`externalTrafficPolicy: Local`** para preservar source IP.
- Configure **connection draining** adequadamente.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  annotations:
    # AWS NLB
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: api
  ports:
    - port: 443
      targetPort: 8080
```

## 6. Network Troubleshooting

### 6.1 Ferramentas Essenciais
```bash
# Debug de DNS
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash
nslookup api.production.svc.cluster.local
dig +search api

# Teste de conectividade
kubectl exec -it debug-pod -- curl -v http://api.production:8080/health

# Verificar endpoints
kubectl get endpoints api -n production

# Verificar Network Policies
kubectl get networkpolicies -n production
kubectl describe networkpolicy default-deny -n production

# Debug com ephemeral containers
kubectl debug pod/myapp-xxx -it --image=nicolaka/netshoot
```

### 6.2 Problemas Comuns
| Problema | Causa Provável | Solução |
|----------|---------------|---------|
| DNS timeout | CoreDNS sobrecarregado | NodeLocal DNSCache, aumentar replicas |
| Connection refused | Service sem endpoints | Verificar labels/selectors |
| 503 errors | Pod not ready | Verificar readiness probe |
| Intermittent failures | Network Policy | Verificar regras de egress/ingress |
| Slow DNS | ndots alto | Reduzir ndots para 2 |
