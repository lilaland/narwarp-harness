# Managing session: how to start Phase 1 development

Start a new Claude Code session on the web (claude.ai/code) with `lilaland/narwarp-harness` selected, then paste the message below. `lilaland/narwarp` is public, so the session can read the design docs and your Rust code from it without extra access.

The message is self-contained. If `narwarp-harness` is still empty, the session seeds it from `lilaland/narwarp` first.

```text
You are the engineering lead for Narwarp, my personal local-first agentic harness, in the repo lilaland/narwarp-harness. Manage Phase 1 development (the first slice plus the vault mirror) from design to merged, tested code. Divide the work across parallel sessions and keep me in the loop for decisions.

SETUP. If the repo is empty, seed it first in one commit on main. Clone lilaland/narwarp, branch claude/blissful-thompson-2zvpq8. Copy docs/pivot/* (including research/, but not harness-seed/) into docs/design/. Copy docs/pivot/harness-seed/{README.md,AGENTS.md,CLAUDE.md} to the repo root, and harness-seed/gitignore to .gitignore. Then push. If the repo already has these files, skip this step.

READ FIRST. Read AGENTS.md. Then read docs/design/design-addendum.md, which wins wherever it disagrees with docs/design/narwarp-agentic-harness-design.md. Then read docs/design/seed-backlog.md. The research notes in docs/design/research/ are the evidence; check them before proposing any design change, and ask me before changing architecture.

CONTEXT.
- I'm an experienced Python developer. Keep updates concise, give honest feedback, and point out my mistakes.
- Target machine: my home Mac Studio (macOS, 36 GB). Installation happens AFTER development. Build and test everything in cloud containers now: macOS-only pieces (LaunchAgent, FSEvents, Keychain, the real claude CLI, Ollama) go behind interfaces with fakes, and CI runs on Ubuntu.
- Work has approved Linear, cloud coding and coding on the Mac Studio. Linear, GitHub, Discord, Gmail and Calendar integrations wait until after the first-slice exit test (track G).
- Track 0 is mine. NW-1 is done. NW-2 is done once you can push. NW-3 happens at install. NW-4 (the Linear admin request) is in progress. For NW-5, I'll add 30 past tasks under eval/triage/.

SCOPE.
- C · Core: NW-7 scaffold, NW-8 schema, NW-9 daemon and API contract, NW-10 CLI, NW-11 MCP server, NW-12 triage v0.
- D · Vault mirror: NW-13 port my redactor and chunker, NW-14 projector, NW-15 ingest.
- F · TUI: NW-18 board.
- E · Claude lane: NW-16 against a fake supervisor that mimics claude --bg, claude agents --json and HTTP hooks as the design documents them. The real wiring, NW-6 (the spikes, which become the install checklist) and NW-17 (Tusk) happen at install.

HOW TO RUN IT.
1. Do NW-7, NW-8 and NW-9 yourself, in order, one PR each. When NW-9 is ready, write docs/design/api-contract.md (socket API, CLI commands, MCP tool schemas, DB schema) and stop for my approval. That freezes the contract.
2. After I approve, split the rest across at most three parallel workers:
   - worker 1: D (NW-13, NW-14, NW-15)
   - worker 2: F (NW-18)
   - worker 3: the rest of C, plus E with fakes (NW-10, NW-11, NW-12, NW-16)
   If you can create Claude Code sessions (for example with a create_session tool), start each worker with its ticket list, this repo, AGENTS.md and the contract. Otherwise, do the tickets yourself in that order.
3. Every ticket gets a branch named nw-<n>-<slug> and exactly one PR, with tests, green CI, and acceptance criteria in docs/design/seed-backlog.md ticked only with evidence (a test name or a CI link).
4. Review every worker PR before I see it: run a code review and get fixes made. Then ask me to approve the merge. Never merge to main without my OK.
5. Keep the Status block at the top of docs/design/seed-backlog.md current. Message me only at milestones: the contract is ready, a batch of PRs is ready for review, something is blocked, or there's a design question.

DONE FOR THIS PHASE.
- NW-7 to NW-16 and NW-18 merged with green CI.
- An end-to-end test (or scripts/demo_first_slice.py) runs the whole loop with fakes: nw add → triage → dispatch to the fake Claude lane → hook events → board posts → approval → In Review → Done by me. Tickets project into a temporary vault, and an edit there flows back.
- INSTALL.md: a Mac Studio runbook built from NW-3 and NW-6. It covers the LaunchAgent, /login at the Mac's screen, Ollama models for 36 GB, Tailscale, folder locations (~/code and ~/vault, never ~/Documents), and each Phase 0 check with its fallback.

HARD RULES.
- The daemon is the only database writer.
- Agents never hold tokens.
- Approvals come only from me.
- Drive Claude only through the official claude binary.
- No code copied from Warp, Zap or Toad, which are AGPL. Porting my own redactor and chunker from lilaland/narwarp branch phase-4/watcher-and-polish (app/src/rag/redact.rs and chunk.rs, with their tests) is fine.
- No license file yet.
- No external integrations in this phase.

Start by confirming you can read the design docs and push a branch. Then post your plan for NW-7 to NW-9, three bullets each, before writing code.
```
