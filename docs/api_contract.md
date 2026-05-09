# BiteIQ — API Contract

> **Phase 2 deliverable** owned by the Architect Agent.
> Source of truth for DEV (Phase 3) and QA (Phase 4). Every endpoint here MUST be implemented; QA tests every status code listed.
>
> Stack: Next.js 15 App Router route handlers under `app/api/**/route.ts`. All endpoints are JSON unless explicitly noted (`/api/ai/recognize` is `multipart/form-data`; `/api/ai/chat` streams SSE).
>
> Conventions:
> - All times are ISO-8601 UTC strings unless noted. "Day" boundaries follow the user's profile timezone (see `tech_design.md` §D).
> - All units are metric on the wire (kg, cm, g, kcal). Client converts to imperial for display.
> - Auth: Supabase Auth via cookie session (`sb-access-token` / `sb-refresh-token` HTTP-only cookies set by `@supabase/ssr`). Server route handlers read the cookie and resolve `auth.uid()`. Endpoints marked **Auth: required** return `401` if no valid session.
> - RLS does most of the authorization work; route handlers add input validation + business rules (quota, ownership cross-checks).

---

## Assumptions

> Locked here so DEV/QA are not blocked. Each item also appears in `task_board.md` under "## N/A-derived decisions".

- **A1 (PO).** Onboarding formula = Mifflin-St Jeor BMR × activity multiplier (1.2 / 1.375 / 1.55 / 1.725 / 1.9) ± 500 kcal goal delta. **Confirmed.** Default macro split P/C/F = 30/40/30 by calories. **Confirmed.** Computed server-side in `POST /api/profile` and `PATCH /api/profile`; result returned in response so the UI shows it before user confirms (US-2 AC3).
- **A2 (PO).** Paid-tier billing is stubbed. `profiles.paid_tier_flag` is a boolean toggled only via service-role (admin) for MVP. No `/api/billing` endpoints in this contract. `POST /api/account/upgrade-stub` returns `202` and a "join waitlist" payload — wired so US-12 AC3 / US-14 AC3 link somewhere.
- **A4 (PO).** AI chat free-tier quota = **10 messages/day**. Same `day_local` semantics as photo quota. Counted on every successful streamed response (cache-miss only — see §AI chat caching below).
- **A7 (PO).** All "daily" boundaries (dashboard, quota reset, weekly insights) use `(now() AT TIME ZONE profiles.timezone)::date`. Timezone defaults to `UTC` if unset, captured at signup, editable via `PATCH /api/profile`.
- **AR1 (Architect).** Voice input has no server-side transcription endpoint — Web Speech API runs in the browser (US-5 AC1). The transcribed text is parsed by `POST /api/ai/parse-text` (cheap LLM call, no quota — text-only is a small fraction of vision cost). After parsing, the client posts to `POST /api/meals` like any other source.
- **AR2 (Architect).** AI photo cache is **global** (keyed by SHA-256 of image bytes, shared across users). Cache hits do NOT decrement the user's quota (US-3 AC10, US-12 AC6).
- **AR3 (Architect).** Pagination uses cursor-based pagination (`?cursor=<id>&limit=N`, max `limit=100`) on list endpoints. For MVP, only `/api/meals` and `/api/measurements` lists need it; everything else is bounded by date range.
- **AR4 (Architect).** Image upload format: JPEG / PNG / WebP / HEIC, max 10 MB per photo. Server resizes to max 1568px (Claude vision optimal) before forwarding to Anthropic and storing.
- **AR5 (Architect).** Weekly insights are computed on read for MVP (≤7 rows aggregated). The cron-precomputed Edge Function noted in `1_project_description.md` is deferred until cost requires it.
- **AR6 (Architect).** Goal history — when goals change, we keep history via `goals_history` (append-only on every PATCH). Dashboard for past dates uses the goal active at end-of-day for that date (US-13 AC4).

---

## Standard error response

Every non-2xx response (except `204 No Content`) returns:

```ts
interface ErrorBody {
  error: {
    code: string;        // machine-readable, kebab-case (e.g. "quota-exceeded", "validation-failed", "unauthenticated")
    message: string;     // human-readable, English (i18n later)
    details?: unknown;   // optional — field-level errors for 400, retry-after for 429, etc.
  };
}
```

### Common error codes (re-used across endpoints)

| HTTP | code | When |
|-----:|------|------|
| 400 | `validation-failed` | Body fails Zod schema. `details` is `{ field: string; message: string }[]`. |
| 401 | `unauthenticated` | No valid Supabase session. |
| 403 | `forbidden` | RLS denied or user attempted to access another user's resource. |
| 404 | `not-found` | Resource ID doesn't exist or belongs to another user (RLS-masked as 404 to avoid leaking IDs). |
| 409 | `conflict` | Duplicate, e.g. profile already exists on `POST /api/profile`. |
| 413 | `payload-too-large` | Image > 10 MB. |
| 415 | `unsupported-media-type` | Image MIME not in {jpeg, png, webp, heic}. |
| 429 | `quota-exceeded` | Free-tier daily quota hit. `details: { kind: "photo" \| "chat"; limit: number; used: number; resets_at: string }`. |
| 429 | `rate-limited` | Per-user req/sec rate limit (separate from daily quota). `details: { retry_after_seconds: number }`. |
| 500 | `internal-error` | Unhandled. |
| 502 | `upstream-failed` | Anthropic, Open Food Facts, or Supabase Storage failed. `details.upstream` names which. |
| 503 | `service-unavailable` | Temporary, e.g. AI cold-boot timeout. |

---

## Endpoint index

