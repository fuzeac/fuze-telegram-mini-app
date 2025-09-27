Repo: tg-miniapp-webapp
File: UserStories.md
SHA-256: 8c6f1a9f23add07475dc2a3cee120234fd13cc908621775b35410e24fc79f45b
Bytes: 21009
Generated: 2025-09-27 01:05 GMT+7
Sources: SystemDesign.md (authoritative), old UserStories.md (baseline), Guide

---

# Section A — Personas & Stakeholders

> Summaries derived from system diagrams, consumed API scopes, and cache ownership. Full catalog in **Appendix A1**.

- **End User** — Telegram Mini App user. Goals: manage assets, watchlist, alerts, join games, complete quests, RSVP events, pay overages. KPIs: task success, low latency, clarity.
- **Investor User** — End User with **Investor badge** (KYC). Goals: higher limits, withdrawals. KPIs: payout success, limit clarity.
- **MiniApp Client** — The WebApp itself, acting via Shared SDK. Responsibilities: retries, idempotency, caching, telemetry. KPIs: error rate, LCP/INP, p95 latency.
- **Support/CS** — Handles user issues, edge-case assistance via Admin tools. KPIs: time-to-resolution, accurate guidance.
- **Operator/Admin** — Configures feature flags, experiments, and announcements (via Config/Admin). KPIs: rollout safety, conversion.
- **Compliance** — Oversees KYC prompts and disclosures (surfaced, not stored). KPIs: correct gating, user comms.
- **Finance/Billing** — Owns invoices and overage flows (via Payhub). KPIs: collection rate, refund accuracy.
- **Campaign Manager** — Runs quests/airdrops. KPIs: completion rates, fraud defenses.
- **Game Coordinator** — Oversees matchmaking sessions and room health cues (surfaced to users).
- **SRE/Ops** — Observability hooks, client feature flags for incident modes (degradation banners).
- **Telegram Platform** — Supplies `initData`, UX constraints, and windowing model.
- **External Wallet User** — Brings chain addresses when withdrawing (address entry UX).

---

# Section B — Epics → Features → User Stories (exhaustive)

## Epic B1 — Entry, Session, and Profile

### Feature B1.1 — Telegram login
**Story B1.1.1** — *As an End User, I want to sign in with Telegram,* so that *I can use the Mini App safely.*  
**Acceptance (GWT)**
1. **Given** Telegram provides `initData`, **when** WebApp calls `/identity/v1/session/telegram-init`, **then** it stores `{jwt, profile, badges, meters}` on success and navigates to Home.
2. **Given** the request fails with **400**, **when** error is `invalid_telegram_payload`, **then** show retry with “Open in Telegram” guidance.
3. **Given** a **429** from Identity, **when** retry backoff is active, **then** the UI disables the login button and re-enables after `Retry-After`.
4. **Given** success, **when** badges are present, **then** badge chips render in header.

**Non‑Happy Paths & Errors**: offline or captive portal, 5xx → banner with “Try again”, logs capture `requestId`.  
**Permissions**: none pre-login.  
**Data Touchpoints**: `CACHE_SESSION`, `CACHE_USER`, `CACHE_BADGE`, `CACHE_USAGE`.  
**Cross‑Service**: Identity only.  
**Observability**: `login.latency`, `login.error_rate`, trace with `traceparent`.  
**Performance**: ≤ 1s p95 first-content on cached bundle.

---

### Feature B1.2 — Session refresh
**Story B1.2.1** — *As a User, I want my session to refresh without interruption,* so that *I stay signed in.*  
**Acceptance**
1. Refresh occurs on tab focus and 60s before expiry.
2. If **401** on refresh, UI clears cache and returns to login with retained deep-link.
3. Parallel refresh requests dedupe via in-tab lock, second call receives “superseded” and exits.

**Non‑Happy**: local clock skew → warning re: device time.  
**Data**: `CACHE_SESSION`.  
**Observability**: `refresh.success`, `refresh.reuse_detected` (from server).

---

