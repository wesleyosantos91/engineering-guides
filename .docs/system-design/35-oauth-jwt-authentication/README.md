# 35. OAuth 2.0 / JWT / Autenticação

> **Categoria:** Fundamentos e Building Blocks  
> **Nível:** Essencial para qualquer sistema com autenticação  
> **Complexidade:** Média-Alta

---

## Definição

**OAuth 2.0** é o protocolo padrão de **autorização** que permite aplicações third-party acessarem recursos em nome do usuário sem expor credenciais. **JWT (JSON Web Token)** é um formato de token compacto e self-contained usado para transmitir claims entre partes. Juntos, formam a base de **autenticação e autorização** em APIs modernas.

---

## Conceitos Fundamentais

```
┌─────────────────────────────────────────────────────────────────┐
│  Authentication vs Authorization                                 │
│                                                                  │
│  Authentication (AuthN):  "QUEM é você?"                        │
│    → Login, credenciais, prova de identidade                     │
│    → Resultado: identity token (JWT, session)                    │
│                                                                  │
│  Authorization (AuthZ):   "O que você PODE fazer?"              │
│    → Permissões, roles, scopes                                   │
│    → Resultado: access token com scopes/claims                   │
│                                                                  │
│  OAuth 2.0 → Protocolo de AUTORIZAÇÃO                           │
│  OpenID Connect (OIDC) → Camada de AUTENTICAÇÃO sobre OAuth 2.0│
│  JWT → FORMATO de token (pode ser usado em ambos)               │
└─────────────────────────────────────────────────────────────────┘
```

---

## OAuth 2.0

### Participantes

```
┌──────────────────────────────────────────────────┐
│  Resource Owner  →  Usuário (dono dos dados)     │
│  Client          →  Aplicação (quer acessar)     │
│  Auth Server     →  Emite tokens (Google, Auth0) │
│  Resource Server →  API protegida (seus dados)   │
└──────────────────────────────────────────────────┘

  Exemplo: App "FotoEdit" quer acessar suas fotos do Google Photos

  Resource Owner:  Você
  Client:          FotoEdit App
  Auth Server:     accounts.google.com
  Resource Server: photos.googleapis.com
```

### Authorization Code Flow (mais seguro, mais comum)

```
  ┌─────────┐                ┌────────────┐              ┌─────────────┐
  │  User    │                │  Client    │              │ Auth Server │
  │ (Browser)│                │ (FotoEdit) │              │  (Google)   │
  └────┬─────┘                └─────┬──────┘              └──────┬──────┘
       │  1. Click "Login with Google"│                          │
       │──────────────────────────────▶                          │
       │                              │  2. Redirect to Auth Server
       │◀─────────────────────────────│  /authorize?             │
       │  redirect                    │  client_id=xxx&          │
       │                              │  redirect_uri=xxx&       │
       │  3. User authenticates       │  scope=photos.read&      │
       │─────────────────────────────────────────────────────────▶
       │                              │                          │
       │  4. "Allow FotoEdit to       │                          │
       │      access your photos?"    │                          │
       │  5. User clicks "Allow"      │                          │
       │─────────────────────────────────────────────────────────▶
       │                              │                          │
       │  6. Redirect back with code  │                          │
       │◀─────────────────────────────────────────────────────────
       │  /callback?code=AUTH_CODE    │                          │
       │──────────────────────────────▶                          │
       │                              │  7. Exchange code→tokens │
       │                              │  POST /token             │
       │                              │  code=AUTH_CODE&         │
       │                              │  client_secret=xxx       │
       │                              │─────────────────────────▶│
       │                              │                          │
       │                              │  8. Receive tokens       │
       │                              │◀─────────────────────────│
       │                              │  { access_token,         │
       │                              │    refresh_token,        │
       │                              │    id_token (OIDC) }     │
       │                              │                          │
       │                              │  9. Use access_token     │
       │                              │  GET /photos             │
       │                              │  Authorization: Bearer   │
       │                              │  xxx_access_token        │
       │                              │─────────────────────────▶│
       │                              │                       Resource
       │                              │                       Server
```

### Outros Flows

