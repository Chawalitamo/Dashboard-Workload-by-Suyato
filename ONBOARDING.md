# Marketing Workload — CEO Dashboard · Project Handover

Single-file web app tracking the CJx marketing team's workload. This document is
written so a **fresh Claude session on any account** can pick the project up and
ship a change safely on day one.

- **Live app:** https://chawalitamo.github.io/Dashboard-Workload-by-Suyato/
- **Repo:** `github.com/Chawalitamo/Dashboard-Workload-by-Suyato` — **PUBLIC**
- **Supabase project ref:** `fxmigukayxvieziseqil`
- **Current version:** `v20260819a`
- **Owner:** Chawalit (display name **Sun**), CJx marketing
- **Last updated:** 2026-08-19

---

## 1. Read this before you touch anything

These are the rules that have caused real damage when broken.

1. **The repo is PUBLIC.** Never commit task data, database dumps, backups, or
   staff email addresses. GitHub Actions artifacts on a public repo are publicly
   downloadable too — don't put data in them either.
2. **Never put the Supabase `service_role` key in the HTML or the repo.** The
   **anon** key is in the page on purpose; it is safe because RLS is enforced.
3. **Staff emails must not appear in the page source.** They live in
   `mkt_owners`, readable only by authenticated users. There used to be a
   hardcoded `EMAIL_TO_NAME` map in the HTML; it was removed deliberately.
4. **`mkt_chat` is deliberately excluded from the emailed backup** — the dump
   leaves the system by email, and private messages shouldn't.
5. **Development branch:** `claude/status-check-n7uts3`. Push there, open a PR,
   squash-merge. Don't push to `main`.
6. **Never delete storage objects to "clean up".** Removing an attachment in the
   UI unlinks it and leaves the file, on purpose — unlinking is reversible,
   deletion is not.

---

## 2. Layout

```
Marketing Workload — CEO Dashboard.html   ← THE app. One file. Edit this.
index.html                                ← redirect, preserves auth hash/query
Marketing Workload — CEO Dashboard_files/ ← vendored chart.umd.min.js, tabler-icons
.github/workflows/                        ← daily-summary, notify-overdue, daily-backup
ONBOARDING.md                             ← this file
```

The app is **one HTML file**: inline `<style>`, one large inline `<script>`
(~line 1290 to ~5175), plus a small `<script type="module">` that boots the
Supabase client. There is no build step and no framework.

---

## 3. Deploy procedure — follow it exactly

Getting this wrong means users sit on a stale app or see a broken one.

1. Edit the HTML.
2. **Bump the version in two places** (they must match):
   - line 2: `<!-- v20260819a -->`
   - `const APP_VERSION='v20260819a'` (~line 1292)
   - Format `vYYYYMMDD` + a letter; same day → next letter, new day → `a`.
3. Commit, push to `claude/status-check-n7uts3`, open a PR, **squash-merge**.
4. **Wait for the Pages deploy, then wait a few more minutes** for the CDN.
   Check the build:
   ```
   curl -sS -H "Authorization: Bearer $GH_TOKEN" \
     "https://api.github.com/repos/Chawalitamo/Dashboard-Workload-by-Suyato/actions/runs?per_page=1"
   ```
   Total wait from merge to safe: **~7–9 minutes**.
5. **Only then** flip the flag that reloads everyone's open tabs:
   ```sql
   update mkt_settings set value='v20260819a' where key='app_version';
   ```
   `loadFromSupabase()` polls this; a mismatch triggers a one-shot reload,
   guarded by `_verReload` in `sessionStorage` so it can't loop.

**If you flip the flag before the CDN has the new file, tabs reload into the old
version.** Order matters.

### Branch hygiene gotcha

Because PRs are **squash**-merged, the dev branch keeps the pre-squash commit and
the next PR conflicts. Fix:

```bash
git fetch origin main
git rebase --onto origin/main <last-merged-commit> claude/status-check-n7uts3
git push --force-with-lease -u origin claude/status-check-n7uts3
```

Force-with-lease is fine **only** when the discarded history is already merged.

---

## 4. Testing — how this project is actually verified

There is no test runner in the repo. Tests live in the session scratchpad and are
rebuilt as needed. The pattern is worth keeping because it has caught many real
bugs:

**A. Node DOM-stub harness.** `harness.js` fakes `document`, `localStorage`,
`Chart`, etc. Extract the inline script and `eval` it, exposing internals:

