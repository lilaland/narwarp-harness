# Design addendum: Tusk, the vault mirror and work lanes

Decided 2026-10-13. This addendum amends [narwarp-agentic-harness-design.md](narwarp-agentic-harness-design.md). Where they disagree, this file wins.

Three changes since the report:

- **Tusk, the conductor.** A conversational front door that only proposes.
- **The vault mirror.** Everything readable and editable in Obsidian.
- **Work approval (13 October).** Work approved Linear, coding in the cloud, and coding on the home Mac Studio. That adds a cloud lane and a Linear OAuth app.

The Mac Studio is the permanent home of the daemon.

## 1. Tusk, the conductor

A conductor is a long-lived agent you talk to instead of the board. Gas Town and Agent Deck use them. Gas Town also shows that a hierarchy of supervisor agents only pays off around 100 agents, so Narwarp gets exactly one conductor with no authority of its own. **Tusk proposes, and the reconciler applies what you accept.**

| Aspect | Decision |
|---|---|
| Runs as | `claude --bg -n tusk` in its own workspace, whose `CLAUDE.md` holds the Tusk persona (salvaged from the fork) and its rules. It runs on the Mac Studio, on your subscription. |
| Lifecycle | A fresh session every morning at the daily plan, seeded with yesterday's conductor handoff and the overnight digest. The daemon runs `claude respawn` after a reboot. |
| Reads | Board, calendar and memory, through read-only MCP tools. Life data arrives only as locally redacted summaries. |
| Writes | Proposals only: `propose_ticket`, `propose_triage`, `propose_decomposition`, `propose_plan`, `propose_dispatch`, `draft_reply`, plus ritual drafts (daily plan, shutdown, weekly review). |
| Who applies proposals | You, in the Needs-you queue or the plan diff. Low-risk proposals, such as triage above the calibrated threshold or digests, can be auto-accepted by a policy kept in config. |
| Channels | A TUI chat pane; the Narwarp HQ Discord DM, which the bot relays through the daemon into Tusk's inbox socket; your phone through Remote Control. |
| Cost control | The daemon wakes Tusk only for your messages and the three ritual windows. A message to an idle session starts a turn and counts as usage. |
| Guardrails | Tusk can't approve anything, hold credentials, exceed the agent cap, or message anyone but you. Quiet hours are enforced by the daemon, not by the prompt. |

**What it replaces.** System 11, the Life coach, becomes "Conductor (Tusk)". Its rituals move to Tusk unchanged: daily plan, shutdown, weekly review, 12-week review, and no failure counts. There are still twelve systems: three Claude, three local-model and six plain code.

## 2. The vault mirror

SQLite stays the only authority, with `narwarpd` as its only writer. The vault is a projection you can read and edit in Obsidian on the Mac Studio or your phone.

**Location.** `~/vault`, a git repo. Do not use `~/Documents`: the old default was `~/Documents/unwarp-vault`. On an unattended Mac, background Claude sessions and launchd jobs need their own macOS privacy grants for Documents, Desktop and Downloads, and nobody is at the screen to click Allow. Repos go in `~/code` for the same reason.

**Layout:**

| Path | Contents | Written by |
|---|---|---|
| `work/<project>/NW-142.md` | Frontmatter (status, mode, project, priority, due, Linear links, scheduled blocks, `nw_rev`), description, criteria checklist, latest digest, handoff link | Projector |
| `work/<project>/BOARD.md` | Project digest and recent message headers | Projector |
| `life/goals/<slug>.md` | Goal, why, milestones, this week's if-then commitments | Projector |
| `daily/YYYY-MM-DD.md` | The day's plan, shutdown notes, reflection | Projector, with your edits |
| `weekly/YYYY-Www.md` | Weekly review and execution score | Projector, with your edits |
| `notes/` | Your own notes | You only. Indexed, never written, carried over from the old vault layout |
| `.narwarp/` | Mirror state: hashes and last-synced revisions | Projector |

**Writes out.**
- Debounced, by about 2 to 5 seconds.
- Atomic: write to a temporary file, then rename.
- Each file is stamped with the `nw_rev` it was rendered from.
- The daemon records a hash of every file it writes and ignores its own changes.
- Git commits are coalesced every 15 minutes and at the shutdown ritual, never per message.

