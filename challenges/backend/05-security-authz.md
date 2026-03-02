# Level 5 — Segurança e Autorização

> **Objetivo:** Implementar autenticação JWT, autorização baseada em roles (RBAC), proteção de endpoints e boas práticas de segurança em APIs REST.

---

## Objetivo de Aprendizado

- Implementar autenticação via JWT (JSON Web Token)
- Implementar autorização baseada em roles (RBAC)
- Proteger endpoints com middleware/filter de segurança
- Implementar login e registro de usuários
- Aplicar rate limiting para proteção contra abuso
- Configurar CORS adequadamente
- Entender OAuth2 Resource Server pattern
- Implementar refresh token

---

## Escopo Funcional

### Novos Endpoints

```
── Auth ──
POST /api/v1/auth/register          → Registro (201) — cria user + credentials
POST /api/v1/auth/login              → Login (200) — retorna access_token + refresh_token
POST /api/v1/auth/refresh            → Refresh token (200) — retorna novo access_token
POST /api/v1/auth/logout             → Logout (204) — invalida refresh_token

── Admin ──
GET    /api/v1/admin/users           → Listar todos os usuários (role: ADMIN)
DELETE /api/v1/admin/users/{id}      → Desativar usuário (role: ADMIN)
```

### Regras de Autorização (RBAC)

| Endpoint | Role necessária |
|---|---|
| `POST /auth/register` | Público |
| `POST /auth/login` | Público |
| `POST /auth/refresh` | Público (com refresh_token) |
| `POST /auth/logout` | Autenticado |
| `GET /users/{id}` | Autenticado + próprio user OU ADMIN |
| `PUT /users/{id}` | Autenticado + próprio user |
| `GET /wallets/*` | Autenticado + dono da wallet |
| `POST /wallets/*/transactions/*` | Autenticado + dono da wallet |
| `GET /admin/*` | ADMIN |
| `GET /health/*` | Público |
| `GET /metrics` | Público (ou protegido em prod) |

### Token Spec

```json
// Access Token Payload
{
  "sub": "uuid-do-usuario",
  "email": "user@example.com",
  "roles": ["USER"],
  "iat": 1700000000,
  "exp": 1700003600    // 1 hora
}

// Refresh Token — opaco (UUID armazenado no banco)
```

### Regras de Negócio

1. **Access token** expira em 1 hora
2. **Refresh token** expira em 7 dias, armazenado no banco
3. **Password** armazenado com bcrypt (cost ≥ 12)
4. Usuário só acessa **seus próprios** recursos (exceto ADMIN)
5. **Rate limiting**: 100 requests/minuto por IP; 5 tentativas de login/minuto por email
6. Tokens inválidos/expirados retornam 401 com ProblemDetail
7. Acesso negado retorna 403 com ProblemDetail

---

## Escopo Técnico

### Autenticação por Stack

| Stack | Framework JWT | Hash | Filter/Middleware |
|---|---|---|---|
| **Go (Gin)** | golang-jwt/jwt | bcrypt (golang.org/x/crypto) | `gin.HandlerFunc` middleware |
| **Spring Boot** | Spring Security OAuth2 Resource Server | `BCryptPasswordEncoder` | `SecurityFilterChain` |
| **Quarkus** | SmallRye JWT | bcrypt | `@RolesAllowed`, `SecurityIdentity` |
| **Micronaut** | Micronaut Security JWT | bcrypt | `@Secured`, `SecurityRule` |
| **Jakarta EE** | MicroProfile JWT | bcrypt | `@RolesAllowed`, `JsonWebToken` |

### Schema Adicional

```sql
-- V4__create_credentials.sql
CREATE TABLE credentials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL UNIQUE REFERENCES users(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    roles           VARCHAR(100) NOT NULL DEFAULT 'USER',  -- USER,ADMIN
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- V5__create_refresh_tokens.sql
CREATE TABLE refresh_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    token           UUID NOT NULL UNIQUE,
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked         BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_refresh_tokens_token ON refresh_tokens(token);
CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
```

---

## Critérios de Aceite

- [ ] Registro cria user + credentials com senha hasheada (bcrypt)
- [ ] Login com credenciais válidas retorna access_token + refresh_token
- [ ] Login com credenciais inválidas retorna 401 ProblemDetail
- [ ] Access token JWT contém sub, email, roles, exp
- [ ] Endpoints protegidos retornam 401 sem token
- [ ] Endpoints protegidos retornam 401 com token expirado
- [ ] Endpoints protegidos retornam 403 com role insuficiente
- [ ] Usuário não-ADMIN só acessa seus próprios recursos
- [ ] Refresh token gera novo access_token
- [ ] Logout invalida refresh_token
- [ ] Rate limiting bloqueia após limite (429 Too Many Requests)
- [ ] CORS configurado para origens permitidas
- [ ] Password nunca aparece em logs, responses ou traces

