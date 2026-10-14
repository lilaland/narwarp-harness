# Agent coordination via shared message boards, mailboxes, task lists and handoff documents (state as of 2026-10-13)

Conventions: "(>6 mo)" marks a source dated before mid-April 2026, so it may be stale. "Local inspection" means I observed it directly on 2026-10-13 in this environment: the Claude Code binary `/opt/claude-code/bin/claude` v2.1.296, its task files under `~/.claude/tasks/`, or the Narwarp repo. Undocumented internals can change without notice. Practitioner or vendor claims are labelled as such.

---

## 1. How existing coordination systems work internally, and what Narwarp can borrow

### Takeaway
Mature 2025–2026 systems have converged on four parts. First, an authoritative, structured work-state store with atomic claims or leases (Beads, Claude Code task list). Second, a separate append-only, threaded, addressable message log with per-recipient read and ack state (MCP Agent Mail, Claude Code mailboxes). Third, delivery at turn or tool boundaries through hooks rather than mid-tool interruption (Gas Town, Claude Code). Fourth, a human "overseer" channel that outranks agents, with peer messages explicitly marked as not being user consent (Claude Code, Agent Mail).

Linear's AgentActivity types and signals, plus A2A's task-state machine, are the best ready-made vocabularies for message and status types. LangGraph's shared state, OpenAI handoffs and Letta shared blocks are in-process designs and less applicable to cross-session boards.

### Cited Findings

