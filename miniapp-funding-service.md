# Miniapp-Funding-Service
*Version:* v0.1.0  
*Last Updated:* 2025-09-24 02:22 +07  
*Owner:* FUZE Capital Formation Engineering — Funding

> High‑level architectural blueprint for the **Funding Service** powering private sales, IDO style allocations, and presales within the Telegram Mini App. MVP operates with **off chain custody** using STAR, FZ, PT, FUZE, USDT via Payhub. It manages sale definitions, whitelists, purchase flows, referral benefits, allocations, and vesting schedules. No direct ledger mutations — all value changes go through **Payhub**. Identity provides auth and referral binding. Admin governs listings and audits. Workers handle vesting unlocks and retries.

---

## 1) Architecture Diagram
```mermaid
flowchart LR
  subgraph Clients
    WEB[Telegram WebApp - Nextjs]:::client
    ADMIN[Admin Console - Staff]:::client
  end

  subgraph Funding
    API[/Public REST API\nhealthz and readyz/]:::svc
    ENGINE[Sale Engine and Allocations]:::logic
    VEST[Vesting Scheduler]:::logic
    DB[(MongoDB - Sale Tier Whitelist Purchase Allocation Vesting Audit)]:::db
    CACHE[(Redis - idem locks rate limits)]:::cache
  end

  subgraph Core Services
    IDP[Identity - JWKS profiles referrals]:::peer
    PAY[Payhub - holds settlements conversions]:::peer
    PRICE[Price Service - quotes and bars]:::peer
    WORK[Workers - unlocks retries DLQ]:::peer
    CFG[Config - flags limits]:::peer
  end

  WEB -->|browse purchase claim| API
  ADMIN -->|create publish manage| API

  API --> ENGINE
  API --> VEST
  API --> DB
  API --> CACHE

  ENGINE --> PAY
  ENGINE --> PRICE
  VEST --> PAY
  CFG --> Funding
  WORK -->|vesting unlock and retries| API

  classDef client fill:#E3F2FD,stroke:#1E88E5;
  classDef svc fill:#E8F5E9,stroke:#43A047;
  classDef logic fill:#F1F8E9,stroke:#7CB342;
  classDef db fill:#FFF3E0,stroke:#FB8C00;
  classDef cache fill:#F3E5F5,stroke:#8E24AA;
  classDef peer fill:#ECEFF1,stroke:#546E7A;
```
*Notes:* Funding owns sale metadata and allocation math. Purchases place **holds** or perform **deposit credits** via Payhub depending on flow. Vesting creates scheduled **grants** and releases as credits over time. Identity referral codes can apply bonus allocations or whitelist priority per config.

---

## 2) Technology Stack
| Layer | Choice | Rationale |
|---|---|---|
| Runtime | Nodejs 20 plus TypeScript | Shared stack |
| Framework | Express plus Zod | Predictable DTO validation |
| Storage | MongoDB | Flexible sale and vesting models |
| Cache | Redis | Idempotency, locks, rate limits, schedules |
| Auth | jose Ed25519 JWT | End user sessions and service JWTs |
| Telemetry | OpenTelemetry plus Pino | Standard tracing and logs |
| Config | tg miniapp config | Fee bps, caps, asset allow list |
| Deploy | Docker plus Helm | Same CI and CD as other services |

---

## 3) Responsibilities and Scope
**Owns**
- **Sales**: define offerings, tiers, price schedules, per user caps, and whitelists.  
- **Purchases**: secure purchase path with idempotency and optional conversion quotes.  
- **Allocations**: compute purchased allocation amounts and referral bonuses.  
- **Vesting**: schedule releases with cliffs and linear unlocks; mint credits on schedule.  
- **Claims**: expose read‑only allocation and vesting status to users and Admin.  
- **Audit**: immutable audit trail for purchases and unlocks.

**Out of scope**
- On chain token distribution in MVP (planned adapter).  
- KYC collection and storage — integrate with external provider if required, store provider reference only.  
- Public price discovery; relies on **Price Service** or fixed price per tier.

---

## 4) Data Flows

