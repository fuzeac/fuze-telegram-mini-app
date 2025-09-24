# FUZE Telegram Mini App — Platform Overview

> These architectural blueprints are the public, “short version” that explains how the **FUZE Telegram Mini App** works at a technical level — without making you read a stack of internal docs. It’s the master map across all repos, with enough depth for builders, partners, and curious users.

_Last updated: 2025-09-24 11:02 +07_

---

## The App Story (Why we’re building this)
Crypto is finally useful when it meets people where they already are. Telegram is that place: dense communities, instant network effects, and global reach. The FUZE Mini App turns Telegram into a **utility layer** — play, discover, join events, earn, and trade within safe boundaries. We obsess over **speed, fairness, and trust**: integer math, signed configs, auditable flows, no hand‑wavy balances.

---

## Business Opportunity (What this unlocks)
- **Casual → committed funnel:** lightweight Telegram entry points that graduate users into higher‑trust actions (games, events, contributions, purchases).  
- **City‑level networks:** localization atoms (areas / buckets) make communities tangible and sponsorable.  
- **Project enablement:** a turn‑key way for token teams to reach users with fair mechanics, campaigns, and events.  
- **Partner surface:** external games and tools can plug into PlayHub and ride our identity, wallets, and price rails.

---

## Participants (Target Users)
- **Players & bettors** — quick matches, fair price bets (CFB v1), verifiable results.  
- **Explorers** — watchlists, token pages, curated feeds and predictions.  
- **Organizers** — events and community activations, linked to platforms like Lu.ma, Eventbrite, Meetup, Eventpop.  
- **Projects & teams** — funding windows, airdrops/campaigns, listings, partner games.  
- **Operators** — finance/support/compliance via the Admin console.  

---

## Benefits to Participants
- **Speed & simplicity:** open in Telegram, act in seconds.  
- **Fairness by design:** server‑to‑server results, signed price snapshots, audit logs.  
- **Safety rails:** holds/settlements via Payhub; no silent balance edits.  
- **Local community:** area buckets to meet IRL; content localized (see next section).

---

## FUZE Utility & Ecosystem Impact
- **Utility tokens for action:** STAR / FZ / PT in MVP for gameplay and CFB v1; future bridges to on‑chain custody.  
- **Shared rails:** identity, pricing, and ledger primitives enable many apps to coexist.  
- **Economic flywheels:** fees/rakes reinvested into campaigns, grants, and events to grow local communities.

---

## Growth Levers & Loops (Opportunities)
- **Localization (core highlight):**
  - **Geo/Area buckets** → city‑level rooms, leaderboards, and events.  
  - **i18n** → `en`, `zh`, `hi`, `es`, `ar`, `ja`, `ko`, `de`, `pt`, `id`, `th`.  
- **Creator & organizer loops:** reward quality submissions; surface top organizers per area.  
- **Project loops:** listings drive watchlists → campaigns → events → funding.  
- **Gameplay loops:** PlayHub tournaments and seasonal ladders; fair bet (CFB v1) hubs by city.  
- **Referral loops:** clear, anti‑sybil rewards with vesting and limits.

---

## How the App Helps Crypto Projects
- **Launch & grow:** simple presales, allowlists, and vesting via Funding.  
- **Activate & retain:** events, quests, and watchlist presence; CFB v1 to energize communities around price milestones.  
- **Operate safely:** Admin approvals, audit trails, and predictable rails for finance tasks.  
- **Integrate games:** partner backends plug into PlayHub with a small, well‑documented contract.

---

## Platform Monetization (How we earn)
- **CFB v1 rake:** default 7% taken from winners (configurable per asset).  
- **Game fees:** per‑match or seasonal ladders via PlayHub partners.  
- **Listing & campaign packages:** boosted discovery, sponsored events, curated placements.  
- **Conversion & payout fees:** when bridges and providers are enabled.  
- **B2B tooling:** dashboards, data access, and premium moderation for projects.

---

## KPIs & Guardrails (first pass)
**User KPIs**
- WAU / MAU, activation rate (first action within 24h), retention cohorts.  
- Matchmaking time to match, CFB bet participation rate, event RSVPs.  

