# Level 9 — Configuration Management

> **Objetivo:** Dominar gerenciamento de configuração no Kubernetes — ConfigMaps, Secrets,
> External Secrets, Sealed Secrets, Helm values e config reload automático.

**Referência:** [Configuration Best Practices](../../.docs/k8s/09-configuration.md)

---

## Objetivo de Aprendizado

- Criar ConfigMaps com diferentes métodos de injeção (env, volume, envFrom)
- Gerenciar Secrets de forma segura (External Secrets, Sealed Secrets)
- Usar ConfigMaps imutáveis para performance
- Implementar config reload automático (Reloader)
- Estruturar Helm values para multi-environment
- Separar configuração de código

---

## Contexto do Domínio

A plataforma CloudShop tem configurações que variam por ambiente (URLs, feature flags, tuning), e secrets sensíveis (credentials de banco, API keys, certificados). A gestão segura e automatizada dessas configurações é crítica para segurança e operação.

---

## Desafios

### Desafio 9.1 — ConfigMaps

Crie ConfigMaps para todos os microsserviços com injeção via env e volume.

**Requisitos:**
- ConfigMap com dados de configuração (env vars)
- ConfigMap com arquivo de configuração (volume mount)
- `envFrom` para injeção em massa
- ConfigMap imutável para configurações estáticas

```yaml
# ConfigMap — key-value (env vars)
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: cloudshop-prod
data:
  DATABASE_HOST: "postgresql.cloudshop-data.svc.cluster.local"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "cloudshop"
  REDIS_HOST: "redis.cloudshop-data.svc.cluster.local"
  REDIS_PORT: "6379"
  LOG_LEVEL: "info"
  FEATURE_ASYNC_PAYMENTS: "true"
---
# ConfigMap — arquivo de configuração
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-app-config
  namespace: cloudshop-prod
data:
  application.yaml: |
    server:
      port: 8080
      shutdown: graceful
    spring:
      datasource:
        url: jdbc:postgresql://${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_NAME}
      redis:
        host: ${REDIS_HOST}
        port: ${REDIS_PORT}
    management:
      endpoints:
        web:
          exposure:
            include: health,info,metrics,prometheus
---
# ConfigMap imutável (não pode ser alterado)
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-static-v1
  namespace: cloudshop-prod
immutable: true
data:
  APP_VERSION: "1.2.0"
  CLUSTER_NAME: "cloudshop-prod"
```

```yaml
# Deployment usando ConfigMaps
spec:
  containers:
    - name: order-service
      # envFrom — injetar todas as keys como env vars
      envFrom:
        - configMapRef:
            name: order-service-config
      # env — injetar keys específicas
      env:
        - name: APP_VERSION
          valueFrom:
            configMapKeyRef:
              name: order-service-static-v1
              key: APP_VERSION
      # volume — montar arquivo
      volumeMounts:
        - name: app-config
          mountPath: /app/config
          readOnly: true
  volumes:
    - name: app-config
      configMap:
        name: order-service-app-config
```

**Critérios de aceite:**
- [ ] ConfigMap com env vars para cada microsserviço
- [ ] ConfigMap com arquivo de configuração (volume mount)
- [ ] `envFrom` usado para injeção em massa
- [ ] ConfigMap imutável para dados estáticos
- [ ] Env vars verificadas: `kubectl exec <pod> -- env | grep DATABASE`

---

### Desafio 9.2 — Secrets Management

Gerencie Secrets de forma segura com External Secrets Operator.

**Requisitos:**
- External Secrets Operator instalado
- SecretStore apontando para vault (ou AWS SSM/Secrets Manager)
- ExternalSecret para credentials do banco
- Secrets montados como volume (não env vars para dados sensíveis)
- automountServiceAccountToken: false

```yaml
# SecretStore — conectar ao backend de secrets
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: cloudshop-prod
spec:
  provider:
    vault:
      server: "https://vault.internal:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "cloudshop-prod"
---
# ExternalSecret — puxar credentials do vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service-db
  namespace: cloudshop-prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: order-service-db-credentials
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: cloudshop/prod/database
        property: username
    - secretKey: password
      remoteRef:
        key: cloudshop/prod/database
        property: password
```

