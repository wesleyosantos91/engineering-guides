# Level 0 — Foundations & Setup

> **Objetivo:** Configurar o ambiente de desenvolvimento, entender a estrutura dos desafios
> e dominar as ferramentas necessárias: DrawIO, templates de ADR, Go e Java.

**Referência:**
- [System Design — 50 Conceitos](../../.docs/SYSTEM-DESIGN/README.md)
- [ADR Foundations](../../.docs/adrs/01-adr-foundations.md)
- [ADR Templates](../../.docs/adrs/04-adr-templates-examples.md)

---

## Objetivo de Aprendizado

- Configurar ambiente completo para Go e Java
- Instalar e dominar DrawIO (extensão VS Code ou desktop)
- Entender o formato MADR para Architecture Decision Records
- Criar a estrutura base do projeto para todos os desafios
- Fazer o primeiro ADR e primeiro diagrama como exercício

---

## Parte 1 — Setup do Ambiente

### 1.1 — Tooling

| Ferramenta | Versão | Propósito |
|-----------|--------|----------|
| **Go** | 1.24+ | Implementação dos desafios |
| **Java (JDK)** | 25+ | Implementação dos desafios |
| **Maven** | 3.9+ | Build tool Java |
| **Docker** | 24+ | Containers para infra (Redis, Kafka, Postgres) |
| **Docker Compose** | 2.x | Orquestração local |
| **DrawIO** | Latest | Diagramas de arquitetura |
| **VS Code** | Latest | IDE principal |
| **protoc** | 3.x | Compilador Protocol Buffers (para gRPC) |
| **Make** | 4.x | Automação de build |

### 1.2 — Extensões VS Code Recomendadas

```
- Draw.io Integration (hediet.vscode-drawio)
- Go (golang.go)
- Extension Pack for Java (vscjava.vscode-java-pack)
- Spring Boot Extension Pack (vmware.vscode-boot-dev-pack)
- Docker (ms-azuretools.vscode-docker)
- REST Client (humao.rest-client)
- Thunder Client (rangav.vscode-thunder-client)
```

---

## Parte 2 — Estrutura do Projeto

### 2.1 — Crie a estrutura base

```
system-design-challenges/
├── README.md
├── docker-compose.yml                 ← Infra compartilhada
├── Makefile                           ← Comandos globais
│
├── level-01-load-balancing/
│   ├── README.md
│   ├── docs/
│   │   ├── adrs/
│   │   │   └── .gitkeep
│   │   └── diagrams/
│   │       └── .gitkeep
│   ├── go/
│   │   ├── go.mod
│   │   ├── cmd/
│   │   │   └── main.go
│   │   ├── internal/
│   │   └── Makefile
│   └── java/
│       ├── pom.xml
│       ├── src/
│       │   ├── main/java/
│       │   └── test/java/
│       └── mvnw
│
├── level-02-caching/
│   └── ... (mesma estrutura)
│
└── ... (um diretório por level)
```

**Critérios de aceite:**
- [ ] Estrutura de diretórios criada para todos os 23 levels
- [ ] `docker-compose.yml` com Redis, PostgreSQL, Kafka, Zookeeper
- [ ] `Makefile` raiz com targets: `setup`, `verify`, `clean`
- [ ] Cada `go/` com `go.mod` inicializado
- [ ] Cada `java/` com `pom.xml` configurado (JDK 25, JUnit 5, Spring Boot)

---

## Parte 3 — Primeiro ADR (Exercício)

### 3.1 — Escreva seu primeiro ADR

Crie um ADR documentando a escolha de tecnologias para os desafios:

**Arquivo:** `docs/adrs/ADR-001-technology-stack-selection.md`

**Requisitos do ADR:**
- **Context:** Preciso escolher as linguagens e frameworks para implementar os desafios de System Design
- **Decision Drivers:** Performance, ecossistema, experiência do time, adequação para sistemas distribuídos
- **Options:** Go vs Rust vs Node.js / Java (Spring Boot) vs Kotlin (Ktor) vs C# (.NET)
- **Decision:** Go + Java (Spring Boot) — justifique com dados concretos
- **Consequences:** Liste 3 positivas e 2 negativas

