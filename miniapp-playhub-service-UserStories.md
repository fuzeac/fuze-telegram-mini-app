# Miniapp Playhub Service User Stories


---

# Section A — Personas & Stakeholders

> Discovered from architecture, APIs, data ownership, jobs, and cross-service contracts. Full catalog in **Appendix A1**.

- **Player** — Telegram/Web3 user who joins matchmaking and plays **CFB v1** and future partner games. Cares about fairness, speed, payouts.
- **Investor Player** — Player holding **Investor badge/KYC** enabling higher stakes and limits.
- **Room Host (System)** — Playhub allocates rooms, enforces timeouts, starts and finalizes games.
- **Partner Game Service** — Implemented from `tg-miniapp-game-service-template`, plugs into rooms via webhook/API for custom logic.
- **Payhub** — Downstream custody for **holds**, **transfers**, **refunds**, **settlements**, and **ledger**.
- **Identity Service** — Auth, **badges/KYC** for stake limits and gating.
- **Config Publisher** — Provides **signed configs** for stake ladders, fees, timeouts, RNG policies.
- **Admin/Operator** — Uses Admin console for bans, refunds, configuration, manual resolution.
- **Anti‑Cheat / Risk** — Reviews anomalies, collusion, botting; manages suspensions.
- **Support/CS** — Handles disputes and user communications.
- **SRE/Infra** — Observability, scaling, DLQ drains, incident response.
- **Workers/Schedulers** — Background executors for timeouts, expirations, payouts, retries.
- **Events Bus / Webhook Consumers** — WebApp/Portal that render room state and results.

---

# Section B — Epics → Features → User Stories (exhaustive)

## Epic B1 — Matchmaking & Rooms

### Feature B1.1 — Join matchmaking queue
**Story B1.1.1** — *As a Player, I want to join matchmaking with a stake,* so that *I can be matched fairly and quickly.*  
**Acceptance (GWT)**
1. **Given** I am authenticated, **when** I POST `/v1/matchmaking/join { gameId, betAmount, currency }` with an **Idempotency-Key**, **then** Playhub validates stake ladder and currency, checks **badge/limit** with Identity, and **places a hold** via Payhub.
2. **Given** the hold succeeds, **when** a compatible opponent is available, **then** Playhub creates a **room** and returns `{roomId, state="waiting"}` or **redirects** to an existing pending room.
3. **Given** matchmaking timeout elapses, **when** no opponent is found, **then** the hold is **released** and I receive **204 no content** with `reason=timeout`.
4. **Given** stake is below minimum or above my limit, **when** I join, **then** I receive **403 stake_not_allowed** with limit details.
5. **Given** my balance is insufficient, **when** I join, **then** I receive **402 insufficient_funds** and the response includes a top‑up hint.

**Non‑Happy Paths & Errors**
- Duplicate request within window → **200** with the **same** queue ticket (idempotent).  
- Payhub hold fails transiently → **503 try_again**, client retries with backoff; Playhub worker retries **3×** with jitter.  
- User already in active room → **409 already_in_room** returning `roomId`.  
- Identity unreachable → **502 identity_unavailable**, queue join denied.

**Permissions/Badges/KYC**: higher stakes require **Investor**; minors stake tier blocked if policy.  
**Data Touchpoints**: `QUEUE_TICKET`, `ROOM`, `ROOM_PLAYER`, `HOLD`, `MATCH_PARAMS`.  
**Cross‑Service**: Payhub `hold.create`, Identity `limits.get`, Config `stakeLadder.get`.  
**State & Lifecycle**: `ticket.created → matched|expired`, `room.created → ready → live → settled|aborted`.  
**Observability**: queue depth, time‑to‑match p95, hold success rate.  
**Notifications**: `playhub.matchmaking.state.changed@v1` to clients.

---

