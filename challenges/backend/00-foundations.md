# Level 0 — Fundamentos

> **Objetivo:** Construir a base sólida nas duas linguagens (Go e Java 25+) antes de tocar em frameworks. Sem atalhos.

---

## Objetivo de Aprendizado

- Dominar sintaxe, tipos, controle de fluxo, e idiomático de Go e Java 25+
- Entender o modelo de execução de cada linguagem (goroutines vs JVM threads/virtual threads)
- Configurar tooling profissional (IDE, linter, formatter, build tool)
- Criar o projeto base seguindo as convenções dos READMEs de referência

---

## Escopo Funcional

Nenhum endpoint HTTP ainda. Este nível foca exclusivamente em:

1. **Setup do projeto** com a estrutura de diretórios correta
2. **Exercícios de linguagem** (sem framework web)
3. **Primeiro teste** (provar que o tooling funciona)

---

## Escopo Técnico

### Go

| Tópico | O que dominar |
|---|---|
| **Setup** | Go 1.24+, `go mod init`, VS Code + gopls, golangci-lint |
| **Tipos** | Structs, interfaces, slices, maps, pointers |
| **Erros** | `error` interface, `fmt.Errorf("%w")`, `errors.Is/As`, sentinel errors |
| **Packages** | `internal/`, visibility (PascalCase vs camelCase), package naming |
| **Interfaces** | Implícitas, interface segregation, "accept interfaces, return structs" |
| **Testing** | `testing` stdlib, `go test`, table-driven tests, `t.Run()` |
| **Concorrência (intro)** | Goroutines, channels (básico), `sync.WaitGroup` |
| **Context** | `context.Context`, `context.WithCancel`, `context.WithTimeout`, propagation |
| **Tooling** | `go build`, `go test`, `go vet`, `golangci-lint`, `Makefile` |

### Java

| Tópico | O que dominar |
|---|---|
| **Setup** | JDK 25, Maven wrapper, VS Code ou IntelliJ, JaCoCo |
| **Java moderno** | Records, sealed classes, pattern matching (`switch`, `instanceof`), text blocks |
| **Collections** | List, Map, Set, Streams API, Optional, collectors |
| **Concorrência (intro)** | `Thread`, `Runnable`, Virtual Threads (`Thread.ofVirtual()`), `CompletableFuture` |
| **JVM** | Heap, stack, GC basics (G1, ZGC), `-Xmx`, `-Xms`, `jconsole` |
| **Erros** | Checked vs unchecked exceptions, custom exceptions, try-with-resources |
| **Testing** | JUnit 5, `@Test`, `@ParameterizedTest`, `@ValueSource`, `assertThrows` |
| **Build** | Maven lifecycle (clean, compile, test, package, verify), POM basics |

---

## Critérios de Aceite

### Go
- [ ] Projeto inicializado com `go mod init` e estrutura `cmd/api/main.go` + `internal/`
- [ ] Pelo menos 5 exercícios de linguagem com testes (table-driven)
- [ ] `golangci-lint run` passa sem erros
- [ ] `Makefile` com targets: `build`, `test`, `lint`
- [ ] Sentinel errors definidos em `internal/errs/errors.go`
- [ ] Exercício usando `context.WithTimeout`
- [ ] Exercício usando goroutine + channel

### Java
- [ ] Projeto Maven inicializado com `pom.xml`, JDK 25
- [ ] Pelo menos 5 exercícios de linguagem com testes JUnit 5
- [ ] Uso de record, sealed class e pattern matching em pelo menos 1 exercício
- [ ] Streams API aplicado em processamento de coleções
- [ ] Virtual Threads: exercício comparando platform threads vs virtual threads
- [ ] `./mvnw test` passa sem erros
- [ ] JaCoCo configurado (mesmo que sem threshold ainda)

---

## Definição de Pronto (DoD)

- [ ] Código compila sem erros
- [ ] Todos os testes passam
- [ ] Linter/formatter aplicado
- [ ] README.md do projeto criado com instruções de setup e execução
- [ ] Commit semântico: `feat(level-0): setup project structure and language exercises`

---

## Checklist

- [ ] Go instalado e versão verificada (`go version`)
- [ ] Java 25 instalado e versão verificada (`java --version`)
- [ ] Maven wrapper configurado (`./mvnw --version`)
- [ ] IDE configurada com extensões (Go / Java)
- [ ] Projeto Go criado com estrutura do README
- [ ] Projeto Java criado com estrutura do README
- [ ] Exercícios de linguagem implementados
- [ ] Testes escritos e passando
- [ ] Linting configurado e passando

---

## Tarefas Sugeridas por Stack

### Go (Gin)

```bash
# 1. Criar projeto
mkdir -p wallet-go && cd wallet-go
go mod init github.com/{seu-user}/wallet-go

# 2. Criar estrutura (conforme README de referência)
mkdir -p cmd/api internal/{config,handler,service,store,model,middleware,router,platform,errs}
mkdir -p migrations test/{integration,fixture}

# 3. Criar main.go mínimo
cat > cmd/api/main.go << 'EOF'
package main

import "fmt"

func main() {
    fmt.Println("wallet-go starting...")
}
EOF

# 4. Verificar
go build ./cmd/api
go vet ./...
```

**Exercícios Go recomendados (com testes):**

