# tg-miniapp-discovery-service

Watchlists, price alerts, asset/project metadata, curated news, community links, events directory pointers, and lightweight “predict” signals for the Telegram Mini App. Pure **content & signals** layer — no balances or custody here.

---

## 1) Service Story

**Discovery** helps users track tokens and projects in one place. It stores user watchlists and alert rules, aggregates news/updates from approved sources, and exposes normalized project metadata (team, products, links, socials). It’s designed to be cache-first and provider-friendly: heavy fetches happen out-of-band and are served from our cache to the WebApp.

---

## 2) Duties & Scope

### Owns (SoR)

* **Watchlists**: per-user lists of assets/projects
* **Alerts**: price thresholds, percent‑move alerts, news mentions
* **Project Metadata**: team members, products, community links, docs, risk flags
* **News Feed**: normalized articles/announcements (title, source, URL, timestamp, tags)
* **Signals (predict)**: simple off‑chain signals (momentum, social mentions) with methodology tags

### Not owned here

* Wallets/ledger/conversions → **Payhub**
* CFB/Gameplay → **PlayHub**
* Event booking → external platforms (we only surface links)

---

## 3) Public API (MVP)

> All mutations require `Idempotency-Key` and a user session token. Read routes are cache‑friendly.

* `GET  /v1/discovery/assets?query=eth` → search assets/projects
* `GET  /v1/discovery/assets/:id` → metadata, links, risk flags
* `GET  /v1/discovery/news?assets=btc,eth&limit=50` → recent normalized news
* `GET  /v1/discovery/signals?asset=btc` → current signals & methodology
* `GET  /v1/discovery/watchlists` → user lists
* `POST /v1/discovery/watchlists` `{ name, items:[assetId] }`
* `POST /v1/discovery/watchlists/:listId/items` `{ assetId }`
* `DELETE /v1/discovery/watchlists/:listId/items/:assetId`
* `POST /v1/discovery/alerts` `{ assetId, type:"above|below|pct_change", value: number, window:"1h|24h" }`
* `GET  /v1/discovery/alerts` → user alerts

**Headers**

* `Authorization: Bearer <session-token>`
* `Idempotency-Key: <uuid>` on POST/DELETE
* `X-Request-Id: <uuid>`

---

## 4) Internal API & Jobs

* **Ingestors (workers)** pull from providers on schedules, normalize, and store in cache/DB
* `POST /internal/v1/alerts/dispatch` — worker endpoint to fan out alert notifications

---

## 5) Architecture

* **Node.js + TypeScript + Express**
* **PostgreSQL** (good for relational watchlists & alerts) or MongoDB (either works)
* **Redis** (caching: news lists, asset lookups; rate limiting)
* **BullMQ** for ingestors & alert dispatch
* **OpenTelemetry** + Pino
* **Zod** for DTOs

**Cache-first pattern**

* Read paths serve from Redis with TTLs & ETags
* Ingestors refresh caches on schedule (e.g., news every 2–5 min; signals every 1–5 min)

---

## 6) Data Model (sample)

* `Asset` `{ assetId, symbol, name, category, riskFlags[], socials{ twitter, telegram, discord }, website, docs, createdAt }`
* `Project` `{ projectId, team[], products[], communities[], tags[] }` (optional split)
* `NewsItem` `{ id, assetIds[], source, title, url, publishedAt, tags[] }`
* `Watchlist` `{ listId, userId, name, items:[assetId], createdAt }`
* `Alert` `{ alertId, userId, assetId, type, value, window, active, createdAt }`
* `Signal` `{ assetId, metric, value, window, updatedAt, methodology }`

Indexes tuned for: `userId`, `assetId`, `publishedAt DESC`

---

## 7) ENV & Configuration

```dotenv
SERVICE_ID=discovery
SERVICE_NAME=tg-miniapp-discovery-service
PORT=8093
NODE_ENV=development
LOG_LEVEL=info

# DBs
PG_URL=postgres://user:pass@localhost:5432/tg_discovery
REDIS_URL=redis://localhost:6379

# Providers (optional, Phase‑1 can be mock/stub)
PROVIDER_NEWS=coindesk,cointelegraph,official_blogs
PROVIDER_SIGNALS=social_mentions,price_volatility
FETCH_CONCURRENCY=4

# Alerts
ALERTS_MAX_PER_USER=50
ALERTS_WINDOW_ALLOWED=1h,24h
```

Secrets: API keys for providers (when used) must be injected via secret manager; never checked in.

---

## 8) Dependencies

* `express`, `zod`, `pg` or `mongodb`, `ioredis`, `bullmq`
* `pino`, `pino-http`, `helmet`, `cors`, `ky` (fetch)
* Dev: `typescript`, `tsx`, `vitest`/`jest`, `supertest`, `eslint`

---

## 9) Develop & Run

```bash
pnpm i
pnpm dev       # http://localhost:8093
```

Docker/Compose similar to other services; include Postgres and Redis.

---

## 10) Testing

* **API**: watchlist CRUD, alerts CRUD, cache headers (ETag/TTL)
* **Ingestors**: rate‑limit behavior, dedupe, provider failure fallback
* **Alerts**: trigger conditions, throttling, notification enqueue

Run:

```bash
pnpm test
```

---

## 11) Deploy

Helm (excerpt):

```yaml
image: { repository: ghcr.io/yourorg/tg-miniapp-discovery-service, tag: "v0.1.0" }
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
probePaths: { liveness: /healthz, readiness: /readyz }
```

---

## 12) Roadmap

* Full-text search over assets/news
* Per‑user daily briefing email/notification
* Provider adapters with backfill & SLA monitors
* Graph linking of projects ↔ teams ↔ products
