# Lockers — N.Seattle.Housing.Homeless.SHARE.Lockers

A vanilla JS, single-file Sage_OS dictionary app for managing a shelter's locker
program: participant records, locker assignments, bars, security check-in/out,
staff users, and reports. Backend is Supabase (Postgres + Auth + REST).

## Where things live

- **Main app file**: `index.html` — this is the ONLY file that matters. Everything
  else in this folder (`Lockers 1.0.html`, `lockers 2.7.html`, `lockers backup
  6-9-25.html`, etc.) is an old version snapshot, not in active use. Don't edit them.
- **GitHub**: `https://github.com/jwolff617/lockers.git` — local branch is `master`,
  remote default branch is `main`. Push with `git push origin master:main`.
- **Live site**: `https://seattlelockers.vercel.app` (auto-deploys from GitHub `main`
  via Vercel).
- **Supabase project**: `https://aqwqugxxxxgrsgtiqwxn.supabase.co` (anon key is
  hardcoded in `index.html` as `SUPA_KEY` — this is expected/normal for an anon key,
  but it means **all data security depends on RLS policies being correctly set on
  every table**).

## Architecture in one paragraph

Single HTML file: CSS in a `<style>` block, all JS in one `<script>` at the bottom.
A "Sage_OS"-style nested dictionary (`CONTENTS` → `DICT` categories → notes →
details) defines the navigable structure. `S.path` is the current location (array
of keys), `nav(path)` moves there, `render()` redraws everything. Screens are
registered with `regScreen('Cat.note.detail', renderFn)` into a `SCREENS` map; if a
path has no registered screen but has children, a clickable nav-card grid renders
instead (added 2026-06-15). `S.data` holds all loaded records; `S.user` holds the
logged-in user `{id, name, role, access_token}`. `supa(table, method, body, query,
prefer)` is the one function that talks to Supabase REST; `db.select/upsert/delete`
are thin wrappers around it.

## Roles

- `ec` — Executive Coordinator (full access)
- `clerk` — day-to-day staff (full access, same areas as EC)
- `security` (stored as `'security'` in DB, code accepts `'sec'` as an alias too)
- `part` — Participant / guest, shared password login (not a real Supabase Auth
  account). Category now labeled **"Participant Log In"** (renamed 2026-06-15 to be
  unambiguous from "Participants" management screens).

## Auth model (important, was rebuilt from scratch this project)

- Staff (`ec`/`clerk`/`security`) use **real Supabase Auth** accounts.
  Username-only accounts fake an email as `username@lockers.internal`.
  `get_staff_email(p_identifier)` is a SECURITY DEFINER RPC that resolves a
  username or email to the real auth email; there's also a client-side fallback
  if the RPC returns null.
- Role/name come from the `sl_users` table, matched by email (or
  `username@lockers.internal`) against the Supabase Auth session's email — NOT
  from a password column (there is no password column in `sl_users`).
- Participant login is a single shared password stored in `sl_config` under key
  `part.password` — not a real account.
- **Persistent login (added 2026-06-15)**: after login, the Supabase refresh token
  + user info are saved to `localStorage`. On page load, `tryRestoreSession()`
  exchanges the refresh token for a fresh access token, so refreshing the browser
  keeps you logged in. Participants are restored directly from localStorage (no
  token needed). Logout clears localStorage.

## sl_config table

Key-value store, replaced an earlier broken `sl_setup` approach:
```sql
CREATE TABLE sl_config (key text PRIMARY KEY, val text NOT NULL DEFAULT '', updated_at timestamptz DEFAULT now());
```
Used for: `info.*` pages (editable info content, e.g. `info.about`) and
`part.password` (the shared participant login password). Upserts must use
`?on_conflict=key` (not `id` — `db.upsert()`'s default `on_conflict=id` will fail
on this table).

## SQL that may still need to be run (check before assuming done)

```sql
-- sl_config table + seed
CREATE TABLE IF NOT EXISTS sl_config (key text PRIMARY KEY, val text NOT NULL DEFAULT '', updated_at timestamptz DEFAULT now());

-- sl_users needs email/auth_id columns
ALTER TABLE sl_users ADD COLUMN IF NOT EXISTS email text;
ALTER TABLE sl_users ADD COLUMN IF NOT EXISTS auth_id text;

-- staff email resolver RPC
CREATE OR REPLACE FUNCTION get_staff_email(p_identifier text) RETURNS text
LANGUAGE sql SECURITY DEFINER AS $$
  SELECT COALESCE(email, username || '@lockers.internal') FROM sl_users
  WHERE lower(email) = lower(p_identifier) OR lower(username) = lower(p_identifier)
  LIMIT 1;
$$;

-- sl_checkins needs a ts column for accurate session timer persistence
ALTER TABLE sl_checkins ADD COLUMN IF NOT EXISTS ts timestamptz;
```
If session timers show wrong elapsed time after a browser refresh, the `ts` column
above is the likely cause — confirm it exists in Supabase.

## Known bugs not yet fixed (full list from a 2026-06-15 static-analysis pass)

A subagent did a thorough line-by-line review and found 37 issues. The 8
HIGH-severity ones (data corruption / security gaps) — **not yet fixed as of
2026-06-16**:

1. Participant `my_status` screen checks only `p.barred` flag, not the
   `p.bars[]` array — can show "Good Standing" for someone with an active bar if
   the flag wasn't set (e.g. via the `issueBarFromRec` bug below).