```bash
# recompute the line numbers each time — they move
grep -n "^<script>$" "Marketing Workload — CEO Dashboard.html" | tail -1
grep -n "^</script>$" "Marketing Workload — CEO Dashboard.html" | tail -1
sed -n '1291,5175p' "Marketing Workload — CEO Dashboard.html" > inline.js
node --check inline.js
```

```js
const setters = `global.__renderKanban=renderKanban; global.__set=(t,u)=>{tasks=t;_currentUser=u;};`;
eval(fs.readFileSync('./inline.js','utf8') + setters);
```

**B. Real Chromium** via `playwright-core` with
`executablePath: '/opt/pw-browsers/chromium'` — for anything involving layout,
canvas, or computed styles. Load with `page.goto('file://' + FILE)`.

Test suites written so far (t7 themes, t8 board filter, t9–t13 NSG, t14 login
pop-up, t15 storage/bandwidth). **Make fixtures relative to today** — t9 rotted
because it hardcoded dates that the calendar moved past.

### Lessons that cost real time

- **Malformed CSS is silent.** `--gold:#f0b councils` is dropped by the browser
  with no error. There is a validator assertion for this in t7 — keep it.
- **A `"` inside `style="…"` truncates the attribute.** Use single quotes for
  `url('…')` in inline styles.
- **`getComputedStyle(el,'::-webkit-slider-thumb')` returns the host element's
  box in Chromium.** It cannot measure pseudo-element size — screenshot and
  inspect pixels instead.
- **`preserveAspectRatio="none"`** stretched SVG circles into ovals.
- **Storage keys are case-sensitive.** `Lucifer.jpg` ≠ `lucifer.jpg`, and a
  wrong wallpaper filename shows no picture and no error. Cost three rounds.
- **Prove a new guard fails when broken** before trusting it.

---

## 5. Supabase

### Tables (all RLS-enabled)

| Table | Rows | Notes |
|---|---|---|
| `mkt_tasks` | 459 | the work items |
| `mkt_activity_log` | 2179 | audit trail |
| `mkt_worklog` | 265 | per-user active seconds per day |
| `mkt_nsg_stores` | 21 | new-store timeline, **admin-only** |
| `mkt_owners` | 15 | name, email, avatar, theme, last_login |
| `mkt_projects` | 5 | |
| `mkt_settings` | 4 | key/value |
| `mkt_chat` | 38 | team chat + DMs |
| `mkt_backup_runs` | 16 | backup audit; RLS on, **no policies** = nobody reads it |
| `team_schedules`, `app_assets` | | |

`mkt_tasks` columns: `id uuid, title, owner, project, stage, due date, note,
sort_order, created_at, updated_at, attachments jsonb, lead, proof jsonb,
pics jsonb, deleted_at, deleted_by, deleted_batch`.

`mkt_settings` keys: `app_version`, `deadline_warning_days`, `nsg_sheet_url`,
`teams_webhook_url`.

### Naming trap — learn this or you will write the wrong field

- **PIC = "Person in charge"** → historically the `owner` column, now the
  `pics` jsonb **array** (multi-PIC). `owner` is kept as the primary PIC.
- The UI label **"Owner"** maps to the **`lead`** column. Yes, it's inverted.
- Helpers: `_PICS(t)`, `_isPIC(t,name)`, `_picsLabel(t)`. Use them; don't read
  `owner` directly.

### Access control

`ADMIN_USERS = ['Sun','Por']` are **Super Admins**. They see all tasks plus the
Activity Log, Work Log, and NSG tabs. Everyone else sees tasks where they are a
PIC or the lead.

**Hiding a tab is not security** — the anon key ships in the page. NSG is
enforced server-side: policy `nsg_admin_all` on `mkt_nsg_stores` uses
`mkt_me() in ('Sun','Por')`. This was verified by proving anon gets denied and a
non-admin authenticated user reads 0 rows.

### Functions (all `SECURITY DEFINER`, revoked from anon/authenticated)

`mkt_me()`, `mkt_run_backup(p_reason)`, `mkt_backup_guard()`,
`mkt_backup_collect()`, `mkt_touch_worklog()`, `rls_auto_enable()`.

### Storage buckets

| Bucket | Public | Contents |
|---|---|---|
| `task-files` | yes | attachments, proof, chat files, `*.thumb.webp` |
| `wallpapers` | yes | 18 theme backgrounds |
| `db-backups` | no | nightly dumps |
| `app`, `Calendar_MKT Production` | | small assets |

