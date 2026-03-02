# Level 5 — gRPC: Protobuf & Comunicação

> **Objetivo:** Projetar e implementar serviços gRPC para comunicação interna do Marketplace,
> dominando Protobuf, os 4 padrões de comunicação e error handling rico.

**Referência:** [.docs/API/03-grpc-best-practices.md](../../../.docs/API/03-grpc-best-practices.md) — Seções 1–8

---

## Dependências Permitidas

| Java 25 | Go 1.26 |
|---------|---------|
| `io.grpc` (gRPC runtime) | `google.golang.org/grpc` |
| `com.google.protobuf` (protobuf runtime) | `google.golang.org/protobuf` |
| `protoc` + `protoc-gen-grpc-java` (codegen) | `protoc` + `protoc-gen-go` + `protoc-gen-go-grpc` |

> **Nota:** gRPC usa HTTP/2 como transporte. Diferente de REST/GraphQL, não usamos
> `com.sun.net.httpserver` — o runtime gRPC gerencia o servidor.

---

## Desafio 5.1 — Protobuf Schema Design

**Contexto:** Protobuf é o contrato de comunicação em gRPC. Um design ruim de `.proto`
gera problemas de compatibilidade que são difíceis de corrigir depois.

### Requisitos

Criar a estrutura de arquivos `.proto` do Marketplace:

```
proto/
├── marketplace/
│   ├── common/
│   │   └── v1/
│   │       ├── money.proto          # Tipos compartilhados
│   │       └── pagination.proto     # Paginação reutilizável
│   ├── product/
│   │   └── v1/
│   │       └── product_service.proto
│   ├── order/
│   │   └── v1/
│   │       └── order_service.proto
│   ├── inventory/
│   │   └── v1/
│   │       └── inventory_service.proto
│   ├── payment/
│   │   └── v1/
│   │       └── payment_service.proto
│   └── notification/
│       └── v1/
│           └── notification_service.proto
```

**Common types (`money.proto`):**

```protobuf
syntax = "proto3";

package marketplace.common.v1;

option java_package = "com.marketplace.common.v1";
option java_multiple_files = true;
option go_package = "github.com/marketplace/proto/common/v1;commonv1";

import "google/protobuf/timestamp.proto";

// Valor monetário em centavos para evitar imprecisão de ponto flutuante.
message Money {
  // Valor em centavos (ex: 15999 = R$ 159,99)
  int64 amount = 1;

  // Código ISO 4217 da moeda
  string currency_code = 2;
}

// Status genérico de operação
message OperationStatus {
  bool success = 1;
  string message = 2;
  string trace_id = 3;
}
```

**Pagination (`pagination.proto`):**

```protobuf
message PaginationRequest {
  int32 page_size = 1;     // max 100, default 20
  string page_token = 2;   // cursor opaco
}

message PaginationResponse {
  string next_page_token = 1;   // vazio se última página
  int32 total_count = 2;
}
```

**Product Service:**

```protobuf
syntax = "proto3";

package marketplace.product.v1;

import "marketplace/common/v1/money.proto";
import "marketplace/common/v1/pagination.proto";
import "google/protobuf/timestamp.proto";
import "google/protobuf/field_mask.proto";

// Serviço de gerenciamento de produtos do Marketplace.
service ProductService {
  // Cria um novo produto no catálogo.
  rpc CreateProduct(CreateProductRequest) returns (CreateProductResponse);

  // Busca um produto por ID.
  rpc GetProduct(GetProductRequest) returns (Product);

  // Lista produtos com filtros e paginação.
  rpc ListProducts(ListProductsRequest) returns (ListProductsResponse);

  // Atualiza campos específicos de um produto.
  rpc UpdateProduct(UpdateProductRequest) returns (Product);

  // Remove um produto do catálogo.
  rpc DeleteProduct(DeleteProductRequest) returns (DeleteProductResponse);
}

message Product {
  string id = 1;
  string name = 2;
  string description = 3;
  marketplace.common.v1.Money price = 4;
  string category_id = 5;
  string seller_id = 6;
  int32 stock = 7;
  ProductStatus status = 8;
  repeated string tags = 9;
  google.protobuf.Timestamp created_at = 10;
  google.protobuf.Timestamp updated_at = 11;

  // Reservados para campos removidos (nunca reutilizar números)
  reserved 12, 13;
  reserved "legacy_price", "old_category";
}

enum ProductStatus {
  PRODUCT_STATUS_UNSPECIFIED = 0;  // Sempre valor 0 = UNSPECIFIED
  PRODUCT_STATUS_ACTIVE = 1;
  PRODUCT_STATUS_INACTIVE = 2;
  PRODUCT_STATUS_OUT_OF_STOCK = 3;
  PRODUCT_STATUS_DISCONTINUED = 4;
}

message CreateProductRequest {
  string name = 1;
  string description = 2;
  marketplace.common.v1.Money price = 3;
  string category_id = 4;
  int32 stock = 5;
  repeated string tags = 6;
}

message CreateProductResponse {
  Product product = 1;
}

message GetProductRequest {
  string id = 1;
}

message ListProductsRequest {
  marketplace.common.v1.PaginationRequest pagination = 1;
  ProductFilter filter = 2;
  ProductSort sort = 3;
}

message ProductFilter {
  string category_id = 1;
  ProductStatus status = 2;
  marketplace.common.v1.Money min_price = 3;
  marketplace.common.v1.Money max_price = 4;
  string seller_id = 5;
  string query = 6;  // full-text search
}

message ProductSort {
  ProductSortField field = 1;
  SortDirection direction = 2;
}

enum ProductSortField {
  PRODUCT_SORT_FIELD_UNSPECIFIED = 0;
  PRODUCT_SORT_FIELD_NAME = 1;
  PRODUCT_SORT_FIELD_PRICE = 2;
  PRODUCT_SORT_FIELD_CREATED_AT = 3;
}

enum SortDirection {
  SORT_DIRECTION_UNSPECIFIED = 0;
  SORT_DIRECTION_ASC = 1;
  SORT_DIRECTION_DESC = 2;
}

message ListProductsResponse {
  repeated Product products = 1;
  marketplace.common.v1.PaginationResponse pagination = 2;
}

// Usar FieldMask para partial updates (equivalente a PATCH)
message UpdateProductRequest {
  string id = 1;
  Product product = 2;
  google.protobuf.FieldMask update_mask = 3;  // campos a atualizar
}

message DeleteProductRequest {
  string id = 1;
}

message DeleteProductResponse {
  marketplace.common.v1.OperationStatus status = 1;
}
```

