# Miniapp Identity Service
*Version:* v0.1.0  
*Last Updated:* 2025-09-24 02:46 +07  
*Owner:* FUZE Platform Identity — Auth, Profiles, Referrals

> High‑level architectural blueprint for the **Identity Service**. It authenticates Telegram users, issues short‑lived **session JWTs** for the WebApp, exposes **JWKS** for verification by services, manages **profiles, roles, referrals, language and area**, and provides **service JWTs** for trusted backends. It does **not** touch balances or ledgers.

---

## 1) Architecture Diagram

```mermaid
flowchart LR
  subgraph "Clients"
    TG["Telegram WebApp"]
    W3["Web3 Portal"]
    ADM["Admin Console"]
  end

  subgraph "Identity Service (this repo)"
    API["REST API"]
    AUTH["JWT/JWKS Engine"]
    POL["Policy Engine"]
    AC["Access Control (RBAC/Badges)"]
    ER["Event Router"]
    Q["Jobs Queue"]
    DB["Primary DB"]
    CACH["Cache"]
    WH["Webhook Dispatcher"]
  end

  subgraph "Platform Services (consumers)"
    PH["Playhub"]
    PAY["Payhub"]
    ESC["Escrow"]
    FND["Funding"]
    CMP["Campaigns"]
    EVT["Events"]
    WAT["Watchlist"]
    PRC["Price"]
    ADM2["Admin/BFF"]
    WRK["Workers"]
  end

  subgraph "External Providers"
    TGPLAT["Telegram Platform"]
    KYC["KYC Provider"]
    RPC["Chain RPC (wallet proofs)"]
  end

  TG -->|"/v1/auth/telegram/verify"| API
  W3 -->|"SIWE/TonProof Flow"| API
  ADM -->|"/v1/admin/*"| API

  API --> AUTH
  API --> POL
  POL --> AC
  API --> DB
  API --> CACH
  API --> Q
  API --> WH

  API -->|"Webhook Callbacks"| KYC
  API -->|"Verify Signature"| RPC

  PH -->|"/v1/introspect"| API
  PAY -->|"/v1/introspect"| API
  ESC -->|"/v1/introspect"| API
  FND -->|"/v1/introspect"| API
  CMP -->|"/v1/webhooks (consume)"| WH
  EVT -->|"/v1/introspect"| API
  WAT -->|"/v1/introspect"| API
  PRC -->|"/v1/introspect"| API
  ADM2 -->|"/v1/admin/*"| API
  WRK -->|"/jobs"| Q
```

*Notes:* Identity verifies Telegram **initData** server side, issues a session token for the WebApp, and exposes **JWKS** so other services can verify that token without calling back. Services use **service JWTs** for internal calls.

---

## 2) Technology Stack
| Layer | Choice | Rationale |
|---|---|---|
| Runtime | Nodejs 20 plus TypeScript | Shared stack across repos |
| Framework | Express plus Zod | Schema validation and small footprint |
| Storage | MongoDB | Flexible user and session documents |
| Cache | Redis | Session cache, rate limits, idempotency |
| Auth | jose Ed25519 JWT | Fast verification and compact keys |
| Telemetry | OpenTelemetry plus Pino | Tracing and structured logs |
| Deploy | Docker plus Helm | Standard CI and CD |

---

## 3) Responsibilities and Scope
**Owns**
- **Telegram login**: verify initData HMAC, create or update user, issue session JWT.  
- **Profiles**: username, language, coarse area, preferences.  
- **Roles**: user, staff, organizer, partner; entitlements for Admin.  
- **Referrals**: bind parent child relationships and track attribution.  
- **JWKS**: publish public keys for session verification; rotate keys.  
- **Service tokens**: mint and rotate service JWTs for trusted backends.  
- **Rate limits**: protect login and referral endpoints.  
- **Audit**: record auth events and role grants.

**Out of scope**
- Wallets, balances, or settlements — handled by Payhub.  
- Business logic such as games or campaigns — delegated to domain services.

---

## 4) Data Flows

### A) SIWE login and wallet link

