# BiteIQ — Design Specification

> **Phase 2 deliverable** owned by the Designer Agent.
> Source: user stories alone (`_input/4_design_input/` empty; `figma_file_url` = N/A).
> Stack: Next.js 15 + Tailwind CSS + shadcn/ui + Supabase.
> Accessibility target: WCAG AA.
> Web-only MVP — "Mobile considerations" section omitted per project scope.

---

## Assumptions

All of the following were inferred without source design files, Figma frames, or a design brief. The Designer Agent made these choices based on `docs/user_stories.md`, `_input/1_project_description.md`, and the ParrotPal App Store listing as a visual reference only (no screenshots obtained).

- **DA-1 Visual tone.** ParrotPal (reference app) uses a soft-pastel, mascot-driven, large-white-space aesthetic with big number-circle progress rings. BiteIQ targets a smarter/minimal positioning — no parrot mascot. Visual language: clean, confident, data-forward, friendly without being childish. Icon language: fork + leaf motif in the logo/favicon.
- **DA-2 Color palette.** Primary green (health/nutrition) + accent orange (warmth/food energy). Both calibrated to pass WCAG AA 4.5:1 on white and on the dark-mode surface. Full palette specified in Design Tokens.
- **DA-3 Typography.** Inter (ships with shadcn/ui, Google Fonts CDN, zero-cost). Geist as the system-font fallback chain. No display-only font needed at MVP scale.
- **DA-4 Component library.** shadcn/ui is the baseline. Custom variants layered on top — no design system rebuild needed. Component names in this spec reference shadcn primitives exactly.
- **DA-5 Dark mode.** Proposed as the primary mode (tracking apps are often used in dim environments; dark backgrounds make progress rings and chart lines pop). Light mode tokens also specified. User can toggle in Settings.
- **DA-6 Layout paradigm.** Mobile-first (375px baseline). Single-column on mobile. Two-column sidebar + main on desktop (1024px+). No three-column layouts in MVP.
- **DA-7 Navigation.** Bottom tab bar on mobile (375–767px); left sidebar nav on desktop (1024px+); tab row on tablet (768–1023px). Tabs: Dashboard · Log · Insights · Chat · Body · Settings.
- **DA-8 Calorie ring.** Large SVG ring (inspired by Apple Watch Activity / ParrotPal ring) centered on the dashboard. Diameter 200px mobile, 240px desktop. Shows consumed / goal numerics inside.
- **DA-9 Log Meal entry point.** Floating Action Button (FAB) on mobile (bottom-right, 56px circle). Standard `<Button size="lg">` on desktop top of meal list.
- **DA-10 Log Meal surface.** `<Sheet>` (bottom drawer) on mobile; `<Dialog>` on desktop. Contains four `<Tabs>`: Photo · Barcode · Voice · Manual.
- **DA-11 Charts.** Recharts (compatible with Next.js RSC; already in shadcn/ui chart component). Line chart for body measurements; bar chart for weekly insights. Respect `prefers-reduced-motion` by disabling entry animations.
- **DA-12 Upgrade / billing.** Stub UI only. No payment gateway in MVP. "Upgrade" CTA opens a `<Dialog>` with a "Join waitlist" email-capture form.

---

## Design Tokens

### Colors

All hex values given for both light and dark mode. Dark mode is the **recommended default**; light mode is opt-in.

#### Primary palette

| Token | Light mode | Dark mode | Usage |
|-------|-----------|-----------|-------|
| `--color-primary` | `#16A34A` (green-600) | `#22C55E` (green-500) | Primary CTA, active states, calorie ring fill |
| `--color-primary-foreground` | `#FFFFFF` | `#052E16` | Text on primary bg |
| `--color-primary-subtle` | `#DCFCE7` (green-100) | `#14532D` (green-900) | Chip backgrounds, tag fills |
| `--color-accent` | `#EA580C` (orange-600) | `#FB923C` (orange-400) | Accent CTAs, macro highlights (carbs), streak indicators |
| `--color-accent-foreground` | `#FFFFFF` | `#431407` | Text on accent bg |

#### Neutral / surface palette

| Token | Light mode | Dark mode |
|-------|-----------|-----------|
| `--color-background` | `#FFFFFF` | `#09090B` (zinc-950) |
| `--color-surface` | `#F4F4F5` (zinc-100) | `#18181B` (zinc-900) |
| `--color-surface-elevated` | `#FFFFFF` | `#27272A` (zinc-800) |
| `--color-border` | `#E4E4E7` (zinc-200) | `#3F3F46` (zinc-700) |
| `--color-muted` | `#71717A` (zinc-500) | `#A1A1AA` (zinc-400) |
| `--color-foreground` | `#09090B` (zinc-950) | `#FAFAFA` (zinc-50) |
| `--color-foreground-secondary` | `#52525B` (zinc-600) | `#D4D4D8` (zinc-300) |

#### Semantic palette

| Token | Light | Dark |
|-------|-------|------|
| `--color-success` | `#16A34A` | `#22C55E` |
| `--color-success-bg` | `#F0FDF4` | `#052E16` |
| `--color-warning` | `#D97706` (amber-600) | `#FBBF24` (amber-400) |
| `--color-warning-bg` | `#FFFBEB` | `#451A03` |
| `--color-error` | `#DC2626` (red-600) | `#F87171` (red-400) |
| `--color-error-bg` | `#FEF2F2` | `#450A0A` |
| `--color-info` | `#2563EB` (blue-600) | `#60A5FA` (blue-400) |
| `--color-info-bg` | `#EFF6FF` | `#172554` |

#### Macro-specific colors (used in rings, charts, legends)

| Macro | Token | Hex (dark mode) |
|-------|-------|-----------------|
| Protein | `--color-macro-protein` | `#60A5FA` (blue-400) |
| Carbs | `--color-macro-carbs` | `#FB923C` (orange-400) |
| Fat | `--color-macro-fat` | `#FACC15` (yellow-400) |

Contrast on `zinc-950` background: protein 6.3:1 ✓, carbs 4.6:1 ✓, fat 11.2:1 ✓.

---

### Typography

**Family:** `Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif`

| Scale token | Size | Weight | Line-height | Usage |
|-------------|------|--------|-------------|-------|
| `text-display` | 48px / 3rem | 700 | 1.1 | Hero headline (Landing only) |
| `text-h1` | 32px / 2rem | 700 | 1.2 | Page titles (Dashboard date, section heads) |
| `text-h2` | 24px / 1.5rem | 600 | 1.3 | Card section titles, modal titles |
| `text-h3` | 20px / 1.25rem | 600 | 1.35 | Sub-section heads, group labels |
| `text-body-lg` | 18px / 1.125rem | 400 | 1.6 | Onboarding copy, AI chat messages |
| `text-body` | 16px / 1rem | 400 | 1.6 | Default body text, form labels |
| `text-body-sm` | 14px / 0.875rem | 400 | 1.5 | Helper text, timestamps, secondary meta |
| `text-caption` | 12px / 0.75rem | 400 | 1.4 | Quota counter, chart axis labels, footnotes |
| `text-label` | 14px / 0.875rem | 500 | 1.2 | Form labels, tab labels, button text (sm) |
| `text-num-lg` | 48px / 3rem | 700 | 1 | Calorie ring center number |
| `text-num-md` | 28px / 1.75rem | 700 | 1 | Macro ring totals |

---

### Spacing scale

`4 / 8 / 12 / 16 / 20 / 24 / 32 / 48 / 64` (px) — maps to Tailwind `p-1 / p-2 / p-3 / p-4 / p-5 / p-6 / p-8 / p-12 / p-16`.

Page horizontal padding: 16px mobile · 24px tablet · 32px desktop.
Max content width: `1280px` (Tailwind `max-w-screen-xl`).

---

### Radius

Follows shadcn/ui defaults (configured in `tailwind.config.ts` via CSS variables):

| Token | Value | Usage |
|-------|-------|-------|
| `radius-sm` | 4px | Badges, tags, small chips |
| `radius-md` | 6px | Inputs, small buttons |
| `radius-lg` | 8px | Cards, modals default |
| `radius-xl` | 12px | Sheets, large cards |
| `radius-2xl` | 16px | Bottom drawer (Sheet), FAB |
| `radius-full` | 9999px | FAB button, avatar, pill buttons |

---

### Shadows (elevation)

| Token | Value | Usage |
|-------|-------|-------|
| `shadow-sm` | `0 1px 2px rgb(0 0 0 / 0.08)` | Hover state on list items |
| `shadow-md` | `0 4px 8px rgb(0 0 0 / 0.12)` | Cards, dropdowns |
| `shadow-lg` | `0 8px 24px rgb(0 0 0 / 0.16)` | Modals, dialogs |
| `shadow-xl` | `0 16px 48px rgb(0 0 0 / 0.24)` | Sheet/drawer overlay |
| `shadow-ring` | `0 0 0 3px var(--color-primary) / 0.4` | Focus-visible ring |

---

### Iconography

**Library:** Lucide React (ships with shadcn/ui). Default size 20px (`size-5`), stroke-width 1.5.

Key icon assignments:
- Dashboard: `LayoutDashboard`
- Log Meal: `PlusCircle` (nav) / `Plus` (FAB)
- Photo: `Camera`
- Barcode: `ScanBarcode`
- Voice: `Mic`
- Manual: `PencilLine`
- Insights: `TrendingUp`
- Chat: `MessageCircle`
- Body: `Scale`
- Settings: `Settings`
- Delete: `Trash2`
- Edit: `Pencil`
- Check: `CheckCircle2`
- Warning: `AlertTriangle`
- Info: `Info`
- Upgrade: `Sparkles`

---

## Breakpoints

| Name | Min-width | Layout |
|------|-----------|--------|
| `xs` (base) | 375px | Single column, bottom nav, Sheet-based modals |
| `sm` | 640px | Single column, wider gutters |
| `md` (tablet) | 768px | Tab-row nav (top), Dialog modals |
| `lg` (desktop) | 1024px | Sidebar nav (left 240px) + main content area |
| `xl` | 1280px | Same as lg, max-width capped |

---

## Global Layout Shell

### Mobile (< 768px)

```
┌──────────────────────────────────────┐
│ App Header (56px):                   │
│  [BiteIQ logo]        [Avatar icon]  │
├──────────────────────────────────────┤
│                                      │
│  Page content (flex-1, scrollable)   │
│                                      │
│  [FAB: + Log] (bottom-right, fixed)  │
├──────────────────────────────────────┤
│ Bottom Nav (56px, safe-area-bottom)  │
│ [Dashboard][Log][Insights][Chat][Me] │
└──────────────────────────────────────┘
```

### Desktop (≥ 1024px)

```
┌────────────────────────────────────────────────────┐
│  Sidebar (240px, fixed left)                       │
│  BiteIQ logo                                       │
│  ─────────────                                     │
│  Dashboard                                         │
│  Insights                                          │
│  AI Chat                                           │
│  Body                                              │
│  Settings                                          │
│  ─────────────                                     │
│  [Upgrade CTA (free tier)]                         │
│  Avatar + email (bottom)                           │
├────────────────────────────────────────────────────┤
│  Main content (flex-1)                             │
│  ┌──────────────────────────────────────────────┐  │
│  │ Page header: title + [Log Meal button]       │  │
│  │ Page body                                    │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
```

---

## Screen: Landing / Marketing

**Maps to:** US-1 (pre-auth entry point)
**Source:** Inferred from user stories

### Layout

- Full-viewport hero section above the fold (100vh).
- Hero: centered content, max-width 640px, horizontal padding 24px.
- Features grid: 3-column on desktop, 2-column on tablet, 1-column on mobile. Cards with 24px gap.
- Footer: single row with links (Privacy · Terms · Disclaimer) centered.

### Components

- **Hero badge:** `<Badge variant="secondary">` — text: "Beta · Free to start"
- **Hero headline:** `<h1 className="text-display">` — text: "Track meals in seconds,\nnot minutes."
- **Hero sub-copy:** `<p className="text-body-lg text-muted">` — text: "AI identifies your food from a photo, barcode, or voice. See your calories and macros instantly."
- **Primary CTA:** `<Button size="lg" className="w-full sm:w-auto">` — text: "Get started free"
- **Secondary link:** `<Button variant="ghost">` — text: "Already have an account? Log in"
- **Feature cards:** `<Card>` with Lucide icon (64px, colored), `<h3>`, `<p>`. Cards: "Snap a photo" / "Scan a barcode" / "Talk to log" / "Smart dashboard" / "Weekly insights" / "AI nutrition coach"
- **Disclaimer strip:** `<p className="text-caption text-muted">` at bottom of hero — "Nutrition tracking only — not medical advice."

### States

- Default: loads immediately, no auth check needed.
- Loading: static page, no skeletons needed.

### Content — Exact Copy

| Element | Copy |
|---------|------|
| Hero badge | `Beta · Free to start` |
| H1 | `Track meals in seconds,\nnot minutes.` |
| Sub-copy | `AI identifies your food from a photo, barcode, or voice. See your calories and macros instantly — no manual entry required.` |
| CTA | `Get started free` |
| Sign-in link | `Already have an account? Log in →` |
| Disclaimer | `BiteIQ provides general nutrition tracking — not medical, clinical, or dietetic advice.` |

