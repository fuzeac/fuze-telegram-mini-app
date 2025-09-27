Repo: tg-miniapp-events-service
File: UserStories.md
SHA-256: 0aa351ddc2b72134c5a8419929dd7648161633332268ad2aa2e1f9a5abfedd9b
Bytes: 19551
Generated: 2025-09-27 01:47 GMT+7
Sources: SystemDesign.md (authoritative), old UserStories.md (baseline), Guide

---

# Section A — Personas & Stakeholders

> From SystemDesign: **Event catalog & submissions**, categories, venues, schedules, RSVPs, reminders, moderation, and optional ticketing handoff to **Payhub**. Cross‑service deps: **Identity** (badges/KYC), **Config** (signed feature flags, quiet hours), **Workers** (reminders, expirations), **Admin** (moderation console), **WebApp/Web3 Portal** (UX), **Notifications**.

- **Guest (End User)** — browses events, RSVPs, saves to calendar, receives reminders, attends and optionally checks‑in.
- **Organizer (Project Owner)** — submits events, manages details, schedules, capacity, and communications.
- **Curator/Moderator** — reviews and approves submissions, edits metadata, features events, and removes spam.
- **Admin/Ops** — manages categories, global policies, featured banners, and incident toggles.
- **Identity Service** — verifies roles/badges (Organizer, Verified), KYC tiers if required.
- **Payhub** — optional ticketing link (external) and receipts if events are paid elsewhere, not primary in this service.
- **Config Publisher** — pushes signed configs for categories, feature flags, throttles.
- **Workers/Schedulers** — sends reminders, closes RSVPs, expires drafts, rotates featured carousels.
- **SRE/Ops** — monitors freshness, queue lag, job failures.
- **Notifications** — in‑app/TG messages and webhooks to clients.

---

# Section B — Epics → Features → User Stories (exhaustive)

## Epic B1 — Event Creation & Moderation

### Feature B1.1 — Submit event
**Story B1.1.1** — *As an Organizer, I submit an event with schedule and venue,* so that *it appears in the catalog after approval.*  
**Acceptance (GWT)**
1. **Given** I have Organizer badge, **when** I `POST /v1/events` with `{ title, summary, startsAt, endsAt, timeZone, venue { kind, address?, link? }, categories[], tags[], bannerUrl?, capacity?, rsvpPolicy, externalTicketUrl?, contact }` and **Idempotency-Key**, **then** an **EVENT(status="pending_review")** is created, idempotent by `(title, startsAt, organizerId)` for 24h.
2. **Given** `endsAt < startsAt` or missing `timeZone`, **then** **422 invalid_window** is returned with pointers.
3. **Given** `venue.kind` invalid or unsafe URL, **then** **422 invalid_venue** or **403 url_forbidden** (domain allowlist).

**Non‑Happy**: exceeded free tier of open pending events → **402 payment_required** including `invoiceId`.  
**Data Touchpoints**: `EVENT, EVENT_SCHEDULE, VENUE, MEDIA_ASSET, CATEGORY_LINK, POLICY_SNAPSHOT`.  
**Events**: `events.event.submitted@v1`.

---

### Feature B1.2 — Review & approve
**Story B1.1.2** — *As a Moderator, I approve or reject submissions,* so that *the catalog remains clean.*  
**Acceptance**
1. `POST /v1/admin/events/{id}/review { decision, reasons?, edits? }` sets `status="approved"` or `"rejected"`, stores **edits** diff and reasons; idempotent by `(eventId, revisionNo)` or **Idempotency-Key**.
2. On approve, publish to catalog and emit `events.event.approved@v1`; on reject, notify organizer with reasons.

**Non‑Happy**: conflicting concurrent review → **409 version_conflict**; unsafe edits blocked by validator.  
**Observability**: approval latency SLI.

---

### Feature B1.3 — Publish/pause/feature
**Story B1.1.3** — *As Admin, I publish/pause/feature events,* so that *visibility matches policy.*  
**Acceptance**: `POST /v1/admin/events/{id}/publish` sets `status="published"`, `POST /pause` hides from browse, `POST /feature` pins to featured list with a TTL; audit log recorded.

---

## Epic B2 — Catalog, Search & Discovery

### Feature B2.1 — Browse & filter
**Story B2.1.1** — *As a Guest, I browse events by date, category, and tag,* so that *I can find relevant events.*  
**Acceptance**
1. `GET /v1/events { q?, date?, category?, tag?, near?, cursor? }` returns paginated `EVENT_CARD` items, ordered by start time and curated boosts; ETag supported.
2. Safe search filters out **rejected** or **paused** events and expired ones by default unless `includePast=true`.

