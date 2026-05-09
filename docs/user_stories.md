# BiteIQ — User Stories

> **Phase 1 deliverable** owned by the PO Agent.
> Source: `_input/1_project_description.md`. `_input/3_po_input/` was empty (only README), so the `## Assumptions` section at the bottom records every inference made without supporting docs.
>
> Stories are user-facing only — endpoints, schemas, and status codes are the Architect's job in Phase 2. Stories are ordered by dependency (auth → goals → logging → dashboard → insights → assistant → body → account/quota/legal).

---

## US-1: Sign up with email or Google

**As a** new visitor, **I want to** create a BiteIQ account with email/password or Google, **so that** my food log is saved across devices and sessions.

### Acceptance Criteria
- [ ] AC1: A visitor on the landing page sees a primary "Sign up" call-to-action and a secondary "Log in" link.
- [ ] AC2: Sign up offers two methods: email + password, and "Continue with Google".
- [ ] AC3: Email sign-up requires a valid email format and a password meeting minimum strength (length/complexity rules surfaced inline as the user types).
- [ ] AC4: After successful sign-up, the user lands on the onboarding flow (US-2), not the dashboard.
- [ ] AC5: If the email is already registered, the user sees a clear message and a one-click switch to "Log in instead" (no silent failure, no duplicate account).
- [ ] AC6: Google OAuth that is cancelled/denied returns the user to the sign-up page with a non-blocking notice ("Sign-in cancelled").
- [ ] AC7: Email verification (if required by Supabase config) is communicated clearly: the user sees a "Check your inbox" screen with a resend option.
- [ ] AC8: Users can log out from any authenticated screen; logging out returns them to the landing page.

---

## US-2: Onboard with profile + calorie/macro goals

**As a** newly signed-up user, **I want to** enter my body profile and goal once, **so that** BiteIQ can compute my daily calorie + macro targets automatically.

### Acceptance Criteria
- [ ] AC1: Onboarding collects: sex, age (years), height (cm or ft/in), current weight (kg or lb), activity level (sedentary / light / moderate / active / very active), and goal (lose / maintain / gain weight) with an optional target weight.
- [ ] AC2: User can switch between metric and imperial units; the choice is remembered.
- [ ] AC3: On submit, BiteIQ computes a daily calorie target and a macro split (protein / carbs / fat in grams) using a documented formula (e.g. Mifflin-St Jeor + activity multiplier + goal delta) and shows the result before final confirmation.
- [ ] AC4: User can manually override the suggested calorie or macro values before confirming (advanced users with their own targets).
- [ ] AC5: User can skip optional fields (e.g. target weight); required fields are clearly marked and block submission if missing.
- [ ] AC6: After confirmation, the user is taken to the daily dashboard (US-7) with goals applied.
- [ ] AC7: Edge case — out-of-range inputs (e.g. age < 13, weight ≤ 0, height ≤ 0) are blocked with inline validation messages; users under 13 cannot proceed (under-13 disclaimer).
- [ ] AC8: Goals can be edited later from Settings (US-13); editing recomputes the suggestion but preserves any manual override unless the user re-accepts the suggestion.

---

## US-3: Log a meal by photo (AI vision)

**As a** logged-in user, **I want to** snap or upload a photo of my meal and have BiteIQ recognize the food and estimate calories/macros, **so that** I can log a meal in under 30 seconds without typing.

### Acceptance Criteria
- [ ] AC1: From the dashboard, a "Log meal" action opens a modal/screen with four input methods clearly labeled: Photo, Barcode, Voice, Manual.
- [ ] AC2: Photo input lets the user (a) take a photo with the device camera or (b) upload an image from their device.
- [ ] AC3: After upload, the user sees a loading state with a clear progress indicator while AI vision runs (target p95 < 5s, per project SLO).
- [ ] AC4: AI returns one or more recognized food items, each with an estimated portion size and an editable calorie + macro breakdown (protein/carbs/fat in grams).
- [ ] AC5: User can edit any recognized item — change name, change portion size (slider or numeric), add/remove items — before saving.
- [ ] AC6: User picks the meal slot (Breakfast / Lunch / Dinner / Snack) and a date/time (defaulting to "now"); on save, the meal appears in the day's log.
- [ ] AC7: Edge case — AI fails to recognize anything (low confidence or error): the user is told "We couldn't recognize this — add it manually" with a one-tap fallback to Manual entry (US-6) prefilled with the photo attached.
- [ ] AC8: Edge case — user is offline: photo and metadata are queued locally and uploaded automatically when connectivity returns; the meal appears with a "Pending sync" indicator until uploaded.
- [ ] AC9: Edge case — user uploads a non-food image (selfie, document, etc.): AI flags low confidence and prompts for confirmation or manual entry; nothing is auto-saved.
- [ ] AC10: Edge case — duplicate photo (same image hash uploaded again): system uses the cached recognition result (no re-charge to AI quota) but still creates a new meal log entry.
- [ ] AC11: Free-tier users see the remaining photo-recognition quota for the day before submitting (e.g. "3 of 5 left today"). See US-14 for quota enforcement.

