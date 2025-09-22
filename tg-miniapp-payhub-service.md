# tg-miniapp-payhub-service

Ledger & payouts bank for the Telegram Mini App. Source of Record (SoR) for balances, ledger entries, holds, settlements, conversions, deposits, withdrawals, and rewards across currencies: **STAR, FZ, PT, FUZE, USDT** (configure per env).

---

## 1) Service Story

**Payhub** is the platform’s money brain. It enforces balance invariants, performs idempotent holds and settlements, and exposes internal APIs to domain services (PlayHub, Campaigns, Escrow, Funding). It is intentionally minimal: **no games or business rules**, only financial primitives with strict validation and audit trails.

---

## 2) Duties & Scope

### Owns (SoR)

* **Wallets & Balances** (per user, per currency)
* **Ledger** (immutable entries; double-entry style)
* **Holds** (escrowed amounts for bets/escrows/withdrawals)
* **Settlements** (apply wins/losses; release holds)
* **Conversions** (e.g., STAR↔FZ, FUZE→PT/FZ)
* **Deposits / Withdrawals** (off-chain now; on-chain adapters later)
* **Rewards** (mints/payouts from campaigns/airdrop)

### Not owned here

* Game/CFB logic → PlayHub
* Campaign rules → Campaigns Service
* Escrow business rules → Escrow Service

---

## 3) Internal API (MVP)

> All endpoints are **internal**. Authenticate with Service JWT (`Authorization: Bearer <svc-jwt>`). Mutations require `Idempotency-Key`.

### Wallets

* `GET  /internal/v1/wallets/:userId/balances` → `{ balances: { CURRENCY: integer } }`
* `GET  /internal/v1/ledger?userId&cursor&limit` → paginated entries

### Holds

* `POST /internal/v1/holds`

```json
{
  "userId":"u_1", "currency":"FZ", "amount":100,
  "reason":"matchmaking|cfb|escrow|withdrawal|campaign",
  "reference":"<freeform>",
  "idempotencyKey":"uuid"
}
```

→ `{ holdId:"h_abc", status:"held" }`

* `POST /internal/v1/holds/release`

```json
{ "holdId":"h_abc", "idempotencyKey":"uuid" }
```

→ `{ status:"released" }`

### Settlements

* `POST /internal/v1/settlements`

```json
{
  "holdId":"h_abc",
  "outcome":"win|loss",
  "payout": { "currency":"FZ", "amount": 200 },
  "roomId":"optional-correlation",
  "idempotencyKey":"uuid"
}
```

* **Rules**:

  * `win` credits payout to the same wallet and marks hold consumed.
  * `loss` debits the hold amount from wallet and marks hold consumed.
  * Payout currency must equal hold currency (MVP). Conversions handled separately.

### Conversions

* `POST /internal/v1/conversions`

```json
{
  "userId":"u_1",
  "from":"FUZE", "to":"FZ", "amount": 100,
  "rateSnapshotId":"rs_...",
  "feeBps":25,
  "idempotencyKey":"uuid"
}
```

→ `{ conversionId:"c_abc", toAmount: <int>, fee: <int> }`

### Withdrawals / Deposits (off-chain MVP)

* `POST /internal/v1/withdrawals` `{ userId, currency, amount, dest, idempotencyKey }`
* `POST /internal/v1/deposits/credit` `{ userId, currency, amount, source, idempotencyKey }`

---

## 4) Architecture

* **Node.js + TypeScript + Express**
* **MongoDB** (wallets, holds, ledger, conversions, withdrawals)
* **Redis** (idempotency store, locking, queues)
* **BullMQ** for async jobs (reconcilers, DLQ replays)
* **OpenTelemetry** metrics/traces; Pino logs

**Invariants**

* `Σ(user balances) + Σ(active holds) = Σ(ledger debits) - Σ(ledger credits)` per currency
* Each **hold** transitions: `held → (released | settled)`; never reused
* Each **settlement** is **idempotent** by `idempotencyKey`

---

## 5) Data Model (MVP)

