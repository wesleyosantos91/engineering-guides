# Level 7 — Capstone: Multi-Paradigm API Platform

> **Objetivo:** Integrar REST, GraphQL e gRPC em uma plataforma completa do Online Marketplace.
> REST para APIs externas públicas, GraphQL como BFF para storefront, gRPC para comunicação interna entre microserviços.

**Referência:** Todos os documentos em [.docs/API/](../../../.docs/API/)

---

## Arquitetura Alvo

```
                    ┌──────────────────────────────────────────────────────┐
                    │                   API Gateway                        │
                    │        (Auth, Rate Limit, Routing)                    │
                    └────────┬──────────────┬──────────────┬───────────────┘
                             │              │              │
                    ┌────────▼───────┐ ┌────▼────────┐ ┌──▼──────────────┐
                    │  REST API      │ │ GraphQL BFF │ │ gRPC-Gateway    │
                    │  /api/v1/*     │ │ /graphql    │ │ (REST → gRPC)   │
                    │  (Parceiros)   │ │ (Storefront)│ │                 │
                    └────────┬───────┘ └────┬────────┘ └──┬──────────────┘
                             │              │              │
                    ┌────────▼──────────────▼──────────────▼───────────────┐
                    │                    gRPC Internal                      │
                    ├──────────┬──────────┬──────────┬──────────┬──────────┤
                    │ Product  │  Order   │ Inventory│ Payment  │ Notif.   │
                    │ Service  │  Service │ Service  │ Service  │ Service  │
                    └──────────┴──────────┴──────────┴──────────┴──────────┘
```

**Paradigma por camada:**

| Camada | Protocolo | Consumidores | Justificativa |
|--------|:---------:|-------------|---------------|
| API Pública | REST | Parceiros, Apps 3rd-party | Interoperabilidade máxima |
| Storefront BFF | GraphQL | Web SPA, Mobile App | Flexibilidade de queries, reduz over-fetching |
| Admin BFF | GraphQL | Dashboard admin | Queries complexas com aggregations |
| Internal | gRPC | Microserviços | Performance, type safety, streaming |
| Gateway REST→gRPC | gRPC-Gateway | Legacy clients | Backward compatibility |

---

## Desafio 7.1 — gRPC Microservices Core

**Contexto:** O backbone interno do Marketplace são 5 microserviços gRPC.
Cada um é independente, com seu próprio `.proto`, health check e interceptors.

### Requisitos

Implementar 5 serviços gRPC internos:

| Serviço | Responsabilidade | RPCs Principais |
|---------|-----------------|-----------------|
| ProductService | Catálogo de produtos | CRUD, Search, BulkCreate |
| OrderService | Gestão de pedidos | Create, Get, List, UpdateStatus, Cancel |
| InventoryService | Controle de estoque | Reserve, Release, GetStock, WatchUpdates |
| PaymentService | Processamento de pagamentos | Process, Refund, GetStatus |
| NotificationService | Envio de notificações | Send, Subscribe, GetHistory |

**Order Service com workflow:**

```protobuf
service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse);
  rpc UpdateOrderStatus(UpdateOrderStatusRequest) returns (Order);
  rpc CancelOrder(CancelOrderRequest) returns (CancelOrderResponse);

  // Stream de status updates de um pedido
  rpc WatchOrderStatus(WatchOrderStatusRequest) returns (stream OrderStatusUpdate);
}

message Order {
  string id = 1;
  string buyer_id = 2;
  repeated OrderItem items = 3;
  marketplace.common.v1.Money total = 4;
  OrderStatus status = 5;
  google.protobuf.Timestamp created_at = 6;
  google.protobuf.Timestamp updated_at = 7;
  ShippingAddress shipping_address = 8;
  string payment_id = 9;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_PAID = 3;
  ORDER_STATUS_SHIPPED = 4;
  ORDER_STATUS_DELIVERED = 5;
  ORDER_STATUS_CANCELLED = 6;
  ORDER_STATUS_REFUNDED = 7;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  marketplace.common.v1.Money unit_price = 3;
}
```

**Workflow de pedido (orchestração via gRPC):**

```
CreateOrder:
  1. OrderService.CreateOrder (status: PENDING)
  2. InventoryService.Reserve (reservar estoque)
     → Se falhar: OrderService.CancelOrder (status: CANCELLED)
  3. PaymentService.Process (cobrar pagamento)
     → Se falhar: InventoryService.Release + OrderService.CancelOrder
  4. OrderService.UpdateStatus (status: CONFIRMED → PAID)
  5. NotificationService.Send (notificar buyer)
```

### Go 1.26 — Order orchestration

