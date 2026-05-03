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
9. **Change Request loop** *(any time after QA — design tweak, new requirement, bug found post-deploy)*. The kickoff prompt for this lives in [`../3. PROMPT_CHANGE_REQUEST.md`](../3.%20PROMPT_CHANGE_REQUEST.md). Team Lead classifies the change and re-runs the **minimum** set of agents — do not re-run the whole pipeline:
   - **Small** (copy, color, spacing, single-component bugfix) → **DEV Agent** → **QA Agent** (regression on affected flows). Skip PO / Architect / Designer.
   - **Medium** (layout change, new component, design tokens) → **Designer Agent** → **DEV / Flutter Agent** → **QA Agent**. Skip PO / Architect.
   - **Large — backend/API** (new endpoint, schema change, business logic) → **PO Agent** (append story) → **Architect Agent** (update contract) → **DEV / Flutter Agent** → **QA Agent**. Skip Designer.
   - **Large — UX/flow** (new screens, changed user journey) → full re-run **PO → Architect → Designer → DEV / Flutter → QA** (treat as v2).

   Rules for the loop:
   - **Append, don't rewrite** — add new sections to `docs/user_stories.md`, `docs/design_spec.md`, `docs/api_contract.md` rather than overwriting, so history is preserved.
   - **Regression scope** — QA must test all flows that share code with the changed component, not only the new behavior.
   - **Task board** — open a new section `## Phase 9 — Change Request: <short title>` in `task_board.md`; agents pass the baton through it as in normal phases.
   - **Change report** — Team Lead writes `docs/change_report_<short-title>.md` after QA passes (what changed, agents run, QA results, files touched).
   - **Redeploy gate** — if `docs/deployment.md` exists (project already shipped), STOP after QA and ask the user before spawning DevOps Agent for redeploy (same human gate as step 6).

Agents communicate via `task_board.md` — never via chat. Each agent updates the task board's messages table when completing its phase.

---

## 5. External resources

Concrete identifiers (URLs, slugs, project refs, bundle IDs, paths) live in **[`resources.md`](resources.md)** — a separate file gitignored by default. Copy [`resources.example.md`](resources.example.md) → `resources.md` and fill it in once for the project. Agents read it before asking you for any value.

Secrets (tokens, API keys) live in `.env` (gitignored, copy from [`../.env.example`](../.env.example)). See [README §1.4](../README.md#14-mcp-configuration) for setup.

Mobile signing keys go in **GitHub Secrets**, not `.env` or `resources.md`. DevOps Agent will list the exact names in `docs/deployment.md`.

## 5.1 How to handle missing values

When filling in `project_description.md` or `resources.md`, you have three options for any field:

| Marker | Meaning | What agents do |
|--------|---------|----------------|
| **A real value** | You know what you want | Use it as-is. |
| **`[PLACEHOLDER]`** *(square brackets, default in template)* | Not filled in yet | Ask you in chat, then write the answer back to the file. |
| **`N/A`** | You explicitly defer the decision | Do NOT ask you again. The responsible agent (PO / Architect / Designer / DEV / Flutter / DevOps) makes a reasonable default decision, documents it in their deliverable, and flags it in `task_board.md`. Team Lead surfaces all `N/A`-derived decisions in the interim/final report so you can override later. |

Use `N/A` when you trust the relevant agent's judgment more than your own (e.g. you know nothing about state management → write `state_management: N/A` and let Flutter Agent pick).

---

## 6. Constraints & Notes

- **Performance:** [e.g. p95 < 200ms / N/A]
- **Security:** [e.g. JWT auth, OWASP Top 10 review / N/A]
- **Scalability:** [e.g. 1k concurrent users / N/A]
- **Compliance:** [e.g. GDPR, HIPAA / N/A]
- [Hard constraints — e.g. must run on Windows, no external SaaS calls, MIT-compatible deps only, etc.]
- [Anything else the team should know but a reader could not infer from the code alone]