### Interactions

- "Get started free" → navigates to `/signup`.
- "Log in" → navigates to `/login`.
- Feature cards: hover lifts with `shadow-md`, 150ms ease transition.

### Accessibility

- H1 is the first heading on the page; landmark `<main>` wraps content.
- CTA button has explicit `aria-label="Get started free — create your BiteIQ account"`.
- Focus order: logo → CTA → Log in → feature cards → footer links.
- Color contrast: hero text (zinc-50) on `zinc-950` bg = 19.1:1 ✓.

---

## Screen: Sign Up / Login

**Maps to:** US-1
**Source:** Inferred from user stories

### Layout

- Centered single-column card, max-width 400px, vertically centered on viewport.
- `<Card className="p-8">` with BiteIQ logo above (48px, centered).
- Toggle between Sign Up / Log In via tabs or a text link below the form.

### Components

- **Tabs:** `<Tabs defaultValue="signup">` with "Sign up" / "Log in" triggers (or separate pages `/signup` `/login`).
- **Email field:** `<Input type="email" placeholder="you@example.com" />` with `<Label>`.
- **Password field:** `<Input type="password" />` with show/hide toggle (`<Button variant="ghost" size="icon"><Eye /></Button>`).
- **Password strength indicator** (sign-up only): 4-segment bar below password field. Colors: red → orange → yellow → green per strength level.
- **Submit:** `<Button className="w-full" size="lg">`.
- **Google OAuth:** `<Button variant="outline" className="w-full">` with Google SVG logo (16px). Text: "Continue with Google".
- **Divider:** `<Separator>` with "or" label, centered.
- **Error banner:** `<Alert variant="destructive">` — shown when email already registered or credentials wrong.
- **Check inbox screen:** Replaces form after email sign-up; shows `<MailCheck size={48} />`, heading "Check your inbox", sub-copy "We sent a confirmation link to {email}", `<Button variant="outline">Resend email</Button>`.
- **"Sign-in cancelled" notice:** `<Alert variant="default">` — appears after Google OAuth cancel, dismissible.

### States

| State | Visual |
|-------|--------|
| Default | Form visible, submit enabled when fields non-empty |
| Loading | Submit button shows `<Loader2 className="animate-spin" />` + disabled, 300ms minimum display |
| Error — wrong creds | `<Alert variant="destructive">` beneath form: "Incorrect email or password." |
| Error — email exists | `<Alert>` with "This email is registered. [Log in instead →]" |
| Success — email sent | "Check your inbox" screen |
| Success — OAuth | Redirect to `/onboarding` or `/dashboard` |

### Content

| Element | Copy |
|---------|------|
| Sign-up title | `Create your account` |
| Log-in title | `Welcome back` |
| Email label | `Email` |
| Password label | `Password` |
| Sign-up CTA | `Create account` |
| Log-in CTA | `Log in` |
| Google button | `Continue with Google` |
| Forgot password | `Forgot password?` |

### Interactions

- Submit on Enter in last field.
- Password strength updates live as user types.
- Google OAuth opens popup; on cancel shows non-blocking `<Alert>`.
- "Log in instead →" in duplicate-email error is a link, not a button.

### Accessibility

- `autocomplete="email"` and `autocomplete="new-password"` / `autocomplete="current-password"` on inputs.
- Password toggle button `aria-label="Show password"` / `"Hide password"`.
- Error alerts have `role="alert"` so screen readers announce immediately.
- Form `aria-label="Sign up form"` / `"Log in form"`.

---

## Screen: Onboarding Wizard

**Maps to:** US-2
**Source:** Inferred from user stories

### Layout

- Full-page wizard, max-width 560px, centered.
- Step indicator at top: 4 dots (active = filled primary, past = primary outline, future = muted). With step label text below: "Profile · Activity · Goal · Your targets".
- Single `<Card className="p-8">` per step.
- Sticky bottom bar: `<Button>` Back (ghost) + `<Button>` Next/Confirm (primary).

### Steps

**Step 1 — Profile**
Fields (all inside a `<form>`):
- Sex: `<RadioGroup>` with two `<RadioGroupItem>` cards: "Male" / "Female". Large tap targets, 48px min height.
- Age: `<Input type="number" min="13" max="120" />` — inline validation: "Must be at least 13".
- Height: dual input (cm OR ft + in), unit toggle `<ToggleGroup>` ("cm" / "ft·in"). Show only the active format.
- Weight: `<Input type="number" />` + unit toggle ("kg" / "lb").
- Required fields marked with `*`; validation on Next click, not on blur.

**Step 2 — Activity level**
- `<RadioGroup>` vertical, 5 options as large `<Card>` rows (clickable):
  - Sedentary — "Little or no exercise"
  - Lightly active — "1–3 days/week"
  - Moderately active — "3–5 days/week"
  - Active — "6–7 days/week"
  - Very active — "Physical job or 2× daily training"

**Step 3 — Goal**
- `<RadioGroup>` 3 option cards: "Lose weight" / "Maintain weight" / "Gain weight". Each shows calorie delta hint: "−500 kcal/day" / "±0" / "+500 kcal/day".
- Optional target weight field appears on Lose/Gain selection: `<Input type="number" placeholder="e.g. 65 kg" />`.

**Step 4 — Your targets (result screen)**
- Computed values displayed in a 3-column stat grid:
  - Calories: `{N} kcal/day` (large, `text-num-md`)
  - Protein: `{N}g`
  - Carbs: `{N}g`
  - Fat: `{N}g`
- Formula note: `<p className="text-caption text-muted">Calculated using Mifflin-St Jeor with your activity level and goal.</p>`
- Override section: `<Collapsible>` — "Customize targets" toggle reveals editable `<Input>` fields for each value.
- Medical disclaimer: `<Alert variant="default" className="mt-4">` — "BiteIQ provides general nutrition tracking — not medical advice. Consult a qualified professional for health concerns."
- CTA: `<Button size="lg" className="w-full">Start tracking</Button>`.

### States

| State | Visual |
|-------|--------|
| Step validation error | Inline `<p className="text-error text-body-sm">` below the field; border turns `border-error` |
| Under-13 age | Error: "BiteIQ is not available for users under 13." — blocks Next |
| Out-of-range values | Error message per field; confirms before allowing extreme values |
| Loading (computing targets) | Spinner on Next CTA of Step 3 for ~300ms |
| Override expanded | Collapsible open; edited values shown in accent color |

### Content

| Element | Copy |
|---------|------|
| Step 1 title | `Tell us about yourself` |
| Step 2 title | `How active are you?` |
| Step 3 title | `What's your goal?` |
| Step 4 title | `Your daily targets` |
| Disclaimer | `BiteIQ provides general nutrition tracking — not medical, clinical, or dietetic advice. Consult a qualified professional for health concerns.` |
| Confirm CTA | `Start tracking` |

### Interactions

- Back button goes to previous step; forward data is preserved.
- Unit toggle (cm ↔ ft/in, kg ↔ lb) converts current value live.
- "Start tracking" → redirects to `/dashboard`.
- On target screen, editing a value turns the computed badge to "custom" label.

### Accessibility

- Wizard `aria-label="Onboarding wizard, step {N} of 4"`.
- Each radio card has `role="radio"` with visible label.
- Step indicator dots: `aria-label="Step {N}: {name}"`, current step `aria-current="step"`.
- Focus moves to the heading of the new step when advancing.

---

## Screen: Daily Dashboard

**Maps to:** US-7, US-15
**Source:** Inferred from user stories

### Layout

**Mobile (375px):**
```
┌──────────────────────────────┐
│ Header: "Tuesday, May 4"     │
│         [< Today >] nav      │
├──────────────────────────────┤
│    Calorie ring (200px)      │
│  [1,240] kcal consumed       │
│  of [2,000] goal             │
│  [360 remaining]             │
├──────────────────────────────┤
│ Macro row (3 mini-rings):    │
│  Protein  Carbs  Fat         │
│  82g/150g 140g/250g 44g/65g  │
├──────────────────────────────┤
│ Quick widgets row:           │
│  [Water: 1.2L] [Weight: 74kg]│
├──────────────────────────────┤
│ Meal list:                   │
│  Breakfast (2 items, 480cal) │
│  Lunch (1 item, 640cal)      │
│  Dinner (empty state)        │
│  Snack (empty state)         │
├──────────────────────────────┤
│ Disclaimer footer            │
└──────────────────────────────┘
[FAB: + Log Meal]
```

**Desktop (1024px+):** Two-column. Left col (60%): calorie ring + macro row + meal list. Right col (40%): macro detail cards + quick widgets + weekly snapshot mini-chart.

### Components

- **Calorie ring:** Custom SVG `<CircleProgress>` component. Track color: `--color-border`. Fill: `--color-primary`. Over-goal: fill switches to `--color-error` for the excess arc. Center: `<span className="text-num-lg">1,240</span>` + `<span className="text-body-sm text-muted">of 2,000 kcal</span>`.
- **Macro mini-rings:** Three `<CircleProgress size="80px">` side by side. Each labeled with macro name, grams consumed / goal. Colors: protein blue, carbs orange, fat yellow.
- **Date navigator:** `<Button variant="ghost" size="icon"><ChevronLeft /></Button>` / date text / `<Button variant="ghost" size="icon"><ChevronRight /></Button>`. Future dates disabled.
- **Quick widgets:** `<Card className="p-3">` row. Water widget: droplet icon + "1.2L" + "+ Add" link. Weight widget: scale icon + "74 kg logged today" or "+ Log weight".
- **Meal group headers:** `<h3 className="text-h3">` with meal slot name + total calories badge.
- **Meal log item:** `<Card className="flex p-3">` — food name (text-body), portion + macros (text-body-sm text-muted), calorie count (text-label, right-aligned). Hover: shows edit/delete actions (`<Button variant="ghost" size="icon">`).
- **Log Meal button (desktop):** `<Button size="lg"><Plus />Log meal</Button>` — top-right of meal list.
- **FAB (mobile):** `<Button size="icon" className="fixed bottom-20 right-4 h-14 w-14 rounded-full shadow-xl">` with `<Plus size={24} />`.
- **Disclaimer footer:** `<p className="text-caption text-muted border-t pt-3">` — short disclaimer + "Learn more" link.
- **Empty state (no meals):** Centered illustration area (fork/leaf SVG 80px), `<p>No meals logged yet today</p>`, `<Button>Log your first meal</Button>`.
- **Over-goal indicator:** When consumed > goal, ring fill turns `--color-error`; badge shows "+180 over goal" in error color. No shaming copy — neutral language only.

### States

| State | Visual |
|-------|--------|
| Loading | Skeleton placeholders: ring `<Skeleton className="w-48 h-48 rounded-full">`, 3 macro skeletons, 3 meal-item skeletons |
| Empty (no meals) | Empty-state component (see above) |
| At goal | Ring full, primary color, "Goal reached! 🎯" badge (optional) |
| Over goal | Ring shows over-portion in error red; warning label |
| Past day (read-only) | Edit/delete buttons hidden; "Read-only — past day" notice |

### Content

| Element | Copy |
|---------|------|
| Remaining | `{N} kcal remaining` |
| Over goal | `{N} kcal over goal` |
| Empty state | `No meals logged yet today. Start by logging your first meal.` |
| Disclaimer | `Not medical advice — BiteIQ tracks nutrition only. [Learn more]` |

### Interactions

- Tapping a meal item expands inline edit panel (same row, animated `<Collapsible>`).
- Swipe-left on mobile not implemented (web) — trash icon appears on hover/focus instead.
- Date prev/next triggers data refetch (TanStack Query).
- FAB click → opens Log Meal Sheet.
- Dashboard loads <2s p95; show skeleton immediately, replace on data arrival.

### Accessibility

- Calorie ring `role="img" aria-label="Calorie progress: 1,240 of 2,000 kcal consumed."`.
- Each macro ring `aria-label="{macro}: {consumed}g of {goal}g"`.
- FAB `aria-label="Log meal"`.
- Meal items keyboard navigable; Enter/Space opens edit; Delete key prompts confirmation.
- Disclaimer link opens in same tab (footer anchor).

---

## Screen: Log Meal Modal / Sheet

**Maps to:** US-3, US-4, US-5, US-6
**Source:** Inferred from user stories

### Layout

- **Mobile:** `<Sheet side="bottom">` — full-width bottom drawer, height 92vh, draggable handle at top (8px rounded pill).
- **Desktop:** `<Dialog className="max-w-2xl">` centered modal, height auto (max 80vh), scrollable content.
- Inside: `<Tabs defaultValue="photo">` with four triggers: Photo · Barcode · Voice · Manual. Tab bar sticky at top of modal.
- Below tabs: tab-specific content (see sub-screens below).
- Sticky bottom bar: Meal slot selector + Date/time picker + Save CTA.

### Sticky bottom bar (all tabs)

- **Meal slot:** `<Select>` — Breakfast / Lunch / Dinner / Snack. Defaults to time-of-day heuristic (e.g. before 10am = Breakfast).
- **Date/time:** `<Popover>` with `<Calendar>` and time `<Input>`. Default: "Now".
- **Save CTA:** `<Button size="lg" className="flex-1">Save meal</Button>`. Disabled until food item(s) added.

