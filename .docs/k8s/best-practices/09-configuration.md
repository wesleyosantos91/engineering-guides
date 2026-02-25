# Configuration Management — Best Practices

## 1. ConfigMaps

### 1.1 Criando ConfigMaps
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: myapp
data:
  # Valores simples
  LOG_LEVEL: "info"
  DATABASE_HOST: "postgres.production.svc.cluster.local"
  DATABASE_PORT: "5432"
  CACHE_TTL: "300"
  
  # Arquivo de configuração completo
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
    logging:
      level: info
      format: json
    features:
      new-checkout: true
      dark-mode: false
```

### 1.2 Consumindo ConfigMaps
```yaml
spec:
  containers:
    - name: myapp
      # Opção 1: envFrom (todas as chaves como env vars)
      envFrom:
        - configMapRef:
            name: app-config
      
      # Opção 2: env específico
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
      
      # Opção 3: Volume mount (arquivos)
      volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
  volumes:
    - name: config
      configMap:
        name: app-config
        items:
          - key: config.yaml
            path: config.yaml
```

### 1.3 ConfigMap Reload Automático
```yaml
# Opção 1: Reloader (Stakater)
# https://github.com/stakater/Reloader
metadata:
  annotations:
    reloader.stakater.com/auto: "true"  # restart no change

# Opção 2: Hash no template (Helm)
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

### 1.4 Boas Práticas de ConfigMaps
- **Não armazene secrets** em ConfigMaps (use Secrets ou External Secrets).
- Use **volume mount** para arquivos de configuração (hot-reload possível).
- Use **envFrom** com cautela — todas as keys viram env vars.
- Implemente **config reload** automático (Reloader ou similar).
- Nome descritivo com versionamento: `app-config-v2`.
- **Tamanho máximo**: 1MiB por ConfigMap.
- Use **immutable ConfigMaps** quando o conteúdo não muda.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v3
immutable: true  # ✅ mais eficiente, previne mudanças acidentais
data:
  config.yaml: |
    ...
```

## 2. Secrets

### 2.1 Tipos de Secrets
| Tipo | Uso |
|------|-----|
| `Opaque` | Dados genéricos (default) |
| `kubernetes.io/tls` | Certificados TLS |
| `kubernetes.io/dockerconfigjson` | Registry credentials |
| `kubernetes.io/basic-auth` | Basic auth |
| `kubernetes.io/ssh-auth` | SSH keys |

### 2.2 Secret como Volume vs Env Var
```yaml
# ✅ PREFERIDO — Volume mount (mais seguro)
volumeMounts:
  - name: db-creds
    mountPath: /etc/secrets
    readOnly: true
volumes:
  - name: db-creds
    secret:
      secretName: db-credentials
      defaultMode: 0400  # read-only pelo owner

# ⚠️ MENOS SEGURO — Env var (pode aparecer em logs, /proc)
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
```

### 2.3 External Secrets Operator
```yaml
# ClusterSecretStore — conexão com o vault
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets

---
# ExternalSecret — referência ao secret externo
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
    template:
      type: Opaque
      data:
        DB_HOST: "{{ .host }}"
        DB_USER: "{{ .username }}"
        DB_PASSWORD: "{{ .password }}"
  data:
    - secretKey: host
      remoteRef:
        key: prod/db
        property: host
    - secretKey: username
      remoteRef:
        key: prod/db
        property: username
    - secretKey: password
      remoteRef:
        key: prod/db
        property: password
```

### 2.4 Sealed Secrets
```bash
# Encriptar secret para armazenar no Git
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
```

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    password: AgBy8h...encrypted...data
  template:
    metadata:
      name: db-credentials
      namespace: production
    type: Opaque
```

## 3. Helm

### 3.1 Chart Best Practices
```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.2.0        # chart version
appVersion: "3.5.0"    # app version
description: My Application
type: application
dependencies:
  - name: postgresql
    version: "13.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### 3.2 Values Structure
```yaml
# values.yaml — defaults sensatos
replicaCount: 2

image:
  repository: myapp
  tag: ""  # default to chart appVersion
  pullPolicy: IfNotPresent

serviceAccount:
  create: true
  automount: false
  annotations: {}

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts: []
  tls: []

networkPolicy:
  enabled: true

podDisruptionBudget:
  enabled: true
  minAvailable: 1
```

### 3.3 Template Best Practices
```yaml
# templates/_helpers.tpl
{{- define "myapp.labels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ include "myapp.chart" . }}
{{- end }}
```

### 3.4 Boas Práticas de Helm
- **Sempre inclua** `NOTES.txt` com instruções pós-install.
- **Teste** charts com `helm template`, `helm lint`, `helm test`.
- Use **semantic versioning** para charts.
- **Defaults seguros** — security context, resource limits no values default.
- Armazene charts em **OCI registry** (Harbor, ECR, ACR).
- Use **`helm diff`** plugin antes de upgrades.
- **Nunca use `helm install` sem `--dry-run` primeiro** em produção.

```bash
# Workflow de Helm
helm lint ./charts/myapp
helm template myapp ./charts/myapp -f values-prod.yaml | kubectl apply --dry-run=server -f -
helm diff upgrade myapp ./charts/myapp -f values-prod.yaml
helm upgrade --install myapp ./charts/myapp -f values-prod.yaml --atomic --timeout 10m
```

## 4. Kustomize vs Helm

| Aspecto | Kustomize | Helm |
|---------|-----------|------|
| **Abordagem** | Patches sobre base | Templates com valores |
| **Complexidade** | Baixa | Média-Alta |
| **Reusabilidade** | Overlays | Charts empacotados |
| **Lógica** | Sem lógica | Conditionals, loops |
| **Dependências** | Manual | Chart dependencies |
| **Ecossistema** | Built-in kubectl | Enorme (ArtifactHub) |
| **Recomendação** | Apps internas | Pacotes distribuíveis |

**Regra geral:**
- **Kustomize** para aplicações internas com variações simples entre ambientes.
- **Helm** para pacotes distribuíveis ou quando precisa de lógica complexa.
- **Ambos** podem ser usados juntos (Helm para base, Kustomize para overlay).

## 5. Environment Management

### 5.1 Estratégia de Promoção
```
dev → staging → production

dev:
  - Auto-deploy on merge
  - Reduced resources
  - Debug logging enabled
  - Feature flags: all on

staging:
  - Manual promotion
  - Production-like resources
  - Info logging
  - Feature flags: production mirror

production:
  - Approval required
  - Full resources
  - Warn logging
  - Feature flags: controlled rollout
```

### 5.2 Feature Flags
```yaml
# ConfigMap com feature flags
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  flags.json: |
    {
      "new-checkout": {
        "enabled": true,
        "percentage": 25
      },
      "dark-mode": {
        "enabled": false
      }
    }
```

**Ferramentas de Feature Flags:**
| Ferramenta | Tipo | Descrição |
|-----------|------|-----------|
| **LaunchDarkly** | SaaS | Enterprise feature management |
| **Unleash** | Open Source | Self-hosted feature toggles |
| **Flagsmith** | Open Source | Feature flags + remote config |
| **OpenFeature** | Standard | Vendor-neutral SDK |
