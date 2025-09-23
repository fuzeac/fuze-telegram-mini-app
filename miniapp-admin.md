# Miniapp Admin
*Version:* v0.2.0  
*Last Updated:* 2025-09-24 03:22 +07  
*Owner:* FUZE Operations Engineering — Admin Dashboard

> High‑level architectural blueprint for the **Admin Dashboard**. This repo delivers the operator console for finance ops, support, moderation, growth, and game ops. It implements a **UI + BFF** pattern: the browser never calls core services directly; every sensitive action is proxied through the Admin BFF with strict **RBAC**, **two‑person approvals**, idempotency, and full **audit logs**.

---

## 1) Architecture Diagram (HLD)
```mermaid
flowchart LR
  subgraph Operator
    OP[Admin User]:::client
  end

  subgraph Admin
    UI[Admin UI]:::svc
    BFF[Admin API BFF]:::svc
    CACHE[(Redis sessions and rate limits)]:::cache
    DB[(MongoDB audit and admin prefs)]:::db
  end

  subgraph Core Services
    IDP[Identity]:::peer
    PAY[Payhub]:::peer
    PLAY[PlayHub]:::peer
    WATCH[Watchlist]:::peer
    CAMP[Campaigns]:::peer
    FUND[Funding]:::peer
    ESC[Escrow]:::peer
    EVTS[Events]:::peer
    PRICE[Price Service]:::peer
    WORK[Workers]:::peer
  end

  OP -->|browser https| UI
  UI -->|REST calls| BFF
  BFF -->|verify roles| IDP
  BFF --> CACHE
  BFF --> DB

  BFF --> PAY
  BFF --> PLAY
  BFF --> CAMP
  BFF --> WATCH
  BFF --> FUND
  BFF --> ESC
  BFF --> EVTS
  BFF --> PRICE
  BFF --> WORK

  classDef client fill:#E3F2FD,stroke:#1E88E5;
  classDef svc fill:#E8F5E9,stroke:#43A047;
  classDef db fill:#FFF3E0,stroke:#FB8C00;
  classDef cache fill:#F3E5F5,stroke:#8E24AA;
  classDef peer fill:#ECEFF1,stroke:#546E7A;
```
*Notes:* The BFF signs **service‑to‑service** requests with a service JWT. All privileged mutations require **capability checks** on the staff role and may require **two approvals** before execution (e.g., withdrawals, manual credits, forced settles).

---

## 2) Technology Stack
| Layer | Choice | Rationale |
|---|---|---|
| UI | React or Next.js (SPA) | Fast internal tooling, familiar stack |
| BFF | Node.js + Express + Zod | Simple, secure request validation and DTOs |
| Auth | OIDC with Identity or Staff Token | Centralized roles and entitlements |
| Storage | MongoDB | Append‑only audit logs, admin prefs, approval queue |
| Cache | Redis | Sessions, rate limits, CSRF nonces, short‑term approvals |
| Telemetry | OpenTelemetry + Pino | Uniform tracing and logs |
| Deploy | Docker + Helm | Same pipeline as other services |

---

## 3) Responsibilities and Scope
**Owns**
- Staff authentication, session management, RBAC and capability gating.  
- Read views across **Payhub**, **PlayHub**, **Funding**, **Escrow**, **Campaigns**, **Watchlist**, **Events**, **Price**.  
- Sensitive actions behind **two‑person approvals** (4‑eyes): withdrawal approve, manual deposit credit, force settle, dispute resolve, CFB recompute.  
- Full **audit** of every admin action with immutable history and exports.  
- Operator UX: search users, drill into balances, matches, bets, escrows, sales, claims, events.

**Out of scope**
- End‑user features and gameplay UI.  
- Long‑running jobs (Workers handle them).  
- Ledger math or price computation.

---

## 4) Primary Surfaces (UI)
- **Dashboard**: system health, recent audits, fast links.  
- **Users**: profile, roles, balances, recent actions.  
- **Finance**: accounts, ledger, holds, settlements, conversions; withdrawals review; manual credits.  
- **PlayHub**: rooms, matches, **CFB** bets, result summaries; safe recompute and force settle.  
- **Funding**: sales, purchases, allocations, vesting schedules; refunds.  
- **Escrow**: contracts, disputes; adjudication tools.  
- **Campaigns**: campaigns, claims; retry queue view.  
- **Watchlist**: sources and profiles moderation.  
- **Events**: submissions, approvals, catalog edits.  
- **Price**: snapshots and reports for audits.  
- **Approvals**: pending requests requiring a second approver.  
- **Audit**: export by time range and action type.

