# User Stories: WEB3 Portal


# Section A — Personas & Stakeholders

> Derived from SystemDesign actors, auth scopes, and data ownership. Full catalog in **Appendix A1**.

- **Web3 User** — signs with wallet, manages assets, participates in funding, OTC escrows, quests, and campaigns, configures alerts, manages portfolio.
- **KYC’d Investor** — Web3 User holding **Investor badge**. Higher limits, OTC and funding participation, fiat ramps where applicable.
- **Project Owner** — creates campaigns, events, and escrow offers as a counterparty, manages allocations (via Admin/Backoffice but flows surface in portal).
- **Counterparty (OTC)** — accepts escrow offers, funds and settles contracts.
- **Portal Client** — the web app, using Shared SDK, WalletConnect and on-chain RPCs.
- **Operator/Admin** — sets feature flags, allowlists, incident banners (via Config/Admin).
- **Compliance** — validates KYC prompts, disclosures.
- **Finance/Billing** — handles overage invoices and receipts with Payhub.
- **Custody/Settlement** — downstream service managing holds, settlement, ledger (Payhub/Escrow).
- **SRE/Ops** — monitors telemetry, enables degraded modes, incident comms.
- **External Wallet Provider** — MetaMask, WalletConnect, TON/EVM wallets, Ledger.
- **Price/Oracle Provider** — signs snapshots consumed for quotes (Price Service).

---

# Section B — Epics → Features → User Stories (exhaustive)

## Epic B1 — Identity & Session (Web Login)

### Feature B1.1 — Web login with PKCE
**Story B1.1.1** — *As a Web3 User, I want to log in to the Portal,* so that *I can access my account and link wallets.*  
**Acceptance (GWT)**
1. **Given** I initiate login, **when** the Portal performs PKCE with Identity, **then** I receive a session JWT and profile.
2. **Given** the code verifier is invalid, **when** the callback occurs, **then** Identity returns **400 invalid_pkce**, the Portal shows “Try again” with a new challenge.
3. **Given** I’m already signed in, **when** I open the Portal, **then** the session refresh occurs silently on focus.
4. **Given** rate limit exceeded, **when** I retry, **then** I see **429** UX with countdown.

**Non‑Happy**: clock skew, blocked third‑party cookies, email‑magic fallback if configured.  
**Permissions**: none pre-login.  
**Data**: `CACHE_SESSION`, `CACHE_USER`, `CACHE_BADGE`.  
**Observability**: `login.pkce.latency`, error codes distribution.

### Feature B1.2 — Link / unlink wallet
**Story B1.2.1** — *As a Web3 User, I want to link my wallet,* so that *I can sign and withdraw.*  
**Acceptance**
1. Connect wallet via WalletConnect or injected provider.
2. The Portal asks user to sign **“Link this wallet”** message, including nonce, domain, and expiry.
3. On success, Identity records the address and chain, and a confirmation toast appears.
4. Unlink requires signature or session re-auth; if the wallet is primary for withdrawals, show guard rail.

**Non‑Happy**: wrong chain → prompt to switch; signature denied → cancel cleanly; nonce replay → **400 replayed_nonce**; hardware wallet locked → retry guidance.  
**Security**: domain binding, SIWE/TW hash canonicalization, anti-phishing banner.

---

## Epic B2 — Portfolio, Balances, and Billing

### Feature B2.1 — View portfolio & balances
**Story B2.1.1** — *As a Web3 User, I want to see my balances by asset and chain,* so that *I can plan actions.*  
**AC**: GET `/payhub/v1/wallets/balances` grouped by chain and token; stale‑while‑revalidate, “last updated” label.

### Feature B2.2 — Deposit & Withdraw (USDT and supported tokens)
**Story B2.2.1** — *As a KYC’d Investor, I want to withdraw to my wallet,* so that *I can move funds securely.*  
**AC**
1. Withdrawal requires Investor badge; 403 otherwise with CTA to apply.
2. Network selection shows min, fee estimate, confirmations; confirm screen displays irreversible warning.
3. 202 submitted → status pill with polling; receipt view when settled.
**Non‑Happy**: fee spikes, RPC failures, reorgs → “Delayed” and retry/backoff messaging.