**Edits in.** A file watcher (FSEvents) feeds an ingest step:

| Edit | Outcome |
|---|---|
| Notes and reflection text, commitment outcomes (kept, partial, missed, renegotiated), priority, due date, moves between Backlog, Todo and Cancelled | Applied directly and journaled |
| Marking a ticket Done; anything that would post to Linear, Discord or a calendar; any edit to a ticket an agent has claimed | Becomes a request in Needs you |
| Machine fields: ids, links, run state, `nw_rev` | Rejected; the file is re-rendered |
| The file's `nw_rev` is older than the database's | Edits to separate fields merge. Otherwise Narwarp writes `NW-142.conflict.md` and opens a Needs-you item |

**Agent access.**
- Agents never get the vault path, and harness sessions deny writes to it in their permission rules.
- Workers get context through hooks. Tusk reads through MCP tools.

**Phone.** Prefer Obsidian Sync, or git through a client app, over iCloud Drive for a folder a daemon writes to.

**Indexing.** System 12 becomes "Memory index and vault mirror". The projector, watcher and FTS5 indexer share one module, and sqlite-vec comes later. The ported redactor runs on every projected file and every embedding.

## 3. Where work runs

All three lanes draw on the same subscription, so the 3–5 agent cap counts them together.

| Lane | How it runs | Best for | Tracking |
|---|---|---|---|
| Mac Studio (default) | `claude --bg` in a worktree under Anthropic's supervisor; attach over SSH in tmux | Hands-on work, local tools or data, fun coding, all Life work | `claude agents --json` and HTTP hooks |
| Cloud | `claude --cloud "<ticket prompt>"` in an Anthropic VM; steer with `claude -p "…" --cloud <id>`; pull local with `--teleport` | Well-specified work tickets, overflow past the local cap, work that should continue while the Mac Studio is busy | Branch, PR and checks on GitHub, plus the session URL. No hooks: cloud sessions can't reach the home daemon |
| Local model | The same `claude` binary pointed at Ollama, with its own `CLAUDE_CONFIG_DIR` | Private or simple tasks; real coding only at 64 GB and up | Same hooks as the Mac Studio lane |

Life work and anything private never leave the Mac Studio. Work tickets default to the Mac Studio and overflow to the cloud.

**Linear.** With work's approval, ask the Linear admin to install Narwarp as an OAuth app with the app actor. Narwarp then:
- writes as itself instead of as you;
- gets agent sessions and delegation;
- can receive webhooks through Tailscale Funnel.

Until the app is installed, use a team-restricted personal API key with one-minute polling.

## 4. Running on the Mac Studio

| Topic | Decision |
|---|---|
| Process | `narwarpd` is a LaunchAgent with KeepAlive in your logged-in user session. The screen can stay locked. Never a LaunchDaemon: Keychain access and Claude's credentials depend on the login session. |
| Boot | The daemon starts Claude's supervisor (it starts on demand), runs `claude respawn` for every in-progress ticket session and for Tusk, then rebuilds state from `claude agents --json` and any hook events it missed. |
| Access | Tailscale SSH from the laptop and phone; the Textual TUI runs over SSH. No inbound ports, unless Tailscale Funnel is added for webhooks. |
| Auth | `/login` at the Mac's own screen, which keeps Remote Control and messaging between machines. Narwarp watches for the 3-day expiry warning and opens a Needs-you item. `claude setup-token` is the fallback, but its sessions can't use Remote Control. |
| macOS | Claude's native installer, so privacy grants survive updates. Repos in `~/code`, vault in `~/vault`, system sleep off. Grant Local Network if agents need LAN hosts. Use `sudo fdesetup authrestart` for planned restarts with FileVault. |
| Permission prompts | Per-worktree permission rules cut most prompts. A `PermissionRequest` HTTP hook answers the rest by policy and can hold up to the 600-second hook timeout while you approve in the TUI. Whether it fires in background sessions is a Phase 0 test. |
| Messages into sessions | Harness sessions set `crossSessionInbound: "accept"`. The daemon learns each session's inbox socket from a `SessionStart` command hook. |

## 5. Build plan changes

