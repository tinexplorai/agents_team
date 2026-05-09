# `_input/` — Everything you fill in for a new project

> **One folder, one checklist.** Work through the steps below in order. When you finish step 7, the agent team takes over.

Files inside this folder are the **only** things you should edit per project. Framework internals (agent prompts, MCP config, models) live elsewhere and rarely change — see [folder map](#folder-map) at the bottom.

---

## Step-by-step

### ☐ Step 1 — Project description
Open [`1_project_description.md`](1_project_description.md) and fill in every `[...]` placeholder (overview, tech stack, goals, constraints).

- Don't know a value? Write `N/A` — the responsible agent will decide and document the choice.
- Idea still rough? Skip ahead to **Step 6 (scope)** and let PO + Architect propose values for you.

### ☐ Step 2 — Resources (concrete identifiers)
Copy [`2_resources.example.md`](2_resources.example.md) → `2_resources.md` (gitignored), then fill in non-secret IDs:
- Supabase `project_ref` (if using Supabase)
- GitHub repo URL, Vercel slugs (if web)
- `bundle_id`, `apple_team_id`, `android_package` (if mobile)
- Figma file URL (optional, fallback for Designer)

Use `[PLACEHOLDER]` if you don't know yet (agents will ask once); use `N/A` to defer.

### ☐ Step 3 — PO input materials *(optional)*
Drop reference docs into [`3_po_input/`](3_po_input/): PRDs, briefs, customer interviews, competitor notes, support tickets. PO Agent reads everything here before writing user stories.

Empty folder is fine — PO writes from `1_project_description.md` alone and notes assumptions.

### ☐ Step 4 — Design input *(optional, skip for backend-only)*
Drop design files into [`4_design_input/`](4_design_input/): PDFs, PNG, JPG. Number-prefix filenames if order matters (`01_login.png`, `02_dashboard.png`).

Empty folder + Figma URL configured in `2_resources.md` → Designer falls back to Figma MCP. Both empty → Designer writes a best-effort spec from user stories.

### ☐ Step 5 — Secrets (one-time per machine)
At the project root (not in this folder), copy `../.env.example` → `../.env` and paste tokens:
- `FIGMA_API_KEY` (Designer)
- `GITHUB_TOKEN` (DevOps)
- `VERCEL_TOKEN` (DevOps, web only)
- `SUPABASE_ACCESS_TOKEN` + runtime keys (if using Supabase)

`.env` is gitignored. Skip blanks for agents you don't use.

### ☐ Step 6 — *(Optional)* Scope a rough idea with PO + Architect
**When:** `1_project_description.md` is still half-empty / you haven't picked a tech stack yet.
**When to skip:** Step 1 is filled in carefully → go to Step 7.

Start Claude Code (`claude` in this directory), then paste the contents of [`prompts/1_scope.md`](prompts/1_scope.md) into the chat. Fill in your rough idea between the `=== START ===` / `=== END ===` markers. PO + Architect will propose concrete values, ask questions, and update `1_project_description.md` after you answer.

### ☐ Step 7 — Kick off the build
Start Claude Code, then paste the contents of [`prompts/2_kickoff.md`](prompts/2_kickoff.md) into the chat. Team Lead reads `1_project_description.md`, spawns the agent team, and runs the full pipeline through QA. After QA, it stops at the **deployment gate** and asks before pushing/deploying.

### ☐ Step 8 — *(Later)* Change request after QA / deploy
Paste [`prompts/3_change_request.md`](prompts/3_change_request.md) and fill in the `What / Where / Why / Severity / Constraints` form. Team Lead classifies the change (Small / Medium / Large-backend / Large-UX) and re-runs only the agents needed.

---

## Folder map

```
_input/                         ← YOU EDIT FILES HERE
├── README.md                   ← this file (step-by-step)
├── 1_project_description.md    ← project spec (required)
├── 2_resources.example.md      ← template for concrete identifiers
├── 2_resources.md              ← YOU CREATE (gitignored)
├── 3_po_input/                 ← drop PRDs / briefs (PO reads)
├── 4_design_input/             ← drop PDFs / images (Designer reads)
└── prompts/                    ← paste these into Claude Code chat
    ├── 1_scope.md              ← Step 6: refine rough idea
    ├── 2_kickoff.md            ← Step 7: start the build
    └── 3_change_request.md     ← Step 8: change after QA / deploy

../.env                         ← YOU CREATE (secrets, gitignored)
../.env.example                 ← template for ../.env

../.agent_team/                 ← FRAMEWORK (rarely touch)
├── workflow.md                 ← agent roster + Process (phases) + N/A rule
├── agents_config.md            ← which model per agent (optional tweak)
├── agents/                     ← per-agent system prompts
└── task_board.md               ← auto-created at kickoff

../docs/                        ← AGENT OUTPUTS (auto-generated)
├── user_stories.md             ← PO
├── api_contract.md             ← Architect
├── design_spec.md              ← Designer
├── qa_report.md                ← QA
├── interim_report.md           ← Team Lead (gate)
├── deployment.md               ← DevOps
└── final_report.md             ← Team Lead

../.mcp.json                    ← MCP server config (edit Vercel slug once)
../CLAUDE.md                    ← Team Lead instructions (auto-loaded)
../README.md                    ← framework docs
```

## Quick reference

| Marker in any field | Meaning |
|---------------------|---------|
| **A real value** | Used as-is. |
| **`[PLACEHOLDER]`** | Not filled yet — agent asks you once, then writes the answer back. |
| **`N/A`** | You explicitly defer — responsible agent decides and documents in their deliverable. |

See [`../.agent_team/workflow.md` §3.1](../.agent_team/workflow.md) for the full N/A rule.
