# Agent runtime and process architectures of multi-session coding-agent harnesses (2025 – Oct 2026)

Scope: how harnesses that run many AI coding-agent sessions (mainly Claude Code) launch, drive, isolate, observe and resume those agents, and which architectures suit Narwarp, a single-user, local-first macOS terminal harness that drives Claude Code plus local Ollama-backed agents. Other researchers cover kanban/ticket data models, message-board and handoff protocols, life-management agents, calendars, app-integration APIs, Claude Code SDK/hooks/Channels features and terms, Ollama model choice, and TUI frameworks. Those topics appear here only where they shape the runtime architecture.

Research date: 2026-10-13. Status was checked against primary artifacts fetched that day:
- READMEs and docs on `raw.githubusercontent.com` (main branch).
- Release timestamps from the Go module proxy, the npm registry and PyPI.
- Vendor docs (code.claude.com, learn.chatgpt.com, opencode.ai, conductor.build, docs.superset.sh).

"STALE?" marks information more than about 6 months old (before about mid-April 2026). "Practitioner opinion" marks non-vendor commentary.

---

## Q1. What is each harness's process model, stack, state store and current status (as of 2026-10-13)?

### Takeaway
By October 2026 the field has converged on a few designs:
- **Separate the agent process host from the UI.** A daemon, a tmux server or a first-party supervisor owns the PTY or agent process, and UIs attach to it.
- **One git worktree per task.**
- **Hooks or structured protocols for status, in place of screen scraping.** The protocols are Codex app-server JSON-RPC, OpenCode HTTP+SSE, ACP, and Claude's stream-json control protocol.

Claude Code now ships first-party pieces of this stack: a background-session supervisor (agent view, May 2026), per-session inbox sockets (Aug 2026), agent teams (Feb 2026), and Remote Control (Feb 2026). A Narwarp harness can build on these instead of re-implementing them. Several third-party projects shut down, were deprecated or pivoted in H1 2026: Terragon, Crystal, Vibe Kanban, Omnara and Happy.

### Cited Findings

#### A. tmux-substrate session managers (tmux server owns every PTY)

