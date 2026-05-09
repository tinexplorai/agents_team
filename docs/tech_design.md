# BiteIQ — Tech Design

> **Phase 2 deliverable** owned by the Architect Agent.
> Companion to [`api_contract.md`](api_contract.md). Schema, RLS, AI pipeline, and cross-cutting decisions worth recording so DEV (Phase 3) and future maintainers don't re-litigate them.
>
> DEV owns Phase 3 migrations — every SQL block in this file is a *proposal* that DEV applies via `supabase migration new ...` + `mcp__supabase__apply_migration`. Architect did NOT apply DDL.
>
> Supabase project: `lgmaotvngipnsmmhdgkp` (URL `https://lgmaotvngipnsmmhdgkp.supabase.co`).
> Schema designed for new project — Architect attempted `mcp__supabase__list_tables` to confirm; MCP returned `Unauthorized` (token in `.env` not loaded into the MCP runtime). DEV should re-verify with a fresh token before applying migrations. The project is brand-new per resources file, so empty-DB assumption holds.

---

## A. Supabase Schema

10 tables in the `public` schema. Every user-data table has an indexed `user_id uuid` FK to `auth.users(id)` `ON DELETE CASCADE` — Supabase's auth deletion cleans most rows automatically (account-deletion endpoint still does explicit deletes for storage + audit).

### Enums

```sql
create type meal_source as enum ('photo', 'barcode', 'voice', 'manual');
create type meal_slot as enum ('breakfast', 'lunch', 'dinner', 'snack');
create type sex_enum as enum ('male', 'female');
create type activity_level as enum ('sedentary', 'light', 'moderate', 'active', 'very_active');
create type goal_direction as enum ('lose', 'maintain', 'gain');
create type units_pref as enum ('metric', 'imperial');
create type body_metric as enum ('weight', 'waist', 'hip', 'chest', 'body_fat');
create type chat_role as enum ('user', 'assistant');
```

### `profiles`

One row per user. `user_id` is the PK *and* the FK — enforces 1:1 with `auth.users`.

```sql
create table public.profiles (
  user_id uuid primary key references auth.users(id) on delete cascade,
  sex sex_enum not null,
  age_years int not null check (age_years between 13 and 120),
  height_cm numeric(5,2) not null check (height_cm between 50 and 250),
  weight_kg numeric(5,2) not null check (weight_kg between 20 and 300),
  activity_level activity_level not null,
  goal goal_direction not null,
  target_weight_kg numeric(5,2) check (target_weight_kg is null or target_weight_kg between 20 and 300),
  units_preference units_pref not null default 'metric',
  timezone text not null default 'UTC',          -- IANA name
  paid_tier_flag boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
create index idx_profiles_paid_tier on public.profiles (paid_tier_flag) where paid_tier_flag = true;
```

### `goals`

Current goals (1:1 with profile). History is in `goals_history`.

```sql
create table public.goals (
  user_id uuid primary key references auth.users(id) on delete cascade,
  daily_calories int not null check (daily_calories > 0 and daily_calories < 20000),
  protein_g int not null check (protein_g >= 0),
  carbs_g int not null check (carbs_g >= 0),
  fat_g int not null check (fat_g >= 0),
  is_manual_override boolean not null default false,
  formula text not null default 'mifflin_st_jeor',
  updated_at timestamptz not null default now()
);
```

### `goals_history`

Append-only. Used to render past-date dashboards with the goal active that day (US-13 AC4, AR6).

```sql
create table public.goals_history (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  daily_calories int not null,
  protein_g int not null,
  carbs_g int not null,
  fat_g int not null,
  is_manual_override boolean not null,
  effective_from timestamptz not null default now(),
  effective_until timestamptz                          -- null = currently active row
);
create index idx_goals_history_user_eff on public.goals_history (user_id, effective_from desc);
```

When `goals` is updated: server sets `effective_until = now()` on the previous active row, then inserts a new row with `effective_until = null`.

### `meals`

```sql
create table public.meals (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  logged_at timestamptz not null,
  slot meal_slot not null,
  source meal_source not null,
  notes text,
  photo_storage_path text,                         -- e.g. 'food-photos/<uid>/<sha>.jpg'
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
create index idx_meals_user_logged on public.meals (user_id, logged_at desc);
create index idx_meals_user_slot on public.meals (user_id, slot, logged_at desc);
```

The dashboard's hot query is `where user_id = ? and logged_at between ? and ?` so the (`user_id`, `logged_at desc`) composite index is the workhorse. No partition needed for MVP scale (1k → 10k MAU).

### `meal_items`

Each meal can have N items (US-3 AC4 returns an array). Macros stored at item level so the user can edit one without recomputing others.

```sql
create table public.meal_items (
  id uuid primary key default gen_random_uuid(),
  meal_id uuid not null references public.meals(id) on delete cascade,
  food_name text not null check (length(food_name) between 1 and 200),
  brand text,
  serving_grams numeric(7,2) not null check (serving_grams > 0),
  calories numeric(7,2) not null check (calories >= 0),
  protein_g numeric(6,2) not null check (protein_g >= 0),
  carbs_g numeric(6,2) not null check (carbs_g >= 0),
  fat_g numeric(6,2) not null check (fat_g >= 0),
  source_ref text,                                 -- OFF barcode, AI cache key, or null
  position int not null default 0
);
create index idx_meal_items_meal on public.meal_items (meal_id, position);
```

### `body_measurements`

```sql
create table public.body_measurements (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  metric body_metric not null,
  value numeric(7,2) not null,
  unit text not null,                              -- 'kg' | 'cm' | 'percent', validated app-side
  measured_at timestamptz not null,
  created_at timestamptz not null default now()
);
create index idx_measurements_user_metric_time on public.body_measurements (user_id, metric, measured_at desc);
```

### `ai_photo_cache` *(global, not per-user)*

```sql
create table public.ai_photo_cache (
  image_sha256 text primary key,
  recognition_json jsonb not null,                 -- the structured Claude response
  model text not null,                             -- e.g. 'claude-sonnet-4-6' — invalidate cache when model changes
  created_at timestamptz not null default now(),
  hit_count int not null default 0
);
create index idx_ai_photo_cache_created on public.ai_photo_cache (created_at desc);
```

