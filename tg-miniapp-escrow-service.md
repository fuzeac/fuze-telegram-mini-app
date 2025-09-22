# tg-miniapp-escrow-service

OTC/P2P **escrow** engine for off‑chain custody (STAR/FZ/PT/FUZE/USDT). Creates bilateral trades with funds locked in **Payhub** holds, and settles when both sides fulfill conditions. Disputes are supported with staff adjudication via Admin.

---

## 1) Service Story

**Escrow Service** enables safe peer‑to‑peer exchanges (e.g., OTC buys, swaps) by orchestrating **dual holds** in Payhub, enforcing a state machine with timeouts, and releasing funds atomically on completion. It’s business‑rule light: it doesn’t price assets or handle on‑chain custody in v1; those are future adapters.

---

## 2) Duties & Scope

### Owns (SoR)

* **Escrow Deals**: lifecycle, participants, terms, deadlines
* **Disputes**: evidence references, status, resolution
* **Compliance Flags**: simple AML/KYC checks (hooks)

### Not owned here

* Ledger balances/settlements → **Payhub**
* Identity, KYC status → **Identity Service** / external KYC
* Price discovery → **Discovery** (optional)

---

## 3) Public API (MVP)

> All mutations require `Idempotency-Key` and a user session token.

* `POST /v1/escrows` — create draft escrow

```json
{
  "maker": { "userId": "u_maker", "currency": "FZ", "amount": 500 },
  "taker": { "currency": "FZ", "amount": 500 },
  "expiresAt": "2025-09-30T12:00:00Z",
  "terms": { "type": "p2p", "notes": "Local meet-up" }
}
```

→ `{ escrowId, status:"pending_accept", makerHoldId }` (places maker hold)

* `POST /v1/escrows/:escrowId/accept` — accept & place taker hold

```json
{ "currency": "FZ", "amount": 500 }
```

→ `{ takerHoldId, status:"funded" }`

* `POST /v1/escrows/:escrowId/complete` — both parties confirm delivery → release
  → `{ status:"released" }`

* `POST /v1/escrows/:escrowId/cancel` — cancel (rules below)
  → `{ status:"cancelled" }`

* `POST /v1/escrows/:escrowId/dispute` — open dispute with evidence links
  → `{ status:"disputed" }`

**Cancel rules (MVP)**

* Before taker funds: maker may cancel → release maker hold.
* After both funded: cancel requires **both** parties or staff resolution.
* After `expiresAt`: either party may request cancel → staff review or auto rules.

---

## 4) Internal API

* `POST /internal/v1/escrows/settle` `{ escrowId }` — internal settle path (workers/admin)
* **Payhub**: `/internal/v1/holds`, `/internal/v1/holds/release`, `/internal/v1/settlements`
* **Identity** (optional): `/v1/users/:id/kyc` read

---

## 5) State Machine

`pending_accept → funded → (completed | disputed | expired) → (released | refunded | adjudicated)`

* **pending\_accept**: maker hold placed
* **funded**: taker hold placed
* **completed**: both confirm delivery →

  * settle taker hold as **loss**, credit maker (`win`) *or* symmetric depending on terms
* **disputed**: staff resolves → partial or full outcomes
* **expired**: past `expiresAt` without completion → refund logic

> Settlement mapping is defined by `terms.type`. MVP supports **same-currency escrow** and **simple swap** (A↔B amounts) by running two independent escrows or a combined atomic path.

---

## 6) Architecture

* **Node.js + TS + Express**
* **MongoDB**: `Escrow`, `Dispute`, `Audit`
* **Redis**: locks, idempotency cache
* **BullMQ**: `escrow.expire`, `escrow.adjudicate`
* **OpenTelemetry** + Pino
* **Zod** for DTOs

**Security**

* Service JWTs for internal calls; roles from Identity
* PII minimized; only userId references and evidence URLs/hashes

---

## 7) Data Model

* `Escrow` `{ escrowId, maker{userId,currency,amount,holdId}, taker{userId?,currency,amount,holdId?}, status, expiresAt, terms, createdAt, updatedAt }`
* `Dispute` `{ escrowId, openedBy, reason, evidence:[{url,hash}], status, resolution?, createdAt }`
* `Audit` `{ escrowId, actor, action, at, payloadHash }`

Indexes: `Escrow(status,expiresAt)`, `Escrow(maker.userId)`, `Escrow(taker.userId)`

---

## 8) ENV & Configuration

```dotenv
SERVICE_ID=escrow
SERVICE_NAME=tg-miniapp-escrow-service
PORT=8092
NODE_ENV=development
LOG_LEVEL=info

MONGO_URL=mongodb://localhost:27017/tg_escrow
REDIS_URL=redis://localhost:6379

SVC_PAYHUB_URL=http://svc-payhub:8081
SVC_IDENTITY_URL=http://svc-identity:8080

# Business
ESCROW_TTL_HOURS=48
DISPUTE_AUTO_REFUND_AFTER_HOURS=72
ALLOWED_CURRENCIES=STAR,FZ,PT,FUZE,USDT
```

---

## 9) Dependencies

* `express`, `zod`, `mongodb`, `ioredis`, `bullmq`
* `jose`, `pino`, `pino-http`, `helmet`, `cors`
* Dev: `typescript`, `tsx`, `vitest`/`jest`, `supertest`, `eslint`

---

## 10) Develop & Run

```bash
pnpm i
pnpm dev       # http://localhost:8092
```

Docker/Compose similar to other services; link to Mongo/Redis/Payhub.

---

## 11) Testing

* **Lifecycle**: create → accept → complete → released
* **Cancel rules**: before/after funding
* **Dispute**: open → adjudicate → settlement
* **Idempotency** on all POSTs

Run:

```bash
pnpm test
```

---

## 12) Deploy

Helm (excerpt):

```yaml
image: { repository: ghcr.io/yourorg/tg-miniapp-escrow-service, tag: "v0.1.0" }
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
probePaths: { liveness: /healthz, readiness: /readyz }
```

---

## 13) Roadmap

* On‑chain vault adapter (EVM) with same state machine
* Escrow templates (milestone‑based, deliverables)
* Reputation/ratings integration
