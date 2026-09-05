# Paperbark Farm — instructions for Claude

Paperbark Farm is a dog-boarding business in NSW, Australia. This project
replaces its Podio workspace with a Supabase database, a staff dashboard, a
read-only MCP server, and (later) a customer booking portal. Staff use the
dashboard daily, often on a phone, to see who is arriving, who is on site,
who owes money, and which dogs are missing vaccination or waiver paperwork.

Lines marked `TODO(nathan)` are gaps Claude could not fill from the code.
Edit or delete them.

## Where things live

| Repo | Role |
|---|---|
| `papaerbark2030-dev/paperbark-podio-migration` (private) | **The real project.** Dashboard, MCP server, nightly sync, schema, migration tooling. Deploys to Vercel at `paperbark-podio-migration.vercel.app`. |
| `papaerbark2030-dev/paperbark-dashboard` (public) | Retired GitHub Pages site. `index.html` is only a redirect to the Vercel dashboard so old bookmarks work. Do not add features here. |

Supabase project: `ekcbynduceipgcqvtpyf`. Postgres runs in UTC.

## Non-negotiables

These protect live customer data and a business in the middle of a
migration. Do not relax them without an explicit instruction in the task.

1. **Bookings are never deleted.** Cancelling sets `status = 'Cancelled'`.
   The `no_double_booking` exclusion constraint then frees the dates.
2. **Nothing writes to Podio.** The sync's only Podio calls are the app-auth
   token POST and `/item/app/{id}/filter/` reads. Podio remains source of
   truth until cutover. `TODO(nathan): confirm cutover status as of now.`
3. **`SUPABASE_SERVICE_ROLE_KEY` is server-side only.** It is used by
   `api/_lib/db.js` and the MCP server. Never import those from client code,
   never put the key in `dashboard/` or `shared/`. The dashboard uses the
   publishable key and relies on RLS via `is_staff()`.
4. **Customer PII never enters git.** `raw/`, `transformed/`, `load/`,
   `transform_report.txt` and `extract.log` are gitignored and
   vercelignored. Do not paste real owner names, phones, emails or
   addresses into commits, tests, fixtures, PR descriptions or comments.
5. **Schema changes are numbered SQL migration files at the repo root**, not
   ad-hoc edits through a client. Write the file, explain it, and let a
   human apply it. `TODO(nathan): may Claude apply migrations directly via the
   Supabase MCP tools, or only write the SQL file?`
6. **Do not "fix" the cron drift.** `0 14 * * *` UTC is midnight AEST and
   1am during NSW daylight saving. That drift is accepted on purpose.
7. **Do not revoke EXECUTE on `is_staff()`.** Every `staff_all` RLS policy
   evaluates it. The residual Supabase advisor warnings for it are expected.
8. **Do not filter `is_deleted` in the views or dashboard yet.** The calendar
   integrations and the Telegram morning briefing read those views and the
   change is pending sign-off. `TODO(nathan): has this been decided?`

## Architecture map (podio-migration repo)

| Path | What it is |
|---|---|
| `dashboard/index.html` | Single-file staff dashboard. Vanilla JS, no build step, no framework. Tabs: Today, Pipeline, Calendar (month), Clients, Money, Compliance. Booking modal, full data entry. |
| `dashboard/supabase.js` | Self-hosted copy of supabase-js UMD. Do not swap back to a CDN. |
| `shared/paperbark-data.js` | **All validation and write logic.** UMD module (`window.PaperbarkData`, also CommonJS). No DOM, no global client. Will be reused by the customer portal. |
| `api/mcp.js`, `mcp/` | `paperbark-mcp`: read-only MCP server, 8 tools, OAuth 2.1 via WorkOS AuthKit, `data_as_of` on every response. See `mcp/README.md`. |
| `api/sync-podio.js`, `api/_lib/` | Nightly one-way Podio to Supabase sync. Cron-secret guarded, 40s soft time budget, checkpoints to `sync_state` after every page, weekly soft-delete sweep. |
| `api/oauth-*.js` | RFC 9728 / OAuth discovery documents that point MCP clients at AuthKit. Issue nothing, hold no keys. |
| `paperbark-farm-supabase-schema.sql`, `00N_*.sql` | Schema and migrations. Note two files share the `005` prefix (`005_security_hardening.sql`, `005_sync_state.sql`); the next migration is `006`. |
| `transform.py` | One-off migration transform (Podio JSON to load-ready entities). Hard-codes a Mac path and `TODAY = 2026-07-03`; historical, not run in CI. `api/_lib/mapping.js` is the maintained port. |
| `test/`, `mcp/test/` | `node --test` suites. No network. |