`hit_count` increments on every cache hit so we can monitor real-world hit rate (target ≥ 30% per `1_project_description.md`). `model` is part of the design so a future model swap doesn't return stale cached recognitions — server reads only rows matching the current `vision_model` env var.

### `ai_usage_quota`

```sql
create table public.ai_usage_quota (
  user_id uuid not null references auth.users(id) on delete cascade,
  day_local date not null,                         -- (now() at time zone profile.timezone)::date
  photo_count int not null default 0 check (photo_count >= 0),
  chat_count int not null default 0 check (chat_count >= 0),
  primary key (user_id, day_local)
);
create index idx_quota_day on public.ai_usage_quota (day_local);
```

PK = `(user_id, day_local)`, exactly per A7. Increment via:

```sql
insert into ai_usage_quota (user_id, day_local, photo_count, chat_count)
values ($1, $2, case when $3 = 'photo' then 1 else 0 end, case when $3 = 'chat' then 1 else 0 end)
on conflict (user_id, day_local) do update
  set photo_count = ai_usage_quota.photo_count + (case when $3 = 'photo' then 1 else 0 end),
      chat_count  = ai_usage_quota.chat_count  + (case when $3 = 'chat'  then 1 else 0 end);
```

### `chat_messages`

```sql
create table public.chat_messages (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  conversation_id uuid not null,                   -- groups messages; not a FK (lightweight)
  role chat_role not null,
  content text not null,
  created_at timestamptz not null default now()
);
create index idx_chat_user_conv_time on public.chat_messages (user_id, conversation_id, created_at);
create index idx_chat_user_recent on public.chat_messages (user_id, created_at desc);
```

`conversation_id` is just a uuid the client/server generates — no separate `conversations` table for MVP (saves a join; a "title" column can be added later as part of `chat_messages` or a side table).

### `food_favorites`

```sql
create table public.food_favorites (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  food_name text not null,
  brand text,
  serving_grams numeric(7,2) not null check (serving_grams > 0),
  calories numeric(7,2) not null check (calories >= 0),
  protein_g numeric(6,2) not null,
  carbs_g numeric(6,2) not null,
  fat_g numeric(6,2) not null,
  created_at timestamptz not null default now(),
  unique (user_id, food_name, brand)               -- prevents accidental duplicates
);
create index idx_food_favorites_user on public.food_favorites (user_id, created_at desc);
```

### `paid_tier_waitlist` *(stub for A2)*

```sql
create table public.paid_tier_waitlist (
  user_id uuid primary key references auth.users(id) on delete cascade,
  joined_at timestamptz not null default now()
);
```

### Storage

Single private bucket `food-photos`. Path convention: `{user_id}/{sha256}.jpg`. Bucket policies (set via Supabase dashboard or migration):

- `select` allowed when path's first segment = `auth.uid()::text` OR signed URL.
- `insert`: server-only via service role.
- No public read.

---

## B. Row Level Security (RLS)

Every user-data table has RLS enabled. The pattern below repeats for `profiles`, `goals`, `goals_history`, `meals`, `meal_items`, `body_measurements`, `ai_usage_quota`, `chat_messages`, `food_favorites`, `paid_tier_waitlist`.

```sql
alter table public.profiles enable row level security;

create policy "users select own profile" on public.profiles
  for select using (auth.uid() = user_id);
create policy "users insert own profile" on public.profiles
  for insert with check (auth.uid() = user_id);
create policy "users update own profile" on public.profiles
  for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "users delete own profile" on public.profiles
  for delete using (auth.uid() = user_id);
```

### `meal_items` exception

