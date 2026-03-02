# Level 5 — Benchmarking e Microbenchmarks

> **Objetivo:** Dominar benchmarking rigoroso com JMH (Java Microbenchmark Harness) e Go `testing.B` — medindo performance de código isolado, evitando armadilhas de dead code elimination e constant folding, usando benchstat para análise estatística e integrando regressão de performance no CI/CD.

---

## Objetivo de Aprendizado

- Entender a diferença entre load test (sistema) e microbenchmark (código)
- Criar benchmarks JMH com anotações `@Benchmark`, `@Fork`, `@Warmup`, `@Measurement`
- Evitar armadilhas de JIT: dead code elimination, constant folding, loop optimization
- Usar `Blackhole` para prevenir eliminação de código pelo compilador
- Criar benchmarks Go com `testing.B`, `b.ResetTimer()`, `b.ReportAllocs()`
- Analisar resultados com `benchstat` (análise estatística, comparação, regressão)
- Integrar benchmark gates no CI/CD para detectar regressões automaticamente
- Aplicar benchmarks a cenários reais: serialização, hashing, pool de conexões, queries

---

## Escopo Funcional

### Cenários de Benchmark — Digital Wallet

```
CENÁRIOS DE MICROBENCHMARK:

1. Serialização JSON
   → Comparar Jackson vs Gson vs kotlinx.serialization (JVM)
   → Comparar encoding/json vs json-iterator vs easyjson (Go)

2. Hashing de Senha
   → Comparar bcrypt cost factors (10, 12, 14)
   → Comparar bcrypt vs argon2 vs scrypt

3. Cálculo de Saldo
   → Iteração simples vs stream vs parallel stream (JVM)
   → Loop vs goroutine fan-out (Go)

4. Pool de Conexões
   → HikariCP pool sizes: 5, 10, 20, 50, 100
   → pgxpool sizes equivalentes em Go

5. Validação de Transação
   → Regex vs manual parsing
   → Bean Validation vs validação manual (JVM)

6. String Building
   → Concatenação vs StringBuilder vs String.format (JVM)
   → Concatenação vs strings.Builder vs fmt.Sprintf (Go)
```

---

## Escopo Técnico

### 1. JMH — Java Microbenchmark Harness

```java
// JsonSerializationBenchmark.java
package com.wallet.benchmark;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.google.gson.Gson;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)       // Mede tempo médio por operação
@OutputTimeUnit(TimeUnit.NANOSECONDS)  // Resultado em nanosegundos
@State(Scope.Benchmark)                // Estado compartilhado entre threads
@Fork(value = 3, warmups = 1)          // 3 forks (processos JVM separados)
@Warmup(iterations = 5, time = 1)     // 5 iterações de warmup de 1s cada
@Measurement(iterations = 5, time = 1) // 5 iterações de medição de 1s cada
public class JsonSerializationBenchmark {

    private ObjectMapper jackson;
    private Gson gson;
    private WalletDTO wallet;
    private String walletJson;

    @Setup(Level.Trial)
    public void setup() {
        jackson = new ObjectMapper();
        gson = new Gson();
        wallet = new WalletDTO(1L, "user-123", 15000.50, "BRL");
        walletJson = """
            {"id":1,"userId":"user-123","balance":15000.50,"currency":"BRL"}
            """;
    }

    // ─── Serialização (Object → JSON) ───

    @Benchmark
    public void jackson_serialize(Blackhole bh) throws Exception {
        bh.consume(jackson.writeValueAsString(wallet));  // Blackhole previne DCE
    }

    @Benchmark
    public void gson_serialize(Blackhole bh) {
        bh.consume(gson.toJson(wallet));
    }

    // ─── Desserialização (JSON → Object) ───

    @Benchmark
    public void jackson_deserialize(Blackhole bh) throws Exception {
        bh.consume(jackson.readValue(walletJson, WalletDTO.class));
    }

    @Benchmark
    public void gson_deserialize(Blackhole bh) {
        bh.consume(gson.fromJson(walletJson, WalletDTO.class));
    }

    // ─── Runner ───
    public static void main(String[] args) throws Exception {
        Options opt = new OptionsBuilder()
            .include(JsonSerializationBenchmark.class.getSimpleName())
            .build();
        new Runner(opt).run();
    }
}
```

