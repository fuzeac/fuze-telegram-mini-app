# Fuze Telegram Mini App

A Telegram-first, modular web application that brings **wallet + gaming + prediction markets + discovery + campaigns + events** into one lightweight experience. Built as a set of focused microservices with off‑chain custody today and clean adapters for on‑chain tomorrow.

> This README frames the *why* and the *what*: story, opportunity, target users, benefits, FUZE utility, monetization, growth, and how we help crypto projects.

---

## 1) App Story

* **Problem**: Crypto communities live on Telegram, but most actions (earn, play, discover, support projects) are fragmented across bots, spreadsheets, and third‑party sites. Users drop off; projects can’t activate, retain, or measure.
* **Solution**: A **single Telegram Mini App** where users can:

  * Manage an **off‑chain wallet** (STAR/FZ/PT/FUZE/USDT) with frictionless deposits/withdrawals.
  * **Play** head‑to‑head games (via PlayHub) and join **CFB** (Crypto Fair Bet) markets using in‑app currencies.
  * **Discover** tokens/projects with watchlists, alerts, news, and signals.
  * Complete **campaigns/airdrops** with verifiable tasks; get paid instantly.
  * Browse **events** and jump to booking links.
* **Design principles**: instant boot (Telegram), fully‑collateralized flows, strict idempotency, auditability, and a clear path to on‑chain when/where it helps.

---

## 2) Business Opportunity

* **Where users are**: Telegram hosts thousands of crypto communities; Mini Apps remove install friction and boost trust via the parent bot.
* **Multi‑surface engagement**: games + prediction + quests + discovery + events = higher session count and time‑on‑platform.
* **Liquidity & intent**: CFB and Escrow create *priced* intent data (who risks what on which assets), valuable for projects and partners.
* **Distribution flywheel**: campaigns fund user rewards which cycle back into games/CFB, increasing balances and activity, lowering CAC.
* **Expandable rails**: off‑chain first makes compliance and iteration fast; future on‑chain adapters unlock new partner ecosystems.

---

## 3) Participants (Target Users)

* **Everyday Crypto Users**: want fun, light games, simple prediction markets, quick rewards, and curated info without leaving Telegram.
* **Active Traders/Power Users**: track assets, set alerts, participate in fair bets, and use OTC escrow for P2P deals.
* **Project Teams & Creators**: launch campaigns/airdrops, list events, share updates, and observe engagement metrics.
* **Event Organizers / Communities**: publish meetups or AMAs, drive to booking sites, and optionally gate perks via campaigns.
* **Partners/Exchanges/Wallets** (Phase‑2): co‑market listings, provide liquidity or sponsor campaigns.

---

## 4) Benefits to Participants/Users

* **One place, zero friction**: open in Telegram; auth via WebApp HMAC.
* **Instant actions**: holds/settlements are fast and final; no manual proofs.
* **Transparent economics**: CFB shows exact rake (7%), fully‑collateralized payouts, and auditable oracle reports.
* **Earn & engage**: campaigns reward tasks; games/CFB create skill‑and‑conviction based upside.
* **Signal‑rich discovery**: watchlists, alerts, curated news, and simple momentum/mention signals.
* **Safety**: Escrow for P2P with dual holds and dispute tools (off‑chain v1).

---

## 5) FUZE Utility & Ecosystem Impact

> FUZE is the ecosystem token; the app increases its **velocity, demand, and stickiness** via these mechanisms.

* **Unit of account (internal conversions)**: FUZE → FZ/PT (or future credit units) with configurable fees, creating **conversion demand**.
* **Fee token & discounts**: pay platform rake/fees in FUZE for discounts or tiered benefits (Phase‑2), driving **explicit FUZE utility**.
* **Staking for access** (Phase‑2): stake FUZE to unlock higher limits (CFB sizes, campaign budgets) or to list projects/events.
* **Treasury sinks**: portions of rakes/fees denominated in FZ/PT can be periodically **market-bought into FUZE** to bolster treasury or burn.
* **Ecosystem rails**: campaigns airdrop FUZE or FUZE‑denominated rewards; discovery surfaces FUZE‑centric promotions.

---

## 6) Platform Monetization (How we earn)

* **CFB Rake**: 7% of the **losing side** (configurable per config repo). Transparent and programmatic.
* **Game Rakes/Fees**: small per‑match rake or room fee (game‑specific).
* **Conversion Spreads**: configurable basis points for STAR↔FZ↔PT↔FUZE (via Payhub conversions).
* **Escrow Fees**: fixed or bps fee on completed P2P trades; optional dispute fee.
* **Sponsored Campaigns**: projects pay for budgets, featured placements, or cost‑per‑completed‑task.
* **Data/Insights (opt‑in)**: aggregate, anonymized performance insights for projects (activity funnels, campaign efficacy).

---

## 7) Growth Levers & Loops (Opportunity for Growth)