| # | Method | Path | Auth | Purpose | Story |
|--:|--------|------|:----:|---------|-------|
| 1 | (n/a) | Supabase Auth SDK on client | — | Signup/login (email + Google) | US-1 |
| 2 | `POST` | `/api/profile` | yes | Create profile (onboarding) | US-2 |
| 3 | `GET` | `/api/profile` | yes | Read profile + computed goals | US-2, US-13 |
| 4 | `PATCH` | `/api/profile` | yes | Update profile (recomputes suggested goals) | US-13 |
| 5 | `GET` | `/api/goals` | yes | Read current goals | US-2, US-13 |
| 6 | `PATCH` | `/api/goals` | yes | Manually override goals | US-2 AC4, US-13 AC3 |
| 7 | `POST` | `/api/meals` | yes | Log meal (any source) | US-3..US-6 |
| 8 | `GET` | `/api/meals` | yes | List meals for a date range | US-7 AC2, US-7 AC5 |
| 9 | `GET` | `/api/meals/:id` | yes | Read one meal (detail view) | US-7 AC3 |
| 10 | `PATCH` | `/api/meals/:id` | yes | Edit meal | US-7 AC3 |
| 11 | `DELETE` | `/api/meals/:id` | yes | Delete meal | US-7 AC3 |
| 12 | `POST` | `/api/ai/recognize` | yes | Photo → recognized food items (multipart) | US-3 |
| 13 | `POST` | `/api/ai/parse-text` | yes | Voice/manual text → structured items | US-5 AC3, US-6 AC2 |
| 14 | `GET` | `/api/barcode/:code` | yes | Open Food Facts proxy | US-4 |
| 15 | `GET` | `/api/dashboard` | yes | Daily totals vs goals | US-7 |
| 16 | `GET` | `/api/insights/weekly` | yes | 7-day rollup + insights | US-8 |
| 17 | `POST` | `/api/ai/chat` | yes | Streaming chat (SSE) | US-9 |
| 18 | `GET` | `/api/ai/chat/history` | yes | Conversation history | US-9 AC6 |
| 19 | `DELETE` | `/api/ai/chat/history` | yes | Clear chat history | US-9 AC6 |
| 20 | `POST` | `/api/measurements` | yes | Add body measurement | US-10 |
| 21 | `GET` | `/api/measurements` | yes | List measurements (filterable by metric, date range) | US-10 AC3 |
| 22 | `PATCH` | `/api/measurements/:id` | yes | Edit measurement | US-10 AC4 |
| 23 | `DELETE` | `/api/measurements/:id` | yes | Delete measurement | US-10 AC4 |
| 24 | `GET` | `/api/quota` | yes | Current daily usage (photo + chat) | US-12 AC1, US-14 AC1 |
| 25 | `GET` | `/api/account/export` | yes | GDPR export (JSON) | US-11 AC2 |
| 26 | `DELETE` | `/api/account` | yes | GDPR delete | US-11 |
| 27 | `POST` | `/api/account/upgrade-stub` | yes | Paid-tier waitlist stub | US-12 AC3, US-14 AC3 |
| 28 | `POST` | `/api/foods/search` | yes | Suggestion search (history + OFF) | US-6 AC2 |
| 29 | `POST` | `/api/foods/favorite` | yes | Save favorite for one-tap re-log | US-6 AC4 |
| 30 | `GET` | `/api/foods/favorites` | yes | List favorites | US-6 AC4 |
| 31 | `DELETE` | `/api/foods/favorites/:id` | yes | Remove favorite | US-6 AC4 |

**Total: 30 endpoints + Supabase-managed auth.**

---

## 1. Auth (US-1)

**Handled by Supabase Auth SDK on the client.** No custom endpoints. The client uses `@supabase/ssr` to:

- `supabase.auth.signUp({ email, password })` — sign up with email + password.
- `supabase.auth.signInWithOAuth({ provider: 'google', options: { redirectTo: '/auth/callback' } })` — Google OAuth.
- `supabase.auth.signInWithPassword({ email, password })` — log in.
- `supabase.auth.signOut()` — log out.

The OAuth callback route `app/auth/callback/route.ts` exchanges the code for a session cookie via `supabase.auth.exchangeCodeForSession(code)`.

**Email verification:** Supabase project settings — set "Confirm email" ON. The `signUp` response includes `data.user` but no session until the user clicks the link; the client shows the "Check your inbox" screen (US-1 AC7).

**Edge cases (covered without custom endpoints):**
- `AuthApiError: User already registered` → US-1 AC5 (client shows "Log in instead" link).
- OAuth cancel → user lands back on `/auth/callback?error=...` with the error in the URL (US-1 AC6).

---

## 2. `POST /api/profile` (US-2)

**Auth:** required.
**Purpose:** Create a profile + initial goals after signup. Idempotent: if profile already exists, returns 409.

### Request

```ts
interface CreateProfileBody {
  sex: 'male' | 'female';
  age_years: number;             // integer, 13..120
  height_cm: number;              // 50..250
  weight_kg: number;              // 20..300
  activity_level: 'sedentary' | 'light' | 'moderate' | 'active' | 'very_active';
  goal: 'lose' | 'maintain' | 'gain';
  target_weight_kg?: number | null;   // optional, US-2 AC5
  units_preference: 'metric' | 'imperial';   // for UI display, not server math
  timezone: string;                   // IANA, e.g. "Asia/Ho_Chi_Minh". Defaults to "UTC" if client omits.
  // Optional manual override at onboarding (US-2 AC4)
  override_goals?: {
    daily_calories: number;       // > 0
    protein_g: number;            // >= 0
    carbs_g: number;
    fat_g: number;
  };
}
```

### Response — `201 Created`

```ts
interface ProfileWithGoals {
  profile: {
    user_id: string;              // uuid
    sex: 'male' | 'female';
    age_years: number;
    height_cm: number;
    weight_kg: number;
    activity_level: string;
    goal: string;
    target_weight_kg: number | null;
    units_preference: 'metric' | 'imperial';
    timezone: string;
    paid_tier_flag: boolean;
    created_at: string;
    updated_at: string;
  };
  goals: {
    daily_calories: number;
    protein_g: number;
    carbs_g: number;
    fat_g: number;
    is_manual_override: boolean;
    formula: 'mifflin_st_jeor';
    updated_at: string;
  };
  suggested_goals: {              // What the formula computed (shown in US-2 AC3)
    daily_calories: number;
    protein_g: number;
    carbs_g: number;
    fat_g: number;
  };
}
```

### Errors

- `400 validation-failed` — out-of-range fields (US-2 AC7).
- `409 conflict` — profile already exists. `details: { existing_profile_id: string }`.

### Server logic

1. Validate body.
2. Compute Mifflin-St Jeor BMR + activity + goal delta → `suggested_goals`.
3. Insert into `profiles` (`auth.uid()` from session).
4. Insert into `goals` — use `override_goals` if present, else `suggested_goals`. Set `is_manual_override = !!override_goals`.
5. Append to `goals_history`.

---

## 3. `GET /api/profile` (US-2, US-13)

**Auth:** required. Returns the current user's profile + goals.

### Response — `200 OK`

`ProfileWithGoals` (same shape as section 2 — `suggested_goals` is recomputed every read so it always reflects current profile values).

### Errors

- `404 not-found` — profile not yet created (client should redirect to onboarding).

---

## 4. `PATCH /api/profile` (US-13)

**Auth:** required. Partial update. Recomputes `suggested_goals` and returns them. Does **not** auto-apply suggested goals to current `goals` row — caller must `PATCH /api/goals` separately if they want to accept the new suggestion (US-13 AC3).

### Request

All fields from `CreateProfileBody` are optional. At least one must be present.

### Response — `200 OK`

`ProfileWithGoals` — `goals` is unchanged unless caller also patched goals; `suggested_goals` reflects the new profile.

### Errors

- `400 validation-failed`.
- `404 not-found` — profile doesn't exist.

---

## 5. `GET /api/goals` (US-2, US-13)

**Auth:** required.

### Response — `200 OK`

```ts
interface GoalsResponse {
  daily_calories: number;
  protein_g: number;
  carbs_g: number;
  fat_g: number;
  is_manual_override: boolean;
  formula: 'mifflin_st_jeor';
  updated_at: string;
}
```

