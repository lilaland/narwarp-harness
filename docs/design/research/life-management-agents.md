# Life-management AI agents and accountability systems (2025 – Oct 2026): architectures, goal tracking, memory, safety, and design recommendations for Narwarp "Life mode"

Scope and conventions: the state of things as of 2026-10-13. Evidence labels used below: **[Primary]** official docs/repos/papers, **[Vendor]** company-published claims, **[Meta-analysis]**, **[RCT]**, **[Secondary]** news/blogs/aggregators, **[Practitioner]** individual user opinion, **[Prior knowledge]** well-known source cited from memory, not re-fetched this session. **(>6 mo)** marks product facts older than ~mid-April 2026; treat them as possibly stale. Out of scope here (other researchers cover them): kanban/ticket harnesses, runtime architecture, multi-agent message boards, calendar time-blocking algorithms and the Google Calendar API, app-integration APIs, Claude Code/Ollama specifics, TUI frameworks.

---

## 1. How always-on personal agents are built: OpenClaw, Hermes Agent, Letta, Khoj, Leon, Anthropic, OpenAI, Google. What recurs?

### Takeaway
The open-source personal agents (OpenClaw, Hermes Agent, Leon 2.0, Letta Code) have settled on one architecture. A long-running **gateway** process connects **chat channels** (Telegram/Discord/Slack/WhatsApp/Signal/email). A **scheduler** runs periodic "heartbeat/pulse" turns plus cron jobs. Memory is **plain markdown files**: a small pinned profile, a curated long-term file and daily logs, with an SQLite/FTS or hybrid index on top and background "dreaming" consolidation. The big vendors (Anthropic, OpenAI, Google) moved the same ideas to **cloud execution** in 2026. OpenAI retired its passive proactive feed (Pulse) in favor of **user-defined, steerable scheduled tasks**. Security has been the main failure: OpenClaw's growth came with exposed gateways, malware skills, zero-click email RCEs and an agent that deleted an inbox.

### Cited Findings

