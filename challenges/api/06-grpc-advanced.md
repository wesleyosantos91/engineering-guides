# Level 6 — gRPC Advanced: Produção & Operação

> **Objetivo:** Levar serviços gRPC para produção com segurança, observabilidade,
> performance, versionamento e integração com REST via gRPC-Gateway.

**Referência:** [.docs/API/03-grpc-best-practices.md](../../../.docs/API/03-grpc-best-practices.md) — Seções 9–18

---

## Desafio 6.1 — Segurança: TLS, mTLS & Token Auth

**Contexto:** Em produção, toda comunicação gRPC deve ser criptografada.
mTLS garante autenticação mútua entre microserviços.

### Requisitos

Implementar 3 camadas de segurança:

**1. TLS (Transport Layer Security):**

| Componente | Arquivo |
|-----------|---------|
| CA Certificate | `certs/ca.crt` |
| Server Certificate | `certs/server.crt` |
| Server Key | `certs/server.key` |
| Client Certificate (mTLS) | `certs/client.crt` |
| Client Key (mTLS) | `certs/client.key` |

**2. mTLS (Mutual TLS):**
Server e client se autenticam mutuamente — ambos apresentam certificados.

**3. Token-based Auth (JWT via metadata):**
Para autenticação de usuários finais (além da autenticação de serviço via mTLS).

### Java 25

```java
// Server com TLS
var serverCert = new File("certs/server.crt");
var serverKey = new File("certs/server.key");
var caCert = new File("certs/ca.crt");

Server server = NettyServerBuilder.forPort(9090)
    .sslContext(GrpcSslContexts.forServer(serverCert, serverKey)
        .trustManager(caCert)                   // mTLS: validar client cert
        .clientAuth(ClientAuth.REQUIRE)          // mTLS: exigir client cert
        .build())
    .addService(new ProductServiceImpl())
    .build()
    .start();

// Client com mTLS
var channel = NettyChannelBuilder.forAddress("localhost", 9090)
    .sslContext(GrpcSslContexts.forClient()
        .trustManager(new File("certs/ca.crt"))
        .keyManager(
            new File("certs/client.crt"),     // client cert
            new File("certs/client.key"))     // client key
        .build())
    .build();

// Auth interceptor: JWT via metadata
public class AuthInterceptor implements ServerInterceptor {
    private static final Metadata.Key<String> AUTH_KEY =
        Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);

    @Override
    public <R, S> ServerCall.Listener<R> interceptCall(
            ServerCall<R, S> call, Metadata headers,
            ServerCallHandler<R, S> next) {

        var token = headers.get(AUTH_KEY);
        if (token == null || !token.startsWith("Bearer ")) {
            call.close(Status.UNAUTHENTICATED.withDescription("missing token"), headers);
            return new ServerCall.Listener<>() {};
        }

        try {
            var claims = validateJwt(token.substring(7));
            // Propagar claims via Context
            var ctx = Context.current()
                .withValue(USER_ID_KEY, claims.userId())
                .withValue(ROLES_KEY, claims.roles());
            return Contexts.interceptCall(ctx, call, headers, next);
        } catch (Exception e) {
            call.close(Status.UNAUTHENTICATED.withDescription("invalid token"), headers);
            return new ServerCall.Listener<>() {};
        }
    }
}
```

### Go 1.26

```go
// Server com mTLS
cert, _ := tls.LoadX509KeyPair("certs/server.crt", "certs/server.key")
caCert, _ := os.ReadFile("certs/ca.crt")
pool := x509.NewCertPool()
pool.AppendCertsFromPEM(caCert)

creds := credentials.NewTLS(&tls.Config{
    Certificates: []tls.Certificate{cert},
    ClientAuth:   tls.RequireAndVerifyClientCert,
    ClientCAs:    pool,
})

server := grpc.NewServer(grpc.Creds(creds))

// Auth interceptor: extrair JWT do metadata
func authInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler) (any, error) {

    // Skip auth para health check
    if info.FullMethod == "/grpc.health.v1.Health/Check" {
        return handler(ctx, req)
    }

    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Errorf(codes.Unauthenticated, "missing metadata")
    }

    tokens := md.Get("authorization")
    if len(tokens) == 0 || !strings.HasPrefix(tokens[0], "Bearer ") {
        return nil, status.Errorf(codes.Unauthenticated, "missing token")
    }

    claims, err := validateJWT(strings.TrimPrefix(tokens[0], "Bearer "))
    if err != nil {
        return nil, status.Errorf(codes.Unauthenticated, "invalid token: %v", err)
    }

    // Propagar claims via context
    ctx = context.WithValue(ctx, userIDKey, claims.UserID)
    ctx = context.WithValue(ctx, rolesKey, claims.Roles)
    return handler(ctx, req)
}

// Client: enviar JWT via metadata
type jwtCredentials struct {
    token string
}

func (j *jwtCredentials) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
    return map[string]string{"authorization": "Bearer " + j.token}, nil
}

func (j *jwtCredentials) RequireTransportSecurity() bool { return true }

conn, _ := grpc.NewClient("localhost:9090",
    grpc.WithTransportCredentials(tlsCreds),
    grpc.WithPerRPCCredentials(&jwtCredentials{token: myJWT}),
)
```

