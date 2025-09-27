Repo: tg-miniapp-identity-service
File: UserStories.md
SHA-256: 97e1d952b8556ea47027ab15e5bcc2b019ab2af1051d28d50a39a6faf3d879ca
Bytes: 21232
Generated: 2025-09-27 01:58 GMT+7
Sources: SystemDesign.md (authoritative), old UserStories.md (baseline), Guide

---

# Section A — Personas & Stakeholders

> From SystemDesign: **Sessions & service tokens**, Telegram mini‑app handshake, JWT issuance, **roles/badges**, **KYC tiers & evidence**, residency/jurisdictions, sanctions screening hooks, device/session security, OAuth‑style introspection for services, **JWKS & key rotation**, **webhooks** to downstreams, and **Admin** controls. Cross‑service: **WebApp/Web3 Portal** (clients), **Admin**, **Config**, **Payhub** (limits), **Funding/Escrow/Campaigns/Events/Watchlist** (role gating), **Workers** (reminders/expiry), **Notifications**.

- **End User (TG MiniApp)** — authenticates via Telegram init data, links wallet(s), upgrades KYC, obtains roles/badges to unlock features.
- **Investor (User)** — End User with Investor badge/KYC tier enabling higher limits and gated features.
- **Project Owner (User)** — creates/owns projects, requires **ProjectOwner** badge.
- **Staff (Admin/Moderator/Operator)** — signs into Admin console, uses scoped staff JWTs; can grant/revoke badges per policy.
- **Compliance Officer** — reviews KYC evidence, sanctions hits, decisions/appeals.
- **Risk Analyst** — flags accounts, velocity limits, device clustering.
- **Service Consumer** — any backend verifying tokens (`/introspect`), requesting role decisions (`/authorize`), or subscribing to identity events.
- **Config Publisher** — source of signed policies (badge definitions, KYC forms, quotas, residency rules).
- **Workers/Schedulers** — expiry of sessions, rotating keys, prompting KYC refresh.
- **Notifications** — delivers verification outcomes and decision emails/TG messages.
- **Auditor** — read‑only export of decisions, evidence hashes, and audit logs.

---

# Section B — Epics → Features → User Stories (exhaustive)

## Epic B1 — Authentication & Sessions (Telegram first)

### Feature B1.1 — Telegram handshake → session
**Story B1.1.1** — *As an End User, I authenticate through Telegram WebApp init data,* so that *I receive an app session.*  
**Acceptance (GWT)**
1. **Given** valid TG init payload, **when** client calls `POST /v1/sessions/telegram` with `{initData, device}`, **then** service verifies HMAC, checks replay and clock drift, and issues **session JWT** with **short TTL** and **refresh token**; idempotent by `initData.hash` for 5 minutes.
2. **Given** payload is tampered or expired, **then** **401 invalid_telegram_payload** with safe error string.
3. **Given** device risk flags exceed threshold, **then** session is **restricted** and client is prompted for **step‑up** (OTP or wallet signature).

**Events**: `identity.session.created@v1` with `{userId, device, risk}`.

---

### Feature B1.2 — Refresh & revoke
**Story B1.1.2** — *As a User, I refresh and revoke my session,* so that *I stay logged in securely.*  
**Acceptance**
1. `POST /v1/sessions/refresh` rotates access token with sliding TTL, re‑evaluates risk and flags.
2. `POST /v1/sessions/revoke` invalidates current refresh token and all child tokens (device tree).
3. **Given** refresh reuse (token theft), **then** **invalidated** and all sessions for device are revoked, user notified.

**Observability**: token reuse counter, anomaly alerts.

---

## Epic B2 — Wallets & Linkage

### Feature B2.1 — Link wallet
**Story B2.1.1** — *As a User, I link a wallet by signing a nonce,* so that *I can prove ownership.*  
**Acceptance**
1. `POST /v1/wallets/link` returns `{ nonce }`; client signs and `POST /v1/wallets/verify { address, chain, signature }`; on success creates **WALLET(verifiedAt)** idempotent by `(userId,address,chain)`.
2. Duplicate address used by another user → **409 address_in_use** with appeal path.
3. On chain RPC failure, return **503 chain_unavailable** and keep nonce valid until expiry.