**Claude Squad (smtg-ai/claude-squad). Active but slow-moving.**
- Status: v1.0.0 tagged 2025-04-03; the latest tag is v1.0.20 (2026-08-20) — [Go proxy v1.0.0](https://proxy.golang.org/github.com/smtg-ai/claude-squad/@v/v1.0.0.info), [Go proxy latest](https://proxy.golang.org/github.com/smtg-ai/claude-squad/@latest).
- Description: a terminal app (`cs`) that manages Claude Code, Codex, Gemini and Aider "in separate workspaces". Its "How it works" section lists (1) tmux sessions per agent, (2) git worktrees per session/branch, (3) a TUI. It requires tmux and gh. License AGPL-3.0 — [README](https://github.com/smtg-ai/claude-squad).
- Configuration lives in `~/.claude-squad/config.json`. `profiles` map names to launch commands, for example `aider --model ollama_chat/gemma3:1b`. `-p/--program` picks the agent and `-y/--autoyes` (experimental) auto-accepts prompts — [README](https://github.com/smtg-ai/claude-squad).
- Implementation (Go): the TUI attaches to tmux through a PTY (`creack/pty`, `tmux attach-session`).
  - It detects changes by hashing `capture-pane` output with sha256.
  - It detects a pending prompt by matching literal UI strings: `"No, and tell Claude what to do differently"` for Claude and `"(Y)es/(N)o/(D)on't ask again"` for Aider.
  - It auto-dismisses `"Do you trust the files in this folder?"` by sending Enter.
  - Source: [session/tmux/tmux.go](https://github.com/smtg-ai/claude-squad/blob/main/session/tmux/tmux.go).
- The autoyes daemon is a detached child process with a PID file. It loads stored instances, polls each one, sends Enter (`TapEnter`) whenever `HasUpdated()` reports a prompt, and saves instances on SIGINT/SIGTERM — [daemon/daemon.go](https://github.com/smtg-ai/claude-squad/blob/main/daemon/daemon.go).
- FAQ: "failed to start new session: timed out waiting for tmux session" is fixed by updating the underlying program. This shows the coupling to the CLI's startup UI — [README](https://github.com/smtg-ai/claude-squad).

**Agent Deck (asheshgoplani/agent-deck). Very active.**
- Status: v0.1.0 on 2025-12-04; v1.16.28 on 2026-10-10 — [Go proxy v0.1.0](https://proxy.golang.org/github.com/asheshgoplani/agent-deck/@v/v0.1.0.info), [Go proxy latest](https://proxy.golang.org/github.com/asheshgoplani/agent-deck/@latest).
- Stack: a Go TUI over tmux, MIT-licensed, for macOS, Linux and WSL — [README](https://github.com/asheshgoplani/agent-deck).
- Status detection "smart polling" yields Running, Waiting ("needs your input"), Idle and Error. Waiting sessions also appear in the tmux status bar — [README](https://github.com/asheshgoplani/agent-deck).
- `agent-deck web` serves a web UI on `http://127.0.0.1:8420`. There is a web/API fork endpoint (`POST /api/sessions/{id}/fork`) and a cost dashboard that uses SSE live updates — [README](https://github.com/asheshgoplani/agent-deck).
- "Conductors" are persistent agent sessions that "monitor and orchestrate all your other sessions". They auto-respond when confident and escalate otherwise.
  - Each conductor has a directory under `~/.local/share/agent-deck/conductor/<name>/` holding `CLAUDE.md`/`AGENTS.md`, `meta.json`, `state.json` and `task-log.md`, plus a shared `bridge.py` daemon for Telegram/Slack (the docs also cover Discord).
  - Source: [README](https://github.com/asheshgoplani/agent-deck).
- "Watchers" let external events wake a conductor: GitHub events, gmail, ntfy pushes and meetings.
  - Events are HMAC-verified (GitHub) and deduplicated in SQLite by `(watcher_name, event_id)`.
  - The "Doorbell rule" says watchers forward a short event string to the conductor and never launch sessions themselves. Otherwise the spawned session "becomes an orphan whose status events never route back".
  - Source: [README](https://github.com/asheshgoplani/agent-deck).
- An MCP socket pool shares MCP server processes across sessions over Unix sockets. The README claims "85-90%" lower MCP memory use and ~3 s reconnect after an MCP crash — [README](https://github.com/asheshgoplani/agent-deck).

**Gas Town (steveyegge/gastown). Active; last tag about 4 months ago.**
- Status: v1.0.0 tagged 2026-04-03; latest tag v1.2.1 (2026-06-06) — [Go proxy v1.0.0](https://proxy.golang.org/github.com/steveyegge/gastown/@v/v1.0.0.info), [tag list](https://proxy.golang.org/github.com/steveyegge/gastown/@v/list).
- Description: a "multi-agent orchestration system for Claude Code, GitHub Copilot, and other AI agents with persistent work tracking". The README claims it scales "comfortably to 20-30 agents" — [README](https://github.com/steveyegge/gastown).
- Process model: `gt up` boots Dolt, a Go daemon, the Deacon, the Mayor, and per-rig Witnesses and Refineries.
  - tmux 3.0+ is required for the tmux-backed roles. A "minimal mode (no tmux)" runs runtimes by hand and only tracks state.
  - The HQ is a directory (`~/gt`) that can also be a git repo.
  - Source: [README](https://github.com/steveyegge/gastown).
- State lives in Beads (`bd`), a git-backed issue ledger now backed by Dolt (Beads 0.57.0+ required), with sqlite3 for convoy queries — [README](https://github.com/steveyegge/gastown). Beads itself is active: v1.3.1 on 2026-09-30 — [Go proxy](https://proxy.golang.org/github.com/steveyegge/beads/@latest).
- Roles:
  - **Mayor**: coordinator. A Claude Code instance.
  - **Polecats**: workers with "persistent identity but ephemeral sessions".
  - **Hooks**: git-worktree-backed persistent storage for agent work.
  - **Witness**: per-rig lifecycle manager.
  - **Deacon**: cross-rig patrol.
  - **Dogs**: maintenance.
  - **Refinery**: Bors-style bisecting merge queue.
  - **Scheduler**: `scheduler.max_polecats` capacity governor.
  - **Seance**: queries predecessor sessions via `.events.jsonl`.
  - **Wasteland**: federation via DoltHub.
  - Source: [README](https://github.com/steveyegge/gastown).
- Watchdog chain: "Daemon (Go process) ← heartbeat every 3 min → Boot (AI agent) → Deacon (AI agent) → Witnesses & Refineries". A web dashboard (htmx) and a `gt feed` TUI complete the monitoring, and OpenTelemetry logs and metrics are exported — [README](https://github.com/steveyegge/gastown).

**Gas City (gastownhall/gascity). New in 2026, active.**
- Status: v1.5.0 on 2026-10-05 — [Go proxy](https://proxy.golang.org/github.com/gastownhall/gascity/@latest).
- Description: "an orchestration-builder SDK" that "extracts the reusable infrastructure from Gas Town". It provides declarative `city.toml`, a "controller/supervisor loop that reconciles desired state to running state", and runtime providers "tmux, subprocess, exec, ACP, Kubernetes, and herdr" — [README](https://github.com/gastownhall/gascity).
- tmux is "the default session backend **and** the fallback". Beads/Dolt is the default store; a file-based store (`GC_BEADS=file`) is an alternative — [README](https://github.com/gastownhall/gascity).
- Every says it was rebuilt by Chris Sells and Julian Knutsen "with Yegge's blessing". Knutsen's server runs "roughly 100 agents" merging "about 50 pull requests per day" on "roughly a billion tokens daily". Verdict: "Learn from the ideas. Skip the toolkit for now." — [Every, 2026-05-19](https://every.to/context-window/inside-the-100-agent-software-factory) (journalism/practitioner).

#### B. Desktop "parallel workspaces" apps

**Conductor (conductor.build; Melty, Inc., YC S24). Active, commercial, closed-source Mac app.**
- Company: "Y Combinator Summer 2024 company", founded 2024. The privacy notice names "Melty, Inc." The site says it has "grown 10x since January and raised $24M" — [conductor.build llms-full.txt](https://www.conductor.build/llms-full.txt).
- Harnesses are Claude Code, Codex, Cursor (via the Cursor API) and OpenCode. Conductor "comes bundled with its own installation of Claude Code and Codex" at `~/Library/Application Support/com.conductor.app/bin` and stores data there and in `~/conductor` — [docs](https://www.conductor.build/llms-full.txt).
- Auth: Conductor "uses the auth tokens already saved on your machine (API key or Claude Pro/Max)". It warns that an `ANTHROPIC_API_KEY` in the environment may switch billing to the API — [docs](https://www.conductor.build/llms-full.txt).
- Each workspace gets "its own branch, git worktree, run environment, and shared context folder". The gitignored `.context` folder holds handoffs. Workspaces are named after cities, and `CONDUCTOR_PORT` gives each workspace a port range — [docs](https://www.conductor.build/llms-full.txt).
- Conductor Cloud: "we automatically create a Vercel sandbox for your agent" in us-east-1, so "even if you close the Conductor app, work continues". Chats in cloud workspaces are stored on Conductor's servers. A REST API (`/v0/workspaces/...`) and an MCP server cover cloud workspaces — [docs](https://www.conductor.build/llms-full.txt).
- Caution: Conductor's UI and runtime stack is not documented. The only `src-tauri` string in its docs is a repo-icon lookup path for *your* repository, so it is not evidence that Conductor is a Tauri app — [docs](https://www.conductor.build/llms-full.txt).

**Sculptor (imbue-ai/sculptor). Active open-source "experimental research preview".**
- Platforms: desktop app for macOS (Apple Silicon) and Linux — [README](https://github.com/imbue-ai/sculptor).
- A workspace is now "by default ... a **git worktree**". "Clone" and "In-place" modes sit under Settings > Experimental. Branches follow `<user>/<slug>` and there is a branch-cleanup policy — [workspaces doc](https://github.com/imbue-ai/sculptor/blob/main/docs/help/workspaces.md).
- Running the backend in a container is now experimental. It works through a "Custom Backend Command" that prints a URL the desktop app connects to — [container_backend doc](https://github.com/imbue-ai/sculptor/blob/main/docs/help/experimental/container_backend.md).
- Imbue's product page still describes containers per agent and "Pairing Mode" — [imbue.com/sculptor](https://imbue.com/sculptor), via search summary. That page may be stale relative to the repo.
- Harness integration:
  - "Sculptor runs Claude Code as a streaming JSON process with its control protocol enabled". Claude "runs sandboxed, with tool permissions auto-approved".
  - Sculptor disables Claude's AskUserQuestion and ExitPlanMode tools and registers its own replacements.
  - It adds a pre-compaction hook and asks Claude for its context fullness each turn.
  - Pi runs "in RPC mode".
  - Lifecycle claims: "An interrupted turn survives a quit, or a crash of Sculptor itself"; pending questions survive app reload.
  - Source: [integrated_harnesses doc](https://github.com/imbue-ai/sculptor/blob/main/docs/help/integrated_harnesses.md).

**Crystal, now Nimbalyst (stravu).**
- Crystal "has been deprecated and replaced by **Nimbalyst**. Deprecated: February 2026." — [stravu/crystal README](https://github.com/stravu/crystal).
- Nimbalyst is an MIT-licensed Electron "visual workspace". It offers parallel sessions isolated in git worktrees, a "Session kanban", agent-readable task trackers, heterogeneous agents (Codex and Claude Code; OpenCode and Copilot in alpha) and an iOS companion with push notifications — [Nimbalyst README](https://github.com/Nimbalyst/nimbalyst).
- Its Electron package depends on `@anthropic-ai/claude-agent-sdk` 0.3.293, `@openai/codex-sdk`, `node-pty` and Electron 43.7.0. It embeds the vendor SDKs rather than scraping TUIs — [packages/electron/package.json](https://github.com/Nimbalyst/nimbalyst/blob/main/packages/electron/package.json).

**Superset (superset-sh/superset). Active.**
- Stack: an "Electron desktop IDE, Next.js web apps, and an Expo mobile app". A separate **host-service** package shares a version with the desktop app and CLI, using tRPC and Drizzle — [AGENTS.md](https://github.com/superset-sh/superset/blob/main/AGENTS.md).
- "Terminals are backed by a background daemon, so running processes and scrollback survive app restarts and updates." — [docs](https://docs.superset.sh/llms-full.txt).
- Superset "installs lifecycle hooks and command wrappers into each agent's config" (in `~/.superset/bin`) that report start, finish and waiting-for-input. They "no-op outside Superset terminals" — [docs](https://docs.superset.sh/llms-full.txt).
- Other surfaces: a CLI, a TypeScript SDK, an MCP server, Automations (scheduled agent sessions), Slack/Linear intake, and an iPhone app (Superset Pro, iOS 26+). Remote access goes through a relay and is off by default; hosts register with `superset start --daemon`. Over 20 CLI agents are supported — [README](https://github.com/superset-sh/superset), [docs](https://docs.superset.sh/llms-full.txt).

**Emdash (generalaction/emdash). Active.**
- Stack: a "local-first, cross-platform Electron app". Each task gets its own git worktree, locally or over SSH/SFTP. App state lives in local SQLite. Tasks can come in from Linear, GitHub, Jira and others. License Apache-2.0 — [README](https://github.com/generalaction/emdash).
- Two runtime paths: "Legacy/TUI conversations run through PTY services", while "Structured chat conversations use ACP" via provider plugins and ACP adapters. Data is in SQLite via Drizzle — [AGENTS.md](https://github.com/generalaction/emdash/blob/main/AGENTS.md).
- Emdash installs "marker-tagged" hooks in each agent's user-level config "to track status, notifications, and resumable sessions" — [README](https://github.com/generalaction/emdash).

**Xum, formerly Mux (coder/mux). Active.**
- Renamed "from Mux to Xum after Mux.com raised a trademark concern" — [README](https://github.com/coder/mux).
- Xum has "a custom agent loop", so it does not wrap Claude Code. Runtimes are local, worktree and SSH, and it supports Ollama and OpenRouter. Other features: opportunistic compaction, a cost/token UI, and a "server mode" with a mobile UI. License AGPL-3.0, © 2026 Coder — [README](https://github.com/coder/mux).

**Vibe Kanban (BloopAI/vibe-kanban). Sunsetting; community-maintained. STALE? (last release April 2026)**
- bloop announced the shutdown on 2026-04-10 because it "couldn't find a business model". Remote services lasted 30 days before a move "to a fully local architecture"; "local workspaces will continue to function" — [shutdown post](https://www.vibekanban.com/blog/shutdown).
- The last npm release is 0.1.44 (2026-04-24) — [npm registry](https://registry.npmjs.org/vibe-kanban).
- Stack: a Rust backend (sqlx) plus a web app, a built-in MCP server (`MCP_HOST`/`MCP_PORT`), worktree cleanup of "orphan and expired" workspaces, and a relay tunnel mode — [README](https://github.com/BloopAI/vibe-kanban).
- The Claude executor runs `claude -p --output-format=stream-json --input-format=stream-json --permission-prompt-tool=stdio` with the SDK control protocol and `--resume <session_id>` / `--resume-session-at` — [executors/claude.rs](https://github.com/BloopAI/vibe-kanban/blob/main/crates/executors/src/executors/claude.rs).

#### C. Native terminals and TUIs as agent substrate

**cmux (manaflow-ai/cmux). Active.**
- A "Ghostty-based macOS terminal with vertical tabs and notifications for AI coding agents", built with Swift/AppKit on libghostty and GPL-licensed — [README](https://github.com/manaflow-ai/cmux). It launched on Show HN on Feb 19 (2026 by context) — [cmux blog](https://cmux.com/blog/show-hn-launch).
- Notifications come from OSC 9/99/777 sequences and a `cmux notify` CLI "you can wire into agent hooks". A CLI and socket API can "create workspaces/tabs, split panes, send keystrokes". `cmux claude-teams` runs Claude agent teams "as native splits ... No tmux required" — [README](https://github.com/manaflow-ai/cmux).
- On relaunch it restores layout, cwd, scrollback (best effort) and browser state, but "does not checkpoint arbitrary live process state". `cmux local-tmux` opts into a user-owned tmux server under `~/.cmux/local-tmux` where "the tmux server owns the shell, agent, dev-server, PTY, and scrollback process; cmux owns only the client surface" — [README](https://github.com/manaflow-ai/cmux), [local-tmux doc](https://github.com/manaflow-ai/cmux/blob/main/docs/local-tmux.md).
- Agent sessions resume "when hooks have saved a native session ID". `cmux hooks setup` covers Claude Code, Codex, OpenCode, Gemini, Amp and others — [README](https://github.com/manaflow-ai/cmux).

**Toad (batrachianai/toad; Will McGugan). Active-ish; last PyPI release about 4.5 months ago.**
- Releases: first PyPI 0.5.0a1 on 2025-12-16; latest 0.6.20 on 2026-05-26 — [PyPI](https://pypi.org/project/batrachian-toad/).
- A Python/Textual TUI ("uv tool install batrachian-toad --python 3.14") that is "an interface to coding agents" through the Agent Client Protocol. Features: an agent "App store", an integrated real shell, concurrent sessions (ctrl+s overview) and session resume (ctrl+r) — [README](https://github.com/batrachianai/toad).
- OpenHands announced a Toad partnership on 2025-12-18 — [OpenHands blog](https://openhands.dev/blog/20251218-openhands-toad-collaboration).

**Zed (editor, ACP host).**
- External Agents "integrate with Zed through the Agent Client Protocol". The ACP Registry "is the primary way to install" them. External agents "run as separate processes that communicate with Zed over ACP", and Claude Agent "owns its own authentication and billing". "Terminal Threads" are a separate option for running a CLI/TUI directly — [Zed docs](https://github.com/zed-industries/zed/blob/main/docs/src/ai/external-agents.md).

#### D. Agent-as-server runtimes and protocols

**OpenCode (sst/opencode, now anomalyco/opencode). Very active.**
- "When you run `opencode` it starts a TUI and a server. Where the TUI is the client that talks to the server." `opencode serve` runs a headless HTTP server with an OpenAPI 3.1 spec at `/doc` — [server docs source](https://github.com/anomalyco/opencode/blob/master/packages/web/src/content/docs/server.mdx).
- API surface:
  - `GET /event` is an SSE stream ("First event is `server.connected`, then bus events"). There is also `/global/event`.
  - Session endpoints: create, list, `/session/status`, `fork`, `abort`, `diff`, `summarize`, `revert`, `prompt_async` (fire-and-forget), and `POST /session/:id/permissions/:permissionID` to answer approvals.
  - `/tui/*` endpoints drive a running TUI.
  - HTTP basic auth comes from `OPENCODE_SERVER_PASSWORD`.
  - Source: [server docs source](https://github.com/anomalyco/opencode/blob/master/packages/web/src/content/docs/server.mdx).
- The README lists a desktop app (beta) for macOS, Windows and Linux — [README](https://github.com/anomalyco/opencode).
- Releases: a new `@opencode/cli` package was created 2026-09-02 (v2.0.28 on 2026-10-13); the older `opencode-ai` is at v1.19.0 (2026-10-11) — [npm @opencode/cli](https://registry.npmjs.org/@opencode/cli), [npm opencode-ai](https://registry.npmjs.org/opencode-ai).
- A secondary report says the 2.0 beta (Sep 22) made "a single Hono/SQLite server" back the TUI, a new Electron desktop app and the web UI — [byteiota](https://byteiota.com/?p=23518). Unverified against official notes.

**OpenAI Codex app-server. Active; it is the backbone of all Codex surfaces.**
- OpenAI's 2026-02-04 post (Celia Chen) describes the App Server as "a bidirectional JSON-RPC API" connecting the web app, CLI, IDE extension and macOS app to one harness. "the server can also send its own requests, such as approval prompts, and pause the turn until the client responds" — [OpenAI post](https://openai.com/index/unlocking-the-codex-harness/). The page returns 403 to fetchers; content was read via search snippets.
- Transports: `stdio` (default, "newline-delimited JSON (JSONL)"), `websocket` ("experimental and unsupported") and a Unix socket.
- Framing is JSON-RPC 2.0 without the `jsonrpc` field. Each connection opens with `initialize`, then the `initialized` notification.
- Primitives: Thread ("conversation"), Turn ("a single user request and the agent work that follows"), Item (message, command, file change).
- Methods: `thread/start|resume|fork`, `turn/start`, `turn/steer` ("adds input to the in-flight turn") and `turn/interrupt`.
- Notifications: `turn/started|completed`, `item/started|completed` and `item/agentMessage/delta`. Server-initiated approval requests include `item/permissions/requestApproval`.
- "Threads are stored as JSONL logs on disk" with SQLite-backed metadata. "If you are automating jobs or running Codex in CI, use the Codex SDK instead."
- Source: [Codex app-server docs](https://learn.chatgpt.com/docs/app-server).
- The README notes "local stdio desktop sessions (`Codex Desktop`)", "in-process connections" and "WebSocket and remote-control peers" — [app-server README](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md).
- The Codex CLI is at 0.162.1 (2026-10-09) — [npm](https://registry.npmjs.org/@openai/codex).
- The Codex macOS app launched 2026-02-03 as a "command center": project threads, per-agent git worktrees, Automations whose results land "in a review queue", and skills — [AI Pulse](https://artificialintelligencepulse.it.com/2026/02/03/codex-on-macOS/), [heise](https://heise.de/-11165126) (news coverage).

**Agent Client Protocol (ACP; Zed-originated, now the agentclientprotocol org).**
- JSON-RPC 2.0; agents "typically run as subprocesses of the Client".
- Agent methods: `initialize`, `authenticate`, `session/new`, `session/prompt`, optional `session/load` and `session/set_mode`, and the `session/cancel` notification.
- Client methods: `session/request_permission`, optional `fs/read_text_file|write_text_file` and `terminal/create|output|release|wait_for_exit|kill`, plus `elicitation/create`.
- `session/update` notifications carry message chunks, tool calls, plans, mode changes and compactions.
- Source: [ACP overview](https://agentclientprotocol.com/protocol/overview).
- "The current stable ACP protocol version is `1`". Schemas exist for v1 and v2, with official SDKs in Kotlin, Java, Python, Rust and TypeScript — [ACP repo](https://github.com/agentclientprotocol/agent-client-protocol). The TypeScript SDK is at 1.8.0 (2026-10-09) — [npm](https://registry.npmjs.org/@agentclientprotocol/sdk).
- Claude Code adapter history: `@zed-industries/claude-code-acp` (created 2025-08-28) became `@zed-industries/claude-agent-acp` (2026-02-17) and then `@agentclientprotocol/claude-agent-acp` (2026-03-26), now at 0.89.1 (2026-10-10). The old names are deprecated — [npm claude-code-acp](https://registry.npmjs.org/@zed-industries/claude-code-acp), [npm claude-agent-acp (zed)](https://registry.npmjs.org/@zed-industries/claude-agent-acp), [npm current](https://registry.npmjs.org/@agentclientprotocol/claude-agent-acp). `codex-acp` likewise moved to `@agentclientprotocol/codex-acp` (June 2026) — [npm](https://registry.npmjs.org/@zed-industries/codex-acp).
- The adapter "implements an ACP agent by using the official Claude Agent SDK". It supports permission requests, interactive and background terminals, and subagents as child sessions with states `running`/`requires_action`/`idle`.
  - `session/cancel` "interrupts Claude Code".
  - An experimental ACP v2 mode (`CLAUDE_AGENT_ACP_EXPERIMENTAL_V2=1`) serves `session/new|list|resume (with replay)|close|prompt|cancel|delete|fork`.
  - JetBrains "AIR" has its own extensions.
  - Source: [claude-agent-acp README](https://github.com/zed-industries/claude-agent-acp).

#### E. First-party Claude Code multi-session machinery (fast-moving: v2.1.0 on 2026-01-07, v2.1.296 on 2026-10-09)

- Version cadence: about 300 point releases in 9 months — [npm registry times](https://registry.npmjs.org/@anthropic-ai/claude-code).
- **Agent view / background supervisor.** Introduced in v2.1.139 (2026-05-11), still research preview — [agent-view doc](https://code.claude.com/docs/en/agent-view), [npm times](https://registry.npmjs.org/@anthropic-ai/claude-code).
  - `claude agents` opens "one screen for all your background sessions". `claude --bg "<prompt>"` dispatches one.
  - Shell commands: `claude attach|logs|stop|respawn|rm <id>`, `claude daemon status|logs|stop`.
  - "A separate supervisor process runs them, so you can close agent view, close your shell". "Each session is its own Claude Code process under the supervisor", and "the supervisor runs each background session's terminal in its own host process".
  - State files: `~/.claude/daemon.log`, `~/.claude/daemon/roster.json` and `~/.claude/jobs/<id>/state.json`. The jobs files "are not a stable interface"; `claude agents --json` "is the supported way to read session state from outside Claude Code".
  - Source: [agent-view doc](https://code.claude.com/docs/en/agent-view).
- **Agent teams.** Shipped with Opus 4.6 on 2026-02-05 as experimental — [heise](https://heise.de/-11167248).
  - Enabled by `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. A lead plus "separate Claude Code instances", a shared task list and a mailbox.
  - Mailboxes are JSON files at `~/.claude/teams/{team-name}/inboxes/{agent-name}.json`. Tasks live in `~/.claude/tasks/{team-name}/`, and claiming "uses file locking".
  - Display modes: "in-process" (default) or split panes via tmux or iTerm2 (`teammateMode`: auto|in-process|tmux|iterm2). Split panes are "not supported in VS Code's integrated terminal, Windows Terminal, or Ghostty".
  - "In non-interactive mode with the `-p` flag, including Agent SDK sessions, Claude doesn't spawn teammates."
  - Source: [agent-teams doc](https://code.claude.com/docs/en/agent-teams).
- **Cross-session messaging.** Requires v2.1.224+ on macOS/Linux, released 2026-08-07 — [doc](https://code.claude.com/docs/en/cross-session-messaging), [npm times](https://registry.npmjs.org/@anthropic-ai/claude-code).
  - Sessions discover peers with `ListAgents` and send with `SendMessage`.
  - "Claude Code binds an inbox socket for each session", a Unix domain socket restricted to the OS user. The path is exported as `CLAUDE_CODE_MESSAGING_SOCKET` with a per-session token `CLAUDE_CODE_MESSAGING_TOKEN`.
  - Same-machine delivery goes over the socket, "never through Anthropic servers". `claude -p` workers bind sockets too.
  - Source: [cross-session-messaging doc](https://code.claude.com/docs/en/cross-session-messaging).
- **Remote Control.** Research preview announced 2026-02-24/25 — [Gigazine](https://gigazine.net/gsc_news/en/20260225-claude-code-remote-control).
  - The local process "makes outbound HTTPS requests only and never opens inbound ports ... registers with the Anthropic API and polls for work". The transcript is stored on Anthropic servers while connected.
  - Pro/Max/Team/Enterprise only; "API keys are not supported".
  - Server mode is `claude remote-control --spawn same-dir|worktree|session --capacity <N>` (default 32 concurrent sessions). Stopped servers can be revived for "about four hours".
  - Source: [remote-control doc](https://code.claude.com/docs/en/remote-control).
- **Claude Code on the web / cloud sessions.** Launched as a research preview for Pro/Max; one news report dates it 2025-10-20 (STALE? for launch details) — [Neowin](https://www.neowin.net/news/anthropics-claude-code-is-now-available-on-the-web/), [AIM](https://analyticsindiamag.com/ai-news-updates/anthropic-brings-claude-code-to-the-browser) (news coverage via search summary).
  - Each session runs in a VM. "GitHub credentials ... never enter a session's VM", and git goes through a GitHub proxy.
  - `claude --cloud "<task>"` starts sessions in parallel; `claude --teleport` pulls one local. `claude -p "msg" --cloud <session-id>` queues steering messages, and `--output-format json` returns `{ok, session_id, url}`.
  - Source: [web doc](https://code.claude.com/docs/en/claude-code-on-the-web).
- **Other first-party surfaces** (per the docs index):
  - Projects: cloud "threads" coordinated by Claude; public beta on Pro/Max.
  - Dispatch: phone tasks run on the Desktop app.
  - Channels: push events into a session.
  - Scheduled tasks: CLI `/loop` and cron tools, Desktop, and cloud "routines".
  - Desktop parallel sessions with git isolation.
  - Sources: [agents doc](https://code.claude.com/docs/en/agents), [remote-control doc table](https://code.claude.com/docs/en/remote-control), [llms.txt](https://code.claude.com/docs/llms.txt).

#### F. Remote and mobile control layers

**Happy (slopus/happy). Active, but pivoting to its own harness.**
- The original CLI (`npm i -g happy`; `happy claude`, `happy codex`, `happy acp <agent>`) "wraps the Claude Code and Codex harnesses directly in your terminal" and adds an end-to-end-encrypted relay (`happy-server`) for phone and web control — [happy README](https://github.com/slopus/happy), [happy-cli README](https://github.com/slopus/happy/blob/main/packages/happy-cli/README.md).
- The CLI's daemon "lets you spawn and manage coding sessions remotely"; its state is in `~/.happy/daemon.state.json` — [happy-cli README](https://github.com/slopus/happy/blob/main/packages/happy-cli/README.md).
- The new "Happy Agent is the headless daemon: it owns agents, tools, permissions, durable state, and the public API". It is built "on top of Pi's foundations", recreates Claude Code/Codex/Grok prompts and tools, and "picks up the Claude, Codex, and Grok sign-ins already on your machine" — [happy-agent README](https://github.com/slopus/happy-agent).
- Releases: npm `happy` 1.2.5 (2026-09-22); `happy-coder` is deprecated (last release 2026-05-14) — [npm happy](https://registry.npmjs.org/happy), [npm happy-coder](https://registry.npmjs.org/happy-coder).

**Omnara (omnara-ai/omnara). Pivoted.**
- The README now describes "an open-source managed agent platform ... A self-hostable alternative to Claude Managed Agents and OpenAI's Agents API". Agents are defined in YAML, run on machines or sandboxes, and expose a TypeScript SDK event stream. License Apache 2.0 — [README](https://github.com/omnara-ai/omnara).
- Releases: `omnara` 1.0.27 (2026-10-10); `@omnara/sdk` created 2026-08-19 — [npm](https://registry.npmjs.org/omnara), [npm sdk](https://registry.npmjs.org/@omnara/sdk).

#### G. Cloud and background orchestrators

**Terragon. Shut down.**
- "Snapshot notice (January 16, 2026): This repository is an open-source snapshot of Terragon at the time of shutdown." — [terragon-oss README](https://github.com/terragon-labs/terragon-oss).
- Architecture:
  - Main web app (`apps/www`), a WebSocket broadcast service, PostgreSQL and Redis.
  - Pluggable sandbox providers, with one sandbox and repo copy per task, and auto branches/PRs.
  - The `terry` CLI does "local task takeover" and also acts as an MCP server.
  - "BYO Subscription & API Keys", plus automations.
  - Source: [README](https://github.com/terragon-labs/terragon-oss).

**Dagger Container Use. STALE? (no release since Aug 2025)**
- An "open-source MCP server" (`claude mcp add container-use -- container-use stdio`). "Each agent gets a fresh container in its own git branch"; you review with `git checkout <branch_name>` and can drop into an agent's terminal — [README](https://github.com/dagger/container-use).
- Latest tag v0.4.2 (2025-08-15) — [Go proxy](https://proxy.golang.org/github.com/dagger/container-use/@latest).

**OpenAI Symphony.**
- Open-sourced 2026-04-28 (Apache-2.0) as a spec plus an Elixir reference implementation built on the Codex App Server. OpenAI reports "a 500% increase in landed pull requests" internally (company-reported) — [Help Net Security](https://www.helpnetsecurity.com/2026/04/28/openai-symphony-codex-orchestration-linear/), [README](https://github.com/openai/symphony).
- It is a "long-running automation service that continuously reads work from a configured issue tracker, creates an isolated workspace for each issue, and runs a coding agent session". Policy lives in a repo-owned `WORKFLOW.md`. Restart recovery is "tracker-driven and filesystem-driven (without a durable orchestrator DB)" — [SPEC](https://github.com/openai/symphony/blob/main/SPEC.md).

**Warp Oz.**
- Launched 2026-02-10 as "the orchestration platform for cloud coding agents" with Docker-backed sandboxes, run audit trails, and CLI/API/SDK triggers — [Warp newsroom](https://warp.dev/newsroom/2026/2/10/warp-launches-oz-the-orchestration-platform-for-cloud-coding-agents).
- A secondary summary of a May 2026 post calls it a "multi-harness control plane" for Claude Code, Codex and Warp Agent — [plushcap](https://www.plushcap.com/companies/warp/blog/summaries/2026/5) (unverified). This matters because Narwarp descends from Warp.

**Anthropic Claude Managed Agents.**
- The engineering post (2026-04-08) describes session/harness/sandbox decoupling. See Q7 for details — [Anthropic](https://www.anthropic.com/engineering/managed-agents).

### Inferences
- **Who owns the PTY or process** has become the main architectural axis. Four camps:
  1. A tmux server (Claude Squad, Agent Deck, Gas Town/Gas City, agent-teams split panes, cmux local-tmux).
  2. An app-owned host daemon (Claude Code supervisor, Superset host-service, Happy daemon / Happy Agent).
  3. An in-app PTY that dies with the UI (cmux default, Emdash PTY path, Electron apps without a daemon).
  4. No PTY at all, only a structured subprocess or server (Codex app-server, OpenCode server, ACP agents, stream-json runners, the Agent SDK).
- Claude Code itself moved from camp 3 to camp 2 in May 2026. This lowers the value of third-party tmux managers for Claude-only workflows.
- Durable, maintained projects either own a protocol or SDK surface (OpenCode, Codex app-server, ACP, Claude Code's supervisor) or are thin, fast-shipping UIs over vendor CLIs (Agent Deck, Superset, Conductor, cmux). Cloud-only startups and heavyweight custom orchestrators fared worse in 2026 (Terragon shut down, Vibe Kanban sunsetting, Crystal deprecated). Gas Town spun off Gas City and is called "experimental" by reviewers.
- For Narwarp, a Rust fork of the Warp terminal, the closest analogues are:
  - cmux: terminal as primitive, hooks plus notifications, optional tmux owner.
  - Superset: terminal app plus a host daemon that survives restarts.
  - Agent Deck: TUI plus conductor, watchers and chat bridges, which is very close to Narwarp's "life management" and Discord ambitions.

### Gaps
- How Conductor drives Claude Code internally (PTY vs headless/SDK) is not stated in its public docs, and its UI stack is undocumented.
- Terragon's reason for shutting down was not found in any primary source.
- Omnara's 2025 product (remote control for Claude Code) and the date it pivoted could not be confirmed. The `@omnara/sdk` creation date (2026-08-19) only bounds it.
- Gas Town commit activity after the 2026-06-06 tag could not be checked because the GitHub API was blocked here. Only tag dates via the Go proxy were available.
- OpenCode 2.0 architecture details (Hono/SQLite single server, Electron desktop) come from one secondary article.
- The original Sculptor container plus "Pairing Mode" design is documented only on Imbue's marketing page, so its exact deprecation date is unknown.

---

## Q2. How do harnesses launch and drive Claude Code (PTY/tmux, headless stream-json, Agent SDK, ACP, hooks), and what are the tradeoffs?

### Takeaway
Six launch and drive modes are in use:
1. Interactive TUI in tmux, driven by keystrokes and screen scraping.
2. Interactive TUI in an app- or daemon-owned PTY, with hooks for telemetry.
3. Headless `claude -p` with bidirectional stream-json and the control protocol.
4. The Claude Agent SDK embedded in the app.
5. ACP through the Claude Agent SDK adapter.
6. New first-party control surfaces (`claude --bg` / `claude agents --json`, inbox sockets, Remote Control).

Interactive modes keep full fidelity and plain subscription usage, but expose state only through hooks or scraping. Headless, SDK and ACP modes give structured events, programmatic approvals and cost data, but drop some interactive features. They also face billing and terms friction: `--bare` and SDK use point toward API keys.

### Cited Findings
- **Mode 1, tmux PTY plus send-keys/capture-pane**:
  - Claude Squad decides "needs input" by matching literal UI text and auto-approves by sending Enter — [tmux.go](https://github.com/smtg-ai/claude-squad/blob/main/session/tmux/tmux.go), [daemon.go](https://github.com/smtg-ai/claude-squad/blob/main/daemon/daemon.go).
  - Gas Town uses Claude hooks "in `.claude/settings.json` (managed via `--settings` flag) for mail injection and startup". For runtimes without hooks (Codex) it "sends a startup fallback after the session is ready: `gt prime`...". For Copilot it "uses a 5-second ready delay instead of prompt detection" — [Gas Town README](https://github.com/steveyegge/gastown).
  - Widely copied tmux "agent skill" pages recommend sending text and Enter as separate tmux sends "to avoid paste/multiline edge cases" and grepping panes for "proceed"/"permission" to detect input requests. Example: [skills.sh tmux skill](https://skills.sh/termix-official/cryptoclaw/tmux) (practitioner; read via search summary, exact page attribution uncertain).
  - An RTK issue notes "capture-pane token waste compounds — every poll burns context" when an agent polls panes itself — [goodfirstissue.org/rtk](https://www.goodfirstissue.org/rtk-ai/rtk/issues/2031) (practitioner).
- **Mode 2, PTY plus hooks**:
  - Superset wraps launch commands and installs lifecycle hooks that report start, finish and waiting — [Superset docs](https://docs.superset.sh/llms-full.txt).
  - Emdash installs marker-tagged hooks "to track status, notifications, and resumable sessions" — [Emdash README](https://github.com/generalaction/emdash).
  - cmux wires `cmux notify` into hooks and resumes sessions from hook-saved native session IDs — [cmux README](https://github.com/manaflow-ai/cmux).
  - The crmux project uses SessionStart hooks and the statusLine command for model and context metadata instead of scraping — [crmux on docs.rs](https://docs.rs/crate/crmux/0.20.0).
- **Mode 3, headless stream-json**:
  - Vibe Kanban runs `-p --output-format=stream-json --input-format=stream-json --permission-prompt-tool=stdio`, initializes the control protocol and resumes with `--resume`/`--resume-session-at` — [claude.rs](https://github.com/BloopAI/vibe-kanban/blob/main/crates/executors/src/executors/claude.rs).
  - Sculptor runs Claude Code "as a streaming JSON process with its control protocol enabled" with auto-approved permissions, replacing AskUserQuestion and ExitPlanMode with its own tools — [Sculptor doc](https://github.com/imbue-ai/sculptor/blob/main/docs/help/integrated_harnesses.md).
  - Anthropic's C-compiler loop relaunched `claude` in a bash `while true` loop with `--dangerously-skip-permissions`, a `-p` prompt read from `AGENT_PROMPT.md`, and `--model`, logging output per session — [Anthropic](https://www.anthropic.com/engineering/building-c-compiler). The exact command line is paraphrased.
  - Headless facts:
    - `--output-format json` includes `total_cost_usd`.
    - `--permission-prompts none` exists for unattended runs.
    - `--bare` "never reads OAuth credentials or the system keychain", is "recommended for scripted and SDK calls, and will become the default for `-p` in a future release".
    - Non-interactive `-p` runs don't clean up their worktrees.
    - Sources: [headless doc](https://code.claude.com/docs/en/headless), [worktrees doc](https://code.claude.com/docs/en/worktrees).
  - Agent teams cannot spawn in `-p` or Agent SDK sessions — [agent-teams doc](https://code.claude.com/docs/en/agent-teams).
- **Mode 4, Agent SDK in-app**:
  - Nimbalyst bundles `@anthropic-ai/claude-agent-sdk` and `@openai/codex-sdk` — [package.json](https://github.com/Nimbalyst/nimbalyst/blob/main/packages/electron/package.json).
  - Anthropic renamed the Claude Code SDK to the Claude Agent SDK because the harness "powers many non-coding agents" — [Claude blog, 2025-09-29](https://claude.com/blog/building-agents-with-the-claude-agent-sdk).
  - The SDK is at 0.3.296 (2026-10-09) — [npm](https://registry.npmjs.org/@anthropic-ai/claude-agent-sdk).
- **Mode 5, ACP**:
  - Zed, Toad, Emdash's chat path, Happy (`happy acp <agent>`) and Gas City (ACP provider) drive agents over ACP — [Zed docs](https://github.com/zed-industries/zed/blob/main/docs/src/ai/external-agents.md), [Toad](https://github.com/batrachianai/toad), [Emdash AGENTS.md](https://github.com/generalaction/emdash/blob/main/AGENTS.md), [happy-cli](https://github.com/slopus/happy/blob/main/packages/happy-cli/README.md), [Gas City](https://github.com/gastownhall/gascity).
  - The Claude adapter wraps the Agent SDK and has been renamed twice in 2026 — [npm](https://registry.npmjs.org/@agentclientprotocol/claude-agent-acp).
- **Mode 6, first-party control**:
  - `claude --bg "<prompt>"` is rejected when combined with `-p` "because `--print` never starts the interactive session that `claude agents` attaches to". Background sessions are full interactive sessions hosted by the supervisor.
  - `claude agents --json` exposes `state` (working|blocked|done|failed|stopped), `status` (busy|waiting|idle) and `waitingFor` ("permission prompt", "input needed", "sandbox request", "worker request", "dialog open").
  - Source: [agent-view doc](https://code.claude.com/docs/en/agent-view).
  - A script can post into a session over its inbox socket. Verification of "own-child" messages differs by OS — [cross-session doc](https://code.claude.com/docs/en/cross-session-messaging).
- **Subscription vs API (architecture-relevant facts; the terms researcher covers the details)**. These come from secondary reports:
  - Anthropic deployed server-side checks on 2026-01-09 that blocked third-party tools from using Pro/Max OAuth — [VentureBeat](https://venturebeat.com/technology/anthropic-cracks-down-on-unauthorized-claude-usage-by-third-party-harnesses).
  - In Feb 2026 its terms reportedly limited subscription OAuth to Claude Code and Claude.ai, calling use "including the Agent SDK" unauthorized — [Gigazine](https://gigazine.net/gsc_news/en/20260220-anthropic-third-party-block).
  - OpenCode removed Anthropic OAuth on 2026-03-19 after legal demands — [falcao.org](https://falcao.org/posts/anthropic-claude-access-crackdown-ecosystem-fallout/).
  - From 2026-04-04, third-party harnesses (e.g. OpenClaw) reportedly need pay-as-you-go API billing — [lilting.ch](https://lilting.ch/en/articles/anthropic-claude-code-openclaw-third-party-paygo).
  - Agent Deck's guidance: it "runs the unmodified `claude` CLI with your own login and never reads, copies or reuses a login token". For "heavy many-session or unattended orchestration use an API key or a Team/Enterprise plan". `claude setup-token` provides long-lived tokens (`CLAUDE_CODE_OAUTH_TOKEN`) — [Agent Deck README](https://github.com/asheshgoplani/agent-deck).
  - Remote Control requires a claude.ai subscription; "API keys are not supported" — [remote-control doc](https://code.claude.com/docs/en/remote-control).

### Inferences
- **Fidelity.** Modes 1, 2 and 6 run the real interactive Claude Code, so slash commands, plan mode, agent teams, `/loop` and Remote Control all work. Modes 3 to 5 lose some of these; agent teams, for example, do not spawn in `-p`/SDK.
- **"Needs input" detection, ranked best to worst:**
  1. Structured protocol events: app-server approval requests, ACP `requires_action`, OpenCode permission IDs.
  2. First-party `claude agents --json` `waitingFor`.
  3. Hooks (Notification/Stop-type events). These push in real time but lack detail; cmux's author complains the notification body is "always just 'Claude is waiting for your input'".
  4. Scraping pane text. This breaks on UI copy changes, which are likely given about 300 Claude Code releases in 9 months.
- **Resumability.** Every mode can resume via the Claude session ID (`--resume`). Background sessions add process supervision, and stream-json modes need the harness to persist the session ID itself, as Vibe Kanban does.
- **Permissions.** Headless and SDK modes route approvals to the harness through `--permission-prompt-tool=stdio`, `canUseTool` or ACP `session/request_permission`, which suits rich UI cards. In interactive modes, approvals happen in the TUI, which is fine when Narwarp is the terminal showing it.
- **Billing.** For a single user on a Max plan, the lowest-friction legitimate path is to run the unmodified interactive `claude` CLI (modes 1, 2, 6). Headless `--bare` and Agent-SDK embedding should be assumed to need API keys. This is an inference from the reported terms and the `--bare` docs; confirm with the terms researcher.

### Gaps
- Whether non-bare `claude -p` with subscription OAuth stays within Anthropic's consumer terms once launched by a third-party harness is not settled in the primary docs reviewed here. The terms researcher owns this.
- Practitioner reports of specific breakages when Claude Code UI strings changed (e.g., Claude Squad issues) were not retrieved, because GitHub issue search was blocked.

---

## Q3. How do they isolate parallel agents (worktrees, containers, VMs), name branches, clean up, and handle merge conflicts?

### Takeaway
A git worktree per task is the near-universal default locally. Containers are optional and mostly a security add-on. VMs or sandboxes are the cloud default. Tools now say plainly that worktrees separate files, not processes, permissions or semantic conflicts. Cleanup needs explicit sweeps and locks. Merge conflicts are handled in one of four ways:
- partitioning work up front,
- an agent-run merge queue (Gas Town's bisecting Refinery),
- letting agents resolve their own rebases (Anthropic's C compiler), or
- human review with an agent assisting.

### Cited Findings
- **Worktree defaults and naming**:
  - Claude Code `--worktree <name>` creates `.claude/worktrees/<name>/` on branch `worktree-<name>`. Background sessions move themselves into a worktree "before editing files". `/batch` splits a change into "5 to 30 worktree-isolated subagents" — [worktrees doc](https://code.claude.com/docs/en/worktrees), [agent-view doc](https://code.claude.com/docs/en/agent-view), [agents doc](https://code.claude.com/docs/en/agents).
  - Sculptor branches follow `<user>/<slug>` — [Sculptor workspaces](https://github.com/imbue-ai/sculptor/blob/main/docs/help/workspaces.md).
  - Conductor names workspaces after cities as "a stable name for the workspace directory" — [Conductor docs](https://www.conductor.build/llms-full.txt).
  - Agent Deck offers sibling (`repo-branch`), subdirectory (`repo/.worktrees/branch`) or a custom path template. It supports bare-repo layouts and sparse-checkout inheritance — [Agent Deck README](https://github.com/asheshgoplani/agent-deck).
  - Symphony sanitizes workspace keys to `[A-Za-z0-9._-]`, adds a hash suffix on collision, and requires `cwd == workspace_path` inside the workspace root before launch — [Symphony SPEC](https://github.com/openai/symphony/blob/main/SPEC.md).
  - Remote Control server mode has `--spawn worktree` — [remote-control doc](https://code.claude.com/docs/en/remote-control).
- **Environment bootstrap**:
  - `.worktreeinclude` copies selected gitignored files (Claude Code semantics, also used by Agent Deck) — [Agent Deck README](https://github.com/asheshgoplani/agent-deck).
  - Conductor uses setup scripts, run scripts and `CONDUCTOR_PORT` per-workspace port ranges — [Conductor docs](https://www.conductor.build/llms-full.txt).
  - Superset gives every worktree its own detected ports for previews — [Superset README](https://github.com/superset-sh/superset).
  - Symphony has `after_create`, `before_run`, `after_run` and `before_remove` hooks with a 60 s default timeout — [SPEC](https://github.com/openai/symphony/blob/main/SPEC.md).
  - Agent Deck setup and destruction scripts need sha256-bound approval because "These two hooks are code from the repository, and they run as you" — [Agent Deck README](https://github.com/asheshgoplani/agent-deck).
- **Isolation is not a security boundary**:
  - "Workspace isolation is development isolation, not a security boundary" — [Conductor docs](https://www.conductor.build/llms-full.txt).
  - "Worktrees separate working files; they do not sandbox processes or prevent merge conflicts." — [Superset README](https://github.com/superset-sh/superset).
  - Claude Code adds enforcement: in a worktree it "blocks the tool calls" that escape it and asks approval to enter paths outside `.claude/worktrees/` — [worktrees doc](https://code.claude.com/docs/en/worktrees).
- **Containers and VMs**:
  - Agent Deck can run a session in a Docker sandbox with the project bind-mounted read-write. The macOS Keychain Claude login is the exception that can't be shared — [Agent Deck README](https://github.com/asheshgoplani/agent-deck).
  - Sculptor's container backend is experimental. On macOS, Claude credentials in the Keychain are inaccessible from containers, so `claude setup-token` is recommended — [Sculptor container doc](https://github.com/imbue-ai/sculptor/blob/main/docs/help/experimental/container_backend.md).
  - Container Use gives a container per agent plus a git branch. STALE? — [README](https://github.com/dagger/container-use).
  - Anthropic's C compiler run gave each agent a Docker container that cloned from a bare upstream repo mounted at `/upstream` and pushed back. Their advice: "Run this in a container, not your actual machine" — [Anthropic](https://www.anthropic.com/engineering/building-c-compiler).
  - Cloud options:
    - Claude Code on the web VMs with a GitHub proxy — [web doc](https://code.claude.com/docs/en/claude-code-on-the-web).
    - Conductor Cloud on Vercel sandboxes — [Conductor docs](https://www.conductor.build/llms-full.txt).
    - Warp Oz Docker sandboxes — [Warp](https://warp.dev/newsroom/2026/2/10/warp-launches-oz-the-orchestration-platform-for-cloud-coding-agents).
    - Terragon per-task sandboxes — [terragon-oss](https://github.com/terragon-labs/terragon-oss).
- **Cleanup**:
  - Claude Code prompts on exit when the worktree has changes or commits. A periodic sweep removes Claude-created subagent and background-session worktrees older than `cleanupPeriodDays`. It holds a `git worktree lock` while agents run and writes a marker into git metadata so it never deletes user-made worktrees. Deleting a background session deletes its worktree, so commit first — [worktrees doc](https://code.claude.com/docs/en/worktrees), [agent-view doc](https://code.claude.com/docs/en/agent-view).
  - Agent Deck has `worktree finish` (merge, remove, delete session) and `worktree cleanup` (orphans) — [Agent Deck README](https://github.com/asheshgoplani/agent-deck).
  - Vibe Kanban cleans "orphan and expired" workspaces — [README](https://github.com/BloopAI/vibe-kanban).
  - Symphony does "Startup terminal workspace cleanup" for issues already in terminal states — [SPEC](https://github.com/openai/symphony/blob/main/SPEC.md).
- **Merge conflicts**:
  - Gas Town's Refinery: "Polecat runs `gt done` → branch pushed, MR bead created → Refinery batches → runs verification gates on the merged stack → if green merge all; if red bisect to isolate the failing MR ... polecats never push directly to main" — [Gas Town README](https://github.com/steveyegge/gastown).
  - C compiler: "Merge conflicts are frequent, but Claude is smart enough to figure that out." Kernel compilation could not be parallelized naively ("Every agent would hit the same bug, fix that bug, and then overwrite each other's changes") until GCC was used as an oracle to split work by file — [Anthropic](https://www.anthropic.com/engineering/building-c-compiler).
  - Agent teams don't isolate teammates in worktrees. "Two teammates editing the same file leads to overwrites", so partition files — [agent-teams doc](https://code.claude.com/docs/en/agent-teams), [agents doc](https://code.claude.com/docs/en/agents).
  - Conductor: "If the branch has conflicts, ask an agent to help resolve them, then rerun tests" — [Conductor docs](https://www.conductor.build/llms-full.txt).
  - Xum and Superset surface "git divergence" UIs — [Xum README](https://github.com/coder/mux).
  - Practitioner: in a ten-agent project "three of the ten agents still emitted lint-broken code" and every conflict was resolved by hand — [Bernstein blog](https://bernstein.readthedocs.io/en/latest/blog/ten-agents-one-release/) (practitioner).

### Inferences
- For Narwarp, the user's own repos and a single macOS machine, a reasonable local design is worktree-per-ticket by default, with containers opt-in for untrusted repos. Reuse Claude Code's own worktree machinery (`--worktree`, background auto-isolation, sweep and lock) rather than writing a parallel worktree manager. Otherwise there are two garbage collectors that don't know about each other.
- A merge strategy that fits one human reviewer: PR-per-ticket with a human gate (Symphony's "Human Review" handoff state). A Refinery-style queue is optional, for "fun" repos where auto-merge is acceptable.

### Gaps
- Systematic data on how often parallel worktree agents produce semantic conflicts was not found. Evidence is anecdotal.

---

## Q4. How do they provide observability and control (transcripts, live streaming, status, notifications, steering, cancellation, cost)?

### Takeaway
State models are converging on working / blocked (needs input, with a reason) / done / failed / stopped. Live data arrives by:
- push: SSE, JSON-RPC notifications, hooks, OSC terminal notifications;
- poll: `claude agents --json`, pane hashing.

Steering is now first-class in protocols (Codex `turn/steer`, Claude peek-reply and inbox sockets, OpenCode `prompt_async`). Cost is usually reconstructed from transcripts or result messages.

### Cited Findings
- **Status models**:
  - Claude Code background sessions: `working`, `blocked`, `done`, `failed`, `stopped`, plus `status` and `waitingFor`. "A session that finished its turn ... reads `done`, not `blocked`." — [agent-view doc](https://code.claude.com/docs/en/agent-view).
  - Agent Deck: Running, Waiting, Idle, Error — [README](https://github.com/asheshgoplani/agent-deck).
  - Gas Town problems view: "GUPP Violation" (hooked work, no progress), "Stalled", "Zombie" ("Dead tmux session"), "Working", "Idle" — [README](https://github.com/steveyegge/gastown).
  - ACP subagents: `running`, `requires_action`, `idle` — [claude-agent-acp](https://github.com/zed-industries/claude-agent-acp).
  - Conductor API lesson: "Wait for `working` before trusting `idle`. A queued prompt hasn't started a turn yet, and the session reports `idle` until it does." — [Conductor docs](https://www.conductor.build/llms-full.txt).
- **Live streams and transcripts**:
  - OpenCode `GET /event` (SSE) — [OpenCode server docs](https://github.com/anomalyco/opencode/blob/master/packages/web/src/content/docs/server.mdx).
  - Codex `item/*` and `turn/*` notifications, with threads as JSONL logs — [app-server docs](https://learn.chatgpt.com/docs/app-server).
  - Claude stream-json events — [headless doc](https://code.claude.com/docs/en/headless).
  - Gas Town `.events.jsonl` and OTEL logs/metrics (e.g. `gastown.session.starts.total`) — [Gas Town README](https://github.com/steveyegge/gastown).
  - Agent view row summaries come from "a Haiku-class model", updated at most every 15 s from output without a model call, plus a model rewrite at turn end. Each rewrite is "billed" — [agent-view doc](https://code.claude.com/docs/en/agent-view).
- **Notifications**:
  - Agent view uses the configured terminal notification channel when a background session needs input, finishes or fails — [agent-view doc](https://code.claude.com/docs/en/agent-view).
  - cmux has blue "notification rings" from OSC 9/99/777 and a notification panel — [cmux README](https://github.com/manaflow-ai/cmux).
  - Remote Control has mobile push — [remote-control doc](https://code.claude.com/docs/en/remote-control).
  - Agent Deck uses the tmux status bar plus Telegram/Slack bridges — [README](https://github.com/asheshgoplani/agent-deck).
  - Nimbalyst and Superset have iOS apps — [Nimbalyst](https://github.com/Nimbalyst/nimbalyst), [Superset](https://github.com/superset-sh/superset).
- **Steering mid-run**:
  - Codex `turn/steer` "adds input to the in-flight turn without starting a new one" — [app-server docs](https://learn.chatgpt.com/docs/app-server).
  - Claude Code agent view "peek and reply" — [agent-view doc](https://code.claude.com/docs/en/agent-view).
  - `claude -p "msg" --cloud <id>` queues a message into a cloud session — [web doc](https://code.claude.com/docs/en/claude-code-on-the-web).
  - `SendMessage` and the inbox socket work across local sessions — [cross-session doc](https://code.claude.com/docs/en/cross-session-messaging).
  - Agent-teams direct messaging — [agent-teams doc](https://code.claude.com/docs/en/agent-teams).
  - OpenCode `POST /session/:id/prompt_async` — [OpenCode docs](https://github.com/anomalyco/opencode/blob/master/packages/web/src/content/docs/server.mdx).
- **Cancellation**:
  - Codex `turn/interrupt` (status "interrupted") — [app-server docs](https://learn.chatgpt.com/docs/app-server).
  - ACP `session/cancel` — [ACP](https://agentclientprotocol.com/protocol/overview).
  - OpenCode `/session/:id/abort` — [OpenCode docs](https://github.com/anomalyco/opencode/blob/master/packages/web/src/content/docs/server.mdx).
  - `claude stop <id>`, or Ctrl+X in agent view — [agent-view doc](https://code.claude.com/docs/en/agent-view).
  - Symphony stops runs when tracker state becomes terminal or ineligible ("CanceledByReconciliation") — [SPEC](https://github.com/openai/symphony/blob/main/SPEC.md).
- **Cost and usage**:
  - Claude `--output-format json` includes `total_cost_usd` and a per-model breakdown — [headless doc](https://code.claude.com/docs/en/headless).
  - Agent Deck's cost dashboard (`agent-deck costs sync` "backfills cost data from existing Claude transcript files") — [README](https://github.com/asheshgoplani/agent-deck).
  - Xum's cost and token UI — [README](https://github.com/coder/mux).
  - Claude Code Router logs "tokens, latency, ... cost estimates" per request — [CCR README](https://github.com/musistudio/claude-code-router).
  - Symphony spec §13.5 "Session Metrics and Token Accounting" — [SPEC](https://github.com/openai/symphony/blob/main/SPEC.md).
  - Anthropic says background sessions "consume your subscription usage the same as interactive sessions, so running ten agents in parallel uses quota roughly ten times as fast" — [agent-view doc](https://code.claude.com/docs/en/agent-view).

### Inferences
- Narwarp should normalize every agent source into one internal event model. Codex's Thread/Turn/Item is a clean, documented template. Add a status enum that mirrors Claude's `state` plus `waitingFor`. Sources: Claude hooks, `claude agents --json`, stream-json, OpenCode SSE, ACP.
- A per-ticket and per-session cost ledger can be built without API billing by parsing local Claude transcripts, as Agent Deck does. Exact dollar figures matter less on a subscription than quota burn rate.

### Gaps
- No primary source documents a stable push API (as opposed to polling `--json`) for Claude Code background-session state. Hooks are the push path.

---

## Q5. How do they handle session persistence (reboot, long-running background work, scheduling) and context exhaustion?

### Takeaway
- **Process persistence** comes from a daemon or tmux server outliving the UI. Nobody restores live processes across a machine shutdown. Restart means resuming from a transcript or session ID, ideally automatically via hooks that recorded the native session ID.
- **Durable work state** lives outside the agent: trackers, Beads, feature-list JSON, progress files, session event logs.
- **Context exhaustion** handling moved from forced context resets toward compaction plus external notes as models improved (Opus 4.5 and later). Fresh-session-with-handoff remains the pattern for ephemeral workers.

### Cited Findings
- **Claude Code supervisor lifecycle**:
  - Idle unattended sessions are stopped after "about an hour" and resume on attach. Crashed sessions are restarted. On auto-update the supervisor restarts onto the new version. If the supervisor dies, sessions wait "about a minute" to reconnect.
  - "Sessions are also preserved when your machine sleeps", but "shutting down still stops running sessions", which recover from the saved conversation. `claude respawn --all` restarts every session on a new binary.
  - Source: [agent-view doc](https://code.claude.com/docs/en/agent-view).
- **Agent teams.** "`/resume` and `/rewind` do not restore in-process teammates." The team config is removed at session end, while the task list persists — [agent-teams doc](https://code.claude.com/docs/en/agent-teams).
- **Remote Control.** Server sessions can be revived for "about four hours". After a 10-minute outage, server mode exits — [remote-control doc](https://code.claude.com/docs/en/remote-control).
- **Daemons and reboots**:
  - Happy recommends starting its daemon from the shell profile so it "inherits your normal user session context (PATH, keychain access, OAuth credentials)" — [happy-cli README](https://github.com/slopus/happy/blob/main/packages/happy-cli/README.md).
  - Superset's terminal daemon survives app restarts and updates, and its docs suggest a reboot only if terminals stay broken — [Superset docs](https://docs.superset.sh/llms-full.txt).
  - cmux local-tmux survives cmux quit, crash and update, with "machine-sleep limits" documented — [cmux README](https://github.com/manaflow-ai/cmux).
  - Gas Town uses `gt up --restore` — [Gas Town README](https://github.com/steveyegge/gastown).
- **macOS specifics for background hosts**:
  - The background session host "requests access to protected folders separately from your terminal", needing Files and Folders or Full Disk Access grants.
  - macOS 15+ requires Local Network permission for LAN access from background sessions.
  - Source: [agent-view doc](https://code.claude.com/docs/en/agent-view).
- **Durable work outside sessions**:
  - Symphony recovers from the tracker and filesystem with no database: "Restart recovery is tracker-driven and filesystem-driven" — [SPEC](https://github.com/openai/symphony/blob/main/SPEC.md).
  - Gas Town keeps "persistent identity but ephemeral sessions", git-worktree "hooks", and `gt seance` to query predecessors — [Gas Town README](https://github.com/steveyegge/gastown).
  - Codex threads persist as JSONL with `thread/resume` and `thread/fork` — [app-server docs](https://learn.chatgpt.com/docs/app-server).
  - OpenCode has session `fork`, `revert` and `summarize` endpoints — [OpenCode docs](https://github.com/anomalyco/opencode/blob/master/packages/web/src/content/docs/server.mdx).
- **Scheduling**:
  - Claude Code: `/loop` and cron tools inside a session, Desktop scheduled tasks, and cloud routines — [remote-control doc table](https://code.claude.com/docs/en/remote-control), [llms.txt](https://code.claude.com/docs/llms.txt).
  - Codex app Automations land in a review queue — [AI Pulse](https://artificialintelligencepulse.it.com/2026/02/03/codex-on-macOS/).
  - Superset Automations ("triage issues overnight") — [Superset README](https://github.com/superset-sh/superset).
  - Gas Town's daemon heartbeat runs every 3 minutes and the Deacon patrols — [Gas Town README](https://github.com/steveyegge/gastown).
  - Symphony polls on `polling.interval_ms` — [SPEC](https://github.com/openai/symphony/blob/main/SPEC.md).
  - Agent Deck watchers react to events (gmail, meetings, GitHub) — [README](https://github.com/asheshgoplani/agent-deck).
- **Context exhaustion**:
  - Anthropic: with Sonnet 4.5, "context anxiety" made resets essential. "Opus 4.5 largely removed this behavior, so the team dropped resets and relied on automatic compaction" — [harness-design post, 2026-03-24](https://www.anthropic.com/engineering/harness-design-long-running-apps).
  - Managed Agents: "The resets had become dead weight". The session log serves as "a context object outside Claude's window" because compaction is irreversible — [Managed Agents post, 2026-04-08](https://www.anthropic.com/engineering/managed-agents).
  - The long-running-agent harness relies on `claude-progress.txt`, a JSON feature list where agents may only flip `passes`, `init.sh`, git history and a start-of-session "get bearings" ritual — [Anthropic, 2025-11-26](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents).
  - Agent Deck's conductor proactive `/clear` (`clear_on_compact`) "only arms on an established window" and otherwise falls back to Claude's compaction — [README](https://github.com/asheshgoplani/agent-deck).
  - Sculptor hooks pre-compaction and shows "% context" — [Sculptor doc](https://github.com/imbue-ai/sculptor/blob/main/docs/help/integrated_harnesses.md).
  - Symphony keeps continuation turns on the same live thread and sends "only continuation guidance", not the original prompt — [SPEC](https://github.com/openai/symphony/blob/main/SPEC.md).
  - Critique of fresh-session workers: "each task starts a fresh session that doesn't remember earlier steps, so agents re-read context and miss connections" — [Every](https://every.to/context-window/inside-the-100-agent-software-factory).

### Inferences
- On macOS, Narwarp needs:
  - a LaunchAgent (or shell-profile bootstrap, as Happy does) for its own daemon, so it has keychain and PATH context;
  - a ticket→session-ID map in its own store, so after reboot it can re-spawn with `claude --resume <id> --bg` or `claude respawn`;
  - acceptance that work in flight at shutdown is restarted from transcripts, not continued.
- Treat the ticket plus its message board (owned by other researchers) as the durable "session log" à la Managed Agents. Agent sessions then become disposable.

### Gaps
- No primary evidence that any local harness auto-resumes agent sessions after a full macOS reboot without user action. cmux and Emdash resume via hooks when the app relaunches, which is closest.

---

## Q6. How do harnesses integrate heterogeneous agents (Codex, Gemini CLI, OpenCode, local Ollama models), and what router or dispatcher patterns exist?

### Takeaway
Heterogeneity happens at four layers:
1. **Launch-command profiles.** Any CLI runs in a PTY; this is the most common approach.
2. **Protocol adapters.** ACP registry agents, Codex app-server, OpenCode server.
3. **Model gateways under one harness.** Claude Code → Ollama's Anthropic-compatible endpoint or Claude Code Router.
4. **Harness-agnostic runtimes with their own loop.** OpenCode, Xum, Happy Agent, Pi.

Dispatchers come in two kinds:
- **Deterministic control loops**: Symphony, the Gas City controller, schedulers with concurrency caps.
- **LLM coordinators**: Gas Town Mayor, Agent Deck conductor, Claude agent-teams lead, research lead agent.

A third, leaderless file-lock style also exists (Anthropic's C compiler).

### Cited Findings
- **Profiles and presets**:
  - Claude Squad `profiles` (e.g. `aider --model ollama_chat/gemma3:1b`) — [README](https://github.com/smtg-ai/claude-squad).
  - Gas Town built-in presets `claude, gemini, codex, kiro, cursor, auggie, amp, opencode, copilot, pi, omp`, `gt sling <bead> <rig> --agent cursor` per-dispatch overrides, and `gt config agent set claude-glm "claude-glm --model glm-4"` — [README](https://github.com/steveyegge/gastown).
  - Superset lists 20+ supported CLI agents, and "Any other CLI agent ... Works without configuration" — [README](https://github.com/superset-sh/superset).
  - Emdash "detects installed provider CLIs automatically" — [README](https://github.com/generalaction/emdash).
  - Conductor "runs each chat through one harness" (Claude Code, Codex, OpenCode, Cursor) — [docs](https://www.conductor.build/llms-full.txt).
- **Protocol adapters**:
  - Zed's ACP Registry (Claude, Codex, Gemini CLI, OpenCode, Copilot, Cursor, Pi, Poolside) — [Zed docs](https://github.com/zed-industries/zed/blob/main/docs/src/ai/external-agents.md).
  - Toad's "AI App store" via ACP — [Toad](https://github.com/batrachianai/toad).
  - `happy acp opencode` / `happy acp -- custom-agent` — [happy-cli](https://github.com/slopus/happy/blob/main/packages/happy-cli/README.md).
  - Gas City runtime providers, including ACP and Kubernetes — [Gas City](https://github.com/gastownhall/gascity).
  - Sculptor's "Pi Integrated Harness" in RPC mode, plus "any terminal-based agents" — [Sculptor](https://github.com/imbue-ai/sculptor/blob/main/docs/help/integrated_harnesses.md).
- **Ollama and local models**:
  - Ollama supports the Anthropic Messages API (v0.14.0+). Claude Code connects with `ANTHROPIC_BASE_URL=http://localhost:11434` and `ANTHROPIC_AUTH_TOKEN=ollama`, and `ollama launch claude` configures and launches it. Recommended coding models include glm-4.7, minimax-m2.1 and qwen3-coder — [Ollama docs](https://docs.ollama.com/api/anthropic-compatibility), [Ollama blog](https://registry.ollama.ai/blog/claude).
  - A secondary source recommends at least 64k context — [glukhov.org](https://glukhov.org/ai-devtools/claude-code/).
  - A hands-on test with v0.15 found small local models "misread a simple instruction" under Claude Code — [classmethod](https://dev.classmethod.jp/en/articles/claude-code-ollama-local/) (practitioner).
  - Xum supports Ollama directly with its own agent loop — [Xum README](https://github.com/coder/mux).
  - Agent Deck runs conductors against custom Anthropic-compatible endpoints via `-env ANTHROPIC_BASE_URL=...` (example: z.ai GLM) — [README](https://github.com/asheshgoplani/agent-deck).
- **Model gateways**:
  - Claude Code Router is "a local model gateway and control plane for coding agents". It gives Claude Code, Codex, OpenCode, Pi and others "one stable local endpoint" (`http://127.0.0.1:3456`) with routing, failover and per-request logs/tokens/cost. v3.1.3 shipped 2026-10-09 — [CCR README](https://github.com/musistudio/claude-code-router), [npm](https://registry.npmjs.org/@musistudio/claude-code-router).
- **First-party stance.** "In every approach the workers are Claude sessions. To involve a different tool, expose it to Claude as an MCP server." — [Claude Code agents doc](https://code.claude.com/docs/en/agents).
- **Dispatcher patterns**:
  - *Deterministic*: Symphony's orchestrator "is the only component that mutates scheduling state". It sorts candidates by priority, then `created_at`, then identifier, enforces global and per-state concurrency caps, and retries with backoff `min(10000 * 2^(attempt-1), max_retry_backoff_ms)` (default cap 5 min) — [SPEC](https://github.com/openai/symphony/blob/main/SPEC.md).
  - Gas City "reconciles desired state to running state" — [Gas City](https://github.com/gastownhall/gascity).
  - Gas Town's scheduler "prevents API rate limit exhaustion by batching dispatch under configurable concurrency limits" — [Gas Town](https://github.com/steveyegge/gastown).
  - *LLM coordinator*: "The Mayor in Gas Town never writes code" — [Maggie Appleton](https://maggieappleton.com/gastown).
  - Agent Deck conductors "auto-respond when confident, and escalate to you when they can't". Watchers are "triggers, not launchers" — [Agent Deck README](https://github.com/asheshgoplani/agent-deck).
  - Agent-teams lead assigns tasks or teammates self-claim — [agent-teams doc](https://code.claude.com/docs/en/agent-teams).
  - Gas City's "One pet, many cattle": one persistent supervisor, disposable workers — [Every](https://every.to/context-window/inside-the-100-agent-software-factory).
  - *Leaderless*: "I don't use an orchestration agent". Agents claim work via lock files in `current_tasks/` — [Anthropic C compiler](https://www.anthropic.com/engineering/building-c-compiler).
  - *Multi-model review*: Gas City setups have "multiple models (Claude, Codex, Kimi) review the same code" — [Every](https://every.to/context-window/inside-the-100-agent-software-factory).

### Inferences
- For Narwarp, the simplest robust heterogeneity is two lanes.
  - **Claude lane**: interactive `claude` sessions on the subscription.
  - **Local lane**: Ollama-backed agents run either as Claude Code pointed at Ollama in a separate `CLAUDE_CONFIG_DIR`/profile (same UX and hooks, weaker models), or through OpenCode's server (an HTTP+SSE API that is easy to drive from Rust or Python).
- Routing between lanes should be a deterministic Narwarp policy, Symphony-style. Rules could key on ticket labels, privacy (life-mode tickets → local), cost or quota, and task type (triage/summarize → local; code changes → Claude). An optional LLM "conductor" ticket-triager would sit on top. Not an LLM router in the request path.
- Put a gateway like Claude Code Router in front of Claude only if per-request model routing is needed. It adds a moving part and may interact with subscription auth. Unverified.

### Gaps
- OpenCode's Ollama provider configuration wasn't fetched this session; it is expected but unverified here. Codex's local/OSS-model mode (Ollama) was not verified this session.
- No benchmark comparing local Ollama models under Claude Code vs OpenCode was found.

---

## Q7. What do Anthropic's and OpenAI's engineering write-ups say about harness design? (principles)

### Takeaway
Both labs converge on four ideas:
- The harness is the product, and it encodes assumptions that go stale as models improve, so keep it minimal and re-test which components are load-bearing.
- Externalize durable state (logs, feature lists, progress files, trackers, the repo itself as system of record).
- Separate durable session state from stateless "brains" and disposable "hands".
- Invest in verification oracles and agent-legible environments rather than more agents.

Multi-agent setups pay off for breadth-first or parallelizable work at large token multiples.

### Cited Findings
- **Multi-agent research system** (Anthropic, 2025-06-13; STALE? but foundational) — [Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system):
  - Orchestrator-worker with an Opus 4 lead and Sonnet 4 subagents. It "outperformed single-agent Claude Opus 4 by 90.2%".
  - "token usage by itself explains 80% of the variance". Agents use ~4× chat tokens and multi-agent systems ~15×.
  - "Agents are stateful and errors compound". The answers: resume from checkpoints, "rainbow deployments", full tracing.
  - Synchronous subagent execution is a bottleneck.
  - The plan is saved to memory because context beyond 200k is truncated.
  - Subagents write to external storage and "pass lightweight references back".
  - Delegation needs "an objective, an output format, guidance on the tools and sources to use". An early failure was "spawning 50 subagents for simple queries".
- **Building agents with the Claude Agent SDK** (2025-09-29, Thariq Shihipar) — [Claude blog](https://claude.com/blog/building-agents-with-the-claude-agent-sdk):
  - "gather context -> take action -> verify work -> repeat"; "give your agents a computer".
  - Start with agentic search (filesystem, grep) before semantic search.
  - Subagents give parallelism and context isolation, plus automatic compaction.
  - Verification by rules, then visual feedback, then LLM-as-judge (the last "generally not a very robust method").
- **Effective harnesses for long-running agents** (2025-11-26, Justin Young) — [Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents):
  - An initializer agent plus a coding agent, where "the two differ only in their initial user prompts".
  - A JSON feature list of 200+ items "all initially marked failing", chosen because "the model is less likely to inappropriately change or overwrite JSON files" than Markdown. Plus `claude-progress.txt`, `init.sh` and git commits.
  - Failure modes and fixes: one-shotting, premature victory, broken or undocumented state, and inadequate testing (fixed with Puppeteer MCP).
  - "still unclear whether a single, general-purpose coding agent performs best".
- **Building a C compiler with parallel Claudes** (2026-02-05, Nicholas Carlini; STALE?) — [Anthropic](https://www.anthropic.com/engineering/building-c-compiler):
  - 16 agents, no orchestrator, Docker per agent, lock files, a bare git upstream.
  - "nearly 2,000 Claude Code sessions" over two weeks, ~2B input and 140M output tokens, "just under $20,000".
  - "the task verifier is nearly perfect, otherwise Claude will solve the wrong problem".
  - Avoid context pollution: print only a few lines and log to files.
  - "time blindness": add a `--fast` sampling option.
  - Specialized agents for dedup, performance and docs.
- **Harness design for long-running application development** (2026-03-24, Prithvi Rajasekaran) — [Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps):
  - Planner, generator and evaluator (Playwright MCP) agents communicating "through files". Sprint contracts at first, dropped later.
  - Solo run: 20 min, $9, broken. Full harness on Opus 4.5: 6 h, $200. Updated harness on Opus 4.6: 3 h 50 min, $124.70.
  - "every component in a harness encodes an assumption about what the model can't do on its own"; "find the simplest solution possible"; remove components one at a time.
- **Scaling Managed Agents: decoupling the brain from the hands** (2026-04-08, Lance Martin, Gabe Cemaj, Michael Cohen) — [Anthropic](https://www.anthropic.com/engineering/managed-agents):
  - Components: Session (append-only log), Harness/brain (loop), Sandbox/hands. Interfaces: `provision`, `execute(name, input) → string`, `wake(sessionId)`, `getSession`, `emitEvent`, `getEvents`.
  - A container used to be a "pet". Now a sandbox failure is a tool error, and harness failure is handled by `wake` from the log.
  - "Harnesses encode assumptions that go stale as models improve."
  - Credentials are "never reachable from the sandbox" (vault plus proxy).
  - TTFT fell ~60% at p50 and >90% at p95 because inference starts before containers are ready.
  - Managed Agents is a "meta-harness" neutral to the harness it runs, Claude Code included.
- **Unlocking the Codex harness: App Server** (OpenAI, 2026-02-04, Celia Chen) — [OpenAI](https://openai.com/index/unlocking-the-codex-harness/), via search snippets:
  - One harness behind all surfaces through a bidirectional JSON-RPC API.
  - Low-level events are converted into "a small set of UI-ready notifications".
  - Approvals are server-initiated requests that pause the turn.
- **Harness engineering: leveraging Codex in an agent-first world** (OpenAI, 2026-02-11, Ryan Lopopolo) — [OpenAI](https://openai.com/index/harness-engineering/), via search snippets:
  - An internal product with "zero manually written lines of code", "a million lines of code", built in ~1/10th the time.
  - "Humans steer. Agents execute."
  - The engineer's job becomes "designing environments, specifying intent, and building feedback loops".
  - A secondary write-up says AGENTS.md acted as a table of contents into docs, specs and plans that "must stay current" — [Habr](https://habr.com/en/articles/1063504/) (secondary).
  - Symphony "works best in codebases that have adopted harness engineering" — [Symphony README](https://github.com/openai/symphony).
- **Claude Code docs guidance on team size.** "Start with 3-5 teammates"; "Three focused teammates often outperform five scattered ones"; teams "use significantly more tokens" — [agent-teams doc](https://code.claude.com/docs/en/agent-teams).

### Inferences
Principles for Narwarp, synthesized:
1. Make the ticket and its board the durable session log. Agents are replaceable workers that are `wake`d with context assembled from it.
2. Keep credentials (Linear, Gmail, Discord, Google Calendar tokens) out of agent processes. Expose them through Narwarp-side tools or proxies, following the Managed Agents vault/proxy and Symphony's `linear_graphql` boundary (see Help Net Security).
3. Encode verification per project (`init.sh`, tests, `WORKFLOW.md`) before scaling the number of agents.
4. Default to one agent per ticket. Use teams or subagents only for breadth tasks.
5. Re-audit harness components at each model upgrade.

### Gaps
- The full text of OpenAI's two February 2026 posts could not be fetched (HTTP 403). Quantitative details beyond the search snippets, such as engineer and PR counts, are unverified.

---

## Q8. What failure modes and lessons have been reported (zombie sessions, runaway costs, review bottlenecks, too much parallelism, brittle TUI scraping)?

### Takeaway
The dominant real-world limits are human review and design bandwidth, not agent throughput. Practitioners converge on 3–5 concurrent agents. Next come:
- token and quota burn, scaling linearly or worse with agent count and with multi-step pipelines;
- lifecycle debris: orphaned tmux sessions, worktrees and orphan sessions;
- stalls and "needs input" blindness;
- coupling to fast-changing TUI text.

### Cited Findings
- **Review and design bottlenecks (practitioner)**:
  - Simon Willison: "the natural bottleneck on all of this is how fast I can review the results" — [simonwillison.net](https://simonwillison.net/2025/Oct/5/parallel-coding-agents/).
  - A Codex worktree guide suggests 3–4 parallel sessions because "review and merge become the bottleneck" — [Daniel Vaughan](https://codex.danielvaughan.com/2026/03/26/codex-cli-worktree-parallel-development/).
  - "if you're approving agent output without reading it, you've exceeded your cognitive bandwidth" — [agentmaxxing post](https://codex.danielvaughan.com/2026/04/11/agentmaxxing-parallel-multi-cli-orchestration/).
  - "Design becomes the limiting factor"; Gas Town "was vibe designed too" — [Maggie Appleton](https://maggieappleton.com/gastown).
  - Every's "Dark vs. light factory": keep planning, design and review visible, and push well-defined work to the background — [Every](https://every.to/context-window/inside-the-100-agent-software-factory).
- **Runaway cost**:
  - Gas Town burns "thousands of dollars a month in API costs" — [Maggie Appleton](https://maggieappleton.com/gastown).
  - Gas City: ~1B tokens/day for ~100 agents; "a six-step job can cost six times as much as one Claude session" — [Every](https://every.to/context-window/inside-the-100-agent-software-factory).
  - C compiler: ~$20k for ~2,000 sessions — [Anthropic](https://www.anthropic.com/engineering/building-c-compiler).
  - The full app harness cost 20× the solo run — [Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps).
  - Ten background agents ≈ 10× quota burn — [agent-view doc](https://code.claude.com/docs/en/agent-view).
- **Too much parallelism**:
  - Same-bug thrash in kernel compilation: agents "overwrite each other's changes" — [Anthropic](https://www.anthropic.com/engineering/building-c-compiler).
  - "spawning 50 subagents for simple queries" — [Anthropic research](https://www.anthropic.com/engineering/multi-agent-research-system).
  - "Three focused teammates often outperform five scattered ones" — [agent-teams doc](https://code.claude.com/docs/en/agent-teams).
  - The ten-agent experience: whether "the codebase can absorb ten parallel writers without thrashing" — [Bernstein](https://bernstein.readthedocs.io/en/latest/blog/ten-agents-one-release/) (practitioner).
- **Zombies, orphans and debris**:
  - Gas Town's "Zombie" state = "Dead tmux session" — [Gas Town](https://github.com/steveyegge/gastown).
  - Agent teams: "Orphaned tmux sessions ... `tmux kill-session`" — [agent-teams doc](https://code.claude.com/docs/en/agent-teams).
  - Agent Deck's doorbell rule exists so spawned sessions don't become orphans "whose status events never route back"; there is also `worktree cleanup` for orphaned worktrees — [Agent Deck](https://github.com/asheshgoplani/agent-deck).
  - Claude Code troubleshooting: "`.claude/worktrees/` is filling up" — [agent-view doc](https://code.claude.com/docs/en/agent-view).
  - Symphony's startup cleanup "prevents stale terminal workspaces from accumulating" — [SPEC](https://github.com/openai/symphony/blob/main/SPEC.md).
- **Stalls and blind waits**:
  - Symphony: approval and input requests "MUST NOT leave a run stalled indefinitely". It has `stall_timeout_ms` and maps `turn_input_required` to failure in the high-trust example — [SPEC](https://github.com/openai/symphony/blob/main/SPEC.md).
  - Agent teams: "Task status can lag"; teammates and the lead may stop early — [agent-teams doc](https://code.claude.com/docs/en/agent-teams).
  - Gas Town "GUPP Violation" detection — [Gas Town](https://github.com/steveyegge/gastown).
  - Conductor's "Wait for `working` before trusting `idle`" — [Conductor docs](https://www.conductor.build/llms-full.txt).
- **Brittle TUI scraping**:
  - Claude Squad's literal-string prompt detection and Enter-tapping auto-yes — [tmux.go](https://github.com/smtg-ai/claude-squad/blob/main/session/tmux/tmux.go).
  - Gas Town's fixed "5-second ready delay instead of prompt detection" for Copilot — [Gas Town](https://github.com/steveyegge/gastown).
  - Claude Code shipped ~300 releases between 2026-01-07 and 2026-10-09 — [npm](https://registry.npmjs.org/@anthropic-ai/claude-code).
  - cmux's author: the Claude notification body is "always just 'Claude is waiting for your input' with no context" — [cmux README](https://github.com/manaflow-ai/cmux) (practitioner).
- **Agent self-harm and blind spots**:
  - Claude "ran `pkill -9 bash` by accident, killing itself and ending the loop"; also "time blindness" — [Anthropic](https://www.anthropic.com/engineering/building-c-compiler).
  - "Premature victory", "one-shotting", broken state between sessions — [Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents).
- **Overbuilt safeguards and jargon**:
  - "review loops and mid-task check-ins built for weaker models are now mostly unnecessary".
  - "Beads is hard for humans to view, so production teams often pair it with Jira or Linear". This dual-tracking cost is relevant to Narwarp's Linear sync.
  - Source: [Every](https://every.to/context-window/inside-the-100-agent-software-factory).
- **Business and ecosystem fragility**: Terragon shut down (Jan 2026), Vibe Kanban sunset (Apr 2026), Crystal deprecated (Feb 2026), and third-party subscription OAuth was blocked (Jan–Apr 2026) — [Terragon](https://github.com/terragon-labs/terragon-oss), [Vibe Kanban](https://www.vibekanban.com/blog/shutdown), [Crystal](https://github.com/stravu/crystal), [VentureBeat](https://venturebeat.com/technology/anthropic-cracks-down-on-unauthorized-claude-usage-by-third-party-harnesses).

### Inferences
- Narwarp's defaults should include:
  - a global cap of 3–5 concurrently *working* coding agents, with life and admin agents counted separately;
  - per-ticket token and quota budgets with auto-pause;
  - stall timers that escalate to the user's attention queue rather than auto-approving;
  - a single GC owner for worktrees and sessions;
  - "needs-review" as a first-class kanban column, so review is visible work rather than hidden backlog.
- Relying on vendor-run supervisors and protocols insulates Narwarp from UI-text churn. So does keeping its own scraping to hook payloads and `--json` output.

### Gaps
- No controlled study of optimal agent counts for a solo developer was found. The 3–5 figure is a practitioner heuristic.

---

## Q9. Pattern catalogue: which runtime patterns proved durable vs abandoned? (diagram-in-words)

### Takeaway
Durable: worktree-per-task; a daemon or supervisor separated from UIs; structured agent-server protocols; hooks-based telemetry; externalized durable work records; deterministic reconcile loops with concurrency caps; merge gates.

Declining or abandoned:
- containers-by-default for local parallel work (cloud sandboxes remain strong);
- pure tmux scraping as the primary status mechanism;
- context-reset-heavy harnesses;
- third-party reuse of subscription OAuth;
- venture-backed standalone cloud orchestrators and kanban wrappers.

### Cited Findings
- **P1. tmux-substrate manager.** `[Manager TUI] -(tmux CLI)-> [tmux server: owns PTYs] -> [claude TUI in pane, cwd = worktree]`. Status comes from `capture-pane` hashed or regex-matched; control from `send-keys`; state from JSON files.
  - Examples: Claude Squad, Agent Deck, Gas Town, Gas City (tmux default) — [Claude Squad](https://github.com/smtg-ai/claude-squad), [Agent Deck](https://github.com/asheshgoplani/agent-deck), [Gas Town](https://github.com/steveyegge/gastown), [Gas City](https://github.com/gastownhall/gascity).
  - Still maintained in Oct 2026; Agent Deck ships near-daily — [Go proxy](https://proxy.golang.org/github.com/asheshgoplani/agent-deck/@latest).
- **P2. Desktop app with an in-process PTY or runner.** `[Electron/Swift UI + main process] -> node-pty / headless runner -> CLIs in worktrees; SQLite`.
  - Examples: Emdash (PTY plus ACP), Nimbalyst (Agent SDK plus node-pty), Vibe Kanban (stream-json) — [Emdash AGENTS.md](https://github.com/generalaction/emdash/blob/main/AGENTS.md), [Nimbalyst package.json](https://github.com/Nimbalyst/nimbalyst/blob/main/packages/electron/package.json), [Vibe Kanban claude.rs](https://github.com/BloopAI/vibe-kanban/blob/main/crates/executors/src/executors/claude.rs).
- **P3. Host daemon plus thin clients.** `[daemon: owns PTYs/agent procs + state] <-(unix socket | HTTP/tRPC | JSON-RPC)-> {terminal UI, desktop, CLI, web, phone via relay}`.
  - Examples:
    - Claude Code supervisor — [agent-view](https://code.claude.com/docs/en/agent-view)
    - Superset host-service — [Superset docs](https://docs.superset.sh/llms-full.txt)
    - Happy daemon / Happy Agent — [happy-agent](https://github.com/slopus/happy-agent)
    - OpenCode server — [OpenCode](https://github.com/anomalyco/opencode/blob/master/packages/web/src/content/docs/server.mdx)
    - Codex app-server — [app-server docs](https://learn.chatgpt.com/docs/app-server)
    - cmux local-tmux — [cmux](https://github.com/manaflow-ai/cmux/blob/main/docs/local-tmux.md)
    - Xum server mode — [Xum](https://github.com/coder/mux)
- **P4. Agent-as-server structured protocol.** `[client] -spawn-> [agent subprocess] ; JSON-RPC over stdio (ACP / app-server) or HTTP+SSE (OpenCode); thread/turn/item events; approvals = server→client requests`.
  - Examples: ACP, app-server, OpenCode — [ACP](https://agentclientprotocol.com/protocol/overview), [app-server](https://learn.chatgpt.com/docs/app-server), [OpenCode](https://github.com/anomalyco/opencode/blob/master/packages/web/src/content/docs/server.mdx).
  - Churn: ACP v2 drafts and two adapter renames in 2026 — [claude-agent-acp](https://github.com/zed-industries/claude-agent-acp), [npm](https://registry.npmjs.org/@agentclientprotocol/claude-agent-acp).
- **P5. Ticket-driven reconciler.** `[tracker] <-poll- [orchestrator: single authority, claims, retry queue] -> [workspace per issue + hooks] -> [agent run (app-server)] -> events -> orchestrator; stall detection; terminal state → cleanup; success → handoff state (e.g., Human Review)`.
  - Examples: Symphony, Gas City controller loop — [Symphony SPEC](https://github.com/openai/symphony/blob/main/SPEC.md), [Gas City](https://github.com/gastownhall/gascity).
- **P6. Hierarchical LLM supervision.** `[human] <-> [coordinator agent (never codes)] -> [workers]; [watchdog agents] nudge/handoff stuck workers; [merge-queue agent]`.
  - Examples: Gas Town Mayor, Witness, Deacon, Refinery; Agent Deck conductors; agent-teams lead — [Gas Town](https://github.com/steveyegge/gastown), [Agent Deck](https://github.com/asheshgoplani/agent-deck), [agent-teams](https://code.claude.com/docs/en/agent-teams).
- **P7. Leaderless swarm.** `[N identical loop workers] -> claim via lock file in shared repo -> work -> rebase/merge -> push -> release lock`.
  - Examples: C compiler; agent-teams self-claim "uses file locking" — [Anthropic](https://www.anthropic.com/engineering/building-c-compiler), [agent-teams](https://code.claude.com/docs/en/agent-teams).
- **P8. Durable external memory and handoff.**
  - Beads ledger and seance — [Gas Town](https://github.com/steveyegge/gastown)
  - feature-list JSON plus progress file — [Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
  - `.context` handoff folder — [Conductor](https://www.conductor.build/llms-full.txt)
  - Managed Agents session log — [Anthropic](https://www.anthropic.com/engineering/managed-agents)
  - Repo-owned `WORKFLOW.md` — [Symphony](https://github.com/openai/symphony/blob/main/SPEC.md)
- **P9. Cloud sandbox plus local handoff.** `[local CLI/app] -submit-> [cloud VM/sandbox per task] ; results -> PR ; pull local via teleport/terry`.
  - Examples: Claude Code on the web (`--teleport`), Conductor Cloud, Warp Oz, Terragon (`terry`, shut down), Omnara, Managed Agents — [web doc](https://code.claude.com/docs/en/claude-code-on-the-web), [Conductor](https://www.conductor.build/llms-full.txt), [Warp](https://warp.dev/newsroom/2026/2/10/warp-launches-oz-the-orchestration-platform-for-cloud-coding-agents), [Terragon](https://github.com/terragon-labs/terragon-oss), [Omnara](https://github.com/omnara-ai/omnara).
- **P10. Relay-based remote control.** `[local session] -outbound HTTPS/WebSocket-> [relay] <- [phone/web]`.
  - Examples: Claude Remote Control (polling via Anthropic API), Happy E2E relay, Superset relay, Vibe Kanban relay tunnel, Agent Deck Telegram/Slack bridge — [remote-control](https://code.claude.com/docs/en/remote-control), [Happy](https://github.com/slopus/happy), [Superset docs](https://docs.superset.sh/llms-full.txt), [Vibe Kanban](https://github.com/BloopAI/vibe-kanban), [Agent Deck](https://github.com/asheshgoplani/agent-deck).
- **P11. Model gateway under the harness.** `[agent CLI] -(Anthropic/OpenAI API)-> [local gateway: route/failover/log] -> {Anthropic, OpenAI, OpenRouter, Ollama}`.
  - Examples: Claude Code Router, Ollama's Anthropic-compatible API — [CCR](https://github.com/musistudio/claude-code-router), [Ollama](https://docs.ollama.com/api/anthropic-compatibility).
- **Evidence of abandonment or decline**:
  - Sculptor made worktrees its default and moved the container backend to "experimental" — [Sculptor](https://github.com/imbue-ai/sculptor/blob/main/docs/help/workspaces.md).
  - Container Use has had no release since 2025-08-15 — [Go proxy](https://proxy.golang.org/github.com/dagger/container-use/@latest).
  - Terragon shut down — [terragon-oss](https://github.com/terragon-labs/terragon-oss).
  - Vibe Kanban sunset — [shutdown](https://www.vibekanban.com/blog/shutdown).
  - Crystal deprecated — [Crystal](https://github.com/stravu/crystal).
  - Context resets dropped with Opus 4.5 — [Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps).
  - The sprint construct was removed with Opus 4.6 — [Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps).
  - OpenCode removed Anthropic OAuth — [falcao.org](https://falcao.org/posts/anthropic-claude-access-crackdown-ecosystem-fallout/).
  - Happy moved from wrapping CLIs toward its own runtime, and Omnara pivoted to a managed-agent platform — [happy-agent](https://github.com/slopus/happy-agent), [Omnara](https://github.com/omnara-ai/omnara).

### Inferences
- **Durable**:
  - P3 as the backbone: first-party vendors adopted it (Claude supervisor, Codex app-server, OpenCode server).
  - P4 for non-Claude agents.
  - P5 for ticket-driven work.
  - P8 for continuity.
  - P10 now exists first-party (Remote Control), so third-party relays matter mainly for non-Claude agents.
- **Declining**:
  - P1 scraping as the *primary* signal. tmux survives as a PTY owner, but status moved to hooks.
  - P6 heavy LLM hierarchies for small setups (Gas Town's complexity critique).
  - Local containers by default.
  - Startups whose only moat was "parallel worktrees UI", because first parties (Claude Code agent view and Desktop, the Codex app) now ship that.

### Gaps
- Usage numbers (active users, retention) for most tools are unavailable, so "durable" is inferred from maintenance cadence and first-party adoption.

---

## Q10. Which reference architectures fit a single-user, local-first macOS harness (Narwarp) that drives Claude Code plus local Ollama agents?

### Takeaway
There are three viable shapes:
- **A. Cockpit over first-party supervisors.** The thinnest, built on `claude --bg` / `claude agents --json` / hooks / inbox sockets.
- **B. Narwarp daemon with structured adapters.** The richest: an own event log plus stream-json, ACP, OpenCode and app-server adapters.
- **C. PTY-first terminal multiplexer with hook telemetry.** The most native to a Warp fork, cmux/Superset-style.

The recommended starting point is a **C+A hybrid** for the Claude lane, a **B-style adapter** for the local Ollama lane, and a **Symphony-style reconciler** linking tickets to sessions. It needs the least new runtime code, stays on the supported subscription path, and survives Narwarp UI restarts.

### Cited Findings (building blocks these designs rely on)
- Claude Code background sessions run under a supervisor with `claude --bg`, `attach`, `logs`, `stop`, `respawn` and `rm`. `claude agents --json` is "the supported way to read session state from outside Claude Code" — [agent-view doc](https://code.claude.com/docs/en/agent-view).
- Per-session inbox sockets (`CLAUDE_CODE_MESSAGING_SOCKET`/`TOKEN`) let processes post messages into sessions; same-machine traffic never transits Anthropic — [cross-session doc](https://code.claude.com/docs/en/cross-session-messaging).
- Hook-based status and resume are used by Superset, Emdash and cmux — [Superset docs](https://docs.superset.sh/llms-full.txt), [Emdash](https://github.com/generalaction/emdash), [cmux](https://github.com/manaflow-ai/cmux).
- `claude --worktree`, background auto-isolation, and the sweep/lock cleanup — [worktrees doc](https://code.claude.com/docs/en/worktrees).
- Remote Control server mode for phone access to local sessions: `--spawn worktree --capacity N`, subscription only — [remote-control doc](https://code.claude.com/docs/en/remote-control).
- Headless stream-json with the control protocol and `--permission-prompt-tool=stdio` (Vibe Kanban, Sculptor). `--bare` needs an API key and is planned as the `-p` default — [claude.rs](https://github.com/BloopAI/vibe-kanban/blob/main/crates/executors/src/executors/claude.rs), [Sculptor](https://github.com/imbue-ai/sculptor/blob/main/docs/help/integrated_harnesses.md), [headless doc](https://code.claude.com/docs/en/headless).
- The OpenCode HTTP+SSE server (sessions, permissions, abort, fork, prompt_async) — [OpenCode docs](https://github.com/anomalyco/opencode/blob/master/packages/web/src/content/docs/server.mdx).
- Ollama's Anthropic-compatible endpoint and `ollama launch claude` — [Ollama docs](https://docs.ollama.com/api/anthropic-compatibility).
- The Symphony reconciler semantics: claims, slots, stall detection, backoff, terminal cleanup, handoff states, `WORKFLOW.md` — [SPEC](https://github.com/openai/symphony/blob/main/SPEC.md).
- The Managed Agents session/harness/sandbox split and credential vault/proxy — [Anthropic](https://www.anthropic.com/engineering/managed-agents).
- macOS gotchas:
  - Keychain-stored Claude credentials don't reach containers; use `claude setup-token` — [Sculptor](https://github.com/imbue-ai/sculptor/blob/main/docs/help/experimental/container_backend.md), [Agent Deck](https://github.com/asheshgoplani/agent-deck).
  - Background hosts need TCC folder grants and Local Network permission — [agent-view doc](https://code.claude.com/docs/en/agent-view).
  - Shell-profile daemon start keeps keychain, PATH and OAuth context — [happy-cli](https://github.com/slopus/happy/blob/main/packages/happy-cli/README.md).
- Native split panes for agent teams without tmux are possible in a custom terminal (cmux `claude-teams`). Claude's built-in split-pane mode supports only tmux and iTerm2, not Ghostty or other terminals — [cmux](https://github.com/manaflow-ai/cmux), [agent-teams doc](https://code.claude.com/docs/en/agent-teams).

### Inferences

**A. "Cockpit over first-party supervisors" (thinnest).**

Diagram in words:

```
[Narwarp UI (Rust, Warp fork)]
  ├─ panes: `claude attach <id>`   (real TUI, approvals happen here)
  ├─ poller: `claude agents --json --all` every ~2–5 s
  ├─ hook sink: unix socket narwarp.sock
  │     ← Claude hooks (SessionStart / Stop / Notification / PostToolUse)
  │       post {session_id, cwd, event}
  └─ sqlite: tickets, projects, board ↔ {claude sessionId, short id, worktree, branch}
dispatch:     `claude --bg --name <ticket-key> "<rendered ticket prompt>"`
steering:     write to session inbox socket, or "peek reply" by attaching
phone:        Claude Remote Control
Ollama lane:  Claude Code with ANTHROPIC_BASE_URL=http://localhost:11434
              under a separate CLAUDE_CONFIG_DIR, dispatched the same way
```

The Ollama lane can run its own supervisor instance, since "the supervisor uses that directory instead of `~/.claude` and runs as a separate instance".
- Pros:
  - Minimal runtime code.
  - Full fidelity, with sessions surviving Narwarp restarts and sleep.
  - The supported subscription path (unmodified CLI).
  - First-party status semantics (`waitingFor`).
  - First-party worktree isolation and cleanup.
- Cons:
  - Research-preview surface that churns.
  - Status is polled (supplemented by hooks).
  - Claude-centric. `claude --bg --exec '<cmd>'` can host other CLIs as PTY-backed jobs, but their output stays in memory only and the row is cleaned up about 5 minutes after exit ([agent-view doc](https://code.claude.com/docs/en/agent-view)), so non-Claude agents realistically need a separate path.
  - Approvals must be answered in an attached TUI or via Remote Control.
  - The internal state files are explicitly unstable.

**B. "narwarpd: own daemon with structured adapters" (richest).**

Diagram in words:

```
[narwarpd (Rust or Python), LaunchAgent / shell-profile started]
  owns: per-ticket workspaces (worktrees) and agent subprocesses
  writes: append-only per-session event log (JSONL) + SQLite index
          (Managed-Agents-style "session"; thread/turn/item normalized events)
  adapters:
    (1) Claude: `claude -p --input-format stream-json --output-format stream-json
                 --permission-prompt-tool stdio --resume <id>`,
        or ACP via @agentclientprotocol/claude-agent-acp
    (2) OpenCode server (HTTP+SSE) configured for Ollama models
    (3) Codex app-server (stdio JSON-RPC) if Codex is used
    (4) a small native Python/Rust loop for trivial local agents
  clients: Narwarp TUI panes (render cards, diffs, approval prompts natively),
           optional localhost web UI, optional phone relay
  reconciler: Symphony-style poll / claim / dispatch / stall / retry / handoff
              against the Narwarp board
```

- Pros:
  - Uniform rich UI and native approvals.
  - Exact status and cost (`total_cost_usd`).
  - Mid-turn steering via streamed input.
  - Easy heterogeneous agents.
  - Durable logs Narwarp owns.
- Cons:
  - The most code.
  - Interactive-only Claude features are lost (agent teams don't spawn in `-p`; slash-command TUI UX).
  - `--bare` and Agent SDK paths point to API-key billing.
  - Protocol churn (ACP v1→v2, adapter renames, Claude's `-p` default changes).

**C. "PTY-first multiplexer with hook telemetry" (most native to a terminal fork).**

Diagram in words:

```
[Narwarp terminal]
  each agent = real TUI in a Narwarp pane whose PTY is owned by
    narwarp-host (a small per-user daemon, Superset/Claude-supervisor style)
    or a user-owned tmux/zellij server (cmux local-tmux style)
  so panes survive UI restart
  telemetry: agent hooks → `narwarp notify` CLI → host socket;
             OSC 9/777 for attention rings
  isolation: `claude --worktree <ticket>`
  control:   input injection only for explicit nudges; no screen-scraping for state
  orchestrator: an always-on "conductor" Claude session given Narwarp MCP tools
                (create / assign / move ticket, read board),
                fed by watchers (Linear, Gmail, Calendar, Discord) as "doorbells"
                per Agent Deck's rule
```

- Pros:
  - Plays to Narwarp's strengths (it is a terminal).
  - Works for any CLI agent (Claude, Codex, OpenCode-TUI, aider+Ollama).
  - Full fidelity and subscription-friendly.
  - Native agent-team splits are possible (as cmux does).
- Cons:
  - Structured detail is limited to hook payloads.
  - Approvals are in-TUI only.
  - Own PTY host and GC to maintain.
  - Keystroke injection is brittle if over-used.

**Recommended hybrid.**
- Implement C's pane and host model.
- Use A's first-party supervisor for Claude sessions where possible: attach `claude attach <id>` inside Narwarp panes instead of owning Claude's PTY, and use `claude agents --json` plus hooks for state.
- Use B's adapter only for the local lane, via OpenCode server or Claude-Code-on-Ollama.
- Add the Symphony-style reconciler, with:
  - concurrency caps (3–5 working coding agents);
  - per-state slots;
  - stall timers that escalate to a "Needs you" column;
  - a "Human Review" handoff column;
  - terminal-state cleanup.
- Keep durable truth in Narwarp's SQLite plus per-ticket JSONL logs, with credentials held by Narwarp-side tools, never by agents.
- Re-audit adapters at each Claude Code or model upgrade, given the cadence of about 30 releases a month.

### Gaps
- Unvalidated assumptions to test in a spike:
  - Whether `claude attach` can be embedded cleanly in a non-tmux custom terminal pane.
  - Hook payload completeness for "waiting for approval" across Claude Code versions.
  - Whether Claude Code pointed at Ollama behaves acceptably for Narwarp's local-lane tasks.
- Whether Narwarp's upstream (Warp/Zap) already contains reusable agent-session plumbing was out of scope here. The TUI and fork researcher covers that.
