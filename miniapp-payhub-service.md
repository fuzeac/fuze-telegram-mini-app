# Miniapp Payhub Service
*Version:* v0.1.0  
*Last Updated:* 2025-09-24 03:00 +07  
*Owner:* FUZE Platform Finance — Ledger, Holds, Settlements

> High‑level architectural blueprint for the **Payhub Service**. Payhub is the **single source of truth for balances** in platform currencies (**USDT, FUZE, STAR, FZ, PT** in MVP). It provides internal primitives: **Accounts, Ledger, Holds, Settlements, Conversions, Deposits, Withdrawals**. No games or business rules live here. Domain services (PlayHub, Funding, Campaigns, Escrow) call Payhub to **lock value** and **settle outcomes**. End users never call Payhub directly — WebApp uses domain services; Admin uses a BFF.

---

## 1) Architecture Diagram
```mermaid
flowchart LR
  subgraph Callers
    PH["PlayHub Service"]
    FUN["Funding Service"]
    ESC["Escrow Service"]
    CMP["Campaigns Service"]
    ADM["Admin Service"]
    WEB["Telegram Mini App"]
  end

  subgraph "Payhub Service"
    API["HTTP API"]
    LED["Double-Entry Ledger"]
    HLD["Hold Manager"]
    SET["Settlement Engine"]
    FX["Conversion Engine (Quotes/Confirm)"]
    INV["Invoices & Overage Billing"]
    ACC["Accounts & Balances"]
    WH["Webhooks Dispatcher"]
    MTR["Metering"]
    JOB["Schedulers/Workers"]
    CCH["Redis Cache"]
    DB["MongoDB"]
    SIG["Signing (Ed25519/HMAC)"]
  end

  subgraph "Platform Services"
    IDN["Identity Service"]
    PRC["Price Service"]
    CFG["Config Service"]
    WRK["Global Workers/Dunning"]
  end

  subgraph "External"
    KMS["KMS/HSM"]
    CHA["Chain Gateways"]
  end

  PH -->|holds, settlements, rake| API
  FUN -->|holds, settlements, treasury payout| API
  ESC -->|holds, split payouts| API
  CMP -->|rewards/airdrop credits| API
  ADM -->|adjustments, webhooks, limits| API
  WEB -->|balances, conversions, invoices| API

  API --> ACC
  API --> LED
  API --> HLD
  API --> SET
  API --> FX
  API --> INV
  API --> MTR
  API --> WH
  API --> DB
  API --> CCH

  FX --> PRC
  INV --> WRK
  API --> IDN
  SIG --> KMS
  JOB --> CHA
```
*Notes:* Payhub performs **atomic** balance updates using Mongo transactions. All POSTs require `Idempotency-Key`. Domain services pass a **correlation id** (room id, bet id, escrow id) for traceability.

---

## 2) Technology Stack
| Layer | Choice | Rationale |
|---|---|---|
| Runtime | Nodejs 20 plus TypeScript | Shared platform stack |
| Framework | Express plus Zod | Predictable schemas and errors |
| Storage | MongoDB transactions | Per currency accounts and ledger entries |
| Cache | Redis | Idempotency, locks, and rate limits |
| Auth | jose Ed25519 JWT | Service to service auth |
| Telemetry | OpenTelemetry plus Pino | Tracing and structured logs |
| Deploy | Docker plus Helm | Uniform CI and CD |

---

## 3) Responsibilities and Scope
**Owns**
- **Accounts**: per user per currency balances with available and locked.  
- **Ledger**: append‑only double entry journal per movement with correlation id.  
- **Holds**: create and cancel holds that reduce available balance.  
- **Settlements**: settle holds to **win** or **loss** and move funds accordingly.  
- **Deposits**: credit from platform treasuries or external providers (MVP manual credit).  
- **Withdrawals**: request, review, and approve; debit to external addresses or off platform (MVP manual).  
- **Conversions**: optional internal FX between platform currencies using provided quotes.  

**Out of scope**
- Business decisions (who wins, bet logic, escrow dispute) — decided by domain services.  
- External custody or blockchain integration in MVP (future adapters).  
- End user UI — handled by WebApp via domain services or Admin BFF.

---

## 4) Data Flows

### 4.1 Hold then settle with 7% rake

```mermaid
sequenceDiagram
  autonumber
  participant PH as PlayHub
  participant PAY as Payhub
  PH->>PAY: POST /internal/v1/holds (Idem-Key)
  PAY->>PAY: Reserve funds, post journal
  PAY-->>PH: 200 { holdId }
  Note over PH,PAY: Later when outcome known
  PH->>PAY: POST /internal/v1/settlements { holdId, outcome, rakePct }
  PAY->>PAY: Compute payouts and fee
  PAY-->>PH: 200 { settlementId, receipt }
```