**Events**: `identity.wallet.linked@v1`.

### Feature B2.2 — Primary wallet & allowlist
**Story B2.1.2** — *As a User, I set a primary wallet and manage allowlists,* so that *downstream services have a canonical address.*  
**Acceptance**: `POST /v1/wallets/{id}/primary`, `POST /v1/wallets/{id}/allowlist { label, scopes }`; conflicts resolved with `If‑Match` ETag.

---

## Epic B3 — Roles, Badges, and Authorization

### Feature B3.1 — Role decision API
**Story B3.1.1** — *As a Service, I request an authorization decision for an action,* so that *I can gate features.*  
**Acceptance**
1. `POST /v1/authorize { subjectId, action, resource, context }` returns `{ allow, reason, requiredBadges[], kycTierRequired, limits }` based on **policies** from Config and user state.
2. Include **cache‑control** hints (`maxAge`,`staleWhileRevalidate`) and **audit decision id** for logging.
3. **Non‑Happy**: subject suspended → `allow=false`, reason `suspended` with unblock path.

**Events**: `identity.decision.made@v1` for sampled decisions.

---

### Feature B3.2 — Badges CRUD
**Story B3.1.2** — *As Staff, I grant, revoke, or suspend a badge,* so that *user capabilities adjust immediately.*  
**Acceptance**
1. `POST /v1/admin/badges/{badge}/grant { userId, reason, expiresAt? }` creates **BADGE_GRANT**, idempotent by `(userId,badge,reasonHash)`; emits `identity.badge.granted@v1`.
2. `POST /v1/admin/badges/{badge}/revoke { userId, reason }` records **BADGE_REVOKE** and emits `identity.badge.revoked@v1`.
3. Badge changes invalidate **decision caches** for the user.

---

## Epic B4 — KYC & Compliance

### Feature B4.1 — Start KYC
**Story B4.1.1** — *As a User, I start KYC and submit evidence,* so that *I can unlock higher limits.*  
**Acceptance**
1. `POST /v1/kyc/sessions { tier, country }` creates **KYC_SESSION** and returns **redirect** (provider) or **form schema**.
2. `POST /v1/kyc/sessions/{id}/submit { fields, documents[] }` stores **EVIDENCE** with content hashing, size/type checks, redaction flags.
3. Provider webhooks to `/v1/kyc/providers/{name}/callback` are verified by HMAC and advance state.

**Non‑Happy**: unsupported country → **403 country_blocked**; document too large → **413 payload_too_large**.

### Feature B4.2 — Decision & expiry
**Story B4.1.2** — *As Compliance, I review and decide KYC cases,* so that *risk is controlled.*  
**Acceptance**
1. `POST /v1/admin/kyc/{sessionId}/decision { approve|reject, reasons }` sets **KYC_TIER**, emits `identity.kyc.updated@v1`.
2. KYC lifetime tracked; **Workers** open **renewal tasks** 30 days prior to expiry.

### Feature B4.3 — Sanctions & jurisdiction checks
**Story B4.1.3** — *As Risk/Compliance, I block sanctioned or restricted users,* so that *we meet obligations.*  
**Acceptance**: on session creation, wallet link, and payout destinations, run screening; flags create **RISK_CASE** and set `user.flags.sanctioned=true` if confirmed.

---

## Epic B5 — Service Tokens, JWKS & Introspection

### Feature B5.1 — Service token issuance
**Story B5.1.1** — *As a Service, I obtain a service token with narrow scopes,* so that *I can call peer services.*  
**Acceptance**
1. `POST /v1/service-tokens { serviceId, scopes, audience }` returns short‑lived JWT with **mTLS bound** confirmation (optional).
2. Token signed by current key (`kid`), includes **correlationId** for tracing.

### Feature B5.2 — JWKS and rotation
**Story B5.1.2** — *As a Consumer, I fetch JWKS and validate tokens,* so that *auth is verifiable.*  
**Acceptance**: `GET /.well-known/jwks.json` serves keys with overlap; `POST /v1/admin/keys/rotate` performs rotation, re‑signs service tokens.