```java
// BalanceCalculationBenchmark.java
@BenchmarkMode({Mode.AverageTime, Mode.Throughput})
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Benchmark)
@Fork(3)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 5, time = 1)
public class BalanceCalculationBenchmark {

    @Param({"100", "1000", "10000", "100000"})  // Parametrizado!
    private int transactionCount;

    private List<Transaction> transactions;

    @Setup(Level.Trial)
    public void setup() {
        transactions = new ArrayList<>();
        Random rng = new Random(42);  // Seed fixa para reprodutibilidade
        for (int i = 0; i < transactionCount; i++) {
            transactions.add(new Transaction(
                rng.nextBoolean() ? TransactionType.DEPOSIT : TransactionType.WITHDRAWAL,
                BigDecimal.valueOf(rng.nextDouble() * 1000)
            ));
        }
    }

    @Benchmark
    public BigDecimal iterative(Blackhole bh) {
        BigDecimal balance = BigDecimal.ZERO;
        for (Transaction tx : transactions) {
            if (tx.type() == TransactionType.DEPOSIT) {
                balance = balance.add(tx.amount());
            } else {
                balance = balance.subtract(tx.amount());
            }
        }
        return balance;  // Retornar valor também previne DCE
    }

    @Benchmark
    public BigDecimal stream_sequential() {
        return transactions.stream()
            .map(tx -> tx.type() == TransactionType.DEPOSIT
                ? tx.amount() : tx.amount().negate())
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    @Benchmark
    public BigDecimal stream_parallel() {
        return transactions.parallelStream()
            .map(tx -> tx.type() == TransactionType.DEPOSIT
                ? tx.amount() : tx.amount().negate())
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

```java
// PasswordHashingBenchmark.java — Comparar cost factors
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Warmup(iterations = 3, time = 2)
@Measurement(iterations = 5, time = 2)
public class PasswordHashingBenchmark {

    @Param({"10", "12", "14"})
    private int bcryptCost;

    private String password;

    @Setup
    public void setup() {
        password = "MySecureP@ssw0rd!2024";
    }

    @Benchmark
    public String bcrypt_hash() {
        return BCrypt.hashpw(password, BCrypt.gensalt(bcryptCost));
    }

    @Benchmark
    public boolean bcrypt_verify() {
        String hash = BCrypt.hashpw(password, BCrypt.gensalt(bcryptCost));
        return BCrypt.checkpw(password, hash);
    }
}
```

#### JMH — Armadilhas Comuns

```java
// ❌ ERRADO: Dead Code Elimination (DCE)
@Benchmark
public void wrong_dce() {
    jackson.writeValueAsString(wallet);  // JIT elimina — resultado não usado!
}

// ✅ CORRETO: Usar Blackhole ou retornar valor
@Benchmark
public void correct_blackhole(Blackhole bh) {
    bh.consume(jackson.writeValueAsString(wallet));
}

@Benchmark
public String correct_return() {
    return jackson.writeValueAsString(wallet);
}

// ─────────────────────────────────────

// ❌ ERRADO: Constant Folding
@Benchmark
public int wrong_constant() {
    return 2 + 2;  // JIT pré-calcula em compile time!
}

// ✅ CORRETO: Usar @State para valores não-constantes
@State(Scope.Thread)
public static class MyState {
    int a = 2, b = 2;
}

@Benchmark
public int correct_state(MyState state) {
    return state.a + state.b;
}

// ─────────────────────────────────────

// ❌ ERRADO: Loop optimization
@Benchmark
public int wrong_loop() {
    int sum = 0;
    for (int i = 0; i < 1000; i++) {
        sum += i;  // JIT otimiza para fórmula fechada: n*(n-1)/2
    }
    return sum;
}

// ✅ CORRETO: JMH já faz o loop internamente!
// Cada invocação do @Benchmark = UMA operação
@Benchmark
public int correct_no_manual_loop(MyState state) {
    return state.a + state.b;
}

// ─────────────────────────────────────

// ❌ ERRADO: Sem Fork (JVM quente de benchmarks anteriores)
@Fork(0)  // NUNCA em produção — JVM contaminada
public class WrongBenchmark { }

