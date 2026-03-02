# Level 4 — Security Testing

> **Objetivo:** Implementar testes de segurança completos para a Digital Wallet — OWASP Top 10,
> SAST, DAST, SCA, secret scanning, threat modeling e segurança de APIs — com pipeline DevSecOps
> integrado ao CI/CD em Go e Java (multi-framework).

**Referência:** [.docs/QUALITY-ENGINEERING/04-security-testing.md](../../.docs/QUALITY-ENGINEERING/04-security-testing.md)

---

## Contexto do Domínio

A Digital Wallet processa **dados financeiros e pessoais sensíveis** — saldos, transações, CPFs,
tokens de autenticação. Uma vulnerabilidade de segurança pode resultar em perda financeira direta,
vazamento de dados pessoais (LGPD) e destruição de confiança do usuário. Neste nível, você
implementará uma estratégia de segurança Defense-in-Depth com múltiplas camadas de proteção.

---

## Desafios

### Desafio 4.1 — Threat Modeling da Digital Wallet (STRIDE)

**Contexto:** Antes de escrever qualquer teste de segurança, a equipe precisa mapear as ameaças
específicas do domínio financeiro usando o framework STRIDE.

**Requisitos:**

- Criar o **Data Flow Diagram (DFD)** da Digital Wallet com trust boundaries:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     TRUST BOUNDARY — Internet                       │
│  ┌──────────┐                                                       │
│  │  Client  │                                                       │
│  │ (Mobile/ │                                                       │
│  │   Web)   │                                                       │
│  └────┬─────┘                                                       │
├───────┼─────────────────────────────────────────────────────────────┤
│       │ HTTPS + JWT                                                 │
│  ┌────▼─────┐    ┌──────────────┐    ┌──────────────┐              │
│  │   API    │───►│   Wallet     │───►│  PostgreSQL  │              │
│  │ Gateway  │    │   Service    │    │              │              │
│  └──────────┘    └──────┬───────┘    └──────────────┘              │
│                         │                                           │
│                    ┌────▼─────┐    ┌──────────────┐                │
│                    │  Event   │───►│    Kafka      │                │
│                    │ Publisher│    │              │                │
│                    └──────────┘    └──────────────┘                │
│                     TRUST BOUNDARY — Internal Network               │
└─────────────────────────────────────────────────────────────────────┘
```

- Aplicar STRIDE em cada componente:

| Componente | Ameaça (STRIDE) | Exemplo concreto | Impacto | Mitigação |
|---|---|---|---|---|
| API Gateway | **Spoofing** | Token JWT forjado ou expirado | Crítico | Validação de JWT com JWKS, expiração curta |
| API Gateway | **DoS** | Flood de requests em `/deposit` | Alto | Rate limiting (100 req/min por IP/user) |
| Wallet Service | **Tampering** | Alterar valor de transferência no payload | Crítico | Validação server-side, nunca confiar no client |
| Wallet Service | **Elevation** | Usuário acessa wallet de outro usuário | Crítico | Ownership check: `wallet.userId == auth.userId` |
| PostgreSQL | **Info Disclosure** | SQL injection expõe dados de outros usuários | Crítico | Parameterized queries, principle of least privilege |
| PostgreSQL | **Tampering** | Alterar saldo diretamente no banco | Crítico | DB user com permissões mínimas, audit trail |
| Kafka | **Repudiation** | Transação processada mas não rastreável | Alto | Audit log imutável, correlation IDs |
| Kafka | **Info Disclosure** | Dados sensíveis no payload do evento | Alto | Criptografia de payload, mascaramento de PII |

- Documentar: para cada ameaça, o **teste** que valida a mitigação

**Critérios de aceite:**

- [ ] DFD com trust boundaries desenhado
- [ ] STRIDE aplicado com pelo menos 8 ameaças mapeadas
- [ ] Mitigação definida para cada ameaça
- [ ] Testes associados a cada mitigação
- [ ] Documento: `decisions/04-threat-model.md`

---

### Desafio 4.2 — Testes de Autorização e Broken Access Control (OWASP A1)

**Contexto:** Broken Access Control é a vulnerabilidade #1 do OWASP Top 10. Na Digital Wallet,
um usuário **nunca** deve acessar, modificar ou visualizar wallets/transações de outro usuário.

**Requisitos:**

- Implementar testes unitários e de integração para:
  - **IDOR (Insecure Direct Object Reference):** acessar wallet de outro usuário via ID
  - **Privilege escalation:** usuário com role USER tenta acessar endpoint admin
  - **Horizontal access:** usuário A tenta ver transações de usuário B
  - **Missing authorization:** endpoint sem autenticação retorna dados sensíveis

**Java 25 (Spring Boot / Spring Security):**
```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class AuthorizationSecurityTest {

    @Autowired
    private TestRestTemplate restTemplate;

    // --- IDOR: Acessar wallet de outro usuário ---
    @Test
    void getWallet_fromAnotherUser_returns403() {
        // Arrange — Usuário A cria uma wallet
        var userA = createUser("user-a@test.com");
        var walletA = createWallet(userA.getId(), tokenOf(userA));

        // Act — Usuário B tenta acessar wallet de A
        var userB = createUser("user-b@test.com");
        var response = restTemplate.exchange(
                "/api/v1/wallets/{id}", HttpMethod.GET,
                authenticatedRequest(tokenOf(userB)),
                WalletResponse.class, walletA.getId()
        );

        // Assert
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.FORBIDDEN);
    }

    // --- Privilege Escalation: USER tenta endpoint admin ---
    @Test
    void listAllUsers_withUserRole_returns403() {
        var regularUser = createUser("regular@test.com");

        var response = restTemplate.exchange(
                "/api/v1/admin/users", HttpMethod.GET,
                authenticatedRequest(tokenOf(regularUser)),
                String.class
        );

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.FORBIDDEN);
    }

    // --- Missing Auth: endpoint sem token ---
    @Test
    void getWallet_withoutToken_returns401() {
        var response = restTemplate.getForEntity(
                "/api/v1/wallets/{id}", String.class, UUID.randomUUID()
        );

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.UNAUTHORIZED);
    }

    // --- Horizontal Access: ver transações de outro usuário ---
    @Test
    void getTransactions_ofAnotherUser_returns403() {
        var userA = createUser("userA@test.com");
        var walletA = createWallet(userA.getId(), tokenOf(userA));
        deposit(walletA.getId(), "100.00", tokenOf(userA));

        var userB = createUser("userB@test.com");
        var response = restTemplate.exchange(
                "/api/v1/wallets/{id}/transactions", HttpMethod.GET,
                authenticatedRequest(tokenOf(userB)),
                String.class, walletA.getId()
        );

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.FORBIDDEN);
    }
}
```

**Go 1.26 (middleware de autorização):**
```go
func TestAuthorization_IDOR(t *testing.T) {
    app := setupTestApp(t)

    // Arrange — Usuário A cria wallet
    userA := app.CreateUser(t, "userA@test.com")
    walletA := app.CreateWallet(t, userA.ID, userA.Token)

    // Arrange — Usuário B tenta acessar
    userB := app.CreateUser(t, "userB@test.com")

    // Act
    resp := app.GET(t, fmt.Sprintf("/api/v1/wallets/%s", walletA.ID), userB.Token)

    // Assert
    assert.Equal(t, http.StatusForbidden, resp.StatusCode)
}

