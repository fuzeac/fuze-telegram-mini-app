# tg-miniapp-playhub-service

Matchmaking, game orchestration, fairness, and **CFB v1 (Crypto Fair Bet)** for the Telegram Mini App. PlayHub coordinates gameplay and price bets while delegating custody to **Payhub**.

---

## 1) Service Story

**PlayHub** provides trusted, low-latency gameplay and price-based betting. It never touches balances directly; instead, it places **holds** and executes **settlements** via Payhub. For head‑to‑head games, it issues **GST** (Game Session Tokens) for client→game authentication. For **CFB v1**, it runs oracle-driven settlement with a transparent 7% rake.

---

## 2) Duties & Scope

### Owns (SoR)

* **Game Catalog & Sessions** (IDs, rules, stake bounds, lifecycle)
* **Matchmaking** pools & room creation (server‑to‑server with game services)
* **GST** mint/verify (Ed25519)
* **CFB v1**: bet creation/acceptance, price sampling, settlement receipts

### Not owned here

* Wallets/ledger/conversions → **Payhub**
* Identity/auth sessions → **Identity Service**
* Independent game logic servers (e.g., “Game A”) → separate services

---

## 3) Public API (MVP)

> All mutations accept `Idempotency-Key`. Public requests require a user session token.

### Games (H2H)

* `GET  /v1/games` → game list (stake limits, currency)
* `POST /v1/matchmaking/join` `{ gameId, bet:{amount,currency} }` → waiting | matched `{ roomId, gst, redirectUrl }`
* `GET  /v1/games/results/:roomId` → final result receipt

### CFB v1 (Crypto Fair Bet; STAR/FZ/PT only)

* `POST /v1/cfb/bets` `{ asset, quote:"USD", currency, stake:{amount}, condition:{op,target|"spot@create"}, duration, closeAcceptAt }`
* `GET  /v1/cfb/bets/:betId`
* `POST /v1/cfb/bets/:betId/accept` `{ stake:{amount} }`
* `GET  /v1/cfb/bets` (filters: `asset`, `currency`, `sort`)
* `GET  /v1/cfb/bets/:betId/result`

**Headers**

* `Authorization: Bearer <session-token>`
* `Idempotency-Key: <uuid>` (required for POST)
* `X-Request-Id: <uuid>` (tracing)

---

## 4) Internal API (Service↔Service)

### PlayHub → Game services

* `POST /internal/v1/rooms` `{ roomId, players:[userId], gameId }` (idempotent; Authorization: service JWT)

### Game services → PlayHub

* `POST /internal/v1/results` `{ roomId, winnerId, evidence }` (idempotent by `roomId`)

### PlayHub → Payhub

* `POST /internal/v1/holds` / `.../holds/release`
* `POST /internal/v1/settlements`

### CFB v1 jobs

* `POST /internal/v1/cfb/settle` `{ betId }` (scheduler/worker invokes at/after T\_end)

---

## 5) Architecture

* **Node.js + TypeScript + Express**
* **MongoDB**: `GameSession`, `Match`, `MatchTicket`, `CfbBet`, `CfbAcceptance`, `CfbResult`, `Idempotency`
* **Redis**: matchmaking pools, room & result cache, locks, idempotency cache
* **BullMQ**: `cfb.settle`, `games.settlement`, reconcilers
* **OpenTelemetry** + Pino logs
* **Zod** schemas for strict request/response types

**Key concepts**

* **GST**: Ed25519 JWT with claims `{ iss:playhub, aud:<game-id>, sub:user:<id>, roomId, gameId, iat, exp≤10m, jti }`.
* **Matchmaking**: sorted pools per `{gameId,currency,amount}` with NX locks to pair players.
* **CFB v1**: Owner vs Pool; rake 7% from losing side; integer math with remainder distribution.

---

## 6) Data Model (high level)

* `MatchTicket` `{ ticketId, userId, gameId, bet{amount,currency}, holdId, status, createdAt }`
* `Match` `{ roomId, gameId, players:[userId], holdIds{userId:holdId}, bet, status, createdAt }`
* `GameSession` `{ roomId, gstJtis:[string], createdAt, expiresAt }`
* `CfbBet` `{ betId, ownerId, asset, quote, currency, stakeOwner, S_acc, condition, refPriceAtCreate?, T_create, T_end, closeAcceptAt, ownerHoldId, status }`
* `CfbAcceptance` `{ betId, userId, stake, holdId, createdAt }`
* `CfbResult` `{ betId, winnerSide, rakeBps, priceReport{samples[], twap, hash}, payouts{ownerAmount, acceptorBreakdown[]}, settlementIds[], finalizedAt }`