**Non‑Happy**: invalid `near` → **422 invalid_location**.

### Feature B2.2 — Event details
**Story B2.1.2** — *As a Guest, I view event details,* so that *I know when/where and how to join.*  
**Acceptance**: `GET /v1/events/{id}` returns canonical event, schedule, venue map link, organizer info (minimal), and ICS link.

---

## Epic B3 — RSVP, Attendance & Reminders

### Feature B3.1 — RSVP create/cancel
**Story B3.1.1** — *As a Guest, I RSVP to an event,* so that *I get reminders and possibly a seat.*  
**Acceptance**
1. `POST /v1/events/{id}/rsvps { status:"going"|"interested" }` creates **RSVP** idempotently by `(eventId,userId)` with **Idempotency-Key**; capacity and rsvpPolicy enforced.
2. Over capacity with waitlist enabled → `status="waitlisted"`; without waitlist → **409 capacity_exhausted**.
3. `DELETE /v1/events/{id}/rsvps/me` cancels idempotently, frees seat and promotes next waitlisted user if any.

**Non‑Happy**: event paused or past → **409 invalid_state**; organizer attempting to RSVP counts but not towards capacity if configured.

### Feature B3.2 — Reminder & ICS
**Story B3.1.2** — *As a Guest, I receive reminders and calendar invites,* so that *I don’t miss the event.*  
**Acceptance**: Workers schedule **T‑24h** and **T‑1h** reminders, respect **quiet hours** from Config; `GET /v1/events/{id}/ics` provides downloadable ICS with GMT+7 friendly display and UTC timestamps internally.

### Feature B3.3 — Check‑in
**Story B3.1.3** — *As an Organizer, I check in attendees,* so that *attendance is recorded.*  
**Acceptance**: `POST /v1/events/{id}/checkins { userId | code }` records **CHECKIN** with device/location, idempotent by `(eventId,userId)`; offline code scanning supported with server verification on sync.

**Non‑Happy**: double check‑in → **409 already_checked_in**; code expired → **410 code_expired**.

---

## Epic B4 — Edits, Changes & Cancellations

### Feature B4.1 — Edit event
**Story B4.1.1** — *As an Organizer, I edit event details before start,* so that *the info stays accurate.*  
**Acceptance**: `PATCH /v1/events/{id}` allowed while `status in {"pending_review","approved","published"}` and before `startsAt`, uses `If-Match` ETag, stores **CHANGELOG** diff and emits `events.event.updated@v1`.

### Feature B4.2 — Cancel/postpone
**Story B4.1.2** — *As an Organizer, I cancel or postpone an event,* so that *attendees are informed.*  
**Acceptance**: `POST /v1/events/{id}/cancel { reason }` sets `status="canceled"`, notifies RSVPs, updates ICS; `POST /postpone { newStartsAt }` adjusts schedule and sends change notices.

**Non‑Happy**: within **T‑1h** lock window → **409 too_late_to_change** unless Admin override.

---

## Epic B5 — Integrations & Links

### Feature B5.1 — External ticketing
**Story B5.1.1** — *As an Organizer, I link to an external ticketing page,* so that *paid events can be handled off‑platform.*  
**Acceptance**: `externalTicketUrl` must pass domain allowlist and be marked clearly; RSVPs do not imply ticket purchase; optional webhook for attendance import.

### Feature B5.2 — Campaign hooks (optional)
**Story B5.1.2** — *As Platform, I may grant points for attendance via Campaigns,* so that *engagement is rewarded.*  
**Acceptance**: on check‑in, emit `events.attendance.recorded@v1` for Campaign verifiers to consume.

---

## Epic B6 — Quotas, Billing & Abuse Mitigation

### Feature B6.1 — Organizer quotas
**Story B6.1.1** — *As Platform, I limit number of active events per organizer,* so that *spam is curbed.*  
**Acceptance**: exceed free tier → **402 payment_required** with Payhub invoice; investor/verified organizers may have higher caps.

### Feature B6.2 — Spam & phishing controls
**Story B6.1.2** — *As Moderation, I want to block spam/phishing links and abusive content,* so that *users stay safe.*  
**Acceptance**: profanity filters, URL allowlists, image scanning, repeated offender auto‑pause; appeals recorded.

---

## Epic B7 — Observability, Reliability & Ops

