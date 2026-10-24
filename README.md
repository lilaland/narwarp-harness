# Narwarp harness

A local-first agentic harness built into the Narwarp terminal. Coding tickets and todos live on a kanban board, each coding ticket gets its own Claude Code session, local Ollama models triage, and tools plug in as MCP connectors.

**Status:** the design is done and Phase 1 development (coding tickets, todos and connectors, with the board in the terminal) is starting. It can't be installed yet.

| Part | What it is |
|---|---|
| `narwarpd` | The daemon. It is the only writer of the store, and runs Claude Code sessions, the reconciler, connectors and local-model jobs |
| `nw` | The command-line client |
| MCP gateway | Board, claim and handoff tools, plus the tools of whichever connectors a ticket's scope allows. Agents never see the credentials |
| Narwarp terminal | The main client, in [`lilaland/narwarp`](https://github.com/lilaland/narwarp): a board panel with five columns, project rows and a Needs-you queue, and a pane per ticket session |
| Tusk | The conductor, a Claude background session that proposes and never applies |
| Vault mirror | Markdown views in `~/vault`, editable in Obsidian (Phase 2) |

Start with [AGENTS.md](AGENTS.md), then [docs/design/design-addendum.md](docs/design/design-addendum.md). The backlog is in [docs/design/seed-backlog.md](docs/design/seed-backlog.md).
