# Project Description

> **Single source of truth for the agent team kickoff.**
> Fill in every `[...]` placeholder below, then start Claude Code and use the short kickoff prompt in [README §3, Step 5](../README.md#step-5-kick-off-the-agent-team). If your idea is still rough, [Step 4](../README.md#step-4-optional-scope-the-project-with-po--architect) lets PO + Architect help flesh it out first.
>
> The Team Lead and every sub-agent will read this file at the start of the project.
>
> **Model assignments live in [agents_config.md](agents_config.md)** — edit that file to change which model each agent uses.
> **Per-agent instructions live in [agents/](agents/)** — edit those files to change agent behavior.

---

## 1. Project Overview

**Name:** [PROJECT NAME]

**One-line summary:** [What this project is, in one sentence.]

**Description:**
[2–4 sentences describing what you are building, who it is for, and why it matters. The PO Agent will break this down into User Stories — be specific about the user and the problem.]

**Goals / success criteria:**
- [Goal 1 — measurable if possible]
- [Goal 2]
- [Goal 3]

**Out of scope (optional but recommended):**
- [Things the team should NOT build]

---

## 2. Tech Stack

> **Targets** — fill in only what applies. The agent team uses this to decide which agents to spawn.

- **Web:** `[yes / no]` — if yes, fill in backend + web frontend below.
- **Mobile:** `[yes / no — iOS only / Android only / both]` — if yes, fill in Flutter section below.

**Backend** *(if web=yes)*:
- **Framework:** [FastAPI / Express / Go Gin / Django / ...]
- **Database:** [Supabase (Postgres) / SQLite / PostgreSQL / MongoDB / N/A]
- **Other:** [Redis / Celery / WebSocket / ...]

**Web frontend** *(if web=yes)*:
- **Framework:** [React / Vue / Next.js / ...]
- **Testing:** [Playwright (E2E required) + jest / vitest for unit]

**Mobile (Flutter)** *(if mobile=yes)*:
- **Min SDK:** [e.g. iOS 14, Android 7 (API 24)]
- **State management:** [Riverpod / Bloc / Provider — default Riverpod]
- **Distribution:** [TestFlight / Play Console internal / Firebase App Distribution / public stores]
- **Testing:** [`flutter test` for unit/widget + `integration_test` for E2E]

---

## 3. Agent Team

The team and per-agent instructions:

- **Team Lead / Orchestrator** — coordinates progress via the shared task board (your main Claude Code session).
- **PO Agent** — see [agents/PO_agent.md](agents/PO_agent.md).
- **Architect Agent** — see [agents/Architect_agent.md](agents/Architect_agent.md).
- **Designer Agent** — see [agents/Designer_agent.md](agents/Designer_agent.md). *(skip for backend-only projects)*
- **DEV Agent** — see [agents/DEV_agent.md](agents/DEV_agent.md). *(skip if mobile-only — Flutter Agent covers it)*
- **Flutter Agent** — see [agents/Flutter_agent.md](agents/Flutter_agent.md). *(only when project targets mobile — see §2 Tech Stack)*
- **QA Agent** — see [agents/QA_agent.md](agents/QA_agent.md).
- **DevOps Agent** — see [agents/DevOps_agent.md](agents/DevOps_agent.md).

Models for each agent are defined in [agents_config.md](agents_config.md).

> Add or remove agents to fit the project. See [README §6](../README.md#6-scaling--adding-agents) for the catalog of agent types and recommended models.

---

## 4. Process

1. Create `.agent_team/task_board.md` (read this file for project context).
2. **PO Agent** → write `docs/user_stories.md` (reads `docs/po_input/` + `project_description.md`).
3. **Architect Agent** + **Designer Agent** *(in parallel)* → `docs/api_contract.md` (+ `docs/tech_design.md` if non-trivial) and `docs/design_spec.md`. *(Skip Designer for backend-only projects.)*
4. **Implementation phase** — picks one of these paths based on Tech Stack (§2):
   - **Web only:** **DEV Agent** → backend + web frontend + unit tests + E2E tests.
   - **Mobile only:** **Flutter Agent** → mobile app + unit tests + integration tests.
   - **Web + Mobile:** **DEV Agent** first (so the API client and patterns are settled), then **Flutter Agent** consumes the same API contract.
5. **QA Agent** → run all tests **locally** (Playwright for web, `integration_test` for Flutter), review code, write `docs/qa_report.md`.
6. **Team Lead — INTERIM report + deployment gate.** Compile a report covering everything done locally so far (what was built, QA results, known issues), write it to `docs/interim_report.md`, then **stop and ask the user**: *"Ready to push to GitHub and deploy via DevOps Agent? Or stop here / fix something first?"* **Do not spawn DevOps without explicit user approval.**
7. **DevOps Agent** *(only if user approves at step 6)* → push to GitHub, set up CI; deploy to Vercel for web and/or set up GitHub Actions macOS runner + TestFlight/Play Console pipeline for mobile. Write `docs/deployment.md`.
8. **Team Lead — FINAL report.** After DevOps finishes, compile a final report at `docs/final_report.md` covering everything from the interim report **plus** deployment URLs, CI status, GitHub Secrets the user must add manually, and any first-deploy follow-ups (custom domain, store review, etc.).

Agents communicate via `task_board.md` — never via chat. Each agent updates the task board's messages table when completing its phase.

---

## 5. External resources (for MCP-using agents)

The Designer, DevOps, and DB-using agents need these references. Fill in only what applies — leave blank to skip.

**Design (Designer Agent):**
- **Design input folder:** drop PDFs / PNG / JPG into `docs/design_input/` (preferred). See `docs/design_input/README.md`.
- **Figma file** *(optional fallback)*: `[https://www.figma.com/file/<KEY>/<NAME>]`

**Database (Architect + DEV agents — only if using Supabase):**
- **Supabase project ref:** `[<project-ref>]` *(found in Supabase dashboard URL)*
- **Supabase project URL:** `[https://<project-ref>.supabase.co]`

**Source control + Web deploy (DevOps Agent):**
- **GitHub repo:** `[https://github.com/<owner>/<repo>]`
- **Default branch:** `[main]`
- **Vercel team slug:** `[<team-slug>]` *(if web=yes)*
- **Vercel project slug:** `[<project-slug>]` *(if web=yes)*
- **Vercel env vars to set:** `[e.g. DATABASE_URL, JWT_SECRET — names only; provide values in chat]`

**Mobile distribution (DevOps Agent — only if mobile=yes):**
- **Bundle ID:** `[com.example.myapp]`
- **Apple Developer Team ID:** `[<team-id>]` *(if iOS=yes)*
- **iOS distribution channel:** `[TestFlight / App Store]`
- **Google Play package name:** `[com.example.myapp]` *(if Android=yes)*
- **Android distribution channel:** `[Play Console internal track / Firebase App Distribution / Play Store]`
- **GitHub Secrets to add manually** *(DevOps will list these in `docs/deployment.md`)* — signing certs, App Store Connect API key, Play service account JSON, etc.

> Tokens for MCP servers live in `.env` (copy from `.env.example`). See [README §1.4](../README.md#14-mcp-configuration). Mobile signing keys go in **GitHub Secrets**, not `.env`.

---

## 6. Constraints & Notes

- **Performance:** [e.g. p95 < 200ms / N/A]
- **Security:** [e.g. JWT auth, OWASP Top 10 review / N/A]
- **Scalability:** [e.g. 1k concurrent users / N/A]
- **Compliance:** [e.g. GDPR, HIPAA / N/A]
- [Hard constraints — e.g. must run on Windows, no external SaaS calls, MIT-compatible deps only, etc.]
- [Anything else the team should know but a reader could not infer from the code alone]