### 4.1 Purchase Flow (hold then capture)
```mermaid
sequenceDiagram
participant UI as WebApp
participant FUN as Funding
participant PAY as Payhub
UI->>FUN: POST /v1/sales/:id/purchase with currency and amount + Idem
FUN->>FUN: Validate sale window, caps, whitelist, tier
FUN->>PAY: Create hold for amount
PAY-->>FUN: 200 holdId
FUN->>FUN: Compute units = floor(amount / price)
FUN->>FUN: Apply referral bonus if any
FUN-->>UI: 200 purchaseId and status pending with units
Note over FUN: On finalize or at end of window
FUN->>PAY: Settle hold outcome win to treasury
PAY-->>FUN: 200 settlementId
FUN->>FUN: Mark purchase paid and update allocation and vesting
```
### 4.2 Vesting Release
```mermaid
sequenceDiagram
participant WRK as Workers
participant FUN as Funding
participant PAY as Payhub
WRK->>FUN: POST /internal/v1/vesting/release { saleId, userId }
FUN->>FUN: Compute releasable units
alt units > 0
  FUN->>PAY: deposits credit treasury->user amount for releasable units
  PAY-->>FUN: 200 { refId }
  FUN->>FUN: Update VestingSchedule and Allocation claimed units
end
```

### 4.3 Refunds
- If a purchase is refundable (policy or failure), Admin triggers refund: Funding settles hold with `loss` or issues a negative credit by agreement; updates Allocation accordingly. All refunds audited.

---

## 5) Rules and Calculations

- **Price**: `units = floor(amountPaid / pricePerUnit)` using integer math.  
- **Referral bonus**: `bonusUnits = floor(units * referralBonusBps / 10000)` if parent exists.  
- **Caps**: enforce `perUserCap` and `hardCap`; holds rejected once caps would be exceeded.  
- **Vesting**:  
  - *cliff*: all unlocked at `cliffAt`.  
  - *linear*: unlock `rate = totalUnits / (endAt - startAt)`; release in discrete steps.  
  - *cliff_linear*: release `X%` at cliff then linear thereafter.  
- **Rounding**: deterministic rounding down; remainders accumulate and release on final epoch.  
- **Currencies**: allowed by config; conversion path uses Payhub conversions and Price snapshots for quotes.

---

## 6) Security and Compliance
- **Auth**: Identity session tokens for users; service JWTs for internal calls.  
- **Idempotency**: required for all POSTs; Redis backed keys retained 48 h.  
- **Rate limits**: per user purchase rate and per sale caps; Redis counters.  
- **Validation**: Zod DTOs; integer amounts; asset and currency allow‑lists.  
- **Audit**: every sale state change, purchase, and vesting release logged.  
- **KYC**: optional external integration — store only `providerRef` and decision, never raw documents.  
- **Secrets**: managed via secret manager; no secrets in client or repo.  
- **Abuse**: deny lists and bot heuristics; referral self‑dealing prevented (`userId != parentId`).  
- **Legal**: display disclaimers per jurisdiction and restrict countries via allow lists if configured.

---

## 7) Scalability and Reliability
- Stateless API; horizontal scale; Redis for hot paths and schedules.  
- MongoDB with compound indexes for sales and purchases.  
- Workers handle vesting releases and retries with backoff.  
- SLOs: p95 < 150 ms on reads and < 300 ms on purchase mutations.  
- Health checks `/healthz` and `/readyz` include DB, Redis, and config freshness.  
- DR: backups; PITR recommended.

---

## 8) Observability
- **Tracing**: propagate `requestId`, `saleId`, `purchaseId`, `vestingId`.  
- **Metrics**: purchases per minute, conversion latency, unlock throughput, refund rate.  
- **Logs**: structured redacted logs; no sensitive PII.  
- **Alerts**: cap near exhaustion, settlement failures, vesting backlog spike.

---

## 9) User Stories and Feature List
### Feature List
- Sales and tier definitions with caps and whitelists.  
- Purchases with holds and conversions.  
- Allocation and referral bonuses.  
- Vesting schedules and automated releases.  
- Admin dashboards and exports.

### User Stories
- *As a participant*, I can purchase a presale allocation using USDT or platform currencies and see my vesting.  
- *As a project owner*, I can configure a sale with tiers and caps and monitor purchases.  
- *As a referrer*, I receive bonus allocation based on my referees purchases.  
- *As finance*, I can audit allocations and releases with clear receipts.

---

## 10) Roadmap
- On chain distribution adapter with Merkle claims or direct transfers.  
- Dynamic pricing tiers and oversubscription handling.  
- KYC gating and jurisdiction restrictions.  
- Secondary transfer of allocations within policy.

---

## 11) Compatibility Notes
- Works with Identity for auth and referrals.  
- Uses Payhub for all value operations and optional conversions.  
- Uses Price snapshots when needed.  
- Surfaces flows to WebApp and Admin; Workers run unlocks and retries.
