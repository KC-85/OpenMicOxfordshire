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

## Key Workflows

### Submission (guest)

- Guest fills `/submit/` + Turnstile -> **PENDING**, `email_verified=False`
- Email with signed token -> `/verify-email/?token...`
- After verify -> stays **PENDING** until moderator approval
- Moderator approves -> **PUBLISHED**, `published_at` set; confirmation email

### Submission (logged-in)

- Auth user submits -> **PENDING** (`email_verified=True`)
- Moderator approves -> **PUBLISHED**

### Moderation 

- Queue shows newest first; Approve/Reject inline (HTMX).
- Block approval if guest not verified.
- Bulk actions for power-mods (Admin).

### Archival

- Celery job hourly: `end_at < now()` -> `status=ARCHIVED`.

### Subscriptions

- User submits email (+ optional town/category) -> verify link -> weekly digest Mon 09:00

### Abuse Reporting

- Detail page -> Report -> mod inbox/queue -> mark Reviewed/Dismissed.

## UI/UX (CSS(Minimal), JavaScript(Minimal), Bootstrap 5 + HTMX)

### Intent

- Keep custom styling and scripting as light as possible
- Bootstrap covers the UI; HTMX handles interactivity.
- Custom CSS/JS is limited to small helpers that improve usability, accessibility and perceived performance.

### UI/UX scope

- Use Bootstrap 5 as the primary design system (layout, components, utilities).
- Use HTMX for all dynamic interactions (filters, paginations, moderation actions).
- Add a small custom CSS file for a few utility classes (e.g., consistent card image aspect, subtle line-clamping for excerpts, visible focus outline, and a simple loading overlay for HTMX targets).
-Add a small JS "glue" file for: CSRF header on HTMX requests, a basic confirm dialog for destructive actions, make Bootstrap cards dynamic on mouse hover, and optional Bootstrap toasts triggered by server responses.

### File Structure

- One minimal CSS file for utility rules only (Maybe site-wide fonts/colours).
- One minimal JS file for HTMX/Bootstrap glue only.
- Optional: One tiny "app init" JS file if we later enable Bootstrap tooltips or similar niceties.

### Usage Patterns

- Listing pages: Use HTMX to update the grid when filters change or on "Load more". The loading overlay appears only on the affected container; content remains readable.
- Moderation queue: Approve/Reject actions dispatch small HTMX requests; responses may trigger a toast notification (success/error).
- Forms: Rely on Bootstrap validation states; JS only prevents double submission on long operations if needed.

### Accessibility

- Ensure visible focus outlines and keyboard navigability across filters, buttons, and links.
- Provide alt text for all images and meaningful labels for form controls.
- Keep interactions usable without JavaScript; HTMX enhances but does not block basic navigation and form submission.
- WCAG everything.

### Performance and UX

Keep custom CSS under ~50 lines at launch; avoid component re-styling unless necessary.
Defer all custom JS; avoid heavy libraries.
Lazy-load images via native attributes; prefer fixed image heights on cards to minimise layout shift.
Limit DOM changes to specific containers; avoid full-page swaps.

### Conventions

- Prefer Bootstrap utility classes over writing new CSS.
- Keep HTMX endpoints returning small, cache-friendly HTML fragments.
- Use concise, consistent microcopy for actions (e.g., Approve, Reject, Load more).
- Centralise any “flash” messages as Bootstrap toasts; trigger them via response headers (no global event bus).

### Testing considerations

- Verify keyboard-only flows for filtering, submitting, and moderating.
- Confirm that list updates, toasts, and confirm prompts behave consistently across modern browsers.
- Check that pages render and remain usable when JS is disabled (progressive enhancement).
- JavaScript testing to be used (however minimal).

### Risks & mitigations

- Drift toward heavier front-end code: review PRs to keep CSS/JS minimal; prefer Bootstrap utilities and server-rendered partials.
- UI regressions from ad or analytics scripts: load non-essential scripts after content, and monitor Core Web Vitals.
- Overuse of custom styles: document any additions; remove unused rules during iterations.

### Definition of done (for this section)