### Feature B1.2 — Leave queue / cancel
**Story B1.2.1** — *As a Player, I want to cancel matchmaking,* so that *I can exit without losing funds.*  
**Acceptance**
1. `DELETE /v1/matchmaking/tickets/{id}` cancels if not matched, **releases hold**, emits `matchmaking.canceled`.
2. If already matched, API returns **409 matched**, with `{roomId}` for redirect.

**Non‑Happy**: race with match creation → cancel returns **202 pending_release**, worker finalizes release.

---

### Feature B1.3 — Room creation and readiness
**Story B1.3.1** — *As Playhub, I want to create a room with both players and a fairness seed,* so that *the game can start safely.*  
**Acceptance**
1. When two tickets match, Playhub creates `ROOM` with `seed_commit` (keccak hash of secret), `rng_policy`, `fee_schedule`, `timeout_s`.  
2. Sends **room.ready** event to both players; Partner Game receives `room.webhook` with `roomId` and participants.  
3. If any player fails to confirm **ready** within `ready_ttl`, room is **aborted** and both holds **released**.

**Non‑Happy**: webhook delivery fails → **retry 5× exp backoff**, then mark `partner_unreachable` and auto‑fallback to standard CFB.

---

## Epic B2 — CFB v1 (Coin Flip Battle) — Core Game

### Feature B2.1 — Start and reveal
**Story B2.1.1** — *As a Player, I want a provably fair coin flip,* so that *I trust the outcome.*  
**Acceptance**
1. On start, Playhub publishes `seed_commit` and requires both players to **ready**.  
2. At flip, Playhub computes `seed_reveal` and verifies `keccak(seed_reveal) == seed_commit`.  
3. The outcome is `heads|tails` determined by `HMAC_SHA256(seed_reveal, roomId)` parity, recorded on `ROOM_RESULT` with audit trail.  
4. A **fee** per schedule is applied; **net settlement** computed.

**Non‑Happy**: mismatch commit/reveal → **internal alert**, room aborted, full refunds. RNG policy fallback if VRF unavailable.

---

### Feature B2.2 — Settlement
**Story B2.2.1** — *As a Player, I want timely settlement,* so that *winnings credit fast.*  
**Acceptance**
1. On finalization, worker calls Payhub `transfer.settle` for **winner**, `release.hold` for **loser remainder** minus fees.  
2. If Payhub down, job retries `N=8` times with exponential backoff; after `T=2h`, marks `settlement_delayed` and notifies players.  
3. Settlement idempotency: `settlement_id = roomId`, all outbound transfers use the same `Idempotency-Key`.

**Non‑Happy**: partial success (winner credited, loser release failed) → compensating job retries until reconciled; ledger DR checks nightly.

---

### Feature B2.3 — Dispute & admin override
**Story B2.3.1** — *As Support, I need to mark a room disputed and issue refunds or adjustments,* so that *we resolve user issues.*  
**Acceptance**
1. `POST /v1/admin/rooms/{roomId}/dispute` sets `status=disputed`, freezes payouts if pending.  
2. `POST /v1/admin/rooms/{roomId}/adjustments` creates ledger adjustments via Payhub with reason.  
3. All actions audit‑logged with actor and case id.

**Non‑Happy**: adjustment idempotency collision → **409** with reference to previous adjustment.

---

## Epic B3 — Partner Game Integration

### Feature B3.1 — Partner room lifecycle hooks
**Story B3.1.1** — *As a Partner Game, I want webhooks for room ready, start, and finalize,* so that *I can implement custom logic.*  
**Acceptance**
1. Playhub calls partner endpoints: `/rooms/{id}/ready`, `/start`, `/finalize` with **HMAC signature** and `x-event-id`.  
2. Partner responds within SLA `2s`. Timeouts lead to retries; after max retries, room switches to **fallback** mode.  
3. Partner returns **deterministic result** hash and evidence; Playhub verifies against policy.

**Non‑Happy**: signature invalid → **401** stop integration; SLAs breached → circuit open for 5m.

---