func TestAuthorization_MissingToken(t *testing.T) {
    app := setupTestApp(t)

    resp := app.GET(t, "/api/v1/wallets/"+uuid.NewString(), "")

    assert.Equal(t, http.StatusUnauthorized, resp.StatusCode)
}

func TestAuthorization_PrivilegeEscalation(t *testing.T) {
    app := setupTestApp(t)

    regularUser := app.CreateUser(t, "regular@test.com") // role: USER

    resp := app.GET(t, "/api/v1/admin/users", regularUser.Token)

    assert.Equal(t, http.StatusForbidden, resp.StatusCode)
}
```

**Quarkus (JAX-RS + @RolesAllowed):**
```java
@QuarkusTest
@TestHTTPEndpoint(WalletResource.class)
class WalletAuthorizationTest {

    @Test
    @TestSecurity(user = "user-b", roles = "USER")
    void getWallet_fromAnotherUser_returns403() {
        given()
            .pathParam("id", walletOfUserA.getId())
        .when()
            .get("/{id}")
        .then()
            .statusCode(403);
    }

    @Test
    void getWallet_withoutAuth_returns401() {
        given()
            .pathParam("id", UUID.randomUUID())
        .when()
            .get("/{id}")
        .then()
            .statusCode(401);
    }
}
```

**Critérios de aceite:**

- [ ] Testes de IDOR para wallet e transações (pelo menos 3 cenários)
- [ ] Teste de privilege escalation (USER → ADMIN)
- [ ] Teste de endpoint sem autenticação (401)
- [ ] Teste de acesso horizontal entre usuários (403)
- [ ] Testes passando em todas as stacks (`go test`, `./mvnw test`)
- [ ] Cobertura de autorização ≥ 90% nos endpoints

---

### Desafio 4.3 — Testes de Injection e Input Validation (OWASP A3)

**Contexto:** A Digital Wallet recebe inputs em diversos endpoints — nomes, documentos, valores,
descrições. Cada input é um vetor potencial de ataque se não validado corretamente.

**Requisitos:**

- Implementar testes contra:
  - **SQL Injection:** em buscas, filtros e parâmetros de consulta
  - **NoSQL Injection:** se usar MongoDB/Redis para cache
  - **XSS (Cross-Site Scripting):** em campos de texto que são retornados ao cliente
  - **Mass Assignment:** enviar campos não permitidos no payload (ex.: `balance`, `role`)

**Java 25 (testes de integração):**
```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class InjectionSecurityTest {

    @Autowired
    private TestRestTemplate restTemplate;

    // --- SQL Injection ---
    @ParameterizedTest
    @ValueSource(strings = {
        "'; DROP TABLE wallets; --",
        "1' OR '1'='1",
        "1; SELECT * FROM users --",
        "' UNION SELECT id, balance FROM wallets --"
    })
    void searchTransactions_withSQLInjection_returnsEmptyOrBadRequest(String maliciousInput) {
        var user = createAuthenticatedUser();
        var wallet = createWallet(user);

        var response = restTemplate.exchange(
                "/api/v1/wallets/{walletId}/transactions?description={desc}",
                HttpMethod.GET,
                authenticatedRequest(user.getToken()),
                String.class, wallet.getId(), maliciousInput
        );

        assertThat(response.getStatusCode())
                .isIn(HttpStatus.OK, HttpStatus.BAD_REQUEST);
        // Verificar que tabela ainda existe
        assertThat(walletRepository.count()).isGreaterThanOrEqualTo(0);
    }

    // --- XSS ---
    @Test
    void createUser_withXSSInName_sanitizesOrRejects() {
        var request = new CreateUserRequest(
                "<script>alert('xss')</script>",
                "xss@test.com",
                "12345678900"
        );

        var response = restTemplate.postForEntity(
                "/api/v1/users", request, String.class
        );

        if (response.getStatusCode().is2xxSuccessful()) {
            assertThat(response.getBody()).doesNotContain("<script>");
        } else {
            assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
        }
    }

    // --- Mass Assignment ---
    @Test
    void createUser_withRoleInPayload_ignoresRoleField() {
        var json = """
            {
                "name": "Hacker",
                "email": "hacker@test.com",
                "document": "12345678900",
                "role": "ADMIN",
                "balance": 999999.99
            }
            """;

        var response = restTemplate.exchange(
                "/api/v1/users", HttpMethod.POST,
                new HttpEntity<>(json, jsonHeaders()),
                UserResponse.class
        );

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().role()).isNotEqualTo("ADMIN");
    }
}
```

**Go 1.26:**
```go
func TestInjection_SQLInjection(t *testing.T) {
    app := setupTestApp(t)
    user := app.CreateAuthenticatedUser(t)
    wallet := app.CreateWallet(t, user.ID, user.Token)

    maliciousInputs := []string{
        "'; DROP TABLE wallets; --",
        "1' OR '1'='1",
        "1; SELECT * FROM users --",
        "' UNION SELECT id, balance FROM wallets --",
    }

    for _, input := range maliciousInputs {
        t.Run(input, func(t *testing.T) {
            url := fmt.Sprintf("/api/v1/wallets/%s/transactions?description=%s",
                wallet.ID, url.QueryEscape(input))

            resp := app.GET(t, url, user.Token)

            assert.Contains(t, []int{http.StatusOK, http.StatusBadRequest}, resp.StatusCode)
            // Tabela ainda existe
            count, err := app.DB.QueryContext(context.Background(),
                "SELECT COUNT(*) FROM wallets")
            require.NoError(t, err)
            count.Close()
        })
    }
}