---

## Definição de Pronto (DoD)

- [ ] 2 migrações adicionais (credentials + refresh_tokens)
- [ ] Testes de integração para todos os fluxos de auth
- [ ] Testes para autorização: acesso próprio, acesso cruzado (403), admin
- [ ] Rate limiting testado
- [ ] Secret JWT configurável via env var (nunca hardcoded)
- [ ] `Authorization: Bearer <token>` em todos os endpoints protegidos
- [ ] Swagger/OpenAPI com security scheme JWT
- [ ] Commit: `feat(level-5): add JWT authentication and RBAC authorization`

---

## Checklist

### Go (Gin)

- [ ] `internal/handler/auth.go` — handlers `Register`, `Login`, `Refresh`, `Logout`
- [ ] `internal/service/auth.go` — lógica de autenticação/registro
- [ ] `internal/service/token.go` — geração/validação JWT com golang-jwt
- [ ] `internal/store/credential.go` — CRUD de credenciais
- [ ] `internal/store/refresh_token.go` — CRUD de refresh tokens
- [ ] `internal/model/auth.go` — `RegisterInput`, `LoginInput`, `TokenPair`, `Claims`
- [ ] `internal/middleware/auth.go` — extrai JWT do header, valida, injeta claims no context
- [ ] `internal/middleware/authorize.go` — verifica roles requeridos
- [ ] `internal/middleware/ratelimit.go` — rate limiting por IP (golang.org/x/time/rate)
- [ ] `internal/middleware/cors.go` — CORS config
- [ ] `internal/platform/security/bcrypt.go` — hash/compare de senha

### Java — Spring Boot

- [ ] `pom.xml` — `spring-boot-starter-oauth2-resource-server`, `spring-security-crypto`
- [ ] `core/config/SecurityConfig.java` — `SecurityFilterChain` bean
- [ ] `core/security/JwtTokenProvider.java` — geração/validação de tokens
- [ ] `core/security/JwtAuthenticationFilter.java` — filtro que extrai e valida JWT
- [ ] `core/security/UserPrincipal.java` — record com dados do usuário autenticado
- [ ] `api/rest/v1/controller/AuthController.java` — registro, login, refresh, logout
- [ ] `domain/service/AuthService.java` — lógica de autenticação
- [ ] `domain/service/TokenService.java` — geração e validação JWT
- [ ] `infrastructure/entity/CredentialEntity.java`
- [ ] `infrastructure/entity/RefreshTokenEntity.java`
- [ ] `infrastructure/jpa/CredentialJpaRepository.java`
- [ ] `infrastructure/jpa/RefreshTokenJpaRepository.java`
- [ ] `api/exception/AccessDeniedException.java` → 403
- [ ] `api/exception/UnauthorizedException.java` → 401
- [ ] `application.yml` — `jwt.secret`, `jwt.expiration`, `jwt.refresh-expiration`

### Quarkus

- [ ] `@RolesAllowed("USER")` / `@RolesAllowed("ADMIN")` nos resources
- [ ] `SecurityIdentity` injetado para obter claims do JWT atual
- [ ] `@PermitAll` para endpoints públicos
- [ ] `mp.jwt.verify.publickey.location` / `smallrye.jwt.*` em `application.properties`
- [ ] Custom `IdentityProvider` se não usar Keycloak

### Micronaut

- [ ] `@Secured(SecurityRule.IS_AUTHENTICATED)` nos controllers
- [ ] `@Secured({"ADMIN"})` para endpoints admin
- [ ] `micronaut.security.token.jwt.signatures.secret.*` em `application.yml`
- [ ] `AuthenticationProvider` customizado para login
- [ ] `TokenRenderer` para formato de resposta customizado

### Jakarta EE

- [ ] `@RolesAllowed("USER")` / `@RolesAllowed("ADMIN")` nos resources
- [ ] `@Inject JsonWebToken jwt` para acessar claims
- [ ] `mp.jwt.verify.publickey` / `mp.jwt.verify.issuer` em `microprofile-config.properties`
- [ ] `IdentityStore` customizado (opcional, se não usar Keycloak)

---

## Tarefas Sugeridas por Stack

### Go — Middleware de Auth