### Critérios de Aceite

- [ ] **TLS:** Server com certificado, client verifica CA
- [ ] **mTLS:** Server exige e valida certificado do client
- [ ] Conexão sem certificado → rejeitada
- [ ] **JWT Auth:** Token via metadata `authorization: Bearer <jwt>`
- [ ] RPCs sem token → `UNAUTHENTICATED`
- [ ] Claims propagadas via Context (userId, roles)
- [ ] Health check bypass auth (precisa funcionar sem token)
- [ ] Gerar certificados auto-assinados para testes (script `gen-certs.sh`)

---

## Desafio 6.2 — Health Checking & Load Balancing

**Contexto:** O protocolo de health check gRPC (`grpc.health.v1.Health`) é o padrão
para load balancers e orquestradores como Kubernetes.

### Requisitos

**Health Check Protocol (`grpc.health.v1`):**

```protobuf
// Protocolo padrão — NÃO precisa definir, usar a spec oficial
service Health {
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}

// Respostas possíveis
enum ServingStatus {
  UNKNOWN = 0;         // Não sabe se está healthy
  SERVING = 1;         // Pronto para receber requests
  NOT_SERVING = 2;     // Não pode receber requests
  SERVICE_UNKNOWN = 3; // Serviço não registrado
}
```

**Health check por serviço:**

| Service Name | O que verifica |
|-------------|----------------|
| `""` (vazio) | Health geral do server |
| `marketplace.product.v1.ProductService` | Serviço de produtos |
| `marketplace.order.v1.OrderService` | Serviço de pedidos |
| `marketplace.inventory.v1.InventoryService` | Serviço de inventário |

**Implementar dependency checks:**

```
ProductService health:
  ├── Store (in-memory/DB) acessível?
  ├── InventoryService (downstream) respondendo?
  └── Último request < 30s atrás? (liveness)
```

### Go 1.26

```go
type HealthServer struct {
    grpc_health_v1.UnimplementedHealthServer
    mu       sync.RWMutex
    statuses map[string]grpc_health_v1.HealthCheckResponse_ServingStatus
    watchers map[string][]chan grpc_health_v1.HealthCheckResponse_ServingStatus
}

func (h *HealthServer) Check(ctx context.Context,
    req *grpc_health_v1.HealthCheckRequest) (*grpc_health_v1.HealthCheckResponse, error) {

    h.mu.RLock()
    defer h.mu.RUnlock()

    status, ok := h.statuses[req.GetService()]
    if !ok {
        return nil, grpcstatus.Errorf(codes.NotFound, "unknown service: %s", req.GetService())
    }

    return &grpc_health_v1.HealthCheckResponse{Status: status}, nil
}

func (h *HealthServer) Watch(req *grpc_health_v1.HealthCheckRequest,
    stream grpc_health_v1.Health_WatchServer) error {

    updates := h.subscribe(req.GetService())
    defer h.unsubscribe(req.GetService(), updates)

    for {
        select {
        case status := <-updates:
            err := stream.Send(&grpc_health_v1.HealthCheckResponse{Status: status})
            if err != nil {
                return err
            }
        case <-stream.Context().Done():
            return nil
        }
    }
}

// SetStatus atualiza e notifica watchers
func (h *HealthServer) SetStatus(service string,
    status grpc_health_v1.HealthCheckResponse_ServingStatus) {

    h.mu.Lock()
    h.statuses[service] = status
    watchers := h.watchers[service]
    h.mu.Unlock()

    for _, ch := range watchers {
        select {
        case ch <- status:
        default: // non-blocking send
        }
    }
}

// Background health checker
func (h *HealthServer) StartPeriodicCheck(ctx context.Context,
    service string, check func() bool, interval time.Duration) {

    go func() {
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        for {
            select {
            case <-ticker.C:
                if check() {
                    h.SetStatus(service, grpc_health_v1.HealthCheckResponse_SERVING)
                } else {
                    h.SetStatus(service, grpc_health_v1.HealthCheckResponse_NOT_SERVING)
                }
            case <-ctx.Done():
                return
            }
        }
    }()
}
```

