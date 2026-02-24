# Configuração Externa (Externalized Configuration)

> **Categoria:** Infraestrutura e Deploy
> **Complementa:** Feature Flags, Health Checks, Service Discovery
> **Keywords:** configuração externa, externalized configuration, config server, environment variables, secrets management, twelve-factor

---

## Problema

Configurações hardcoded no código ou em arquivos empacotados junto com o artefato criam problemas:

- **Rebuild para cada ambiente** — dev, staging, prod precisam de configurações diferentes
- **Secrets no repositório** — credenciais, tokens, API keys expostos no código-fonte
- **Sem mudança dinâmica** — alterar configuração exige novo deploy
- **Inconsistência** — cada serviço gerencia configuração de forma diferente

---

## Solução

Separar **configuração do código**. Configurações são armazenadas externamente e injetadas no serviço em **runtime**, sem necessidade de rebuild ou redeploy.

```
┌──────────────┐         ┌─────────────────────────┐
│   Serviço    │◀────────│  Configuração Externa   │
│  (código)    │         │                         │
│              │         │  - Variáveis de ambiente │
│ Mesmo        │         │  - Config Server         │
│ artefato em  │         │  - Secrets Manager       │
│ todos os     │         │  - Config Maps (K8s)     │
│ ambientes    │         │                         │
└──────────────┘         └─────────────────────────┘
```

**Princípio:** O mesmo artefato (binário, container image) roda em dev, staging e prod — apenas a configuração muda.

---

## Tipos de Configuração

| Tipo | Exemplos | Sensibilidade | Frequência de Mudança |
|------|---------|---------------|----------------------|
| **Infraestrutura** | URLs de banco, hosts de cache, brokers | Média | Rara |
| **Comportamento** | Timeouts, pool sizes, retry attempts | Baixa | Ocasional |
| **Feature Flags** | Features habilitadas/desabilitadas | Baixa | Frequente |
| **Secrets** | Passwords, tokens, API keys, certificates | **Alta** | Rara |
| **Negócio** | Limites de crédito, taxas, percentuais | Baixa | Variável |

---

## Mecanismos de Configuração Externa

### 1. Variáveis de Ambiente

O mecanismo mais simples e universal:

```
Execução do serviço:
  DATABASE_URL=postgres://prod-host:5432/mydb
  CACHE_HOST=redis-prod:6379
  LOG_LEVEL=INFO
  MAX_CONNECTIONS=50

No código:
  dbUrl = env("DATABASE_URL")
  cacheHost = env("CACHE_HOST")
```

| Vantagem | Desvantagem |
|----------|------------|
| Universal (toda plataforma suporta) | Sem versionamento |
| Simples | Difícil de gerenciar muitas variáveis |
| 12-Factor App compliant | Sem mudança dinâmica (exige restart) |

### 2. Arquivos de Configuração (por ambiente)

Arquivos diferentes por ambiente, montados no container ou injetados:

```
configs/
  application-dev.yaml
  application-staging.yaml
  application-prod.yaml

O ambiente determina qual arquivo carregar:
  ENVIRONMENT=prod → carrega application-prod.yaml
```

### 3. Config Server (Centralizado)

Um serviço dedicado que serve configurações para todos os microsserviços:

```
┌───────────┐     ┌──────────────┐     ┌─────────────┐
│ Service A │────▶│ Config Server│◀────│ Git / DB /  │
│ Service B │────▶│              │     │ Key-Value   │
│ Service C │────▶│              │     └─────────────┘
└───────────┘     └──────────────┘

Config Server:
  GET /config/order-service/prod
  → { "database.url": "postgres://...", "cache.ttl": 30, ... }
```

| Vantagem | Desvantagem |
|----------|------------|
| Centralizado — uma fonte de verdade | Single point of failure |
| Versionado (se usa Git como backend) | Latência adicional no startup |
| Mudança sem redeploy | Mais infra para gerenciar |

### 4. ConfigMaps e Secrets (Kubernetes)

```
Kubernetes ConfigMap:
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: order-service-config
  data:
    DATABASE_HOST: "postgres-prod"
    CACHE_TTL: "30"
    LOG_LEVEL: "INFO"

Kubernetes Secret:
  apiVersion: v1
  kind: Secret
  metadata:
    name: order-service-secrets
  type: Opaque
  data:
    DATABASE_PASSWORD: <base64 encoded>
    API_KEY: <base64 encoded>

Pod monta ConfigMap como env vars ou volume:
  envFrom:
    - configMapRef:
        name: order-service-config
    - secretRef:
        name: order-service-secrets
```

### 5. Secrets Manager (para dados sensíveis)

Para **secrets** (passwords, tokens, certificates), use um gerenciador dedicado:

| Ferramenta | Descrição |
|-----------|-----------|
| **HashiCorp Vault** | Secrets management, encryption, dynamic secrets |
| **AWS Secrets Manager** | Secrets com rotação automática |
| **Azure Key Vault** | Secrets, keys e certificates |
| **GCP Secret Manager** | Secrets com versionamento |

