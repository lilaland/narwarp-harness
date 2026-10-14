# App integrations for a single-user, local-first macOS harness (Linear, Discord, personal email, GitHub), as of 2026-10-13

Scope note: Google Calendar is out of scope (the calendar researcher covers it). Kanban/ticket models, message-board design, Claude Code/Ollama internals and TUI build-vs-fork are covered by others. This file is about the external integrations themselves and how a laptop app should connect to them.

Staleness convention: all vendor doc pages below were fetched on 2026-10-13 unless a date is given. Items dated before ~2026-04-13 are flagged **[>6 mo]**: they may be stale. Third-party, vendor-marketing and practitioner sources are labeled.

---

## 1. Linear: API, SDKs, auth, webhooks, rate limits, the official MCP server, and "Linear for Agents"

### Takeaway
Linear is GraphQL-only, with an official TypeScript SDK and no first-party Python SDK. A personal API key (scoped and team-restricted) is enough to read and write, but **only workspace admins can create webhooks**, and installing Narwarp as a first-class Linear *agent* (actor=app, delegation, Agent Sessions) also needs an admin. So in a work workspace the user doesn't administer, Narwarp should plan on **polling within generous rate limits** (2,500 req/h and 3M complexity points/h per API key), and should treat agent mode as an opt-in upgrade if the admin cooperates. The official MCP server (`https://mcp.linear.app/mcp`, with a `/mcp/readonly` variant) works well for Claude Code sessions. Narwarp's own sync should stay on direct GraphQL.

### Cited Findings

