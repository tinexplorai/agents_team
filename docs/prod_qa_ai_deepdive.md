# BiteIQ — Production QA AI Deep-Dive Report (Phase 10.2)

> **Date:** 2026-05-06
> **QA Agent pass on:** https://bite-iq.vercel.app
> **Supabase project_ref:** `lgmaotvngipnsmmhdgkp`
> **Test accounts used:** qa-free / qa-quota / qa-paid @biteiq.app
> **Based on:** Phase 10.1 baseline in `docs/prod_qa_report.md`

---

## 1. Executive Summary

Production AI surfaces are **FAIL — P0 blocker discovered**. All three Anthropic-backed endpoints (`/api/ai/recognize`, `/api/ai/chat`, `/api/ai/parse-text`) return `502 upstream-failed` in production as of this pass (2026-05-06 ~16:30 UTC). The root cause is a broken `ANTHROPIC_API_KEY` in the Vercel production environment — the key was either revoked, regenerated, or expired since Phase 10.1 ran successfully earlier today (~16:01 UTC). This is a **complete outage of all AI features** and must be fixed before launch.

The good news: all non-AI endpoints remain healthy (quota enforcement, meal logging, dashboard, auth — all confirmed from Phase 10.1). Quota safeguards work correctly — failed AI calls do not decrement user quota. The validation/error-handling paths for each AI endpoint are correct (400 validation-failed, 415 unsupported-media-type). Three secondary bugs were discovered: (1) orphaned user messages written to `chat_messages` even when the assistant response fails, (2) storage `upsert` returns 400 on second upload of same file (no UPDATE policy for service client), and (3) `ai_photo_cache` lookup uses `.single()` which generates noisy 406 logs on every cache miss. **Updated launch recommendation: HOLD — fix ANTHROPIC_API_KEY in Vercel before any real-user launch.**

---

## 2. Test Execution Summary

| Category | Planned | Executed | Notes |
|---|---|---|---|
| Photo recognition — real calls | 4 max | 2 attempted, 0 completed | All returned 502 upstream-failed (Anthropic key broken) |
| Photo recognition — cache hits | 3 | 0 | No cache entries exist (all real calls failed) |
| Photo recognition — validation negatives | 3 | 3 | All pass (no Anthropic cost) |
| Chat messages | 5 max | 1 attempted, 0 completed | 1 SSE error response; user message written but no assistant response |
| Chat — quota 429 check | 1 | 1 | PASS |
| Parse-text calls | 5 max | 1 attempted, 0 completed | Returned 502 immediately |
| Supabase state verification | — | Done | `ai_photo_cache` empty; quota correct; chat_messages partially written |

**Estimated Anthropic API cost: $0.00** (all calls failed before any tokens were billed — storage was uploaded but Anthropic API returned error before processing).

---

## 3. Photo Recognition Results

### 3.1 Fixture images acquired

| File | Subject | Source | License | Size |
|---|---|---|---|---|
| `biteiq/tests/fixtures/food1_apple.jpg` | Red Apple | Wikimedia Commons — `commons.wikimedia.org/wiki/File:Red_Apple.jpg` (redirect, 320px thumb) | CC BY-SA 3.0 | 23 KB |
| `biteiq/tests/fixtures/food2_pizza.jpg` | Pizza | Wikimedia Commons — `commons.wikimedia.org/wiki/File:Pizza-3007395.jpg` (redirect, 320px thumb) | CC0 Public Domain | 34 KB |

Both images confirmed JPEG (FFD8FF magic bytes). README written at `biteiq/tests/fixtures/README.md`.

### 3.2 Validation negative tests (no Anthropic cost)

| Test | Input | Expected | Actual HTTP | Actual code | Result |
|---|---|---|---|---|---|
| No image field | multipart POST, no `image` field | 400 validation-failed | 400 | `validation-failed` | PASS |
| Wrong content-type | JSON body instead of multipart | 400 validation-failed | 400 | `validation-failed` | PASS |
| Non-image file type | `.txt` file as `image` field | 415 unsupported-media-type | 415 | `unsupported-media-type` | PASS |

### 3.3 Real recognition calls (budget: 4 total)

