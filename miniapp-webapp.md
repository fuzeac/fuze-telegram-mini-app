# Miniapp Webapp
*Version:* v0.2.1  

> High‑level architectural blueprint for the **Telegram Mini App frontend**. This document covers UI architecture, integration contracts, client‑side state, data flows, security posture, and cross‑repo interoperability for MVP features: login, matchmaking and gameplay handoff, **CFB v1** bets, watchlists/feeds, campaigns, and events.

---

## 1) Architecture Diagram (HLD)
```mermaid
flowchart LR
  subgraph "Clients"
    TG["Telegram WebApp"]
  end

  subgraph "WebApp (this repo)"
    UI["SPA UI (React/Vite)"]
    BFF["BFF API (Fastify)"]
    SESS["Session Store (Redis)"]
    LC["Local Cache (IndexedDB)"]
    HealthAggregator["Health Aggregator"]
  end

  subgraph "Platform Services"
    IDN["Identity API"]
    PAY["Payhub API (for Stars & FZ)"]
    PRC["Price API"]
    WAT["Watchlist API"]
    PH["Playhub API"]
    CMP["Campaigns API"]
    FND["Funding API"]
    SEX["Star Exchange API"]
    EVT["Events API"]
    CFG["Config API"]
  end

  subgraph "External"
    TGPLAT["Telegram Platform"]
  end

  TG -->|"initData, UI assets"| UI
  UI -->|"REST/JSON"| BFF
  
  BFF -->|"verify session, badges"| IDN
  BFF -->|"process Star/FZ payments, get balances"| PAY
  BFF -->|"get price data"| PRC
  BFF -->|"manage watchlists"| WAT
  BFF -->|"join games"| PH
  BFF -->|"submit campaign proofs"| CMP
  BFF -->|"manage events"| EVT
  BFF -->|"get feature flags"| CFG
  
  %% Updated Service Interactions
  BFF -->|"subscribe to sales with Stars"| FND
  BFF -->|"execute Star-to-Crypto swaps, manage P2P offers"| SEX

  BFF --> SESS
  BFF --> LC
  BFF --> HealthAggregator
  TGPLAT -->|"launch params"| UI
```
*Notes:* WebApp calls **public endpoints** of domain services. Payhub is never called directly by the client. Any balance snapshots shown come from domain responses (e.g., game results or optional summary endpoints).

---

## 2) Technology Stack
| Layer | Choice | Rationale |
|---|---|---|
| Framework | **Next.js** (App Router) with React | Ideal for Telegram WebApp; SSR/ISR optional |
| Language | TypeScript | Type safety and shared DTOs |
| Styling | Tailwind CSS + shadcn/ui | Fast, consistent UI |
| State | React Query + Zustand | Network cache and local UI state |
| Auth | Telegram JS SDK initData → Identity JWT | Simple and secure |
| Transport | fetch with retry and backoff | Standard, minimal deps |
| Telemetry | Sentry (optional) + Web Vitals | Production visibility |
| Build/Deploy | Next build + Docker | CI friendly |

---

## 3) Responsibilities and Scope
**Owns**
- Telegram WebApp bootstrap, capturing **initData**, and login with Identity.  
- Shell layout, navigation, localization, and theme.  
- Pages for: matchmaking and result view; **CFB v1** create, accept, and detail; Watchlists/Feeds; Campaigns; Events.  
- Client‑side cache and optimistic UX where safe.

**Not owned**
- Business logic and settlement (PlayHub and Payhub).  
- Price oracles (Price Service).  
- Admin tooling (separate repo).

---

## 4) Page Map
- `/` — Home hub with tabs: Play, CFB, Watchlist, Campaigns, Events.  
- `/play` — Games list and matchmaking join.  
- `/play/result/:roomId` — Verified results screen.  
- `/cfb` — CFB list and my bets.  
- `/cfb/new` — Create owner bet.  
- `/cfb/:id` — Bet detail, accept flow, countdown.  
- `/watchlist` — My watchlists and aggregated feed.  
- `/campaigns` — Active campaigns and claims.  
- `/events` — Browse and detail pages.  

---

## 5) Data Design — Client State
```
session: (token, exp, userId)
profile: (id, username, languageCode, roles, settings, area)

matchmaking: (status, roomId?, gst?, redirectUrl?)
cfbComposer: (asset, currency, horizon, ownerStake, closeAcceptAt)
cfbBetView: (betId, status, ownerStake, acceptPool, acceptors, timers)
feedsCache: keyed by (tab, filter)

requestMeta: (requestId, retryCount, lastError?)
```
Persistence: session token in memory + `sessionStorage` fallback (`tg-miniapp-session`). No secrets in `localStorage`.

---

## 6) Data Flows

## 6.1 Authentication and badge gating prompt

```mermaid
sequenceDiagram
  autonumber
  participant TG as "Telegram App"
  participant UI as WebApp
  participant ID as Identity

  TG->>UI: Launch with initData
  UI->>ID: POST /v1/session/telegram { initData }
  ID-->>UI: 200 { jwt, profile, badges, meters }
  UI->>UI: Render dashboard, show gated CTAs if badges missing
```