func TestInjection_MassAssignment(t *testing.T) {
    app := setupTestApp(t)

    body := `{
        "name": "Hacker",
        "email": "hacker@test.com",
        "document": "12345678900",
        "role": "ADMIN",
        "balance": 999999.99
    }`

    resp := app.POST(t, "/api/v1/users", body, "")

    assert.Equal(t, http.StatusCreated, resp.StatusCode)
    var user UserResponse
    json.NewDecoder(resp.Body).Decode(&user)
    assert.NotEqual(t, "ADMIN", user.Role)
}
```

**Critérios de aceite:**

- [ ] Testes de SQL injection com pelo menos 4 payloads distintos
- [ ] Testes de XSS em campos de texto (name, description)
- [ ] Testes de mass assignment em criação de user e wallet
- [ ] Nenhum teste de injection causa erro 500 (server error)
- [ ] Testes passando em Go e Java
- [ ] Evidência de que parameterized queries são usadas (não concatenação)

---

### Desafio 4.4 — SAST — Static Application Security Testing

**Contexto:** SAST analisa o código-fonte **sem executá-lo**, detectando vulnerabilidades
como secrets hardcoded, uso de funções inseguras e padrões perigosos.

**Requisitos:**

- **Go — gosec + golangci-lint:**
  - Configurar `gosec` como linter no `.golangci.yml`
  - Criar regras customizadas para o domínio financeiro
  - Rodar `golangci-lint run` sem findings HIGH/CRITICAL

```yaml
# .golangci.yml — security linters
linters:
  enable:
    - gosec          # Security scanner
    - govet          # Vet examines Go source code
    - staticcheck    # Static analysis
    - errcheck       # Error handling check
    - bodyclose      # HTTP response body close check
    - sqlclosecheck  # SQL rows/stmt close check