```go
func (s *OrderServer) CreateOrder(ctx context.Context,
    req *pb.CreateOrderRequest) (*pb.CreateOrderResponse, error) {

    // 1. Criar pedido com status PENDING
    order := &pb.Order{
        Id:              uuid.New().String(),
        BuyerId:         req.GetBuyerId(),
        Items:           req.GetItems(),
        Status:          pb.OrderStatus_ORDER_STATUS_PENDING,
        ShippingAddress: req.GetShippingAddress(),
        CreatedAt:       timestamppb.Now(),
        UpdatedAt:       timestamppb.Now(),
    }

    // Calcular total
    var totalAmount int64
    for _, item := range order.Items {
        totalAmount += item.GetUnitPrice().GetAmount() * int64(item.GetQuantity())
    }
    order.Total = &commonpb.Money{Amount: totalAmount, CurrencyCode: "BRL"}

    s.mu.Lock()
    s.orders[order.Id] = order
    s.mu.Unlock()

    // 2. Reservar estoque (com deadline propagado)
    for _, item := range order.Items {
        _, err := s.inventoryClient.Reserve(ctx, &inventorypb.ReserveRequest{
            ProductId: item.GetProductId(),
            Quantity:  item.GetQuantity(),
            OrderId:   order.Id,
        })
        if err != nil {
            // Compensação: liberar estoque já reservado + cancelar pedido
            s.compensateReservations(ctx, order)
            return nil, status.Errorf(codes.FailedPrecondition,
                "insufficient stock for product %s", item.GetProductId())
        }
    }

    // 3. Processar pagamento
    paymentResp, err := s.paymentClient.Process(ctx, &paymentpb.ProcessRequest{
        OrderId: order.Id,
        Amount:  order.Total,
        BuyerId: order.BuyerId,
    })
    if err != nil {
        // Compensação: liberar todo o estoque reservado
        s.compensateReservations(ctx, order)
        return nil, status.Errorf(codes.Internal, "payment failed: %v", err)
    }

    // 4. Atualizar status → PAID
    s.mu.Lock()
    order.Status = pb.OrderStatus_ORDER_STATUS_PAID
    order.PaymentId = paymentResp.GetPaymentId()
    order.UpdatedAt = timestamppb.Now()
    s.mu.Unlock()

    // 5. Notificar (fire-and-forget via goroutine)
    go func() {
        s.notificationClient.Send(context.Background(), &notifpb.SendRequest{
            UserId:  order.BuyerId,
            Type:    notifpb.NotificationType_NOTIFICATION_TYPE_ORDER_CONFIRMED,
            Title:   "Pedido confirmado!",
            Message: fmt.Sprintf("Pedido %s foi confirmado.", order.Id),
        })
    }()

    return &pb.CreateOrderResponse{Order: order}, nil
}
```

### Critérios de Aceite

- [ ] 5 serviços gRPC implementados com CRUD completo
- [ ] Workflow de pedido com orchestração sequencial
- [ ] Compensação (saga): rollback de reservas se pagamento falhar
- [ ] Deadlines propagados entre serviços
- [ ] Notificação fire-and-forget (não bloqueia o response)
- [ ] Cada serviço com health check independente
- [ ] Testes: fluxo completo sucesso + falha em cada passo

---

## Desafio 7.2 — REST API Pública (Facade)

**Contexto:** A API REST pública é o contrato com parceiros e apps de terceiros.
Funciona como facade sobre os serviços gRPC internos.

### Requisitos

Implementar a API REST do Marketplace usando stdlib Java/Go,
delegando para os serviços gRPC internos:

**Endpoints REST:**

| Método | URI | gRPC Backend |
|--------|-----|-------------|
| `GET /api/v1/products` | `ProductService.ListProducts` |
| `GET /api/v1/products/{id}` | `ProductService.GetProduct` |
| `POST /api/v1/products` | `ProductService.CreateProduct` |
| `PATCH /api/v1/products/{id}` | `ProductService.UpdateProduct` |
| `DELETE /api/v1/products/{id}` | `ProductService.DeleteProduct` |
| `POST /api/v1/orders` | `OrderService.CreateOrder` |
| `GET /api/v1/orders/{id}` | `OrderService.GetOrder` |
| `GET /api/v1/orders` | `OrderService.ListOrders` |
| `POST /api/v1/orders/{id}/cancel` | `OrderService.CancelOrder` |

**Features REST (dos Levels 1-2):**

- Paginação cursor-based (`?cursor=xxx&size=20`)
- Filtering (`?category=electronics&min_price=1000`)
- HATEOAS links (HAL format)
- Error handling RFC 9457 Problem Details
- ETag & Cache-Control headers
- Idempotency-Key para POST
- Rate limiting (3 tiers: free=100/h, pro=1000/h, enterprise=10000/h)
- JWT authentication

### Go 1.26 — REST facade