```
┌────────────────────────────┬──────────────────────────────────────┐
│  Flow                      │  Quando usar                        │
├────────────────────────────┼──────────────────────────────────────┤
│  Authorization Code + PKCE │  SPAs, Mobile (sem client_secret)   │
│  Client Credentials        │  Machine-to-Machine (M2M)           │
│  Device Authorization      │  Smart TVs, CLIs (sem browser)      │
│  Refresh Token             │  Renovar access_token expirado      │
├────────────────────────────┼──────────────────────────────────────┤
│  ❌ Implicit (deprecated)  │  Era usado por SPAs (inseguro)      │
│  ❌ Resource Owner Password│  Legacy (compartilha password)      │
└────────────────────────────┴──────────────────────────────────────┘
```

### PKCE (Proof Key for Code Exchange)

```
Para clients que NÃO podem guardar secrets (SPA, mobile):

  1. Client gera code_verifier (random string, 43-128 chars)
  2. Client calcula code_challenge = SHA256(code_verifier)
  3. Authorization request inclui code_challenge
  4. Token request inclui code_verifier
  5. Auth server verifica: SHA256(code_verifier) == code_challenge?

  Previne: interceptação do authorization code
  (atacante não tem o code_verifier para trocar por token)
```

---

## JWT (JSON Web Token)

### Estrutura

```
  eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.
  eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4iLCJpYXQiOjE1MTYyMzkwMjJ9.
  SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

  ├──────── Header ─────────┤├────────── Payload ──────────┤├── Signature ──┤
       (Base64URL)                  (Base64URL)              (Verifiable)

  Header:
  {
    "alg": "RS256",    // Algoritmo de assinatura
    "typ": "JWT",      // Tipo do token
    "kid": "key-id-1"  // Key ID (para key rotation)
  }

  Payload (Claims):
  {
    "sub": "1234567890",          // Subject (user ID)
    "name": "João Silva",         // Custom claim
    "email": "joao@example.com",  // Custom claim
    "role": "admin",              // Custom claim
    "iat": 1516239022,            // Issued At
    "exp": 1516242622,            // Expiration (1h)
    "iss": "auth.myapp.com",      // Issuer
    "aud": "api.myapp.com"        // Audience
  }

  Signature:
  RSASHA256(
    base64UrlEncode(header) + "." + base64UrlEncode(payload),
    privateKey
  )
```

### Verificação

```
  ┌───────────────────────────────────────────────────────┐
  │  Como o Resource Server verifica um JWT:               │
  │                                                        │
  │  1. Split por "." → header, payload, signature         │
  │  2. Decode header → extrair alg e kid                  │
  │  3. Buscar public key (JWKS endpoint ou local)         │
  │  4. Verificar signature com public key                 │
  │  5. Verificar claims:                                  │
  │     - exp: não expirou?                                │
  │     - iss: emitido pelo Auth Server correto?           │
  │     - aud: destinado a esta API?                       │
  │                                                        │
  │  ✅ Se tudo OK → request autenticado                   │
  │  ❌ Se falhar → 401 Unauthorized                       │
  │                                                        │
  │  IMPORTANTE: Nenhuma chamada ao Auth Server necessária!│
  │  JWT é SELF-CONTAINED → verificável localmente         │
  └───────────────────────────────────────────────────────┘
```

---

## Tipos de Tokens

```
┌─────────────────────────────────────────────────────────────────┐
│  Access Token                                                    │
│  ├── Curta duração: 15 min – 1 hora                             │
│  ├── Enviado em CADA request (Authorization: Bearer xxx)         │
│  ├── Contém scopes/permissions                                   │
│  └── Se vazou: dano limitado (expira rápido)                    │
│                                                                  │
│  Refresh Token                                                   │
│  ├── Longa duração: dias, semanas, meses                        │
│  ├── Usado SOMENTE para obter novo access_token                 │
│  ├── Armazenado de forma segura (httpOnly cookie, DB)           │
│  ├── Se vazou: pode ser revogado no Auth Server                 │
│  └── Rotation: cada uso gera novo refresh_token                 │
│                                                                  │
│  ID Token (OpenID Connect)                                       │
│  ├── Contém info do usuário (name, email, picture)              │
│  ├── Usado pela aplicação CLIENT (não enviado para APIs)         │
│  └── JWT format sempre                                           │
└─────────────────────────────────────────────────────────────────┘
```

### Token Lifecycle