// ✅ CORRETO: Fork >= 2 para isolamento
@Fork(3)  // 3 processos JVM separados
public class CorrectBenchmark { }
```

#### JMH Maven Setup

```xml
<!-- pom.xml para projeto JMH -->
<dependencies>
    <dependency>
        <groupId>org.openjdk.jmh</groupId>
        <artifactId>jmh-core</artifactId>
        <version>1.37</version>
    </dependency>
    <dependency>
        <groupId>org.openjdk.jmh</groupId>
        <artifactId>jmh-generator-annprocess</artifactId>
        <version>1.37</version>
        <scope>provided</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.5.1</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals><goal>shade</goal></goals>
                    <configuration>
                        <transformers>
                            <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                <mainClass>org.openjdk.jmh.Main</mainClass>
                            </transformer>
                        </transformers>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

```bash
# Executar JMH
mvn clean package
java -jar target/benchmarks.jar

# Filtrar benchmarks específicos
java -jar target/benchmarks.jar "JsonSerialization"

# Output em JSON para análise
java -jar target/benchmarks.jar -rf json -rff results.json

# Listar benchmarks disponíveis
java -jar target/benchmarks.jar -l
```

### 2. Go Benchmarks — `testing.B`

```go
// benchmark_test.go
package wallet

import (
    "encoding/json"
    "fmt"
    "strings"
    "testing"

    jsoniter "github.com/json-iterator/go"
)

// ── Estrutura de teste ──

type WalletDTO struct {
    ID       int64   `json:"id"`
    UserID   string  `json:"userId"`
    Balance  float64 `json:"balance"`
    Currency string  `json:"currency"`
}

var testWallet = WalletDTO{
    ID: 1, UserID: "user-123", Balance: 15000.50, Currency: "BRL",
}

var testJSON = []byte(`{"id":1,"userId":"user-123","balance":15000.50,"currency":"BRL"}`)

// ── Benchmark: Serialização JSON ──

func BenchmarkJSONMarshal_Stdlib(b *testing.B) {
    b.ReportAllocs() // Reporta alocações de memória
    for i := 0; i < b.N; i++ {
        data, err := json.Marshal(testWallet)
        if err != nil {
            b.Fatal(err)
        }
        _ = data // Previne otimização (compiler vê como usado)
    }
}

func BenchmarkJSONMarshal_JsonIterator(b *testing.B) {
    var jsonApi = jsoniter.ConfigCompatibleWithStandardLibrary
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        data, err := jsonApi.Marshal(testWallet)
        if err != nil {
            b.Fatal(err)
        }
        _ = data
    }
}

func BenchmarkJSONUnmarshal_Stdlib(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        var w WalletDTO
        if err := json.Unmarshal(testJSON, &w); err != nil {
            b.Fatal(err)
        }
    }
}

func BenchmarkJSONUnmarshal_JsonIterator(b *testing.B) {
    var jsonApi = jsoniter.ConfigCompatibleWithStandardLibrary
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        var w WalletDTO
        if err := jsonApi.Unmarshal(testJSON, &w); err != nil {
            b.Fatal(err)
        }
    }
}

// ── Benchmark: String Building ──

func BenchmarkStringConcat(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        s := ""
        for j := 0; j < 100; j++ {
            s += fmt.Sprintf("transaction-%d,", j)
        }
        _ = s
    }
}

func BenchmarkStringBuilder(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for j := 0; j < 100; j++ {
            fmt.Fprintf(&sb, "transaction-%d,", j)
        }
        _ = sb.String()
    }
}

// ── Benchmark: Sub-benchmarks parametrizados ──

func BenchmarkBalanceCalculation(b *testing.B) {
    sizes := []int{100, 1000, 10000, 100000}
    
    for _, size := range sizes {
        transactions := generateTransactions(size)
        
        b.Run(fmt.Sprintf("iterative/%d", size), func(b *testing.B) {
            b.ReportAllocs()
            for i := 0; i < b.N; i++ {
                _ = calculateBalanceIterative(transactions)
            }
        })
        
        b.Run(fmt.Sprintf("channel/%d", size), func(b *testing.B) {
            b.ReportAllocs()
            for i := 0; i < b.N; i++ {
                _ = calculateBalanceChannel(transactions)
            }
        })
    }
}

// ── Benchmark: Setup pesado com b.ResetTimer() ──

func BenchmarkWithHeavySetup(b *testing.B) {
    // Setup pesado (NÃO conta no benchmark)
    transactions := generateTransactions(100000)
    db := setupTestDatabase()
    defer db.Close()
    
    b.ResetTimer() // ← Reseta o timer APÓS o setup
    b.ReportAllocs()
    
    for i := 0; i < b.N; i++ {
        _ = processTransactions(db, transactions)
    }
}

// ── Benchmark: Paralelo ──

func BenchmarkParallelWalletLookup(b *testing.B) {
    cache := setupWalletCache()
    
    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            id := int64(b.N % 1000)
            _ = cache.Get(id)
        }
    })
}
```