---

## US-4: Log a meal by barcode

**As a** logged-in user, **I want to** scan a packaged-food barcode with my camera, **so that** I can log a packaged item without typing nutrition data.

### Acceptance Criteria
- [ ] AC1: From "Log meal", the Barcode tab opens a camera viewfinder that scans EAN-13 / UPC-A barcodes.
- [ ] AC2: A successful scan recognizes the barcode within 1s of focus (per project SLO) and shows the matched product (name, brand, default serving size, calories + macros) from Open Food Facts.
- [ ] AC3: The user can adjust serving size or quantity before saving; calories/macros recalculate live.
- [ ] AC4: The user picks a meal slot and date/time, then saves; the meal appears in the day's log.
- [ ] AC5: Edge case — barcode not found in Open Food Facts: the user sees "Product not in database — enter manually" with a one-tap fallback to Manual entry (US-6) prefilled with the scanned barcode for traceability.
- [ ] AC6: Edge case — camera permission denied: the user sees a clear permission-help message with a "Try again" / "Switch to manual" option; no app crash.
- [ ] AC7: Edge case — barcode read but missing nutrition fields in Open Food Facts: visible fields are filled, missing fields are flagged with "—" and are editable; user must complete required fields (calories at minimum) before save.
- [ ] AC8: Manual barcode entry (typing the digits) is offered as a fallback if camera scanning fails repeatedly.

---

## US-5: Log a meal by voice

**As a** logged-in user, **I want to** describe my meal in spoken English (e.g. "two slices of whole wheat toast with peanut butter"), **so that** I can log hands-free without using the camera.

### Acceptance Criteria
- [ ] AC1: From "Log meal", the Voice tab uses the browser's Web Speech API; a press-and-hold (or tap-to-start/tap-to-stop) microphone button captures audio.
- [ ] AC2: Live transcription appears on screen as the user speaks.
- [ ] AC3: On stop, the transcribed text is parsed (by AI) into one or more food items with portions and estimated calories/macros, presented in the same editable form as photo logs (US-3 AC4).
- [ ] AC4: Edge case — empty / unintelligible transcription: user sees "We didn't catch that — try again or type it" with retry and Manual-entry fallbacks; nothing is saved.
- [ ] AC5: Edge case — browser does not support Web Speech API (e.g. Firefox without flag): the Voice tab is disabled with a tooltip explaining the limitation; the user is steered to Manual entry.
- [ ] AC6: Edge case — microphone permission denied: clear permission-help message; no crash.
- [ ] AC7: User can edit the transcribed text before AI parsing if it captured the wrong words.
- [ ] AC8: User picks meal slot and date/time, then saves; the meal appears in the day's log.

---

## US-6: Log a meal manually (text)

**As a** logged-in user, **I want to** search a food name and enter portion + macros manually, **so that** I have full control when AI/barcode aren't available or accurate.

### Acceptance Criteria
- [ ] AC1: From "Log meal", the Manual tab offers a search box and free-form fields (name, portion size, calories, protein, carbs, fat).
- [ ] AC2: As the user types, suggestions appear from the user's previously logged foods first, then from Open Food Facts; tapping a suggestion prefills the macro fields.
- [ ] AC3: User can override any prefilled value before saving.
- [ ] AC4: User can save a manual entry as a "favorite" / quick-add for one-tap re-logging later.
- [ ] AC5: Required fields (name, calories) are validated before save; macros may be left blank but a warning is shown ("Macro split won't reflect this item").
- [ ] AC6: User picks meal slot and date/time, then saves; the meal appears in the day's log.
- [ ] AC7: Manual entry is always available as the fallback target from US-3 / US-4 / US-5 failure paths.

