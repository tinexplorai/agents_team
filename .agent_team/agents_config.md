# Agent Configuration

> **Single place to change which model each agent uses.**
> Edit the `Model` column below — agents read this file to know which model to spawn with.

## Models

| Agent | Model | Description File | MCP servers used |
|-------|-------|------------------|------------------|
| Team Lead | `opus` | (your main session — set with `/model opus`) | — |
| PO Agent | `opus` | [agents/PO_agent.md](agents/PO_agent.md) | — |
| Architect Agent | `opus` | [agents/Architect_agent.md](agents/Architect_agent.md) | `supabase` *(if DB)* |
| Designer Agent | `sonnet` | [agents/Designer_agent.md](agents/Designer_agent.md) | `figma` *(optional)* |
| DEV Agent | `sonnet` | [agents/DEV_agent.md](agents/DEV_agent.md) | `supabase` *(if DB)* |
| Flutter Agent | `sonnet` | [agents/Flutter_agent.md](agents/Flutter_agent.md) | `supabase` *(if DB)* |
| QA Agent | `sonnet` | [agents/QA_agent.md](agents/QA_agent.md) | — |
| DevOps Agent | `sonnet` | [agents/DevOps_agent.md](agents/DevOps_agent.md) | `github`, `vercel` |

> The agent description files above are siblings of this file (same `.agent_team/agents/` folder). Project-level inputs the user fills in (project description, resources, design files) live in [`../_input/`](../_input/) — see [`../_input/README.md`](../_input/README.md) for the step-by-step.

## Available models

| Alias | When to use |
|-------|-------------|
| `opus` | Deep reasoning — PO, Architect, Security |
| `sonnet` | Balanced — DEV, QA, DevOps, DBA |
| `haiku` | Fast/cheap — simple lookups, summaries |

> Aliases auto-resolve to the latest version. Pin a specific version (e.g. `claude-opus-4-7`) only if you need reproducibility.

## Provider note

Claude Code only spawns **Anthropic** models via the `Agent` tool. To use ChatGPT or Gemini, you would need a different orchestrator (LangGraph, CrewAI, custom multi-provider setup) — this template does not support that.

## MCP servers

Some agents call external tools via MCP. Configuration lives in [`../.mcp.json`](../.mcp.json) at the project root; secrets live in `.env` (copy from `.env.example`). See [../README.md §1.4](../README.md#14-mcp-configuration) for setup.

## Adding a new agent

1. Add a row to the **Models** table above with the agent name, chosen model, path to its description file, and any MCP servers it uses.
2. Create the description file at `.agent_team/agents/<NAME>_agent.md` (copy an existing one as a template).
3. Reference the new agent in [`../_input/1_project_description.md` §3](../_input/1_project_description.md).
4. If the agent needs a new MCP server, add it to [`../.mcp.json`](../.mcp.json) and document required secrets in `.env.example`.

See [README §6](../README.md#6-scaling--adding-agents) for the catalog of common agent types and recommended models.
