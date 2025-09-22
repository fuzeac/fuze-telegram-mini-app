# tg-miniapp-game-service-template

Boilerplate for independent **game backends** that integrate with **tg-miniapp-playhub-service**. Implements room creation (idempotent), GST‑based player auth, realtime gameplay (WebSocket), and secure server→server result callback.

> Use this template to build “Game A”, “Game B”, etc. Each game runs its own logic but follows the same contracts.

---

## 1) Service Story

Game services own **gameplay** while delegating custody/settlement to PlayHub → Payhub. PlayHub creates rooms and issues **GST** (Game Session Tokens) for players. The game verifies GST on connect, runs the match, determines the result, and reports the winner back to PlayHub **server‑to‑server** (no client can forge results).

---

## 2) Duties & Scope

### Owns

* Game rules/state machines
* Room registry & membership
* WebSocket server for realtime play
* Result determination & secure callback to PlayHub

### Does **not** own

* Wallet/ledger (Payhub)
* Matchmaking (PlayHub)
* GST minting (PlayHub)

---

## 3) External Contracts

### PlayHub → Game

* `POST /internal/v1/rooms` `{ roomId, gameId, players:[userId] }` (idempotent)

  * Auth: **service JWT** (`iss=playhub`, `aud=<this-game-id>`)
  * Response `{ ok:true }` and persist room

### Client → Game

* **WebSocket** `GET /ws?roomId=<id>` with header `Authorization: Bearer <GST>`

  * Verify GST (Ed25519 JWT) with claims `{ iss:playhub, aud:<this-game-id>, sub:user:<id>, roomId, exp }`
  * Reject if not in room, expired, or wrong audience

### Game → PlayHub

* `POST PLAYHUB /internal/v1/results` `{ roomId, winnerId, evidence }`

  * Auth: **service JWT** (`iss=<this-game-id>`, `aud=playhub`)
  * Idempotent by `roomId`

---

## 4) API (This Service)

> Besides `/internal/v1/rooms`, keep the external surface minimal. Gameplay rides over WS.

* `GET  /healthz` — liveness probe
* `GET  /readyz` — readiness probe
* `POST /internal/v1/rooms` — idempotent room creation (see above)
* `GET  /debug/rooms/:roomId` — (dev only) inspect room state

---

## 5) Architecture

* **Node.js + TypeScript + Fastify/Express**
* **ws** (WebSocket) or **socket.io**
* **Redis** for ephemeral room state / pubsub (optional)
* **MongoDB** (optional) for match archives
* **OpenTelemetry** + Pino logs
* **Zod** DTOs

**Room lifecycle (suggested)**

1. `POST /internal/v1/rooms` creates `{ roomId, players, status: open }` (idempotent key = `roomId`)
2. First player connects via WS (GST OK) → mark present
3. Second player connects (GST OK) → `status: in_progress`
4. Run game loop (tick/turns). On finish → determine `winnerId`
5. Call PlayHub `/internal/v1/results` with `Idempotency-Key: roomId`
6. Transition `status: finished`; close sockets

---

## 6) ENV & Configuration

```dotenv
SERVICE_ID=game-a
SERVICE_NAME=tg-miniapp-game-service-template
PORT=8086
NODE_ENV=development
LOG_LEVEL=info

# JWTs
SERVICE_JWT_PRIVATE_JWK={"kty":"OKP","crv":"Ed25519","d":"REPLACE","x":"REPLACE"}
SERVICE_JWT_ISSUER=game-a
PLAYHUB_JWKS_URL=http://svc-playhub:8082/.well-known/jwks.json
PLAYHUB_RESULTS_URL=http://svc-playhub:8082/internal/v1/results

# Data stores (optional)
REDIS_URL=redis://localhost:6379
MONGO_URL=mongodb://localhost:27017/tg_game_a

# Gameplay
TICK_MS=100
ROOM_TTL_MINUTES=30
```

Secrets: keep private JWK in a secret manager.

---

## 7) Dependencies

* `fastify` or `express`, `zod`
* `ws` or `socket.io`
* `ioredis`, `mongodb` (optional)
* `jose` (JWT verify/sign), `pino`, `pino-http`
* Dev: `typescript`, `tsx`, `vitest`/`jest`, `supertest`, `eslint`

---

## 8) Develop & Run

```bash
pnpm i
pnpm dev           # http://localhost:8086
```

Start WS client (example):

```js
const ws = new WebSocket("ws://localhost:8086/ws?roomId=r_123", { headers: { Authorization: `Bearer ${gst}` } });
```

Docker/Compose similar to other services.

---

## 9) Testing

* **JWT**: GST verification (valid/expired/wrong aud)
* **Rooms**: idempotent create, duplicate POST behavior
* **Gameplay**: determinism of winner for test seeds
* **Callback**: success and retry to PlayHub with `Idempotency-Key`

Run:

```bash
pnpm test
```

---

## 10) Deploy

Helm (excerpt):

```yaml
image: { repository: ghcr.io/yourorg/tg-miniapp-game-service-template, tag: "v0.1.0" }
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
probePaths: { liveness: /healthz, readiness: /readyz }
```

---

## 11) Security & Ops

* Allow‑list **PlayHub** IPs / service account for `/internal/*`
* Close WS if GST reused (track `jti` per room)
* Emit domain events: `game.room_created`, `game.started`, `game.finished`

---

## 12) Roadmap

* Spectator channels with rate‑limited broadcasts
* Realtime metrics (tick lag, dropped frames)
* Deterministic replays for audits