---

## US-7: View daily dashboard

**As a** logged-in user, **I want to** see today's calorie + macro totals against my goals at a glance, **so that** I know whether I'm on track without doing math.

### Acceptance Criteria
- [ ] AC1: Dashboard is the default authenticated landing page and shows: today's date, total calories consumed vs. goal (with remaining), and a P/C/F macro split (grams + % vs. goal) shown as a chart or progress bars.
- [ ] AC2: Below the totals, a chronological list of today's meals grouped by slot (Breakfast / Lunch / Dinner / Snack), each showing name, portion, and calorie subtotal.
- [ ] AC3: Tapping a logged meal opens its detail view, where the user can edit fields (name, portion, macros, slot, time) or delete the entry; totals update immediately.
- [ ] AC4: A primary "Log meal" button is always visible (sticky on mobile).
- [ ] AC5: Date navigator lets the user jump to previous days (read-only history) or back to today.
- [ ] AC6: Edge case — user has no meals yet today: dashboard shows an empty state with a clear "Log your first meal" CTA, not a confusing zero-bar chart.
- [ ] AC7: Edge case — user exceeds calorie goal: the over-goal portion is visually distinct (e.g. red over-bar) but no shaming language is used; macros over-goal also indicated.
- [ ] AC8: Disclaimer "Not medical advice" is visible (footer or banner) on the dashboard. See US-15.
- [ ] AC9: Dashboard loads under p95 < 2s (per project SLO).

---

## US-8: View weekly insights / trends

**As a** logged-in user, **I want to** see how my last 7 days compare against my goals, **so that** I can spot patterns and adjust habits.

### Acceptance Criteria
- [ ] AC1: A "Weekly insights" view shows: 7-day average daily calories, 7-day average macro split, days-on-target count, and a simple trend indicator (improving / steady / off-track).
- [ ] AC2: A bar/line chart shows daily totals for the last 7 days against the goal line.
- [ ] AC3: One or more plain-language insights are shown (e.g. "You averaged 12% under your protein goal this week. Try adding a protein source at breakfast."), generated automatically from the data.
- [ ] AC4: Insights are non-medical and non-prescriptive — see US-15 disclaimer.
- [ ] AC5: Edge case — fewer than 7 days of data: insights still render with a "Showing N of 7 days — log more for better insights" notice; no fake projections.
- [ ] AC6: Edge case — zero days logged: insights view shows an empty state, not a broken chart.
- [ ] AC7: User can navigate to previous weeks (read-only history).

---

## US-9: Chat with the AI nutrition assistant

**As a** logged-in user, **I want to** ask the AI assistant questions about my diet (e.g. "what's a high-protein snack under 200 cal?"), **so that** I get personalized guidance without leaving the app.

### Acceptance Criteria
- [ ] AC1: A chat surface is reachable from the main navigation; it shows a streaming-style conversation UI.
- [ ] AC2: The assistant has read-only awareness of the user's goals, today's logged meals, and recent week of data so answers can reference them (e.g. "you've had 80g protein today, 40g remaining").
- [ ] AC3: Responses stream token-by-token for perceived speed.
- [ ] AC4: Each response shows the medical-disclaimer banner ("AI guidance, not medical or clinical advice — consult a professional for medical concerns"). See US-15.
- [ ] AC5: The assistant refuses to make medical / clinical claims (diagnosis, treatment, prescription) and instead suggests consulting a professional; this behavior is testable.
- [ ] AC6: User can clear / start a new conversation; conversation history is saved per user.
- [ ] AC7: Edge case — assistant API fails / times out: user sees "Assistant unavailable, try again shortly" with a retry button; no silent hang.
- [ ] AC8: Edge case — user asks about a logged meal: assistant cites the meal entry (name + slot + day) in the answer to make context obvious.
- [ ] AC9: Free-tier users may have a separate quota for chat messages (decision deferred to Architect/business — surfaced as Assumption A4 below).

---

## US-10: Track body measurements over time