| Phase | Change |
|---|---|
| 0 | Add these checks: whether `PermissionRequest` fires in background sessions, a real reboot followed by `claude respawn`, posting to an inbox socket, one `claude --cloud` dispatch tracked to its PR, and Textual over SSH |
| 1 | Add the vault mirror for tickets and boards, built in parallel. Tusk stays out of the first slice |
| 2 | Linear through the OAuth app (personal key until then), the cloud lane, and Tusk with triage, decomposition and dispatch proposals |
| 3 | Tusk runs the daily plan and the shutdown; daily notes land in the vault |
| 4 | Goals and commitments as vault files; Tusk runs the weekly review; the Discord DM relays to Tusk |

## 6. Getting started

Tracks that can run side by side without touching the same files. **At most three build sessions at once**, because review time is the real limit. Tickets are in [seed-backlog.md](seed-backlog.md).

| Track | Work | Runs in | Starts after |
|---|---|---|---|
| 0 · Unblock | GitHub access, the new repo, Mac Studio prep, the Linear admin request, 30 past tasks for the triage test | You | Now |
| A · Design | This addendum, the blueprint update, the seed backlog | Cloud session | Done 2026-10-13 |
| B · Spikes | The Phase 0 checks, ending in a pass-or-fail report with a fallback for each failure | Mac Studio session, plus you for one reboot | Track 0 |
| C · Core | Scaffold, schema, daemon and socket API, `nw` CLI, MCP server, and a fake Claude adapter so tests run anywhere | Cloud session | The repo exists |
| D · Vault mirror | Projector, watcher, allowlist, conflicts, coalesced commits; port the redactor and chunker with their tests; FTS5 | Cloud session | C's schema |
| E · Claude lane, then Tusk | The real adapter, then Tusk's workspace and proposal tools | Mac Studio session | B and C |
| F · TUI | The Textual board against C's API, with sample data and snapshot tests | Cloud session | C's API contract |
| G · Integrations | Linear, GitHub, the cloud lane | Cloud, then Mac Studio | The first-slice exit test |

**Update, 2026-10-13 (later the same day).** The Mac Studio has 36 GB, which puts it in the 32 GB model tier listed in the harness `AGENTS.md`. Installation happens after development, which changes two tracks:
- Track B becomes the install checklist instead of running first.
- Track E builds the Claude lane against a fake supervisor now. The real wiring and Tusk come at install.

Everything else is built and tested in cloud sessions. The development work is led by the managing session described in [managing-session.md](managing-session.md).

**Order:**
1. Days 0–1: tracks 0, A and B.
2. C runs alone until its schema and API contract are frozen.
3. D and F then run in the cloud while E runs on the Mac Studio. That is three sessions, the cap.
4. Once the board works, the rest of the backlog moves into Narwarp itself and the first-slice exit test starts.
5. G starts after the exit test passes.

**Working rules:**
- One branch and one PR per ticket. Each session gets its ticket, the design docs and the new repo's `AGENTS.md`.
- Another session runs a code review before you look.
- You keep the decisions: repo name, auth mode, the Linear admin conversation, and what counts as done.

## 7. Terminal-first Phase 1

Decided 2026-10-23. Where this section disagrees with sections 1 to 6, this section wins.

Three changes:

- **The Narwarp terminal is the main client.** The board, the Needs-you queue and each ticket's Claude Code session live inside the terminal you already work in. The Textual TUI (NW-18) is dropped. `nw board` prints a plain-text board for SSH.
- **Phase 1 is coding, todos and connectors.** Linear, the Work/Life split, the cloud lane, the calendar and the vault mirror wait for a later phase.
- **Connectors are a general mechanism, not one-off integrations.** Any MCP server can be plugged in per project or per ticket, and the daemon keeps every credential.

### 7.1 The daemon stays the brain

`narwarpd` keeps every responsibility from the report: it is the only database writer, supervises Claude sessions, runs the hook sink and the reconciler, and now hosts connectors. The terminal is a client of its API and stores nothing but UI preferences.

Why the board doesn't move into the Rust app:
- Claude sessions, the hook sink and connector token refresh must keep running when the terminal is closed or has crashed.
- The terminal, `nw`, MCP agents and later the phone all read one source of truth.
- The logic stays in Python, so the Rust code is limited to views and a client.
- The fork is AGPL. The daemon is a separate program that talks to it over a socket, which keeps the harness's own code out of the fork.