2. `saveParticipantForm`: releasing old lockers calls `saveLocker()` without
   `await` — can desync DB if the upsert fails after the participant record saves.
3. New participant form can silently double-assign a locker number already held by
   another participant — no check before overwriting `locker.participantId`.
4. `issueBarFromRec()` creates the bar with `type:'temporary', endDate:null` —
   this combination means `renderSecParticipantCard`'s active-bar filter
   (`type==='permanent' || endDate>=today`) never matches, so the security
   check-in screen shows "In Good Standing" for someone who IS barred
   (`p.barred` is true but no bar shows in the alert). The check-in button is
   still correctly disabled because that uses `p.barred` directly — but the
   displayed status text is wrong/contradictory.
5. Toggling a broken locker back to "working" sets `status='available'` even if a
   participant is still assigned to it — should restore `status='assigned'`.
6. Bag & Tag locker release (`btReleaseLockers`) updates `S.data.lockers` in
   memory but never calls `saveLocker()` — released lockers revert to "assigned"
   on next page load.
7. `removeUser()` deletes the `sl_users` row but never deletes the Supabase Auth
   account — removed staff can still log in (just fail the `sl_users` lookup with
   "no staff record found", but their auth token still works for direct API
   calls).
8. `resetUserPw()` will happily send a password-reset email to an
   `@lockers.internal` placeholder address (it just checks `email.includes('@')`)
   — silently fails for username-only accounts with no real email.

There are ~13 more MEDIUM and ~16 LOW severity issues cataloged (wrong report math,
missing guards on empty search, asymmetric mandatory-shift removal, EC seeing less
report detail than Clerk, etc.) — ask Claude to re-run the static analysis or check
the prior conversation transcript if you want the full list re-derived; it wasn't
saved verbatim to this file to avoid bloat, but the categories above are the worst
of it.

## Recent work (chronological, most recent first)

- **2026-06-16**: Wrote this CLAUDE.md for session handoff.
- **2026-06-15**: New participant form defaults "Renewed Through" to the 15th of
  next month (renewal-by-the-15th rule). Command bar overhaul: renamed `part` to
  "Participant Log In" so it's unambiguous from "Participants" management;
  `parseCommand` now filters category matches by `canSee()` (role visibility) and
  falls through to a global accessible-node search (`getAllNodes()`) so single
  words like "participant", "checkin", "renew" resolve correctly from anywhere,
  not just as a literal category name; multi-token commands also try every
  visible category root as a relative base. Suggestion dropdown shows path context
  for non-local matches and expands to 8 items when empty. Unregistered screens
  with children now render a clickable nav-card grid instead of "Coming Soon";
  true dead-end screens show a clickable parent link.
- **2026-06-15**: Persistent login across browser refresh via localStorage +
  Supabase refresh token (see Auth model above).
- **2026-06-15**: Session timers (security check-in/out) reconstructed from
  `sl_checkins.ts` on load so they survive a browser refresh with correct elapsed
  time; session timer screen auto-refreshes every 10s while open.
- **2026-06-15**: New participant defaults `renewedThru` to 15th of next month
  (date math avoids month-overflow by setting day=1 before adding a month).
- **2026-06-14**: Fixed check-in button breaking for participant names containing
  apostrophes (`esc()` doesn't escape `'`, which broke inline `onclick` JS) —
  `logCheckIn`/`logCheckOut` now take only a participant ID and look up the name
  internally. Added password-confirm field to new-staff-user form (plaintext, typed
  twice, validated to match before submit). Improved "In Good Standing" /
  barred-status logic on the security participant card to also check the
  `bars[]` array (not just the `barred` flag) and to show all active bars with
  reasons; disabled the Check In button for barred participants. Added a "Pending
  Bar Recommendations" list to the security Check In screen. Replaced the
  check-in `alert()` popup with an inline green success banner that auto-clears
  after 2.2s and refocuses the search box.
- **2026-06-14 (earlier)**: Navigation overhaul — fixed breadcrumb duplicating
  "Lockers", `parseCommand` rewritten with label matching + space-as-dot +
  deepest-valid-match walking, `getSuggestions` scored ranking, Ctrl+Left/Right
  now navigate to sibling nodes at the current tree level (not just up/into-first-
  child).
- **Earlier sessions**: Rebuilt auth from a broken password-column approach to
  real Supabase Auth + `sl_users` role lookup; replaced broken `sl_setup` with
  `sl_config` key-value table; fixed role constraint (`'security'` not `'sec'`);
  fixed Power Lunch report (live closures/opens/bars-issued instead of static
  placeholders); added bar list screen + live participant search on issue-bar
  form; fixed locker inventory to use participant `lockers[]` arrays as ground
  truth instead of stale `sl_lockers.status`.

## Workflow notes for whoever picks this up

- Always `git add index.html && git commit ... && git push origin master:main` —
  pushing to `main` (not `master`) is what triggers the Vercel deploy.
- The repo is full of old numbered snapshot files (`lockers 2.7.html`, `lockers
  3.6.html`, etc.) that are NOT part of the live app and shouldn't be edited or
  treated as current — they're leftover local backups. Only `index.html` is real.
- `git status` currently shows a `Core_Dictionary.md` deletion pending and several
  of those numbered HTML snapshots as untracked — these were already in this state
  before this session and haven't been investigated; don't assume they need
  cleanup unless asked.