```go
type RESTHandler struct {
    productClient   pb.ProductServiceClient
    orderClient     pb.OrderServiceClient
    inventoryClient pb.InventoryServiceClient
}

func (h *RESTHandler) GetProduct(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")

    // Propagar deadline do HTTP para gRPC
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()

    product, err := h.productClient.GetProduct(ctx, &pb.GetProductRequest{Id: id})
    if err != nil {
        writeGRPCError(w, err) // Converte gRPC status → RFC 9457
        return
    }

    // ETag baseado no updated_at
    etag := fmt.Sprintf(`"%x"`, product.GetUpdatedAt().AsTime().UnixNano())
    if r.Header.Get("If-None-Match") == etag {
        w.WriteHeader(http.StatusNotModified)
        return
    }

    response := ProductResponse{
        ID:          product.GetId(),
        Name:        product.GetName(),
        Description: product.GetDescription(),
        Price:       MoneyResponse{Amount: product.GetPrice().GetAmount(), Currency: product.GetPrice().GetCurrencyCode()},
        Status:      product.GetStatus().String(),
        Links: map[string]Link{
            "self":     {Href: fmt.Sprintf("/api/v1/products/%s", product.GetId())},
            "seller":   {Href: fmt.Sprintf("/api/v1/users/%s", product.GetSellerId())},
            "reviews":  {Href: fmt.Sprintf("/api/v1/products/%s/reviews", product.GetId())},
        },
    }

    w.Header().Set("ETag", etag)
    w.Header().Set("Cache-Control", "public, max-age=60")
    writeJSON(w, http.StatusOK, response)
}

// gRPC status → RFC 9457 Problem Details
func writeGRPCError(w http.ResponseWriter, err error) {
    st := status.Convert(err)
    httpCode := grpcToHTTPStatus(st.Code())

    problem := ProblemDetail{
        Type:     fmt.Sprintf("https://marketplace.com/errors/%s", strings.ToLower(st.Code().String())),
        Title:    st.Code().String(),
        Status:   httpCode,
        Detail:   st.Message(),
        Instance: fmt.Sprintf("/errors/%s", uuid.New().String()),
    }

    // Extrair field violations do Rich Error Model
    for _, detail := range st.Details() {
        if br, ok := detail.(*errdetails.BadRequest); ok {
            for _, fv := range br.GetFieldViolations() {
                problem.Errors = append(problem.Errors, FieldError{
                    Field:   fv.GetField(),
                    Message: fv.GetDescription(),
                })
            }
        }
    }

    w.Header().Set("Content-Type", "application/problem+json")
    writeJSON(w, httpCode, problem)
}

func grpcToHTTPStatus(code codes.Code) int {
    switch code {
    case codes.OK:                 return 200
    case codes.InvalidArgument:    return 400
    case codes.Unauthenticated:    return 401
    case codes.PermissionDenied:   return 403
    case codes.NotFound:           return 404
    case codes.AlreadyExists:      return 409
    case codes.ResourceExhausted:  return 429
    case codes.Internal:           return 500
    case codes.Unavailable:        return 503
    case codes.DeadlineExceeded:   return 504
    default:                       return 500
    }
}
```

### Java 25 — REST facade

```java
void handleGetProduct(HttpExchange exchange) throws IOException {
    var id = extractPathParam(exchange, "id");

    try {
        var product = productStub
            .withDeadlineAfter(2, TimeUnit.SECONDS)
            .getProduct(GetProductRequest.newBuilder().setId(id).build());

        // ETag
        var etag = "\"%x\"".formatted(
            product.getUpdatedAt().getSeconds());
        if (etag.equals(exchange.getRequestHeaders().getFirst("If-None-Match"))) {
            exchange.sendResponseHeaders(304, -1);
            return;
        }

        var response = Map.of(
            "id", product.getId(),
            "name", product.getName(),
            "price", Map.of(
                "amount", product.getPrice().getAmount(),
                "currency", product.getPrice().getCurrencyCode()
            ),
            "_links", Map.of(
                "self", Map.of("href", "/api/v1/products/" + product.getId()),
                "seller", Map.of("href", "/api/v1/users/" + product.getSellerId())
            )
        );

        exchange.getResponseHeaders().add("ETag", etag);
        exchange.getResponseHeaders().add("Cache-Control", "public, max-age=60");
        writeJson(exchange, 200, response);
    } catch (StatusRuntimeException e) {
        writeGrpcError(exchange, e);
    }
}
```

### Critérios de Aceite

- [ ] Todos os endpoints REST da tabela implementados
- [ ] REST facade delega para gRPC services (não acessa store diretamente)
- [ ] Conversão gRPC status → HTTP status correto (tabela de mapeamento)
- [ ] Rich Error Model → RFC 9457 Problem Details com field violations
- [ ] ETag + Cache-Control funcional
- [ ] Paginação cursor-based (gRPC page_token → cursor)
- [ ] HATEOAS links HAL em todas as respostas
- [ ] Rate limiting por tier
- [ ] Idempotency-Key para POST/PATCH

---

## Desafio 7.3 — GraphQL BFF (Storefront)

**Contexto:** O GraphQL BFF agrega dados de múltiplos serviços gRPC para
o storefront web/mobile, evitando over-fetching.