**As a** logged-in user, **I want to** record my weight (and optionally waist, hips, body fat %) over time, **so that** I can correlate dietary changes with body changes.

### Acceptance Criteria
- [ ] AC1: A "Body measurements" view lets the user add an entry with a date and one or more fields: weight, waist, hips, chest, body fat %.
- [ ] AC2: Only weight is required when adding an entry; other fields are optional.
- [ ] AC3: Past entries are shown as a chronological list and as a line chart over time per metric.
- [ ] AC4: User can edit or delete any past entry.
- [ ] AC5: Adding a new weight does NOT automatically update the user's profile weight or recompute calorie goals; instead the user is shown a banner "Your weight changed by Xkg — recalculate goals?" with a one-tap "Recalculate" button (links to US-2 goal edit).
- [ ] AC6: Edge case — out-of-range values (e.g. weight ≤ 0, body fat % > 60% or < 1%) are flagged with a confirmation prompt before save (typo guard).
- [ ] AC7: Unit (metric/imperial) follows the user's onboarding preference (US-2 AC2).

---

## US-11: Export and delete account (GDPR)

**As a** logged-in user, **I want to** export all my data and permanently delete my account, **so that** I can exercise my GDPR rights.

### Acceptance Criteria
- [ ] AC1: From Settings, a "Privacy & data" section offers two actions: "Export my data" and "Delete my account".
- [ ] AC2: "Export my data" produces a downloadable file (JSON or ZIP of JSONs) containing the user's profile, goals, all meals, all body measurements, and all chat history. The file is delivered within a documented time window (e.g. immediate or via email link if generation is async).
- [ ] AC3: "Delete my account" requires explicit confirmation (e.g. typing the word "DELETE" or re-entering password) before proceeding.
- [ ] AC4: On delete, ALL user data is removed: profile, goals, meals, body measurements, chat history, AND all uploaded photos in storage. No orphan data remains.
- [ ] AC5: The user is logged out and returned to the landing page after deletion, with a non-blocking confirmation ("Your account has been deleted").
- [ ] AC6: Edge case — deletion fails partway (e.g. storage error): the system retries and surfaces a clear error if it cannot complete; user data is never left in a partially-deleted state without notification.
- [ ] AC7: After deletion, the same email can be used to register a new (empty) account.
- [ ] AC8: Account deletion is irreversible; this is communicated clearly in the confirmation step.

---

## US-12: Free-tier vs. paid-tier gating

**As a** product owner, **I want** free-tier users limited to 5 AI photo recognitions per calendar day while paid-tier users have unlimited usage, **so that** AI inference costs stay predictable and the paid tier has clear value.

