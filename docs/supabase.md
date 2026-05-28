# Supabase at Atram — how it's wired and how to work with it

Operational reference for the production Supabase that backs the Rada platform
(Kenya, Colombia, Ethiopia pilots). Read this **before** opening any DB
connection, writing a migration, or hand-rolling a connection string.

## Projects

Org: `zolhnojtkeyjkwapwpjt`. Confirm with `supabase projects list`.

| Ref                       | Display name                       | Role                                                                  |
| ------------------------- | ---------------------------------- | --------------------------------------------------------------------- |
| `fljikmhgsywkrmqckeaq`     | VoiceFlow_Kenya_Pilot              | **Production.** Shared across Kenya/Colombia/Ethiopia despite the legacy name — do not be misled by the "Kenya_Pilot" label. |
| `txfdwbilttroflxvgwur`     | Kenya Pilot DB Development Copy    | Dev/staging clone of prod for risky experiments.                       |
| `owtfvrvmcxhlpzdcicjp`     | weather_aggregation                | Weather-pipeline scratch project.                                      |
| `wfkpuvmoqwmryrunjncy`     | Rada-weather                       | Rada-weather ETL/scratch project.                                      |

When this doc says "Supabase" without a qualifier it means `fljikmhgsywkrmqckeaq`.

## Repos and their relationship to Supabase

| Repo                            | Role                                                                                                  |
| ------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `rada-message-system`            | **Canonical home for migrations** (`supabase/migrations/`). Composes & dispatches alerts; writes `scheduled_alerts`. |
| `alert-manager`                  | Vite/React frontend. Reads via the `meteops-api` Supabase Edge Function (`supabase/functions/meteops-api/`). |
| `yaya-engine`                    | Laravel app. **Writes** `users` and `alert_subscriptions` via PostgREST (`SupabaseService`). Has no DDL ability. |
| `rada-weather`                   | Python ETL. Writes `town_forecasts_historic`, `gauge_*`. Scheduled via GitHub Actions cron.            |
| `supabase-crons-and-functions`   | **Archived.** Do not add migrations or functions here. Canonical SQL lives in `rada-message-system`. The archived schema dump (`20250926094558_remote_schema.sql`) is still useful as a reference but is not authoritative. |

## Authentication — there are three layers, know which you need

### 1. PostgREST keys (REST/RPC only — no DDL)

Stored in `rada-message-system/.env`:

- `SUPABASE_URL` — `https://fljikmhgsywkrmqckeaq.supabase.co`
- `SUPABASE_KEY` — anon key, browser-safe
- `SUPABASE_SERVICE_ROLE_KEY` — service-role, bypasses RLS

Use these from app code (yaya-engine reads them from `services.supabase.url` /
`services.supabase.api_key`) or from `curl`:

```bash
curl -s "${SUPABASE_URL}/rest/v1/alert_subscriptions?country=eq.Ethiopia&select=count" \
  -H "apikey: ${SUPABASE_SERVICE_ROLE_KEY}" \
  -H "Authorization: Bearer ${SUPABASE_SERVICE_ROLE_KEY}" \
  -H "Prefer: count=exact"
```

PostgREST **cannot run DDL**. Anything that touches schema needs layer 2 or 3.

### 2. Supabase CLI session — the path for migrations

`supabase login` (one-time) stores a personal access token in the macOS
Keychain under `service="Supabase CLI"`, `account="supabase"`. The CLI uses
this to talk to `api.supabase.com` and, critically, to **provision a temporary
login role inside Postgres** when running remote DB operations.

**This means you do not need a DB password to apply migrations.** Once the
project is linked (`supabase link --project-ref fljikmhgsywkrmqckeaq` from a
repo with `supabase/config.toml`), these all work without ever prompting for a
DB password:

```bash
supabase migration list --linked        # diff Local vs Remote migrations
supabase db push --linked --yes         # apply pending migrations
supabase db dump --linked --schema=public --file schema.sql
```

You will see `Initialising login role…` then `Connecting to remote database…`
in the output — that is the temporary role mechanism.

The raw token in the keychain is a wrapped/encoded blob (its prefix is
`go-k…`, not the `sbp_…` of a Supabase personal access token). **Don't try
to lift it out and hit `api.supabase.com` with `curl` directly — it won't
authenticate.** Use the CLI; it knows how to unwrap.

### 3. Direct Postgres password — usually unnecessary

The DB password is **not stored locally anywhere** (verified across all
atram repos, `~/.pgpass`, keychain, dotfiles, shell history). The only
"connection strings" you'll find are templates with `[YOUR-PASSWORD]` placeholders.

You only need a real DB password if:

- You want to run `psql` directly against the pooler.
- You want to use a tool the CLI can't drive.

To get one: Supabase Dashboard → Project Settings → Database → Reset database
password. **This invalidates the previous password for anything that hardcoded
it**, so coordinate with whoever might be using it. Almost always, prefer the
CLI path in §2.

## Applying a schema change — the canonical recipe

```bash
# 1. From rada-message-system (canonical home for migrations)
cd ~/Documents/atram/dev/rada-message-system

# 2. Create the migration file
#    Convention: YYYYMMDDhhmmss_short_description.sql (see existing files)
echo "ALTER TABLE public.users ALTER COLUMN phone_number DROP NOT NULL;" \
  > supabase/migrations/$(date -u +%Y%m%d%H%M%S)_drop_phone_not_null.sql

# 3. Confirm only YOUR migration is pending (no drift)
supabase migration list --linked
# expect: every existing migration shows in both Local and Remote columns;
# your new one shows in Local only.

# 4. Apply (CLI provisions the login role; no DB password prompt)
supabase db push --linked --yes

# 5. Re-list to confirm it landed on Remote
supabase migration list --linked
```