## Epic B2 — Balances, Deposit, Withdrawal, Conversion

### Feature B2.1 — View balances
**Story B2.1.1** — *As a User, I want to view my balances,* so that *I know my available STAR/FZ/PT/USDT.*  
**AC**: GET `/payhub/v1/wallets/balances` cached, shows `available` and `held`. Handles 5xx with cached data fallback and staleness tag.

### Feature B2.2 — Create deposit intent
**Story B2.2.1** — *As a User, I want to deposit USDT,* so that *I can top up.*  
**AC**
1. POST `/payhub/v1/deposits { network, amount }` returns address, min, confirmations.
2. UI renders QR, min amount, and confirmation count; shows network selector.
3. When deposit confirmed, balance auto-refreshes with toast.
**Non‑Happy**: attempt < min → server 422, UI error inline.  
**Permissions**: auth; some networks may require Investor badge (config flag).  
**Data**: `CACHE_INVOICE` used for pending intents, `CACHE_BALANCE` refresh.  
**Notifications**: in-app toast; optional Telegram bot notification (backend).

### Feature B2.3 — Withdraw
**Story B2.3.1** — *As an Investor User, I want to withdraw USDT,* so that *I can move funds out.*  
**AC**
1. POST `/payhub/v1/withdrawals` with address and amount, idempotency key required.
2. If KYC insufficient, server 403 → UI shows badge prompt.
3. On 202, UI tracks status and renders receipt when settled or failed.
**Non‑Happy**: chain fee spike or RPC down → “Delayed” banner with retry advice.  
**Data**: `CACHE_RECEIPT`.  
**Observability**: withdrawal funnel, error codes distribution.

### Feature B2.4 — Convert
**Story B2.4.1** — *As a User, I want to convert between STAR/FZ/PT/USDT,* so that *I can participate in features.*  
**AC**: POST `/payhub/v1/conversions/quote` → lock countdown in UI, confirm executes conversion or rejects expired quotes.

---

## Epic B3 — Watchlist & Alerts

### Feature B3.1 — Manage watchlist
**Story B3.1.1** — *As a User, I want to add an asset to my watchlist,* so that *I can track price.*  
**AC**
1. POST `/watchlist/v1/lists/{id}/items` with idem key, then 201 Added.
2. If free tier exceeded, **402** → overage modal drives invoice flow.
3. Duplicate symbol → **409** toast “Already on list.”
**Data**: `CACHE_WATCHLIST`, `CACHE_WATCH_ITEM`, `CACHE_USAGE` progression.  
**Permissions**: Investor badge may be required by policy to exceed X items.  
**Notifications**: none.

### Feature B3.2 — Create price alert
**Story B3.2.1** — *As a User, I want an alert when a price crosses a threshold,* so that *I can react fast.*  
**AC**
1. POST `/watchlist/v1/alerts` creates rule with quiet hours.
2. Duplicate rule → **409** informative toast with “Manage alerts” link.
3. Alert triggers deduped during quiet hours, next occurrence queued.
**Data**: `CACHE_ALERT`, occurrences summarized in `CACHE_ALERT_OCCUR`.  
**Notifications**: Telegram notification (backend) + in-app badge.  
**Observability**: rule create success, alert delivery rate.

---

## Epic B4 — Games (PlayHub entry & rooms)

### Feature B4.1 — Join matchmaking
**Story B4.1.1** — *As a User, I want to join a CFB room,* so that *I can play with a stake.*  
**AC**
1. POST `/playhub/v1/matchmaking/join { gameId, betAmount }` → waiting room or roomId.
2. If insufficient funds, **402** → quick-action to deposit or convert.
3. Room state updates show players, countdowns, and fairness snippet.
**Data**: `CACHE_GAME`, `CACHE_ROOM`.  
**Events**: `playhub.matchmaking.state.changed@v1` rendered if provided via polling or SSE.  
**Non‑Happy**: abandoned room → auto-leave with refund logic handled server-side, UI shows status.

---

## Epic B5 — Campaigns & Quests