### Requisitos

**Schema GraphQL do Storefront:**

```graphql
type Query {
  product(id: ID!): Product
  products(
    filter: ProductFilterInput
    sort: ProductSortInput
    first: Int
    after: String
  ): ProductConnection!

  order(id: ID!): Order
  myOrders(first: Int, after: String): OrderConnection!

  # Aggregations para dashboard
  categoryStats: [CategoryStat!]!
}

type Mutation {
  addToCart(input: AddToCartInput!): AddToCartPayload!
  checkout(input: CheckoutInput!): CheckoutPayload!
  cancelOrder(id: ID!): CancelOrderPayload!
}

type Subscription {
  orderStatusChanged(orderId: ID!): OrderStatusUpdate!
  inventoryAlert(productIds: [ID!]!): InventoryAlert!
}

type Product implements Node {
  id: ID!
  name: String!
  description: String!
  price: Money!
  category: Category!        # Resolvido via DataLoader → gRPC
  seller: User!              # Resolvido via DataLoader → gRPC
  stock: Int!                # Resolvido via InventoryService gRPC
  reviews(first: Int): ReviewConnection!
}

type Order implements Node {
  id: ID!
  items: [OrderItem!]!
  total: Money!
  status: OrderStatus!
  createdAt: DateTime!
  buyer: User!               # Resolvido via DataLoader → gRPC
  payment: Payment           # Resolvido via PaymentService gRPC
  trackingUpdates: [TrackingUpdate!]!
}

# Inputs usando o padrão Input/Payload
input CheckoutInput {
  clientMutationId: String
  items: [OrderItemInput!]!
  shippingAddress: AddressInput!
}

type CheckoutPayload {
  clientMutationId: String
  order: Order
  userErrors: [UserError!]!
}
```

**DataLoaders batching gRPC calls:**

### Go 1.26 — GraphQL resolver com gRPC

```go
// DataLoader: batch GetProduct calls
type ProductLoader struct {
    client pb.ProductServiceClient
}

func (l *ProductLoader) BatchGet(ctx context.Context, ids []string) ([]*pb.Product, []error) {
    // Batch: uma chamada ListProducts com filtro por IDs
    resp, err := l.client.ListProducts(ctx, &pb.ListProductsRequest{
        Filter: &pb.ProductFilter{
            Ids: ids, // assumindo que adicionamos filtro por IDs
        },
        Pagination: &commonpb.PaginationRequest{PageSize: int32(len(ids))},
    })
    if err != nil {
        errs := make([]error, len(ids))
        for i := range errs { errs[i] = err }
        return nil, errs
    }

    // Mapear resultados na mesma ordem dos IDs solicitados
    productMap := make(map[string]*pb.Product)
    for _, p := range resp.GetProducts() {
        productMap[p.GetId()] = p
    }

    results := make([]*pb.Product, len(ids))
    errors := make([]error, len(ids))
    for i, id := range ids {
        if p, ok := productMap[id]; ok {
            results[i] = p
        } else {
            errors[i] = fmt.Errorf("product not found: %s", id)
        }
    }
    return results, errors
}

// Resolver: checkout = orchestration via gRPC
func (r *MutationResolver) Checkout(ctx context.Context,
    input CheckoutInput) (*CheckoutPayload, error) {

    // Converter GraphQL input → gRPC request
    grpcItems := make([]*orderpb.OrderItem, len(input.Items))
    for i, item := range input.Items {
        grpcItems[i] = &orderpb.OrderItem{
            ProductId: item.ProductID,
            Quantity:  int32(item.Quantity),
        }
    }

    // Chamar OrderService.CreateOrder (que orquestra inventory + payment)
    resp, err := r.orderClient.CreateOrder(ctx, &orderpb.CreateOrderRequest{
        BuyerId: getCurrentUserID(ctx),
        Items:   grpcItems,
        ShippingAddress: &orderpb.ShippingAddress{
            Street:  input.ShippingAddress.Street,
            City:    input.ShippingAddress.City,
            ZipCode: input.ShippingAddress.ZipCode,
        },
    })
    if err != nil {
        return &CheckoutPayload{
            UserErrors: convertGRPCErrors(err),
        }, nil
    }

    return &CheckoutPayload{
        Order:            convertOrder(resp.GetOrder()),
        ClientMutationId: input.ClientMutationId,
    }, nil
}

// Subscription: gRPC stream → GraphQL subscription
func (r *SubscriptionResolver) OrderStatusChanged(ctx context.Context,
    orderID string) (<-chan *OrderStatusUpdate, error) {

    stream, err := r.orderClient.WatchOrderStatus(ctx,
        &orderpb.WatchOrderStatusRequest{OrderId: orderID})
    if err != nil {
        return nil, err
    }

    ch := make(chan *OrderStatusUpdate, 10)
    go func() {
        defer close(ch)
        for {
            update, err := stream.Recv()
            if err != nil {
                return
            }
            ch <- &OrderStatusUpdate{
                OrderID:    update.GetOrderId(),
                OldStatus:  update.GetOldStatus().String(),
                NewStatus:  update.GetNewStatus().String(),
                OccurredAt: update.GetTimestamp().AsTime(),
            }
        }
    }()
    return ch, nil
}
```

