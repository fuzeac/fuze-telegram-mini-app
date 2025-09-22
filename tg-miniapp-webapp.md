# tg-miniapp-webapp

Telegram WebApp frontend (Next.js) for wallet, games/CFB, discovery, events, campaigns. Authenticates users via Telegram WebApp `initData`, talks to backend services with a short‑lived session token from **Identity Service**.

---

## 1) App Story

The **WebApp** is the primary user interface inside Telegram. It boots with Telegram’s WebApp SDK, validates the `initData` with the **Identity Service**, and renders feature surfaces (Wallet, Games/CFB, Discovery, Campaigns, Events) based on config flags. UX prioritizes low‑latency navigation, offline‑friendly data views, and clear error envelopes.

---

## 2) Duties & Scope

### Owns

* UI/UX, routing, i18n (EN/TH), session handling, API clients
* Presentation of balances, bets, games, discovery feeds, events
* Local feature gating from `tg-miniapp-config` (public subset)

### Does **not** own

* Business logic (ledger, holds/settlements, bets) → server services
* Admin/staff tools → `tg-miniapp-admin`

---

## 3) Architecture

* **Next.js 14** (App Router) + **React 18**
* **Telegram WebApp SDK** for init & UI theme sync
* **TypeScript**, **Zod** for API DTOs, **React Query** for data fetching/cache
* **Tailwind CSS** for styling, **i18next** for translations
* **SWR/React Query** stale‑while‑revalidate patterns

**Key Flows**

1. **Boot**: Read `window.Telegram.WebApp.initData`; send to `/v1/auth/telegram` (Identity) → receive session token.
2. **Config**: fetch public config subset for gating (PlayHub/CFB enabled, currency display, etc.).
3. **Features**:

   * **Wallet**: show balances (Payhub), ledger list; link to conversions.
   * **Games / Matchmaking**: call PlayHub endpoints; handle GST redirect.
   * **CFB v1**: create/accept/open bets, results view.
   * **Discovery/Events/Campaigns**: list feeds, detail screens, external links.

---

## 4) ENV & Runtime Config

Create `.env.local` for local dev; `.env` for container builds.

```dotenv
# Public URLs (exposed to client)
NEXT_PUBLIC_API_BASE=https://api.tg-miniapp.example.com
NEXT_PUBLIC_TELEGRAM_BOT_NAME=YourBotName
NEXT_PUBLIC_FEATURES=wallet,games,cfb,discovery,campaigns,events

# Internal server runtime (edge/server actions)
IDENTITY_API_BASE=https://identity.tg-miniapp.example.com
CONFIG_PUBLIC_URL=https://config.tg-miniapp.example.com/public.json
```

**Notes**

* Keep server‑only keys out of `NEXT_PUBLIC_*` vars.
* Use `CONFIG_PUBLIC_URL` to fetch non‑sensitive flags and numeric limits for rendering.

---

## 5) Dependencies

* Runtime: `next`, `react`, `react-dom`, `zod`, `@tanstack/react-query`, `i18next`, `react-i18next`, `clsx`, `ky` (HTTP client)
* UI: `tailwindcss`, `@headlessui/react`, `lucide-react`
* Dev/Test: `typescript`, `eslint`, `playwright`, `vitest` (optional), `msw` (mock APIs)

---

## 6) Development

### Prereqs

* Node.js ≥ 20, pnpm ≥ 9
* Telegram bot set up with WebApp support (App menu → Test)
* Identity service reachable from your browser (CORS enabled during dev)

### Run locally

```bash
pnpm i
pnpm dev     # http://localhost:3000
```

**Local Telegram test**

* In BotFather, set WebApp URL to `https://<your-ngrok>/` during development.
* Use `ngrok http 3000` and update the link.

### Lint/format

```bash
pnpm lint
pnpm typecheck
```

---

## 7) Build & Deploy

### Vercel (simple)

* Add env vars in Vercel Project Settings (copy from `.env.local`).
* `vercel --prod` or via Git integration.

### Container / Helm

* Dockerfile example:

```Dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm i --frozen-lockfile

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./
RUN corepack enable && pnpm i --prod --frozen-lockfile
EXPOSE 3000
CMD ["pnpm","start"]
```

* Helm values (excerpt):

```yaml
image: { repository: ghcr.io/yourorg/tg-miniapp-webapp, tag: "v0.1.0" }
env:
  NEXT_PUBLIC_API_BASE: https://api.tg-miniapp.example.com
  NEXT_PUBLIC_TELEGRAM_BOT_NAME: YourBotName
  CONFIG_PUBLIC_URL: https://config.tg-miniapp.example.com/public.json
```

---

## 8) Testing

### Unit/UI

* Use Vitest + React Testing Library for components and hooks.

### E2E (Playwright)

Scenarios:

* Telegram `initData` → Identity → session cookie set
* Wallet balances list renders
* Matchmaking join flow → GST redirect URL present
* CFB create/accept → result appears after settlement (mock price API)

Run:

```bash
pnpm test
pnpm test:e2e
```

---

## 9) Security & Hardening

* Enforce **CSP** with allowed Telegram origins and your API domains.
* Validate `initData` only server‑side; never trust client values alone.
* Store session in **httpOnly** cookie; short TTL; renew on activity.
* Guard SSR routes from being embedded outside Telegram unless explicitly allowed.

---

## 10) Roadmap

* Offline cache for wallet/ledger
* Deep links for CFB bet detail & results
* In‑WebApp notifications for settlements and campaign claims