* **K‑factor via campaigns**: referral tasks and squad quests create viral loops with measurable ROI.
* **Creator/Game ecosystem**: third‑party game servers integrate through a simple template; each new game is a new acquisition surface.
* **CFB breadth**: add more assets/timeframes; leaderboards and seasons increase retention.
* **Events to campaigns**: event attendance → quest claims → token payouts → re‑engagement in games/CFB.
* **Trust & proof**: public payout stats, provable oracle reports, and fast withdrawals build reputation → lower CAC.
* **Localization**: geo/area buckets enable city‑level communities and events; content localization (EN/TH first).

---

## 8) How the App Helps Crypto Projects

* **Turn community into measurable actions**: quests drive follows, joins, AMAs, content shares, and event attendance with anti‑abuse checks.
* **Fast distribution**: instant, idempotent payouts in FZ/PT/STAR; optional FUZE rewards.
* **Insightful reporting**: cohort performance, quest completion drop‑offs, and event‑linked conversions.
* **Flexible promotion surfaces**: featured discovery cards, sponsored CFB slates, campaign pinning.
* **Low integration cost**: no smart contract required for v1; on‑chain adapters later for trustless flows.

---

## 9) System Overview (brief)

* **Identity Service** — Telegram auth, users, roles, geo buckets.
* **Payhub** — ledger, balances, holds/settlements, conversions.
* **PlayHub** — games + **CFB** (owner vs pool, 7% rake, oracle TWAP).
* **Discovery** — watchlists, alerts, news, signals.
* **Campaigns** — quests/airdrops with proof checks and payouts.
* **Escrow** — P2P OTC with dual holds and disputes.
* **Events** — directory and external booking links.
* **Workers** — async jobs, reconcilers, DLQs.
* **Config & Infra** — versioned flags/limits, IaC, CI/CD, observability.

> See each service’s README for endpoints, ENV, deploy, and tests.

---

## 10) KPIs & Guardrails (first pass)

* **Activation**: % of Telegram sessions that mint a session token and visit 2+ surfaces.
* **Liquidity**: total holds/settlements per day; CFB S\_owner/S\_acc ratios; escrow volume.
* **Monetization**: rake revenue (CFB/games), spreads (conversions), campaign spend.
* **Trust**: settlement success rate, orphan-hold age, withdrawal SLA, dispute resolution time.
* **Safety**: abuse flags per 1k actions; failed proof rates; KYC pass‑through (when enabled).

---

## 11) Roadmap Highlights

* **Phase‑1 (MVP)**: off‑chain custody, CFB v1 with STAR/FZ/PT, 1–2 games, discovery basics, campaigns simple tasks, events directory, escrow same‑currency, price service optional.
* **Phase‑2**: FUZE fee discounts & staking tiers, more games, richer discovery/providers, on‑chain adapters (custody/oracle), advanced escrow templates, creator SDK.

---

## 12) Repo Map

* [tg-miniapp-identity-service](tg-miniapp-identity-service.md) — users/auth/roles/geo
* [tg-miniapp-payhub-service](tg-miniapp-payhub-service.md) — ledger/holds/settlements/conversions
* [tg-miniapp-playhub-service](tg-miniapp-playhub-service.md) — games + **CFB**
* [tg-miniapp-discovery-service](tg-miniapp-discovery-service.md) — watchlists/alerts/news/signals
* [tg-miniapp-campaigns-service](tg-miniapp-campaigns-service.md) — quests/airdrops
* [tg-miniapp-escrow-service](tg-miniapp-escrow-service.md) — OTC P2P
* [tg-miniapp-events-service](tg-miniapp-events-service.md) — event directory
* [tg-miniapp-webapp](tg-miniapp-webapp.md) — Telegram UI
* [tg-miniapp-admin](tg-miniapp-admin.md) — admin UI
* [tg-miniapp-workers](tg-miniapp-workers.md) — async jobs
* [tg-miniapp-config](tg-miniapp-config.md) — versioned flags/limits
* [tg-miniapp-infra](tg-miniapp-infra.md) — IaC/CI‑CD/observability
* [tg-miniapp-price-service](tg-miniapp-price-service.md) — (optional) price/TWAP service
* [tg-miniapp-game-service-template](tg-miniapp-game-service-template.md) — third‑party game boilerplate
* [tg-miniapp-shared](tg-miniapp-shared.md) — shared DTOs/auth/idempotency/tracing

---

## 13) Compliance & Trust (quick note)

This app involves **betting** and **payouts**. Always review local regulations (licensing, KYC/AML, age gating). The architecture supports feature flagging per region, responsible gaming controls (limits, cooldowns, self‑exclusion), audit logs, and data export/deletion.

---

## 14) Getting Started (dev quickstart)

1. Boot **Identity** and **Payhub**, then **PlayHub**.
2. Run **WebApp** with your Telegram bot’s WebApp URL (ngrok for local).
3. Optional: run **Workers** and **Price Service** for CFB.
4. Use **Config** repo to enable/disable features and set limits.

Happy building! 🚀