### Critérios de Aceite

- [ ] Schema GraphQL completo com Product, Order, User, Payment
- [ ] Queries resolvidas via gRPC clients (não acessa store diretamente)
- [ ] DataLoaders para batch gRPC calls (evitar N+1)
- [ ] Mutations delegam para gRPC services (checkout → OrderService)
- [ ] Subscriptions conectadas a gRPC server streams
- [ ] Error handling: gRPC errors → GraphQL UserError en payload
- [ ] Relay Connection pagination (ProductConnection, OrderConnection)
- [ ] Query depth limit ≤ 5, complexity limit

---

## Desafio 7.4 — API Gateway Unificado

**Contexto:** O API Gateway é o ponto de entrada único que roteia para
REST, GraphQL ou gRPC-Gateway conforme o path/protocolo.

### Requisitos

**Routing rules:**

| Rota | Destino | Auth |
|------|---------|:----:|
| `/api/v1/*` | REST Facade | JWT |
| `/graphql` | GraphQL BFF | JWT |
| `/grpc.*` | gRPC (HTTP/2) | mTLS + JWT |
| `/_health` | Health aggregator | Nenhum |
| `/_metrics` | Métricas consolidadas | Internal |

### Go 1.26 — API Gateway

```go
func main() {
    mux := http.NewServeMux()

    // Middleware chain (aplicado a todos os requests)
    handler := chain(
        corsMiddleware,
        requestIDMiddleware,
        loggingMiddleware,
        rateLimitMiddleware,
        mux,
    )

    // REST routes → REST facade
    restHandler := NewRESTHandler(productGRPC, orderGRPC)
    mux.HandleFunc("GET /api/v1/products", restHandler.ListProducts)
    mux.HandleFunc("GET /api/v1/products/{id}", restHandler.GetProduct)
    mux.HandleFunc("POST /api/v1/products", withAuth(restHandler.CreateProduct))
    mux.HandleFunc("PATCH /api/v1/products/{id}", withAuth(restHandler.UpdateProduct))
    mux.HandleFunc("DELETE /api/v1/products/{id}", withAuth(restHandler.DeleteProduct))
    mux.HandleFunc("POST /api/v1/orders", withAuth(restHandler.CreateOrder))
    mux.HandleFunc("GET /api/v1/orders/{id}", withAuth(restHandler.GetOrder))

    // GraphQL route → BFF
    graphqlHandler := NewGraphQLHandler(productGRPC, orderGRPC, inventoryGRPC)
    mux.Handle("POST /graphql", withAuth(graphqlHandler))

    // Health aggregator
    mux.HandleFunc("GET /_health", func(w http.ResponseWriter, r *http.Request) {
        healthStatus := aggregateHealth(
            checkGRPCHealth(productConn, "marketplace.product.v1.ProductService"),
            checkGRPCHealth(orderConn, "marketplace.order.v1.OrderService"),
            checkGRPCHealth(inventoryConn, "marketplace.inventory.v1.InventoryService"),
        )
        if healthStatus.Overall == "SERVING" {
            writeJSON(w, 200, healthStatus)
        } else {
            writeJSON(w, 503, healthStatus)
        }
    })

    // Metrics endpoint
    mux.HandleFunc("GET /_metrics", metricsHandler.ServeHTTP)

    server := &http.Server{
        Addr:    ":8080",
        Handler: handler,
    }

    slog.Info("API Gateway started", "port", 8080)
    go gracefulShutdown(server)
    server.ListenAndServe()
}
```

### Java 25 — API Gateway

```java
void main() throws Exception {
    var server = HttpServer.create(new InetSocketAddress(8080), 0);
    server.setExecutor(Executors.newVirtualThreadPerTaskExecutor());

    // REST routes
    var restHandler = new RESTHandler(productStub, orderStub);
    server.createContext("/api/v1/products", restHandler::handle);
    server.createContext("/api/v1/orders", restHandler::handleOrders);

    // GraphQL
    var graphqlHandler = new GraphQLHandler(productStub, orderStub, inventoryStub);
    server.createContext("/graphql", graphqlHandler::handle);

    // Health aggregator
    server.createContext("/_health", exchange -> {
        var health = aggregateHealth(
            checkGRPCHealth(productChannel),
            checkGRPCHealth(orderChannel),
            checkGRPCHealth(inventoryChannel)
        );
        var code = health.overall().equals("SERVING") ? 200 : 503;
        writeJson(exchange, code, health);
    });

    server.start();
    System.out.println("API Gateway started on :8080");
}
```