linters-settings:
  gosec:
    excludes:
      - G104  # Audit errors not checked (use errcheck instead)
    config:
      global:
        audit: true
```

- **Java — Semgrep + SpotBugs:**
  - Configurar Semgrep com regras OWASP:
  - Configurar SpotBugs no `pom.xml` com plugin FindSecBugs

```xml
<!-- pom.xml — SpotBugs + FindSecBugs -->
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.8.6</version>
    <configuration>
        <effort>Max</effort>
        <threshold>Low</threshold>
        <plugins>
            <plugin>
                <groupId>com.h3xstream.findsecbugs</groupId>
                <artifactId>findsecbugs-plugin</artifactId>
                <version>1.13.0</version>
            </plugin>
        </plugins>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

- Regras customizadas com Semgrep para Digital Wallet:

```yaml
# .semgrep/digital-wallet.yml
rules:
  - id: no-float-for-money
    patterns:
      - pattern: |
          float $VAR = ...;
      - pattern: |
          double $VAR = ...;
    message: "Valores monetários devem usar BigDecimal, não float/double"
    severity: ERROR
    languages: [java]

  - id: no-log-sensitive-data
    patterns:
      - pattern: |
          log.$METHOD(..., $EXPR.getCpf(), ...)
      - pattern: |
          log.$METHOD(..., $EXPR.getDocument(), ...)
      - pattern: |
          log.$METHOD(..., $EXPR.getPassword(), ...)
    message: "Dados sensíveis (CPF, senha) NÃO devem ser logados"
    severity: ERROR
    languages: [java]

  - id: wallet-ownership-check
    patterns:
      - pattern: |
          walletRepository.findById($ID)
      - pattern-not-inside: |
          ... .filter(w -> w.getUserId().equals($USER)) ...
      - pattern-not-inside: |
          ... if (...getUserId()...equals...) ...
    message: "Acesso a wallet deve verificar ownership do usuário"
    severity: WARNING
    languages: [java]
```

