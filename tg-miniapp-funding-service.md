# tg-miniapp-funding-service

Private sale / IDO / presale orchestration. Manages whitelists, allocations, tranche pricing, vesting schedules, and settlement intents. All cash flows are executed via **tg-miniapp-payhub-service**.

---

## 1) Service Story

**Funding** coordinates token sale business logic while keeping custody off‑chain in Payhub. It defines rounds (Private, IDO, Presale), admits users via whitelists/KYC hooks, allocates purchase limits, collects payment intents, and exposes vesting schedules for later claims. It never mutates balances directly—it calls Payhub for holds/settlements and refunds.

---

## 2) Duties & Scope

### Owns (SoR)

* **Rounds**: id, status, timelines, tranche ladder (price per tranche, caps)
* **Whitelists / Allowlist entries**: per user eligibility (+ optional KYC flag)
* **Allocations**: per user hard‑cap, per round/tranche
* **Payment Intents**: purchase requests that lead to Payhub holds/settlements
* **Vesting Schedules**: cliffs, periods, total entitlement (for read‑only claim UIs)

### Not owned here

* Ledger/balances/settlements → **Payhub**
* Token minting / on‑chain distribution (Phase‑2)

---

## 3) Public API (MVP)

> Auth via user session token. Mutations require `Idempotency-Key`.

* `GET  /v1/funding/rounds` → list active rounds with tranche info
* `GET  /v1/funding/rounds/:roundId` → details, user eligibility & remaining allocation
* `POST /v1/funding/rounds/:roundId/intent`

```json
{ "amount": 1000, "currency": "FZ" }
```

→ creates a **payment intent** and places a **hold** via Payhub if within limits

* `GET  /v1/funding/intents/:intentId` → status (`created|held|settled|refunded|expired`)
* `GET  /v1/funding/vesting/:programId` → schedule & accrued amounts (read‑only)

**Headers**

* `Authorization: Bearer <session-token>`
* `Idempotency-Key: <uuid>`

---

## 4) Internal API (Service↔Service)

* `POST /internal/v1/funding/settle` `{ intentId }` (called by worker after confirmation)
* `POST /internal/v1/funding/expire` `{ intentId }` (releases hold on timeout)
* **Payhub**: `/internal/v1/holds`, `/internal/v1/holds/release`, `/internal/v1/settlements`
* **Identity (optional)**: `/v1/users/:id/kyc` (read) if enforced

---

## 5) Architecture

* **Node.js + TS + Express**
* **MongoDB**: `Round`, `Tranche`, `Whitelist`, `Allocation`, `PaymentIntent`, `VestingProgram`
* **Redis**: idempotency cache, short‑lived locks
* **BullMQ**: `funding.intent.expire`, `funding.intent.settle`
* **OpenTelemetry**: traces/metrics; Pino logs
* **Zod** schemas for DTOs

**Key flows**

1. **Intent create** → validate eligibility/limits → compute tranche price → **Payhub hold** → `intent.status=held`.
2. **Confirmation** → settle via Payhub (`settlement win` to seller treasury wallet) → `settled`.
3. **Timeout** (no confirmation in N minutes) → **release** hold → `expired`.

---

## 6) Data Model

* `Round` `{ roundId, name, status: draft|active|paused|ended, startAt, endAt, baseCurrency, maxPerUser, tranches:[{price, cap}], createdAt }`
* `Whitelist` `{ roundId, userId, eligible:bool, kyc:bool, maxAllocation, createdAt }`
* `Allocation` `{ roundId, userId, remaining, updatedAt }`
* `PaymentIntent` `{ intentId, roundId, userId, currency, amount, price, holdId, status, createdAt, expiresAt }`
* `VestingProgram` `{ programId, roundId, schedule:{ cliffDays, periods, periodDays }, totalAmount }`

Indexes: `Round(status,startAt)`, `Whitelist(roundId,userId)`, `PaymentIntent(userId,roundId,createdAt)`

---

## 7) ENV & Configuration

```dotenv
SERVICE_ID=funding
SERVICE_NAME=tg-miniapp-funding-service
PORT=8091
NODE_ENV=development
LOG_LEVEL=info

MONGO_URL=mongodb://localhost:27017/tg_funding
REDIS_URL=redis://localhost:6379

# Peers
SVC_PAYHUB_URL=http://svc-payhub:8081
SVC_IDENTITY_URL=http://svc-identity:8080

# Business config
INTENT_TTL_MINUTES=15
ENFORCE_KYC=false
BASE_CURRENCIES=FZ,STAR,PT
```

Secrets: none beyond standard service JWT trust; keep via a secret manager.

---

## 8) Dependencies

* `express`, `zod`, `mongodb`, `ioredis`, `bullmq`
* `jose`, `pino`, `pino-http`, `helmet`, `cors`
* Dev: `typescript`, `tsx`, `vitest`/`jest`, `supertest`, `eslint`

---

## 9) Develop & Run

```bash
pnpm i
pnpm dev     # http://localhost:8091
```

Docker:

```bash
docker build -t tg-funding:dev .
docker run --rm -p 8091:8091 --env-file .env tg-funding:dev
```

Compose (example): see Payhub README and link this service to it.

---

## 10) Testing

* **Eligibility**: whitelist + limits honored
* **Pricing**: correct tranche selection; rounding
* **Idempotency**: duplicate intents reuse same hold
* **Lifecycle**: held → settled, held → expired (release)

Run:

```bash
pnpm test
```

---

## 11) Deploy

Helm (excerpt):

```yaml
image: { repository: ghcr.io/yourorg/tg-miniapp-funding-service, tag: "v0.1.0" }
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
probePaths: { liveness: /healthz, readiness: /readyz }
```

CI/CD: build on tag → scan → push → GitOps deploy.

---

## 12) Roadmap

* Payment provider integrations (fiat on‑ramp)
* On‑chain vesting contract mirror (EVM adapter)
* Anti‑sybil checks & duplicate prevention