**Client-side load balancing:**

```go
// Round-robin entre instâncias
conn, _ := grpc.NewClient(
    "dns:///product-service:9090",
    grpc.WithDefaultServiceConfig(`{"loadBalancingConfig": [{"round_robin":{}}]}`),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

### Critérios de Aceite

- [ ] `Health.Check` retorna status por serviço
- [ ] `Health.Watch` envia stream de status changes
- [ ] Health geral (`""`) = SERVING se todos os serviços estão SERVING
- [ ] Dependency checks periódicos (store, downstream services)
- [ ] Status muda para NOT_SERVING quando downstream fica indisponível
- [ ] `grpcurl` funciona: `grpcurl -plaintext localhost:9090 grpc.health.v1.Health/Check`
- [ ] Client-side round-robin configurado
- [ ] Testes: simular downstream failure → status changes

---

## Desafio 6.3 — Versionamento & Schema Evolution

**Contexto:** APIs gRPC evoluem. O versionamento via package (`v1`, `v2`) e as
regras de compatibilidade do Protobuf garantem evolução sem quebras.

### Requisitos

**Evolução backward-compatible (pode fazer):**

| Mudança | Safe? | Razão |
|---------|:-----:|-------|
| Adicionar field (novo número) | ✅ | Clients antigos ignoram campo novo |
| Adicionar novo RPC | ✅ | Clients antigos não chamam |
| Adicionar valor ao enum | ✅ | Clients antigos leem como 0 |
| Renomear field | ✅ | Wire format usa números, não nomes |
| Adicionar `oneof` | ✅ | Expansão natural |

**Breaking changes (NUNCA fazer):**

| Mudança | Safe? | Razão |
|---------|:-----:|-------|
| Remover field sem `reserved` | ❌ | Número pode ser reutilizado |
| Mudar tipo de field | ❌ | Desserialização quebra |
| Mudar número de field | ❌ | Dados antigos perdem mapeamento |
| Renumerar enum values | ❌ | Valores mudam de significado |
| Mudar package name | ❌ | Full method name muda |

**Versão major (breaking change necessária) → Novo package:**

```protobuf
// v1 — mantém para backward compatibility
package marketplace.product.v1;

service ProductService {
  rpc GetProduct(GetProductRequest) returns (Product);
}

// v2 — novos features, breaking changes isoladas
package marketplace.product.v2;

service ProductService {
  rpc GetProduct(GetProductRequest) returns (ProductResponse);  // wrapper response
  rpc SearchProducts(SearchRequest) returns (stream Product);   // novo RPC
}
```

**Schema validation com `buf`:**

```yaml
# buf.yaml
version: v2
breaking:
  use:
    - FILE        # Breaking change detection
lint:
  use:
    - DEFAULT     # Enforce naming conventions
    - COMMENTS    # Require comments
  except:
    - PACKAGE_VERSION_SUFFIX  # Permitir sem versão em testes
```

### Java 25 — Dual version server

```java
// Servir v1 e v2 simultaneamente
Server server = ServerBuilder.forPort(9090)
    .addService(new ProductServiceV1Impl())  // v1 para clients antigos
    .addService(new ProductServiceV2Impl())  // v2 para clients novos
    .build();

// V2 delega para V1 quando possível
public class ProductServiceV2Impl extends ProductServiceV2Grpc.ProductServiceImplBase {
    private final ProductServiceV1Impl v1;

    @Override
    public void getProduct(GetProductRequestV2 request,
                            StreamObserver<ProductResponseV2> responseObserver) {
        // Converter v2 request → v1 request
        var v1Request = GetProductRequestV1.newBuilder()
            .setId(request.getId())
            .build();

        // Chamar v1 implementation
        v1.getProduct(v1Request, new StreamObserver<>() {
            @Override
            public void onNext(ProductV1 product) {
                // Converter v1 response → v2 response (com campos extras)
                responseObserver.onNext(convertToV2(product));
            }
            @Override public void onError(Throwable t) { responseObserver.onError(t); }
            @Override public void onCompleted() { responseObserver.onCompleted(); }
        });
    }
}
```

### Go 1.26 — `buf` breaking detection

```bash
# Instalar buf
go install github.com/bufbuild/buf/cmd/buf@latest

# Lint: verificar convenções de nomenclatura
buf lint