### 7.2 What the terminal adds

| Piece | What it does |
|---|---|
| `narwarp_client` crate | A typed client for the daemon API over a local unix socket or an SSH-forwarded one. Requests, an event subscription and reconnect with resume |
| Board panel | Backlog, Todo, In Progress, In Review and Done, with project rows, the Needs-you queue and a ticket view with its message log. Keyboard first |
| Session panes | Opening a ticket runs `nw attach <ticket>` in a pane or tab named after it: it records the pane's focus link with the daemon, then runs `claude attach <id>`. Badges come from the daemon's run state (see 7.7) |
| Approvals | Approve or deny from the Needs-you queue. The daemon signs and applies the decision |
| Built-in agent | The fork's agent (Tusk on Ollama) is pointed at the daemon's MCP server through the existing file-based MCP config, so it can read the board and file proposals as a low-trust principal |

**Status comes from the daemon.** Warp's own Claude Code integration installs the MIT-licensed `warpdotdev/claude-code-warp` plugin. Its hooks emit OSC 777 escape sequences carrying JSON events (permission requests, questions, stop), which the terminal parses (`app/src/terminal/cli_agent_sessions/`). Since Claude Code 2.1.141, hooks can hand Claude Code the sequence through a `terminalSequence` output field, so it may reach a pane running `claude attach`; NW-6 checks that. Either way the board's source of truth is the daemon's hooks and `claude agents --json`, and pane badges are a bonus.

**Messages into sessions.** The same repo's `oz-harness-support` plugin is how Warp's Oz feeds a lead agent's messages into a Claude Code child:
- `SessionStart` starts a listener.
- `UserPromptSubmit` and `PostToolUse` drain a mailbox into the session as extra context.
- `Stop` keeps the session going while messages are pending.

It is a tested alternative to cross-session messaging for the daemon's inbox, and it's MIT, so NW-16 can borrow it.

**Rules for the fork**, so it stays cheap to maintain:
- New code goes in new crates or modules. Edits to hub files (`app/src/workspace/view.rs`, `left_panel.rs`, `settings_view/`, the persistence schema) are registrations of a few lines, each listed in `NARWARP_PATCHES.md`.
- No harness state goes in the terminal's own database.
- Harness work goes through `narwarpd`. The terminal never calls Claude or a connector on the harness's behalf.

**Where it's built.** On your Mac, in a local Claude Code session. The app needs a macOS build and a screen to check it, and a cloud container has neither (this session's had 4 cores, 15 GB of RAM, about 29 GB of disk and no display). The `narwarp_client` crate and its tests can run anywhere.

### 7.3 Connectors

The goal is to plug in a tool when a project needs it, without handing the credential to an agent.

| Part | Decision |
|---|---|
| Registry | A daemon table managed with `nw connector add/list/enable/disable`. Each connector has a transport (a stdio command or an HTTP URL), an auth method (none, a secret in the Keychain, or OAuth refreshed by the daemon), a scope (everywhere, a project or a ticket) and a tool policy |
| Tool policy | An allowlist. Each tool is marked read or write. Write tools need a signed approval unless the policy allows them |
| Gateway | The daemon's MCP server is the only MCP server a worker session sees (`--strict-mcp-config` with a generated `--mcp-config`, plus a generated `--plugin-dir` so a third-party mod can't widen tools or approve calls). It serves the board tools plus the tools of the connectors in scope, prefixed with the connector's name. The daemon runs stdio connectors itself, with their secrets in their own environment, and calls HTTP connectors with the token. Nothing secret appears in a session's config, environment or context |
| Broker rules | Taken from Buzz's broker contract. The session's identity, ticket and scope come from its connection, never from tool arguments. Schemas are strict: unknown fields and explicit nulls are rejected. Each tool declares the scopes it needs, as in Arcade's `requires_auth` |
| Child environments | Every process the daemon starts (`claude --bg`, connectors, terminal panes it asks for) gets an environment built from an allowlist, never the daemon's own. Buzz leaked a private key through a denylist |
| Audit | Every call is logged on the ticket's board with the tool, a summary of the arguments and the result size. The redactor runs on what gets stored |
| Untrusted output | Connector results are data, never instructions, and carry their provenance |
| Other clients | The terminal's built-in agent and `nw` use the same gateway |
| First connector | GitHub, through the official `github-mcp-server` with a fine-grained token in the Keychain: issues, PRs and checks to read, comments to write with approval. Pushing branches and opening PRs stays a daemon action behind an approval |