### States (global modal)

| State | Visual |
|-------|--------|
| Loading | Tab-content replaced with centered `<Loader2 className="animate-spin" />` + status copy |
| Error (API) | `<Alert variant="destructive">` at top of tab content |
| Empty (no items added yet) | Save button disabled, tooltip "Add at least one food item" |

### Accessibility

- Modal `role="dialog" aria-modal="true" aria-labelledby="log-meal-title"`.
- `<h2 id="log-meal-title">Log meal</h2>` (visually hidden or shown as sheet header).
- Esc closes the dialog.
- Focus trapped inside modal when open.
- Tab triggers have `role="tab"` with `aria-selected`.

---

## Screen: Photo Input Flow

**Maps to:** US-3, US-12, US-14
**Source:** Inferred from user stories

### Layout

Inside the Photo tab of Log Meal modal.

```
[Quota counter: "3 of 5 free recognitions left today · Upgrade]  ← free tier only
┌──────────────────────────────────────────┐
│  Upload zone (dashed border, rounded-xl) │
│  [Camera icon 48px]                      │
│  "Drag a photo here, or"                 │
│  [Take photo]  [Upload from device]      │
└──────────────────────────────────────────┘
          ↓ after image selected
┌──────────────────────────────────────────┐
│  [Image preview, max-h-48, object-cover] │
└──────────────────────────────────────────┘
          ↓ recognizing
[Spinner + "Recognizing your meal..."]
          ↓ success
[Recognized items list — see below]
```

### Components

- **Quota badge (free tier):** `<Badge variant="outline">` — "3 of 5 left today". Tap → opens Upgrade modal.
- **Upload zone:** `<label>` wrapping `<input type="file" accept="image/*" capture="environment" className="sr-only">`. Styled as dashed-border drop zone. Drag-over: border turns primary, bg tints.
- **Take photo button:** `<Button variant="outline"><Camera />Take photo</Button>` — uses `capture="environment"`.
- **Upload button:** `<Button variant="outline"><ImagePlus />Upload from device</Button>`.
- **Image preview:** `<img>` inside `<Card className="overflow-hidden">` — shows selected image before submit.
- **Recognition progress:** `<Progress>` bar (indeterminate / animated) + `<p>"Recognizing your meal…"</p>`. Duration max 5s per SLO.
- **Recognized items list:** One row per item:
  - Food name (editable `<Input>`)
  - Portion (`<Input type="number">` + unit `<Select>`)
  - Macros grid: `Calories / Protein / Carbs / Fat` — each an editable `<Input type="number">` with label.
  - Remove button: `<Button variant="ghost" size="icon"><Trash2 /></Button>`.
- **Add item button:** `<Button variant="outline" size="sm"><Plus />Add another item</Button>`.
- **AI fail state:** `<Alert variant="destructive">` — "We couldn't recognize this meal. [Enter manually →]" — the link switches to the Manual tab and preserves the uploaded photo reference.
- **Non-food warning:** `<Alert variant="warning">` — "This doesn't look like food. Please confirm or add manually." — with "Confirm & save" / "Enter manually" options.

### States

| State | Visual |
|-------|--------|
| Default (empty) | Upload zone |
| Image selected | Preview + "Recognize" button becomes active |
| Recognizing | Spinner, progress bar, upload zone hidden |
| Success | Item list, editable |
| AI fail | Destructive alert + manual fallback link |
| Quota exhausted | Upload zone replaced with `<Alert>` "Daily limit reached. [Upgrade] or [Log manually]" |
| Offline | `<Alert>` "You're offline — photo will be recognized when you reconnect." Still shows preview and "Pending sync" badge |

### Content

| Element | Copy |
|---------|------|
| Upload prompt | `Drag a photo here, or` |
| Recognizing | `Recognizing your meal…` |
| AI fail | `We couldn't identify this meal. Try again or enter it manually.` |
| Non-food | `This doesn't look like food — please confirm or add details manually.` |
| Quota full | `You've used all 5 free photo recognitions today. Upgrade for unlimited, or log manually.` |

### Accessibility

- Upload zone `role="button" tabindex="0"` — activates file picker on Enter/Space.
- Progress bar `role="progressbar" aria-valuemin="0" aria-valuemax="100" aria-valuenow="indeterminate"`.
- Alert regions `role="alert"` for AI fail and quota messages.
- Each recognized item row `aria-label="Food item {N}: {name}"`.
- Quota badge `aria-label="{N} of 5 free photo recognitions remaining today"`.

---

## Screen: Barcode Input Flow

**Maps to:** US-4
**Source:** Inferred from user stories

### Layout

Inside the Barcode tab of Log Meal modal.

```
┌──────────────────────────────┐
│  Camera viewfinder           │
│  (16:9 aspect ratio)         │
│  [scanning animation line]   │
│  "Aim at barcode"            │
└──────────────────────────────┘
[Enter barcode manually]  ← text link below viewfinder
       ↓ scan success
[Product card — see below]
       ↓ not found
[Alert: "Product not in database — enter manually →"]
```

### Components

- **Viewfinder:** `<div>` with `<video>` element behind; overlay with corner brackets (CSS borders); animated horizontal scan line (`translate-y` keyframe, `prefers-reduced-motion: none` guard).
- **Camera permission denied:** `<Alert variant="default">` — "Camera access is required for scanning. [Open settings] or [Type barcode manually]."
- **Manual barcode input:** `<Input type="tel" placeholder="Enter barcode digits" />` + `<Button>Look up</Button>`.
- **Product card (found):** `<Card>` — product name (text-h3), brand (text-body-sm text-muted), default serving, macro table. Serving size `<Input type="number">` with unit. Macros auto-recalculate on serving change.
- **Not-found alert:** `<Alert>` — "Product not found in database — add it manually." Link switches to Manual tab with barcode pre-populated.
- **Partial data warning:** Rows with missing macro values show `<Badge>—</Badge>` + `<span className="text-warning">Required — please fill in</span>`.

### States

| State | Visual |
|-------|--------|
| Default | Camera viewfinder active, scanning animation |
| Scanning | Animation active, "Aim at barcode" copy |
| Found | Product card shown, viewfinder hidden |
| Not found | Alert, manual-entry CTA |
| Camera permission denied | Permission alert, manual fallback |
| Manual entry mode | Text input + look up |
| Partial data | Warning badges on missing fields |

### Accessibility

- Viewfinder `aria-label="Barcode scanner viewfinder — point camera at barcode"`.
- Scan-line animation respects `prefers-reduced-motion` (line is static if motion reduced).
- Product card appears with `role="region" aria-label="Scanned product details"` and focus moves to it.
- Manual input `aria-label="Enter barcode number manually"`.

---

## Screen: Voice Input Flow

**Maps to:** US-5
**Source:** Inferred from user stories

### Layout

Inside the Voice tab of Log Meal modal.

```
┌──────────────────────────────┐
│  [Waveform animation or      │
│   static placeholder]        │
│  [Mic button — 72px circle]  │
│  "Tap to speak"              │
│  (or "Recording…" when live) │
└──────────────────────────────┘
[Transcript text area — appears after first word spoken]
[Edit transcript] button
       ↓ after stop
[Recognized items list — same as Photo result list]
```

### Components

- **Unsupported browser banner:** `<Alert>` at top — "Voice input isn't supported in your browser. [Use manual entry →]" — Voice tab disabled with `aria-disabled="true"`.
- **Mic button:** `<Button variant="outline" size="icon" className="h-[72px] w-[72px] rounded-full">` — default: `<Mic size={32} />` in primary color. Recording: `<MicOff />` in error red + pulsing ring animation.
- **Waveform:** Three animated bars (`scaleY` keyframe); static (height 4px) when idle; animating when recording. Respects `prefers-reduced-motion` (bars static if motion reduced).
- **Transcript area:** `<Textarea readonly>` — live updated as speech recognized. After stop: editable `<Textarea>` with "Edit transcript" hint.
- **Retry / Manual fallback:** `<Button variant="outline"><RotateCcw />Try again</Button>` + `<Button variant="ghost">Enter manually →</Button>`.
- **Permission denied:** `<Alert>` — "Microphone access is needed. [Open settings] or [Enter manually]."

### States

| State | Visual |
|-------|--------|
| Idle | Mic button, "Tap to speak" |
| Recording | Pulsing mic button, waveform animated, live transcript |
| Processing | Spinner, "Analyzing your description…" |
| Success | Item list (same as photo result) |
| Empty transcript | Alert: "We didn't catch anything — try again or enter manually." |
| Browser unsupported | Alert banner; tab disabled |
| Permission denied | Permission alert |

### Content

| Element | Copy |
|---------|------|
| Idle prompt | `Tap to speak your meal` |
| Example hint | `e.g. "two slices of toast with peanut butter and a coffee"` |
| Recording | `Recording… tap to stop` |
| Processing | `Analyzing your description…` |
| Empty | `We didn't catch anything — try again or type it manually.` |

### Accessibility

- Mic button `aria-label="Start recording"` / `"Stop recording"` toggled on state change.
- `aria-live="polite"` on transcript area so screen readers announce new words.
- Waveform `aria-hidden="true"` (decorative).
- Browser-unsupported alert has `role="alert"`.

---

## Screen: Manual Input Flow

**Maps to:** US-6
**Source:** Inferred from user stories

### Layout

Inside the Manual tab of Log Meal modal. Vertically stacked form.

```
[Search box: "Search food name…"]
  ↓ suggestions dropdown (history first, then Open Food Facts)
[Selected food fields:]
  Name: [Input]
  Portion: [Input] [Unit Select]
  Calories: [Input] kcal  ← required
  Protein: [Input] g
  Carbs: [Input] g
  Fat: [Input] g
[☆ Save as favorite]
[+ Add another item]
```

### Components

- **Search box:** `<Input>` with `<Search size={16} />` prepended. Debounced 300ms. Shows `<CommandList>` dropdown:
  - Section "Recent" (last 5 user foods, with star icon if favorited)
  - Section "Open Food Facts" (up to 10 suggestions, with database icon)
  - Footer: "Can't find it? Fill in manually"
- **Food name:** `<Input placeholder="e.g. Grilled salmon" />`.
- **Portion:** `<Input type="number" className="w-24" />` + `<Select>` — unit options: g / ml / oz / cup / tbsp / tsp / serving / piece.
- **Macro inputs:** Grid (2-col on mobile, 4-col on desktop): Calories · Protein · Carbs · Fat. Each `<Input type="number" step="0.1">`.
- **Missing macros warning:** `<Alert variant="default" className="mt-2">` — "Macro split won't reflect this item." Shown when Protein/Carbs/Fat are blank.
- **Favorite toggle:** `<Button variant="ghost" size="sm"><Star />Save as favorite</Button>` — toggles to `<StarOff />` if already favorited.

### States

| State | Visual |
|-------|--------|
| Empty search | Placeholder; no dropdown |
| Typing | Dropdown with skeleton rows during fetch |
| Suggestion selected | Form prefills; user can override |
| Missing calories | Save disabled; `<p className="text-error">Calories are required.</p>` |
| Missing macros | Warning alert (non-blocking) |
| Saving favorite | Star icon animates to filled; toast "Added to favorites" |

### Accessibility

- Search box `role="combobox" aria-autocomplete="list" aria-expanded={open}`.
- Suggestion list `role="listbox"`; each item `role="option"`.
- Required fields `aria-required="true"`.
- Macro inputs grouped in `<fieldset><legend>Nutrition per portion</legend>`.

---

## Screen: Meal History / Today's Meals

**Maps to:** US-7 (AC3 — edit/delete)
**Source:** Inferred from user stories

### Layout

A `<Sheet side="right">` (mobile) or inline expandable section (desktop) that shows the full list of meals for the selected date. Also reachable as a standalone `/history` route for past days.

### Components

- **Date picker header:** Same date navigator as Dashboard.
- **Meal group:** `<Accordion>` per slot (Breakfast / Lunch / Dinner / Snack). Expanded by default for today, collapsed for past days.
- **Meal row:** `<Card className="flex items-center p-3 gap-3">`:
  - Left: food name + macros badge row (text-body-sm).
  - Right: calorie count (text-label) + edit icon + trash icon (appear on hover/focus).
- **Edit inline panel:** Expands below meal row inside the accordion. Same fields as Manual tab (name, portion, macros, slot, time). Save / Cancel buttons.
- **Delete confirmation:** `<AlertDialog>` — "Delete {food name}?" + "This removes it from your daily totals." — Cancel / Delete (destructive button).
- **Empty slot:** `<p className="text-muted text-body-sm p-3">No meals logged for {slot}.</p>`.

### States

| State | Visual |
|-------|--------|
| Default | Accordion expanded for current slot |
| Edit mode | Row expands, edit fields visible |
| Deleting | Row fades (opacity-50) + spinner briefly |
| Deleted | Row removed with slide-up animation (200ms) |
| Empty | Empty-slot copy per slot |

### Accessibility