**Quality KPIs**
- Push rate in CFB (oracle issues) < 1.5%.  
- Dispute rate per 1k matches < 2; resolved < 24h.  
- Fraud flags per 1k actions with decreasing trend.

**Financial KPIs**
- Locked vs available balances (healthy distribution).  
- Settlement latency p95 < 2s; ledger reconciliation zero‑drift daily.  

**Guardrails**
- Integer money math; idempotent POSTs; service JWTs only.  
- Two‑person approvals for withdrawals, manual credits, forced settles.  
- Signed configs with anti‑rollback; JWKS rotation; least‑privilege IAM.

---

## Compliance & Trust
- **Privacy‑lite:** store user IDs; minimal PII; GDPR‑aligned deletion hooks.  
- **KYC/AML ready:** pluggable providers for payouts and conversions.  
- **Fairness guarantees:** price data from Price Service with signed reports; game results via server‑to‑server only.  
- **Transparency:** per‑action audit logs; reproducible configs in a public format.

---

## Roadmap Highlights
- **MVP (shipping):** Identity, WebApp, PlayHub (matchmaking + CFB v1 off‑chain), Payhub (ledger/holds/settlements), Watchlist, Funding, Campaigns, Events, Price, Workers, Admin, Config, Infra.  
- **Near term:** results and feed streaming; richer city hubs; organizer tools; low‑friction top‑ups.  
- **Next wave:** on‑chain custody adapters; fiat ramps; reputation & badges; more partner games; verified project portals.

---

## Architecture at a Glance
- **Identity** issues sessions from Telegram initData and service JWTs.  
- **WebApp** is the Telegram UI; never touches Payhub directly.  
- **PlayHub** pairs players, creates rooms, verifies game results, and runs **CFB v1**; calls **Payhub** for holds/settlements.  
- **Payhub** is the source of truth for balances, holds, and ledger entries.  
- **Price Service** provides signed price snapshots and TWAPs.  
- **Watchlist** curates assets and feeds. **Funding** handles sales & vesting.  
- **Campaigns**/ **Events** run growth and IRL activation.  
- **Workers** schedule settlements, releases, retries, and ingestion.  
- **Admin** enforces RBAC, approvals, audits.  
- **Config** ships signed, versioned configs; **Infra** runs everything.

---

## Repo Map (Architectural Blueprint lives in each repo)
- [miniapp-identity-service](miniapp-identity-service.md) — Sessions & service tokens.  
- [miniapp-webapp](miniapp-webapp.md) — Telegram frontend.  
- [miniapp-playhub-service](miniapp-playhub-service.md) — Matchmaking, rooms, **CFB v1**.  
- [miniapp-payhub-service](miniapp-payhub-service.md) — Accounts, holds, settlements, ledger.  
- [miniapp-price-service](miniapp-price-service.md) — Signed price snapshots, TWAP.  
- [miniapp-watchlist-service](miniapp-watchlist-service.md) (formerly discovery) — Assets, feeds, predictions.  
- [miniapp-funding-service](miniapp-funding-service.md) — Sales, allocations, vesting.  
- [miniapp-escrow-service](miniapp-escrow-service.md) — OTC/P2P contracts.  
- [miniapp-campaigns-service](miniapp-campaigns-service.md) — Quests, airdrops, rewards.  
- [miniapp-events-service](miniapp-events-service.md) — Event catalog & submissions.  
- [miniapp-admin](miniapp-admin.md) — Operator console & BFF.  
- [miniapp-workers](miniapp-workers.md) — Schedulers & background executors.  
- [miniapp-config](miniapp-config.md) — Signed configs & publisher.  
- [miniapp-infra](miniapp-infra.md) — Terraform/Helm/GitOps, observability.  
- [miniapp-game-service-template](miniapp-game-service-template.md) — Partner game blueprint.  
- [miniapp-shared](miniapp-shared.md) — DTOs, auth, HTTP client, telemetry, config SDK.

> Each repo’s **SystemDesign.md** is intentionally concise and compatible. Together, they form the complete blueprint for a working app.

---

## Closing
**The FUZE Mini App** is our bet on useful, everyday crypto where people already are: Telegram. It’s simple to open, fast to use, and designed to grow a healthier FUZE economy — one that rewards participation, not just speculation.

**More updates soon.** If you’d like a private walkthrough or to list your project for the first wave, ping the team.