#### Executar Benchmarks Go

```bash
# Executar todos os benchmarks
go test -bench=. -benchmem ./...

# Filtrar por nome
go test -bench=BenchmarkJSONMarshal -benchmem ./wallet/

# Executar N vezes para estabilidade
go test -bench=. -benchmem -count=10 ./... | tee bench_current.txt

# Salvar baseline
git stash  # ou checkout da versão anterior
go test -bench=. -benchmem -count=10 ./... | tee bench_baseline.txt
git stash pop

# Comparar com benchstat
benchstat bench_baseline.txt bench_current.txt
```

### 3. benchstat — Análise Estatística

```bash
# Instalar benchstat
go install golang.org/x/perf/cmd/benchstat@latest

# Comparar duas versões
benchstat old.txt new.txt

# Output exemplo:
#
# goos: linux
# goarch: amd64
# pkg: github.com/company/wallet
#
# name                     old time/op    new time/op    delta
# JSONMarshal_Stdlib-8       452ns ± 3%     445ns ± 2%     ~     (p=0.095 n=10+10)
# JSONMarshal_JsonIter-8     312ns ± 2%     198ns ± 1%   -36.5%  (p=0.000 n=10+10)
# BalanceCalc/iterative/100  1.23µs ± 1%   1.25µs ± 2%     ~     (p=0.182 n=10+10)
# BalanceCalc/iterative/10k  125µs ± 1%     89µs ± 1%   -28.8%  (p=0.000 n=10+10)
#
# name                     old alloc/op   new alloc/op   delta
# JSONMarshal_Stdlib-8       128B ± 0%       96B ± 0%   -25.0%  (p=0.000 n=10+10)
# JSONMarshal_JsonIter-8     64B ± 0%        32B ± 0%   -50.0%  (p=0.000 n=10+10)

# Interpretação:
#   ± X%     → variabilidade entre runs (ideal: < 5%)
#   ~        → sem diferença estatisticamente significativa
#   -36.5%   → melhoria de 36.5% (p=0.000 = altamente significativo)
#   p=0.095  → p-value > 0.05 = não significativo
#   n=10+10  → 10 amostras old + 10 amostras new
```

### 4. CI/CD Benchmark Regression Gate

```yaml
# .github/workflows/benchmark.yml
name: Benchmark Regression Check

on:
  pull_request:
    paths:
      - '**.go'
      - '**Benchmark**.java'

jobs:
  benchmark-go:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'

      - name: Install benchstat
        run: go install golang.org/x/perf/cmd/benchstat@latest

      - name: Run benchmarks (current)
        run: go test -bench=. -benchmem -count=10 ./... | tee bench_new.txt

      - name: Checkout base branch
        run: git checkout ${{ github.event.pull_request.base.sha }}

      - name: Run benchmarks (baseline)
        run: go test -bench=. -benchmem -count=10 ./... | tee bench_old.txt

      - name: Compare with benchstat
        run: |
          benchstat bench_old.txt bench_new.txt | tee benchstat_output.txt
          
          # Detectar regressões > 10%
          if grep -E '\+[1-9][0-9]\.[0-9]+%' benchstat_output.txt; then
            echo "❌ REGRESSION DETECTED: >10% performance degradation"
            exit 1
          fi
          echo "✅ No significant regressions"

      - name: Post results as PR comment
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const output = fs.readFileSync('benchstat_output.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Benchmark Results\n\`\`\`\n${output}\n\`\`\``
            });

  benchmark-jmh:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build and run JMH
        run: |
          cd benchmark
          mvn clean package -q
          java -jar target/benchmarks.jar \
            -rf json -rff results.json \
            -wi 3 -i 5 -f 2

      - name: Check for regressions
        run: |
          python3 scripts/check_jmh_regression.py \
            --current results.json \
            --baseline benchmark/baseline.json \
            --threshold 10
