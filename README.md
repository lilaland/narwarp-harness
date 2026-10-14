# Narwarp harness

A local-first agentic harness that connects you, Claude Code, local Ollama models and your apps (Linear, GitHub, Discord, Google Calendar, email). Everything is organized around a kanban board split into Work and Life.

**Status:** the design is done and Phase 1 development (the first slice plus the vault mirror) is starting. It can't be installed yet.

| Part | What it is |
|---|---|
| `narwarpd` | The daemon. It is the only writer of the store, and runs sync, the planner, the reconciler and local-model jobs |
| `nw` | The command-line client |
| MCP server | Board, claim and handoff tools for Claude Code and Ollama agents |
| TUI | A Textual board with five columns, project rows, a Work/Life toggle and a Needs-you queue |
| Tusk | The conductor, a Claude background session that proposes and never applies |
| Vault mirror | Markdown views of everything in `~/vault`, editable in Obsidian |

Start with [AGENTS.md](AGENTS.md), then [docs/design/design-addendum.md](docs/design/design-addendum.md). The backlog is in [docs/design/seed-backlog.md](docs/design/seed-backlog.md).
