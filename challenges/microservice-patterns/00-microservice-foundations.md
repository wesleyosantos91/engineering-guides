# Level 0 — Fundações de Microsserviços & Setup do Domínio

> **Objetivo:** Estabelecer a arquitetura base da plataforma RideFlow, criar os microsserviços iniciais
> com comunicação síncrona (HTTP/REST), entender os desafios fundamentais de sistemas distribuídos
> e preparar a infraestrutura local com Docker Compose.

**Referência:** Conceitos fundamentais (pré-requisito para [01-resilience-circuit-breaker-retry-timeout.md](01-resilience-circuit-breaker-retry-timeout.md))

---

## Contexto do Domínio

Neste nível, você criará o **esqueleto da plataforma RideFlow** — uma plataforma de mobilidade urbana com os seguintes microsserviços iniciais:

- **Ride Service** — Gerencia o ciclo de vida das corridas (solicitar, aceitar, iniciar, finalizar)
- **Driver Service** — Gerencia motoristas, veículos e disponibilidade
- **Passenger Service** — Gerencia passageiros e preferências
- **Pricing Service** — Calcula o preço da corrida (distância, tempo, surge)

Cada serviço terá seu próprio banco de dados (Database per Service pattern) e se comunicará via HTTP/REST.

---

## Desafios

### Desafio 0.1 — Estrutura do Projeto Multi-Serviço

Configure a estrutura de projeto para 4 microsserviços independentes com build e execução isolados.

**Requisitos:**
- Criar 4 projetos independentes: `ride-service`, `driver-service`, `passenger-service`, `pricing-service`
- Cada serviço roda em porta diferente (8081, 8082, 8083, 8084)
- Cada serviço tem seu próprio `Dockerfile` multi-stage
- `docker-compose.yml` sobe todos os serviços + bancos de dados
- Cada serviço tem endpoint `GET /info` retornando nome, versão e timestamp de startup

**Java 25 (Spring Boot):**
```java
@SpringBootApplication
public class RideServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(RideServiceApplication.class, args);
    }
}

@RestController
@RequestMapping("/info")
public class InfoController {
    
    @Value("${spring.application.name}")
    private String appName;
    
    private final Instant startupTime = Instant.now();
    
    @GetMapping
    public Map<String, Object> info() {
        return Map.of(
            "service", appName,
            "version", "1.0.0",
            "startedAt", startupTime.toString()
        );
    }
}
```

**Go 1.26:**
```go
package main

import (
    "encoding/json"
    "net/http"
    "time"
    
    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
)

var startupTime = time.Now()

func main() {
    r := chi.NewRouter()
    r.Use(middleware.Logger)
    r.Use(middleware.Recoverer)
    
    r.Get("/info", func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]any{
            "service":   "ride-service",
            "version":   "1.0.0",
            "startedAt": startupTime.Format(time.RFC3339),
        })
    })
    
    http.ListenAndServe(":8081", r)
}
```

**Critérios de aceite:**
- [ ] 4 serviços independentes com build separado
- [ ] Cada serviço roda em porta distinta
- [ ] `docker-compose.yml` sobe toda a stack com um comando
- [ ] Dockerfile multi-stage (build + runtime) para imagem mínima
- [ ] Endpoint `/info` funcional em todos os serviços
- [ ] Cada serviço tem seu banco PostgreSQL isolado (schema ou instância separada)

---

### Desafio 0.2 — Entidades Base & Database per Service

Modele e implemente as entidades de domínio de cada serviço com banco de dados isolado.

**Requisitos:**
- **Ride Service:** `Ride` (id, passengerId, driverId, origin, destination, status, price, createdAt, updatedAt)
- **Driver Service:** `Driver` (id, name, licensePlate, vehicleModel, category, status, latitude, longitude, rating)
- **Passenger Service:** `Passenger` (id, name, email, phone, rating, preferredPayment)
- **Pricing Service:** `PricingRule` (id, category, baseFare, perKmRate, perMinuteRate, surgeMultiplier, activeFrom, activeTo)
- Migrations automáticas via Flyway (Java) ou golang-migrate (Go)
- CRUD básico para cada entidade com validação de entrada

