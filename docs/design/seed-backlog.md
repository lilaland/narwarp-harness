# Seed backlog

Narwarp's tickets, grouped by track. Section 7 of [design-addendum.md](design-addendum.md#7-terminal-first-phase-1) (2026-10-23) re-scoped Phase 1 to coding tickets, todos and connectors, with the board in the terminal. Deferred and dropped tickets are marked and kept for their history. Each ticket is one branch and one PR. Once track C ships `nw add` and the board, import this file into Narwarp itself so the rest of the build is tracked there.

Format per ticket: track, where it runs, dependencies, then acceptance criteria. Agents may tick a criterion only with evidence attached: a test run, a commit or a PR link.

## Status

The managing session keeps this block current.

- **Done:** NW-1 (GitHub access restored on 2026-10-13), NW-2 (repo created and seeded on 2026-10-14)
- **Next:** NW-7, NW-8, NW-9 in order, in the cloud. In parallel, NW-23 on the owner's Mac. Once the API contract is frozen, work fans out to tracks K, T and the rest of C and E
- **At install time:** NW-3, NW-6, the real Claude wiring in NW-16, and NW-17
- **Deferred:** NW-4, NW-14, NW-15, NW-19, NW-20. **Dropped:** NW-18
- **Blocked:** none

## Track 0 · Unblock (you)

### NW-1 · Restore GitHub access for Claude sessions
- Track: 0 · Runs in: you · Depends on: none
- [x] GitHub reconnected, and the Claude GitHub App installed for `lilaland` (owner, 2026-10-13)
- [x] A Claude session can push to `lilaland/narwarp` (branch `claude/blissful-thompson-2zvpq8` pushed on 2026-10-13)

### NW-2 · Create the harness repo
- Track: 0 · Runs in: you · Depends on: NW-1
- [x] Repo created as `lilaland/narwarp-harness`, with `main` as the default branch (2026-10-14). It is public; make it private in the repo settings if you want that
- [x] Repo attached to a Claude session that can push to it (seed commit `36dc541`)

### NW-3 · Prepare the Mac Studio
- Track: 0 · Runs in: you · Depends on: none
- [ ] Claude Code via the native installer, logged in with `/login` at the Mac's own screen
- [ ] Ollama, uv, tmux, gh and Tailscale installed; system sleep off; `~/code` and `~/vault` created outside Documents
- [ ] From the laptop over Tailscale SSH, `claude agents --json` and `ollama list` both work
- [x] Unified memory size recorded in the repo: 36 GB, which puts it in the 32 GB model tier (see `AGENTS.md`)
- Note: done at install time, after development

### NW-4 · Ask the Linear admin for the OAuth app
- **Deferred (2026-10-23):** Linear returns in a later phase
- Track: 0 · Runs in: you · Depends on: none
- [ ] Request sent: install Narwarp as an OAuth app with the app actor, plus a webhook for your teams
- [ ] Decision recorded: OAuth app, or a team-restricted personal API key for now

### NW-5 · Export 30 past tasks for the triage test
- Track: 0 · Runs in: you · Depends on: none
- [ ] 30 real tasks as CSV or markdown: title, description, the project each belonged to, and whether it was code or a todo
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
- [ ] Triage with `qwen3.5:9b`, and `nimble` side by side, scored on the 30 tasks by project and kind
- [ ] sqlite-vec loads under uv-managed Python on macOS
- [ ] `claude attach <id>` renders and takes input correctly inside a Narwarp pane, and it's recorded whether the Warp plugin's OSC 777 status events reach that pane
- [ ] A worker session with Claude Code's sandbox and the planned permission rules can't read a Keychain item through `security`, connect to the daemon socket directly, or write to the vault
- [ ] `github-mcp-server` runs behind the gateway with its token in the Keychain, and a worker session lists its tools without ever seeing the token
- [ ] The setting that turns off auto-push and draft PRs for background sessions works, and a worker without Git credentials fails cleanly when it tries to push
- [ ] Every failed check has a written fallback