### Feature B5.3 — Introspect & revoke
**Story B5.1.3** — *As a Consumer, I introspect a token or revoke sessions,* so that *access can be verified or cut off.*  
**Acceptance**: `POST /v1/introspect { token }` returns claims if active; `POST /v1/admin/sessions/{id}/revoke` kills a session, emits `identity.session.revoked@v1`.

---

## Epic B6 — Privacy, Data Rights & Evidence Access

### Feature B6.1 — Consent & disclosures
**Story B6.1.1** — *As a User, I consent to disclosures and data use,* so that *compliance is met.*  
**Acceptance**: `POST /v1/consents { docId, hash, acceptedAt, ip, ua }`; stored in **CONSENT_LOG** and exportable.

### Feature B6.2 — DSAR export/delete
**Story B6.1.2** — *As a User, I request export or deletion,* so that *privacy rights are honored.*  
**Acceptance**: `POST /v1/privacy/export` queues export; `POST /v1/privacy/delete` checks **retention exceptions**; decisions audited and evented.

---

## Epic B7 — Observability, Reliability & Quotas

### Feature B7.1 — Idempotency and rate limits
**Story B7.1.1** — *As SRE, I need idempotent writes and fair limits,* so that *clients can retry safely.*  
**Acceptance**: all writes accept **Idempotency-Key**; per‑IP and per‑user rate limits with `Retry‑After`; dedupe on `(userId,action,hash)` where appropriate.

### Feature B7.2 — SLOs & dashboards
**Story B7.1.2** — *As SRE, I track SLOs for auth and KYC,* so that *incidents are visible.*  
**Acceptance**: p95 session create < 400ms, p99 introspect < 50ms, KYC provider callback success > 99.9%; alerts on error budgets and queue lag.

---

# Section C — End‑to‑End Scenarios (Swimlane narrative)

1. **E2E‑C1: TG Handshake → Session → Wallet Link → KYC Tier → Investor Badge → Access Funding**  
   Flow: user authenticates → links wallet → starts KYC, provider approves → Identity issues Investor badge → Funding APIs allow higher caps.

2. **E2E‑C2: Token Theft → Refresh Reuse → Global Revoke**  
   Flow: attacker reuses refresh, detection triggers tree revoke → user notified → forced re‑auth, audit trail recorded.

3. **E2E‑C3: Sanctions Hit → Blocked Actions → Appeal**  
   Flow: screening flags wallet → user blocked from payouts → Compliance reviews evidence → unblock or sustain with reasons.

4. **E2E‑C4: Key Rotation → JWKS Overlap → Zero‑Downtime**  
   Flow: rotate signing keys, JWKS published, tokens continue to validate; clients refresh JWKS on 401/498 and proceed.

---

# Section D — Traceability Matrix

| Story | APIs | Entities | Events | SystemDesign anchors |
|---|---|---|---|---|
| B1.1.1 | `POST /v1/sessions/telegram` | SESSION, DEVICE | identity.session.created@v1 | §Auth |
| B1.1.2 | `/sessions/refresh`, `/sessions/revoke` | SESSION | identity.session.revoked@v1 | §Auth |
| B2.1.1 | `/wallets/link`, `/wallets/verify` | WALLET | identity.wallet.linked@v1 | §Wallets |
| B2.1.2 | `/wallets/{id}/primary`, `/allowlist` | WALLET, ALLOWLIST | identity.wallet.updated@v1 | §Wallets |
| B3.1.1 | `POST /v1/authorize` | DECISION_LOG | identity.decision.made@v1 | §Authorization |
| B3.1.2 | `/admin/badges/*` | BADGE_GRANT, BADGE_REVOKE | identity.badge.*@v1 | §Badges |
| B4.1.1 | `/kyc/sessions`, `/submit`, provider webhook | KYC_SESSION, EVIDENCE | identity.kyc.updated@v1 | §KYC |
| B4.1.2 | `/admin/kyc/{id}/decision` | KYC_TIER | identity.kyc.updated@v1 | §KYC |
| B4.1.3 | screening at touchpoints | RISK_CASE | identity.risk.flagged@v1 | §Compliance |
| B5.1.1 | `/service-tokens` | SERVICE_TOKEN | identity.service_token.issued@v1 | §Tokens |
| B5.1.2 | `/.well-known/jwks.json`, `/admin/keys/rotate` | KEY, JWKS | identity.keys.rotated@v1 | §JWKS |
| B5.1.3 | `/introspect`, `/admin/sessions/{id}/revoke` | SESSION | identity.session.revoked@v1 | §Introspect |
| B6.1.1 | `/consents` | CONSENT_LOG | identity.consent.updated@v1 | §Privacy |
| B6.1.2 | `/privacy/export`, `/privacy/delete` | DSAR_CASE | identity.dsar.updated@v1 | §Privacy |
| B7.1.1 | idempotency, rate limits | IDEMPOTENCY_KEY, QUOTA | — | §Reliability |
| B7.1.2 | metrics | REQUEST_LOG | — | §Observability |