---

## 6. `PATCH /api/goals` (US-2 AC4, US-13 AC3)

**Auth:** required. Manual override OR re-accept formula suggestion.

### Request

```ts
interface PatchGoalsBody {
  // Either set explicit values...
  daily_calories?: number;
  protein_g?: number;
  carbs_g?: number;
  fat_g?: number;
  // ...OR ask the server to apply the current suggested values:
  apply_suggested?: boolean;
}
```

If `apply_suggested: true`, server recomputes from profile and writes those values; sets `is_manual_override = false`. Else writes the provided values; sets `is_manual_override = true`. Cannot mix both in one call.

### Response — `200 OK`

`GoalsResponse`.

### Errors

- `400 validation-failed` — both `apply_suggested` and explicit values present, or values out of range.

### Server logic

Append to `goals_history` on every successful PATCH so past-day dashboards (US-13 AC4) can find the goal active on each historical day.

---

## 7. `POST /api/meals` (US-3..US-6)

**Auth:** required. The single write endpoint for all four input methods. Source-specific endpoints (`/api/ai/recognize`, `/api/barcode/:code`, `/api/ai/parse-text`) only return *recognition data*; the user reviews/edits, then this endpoint persists the meal.

### Request

```ts
interface CreateMealBody {
  logged_at: string;                  // ISO-8601, defaults to now() if omitted
  slot: 'breakfast' | 'lunch' | 'dinner' | 'snack';
  source: 'photo' | 'barcode' | 'voice' | 'manual';
  notes?: string | null;
  photo_storage_path?: string | null; // returned by /api/ai/recognize when applicable
  items: Array<{
    food_name: string;                // 1..200 chars
    brand?: string | null;
    serving_grams: number;            // > 0
    calories: number;                 // >= 0
    protein_g: number;                // >= 0
    carbs_g: number;
    fat_g: number;
    source_ref?: string | null;       // OFF barcode, AI cache key, or null for manual
  }>;
}
```

### Response — `201 Created`

```ts
interface Meal {
  id: string;                         // uuid
  user_id: string;
  logged_at: string;
  slot: string;
  source: string;
  notes: string | null;
  photo_storage_path: string | null;
  items: Array<{
    id: string;
    food_name: string;
    brand: string | null;
    serving_grams: number;
    calories: number;
    protein_g: number;
    carbs_g: number;
    fat_g: number;
    source_ref: string | null;
  }>;
  created_at: string;
  updated_at: string;
}
```

### Errors

- `400 validation-failed` — empty `items[]`, negative macros, etc.

---

## 8. `GET /api/meals` (US-7 AC2, US-7 AC5)

**Auth:** required.

### Query

| param | type | required | notes |
|-------|------|:--------:|-------|
| `from` | date `YYYY-MM-DD` (user-local) | yes | Inclusive. |
| `to`   | date `YYYY-MM-DD` (user-local) | yes | Inclusive. Max 31 days span. |
| `slot` | enum | no | Filter by slot. |
| `cursor` | string | no | For pagination. |
| `limit` | number 1..100 | no | Default 50. |

The server converts `from` / `to` to UTC ranges using `profiles.timezone`.

### Response — `200 OK`

```ts
interface MealList {
  meals: Meal[];
  next_cursor: string | null;
}
```

### Errors

- `400 validation-failed` — `to < from`, span > 31 days.

---

## 9. `GET /api/meals/:id` (US-7 AC3)

**Auth:** required.

### Response — `200 OK`

`Meal`.

### Errors

- `404 not-found`.

---

## 10. `PATCH /api/meals/:id` (US-7 AC3)

**Auth:** required.

### Request

All fields from `CreateMealBody` are optional. Updating `items` replaces the entire `meal_items` set for this meal (simpler than per-item diff for MVP — items are cheap rows).

### Response — `200 OK`

`Meal`.

### Errors

- `400 validation-failed`.
- `404 not-found`.

---

## 11. `DELETE /api/meals/:id` (US-7 AC3)

**Auth:** required.

### Response

`204 No Content`.

### Errors

- `404 not-found`.

### Server logic

Cascade-deletes `meal_items`. If `photo_storage_path` is set, also deletes the storage object (best-effort — storage error returns `502 upstream-failed` but the meal row is rolled back).

---

## 12. `POST /api/ai/recognize` (US-3)

**Auth:** required.
**Content-Type:** `multipart/form-data`.
**Quota:** counts against `photo_count` (5/day free) UNLESS cache hit (US-3 AC10, US-12 AC6).

### Request

| field | type | required | notes |
|-------|------|:--------:|-------|
| `image` | File | yes | JPEG/PNG/WebP/HEIC, ≤ 10 MB. |
| `meal_slot_hint` | string | no | One of slot enum — passed to Claude as context for portion-size estimation. |

### Response — `200 OK`

```ts
interface RecognitionResponse {
  cache_hit: boolean;                 // true if served from ai_photo_cache
  photo_storage_path: string;         // signed-URL-able path; pass back into POST /api/meals
  image_sha256: string;
  items: Array<{
    food_name: string;
    brand: string | null;
    serving_grams: number;
    calories: number;
    protein_g: number;
    carbs_g: number;
    fat_g: number;
    confidence: number;               // 0..1; client can flag low-confidence per US-3 AC9
  }>;
  low_confidence: boolean;            // true if max(confidence) < 0.5 — surface "couldn't recognize" UX (US-3 AC7, AC9)
  quota: {
    photo_used: number;
    photo_limit: number | null;        // null for paid tier
    chat_used: number;
    chat_limit: number | null;
  };
}
```

### Errors

- `400 validation-failed` — no image attached.
- `413 payload-too-large` — > 10 MB.
- `415 unsupported-media-type`.
- `429 quota-exceeded` — `details: { kind: 'photo', limit: 5, used: 5, resets_at: string }`. Body and metadata are NOT consumed (US-12 AC8 — client preserves form state).
- `502 upstream-failed` — Anthropic call failed (`details.upstream = 'anthropic'`) or Storage upload failed (`details.upstream = 'storage'`).

### Server logic

See `tech_design.md` §C for the full pipeline. Key points:
1. Hash image (SHA-256). Look up `ai_photo_cache`.
2. If hit → return cached items + `cache_hit: true`. Do **not** decrement quota. Still upload image to storage so the user has the photo on their meal record.
3. If miss → check `ai_usage_quota`. If at limit → 429.
4. Resize image (max 1568px), upload to Storage (`food-photos/{user_id}/{sha256}.jpg`).
5. Call Anthropic Claude vision (`claude-sonnet-4-6`) with structured output.
6. Insert into `ai_photo_cache` (upsert on conflict).
7. Increment `ai_usage_quota` for `(user_id, day_local)`.

---

## 13. `POST /api/ai/parse-text` (US-5 AC3, US-6 AC2)

