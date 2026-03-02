# Level 7 — Produção e Deploy

> **Objetivo:** Preparar a aplicação para deploy em produção com Docker multi-stage builds, docker-compose completo, graceful shutdown, CI/CD pipeline e manifests Kubernetes.

---

## Objetivo de Aprendizado

- Construir imagens Docker otimizadas (multi-stage build)
- Aplicar 12-factor app principles
- Implementar graceful shutdown
- Configurar docker-compose de produção
- Criar CI/CD pipeline básico (GitHub Actions)
- Gerar manifests Kubernetes (Deployment, Service, ConfigMap, Secret)
- Implementar feature flags para deploy seguro
- Entender blue-green / canary deploy concepts

---

## Escopo Funcional

### Nenhum endpoint novo — foco em infraestrutura de deploy

O aplicativo deve funcionar exatamente como nos levels anteriores, mas agora:

1. Rodando em container Docker otimizado
2. Com todas as dependências em `docker-compose.yml`
3. Com variáveis de ambiente externalizadas (zero segredos no código)
4. Com health checks funcionais para orquestrador
5. Com shutdown gracioso (draining connections)
6. Com pipeline CI/CD automatizado
7. Com feature flags para rollout gradual

---

## Escopo Técnico

### Docker Build por Stack

| Stack | Estratégia | Base Image | Tamanho Final |
|---|---|---|---|
| **Go (Gin)** | Multi-stage: build → scratch/distroless | `gcr.io/distroless/static-debian12` | ~15-20 MB |
| **Spring Boot** | CDS (Class Data Sharing) multi-stage | `eclipse-temurin:25-jre-alpine` | ~150-200 MB |
| **Quarkus** | Native build (GraalVM) | `quay.io/quarkus/quarkus-micro-image` | ~50-80 MB |
| **Micronaut** | Native build (GraalVM) | `gcr.io/distroless/cc-debian12` | ~60-90 MB |
| **Jakarta EE** | Thin WAR + runtime | `quay.io/wildfly/wildfly:33.0.0.Final-jdk21` | ~300-500 MB |

### 12-Factor App Checklist

| Factor | Implementação |
|---|---|
| 1. Codebase | 1 repo por stack |
| 2. Dependencies | `go.mod` / `pom.xml` — versionadas explicitamente |
| 3. Config | Env vars: `DB_URL`, `KAFKA_BROKERS`, `JWT_SECRET` |
| 4. Backing services | PostgreSQL, Kafka, Jaeger via URL |
| 5. Build, release, run | Docker build → tag → run |
| 6. Processes | Stateless — nenhum estado em filesystem |
| 7. Port binding | `PORT` env var |
| 8. Concurrency | Goroutines / virtual threads / reactive |
| 9. Disposability | Graceful shutdown |
| 10. Dev/prod parity | Docker-compose ≈ produção |
| 11. Logs | Stdout JSON |
| 12. Admin processes | Migrações via Flyway CLI / golang-migrate |

### Feature Flags

| Stack | Ferramenta |
|---|---|
| **Go (Gin)** | go-feature-flag (thomaspoignant/go-feature-flag) |
| **Spring Boot** | Togglz |
| **Quarkus** | Togglz / LaunchDarkly |
| **Micronaut** | Togglz / LaunchDarkly |
| **Jakarta EE** | Togglz / MicroProfile Config |

---

## Critérios de Aceite

- [ ] `docker build` produz imagem funcional
- [ ] Imagem final ≤ tamanho esperado (Go ≤ 25MB, Spring ≤ 250MB, Quarkus native ≤ 100MB)
- [ ] `docker-compose up` sobe toda a stack (app + postgres + kafka + observability)
- [ ] Health check funcional no container (Docker HEALTHCHECK)
- [ ] Graceful shutdown: requests em andamento completam antes de parar
- [ ] Zero segredos hardcoded (todos via env vars / secrets)
- [ ] CI pipeline: build → test → lint → Docker build → push
- [ ] Feature flag funcional: endpoint habilitado/desabilitado via flag
- [ ] Kubernetes manifests aplicáveis: `kubectl apply -f k8s/`
- [ ] Startup time medido e documentado
- [ ] Logs em stdout/stderr (JSON em produção)

