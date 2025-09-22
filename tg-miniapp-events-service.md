# tg-miniapp-events-service

Events directory for the Telegram Mini App. Hosts basic event metadata and links out to external booking systems (Lu.ma, Eventbrite, Eventpop, Meetup, etc.). No ticketing is processed here.

---

## 1) Service Story

**Events Service** lets organizers list events discoverable inside the WebApp while keeping bookings on trusted external platforms. It stores a clean schema (title, image, description, host, place name, location, start/end time, external link, conditions) and exposes fast read APIs for feeds and detail pages. Admins can create/update events and toggle visibility.

---

## 2) Duties & Scope

### Owns (SoR)

* **Events**: metadata, visibility, categories/tags
* **Hosts**: name, links, verified flag
* **Attachments/Images**: URLs or file references (hosted elsewhere/CDN)

### Not owned here

* Ticketing/RSVP/payment — handled by external platforms via `externalUrl`
* Wallet/ledger — Payhub
* Campaigns — separate service (can reference event IDs for quests)

---

## 3) Public API (MVP)

> All admin mutations require `Idempotency-Key`. Public reads are cache‑friendly.

* `GET  /v1/events?from=<ISO>&to=<ISO>&q=<text>&tag=<tag>&city=<city>&limit=50` → list
* `GET  /v1/events/:eventId` → detail

**Response shape (detail)**

```json
{
  "eventId": "evt_abc",
  "title": "Crypto Builders Meetup",
  "imageUrl": "https://cdn.example.com/img/evt_abc.jpg",
  "description": "…",
  "host": { "name": "YourOrg", "url": "https://yourorg.example" },
  "placeName": "Hub Coworking",
  "location": { "city": "Bangkok", "country": "TH", "lat": 13.736, "lng": 100.523 },
  "startsAt": "2025-10-01T11:00:00Z",
  "endsAt": "2025-10-01T14:00:00Z",
  "externalUrl": "https://lu.ma/xyz",
  "conditions": ["18+", "Bring ID"],
  "tags": ["meetup", "builders"],
  "visibility": "public"
}
```

---

## 4) Admin API (used by tg-miniapp-admin)

* `POST   /admin/v1/events` — create draft
* `PATCH  /admin/v1/events/:eventId` — update
* `POST   /admin/v1/events/:eventId/publish` — set `visibility=public`
* `POST   /admin/v1/events/:eventId/unpublish` — set `visibility=hidden`
* `GET    /admin/v1/events` — search/paginate

Role required: `admin.events.*` via Identity.

---

## 5) Architecture

* **Node.js + TypeScript + Express**
* **MongoDB** for flexible metadata, or **PostgreSQL** (either is fine; examples assume Mongo)
* **Redis** for response caching and rate limiting
* **OpenTelemetry** traces/metrics; Pino logs
* **Zod** for DTOs

**Caching model**

* Cache list/detail responses in Redis with short TTL (e.g., 60–120s)
* Invalidate on publish/update

---

## 6) Data Model

* `Event` `{ eventId, title, imageUrl, description, host{name,url,verified?}, placeName, location{city,country,lat?,lng?}, startsAt, endsAt, externalUrl, conditions[], tags[], visibility: public|hidden|draft, createdAt, updatedAt }`
* `Host`  `{ hostId, name, url, verified, createdAt }` (optional; can be embedded in Event)

Indexes: `Event(visibility, startsAt)`, `Event(location.city)`, text index on `title, description`

---

## 7) ENV & Configuration

```dotenv
SERVICE_ID=events
SERVICE_NAME=tg-miniapp-events-service
PORT=8096
NODE_ENV=development
LOG_LEVEL=info

MONGO_URL=mongodb://localhost:27017/tg_events
REDIS_URL=redis://localhost:6379

# CORS/Origins for WebApp/Admin
ALLOWED_ORIGINS=https://app.tg-miniapp.example.com,https://admin.tg-miniapp.example.com

# Pagination & cache
DEFAULT_PAGE_SIZE=50
CACHE_TTL_SECONDS=90
```

---

## 8) Dependencies

* `express`, `zod`, `mongodb`, `ioredis`
* `pino`, `pino-http`, `helmet`, `cors`
* Dev: `typescript`, `tsx`, `vitest`/`jest`, `supertest`, `eslint`

---

## 9) Develop & Run

```bash
pnpm i
pnpm dev       # http://localhost:8096
```

Docker/Compose similar to other services; link Mongo/Redis.

---

## 10) Testing

* **API**: create/publish/unpublish flow, list filters (time window, city, tags), detail shape
* **Cache**: list/detail caching and invalidation
* **Validation**: time ordering (endsAt after startsAt), URL formats

Run:

```bash
pnpm test
```

---

## 11) Deploy

Helm (excerpt):

```yaml
image: { repository: ghcr.io/yourorg/tg-miniapp-events-service, tag: "v0.1.0" }
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
probePaths: { liveness: /healthz, readiness: /readyz }
```

---

## 12) Roadmap

* Geospatial index & radius search
* Organizer verification workflow
* Calendar export (iCal) and per‑user reminders