- Accordion uses `<details>` / shadcn `<Accordion>` with proper `aria-expanded`.
- Delete `<Button>` has `aria-label="Delete {food name}"`.
- `<AlertDialog>` for delete confirmation has `role="alertdialog"`.

---

## Screen: Weekly Insights

**Maps to:** US-8, US-15
**Source:** Inferred from user stories

### Layout

- Single-column page, max-width 800px.
- Top: week selector (prev/next week arrows + "This week" text).
- Section 1: Summary stat cards (4 cards in 2×2 grid on mobile, 4-col row on desktop).
- Section 2: Bar chart — 7-day calories vs. goal line.
- Section 3: Plain-language insights list.
- Section 4: Macro distribution stacked bar (7 days).
- Footer: disclaimer.

### Components

- **Stat cards:** `<Card className="p-4">` — label (text-caption) + number (text-num-md) + trend arrow (`<TrendingUp />` or `<TrendingDown />` in success/warning color).
  - "7-day avg calories"
  - "Days on target"
  - "Avg protein"
  - "Streak"
- **Bar chart:** Recharts `<BarChart>`. Bar fill: `--color-primary`. Goal line: `<ReferenceLine>` in `--color-muted` dashed. X-axis: day abbreviations (Mon–Sun). Y-axis: calories (kcal).
- **Insights list:** `<ul>` of `<li>` items with `<Info size={16} />` icon. Each item: plain-language observation. Max 3 items shown; "Show more" link if more available.
- **Macro stacked bar:** Recharts `<BarChart stacked>`. Three stacks: protein (blue) / carbs (orange) / fat (yellow).
- **"Less data" notice:** `<Alert variant="default">` — "Showing {N} of 7 days — log more meals for better insights." (if N < 7).
- **Empty state (0 days):** Illustration + "No meals logged this week. Start logging to see insights."
- **Disclaimer:** Same footer as Dashboard.

### States

| State | Visual |
|-------|--------|
| Loading | Skeleton for stat cards + chart placeholder |
| Fewer than 7 days | Notice alert |
| Zero days | Empty state |
| Chart animation | Bars animate on mount (200ms); disabled if `prefers-reduced-motion` |

### Accessibility

- Charts wrapped in `<figure><figcaption>` describing the data.
- `aria-label` on each stat card: "{label}: {value}".
- Insight list items are `<li>` elements within `<ul aria-label="Weekly insights">`.

---

## Screen: AI Chat Assistant

**Maps to:** US-9, US-15
**Source:** Inferred from user stories

### Layout

```
┌──────────────────────────────┐
│ [Sidebar: conversation list] │  ← desktop only
├──────────────────────────────┤
│ [Disclaimer banner]          │
│                              │
│ Chat messages (scrollable)   │
│                              │
└──────────────────────────────┘
│ Input row (sticky bottom):   │
│ [Message input] [Send]       │
└──────────────────────────────┘
```

- **Mobile:** Full-screen chat, sidebar hidden (accessible via hamburger/back button).
- **Desktop:** Left sidebar (240px) lists past conversations; main area is chat.

### Components

- **Disclaimer banner:** `<Alert variant="default" className="sticky top-0 z-10">` — "AI guidance, not medical or clinical advice — consult a professional for health concerns." Dismissible per session (`<Button variant="ghost" size="icon"><X /></Button>`).
- **Message bubble — user:** Right-aligned, `<Card>` with `--color-primary-subtle` bg. `text-body`.
- **Message bubble — assistant:** Left-aligned, `<Card>` with `--color-surface-elevated` bg. `text-body`. Streams token-by-token.
- **Streaming cursor:** `<span className="animate-blink">|</span>` appended to last word while streaming. Disabled with `prefers-reduced-motion`.
- **Source citation:** Inline `<Badge>` with meal name if assistant references a logged meal.
- **Input row:** `<Textarea rows={1} className="resize-none flex-1">` (auto-expands up to 5 rows) + `<Button size="icon"><SendHorizonal /></Button>`. Send on Enter; Shift+Enter for newline.
- **New conversation:** `<Button variant="ghost" size="sm"><Plus />New chat</Button>` at top of sidebar.
- **API error state:** `<Alert variant="destructive">` — "Assistant unavailable — try again shortly." + `<Button>Retry</Button>`.
- **Quota counter (free tier):** `<p className="text-caption">` above input — "{N} of 10 messages remaining today. [Upgrade]".

### States

| State | Visual |
|-------|--------|
| Empty (new chat) | Centered prompt suggestions: 3 example questions as `<Button variant="outline">` chips |
| Streaming | Cursor animation, send button disabled |
| Error | Destructive alert |
| Quota exhausted | Input disabled, quota-exhausted alert with upgrade CTA |

### Content

| Element | Copy |
|---------|------|
| Disclaimer | `AI guidance only — not medical, clinical, or dietetic advice. Consult a qualified professional for health concerns.` |
| Empty state chips | `"What's a high-protein breakfast under 400 cal?"` / `"How am I doing on protein this week?"` / `"Suggest a meal under 500 cal for tonight."` |
| API error | `The assistant is unavailable right now. Please try again shortly.` |

### Accessibility

- Chat area `role="log" aria-live="polite" aria-atomic="false"` — announces new messages.
- Streaming messages: live region updates each token; screen reader won't over-read (polite).
- Disclaimer `role="note" aria-label="Medical disclaimer"`.
- Send button `aria-label="Send message"`.

---

## Screen: Body Measurements

**Maps to:** US-10
**Source:** Inferred from user stories

### Layout

- Two-panel: top half = chart; bottom half = entry form + history list.
- **Mobile:** Vertical stack: chart → "Log measurement" button → history list.
- **Desktop:** Side-by-side: chart (left 60%) + form + history (right 40%).

### Components

- **Metric tabs:** `<Tabs>` — Weight · Waist · Hips · Body Fat%. Switching tab updates chart.
- **Line chart:** Recharts `<LineChart>`. X-axis: dates. Y-axis: measurement value. Time range toggle: `<ToggleGroup>` — "7 days" / "30 days" / "90 days".
- **Log measurement form:** `<Card className="p-4">` — date picker (default today) + one `<Input>` per metric (only weight required, others optional). Unit toggles per field.
- **Recalculate banner:** `<Alert>` — "Your weight has changed by {X} kg — recalculate goals? [Recalculate →]" — shown after saving a weight entry that differs from the profile weight.
- **History list:** Chronological table rows: date / values / edit / delete. Inline editing (same as meal history pattern).
- **Out-of-range confirmation:** `<AlertDialog>` — "This value looks unusual (e.g. weight 0.5 kg). Did you mean {corrected}?" with Confirm / Edit options.

### States

| State | Visual |
|-------|--------|
| Loading | Chart skeleton |
| Empty | "No measurements yet — log your first." |
| Out-of-range | Confirmation dialog |
| Recalculate needed | Banner alert |

### Accessibility

- Chart `<figure><figcaption>"{Metric} over time"</figcaption>`.
- Log form `aria-label="Log body measurement"`.
- Required weight field `aria-required="true"`.

---

## Screen: Settings / Profile

**Maps to:** US-13, US-11, US-14 (partial)
**Source:** Inferred from user stories

### Layout

- `<Tabs>` vertical (sidebar on desktop) or horizontal scroll (mobile): Profile & Goals · Notifications · Plan & Upgrade · Privacy & Data · About.

### Components

**Profile & Goals tab:**
- Same fields as Onboarding (sex / age / height / weight / activity / goal).
- `<Button size="lg">Save changes</Button>`.
- Recomputed calorie suggestion shows in real-time as fields change.

**Plan & Upgrade tab (US-14):**
- `<Card>` — "Current plan: Free". Feature comparison table: 5 photo recognitions/day vs. unlimited. Chat limit comparison.
- `<Button size="lg" className="w-full"><Sparkles />Upgrade to Paid</Button>` — opens Upgrade `<Dialog>`.

**Notifications tab:**
- `<Switch>` toggles for: Daily reminder · Weekly insights summary.
- Reminder time: `<Input type="time">`.

**Privacy & Data tab (US-11 GDPR):**
- `<Button variant="outline" className="w-full">Export my data (JSON)</Button>` — triggers download.
- `<Button variant="destructive" className="w-full">Delete my account</Button>` — opens delete confirmation dialog.

**Delete confirmation dialog:**
- `<AlertDialog>` — title "Delete your account?", body copy explaining irreversibility, type-to-confirm `<Input placeholder="Type DELETE to confirm">`, confirm button disabled until "DELETE" typed exactly, destructive variant.

**About tab:**
- App version, Terms link, Privacy link, Full disclaimer link.

### States

| State | Visual |
|-------|--------|
| Saving profile | Submit button loading state |
| Export downloading | Button shows spinner + "Preparing…" |
| Delete typing | Confirm button disabled until "DELETE" matches |
| Delete in progress | Full-page spinner overlay |
| Delete success | Redirect to landing with toast "Your account has been permanently deleted." |

### Accessibility

- Settings tabs `role="tablist"` with individual `role="tab"`.
- Delete input `aria-label="Type DELETE to confirm account deletion"` + `aria-describedby` pointing to warning text.
- Destructive confirm button `aria-disabled="true"` until condition met.

---

## Screen: Quota / Upgrade Prompt

**Maps to:** US-12, US-14
**Source:** Inferred from user stories

### Layout

Triggered as a `<Dialog>` over the current page when:
- Free user attempts 6th photo recognition.
- Free user exhausts chat quota.
- User clicks any "Upgrade" CTA.

### Components

- **Icon:** `<Sparkles size={48} className="text-accent">` — centered.
- **Title:** `<h2>You've reached your daily limit</h2>` (photo) or `<h2>Upgrade BiteIQ</h2>` (generic).
- **Body copy:** Explains what the limit is and what paid unlocks.
- **Feature list:** `<ul>` with checkmarks: "Unlimited photo recognitions" / "Unlimited AI chat" / "Priority support".
- **CTA:** `<Button size="lg" className="w-full"><Sparkles />Join the waitlist</Button>` — links to stub email capture form inside the same dialog.
- **Secondary:** `<Button variant="ghost">Log manually instead</Button>` — dismisses dialog, switches to Manual tab.
- **Email capture (stub):** `<Input type="email" placeholder="you@example.com">` + `<Button>Notify me</Button>`.

### States

| State | Visual |
|-------|--------|
| Default | Upgrade dialog |
| Email submitted | "You're on the list! We'll reach out when paid plans launch." |

### Content

| Element | Copy |
|---------|------|
| Title (quota hit) | `You've used all 5 free photo recognitions today` |
| Sub-copy | `Upgrade for unlimited photo logging, unlimited AI chat, and more.` |
| CTA | `Join the waitlist` |
| Secondary | `Log manually instead` |

### Accessibility

- `role="dialog" aria-modal="true"`.
- Focus on close button or primary CTA when dialog opens.
- `aria-label="Upgrade prompt"`.

---

## Screen: GDPR Account Settings

**Maps to:** US-11
**Source:** Inferred from user stories

> This screen is a sub-section of the Settings screen (Privacy & Data tab). Documented separately for DEV clarity.

### Components

- **Export data button:** `<Button variant="outline"><Download />Export my data (JSON)</Button>`. On click: shows loading state, then triggers file download `biteiq-export-{YYYY-MM-DD}.json`.
- **Export loading state:** Button shows `<Loader2 className="animate-spin" />` + "Preparing your export…" — disabled during generation.
- **Delete account button:** `<Button variant="destructive"><Trash2 />Delete my account</Button>` — opens confirmation dialog.
- **Delete confirmation dialog (detail):**
  - Header: "Permanently delete your account"
  - Body: "This will permanently remove all your data — profile, meals, body measurements, and chat history. This action cannot be undone."
  - Type-to-confirm input: `<Input placeholder='Type "DELETE" to confirm'>`. Case-sensitive match required.
  - Confirm button: `<Button variant="destructive" disabled={!confirmed}>Delete permanently</Button>`.
  - Cancel: `<Button variant="outline">Cancel</Button>`.
- **Post-deletion:** Redirect to `/` with `<Toast>` — "Your account has been permanently deleted."
- **Deletion error:** `<Alert variant="destructive">` — "We couldn't complete the deletion. Our team has been notified — please try again or contact support."

### Accessibility

- Type-to-confirm input `aria-label='Type DELETE in capitals to confirm account deletion'`.
- Confirm button `aria-disabled="true"` when input doesn't match.
- Dialog `role="alertdialog"` (destructive action).
- After deletion, focus returns to landing page's primary heading.

---

## Screen: Medical Disclaimer

**Maps to:** US-15
**Source:** Inferred from user stories

### Surfaces

This is not a dedicated page but a recurring UI element appearing in three forms:

**1. First-login modal:**
- `<Dialog>` shown once on first authenticated session.
- Title: "Before you start"
- Body: Full disclaimer text.
- CTA: `<Button className="w-full">I understand</Button>` — dismisses and sets `disclaimer_acknowledged = true` in user profile.
- Cannot be dismissed without clicking the button (no backdrop click to close).

**2. Footer link (all authenticated pages):**
- `<footer>` contains `<p className="text-caption text-muted">Not medical advice · <a href="/disclaimer">Learn more</a></p>`.
- `/disclaimer` is a simple page with full text + link to Terms.