---

## Definição de Pronto (DoD)

- [ ] Dockerfile multi-stage funcional e otimizado
- [ ] `docker-compose.yml` completo com todos os serviços
- [ ] `.env.example` com todas as variáveis documentadas
- [ ] `Makefile` ou scripts com comandos: build, run, test, docker-build, docker-run
- [ ] `.github/workflows/ci.yml` com pipeline CI
- [ ] `k8s/` directory com manifests básicos
- [ ] Feature flag para pelo menos 1 funcionalidade
- [ ] Teste de graceful shutdown (kill container durante request)
- [ ] README do projeto com instruções de setup em 5 minutos
- [ ] Commit: `feat(level-7): add Docker, CI/CD, and production readiness`

---

## Checklist

### Go (Gin)

- [ ] `Dockerfile` — multi-stage: `golang:1.24-alpine` → `gcr.io/distroless/static-debian12`
- [ ] `cmd/api/main.go` — graceful shutdown com `signal.NotifyContext`
- [ ] `.env.example` — todas variáveis
- [ ] `Makefile` — targets: `build`, `run`, `test`, `lint`, `docker-build`, `docker-push`
- [ ] `.golangci.yml` — configuração de linters
- [ ] `internal/platform/featureflag/flags.go` — go-feature-flag setup
- [ ] `internal/middleware/featureflag.go` — middleware que verifica flag antes de endpoint
- [ ] `k8s/deployment.yaml`, `k8s/service.yaml`, `k8s/configmap.yaml`, `k8s/secret.yaml`

### Java — Spring Boot

- [ ] `Dockerfile` — multi-stage CDS build
- [ ] `src/main/java/.../Application.java` — `@SpringBootApplication` (graceful shutdown é automático)
- [ ] `application.yml` — `server.shutdown=graceful`, `spring.lifecycle.timeout-per-shutdown-phase=30s`
- [ ] `application-prod.yml` — configurações de produção
- [ ] `.env.example`
- [ ] `Makefile` / `justfile`
- [ ] `core/config/FeatureFlagConfig.java` — Togglz setup
- [ ] `k8s/deployment.yaml`, `k8s/service.yaml`

### Quarkus

- [ ] `Dockerfile.native` — GraalVM native build
- [ ] `Dockerfile.jvm` — JVM build (fallback)
- [ ] Graceful shutdown configurado: `quarkus.shutdown.timeout=30s`
- [ ] `.env.example`
- [ ] `k8s/` — Quarkus Kubernetes extension (gerado automaticamente com `quarkus-kubernetes`)

### Micronaut

- [ ] `Dockerfile` — GraalVM native build com `mn:dockerfile`
- [ ] Graceful shutdown: built-in com Netty
- [ ] `.env.example`
- [ ] `k8s/` — manifests gerados com `micronaut-kubernetes`

### Jakarta EE

- [ ] `Dockerfile` — WAR build + WildFly runtime
- [ ] Graceful shutdown via WildFly subsystem
- [ ] `.env.example`
- [ ] `k8s/deployment.yaml` — com WildFly

---

## Tarefas Sugeridas por Stack

### Go — Dockerfile Multi-Stage

```dockerfile
# ── Build stage ──
FROM golang:1.24-alpine AS builder

RUN apk add --no-cache git ca-certificates tzdata

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w -X main.version=$(git describe --tags --always)" \
    -o /bin/api ./cmd/api

# ── Runtime stage ──
FROM gcr.io/distroless/static-debian12

COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /bin/api /bin/api
COPY --from=builder /app/migrations /migrations

EXPOSE 8080

ENTRYPOINT ["/bin/api"]
```