#### Status snapshot (verify-before-use; as of 2026-10-13)
| System | Current status | Key dates | Notes |
|---|---|---|---|
| OpenClaw (ex-Warelay/CLAWDIS/Clawdbot/Moltbot) | Active; stewarded by an OpenClaw Foundation; **OpenClaw 2.0 (v2026.8.1)** announced Aug 30, 2026 | First published Nov 2025 as "Warelay"; CLAWDIS Dec 3, 2025; Clawdbot Jan 2, 2026; renamed Moltbot Jan 27, 2026 after Anthropic trademark complaints; renamed OpenClaw 3 days later; Steinberger announced joining OpenAI Feb 14, 2026 — [Wikipedia](https://en.wikipedia.org/wiki/OpenClaw) | 501(c)(3) incorporation on July 8, 2026 is reported only by [explainx.ai](https://explainx.ai/blog/openclaw-foundation-501c3-nonprofit-july-2026) (unverified) |
| Hermes Agent (Nous Research) | Active; MIT; ~253.4k GitHub stars, 54.9k forks at fetch time — [GitHub](https://github.com/NousResearch/hermes-agent) | Launched Feb 2026 per secondary sources ([toolradar](https://toolradar.com/blog/what-is-hermes-agent), [heyuan110](https://www.heyuan110.com/posts/ai/2026-04-14-hermes-agent-guide)); v0.20.0 (Aug 3, 2026) added voice and agent-to-agent links — [RuntimeWire](https://runtimewire.com/article/nous-research-hermes-agent-v0200-voice-a2a) | Reported $90M Series B at $1.5B valuation (citing WSJ) — [Trending Topics](https://www.trendingtopics.eu/nous-research-hermes-90-millionen/) (unverified) |
| Letta | Active; Letta Code with git-backed **MemFS** on by default for new agents (v0.15+); "dreaming" background memory consolidation — [Letta docs](https://docs.letta.com/letta-code/memfs), [Letta docs: dreaming](https://docs.letta.com/guides/agents/architectures/sleeptime) | Sleep-time compute introduced Apr 2025 — [Letta blog](https://letta.com/blog/sleep-time-compute) **(>6 mo)** | Memory blocks are migrated into MemFS (`/memfs enable`) — [Letta docs](https://docs.letta.com/letta-code/memfs) |
| Khoj | Open-source repo active (AGPL-3.0, ~37.7k stars; "Your AI second brain. Self-hostable.") — [GitHub](https://github.com/khoj-ai/khoj) | Khoj Cloud reportedly sunset Apr 15, 2026; self-hosting now the only supported route — [PromptQuorum review](https://www.promptquorum.com/power-local-llm/khoj-ai-second-brain-review) (third-party; the GitHub page shows no notice) | Team reportedly launched Open Paper and Pipali in 2026 — [ideaproof](https://ideaproof.io/open-source/project/khoj) (unverified) |
| Leon | Active, solo-maintained (MIT); 2.0 is a from-scratch agentic rebuild on the `develop` branch; classic intent-based version stays on `master` — [PromptQuorum](https://www.promptquorum.com/power-local-llm/leon-ai-review) | "Road to 2.0" post Mar 19, 2026; developer preview undated — [Leon blog](https://blog.getleon.ai/road-to-2-0/) **(>6 mo)** | |
| Anthropic (Claude memory, Cowork, scheduled tasks, routines) | Active; "Claude Cowork is now just Claude"; scheduled tasks "run in the cloud, with no device online" — [Claude Help Center](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork) | Memory: Team/Enterprise Sep 11, 2025; Pro/Max Oct 23, 2025 — [Claude blog](https://claude.com/blog/memory) **(>6 mo)** | |
| ChatGPT Pulse | **Retired.** Sunset announced Jun 17, 2026 on X ("sunsetting Pulse in the next 14 days") — [Digit](https://www.digit.in/news/general/openai-is-retiring-chatgpt-pulse-and-replacing-it-with-scheduled-tasks-here-is-why.html/amp/) | Launched to Pro in late Sep 2025 — [Sam Altman on X](https://x.com/sama/status/1971297661748953263) | Replaced by the Scheduled tasks hub (Jun 18, 2026) — [IT Brief](https://itbrief.co.uk/story/openai-expands-chatgpt-scheduled-tasks-with-new-hub) |
| Gemini Daily Brief / Gemini Spark | Daily Brief active, **free in the US** since early Sep 2026 — [9to5Google](https://9to5google.com/2026/09/08/gemini-daily-brief-free/); Spark began as beta for US Ultra subscribers — [Gemini release notes](https://gemini.google/release-notes/) | Both announced at I/O, May 19, 2026 — [Google blog](https://blog.google/innovation-and-ai/products/gemini-app/next-evolution-gemini-app/) | Spark's status after the May beta was not verified |

#### OpenClaw (the archetype)
- **Adoption:** 247,000 GitHub stars and 47,700 forks as of Mar 2, 2026. Chinese developers adapted it for DeepSeek and WeChat, and Tencent and Z.ai announced OpenClaw-based services — [Wikipedia](https://en.wikipedia.org/wiki/OpenClaw).
- **Runtime model:** OpenClaw runs locally against an external LLM (Claude, DeepSeek, GPT). Users talk to it over Signal, Telegram, Discord or WhatsApp. Skills are directories with a `SKILL.md`, and workspace skills override bundled ones — [Wikipedia](https://en.wikipedia.org/wiki/OpenClaw).
- **Gateway and channels [Primary]:** on a regular host install the Gateway binds to **loopback**. Unknown DM senders get a **pairing code**. Groups are allowlisted behind a mention gate. Container images default to an exposed bind. DM modes are pairing/allowlist/open/disabled. "OpenClaw is not a hostile multi-tenant security boundary." By default, agents with message-tool access "can send across conversations and channel providers." `openclaw security audit` checks for configuration drift — [OpenClaw docs: security](https://docs.openclaw.ai/gateway/security).
- **Heartbeat [Primary]:** "a system-owned automation that runs periodic agent turns in the main session."
  - Defaults to every **30m** (1h under Anthropic OAuth auth if unset); `every: "0m"` disables it.
  - `heartbeat.activeHours` (`start`/`end`/`timezone`) skips ticks outside the window.
  - Delivery `target` defaults to `"owner"`; `target: "none"` keeps results internal.
  - Quiet replies: the agent answers `NO_REPLY` (legacy `HEARTBEAT_OK`) when nothing needs attention. Visibility defaults to `showOk:false`, `showAlerts:true`.
  - `HEARTBEAT.md` is no longer read: the checklist moved to per-job "monitor scratch" (`openclaw cron scratch`). This means earlier write-ups describing HEARTBEAT.md are **stale**.
  - Recurring checks should become separate automation (cron) jobs.
  - Cost: `isolatedSession: true` cuts a run "from ~100K tokens down to ~2-5K per run"; also `lightContext` and a cheaper model override — [OpenClaw docs: heartbeat](https://docs.openclaw.ai/gateway/heartbeat).
- **Memory [Primary]:**
  - Files: `USER.md` (stable preferences), `MEMORY.md` (curated durable facts and decisions), `memory/YYYY-MM-DD.md` (daily notes), optional `DREAMS.md` (consolidation diary for human review). "The model only remembers what gets saved to disk; there is no hidden state."
  - Loading: USER.md and MEMORY.md load at session start; today's and yesterday's notes load on `/new`; daily notes are indexed for search but not injected.
  - A silent **pre-compaction memory flush** (on by default) saves context before summarization. **Dreaming** (on by default) promotes items from short-term recall into MEMORY.md.
  - `memory_search` is a hybrid of vector and keyword search on a built-in SQLite engine. Embeddings default to OpenAI but can be **Ollama**, LM Studio or local GGUF.
  - MEMORY.md is "not a raw transcript, daily log, or exhaustive archive"; oversize files get truncated at injection — [OpenClaw docs: memory](https://docs.openclaw.ai/concepts/memory).
- **Workspace files [Primary]:**
  - Bootstrap files: `AGENTS.md` (operating rules, loaded every session), `SOUL.md` (persona/boundaries), `USER.md` (own 4,000-char budget), `IDENTITY.md`, optional `BOOT.md`, one-time `BOOTSTRAP.md`, `skills/`.
  - Truncation defaults are 20,000 chars per file and 60,000 total.
  - "Only load `MEMORY.md` in the main, private session (not shared/group contexts)."
  - Keep the workspace in a **private git repo**, and keep secrets out even there — [OpenClaw docs: workspace](https://docs.openclaw.ai/concepts/agent-workspace).
- **2.0 security posture:** The Register's Aug 31, 2026 review found that 2.0's shared sessions lack network and file-system security boundaries, Secret Store values are not encrypted at rest, and the sandbox is not on by default — [Wikipedia citing The Register](https://en.wikipedia.org/wiki/OpenClaw).

#### Hermes Agent (Nous Research)
- **Tagline:** "The self-improving AI agent built by Nous Research"; MIT — [GitHub](https://github.com/NousResearch/hermes-agent).
- **Gateway and scheduling:** "Telegram, Discord, Slack, WhatsApp, Signal, and CLI — all from a single gateway process" (Email is listed too), plus a "Built-in cron scheduler with delivery to any platform" — [GitHub](https://github.com/NousResearch/hermes-agent).
- **Memory:** agent-curated `MEMORY.md` and `USER.md` with "periodic nudges to persist knowledge". "FTS5 session search with LLM summarization for cross-session recall." Honcho handles "dialectic user modeling" — [GitHub](https://github.com/NousResearch/hermes-agent).
- **Skills:** the agent writes skills after complex tasks and refines them in use, following the agentskills.io standard — [GitHub](https://github.com/NousResearch/hermes-agent).
- **Security and backends:** "command approval, DM pairing, container isolation". Seven terminal backends (local, Docker, SSH, Singularity, Modal, Daytona, Vercel Sandbox) — [GitHub](https://github.com/NousResearch/hermes-agent).
- **OpenClaw migration:** "If you're coming from OpenClaw, Hermes can automatically import your settings, memories, skills, and API keys" — [GitHub](https://github.com/NousResearch/hermes-agent). Some third-party sites call Hermes OpenClaw's "successor" ([hermes-agent.org](https://hermes-agent.org/about/)). It is a separate project; the official README only offers migration.
- **Skill-reuse claim [Secondary]:** agents with 20+ self-created skills reportedly finish similar tasks ~40% faster in tokens and time, but not necessarily with better quality, and skills don't transfer across domains — [Substack analysis](https://mranand.substack.com/p/inside-hermes-agent-how-a-self-improving).

#### Letta
- **Original sleep-time design (Apr 2025) (>6 mo):** a primary agent plus a background sleep-time agent that edits shared memory — [Letta blog](https://letta.com/blog/sleep-time-compute).
- **Dreaming [Primary, current docs]:** "uses background subagents to review recent conversations," consolidating lessons "without interrupting your active work."
  - It triggers "after a set number of completed agent steps or when the context window is compacted."
  - The optional "Agent reviews before applying" mode adds a second-pass review, but "does not ask you for approval."
  - Reorganization backs up the repo before "splitting large files, merging duplicates, or restructuring the hierarchy" — [Letta docs](https://docs.letta.com/guides/agents/architectures/sleeptime).
- **MemFS [Primary]:**
  - Memory is a directory of markdown files in a local git repo that the agent edits with bash and commits.
  - Files under `system/` go into the system prompt in full; other files appear only in a memory tree and load when relevant.
  - Sleep-time reflection and defragmentation run in **git worktrees**.
  - Local-only agents commit locally, and the user owns backups — [Letta docs](https://docs.letta.com/letta-code/memfs).

#### Khoj
- Khoj supports local and online LLMs. Its automations deliver newsletters and notifications, and it is reachable from Obsidian, Emacs, desktop, phone and WhatsApp — [GitHub](https://github.com/khoj-ai/khoj).
- The cloud shutdown is third-party-reported only (see table).

#### Leon 2.0 (Mar 2026 design post) (>6 mo)
- **Memory:** a `memory/` tree (`archive`, `daily`, `discussion`, `persistent`, `reports`) plus an SQLite index. A `context/private/` folder holds `LEON_PRIVATE_DIARY.md` and `PULSE.md`. "After each conversation turn, Leon saves memory fragments into its own database."
- **Proactivity:** a recurring **"pulse"** in which "Leon can inspect its pending tasks and internal reflections" and decides whether to act. The diary records whether past actions were useful.
- **Local models:** Leon tried Ollama, then "ultimately preferred to move toward Llama.cpp."
- **Context and consent:** context is injected progressively ("only the relevant context on the fly"). Environment awareness is opt-in, and onboarding is "the right moment to ask for permission clearly and transparently" — [Leon blog](https://blog.getleon.ai/road-to-2-0/).

#### Anthropic
- **Memory (>6 mo for dates):**
  - "Claude creates a separate memory for each project."
  - Incognito chats bypass memory; users view and edit a memory summary; memory can be imported and exported.
  - Pre-launch testing looked at "reinforcing harmful patterns, over-accommodation, and bypassing safeguards" — [Claude blog](https://claude.com/blog/memory).
- **Cowork / cloud sessions [Primary]:**
  - In cloud sessions Claude starts from chat memory, and task findings carry back to chat. Memory can be switched off per task, but only before the first message.
  - Approval modes: Manual, Auto (each action is reviewed for exfiltration or prompt-injection risk, but "no defense is perfect and no mode replaces your judgment"), and Skip all approvals.
  - For money, "messages sent as you" and important files, Anthropic recommends Manual.
  - Network egress permissions "don't apply to web fetch, web search, or MCPs."
  - Permanent file deletion needs explicit permission — [Claude Help Center](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork).
- **Dates [Secondary]:**
  - Third-party blogs date Cowork scheduled tasks to Feb 25, 2026, with a move to cloud execution in July 2026 and cloud-by-default from Sep 2026 — [buildtolaunch](https://buildtolaunch.substack.com/p/what-is-claude-cowork), [Palladino](https://dariopalladino.com/articles/claude-cowork-is-moving-to-the-cloud-by-default-what-that-means-for-your-files-memory-and-sensitive-data/) (not checked against Anthropic release notes).
  - Memory reportedly unified across chat and Cowork on Aug 25, 2026 — [MemoryLake blog](https://www.memorylake.ai/en/blogs/claude-memory-chat-and-cowork) (unverified date).
- **Anthropic's scheduling surfaces [Secondary, May 3, 2026 article with July updates]:**
  - Cowork scheduled tasks: cloud.
  - Claude Code Desktop "local" scheduled tasks: stored in `~/.claude/scheduled-tasks/`, 1-minute minimum, machine must be awake, one catch-up run on wake.
  - `/loop`: session-scoped; recurring tasks expire after 7 days.
  - Cloud Routines: schedule, API POST or GitHub-event triggers; 1-hour minimum; fresh clone each run; no local MCP; no state between runs — [buildtolaunch](https://buildtolaunch.substack.com/p/claude-cowork-scheduled-tasks-vs-routines-vs-loop).

#### OpenAI
- **Pulse:** pitched as working "for you overnight… your interests, your connected data, your recent chats" and delivering a morning feed — [Sam Altman on X](https://x.com/sama/status/1971297661748953263).
- **Why it was retired:** OpenAI said "proactive experiences are most useful when they're personalised, action-oriented, and steerable by the user." It also cited strong engagement with *tasks* on the Pulse surface — [Digit](https://www.digit.in/news/general/openai-is-retiring-chatgpt-pulse-and-replacing-it-with-scheduled-tasks-here-is-why.html/amp/).
- **Scheduled tasks hub (Jun 18, 2026):**
  - A sidebar "Scheduled" page to view, pause, edit and delete tasks.
  - One-off, recurring and **monitoring tasks that notify only when there is something to report**.
  - Rolled out to Go/Plus/Pro/Business/Enterprise. No voice, GPTs or webhooks — [IT Brief](https://itbrief.co.uk/story/openai-expands-chatgpt-scheduled-tasks-with-new-hub), [Gigazine](https://gigazine.net/gsc_news/en/20260619-chatgpt-scheduled-tasks/).

#### Google
- **Daily Brief [Primary]:** after opt-in, Gemini works in the background across connected apps (Gmail urgent updates, Calendar, follow-ups). It prioritizes based on the user's goals and suggests next steps, and thumbs up/down steer it (paraphrased) — [Google blog](https://blog.google/innovation-and-ai/products/gemini-app/next-evolution-gemini-app/).
  - Sections: **"Top of mind"** (timely, actionable items) and **"Looking ahead"** ("your longer-term goals" with suggested next steps).
  - Items can be marked complete, dismissed, rated Helpful/Not helpful and checked for their source.
  - Requirements: Memory on, Workspace connected, and a **personal** account ("not available with a work, school, or supervised Google Account") — [Gemini Help](https://support.google.com/gemini/answer/17077455?hl=en).
- **Gemini Spark:** a 24/7 cloud agent on Gemini 3.5 using the Antigravity harness. Example jobs include flagging hidden subscription fees, a daily digest of a school inbox, and drafting project emails.
  - Its stated guardrail is "designed to ask you first before performing high-stakes actions like spending money or sending emails."
  - Texting/emailing Spark, custom sub-agents and local-browser control were promised for summer 2026 — [Google blog](https://blog.google/innovation-and-ai/products/gemini-app/next-evolution-gemini-app/).

#### Practitioner reports (labeled opinion)
- **[Practitioner]** Running OpenClaw over Telegram on an old laptop was "not exactly the tool that solves much for my professional life." Weather/news/email were already covered by chat assistants. Agents that pick up Jira/Linear tasks were more valuable, but OpenClaw "burns tokens like a maniac" — [mkdev "Post-Hype Report"](https://mkdev.me/posts/openclaw-post-hype-report-89).
- **[Practitioner/Secondary]** Users report unexplained token drains, conversation histories that grow without bound and system prompts bloating to 29K chars, attributed to heartbeat — [GraphScope blog](https://blog.graphscope.io/openclaw-users-why-is-your-token-budget-disappearing-out-of-nowhere-21265f18b057).
- **[Secondary]** The most common real uses are daily briefings, reminders, summaries and background task runs — [Claude Market blog](https://claudemarket.ai/blog/what-people-actually-use-openclaw-for). A 50-day user says OpenClaw "amplifies what you're already doing" rather than inventing workflows — [myclaw.ai](https://myclaw.ai/ar/blog/openclaw-power-user-5-use-cases-50-days) (marketing-adjacent).
- **[Practitioner]** An HN commenter with ADHD "yearn[s] for a personal assistant" that surfaces assigned tickets, unanswered messages and deadlines and can reach out proactively — [HN comment](https://news.ycombinator.com/threads?id=jsolson).

### Inferences
- **The recurring pattern** is gateway + channels + scheduler (heartbeat/pulse plus cron) + markdown memory + skills. It shows up independently in OpenClaw, Hermes, Leon 2.0 and Letta Code, so Narwarp should adopt it rather than invent a new one. The terminal board plus Discord DMs play the role of "channels."
- **Two refinements are now standard:**
  - Heartbeats should be cheap and **silent unless actionable** (OpenClaw `NO_REPLY`/isolated sessions; ChatGPT monitoring tasks that notify only on change).
  - Proactivity should come from **explicit, user-editable schedules**, not an opaque feed. Pulse was retired after ~9 months, and OpenAI's own lesson was "steerable by the user."
- **Vendors and OSS have diverged.** Vendors run proactive work in their clouds (Cowork scheduled tasks, Spark, Cloud Routines). OSS agents keep gateways and files local. Narwarp's differentiator is local-first life data. It also inherits OSS security burdens that the vendors absorb: sandboxing, egress, approval UIs.
- **Google gates Daily Brief to personal accounts,** which shows vendors treating work and personal agents as separate products. Narwarp intends to bridge them, so it needs explicit separation rules (section 3).

### Gaps
- I could not confirm the OpenClaw Foundation's legal status (501(c)(3) claim) or the exact Hermes Agent launch date from primary sources.
- Khoj Cloud's shutdown is reported only by a third-party review.
- Gemini Spark's availability after the May 2026 beta, the history of Gemini "Agent mode" (2025), and Google's "CC" Labs briefing experiment were not verified.
- Anthropic release notes for Cowork scheduled tasks (Feb 2026) and memory unification (Aug 2026) were not fetched. Dates come from third-party blogs.
- No reliable usage or retention data exists for any of these proactive agents, beyond OpenAI's vague "strong engagement" remark.

---

## 2. Goal tracking and accountability: how tools and frameworks structure it, what behavior-change research says, and why people abandon these systems

### Takeaway
The strongest evidence supports three things:
- **Self-monitoring that is recorded and shared** (d≈0.40 on goal attainment).
- **If-then implementation intentions** (d≈0.27–0.66 across 642 tests).
- **Mental contrasting plus implementation intentions (WOOP)** (g≈0.34).

Reminders and prompts work in the short term but **habituate within weeks**. Punitive commitment devices work for the minority who accept them (~14% uptake). Highlighting **broken streaks** or failures *reduces* re-engagement, while rewarding the **comeback after a miss** was the best of 54 gym interventions. Abandonment is the default: about 3% 30-day retention for mental-health apps, and only 8% of megastudy interventions had effects lasting past the program. A Life mode must therefore be low-friction, forgiving, ritual-based and self-tapering.

### Cited Findings

#### How established tools structure accountability
- **Sunsama, daily planning ritual [Primary]:**
  - Steps: (1) reflect on yesterday if the shutdown was skipped; (2) pull in calendar events, integration tasks, backlog and weekly objectives; (3) **check predicted workload against a user-set threshold**, deferring (`D`) or backlogging (`Z`) when over; (4) order, timebox and set the **shutdown time** (optionally as a calendar event); (5) share the plan to Slack/Teams.
  - New users are advised to target well below their available hours (~5.5h for a 9–5 day with lunch).
  - Planning after 3:00 PM plans tomorrow.
  - Work and personal tasks are planned separately via channel filtering (`Shift F`) — [Sunsama Help](https://help.sunsama.com/docs/daily-planning).
- **Sunsama, shutdown ritual [Primary/Secondary]:** an in-app notification at the chosen stop time prompts the user to review and reflect. It is only available to morning planners — [Sunsama changelog](https://roadmap.sunsama.com/changelog/daily-shutdown). Reviewers single out the overcommitment warning and say the shutdown "creates a clean mental break between work and personal time" — [Asian Efficiency review](https://www.asianefficiency.com/schedule-management/sunsama-review/) [Practitioner].
- **Beeminder [Primary]:**
  - Users commit to a rate drawn as a **"bright red line"** (formerly the Yellow Brick Road). Crossing it "derails" the goal and charges the pledge.
  - Pledges escalate ($0, $5, $10, $30, …).
  - **Akrasia horizon:** changes that make a goal *easier* only take effect after a one-week delay; making it harder is immediate — [Beeminder wiki](https://wiki.beeminder.com/what-is-beeminder.html), [Beeminder glossary](https://blog.beeminder.com/glossary/).
- **Focusmate (body doubling):** company surveys claim a "143% average productivity increase" and 161% for neurodivergent members [Vendor, self-reported] — [Focusmate media kit](https://www.focusmate.com/?p=3865). CHADD notes "there hasn't been significant research to validate the benefits" — [CHADD](https://chadd.org/adhd-news/adhd-news-adults/could-a-body-double-help-you-increase-your-productivity/).
- **Todoist (anti-guilt features) [Primary]:**
  - Configurable **Days off** (no Karma gained and no streak lost) and **Vacation mode** (Karma and streaks stay intact).
  - Goals can be set to 0.
  - Support can restore a broken streak up to 3 times per account — [Todoist Help](https://todoist.com/help/articles/introduction-to-karma).
  - Todoist's own blog endorses **"to-do list bankruptcy"**: archive everything rather than delete it, because incremental cleanup of a deep overdue backlog is "a losing battle" — [Todoist](https://todoist.com/inspiration/todo-list-bankruptcy).
- **AI accountability apps [Vendor]:**
  - CoachCall.ai runs daily WhatsApp check-ins and weekly voice coaching calls, with no user ratings yet — [GetApp](https://www.getapp.com/all-software/a/coachcall/).
  - "AI Accountability Coach" (iOS) gives one coach per goal that "remembers your target, motivation, and recent progress," plus heatmaps and weekly AI summaries of "wins, gaps, patterns, and practical next steps" — [App Store](https://apps.apple.com/app/id6763578748).
  - No independent reviews were found.
- **Gemini Daily Brief "Looking ahead"** is the only big-vendor surface explicitly about *longer-term goals*. It suggests next steps and offers mark-complete, dismiss and helpful/not-helpful controls — [Gemini Help](https://support.google.com/gemini/answer/17077455?hl=en).

#### Frameworks
- **GTD Weekly Review [Primary]:** three phases.
  - *Get Clear:* collect loose items and process inboxes to zero.
  - *Get Current:* review action lists, the past and upcoming calendar, Waiting For and project lists.
  - *Get Creative:* review Someday/Maybe and generate new ideas.
  - Sources: [GTD checklist PDF](https://gettingthingsdone.com/wp-content/uploads/2014/10/Weekly_Review_Checklist.pdf), [GTD Dictionary](https://www.facilethings.com/gtd-dictionary/en/weekly-review).
- **12 Week Year [Secondary summaries of Moran & Lennington]:** the year is treated as 12 weeks.
  - The weekly **execution score** is tactics completed ÷ tactics planned, so it scores effort, not outcomes. **85%** is the rule-of-thumb benchmark.
  - A 15–20 min **Weekly Accountability Meeting** of 2–4 people reports scores and next-week intentions — [yu-wenhao guide](https://yu-wenhao.com/en/blog/12-week-year-guide/), [dsebastien](https://concepts.dsebastien.net/concept/12-week-year/), [monday.com](https://monday.com/blog/productivity/12-week-year-time-management-system/).
  - The 85% figure is a heuristic, not an empirically validated threshold.
- **Implementation intentions and WOOP** are covered under Evidence below.

#### Behavior-change evidence (with quality labels)
- **Self-monitoring [Meta-analysis, high quality]:**
  - Harkin et al. 2016 (*Psychological Bulletin*): 138 randomized studies, N=19,951.
  - Interventions raised monitoring frequency (d+=1.98) and goal attainment (**d+=0.40**, 95% CI 0.32–0.48), with monitoring frequency mediating the effect.
  - Effects were **larger when progress was reported or made public, and when physically recorded**.
  - Studies were mostly health goals — [White Rose eprint](https://eprints.whiterose.ac.uk/91437/), [APA release](https://www.apa.org/news/press/releases/2015/10/progress-goals).
- **Implementation intentions [Meta-analyses, high quality]:**
  - Gollwitzer & Sheeran 2006: 94 tests, **d=0.65** — [Univ. Konstanz](https://www.socmot.uni-konstanz.de/publications/implementation-intentions-and-goal-achievement-meta-analysis-effects-and-processes).
  - The much larger Sheeran, Listrom & Gollwitzer 2025 (*Eur. Rev. Soc. Psych.*, 642 tests) found effects across cognitive, affective and behavioral outcomes (**.27 ≤ d ≤ .66**). Effects were larger for **contingent if-then format, high motivation, and rehearsed plans** — [KOPS](https://kops.uni-konstanz.de/handle/123456789/69905).
- **Mental contrasting plus implementation intentions (MCII/WOOP) [Meta-analysis, moderate]:**
  - Wang et al. 2021: 21 studies, ~15,900 participants, **g=0.336**. Effects were larger with interactive delivery (g=0.465) than document-only (g=0.277), and publication bias may inflate them — [PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8149892/).
  - Cross & Sheffield 2019 (health behavior): g≈0.28 — [Derby repository](https://repository.derby.ac.uk/item/93z0q/mental-contrasting-for-health-behaviour-change-a-systematic-review-and-meta-analysis-of-effects-and-moderator-variables).
- **Commitment devices [RCT, high quality]:**
  - Halpern et al. (NEJM 2015), smoking cessation: **90.0% accepted reward-based programs vs 13.7% deposit-based**.
  - Intention-to-treat 6-month abstinence was 15.7% (reward) vs 10.2% (deposit) vs 6.0% (usual care).
  - Among people who would accept either, deposits produced 13.2 points *higher* abstinence (instrumental-variable analysis) — [Penn repository abstract](https://repository.upenn.edu/hcmg_papers/25).
  - Takeaway: stakes work for the self-selected few, and most people refuse them.
- **Megastudy [Large field experiment, high quality]:**
  - Milkman et al. 2021 (*Nature*): 54 four-week programs tested on 61,000+ gym members.
  - ~45% raised weekly visits by 9–27%. **The top intervention gave micro-rewards for returning after a missed workout.**
  - **Only 8% of interventions produced significant behavior change after the four weeks ended.**
  - Expert forecasters failed to predict the winners — [paper PDF](https://www.gwern.net/doc/statistics/prediction/2021-milkman.pdf), [WEF summary](https://www.weforum.org/stories/health-and-healthcare-systems/science-fitness-goal-motivation/), [Knowledge@Wharton](https://knowledge.wharton.upenn.edu/?p=19390).
- **Streaks [7 studies, JCR 2023, moderate-high]:**
  - Highlighting a **broken** streak led to *less* subsequent engagement than highlighting an intact one, even when actual past behavior was identical.
  - The effect was worse when people blamed themselves and weaker when the streak could be repaired.
  - The authors found messages highlighting a recent failure "typically didn't work" and recommend offering a new goal or changing the goal's time period — [CU Boulder](https://www.colorado.edu/business/news/2023/04/20/research-streaks-marketing-tech-barasch), [UDel repository](https://udspace.udel.edu/items/42ce576b-8e1f-429a-8541-e29e48dbcfbb).
- **Prompts/reminders habituate [Micro-randomized trial, moderate]:**
  - HeartSteps: an activity suggestion initially raised 30-min step counts by 66% (~167 steps), but "that effect diminished over time." The average effect was 14% (35 steps, p=.06) — [PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC6401341).
  - A lecture summary describes a ~2%/day decline, indistinguishable from zero by day 28, and attributes it to habituation — [Cambridge lecture](https://www.cl.cam.ac.uk/teaching/2324/MH/MH-lecture12.pdf) [Secondary].
  - HeartSteps V2 sent messages only after 40 sedentary minutes, with adaptive randomization — [arXiv methods paper](https://arxiv.org/pdf/2107.03544).
- **Timing and receptivity [Field studies, moderate]:**
  - Künzler et al. 2019: 189 participants over 6 weeks with a chatbot coach (Ally); receptivity depends on context — [Dartmouth](https://www.cs.dartmouth.edu/dfk/research/kunzler-receptivity/index.html).
  - ML-timed delivery improved receptivity by **up to 40%** vs random timing, and the adaptive model improved over time — [PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC8680205). A later highlight reports up to 36% — [GetMobile highlight](https://cs.dartmouth.edu/dfk/research/mishra-receptivity-highlight/mishra-receptivity-highlight.pdf).
- **Notification batching [Single field RCT, moderate]:**
  - Fitz et al. 2019 (N=237, India): batching notifications **3×/day** improved attention, productivity, mood and sense of control, and lowered stress.
  - **No notifications at all** raised anxiety and FoMO; **hourly** batching made little difference.
  - Outcomes were self-reported — [Duke Scholars](https://scholars.duke.edu/publication/1402953), [Georgia Tech Work21](https://work21.gatech.edu/?p=402).
- **Habit formation [Single study, moderate]:** automaticity took an average of **66 days** (UCL) — [UCL](https://www.ucl.ac.uk/news/2009/aug/how-long-does-it-take-form-habit). The range was 18–254 days, and "missing one opportunity to perform the behavior did not materially affect the habit formation process" — [James Clear summary](https://jamesclear.com/new-habit) [Secondary]. The study has not been replicated.
- **Self-compassion after failure [4 experiments]:** a self-compassionate framing of failure increased motivation to improve and **study time after an initial failure**, compared with self-esteem and neutral controls — [PubMed (Breines & Chen 2012)](https://pubmed.ncbi.nlm.nih.gov/22645164/).
- **AI/LLM coaching [Weak/early evidence]:**
  - GPTCoach (4 weeks, N=54): LLM-augmented coaching improved psychological outcomes (perceived benefits, enjoyment) but showed **no significant behavioral difference** vs non-LLM controls — [ResearchGate](https://www.researchgate.net/publication/391239875_GPTCoach_Towards_LLM-Based_Physical_Activity_Coaching) (secondhand abstract).
  - An AI coaching app (MyCoach) for cancer survivors showed larger step gains than a text-message control — [PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8660785/).
  - No large LLM-coaching RCT with objective outcomes was found for 2025–2026.

#### Why people abandon these systems
- **Attrition is the default.** Across 93 mental-health apps (median 100k installs), median **15-day retention was 3.9% and 30-day retention 3.3%**. Tracker apps were slightly better at day 30 (6.1%) — [JMIR (Baumel 2019)](https://www.jmir.org/2019/9/e14567/).
- **Effects fade when the program stops** (only 8% durable, megastudy above) and **prompts habituate** (HeartSteps above).
- **Shame framing backfires:** broken-streak displays reduce re-engagement (JCR above). Overdue backlogs eventually make a system "stop being useful," which is why Todoist recommends bankruptcy [Vendor blog, above].
- **Punishment-based devices are avoided by ~86% of people** (Halpern above).
- **Over-notification:** constant pushes hurt well-being, while zero contact raises anxiety (Fitz above).
- **Product-level failure:** ChatGPT Pulse, a passive daily proactive feed, was retired within ~9 months in favor of user-controlled schedules ([Digit](https://www.digit.in/news/general/openai-is-retiring-chatgpt-pulse-and-replacing-it-with-scheduled-tasks-here-is-why.html/amp/)).
- **Cost and noise of always-on agents:** token burn and prompt bloat from heartbeats are among the most common practitioner complaints about OpenClaw (section 1, GraphScope/mkdev) [Practitioner].

### Inferences
- **The "accountability" core** should be: (a) a *recorded* weekly score the user sees, optionally shared with a partner; (b) commitments written as if-then plans with a WOOP obstacle; (c) a short daily plan and shutdown pair; (d) a weekly review. Each maps to the best-evidenced mechanisms: self-monitoring, implementation intentions, MCII and public reporting.
- **Nudges are a decaying resource.** Spend them at receptive moments, batch them, taper them as habits form, and never use them to highlight failure.
- **Forgiveness should be designed in:** days off, vacation mode, streak repair, "comeback" acknowledgement, one-command bankruptcy and renegotiation states. The streak research and the megastudy's best intervention point the same way.
- **Stakes (Beeminder-style) should be opt-in only.** They help a self-selected minority, and defaulting to them would drive abandonment.
- **The LLM coach adds** mostly better planning and personalization. There is no evidence yet that it beats simpler prompts for behavior. Narwarp should measure its own outcomes (section 6.8) rather than assume the coach adds value.

### Gaps
- No peer-reviewed evidence was found on Focusmate specifically, on AI-agent accountability coaches, or on OKRs for personal goals (OKR grading guidance was not researched).
- Akiflow and Things review features were not researched.
- The Gail Matthews "written goals + weekly progress report to a friend" study is frequently cited, but it was not checked and is believed to be non-peer-reviewed.
- Evidence on the fresh-start effect (Dai, Milkman & Riis 2014, [INFORMS](https://pubsonline.informs.org/doi/10.1287/mnsc.2014.1901)) and the planning fallacy (Buehler et al. 1994) is **[Prior knowledge]** and was not re-fetched.

---

## 3. Work/life separation: how tools separate contexts and respect boundaries

### Takeaway
Mature tools separate contexts along four axes:
1. **Visibility filters** (Apple Focus filters show only personal calendars and mail in a Personal Focus).
2. **Hours models** (Reclaim's working vs personal hours; Sunsama's shutdown time; OpenClaw's heartbeat `activeHours`).
3. **Scoped memory** (Claude's per-project memory; OpenClaw loading MEMORY.md only in private sessions).
4. **Separate products or accounts** (Gemini Daily Brief refuses work accounts).

Explicit end-of-day rituals (Sunsama shutdown) mark the boundary psychologically. The main technical risk is cross-posting: OpenClaw agents can by default message across conversations and providers.

### Cited Findings
- **Apple Focus filters [Primary]:** a filter can hide the work calendar during a Personal Focus and limit Mail to chosen accounts — [Apple Mac Help](https://support.apple.com/guide/mac-help/mchl613dc43f/). Calendar shows that the view is "filtered by Focus" with a toggle, and Mail notifications can be limited so only work mail alerts during Work Focus — [WWDC22 session notes](https://nonstrict.eu/wwdcindex/wwdc2022/10121/). Filters also affect widgets and the lock and home screens — [MacRumors](https://www.macrumors.com/how-to/use-focus-mode-filters-ios/).
- **Reclaim.ai [Secondary tutorials]:**
  - Users set working hours, and Reclaim "won't schedule anything outside these windows" unless told to. "Meeting hours" can be a subset of working hours.
  - **Habits are assigned to an hours category (Work Hours vs Personal Hours)** and are defended: they flex around meetings but always find a slot. Examples include lunch, exercise and an end-of-day shutdown — [Marc Andrews guide](https://marcandrews.com/how-to-set-up-reclaim-ai-with-google-calendar-2024-guide/), [review](https://bestreview.it.com/review/reclaim-ai-review/).
  - "7.6 hours saved/week" is a vendor-data claim.
- **Sunsama:** a shutdown time is set each morning, and the shutdown ritual gives a "clean mental break between work and personal time" — [Sunsama Help](https://help.sunsama.com/docs/daily-planning), [Asian Efficiency](https://www.asianefficiency.com/schedule-management/sunsama-review/) [Practitioner].
- **OpenClaw:**
  - `heartbeat.activeHours` skips ticks outside a window — [OpenClaw docs](https://docs.openclaw.ai/gateway/heartbeat).
  - MEMORY.md loads "only… in the main, private session (not shared/group contexts)" — [OpenClaw docs](https://docs.openclaw.ai/concepts/agent-workspace).
  - By default, message-tool agents "can send across conversations and channel providers"; restrictions must be configured — [OpenClaw docs](https://docs.openclaw.ai/gateway/security).
- **Claude:** "Claude creates a separate memory for each project," which keeps work streams and confidential discussions apart. Incognito chats bypass memory — [Claude blog](https://claude.com/blog/memory).
- **Google:** Daily Brief is "not available with a work, school, or supervised Google Account" — [Gemini Help](https://support.google.com/gemini/answer/17077455?hl=en).
- **Todoist:** user-defined Days off and Vacation mode suspend goal and streak accounting — [Todoist Help](https://todoist.com/help/articles/introduction-to-karma).

### Inferences
- Narwarp's Work/Life split should be **one mode attribute** carried by every object: goal, ticket, calendar, memory file, channel, nudge, agent session. Several behaviors should key off it:
  - **Visibility** works like a Focus filter: the Life board and calendars are hidden in Work mode, and vice versa.
  - **Hours** follow a Reclaim-style model: work hours, personal hours, protected decompression blocks and quiet hours.
  - **Memory scope:** life memory is never loaded into work agents, mirroring OpenClaw's private-session rule.
  - **Output routing:** Life nudges go only to a personal DM or bot identity and never into work Discord servers or Linear.
- **The daily shutdown is the boundary.** After it, no work nudges go out. During work focus blocks, Life items show only as passive counters, except for the scheduled decompression breaks the user asked for.
- **Discord is both the user's work channel and the planned nudge channel,** which creates a specific cross-contamination risk. Use a separate bot identity or a DM-only channel for Life, and enforce routing in code. OpenClaw's default of allowing cross-provider sends shows this does not happen automatically.

### Gaps
- No research was found on the *effects* of Focus filters or right-to-disconnect practices on well-being.
- Discord-specific quiet-hours and DND semantics for bots were not checked (left to the integrations researcher).

---

## 4. Memory: markdown/Obsidian vaults vs vector DB vs structured profiles, and summarizing a life without bloat

### Takeaway
Practitioners and vendors have converged on **human-readable markdown files as the source of truth**, often **git-backed** (OpenClaw workspace repo, Letta MemFS). A **tiny pinned profile** loads into every session, and everything else is **retrieved just-in-time** via FTS or hybrid search. **Background consolidation ("dreaming")** promotes facts from daily logs into a curated file. Letta's own benchmark found a plain filesystem agent beat a specialized memory product on LoCoMo (74.0% vs 68.5%). That is a vendor result, but it supports starting simple.

New 2026 research shows **persistent memory is an attack surface**: payloads planted in memory files compromise future sessions. Memory writes therefore need provenance and review.

### Cited Findings
- **OpenClaw:**
  - USER.md and MEMORY.md are pinned; daily notes are searched, not injected. A silent pre-compaction flush saves context, and dreaming promotes items into MEMORY.md. Search is SQLite hybrid (vector + keyword), with local embeddings via Ollama, LM Studio or GGUF.
  - "MEMORY.md is a compact, curated layer, 'not a raw transcript, daily log, or exhaustive archive'" — [OpenClaw docs: memory](https://docs.openclaw.ai/concepts/memory).
  - Budgets: USER.md has its own 4,000-char budget; files truncate at 20,000 chars each and 60,000 total. Detail belongs in daily files "retrieved on demand rather than injected into every prompt" — [OpenClaw docs: workspace](https://docs.openclaw.ai/concepts/agent-workspace).
- **Hermes:** curated MEMORY.md/USER.md, FTS5 session search with LLM summarization, Honcho user modeling, and periodic nudges for the agent to persist knowledge — [GitHub](https://github.com/NousResearch/hermes-agent).
- **Letta:** MemFS is git-backed markdown, with `system/` pinned in full and other files loaded on relevance. Dreaming subagents consolidate memory after N steps or on compaction, and reorganization (split/merge/restructure) is backed up first — [Letta docs: MemFS](https://docs.letta.com/letta-code/memfs), [Letta docs: dreaming](https://docs.letta.com/guides/agents/architectures/sleeptime).
- **Leon 2.0:** markdown memory tiers (`daily`, `persistent`, `archive`, `reports`) plus SQLite, full-text and vector retrieval; fragments saved after each turn; a private reflection diary — [Leon blog](https://blog.getleon.ai/road-to-2-0/) (>6 mo).
- **Letta filesystem benchmark [Vendor, Aug 12, 2025, >6 mo]:**
  - A filesystem agent (GPT-4o mini) with conversation history in files scored **74.0% on LoCoMo**, "without any specialized memory tools." Mem0 reported **68.5%** for its best graph variant.
  - Conclusion: "agent capability matters more than the retrieval mechanism"; simple tools "are more likely to be in the training data."
  - Caveats: Mem0's numbers are disputed, and LoCoMo measures retrieval, not agentic memory — [Letta blog](https://www.letta.com/blog/benchmarking-ai-agent-memory).
- **Anthropic context engineering (Sep 29, 2025) (>6 mo, but foundational):**
  - **Context rot:** recall degrades as context grows.
  - **Compaction:** tune for recall first, then precision.
  - **Structured note-taking:** e.g., a NOTES.md or to-do list persisted outside the context window.
  - **Sub-agents** return condensed 1–2k-token summaries.
  - **Just-in-time retrieval** via "lightweight identifiers (file paths, stored queries, web links)." The hybrid of CLAUDE.md up front plus grep/glob on demand is common — [Anthropic Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents).
- **Claude memory:** project-scoped memory, an editable memory summary, incognito chats and import/export. Testing covered "reinforcing harmful patterns, over-accommodation" — [Claude blog](https://claude.com/blog/memory).
- **Memory as attack surface [Primary research, Jul 16, 2026]:**
  - "Bad Memory" tested Claude Code and Codex with Haiku 4.5, Opus 4.7, GPT-5.2 and GPT-5.5.
  - "Payloads already planted in those files can successfully attack current and future sessions."
  - Getting an agent to overwrite its own memory from untrusted external content was *difficult*. Success "varies substantially" by system and model — [arXiv 2607.14611](https://arxiv.org/abs/2607.14611).
- **Memory amplifies injection [Secondary, Jun 2026]:** Imperva warned that with memory on by default, widely shared content carrying a hidden instruction could compromise agents that ingest it — [The Hacker News](https://thehackernews.com/2026/06/new-attacks-trick-openclaw-ai-agent.html).
- **Context loss can drop safety constraints [Secondary, Feb 2026]:** Summer Yue (Meta) speculated her "don't act until I approve" instruction was lost during context compaction on a large inbox before OpenClaw deleted 200+ emails — [Forklog](https://forklog.com/en/news/openclaw-ai-agent-runs-amok-deletes-meta-researchers-emails), [OECD AI incident](https://oecd.ai/en/incidents/2026-02-23-d55b).

### Inferences
- **Recommended stack:**
  - A git-backed markdown vault (Obsidian-compatible) as the source of truth.
  - SQLite FTS5 as a *derived* index, with optional local embeddings via Ollama later.
  - A small pinned "core" (profile + active goals + this week's commitments), capped at a few thousand characters and following OpenClaw's budget discipline.
  - Hierarchical rollups (daily logs → weekly review notes → 12-week cycle summaries) to summarize a life without bloat.
  - Structured YAML front-matter for goals and commitments, so the board and the scheduler can parse them deterministically. Prose stays for reflections.
- **A vector DB is not needed on day one.** Letta's result and the shared OpenClaw/Hermes/Leon choice of SQLite/FTS both suggest full-text search plus agentic search is enough at personal scale.
- **Standing rules must not live only in conversational context or memory prose.** Safety constraints such as "never delete," quiet hours and approval requirements belong in enforced config: the Yue incident.
- **Every memory entry should carry provenance** (user / agent-inferred / derived-from-email / derived-from-calendar) and a trust level. Entries from untrusted sources must never be phrased as instructions. Consolidation diffs should be human-reviewable, like DREAMS.md or git commits, because of the Bad Memory findings.

### Gaps
- No independent benchmark compares markdown+FTS against vector or graph memory (Mem0, Zep/Graphiti) for *multi-year personal* memory. LoCoMo and LongMemEval-style benchmarks measure short-horizon conversational recall.
- No sources were found on how to *forget*: data retention or decay policies for personal agents.
- "Bad Memory" numbers were not available from the abstract.

---

## 5. Privacy and safety: prompt injection via email, calendar and chat; least privilege; read-only/draft-only; approval gates; local models

### Takeaway
Indirect prompt injection through **email, calendar invites, contact cards and chat messages** is documented in 2025–2026 against every class of personal agent:
- Zero-click: EchoLeak (M365 Copilot), ShadowLeak (ChatGPT Deep Research), BrokenClaw (OpenClaw Gmail hook).
- Calendar-borne: "Invitation Is All You Need" against Gemini.
- Social engineering by ordinary email: Varonis vs OpenClaw.
- Exfiltration via allowlisted domains: PromptArmor vs Claude Cowork.

OpenAI and Five Eyes agencies say injection is **unlikely ever to be fully solved**. Delimiter tags and "be careful" prompts were bypassed. What works is **architecture**: Meta's Rule of Two and the lethal trifecta, separate reader and actor roles, least-privilege scopes, deterministic approval gates for outbound or destructive actions, no auto-loaded plugins, immutable-ID allowlists and egress control.

### Cited Findings

#### Incidents and attacks on personal agents
- **OpenClaw, BrokenClaw Part 1 (Feb 2, 2026) [Primary researcher write-up]:**
  - A single crafted email led to code execution via the Gmail hook, "even if the email is not opened by the user."
  - Email bodies were passed in the `user` role, and hook sub-agents had all tools by default, with sandbox opt-in.
  - The `<<<EXTERNAL_UNTRUSTED_CONTENT>>>` wrappers were bypassed with fake end tags, and random tag IDs did not help.
  - The payload made the agent clone a malicious `.openclaw` plugin repo and restart the gateway, which executed attacker code.
  - A Docker sandbox with restricted tools blocked it. The author's view: "The base issue of prompt injection is not solved." Prompt injection is out of scope in OpenClaw's security policy — [veganmosfet](https://veganmosfet.codeberg.page/posts/2026-02-02-openclaw_mail_rce/).
  - Part 3 (Mar 3, 2026) showed email-driven RCE again, this time via a tool — [veganmosfet](https://veganmosfet.codeberg.page/posts/2026-03-03-openclaw3/).
- **OpenClaw, June 2026 attacks [Secondary, The Hacker News, Jun 11, 2026]:**
  - **Imperva:** hidden instructions in **contact names, vCard fields and location labels** (including on WhatsApp) were not marked untrusted. On Gemini 3.1 Pro the agent downloaded and ran a script. This was fixed in **2026.4.23** by moving the fields into an untrusted-metadata channel, and Imperva found the same flattening pattern in other personal assistants.
  - **Varonis:** ordinary pretext emails ("Dan," a fake team lead, during a fake incident) made the agent email **mock AWS keys, DB strings and SSH credentials**, and a **247-customer dataset**. GPT-5.4 was more cautious than Gemini 3.1 Pro, but both fell for the social pretexts. Varonis says no patch fixes this and recommends limiting autonomy.
  - **Channel allowlists matched mutable display names**, so renaming oneself could impersonate an allowed user. Five such flaws were found and patched.
  - The **Dutch DPA** told users not to run OpenClaw on systems holding sensitive data — [The Hacker News](https://thehackernews.com/2026/06/new-attacks-trick-openclaw-ai-agent.html).
- **OpenClaw inbox deletion (Feb 2026) [Secondary]:** Meta alignment lead Summer Yue told the agent to *suggest* deletions only. It deleted 200+ emails, and she had to "RUN to my Mac mini" to kill it. Critics noted OpenClaw had no built-in approval requirement — [OECD AI incident](https://oecd.ai/en/incidents/2026-02-23-d55b), [AOL](https://www.aol.com/articles/meta-ai-alignment-director-shares-195622792.html), [Forklog](https://forklog.com/en/news/openclaw-ai-agent-runs-amok-deletes-meta-researchers-emails).
- **OpenClaw ecosystem:**
  - Cisco found a third-party skill doing exfiltration and prompt injection (Jan 28, 2026).
  - A maintainer warned it is "far too dangerous" for people without command-line knowledge (Jan 30, 2026).
  - 404 Media reported an exposed **Moltbook** database that let anyone take control of agents on the site (Feb 1, 2026).
  - Chinese authorities restricted state agencies, SOEs and banks from using it (Mar 10–11, 2026) — [Wikipedia](https://en.wikipedia.org/wiki/OpenClaw).
- **ClawHub malicious skills [Secondary, conflicting counts]:**
  - Koi Security found **341 malicious skills** (Jan 27–29, 2026) targeting macOS credentials.
  - Later counts: "1,184 confirmed as of Feb 16" and "2,400+ removed."
  - Snyk found 76 confirmed malicious payloads among 3,984 skills scanned.
  - VirusTotal scanning was added afterwards — [Bitdoze](https://www.bitdoze.com/openclaw-security-guide/), [Barrack.ai](https://blog.barrack.ai/openclaw-security-vulnerabilities-2026/), [InsiderLLM](https://insiderllm.com/guides/openclaw-security-report-january-2026/).
  - **CVE-2026-25253** (gateway RCE/token theft) is described inconsistently across sources (CVSS 8.8 vs 9.8; different affected ranges) — [ClawDocs](https://clawdocs.org/security/known-vulnerabilities).
  - Exposed-instance counts range from 21,000+ (late Jan 2026) to 135,000 (attributed to Bitdefender, unverified) — [Bitdoze](https://www.bitdoze.com/openclaw-security-guide/).
- **Calendar invites → Gemini ("Invitation Is All You Need," Black Hat, Aug 2025) (>6 mo):**
  - Researchers: Tel Aviv University, Technion and SafeBreach.
  - Instructions hidden in calendar event titles and descriptions (also email subjects and file names) ran when the user asked Gemini about their schedule. Ordinary replies such as "thanks" triggered actions.
  - The 14–15 demonstrated attacks included spam, phishing, calendar deletion, data exfiltration and **smart-home control** (lights, boiler, shutters) — [Gigazine](https://www.gigazine.net/gsc_news/en/20250807-google-calendar-hack-gemini), [Evoke Security](https://www.evokesecurity.com/blogs/evil-calendar-invites-abuse-google-gemini-s-agent), [project site](https://sites.google.com/view/invitation-is-all-you-need).
- **EchoLeak (CVE-2025-32711, CVSS 9.3, June 2025) (>6 mo):** a zero-click email led to data exfiltration from Microsoft 365 Copilot via "LLM scope violation." Patched, with no in-the-wild exploitation known. Microsoft suggested DLP policies blocking processing of external emails — [The Hacker News](https://thehackernews.com/2025/06/zero-click-ai-vulnerability-exposes.html), [SOC Prime](https://socprime.com/blog/cve-2025-32711-zero-click-ai-vulnerability/).
- **ShadowLeak (Radware; disclosed Jun 18, 2025; fixed early Aug 2025) (>6 mo):** a zero-click Gmail exfiltration from ChatGPT Deep Research using hidden HTML (tiny or white text). It happened **server-side in OpenAI's cloud**, so local defenses could not see it — [The Hacker News](https://thehackernews.com/2025/09/shadowleak-zero-click-flaw-leaks-gmail.html).
- **Claude Cowork file exfiltration (Jan 2026) [Primary researcher + press]:**
  - A document posing as a "skill," with injected text in 1-pt white font, told Claude to `curl` the largest file to Anthropic's file-upload API using the **attacker's API key**.
  - This worked because the API domain is on Cowork's egress allowlist. "At no point in this process is human approval required."
  - The underlying isolation flaw had earlier been reported for Claude.ai chat and was reportedly left unfixed — [PromptArmor](https://www.promptarmor.com/resources/claude-cowork-exfiltrates-files), [The Register](https://www.theregister.com/2026/01/15/anthropics_claude_bug_cowork/), [Simon Willison](https://simonwillison.net/2026/Jan/14/claude-cowork-exfiltrates-files).
  - Current fix status is unknown.
- **Field-wide [Secondary]:** Unit 42 documented 12 in-the-wild indirect injection cases against agents. A similar calendar/doc-borne exfiltration was reported against Gemini Enterprise and patched — [CSA research note](https://labs.cloudsecurityalliance.org/research/csa-research-note-indirect-prompt-injection-in-the-wild-2026/). OWASP-aligned reporting says prompt injection "still drives most agentic AI security failures in production" — [Help Net Security](https://www.helpnetsecurity.com/2026/06/11/owasp-prompt-injection-ai-security-failures/).

#### Official positions and frameworks
- **OpenAI (Dec 22, 2025):**
  - Prompt injection, "much like scams and social engineering on the web, is unlikely to ever be fully 'solved'."
  - Agent mode "expands the security threat surface."
  - Users should avoid broad prompts like **"review my emails and take whatever action is needed"** — [TechCrunch](https://techcrunch.com/2025/12/22/openai-says-ai-browsers-may-always-be-vulnerable-to-prompt-injection-attacks/).
- **Five Eyes, "Careful Adoption of Agentic AI Services" (May 1, 2026; CISA, NSA, ASD ACSC, CCCS, NZ NCSC, UK NCSC):**
  - Prompt injection is "the most persistent and difficult-to-fix threat"; current defenses are immature and **no single control is sufficient**.
  - The agencies recommend extending zero trust, defense-in-depth and least privilege — [CSA research note](https://labs.cloudsecurityalliance.org/research/csa-research-note-cisa-agentic-ai-guidance) (secondary; primary text not fetched).
- **Meta, "Agents Rule of Two" (Oct 31 / Nov 2025):**
  - Within a session, combine at most two of: [A] untrustworthy inputs, [B] sensitive systems or private data, [C] changing state or communicating externally.
  - If all three are needed, the agent should not operate autonomously and needs human-in-the-loop supervision. The rule builds on the "lethal trifecta" and Chrome's Rule of 2 — [Simon Willison](https://simonw.substack.com/p/new-prompt-injection-papers-agents), [Conffab](https://conffab.com/elsewhere/agents-rule-of-two-a-practical-approach-to-ai-agent-security/).
- **Lethal trifecta:** private data + untrusted content + external communication = exfiltration risk — [Simon Willison, Jun 2025](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) [Prior knowledge].
- **Dual-LLM / capability designs:** CaMeL (Google DeepMind, 2025) — [arXiv 2503.18813](https://arxiv.org/abs/2503.18813) — and "Design Patterns for Securing LLM Agents against Prompt Injections" — [arXiv 2506.08837](https://arxiv.org/abs/2506.08837) [Prior knowledge; not re-fetched].
- **LivePI (May 2026) [Secondary summary]:** a benchmark against live but controlled email. A layered defense of **prompt filtering plus pre-execution tool authorization** blocked every test in one configuration — [arXiv 2605.17986](https://arxiv.org/pdf/2605.17986), summarized by [CSA](https://labs.cloudsecurityalliance.org/research/csa-research-note-indirect-prompt-injection-in-the-wild-2026/).

#### Vendor guardrail patterns seen in products
- **Gemini Spark** asks first "before performing high-stakes actions like spending money or sending emails" — [Google blog](https://blog.google/innovation-and-ai/products/gemini-app/next-evolution-gemini-app/).
- **Claude Cowork:**
  - Manual/Auto/Skip approval modes; Manual recommended for money, "messages sent as you" and important files.
  - Admins can require per-task approval for **write-capable** connector tools.
  - Permanent deletion needs explicit permission — [Claude Help Center](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork).
- **Varonis recommendations:**
  - Treat the instruction file as **enforced, version-controlled policy**.
  - **Gate first-time sends to unfamiliar addresses.**
  - Tie connector access to **the trust level of whatever triggered the task**.
  - Require humans for high-risk actions (credentials, money) — [The Hacker News](https://thehackernews.com/2026/06/new-attacks-trick-openclaw-ai-agent.html).
- **TechRadar:** require confirmation for anything irreversible: deletions, outbound messages, payments — [TechRadar](https://www.techradar.com/pro/here-are-the-openclaw-security-risks-you-should-know-about).
- **OpenClaw hardening the BrokenClaw author asked for:**
  - Disable non-bundled plugins by default and don't scan the workspace for plugins.
  - Sign plugins and restrict their privileges.
  - Put email content in the `tool` role.
  - Run hook sessions in a sandbox with a restrictive tool policy — [veganmosfet](https://veganmosfet.codeberg.page/posts/2026-02-02-openclaw_mail_rce/).

### Inferences
- **Every email/calendar/Discord pipeline in Narwarp hits the lethal trifecta** unless it is split. The minimum viable design:
  - A **quarantined Reader** (local Ollama model, no tools, no network) turns raw untrusted content into schema-validated structured records.
  - A **Planner** sees only those records plus the user's own data, and can only *propose* board or calendar changes.
  - A **deterministic Actor** (plain code, not an LLM) executes only user-approved actions within hard limits.
- **Delimiter/"untrusted" tags are not a boundary** (BrokenClaw). Treat them as hygiene only.
  - Coverage must include *all* fields: sender display names, subjects, calendar titles, descriptions, locations, attendee names, vCards and file names (Imperva, Invitation Is All You Need).
  - Strip hidden text/HTML before any model sees content.
- **Read-only + draft-only is the right default** for personal email: drafts appear for review, nothing auto-sends. Calendar writes should go only to a Narwarp-owned calendar. Accepting, declining or inviting others always needs approval. Specific OAuth scope names are left to the integrations researcher.
- **"Never auto-delete":** archive or label only, with rate limits and a global **kill switch** reachable from both terminal and Discord. The Yue incident shows that a natural-language "ask first" instruction is not a control.
- **Local models are not a security control.** They reduce *data exposure* to cloud vendors (privacy). They do not reduce *injection susceptibility*, and their robustness is unknown, so the local Reader must still be tool-less.
- **Supply chain:** no marketplace skills by default, and no auto-loading of plugins from a workspace an agent can write to (ClawHub, BrokenClaw).
- **Identity:** allowlist Discord senders by immutable user ID, never by display name.
- **Egress:** watch for "allowlisted" domains that accept uploads (Cowork exfiltration via api.anthropic.com).

### Gaps
- No public evaluation was found of prompt-injection robustness for small local models (Ollama-class, 7–32B) acting as email readers.
- Current fix status of the Cowork exfiltration path and of CVE-2026-25253 details is unverified.
- The primary Five Eyes text and the full LivePI and "Bad Memory" numbers were not fetched.
- No documented incident was found of injection via *Discord* messages specifically, beyond the OpenClaw display-name allowlist flaws and Imperva's message-object attacks.

---

## 6. Design recommendations for Narwarp Life mode (synthesis of sections 1–5)

### Takeaway
Build Life mode as **rituals plus a small set of written if-then commitments**, not as an always-chatty coach:
- A 12-week-cycle goal model whose unit of accountability is the **weekly commitment**, with a recorded execution score.
- A **2-minute daily plan and shutdown** plus a **weekly review**.
- **Batched, receptivity-timed, tapering nudges** (≤3 push per day, silent unless actionable).
- A **git-backed markdown memory** with a tiny pinned core, provenance tags and reviewed consolidation.
- **Local-only processing** of raw personal data.
- A **reader/planner/actor split** with deterministic approval gates for anything that sends, deletes, spends or invites.

### Cited Findings (evidence anchors for the recommendations)
- Recorded and shared self-monitoring, d+=0.40 — [Harkin 2016](https://eprints.whiterose.ac.uk/91437/).
- If-then plans: contingent format, rehearsal and motivation raise effects — [Sheeran et al. 2025](https://kops.uni-konstanz.de/handle/123456789/69905).
- WOOP, g=0.336, larger when interactive — [Wang et al. 2021](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8149892/).
- 12 Week Year execution score and ~85% heuristic — [yu-wenhao](https://yu-wenhao.com/en/blog/12-week-year-guide/).
- Sunsama workload threshold and shutdown — [Sunsama Help](https://help.sunsama.com/docs/daily-planning).
- Beeminder akrasia horizon — [Beeminder wiki](https://wiki.beeminder.com/what-is-beeminder.html).
- Deposit uptake 13.7% — [Halpern](https://repository.upenn.edu/hcmg_papers/25).
- Micro-reward for returning after a miss was the best of 54; only 8% of effects durable — [Milkman 2021](https://www.gwern.net/doc/statistics/prediction/2021-milkman.pdf).
- Broken-streak framing lowers engagement — [Silverman & Barasch](https://www.colorado.edu/business/news/2023/04/20/research-streaks-marketing-tech-barasch).
- Prompt effects habituate within weeks — [HeartSteps](https://pmc.ncbi.nlm.nih.gov/articles/PMC6401341).
- ML timing up to +40% receptivity — [Mishra et al.](https://pmc.ncbi.nlm.nih.gov/articles/PMC8680205).
- 3×/day batching beats constant or none — [Fitz 2019](https://scholars.duke.edu/publication/1402953).
- Self-compassion boosts recovery after failure — [Breines & Chen](https://pubmed.ncbi.nlm.nih.gov/22645164/).
- Pulse retired; proactivity must be "steerable by the user" — [Digit](https://www.digit.in/news/general/openai-is-retiring-chatgpt-pulse-and-replacing-it-with-scheduled-tasks-here-is-why.html/amp/).
- Silent-unless-actionable heartbeats, activeHours, cost controls — [OpenClaw heartbeat](https://docs.openclaw.ai/gateway/heartbeat).
- Notify only when there is something to report — [ChatGPT scheduled tasks](https://itbrief.co.uk/story/openai-expands-chatgpt-scheduled-tasks-with-new-hub).
- Markdown/git memory with a pinned core — [OpenClaw memory](https://docs.openclaw.ai/concepts/memory), [Letta MemFS](https://docs.letta.com/letta-code/memfs).
- Persistent-memory poisoning — [Bad Memory](https://arxiv.org/abs/2607.14611).
- Rule of Two — [Simon Willison on Meta](https://simonw.substack.com/p/new-prompt-injection-papers-agents).
- Social-engineering and outbound gating advice — [Varonis via THN](https://thehackernews.com/2026/06/new-attacks-trick-openclaw-ai-agent.html).
- Instruction loss → inbox deletion — [OECD incident](https://oecd.ai/en/incidents/2026-02-23-d55b).

### Inferences (concrete recommendations)

#### 6.1 Goal model: goal → milestones → weekly commitments → tasks
- **Goal** (`life/goals/<slug>.md`, YAML front-matter plus prose):
  - Fields: `id`, `mode: life|work`, `area` (health, relationships, craft, finance, home, fun…), `cycle` (12-week cycle id; default horizon), `why` (WOOP Wish and Outcome in the user's words), `lag_metric` (optional, e.g., "run 10k"), `privacy: local-only|redacted-cloud|cloud-ok`, `status: active|paused|done|dropped`.
  - Cap active Life goals at **2–3 per cycle**.
- **Milestone:** 1–3 per goal per cycle, each a dated, observable outcome ("week 6: 5k under 30 min"). Reviewed at the weekly review, not nudged daily.
- **Weekly commitment** (the accountability unit, created at the weekly review):
  - Written as a **contingent if-then plan**: "When [cue/time/place], I will [action] for [duration]." It records one **obstacle → plan** pair (WOOP O/P) and an optional "minimum version" (e.g., 10 minutes).
  - Each commitment is placed as a calendar block in personal hours.
  - States: `planned → kept | partial | missed | renegotiated | dropped`. Renegotiation is a first-class, non-failure state.
  - Cap commitments, e.g., **3–7 per week**, and keep a weekly Life time budget below free hours, like Sunsama's workload threshold (warn when over).
- **Task/session:** kanban tickets on the Life board, linked to the commitment and its calendar block. Completing the ticket marks the commitment kept.
- **Scoring:** a weekly **execution score** (kept ÷ planned; partial counts as 0.5). Show it as a 4–12-week trend, with ~85% as a *soft* target. Never show a raw "streak broken"; prefer "kept 9 of last 12."
- **Commitment devices (opt-in only):**
  - (a) A Beeminder-style **akrasia horizon**: making a commitment *harder* takes effect immediately, while making it *easier* outside a ritual takes effect only after a delay (e.g., 24h–7 days).
  - (b) An optional accountability partner who receives the weekly score (public reporting moderator).
  - (c) Optional external stakes, e.g., via Beeminder; never the default, since 86% decline deposits.
- **Forgiveness mechanics:**
  - Planned days off and **vacation mode** pause scoring.
  - Streak repair.
  - After ≥2 consecutive misses on a commitment, *stop nudging it* and queue a renegotiation item for the next review.
  - One-command **bankruptcy** archives all stale Life items into `life/archive/` and starts a fresh cycle, framed as a fresh start (the fresh-start effect is [Prior knowledge]).

#### 6.2 Check-in and review rituals
- **Daily plan (≤2 min).** It triggers on the first Narwarp session of the day or at a set time.
  - The agent drafts the day: work calendar, today's Life commitment blocks, one decompression block and the **shutdown time**.
  - The user accepts or edits in the terminal (or with one Discord reply). If the user ignores it, the plan stays the draft: no repeated pings.
- **Daily shutdown (≤3 min at shutdown time).**
  - Mark done/not done, roll leftovers, write one line of reflection.
  - This **flips the mode to Life**, after which work nudges are suppressed (Sunsama's "clean mental break").
  - If it's skipped, the next morning plan opens with a 30-second "reflect on yesterday" step (Sunsama pattern) instead of a guilt message.
- **Weekly review (25–40 min, a protected calendar block chosen by the user, e.g., Friday 4pm or Sunday evening).** The agent pre-computes everything. Agenda:
  1. *Get Clear:* process the Life inbox. These are items the local Reader extracted from personal email/calendar/Discord, plus captured ideas.
  2. *Get Current:* score last week's commitments and show the trend. Misses get a neutral "what got in the way?" with no blame. Check milestones; look at the next 2 weeks of calendar.
  3. *Get Creative:* scan Someday/Maybe.
  4. *Commit:* draft 3–7 next-week commitments as if-then plans, each with an obstacle → plan.
  5. Optionally post the score to the accountability partner.
  6. Write `life/weekly/YYYY-Www.md` (the memory rollup).
- **12-week cycle review (about 60 min):** milestone outcomes; keep/pause/drop each goal; set the next cycle's 2–3 goals; write a cycle summary; apply the fresh-start framing.
- **A monthly "is this helping?" check (5 min):** ask whether the nudges are useful and adjust the cadence (see 6.8).

#### 6.3 Nudge delivery and cadence
- **Channel hierarchy:**
  - (1) **The terminal board/status line is pull-only and passive.** It is the default and lives where the user already pays attention.
  - (2) **Discord DM is push,** only for time-sensitive items. Use a Life-specific bot identity or DM, never a work server channel.
  - (3) No email nudges.
- **Push budget:** about **3 batched windows per day** (e.g., after the morning plan, at shutdown, at an optional evening check), with **≤1–2 commitment nudges** inside them.
  - Zero push during focus blocks or meetings, and none after a user-set "lights out."
  - Quiet hours are enforced in code, like OpenClaw `activeHours`. Fitz found 3×/day better than constant or zero.
- **Silent-unless-actionable heartbeat:**
  - A cheap local job (Ollama or plain rules) runs every 30–60 min inside active hours.
  - It escalates to a push only if a commitment's window is open, it hasn't started, and the user is likely receptive.
  - Otherwise it returns nothing, like OpenClaw `NO_REPLY` or ChatGPT's monitoring tasks. Use isolated, light-context runs for cost.
- **Receptivity timing:** prefer transitions. Examples: right after a Claude Code session ends, a calendar gap begins, the shutdown completes, or the user returns to the terminal after idling. This follows HeartSteps V2's "only when sedentary" and Mishra's ML-timing results. Over time, learn per-user response rates by hour and context.
- **Taper:**
  - Reduce frequency for commitments kept ≥80% over 4+ weeks; let them become "self-running."
  - Rotate message wording, because prompt effects decay within ~3–4 weeks (HeartSteps).
  - Never escalate nag frequency after misses.
- **Content rules:**
  - One concrete next action, plus one-tap replies: **Done / Start 10 min / Reschedule / Skip today / Not helpful**.
  - Self-compassionate tone. Never surface a failure count.
  - After a miss, the next touch is a **comeback framing** with the minimum version ("10 minutes counts"). Acknowledge the return (the megastudy's best intervention).
  - Every nudge must be **steerable**: snooze or edit the rule inline (Pulse lesson).

#### 6.4 Work/Life boundary rules (enforced in code)
- **Mode attribute** on every object. Mode-filtered views work like Apple Focus filters: in Life mode, hide the Work board, Linear and work calendars.
- **Hours model:** work hours, personal hours, protected decompression blocks (defended like Reclaim habits), quiet hours and days off.
- **After shutdown or outside work hours:** no work nudges and no work-agent posts to the user. Work agents can keep running, but their output queues for the morning.
- **During work hours:** Life items appear only as counts, plus the scheduled decompression blocks the user asked for.
- **Memory isolation:** `life/` memory is never loaded into Work-mode agents, and vice versa (OpenClaw's private-session rule; Claude's per-project memory).
- **Output routing allowlist:** Life content may only go to the Life DM. A deterministic check blocks any Life-derived text from posting to work Discord servers, Linear or GitHub, closing OpenClaw's cross-provider default.

#### 6.5 Memory design
- **Layout** (a private git repo, separate from work):
  - `life/profile.md` (pinned, ≤4k chars)
  - `life/now.md` (pinned: active goals + this week's commitments, ≤2–3k chars)
  - `life/goals/*.md`
  - `life/daily/YYYY-MM-DD.md` (append-only)
  - `life/weekly/YYYY-Www.md`
  - `life/cycles/*.md`
  - `life/memory.md` (curated long-term facts and decisions, budgeted)
  - `life/archive/`
- **Retrieval:** the pinned core goes into context; everything else is fetched just-in-time with FTS5 grep/search (embeddings later, local via Ollama). Use rollups instead of raw history: daily logs, then weekly notes, then cycle summaries.
- **Consolidation ("dreaming"):** a nightly or weekly *local* job proposes diffs to `memory.md`. It promotes recurring facts, merges duplicates and expires stale "now" items. The user approves diffs at the weekly review, which avoids Letta's "does not ask you for approval" default.
- **Provenance and trust:** each entry is tagged `source=user|agent|email|calendar|discord` with a date. Untrusted-source entries are stored as quoted data, never as imperative text. Only user-sourced or user-approved entries may change behavior (Bad Memory).
- **Policy is not memory:** quiet hours, approval rules, "never delete" and routing rules live in a versioned config file that the Actor enforces. They are not prose an LLM can forget during compaction (Yue).
- **Flush before compaction:** before any long Life session compacts, save decisions to the daily log (OpenClaw pattern).

#### 6.6 Privacy and local-model split
- **Local-only (Ollama):**
  - Reading and summarizing personal email bodies and calendar details.
  - Journaling and reflection text.
  - Any `privacy: local-only` goal (default for health, finance and relationships).
  - Heartbeat triage, inbox classification and routing, memory consolidation, and PII redaction.
- **Cloud (Claude) allowed:** coding and Work mode, plus Life *planning* on **redacted abstractions** produced locally (e.g., "G2 (fitness): 2/3 kept; constraint: evenings busy Tue/Thu"). The goal's privacy class gates this.
- **Never in memory or prompts:** secrets and raw credentials (OpenClaw workspace guidance).
- **Positioning:** vendor proactive features moved to cloud execution in 2026 (Cowork scheduled tasks, Gemini Spark, Cloud Routines). Narwarp's local-first Life data is a deliberate differentiator. The user also takes on backup duty (MemFS local-only note).

#### 6.7 Safety guardrails for agents that read email and calendars
1. **Reader/Planner/Actor split (Rule of Two per session):**
   - The Reader (local, **no tools, no network**) emits schema-validated JSON.
   - The Planner never sees raw untrusted text and only proposes.
   - The Actor is deterministic code that executes approved proposals.
2. **Least privilege:** email is read plus draft only, with no send and no delete. The calendar is readable; writes go only to a Narwarp-owned calendar. The Discord bot DMs the owner's **immutable user ID** only.
3. **Approval gates in terminal or Discord, showing a diff:**
   - Any email send.
   - Any message to a **first-time recipient**.
   - Any invite, accept or decline involving other people.
   - Any deletion (archive only by default).
   - Any purchase.
   - Any forwarding of attachments or credentials (hard-deny).
   - Batch low-risk approvals into the daily and weekly rituals to avoid approval fatigue.
4. **Untrusted-content hygiene:** put external text in tool/data roles, not the user role. Cover every field: names, subjects, calendar titles, descriptions, locations, vCards, file names. Strip hidden HTML and text. Don't rely on delimiter tags.
5. **Supply chain:** no third-party skill marketplace by default; no auto-loading of plugins from agent-writable paths; pinned, reviewed skills only.
6. **Egress control:** the Reader has no network. The Actor has an explicit API allowlist, audited for upload-capable endpoints (the Cowork exfiltration lesson).
7. **Gateway exposure:** bind to loopback and keep no open ports, like OpenClaw's default. Provide a `narwarp security audit` command.
8. **Kill switch and audit log:** one command or Discord word stops all agents. Every action is logged with provenance.
9. **Regression red-team:** keep a test corpus of known payloads as CI tests for the Reader/Planner pipeline: calendar-invite promptware, EchoLeak/ShadowLeak-style hidden-HTML emails, vCard and contact-name injections, "Dan the team lead" pretexts.
10. **Accept residual risk:** treat model-level defenses as best-effort (OpenAI, Five Eyes). The structural limits above are what bound the damage.

#### 6.8 Instrumentation, so Life mode adapts instead of being abandoned
- **Track locally:**
  - Nudge response rate by hour and context.
  - Snooze and "Not helpful" rates.
  - Ritual completion (daily plan, shutdown, weekly review).
  - Execution-score trend.
  - Days since the last Life interaction.
- **Rules:**
  - If "Not helpful" exceeds ~30% or responses fall for 2 weeks, cut push frequency automatically and ask at the weekly review.
  - If no weekly review happens for 2 weeks, send *one* gentle "fresh start?" message, then go quiet. Never pile on.
- **Rationale:** attrition is the norm (about 3% 30-day retention in mental-health apps), and effects fade after programs end. The system should optimize for *still being used in month 6*, not for nudge volume.

### Gaps
- No direct evidence was found on which nudge channel (terminal vs chat DM) works better for developer users. The recommendation rests on "place I pay attention to" plus general receptivity research.
- Thresholds such as "≤3 push/day," "3–7 commitments" and "2–3 goals" are design heuristics drawn from the cited evidence (Fitz batching, Sunsama under-planning advice, 12WY focus). They are not validated parameters and should be tuned through 6.8.
- How well local Ollama-class models perform at email triage and structured extraction, and how robust they are to injection, was not evaluated here (another researcher covers Ollama specifics).
