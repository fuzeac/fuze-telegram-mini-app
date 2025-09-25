# Miniapp Playhub Service
*Version:* v0.2.0  
*Last Updated:* 2025-09-24 03:09 +07  
*Owner:* FUZE Games Platform — Matchmaking, Rooms, CFB v1

> High‑level architectural blueprint for **PlayHub**. This service powers PvP matchmaking and room orchestration for first‑party and partner games, and implements **Crypto Fair Bet v1 (CFB v1)** with off‑chain custody limited to **STAR, FZ, PT** for MVP. PlayHub performs **server‑to‑server** calls to game backends and Payhub, and reads prices from Price Service. End users never interact with Payhub directly.

---

## 1) Architecture Diagram
```mermaid
flowchart LR
  subgraph Client
    UI["Telegram Mini App"]
    WEB["Web3 Portal"]
  end

  subgraph "PlayHub Service"
    API["HTTP API"]
    MM["Matchmaking Engine"]
    GST["Game Session Tokens"]
    CFB["Crypto Fair Bet (v1 off-chain custody)"]
    ORA["Price Oracle Adapter"]
    EVQ["Events and Jobs"]
    DB["Primary DB"]
    CCH["Redis"]
  end

  subgraph "Platform Services"
    IDN["Identity Service"]
    PAY["Payhub Service"]
    PRC["Price Service"]
    CFG["Config Service"]
    ADM["Admin Service"]
    WRK["Workers Schedulers"]
  end

  subgraph "External Game"
    GMA["Game Service A (template)"]
  end

  UI -->|join match, view results| API
  WEB -->|create CFB, accept, track| API

  API --> MM
  API --> GST
  API --> CFB
  API --> DB
  API --> CCH
  API --> EVQ

  MM -->|create room| GMA
  GMA -->|post result| API

  API -->|verify badges| IDN
  API -->|holds and settlements| PAY
  CFB --> ORA
  ORA --> PRC
  API --> CFG
  EVQ --> WRK
```

*Notes:* Game services are **independent** deployments that trust PlayHub via allow lists. Game UIs authenticate to game backends using a **GST** (Game Session Token) minted by PlayHub.

---

## 2) Technology Stack
| Layer | Choice | Rationale |
|---|---|---|
| Runtime | Nodejs 20 and TypeScript | Shared stack across repos |
| Framework | Express and Zod | Schema validation and consistent errors |
| Storage | MongoDB | Flexible matches and bets |
| Cache | Redis | Pools, tickets, idempotency, rates |
| Auth | jose Ed25519 JWT | Session and GST tokens |
| Telemetry | OpenTelemetry and Pino | Tracing and logs |
| Deploy | Docker and Helm | Standard CI and CD |

---

## 3) Responsibilities and Scope
**Owns**
- **Matchmaking**: queue players by game and stake, pair, create rooms, mint GSTs, redirect to game URL.  
- **Results intake**: accept server‑to‑server results from game backends.  
- **Settlement**: call **Payhub** to settle stakes.  
- **CFB v1**: create and accept owner bets on token prices using STAR FZ PT only; compute results with Price Service and settle with rake.  
- **Audit**: immutable events and operator queries.

**Out of scope**
- Price ingestion and signing — delegated to **Price Service** and **Workers**.  
- Ledger management — delegated to **Payhub**.  
- Gameplay networking — in the game backend.

---

## 4) Flows

### 4.1 Matchmaking and Game Result
```mermaid
sequenceDiagram
  autonumber
  participant UI as Client
  participant PH as PlayHub
  participant ID as Identity
  participant PY as Payhub
  participant GM as "Game Service"

  UI->>PH: POST /v1/matchmaking/join (gameId, currency, amount, Idem-Key)
  PH->>ID: Introspect (gates based on stake thresholds)
  ID-->>PH: Allowed
  PH->>PY: POST /internal/v1/holds (amount, currency, purpose=playhub, metadata)
  PY-->>PH: 201 holdId
  PH->>PH: Enqueue into pool
  Note over PH: Another player arrives and matches
  PH->>GM: POST /internal/v1/rooms {{ roomId, players[] }}
  GM-->>PH: 200 Created
  PH->>PH: Mint GST for each player
  PH-->>UI: 200 {{ roomId, gst }}
```

### 4.2 CFB v1 Create and Accept
```mermaid
sequenceDiagram
  participant GM as "Game Service"
  participant PH as PlayHub
  participant PY as Payhub
  GM->>PH: POST /internal/v1/results {{ roomId, winnerId }}
  PH->>PH: Verify signature and room state
  PH->>PY: POST /internal/v1/settlements (p1 hold, outcome=win or loss, feeBps=700)
  PH->>PY: POST /internal/v1/settlements (p2 hold, outcome=win or loss, feeBps=700)
  PY-->>PH: 200 Settled
  PH->>PH: Mark match completed
```