**Java 25 (Spring Data JPA):**
```java
@Entity
@Table(name = "rides")
public class Ride {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false)
    private UUID passengerId;

    private UUID driverId;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "latitude", column = @Column(name = "origin_lat")),
        @AttributeOverride(name = "longitude", column = @Column(name = "origin_lng"))
    })
    private GeoPoint origin;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "latitude", column = @Column(name = "dest_lat")),
        @AttributeOverride(name = "longitude", column = @Column(name = "dest_lng"))
    })
    private GeoPoint destination;

    @Enumerated(EnumType.STRING)
    private RideStatus status;

    private BigDecimal price;

    @CreationTimestamp
    private Instant createdAt;

    @UpdateTimestamp
    private Instant updatedAt;
}

public enum RideStatus {
    REQUESTED, MATCHED, DRIVER_EN_ROUTE, IN_PROGRESS, COMPLETED, CANCELLED
}
```

**Go 1.26:**
```go
type Ride struct {
    ID          uuid.UUID  `db:"id" json:"id"`
    PassengerID uuid.UUID  `db:"passenger_id" json:"passengerId"`
    DriverID    *uuid.UUID `db:"driver_id" json:"driverId,omitempty"`
    OriginLat   float64    `db:"origin_lat" json:"originLat"`
    OriginLng   float64    `db:"origin_lng" json:"originLng"`
    DestLat     float64    `db:"dest_lat" json:"destLat"`
    DestLng     float64    `db:"dest_lng" json:"destLng"`
    Status      RideStatus `db:"status" json:"status"`
    Price       *float64   `db:"price" json:"price,omitempty"`
    CreatedAt   time.Time  `db:"created_at" json:"createdAt"`
    UpdatedAt   time.Time  `db:"updated_at" json:"updatedAt"`
}

type RideStatus string

const (
    RideStatusRequested    RideStatus = "REQUESTED"
    RideStatusMatched      RideStatus = "MATCHED"
    RideStatusDriverEnRoute RideStatus = "DRIVER_EN_ROUTE"
    RideStatusInProgress   RideStatus = "IN_PROGRESS"
    RideStatusCompleted    RideStatus = "COMPLETED"
    RideStatusCancelled    RideStatus = "CANCELLED"
)
```

**Critérios de aceite:**
- [ ] Entidades mapeadas em todas as 4 bases de dados
- [ ] Migrations versionadas (Flyway ou golang-migrate)
- [ ] CRUD funcional com endpoints REST para cada entidade
- [ ] Validação de entrada (campos obrigatórios, formatos, ranges)
- [ ] Testes de integração com Testcontainers (PostgreSQL real)
- [ ] Cada serviço conecta APENAS ao seu banco — sem acesso cross-database

---

### Desafio 0.3 — Comunicação Síncrona entre Serviços (HTTP Client)

Implemente a comunicação HTTP entre serviços para o fluxo de solicitar corrida.

**Cenário:** Quando um passageiro solicita uma corrida, o Ride Service precisa:
1. Chamar Passenger Service para validar que o passageiro existe
2. Chamar Pricing Service para calcular o preço estimado
3. Chamar Driver Service para buscar motoristas disponíveis na região

**Requisitos:**
- Ride Service faz chamadas HTTP para os outros 3 serviços
- Usar HTTP client com timeout configurável (não infinito!)
- Tratar erros de comunicação (serviço indisponível, timeout, erro 5xx)
- Retornar resposta agregada com preço estimado e motoristas próximos
- URLs dos serviços configuráveis via variáveis de ambiente (não hardcoded)

**Java 25 (WebClient):**
```java
@Service
public class DriverServiceClient {
    
    private final WebClient webClient;
    
    public DriverServiceClient(
            @Value("${services.driver.url}") String driverServiceUrl,
            WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl(driverServiceUrl)
            .build();
    }
    
    public List<DriverDTO> findAvailableDrivers(double lat, double lng, double radiusKm) {
        return webClient.get()
            .uri(uriBuilder -> uriBuilder
                .path("/drivers/available")
                .queryParam("lat", lat)
                .queryParam("lng", lng)
                .queryParam("radiusKm", radiusKm)
                .build())
            .retrieve()
            .bodyToFlux(DriverDTO.class)
            .collectList()
            .block(Duration.ofSeconds(5));
    }
}
```