```

```python
# scripts/check_jmh_regression.py
import json
import sys
import argparse

def check_regression(current_file, baseline_file, threshold_pct):
    with open(current_file) as f:
        current = {b["benchmark"]: b for b in json.load(f)}
    with open(baseline_file) as f:
        baseline = {b["benchmark"]: b for b in json.load(f)}

    regressions = []
    for name, curr in current.items():
        if name not in baseline:
            continue
        base = baseline[name]
        
        curr_score = curr["primaryMetric"]["score"]
        base_score = base["primaryMetric"]["score"]
        
        # Para AverageTime: maior é pior
        if curr["mode"] == "avgt":
            delta_pct = ((curr_score - base_score) / base_score) * 100
            if delta_pct > threshold_pct:
                regressions.append((name, base_score, curr_score, delta_pct))
        # Para Throughput: menor é pior
        elif curr["mode"] == "thrpt":
            delta_pct = ((base_score - curr_score) / base_score) * 100
            if delta_pct > threshold_pct:
                regressions.append((name, base_score, curr_score, -delta_pct))

    if regressions:
        print(f"❌ {len(regressions)} regression(s) detected (threshold: {threshold_pct}%):\n")
        for name, old, new, delta in regressions:
            print(f"  {name}: {old:.2f} → {new:.2f} ({delta:+.1f}%)")
        sys.exit(1)
    else:
        print(f"✅ No regressions above {threshold_pct}% threshold")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--current", required=True)
    parser.add_argument("--baseline", required=True)
    parser.add_argument("--threshold", type=float, default=10.0)
    args = parser.parse_args()
    check_regression(args.current, args.baseline, args.threshold)
```

---

## Critérios de Aceite

- [ ] **JMH**: 3+ benchmark classes cobrindo serialização, hashing, e cálculo de saldo
- [ ] **JMH**: Uso correto de `Blackhole`, `@State`, `@Param`, `@Fork(>=2)`
- [ ] **JMH**: Output em JSON (`-rf json`) para análise automatizada
- [ ] **Go**: 3+ benchmark functions com `b.ReportAllocs()`, sub-benchmarks parametrizados
- [ ] **Go**: Benchmark paralelo com `b.RunParallel()`
- [ ] **benchstat**: Comparação entre duas versões com `-count=10`
- [ ] **benchstat**: Identificar pelo menos 1 melhoria e 1 regressão entre versões
- [ ] CI/CD pipeline que detecta regressões de performance automaticamente
- [ ] Script `check_jmh_regression.py` funcional
- [ ] Relatório `BENCHMARK_REPORT.md` com:
  - Tabela de resultados JMH e Go benchmarks
  - Análise de alocações de memória
  - Recomendações baseadas nos resultados

---

## Definição de Pronto (DoD)

- [ ] Projeto JMH em `benchmark/jmh/` com `pom.xml` e classes de benchmark
- [ ] Benchmarks Go em `*_test.go` com tag `// +build bench` ou sub-pasta
- [ ] Script `run-benchmarks.sh` para executar JMH + Go + benchstat
- [ ] Pipeline CI com benchmark regression gate
- [ ] Baseline salva em `benchmark/baseline/`
- [ ] Relatório `BENCHMARK_REPORT.md`
- [ ] Commit: `feat(perf-level-5): add JMH and Go microbenchmarks with CI regression gate`

---

## Checklist

### JMH

- [ ] `pom.xml` com `jmh-core` e `jmh-generator-annprocess`
- [ ] Classe `JsonSerializationBenchmark` com `Blackhole`
- [ ] Classe `BalanceCalculationBenchmark` com `@Param`
- [ ] Classe `PasswordHashingBenchmark` com cost factors
- [ ] Verificar que NENHUM benchmark tem `@Fork(0)` em produção
- [ ] Run com `java -jar benchmarks.jar -rf json`

### Go Benchmarks