- Minimal CSS and JS files exist, documented, and loaded globally.
- Key pages (list, detail, submit, moderation) use Bootstrap defaults with minimal overrides.
- HTMX interactions are scoped, accessible, and progressively enhanced.
- No custom JS frameworks included; bundle remains small and deferrable.

## Anti-Abuse, Trust & Content Policy

- Cloudflare Turnstile on submit/report; `django-ratelimit` (e.g., 5/min/IP).
- Disallow approval if `guest_email` not verified.
- Profanity/banned words check (server-side list).
- Contact relay form hides organiser email (prevents scraping).
- Policy pages: Terms, Privacy, Content Guidelines (no hate speech, illegal content, scams, explicit content, counterfeit ticketing).
- If there are tickets required for an event, then a link to the official ticket seller will be provided.

## Privacy, GDPR & Legal

- Lawful basis: **Legitimate interests** (service provision) **+ Consent** (marketing emails/subscriptions).
- Store minimal organiser PII; never display raw emails; purge unneeded data.
- DSR (Data Subject Rights): export/delete on request; contact email published.
- Logs: store IPs for abuse reports up to 90 days; rotate logs.
- Cookies: essential only (session, CSRF); provide cookie notice if adding analytics.

## Tech Stack & Architecture

- Backend: Django 5.x
- DB: PostgreSQL 15+ (with `django.contrib.postgres` FTS) or managed database (Postgres)
- Async: Celery + Redis (expiry, digests)
- Frontend: Django Templates, HTMX, Bootstrap 5, minimal CSS/JS
- Email: SendGrid/Mailgun (prod), console backend (dev)
- Storage: Local in dev; S3/R2/Ionos/Cloudinary for prod media
- Reverse proxy: Nginx; Gunicorn app server
- Infra: Docker Compose or K3s (dev/prod parity)

### Environments

- Development: DEBUG on, console emails, PostgreSQL
- Staging: DEBUG off, test SMTP
- Production: DEBUG off, HTTPS, object storage, managed PostgreSQL

## Deployment, CI/CD & Operations (DevOps)

### Environments

- Development: local Docker Compose; debug on; console emails.
- Staging: prod-like; sanitized data; feature flags on.
- Production: autoscaled or single-node; strict env vars; monitoring + backups.

### CI (build + quality gates)

- Providers: GitHub Actions (default) • GitLab CI • CircleCI • Azure DevOps.
- Static checks: ruff/flake8, black (check), isort, mypy (optional).
- Tests: pytest with coverage gate (e.g., ≥90% on critical paths). Jest for JS.
- Security scans: pip-audit • Trivy (image scan) • Bandit.
- Caches: pip cache, Docker layer cache.
- Artifacts: test reports, coverage, migration plan output.

### Build & Registry

- Image build: Docker Buildx (multi-arch) • Buildpacks (pack) • Nix (advanced).
- Registries: GHCR (default) • Docker Hub • GitLab Registry • AWS ECR • GCP GAR.
- Tagging: `main-<shortsha>` + `release-<semver>`; signed images (cosign, optional).

### CD (deploy strategies)

- Simple: SSH/rsync + Compose (VPS).
- Managed: Fly.io • Render • Railway (zero-maintenance).
- Cloud: ECS Fargate • GKE Autopilot • Azure Web Apps.
- Strategies: Rolling (default) • Blue/Green (two stacks, instant cutover) • Canary (small % first).
- Migrations: “Deploy → run migrations → flip traffic.” Gate with health checks.

### Infra/Hosting Options

- Self-hosting (Possible)
- VPS: Hetzner/OVH/DigitalOcean/IONOS (cost-effective, hands-on).
- PaaS: Fly.io/Render (fastest to ship).
- Kubernetes (later): k3s on VPS • GKE/EKS/AKS (if multi-service scale).
- Workers: Celery worker + beat as separate processes/containers.

### Secrets & Config

- Source of truth: .env in secrets manager; never in repo.
- Stores: GitHub Encrypted Secrets • Doppler • 1Password Secrets • AWS SSM • GCP Secret Manager.
- Rotation: quarterly or on role changes; short-lived keys where possible.