**Critérios de aceite:**

- [ ] `gosec` configurado no golangci-lint e passando
- [ ] SpotBugs + FindSecBugs configurado no pom.xml
- [ ] Semgrep com regras OWASP + pelo menos 3 regras customizadas
- [ ] 0 findings HIGH/CRITICAL no scan
- [ ] Relatório SAST gerado (HTML ou SARIF)
- [ ] Regra customizada para "no-float-for-money" implementada

---

### Desafio 4.5 — SCA — Software Composition Analysis

**Contexto:** A Digital Wallet usa dezenas de dependências (frameworks, drivers, bibliotecas).
Cada dependência pode ter CVEs (Common Vulnerabilities and Exposures) conhecidas que
expõem a aplicação a ataques.

**Requisitos:**

- **Go — govulncheck:**
  - Instalar e rodar `govulncheck ./...`
  - Corrigir ou mitigar vulnerabilidades encontradas
  - Integrar ao Makefile

```bash
# Makefile targets
security-scan:
	@echo "==> Running govulncheck..."
	govulncheck ./...
	@echo "==> Running gosec..."
	golangci-lint run --enable gosec

security-deps:
	@echo "==> Checking for outdated dependencies..."
	go list -m -u all | grep '\[' || echo "All dependencies up to date"
```

- **Java — OWASP Dependency-Check + Trivy:**

```xml
<!-- pom.xml — OWASP Dependency-Check -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>10.0.3</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS> <!-- Falha em HIGH+ -->
        <formats>
            <format>HTML</format>
            <format>JSON</format>
        </formats>
        <suppressionFile>owasp-suppressions.xml</suppressionFile>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

- Criar política de vulnerabilidades:

| Severidade | SLA de correção | Ação no CI |
|---|---|---|
| **Critical (CVSS ≥ 9.0)** | ≤ 24 horas | ❌ Bloqueia merge |
| **High (CVSS 7.0-8.9)** | ≤ 7 dias | ❌ Bloqueia merge |
| **Medium (CVSS 4.0-6.9)** | ≤ 30 dias | ⚠️ Warning |
| **Low (CVSS < 4.0)** | Próximo ciclo de manutenção | ℹ️ Info |

- Configurar suppression para falsos positivos (com justificativa documentada):

```xml
<!-- owasp-suppressions.xml -->
<suppressions>
    <suppress>
        <notes>Falso positivo: CVE-XXXX-YYYY não se aplica porque usamos a versão X
               que já contém o fix. Verificado em 2026-03-01.</notes>
        <cve>CVE-XXXX-YYYY</cve>
    </suppress>
