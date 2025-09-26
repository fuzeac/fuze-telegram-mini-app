# Miniapp-Funding-Service
*Version:* v0.1.0  
*Last Updated:* 2025-09-24 02:22 +07  
*Owner:* FUZE Capital Formation Engineering — Funding

> High‑level architectural blueprint for the **Funding Service** powering private sales, IDO style allocations, and presales within the Telegram Mini App. MVP operates with **off chain custody** using STAR, FZ, PT, FUZE, USDT via Payhub. It manages sale definitions, whitelists, purchase flows, referral benefits, allocations, and vesting schedules. No direct ledger mutations — all value changes go through **Payhub**. Identity provides auth and referral binding. Admin governs listings and audits. Workers handle vesting unlocks and retries.

---

## 1) Architecture Diagram

```mermaid
flowchart LR
  subgraph "Clients"
    TG["Telegram WebApp"]
    W3["Web3 Portal"]
    ADM["Admin Console"]
  end

  subgraph "Funding Service"
    API["REST API v1"]
    EVAL["Evaluation Engine"]
    VOTE["Voting Manager"]
    SALE["Sale Manager"]
    PUR["Purchase Engine"]
    VEST["Vesting Manager"]
    EVT["Event Dispatcher"]
    DB["MongoDB"]
    RDS["Redis cache, jobs"]
    WRK["Workers schedulers"]
  end

  subgraph "Platform Services"
    ID["Identity"]
    PAY["Payhub"]
    PRICE["Price Service"]
    CFG["Config Service"]
    ADMIN["Admin Service"]
  end

  TG -->|submit project, browse, purchase| API
  W3 -->|browse, purchase| API
  ADM -->|review dashboards, overrides| API

  API --> EVAL
  API --> VOTE
  API --> SALE
  API --> PUR
  API --> VEST
  API --> DB
  API --> RDS
  WRK --> EVAL
  WRK --> VOTE
  WRK --> SALE
  WRK --> PUR
  WRK --> VEST
  WRK --> EVT

  API --> ID
  PUR --> PAY
  VEST --> PAY
  API --> PRICE
  API --> CFG
  ADMIN --> API
  EVT --> ADMIN
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

### 4.1 Project submission → AI scoring → Voting → Decision

```mermaid
sequenceDiagram
  autonumber
  participant UI as "Client"
  participant FN as "Funding API"
  participant ID as "Identity"
  participant CF as "Config Service"
  participant WR as "Workers"
  UI->>FN: POST /v1/projects { title, summary, metadata } (Idem)
  FN->>ID: verify badge 'Project Owner'
  alt badge ok
    FN->>CF: fetch thresholds and window
    FN->>FN: create PROJECT and EVALUATION, state=submitted
    FN-->>UI: 201 Created
    WR->>FN: start AI scoring job
    FN->>FN: aiScore = 0..50
    alt aiScore > 40
      FN->>FN: open voting, set windowEnd = now + 5d
    else
      FN->>FN: reject, state=rejected_ai
    end
  else no badge
    FN-->>UI: 403 Forbidden
  end
  WR->>FN: on windowEnd, compute finalScore and decision
  FN->>FN: decision approved if finalScore >= 75
```

### 4.2 Preview → Hold → Settle with cap and Investor bypass

```mermaid
sequenceDiagram
  autonumber
  participant UI as "Client"
  participant FN as "Funding API"
  participant ID as "Identity"
  participant PR as "Price Service"
  participant PY as "Payhub"
  UI->>FN: POST /v1/sales/{{saleId}}/preview { currency, amount }
  FN->>ID: check badges 'Investor' for cap bypass
  FN->>PR: convert amount to USDT equivalent
  alt cap exceeded and not Investor
    FN-->>UI: 403 cap_exceeded with remaining headroom
  else ok
    UI->>FN: POST /v1/sales/{{saleId}}/purchase { currency, amount } (Idem)
    FN->>PY: create hold for amount in STAR/FZ/PT
    alt hold ok
      FN-->>UI: 200 {{ purchaseId, status: pending }}
    else insufficient funds
      FN-->>UI: 409 insufficient_funds
    end
  end
  Note over FN: Workers finalize settlements at sale end
```

### 4.3 Vesting unlock

```mermaid
sequenceDiagram
  autonumber
  participant WR as "Worker"
  participant FN as "Funding API"
  participant PY as "Payhub"
  WR->>FN: scan due VESTING by unlockAt
  FN->>PY: credit off‑chain entitlement, record receipt
  PY-->>FN: receiptId
  FN->>FN: mark VESTING status=unlocked
```

### 4.4 Non-happy paths

```mermaid
sequenceDiagram
  autonumber
  participant FN as "Funding API"
  participant PY as "Payhub"
  participant PR as "Price Service"
  FN->>PR: convert amount at hold time
  PR-->>FN: timeout
  FN->>FN: fallback to cached snapshot if age < cfg.maxCacheAge, else return 503 retryable
  FN->>PY: create hold
  PY-->>FN: 429 rate limited
  FN->>FN: retry with backoff, circuit break after threshold
```

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
