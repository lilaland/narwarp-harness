# Seed backlog

The first twenty Narwarp tickets, grouped by the tracks in [design-addendum.md](design-addendum.md#6-getting-started). Each ticket is one branch and one PR. Once track C ships `nw add` and the board, import this file into Narwarp itself so the rest of the build is tracked there.

Format per ticket: track, where it runs, dependencies, then acceptance criteria. Agents may tick a criterion only with evidence attached: a test run, a commit or a PR link.

## Status

The managing session keeps this block current.

- **Done:** NW-1 (GitHub access restored on 2026-10-13)
- **In progress:** NW-2 (the owner is creating the repo), NW-4 (Linear admin request)
- **Next:** NW-7, NW-8, NW-9 in order. Then the API contract is frozen, and work fans out to tracks D, F and the rest of C and E
- **At install time:** NW-3, NW-6, the real Claude wiring in NW-16, and NW-17
- **Blocked:** none

## Track 0 · Unblock (you)

### NW-1 · Restore GitHub access for Claude sessions
- Track: 0 · Runs in: you · Depends on: none
- [x] GitHub reconnected, and the Claude GitHub App installed for `lilaland` (owner, 2026-10-13)
- [x] A Claude session can push to `lilaland/narwarp` (branch `claude/blissful-thompson-2zvpq8` pushed on 2026-10-13)

### NW-2 · Create the harness repo
- Track: 0 · Runs in: you · Depends on: NW-1
- [ ] Private repo created (default name `lilaland/narwarp-harness`), with `main` as the default branch
- [ ] Repo attached to a Claude session that can push to it

### NW-3 · Prepare the Mac Studio
- Track: 0 · Runs in: you · Depends on: none
- [ ] Claude Code via the native installer, logged in with `/login` at the Mac's own screen
- [ ] Ollama, uv, tmux, gh and Tailscale installed; system sleep off; `~/code` and `~/vault` created outside Documents
- [ ] From the laptop over Tailscale SSH, `claude agents --json` and `ollama list` both work
- [x] Unified memory size recorded in the repo: 36 GB, which puts it in the 32 GB model tier (see `AGENTS.md`)
- Note: done at install time, after development

### NW-4 · Ask the Linear admin for the OAuth app
- Track: 0 · Runs in: you · Depends on: none
- [ ] Request sent: install Narwarp as an OAuth app with the app actor, plus a webhook for your teams
- [ ] Decision recorded: OAuth app, or a team-restricted personal API key for now

### NW-5 · Export 30 past tasks for the triage test
- Track: 0 · Runs in: you · Depends on: none
- [ ] 30 real tasks as CSV or markdown: title, description, the project each belonged to, and Work or Life
- [ ] File committed to the harness repo under `eval/triage/`

## Track B · Phase 0 spikes (Mac Studio)

### NW-6 · Phase 0 spike report
- Track: B · Runs in: Mac Studio session, plus you for the reboot · Depends on: NW-3, NW-5
- Note: runs at install time, after development. `INSTALL.md` turns these checks into the install runbook
- [ ] `claude --bg -n t1 -w t1 --settings <hooks file>` fires the HTTP hooks, and only in that session
- [ ] `claude agents --json --all` returns `state`, `status`, `waitingFor`, `sessionId` and `name`
- [ ] A `PermissionRequest` HTTP hook fires in a background session and can allow or deny
- [ ] After a real reboot, `claude respawn <id>` resumes the saved conversation
- [ ] A script can post to a session's `CLAUDE_CODE_MESSAGING_SOCKET` with `crossSessionInbound: "accept"`
- [ ] `claude --cloud "<task>"` returns a session id and URL, and the result is traceable to a branch or PR
- [ ] Triage with `qwen3.5:9b`, and `nimble` side by side, scored on the 30 tasks by mode and project
- [ ] sqlite-vec loads under uv-managed Python on macOS
- [ ] A sample Textual app is usable over SSH from the laptop
- [ ] Every failed check has a written fallback

## Track C · Core (cloud)

### NW-7 · Scaffold the harness repo
- Track: C · Runs in: cloud · Depends on: NW-2
- [ ] A uv project with ruff, pytest and pre-commit; `uv run pytest` passes in GitHub Actions
- [ ] `AGENTS.md` and `CLAUDE.md` link the design docs and state the working rules (one PR per ticket, evidence for criteria)
- [ ] Package `narwarp`, with entry points `narwarpd` and `nw`

### NW-8 · Schema and migrations
- Track: C · Runs in: cloud · Depends on: NW-7
- [ ] Ticket store tables: `project`, `ticket` (seven statuses), `run`, `ticket_event`, `ticket_external_link`, `external_object`, `decomposition`, `goal`
- [ ] Board tables: `principals`, `boards`, `messages`, `recipients`, `claims`, `handoffs`, `approvals`, and FTS5 over messages
- [ ] Tests: migrations up and down, one active exclusive claim per target, and a `result` without evidence is rejected

### NW-9 · Daemon skeleton and API contract
- Track: C · Runs in: cloud · Depends on: NW-8
- [ ] An asyncio daemon that is the only writer, with a versioned JSON API on a unix socket (mode 0600)
- [ ] API contract documented and frozen for tracks D, E and F
- [ ] LaunchAgent plist template with KeepAlive; a restart loses no committed state

### NW-10 · The `nw` CLI
- Track: C · Runs in: cloud · Depends on: NW-9
- [ ] `nw add`, `nw board`, `nw start`, `nw post` and `nw approve` work against a test daemon
- [ ] Output in both JSON and plain text

### NW-11 · MCP server with six tools
- Track: C · Runs in: cloud · Depends on: NW-9
- [ ] `start_session`, `post`, `ask`, `claim`, `handoff_write` and `request_approval`, with schema validation
- [ ] The daemon stamps author and trust tier on every message, and approvals are signed and single-use
- [ ] A fake Claude adapter drives the tools in tests

### NW-12 · Triage pipeline v0
- Track: C · Runs in: cloud · Depends on: NW-8
- [ ] Rules, then embedding similarity, then a local model limited to a fixed-choice JSON schema
- [ ] Below the threshold, the ticket goes to the mode's Inbox; corrections are stored as hints
- [ ] Runs over the 30-task file in CI with a fake model; the real-model score comes from NW-6

## Track D · Vault mirror (cloud)

### NW-13 · Port the redactor and chunker
- Track: D · Runs in: cloud · Depends on: NW-7
- [ ] Redactor: 7 secret patterns plus Luhn and entropy checks, with the 18 tests ported from `app/src/rag/redact.rs`
- [ ] Chunker with the command-prefix rule, with the 11 tests ported from `app/src/rag/chunk.rs`

### NW-14 · Vault projector
- Track: D · Runs in: cloud · Depends on: NW-8, NW-13
- [ ] Tickets and `BOARD.md` rendered with frontmatter and `nw_rev`; golden-file tests
- [ ] Debounced atomic writes, own-write skip by hash, git commits coalesced every 15 minutes
- [ ] Never writes under `notes/`; redactor runs on every file

### NW-15 · Vault ingest
- Track: D · Runs in: cloud · Depends on: NW-14
- [ ] Watcher plus allowlist, with tests for each outcome: applied, request in Needs you, rejected, conflict file
- [ ] Merging edits to separate fields when `nw_rev` is stale; conflict copy otherwise

## Track E · Claude lane, then Tusk (Mac Studio)

### NW-16 · Claude lane adapter
- Track: E · Runs in: cloud first, against a fake supervisor; Mac Studio at install · Depends on: NW-11 (fake), NW-6 (real)
- [ ] A fake supervisor mimics `claude --bg`, `claude agents --json` and HTTP hook delivery as documented in the design, and the adapter passes against it in CI
- [ ] Dispatch with `claude --bg`, a worktree per ticket, and hooks passed per session
- [ ] Hook sink and `claude agents --json` poller keep run state current; respawn on boot
- [ ] First-slice loop works end to end: `nw add` → session → board posts → approve in the TUI

### NW-17 · Tusk v0
- Track: E · Runs in: Mac Studio session, at install time · Depends on: NW-16
- [ ] Conductor workspace with `CLAUDE.md`, the Tusk persona and rules
- [ ] Read tools plus `propose_ticket`, `propose_triage` and `propose_dispatch`; proposals appear in Needs you
- [ ] Fresh session each morning, seeded from the conductor handoff; wakes only for your messages and ritual windows

## Track F · TUI (cloud)

### NW-18 · Textual board v0
- Track: F · Runs in: cloud · Depends on: NW-9 (API contract)
- [ ] Five columns, project rows, a Work/Life toggle and a Needs-you queue, with keys to approve and attach
- [ ] Snapshot tests against sample data; usable over SSH

## Track G · Integrations (after the first-slice exit test)

### NW-19 · Linear sync v0
- Track: G · Runs in: cloud, then Mac Studio · Depends on: NW-4, the exit test
- [ ] Link and track issues, attachment backlinks to the ticket URL, roll-up by workflow-state type
- [ ] Write journal so Narwarp ignores its own echoes; OAuth app if installed, personal key otherwise

### NW-20 · Cloud lane
- Track: G · Runs in: Mac Studio · Depends on: NW-6, NW-16
- [ ] Work tickets can dispatch with `claude --cloud`; run state follows the session URL, branch, PR and checks
- [ ] The 3–5 agent cap counts cloud and local sessions together