### Go — Graceful Shutdown

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    // ... setup (stores, services, handlers, router)

    srv := &http.Server{
        Addr:         ":" + cfg.Server.Port,
        Handler:      router,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Start server
    go func() {
        logger.Info("starting server", zap.String("port", cfg.Server.Port))
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            logger.Fatal("server failed", zap.Error(err))
        }
    }()

    // Wait for signal
    <-ctx.Done()
    logger.Info("shutting down gracefully...")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(shutdownCtx); err != nil {
        logger.Error("forced shutdown", zap.Error(err))
    }

    // Close Kafka producer, DB connections, etc.
    producer.Close()
    db.Close()

    logger.Info("server stopped")
}
```

### Spring Boot — Dockerfile CDS

```dockerfile
# ── Build stage ──
FROM eclipse-temurin:25-jdk-alpine AS builder

WORKDIR /app
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline -B

COPY src/ src/
RUN ./mvnw package -DskipTests -B

# ── CDS Training Run ──
FROM eclipse-temurin:25-jre-alpine AS cds

WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
RUN java -Dspring.context.exit=onRefresh -XX:ArchiveClassesAtExit=app.jsa -jar app.jar || true

# ── Runtime ──
FROM eclipse-temurin:25-jre-alpine

RUN addgroup -S app && adduser -S app -G app
USER app

WORKDIR /app
COPY --from=cds /app/app.jar app.jar
COPY --from=cds /app/app.jsa app.jsa

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget --spider --quiet http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-XX:SharedArchiveFile=app.jsa", "-jar", "app.jar"]
```

### Quarkus — Native Build

```dockerfile
# ── Build native ──
FROM quay.io/quarkus/ubi-quarkus-mandrel-builder-image:jdk-21 AS builder

WORKDIR /app
COPY --chown=quarkus:quarkus . .
RUN ./mvnw package -Dnative -DskipTests -B

# ── Runtime ──
FROM quay.io/quarkus/quarkus-micro-image:2.0