### TLS, Domains & Edge

- TLS: Let’s Encrypt via Caddy/Traefik/Certbot (auto-renew).
- DNS: Cloudflare or Route53; A/AAAA + CNAME for CDN if used.
- CDN (optional): Cloudflare/Akamai/Fastly for static/media caching.
- HTTP: HSTS, gzip/brotli, sane cache headers for static.

### Database Operations

- Engine: PostgreSQL managed (RDS/CloudSQL/Neon/DO-Droplet/IONOS) or self-hosted in Compose.
- Migrations: Django migrate in CI gate + pre-deploy dry run on staging.
- Connections: pgbouncer (if high concurrency).
- Maintenance: weekly VACUUM ANALYZE (managed DBs handle this).

### Backups & Disaster Recovery

- Backups: nightly pg_dump + weekly base backup (WAL if available).
- Storage: encrypted in S3/R2/Backblaze (7/30/90 retention).
- Tests: quarterly restore drill to staging; checksum/size alerts.
- RTO/RPO targets: RTO ≤ 4h, RPO ≤ 24h (tune if donations/ads grow).

### Observability (logs, metrics, tracing)

- Errors: Sentry (default) or Rollbar.
- Logs: ELK/Opensearch • Loki + Grafana • Papertrail/Logtail (SaaS).
- Metrics: Prometheus + Grafana • UptimeRobot/Healthchecks.io (external).
- Health: /healthz (app) + /readiness (DB/redis checks).

### Analytics & Privacy

- Analytics: Plausible/Matomo (default) • GA4 (only with consent).
- Consent: CMP banner controls ads/analytics load.
- Anonymization: IP anonymize; respect DNT; minimal cookies.

### Performance & Caching

- App-level: Django cache (Redis) for fragments/HTMX partials (30–120s).
- Static/media: CDN with long TTL + cache-busting.
- Images: thumbnails; lazy-load; fixed card heights to reduce CLS.

### Scheduling & Automations

- Celery Beat (default): expiry sweeper, weekly digest, backup triggers.
- Alt schedulers: systemd timers • GitHub Actions cron (for ancillary tasks).
- Job reliability: idempotent tasks; retry with backoff; dead-letter queue (optional).
- Ansible also possible for automation - also consider n8n.

### Rollbacks & Releases

- Versioning: semantic tags; changelog per release.
- Rollback: keep previous image + DB state; “migrations down” not guaranteed—prefer forward fixes.
- Feature flags: gate risky features (django-waffle) to staging/users.

### Access & Security

- Access control: least privilege; SSH keys or SSO; no shared root.
- Firewall: only 80/443 open; DB/Redis private network only.
- Patching: base image updates monthly; dependency scans in CI.
- Policies: incident response runbook; on-call escalation (lightweight).

### Cost Tiers (rough)

- Lean (≈ £10–£25/mo): single VPS + managed email + R2 storage.
- Comfort (≈ £25–£60/mo): VPS + managed Postgres + CDN + Sentry.
- Scale (≥ £100/mo): managed DB + autoscaling app + observability stack.

### Go/No-Go Checks (per deploy)

- CI green (tests, lint, security scans).
- Staging smoke tests passed; migrations ran.
- Health checks green; error rate baseline stable.
- Backups completed within last 24h.

## Configuration & Environment Variables (Sample)

- DEBUG=false 
- SECRET_KEY=... 
- DATABASE_URL=postgres://user:pass@db:00000/openmic 
- REDIS_URL=redis://redis:00000/0 
- TIME_ZONE=Europe/London 
- ALLOWED_HOSTS=openmic-oxon.example.com
- EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend 
- EMAIL_HOST=smtp.sendgrid.net 
- EMAIL_HOST_USER=apikey 
- EMAIL_HOST_PASSWORD=... 
- DEFAULT_FROM_EMAIL=no-reply@openmic-oxon.uk 
- TURNSTILE_SITE_KEY=... 
- TURNSTILE_SECRET_KEY=... 
- MEDIA_BACKEND=s3
- S3_BUCKET=... 
- S3_ACCESS_KEY=... 
- S3_SECRET_KEY=... 