**Convenções de Nomenclatura Protobuf:**

| Elemento | Convenção | Exemplo |
|----------|-----------|---------|
| Package | `lowercase.dot.separated` | `marketplace.product.v1` |
| Service | PascalCase + sufixo `Service` | `ProductService` |
| RPC | PascalCase (verbo + substantivo) | `CreateProduct`, `ListProducts` |
| Message | PascalCase | `CreateProductRequest` |
| Field | snake_case | `category_id`, `created_at` |
| Enum | PascalCase, values UPPER_SNAKE com prefixo | `PRODUCT_STATUS_ACTIVE` |
| File | snake_case | `product_service.proto` |

### Critérios de Aceite

- [ ] Estrutura de pastas organizada por domínio + versão (`v1`)
- [ ] Common types reutilizáveis (Money, PaginationRequest)
- [ ] Enums com valor 0 = UNSPECIFIED
- [ ] Enum values com prefixo do tipo (`PRODUCT_STATUS_`)
- [ ] `reserved` para campos removidos
- [ ] `FieldMask` para partial updates
- [ ] Timestamps com `google.protobuf.Timestamp`
- [ ] Código gerado compila em Java e Go
- [ ] Nomenclatura consistente (snake_case para fields, PascalCase para messages)

---

## Desafio 5.2 — Unary RPC (Request/Response)

**Contexto:** Unary RPC é o padrão mais comum — uma request, uma response.
Implemente o ProductService completo com CRUD.

### Requisitos

Implementar todos os RPCs do `ProductService`:

### Java 25

```java
public class ProductServiceImpl extends ProductServiceGrpc.ProductServiceImplBase {

    private final Map<String, Product> store = new ConcurrentHashMap<>();

    @Override
    public void createProduct(CreateProductRequest request,
                               StreamObserver<CreateProductResponse> responseObserver) {
        // Validação
        if (request.getName().isBlank()) {
            responseObserver.onError(Status.INVALID_ARGUMENT
                .withDescription("name is required")
                .asRuntimeException());
            return;
        }

        var product = Product.newBuilder()
            .setId(UUID.randomUUID().toString())
            .setName(request.getName())
            .setDescription(request.getDescription())
            .setPrice(request.getPrice())
            .setCategoryId(request.getCategoryId())
            .setStock(request.getStock())
            .addAllTags(request.getTagsList())
            .setStatus(ProductStatus.PRODUCT_STATUS_ACTIVE)
            .setCreatedAt(Timestamps.now())
            .setUpdatedAt(Timestamps.now())
            .build();

        store.put(product.getId(), product);

        responseObserver.onNext(CreateProductResponse.newBuilder()
            .setProduct(product)
            .build());
        responseObserver.onCompleted();
    }

    @Override
    public void getProduct(GetProductRequest request,
                            StreamObserver<Product> responseObserver) {
        var product = store.get(request.getId());
        if (product == null) {
            responseObserver.onError(Status.NOT_FOUND
                .withDescription("product not found: " + request.getId())
                .asRuntimeException());
            return;
        }
        responseObserver.onNext(product);
        responseObserver.onCompleted();
    }

    @Override
    public void updateProduct(UpdateProductRequest request,
                               StreamObserver<Product> responseObserver) {
        var existing = store.get(request.getId());
        if (existing == null) {
            responseObserver.onError(Status.NOT_FOUND
                .withDescription("product not found")
                .asRuntimeException());
            return;
        }

        // Aplicar FieldMask para partial update
        var builder = existing.toBuilder();
        var mask = request.getUpdateMask();
        for (String path : mask.getPathsList()) {
            switch (path) {
                case "name" -> builder.setName(request.getProduct().getName());
                case "description" -> builder.setDescription(request.getProduct().getDescription());
                case "price" -> builder.setPrice(request.getProduct().getPrice());
                case "stock" -> builder.setStock(request.getProduct().getStock());
                default -> {
                    responseObserver.onError(Status.INVALID_ARGUMENT
                        .withDescription("unknown field in update_mask: " + path)
                        .asRuntimeException());
                    return;
                }
            }
        }

        var updated = builder.setUpdatedAt(Timestamps.now()).build();
        store.put(updated.getId(), updated);
        responseObserver.onNext(updated);
        responseObserver.onCompleted();
    }
}
```

