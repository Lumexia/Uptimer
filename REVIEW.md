# REVIEW.md

Date: 2026-02-04
Scope: Uptimer repo (exclude `UptimeFlare/` unless explicitly requested)

This document converts the current gaps (vs `Application.md` + `Plan.md`) into a step-by-step TODO plan.

## Baseline (What Already Exists)

- Worker: Hono + Zod API (`/api/v1/public/*`, `/api/v1/admin/*`), scheduled monitor engine, retention, daily rollups.
- Storage: D1 schema + migrations (monitors/state/results/outages/incidents/maintenance/notifications/settings/snapshots).
- Public: status snapshot fast path, status page payload (monitors + 30d uptime bars + incidents + maintenance windows), latency/uptime/outages endpoints.
- Admin: monitor CRUD + test, notification channel CRUD + test, incidents CRUD + updates + resolve, maintenance windows CRUD, analytics + CSV exports, settings.

## Key Gaps (User-Facing)

- Public status page lacks an incident history view (resolved incidents + pagination).
- Public status page lacks a heartbeat bar (last N checks) per monitor; UI has `HeartbeatBar` but it is unused.
- Admin monitor list shows config (`is_active`) but not runtime state (UP/DOWN/UNKNOWN, last check, last error/latency).
- "Test" actions do not surface results in the UI (monitor test + webhook test).
- Monitor creation UI exposes only a subset of HTTP config (headers/body/assertions are supported server-side).
- CSV exports exist server-side but are not exposed in the UI.
- Notification reliability is basic (no retry/backoff workflow; no delivery log UI).
- Automated tests are effectively missing for core logic (state machine, uptime/unknown math, target validation, templates).

## TODO Plan (Steps)

### Step 1 — Public: Incident History (Resolved + Pagination)

- Add an "Incident History" section to `apps/web/src/pages/StatusPage.tsx`.
- Wire `apps/web/src/api/client.ts:fetchPublicIncidents(limit, cursor)` with a "Load more" button.
- UI requirements:
  - Keep current "Active Incidents" as-is (from `/public/status`).
  - Show resolved incidents (from `/public/incidents`) below, newest-first.
  - Open the existing incident detail modal on click.
- Acceptance:
  - Works with cursor pagination; handles empty/loading/error states.

### Step 2 — Public: Heartbeat Bar (Last 60 Checks) Per Monitor

- Decide the data source:
  - Option A (preferred for status page TTFB): extend `/api/v1/public/status` to include per-monitor `heartbeats` (<= 60 points).
  - Option B (minimal API changes): fetch per-monitor data from `/public/monitors/:id/latency` and derive last 60 points client-side (more requests).
- If Option A:
  - Update schema/types:
    - `apps/worker/src/schemas/public-status.ts` (add `heartbeats`).
    - `apps/web/src/api/types.ts` (add `heartbeats`).
  - Update payload generation:
    - `apps/worker/src/public/status.ts` fetch last N rows per monitor efficiently (single SQL using a window function preferred).
    - Ensure payload size stays bounded (N=60, only `checked_at/status/latency_ms`).
  - Render:
    - Use `apps/web/src/components/HeartbeatBar.tsx` inside each monitor card.
- Acceptance:
  - With existing data, each monitor shows a stable 60-bar heartbeat strip.
  - With no data, the UI degrades gracefully.

### Step 3 — Admin: Show Runtime State In Monitor List

- Extend Admin monitors list response to include runtime fields from `monitor_state`:
  - `status`, `last_checked_at`, `last_latency_ms`, `last_error`.
- Update UI:
  - `apps/web/src/pages/AdminDashboard.tsx` monitor table columns to show status + last check time + last error.
- Acceptance:
  - Admin can see "is it down" without opening the public status page.

### Step 4 — Admin: Pause/Resume (True `paused` State)

- Add Admin endpoints (API shape TBD) to pause/resume a monitor via `monitor_state.status`:
  - Pause: upsert `monitor_state.status='paused'`.
  - Resume: set `monitor_state.status='unknown'` (or clear status) so scheduler picks it up.
- Ensure scheduler semantics:
  - `apps/worker/src/scheduler/scheduled.ts` already skips paused monitors; verify resume triggers checks.
- Update UI:
  - Add pause/resume action buttons on the monitors list.
- Acceptance:
  - Paused monitors remain visible on status page with `paused` status.
  - Paused monitors stop scheduled checks.

### Step 5 — Admin: Expose Advanced HTTP Monitor Config In UI

- Extend `apps/web/src/components/MonitorForm.tsx` to support HTTP-only fields already supported in Worker:
  - `http_headers_json`, `http_body`, `expected_status_json`, `response_keyword`, `response_forbidden_keyword`.
- Client-side validation:
  - JSON fields must parse to the expected shape.
- Acceptance:
  - Creating/updating HTTP monitors can configure assertions without using curl.

### Step 6 — Admin: Make "Test" Actions Visible

- Monitor test (`POST /api/v1/admin/monitors/:id/test`): show a result panel/toast in `AdminDashboard`.
- Webhook test (`POST /api/v1/admin/notification-channels/:id/test`): show delivery outcome (status/http_status/error).
- Acceptance:
  - A failed test is diagnosable from the UI.

### Step 7 — Admin: Expose CSV Exports In UI

- Add download links/buttons for:
  - `GET /api/v1/admin/exports/monitors/:id/outages.csv`
  - `GET /api/v1/admin/exports/monitors/:id/check-results.csv`
  - `GET /api/v1/admin/exports/incidents.csv`
- Place in `apps/web/src/pages/AdminAnalytics.tsx` (and/or monitor analytics section).
- Acceptance:
  - Admin can export without manually crafting URLs.

### Step 8 — Notifications: Retry/Backoff + Delivery Log UI

- Retry strategy (no new services baseline):
  - Store failed deliveries as rows in `notification_deliveries` (already done) and introduce a scheduled "retry" job that re-sends recent failures with capped attempts.
  - Keep idempotency (`UNIQUE(event_key, channel_id)`) intact.
- Add Admin API to query deliveries (filter by channel/event/status/time).
- Add a minimal UI page/panel to view recent deliveries and errors.
- Acceptance:
  - Transient webhook failures self-heal; persistent failures are visible.

### Step 9 — Hardening & Observability (MVP Productization)

- Rate limiting (admin endpoints): document Cloudflare WAF/Rate Limiting rules; optionally add lightweight in-app guards.
- Scheduler logs: emit structured logs per tick (counts, durations, failure reasons) to aid debugging.
- Acceptance:
  - Production deployment has a clear recommended protection posture and basic operational visibility.

### Step 10 — Tests (Minimal, High-ROI)

- Add unit tests for:
  - `apps/worker/src/monitor/state-machine.ts` (transitions + thresholds).
  - `apps/worker/src/monitor/targets.ts` (SSRF/blocked ranges).
  - `apps/worker/src/analytics/uptime.ts` (unknown/downtime math).
  - `apps/worker/src/notify/template.ts` (template resolution + forbidden keys).
- Acceptance:
  - CI runs tests; core logic changes are protected against regressions.