`meal_items` has no `user_id` column (it's reachable via `meal_id → meals.user_id`). Policies must join:

```sql
alter table public.meal_items enable row level security;

create policy "users select own meal items" on public.meal_items
  for select using (
    exists (select 1 from public.meals m where m.id = meal_items.meal_id and m.user_id = auth.uid())
  );
-- repeat for insert/update/delete
```

### `ai_photo_cache` exception (global, not per-user)

```sql
alter table public.ai_photo_cache enable row level security;

create policy "anyone authenticated can read cache" on public.ai_photo_cache
  for select using (auth.role() = 'authenticated');
-- No insert/update/delete via anon/authenticated. Only service role writes.
```

This is intentional — the cache is a cost-saving global, and a row hashes a meal photo with no user-identifying data attached. Read access is gated behind login (so we don't leak the full cache to the world) but cross-user (so user A's meal photo benefits user B's identical photo).

### `paid_tier_flag` write protection

`profiles.paid_tier_flag` should NOT be writable by users themselves (they can't grant themselves paid tier). The `update` policy above is too permissive. Add a column-level check:

```sql
create policy "users update own profile (no paid_tier_flag)" on public.profiles
  for update using (auth.uid() = user_id)
  with check (
    auth.uid() = user_id
    and paid_tier_flag = (select paid_tier_flag from public.profiles p where p.user_id = auth.uid())
  );
```

The service role bypasses RLS, so the upgrade-stub admin path can flip the flag manually.

---

## C. AI Vision Pipeline

End-to-end request flow for `POST /api/ai/recognize`:

```
client                   route handler                     supabase                 anthropic
  │                            │                                │                        │
  │ multipart upload (image)   │                                │                        │
  ├───────────────────────────>│                                │                        │
  │                            │ (1) auth check (cookie)        │                        │
  │                            │ (2) sha256(image_bytes)        │                        │
  │                            │ (3) lookup ai_photo_cache      │                        │
  │                            ├───────────────────────────────>│                        │
  │                            │<──── { hit | miss } ───────────┤                        │
  │                            │                                │                        │
  │                            │ if HIT:                        │                        │
  │                            │   upload original to Storage   │                        │
  │                            │   return cached items          │                        │
  │                            │   (no quota decrement)         │                        │
  │                            │                                │                        │
  │                            │ if MISS:                       │                        │
  │                            │   read ai_usage_quota for      │                        │
  │                            │     (uid, day_local)           │                        │
  │                            ├───────────────────────────────>│                        │
  │                            │<──── { used: N } ──────────────┤                        │
  │                            │                                │                        │
  │                            │   if used >= 5 and not paid:   │                        │
  │                            │     return 429 quota-exceeded  │                        │
  │                            │                                │                        │
  │                            │   resize image (max 1568px)    │                        │
  │                            │   upload to Storage            │                        │
  │                            │   call Claude vision           │                        │
  │                            ├──────────────────────────────────────────────────────>  │
  │                            │<──── structured output ──────────────────────────────── │
  │                            │   insert ai_photo_cache         │                       │
  │                            │   increment quota               │                       │
  │                            │   return items + photo_path    │                        │
  │<───────────────────────────┤                                │                        │
```

### Claude vision request shape

Anthropic Messages API with vision content + Zod-validated structured output via tool-use:

```ts
const recognitionTool = {
  name: 'record_food_recognition',
  description: 'Record the food items visible in the image with portion size and macros.',
  input_schema: {
    type: 'object',
    properties: {
      items: {
        type: 'array',
        minItems: 0,
        items: {
          type: 'object',
          properties: {
            food_name: { type: 'string' },
            brand: { type: ['string', 'null'] },
            serving_grams: { type: 'number', minimum: 0 },
            calories: { type: 'number', minimum: 0 },
            protein_g: { type: 'number', minimum: 0 },
            carbs_g: { type: 'number', minimum: 0 },
            fat_g: { type: 'number', minimum: 0 },
            confidence: { type: 'number', minimum: 0, maximum: 1 }
          },
          required: ['food_name', 'serving_grams', 'calories', 'protein_g', 'carbs_g', 'fat_g', 'confidence']
        }
      },
      low_confidence: { type: 'boolean' }
    },
    required: ['items', 'low_confidence']
  }
};
```

`tool_choice: { type: 'tool', name: 'record_food_recognition' }` forces structured output; we Zod-parse the result before persisting. If parsing fails → `502 upstream-failed`.

Model: `claude-sonnet-4-6` (per `_input/2_resources.md`). System prompt mentions Vietnamese + Western food families since target user-base is Vietnamese-leaning (`1_project_description.md` mentions "top-100 món Việt + Western").

### AI chat caching note

`POST /api/ai/chat` does NOT cache — chat is conversational and per-user-context. `chat_count` in `ai_usage_quota` increments on every successful streamed response (after the stream completes; if Anthropic dies mid-stream, the count is NOT incremented — failed call shouldn't burn quota).

---

## D. Quota & Timezone Semantics

### `day_local` calculation

Server-side, on every quota-touching request:

```sql
select (now() at time zone p.timezone)::date as day_local
from public.profiles p
where p.user_id = $1;
```

Lifted into a Postgres function so it's reusable:

```sql
create or replace function public.user_local_date(uid uuid)
returns date
language sql
stable
as $$
  select (now() at time zone coalesce(p.timezone, 'UTC'))::date
  from public.profiles p
  where p.user_id = uid;
$$;
```

### Reset-at calculation

For `/api/quota.resets_at`: next local midnight in UTC =

```sql
select ((current_date + interval '1 day') at time zone p.timezone) as resets_at_utc
from public.profiles p
where p.user_id = $1;
```

(Subtle: this gives the UTC instant of midnight in the user's TZ — not "midnight UTC of tomorrow".)

### DST + traveling

Per A7. If the user crosses timezones mid-day and then changes `profile.timezone`, the quota row keyed by their *previous* `day_local` is preserved — the *new* `day_local` may be a different calendar day so they could effectively get a fresh quota. This is acceptable for MVP (rare edge case, low abuse risk).

---

## E. Cost Guardrails

Per `_input/1_project_description.md` §3:

- Free tier: **5 photo recognitions/day**, **10 chat messages/day** (A4).
- Paid tier: unlimited (no quota row consulted).
- Cache hits do NOT count.
- Cache hit-rate target: **≥ 30%**. Monitored via `select sum(hit_count)::float / (sum(hit_count) + count(*)) from ai_photo_cache;` (rough — `count(*)` is misses-that-became-rows; `hit_count` is hits-on-existing).

Marginal cost rough math (Claude Sonnet 4.6, vision):
- ~$3 / 1M input + ~$15 / 1M output tokens (subject to actual pricing).
- One photo ≈ 1568×1568 image ≈ ~1.5k input tokens + ~250 output tokens (structured tool-use is short).
- Cost per photo ≈ $0.005–0.01.
- 5 photos × 1k MAU × 30 days × 0.7 (1 − cache hit rate) ≈ ~$100/mo cap on free tier vision before paid offsets.

If real-world cost runs over: cache hit-rate is the lever (improve via global cache + re-prompting users to upload at consistent angles), or downgrade vision model to `claude-haiku-4-5-20251001`.

---

## F. Architecture Decisions

### F1. Why Next.js API routes over Edge Functions

- AI vision endpoint needs flexible runtime (image resize via `sharp`, ~5s p95 budget). Vercel Edge Functions cap at 25s but cold-start can spike. Next.js API routes on Vercel run on Node (more predictable for `sharp` + larger memory) with the `runtime = 'nodejs'` directive.
- Edge Functions reserved for: weekly insights cron (deferred per AR5), AI cost metering rollups (also deferred).

### F2. Why streaming for AI chat (not request-response)

- US-9 AC3 explicitly demands streaming. Vercel AI SDK's `StreamingTextResponse` + Anthropic's `messages.stream()` is the standard pattern.
- Streaming also lets us interrupt early if the user closes the tab — saves Anthropic tokens.

### F3. Why image-hash global cache

- Same food photo (e.g. a popular delivery-app dish photo, a chain restaurant menu shot) uploaded by different users has identical SHA-256. Global cache turns that into a free recognition for both users.
- Privacy: the cache stores only the recognition JSON, not the image or any user attribution. Read is gated behind login. No PII leak.
- Expected hit rate ≥ 30% based on similar product implementations; will monitor.

### F4. Why `timestamptz` everywhere + per-user timezone column

- `timestamptz` stores UTC and lets clients render in any zone — covers users traveling, DST.
- Per-user timezone needed for daily totals + quota reset (A7). One column on `profiles` is cheaper than re-deriving from IP every request.

### F5. Why `goals_history` instead of soft-deleting `goals`

- Past-day dashboards need the goal active *that* day (US-13 AC4).
- Append-only history table is conceptually simpler than `goals.deleted_at` + per-row `effective_until` and querying becomes a straightforward range scan.

### F6. Why one bucket for photos (not per-user)

- Supabase has a per-bucket policy ceiling and per-user buckets don't scale. Single bucket + path-prefix RLS (`storage.objects.name like auth.uid() || '/%'`) is the pattern recommended in Supabase docs.

### F7. Why `meal_items` has no `user_id`

- Pure normalization — `user_id` lives on `meals`, items are children. Joining to enforce RLS is acceptable for MVP volumes (every read goes through a covered index).
- If the join becomes hot-path expensive at scale, denormalize `user_id` into `meal_items` (same RLS rule, no join). Defer.

### F8. Why no `recipes` / `food_db` table

- Open Food Facts is the source of truth for branded foods (US-4). Caching their results in our DB invites staleness and dependency on their data quality.
- User's history (queryable from `meal_items`) is the de facto "personal food DB" (US-6 AC2).
- `food_favorites` is the user's curated list — explicit opt-in, simpler than implicit history mining.

### F9. Why synchronous account deletion (US-11)

- Per A11, MVP volumes per user are small (kilobytes of meals + a few photos). Sync deletion finishes inside a single 30s request; if it doesn't, return 503 and we'll revisit with an async job.

### F10. Why Web Speech API client-side (no server endpoint for transcription)

- Saves cost (no Whisper / Assembly API). Browser support is acceptable for MVP (Chrome, Edge, Safari iOS); Firefox-without-flag fallback documented in US-5 AC5.
- Server-side `/api/ai/parse-text` handles structuring the transcribed text into items — that's AI work that has to happen somewhere.

---

## G. Open Questions / N/A-Derived Decisions

The following decisions were made by the Architect because the corresponding fields in `_input/2_resources.md` or upstream specs were `N/A` or unspecified. Each is also added to `task_board.md` `## N/A-derived decisions`.

| # | Question | Decision | Rationale |
|--:|----------|----------|-----------|
| 1 | Chat free-tier message cap (PO A4 deferred number) | **10 messages/day** | Parity with photo cap relative to expected use; cheap to revisit. |
| 2 | Conversation grouping for chat (no `conversations` table?) | `conversation_id` is a uuid stored on each row in `chat_messages`; no separate table | Single-table is cheaper and adequate; a side table can be added if titles/summaries are needed. |
| 3 | Image MIME + size limits | JPEG/PNG/WebP/HEIC, 10 MB, server-resize to 1568px max | Covers iPhone (HEIC) + Android (JPEG); 1568px is Claude's documented optimal for vision. |
| 4 | Weekly insights generation: cron-precomputed vs. on-read | **On-read** for MVP | ≤ 7 rows aggregated; cron defers until cost/latency requires. |
| 5 | Goal history tracking | Dedicated `goals_history` append-only table | Past-day dashboards need the historic goal (US-13 AC4); soft-delete on `goals` is uglier. |
| 6 | Cache-cross-user privacy | Global cache, read-only via authenticated users, store no PII | Cache contains only macro JSON; major cost lever. |
| 7 | Anthropic structured output | Tool-use with forced `tool_choice` + Zod validation | Most reliable structured-output pattern on Claude API. |
| 8 | Rate limit beyond daily quota | 30 req/sec/user on `/api/ai/*` (in-memory + Vercel Edge Config) | Daily quota stops abuse over 24h; per-second limit stops burst attacks. |
| 9 | OFF endpoint stability | Server-side proxy + 24h public Cache-Control on `/api/barcode/:code` | OFF responses are stable; we can cache aggressively. |
| 10 | Account-deletion partial-failure handling | Internal retries up to 3×, then surface 502 with phase | Per US-11 AC6 — never leave half-deleted state silently. |

DEV may revisit any of these in Phase 3; if they do, append a row to `task_board.md` `## N/A-derived decisions` recording the new choice.

---

## H. Migration Notes for DEV (Phase 3)

- Use `supabase migration new <name>` for each logical step (one for enums, one for tables, one for RLS, one for storage policies). Easier to roll back.
- Order: enums → tables → indexes → RLS policies → storage bucket + policies → seed/test data (if any).
- Apply via `mcp__supabase__apply_migration` with project_ref `lgmaotvngipnsmmhdgkp`.
- After applying, run `mcp__supabase__get_advisors` (security) — every user-data table missing an RLS policy will surface as a finding. Fix before moving on.
- Generate TypeScript types via `mcp__supabase__generate_typescript_types` and check the result into `lib/database.types.ts`. Every `lib/api/*.ts` route handler imports from there.

---

## Phase 12 — Multi-provider vision + i18n schema

> **Change request appended 2026-05-08.** Companion to `docs/api_contract.md` "## Phase 12 — Multi-provider vision + vi-default". Append-only — §A–§H above remain canonical for the existing pipeline.
>
> **Schema baseline confirmed via `mcp__supabase__list_tables` 2026-05-08:** project `lgmaotvngipnsmmhdgkp` has 11 public tables incl. `profiles` (no `locale_preference` column yet) and `ai_usage_quota` (no `provider_quota_state` column). Existing migrations end at `009_increment_quota_rpcs.sql`. Phase 12 starts at `010_*`.

### 12.1. Migration `010_locale_preference` *(required)*

Adds `profiles.locale_preference text` column. Backed by `api_contract.md` §12.F + §12.G. Old rows (4 existing users at time of writing) backfill to `'vi'` automatically via DEFAULT.

```sql
-- 010_locale_preference.sql
alter table public.profiles
  add column locale_preference text not null default 'vi'
    check (locale_preference in ('vi', 'en'));

-- Optional: small index for analytics ("how many vi vs en users?").
-- Not on the hot path — cheap to add later. Leave commented for now.
-- create index idx_profiles_locale on public.profiles (locale_preference);

-- Update updated_at on existing rows so audit logs reflect the migration.
-- (No-op behaviorally — DEFAULT already populates the new column.)
update public.profiles set updated_at = now() where true;
```

**RLS impact:** none. `locale_preference` is owned by the row owner like every other profile column — existing self-RLS policies cover it. DEV must verify after applying via `mcp__supabase__get_advisors`.

**Type generation:** DEV regenerates `lib/database.types.ts` via `mcp__supabase__generate_typescript_types` after this migration.

### 12.2. Migration `011_*` *(NOT created — Option A chosen)*

Per `api_contract.md` §12.I, Architect picked **Option A** (in-memory + Vercel Edge Config) for provider quota-state tracking. Therefore **no `ai_usage_quota.provider_quota_state` column is added.** Phase 12 ships exactly one migration (`010_locale_preference`).

If Ops later finds the cold-start fallback noise unacceptable, a future CR can introduce migration `011_provider_quota_state`:
```sql
-- DEFERRED — not part of Phase 12. Documented for future reference.
-- alter table public.ai_usage_quota
--   add column provider_quota_state jsonb;
```
But this is a deferred path, not Phase-12 scope.

### 12.3. Migration `012_ai_photo_cache_provider` *(required, but additive + nullable)*

The cache stores `model` already (e.g. `'claude-sonnet-4-6'`). With three providers, we also need the provider identifier so:
1. Cache-hit responses can return the correct `provider` field (api_contract §12.D).
2. Cache invalidation when a provider's model changes is provider-scoped.

```sql
-- 012_ai_photo_cache_provider.sql
alter table public.ai_photo_cache
  add column provider text;        -- 'gemini' | 'ollama' | 'claude' | NULL (legacy rows)

-- Backfill: 0 rows currently in cache (per list_tables result), but add the
-- backfill statement defensively in case rows exist by deploy time.
update public.ai_photo_cache set provider = 'claude' where provider is null;

-- Composite index — current code reads by (image_sha256, model); after Phase 12
-- the read filters by (image_sha256, model, provider). Add the index for that path.
create index if not exists idx_ai_photo_cache_sha_model_provider
  on public.ai_photo_cache (image_sha256, model, provider);
```

**Why nullable, not NOT NULL:** keeps the migration zero-downtime — Vercel rolling deploys mean an old container can still INSERT without `provider` for ~30s. The route-handler refactor in Phase 12.3 always sets `provider`, so post-deploy steady-state is no NULLs. A subsequent CR can flip to NOT NULL once steady state is verified.

### 12.4. Quota table — `photo_count` column treatment

Migration `010_locale_preference` does **not** drop or alter `ai_usage_quota.photo_count`. Per `api_contract.md` §12.D:

- Column **stays** in the schema (cheap; preserves history).
- Route handler **may continue** to increment it for observability (optional, DEV's call).
- It is **no longer read for gating** — `quota.photo_used` in the API response is now informational only.

This avoids a destructive schema change while satisfying user clarification 4 (per-day cap removed).

### 12.5. `runVisionChain` orchestrator (pseudocode)

```ts
// biteiq/lib/ai/vision/index.ts

import { GeminiVisionProvider } from './gemini';
import { OllamaVisionProvider } from './ollama';
import { ClaudeVisionProvider } from './claude';
import { getCooldown, setCooldown, COOLDOWN_MS } from './cooldown';
import type { VisionProvider, VisionProviderId, RecognizeHints, RecognizeResult } from './types';

const REGISTRY: Record<VisionProviderId, VisionProvider> = {
  gemini: new GeminiVisionProvider(),
  ollama: new OllamaVisionProvider(),
  claude: new ClaudeVisionProvider(),
};

const DEFAULT_CHAIN: VisionProviderId[] = ['gemini', 'ollama', 'claude'];

function resolveChain(): VisionProviderId[] {
  const raw = process.env.AI_VISION_PROVIDER_CHAIN;
  if (!raw) return DEFAULT_CHAIN;
  const ids = raw.split(',').map((s) => s.trim()).filter(Boolean) as VisionProviderId[];
  const valid =
    ids.length > 0 &&
    ids.every((id) => id in REGISTRY) &&
    new Set(ids).size === ids.length; // no dupes
  if (!valid) {
    console.warn('[runVisionChain] Invalid AI_VISION_PROVIDER_CHAIN, using default', { raw });
    return DEFAULT_CHAIN;
  }
  return ids;
}

export class AllProvidersExhaustedError extends Error {
  readonly attempts: VisionProviderId[];
  constructor(attempts: VisionProviderId[]) {
    super('All vision providers reported quota-exhausted in this request.');
    this.name = 'AllProvidersExhaustedError';
    this.attempts = attempts;
  }
}

export async function runVisionChain(
  image: Buffer,
  mimeType: string,
  hints?: RecognizeHints
): Promise<{ result: RecognizeResult; provider: VisionProviderId; attempts: VisionProviderId[] }> {
  const chain = resolveChain();
  const attempts: VisionProviderId[] = [];

  for (const id of chain) {
    // Skip providers in cooldown (Option A — in-memory + Edge Config).
    const cooldownUntil = getCooldown(id);
    if (cooldownUntil && cooldownUntil > Date.now()) {
      attempts.push(id);                 // still record attempt for observability
      continue;
    }

    const provider = REGISTRY[id];
    try {
      const result = await provider.recognize(image, mimeType, hints);
      attempts.push(id);
      return { result, provider: id, attempts };
    } catch (err) {
      attempts.push(id);
      if (provider.isQuotaExhaustedError(err)) {
        // Set cooldown so subsequent requests skip this provider for COOLDOWN_MS (60s).
        setCooldown(id, COOLDOWN_MS);
        console.info('[runVisionChain] provider quota-exhausted, falling back', {
          provider: id,
          nextInChain: chain[chain.indexOf(id) + 1] ?? null,
        });
        continue;                       // try next provider
      }
      // Non-quota error: re-throw. Caller surfaces as 502 upstream-failed.
      console.error('[runVisionChain] provider non-quota error', { provider: id, err });
      // Annotate err so the caller can name the upstream in details.
      const annotated = err instanceof Error ? err : new Error(String(err));
      (annotated as Error & { upstream?: string; provider_id?: VisionProviderId }).upstream = id;
      (annotated as Error & { upstream?: string; provider_id?: VisionProviderId }).provider_id = id;
      throw annotated;
    }
  }

  // Loop fell through — every provider in the chain returned quota-exhausted.
  throw new AllProvidersExhaustedError(attempts);
}
```

**Route-handler integration in `app/api/ai/recognize/route.ts`** (Phase 12.3 DEV task — sketch):

```ts
try {
  const chainResult = await runVisionChain(buffer, mimeType, { mealSlotHint });
  // ... cache write (with chainResult.provider) + storage upload ...
  return Response.json({ /* ...existing fields */ provider: chainResult.provider, /* ... */ });
} catch (err) {
  if (err instanceof AllProvidersExhaustedError) {
    return errorResponse('all-providers-exhausted',
      'All AI providers are temporarily out of free capacity. Please try again in a few minutes.',
      503,
      { attempts: err.attempts }
    );
  }
  const upstream = (err as Error & { upstream?: string }).upstream ?? 'anthropic';
  return errorResponse('upstream-failed', 'AI vision service unavailable.', 502, { upstream });
}
```

### 12.6. Provider implementations — outline

#### `gemini.ts` — `GeminiVisionProvider`

- SDK: `@google/generative-ai` (DEV adds to `biteiq/package.json`).
- Init: `new GoogleGenerativeAI(process.env.GEMINI_API_KEY!).getGenerativeModel({ model: process.env.GEMINI_VISION_MODEL ?? 'gemini-2.5-pro' })`.
- Request: `generateContent([{ inlineData: { data: image.toString('base64'), mimeType } }, prompt])`.
- Response parsing: ask Gemini to return JSON in a fenced code block (Gemini's structured output via `responseSchema` is supported in 2.5; use it). Parse + Zod-validate against the same `RecognizedItem` schema as `claude.ts` so the orchestrator returns a uniform shape.
- **`isQuotaExhaustedError(err)`:** returns `true` when:
  - `err.status === 429`, OR
  - `err.message` includes `RESOURCE_EXHAUSTED` (case-insensitive), OR
  - `err.errorDetails?.[0]?.reason === 'RATE_LIMIT_EXCEEDED'` (Google AI error envelope), OR
  - `err.message.toLowerCase()` contains `'quota'` or `'rate limit'`.

#### `ollama.ts` — `OllamaVisionProvider`

- No SDK — plain `fetch` to `${OLLAMA_BASE_URL}/chat` (or `/api/chat` depending on Ollama Cloud's documented path; DEV verifies on first integration test).
- Headers: `Authorization: Bearer ${OLLAMA_API_KEY}`, `Content-Type: application/json`.
- Body (Ollama Cloud chat-completion vision shape):
  ```json
  {
    "model": "llama3.2-vision",
    "messages": [
      {
        "role": "user",
        "content": "Identify all foods in this image and return JSON: {items:[{food_name,brand,serving_grams,calories,protein_g,carbs_g,fat_g,confidence}], low_confidence: bool}.",
        "images": ["<base64-image>"]
      }
    ],
    "stream": false,
    "format": "json"
  }
  ```
- Response: parse `body.message.content` as JSON, Zod-validate, map to `RecognizeResult`.
- **`isQuotaExhaustedError(err)`:** returns `true` when:
  - HTTP response status `=== 429`, OR
  - response body's `error` field matches `/rate.?limit|quota|too many requests|exhausted/i`, OR
  - thrown `err.message` contains those tokens.
- **DEV verification step:** Ollama Cloud's exact rate-limit error envelope is not yet stable in their public docs. DEV must confirm by triggering one real rate-limit (or by reading their support docs at integration time) and update the matcher list.

#### `claude.ts` — `ClaudeVisionProvider`

Refactor of the existing inline code in `app/api/ai/recognize/route.ts` lines 130–220. Same Anthropic SDK + tool-use + structured-output pattern documented in §C above. No behavior change versus today's code — just wrapped in a class implementing `VisionProvider`.

- **`isQuotaExhaustedError(err)`:** returns `true` when:
  - `err.status === 429` (Anthropic SDK sets `.status` on `APIError`), OR
  - `err.error?.error?.type === 'rate_limit_error'` (Anthropic error envelope), OR
  - `err.message` matches `/rate.?limit|quota/i`.

### 12.7. Cooldown helper (Option A implementation sketch)

```ts
// biteiq/lib/ai/vision/cooldown.ts

export const COOLDOWN_MS = 60_000;

const local: Map<string, number> = new Map();   // providerId -> cooldown-until-epoch-ms

export function getCooldown(providerId: string): number | null {
  const v = local.get(providerId);
  if (!v) return null;
  if (v <= Date.now()) {
    local.delete(providerId);
    return null;
  }
  return v;
}

export function setCooldown(providerId: string, durationMs: number): void {
  local.set(providerId, Date.now() + durationMs);
}
```

**Vercel Edge Config integration (deferred to DEV):** read `vision_provider_cooldowns` blob at module init; write back via Vercel API on `setCooldown`. DEV may ship Phase 12 with the in-memory layer only and add Edge Config as a follow-up — the contract guarantees correctness either way (worst case = one wasted call per cold start).

### 12.8. i18n loading strategy (`next-intl`)

`next-intl` is already listed in the project description ("Internationalization: MVP English only, nhưng dùng `next-intl` từ đầu"). DEV verifies it is installed in `biteiq/package.json`; if not, install `next-intl@^3`.

#### `i18n.ts` (App Router config)

```ts
// biteiq/i18n.ts
import { notFound } from 'next/navigation';
import { getRequestConfig } from 'next-intl/server';

export const LOCALES = ['vi', 'en'] as const;
export type Locale = (typeof LOCALES)[number];
export const DEFAULT_LOCALE: Locale = 'vi';

export default getRequestConfig(async ({ locale }) => {
  if (!LOCALES.includes(locale as Locale)) notFound();

  // Load active-locale catalog. Fall back to vi (source-of-truth) for missing keys.
  const active = (await import(`./messages/${locale}.json`)).default;
  const fallback = locale === 'vi' ? null : (await import('./messages/vi.json')).default;

  return {
    locale,
    messages: fallback ? { ...fallback, ...active } : active,
    // next-intl renders the key string itself if missing in BOTH catalogs
    // — we explicitly do NOT define `getMessageFallback` to throw, so devs notice in QA.
    onError: (err) => console.warn('[i18n]', err.message),
  };
});
```

#### `middleware.ts` (locale resolution)

Per user clarification 5: do **not** honor `Accept-Language`. Resolution order is:
1. **Logged-in user:** read `profiles.locale_preference` via Supabase server client.
2. **Logged-out user:** read `NEXT_LOCALE` cookie if set (e.g. user toggled in a past session and is now signed-out).
3. **Else:** default to `'vi'`.

```ts
// biteiq/middleware.ts (sketch — DEV merges with existing auth middleware)
import createIntlMiddleware from 'next-intl/middleware';
import { LOCALES, DEFAULT_LOCALE } from './i18n';

const intl = createIntlMiddleware({
  locales: [...LOCALES],
  defaultLocale: DEFAULT_LOCALE,
  localeDetection: false,        // CRITICAL — disable Accept-Language sniffing
  localePrefix: 'never',         // No /vi/* or /en/* in URLs — locale lives in cookie+profile.
});

export async function middleware(req: NextRequest) {
  // ... existing auth resolution ...
  const userLocale = userId
    ? await getUserLocale(userId)        // SELECT profiles.locale_preference
    : (req.cookies.get('NEXT_LOCALE')?.value as Locale | undefined) ?? DEFAULT_LOCALE;

  // Stamp the resolved locale into a request header that i18n.ts reads.
  const res = intl(req);
  res.headers.set('x-bitiq-locale', userLocale);
  return res;
}
```

**Server components** read messages via `useTranslations()` (works in server components in next-intl v3) or `getTranslations()` for explicit-locale calls. **Client components** read via `useTranslations()` from `'next-intl'`.

#### Missing-key fallback policy (US-I18N1 AC7, US-I18N2 AC9)

1. Active locale catalog has the key → render its value. ✅
2. Active is `en` and key is missing → fall back to `vi` (source-of-truth). Log `console.warn`.
3. Both catalogs missing the key → render the key string itself (e.g. `dashboard.cta.logMeal`) so dev/QA notices. Log `console.error`.

Implemented via the `messages: { ...fallback, ...active }` merge in `i18n.ts` above (active overrides fallback; missing-in-active falls through to vi).

### 12.9. LanguageSwitcher mounting

```ts
// biteiq/app/layout.tsx (root, server component) — sketch
import { NextIntlClientProvider } from 'next-intl';
import { getMessages, getLocale } from 'next-intl/server';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const locale = await getLocale();         // resolved by middleware → i18n.ts
  const messages = await getMessages();
  return (
    <html lang={locale}>                    {/* US-I18N1 AC5 — screen-reader correctness */}
      <body>
        <NextIntlClientProvider locale={locale} messages={messages}>
          {children}
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

```ts
// biteiq/components/settings/LanguageSwitcher.tsx (client component) — sketch
'use client';
import { useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { useTranslations, useLocale } from 'next-intl';

export function LanguageSwitcher() {
  const router = useRouter();
  const locale = useLocale();
  const t = useTranslations('settings.language');
  const [pending, start] = useTransition();

  const change = (next: 'vi' | 'en') => {
    if (next === locale) return;
    start(async () => {
      const res = await fetch('/api/profile', {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ locale_preference: next }),
      });
      if (!res.ok) {
        // US-I18N2 AC8: surface non-blocking notice, but flip the UI locally anyway.
        document.cookie = `NEXT_LOCALE=${next}; path=/; max-age=${60*60*24*365}`;
        toast.warning(t('persistFailed'));
      } else {
        document.cookie = `NEXT_LOCALE=${next}; path=/; max-age=${60*60*24*365}`;
      }
      router.refresh();        // US-I18N2 AC2 — soft re-render, no full reload
    });
  };
  // ... segmented-control UI per Designer's spec ...
}
```

### 12.10. Phase 12 Decision-log

| # | Decision | Choice | Rationale |
|--:|----------|--------|-----------|
| 12.D1 | Gemini vision model ID | **`gemini-2.5-pro`** (default, env-overridable via `GEMINI_VISION_MODEL`) | User requested `gemini-3.1-pro-preview`. As of 2026-05-08 the publicly-documented Gemini Pro Preview lineage on Google AI Studio is `gemini-2.5-pro` (vision-capable, generally-available paid tier with a generous free quota for hobby projects). No `gemini-3.1-pro-preview` ID exists in the public AI Studio model catalog. Architect substitutes the closest available production-class vision model and exposes it via env var so DEV/Ops can flip to `gemini-3.1-pro-preview` (or any future ID) the moment it becomes available — zero code change required. **Action item for DEV:** check `GEMINI_VISION_MODEL` env at boot; if `gemini-3.1-pro-preview` is set, attempt; on first 404 from Google fall back to `gemini-2.5-pro` and log a warning (do NOT crash). |
| 12.D2 | Ollama Cloud vision model ID | **`llama3.2-vision`** (default, env-overridable via `OLLAMA_VISION_MODEL`) | Ollama Cloud's catalog (verified per task-board user clarification 2) ships `llama3.2-vision` as a production-grade vision-capable model with multilingual output incl. Vietnamese. Alternatives like `llava` exist but `llama3.2-vision` has stronger benchmark numbers on food/object recognition per Meta's release notes. DEV may swap to a larger variant (`llama3.2-vision:90b`) if cost permits — env var allows zero-code swap. |
| 12.D3 | Quota-state strategy | **Option A** — in-memory + Vercel Edge Config | See `api_contract.md` §12.I trade-off table. Provider quotas are global signals, not per-user; Option B's per-user JSONB column would model at the wrong cardinality and add ~30–80ms DB latency per call. Option A piggy-backs on the existing rate-limit infra (§G #8) at zero marginal cost. Worst case = one wasted call per cold start per provider. |
| 12.D4 | i18n locale-detection | **OFF** (`localeDetection: false` in middleware) | Per user clarification 5 — Vietnamese is the default for new sessions regardless of `Accept-Language`. Locale resolution comes from `profiles.locale_preference` (logged-in) or `NEXT_LOCALE` cookie (logged-out), never from browser headers. |
| 12.D5 | Missing-translation key fallback | **vi → key-string** | `vi` is the source-of-truth catalog for Phase 12. EN missing a key → fall back to vi (US-I18N1 AC7). Both missing → render the key string + `console.error` so devs see the gap during QA. |
| 12.D6 | LanguageSwitcher placement | **Settings page only**, segmented control (vi / en) | Per user clarification 5. Not in onboarding (US-I18N2 AC6), not in logged-out pages (US-I18N2 AC5). Designer owns final visual; this design just specifies the wiring (`PATCH /api/profile { locale_preference }` + `router.refresh()`). |
| 12.D7 | Cache-row provider tracking | **Add nullable `provider` text column** (migration `012`) | Cache hits must return `provider` field per US-MP1 AC8. Nullable for zero-downtime rolling deploy; legacy rows treated as `'claude'`. |
| 12.D8 | Photo-quota column lifecycle | **Keep `photo_count`, stop reading for gating** | Avoids destructive schema change (per task-board guardrail "do not break free-vs-paid distinction elsewhere"). Column still useful for analytics on per-user upload volume. |
| 12.D9 | 30 req/sec/user rate-limit (§G #8) | **Preserved unchanged** | Per user clarification 4 — burst guard stays after per-day cap removal. No code change needed in the rate-limit middleware; only the `/api/ai/recognize` route handler stops reading `quotaInfo.photo.limit` for gating. Confirmed: §G #8 is the ONLY rate-limit infrastructure on the AI surface; Phase 12 does not add or remove any layer. |
| 12.D10 | NEW-7 / NEW-8 deferred-bug status | **Not addressed in Phase 12** | Per user clarification 6 — `ai_photo_cache` INSERT silent fail (NEW-7) and chat quota plateau (NEW-8) are deferred to a separate CR. The `runVisionChain` refactor MAY incidentally fix NEW-7 (if the cause was the inline upsert race); if so, DEV must document but NOT claim fix. QA must NOT add Phase-12 tests for either bug. |

### 12.11. Cost & latency estimates

- **Gemini 2.5 Pro vision:** ~$1.25/1M input tokens, ~$5/1M output (paid tier; free tier is ~50 RPM for hobby projects). One photo ≈ ~1.2k input + ~250 output ≈ ~$0.003/photo paid; free tier covers the first ~1k photos/day across the whole BiteIQ deployment.
- **Ollama Cloud llama3.2-vision:** pricing tier-dependent. Currently free for low-volume hobby usage with 429 rate-limits as the gate.
- **Claude Sonnet 4.6 (last fallback):** unchanged from §E above (~$0.005–0.01/photo).
- **Effective per-photo cost (steady state, 1k MAU):** dominated by Gemini free tier → near-zero unless we exceed Gemini's daily quota, at which point Ollama free tier covers most of the overflow, with Claude paid as the safety net. This satisfies the project's "Supabase free tier đủ cho 1k MAU" goal AND the Phase 12 ask of "remove per-day cap without blowing up cost".
- **Latency p95 budget (5s, project goal):** Gemini 2.5 Pro vision typically returns in 1.5–3s; Ollama Cloud 2–4s; Claude 1.5–3s. Worst-case fall-through-all-three ~10s, but the cooldown system means it's 3s for the steady-state second-request-after-Gemini-exhaustion case (skip Gemini, hit Ollama directly).

### 12.12. QA / DEV verification checklist (cross-references task-board Phase 12.3 + 12.4)

DEV (Phase 12.3):
- [ ] Apply migrations `010_locale_preference.sql` and `012_ai_photo_cache_provider.sql` via `mcp__supabase__apply_migration`. Run `mcp__supabase__get_advisors` after each.
- [ ] Regenerate `lib/database.types.ts` via `mcp__supabase__generate_typescript_types`.
- [ ] Verify `next-intl` is in `biteiq/package.json`; install if missing.
- [ ] Verify Gemini SDK boot does not crash on `gemini-3.1-pro-preview` — implement the 404-retry-with-`gemini-2.5-pro` fallback per decision 12.D1.
- [ ] Confirm Ollama Cloud rate-limit error envelope matches the matcher in `ollama.ts.isQuotaExhaustedError`; update if the docs have evolved by integration time.

QA (Phase 12.4):
- [ ] All test cases listed in `api_contract.md` §12.K.
- [ ] Migration round-trip: with a fresh `qa-paid` test account, `GET /api/profile` returns `locale_preference: 'vi'`. `PATCH /api/profile { locale_preference: 'en' }` → 200 → re-`GET` returns `'en'` → switch back, persists.
- [ ] HTML root `lang` attribute flips on switcher toggle (US-I18N1 AC5, US-I18N2 AC4).
- [ ] No flash of English before hydration on `/`, `/dashboard`, `/onboarding` (US-I18N1 AC8).
- [ ] After applying migrations, run `mcp__supabase__get_advisors` and confirm 0 new findings.


---

## Phase 13 — Service-role client pattern

### Service-role client pattern

The app has two server-side Supabase client factories with different RLS semantics:

- **`createClient()` / `createServiceClient()` in `lib/supabase/server.ts`** — built with `@supabase/ssr`. Both wire a cookie store so Supabase can manage session tokens for SSR. Even when `createServiceClient()` is called with the service-role key, the SSR helper applies cookie-based JWT negotiation, which means the service-role JWT is **not** reliably applied at the Postgres layer and RLS can still block writes.
- **`createAdminClient()` in `lib/supabase/admin.ts`** — built with `@supabase/supabase-js` directly (no cookie store). The service-role key is passed to the plain `createClient()`, which applies it as a JWT unconditionally. Postgres receives the service-role JWT → RLS is bypassed at the database layer, no policies needed.

```ts
// lib/supabase/admin.ts
import { createClient } from "@supabase/supabase-js";

export function createAdminClient() {
  return createClient(url, serviceRoleKey, {
    auth: { persistSession: false, autoRefreshToken: false },
  });
}
```

**Rule:** use `createAdminClient()` only for server-side writes that legitimately bypass RLS — cache population (`ai_photo_cache`), audit logs, admin-only mutations. User-facing reads/writes that must respect per-user data isolation continue to use the cookie-bound server client.