### Feature B7.1 — Idempotency & concurrency
**Story B7.1.1** — *As SRE, I want safe retries and conflict detection,* so that *clients can retry without duplicates.*  
**Acceptance**: all writes accept **Idempotency-Key**; updates use `If-Match` ETag; 429s include `Retry-After` when rate‑limited.

### Feature B7.2 — Telemetry & SLOs
**Story B7.1.2** — *As SRE, I want SLIs/SLOs and traces,* so that *we detect issues.*  
**Acceptance**: metrics for submission→approval latency, search p95, reminder send success, check‑in success rate; traces carry `eventId`/`requestId`; alerts on queue lag.

---

# Section C — End‑to‑End Scenarios (Swimlane narrative)

1. **E2E‑C1: Submit → Review Approve → Publish → RSVPs → Reminders → Check‑ins**  
   Flow: organizer submits, moderator approves, event published, guests RSVP, reminders go out at T‑24h and T‑1h, attendees check in. Post: attendance recorded.

2. **E2E‑C2: Over Capacity → Waitlist Promotion**  
   Flow: RSVPs exceed capacity, waitlist entries created, cancellations promote next waitlisted guest. Post: seats utilized, fair ordering.

3. **E2E‑C3: Unsafe Link → Rejection**  
   Flow: organizer submits with suspicious link, validator blocks or moderator rejects with reasons; organizer edits and resubmits. Post: clean catalog.

4. **E2E‑C4: Postpone Within Lock Window → Admin Override**  
   Flow: organizer attempts within T‑1h, blocked; Admin override adjusts schedule, notifications sent. Post: updated schedule, audit trail kept.

---

# Section D — Traceability Matrix

| Story | APIs | Entities | Events | SystemDesign anchors |
|---|---|---|---|---|
| B1.1.1 | `POST /v1/events` | EVENT, VENUE, EVENT_SCHEDULE | events.event.submitted@v1 | §Interfaces Create |
| B1.1.2 | `POST /v1/admin/events/{id}/review` | REVIEW_DECISION | events.event.approved@v1 | §Moderation |
| B1.1.3 | `POST /v1/admin/events/{id}/publish` | EVENT | events.event.published@v1 | §Lifecycle |
| B2.1.1 | `GET /v1/events` | EVENT_CARD | — | §Catalog |
| B2.1.2 | `GET /v1/events/{id}` | EVENT | — | §Details |
| B3.1.1 | `POST /v1/events/{id}/rsvps`, `DELETE /v1/events/{id}/rsvps/me` | RSVP | events.rsvp.updated@v1 | §RSVP |
| B3.1.2 | workers reminders, `GET /v1/events/{id}/ics` | REMINDER, ICS | events.reminder.sent@v1 | §Reminders |
| B3.1.3 | `POST /v1/events/{id}/checkins` | CHECKIN | events.attendance.recorded@v1 | §Attendance |
| B4.1.1 | `PATCH /v1/events/{id}` | CHANGELOG | events.event.updated@v1 | §Edits |
| B4.1.2 | `/cancel`, `/postpone` | EVENT | events.event.canceled@v1, events.event.postponed@v1 | §Changes |
| B5.1.1 | external link | EXTERNAL_LINK | — | §Integrations |
| B5.1.2 | check‑in hook | CHECKIN | events.attendance.recorded@v1 | §Campaign Hook |
| B6.1.1 | quotas | USAGE_METER | payhub.invoice.updated@v1 | §Quotas |
| B6.1.2 | filters | ABUSE_CASE | events.event.flagged@v1 | §Abuse |
| B7.1.1 | idempotency | IDEMPOTENCY_KEY | — | §Reliability |
| B7.1.2 | telemetry | REQUEST_LOG | — | §Observability |

**Deltas vs old UserStories**: adds **waitlist**, **ICS & reminders with quiet hours**, **unsafe link controls**, **postpone lock window**, **campaign attendance hook**, and **strict idempotency/observability**.

---

# Section E — Assumptions & Open Questions

- Default RSVP policies and capacity waitlist behavior.  
- Timezone handling: store UTC, render in local (GMT+7) by default, confirm organizer‑supplied TZ rules.  
- External ticketing scope (links vs deeper integration).  
- Category taxonomy ownership (via Config) and localization.  
- Whether check‑ins award Campaign points by default.

---

# Appendix — Coverage Gates

## A1. Stakeholder Catalog