```yaml
# Montar secret como volume (mais seguro que env var)
spec:
  automountServiceAccountToken: false
  containers:
    - name: order-service
      volumeMounts:
        - name: db-credentials
          mountPath: /etc/secrets/db
          readOnly: true
  volumes:
    - name: db-credentials
      secret:
        secretName: order-service-db-credentials
        defaultMode: 0400  # read-only owner
        items:
          - key: username
            path: username
          - key: password
            path: password
```

**Critérios de aceite:**
- [ ] External Secrets Operator instalado
- [ ] SecretStore configurado
- [ ] ExternalSecret sincronizando com backend
- [ ] Secret montado como volume (não env var)
- [ ] `defaultMode: 0400` para permissões restritas
- [ ] `automountServiceAccountToken: false`
- [ ] `kubectl get externalsecret` mostra status SecretSynced

---

### Desafio 9.3 — Sealed Secrets (Alternativa)

Configure Sealed Secrets para armazenar secrets criptografados no Git.

**Requisitos:**
- Instalar Sealed Secrets controller
- Criar SealedSecret (criptografado com chave pública do cluster)
- Commitar SealedSecret no Git (seguro!)
- Controller decripta e cria Secret no cluster

```bash
# Instalar controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.0/controller.yaml

# Instalar kubeseal CLI
# (download de https://github.com/bitnami-labs/sealed-secrets/releases)

# Criar secret normal
kubectl create secret generic order-service-db \
  --from-literal=username=cloudshop \
  --from-literal=password=super-secret-123 \
  --namespace cloudshop-prod \
  --dry-run=client -o yaml > secret.yaml

# Seal (encriptar com chave pública do cluster)
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# O sealed-secret.yaml é SEGURO para commitar no Git!
cat sealed-secret.yaml
```

```yaml
# SealedSecret (encriptado — seguro para Git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: order-service-db
  namespace: cloudshop-prod
spec:
  encryptedData:
    username: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
    password: AgBjY3cY3JzPGvJI+0PdLzX3e7s2v2h3...
  template:
    metadata:
      name: order-service-db
      namespace: cloudshop-prod
```

**Critérios de aceite:**
- [ ] Sealed Secrets controller instalado
- [ ] SealedSecret criado via kubeseal
- [ ] SealedSecret commitado no Git (demonstrar segurança)
- [ ] Controller decripta → Secret criado no cluster
- [ ] Secret acessível pelo Pod
- [ ] Comparação documentada: External Secrets vs Sealed Secrets

---

### Desafio 9.4 — Config Reload Automático

Configure reload automático de ConfigMaps e Secrets usando Reloader.

**Requisitos:**
- Instalar Reloader
- Annotation para auto-reload quando ConfigMap muda
- Testar: alterar ConfigMap → Pod reinicia automaticamente
- Estratégia: rolling restart (sem downtime)

```bash
# Instalar Reloader
kubectl apply -f https://raw.githubusercontent.com/stakater/Reloader/master/deployments/kubernetes/reloader.yaml
```

```yaml
# Deployment com annotation para Reloader
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: cloudshop-prod
  annotations:
    reloader.stakater.com/auto: "true"  # reload em qualquer ConfigMap/Secret referenciado
    # OU especificar quais:
    # configmap.reloader.stakater.com/reload: "order-service-config"
    # secret.reloader.stakater.com/reload: "order-service-db-credentials"
spec:
  template:
    # ...
```

```bash
# Testar: alterar ConfigMap
kubectl edit configmap order-service-config -n cloudshop-prod
# Mudar LOG_LEVEL de "info" para "debug"

# Verificar que Reloader reiniciou o Deployment
kubectl get events -n cloudshop-prod --field-selector reason=Reloaded

# Verificar que Pods foram recriados
kubectl get pods -n cloudshop-prod -l app.kubernetes.io/name=order-service -w
```

**Critérios de aceite:**
- [ ] Reloader instalado e funcional
- [ ] Annotation `reloader.stakater.com/auto: "true"` nos Deployments
- [ ] ConfigMap alterado → Pod reinicia automaticamente
- [ ] Secret alterado → Pod reinicia automaticamente
- [ ] Rolling restart (sem downtime)
- [ ] Event de Reloader visível

---

### Desafio 9.5 — Helm Values Multi-Environment

Estruture Helm values para gerenciar configuração por ambiente.

**Requisitos:**
- values.yaml com defaults
- values-dev.yaml, values-staging.yaml, values-prod.yaml
- Usar Helm secrets para dados sensíveis
- Validação de values com JSON Schema