### Critérios de Aceite

- [ ] Gateway roteia corretamente para REST, GraphQL e gRPC
- [ ] Middleware chain aplicado a todos os requests (CORS, logging, rate limit)
- [ ] Auth JWT validado no gateway (não em cada service)
- [ ] `/_health` agrega health de todos os gRPC services
- [ ] `/_metrics` consolida métricas de todos os backends
- [ ] Request ID propagado: gateway → REST/GraphQL → gRPC (via metadata)
- [ ] Rate limiting por API key/tier
- [ ] Graceful shutdown coordenado (gateway + backends)

---

## Desafio 7.5 — Observabilidade End-to-End

**Contexto:** Rastrear um request desde o gateway até o último microserviço gRPC.

### Requisitos

**Trace propagation completa:**

```
Client (trace_id: abc123)
  └─ Gateway (span: gateway, trace_id: abc123)
      ├─ REST /api/v1/orders (span: rest.createOrder)
      │   └─ OrderService.CreateOrder (span: grpc.order.create)
      │       ├─ InventoryService.Reserve (span: grpc.inventory.reserve)
      │       ├─ PaymentService.Process (span: grpc.payment.process)
      │       └─ NotificationService.Send (span: grpc.notification.send)
      └─ GraphQL /graphql (span: graphql.checkout)
          └─ OrderService.CreateOrder (span: grpc.order.create)
              └─ ... (mesma cadeia)
```

**Structured logging unificado:**

```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "INFO",
  "service": "order-service",
  "trace_id": "abc123",
  "span_id": "def456",
  "parent_span_id": "ghi789",
  "method": "marketplace.order.v1.OrderService/CreateOrder",
  "protocol": "grpc",
  "duration_ms": 45,
  "status": "OK",
  "user_id": "user-789",
  "metadata": {
    "order_id": "order-456",
    "items_count": 3,
    "total_amount": 15999
  }
}
```

**Dashboard queries (métricas consolidadas):**

| Query | Descrição |
|-------|-----------|
| Latency p99 por endpoint | REST + GraphQL + gRPC |
| Error rate por serviço | Últimos 5 minutos |
| Throughput total | Requests/sec por protocolo |
| Trace mais lento | Waterfall completo |
| N+1 detection | DataLoader savings ratio |

### Critérios de Aceite

- [ ] Trace ID único propagado: Gateway → REST/GraphQL → gRPC chain
- [ ] Cada hop gera span com parent_span_id correto
- [ ] Logs estruturados com trace_id em todos os serviços
- [ ] Métricas por protocolo (REST, GraphQL, gRPC) separadas
- [ ] Endpoint `/_metrics` com todas as métricas consolidadas
- [ ] Alertas: error rate > 5%, p99 > 500ms
- [ ] Testes: verificar trace completo com 4+ spans

---

## Desafio 7.6 — Resilience: Circuit Breaker & Saga

**Contexto:** Em um sistema distribuído, falhas são inevitáveis.
Circuit breakers protegem contra cascading failures.

### Requisitos

**Circuit Breaker (3 estados):**

```
       successThreshold
            reached
  ┌─────────────────────┐
  │                     ▼
CLOSED ──────────► OPEN ──────────► HALF_OPEN
  ▲  failureThreshold   timeout        │
  │  exceeded            elapsed       │
  └─────────────────────────────────────┘
              successThreshold
              NOT reached → OPEN
```

| Parâmetro | Valor |
|-----------|-------|
| Failure threshold | 5 falhas consecutivas |
| Success threshold | 3 sucessos em HALF_OPEN |
| Open timeout | 10 segundos |
| Monitored codes | `UNAVAILABLE`, `DEADLINE_EXCEEDED`, `INTERNAL` |

### Go 1.26 — Circuit Breaker

```go
type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

type CircuitBreaker struct {
    mu               sync.Mutex
    state            State
    failures         int
    successes        int
    failureThreshold int
    successThreshold int
    openTimeout      time.Duration
    lastFailure      time.Time
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    cb.mu.Lock()
    if cb.state == StateOpen {
        if time.Since(cb.lastFailure) > cb.openTimeout {
            cb.state = StateHalfOpen
            cb.successes = 0
        } else {
            cb.mu.Unlock()
            return fmt.Errorf("circuit breaker is open")
        }
    }
    cb.mu.Unlock()

    err := fn()

    cb.mu.Lock()
    defer cb.mu.Unlock()

    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        if cb.failures >= cb.failureThreshold {
            cb.state = StateOpen
            slog.Warn("circuit breaker opened", "failures", cb.failures)
        }
        if cb.state == StateHalfOpen {
            cb.state = StateOpen
        }
        return err
    }

    if cb.state == StateHalfOpen {
        cb.successes++
        if cb.successes >= cb.successThreshold {
            cb.state = StateClosed
            cb.failures = 0
            slog.Info("circuit breaker closed")
        }
    } else {
        cb.failures = 0
    }
    return nil
}

// Usar com gRPC client
func (h *RESTHandler) GetProduct(w http.ResponseWriter, r *http.Request) {
    err := h.productCB.Execute(func() error {
        product, err := h.productClient.GetProduct(r.Context(),
            &pb.GetProductRequest{Id: r.PathValue("id")})
        if err != nil {
            code := status.Code(err)
            if code == codes.Unavailable || code == codes.DeadlineExceeded {
                return err // que o CB monitore
            }
            writeGRPCError(w, err) // erro de negócio, não conta
            return nil
        }
        writeJSON(w, 200, convertProduct(product))
        return nil
    })
    if err != nil {
        writeProblem(w, 503, "Service temporarily unavailable")
    }
}
```