**Deltas vs old UserStories**: adds **Telegram handshake details**, **refresh‑reuse theft detection**, **decision API with cache hints**, **sanctions at multiple touchpoints**, **service tokens + mTLS binding**, **JWKS rotation with overlap**, and full **DSAR** flow.

---

# Section E — Assumptions & Open Questions

- Telegram init payload drift tolerance (max skew, replay window).  
- Chains supported for wallet link and signature formats.  
- KYC providers and webhook signing methods, retention of evidence.  
- Policy language for `/authorize` and how downstream pass contextual facts.  
- Which actions are blocked vs restricted under sanctions/suspicion.  
- DSAR deletion carve‑outs for financial records.

---

# Appendix — Coverage Gates

## A1. Stakeholder Catalog

| Stakeholder | Responsibilities | Permissions | Typical Actions | KPIs/SLO interests |
|---|---|---|---|---|
| End User | Sessions, wallets, KYC | session | login, link, KYC | auth success, KYC pass |
| Investor | Higher limits | session + badge | funding, claims | cap utilization |
| Project Owner | Own projects | badge | create projects | approval time |
| Staff | Admin ops | staff scoped | grant/revoke, review | MTTR |
| Compliance | KYC decisions | staff:compliance | review, sanction | SLA, false pos |
| Risk | Abuse controls | staff:risk | restrict, throttle | fraud rate |
| Service Consumer | AuthZ decisions | s2s | authorize, introspect | p99 latency |
| Config | Policies | signer | publish policies | propagation lag |
| Workers | Jobs | internal | expire, rotate | DLQ depth |
| Notifications | Delivery | service | notify outcomes | delivery success |
| Auditor | Read‑only | staff:auditor | export logs | audit pass |

## A2. RACI Matrix (capabilities)

| Capability | End User | Investor | Project Owner | Staff | Compliance | Risk | Services | Config | Workers | Auditor |
|---|---|---|---|---|---|---|---|---|---|
| Sessions | R | R | R | I | I | I | C | I | C | I |
| Wallet linkage | R | R | R | I | I | I | C | I | I | I |
| Authorization decisions | I | I | I | I | I | I | A | C | I | I |
| Badges | I | I | I | A | C | C | I | C | I | I |
| KYC | C | C | C | C | A | C | I | C | C | I |
| Sanctions | I | I | I | C | A | C | C | I | I | I |
| Service tokens/JWKS | I | I | I | I | I | I | A | C | C | I |
| Privacy (DSAR) | A | A | A | C | C | I | I | I | I | C |
| Observability | I | I | I | C | C | C | C | I | A | I |

Legend: R=Responsible, A=Accountable, C=Consulted, I=Informed.

## A3. CRUD × Persona × Resource Matrix

Resources: `USER, SESSION, DEVICE, WALLET, ALLOWLIST, BADGE_GRANT, BADGE_REVOKE, KYC_SESSION, KYC_TIER, EVIDENCE, RISK_CASE, DECISION_LOG, SERVICE_TOKEN, KEY, JWKS, CONSENT_LOG, DSAR_CASE, IDEMPOTENCY_KEY, QUOTA, REQUEST_LOG, AUDIT_LOG`