### 4.2 Conversion quote then confirm with overage charge

```mermaid
sequenceDiagram
  autonumber
  participant UI as Client
  participant PAY as Payhub
  participant PRC as Price

  UI->>PAY: POST /v1/conversions/quote (Idem-Key)
  PAY->>PRC: TWAP or spot rate
  PRC-->>PAY: rate, expiresAt
  PAY-->>UI: 200 { quoteId, rate, expiresAt, sig }
  UI->>PAY: POST /v1/conversions/{quoteId}/confirm
  PAY->>PAY: Check free tier, maybe create invoice
  alt Overage due
    PAY-->>UI: 402 Payment Required
  else Within free tier
    PAY->>PAY: Post journals, update balances
    PAY-->>UI: 200 Receipt
  end
```

### 4.3 Invoice payment flow

```mermaid
sequenceDiagram
  participant UI as Client
  participant PAY as Payhub

  UI->>PAY: GET /v1/invoices/{id}
  alt Unpaid
    PAY-->>UI: 200 { status: open, amount, currency }
    UI->>PAY: POST /v1/invoices/{id}/pay { currency }
    PAY-->>UI: 200 { status: paid, receipt }
  else Paid
    PAY-->>UI: 200 { status: paid }
  end
```

### 4.4 Webhook delivery with retry

```mermaid
sequenceDiagram
  participant PAY as Payhub
  participant SVC as Service

  PAY->>SVC: POST /webhook { eventType, payload, ts, sig }
  alt 2xx
    SVC-->>PAY: 204
  else Error
    SVC-->>PAY: 500
    PAY->>PAY: Schedule retry with backoff
  end
```


---

## 5) Accounting Model

- **Double entry**: every movement is recorded with a **debit** and a **credit** entry between two accounts.  
- **Holds**: reduce `available` and increase `locked` on the **holder** account; no ledger entries until settlement.  
- **Settlement win**: decrease holder `locked`, increase winner `available`; write two ledger entries: `hold_release` and `settlement_win`.  
- **Settlement loss**: decrease holder `locked`, increase **treasury** `available`.  
- **Release**: decrease `locked`, increase `available` on the same account.  
- **Fees**: optional fee basis points charged to the **winner** and credited to **treasury**.  
- **Conversions**: two ledger entries for the buy and fee; rate snapshot id recorded.

All operations are transactional and idempotent per `Idempotency-Key`.

---

## 6) Security and Compliance
- **Auth**: service JWTs with issuer and audience allow lists; optional mTLS.  
- **Idempotency**: Redis backed keys with 48 h retention for all POSTs.  
- **Rate limits**: per caller and per user operations.  
- **Validation**: Zod DTOs; currency allow list and integer amounts.  
- **Audit**: all POSTs create a ledger trail plus request audit record.  
- **Secrets**: signer keys and DB creds in secret manager.  
- **Privacy**: store user ids only; no PII.  
- **Change control**: withdrawal approvals require Admin two person rule.

---

## 7) Scalability and Reliability
- Stateless API nodes with Redis for idempotency; horizontal scale.  
- MongoDB transactional writes; tune write concerns and indexes.  
- SLOs: p95 < 120 ms for holds and settlements under light contention.  
- Health probes `/healthz` and `/readyz` include DB, Redis, and key freshness.  
- DR: daily backups; PITR recommended.

---

## 8) Observability
- **Tracing**: propagate `requestId`; spans annotate `userId`, `currency`, `holdId`.  
- **Metrics**: holds created per minute, settlement rates, locked vs available, failed idempotency conflicts.  
- **Logs**: structured logs without secrets; correlation id present.  
- **Alerts**: surge in failed settlements, DB latency, lock contention.

---

## 9) User Stories and Feature List
### Feature List
- Internal holds and settlements with optional fees.  
- Deposits credit and withdrawal review flow.  
- Ledger query and receipts for audits.  
- Optional conversions with quotes.

### User Stories
- *As PlayHub*, I can lock stakes and settle winners safely.  
- *As Funding*, I can credit participants and later process withdrawals.  
- *As Campaigns*, I can credit rewards from a treasury account idempotently.  
- *As Admin*, I can review withdrawals with a clear ledger trail.

---

## 10) Compatibility Notes
- Trusted by **PlayHub**, **Funding**, **Campaigns**, **Escrow** via internal network only.  
- WebApp does not call Payhub directly; Admin BFF proxies staff actions.  
- DTOs and error envelopes align with `tg-miniapp-shared`; flags come from `tg-miniapp-config`.
