# OpenMicOxfordshire

## Overall Vision

A moderated, county-wide hub for open-mic nights (music, comedy, spoken word, jams) across Oxfordshire,
with simple submissions for organisers, easy discovery for performers/audiences, and reliable listings.

This will become the single source for "what's on" in Oxfordshire's grassroots scene.

### Goals & Success Metrics

#### Primary Goals

- Centralise event discovery for the county
- Lower friction for organisers to publish accurate events
- Build trust via light moderation & verification

#### Success Metrics

- 20+ approved events published
- 10+ unique organisers posting
- 250 monthly unique visitors/2+ pages per session
- Email digest CTR => 15%, bounce rate <35%

### Audience & Personas

- **Venue Owner/Organizer (Person 1):** Needs fast event submission, editing and simple verification.
- **Performer (Person 2):** Filters by town/category/date; wants .ics add-to-calendar; may subscribe to towns.
- **Audience Member (Person 3):** Wants "Tonight/This Week lists; mobile-friendly browsing.
- **Moderator/Admin (Person 4):** Needs an efficient approval queue and anti-spam tools.

## Scope

### MVP (4-6 weeks)

- Event submission (guest or logged-in), email verification, moderation queue.
- Public list (cards) with filters: **Town, Category, Date, Search.**
- Event detail with .ics download and organiser contact relay form.
- Auto-archive past events (Celery).
- Weekly "What's On" email digest (opt-in subscribers).
- Bootstrap 5 UI; HTMX for filters/pagination; Postgres FTS for search.
- Anti-abuse: Cloudflare Turnstile, rate limit (may add Axes), basic bad-word filter.

### v1.0 (8-12 weeks(after thorough testing))

- Subscriptions: By town and/or category, double opt-in + unsubscribe link.
- Featured events basic analytics (views) per event.
- Venue/Town directory page; CSV export for organisers.
- Public JSON feed (Read-Only) of upcoming events.

### Out of scope/later additions

- Paid ticketing, seat maps, two-way calendar sync, performer profiles, reviews/ratings.

## Non-Functional Requirements

- **Performance:** P95 page load < 1.5s on 4G/5G; list queries P95 < 300ms.
- **Availability:** => 99.5% monthly (single-node acceptable for development).
- **Accessibility:** WCAG 2.2 AA; full keyboard navigation; alt text required.
- **Security:** HTTPS everywhere; OWASP Top 10 mitigations, data minimisation.
- **Privacy:** GDPR-compliant (lawful basis, retention limits, unsubscribe).
- **Portability:** Dockerised/K3s local/production parity.

## Information Architecture & URLs

### Public

- / Home: Upcoming events list (filters + "Load more").
- `/e/<uuid>/` - Event detail (+ Add to Calendar .ics).
- `/submit/` - Submit event (guest or user).
- `/verify-email/?token...` - Verify guest email.
- `/subscribe/` - Subscribe to digest(towm/category).
- `/unsubscribe/?token...` - One-click unsubscribe.
- `/towns/` - List towns; each links to filtered view.
- `/feed/upcoming.json` - Read-only JSON feed (v1.0).

### Moderation (protected)

- `/moderation/` - Pending queue (cards with Approve/Reject).
- `/moderation/events/<uuid>/approve` (POST).
- `/moderation/events/<uuid>/reject` (POST).

### HTMX Partials

- `/events/partial-list/` - Returns grid + updated "Load more" button.
- `/moderation/partial-queue/` - Returns current queue.

## Data Model (first pass)

### Entities

#### Event

- `id (UUID, PK), title (140), description (Text), category (music|comedy|spoken|jam|other)`
- `venue_name (140), address (240), town (FK Town, nullable), postcode (10)`
- `start_at (datetime), end_at (datetime)`
- `organiser_name (120), organiser_email (email)`
- `image (optional), status (PENDING|PUBLISHED|REJECTED|ARCHIVED)`
- `email_verified (bool), created_by (FK, User, nullable)`
- **TImestamps:** `published_at, created_at, updated_at`
- **Search:** `search_vector (tsvector)` **+ GIN index**

#### Town

- `name (unique), slug (unique)`

#### Subscription

- `email, town (FK, nullable), category (nullable)`
- `verified (bool), token (unique), created_at`

#### Abuse Report

- `event (FK), reason (short text), reporter_ip, status (OPEN|REVIEWED|DISMISSED), created_at`

#### Indexes

- `Event(status, start_at), Event(category), Event(town)`**, GIN ON** `search_vector`

#### Retention

- Archive past events immediately; hard delete images + PII after 180 days unless legal holds.