### Feature B3.2 — Payout splits and fees
**Story B3.2.1** — *As an Operator, I want to define partner revenue share and platform fees,* so that *payouts are correct.*  
**Acceptance**
1. Fee schedule and split percentages are in **signed config** with version.  
2. Settlement uses config version **captured at room creation**; later changes do not alter past rooms.

---

## Epic B4 — Risk, Anti‑Cheat, and Bans

### Feature B4.1 — Collusion and bot detection
**Story B4.1.1** — *As Risk, I want automated signals for collusion/botting,* so that *we can intervene.*  
**Acceptance**
1. Signals: repeated pairings, time‑to‑ready anomalies, stake pattern outliers, device/IP correlations.  
2. Rooms flagged are **quarantined** post‑settlement for **manual review**; payouts can be held temporarily.  
3. **Actions**: suspend user, void room, refund, or confiscate per policy.

**Non‑Happy**: false positives → appeal flow with Support; audit trails preserved.

---

### Feature B4.2 — Player and device bans
**Story B4.2.1** — *As an Admin, I can ban a player or device from matchmaking,* so that *abuse volume drops.*  
**Acceptance**: `POST /v1/admin/bans` with scope `player|device`, propagates to cache within 2s, returns ban id, emits `playhub.security.banned@v1`.

---

## Epic B5 — Operations, Reliability, and Observability

### Feature B5.1 — Timeouts and DLQs
**Story B5.1.1** — *As SRE, I want deterministic timeouts and DLQ handling,* so that *rooms never hang.*  
**Acceptance**
1. **Ready TTL**, **Start TTL**, **Finalize TTL** enforced by workers; overdue rooms move to `aborted|refunded` states.  
2. Jobs publish to **DLQ** on permanent failure with payload and last error; alert opens when DLQ depth > threshold.  
3. DLQ replay is **idempotent** by `(roomId, stage)`.

### Feature B5.2 — Health & SLOs
**Story B5.2.1** — *As SRE, I want SLOs for matchmaking and settlement,* so that *we detect regressions.*  
**Acceptance**:  
- Matchmaking time‑to‑match p95 < 30s during peak.  
- Settlement p95 < 60s when Payhub ≥ GREEN.  
- Error rate < 1% for holds and settlements, otherwise alert.

### Feature B5.3 — Metrics, logs, traces
**Story B5.3.1** — *As SRE, I want correlated telemetry,* so that *I can trace a room from join to payout.*  
**Acceptance**: `traceId` propagation across Identity, Payhub, Playhub; room events carry `eventId`, logs with `roomId`, metrics for queue depth, ready confirmations, RNG verifications, payout attempts.

---

## Epic B6 — Admin & Backoffice

### Feature B6.1 — Manual refund
**Story B6.1.1** — *As an Operator, I can issue a manual refund,* so that *I can resolve exceptional cases.*  
**Acceptance**: `POST /v1/admin/refunds` referencing `roomId` or `holdId`, idempotent by `refundId`.

### Feature B6.2 — Config rollouts
**Story B6.2.1** — *As an Operator, I can roll out signed configs,* so that *stake limits and fees change safely.*  
**Acceptance**: staged rollout with percentage and environment, quick **rollback** path; Playhub advertises `config_version` on events.

---

# Section C — End‑to‑End Scenarios (Swimlane narrative)

1. **E2E‑C1: Join Queue → Hold Placed → Matched → Room Ready → CFB Flip → Settlement**  
   *Pre*: Player authenticated, balance sufficient.  
   *Flow*: `/matchmaking/join` → Payhub hold → matched → room created with `seed_commit` → both ready → flip with `seed_reveal` → winner calculated → Payhub settle & release → result event → UI shows payout.  
   *Post*: Ledger consistent, room archived.

2. **E2E‑C2: Join Queue → Timeout → Refund**  
   *Pre*: Niche stake and low liquidity.  
   *Flow*: join → no opponent by TTL → worker releases hold → ticket expired → UI toast.  
   *Post*: Balance unchanged, ticket closed.

