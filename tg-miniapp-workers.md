# tg-miniapp-workers

Background workers for asynchronous, scheduled, and retryable jobs across the Telegram Mini App platform. Implements settlement pipelines, reconcilers, DLQ replays, and periodic samplers. Runs independently from API services but talks to them via internal APIs.

---

## 1) Service/Repo Story

Many operations are not request/response (e.g., settlement retries, reconciliation, price sampling, queue draining). **tg-miniapp-workers** centralizes these background tasks with observable queues and predictable retry semantics. It does **not** own business data; it orchestrates by calling domain services (Payhub, PlayHub, Discovery, Campaigns, Escrow) and updating their state via internal APIs.

---

## 2) Duties & Scope

### Owns

* **Job definitions & queues** for:

  * `cfb.settle` — trigger PlayHub CFB settlement at/after `T_end`
  * `cfb.price.sample` — optional sampler if no dedicated Price Service (writes to PlayHub/Redis)
  * `games.settlement` — finalize head‑to‑head games on callback failure
  * **Reconciliation jobs**:

    * `payhub.orphan_holds.gc` — release holds older than TTL with no correlation
    * `payhub.ledger.recalc` — (optional) verify invariants & emit alerts
  * **Campaigns**: `campaigns.verify_async`, `campaigns.claim`
  * **Escrow**: `escrow.expire`, `escrow.adjudicate`
  * **DLQ replayer**: re‑enqueue failed jobs after operator review
* **Schedules** (cron) and concurrency settings
* **Operational tooling**: queue stats, pause/resume, drain, backfill

### Not owned here

* Ledger/balances — **Payhub**
* CFB/Game logic — **PlayHub**
* Long‑term price history — use **Price Service** if present

---

## 3) Architecture

* **Node.js + TypeScript** workers
* **BullMQ** on **Redis** as the queue broker
* **Pino** logging + **OpenTelemetry** traces/metrics
* Config pulled from **tg-miniapp-config** (hot‑reload)

**Process model**

* Single container image; multiple **named processors** can be enabled/disabled via env flags.
* Each processor registers queues with dedicated concurrency and backoff strategies.

---

## 4) ENV & Configuration

Create `.env`:

```dotenv
SERVICE_ID=workers
SERVICE_NAME=tg-miniapp-workers
PORT=8099                     # optional health/metrics port
NODE_ENV=development
LOG_LEVEL=info

REDIS_URL=redis://localhost:6379

# Peers (internal APIs)
SVC_PAYHUB_URL=http://svc-payhub:8081
SVC_PLAYHUB_URL=http://svc-playhub:8082
SVC_CAMPAIGNS_URL=http://svc-campaigns:8094
SVC_ESCROW_URL=http://svc-escrow:8092
SVC_DISCOVERY_URL=http://svc-discovery:8093
SVC_PRICE_URL=http://svc-price:8085      # optional

# Auth
SERVICE_JWT_PRIVATE_JWK={"kty":"OKP","crv":"Ed25519","d":"REPLACE","x":"REPLACE"}
SERVICE_JWT_ISSUER=workers

# Feature toggles (enable processors)
ENABLE_CFB_SETTLE=true
ENABLE_PRICE_SAMPLER=false
ENABLE_GAMES_SETTLEMENT=true
ENABLE_RECON_ORPHAN_HOLDS=true
ENABLE_CAMPAIGNS=true
ENABLE_ESCROW=true

# Schedules (cron or seconds)
CRON_CFB_PRICE_SAMPLE=*/1 * * * *      # every minute
CRON_RECON_ORPHAN_HOLDS=*/10 * * * *   # every 10 minutes

# Queues
QUEUE_PREFIX=tg
MAX_ATTEMPTS_DEFAULT=10
BACKOFF_DEFAULT_MS=2000
DLQ_SUFFIX=.dlq
```

Secrets: keep the private JWK in a secret manager; never commit.

---

## 5) Dependencies

* `bullmq`, `ioredis`
* `ky` or `axios` (service HTTP calls)
* `pino`, `@opentelemetry/api`, `@opentelemetry/sdk-node`
* `dotenv`, `zod`, `jose`
* Dev: `typescript`, `tsx`, `vitest`/`jest`, `eslint`

---

## 6) How to Run (Dev)

```bash
pnpm i
pnpm dev            # starts processors enabled by env
```

Run a specific processor only:

```bash
PROCESSORS=cfb.settle,escrow pnpm dev
```

Docker:

```bash
docker build -t tg-workers:dev .
docker run --rm --env-file .env tg-workers:dev
```

---

## 7) Job Specs (MVP)

### `cfb.settle`

* **When**: At/after `T_end` for each open CFB bet
* **Input**: `{ betId }`
* **Action**: Call `POST PLAYHUB /internal/v1/cfb/settle` with service JWT
* **Retry**: exponential backoff, max 10 attempts → DLQ

### `cfb.price.sample` (optional)

* **When**: every minute
* **Action**: fetch prices from `SVC_PRICE_URL` or external provider; store minute bars in PlayHub/Redis

### `payhub.orphan_holds.gc`

* **When**: every 10 minutes
* **Action**: scan for holds older than TTL without correlation; release via Payhub

### `games.settlement`

* **When**: on callback failure from a game server
* **Action**: finalize a match by calling Payhub settlements using PlayHub state

### `campaigns.verify_async` / `campaigns.claim`

* **When**: async proofs or queued claims
* **Action**: verify proof; issue reward via Payhub; record receipt

### `escrow.expire` / `escrow.adjudicate`

* **When**: at `expiresAt` or moderator decision
* **Action**: release/settle holds accordingly

---

## 8) Health, Metrics & Admin

* **/healthz** HTTP probe port (optional)
* Metrics: queue depths, processing rates, success/failure counters, retry backoff histograms
* DLQ utilities: requeue, drop, export (via small CLI or admin endpoints)

---

## 9) Testing

* Unit: backoff calculation, JWT signer, request wrapper, idempotent semantics
* Integration: spin up Redis and mock HTTP servers for PlayHub/Payhub; assert retries & DLQ routing

Run:

```bash
pnpm test
```

---

## 10) Deploy

Helm (excerpt):

```yaml
image: { repository: ghcr.io/yourorg/tg-miniapp-workers, tag: "v0.1.0" }
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
envFrom:
  - secretRef: workers-secrets
```

Horizontal scaling:

* Scale **processors with partitions** (BullMQ group keys) for high‑volume topics.
* Ensure idempotent external calls (all service mutations accept `Idempotency-Key`).

---

## 11) Roadmap

* Web UI for queue inspection in Admin
* Per‑job SLOs & alert rules (pager on backlog/latency)
* Multi‑region active‑active with Redis Enterprise