### Go 1.26

```go
type ProductServer struct {
    pb.UnimplementedProductServiceServer
    mu    sync.RWMutex
    store map[string]*pb.Product
}

func (s *ProductServer) CreateProduct(ctx context.Context, req *pb.CreateProductRequest) (*pb.CreateProductResponse, error) {
    if req.GetName() == "" {
        return nil, status.Errorf(codes.InvalidArgument, "name is required")
    }

    product := &pb.Product{
        Id:          uuid.New().String(),
        Name:        req.GetName(),
        Description: req.GetDescription(),
        Price:       req.GetPrice(),
        CategoryId:  req.GetCategoryId(),
        Stock:       req.GetStock(),
        Tags:        req.GetTags(),
        Status:      pb.ProductStatus_PRODUCT_STATUS_ACTIVE,
        CreatedAt:   timestamppb.Now(),
        UpdatedAt:   timestamppb.Now(),
    }

    s.mu.Lock()
    s.store[product.Id] = product
    s.mu.Unlock()

    return &pb.CreateProductResponse{Product: product}, nil
}

func (s *ProductServer) GetProduct(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error) {
    s.mu.RLock()
    product, ok := s.store[req.GetId()]
    s.mu.RUnlock()

    if !ok {
        return nil, status.Errorf(codes.NotFound, "product not found: %s", req.GetId())
    }
    return product, nil
}

func (s *ProductServer) UpdateProduct(ctx context.Context, req *pb.UpdateProductRequest) (*pb.Product, error) {
    s.mu.Lock()
    defer s.mu.Unlock()

    existing, ok := s.store[req.GetId()]
    if !ok {
        return nil, status.Errorf(codes.NotFound, "product not found")
    }

    // Apply FieldMask
    for _, path := range req.GetUpdateMask().GetPaths() {
        switch path {
        case "name":
            existing.Name = req.GetProduct().GetName()
        case "description":
            existing.Description = req.GetProduct().GetDescription()
        case "stock":
            existing.Stock = req.GetProduct().GetStock()
        default:
            return nil, status.Errorf(codes.InvalidArgument, "unknown field: %s", path)
        }
    }
    existing.UpdatedAt = timestamppb.Now()
    s.store[existing.Id] = existing
    return existing, nil
}
```

### Critérios de Aceite

- [ ] CRUD completo via gRPC (Create, Get, List, Update, Delete)
- [ ] `FieldMask` para partial updates funcional
- [ ] Validação retorna `INVALID_ARGUMENT` com descrição clara
- [ ] Recurso inexistente retorna `NOT_FOUND`
- [ ] Paginação com `page_token` + `page_size`
- [ ] Filtros aplicados em `ListProducts`
- [ ] Store thread-safe (`ConcurrentHashMap`/`sync.RWMutex`)
- [ ] Testes para cada RPC (sucesso + erro)

---

## Desafio 5.3 — Server Streaming RPC

**Contexto:** O serviço de inventário precisa enviar atualizações de estoque
em tempo real conforme vendas acontecem.

### Requisitos

**Inventory Service com streaming:**

```protobuf
service InventoryService {
  // Stream de atualizações de estoque em tempo real
  rpc WatchInventory(WatchInventoryRequest) returns (stream InventoryUpdate);

  // Stream de produtos com estoque baixo
  rpc GetLowStockProducts(LowStockRequest) returns (stream Product);

  // Exportar inventário completo (dataset grande)
  rpc ExportInventory(ExportInventoryRequest) returns (stream InventoryChunk);
}

message WatchInventoryRequest {
  repeated string product_ids = 1;  // produtos a monitorar (vazio = todos)
}

message InventoryUpdate {
  string product_id = 1;
  int32 previous_stock = 2;
  int32 current_stock = 3;
  string reason = 4;          // "sale", "restock", "adjustment"
  google.protobuf.Timestamp timestamp = 5;
}

message LowStockRequest {
  int32 threshold = 1;   // produtos com stock <= threshold
}

message ExportInventoryRequest {
  string category_id = 1;  // filtrar por categoria (vazio = todos)
}

message InventoryChunk {
  repeated Product products = 1;
  int32 chunk_number = 2;
  int32 total_chunks = 3;
}
```

### Java 25