### Acceptance Criteria
- [ ] AC1: Free-tier users see their current photo-recognition usage on the Photo log screen (e.g. "3 of 5 left today").
- [ ] AC2: When a free-tier user attempts a 6th photo recognition in a single calendar day (user's local timezone), the request is blocked before AI is called and the user sees an upgrade prompt: "You've used your 5 free photo recognitions today. Upgrade for unlimited, or log this meal manually / by barcode / by voice."
- [ ] AC3: The upgrade prompt links to a paid-tier upgrade flow (the upgrade flow itself may be a stub for MVP — surfaced as Assumption A2 below).
- [ ] AC4: Paid-tier users see no quota counter and no upgrade prompts on photo log.
- [ ] AC5: Quota counter resets at user's local midnight.
- [ ] AC6: Quota only counts billable AI calls — cached recognitions (US-3 AC10) do NOT decrement the counter.
- [ ] AC7: Barcode, Voice, and Manual logging are NEVER quota-limited on the free tier (only Photo).
- [ ] AC8: Edge case — user hits quota mid-upload: photo and form state are preserved so the user can switch to Manual entry without retyping; no data loss.
- [ ] AC9: AI chat assistant quota (if any) is governed separately — see Assumption A4.

---

## US-13: Edit profile and goals

**As a** logged-in user, **I want to** edit my profile (height, age, activity level, target) and macro/calorie goals after onboarding, **so that** my targets stay accurate as my body or lifestyle changes.

### Acceptance Criteria
- [ ] AC1: From Settings, a "Profile & goals" section shows current values for everything collected in onboarding (US-2 AC1) plus the current calorie + macro goals.
- [ ] AC2: User can edit any field; the suggested calorie/macro recomputation appears immediately.
- [ ] AC3: User can choose to accept the new suggestion OR keep their existing manual override OR enter new manual values.
- [ ] AC4: Saving applies the new goals to the current and future days; past days' totals-vs-goal are preserved against the goal that was active on that day (no retroactive rewrite).
- [ ] AC5: Edge case — user changes unit (metric ↔ imperial): existing values are converted in-place, not zeroed.
- [ ] AC6: Edge case — invalid input (out-of-range): same validation as US-2 AC7.

---

## US-14: See remaining quota and upgrade path

**As a** free-tier user, **I want to** see my remaining AI photo quota and a clear upgrade path, **so that** I can plan my day or convert to paid when I hit the limit.

### Acceptance Criteria
- [ ] AC1: A persistent indicator on the Photo log screen shows quota used / remaining today (e.g. "2 of 5 used").
- [ ] AC2: A user-facing "Plan" page or Settings section shows the user's current tier (Free / Paid) and a comparison of free vs. paid limits.
- [ ] AC3: An "Upgrade" CTA is reachable from at least: the dashboard (subtle), the photo-log quota indicator, and the upgrade prompt at quota exhaustion (US-12 AC2).
- [ ] AC4: Edge case — user upgrades to paid mid-day: quota indicator disappears immediately and any blocked photo can be retried.
- [ ] AC5: Edge case — user downgrades to free: any usage already incurred today still counts; effectively starts at "X of 5 used".

---

## US-15: See "not medical advice" disclaimer in key surfaces

**As a** product owner, **I want** a clear "not medical advice" disclaimer surfaced wherever BiteIQ presents calculated targets, insights, or AI-generated guidance, **so that** users are not misled and we comply with the project's compliance constraint.

### Acceptance Criteria
- [ ] AC1: A short disclaimer ("BiteIQ provides general nutrition tracking — not medical, clinical, or dietetic advice. Consult a qualified professional for health concerns.") is visible on: the onboarding goal-result screen, the daily dashboard, the weekly insights screen, and every AI chat response.
- [ ] AC2: A full disclaimer (with link to Terms) is reachable from the footer of every authenticated page.
- [ ] AC3: AI assistant responses always include the short disclaimer (US-9 AC4) and the assistant refuses to make medical / clinical / diagnostic claims (US-9 AC5).
- [ ] AC4: Onboarding for users entering implausibly low/high inputs surfaces the disclaimer prominently (e.g. weight + goal combination implying very low calorie target) — no app shaming, just a "consult a professional" nudge.
- [ ] AC5: The disclaimer copy is in English (MVP); the i18n framework supports adding Vietnamese later without code refactor.

---

## Out of Scope (MVP)

These are explicitly **not** built in MVP and should not appear in any user story or design:

- **No podcast feature** — no audio content browser, no playback.
- **No recipe library** (16k+ recipes a la ParrotPal) — no recipe browse, no recipe import from social links.
- **No HealthKit / Google Fit integration** — no syncing weight/activity from device health stores. (Revisit in Phase 2 with mobile.)
- **No social feed, no friends, no sharing** — single-user app only.
- **No Apple Watch / wearable companion.**
- **No workout / exercise tracking** — BiteIQ tracks calories *in* only, never calories burned.
- **No native mobile app** — Phase 2 will add Flutter (iOS + Android, Play Store + App Store). MVP is web only.
- **No multilingual UI in MVP** — English only; the i18n framework is wired so Vietnamese can be added without refactor.
- **No automated paid-tier billing in MVP** — see Assumption A2.

---

## Assumptions

> `_input/3_po_input/` was empty at Phase 1 kickoff (only README present). The following inferences were made from `_input/1_project_description.md` alone. Architect / Designer / DEV should validate these in Phase 2; Team Lead should surface them in the interim report.

- **A1 — Onboarding formula.** The project description does not name a formula. Assumed Mifflin-St Jeor for BMR + standard activity multipliers (1.2 / 1.375 / 1.55 / 1.725 / 1.9) + a calorie delta for goal (e.g. ±500 kcal/day for lose/gain). Architect to lock the exact formula in Phase 2.
- **A2 — Paid-tier billing.** The project description mentions free vs. paid tiering but doesn't specify a billing provider or pricing for MVP. Assumed: the paid-tier upgrade flow is a stub for MVP (e.g. "Coming soon — join waitlist") with a feature flag for paid status, so quota gating logic is fully built and tested without integrating a payment gateway. Architect / Team Lead to confirm before Phase 3.
- **A3 — Macro split defaults.** No default protein/carb/fat ratio is specified. Assumed a balanced default of P/C/F = 30 / 40 / 30 by calories, editable per user. Architect to confirm.
- **A4 — AI chat quota.** The description caps photo AI at 5/day on free tier but is silent on chat assistant quota. Assumed: free tier gets a daily chat-message cap (e.g. 10/day) to keep AI cost predictable; paid is unlimited. Exact number deferred to Architect; user-facing behavior mirrors US-12 (visible counter + upgrade prompt at exhaustion). Surface this for user confirmation.
- **A5 — Meal slots.** Assumed the four standard slots Breakfast / Lunch / Dinner / Snack with no user customization in MVP. (Custom slot names can be a Phase 2 enhancement.)
- **A6 — Photo retention.** Assumed photos are kept indefinitely until the user deletes the meal or their account (US-11 AC4). No automatic photo-purge schedule in MVP.
- **A7 — Date/timezone semantics.** Assumed all "daily" totals (dashboard, quota reset) use the user's local timezone, captured at sign-up and editable in settings. Architect to confirm storage semantics.
- **A8 — Voice language.** Web Speech API supports many languages; MVP assumed English (en-US / en-GB) only, matching the English-only UI constraint. Multilingual voice deferred.
- **A9 — Data export format.** Assumed JSON (single file or ZIP-of-JSONs) for the GDPR export — easiest for users to inspect and for engineering to ship. CSV export deferred.
- **A10 — Body fat measurement.** Body fat % is a self-reported number (no AI estimation, no caliper integration). Optional field only.
- **A11 — Account deletion latency.** Assumed deletion is synchronous and complete within the same request (small data volumes). If volumes grow, async deletion with email confirmation can replace the sync flow without breaking US-11 ACs.

---

## Phase 12 — Multi-provider vision + vi-default

> **Change request appended 2026-05-08.** Added per Team Lead spawn for Phase 12 (Multi-provider AI vision fallback + Vietnamese-default i18n with switcher). All clarifications below are non-negotiable inputs captured by Team Lead in `.agent_team/task_board.md` Phase 12 §"User clarifications captured 2026-05-08". Stories are user-facing only — provider model IDs, env vars, schema columns, and HTTP status codes are intentionally omitted (Architect's domain).

---

## US-MP1: Photo recognition keeps working when one AI provider runs out of free capacity

**As a** BiteIQ user logging a meal by photo, **I want** the app to silently try a backup AI vision provider when the primary one is out of free capacity, **so that** I can keep logging meals reliably without seeing failures caused by provider-side quota limits.

### Acceptance Criteria

**Happy path**

- [ ] AC1: When I submit a meal photo, BiteIQ tries the configured primary vision provider (Gemini) first. If it returns a successful recognition, I see the recognized items + estimated calories/macros exactly as before — no UX change versus today.
- [ ] AC2: When the primary provider reports it is **out of free capacity** (quota-exhausted / rate-limit-from-upstream), BiteIQ automatically tries the next provider in the chain (Ollama Cloud), then the third (Claude). I do not see any error during this fallback — only the final recognized result.
- [ ] AC3: Each successful recognition result is tagged with which provider produced it (a `provider` field is part of the response payload). This is for observability and developer/QA introspection — it does not have to be displayed in the user-facing UI, but it MUST be present so QA, support, and analytics can answer "which provider served this call?".

**Edge cases**

- [ ] AC4: **All three providers report quota-exhausted in the same request.** I see a clear, user-friendly error: "All AI providers are temporarily out of free capacity. Please try again in a few minutes." This must NOT appear as a generic 500 / "Something went wrong". The error message is translated (see US-I18N1).
- [ ] AC5: **A provider returns a non-quota error** (network failure, generic 5xx, malformed response, auth failure, etc.). BiteIQ does **not** silently fall back — it surfaces the failure as a real error so the bug stays visible to engineering. Falling back is reserved exclusively for the explicit quota-exhausted signal (HTTP 429 or upstream "rate limit" / "quota" wording). User-facing copy for this case: a clear "Photo recognition failed — please try again or log this meal manually" with the manual-log path one tap away.
- [ ] AC6: **First provider succeeds.** No fallback occurs and no extra provider calls are made (cost guardrail). The `provider` field reflects the actually-used provider, not the configured first-choice.
- [ ] AC7: **Provider chain order is deterministic** for a given request — Gemini → Ollama Cloud → Claude. The user never sees a non-deterministic ordering and Architect/Ops can override the chain via configuration without a code change (user-facing behavior is unaffected by that override beyond the order in which fallbacks happen).
- [ ] AC8: **Cached recognitions are not re-charged.** If the same image (by content hash) was already recognized, the cached result is returned and no provider is called. The cached result still carries the original `provider` value from when it was first computed.

---

## US-MP2: Free-tier users can submit unlimited meal photos as long as a provider has free capacity

**As a** free-tier BiteIQ user, **I want** the previous "5 photos per day" cap removed, **so that** I can log every meal by photo without hitting an artificial daily limit, while BiteIQ still protects itself from runaway abuse.

### Acceptance Criteria

**Happy path**

- [ ] AC1: As a free-tier user, I can submit a 6th, 7th, 20th, etc. photo on the same calendar day and each one is recognized normally (subject to US-MP1's provider chain). I never see a "daily photo limit reached" / "upgrade to continue" prompt for the photo flow.
- [ ] AC2: The dashboard / settings UI no longer surfaces a "X / 5 photos used today" counter or any photo-quota progress indicator. (Any leftover photo-quota counter widget from the previous design is removed or hidden.)
- [ ] AC3: The "Upgrade for unlimited photos" prompt that previously fired at the 5-photo cap is removed from the photo logging flow. (The general upgrade CTA elsewhere in the app — e.g. settings — may stay; only the photo-cap-triggered prompt is gone.)

**Edge cases**

- [ ] AC4: **Burst abuse protection still applies.** If I send photos faster than the per-second per-user rate limit (30/sec/user, already in place from earlier phases), I see a friendly "You're submitting too fast — slow down a moment" message. This guardrail is **not** removed; only the per-day count is.
- [ ] AC5: **Chat AI quota is unchanged.** The free-tier cap on AI chat messages (US-9 / A4) stays exactly as it is. Removing the photo cap does NOT remove or alter the chat cap. Settings copy describing chat quota stays accurate.
- [ ] AC6: **All three providers exhausted simultaneously.** I see the US-MP1 AC4 error ("all providers temporarily out of free capacity"), not a "you've used your daily photos" message. The two failure modes are clearly distinct in copy so the user knows whether to retry later vs. upgrade.
- [ ] AC7: **Paid-tier users** see no behavior change from this story (they were already unlimited).

---

## US-I18N1: Vietnamese is the default UI language

**As a** new visitor to BiteIQ (logged-out or just-signed-up), **I want** the app to be in Vietnamese by default, **so that** the primary target audience can use BiteIQ without first hunting for a language setting.

### Acceptance Criteria

**Happy path**

- [ ] AC1: A logged-out visitor landing on any public page (marketing landing, login, signup) sees Vietnamese (vi-VN) copy. This is independent of the browser's `Accept-Language` header — vi is the default for new sessions regardless of browser language.
- [ ] AC2: A user who has just signed up and not yet chosen a language sees Vietnamese on every authenticated screen they encounter during onboarding and beyond — at minimum: signup confirmation, onboarding (profile + goals), dashboard, log-meal modal (all four input methods: photo / barcode / voice / text), meal history, weekly insights, AI chat, body measurements, settings, and the "not medical advice" disclaimer surfaces.
- [ ] AC3: All dynamic copy is translated, not just static labels — this includes form validation errors, toast notifications, empty states, loading states, the upgrade CTA wording, the "all providers exhausted" error from US-MP1 AC4, and AI chat assistant disclaimers.
- [ ] AC4: Date and number formatting follows the vi-VN locale: dates show in a Vietnamese-natural format (e.g. `8 thg 5, 2026` rather than ISO `2026-05-08` or US `May 8, 2026`); thousands separator uses `.` and decimal separator uses `,` per Vietnamese convention; weekday names appear in Vietnamese on the weekly insights view.
- [ ] AC5: The HTML root element correctly declares `lang="vi"` so screen readers pronounce Vietnamese content correctly. (Switches to `lang="en"` only when the user explicitly opts into English — see US-I18N2.)

**Edge cases**

- [ ] AC6: **Tight UI elements with longer Vietnamese text.** Buttons, tab labels, navigation items, and dashboard CTAs that fit in EN but overflow in VI must wrap or shrink gracefully on small mobile viewports (≤ 360 px wide) without breaking layout, clipping text, or overlapping other elements. No horizontal scroll on any of the 17 screens. Designer flags at-risk components in their spec.
- [ ] AC7: **Missing translation key.** If the runtime tries to render a copy key that is missing in the active language catalog, the user sees the **vi** translation as the fallback (vi is the source-of-truth catalog for Phase 12). If the key is missing in vi too, the raw key string is shown and the gap is logged for engineering — the user never sees a blank label.
- [ ] AC8: **Server-rendered pages** (dashboard, history, insights) render in vi from the first paint — no flash of English before hydration.
- [ ] AC9: **AI chat responses.** When a user types in Vietnamese, the AI assistant replies in Vietnamese. The "not medical advice" disclaimer is shown in Vietnamese alongside each response. (Chat reply language follows UI language; assistant behavior re: medical claims from US-9 / US-15 is unchanged.)

---

## US-I18N2: Language switcher in Settings (vi ↔ en)

**As a** BiteIQ user who prefers English, **I want** a language toggle in Settings that switches the entire UI to English and remembers my choice, **so that** I can use BiteIQ in my preferred language across sessions and devices without losing my preference.

### Acceptance Criteria

**Happy path**

- [ ] AC1: The Settings screen contains a clearly-labelled language control offering exactly two choices: **Tiếng Việt (vi)** and **English (en)**. The currently-active language is visually indicated.
- [ ] AC2: Toggling from vi to en (or en to vi) updates **every visible UI string on the current screen immediately** — no hard page reload required, no logout-and-back-in. A single soft re-render is acceptable; a full-page reload is not.
- [ ] AC3: The new selection is persisted to my user profile so that on my next visit (any device, any session — desktop and mobile browser, including after sign-out and sign-in again), BiteIQ loads in my chosen language. The persistence survives browser cache clears.
- [ ] AC4: After switching, navigating to other screens (dashboard → history → insights → chat → body measurements → settings, etc.) all show the chosen language consistently. The HTML `lang` attribute (US-I18N1 AC5) updates to match the new selection.

**Edge cases**

- [ ] AC5: **Logged-out user.** A user who is signed out always sees vi on the public pages (US-I18N1 AC1) regardless of any previously-saved preference for an account on the same device. The switcher is not shown on logged-out pages (settings is auth-only).
- [ ] AC6: **First-time user.** New accounts default to vi (matching US-I18N1) until the user explicitly toggles to en. The switcher is not part of the onboarding flow — it lives only in Settings.
- [ ] AC7: **Switch during an in-flight action.** If the user switches language while a meal is being logged or a chat reply is streaming, the in-flight action completes without errors. Any text rendered after the switch (e.g. the next chat message, the next toast) appears in the new language; already-rendered text on persistent surfaces (dashboard, etc.) updates on the next soft re-render. No data loss.
- [ ] AC8: **Persistence write fails.** If saving the preference fails (network drop, server error), the UI still flips locally for the rest of the session AND surfaces a small non-blocking notice ("Could not save language preference — we will retry") so the user is not silently left with a non-persisted choice. On the user's next session the preference reverts to the last successfully-saved value (or to vi if none was saved).
- [ ] AC9: **EN catalog has a missing key.** Per US-I18N1 AC7, fall back to the vi translation (the source-of-truth catalog) for that specific key rather than showing a broken label. Document this fallback policy in the design / copy spec so Designer + DEV are aligned.
- [ ] AC10: **Disclaimer + medical-advice wording.** The "not medical advice" disclaimer (US-15) is fully translated and shown in the active language on every surface that previously showed the EN version. Switching language does not remove or hide the disclaimer.
