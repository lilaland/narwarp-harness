# Kanban- and ticket-centric harnesses for AI coding agents (2025 – Oct 2026)

*Method note: these notes reflect the state on 2026-10-13. Star counts, created/pushed dates and licenses come from the GitHub search API, read on 2026-10-13. Last-commit and tag dates come from shallow `git clone`s of each repo made the same day. Schema quotes come from the files in those clones; each one cites the file's GitHub URL. Anything older than about 2026-04-13 is marked "possibly stale". Opinions from practitioners are labelled **[practitioner]**. Vendor-written or SEO-style sources are labelled **[vendor/secondary]**.*

---

## 1. Which tools matter, what state are they in, and what are their architecture, license, traction and target user?

### Takeaway
The field moved a lot in 2026. The most-starred ticket-centric tools are now new tools that present themselves as an "issue tracker or org for agents": Paperclip (~101k★, March 2026), Multica (~53k★), Beads (~28k★, now Dolt-backed under `gastownhall`) and OpenAI's Symphony (~28k★, which uses Linear as its board). The first wave of "kanban for Claude Code" tools has mostly died or stalled. Vibe Kanban announced its sunset on 2026-04-10. Automaker says it is unmaintained. Task Master, CCPM, Kanbots and ticket have had no commits for 4–7 months. Aperant (formerly Auto-Claude) has 2.x in maintenance while a rewrite happens elsewhere. Linear has become a complete agent platform in its own right, with its own coding agent, triage AI and recurring "Loops".

### Cited Findings

