# Marketing Workload — CEO Dashboard · Session Handoff

Single-file web app for tracking the marketing team's workload. This doc lets a
fresh session pick up instantly. Current version: **v20260623p** (`version-A` tag is at v20260623n).

## Where everything lives

- **App:** `Marketing Workload — CEO Dashboard.html` — ONE file, inline CSS + JS,
  Chart.js. Edit this file directly. Version string is an HTML comment on **line 2**
  (`<!-- v2026MMDDx -->`); bump it on every change.
- **Repo:** `git@github.com:Chawalitamo/Dashboard-Workload-by-Suyato.git`, branch
  `main`. **PUBLIC** repo, served via GitHub Pages. Never commit private data.
- **Backend:** Supabase project `fxmigukayxvieziseqil`.
- **Local preview:** `.claude/launch.json` → `python3 -m http.server 8765`.

## Workflow conventions

1. Make the edit in the HTML file.
2. Bump the version comment on line 2 (e.g. `v20260623n` → `v20260623o`; if the
   date changed, use the new date + `a`).
3. `git add`, commit (Co-Authored-By trailer), `git push`. Changes go live on
   GitHub Pages after a hard refresh (Cmd/Ctrl+Shift+R). The page sets no-cache
   headers but the CDN can lag a minute.

## Supabase

- **Tables:** `mkt_tasks`, `mkt_projects`, `mkt_owners`, `mkt_settings`,
  `mkt_activity_log`. Storage bucket: `task-files`.
- `mkt_tasks` key columns: `id` (uuid), `title`, `owner`, `lead`, `project`,
  `stage`, `due` (date), `note`, `attachments` (jsonb), `created_at`.
- **RLS:** `mkt_tasks` has permissive `{public}` policies (allow all).
- **Edge functions:** `setup-auth-users`, `notify-overdue`, `notify-new-task`,
  `daily-summary`, `notify-teams`.
- **GitHub Actions:** `.github/workflows/notify-overdue.yml` and `daily-summary.yml`
  run cron `0 3 * * *` (= 10:00 Asia/Bangkok) and POST to the edge functions.

## SECURITY (must keep)

- The Resend API key is embedded in the `notify-overdue` edge function. **Do NOT
  expose it elsewhere or log it.** The Supabase **anon** key is in the HTML by
  design (public, protected by RLS) — that's fine. Never put the **service_role**
  key in the HTML or the public repo.

## Domain model & naming gotcha

- **PIC = "Person in charge" = `owner` DB field.** This is who the task is assigned to.
- The UI label **"Owner" = `lead` DB field** (confusing, but that's the mapping).
- `EMAIL_TO_NAME` maps each `@cjmart.co.th` email → display name (e.g.
  `warut.ka@cjmart.co.th` → `Ton`). `_currentUser` holds the display name.

## Auth & sessions

- Email/password login. Default password `123456789`, forces change on first login
  (`user_metadata.must_change_password`). 24-hour session via `mkt_login_time`
  localStorage + Supabase `persistSession`.

## Access control

- `ADMIN_USERS = ['Sun','Por']` → **Super Administrators**, see ALL tasks + the
  **Activity Log** tab. Everyone else sees only tasks where they are `owner` OR
  `lead` (`applyUserFilter`), and the Activity Log tab is hidden from them.

## New-brief notification (in-app pop-up)

- **Acknowledge-based, poll-driven** (NOT realtime broadcast — that was unreliable).
  On each 30s `loadFromSupabase`, `checkForNewAssignedTasks()` finds tasks where
  `owner === _currentUser`, created within the last 7 days, not yet acknowledged.
- Pop-up is a **centered modal** that is **permanent** until the PIC clicks
  **Acknowledge** (`dismissToast` → `_ackPending`) or **Go to Task** (`toastViewTask`
  → opens Kanban, clears filters, highlights the card). Acknowledged IDs persist in
  `localStorage['mkt_ack_<user>']`. Creators are auto-acknowledged for their own briefs.

## PIC Calendar (timeline)

- Tab between Kanban and Activity Log. `renderCalendar()` draws a Gantt-style
  timeline grouped by PIC (`owner`): each ongoing task (not Complete, has both
  `created_at` and `due`) is a bar from **brief date → deadline**. Bar colour =
  project; overdue = red outline; On Hold = dimmed; gold line = today. Click a bar
  → `openEdit`.

## Date inputs (flatpickr) & the Buddhist-year gotcha

- All due-date fields are **flatpickr** click-only calendar dropdowns (CDN), not
  typed inputs — Thai-locale browsers were saving Buddhist years (e.g. `2569`
  instead of `2026`), which broke the calendar (bars spanned ~543 years).
- `_normDue(v)` is a backstop on every save path: any year > 2200 → subtract 543.
- `_initDP()` (re)initialises pickers; called on load + after `renderList()`.
  `_setDP(el,val)` sets a picker's value programmatically (use instead of `.value`).

## Kanban specifics

- Cards have `data-id="<uuid>"`; drag/drop + actions wired in `setupDragDrop()`.
- Note URLs render as **favicon link chips** (`noteHTML`, `_linkFavicon`,
  `_linkLabel`) — favicons via Google's `s2/favicons` service.
- **Gotcha that bit us:** task `id` is a UUID. Inline `on*` handlers MUST quote it,
  e.g. `slideProgress('${t.id}',...)`. Unquoted → "Invalid or unexpected token".

## Backups

- `backups/version-A/*.json` — full table export taken 2026-06-23 (gitignored,
  local only). See `backups/version-A/MANIFEST.md` for counts + restore steps.
- Code snapshot: git tag **`version-A`** (`git checkout version-A`).

## Possible next steps / open items

- Email notifications to PICs (Resend domain `cjmart.co.th` was being verified in
  Resend — DNS records need to be added in Squarespace by the user; not finished).
- Activity Log is unfiltered beyond the admin-only gate (shows all tasks to admins).
- Task List note URLs are still plain (chips only added to Kanban so far).