## 6.2 Purchase with free tier, then overage payment if needed

```mermaid
sequenceDiagram
  autonumber
  participant UI as WebApp
  participant FUN as Funding
  participant PAY as Payhub

  UI->>FUN: POST /v1/sales/{{id}}/purchase (Idem-Key)
  FUN-->>UI: 202 { purchaseId, status: pending }
  UI->>PAY: GET /v1/invoices/{{purchaseId}}
  alt Over free limit
    PAY-->>UI: 402 Payment Required in FZ or PT
    UI->>PAY: POST /v1/invoices/{{purchaseId}}/pay { currency }
    PAY-->>UI: 200 Receipt
  else Within free tier
    PAY-->>UI: 204 No Content
  end
  UI->>FUN: GET /v1/purchases/{{id}}/status
  FUN-->>UI: 200 { status }
```

## 6.3 PlayHub matchmaking and CFB

```mermaid
sequenceDiagram
  autonumber
  participant UI as WebApp
  participant PH as PlayHub
  participant PAY as Payhub

  UI->>PH: POST /v1/matchmaking/join (Idem-Key)
  PH-->>UI: 202 Waiting
  UI->>PH: GET /v1/matches/{{id}}
  PH-->>UI: 200 { roomId, gst }
  Note over UI,PH: UI redirects to game with GST

  UI->>PH: POST /v1/cfb/bets (Idem-Key)
  PH-->>UI: 201 Created
  UI->>PH: POST /v1/cfb/bets/{{id}}/accept (Idem-Key)
  PH-->>UI: 201 Accepted
  UI->>PAY: GET /v1/invoices/{{id}}
  alt Overage due
    PAY-->>UI: 402 Payment Required
    UI->>PAY: POST /v1/invoices/{{id}}/pay { currency }
    PAY-->>UI: 200 Receipt
  else Covered by free tier
    PAY-->>UI: 204 No Content
  end
```


---

## 7) Security and Privacy
- **Token handling**: keep session token in memory with optional `sessionStorage` backup; never in URLs.  
- **Telegram verification**: client does not verify HMAC; server does.  
- **CSP**: strict policy; only API origins and allowed game hosts.  
- **XSS**: sanitize any user‑generated text; safe links for external sites.  
- **Clickjacking**: rely on Telegram WebView context; no external iframes except allowed game hosts.  
- **PII**: display names and usernames only; no precise location.  
- **Rate limit hints**: retries with backoff only on retryable errors; expose friendly error states.

---

## 8) Performance and Scalability
- React Query caching and background revalidation.  
- Static asset caching via immutable content hashes and CDN.  
- Route level code splitting and lazy loading for heavy pages.  
- Skeletons and optimistic updates where safe.  
- Optional service worker for read‑only caches (future).

---

## 9) Observability and Quality
- **Error tracking**: Sentry or similar; attach `requestId` to breadcrumbs.  
- **Metrics**: Web Vitals; custom events like `matchmaking_join_ok`, `cfb_accept_ok`, `result_fetch_time`.  
- **E2E**: Playwright tests in Telegram WebApp context; mock Identity and PlayHub.  
- **Accessibility**: semantic HTML, proper focus management, contrast.

---

## 10) Compatibility Notes
- Aligns with **Identity** token semantics and JWKS rotation; refresh on 401 then re‑login.  
- Matches **PlayHub** endpoints for matchmaking and **CFB v1**.  
- Updated to **Watchlist** service naming; uses `NEXT_PUBLIC_WATCHLIST_URL`.  
- Does not call **Payhub** directly; balances are shown only from domain responses.

---

## 11) User Stories and Feature List
### Feature List
- Telegram login and profile view.  
- Game selection, matchmaking join, gameplay handoff, and result view.  
- CFB v1: create, accept, browse, and result screens.  
- Watchlists and aggregated feeds.  
- Campaigns list and claim action.  
- Events browse and detail view.

### User Stories
- *As a new user*, I log in with Telegram and see a localized home screen.  
- *As a player*, I enter a stake and join a match, then be redirected to the game and later see my result.  
- *As a bettor*, I open a price bet and others accept with clear timers and payouts.  
- *As an explorer*, I browse token watchlists and news in one feed.  
- *As a participant*, I see campaigns and claim rewards when eligible.  
- *As an attendee*, I browse events and open external booking links.

---

## 12) Implementation Plan
- M1: Auth shell, layout, i18n, Identity integration.  
- M2: Matchmaking join and result flow, game handoff.  
- M3: **CFB v1** create and accept screens.  
- M4: Watchlist, Campaigns, Events tabs.  
- M5: Error tracking, web vitals, polish.

---

## 13) Risks and Mitigations
- **Token leakage** → never place tokens in URLs; only headers.  
- **Version skew with backends** → pin DTO versions and capability flags.  
- **WebView quirks** → feature detection for back navigation and viewport resize.

---

## 14) Future Work
- Push updates via SSE or WebSocket for results and feeds.  
- Partner theming and white‑labeling.  
- Offline cache for watchlists and event listings.