### 4.3 CFB v1 Settle
```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant PH as PlayHub
  participant PY as Payhub
  participant PR as Price

  U->>PH: POST /v1/cfb/bets (symbol, timeframe, targetAt, acceptCloseAt, currency, amount, direction, Idem-Key)
  PH->>PH: Validate cutoff <= 0.5 * timeframe
  PH->>PY: Hold ownerAmount
  PY-->>PH: 201 holdId
  PH-->>U: 201 Created (open)

  U->>PH: POST /v1/cfb/bets/{{id}}/accept (amount, Idem-Key)
  PH->>PH: Ensure now < acceptCloseAt and state=open
  PH->>PY: Hold accept amount
  PY-->>PH: 201 holdId
  PH-->>U: 201 Accepted

  Note over PH: At targetAt, job triggers resolution
  PH->>PR: Get minute price at open and close
  PR-->>PH: Prices
  PH->>PH: Compute winner and payout shares
  PH->>PY: Settle holds accordingly with feeBps=700
  PY-->>PH: 200 Settled
  PH->>PH: Mark bet resolved and create payouts
```

**CFB payout policy (MVP)**  
**Matchmaking**
- Entry requires sufficient balance and may require Gamer badge above stake thresholds.
- Holds are created before entering pool. Cancellation releases hold.
- GST TTL default 10 minutes; single use.

**CFB v1**
- Owner sets timeframe in [1h, 2h, 3h, 1d], targetAt, acceptCloseAt <= half the timeframe.
- Stake currencies: FZ, PT, STAR only.
- Pot P = ownerAmount + sum(acceptAmount).
- Winner side receives P - fee, where fee = P * 0.07 (configurable via Config).
- Owner wins if (direction=up and closePrice > openPrice) or (direction=down and closePrice < openPrice). Equality is push (refund).
- If accept side wins, each acceptor gets payout_i = (accept_i / sum_accept) * (P - fee).
- Price source: minute bars from Price Service. Use nearest minute at bet start and targetAt. If missing, use last known minute prior.
- Over free tier: creating or accepting beyond monthly limits triggers invoice in FZ/PT via Payhub.

**Idempotency**
- All create/accept actions require Idempotency-Key. Duplicate keys replay original result.

**Time and Locale**
- All user-visible times in GMT+7, stored as UTC.

---

## 5) Security and Anti‑Abuse
- **GST**: signed token with `userId`, `roomId`, `gameId`, and short expiry; verified by game backends.  
- **Server to server**: only game backends can call results endpoint; allow list and service JWT.  
- **Idempotency**: all POST routes require `Idempotency-Key`.  
- **Rate limits**: per user for join and accept; Redis sliding windows.  
- **Validation**: Zod DTOs; currency allow list and integer amounts.  
- **Fairness**: price sources via Price Service with signed reports.  
- **Audit**: every state transition and settlement recorded.  
- **Abuse**: deny self matching in CFB and excessive self acceptance; optional owner cap per bet.

---

## 6) Scalability and Reliability
- Stateless API with Redis for pools and locks; horizontal scale.  
- Matchmaking uses Redis sorted sets and streams to avoid hot spots.  
- MongoDB for match and bet records with time based partitioning.  
- Workers schedule settle jobs for CFB and retry on degraded oracle.  
- SLOs: p95 < 120 ms for read and simple write paths.

---

## 7) Observability
- **Tracing**: spans include `roomId`, `betId`, `holdId`, `requestId`.  
- **Metrics**: matches created, queue time, accept rate, settlement latency, push rate.  
- **Logs**: structured logs with redaction; no GST printed.  
- **Alerts**: spike in settlement failures, high push rate, queue backlog.

---

## 8) User Stories and Feature List
### Feature List
- PvP matchmaking and room orchestration with GST mint and redirect.  
- Trusted results intake with Payhub settlement.  
- CFB v1 owner and accept flows, settlement with rake, and push handling.  
- Operator views via Admin and audit exports.

### User Stories
- *As a player*, I enter a stake and get matched to play a game fairly.  
- *As a game backend*, I receive a room create call and later report results securely.  
- *As a bettor*, I open a price bet and others accept; at target time the platform settles fairly using signed prices.  
- *As staff*, I can inspect rooms and bets and resolve issues via Admin.

---

## 9) Risks and Mitigations
- **Oracle degradation**: mark as push or delay via Workers; log report ids.  
- **Race conditions**: locks with Redis and idempotency keys; settlement checks.  
- **Version skew**: DTO versioning and capability flags.  
- **Fraud**: deny self dealing and rapid open close loops; rate limits and heuristics.

---

## 10) Compatibility Notes
- Verifies Identity session tokens and uses JWKS.  
- Calls Payhub for holds and settlements; never mutates balances directly.  
- Reads prices from Price Service only; no direct provider calls.  
- Works with WebApp, Admin, Workers, and the **game service template** for partners.