**3. Inline on key surfaces:**
- Dashboard footer strip (see Dashboard screen).
- Onboarding Step 4 alert (see Onboarding screen).
- Every AI chat response (see Chat screen).
- Weekly Insights footer strip.

### Content

**Short form (inline):** `BiteIQ provides general nutrition tracking — not medical, clinical, or dietetic advice. Consult a qualified professional for health concerns.`

**Long form (first-login modal + /disclaimer page):**
```
BiteIQ is a nutrition tracking tool designed to help you log and monitor your food intake. It is not a medical device and does not provide medical, clinical, or dietetic advice.

The calorie and macro calculations shown are estimates based on publicly available formulas (Mifflin-St Jeor) and self-reported data. Individual results may vary.

AI-generated insights are for general informational purposes only. They should not be used to diagnose, treat, or manage any medical condition. Always consult a registered dietitian, nutritionist, or physician before making significant changes to your diet, especially if you have a medical condition.

BiteIQ is not liable for decisions made based on information provided in the app.
```

### Accessibility

- First-login modal cannot be dismissed via Esc or backdrop click (`onEscapeKeyDown` prevented).
- `role="dialog" aria-labelledby="disclaimer-title" aria-describedby="disclaimer-body"`.
- `/disclaimer` page has proper heading hierarchy (H1 → H2).

---

## Responsive Layout Summary

| Screen | Mobile (375px) | Tablet (768px) | Desktop (1024px+) |
|--------|---------------|---------------|------------------|
| Dashboard | Single col, bottom nav, FAB | Single col, top tabs | Sidebar nav, 2-col content |
| Log Meal | Bottom Sheet (92vh) | Bottom Sheet | Centered Dialog (max-w-2xl) |
| Weekly Insights | Single col, full-width chart | Same | Chart left, stat cards right |
| AI Chat | Full-screen, no sidebar | Full-screen | Left sidebar 240px |
| Settings | Tab scroll horizontal | Same | Vertical tab sidebar |
| Onboarding | Single col, wizard card | Same | Narrower max-width centered |

---

## PWA Shell (Offline)

- `manifest.json`: name "BiteIQ", `theme_color: #16A34A`, `background_color: #09090B`, `display: standalone`, icons at 192px + 512px.
- Service worker caches: app shell (HTML/CSS/JS), static icons, and the last-fetched dashboard data.
- Offline fallback: shows cached dashboard with `<Alert>` banner — "You're offline — data may be out of date."
- Background sync: queued photo uploads sent when connectivity returns.

---

## Animation Guidelines

- Transitions: 150ms for hover/focus, 200ms for panel open/close, 300ms for modal enter/exit.
- Easing: `ease-out` for enter, `ease-in` for exit.
- `prefers-reduced-motion: reduce` — disable ALL keyframe animations (chart entry, scan line, waveform bars, streaming cursor, ring fill animation). Static versions shown instead.
- No autoplay video or flashing content.

---

*Generated by Designer Agent — 2026-05-04. Web-only MVP. All values are design-language specifications; no production CSS or component code included.*

## Phase 12 — Vietnamese copy catalog + LanguageSwitcher

> **Appended 2026-05-08** by Designer Agent for Phase 12.2. Source: user stories US-I18N1 + US-I18N2 and user clarifications in `.agent_team/task_board.md §Phase 12`. No new screens are introduced. Existing screens are not redesigned.

---

### 1. Vietnamese Copy Catalog

#### Catalog conventions

- `key` — dot-notation string key, used as the `next-intl` message ID.
- `en` — current English copy found in the codebase.
- `vi` — Vietnamese translation (casual-friendly, "bạn" for 2nd person, numbers stay numeric, "BiteIQ" untranslated).
- Scope: **present in current build** (all 17 screens below). PWA install prompt is **future scope** — not catalogued.
- Missing-key fallback policy: if a key is absent in `en.json`, fall back to `vi.json` value (vi is source-of-truth catalog). If absent in vi too, show the raw key so engineering can spot the gap.

---

#### 1.1 Global / Shared

| key | en | vi |
|-----|----|----|
| `global.app_name` | BiteIQ | BiteIQ |
| `global.beta_badge` | Beta · Free to start | Beta · Miễn phí để bắt đầu |
| `global.disclaimer_short` | BiteIQ provides general nutrition tracking — not medical, clinical, or dietetic advice. | BiteIQ chỉ hỗ trợ theo dõi dinh dưỡng cơ bản — không phải lời khuyên y tế, lâm sàng hay dinh dưỡng trị liệu. |
| `global.disclaimer_full_link` | Full disclaimer | Xem tuyên bố đầy đủ |
| `global.save` | Save | Lưu |
| `global.cancel` | Cancel | Huỷ |
| `global.delete` | Delete | Xoá |
| `global.confirm` | Confirm | Xác nhận |
| `global.back` | Back | Quay lại |
| `global.next` | Next | Tiếp tục |
| `global.loading` | Loading… | Đang tải… |
| `global.saving` | Saving… | Đang lưu… |
| `global.saved` | Saved! | Đã lưu! |
| `global.error_generic` | Something went wrong. Please try again. | Có lỗi xảy ra. Thử lại nhé. |
| `global.retry` | Try again | Thử lại |
| `global.or` | or | hoặc |
| `global.logout` | Log out | Đăng xuất |
| `global.required_field` | Required | Bắt buộc |
| `global.optional` | Optional | Tuỳ chọn |

---

#### 1.2 Navigation

| key | en | vi |
|-----|----|----|
| `nav.dashboard` | Dashboard | Trang chủ |
| `nav.insights` | Insights | Tuần này |
| `nav.chat` | Chat | Chat |
| `nav.body` | Body | Đo lường |
| `nav.settings` | Settings | Cài đặt |
| `nav.settings_me` | Me | Tôi |

---

#### 1.3 Landing page (`/`)

| key | en | vi |
|-----|----|----|
| `landing.hero_headline` | Track meals in seconds, not minutes. | Ghi bữa ăn trong vài giây, không mất vài phút. |
| `landing.hero_subtext` | AI identifies your food from a photo, barcode, or voice. See your calories and macros instantly — no manual entry required. | AI nhận diện món ăn từ ảnh, barcode hoặc giọng nói. Xem calo và macro ngay lập tức — không cần nhập tay. |
| `landing.cta_signup` | Get started free | Bắt đầu miễn phí |
| `landing.cta_login` | Already have an account? Log in → | Đã có tài khoản? Đăng nhập → |
| `landing.feature_photo_title` | Snap a photo | Chụp ảnh |
| `landing.feature_photo_desc` | AI identifies your meal and estimates macros in seconds. | AI nhận diện món ăn và ước tính macro trong vài giây. |
| `landing.feature_barcode_title` | Scan a barcode | Quét barcode |
| `landing.feature_barcode_desc` | Instantly log packaged foods from the Open Food Facts database. | Log thực phẩm đóng gói từ database Open Food Facts ngay lập tức. |
| `landing.feature_voice_title` | Talk to log | Nói để ghi |
| `landing.feature_voice_desc` | Just say "two eggs and toast" — we handle the rest. | Chỉ cần nói "hai quả trứng và bánh mì" — BiteIQ lo phần còn lại. |
| `landing.feature_dashboard_title` | Smart dashboard | Dashboard thông minh |
| `landing.feature_dashboard_desc` | See calories and macros vs. your daily goals at a glance. | Xem calo và macro so với mục tiêu ngày chỉ trong một cái nhìn. |
| `landing.feature_insights_title` | Weekly insights | Nhìn lại tuần |
| `landing.feature_insights_desc` | Track trends and get plain-language nutrition observations. | Theo dõi xu hướng và nhận nhận xét dinh dưỡng dễ hiểu. |
| `landing.feature_chat_title` | AI nutrition coach | Coach dinh dưỡng AI |
| `landing.feature_chat_desc` | Ask questions about your diet and get personalized guidance. | Hỏi về chế độ ăn của bạn và nhận gợi ý cá nhân hoá. |

---

#### 1.4 Signup (`/signup`)

| key | en | vi |
|-----|----|----|
| `signup.title` | Create your account | Tạo tài khoản |
| `signup.email_label` | Email | Email |
| `signup.email_placeholder` | you@example.com | ban@example.com |
| `signup.password_label` | Password | Mật khẩu |
| `signup.password_placeholder` | Min. 8 characters | Tối thiểu 8 ký tự |
| `signup.show_password` | Show password | Hiện mật khẩu |
| `signup.hide_password` | Hide password | Ẩn mật khẩu |
| `signup.password_strength.too_short` | Too short | Quá ngắn |
| `signup.password_strength.weak` | Weak | Yếu |
| `signup.password_strength.fair` | Fair | Tạm được |
| `signup.password_strength.strong` | Strong | Mạnh |
| `signup.submit` | Sign up | Đăng ký |
| `signup.google_cta` | Continue with Google | Tiếp tục với Google |
| `signup.already_have_account` | Already have an account? | Đã có tài khoản? |
| `signup.login_link` | Log in | Đăng nhập |
| `signup.error.already_registered` | This email is already registered. Log in instead? | Email này đã được đăng ký. Đăng nhập thay thế nhé? |
| `signup.error.oauth_cancelled` | Sign-in cancelled | Đăng nhập đã bị huỷ |
| `signup.email_sent.title` | Check your inbox | Kiểm tra hộp thư |
| `signup.email_sent.body` | We sent a confirmation link to {email}. Click the link to activate your account. | Chúng tôi đã gửi link xác nhận đến {email}. Nhấn vào link để kích hoạt tài khoản. |
| `signup.email_sent.resend` | Resend email | Gửi lại email |
| `signup.email_sent.go_login` | Back to log in | Về trang đăng nhập |

---

#### 1.5 Login (`/login`)

| key | en | vi |
|-----|----|----|
| `login.title` | Welcome back | Chào mừng trở lại |
| `login.email_label` | Email | Email |
| `login.password_label` | Password | Mật khẩu |
| `login.show_password` | Show password | Hiện mật khẩu |
| `login.hide_password` | Hide password | Ẩn mật khẩu |
| `login.submit` | Log in | Đăng nhập |
| `login.google_cta` | Continue with Google | Tiếp tục với Google |
| `login.no_account` | Don't have an account? | Chưa có tài khoản? |
| `login.signup_link` | Sign up | Đăng ký |
| `login.forgot_password` | Forgot password? | Quên mật khẩu? |
| `login.error.wrong_credentials` | Incorrect email or password. | Email hoặc mật khẩu không đúng. |

---

#### 1.6 Forgot password (`/forgot-password`)

| key | en | vi |
|-----|----|----|
| `forgot_password.title` | Reset password | Đặt lại mật khẩu |
| `forgot_password.body` | Enter your email and we'll send you a reset link. | Nhập email và chúng tôi sẽ gửi link đặt lại mật khẩu cho bạn. |
| `forgot_password.email_label` | Email | Email |
| `forgot_password.submit` | Send reset link | Gửi link đặt lại |
| `forgot_password.sent_title` | Email sent | Đã gửi email |
| `forgot_password.sent_body` | Check your inbox for a password reset link. | Kiểm tra hộp thư để nhận link đặt lại mật khẩu. |
| `forgot_password.back_to_login` | Back to log in | Về trang đăng nhập |

---

#### 1.7 Onboarding 4-step wizard (`/onboarding`)