1. **Erros de domínio** — Criar `errs/errors.go` com `ErrNotFound`, `ErrConflict`, `ErrBadRequest` como sentinel errors + custom error types com `Error()` e `Unwrap()`
2. **Struct + interface** — Criar um `UserValidator` que valida email e nome usando interface
3. **Slice processing** — Função que filtra, mapeia e reduz uma slice de structs
4. **Context timeout** — Função que simula operação lenta e cancela via `context.WithTimeout`
5. **Goroutine + channel** — Worker pool que processa N tasks concorrentemente
6. **Pagination** — Implementar `Params` e `Page[T]` genéricos (Go 1.18+ generics)
7. **Config loading** — Implementar `config.Load()` com `caarlos0/env`

### Spring Boot

```bash
# 1. Gerar projeto (Spring Initializr ou manualmente)
# https://start.spring.io com: Web, Validation, DevTools, JPA, Flyway, Actuator

# 2. Criar estrutura de pacotes (conforme README de referência)
# {group-id}/
#   ├── api/exception/
#   ├── api/rest/v1/{controller,request,response,openapi}/
#   ├── core/{mapper,validation}/
#   ├── domain/{entity,repository/mysql,service,exception}/
#   └── infrastructure/

# 3. Verificar
./mvnw compile
./mvnw test
```

**Exercícios Java recomendados (com testes):**

1. **Records** — Criar `UserRequest`, `UserResponse` e `WalletResponse` como records com métodos utilitários
2. **Sealed classes** — Criar hierarquia `TransactionType` (DEPOSIT, WITHDRAWAL, TRANSFER) como sealed
3. **Pattern matching** — Switch expression com pattern matching sobre sealed class
4. **Streams** — Processar lista de transações: filtrar por tipo, agrupar por status, calcular soma
5. **Custom exceptions** — Criar `ResourceNotFoundException`, `BusinessException`, `InsufficientBalanceException`
6. **Virtual Threads** — Benchmark: executar 10.000 tasks I/O-bound com platform threads vs virtual threads, medir tempo
7. **Optional** — Exercício com chain de Optional (map, flatMap, orElseThrow)

### Quarkus / Micronaut / Jakarta EE

> **Neste nível:** Apenas configure o projeto base com a estrutura de diretórios. Os exercícios de linguagem Java são compartilhados (não precisa refazer). O foco é garantir que o tooling de cada framework funciona.

```bash
# Quarkus
mvn io.quarkus:quarkus-maven-plugin:create \
    -DprojectGroupId=com.example \
    -DprojectArtifactId=wallet-quarkus

# Micronaut
mn create-app com.example.wallet-micronaut --build=maven --lang=java

# Jakarta EE (Maven archetype)
mvn archetype:generate -DarchetypeGroupId=org.wildfly.archetype \
    -DarchetypeArtifactId=wildfly-jakartaee-webapp-archetype
```

---

## Extensões Opcionais

- [ ] Implementar um CLI simples em Go que processa argumentos
- [ ] Explorar `go generate` e code generation
- [ ] Implementar benchmark com `testing.B` em Go
- [ ] Estudar `jshell` para prototipar código Java interativamente
- [ ] Explorar `jcmd` e `jstat` para monitorar JVM
- [ ] Implementar `ScopedValue` (Java 25 preview) como alternativa a ThreadLocal

---

## Erros Comuns

| Erro | Stack | Solução |
|---|---|---|
| Importar pacote `internal/` de fora do módulo | Go | `internal/` é enforced pelo Go — é por design |
| Esquecer `go mod tidy` | Go | Sempre rode após adicionar dependência |
| Usar `var err error` sem checar | Go | Sempre `if err != nil { return }` |
| Não entender diferença entre `nil` interface e `nil` pointer | Go | Interface nil ≠ pointer nil — clássica pegadinha |
| Usar `synchronized` com Virtual Threads | Java | Causa thread pinning — use `ReentrantLock` |
| Não configurar JDK 25 no Maven | Java | `<maven.compiler.release>25</maven.compiler.release>` |
| Ignorar `Optional.isPresent()` antes de `get()` | Java | Use `orElseThrow()`, `map()`, `ifPresent()` |
| Record com campo mutável | Java | Records são shallowly immutable — cuidado com `List` não immutable |

---

## Como Isso Aparece em Entrevistas

### Go
- "Explique como funciona o error handling em Go e por que não tem exceptions"
- "Qual a diferença entre goroutine e thread do OS?"
- "O que é interface implícita e por que Go funciona assim?"
- "Como context.Context é usado em Go e por que é importante?"
- "Explique o modelo de memória do Go e race conditions"

### Java
- "Quais as vantagens de records sobre classes POJO?"
- "Explique sealed classes e quando usá-las"
- "O que são Virtual Threads e como diferem de platform threads?"
- "Qual a diferença entre G1 e ZGC?"
- "Explique checked vs unchecked exceptions e quando usar cada"
- "Como funciona o class loading na JVM?"

---

## Comandos de Execução/Teste

### Go
```bash
# Build
go build -o bin/api ./cmd/api

# Testes
go test ./internal/... -v

# Cobertura
go test ./internal/... -coverprofile=coverage.out
go tool cover -html=coverage.out

# Lint
golangci-lint run ./...

# Vet
go vet ./...
```

### Java (Maven)
```bash
# Compile
./mvnw compile

# Testes
./mvnw test

# Package
./mvnw package -DskipTests

# Verificar versão
java --version
./mvnw --version
```