3. **E2E‑C3: Partner Game Unresponsive → Fallback → Settlement**  
   *Pre*: Integration outage.  
   *Flow*: room.ready webhook timeouts → retries → switch to fallback logic → finalize → settle.  
   *Post*: User informed, audit notes partner outage.

4. **E2E‑C4: Dispute → Admin Adjustment**  
   *Pre*: Player raises complaint.  
   *Flow*: mark disputed → freeze payouts if pending → create adjustment via Payhub → notify players.  
   *Post*: Case closed with audit trail.

---

# Section D — Traceability Matrix

| Story ID | APIs | Entities | Events | SystemDesign anchors |
|---|---|---|---|---|
| B1.1.1 | `POST /v1/matchmaking/join` | QUEUE_TICKET, HOLD | playhub.matchmaking.requested@v1 | §Interfaces, §Flows Matchmaking |
| B1.2.1 | `DELETE /v1/matchmaking/tickets/{id}` | HOLD | playhub.matchmaking.canceled@v1 | §Interfaces |
| B1.3.1 | internal create room | ROOM, ROOM_PLAYER | playhub.room.ready@v1 | §Data, §Flows Rooms |
| B2.1.1 | internal flip | ROOM_RESULT | playhub.cfb.result@v1 | §Rules RNG |
| B2.2.1 | Payhub settle | TRANSFER | playhub.settlement.succeeded@v1 | §Flows Settlement |
| B2.3.1 | `/v1/admin/rooms/{id}/dispute` | DISPUTE, ADJUSTMENT | playhub.room.disputed@v1 | §Admin |
| B3.1.1 | partner webhooks | ROOM_HOOK | playhub.partner.webhook.sent@v1 | §Interfaces Webhooks |
| B3.2.1 | config versioning | CONFIG_VERSION | — | §Configuration |
| B4.1.1 | risk signals | RISK_FLAG | playhub.risk.flagged@v1 | §Security |
| B4.2.1 | `/v1/admin/bans` | BAN | playhub.security.banned@v1 | §Security |
| B5.1.1 | workers | JOB, DLQ | playhub.job.failed@v1 | §Reliability |
| B5.2.1 | SLOs | — | — | §Observability |
| B5.3.1 | tracing | — | — | §Observability |
| B6.1.1 | `/v1/admin/refunds` | REFUND | playhub.refund.issued@v1 | §Admin |
| B6.2.1 | config rollouts | CONFIG | playhub.config.applied@v1 | §Configuration |

**Deltas vs old UserStories**: clarified **idempotency**, **seed commit/reveal**, **partner fallback**, **settlement retries**, **DLQ**, **risk flags**, **manual adjustments/refunds**, and **signed config version pinning**.

---

# Section E — Assumptions & Open Questions

- Exact **stake ladder** per currency and per badge tier.  
- **VRF** vs commit‑reveal policy and whether on‑chain proofs will be required later.  
- **Partner SLA** values and failure thresholds per environment.  
- **Dispute** SLA and refund policies.  
- Whether **SSE/WebSocket** will complement polling for room state to clients.

---

# Appendix — Coverage Gates

## A1. Stakeholder Catalog

| Stakeholder | Responsibilities | Permissions | Typical Actions | KPIs/SLO interests |
|---|---|---|---|---|
| Player | Join, play, receive payouts | session | join, ready, play | time‑to‑match, payout time |
| Investor Player | High stakes | badge KYC | join high tier | payout success |
| Room Host (System) | Lifecycle control | internal | create room, enforce TTL | stuck rooms |
| Partner Game Service | Custom game logic | webhook secret | respond to hooks | SLA latency |
| Payhub | Holds, settle, refunds | s2s token | hold, settle, refund | success rates |
| Identity | Auth, limits, badges | s2s token | introspect, limits | uptime |
| Config Publisher | Signed configs | signer keys | publish ladders, fees | rollout safety |
| Admin/Operator | Overrides, config, bans | staff badge | dispute, adjust, refund | MTTR |
| Anti‑Cheat / Risk | Detect abuse | staff badge | flag, suspend | false positive rate |
| Support/CS | Handle disputes | staff badge (read) | communicate, escalate | resolution time |
| SRE/Infra | Keep SLOs | ops access | scale, drain DLQ | error budget |
| Workers/Schedulers | Execute jobs | internal | timeouts, payouts | DLQ depth |
| Events/Webhooks | Notify clients | webhook secrets | deliver events | delivery success |