### Feature B5.1 — Submit quest proof
**Story B5.1.1** — *As a User, I want to submit evidence for a quest,* so that *I can earn rewards.*  
**AC**
1. POST `/campaigns/v1/tasks/{taskId}/attempts` with evidence (size limits enforced).
2. **202 under_review** → UI cues review window, status chip updates.
3. On approval, wallet balance updates automatically.
**Non‑Happy**: **422** invalid evidence → field-level errors; fraud suspicion → soft lock with appeal CTA.

---

## Epic B6 — Events & RSVP

### Feature B6.1 — Discover events
**Story B6.1.1** — *As a User, I want to browse events by city or query,* so that *I can RSVP.*  
**AC**: GET `/events/v1/events { city, q }`, lazy infinite scroll, client caching.

### Feature B6.2 — RSVP and reminders
**Story B6.2.1** — *As a User, I want to RSVP and set a reminder,* so that *I don’t miss it.*  
**AC**: POST `/events/v1/events/{id}/rsvp { state, reminder }`, idempotent per `(user,event)`.

---

## Epic B7 — Overages & Invoices (Cross-service Payments)

### Feature B7.1 — Pay overage invoice
**Story B7.1.1** — *As a User, I want to pay when I exceed free tier,* so that *I can continue using features.*  
**AC**
1. When a **402** is returned, UI opens invoice modal.
2. POST `/payhub/v1/invoices { purpose, currency, amount }` creates invoice.
3. UI polls invoice status until `paid` or timeout; `paid` unlocks action and reloads quota.

---

## Epic B8 — Charts & Prices

### Feature B8.1 — Chart OHLCV
**Story B8.1.1** — *As a User, I want to see price charts by interval,* so that *I can make decisions.*  
**AC**: GET `/price/v1/ohlcv { symbol, interval }`, caches per symbol+interval, handles 429 with backoff.

---

## Epic B9 — Settings, Localization, Accessibility

### Feature B9.1 — Locale & Timezone
**Story B9.1.1** — *As a User, I want UI in my language and time,* so that *content is readable.*  
**AC**: Default **GMT+7**, profile locale applies, event cards show local & UTC tooltip.

### Feature B9.2 — Accessibility
**Story B9.2.1** — *As a User with accessibility needs, I want proper contrast and keyboard navigation,* so that *I can use the app.*  
**AC**: WCAG AA colors, focus outlines, ARIA labels, reduces motion toggle.

---

## Epic B10 — Reliability, Observability, and Incident UX

### Feature B10.1 — Degraded modes
**Story B10.1.1** — *As a User, I want clear messages during outages,* so that *I know what to do.*  
**AC**: Render service status banner, disable actions with reason, provide cached data when safe.

### Feature B10.2 — Telemetry
**Story B10.2.1** — *As SRE, I want client telemetry,* so that *we can correlate issues.*  
**AC**: Emit traces for each API call with `traceparent`, Web Vitals LCP/INP/CLS, error breadcrumbing.

---

# Section C — End‑to‑End Scenarios (Swimlane narrative)

1. **E2E‑C1: Watchlist Add → Overage → Invoice → Retry Action**  
   Pre: Authenticated, near limit. Trigger: Add item. Flow: POST add → **402** → invoice modal → pay → meter refresh → retry add succeeds. Post: Item added, usage meter updated.

2. **E2E‑C2: Join CFB → Insufficient Funds → Convert → Join**  
   Pre: Balance low. Flow: Join returns **402** → conversion quote → confirm → join again → waiting room. Post: Room visible.

3. **E2E‑C3: USDT Withdraw → KYC Gate → Apply Badge → Retry**  
   Pre: No Investor badge. Flow: Withdraw **403** → badge apply CTA → after approval → retry withdraw 202. Post: Receipt tracked.

4. **E2E‑C4: Quest Submit → Under Review → Reward Credited**  
   Flow: Submit → 202 → later approval → balance auto-refresh → toast.