</suppressions>
```

**Critérios de aceite:**

- [ ] `govulncheck` rodando sem CVEs HIGH/CRITICAL (Go)
- [ ] OWASP Dependency-Check configurado com failBuildOnCVSS=7 (Java)
- [ ] Política de SLA de vulnerabilidades documentada
- [ ] Suppressions documentadas com justificativa para cada falso positivo
- [ ] Makefile/pom.xml com targets de security scan
- [ ] Relatório SCA gerado (HTML + JSON)

---

### Desafio 4.6 — DAST — Dynamic Application Security Testing

**Contexto:** DAST testa a aplicação **em execução**, simulando ataques reais contra a API da
Digital Wallet. Diferente do SAST, DAST encontra problemas de runtime: CORS, headers, auth bypass.

**Requisitos:**

- Configurar **OWASP ZAP** para scan da API da Digital Wallet:
  - Fornecer o OpenAPI spec como target
  - Configurar autenticação (JWT) para ZAP acessar endpoints protegidos
  - Definir regras de alerta customizadas

```yaml
# .zap/rules.tsv — Customizar alertas ZAP
# ID    Action    Name
10038   FAIL      Content Security Policy Header Missing
10063   FAIL      X-Content-Type-Options Missing
10202   IGNORE    Absence of Anti-CSRF Tokens (API REST não usa CSRF)
40003   FAIL      CRLF Injection
40012   FAIL      Cross Site Scripting (Reflected)
40014   FAIL      Cross Site Scripting (Persistent)
40018   FAIL      SQL Injection
90001   FAIL      Insecure JSF ViewState (não aplicável)
```

- Implementar **testes de headers de segurança**:

**Java 25:**
```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class SecurityHeadersTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void apiResponses_shouldContainSecurityHeaders() {
        var response = restTemplate.getForEntity("/api/v1/health", String.class);

        assertThat(response.getHeaders())
                .satisfies(headers -> {
                    assertThat(headers.getFirst("X-Content-Type-Options"))
                            .isEqualTo("nosniff");
                    assertThat(headers.getFirst("X-Frame-Options"))
                            .isEqualTo("DENY");
                    assertThat(headers.getFirst("Cache-Control"))
                            .contains("no-store");
                    // Não deve expor tecnologia
                    assertThat(headers.getFirst("Server")).isNull();
                    assertThat(headers.getFirst("X-Powered-By")).isNull();
                });
    }

    @Test
    void errorResponses_shouldNotExposeStackTrace() {
        var response = restTemplate.getForEntity(
                "/api/v1/wallets/" + UUID.randomUUID(),
                String.class
        );

        assertThat(response.getBody())
                .doesNotContain("stackTrace", "at com.", "at org.",
                        "NullPointerException", "Exception");
    }
}
```

**Go 1.26:**
```go
func TestSecurityHeaders(t *testing.T) {
    app := setupTestApp(t)

    resp := app.GET(t, "/api/v1/health", "")

    assert.Equal(t, "nosniff", resp.Header.Get("X-Content-Type-Options"))
    assert.Equal(t, "DENY", resp.Header.Get("X-Frame-Options"))
    assert.Contains(t, resp.Header.Get("Cache-Control"), "no-store")
    assert.Empty(t, resp.Header.Get("Server"))
    assert.Empty(t, resp.Header.Get("X-Powered-By"))
}

func TestErrorResponse_NoStackTrace(t *testing.T) {
    app := setupTestApp(t)

    resp := app.GET(t, "/api/v1/wallets/"+uuid.NewString(), app.AdminToken)
    body, _ := io.ReadAll(resp.Body)

    assert.NotContains(t, string(body), "goroutine")
    assert.NotContains(t, string(body), "runtime.")
    assert.NotContains(t, string(body), "panic")
}
```

- Configurar ZAP scan no CI:

```yaml
# .github/workflows/security-dast.yml
security-dast:
  runs-on: ubuntu-latest
  needs: [deploy-test]
  steps:
    - uses: actions/checkout@v4
    - name: Start application
      run: docker compose -f docker-compose.test.yml up -d
    - name: Wait for app
      run: |
        for i in {1..30}; do
          curl -sf http://localhost:8080/api/v1/health && break
          sleep 2
        done
    - name: ZAP API Scan
      uses: zaproxy/action-api-scan@v0.7
      with:
        target: 'http://localhost:8080/v3/api-docs'
        rules_file_name: '.zap/rules.tsv'
        fail_action: true
    - name: Upload ZAP Report
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: zap-report
        path: zap-report.html
```

**Critérios de aceite:**

- [ ] OWASP ZAP configurado com OpenAPI spec como target
- [ ] Regras de alerta customizadas (`.zap/rules.tsv`)
- [ ] Testes de headers de segurança passando (Go + Java)
- [ ] Testes de error response sem stack trace
- [ ] Pipeline DAST no CI (GitHub Actions ou equivalente)
- [ ] 0 findings HIGH/CRITICAL no ZAP scan
- [ ] Relatório ZAP salvo como artifact

---

### Desafio 4.7 — Secret Scanning e Proteção de Dados Sensíveis

**Contexto:** A Digital Wallet lida com credenciais de banco, chaves de API de pagamento
e dados pessoais (CPF, email). Secrets hardcoded no código são uma das vulnerabilidades
mais comuns e mais perigosas.

**Requisitos:**

- Configurar **gitleaks** como pre-commit hook:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

- Configurar **regras customizadas** para domínio financeiro:

```toml
# .gitleaks.toml
title = "Digital Wallet Secret Scanning Rules"

[[rules]]
id = "database-connection-string"
description = "Database connection string with credentials"
regex = '''(?i)(jdbc|postgres|mysql):\/\/[^:]+:[^@]+@'''
tags = ["database", "credentials"]

