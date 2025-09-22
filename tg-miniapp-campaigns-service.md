# tg-miniapp-campaigns-service

Quests, airdrops, giveaways, and reward campaigns with condition checks (follow/join/KYC/hold, etc.). Executes rewards via **tg-miniapp-payhub-service**. Built for transparency, anti-abuse, and replay-safe claims.

---

## 1) Service Story

**Campaigns** lets projects and the ops team run incentive programs. A campaign defines tasks ("join Telegram", "follow X", "hold ≥ N FZ", "attend event"), a reward budget in in‑app currencies (STAR/FZ/PT), and claim rules. The service verifies completion, records participation, and issues payouts through Payhub idempotently.

---

## 2) Duties & Scope

### Owns (SoR)

* **Campaigns**: metadata, budget, timelines, visibility, task list
* **Tasks/Conditions**: verifiable checks (social, on-platform, KYC, wallet balance)
* **Participations**: user↔campaign state, proofs, timestamps
* **Claims**: payout state, reward amounts, receipts

### Not owned here

* Wallet balances/ledger → **Payhub**
* Identity/KYC flags → **Identity Service** / external KYC
* Events directory → **Events Service** (we can reference event IDs as tasks)

---

## 3) Public API (MVP)

> Auth via user session token. Mutations require `Idempotency-Key`.

* `GET  /v1/campaigns` → list active campaigns
* `GET  /v1/campaigns/:id` → details, tasks, user progress
* `POST /v1/campaigns/:id/progress` `{ taskId, proof }` → record completion
* `POST /v1/campaigns/:id/claim` → verify all tasks & budget → issue reward via Payhub
* `GET  /v1/campaigns/:id/claim` → claim status & receipt

**Headers**

* `Authorization: Bearer <session-token>`
* `Idempotency-Key: <uuid>` on POST
* `X-Request-Id: <uuid>`

---

## 4) Admin API (used by tg-miniapp-admin)

* `POST   /admin/v1/campaigns` — create draft
* `PATCH  /admin/v1/campaigns/:id` — edit
* `POST   /admin/v1/campaigns/:id/publish` — publish
* `POST   /admin/v1/campaigns/:id/pause` — pause
* `POST   /admin/v1/campaigns/:id/budget/topup` — add budget (via Payhub treasury transfer)
* `GET    /admin/v1/campaigns/:id/participants` — export list

Role required: `admin.campaigns.*` (checked via Identity).

---

## 5) Tasks & Verifications

MVP task types (extensible):

* **social.follow\_x**: proof via OAuth or signed callback (optional placeholder in MVP)
* **telegram.join\_channel**: check via Telegram bot member API (server‑side)
* **identity.kyc\_passed**: read from Identity
* **wallet.hold\_at\_least**: check Payhub balance in currency
* **events.attend**: reference an event check‑in from Events Service

Each task stores: `taskId, type, params{}, verifier, proofSchema`.

---

## 6) Architecture

* **Node.js + TS + Express**
* **MongoDB**: `Campaign`, `Task`, `Participation`, `Claim`
* **Redis**: idempotency cache, short‑lived proofs, rate limits
* **BullMQ**: `campaigns.claim`, `campaigns.verify_async` (for slow checks)
* **OpenTelemetry** + Pino
* **Zod** DTOs & proof schemas

**Claim flow**

1. Validate tasks complete (sync/async verifiers)
2. Check campaign budget & per-user caps
3. Call **Payhub** to **settle/mint reward**
4. Record `Claim` with payout receipt; enforce idempotency by `(campaignId,userId)`

---

## 7) Data Model

* `Campaign` `{ id, name, status:draft|published|paused|ended, startAt, endAt, budget{ currency, total, remaining }, reward{ currency, amount|formula }, tasks:[taskId], visibility, createdAt }`
* `Task` `{ taskId, type, params, verifier, proofSchema }`
* `Participation` `{ campaignId, userId, progress:{ taskId: { done, proofRef, at } }, updatedAt }`
* `Claim` `{ claimId, campaignId, userId, amount, currency, receipt, status, createdAt }`

Indexes: `Campaign(status,startAt)`, `Participation(campaignId,userId)`, `Claim(campaignId,userId)` unique on `(campaignId,userId)`

---

## 8) ENV & Configuration

```dotenv
SERVICE_ID=campaigns
SERVICE_NAME=tg-miniapp-campaigns-service
PORT=8094
NODE_ENV=development
LOG_LEVEL=info

MONGO_URL=mongodb://localhost:27017/tg_campaigns
REDIS_URL=redis://localhost:6379

SVC_PAYHUB_URL=http://svc-payhub:8081
SVC_IDENTITY_URL=http://svc-identity:8080
SVC_EVENTS_URL=http://svc-events:8096

# Business
DEFAULT_REWARD_CURRENCY=PT
MAX_CLAIMS_PER_USER_PER_DAY=5
PROOF_URL_TTL_SECONDS=600
```

Secrets: third‑party API tokens for social checks (when enabled) via secret manager.

---

## 9) Dependencies

* `express`, `zod`, `mongodb`, `ioredis`, `bullmq`
* `jose`, `pino`, `pino-http`, `helmet`, `cors`, `ky`
* Dev: `typescript`, `tsx`, `vitest`/`jest`, `supertest`, `eslint`

---

## 10) Develop & Run

```bash
pnpm i
pnpm dev       # http://localhost:8094
```

Docker/Compose similar to other services; link Mongo/Redis/Payhub.

---

## 11) Testing

* **Proof paths**: telegram join check (mock), identity KYC flag, balance threshold
* **Budget**: cap enforcement, top‑up effects, race conditions
* **Idempotency**: claim replay returns same receipt
* **Rate limits**: MAX\_CLAIMS\_PER\_USER\_PER\_DAY

Run:

```bash
pnpm test
```

---

## 12) Deploy

Helm (excerpt):

```yaml
image: { repository: ghcr.io/yourorg/tg-miniapp-campaigns-service, tag: "v0.1.0" }
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
probePaths: { liveness: /healthz, readiness: /readyz }
```

---

## 13) Roadmap

* OAuth-based social proofs (X/Discord/GitHub)
* Dynamic reward formulas (ranked leaderboards)
* Anti‑sybil (device/graph heuristics)