## A2. RACI Matrix (capabilities)

| Capability | Player | Investor | Partner | Admin | Risk | Support | Payhub | Identity | Config | SRE |
|---|---|---|---|---|---|---|---|---|---|---|
| Matchmaking | R | R | C | I | I | I | I | I | I | C |
| Room lifecycle | R | R | C | I | I | I | I | I | I | C |
| CFB result | R | R | C | I | I | I | I | I | I | I |
| Settlement | I | I | I | C | I | I | A | I | I | C |
| Partner hooks | I | I | R | I | I | I | I | I | I | C |
| Bans & risk | I | I | I | A | C | C | I | I | I | I |
| Refund/adjust | I | I | I | A | C | C | C | I | I | I |
| Config rollout | I | I | I | A | I | I | I | I | C | C |
| Observability | I | I | I | I | I | I | I | I | I | A |

Legend: R=Responsible, A=Accountable, C=Consulted, I=Informed.

## A3. CRUD × Persona × Resource Matrix

Resources: `QUEUE_TICKET, ROOM, ROOM_PLAYER, ROOM_RESULT, HOLD, TRANSFER, DISPUTE, ADJUSTMENT, RISK_FLAG, BAN, JOB, DLQ, CONFIG_VERSION, CONFIG, WEBHOOK_DELIVERY, PARTNER_ENDPOINT`

| Resource \ Persona | Player | Investor | Partner | Admin | Risk | Workers | SRE |
|---|---|---|---|---|---|---|
| QUEUE_TICKET | C/R/U/D | C/R/U/D | R (read own) | R (view) | R (view) | C/R/U/D | R (view) |
| ROOM | R (read) | R (read) | R (read/write hooks) | C/R/U/D | R (read) | C/R/U/D | R (read) |
| ROOM_PLAYER | R (read) | R (read) | R (read) | C/R/U/D | R (read) | C/R/U/D | R (read) |
| ROOM_RESULT | R (read) | R (read) | R (read) | C/R/U/D | C/R (flag) | C/R | R (read) |
| HOLD | R (read own) | R (read own) | N/A | R (view) | R (view) | C/R/U/D | R (view) |
| TRANSFER | R (read own) | R (read own) | N/A | R (view) | R (view) | C/R/U/D | R (view) |
| DISPUTE | R (open) | R (open) | N/A | C/R/U/D | C/R | C/R | R (read) |
| ADJUSTMENT | N/A | N/A | N/A | C/R/U/D | C | C/R | R (read) |
| RISK_FLAG | N/A | N/A | N/A | R (view) | C/R/U/D | C/R | R (read) |
| BAN | N/A | N/A | N/A | C/R/U/D | C | R (view) | R (view) |
| JOB | N/A | N/A | N/A | R (view) | R (view) | C/R/U/D | R (view) |
| DLQ | N/A | N/A | N/A | R (view) | R (view) | C/R/U/D | C/R/U/D |
| CONFIG_VERSION | N/A | N/A | R (read) | C/R/U/D | I | R (view) | R (view) |
| CONFIG | N/A | N/A | R (read) | C/R/U/D | I | R (view) | R (view) |
| WEBHOOK_DELIVERY | N/A | N/A | R (receive) | R (view) | I | C/R/U/D | R (view) |
| PARTNER_ENDPOINT | N/A | N/A | C/R/U/D | C/R/U/D | I | I | I |

**N/A** indicates intentional restriction.

## A4. Permissions / Badge / KYC Matrix