```
  ┌────────┐     ┌───────────┐     ┌──────────────┐
  │ Login  │────▶│ Auth Server│────▶│ access_token │
  │        │     │           │     │ (15min)      │
  │        │     │           │     │ refresh_token│
  │        │     │           │     │ (30 days)    │
  └────────┘     └───────────┘     └──────┬───────┘
                                          │
                    ┌─────────────────────▼──────────────────────┐
                    │                                            │
                    │  Use access_token for API calls            │
                    │  Authorization: Bearer <access_token>      │
                    │                                            │
                    │  When access_token expires (15 min):       │
                    │  POST /token                               │
                    │    grant_type=refresh_token                │
                    │    refresh_token=xxx                       │
                    │  → Get NEW access_token (+ new refresh)   │
                    │                                            │
                    │  When refresh_token expires (30 days):     │
                    │  → User must login again                   │
                    └───────────────────────────────────────────┘
```

---

## JWT vs Session-Based Auth

```
┌─────────────────┬──────────────────────┬──────────────────────┐
│                 │  Session (Opaque)     │  JWT (Self-contained)│
├─────────────────┼──────────────────────┼──────────────────────┤
│  Armazenamento  │  Server-side (Redis)  │  Client-side         │
│  Verificação    │  Lookup no Redis      │  Verify assinatura   │
│  Revogação      │  Delete da session    │  Difícil (stateless) │
│  Escalabilidade │  Session store = SPOF │  Sem state no server │
│  Cross-origin   │  Cookie → same-origin │  Header → qualquer   │
│  Payload        │  Referência (ID)      │  Claims completas    │
│  Tamanho        │  ~32 bytes            │  ~500+ bytes         │
│  Microservices  │  Shared session store │  Cada service verifica│
└─────────────────┴──────────────────────┴──────────────────────┘

                  Session                    JWT
               ┌──────────┐            ┌──────────┐
 Request ─────▶│ API Server├──lookup──▶│  Redis   │
               │          │            │ (session)│
               └──────────┘            └──────────┘
               
               ┌──────────┐
 Request ─────▶│ API Server│  ← Verifica JWT localmente
 (JWT header)  │ (no state)│     Sem chamada externa!
               └──────────┘
```

---

## Segurança

### Onde Armazenar Tokens (Client-side)

```
┌──────────────────────┬───────────┬──────────────┬──────────────┐
│  Storage             │  XSS Safe │  CSRF Safe   │  Recomendado │
├──────────────────────┼───────────┼──────────────┼──────────────┤
│  localStorage        │  ❌ Não   │  ✅ Sim      │  ❌          │
│  sessionStorage      │  ❌ Não   │  ✅ Sim      │  ❌          │
│  Cookie (httpOnly,   │  ✅ Sim   │  ❌ Precisa  │  ✅ (com     │
│   secure, sameSite)  │           │  CSRF token  │   CSRF prot) │
│  In-memory (variable)│  ✅ Sim   │  ✅ Sim      │  🟡 (perde   │
│                      │           │              │  no refresh) │
└──────────────────────┴───────────┴──────────────┴──────────────┘

  Melhor prática (SPA):
  - Access token: in-memory (JavaScript variable)
  - Refresh token: httpOnly secure cookie
  - CSRF protection: SameSite=Strict ou CSRF token
```

### Revogação de JWT

```
Problema: JWT é stateless → como invalidar antes de expirar?

  Soluções:
  1. Short-lived tokens (15 min)
     → Dano limitado se vazou
     
  2. Token blacklist (Redis):
     SET blacklist:{jti} EX 900  -- TTL = remaining token lifetime
     → Cada request verifica blacklist (adiciona latência)
     
  3. Token versioning:
     User tem token_version no DB
     JWT inclui claim "ver": 5
     Se user.token_version > token.ver → reject
     → Revogar = incrementar token_version
     
  4. Refresh token rotation:
     Cada refresh gera novo refresh_token
     Se refresh_token reusado → revogar toda a família
     → Detecta roubo de token
```

### Key Rotation

```
  ┌────────────────────────────────────────────────────────┐
  │  Problem: Se a signing key é comprometida,             │
  │           TODOS os tokens são inválidos após rotação   │
  │                                                        │
  │  Solução: Key Rotation com JWKS                        │
  │                                                        │
  │  1. JWT header inclui "kid" (key ID)                   │
  │  2. Auth Server publica JWKS (JSON Web Key Set):       │
  │     GET /.well-known/jwks.json                         │
  │     { "keys": [                                        │
  │       { "kid": "key-2024-01", "kty": "RSA", ... },    │
  │       { "kid": "key-2024-02", "kty": "RSA", ... }     │
  │     ]}                                                 │
  │  3. Resource Server cacheia keys + refresh periódico    │
  │  4. Rotação: adicionar nova key, assinar com nova,     │
  │     manter antiga até tokens existentes expirarem       │
  └────────────────────────────────────────────────────────┘
```