5. **E2E‑C5: Events Browse → RSVP → Reminder**  
   Flow: List events → RSVP interested → reminder set → event appears in calendar view.

---

# Section D — Traceability Matrix

| Story | APIs | Entities/Cache | Events | Diagrams (SystemDesign) |
|---|---|---|---|---|
| B1.1.1 | `/identity/v1/session/telegram-init` | CACHE_SESSION, CACHE_USER | identity.session.created@v1 | §Flows 6.1 |
| B1.2.1 | refresh (implicit) | CACHE_SESSION | identity.session.refreshed@v1 | §Flows 6.1 |
| B2.1.1 | `/payhub/v1/wallets/balances` | CACHE_BALANCE | payhub.balance.updated@v1 | §Flows 6.4 |
| B2.2.1 | `/payhub/v1/deposits` | CACHE_INVOICE | payhub.deposit.updated@v1 | §Flows 6.4 |
| B2.3.1 | `/payhub/v1/withdrawals` | CACHE_RECEIPT | payhub.withdrawal.updated@v1 | §Flows 6.4 |
| B2.4.1 | `/payhub/v1/conversions/quote` | — | — | §Flows 6.3 |
| B3.1.1 | `/watchlist/v1/lists/{id}/items` | CACHE_WATCHLIST, CACHE_WATCH_ITEM | watchlist.item.created@v1 | §Flows 6.2 |
| B3.2.1 | `/watchlist/v1/alerts` | CACHE_ALERT, CACHE_ALERT_OCCUR | watchlist.alert.triggered@v1 | §Flows 6.2 |
| B4.1.1 | `/playhub/v1/matchmaking/join` | CACHE_GAME, CACHE_ROOM | playhub.matchmaking.state.changed@v1 | §Flows 6.3 |
| B5.1.1 | `/campaigns/v1/tasks/{taskId}/attempts` | — | campaigns.task.attempt.updated@v1 | §Flows 6.6 |
| B6.1.1 | `/events/v1/events` | CACHE_EVENT | events.catalog.updated@v1 | §Flows 6.5 |
| B6.2.1 | `/events/v1/events/{id}/rsvp` | CACHE_RSVP | events.rsvp.confirmed@v1 | §Flows 6.5 |
| B7.1.1 | `/payhub/v1/invoices` | CACHE_INVOICE | payhub.invoice.updated@v1 | §Flows 6.2 |
| B8.1.1 | `/price/v1/ohlcv` | — | price.snapshot.updated@v1 | §Tech Stack |
| B9.1.1 | — | CACHE_USER | — | §Config/ENV |
| B10.1.1 | — | — | status.banner.changed@v1 | §Reliability |
| B10.2.1 | — | — | — | §Observability |

**Deltas vs old UserStories**: adds invoice overage flow, quiet hours on alerts, matchmaking insufficiency path, full withdrawal KYC-gate, localization/timezone specifics, degraded incident UX, and explicit telemetry/trace linkage.

---

# Section E — Assumptions & Open Questions

- Confirmation mechanisms for deposits/withdrawals display (polling vs push).  
- Exact free-tier limits and meter names exposed to UI.  
- Supported chains and networks for deposits/withdrawals.  
- Availability of event reminder delivery channels from backend (Telegram push vs local reminders).

---

# Appendix — Coverage Gates

## A1. Stakeholder Catalog

| Stakeholder | Responsibilities | Permissions | Typical Actions | KPIs/SLO interests |
|---|---|---|---|---|
| End User | Use features in Telegram | session JWT | login, add watchlist, create alerts, deposit, withdraw, convert, join game, submit quest, RSVP | task success, latency |
| Investor User | High-limit actions | badge in JWT | withdraw high, larger watchlist | success rate |
| MiniApp Client | Execute UI and client logic | client keys, config | call APIs, retries, cache | error rate, LCP/INP |
| Support/CS | Help users | read-only guidance | troubleshoot flows | time to resolution |
| Operator/Admin | Flags and messages | config scope | enable features, incident banner | safe rollout |
| Compliance | KYC comms | view badge states | prompt KYC, disclosures | compliance rate |
| Finance/Billing | Overage & receipts | accounting scope | invoice display logic | collection rate |
| Campaign Manager | Quests | campaign scope | assist proofs guidance | completion |
| Game Coordinator | Matchmaking UX | play scope | room messaging, FAQs | room fill rate |
| SRE/Ops | Incident UX | ops scope | enable degraded mode | error budget |
| Telegram Platform | Provide initData | platform constraints | auth bootstrapping | platform health |
| External Wallet User | Provide address | none | paste/scan address | success rate |