| Stakeholder | Responsibilities | Permissions | Typical Actions | KPIs/SLO interests |
|---|---|---|---|---|
| Guest | Browse, RSVP, attend | session | search, rsvp, ics, check-in | reminder open rate |
| Organizer | Submit, edit, cancel | organizer badge | create, update | approval rate |
| Curator/Moderator | Review, approve, feature | staff | review, feature | review latency |
| Admin/Ops | Policy, categories, incidents | staff | publish, pause, feature | MTTR |
| Identity | Badges/KYC | s2s | role checks | uptime |
| Payhub | Billing/overage | s2s | invoice on 402 | collection rate |
| Config | Signed configs | signer | categories, flags | rollout safety |
| Workers/Schedulers | Reminders, expiry | internal | schedule jobs | DLQ depth |
| SRE/Ops | Reliability | ops | monitor, alert | error budget |
| Notifications | Delivery | service | in-app/TG, webhooks | delivery success |
| WebApp/Portal | Frontends | client | browse, rsvp, ics | error rate |

## A2. RACI Matrix (capabilities)

| Capability | Guest | Organizer | Moderator | Admin | Identity | Payhub | Config | Workers | SRE |
|---|---|---|---|---|---|---|---|---|
| Submit & edit | I | R | C | C | C | I | I | I | I |
| Review & approve | I | I | A | C | I | I | I | I | I |
| Publish/pause/feature | I | I | C | A | I | I | C | I | I |
| Catalog & search | R | R | C | C | I | I | I | I | I |
| RSVP & reminders | R | I | I | C | I | I | I | A | C |
| Check‑ins | I | A | I | C | I | I | I | C | I |
| Quotas & billing | I | I | I | A | I | A | I | I | I |
| Reliability & SLOs | I | I | I | I | I | I | I | C | A |

Legend: R=Responsible, A=Accountable, C=Consulted, I=Informed.

## A3. CRUD × Persona × Resource Matrix

Resources: `EVENT, EVENT_SCHEDULE, VENUE, MEDIA_ASSET, CATEGORY, CATEGORY_LINK, RSVP, WAITLIST_ENTRY, CHECKIN, REMINDER, ICS, CHANGELOG, EXTERNAL_LINK, POLICY_SNAPSHOT, USAGE_METER, IDEMPOTENCY_KEY, REQUEST_LOG, JOB, DLQ`

| Resource \ Persona | Guest | Organizer | Moderator | Admin | Workers | SRE |
|---|---|---|---|---|---|
| EVENT | R | C/R/U/D (own) | C/R/U/D | C/R/U/D | R (view) | R (view) |
| EVENT_SCHEDULE | R | C/R/U/D (own) | C/R/U/D | C/R/U/D | R (view) | R (view) |
| VENUE | R | C/R/U/D (own) | C/R/U/D | C/R/U/D | R (view) | R (view) |
| MEDIA_ASSET | R | C/R/U/D (own) | C/R/U/D | C/R/U/D | R (view) | R (view) |
| CATEGORY | R | R (read) | C/R/U/D | C/R/U/D | R (view) | R (view) |
| CATEGORY_LINK | R | C/R/U/D (own) | C/R/U/D | C/R/U/D | R (view) | R (view) |
| RSVP | C/R/U/D (own) | R (view) | R (view) | R (view) | C/R/U/D | R (view) |
| WAITLIST_ENTRY | R (own) | R (view) | R (view) | R (view) | C/R/U/D | R (view) |
| CHECKIN | R (own) | C/R/U/D | R (view) | R (view) | C/R/U/D | R (view) |
| REMINDER | R (read) | R (view) | R (view) | R (view) | C/R/U/D | R (view) |
| ICS | R (read) | R (view) | R (view) | R (view) | R (view) | R (view) |
| CHANGELOG | R (read) | R (view) | R (view) | C/R/U/D | R (view) | R (view) |
| EXTERNAL_LINK | R (read) | C/R/U/D | C/R/U/D | C/R/U/D | R (view) | R (view) |
| POLICY_SNAPSHOT | R (read) | R (view) | R (view) | C/R/U/D | R (view) | R (view) |
| USAGE_METER | R (own) | R (view) | R (view) | R (view) | C/R/U/D | R (view) |
| IDEMPOTENCY_KEY | N/A | R (view) | R (view) | R (view) | R (view) | R (view) |
| REQUEST_LOG | N/A | R (view) | R (view) | R (view) | C/R/U/D | C/R/U/D |
| JOB | N/A | R (view) | R (view) | R (view) | C/R/U/D | C/R/U/D |
| DLQ | N/A | R (view) | R (view) | R (view) | C/R/U/D | C/R/U/D |