### Feature B2.3 — Overages and invoices
**Story B2.3.1** — *As a Web3 User, I want to pay overage invoices,* so that *I can exceed free limits.*  
**AC**: POST `/payhub/v1/invoices`, pay via FZ/PT/USDT; progress bar until paid, then action unblocked.

---

## Epic B3 — Prices, Watchlist, and Alerts

### Feature B3.1 — Add to watchlist
**Story B3.1.1** — *As a Web3 User, I want to add tokens to a watchlist,* so that *I can track markets.*  
**AC**: POST `/watchlist/v1/lists/{id}/items` idempotent; 402 → invoice flow; duplicates → 409.

### Feature B3.2 — Price alert rules
**Story B3.2.1** — *As a Web3 User, I want threshold alerts,* so that *I can respond to moves.*  
**AC**: POST `/watchlist/v1/alerts { symbol, type, threshold, quietHours }`; cooldown and quiet hours enforced; delivery to Telegram and email if linked.

### Feature B3.3 — Charts
**Story B3.3.1** — *As a Web3 User, I want OHLCV charts,* so that *I can analyze trends.*  
**AC**: GET `/price/v1/ohlcv` intervals `[1m,5m,1h,4h,1d]`, handles 429 with backoff.

---

## Epic B4 — Funding & Vesting

### Feature B4.1 — Discover offerings
**Story B4.1.1** — *As a KYC’d Investor, I want to view investment opportunities,* so that *I can subscribe.*  
**AC**: list with filters, risk disclosures, soft caps; eligibility via badge check.

### Feature B4.2 — Allocate and vest
**Story B4.2.1** — *As a KYC’d Investor, I want to allocate and track vesting,* so that *I know unlocks.*  
**AC**
1. Allocation requires balance and KYC; 402 triggers invoice, 403 triggers KYC prompt.
2. Vesting schedule page shows cliffs and linear unlocks; claim buttons gated by schedule.
3. Receipts stored in Payhub, downloadable as CSV/PDF.

**Non‑Happy**: allocation window closed → 409; vesting revocation by issuer → banner and email.

---

## Epic B5 — Escrow (OTC/P2P)

### Feature B5.1 — Create escrow offer
**Story B5.1.1** — *As a Project Owner, I want to create a P2P escrow offer,* so that *counterparties can settle safely.*  
**AC**
1. Fill form with asset, amount, terms, expiry, and counterparty allowlist or open listing.
2. Confirm hold via Payhub, on‑chain signatures where required; 201 returns `escrowId`.
3. Offer appears in “My Escrows” with status `open`.

### Feature B5.2 — Accept and settle
**Story B5.2.1** — *As a Counterparty, I want to accept and settle an escrow,* so that *I can complete the trade.*  
**AC**
1. Accept requires KYC when exceeding limits.
2. Settlement triggers ledger updates and receipts; 200 on success, 409 if already accepted.
3. Dispute flow sends case to Support; funds held until resolution.

**Non‑Happy**: oracle price mismatch → 409 re‑quote; chain reorg → retry settlement.

---

## Epic B6 — Campaigns & Quests (On-chain aware)

### Feature B6.1 — Complete quest with on-chain proof
**Story B6.1.1** — *As a Web3 User, I want to submit on‑chain proof for quests,* so that *I can earn rewards.*  
**AC**: submit evidence including tx hash; backend verifies chain state and confirms.

---

## Epic B7 — Events & RSVPs (Portal)

### Feature B7.1 — RSVP
**Story B7.1.1** — *As a Web3 User, I want to RSVP to events and get reminders,* so that *I attend on time.*  
**AC**: same contracts as WebApp, with calendar export (ICS) option.

---

## Epic B8 — Settings, Security, and Keys

### Feature B8.1 — Address book and allowlists
**Story B8.1.1** — *As a KYC’d Investor, I want to maintain an address book,* so that *withdrawals are safer.*  
**AC**: add/edit/delete addresses, require signature to add, soft delay before first use.