```go
// internal/middleware/auth.go
func Auth(tokenService *service.TokenService) gin.HandlerFunc {
    return func(c *gin.Context) {
        header := c.GetHeader("Authorization")
        if header == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized,
                httperr.ProblemDetail{
                    Type:   "UNAUTHORIZED",
                    Title:  "Missing authorization",
                    Status: 401,
                    Detail: "Authorization header is required",
                })
            return
        }

        token := strings.TrimPrefix(header, "Bearer ")
        claims, err := tokenService.ValidateAccessToken(token)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized,
                httperr.ProblemDetail{
                    Type:   "UNAUTHORIZED",
                    Title:  "Invalid token",
                    Status: 401,
                    Detail: err.Error(),
                })
            return
        }

        c.Set("claims", claims)
        c.Set("userId", claims.Subject)
        c.Next()
    }
}

// internal/middleware/authorize.go
func RequireRole(roles ...string) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims, _ := c.Get("claims")
        userClaims := claims.(*model.Claims)

        for _, required := range roles {
            for _, userRole := range userClaims.Roles {
                if userRole == required {
                    c.Next()
                    return
                }
            }
        }

        c.AbortWithStatusJSON(http.StatusForbidden,
            httperr.ProblemDetail{
                Type:   "FORBIDDEN",
                Title:  "Access denied",
                Status: 403,
                Detail: "Insufficient permissions",
            })
    }
}
```

### Spring Boot — SecurityFilterChain

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http,
                                            JwtAuthenticationFilter jwtFilter) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/health/**", "/metrics/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/v1/**").authenticated()
                .anyRequest().denyAll()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, e) -> {
                    res.setStatus(401);
                    res.setContentType("application/problem+json");
                    res.getWriter().write("""
                        {"type":"UNAUTHORIZED","title":"Authentication required","status":401}
                    """);
                })
                .accessDeniedHandler((req, res, e) -> {
                    res.setStatus(403);
                    res.setContentType("application/problem+json");
                    res.getWriter().write("""
                        {"type":"FORBIDDEN","title":"Access denied","status":403}
                    """);
                })
            )
            .build();
    }
}
```

### Quarkus — JAX-RS Resource com RBAC

```java
@Path("/api/v1/admin/users")
@ApplicationScoped
@RolesAllowed("ADMIN")
public class AdminUserResource {

    @Inject
    SecurityIdentity identity;

    @Inject
    UserService userService;

    @GET
    public Response listAll(@QueryParam("page") @DefaultValue("0") int page,
                            @QueryParam("size") @DefaultValue("20") int size) {
        return Response.ok(userService.findAll(page, size)).build();
    }
}
```

---

## Extensões Opcionais

- [ ] Integrar com Keycloak como Identity Provider externo
- [ ] Implementar OAuth2 Authorization Code Flow
- [ ] Adicionar MFA (Multi-Factor Authentication) via TOTP
- [ ] Implementar API key authentication para serviços externos
- [ ] Adicionar audit log para ações administrativas
- [ ] Implementar password policy (mínimo 8 chars, uppercase, number, special)
- [ ] CORS dinâmico (allowed origins via config/banco)

---

## Erros Comuns

| Erro | Stack | Como evitar |
|---|---|---|
| JWT secret hardcoded no código | Todos | Usar env var ou vault |
| Não validar expiração do token | Todos | Lib padrão valida — mas testar o cenário |
| Retornar 200 quando deveria ser 401/403 | Todos | Testar com token inválido, expirado, role errado |
| Password em plaintext no banco | Todos | Sempre bcrypt com cost ≥ 12 |
| Não revogar refresh token no logout | Todos | Marcar como revoked no banco |
| CORS com `*` em produção | Todos | Listar origens específicas |
| Rate limiter per-instance em cluster | Todos | Em produção, usar Redis para rate limiting distribuído |
| `@Secured` ignorado por falta de annotation processor | Micronaut | Verificar que `micronaut-security-jwt` está no classpath |
| SQL Injection em queries manuais | Todos | Usar prepared statements SEMPRE |

---

## Como Isso Aparece em Entrevistas

- "Como funciona JWT? Quais são as partes do token?"
- "Qual a diferença entre autenticação e autorização?"
- "O que é RBAC vs ABAC?"
- "Como você implementa refresh token de forma segura?"
- "Qual a diferença entre OAuth2 e JWT?"
- "Como você previne brute force em login?"
- "O que é CORS e como configurar corretamente?"
- "Em Spring, qual a diferença entre `@Secured`, `@PreAuthorize` e `@RolesAllowed`?"
- "Como funciona o SecurityIdentity no Quarkus vs Authentication no Spring?"
- "Por que bcrypt e não SHA-256 para passwords?"

---

## Comandos de Execução

```bash
# Registrar usuário
curl -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"João","email":"joao@test.com","password":"SecureP@ss123","document":"123.456.789-00"}'

# Login
TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"joao@test.com","password":"SecureP@ss123"}' | jq -r '.accessToken')

echo $TOKEN

# Acessar endpoint protegido
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/v1/users/me

# Acessar endpoint admin (sem role ADMIN → 403)
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/v1/admin/users

# Sem token → 401
curl http://localhost:8080/api/v1/users/me

# Refresh token
curl -X POST http://localhost:8080/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken":"<refresh-token-uuid>"}'
```