WORKDIR /app
COPY --from=builder /app/target/*-runner /app/application

EXPOSE 8080

HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/q/health/live || exit 1

ENTRYPOINT ["./application", "-Dquarkus.http.host=0.0.0.0"]
```

---

## CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      # Go example
      - uses: actions/setup-go@v5
        with:
          go-version: '1.24'

      - name: Lint
        uses: golangci/golangci-lint-action@v4

      - name: Test
        run: go test ./... -v -cover -coverprofile=coverage.out
        env:
          DB_URL: postgres://test:test@localhost:5432/testdb?sslmode=disable

      - name: Coverage check
        run: |
          COVERAGE=$(go tool cover -func=coverage.out | tail -1 | awk '{print $NF}' | tr -d '%')
          echo "Coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 70" | bc -l) )); then
            echo "Coverage below 70%"
            exit 1
          fi

      - name: Build Docker image
        run: docker build -t digital-wallet:${{ github.sha }} .

      # Java example (alternative job)
      # - uses: actions/setup-java@v4
      #   with:
      #     java-version: '25'
      #     distribution: 'temurin'
      # - name: Build & Test
      #   run: ./mvnw verify
```

---

## Kubernetes Manifests

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: digital-wallet
  labels:
    app: digital-wallet
spec:
  replicas: 2
  selector:
    matchLabels:
      app: digital-wallet
  template:
    metadata:
      labels:
        app: digital-wallet
    spec:
      containers:
        - name: api
          image: digital-wallet:latest
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: digital-wallet-config
            - secretRef:
                name: digital-wallet-secrets
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: digital-wallet
spec:
  type: ClusterIP
  selector:
    app: digital-wallet
  ports:
    - port: 80
      targetPort: 8080
---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: digital-wallet-config
data:
  SERVER_PORT: "8080"
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  DB_NAME: "digital_wallet"
  KAFKA_BROKERS: "kafka-service:9092"
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: digital-wallet-secrets
type: Opaque
stringData:
  DB_USER: wallet_user
  DB_PASSWORD: wallet_pass
  JWT_SECRET: change-me-in-production
```

---

## docker-compose.yml (completo)

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_USER=wallet_user
      - DB_PASSWORD=wallet_pass
      - DB_NAME=digital_wallet
      - KAFKA_BROKERS=kafka:9092
      - JWT_SECRET=${JWT_SECRET:-dev-secret-change-me}
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
      - LOG_LEVEL=info
      - LOG_FORMAT=json
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_started
    healthcheck:
      test: ["CMD", "wget", "--spider", "--quiet", "http://localhost:8080/health/live"]
      interval: 10s
      timeout: 3s
      retries: 3

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: digital_wallet
      POSTGRES_USER: wallet_user
      POSTGRES_PASSWORD: wallet_pass
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U wallet_user -d digital_wallet"]
      interval: 5s
      timeout: 5s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:29093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:29093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./infra/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "4317:4317"
    environment:
      COLLECTOR_OTLP_ENABLED: "true"

volumes:
  pgdata:
```

---

## Extensões Opcionais

- [ ] Implementar Helm chart para Kubernetes
- [ ] Adicionar Terraform para provisionar infraestrutura cloud
- [ ] Implementar blue-green deploy com Nginx ou Istio
- [ ] Adicionar ArgoCD para GitOps
- [ ] Configurar auto-scaling (HPA) baseado em CPU/custom metrics
- [ ] Implementar multi-arch build (amd64 + arm64)
- [ ] Adicionar Trivy scan para vulnerabilidades na imagem Docker

---

## Erros Comuns

| Erro | Stack | Como evitar |
|---|---|---|
| Segredos em env vars no docker-compose committed | Todos | Usar `.env` no `.gitignore` + `.env.example` committado |
| Imagem Docker gigante (1GB+) | Java | Multi-stage build com JRE (não JDK) |
| Sem graceful shutdown → requests cortados | Todos | Implementar signal handling (SIGTERM) |
| HEALTHCHECK sem timeout → container fica unhealthy | Todos | Timeout ≤ interval |
| CI sem cache → build lento | Todos | Cache de dependências (go mod, maven repo) |
| `latest` tag em produção | Todos | Usar tags semânticas ($SHA, $VERSION) |
| Rodar como root no container | Todos | `USER nonroot` / `USER app` no Dockerfile |
| Logs em arquivo dentro do container | Todos | Logs em stdout/stderr — orquestrador coleta |

---

## Como Isso Aparece em Entrevistas

- "Explique os princípios de 12-factor app. Quais você aplica?"
- "Como funciona um multi-stage Docker build?"
- "O que é CDS no contexto de Spring Boot?"
- "Qual a diferença entre liveness e readiness probe no Kubernetes?"
- "Como você implementa graceful shutdown?"
- "O que são feature flags e como ajudam em deploy?"
- "Qual a diferença entre blue-green e canary deploy?"
- "Como você gerencia segredos em Kubernetes?"
- "O que é o GraalVM native image? Quais os trade-offs?"
- "Como funciona o Quarkus native comparado ao Spring Boot CDS em termos de startup?"

---

## Comandos de Execução

```bash
# Build Docker
docker build -t digital-wallet:latest .

# Subir tudo
docker-compose up -d

# Ver logs
docker-compose logs -f api

# Testar health
curl http://localhost:8080/health/live
curl http://localhost:8080/health/ready

# Testar graceful shutdown
docker-compose stop api --timeout 30

# Medir startup time
time docker run --rm digital-wallet:latest --help 2>&1 | head -1

# Medir tamanho da imagem
docker images digital-wallet:latest --format "{{.Size}}"

# Kubernetes (minikube/kind)
kubectl apply -f k8s/
kubectl get pods -w
kubectl logs -f deployment/digital-wallet
kubectl port-forward svc/digital-wallet 8080:80
```