`task-files` allowed MIME types include `image/webp` — **required**, or every
thumbnail upload is rejected.

### Edge functions

`notify-overdue`, `notify-new-task`, `notify-teams`, `notify-password-reset`,
`add-to-calendar`, `daily-summary`, `daily-backup`.

The **Resend API key is embedded in the notify functions** — do not log or copy
it. Note: `get_edge_function` / `deploy_edge_function` MCP tools have been
blocked in these sessions, so edge function source could not be read or updated
from the agent. Changes there need the Supabase dashboard or CLI.

### Scheduling — pg_cron, not GitHub Actions

GitHub Actions cron **failed silently** (runner never allocated: `runner_id: 0`,
no logs, cancelled after 15 min queued). Backups were moved into the database:

| Job | Schedule (UTC) | Command |
|---|---|---|
| `daily-backup-primary` | `0 23 * * *` | `mkt_run_backup('scheduled')` |
| `daily-backup-guard-1` | `30 0 * * *` | `mkt_backup_guard()` |
| `daily-backup-guard-2` | `30 2 * * *` | `mkt_backup_guard()` |
| `daily-backup-collect` | `20,50 23,0,2 * * *` | `mkt_backup_collect()` |

`.github/workflows/daily-backup.yml` is now `workflow_dispatch` only.

Two traps found the hard way: **pg_net's default timeout is 5 s** (set
`timeout_milliseconds := 300000`), and **`net._http_response` is purged after
6 hours**, so status had to be copied into durable columns by
`mkt_backup_collect()`. The guard window is anchored to 05:30 Bangkok rather
than being a rolling 20 hours, so an ad-hoc evening backup can't mask a failure.

---

## 6. Feature map

- **CEO Dashboard** — Chart.js charts, per-PIC aggregation, PDF export.
- **Task List** — inline edit; progress slider whose handle is a ☀️ drawn with a
  CSS `mask` (`--sun-mask`), coloured by `--gold`.
- **Kanban Board** — 7 stages: Job Brief, Design, Production, Review, Approval
  Stage, On Hold, Complete. Drag-and-drop. Opens filtered to **your own** tasks;
  Super Admins open on **All members** (`applyDefaultBoardFilter`). The 🏆 button
  jumps a card straight to Complete after a confirm. The Complete column renders
  its **newest 20** with a "show more"; the header shows the true total.