Later integrations (Linear, Calendar, Gmail, Discord) start as connectors. Anything that also needs sync, such as Linear's roll-ups, adds a daemon job on top.

**Known limit.** Workers run as the same macOS user as the daemon, so a determined agent could try the Keychain or the daemon socket directly. Claude Code's sandbox and permission rules (deny the `security` command, the socket path and the vault) narrow that, and NW-6 tests it. A separate macOS user for workers is the stronger option if that test fails.

### 7.4 Todos

Tickets have a kind:
- **code:** has a repo and a worktree, and can be dispatched to a Claude lane.
- **todo:** no agent. A checklist and an optional due date.

Triage files both kinds by project. Models may propose priorities or a split into subtasks, and you apply them, as with every other proposal.

### 7.5 API contract changes for a Rust client (NW-9)

- JSON-RPC 2.0 over the unix socket, one JSON message per line, starting with a `hello` that negotiates the API version.
- `subscribe` pushes `board.changed`, `run.changed`, `needs_you.changed` and `message.posted` events with sequence numbers, and resumes from `since` after a reconnect. If the daemon can't replay that far, it sends `events_lost` and the client takes a fresh snapshot (herdr's socket does this).
- A ticket's displayed status is derived when it's read, from stored facts (run state, approvals pending, PR and checks) by a fixed precedence, rather than stored separately (Orchestrator.inc does this).
- JSON Schema for every method, result and event, generated from the Python models and committed under `docs/design/api/`. The Rust client derives its types from it, and a conformance test runs both sides against the same fixtures.
- The socket is at `$NARWARP_SOCKET` or `~/.narwarp/narwarpd.sock`. From another machine, forward it over SSH (Tailscale SSH to the Mac Studio).
- `narwarpd --fake` serves seeded sample data with no Claude and no network, for client development and snapshot tests.

### 7.6 Deferred

| Deferred | Until | Notes |
|---|---|---|
| Linear (NW-4, NW-19) | A later phase | Comes back as a connector plus a sync job |
| Work/Life split (AGENTS.md rule 8) | Linear's return | One space for now, grouped by project. The mode column and routing come back with work data |
| Cloud lane (NW-20) | A later phase | |
| Vault mirror (NW-14, NW-15) and the chunker | Phase 2 | The redactor stays in Phase 1, because the gateway uses it |
| Textual TUI (NW-18) | Dropped | Replaced by the terminal board and `nw board` |
| Tusk (NW-17) | Install, as before | Its chat surface can later be the terminal's agent panel |

### 7.7 Upstream Warp

Checked 2026-10-23 against `warpdotdev/warp` master `1e6c3ae`. Full notes: [research/warp-upstream-2026-10.md](research/warp-upstream-2026-10.md).

**Decision: stay on the current base for Phase 1, take a small bundle now, and re-fork later.** Warp has 2,298 commits that Zap never took, but most of the new code is cloud and agent code (Oz, "software factories") that a local-first fork would hide. What helps a terminal-first harness is small and local:

| Take now (NW-23) | Why | Cost |
|---|---|---|
| Six link commits: `27f4933b` (focus a pane by link), `207f9d5` (open a saved tab config by link), `a017b9a` (tab-config commands run in order), `679e426d` (`WARP_FOCUS_URL` and the pane's session UUID exported into every pane), `e1be417` (open a file by link), `482f63a` (tab commands from launch-config links) | `nw open` and notifications can open a ticket's session tab and jump back to its pane | 14 conflict hunks, no cloud |
| Two features the fork already has but leaves off: `vertical_tabs_summary_mode` and `git_operations_in_code_review` | Add them to the `default` features | A flag flip each |
| The new CLI-agent events: needs-input (#16232), stop-failure (#13784), pending background work (#16367) | Better pane badges for Claude Code | About 200 lines by hand. The parser ignores unknown fields, so a ticket ID can ride along |
| Optional: MCP OAuth and log fixes, terminal lifecycle recovery (#13788) | Stability | Medium |

**Not useful now:**
- **ACP** runs only inside `warp agent run`, which needs a Warp login and records every run on Warp's server. There is no in-app ACP client yet ([#7326](https://github.com/warpdotdev/warp/issues/7326) is open). It starts Claude through the `claude-code-acp` adapter, which is a rule 9 question for later.
- **Official bring-your-own-key** sends each request through Warp's server and needs an account, so the fork's local Ollama support (from Zap) stays.
- **The Agent Management view, orchestration and `warp_tui`** depend on Warp's server.
- **Tab groups and pinned tabs** would be nice, but they bring 112 conflict hunks, so they wait for a re-fork.

There is no plugin API for custom panels, so the board has to be compiled into the fork. That is why the fork rules in 7.2 matter.

**How panes use the links.** Inside the terminal, the board opens a ticket's pane directly. `nw attach NW-142` is a small wrapper: it reports the pane's `$WARP_FOCUS_URL` to the daemon, then runs `claude attach <id>`. The board, `nw open` and notifications then jump back to the pane through that link.

**Re-fork later.** About 1.5 to 2.5 weeks for a minimal re-fork: lift the login gates, change the app's identity, and port the fork's own layer (21 of its 33 commits replay cleanly). Allow 3 to 5 weeks if the Ollama agent is rebuilt on Warp's new external-harness API. Triggers:
- the board is working and stable;
- Warp ships an in-app ACP client, which could replace Zap's Ollama layer with an ACP agent such as OpenCode;
- an upstream security fix lands that is hard to port.

**The cost of staying.** The base stays about 2,460 commits behind and gets no upstream fixes, security fixes included, unless they're ported. NW-23 sets up a monthly check for them.

### 7.8 Projects to study next

Ranked. Each one answers a question for an upcoming ticket. Full notes: [research/harness-survey-2026-10.md](research/harness-survey-2026-10.md) and [research/buzz-and-goose.md](research/buzz-and-goose.md).

| # | Project | License | Question it answers | For |
|---|---|---|---|---|
| 1 | Warp's `cli_agent_sessions`, Agent Management Panel and `claude-code-warp` | AGPL app, MIT plugin | Can tickets ride the existing CLI-agent event schema with a ticket ID added, and how does the daemon feed unattached sessions into the panel? | NW-25, NW-26 |
| 2 | cmux Feed | GPL, study only | How a permission hook parks on a request ID for up to 120 s while you answer from the sidebar, adapted to signed single-use approvals | NW-26 |
| 3 | herdr | Apache-2.0 | Sequenced state reports, `events.subscribe` and re-snapshot after `events_lost`, as the shape of the terminal-daemon socket | NW-9, NW-24 |
| 4 | Orchestrator.inc | Apache-2.0 | Board columns derived from stored facts by precedence, pushed from a SQLite change log | NW-9, NW-25 |
| 5 | Buzz | Apache-2.0 | The environment fence, the broker contract, per-ticket supervision and bounded Stop objections | NW-16, NW-21 |
| 6 | goose recipes | Apache-2.0 | `retry.checks` and `available_tools` as ticket templates with deterministic checks | NW-16 |
| 7 | OpenAI Symphony SPEC | Apache-2.0 | Claim states, tracker tools on the host, success at a handoff state, never stalling on input | NW-11, NW-16 |
| 8 | kata | MIT | Closing with evidence, attention states, idempotency keys, MCP tools exposed by section | NW-8, NW-11 |
| 9 | MCP 2026-07-28 and arcade-mcp | spec, MIT | Tokens held by the server, per-tool scopes, the owner's OAuth by URL elicitation | NW-21 |
| 10 | ToolHive vMCP, Docker MCP Gateway, or plain `--strict-mcp-config` | Apache-2.0, MIT | Which gives per-ticket tool allowlists with the least machinery | NW-21 |
| 11 | nono and Infisical Agent Vault | Apache-2.0, MIT | Phantom tokens for `git` and `gh`, if the daemon ever needs workers to push | NW-22 |
| 12 | Agent of Empires | MIT | Hook status plus pane checks plus freshness bounds, as the watchdog for stale states | NW-16 |

Don't copy code from GPL or AGPL projects (cmux, Xum, Archestra, Warp's app) into this repo. Apache-2.0 and MIT code can be borrowed with attribution.