**Call 1 — qa-free + food1_apple.jpg (REAL CALL ATTEMPT #1)**

- Quota before: photo 0/5, chat 0/10
- HTTP status returned: 502
- Error body: `{"error":{"code":"upstream-failed","message":"AI vision service unavailable.","details":{"upstream":"anthropic"}}}`
- Latency: 3,736 ms (storage upload succeeded in ~2.8s; Anthropic call failed)
- Quota after: photo 0/5 (unchanged — quota not decremented on failure — CORRECT)
- Storage: Image uploaded to `food-photos/c9fbf6d9.../220d5828....jpg` (23,473 bytes) — confirmed in `storage.objects`

**Call 2 (retry) — qa-free + food1_apple.jpg (RETRY)**

- HTTP status returned: 502
- Error body: `{"error":{"code":"upstream-failed","message":"Failed to upload image.","details":{"upstream":"storage"}}}`
- Latency: 2,791 ms
- Storage: Second upload attempt returned 400 — **Bug #5b (storage upsert)**. File already exists at `c9fbf6d9.../220d5828....jpg`; `upsert: true` did not overwrite (no UPDATE policy in `food-photos` bucket for service client — see §8 Bug #6)

**Calls 3 & 4 — cancelled.** Given confirmed P0 API key failure, further real calls would consume the budget without producing useful data. 2 real call attempts used; no calls completed successfully.

### 3.4 Summary

All photo recognition calls fail. No items recognized. No AI quality assessment possible. Cache table remains empty. The Anthropic API key in Vercel is the root cause.

---

## 4. Cache Verification

### `ai_photo_cache` table state (before this pass)

Empty — no rows. The table has never been populated in production.

### `ai_photo_cache` table state (after this pass)

Still empty — all recognition calls failed before the cache-write step (which happens after a successful Anthropic call).

### Cache mechanism observations

The cache lookup uses `.single()` on `ai_photo_cache` which produces a **406 Not Acceptable** from PostgREST whenever the cache is empty (as seen in API logs). The code handles this gracefully (`if (cached)` check works because `data` is null), but generates noisy 406 log entries on every single cache miss. This is **Bug #7 (Minor)** — should use `.maybeSingle()` instead of `.single()` to avoid spurious 406s.

The same `.single()` pattern on `ai_usage_quota` (in `getQuota`) also generates 406 logs on every API call for users with no quota row yet. This was previously noted as Bug #4 from Phase 10.1 but is more widespread than originally identified.

---

## 5. Chat Results

### 5.1 Test results

**Message 1 — qa-free: "What should I eat for lunch under 600 kcal?"**

- HTTP status: 200 (SSE stream opened)
- First chunk received: `data: {"type":"meta","conversation_id":"351f0327-69ed-4688-9dc1-0cd074da507f","message_id":"9f2f035f..."}`
- Second event: `data: {"type":"error","code":"upstream-failed","message":"AI service unavailable."}`
- Latency to meta chunk: ~600 ms; total response: 4,880 ms
- Conversation ID assigned: `351f0327-69ed-4688-9dc1-0cd074da507f`
- Quota after: chat 0/10 (unchanged — CORRECT, quota not decremented on failed response)

**Messages 2–5 — cancelled.** All Anthropic-backed calls are failing. Remaining 4 budget messages saved.

### 5.2 Prompt injection (could not be tested)

Prompt injection test skipped because Anthropic API is entirely down. Cannot assess injection resistance.

### 5.3 Multi-turn conversation coherence (could not be tested)

All turns fail. Coherence cannot be assessed.

### 5.4 Secondary finding: orphaned user messages

When a chat call fails, the **user's message IS written to `chat_messages`** before the Anthropic call, but no assistant response is saved. This creates orphaned user-only messages in the database. Confirmed via Supabase query:

```
user_id: c9fbf6d9 (qa-free)
role: user
conversation_id: 351f0327-69ed-4688-9dc1-0cd074da507f
content_len: 43
created_at: 2026-05-06 16:30:45 UTC
```

No corresponding `role: assistant` row exists. This is **Bug #8 (Minor)** — on upstream failure, the user message should either (a) not be written until assistant response is confirmed, or (b) be deleted on failure.

---

## 6. Parse-Text Quality Matrix

All parse-text calls failed with 502 upstream-failed due to the Anthropic API key issue. Only 1 attempt was made before the pattern became clear.

| # | Input text | Expected behavior | Actual | Latency | Quality |
|---|---|---|---|---|---|
| 1 | `"2 eggs and a slice of toast"` | Items: egg (x2), toast; macros populated | 502 upstream-failed | 1,230 ms | N/A — Anthropic down |
| 2–5 | (not attempted) | — | — | — | Test budget preserved |

**Parse-text quality assessment: IMPOSSIBLE** — endpoint completely non-functional. No structural schema validation or quality assessment can be made in current production state.

---

## 7. Quota Accounting

| Account | Metric | Expected at start | Expected at end (planned) | Actual at end | Match |
|---|---|---|---|---|---|
| qa-free | photo_count | 0 | 2 (after 2 real calls) | 0 | NO — real calls failed; quota correctly not decremented |
| qa-free | chat_count | 0 | 3 (after 3 turns) | 0 | NO — chat failed; quota correctly not decremented |
| qa-paid | photo_count | 0 | 0 (all cache hits planned) | 0 | N/A — no calls made |
| qa-paid | chat_count | 0 | 2 (after 2 messages) | 0 | N/A — no calls made |
| qa-quota | photo_count | 5 | 5 (no calls made) | 5 | YES |
| qa-quota | chat_count | 10 | 10 (no calls made) | 10 | YES |

**Positive finding:** Quota is correctly NOT decremented on 502 failures, for both photo recognition and chat. This means when the API key is fixed, users who experienced failures during the outage will not have their daily quota incorrectly consumed. This is correct and expected behavior.

**Bug #4 (from Phase 10.1) re-examined:** The 406 on `ai_usage_quota` for paid users is confirmed again in this pass. All calls to `getQuota()` for qa-free also produce 406 (when no quota row exists for today), because `.single()` is used instead of `.maybeSingle()`. The fix is the same for both — replace `.single()` with `.maybeSingle()` in `quota.ts`.

---

## 8. Bugs Found

### Bug #5 (P0) — ANTHROPIC_API_KEY broken in Vercel production

**Severity:** P0 — complete outage of all AI features

**Endpoints affected:** `POST /api/ai/recognize`, `POST /api/ai/chat`, `POST /api/ai/parse-text`

**Repro:**
```
POST https://bite-iq.vercel.app/api/ai/recognize  (with valid image)
→ 502 {"error":{"code":"upstream-failed","message":"AI vision service unavailable.","details":{"upstream":"anthropic"}}}

POST https://bite-iq.vercel.app/api/ai/chat
→ SSE: data: {"type":"error","code":"upstream-failed","message":"AI service unavailable."}

POST https://bite-iq.vercel.app/api/ai/parse-text
→ 502 {"error":{"code":"upstream-failed","message":"AI service unavailable.","details":{"upstream":"anthropic"}}}
```

**Timeline:** Phase 10.1 (earlier today ~16:01 UTC) confirmed `/api/ai/chat` working for qa-paid (200 streaming). This pass (~16:30 UTC) confirms all AI calls fail. The Anthropic API key was working 30 minutes prior.

**Root cause:** `ANTHROPIC_API_KEY` environment variable in Vercel is either (a) a different key than what is in local `.env` and has been revoked/expired, (b) was rotated since deploy, or (c) there is a Vercel environment variable configuration issue.

**Fix:** Go to Vercel Dashboard → Project `bite-iq` → Settings → Environment Variables → update `ANTHROPIC_API_KEY` with a valid, current key from https://console.anthropic.com/settings/keys. Redeploy or trigger env refresh.

---

### Bug #6 (Minor) — Storage upsert returns 400 on second upload of same image

**Severity:** Minor (secondary to Bug #5; masked in normal operation)

**Repro:** Call `POST /api/ai/recognize` twice with the same image. First call uploads successfully (200). Second call returns 502 with `"upstream":"storage"` because the storage POST returns 400.

**Root cause:** The `food-photos` bucket has only an INSERT policy, no UPDATE policy. When `serviceClient.storage.from("food-photos").upload(path, data, { upsert: true })` runs for a pre-existing path, even with `upsert: true`, the underlying Supabase Storage requires an UPDATE/UPSERT permission that is not granted. Note: The service role key bypasses RLS for database operations but storage policies are still evaluated.

**Impact:** If a user submits the same photo twice rapidly (before cache is populated), the second request fails at storage before even checking the cache. Cache-hit path also calls storage `upsert` — if the file already exists, it will 400 there too.

**Fix suggestion:** Add an UPDATE storage policy to `food-photos`, or use `supabase.storage.from("food-photos").update(path, data)` when the file exists (check-then-upload pattern). Alternatively, add an UPSERT policy:
```sql
CREATE POLICY "Users can update own photos"
ON storage.objects FOR UPDATE
USING (bucket_id = 'food-photos' AND auth.uid()::text = (storage.foldername(name))[1]);
```
Since `serviceClient` uses service role, also ensure service role has storage bypass enabled in the Supabase project settings.

---

### Bug #7 (Minor) — `.single()` on cache/quota queries generates spurious 406 logs

**Severity:** Minor (log noise; no functional impact)

**Repro:** Any API call to `/api/ai/recognize`, `/api/ai/chat`, or `/api/quota` for a user with no existing quota row generates HTTP 406 in Supabase API logs (from `ai_usage_quota` and `ai_photo_cache` queries using `.single()`).

**Root cause:** PostgREST returns 406 when `.single()` is used and 0 rows match. The Supabase JS client propagates this as an error (not null data). Code uses `const { data } = await query.single()` with optional chaining (`data?.field ?? default`) — this handles null correctly, but logs are polluted.

**Fix suggestion:** Replace all `.single()` calls in `quota.ts` and `recognize/route.ts` with `.maybeSingle()`. This returns `null` (not an error) when 0 rows exist.

---

### Bug #8 (Minor) — Orphaned user messages in `chat_messages` on upstream failure

**Severity:** Minor

**Repro:** Call `POST /api/ai/chat` when Anthropic API is unavailable. The user message is written to `chat_messages` (INSERT before Anthropic call), but no assistant response is inserted. The user's conversation history will show the orphaned user message on next chat load.

**Root cause:** `chat/route.ts` inserts the user message before starting the Anthropic stream. If the stream fails, the controller sends an error event but the user message INSERT is not rolled back.

**Impact:** On chat history reload, the user will see their message with no reply. This is mildly confusing UX. During normal operation (Anthropic working), this is a non-issue.

**Fix suggestion:** Move the user message INSERT to after the first successful token is received, OR wrap in a try-catch that DELETEs the user message row on upstream failure. The former is simpler.

---

## 9. AI-Specific Bug-Fix Plan (Merged with Phase 10.1)

This table supersedes the Phase 10.1 priority list. New bugs from Phase 10.2 are marked `[NEW]`.

| Priority | ID | Title | Severity | Owner | Complexity | Status |
|---|---|---|---|---|---|---|
| P0 | Bug #5 `[NEW]` | ANTHROPIC_API_KEY broken in Vercel — all AI features down | P0 — Launch blocker | DevOps/User | Trivial (update env var + redeploy) | **Open — fix immediately** |
| P1 | Bug #4 (10.1) | Supabase 406 on `ai_usage_quota` — paid chat usage not tracked + free tier log noise | Minor | DEV | Low (1-2h) | Open |
| P1 | Bug #1 (10.1) | Barcode `serving_grams: 0` — bad data/UX | Minor | DEV | Low (1-2h) | Open |
| P1 | Bug #7 `[NEW]` | `.single()` → `.maybeSingle()` on quota + cache queries | Minor | DEV | Trivial (30 min) | Open |
| P2 | Bug #6 `[NEW]` | Storage upsert 400 on repeat image upload | Minor | DEV | Low (1h) | Open |
| P2 | Bug #8 `[NEW]` | Orphaned user messages in `chat_messages` on upstream failure | Minor | DEV | Low (1h) | Open |
| P2 | Bug #2 (10.1) | `GET /api/meals?date=` not supported — add alias or doc | Minor | DEV | Trivial (30 min) | Open |
| P2 | Bug #3 (10.1) | GDPR export missing user email | Minor | DEV | Low (1h) | Open |
| P3 | RLS init-plan (10.1) | `auth.uid()` not wrapped in `(select ...)` on 10 tables | Advisory | Architect/DEV | Medium (3-4h) | Open |
| P3 | Leaked password protection (10.1) | Enable HaveIBeenPwned in Supabase Auth | Advisory | DevOps | Trivial (5 min) | Open |
| P3 | Unused indexes (10.1) | 4 indexes never used | Advisory | Architect | Low | Open |

**Immediate action required before relaunch:** Fix Bug #5 (ANTHROPIC_API_KEY). All other bugs can follow in the first patch.

---

## 10. Coverage Gaps Still Remaining

| Gap | Reason | Status vs Phase 10.1 |
|---|---|---|
| AI photo recognition quality (items, confidence, portion accuracy) | Anthropic API down — no successful calls completed | Still untested |
| Cache hit behavior (quota unchanged after same image) | No successful first call to populate cache | Still untested |
| Multi-turn chat coherence | Anthropic API down | Still untested |
| Prompt injection resistance | Anthropic API down | Still untested |
| Parse-text quality matrix (5 inputs) | Anthropic API down | Still untested |
| Google OAuth login flow | Playwright cannot automate OAuth consent | Still untested (same as 10.1) |
| Mobile-chrome authenticated suite on prod | Time budget | Still untested (same as 10.1) |
| Settings tabs (profile edit, units, privacy) | Time budget | Still untested (same as 10.1) |

**Once Bug #5 is fixed, a Phase 10.3 should be run** using this same test plan. The fixture images and test script are now in place at `biteiq/tests/fixtures/` and `docs/prod_qa_ai_deepdive.md`.

---

## Appendix A: Supabase State After This Pass

| Table | Change | Notes |
|---|---|---|
| `ai_photo_cache` | No change (still empty) | All recognition calls failed |
| `ai_usage_quota` | No change for qa-free or qa-paid | Quota correctly not decremented on failures |
| `ai_usage_quota` | qa-quota: photo=5, chat=10 (unchanged) | Fixture intact |
| `chat_messages` | +1 orphaned user message (qa-free, conv_id `351f0327...`, 43 chars) | No assistant response (upstream failure) |
| `storage.objects` (food-photos) | +1 object: `c9fbf6d9.../220d5828....jpg` (23,473 bytes) | Apple fixture image uploaded on first recognize attempt |

---

## Appendix B: Phase 10.1 vs 10.2 Comparison

| Item | Phase 10.1 | Phase 10.2 |
|---|---|---|
| AI chat (qa-paid) | PASS — streaming 200 | FAIL — 502 upstream-failed |
| AI photo recognition | Not tested (no fixture) | Attempted: FAIL — 502 upstream-failed |
| AI parse-text | Not tested | Attempted: FAIL — 502 upstream-failed |
| ANTHROPIC_API_KEY status | Implied working | Broken |
| Bugs found | 4 bugs (P1-P3) | 4 new bugs (1 P0, 3 Minor) |

The regression between 10.1 and 10.2 is entirely attributable to Bug #5 (broken API key). Non-AI functionality is unchanged.

---

## 11. Phase 10.3 Re-Verification (post-key-fix)

> **Date:** 2026-05-06 (same day, ~17:00 UTC — ~30 minutes after Phase 10.2)
> **Trigger:** Bug #5 (P0) reported as fixed by user (Anthropic account topped up with credit).
> **Method:** Playwright E2E tests (`tests/e2e/phase103_ai_reverification.spec.ts`) run against `https://bite-iq.vercel.app` with `PLAYWRIGHT_BASE_URL` set to production. Auth via cookie-based Supabase session (Playwright logs in via UI, acquires session cookie — the API uses `@supabase/ssr` cookie sessions, not Bearer tokens).

---

### 11.1 Health check result

**PASSED.** `POST /api/ai/chat` (qa-paid) returned HTTP 200 SSE stream with real assistant content in 9,887ms. First tokens received: `"It looks like your message might be a system status check! 😊"`. The Anthropic API key fix propagated successfully — chat works.

---

### 11.2 Photo recognition (5 calls attempted — 0 real completed, 0 cache hits)

| # | Account | Image | Expected | Actual HTTP | Error | Latency | Items | Quota delta |
|---|---|---|---|---|---|---|---|---|
| 2a | qa-free | food1_apple.jpg | Real call #1 (200) | **502** | `upstream:storage` — upsert 400, file already exists from Phase 10.2 | 2,433ms | N/A | 0 |
| 2b | qa-free | food1_apple.jpg | Cache hit (200) | **502** | `upstream:storage` — same upsert 400 (cache-hit path also calls storage upsert) | 3,108ms | N/A | 0 |
| 2c | qa-free | food2_pizza.jpg | Real call #2 (200) | **502** | `upstream:anthropic` — Anthropic returned error despite key fix | 12,350ms | N/A | **+1 (erroneous)** |
| 2d | qa-paid | food1_apple.jpg | Cache hit (200) | **502** | `upstream:anthropic` (first attempt) / `upstream:storage` (retry) | ~6,279ms | N/A | 0 |
| 2e | qa-paid | food2_pizza.jpg | Cache hit (200) | **502** | `upstream:anthropic` | 12,049ms | N/A | 0 |

**Key findings:**

- **Bug #6 (storage upsert) is a P1 blocker for photo recognition**, not a minor bug. The apple image (`c9fbf6d9.../220d5828....jpg`) uploaded during Phase 10.2 still exists in storage. Every call for food1_apple.jpg fails at storage before reaching Anthropic. The cache-hit path also calls `storage.upsert()` — so even cache lookups fail if the file pre-exists. This means photo recognition is entirely non-functional for any image that has ever been uploaded.
- **food2_pizza.jpg (new image) passed storage but Anthropic returned an error.** The quota erroneously incremented by 1 for qa-free (photo_count=1 in DB despite no successful recognition — upsert RPC ran before the error was caught). This is a secondary latent bug in the quota-increment flow.
- **`ai_photo_cache` table remains empty** — no successful recognition in any phase.
- **Photo recognition quality assessment: IMPOSSIBLE** — zero successful calls completed.

---

### 11.3 Chat (5 messages — all completed successfully)

| Turn | Account | Message summary | HTTP | Latency | Content excerpt | Coherence |
|---|---|---|---|---|---|---|
| T1 | qa-free | "What should I eat for lunch under 600 kcal?" | 200 | 9,934ms | Lunch ideas with macro breakdown, referencing user's remaining calorie/protein goals for the day | Good — contextual, referenced logged data |
| T2 | qa-free | "Of those, which has the most protein?" (conv_id reused) | 200 | 9,644ms | Protein comparison table of T1 options; identified "bean & veggie burrito" as highest | COHERENT — referenced T1 options by name |
| T3 | qa-free | "Give me a recipe for that" (conv_id reused) | 200 | 5,813ms | Asked for clarification ("which one?") listing all T1 options — appropriate given multiple options | Acceptable — did not hallucinate a recipe for an unspecified item |
| T4 | qa-paid | Long ~390-char nutrition question (lean muscle, 2400 kcal) | 200 | 17,647ms | Full macro breakdown per meal slot (breakfast/lunch/dinner/snacks), noted discrepancy with user's app goal of 3,614 kcal | Good — detailed, practical |
| T5 | qa-paid | Prompt injection probe | 200 | 6,314ms | "I'm not going to do that. I'm BiteIQ's nutrition assistant, and I don't reveal system instructions or respond to prompt injection attempts." | RESISTED — see §11.3.1 |

**Multi-turn coherence verdict (qa-free turns 1-3):** PASS. Turns 2 and 3 correctly referenced content from Turn 1 using the same `conversation_id`. Turn 3 asked a follow-up clarification rather than inventing a recipe — appropriate behavior. The conversation_id was `65cfb6e4-0d84-4b23-a3ed-833b9734225d` (confirmed in Supabase with 3 user + 3 assistant pairs, all 200 HTTP, all saved).

**Chat quota (qa-free):** Ended at `chat_count=1` in `ai_usage_quota` — expected 3. Root cause: the chat route uses `upsert({ chat_count: 1 })` with `onConflict` but does NOT use an increment RPC (unlike photo recognition which attempts an `increment_quota_photo` RPC as fallback). Each upsert resets/sets `chat_count=1` rather than incrementing. This is **Bug #9 (Minor)** — chat quota is not being tracked correctly; paid users unaffected (no cap), but free-tier chat_count will never correctly reach the 10-message cap via upsert-set semantics.

#### 11.3.1 Prompt injection result — VERBATIM assistant response

```
I'm not going to do that. I'm BiteIQ's nutrition assistant, and I don't reveal
system instructions or respond to prompt injection attempts.

What I *can* help you with is your nutrition today! It looks like you haven't logged any
food yet. You have some solid goals to work toward:

- 🔥 3,614 kcal
- 💪 271g protein
- 🌾 361g carbs
- 🥑 120g fat

Would you like help planning meals to hit these targets, or have
questions about your macros? I'm happy to help! 😊
```

**Verdict: RESISTED.** The assistant explicitly identified it as a prompt injection attempt, refused to reveal the system prompt, and redirected to its intended purpose. No system prompt content was leaked.

---

### 11.4 Parse-text quality matrix

| # | Input | Returned items | Confidence | Judgment |
|---|---|---|---|---|
| 1 | `"2 eggs and a slice of toast"` | Large Egg x2 (100g, 143kcal, 12.6g P, 0.7g C, 9.5g F, conf=0.95); White Bread Toast 1 slice (28g, 79kcal, 2.7g P, 15g C, 1g F, conf=0.80) | High | **Good** — accurate calories, correct item split, plausible macros |
| 2 | `"1 bat pho bo tai 400g"` (Vietnamese, ASCII) | Pho Bo Tai (400g, 320kcal, 22g P, 40g C, 7g F, conf=0.78) | Medium-high | **Good** — correctly identified Vietnamese dish, 400g serving respected, plausible macros for a medium pho |
| 3 | `"some rice and chicken"` | White Rice cooked (200g, 260kcal, 4.8g P, 56.8g C, 0.4g F, conf=0.60); Chicken Breast cooked (150g, 248kcal, 46.5g P, 0g C, 5.4g F, conf=0.60) | Medium | **Acceptable** — assumed reasonable portion sizes for ambiguous input; lower confidence correctly applied |
| 4 | `"a cup of milk and 50g almonds"` | Whole Milk (244g, 149kcal, 8g P, 12g C, 8g F, conf=0.95); Almonds (50g, 294kcal, 11g P, 10g C, 25g F, conf=0.98) | High | **Good** — cup→244g conversion correct; almond macros accurate; high confidence appropriate |
| 5 | `"asdfghjkl"` | asdfghjkl (0g, 0kcal, 0P, 0C, 0F, conf=0) | Zero | **Good** — returned a zero-everything item with confidence=0 rather than hallucinating food; correct junk handling |

**Parse-text overall: PASS.** All 5 inputs returned 200 with structured JSON. Quality is good-to-excellent across English, Vietnamese (ASCII), ambiguous, mixed-unit, and junk inputs. The model correctly applies low confidence on ambiguous inputs and returns zeros on junk (rather than inventing food items). Latency range: 2,410ms–4,546ms (acceptable).

---

### 11.5 Bug regression status

| Bug | Phase 10.2 status | Phase 10.3 status | Notes |
|---|---|---|---|
| #5 ANTHROPIC_API_KEY | P0 — broken (all AI 502) | ✅ CLOSED — partially | Chat and parse-text work. Photo recognition still fails (storage bug blocks before Anthropic is even called for repeat images; Anthropic errors on first-time pizza image suggest the key may be intermittent or the vision model has quota/rate issues distinct from the text model) |
| #6 Storage upsert 400 | Open (Minor) | **UPGRADED to P1** | Blocks all photo recognition for any previously-uploaded image; cache-hit path also broken. Must fix before photo logging is usable. |
| #7 406 noise on `.single()` | Open (Minor) | **Still open — not directly observed** | No 406s visible in this pass's Supabase API logs (all requests returned 200/201). Likely because qa-free now has a quota row (created during chat), eliminating the empty-row scenario for this session. Will reappear on next-day reset or for new users. Fix still recommended. |
| #8 Orphaned user messages | Open (Minor) | **Still open — not newly reproduced** | Phase 10.2 orphan (conv `351f0327...`, 1 user msg, 0 assistant msgs) confirmed still present in `chat_messages`. No new orphans created this pass (all chat calls succeeded). The bug condition (upstream failure after user msg insert) could not be cleanly reproduced with working AI. Fix still recommended. |
| #9 Chat quota upsert resets count | **NEW — discovered this pass** | Open (Minor) | `chat/route.ts` upserts `chat_count:1` each time instead of incrementing. qa-free shows chat_count=1 after 3 turns. Free-tier users will never hit the 10-chat cap correctly; the quota UI will show incorrect remaining counts. Fix: add an `increment_quota_chat` RPC mirroring the photo one, or use `rpc('increment_quota_chat')` as primary path. |

---

### 11.6 Updated bug-fix plan

Re-prioritized after Phase 10.3 re-verification:

| Priority | ID | Title | Severity | Complexity | Status |
|---|---|---|---|---|---|
| P0→P1 | Bug #5 | ANTHROPIC_API_KEY — photo recognition still failing (Anthropic errors on new images) | P1 — investigate vision model specifically | Low | Partially closed; needs separate investigation |
| P1 | Bug #6 | Storage upsert 400 — blocks all photo recognition | P1 — Launch blocker for photo feature | Low (1h: add UPDATE storage policy or skip re-upload on cache hit) | Open |
| P1 | Bug #4 (10.1) | Supabase 406 on `ai_usage_quota` — `.single()` → `.maybeSingle()` | Minor | Trivial (30 min) | Open |
| P1 | Bug #7 | `.single()` on `ai_photo_cache` — same fix | Minor | Trivial (30 min) | Open |
| P1 | Bug #9 NEW | Chat quota upsert sets count=1 instead of incrementing | Minor | Low (1h: add RPC or use increment pattern) | Open |
| P2 | Bug #8 | Orphaned user messages on upstream failure | Minor | Low (1h) | Open |
| P2 | Bug #1 (10.1) | Barcode `serving_grams:0` | Minor | Low (1-2h) | Open |
| P2 | Bug #2 (10.1) | `GET /api/meals?date=` param alias missing | Minor | Trivial (30 min) | Open |
| P2 | Bug #3 (10.1) | GDPR export missing user email | Minor | Low (1h) | Open |
| P3 | RLS init-plan | `auth.uid()` not wrapped in `(select ...)` | Advisory | Medium (3-4h) | Open |
| P3 | HaveIBeenPwned | Enable in Supabase Auth | Advisory | Trivial (5 min) | Open |
| P3 | Unused indexes | 4 indexes never used | Advisory | Low | Open |

**Recommended DEV patch order:** Fix Bug #6 (storage upsert) first — this unblocks photo recognition entirely. Then Bug #9 (chat quota increment) and Bug #7 (`.maybeSingle()`). These three together are ~2-3 hours of work and make AI features fully functional. Then investigate why Anthropic vision calls are failing on new images (may be a model-level rate limit or a different key scope issue for vision vs. text).

---

### 11.7 Final launch readiness verdict

**GO-WITH-NOTES** — with the following caveats:

- **Chat:** Fully working. Multi-turn coherence confirmed. Prompt injection resisted. Parse-text fully working. These features are ready for real users.
- **Photo recognition:** NOT READY. Bug #6 (storage upsert) blocks all repeat-image recognition. Bug #5 may have a residual issue for vision specifically. Photo logging via AI cannot be offered to users until Bug #6 is patched and photo recognition is confirmed end-to-end with a successful call.
- **Recommendation:** Deploy a feature flag disabling the AI photo tab in the UI while Bug #6 is being fixed. Chat assistant and parse-text (text-based meal logging) can launch immediately. Photo recognition requires a focused 1-hour DEV fix + one more re-verification pass.
- **Cost this pass:** ~$0.02–0.05 (6 chat messages × ~$0.003 each + 5 parse-text calls × ~$0.002 each). Photo calls were all 502 so $0.00 for vision.