### Feature B8.2 — Session & devices
**Story B8.2.1** — *As a Web3 User, I want to manage active devices,* so that *I can log out remotely.*  
**AC**: list sessions, revoke, force refresh across devices.

---

## Epic B9 — Reliability, Observability, and Incident UX

### Feature B9.1 — Degraded modes
**Story B9.1.1** — *As a User, I want clear guidance during chain or RPC outages,* so that *I avoid failed actions.*  
**AC**: incident banner with affected chains, disable risky operations, show ETA.

### Feature B9.2 — Telemetry and consent
**Story B9.2.1** — *As SRE, I want client telemetry with consent,* so that *we diagnose issues.*  
**AC**: Web Vitals, API traces, wallet connect metrics (connect, switch, sign), opt‑in banner.

---

# Section C — End‑to‑End Scenarios (Swimlane narrative)

1. **E2E‑C1: Link Wallet → Deposit → Allocate Funding → Vesting Track**  
   Pre: Logged in, no linked wallet. Flow: Link wallet signature → deposit USDT → allocate to offering → schedule appears with next unlock → receipt stored. Post: Allocation confirmed, next claim date visible.

2. **E2E‑C2: Create Escrow → Counterparty Accepts → Settlement**  
   Pre: KYC’d user. Flow: Create offer (hold placed) → counterparty accepts → settlement executes → receipts issued. Post: Funds transferred, audit trail complete.

3. **E2E‑C3: Watchlist Add → Alert Triggered → Action**  
   Flow: Add asset → rule set → alert fired during quiet hours → deduped until window ends → user clicks through to chart and converts.

4. **E2E‑C4: Withdraw → Address Book Guard → Success**  
   Flow: Choose saved address → sign confirmation → withdrawal submitted → receipt view.

---

# Section D — Traceability Matrix

| Story | APIs | Entities/Cache | Events | Diagrams (SystemDesign) |
|---|---|---|---|---|
| B1.1.1 | Identity PKCE endpoints | session cache | identity.session.created@v1 | §Auth |
| B1.2.1 | Wallet link endpoints | wallet link cache | identity.wallet.linked@v1 | §Auth |
| B2.1.1 | `/payhub/v1/wallets/balances` | balance cache | payhub.balance.updated@v1 | §Portfolio |
| B2.2.1 | `/payhub/v1/withdrawals`, `/deposits` | receipts | payhub.withdrawal.updated@v1 | §Payments |
| B2.3.1 | `/payhub/v1/invoices` | invoice cache | payhub.invoice.updated@v1 | §Billing |
| B3.1.1 | `/watchlist/v1/lists/{id}/items` | watchlist cache | watchlist.item.created@v1 | §Watchlist |
| B3.2.1 | `/watchlist/v1/alerts` | alert cache | watchlist.alert.triggered@v1 | §Alerts |
| B3.3.1 | `/price/v1/ohlcv` | — | price.snapshot.updated@v1 | §Charts |
| B4.1.1 | Funding browse | allocation cache | funding.allocation.updated@v1 | §Funding |
| B4.2.1 | Funding allocate/claim | vesting cache | funding.vesting.claimed@v1 | §Funding |
| B5.1.1 | Escrow create | escrow cache | escrow.offer.created@v1 | §Escrow |
| B5.2.1 | Escrow accept/settle | escrow cache | escrow.settled@v1 | §Escrow |
| B6.1.1 | Campaigns attempts | — | campaigns.task.attempt.updated@v1 | §Campaigns |
| B7.1.1 | Events list/rsvp | event cache | events.rsvp.confirmed@v1 | §Events |
| B8.1.1 | Address book | address cache | payhub.address.created@v1 | §Security |
| B8.2.1 | Sessions list/revoke | session cache | identity.session.revoked@v1 | §Security |
| B9.1.1 | Incident banner | — | status.banner.changed@v1 | §Reliability |
| B9.2.1 | Telemetry consent | — | — | §Observability |

**Deltas vs old UserStories**: adds wallet link signature flows, address book, escrow open/accept/settle lifecycle, vesting claims, incident chain‑aware banners, telemetry consent, and invoice overage UX carried over from WebApp.

---

# Section E — Assumptions & Open Questions