```
Fluxo:
  Serviço inicia → autentica no Vault → solicita secret
  Vault verifica permissões → retorna secret
  Serviço usa o secret → secret rotaciona automaticamente

Benefícios:
  - Secrets nunca ficam no código ou repo
  - Rotação automática (sem redeploy)
  - Audit log de quem acessou
  - Criptografia em repouso e trânsito
```

---

## Configuração Dinâmica (sem redeploy)

| Mecanismo | Como funciona |
|-----------|--------------|
| **Polling** | Serviço consulta o config server periodicamente (a cada 30s) |
| **Push (webhook)** | Config server notifica serviços quando configuração muda |
| **Watch** | Serviço observa mudanças em tempo real (Consul watch, K8s watch) |
| **Hot-reload** | Serviço recarrega configuração sem restart |

```
Polling:
  every 30s:
    newConfig = configServer.get("order-service")
    if newConfig != currentConfig:
        applyConfig(newConfig)
        log.info("Configuração atualizada dinamicamente")

Push (webhook):
  configServer.onChange("order-service", () -> {
      reloadConfig()
  })
```

---

## Hierarquia de Configuração

Configurações devem ter uma **hierarquia de precedência** (últimas sobrescrevem primeiras):

```
1. Defaults no código          (menor prioridade)
2. Arquivo de configuração     
3. ConfigMap / Config Server   
4. Variáveis de ambiente       
5. Command-line arguments      (maior prioridade)

Exemplo:
  Código:     timeout = 5000ms (default)
  ConfigMap:  timeout = 3000ms (sobrescreve)
  Env var:    TIMEOUT = 1000ms (sobrescreve tudo)
  
  Resultado:  timeout = 1000ms
```

---

## Exemplo Conceitual (Pseudocódigo)

```
// Carregamento de configuração com hierarquia
class ConfigLoader:
    function loadConfig(serviceName, environment):
        config = {}
        
        // 1. Defaults
        config.merge(loadDefaults())
        
        // 2. Arquivo de configuração
        config.merge(loadFile("config-{environment}.yaml"))
        
        // 3. Config Server (se disponível)
        try:
            remoteConfig = configServer.get(serviceName, environment)
            config.merge(remoteConfig)
        catch:
            log.warn("Config server indisponível, usando configuração local")
        
        // 4. Variáveis de ambiente
        config.merge(loadEnvVars())
        
        // 5. Secrets Manager
        secrets = secretsManager.get(serviceName)
        config.merge(secrets)
        
        return config

// Uso
config = ConfigLoader.loadConfig("order-service", "prod")
dbUrl = config.get("database.url")
timeout = config.get("http.timeout", default=5000)
```

---

## Antipadrões

| Antipadrão | Problema | Solução |
|-----------|----------|---------|
| Secrets no código/repo | Exposição de credenciais | Use Secrets Manager ou variáveis cifradas |
| Config diferente por build | Artefatos diferentes por ambiente | Mesmo artefato + config externa |
| Sem defaults | Erro se config não estiver presente | Sempre tenha defaults sensatos |
| Config server sem fallback | Config server cai = serviço não inicia | Cache local da última config válida |
| Configuração imutável (sem reload) | Precisa de redeploy para mudar timeout | Implemente hot-reload para configs não-críticas |
| Muitas variáveis de ambiente | Difícil de gerenciar | Agrupe em arquivos/configmaps, use naming convention |

---

## Relação com Outros Padrões

| Padrão | Relação |
|--------|---------|
| **Feature Flags** | Feature flags são um tipo de configuração dinâmica |
| **Health Checks** | Config incorreta causa unhealthy — health check detecta |
| **Service Discovery** | URLs de serviços são configuração externa |
| **Circuit Breaker** | Thresholds do CB podem ser configuração dinâmica |
| **Sidecar** | Sidecar pode gerenciar config/secrets transparentemente |

---

## Boas Práticas

1. **Nunca** coloque secrets no código-fonte ou no repositório.
2. Use **variáveis de ambiente** para configuração simples (12-Factor App).
3. Use **Secrets Manager** (Vault, AWS SM) para dados sensíveis.
4. Defina **defaults sensatos** — o serviço deve funcionar com configuração mínima.
5. Implemente **hierarquia de precedência** (defaults < arquivo < server < env).
6. Considere **hot-reload** para configurações que mudam frequentemente (timeouts, feature flags).
7. Use **naming convention** consistente (`SERVICE_DATABASE_HOST`, `SERVICE_CACHE_TTL`).
8. O **mesmo artefato** deve rodar em todos os ambientes — apenas a configuração muda.

---

## Referências

- 12-Factor App — [III. Config](https://12factor.net/config)
- HashiCorp Vault — [Documentation](https://www.vaultproject.io/docs)
- Kubernetes — [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- Chris Richardson — *Microservices Patterns* — Externalized Configuration
