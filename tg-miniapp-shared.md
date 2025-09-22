# tg-miniapp-shared

Shared TypeScript library for DTOs, validation schemas, auth utilities, idempotency helpers, tracing middleware, and error envelopes used across the Telegram Mini App services and apps.

> Publish as a private npm package or use via monorepo workspace. Keeps cross-service contracts consistent.

---

## 1) Package Story

**tg-miniapp-shared** centralizes common types and helpers so each service shares the same request/response contracts and middleware. It includes Zod schemas for all public/internal APIs, JWT helpers (service JWT & GST), idempotency utilities, and standardized error handling with trace IDs.

---

## 2) Duties & Scope

### Provides

* **DTOs & Zod Schemas**: Identity, Payhub, PlayHub/CFB, Campaigns, Escrow, Events
* **Auth utils**: Ed25519 JWT sign/verify (service JWT, GST), Telegram `initData` HMAC verifier
* **HTTP middleware**: request logging, `X-Request-Id`, error envelope mapper
* **Idempotency helpers**: request hashing, Redis adapter interface
* **Tracing**: OpenTelemetry setup helpers and span decorators
* **Result types**: `Result<T, E>` helpers, error codes enum

### Out of scope

* Business logic of any single service
* Data access layers (each service implements its own persistence)

---

## 3) Structure

```
src/
  auth/
    jwt.ts           # Ed25519 sign/verify (service JWT + GST)
    telegram.ts      # WebApp initData verifier
  dto/
    identity.ts
    payhub.ts
    playhub.ts
    cfb.ts
    campaigns.ts
    escrow.ts
    events.ts
  http/
    middleware.ts    # pino logger, request-id, error handler
    errors.ts        # error envelope with codes
  idempotency/
    hash.ts
    store.ts         # interface; redis + memory impls in adapters/
  tracing/
    otel.ts          # minimal OTel setup
adapters/
  redis-idem.ts
  express-middleware.ts
index.ts
```

---

## 4) ENV (only for tests/examples)

```dotenv
LOG_LEVEL=info
# Keys below are used in tests only; production services inject their own
SESSION_JWT_PRIVATE_JWK={"kty":"OKP","crv":"Ed25519","d":"REPLACE","x":"REPLACE"}
SERVICE_JWT_PRIVATE_JWK={"kty":"OKP","crv":"Ed25519","d":"REPLACE","x":"REPLACE"}
```

---

## 5) Dependencies

* Runtime: `zod`, `jose`, `pino`, `uuid`, `@opentelemetry/api`
* Optional peers: `express` (for middleware types), `ioredis` (for redis adapter)
* Dev: `typescript`, `tsx`, `vitest`, `eslint`, `tsup` or `tsup-node` for bundling

---

## 6) Build & Use

### Monorepo (workspace) usage

Add to service `package.json`:

```json
{
  "dependencies": {
    "tg-miniapp-shared": "workspace:*"
  }
}
```

Import in code:

```ts
import { ServiceJwt, ErrorEnvelope, Idempotency } from "tg-miniapp-shared";
```

### Publish (private npm)

1. Set `NPM_TOKEN` in CI
2. Build & publish:

```bash
pnpm build
pnpm publish --access restricted
```

---

## 7) Scripts

```json
{
  "scripts": {
    "build": "tsup src/index.ts --dts --format esm,cjs",
    "lint": "eslint .",
    "test": "vitest run",
    "dev": "vitest"
  }
}
```

---

## 8) Testing

* **Auth**: JWT sign/verify round‑trip; Telegram `initData` HMAC fixtures
* **Schemas**: happy/invalid cases for each DTO
* **Idempotency**: hash stability; redis adapter semantics

Run:

```bash
pnpm test
```

---

## 9) Versioning & Contracts

* Semantic versioning (`major.minor.patch`). Bump **major** on breaking schema changes.
* Each schema/DOT exports a `version` field; include it in responses for debuggability.
* Keep a `CHANGELOG.md` with migration notes when fields change.

---

## 10) Roadmap

* OpenAPI generator from Zod schemas
* Client SDK generation for WebApp/Admin
* gRPC/protobuf schema parity (if adopted later)