**API and SDKs**
- Linear's public GraphQL schema is published in the `linear/linear` monorepo at `packages/sdk/src/schema.graphql`. The copy fetched on 2026-10-13 has 52,907 lines, and it is the authoritative source for the field names cited in this file — [schema.graphql](https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql)
- The official client is the TypeScript `@linear/sdk` — [npm @linear/sdk](https://www.npmjs.com/package/@linear/sdk). Search snippets disagree on the current version (an npm snippet says 33.0.0, a registry mirror lists 59.1.0), so check npm directly — [tessl registry](https://tessl.io/registry/tessl/npm-linear--sdk/59.1.0/files)
- There is no first-party Python SDK. The community `linear-api` package on PyPI is a Pydantic wrapper with a raw `execute_graphql` method. Its docs mirror says it was "archived in June 2026", so its maintenance status is unclear — [tessl: pypi-linear-api](https://tessl.io/registry/tessl/pypi-linear-api/files/docs/client-management.md) (third-party)
- A third-party review (March 2026) describes the API as "GraphQL-only, no REST fallback". The same review says the limit is "10,000 points per hour", which conflicts with Linear's docs: 10,000 is the *per-query* maximum (see below) — [rhumb.dev](https://rhumb.dev/service/linear) (third-party; contradicted by [Linear rate limiting](https://linear.app/developers/rate-limiting))

**Auth**
- Personal API keys: "Admins and permitted Members can create personal API keys". Admins can switch off member API keys (Settings > Administration > API > Member API keys). Keys can be limited to "Read, Write, Admin, Create issues, Create comments" and to specific teams — [Linear docs: API and webhooks](https://linear.app/docs/api-and-webhooks)
- OAuth2 parameters: `actor=user` (the default) or `actor=app`. Scopes are `read`, `write`, `issues:create`, `comments:create`, `timeSchedule:write` and `admin`. On `admin`, the docs say: "You should never ask for this permission unless it's absolutely needed" — [Linear OAuth 2.0](https://linear.app/developers/oauth-2-0-authentication)
- PKCE is supported (`plain` or `S256`). With PKCE, `client_secret` becomes optional at the token endpoint, and the docs' examples use `http://localhost:3000/oauth/callback` — [Linear OAuth 2.0](https://linear.app/developers/oauth-2-0-authentication)
- "The access token is valid for 24 hours and will need to be refreshed". Refreshing returns a new refresh token, with a 30-minute grace period for replaying a request. "All OAuth2 applications were migrated to the new refresh token system on April 1, 2026." — [Linear OAuth 2.0](https://linear.app/developers/oauth-2-0-authentication)
- Client-credentials grant: it must be toggled on per app. It issues `app`-actor tokens with access to all public teams, valid for 30 days, with no refresh token. Rotating the client secret invalidates them — [Linear OAuth 2.0](https://linear.app/developers/oauth-2-0-authentication)
- Tokens are revoked with `POST https://api.linear.app/oauth/revoke` — [Linear OAuth 2.0](https://linear.app/developers/oauth-2-0-authentication)

**Rate limits**

| Auth | Requests/hour | Complexity points/hour | Scope |
|---|---|---|---|
| API key | 2,500 | 3,000,000 | per user |
| OAuth app | 5,000 | 2,000,000 | per user/app user |
| Unauthenticated | 600 | 100,000 | per IP |

- The table above is from [Linear rate limiting](https://linear.app/developers/rate-limiting). The maximum complexity for a single query is 10,000 points. Each property costs 0.1 point and each object 1 point. A connection multiplies its children by its `first` argument, or by 50 if none is given. When a limit is hit, the API returns HTTP **400** with `extensions.code = RATELIMITED`, and `X-RateLimit-*` / `X-Complexity` headers report usage — [Linear rate limiting](https://linear.app/developers/rate-limiting)
- Linear's own guidance: "One thing that we especially discourage is polling the API to fetch updates"; it recommends webhooks instead — [Linear rate limiting](https://linear.app/developers/rate-limiting)
- Workspace-level OAuth apps using actor authorization can get higher, dynamic limits — [Linear rate limiting](https://linear.app/developers/rate-limiting)

**Webhooks**
- Resource types: Issues, Issue attachments, Issue comments, Issue labels, Comment reactions, Projects, Project updates, Documents, Initiatives, Initiative updates, Cycles, Customers, Customer requests and Users. There are also convenience events for Issue SLA and OAuthApp revoked — [Linear webhooks](https://linear.app/developers/webhooks)
- "Only workspace admins, or OAuth applications with the `admin` scope, can create or read webhooks." A webhook covers either all public teams (`allPublicTeams`) or one `teamId` — [Linear webhooks](https://linear.app/developers/webhooks); also [Linear docs: API and webhooks](https://linear.app/docs/api-and-webhooks)
- Payload fields are `action` (create/update/remove), `type`, `actor`, `data`, `url`, `createdAt`, `updatedFrom` (previous values) and `webhookTimestamp` — [Linear webhooks](https://linear.app/developers/webhooks)
- Security and delivery:
  - The `Linear-Signature` header is a hex HMAC-SHA256 of the raw body. Linear recommends rejecting requests whose `webhookTimestamp` is more than about 60 s old.
  - Your endpoint must return HTTP 200 within 5 s.
  - Failed deliveries are retried 3 times, after 1 min, 1 h and 6 h, and the webhook may then be disabled.
  - The endpoint must be a "publicly accessible HTTPS, non-localhost URL".
  — [Linear webhooks](https://linear.app/developers/webhooks)

**Official MCP server**
- The Linear MCP launched 2025-05-01 **[>6 mo]** — [Linear changelog](https://linear.app/changelog/2025-05-01-mcp)
- Endpoints: `https://mcp.linear.app/mcp` (Streamable HTTP, the primary one) and `https://mcp.linear.app/mcp/readonly`, which "only ever exposes read tools". `/sse` is a deprecated fallback — [Linear MCP docs](https://linear.app/docs/mcp)
- Auth is OAuth 2.1 with dynamic client registration. Alternatively, send `Authorization: Bearer <token>` with an API key (a restricted key works) or an OAuth token. That lets you act as an `app` user — [Linear MCP docs](https://linear.app/docs/mcp)
- Three ways to make it read-only: use the `/mcp/readonly` endpoint, request only the `read` OAuth scope, or use a Read-only API key — [Linear MCP docs](https://linear.app/docs/mcp)
- Claude Code setup is `claude mcp add --transport http linear-server https://mcp.linear.app/mcp`, then `/mcp` to authenticate — [Linear MCP docs](https://linear.app/docs/mcp)
- Known gotcha: "Each workspace needs its own auth context", and reconnecting does not switch workspaces (for `mcp-remote`, use `MCP_REMOTE_CONFIG_DIR`). This matters if the user has both a personal and a work Linear workspace — [Linear MCP docs](https://linear.app/docs/mcp)
- The docs do not list individual tools. Third-party write-ups say `save_issue` merges create and update, and they disagree on the tool count (22 vs "25+"). Both counts look outdated — [mcpbundles blog](https://docs.mcpbundles.com/blog/linear-mcp-server), [La Growth Machine](https://lagrowthmachine.com/linear-mcp/) (third-party)

**Linear for Agents**

*Launch and status*
- Linear for Agents launched 2025-05-20 **[>6 mo]** — [changelog](https://linear.app/changelog/2025-05-20-linear-for-agents)
- The Agent Interaction SDK and guidelines entered developer preview on 2025-07-30 **[>6 mo]** — [changelog](https://linear.app/changelog/2025-07-30-agent-interaction-guidelines-and-sdk)
- The Agent APIs are still labeled "Developer Preview ... Functionality and Agent APIs may change before general availability" — [Linear developers: agents](https://linear.app/developers/agents)

*Install and identity*
- Agents install via OAuth with `actor=app`, plus the optional scopes `app:assignable` ("be assigned as a delegate on issues and made a member of projects") and `app:mentionable`. "Workspace-scope installs require admin permissions." `actor=app` apps cannot request `admin`. Installed agents are not billable users — [Linear developers: agents](https://linear.app/developers/agents)
- Delegation semantics: assigning an issue to an agent sets the `delegate`, not the `assignee`, so "the human teammate remains responsible" — [Linear developers: agents](https://linear.app/developers/agents); [Linear docs: AI agents](https://linear.app/docs/agents-in-linear)

*Sessions and webhooks*
- An AgentSession is created automatically when the agent is @mentioned or delegated an issue. Agents can also open sessions themselves with `agentSessionCreateOnIssue` / `agentSessionCreateOnComment` — [agent interaction](https://linear.app/developers/agent-interaction)
- Webhooks are `AgentSessionEvent` with `created` and `prompted` events. The receiver must answer within 5 s, and the agent must post an activity or external URL within 10 s of `created`, or the session is marked unresponsive. Enabling this webhook category shows Agent Session UI to the workspace immediately — [agent interaction](https://linear.app/developers/agent-interaction)

*Activities, context and plans*
- Activity types: `thought`, `elicitation`, `action` (tool call with optional result), `response` and `error`. `prompt` is user-only, and only `thought` and `action` can be ephemeral — [agent interaction](https://linear.app/developers/agent-interaction)
- Session states are set automatically: `pending`, `active`, `error`, `awaitingInput`, `complete`, `stale` — [agent interaction](https://linear.app/developers/agent-interaction)
- `promptContext` is a pre-formatted context string. Structured fields are `issue`, `comment`, `previousComments` and `guidance`, and guidance can be set at the workspace, parent-team or team level — [agent interaction](https://linear.app/developers/agent-interaction)
- Agent "plans" are in technology preview: a list of `content`/`status` items, and every update replaces the whole plan — [agent interaction](https://linear.app/developers/agent-interaction)

*What the schema shows (master, 2026-10-13)*
- `AgentSessionStatus` adds `stopping` (`active awaitingInput complete error pending stale stopping`), and `AgentActivitySignal` is `auth continue select stop` — [schema.graphql](https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql)
- `AgentSession` has `externalLinks`, `pullRequests`, `codingHarness`, `plan` and `modelSelection`. Its `externalUrls` field is marked `@deprecated(reason: "Use externalLinks instead.")`, which **conflicts** with the agent-interaction page: that page deprecates `externalLink` *in favor of* `externalUrls`. The schema also has `agentSessionSandbox` / `agentSessionSshAddress` queries and an "[Internal] first-class code diff" model — [schema.graphql](https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql) vs [agent interaction](https://linear.app/developers/agent-interaction)
- A third-party release tracker reports a built-in "Linear Agent" AI assistant in March 2026 — [releases.sh](https://releases.sh/linear/linear/highlights) (third-party; dates on that page are internally inconsistent)
- Linear's sample agent is the Weather Bot, built on the TypeScript SDK and Cloudflare — [linear/weather-bot](https://github.com/linear/weather-bot) via [Linear developers: agents](https://linear.app/developers/agents)

### Inferences
- **Two operating modes for Narwarp against a work Linear workspace:**
  - **Mode A, no admin help (the default).** Use a personal API key restricted to *Create issues + Create comments + Read* and to the user's teams. Poll for changes (60 s while active, 5 min idle). Everything Narwarp writes then appears as authored by the user.
  - **Mode B, admin installs a "Narwarp" OAuth app with `actor=app`.** Narwarp gets its own identity in Linear, can be delegated issues and @mentioned, and can stream AgentActivities from Claude Code/Ollama work into Linear. It can also receive webhooks through a tunnel. Mode B is the only way to separate "Narwarp/agent wrote this" from "the user wrote this" inside Linear: `createAsUser` and `displayIconUrl` on attachments and comments are actor=app-only ([schema.graphql](https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql)).
- Polling cost, worked from Linear's formula: one poll fetching 50 issues with about 10 scalar fields each, plus `labels(first:5)` and `attachments(first:5)` at 2 fields each, costs roughly 50 × (2 + 6 + 6) ≈ 700 points. At 60 polls/h that is about 42k points/h (≈1.4% of 3M) and 60 requests/h (2.4% of 2,500). Polling is comfortably within limits, even though Linear discourages it.
- Language choice: for Rust in Narwarp, generate typed queries from the published schema. The fork already has a `graphql` infrastructure crate, per `AGENTS.md`. For a Python sidecar, `httpx` plus a schema-driven codegen such as ariadne-codegen avoids depending on the shaky community SDK. Neither has been benchmarked.
- Observation from this research environment, not a vendor doc: the Linear MCP connector registered in this session advertises about 95 tools. It failed to connect, so none were exercised. Names include:
  - core objects: `list_issues`, `get_issue`, `save_issue`, `list_comments`, `save_comment`
  - attachments: `create_attachment`, `get_attachment`, `delete_attachment`, `prepare_attachment_upload`, `create_attachment_from_upload`
  - planning: `list_projects`/`save_project`, `list_initiatives`/`save_initiative`, `list_cycles`, `list_milestones`/`save_milestone`, `list_custom_views`
  - labels and docs: `list_issue_labels`/`save_issue_label`, `list_documents`/`save_document`, `get_status_updates`/`save_status_update`
  - inbox and sharing: `get_notifications`/`mark_notification`, `share_issue`/`unshare_issue`
  - newer areas: `list_releases`/`save_release`, `list_diffs`/`get_diff`/`merge_diff`/`submit_diff_review`, `list_agent_skills`/`get_agent_skill`, `search_documentation`

  So the official MCP now reaches well beyond the 22–25 tools that third-party blogs describe, including attachments, notifications, initiatives, releases and diffs.
- Use the MCP for *interactive* Claude Code work, preferably `/mcp/readonly` or a Read-only key. Keep Narwarp's *sync* on direct GraphQL. Sync needs cursors, idempotency (attachment URLs), batching (`issueBatchUpdate`, up to 50 IDs), and a write journal, none of which an LLM-driven MCP call loop provides.

### Gaps
- No Linear doc found on whether a non-admin *member* can authorize a third-party OAuth app with `actor=user`, or whether admins can block that. The docs only say OAuth apps are managed in admin-only settings.
- I couldn't confirm whether OAuth-app-level webhooks (configured on the app rather than via `webhookCreate`) deliver events for a workspace where a non-admin authorized the app.
- Linear documents no official tool list for its MCP server, and no per-tool rate limits. The roughly 95-tool list above comes from this environment's connector registry only.
- No GA date for the Agent APIs. The `externalUrls` vs `externalLinks` deprecation conflict is unresolved.

---

## 2. Linear: the issue model, and how one Narwarp ticket should aggregate many Linear issues (plus two-way sync pitfalls)

### Takeaway
In Linear, an issue belongs to **exactly one team, at most one project, at most one parent, and at most one cycle**, but it can carry **many labels and many attachments**. Attachments are upserted by `(issueId, url)` and can be queried by URL. So the cleanest many-to-many-safe pattern for "one Narwarp ticket aggregates N Linear issues" is: **put one attachment on each Linear issue pointing at the Narwarp ticket's stable URL, and keep a local mapping table as the source of truth.** An optional personal Custom View filtered on that attachment URL shows the group inside Linear. Projects, parent issues and labels are shared team structure: use them only when the Narwarp ticket genuinely *is* a team-level project or epic.

### Cited Findings

**Issue shape** — [schema.graphql](https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql)
- One team: `team: Team!`
- At most one project: `project: Project`, "Null if the issue is not part of any project"
- At most one parent: `parent: Issue`, with `children` for sub-issues
- At most one cycle and one milestone: `cycle`, `projectMilestone`
- Many labels and attachments: `labels(...)` and `attachments(...)` connections
- Relations: `relations` / `inverseRelations`
- Ownership: `delegate: User` ("Set when an AI agent has been assigned") alongside `assignee`
- `syncedWith: [ExternalEntityInfo!]`, where `ExternalSyncService` is `github | jira | slack`
- `sharedAccess: IssueSharedAccess`, for issue sharing with field restrictions

**Relations and workflow states**
- The relation types are `blocks`, `duplicate`, `related` and `similar` (`enum IssueRelationType`) — [schema.graphql](https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql)
- `WorkflowState.type` is one of "triage", "backlog", "unstarted", "started", "completed", "canceled", "duplicate", and states are per team (`team: Team!`) — [schema.graphql](https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql)

**Projects, initiatives and links** — [schema.graphql](https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql)
- Projects can span several teams (`teams(...)`) and belong to several initiatives (`initiatives(...)`). They also have `projectMilestones`, `externalLinks`, `documents`, `labels`, `members`, `lead` and `status`.
- Initiatives nest (`parentInitiative`, `parentInitiatives`, `subInitiatives`) and contain `projects` and `links`.
- `EntityExternalLinkCreateInput { url!, label!, projectId | initiativeId | cycleId | teamId | releaseId | resourceFolderId }` attaches "resource" links to projects, initiatives and similar objects.

**Labels and custom views** — [schema.graphql](https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql)
- A label group "acts as a container for child labels and cannot be directly applied". `IssueLabel.team` is nullable, which means workspace-level labels exist. Labels can be retired (`retiredAt`).
- `CustomView` stores `filterData`, `projectFilterData` and `initiativeFilterData`, plus `shared`, `owner` and an optional `team`. In other words, a custom view is a saved filter, private or shared.

**Filtering issues by attachment** — [schema.graphql](https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql)
- `IssueFilter` includes `attachments: AttachmentCollectionFilter`. That filter has `some`/`every` (each an `AttachmentFilter`) and `length`, plus direct `url: StringComparator`, `title`, `subtitle`, `sourceType`, `creator` and timestamps. So `issues(filter:{attachments:{some:{url:{eq:"…"}}}})` is the unambiguous form.
- It also has `delegate`, `parent`, `children`, `project`, `labels`, `hasBlockedByRelations`/`hasBlockingRelations`/`hasRelatedRelations`/`hasDuplicateRelations`, `searchableContent` and `updatedAt`.

**Attachments**
- `AttachmentCreateInput` takes these fields — [schema.graphql](https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql):
  - `url!`: "used also as an unique identifier for the attachment. If another attachment is created with the same `url` value, existing record is updated instead"
  - `issueId!`, which accepts a UUID or an identifier like 'LIN-123'
  - `title!`, `subtitle`, `metadata` (string and number values)
  - `iconUrl`: jpg/png, 1 MB max, 20×20 px recommended
  - `groupBySource`, to group attachments from the same source app in the UI
  - `commentBody`, to also create a linked comment
  - `createAsUser` / `displayIconUrl`, actor=app only
- Mutations and queries:
  - `attachmentLinkURL(issueId, url, title)` is a lightweight link mutation, alongside typed variants (`attachmentLinkGitHubPR`, `attachmentLinkSlack`, `attachmentLinkDiscord`, and others).
  - `attachmentsForURL(url)` returns the attachments for a URL together with their issues. The older issue-by-attachment lookup is deprecated in favor of it.
  — [schema.graphql](https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql)
- Linear's docs: "Attachment URL is used as an idempotent value if used in conjunction with the same issue id", so scripts don't need to store attachment IDs — [Linear attachments](https://linear.app/developers/attachments)
- Rich metadata keys are `title`, `messages[]` (subject/body/timestamp, with bodies kept under 10k characters) and `attributes[]`. Subtitle templates `{var__since}` and `{var__relativeTimestamp}` render relative times. Metadata is "only exposed through the API" — [Linear attachments](https://linear.app/developers/attachments)
- Webhooks can notify about new and updated attachments ("Issue attachments" resource type) — [Linear attachments](https://linear.app/developers/attachments); [Linear webhooks](https://linear.app/developers/webhooks)
- Batch mutations: `issueBatchCreate` "creates a list of issues atomically". `issueBatchUpdate` takes IDs that "Can't be more than 50 at a time" — [schema.graphql](https://github.com/linear/linear/blob/master/packages/sdk/src/schema.graphql)
- Webhook payloads include `actor` and `updatedFrom` (previous values), which are the hooks for loop prevention and diffing — [Linear webhooks](https://linear.app/developers/webhooks)

### Inferences

**Comparing grouping options for "Narwarp ticket T ↔ Linear issues {I1..In}":**

| Option | Cardinality fit | Pollutes shared team structure? | Needs admin/elevated rights? | Queryable from Linear | Verdict |
|---|---|---|---|---|---|
| **Attachment on each issue** (url = `https://<stable-host>/t/<ticket-uuid>`) | Many-to-many: an issue can carry several Narwarp-ticket attachments | No: it shows as a link card on the issue | No: works with a personal API key | Yes: `attachmentsForURL(url)`, or `issues(filter:{attachments:{some:{url:{eq:…}}}})` | **Primary mechanism** |
| Linear **Project** per ticket | Each issue is in ≤1 project, so this collides with the team's real project usage | Yes | Usually needs project-creation rights, and is socially costly | Yes | Only when T *is* a team-sanctioned project |
| **Parent issue** with sub-issues | ≤1 parent per issue, so it restructures hierarchy | Yes | No | Yes | Only when T maps to a real epic the user owns |
| **Label** per ticket | Many-to-many | Yes: shared taxonomy, label sprawl | Often team or workspace label rights | Yes | Avoid. At most one personal label such as `narwarp` to mark "tracked" |
| **Custom view** | A view over the attachment filter | No, if private (`shared:false`) | No | n/a, it is a view | Nice-to-have UI for the user inside Linear |
| **Relations** (blocks/related) | Pairwise issue↔issue | Mildly | No | Yes | Use for real dependencies only, not for grouping |

**The recommended data model, in Narwarp:**
- Keep a `ticket_links(ticket_id, linear_issue_id, linear_identifier, link_role, created_by, created_at)` table as the **source of truth**.
- When linking an issue, upsert an attachment whose `url` is the ticket's stable URL and whose `metadata` is `{narwarp_ticket_id, project, mode}`. Because the attachment is idempotent per `(issue, url)`, re-running the link is safe.
- Reconcile periodically with `attachmentsForURL(ticketUrl)`, so that links made or removed by hand in Linear show up in Narwarp.
- If T is promoted into a real team-level body of work, optionally create a Linear Project and add an `EntityExternalLink` (resource link) from the project back to T.

**Choosing the attachment URL:** it has to be stable and unique per ticket. An `https://` URL is the safest choice. It could be a static page, or a Tailscale/`ts.net` URL that serves a read-only ticket summary to the user's devices. Whether Linear rejects custom schemes such as `narwarp://` is not documented; see Gaps.

**Rolling status up from issues to the ticket:** compute it from `WorkflowState.type` across the linked issues, because team-specific state *names* differ. For example:
- the ticket is "Done" only when every linked issue is `completed`, `canceled` or `duplicate`
- it is "In Progress" if any linked issue is `started`
- otherwise it is "Todo" or "Backlog"

Narwarp should not push its 4-column status *into* Linear automatically. The many-to-one mapping is lossy, and Linear's states (triage, duplicate, canceled) have team semantics.

**Two-way sync pitfalls, and how to mitigate each:**
1. **Echo loops.** With a personal API key, the webhook or poll `actor` is the user, so Narwarp can't tell its own writes from the user's edits in Linear. Keep a local write journal of mutation results (`issue.id` plus `updatedAt`) and ignore changes that match your own recent writes. In Mode B, ignore events whose `actor` is the app user.
2. **Field ownership.** Decide a single writer per field. Linear owns title, state, assignee and estimate. Narwarp owns ticket grouping, project/mode and schedule. Never mirror free-text descriptions both ways; append a backlink line instead.
3. **Lost events while the laptop sleeps.** Linear stops retrying after the 6 h attempt, so run an `updatedAt > lastCursor` reconciliation on wake, even when webhooks are in use.
4. **Attribution and noise in a shared workspace.** In Mode A, agent-generated comments appear as the user. Require approval for any Linear write visible to teammates, and prefix agent-authored text.
5. **Identity drift.** Store both the UUID and the identifier (`ENG-123`). Identifiers change when an issue moves teams; UUIDs don't.
6. **Deletion and archival.** Handle `remove` webhook actions and `archivedAt`/`trashed` fields, and don't resurrect issues on the next sync.

### Gaps
- Not documented: whether attachment URLs must be http(s) or must be reachable, and whether there is a maximum number of attachments per issue.
- Not documented: whether a private custom view can filter on `attachments.url`. The `filterData` JSON shape for that case was not verified.
- I could not find Linear guidance specifically about third-party two-way sync patterns. The mitigations above are inferences from the webhook payload fields and the attachment semantics.

---

## 3. Discord: bots vs user tokens, reading a work server you don't administer, user-installable apps, threads/forums, rate limits, discord.py, MCP servers, Claude Code Channels, Kimaki

### Takeaway
Automating a user account (a "self-bot") is forbidden and can lead to account termination. That rules out any MCP server or harness that logs in as the user. To read a **work server the user doesn't administer**, someone with Manage Server must invite a bot. The Message Content intent is just a portal toggle for apps under 10,000 users, a threshold changed on 2026-06-10. Without an admin, the only legitimate path is a **user-installed app**. It cannot read channels passively, but its *message context-menu command* can capture specific messages on demand, because the content of the targeted message is delivered to the app. For agent message boards and nudges, a **personal server plus your own bot** (forum channel = board, thread = ticket, DMs = nudges) gives full control with no ToS risk. Claude Code Channels does officially ship a Discord plugin (research preview, Bun-based, pairing plus an allowlist). Kimaki is a Discord↔OpenCode bridge, not a Claude Code bridge.

### Cited Findings

**Policy**
- Discord's support article on self-bots states that automating normal user accounts outside the OAuth2/bot API is forbidden and "can result in an account termination if found". Discord directs automation to bot accounts instead. This came via a search summary; the page itself returned HTTP 403 when fetched — [Discord support: Automated user accounts](https://support.discord.com/hc/articles/115002192352)
- The Developer Policy (effective July 8, 2024) **[>6 mo]**: "Do not use API Data for any purpose outside of what is necessary to provide your stated functionality". Discord's policy FAQ adds: "we do not allow AI companies to train their models using your messages". Both came via search snippets — [Discord Developer Policy](https://support-dev.discord.com/hc/en-us/articles/8563934450327); [Discord: Important policy updates](https://discord.com/safety/important-policy-updates)

**Server installs and the Message Content intent**
- A server (guild) install "must be authorized by a member with `MANAGE_GUILD`" — [Discord: Application resource](https://docs.discord.com/developers/resources/application)
- The privileged intents are `GUILD_PRESENCES`, `GUILD_MEMBERS` and `MESSAGE_CONTENT`. Without `MESSAGE_CONTENT`, the `content`, `embeds`, `attachments`, `components` and `poll` fields come back empty, *except* for:
  - messages the app sends
  - DMs with the app
  - messages that mention the app
  - "the message a message context menu command is used on"

  Passing an unapproved intent closes the connection with code 4014 — [Discord: Gateway](https://docs.discord.com/developers/events/gateway)
- 2026-06-10 change: review is now tied to total reach. "Apps that reach 10,000 or more total users now need to go through a review process", with annual reapplication. Apps under 10,000 users "see no changes". Previously the trigger was 100+ servers — [Discord blog, 2026-06-10](https://discord.com/blog/updated-requirements-to-how-apps-access-data-in-servers). The gateway page still says "verified apps (required for apps in 100+ guilds)" must also be approved, so the docs are mid-transition — [Discord: Gateway](https://docs.discord.com/developers/events/gateway)

**Gateway and REST limits**
- Gateway: 120 events sent per connection per 60 s. There is a limit of "1000 IDENTIFY calls ... in a 24-hour period" (RESUMEs excluded), and exceeding it terminates sessions and **resets the bot token**. `max_concurrency` limits identifies per 5 s — [Discord: Gateway](https://docs.discord.com/developers/events/gateway)
- REST: "All bots can make up to 50 requests per second", plus per-route buckets (`X-RateLimit-Bucket`). The invalid-request limit is 10,000 per 10 minutes (401/403/429 responses), after which the IP is "automatically and temporarily restricted". Interaction endpoints are not bound by the global limit. On a 429, honor `Retry-After`, and don't hard-code limits — [Discord: Rate limits](https://docs.discord.com/developers/topics/rate-limits)

**User-installable apps and OAuth scopes**
- Integration types are `GUILD_INSTALL` (0) and `USER_INSTALL` (1). User installs are "visible *only* to the authorizing user", need no server permissions, and are usable "across all of an authorizing user's servers, DMs, and GDMs", but they "respect the user's permissions" there. "Apps installed only to a user can't take actions in a server." The example user-install config uses only the `applications.commands` scope (`integration_type=1`) — [Discord: Application resource](https://docs.discord.com/developers/resources/application); [Discord: OAuth2](https://docs.discord.com/developers/topics/oauth2)
- User-installed apps became GA on 2024-06-27 **[>6 mo]** and are usable in all servers. The `USE_EXTERNAL_APPS` permission is on by default. When it is disabled and the app isn't in the server, responses are ephemeral. Follow-ups are capped at 5 — [discord-api-docs changelog (PR #6958)](https://github.com/discord/discord-api-docs/pull/6958/files)
- OAuth scopes offer no general way to read a user's messages:
  - `messages.read` is "for local rpc server api access"
  - `rpc` and `dm_channels.read` are "only available to approved partners"
  - "Some scopes require approval from Discord"

  — [Discord: OAuth2](https://docs.discord.com/developers/topics/oauth2)

**Threads and forums**
- Forum channels (type 15) contain only threads. Each post is a thread, and messages can't be sent directly to the forum. Tags are set via `available_tags`/`applied_tags`, and `REQUIRE_TAG` forces one. `SEND_MESSAGES_IN_THREADS` is needed to talk in threads, while creating a forum post only needs `SEND_MESSAGES`. Apps are "automatically subscribed to thread events and active threads" on connect, and `THREAD_LIST_SYNC` fires when access to a channel changes — [Discord: Threads](https://docs.discord.com/developers/topics/threads)

**Libraries**
- discord.py's latest release is 2.7.1 (2026-03-03), after 2.7.0 (2026-02-27). The 2.6.x line ran from Aug to Oct 2025. It requires Python ≥3.8 and is maintained by Rapptz. The last release is now about 7 months old **[>6 mo]** — [PyPI discord.py](https://pypi.org/project/discord.py/)
- A trends aggregator reports a last push to the discord.py repo on 2026-04-20 — [gittrend](https://gittrend.io/repo/Rapptz/discord.py) (third-party)
- py-cord 2.8.0 is reported released 2026-05-18 — [pepy py-cord](https://pepy.tech/project/py-cord) (third-party snippet)

**Claude Code Channels (verified in official docs)**
- What it is: "A channel is an MCP server that pushes events into your running Claude Code session". It is in research preview. "Telegram, Discord, and iMessage are included." Each plugin requires Bun. "Events only arrive while the session is open" — [Claude Code docs: Channels](https://code.claude.com/docs/en/channels)
- Discord setup:
  1. Create your own bot and enable the **Message Content Intent**.
  2. Invite it to a server with View Channels, Send Messages, Send Messages in Threads, Read Message History, Attach Files and Add Reactions.
  3. Run `/plugin install discord@claude-plugins-official`, then `/discord:configure <token>`. The token is saved to `~/.claude/channels/discord/.env`.
  4. Start `claude --channels plugin:discord@claude-plugins-official`.
  5. DM the bot to get a pairing code, run `/discord:access pair <code>`, then `/discord:access policy allowlist`.

  — [Claude Code docs: Channels](https://code.claude.com/docs/en/channels); plugin source at [claude-plugins-official/discord](https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins/discord)
- Requirements and enforcement: Channels need claude.ai or Console API-key auth and are not on Bedrock, Vertex or Foundry. Team/Enterprise admins must enable `channelsEnabled`. During the preview, `--channels` accepts only allowlisted plugins, and custom channels need `--dangerously-load-development-channels` — [Claude Code docs: Channels](https://code.claude.com/docs/en/channels)
- Third-party write-ups conflict with the official doc: DataCamp says API-key logins are *not* supported and that v2.1.80+ is required, while the official doc says Console API keys work — [DataCamp](https://www.datacamp.com/tutorial/claude-code-channels) (third-party). The research preview was announced around 2026-03-20 — [claudefa.st](https://claudefa.st/blog/guide/development/claude-code-channels) (third-party)
- Building a custom channel — [Channels reference](https://code.claude.com/docs/en/channels-reference):
  - The server declares `capabilities.experimental['claude/channel']`, emits `notifications/claude/channel` with `{content, meta}`, and runs over stdio.
  - Optional permission relay: declare `claude/channel/permission`, receive `notifications/claude/channel/permission_request` (with `request_id`, `tool_name`, `description`, `input_preview`) and reply with `.../permission` `{request_id, behavior}`.
  - Chat plugins "poll the platform's API ... No URL to expose".
  - "An ungated channel is a prompt injection vector". Gate on the sender, not the room.
  - Notifications are not acknowledged and are dropped silently if the channel isn't registered.
  - On the v2 MCP client runtime, a channel server that negotiates protocol revision 2026-07-28 does not register.

  — [Claude Code docs: Channels reference](https://code.claude.com/docs/en/channels-reference); [Channels](https://code.claude.com/docs/en/channels)

**Kimaki and Discord MCP servers**
- Kimaki is a "collaborative agent orchestrator, inside Discord" that drives **OpenCode** (requires OpenCode 2). A Discord channel maps to a project directory and a thread to a session. The local CLI is "the bridge between Discord and your machine". It runs either in gateway mode with Kimaki's pre-built bot or with a self-hosted bot. It offers voice notes and calls and role-based access (Manage Server, or a "Kimaki" role). MIT license, about 1.4k stars — [github.com/remorses/kimaki](https://github.com/remorses/kimaki)
- Discord MCP servers:
  - Bot-token servers such as `v-3/discordmcp` read and send channel messages via a bot — [glama listing](https://glama.ai/mcp/servers/@v-3/discordmcp) (directory listing; its repo links are placeholders)
  - `elyxlz/discord-mcp` "uses web scraping" and a user login with "no bot permissions or API tokens", which is self-bot territory — [mcpservers.org](https://mcpservers.org/es/servers/elyxlz/discord-mcp)
  - No official Discord MCP server was found.

### Inferences

**Work server you don't administer, options ranked:**
1. **Ask the admin to invite a minimal "Narwarp (Elijah)" bot.** Give it read-only access in specific channels (View Channel and Read Message History only), add Send Messages only where desired, and turn on Message Content. That toggle is trivial: a single-server personal bot sits far below 10,000 users.
2. **A user-installed app with a message context-menu command** (for example "Apps → Send to Narwarp"). It captures the targeted message's content legitimately, on demand, in any server, with no admin. It offers no passive reading, no history and no thread watching, and replies are ephemeral if the server disables External Apps.
3. **Manual forwarding** of work messages to the bot's DMs: DM content is always visible to the app.
4. **Never** use user tokens or scraping MCP servers. Whatever their technical appeal, they put the user's *work* account at risk of termination.

**Agent boards and nudges:** create a private "Narwarp HQ" server. Give each Narwarp project a forum channel, make each ticket a post (thread), and use tags for mode (Work/Life) and status. Agents post updates and handoffs as thread messages, and the user's phone gets notifications for free. Send nudges and accountability check-ins by DM. Narwarp's own board stays the source of truth, and Discord acts as a mirror and remote UI.

**Laptop-specific gotchas:**
- The gateway is an *outbound* WebSocket, so no tunnel is needed.
- Sleep/wake cycles must RESUME rather than IDENTIFY. A reconnect bug that re-IDENTIFIES in a loop can hit 1,000 per day and reset the token.
- Slash commands and context menus can be received over the gateway connection, so no public interactions URL is needed. That is my understanding of the interactions docs, which were not re-fetched; see Gaps.

**Library choice:** discord.py is still the default for a Python sidecar, but releases have slowed (last one in March 2026). For in-process Rust, serenity or twilight would be the candidates (not researched).

**Claude Code Channels fit:** the Discord plugin is a ready-made "phone → running Claude Code session" bridge. But it is per-session, a research preview, Bun-based, and keeps the bot token in a plaintext `.env`. A better long-term fit for Narwarp is to ship its own channel MCP server. It would push Narwarp board, ticket and integration events into the Claude Code sessions it spawns, and optionally relay permission prompts to the Narwarp TUI or Discord DM. It would gate on sender identity, as the reference requires.

**Kimaki** is a useful design reference (channel = project, thread = session), but it targets OpenCode. Its README also mentions using Claude Pro/Max subscriptions via OAuth, which has its own terms questions; see Gaps.

### Gaps
- Not fetched: the support article on Discord's "Request a copy of your data" package, so whether it is a usable fallback for work-server history is unverified. My understanding is that it contains only the user's own messages and can take days.
- Not confirmed from the docs: whether user-installed apps receive `INTERACTION_CREATE` over the bot's gateway connection (I believe they do), and exactly what message data a context-menu interaction carries when the app isn't in the server.
- The current text of Discord's Developer Policy about LLM processing of message content (as opposed to training) wasn't verified. Sending work-server messages to an LLM API may also conflict with the *employer's* policies.
- Whether Kimaki's or OpenCode's use of Claude Pro/Max OAuth complies with Anthropic's current consumer terms is out of scope here; the Claude Code researcher should check it.

---

## 4. Personal email: Gmail API (labels, drafts, history, Pub/Sub push), IMAP IDLE/SMTP and JMAP for other providers, MCP servers, safe patterns

### Takeaway
For Gmail:
- Use **`gmail.readonly` plus `history.list` incremental sync**, triggered by **Cloud Pub/Sub** (a *pull* subscription needs no public endpoint) or by cheap polling.
- `history.list` costs 2 quota units against 6,000 per user per minute.
- Every Gmail scope that can read bodies, or create drafts, is **restricted**. `gmail.compose` ("manage drafts and send") cannot technically stop sending, so **draft-only has to be enforced in Narwarp's tool layer**.
- Google now offers an official **Gmail MCP server (Developer Preview)** that deliberately has no send tool: search, read threads, drafts and labels only.

For a personal-use OAuth app, publish the consent screen to "In production" and accept the unverified-app warning (the personal-use exception, under 100 users). Otherwise refresh tokens die every 7 days.

For iCloud, IMAP IDLE with an app-specific password is the only option, and that password grants full mailbox access. Fastmail offers JMAP with API tokens.

Inbound email is the highest-risk prompt-injection surface in this whole design. Agents that read mail must not also hold an exfiltration channel.

### Cited Findings

**Gmail scopes** (page last updated 2026-09-10) — [Gmail API scopes](https://developers.google.com/workspace/gmail/api/auth/scopes)

| Class | Scopes |
|---|---|
| Non-sensitive | `gmail.labels` ("View and change email labels") |
| Sensitive | `gmail.send` |
| Restricted | `gmail.readonly`, `gmail.compose` ("Manage drafts and send email"), `gmail.insert`, `gmail.modify`, `gmail.metadata`, `gmail.settings.basic`, `gmail.settings.sharing`, `https://mail.google.com/` |

"If you store restricted scope data on servers (or transmit), then you must go through a security assessment."

**Official Gmail MCP server** (page last updated 2026-10-09) — [Gmail MCP server](https://developers.google.com/workspace/gmail/api/guides/configure-mcp-server)
- Status and endpoint: "Developer Preview" via the Google Workspace Developer Preview Program, at `https://gmailmcp.googleapis.com/mcp/v1` (Streamable HTTP).
- Auth: OAuth 2.0 with a Web-app client, scopes `gmail.readonly` + `gmail.compose`. You must enable both `gmail.googleapis.com` and `gmailmcp.googleapis.com`.
- Tools: `create_draft`, `get_thread`, `label_message`, `label_thread`, `list_drafts`, `list_labels`, `search_threads`, `unlabel_message`, `unlabel_thread`. The sidebar also lists `get_message` and `create_label`. There is no send tool.
- Claude setup: a custom connector with redirect `https://claude.ai/api/mcp/auth_callback`. There are no Claude Code-specific instructions.
- It warns about "indirect prompt injection" from untrusted emails.
- A vendor blog (July 2026) gives the same picture (10 tools, gmail.readonly plus gmail.compose, cannot send). It also reports that the long-popular `GongRzhe/Gmail-MCP-Server` (about 1.2k stars) was **archived in early 2026**: it sends immediately and requests `gmail.modify` + `gmail.settings.basic`. Alternatives it names are the ArtyMcLabin fork and `shinzo-labs/gmail-mcp` (50+ endpoints, sending included). It notes that IMAP-app-password-based servers get full-mailbox access with no scopes — [Drag blog](https://www.dragapp.com/blog/gmail-mcp-server/) (vendor blog; its own product is promoted)

**Push and incremental sync**
- `users.watch` → a Pub/Sub topic, after you grant publish rights to `gmail-api-push@system.gserviceaccount.com`. You can filter with `labelIds` (e.g. INBOX) and `labelFilterBehavior`. Re-call watch "at least once every 7 days" (daily is recommended). The payload is `{emailAddress, historyId}`. There is a "maximum notification rate of one event per second" per user, and excess notifications are dropped. Subscriptions can be push (webhook) or pull. Notifications "may occasionally be delayed or dropped", so fall back to periodic `history.list` — [Gmail push](https://developers.google.com/workspace/gmail/api/guides/push)
- Sync (page updated 2026-09-15): do a full sync first, then partial sync via `history.list(startHistoryId)`. "History records are typically available for at least one week", but the window can be "significantly shorter". A 404 means you must do a full sync — [Gmail sync](https://developers.google.com/workspace/gmail/api/guides/sync)
- Pull subscriptions are initiated by the subscriber, which is what makes them laptop-friendly — [Cloud Pub/Sub pull](https://cloud.google.com/pubsub/docs/pull) (not re-fetched this session)

**Quotas** (page updated 2026-09-10) — [Gmail quota](https://developers.google.com/workspace/gmail/api/reference/quota)
- 1,200,000 units/min per project and 6,000 units/min per user per project.
- Costs: `messages.get` 20, `messages.list` 5, `history.list` 2, `drafts.create` 10, `messages.send` 100, `threads.get` 40, `watch` 100.
- New: "Charges for exceeding quota limits are planned for later in 2026", with an 80,000,000 units/day/project billing threshold.

**OAuth for a personal app**
- "Testing" publishing status (External user type) means refresh tokens expire in 7 days, unless the app requests only basic profile scopes. There is a limit of 100 refresh tokens per Google Account per client: the oldest is silently invalidated. Tokens die after 6 months unused, and a password change can revoke tokens that carry Gmail scopes — [Google OAuth 2.0](https://developers.google.com/identity/protocols/oauth2)
- Google's personal-use exception: apps for personal use with fewer than 100 users can skip verification, and users click through the "unverified app" warning — [Google Cloud help: unverified apps](https://support.google.com/cloud/answer/13464323) (via search summary)
- Practitioner report: after switching Testing → Production, re-consent once, because tokens issued earlier may keep the 7-day life — [sailjada tech blog, 2026-05](https://tech.sailjada.com/posts/2026-05-06-1924.html) (practitioner)
- Limited Use: Workspace API data may not be used to train non-personalized/foundational AI/ML models — [Google Workspace blog](https://workspace.google.com/blog/ai-and-machine-learning/api-policy-protections?hl=en); [Google Cloud help: application use cases](https://support.google.com/cloud/answer/13805798) (via search summary)

**Other providers**
- iCloud: IMAP at `imap.mail.me.com:993` (SSL) and SMTP at `smtp.mail.me.com:587` (STARTTLS). An app-specific password (`xxxx-xxxx-xxxx-xxxx`) is required with two-factor authentication. There is no POP — [Apple support 102525](https://support.apple.com/en-me/102525)
- Fastmail:
  - Mail is available over JMAP, IMAP or POP.
  - The JMAP session is at `https://api.fastmail.com/jmap/session` with `Authorization: Bearer`.
  - API tokens for JMAP are created in Settings → Privacy & Security → Manage API tokens. App passwords cover IMAP/SMTP.
  - Tokens are meant for personal apps; distributed apps should use OAuth. JMAP is CORS-enabled.

  — [Fastmail developer docs](https://www.fastmail.com/dev/)
- Protocol standards: IMAP IDLE (RFC 2177) has servers drop idle connections after about 30 min, so clients re-issue IDLE every ≤29 min — [RFC 2177](https://datatracker.ietf.org/doc/html/rfc2177). JMAP defines push via an EventSource stream — [RFC 8620 §7](https://datatracker.ietf.org/doc/html/rfc8620#section-7) (both standards; not re-fetched)

**Email MCP servers beyond Gmail**
- Fastmail: no official server was found. Community JMAP servers include `jmhron/FastMailMCP`, `MadLlama25/fastmail-mcp` (mail, contacts and calendar, with a dry-run mode) and `NOVA-3951/Fastmail-MCP` (read-oriented) — [glama: MadLlama25/fastmail-mcp](https://glama.ai/mcp/servers/@MadLlama25/fastmail-mcp/blob/e3ce2823c4dc4c9d85f8a57306b549f894414432/README.md); [glama: NOVA-3951](https://glama.ai/mcp/servers/@NOVA-3951/Fastmail-MCP/blob/958429321b8552b770d1c8af4d182f7033d1c3fe/README.md) (community)
- Generic IMAP/SMTP:
  - `ai-zerolab/mcp-email-server` (Python; read-only IMAP mode when SMTP is omitted) — [sourcepulse](https://www.sourcepulse.org/projects/31018457)
  - `codefuturist/email-mcp` (TypeScript, LGPL-3.0, early-stage) — [Enterprise DNA directory](https://enterprisedna.co/directories/mcp/codefuturist-email-mcp)
  - `jdickey1/imap-email-mcp` (draft management) — [glama](https://glama.ai/mcp/servers/jdickey1/imap-email-mcp)

  All are community projects found via directory listings.

**Prompt injection**
- Practitioner framing: the "lethal trifecta" is an agent with private data, exposure to untrusted content, and an exfiltration channel. Email readers hit the first two by default — [Simon Willison, 2025-06-16](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) (practitioner) **[>6 mo]**
- Google's Gmail MCP page warns about indirect prompt injection from untrusted emails — [Gmail MCP server](https://developers.google.com/workspace/gmail/api/guides/configure-mcp-server)

### Inferences

**Gmail recipe for Narwarp:**
- Create a GCP project, use a Desktop OAuth client with a loopback redirect, and request scopes `gmail.readonly` + `gmail.compose`. Optionally add `gmail.modify` *only* if Narwarp should apply labels or archive. Set the publishing status to *In production* (unverified), then re-consent once.
- Start with an initial full sync, then keep a `historyId` cursor.
- For triggers, either:
  - (a) a Pub/Sub **pull** subscription. A local client pulls or streams from the topic, so no tunnel is needed, but you must re-`watch` daily; or
  - (b) plain `history.list` every 60 s while awake. That is 2 units/min, roughly 0.03% of the per-user quota, and simpler.
- Whichever trigger you choose, also sync on wake.
- Personal-use exception: because Narwarp stores data only on the user's own laptop, the "store restricted scope data on servers" assessment trigger arguably doesn't apply. This is unverified.

**Draft-only enforcement:** `gmail.compose` allows sending, so Narwarp's email tool surface must expose only `create_draft`, `update_draft` and `list_drafts` to agents. Sending stays a human action in the Gmail UI, or a typed confirmation in Narwarp that the agent cannot trigger. The official Gmail MCP is the only off-the-shelf server that matches this policy by construction. It is in Developer Preview and needs enrollment, so treat it as optional for Claude Code sessions, not as Narwarp's ingestion path.

**Provider-agnostic design:** abstract a `MailSource` trait/protocol with three implementations:
- Gmail: API plus history IDs
- JMAP: Fastmail, with an EventSource push stream
- IMAP: iCloud and others, using IDLE on INBOX, CONDSTORE/UIDs for incremental sync, and opening mailboxes with `EXAMINE` (read-only) unless an action is approved

For iCloud the app-specific password is all-or-nothing, so the least-privilege boundary has to live inside Narwarp.

**Injection containment:**
- Raw email bodies go to a quarantined summarizer, ideally a local Ollama model, that has **no tools**.
- Downstream agents see only the structured extraction (sender, intent, dates, action items) tagged `untrusted_source=email`.
- Any agent that has read raw mail in its context must not hold web-fetch, outbound messaging, or "send" capabilities in that same session.
- Links and attachments from mail are never auto-fetched.

### Gaps
- Not determined: whether the official Gmail MCP's `label_*` tools work under only `gmail.readonly` + `gmail.compose`, given that applying labels via the REST API normally needs `gmail.modify`.
- Fastmail API token scope granularity (for example, a read-only mail token) is not described on Fastmail's dev page. Whether Fastmail's session object exposes `eventSourceUrl` was not verified.
- iCloud's IDLE behavior and timeouts are not documented by Apple.
- Not verified: Google's exact current wording on whether a personal-use app requesting *restricted* scopes in production must be verified once a second account (e.g. a spouse) is added.

---

## 5. GitHub: gh CLI, the official GitHub MCP server, REST/GraphQL, the notifications API, webhooks vs polling on a laptop, PR/check status, Projects v2, fine-grained PATs vs a GitHub App

### Takeaway
For a single user on a laptop:
- **Poll, don't webhook.** The notifications API is built for polling: `If-Modified-Since` returns a 304 that "leaves your rate limit untouched", and `X-Poll-Interval` tells you how often to poll. `gh webhook forward` is explicitly "only designed for use during testing".
- **The notifications endpoints only accept a classic PAT.** Fine-grained PATs, which GitHub recommends for everything else, can't be used there, and they also can't call the Checks API or reach user-owned Projects.
- So use `gh` (and its token) for the user's own interactive actions and notifications polling. Use fine-grained PATs scoped to specific repositories for agent tools. Create a personal GitHub App only if Narwarp needs its own bot identity or webhooks.
- The official GitHub MCP server (remote and local) is mature. Configure it with narrow toolsets, read-only where possible, and **lockdown mode** for public repos.

### Cited Findings

**Official GitHub MCP server**
- Remote server at `https://api.githubcopilot.com/mcp/` (OAuth or PAT). The local server is the Docker image `ghcr.io/github/github-mcp-server` or the binary (`github-mcp-server stdio`). Local OAuth opens a browser login on `127.0.0.1:8085`, and a PAT takes precedence over it — [github-mcp-server README](https://github.com/github/github-mcp-server)
- Toolsets — [README](https://github.com/github/github-mcp-server):
  - Default: `context, repos, issues, pull_requests, users`
  - Others: `actions, code_quality, code_security, copilot, copilot_issue_intents, dependabot, discussions, gists, git, governance, labels, notifications, orgs, projects, secret_protection, security_advisories, stargazers`
  - Remote-only: `copilot_spaces` and `github_support_docs_search`
  - `all` enables everything. `--read-only` "takes priority" over `--tools`.
  - Insiders mode is available via `/mcp/insiders` or `X-MCP-Insiders: true`.
  - The notifications toolset has `list_notifications`, `get_notification_details`, `dismiss_notification`, `mark_all_notifications_read` and `manage_*_subscription`. The projects toolset has `projects_get/list/write`.
- History: remote public preview 2025-06-12, GA 2025-09-04 with "OAuth-based authentication" **[>6 mo]** — [GitHub changelog (preview)](https://github.blog/changelog/2025-06-12-remote-github-mcp-server-is-now-available-in-public-preview/); [GitHub changelog (GA)](https://github.blog/changelog/2025-09-04-remote-github-mcp-server-is-now-generally-available/)
- On 2025-12-10 **[>6 mo]**, GitHub added the `X-MCP-Tools` header (locally `--tools=`) for per-tool allowlists, and **lockdown mode** via `X-MCP-Lockdown: true`. In lockdown mode, `issue_read:get` errors when the author lacks push access, and comment lists filter out such users. Private repos are unaffected — [GitHub changelog](https://github.blog/changelog/2025-12-10-the-github-mcp-server-adds-support-for-tool-specific-configuration-and-more)
- Claude Code setup (2.1.1+): `claude mcp add-json github '{"type":"http","url":"https://api.githubcopilot.com/mcp","headers":{"Authorization":"Bearer YOUR_GITHUB_PAT"}}'`. The official guide documents OAuth only for the local Docker/binary server, not for remote use in Claude Code — [install-claude.md](https://github.com/github/github-mcp-server/blob/main/docs/installation-guides/install-claude.md). A practitioner note from September 2025 also said Claude Code lacked OAuth for the remote server — [Classmethod](https://dev.classmethod.jp/en/articles/github-remote-mcp-ga) (practitioner, **[>6 mo]**)

**Notifications API**
- "These endpoints only support authentication using a personal access token (classic)", with the `notifications` or `repo` scope. Poll with `If-Modified-Since`: a 304 "leaves your rate limit untouched". Obey `X-Poll-Interval`. Reasons include `review_requested`, `mention`, `assign`, `ci_activity`, `approval_requested`, `security_alert` and the newer `agent_session_finished` (Copilot cloud agent) — [GitHub REST: notifications](https://docs.github.com/en/rest/activity/notifications)

**Tokens**
- Fine-grained PATs are limited to one owner (user *or* org) and selected repositories. They **cannot**:
  - contribute to public repos where you aren't a member
  - work as an outside collaborator
  - access multiple orgs
  - access Packages
  - call the **Checks API**
  - access **Projects owned by a user account**

  Org owners can require approval; until it's granted, the token reads only public resources. There is a cap of 50 fine-grained tokens per user. GitHub recommends a GitHub App "for long-lived integrations", and `gh` / Git Credential Manager for command-line use — [GitHub docs: managing PATs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- GitHub App user access tokens expire after 8 hours and their refresh tokens after 6 months — [GitHub docs: refreshing user access tokens](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/refreshing-user-access-tokens) (from prior knowledge of this page; not re-fetched this session)

**Rate limits**
- Authenticated users get 5,000 requests/h. App installations get 5,000/h, growing with repository count up to 12,500 (15,000 on GHEC). Unauthenticated: 60/h — [GitHub REST rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)
- Secondary limits: 100 concurrent requests (shared REST and GraphQL) and 900 points/min for REST (GET = 1 point, mutations = 5). Content creation is capped at 80/min and 500/h — [GitHub REST rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)

**Webhook forwarding**
- `gh extension install cli/gh-webhook`, then `gh webhook forward --repo=… --events=… --url=…`. It works for repos and orgs only, with "Only one person can use webhook forwarding at a time" per repo/org ("Hook already exists"). It runs only while the command runs. It "is only designed for use during testing and development ... not supported for use in production". Org forwarding needs `admin:org_hook` — [GitHub docs: gh webhook forward](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/using-the-github-cli-to-forward-webhooks-for-testing)
- smee.io is a webhook relay commonly used in GitHub App tutorials — [smee.io](https://smee.io) (not evaluated in depth)

### Inferences
- **Event ingestion on the laptop:** one poll loop on `GET /notifications` with `If-Modified-Since`, at the interval `X-Poll-Interval` dictates.
  - It covers review requests, mentions, assignments, CI on runs you triggered, and Copilot agent completions, essentially for free in rate-limit terms.
  - For each new thread, fetch the subject (PR or issue) and its check status via GraphQL, or via REST commit status. Check runs need a classic PAT or `gh` token, because fine-grained PATs can't call the Checks API.
  - Map the PR/issue URL into Narwarp tickets via the same link table used for Linear.
- **Tokens:**
  - (1) The `gh` CLI's stored credential is for interactive and user-authored actions. Narwarp can shell out to `gh` or read its token with `gh auth token`.
  - (2) Use separate fine-grained PATs scoped to specific repositories for Claude Code's GitHub MCP, so an injected agent can't touch the user's other repos or orgs.
  - (3) A personal GitHub App is worth it only if Narwarp should comment as `narwarp[bot]` or receive webhooks. Those need a stable tunnel and must be reconciled with polling after sleep.
- **GitHub MCP configuration for agent sessions:** use the remote server with a fine-grained PAT. Restrict toolsets to `repos,issues,pull_requests,actions`, set `X-MCP-Lockdown: true`, and use read-only for "research" agents. Allow write tools such as `merge_pull_request`, `issue_write`, `create_or_update_file` and `push_files` only behind Narwarp approval. Observation from this research environment's GitHub connector: tools are now *consolidated* (`issue_read`, `issue_write`, `sub_issue_write`, `pull_request_read`, `pull_request_review_write`, `actions_get`/`actions_list`/`actions_run_trigger`, `get_job_logs`, `enable_pr_auto_merge`, `request_copilot_review`, `run_secret_scanning`, `list_issue_types`/`list_issue_fields`). That keeps tool count, and so context cost, manageable.
- **Projects v2:** not relevant as an integration target. Narwarp's kanban is the user's planning surface; Projects v2 is GraphQL-only and user-owned projects are unreachable with fine-grained PATs. Integrate only if the work org actually plans in Projects.

### Gaps
- The docs don't clarify whether `gh`'s OAuth-app token, which uses classic-style scopes, works for `/notifications`. In practice `gh api notifications` is widely used, but no doc says so explicitly.
- Not verified: the typical `X-Poll-Interval` value (60 s is commonly seen), and whether lockdown mode is available as a local flag (`--lockdown-mode` or `GITHUB_LOCKDOWN_MODE`, per third-party guides only).
- Not determined: whether `projects_v2_item` webhooks exist for user-owned (as opposed to org-owned) projects.

---

## 6. Cross-cutting: event ingestion on a laptop, tunnels, macOS Keychain, OAuth for personal apps, integration platforms

### Takeaway
Prefer **outbound-only** mechanisms, which need no inbound exposure at all:
- Discord gateway WebSocket
- Gmail Pub/Sub pull or `history.list`
- IMAP IDLE or JMAP EventSource
- Linear and GitHub polling

Webhooks only help when you control the app registration (a Linear OAuth app with admin consent, or a GitHub App). Even then they need a stable public HTTPS endpoint (Tailscale Funnel or a named Cloudflare Tunnel, *not* Quick Tunnels) and a reconciliation sweep on wake. Store all tokens in the macOS Keychain; the fork already contains a Keychain wrapper. Managed integration platforms (Composio, Nango, Pipedream, n8n) are poor value for a single user, because they put a third party in custody of every token.

### Cited Findings

**Tunnels**
- Tailscale Funnel ([Tailscale Funnel](https://tailscale.com/kb/1223/funnel)):
  - It is beta and "available for all plans".
  - It listens only on ports `443`, `8443` and `10000`, under the tailnet's `*.ts.net` domain, and only over TLS, which is terminated on the device.
  - Bandwidth limits are "non-configurable".
  - It needs Tailscale v1.38.3+, MagicDNS, HTTPS certificates and a `funnel` node attribute.
  - Let's Encrypt rate limits can block new certificates for up to 34 h.
- Cloudflare Quick Tunnels (TryCloudflare) — [Cloudflare docs: Quick Tunnels](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/trycloudflare/):
  - They need no account or domain, but "The hostname changes each time".
  - Each tunnel supports 200 in-flight requests, with no SSE and "no uptime guarantee". They are "for testing and development".
  - For stable hostnames, use a (named) Cloudflare Tunnel.

**Webhook constraints by source**
- Linear needs a public HTTPS non-localhost URL, a 5 s response, and has retries that end after 1 min, 1 h and 6 h — [Linear webhooks](https://linear.app/developers/webhooks)
- GitHub's `gh webhook forward` is for testing only — [GitHub docs](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/using-the-github-cli-to-forward-webhooks-for-testing)
- Gmail push can use pull subscriptions, but notifications can be dropped, so a polling fallback is recommended — [Gmail push](https://developers.google.com/workspace/gmail/api/guides/push)
- Claude Code channel plugins for chat platforms poll, so there is "No URL to expose" — [Channels reference](https://code.claude.com/docs/en/channels-reference)

**Keychain**
- Python `keyring` 25.7.0 (2025-11-16) **[>6 mo]** offers a macOS Keychain backend. Any script running under the same Python executable can read secrets keyring created without a prompt; restrict that via the item's Access Control in Keychain Access. It requires Python ≥3.9, and on Big Sur+ a universal2 Python ≥3.8.7 — [PyPI keyring](https://pypi.org/project/keyring/)
- Narwarp fork, local repo evidence (not a URL):
  - `crates/warpui_extras/src/secure_storage/mac.rs` implements a `SecureStorage` service with `security_framework::os::macos::keychain::SecKeychain::default()?.set_generic_password(service, key, value)`.
  - `AGENTS.md` lists `crates/managed_secrets` as a "Keychain / DPAPI / Linux Keyring abstraction + WASM proxy" that also uses Tink HPKE, per its `Cargo.toml`.

**OAuth for personal-use apps**
- Google: Testing mode means 7-day refresh tokens; the personal-use exception applies under 100 users, with an unverified-app warning — [Google OAuth 2.0](https://developers.google.com/identity/protocols/oauth2); [Google Cloud help](https://support.google.com/cloud/answer/13464323)
- Linear: PKCE, 24 h access tokens with refresh (since 2026-04-01 for all apps), localhost redirect examples — [Linear OAuth 2.0](https://linear.app/developers/oauth-2-0-authentication)
- Discord: a bot token is a static credential. 1,000 IDENTIFY calls per day, and exceeding that resets the token — [Discord: Gateway](https://docs.discord.com/developers/events/gateway)
- GitHub: fine-grained PATs can have infinite lifetimes unless policy blocks it; classic tokens unused for a year are removed — [GitHub docs: managing PATs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)

**Integration platforms** (mostly vendor-authored comparisons)
- Composio markets "50,000+ pre-built, LLM-optimized tools" with managed credentials — [Composio vs Nango](https://composio.dev/blog/composio-vs-nango) (vendor)
- Nango is described as open-source and code-first: you write TypeScript integration and sync logic, covering 800+ APIs — [Composio: Nango alternatives](https://composio.dev/content/nango-alternatives-ai-agents) (competitor-authored)
- n8n is framed as a workflow tool that can connect agents to MCP servers — [Merge: n8n vs Composio](https://www.merge.dev/blog/n8n-vs-composio) (vendor)
- Pipedream: Workday signed a definitive agreement to acquire it on 2025-11-19, with closing expected by 2026-01-31. Pipedream has 3,000+ connectors — [Workday newsroom](https://newsroom.workday.com/2025-11-19-Workday-Signs-Definitive-Agreement-to-Acquire-Pipedream?asPDF=1); [Pipedream blog](https://pipedream.com/blog/pipedream-to-be-acquired-by-workday/); [SiliconANGLE](https://siliconangle.com/2025/11/19/workday-acquire-pipedream-extend-ai-agent-integrations-across-enterprise-apps/) **[>6 mo]**

### Inferences

**Suggested polling cadences** (all well within published limits):
- Linear: 60 s active, 5 min idle
- GitHub: whatever `X-Poll-Interval` says (304s are free)
- Gmail: `history.list` every 60 s, or a Pub/Sub pull
- IMAP: IDLE with a 25-min re-arm
- Discord: a persistent gateway connection

Run an immediate catch-up sweep on macOS wake and on network change. With these cadences, a laptop that sleeps loses nothing permanently, because every source has a cursor (Linear `updatedAt`, Gmail `historyId`, GitHub `Last-Modified`, IMAP UID/MODSEQ).

**Webhook endpoint, if Mode B or a GitHub App is adopted:** Tailscale Funnel on 443 to a local `narwarpd` listener that does nothing but verify the signature (Linear HMAC, GitHub `X-Hub-Signature-256`), enqueue the event and return 200 within 5 s. Funnel gives a stable `*.ts.net` name with no domain purchase. Cloudflare named tunnels are the alternative if the user owns a domain. Never use Quick Tunnels for registered webhooks, since the hostname changes on every run.

**Secrets:**
- Reuse `warpui_extras::secure_storage`, and the `managed_secrets` abstraction, for one Keychain item per integration credential, under a Narwarp service name.
- The `SecKeychain*` APIs that the current code uses are the legacy file-based keychain. Consider SecItem with the data-protection keychain and an access group for a signed build. This is unverified; check Apple's docs.
- Never put tokens in `.env` files. Note that Claude Code Channels does write the Discord token to `~/.claude/channels/discord/.env`; inject it from the Keychain at launch instead.
- Agents should never see raw tokens: `narwarpd` makes the calls.

**Integration platforms:** not worth it for one user.
- The value they offer is OAuth app registration, multi-tenant token storage and connector breadth. Narwarp needs four integrations, and three of them (Linear key, GitHub PAT/`gh`, Discord bot token) need no OAuth dance at all.
- They add a third-party custodian of work credentials, which may conflict with employer policy, plus a cloud dependency and a cost.
- Possible exception: a self-hosted n8n for no-code "life" automations. Licensing wasn't verified.

### Gaps
- Not verified: Apple's current guidance on the legacy keychain vs the data-protection keychain for unsigned or self-built apps.
- No independent, non-vendor comparison of Composio, Nango, Pipedream and n8n for single-user use was found. Pipedream's post-acquisition product and pricing status is unknown.
- Tailscale Funnel's actual bandwidth caps are unpublished.

---

## 7. Recommended integration architecture for Narwarp (SDK vs MCP, event ingestion, secrets, least-privilege/approval)

### Takeaway
**Narwarp's daemon owns ingestion and sync through direct API code.** It handles cursors, idempotency, rate limits, an audit journal, and the Keychain-held credentials. **Agents get capabilities through one Narwarp-hosted MCP "gateway"** that exposes curated, least-privilege, approval-gated tools, plus a Narwarp *channel* server that pushes events into Claude Code sessions. Vendor MCP servers (Linear `/mcp/readonly`, GitHub with lockdown and narrow toolsets, Gmail MCP's draft-only design) are allowed in Claude Code sessions for read-mostly, ad-hoc exploration. They are not the system of record. Anything visible to other people, or irreversible, requires explicit human approval. Sending email and merging PRs are never agent-initiated.

### Cited Findings (key facts the architecture rests on)
- Linear: webhooks and agent installs need admin, and polling is "especially discourage[d]" but budgeted (2,500 req/h, 3M points/h). Attachments are idempotent per URL and queryable by URL — [Linear webhooks](https://linear.app/developers/webhooks); [Linear developers: agents](https://linear.app/developers/agents); [Linear rate limiting](https://linear.app/developers/rate-limiting); [Linear attachments](https://linear.app/developers/attachments)
- Linear MCP has a read-only endpoint and accepts restricted API keys as bearer tokens — [Linear MCP docs](https://linear.app/docs/mcp)
- GitHub notifications are polling-optimized and need a classic PAT. The GitHub MCP supports read-only, per-tool allowlists and lockdown mode — [GitHub REST: notifications](https://docs.github.com/en/rest/activity/notifications); [GitHub changelog 2025-12-10](https://github.blog/changelog/2025-12-10-the-github-mcp-server-adds-support-for-tool-specific-configuration-and-more); [github-mcp-server](https://github.com/github/github-mcp-server)
- Gmail's restricted scopes include compose, which can send. The official Gmail MCP has no send tool. Pub/Sub pull and `history.list` are laptop-friendly — [Gmail scopes](https://developers.google.com/workspace/gmail/api/auth/scopes); [Gmail MCP](https://developers.google.com/workspace/gmail/api/guides/configure-mcp-server); [Gmail push](https://developers.google.com/workspace/gmail/api/guides/push)
- Discord: self-bots are banned; a bot needs a MANAGE_GUILD install; user-installed apps can capture context-menu-targeted messages — [Discord support](https://support.discord.com/hc/articles/115002192352); [Discord: Application](https://docs.discord.com/developers/resources/application); [Discord: Gateway](https://docs.discord.com/developers/events/gateway)
- Claude Code channels: the `claude/channel` capability, permission relay, and the warning that "An ungated channel is a prompt injection vector" — [Channels reference](https://code.claude.com/docs/en/channels-reference)

### Inferences (the recommendation)

**A. Components**
1. **`narwarpd` integration daemon** (a Rust module in the fork, or a Python sidecar), with:
   - a SQLite event store
   - per-source pollers or listeners with cursors
   - a `ticket_links` table (Narwarp ticket ↔ Linear issue / GitHub PR / email thread / Discord message)
   - an outbound **write journal**, for echo suppression and audit
   - a Keychain-backed credential vault
2. **Narwarp MCP gateway** (local stdio or HTTP on 127.0.0.1). It exposes *Narwarp-semantic* tools, for example:
   - `ticket.link_linear_issue`
   - `ticket.post_update`
   - `linear.create_issue_draft`
   - `email.search`
   - `email.create_draft`
   - `github.pr_status`
   - `discord.post_to_ticket_thread`

   Every tool carries a policy tier (section C). Both Claude Code sessions and Ollama agents (via an MCP-capable client) use the same gateway, so policy is enforced in one place.
3. **Narwarp channel server** (`claude/channel` + `claude/channel/permission`). It pushes ticket-board events and integration events, such as "PR #42 checks failed" or "Linear ENG-123 moved to In Review", into the relevant running Claude Code session. It relays that session's permission prompts to the Narwarp TUI or the user's Discord DM, gated on the user's identity.

**B. Which integrations go direct (SDK/HTTP) and which via MCP**

| Integration | Narwarp ingestion/sync (daemon) | Agent access | Real-time mechanism on laptop |
|---|---|---|---|
| **Linear** | Direct GraphQL with a personal API key (Read + Create issues + Create comments, team-restricted). Mode B (admin-installed actor=app OAuth app) if available | Narwarp gateway for writes. Optional official Linear MCP `/mcp/readonly` for context | Poll `issues(filter:{updatedAt:{gt:cursor}})` plus the notifications query. Mode B: webhooks via Funnel plus a poll reconcile |
| **GitHub** | `gh` CLI and REST/GraphQL with the `gh` token (or a classic PAT with `notifications`, `repo`) for notifications | Official GitHub MCP with a repo-scoped fine-grained PAT, narrow toolsets, `X-MCP-Lockdown`, read-only for most agents | Poll `/notifications` with `If-Modified-Since`, obeying `X-Poll-Interval` |
| **Gmail** | Direct Gmail API (`gmail.readonly` + `gmail.compose`) in a personal-use *production* OAuth app | Narwarp gateway only (search, summarize, `create_draft`). Optional official Gmail MCP (Developer Preview) | Pub/Sub pull, or `history.list` every 60 s, plus a daily `watch` renewal |
| **Fastmail / iCloud** | JMAP (API token) / IMAP IDLE (app-specific password) | Narwarp gateway only | JMAP EventSource / IMAP IDLE |
| **Discord** | A bot in a personal "Narwarp HQ" server, plus (if the admin agrees) a minimal read-only bot in the work server. A user-installed context-menu app as the no-admin fallback | Narwarp gateway (`post_to_ticket_thread`, `dm_user`). Avoid third-party Discord MCPs | Gateway WebSocket (outbound), RESUME on wake |

**Why the daemon rather than MCP for sync:** sync must be deterministic, idempotent and cheap, and it should run without an LLM in the loop. MCP calls are model-driven, unjournaled, and spend context tokens. Use vendor MCPs where their breadth helps exploratory agent work, and keep them read-only by default.

**C. Least-privilege and approval policy (tiers)**

| Tier | Examples | Policy |
|---|---|---|
| T0: read, own data | Search email, list Linear issues, PR status, read the Narwarp board | Auto-allow. Results from untrusted sources are tagged |
| T1: write to Narwarp-owned state | Create or update Narwarp tickets and board posts, add a Linear *attachment* backlink, create Gmail drafts, apply Gmail labels | Auto-allow, journaled, undoable |
| T2: externally visible, reversible | Create a Linear issue or comment, post in work Discord, comment on a GitHub PR/issue, push a branch | **User approval** in the TUI or via channel permission relay. The approval shows the final rendered text. Agent-authored text is marked as such |
| T3: irreversible or high impact | Send email, merge a PR, close or cancel Linear issues in bulk, delete anything, change repo or Discord settings | **Human performs it**, or gives a typed confirmation in the Narwarp UI. It is never exposed as an agent tool |

**Per-session capability isolation** (the lethal trifecta): a session that ingests untrusted content gets T0/T1 tools only, with no web fetch and no outbound messaging. That covers email bodies, Discord messages from others, and GitHub issues from non-collaborators. Planning or acting sessions receive only sanitized summaries.

**D. Secrets**
- Use one Keychain item per credential, via the existing `secure_storage`/`managed_secrets` crates: Linear API key (or OAuth refresh token), GitHub fine-grained PATs, Google OAuth refresh token, Discord bot tokens, the Fastmail token and the iCloud app password.
- The daemon refreshes short-lived tokens: Linear access tokens last 24 h, Google access tokens about 1 h.
- Tokens are injected into child processes, such as the Claude Code MCP config or the Channels plugin, at spawn time, never written to dotfiles.
- Rotate the iCloud app-specific password and the Discord token on any suspicion. Both are all-or-nothing credentials.

**E. Operational gotchas to design for**
- **Linear:**
  - Mode A attribution (writes appear as the user).
  - Each workspace needs a separate MCP auth context.
  - The HTTP 400 `RATELIMITED` error must be handled; it is not a 429.
- **Discord:**
  - The IDENTIFY cap; the invalid-request ban (10k per 10 min, so never retry 401/403 in a loop).
  - Message Content is needed in guilds. Without it, content is visible only for DMs, mentions and context-menu targets.
- **Gmail:**
  - Watch renewals.
  - Testing-mode 7-day tokens.
  - The silent revocation of the oldest of 100 refresh tokens.
  - Quota *billing* reportedly arriving later in 2026.
  - History 404 → full resync.
- **GitHub:**
  - The notifications API needs a classic PAT or the `gh` token, and fine-grained PATs can't call the Checks API. Plan two credentials.
  - Secondary limits on content creation (80/min).
- **Claude Code Channels:**
  - Research preview, an allowlist, `--dangerously-load-development-channels` for custom channels.
  - Events are lost if the session isn't running, so Narwarp should queue them and replay on session start.

### Gaps
- Not validated: whether running Claude Code sessions with a project-scoped `.mcp.json` pointing at the Narwarp gateway (instead of vendor MCPs) loses meaningful capability. That belongs to the Claude Code researcher.
- No measurements of real-world polling costs. The Linear complexity estimate is arithmetic from the published formula.
- Whether the user's employer permits Narwarp to send work Linear or Discord content to cloud LLMs at all is a policy question this research can't answer. The Ollama-local path may be required for work data.
