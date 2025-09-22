# tg-miniapp-admin

Admin console (Next.js) for operating the Telegram Mini App platform. Provides feature flags & config editing, visibility into ledgers and settlements (read‑only), dispute tools, campaign and event management, user moderation, and observability dashboards.

---

## 1) App Story

**tg-miniapp-admin** is the staff‑only UI for running day‑to‑day operations across all services (Identity, Payhub, PlayHub/CFB, Campaigns, Discovery, Events, Funding, Escrow). It does not store business data itself; it calls each service’s admin/API endpoints and renders safe, audit‑logged actions.

---

## 2) Duties & Scope

### Owns

* Admin UI workflows, navigation, and permissions gating (RBAC)
* Feature flags & configuration publishing (via `tg-miniapp-config`)
* Read‑only financial visibility (wallets, ledger, holds, settlements)
* Dispute tooling (Escrow/CFB), manual retries (DLQ replays through Workers)
* Campaign/event CRUD and approvals
* User moderation (ban/unban, role assignment) via Identity

### Does **not** own

* Ledger mutations, custody, or settlement rules → **Payhub**
* Game/CFB logic → **PlayHub**
* Persistent config store (only edits/publishes) → **tg-miniapp-config**

---

## 3) Architecture

* **Next.js 14** (App Router) + **React 18**
* **TypeScript**, **Zod** for DTOs, **React Query** for data fetching
* **Tailwind CSS** for styling, **shadcn/ui** for primitives
* **OIDC** staff SSO (Clerk/Auth0/Keycloak), roles/entitlements pulled from Identity
* **Ky** or **Axios** for HTTP client; server actions for privileged calls

---

## 4) ENV & Runtime Config

Create `.env.local`:

```dotenv
NEXT_PUBLIC_APP_NAME=tg-miniapp-admin

# Staff SSO (OIDC)
OIDC_ISSUER_URL=https://your-idp.example.com/oidc
OIDC_CLIENT_ID=replace_me
OIDC_CLIENT_SECRET=replace_me
OIDC_REDIRECT_URI=https://admin.tg-miniapp.example.com/api/auth/callback
SESSION_SECRET=replace_me

# Service endpoints
IDENTITY_API_BASE=https://identity.tg-miniapp.example.com
PAYHUB_API_BASE=https://payhub.tg-miniapp.example.com
PLAYHUB_API_BASE=https://playhub.tg-miniapp.example.com
CAMPAIGNS_API_BASE=https://campaigns.tg-miniapp.example.com
DISCOVERY_API_BASE=https://discovery.tg-miniapp.example.com
EVENTS_API_BASE=https://events.tg-miniapp.example.com
ESCROW_API_BASE=https://escrow.tg-miniapp.example.com
FUNDING_API_BASE=https://funding.tg-miniapp.example.com
CONFIG_PUBLISH_URL=https://config.tg-miniapp.example.com/publish
CONFIG_READ_URL=https://config.tg-miniapp.example.com/current

# Observability
GRAFANA_PUBLIC_URL=https://grafana.example.com/d/miniapp/overview
```

Secrets guidance

* Keep `SESSION_SECRET`, OIDC client secret in a secret manager; do not commit.
* Restrict admin origin/hostnames via CSP and reverse proxy ACLs.

---

## 5) Dependencies

* Runtime: `next`, `react`, `react-dom`, `zod`, `@tanstack/react-query`, `ky`, `next-auth` or OIDC library
* UI: `tailwindcss`, `@/components/ui/*` (shadcn/ui), `lucide-react`
* Dev/Test: `typescript`, `eslint`, `playwright`, `vitest`, `msw`

---

## 6) Key Screens & Actions (MVP)

* **Dashboard**: service health, key SLOs, open DLQs, recent incidents
* **Users**: search, profile, roles/entitlements editor (Identity)
* **Wallets**: balances & ledger (read‑only), user lookup (Payhub)
* **Games/CFB**: open rooms/bets, settlement status, force‑retry buttons (calls Workers)
* **Campaigns**: list, create/update, approve, view claims
* **Events**: list, create/update, publish/unpublish
* **Config**: edit YAML (with schema validation), publish → rollout status
* **Audits**: chronicle of admin actions with actor, time, payload hash

---

## 7) Development

```bash
pnpm i
pnpm dev     # http://localhost:3001 (recommended distinct port from WebApp)
```

Login flow (dev)

* Configure your IdP redirect URL to the local dev URL.
* Seed a staff user in Identity with proper roles (`admin.*`).

Lint/typecheck

```bash
pnpm lint
pnpm typecheck
```

---

## 8) Build & Deploy

### Docker

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

### Helm (excerpt)

```yaml
image: { repository: ghcr.io/yourorg/tg-miniapp-admin, tag: "v0.1.0" }
env:
  OIDC_ISSUER_URL: https://your-idp.example.com/oidc
  OIDC_CLIENT_ID: your-client
  CONFIG_READ_URL: https://config.tg-miniapp.example.com/current
  # ...
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
probePaths:
  liveness: /api/healthz
  readiness: /api/readyz
```

GitHub Actions

* Build on tag → push image → deploy via ArgoCD/GitOps

---

## 9) Testing

### E2E (Playwright)

* Staff SSO login → sessions established
* Config edit → publish → verify in consumer service
* DLQ replay action triggers worker API and shows success toast
* CFB settlement inspection page renders payout math

### Unit/UI

* Form validation with Zod, RBAC guards, table filters

Run:

```bash
pnpm test
pnpm test:e2e
```

---

## 10) Security & Compliance

* **RBAC** enforced per route and component; hide and block
* **CSRF** on admin mutations; **CSP** allowlist
* **Audit log** every privileged action with before/after diff hash
* **2‑person rule** option for config publishes (Phase‑2)

---

## 11) Roadmap

* Incidents module (create/update, link to Grafana panels)
* Built‑in ArgoCD view of rollout health
* One‑click canary/rollback for config and deployments
