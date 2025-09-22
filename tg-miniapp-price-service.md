# tg-miniapp-price-service

Centralized **price sampler & normalizer** for assets used by the Telegram Mini App (CFB settlement, discovery alerts, dashboards). Produces signed minute bars and TWAP endpoints with provider-aware fallbacks and caching.

> Optional but recommended. If not deployed, PlayHub can fetch prices directly from providers; this service standardizes logic and improves observability.

---

## 1) Service Story

**Price Service** ingests quotes from one or more upstream providers (aggregators like Chainlink/Pyth/Kaiko/CoinAPI or exchange APIs), normalizes them to a canonical schema, and exposes **minute bars** and **TWAP** queries for downstream consumers. It also signs price reports so CFB results are auditable.

---

## 2) Duties & Scope

### Owns (SoR)

* **Minute bars**: OHLCV at 1‑minute resolution (in‑memory cache + persisted window)
* **Price reports**: signed blobs with sample set and hashing for audits
* **Provider adapters**: pluggable fetchers with health metrics and rate limits

### Not owned here

* Business state (bets, alerts) → PlayHub/Discovery
* Ledger/custody → Payhub

---

## 3) Public/Internal API (MVP)

> All endpoints require a service JWT (`Authorization: Bearer <svc-jwt>`). Read‑only.

* `GET /internal/v1/prices/bars?asset=BTC&quote=USD&from=ISO&to=ISO`

  * Returns 1‑minute bars $inclusive of `from`, exclusive of `to`$
* `GET /internal/v1/prices/twap?asset=BTC&quote=USD&t=ISO&windowSec=60`

  * Returns `{ twap, samples: [...], hash, signedReport }`
* `GET /internal/v1/prices/providers` → health & latency stats

**Bar schema**

```json
{ "t": "2025-09-22T10:34:00Z", "o": 64000.12, "h": 64010.00, "l": 63990.10, "c": 64005.55, "v": 23.4 }
```

---

## 4) Architecture

* **Node.js + TypeScript + Fastify/Express**
* **Redis** for hot bar cache; **PostgreSQL/TimescaleDB** or **MongoDB** for short retention (e.g., 7–30 days)
* **BullMQ** sampler job every minute
* **OpenTelemetry** metrics/traces; Pino logs
* **Zod** for DTOs

**Sampling**

* Poll multiple providers concurrently; apply **trimmed mean** or **median** on last trade/mid price.
* If provider count < threshold or data stale beyond `STALE_SEC`, mark bar as **partial**; downstream may reject.
* Write resulting bar; sign optional report for exact sample set used.

---

## 5) Data Model

* `Bar` `{ asset, quote, t, o, h, l, c, v, sourceCount, staleFlags }`
* `Report` `{ id, asset, quote, tStart, tEnd, samples:[{provider, t, price}], twap, hash, sig }`
* `ProviderHealth` `{ provider, latencyMs, errorRate, lastOkAt }`

Indexes: `(asset,quote,t)`

---

## 6) ENV & Configuration

```dotenv
SERVICE_ID=price
SERVICE_NAME=tg-miniapp-price-service
PORT=8085
NODE_ENV=development
LOG_LEVEL=info

REDIS_URL=redis://localhost:6379
PG_URL=postgres://user:pass@localhost:5432/tg_price

# Providers (comma‑sep list of enabled adapters)
PROVIDERS=chainlink,pyth,coinapi
PROVIDER_CHAINLINK_URL=https://chain.link/api
PROVIDER_PYTH_URL=https://hermes.pyth.network
PROVIDER_COINAPI_KEY=replace_me

# Sampling
SAMPLE_WINDOW_SECONDS=60
STALE_PRICE_SECONDS=10
REQUIRED_PROVIDER_MIN=2
OUTLIER_TRIM_BPS=50

# Signing
REPORT_SIGNING_PRIVATE_JWK={"kty":"OKP","crv":"Ed25519","d":"REPLACE","x":"REPLACE"}
REPORT_SIGNING_KEY_ID=price-v1
```

Secrets: provider keys and signing key via secret manager.

---

## 7) Dependencies

* `fastify` or `express`, `zod`
* `ioredis`, `pg` (or `mongodb`)
* `bullmq`
* `jose` (signing), `pino`, `pino-http`
* `undici`/`ky` for HTTP
* Dev: `typescript`, `tsx`, `vitest`/`jest`, `eslint`

---

## 8) Run & Develop

```bash
pnpm i
pnpm dev           # http://localhost:8085
```

Docker similarly to other services; include Postgres and Redis in Compose.

---

## 9) Testing

* **Sampler**: multi‑provider merge, outlier trim, stale detection
* **TWAP**: exact window math, rounding
* **Reports**: signature verifies against public JWK

Run:

```bash
pnpm test
```

---

## 10) Deploy

Helm (excerpt):

```yaml
image: { repository: ghcr.io/yourorg/tg-miniapp-price-service, tag: "v0.1.0" }
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
probePaths: { liveness: /healthz, readiness: /readyz }
```

---

## 11) Roadmap

* WebSocket streaming for live ticks
* On‑chain price oracles adapter (for parity with future on‑chain settle)
* Historical backfill and export tooling