---

## 5) Critical Flows

### 5.1 Staff Login
```mermaid
sequenceDiagram
participant OP as Operator
participant UI as Admin UI
participant BFF as Admin BFF
participant IDP as Identity
OP->>UI: Open admin site
UI->>BFF: Start login
BFF->>IDP: OIDC or staff token exchange
IDP-->>BFF: Staff identity and roles
BFF-->>UI: Session cookie and profile
```

### 5.2 Two‑Person Approval (example: Manual Deposit Credit)
```mermaid
sequenceDiagram
participant UI as Admin UI
participant BFF as Admin BFF
participant PAY as Payhub
UI->>BFF: Request manual credit with user id and amount
BFF->>BFF: Create ApprovalRequest status pending
UI-->>BFF: Second approver approves
BFF->>PAY: Post deposit credit with correlation id
PAY-->>BFF: OK with deposit id
BFF->>BFF: Mark approval executed and write audit
BFF-->>UI: Success with receipt
```
*Policy:* requester cannot approve their own request. Expire pending requests after a timeout.

### 5.3 CFB Oversight and Safe Force Settle
```mermaid
sequenceDiagram
participant UI as Admin UI
participant BFF as Admin BFF
participant PH as PlayHub
UI->>BFF: Request CFB recompute or force settle
BFF->>BFF: Check capabilities and approval policy
BFF->>PH: Call internal route for recompute or settle
PH-->>BFF: OK with result status
BFF->>BFF: Write audit with bet id and request id
BFF-->>UI: Result summary
```

### 5.4 Withdrawal Review
```mermaid
sequenceDiagram
participant UI as Admin UI
participant BFF as Admin BFF
participant PAY as Payhub
UI->>BFF: Approve withdrawal request
BFF->>BFF: Two‑person approval check
BFF->>PAY: Approve withdrawal
PAY-->>BFF: Paid
BFF->>BFF: Audit and return receipt
BFF-->>UI: Success
```

---

## 6) Security and Compliance
- **RBAC** with capabilities per route (e.g., `finance.withdrawals.approve`).  
- **Two‑person rule** for high‑risk ops; enforced in BFF with ApprovalRequest.  
- **CSRF** protection for state‑changing routes; SameSite and secure cookies.  
- **Sessions** short TTL and rotation; IP and UA hints for anomaly detection.  
- **Input validation** with Zod; all writes require `Idempotency-Key`.  
- **Audit** every admin action with request id and payload hash.  
- **Secrets** in secret manager; never in client.  
- **Allow lists** for service JWT audiences and issuers; optional mTLS.  
- **PII** avoidance: use user ids; never store sensitive personal data.

---

## 7) Scalability and Reliability
- Stateless BFF nodes; horizontal scale.  
- Redis for sessions and rate limits; MongoDB for audits and approvals.  
- Idempotent mutations; retries safe with idempotency keys.  
- SLOs: p95 < 150 ms for read operations.  
- Health probes `/healthz` and `/readyz` include DB, Redis, JWKS freshness.  
- DR: backups for audit logs; key rotation runbooks.

---

## 8) Observability
- **Tracing**: propagate `requestId`; spans include target service and action.  
- **Metrics**: approvals pending, approval latency, error rates, downstream latency.  
- **Logs**: structured and redacted; include actor id and action.  
- **Alerts**: surge in sensitive actions, repeated auth failures, downstream outage.

---

## 9) User Stories and Feature List
### Feature List
- Staff login and session management.  
- Cross‑service read dashboards.  
- Two‑person approvals for sensitive ops.  
- CFB oversight and safe recompute.  
- Withdrawals review and manual credits.  
- Full audit and exports.

### User Stories
- *As a finance operator*, I approve withdrawals with two‑person controls and see clear receipts.  
- *As a game operator*, I inspect a match or CFB bet and trigger a safe recompute if callbacks failed.  
- *As support*, I search a user and view their balances and recent actions.  
- *As compliance*, I export audit logs and review sensitive actions.

---

## 10) Compatibility Notes
- Works with **Identity** JWKS and staff roles.  
- Proxies to **Payhub**, **PlayHub**, **Funding**, **Escrow**, **Campaigns**, **Watchlist**, **Events**, **Price**, **Workers** using service JWTs.  
- Never exposes internal service credentials to the browser; all calls go through the BFF.