#### MCP Agent Mail (Dicklesworthstone): Python version superseded, Rust rewrite current
- **Status:** The Python README says the project is superseded by `mcp_agent_mail_rust` and is "no longer maintained". The installer now uses Beads Rust (`br`) and aliases `bd` to it. — [mcp_agent_mail README](https://github.com/Dicklesworthstone/mcp_agent_mail)
- **Why Rust:** The rewrite targets Python-version failures under many agents: "Git lock contention, SQLite pool exhaustion, and cascading failures". It addresses them with batching, bounded pools and backpressure. Release notes cite v0.3.31 (new release trust model) and v0.3.39 (session-bound identity, `project:<slug>` addressing). Agent Mail has been available since October 2025. No Rust release dates were visible. — [mcp_agent_mail_rust README](https://github.com/Dicklesworthstone/mcp_agent_mail_rust)
- **Data model (Python README):**
  - `projects(id, human_key, slug, created_at)`
  - `agents(id, project_id, name, program, model, task_description, inception_ts, last_active_ts, attachments_policy, contact_policy)`. Agent names are memorable adjective+noun pairs such as "GreenCastle".
  - `messages(id, project_id, sender_id, thread_id, subject, body_md, created_ts, importance, ack_required, attachments)`
  - `message_recipients(message_id, agent_id, kind, read_ts, ack_ts)`, where `kind` is To/Cc/Bcc
  - `file_reservations(... path_pattern, exclusive, reason, created_ts, expires_ts, released_ts)`
  - Replies inherit `thread_id`. If none exists, the reply's `thread_id` is set to the original message id.
  - Source: [mcp_agent_mail README](https://github.com/Dicklesworthstone/mcp_agent_mail)
- **Storage is dual:**
  - A Git repo holds human-auditable Markdown: `agents/<Name>/inbox|outbox/YYYY/MM/<id>.md`, `messages/YYYY/MM/<id>.md`, `file_reservations/<sha1>.json` and content-addressed attachments.
  - SQLite with FTS5 handles indexing, queries and reservations.
  - Each message is GitHub-Flavored Markdown with a `---json` frontmatter block: `id, thread_id, from, to, created, importance, ack_required, attachments`.
  - Source: [mcp_agent_mail README](https://github.com/Dicklesworthstone/mcp_agent_mail)
- **MCP tools:**
  - Identity and project: `ensure_project`, `register_agent`, `whois`, `list_agents`
  - Messaging: `send_message` (with `to/cc/bcc`, `thread_id`, `ack_required`, `auto_contact_if_blocked`), `reply_message`, `fetch_inbox` (`unread_only`, `urgent_only`, `include_bodies`, `limit`), `acknowledge_message`, `mark_message_read`, `search_messages`, `summarize_thread`
  - Reservations: `file_reservation_paths`, `renew_file_reservations`, `release_file_reservations`, `force_release_file_reservation`
  - Contacts: `request_contact`, `respond_contact`, `set_contact_policy`, `list_contacts`
  - Macros: `macro_start_session` (returns `{project, agent, file_reservations, inbox}`), `macro_prepare_thread`, `macro_file_reservation_cycle`, `macro_contact_handshake`
  - The README recommends macros "for speed or smaller models".
  - Sources: [README](https://github.com/Dicklesworthstone/mcp_agent_mail); [SKILL.md](https://cdn.jsdelivr.net/gh/dicklesworthstone/mcp_agent_mail@main/SKILL.md)
- **Contact policies:**
  - `open`: accept any message in the project
  - `auto` (default): allow if shared context exists, meaning the same thread, overlapping reservations or recent contact
  - `contacts_only`: explicit approval required first
  - `block_all`: reject new contacts. A blocked send returns `CONTACT_BLOCKED`.
  - Cross-project messaging needs an approved contact link even when the recipient's policy is `open`.
  - Sources: [SKILL.md](https://cdn.jsdelivr.net/gh/dicklesworthstone/mcp_agent_mail@main/SKILL.md); [Rust README](https://github.com/Dicklesworthstone/mcp_agent_mail_rust)
- **Importance levels:** `low`, `normal`, `high`, `urgent`. — [SKILL.md](https://cdn.jsdelivr.net/gh/dicklesworthstone/mcp_agent_mail@main/SKILL.md)
- **File reservations:**
  - They are advisory, TTL-based leases on globs, exclusive or shared. The documented signature is `file_reservation_paths(project_key, agent_name, paths[], ttl_seconds, exclusive, reason)`.
  - "Conflicts are reported but reservations are still granted."
  - An optional pre-commit guard (`mcp-agent-mail-guard`, keyed on the `AGENT_NAME` env var) blocks commits that touch others' exclusive reservations. Emergency bypass is `AGENT_MAIL_BYPASS=1`.
  - The Rust pre-push guard is bounded (2,000 commits, 100,000 paths, 120 s by default) and fails closed with exit 2.
  - No default TTL is documented. Examples use `ttl_seconds=3600`.
  - Sources: [README](https://github.com/Dicklesworthstone/mcp_agent_mail); [SKILL.md](https://cdn.jsdelivr.net/gh/dicklesworthstone/mcp_agent_mail@main/SKILL.md); [Rust README](https://github.com/Dicklesworthstone/mcp_agent_mail_rust)
- **Notification is pull-based:**
  - Agents call `fetch_inbox` or read the resources `resource://inbox/{agent}` and `resource://thread/{id}`.
  - Rust adds cursor-based `am inbox-events` / `fetch_inbox_events` with the rule: "advance a recipient delivery cursor only after handling an oldest-first page; a message ID is never a safe cursor substitute".
  - No Claude Code hook integration was documented in the visible README text.
  - Sources: [README](https://github.com/Dicklesworthstone/mcp_agent_mail); [Rust README](https://github.com/Dicklesworthstone/mcp_agent_mail_rust)
- **Thread convention with Beads:**
  - Use the Beads id (e.g., `bd-123`) as `thread_id`, as the subject prefix `[bd-123]`, as the reservation `reason`, and in commit messages.
  - Start: announce in the thread. Progress: "Reply in thread with progress." Finish: post "[bd-###] Completed" and release reservations.
  - "Use Beads for task status and dependencies, and Mail for conversation and audit."
  - Sources: [SKILL.md](https://cdn.jsdelivr.net/gh/dicklesworthstone/mcp_agent_mail@main/SKILL.md); [README](https://github.com/Dicklesworthstone/mcp_agent_mail)
- **Human overseer:**
  - A web composer sends messages from a special `HumanOverseer` sender. They are always high importance and bypass contact policies.
  - A preamble tells agents to pause, prioritize the request, then resume.
  - There is a web UI at `/mail`. The Rust TUI has 16 screens, including Dashboard, Messages, Reservations and "ATC". Robot mode offers 19 subcommands with `toon/json/md` output.
  - Sources: [README](https://github.com/Dicklesworthstone/mcp_agent_mail); [Rust README](https://github.com/Dicklesworthstone/mcp_agent_mail_rust)
- **"ATC" advisories:** These are probe or advisory mails to agents, spaced by `AM_ATC_ADVISORY_COOLDOWN_SECS`, which defaults to 300. Design assumptions are stated: "Stale lock cleanup is expected" and "agents crash and processes die unexpectedly". — [Rust README](https://github.com/Dicklesworthstone/mcp_agent_mail_rust)
- **Stress results (Rust README, earlier campaign):** These are vendor self-reports, explicitly "not a verdict for the current release candidate".
  - 30-agent pipeline: 150/150 succeeded, p99 6.8 s
  - Commit coalescer: 100 writes became 11 commits (9.1×)
  - Sustained mixed load: about 49 RPS, p99 2.6 s
  - Source: [Rust README](https://github.com/Dicklesworthstone/mcp_agent_mail_rust)
- **Security (as documented):**
  - Bearer token or JWT/JWKS. Localhost-unauthenticated mode is for development only.
  - Bleach HTML sanitization, and a secret-scrubbing preset covering GitHub, Slack and OpenAI tokens and JWTs.
  - Session-bound identity is opt-in and "does not authenticate the model or provider". It cannot isolate hostile clients that share one bearer credential.
  - Reservations are advisory, so they don't stop agents that ignore them.
  - Prompt injection via message content is not discussed.
  - Sources: [README](https://github.com/Dicklesworthstone/mcp_agent_mail); [Rust README](https://github.com/Dicklesworthstone/mcp_agent_mail_rust)
- **Promotional content:** The README promotes a commercial Companion app, and its productivity claims are unverified. — [README](https://github.com/Dicklesworthstone/mcp_agent_mail)

#### Claude Code Agent Teams: mailbox and shared task list (experimental; launched 2026-02-05)
- **Launch and status:** Agent Teams shipped as a research preview with Claude Opus 4.6 on 2026-02-05. As of 2026-10-13 the docs still say "experimental and disabled by default", enabled with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. Spawning teammates requires an interactive session, not `-p` or Agent SDK. — [TechCrunch](https://techcrunch.com/2026/02/05/anthropic-releases-opus-4-6-with-new-agent-teams); [Anthropic](https://www.anthropic.com/news/claude-opus-4-6); [Agent teams docs](https://code.claude.com/docs/en/agent-teams)
- **Architecture:** A team lead, teammates (separate Claude Code instances), a shared task list and a mailbox.
  - Each agent's mailbox is a JSON file at `~/.claude/teams/{team-name}/inboxes/{agent-name}.json`. Entries are validated on read, and invalid entries are removed (before v2.1.207 one malformed entry blocked delivery).
  - A send is reported as sent only when the write to the recipient's mailbox file succeeds.
  - Team config is at `~/.claude/teams/{team-name}/config.json` (with a `members` array) and is deleted at session end.
  - The task list at `~/.claude/tasks/{team-name}/` persists.
  - Source: [Agent teams docs](https://code.claude.com/docs/en/agent-teams)
- **Task list:**
  - Three states: pending, in progress, completed.
  - A pending task with unresolved dependencies can't be claimed.
  - "Task claiming uses file locking to prevent race conditions."
  - Completing a task auto-unblocks its dependents.
  - Teammates either self-claim the "next unassigned, unblocked task" or are assigned by the lead.
  - Source: [Agent teams docs](https://code.claude.com/docs/en/agent-teams)
- **Task file format (local inspection, v2.1.296):** Each task is a separate JSON file (`1.json`, `2.json`, …) next to a `.lock` file. Fields are `id, subject, description, activeForm, status ("pending"|"in_progress"|"completed"), blocks[], blockedBy[]`. — local inspection of `~/.claude/tasks/<session-id>/`
- **Notification:**
  - "Automatic message delivery … The lead doesn't need to poll."
  - Idle notifications: a teammate that stops notifies the lead and includes its final answer, or the API error text if it failed.
  - Messages go to one teammate by name. "To reach everyone, send one message per recipient."
  - Teammates don't inherit the lead's history, only the spawn prompt plus CLAUDE.md, MCP servers and skills.
  - Source: [Agent teams docs](https://code.claude.com/docs/en/agent-teams)
- **Injection format (local inspection, undocumented):** Received teammate messages are parsed from tags matching `<teammate-message teammate_id="…" color="…"? summary="…"? verified="false"?>`. Mailbox module symbol names reveal structured protocol message types: idle notification, permission request/response, sandbox permission request/response, shutdown request/approved/rejected, plan approval request/response, task assignment, mode-set request, team permission update and teammate wake-up prompt, plus `readUnreadMessages` and `markMessagesAsRead`. — local inspection of `/opt/claude-code/bin/claude` v2.1.296
- **Trust:**
  - A receiving agent is told the message came "from another Claude session, not from you". A teammate "can't approve a permission prompt or supply consent on your behalf", and a denied action can't be relayed to another teammate to bypass the check.
  - In auto mode, a classifier treats relayed approval claims as untrusted. It also reviews each inter-agent message before delivery and blocks some.
  - **Caveat:** plan approval requests are auto-approved by the lead session "without the lead reviewing it".
  - Source: [Agent teams docs](https://code.claude.com/docs/en/agent-teams)
- **Hooks:** `TeammateIdle` (exit 2 keeps the teammate working), `TaskCreated` (exit 2 rolls back creation) and `TaskCompleted` (exit 2 prevents completion). — [Agent teams docs](https://code.claude.com/docs/en/agent-teams); [Hooks docs](https://code.claude.com/docs/en/hooks)
- **Limitations and best practices:**
  - No resumption of in-process teammates. "Task status can lag" (teammates forget to mark completion). One team per session, no nested teams, and the lead is fixed.
  - Start with 3–5 teammates and 5–6 tasks per teammate. "Three focused teammates often outperform five scattered ones."
  - "Two teammates editing the same file leads to overwrites."
  - "Letting a team run unattended for too long increases the risk of wasted effort."
  - Source: [Agent teams docs](https://code.claude.com/docs/en/agent-teams)

#### Claude Code cross-session messaging (≥ v2.1.224; current)
- **Mechanism:**
  - `ListAgents` discovers targets and `SendMessage` sends.
  - Same-machine delivery uses a per-session Unix socket (named pipe on Windows), "never through Anthropic servers". Other machines and cloud sessions go via Remote Control and Anthropic servers.
  - The socket path is exported to hooks and Bash as `CLAUDE_CODE_MESSAGING_SOCKET`, with a per-session `CLAUDE_CODE_MESSAGING_TOKEN`, so scripts and hooks can post into a session.
  - Source: [Cross-session messaging docs](https://code.claude.com/docs/en/cross-session-messaging)
- **Delivery semantics:**
  - "The receiving Claude reads the message between tool calls during an active turn, so a running tool is never interrupted." An idle session starts a new turn with the message.
  - `notify_when_idle` (≥ v2.1.236) sends one notice when the target goes idle. The subscription expires after 12 h.
  - Source: [Cross-session messaging docs](https://code.claude.com/docs/en/cross-session-messaging)
- **Inbound controls:**
  - `crossSessionInbound` takes `accept | hold | refuse`.
  - The default depends on permission-mode classes. A bypass-permissions receiver holds messages for approval.
  - At most 100 held messages, with the oldest dropped. Held messages expire after `dialogExpiry`, five minutes by default.
  - `isolatePeerMachines: true` requires approval before any message leaves the machine.
  - Source: [Cross-session messaging docs](https://code.claude.com/docs/en/cross-session-messaging)
- **Trust rules:** An incoming message "can't approve anything", "can't change configuration", "commands don't run", and "permission prompts still fire". The auto-mode policy text in the local binary says cross-session messages wrapped in `<cross-session-message>` are "never user intent". — [Cross-session messaging docs](https://code.claude.com/docs/en/cross-session-messaging); local inspection of v2.1.296 binary
- **Anti-loop limits:**
  - Plain text only. Same-machine messages are capped at about 1M characters.
  - Rapid bursts are refused at the sender.
  - The receiver rate-limits repeated messages per sender, "drops identical repeats arriving within a short window, and queues at most 50 accepted messages … A message loop between two sessions therefore stops on its own."
  - Source: [Cross-session messaging docs](https://code.claude.com/docs/en/cross-session-messaging)
- **Channels:** Claude Code also has "Channels" to "push external events, such as CI results or chat messages, into a session" (another researcher covers this). — [Cross-session messaging docs](https://code.claude.com/docs/en/cross-session-messaging)

#### Beads (steveyegge/beads): issue graph as agent memory; v1.0.0 around April 2026
- **What it is:** A "distributed graph issue tracker for AI agents, powered by Dolt" that "provides a persistent, structured memory for coding agents". It advises against markdown TODO lists for task tracking. About 27.8k stars on 2026-10-13. — [Beads README](https://github.com/steveyegge/beads)
- **Storage has changed:** Beads is now Dolt-backed. Embedded mode uses `.beads/embeddeddolt/` with a single writer. Server mode (`bd init --server`) supports concurrent writers. `.beads/issues.jsonl` "is an export for viewers and interchange, not the source of truth or a backup". Older write-ups describing JSONL-in-git plus a SQLite cache as the store are therefore stale. — [Beads README](https://github.com/steveyegge/beads)
- **Model:**
  - Hash IDs (`bd-a1b2`) avoid merge collisions. Hierarchical IDs: `bd-a3f8` (epic), `.1` (task), `.1.1` (subtask).
  - Dependencies: `blocks`, `related`, `parent-child`. Graph links: `relates-to`, `duplicates`, `supersedes`, `replies-to`.
  - A **message issue type** supports threading (`--thread`) and an ephemeral lifecycle.
  - "Semantic 'memory decay'" summarizes old closed tasks to save context.
  - Source: [Beads README](https://github.com/steveyegge/beads)
- **Agent commands:**
  - `bd ready`: unblocked work
  - `bd update <id> --claim`: "Atomically claim a task (sets assignee + in_progress)"
  - `bd close`: releases blockers
  - `bd prime`: prints workflow context and memories for injection
  - `bd remember "insight"`: project memory re-injected by `bd prime`. Per-operator preferences "belong in the harness's own memory".
  - `bd setup claude`: installs hooks and settings
  - Source: [Beads README](https://github.com/steveyegge/beads)
- **1.0 date:** Gas Town and Beads 1.0.0 reportedly shipped together around 2026-04-03, per Yegge's Medium as relayed by an aggregator. I could not see the primary post. — [feedbagel](https://feedbagel.com/post/gas-town-software-reaches-version-10-after-3-month-development-journey)

#### Gas Town (Jan 2026) → Gas City (successor SDK, announced 2026-04-24)
- **Roles:**
  - Mayor (coordinator)
  - Polecats (workers with "persistent identity but ephemeral sessions", in isolated worktrees)
  - Witness (per-rig monitor that "detects stuck agents" and triggers recovery)
  - Deacon (cross-rig watchdog "running continuous patrol cycles")
  - Dogs (maintenance; "Boot" does triage)
  - Refinery (Bors-style merge queue that bisects failures)
  - Crew (human workspaces)
  - Scale target: from "4-10 agents" to "20-30 agents".
  - Sources: [Gas Town README](https://github.com/steveyegge/gastown); [glossary](https://raw.githubusercontent.com/steveyegge/gastown/main/docs/glossary.md)
- **Work pinning:** A **Hook** is "a special pinned Bead for each agent" that acts as its work queue. **GUPP** (Gas Town Universal Propulsion Principle): "If there is work on your Hook, YOU MUST RUN IT." The problems view flags a "GUPP Violation" when hooked work shows no progress. — [glossary](https://raw.githubusercontent.com/steveyegge/gastown/main/docs/glossary.md); [README](https://github.com/steveyegge/gastown)
- **Messaging:**
  - Mail is injected through Claude Code hooks: `UserPromptSubmit` runs `gt mail check --inject`, while `SessionStart` and `PreCompact` run `gt prime --hook` to re-prime context before compaction.
  - `Stop` records costs. PreToolUse guards cover dangerous commands and PR-workflow bypass.
  - **Nudge** (`gt nudge`) is "real-time messaging between agents" that bypasses mail.
  - Escalation (`gt escalate -s HIGH`) has severities CRITICAL (P0), HIGH (P1) and MEDIUM (P2), routing via Deacon, then Mayor, then the human Overseer.
  - For runtimes without hooks (e.g., Codex), Gas Town types `gt prime`, `gt mail check --inject` and `gt nudge deacon session-started` into the session.
  - Sources: [HOOKS.md](https://raw.githubusercontent.com/steveyegge/gastown/main/docs/HOOKS.md); [glossary](https://raw.githubusercontent.com/steveyegge/gastown/main/docs/glossary.md); [README](https://github.com/steveyegge/gastown)
- **Handoff:** "Agent session refresh via `/handoff`" carries work state into a new session. **Seance** (`gt seance`, `--talk <id>`) lets an agent query predecessor sessions found via `.events.jsonl` logs about earlier decisions. — [glossary](https://raw.githubusercontent.com/steveyegge/gastown/main/docs/glossary.md); [README](https://github.com/steveyegge/gastown)
- **Gas City:** It "extracts the reusable infrastructure from Gas Town into a configurable toolkit". Its beads store covers "work, mail, convoys". It runs a controller/supervisor loop that "reconciles desired state to running state". Announced 2026-04-24 (aggregator), with v1.5.0 described as the current stable release. — [Gas City README](https://raw.githubusercontent.com/gastownhall/gascity/HEAD/README.md); [dsebastien.net](https://www.dsebastien.net/tag/news/page/7/); [newreleases v1.5.0](https://newreleases.io/project/github/gastownhall/gascity/release/v1.5.0)

#### CCPM (automazeio/ccpm): GitHub issues as the coordination log
- **Status:** Now packaged as an "Agent Skill" for Claude Code, Codex, OpenCode, Factory, Amp and Cursor. The old `/pm:*` slash commands are kept on the `v1` branch. Examples are dated 2026-03-18. — [CCPM README](https://github.com/automazeio/ccpm)
- **Flow:**
  - PRD (`.claude/prds/<name>.md`), then epic (`.claude/epics/<name>/epic.md`), then task files with `depends_on`, `parallel` and `conflicts_with` (about 10 per epic), then a GitHub epic issue plus sub-issues, then parallel agents in an epic worktree `../epic-<name>/`.
  - Progress "is visible in real-time through issue comments" and "Comments are the audit trail".
  - Local `updates/` folders hold agent progress, synced to GitHub on demand.
  - Source: [CCPM README](https://github.com/automazeio/ccpm)
- **Unverified claims:** "89% less time" lost to context switching, "75% reduction" in bug rates, and 100% vs 27.7% on a three-scenario benchmark come with no methodology. These are vendor claims. — [CCPM README](https://github.com/automazeio/ccpm)

#### Linear Agent Sessions and AgentActivity: a ready-made update schema
- **Sessions:** States are `pending, active, error, awaitingInput, complete, stale`. "Linear tracks session lifecycle automatically based on the last emitted activity." Sessions are created on mention or delegation, or proactively via `agentSessionCreateOnIssue` / `agentSessionCreateOnComment`. — [Linear agent interaction](https://linear.app/developers/agent-interaction)
- **Agent-emitted activity types:**
  - `thought {body}`
  - `elicitation {body}` (asks the user)
  - `action {action, parameter, result?}`
  - `response {body}` (work complete; also creates a comment)
  - `error {body}`
  - `prompt` is user-generated only, and agents cannot create it.
  - Only `thought` and `action` can be **ephemeral**, meaning replaced by the next activity.
  - Source: [Linear agent interaction](https://linear.app/developers/agent-interaction)
- **Signals:**
  - `stop` (human to agent, on prompt activities): "halt immediately … then emit a final `response` or `error` activity confirming the agent has stopped and reporting its current state."
  - `auth` (agent to human, on elicitation, with `signalMetadata.url`, optional `userId` and `providerName`).
  - `select` (agent to human, with `options[{label, value}]`). Users may reply in free text, so "always use an LLM to interpret the reply".
  - Source: [Linear agent signals](https://linear.app/developers/agent-signals)
- **Plans and external URLs:** Plans (technology preview) are `[{content, status}]` with `pending, inProgress, completed, canceled`. The whole plan is replaced on each update. `externalUrls[{label, url}]` lists things like PRs, and setting one prevents the "unresponsive" mark. — [Linear agent interaction](https://linear.app/developers/agent-interaction)
- **Timing:** The webhook receiver must respond within 5 seconds. "The first response must be sent within 10 seconds of receiving the `created` event." — [Linear agent interaction](https://linear.app/developers/agent-interaction); [Linear best practices](https://linear.app/developers/agent-best-practices)
- **Best practices:**
  - Move the issue to the first `started` status when work begins.
  - Set yourself as `delegate` if none is set.
  - If an automation delegated the issue, leave assignment to a human.
  - Comments "may not be reliable to read from" because they are editable, so rebuild the conversation from Agent Activities, which are "frozen-in-time snapshots of user input".
  - Source: [Linear best practices](https://linear.app/developers/agent-best-practices)

#### Letta shared memory blocks (legacy v1 API)
- **Model:** One block (`label, value, description, limit, read_only`) can be attached to many agents, and updates are visible "immediately" after the writing turn completes. The page warns that blocks "may be deprecated in the future" and points new work to shared memory repositories and MemFS. — [Letta shared memory](https://docs.letta.com/guides/agents/multi-agent-shared-memory)
- **Concurrency:** `memory_insert` (append) is safe. `memory_replace` mostly works but fails if the target string changed. `memory_rethink` is last-writer-wins. Letta recommends one owner agent for heavy edits while others append, and separate blocks for high-contention data. — [Letta shared memory](https://docs.letta.com/guides/agents/multi-agent-shared-memory)

#### OpenAI Agents SDK handoffs
- **Mechanism:**
  - Handoffs appear to the LLM as tools (`transfer_to_<agent>`).
  - `handoff()` takes `tool_name_override`, `tool_description_override`, `on_handoff`, `input_type` (model-filled metadata such as `reason` or `priority`), `input_filter` and `is_enabled`.
  - By default the receiver sees the full history. `handoff_filters.remove_all_tools` strips tool items but "does not redact tool arguments or results that appear in ordinary messages".
  - Beta `nest_handoff_history` compacts prior history into a `<CONVERSATION HISTORY>` summary block.
  - Handoffs stay "within a single run". Tool input guardrails don't apply to handoffs.
  - Source: [OpenAI handoffs docs](https://openai.github.io/openai-agents-python/handoffs/)

#### LangChain/LangGraph multi-agent patterns
- **Patterns:** Subagents (stateless per call), Handoffs (a state variable switches the active agent), Skills, Router and Custom workflow. The docs say "each agent should see only the data its task needs". Handoffs run sequentially and can't consult several domains in parallel. — [LangChain multi-agent docs](https://docs.langchain.com/oss/python/langchain/multi-agent)
- **Cost table (multi-domain):** Subagents use 5 calls and about 9K tokens. Handoffs use 7+ calls and about 14K+ tokens. Router uses 5 calls and about 9K tokens. — [LangChain multi-agent docs](https://docs.langchain.com/oss/python/langchain/multi-agent)

#### Google/LF A2A protocol: v1.0 (spring 2026), now under the Linux Foundation and AAIF
- **Spec:**
  - Latest spec version is 1.0.0 (previous: 0.3.0, 0.2.6, 0.1.0).
  - `TaskState`: `TASK_STATE_SUBMITTED, WORKING, INPUT_REQUIRED, AUTH_REQUIRED, COMPLETED, FAILED, CANCELED, REJECTED, UNSPECIFIED`.
  - Terminal states (completed, failed, canceled, rejected) accept no further messages. "Interrupted" states are input-required and auth-required.
  - Messages carry `messageId, contextId, taskId, role (ROLE_USER|ROLE_AGENT), parts, referenceTaskIds, extensions`. A `contextId` groups related tasks.
  - Push notifications are webhooks.
  - Source: [A2A spec](https://a2a-protocol.org/latest/specification/)
- **Adoption and dates:** The Linux Foundation press release of 2026-04-09 reports more than 150 supporting organizations. Sources disagree on the v1.0 month: March (Google blog) versus April 2026 (Alice Labs). The spec site links an announcement dated 2026-08-27 about joining the Agentic AI Foundation, while one blog says 2026-08-17. — [LF press](https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year); [Google OSS blog](https://opensource.googleblog.com/2026/04/a-year-of-open-collaboration-celebrating-the-anniversary-of-a2a.html); [A2A spec](https://a2a-protocol.org/latest/specification/); [Pebblous](https://blog.pebblous.ai/blog/a2a-mcp-agentic-ai-foundation-authorization/en/)
- **Secondary claim:** v1.0 reportedly added Signed Agent Cards and three bindings (JSON-RPC, gRPC, HTTP+JSON). This is not verified against the spec text. — [Alice Labs](https://alicelabs.ai/en/insights/a2a-protocol-guide-2026)

#### Chat-based coordination: Kimaki (Discord ↔ OpenCode)
- **Mapping:** A Discord channel maps to a project directory, and each thread is one OpenCode session. Claude Pro/Max can be used via OpenCode logins. — [Kimaki README](https://github.com/remorses/kimaki)
- **Interrupt vs queue:** A new message during a run **interrupts** it and cancels pending questions and permission prompts. Ending a message with `. queue` (or `/queue`) waits instead. `/btw` forks a side thread. — [Kimaki README](https://github.com/remorses/kimaki)
- **Access and security:** Access is gated by Discord roles (owner, Manage Server, a "Kimaki" role, or a blocking "no-kimaki" role). Messages from other bots are ignored, and scripts must use `kimaki send`. It recommends a dedicated Discord server. Agent-to-agent threads are not described. The repo had 1.4k stars on 2026-10-13. — [Kimaki README](https://github.com/remorses/kimaki)

### Inferences
**What Narwarp should borrow, by source:**

| Source | Borrow | Avoid |
|---|---|---|
| Agent Mail | `messages` + `message_recipients(read_ts, ack_ts)` split; importance enum; `ack_required`; thread_id = ticket key with `[KEY]` subject prefix; advisory TTL leases with conflict reporting; `macro_start_session` one-call bootstrap for small models; HumanOverseer that preempts; Markdown+frontmatter mirror for audit; cursor-by-sequence rule | per-message git commits (lock contention); a shared bearer token for all agents; no injection model |
| Claude Code teams / cross-session | delivery *between tool calls*; idle notifications carrying the final answer; structured protocol message kinds (permission, plan approval, shutdown, task assignment); `verified="false"`-style provenance attributes; "peer messages are never user consent"; loop throttles (dupe drop, ≤50 queue) | depending on undocumented `~/.claude/teams` files as Narwarp's store; auto plan approval without human review |
| Beads | atomic claim = (assignee + in_progress) in one operation; `ready` = unblocked work; `prime` re-injection; `remember`; memory decay of closed items; hash IDs | (data model is another researcher's scope) |
| Gas Town | inject mail on `UserPromptSubmit`, re-prime on `SessionStart`/`PreCompact`; pinned "hook" work + GUPP-violation detector; watchdog tiers (Witness per project, Deacon global); severity-tagged escalation to human; "seance" pointer to predecessor session | Mayor-style hierarchy for a single user |
| CCPM | mirror *selected* progress to an external tracker as an audit trail | treating unverified throughput claims as evidence |
| Linear | activity types (thought/action/elicitation/response/error), ephemeral progress, `stop/auth/select` signals, plan steps, external URLs, immutable activity log over editable comments | — |
| A2A | `input-required` / `auth-required` as explicit interrupted states; terminal states closed; `contextId` grouping; `referenceTaskIds` | full protocol stack locally |
| Letta | single-owner rule for mutable shared summaries; append-only for others | last-writer-wins shared blobs |
| OpenAI/LangGraph | typed handoff metadata (`reason`, `priority`); history filtering at handoff | in-process handoffs as the cross-session mechanism |
| Kimaki | explicit "interrupt now" vs "queue" choice for human messages; role-gated access | interrupt-by-default for agent-to-agent traffic |

### Gaps
- **Agent Teams entry schema:** I could not find official mailbox entry fields such as `from`, `text`, `timestamp` or `read`. The docs give only the path and validation behaviour. The local binary shows only the injection tag attributes and protocol message names.
- **Gas Town mail storage:** The format behind `gt mail` was not documented in the pages I could fetch. Gas City documents mail as beads-backed, and Beads has a message issue type, so Gas Town mail is probably also beads-backed (not confirmed). GitHub API access to these repos was blocked in this session.
- **Agent Mail Rust:** The license and release dates were not visible. Default reservation TTLs are undocumented.
- **Linear `stale` timeout:** The duration is not stated in the docs I read.
- **Amp Handoff date:** I did not capture the post's date.

---

## 2. Blackboard architecture and 2025–2026 evidence: what improves and what degrades shared-board / message-passing multi-agent systems

### Takeaway
A shared board helps when work is decomposable, contributions are structured and pruned, and a central validator (orchestrator, verifier or human) gates results. It hurts on sequential reasoning and when agents chat freely. Specific harms are error amplification without a hub, conformity to wrong peers, context rot from accumulated messages, duplicated work, and "excessive updates" that distract. Expect weak verification and premature or unaware termination to dominate failures, not ignored messages.

### Cited Findings
#### Classic and LLM-era blackboards
- **Classic blackboard:** "Knowledge sources" contribute to a shared "blackboard" when their constraints match its state, and a "control shell" decides which source acts. Examples are Hearsay-II (speech recognition) and BB1 (opportunistic planning with meta-level control sources that switch strategy when progress stalls). — [Wikipedia: Blackboard system](https://en.wikipedia.org/wiki/Blackboard_system)
- **bMAS (Han & Zhang, 2025-07-02, >6 mo):** All agents see the whole blackboard. A selection step picks the next agent "based on the current content of the blackboard". Rounds repeat until consensus. It was "competitive with SOTA static and dynamic MAS … using fewer tokens" (no figures in the abstract). — [arXiv 2507.01701](https://arxiv.org/abs/2507.01701)
- **Salemi et al. (Google Cloud AI Research and UMass; v1 2025-09-30, v2 2026-01-31):** A central agent *posts requests* to a blackboard, and subordinate agents "volunteer to respond based on their capabilities". The authors argue that master-slave designs need a controller with "precise knowledge of each sub-agent's capabilities". Results were 13–57% relative end-to-end improvement and up to 9% relative data-discovery F1 over the best baseline. — [arXiv 2510.01285v2](https://arxiv.org/abs/2510.01285v2)
- **Terrarium (2025-10-16, >6 mo):** Reuses the blackboard as a testbed for MAS safety, privacy and security, studying misalignment, malicious agents, compromised communication and data poisoning. — [arXiv 2510.14312](https://arxiv.org/abs/2510.14312)

#### Failure taxonomies and scaling evidence
- **MAST, "Why Do Multi-Agent LLM Systems Fail?" (v1 2025-03-17, v3 2025-10-26):** 1,600+ annotated traces across 7 frameworks, κ = 0.88, 14 failure modes. Prevalence:
  - Step repetition 15.7%
  - Reasoning-action mismatch 13.2%
  - Unaware of termination conditions 12.4%
  - Disobey task specification 11.8%
  - Incorrect verification 9.1%
  - No or incomplete verification 8.2%
  - Task derailment 7.4%
  - Fail to ask for clarification 6.8%
  - Premature termination 6.2%
  - Loss of conversation history 2.8%
  - Conversation reset 2.2%
  - Ignored other agent's input 1.9%
  - Disobey role specification 1.5%
  - Information withholding 0.85%
  - Sources: [arXiv 2503.13657](https://arxiv.org/abs/2503.13657); [HTML v3](https://arxiv.org/html/2503.13657v3)
- **MAST interventions (ChatDev):** Better role specifications gave +9.4% task success. Adding a high-level objective-verification step gave +15.6%, the largest gain. Completion rates stayed low. The authors note agents "mostly communicate in unstructured text, which causes ambiguity". They recommend structured message content, but say protocols alone (MCP/A2A) are not enough. — [HTML v3](https://arxiv.org/html/2503.13657v3)
- **Google, "Towards a science of scaling agent systems" (blog 2026-01-28; arXiv Dec 2025):** 180 configurations over 5 architectures and 4 benchmarks.
  - Independent agents amplified errors **17.2×**, while centralized coordination held this to **4.4×** because the orchestrator acts as a "validation bottleneck".
  - Centralized coordination gave **+80.9%** on parallelizable Finance-Agent.
  - **Every** multi-agent variant degraded sequential PlanCraft by **39–70%**.
  - Coordination overhead grows with tool count.
  - A predictive model picks the best architecture for 87% of held-out configurations. Version figures differ slightly (R² 0.513 vs 0.524).
  - Sources: [Google Research blog](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/); [arXiv 2512.08296v2](https://arxiv.org/abs/2512.08296v2)
- **Anthropic multi-agent Research (2025-06-13, >6 mo):**
  - Multi-agent (Opus 4 lead, Sonnet 4 subagents) beat single-agent Opus 4 by 90.2% on an internal eval.
  - Token usage explained 80% of BrowseComp variance. Agents use about 4× chat tokens, and multi-agent systems about 15×.
  - Early failures: "spawning 50 subagents for simple queries", "distracting each other with excessive updates", and duplicated searches.
  - Fixes: each subagent gets an objective, output format, tool guidance and boundaries. The lead saves its plan to Memory. Subagents write outputs to storage and pass "lightweight references".
  - Synchronous lead-waits-for-batch blocking was a bottleneck.
  - Source: [Anthropic engineering](https://www.anthropic.com/engineering/multi-agent-research-system)
- **Cognition, "Don't Build Multi-Agents" (2025-06-12, >6 mo, practitioner opinion):** "Share context, and share full agent traces, not just individual messages". "Actions carry implicit decisions, and conflicting decisions carry bad results" (the Flappy Bird example). "In 2025, running multiple agents in collaboration only results in fragile systems." It suggests a context-compression model, possibly a fine-tuned smaller model. — [Cognition blog](https://cognition.com/blog/dont-build-multi-agents)
- **"Talk Isn't Always Cheap" (Wynn, Satija, Hadfield; 2025-09-05, v2 2025-10-13):** In multi-agent debate, accuracy can fall over rounds "even when stronger models outnumber weaker ones". Models switch from correct to incorrect answers, "favoring agreement over challenging flawed reasoning". — [arXiv 2509.05396](https://arxiv.org/abs/2509.05396)
- **AgentPrune, "Cut the Crap" (2024-10-03, >6 mo):** Formalizes "communication redundancy". One-shot pruning of the spatial-temporal message graph cut tokens 28.1–72.8% with comparable results. The cost was about $5.6 versus $43.7 for state-of-the-art topologies. It also defended against agent-based adversarial attacks, with a 3.5–10.8% boost. — [arXiv 2410.02506](https://arxiv.org/abs/2410.02506)
- **Chroma "Context Rot" (2025-07-14, >6 mo):**
  - Across 18 models, performance "grows increasingly unreliable as input length grows", even on simple tasks.
  - A single distractor lowers accuracy, and coherent haystacks hurt more than shuffled ones.
  - On LongMemEval, focused prompts (~300 tokens) far outperformed full ones (~113k tokens).
  - Source: [Chroma research](https://www.trychroma.com/research/context-rot)
- **Anthropic C compiler (2026-02-05, >6 mo):**
  - 16 parallel Opus 4.6 agents, about 2,000 sessions over two weeks, about 2B input tokens and 140M output tokens, "just under $20,000".
  - Coordination ran through a shared bare git repo. An agent claims a task by writing a lock file into `current_tasks/`, and git sync forces a loser to choose another task. "No orchestrator and no other inter-agent communication method."
  - Agents kept READMEs and progress files, plus docs of failed approaches.
  - Failure: on one monolithic task (the Linux kernel) all agents hit the same bug and overwrote each other. The fix was a known-good oracle (GCC) to split the work.
  - Harness lessons: print few lines, log to files, use an "ERROR" grep tag, offer a `--fast` 1–10% sample to counter "time blindness", and make verifiers nearly perfect.
  - Source: [Anthropic engineering](https://www.anthropic.com/engineering/building-c-compiler)

### Inferences
- **Coordination fabric, not debate forum:** Narwarp's boards should be a coordination fabric (claims, questions, decisions, handoffs, results) and not a place where agents reason together. Evidence for this: sequential-task degradation (Google), conformity (Wynn et al.), and context fragmentation (Cognition).
- **Central validation:** Keep a "validation bottleneck" (Narwarp daemon rules, a verifier step, or the human) between agent claims and accepted state. Independent unchecked agents amplified errors about 4× more than centralized ones.
- **Default to quiet:** Post at state transitions only. Make progress updates ephemeral and superseding (like Linear). Prune redundant messages before they enter any agent's context. AgentPrune's 28–73% token savings and Anthropic's "excessive updates" failure both support this.
- **Verification is the biggest lever:** Require evidence references (test command, exit code, commit) on `result` and `decision` messages. Verification failures (8.2% + 9.1%) plus premature or unaware termination (6.2% + 12.4%) outweigh "ignored input" (1.9%) and "withholding" (0.85%) in MAST.
- **Design the delegation:** Use one owner per ticket, volunteer-on-request (Salemi) for optional help, and lock-file-style claims (C compiler) instead of an LLM "mayor" deciding everything.

### Gaps
- **Personal-harness benchmarks:** I found no benchmark for human-plus-heterogeneous-agent boards in a personal harness. All evidence is from coding, research or QA benchmarks, or from single-org case studies.
- **MAST venue:** NeurIPS 2025 is not confirmed on the arXiv page.
- **Quantitative blackboard comparisons:** bMAS and Terrarium abstracts give no figures. I did not read their full papers.

---

## 3. What must a good handoff contain? (with recommended template)

### Takeaway
A good handoff is a *structured, verified state snapshot with pointers*, not a narrative. It should contain the following:
- the goal and acceptance criteria, with pass/fail status
- what was verified and how
- the commits and branch
- the exact next step
- decisions with rationale
- failed approaches
- open questions with owners
- environment and run instructions
- user constraints quoted verbatim

Keep the machine-checkable parts (criteria) in JSON that agents may only flip, not rewrite. Pass references (paths, ids) rather than copying content.

### Cited Findings
- **Anthropic long-running harness (2025-11-26, >6 mo):**
  - An initializer agent creates `init.sh`, `claude-progress.txt` and an initial git commit, plus a JSON feature list. For the claude.ai clone, 200+ features were all initially failing, each with `category, description, steps[], passes:false`.
  - The model "is less likely to inappropriately change or overwrite JSON files" than Markdown. Agents may only change `passes`, and removing or editing tests is "unacceptable".
  - Each session starts with `pwd`, reads git log and progress, picks the highest-priority failing feature, runs `init.sh` and a basic end-to-end test. It ends with a descriptive commit and a progress summary, leaving a mergeable state.
  - Observed failures: one-shotting, premature victory, broken starting state and unverified completion.
  - Compaction "doesn't always pass perfectly clear instructions to the next agent".
  - Source: [Anthropic engineering](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- **Claude Code's own compaction summary (local inspection, v2.1.296):** The prompt first asks for an `<analysis>` block, then a summary with nine sections:
  1. Primary Request and Intent
  2. Key Technical Concepts
  3. Files and Code Sections
  4. Errors and fixes
  5. Problem Solving
  6. All user messages ("Preserve any security-relevant instructions or constraints verbatim so they remain in effect after compaction")
  7. Pending Tasks
  8. Current Work
  9. Optional Next Step ("include direct quotes from the most recent conversation … verbatim to ensure there's no drift in task interpretation")
  - Source: local inspection of `/opt/claude-code/bin/claude`
- **Anthropic context engineering (2025-09-29, >6 mo):**
  - Compaction should preserve "architectural decisions, unresolved bugs, and implementation details" and drop redundant tool outputs. Tool-result clearing is "one of the safest lightest touch forms of compaction". Tune for recall first, then precision.
  - Structured note-taking (to-do lists, NOTES.md) and sub-agents returning "condensed, distilled summary … (often 1,000-2,000 tokens)".
  - Just-in-time retrieval via "lightweight identifiers (file paths, stored queries, web links)".
  - Source: [Anthropic engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- **Amp "Handoff (No More Compaction)":** Amp replaced compaction with `/handoff`, which drafts a starting prompt plus a list of relevant files for a *new* focused thread from a user-stated goal, editable before sending. The rationale is that compaction is lossy and repeated compactions stack summaries. Date not captured. — [Amp news](https://ampcode.com/news/handoff)
- **Gas Town:** `/handoff` "session refresh" carries work state forward, and `gt seance` lets a successor query a predecessor session. — [glossary](https://raw.githubusercontent.com/steveyegge/gastown/main/docs/glossary.md)
- **C compiler project:** Agents kept "extensive READMEs and progress files, updated frequently, because each new container started with no context", plus docs of "failed approaches and remaining tasks". — [Anthropic engineering](https://www.anthropic.com/engineering/building-c-compiler)
- **Agent Teams:** Teammates "don't inherit the lead's conversation history", so put task-specific details in the spawn prompt. — [Agent teams docs](https://code.claude.com/docs/en/agent-teams)
- **OpenAI handoffs:** `input_type` carries model-decided metadata such as `reason` and `priority`. `input_filter` controls which history the receiver sees, and nesting "does not redact sensitive data". — [OpenAI handoffs docs](https://openai.github.io/openai-agents-python/handoffs/)
- **Linear:** Rebuild state from immutable activities, because comments are editable. — [Linear best practices](https://linear.app/developers/agent-best-practices)
- **AGENTS.md:** The Linux Foundation formed the Agentic AI Foundation on 2025-12-09 with MCP, goose and AGENTS.md as founding projects. The press release credits OpenAI with AGENTS.md, which "gives AI coding agents a consistent source of project-specific guidance". — [LF press](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation); [InfoQ](https://infoq.com/news/2025/12/agentic-ai-foundation/)
- **Beads:** `bd prime` re-injects workflow context and memories, and `bd remember` stores project facts. — [Beads README](https://github.com/steveyegge/beads)

### Inferences
**Layering:** AGENTS.md/CLAUDE.md holds *durable project rules*. The ticket handoff holds *task state*. The board holds the *conversation*. The digest holds *recent board state*. Never mix durable rules into handoffs, because they bloat and drift.

**Recommended `HANDOFF.md` template.** Store one per ticket, versioned. The JSON sidecar `handoff.criteria.json` holds the acceptance criteria.

```markdown
---
ticket: NW-142            # Narwarp ticket; linked: [linear:ENG-881, linear:ENG-882]
handoff_id: 01JA9X...     # ULID; version: 7
from: claude:nw142-impl   # principal; session_id: <claude session id> (for "seance"/resume)
to: any | @role:reviewer | @human
state: in_progress | blocked | ready_for_review | done
branch: nw/142-oauth  head: 3f2c9ab  worktree: ~/wt/nw-142
claims_released: true     # ticket + path leases
created: 2026-10-13T14:02:00Z
---
## 1. Goal & definition of done
- Goal (verbatim from ticket): ...
- Acceptance criteria → see handoff.criteria.json (3/5 passing)

## 2. Current state (≤3 sentences) and how it was verified
- Verified: `cargo nextest run -p auth` → exit 0 (14:01Z); manual: login flow OK in dev server
- NOT verified: token refresh on expiry

## 3. Done this session
- 3f2c9ab  Add PKCE flow to auth client
- 91be0d2  Store refresh token in keychain

## 4. Next step (exact)
- Next: implement refresh-on-401 in `crates/auth/src/client.rs:212`
- First command: `./script/dev && cargo nextest run -p auth refresh_`

## 5. Decisions (with rationale; link board msgs)
- D1 Use system keychain, not file store (msg 01JA9T...); alternatives rejected: ...

## 6. Failed approaches / dead ends (do not retry)
- Tried reqwest middleware for retry → conflicts with streaming bodies (see msg 01JA9V...)

## 7. Open questions (owner, blocking?)
- Q1 @human: OK to require re-login after 30 days? (blocking for criterion C4; msg 01JA9W...)

## 8. Environment & gotchas
- Needs `OAUTH_CLIENT_ID` (in 1Password item "nw-dev"); dev server port 7300; tests flaky if keychain locked

## 9. Key files / locations
- crates/auth/src/client.rs (refresh logic), crates/auth/tests/refresh.rs

## 10. Risks / unknowns
- ...

## 11. User constraints (verbatim; security-relevant first)
- "Never push to main without asking me." (user, 2026-10-12, msg 01JA7...)
```

```json
{"ticket":"NW-142","criteria":[
 {"id":"C1","description":"PKCE login succeeds","verify":"cargo nextest run -p auth pkce_","passes":true,"evidence":"run 14:01Z exit 0"},
 {"id":"C4","description":"Refresh on 401","verify":"cargo nextest run -p auth refresh_","passes":false,"evidence":null}]}
```

**Handoff rules:**
- Agents may flip `passes` only with an `evidence` value. Only the human or a ticket-creation step can add or remove criteria (Anthropic's "unacceptable to remove or edit tests").
- The receiver must `ack` the handoff before starting.
- A handoff is required when a claim holder stops, before compaction if possible, on reassignment, and when blocked longer than a threshold.
- Prefer Amp-style "new focused session from handoff" over repeated compaction for long tickets. Keep the predecessor's session id for Gas Town-style "seance" questions.

### Gaps
- I found no controlled study comparing handoff formats (JSON vs Markdown, length, sections) on downstream success. The Anthropic JSON-vs-Markdown claim is a qualitative observation.
- The Amp post date and any outcome data were not available.

---

## 4. Heterogeneous teams: what small local (Ollama) models do reliably vs poorly in triage, routing, digests and watchdog roles

### Takeaway
Evidence supports giving local models narrow, schema-constrained jobs:
- classification and routing
- extractive or faithful summarization into digests
- deterministic-rule-assisted watchdog checks
- quarantined reading of untrusted external content

Evidence warns against small models for multi-step tool use (reliability drops sharply below about 4–9B), long-context synthesis (context rot, plus Ollama's small default context on smaller-VRAM machines), and any consequential decision. Validate every local-model output with schemas and cited-id checks, and escalate to Claude or the human on failure.

### Cited Findings
- **NVIDIA "Small Language Models are the Future of Agentic AI" (2025-06, position paper, >6 mo):** SLMs (roughly under 10B) are "powerful enough, better suited, and cheaper" for many repetitive, narrow agentic subtasks. Heterogeneous multi-model systems are the natural fit when general conversation is needed. The paper includes an LLM-to-SLM conversion algorithm. It is a position paper, not a benchmark. Blog-reported "10–30× cheaper" figures were not confirmed in the abstract. — [arXiv 2506.02153 (PDF)](https://arxiv.org/pdf/2506.02153); [HF page](https://huggingface.co/papers/2506.02153)
- **Arch-Router (Katanemo, 2025-06-19, >6 mo):** A "compact 1.5B model" maps queries to user-defined domain and action preferences to pick a model. It claims SOTA preference-matching that outperforms top proprietary models (no numbers in the abstract), and new routes can be added without retraining. — [arXiv 2506.16655](https://arxiv.org/abs/2506.16655)
- **Vectara hallucination leaderboard (updated 2026-09-22; HHEM-2.3, document summarization at temperature 0):**
  - Small open models are competitive on *faithful summarization*: Phi-4 (14B) 3.7%, Gemma-3-12b 4.4%, Qwen3-8B 4.8%, Qwen3-14B 5.4%, Qwen3-4B 5.7%, Gemma-3-4b 6.4% (answer rate only 67.3%), Ministral-3B 7.3%. Outlier: Phi-4-mini 23.5%.
  - Frontier comparisons: GPT-5.4-nano 3.1%, Claude Haiku 4.5 9.8%, Sonnet 4.6 10.6%, Opus 4.6 12.2%.
  - This measures strict consistency with the source only, not overall quality.
  - Source: [vectara/hallucination-leaderboard](https://github.com/vectara/hallucination-leaderboard)
- **Tool calling (secondary guide, last reviewed 2026-07-18, citing BFCL-v4):**
  - Qwen3.5 scores: 27B 0.685, 9B 0.661, 4B 0.503, 2B 0.436. The guide rates 9B "usable", 4B "marginal" and ≤2B "unreliable", and says "reliability falls off a cliff below about 4 billion parameters".
  - Native tool templates help choose the right call, while grammar-constrained decoding only guarantees shape. Use both.
  - This is a vendor-style guide and was not cross-checked against the leaderboard.
  - Source: [D-Central](https://d-central.tech/local-llm-agent-capability/)
- **Ollama default context:** It depends on VRAM: "< 24 GiB VRAM: 4k context", "24-48 GiB: 32k", "≥ 48 GiB: 256k". Ollama recommends "at least 64000 tokens" for "agents, and coding tools". Set it with `OLLAMA_CONTEXT_LENGTH` or the app slider, and check with `ollama ps`. — [Ollama docs: context length](https://docs.ollama.com/context-length)
- **Context rot hits small models too:** Qwen3 models were among the 18 tested, and degradation with length and distractors was universal. — [Chroma research](https://www.trychroma.com/research/context-rot)
- **Agent Mail macros:** Agent Mail ships macros (`macro_start_session` etc.) explicitly "for speed or smaller models". — [mcp_agent_mail README](https://github.com/Dicklesworthstone/mcp_agent_mail)
- **Gas Town watchdogs:** Watchdog duties are separate roles: Witness detects stuck agents, Deacon patrols, and Dogs/Boot do triage. Hooked work with no progress is flagged as a "GUPP Violation". — [Gas Town README](https://github.com/steveyegge/gastown); [glossary](https://raw.githubusercontent.com/steveyegge/gastown/main/docs/glossary.md)
- **Cognition's compression model:** Cognition suggests a dedicated "context compression model", noting "fine-tuning a smaller model may help" but that it is hard to get right (practitioner opinion, >6 mo). — [Cognition blog](https://cognition.com/blog/dont-build-multi-agents)
- **Centralized validation:** It contains error amplification (4.4× vs 17.2×), which argues for a validator around weaker models. — [Google Research blog](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/)

### Inferences
**Recommended local-model duties:** Run each with JSON-schema-constrained output, a deterministic validator, and an escalation path.

| Duty | Fit | Guardrail |
|---|---|---|
| Ticket triage: NL prompt → {project, mode work/life, type, priority, Linear-candidate} | Good (routing/classification; Arch-Router evidence) | enum-constrained output; low-confidence → ask human in TUI |
| Board digests (ticket ≤400 tokens; project ≤800) | Good for *faithful* summaries (Qwen3-8B ≈4.8% on Vectara) | every bullet must cite existing message ids; structured items (decisions/approvals/claims) copied, not paraphrased; fallback to extractive header list |
| Watchdog (stuck claim, unanswered blocking question, loops, repetition, "done" claims without evidence) | Good if rules do the detecting and the model only classifies or wordsmiths | daemon rules first; model can *flag* or *nudge*, never reassign or approve |
| Quarantined reader for email/Discord/Linear/GitHub inbound | Good (Dual-LLM pattern; see Q6) | no tools; outputs fixed fields; output still labelled untrusted |
| Notification text, titles, commit-message drafts | Good | cosmetic only |
| Multi-step tool use, coding, planning, cross-ticket synthesis over long boards, anything consequential | Poor | route to Claude; human approval for consequential actions |

**Mac-specific gotcha:** On Apple Silicon, "VRAM" is the GPU-allocatable share of unified memory. A 16–32 GB Mac likely gets the **4k default context**, which would silently cut board text fed to a digest model (inferred from the Ollama table). Narwarp should set `OLLAMA_CONTEXT_LENGTH` explicitly (e.g., 32k for digests). It should also count tokens before calling, and chunk hierarchically by thread, then ticket, then project.

**Model size:** Prefer models of 8B or more (e.g., Qwen3-8B / Qwen3.5-9B class) for tool-calling roles, and 4B for pure classification. The other researcher covers model selection.

### Gaps
- I found no rigorous study of small local models specifically as *board summarizers*, *watchdogs* or *routers* inside coding-agent harnesses. The evidence is from adjacent benchmarks (summarization faithfulness, function calling, routing).
- Quantization effects on tool-calling reliability were not quantified in the sources I read.
- No primary BFCL leaderboard page was fetched. Figures are from a secondary guide.

---

## 5. Delivery mechanics: push vs pull, and how to avoid derailing a working agent

### Takeaway
Use **pull for bodies and push for small headers, at safe boundaries**. Concretely:
- Inject a compact unread summary at `SessionStart` and `UserPromptSubmit`, and, rate-limited, after tool calls.
- Deliver only urgent or human-addressed items mid-turn, and only *between* tool calls.
- Wake idle sessions with a one-line doorbell.
- Let the receiver fetch full bodies on demand.

Hook-injected text should state facts, not commands.

### Cited Findings
- **Claude Code hooks: injection points.** `SessionStart`, `UserPromptSubmit`, `UserPromptExpansion` and `PostModelSwitch` accept plain stdout or `hookSpecificOutput.additionalContext`. `PreToolUse`, `PostToolUse`, `PostToolUseFailure` and `PostToolBatch` accept `additionalContext`, which lands next to the tool result. `Stop`/`SubagentStop` accept `additionalContext`, and the conversation continues. `Notification` cannot inject. Injected text is wrapped in a system reminder. — [Hooks docs](https://code.claude.com/docs/en/hooks)
- **Hooks: limits.**
  - Each injection is capped at **10,000 characters**. Overflow goes to a file plus a 2,000-character preview, and Claude isn't told to read the file.
  - Several hooks' contexts are all delivered.
  - `<system-reminder>` tags are escaped.
  - "Write facts, not imperatives". Text framed as system commands "can trigger Claude's prompt-injection defenses".
  - On `--resume`, mid-session hook text is replayed, not re-run.
  - Source: [Hooks docs](https://code.claude.com/docs/en/hooks)
- **Hooks: blocking and wake.** `Stop` exit 2 blocks stopping and feeds stderr back as the reason. `asyncRewake: true` runs a background hook that "wakes Claude on exit code 2", which is the only documented way for a background hook to wake a session. — [Hooks docs](https://code.claude.com/docs/en/hooks)
- **Cross-session messaging:** Messages arrive "between tool calls … a running tool is never interrupted", and an idle session starts a new turn. Scripts can post to `CLAUDE_CODE_MESSAGING_SOCKET`. Throttles: dupe drop, ≤50 queued, burst refusal. — [Cross-session messaging docs](https://code.claude.com/docs/en/cross-session-messaging)
- **Agent Teams:** Delivery is automatic with no lead polling, and idle notifications include the final answer. `TeammateIdle` exit 2 keeps a teammate working. — [Agent teams docs](https://code.claude.com/docs/en/agent-teams)
- **Gas Town:** Mail is injected on every `UserPromptSubmit` (`gt mail check --inject`), and context is re-primed on `SessionStart` and `PreCompact`. Runtimes without hooks get typed-in `gt prime` / `gt mail check --inject`. `gt nudge` is a real-time bypass. — [HOOKS.md](https://raw.githubusercontent.com/steveyegge/gastown/main/docs/HOOKS.md); [README](https://github.com/steveyegge/gastown)
- **Agent Mail is pure pull:** `fetch_inbox` with `unread_only` or `urgent_only` and `include_bodies=false` reduces tokens. Cursor-based inbox events exist, and ATC advisories have a 300 s cooldown per agent. The SKILL says only "Check Inbox Periodically". — [README](https://github.com/Dicklesworthstone/mcp_agent_mail); [SKILL.md](https://cdn.jsdelivr.net/gh/dicklesworthstone/mcp_agent_mail@main/SKILL.md); [Rust README](https://github.com/Dicklesworthstone/mcp_agent_mail_rust)
- **Human message preemption:** Kimaki **interrupts** a running session on a new human message by default (cancelling pending prompts), with `. queue` to defer. Agent Mail's HumanOverseer preamble says "pause, prioritize, then resume". — [Kimaki README](https://github.com/remorses/kimaki); [mcp_agent_mail README](https://github.com/Dicklesworthstone/mcp_agent_mail)
- **Distraction:** Excessive updates distracted agents in Anthropic's Research system. — [Anthropic engineering](https://www.anthropic.com/engineering/multi-agent-research-system)
- **Linear liveness:** Linear requires a first activity within 10 s and supports ephemeral progress that replaces itself, avoiding feed spam. — [Linear agent interaction](https://linear.app/developers/agent-interaction)

### Inferences
**Delivery tiers for Narwarp:**

| Tier | Examples | Delivery | Mechanism |
|---|---|---|---|
| P0 interrupt | `signal:stop` from human; `urgent` from @human; security alert | between tool calls (never mid-tool); for idle sessions, doorbell | daemon → `CLAUDE_CODE_MESSAGING_SOCKET` one-liner (session launched with `crossSessionInbound: accept` in its `--settings`) and/or `PostToolUse` hook bypassing rate limit |
| P1 addressed | question/answer/approval result addressed to this agent; handoff to it | next `PostToolUse` (rate-limited, e.g., ≤1 per 3–5 min) or next `UserPromptSubmit`/`Stop` | `nw board check --inject --only addressed` |
| P2 normal | status from teammates on same ticket; decisions on the project | at `SessionStart`/`UserPromptSubmit` only, as headers | same hook, headers only |
| P3 FYI | low-importance notes, other tickets | never pushed; appears in digest | digest |

**Anti-derailment rules:**
1. Inject *headers* (id, kind, from, subject, ≤120 chars), never bodies. The agent fetches bodies with `nw_read`.
2. Phrase injections as facts, e.g., "Board NW-142: 1 unread question from @human (01JA…)". Never write "You must…".
3. Batch and dedupe by deriving content hashes per recipient and window.
4. The `Stop` gate fires at most once per stop attempt to avoid loops.
5. Humans choose "send now (interrupt)" or "queue" (Kimaki).
6. Ollama agents run inside Narwarp's own loop, so the loop checks the inbox at each step boundary.

### Gaps
- I found no controlled study of interrupt timing (mid-turn vs boundary) on coding-agent success. The recommendations rest on vendor design choices (Claude Code, Kimaki) and the Anthropic "excessive updates" observation.
- Whether `PostToolUse` injection frequency measurably degrades long-run performance is untested publicly.

---

## 6. Security: prompt injection through shared boards; provenance and trust labels; approval gates

### Takeaway
Treat every board message as data with provenance, never as instructions or consent. Separate capabilities so that no single agent session combines untrusted input, private data and external communication without human approval (Rule of Two / lethal trifecta).

Route external content (email, Discord, Linear, GitHub) through a quarantined, tool-less reader (Dual-LLM). Let the daemon stamp and sign authorship. Make approvals a human-only TUI action that agents cannot forge or relay.

### Cited Findings
- **Prompt Infection (Lee & Tiwari, 2024-10-09, >6 mo):** Malicious prompts "self-replicate across interconnected agents, behaving much like a computer virus". "Multi-agent systems are highly susceptible" even without fully public communication. "LLM Tagging" combined with existing safeguards "significantly mitigates infection spread". — [arXiv 2410.07283](https://arxiv.org/abs/2410.07283)
- **Paul & Nandy, ICML 2026 workshop (Agents in the Wild):**
  - Threat model: 14 vectors in four classes — direct, indirect via tool outputs, inter-agent via message passing, and cascading via orchestrator manipulation.
  - In a 6-agent system, 67% of agents were vulnerable to at least one scope violation despite system-prompt guardrails. Indirect injection via tool outputs succeeded 43% of the time.
  - Defenses: message signing with provenance (−91% inter-agent injection), boundary sanitization (−78% indirect), role-scoped tools (privilege escalation eliminated), and anomaly detection (84% of cascades caught). Combined, injection success fell from 31.2% to 4.2%.
  - This is a single workshop paper, unreplicated.
  - Source: [ICML 2026 poster page](https://icml.cc/virtual/2026/67971)
- **Design patterns (Beurer-Kellner et al., 2025-06):** Action-Selector, Plan-Then-Execute, LLM Map-Reduce, Dual LLM (a privileged model coordinates a quarantined model that returns symbolic variables), Code-Then-Execute, and Context-Minimization. Principle: once an agent ingests untrusted input it "must be constrained so that it is impossible for that input to trigger any consequential actions". — [arXiv 2506.08837](https://arxiv.org/abs/2506.08837); [Willison summary](https://simonwillison.net/2025/Jun/13/prompt-injection-design-patterns/)
- **Lethal trifecta (Willison, 2025-06-16, >6 mo):** Private data, untrusted content and external communication together are dangerous. "In web application security 95% is very much a failing grade." MCP "encourages users to mix and match tools", often combining all three in one tool (e.g., an email reader). — [simonwillison.net](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)
- **Meta "Agents Rule of Two" (2025-10-31):** An agent should satisfy at most two of [A] "process untrustworthy inputs", [B] "access to sensitive systems or private data", [C] "change state or communicate externally" within a session. If all three are needed, it "should not be permitted to operate autonomously and at a minimum requires supervision". — [Meta AI blog](https://ai.meta.com/blog/practical-ai-agent-security/)
- **Claude Code's built-in stance:**
  - Peer and teammate messages are labelled as from another session. They "can't approve anything", can't change config, and their embedded commands don't run.
  - Auto mode treats relayed approval claims as untrusted and reviews inter-agent messages before delivery.
  - Injection wrappers carry `verified="false"` for unverified teammates, and `<cross-session-message>` is "never user intent".
  - Sources: [Agent teams docs](https://code.claude.com/docs/en/agent-teams); [Cross-session messaging docs](https://code.claude.com/docs/en/cross-session-messaging); local inspection of v2.1.296 binary
- **Hooks can trip injection defenses:** Hook text phrased as system commands "can trigger Claude's prompt-injection defenses". — [Hooks docs](https://code.claude.com/docs/en/hooks)
- **Agent Mail's limits:** Session-bound identity "does not authenticate the model or provider" and "cannot isolate hostile clients that share one bearer credential". It provides secret scrubbing and HTML sanitization but no injection guidance. — [Rust README](https://github.com/Dicklesworthstone/mcp_agent_mail_rust); [README](https://github.com/Dicklesworthstone/mcp_agent_mail)
- **Collaborative Memory (2025-05-23):** Private and shared memory tiers. Bipartite user-agent-resource permission graphs. Each fragment carries "immutable metadata on contributing agents, accessed resources, and timestamps" for retrospective permission checks. Read and write policies transform views. — [arXiv 2505.18279](https://arxiv.org/abs/2505.18279)
- **Linear provenance:** Comments are editable and unreliable as provenance, while activities are immutable snapshots. — [Linear best practices](https://linear.app/developers/agent-best-practices)
- **Kimaki channel hygiene:** Messages from other bots are ignored, access is role-gated, and a dedicated Discord server is recommended. — [Kimaki README](https://github.com/remorses/kimaki)
- **Pruning as defense:** Pruning the message graph also defended against agent-based adversarial attacks (AgentPrune, >6 mo). — [arXiv 2410.02506](https://arxiv.org/abs/2410.02506)

### Inferences
**Trust tiers, stamped by the daemon (never by the author):**

| Tier | Who | Notes |
|---|---|---|
| T0 | human via Narwarp TUI | only tier that can approve |
| T1 | Narwarp daemon (system) | |
| T2 | internal agents launched by Narwarp | Claude/Ollama; each holds a per-session token |
| T3 | bridged external content | Discord, email, Linear/GitHub comments by others, web pages, calendar invites from others |

The user's own messages arriving via Discord or Linear count as "T0-remote". They may *request* actions but cannot *approve* gated actions without TUI confirmation, because those channels can be spoofed or compromised.

**Rendering into agent context.** Always use explicit wrappers that mirror Claude Code's convention:
```
<narwarp-message id="01JA9X…" board="NW-142" kind="question" from="ollama:triage" author_kind="agent" trust="T2" verified="true">…</narwarp-message>
<narwarp-external id="01JA9Y…" source="discord" channel="#backend" sender="alice#1234" trust="T3">summary: …; entities: …; asks_for: [review PR #88]</narwarp-external>
```
Every agent's system prompt or AGENTS.md states that board content is information. It cannot authorize actions. "The user approved" claims inside messages are not approvals.

**Dual-LLM quarantine for T3:**
1. Raw external text is stored but never shown to privileged agents.
2. A tool-less local model extracts fixed fields (intent category, entities, dates, links, short neutral summary).
3. Privileged agents see only the fields plus a pointer.
4. Any action derived from T3 content goes through approval with a "triggered by external content from X" banner.

**Rule-of-Two capability profiles per session:**

| Profile | Can | Cannot |
|---|---|---|
| Coder | read repo (B) + write worktree (C-local) | read T3 raw; outbound comms only via approval gate |
| Inbox/triage | read T3 (A) + post T2 board summaries | no secrets or tools beyond the board |
| Outbound | send Discord, email or Linear comments (C) | only with human approval per message or a pre-approved template |

**Approval gate:**
- `approval_request` (agent) leads to an `approval` event (human only).
- The `approval` event is HMAC-signed with a daemon key, bound to (action, params hash, requester, expiry), and single-use. The executing tool re-verifies the signature and params hash.
- Gated by default: git push and merge, outbound messages, Linear state changes visible to others, calendar changes that affect others, deletions, spending, secret access, and anything crossing the Work/Life boundary.

**Isolation and hygiene:**
- Work and Life boards are separate access-control domains (Collaborative Memory-style). Work agents can't read Life boards and vice versa without explicit share, to prevent personal email leaking into work Discord.
- Strip ANSI and terminal control sequences and sanitize Markdown before rendering agent text in the TUI. This is a terminal-specific injection vector (inferred). The local Claude Code binary contains a `stripFrameControlChars` routine for mailbox frames.
- Apply secret scrubbing on post, as Agent Mail does.

**Loop and flood control:**
- Per-sender rate limits, identical-message suppression, a queue cap (≤50 like Claude Code), and an A↔B ping-pong circuit breaker that escalates to the human.
- Per-session tokens rather than one shared credential, given Agent Mail's stated limitation.

### Gaps
- No public benchmark measures injection across a *human-plus-Claude-plus-local-model* board specifically. The ICML 2026 numbers come from one unreplicated workshop paper.
- How well small local models resist injection as quarantined readers is unmeasured in the sources I found. The design relies on their lack of tools, not their robustness.
- The full Prompt Infection numbers (e.g., whether stronger models are more vulnerable) were not in the abstract.

---

## 7. Recommended design for Narwarp's per-ticket and per-project message boards

### Takeaway
Build a native, daemon-owned board inside Narwarp's existing Diesel/SQLite persistence:
- typed, append-only messages with per-recipient read/ack state
- ticket and project boards with threads keyed by ticket id
- daemon-stamped provenance and trust tiers
- TTL claims with heartbeats
- versioned HANDOFF.md documents with JSON acceptance criteria
- local-model digests with cited ids
- human-only signed approvals

Agents access it through a small MCP server and an equivalent `nw` CLI. Claude Code sessions get a few hooks: prime on `SessionStart`, headers on `UserPromptSubmit` and rate-limited `PostToolUse`, a handoff gate on `Stop`, and a doorbell over the cross-session socket for urgent items. Do not adopt Claude Code's undocumented team files or Agent Mail's repo-centric model as the system of record. Borrow their semantics instead.

### Cited Findings
- **Narwarp persistence:** The app already uses Diesel with SQLite (`app/Cargo.toml:259`: `diesel = { workspace = true, features = ["sqlite", "chrono"] }`). The `crates/persistence` crate has 139 Diesel migrations. — local inspection of `/home/user/narwarp/app/Cargo.toml` and `/home/user/narwarp/crates/persistence/migrations/`
- **Prior art for the storage split:** Agent Mail's Git-plus-SQLite FTS5 dual store hit git lock contention and SQLite pool exhaustion under many agents. The Rust rewrite fixed this with batching and backpressure, and its commit coalescer turned 100 writes into 11 commits. — [Rust README](https://github.com/Dicklesworthstone/mcp_agent_mail_rust)
- **Separate task state from conversation:** "Use Beads for task status and dependencies, and Mail for conversation and audit." — [mcp_agent_mail README](https://github.com/Dicklesworthstone/mcp_agent_mail)
- **Team files are ephemeral runtime state:** Claude Code's team config is deleted at session end and "don't edit it by hand or pre-author it". — [Agent teams docs](https://code.claude.com/docs/en/agent-teams)
- **Injection limits:** 10,000-character cap per injection, plus the facts-not-imperatives guidance. — [Hooks docs](https://code.claude.com/docs/en/hooks)
- **Doorbell transport:** Scripts can post into a session via `CLAUDE_CODE_MESSAGING_SOCKET`/`TOKEN`, delivered between tool calls. — [Cross-session messaging docs](https://code.claude.com/docs/en/cross-session-messaging)
- **Activity vocabulary and signals** from Linear; **interrupted states** from A2A; **failure mode prevalence** from MAST (stated in Q1/Q2). — [Linear](https://linear.app/developers/agent-interaction); [A2A](https://a2a-protocol.org/latest/specification/); [MAST](https://arxiv.org/html/2503.13657v3)

### Inferences

#### 7.1 Design principles (from Q1–Q6)
1. **Separate state from talk.** Tickets, claims, criteria and approvals are authoritative rows. The board is an append-only event log that references them.
2. **Typed, quiet, verifiable messages.** Use a small kind enum. Post on state transitions. Make progress ephemeral. Results and decisions carry evidence refs.
3. **Hub validation.** The daemon validates, stamps, rate-limits and routes, and the human approves. There is no free-for-all peer chat and no debate-to-consensus.
4. **Pull bodies, push headers at boundaries.**
5. **Provenance everywhere.** Trust tiers are stamped by the daemon, untrusted content is quarantined, and approvals are human-only and signed.

#### 7.2 Storage
- **Database:** Tables live in Narwarp's SQLite via Diesel, or in a separate `boards.db` to isolate WAL contention from UI state. Use WAL mode.
- **Single writer:** All writes go through the Narwarp daemon (`narwarpd`). This avoids forged authorship and SQLite contention.
- **Search:** FTS5 via raw `sql_query`, since Diesel has no FTS DSL.
- **Mirror (derived, read-only, coalesced writes):** `~/.narwarp/boards/<project>/<ticket>/BOARD.md` (digest plus last N headers) and `HANDOFF.md` plus `handoff.criteria.json`. These exist for human audit, git-diffability and agents without MCP. There are no per-message commits.

```sql
-- principals: humans, agents, bridges, system
CREATE TABLE principals(id TEXT PRIMARY KEY, kind TEXT NOT NULL CHECK(kind IN('human','claude','ollama','bridge','system')),
  name TEXT UNIQUE NOT NULL, role TEXT, model TEXT, session_id TEXT, worktree TEXT, mode_scope TEXT CHECK(mode_scope IN('work','life','both')),
  trust_tier INTEGER NOT NULL, token_hash TEXT, status TEXT, created_at INTEGER, last_seen_at INTEGER);
CREATE TABLE boards(id TEXT PRIMARY KEY, scope TEXT NOT NULL CHECK(scope IN('ticket','project','mode','global')),
  ticket_id TEXT, project_id TEXT, mode TEXT CHECK(mode IN('work','life')), title TEXT, created_at INTEGER);
CREATE TABLE messages(seq INTEGER PRIMARY KEY AUTOINCREMENT, id TEXT UNIQUE NOT NULL,            -- ULID
  board_id TEXT NOT NULL REFERENCES boards(id), thread_id TEXT NOT NULL, parent_id TEXT,
  kind TEXT NOT NULL, author_id TEXT NOT NULL REFERENCES principals(id),
  importance TEXT NOT NULL DEFAULT 'normal' CHECK(importance IN('low','normal','high','urgent')),
  requires TEXT NOT NULL DEFAULT 'none' CHECK(requires IN('none','ack','answer','approval')),
  state TEXT,                       -- open|answered|accepted|superseded|approved|rejected|expired|withdrawn
  subject TEXT NOT NULL,            -- ≤120 chars, used for header injection
  body_md TEXT,                     -- ≤4,000 chars; longer → artifact file
  payload_json TEXT,                -- kind-specific, schema-validated
  refs_json TEXT,                   -- [{type:commit|file|pr|linear|github|msg|url|test_run, id, label}]
  provenance_json TEXT NOT NULL,    -- {trust:"T2", origin:"internal|external", source, channel, sender_ext, quarantined, derived_from:[ids]}
  ephemeral INTEGER NOT NULL DEFAULT 0, supersedes TEXT,
  created_at INTEGER NOT NULL, expires_at INTEGER, sig TEXT NOT NULL);   -- HMAC(daemon_key, canonical row)
CREATE TABLE recipients(message_id TEXT, principal_id TEXT, via TEXT CHECK(via IN('to','cc','mention','role','owner','subscription')),
  delivered_at INTEGER, read_at INTEGER, acked_at INTEGER, PRIMARY KEY(message_id, principal_id));
CREATE TABLE cursors(principal_id TEXT, board_id TEXT, last_seq INTEGER NOT NULL, PRIMARY KEY(principal_id, board_id));
CREATE TABLE subscriptions(principal_id TEXT, board_id TEXT, level TEXT CHECK(level IN('all','addressed','digest')), PRIMARY KEY(principal_id, board_id));
CREATE TABLE claims(id TEXT PRIMARY KEY, target_kind TEXT CHECK(target_kind IN('ticket','task','path_glob','resource')), target TEXT NOT NULL,
  holder_id TEXT NOT NULL, exclusive INTEGER NOT NULL DEFAULT 1, reason TEXT,
  acquired_at INTEGER, heartbeat_at INTEGER, expires_at INTEGER NOT NULL, released_at INTEGER);
CREATE UNIQUE INDEX one_active_exclusive ON claims(target_kind, target) WHERE released_at IS NULL AND exclusive = 1;
CREATE TABLE handoffs(id TEXT PRIMARY KEY, ticket_id TEXT NOT NULL, version INTEGER NOT NULL, author_id TEXT, to_spec TEXT,
  doc_md TEXT NOT NULL, criteria_json TEXT NOT NULL, git_ref TEXT, session_ref TEXT, created_at INTEGER, acked_by TEXT, acked_at INTEGER,
  UNIQUE(ticket_id, version));
CREATE TABLE digests(id TEXT PRIMARY KEY, board_id TEXT, through_seq INTEGER, model TEXT, body_md TEXT, cited_ids_json TEXT,
  validated INTEGER, created_at INTEGER);
CREATE TABLE approvals(id TEXT PRIMARY KEY, request_msg_id TEXT NOT NULL, action TEXT NOT NULL, params_hash TEXT NOT NULL,
  risk TEXT CHECK(risk IN('low','medium','high')), state TEXT CHECK(state IN('pending','approved','rejected','expired','consumed')),
  decided_by TEXT, decided_at INTEGER, expires_at INTEGER, hmac TEXT);
CREATE VIRTUAL TABLE messages_fts USING fts5(subject, body_md, content='messages', content_rowid='seq');
```
Expired exclusive claims must be closed (by setting `released_at`) before a new claim can be inserted. The daemon does this in the same transaction as the acquire, which makes the acquire atomic, like Beads `--claim` and Claude Code's file lock.

#### 7.3 Message kinds
Kinds merge Linear activities, A2A states, Agent Teams protocol messages and Gas Town escalation.

| kind | Purpose | Created by | `requires` | Payload highlights |
|---|---|---|---|---|
| `status` | milestone progress; `ephemeral=1` supersedes author's prior ephemeral status in thread | agents | none | `{phase, pct?, next}` |
| `claim` / `release` | mirror of claims table | daemon | none | `{claim_id, target, ttl}` |
| `question` | ask a principal/role; optional options (Linear `select`) | agents, human | answer | `{blocking:bool, options:[{label,value}], deadline?}` |
| `answer` | reply to question | addressed principal | none | `{question_id}` |
| `decision` | record a decision with rationale | agent proposes; owner/human accepts | ack | `{status:proposed|accepted|superseded, rationale, alternatives[], evidence_refs[]}` |
| `approval_request` | gated action (A2A auth-required; Linear `auth`) | agents | approval | `{action, params, params_hash, risk, preview, expires_at}` |
| `approval` | grant/deny | **human via TUI only** (daemon writes) | none | `{approval_id, verdict, conditions?}` |
| `handoff` | pointer to handoff version | agents/human | ack | `{handoff_id, version, to}` |
| `blocker` | blocked on X; severity | agents, watchdog | answer | `{severity:P0|P1|P2, waiting_on}` |
| `result` | task done claim (Linear `response`) | agents | ack | `{criteria_passed[], evidence_refs[]}` (must be non-empty) |
| `error` | failure report | agents, system | none | `{error_class, retryable}` |
| `artifact` | PR/commit/test run/file link | agents | none | `{refs[]}` |
| `digest` | derived summary | digest agent | none | `{through_seq, cited_ids[]}` |
| `external` | bridged inbound item (quarantined) | bridge | none | `{source, extracted_fields, raw_ref}` |
| `signal` | `stop`/`pause`/`resume`/`reprioritize` (Linear `stop`) | human (watchdog may `pause`) | ack | `{signal, reason}` |
| `note` | FYI | any | none | — |
| `system` | session start/end, idle, lease expiry, hook events | daemon | none | — |

**Validation rules:**
- `result` without evidence is rejected, and the agent is asked for `status` instead.
- Only T0 may create `approval` or `signal:stop`.
- Bodies over 4,000 characters are rejected with "attach as artifact".
- A `decision` that changes acceptance criteria requires human acceptance.

#### 7.4 Threading and addressing
- **Boards:** One per ticket, one per project, one per mode (Work/Life inbox), and a derived global "@human needs-you" view.
- **Threads:** The main thread id is the ticket key (e.g., `NW-142`), with subject prefix `[NW-142]` (Agent Mail convention). `question` and `decision` messages can root sub-threads.
- **Cross-posting:** Project boards automatically receive `decision`, `handoff`, `blocker` and `result` from their tickets as `refs`, not copies.
- **Addresses:**
  - `@human`
  - `@<principal-name>` (e.g., `@claude-nw142-impl`, matching the Claude Code `--name` for doorbell routing)
  - `@role:<role>` (coder | reviewer | triage | digest | watchdog), resolved at send time
  - `@owner` (current ticket claim holder)
  - `@board` (subscribers; agents may use it only on their own ticket board, to limit chatter)
- **Linear aggregation:** A ticket board lists linked Linear issues in `refs`. Inbound Linear prompts and comments become `external`, or T0-remote if authored by the user.

#### 7.5 Read/write protocol
**MCP server `narwarp-board`:** stdio, with the session token passed in env so the daemon can bind identity. Tools:
- `nw_start_session(ticket, role)` returns `{principal, claim, handoff_latest, digest, open_items_for_me, unread_counts}`. This one-call bootstrap is modelled on Agent Mail's `macro_start_session` and is especially good for Ollama agents.
- `nw_inbox(scope="mine"|board_id, since_seq?, kinds?, limit=20, bodies=false)` returns headers.
- `nw_read(ids[])` returns bodies wrapped with provenance tags. Reading marks `read_at`.
- `nw_post(board, kind, subject, body?, to?, refs?, payload?)`, `nw_reply(id, …)`, `nw_ack(ids[])`.
- `nw_ask(board, to, question, blocking, options?)` and `nw_answer(question_id, body)`.
- `nw_request_approval(action, params, risk, preview)` returns `approval_id`. The agent should continue other work or post `status` while waiting.
- `nw_claim(target_kind, target, ttl_s=1800, exclusive=true, reason)`, `nw_renew(claim_id)`, `nw_release(claim_id)`.
- `nw_handoff_write(ticket, doc_md, criteria_json)` and `nw_handoff_read(ticket, version?)`.
- `nw_search(query, board?)` and `nw_digest(board)`.

**CLI `nw board …`:** Same operations, plus `nw prime --hook` and `nw board check --inject [--only addressed,urgent] [--max-chars 1500]`. This serves hooks, scripts, Ollama tools and the human. Output can be JSON or plain text.

**Claude Code hook set:** Installed by Narwarp when it launches a session, with env `NARWARP_PRINCIPAL`, `NARWARP_TICKET` and `NARWARP_TOKEN`.

| Event | Command | Injects | Notes |
|---|---|---|---|
| `SessionStart` (startup/resume/after compaction) | `nw prime --hook` | ≤8,000 chars: identity/role; ticket goal + criteria status; latest HANDOFF (summary + path); board digest; open items for me; my claims; board etiquette as facts | Gas Town/Beads `prime` pattern |
| `UserPromptSubmit` | `nw board check --inject --max-chars 1500` | unread *headers* addressed to me / urgent | silent if none |
| `PostToolUse` | `nw board check --inject --only addressed,urgent --rate 300s` + claim heartbeat | headers only | daemon enforces rate limit except P0 |
| `PreCompact` | `nw handoff checkpoint` | — | snapshot git state + criteria into a draft handoff; re-prime follows |
| `Stop` | `nw board stop-gate` | exit 2 with reason once, if: claim held and no `status`/`handoff`/`result` since last check, or a blocking question to me is unanswered | prevents silent abandonment; one-shot to avoid loops |
| `TaskCompleted` (if using Agent Teams) | `nw criteria verify` | exit 2 if criteria evidence missing | mirrors MAST verification fix |

**Doorbell:** For P0 items to an idle or busy Claude session, the daemon writes a one-liner such as "[Narwarp] urgent board item 01JA… from @human on NW-142" to that session's `CLAUDE_CODE_MESSAGING_SOCKET`. Narwarp-managed sessions start with `crossSessionInbound: accept` in their `--settings`, because daemon posts are not "own-child" messages and could otherwise be held.

**Ollama agents:** These run in Narwarp's own Rust loop and call the same API in-process. The inbox is checked at each step boundary. They use `nw_start_session` and other macro-style tools, with structured JSON outputs.

#### 7.6 Claiming and locking
- **Claim targets:**
  - ticket: exactly one owner, who writes code and state
  - task: subtask within a ticket
  - path_glob: advisory, conflicts reported (Agent Mail semantics), with an optional git pre-commit guard when sessions share a checkout. Worktrees are covered by another researcher.
  - resource: e.g., `calendar:work:write`, `linear:ENG-881:write`, `discord:#backend:post`, to serialize side effects.
- **Acquire:** A daemon transaction closes expired rows and then inserts. The unique partial index guarantees one active exclusive claim per target.
- **TTL and heartbeat:** Default TTL is 30 min, with heartbeat via the `PostToolUse` hook.
- **Expiry:** On expiry, the daemon posts a `system` message and the watchdog checks for recent activity. Reassignment requires expiry plus a grace period, or a human action, and auto-requests a handoff from the old holder's last state. "Stale lock cleanup is expected."
- **Non-owners:** Agents without a claim may post `note`, `question` or `artifact` but not `result`.

#### 7.7 Handoff documents
- **Template:** As in Q3. Versioned in `handoffs`, mirrored to `HANDOFF.md` and `handoff.criteria.json`, announced with a `handoff` message, and acked by the receiver.
- **Required when:** The claim holder stops, context is near its limit, the ticket is reassigned, the holder is blocked for more than T, or the human requests one.
- **Seance pointer:** Keep `session_ref` (the predecessor's Claude session id) so a successor or human can resume it and question it.
- **Linting:** Narwarp lints handoffs. All 11 sections must be present, the next step must be non-empty, evidence is required for each `passes: true`, and user-constraint quotes must be preserved from the previous version.

#### 7.8 Digests and context budgets
- **Ticket digest:** ≤400 tokens, with the sections State, Decisions (ids), Open questions (owner, blocking), Blockers, Latest artifacts, and "Since your last read".
- **Project digest:** ≤800 tokens, built from ticket digests plus project decisions.
- **Daily human digest:** Lists "needs you" items first.
- **Generation:** A local model regenerates a digest when there are 10 or more new messages, at idle, or on demand.
- **Validation:** Every bullet must cite at least one id that exists on that board. Decision and approval text is copied from payloads, not paraphrased. If validation fails, fall back to an extractive header list. Digests are marked `derived` and inherit the lowest trust of their sources.
- **Budgets:** `SessionStart` ≤8,000 chars, per-turn check ≤1,500 chars (under the 10,000-char hook cap). Bodies are fetched just in time.
- **Retention:** Superseded ephemeral statuses are hidden from digests. Threads closed for more than 14 days collapse to their digest ("memory decay"). Nothing is deleted from SQLite.

#### 7.9 Human in the loop
- **Mentions:** `@human` mentions and `requires=answer|approval` items appear in a TUI "Needs you" queue. It is sorted by blocking first, then urgency, then age, with badges per board and an optional macOS notification. Outbound Discord DMs are optional and approval-free only for notifications to the user themself.
- **Approval card:** Shows the action, params diff or preview, risk, requester, provenance chain (e.g., "triggered by external email from X"), and expiry. The human can approve, deny or edit-and-approve. An approval is single-use and bound to `params_hash`.
- **Signals:**
  - `stop`: the agent halts and posts a final `result` or `error` with its state (Linear semantics).
  - `pause`/`resume`.
  - `reprioritize`: carries the new priority and reason, and can be emitted by the calendar module.
- **Overseer semantics:** Human messages preempt per Q5, with an option to queue.

#### 7.10 Security model
This section summarizes Q6:
- T0–T3 tiers stamped and signed by the daemon
- wrappers in every rendering
- Dual-LLM quarantine for T3
- Rule-of-Two session profiles
- human-only signed approvals
- Work/Life access-control separation
- secret scrubbing
- ANSI/control-character stripping in the TUI
- rate limits, dupe suppression and loop breakers
- per-session tokens
- AGENTS.md text stating that board content is information, not authorization

#### 7.11 External mirroring (outbound is gated; inbound becomes `external`)
- **Linear:** Optional for work tickets. Represent Narwarp agents as a Linear agent: `status` maps to ephemeral `thought`/`action`, `question` to `elicitation` (+`select`), `approval_request` (OAuth) to `elicitation`+`auth`, `result` to `response`, and `error` to `error`. Plan steps come from criteria. Meet the 10-second first-activity rule, rate-limit, and only for linked issues. API mechanics belong to the integrations researcher.
- **GitHub:** PR and commit links become `artifact`. PR comments by others become `external`.
- **Discord:** Inbound becomes `external` via the quarantine reader. Outbound posts require approval or a pre-approved template.

#### 7.12 Watchdog rules mapped to MAST failure modes
Deterministic rules run first, and a local model only classifies or words the nudge.
- **Step repetition (15.7%):** 3 or more near-duplicate statuses, or an identical tool-error streak. Action: nudge, then escalate.
- **Unaware of termination (12.4%):** All criteria pass but no `result`, or a claim held more than T with no progress (GUPP-style). Action: nudge.
- **Reasoning-action mismatch (13.2%) and unverified completion (8.2% / 9.1%):** A `result` or "tests pass" claim without a test-run ref. Action: reject, and request evidence.
- **Ignored input / withholding (1.9% / 0.85%):** An addressed question unanswered for more than T, or an unacked handoff. Action: re-notify, then human.
- **Loops:** A↔B more than K messages within W minutes. Action: circuit-break and escalate to the human.
- **Derailment (7.4%):** Digest drift from ticket goal, as an LLM similarity check. Action: flag to human only.

#### 7.13 Phasing
- **MVP:** Tables, daemon API, `nw` CLI, MCP tools (start_session, inbox/read/post/ack, ask/answer, claim/renew/release, handoff read/write), `SessionStart` and `UserPromptSubmit` hooks, TUI board pane, "Needs you" queue, and approvals for push, merge and outbound messages.
- **V2:** Local-model digests and watchdog, `PostToolUse`/`Stop` hooks, Linear agent mirroring, Discord bridge with quarantine.
- **V3:** Socket doorbell, criteria verification on `TaskCompleted`, optional A2A façade (only if remote agents are ever needed), and optional Agent Mail interop (tool-name aliases) if users want the agent ergonomics of an existing tool.

### Gaps
- **Thresholds are unvalidated:** These come from design inference, not measured studies on this kind of harness: 30-minute TTL, 300-second injection rate, ≤400/800-token digests, 4,000-character bodies, and 10-message digest triggers. They should be tuned with Narwarp's own telemetry.
- **Doorbell behaviour:** Not tested here. This includes whether `crossSessionInbound: accept` set via `--settings` behaves as documented for Narwarp-spawned sessions, and how the doorbell interacts with permission modes. Same for re-priming after compaction, where the `SessionStart` source value for post-compaction was not confirmed in the excerpt I read.
- **Reuse vs build:** Whether to reuse `mcp_agent_mail_rust` as the backend could not be evaluated. Its license and stability were unverified, and its project_key-is-a-repo-path model doesn't fit Life-mode tickets. Ticket data models belong to another researcher.