## Track C · Core (cloud)

### NW-7 · Scaffold the harness repo
- Track: C · Runs in: cloud · Depends on: NW-2
- [ ] A uv project with ruff, pytest and pre-commit; `uv run pytest` passes in GitHub Actions
- [ ] `AGENTS.md` and `CLAUDE.md` link the design docs and state the working rules (one PR per ticket, evidence for criteria)
- [ ] Package `narwarp`, with entry points `narwarpd` and `nw`

### NW-8 · Schema and migrations
- Track: C · Runs in: cloud · Depends on: NW-7
- [ ] Ticket store tables: `project`, `ticket` (seven statuses, and a kind of `code` or `todo`), `run`, `ticket_event`, `external_link` (PRs and issues a ticket points to), `decomposition`
- [ ] Board tables: `principals`, `boards`, `messages`, `recipients`, `claims`, `handoffs`, `approvals`, and FTS5 over messages
- [ ] Connector tables: `connector`, `connector_scope` and `tool_call` (the audit log)
- [ ] No mode column and no Linear-specific tables (section 7.6)
- [ ] Tests: migrations up and down, one active exclusive claim per target, and a `result` without evidence is rejected

### NW-9 · Daemon skeleton and API contract
- Track: C · Runs in: cloud · Depends on: NW-8
- [ ] An asyncio daemon that is the only writer, with a versioned JSON-RPC 2.0 API on a unix socket (mode 0600) at `$NARWARP_SOCKET` or `~/.narwarp/narwarpd.sock`, one JSON message per line, opened with a `hello` that negotiates the version
- [ ] `subscribe` pushes `board.changed`, `run.changed`, `needs_you.changed` and `message.posted` events with sequence numbers, and resumes from `since` after a reconnect, or sends `events_lost` so the client takes a fresh snapshot
- [ ] A ticket's displayed status is derived at read time from stored facts by a fixed precedence
- [ ] JSON Schema for every method, result and event, generated from the Python models into `docs/design/api/`, with a test that fails when the committed schema is stale
- [ ] `narwarpd --fake` serves seeded sample data with no Claude and no network
- [ ] API contract documented and frozen for tracks E, K and T
- [ ] LaunchAgent plist template with KeepAlive; a restart loses no committed state

### NW-10 · The `nw` CLI
- Track: C · Runs in: cloud · Depends on: NW-9
- [ ] `nw add`, `nw board`, `nw start`, `nw post`, `nw approve` and `nw done` work against a test daemon. `nw add` takes coding tickets and todos
- [ ] `nw board` prints a readable plain-text board, for use over SSH
- [ ] Output in both JSON and plain text

### NW-11 · MCP server with six tools
- Track: C · Runs in: cloud · Depends on: NW-9
- [ ] `start_session`, `post`, `ask`, `claim`, `handoff_write` and `request_approval`, with schema validation
- [ ] The daemon stamps author and trust tier on every message, and approvals are signed and single-use
- [ ] A fake Claude adapter drives the tools in tests

### NW-12 · Triage pipeline v0
- Track: C · Runs in: cloud · Depends on: NW-8
- [ ] Rules, then embedding similarity, then a local model limited to a fixed-choice JSON schema. Output: a project and a kind (`code` or `todo`)
- [ ] Below the threshold, the ticket goes to the Inbox; corrections are stored as hints
- [ ] Runs over the 30-task file in CI with a fake model; the real-model score comes from NW-6

## Track D · Vault mirror (cloud)

### NW-13 · Port the redactor
- Track: D, kept in Phase 1 because the gateway uses it · Runs in: cloud · Depends on: NW-7
- [ ] Redactor: 7 secret patterns plus Luhn and entropy checks, with the 18 tests ported from `app/src/rag/redact.rs`
- [ ] The chunker (11 tests in `app/src/rag/chunk.rs`) moves to Phase 2 with the vault mirror