# Breaking: detectar breaking changes vs branch main
buf breaking --against '.git#branch=main'

# Gerar código a partir dos .proto
buf generate
```

```yaml
# buf.gen.yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen/go
    opt: paths=source_relative
  - remote: buf.build/grpc/go
    out: gen/go
    opt: paths=source_relative
  - remote: buf.build/protocolbuffers/java
    out: gen/java
```

### Critérios de Aceite

- [ ] v1 e v2 coexistem no mesmo server (registrar ambos os services)
- [ ] Clients v1 continuam funcionando após deploy de v2
- [ ] `reserved` usado para todos os campos removidos
- [ ] `buf lint` — zero warnings com regras DEFAULT
- [ ] `buf breaking` — detecta mudanças incompatíveis
- [ ] Migração v1→v2 com adapter/converter entre versões
- [ ] Enum values nunca renumerados (novas entries no final)
- [ ] Testes: client v1 fala com server v1+v2, client v2 fala com server v1+v2

---

## Desafio 6.4 — Performance: Connection Pooling, Retry & Benchmarks

**Contexto:** gRPC usa HTTP/2, o que permite multiplexação de streams em uma
única conexão. Configuração correta é crucial para throughput.

### Requisitos

**Connection management:**

| Configuração | Valor | Razão |
|-------------|-------|-------|
| Keep-alive time | 30s | Detectar conexões mortas |
| Keep-alive timeout | 5s | Tempo para resposta ao ping |
| Max connection idle | 5min | Liberar conexões ociosas |
| Max concurrent streams | 100 | Por conexão HTTP/2 |
| Max message size (receive) | 4MB | Proteger contra payloads enormes |
| Max message size (send) | 4MB | Limitar response size |

**Retry policy:**

```json
{
  "methodConfig": [{
    "name": [{"service": "marketplace.product.v1.ProductService"}],
    "waitForReady": true,
    "timeout": "5s",
    "retryPolicy": {
      "maxAttempts": 3,
      "initialBackoff": "0.1s",
      "maxBackoff": "2s",
      "backoffMultiplier": 2.0,
      "retryableStatusCodes": ["UNAVAILABLE", "DEADLINE_EXCEEDED"]
    }
  }]
}
```

**Hedging (requests especulativos):**

```json
{
  "methodConfig": [{
    "name": [{"service": "marketplace.product.v1.ProductService", "method": "GetProduct"}],
    "hedgingPolicy": {
      "maxAttempts": 3,
      "hedgingDelay": "200ms",
      "nonFatalStatusCodes": ["UNAVAILABLE"]
    }
  }]
}
```

> **Nota:** Hedging é APENAS para RPCs idempotentes de leitura. NUNCA usar para mutações.

### Go 1.26 — Server tuning

```go
server := grpc.NewServer(
    grpc.KeepaliveParams(keepalive.ServerParameters{
        Time:    30 * time.Second,
        Timeout: 5 * time.Second,
    }),
    grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
        MinTime:             10 * time.Second,
        PermitWithoutStream: true,
    }),
    grpc.MaxRecvMsgSize(4 * 1024 * 1024), // 4MB
    grpc.MaxSendMsgSize(4 * 1024 * 1024),
    grpc.MaxConcurrentStreams(100),
)
```

**Benchmarks com `ghz`:**

```bash
# Instalar ghz
go install github.com/bojand/ghz/cmd/ghz@latest

# Benchmark unary
ghz --insecure \
    --proto proto/marketplace/product/v1/product_service.proto \
    --call marketplace.product.v1.ProductService/GetProduct \
    --data '{"id": "product-123"}' \
    --connections 10 \
    --concurrency 100 \
    --total 10000 \
    localhost:9090

# Resultado esperado (referência)
# Requests/sec: ~15000
# Avg latency:  ~6ms
# p99 latency:  ~20ms
```

### Java 25 — Connection pooling client

```java
// Channel com configuração de performance
var channel = ManagedChannelBuilder.forAddress("localhost", 9090)
    .usePlaintext()
    .maxInboundMessageSize(4 * 1024 * 1024)
    .keepAliveTime(30, TimeUnit.SECONDS)
    .keepAliveTimeout(5, TimeUnit.SECONDS)
    .keepAliveWithoutCalls(true)
    .defaultServiceConfig(Map.of(
        "methodConfig", List.of(Map.of(
            "name", List.of(Map.of("service", "marketplace.product.v1.ProductService")),
            "retryPolicy", Map.of(
                "maxAttempts", 3.0,
                "initialBackoff", "0.1s",
                "maxBackoff", "2s",
                "backoffMultiplier", 2.0,
                "retryableStatusCodes", List.of("UNAVAILABLE")
            )
        ))
    ))
    .enableRetry()
    .build();
