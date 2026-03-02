# Level 14 — Estimation, Data Partitioning, Replication Strategies & Auth

> **Objetivo:** Dominar back-of-the-envelope estimation, implementar data partitioning
> strategies, replication configurável e um sistema de autenticação OAuth 2.0 / JWT.

**Referência:**
- [31-back-of-the-envelope-estimation.md](../../.docs/SYSTEM-DESIGN/31-back-of-the-envelope-estimation.md)
- [32-data-partitioning-strategies.md](../../.docs/SYSTEM-DESIGN/32-data-partitioning-strategies.md)
- [33-replication-strategies.md](../../.docs/SYSTEM-DESIGN/33-replication-strategies.md)
- [35-oauth-jwt-authentication.md](../../.docs/SYSTEM-DESIGN/35-oauth-jwt-authentication.md)

**Pré-requisito:** Level 13 completo.

---

## Parte 1 — ADR: Authentication Design

**Arquivo:** `docs/adrs/ADR-001-authentication-authorization-design.md`

**Decisão:** Strategy de autenticação e autorização.

**Options:**
1. **Session-based** — server-side session, cookie-based
2. **JWT (stateless)** — token auto-contido, sem server state
3. **OAuth 2.0 + OIDC** — delegated auth com IdP
4. **API Keys** — para service-to-service
5. **mTLS** — mutual TLS para service mesh

**Decision Drivers:**
- Stateless vs stateful trade-offs
- Token revocation strategy
- Refresh token rotation
- Multi-tenant support
- Service-to-service auth

**Critérios de aceite:**
- [ ] 5 options comparadas
- [ ] Token lifecycle documentado (issue, refresh, revoke)
- [ ] OAuth 2.0 flows documentados (authorization code, client credentials)
- [ ] JWT claims design

---

## Parte 2 — Diagrama DrawIO

**Arquivo:** `docs/diagrams/14-estimation-partitioning-auth.drawio`

**View 1 — Data Partitioning Strategies:**
```
Horizontal Partitioning (Sharding):
┌──────┐ ┌──────┐ ┌──────┐
│Users │ │Users │ │Users │
│A-H   │ │I-P   │ │Q-Z   │
└──────┘ └──────┘ └──────┘

Vertical Partitioning:
┌──────┐ ┌──────┐ ┌──────┐
│User  │ │User  │ │User  │
│Core  │ │Profile│ │Prefs │
│(id,  │ │(bio, │ │(theme│
│email)│ │avatar)│ │,lang)│
└──────┘ └──────┘ └──────┘

Directory-Based:
┌──────────┐    ┌──────┐
│ Lookup   │───▶│Shard │
│ Service  │───▶│  N   │
└──────────┘    └──────┘
```

**View 2 — OAuth 2.0 Authorization Code Flow:**
```
Client → Authorization Server: /authorize
User authenticates + consents
Authorization Server → Client: authorization_code
Client → Authorization Server: /token (code + client_secret)
Authorization Server → Client: access_token + refresh_token
Client → Resource Server: API call + Bearer token
Resource Server validates JWT → Response
```

**View 3 — Replication Strategies:** Single-leader, Multi-leader, Leaderless side-by-side

**Critérios de aceite:**
- [ ] 3 partitioning strategies visualizadas
- [ ] OAuth 2.0 flow completo
- [ ] 3 replication strategies comparadas

---

## Parte 3 — Implementação

### 3.1 — Go

**Estrutura:**
```
go/
├── cmd/
│   ├── auth-server/main.go       ← OAuth 2.0 / JWT auth server
│   ├── resource-server/main.go   ← Protected resource API
│   └── estimation/main.go        ← Estimation calculator
├── internal/
│   ├── auth/
│   │   ├── oauth.go              ← OAuth 2.0 flows
│   │   ├── jwt.go                ← JWT generation/validation
│   │   ├── token_store.go        ← Token storage (Redis)
│   │   ├── refresh.go            ← Refresh token rotation
│   │   ├── middleware.go         ← Auth middleware
│   │   └── auth_test.go
│   ├── partition/
│   │   ├── horizontal.go         ← Range/Hash partitioning
│   │   ├── vertical.go           ← Vertical partitioning
│   │   ├── directory.go          ← Directory-based
│   │   ├── router.go             ← Partition-aware query router
│   │   └── partition_test.go
│   ├── replication/
│   │   ├── single_leader.go      ← Sync + async followers
│   │   ├── multi_leader.go       ← Multi-datacenter
│   │   ├── leaderless.go         ← Quorum-based
│   │   ├── conflict.go           ← Conflict resolution
│   │   └── replication_test.go
│   └── estimation/
│       ├── calculator.go         ← Estimation formulas
│       ├── templates.go          ← Common system templates
│       └── calculator_test.go
├── go.mod
└── Makefile
```

**Funcionalidades Go:**
1. **OAuth 2.0 Server** — authorization code + client credentials flows
2. **JWT** — RS256 signing, claims validation, expiration
3. **Refresh Token Rotation** — one-time use, family detection
4. **Token Revocation** — blacklist com Redis
5. **Data Partitioning** — horizontal, vertical, directory-based
6. **Partition Router** — routes queries to correct partition
7. **3 Replication Strategies** implementadas
8. **Estimation Calculator** — QPS, storage, bandwidth, cache formulas
9. **RBAC** — role-based access control com JWT claims

**Critérios de aceite Go:**
- [ ] OAuth 2.0 authorization code flow end-to-end
- [ ] JWT RS256 com key rotation support
- [ ] Refresh token rotation com reuse detection
- [ ] Token revocation via Redis blacklist
- [ ] 3 partitioning strategies com router
- [ ] 3 replication strategies
- [ ] Estimation calculator com templates para 5+ systems
- [ ] ≥ 20 testes

---

### 3.2 — Java

**Funcionalidades Java:**
1. **Spring Security OAuth 2.0 Authorization Server**
2. **Spring Security Resource Server** (JWT validation)
3. **Refresh token rotation** com Spring Security
4. **Partitioning** com Spring Data JPA custom routing
5. **Records** para JWT claims e partition configs
6. **Estimation** tool com CLI

**Critérios de aceite Java:**
- [ ] Spring Authorization Server funcional
- [ ] Resource server validando JWTs
- [ ] Refresh token rotation
- [ ] Partitioning com custom DataSource routing
- [ ] JaCoCo ≥ 80%

---

## Parte 4 — Back-of-the-Envelope Exercises

Complete as estimativas para cada sistema:

| Sistema | DAU | QPS (read) | QPS (write) | Storage/day | Cache | Bandwidth |
|---------|-----|-----------|-------------|-------------|-------|-----------|
| URL Shortener | 100M | | | | | |
| Twitter | 500M | | | | | |
| Instagram | 1B | | | | | |
| WhatsApp | 2B | | | | | |
| YouTube | 2B | | | | | |

---

## Definição de Pronto (DoD)

- [ ] ADR de authentication design
- [ ] DrawIO com 3 views
- [ ] Go e Java: OAuth + JWT + Partitioning + Replication + Estimation
- [ ] 5 estimativas back-of-the-envelope completas
- [ ] Commit: `feat(system-design-14): estimation partitioning replication auth`