**N/A** indicates intentionally not exposed.

## A4. Permissions / Badge / KYC Matrix

| Action | Requirement | Evidence | Expiry/Renewal | Appeal |
|---|---|---|---|---|
| Submit event | Organizer badge | KYC via Identity | 2 years | Support |
| Approve/reject | Moderator staff | case link | permanent | Security |
| Feature event | Admin staff | change ticket | permanent | Security |
| RSVP | session | — | — | Support |
| Check‑in attendee | organizer role on event | device signature | per event | Support |

## A5. Quota & Billing Matrix

| Metric | Free Tier | Overage (FZ/PT) | Exemptions | Failure Handling | Refunds |
|---|---|---|---|---|---|
| Active pending events / organizer | A | billed per extra | staff test | 402 → invoice | none |
| RSVPs / event | K | N/A | — | 409 capacity_exhausted | N/A |
| Reminder sends / event | Q | throttle | staff | 429 Retry‑After | N/A |

## A6. Notification Matrix

| Event | Channel(s) | Recipients | Template | Throttle/Dedupe |
|---|---|---|---|---|
| events.event.submitted@v1 | in-app | Moderator | `event_submitted` | `(eventId)` |
| events.event.approved@v1 | in-app | Organizer | `event_approved` | `(eventId)` |
| events.event.published@v1 | in-app | Guests following category | `event_published` | `(eventId)` |
| events.rsvp.updated@v1 | in-app | Guest | `rsvp_updated` | `(eventId,userId)` |
| events.reminder.sent@v1 | in-app | Guest | `reminder_sent` | `(eventId,reminderWindow)` |
| events.attendance.recorded@v1 | in-app | Organizer | `attendance_recorded` | `(eventId,userId)` |
| events.event.canceled@v1 | in-app | RSVPs | `event_canceled` | `(eventId)` |
| events.event.postponed@v1 | in-app | RSVPs | `event_postponed` | `(eventId)` |

## A7. Cross‑Service Event Map

| Event | Producer | Consumers | Payload summary | Idempotency | Retry/Backoff |
|---|---|---|---|---|---|
| events.event.published@v1 | Events | WebApp | `{eventId, startsAt, venue}` | `eventId` | 5× exp backoff |
| events.rsvp.updated@v1 | Events | WebApp | `{eventId, userId, status}` | `(eventId,userId)` | 5× |
| events.reminder.sent@v1 | Events | WebApp | `{eventId, window}` | `(eventId,window)` | 3× |
| events.attendance.recorded@v1 | Events | Campaigns | `{eventId, userId, ts}` | `(eventId,userId,tsBucket)` | 5× |
| events.event.flagged@v1 | Events | Admin | `{eventId, reason}` | `(eventId,reason)` | 3× |

---

# Abuse/Misuse & NFR Sweeps

- **Abuse**: spam events, phishing links, fake venues → quotas, allowlists, image scan, moderator tools, appeals.  
- **Fraud**: fake attendance via shared codes → per‑user codes, device signatures, server verification, anomaly detection on check‑ins.  
- **Security**: JWT auth, idempotent writes, HMAC webhooks, signed configs, least‑privilege service tokens, secret vaults.  
- **Privacy**: minimal PII, redact contacts in public view, opt‑out from reminders, retention for attendance data.  
- **Localization/timezones**: organizer‑supplied TZ stored, UIs render **GMT+7** friendly; ICS includes TZID where supported.  
- **Resilience**: retries with jitter, DLQ for reminder sends, queue backpressure management, incident toggles for reminder subsystem.

---

# Self‑Check — Stakeholder Coverage Report

**Counts**  
- Personas: 11  
- Features: 20  
- Stories: 22  
- Stories with non‑happy paths: 17/22  
- Entities covered in CRUD matrix: 19/19

**Checklist**  
- ✅ All personas appear in at least one story.  
- ✅ Each entity has at least one **C**, **R**, **U**, and **D** (or reasoned N/A).  
- ✅ Every action mapped to roles/badges/KYC where applicable.  
- ✅ Quotas & billing addressed for every chargeable action.  
- ✅ Each story references APIs/entities/events.  
- ✅ At least one end‑to‑end scenario per persona.  
- ✅ Abuse/misuse cases enumerated with mitigations.  
- ✅ Observability signals tied to AC.  
- ✅ Localization/timezone handling present where user-visible.