**Auth:** required.
**Quota:** does NOT count against photo or chat quota — text parsing is cheap.
**Purpose:** Convert "two slices of whole wheat toast with peanut butter" into structured items.

### Request

```ts
interface ParseTextBody {
  text: string;            // 1..1000 chars
}
```

### Response — `200 OK`

```ts
interface ParseTextResponse {
  items: Array<{
    food_name: string;
    serving_grams: number;
    calories: number;
    protein_g: number;
    carbs_g: number;
    fat_g: number;
    confidence: number;
  }>;
}
```

### Errors

- `400 validation-failed`.
- `502 upstream-failed`.

---

## 14. `GET /api/barcode/:code` (US-4)

**Auth:** required.
**Path param:** `code` — EAN-13 / UPC-A digits.

### Response — `200 OK`

```ts
interface BarcodeResponse {
  found: boolean;
  product: null | {
    barcode: string;
    name: string;
    brand: string | null;
    serving_grams: number | null;       // null if missing in OFF (US-4 AC7)
    calories: number | null;
    protein_g: number | null;
    carbs_g: number | null;
    fat_g: number | null;
    image_url: string | null;
    source: 'openfoodfacts';
  };
}
```

When `found: false`, client shows "Product not in database — enter manually" with the barcode prefilled (US-4 AC5).

### Errors

- `400 validation-failed` — non-numeric or wrong-length code.
- `502 upstream-failed` — OFF unreachable.

---

## 15. `GET /api/dashboard` (US-7)

**Auth:** required.

### Query

