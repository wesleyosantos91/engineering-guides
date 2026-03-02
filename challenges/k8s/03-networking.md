# Level 3 — Networking

> **Objetivo:** Dominar networking no Kubernetes — Services, Ingress, Gateway API, DNS, Service Mesh
> e Load Balancing — para expor e conectar os microsserviços da plataforma CloudShop.

**Referência:** [Networking Best Practices](../../.docs/k8s/03-networking.md)

---

## Objetivo de Aprendizado

- Criar Services (ClusterIP, NodePort, LoadBalancer, Headless)
- Configurar Ingress com TLS e cert-manager
- Implementar Gateway API para traffic management avançado
- Otimizar DNS (CoreDNS, ndots, NodeLocal DNSCache)
- Avaliar e implementar Service Mesh (Istio/Linkerd) para mTLS
- Debugar problemas de networking

---

## Contexto do Domínio

A plataforma CloudShop precisa expor suas APIs via Ingress com TLS, rotear tráfego entre versões (canary), configurar mTLS entre serviços e otimizar DNS para alta performance.

---

## Desafios

### Desafio 3.1 — Services

Crie Services para todos os microsserviços da CloudShop com tipos adequados.

**Requisitos:**
- ClusterIP para comunicação interna (order-service, product-service, etc.)
- Headless para StatefulSets (postgresql, redis)
- Portas nomeadas em todos os Services
- `targetPort` usando nome (não número)

```yaml
# ClusterIP — order-service
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: cloudshop-prod
  labels:
    app.kubernetes.io/name: order-service
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: order-service
  ports:
    - name: http
      port: 80
      targetPort: http  # referencia a named port no container
      protocol: TCP
---
# Headless — postgresql
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  namespace: cloudshop-data
spec:
  clusterIP: None
  selector:
    app: postgresql
  ports:
    - name: postgres
      port: 5432
```

**Critérios de aceite:**
- [ ] ClusterIP Services para 6 microsserviços
- [ ] Headless Services para PostgreSQL e Redis
- [ ] Todas as portas nomeadas
- [ ] `kubectl get endpoints` mostra IPs dos Pods
- [ ] DNS funciona: `curl order-service.cloudshop-prod.svc.cluster.local`

---

### Desafio 3.2 — Ingress com TLS

Configure Ingress Controller (NGINX) com TLS terminado via cert-manager.