```

### Critérios de Aceite

- [ ] Keep-alive configurado (30s time, 5s timeout)
- [ ] Max message size limitado (4MB)
- [ ] Max concurrent streams = 100 por conexão
- [ ] Retry policy: 3 tentativas, backoff exponential, apenas `UNAVAILABLE`
- [ ] Hedging configurado apenas para **read-only** RPCs
- [ ] Mutações (Create, Update, Delete) **sem** hedging
- [ ] Benchmark com `ghz`: documentar throughput e latency (p50, p99)
- [ ] Testes: simular server restart → client retry + reconnect

---

## Desafio 6.5 — Observabilidade: OpenTelemetry & Metrics

**Contexto:** Observabilidade em gRPC requer tracing distribuído, métricas
e logging estruturado integrados via interceptors.

### Requisitos

**Métricas (Prometheus-style):**

| Métrica | Tipo | Labels |
|---------|------|--------|
| `grpc_server_handled_total` | Counter | `method`, `code` |
| `grpc_server_handling_seconds` | Histogram | `method` |
| `grpc_server_msg_received_total` | Counter | `method` |
| `grpc_server_msg_sent_total` | Counter | `method` |
| `grpc_server_started_total` | Counter | `method` |
| `grpc_server_streams_active` | Gauge | `method` |

**Distributed Tracing (OpenTelemetry):**

```
Client                   ProductService           InventoryService
  │                          │                          │
  │   GetProduct (trace_id)  │                          │
  ├─────────────────────────►│                          │
  │                          │  GetStock (same trace_id)│
  │                          ├─────────────────────────►│
  │                          │◄─────────────────────────┤
  │◄─────────────────────────┤                          │
```

### Go 1.26

```go
// Metrics interceptor
type MetricsInterceptor struct {
    requestsTotal   map[string]map[string]int64 // method → code → count
    requestDuration map[string][]float64         // method → durations
    mu              sync.RWMutex
}

func (m *MetricsInterceptor) UnaryServerInterceptor() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler) (any, error) {

        start := time.Now()
        resp, err := handler(ctx, req)
        duration := time.Since(start).Seconds()
        code := status.Code(err).String()

        m.mu.Lock()
        if m.requestsTotal[info.FullMethod] == nil {
            m.requestsTotal[info.FullMethod] = make(map[string]int64)
        }
        m.requestsTotal[info.FullMethod][code]++
        m.requestDuration[info.FullMethod] = append(
            m.requestDuration[info.FullMethod], duration)
        m.mu.Unlock()

        return resp, err
    }
}

// Tracing interceptor (propagação de trace via metadata)
func tracingInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler) (any, error) {

    md, _ := metadata.FromIncomingContext(ctx)
    traceID := extractOrGenerate(md, "x-trace-id")
    spanID := generateSpanID()

    ctx = metadata.AppendToOutgoingContext(ctx,
        "x-trace-id", traceID,
        "x-parent-span-id", spanID,
    )

    slog.InfoContext(ctx, "span_start",
        "trace_id", traceID,
        "span_id", spanID,
        "method", info.FullMethod,
    )

    start := time.Now()
    resp, err := handler(ctx, req)

    slog.InfoContext(ctx, "span_end",
        "trace_id", traceID,
        "span_id", spanID,
        "method", info.FullMethod,
        "code", status.Code(err).String(),
        "duration_ms", time.Since(start).Milliseconds(),
    )

    return resp, err
}

// Endpoint para expor métricas (/metrics)
func (m *MetricsInterceptor) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    m.mu.RLock()
    defer m.mu.RUnlock()

    for method, codes := range m.requestsTotal {
        for code, count := range codes {
            fmt.Fprintf(w, "grpc_server_handled_total{method=%q,code=%q} %d\n",
                method, code, count)
        }
    }
    for method, durations := range m.requestDuration {
        p50, p99 := percentiles(durations, 0.50, 0.99)
        fmt.Fprintf(w, "grpc_server_handling_seconds{method=%q,quantile=\"0.5\"} %f\n", method, p50)
        fmt.Fprintf(w, "grpc_server_handling_seconds{method=%q,quantile=\"0.99\"} %f\n", method, p99)
    }
}
```

### Java 25

```java
// Metrics interceptor com Virtual Threads para coleta
public class MetricsInterceptor implements ServerInterceptor {
    private final ConcurrentHashMap<String, LongAdder> counters = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, List<Double>> histograms = new ConcurrentHashMap<>();