[[rules]]
id = "jwt-secret"
description = "JWT signing secret"
regex = '''(?i)(jwt[._-]?secret|signing[._-]?key)\s*[=:]\s*['"][^'"]{16,}'''
tags = ["jwt", "authentication"]

[[rules]]
id = "api-key-payment"
description = "Payment provider API key"
regex = '''(?i)(stripe|paypal|pagseguro|mercadopago)[._-]?(api[._-]?key|secret)\s*[=:]\s*['"][^'"]+'''
tags = ["payment", "api-key"]

[allowlist]
paths = [
    '''.*_test\.go$''',
    '''.*Test\.java$''',
    '''testdata/.*''',
]
```

- Implementar **testes de proteção de dados sensíveis em logs**:

**Java 25:**
```java
@Test
void logs_shouldNotContainSensitiveData() {
    try (LogCaptor logCaptor = LogCaptor.forRoot()) {
        // Arrange — operação com dados sensíveis
        var user = createUser("sensitive@test.com", "12345678901");
        var wallet = createWallet(user.getId());
        deposit(wallet.getId(), "1000.00");

        // Act — coletar todos os logs
        String allLogs = String.join("\n", logCaptor.getLogs());

        // Assert — dados sensíveis NÃO devem aparecer nos logs
        assertThat(allLogs).doesNotContain("12345678901");        // CPF
        assertThat(allLogs).doesNotContain("sensitive@test.com"); // Email
        assertThat(allLogs)
                .doesNotContainPattern("\\b\\d{11}\\b");          // Qualquer CPF
    }
}
```

**Go 1.26:**
```go
func TestLogs_ShouldNotContainSensitiveData(t *testing.T) {
    // Arrange — capturar logs
    var logBuf bytes.Buffer
    logger := slog.New(slog.NewJSONHandler(&logBuf, nil))

    app := setupTestAppWithLogger(t, logger)
    user := app.CreateUser(t, "sensitive@test.com")
    wallet := app.CreateWallet(t, user.ID, user.Token)
    app.Deposit(t, wallet.ID, 1000.00, user.Token)

    // Assert — dados sensíveis NÃO nos logs
    logs := logBuf.String()
    assert.NotContains(t, logs, user.Document)         // CPF
    assert.NotContains(t, logs, "sensitive@test.com")   // Email
}
```

**Critérios de aceite:**

- [ ] gitleaks configurado como pre-commit hook
- [ ] Regras customizadas para domínio financeiro (`.gitleaks.toml`)
- [ ] Testes de log sem dados sensíveis (Go + Java)
- [ ] Secret scanning no CI (pipeline)
- [ ] 0 secrets detectados no código-fonte
- [ ] Documento: `decisions/04-secret-management.md`

---

## Definição de Pronto (DoD) — Security Testing

- [ ] Todos os testes de segurança passam (`go test`, `./mvnw test`)
- [ ] SAST: 0 findings HIGH/CRITICAL
- [ ] SCA: 0 CVEs HIGH/CRITICAL sem suppression justificada
- [ ] DAST: 0 findings HIGH/CRITICAL no ZAP
- [ ] Secret scanning: 0 secrets detectados
- [ ] Testes de autorização cobrem IDOR, privilege escalation, missing auth
- [ ] Testes de injection cobrem SQLi, XSS, mass assignment
- [ ] Headers de segurança validados
- [ ] Commit semântico: `feat(level-4): implement security testing suite`

---

## Checklist

- [ ] Threat model (STRIDE) documentado
- [ ] Testes de autorização implementados (IDOR, escalation, horizontal)
- [ ] Testes de injection implementados (SQLi, XSS, mass assignment)
- [ ] SAST configurado e passando (gosec, SpotBugs/Semgrep)
- [ ] SCA configurado e passando (govulncheck, OWASP Dependency-Check)
- [ ] DAST configurado (ZAP) com scan automatizado
- [ ] Secret scanning configurado (gitleaks)
- [ ] Testes de headers de segurança passando
- [ ] Pipeline de segurança completo no CI
- [ ] Relatórios de segurança gerados e arquivados