```mermaid
sequenceDiagram
  participant U as "User",
  participant ID as "Identity",
  U->>ID: POST /v1/session/siwe/prepare { address, chainId, domain },
  ID-->>U: 200 { nonce, message, expiresAt },
  U->>ID: POST /v1/session/siwe/verify { signature, message },
  alt first time wallet
    ID->>ID: create USER and WALLET,
  else existing user
    ID->>ID: rotate session,
  end
  ID-->>U: 200 { session, cookie }
```

### B) Badge application and admin review

```mermaid
sequenceDiagram
  participant U as "User",
  participant ID as "Identity",
  participant AD as "Admin",
  U->>ID: POST /v1/badges/{type}/apply { evidence },
  ID-->>U: 202 accepted,
  AD->>ID: POST /admin/v1/badges/{applicationId}/review { approve|reject },
  alt approve
    ID->>ID: issue BADGE state active,
    ID-->>U: emit identity.badge.updated,
  else reject
    ID-->>U: record reason and notify,
  end
```

### C) KYC provider callback

```mermaid
sequenceDiagram
  participant K as "KYC Provider",
  participant ID as "Identity",
  K->>ID: POST /v1/kyc/webhook { reference, state },
  ID->>ID: verify HMAC, 
  alt approved
    ID->>ID: update KYC_CASE, activate related badges,
  else rejected
    ID->>ID: suspend or keep badges pending,
  end
  ID-->>K: 202 accepted
```

### D) Service token mint and introspect

```mermaid
sequenceDiagram
  participant S as "Internal Service",
  participant ID as "Identity",
  S->>ID: POST /v1/tokens/service { serviceId, subjectId, scopes, ttlSeconds },
  ID-->>S: 201 { token },
  S->>ID: POST /v1/introspect { token },
  ID-->>S: 200 { active, sub, scope, exp }
```
---

## 5) Security and Privacy
- **Token model**: short lived session JWT such as 15 minutes, renewable via refresh route or silent reauth using Telegram if needed.  
- **Signing**: Ed25519 keys rotated regularly; JWKS exposes public keys with `kid`.  
- **Session binding**: store `uaHash` and `ipHash` as hints to flag anomalies.  
- **Rate limits**: login and referral bind limited per user and IP.  
- **Input validation**: Zod schemas; strict TTL on Telegram initData per Telegram docs.  
- **Least PII**: store only Telegram id and coarse area; avoid sensitive data.  
- **Secrets**: bot token and signing keys stored in secret manager; not in repo.  
- **Audit**: every role grant and session revoke recorded.  
- **CORS**: restrict origins to Telegram WebApp domain and Admin origin.  
- **CSRF**: not applicable for pure token header auth; still use SameSite cookies if any.

---

## 6) Scalability and Reliability
- Stateless API with Redis for session cache and rate limits; horizontal scale.  
- MongoDB with indexes for user and session lookups.  
- Health probes `/healthz` and `/readyz` include DB, Redis, key freshness.  
- SLOs: p95 < 80 ms for login after cache warm; 99.95 percent availability.  
- DR: daily backups; consider PITR; runbook for emergency key rotation.

---

## 7) Observability
- **Tracing**: propagate `requestId`; spans for login, referral bind, and JWKS fetch.  
- **Metrics**: login success rate, referral bind rate, token verification failures.  
- **Logs**: structured logs without raw initData; include user id and request id.  
- **Alerts**: spike in failed HMAC verify, abnormal login rate, JWKS staleness.

---

## 8) User Stories and Feature List
### Feature List
- Telegram login and session issuance.  
- Profiles, preferences, and coarse area.  
- Referrals and binding with anti self‑deal checks.  
- Roles and entitlements for Admin.  
- JWKS and service token minting.

### User Stories
- *As a new user*, I log in via Telegram and receive a session token so that I can use the app securely.  
- *As a referrer*, I share my referral code so that new users bind to me automatically.  
- *As staff*, I grant the organizer role to a partner so that they can submit events.  
- *As a backend service*, I verify session tokens using JWKS without extra network calls.

---

## 9) Roadmap
- Device binding and risk scoring on login.  
- WebAuthn step up for staff accounts.  
- Phone verification via Telegram or SMS if needed.  
- Domain events stream for user lifecycle.

---

## 10) Compatibility Notes
- WebApp consumes login and profile endpoints.  
- All domain services verify session JWTs via **JWKS**.  
- Admin relies on roles from Identity.  
- No direct calls to Payhub; identity id is the key for ledgers across services.