    @Override
    public <R, S> ServerCall.Listener<R> interceptCall(
            ServerCall<R, S> call, Metadata headers,
            ServerCallHandler<R, S> next) {

        var method = call.getMethodDescriptor().getFullMethodName();
        var start = Instant.now();

        counters.computeIfAbsent(method + ":started", _ -> new LongAdder()).increment();

        return new ForwardingServerCallListener.SimpleForwardingServerCallListener<>(
                next.startCall(new ForwardingServerCall.SimpleForwardingServerCall<>(call) {
                    @Override
                    public void close(Status status, Metadata trailers) {
                        var duration = Duration.between(start, Instant.now()).toNanos() / 1e9;
                        var key = method + ":" + status.getCode().name();
                        counters.computeIfAbsent(key, _ -> new LongAdder()).increment();
                        histograms.computeIfAbsent(method, _ ->
                            Collections.synchronizedList(new ArrayList<>())).add(duration);
                        super.close(status, trailers);
                    }
                }, headers)) {};
    }

    // Expor métricas no formato Prometheus
    public String scrape() {
        var sb = new StringBuilder();
        counters.forEach((key, value) ->
            sb.append("grpc_server_handled_total{key=\"%s\"} %d\n"
                .formatted(key, value.sum())));
        return sb.toString();
    }
}
```

### Critérios de Aceite

- [ ] 6 métricas gRPC coletadas (tabela acima)
- [ ] Endpoint HTTP `/metrics` expondo métricas (formato Prometheus)
- [ ] Percentis calculados: p50, p95, p99 para latency
- [ ] Trace ID propagado via metadata entre serviços
- [ ] Spans aninhados: ProductService → InventoryService com parent relationship
- [ ] Logs estruturados com trace_id, span_id, method, code, duration
- [ ] Dashboard: query para top-5 RPCs mais lentos
- [ ] Testes: verificar métricas incrementam, trace propagation funciona

---

## Desafio 6.6 — Testing: bufconn, Integration & Load

**Contexto:** gRPC requer estratégias de teste específicas.
`bufconn` permite testes in-process sem TCP real.

### Requisitos

**Pirâmide de testes gRPC:**

| Tipo | Ferramenta | O que testa |
|------|-----------|-------------|
| Unit | bufconn (Go) / InProcessServer (Java) | Service logic isolada |
| Integration | Server real + client | Interceptors + services + serialização |
| Contract | `buf breaking` | Compatibilidade de schema |
| Load | `ghz` | Performance sob carga |

### Go 1.26 — bufconn

```go
func setupTestServer(t *testing.T) pb.ProductServiceClient {
    t.Helper()

    lis := bufconn.Listen(1024 * 1024)

    server := grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            loggingInterceptor,
            validationInterceptor,
        ),
    )
    pb.RegisterProductServiceServer(server, NewProductServer())

    go func() {
        if err := server.Serve(lis); err != nil {
            t.Errorf("server exited with error: %v", err)
        }
    }()
    t.Cleanup(func() { server.GracefulStop() })

    dialer := func(context.Context, string) (net.Conn, error) {
        return lis.Dial()
    }

    conn, err := grpc.NewClient("passthrough:///bufnet",
        grpc.WithContextDialer(dialer),
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil { t.Fatal(err) }
    t.Cleanup(func() { conn.Close() })

    return pb.NewProductServiceClient(conn)
}

func TestCreateProduct(t *testing.T) {
    client := setupTestServer(t)
    ctx := context.Background()

    resp, err := client.CreateProduct(ctx, &pb.CreateProductRequest{
        Name:  "Test Product",
        Price: &pb.Money{Amount: 1990, CurrencyCode: "BRL"},
        Stock: 10,
    })
    if err != nil { t.Fatal(err) }

    if resp.GetProduct().GetName() != "Test Product" {
        t.Errorf("expected name 'Test Product', got %q", resp.GetProduct().GetName())
    }
    if resp.GetProduct().GetStatus() != pb.ProductStatus_PRODUCT_STATUS_ACTIVE {
        t.Errorf("expected ACTIVE status")
    }
}

func TestCreateProduct_Validation(t *testing.T) {
    client := setupTestServer(t)
    ctx := context.Background()

    _, err := client.CreateProduct(ctx, &pb.CreateProductRequest{
        Name: "", // empty — should fail validation
    })
    if err == nil { t.Fatal("expected error") }

    st := status.Convert(err)
    if st.Code() != codes.InvalidArgument {
        t.Errorf("expected InvalidArgument, got %s", st.Code())
    }
}