**Requisitos:**
- Instalar NGINX Ingress Controller
- Instalar cert-manager com ClusterIssuer (Let's Encrypt ou self-signed)
- Ingress com TLS para `api.cloudshop.local`
- Roteamento por path: `/api/v1/orders` → order-service, `/api/v1/products` → product-service
- Rate limiting e segurança via annotations

```yaml
# ClusterIssuer (self-signed para dev)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cloudshop-api
  namespace: cloudshop-prod
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/rate-limit-connections: "10"
    nginx.ingress.kubernetes.io/rate-limit-rps: "100"
    cert-manager.io/cluster-issuer: selfsigned-issuer
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.cloudshop.local
      secretName: cloudshop-api-tls
  rules:
    - host: api.cloudshop.local
      http:
        paths:
          - path: /api/v1/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
          - path: /api/v1/products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 80
          - path: /api/v1/users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
```

**Critérios de aceite:**
- [ ] NGINX Ingress Controller instalado e Running
- [ ] cert-manager instalado com ClusterIssuer
- [ ] Certificado TLS gerado automaticamente
- [ ] Roteamento por path funciona corretamente
- [ ] Rate limiting ativo (annotations NGINX)
- [ ] HTTP → HTTPS redirect funciona

---

### Desafio 3.3 — Gateway API

Implemente Gateway API para traffic management avançado (canary deploy).

**Requisitos:**
- Gateway com HTTPS listener
- HTTPRoute com traffic splitting (90/10 canary)
- Header-based routing para debug

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cloudshop-gateway
  namespace: cloudshop-prod
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: cloudshop-api-tls
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
  name: order-canary
  namespace: cloudshop-prod
spec:
  parentRefs:
    - name: cloudshop-gateway
  hostnames:
    - "api.cloudshop.local"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api/v1/orders
      backendRefs:
        - name: order-service
          port: 80
          weight: 90
        - name: order-service-canary
          port: 80
          weight: 10
    - matches:
        - headers:
            - name: x-canary
              value: "true"
          path:
            type: PathPrefix
            value: /api/v1/orders
      backendRefs:
        - name: order-service-canary
          port: 80
```

**Critérios de aceite:**
- [ ] Gateway API CRDs instalados
- [ ] Gateway com HTTPS listener
- [ ] Traffic splitting 90/10 funciona
- [ ] Header-based routing funciona (`x-canary: true`)
- [ ] Métricas de cada backend monitoradas

---

### Desafio 3.4 — DNS Optimization

Otimize DNS para performance em alta escala.

**Requisitos:**
- Reduzir `ndots` de 5 para 2
- Usar FQDN com ponto final para evitar search domains
- Configurar CoreDNS com cache e monitoramento

```yaml
# Pod com dnConfig otimizado
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    options:
      - name: ndots
        value: "2"  # reduzir queries DNS desnecessárias
```

```bash
# Testar resolução DNS
kubectl run dns-test --rm -it --image=nicolaka/netshoot -- bash
nslookup order-service.cloudshop-prod.svc.cluster.local
dig +search order-service
dig order-service.cloudshop-prod.svc.cluster.local. +short  # FQDN com ponto

# Monitorar CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

**Critérios de aceite:**
- [ ] `ndots: 2` configurado nos Pods
- [ ] Resolução DNS funciona para todos os Services
- [ ] CoreDNS com cache habilitado
- [ ] Métricas do CoreDNS expostas para Prometheus
- [ ] Documentado: quando usar FQDN vs short name

---

### Desafio 3.5 — Service Mesh (Opcional/Avançado)

Implemente mTLS entre serviços usando Istio ou Linkerd.

**Requisitos:**
- Instalar service mesh (Istio sidecar ou Linkerd)
- habilitar mTLS strict no namespace de produção
- Configurar traffic management (timeout, retry, circuit breaker)
- Observabilidade L7 entre serviços

```yaml
# Istio — mTLS strict
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: cloudshop-prod
spec:
  mtls:
    mode: STRICT
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: order-service
  namespace: cloudshop-prod
spec:
  host: order-service
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

**Critérios de aceite:**
- [ ] Service mesh instalado e sidecars injetados
- [ ] mTLS strict habilitado
- [ ] Comunicação sem mesh é bloqueada
- [ ] Dashboard de observabilidade L7 funciona
- [ ] Outlier detection configurado

---

### Desafio 3.6 — Network Troubleshooting

Pratique debugging de problemas de rede comuns.

**Requisitos:**
- Simular e diagnosticar: DNS timeout, Connection refused, 503 errors
- Usar ferramentas: `netshoot`, `dig`, `curl`, `nslookup`
- Verificar endpoints, network policies e connectivity

```bash
# Deploy de debug pod
kubectl run debug --rm -it --image=nicolaka/netshoot -n cloudshop-prod -- bash

# DNS issues
nslookup order-service.cloudshop-prod.svc.cluster.local
dig +search order-service

# Connectivity
curl -v http://order-service:8080/health
curl -v http://postgresql.cloudshop-data:5432

# Endpoints
kubectl get endpoints order-service -n cloudshop-prod
kubectl get endpointslices -n cloudshop-prod

# Network Policies
kubectl get networkpolicies -n cloudshop-prod -o wide
kubectl describe networkpolicy default-deny-all -n cloudshop-prod

# Ephemeral debug container
kubectl debug pod/order-service-xxx -it --image=nicolaka/netshoot -n cloudshop-prod
```

**Critérios de aceite:**
- [ ] DNS debugging demonstrado (nslookup, dig)
- [ ] Connectivity testing demonstrado (curl)
- [ ] Endpoints verificados (service → pods)
- [ ] Network policy impact diagnosticado
- [ ] Problema simulado → diagnosticado → resolvido

---

## Definição de Pronto (DoD)

- [ ] Services para todos os microsserviços + databases
- [ ] Ingress com TLS e roteamento por path
- [ ] Gateway API com canary deploy (traffic splitting)
- [ ] DNS otimizado (ndots: 2)
- [ ] Network troubleshooting praticado
- [ ] Commit: `feat(level-3): add networking configuration`

---

## Checklist

- [ ] 6 ClusterIP Services + 2 Headless Services
- [ ] NGINX Ingress Controller instalado
- [ ] cert-manager instalado com ClusterIssuer
- [ ] Ingress com TLS configurado
- [ ] Gateway API com traffic splitting
- [ ] DNS otimizado (ndots: 2, CoreDNS cache)
- [ ] Debug pod testado (netshoot)
- [ ] Service mesh avaliado (opcional: instalado)
- [ ] Network troubleshooting praticado