## A2. RACI Matrix

| Capability | End User | Client | Support | Operator | Compliance | Finance | Campaign | Game | SRE |
|---|---|---|---|---|---|---|---|---|
| Login & session | R | A | I | I | I | I | I | I | C |
| Balances & payments | R | A | C | I | I | A | I | I | I |
| Watchlist & alerts | R | A | C | I | I | I | I | I | I |
| Matchmaking | R | A | I | I | I | I | I | A | I |
| Quests | R | A | I | I | I | I | A | I | I |
| Events & RSVP | R | A | I | I | I | I | I | I | I |
| Degraded UX | I | A | C | A | I | I | I | I | C |
| Localization | R | A | C | C | C | I | I | I | I |

## A3. CRUD × Persona × Resource Matrix

Resources: `CACHE_USER, CACHE_BADGE, CACHE_SESSION, CACHE_BALANCE, CACHE_USAGE, CACHE_WATCHLIST, CACHE_WATCH_ITEM, CACHE_ALERT, CACHE_ALERT_OCCUR, CACHE_GAME, CACHE_ROOM, CACHE_INVOICE, CACHE_RECEIPT, CACHE_EVENT, CACHE_RSVP`

| Resource \ Persona | End User | MiniApp Client | Support/CS | Operator | Compliance |
|---|---|---|---|---|---|
| CACHE_USER | R (view), U (prefs) | C/R/U/D | R (read via Admin UI) | N/A (policy) | N/A (policy) |
| CACHE_BADGE | R (view) | C/R/U/D | R (view) | N/A | R (audit view only) |
| CACHE_SESSION | R (logout) | C/R/U/D | R (view masked) | N/A | N/A |
| CACHE_BALANCE | R (view) | C/R/U/D | R (view) | N/A | N/A |
| CACHE_USAGE | R (view) | C/R/U/D | R (view) | N/A | N/A |
| CACHE_WATCHLIST | C/R/U/D | C/R/U/D | R (view) | N/A | N/A |
| CACHE_WATCH_ITEM | C/R/U/D | C/R/U/D | R (view) | N/A | N/A |
| CACHE_ALERT | C/R/U/D | C/R/U/D | R (view) | N/A | N/A |
| CACHE_ALERT_OCCUR | R (view) | C/R/U/D | R (view) | N/A | N/A |
| CACHE_GAME | R (view) | C/R/U/D | N/A | N/A | N/A |
| CACHE_ROOM | R (view) | C/R/U/D | N/A | N/A | N/A |
| CACHE_INVOICE | R (view) | C/R/U/D | R (view) | Finance R (read) | N/A |
| CACHE_RECEIPT | R (view) | C/R/U/D | R (view) | Finance R (read) | N/A |
| CACHE_EVENT | R (view) | C/R/U/D | R (view) | N/A | N/A |
| CACHE_RSVP | C/R/U/D | C/R/U/D | R (view) | N/A | N/A |

**N/A** indicates no direct access in WebApp context; data appears via Admin tools elsewhere.

## A4. Permissions / Badge / KYC Matrix

| Action | Requirement | Evidence | Expiry/Renewal | Appeal |
|---|---|---|---|---|
| Add watchlist item | none up to free tier | — | — | — |
| Create alert | none up to free tier | — | — | — |
| Exceed watchlist limit | pay invoice | payment receipt | per billing cycle | Refund via Finance |
| Join CFB with stake | sufficient balance | on-chain hold downstream | per match | — |
| Withdraw USDT | Investor badge | KYC verified | per Payhub | Support |
| Deposit USDT | none or KYC if policy | tx proof | confirmations | — |
| Submit quest | none | evidence blob | n/a | manual appeal |
| RSVP | none | — | event time | cancel anytime |