| key | en | vi |
|-----|----|----|
| `onboarding.step_label.profile` | Profile | Thông tin cá nhân |
| `onboarding.step_label.activity` | Activity | Mức hoạt động |
| `onboarding.step_label.goal` | Goal | Mục tiêu |
| `onboarding.step_label.targets` | Your targets | Mục tiêu của bạn |
| `onboarding.step_counter` | Step {current} of {total} | Bước {current}/{total} |
| `onboarding.profile.title` | Tell us about yourself | Cho chúng tôi biết về bạn |
| `onboarding.profile.sex_label` | Sex | Giới tính |
| `onboarding.profile.sex_male` | Male | Nam |
| `onboarding.profile.sex_female` | Female | Nữ |
| `onboarding.profile.age_label` | Age (years) | Tuổi |
| `onboarding.profile.age_placeholder` | e.g. 28 | vd. 28 |
| `onboarding.profile.height_label` | Height | Chiều cao |
| `onboarding.profile.height_cm_placeholder` | cm | cm |
| `onboarding.profile.weight_label` | Current weight | Cân nặng hiện tại |
| `onboarding.profile.weight_kg_placeholder` | kg | kg |
| `onboarding.profile.units_metric` | Metric (kg / cm) | Metric (kg / cm) |
| `onboarding.profile.units_imperial` | Imperial (lb / in) | Imperial (lb / in) |
| `onboarding.profile.error.sex_required` | Please select your sex. | Vui lòng chọn giới tính. |
| `onboarding.profile.error.age_required` | Age is required. | Tuổi là bắt buộc. |
| `onboarding.profile.error.age_under_13` | BiteIQ is not available for users under 13. | BiteIQ chưa hỗ trợ người dùng dưới 13 tuổi. |
| `onboarding.profile.error.age_invalid` | Please enter a valid age. | Vui lòng nhập tuổi hợp lệ. |
| `onboarding.profile.error.height_range` | Height must be between 50–250 cm. | Chiều cao phải từ 50–250 cm. |
| `onboarding.profile.error.weight_range` | Weight must be between 20–300 kg. | Cân nặng phải từ 20–300 kg. |
| `onboarding.activity.title` | How active are you? | Mức độ hoạt động của bạn? |
| `onboarding.activity.sedentary_label` | Sedentary | Ít vận động |
| `onboarding.activity.sedentary_sub` | Little or no exercise | Hầu như không tập thể dục |
| `onboarding.activity.light_label` | Lightly active | Hoạt động nhẹ |
| `onboarding.activity.light_sub` | 1–3 days/week | 1–3 ngày/tuần |
| `onboarding.activity.moderate_label` | Moderately active | Hoạt động vừa |
| `onboarding.activity.moderate_sub` | 3–5 days/week | 3–5 ngày/tuần |
| `onboarding.activity.active_label` | Active | Năng động |
| `onboarding.activity.active_sub` | 6–7 days/week | 6–7 ngày/tuần |
| `onboarding.activity.very_active_label` | Very active | Rất năng động |
| `onboarding.activity.very_active_sub` | Physical job or 2× daily training | Công việc thể lực hoặc tập 2 buổi/ngày |
| `onboarding.activity.error.required` | Please select your activity level. | Vui lòng chọn mức độ hoạt động. |
| `onboarding.goal.title` | What's your goal? | Mục tiêu của bạn là gì? |
| `onboarding.goal.lose_label` | Lose weight | Giảm cân |
| `onboarding.goal.lose_delta` | −500 kcal/day | −500 kcal/ngày |
| `onboarding.goal.maintain_label` | Maintain weight | Duy trì cân nặng |
| `onboarding.goal.maintain_delta` | ±0 kcal/day | ±0 kcal/ngày |
| `onboarding.goal.gain_label` | Gain weight | Tăng cân |
| `onboarding.goal.gain_delta` | +500 kcal/day | +500 kcal/ngày |
| `onboarding.goal.target_weight_label` | Target weight (optional) | Cân nặng mục tiêu (tuỳ chọn) |
| `onboarding.goal.error.required` | Please select your goal. | Vui lòng chọn mục tiêu. |
| `onboarding.targets.title` | Your daily targets | Mục tiêu hàng ngày của bạn |
| `onboarding.targets.calories_label` | Calories | Calo |
| `onboarding.targets.protein_label` | Protein | Đạm |
| `onboarding.targets.carbs_label` | Carbs | Tinh bột |
| `onboarding.targets.fat_label` | Fat | Béo |
| `onboarding.targets.customize_show` | Customize targets | Tuỳ chỉnh mục tiêu |
| `onboarding.targets.customize_hide` | Hide custom targets | Ẩn tuỳ chỉnh |
| `onboarding.targets.calories_unit` | Calories (kcal) | Calo (kcal) |
| `onboarding.targets.protein_unit` | Protein (g) | Đạm (g) |
| `onboarding.targets.carbs_unit` | Carbs (g) | Tinh bột (g) |
| `onboarding.targets.fat_unit` | Fat (g) | Béo (g) |
| `onboarding.targets.confirm_btn` | Looks good — let's go! | Ổn rồi — bắt đầu thôi! |
| `onboarding.calculate_btn` | Calculate targets | Tính mục tiêu |
| `onboarding.error.save_failed` | Failed to save profile. | Không lưu được thông tin. Thử lại nhé. |
| `onboarding.disclaimer_under13` | Please consult a healthcare professional before making significant dietary changes. | Hãy tham khảo chuyên gia y tế trước khi thay đổi chế độ ăn đáng kể. |

---

#### 1.8 Disclaimer modal (first-login)

> Note: The full legal text on the Disclaimer page MUST be reviewed by legal counsel before launch in Vietnamese. The translation below is best-effort and is marked **[needs legal review before launch]**.

| key | en | vi |
|-----|----|----|
| `disclaimer.title` | Medical Disclaimer | Tuyên bố miễn trách nhiệm y tế |
| `disclaimer.last_updated` | Last updated: {date} | Cập nhật lần cuối: {date} |
| `disclaimer.section_not_advice_title` | Not Medical Advice | Không phải lời khuyên y tế |
| `disclaimer.section_not_advice_body` | BiteIQ is a nutrition tracking tool provided for general informational and educational purposes only. The calorie targets, macro recommendations, AI-generated insights, and any other content provided by BiteIQ are not medical advice and should not be treated as such. | BiteIQ là công cụ theo dõi dinh dưỡng chỉ dành cho mục đích thông tin và giáo dục. Mục tiêu calo, gợi ý macro, thông tin từ AI và mọi nội dung khác từ BiteIQ không phải lời khuyên y tế và không nên được xem như vậy. **[needs legal review before launch]** |
| `disclaimer.section_accuracy_title` | AI Accuracy Limitations | Giới hạn độ chính xác của AI |
| `disclaimer.section_accuracy_body` | Calorie and nutrient estimates generated by AI photo recognition or text parsing are approximate. Actual nutritional content varies based on preparation methods, portion sizes, ingredient variations, and other factors. Always verify important nutritional information against authoritative sources. | Ước tính calo và dinh dưỡng từ AI qua ảnh hoặc văn bản là tương đối. Hàm lượng dinh dưỡng thực tế thay đổi tuỳ cách chế biến, khẩu phần và nguyên liệu. Luôn kiểm chứng thông tin dinh dưỡng quan trọng từ nguồn đáng tin cậy. **[needs legal review before launch]** |
| `disclaimer.section_consult_title` | Consult a Professional | Tham khảo chuyên gia |
| `disclaimer.section_consult_body` | Before making significant changes to your diet or exercise regimen, especially if you have a pre-existing medical condition, are pregnant, or are under the age of 18, consult a qualified healthcare provider, registered dietitian, or nutritionist. BiteIQ is not a substitute for professional medical, dietary, or fitness advice, diagnosis, or treatment. | Trước khi thay đổi đáng kể chế độ ăn hoặc tập luyện — đặc biệt nếu bạn có bệnh nền, đang mang thai hoặc dưới 18 tuổi — hãy tham khảo bác sĩ, chuyên gia dinh dưỡng có chứng chỉ. BiteIQ không thay thế tư vấn y tế, dinh dưỡng hoặc thể dục chuyên nghiệp. **[needs legal review before launch]** |
| `disclaimer.accept_btn` | I understand — continue | Tôi hiểu rồi — tiếp tục |
| `disclaimer.back_link` | Back | Quay lại |

---

#### 1.9 Dashboard (`/dashboard`)

| key | en | vi |
|-----|----|----|
| `dashboard.today_label` | Today | Hôm nay |
| `dashboard.calories_label` | Calories | Calo |
| `dashboard.calories_ring_aria` | Calories: {consumed} of {goal} | Calo: {consumed} trên {goal} |
| `dashboard.kcal_remaining` | {n} kcal remaining | Còn {n} kcal |
| `dashboard.kcal_over` | {n} kcal over goal | Vượt {n} kcal |
| `dashboard.protein_label` | Protein | Đạm |
| `dashboard.carbs_label` | Carbs | Tinh bột |
| `dashboard.fat_label` | Fat | Béo |
| `dashboard.slot.breakfast` | Breakfast | Bữa sáng |
| `dashboard.slot.lunch` | Lunch | Bữa trưa |
| `dashboard.slot.dinner` | Dinner | Bữa tối |
| `dashboard.slot.snack` | Snack | Bữa phụ |
| `dashboard.log_meal_btn` | Log meal | Ghi bữa ăn |
| `dashboard.empty_state_title` | Nothing logged yet | Chưa có gì được ghi hôm nay |
| `dashboard.empty_state_body` | Tap "Log meal" to add your first meal of the day. | Nhấn "Ghi bữa ăn" để thêm bữa đầu tiên trong ngày. |
| `dashboard.edit_meal_label` | Edit meal | Chỉnh bữa ăn |
| `dashboard.delete_meal_label` | Delete meal | Xoá bữa ăn |
| `dashboard.delete_meal_confirm` | Delete this meal entry? | Xoá bữa ăn này? |
| `dashboard.nav_prev_day` | Previous day | Ngày trước |
| `dashboard.nav_next_day` | Next day | Ngày sau |
| `dashboard.not_medical_advice` | BiteIQ provides general nutrition tracking — not medical, clinical, or dietetic advice. | BiteIQ chỉ hỗ trợ theo dõi dinh dưỡng cơ bản — không phải lời khuyên y tế, lâm sàng hay dinh dưỡng trị liệu. |
| `dashboard.track_progress_btn` | Track progress | Theo dõi tiến độ |

---

#### 1.10 Log Meal modal — shared

| key | en | vi |
|-----|----|----|
| `log_meal.modal_title` | Log meal | Ghi bữa ăn |
| `log_meal.close_btn` | Close | Đóng |
| `log_meal.tab.photo` | Photo | Ảnh |
| `log_meal.tab.barcode` | Barcode | Barcode |
| `log_meal.tab.voice` | Voice | Giọng nói |
| `log_meal.tab.manual` | Manual | Nhập tay |
| `log_meal.slot_label` | Meal slot | Bữa |
| `log_meal.slot.breakfast` | Breakfast | Bữa sáng |
| `log_meal.slot.lunch` | Lunch | Bữa trưa |
| `log_meal.slot.dinner` | Dinner | Bữa tối |
| `log_meal.slot.snack` | Snack | Bữa phụ |
| `log_meal.date_label` | Date & time | Ngày & giờ |
| `log_meal.save_btn` | Save meal | Lưu bữa ăn |
| `log_meal.error.name_required` | Food name is required for all items. | Tên món ăn là bắt buộc cho mỗi mục. |
| `log_meal.error.calories_required` | Calories are required. Please enter a value. | Calo là bắt buộc. Vui lòng nhập giá trị. |
| `log_meal.error.save_failed` | Failed to save meal. | Không lưu được bữa ăn. |
| `log_meal.item.food_name_label` | Food name | Tên món |
| `log_meal.item.calories_label` | Calories | Calo |
| `log_meal.item.protein_label` | Protein (g) | Đạm (g) |
| `log_meal.item.carbs_label` | Carbs (g) | Tinh bột (g) |
| `log_meal.item.fat_label` | Fat (g) | Béo (g) |
| `log_meal.item.serving_label` | Serving (g) | Khẩu phần (g) |
| `log_meal.item.remove` | Remove item | Bỏ mục này |
| `log_meal.add_item` | Add item | Thêm món |
| `log_meal.macro_warning` | Macro split won't reflect this item. | Tỉ lệ macro sẽ không phản ánh món này. |

---

#### 1.11 Log Meal — Photo tab

| key | en | vi |
|-----|----|----|
| `log_meal.photo.take_photo_btn` | Take photo | Chụp ảnh |
| `log_meal.photo.upload_btn` | Upload image | Tải ảnh lên |
| `log_meal.photo.analyzing` | Analyzing your meal… | Đang phân tích bữa ăn… |
| `log_meal.photo.ai_result_title` | Recognized items | Món nhận diện được |
| `log_meal.photo.error.unrecognized` | We couldn't identify this meal. Try again or enter it manually. | BiteIQ không nhận diện được món này. Thử lại hoặc nhập tay nhé. |
| `log_meal.photo.error.all_providers_exhausted` | All AI providers are temporarily out of free capacity. Please try again in a few minutes. | Tất cả nhà cung cấp AI đang tạm hết quota miễn phí. Thử lại sau vài phút nhé. |
| `log_meal.photo.error.recognition_failed` | Photo recognition failed — please try again or log this meal manually. | Nhận diện ảnh thất bại — thử lại hoặc nhập tay bữa ăn này nhé. |
| `log_meal.photo.provider_label` | Recognized by: {provider} | Nhận diện bởi: {provider} |
| `log_meal.photo.switch_manual` | Log manually instead | Nhập tay thay thế |

---

#### 1.12 Log Meal — Barcode tab

| key | en | vi |
|-----|----|----|
| `log_meal.barcode.viewfinder_hint` | Point your camera at the barcode | Hướng camera vào barcode |
| `log_meal.barcode.manual_entry_placeholder` | Enter barcode digits | Nhập số barcode |
| `log_meal.barcode.lookup_btn` | Look up | Tra cứu |
| `log_meal.barcode.scanning` | Scanning… | Đang quét… |
| `log_meal.barcode.error.not_found` | Product not in database — enter manually | Không tìm thấy sản phẩm — nhập tay nhé |
| `log_meal.barcode.error.camera_denied` | Camera access denied. Allow camera access in browser settings, or switch to manual entry. | Không có quyền truy cập camera. Cấp quyền trong cài đặt trình duyệt hoặc chuyển sang nhập tay. |
| `log_meal.barcode.error.lookup_failed` | Lookup failed. | Tra cứu thất bại. |
| `log_meal.barcode.product_details_label` | Scanned product details | Thông tin sản phẩm đã quét |
| `log_meal.barcode.per100g_fallback_note` | Nutrition data shown per 100g (serving size not available). | Thông tin dinh dưỡng hiển thị theo 100g (chưa có thông tin khẩu phần). |