| Action | Requirement | Evidence | Expiry/Renewal | Appeal |
|---|---|---|---|---|
| Join stake ≤ Tier1 | Auth session | — | policy | — |
| Join stake Tier2+ | Investor badge | KYC evidence in Identity | 2 years | Support |
| Create partner endpoint | Operator | domain ownership | 1 year | Admin |
| Accept partner webhooks | Valid signature | shared secret | rotate 90d | Operator |
| Admin refund/adjust | Staff badge | case id | immediate | Security |
| Ban player/device | Staff badge | abuse evidence | until lifted | Support |

## A5. Quota & Billing Matrix

| Metric | Free Tier | Overage (FZ/PT) | Exemptions | Failure Handling | Refunds |
|---|---|---|---|---|---|
| Matchmaking joins / user / day | K | billed per extra join | events promo | 402 → invoice | promo credit |
| Active rooms / partner | N | billed per extra | vetted partners | 402 → invoice | prorata |
| Partner webhook QPS | Q | throttle beyond | internal partners | 429 with Retry‑After | N/A |
| Dispute submissions / user / month | D | N/A | none | 429 backoff | N/A |

## A6. Notification Matrix

| Event | Channels | Recipients | Template | Throttle/Dedupe |
|---|---|---|---|---|
| matchmaking.state.changed | webhook + in-app | Player | `matchmaking_state` | collapse by ticketId |
| room.ready | webhook + in-app | Player, Partner | `room_ready` | single |
| game.result | webhook + in-app | Player | `game_result` | single |
| settlement.delayed | in-app | Player | `settlement_delayed` | 1 per room |
| dispute.opened | in-app + ops | Support, Admin | `dispute_opened` | collapse by roomId |
| security.banned | ops | Risk, Admin | `security_banned` | collapse by actor |

## A7. Cross‑Service Event Map

| Event | Producer | Consumers | Payload summary | Idempotency | Retry/Backoff |
|---|---|---|---|---|---|
| playhub.matchmaking.requested@v1 | Playhub | WebApp | `{ticketId, userId, stake, currency}` | `ticketId` | 5× exp backoff |
| playhub.matchmaking.state.changed@v1 | Playhub | WebApp | `{ticketId, state}` | `ticketId,state_ts` | 5× |
| playhub.room.ready@v1 | Playhub | WebApp, Partner | `{roomId, participants, seedCommit}` | `roomId` | 5× |
| playhub.cfb.result@v1 | Playhub | WebApp | `{roomId, outcome, reveal, audit}` | `roomId` | 5× |
| playhub.settlement.succeeded@v1 | Playhub | WebApp, Payhub | `{roomId, transfers}` | `roomId` | 5× |
| playhub.settlement.failed@v1 | Playhub | Ops | `{roomId, error}` | `roomId,retry_no` | 5× |
| playhub.partner.webhook.sent@v1 | Playhub | Ops | `{roomId, endpoint, status}` | `deliveryId` | 5× |
| playhub.security.banned@v1 | Playhub | Admin | `{actor, scope, reason}` | `banId` | 3× |

---

# Abuse/Misuse & NFR Sweeps

- **Abuse**: queue/room **griefing** (join‑leave spam → cooldown per user), **collusion** (repeat pair patterns → risk flags), **botting** (unnatural timings → captcha step‑up), **result spoofing** (commit/reveal verify), **webhook replay** (HMAC + nonce + expiry), **front‑running** RNG (seed commit before reveal).  
- **Fraud**: payment chargebacks not applicable on ledger; manual adjustments documented; stake laundering via partners mitigated by allowlists and quotas.  
- **Security**: JWT verification, HMAC‑signed webhooks, idempotency, Payhub s2s tokens, config signature verification, strict timeouts.  
- **Privacy**: no PII stored beyond user ids; disputes redact chat logs; audit trails kept per retention.  
- **Localization/timezones**: timestamps UTC in events, display **GMT+7** in clients.  
- **Resilience**: retries with jitter, DLQ replay, partner circuit breakers, settlement compensations, idempotent room transitions.

---