| param | type | required | notes |
|-------|------|:--------:|-------|
| `date` | `YYYY-MM-DD` | no | User-local date. Defaults to today (in user's timezone). |

### Response — `200 OK`

```ts
interface DashboardResponse {
  date: string;                       // user-local YYYY-MM-DD
  goals: {
    daily_calories: number;
    protein_g: number;
    carbs_g: number;
    fat_g: number;
  };
  totals: {
    calories: number;
    protein_g: number;
    carbs_g: number;
    fat_g: number;
  };
  remaining: {
    calories: number;                 // negative if over goal (US-7 AC7)
    protein_g: number;
    carbs_g: number;
    fat_g: number;
  };
  meals_by_slot: {
    breakfast: Meal[];
    lunch: Meal[];
    dinner: Meal[];
    snack: Meal[];
  };
  is_empty: boolean;                  // true if no meals (US-7 AC6)
}
```

For past dates, `goals` is the goal active at end-of-day (`AR6`).

---

## 16. `GET /api/insights/weekly` (US-8)

**Auth:** required.

### Query

| param | type | required | notes |
|-------|------|:--------:|-------|
| `week_ending` | `YYYY-MM-DD` | no | Last day of the 7-day window (user-local). Defaults to today. |

### Response — `200 OK`

```ts
interface WeeklyInsightsResponse {
  week_start: string;                 // user-local
  week_ending: string;
  days_logged: number;                 // 0..7
  daily_totals: Array<{                // length = 7, in chronological order; days with no log have nulls
    date: string;
    calories: number | null;
    protein_g: number | null;
    carbs_g: number | null;
    fat_g: number | null;
    goal_calories: number;             // active goal that day
  }>;
  averages: {
    calories: number;
    protein_g: number;
    carbs_g: number;
    fat_g: number;
  };
  on_target_days: number;              // days within ±10% of calorie goal
  trend: 'improving' | 'steady' | 'off_track';
  insights: Array<{
    severity: 'info' | 'suggestion';
    text: string;                      // English, plain-language, non-medical (US-8 AC4)
  }>;
}
```

When `days_logged < 7`, response still renders; client shows "Showing N of 7 days" notice (US-8 AC5).

---

## 17. `POST /api/ai/chat` (US-9)

**Auth:** required.
**Content-Type (request):** `application/json`.
**Content-Type (response):** `text/event-stream` (SSE; using Vercel AI SDK `StreamingTextResponse`).
**Quota:** counts against `chat_count` (10/day free, A4).

### Request

```ts
interface ChatRequest {
  message: string;                    // 1..2000 chars
  conversation_id?: string | null;     // null/omit = start new
}
```

### Response — `200 OK` (streaming SSE)

Stream events follow Vercel AI SDK protocol:

- `data: {"type":"meta","conversation_id":"<uuid>","message_id":"<uuid>"}\n\n` (first event)
- `data: {"type":"token","value":"..."}\n\n` (repeated, token-by-token)
- `data: {"type":"done","disclaimer":"BiteIQ provides..."}\n\n` (last)
- `data: [DONE]\n\n` (terminator)

The disclaimer (US-9 AC4) is included in the final event so the client can render it under every response. Client also injects the medical-claim refusal behavior via system prompt (US-9 AC5).

### Errors (sent as a single SSE error frame, then stream closes)

- `401 unauthenticated` (returned as HTTP status before stream starts).
- `429 quota-exceeded` (HTTP status before stream starts) — `details: { kind: 'chat', limit: 10, used: 10, resets_at: string }`.
- `502 upstream-failed` — emitted as `data: {"type":"error","code":"upstream-failed",...}\n\n` if Anthropic dies mid-stream.

### Server logic

1. Auth + quota check (HTTP 429 if exceeded).
2. Resolve `conversation_id` (insert new if null).
3. Insert user message into `chat_messages`.
4. Build system prompt with: goals + today's meals + last 7 days summary (read-only awareness, US-9 AC2). System prompt also includes the medical-refusal behavior.
5. Stream from Anthropic; tee to client AND buffer for DB insert on completion.
6. On stream completion → insert assistant message into `chat_messages`; increment `ai_usage_quota.chat_count`.

---

## 18. `GET /api/ai/chat/history` (US-9 AC6)

**Auth:** required.

### Query

| param | type | required | notes |
|-------|------|:--------:|-------|
| `conversation_id` | string | no | If omitted, returns the current (most recent) conversation. |
| `limit` | number 1..100 | no | Default 50. Most recent first. |
| `cursor` | string | no | Pagination. |

### Response — `200 OK`

```ts
interface ChatHistory {
  conversation_id: string;
  messages: Array<{
    id: string;
    role: 'user' | 'assistant';
    content: string;
    created_at: string;
  }>;
  next_cursor: string | null;
}
```

---

## 19. `DELETE /api/ai/chat/history` (US-9 AC6)

**Auth:** required.

### Query

| param | type | required | notes |
|-------|------|:--------:|-------|
| `conversation_id` | string | no | If omitted, clears ALL conversations. Otherwise just that one. |

### Response

`204 No Content`.

---

## 20. `POST /api/measurements` (US-10)

**Auth:** required.

### Request

```ts
interface CreateMeasurementBody {
  metric: 'weight' | 'waist' | 'hip' | 'chest' | 'body_fat';
  value: number;                       // metric units: kg for weight, cm for waist/hip/chest, % for body_fat
  unit: 'kg' | 'cm' | 'percent';        // server validates metric/unit pair
  measured_at: string;                  // ISO-8601, defaults to now()
  confirm_out_of_range?: boolean;       // for typo guard (US-10 AC6); see below
}
```

### Response — `201 Created`

```ts
interface Measurement {
  id: string;
  user_id: string;
  metric: string;
  value: number;
  unit: string;
  measured_at: string;
  created_at: string;
}
```

If `metric=weight` and the new value differs from `profiles.weight_kg` by > 1 kg, the response body adds:

```ts
{
  weight_change_alert: {
    delta_kg: number;
    suggest_recalculate_goals: true;     // US-10 AC5
  }
}
```

The client renders a banner; the actual recompute is the user's choice (`PATCH /api/profile` + `PATCH /api/goals { apply_suggested: true }`).

### Errors

- `400 validation-failed` — wrong metric/unit pair.
- `409 conflict` — out-of-range (weight ≤ 0, body_fat > 60% / < 1%) AND `confirm_out_of_range !== true`. `details: { metric, value, suggested_range: [min, max] }`. Client re-prompts; on retry sets `confirm_out_of_range: true`.

---

## 21. `GET /api/measurements` (US-10 AC3)

**Auth:** required.

### Query

| param | type | required | notes |
|-------|------|:--------:|-------|
| `metric` | enum | no | Filter to one metric. |
| `from` | `YYYY-MM-DD` | no | Inclusive. |
| `to` | `YYYY-MM-DD` | no | Inclusive. |
| `cursor` | string | no | |
| `limit` | number 1..100 | no | Default 50. |

### Response — `200 OK`

```ts
interface MeasurementList {
  measurements: Measurement[];
  next_cursor: string | null;
}
```

---

## 22. `PATCH /api/measurements/:id` (US-10 AC4)

**Auth:** required.

### Request

```ts
interface PatchMeasurementBody {
  value?: number;
  measured_at?: string;
  confirm_out_of_range?: boolean;
}
```

`metric` and `unit` are immutable; create a new measurement instead.

### Response — `200 OK`

`Measurement`.

### Errors

- Same as `POST /api/measurements`.
- `404 not-found`.

---

## 23. `DELETE /api/measurements/:id` (US-10 AC4)

**Auth:** required.

### Response

`204 No Content`.

### Errors

- `404 not-found`.

---

## 24. `GET /api/quota` (US-12 AC1, US-14 AC1)

**Auth:** required.

### Response — `200 OK`

```ts
interface QuotaResponse {
  tier: 'free' | 'paid';
  day_local: string;                  // YYYY-MM-DD
  resets_at: string;                  // next local-midnight in UTC
  photo: {
    used: number;
    limit: number | null;             // null on paid tier
    remaining: number | null;
  };
  chat: {
    used: number;
    limit: number | null;
    remaining: number | null;
  };
}
```

---

## 25. `GET /api/account/export` (US-11 AC2)

**Auth:** required.

### Response — `200 OK`

`Content-Type: application/json`
`Content-Disposition: attachment; filename="biteiq-export-<user_id>-<YYYY-MM-DD>.json"`

```ts
interface AccountExport {
  exported_at: string;
  user: {
    id: string;
    email: string;
    created_at: string;
  };
  profile: ProfileWithGoals['profile'];
  goals: GoalsResponse;
  goals_history: Array<GoalsResponse & { effective_until: string | null }>;
  meals: Meal[];
  measurements: Measurement[];
  chat_history: Array<{
    conversation_id: string;
    messages: Array<{ role: string; content: string; created_at: string }>;
  }>;
  // Photo files are NOT inlined; their storage paths + signed URLs (24h) are referenced from meals[i].photo_storage_path.
  photo_signed_urls: Record<string, string>;   // path → signed URL, 24h validity
}
```

For MVP, generation is synchronous (US-11 AC2 — small data volumes per A11). Endpoint times out at 30s; if exceeded, return `503 service-unavailable` and we'll add an async path later.

---

## 26. `DELETE /api/account` (US-11)

**Auth:** required.

### Request

```ts
interface DeleteAccountBody {
  confirmation: 'DELETE';              // must be exact string (US-11 AC3)
}
```

### Response

`204 No Content`. Client clears session cookies and redirects to landing.

### Errors

- `400 validation-failed` — `confirmation !== 'DELETE'`.
- `502 upstream-failed` — partial deletion (US-11 AC6). `details: { phase: 'storage' | 'auth' | 'rows', completed: string[] }`. Server retries internally up to 3× before returning this.

### Server logic (in order, transactional where possible)

1. Delete all storage objects under `food-photos/{user_id}/`.
2. Delete rows in: `chat_messages`, `meal_items`, `meals`, `body_measurements`, `goals_history`, `goals`, `ai_usage_quota`, `food_favorites`, `profiles`. (`ai_photo_cache` is global — NOT deleted.)
3. Delete the auth user via `supabase.auth.admin.deleteUser(user_id)` (service-role).
4. Return `204`. Client signs out (US-11 AC5).

---

## 27. `POST /api/account/upgrade-stub` (US-12 AC3, US-14 AC3)

**Auth:** required.

### Request

```ts
{}   // empty
```

### Response — `202 Accepted`

```ts
interface UpgradeStubResponse {
  status: 'waitlisted';
  message: 'Paid tier coming soon — we\'ll email you.';
}
```

Server records the user_id in a `paid_tier_waitlist` table for later marketing reach-out. No payment, no plan change.

---

## 28. `POST /api/foods/search` (US-6 AC2)

**Auth:** required.

### Request

```ts
interface FoodSearchBody {
  query: string;                       // 1..100 chars
}
```

### Response — `200 OK`

```ts
interface FoodSearchResponse {
  history: Array<{                     // user's previously logged foods, deduplicated by name+brand
    food_name: string;
    brand: string | null;
    serving_grams: number;
    calories: number;
    protein_g: number;
    carbs_g: number;
    fat_g: number;
    last_used_at: string;
  }>;
  external: Array<{                    // Open Food Facts results
    food_name: string;
    brand: string | null;
    serving_grams: number | null;
    calories: number | null;
    protein_g: number | null;
    carbs_g: number | null;
    fat_g: number | null;
    barcode: string;
  }>;
}
```

History first (per US-6 AC2). Capped at 10 results each side.

### Errors

- `502 upstream-failed` — OFF down. Returns `history` only with a 200 if history is non-empty; only 502s if both sides fail.

---

## 29. `POST /api/foods/favorite` (US-6 AC4)

**Auth:** required.

### Request

```ts
interface CreateFavoriteBody {
  food_name: string;
  brand?: string | null;
  serving_grams: number;
  calories: number;
  protein_g: number;
  carbs_g: number;
  fat_g: number;
}
```

### Response — `201 Created`

```ts
interface Favorite extends CreateFavoriteBody {
  id: string;
  user_id: string;
  created_at: string;
}
```

---

## 30. `GET /api/foods/favorites` (US-6 AC4)

**Auth:** required.

### Response — `200 OK`

```ts
interface FavoritesList {
  favorites: Favorite[];
}
```

---

## 31. `DELETE /api/foods/favorites/:id` (US-6 AC4)

**Auth:** required.

### Response

`204 No Content`.

---

## Notes for DEV (Phase 3)

- Validate every body with Zod; export the schemas from `lib/api/schemas.ts` and re-use them in clients (`zod-to-typescript-types` or `tsoa`-style) to avoid drift.
- All endpoints set `Cache-Control: private, no-store` except `/api/barcode/:code` which can be `public, max-age=86400`.
- Per-user rate limit (separate from daily quota): max 30 req/sec on AI endpoints (`/api/ai/*`). Use Supabase `pg_cron` + a `rate_limits` table or in-memory + Vercel Edge Config for MVP.
- The server NEVER trusts client-side `tier`. Resolve `paid_tier_flag` from `profiles` on every quota-gated request.

## Notes for QA (Phase 4)

Every error code in the table above is a test case. Specifically verify:
- `429 quota-exceeded` on the 6th photo of the day for free-tier users (US-12 AC2).
- Cache-hit photo upload does NOT decrement quota (US-12 AC6).
- RLS — log in as user A, try `GET /api/meals/:id` for user B's meal → must `404`, not `403`.
- Account deletion removes the storage objects (US-11 AC4).
- Past-date dashboard uses the goal that was active that day, not the current goal (US-13 AC4).

---

## Phase 12 — Multi-provider vision + vi-default

> **Change request appended 2026-05-08.** Implements US-MP1, US-MP2, US-I18N1, US-I18N2 from `docs/user_stories.md` "## Phase 12 — Multi-provider vision + vi-default" section. Companion schema + provider implementation outlines live in `docs/tech_design.md` "## Phase 12 — Multi-provider vision + i18n schema".
>
> **Append-only.** All §1–§31 endpoints above remain valid. Any delta to an existing endpoint is restated in this section and reflected in the change-log at the bottom.
>
> **Non-negotiable inputs:** the 6 user clarifications captured in `task_board.md` Phase 12 §"User clarifications captured 2026-05-08". Architect did NOT re-litigate them.

### 12.A. Provider abstraction (TypeScript interface contract)

The single Claude vision call inside `app/api/ai/recognize/route.ts` is replaced by a **provider chain**. DEV places provider implementations under `biteiq/lib/ai/vision/` (`gemini.ts`, `ollama.ts`, `claude.ts`, plus `index.ts` that exports `runVisionChain`). Every provider conforms to the same interface so the orchestrator is provider-agnostic:

```ts
// biteiq/lib/ai/vision/types.ts

export type VisionProviderId = 'gemini' | 'ollama' | 'claude';

export interface RecognizeHints {
  /** Slot the user said the photo belongs to (breakfast | lunch | dinner | snack). */
  mealSlotHint?: string | null;
}

export interface RecognizedItem {
  food_name: string;
  brand: string | null;
  serving_grams: number;
  calories: number;
  protein_g: number;
  carbs_g: number;
  fat_g: number;
  confidence: number;        // 0..1
}

export interface RecognizeResult {
  items: RecognizedItem[];
  low_confidence: boolean;   // true when max(confidence) < 0.5 OR items.length === 0
  /** Provider-supplied raw payload, opaque to the orchestrator. Stored in ai_photo_cache.recognition_json
   *  so QA / support can inspect what each provider returned. Should NOT contain PII. */
  raw?: unknown;
}

export interface VisionProvider {
  /** Stable id used in chain config + response payload. */
  readonly id: VisionProviderId;

  /** Run vision recognition. MUST throw on any failure (quota, network, parse, auth). */
  recognize(image: Buffer, mimeType: string, hints?: RecognizeHints): Promise<RecognizeResult>;

  /** Returns true ONLY for "provider is out of free / paid capacity right now"
   *  (HTTP 429, RESOURCE_EXHAUSTED, rate_limit_error, "quota" wording).
   *  Returns false for everything else (network, 5xx, parse, auth) so the orchestrator
   *  surfaces those as 502 and they stay visible to engineering (per user clarification 3). */
  isQuotaExhaustedError(err: unknown): boolean;
}
```

### 12.B. Orchestrator contract (`runVisionChain`)

```ts
// biteiq/lib/ai/vision/index.ts

export interface VisionChainSuccess {
  result: RecognizeResult;
  provider: VisionProviderId;          // the provider that actually served this call
  attempts: VisionProviderId[];        // ordered list of providers tried (last = winner)
}

export class AllProvidersExhaustedError extends Error {
  readonly attempts: VisionProviderId[];
  constructor(attempts: VisionProviderId[]) {
    super('All vision providers reported quota-exhausted in this request.');
    this.attempts = attempts;
  }
}

export async function runVisionChain(
  image: Buffer,
  mimeType: string,
  hints?: RecognizeHints
): Promise<VisionChainSuccess>;
```

Behavior (binding):

1. Read the chain order from `AI_VISION_PROVIDER_CHAIN` env (CSV) — defaults to `gemini,ollama,claude`. Validate at orchestrator-construction time: must be a non-empty subset of `{gemini, ollama, claude}` AND a permutation (no duplicates). Invalid config → process boots with a noisy warning and falls back to default order; **never silently swallows misconfiguration**.
2. For each provider in order: call `provider.recognize(image, mimeType, hints)`.
3. On success → return `{ result, provider: provider.id, attempts: [...allTried, provider.id] }`. Do **not** call subsequent providers.
4. On thrown error: call `provider.isQuotaExhaustedError(err)`.
   - `true` → log `info` ("provider X quota-exhausted, falling back to Y"), continue to next provider.
   - `false` → re-throw the original error. Caller surfaces as `502 upstream-failed` (existing semantics — see §12.D).
5. If the loop ends without a success → throw `AllProvidersExhaustedError`. Caller surfaces as `503 all-providers-exhausted` (new code — see §12.D).

**Cost guardrail.** `runVisionChain` calls at most one provider per request *unless* an earlier one returned the quota-exhausted signal. Successful first-provider response means zero extra calls (US-MP1 AC6).

### 12.C. Provider registry default + override

| env var | default | semantics |
|---------|---------|-----------|
| `AI_VISION_PROVIDER_CHAIN` | `gemini,ollama,claude` | CSV. Order = priority. Any subset/permutation of `{gemini, ollama, claude}` is valid; duplicates or unknown names are rejected with a console warning + fallback to default. |

Ops can set `AI_VISION_PROVIDER_CHAIN=claude,gemini,ollama` (e.g. emergency rollback) without redeploying code.

### 12.D. Endpoint contract delta — `POST /api/ai/recognize`

This section restates only the parts that change versus §12 above. Cache lookup, storage upload, image hashing, multipart upload, MIME/size limits, and `cache_hit` semantics are unchanged.

**Behavioral changes:**
- Vision call goes through `runVisionChain` instead of a direct Anthropic call.
- **Per-day photo cap (5/day free) is REMOVED** (US-MP2 AC1, user clarification 4). The route handler no longer reads `quotaInfo.photo.limit` to gate the request. Cache write + storage upload still happen.
- `quota.photo_count` column on `ai_usage_quota` continues to be incremented for **observability** (not for gating). DEV may stop incrementing entirely if it simplifies code; documented in tech_design §12.4. Either way, `GET /api/quota` photo fields stay in the response shape (additive removal would break backward compat).
- The 30 req/sec/user rate-limit (tech_design §G #8) is **explicitly retained** as the only abuse guard. `429 rate-limited` still applies.

**Response — `200 OK` (additive: `provider` field):**

```ts
interface RecognitionResponse {
  cache_hit: boolean;
  photo_storage_path: string;
  image_sha256: string;
  items: Array<{
    food_name: string;
    brand: string | null;
    serving_grams: number;
    calories: number;
    protein_g: number;
    carbs_g: number;
    fat_g: number;
    confidence: number;
  }>;
  low_confidence: boolean;
  /** NEW (Phase 12). The provider that produced this recognition.
   *  - For cache hits: the provider that produced the originally-cached result
   *    (read from `ai_photo_cache.provider`; if NULL, return `'claude'` for legacy rows). */
  provider: 'gemini' | 'ollama' | 'claude';
  quota: {
    photo_used: number;
    photo_limit: number | null;        // remains in shape; for free tier this is null after Phase 12 cap removal — see §12.G.
    chat_used: number;
    chat_limit: number | null;         // chat quota unchanged
  };
}
```

**Errors (changed):**

| HTTP | code | When | `details` |
|-----:|------|------|-----------|
| 400 | `validation-failed` | unchanged | — |
| 413 | `payload-too-large` | unchanged | — |
| 415 | `unsupported-media-type` | unchanged | — |
| ~~429~~ | ~~`quota-exceeded` (kind=`photo`)~~ | **REMOVED Phase 12** — free-tier per-day photo cap deleted. Will never fire for `kind: 'photo'` from this endpoint. `kind: 'chat'` still applies on `/api/ai/chat`. | — |
| 429 | `rate-limited` | unchanged — 30 req/sec/user burst guard | `{ retry_after_seconds: number }` |
| 502 | `upstream-failed` | A provider in the chain threw a NON-quota error (network, 5xx, parse, auth). Surfaces engineering bugs. | `{ upstream: 'storage' \| 'gemini' \| 'ollama' \| 'anthropic'; provider_id?: 'gemini'\|'ollama'\|'claude'; attempts?: VisionProviderId[] }` |
| **503** | **`all-providers-exhausted`** | **NEW Phase 12.** Every provider in the chain returned the quota-exhausted signal in this single request (US-MP1 AC4). Translated user copy supplied by `messages/{vi,en}.json` (Designer's catalog). | `{ attempts: ['gemini','ollama','claude']; retry_after_seconds?: number }` |

**`502` vs `503` discriminator (binding for QA tests):**
- `503 all-providers-exhausted` ⇔ EVERY provider returned `isQuotaExhaustedError(err) === true`. Even one non-quota error short-circuits to `502`.
- `502 upstream-failed.details.upstream` names the FIRST provider that threw a non-quota error.

### 12.E. Env vars (additive — does NOT remove `ANTHROPIC_API_KEY`)

| env var | required | default | notes |
|---------|:--------:|---------|-------|
| `ANTHROPIC_API_KEY` | yes | — | unchanged. Last-fallback Claude provider. |
| `VISION_MODEL` | no | `claude-sonnet-4-6` | unchanged. Used by `claude.ts` only. Kept for backward-compat with `ai_photo_cache.model` rows. |
| `GEMINI_API_KEY` | yes | — | New. Google AI Studio key for `@google/generative-ai`. |
| `GEMINI_VISION_MODEL` | no | `gemini-2.5-pro` (see §12.J decision-log for why, not `gemini-3.1-pro-preview`) | DEV passes to `genAI.getGenerativeModel({ model })`. |
| `OLLAMA_BASE_URL` | yes | `https://ollama.com/api` | New. Ollama Cloud endpoint. Trailing slash optional — DEV normalizes. |
| `OLLAMA_API_KEY` | yes | — | New. Sent as `Authorization: Bearer <key>`. |
| `OLLAMA_VISION_MODEL` | no | `llama3.2-vision` | New. Vision-capable model from Ollama Cloud catalog. See §12.J decision-log. |
| `AI_VISION_PROVIDER_CHAIN` | no | `gemini,ollama,claude` | CSV. Validation rules in §12.C. |

DEV also appends these to `biteiq/.env.example` so local-dev onboarding stays runnable.

### 12.F. `PATCH /api/profile` — locale_preference delta

Adds **one** optional field. All other fields unchanged.

**Request — `PatchProfileBody` (additive):**
```ts
interface PatchProfileBody {
  // ...existing fields (sex, age_years, height_cm, weight_kg, activity_level, goal,
  //                     target_weight_kg, units_preference, timezone) ...
  locale_preference?: 'vi' | 'en';   // NEW Phase 12. Validated with z.enum(['vi','en']).
}
```

**Response — `200 OK` (additive — see §12.G for the shape change to `ProfileWithGoals`):**
The standard `ProfileWithGoals` response now includes `profile.locale_preference`.

**Errors:**
- `400 validation-failed` — `locale_preference` not in `{'vi','en'}`. `details: [{ field: 'locale_preference', message: 'Must be one of: vi, en' }]`.
- All existing errors unchanged.

**Server logic:**
- Validates `locale_preference` with Zod `z.enum(['vi', 'en']).optional()`.
- Persists to `profiles.locale_preference` (column added in migration `010_locale_preference` — see tech_design §12.1).
- Does **not** trigger any quota / goal recompute side-effect (cheap profile column).

### 12.G. `GET /api/profile` — response delta

Adds `locale_preference` to `Profile` shape inside `ProfileWithGoals`. Existing endpoint behavior unchanged.

```ts
interface Profile {
  // ...existing fields (user_id, sex, age_years, height_cm, weight_kg, activity_level, goal,
  //                     target_weight_kg, units_preference, timezone, paid_tier_flag,
  //                     disclaimer_acknowledged, created_at, updated_at) ...
  locale_preference: 'vi' | 'en';   // NEW Phase 12. NOT NULL with DB default 'vi'.
}
```

For users created before migration `010_locale_preference` ran, the DB DEFAULT backfills `'vi'` automatically — no app-side migration code needed (per user clarification 5: vi is the default for new sessions; pre-existing users effectively become vi-default until they opt into en in Settings).

This same shape is returned by `POST /api/profile` (201) and `PATCH /api/profile` (200).

### 12.H. Settings page contract reminder

The Settings page (US-I18N2) reads the locale via `GET /api/profile` (already in spec) and writes it via `PATCH /api/profile { locale_preference }`. **No new endpoint is added.** Designer owns the LanguageSwitcher UI shape; this contract just specifies the field flowing in/out.

### 12.I. Quota-state design (Option A vs Option B — Architect's pick)

User clarification 4 removed the per-day photo cap but the multi-provider chain introduces a NEW need: track which providers are currently in cooldown so the orchestrator can short-circuit faster than blowing through them serially every request. Two options were evaluated:

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A. In-memory + Vercel Edge Config** (per-provider cooldown timestamps) | The orchestrator keeps `Map<VisionProviderId, { cooldownUntil: number }>` in module-level memory; on quota-exhausted, sets `cooldownUntil = now + 60s`; subsequent requests skip that provider until expiry. Edge Config holds a global override blob for ops (`{provider, cooldownUntil}[]`) read every cold start. | Zero DB hits per call. Aligns with existing rate-limit infra (tech_design §G #8 already uses in-memory + Edge Config). Lost-on-cold-start is acceptable — at most one extra request blows through the chain after a deploy. | Per-instance memory, not globally consistent — ~30s of stale cooldown after a serverless cold-start. Acceptable noise. |
| **B. `ai_usage_quota.provider_quota_state jsonb`** | Persist `{ gemini: { cooldown_until: timestamptz }, ollama: { ... }, claude: { ... } }` per (user, day) row. Read/write on every recognize call. | Durable across cold starts. Per-user observability (e.g. "gemini exhausted for user X 3× today"). | +1 DB read + +1 DB write per recognize call → adds ~30–80ms latency, eats Supabase free-tier connection budget for 1k MAU target. Per-user quota tracking is overkill: provider quota is a **provider-side** signal, not a user-side one. |

**Architect's pick: Option A.** Provider quota-exhaustion is a transient global signal (Gemini's free quota is a daily-rolling limit on the *Google project*, not per-end-user) — tracking it per-user-row is data-modeling at the wrong cardinality. Option A piggy-backs on the existing in-memory + Edge Config infrastructure already shipped for the 30 req/sec rate-limit (tech_design §G #8) and adds ~0ms to the happy path.

**Trade-off accepted:** after a Vercel cold start, ONE request may still attempt a provider that was in cooldown right before. Worst case it hits 429 from upstream and the orchestrator falls back as designed. No correctness loss, only one wasted call per cold start per provider.

→ No DB migration for quota-state. Migration `011_*` is therefore **not** created (see tech_design §12.2). DEV can implement a small `lib/ai/vision/cooldown.ts` module with `getCooldown(providerId): number | null` + `setCooldown(providerId, durationMs): void` + Edge Config integration TBD-by-DEV.

### 12.J. Phase 12 Assumptions (additive to top-of-file Assumptions)

- **AR7 (Phase 12).** `AI_VISION_PROVIDER_CHAIN` env validation: invalid configs (duplicates, unknown ids, empty) produce a startup `console.warn` and the orchestrator silently falls back to `gemini,ollama,claude`. We do NOT crash the process — Vercel rolling deploys could otherwise take down the whole AI surface from a typo'd env var.
- **AR8 (Phase 12).** `ai_photo_cache.provider` (added in tech_design §12.3) is `text NULL` for backward-compat with existing 0 rows; new rows always populate it. On read, if NULL → cache row is treated as `provider='claude'` (legacy rows pre-Phase-12 came exclusively from the Claude pipeline).
- **AR9 (Phase 12).** `provider` field in the recognize response is **not** considered PII. Safe to log and surface in client-side debug overlays.
- **AR10 (Phase 12).** `locale_preference` is per-account, not per-device. The same user logging in on phone+desktop sees the same language (the trade-off vs. cookie-only persistence — chosen per user clarification 5: "preference persisted to `profiles.locale_preference`").

### 12.K. QA test additions (binding)

Phase 12 QA must verify:
- `provider` field present on every successful `/api/ai/recognize` response, value matches the actually-used provider.
- Mock Gemini → 429 → next call Ollama → 200 → response says `provider: 'ollama'`.
- Mock all 3 → quota errors → response is `503 all-providers-exhausted`, NOT `502 upstream-failed`, NOT `429 quota-exceeded`.
- Mock Gemini → generic 500 → response is `502 upstream-failed.details.upstream='gemini'` (NO fallback, NO 503).
- Free-tier user posts 6 photos in succession → all return 200 (or 429 rate-limit if faster than 30/s — never `429 quota-exceeded` photo-kind).
- Cache-hit response includes `provider` reflecting the originally-cached provider.
- `PATCH /api/profile { locale_preference: 'fr' }` → `400 validation-failed`.
- `PATCH /api/profile { locale_preference: 'vi' }` → `200 OK`, subsequent `GET /api/profile` returns `locale_preference: 'vi'`.

---

## Phase 12 change-log

> Compact ops/audit row. Mirrors the section above; do not edit retroactively.

| Date | Endpoint(s) | Change | Backward-compat? | Reason |
|------|-------------|--------|:----------------:|--------|
| 2026-05-08 | `POST /api/ai/recognize` | Add `provider: 'gemini'\|'ollama'\|'claude'` to 200 response. | yes (additive) | US-MP1 AC3 — observability for which provider served each call. |
| 2026-05-08 | `POST /api/ai/recognize` | Remove free-tier 5/day photo cap. `429 quota-exceeded` (kind=`photo`) no longer fires. | partial — clients that branched on `kind:'photo'` 429 must remove that branch; chat 429 unchanged. | US-MP2 AC1 — multi-provider chain replaces per-user cap as the cost guardrail. |
| 2026-05-08 | `POST /api/ai/recognize` | New `503 all-providers-exhausted` error code. | yes (new code only fired post-Phase-12) | US-MP1 AC4 — distinct copy from "your daily photos used up" (US-MP2 AC6). |
| 2026-05-08 | `PATCH /api/profile`, `GET /api/profile`, `POST /api/profile` | Add `locale_preference: 'vi'\|'en'` field. DB default `'vi'`. | yes (additive; old clients ignore unknown field) | US-I18N1 AC1, US-I18N2 AC1/AC3 — persist user's UI language choice across sessions/devices. |
| 2026-05-08 | (env) | Add `GEMINI_API_KEY`, `GEMINI_VISION_MODEL`, `OLLAMA_BASE_URL`, `OLLAMA_API_KEY`, `OLLAMA_VISION_MODEL`, `AI_VISION_PROVIDER_CHAIN`. `ANTHROPIC_API_KEY` retained. | yes (additive — Vercel deploy needs them set; documented in DevOps redeploy gate Phase 12.5) | US-MP1 AC1/AC2 — fallback chain config. |
| 2026-05-08 | (rate limit) | 30 req/sec/user on `/api/ai/*` **explicitly retained** unchanged from tech_design §G #8. | yes (no change) | User clarification 4 — burst guard stays after per-day cap removal. |