**Tier A: active, and the ticket or board is the core UX**
- **Paperclip** (`paperclipai/paperclip`). Created 2026-03-02. 101,401★, 17,025 forks, 6,061 open issues. MIT, TypeScript. Last push 2026-10-13. Tagline: "The open-source app everyone uses to manage agents at work" — [GitHub](https://github.com/paperclipai/paperclip). It uses calendar-style versions (stable `v2026.1001.0`, `v2026.1005.0`, `v2026.1012.0`) plus daily canary tags; 1,087 tags were created in the last 90 days — [tags](https://github.com/paperclipai/paperclip/tags). Architecture: "a Node.js server and React UI that orchestrates a team of AI agents to run a business… It looks like a task manager. Under the hood: org charts, budgets, governance, goal alignment". The pitch is "If OpenClaw is an employee, Paperclip is the company" — [README](https://github.com/paperclipai/paperclip/blob/master/README.md). Storage is Postgres through Drizzle, with an embedded-Postgres option (`packages/db/src/embedded-postgres-*.ts`) — [schema dir](https://github.com/paperclipai/paperclip/tree/master/packages/db/src/schema). Target user: anyone running many agents. The README says it is for you if "You have 20 simultaneous Claude Code terminals open and lose track of what everyone is doing" — [README](https://github.com/paperclipai/paperclip/blob/master/README.md).
- **Multica** (`multica-ai/multica`). Created 2026-01-13. 52,686★. Go server. Last commit 2026-10-13, tag v0.6.2 — [GitHub](https://github.com/multica-ai/multica). It calls itself "source-available": "Agents that show up on the board… assign work to AI coding agents the way you'd assign it to a teammate — they pick up the issue, report progress, raise blockers, and hand it back for review" — [README](https://github.com/multica-ai/multica/blob/main/README.md). The license is Apache-2.0 plus "additional conditions" from Index Labs (Hong Kong) Ltd that restrict hosted/embedded commercial use without a commercial license — [LICENSE](https://github.com/multica-ai/multica/blob/main/LICENSE). Runtime: "a daemon on your laptop or cloud box. Code never leaves it". Web, desktop and mobile clients — [README](https://github.com/multica-ai/multica/blob/main/README.md). The schema has 598 `up` migrations — [migrations](https://github.com/multica-ai/multica/tree/main/server/migrations).
- **Beads / `bd`** (`gastownhall/beads`, moved from `steveyegge/beads`; the old git URL still resolves). Created 2025-10-12. 27,840★, 1,448 open issues. Go. MIT — [GitHub](https://github.com/gastownhall/beads), [LICENSE](https://github.com/gastownhall/beads/blob/main/LICENSE). Very active: 1,595 commits in 90 days and 455 in 30 days. Releases: v1.3.0 (2026-09-15), v1.3.1 (2026-09-30), v1.3.2-rc.2 (2026-10-10) — [CHANGELOG](https://github.com/gastownhall/beads/blob/main/CHANGELOG.md), [tags](https://github.com/gastownhall/beads/tags). It is now a "Distributed graph issue tracker for AI agents, powered by Dolt". Docs moved to beads.gascity.com — [README](https://github.com/gastownhall/beads/blob/main/README.md). Sibling project **Gas Town** (`gastownhall/gastown`, "multi-agent workspace manager") has 18,360★ and was created 2025-12-16 — [GitHub](https://github.com/gastownhall/gastown). Beads' type system includes Gas Town-style swarm constructs (`molecule`, convoy `tracks` edges, `wisps`) — [types.go](https://github.com/gastownhall/beads/blob/main/internal/types/types.go).
- **Backlog.md** (`MrLesk/Backlog.md`). Created 2025-06-04. 7,018★. MIT, TypeScript (Bun). Last commit 2026-10-12. Releases v1.50.1 (08-10), v1.51.0 (09-02), v1.52.0 (09-12), v1.53.0 (09-24); 9 tags in 90 days — [GitHub](https://github.com/MrLesk/Backlog.md), [tags](https://github.com/MrLesk/Backlog.md/tags). It "turns any folder into a self-contained project board powered by plain Markdown files and a zero-config CLI". Local-first: "no server, no account, no telemetry". It has a terminal Kanban (`backlog board`), a web UI (`backlog browser`) and MCP — [README](https://github.com/MrLesk/Backlog.md/blob/main/README.md).
- **OpenAI Symphony** (`openai/symphony`). Created 2026-02-26. 27,662★. Elixir reference implementation. Apache-2.0. v0.0.3, last commit 2026-09-15 — [GitHub](https://github.com/openai/symphony). "Symphony turns project work into isolated, autonomous implementation runs… monitors a Linear board for work and spawns agents". It is labelled "a low-key engineering preview for testing in trusted environments". It is spec-first: the README suggests "Implement Symphony according to the following spec" — [README](https://github.com/openai/symphony/blob/main/README.md), [SPEC.md](https://github.com/openai/symphony/blob/main/SPEC.md).
- **Agor** (`preset-io/agor`, from Preset). Created 2025-10-04. 1,439★. TypeScript. **Business Source License 1.1**. v0.26.10 (2026-10-06); 15 tags in 90 days — [GitHub](https://github.com/preset-io/agor), [LICENSE](https://github.com/preset-io/agor/blob/main/LICENSE). It is a "live spatial board" (2D canvas). "Branches as cards, zones as regions". Zones "can fire a templated prompt when a branch is dropped in". Agor Cloud is in open beta — [README](https://github.com/preset-io/agor/blob/main/README.md).
- **Cyrus** (`ceedaragents/cyrus`). Apache-2.0. v0.2.73 (2026-09-30); 7 tags in 90 days; last commit 2026-10-08 — [GitHub](https://github.com/ceedaragents/cyrus). It is a Linear/GitHub/GitLab/Slack agent: it "monitors… issues assigned to it, creates isolated Git worktrees for each issue, runs (Claude Code|Codex|Cursor|Gemini|Opencode)". There is a free self-hosted community tier and paid Pro/Team plans at app.atcyrus.com — [README](https://github.com/ceedaragents/cyrus).
- **Cline Kanban** (`cline/kanban`). Created 2026-03-09. 1,356★. Apache-2.0. v0.1.70, last commit 2026-09-04. A "Research Preview" started with `npx kanban`: "Each task card gets its own terminal and worktree" — [README](https://github.com/cline/kanban/blob/main/README.md).

**Tier B: dead, sunsetting or stalled**
- **Vibe Kanban** (`BloopAI/vibe-kanban`). Created 2025-06-14. 28,344★, 543 open issues. Rust plus a TS frontend. Apache-2.0 — [GitHub](https://github.com/BloopAI/vibe-kanban). On **2026-04-10** Louis Knight-Webb announced that bloop is closing because the team "couldn't find a business model". The project continues as "open source and community maintained". Remote services (kanban issues, comments, projects, organisations) stayed up for 30 days before the move to "a fully local architecture"; local workspaces keep working — [shutdown post](https://www.vibekanban.com/blog/shutdown). The README header now reads "Vibe Kanban is sunsetting". A self-hosting guide remains for the cloud server — [README](https://github.com/BloopAI/vibe-kanban/blob/main/README.md). Release cadence collapsed: v0.1.44 (2026-04-24), then v0.1.45 (2026-09-19) — [tags](https://github.com/BloopAI/vibe-kanban/tags).
- **Claude Task Master** (`eyaltoledano/claude-task-master`). Created 2025-03-04. 28,215★. JS/TS. **MIT + "Commons Clause"** (selling is not allowed) — [LICENSE](https://github.com/eyaltoledano/claude-task-master/blob/main/LICENSE). Last commit 2026-04-23, so **possibly stale**. Docs and branding have moved to Hamster (tryhamster.com), a commercial product — [README](https://github.com/eyaltoledano/claude-task-master/blob/main/README.md).
- **CCPM** (`automazeio/ccpm`). Created 2025-08-18. 8,409★. MIT, Shell. Last commit 2026-03-18, so **stale**. It has been repackaged: "CCPM is now an AGENT SKILL!" (Claude Code, Codex, OpenCode, Factory, Amp, Cursor) — [GitHub](https://github.com/automazeio/ccpm), [README](https://github.com/automazeio/ccpm/blob/main/README.md).
- **Automaker** (`AutoMaker-Org/automaker`). Created 2025-12-07. 3,227★. TypeScript. MIT. Last commit 2026-05-22. Its LICENSE now states: "**This project is no longer actively maintained.**" — [LICENSE](https://github.com/AutoMaker-Org/automaker/blob/main/LICENSE), [GitHub](https://github.com/AutoMaker-Org/automaker). **Treat as dead.**
- **Aperant (formerly Auto-Claude)** (`AndyMik90/Aperant`; the `Auto-Claude` URL still resolves). Created 2025-12-04. 14,577★. TypeScript/Electron. AGPL-3.0 — [GitHub](https://github.com/AndyMik90/Aperant), [LICENSE](https://github.com/AndyMik90/Aperant/blob/develop/LICENSE). The README says "the current 2.x desktop app is in maintenance mode… Pull requests are paused… rebuilding the entire app from the ground up for Aperant 3.0" (adding cloud features, staying AGPL). "No public release date yet". Last commit on the default branch was 2026-06-14 — [README](https://github.com/AndyMik90/Aperant/blob/develop/README.md). **Flag: in limbo.**
- **Kanbots** (`leodavinci1/kanbots`). Created 2026-04-25. 613★. MIT. v1.2.0. Last commit 2026-05-25, so stalled for about 4.5 months — [GitHub](https://github.com/leodavinci1/kanbots). Its Show HN drew 263 points and 166 comments (about June 2026) — [HN](https://news.ycombinator.com/item?id=48239413).
- **ticket / `tk`** (`wedow/ticket`). Created 2026-01-02. 909★. MIT. A single bash script. Last commit 2026-03-15, so **stale** — [GitHub](https://github.com/wedow/ticket).
- **dex** (`dcramer/dex`, by Sentry co-founder David Cramer). Created 2026-01-22. 385★. "Task tracking for AI agents". JSONL storage. Last commit 2026-03-02, so **stale** — [GitHub](https://github.com/dcramer/dex).

**Vendor platforms**
- **Linear.** Linear Agent entered public beta on **2026-03-24** — [changelog](https://linear.app/changelog/2026-03-24-introducing-linear-agent). Code Intelligence followed on 2026-05-14 — [changelog](https://linear.app/changelog/2026-05-14-code-intelligence) (summary from search; not fetched). **Coding sessions** launched 2026-06-11 ("Linear Agent can now write code using Claude Code and Codex"; sessions "run in the cloud") — [changelog](https://linear.app/changelog/2026-06-11-coding-sessions). **Loops**, recurring agent workflows, arrived 2026-07-20 — [changelog](https://linear.app/changelog/2026-07-20-introducing-loops). Later entries: GitHub Copilot for Linear (2026-07-30), "New controls for Linear coding agent" including open-weight models starting with GLM (2026-09-24), and live diffs while the agent works (2026-10-08) — [changelog](https://linear.app/changelog). Triage Intelligence covers routing and duplicates — [docs](https://linear.app/docs/triage-intelligence). The third-party agent API covers app actors, delegation and AgentSessions — [developers/agents](https://linear.app/developers/agents).
- **GitHub Agent HQ.** On **2026-02-04** GitHub said Claude and Codex were "now available in public preview on GitHub and VS Code" alongside Copilot. "Assign an issue to Copilot, Claude, Codex, or all three to compare results". It requires Copilot Pro+ or Enterprise, and "Each coding agent session consumes one premium request" — [GitHub blog](https://github.blog/news-insights/company-news/pick-your-agent-use-claude-and-codex-on-agent-hq/), [community discussion](https://github.com/orgs/community/discussions/186179).
- **Cursor.** Cursor 2.0 shipped on 2025-10-29 with an agent-centric view: up to 8 parallel agents in worktrees or remote sandboxes — [AlternativeTo](https://alternativeto.net/news/2025/10/cursor-2-0-introduces-composer-its-first-coding-model-and-parallel-agent-workflows), [Techzine](https://www.techzine.eu/news/devops/135916/cursor-2-0-introduces-parallel-agents-and-new-model/). Secondary sources say Cursor 3 (April 2026) added an "Agents Window" listing local, worktree, cloud and SSH agents (**unverified, secondary**). The Linear integration lets you delegate or @mention Cursor on Linear issues — [Cursor blog](https://cursor.com/blog/linear), [Linear changelog 2025-08-21](https://linear.app/changelog/2025-08-21-cursor-agent) (**possibly stale**).
- **OpenAI Codex app.** The macOS app launched in early Feb 2026 — [9to5Mac](https://9to5mac.com/2026/02/02/openai-launches-codex-app-for-macos-here-are-the-details). Threads are grouped by project and run in Local, Worktree or Cloud mode. Automations run in background worktrees and put results in a review queue — [Codex app docs](https://developers.openai.com/codex/app), [features](https://developers.openai.com/codex/app/features). I found no feature called a "task list". OpenAI's ticket-centric offering is Symphony (above).
- **Kiro.** Specs consist of `requirements.md` (EARS-style acceptance criteria), `design.md` and `tasks.md`. Task status is tracked as in-progress or completed — [Kiro docs](https://kiro.dev/docs/specs/index). IDE 0.12 added parallel waves built from a task dependency graph, claiming "up to 4x" faster runs — [changelog 0.12](https://kiro.dev/changelog/ide/0-12/). The public issue repo has 4,363★ — [GitHub](https://github.com/kirodotdev/Kiro).
- **Claude Code itself.** Todos became **Tasks** in v2.1.16 (about 2026-01-22). Tasks are stored in `~/.claude/tasks`, can have dependencies, and can be shared across sessions with `CLAUDE_CODE_TASK_LIST_ID` — [ClaudeLog](https://www.claudelog.com/faqs/what-are-tasks-in-claude-code/), [reposted announcement](https://youmind.com/landing/x-viral-articles/claude-code-tasks-update) **[secondary]**. One source claims Beads "inspired Claude Code Tasks" — [MorphLLM](https://www.morphllm.com/beads-agent-memory) **[secondary, unverified]**. Nimbalyst's CEO describes "Agent View" as "a tmux-based session manager built into Claude Code… in research preview", and notes it is not kanban-based — [Nimbalyst](https://nimbalyst.com/blog/claude-code-session-kanban-organize-ai-agents/) **[vendor/secondary; not verified against Anthropic docs]**.

**Other 2026 newcomers (brief)**
- These are agent dev environments built on worktrees rather than ticket systems: Superset (15,123★), emdash (5,972★, YC W26; its topics list `jira` and `linear`) and Nimbalyst, formerly Crystal (1,886★, MIT, "track tasks"; Crystal itself had 3,128★) — [Superset](https://github.com/superset-sh/superset), [emdash](https://github.com/generalaction/emdash), [Nimbalyst](https://github.com/nimbalyst/nimbalyst), [Crystal](https://github.com/stravu/crystal).
- Small kanban boards: Kanban Code (langwatch, Swift macOS app, 329★), Claw-Kanban (72★, "role-based auto-assignment"; columns Inbox/Planned/In Progress/Review-Test/Done/Stopped), DanWahlin/ai-agent-board (64★). A Beads ecosystem has also formed: beads_viewer TUI (1,713★, "PageRank, critical path, kanban") and beads_rust (1,126★, SQLite plus JSONL) — [kanban-code](https://github.com/langwatch/kanban-code), [Claw-Kanban](https://github.com/GreenSheep01201/Claw-Kanban), [ai-agent-board](https://github.com/DanWahlin/ai-agent-board), [beads_viewer](https://github.com/Dicklesworthstone/beads_viewer), [beads_rust](https://github.com/Dicklesworthstone/beads_rust).

### Inferences
- Projects in this space die fast. The brief named nine open-source projects:
  - two are dead or sunsetting (Vibe Kanban, Automaker)
  - one is in rewrite limbo (Aperant)
  - two have had no commits for about 6–7 months (Task Master, CCPM)
  - only four are clearly shipping (Paperclip, Beads, Backlog.md, Agor)

  Among the 2026 newcomers, Kanbots, ticket and dex have also stalled. Narwarp should borrow patterns and schemas from these tools, not depend on them.
- Interest has shifted from "a board in front of N parallel agents" (Vibe Kanban, Kanbots, Cline Kanban) to "an issue tracker or org in which agents are first-class actors" (Paperclip, Multica, Linear Agent) and "an issue tracker *as* agent memory" (Beads, Claude Code Tasks).
- For a single user who already uses Linear at work, the strongest direct competitor for work tickets is Linear itself: it has agent sessions, coding sessions, triage AI and Loops. Narwarp's differentiation would be the local, cross-tool, work-plus-life aggregation layer that sits *above* Linear.

### Gaps
- I could not get Cyrus' star count. The GitHub search API reports that the `ceedaragents` owner "cannot be searched", which suggests a rename or transfer; the git URL still resolves.
- I could not confirm Claude Code "Agent View" or Cursor 3's "Agents Window" from primary vendor docs.
- Commit counts for repos fetched with `--shallow-since` came back unreliable for some repos (shown as 1). Tag dates are reliable and are what is reported above.

---

## 2. Task data model and storage: fields, statuses, hierarchy, dependencies, attempts, and why each storage choice was made

### Takeaway
The data models are converging on one shape. A ticket has project, parent, status (about 6–7 states including `in_review`, `blocked` and `cancelled`), Linear-style priority, structured acceptance criteria, typed dependencies, labels and a JSON `metadata` escape hatch. Runs or attempts are a **separate table**, linked to the ticket and carrying the agent session ID, branch/worktree, process status and cost. Storage splits three ways. Repo-local markdown (Backlog.md, ticket, CCPM) is chosen for human and agent readability in git. A local database (Vibe Kanban SQLite, Kanbots SQLite, Beads, which moved to **Dolt**) is chosen for concurrency and queries. An **external tracker as the only source of truth** (Symphony, Cyrus, CCPM's GitHub sync) avoids a database entirely.

### Cited Findings

**Vibe Kanban (SQLite via sqlx; Postgres plus ElectricSQL for the cloud)**
- Local `Task` struct: `id, project_id, title, description, status, parent_workspace_id, created_at, updated_at`, with `TaskStatus { Todo, InProgress, InReview, Done, Cancelled }` — [task.rs](https://github.com/BloopAI/vibe-kanban/blob/main/crates/db/src/models/task.rs).
- The 2025-06 initial schema had `tasks` → `task_attempts(worktree_path, merge_commit, executor, …)` → `task_attempt_activities(status IN ('init','setuprunning','setupcomplete','setupfailed','executorrunning','executorcomplete','executorfailed','paused'))` — [init.sql](https://github.com/BloopAI/vibe-kanban/blob/main/crates/db/migrations/20250617183714_init.sql).
- On 2025-12-16, "Refactor task_attempts into workspaces and sessions": `task_attempts`→`workspaces`; new `sessions(workspace_id, executor)`; `execution_processes(session_id, run_reason IN ('setupscript','codingagent','devserver','cleanupscript'), status IN ('running','completed','failed','killed'), exit_code)`; `executor_sessions`→`coding_agent_turns(agent_session_id, prompt, summary)` — [migration](https://github.com/BloopAI/vibe-kanban/blob/main/crates/db/migrations/20251216142123_refactor_task_attempts_to_workspaces_sessions.sql).
- On 2026-02-17 they removed the FK from `workspaces.task_id`, "so deleting a task no longer deletes workspaces". Workspaces gained `archived`, `pinned` and `name` — [migration](https://github.com/BloopAI/vibe-kanban/blob/main/crates/db/migrations/20260217120312_remove_task_fk_from_workspaces.sql). Shared tasks were dropped on 2026-01-13 — [migration](https://github.com/BloopAI/vibe-kanban/blob/main/crates/db/migrations/20260113144821_remove_shared_tasks.sql). `pull_requests(workspace_id, pr_url, pr_number, pr_status, target_branch_name, merged_at, merge_commit_sha, synced_at)` was added on 2026-03-11 — [migration](https://github.com/BloopAI/vibe-kanban/blob/main/crates/db/migrations/20260311000000_add_tracked_prs.sql).
- The richer cloud issue model, Postgres from 2026-01-12, includes:
  - `project_statuses(name, color, sort_order, hidden)`, i.e. custom columns
  - `issues(issue_number, simple_id e.g. 'ISS-12', status_id, priority ENUM('urgent','high','medium','low'), start_date, target_date, completed_at, sort_order DOUBLE, parent_issue_id, extension_metadata JSONB)`
  - `issue_assignees`, `issue_followers`, `issue_relationships(type ENUM('blocking','related','has_duplicate'))`, `tags`, threaded `issue_comments(parent_id)` with reactions, and `notifications`

  — [remote-projects.sql](https://github.com/BloopAI/vibe-kanban/blob/main/crates/remote/migrations/20260112000000_remote-projects.sql). On 2026-03-16 a many-to-many `pull_request_issues` join table was added — [migration](https://github.com/BloopAI/vibe-kanban/blob/main/crates/remote/migrations/20260316000000_pull_request_issues.sql).

**Backlog.md (one markdown file per task, YAML frontmatter, in `backlog/`)**
- `Task` interface fields:
  - `id, title, status (string, configurable), assignee[], reporter, createdDate, updatedDate, dueDate, labels[], milestone, dependencies[], references[], documentation[], modifiedFiles[]`
  - `description, implementationPlan, implementationNotes, comments[], finalSummary`
  - `acceptanceCriteriaItems[{index,text,checked}], definitionOfDoneItems[]`
  - `parentTaskId, subtasks[], priority, type, project ("Monorepo project/component"), branch, ordinal`
  - `source: "local"|"remote"|"completed"|"local-branch"` and `onStatusChange` ("per-task callback command to run on status change")

  — [src/types/index.ts](https://github.com/MrLesk/Backlog.md/blob/main/src/types/index.ts).
- On disk, frontmatter holds `id, title, status: To Do, assignee, created_date, updated_date, labels, dependencies, priority`. The body has `## Description` and `## Acceptance Criteria` with `<!-- AC:BEGIN -->- [ ] #1 …` markers. Subtask IDs are dotted, e.g. `back-367.02` — [backlog/tasks](https://github.com/MrLesk/Backlog.md/tree/main/backlog/tasks).
- Why markdown: "every task is a plain .md file in your repo… completed tasks remain in Git as a permanent record of what was attempted and why". Read commands offer "stable, versioned JSON", plus `--watch` — [README](https://github.com/MrLesk/Backlog.md/blob/main/README.md).

**Beads (Dolt: version-controlled SQL; JSONL is export only)**
- `Issue` struct fields:
  - Content: `ID (hash, e.g. bd-a1b2), Title, Description, Design, AcceptanceCriteria, Notes, SpecID`
  - Workflow: `Status, Priority (int, 0=P0), IssueType, IsBlocked`, `Assignee, Owner, EstimatedMinutes`
  - Lifecycle: `CreatedBy, StartedAt, ClosedAt, CloseReason, ClosedBySession ("Claude Code session that closed this issue")`
  - Leasing: `LeaseExpiresAt, HeartbeatAt, LeaseGrantedNode`, plus `RowVersion` for optimistic concurrency
  - Scheduling: `DueAt, DeferUntil ("Hide from bd ready until this time")`
  - External and extension: `ExternalRef ("gh-9", "jira-ABC"), SourceSystem, Metadata (JSON)`, compaction fields, `Labels, Dependencies, Comments`
  - Messaging: `Sender, Ephemeral, WispType`

  — [types.go](https://github.com/gastownhall/beads/blob/main/internal/types/types.go).
- Statuses are `open, in_progress, blocked, deferred, closed, pinned, hooked`. Types are `bug, feature, task, epic, chore, decision, message, molecule, gate, spike, story, milestone`. Dependency types include `blocks, parent-child, conditional-blocks ("B runs only if A fails"), waits-for, related, discovered-from, replies-to, relates-to, duplicates, supersedes, … tracks, until, caused-by, validates, delegated-from ("completion cascades up")` — [types.go](https://github.com/gastownhall/beads/blob/main/internal/types/types.go).
- Hierarchical IDs: `bd-a3f8` (epic) → `bd-a3f8.1` → `bd-a3f8.1.1`. The README says "Hash-based IDs… prevent merge collisions in multi-agent/multi-branch workflows". Storage is embedded Dolt by default (single writer) or a Dolt server for concurrent writers. "`.beads/issues.jsonl` is an export for viewers and interchange, not the source of truth" — [README](https://github.com/gastownhall/beads/blob/main/README.md). Sync uses `bd dolt push/pull` against `refs/dolt/data` on the git remote, and "JSONL import is upsert-only; it cannot infer that records absent from an export were deleted" — [sync-concepts.md](https://github.com/gastownhall/beads/blob/main/docs/core-concepts/sync-concepts.md).

**Paperclip (Postgres/Drizzle)**
- `issues` columns:
  - Scope: `companyId, projectId, projectWorkspaceId, goalId, parentId, visibility (open|private)`
  - Content and state: `title, titleNeedsGeneration, description, status (default 'backlog'), statusVersion, workMode, priority, reviewPolicy`
  - People: `assigneeAgentId, assigneeUserId, createdByAgentId/UserId, responsibleUserId`
  - Execution lock: `checkoutRunId, executionRunId, executionLockedAt`
  - Identity and origin: `issueNumber, identifier, originKind (default 'manual'), originId, originRunId, requestDepth, billingCode`
  - Execution and monitoring: `executionPolicy/State (jsonb), monitorNextCheckAt…, executionWorkspaceId, unblockDescriptor, blockedTransitionAt`
  - Timestamps: `startedAt, completedAt, cancelledAt`

  — [issues.ts](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/issues.ts).
- Enums:
  - `ISSUE_STATUSES = backlog, todo, in_progress, in_review, done, blocked, cancelled`; `ISSUE_PRIORITIES = critical, high, medium, low`
  - `ISSUE_REVIEW_POLICIES = anyone, not_creator, human_only`; `ISSUE_WORK_MODES = standard, ask, planning, skill_test`; `MAX_ISSUE_REQUEST_DEPTH = 1024`
  - `GOAL_LEVELS = company, team, agent, task`; `GOAL_STATUSES = planned, active, achieved, cancelled`; `PROJECT_STATUSES = backlog, planned, in_progress, completed, cancelled`

  — [constants.ts](https://github.com/paperclipai/paperclip/blob/master/packages/shared/src/constants.ts).
- `goals(level, status, parentId, ownerAgentId)` — [goals.ts](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/goals.ts). `projects(goalId, visibility, privacyOwnerUserId, personalOwnerUserId, status, leadAgentId, targetDate, color, icon, env, pauseReason, executionWorkspacePolicy, archivedAt)` — [projects.ts](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/projects.ts). `issue_relations(type: 'blocks')` — [issue_relations.ts](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/issue_relations.ts). `issue_work_products(type, provider, externalId, url, status, reviewState, isPrimary, healthStatus, summary, createdByRunId)` — [issue_work_products.ts](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/issue_work_products.ts). The schema directory holds 167 table modules, including `heartbeat_runs`, `approvals`, `budget_policies` and `routines` — [schema dir](https://github.com/paperclipai/paperclip/tree/master/packages/db/src/schema).

**Multica (Postgres)**
- The initial `issue` table:
  - `status IN ('backlog','todo','in_progress','in_review','done','blocked','cancelled')`
  - `priority IN ('urgent','high','medium','low','none')`
  - `assignee_type IN ('member','agent'), assignee_id, creator_type IN ('member','agent')`
  - `parent_issue_id, acceptance_criteria JSONB, context_refs JSONB, position FLOAT, due_date`

  Related tables: `issue_dependency(type IN ('blocks','blocked_by','related'))`; `comment(author_type member|agent, type IN ('comment','status_change','progress_update','system'))`; `inbox_item(severity IN ('action_required','attention','info'))`; `agent_task_queue(agent_id, issue_id, status IN ('queued','dispatched','running','completed','failed','cancelled'), priority, result JSONB, error)` — [001_init.up.sql](https://github.com/multica-ai/multica/blob/main/server/migrations/001_init.up.sql).
- Later migrations added `issue.project_id` (034), `origin_type` autopilot / quick-create / Lark chat / Slack chat (042, 060, 111, 131), `issue_pull_request` (079), squad assignees (084), size-limited `metadata JSONB` (105) and `stage` (123) — [migrations](https://github.com/multica-ai/multica/tree/main/server/migrations).

**Task Master (`tasks.json`)**
- `Task { id, title, description, status, priority, dependencies[], details, testStrategy, subtasks[], effort, actualEffort, tags, assignee, databaseId, complexity ('simple'|'moderate'|'complex'|'very-complex' | 1–10), recommendedSubtasks, expansionPrompt, complexityReasoning, metadata ("Use for external IDs… preserved through AI operations") }`. Statuses are `pending, in-progress, done, deferred, cancelled, blocked, review, completed`; priorities are `low…critical`. The types file also refers to "AI Metadata Types (from Supabase task metadata)", i.e. the Hamster cloud — [types/index.ts](https://github.com/eyaltoledano/claude-task-master/blob/main/packages/tm-core/src/common/types/index.ts).

**Others**
- **CCPM** task file frontmatter: `name, status: open, created, updated, github: (set on sync), depends_on: [], parallel: true, conflicts_with: []`. Files live in `.claude/epics/<name>/001.md…` — [structure.md](https://github.com/automazeio/ccpm/blob/main/skill/ccpm/references/structure.md).
- **ticket**: "markdown files with YAML frontmatter in `.tickets/`". Fields: `type (bug|feature|task|epic|chore), priority 0-4, assignee, external-ref, parent, tags, design, acceptance`; statuses `open|in_progress|closed`; `dep`, `link` and `ready`. The author's rationale: "without the need for keeping a SQLite file in sync or a rogue background daemon"; the files also let agents search without "dumping ten thousand character JSONL lines into their context" — [README](https://github.com/wedow/ticket/blob/master/README.md).
- **dex**: "JSONL format (one task per line)". Each task has Description (title), Context (body) and Result ("like a PR description") — [GitHub](https://github.com/dcramer/dex).
- **Kanbots**: tables `threads(repo_owner, repo_name, issue_number)`, `agent_runs(thread_id, worktree_path, branch_name, pid, status, token_usage_input/output, exit_reason)`, `messages(agent_run_id, promoted_github_comment_id)`, `cards(type, payload, status, resolved_value)` (decision prompts) and `agent_events(seq, type)` — [0001-initial.ts](https://github.com/leodavinci1/kanbots/blob/main/packages/local-store/src/migrations/0001-initial.ts). Local mode uses `local_issues(number, title, body, state, labels JSON, assignees JSON)` in `.kanbots/db.sqlite` — [0003-local-issues.ts](https://github.com/leodavinci1/kanbots/blob/main/packages/local-store/src/migrations/0003-local-issues.ts), [README](https://github.com/leodavinci1/kanbots/blob/main/README.md).
- **Automaker**: `Feature { id, title, titleGenerating, category, description, passes, priority, status, dependencies, spec, model, branchName, skipTests, planningMode, requirePlanApproval, planSpec, error, summary, descriptionHistory, … }` — [feature.ts](https://github.com/AutoMaker-Org/automaker/blob/main/libs/types/src/feature.ts).
- **Aperant**: task events are validated with `{type, taskId, specId, projectId, timestamp, eventId, sequence}`, an event-sourced task timeline — [task-event-schema.ts](https://github.com/AndyMik90/Aperant/blob/develop/apps/desktop/src/main/agent/task-event-schema.ts).
- **Linear agent objects**: `AgentSession` states are `pending, active, error, awaitingInput, complete, stale`. Activity types are `thought, elicitation, action{action,parameter,result}, response, error, prompt`. A session `plan` is an array of `{content, status: pending|inProgress|completed|canceled}`, and `externalUrls` is `[{label,url}]` — [agent-interaction](https://linear.app/developers/agent-interaction). "Assigning an issue to your app now sets it as the `delegate`, not the `assignee`. Humans keep ownership" — [developers/agents](https://linear.app/developers/agents).

### Inferences
- Paperclip and Multica, the two most-starred 2026 tools, independently arrived at the **same 7-status set** (`backlog, todo, in_progress, in_review, done, blocked, cancelled`). Multica's priority set also matches Linear's. That makes it a safe default whose statuses map almost one-to-one onto Linear state types.
- "Attempts" became first-class run records everywhere: VK workspaces and sessions, Kanbots `agent_runs`, Multica `agent_task_queue`, Paperclip `heartbeat_runs` plus `checkoutRunId`, Linear `AgentSession`. Vibe Kanban's history (task_attempts → workspaces + sessions → workspace no longer FK-bound to task) shows that runs need to outlive, and sometimes exist without, a ticket.
- Agents are first-class actors in the schema: `*_type member|agent` (Multica), separate agent/user FKs (Paperclip), Linear's delegate-vs-assignee. Narwarp needs this because tickets will be created and updated by Claude Code and Ollama agents as well as by the user.
- The storage choice tracks the deployment model. Markdown in the repo suits one repo and git-native review. A database suits cross-repo, multi-writer use, and work plus life tasks. Beads' move from JSONL/SQLite to Dolt and the explicit warning "JSONL… not the source of truth" are evidence that file-sync approaches drift under multiple writers.

### Gaps
- I did not inspect Paperclip's or Multica's full comment, approval and run-event schemas (this overlaps with the message-board researcher's scope).
- After the shutdown, it is unclear whether VK's local board uses the legacy local `tasks` table or a local port of the cloud `issues` model. The README still says "Use kanban issues to plan work… When you're ready to begin, create workspaces".

---

## 3. Prompt or PRD → tickets: how tools decompose free-form input, and whether any of them auto-assign tickets to projects

### Takeaway
Decomposition is almost always done **by the coding agent itself, following instructions or templates** that the tool injects: PRD → epic → tasks with acceptance criteria and dependency/parallelism metadata, followed by a human review checkpoint. **Automatic routing to a project or area is rare.** The only mature, documented implementation is **Linear Triage Intelligence**. It uses LLMs plus "historical trends" to suggest team, project, assignee and labels, and it lets teams show, hide or auto-apply suggestions per property. No open-source agent board I reviewed classifies tickets into projects with embeddings or LLMs and reports accuracy.

### Cited Findings
- **Backlog.md** has a three-checkpoint flow:
  1. "the agent decomposes your idea into tasks with descriptions, acceptance criteria, and milestones before implementation starts"
  2. "the agent… writes its implementation plan into the task. Approve it or steer"
  3. "one task = one context window = one PR"

  Recommended prompt: "Please decompose this into small Backlog.md tasks". For a bad result: "clear the plan/notes/final summary, refine the task… run the task again in a fresh session" — [README](https://github.com/MrLesk/Backlog.md/blob/main/README.md).
- **Task Master**: `parse-prd` turns a PRD into a structured `tasks.json` with tasks, dependencies, priorities and test strategies — [tutorial (mirror)](https://glama.ai/mcp/servers/@eyaltoledano/claude-task-master/blob/2d1211bfb32752b2b1130c9ce7f49b6217244f60/docs/tutorial.md) **[secondary mirror of the docs; possibly stale; summarized by search]**. A community guide reports that `parse-prd --append` adds tasks from a further PRD **[unverified]**. The task schema carries complexity analysis outputs (`complexity`, `recommendedSubtasks`, `expansionPrompt`, `complexityReasoning`) — [types](https://github.com/eyaltoledano/claude-task-master/blob/main/packages/tm-core/src/common/types/index.ts).
- **CCPM**: "Guided brainstorming + PRD creation" → "Parallelizable task files with dependencies" → "Epic issue + sub-issues + worktree" — [README](https://github.com/automazeio/ccpm/blob/main/README.md). Decomposition sorts work into task types (Setup/Data/API/UI/Tests/Docs). It sizes batches as "Small (<5 tasks): sequentially… Large (>10): analyze dependencies first, launch parallel agents (max 5 concurrent)" and spawns parallel subagents to write the task files — [structure.md](https://github.com/automazeio/ccpm/blob/main/skill/ccpm/references/structure.md).
- **Kiro**: the flow is requirements (EARS) → design → tasks. "Quick Plan" "generates requirements, design, and tasks in a single pass without approval gates" — [Kiro specs](https://kiro.dev/docs/specs/index), [changelog 0.12](https://kiro.dev/changelog/ide/0-12/).
- **Cline Kanban**: "open the sidebar chat and ask your agent to break work down into tasks for you. Kanban injects board-management instructions into that session so you can simply ask it to add tasks, link tasks" — [README](https://github.com/cline/kanban/blob/main/README.md).
- **Paperclip** persists decompositions as data: `issue_plan_decompositions(sourceIssueId, acceptedPlanRevisionId, status default 'in_flight', requestFingerprint, requestedChildCount, requestedChildren jsonb, childIssueIds jsonb, ownerAgentId, ownerRunId)` — [schema](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/issue_plan_decompositions.ts). Issues have `titleNeedsGeneration`, so a title can be generated later from a rough description — [issues.ts](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/issues.ts). 2026 reviews say "task decomposition [is] the strongest showing" — [eesel](https://www.eesel.ai/blog/paperclip-ai-review), [fast.io](https://fast.io/resources/paperclip-ai-review-2026/) **[secondary/SEO; summarized by search, not individually verified]**.
- **Multica**: "It starts as three rough sentences in an issue. It ends as a pull request." "Squads" let "the leader route the work" — [README](https://github.com/multica-ai/multica/blob/main/README.md).
- **Linear Agent** (2026-03-24): "Make issues based on the discussion here and assign them to me" (in Slack). A "Split into sub-issues" skill. It can extract common requirements from customer requests to "suggest a starting point for a spec" — [changelog](https://linear.app/changelog/2026-03-24-introducing-linear-agent). "Draft projects with Linear Agent" (2026-09-03) builds a project brief and plan before the project goes live — [changelog](https://linear.app/changelog).
- **Linear Triage Intelligence** is the auto-categorization benchmark:
  - It "Route[s] issues by using LLMs to infer issue properties and relationships". Suggestions cover "teams, projects, assignees, and labels" and duplicates or related issues with "strong semantic similarity". It "uses historical trends evaluated against the current issue's content".
  - Each property can be shown, hidden or auto-applied. Guidance can be set at workspace, team and sub-team level, and the most local guidance is weighted most heavily. Hovering shows the reasoning.
  - Processing takes "about 1–4 minutes". It is Business/Enterprise only; other plans get "quick, search-based suggestions" that are "faster but much less thorough".
  - Linear recommends adding guidance reactively to fix persistent mistakes.

  — [docs](https://linear.app/docs/triage-intelligence). Auto-apply arrived on 2025-09-19 — [changelog](https://linear.app/changelog/2025-09-19-auto-apply-triage-suggestions).
- **Cursor's Linear integration** "analyzes issues and filters out non-development work automatically", and triage rules can auto-assign issues to Cursor — [Cursor blog](https://cursor.com/blog/linear), [docs](https://docs.cursor.com/en/integrations/linear) **[possibly stale, 2025]**.
- **Light-weight routing elsewhere**: Kanbots adds "an Inbox for unlabeled cards" — [README](https://github.com/leodavinci1/kanbots/blob/main/README.md). Claudine "reads the native JSONL session files… extracts titles, status, and categories automatically" — [Nimbalyst](https://nimbalyst.com/blog/claude-code-session-kanban-organize-ai-agents/) **[vendor/secondary]**. Claw-Kanban advertises "role-based auto-assignment" of tasks to agents — [GitHub](https://github.com/GreenSheep01201/Claw-Kanban). Backlog.md's `project` field ("Monorepo project/component") is limited to config-declared values — [types](https://github.com/MrLesk/Backlog.md/blob/main/src/types/index.ts).

### Inferences
- For Narwarp's prompt → ticket → project sorting, Linear's design is the one to copy:
  - propose, explain and let the user confirm
  - auto-apply only per property or above a confidence threshold
  - learn from guidance and corrections
  - fall back to an Inbox when unsure

  A local Ollama model combined with project "cards" (descriptions, keywords, linked repos and Linear teams) can reproduce the "historical trends" signal by retrieving recently confirmed tickets for each project.
- Store the decomposition as a reviewable artifact (Paperclip's `issue_plan_decompositions`, Backlog.md's checkpoint #1) rather than writing child tickets straight onto the board. This keeps an LLM's bad split from flooding the board.

### Gaps
- No published accuracy numbers exist for Linear Triage Intelligence or any agent tool's classification.
- I found no open-source tool that classifies into *projects* or *life areas* with embeddings, and no evidence on how well agent-driven decomposition (Task Master, CCPM) holds up beyond anecdotes.

---

## 4. Ticket → agent work: worktree per attempt, supported CLIs, how status flows back, review/merge, the definition of "done", and board UX

### Takeaway
The standard execution unit is **one ticket → one or more runs, each in its own git worktree or branch**. The board learns run state from **harness-side signals**, not from the agent's discipline. Those signals are hooks or streamed events (Cline Kanban, Kanbots, VK), activity emissions (Linear), polling plus reconciliation against the tracker (Symphony), or leases and heartbeats (Beads, Paperclip). The mature tools define success as **reaching a review or handoff state**, not "Done". Done then needs a human gate: Symphony's `Human Review`, Paperclip's `reviewPolicy: human_only`, Multica's "Work lands in review, not in main".

### Cited Findings
**Isolation and run mapping**
- **VK**: "each workspace gives an agent a branch, a terminal, and a dev server". Supports 10+ CLIs: "Claude Code, Codex, Gemini CLI, GitHub Copilot, Amp, Cursor, OpenCode, Droid, CCR, and Qwen Code" — [README](https://github.com/BloopAI/vibe-kanban/blob/main/README.md). Execution processes are typed by `run_reason` (setupscript / codingagent / devserver / cleanupscript) — [migration](https://github.com/BloopAI/vibe-kanban/blob/main/crates/db/migrations/20251216142123_refactor_task_attempts_to_workspaces_sessions.sql).
- **Kanbots**: "Each run is isolated in a per-run worktree; a pre-push hook prevents agents from pushing". "Decision prompts pop into the UI; click an option, the run continues". "Promote — land an agent's worktree as a real commit, or open a draft PR". It supports 11 CLIs "plus any ACP-compatible CLI"; CCR "routes to alternative models" — [README](https://github.com/leodavinci1/kanbots/blob/main/README.md).
- **Cline Kanban**: an "ephemeral worktree just for that task… symlinks gitignored files like node_modules". It "uses hooks to display the latest message or tool call on each card, so you can monitor hundreds of agents at a glance". Linking cards lets "linked tasks auto-start" when a card completes, giving "fully autonomous dependency chains". "Commit or Open PR" sends a dynamic prompt so the agent turns the worktree into a commit or PR "and work[s] through any merge conflicts" — [README](https://github.com/cline/kanban/blob/main/README.md).
- **Symphony**:
  - It is a "long-running automation service that continuously reads work from a configured issue tracker, creates an isolated workspace for each issue". It polls "on a fixed cadence and dispatch[es] work with bounded concurrency", and must "Stop active runs when issue state changes make them ineligible".
  - "Ticket writes (state transitions, comments, PR links) are typically performed by the coding agent through provider-native tools". "A successful run can end at a workflow-defined handoff state (for example `Human Review`)". Restart recovery works "without requiring a persistent database".

  — [SPEC.md](https://github.com/openai/symphony/blob/main/SPEC.md). The sample WORKFLOW sets `active_states: [Todo, In Progress, Merging, Rework]`, `terminal_states: [Closed, Cancelled, Canceled, Duplicate, Done]`, `polling.interval_ms: 5000`, `max_concurrent_agents: 10`, `max_turns: 20`, and workspace hooks `after_create` / `before_remove`. The prompt template exposes `{{ issue.identifier }}`, `{{ attempt }}` and others — [WORKFLOW.md](https://github.com/openai/symphony/blob/main/elixir/WORKFLOW.md).
- **Cyrus**: on assignment it "creates isolated Git worktrees for each issue, runs (Claude Code|Codex|Cursor|Gemini|Opencode)", and can run under tmux, pm2 or systemd — [README](https://github.com/ceedaragents/cyrus).
- **Agor**: "Every piece of work is a git branch with its own working directory, dev environment, conversation history, and PR". Ports are auto-assigned so branches never collide. Session trees support fork/spawn. "Agor also exposes itself over MCP, so agents can fork, spawn, schedule, and report on their own work" — [README](https://github.com/preset-io/agor/blob/main/README.md).
- **Aperant**: "up to 12 agent terminals", worktrees, a "Self-Validating QA" loop and "AI-Powered Merge" — [README](https://github.com/AndyMik90/Aperant/blob/develop/README.md).
- **Paperclip**: "Atomic task checkout. A single assignee and execution locks prevent competing runs from claiming the same task". "Agents wake for assigned work, follow-up messages, or configured schedules" (heartbeats). Adapters cover OpenClaw, Claude Code, Codex, Cursor, Gemini CLI, OpenCode, Pi, Hermes, Grok Build, Kimi, plus "Custom processes, HTTP endpoints" — [README](https://github.com/paperclipai/paperclip/blob/master/README.md).
- **Beads**: `bd ready` lists unblocked work. `bd update <id> --claim` "Atomically claim[s] a task (sets assignee + in_progress)" — [README](https://github.com/gastownhall/beads/blob/main/README.md). Claims carry lease TTLs and heartbeats — [types.go](https://github.com/gastownhall/beads/blob/main/internal/types/types.go).
- **Multica**: "Execution log → Replay every tool call", "Steer a running agent → Reply while it works and the message lands in the current run", "Retries and timeouts → Failed runs retry on their own, or stop and tell you why", and a "Usage analytics" cost view per agent and per issue — [README](https://github.com/multica-ai/multica/blob/main/README.md).
- **Linear**:
  - "Session state… updates from the agent's emitted activities. No manual state management is required." An agent must "Emit a `thought` activity within 10 seconds" — [developers/agents](https://linear.app/developers/agents).
  - "Linear tracks session lifecycle automatically based on the last emitted activity". Webhook actions `created` and `prompted` drive the loop — [agent-interaction](https://linear.app/developers/agent-interaction).
  - Coding sessions return "a new diff for review" with preview links — [changelog](https://linear.app/changelog/2026-06-11-coding-sessions). Live diffs while the agent works arrived 2026-10-08 — [changelog](https://linear.app/changelog).
- **GitHub Agent HQ**: agents "submit draft pull requests", and output goes through the same review as a teammate's — [GitHub blog](https://github.blog/news-insights/company-news/pick-your-agent-use-claude-and-codex-on-agent-hq/).

**Review and the definition of done**
- Backlog.md: "Mark verified work Done". `task complete` moves a task off the board while keeping its record. `task archive` is for "canceled, duplicate, or invalid work" — [README](https://github.com/MrLesk/Backlog.md/blob/main/README.md).
- Paperclip has review policies `anyone | not_creator | human_only` — [constants.ts](https://github.com/paperclipai/paperclip/blob/master/packages/shared/src/constants.ts). "Configure review and approval stages" — [README](https://github.com/paperclipai/paperclip/blob/master/README.md).
- Symphony: "Workflow-specific success often means 'reached the next handoff state' (for example `Human Review`) rather than tracker terminal state `Done`" — [SPEC.md](https://github.com/openai/symphony/blob/main/SPEC.md).
- VK: "Review diffs and leave inline comments — send feedback directly to the agent… Create pull requests and merge" — [README](https://github.com/BloopAI/vibe-kanban/blob/main/README.md).

**Board UX patterns**
- Agor zones, where dropping a card fires a templated prompt, turn columns into automation triggers — [README](https://github.com/preset-io/agor/blob/main/README.md).
- Kanbots has 5 columns plus an Inbox. In GitHub mode, moves are mirrored as `status:*` labels — [README](https://github.com/leodavinci1/kanbots/blob/main/README.md).
- Backlog.md offers a live terminal board and `backlog board export` "shareable markdown reports" — [README](https://github.com/MrLesk/Backlog.md/blob/main/README.md).
- Multica's inbox severities (`action_required, attention, info`) mean "Get pinged when an agent needs a call, not for every step" — [README](https://github.com/multica-ai/multica/blob/main/README.md), [schema](https://github.com/multica-ai/multica/blob/main/server/migrations/001_init.up.sql).
- Nimbalyst claims sessions "automatically move between columns based on their actual state" — [Nimbalyst](https://nimbalyst.com/blog/claude-code-session-kanban-organize-ai-agents/) **[vendor]**.
- Linear session `plan` checklists come with `pending/inProgress/completed/canceled` items — [agent-interaction](https://linear.app/developers/agent-interaction).

### Inferences
- Narwarp's run state should come from Claude Code hooks, stream-json events and process exits, and from Ollama-agent events. It should not depend on agents remembering to call "update status". Agents should only be allowed to *request* `in_review` with a summary and work products. Promotion to `done` should stay a human or rule decision.
- Narwarp also needs a leasing or checkout mechanism (Beads, Paperclip) so two agents never grab the same ticket.
- No reviewed tool names Ollama. Kanbots/VK route alternative models through CCR, Paperclip supports "Custom processes, HTTP endpoints", and Linear added open-weight models on 2026-09-24. A generic adapter interface (command or HTTP plus an event protocol) is the norm.

### Gaps
- I did not verify how VK or Kanbots detect "agent finished" internally (process exit versus hooks). The worktree internals are covered by the runtime researcher.
- I found no data on how many parallel runs per ticket users actually find useful.

---

## 5. External tracker sync (Linear, GitHub Issues, Jira): direction, conflicts, ID mapping, and one internal item ↔ many external issues

### Takeaway
Two-way sync is rare. The best-documented example is **Beads ↔ Linear**: pull, push or bidirectional, incremental, newest-timestamp-wins with prefer-local and prefer-Linear overrides, configurable state, priority, label and relation maps, and a 5-minute debounce "to prevent agent loops". The other approaches are:
- **External tracker as the only source of truth**: Symphony, Cyrus, CCPM.
- **One-way push**: CCPM epic → GitHub epic issue plus sub-issues.
- **Mirroring board moves as labels**: Kanbots.
- **Linking without syncing**: Paperclip's `external_objects` and `external_object_mentions`, a many-to-many, live-resolved cache.

**No tool offers "one internal ticket aggregates many Linear issues" as a first-class synced feature.** The closest precedents are:
- Beads' `--milestones`, which turns one Linear projectMilestone into one local epic with many child issues
- Paperclip's many-to-many external-object mentions
- VK's `pull_request_issues` join table
- CCPM's epic → sub-issue fan-out

### Cited Findings
- **Beads ↔ Linear** — [cli-reference/linear.md](https://github.com/gastownhall/beads/blob/main/docs/cli-reference/linear.md), [linear-workflow example](https://github.com/gastownhall/beads/blob/main/examples/linear-workflow/README.md):
  - `bd linear sync --pull | --push | (none = "Bidirectional sync: pull then push, with conflict resolution")`, plus `--dry-run`, `--pull-if-stale --threshold 20m`, and "A 5-minute debounce prevents agent loops".
  - Conflicts: "Default: newer timestamp wins conflicts". `--prefer-local` and `--prefer-linear` override this. "Incremental Sync: Only sync issues changed since last sync". `--create-only` and `--update-refs=false` exist.
  - Mapping config: `linear.priority_map.*` ("0 (no priority) → 4 (backlog)", "1 (urgent) → 0 (critical)"…). `linear.state_map.backlog/unstarted/started/completed/canceled` map onto beads statuses. `linear.label_type_map.*` maps labels to issue types. `linear.relation_map.blocks/blockedBy/duplicate/related` maps relations. `linear.team_ids` supports multiple teams; `linear.project_id` restricts sync to one project.
  - ID mapping: `bd create "Fix login" --external-ref https://linear.app/team/issue/TEAM-123`. `bd linear status` shows "Issues pending push (no external_ref)". This is a 1:1 `external_ref`.
  - **Aggregation precedent**: "With `--milestones`, bd creates or reuses one local epic per Linear `projectMilestone`, then adds parent-child links from each pulled issue to its milestone epic. Milestone epics are marked as Linear milestone records and are skipped by later Linear pushes".
  - Auth: personal API key, or OAuth client credentials so CI workers act as `actor=application`.
- Beads also ships GitHub, GitLab and Jira adapters, plus an `ado` adapter that is presumably Azure DevOps (`internal/github`, `internal/gitlab`, `internal/jira`, `internal/ado`), and CLI docs for GitHub and Jira — [internal/](https://github.com/gastownhall/beads/tree/main/internal), [cli-reference/jira.md](https://github.com/gastownhall/beads/blob/main/docs/cli-reference/jira.md). A dedicated sync-failure recovery guide exists — [sync-failures.md](https://github.com/gastownhall/beads/blob/main/docs/recovery/sync-failures.md).
- **CCPM → GitHub** is one-way push:
  - "Create epic issue" with `gh issue create --label "epic,epic:<name>,feature"`, then task sub-issues through the `yahsan2/gh-sub-issue` extension when installed. Large epics are created by parallel subagents in batches.
  - "pushing local epics/tasks to GitHub as issues, syncing progress as comments, and closing issues when work is done". There is a safety check that refuses to sync to the template repo.

  — [sync.md](https://github.com/automazeio/ccpm/blob/main/skill/ccpm/references/sync.md). The README states "CCPM uses GitHub Issues as the source of truth" — [README](https://github.com/automazeio/ccpm/blob/main/README.md).
- **Symphony ↔ Linear** has no database. The tracker is the truth, and the agent writes Linear through a Linear MCP server or an injected `linear_graphql` tool. The prompt says: "Treat a single persistent Linear comment as the source of truth for progress. Use that single workpad comment for all progress and handoff notes; do not post separate 'done'/summary comments". Another instruction: "Keep ticket metadata current (state, checklist, acceptance criteria, links)" — [WORKFLOW.md](https://github.com/openai/symphony/blob/main/elixir/WORKFLOW.md), [SPEC.md](https://github.com/openai/symphony/blob/main/SPEC.md).
- **Paperclip** links rather than syncs:
  - `external_objects(providerKey, objectType, externalId, sanitizedCanonicalUrl, canonicalIdentityHash, displayKey, displayTitle, statusKey, statusLabel, statusCategory, liveness, isTerminal, data jsonb, remoteVersion, etag, lastResolvedAt, lastChangedAt, nextRefreshAt, lastErrorCode…)` — [external_objects.ts](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/external_objects.ts).
  - `external_object_mentions(sourceIssueId, sourceKind, documentKey, propertyKey, objectId, providerKey, detectorKey, objectType, confidence default 'exact')` lets one issue mention many external objects, and an object be mentioned by many issues — [external_object_mentions.ts](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/external_object_mentions.ts).
  - Linear access goes through Linear's hosted MCP (`https://mcp.linear.app/mcp`, OAuth with dynamic client registration, `riskTier: "S2"`, `requiredResourceFilters: workspace, team, project`) — [linear.json](https://github.com/paperclipai/paperclip/blob/master/packages/shared/src/app-definitions/linear.json).
- **VK** has a cloud many-to-many `pull_request_issues` table: "Scope pull_requests per project and introduce many-to-many pull_request_issues join table" — [migration](https://github.com/BloopAI/vibe-kanban/blob/main/crates/remote/migrations/20260316000000_pull_request_issues.sql).
- **Kanbots**: "Switch to GitHub mode to drive real issues on a repo… the move is mirrored as `status:*` label edits". It also imports Sentry error groups onto the board — [README](https://github.com/leodavinci1/kanbots/blob/main/README.md).
- **Aperant**: "Linear Integration | Sync tasks with Linear for team progress tracking". GitHub/GitLab issues can be imported, investigated with AI, and turned into merge requests — [README](https://github.com/AndyMik90/Aperant/blob/develop/README.md) (direction and conflict handling not documented). **Automaker**: "Import and validate GitHub issues, convert to tasks" — [README](https://github.com/AutoMaker-Org/automaker/blob/main/README.md). **Multica**: GitHub/GitLab/Gitea/Forgejo integration (`issue_pull_request`, migration 079). No Linear sync appears in its README — [README](https://github.com/multica-ai/multica/blob/main/README.md), [migrations](https://github.com/multica-ai/multica/tree/main/server/migrations).
- **Linear constructs that integrations use**:
  - Attachments: "Attachment URL is used as an idempotent value if used in conjunction with the same issue id". You can "query an attachment, and the associated issue, by its URL" (`attachmentsForURL`). Metadata is key-value, and reserved keys (`title`, `messages`, `attributes`) render a rich modal. Webhooks fire on attachment changes — [attachments](https://linear.app/developers/attachments).
  - Agents: the `delegate` field, AgentSessions, `externalUrls` for links to run dashboards, and the session `plan` — [agents](https://linear.app/developers/agents), [agent-interaction](https://linear.app/developers/agent-interaction).
  - Beads maps Linear state *types* (backlog/unstarted/started/completed/canceled) and relations (blocks/blockedBy/duplicate/related), and reconstructs projectMilestones — [linear.md](https://github.com/gastownhall/beads/blob/main/docs/cli-reference/linear.md).
  - Third-party coding agents in Linear: Cursor, Copilot (2026-07-30) and Linear's own coding sessions — [changelog](https://linear.app/changelog).

### Inferences
- For "one Narwarp ticket ↔ many Linear issues", the evidence points to **links, not field sync**. Use a Narwarp-side `external_object` cache: provider, ID, key, URL, state *type*, remote `updatedAt`/etag and refresh times, as in Paperclip. Then add a many-to-many `ticket_external_link` table with a *role*:
  - `tracks` / aggregates
  - `spawned` (created by Narwarp)
  - `mirror` (1:1 owned)
  - `mentions`
- Linear stays the owner of Linear-issue fields. Narwarp shows a roll-up (e.g. "Linear 3/5 done") and, optionally, a rule such as "propose Done when all tracked issues reach `completed`".
- Outbound, attach the Narwarp ticket's deep link to every Linear issue it spawns or tracks. Because attachments are idempotent per URL and issue, and `attachmentsForURL` exists, the reverse lookup works without storing attachment IDs. For progress use one in-place "workpad" comment per issue (Symphony), not a stream of comments.
- If any fields do sync two ways (status on 1:1 mirrors), follow Beads' policy:
  - an explicit per-field owner
  - incremental sync by `updatedAt`
  - newest wins by default, with prefer-remote and prefer-local switches
  - a debounce so agent-triggered pushes cannot loop

### Gaps
- No tool documents conflict handling for *aggregated* relationships, e.g. what happens when a tracked Linear issue is moved to another project or is split.
- Aperant's Linear-sync semantics are undocumented in the README.
- The Linear attachments doc does not say explicitly whether one URL can be attached to many issues. Idempotency is scoped to URL plus issue, which implies it can, but this is unconfirmed — [attachments](https://linear.app/developers/attachments).

---

## 6. Do any of these tools support non-coding or personal tasks, a work-vs-life split, or calendar integration?

### Takeaway
The support is partial. Paperclip is the most "life-capable": personal or private projects, a goal hierarchy, routines, and connectors including Google Calendar, Gmail, Discord, Todoist and TickTick. Recurring agent work is now common (Linear Loops, Multica Autopilots, Paperclip Routines, Agor Schedules, Codex Automations). Beads has `due_at`, `defer_until` and `estimated_minutes`. **No ticket-centric tool has a work/life "mode" or a calendar or time-blocking view.**

### Cited Findings
- Backlog.md: `backlog init "Personal Planning" --no-git` ("for local/non-code projects") — [README](https://github.com/MrLesk/Backlog.md/blob/main/README.md).
- Paperclip:
  - Projects have `personalOwnerUserId`, `privacyOwnerUserId` and `visibility (open|private)`; goals have levels `company/team/agent/task` — [projects.ts](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/projects.ts), [constants.ts](https://github.com/paperclipai/paperclip/blob/master/packages/shared/src/constants.ts).
  - "Routines create assigned tasks on a schedule". "One deployment, many organizations. Separate tasks, agents, permissions, and activity histories for each". There are experimental chat and email connectors (Slack, Discord, Telegram, AgentMail) — [README](https://github.com/paperclipai/paperclip/blob/master/README.md).
  - App definitions include `google-calendar`, `gmail`, `discord`, `github`, `linear`, `todoist`, `ticktick` and `calendly` — [app-definitions](https://github.com/paperclipai/paperclip/tree/master/packages/shared/src/app-definitions).
- Multica: "Autopilots → Run standups, audits, and reports on a cron". "Workspaces → Separate agents, issues, and settings per team". Issues can originate from Slack, Lark chat or quick-create (`origin_type`) — [README](https://github.com/multica-ai/multica/blob/main/README.md), [migrations](https://github.com/multica-ai/multica/tree/main/server/migrations).
- Linear Loops are described in plain language, run "on a schedule or in response to an event", and can create "follow-up work from meeting transcripts" — [changelog](https://linear.app/changelog/2026-07-20-introducing-loops).
- Beads: `DueAt`, `DeferUntil` ("Hide from bd ready until this time"), `EstimatedMinutes`, and a `message` issue type for inter-agent mail — [types.go](https://github.com/gastownhall/beads/blob/main/internal/types/types.go), [README](https://github.com/gastownhall/beads/blob/main/README.md).
- Agor: "Schedules. Run standups, audits, digests, and other recurring work" — [README](https://github.com/preset-io/agor/blob/main/README.md). Codex app Automations run on a schedule into a review queue — [Codex app](https://developers.openai.com/codex/app).

### Inferences
- Narwarp's work/life split has no direct precedent. The closest analogues are hard tenant boundaries (Paperclip companies, Multica workspaces) and per-project privacy. A `mode` on the project, inherited by tickets and carried into per-mode agent permissions and integrations, would be a new contribution.
- Fields that the calendar researcher's design will need are already common: `estimate_minutes`, `due_at`, `defer_until`, `start/target_date` (VK cloud issues).

### Gaps
- I found no ticket tool that schedules time blocks from tickets. That is covered by the calendar/time-blocking researcher.

---

## 7. What do users praise and complain about, and what failure modes recur?

### Takeaway
**[practitioner]** Praise centres on the productivity jump of putting agents on cards, and on persistent, structured memory. The complaints repeat across years and tools:
1. Review becomes the bottleneck.
2. Too much work in progress and cognitive overload from parallel cards.
3. Merge conflicts and agents "clean[ing] up after agents".
4. Cost.
5. Agents marking work done without verifying it.
6. Integration and status flakiness.
7. Projects being abandoned.

The board drifting from reality is mostly addressed in *design* (reconciliation, leases, hooks) rather than reported with data.

### Cited Findings
- **Review bottleneck [practitioner]**:
  - "if we can review 10 PRs a day and AI writes one of them, we now have to review 11 PRs" (sexeriy237). Parallel PRs are "so much noise and slop" (deepdarkforest) — [HN: Show HN Vibe Kanban, 2025-07-11](https://news.ycombinator.com/item?id=44533004) **[possibly stale]**.
  - "the bottleneck moves from generation to review. agents parallelize, humans review sequentially" (cold_harbor) — [HN: Kanbots, ~2026-06](https://news.ycombinator.com/item?id=48239413).
  - The vendors agree. Backlog.md: "The bottleneck is no longer writing code. It's your attention" — [README](https://github.com/MrLesk/Backlog.md/blob/main/README.md). VK's shutdown post: engineers increasingly spend their time "planning and reviewing" — [shutdown](https://www.vibekanban.com/blog/shutdown).
- **WIP and overload [practitioner]**: "The more chats I try to run at once, the more I got lost and overwhelmed" (BosunoB). The real challenge is keeping WIP bounded (okandship). Kesor objected that the tools drop kanban's flow-limiting principle. "a mock version of work… virtual post it notes" (genxy) — [HN Kanbots](https://news.ycombinator.com/item?id=48239413).
- **Conflicts and cleanup [practitioner]**: "its cumbersome to merge the work" (speedgoose); "I end up needing agents to clean up after agents" (jms703) — [HN Kanbots](https://news.ycombinator.com/item?id=48239413). "you're still sitting around waiting for Claude to merge conflicts" (helsinki); PRs "often need to be dependent on each other" (lharries) — [HN VK](https://news.ycombinator.com/item?id=44533004).
- **Resources and cost [practitioner]**: the MacBook slows "after about four concurrent runs, mainly due to rustc" (louiskw, VK author). "tens of thousands on Opus tokens and worktrees" (helsinki) — [HN VK](https://news.ycombinator.com/item?id=44533004). Multi-agent workflows "cost too much" (hgoel) — [HN Kanbots](https://news.ycombinator.com/item?id=48239413).
- **Trust, privacy and permissions [practitioner]**: VK telemetry was on by default and "harvesting email addresses and github usernames" (gpm), fixed by an opt-in PR the same day. "asks for unlimited private access to ones repo" (csomar) — [HN VK](https://news.ycombinator.com/item?id=44533004).
- **Hidden failures and false completion**: "compounded false affirmatives" (swalsh) — [HN VK](https://news.ycombinator.com/item?id=44533004). 2026 Paperclip reviews report that "agents mark tasks done without verifying output accuracy" and that "the autonomous business promise does not hold up yet" — [eesel](https://www.eesel.ai/blog/paperclip-ai-review), [Kunal Ganglani](https://www.kunalganglani.com/blog/paperclip-ai-review-zero-human-company.md) **[secondary/SEO; summarized by search, not individually verified]**.
- **Integration and status flakiness [practitioner]**: "Cursor agent did not respond" with no visible errors, and "No default repository configured" — [Cursor forum 1](https://forum.cursor.com/t/linear-integration-stopped-working-no-visible-errors/145035), [Cursor forum 2](https://forum.cursor.com/t/linear-integration-not-working-no-default-repository-configured/146560) **[possibly stale]**. Early Cursor 2.0 parallel agents were criticised for thin docs and rough worktree handling — [Cursor forum](https://forum.cursor.com/t/cursor-2-0-x-what-are-parallel-agents-doing/138911).
- **Sync and drift (design evidence)**: Beads warns that JSONL import "cannot infer that records absent from an export were deleted" and keeps a sync-failure recovery guide — [sync-concepts](https://github.com/gastownhall/beads/blob/main/docs/core-concepts/sync-concepts.md), [sync-failures](https://github.com/gastownhall/beads/blob/main/docs/recovery/sync-failures.md). ticket's author rejected Beads' SQLite sync and "rogue background daemon" **[practitioner, competitor]** — [README](https://github.com/wedow/ticket/blob/master/README.md). Yegge calls Beads "100% vibe coded" with a "crummy architecture (by pre-AI standards)" **[self-assessment; search snippet]** — [Medium](https://steve-yegge.medium.com/beads-blows-up-a0a61bb889b4). Symphony's spec requires reconciliation and stopping runs on ineligible state changes — [SPEC.md](https://github.com/openai/symphony/blob/main/SPEC.md).
- **Abandonment**: Kanbots HN commenters said VK "stopped improving and has unfixed bugs" (KerrickStaley) — [HN](https://news.ycombinator.com/item?id=48239413). VK shut down for lack of a business model — [shutdown](https://www.vibekanban.com/blog/shutdown). Automaker is unmaintained — [LICENSE](https://github.com/AutoMaker-Org/automaker/blob/main/LICENSE).
- **Board-metaphor limits [practitioner]**: on boards, humans end up routing and decomposing by hand, and cards don't share memory across agents — [Munder Difflin blog](https://munderdiffl.in/blog/vibe-kanban-alternative/) (search snippet; not fetched).
- **Praise [practitioner]**: "it's excellent - feels like the same increase in productivity increase from when I first used Cursor" (lharries) — [HN VK](https://news.ycombinator.com/item?id=44533004). "Having agents on kanban is really a level-up in terms of what you can do and how you can organize" (aqme28) — [HN Kanbots](https://news.ycombinator.com/item?id=48239413). Yegge says "tens of thousands of people" use Beads daily **[self-reported]** — [Medium](https://steve-yegge.medium.com/beads-blows-up-a0a61bb889b4).

### Inferences
- Narwarp's board should be designed against these failure modes:
  - WIP limits per column and per mode
  - a "needs-you" inbox that shows only elicitations and failures, as Multica does
  - a review queue ordered by size or risk
  - harness-derived status rather than agent-claimed status
  - one-ticket-one-PR sizing (Backlog.md)
  - cost per ticket and per run on the card (Multica, Paperclip)
- Since Linear already holds the team's state, Narwarp's ticket layer should be the *personal orchestration* layer. It should avoid replicating Linear fields, which is the main source of drift.

### Gaps
- Reddit searches (r/ClaudeAI, r/ChatGPTCoding, r/LocalLLaMA) turned up no citable threads in this pass.
- I found no HN thread for Paperclip or Task Master, and no quantitative study of board drift or of how often agents update status.

---

## 8. Synthesis: what ticket/project data model should Narwarp adopt?

### Takeaway
Use a **central local SQLite store owned by the Narwarp daemon** as Narwarp's source of truth for projects, tickets, runs and links. Agents reach it only through a CLI and MCP that return compact JSON or markdown; they never write the database file. It should combine:
- **Paperclip/Multica**: the 7-status set, agent-or-human actors, goal ancestry and review policy
- **VK/Kanbots/Multica**: separate run (attempt) records
- **Beads**: claims, leases and dependency semantics
- **Backlog.md**: acceptance criteria, plan and final summary
- **Linear triage**: classification provenance with suggest-or-auto-apply
- **Paperclip external objects**: many-to-many external links, with Linear remaining the owner of Linear fields

### Cited Findings (evidence each design choice rests on)
- **Statuses** `backlog, todo, in_progress, in_review, done, blocked, cancelled` are used identically by Paperclip and Multica — [constants.ts](https://github.com/paperclipai/paperclip/blob/master/packages/shared/src/constants.ts), [Multica 001](https://github.com/multica-ai/multica/blob/main/server/migrations/001_init.up.sql). VK uses `inreview` — [task.rs](https://github.com/BloopAI/vibe-kanban/blob/main/crates/db/src/models/task.rs). Linear state types are backlog, unstarted, started, completed and canceled — [beads linear.md](https://github.com/gastownhall/beads/blob/main/docs/cli-reference/linear.md).
- **Priority** `urgent, high, medium, low, none` matches Multica and Linear's 0–4 scale — [Multica 001](https://github.com/multica-ai/multica/blob/main/server/migrations/001_init.up.sql), [beads linear-workflow](https://github.com/gastownhall/beads/blob/main/examples/linear-workflow/README.md).
- **Separate runs** that can outlive the ticket — [VK FK removal](https://github.com/BloopAI/vibe-kanban/blob/main/crates/db/migrations/20260217120312_remove_task_fk_from_workspaces.sql), [Kanbots agent_runs](https://github.com/leodavinci1/kanbots/blob/main/packages/local-store/src/migrations/0001-initial.ts), [Multica agent_task_queue](https://github.com/multica-ai/multica/blob/main/server/migrations/001_init.up.sql).
- **Claims and leases** — [beads types.go](https://github.com/gastownhall/beads/blob/main/internal/types/types.go), [Paperclip README "Atomic task checkout"](https://github.com/paperclipai/paperclip/blob/master/README.md).
- **Owner vs delegate** — [Linear agents](https://linear.app/developers/agents). Actor types — [Multica](https://github.com/multica-ai/multica/blob/main/server/migrations/001_init.up.sql).
- **Review gate and handoff state** — [Symphony SPEC](https://github.com/openai/symphony/blob/main/SPEC.md), [Paperclip review policies](https://github.com/paperclipai/paperclip/blob/master/packages/shared/src/constants.ts).
- **Classification UX** — [Linear Triage Intelligence](https://linear.app/docs/triage-intelligence). Inbox for unsorted work — [Kanbots](https://github.com/leodavinci1/kanbots/blob/main/README.md).
- **Decomposition as a stored artifact** — [Paperclip issue_plan_decompositions](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/issue_plan_decompositions.ts), [Backlog.md checkpoints](https://github.com/MrLesk/Backlog.md/blob/main/README.md).
- **External links** — [Paperclip external_objects](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/external_objects.ts) and [mentions](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/external_object_mentions.ts). Aggregation precedent — [beads --milestones](https://github.com/gastownhall/beads/blob/main/examples/linear-workflow/README.md). Idempotent Linear attachments — [Linear attachments](https://linear.app/developers/attachments). Single workpad comment — [Symphony WORKFLOW](https://github.com/openai/symphony/blob/main/elixir/WORKFLOW.md). Conflict policy and debounce — [beads linear.md](https://github.com/gastownhall/beads/blob/main/docs/cli-reference/linear.md).
- **Work products, PR↔issue many-to-many** — [Paperclip issue_work_products](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/issue_work_products.ts), [VK pull_request_issues](https://github.com/BloopAI/vibe-kanban/blob/main/crates/remote/migrations/20260316000000_pull_request_issues.sql).
- **Origin tracking**, for tickets from Discord, email or prompts — [Paperclip originKind](https://github.com/paperclipai/paperclip/blob/master/packages/db/src/schema/issues.ts), [Multica origin_type migrations](https://github.com/multica-ai/multica/tree/main/server/migrations).
- **Scheduling fields** — [beads DueAt/DeferUntil/EstimatedMinutes](https://github.com/gastownhall/beads/blob/main/internal/types/types.go).
- **Central rather than repo-local storage** — Claude Code Tasks live in `~/.claude/tasks` and are shared through a list ID — [ClaudeLog](https://www.claudelog.com/faqs/what-are-tasks-in-claude-code/) **[secondary]**. Beads abandoned JSONL as source of truth — [sync-concepts](https://github.com/gastownhall/beads/blob/main/docs/core-concepts/sync-concepts.md).

### Inferences (recommended model; SQLite DDL sketch)
```sql
-- Narwarp ticket store (single writer: narwarp daemon; agents use CLI/MCP). IDs are ULIDs.
CREATE TABLE goal (                    -- optional; enables "why" context + life accountability (Paperclip goals)
  id TEXT PRIMARY KEY, mode TEXT NOT NULL CHECK (mode IN ('work','life')),
  title TEXT NOT NULL, description TEXT, parent_id TEXT REFERENCES goal(id),
  level TEXT NOT NULL CHECK (level IN ('north_star','area','quarter','project')),
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('planned','active','achieved','cancelled')),
  target_date TEXT, review_cadence TEXT, created_at TEXT NOT NULL, updated_at TEXT NOT NULL);

CREATE TABLE project (
  id TEXT PRIMARY KEY, key TEXT NOT NULL UNIQUE,         -- 'NW' -> tickets NW-12 (VK simple_id / Linear identifiers)
  name TEXT NOT NULL, mode TEXT NOT NULL CHECK (mode IN ('work','life')),
  is_inbox INTEGER NOT NULL DEFAULT 0,                    -- one Inbox per mode for low-confidence sorts
  status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('backlog','planned','active','completed','cancelled','archived')),
  goal_id TEXT REFERENCES goal(id),
  description TEXT,                                       -- doubles as the classifier's "project card"
  classifier_hints TEXT NOT NULL DEFAULT '{}',            -- JSON: keywords, repo paths, linear team/project ids, discord channel ids, email senders
  embedding BLOB,                                         -- of name+description+hints (local Ollama embed model)
  default_repos TEXT NOT NULL DEFAULT '[]', default_agent_profile TEXT,
  wip_limit INTEGER, ticket_counter INTEGER NOT NULL DEFAULT 0,
  color TEXT, icon TEXT, sort_order REAL NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL, updated_at TEXT NOT NULL, archived_at TEXT);

CREATE TABLE ticket (
  id TEXT PRIMARY KEY, key TEXT NOT NULL UNIQUE,          -- human key; never reuse Linear IDs as Narwarp IDs
  project_id TEXT NOT NULL REFERENCES project(id), parent_id TEXT REFERENCES ticket(id),   -- epic/sub-ticket
  type TEXT NOT NULL DEFAULT 'task' CHECK (type IN ('task','bug','feature','chore','epic','errand','habit','goal_step')),
  title TEXT NOT NULL, description TEXT,
  acceptance_criteria TEXT NOT NULL DEFAULT '[]',         -- JSON [{text, checked}] (Backlog.md / Multica)
  plan TEXT, final_summary TEXT,                          -- Backlog.md checkpoint #2 / handoff summary
  status TEXT NOT NULL DEFAULT 'backlog'
    CHECK (status IN ('backlog','todo','in_progress','in_review','done','blocked','cancelled')),
  blocked_reason TEXT,                                    -- explicit blocks; dependency blocking is derived
  priority TEXT NOT NULL DEFAULT 'none' CHECK (priority IN ('urgent','high','medium','low','none')),
  estimate_minutes INTEGER, due_at TEXT, defer_until TEXT,-- calendar/time-blocking hooks (beads)
  owner TEXT NOT NULL DEFAULT 'user',                     -- accountable human (Linear 'assignee')
  delegate TEXT,                                          -- 'agent:claude_code:<profile>' | 'agent:ollama:<model>' (Linear 'delegate')
  review_policy TEXT NOT NULL DEFAULT 'human' CHECK (review_policy IN ('human','any','none')),
  origin_kind TEXT NOT NULL DEFAULT 'manual'
    CHECK (origin_kind IN ('manual','prompt','discord','email','linear','github','calendar','agent','routine')),
  origin_ref TEXT, created_by TEXT NOT NULL,              -- 'user' | 'agent:<id>' | 'run:<id>'
  classification TEXT,                                    -- JSON {method:'rule'|'embed'|'llm', project_id, confidence, rationale,
                                                          --        state:'auto_applied'|'suggested'|'confirmed'|'overridden'}
  active_run_id TEXT, lease_expires_at TEXT,              -- atomic checkout (beads/Paperclip)
  sort_order REAL NOT NULL DEFAULT 0, metadata TEXT NOT NULL DEFAULT '{}',
  version INTEGER NOT NULL DEFAULT 0,                     -- optimistic concurrency for agent writes
  created_at TEXT NOT NULL, updated_at TEXT NOT NULL, started_at TEXT, completed_at TEXT, cancelled_at TEXT);

CREATE TABLE ticket_relation (                            -- beads/Multica/VK-remote semantics, trimmed
  ticket_id TEXT NOT NULL REFERENCES ticket(id), related_id TEXT NOT NULL REFERENCES ticket(id),
  type TEXT NOT NULL CHECK (type IN ('blocks','related','duplicates','discovered_from')),
  PRIMARY KEY (ticket_id, related_id, type), CHECK (ticket_id <> related_id));
CREATE TABLE label (id TEXT PRIMARY KEY, mode TEXT, name TEXT NOT NULL, color TEXT, UNIQUE(mode, name));
CREATE TABLE ticket_label (ticket_id TEXT REFERENCES ticket(id), label_id TEXT REFERENCES label(id), PRIMARY KEY (ticket_id, label_id));

CREATE TABLE run (                                        -- one row per attempt (VK workspace+session / Linear AgentSession)
  id TEXT PRIMARY KEY, ticket_id TEXT REFERENCES ticket(id) ON DELETE SET NULL,   -- runs may outlive/lack tickets (VK 2026-02)
  agent_kind TEXT NOT NULL CHECK (agent_kind IN ('claude_code','ollama','codex','other')),
  agent_profile TEXT, model TEXT, agent_session_id TEXT,  -- Claude Code session UUID for --resume (beads ClosedBySession)
  repo_path TEXT, branch TEXT, worktree_path TEXT,
  status TEXT NOT NULL DEFAULT 'queued'
    CHECK (status IN ('queued','running','awaiting_input','succeeded','failed','cancelled','stale')),
  heartbeat_at TEXT, started_at TEXT, ended_at TEXT, exit_reason TEXT,
  tokens_in INTEGER, tokens_out INTEGER, cost_usd REAL, summary TEXT);

CREATE TABLE work_product (                               -- Paperclip issue_work_products / VK pull_requests
  id TEXT PRIMARY KEY, ticket_id TEXT REFERENCES ticket(id), run_id TEXT REFERENCES run(id),
  kind TEXT NOT NULL CHECK (kind IN ('pr','commit','branch','doc','file','calendar_event','message')),
  provider TEXT, external_id TEXT, url TEXT, status TEXT, review_state TEXT DEFAULT 'none', is_primary INTEGER DEFAULT 0);

CREATE TABLE external_object (                            -- cache of remote truth (Paperclip external_objects)
  id TEXT PRIMARY KEY, provider TEXT NOT NULL CHECK (provider IN ('linear','github','discord','gcal','email')),
  object_type TEXT NOT NULL,                              -- 'issue' | 'project' | 'milestone' | 'pr' | 'thread' | 'event'
  external_id TEXT NOT NULL, key TEXT, url TEXT, title TEXT,  -- e.g. Linear UUID + 'ENG-123'
  state_name TEXT, state_category TEXT CHECK (state_category IN ('backlog','unstarted','started','completed','canceled','unknown')),
  assignee TEXT, priority INTEGER, parent_external_id TEXT, container_external_id TEXT,   -- Linear parent issue / project
  remote_updated_at TEXT, etag TEXT, last_synced_at TEXT, next_refresh_at TEXT, last_error TEXT, raw TEXT,
  UNIQUE (provider, object_type, external_id));

CREATE TABLE ticket_external_link (                       -- the one-ticket <-> many-Linear-issues mechanism
  ticket_id TEXT NOT NULL REFERENCES ticket(id), external_object_id TEXT NOT NULL REFERENCES external_object(id),
  role TEXT NOT NULL CHECK (role IN ('mirror','tracks','spawned','mentions','blocked_by')),
  sync_policy TEXT NOT NULL DEFAULT 'read_only' CHECK (sync_policy IN ('read_only','push_status','two_way')),
  workpad_comment_id TEXT,                                -- single in-place progress comment per Linear issue (Symphony)
  created_by TEXT NOT NULL, created_at TEXT NOT NULL, PRIMARY KEY (ticket_id, external_object_id));

CREATE TABLE decomposition (                              -- reviewable split before children hit the board (Paperclip)
  id TEXT PRIMARY KEY, source_ticket_id TEXT REFERENCES ticket(id), input_text TEXT NOT NULL,
  model TEXT, proposal TEXT NOT NULL,                     -- JSON children w/ AC, deps, project guesses
  status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','accepted','rejected')),
  child_ticket_ids TEXT NOT NULL DEFAULT '[]', created_at TEXT NOT NULL);

CREATE TABLE ticket_event (                               -- append-only audit (status changes, classification, sync);
  id TEXT PRIMARY KEY, ticket_id TEXT, run_id TEXT, actor TEXT NOT NULL,   -- the agent message board itself is
  kind TEXT NOT NULL, payload TEXT NOT NULL, created_at TEXT NOT NULL);    -- designed by the message-board researcher
```
**Behavioral rules suggested by the evidence:**
1. **Board.** Show the user's four columns (Backlog/Todo/In Progress/Done). Render `in_review` as a lane or badge inside In Progress, or as an optional fifth column, and render `blocked` and `cancelled` as badges or filters. Swimlanes are projects; the top-level toggle is mode. Cards show run state (running, awaiting input, failed), PR state, cost and a "Linear 3/5" roll-up. Enforce per-project and per-mode WIP limits (HN WIP complaints). Optionally, Agor-style "zones": dropping into In Progress dispatches the project's default agent.
2. **Status authority.** Harness hooks and events move runs. Agents can set `in_review` with `final_summary` and `work_product`s. Only the user, or an explicit rule for low-risk life tasks, sets `done` when `review_policy='human'` (Symphony, Paperclip, Multica).
3. **Prompt → ticket.** Use a local LLM to extract title, AC, type, estimate and due date. Then sort through:
   1. deterministic rules (repo, Linear team, Discord channel, email sender)
   2. embedding similarity against project cards plus recent confirmed tickets
   3. an LLM pick with a rationale

   Auto-apply above a threshold; otherwise file to the mode's Inbox with the suggestion. Corrections update `classifier_hints` and serve as few-shot examples. This mirrors Linear Triage Intelligence's show/hide/auto-apply and guidance model. Large inputs produce a `decomposition` draft, never direct children.
4. **Linear one-to-many.**
   - Create Linear issues from a ticket as `spawned` links, optionally under a Linear parent issue, CCPM-style. Attach the Narwarp deep link to each issue so `attachmentsForURL` gives the reverse lookup. Keep one workpad comment per issue.
   - Link existing issues as `tracks`. Linear owns their fields; Narwarp only reads and rolls up.
   - Use `mirror` + `push_status` only for 1:1 tickets. Resolve conflicts newest-wins by default with prefer-remote and prefer-local switches, sync incrementally by `updatedAt`, and debounce agent-triggered pushes for 5 minutes (Beads).
5. **Agent access.** `narwarp ticket ready|show|claim|update --json` plus MCP tools. Writes include `version` for optimistic concurrency, and `claim` sets the lease and `active_run_id` atomically (Beads, Paperclip). Provide `narwarp ticket context <key>` that prints ticket → project → goal ancestry (Paperclip "goal-aware execution"; `bd prime`).
6. **Mode isolation.** Each mode maps to its own allowed integrations and agent permissions, for example a work mode bound to Linear, Discord and the work calendar, and a life mode bound to personal email and the personal calendar. This is the analogue of Paperclip's company boundaries and Multica's workspaces.
7. **Exports.** Offer an optional per-project markdown export (Backlog.md-style frontmatter plus AC checkboxes) for git history and human reading. It is a view, not a second source of truth (Beads' JSONL lesson).

### Gaps
- None of the reviewed tools validates the aggregation semantics for one ticket and many Linear issues, such as roll-up rules or handling when a tracked issue is moved or split. Narwarp will have to define and test them.
- I found no published thresholds for auto-applying LLM classification. They will need local calibration on the user's own tickets.
- The scalability of SQLite with many concurrent agent runs is assumed to be fine with a single writer (WAL mode) but was not benchmarked here. Process and runtime architecture belongs to the runtime researcher.
