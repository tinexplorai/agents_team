# SDLC Agent Team Framework

> A reusable framework for building software with **Claude Code Agent Teams**.
> Each AI agent plays a specialized role (PO, DEV, QA...) and they coordinate through shared files — just like a real dev team.
>
> Works for any project: SaaS, API, mobile backend, e-commerce, chat app, dashboard...

---

## Table of Contents

1. [Setup & Configuration](#1-setup--configuration)
2. [Architecture](#2-architecture)
3. [Step-by-Step Guide](#3-step-by-step-guide)
4. [Prompt Templates](#4-prompt-templates)
5. [Folder Structure](#5-folder-structure)
6. [Scaling & Adding Agents](#6-scaling--adding-agents)
7. [Examples](#7-examples)

---

## 1. Setup & Configuration

### 1.1 Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/project_code/documentation/claude-code) or VS Code Extension
- Git (recommended)
- Runtime for your tech stack (Python, Node.js, Go, etc.)

### 1.2 Enable Agent Teams

**Linux / macOS:**
```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
claude
```

**Windows (PowerShell):**
```powershell
$env:CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS = "1"
claude
```

**Windows (CMD):**
```cmd
set CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
claude
```

### 1.3 Model Configuration

> **Provider:** This template uses Claude Code's native `Agent` tool, which only spawns **Anthropic** models (Opus / Sonnet / Haiku). To use ChatGPT or Gemini, you'd need a different orchestrator (LangGraph, CrewAI, custom multi-provider setup) — that is outside the scope of this template.

#### Team Lead (your main session)
```bash
/model opus          # Orchestrator should use the most capable model
```

#### Per-agent model assignment

Models are configured in **[agent_team/agents_config.md](agent_team/agents_config.md)** — one file, one place to change.

To switch a model: open that file, edit the `Model` column for the agent you want to change, save. Agents read this file when spawned.

#### Recommended assignment

| Agent | Model | Why |
|-------|-------|-----|
| **Team Lead** | `opus` | Complex coordination, architecture decisions |
| **PO Agent** | `opus` | Deep requirement analysis, edge case thinking |
| **TechLead Agent** | `opus` | API contract + tech design, decisions costly to change |
| **Designer Agent** | `sonnet` | Translate design refs → spec; pattern-matching task |
| **DEV Agent** | `sonnet` | Fast code generation, good quality |
| **Flutter Agent** | `sonnet` | Mobile code generation; same reasoning as DEV |
| **QA Agent** | `sonnet` | Test execution, code review patterns |
| **DevOps Agent** | `sonnet` | Push to GitHub, Vercel deploy — formulaic |
| **Security Agent** | `opus` | Vulnerability analysis needs deep reasoning |

> **Available aliases:** `opus` (most capable), `sonnet` (balanced), `haiku` (fastest/cheapest). Aliases auto-resolve to the latest version — pin a specific version (e.g. `claude-opus-4-7`) only if you need reproducibility.
>
> **Rule of thumb:** Use `opus` for agents that *think* (PO, Security, Architect). Use `sonnet` for agents that *execute* (DEV, QA, DevOps).

### 1.4 MCP Configuration

Some agents call external tools via [MCP](https://modelcontextprotocol.io):

| Agent | MCP server | Purpose |
|-------|------------|---------|
| Designer Agent | `figma` *(optional)* | Pull design frames if no `project_setup/step_3_design/` files are provided |
| TechLead Agent | `supabase` *(if DB)* | Inspect / propose schema |
| DEV Agent | `supabase` *(if DB)* | Run migrations, seed data |
| Flutter Agent | `supabase` *(if DB)* | Same — for mobile apps that talk directly to Supabase |
| DevOps Agent | `github` | Push code, create CI workflow, set branch protection |
| DevOps Agent | `vercel` *(web)* | Link project, set env vars, trigger production deploy |

**Setup (one-time per project):**

1. **Create the resources** (you do this manually, *not* the agent):
   - Figma file with your designs.
   - GitHub repo (empty is fine — DevOps will populate it).
   - Vercel project linked to that repo.
2. **Generate tokens:**
   - **Figma:** [Personal Access Token](https://www.figma.com/developers/api#access-tokens) (read-only is enough).
   - **GitHub:** [Fine-scoped PAT](https://github.com/settings/personal-access-tokens) with `repo` + `workflow` scopes.
   - **Vercel:** [Access Token](https://vercel.com/account/tokens).
   - **Supabase** *(only if project uses Supabase):* [Personal Access Token](https://supabase.com/dashboard/account/tokens).
3. **Configure secrets:**
   ```bash
   cp .env.example .env
   # then edit .env and paste in tokens + Supabase runtime keys (URL, anon key, service role key)
   ```
4. **Configure resources:**
   ```bash
   cp .env.example.md .env.md
   # then edit .env.md and fill in non-secret identifiers (project_ref, GitHub repo URL, Vercel slugs, bundle ID, etc.)
   ```
5. **Update [`.mcp.json`](.mcp.json)** — replace `<team-slug>/<project-slug>` in the `vercel` server URL with your actual Vercel team and project slugs.
6. **Restart Claude Code** — `.mcp.json` is loaded at startup, so the servers won't be available until you restart.

> `.env` and `.env.md` are both gitignored. Never commit them. The `.example` files are safe to commit.
>
> **Three-file split:**
> - **`.env`** — secrets only (tokens, API keys, runtime keys like `SUPABASE_ANON_KEY`).
> - **`.env.md`** — non-secret identifiers (URLs, slugs, project refs, bundle IDs, paths). Agents look up values here before asking you.
> - **`project_setup/step_1_project.md`** — per-project spec (goals, tech stack, constraints). Stable after kickoff. Agent roster + Process live in `agent_team/workflow.md`.
>
> **Skip what you don't need:**
> - Backend-only project → leave `FIGMA_API_KEY` blank and skip the Designer Agent.
> - Not deploying yet → leave the GitHub / Vercel tokens blank and skip the DevOps Agent.
> - Not using Supabase → leave `SUPABASE_ACCESS_TOKEN` blank.
> - Mobile-only project → no Vercel needed; you only need GitHub + (later) Apple Developer / Google Play credentials in **GitHub Secrets** (not `.env`) for CI builds.

---

## 2. Architecture

### 2.1 Overview

```
                    ┌──────────────────────┐
                    │   YOU (User)         │
                    │  "Build App X"       │
                    └─────────┬────────────┘
                              │
                    ┌─────────▼────────────┐
                    │  TEAM LEAD (Claude)  │
                    │  model: opus         │
                    │  Reads/writes board  │
                    └─────────┬────────────┘
                              │
                    ┌─────────▼────────────┐
                    │  PO Agent (opus)     │  Phase 1
                    │  + project_setup/step_2_requirements/    │
                    └─────────┬────────────┘
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
              ┌──────────────┐    ┌────────────────┐
              │ Architect    │    │ Designer       │  Phase 2 (parallel)
              │ opus         │    │ sonnet         │
              │ + supabase*  │    │ + 4_design_input │
              └──────┬───────┘    └──────┬─────────┘
                     └─────────┬─────────┘
                               │
                Phase 3 — pick by tech stack:
                               │
       web only ───►  ┌────────▼─────────┐
                      │  DEV (sonnet)    │
                      │  + supabase*     │
                      └────────┬─────────┘
                               │
       mobile only ──►  ┌──────▼─────────┐
                        │ Flutter (sonnet)│
                        │ + supabase*     │
                        └──────┬──────────┘
                               │
       both ──►  DEV first → Flutter (sequential)
                               │
                    ┌──────────▼──────────┐
                    │  QA Agent (sonnet)  │  Phase 4
                    │  Playwright + Flutter
                    │  integration_test   │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Team Lead          │  Phase 5
                    │  → INTERIM report   │
                    │  → ASK USER: deploy?│  ◄── HUMAN GATE
                    └──────────┬──────────┘
                               │ (user approves)
                    ┌──────────▼──────────┐
                    │  DevOps (sonnet)    │  Phase 6/7
                    │  GitHub +           │
                    │  Vercel (web) /     │
                    │  TestFlight, Play   │
                    │  Console (mobile)   │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Team Lead          │  Phase 8
                    │  → FINAL report     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────────────┐
                    │  Phase 9 — Change Request   │  ◄── any time after QA
                    │  (loop, on user request)    │      see project_kickoff/2_prompt_change_request.md
                    │  Team Lead classifies:      │
                    │   Small  → DEV → QA         │
                    │   Medium → Designer → DEV   │
                    │            → QA             │
                    │   Large-API → PO → Arch     │
                    │            → DEV → QA       │
                    │   Large-UX → full re-run    │
                    │  HUMAN GATE before redeploy │
                    └─────────────────────────────┘

  * supabase MCP only when project uses Supabase
```

### 2.2 Agent Roles

| Agent | Role | Input | Output | MCP |
|-------|------|-------|--------|-----|
| **Team Lead** | Orchestrate, spawn agents, compile reports | User request | Final report | — |
| **PO Agent** | Analyze requirements, write User Stories | `project_setup/step_1_project.md` + `project_setup/step_2_requirements/` | `project_code/documentation/user_stories.md` | — |
| **TechLead Agent** | Define API contract + cross-cutting tech design + DB schema | User Stories + tech stack | `project_code/documentation/api_contract.md` (+ `project_code/documentation/tech_design.md`) | `supabase` *(if DB)* |
| **Designer Agent** | Translate design refs + user stories into UI spec | User Stories + `project_setup/step_3_design/` (or Figma) | `project_code/documentation/design_spec.md` (+ `project_code/documentation/design_assets/`) | `figma` *(optional)* |
| **DEV Agent** | Code backend + web frontend + tests | User Stories + API contract + Design spec | Source code in `backend/`, `frontend/` + tests | `supabase` *(if DB)* |
| **Flutter Agent** | Code mobile app + tests; APK locally, IPA via CI | User Stories + API contract + Design spec | Source code in `mobile/` + tests | `supabase` *(if DB)* |
| **QA Agent** | Run tests (Playwright + Flutter integration_test), review code, report bugs | Source code | `project_code/documentation/qa_report.md` | — |
| **DevOps Agent** | Push to GitHub + CI; deploy web to Vercel and/or set up mobile CI for TestFlight/Play Console | Source code + QA pass | `project_code/documentation/deployment.md` | `github`, `vercel` |

### 2.3 Communication

Agents communicate through **shared files**, not conversation:

```
_input/                       ← YOU fill in (project spec, resources, design refs, prompts)
├── 1_project_description.md  ← Project kickoff config (you fill this in)
├── 2_resources.md            ← Concrete identifiers (you create + fill in)
├── 3_po_input/               ← PO reference docs
├── 4_design_input/           ← Design files
└── project_kickoff/                  ← Paste these into Claude Code

agent_team/                  ← FRAMEWORK (rarely edit)
├── workflow.md               ← Agent roster + Process (phases) + N/A rule
├── agents_config.md          ← Model assignments
├── agents/                   ← Per-agent system prompts
├── task_board.md             ← Task tracking + message log (auto-generated)
└── decisions.md              ← (Optional) Architecture Decision Records
```

**Message flow:**
```
Team Lead ──→ ALL:                   "Project kickoff"
PO Agent  ──→ Architect + Designer:  "User stories ready"        (Phase 1 done)
Architect ──→ DEV / Flutter:         "API contract ready"        (Phase 2a done)
Designer  ──→ DEV / Flutter:         "Design spec ready"         (Phase 2b done)
DEV Agent ──→ Flutter / QA:          "Web code complete"         (Phase 3a done)*
Flutter   ──→ QA Agent:              "Mobile code complete"      (Phase 3b done)*
QA Agent  ──→ Team Lead:              "QA complete — see report" (Phase 4 done)
Team Lead ──→ User (HUMAN GATE):     "Interim report ready —     (Phase 5)
                                      deploy? yes / no / fix X"
   ↓ user says "yes, deploy"
Team Lead ──→ DevOps Agent:          "Approved — go ship"        (Phase 6)
DevOps    ──→ Team Lead:             "Shipped — see deployment"  (Phase 7 done)
Team Lead ──→ User:                   Final summary               (Phase 8)
   ↓ user later requests a change (paste project_kickoff/2_prompt_change_request.md)
Team Lead ──→ User:                  "Classified as <S/M/L>,        (Phase 9)
                                       plan: <agents>. Approve?"
   ↓ user approves
Team Lead ──→ <subset of agents>:    Re-run minimum set; QA regress
QA Agent  ──→ Team Lead:             "Regression passed"
Team Lead ──→ User (HUMAN GATE):     "Redeploy? (only if shipped)"

* Phase 3a/3b only when project has both web + mobile. Web-only skips Flutter; mobile-only skips DEV.
```

---

## 3. Step-by-Step Guide

> **Single canonical checklist:** [`project_setup/README.md`](project_setup/README.md). The summary below mirrors it — when in doubt, follow `project_setup/README.md` and the prompt files in [`project_kickoff/`](project_kickoff/).

### Step 1: Copy this template

```bash
# Copy the template folder to your new project
cp -r agentteam_demoproject/ /path/to/my-new-project/
cd /path/to/my-new-project/
```

### Step 2: Fill in the inputs

Everything you fill in per project lives in [`_input/`](_input/). Required:

- **[project_setup/step_1_project.md](project_setup/step_1_project.md)** — replace every `[...]` placeholder (overview, tech stack, constraints).
- **[.env.md](.env.md)** — copy from [.env.example.md](.env.example.md) and fill in concrete identifiers (URLs, slugs, project refs). Use `N/A` to defer (see [agent_team/workflow.md §3.1](agent_team/workflow.md)).

Optional:

- **[project_setup/step_2_requirements/](project_setup/step_2_requirements/)** — drop PRDs / briefs / interviews (PO reads).
- **[project_setup/step_3_design/](project_setup/step_3_design/)** — drop PDFs / images (Designer reads).
- **`.env`** — copy from `.env.example`, fill in tokens (Designer / DevOps / Supabase).
- **[agent_team/agents_config.md](agent_team/agents_config.md)** — change model assignment per agent.
- **[agent_team/agents/](agent_team/agents/)** — edit any `<NAME>_agent.md` to change agent behavior.
- **[.mcp.json](.mcp.json)** — only if Designer or DevOps agent is in scope. See [§1.4](#14-mcp-configuration).

> **Don't have a fully-formed project description yet?** Skip the placeholders and use the scoping prompt at Step 4 — PO + Architect will help flesh it out before any code is written.

### Step 3: Start Claude Code

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
claude
```

### Step 4: (Optional) Scope the project with PO + Architect

**When to use:** `project_setup/step_1_project.md` is rough, no tech stack picked yet, or you want a fresh perspective.
**When to skip:** the file is already filled in carefully — go straight to Step 5.


**What you'll see:**

1. ~30 seconds later, PO and Architect both report back with concrete proposals + 3–5 questions each.
2. You answer the questions in one short reply (bullet points are fine).
3. Team Lead writes the refined values into `project_setup/step_1_project.md` and shows you the diff.
4. You either say *"approve"* (Team Lead runs the kickoff prompt automatically) or *"refine: change X to Y"* to loop again.

**Tip:** mid-scoping you can always change direction — say *"actually drop mobile, web only"* or *"shrink scope to MVP only"* and the loop adjusts. No code has been written yet, so changes are free.

### Step 5: Kick off the agent team

If you skipped Step 4, paste the contents of [`project_kickoff/1_prompt_kickoff.md`](project_kickoff/1_prompt_kickoff.md) into Claude Code — no editing needed.

If you used Step 4, the scoping flow runs this prompt automatically once you say "approve".

The prompt is reusable across projects: only the contents of `_input/` change.

### Step 6: Interact while running

You can intervene at any time during execution:

| What you want | Example prompt |
|---------------|----------------|
| Check progress | `"What's the current status?"` |
| Change requirements | `"Add feature X to the user stories"` |
| Ask a specific agent | `"Have DEV Agent explain the auth flow"` |
| Request a fix | `"QA found a bug, have DEV fix it"` |
| Add a new agent | `"Spawn a Security Agent to do an OWASP review"` |
| See reports | `"Summarize the QA report"` |

### Step 7: Request a change after QA / deploy

When QA has finished (or the project is already deployed) and you want to change something — design tweak, new feature, bug found in production — paste the contents of [`project_kickoff/2_prompt_change_request.md`](project_kickoff/2_prompt_change_request.md) into Claude Code. Fill the `What / Where / Why / Severity / Constraints` form at the bottom.

Team Lead will:
1. Classify the change as **Small / Medium / Large-backend / Large-UX** (rules in [agent_team/workflow.md §2 step 9](agent_team/workflow.md#2-process)).
2. Tell you which agents it plans to spawn and STOP for your approval — so a one-line copy fix doesn't trigger PO + Architect.
3. After you approve, re-run only those agents; QA does regression on shared flows.
4. Write `project_code/documentation/change_report_<title>.md`.
5. If `project_code/documentation/deployment.md` exists, ask before re-spawning DevOps to redeploy.

This is the loop you'll use most often once a project is live — the rest of the README assumes a fresh build.

---

## 4. Agent Instructions

Each agent's instructions live in its own file under `agent_team/agents/`. Edit these files to change agent behavior — Team Lead reads them when spawning each sub-agent.

| Agent | Instruction file |
|-------|------------------|
| PO Agent | [agent_team/agents/Agent_01_PO.md](agent_team/agents/Agent_01_PO.md) |
| TechLead Agent | [agent_team/agents/Agent_02_Architect.md](agent_team/agents/Agent_02_Architect.md) |
| Designer Agent | [agent_team/agents/Agent_03_Designer.md](agent_team/agents/Agent_03_Designer.md) |
| DEV Agent | [agent_team/agents/Agent_04_DEV.md](agent_team/agents/Agent_04_DEV.md) |
| Flutter Agent | [agent_team/agents/Agent_05_Flutter.md](agent_team/agents/Agent_05_Flutter.md) |
| QA Agent | [agent_team/agents/Agent_06_QA.md](agent_team/agents/Agent_06_QA.md) |
| DevOps Agent | [agent_team/agents/Agent_07_DevOps.md](agent_team/agents/Agent_07_DevOps.md) |

To add a new agent: create `agent_team/agents/<NAME>_agent.md`, add a row to [agent_team/agents_config.md](agent_team/agents_config.md), and reference it in [agent_team/workflow.md](agent_team/workflow.md) §1.

---

## 5. Folder Structure

### 5.1 Template (this folder)

```
agentteam_demoproject/
│
├── _input/                           # ← YOU EDIT EVERYTHING HERE per project
│   ├── README.md                     #   Step-by-step checklist
│   ├── 1_project_description.md      #   ← EDIT THIS: project spec (overview, tech stack, process)
│   ├── 2_resources.example.md        #   Template for concrete identifiers — copy to 2_resources.md
│   ├── 2_resources.md                #   ← YOU CREATE: URLs, slugs, project refs (gitignored)
│   ├── 3_po_input/                   #   ← YOU DROP HERE: PRDs, briefs, interviews (PO reads)
│   │   └── README.md
│   ├── 4_design_input/               #   ← YOU DROP HERE: PDFs/images (Designer reads)
│   │   └── README.md
│   └── project_kickoff/                      #   Paste these into Claude Code chat
│       ├── 1_scope.md                #     Step 4: refine rough idea
│       ├── 2_kickoff.md              #     Step 5: start the build
│       └── 3_change_request.md       #     Step 7: change after QA / deploy
│
├── agent_team/                      # Framework internals (rarely edit)
│   ├── workflow.md                   #   Agent roster + Process (phases) + N/A rule
│   ├── agents_config.md              #   ← OPTIONAL: change which model each agent uses
│   ├── agents/                       #   ← OPTIONAL: per-agent system prompts
│   │   ├── Agent_01_PO.md
│   │   ├── Agent_02_Architect.md
│   │   ├── Agent_03_Designer.md
│   │   ├── Agent_04_DEV.md
│   │   ├── Agent_05_Flutter.md
│   │   ├── Agent_06_QA.md
│   │   └── Agent_07_DevOps.md
│   └── task_board.md                 #   ← Auto-generated by Team Lead at kickoff
│
├── CLAUDE.md                         # Team Lead instructions (auto-loaded by Claude Code)
├── .mcp.json                         # MCP server declarations (figma, github, vercel, supabase)
├── .env.example                      # Template for secrets + runtime keys — copy to .env
├── .env                              #   ← YOU CREATE: real secrets (gitignored, never commit)
├── .gitignore
│
├── project_code/documentation/                             # Agent outputs (auto-generated)
│   ├── .gitkeep
│   ├── (user_stories.md)             #   Created by PO Agent
│   ├── (api_contract.md)             #   Created by TechLead Agent
│   ├── (tech_design.md)              #   Created by TechLead Agent (if non-trivial)
│   ├── (design_spec.md)              #   Created by Designer Agent
│   ├── (design_assets/)              #   Created by Designer Agent (Figma exports only)
│   ├── (qa_report.md)                #   Created by QA Agent
│   ├── (interim_report.md)           #   Created by Team Lead after QA (gate to ask: deploy?)
│   ├── (deployment.md)               #   Created by DevOps Agent
│   └── (final_report.md)             #   Created by Team Lead after DevOps
│
├── backend/                          # Server code (auto-generated; web only)
│   ├── .gitkeep
│   ├── (app/ or src/)                #   Created by DEV Agent
│   └── (tests/)                      #   Created by DEV Agent
│
├── frontend/                         # Web client code (auto-generated; web only)
│   ├── .gitkeep
│   ├── (src/)                        #   Created by DEV Agent
│   └── (tests/)                      #   Created by DEV Agent (Playwright)
│
├── (mobile/)                         # Flutter app (auto-generated; mobile only)
│   ├── (pubspec.yaml)                #   Created by Flutter Agent
│   ├── (lib/)                        #   Created by Flutter Agent
│   ├── (test/)                       #   Unit + widget tests
│   └── (integration_test/)           #   E2E tests
│
├── (.github/workflows/)              # CI workflows — Created by DevOps Agent
│   ├── (ci.yml)                      #   Web CI (Ubuntu)
│   └── (mobile-ci.yml)               #   Mobile CI (macOS for iOS, Ubuntu for Android)
│
└── README.md                         # This file
```

> Files in `(parentheses)` are created by agents during execution. You must edit **`project_setup/step_1_project.md`** + **`.env.md`** (copy from `.env.example.md`) for every project, and **`.env` + `.mcp.json`** once per machine if you use Designer or DevOps agents.

### 5.2 Adapt by tech stack

**Python + FastAPI + React:**
```
backend/
├── requirements.txt
├── app/
│   ├── main.py, models.py, schemas.py, database.py, routes.py
└── tests/test_api.py

frontend/
├── package.json, vite.config.js, playwright.config.ts
├── src/main.jsx, App.jsx, App.css, api.js
└── tests/app.spec.ts
```

**Node.js + Express + Vue:**
```
backend/
├── package.json
├── src/index.js, routes/, models/, middleware/
└── tests/api.test.js

frontend/
├── package.json, vite.config.js, playwright.config.ts
├── src/main.js, App.vue, api.js, components/
└── tests/app.spec.ts
```

**Go + Gin + Next.js:**
```
backend/
├── go.mod
├── cmd/server/main.go
├── internal/handlers/, models/, database/
└── tests/api_test.go

frontend/
├── package.json, next.config.js, playwright.config.ts
├── app/page.tsx, layout.tsx, api.ts
└── tests/app.spec.ts
```

**Backend-only (Django):**
```
backend/
├── requirements.txt
├── manage.py
├── app/models.py, views.py, serializers.py, urls.py
└── tests/test_api.py

(no frontend/ folder)
```

---

## 6. Scaling & Adding Agents

### 6.1 Add New Agents On-the-fly

Spawn any new agent at any time during the project:

```
Spawn a [AGENT NAME] to [TASK DESCRIPTION].
Use model: [opus/sonnet/haiku].
Read context from agent_team/task_board.md and project_code/documentation/.
When done, update task_board.md.
```

**Available agent types** (the default flow already includes PO, Architect, Designer, DEV, QA, DevOps — the table below covers add-ons):

| Agent | When to use | Suggested model |
|-------|-------------|-----------------|
| **Security Agent** | Production app, compliance, auth-heavy | opus |
| **DBA Agent** | Complex DB, migrations, query tuning | sonnet |
| **Tech Writer Agent** | Need API docs, user guide, READMEs | sonnet |
| **Perf Agent** | App is slow, need benchmarks | sonnet |
| **Refactor Agent** | Tech debt cleanup | sonnet |

### 6.2 Add New Phases

Extend the workflow by adding phases to the task board:

```markdown
### Phase 6: Security Review (Security Agent)
- [ ] OWASP Top 10 scan
- [ ] Auth/authz review
- [ ] Secrets management check

### Phase 7: Performance (Perf Agent)
- [ ] Benchmark critical endpoints
- [ ] Load test under expected concurrency
- [ ] Identify and document bottlenecks
```

Prompt to add:
```
Add Phase 6: Security Review to the project.
Spawn a Security Agent (model: opus) to run an OWASP Top 10 review and check auth/secrets handling.
Update task_board.md with the new phase.
```

### 6.3 Handle New Features on Existing Project

**Add a new feature (full cycle):**
```
Add [FEATURE DESCRIPTION] to the existing project.
1. PO Agent (model: opus): add new User Stories to project_code/documentation/user_stories.md
2. TechLead Agent (model: opus): update project_code/documentation/api_contract.md for the new endpoints
3. DEV Agent (model: sonnet): implement the feature, add tests
4. QA Agent (model: sonnet): run all tests (regression + new), update report
```

**Fix a bug (targeted):**
```
QA found a bug: [BUG DESCRIPTION].
Have DEV Agent (model: sonnet) fix it, then QA Agent verify.
```

**Refactor (no new features):**
```
Spawn a Refactor Agent (model: sonnet) to:
- Review code quality in backend/
- Refactor and improve
- Run all tests to ensure no regressions
```

### 6.4 Scaling DEV — Multiple DEV Agents in Parallel

The default flow uses one DEV Agent. For large projects (~50+ user stories or genuinely independent feature sets), you can split implementation across multiple DEV agents running in parallel. This is **not** the default — pull this pattern only when you actually need it. Adding multi-DEV to a small project costs more tokens, surfaces integration bugs, and rarely speeds anything up because Architect's planning overhead grows.

**When it pays off:**
- The product has clear, independent features (e.g. `auth`, `products`, `orders`, `notifications`) where each owns its own files and routes.
- A single DEV would push the context window or take many turns.
- Microservices or modular monoliths where modules don't share much code.

**When it doesn't:**
- A small CRUD app — just use one DEV.
- Features with heavy shared utilities, shared models, or shared state — the integration friction outweighs the parallelism.
- Backend-only with one resource type — overkill.

**Pattern: vertical slicing by feature**

Each DEV owns a feature end-to-end: its routes, its models, its tests. Two DEVs never write to the same file.

1. **At Phase 2**, ask Architect to add an *Implementation Plan* section to `project_code/documentation/tech_design.md`:

   ```
   TechLead Agent: in addition to the API contract, write an Implementation Plan in
   project_code/documentation/tech_design.md that splits work into N independent slices. For each slice list:
     - Slice name (e.g. "auth", "products")
     - User stories it covers (US-N references)
     - API endpoints it owns
     - File paths it is allowed to write to (so slices don't overlap)
     - Shared utilities it may READ but not modify
   ```

2. **At Phase 3**, ask Team Lead to spawn one DEV per slice in parallel:

   ```
   Spawn one DEV Agent (model: sonnet) per slice in project_code/documentation/tech_design.md's Implementation Plan.
   Run them in parallel. Each DEV gets:
     - Its slice name, user stories, endpoints, allowed file paths
     - Read access to project_code/documentation/ and to other slices' files (read-only)
     - The full api_contract.md and design_spec.md
   Each DEV updates task_board.md with its slice ID when complete.
   ```

3. **QA still runs once** at Phase 4 against the combined output. Add an extra acceptance check: cross-slice integration tests (does auth + products work together?).

**Rules of thumb:**

| Project size | DEV count |
|--------------|-----------|
| < 30 user stories | 1 (default) |
| 30–80 stories, clean feature boundaries | 2–3 |
| 80+ stories, microservices | 4+ — also consider a DEV Lead agent |

**Cost vs. benefit:** N parallel DEVs cost ~Nx tokens for ~Nx wallclock speedup *if* the split is clean. If slices overlap, you pay Nx for less than Nx benefit and often more bugs. Spend an extra Architect turn on a clean split before parallelizing.

**For mobile (Flutter Agent):** the same pattern applies if the mobile app is large, but in practice mobile apps split less cleanly than backends. Default is one Flutter Agent.

### 6.5 Run Agents in Parallel

Some agents can run simultaneously if they don't depend on each other:

```
                    Team Lead
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
          PO Agent  Design    DBA Agent        ← Phase 1 (parallel)
          (opus)    Agent     (sonnet)
              │     (sonnet)     │
              └────────┬────────┘
                       ▼
                   DEV Agent (sonnet)          ← Phase 2 (needs Phase 1)
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
          QA Agent  Security  Perf Agent       ← Phase 3 (parallel)
          (sonnet)  Agent     (sonnet)
              │     (opus)       │
              └────────┬────────┘
                       ▼
                  DevOps Agent (sonnet)        ← Phase 4
```

Prompt for parallel execution:
```
Run these 2 agents in parallel:
1. PO Agent (model: opus): write user stories
2. UI/UX Agent (model: sonnet): analyze screenshots and prepare component specs

When both complete, run these 2 in parallel:
3. TechLead Agent (model: opus): write API contract from user stories
4. DBA Agent (model: sonnet): design database schema + migrations

After all 4 complete, spawn DEV Agent with their combined output.
All agents update task_board.md when done.
```

---

## 7. Examples

### 7.1 Todo App

```
Build an Agent Team to develop a Todo App with FastAPI + React.
Features: CRUD todos, filter by status, search by keyword, priority levels.
```

### 7.2 E-commerce Platform

```
Build an Agent Team to develop an E-commerce API with Express + Next.js.

Features:
- User registration & login (JWT auth)
- Product catalog with categories and search
- Shopping cart (add/remove/update quantity)
- Checkout flow with order creation
- Order history and status tracking

Tech Stack:
- Backend: Express.js, PostgreSQL, Prisma ORM
- Frontend: Next.js 14 (App Router), TailwindCSS
- Testing: jest, Playwright

Agent Team:
- PO Agent (model: opus): user stories
- TechLead Agent (model: opus): API contract + tech design
- DEV Agent (model: sonnet): fullstack code + tests
- QA Agent (model: sonnet): API + E2E tests + security review
```

### 7.3 Real-time Chat App

```
Build an Agent Team to develop a Real-time Chat App with Go + React.

Features:
- User auth (register/login)
- Create/join chat rooms
- Real-time messaging (WebSocket)
- Message history with pagination
- Online status indicator

Tech Stack:
- Backend: Go (Gin), WebSocket (gorilla/websocket), PostgreSQL
- Frontend: React, TailwindCSS
- Testing: go test, Playwright

Agent Team:
- PO Agent (model: opus)
- TechLead Agent (model: opus)
- DEV Agent (model: sonnet)
- QA Agent (model: sonnet)
```

### 7.4 Backend-only API

```
Build an Agent Team to develop an Inventory Management API with Django.

Features:
- Products CRUD with SKU
- Warehouse management
- Stock in/out tracking
- Low stock alerts
- Daily/weekly reports

Tech Stack:
- Backend: Django REST Framework, PostgreSQL, Celery
- Testing: pytest-django
- No frontend (API only, use Django Admin)

Agent Team (adjusted):
- PO Agent (model: opus): User Stories
- TechLead Agent (model: opus): API contract + DB schema
- DEV Agent (model: sonnet): backend + admin panel + unit tests
- QA Agent (model: sonnet): API tests + load tests
```

### 7.5 Mobile Backend

```
Build an Agent Team to develop a Fitness Tracker API with FastAPI.

Features:
- User auth (OAuth2 + JWT)
- Log workouts (type, duration, calories)
- Track daily goals and streaks
- Leaderboard (friends)
- Push notification triggers

Tech Stack:
- Backend: FastAPI, PostgreSQL, Redis (caching)
- Testing: pytest, httpx
- No frontend (mobile app will consume the API)

Agent Team:
- PO Agent (model: opus)
- TechLead Agent (model: opus)
- DEV Agent (model: sonnet)
- QA Agent (model: sonnet)
- Security Agent (model: opus): auth flow review, rate limiting
```

---

## Quick Reference

### Commands during a session

| Command | What it does |
|---------|-------------|
| `/model opus` | Switch your session to Opus |
| `/model sonnet` | Switch your session to Sonnet |
| `/status` | Check current model and usage |
| `"Status?"` | Ask Team Lead for project progress |
| `"Add Phase 6: Security Review"` | Extend the workflow |
| `"Spawn Security Agent"` | Add a new agent on-the-fly |

### Tips & Best Practices

| Tip | Details |
|-----|---------|
| **Communicate via files** | Always use `agent_team/task_board.md` as the single source of truth |
| **Sequential phases** | Phase N output is Phase N+1 input — this ensures quality |
| **Self-contained prompts** | Each agent prompt must include all context it needs |
| **Screenshots for UI** | Paste Figma/design screenshots into chat for pixel-perfect UI |
| **Git after each phase** | Commit after each phase completes for easy rollback |
| **Model = cost control** | Use opus only where deep reasoning matters; sonnet for everything else |
| **No agent limit** | Spawn as many agents as you need — just give each a clear mission |
| **Verify Python version** | Python < 3.9 needs Pydantic v1, `List[]` instead of `list[]` |