func TestWatchInventory_Stream(t *testing.T) {
    client := setupTestServer(t)
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    stream, err := client.WatchInventory(ctx, &pb.WatchInventoryRequest{})
    if err != nil { t.Fatal(err) }

    // Simular 3 updates no servidor (via test helper)
    triggerInventoryUpdates(3)

    received := 0
    for received < 3 {
        update, err := stream.Recv()
        if err != nil { t.Fatal(err) }
        if update.GetCurrentStock() < 0 {
            t.Error("stock must be >= 0")
        }
        received++
    }
    cancel()
}
```

### Java 25 — InProcessServer

```java
// Test com InProcessServer (sem TCP)
var serverName = InProcessServerBuilder.generateName();

var server = InProcessServerBuilder.forName(serverName)
    .directExecutor()
    .addService(new ProductServiceImpl())
    .build()
    .start();

var channel = InProcessChannelBuilder.forName(serverName)
    .directExecutor()
    .build();

var client = ProductServiceGrpc.newBlockingStub(channel);

// Test CRUD
var response = client.createProduct(CreateProductRequest.newBuilder()
    .setName("Test")
    .setPrice(Money.newBuilder().setAmount(990).setCurrencyCode("BRL"))
    .build());

assert response.getProduct().getName().equals("Test");
assert response.getProduct().getStatus() == ProductStatus.PRODUCT_STATUS_ACTIVE;

// Test validation error
try {
    client.createProduct(CreateProductRequest.newBuilder().build());
    throw new AssertionError("Expected exception");
} catch (StatusRuntimeException e) {
    assert e.getStatus().getCode() == Status.Code.INVALID_ARGUMENT;
}

// Cleanup
channel.shutdown();
server.shutdown();
```

**Load Test com `ghz`:**

```bash
# Criar cenário de carga
ghz --insecure \
    --proto proto/marketplace/product/v1/product_service.proto \
    --call marketplace.product.v1.ProductService/CreateProduct \
    --data '{"name":"Load Test","price":{"amount":990,"currency_code":"BRL"},"stock":10}' \
    --concurrency 50 \
    --duration 30s \
    --connections 5 \
    localhost:9090

# Verificar:
# - p99 < 100ms
# - Error rate < 1%
# - Throughput > 5000 req/s
```

### Critérios de Aceite

- [ ] **Unit tests** com bufconn (Go) / InProcessServer (Java) — sem TCP
- [ ] Testes para sucesso e cada tipo de erro (INVALID_ARGUMENT, NOT_FOUND, etc.)
- [ ] Testes de streaming: receive N messages, cancel, verify cleanup
- [ ] **Contract tests**: `buf breaking` integrado no CI
- [ ] **Load tests** com `ghz`: cenários para unary e streaming
- [ ] Métricas de performance documentadas: p50, p99, throughput
- [ ] Cobertura de testes ≥ 80% dos RPCs
- [ ] Test fixtures reutilizáveis (setup/teardown do server)

---

## Desafio 6.7 — Architecture: gRPC-Gateway, Reflection & Graceful Shutdown

**Contexto:** gRPC-Gateway permite expor serviços gRPC como REST API automaticamente.
Integre com o REST API público (Levels 1-2) e GraphQL BFF (Levels 3-4).

### Requisitos

**gRPC-Gateway (REST → gRPC translation):**

```protobuf
import "google/api/annotations.proto";

