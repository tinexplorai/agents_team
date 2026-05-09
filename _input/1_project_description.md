# Project Description

> **Per-project spec** — the agent team reads this file at kickoff for project-specific context.
> Fill in every `[...]` placeholder below, then start Claude Code and use the short kickoff prompt in [`prompts/2_kickoff.md`](prompts/2_kickoff.md). If your idea is still rough, [`prompts/1_scope.md`](prompts/1_scope.md) lets PO + Architect help flesh it out first. See [`README.md`](README.md) for the full step-by-step.
>
> **Framework-level info** (which agents exist, the Process / phases, external-resource rules, the `[PLACEHOLDER]` / `N/A` rule) lives in [`../.agent_team/workflow.md`](../.agent_team/workflow.md).
> **Model assignments live in [`../.agent_team/agents_config.md`](../.agent_team/agents_config.md)** — edit that file to change which model each agent uses.
> **Per-agent instructions live in [`../.agent_team/agents/`](../.agent_team/agents/)** — edit those files to change agent behavior.

---

## 1. Project Overview

**Name:** BiteIQ

**One-line summary:** AI-powered calorie & macro tracker — log bữa ăn trong 30 giây bằng ảnh, giọng nói, barcode hoặc text.

**Description:**
BiteIQ là web app theo dõi dinh dưỡng dành cho người muốn build thói quen ăn uống lành mạnh mà không tốn thời gian log thủ công. Người dùng chụp ảnh đĩa ăn → AI nhận diện món + ước lượng calo/macro tự động; hoặc quét barcode, nhập giọng nói/text. Dashboard hiển thị tiến độ vs mục tiêu calo & macro theo ngày, weekly insights, và AI chat tư vấn dinh dưỡng. MVP target web; Flutter mobile + publish Google Play / App Store là Phase 2.

**Goals / success criteria:**
- Time-to-first-log < 2 phút từ lúc signup.
- AI photo recognition đạt ≥ 80% accuracy trên top-100 món Việt + Western phổ biến.
- Retention proxy: ≥ 4 log/tuần sau tuần đầu cho active user.
- Page load p95 < 2s; AI vision response p95 < 5s.
- Supabase free tier đủ cho 1k MAU đầu tiên (không vượt quota DB/storage/edge function).

**Out of scope (MVP):**
- Podcast, recipe library kiểu ParrotPal (16k+ recipes) — bỏ.
- HealthKit / Google Fit integration (chỉ làm khi có mobile Phase 2).
- Social feed, friends, share recipes / import recipe từ social link.
- Apple Watch / wearable companion.
- Workout / exercise tracking (chỉ track calo nạp vào — không tính calo đốt).
- Native mobile app (Phase 2 — Flutter, publish cả Google Play + App Store).

---

## 2. Tech Stack

> **Targets** — fill in only what applies. The agent team uses this to decide which agents to spawn.

- **Web:** `yes` — fill in backend + web frontend below.
- **Mobile:** `no` — Phase 2 sẽ làm Flutter (cả iOS + Android), publish Google Play + App Store. Khi qua Phase 2, update lại file này thành `yes — both` rồi chạy change-request prompt.

**Backend** *(web)*:
- **Framework:** Next.js 15 API routes (App Router) — không cần service backend riêng cho MVP. Server actions + route handlers cho mọi mutation/query.
- **Database:** Supabase (Postgres managed) — chọn vì setup nhanh nhất, có sẵn auth/RLS/storage/edge functions, và Supabase MCP đã configured trong repo này (Architect/DEV agent gọi trực tiếp được).
- **Auth:** Supabase Auth — email/password + Google OAuth.
- **Storage:** Supabase Storage — bucket cho food photos (private, signed URL).
- **AI vision:** Claude API (vision) cho nhận diện món ăn từ ảnh + ước lượng portion size/macro. Cache kết quả theo image hash.
- **Food DB lookup:** Open Food Facts API (free, có barcode endpoint) làm primary; fallback sang Claude inference khi không match.
- **Other:** Supabase Edge Functions cho các tác vụ async (weekly insights cron, AI cost metering).

**Web frontend** *(web)*:
- **Framework:** Next.js 15 (App Router) + React 19 + TypeScript strict mode.
- **Styling:** Tailwind CSS + shadcn/ui component library.
- **Camera / barcode:** `@zxing/browser` cho barcode scan qua webcam; `<input type="file" accept="image/*" capture="environment">` cho photo upload trên mobile browser.
- **Voice input:** Web Speech API (browser native) cho voice-to-text log.
- **Client state:** TanStack Query (server state) + Zustand (UI state).
- **Testing:** **Playwright (E2E required)** cho golden path (signup → log meal → see dashboard); Vitest cho unit/component test.

**Mobile (Flutter)** *(Phase 2 — bỏ qua cho MVP)*:
- N/A cho MVP. Khi qua Phase 2 sẽ điền: Min SDK iOS 14 / Android 8 (API 26), Riverpod, distribution qua Play Console internal + TestFlight, test bằng `flutter test` + `integration_test`.

---

## 3. Constraints & Notes

- **Performance:** Page load p95 < 2s; AI vision response p95 < 5s; barcode scan recognize < 1s sau khi camera focus.
- **Security:** Supabase RLS bật mặc định trên mọi table có user data; OWASP Top 10 review trước khi deploy; không log PII; rate limit AI vision endpoint per-user.
- **Scalability:** MVP target 1k MAU trên Supabase free tier; schema thiết kế sao cho scale lên 10k MAU không cần migration phá vỡ.
- **AI cost guardrails:** Cap chi phí Claude vision per-user — free tier 5 photo/ngày, paid tier unlimited (subscription model giống ParrotPal). Cache recognition theo image hash để tránh re-charge cho cùng ảnh.
- **Compliance:** GDPR cho EU users (data export + account deletion endpoint bắt buộc); disclaimer rõ ràng "không thay thế tư vấn y tế / dinh dưỡng chuyên môn" — không tuyên bố medical/clinical claim.
- **Internationalization:** MVP English only, nhưng dùng `next-intl` từ đầu để add Vietnamese sau không phải refactor.
- **Hosting:** Vercel (Next.js) + Supabase — cả hai có free tier đủ cho MVP. CI/CD qua GitHub Actions (DevOps phase).
- **No external SaaS lock-in beyond Supabase + Vercel + Anthropic** — tránh thêm vendor để giữ migration path mở.