```java
@Override
public void watchInventory(WatchInventoryRequest request,
                            StreamObserver<InventoryUpdate> responseObserver) {
    var productIds = new HashSet<>(request.getProductIdsList());

    // Registrar listener para inventory changes
    Consumer<InventoryUpdate> listener = update -> {
        if (productIds.isEmpty() || productIds.contains(update.getProductId())) {
            responseObserver.onNext(update);
        }
    };

    inventoryEventBus.subscribe("inventory.updated", listener);

    // Manter stream aberto até o client cancelar
    Context.current().addListener(context -> {
        inventoryEventBus.unsubscribe("inventory.updated", listener);
        responseObserver.onCompleted();
    }, Runnable::run);
}

@Override
public void exportInventory(ExportInventoryRequest request,
                             StreamObserver<InventoryChunk> responseObserver) {
    var products = store.values().stream()
        .filter(p -> request.getCategoryId().isEmpty() ||
                     p.getCategoryId().equals(request.getCategoryId()))
        .toList();

    int chunkSize = 50;
    int totalChunks = (int) Math.ceil((double) products.size() / chunkSize);

    for (int i = 0; i < totalChunks; i++) {
        var chunk = products.subList(i * chunkSize,
                                      Math.min((i + 1) * chunkSize, products.size()));
        responseObserver.onNext(InventoryChunk.newBuilder()
            .addAllProducts(chunk)
            .setChunkNumber(i + 1)
            .setTotalChunks(totalChunks)
            .build());
    }
    responseObserver.onCompleted();
}
```

### Go 1.26

```go
func (s *InventoryServer) WatchInventory(req *pb.WatchInventoryRequest,
    stream pb.InventoryService_WatchInventoryServer) error {

    productIDs := make(map[string]bool)
    for _, id := range req.GetProductIds() {
        productIDs[id] = true
    }

    updates := s.eventBus.Subscribe("inventory.updated")
    defer s.eventBus.Unsubscribe("inventory.updated", updates)

    for {
        select {
        case update := <-updates:
            if len(productIDs) == 0 || productIDs[update.ProductId] {
                if err := stream.Send(update); err != nil {
                    return err
                }
            }
        case <-stream.Context().Done():
            return stream.Context().Err()
        }
    }
}

func (s *InventoryServer) ExportInventory(req *pb.ExportInventoryRequest,
    stream pb.InventoryService_ExportInventoryServer) error {

    products := s.getFilteredProducts(req.GetCategoryId())
    chunkSize := 50
    totalChunks := (len(products) + chunkSize - 1) / chunkSize

    for i := 0; i < totalChunks; i++ {
        end := min((i+1)*chunkSize, len(products))
        chunk := &pb.InventoryChunk{
            Products:    products[i*chunkSize : end],
            ChunkNumber: int32(i + 1),
            TotalChunks: int32(totalChunks),
        }
        if err := stream.Send(chunk); err != nil {
            return err
        }
    }
    return nil
}
```

### Critérios de Aceite

- [ ] `WatchInventory` envia updates em tempo real via server stream
- [ ] Filtro por `product_ids` funcional (vazio = todos)
- [ ] Stream encerrado quando client cancela (context done)
- [ ] `ExportInventory` envia dados em chunks de 50 produtos
- [ ] Chunks numerados com total informado
- [ ] `GetLowStockProducts` filtra por threshold
- [ ] Cleanup de recursos quando stream encerra (unsubscribe listener)
- [ ] Testes: receive 3 updates, cancel stream, verify cleanup

---

## Desafio 5.4 — Client Streaming & Bidirectional

**Contexto:** Client streaming permite que o client envie múltiplas mensagens.
Bidirectional combina ambos os direcionais.

### Requisitos

**Client Streaming — Bulk Import:**

```protobuf
service ProductService {
  // Client envia múltiplos produtos, server responde com resumo
  rpc BulkCreateProducts(stream CreateProductRequest) returns (BulkCreateProductsResponse);
}

message BulkCreateProductsResponse {
  int32 total_received = 1;
  int32 total_created = 2;
  int32 total_failed = 3;
  repeated BulkError errors = 4;
}

message BulkError {
  int32 index = 1;
  string message = 2;
}
```

**Bidirectional Streaming — Real-time Chat (Suporte ao Cliente):**

```protobuf
service ChatService {
  // Chat bidirecional entre buyer e support
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message ChatMessage {
  string id = 1;
  string sender_id = 2;
  string content = 3;
  ChatMessageType type = 4;
  google.protobuf.Timestamp timestamp = 5;
}

enum ChatMessageType {
  CHAT_MESSAGE_TYPE_UNSPECIFIED = 0;
  CHAT_MESSAGE_TYPE_TEXT = 1;
  CHAT_MESSAGE_TYPE_SYSTEM = 2;     // "User joined", "User left"
  CHAT_MESSAGE_TYPE_TYPING = 3;      // Typing indicator
}
```

### Go 1.26 — Bidirectional Chat