- Supported chains for the Portal (EVM, TON) and wallet providers list.  
- Exact KYC tiers for withdrawal and funding thresholds.  
- Address book first‑use delay policy (e.g., 24h).  
- Email channel availability for alerts and receipts.

---

# Appendix — Coverage Gates

## A1. Stakeholder Catalog

| Stakeholder | Responsibilities | Permissions | Typical Actions | KPIs/SLO interests |
|---|---|---|---|---|
| Web3 User | Use Portal with wallet | session JWT | link wallet, deposit, withdraw, allocate, watchlist, alerts | conversion, latency |
| KYC’d Investor | High-limit actions | badge | withdrawals, allocations, escrow | payout success |
| Project Owner | Offers & escrows | staff/project badge | create offers, manage events | fill rate |
| Counterparty (OTC) | Accept offers | badge if limits | accept, settle | settlement time |
| Portal Client | Execute UX | shared SDK | retries, cache, telemetry | error rate, LCP |
| Operator/Admin | Flags, banners | config scope | enable features, incident comms | safe rollout |
| Compliance | KYC policies | view decisions | disclosures, appeals | compliance rate |
| Finance/Billing | Billing flows | finance scope | invoices, refunds | collection rate |
| Custody/Settlement | Holds & ledger | internal | settlements, receipts | reconciliation |
| SRE/Ops | Reliability | ops scope | degraded mode, alerts | error budget |
| External Wallet Provider | Wallet UX | none | connect, sign, switch | success rate |
| Price/Oracle Provider | Price feed | signing keys | snapshots | freshness |

## A2. RACI Matrix

| Capability | Web3 User | Portal Client | Project Owner | Counterparty | Operator | Compliance | Finance | Custody | SRE |
|---|---|---|---|---|---|---|---|---|
| Login (PKCE) | R | A | I | I | I | I | I | I | C |
| Wallet link | R | A | I | I | I | I | I | I | I |
| Portfolio & balances | R | A | I | I | I | I | C | C | I |
| Deposit/Withdraw | R | A | I | I | I | C | C | A | I |
| Overages/Billing | R | A | I | I | I | I | A | I | I |
| Watchlist/Alerts | R | A | I | I | I | I | I | I | I |
| Funding/Vesting | R | A | I | I | I | C | I | I | I |
| Escrow OTC | R | A | A | R | I | C | I | A | I |
| Events/RSVP | R | A | A | I | I | I | I | I | I |
| Incident UX | I | A | I | I | A | I | I | I | C |

## A3. CRUD × Persona × Resource Matrix

Resources: `WALLET_LINK, ADDRESS_BOOK, BALANCE_CACHE, INVOICE, RECEIPT, WATCHLIST, WATCH_ITEM, ALERT_RULE, ALERT_OCCURRENCE, FUNDING_OFFER, ALLOCATION, VESTING_SCHEDULE, VESTING_CLAIM, ESCROW_OFFER, ESCROW_CONTRACT, EVENT_ITEM, RSVP, SESSION_CACHE`

| Resource \ Persona | Web3 User | KYC’d Investor | Project Owner | Counterparty | Portal Client |
|---|---|---|---|---|---|
| WALLET_LINK | C/R/U/D | C/R/U/D | N/A | N/A | C/R/U/D |
| ADDRESS_BOOK | C/R/U/D | C/R/U/D | N/A | N/A | C/R/U/D |
| BALANCE_CACHE | R | R | R | R | C/R/U/D |
| INVOICE | R | R | R | R | C/R/U/D |
| RECEIPT | R | R | R | R | C/R/U/D |
| WATCHLIST | C/R/U/D | C/R/U/D | R | R | C/R/U/D |
| WATCH_ITEM | C/R/U/D | C/R/U/D | R | R | C/R/U/D |
| ALERT_RULE | C/R/U/D | C/R/U/D | R | R | C/R/U/D |
| ALERT_OCCURRENCE | R | R | R | R | C/R/U/D |
| FUNDING_OFFER | R | R | C/R/U/D | R | C/R/U/D |
| ALLOCATION | C/R/U/D | C/R/U/D | R | R | C/R/U/D |
| VESTING_SCHEDULE | R | R | C/R/U/D | R | C/R/U/D |
| VESTING_CLAIM | C/R/U/D | C/R/U/D | R | R | C/R/U/D |
| ESCROW_OFFER | R | R | C/R/U/D | R | C/R/U/D |
| ESCROW_CONTRACT | R | R | R | C/R/U/D | C/R/U/D |
| EVENT_ITEM | R | R | C/R/U/D | R | C/R/U/D |
| RSVP | C/R/U/D | C/R/U/D | R | R | C/R/U/D |
| SESSION_CACHE | R | R | R | R | C/R/U/D |