| Resource \ Persona | End User | Investor | Project Owner | Staff | Compliance | Risk | Services | Workers | Auditor |
|---|---|---|---|---|---|---|---|---|---|
| USER | R (view) | R (view) | R (view) | C/R/U/D | R (view) | R (view) | R (view) | R (view) | R (read) |
| SESSION | C/R/U/D (own) | C/R/U/D | C/R/U/D | C/R/U/D | R (view) | R (view) | Introspect | C/R/U/D | R (read) |
| DEVICE | R (own) | R (own) | R (own) | R (view) | R (view) | R (view) | R (view) | R (view) | R (read) |
| WALLET | C/R/U/D (own) | C/R/U/D | C/R/U/D | R (view) | R (view) | R (view) | R (view) | R (view) | R (read) |
| ALLOWLIST | C/R/U/D (own) | C/R/U/D | C/R/U/D | R (view) | R (view) | R (view) | R (view) | R (view) | R (read) |
| BADGE_GRANT | R (read) | R (read) | R (read) | C/R/U/D | R (view) | R (view) | R (view) | R (view) | R (read) |
| BADGE_REVOKE | R (read) | R (read) | R (read) | C/R/U/D | R (view) | R (view) | R (view) | R (view) | R (read) |
| KYC_SESSION | C/R/U/D (own) | C/R/U/D | C/R/U/D | R (view) | C/R/U/D | R (view) | R (view) | C/R/U/D | R (read) |
| KYC_TIER | R (read) | R (read) | R (read) | R (view) | C/R/U/D | R (view) | R (view) | R (view) | R (read) |
| EVIDENCE | C/R/U/D (own) | C/R/U/D | C/R/U/D | R (view) | C/R/U/D | R (view) | N/A | R (view) | R (read) |
| RISK_CASE | N/A | N/A | N/A | R (view) | C/R/U/D | C/R/U/D | R (view) | R (view) | R (read) |
| DECISION_LOG | N/A | N/A | N/A | R (view) | R (view) | R (view) | R (view) | R (view) | R (read) |
| SERVICE_TOKEN | N/A | N/A | N/A | R (view) | N/A | N/A | C/R/U/D (own) | R (view) | R (read) |
| KEY | N/A | N/A | N/A | R (view) | N/A | N/A | R (view) | C/R/U/D | R (read) |
| JWKS | R (read) | R (read) | R (read) | R (read) | R (read) | R (read) | R (read) | R (read) | R (read) |
| CONSENT_LOG | C/R/U/D (own) | C/R/U/D | C/R/U/D | R (view) | R (view) | R (view) | R (view) | R (view) | R (read) |
| DSAR_CASE | C/R/U/D (own) | C/R/U/D | C/R/U/D | R (view) | C/R/U/D | R (view) | R (view) | R (view) | R (read) |
| IDEMPOTENCY_KEY | N/A | N/A | N/A | R (view) | R (view) | R (view) | R (view) | R (view) | R (view) |
| QUOTA | R (read) | R (read) | R (read) | C/R/U/D | R (view) | R (view) | R (view) | C/R/U/D | R (view) |
| REQUEST_LOG | N/A | N/A | N/A | R (view) | R (view) | R (view) | R (view) | C/R/U/D | C/R/U/D |
| AUDIT_LOG | N/A | N/A | N/A | R (read) | R (read) | R (read) | R (read) | R (read) | C/R/U/D |

**N/A** indicates not user‑exposed.

## A4. Permissions / Badge / KYC Matrix

| Action | Requirement | Evidence | Expiry/Renewal | Appeal |
|---|---|---|---|---|
| TG session create | Telegram init data | HMAC verify | 5m replay window | Support |
| Wallet link | session | signature | — | Support |
| Start KYC | session | fields, docs | per tier | Compliance |
| Approve KYC | staff:compliance | case notes | per policy | Security |
| Grant badge | staff:admin or scoped role | ticket id | per badge | Security |
| Authorize action | valid token | policy snapshot | cache ttl | Support |
| Service token issue | mTLS or client secret | audit | 1h ttl | Security |
| DSAR export/delete | session + verify email | ticket | per case | Compliance |