**Go 1.26:**
```go
type DriverServiceClient struct {
    baseURL    string
    httpClient *http.Client
}

func NewDriverServiceClient(baseURL string) *DriverServiceClient {
    return &DriverServiceClient{
        baseURL: baseURL,
        httpClient: &http.Client{
            Timeout: 5 * time.Second,
        },
    }
}

func (c *DriverServiceClient) FindAvailableDrivers(
    ctx context.Context, lat, lng, radiusKm float64,
) ([]DriverDTO, error) {
    url := fmt.Sprintf("%s/drivers/available?lat=%f&lng=%f&radiusKm=%f",
        c.baseURL, lat, lng, radiusKm)
    
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return nil, fmt.Errorf("creating request: %w", err)
    }
    
    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("calling driver service: %w", err)
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("driver service returned status %d", resp.StatusCode)
    }
    
    var drivers []DriverDTO
    if err := json.NewDecoder(resp.Body).Decode(&drivers); err != nil {
        return nil, fmt.Errorf("decoding response: %w", err)
    }
    return drivers, nil
}
```

**Critérios de aceite:**
- [ ] Ride Service chama 3 serviços durante o fluxo de solicitar corrida
- [ ] HTTP client com timeout explícito (não default infinito)
- [ ] Tratamento de erros por status code (4xx, 5xx) e erros de rede
- [ ] URLs configuráveis via environment variables / application.yml
- [ ] Logs de cada chamada inter-serviço (URL, método, status, duração)
- [ ] Testes com WireMock (Java) ou httptest (Go) simulando serviços externos

---

### Desafio 0.4 — Fluxo de Corrida End-to-End (Happy Path)

Implemente o fluxo completo de uma corrida do início ao fim (apenas caminho feliz).

**Cenário:** O passageiro João solicita uma corrida de casa ao trabalho. O sistema encontra a motorista Maria, calcula o preço e inicia a corrida. Maria completa a corrida e o sistema calcula o preço final.

**Requisitos:**
- `POST /rides` — Solicitar corrida (valida passageiro, calcula preço, busca motorista)
- `PATCH /rides/{id}/match` — Motorista aceita a corrida (Driver Service confirma)
- `PATCH /rides/{id}/start` — Motorista inicia a corrida (valida proximidade ao ponto de origem)
- `PATCH /rides/{id}/complete` — Motorista finaliza (calcula preço final baseado em distância/tempo real)
- Máquina de estados: `REQUESTED → MATCHED → DRIVER_EN_ROUTE → IN_PROGRESS → COMPLETED`
- Validação de transição de estado (não pode ir de REQUESTED para COMPLETED diretamente)

**Java 25:**
```java
@Service
public class RideStateMachine {
    
    private static final Map<RideStatus, Set<RideStatus>> VALID_TRANSITIONS = Map.of(
        RideStatus.REQUESTED, Set.of(RideStatus.MATCHED, RideStatus.CANCELLED),
        RideStatus.MATCHED, Set.of(RideStatus.DRIVER_EN_ROUTE, RideStatus.CANCELLED),
        RideStatus.DRIVER_EN_ROUTE, Set.of(RideStatus.IN_PROGRESS, RideStatus.CANCELLED),
        RideStatus.IN_PROGRESS, Set.of(RideStatus.COMPLETED),
        RideStatus.COMPLETED, Set.of(),
        RideStatus.CANCELLED, Set.of()
    );
    
    public void validateTransition(RideStatus current, RideStatus target) {
        if (!VALID_TRANSITIONS.getOrDefault(current, Set.of()).contains(target)) {
            throw new InvalidStateTransitionException(
                "Cannot transition from %s to %s".formatted(current, target));
        }
    }
}
```

**Go 1.26:**
```go
var validTransitions = map[RideStatus]map[RideStatus]bool{
    RideStatusRequested:     {RideStatusMatched: true, RideStatusCancelled: true},
    RideStatusMatched:       {RideStatusDriverEnRoute: true, RideStatusCancelled: true},
    RideStatusDriverEnRoute: {RideStatusInProgress: true, RideStatusCancelled: true},
    RideStatusInProgress:    {RideStatusCompleted: true},
    RideStatusCompleted:     {},
    RideStatusCancelled:     {},
}

func ValidateTransition(current, target RideStatus) error {
    allowed, exists := validTransitions[current]
    if !exists || !allowed[target] {
        return fmt.Errorf("invalid transition from %s to %s", current, target)
    }
    return nil
}
```