---

## System Design: Auth Service

```
  ┌────────────────────────────────────────────────────────────────┐
  │                     Architecture Overview                      │
  │                                                                │
  │  ┌────────┐     ┌─────────────┐     ┌──────────────────────┐  │
  │  │ Client │────▶│ API Gateway │────▶│ Microservices        │  │
  │  │ (SPA)  │     │ (validates  │     │ (trust gateway's     │  │
  │  └────────┘     │  JWT)       │     │  validated claims)   │  │
  │                 └──────┬──────┘     └──────────────────────┘  │
  │                        │                                       │
  │              ┌─────────▼──────────┐                            │
  │              │    Auth Service    │                            │
  │              │  ┌──────────────┐  │                            │
  │              │  │ Login/Signup │  │                            │
  │              │  │ Token Issue  │  │                            │
  │              │  │ Token Refresh│  │                            │
  │              │  │ Key Rotation │  │                            │
  │              │  └──────────────┘  │                            │
  │              │         │          │                            │
  │              │  ┌──────▼───────┐  │                            │
  │              │  │  User DB     │  │  ┌─────────────────────┐  │
  │              │  │  (Postgres)  │  │  │ Redis               │  │
  │              │  └──────────────┘  │  │ - Session store     │  │
  │              │         │          │  │ - Token blacklist   │  │
  │              │  ┌──────▼───────┐  │  │ - Rate limiting     │  │
  │              │  │  JWKS Endpoint│ │  └─────────────────────┘  │
  │              │  │(.well-known) │  │                            │
  │              │  └──────────────┘  │                            │
  │              └────────────────────┘                            │
  └────────────────────────────────────────────────────────────────┘

  Fluxo:
  1. Client faz login → Auth Service valida credenciais
  2. Auth Service emite JWT (access + refresh + id tokens)
  3. Client envia JWT em cada request → API Gateway valida
  4. API Gateway injeta claims nos headers → microsserviços confiam
  5. Token expira → Client usa refresh token → novo access token
```

---

## Uso em Big Techs

| Empresa | Stack | Detalhes |
|---------|-------|----------|
| **Google** | OAuth 2.0 + OIDC | Identity Platform, `accounts.google.com` |
| **Meta** | OAuth 2.0 | Facebook Login, Instagram Graph API |
| **Amazon** | Cognito + OAuth 2.0 | User pools, federated identity |
| **Microsoft** | OAuth 2.0 + OIDC | Azure AD, MSAL library |
| **Auth0 (Okta)** | OAuth 2.0 + OIDC | IDaaS specializado |
| **Stripe** | API Keys + OAuth | OAuth Connect para plataformas |
| **GitHub** | OAuth 2.0 Apps | GitHub Apps com fine-grained permissions |
| **Spotify** | OAuth 2.0 + PKCE | Authorization Code com PKCE para SPAs |

---

## Perguntas Frequentes em Entrevistas

1. **"OAuth 2.0 é autenticação ou autorização?"**
   - Autorização. OIDC (camada sobre OAuth) é autenticação
   - OAuth define COMO acessar recursos, não QUEM é o user

2. **"JWT vs Session? Quando usar qual?"**
   - JWT: microservices, stateless APIs, cross-domain
   - Session: monolitos, necessidade de revogação imediata
   - Híbrido: JWT + blacklist em Redis

3. **"Como revogar um JWT?"**
   - Short-lived (15min) + refresh token rotation
   - Token blacklist em Redis (TTL = remaining lifetime)
   - Token versioning no user record

4. **"O que é PKCE e por que é necessário?"**
   - Proof Key for Code Exchange
   - Protege SPAs e mobile apps que não têm client_secret
   - code_challenge no authorize, code_verifier no token exchange

5. **"Onde armazenar tokens no browser?"**
   - Access: in-memory (JS variable)
   - Refresh: httpOnly secure SameSite cookie
   - NUNCA localStorage (XSS vulnerability)

---

## Referências

- RFC 6749 — *"The OAuth 2.0 Authorization Framework"*
- RFC 7519 — *"JSON Web Token (JWT)"*
- RFC 7636 — *"Proof Key for Code Exchange (PKCE)"*
- OpenID Connect Core Spec — openid.net/connect
- Auth0 Docs — *"OAuth 2.0 Authorization Framework"*
- OWASP — *"JSON Web Token Cheat Sheet"*
- Okta Developer — *"OAuth 2.0 Simplified"*