## Commands

```sh
npm test                          # 55 tests, node --test, no network, under 1s
python3 -m http.server            # serve the REPO ROOT so dashboard/ can load ../shared/
node mcp/scripts/check-setup.mjs  # verify MCP/AuthKit env
```

Requires Node 22 or newer. There is no linter, formatter, or bundler
configured. `TODO(nathan): do you want one added, or keep it tool-free?`

Vercel functions read env vars listed in `.env.example`. Every variable the
code reads is listed there; keep it that way when adding one.

## Conventions

- **Australian English** in prose, comments and UI copy: authorisation,
  colour, enquiry, suburb, postcode. `TODO(nathan): confirm.`
- **Dates are `YYYY-MM-DD` strings** everywhere. "Today" is today in
  `FARM_TIMEZONE` (default `Australia/Sydney`), resolved in JS and passed
  explicitly. Never rely on Postgres `CURRENT_DATE`.
- **Validation functions are pure** and return `{ ok, errors }` with
  `errors` keyed by field name. **Data functions** take the supabase client
  as their first argument and return supabase-style `{ data, error }` with a
  user-friendly `error.message` and the raw error on `error.cause`.
- **Podio fields are pulled by `external_id`**, never by position.
- **Staff writes** stamp `bookings.created_via = 'admin'`; the customer
  portal must pass `'client_portal'`. RLS separates the channels.
- **Upserts key on `podio_item_id`.** Dashboard-created rows have it null
  and are never touched by the sync. Dashboard edits to a mirrored row are
  overwritten at the next sync; this is a known consequence of the parallel
  run, not a bug to fix silently.
- **Sync order matters**: dogs, then bookings (with `booking_dogs`
  reconciliation), then transport_runs, date_entries, documents. Owners are
  not synced because `owners` has no `podio_item_id`.
- **Files open with a block comment** saying what the file is for and the
  one or two decisions a reader would otherwise get wrong. Match that.
- **Commits** are imperative one-liners with a body when the why is not
  obvious. Branches merge via PR (`Merge pull request #N: <title>`).
  `TODO(nathan): PR required for everything, or may Claude push small fixes
  straight to main?`

## Data model vocabulary

Tables: `owners`, `dogs`, `dog_bonds`, `bookings`, `booking_dogs`,
`transport_runs`, `date_entries`, `date_entry_dogs`, `documents`, `waivers`,
`waiver_acceptances`, `projects`, `staff_users`, `sync_state`.

Views the dashboard, MCP, calendar integrations and Telegram briefing read:
`arrivals_today`, `departures_today`, `dogs_on_site_today`,
`bookings_with_balance_due`, `dogs_with_compliance_gaps` (14-day horizon),
`documents_expiring_soon` (30-day horizon).

Booking pipeline, in order: Enquiry, Trial Weekend, Availability Pending,
Availability Confirmed, Deposit Requested, Payment Pending, Paid, Booked,
In Stay, Checked Out, Follow-up, Cancelled. The dashboard displays a Booked
booking whose dates say the dog is on the farm as "In Stay" without changing
the stored status.

Document types: Vaccination Record, Signed Waiver, Contract, Insurance,
Other. Document statuses: missing, pending_review, approved, rejected,
expired.

Auth: dashboard uses Supabase email/password; MCP uses AuthKit bearer
tokens. Both resolve staff status from `staff_users` by email. There is no
second allowlist.

## How to work

- Read `README.md` and `mcp/README.md` in the podio-migration repo before
  touching the sync or MCP. They are current and detailed.
- Run `npm test` before every push. Dashboard changes have no automated
  tests; describe how you verified them. `TODO(nathan): do you want Playwright
  smoke tests for the dashboard?`
- When a task touches the schema, the sync and the dashboard together, do
  the migration first, then `shared/paperbark-data.js`, then the UI.
- Prefer small, reviewable diffs. The dashboard is one 1,000-line file;
  keep changes local and do not reformat unrelated sections.
- Check `sync_state` for per-app status and notes before reading Vercel logs
  when diagnosing a sync failure.
- If something in the code contradicts this file, say so and ask which is
  right rather than picking one.

## Open items Claude should know about

- Owners are not synced from Podio; new Podio dogs land on the
  `Unknown owner (migration)` placeholder.
- Podio GlobiFlow webhooks on New Booking and Dog Profiles must be replaced
  before Podio is retired.
- Next phases: customer portal (waiver acceptance, vaccination uploads).
- `TODO(nathan): where do the Telegram briefing and calendar integrations
  live? They are not in either repo.`
- `TODO(nathan): who else uses the dashboard and on what devices? This shapes
  UI and accessibility choices.`