Indexes optimized for: `roomId`, `betId`, `status`, `T_end`, `userId`

---

## 7) ENV & Configuration

Create `.env`:

```dotenv
SERVICE_ID=playhub
SERVICE_NAME=tg-miniapp-playhub-service
PORT=8082
NODE_ENV=development
LOG_LEVEL=info

MONGO_URL=mongodb://localhost:27017/tg_playhub
REDIS_URL=redis://localhost:6379

# JWTs
PLAYHUB_JWT_PRIVATE_JWK={"kty":"OKP","crv":"Ed25519","d":"REPLACE","x":"REPLACE"}
PLAYHUB_JWT_ISSUER=playhub
PLAYHUB_GST_AUDIENCE_PREFIX=game-

# Peers
SVC_PAYHUB_URL=http://svc-payhub:8081
TRUSTED_GAME_ISSUERS=game-a,game-b

# CFB
CFB_ENABLED=true
CFB_RAKE_BPS=700
CFB_ALLOWED_CURRENCIES=STAR,FZ,PT
CFB_ASSETS=BTC,ETH
CFB_ORACLE_BASE_URL=http://price-service:8085
CFB_ORACLE_GRACE_SECONDS=120

# Matchmaking & Limits
MM_MAX_PENDING_MINUTES=10
JOIN_RATE_LIMIT_PER_10S=5
```

Secrets:

* Keep the PlayHub private JWK in a secret manager; rotate quarterly.
* Maintain allow‑lists: trusted game service issuers and audiences.

---

## 8) Dependencies

* `express`, `zod`, `ioredis`, `mongodb`, `bullmq`
* `jose` (JWT), `pino`, `pino-http`, `helmet`, `cors`
* Dev: `typescript`, `tsx`, `vitest`/`jest`, `supertest`, `eslint`

---

## 9) Run & Develop

```bash
pnpm i
pnpm dev           # http://localhost:8082
```

Docker:

```bash
docker build -t tg-playhub:dev .
docker run --rm -p 8082:8082 --env-file .env tg-playhub:dev
```

Compose (example):

```yaml
services:
  playhub:
    build: .
    ports: ["8082:8082"]
    env_file: .env
    depends_on: [mongo, redis]
  mongo:
    image: mongo:7
  redis:
    image: redis:7
```

---

## 10) Testing

### Suites

* **GST**: mint/verify; WS auth happy/invalid paths
* **Matchmaking**: join→wait, join→pair, TTL expiry, idempotency
* **CFB v1**: create, accept, close, price TWAP, settle owner/accept, tie/push, oracle failure refund
* **Payhub contracts**: mocked integration for holds/settlements

Run:

```bash
pnpm test
```

Coverage target: 85%+ overall; **100%** on payout math utilities.

---

## 11) Deploy

Helm (excerpt):

```yaml
image: { repository: ghcr.io/yourorg/tg-miniapp-playhub-service, tag: "v0.1.0" }
resources:
  requests: { cpu: 200m, memory: 512Mi }
  limits:   { cpu: 1, memory: 1Gi }
probePaths: { liveness: /healthz, readiness: /readyz }
```

GitHub Actions (suggested): build on tag → scan → push → GitOps deploy.

---

## 12) Operational Notes

* **SLOs**: 99.9% availability; p95 `join` < 150ms, `cfb.settle` end‑to‑end < 2s.
* **DLQs**: `cfb.settle.dlq`, `games.settlement.dlq` with manual replay job.
* **Reconciliation**: nightly sweep for orphan holds (unpaired tickets, failed settlements).
* **Observability**: emit events `matchmaking.matched`, `cfb.bet_created`, `cfb.settled` with `roomId`/`betId` correlation.

---

## 13) Roadmap

* Multi‑venue oracle with trimmed‑mean TWAP
* More game adapters & GST device binding
* On‑chain adapters (custody/oracle) using the same domain events
