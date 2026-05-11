# Project Description

> **Per-project spec** — the agent team reads this file at kickoff for project-specific context.
>
> **Framework-level info** (which agents exist, the Process / phases, external-resource rules, the `[PLACEHOLDER]` / `N/A` rule) lives in [`../agent_team/workflow.md`](../agent_team/workflow.md).
> **Model assignments live in [`../agent_team/agents_config.md`](../agent_team/agents_config.md)** — edit that file to change which model each agent uses.
> **Per-agent instructions live in [`../agent_team/agents/`](../agent_team/agents/)** — edit those files to change agent behavior.

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

## 3. Constraints & Notes

- **Performance:** [e.g. p95 < 200ms / N/A]
- **Security:** [e.g. JWT auth, OWASP Top 10 review / N/A]
- **Scalability:** [e.g. 1k concurrent users / N/A]
- **Compliance:** [e.g. GDPR, HIPAA / N/A]
- [Hard constraints — e.g. must run on Windows, no external SaaS calls, MIT-compatible deps only, etc.]
- [Anything else the team should know but a reader could not infer from the code alone]
