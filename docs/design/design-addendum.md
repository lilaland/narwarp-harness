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