- [ ] `benchmark_test.go` com `BenchmarkJSON*`
- [ ] Sub-benchmarks com `b.Run()` para tamanhos variados
- [ ] `b.ResetTimer()` após setup pesado
- [ ] `b.ReportAllocs()` em todos os benchmarks
- [ ] `b.RunParallel()` para benchmark concorrente
- [ ] `go test -bench=. -benchmem -count=10`

### benchstat

- [ ] Instalar: `go install golang.org/x/perf/cmd/benchstat@latest`
- [ ] Gerar baseline: `-count=10 > baseline.txt`
- [ ] Gerar current: `-count=10 > current.txt`
- [ ] Comparar: `benchstat baseline.txt current.txt`
- [ ] Interpretar: `~` = sem diferença, `±X%` = variabilidade, `p=X` = significância

---

## Tarefas Sugeridas por Stack

### Go (Gin)

- [ ] Benchmarks `testing.B` para todos os handlers
- [ ] Benchmark de middleware (logging, auth, rate limiting)
- [ ] `pprof` integrado: `go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof`
- [ ] Comparação: `encoding/json` vs `json-iterator` vs `easyjson`

### Spring Boot

- [ ] JMH benchmarks para services e serialização
- [ ] Spring Boot `@SpringBootTest` benchmark (startup time)
- [ ] JFR recording durante benchmarks: `-XX:StartFlightRecording=filename=bench.jfr`

### Quarkus

- [ ] JMH + GraalVM native image benchmark (startup + throughput)
- [ ] Comparar JIT vs AOT para mesmas operações
- [ ] Quarkus Dev Services benchmark (database queries)

### Micronaut

- [ ] JMH benchmark de injeção de dependências (compile-time DI)
- [ ] Comparar startup time: Micronaut vs Spring Boot vs Quarkus
- [ ] Serialização com Micronaut Serialization vs Jackson

### Jakarta EE

- [ ] JMH benchmarks para EJB vs CDI bean lookup
- [ ] Benchmark de JPA queries parametrizadas vs named queries
- [ ] Comparar connection pool implementations (HikariCP vs pool nativo)

---

## Extensões Opcionais

- [ ] JMH Profiler: `-prof gc` (GC), `-prof stack` (stack sampling), `-prof async` (async-profiler)
- [ ] JMH com GraalVM CE vs Oracle JDK vs OpenJDK — comparar JIT compilers
- [ ] Go benchmark com `-benchtime=5s` para operações lentas
- [ ] Go benchmark com `-cpu 1,2,4,8` para escalar cores
- [ ] Continuous benchmarking com GitHub Actions + benchmark action
- [ ] Comparar resultados de benchmark com resultados de load test (micro vs macro)

---

## Erros Comuns

| Erro | Consequência | Correção |
|------|-------------|----------|
| Sem `Blackhole` no JMH | JIT elimina código — resultado irreal | Sempre `bh.consume()` ou retornar |
| `@Fork(0)` no JMH | JVM contaminada por estado anterior | `@Fork(>=2)`, idealmente 3 |
| `-count=1` no Go bench | Sem significância estatística | `-count=10` + benchstat |
| Manual loop no `@Benchmark` | Mede overhead do loop, não da operação | JMH já faz o loop |
| Setup no corpo do benchmark | Mede setup + operação juntos | `@Setup(Level.Trial)` / `b.ResetTimer()` |
| Constant folding não tratado | JIT pré-calcula resultado | Input via `@State` ou `@Param` |
| Comparar sem p-value | "Melhoria" pode ser variação normal | benchstat com `-count=10` |
| Benchmark em laptop (throttling) | Resultados variam por temperatura | Máquina dedicada ou CI |
| Ignorar alocações | Otimiza CPU mas gera GC pressure | `-benchmem` / `b.ReportAllocs()` |

---

## Como Isso Aparece em Entrevistas

- "O que é dead code elimination e como JMH lida com isso?"
- "Explique Blackhole no JMH. Por que é necessário?"
- "Como você detectaria uma regressão de performance no CI/CD?"
- "Qual a diferença entre microbenchmark e load test? Quando usar cada um?"
- "Descreva o processo de comparar performance de duas implementações com rigor estatístico"
- "O que `@Fork` faz no JMH e por que é importante?"
- "Como `b.ResetTimer()` funciona em Go benchmarks?"
- "Explique p-value no contexto de benchstat"