**Critérios de aceite:**
- [ ] Fluxo completo REQUESTED → MATCHED → DRIVER_EN_ROUTE → IN_PROGRESS → COMPLETED
- [ ] Máquina de estados com validação de transições (rejeitar transições inválidas)
- [ ] Cada transição atualiza `updatedAt` e persiste no banco
- [ ] Cálculo de preço final usa duração e distância reais (mock simples neste nível)
- [ ] Testes end-to-end com todos os serviços rodando (docker-compose)
- [ ] Cancelamento possível em qualquer status exceto COMPLETED e CANCELLED

---

### Desafio 0.5 — Observando os Problemas de Sistemas Distribuídos

Identifique e documente os problemas que surgem com a arquitetura atual (sem padrões de resiliência).

**Cenário:** Com os 4 serviços rodando, simule falhas para observar o comportamento:
1. Derrube o Pricing Service e tente solicitar uma corrida
2. Adicione um `sleep(10s)` no Driver Service e observe o timeout cascading
3. Faça 100 requisições simultâneas ao Ride Service e observe a degradação

**Requisitos:**
- Script/teste que simula cada cenário de falha
- Documentar no `DECISIONS.md` os problemas observados:
  - O que acontece quando um serviço está fora?
  - Como o timeout de um serviço afeta os outros?
  - Como o sistema se comporta sob carga?
- Para cada problema, identificar qual padrão do Level 1+ resolve
- Não implementar soluções ainda — apenas observar e documentar

**Java 25 (Teste de carga simples):**
```java
@Test
void shouldObserveCascadingFailure() throws Exception {
    // Simula Pricing Service lento (10s delay)
    pricingServiceMock.stubFor(get(urlPathEqualTo("/pricing/estimate"))
        .willReturn(aResponse()
            .withFixedDelay(10_000)  // 10 segundos
            .withStatus(200)
            .withBody("{\"price\": 25.0}")));
    
    // 50 requisições simultâneas
    var executor = Executors.newVirtualThreadPerTaskExecutor();
    var futures = IntStream.range(0, 50)
        .mapToObj(i -> CompletableFuture.supplyAsync(() -> requestRide(), executor))
        .toList();
    
    // Observar: threads penduradas, timeouts, erros cascata
    var results = futures.stream()
        .map(f -> f.handle((r, ex) -> ex != null ? "FAILED: " + ex.getMessage() : "OK"))
        .map(CompletableFuture::join)
        .toList();
    
    long failures = results.stream().filter(r -> r.startsWith("FAILED")).count();
    System.out.println("Failures: " + failures + "/" + results.size());
    // Documentar os resultados no DECISIONS.md
}
```

**Go 1.26:**
```go
func TestCascadingFailure(t *testing.T) {
    // Simula 50 requisições concorrentes
    var wg sync.WaitGroup
    results := make([]string, 50)
    
    for i := 0; i < 50; i++ {
        wg.Add(1)
        go func(idx int) {
            defer wg.Done()
            ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
            defer cancel()
            
            _, err := requestRide(ctx)
            if err != nil {
                results[idx] = fmt.Sprintf("FAILED: %v", err)
            } else {
                results[idx] = "OK"
            }
        }(i)
    }
    
    wg.Wait()
    
    failures := 0
    for _, r := range results {
        if strings.HasPrefix(r, "FAILED") {
            failures++
        }
    }
    t.Logf("Failures: %d/50", failures)
    // Documentar os resultados no DECISIONS.md
}
```

**Critérios de aceite:**
- [ ] Script reproduzível que simula serviço indisponível
- [ ] Script reproduzível que simula serviço lento (timeout cascading)
- [ ] Script reproduzível que simula carga alta (concorrência)
- [ ] `DECISIONS.md` documentando cada problema observado com evidências
- [ ] Mapeamento problema → padrão (ex: "timeout cascading → Circuit Breaker + Timeout")
- [ ] Compreensão clara de por que os padrões dos próximos níveis são necessários
