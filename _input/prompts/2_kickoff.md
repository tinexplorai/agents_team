Read _input/1_project_description.md and build the agent team described there.

- Read .agent_team/agents_config.md to get the model for each agent — pass that exact model when spawning.
- For each agent, read its instruction file in .agent_team/agents/ (e.g. agents/PO_agent.md) and use it as the agent's system prompt.
- Read _input/2_resources.md (concrete identifiers — URLs, slugs, project refs). If the file does not exist, copy _input/2_resources.example.md to _input/2_resources.md and tell the user once. Pass `_input/2_resources.md` to every agent that needs it (Architect, Designer, DEV, Flutter, DevOps) so they follow the lookup protocol at the top of that file.
- Enforce the N/A rule from _input/1_project_description.md §5.1: if a value in _input/1_project_description.md or _input/2_resources.md is `N/A`, the responsible agent decides without re-asking the user, documents the decision in their deliverable, and flags it in task_board.md. Surface every N/A-derived decision in the interim/final report.
- Before spawning Designer or DevOps agents: verify the MCP servers they need are available. If a value in _input/2_resources.md is still `[PLACEHOLDER]`, ask the user once and write the answer back to _input/2_resources.md. If they say `N/A`, treat per the rule above.
- Follow the Process section of _input/1_project_description.md in order.
- Create .agent_team/task_board.md as the shared coordination file; agents communicate only through that file.
- Each agent must update the task board's messages table when its phase completes.
- AFTER QA: compile docs/interim_report.md, then STOP and ask the user whether to proceed with DevOps deployment. Do NOT spawn DevOps Agent until the user explicitly says yes.