N/A indicates intentionally not exposed.

## A4. Permissions / Badge / KYC Matrix

| Action | Requirement | Evidence | Expiry/Renewal | Appeal |
|---|---|---|---|---|
| Withdraw over X/day | Investor badge | KYC docs | policy window | Support |
| Allocate to offering | Investor badge | KYC pass | per offering | Compliance |
| Accept escrow | Investor badge if limits | KYC | per contract | Support |
| Create escrow | Project badge or allowlist | project docs | 1 year | Admin |
| Link wallet | session | signature | none | — |
| Add address | session | signature | soft delay | Support |

## A5. Quota & Billing Matrix

| Metric | Free Tier | Overage (FZ/PT) | Exemptions | Failure Handling | Refunds |
|---|---|---|---|---|---|
| Watchlist items | N/month | billed per extra | staff test | 402 → invoice | none |
| Price alerts | M/month | billed per extra | staff test | 402 → invoice | none |
| Escrow offers open | K active | billed per extra | project partners | 402 → invoice | prorata |
| Funding allocations edits | L/month | billed per extra | none | 402 | case-by-case |

## A6. Notification Matrix

| Event | Channel(s) | Recipient | Template | Throttle/Dedupe |
|---|---|---|---|---|
| Withdrawal settled | email + in-app | User | `withdrawal_settled` | collapse by txId |
| Allocation confirmed | email + in-app | Investor | `allocation_confirmed` | single |
| Escrow accepted | in-app | Project, Counterparty | `escrow_accepted` | collapse by contractId |
| Alert triggered | Telegram/email + in-app | User | `alert_triggered` | cooldown 10m |
| Vesting claim ready | in-app | Investor | `vesting_ready` | single per period |

## A7. Cross‑Service Event Map

| Event | Producer | Consumers | Payload summary | Idempotency | Retry/Backoff |
|---|---|---|---|---|---|
| payhub.invoice.updated@v1 | Payhub | Portal | `{invoiceId, status}` | `invoiceId` | 5× |
| payhub.withdrawal.updated@v1 | Payhub | Portal | `{withdrawalId, status, txId}` | `withdrawalId` | 5× |
| funding.allocation.updated@v1 | Funding | Portal | `{allocationId, status}` | `allocationId` | 5× |
| funding.vesting.claimed@v1 | Funding | Portal | `{claimId, scheduleId}` | `claimId` | 5× |
| escrow.offer.created@v1 | Escrow | Portal | `{escrowId, terms}` | `escrowId` | 5× |
| escrow.settled@v1 | Escrow | Portal | `{escrowId, txId}` | `escrowId` | 5× |
| watchlist.alert.triggered@v1 | Watchlist | Portal | `{alertId, symbol, ts}` | `alertId,ts` | 5× |

---

# Abuse/Misuse & NFR Sweeps

- **Abuse**: phishing through signature prompts (human-readable signing text), OTC griefing (holds and dispute flow), escrow spam (quotas + allowlists), alert spam (rate caps).  
- **Fraud**: funding misrepresentation (disclosures, audit trail), wash trading on OTC (limits, KYC).  
- **Security**: chain reorgs, RPC outages, gas spikes; signature domain-binding and nonce; address-book soft delay.  
- **Privacy**: no PII beyond profile; KYC handled by Identity/Compliance services.  
- **Localization/timezones**: default **GMT+7**, display local with UTC tooltips.  
- **Resilience**: degraded incident banners with chain list; retries with jitter; cached portfolio view.