---

#### 1.13 Log Meal — Voice tab

| key | en | vi |
|-----|----|----|
| `log_meal.voice.start_hint` | Tap to speak your meal | Nhấn để nói bữa ăn của bạn |
| `log_meal.voice.recording` | Recording… tap to stop | Đang ghi… nhấn để dừng |
| `log_meal.voice.analyzing` | Analyzing your description… | Đang phân tích mô tả của bạn… |
| `log_meal.voice.parse_btn` | Parse food items | Phân tích món ăn |
| `log_meal.voice.error.not_caught` | We didn't catch anything — try again or type it manually. | Không nghe được gì — thử lại hoặc nhập tay nhé. |
| `log_meal.voice.error.mic_denied` | Microphone access denied or unavailable. | Không có quyền truy cập micro hoặc micro không khả dụng. |
| `log_meal.voice.error.not_supported` | Voice input is not supported in this browser. Please use Chrome or Safari, or switch to Manual entry. | Nhập bằng giọng nói không được hỗ trợ trên trình duyệt này. Dùng Chrome hoặc Safari, hoặc chuyển sang nhập tay. |
| `log_meal.voice.error.parsing_failed` | Parsing failed. | Phân tích thất bại. |

---

#### 1.14 Log Meal — Manual tab

| key | en | vi |
|-----|----|----|
| `log_meal.manual.search_placeholder` | Search food name… | Tìm tên món ăn… |
| `log_meal.manual.suggestions_recent` | Recent | Gần đây |
| `log_meal.manual.suggestions_database` | Database | Cơ sở dữ liệu |
| `log_meal.manual.save_favorite` | Save as favourite | Lưu vào yêu thích |
| `log_meal.manual.calories_warning` | Macro split won't reflect this item. | Tỉ lệ macro sẽ không phản ánh món này. |

---

#### 1.15 History / Meals list

| key | en | vi |
|-----|----|----|
| `history.title` | Meal history | Lịch sử bữa ăn |
| `history.empty_title` | No meals logged | Chưa có bữa ăn nào |
| `history.empty_body` | Start logging to see your meal history here. | Bắt đầu ghi để xem lịch sử bữa ăn ở đây. |
| `history.total_calories` | Total calories | Tổng calo |

---

#### 1.16 Weekly Insights (`/insights`)

| key | en | vi |
|-----|----|----|
| `insights.title` | Weekly insights | Nhìn lại tuần |
| `insights.days_logged_label` | Days logged | Ngày đã ghi |
| `insights.on_target_label` | On-target days | Ngày đạt mục tiêu |
| `insights.avg_calories_label` | Avg calories | Calo TB |
| `insights.trend_label` | Trend | Xu hướng |
| `insights.trend.improving` | Improving | Đang tiến bộ |
| `insights.trend.steady` | Steady | Ổn định |
| `insights.trend.off_track` | Off-track | Lệch mục tiêu |
| `insights.chart_goal_line` | Goal | Mục tiêu |
| `insights.macro_avg_protein` | Protein | Đạm |
| `insights.macro_avg_carbs` | Carbs | Tinh bột |
| `insights.macro_avg_fat` | Fat | Béo |
| `insights.partial_data_notice` | Showing {n} of 7 days — log more for better insights. | Đang hiển thị {n}/7 ngày — ghi thêm để có thông tin chính xác hơn. |
| `insights.empty_title` | No data yet | Chưa có dữ liệu |
| `insights.empty_body` | Log meals this week to see your insights. | Ghi bữa ăn trong tuần để xem thông tin tổng kết. |
| `insights.prev_week` | Previous week | Tuần trước |
| `insights.next_week` | Next week | Tuần sau |
| `insights.disclaimer` | BiteIQ provides general nutrition tracking — not medical, clinical, or dietetic advice. | BiteIQ chỉ hỗ trợ theo dõi dinh dưỡng cơ bản — không phải lời khuyên y tế, lâm sàng hay dinh dưỡng trị liệu. |

---

#### 1.17 AI Chat (`/chat`)

| key | en | vi |
|-----|----|----|
| `chat.title` | Nutrition coach | Coach dinh dưỡng |
| `chat.input_placeholder` | Ask your nutrition coach… | Hỏi coach dinh dưỡng của bạn… |
| `chat.send_btn` | Send | Gửi |
| `chat.clear_history_btn` | Clear history | Xoá lịch sử |
| `chat.clear_history_confirm` | Clear all chat history? | Xoá toàn bộ lịch sử chat? |
| `chat.suggestion.week` | How am I doing this week? | Tuần này tôi ăn uống thế nào? |
| `chat.suggestion.protein_lunch` | What should I eat for a high-protein lunch? | Nên ăn gì cho bữa trưa nhiều đạm? |
| `chat.suggestion.macro_targets` | Explain my macro targets | Giải thích mục tiêu macro của tôi |
| `chat.disclaimer_per_message` | AI guidance — not medical or clinical advice. Consult a professional for health concerns. | Gợi ý từ AI — không phải lời khuyên y tế hay lâm sàng. Tham khảo chuyên gia cho vấn đề sức khoẻ. |
| `chat.error.unavailable` | Assistant unavailable, try again shortly. | Trợ lý không khả dụng, thử lại sau nhé. |
| `chat.error.send_failed` | Error sending message | Lỗi khi gửi tin nhắn |
| `chat.streaming_cursor_label` | Assistant is typing… | Trợ lý đang trả lời… |

---

#### 1.18 Body Measurements (`/body`)

| key | en | vi |
|-----|----|----|
| `body.title` | Body measurements | Đo lường cơ thể |
| `body.add_entry_btn` | Add measurement | Thêm số đo |
| `body.metric.weight` | Weight | Cân nặng |
| `body.metric.body_fat_pct` | Body Fat % | Tỉ lệ mỡ (%) |
| `body.metric.waist` | Waist | Vòng eo |
| `body.metric.hips` | Hips | Vòng hông |
| `body.metric.chest` | Chest | Vòng ngực |
| `body.date_label` | Date | Ngày |
| `body.save_btn` | Save | Lưu |
| `body.edit_btn` | Edit | Chỉnh sửa |
| `body.delete_btn` | Delete | Xoá |
| `body.delete_confirm` | Delete this measurement? | Xoá số đo này? |
| `body.out_of_range_confirm` | This value seems unusual. Save anyway? | Giá trị này có vẻ bất thường. Vẫn lưu? |
| `body.weight_changed_banner` | Your weight changed by {delta}kg — recalculate goals? | Cân nặng thay đổi {delta}kg — tính lại mục tiêu? |
| `body.recalculate_btn` | Recalculate | Tính lại |
| `body.empty_title` | No measurements yet | Chưa có số đo nào |
| `body.empty_body` | Add your first measurement to track progress over time. | Thêm số đo đầu tiên để theo dõi tiến trình theo thời gian. |
| `body.error.save_failed` | Failed to save. | Không lưu được. |

---

#### 1.19 Settings (3 tabs)

**Tab labels:**

| key | en | vi |
|-----|----|----|
| `settings.tab.profile` | Profile & Goals | Hồ sơ & Mục tiêu |
| `settings.tab.plan` | Plan | Gói dịch vụ |
| `settings.tab.privacy` | Privacy & Data | Quyền riêng tư & Dữ liệu |

**Profile & Goals tab:**

| key | en | vi |
|-----|----|----|
| `settings.profile.title` | Profile & Goals | Hồ sơ & Mục tiêu |
| `settings.profile.language_label` | Language | Ngôn ngữ |
| `settings.profile.display_name_label` | Display name | Tên hiển thị |
| `settings.profile.email_label` | Email | Email |
| `settings.profile.sex_label` | Sex | Giới tính |
| `settings.profile.age_label` | Age | Tuổi |
| `settings.profile.height_label` | Height | Chiều cao |
| `settings.profile.weight_label` | Weight | Cân nặng |
| `settings.profile.units_label` | Units | Đơn vị đo |
| `settings.profile.units_metric` | Metric | Metric |
| `settings.profile.units_imperial` | Imperial | Imperial |
| `settings.profile.activity_label` | Activity level | Mức hoạt động |
| `settings.profile.activity.sedentary` | Sedentary | Ít vận động |
| `settings.profile.activity.light` | Lightly Active | Hoạt động nhẹ |
| `settings.profile.activity.moderate` | Moderately Active | Hoạt động vừa |
| `settings.profile.activity.active` | Active | Năng động |
| `settings.profile.activity.very_active` | Very Active | Rất năng động |
| `settings.profile.goals_section` | Daily Goals | Mục tiêu hàng ngày |
| `settings.profile.calories_goal` | Calories | Calo |
| `settings.profile.protein_goal` | Protein | Đạm |
| `settings.profile.carbs_goal` | Carbs | Tinh bột |
| `settings.profile.fat_goal` | Fat | Béo |
| `settings.profile.save_btn` | Save Changes | Lưu thay đổi |
| `settings.profile.saved` | Saved! | Đã lưu! |
| `settings.profile.error.save_failed` | Failed to save | Không lưu được |

**Plan tab:**

| key | en | vi |
|-----|----|----|
| `settings.plan.current_plan_label` | Your plan | Gói của bạn |
| `settings.plan.free_title` | Free | Miễn phí |
| `settings.plan.paid_title` | Pro | Pro |
| `settings.plan.price` | $7.99/mo | $7.99/tháng |
| `settings.plan.feature.unlimited_manual` | Unlimited manual logging | Ghi thủ công không giới hạn |
| `settings.plan.feature.ai_text` | AI text parsing | Phân tích văn bản AI |
| `settings.plan.feature.photo_free` | Photo recognitions (free provider quota) | Nhận diện ảnh (quota miễn phí của nhà cung cấp) |
| `settings.plan.feature.chat_limit` | 10 AI chat messages / day | 10 tin nhắn AI chat / ngày |
| `settings.plan.feature.insights` | Weekly insights | Tổng kết tuần |
| `settings.plan.feature.body` | Body measurements | Theo dõi số đo cơ thể |
| `settings.plan.feature.everything_free` | Everything in Free | Tất cả tính năng gói Miễn phí |
| `settings.plan.feature.unlimited_photo` | Unlimited photo recognitions | Nhận diện ảnh không giới hạn |
| `settings.plan.feature.unlimited_chat` | Unlimited AI chat | AI chat không giới hạn |
| `settings.plan.feature.advanced_analytics` | Advanced analytics | Phân tích nâng cao |
| `settings.plan.feature.export` | Export to CSV / PDF | Xuất dữ liệu CSV / PDF |
| `settings.plan.feature.priority_support` | Priority support | Hỗ trợ ưu tiên |
| `settings.plan.upgrade_btn` | Upgrade to Pro | Nâng lên Pro |
| `settings.plan.upgrade_coming_soon` | Coming soon — join waitlist | Sắp ra mắt — đăng ký danh sách chờ |

**Privacy & Data tab:**

| key | en | vi |
|-----|----|----|
| `settings.privacy.title` | Privacy & Data | Quyền riêng tư & Dữ liệu |
| `settings.privacy.export_btn` | Download export | Tải xuống dữ liệu |
| `settings.privacy.exporting` | Exporting… | Đang xuất… |
| `settings.privacy.export_failed` | Export failed | Xuất thất bại |
| `settings.privacy.delete_account_btn` | Delete my account | Xoá tài khoản của tôi |
| `settings.privacy.deleting` | Deleting… | Đang xoá… |
| `settings.privacy.delete_confirm_prompt` | Type DELETE to confirm | Nhập DELETE để xác nhận |
| `settings.privacy.delete_irreversible` | This action is irreversible. All your data will be permanently deleted. | Hành động này không thể hoàn tác. Toàn bộ dữ liệu của bạn sẽ bị xoá vĩnh viễn. |
| `settings.privacy.delete_failed` | Deletion failed | Xoá thất bại |

---

#### 1.20 Auth callback page

| key | en | vi |
|-----|----|----|
| `auth_callback.loading` | Signing you in… | Đang đăng nhập… |
| `auth_callback.error` | Sign-in failed. Please try again. | Đăng nhập thất bại. Vui lòng thử lại. |
| `auth_callback.back_to_login` | Back to log in | Về trang đăng nhập |

---

**Total string count: approximately 232 keys across 17 screens + global/nav.**

---

### 2. LanguageSwitcher Component Spec

#### 2.1 Placement decision

**Location: Settings → Profile & Goals tab, at the very top of the form, above all other input fields.**

Rationale for Profile tab over Account tab:
- Language is a personal preference tied to the user's identity — it sits naturally alongside display name, units, and activity level (other "who I am" settings).
- The Account (Privacy & Data) tab is reserved for destructive/compliance actions (export, delete). Placing language there buries a commonly-used preference next to rarely-needed controls.
- Goal tab is focused on numeric targets — off-topic for a language toggle.
- Profile tab is the first tab the user sees when opening Settings, maximising discoverability without requiring a dedicated Settings section.

#### 2.2 Visual spec

**Component name:** `LanguageSwitcher`