```go
func (s *ChatServer) Chat(stream pb.ChatService_ChatServer) error {
    // Registrar participante
    sessionID := uuid.New().String()
    s.sessions.Store(sessionID, stream)
    defer s.sessions.Delete(sessionID)

    // Enviar mensagem de sistema
    stream.Send(&pb.ChatMessage{
        Id:        uuid.New().String(),
        SenderId:  "system",
        Content:   "Welcome! An agent will be with you shortly.",
        Type:      pb.ChatMessageType_CHAT_MESSAGE_TYPE_SYSTEM,
        Timestamp: timestamppb.Now(),
    })

    // Loop: receber do client, broadcast para outros participants
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }

        // Broadcast para outras sessões
        s.sessions.Range(func(key, value any) bool {
            if key.(string) != sessionID {
                otherStream := value.(pb.ChatService_ChatServer)
                otherStream.Send(msg)
            }
            return true
        })
    }
}
```

### Java 25 — Client Streaming Bulk Import

```java
@Override
public StreamObserver<CreateProductRequest> bulkCreateProducts(
        StreamObserver<BulkCreateProductsResponse> responseObserver) {

    var results = new CopyOnWriteArrayList<BulkError>();
    var counter = new AtomicInteger(0);
    var created = new AtomicInteger(0);

    return new StreamObserver<>() {
        @Override
        public void onNext(CreateProductRequest request) {
            int index = counter.incrementAndGet();
            try {
                validateAndCreate(request);
                created.incrementAndGet();
            } catch (Exception e) {
                results.add(BulkError.newBuilder()
                    .setIndex(index)
                    .setMessage(e.getMessage())
                    .build());
            }
        }

        @Override
        public void onError(Throwable t) {
            responseObserver.onError(Status.INTERNAL
                .withDescription("bulk import failed: " + t.getMessage())
                .asRuntimeException());
        }

        @Override
        public void onCompleted() {
            responseObserver.onNext(BulkCreateProductsResponse.newBuilder()
                .setTotalReceived(counter.get())
                .setTotalCreated(created.get())
                .setTotalFailed(results.size())
                .addAllErrors(results)
                .build());
            responseObserver.onCompleted();
        }
    };
}
```

### Critérios de Aceite

- [ ] **Client Streaming:** BulkCreate recebe N requests, retorna 1 summary response
- [ ] Bulk import conta received, created, failed com detalhes de erros
- [ ] Erros individuais não abortam o batch inteiro
- [ ] **Bidirectional:** Chat recebe e envia mensagens simultaneamente
- [ ] System messages ao conectar/desconectar
- [ ] Broadcast para todos os participantes (exceto sender)
- [ ] Cleanup de sessão quando stream encerra
- [ ] Testes: client stream com mix de valid/invalid, bidi chat com 2 clients

---

## Desafio 5.5 — Error Handling (Status Codes & Rich Errors)

**Contexto:** gRPC tem 16 status codes e suporta Rich Error Model para detalhes.
Mapeie corretamente erros de negócio para status codes gRPC.

### Requisitos

**Mapeamento de status codes:**

| Cenário | gRPC Code | HTTP Equiv. |
|---------|:---------:|:-----------:|
| Sucesso | `OK` (0) | 200 |
| Validação inválida | `INVALID_ARGUMENT` (3) | 400 |
| Não autenticado | `UNAUTHENTICATED` (16) | 401 |
| Sem permissão | `PERMISSION_DENIED` (7) | 403 |
| Recurso não encontrado | `NOT_FOUND` (5) | 404 |
| Já existe (conflito) | `ALREADY_EXISTS` (6) | 409 |
| Precondição falhou | `FAILED_PRECONDITION` (9) | 412 |
| Sem estoque | `RESOURCE_EXHAUSTED` (8) | 429 |
| Cancelado pelo client | `CANCELLED` (1) | 499 |
| Deadline excedido | `DEADLINE_EXCEEDED` (4) | 504 |
| Erro interno | `INTERNAL` (13) | 500 |
| Não implementado | `UNIMPLEMENTED` (12) | 501 |
| Serviço indisponível | `UNAVAILABLE` (14) | 503 |

**Rich Error Model (google.rpc.Status):**

```protobuf
import "google/rpc/status.proto";
import "google/rpc/error_details.proto";

// Rich error com múltiplos detalhes
// Status {
//   code: INVALID_ARGUMENT
//   message: "Validation failed"
//   details: [
//     BadRequest {
//       field_violations: [
//         { field: "name", description: "must not be empty" },
//         { field: "price.amount", description: "must be >= 0" }
//       ]
//     },
//     RequestInfo {
//       request_id: "req-550e8400"
//       serving_data: "product-service-v1"
//     }
//   ]
// }
```

### Java 25