If `migration list` shows a Remote migration with no Local counterpart, the
canonical migrations dir has drifted from production — stop and reconcile with
whoever pushed out-of-band before proceeding.

For the regular release flow of `rada-message-system` itself (branch
convention, who deploys what), see
[atram-release-flow](https://github.com/Atram-Inc/.github/blob/main/AGENTS.md#release-flow)
or the user-side `atram-release-flow` skill.

## Reading data ad-hoc

| Need                                  | Tool                                                                        |
| ------------------------------------- | --------------------------------------------------------------------------- |
| One-off SQL exploration               | Supabase Studio SQL editor (Dashboard → SQL Editor)                          |
| Programmatic read                     | PostgREST + service-role key (example above)                                |
| Bulk dump                             | `supabase db dump --linked --data-only --file dump.sql`                     |
| `psql` shell                          | Reset DB password (Dashboard), then `psql "$(supabase db remote get)"` (CLI helpers vary by version) |

There is **no shared `pgAdmin` server** or persistent psql connection — every
ad-hoc read should go through one of the above.

## Cross-repo data flow (one line per producer)

```
yaya-engine          ── writes ──▶  users, alert_subscriptions
rada-weather (cron)  ── writes ──▶  town_forecasts_historic, gauge_*
rada-message-system  ── writes ──▶  scheduled_alerts, composed_messages
                                   ▲
alert-manager        ── reads ─────┘  (via meteops-api edge function)
```

## Schema gotchas (the ones that have bitten us)

### Country is stored as a **full English name** in Supabase, not ISO-2

`alert_subscriptions.country`, `scheduled_alerts.country`,
`towns_24h_rain_thresholds.country`, `states_24h_rain_thresholds.country`
all carry **`"Kenya"` / `"Colombia"` / `"Ethiopia"`** — not `KE`/`CO`/`ET`.

`yaya-engine` stores ISO-2 (`users.country = 'ET'`) and converts on sync via
`SupabaseService::resolveCountryName()`, which uses
`\Locale::getDisplayRegion('-ET', 'en')` to produce `"Ethiopia"`. The function
**throws on null/unknown codes** — never silently defaults — because the
previous fallback-to-"Kenya" behaviour silently corrupted data for any non
KE/CO country (see issue `Atram-Inc/yaya-engine#169`).

When querying from `alert-manager` or `meteops-api`, pass the full English
name: `country=eq.Ethiopia`, not `country=eq.ET`.

### `users.phone_number` and `phone_number_id` are **nullable** as of 2026-05-27

Migration: `rada-message-system/supabase/migrations/20260527120000_make_users_phone_number_nullable.sql`.

Before this, the Supabase schema disagreed with the yaya-engine code comment
on `SupabaseService::resolveSupabaseUserId()` (which assumed phone-less Telegram
users would land in `users` fine). The mismatch silently dropped every
phone-less Telegram signup with HTTP 400 (`23502 null value … violates
not-null constraint`).

The UNIQUE INDEX `users_phone_number_unique` still exists. Postgres treats
NULLs as distinct in unique indexes by default, so any number of phone-less
users coexist without violating uniqueness.

### `users.country` (in Supabase) versus `users.country` (in yaya-engine MySQL)

They are **different columns in different databases** with **different
formats**. Supabase `users.country` is the lowercase slug (`"kenya"`,
`"ethiopia"`); yaya-engine `users.country` is the ISO-2 code (`"KE"`,
`"ET"`). The sync writes the lowercase slug into Supabase `alert_subscriptions`,
not into Supabase `users`. Don't conflate them when reading either DB.

### FSP country in yaya-engine is the authoritative country source for a user

A user inherits `country` from their `Fsp` on creation (since
[yaya-engine#303](https://github.com/Atram-Inc/yaya-engine/pull/303)). Make
sure new FSPs are seeded with a correct `country` ISO-2 — otherwise their
users won't sync. As of 2026-05-27 the correct values are:

| `fsps.slug`        | `fsps.country` |
| ------------------ | -------------- |
| `fortune-credit`   | `KE`           |
| `bancamia`         | `CO`           |
| `ethiopia-pilot`   | `ET`           |
| `atram`            | `KE` (internal/test; ambiguous) |

## Things to never do

- **Don't add migrations in `supabase-crons-and-functions`** — it's archived.
  Migrations go in `rada-message-system/supabase/migrations/`.
- **Don't paste DB passwords into chat, into the repo, or into `CLAUDE.md`.**
  See the `Atram-Inc/yaya-engine` post-mortem for the previous leak.
- **Don't hand-craft `postgresql://postgres:…@db.<ref>.supabase.co`
  connection strings** for routine use. If the CLI session can do it, prefer
  the CLI. If it genuinely can't (rare), source the password from
  `~/.config/yaya-engine/secrets.env` (gitignored, `chmod 600`).
- **Don't default unknown country codes to anything.** Any "if not Kenya/Colombia
  then Kenya" pattern is the bug from `yaya-engine#169` recurring — make it
  throw or surface the failure.

## Related references

- `Atram-Inc/yaya-engine` — `SupabaseService.php` is the canonical sync code.
- `Atram-Inc/rada-message-system` — `supabase/migrations/`, `supabase/functions/`.
- `Atram-Inc/alert-manager` — `supabase/functions/meteops-api/` is the read gateway.
- `Atram-Inc/.github/AGENTS.md` — org-wide agent notes.
- `Atram-Inc/yaya-engine/docs/plans/2026-05-04-feat-ethiopia-alert-manager-setup-plan.md` —
  cross-repo plan that motivated several of the schema decisions above.