* `WalletBalance` `{ userId, currency, available:int, updatedAt }`
* `Hold` `{ holdId, userId, currency, amount, reason, reference, status:held|released|settled, createdAt }`
* `LedgerEntry` `{ entryId, userId, currency, amount:int, type: debit|credit, reason, refId, createdAt }`
* `Settlement` `{ settlementId, holdId, outcome, payout?, createdAt }`
* `Conversion` `{ conversionId, userId, from, to, fromAmount, toAmount, fee, rateSnapshotId, createdAt }`
* `Withdrawal` `{ withdrawalId, userId, currency, amount, dest, status, createdAt }`

Indexes:

* `WalletBalance(userId,currency)` unique
* `Hold(holdId)` unique; `Hold(userId,currency,status)`
* `LedgerEntry(userId,currency,createdAt)`
* `Settlement(holdId)` unique
* `Conversion(userId,createdAt)`

---

## 6) ENV & Configuration

Create `.env` from template:

```dotenv
SERVICE_ID=payhub
SERVICE_NAME=tg-miniapp-payhub-service
PORT=8081
NODE_ENV=development
LOG_LEVEL=info

MONGO_URL=mongodb://localhost:27017/tg_payhub
REDIS_URL=redis://localhost:6379

# Auth
SERVICE_JWKS_URL=http://jwks.internal/.well-known/jwks.json
TRUSTED_ISSUERS=playhub,escrow,campaigns,funding,identity

# Rates & Fees
RATES_SOURCE=oracle           # price-service URL or static
CONVERSION_FEE_BPS_STAR_FZ=50
CONVERSION_FEE_BPS_FZ_STAR=50
CONVERSION_FEE_BPS_FUZE_FZ=25
CONVERSION_FEE_BPS_FUZE_PT=25
CONVERSION_FEE_BPS_FZ_PT=10

# Limits
LIMITS_DAILY_WITHDRAW_USDT=1000000
LIMITS_DAILY_CONVERT_COUNT=1000

# Reconciliation
RECON_ORPHAN_HOLDS_MINUTES=30
```

Secrets:

* JWKS (service keys) managed centrally; rotate quarterly.
* Use a secret manager; avoid storing in repo.

---

## 7) Dependencies

* `express`, `zod`, `mongodb`, `ioredis`, `bullmq`
* `jose` (JWT), `pino`, `pino-http`, `helmet`
* Dev: `typescript`, `tsx`, `vitest`/`jest`, `supertest`, `eslint`

---

## 8) Run & Develop

```bash
pnpm i
pnpm dev       # http://localhost:8081
```

Docker:

```bash
docker build -t tg-payhub:dev .
docker run --rm -p 8081:8081 --env-file .env tg-payhub:dev
```

Compose (example):

```yaml
services:
  payhub:
    build: .
    ports: ["8081:8081"]
    env_file: .env
    depends_on: [mongo, redis]
  mongo:
    image: mongo:7
  redis:
    image: redis:7
```

---

## 9) Testing

### What to test

* **Holds**: create → release; create → settle (win/loss)
* **Idempotency**: duplicate `Idempotency-Key` returns same result
* **Conversions**: fee application; rate snapshot integrity
* **Ledger**: invariant checks; per-currency totals

Run:

```bash
pnpm test
```

Target coverage: 85%+ on holds/settlements, 100% on idempotency core.

---

## 10) Deploy

Helm (excerpt):

```yaml
image: { repository: ghcr.io/yourorg/tg-miniapp-payhub-service, tag: "v0.1.0" }
resources:
  requests: { cpu: 200m, memory: 512Mi }
  limits:   { cpu: 1, memory: 1Gi }
probePaths: { liveness: /healthz, readiness: /readyz }
envFrom: secretRef: payhub-secrets
```

GitHub Actions:

* Build on tag, push to registry
* Trivy scan
* Deploy via ArgoCD

---

## 11) Operational Playbooks

* **Orphan holds GC**: sweep holds `status=held` older than `RECON_ORPHAN_HOLDS_MINUTES` with no active correlation; release.
* **Settlement partial failure**: retry with exponential backoff → DLQ → manual replay.
* **Rates outage**: block conversions; show maintenance flag via Config Service.

**SLOs**

* Availability 99.95%
* p95 latency < 100ms for `GET balances`, < 300ms for mutations under light load

---

## 12) Roadmap

* Bulk settlements endpoint
* On-chain custody adapter (EVM vault)
* Ledger snapshotting & point-in-time recovery
* Advanced fraud/velocity rules