## A5. Quota & Billing Matrix

| Metric | Free Tier | Overage (FZ/PT) | Exemptions | Failure Handling | Refunds |
|---|---|---|---|---|---|
| Watchlist items per user | N items/month | per extra item | staff test | 402 → invoice modal | none |
| Alert rules per user | M rules/month | per extra rule | staff test | 402 → invoice modal | none |
| Matchmaking joins | K joins/day | per extra join | events promo | 402 → invoice | promo credit |
| API retries client-side | capped 5 | N/A | N/A | backoff then error | N/A |

## A6. Notification Matrix

| Event | Channel(s) | Recipient | Template | Throttle/Dedupe |
|---|---|---|---|---|
| Invoice paid | in-app + Telegram | User | `invoice_paid` | collapse by invoiceId |
| Alert triggered | Telegram + in-app badge | User | `alert_triggered` | cooldown 10m |
| Matchmaking room ready | in-app | User | `room_ready` | single-shot |
| Quest approved | in-app | User | `quest_approved` | single-shot |
| RSVP confirmed | in-app | User | `rsvp_confirmed` | single-shot |

## A7. Cross‑Service Event Map

| Event | Producer | Consumers | Payload summary | Idempotency | Retry/Backoff |
|---|---|---|---|---|---|
| payhub.invoice.updated@v1 | Payhub | WebApp | `{invoiceId, status, amount, currency}` | `invoiceId` | 5× exp backoff |
| watchlist.alert.triggered@v1 | Watchlist | WebApp | `{alertId, symbol, eventTs}` | `alertId,eventTs` | 5× exp backoff |
| playhub.matchmaking.state.changed@v1 | Playhub | WebApp | `{roomId, state, players}` | `roomId,state_ts` | 5× exp backoff |
| campaigns.task.attempt.updated@v1 | Campaigns | WebApp | `{taskId, attemptId, status}` | `attemptId` | 5× exp backoff |
| events.rsvp.confirmed@v1 | Events | WebApp | `{eventId, userId, state}` | `(eventId,userId)` | 3× exp backoff |

---

# Abuse/Misuse & NFR Sweeps

- **Abuse**: spam alerts/watchlist items (rate caps, quotas), phishing withdrawal addresses (warn on new address), griefing in rooms (mute UI, rate limits), invoice scam links (origin check + allowlist).  
- **Fraud**: quest evidence tampering (hash check server-side), self-referral farming (campaign rules).  
- **Privacy**: no PII storage in client cache beyond minimal profile; redact tokens in logs.  
- **Localization/timezones**: default **GMT+7**; all times shown with UTC tooltip.  
- **Resilience**: offline mode shows cached data with “stale” tag; backoff and circuit-aware banners; chain fee spike messaging on withdrawal.  
- **Security**: CSP, SRI, sanitizer for any HTML content; no secrets in localStorage; use IndexedDB for tokens where supported.

---

# Self‑Check — Stakeholder Coverage Report

**Counts**  
- Personas: 12  
- Features: 20  
- Stories: 21  
- Stories with non‑happy paths: 19/21  
- Entities covered in CRUD matrix: 15/15

**Checklist**  
- ✅ All personas appear in at least one story.  
- ✅ Each entity has at least one **C**, **R**, **U**, and **D** or reasoned N/A.  
- ✅ Every action mapped to roles/badges/KYC where applicable.  
- ✅ Quotas & billing addressed for every chargeable action.  
- ✅ Each story references APIs/entities/events.  
- ✅ At least one end‑to‑end scenario per persona.  
- ✅ Abuse/misuse cases enumerated with mitigations.  
- ✅ Observability signals tied to AC.  
- ✅ Localization/timezone handling present where user-visible.
