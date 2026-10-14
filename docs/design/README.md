# Narwarp design docs

Research and a decided plan for turning Narwarp into a terminal-based agentic harness that connects you, Claude Code, local Ollama models, Linear, GitHub, Discord, Google Calendar and personal email. Prepared 2026-10-13.

- **Readable version (artifact):** https://claude.ai/artifact/CWFTRYmzN5qg8v9qoGup1U (private to its owner)
- **Full report:** [narwarp-agentic-harness-design.md](narwarp-agentic-harness-design.md), with 85 citations
- **Design addendum:** [design-addendum.md](design-addendum.md). It adds Tusk the conductor, the vault mirror, the cloud and Mac Studio work lanes, and the getting-started tracks, and wins where it disagrees with the report.
- **Seed backlog:** [seed-backlog.md](seed-backlog.md), the first 20 tickets with acceptance criteria
- **Managing session:** [managing-session.md](managing-session.md), the message that starts the Claude Code session leading Phase 1 development
- **Research notes:** [research/](research/), eight files with about 82,000 words and 780 sources

## Decision

Freeze this Rust fork at the `phase-4/watcher-and-polish` tip (`b9b9e760`) as the daily terminal, and skip merging Zap. Build the harness as a standalone Python daemon, `narwarpd`. It is the only writer of one SQLite store and exposes an MCP server, an `nw` CLI and a Claude Code hook sink. Claude Code runs as background sessions (`claude --bg`, `claude agents --json`, HTTP hooks, `claude attach` in tmux). Ollama gets small schema-bound jobs: triage, digests, and a quarantined reader for untrusted text. A Textual TUI is the first of several clients. Tusk, a conductor session, is the conversational front door. It proposes triage, plans and dispatches, and the reconciler applies only what you accept. Everything is also mirrored into a git-backed markdown vault at `~/vault`, which you can edit in Obsidian. The fork's designs move to the new repo, but its WarpUI code stays here: the redactor, chunker, vault/RAG schema, vault jobs and the Tusk persona.

## First slice

One supervised loop:

1. A typed prompt becomes an auto-filed ticket.
2. A background Claude Code session works the ticket in its own git worktree.
3. Its status and questions land on the ticket's board through hooks.
4. You answer or approve from the TUI.

Linear, calendar, Discord, email and search come later, in Phases 2–4 of the build plan.

## Research notes

| File | Topic |
|---|---|
| [kanban-agent-harnesses.md](research/kanban-agent-harnesses.md) | Ticket-centric agent tools, data models, Linear sync patterns, recommended SQLite schema |
| [agent-runtime-architectures.md](research/agent-runtime-architectures.md) | How orchestrators launch, isolate, observe and resume agents; candidate architectures |
| [agent-coordination-message-boards.md](research/agent-coordination-message-boards.md) | Message boards, mailboxes, claims, handoffs, digests, trust tiers |
| [life-management-agents.md](research/life-management-agents.md) | Personal agents, accountability evidence, Life mode design, safety |
| [calendar-time-blocking.md](research/calendar-time-blocking.md) | Scheduling products, algorithms, Google Calendar API, scheduler design |
| [app-integrations.md](research/app-integrations.md) | Linear, Discord, email and GitHub APIs, auth, events and MCP servers |
| [claude-code-and-ollama.md](research/claude-code-and-ollama.md) | Claude Code control surfaces and terms, Ollama features, local model tiers |
| [client-tui-and-build-vs-fork.md](research/client-tui-and-build-vs-fork.md) | Upstream health, trial merge, WarpUI effort, porting, TUI frameworks, AGPL |

Facts about upstream activity, product status and model names reflect 2026-10-13 and will drift.