**Critérios de aceite:**
- [ ] ADR segue o template MADR
- [ ] Mínimo 3 opções consideradas por linguagem
- [ ] Cada opção tem ≥ 2 prós e ≥ 2 contras
- [ ] Decision Outcome com justificativa clara
- [ ] Consequences documentadas (boas e ruins)

---

## Parte 4 — Primeiro Diagrama DrawIO (Exercício)

### 4.1 — Crie seu primeiro diagrama

Crie um diagrama DrawIO mostrando a arquitetura geral de um sistema web típico:

**Arquivo:** `docs/diagrams/00-typical-web-architecture.drawio`

**O diagrama deve conter:**

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Client  │────▶│   CDN    │────▶│   Load   │────▶│  App     │
│ (Browser)│     │          │     │ Balancer │     │ Servers  │
└──────────┘     └──────────┘     └──────────┘     └────┬─────┘
                                                        │
                                                   ┌────▼─────┐
                                                   │  Cache   │
                                                   │ (Redis)  │
                                                   └────┬─────┘
                                                        │
                                                   ┌────▼─────┐
                                                   │ Database │
                                                   │(Postgres)│
                                                   └──────────┘
```

**Requisitos do DrawIO:**
- Usar as camadas do DrawIO (Layer) para separar: Network, Application, Data
- Cores: 🔵 Azul = Serviços, 🟢 Verde = Databases, 🟡 Amarelo = Caches, 🔴 Vermelho = External
- Setas com labels indicando protocolo (HTTP, TCP, gRPC)
- Notas com latências esperadas

**Critérios de aceite:**
- [ ] Arquivo `.drawio` criado e abrível no VS Code
- [ ] Mínimo 6 componentes no diagrama
- [ ] Setas com labels de protocolo
- [ ] Cores seguindo a convenção definida
- [ ] Pelo menos 2 camadas (layers) no DrawIO

---

## Parte 5 — Hello World em Go e Java

### 5.1 — Go: HTTP Server Básico

```go
// cmd/main.go
package main

import (
    "encoding/json"
    "log"
    "net/http"
)

type HealthResponse struct {
    Status  string `json:"status"`
    Service string `json:"service"`
}

func main() {
    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(HealthResponse{
            Status:  "UP",
            Service: "system-design-challenge",
        })
    })

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### 5.2 — Java: Spring Boot Health Endpoint

```java
// src/main/java/com/challenge/Application.java
package com.challenge;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// src/main/java/com/challenge/HealthController.java
package com.challenge;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HealthController {

    record HealthResponse(String status, String service) {}

    @GetMapping("/health")
    public HealthResponse health() {
        return new HealthResponse("UP", "system-design-challenge");
    }
}
```

**Critérios de aceite:**
- [ ] `go run cmd/main.go` inicia servidor na porta 8080
- [ ] `curl localhost:8080/health` retorna JSON válido
- [ ] `./mvnw spring-boot:run` inicia servidor na porta 8081
- [ ] `curl localhost:8081/health` retorna JSON válido
- [ ] Testes escritos para ambos os endpoints

---

## Docker Compose Base

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin
      POSTGRES_DB: systemdesign
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      CLUSTER_ID: system-design-cluster-001
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

volumes:
  postgres_data:
```

---

## Definição de Pronto (DoD)

- [ ] Ambiente configurado (Go, Java, Docker, DrawIO)
- [ ] Estrutura de diretórios criada
- [ ] Primeiro ADR escrito (technology stack)
- [ ] Primeiro diagrama DrawIO criado (web architecture)
- [ ] Hello World Go + Java rodando
- [ ] Docker Compose testado (postgres, redis, kafka)
- [ ] Commit: `feat(system-design-00): setup project structure and tooling`

---

## Checklist

- [ ] Go instalado e versão verificada (`go version`)
- [ ] Java 25 instalado (`java --version`)
- [ ] Maven configurado (`./mvnw --version`)
- [ ] Docker rodando (`docker --version`)
- [ ] DrawIO extensão instalada no VS Code
- [ ] Estrutura de pastas criada para todos os levels
- [ ] ADR-001 escrito e revisado
- [ ] Diagrama DrawIO aberto e editável
- [ ] Servidores Go e Java respondendo `/health`
- [ ] Docker Compose up com postgres + redis + kafka
