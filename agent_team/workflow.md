# Agent Team — Workflow

> **How the agents work together.** This file is framework-level and shared across projects. Per-project content (overview, tech stack, constraints) lives in [`../project_setup/step_1_project.md`](../project_setup/step_1_project.md).
>
> Agents and Team Lead read this file at kickoff to know which roles exist, in what order they run, and how they coordinate.

---

## 1. Agent Team

The team and per-agent instructions:

- **Team Lead / Orchestrator** — coordinates progress via the shared task board (your main Claude Code session).
- **PO Agent** — see [`agents/Agent_01_PO.md`](agents/Agent_01_PO.md).
- **TechLead Agent** — see [`agents/Agent_02_TechLead.md`](agents/Agent_02_TechLead.md).
- **Designer Agent** — see [`agents/Agent_03_Designer.md`](agents/Agent_03_Designer.md). *(skip for backend-only projects)*
- **DEV Agent** — see [`agents/Agent_04_DEV.md`](agents/Agent_04_DEV.md). *(skip if mobile-only — Flutter Agent covers it)*
- **Flutter Agent** — see [`agents/Agent_05_Flutter.md`](agents/Agent_05_Flutter.md). *(only when project targets mobile — see Tech Stack in `../project_setup/step_1_project.md`)*
- **QA Agent** — see [`agents/Agent_06_QA.md`](agents/Agent_06_QA.md).
- **DevOps Agent** — see [`agents/Agent_07_DevOps.md`](agents/Agent_07_DevOps.md).

Models for each agent are defined in [`agents_config.md`](agents_config.md).

> Add or remove agents to fit the project. See [`../README.md` §6](../README.md#6-scaling--adding-agents) for the catalog of agent types and recommended models.

---

## 2. Process

1. Create `agent_team/task_board.md` (read `../project_setup/step_1_project/step_1_project.md` for project context).
2. **PO Agent** → write `project_code/documentation/user_stories.md` (reads `../project_setup/step_2_requirements/` + `../project_setup/step_1_project/step_1_project.md`).
3. **TechLead Agent** + **Designer Agent** *(in parallel)* → `project_code/documentation/api_contract.md` (+ `project_code/documentation/tech_design.md` if non-trivial) and `project_code/documentation/design_spec.md`. *(Skip Designer for backend-only projects.)*
4. **Implementation phase** — picks one of these paths based on Tech Stack (see `../project_setup/step_1_project/step_1_project.md` §2):
   - **Web only:** **DEV Agent** → backend + web frontend + unit tests + E2E tests.
   - **Mobile only:** **Flutter Agent** → mobile app + unit tests + integration tests.
   - **Web + Mobile:** **DEV Agent** first (so the API client and patterns are settled), then **Flutter Agent** consumes the same API contract.
5. **QA Agent** → run all tests **locally** (Playwright for web, `integration_test` for Flutter), review code, write `project_code/documentation/qa_report.md`.
6. **Team Lead — INTERIM report + deployment gate.** Compile a report covering everything done locally so far (what was built, QA results, known issues), write it to `project_code/documentation/interim_report.md`, then **stop and ask the user**: *"Ready to push to GitHub and deploy via DevOps Agent? Or stop here / fix something first?"* **Do not spawn DevOps without explicit user approval.**
7. **DevOps Agent** *(only if user approves at step 6)* → push to GitHub, set up CI; deploy to Vercel for web and/or set up GitHub Actions macOS runner + TestFlight/Play Console pipeline for mobile. Write `project_code/documentation/deployment.md`.
8. **Team Lead — FINAL report.** After DevOps finishes, compile a final report at `project_code/documentation/final_report.md` covering everything from the interim report **plus** deployment URLs, CI status, GitHub Secrets the user must add manually, and any first-deploy follow-ups (custom domain, store review, etc.).
9. **Change Request loop** *(any time after QA — design tweak, new requirement, bug found post-deploy)*. The kickoff prompt for this lives in [`../project_kickoff/2_prompt_change_request.md`](../project_kickoff/2_prompt_change_request.md). Team Lead classifies the change and re-runs the **minimum** set of agents — do not re-run the whole pipeline:
   - **Small** (copy, color, spacing, single-component bugfix) → **DEV Agent** → **QA Agent** (regression on affected flows). Skip PO / Architect / Designer.
   - **Medium** (layout change, new component, design tokens) → **Designer Agent** → **DEV / Flutter Agent** → **QA Agent**. Skip PO / Architect.
   - **Large — backend/API** (new endpoint, schema change, business logic) → **PO Agent** (append story) → **TechLead Agent** (update contract) → **DEV / Flutter Agent** → **QA Agent**. Skip Designer.
   - **Large — UX/flow** (new screens, changed user journey) → full re-run **PO → Architect → Designer → DEV / Flutter → QA** (treat as v2).

   Rules for the loop:
   - **Append, don't rewrite** — add new sections to `project_code/documentation/user_stories.md`, `project_code/documentation/design_spec.md`, `project_code/documentation/api_contract.md` rather than overwriting, so history is preserved.
   - **Regression scope** — QA must test all flows that share code with the changed component, not only the new behavior.
   - **Task board** — open a new section `## Phase 9 — Change Request: <short title>` in `task_board.md`; agents pass the baton through it as in normal phases.
   - **Change report** — Team Lead writes `project_code/documentation/change_report_<short-title>.md` after QA passes (what changed, agents run, QA results, files touched).
   - **Redeploy gate** — if `project_code/documentation/deployment.md` exists (project already shipped), STOP after QA and ask the user before spawning DevOps Agent for redeploy (same human gate as step 6).

Agents communicate via `task_board.md` — never via chat. Each agent updates the task board's messages table when completing its phase.

---

## 3. External resources

Concrete identifiers (URLs, slugs, project refs, bundle IDs, paths) and secrets (tokens, API keys) live in **`.env`** (gitignored, copy from [`../.env.example`](../.env.example)). Agents read it before asking the user for any value. See [`../README.md` §1.4](../README.md#14-mcp-configuration) for setup.

Mobile signing keys go in **GitHub Secrets**, not `.env`. DevOps Agent will list the exact names in `docs/deployment.md`.

## 3.1 How to handle missing values

When filling in `../project_setup/step_1_project.md` or `.env`, you have three options for any field:

| Marker | Meaning | What agents do |
|--------|---------|----------------|
| **A real value** | You know what you want | Use it as-is. |
| **`[PLACEHOLDER]`** *(square brackets, default in template)* | Not filled in yet | Ask you in chat, then write the answer back to the file. |
| **`N/A`** | You explicitly defer the decision | Do NOT ask you again. The responsible agent (PO / Architect / Designer / DEV / Flutter / DevOps) makes a reasonable default decision, documents it in their deliverable, and flags it in `task_board.md`. Team Lead surfaces all `N/A`-derived decisions in the interim/final report so you can override later. |

Use `N/A` when you trust the relevant agent's judgment more than your own (e.g. you know nothing about state management → write `state_management: N/A` and let Flutter Agent pick).
