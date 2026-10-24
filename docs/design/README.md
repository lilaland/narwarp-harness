# Narwarp design docs

Research and a decided plan for turning Narwarp into a terminal-based agentic harness that connects you, Claude Code, local Ollama models and your tools. Prepared 2026-10-13; re-scoped 2026-10-23 (see the update below).

- **Readable version (artifact):** https://claude.ai/artifact/CWFTRYmzN5qg8v9qoGup1U (private to its owner)
- **Full report:** [narwarp-agentic-harness-design.md](narwarp-agentic-harness-design.md), with 85 citations
- **Design addendum:** [design-addendum.md](design-addendum.md). It adds Tusk the conductor, the vault mirror, the cloud and Mac Studio work lanes, the getting-started tracks, and (section 7) the terminal-first Phase 1. It wins where it disagrees with the report.
- **Seed backlog:** [seed-backlog.md](seed-backlog.md), tickets NW-1 to NW-27 with acceptance criteria
- **Managing session:** [managing-session.md](managing-session.md), the message that starts the Claude Code session leading Phase 1 development
- **Research notes:** [research/](research/), the eight original files (about 82,000 words and 780 sources) plus the 2026-10-23 notes

## Decision

Freeze this Rust fork at the `phase-4/watcher-and-polish` tip (`b9b9e760`) as the daily terminal, and skip merging Zap. Build the harness as a standalone Python daemon, `narwarpd`. It is the only writer of one SQLite store and exposes an MCP server, an `nw` CLI and a Claude Code hook sink. Claude Code runs as background sessions (`claude --bg`, `claude agents --json`, HTTP hooks, `claude attach` in tmux). Ollama gets small schema-bound jobs: triage, digests, and a quarantined reader for untrusted text. A Textual TUI is the first of several clients. Tusk, a conductor session, is the conversational front door. It proposes triage, plans and dispatches, and the reconciler applies only what you accept. Everything is also mirrored into a git-backed markdown vault at `~/vault`, which you can edit in Obsidian. The fork's designs move to the new repo, but its WarpUI code stays here: the redactor, chunker, vault/RAG schema, vault jobs and the Tusk persona.

**Update, 2026-10-23.** The terminal is now the main client: the board, the Needs-you queue and each ticket's session move into the Narwarp fork, which talks to `narwarpd` over its API. The Textual TUI is dropped. Phase 1 covers coding tickets, todos and connectors; Linear, the Work/Life split, the cloud lane and the vault mirror come later. Details are in [design-addendum.md section 7](design-addendum.md#7-terminal-first-phase-1).

## First slice

One supervised loop:

1. A typed prompt becomes an auto-filed ticket.
2. A background Claude Code session works the ticket in its own git worktree.
3. Its status and questions land on the ticket's board through hooks.
4. You answer or approve from the terminal.

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
| [harness-survey-2026-10.md](research/harness-survey-2026-10.md) | 2026-10-23: terminal-integrated agent managers, task systems for agents, MCP gateways and credential brokers |
| [buzz-and-goose.md](research/buzz-and-goose.md) | 2026-10-23: Block's Buzz and goose: architecture, supervision, broker contract, recipes |
| [warp-upstream-2026-10.md](research/warp-upstream-2026-10.md) | 2026-10-23: what official Warp shipped since Zap stopped syncing, cherry-pick costs, login gating, a re-fork estimate |

Facts about upstream activity, product status and model names reflect each note's date and will drift.