**Saga Pattern (compensação distribuída):**

| Passo | Ação | Compensação |
|:-----:|------|-------------|
| 1 | `InventoryService.Reserve` | `InventoryService.Release` |
| 2 | `PaymentService.Process` | `PaymentService.Refund` |
| 3 | `OrderService.Confirm` | `OrderService.Cancel` |
| 4 | `NotificationService.Send` | — (sem compensação) |

### Critérios de Aceite

- [ ] Circuit breaker implementado com 3 estados
- [ ] OPEN retorna fallback imediato (sem chamar serviço)
- [ ] HALF_OPEN permite N requests de teste
- [ ] Apenas `UNAVAILABLE` e `DEADLINE_EXCEEDED` contam como failure
- [ ] Erros de negócio (NOT_FOUND, INVALID_ARGUMENT) não abrem o circuito
- [ ] Saga completa com compensações em cada falha
- [ ] Compensação executa na ordem reversa
- [ ] Testes: simular 5 falhas → OPEN, esperar timeout → HALF_OPEN → CLOSED

---

## Desafio 7.7 — Contract Testing & Documentation

**Contexto:** Com 3 APIs públicas (REST, GraphQL, gRPC), os contratos precisam
ser documentados e testados automaticamente.

### Requisitos

**Documentação por paradigma:**

| Paradigma | Ferramenta | Formato |
|-----------|-----------|---------|
| REST | OpenAPI 3.1 | YAML/JSON |
| GraphQL | Introspection + SDL | `.graphql` |
| gRPC | Proto files + buf | `.proto` + `buf.yaml` |

**OpenAPI 3.1 para REST (gerado do código):**

```yaml
openapi: "3.1.0"
info:
  title: "Marketplace API"
  version: "1.0.0"
  description: "REST API pública do Online Marketplace"
servers:
  - url: "https://api.marketplace.com/api/v1"
paths:
  /products:
    get:
      operationId: listProducts
      parameters:
        - name: cursor
          in: query
          schema: { type: string }
        - name: size
          in: query
          schema: { type: integer, default: 20, maximum: 100 }
        - name: category
          in: query
          schema: { type: string }
      responses:
        "200":
          description: Lista de produtos
          headers:
            ETag: { schema: { type: string } }
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ProductListResponse"
        "429":
          description: Rate limit exceeded
          content:
            application/problem+json:
              schema:
                $ref: "#/components/schemas/ProblemDetail"
```

**Contract tests (verifica que API real corresponde à spec):**

| Tipo | O que verifica |
|------|---------------|
| REST contract | Responses correspondem ao OpenAPI spec |
| GraphQL contract | Schema corresponde ao SDL versionado |
| gRPC contract | `buf breaking` contra versão anterior |
| Cross-paradigm | Mesma operação retorna dados consistentes via REST, GraphQL e gRPC |

### Go 1.26 — Cross-paradigm consistency test

```go
func TestCrossParadigmConsistency(t *testing.T) {
    // Criar produto via gRPC
    grpcResp, err := grpcClient.CreateProduct(ctx, &pb.CreateProductRequest{
        Name:  "Cross Test Product",
        Price: &pb.Money{Amount: 2990, CurrencyCode: "BRL"},
        Stock: 25,
    })
    if err != nil { t.Fatal(err) }
    productID := grpcResp.GetProduct().GetId()

    // Buscar via REST
    restResp := httpGet(t, fmt.Sprintf("/api/v1/products/%s", productID))
    var restProduct map[string]any
    json.NewDecoder(restResp.Body).Decode(&restProduct)

    // Buscar via GraphQL
    gqlResp := graphqlQuery(t, `query($id: ID!) { product(id: $id) { id name price { amount } } }`,
        map[string]any{"id": productID})
    var gqlProduct struct {
        Data struct {
            Product struct {
                ID    string
                Name  string
                Price struct{ Amount int64 }
            }
        }
    }
    json.NewDecoder(gqlResp.Body).Decode(&gqlProduct)

    // Verificar consistência
    if restProduct["name"] != gqlProduct.Data.Product.Name {
        t.Errorf("REST name %q != GraphQL name %q",
            restProduct["name"], gqlProduct.Data.Product.Name)
    }
    if restProduct["name"] != grpcResp.GetProduct().GetName() {
        t.Errorf("REST name %q != gRPC name %q",
            restProduct["name"], grpcResp.GetProduct().GetName())
    }
}
```