```java
// Rich Error com details
var badRequest = BadRequest.newBuilder()
    .addFieldViolations(BadRequest.FieldViolation.newBuilder()
        .setField("name")
        .setDescription("must not be empty"))
    .addFieldViolations(BadRequest.FieldViolation.newBuilder()
        .setField("price.amount")
        .setDescription("must be >= 0"))
    .build();

var requestInfo = RequestInfo.newBuilder()
    .setRequestId(requestId)
    .setServingData("product-service-v1")
    .build();

var status = com.google.rpc.Status.newBuilder()
    .setCode(Code.INVALID_ARGUMENT.getNumber())
    .setMessage("Validation failed")
    .addDetails(Any.pack(badRequest))
    .addDetails(Any.pack(requestInfo))
    .build();

responseObserver.onError(StatusProto.toStatusRuntimeException(status));
```

### Go 1.26

```go
st := status.New(codes.InvalidArgument, "validation failed")
st, _ = st.WithDetails(
    &errdetails.BadRequest{
        FieldViolations: []*errdetails.BadRequest_FieldViolation{
            {Field: "name", Description: "must not be empty"},
            {Field: "price.amount", Description: "must be >= 0"},
        },
    },
    &errdetails.RequestInfo{
        RequestId:   requestID,
        ServingData: "product-service-v1",
    },
)
return nil, st.Err()

// Client: extrair detalhes
st := status.Convert(err)
for _, detail := range st.Details() {
    switch d := detail.(type) {
    case *errdetails.BadRequest:
        for _, fv := range d.GetFieldViolations() {
            fmt.Printf("Field: %s, Error: %s\n", fv.GetField(), fv.GetDescription())
        }
    case *errdetails.RequestInfo:
        fmt.Printf("Request ID: %s\n", d.GetRequestId())
    }
}
```

### Critérios de Aceite

- [ ] Cada cenário de erro usa o gRPC status code correto (tabela acima)
- [ ] Rich Error Model com `BadRequest.FieldViolation` para validação
- [ ] `RequestInfo` com request_id em todos os erros
- [ ] Client extrai detalhes do Rich Error corretamente
- [ ] `INTERNAL` nunca expõe detalhes do servidor (stack trace, SQL)
- [ ] Erros são consistentes (mesmo cenário → mesmo status code)
- [ ] Testes para cada status code com extração de details

---

## Desafio 5.6 — Deadlines, Timeouts & Cancellation

**Contexto:** Sem deadlines, uma chamada gRPC pode ficar pendurada para sempre.
Implemente propagação de deadlines e cancellation corretos.

### Requisitos

**Deadlines por tipo de operação:**

| Operação | Deadline | Razão |
|----------|:--------:|-------|
| GetProduct (unary leitura) | 500ms | Rápido, dados em memória |
| CreateProduct (unary escrita) | 2s | Pode envolver validação |
| ListProducts (leitura com filtro) | 3s | Pode filtrar muitos dados |
| ExportInventory (server stream) | 30s | Dataset grande |
| BulkCreateProducts (client stream) | 60s | Muitos itens |
| Chat (bidi stream) | Sem deadline | Sessão longa |

**Propagação de deadline entre serviços:**

```
Client (deadline: 3s)
  → ProductService (remaining: 2.8s)
    → InventoryService (remaining: 2.5s)
      → Se expirar: DEADLINE_EXCEEDED em toda a cadeia
```

### Java 25

```java
// Client: definir deadline
var stub = ProductServiceGrpc.newBlockingStub(channel)
    .withDeadlineAfter(500, TimeUnit.MILLISECONDS);

try {
    var product = stub.getProduct(GetProductRequest.newBuilder()
        .setId("product-123").build());
} catch (StatusRuntimeException e) {
    if (e.getStatus().getCode() == Status.Code.DEADLINE_EXCEEDED) {
        System.err.println("Request timed out after 500ms");
    }
}

// Server: verificar deadline restante
@Override
public void getProduct(GetProductRequest request,
                        StreamObserver<Product> responseObserver) {
    var deadline = Context.current().getDeadline();
    if (deadline != null && deadline.isExpired()) {
        responseObserver.onError(Status.DEADLINE_EXCEEDED
            .withDescription("deadline already expired")
            .asRuntimeException());
        return;
    }

    // Propagar deadline para downstream call
    var remainingMs = deadline != null ? deadline.timeRemaining(TimeUnit.MILLISECONDS) : 500;
    var inventoryStub = InventoryServiceGrpc.newBlockingStub(inventoryChannel)
        .withDeadlineAfter(Math.min(remainingMs - 50, 200), TimeUnit.MILLISECONDS);
}
```

### Go 1.26

```go
// Client: definir deadline
ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
defer cancel()

product, err := client.GetProduct(ctx, &pb.GetProductRequest{Id: "product-123"})
if err != nil {
    if st, ok := status.FromError(err); ok && st.Code() == codes.DeadlineExceeded {
        slog.Warn("request timed out", "deadline", "500ms")
    }
}

// Server: verificar deadline e propagar
func (s *ProductServer) GetProduct(ctx context.Context, req *pb.GetProductRequest) (*pb.Product, error) {
    deadline, ok := ctx.Deadline()
    if ok && time.Until(deadline) < 0 {
        return nil, status.Errorf(codes.DeadlineExceeded, "deadline expired")
    }

    // Propagar para downstream com margem de segurança
    remaining := time.Until(deadline)
    childCtx, cancel := context.WithTimeout(ctx, remaining-50*time.Millisecond)
    defer cancel()

    inventory, err := s.inventoryClient.GetStock(childCtx, &pb.GetStockRequest{ProductId: req.GetId()})
    // ...
}

// Cancellation: client cancela, server detecta
func (s *InventoryServer) ExportInventory(req *pb.ExportInventoryRequest,
    stream pb.InventoryService_ExportInventoryServer) error {
    for _, product := range products {
        select {
        case <-stream.Context().Done():
            slog.Info("client cancelled export", "reason", stream.Context().Err())
            return status.FromContextError(stream.Context().Err()).Err()
        default:
            stream.Send(toChunk(product))
        }
    }
    return nil
}
```

