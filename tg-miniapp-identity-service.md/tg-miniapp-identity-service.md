# tg-miniapp-identity-service

Identity, auth, locale/geo, referrals, and roles/entitlements for the Telegram Mini App ecosystem.

---

## 1) Service Story

The **Identity Service** is the source of truth (SoR) for **who the user is** across the platform. It verifies Telegram WebApp sessions, stores user profiles and locale/area buckets, maintains the referral graph, and exposes role-/entitlement-aware lookups to other services. It intentionally **does not** hold balances or settlement logic—those live in **Payhub**.

---

## 2) Duties & Scope

### What it owns (SoR)

* **Users & Profiles**: Telegram identifiers, usernames, avatars, language.
* **Auth**: WebApp HMAC validation → short-lived session token issuance.
* **Locales & Geo Areas**: user content locale + optional area bucketing (city/district/country).
* **Referrals**: code creation, binding, tiered relationships.
* **Roles & Entitlements**: role assignment and `can(user, action)` introspection.

### What it does **not** own

* Wallets, balances, holds, settlements → **tg-miniapp-payhub-service**.
* Game sessions, GST → **tg-miniapp-playhub-service**.
* Campaign/airdrop state, discovery feeds, funding, events → respective services.

---

## 3) Public API (MVP)

> Paths are indicative; see OpenAPI in `/openapi/identity.yaml` when available.

* `POST /v1/auth/telegram` — Verify Telegram `initData` and issue session token.
* `GET  /v1/users/me` — Return current profile + roles/entitlements snapshot.
* `PATCH /v1/users/me` — Update language, geo bucket consent, profile prefs.
* `POST /v1/referrals/use` — Bind to a referral code (idempotent, one-time).
* `GET  /v1/geo/nearby?km=10` — Nearby users count (privacy-preserving bins).

**Headers**

* `Authorization: Bearer <session-token>`
* `Idempotency-Key: <uuid-v4>` on mutating endpoints
* `X-Request-Id: <uuid>` for tracing

---

## 4) Architecture

* **Node.js + TypeScript + Express**
* **MongoDB** for user, referral, role documents
* **Redis** for session/idempotency cache and short-lived locks
* **OpenTelemetry** for traces/metrics; Pino for structured logs
* **Zod** for request/response validation

**Key flows**

* **Telegram WebApp Auth**: Validate `initData` HMAC → create/find user → mint session JWT.
* **Entitlements**: roles → permissions via config snapshot from `tg-miniapp-config`.
* **Geo Bucketing**: opt-in geohash (5–6 precision) to support “nearby” counts without precise location storage.

---

## 5) Data Model (MVP)

* `User` `{ _id, tgId, username, firstName, lastName, photoUrl, languageCode, createdAt, updatedAt, settings{ contentLocale, discoverableAreaOptIn }, area{ geohash, city, country }, referral{ code, parentId, boundAt }, roles:[string] }`
* `IdempotencyLog` `{ scope, key, requestHash, response, status, ttl, createdAt }`
* `RoleBinding` `{ userId, roles:[string], updatedAt }`

Indexes: `User(tgId)`, `User(area.geohash)`, `Idempotency(scope,key)` unique

---

## 6) Configuration & ENV

Create `.env` using this template:

```dotenv
# Service
SERVICE_ID=identity
SERVICE_NAME=tg-miniapp-identity-service
PORT=8080
NODE_ENV=development
LOG_LEVEL=info

# Datastores
MONGO_URL=mongodb://localhost:27017/tg_identity
REDIS_URL=redis://localhost:6379

# Auth / Crypto
TELEGRAM_BOT_TOKEN=replace_me               # for WebApp HMAC validation
SESSION_JWT_PRIVATE_JWK={"kty":"OKP","crv":"Ed25519","d":"REPLACE","x":"REPLACE"}
SESSION_JWT_ISSUER=tg-miniapp
SESSION_JWT_AUDIENCE=webapp
SESSION_TTL_SECONDS=7200                     # 2h session tokens

# Config Service
CONFIG_BASE_URL=http://tg-config:8080       # config publisher endpoint or static URL
CONFIG_REFRESH_SECONDS=60

# Features & Limits
FEATURE_REFERRALS=true
GEOHASH_PRECISION=6                          # area bucketing granularity
RATE_LIMIT_AUTH_PER_MINUTE=10
```

Secrets guidance:

* Generate Ed25519 JWKs offline; rotate quarterly; store in your secret manager.
* Restrict `TELEGRAM_BOT_TOKEN` access to this service only.

---

## 7) Dependencies (runtime)

* `express`, `zod`, `pino`, `pino-http`, `jsonwebtoken` or `jose` (Ed25519), `ioredis`, `mongodb`
* `fast-querystring` (Telegram initData parsing)
* `cors`, `helmet` (HTTP hardening)

**Dev**: `typescript`, `tsx`, `vitest`/`jest`, `ts-node`, `supertest`, `eslint`

---

## 8) Run & Develop

### Using pnpm (recommended)

```bash
pnpm i
pnpm dev           # starts on http://localhost:${PORT:-8080}
```

### Using Docker

```bash
docker build -t tg-identity:dev .
docker run --rm -p 8080:8080 \
  --env-file .env \
  tg-identity:dev
```

### With Docker Compose (example)

```yaml
services:
  identity:
    build: .
    ports: ["8080:8080"]
    env_file: .env
    depends_on: [mongo, redis]
  mongo:
    image: mongo:7
    ports: ["27017:27017"]
  redis:
    image: redis:7
    ports: ["6379:6379"]
```

---

## 9) Testing

* **Unit**: business utilities (HMAC verify, JWT mint/verify, geohash bucketing)
* **API**: Supertest suite for `/v1/auth/telegram`, `/v1/users/me`, `/v1/referrals/use`
* **Contract**: minimal schema validation for session JWT consumed by WebApp/PlayHub

Run tests:

```bash
pnpm test
```

Coverage target (MVP): **80% lines**, **100%** for auth & idempotency utilities.

---

## 10) Deploy

### Helm values (excerpt)

```yaml
image: { repository: ghcr.io/yourorg/tg-miniapp-identity-service, tag: "v0.1.0" }
env:
  - name: NODE_ENV
    value: production
  - name: PORT
    value: "8080"
  - name: CONFIG_BASE_URL
    value: http://tg-config:8080
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
probePaths:
  liveness: /healthz
  readiness: /readyz
```

### GitHub Actions (build & push)

* Build on tag → push to GHCR/ECR
* Scan image (Trivy) → deploy with ArgoCD/GitOps

---

## 11) Operational Notes

* **SLOs**: 99.9% availability; p95 latency < 120ms for `GET /v1/users/me`.
* **Rate limits**: `POST /v1/auth/telegram` ≤ 10/min/user.
* **Idempotency**: All mutations require `Idempotency-Key`.
* **PII/Privacy**: area stored as geohash bin, not exact coords; deletion/export endpoint planned.

---

## 12) Roadmap (next)

* Session revocation & device list
* Self-serve data export/delete (GDPR-style) hooks
* Admin role editor and audit trails
* Hook to **campaigns** for role/quest gating
