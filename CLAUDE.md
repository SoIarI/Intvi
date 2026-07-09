# Visit Intelligence (intvi)

Standalone single-file web app for tracking field visits to business outlets (Maldives-based sales/support team). Repo: `github.com/SoIarI/intvi`.

## What's in this directory

- **`index.html`** — the entire app. One HTML file: inline `<style>` (design tokens as CSS custom properties, dark/light theme via `data-theme` attr) + inline `<script>` (vanilla JS, no build step, no framework). Uses Leaflet (via CDN) for maps.
- **`0001_init_schema.sql`** — Postgres/Supabase migration for a *future* shared backend (see below). Not yet wired up to `index.html`.
- **`phase-0-prompt.md`** — the original prompt describing the backend foundation project (`team-suite/`) that owns that migration. That's a separate project scaffold this repo doesn't contain yet.

## Current architecture (index.html)

- **Storage:** everything lives in `localStorage` (keys under `STORAGE_KEYS`: `outlets`, `visits`, `assignments`, `people`, `departments`, `zones`, `drafts`, `attendance`, `dayReportDrafts`, `dayReports`, `leaveTypes`, `leaveRequests`). No backend calls except Leaflet map tiles from CDN.
- **State:** single in-memory `state` object, mutated directly then persisted via `persist()`. No framework — views are re-rendered by calling `render<View>()` functions that regenerate `innerHTML` and rewire event listeners each time.
- **Nav sections** (sidebar, one `<section class="view">` each): Import Visits (ingest), Review Queue, Map, Outlets, Intern Summary, Software Dominance, Shift Roster, **HR**, Settings. HR has its own internal sub-tab bar (module-level `hrActiveTab`, same pattern as Roster's shift toggle): Directory, Clock In/Out, Attendance, Day Reports, Leave & Absences, Today's Assignment.
- **People:** `state.people` is the single roster used everywhere (visits, roster assignments, HR) — richer records (`role`/`department`/`employment_type`/`status`/`phone`/`email`/`start_date`/`leave_allowance`) than the old flat name list it replaced, but still **joined by name string** across visits/assignments/attendance/leave, not by id (no auth/ids on the frontend yet). Helpers: `personNames()`, `findPerson(name)`. Legacy `vi_interns` localStorage from earlier versions is migrated into `vi_people` once, on first load with no existing `vi_people` key (`migratePeopleFromLegacyInterns`).
- **Ingest pipeline:** paste raw Telegram messages → `parseFullPaste`/`parseMessageText` splits on `business_name`-style labels into draft visit blocks (label-dictionary matching via `FIELD_SYNONYMS`, tolerant of bullets/emoji/missing fields) → drafts queue in Review → confirming calls `commitDraftAsVisit`, which upserts an outlet by exact normalized-name match (`findMatchingOutlet`, no fuzzy matching) and appends a visit record. **Day Reports** (HR) reuses this same label-dictionary technique via `parseDayReport`/`DAY_REPORT_FIELD_SYNONYMS` for end-of-day text instead of visit text.
- **Zones:** drawn as polygons directly on the Leaflet map (click-to-place points, drag-to-edit vertices, midpoint markers to insert points). `pointInPolygon` ray-casting auto-detects an outlet's zone from lat/lng against drawn boundaries.
- **Roster:** per-date/per-shift person→zone assignments, cross-checked against actual visits logged that day/zone to flag coverage gaps and off-zone visits. HR's Today's Assignment tab is a read-only join of this same data against `state.attendance` — no separate assignment state.
- **Attendance/Leave:** `getOrCreateAttendance(person, date)` is the shared entry point for attendance records (used by Clock In/Out, the Attendance tab, Day Reports, and Leave approval alike). Approving a leave request (`applyApprovedLeaveToAttendance`) writes "On Leave" across its date range so Attendance stays accurate without manual entry.

## Planned backend migration (not yet started against this repo)

`phase-0-prompt.md` + `0001_init_schema.sql` describe a separate `team-suite/` project: a shared Supabase backend unifying this app's data (`outlets`/`visits`/`zones`/`assignments`) with an HR app (`people`/`attendance`/`leave_requests`/`candidates`) under one `people` table with `admin`/`intern` roles enforced via RLS. Per that prompt, Phase 0 is backend-only (schema + RLS, no frontend changes); migrating `index.html` off `localStorage` to read/write Supabase is explicitly deferred to a later phase not yet prompted for.

## Working in this repo

- No build/test tooling — it's a static HTML file, open directly in a browser or serve statically.
- When editing `index.html`, keep the "one file" constraint unless asked otherwise — don't introduce a build step or split into modules unprompted.
- Don't wire up Supabase/backend calls here unless explicitly asked — that's intentionally a separate, later phase per `phase-0-prompt.md`.
