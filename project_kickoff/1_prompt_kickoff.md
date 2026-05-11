Read project_setup/step_1_project.md and build the agent team described there.

- Read agent_team/agents_config.md to get the model for each agent — pass that exact model when spawning.
- For each agent, read its instruction file in agent_team/agents/ (e.g. agents/Agent_01_PO.md) and use it as the agent's system prompt.
- Read .env for concrete identifiers (URLs, slugs, project refs). Pass `.env` values to every agent that needs them (Architect, Designer, DEV, Flutter, DevOps).
- Enforce the N/A rule from agent_team/workflow.md §3.1: if a value in project_setup/step_1_project.md or .env is `N/A`, the responsible agent decides without re-asking the user, documents the decision in their deliverable, and flags it in task_board.md. Surface every N/A-derived decision in the interim/final report.
- Before spawning Designer or DevOps agents: verify the MCP servers they need are available. If a value in .env is still empty or `[PLACEHOLDER]`, ask the user once. If they say `N/A`, treat per the rule above.
- Follow the Process section (§2) of agent_team/workflow.md in order.
- Create agent_team/task_board.md as the shared coordination file; agents communicate only through that file.
- Each agent must update the task board's messages table when its phase completes.
- AFTER QA: compile project_code/documentation/interim_report.md, then STOP and ask the user whether to proceed with DevOps deployment. Do NOT spawn DevOps Agent until the user explicitly says yes.