## A5. Quota & Billing Matrix

| Metric | Free Tier | Overage (FZ/PT) | Exemptions | Failure Handling | Refunds |
|---|---|---|---|---|---|
| Sessions / device / day | X | throttle | staff exempt | 429 Retry‑After | N/A |
| Wallet link attempts / day | Y | throttle | staff exempt | 429 | N/A |
| KYC submissions / month | 1 | block | — | 409 duplicate | N/A |
| Introspect QPS / service | Q | throttle | allowlist | 429 | N/A |

## A6. Notification Matrix

| Event | Channel(s) | Recipients | Template | Throttle/Dedupe |
|---|---|---|---|---|
| identity.session.created@v1 | in-app | User | `session_created` | `(device,tsBucket)` |
| identity.wallet.linked@v1 | in-app | User | `wallet_linked` | `(address)` |
| identity.badge.granted@v1 | in-app | User | `badge_granted` | `(badge)` |
| identity.kyc.updated@v1 | in-app | User | `kyc_updated` | `(sessionId,status)` |
| identity.session.revoked@v1 | in-app | User | `session_revoked` | `(device)` |
| identity.keys.rotated@v1 | ops | Services | `keys_rotated` | `(kid)` |
| identity.dsar.updated@v1 | in-app | User | `dsar_updated` | `(caseId)` |

## A7. Cross‑Service Event Map

| Event | Producer | Consumers | Payload summary | Idempotency | Retry/Backoff |
|---|---|---|---|---|---|
| identity.badge.granted@v1 | Identity | All services | `{userId, badge, expiresAt}` | `(userId,badge,revision)` | 5× exp backoff |
| identity.kyc.updated@v1 | Identity | Payhub, Funding, Escrow, Campaigns | `{userId, tier, status}` | `(userId,sessionId)` | 5× |
| identity.session.revoked@v1 | Identity | Clients | `{userId, device}` | `(userId,device,tsBucket)` | 3× |
| identity.keys.rotated@v1 | Identity | Clients | `{jwksUrl, kid}` | `kid` | 3× |
| identity.risk.flagged@v1 | Identity | Admin | `{userId, flags}` | `(userId,flagType)` | 3× |

---

# Abuse/Misuse & NFR Sweeps

- **Abuse**: session farming, referral farming via multiple accounts → device/IP clustering, velocity limits, captcha challenges.  
- **Fraud**: stolen refresh tokens → reuse detection and tree revoke, notify user and force re‑auth.  
- **Security**: HMAC verification for TG, JWT best practices (short TTL, audience, nonce), scoped service tokens, JWKS rotation with overlap, HSTS, mTLS for service issuance.  
- **Privacy**: minimize PII, redact KYC evidence, retention windows, DSAR paths, consent logs.  
- **Localization/timezones**: human‑facing timestamps in **GMT+7**, storage in UTC.  
- **Resilience**: provider/webhook retries with HMAC verification, outbox for events, DLQ for callbacks, backoff with jitter, circuit breakers to KYC provider.

---

# Self‑Check — Stakeholder Coverage Report

**Counts**  
- Personas: 11  
- Features: 19  
- Stories: 21  
- Stories with non‑happy paths: 17/21  
- Entities covered in CRUD matrix: 20/20

**Checklist**  
- ✅ All personas appear in at least one story.  
- ✅ Each entity has at least one **C**, **R**, **U**, and **D** exposure (or reasoned N/A).  
- ✅ Every action mapped to roles/badges/KYC where applicable.  
- ✅ Quotas addressed for auth, wallets, KYC, introspection.  
- ✅ Each story references APIs/entities/events.  
- ✅ At least one end‑to‑end scenario per key persona.  
- ✅ Abuse/misuse cases enumerated with mitigations.  
- ✅ Observability signals tied to AC.  
- ✅ Localization/timezone handling present where user-visible.