### NW-14 · Vault projector
- **Deferred (2026-10-23):** Phase 2
- Track: D · Runs in: cloud · Depends on: NW-8, NW-13
- [ ] Tickets and `BOARD.md` rendered with frontmatter and `nw_rev`; golden-file tests
- [ ] Debounced atomic writes, own-write skip by hash, git commits coalesced every 15 minutes
- [ ] Never writes under `notes/`; redactor runs on every file

### NW-15 · Vault ingest
- **Deferred (2026-10-23):** Phase 2
- Track: D · Runs in: cloud · Depends on: NW-14
- [ ] Watcher plus allowlist, with tests for each outcome: applied, request in Needs you, rejected, conflict file
- [ ] Merging edits to separate fields when `nw_rev` is stale; conflict copy otherwise

## Track E · Claude lane, then Tusk (Mac Studio)

### NW-16 · Claude lane adapter
- Track: E · Runs in: cloud first, against a fake supervisor; Mac Studio at install · Depends on: NW-11 (fake), NW-6 (real)
- [ ] A fake supervisor mimics `claude --bg`, `claude agents --json` and HTTP hook delivery as documented in the design, and the adapter passes against it in CI
- [ ] Dispatch with `claude --bg`, a worktree per ticket, and hooks passed per session
- [ ] Each worker session starts with `--strict-mcp-config`, a generated `--mcp-config` that names only the gateway, a generated `--plugin-dir`, and an environment built from an allowlist
- [ ] Supervision: one prompt in flight per ticket, board messages that arrive mid-turn batched into the next prompt, an idle timeout that resets on output plus an absolute cap, and owner-only cancel handled before any other gate
- [ ] Workers never push. Auto-push and draft PRs on finish are off for worker sessions; the daemon pushes after an approval
- [ ] A `Stop` hook objects while the ticket's criteria are unchecked or a `result` lacks evidence, with a timeout and a cap on objections. The daemon runs the ticket's deterministic checks (tests, lint) before it accepts a `result`
- [ ] Hook status and `claude agents --json` are reconciled with freshness bounds, so a stale "working" expires
- [ ] Hook sink and `claude agents --json` poller keep run state current; respawn on boot
- [ ] First-slice loop works end to end: `nw add` → session → board posts → approve with `nw approve` (and in the terminal once NW-26 lands)

### NW-17 · Tusk v0
- Track: E · Runs in: Mac Studio session, at install time · Depends on: NW-16
- [ ] Conductor workspace with `CLAUDE.md`, the Tusk persona and rules
- [ ] Read tools plus `propose_ticket`, `propose_triage` and `propose_dispatch`; proposals appear in Needs you
- [ ] Fresh session each morning, seeded from the conductor handoff; wakes only for your messages and ritual windows

## Track F · TUI (cloud)

### NW-18 · Textual board v0
- **Dropped (2026-10-23):** replaced by the terminal board (NW-25) and `nw board`
- Track: F · Runs in: cloud · Depends on: NW-9 (API contract)
- [ ] Five columns, project rows, a Work/Life toggle and a Needs-you queue, with keys to approve and attach
- [ ] Snapshot tests against sample data; usable over SSH

## Track G · Integrations (after the first-slice exit test)

### NW-19 · Linear sync v0
- **Deferred (2026-10-23):** returns as a connector plus a sync job
- Track: G · Runs in: cloud, then Mac Studio · Depends on: NW-4, the exit test
- [ ] Link and track issues, attachment backlinks to the ticket URL, roll-up by workflow-state type
- [ ] Write journal so Narwarp ignores its own echoes; OAuth app if installed, personal key otherwise

### NW-20 · Cloud lane
- **Deferred (2026-10-23):** a later phase
- Track: G · Runs in: Mac Studio · Depends on: NW-6, NW-16
- [ ] Work tickets can dispatch with `claude --cloud`; run state follows the session URL, branch, PR and checks
- [ ] The 3–5 agent cap counts cloud and local sessions together

## Track K · Connectors (cloud)