### Critérios de Aceite

- [ ] Deadline configurado por tipo de operação (tabela acima)
- [ ] `DEADLINE_EXCEEDED` retornado quando tempo expira
- [ ] Deadline propagado para chamadas downstream (com margem)
- [ ] Server verifica deadline antes de iniciar trabalho caro
- [ ] Client cancellation detectado via `Context.Done()` (Go) / `Context.current()` (Java)
- [ ] Streaming longo respeita cancellation (não continua enviando)
- [ ] Testes: deadline apertado (100ms) → timeout, deadline largo (5s) → sucesso

---

## Desafio 5.7 — Interceptors (Middleware Chain)

**Contexto:** Interceptors são o equivalente gRPC de middleware HTTP.
Implemente uma cadeia de interceptors para cross-cutting concerns.

### Requisitos

**Cadeia de interceptors (ordem importa):**

```
Request → [Recovery] → [Tracing] → [Logging] → [Metrics] → [Auth] → [RateLimit] → [Validation] → Handler
```

1. **Recovery** — Captura panics/exceptions, retorna `INTERNAL` (nunca crash o server)
2. **Tracing** — Gera/propaga trace_id via metadata (headers gRPC)
3. **Logging** — Registra method, duration, status code, trace_id
4. **Metrics** — Counter de RPCs, histogram de duração
5. **Auth** — Valida token JWT via metadata `authorization`
6. **RateLimit** — Limita RPCs por client (metadata `x-client-id`)
7. **Validation** — Valida request message (campos obrigatórios, ranges)

### Java 25

```java
// Unary interceptor
public class LoggingInterceptor implements ServerInterceptor {
    @Override
    public <R, S> ServerCall.Listener<R> interceptCall(
            ServerCall<R, S> call,
            Metadata headers,
            ServerCallHandler<R, S> next) {

        var method = call.getMethodDescriptor().getFullMethodName();
        var start = Instant.now();
        var traceId = headers.get(TRACE_ID_KEY);

        return new ForwardingServerCallListener.SimpleForwardingServerCallListener<>(
                next.startCall(new ForwardingServerCall.SimpleForwardingServerCall<>(call) {
                    @Override
                    public void close(Status status, Metadata trailers) {
                        var duration = Duration.between(start, Instant.now());
                        System.out.printf("[%s] %s → %s (%dms)%n",
                            traceId, method, status.getCode(), duration.toMillis());
                        super.close(status, trailers);
                    }
                }, headers)) {};
    }
}

// Registrar interceptors na ordem
Server server = ServerBuilder.forPort(9090)
    .intercept(new RecoveryInterceptor())    // primeiro (mais externo)
    .intercept(new TracingInterceptor())
    .intercept(new LoggingInterceptor())
    .intercept(new MetricsInterceptor())
    .intercept(new AuthInterceptor())
    .intercept(new RateLimitInterceptor())
    .intercept(new ValidationInterceptor())  // último (mais próximo do handler)
    .addService(new ProductServiceImpl())
    .build();
```

### Go 1.26

```go
// Unary interceptor
func loggingInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler) (any, error) {

    start := time.Now()
    traceID := extractTraceID(ctx)

    resp, err := handler(ctx, req)

    code := status.Code(err)
    slog.Info("grpc_request",
        "method", info.FullMethod,
        "code", code.String(),
        "duration_ms", time.Since(start).Milliseconds(),
        "trace_id", traceID,
    )
    return resp, err
}

// Stream interceptor
func loggingStreamInterceptor(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo,
    handler grpc.StreamHandler) error {

    start := time.Now()
    err := handler(srv, ss)
    slog.Info("grpc_stream",
        "method", info.FullMethod,
        "code", status.Code(err).String(),
        "duration_ms", time.Since(start).Milliseconds(),
    )
    return err
}

// Registrar (ordem: executam de baixo para cima, último registrado = primeiro a executar)
server := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        recoveryInterceptor,
        tracingInterceptor,
        loggingInterceptor,
        metricsInterceptor,
        authInterceptor,
        rateLimitInterceptor,
        validationInterceptor,
    ),
    grpc.ChainStreamInterceptor(
        recoveryStreamInterceptor,
        tracingStreamInterceptor,
        loggingStreamInterceptor,
    ),
)
```

### Critérios de Aceite

