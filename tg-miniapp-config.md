# tg-miniapp-config

Centralized, versioned **configuration & feature flags** for the Telegram Mini App platform. Produces signed, cacheable config bundles consumed by all services (Identity, Payhub, PlayHub/CFB, Discovery, Campaigns, Escrow, Events, WebApp, Admin).

---

## 1) Repo Story

This repo defines the **single source of truth** for operational settings: feature toggles, fees, stake limits, token lists, game catalogs, CFB rules, rate limits, and environment endpoints. Config is kept in human‑readable **YAML**, validated against a schema, then **published** as versioned artifacts (JSON) to a CDN or Config Service endpoint. Services hot‑reload on a short interval.

---

## 2) Duties & Scope

### Owns

* **Config schema** (Zod/AJV): structure, validation rules
* **Environment overlays**: `base/`, `dev/`, `stage/`, `prod/`
* **Publisher CLI**: validate → sign → publish → provenance hash
* **Public subset** for WebApp (non‑sensitive flags/limits)

### Does **not** own

* Secret values (API keys, private JWKs) → secret manager
* Business state (wallets, bets, campaigns) → services’ databases

---

## 3) Layout

```
config/
  base/
    app.yaml
    games.yaml
    cfb.yaml
    currencies.yaml
    limits.yaml
    rates.yaml
  dev/
    overlay.yaml
  stage/
    overlay.yaml
  prod/
    overlay.yaml
schema/
  index.ts          # Zod schema
scripts/
  publish.ts        # CLI: validate → sign → upload
  diff.ts           # show diff between versions
public/
  readme.md         # docs for public consumers
```

---

## 4) Example Config (snippets)

### `config/base/cfb.yaml`

```yaml
cfb:
  enabled: true
  currencies: [STAR, FZ, PT]
  rakeBps: 700
  assets: [BTC, ETH]
  oracle:
    decision: twap_1m_center
    graceSeconds: 120
```

### `config/base/games.yaml`

```yaml
games:
  - id: game-a
    title: Coin Flip
    currencies: [FZ]
    minStake: 1
    maxStake: 1000
```

### `config/base/limits.yaml`

```yaml
rateLimits:
  joinMatchmakingPer10s: 5
  campaignsClaimsPerDay: 5
```

---

## 5) ENV & Publisher Settings

Create `.env` for local publishing:

```dotenv
# Publishing target (choose one)
PUBLISH_S3_BUCKET=tg-config-dev
PUBLISH_S3_PREFIX=config/
# or
PUBLISH_GCS_BUCKET=tg-config-dev
PUBLISH_GCS_PREFIX=config/

# CDN (served read‑only to services)
CDN_BASE_URL=https://cdn.example.com/tg-config

# Signing (optional but recommended)
SIGNING_PRIVATE_JWK={"kty":"OKP","crv":"Ed25519","d":"REPLACE","x":"REPLACE"}
SIGNING_KEY_ID=config-v1

# Metadata
PUBLISHER_NAME=ops-bot
```

> Secrets live in your secret manager and are injected at CI/CD time; do not commit to git.

---

## 6) Dependencies

* Runtime: `zod` (schema), `yaml`, `fast-glob`, `commander` (CLI), `ajv` (optional JSON Schema), `node-fetch` or `undici`
* Cloud (choose): `@aws-sdk/client-s3` or `@google-cloud/storage`
* Signing: `jose`
* Dev: `typescript`, `tsx`, `eslint`, `vitest`

---

## 7) Validate, Build, Publish

### Install

```bash
pnpm i
```

### Validate locally

```bash
pnpm cfg:validate   # validates YAMLs against Zod schema
```

### Diff two revisions

```bash
pnpm cfg:diff --from v0.1.0 --to v0.1.1
```

### Publish

```bash
pnpm cfg:publish --env dev   # builds base + dev overlay → uploads JSON bundles
```

Outputs (example):

```
artifacts/
  dev/
    current.json            # full config (signed)
    public.json             # safe subset for WebApp
    sha256.txt              # provenance hash
```

---

## 8) Consumers (how services use it)

* Services set `CONFIG_BASE_URL=${CDN_BASE_URL}/<env>/current.json`
* WebApp uses `public.json` to gate features client‑side
* Hot‑reload interval default: **60 seconds** (configurable per service)
* Consumers verify **signature** when `SIGNING_KEY_ID` is enabled

---

## 9) Testing

* **Schema tests**: ensure example configs pass/fail as expected
* **Merge tests**: overlay merges produce deterministic bundles
* **Signature tests**: verify with public JWK

Run:

```bash
pnpm test
```

---

## 10) CI/CD

**GitHub Actions** (suggested):

* On PR: run `cfg:validate` + `cfg:diff`
* On tag: `cfg:publish --env stage|prod` → upload to bucket + purge CDN
* Record artifact SHA & signature in release notes

---

## 11) Operational Notes

* Keep changes **reviewed by 2 people** for prod publishes
* Emergency rollback: re‑point `current.json` to previous version and purge CDN
* Version your schema; consumers should tolerate unknown fields (forward‑compatible)

---

## 12) Roadmap

* Signed **JWKS** endpoint for config verification
* UI editor (via Admin) with schema‑aware forms and linting
* Multi‑region buckets +
