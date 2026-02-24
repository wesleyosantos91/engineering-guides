# Security Testing — Guia Completo

> **Versão:** 1.0
> **Última atualização:** 2026-02-24
> **Público-alvo:** Times de engenharia (back-end, front-end, plataforma, DevSecOps, QA)
> **Documentos relacionados:** [Testing Strategy](testing-strategy.md) · [Code Review & Quality](code-review-quality.md)

---

## Sumário

1. [Visão Geral](#1-visão-geral)
2. [Tipos de Teste de Segurança](#2-tipos-de-teste-de-segurança)
3. [OWASP Top 10 — O que Testar](#3-owasp-top-10--o-que-testar)
4. [SAST — Static Application Security Testing](#4-sast--static-application-security-testing)
5. [DAST — Dynamic Application Security Testing](#5-dast--dynamic-application-security-testing)
6. [SCA — Software Composition Analysis](#6-sca--software-composition-analysis)
7. [Testes de Segurança no CI/CD](#7-testes-de-segurança-no-cicd)
8. [Segurança em APIs](#8-segurança-em-apis)
9. [Segurança em Containers e Infraestrutura](#9-segurança-em-containers-e-infraestrutura)
10. [Threat Modeling](#10-threat-modeling)
11. [Padrões e Receitas](#11-padrões-e-receitas)
12. [Anti-Patterns](#12-anti-patterns)
13. [Glossário](#13-glossário)

---

## 1. Visão Geral

### 1.1 Security Shift-Left

Segurança não é fase — é propriedade do sistema. Deve ser integrada em **todas** as etapas:

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐
│  Design  │→ │  Código  │→ │  Build   │→ │  Deploy  │→ │ Produção  │
│          │  │          │  │          │  │          │  │           │
│ Threat   │  │ SAST     │  │ SCA      │  │ DAST     │  │ WAF       │
│ Modeling │  │ Linting  │  │ Image    │  │ Pentest  │  │ RASP      │
│ Sec Reqs │  │ Secrets  │  │ Scanning │  │          │  │ Monitoring│
└──────────┘  └──────────┘  └──────────┘  └──────────┘  └───────────┘
```

### 1.2 Princípios

| #  | Princípio                       | Descrição                                                                     |
|----|---------------------------------|-------------------------------------------------------------------------------|
| S1 | **Defense in depth**            | Múltiplas camadas de proteção — não confiar em um único controle              |
| S2 | **Principle of least privilege**| Mínimo de permissões necessárias para cada componente/usuário                 |
| S3 | **Fail secure**                 | Em caso de falha, sistema deve negar acesso, não permitir                     |
| S4 | **Zero trust**                  | Nunca confiar, sempre verificar — mesmo entre serviços internos               |
| S5 | **Secure by default**           | Configuração padrão deve ser segura; opt-in para relaxar, não opt-out         |
| S6 | **Automação**                   | Testes de segurança devem ser automatizados e integrados ao pipeline          |
| S7 | **Transparência**               | Vulnerabilidades encontradas devem ser rastreáveis e ter responsável atribuído |

---

## 2. Tipos de Teste de Segurança

| Tipo                     | O que faz                                              | Quando rodar            | Custo   |
|--------------------------|--------------------------------------------------------|-------------------------|---------|
| **SAST**                 | Análise estática do código-fonte                       | Todo commit/PR          | Baixo   |
| **SCA**                  | Verifica vulnerabilidades em dependências               | Todo commit/PR          | Baixo   |
| **Secret Scanning**      | Detecta credenciais hardcoded no código                | Pré-commit + PR         | Baixo   |
| **DAST**                 | Testa a aplicação em execução buscando vulnerabilidades | Staging, pré-release    | Médio   |
| **IAST**                 | Análise em tempo de execução (instrumentação)          | Staging, QA             | Médio   |
| **Container Scanning**   | Verifica vulnerabilidades em imagens Docker             | Build, pré-deploy       | Baixo   |
| **IaC Scanning**         | Verifica configurações inseguras em Terraform/K8s      | PR + merge              | Baixo   |
| **Pentest**              | Teste manual/semi-automático de penetração              | Trimestral ou major     | Alto    |
| **Bug Bounty**           | Pesquisadores externos buscam vulnerabilidades          | Contínuo                | Variável|

---

## 3. OWASP Top 10 — O que Testar

### 3.1 Mapeamento de Testes

| #  | Vulnerabilidade                      | O que testar                                                    | Tipo de teste     |
|----|--------------------------------------|-----------------------------------------------------------------|-------------------|
| A1 | **Broken Access Control**            | Acesso a recursos de outros usuários, privilege escalation      | DAST + Unitário   |
| A2 | **Cryptographic Failures**           | Dados sensíveis em trânsito/repouso sem criptografia adequada   | SAST + Review     |
| A3 | **Injection**                        | SQL, NoSQL, OS, LDAP injection em todos os inputs               | SAST + DAST       |
| A4 | **Insecure Design**                  | Falhas arquiteturais, falta de rate limiting, validação fraca   | Threat Modeling   |
| A5 | **Security Misconfiguration**        | Headers de segurança, CORS, stack traces em erro, defaults      | DAST + IaC Scan   |
| A6 | **Vulnerable Components**            | Dependências com CVEs conhecidas                                 | SCA               |
| A7 | **Auth Failures**                    | Brute force, session fixation, tokens fracos                    | DAST + Unitário   |
| A8 | **Software & Data Integrity**        | CI/CD pipeline poisoning, desserialização insegura              | SAST + IaC Scan   |
| A9 | **Logging & Monitoring Failures**    | Ausência de logs de segurança, alertas inadequados              | Review + Checklist|
| A10| **SSRF**                             | Server-Side Request Forgery via inputs de URL                   | SAST + DAST       |

### 3.2 Exemplos de Testes por Vulnerabilidade

#### Broken Access Control (teste unitário)

```java
@Test
void acessarPedido_deOutroUsuario_retorna403() {
    // Arrange
    var pedidoDoUsuarioA = criarPedido(dono: "user-A");
    var contexto = autenticarComo("user-B");

    // Act + Assert
    assertThrows(AccessDeniedException.class, () ->
        pedidoService.buscar(pedidoDoUsuarioA.getId(), contexto)
    );
}

@Test
void usuarioComum_tentaAcessarEndpointAdmin_retorna403() {
    // Arrange
    var contexto = autenticarComo("user-comum", roles: ["USER"]);

    // Act + Assert
    assertThrows(AccessDeniedException.class, () ->
        adminService.listarTodosUsuarios(contexto)
    );
}
```

#### SQL Injection (teste de integração)

```java
@Test
void buscarPorNome_comInputMalicioso_naoExecutaInjection() {
    // Arrange
    var inputMalicioso = "'; DROP TABLE pedidos; --";

    // Act — não deve lançar exceção nem alterar o banco
    var resultado = pedidoRepository.buscarPorNome(inputMalicioso);

    // Assert
    assertThat(resultado).isEmpty();  // retorna vazio, não erro
    // Verificar que a tabela ainda existe
    assertThat(pedidoRepository.count()).isGreaterThanOrEqualTo(0);
}
```

---

## 4. SAST — Static Application Security Testing

### 4.1 Ferramentas

| Ferramenta      | Linguagens                    | Destaques                                            |
|-----------------|-------------------------------|------------------------------------------------------|
| **SonarQube**   | 30+ linguagens                | Amplo, code quality + security, IDE integration      |
| **Semgrep**     | 30+ linguagens                | Regras customizáveis, rápido, open-source            |
| **CodeQL**      | Java, JS/TS, Python, Go, C++ | GitHub-native, queries poderosas, community rules    |
| **Snyk Code**   | Java, JS/TS, Python, Go      | Análise em tempo real, boa integração com IDEs       |
| **Checkmarx**   | 25+ linguagens                | Enterprise, compliance reports                        |

### 4.2 Configuração no CI/CD

```yaml
# GitHub Actions — Semgrep
security-sast:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: returntocorp/semgrep-action@v1
      with:
        config: >-
          p/owasp-top-ten
          p/java
          p/typescript
      env:
        SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}
```

### 4.3 Regras Customizadas (Semgrep)

```yaml
# Detectar uso de System.exit() em produção
rules:
  - id: no-system-exit
    patterns:
      - pattern: System.exit(...)
    message: "System.exit() não deve ser usado em código de produção"
    severity: ERROR
    languages: [java]

  - id: no-hardcoded-secrets
    patterns:
      - pattern: |
          $VAR = "...$SECRET..."
      - metavariable-regex:
          metavariable: $SECRET
          regex: "(password|secret|api.?key|token).*=.*['\"][^'\"]{8,}"
    message: "Possível credencial hardcoded detectada"
    severity: ERROR
    languages: [java, python, javascript, typescript]
```

---

## 5. DAST — Dynamic Application Security Testing

### 5.1 Ferramentas

| Ferramenta        | Tipo           | Destaques                                               |
|-------------------|----------------|----------------------------------------------------------|
| **OWASP ZAP**     | Open-source    | Full-featured, API scanning, CI/CD friendly              |
| **Burp Suite**    | Comercial      | Pentest profissional, extensível, scanner automático      |
| **Nuclei**        | Open-source    | Templates de vulnerabilidades, rápido, community-driven  |
| **Nikto**         | Open-source    | Scanner web clássico, detecta misconfigs                 |

### 5.2 OWASP ZAP no CI/CD

```yaml
# GitHub Actions — ZAP API Scan
security-dast:
  runs-on: ubuntu-latest
  needs: [deploy-staging]
  steps:
    - uses: actions/checkout@v4
    - name: ZAP API Scan
      uses: zaproxy/action-api-scan@v0.7
      with:
        target: 'https://api.staging.example.com/openapi.json'
        rules_file_name: '.zap/rules.tsv'
        fail_action: true  # falha pipeline se encontrar HIGH/CRITICAL
        cmd_options: '-a'  # incluir alertas alfa
```

### 5.3 O que DAST Encontra

| Categoria                    | Exemplos                                                    |
|------------------------------|-------------------------------------------------------------|
| **Injection**                | SQL, XSS, Command injection em inputs reais                |
| **Auth problems**            | Session não expira, tokens previsíveis, brute-force possível |
| **Misconfiguration**         | CORS permissivo, headers ausentes, TLS fraco               |
| **Information disclosure**   | Stack traces, versões de software, paths internos           |
| **Business logic**           | Rate limiting ausente, mass assignment, IDOR                |

---

## 6. SCA — Software Composition Analysis

### 6.1 Ferramentas

| Ferramenta          | Destaques                                                     |
|---------------------|---------------------------------------------------------------|
| **Snyk**            | Base de vulnerabilidades ampla, fix automático, PRs de correção |
| **Dependabot**      | GitHub-native, PRs de atualização automáticas                  |
| **Trivy**           | Open-source, rápido, cobre deps + containers + IaC            |
| **OWASP Dependency-Check** | Open-source, NVD-based, suporta Java/JS/Python/.NET    |
| **Renovate**        | Atualizações automáticas com configuração granular             |

### 6.2 Política de Vulnerabilidades

| Severidade    | SLA de correção      | Ação no pipeline      |
|---------------|----------------------|-----------------------|
| **Critical**  | ≤ 24 horas           | Bloqueia merge        |
| **High**      | ≤ 7 dias             | Bloqueia merge        |
| **Medium**    | ≤ 30 dias            | Warning (não bloqueia)|
| **Low**       | Próximo ciclo de manutenção | Info               |

### 6.3 Configuração no CI/CD

```yaml
# GitHub Actions — Trivy SCA
security-sca:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - name: Trivy vulnerability scan
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'  # falha pipeline se encontrar HIGH+
        format: 'sarif'
        output: 'trivy-results.sarif'
    - uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: 'trivy-results.sarif'
```

---

## 7. Testes de Segurança no CI/CD

### 7.1 Pipeline Integrado

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     SECURITY PIPELINE                                    │
├─────────────┬───────────────────────────────────────────────────────────┤
│             │                                                           │
│  Pré-commit │  ┌──────────────┐  ┌──────────────┐                      │
│             │  │ Secret scan  │  │ Lint rules   │                      │
│             │  │ (gitleaks)   │  │ (security)   │                      │
│             │  └──────────────┘  └──────────────┘                      │
├─────────────┼───────────────────────────────────────────────────────────┤
│             │                                                           │
│  PR         │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│             │  │ SAST     │  │ SCA      │  │ IaC Scan │  │ License  │ │
│             │  │ Semgrep  │  │ Trivy    │  │ Checkov  │  │ check    │ │
│             │  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
│             │       ↓ GATE: 0 HIGH/CRITICAL findings                   │
├─────────────┼───────────────────────────────────────────────────────────┤
│             │                                                           │
│  Build      │  ┌──────────────┐  ┌──────────────┐                      │
│             │  │ Container    │  │ SBOM         │                      │
│             │  │ image scan   │  │ generation   │                      │
│             │  └──────────────┘  └──────────────┘                      │
├─────────────┼───────────────────────────────────────────────────────────┤
│             │                                                           │
│  Staging    │  ┌──────────────┐  ┌──────────────┐                      │
│             │  │ DAST         │  │ API Security │                      │
│             │  │ (ZAP)        │  │ scan         │                      │
│             │  └──────────────┘  └──────────────┘                      │
├─────────────┼───────────────────────────────────────────────────────────┤
│             │                                                           │
│  Trimestral │  ┌──────────────┐  ┌──────────────┐                      │
│             │  │ Pentest      │  │ Threat model │                      │
│             │  │ (manual)     │  │ review       │                      │
│             │  └──────────────┘  └──────────────┘                      │
└─────────────┴───────────────────────────────────────────────────────────┘
```

### 7.2 Secret Scanning

```yaml
# .pre-commit-config.yaml — gitleaks
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

**O que detecta:**
- API keys, tokens, passwords hardcoded
- Connection strings com credenciais
- Private keys (RSA, SSH, PGP)
- Cloud provider credentials (AWS, GCP, Azure)

---

## 8. Segurança em APIs

### 8.1 Checklist de Segurança para APIs

| Item                           | Descrição                                                            | Teste               |
|--------------------------------|----------------------------------------------------------------------|----------------------|
| **Autenticação**               | Todos os endpoints (exceto públicos) requerem auth válida            | Integração           |
| **Autorização**                | Verificar RBAC/ABAC — usuário só acessa seus recursos                | Unitário + Integração|
| **Rate Limiting**              | Endpoints protegidos contra abuso (brute-force, DDoS)                | DAST + Load test     |
| **Input Validation**           | Todos os inputs validados (tipo, tamanho, formato, range)            | Unitário + DAST      |
| **Output Encoding**            | Respostas sanitizadas contra XSS e information disclosure            | DAST                 |
| **CORS**                       | Origins permitidos são explícitos e mínimos                          | DAST                 |
| **Security Headers**           | HSTS, CSP, X-Content-Type-Options, X-Frame-Options configurados     | DAST                 |
| **TLS**                        | Mínimo TLS 1.2, ciphers fortes                                      | DAST                 |
| **Logging**                    | Tentativas de acesso inválido são logadas (sem dados sensíveis)      | Review + Integração  |
| **Error Handling**             | Erros não revelam stack trace, versões ou paths internos             | DAST                 |
| **Mass Assignment**            | Apenas campos permitidos são aceitos no payload                      | Unitário             |
| **IDOR (Insecure Direct Obj)** | IDs de recursos validados contra o contexto do usuário               | Integração           |

### 8.2 Exemplo — Headers de Segurança (teste)

```java
@Test
void respostaDeveConterHeadersDeSeguranca() {
    var response = restClient.get("/api/v1/health");

    assertThat(response.getHeader("Strict-Transport-Security"))
        .isEqualTo("max-age=31536000; includeSubDomains");
    assertThat(response.getHeader("X-Content-Type-Options"))
        .isEqualTo("nosniff");
    assertThat(response.getHeader("X-Frame-Options"))
        .isEqualTo("DENY");
    assertThat(response.getHeader("Content-Security-Policy"))
        .isNotBlank();
    // Não deve expor technologia
    assertThat(response.getHeader("Server")).isNull();
    assertThat(response.getHeader("X-Powered-By")).isNull();
}
```

---

## 9. Segurança em Containers e Infraestrutura

### 9.1 Container Image Scanning

```yaml
# GitHub Actions — Trivy container scan
container-scan:
  runs-on: ubuntu-latest
  steps:
    - name: Build image
      run: docker build -t myapp:${{ github.sha }} .
    - name: Trivy image scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myapp:${{ github.sha }}'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'
```

### 9.2 Dockerfile Security Best Practices

| Prática                                | Descrição                                                        |
|----------------------------------------|------------------------------------------------------------------|
| **Imagem base mínima**                 | Usar `distroless`, `alpine` ou `slim` — menor superfície de ataque |
| **Non-root user**                      | Nunca rodar como root — `USER nonroot:nonroot`                   |
| **Multi-stage build**                  | Não incluir ferramentas de build na imagem final                 |
| **Pin versions**                       | Fixar versão de imagem base e dependências                       |
| **No secrets in image**                | Credenciais via env vars ou secret manager, nunca na imagem      |
| **COPY específico**                    | Copiar apenas o necessário, não o diretório inteiro              |
| **Read-only filesystem**               | Montar filesystem como read-only onde possível                   |
| **Health check**                       | Incluir HEALTHCHECK para verificação de liveness                 |

### 9.3 IaC Security Scanning

| Ferramenta      | O que verifica                                                       |
|-----------------|----------------------------------------------------------------------|
| **Checkov**     | Terraform, CloudFormation, K8s manifests, Dockerfile, ARM            |
| **tfsec**       | Terraform-specific, regras detalhadas para AWS/GCP/Azure             |
| **Trivy**       | Terraform, CloudFormation, Dockerfile, Kubernetes                     |
| **KICS**        | Multi-plataforma IaC scanner                                          |

---

## 10. Threat Modeling

### 10.1 Processo STRIDE

| Ameaça                      | Descrição                                          | Exemplo                                            |
|-----------------------------|----------------------------------------------------|----------------------------------------------------|
| **S** — Spoofing            | Fingir ser outra entidade                          | Roubo de sessão, token forjado                     |
| **T** — Tampering           | Modificar dados em trânsito ou em repouso          | Alterar payload, manipular parâmetros de URL        |
| **R** — Repudiation         | Negar ter realizado uma ação                       | Ausência de audit logs                              |
| **I** — Info Disclosure     | Acesso não autorizado a informações                | Logs com dados sensíveis, error leaking             |
| **D** — Denial of Service   | Tornar sistema indisponível                        | Request flooding, resource exhaustion               |
| **E** — Elevation of Priv   | Obter permissões acima do concedido                | Escalação de role, bypass de autorização            |

### 10.2 Quando fazer Threat Modeling

| Trigger                                        | Tipo de revisão                   |
|------------------------------------------------|-----------------------------------|
| Novo serviço/sistema                           | Threat model completo             |
| Nova integração com sistema externo            | Revisão incremental               |
| Alteração em fluxo de autenticação/autorização | Revisão focada                    |
| Incidente de segurança                         | Revisão post-mortem + atualização |
| Trimestral (periódica)                         | Revisão geral                     |

### 10.3 Template de Threat Model

```markdown
## Threat Model — [Nome do Componente]

### Escopo
- Componentes envolvidos: [...]
- Dados processados: [...]
- Atores: [...]

### Diagrama de Fluxo de Dados (DFD)
[Desenhar DFD com trust boundaries]

### Ameaças Identificadas (STRIDE)
| ID  | Ameaça                | Impacto | Probabilidade | Risco  | Mitigação                    | Status     |
|-----|----------------------|---------|---------------|--------|------------------------------|------------|
| T1  | SQL Injection em /search | Alto | Média         | Alto   | Parameterized queries         | Mitigado   |
| T2  | IDOR em /orders/{id}    | Alto  | Alta          | Crítico| Validação de ownership        | Mitigado   |
| T3  | DDoS no endpoint público | Médio | Alta         | Alto   | Rate limiting + WAF           | Em progresso|
```

---

## 11. Padrões e Receitas

### 11.1 Input Validation Pattern

```java
// Sempre validar no ponto de entrada (controller/handler)
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(
    @Valid @RequestBody CreateOrderRequest request  // Bean Validation
) {
    // @Valid garante que todas as annotations de validação são verificadas
    // antes de chegar ao service
}

public record CreateOrderRequest(
    @NotBlank @Size(max = 50) String customerId,
    @NotEmpty @Size(max = 100) List<@Valid OrderItemRequest> items,
    @NotNull @Future LocalDateTime deliveryDate
) {}

public record OrderItemRequest(
    @NotBlank @Pattern(regexp = "^[A-Z]{2,10}-\\d{3,6}$") String sku,
    @Min(1) @Max(9999) int quantity,
    @Positive @Digits(integer = 10, fraction = 2) BigDecimal unitPrice
) {}
```

### 11.2 Output Sanitization

```java
// Nunca retornar entidades internas diretamente
// Sempre usar DTOs/Records que expõem apenas campos necessários

// ❌ ERRADO — expõe dados internos
@GetMapping("/users/{id}")
public User getUser(@PathVariable String id) {
    return userRepository.findById(id); // expõe passwordHash, internalId, etc.
}

// ✅ CORRETO — DTO controlado
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable String id) {
    var user = userRepository.findById(id);
    return new UserResponse(user.getName(), user.getEmail(), user.getRole());
}
```

### 11.3 Secure Error Handling

```java
// ❌ ERRADO — expõe internals
@ExceptionHandler(Exception.class)
public ResponseEntity<String> handleError(Exception ex) {
    return ResponseEntity.status(500).body(ex.getMessage() + "\n" + ex.getStackTrace());
}

// ✅ CORRETO — erro genérico para o cliente, log detalhado internamente
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleError(Exception ex) {
    var errorId = UUID.randomUUID().toString();
    log.error("Unexpected error [errorId={}]", errorId, ex);  // log interno com detalhes

    return ResponseEntity.status(500).body(
        new ErrorResponse(errorId, "Internal server error")  // cliente recebe apenas ID
    );
}
```

---

## 12. Anti-Patterns

| Anti-pattern                                    | Problema                                           | Solução                                             |
|-------------------------------------------------|----------------------------------------------------|-----------------------------------------------------|
| Security como última fase                       | Vulnerabilidades descobertas tarde, caras de corrigir | Shift-left: SAST/SCA em todo PR                   |
| Ignorar dependências desatualizadas             | CVEs acumulam, superfície de ataque cresce          | Dependabot/Renovate + SLA de correção              |
| Mesmo secret em todos os ambientes              | Comprometer um ambiente compromete todos            | Secrets separados por ambiente, rotação periódica  |
| Logs com dados sensíveis (CPF, cartão, senha)   | Violação de LGPD/GDPR, exposure via log aggregator | Mascaramento obrigatório, audit de logs            |
| CORS com `*` (allow all origins)                | Qualquer site pode fazer requests à API             | Allow-list explícita de origins                    |
| Trust headers sem validação (`X-Forwarded-For`) | Spoofing de IP, bypass de rate limiting             | Configurar trusted proxies corretamente            |
| Desabilitar scanning "porque dá falso positivo" | Vulnerabilidades reais ficam sem detecção           | Tuning de regras, suppress específico com justificativa |
| Pentest apenas uma vez                          | Sistema muda, novas vulnerabilidades surgem         | Pentest trimestral + scanning contínuo             |

---

## 13. Glossário

| Termo                    | Definição                                                                                         |
|--------------------------|---------------------------------------------------------------------------------------------------|
| **CVE**                  | Common Vulnerabilities and Exposures — identificador único para vulnerabilidades públicas.         |
| **DAST**                 | Dynamic Application Security Testing — teste da aplicação em execução.                            |
| **IAST**                 | Interactive Application Security Testing — análise em tempo de execução via instrumentação.        |
| **IDOR**                 | Insecure Direct Object Reference — acesso direto a recursos de outros usuários via ID.             |
| **NVD**                  | National Vulnerability Database — banco de dados público de vulnerabilidades (NIST).               |
| **OWASP**                | Open Worldwide Application Security Project — comunidade de segurança com guias e ferramentas.     |
| **RASP**                 | Runtime Application Self-Protection — proteção em tempo de execução dentro da aplicação.           |
| **SARIF**                | Static Analysis Results Interchange Format — formato padrão para resultados de análise estática.   |
| **SAST**                 | Static Application Security Testing — análise do código-fonte sem execução.                        |
| **SBOM**                 | Software Bill of Materials — inventário de todas as dependências do software.                      |
| **SCA**                  | Software Composition Analysis — análise de vulnerabilidades em dependências de terceiros.           |
| **SSRF**                 | Server-Side Request Forgery — ataque que faz o servidor realizar requests para alvos internos.     |
| **STRIDE**               | Modelo de categorização de ameaças: Spoofing, Tampering, Repudiation, Info Disclosure, DoS, EoP.  |
| **WAF**                  | Web Application Firewall — firewall especializado em proteger aplicações web.                      |
| **XSS**                  | Cross-Site Scripting — injeção de scripts maliciosos em páginas web.                               |
| **Zero Trust**           | Modelo de segurança que não confia em nenhum ator por padrão, verificando tudo explicitamente.     |