```yaml
# values.yaml (defaults)
replicaCount: 1
image:
  repository: registry.cloudshop.io/order-service
  tag: latest
config:
  logLevel: info
  database:
    host: postgresql
    port: 5432
    name: cloudshop
  redis:
    host: redis
    port: 6379
  features:
    asyncPayments: false
    emailNotifications: false
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
---
# values-dev.yaml (sobrescreve defaults)
replicaCount: 1
config:
  logLevel: debug
  features:
    asyncPayments: true
    emailNotifications: false
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 250m
    memory: 256Mi
---
# values-prod.yaml (sobrescreve defaults)
replicaCount: 3
config:
  logLevel: warn
  features:
    asyncPayments: true
    emailNotifications: true
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi
```

```bash
# Template render com values de prod
helm template order-service charts/microservice \
  -f charts/microservice/values.yaml \
  -f charts/microservice/values-prod.yaml \
  --set image.tag=v1.2.0

# Install com values de prod
helm install order-service charts/microservice \
  -n cloudshop-prod \
  -f charts/microservice/values-prod.yaml

# Diff antes de upgrade
helm diff upgrade order-service charts/microservice \
  -n cloudshop-prod \
  -f charts/microservice/values-prod.yaml
```

**Critérios de aceite:**
- [ ] values.yaml com defaults sensatos
- [ ] values por ambiente com overrides
- [ ] Prod tem mais réplicas, mais resources, features habilitadas
- [ ] Dev tem debug logging, menos réplicas
- [ ] `helm template` gera YAML válido com cada values

---

### Desafio 9.6 — Configuration Audit

Verifique e audite todas as configurações para consistência e segurança.

**Requisitos:**
- Verificar ConfigMaps sem referência (órfãos)
- Verificar Secrets sem referência
- Verificar env vars sensíveis que deveriam ser Secrets
- Policy: proibir senhas em ConfigMaps

```bash
# Listar ConfigMaps
kubectl get configmaps -n cloudshop-prod

# Listar Secrets
kubectl get secrets -n cloudshop-prod

# Verificar referências em Pods
kubectl get pods -n cloudshop-prod -o json | \
  jq '.items[].spec.containers[].envFrom[]?.configMapRef.name' | sort -u

# Verificar env vars que parecem sensíveis
kubectl get pods -n cloudshop-prod -o json | \
  jq '.items[].spec.containers[].env[]? | select(.name | test("PASSWORD|SECRET|KEY|TOKEN")) | select(.valueFrom.secretKeyRef == null) | .name'
# ⚠️ Se retornar algo → dado sensível exposto como plain text!
```

```yaml
# Kyverno Policy — proibir senhas em ConfigMaps
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-secrets-in-configmaps
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-configmap-data
      match:
        any:
          - resources:
              kinds:
                - ConfigMap
      validate:
        message: "ConfigMaps não podem conter dados sensíveis (PASSWORD, SECRET, KEY)"
        deny:
          conditions:
            any:
              - key: "{{ request.object.data | keys(@) }}"
                operator: AnyIn
                value:
                  - PASSWORD
                  - SECRET
                  - API_KEY
                  - TOKEN
```

**Critérios de aceite:**
- [ ] ConfigMaps órfãos identificados (sem referência)
- [ ] Secrets órfãos identificados
- [ ] Nenhuma env var sensível em plain text
- [ ] Kyverno policy proíbe senhas em ConfigMaps
- [ ] Relatório de audit gerado

---

## Definição de Pronto (DoD)

- [ ] ConfigMaps para todos os microsserviços (env + volume)
- [ ] Secrets gerenciados via External Secrets ou Sealed Secrets
- [ ] ConfigMaps imutáveis para dados estáticos
- [ ] Reloader para auto-reload de configuração
- [ ] Helm values estruturados por ambiente
- [ ] Audit de configuração executado
- [ ] Commit: `feat(level-9): add configuration management`

---

## Checklist

- [ ] ConfigMaps com envFrom e volume mount
- [ ] ConfigMaps imutáveis
- [ ] External Secrets Operator (ou Sealed Secrets)
- [ ] Secrets montados como volume (não env var)
- [ ] Reloader com annotation auto
- [ ] Helm values por ambiente
- [ ] Nenhum dado sensível em plain text
- [ ] Kyverno policy para ConfigMaps
- [ ] ConfigMaps/Secrets órfãos limpos