### NW-21 · Connector registry and MCP gateway
- Track: K · Runs in: cloud · Depends on: NW-9, NW-11, NW-13
- [ ] `nw connector add/list/enable/disable`, with a stdio command or an HTTP URL, an auth method (none, a Keychain secret, or OAuth refreshed by the daemon), a scope (everywhere, a project or a ticket) and a tool allowlist with each tool marked read or write
- [ ] The gateway serves the board tools plus the in-scope connectors' tools, prefixed with the connector's name. The daemon runs stdio connectors with their secrets in their own environment and calls HTTP connectors with the token
- [ ] Broker rules: identity, ticket and scope come from the session's connection, never from tool arguments; unknown fields and explicit nulls are rejected; each tool declares the scopes it needs
- [ ] Write tools need a signed approval unless the connector's policy allows them. Every call lands in `tool_call` and on the ticket's board, redacted
- [ ] Tests with a fake connector: a session config contains no secret, an out-of-scope tool is refused, a write without approval is refused, and connector output is stored as data with its provenance

### NW-22 · GitHub connector
- Track: K · Runs in: cloud with a fake; Mac Studio at install · Depends on: NW-21
- [ ] The official `github-mcp-server` behind the gateway, with a fine-grained token from the Keychain interface
- [ ] Read issues, PRs and checks; write comments with approval
- [ ] Pushing a branch and opening a PR stay daemon actions behind an approval, never tools a worker holds

## Track T · Terminal (your Mac, local session, in `lilaland/narwarp`)

Rust work in the fork, run as a local Claude Code session on the owner's Mac. Branches `nw-<n>-<slug>`, PRs into the fork's `main`. Rules: design-addendum section 7.2.

### NW-23 · Fork health and upstream decision
- Track: T · Runs in: your Mac · Depends on: none
- [ ] `phase-4/watcher-and-polish` builds and runs on the Mac. Build time and the exact commands are recorded in the fork's `NARWARP.md`
- [ ] The fork's `main` fast-forwarded to `phase-4/watcher-and-polish`, with the owner's OK, so terminal PRs target `main`
- [ ] `NARWARP_PATCHES.md` lists every hub-file edit Narwarp makes, with the reason
- [ ] The upstream decision in design-addendum section 7.7 is carried out

### NW-24 · `narwarp_client` crate
- Track: T · Runs in: anywhere (pure Rust, no WarpUI) · Depends on: NW-9 (frozen contract)
- [ ] Typed requests and events derived from `docs/design/api/`, over a local or SSH-forwarded unix socket
- [ ] `subscribe` with resume from `since`, a fresh snapshot after `events_lost`, and reconnect with backoff
- [ ] Conformance tests against the contract's fixtures, plus an integration test against `narwarpd --fake`

### NW-25 · Board panel
- Track: T · Runs in: your Mac · Depends on: NW-24
- [ ] Backlog, Todo, In Progress, In Review and Done, with project rows, code and todo tickets, the Needs-you queue and a ticket view with its message log
- [ ] Keyboard first: add (prompt to ticket), open, start, move, search
- [ ] Live updates from the event stream, and snapshot tests against `narwarpd --fake`

### NW-26 · Ticket sessions in panes, and approvals
- Track: T · Runs in: your Mac · Depends on: NW-25, NW-16
- [ ] Opening a ticket runs `claude attach <id>` in a pane or tab named after it, locally or over SSH to the Mac Studio
- [ ] Pane and tab badges from the daemon's run state; a notification when something lands in Needs you
- [ ] Approve or deny from the terminal. The decision goes to the daemon, which signs it; a pending permission request waits on its request ID meanwhile (the cmux Feed pattern)

### NW-27 · The built-in agent on the gateway
- Track: T · Runs in: your Mac · Depends on: NW-21, NW-25
- [ ] The fork's agent (Tusk on Ollama) has the daemon's MCP gateway in its MCP config, as a low-trust principal
- [ ] It can read the board and file proposals, which appear in Needs you; it can't apply anything