service ProductService {
  rpc GetProduct(GetProductRequest) returns (Product) {
    option (google.api.http) = {
      get: "/api/v1/products/{id}"
    };
  }

  rpc ListProducts(ListProductsRequest) returns (ListProductsResponse) {
    option (google.api.http) = {
      get: "/api/v1/products"
    };
  }

  rpc CreateProduct(CreateProductRequest) returns (CreateProductResponse) {
    option (google.api.http) = {
      post: "/api/v1/products"
      body: "*"
    };
  }

  rpc UpdateProduct(UpdateProductRequest) returns (Product) {
    option (google.api.http) = {
      patch: "/api/v1/products/{id}"
      body: "product"
    };
  }

  rpc DeleteProduct(DeleteProductRequest) returns (DeleteProductResponse) {
    option (google.api.http) = {
      delete: "/api/v1/products/{id}"
    };
  }
}
```

**Mapeamento REST ↔ gRPC:**

| REST | gRPC |
|------|------|
| `GET /api/v1/products/123` | `GetProduct({id: "123"})` |
| `GET /api/v1/products?page_size=10` | `ListProducts({pagination: {page_size: 10}})` |
| `POST /api/v1/products` + JSON body | `CreateProduct(body)` |
| HTTP 200 | gRPC OK |
| HTTP 404 | gRPC NOT_FOUND |
| HTTP 400 | gRPC INVALID_ARGUMENT |

### Go 1.26 — gRPC-Gateway + gRPC no mesmo port

```go
func main() {
    // 1. gRPC server
    grpcServer := grpc.NewServer()
    pb.RegisterProductServiceServer(grpcServer, NewProductServer())

    // 2. gRPC-Gateway (REST proxy)
    ctx := context.Background()
    mux := runtime.NewServeMux(
        runtime.WithErrorHandler(customErrorHandler),
        runtime.WithMarshalerOption(
            runtime.MIMEWildcard,
            &runtime.JSONPb{MarshalOptions: protojson.MarshalOptions{
                EmitUnpopulated: true,
                UseProtoNames:   true,
            }},
        ),
    )
    opts := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}
    pb.RegisterProductServiceHandlerFromEndpoint(ctx, mux, "localhost:9090", opts)

    // 3. Multiplexer: HTTP/2 (gRPC) OU HTTP/1.1 (REST)
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.ProtoMajor == 2 && strings.HasPrefix(
            r.Header.Get("Content-Type"), "application/grpc") {
            grpcServer.ServeHTTP(w, r)
        } else {
            mux.ServeHTTP(w, r)
        }
    })

    // 4. Listener com TLS
    lis, _ := net.Listen("tcp", ":9090")
    httpServer := &http.Server{Handler: handler}
    httpServer.Serve(lis)
}
```

**Reflection (para tooling):**

```go
import "google.golang.org/grpc/reflection"

// Habilitar reflection para grpcurl e outros tools
reflection.Register(grpcServer)
```

```bash
# Listar serviços
grpcurl -plaintext localhost:9090 list

# Descrever serviço
grpcurl -plaintext localhost:9090 describe marketplace.product.v1.ProductService

# Chamar RPC
grpcurl -plaintext -d '{"id":"product-123"}' \
    localhost:9090 marketplace.product.v1.ProductService/GetProduct
```

**Graceful Shutdown:**

```go
func gracefulShutdown(grpcServer *grpc.Server, httpServer *http.Server) {
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    slog.Info("initiating graceful shutdown...")

    // 1. Parar de aceitar novas conexões
    // 2. Health check → NOT_SERVING
    healthServer.SetStatus("", grpc_health_v1.HealthCheckResponse_NOT_SERVING)

    // 3. Aguardar drain period (inflight requests)
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // 4. Shutdown HTTP gateway
    httpServer.Shutdown(ctx)

    // 5. Graceful stop gRPC (espera RPCs em andamento terminarem)
    grpcServer.GracefulStop()

    slog.Info("server stopped gracefully")
}
```

### Critérios de Aceite

- [ ] **gRPC-Gateway** gera REST endpoints a partir das annotations `.proto`
- [ ] `GET /api/v1/products/123` → resposta JSON correta
- [ ] Status codes HTTP mapeados corretamente dos gRPC codes
- [ ] gRPC e REST no mesmo port (content-type switch)
- [ ] **Reflection** habilitada — `grpcurl list` funciona
- [ ] **Graceful shutdown** com drain period de 30s
- [ ] Health check muda para NOT_SERVING durante shutdown
- [ ] Inflight requests completam antes do server parar
- [ ] Testes: shutdown durante stream ativo → stream completa

---

## Anti-Patterns a Evitar

| Anti-Pattern | Correto |
|-------------|---------|
| gRPC sem TLS em produção | TLS mínimo, mTLS entre serviços |
| Ignorar health check protocol | Implementar `grpc.health.v1.Health` |
| Sem métricas gRPC | Interceptor de métricas com 6 métricas padrão |
| `server.Stop()` imediato | `GracefulStop()` com drain period |
| Testes com TCP real | `bufconn` (Go) / `InProcessServer` (Java) |
| Sem `buf lint` / `buf breaking` | CI com validação de schema |
| `shutdownNow()` sem timeout | Timeout de 30s + fallback para `shutdownNow()` |
| gRPC-Gateway sem error handler | Custom error handler com mapeamento correto |

---

## Próximo Nível

→ [Level 7 — Capstone: Multi-Paradigm API Platform](07-capstone-multi-paradigm.md)