**Component basis:** Use `shadcn/ui ToggleGroup` (built on Radix UI `ToggleGroup`) if the project's shadcn install includes it; the `role="radiogroup"` + `role="radio"` semantics align exactly. If `ToggleGroup` is not installed, build a custom component with identical ARIA roles — do not use `<Select>` (dropdown hides options behind a click, worse affordance for a 2-option control).

**Layout:**

```
Language
+---------------------------+---------------------------+
|  ● Tiếng Việt             |   English                 |
+---------------------------+---------------------------+
```

- Label: uses `settings.profile.language_label` translation key. Same `text-label` style as other form field labels, `color: var(--color-foreground-secondary)`, `margin-bottom: 8px`.
- Two-button segmented control:
  - Left button: "Tiếng Việt"
  - Right button: "English"
  - Width: `w-full` on mobile (max-width 767px), `max-w-[320px]` on desktop.
  - Height: 40px (touch target meets WCAG 2.5.5 minimum for web).
  - Active button (current locale): `background: var(--color-primary)`, `color: var(--color-primary-foreground)`, border: none.
  - Inactive button: `background: transparent`, `color: var(--color-foreground-secondary)`, `border: 1px solid var(--color-border)`.
  - Separator between buttons: 1px `var(--color-border)`.
  - Border-radius: outer corners `radius-md` (6px); inner corners 0.
  - Font: `text-label` (14px, weight 500).

#### 2.3 Interaction flow

1. **User taps inactive button** (e.g. taps "English" while on Tieng Viet):
   - Immediately move visual active state to the tapped button (optimistic UI).
   - Show a small `Loader2` spinner (16px) inside the now-active button, replacing its text label, for up to 800ms.
2. **In flight:** `PATCH /api/profile` with `{ locale_preference: "en" | "vi" }` (Architect specifies exact contract).
3. **On success (within 800ms):**
   - Spinner disappears, button label returns.
   - Call `router.refresh()` for soft server-side re-render in new locale. No hard browser reload.
   - `<html lang>` attribute updates to `"en"` or `"vi"`.
4. **On failure:**
   - Revert button state to the previous active locale.
   - Show a non-blocking toast using existing shadcn/ui Toast / Sonner:
     - In Vietnamese: "Khong luu duoc — thu lai" (key: `settings.language_switcher.error.save_failed_vi`)
     - In English: "Couldn't save — try again" (key: `settings.language_switcher.error.save_failed_en`)
   - Toast duration: 4000ms.

Additional string keys for this component:

| key | en | vi |
|-----|----|----|
| `settings.language_switcher.vi_label` | Tieng Viet | Tiếng Việt |
| `settings.language_switcher.en_label` | English | English |
| `settings.language_switcher.aria_group` | Language selection | Chon ngon ngu |
| `settings.language_switcher.loading_aria` | Saving language preference… | Dang luu ngon ngu… |
| `settings.language_switcher.error.save_failed_vi` | Khong luu duoc — thu lai | Không lưu được — thử lại |
| `settings.language_switcher.error.save_failed_en` | Couldn't save — try again | Couldn't save — try again |

#### 2.4 Accessibility

- Outer container: `role="radiogroup"`, `aria-label` from `settings.language_switcher.aria_group`.
- Each button: `role="radio"`, `aria-checked={isActive}`.
- Keyboard navigation: Left/Right arrow keys move between options; Space/Enter selects. This matches Radix UI `ToggleGroup` default behaviour.
- Loading state: active button gets `aria-label` from `settings.language_switcher.loading_aria` while spinner is visible; restored to label text on resolve.
- Active vs. inactive uses filled/outlined visual treatment AND `aria-checked` boolean — not colour alone (satisfies WCAG 1.4.1 Use of Color).
- `<html lang>` updates to `"vi"` or `"en"` on each successful locale switch.

---

### 3. Layout Impact Audit

Vietnamese text can run 15–25% longer than English equivalents in some UI contexts.

**Hard requirement (US-I18N1 AC6): No horizontal scroll at 360px viewport in Vietnamese.**

#### 3.1 Dashboard

| Component | EN | VI | Risk | Recommendation |
|-----------|----|----|------|----------------|
| Log meal button | Log meal | Ghi bua an | Low | No change needed |
| Track progress button | Track progress | Theo doi tien do | Medium — longer | Allow 2-line wrap; min-height 44px |
| Dashboard nav label | Dashboard | Trang chu | Low | No change |
| Slot labels | Breakfast / Lunch / Dinner / Snack | Bua sang / Bua trua / Bua toi / Bua phu | Low | No change |
| Today label | Today | Hom nay | None | No change |

#### 3.2 Navigation bar (bottom tab, 360px viewport)

5 tabs at 72px each. All VI labels are safe at `text-caption` (12px):

| Tab | EN | VI | Risk |
|-----|----|----|------|
| Dashboard | Dashboard | Trang chu | Low |
| Insights | Insights | Tuan nay | Low |
| Chat | Chat | Chat | None |
| Body | Body | Do luong | Low |
| Settings (mobile) | Me | Toi | None |

**Rule:** If any label overflows 2 lines at 72px width, switch to icon-only on viewports <= 360px with `sr-only` span for screen readers. Currently all VI labels are safe.

#### 3.3 Onboarding step labels (4-step progress stepper)

| EN | VI | Risk |
|----|----|------|
| Profile | Thong tin ca nhan | High — 3x longer |
| Activity | Muc hoat dong | Medium |
| Goal | Muc tieu | Low |
| Your targets | Muc tieu cua ban | Medium |

**Recommendation:**
- Mobile (<=767px): Show only current step label + "Buoc N/4" counter. Hide inactive step labels.
- Desktop (>=768px): Show all 4 labels with text wrap allowed (min-width 80px, max-width 120px per step pill).

#### 3.4 Log Meal modal — tab bar (4 tabs)

| EN | VI | Risk |
|----|----|------|
| Photo | Anh | None — shorter |
| Barcode | Barcode | None |
| Voice | Giong noi | Low |
| Manual | Nhap tay | Low |

No overflow risk. No change needed.

#### 3.5 Modal action bar

| EN | VI | Risk |
|----|----|------|
| Cancel | Huy | None — shorter |
| Save | Luu | None — shorter |
| Save meal | Luu bua an | Low |
| Confirm Save | Xac nhan luu | Medium |
| Calculate targets | Tinh muc tieu | Low |

**Rule:** Action bar buttons use `min-width: 80px`, allow 2-line text wrap with 12px vertical padding.

#### 3.6 Settings tabs

| EN | VI | Risk |
|----|----|------|
| Profile & Goals | Ho so & Muc tieu | Medium — 15 chars |
| Plan | Goi dich vu | Low |
| Privacy & Data | Quyen rieng tu & Du lieu | High — 26 chars vs 14 |

**Recommendation for "Quyen rieng tu & Du lieu":**
- Mobile (<=767px): Truncate tab label to "Rieng tu" (full text as `aria-label`), or use lock icon + "Rieng tu".
- Desktop: `white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 160px`.

#### 3.7 Weekly Insights — stat card labels

| EN | VI | Risk |
|----|----|------|
| Days logged | Ngay da ghi | Low |
| On-target days | Ngay dat muc tieu | Medium — 19 chars |
| Avg calories | Calo TB | Low — abbreviation used |
| Trend | Xu huong | Low |

**Recommendation:** Set `min-width: 90px` on each stat card; allow 2-line label wrap.

#### 3.8 Toast / error messages

Toast `max-width: min(360px, 90vw)` — allows multi-line text wrap at 360px. No horizontal overflow.

#### 3.9 Empty state titles

Allow 2-line wrap at `text-h2` (24px) centred full-width — no overflow risk.

---

### 4. Date / Number / Unit Formatting

#### 4.1 JS APIs to use

- **Dates:** `new Intl.DateTimeFormat(locale, options).format(date)` — locale `"vi-VN"` / `"en-US"`.
- **Numbers:** `new Intl.NumberFormat(locale, options).format(number)` — locale `"vi-VN"` / `"en-US"`.
- Built into all modern browsers and Node.js — no additional library needed.

#### 4.2 Date formatting

| Context | en-US | vi-VN | Intl options |
|---------|-------|-------|-------------|
| Dashboard headline | Friday, May 8 | Thu Sau, 8 thang 5 | `{ weekday:'long', month:'long', day:'numeric' }` |
| Short date (history) | May 8, 2026 | 8 thg 5, 2026 | `{ year:'numeric', month:'short', day:'numeric' }` |
| Inline date (meal timestamp) | May 8 | 8 thg 5 | `{ month:'short', day:'numeric' }` |
| Insights week range | May 2 – May 8 | 2 thg 5 – 8 thg 5 | Two calls joined with " – " |

Dashboard ribbon examples:
- en-US: "Today, Friday, May 8"
- vi-VN: "Hom nay, Thu Sau, 8 thang 5"

When `date === todayLocal()`, override the weekday/date portion with `dashboard.today_label` ("Today" / "Hom nay").

#### 4.3 Time formatting

Both locales: **24-hour format** — `"08:30"`, `"14:15"`. No AM/PM.
Use `Intl.DateTimeFormat(locale, { hour:'2-digit', minute:'2-digit', hour12:false })`.
Output is identical in vi-VN and en-US at 24h — consistent.

#### 4.4 Number formatting

| Context | en-US | vi-VN |
|---------|-------|-------|
| Calories (integer) | 2,633 kcal | 2.633 kcal |
| Weight (1 decimal) | 68.5 kg | 68,5 kg |
| Macro grams | 165 g | 165 g |
| Percentage | 82% | 82% |

- `Intl.NumberFormat("vi-VN")` handles thousands (`.`) and decimal (`,`) separators automatically.
- Macro abbreviations in vi-VN to save space: `g dam` / `g tinh bot` / `g beo`. On macro ring sub-labels at mobile use single-letter abbreviations D / T / B (aria-label holds full text).

Example dashboard macro summary:
- en-US: `2,633 kcal · 165g protein · 264g carbs · 88g fat`
- vi-VN: `2.633 kcal · 165g dam · 264g tinh bot · 88g beo`

#### 4.5 Weight unit

Both locales default to **kg**. Unit preference (`profiles.units_preference`) is independent of language preference (`profiles.locale_preference`) — imperial users see lb in both locales. Confirm with Architect that these are separate columns.

---

### 5. Accessibility & SSR

#### 5.1 html lang attribute

- Default (new session / logged-out): `lang="vi"` — matches US-I18N1 AC1.
- After user switches to English: `lang="en"`.
- Root `app/layout.tsx` must read `locale` server-side from the session and set `<html lang={locale}>` before hydration — prevents flash of wrong lang.
- Implementation: `next-intl` middleware passes locale via headers/cookies; DEV reads it in the root layout.

#### 5.2 LanguageSwitcher keyboard navigation

- Left/Right arrow keys move focus between the two radio buttons.
- Space or Enter selects the focused option and triggers the switch flow.
- Tab moves focus out of the radiogroup to the next focusable element.
- Focus-visible ring: `shadow-ring` token (`0 0 0 3px var(--color-primary) / 0.4`).

#### 5.3 Screen reader live region

On successful locale switch, a `role="status"` `aria-live="polite"` region announces:
- "Language changed to English" (when switching to en)
- "Da chuyen sang Tieng Viet" (when switching to vi)

Reuse or create a single live region in the Settings page; DEV's choice of implementation.

#### 5.4 Color independence of active state

Active state indicated by:
1. Filled background (`var(--color-primary)`) vs transparent background — shape distinction.
2. `aria-checked="true"` — programmatic distinction.

Not by color alone — satisfies WCAG 1.4.1 Use of Color.

#### 5.5 Server-side rendering

All authenticated pages (dashboard, history, insights, chat, body, settings) must render in the correct locale from the first server paint — no flash of English before hydration (US-I18N1 AC8). Requires reading `locale_preference` from Supabase session server-side and passing to `<NextIntlClientProvider locale={...}>` in the layout before the page component renders.

---

### 6. Out of Scope (explicit)

- **Do not redesign existing screens.** Only string catalog, switcher component spec, and formatting guidance are new.
- **Do not add new screens** beyond the LanguageSwitcher control within the existing Settings page (Profile & Goals tab).
- **AI vision provider chain** (Gemini to Ollama Cloud to Claude) is Architect's scope. This spec only catalogues the two user-visible error strings for quota-exhausted cases (`log_meal.photo.error.all_providers_exhausted`, `log_meal.photo.error.recognition_failed`).
- **Disclaimer legal text** in Vietnamese is provided as best-effort translation in section 1.8. All disclaimer strings are marked **[needs legal review before launch]** — do NOT publish Vietnamese legal copy without review by a qualified legal professional.
- **Brand name "BiteIQ"** is not translated in any locale.
- **PWA install prompt** — not present in the current build. Out of scope.
- **Voice input language switching** — MVP voice is English-only; Web Speech API language is a separate concern deferred to post-MVP.
- **RTL layout** — Vietnamese is LTR; no RTL changes needed.

---

*Phase 12 section generated by Designer Agent — 2026-05-08. Covers 17 screens, approximately 232 string keys. LanguageSwitcher placed in Settings / Profile & Goals tab.*