- **PIC Calendar** — Gantt bars from brief date to deadline, coloured by project.
- **Recycle Bin** — soft delete via `deleted_at`/`deleted_by`/`deleted_batch`,
  restorable. Anyone can clear it (decided deliberately — "stick with the same
  thing"), not admin-only.
- **Activity Log / Work Log** — Super Admin only.
- **NSG** — Super Admin only. New-store timeline for **MKT ซัน** rows, synced
  from a public Google Sheet, 5 phases / 25 checklist items, 11 free-text fields,
  KPI tiles with drill-down, a management overview (pure SVG, no Chart.js), and a
  login pop-up listing stores opening within 7 days.
- **Themes** — 23 options (6 colour + 17 wallpaper), per-user, synced via
  `mkt_owners.theme`.
- **Team chat** — town hall + DMs, stickers, file/photo attachments.

### NSG specifics worth knowing

- `storecode` is the **primary key**, and two sheet rows have a blank store code.
  That's why the list once showed 19 of 21. Blank codes get a placeholder key
  `'__NOCODE__' + name`; `_nsgAdoptPlaceholders()` migrates the data onto the
  real row when a code finally appears, existing values winning.
- **Import and sync deliberately never send `checks` or `notes_items`**, so
  re-syncing the sheet can never wipe someone's ticked boxes.
- `fetchNSGSheet()` explicitly detects Google's login HTML. Without that, a
  sheet that stopped being public would parse as "empty" and blank all 21 stores.
- `_nsgNormMkt()` strips NBSP / zero-width / BOM before matching; `_nsgIsSun()`
  also accepts shared cells like "ซัน/จิ๋ว".

### Charts

Dashboard charts use **Chart.js** (vendored locally). The NSG overview is
**hand-written SVG on purpose** — a not-yet-loaded Chart.js previously left a
permanently stuck skeleton.

---

## 7. Bandwidth / egress — recently fixed, keep it that way

Supabase issued a Fair Use warning (egress over 5 GB; grace to 18 Sep 2026).
Cause: `thumbsHTML()` rendered full-size originals (avg 1.3 MB, max 15 MB) into
54×54 px tiles, the Complete column rendered every card, and uploads used the
default `max-age=3600`.

Fix shipped in `v20260819a`:

- Every uploaded image gets a **400 px WebP** companion via `makeThumb()`;
  `uploadOne()` stores both and records `thumb` on the attachment. Measured
  752 kB → 25.7 kB.
- All uploads pass `cacheControl: CACHE_FOREVER` (`'31536000'`). Paths carry a
  `Date.now()` prefix and are never rewritten, so a year is safe.
- Complete column capped at 20.
- **Profile → Storage & bandwidth** (Super Admin) has *Generate missing
  thumbnails* and *Re-compress wallpapers*, both resumable.

Result: board images went from 339 MB of originals to **4.0 MB of thumbnails**
(98.8% less); wallpapers 4,353 kB → 1,004 kB. Nothing was deleted.

**Rules to preserve:**

- Any new `.upload()` call **must** pass `cacheControl: CACHE_FOREVER`. t15 has a
  source-level guard that fails and names the offending call site if you forget.
- Never render a full-size original into a small box. Use `thumbUrl(a)`, which
  falls back to the original when no thumbnail exists.
- Supabase **image transformations** (`/render/image/public/...?width=`) are a
  **Pro-plan** feature and are NOT available here. Hence the client-side canvas
  approach.

---

## 8. Environment notes for the agent

- **Direct network access to `*.supabase.co` is blocked** by the sandbox proxy
  (403 on CONNECT). Use the Supabase MCP tools for SQL. Anything needing to
  download or upload storage objects **must run in the user's browser** — that's
  why the backfill is a button, not a script.
- `chawalitamo.github.io` and `docs.google.com` are also blocked, so the
  deployed page cannot be fetched to verify. Rely on the Pages build status.
- `api.github.com` **does** work via curl with `$GH_TOKEN`.
- Chromium is at `/opt/pw-browsers/chromium`; do not run `playwright install`.
- No `sharp`, and the bundled `ffmpeg` has **no WebP encoder** — use Chromium's
  canvas for image work.
- MCP tools observed as blocked/needing approval: `get_edge_function`,
  `deploy_edge_function`, `list_extensions`, `get_organization`,
  `list_edge_functions`, `Google_Drive__read_file_content`.

---

## 9. Open items

- **6 recovered tasks still lack a PIC and/or due date** (ปรับ AW ไทยช่วยไทย,
  สินค้าขายดีประจำสัปดาห์, 2× สื่อเปิดสาขาใหม่ Pack GO, สื่อเปิดสาขาใหม่ Pack GO
  ประจำเดือน มิถุนายน, บอร์ดใส่ป้ายราคาแลคน้ำ). The user deprioritised these
  ("ไม่ต้องรอ"). **Do not invent values** — ask.
- **2 images still have no thumbnail** — `Presentation.png` and
  `Presentation2.png` on the "MKT Presentation" card (~4.9 MB). Both files are
  intact; the run failed transiently. Pressing the button again retries them.
- **`autumn.jpg`** was skipped by the wallpaper pass (WebP came out no smaller —
  the guard working correctly), so it keeps a 1-hour cache. 164 kB, ignorable.
- **Supabase Auth Site URL / Redirect URLs** should point at
  `https://chawalitamo.github.io/Dashboard-Workload-by-Suyato/`. Long-standing,
  never confirmed done.
- **Resend domain `cjmart.co.th`** verification needs DNS records added in
  Squarespace by the user; unfinished.
- `teams_webhook_url` in `mkt_settings` is **empty**, so `postToTeams()` is a
  no-op until it's filled in.

---

## 10. Working style the user expects

- Thai and English are both used; reply in whichever the user wrote in.
- The user ships often and checks the live site immediately — **always complete
  the full deploy including the version flip**, then say plainly that it's live.
- Diagnose with evidence (query the DB, screenshot the browser) rather than
  agreeing. Several requests were premised on a wrong cause — 19-vs-21 stores,
  KPI tiles "not summing", "delete attachments to cut bandwidth" — and the
  correct move each time was to check first and explain what was really
  happening.
- Keep code comments in the surrounding style: explain **why**, especially where
  behaviour looks odd on purpose.
