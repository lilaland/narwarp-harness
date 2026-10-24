# Agent-harness survey for Narwarp

Researched 2026-10-23 and 24. Stars are as GitHub showed them on 2026-10-24; "last commit" is the default-branch head in a shallow clone that day (UTC). **Inference** marks my own conclusions. Rule numbers refer to Narwarp's AGENTS.md.

## 1. Terminal-integrated agent management

Common pattern: agent hooks feed a local socket or file, a sidebar shows working, blocked, idle and done states, an OS notification fires on blocked or done, each task gets a worktree, and a persistent server allows detach and reattach. Sources are under each entry.

**Warp upstream (your fork's base).** Warp published its client on 2026-04-28. The blog says AGPL; the README says the `warpui` and `warpui_core` crates are MIT and the rest is AGPL v3. Last commit 2026-10-22. ([blog](https://www.warp.dev/blog/warp-is-now-open-source), [repo](https://github.com/warpdotdev/warp))
- Third-party CLI agents get live status in vertical tabs and an "agent toolbelt", but they don't appear in the Agent Management Panel. That panel lists only Warp's own agents and cloud runs, as Working, Blocked, Canceled, Failed or Success. ([CLI agents](https://docs.warp.dev/agents/cli-agents/overview.md), [panel](https://docs.warp.dev/platform/managing-cloud-agents.md))
- Claude Code reports through the MIT `claude-code-warp` plugin (last commit 2026-10-08). Its hooks emit OSC 777 titled `warp://cli-agent` with a versioned JSON body (`v`, `agent`, `event`, `session_id`, `cwd`). Events include `permission_request`, `idle_prompt`, `stop`, `stop_failure` and `tool_complete`. The receiving side is `cli_agent_protocol.rs` and `app/src/terminal/cli_agent_sessions/`. ([plugin](https://github.com/warpdotdev/claude-code-warp), [protocol](https://github.com/warpdotdev/warp/blob/master/crates/warp_core/src/cli_agent_protocol.rs))
- A sibling Oz plugin in the same repository drains a parent mailbox into the session from `UserPromptSubmit` and `PostToolUse` hooks, and its `Stop` hook keeps the session alive while messages are pending. The agent reports completion with `$OZ_CLI harness-support finish-task`. ([README](https://github.com/warpdotdev/claude-code-warp/tree/main/plugins/oz-harness-support))
- **Inference:** the fork could put tickets in the Agent Management Panel by adding a ticket ID to this event schema. But a `claude --bg` session has no terminal unless attached, so narwarpd needs its own feed into Warp, with OSC only for attached panes.

**Claude Code agent view (the substrate).** It has been in research preview since v2.1.139.
- `claude agents --json` is the supported way to read state. `state` is working, blocked, done, failed or stopped, and `waitingFor` names the blocker, such as a permission prompt or input.
- Background sessions move into worktrees, and since v2.1.198 they commit, push their own branch and open a draft PR when they finish.
- Hooks can POST to a URL (`http`) or call an MCP tool (`mcp_tool`), and can emit allowlisted OSC sequences through `terminalSequence`.
- Each session has an inbox socket, and a peer message "never counts as your consent".

([agent view](https://code.claude.com/docs/en/agent-view.md), [hooks](https://code.claude.com/docs/en/hooks.md), [messaging](https://code.claude.com/docs/en/cross-session-messaging.md))
**Inference:** auto-push needs Git credentials inside the session, which conflicts with rule 3. narwarpd should push itself or broker the push (see nono below).

**cmux** is a native macOS terminal built on libghostty. ★28.4k, GPL-3.0-or-later (server parts BUSL-1.1), last commit 2026-10-23.
- A waiting pane gets a ring. The sidebar shows branch, PR, ports and the last notification.
- It reads OSC 9, 99 and 777, and `cmux notify` can be wired to hooks.
- Its Feed parks a hook on a `request_id` for up to 120 s, until you click Allow or Deny in the sidebar or the notification. Every event is logged to JSONL.
- `cmux events --reconnect` replays events from a cursor file.
- Custom sidebars (beta) ship "Agents Board" and "Panel Todo" templates.

([repo](https://github.com/manaflow-ai/cmux), [Feed](https://github.com/manaflow-ai/cmux/blob/main/docs/feed.md), [sidebars](https://github.com/manaflow-ai/cmux/blob/main/docs/custom-sidebars.md))

**herdr** is a Rust agent multiplexer, a background server with a client you attach. ★45.6k, Apache-2.0, last commit 2026-10-23.
- Agents report state with `pane report-agent --source --state --seq`. States are idle, working or blocked, and a report with a stale `seq` is dropped. A report can carry a resume command.
- The socket speaks newline-delimited JSON. It offers `events.subscribe`, `agent.wait` until blocked or done, and an `events_lost` error that tells clients to take a fresh snapshot.
- Screen reading marks a pane `blocked` only when it matches a known approval prompt, and "idle status is not authorization to approve".

([repo](https://github.com/herdrdev/herdr), [reporting](https://herdr.dev/docs/add-herdr-support/), [socket](https://herdr.dev/docs/socket-api/), [agents](https://herdr.dev/docs/agents/))

**Orchestrator.inc** is a Go daemon with a desktop app; the old ComposioHQ/agent-orchestrator URL now serves it. ★13.5k, Apache-2.0, last commit 2026-10-23.
- "Display status is never stored. It is computed at read time from durable facts" (agent activity, PR, CI and review) by a fixed precedence.
- SQLite triggers write a change_log that is polled and pushed over SSE.
- The board columns are derived from those facts: Working, Needs you, In review, Ready to merge.
- "Automation must never inject input into a blocked session."

([repo](https://github.com/OrchestratorInc/agent-orchestrator), [architecture](https://github.com/OrchestratorInc/agent-orchestrator/blob/main/docs/architecture.md))

**Agent of Empires** manages agents as tmux sessions, with a TUI, a web view and an HTTP API. ★3.4k, MIT, last commit 2026-10-23. Hooks write status files, "pane reconciliation covers gaps hooks cannot see", and freshness bounds expire a stale "running". It can sandbox in Docker, Podman or Apple Containers. ([repo](https://github.com/agent-of-empires/agent-of-empires), [hooks](https://github.com/agent-of-empires/agent-of-empires/blob/main/src/hooks/status_file.rs))

**workmux** pairs tmux windows with worktrees. ★2.9k, MIT, last commit 2026-10-23. Status shows as icons in window names and in a dashboard. An agent with no output for 10 s is marked interrupted. Worktrees follow an add, merge and remove lifecycle. ([repo](https://github.com/raine/workmux))

**emdash** is a desktop app. ★6.1k, Apache-2.0, last commit 2026-10-23. It sends issues from ten trackers to agents and keeps state in local SQLite. It installs marker-tagged hooks that "silently do nothing when the agent runs outside an Emdash session". ([repo](https://github.com/generalaction/emdash))

**Cline Kanban** is a local browser board started with `npx kanban`, in research preview. ★1.4k, Apache-2.0, last commit 2026-09-04. Each card gets a worktree and a terminal. Hooks show the latest message or tool call on the card, inline diff comments go back to the agent, and linked cards start automatically. ([repo](https://github.com/cline/kanban), [docs](https://docs.cline.bot/kanban/core-workflow))

Also active:
- [dmux](https://github.com/standardagents/dmux): ★1.8k, MIT, 2026-08-16. Lifecycle hooks and macOS attention alerts.
- [ccmanager](https://github.com/kbwo/ccmanager): ★1.3k, MIT, 2026-10-18. Busy, waiting and idle detection, plus status-change hooks.
- [NTM](https://github.com/Dicklesworthstone/ntm): ★461, MIT with a rider, 2026-10-23. JSON robot mode, REST, SSE and WebSocket.
- [Supacode](https://github.com/supabitapp/supacode): ★2.4k, FSL-1.1-ALv2, 2026-10-23. Built on libghostty.
- [Xum](https://github.com/coder/mux), formerly Coder Mux: ★2.1k, AGPL, 2026-10-23. It runs its own agent loop.
- Zellij: only personal-scale plugins, such as [zellij-vtabs](https://github.com/otezz/zellij-vtabs) (2026-07-10).

## 2. Task and todo systems

**OpenAI Symphony** is a language-neutral SPEC.md plus an Elixir reference that polls a tracker and runs Codex for each issue. ★27.8k, Apache-2.0, last commit 2026-09-15. The reference code has adapters for Linear, GitHub, GitLab, Jira and Asana. ([repo](https://github.com/openai/symphony), [SPEC](https://github.com/openai/symphony/blob/main/SPEC.md))
- The orchestrator is "the only component that mutates scheduling state". Claims move from Unclaimed to Claimed, Running, RetryQueued and Released.
- Tracker tools run on the host, so "the child receives tool results, not a raw token". Adapters list `secret_environment_names()` to strip from the child's environment.
- Success often means reaching a handoff state such as "Human Review", not Done. An input request "MUST NOT leave a run stalled indefinitely".

**kata** (Wes McKinney) is a local issue ledger: a CLI for agents, a TUI and web view for humans, and a daemon over SQLite. ★500, MIT, last commit 2026-10-23. Version 0.19.0 adds sync with Linear, Notion, Plane, Todoist and others. ([repo](https://github.com/kenn-io/kata))
- Closing needs evidence (`--evidence commit:|pr:|test:`) and a message of at least 40 characters.
- Attention states are ok, stuck and needs-human, and `kata wait --until attention` blocks on them.
- It uses idempotency keys and searches before creating.
- Its MCP server exposes its tools section by section.

**td and sidecar** (Marcus Vorwaller). td: ★255, MIT, last commit 2026-10-19. It records structured handoffs (done, remaining, decisions, uncertain). Each context window gets a session ID, so reviews are attributed to a session and independent review is preferred. Boards are defined by queries. sidecar (2026-10-08) is a terminal dashboard around td. ([td](https://github.com/marcus/td), [sidecar](https://github.com/marcus/sidecar))

**Plannotator** is a plan-review UI. ★9.6k, Apache-2.0 or MIT, last commit 2026-10-23. A `PermissionRequest` hook on `ExitPlanMode` opens it in the browser. Approving continues; feedback denies the plan and returns structured annotations. ([repo](https://github.com/backnotprop/plannotator), [docs](https://docs.plannotator.ai/open-source/agents/claude-code))

**Spec Kit**: ★142.6k, MIT, last commit 2026-10-23. It reached 1.0.0 on 2026-08-21 and 1.2.3 on 2026-10-22. Work flows from spec to plan to tasks files. It adds catalogs of extensions, presets and workflows, and extensions can react to coding-agent lifecycle events. ([changelog](https://github.com/github/spec-kit/blob/main/CHANGELOG.md), [README](https://github.com/github/spec-kit))

**Claude Code's own task list** fires `TaskCreated` and `TaskCompleted` hooks. ([hooks](https://code.claude.com/docs/en/hooks.md)) **Inference:** narwarpd can mirror those tasks as read-only sub-items of a ticket.

## 3. Tools, connectors and credentials

**MCP specification 2026-07-28**, released that day.
- The core is stateless: no `initialize` and no session header.
- Multi round-trip requests replace server-initiated elicitation, and tasks move to an extension polled with `tasks/get`.
- Client ID Metadata Documents replace Dynamic Client Registration, which is deprecated.
- URL-mode elicitation sends the user to a server page for third-party OAuth. "The third-party credentials MUST NOT transit through the MCP client", and the server stores the tokens.
- stdio servers "SHOULD NOT" use MCP auth and "instead retrieve credentials from the environment".

([changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog), [elicitation](https://modelcontextprotocol.io/specification/2026-07-28/client/elicitation), [authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/index), [blog](https://blog.modelcontextprotocol.io/posts/2026-07-28/))
Claude Code's v2 MCP runtime supports this revision, but it won't register a channel server that negotiates it. ([MCP docs](https://code.claude.com/docs/en/mcp.md))
**Inference:** narwarpd's MCP server is the right holder for connector tokens, using URL elicitation for the owner's OAuth.

**Claude Code's extension surface.**
- `--strict-mcp-config` with `--mcp-config` limits a session to the servers you pass, and `claude agents` also accepts `--plugin-dir`.
- `headersHelper` runs a script for fresh auth headers on each connection.
- A plugin's `userConfig` value marked `sensitive: true` goes to secure storage.
- Caution: without managed settings, a mod "can approve a call that an `ask` rule would prompt for".

([MCP docs](https://code.claude.com/docs/en/mcp.md), [agent view](https://code.claude.com/docs/en/agent-view.md), [manifest](https://code.claude.com/docs/en/plugins/manifest-reference.md), [mods](https://code.claude.com/docs/en/plugins/mods/overview.md))
**Inference:** start each ticket's session with a plugin directory and strict MCP config generated by narwarpd, so third-party mods can't widen tools or approve calls.

**Docker MCP Gateway and Toolkit.** ★1.6k, MIT, last commit 2026-10-21. ([repo](https://github.com/docker/mcp-gateway))
- Servers run in containers.
- Profiles carry per-tool allowlists (`docker mcp profile tools --enable github.create_issue`), and `gateway run` takes `--servers` and `--tools`.
- `--block-secrets` is on by default, and image signatures are checked.
- OAuth flows are built in. Secrets live in Docker Desktop, which must be 4.59 or later.

**ToolHive** (Stacklok). ★2.3k, Apache-2.0, last commit 2026-10-23. ([repo](https://github.com/stacklok/toolhive), [secrets](https://github.com/stacklok/toolhive/blob/main/docs/arch/04-secrets-management.md), [local vMCP](https://github.com/stacklok/toolhive/blob/main/docs/arch/vmcp-local.md))
- Each server runs in a container.
- Secrets sit in an AES-256-GCM store whose password is kept in the macOS Keychain; 1Password is an alternative provider.
- Servers are organized in groups. `thv vmcp serve` aggregates a group locally, without Kubernetes, with an optional tool-search optimizer.

**1MCP**: ★524, Apache-2.0, last commit 2026-10-22. It is one aggregated runtime in which "template servers are created from per-client or per-session context". In CLI mode, agents discover tools step by step: `instructions`, then `inspect`, then `run`. ([repo](https://github.com/1mcp-app/agent))

**MCPJungle**: ★1.3k, MPL-2.0, last commit 2026-08-02. Each tool group gets its own endpoint. Enterprise mode issues tokens per client, with a list of allowed servers. ([repo](https://github.com/mcpjungle/MCPJungle))

Heavier team gateways:
- [IBM ContextForge](https://github.com/IBM/mcp-context-forge): ★4.7k, Apache-2.0, 2026-10-23. Virtual servers and encrypted stored credentials.
- [agentgateway](https://github.com/agentgateway/agentgateway): ★5.5k, Apache-2.0, 2026-10-23. Written in Rust, with CEL RBAC.
- [Obot](https://github.com/obot-platform/obot): ★1.1k, MIT, 2026-10-23. Virtual servers pinned to catalog snapshots, plus an audit log.
- [Microsoft MCP Gateway](https://github.com/microsoft/mcp-gateway): ★893, MIT, 2026-10-14. Runs on Kubernetes.

**Official MCP Registry.** ★7.4k, license moving from MIT to Apache-2.0, last commit 2026-10-23. The README still calls it a preview, launched 2025-09-08, with a v0.1 API freeze since 2025-10-24. Docker profiles can reference registry entries. ([registry](https://github.com/modelcontextprotocol/registry), [Docker](https://github.com/docker/mcp-gateway))

**Hosted token brokers.**
- Composio's Tool Router (SDK ★30.7k, MIT, 2026-10-23) scopes a session to one user ID, exposes it at an MCP URL, and manages auth by default.
- Smithery Connect stores credentials "encrypted and write-only" and issues scoped service tokens.
- Arcade's Python `arcade-mcp` (★1.1k, MIT, last commit 2026-10-23) declares auth per tool, as in `requires_auth=GitHub(scopes=["repo"])`, and injects the token at call time: "The client and the LLM never see the secret values". In standalone mode you supply your own tokens.

([Composio](https://docs.composio.dev/tool-router/using-with-mcp-clients), [Smithery](https://www.smithery.ai/docs/use/connect.md), [arcade-mcp](https://github.com/ArcadeAI/arcade-mcp))
**Inference:** Composio's and Smithery's hosted services keep tokens in the vendor's cloud, which conflicts with local-first. Arcade's per-tool scope declaration is the pattern worth copying.

**Credential brokers for agent processes.**
- nono: ★4.7k, Apache-2.0, last commit 2026-10-23. It is a kernel-enforced sandbox plus a proxy that swaps a per-session "phantom token" for the real secret. The secret comes from the macOS Keychain, 1Password or Apple Passwords. Rules can limit each route by method and path, and there are Python bindings. Its docs admit it "cannot prevent a sandboxed process from misusing a credential it legitimately holds".
- Infisical Agent Vault: ★2.4k, MIT except `ee/`, last commit 2026-10-22. It is an `HTTPS_PROXY` man-in-the-middle that swaps placeholder values for real credentials and can deny unknown hosts. Its docs recommend running it on a separate machine.

([nono](https://github.com/nolabs-ai/nono), [injection](https://nono.sh/docs/cli/features/credential-injection), [Agent Vault](https://github.com/Infisical/agent-vault))
**Inference:** use these for `gh` and `git` traffic only. Routing Claude's own login through a proxy would break rule 9.

**Archestra** (a server platform): ★4.4k, AGPL-3.0, last commit 2026-10-23. Its dual-LLM pattern lets the quarantined model answer only multiple-choice questions, because "Numbers can't contain instructions!" This matches Narwarp's quarantined reader. ([repo](https://github.com/archestra-ai/archestra), [blog, 2025-10-13](https://www.archestra.ai/blog/dual-llm))

**Licenses.** cmux is GPL, and Xum and Archestra are AGPL. Study them, but don't copy their code into narwarp-harness.

## Left out

No commits since before 2026-06-24:
- MetaMCP (2026-06-22)
- Klavis Strata (2026-06-01)
- Lasso MCP Gateway (2026-01-22)
- StrongDM Leash (2026-04-06)
- Smithery's CLI repository (2026-05-31)
- GSD (2026-05-31)
- beans (2026-04-06)
- Trekker (2026-04-10)
- ticket (2026-03-15)
- zjbar (2026-03-24)

Everything the brief listed as already researched is also excluded, as are goose and Buzz.

## Shortlist for deep research (ranked)

1. **Warp `cli_agent_sessions`, Agent Management Panel and `claude-code-warp`:** Can tickets ride the existing CLI-agent event schema with an added ticket ID, and how do unattached `--bg` sessions reach the panel through a daemon feed?
2. **Claude Code agent view, hooks and inbox socket:** Which signal is authoritative for each board state (`claude agents --json`, `http` hooks or the inbox socket), and how do they reconcile after a restart?
3. **cmux Feed:** How would its parked-hook approval flow work with daemon-signed, single-use owner approvals answered from Warp?
4. **Orchestrator.inc:** Can narwarpd derive board columns from stored facts by precedence, and push changes to Warp from a SQLite change log?
5. **Symphony SPEC:** Which parts map onto narwarpd's tickets and MCP tools: claim states, host-side tracker tools, success at a handoff state, no stalling on input?
6. **kata:** Should Narwarp's ticket schema copy its close-with-evidence rule, attention states, idempotency keys and sectioned MCP tools?
7. **herdr:** Is its sequenced state-report protocol, with re-snapshot after `events_lost`, the right shape for the Warp–narwarpd socket?
8. **MCP 2026-07-28 and arcade-mcp:** Can narwarpd's Python MCP server hold connector tokens, declare scopes per tool and run the owner's OAuth by URL elicitation within Claude Code's current MCP runtime?
9. **nono, compared with Agent Vault:** Can phantom tokens let background sessions push branches and open PRs on macOS without holding GitHub credentials?
10. **ToolHive vMCP, Docker MCP Gateway or plain `--strict-mcp-config`:** Which gives per-ticket tool allowlists on the Mac Studio with the least machinery?
11. **Agent of Empires:** How does it combine hook status with pane checks and freshness bounds, and does narwarpd need a similar watchdog?