- [ ] 7 interceptors implementados (recovery, tracing, logging, metrics, auth, rate limit, validation)
- [ ] **Unary** e **Stream** interceptors para cada concern
- [ ] Recovery nunca deixa o server crashar (captura todos os panics)
- [ ] Tracing propaga `trace_id` via gRPC metadata
- [ ] Auth valida JWT da metadata `authorization`
- [ ] RPCs sem auth → `UNAUTHENTICATED`
- [ ] Rate limit por client retorna `RESOURCE_EXHAUSTED`
- [ ] Ordem: recovery primeiro (mais externo), validation último (mais interno)
- [ ] Testes para cada interceptor isoladamente

---

## Desafio 5.8 — Server Bootstrap & Client Implementation

**Contexto:** Integre tudo em um servidor gRPC funcional com client para teste.

### Requisitos

**Server:**
- Inicializar com todos os serviços: ProductService, InventoryService, ChatService
- Aplicar cadeia de interceptors
- Registrar serviço de health check (`grpc.health.v1.Health`)
- Reflection habilitado para tooling (`grpcurl`)
- Graceful shutdown em SIGTERM/SIGINT

**Client:**
- Conectar ao server
- Executar operações de cada tipo:
  - Unary: CreateProduct → GetProduct → UpdateProduct → DeleteProduct
  - Server Streaming: WatchInventory (receber 3 updates)
  - Client Streaming: BulkCreateProducts (enviar 5 produtos)
  - Bidirectional: Chat (enviar e receber mensagens)
- Retry policy para erros transitórios (`UNAVAILABLE`)

### Go 1.26 — Client com retry

```go
func main() {
    // Conectar com retry policy
    retryPolicy := `{
        "methodConfig": [{
            "name": [{"service": "marketplace.product.v1.ProductService"}],
            "retryPolicy": {
                "maxAttempts": 3,
                "initialBackoff": "0.1s",
                "maxBackoff": "1s",
                "backoffMultiplier": 2,
                "retryableStatusCodes": ["UNAVAILABLE", "DEADLINE_EXCEEDED"]
            }
        }]
    }`

    conn, err := grpc.NewClient("localhost:9090",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithDefaultServiceConfig(retryPolicy),
    )
    if err != nil { log.Fatal(err) }
    defer conn.Close()

    client := pb.NewProductServiceClient(conn)

    // Unary
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    resp, err := client.CreateProduct(ctx, &pb.CreateProductRequest{
        Name:       "Gaming Mouse",
        Price:      &pb.Money{Amount: 9999, CurrencyCode: "BRL"},
        CategoryId: "electronics",
        Stock:      50,
    })
    if err != nil {
        st := status.Convert(err)
        log.Printf("Error: %s (code: %s)", st.Message(), st.Code())
        return
    }
    log.Printf("Created: %s", resp.GetProduct().GetId())
}
```

### Java 25 — Graceful Shutdown

```java
void main() throws Exception {
    var server = ServerBuilder.forPort(9090)
        .intercept(new RecoveryInterceptor())
        .intercept(new LoggingInterceptor())
        .addService(new ProductServiceImpl())
        .addService(ProtoReflectionService.newInstance())
        .addService(new HealthServiceImpl())
        .build()
        .start();

    System.out.println("gRPC server started on :9090");

    // Graceful shutdown
    Runtime.getRuntime().addShutdownHook(Thread.ofVirtual().unstarted(() -> {
        System.out.println("Shutting down gRPC server...");
        server.shutdown();
        try {
            if (!server.awaitTermination(30, TimeUnit.SECONDS)) {
                server.shutdownNow();
            }
        } catch (InterruptedException e) {
            server.shutdownNow();
        }
        System.out.println("Server stopped.");
    }));

    server.awaitTermination();
}
```

### Critérios de Aceite

- [ ] Server inicia com todos os serviços registrados
- [ ] Health check (`grpc.health.v1.Health`) funcional
- [ ] Reflection habilitada (`grpcurl list` funciona)
- [ ] Graceful shutdown: drain period de 30s
- [ ] Client executa operações de cada tipo (unary, server stream, client stream, bidi)
- [ ] Retry policy para `UNAVAILABLE` e `DEADLINE_EXCEEDED`
- [ ] Client extrai tipos de erro (Rich Error Model)
- [ ] Testes end-to-end com server + client integrados

---

## Anti-Patterns a Evitar

| Anti-Pattern | Correto |
|-------------|---------|
| `UNKNOWN` para tudo | Status code específico por cenário |
| Protobuf com JSON field names | `snake_case` nativo do Protobuf |
| Enum sem `UNSPECIFIED = 0` | Sempre começar com `*_UNSPECIFIED = 0` |
| Reutilizar field numbers removidos | `reserved` para campos deletados |
| Sem deadline em chamadas | Sempre definir deadline por operação |
| Panic sem recovery | Recovery interceptor como primeiro da cadeia |
| `server.shutdownNow()` direto | Graceful shutdown com drain period |

---

## Próximo Nível

→ [Level 6 — gRPC Advanced: Produção & Operação](06-grpc-advanced.md)