### Critérios de Aceite

- [ ] OpenAPI 3.1 spec completa para REST API
- [ ] GraphQL schema exportável via introspection
- [ ] `buf.yaml` com lint + breaking rules
- [ ] Contract test REST: response → OpenAPI validation
- [ ] Contract test gRPC: `buf breaking --against main`
- [ ] **Cross-paradigm test**: mesma entidade retorna dados iguais via REST, GraphQL e gRPC
- [ ] CI pipeline: contract tests rodam automaticamente
- [ ] Documentação versionada junto com código

---

## Desafio 7.8 — Production Readiness Checklist

**Contexto:** Verificar que toda a plataforma está pronta para produção.

### Checklist Completo

**API Design:**
- [ ] REST segue todas as convenções dos Levels 1-2
- [ ] GraphQL segue todas as convenções dos Levels 3-4
- [ ] gRPC segue todas as convenções dos Levels 5-6
- [ ] Consistência entre paradigmas (mesmos dados, mesma semântica)

**Security:**
- [ ] TLS em todas as conexões (client → gateway, gateway → services)
- [ ] mTLS entre microserviços gRPC
- [ ] JWT auth no gateway
- [ ] Rate limiting por tier (free/pro/enterprise)
- [ ] CORS configurado para storefront
- [ ] Security headers (CSP, HSTS, X-Frame-Options)
- [ ] GraphQL depth limit ≤ 5, complexity limit
- [ ] gRPC max message size 4MB

**Reliability:**
- [ ] Circuit breaker em todos os gRPC clients
- [ ] Retry policy para códigos transitórios (UNAVAILABLE)
- [ ] Deadlines em todas as chamadas gRPC
- [ ] Saga com compensações para fluxos distribuídos
- [ ] Graceful shutdown em todos os serviços (30s drain)
- [ ] Health checks: gateway agrega status de todos os backends

**Observability:**
- [ ] Trace ID propagado end-to-end (gateway → REST/GraphQL → gRPC)
- [ ] Structured logging com trace_id, method, duration, code
- [ ] Métricas: latency p50/p95/p99, error rate, throughput
- [ ] Alertas configurados (error rate > 5%, p99 > 500ms)

**Testing:**
- [ ] Unit tests com bufconn/InProcessServer
- [ ] Integration tests end-to-end
- [ ] Contract tests (OpenAPI, buf breaking)
- [ ] Cross-paradigm consistency tests
- [ ] Load tests com ghz (gRPC) e benchmarks HTTP
- [ ] Cobertura ≥ 80%

**Performance:**
- [ ] DataLoaders para N+1 no GraphQL
- [ ] Connection pooling/reuse no gRPC
- [ ] Caching (ETag REST, @cacheControl GraphQL)
- [ ] Keep-alive configurado
- [ ] Benchmark: documentar throughput e latency por paradigma

### Critérios de Aceite Finais

- [ ] Todos os 5 microserviços gRPC rodando
- [ ] API Gateway roteando para REST + GraphQL + gRPC
- [ ] Workflow completo: criar produto (REST) → comprar (GraphQL) → pagamento (gRPC) → notificação (gRPC)
- [ ] Falha em um serviço: circuit breaker + fallback + compensação
- [ ] Trace completo visível de gateway até o último microserviço
- [ ] Documentação completa: OpenAPI + SDL + Proto
- [ ] Todos os contract tests passando
- [ ] Load test: > 1000 req/s com p99 < 200ms

---

## Anti-Patterns do Capstone

| Anti-Pattern | Correto |
|-------------|---------|
| GraphQL chama DB diretamente | GraphQL resolve via gRPC clients |
| REST e GraphQL com stores separados | Ambos delegam para mesmos gRPC services |
| Sem circuit breaker | CB em todo gRPC client |
| Trace ID diferente em cada serviço | Trace ID único propagado por toda a cadeia |
| Health check apenas no gateway | Cada serviço com health check + gateway agrega |
| Documentação desatualizada | Contract tests garantem spec = implementação |
| Shutdown imediato | Graceful shutdown coordenado (gateway último) |

---

## Conclusão

Ao completar este capstone, você terá construído uma **plataforma multi-paradigm** completa:

- **REST** para interoperabilidade com parceiros
- **GraphQL** para flexibilidade do storefront
- **gRPC** para performance interna entre microserviços
- **Observabilidade** end-to-end com tracing distribuído
- **Resiliência** com circuit breakers e sagas
- **Documentação** viva com contract testing

Cada paradigma tem seu lugar. A arte está em saber quando usar cada um.
