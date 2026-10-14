# Calendar time-blocking and auto-scheduling for Narwarp: products, algorithms, Google Calendar API (state as of 2026-10-13)

Conventions used below: [vendor] = the product's own docs or marketing; [competitor] = written by a competing vendor (usually SEO content, so biased); [practitioner] = user reviews or opinion; [possibly stale] = dated more than ~6 months before 2026-10-13, or an undated/legacy doc. GitHub star counts and dates were read from the GitHub search API on 2026-10-13. Everything under "Inferences" is my reasoning, not a sourced fact.

## 1. Which products matter in Oct 2026, how do they schedule, and what do users say works or fails?

### Takeaway
The category consolidated in 2025-26. Clockwise is gone: it shut down on 2026-03-27 after Salesforce hired its team. Motion has pointed its growth at SMB "AI Employees". Reclaim (now owned by Dropbox) shipped 2.0, the default for new sign-ups since May 2026, and 2.0 *stopped creating a calendar event per task*: it now protects Focus Time blocks, recommends tasks inside them, and stages changes in a Preview Mode. Morgen's AI Planner proposes a plan and waits for approval. Sunsama stays manual and warns when you over-commit. Google added native task time-blocking for all accounts (Nov 2025), but Focus Time is still limited to work and school accounts. The shared lesson: protect a few coarse blocks, preview changes, and let the human approve. Continuous per-task reshuffling is the behavior users complain about most.

### Cited Findings

**Clockwise: SHUT DOWN (2026-03-27)**
- Salesforce hired Clockwise's team for Agentforce ("Salesforce is acquiring the workers and not the company"). The service shut down on March 27, 2026. CEO Matt Martin confirmed that all customer data would be deleted and that Salesforce would not get access to the database. Unused subscriptions were refunded. Customers were pointed to Reclaim, which said it would match Clockwise's prices. Clockwise had been in market about 10 years, with customers including Uber, Netflix and Atlassian — [TechRadar via Yahoo Finance, 2026-03-23](https://finance.yahoo.com/sectors/technology/articles/salesforce-recuits-team-behind-calendar-103000806.html)
- Users were notified on March 20, about a week before shutdown — [Schedulee blog [competitor]](https://schedulee.com/blog/clockwise-shutting-down-best-alternatives-migration-guide/); [Cal.com blog [competitor]](https://cal.com/blog/clockwise-alternative). One outlet says Salesforce "acquired" the company in late 2025 — [thedailyclaws.com](https://thedailyclaws.com/blog/2026-03-20-news-clockwise-shutdown-salesforce/). The founders' acqui-hire framing in TechRadar contradicts this.
- Prepaid subscriptions running past March 27 reportedly got automatic prorated refunds — [usecarly [competitor]](https://www.usecarly.com/blog/clockwise-shut-down/)

**Reclaim.ai ("Reclaim.ai from Dropbox"): ACTIVE**
- Dropbox's acquisition was announced 2024-08-20. The whole 22-person team joined Dropbox, and terms were not disclosed — [Reclaim blog](https://reclaim.ai/blog/dropbox-acquires-reclaim); [TechCrunch 2024-08-22](https://techcrunch.com/2024/08/22/dropbox-acquires-index-ventures-backed-ai-scheduling-tool-reclaim-ai/). A "$40.2M" price appears only in secondary blogs, e.g. [temporal.day [competitor]](https://temporal.day/blog/is-reclaim-ai-shutting-down-2026), and is unverified.
- Reclaim 2.0: "If you signed up after May 2026, you're already using Reclaim 2.0." It adds:
  - a Planner and Reclaim Chat;
  - Preview Mode ("those updates can be staged first so you can see what will move before anything is applied to your live calendar");
  - Agents (Focus Time, Habits, Buffers, Smart Meetings, Meeting Quality) that "continuously manage different parts of your schedule based on your preferences";
  - Insights, including fragmented Focus Time;
  - MCP support, so AI tools can read calendar context and trigger scheduling.

  Source: [Reclaim help: Switching from 1.0 to 2.0 [vendor]](https://help.reclaim.ai/en/articles/17438753-switching-from-reclaim-1-0-to-2-0); [Reclaim 2.0 help collection (May 2026+)](https://help.reclaim.ai/en/collections/19647810-reclaim-2-0-help-docs-may-2026)
- Tasks in 2.0: Reclaim no longer creates a separate calendar event for each Task ("Instead of automatically creating a separate calendar event for every Task…"). It surfaces "Relevant Now" and "At Risk" tasks inside Focus Time and Habit blocks. You can assign a saved task search to a Focus Time or Habit Agent, or ask Chat to "Block off time for my Tasks next week" as general work blocks — [Reclaim help: 2.0 Tasks overview [vendor]](https://help.reclaim.ai/en/articles/16558552-reclaim-2-0-tasks-overview). A search-engine summary credited this article with recommending Reclaim 1.0 for per-task auto-scheduling. The fetched article did not contain that line, so that claim is unconfirmed.
- Habits (1.0 docs) start as Free and are marked Busy "as time comes closer to the Habit's start time, or as your day starts to fill up". Each Habit has a Time Defense setting: Most flexible (always free), Let Reclaim decide (default), or Most defensive (always busy, prefers max duration). Lower-priority Habits may be skipped for the week — [Reclaim help: time defense [vendor, possibly stale: 1.0 legacy article]](https://help.reclaim.ai/en/articles/4129290-how-time-defense-settings-work); [Reclaim help: Habits [vendor]](https://help.reclaim.ai/en/articles/12295315-stay-consistent-with-routines-using-habits)
- Buffers: "Decompression Time" can follow all meetings or only meetings with video links, and the title or just "Busy" is shown to others. Travel-time buffers also exist — [Reclaim updates [vendor]](https://updates.reclaim.ai/announcements/decompression-time-for-all-meetings); [Dropbox Learn: Reclaim buffers [vendor]](https://learn.dropbox.com/self-guided-learning/reclaim-fundamentals-course/reclaim-how-to-add-buffers-decompression-and-travel-time). An agent that adds recovery time after meetings with more than 8 attendees is marked "Coming Soon" — [Reclaim agent library [vendor]](https://reclaim.ai/ai-agent-library/add-recovery-time-for-large-meetings)
- User reviews:
  - Dialing in Habits and Task priorities takes "a significant amount of time (and trial and error)".
  - The lack of a native mobile app is the biggest drawback.
  - Flexible habits like lunch "stay flexible until my day gets busy, then it automatically locks them in so I actually get a break".

  Source: [Capterra reviews [practitioner]](https://www.capterra.ca/software/1058797/reclaim)

**Motion: ACTIVE; strategic focus moved to an SMB "agentic work suite"**
- On 2025-09-09 Motion announced $60M across Series B, C and C2, at a $550M valuation; Scale Venture Partners led the Series C. Motion says it is building "the first end-to-end agentic work suite". The AI Employees product reached eight-figure ARR within three months of launch, and Motion reports over 10,000 SMB customers — [Reworked [possibly stale]](https://www.reworked.co/the-wire/motion-raises-60m-at-550m-valuation-to-build-the-agentic-work-suite-for-businesses/). Agent work is metered in credits, e.g. Pro AI includes 7,500 credits per seat per month — [Sacra [analyst]](https://sacra.com/chat/h/aaa4dc90-884b-4b2b-bfac-9f13e8ab6539/)
- Scheduler rules as documented:
  - "Prioritizes deadlines and high-priority tasks first."
  - "Events from connected calendars always override task blocks."
  - "Splits long tasks into chunks if needed", respecting a minimum block size.
  - If a deadline is at risk, the task is pushed earlier or flagged.
  - A past-due task goes to "the next available slot".
  - Rescheduling around new work "can make some tasks shift multiple times".

  Source: [Motion help: How auto-scheduling works [vendor, undated]](https://www.usemotion.com/help/time-management/auto-scheduling/reference-auto-scheduling/how-auto-scheduling-works-behind-the-scenes)
- User sentiment in 2025-26 (aggregated by a competitor from Reddit, App Store and Trustpilot) is mixed to negative on pricing and support. Complaints include tasks scheduled on top of busy events and "supervising the scheduler defeats the purpose". Other users like that unfinished tasks keep getting rescheduled — [usecarly Motion review [competitor/practitioner]](https://www.usecarly.com/blog/motion-ai-review/). A reviewer who moved to Reclaim says Motion's fast rebuild is a real advantage, but "constant reshuffling can be stressful" — [Lindy [competitor]](https://wf.lindy.ai/blog/motion-vs-reclaim)
- 2026 prices conflict across reviews, from $19 to $49 a month — [Ellie [competitor]](https://ellieplanner.com/comparisons/motion-app-review); unresolved.

**Morgen AI Planner: ACTIVE**
- Inputs:
  - tasks, native or imported from Notion, Todoist, ClickUp and others, with duration tags like [30m];
  - Frames, which are templates for when each type of work happens;
  - the Morgen Priority Factor ("After a long period of being overdue, tasks will not be prioritized");
  - preferences: break frequency ("plan intensity"), default duration, an optional 20% duration buffer (justified by the planning fallacy), and splitting long tasks.
- Output and approval:
  - Proposed tasks appear as preview events that "pulsate over Frames". "Schedule all" commits them and Esc discards.
  - Plans cover at most 8 days.
  - Yesterday's unfinished tasks trigger a prompt (reschedule ASAP, rearrange, or unschedule), and reschedules need approval.
  - Time-sensitive tasks that don't fit are listed for manual review.
  - The guide does not mention energy.

  Source: [Morgen guide, 2026-02-20 [vendor, possibly stale]](https://www.morgen.so/guides/plan-your-day-using-the-ai-planner)
- Nothing syncs to Google, Outlook or iCloud until the plan is approved — [Morgen AI Planner page [vendor]](https://www.morgen.so/ai-planner)

**Sunsama: ACTIVE, manual-first**
- Sunsama adds up the planned time of work tasks and warns when it exceeds a workload threshold the user sets. A timeline shows estimated completion time against the preferred shutdown time. Overloaded tasks can be deferred (D) or sent to the backlog (Z). Timeboxing onto the calendar is optional, with two-way Google and Outlook sync — [Sunsama help: Daily planning [vendor]](https://help.sunsama.com/docs/daily-planning)
- "Minimal AI features; planning remains largely manual" — [Amie comparison [competitor]](https://amie.so/alternatives/sunsama)

**Akiflow: ACTIVE**
- The Aki assistant launched on Product Hunt in Nov 2024 and as an in-app beta in March 2025 — [Product Hunt](https://www.producthunt.com/products/akiflow/launches/aki-2); [Akiflow changelog [vendor, possibly stale]](https://akiflow.featurebase.app/en/changelog/aki-in-app-sparkles-your-personal-ai-executive-assistant-beta). Search summaries report a 2026-06-30 Product Hunt launch of an Akiflow MCP connector for Claude, ChatGPT and Cursor. I did not fetch the launch page — [Product Hunt Akiflow launches](https://www.producthunt.com/products/akiflow/launches/akiflow-cdd5f624-4fff-4b75-b885-0e91aa60fcaf)
- One reviewer says you still place and move time blocks yourself. A "Schedule Optimizer" replans scheduled tasks — [usecarly Akiflow review [competitor]](https://www.usecarly.com/blog/akiflow-review/)

**Amie: ACTIVE; its center of gravity is now AI meeting notes**
- Changelog highlights:
  - MCP server (2026-07-14);
  - two-way Outlook sync (2026-08-05);
  - co-host scheduling links (2026-08-20);
  - expanded MCP tools (2026-08-27);
  - a system-audio recorder (2026-09-03).

  Todos with a priority or deadline have been auto-scheduled since 2024-05-07, and an "ai scheduling fix" on 2025-05-14 addressed AI-scheduled todos that overlapped — [Amie changelog [vendor]](https://amie.so/changelog)
- A competitor says Amie dropped its free plan and no longer sells its calendar-and-todo product — [nocal.app, updated 2026-08-04 [competitor]](https://nocal.app/alternatives/amie). Amie's continued calendar releases above partly contradict this.

**Notion Calendar: ACTIVE, manual time-blocking**
- You drag tasks from a Notion database onto time slots. Notion does not automatically block time for tasks — [Notion help [vendor]](https://www.notion.com/help/guides/getting-started-with-notion-calendar); [Akiflow blog [competitor]](https://akiflow.com/blog/notion-calendar-integration-smarter-scheduling)
- Reported April 2026 changes: Notion Agent schedules meetings from chat (Apr 15), and a "Calendar Optimizer" Custom Agent template detects conflicts and protects focus blocks. The template looks for 30-minute to 2-hour gaps and does not learn when the user's focus actually happens — [temporal.day [competitor]](https://temporal.day/blog/notion-agent-vs-ai-calendars-2026). Not checked against Notion's release notes.

**Google Calendar native features (incl. Gemini)**
- Focus Time: "To schedule a focus time event, you must have a work or school account". Eligible editions are Business Standard/Plus, Enterprise Standard/Plus, Education and Nonprofits. It mutes Chat and auto-declines meetings ("new and existing meetings" by default), and events must have specific start and end times — [Google Calendar Help [vendor]](https://support.google.com/calendar/answer/11190973)
- Task time-blocking: select an empty slot, click "task", then set visibility and do-not-disturb. The task stays on the task list with reminders until it is done. It is "Available to all Google Workspace customers, Workspace Individual Subscribers, and users with personal Google accounts", rolled out from 2025-11-06 (Rapid Release) and 2025-12-01 (Scheduled Release), has no admin controls, and the post mentions no API — [Workspace Updates 2025-11-17 [vendor, possibly stale]](https://workspaceupdates.googleblog.com/2025/11/block-time-for-tasks-google-calendar.html)
- The Google Tasks API cannot carry a time: "Only date information is recorded; the time portion of the timestamp is discarded"; "It isn't possible to read or write the time that a task is scheduled for using the API" — [Tasks API reference](https://developers.google.com/workspace/tasks/reference/rest/v1/tasks)
- Gemini "Suggested times" recommends meeting slots from invitees' availability, time zones and working hours, where the organizer can see those calendars. It also shows a reschedule banner when guests decline. Scheduled Release domains got it from 2026-02-02. It is limited to paid Workspace Business/Enterprise Standard and Plus plans and the AI Pro for Education add-on, is not available on mobile, and does not work for events longer than 8 hours — [Workspace Updates 2026-01 [vendor, possibly stale]](https://workspaceupdates.googleblog.com/2026/01/improved-meeting-suggestions-gemini-calendar.html)
- Other Gemini features: a Gemini side panel in Calendar since 2025-03 — [TechCrunch](https://techcrunch.com/2025/03/07/google-adds-a-gemini-panel-to-calendar-to-help-you-manage-your-schedule/); Gmail "Help me schedule" since 2025-10 — [AlternativeTo](https://alternativeto.net/news/2025/10/google-adds-gemini-powered-scheduling-feature-to-gmail-for-easier-meeting-organization/)

**SkedPal: ACTIVE**
- Time Maps set preferred hours for each category of work (e.g. Working Hours, Evenings). Several maps on one task intersect by default, with a toggle to use their union instead. An empty map holds tasks back until hours are drawn with the "black pen", which overrides other preferences — [SkedPal docs: assigning time maps [vendor]](https://docs.skedpal.com/time-maps/assigning-time-maps); [SkedPal docs: black pen [vendor]](https://docs.skedpal.com/calendar/managing-contexts-with-the-time-map-black-pen)
- Capterra rating is 4.5 from 171 reviews. One reviewer says the phone app still feels rough. Prices conflict across listings — [Capterra [practitioner]](https://www.capterra.com/reviews/172391/SkedPal)

**Trevor AI: ACTIVE**
- You drag a task onto the calendar to create a time block. It offers AI time suggestions and "Plan My Day", and connects Google, Outlook and iCloud calendars plus Todoist, Microsoft To Do and Google Tasks — [Trevor AI site [vendor]](https://trevorai.com/); [Trevor docs [vendor]](https://trevorlabs.notion.site/Trevor-AI-Documentation-e6379ce2b7bb4362a0193e1987b609fc). A July 2026 review reports Pro at about $5/month billed annually and says "Trevor plans your day. It doesn't work it." — [usecarly [competitor]](https://www.usecarly.com/blog/trevor-ai/)

**Energy-aware entrant: Lifestack**
- Lifestack uses wearable data (Apple Watch, Oura, WHOOP: sleep and HRV) to put deep work at predicted energy peaks and to schedule breaks. Lifestack 2.0 was announced 2025-09-24 — [press release [vendor, possibly stale]](https://insider.fitt.co/press-release/use-wearable-data-for-smarter-daily-planning-lifestack-2-0-launches/); [G2 listing](https://www.g2.com/products/lifestack/discuss). I found no independent check of its accuracy.

**2026 trend: calendars and planners exposing MCP to agents**
- Examples: Reclaim 2.0's MCP support ([Reclaim help](https://help.reclaim.ai/en/articles/17438753-switching-from-reclaim-1-0-to-2-0)); Amie's MCP server ([changelog](https://amie.so/changelog)); Akiflow's MCP connector (above); open-source open-sunsama with a one-click MCP connector ([GitHub](https://github.com/ShadowWalker2014/open-sunsama)); keeper.sh's calendar MCP ([GitHub](https://github.com/ridafkih/keeper.sh)).

### Inferences
- Reclaim moved from one event per task (1.0) to Focus Time containers plus recommendations (2.0), and added Preview Mode. That is the strongest public signal that placing every task on the live calendar caused churn and distrust. Narwarp should start with coarse "work blocks" per ticket or project and a preview-then-approve loop.
- Clockwise shut down within a week and deleted user data. That argues for keeping Narwarp's plan local and treating Google Calendar as a place the plan is written to, not the source of truth.
- Per their own docs, almost none of the mainstream tools model energy, apart from Lifestack (vendor claims) and FluidCalendar's "energy preferences". A simple user-declared energy curve is a cheap way to stand out.
- Gemini scheduling is paid-Workspace-only, and Focus Time needs a work or school account. So on the personal Gmail side, Narwarp can only use plain events it writes itself.
- Narwarp could drive a vendor such as Reclaim over MCP instead of building a scheduler. But Clockwise's shutdown and Reclaim 2.0's dropping of per-task scheduling show that vendors change the model underneath you.

### Gaps
- Current official pricing for Motion, Reclaim, Sunsama and Akiflow was not checked on vendor pages.
- I retrieved no primary Reddit or HN threads. User sentiment here comes through review sites and competitor blogs, which are biased.
- Notion's April 2026 features and Akiflow's June 2026 MCP launch come only from secondary summaries.
- No vendor publishes how often its scheduler moves blocks, or how much user churn re-plan thrash causes.

## 2. What open-source tools and algorithms exist, and which fits single-user scale (tens of tasks, ~2-week horizon)?

### Takeaway
There is no mature open-source Reclaim or Motion:
- FluidCalendar (MIT, about 1k stars) says it is "not yet recommended for production use".
- taskcheck is a small Taskwarrior scheduler.
- The calendar TUIs (calcure, khal, gcalcli) view and edit calendars; they don't schedule.

At Narwarp's scale, a deterministic greedy placer (earliest-due-date / least-slack ordering) is enough for real-time repairs. OR-Tools CP-SAT is the right optional optimizer: it is pip-installable and has interval variables, a no-overlap constraint, and solution hints. Timefold documents pinning and change-penalized replanning most clearly. But its Python package is a pre-release and "significantly slower" than its Java version, so copy its concepts rather than adopt it.

### Cited Findings
- FluidCalendar: "an open-source alternative to Motion, designed for intelligent task scheduling and calendar management". MIT license, about 998 stars and 73 forks. Stack: Next.js 15, Prisma, PostgreSQL 15+. Google and Outlook are supported, and CalDAV is listed. The README says "This project is in active development and currently contains many bugs and incomplete features" and "It is not yet recommended for production use." — [GitHub](https://github.com/dotnetfactory/fluid-calendar). The project blog says it respects work hours, buffers, project grouping and "schedule locks", and weighs deadlines, priority and energy preferences — [FluidCalendar blog [vendor]](https://blog.fluidcalendar.com/fluidcalendar-open-source-intelligent-task-scheduling/)
- taskcheck (by 00sapo; a fork lives at linuxcaffe) describes itself as "A non-AI automatic scheduler for taskwarrior (i.e. alternative to skedpal/timehero/flowsavvy/reclaim/trevor/motion)". How it works:
  - It uses named time maps for working hours.
  - It allocates time day by day in urgency order, recomputing urgencies for each day, and handles dependencies.
  - Each allocation attempt is capped at 2 h by default (tunable with the `min_block` setting).
  - To meet due dates it auto-fixes: if a task would finish late, "the weight_urgency factor is reduced by 0.1 and the scheduling is repeated".
  - It blocks time using iCal URLs, has a dry-run mode, and exports to iCal or Google.

  Source: [GitHub (fork of 00sapo/taskcheck)](https://github.com/linuxcaffe/taskcheck)
- Smaller OSS projects:
  - open-sunsama: kanban, time blocking and MCP; about 96 stars; created 2026-01-29 — [GitHub](https://github.com/ShadowWalker2014/open-sunsama)
  - Planly: single-file time-blocking PWA, 2026-09 — [GitHub](https://github.com/drkokorev/Planly)
  - nexus-taskscheduler: about 10 stars — [GitHub](https://github.com/parkerdavis-cell/nexus-taskscheduler)
  - An n8n workflow that auto-schedules TickTick tasks into Google Calendar free slots — [GitHub](https://github.com/szatzger/n8n-ticktick-scheduler)
  - A FlowSavvy-to-Linear bridge that auto-schedules Linear "My Issues" — [GitHub](https://github.com/ChrisvanChip/flowsavvy-linear)
- Classic scheduling results:
  - Earliest-due-date order (Jackson's rule) minimizes maximum lateness on one machine when all jobs are available at t=0.
  - Moore-Hodgson minimizes the number of late jobs in O(n log n).
  - With release dates the problem (1|r_j|L_max) becomes strongly NP-hard.

  Sources: [NYU Stern scheduling handout](https://web-static.stern.nyu.edu/om/faculty/pinedo/scheduling/shakhlevich/handout04.pdf); [Berkeley IEOR](https://riot.ieor.berkeley.edu/Applications/Scheduling/algorithms.html); [arXiv 2607.16347](https://arxiv.org/pdf/2607.16347)
- CP-SAT intervals: intervals are "constraints containing three integer variables (start, size, and end)", and NoOverlap and cumulative constraints model scheduling — [OR-Tools scheduling recipes (mirror of google/or-tools)](https://github.com/farao-community/or-tools/blob/stable/ortools/sat/doc/scheduling.md); [UPenn CIS 1921 slides](https://cis.upenn.edu/~cis1921/files25s/lecture8/S25-lec8-slides.pdf). The CP-SAT Primer (CC-BY 4.0) covers `new_interval_var`, `add_no_overlap`, `add_cumulative` and Large Neighborhood Search — [project summary](https://www.sourcepulse.org/projects/2335957)
- CP-SAT solution hints: "A solution hint is not a hard constraint"; "It's OK to have an infeasible solution hint"; "partial solution hints are fine"; Python uses `model.add_hint(x, 1)` — [OR-Tools CP-SAT model docs](https://github.com/google/or-tools/blob/stable/ortools/sat/docs/model.md)
- Timefold (docs version 2.7.1):
  - Pinning (`@PlanningPin` / `@PlanningPinToIndex`) is for assignments that "must not move under any circumstances".
  - Non-disruptive replanning means "allow changes, but make the solver justify them", and "is usually implemented by taxing all planning entities that change". The example applies a soft penalty of 1000 when `timeslot != publishedTimeslot`.

  Source: [Timefold docs: non-disruptive replanning](https://docs.timefold.ai/timefold-solver/latest/responding-to-change/non-disruptive-replanning)
- Timefold real-time planning: external changes are submitted as a `ProblemChange`, and the solver "Restarts from the adjusted best solution (warm start — much faster than a cold start)". In daemon mode it idles until the next change — [Timefold docs: real-time planning](https://docs.timefold.ai/timefold-solver/latest/responding-to-change/real-time-planning). Timefold also treats pinning as hard and offers a soft "Minimize disruption on replanning" constraint — [Timefold Field Service Routing docs](https://docs.timefold.ai/field-service-routing/latest/real-time-planning/real-time-planning-minimize-disruption)
- Timefold for Python: the `timefold-solver` PyPI release was yanked ("Use the timefold package instead"). The `timefold` package shows a 1.24.0b0 pre-release, and Python use is "significantly slower than using Timefold Solver for Java or Kotlin" — [PyPI timefold](https://pypi.org/project/timefold/); [PyPI timefold-solver](https://pypi.org/project/timefold-solver); [Timefold blog](https://timefold.ai/blog/introducing-timefold-python-solver)

### Inferences
- Problem size: 14 days at about 40 quarter-hours of work time a day is roughly 560 slots, holding 20-60 ticket chunks. That is a tiny CP-SAT model; I'd expect solve times from under a second to a few seconds, but I haven't benchmarked it.
- Recommended two tiers:
  - (1) A deterministic greedy repairer that is always on: fast and easy to explain, used for every real-time change.
  - (2) CP-SAT for the morning and weekly plans, with a change-penalty term, seeded with the current plan as a hint.
- Timefold itself isn't needed; its pinning and change-tax ideas map directly onto CP-SAT constraints and objective terms.
- taskcheck's repair loop (lower the urgency weight and re-run until due dates are met) is a pragmatic heuristic worth copying for deadline triage.
- Since the user writes Python and Narwarp's core is Rust, a Python scheduler sidecar using `ortools` is the lowest-friction path. Rust-native solver options were not researched.

### Gaps
- No public benchmarks of CP-SAT on personal-calendar-sized models.
- Rust-native optimization libraries were not evaluated.
- No OSS project implements Reclaim-style free-to-busy "time defense".

## 3. How should re-planning work without thrash (minimal perturbation, pinning, hysteresis, moving meetings, overruns)?

### Takeaway
Thrash is a documented failure mode: Motion's own docs say tasks can "shift multiple times". The proven mitigations are:
- hard pins for in-progress and user-edited items;
- a soft "change tax" so every move must pay for itself (Timefold);
- staged previews (Reclaim 2.0, Morgen);
- soft holds that harden as their time approaches (Reclaim Habits go Free → Busy);
- coarse container blocks rather than one event per task (Reclaim 2.0);
- optimistic concurrency against the user's own edits (Google etags with If-Match).

Estimates overrun as a rule (the planning fallacy), so every plan needs slack.

### Cited Findings
- Motion: "Events from connected calendars always override task blocks", and rescheduling "can make some tasks shift multiple times" — [Motion help [vendor]](https://www.usemotion.com/help/time-management/auto-scheduling/reference-auto-scheduling/how-auto-scheduling-works-behind-the-scenes)
- Reclaim 2.0 stages changes in Preview Mode before they reach the live calendar — [Reclaim help](https://help.reclaim.ai/en/articles/17438753-switching-from-reclaim-1-0-to-2-0). It no longer creates individual Task events — [Reclaim help](https://help.reclaim.ai/en/articles/16558552-reclaim-2-0-tasks-overview)
- Reclaim Habits start Free and turn Busy as the start time nears or the day fills. The decision weighs the time left in the Habit's window and the time left before it is due to start — [Reclaim help [vendor, possibly stale]](https://help.reclaim.ai/en/articles/4129290-how-time-defense-settings-work)
- Morgen: reschedules require approval; ignoring a task frees its slot; dragging a task into a Frame reprioritizes it and shuffles the others — [Morgen guide](https://www.morgen.so/guides/plan-your-day-using-the-ai-planner)
- Timefold's change tax: "a solution will only be accepted if it improves the soft score for at least 1000 points per variable changed (or if it improves the hard score)" — [Timefold docs](https://docs.timefold.ai/timefold-solver/latest/responding-to-change/non-disruptive-replanning). Pin the part of a plan that has already started and plan around it — [Timefold blog: continuous planning with pinning](https://timefold.ai/blog/continuous-planning-optimization-with-pinning)
- CP-SAT hints can warm-start the solver from the previous plan, and need not be complete or feasible — [OR-Tools docs](https://github.com/google/or-tools/blob/stable/ortools/sat/docs/model.md)
- Google returns 412 Precondition Failed when "The etag supplied in the If-match header no longer corresponds to the current etag of the resource"; the fix is to re-fetch and re-apply — [Calendar API errors](https://developers.google.com/workspace/calendar/api/guides/errors). Push notifications "are not 100% reliable" — [Calendar push guide](https://developers.google.com/workspace/calendar/api/guides/push)
- Planning fallacy: predicted completion times were too optimistic across academic and non-academic tasks. Honours-thesis students' average best guess was 33.9 days against 55.5 days actual, and about 30% finished by their predicted date (figures from a secondary account). In one study, linking forecasts to past experience removed the bias — [Buehler, Griffin & Ross 1994 (PDF)](https://bear.warrington.ufl.edu/brenner/mar7588/Papers/buehler-et-al-1994.pdf); [SPSP blog](https://spsp.org/news-center/blog/buehler-planning-fallacy). Morgen offers an optional 20% duration buffer for this reason — [Morgen guide](https://www.morgen.so/guides/plan-your-day-using-the-ai-planner)
- taskcheck's repair loop lowers the urgency weight by 0.1 and repeats until due dates are met — [GitHub](https://github.com/linuxcaffe/taskcheck)

### Inferences
- Policy, detailed in section 8:
  - a frozen near-term window;
  - repair only the affected blocks when an event fires, and run a full re-solve only at planning rituals;
  - a change tax that grows the sooner a block starts and if the user has already seen it;
  - move budgets per block and per day;
  - a 2-5 minute debounce for bursts of calendar edits;
  - far-future blocks kept local-only or written as non-blocking (transparent) holds;
  - any user edit (etag mismatch, moved, resized, deleted) treated as a pin or an explicit "not now".
- Overruns: at the end of a block, prompt "extend / reschedule remainder / done". Learn each project's actual-to-estimate ratio and inflate future estimates by it. This is the "use past experience" de-biasing from Buehler et al.
- When a meeting moves onto a block: repair locally by shifting the displaced block within the same day if possible, and never let it cascade beyond the move budget without asking.

### Gaps
- I found no research on how often a personal scheduler can re-plan before users lose trust. The thresholds in section 8 are design defaults, not evidence-based numbers.

## 4. What does evidence say about breaks, post-meeting recovery and focus-block length, and how do products implement buffers and "habits"?

### Takeaway
Micro-breaks reliably but modestly improve vigor and fatigue. Their performance benefit is small and depends on the task, and longer breaks help more after demanding work. The case that back-to-back meetings need recovery is real but thin: one 14-person vendor EEG study and some industry surveys. Autonomy over what you do in a break matters (lunch-break study). The "90-minute" and "4-hour" focus rules are extrapolations, not established findings. Products implement recovery as buffers after meetings (Reclaim), break-frequency sliders (Morgen), and flexible "habits" that harden as the day fills (Reclaim).

### Cited Findings
- Albulescu et al. 2022 (PLOS ONE), a meta-analysis of micro-breaks (10 minutes or less): 19 records, 22 samples, N=2,335.
  - Vigor d=.36 and fatigue d=.35, both significant but small.
  - Performance d=.16, p=.116, not significant overall; significant only for less cognitively demanding tasks.
  - Longer breaks produced a larger performance boost, and highly depleting tasks may need breaks longer than 10 minutes.

  Sources: [DOAJ record](https://doaj.org/article/c6f37b6ba159469296b2cb968b65505f); [ScienceDaily](https://www.sciencedaily.com/releases/2022/08/220831152754.htm)
- Microsoft Human Factors Lab (2021, n=14, EEG): in four back-to-back 30-minute video meetings, beta-wave (stress) activity built up. With 10-minute meditation breaks between them it stayed stable. Transitions between meetings caused stress spikes, and engagement (frontal alpha asymmetry) stayed positive with breaks. It is small and not peer-reviewed — [Microsoft WorkLab [vendor research, possibly stale]](https://www.microsoft.com/en-us/worklab/work-trend-index/brain-research)
- Biwer et al. 2023 (Br J Educ Psychol, n=87 students) compared systematic breaks (24 min on / 6 off "Pomodoro", and 12/3) with self-regulated breaks. Self-regulators took longer sessions and breaks, with more fatigue and less concentration and motivation; task completion did not differ — [Maastricht University record](https://cris.maastrichtuniversity.nl/en/publications/understanding-effort-regulation-comparing-pomodoro-breaks-and-sel). A 2025 study found Pomodoro breaks raised fatigue faster, with no overall difference in fatigue or productivity — [PMC 2025](https://pmc.ncbi.nlm.nih.gov/articles/PMC12292963/). The two conflict.
- Trougakos et al. 2014 (Academy of Management Journal): how much autonomy people had over their lunch break moderated how its activities affected end-of-day fatigue — [PolyU research portal](https://research.polyu.edu.hk/en/publications/lunch-breaks-unpacked-the-role-of-autonomy-as-a-moderator-of-reco/). "A critical element was having the freedom to choose whether to do it or not" — [ScienceDaily 2013](https://sciencedaily.com/releases/2013/10/131002125507.htm)
- Wendsche & Lohmann-Haislah 2017, a meta-analysis of 86 publications, 91 samples and 38,124 employees: mentally detaching from work went with better mental and physical health, well-being and task performance (small-to-medium effects). The studies are correlational — [Frontiers in Psychology](https://frontiersin.org/articles/10.3389/fpsyg.2016.02072/full)
- "Meeting recovery syndrome" comes from industry surveys (Rogelberg with Mentimeter, 2019): 54% of UK respondents said a poorly run meeting hurt their productivity afterwards — [Workplace Insight](https://workplaceinsight.net/bad-meetings-lead-to-problems-away-from-the-meeting-itself/); [PCMA](https://www.pcma.org/meeting-recovery-syndrome-prevents-best-work/)
- Attention residue (Leroy 2009): switching away before finishing leaves attention on the previous task and hurts performance on the next — [UW Bothell](https://www.uwb.edu/news/2024/05/17/an-ongoing-study-on-the-success-of-staying-focused)
- Focus-duration folklore: Ericsson et al. (1993) cite training studies showing "essentially no benefit from durations exceeding 4 hr per day and reduced benefits from practice exceeding 2 hr". Critics say this summary over-generalizes its sources, and it is often re-cited through *Deep Work* — [Aceso Under Glass "epistemic spot check" [practitioner]](https://acesounderglass.com/2019/06/21/epistemic-spot-check-the-role-of-deliberate-practice-in-the-acquisition-of-expert-performance/)
- How products implement recovery:
  - Reclaim: Decompression Time after meetings (all, or video-only), plus travel buffers — [Reclaim updates](https://updates.reclaim.ai/announcements/decompression-time-for-all-meetings). Habits such as lunch stay flexible, then lock in — [Reclaim help](https://help.reclaim.ai/en/articles/12295315-stay-consistent-with-routines-using-habits)
  - Morgen: a break-frequency setting and an optional 20% buffer — [Morgen guide](https://www.morgen.so/guides/plan-your-day-using-the-ai-planner)
  - Motion: "Respects breaks, working schedules, and external events" — [Motion help](https://www.usemotion.com/help/time-management/auto-scheduling/reference-auto-scheduling/how-auto-scheduling-works-behind-the-scenes)
  - Sunsama: a shutdown time and a workload threshold — [Sunsama help](https://help.sunsama.com/docs/daily-planning)
  - Microsoft: added Outlook settings that make it easier to schedule breaks between meetings — [Microsoft WorkLab](https://www.microsoft.com/en-us/worklab/work-trend-index/brain-research)

### Inferences
- Evidence strength, strongest first:
  - moderate: micro-breaks improve well-being;
  - low to moderate: detachment and recovery correlate with better outcomes;
  - low: specifics of recovery after meetings;
  - very low: specific block lengths such as "90 minutes" or "4 hours a day".
- How Narwarp should implement it:
  - Post-meeting decompression of 10-15 minutes as real calendar blocks.
  - Micro-breaks inside focus blocks as TUI timers or nudges, not calendar events, to avoid clutter.
  - Free time as the most-protected block class, which only the user can trade away.
  - Lunch as a flexible window that hardens as the day fills.
  - Break content chosen by the user and kept out of agents' control (the autonomy finding).
- Defaults, all configurable and labeled as heuristics: a deep-work cap of 4 h/day, focus chunks of 60-120 minutes, a break of at least 10 minutes after 90 minutes of focus, and an end-of-day hard stop.
- To keep free time from being eaten: make it opaque and Busy on the primary calendar, rank it above every ticket except a hard-deadline P0, require explicit user consent to trade it, and track a "free-time debt" that must be repaid within N days.

### Gaps
- I found no meta-analysis of focus-block length for knowledge work. I did not retrieve the peer-reviewed meeting-recovery paper (Allen et al.).
- No evidence compares how well scheduled versus unscheduled free time is protected.

## 5. Google Calendar API mechanics: event types, freeBusy, tagging, sync, push, scopes, quotas, the 7-day refresh-token problem, MCP servers

### Takeaway
Everything Narwarp needs is in Calendar API v3, using an OAuth client the user creates themselves:
- `events.insert` with IDs Narwarp generates, and private `extendedProperties` to tag its own blocks;
- `colorId`, etag/If-Match for concurrency, `syncToken` incremental sync, and `freeBusy`.

The special "status" event types (focusTime, outOfOffice, workingLocation) can be written through the API, but only on the primary calendar of eligible accounts, and Focus Time needs a work or school account. So the personal Gmail side falls back to ordinary opaque events. Push notifications need a public HTTPS endpoint; on a laptop, polling with `syncToken` (or Tailscale Funnel) is simpler. The 7-day refresh-token expiry only applies while the OAuth app is in "Testing"; moving the personal project to "In production" without verification avoids it, at the cost of an "unverified app" warning and a 100-user cap that doesn't matter for one user. Google runs an official remote Calendar MCP server, but it is in Developer Preview. nspady/google-calendar-mcp is the most-starred dedicated community server, and Google's googleworkspace/cli (a Rust CLI, explicitly not an officially supported Google product) is a terminal-native alternative.

### Cited Findings

**Status event types**
- Status events are created with `events.insert`:
  - focusTime needs `focusTimeProperties` and `transparency: opaque`;
  - outOfOffice needs `outOfOfficeProperties` and opaque;
  - workingLocation needs `workingLocationProperties`, `visibility: public` and `transparency: transparent`.
- Restrictions:
  - "Focus times cannot be all-day events"; "Out of office events cannot be all-day events".
  - "These features are only available on primary calendars, and to some Google Calendar users"; "Secondary calendars can't have status events".
  - Working location events can't be created or updated through the batch endpoints.
- `events.list` and `events.watch` accept an `eventTypes` filter.

  Sources: [Calendar status-events guide](https://developers.google.com/workspace/calendar/api/guides/calendar-status); [working hours & location guide](https://developers.google.com/workspace/calendar/api/guides/working-hours-and-location)
- `eventType` values are default, birthday, focusTime, fromGmail, outOfOffice and workingLocation. eventType "cannot be modified after the event is created"; fromGmail events "cannot be created".
- `focusTimeProperties.autoDeclineMode` is one of declineNone, declineAllConflictingInvitations, or declineOnlyNewConflictingInvitations.
- `chatStatus` (available / doNotDisturb) and `declineMessage` are not marked writable in the reference, yet the guide's sample sets them. The docs are ambiguous here, so test before relying on them.

  Source: [Events resource reference](https://developers.google.com/workspace/calendar/api/v3/reference/events)
- Focus Time needs a work or school account — [Google Calendar Help](https://support.google.com/calendar/answer/11190973). Combined with the "primary calendars … some users" note above, a consumer Gmail calendar cannot hold focusTime events.
- Working hours are not exposed through the API. The 12 user settings are autoAddHangouts, dateFieldOrder, defaultEventLength, format24HourTime, hideInvitations, hideWeekends, locale, remindOnRespondedEventsOnly, showDeclinedEvents, timezone, useKeyboardShortcuts and weekStart, with no working-hours entry — [Settings reference](https://developers.google.com/workspace/calendar/api/v3/reference/settings)

**Tagging harness-created blocks, idempotency, concurrency**
- Extended properties come in two kinds:
  - private: specific to one calendar's copy of the event;
  - shared: "visible and editable by all attendees of an event".
- Limits:
  - keys over 44 characters are "silently dropped";
  - values over 1024 characters are "silently truncated";
  - at most 300 properties and 32 kB per event, counted across all copies.
- Querying: `events.list?privateExtendedProperty=name%3Dvalue`. Repeating the same parameter ORs the conditions; combining private and shared parameters ANDs them.
- An update deletes any property the request leaves out, so use patch.

  Source: [Extended properties guide](https://developers.google.com/workspace/calendar/api/guides/extended-properties)
- `colorId` refers to an entry in the `event` section of the colors definition.
- `source.title` and `source.url` can only be seen or modified by the event's creator; the URL must be http or https.
- `transparency` is opaque (blocks time, the default) or transparent. `visibility` can be private ("Event details are visible only to attendees").

  Source: [Events resource reference](https://developers.google.com/workspace/calendar/api/v3/reference/events)
- Client-specified event IDs:
  - "characters allowed in the ID are those used in base32hex encoding, i.e. lowercase letters a-v and digits 0-9";
  - "the length of the ID must be between 5 and 1024 characters";
  - "the ID must be unique per calendar", but "we cannot guarantee that ID collisions will be detected at event creation time", so use an RFC 4122 UUID.
- `sendUpdates` defaults to false.
- Insert accepts any of the scopes calendar, calendar.events, calendar.app.created or calendar.events.owned.

  Source: [events.insert reference](https://developers.google.com/workspace/calendar/api/v3/reference/events/insert)
- Errors that matter for a scheduler:
  - 409 "The requested identifier already exists";
  - 412 etag mismatch: re-fetch and re-apply;
  - 403 "Calendar usage limits exceeded";
  - 403 rateLimitExceeded and 429: back off exponentially;
  - 410 "Sync token is no longer valid, a full sync is required".

  Source: [Calendar API errors](https://developers.google.com/workspace/calendar/api/guides/errors)

**Availability (freeBusy)**
- Request fields are `timeMin`, `timeMax`, `timeZone` (default UTC), `items[].id`, `calendarExpansionMax` ("Maximum value is 50") and `groupExpansionMax` ("Maximum value is 100").
- The response gives `calendars.{id}.busy[]` with an inclusive start and exclusive end, plus per-calendar errors (groupTooBig, tooManyCalendarsRequested, notFound, internalError).
- Accepted scopes: calendar.readonly, calendar, calendar.events.freebusy, calendar.freebusy.
- No maximum time range is documented.

  Source: [freeBusy.query reference](https://developers.google.com/workspace/calendar/api/v3/reference/freebusy/query)

**Incremental sync**
- How it works:
  - Do a full sync first and store `nextSyncToken`, which "is present only on the very last page".
  - "Each list request should use the same set of query parameters, including the initial request."
  - Disallowed restrictions return 400, and "Sync tokens aren't compatible with most filters".
  - Results "will always contain deleted entries".
  - A 410 response "should trigger a full wipe of the client's store and a new full sync".
  - The older `modifiedSince` approach is "no longer recommended".

  Source: [Sync guide](https://developers.google.com/workspace/calendar/api/guides/sync)

**Push notifications vs a laptop**
- Requirements and behavior:
  - The webhook "must use HTTPS" with a valid SSL certificate; self-signed, untrusted, revoked or hostname-mismatched certificates fail.
  - Notifications have no body, only `X-Goog-*` headers (resource state sync, exists or not_exists), so the app must call the API to learn what changed.
  - Events are watched per calendar.
  - "there's no automatic way to renew a notification channel".
  - "Notifications are not 100% reliable."

  Source: [Push guide](https://developers.google.com/workspace/calendar/api/guides/push)
- The default channel TTL is 604800 seconds (7 days) — [events.watch reference](https://developers.google.com/workspace/calendar/api/v3/reference/events/watch). No official maximum was found; a third-party blog's "max ~30 days" uses a different unit and isn't backed by the official docs — [codewords.ai](https://codewords.ai/blog/google-calendar-webhooks)
- Tailscale Funnel publishes a local port at a stable `*.ts.net` HTTPS hostname through Tailscale relays, and lists webhook testing as a use case. It works only "As long as your development machine is turned on and connected to Tailscale" — [Tailscale Funnel examples](https://tailscale.com/kb/1247/funnel-examples). Funnel endpoints get valid certificates automatically, and `--bg` keeps the funnel running in the background — [OneUptime guide, 2026-01 [practitioner]](https://oneuptime.com/blog/post/2026-01-28-tailscale-funnel-service-publishing/markdown)
- Precedent from practitioners:
  - keeper.sh polls with provider sync tokens and pushes to destinations by polling "rather than provider push notifications", with an optional `WEBHOOK_PUBLIC_URL` for real-time push — [keeper.sh GitHub](https://github.com/ridafkih/keeper.sh)
  - tuiboard caches events for 30 minutes and refreshes every 5 minutes — [tuiboard GitHub](https://github.com/NazzarenoGiannelli/tuiboard)

**Quotas and calendar usage limits**
- Limits: 10,000 requests per minute per project, 600 per minute per user per project, and a 1,000,000 per day per project billing threshold, all on a sliding window. "projects created on or after May 1, 2026 are subject to the new quotas."
- Exceeding them returns 403 or 429 usageLimits.
- Recommended handling:
  - truncated exponential backoff;
  - randomize sync times by ±25%;
  - prefer push notifications filtered by eventType over polling everything.
- The page also lists MCP-server query costs per tool, e.g. `delete_event` costs 10.

  Source: [Calendar API quota guide](https://developers.google.com/workspace/calendar/api/guides/quota)
- Calendar product limits:
  - more than 100,000 events created "in a short period" reduces event creation (and may block edits for a few hours);
  - 10,000 external invites;
  - about 2,000 external guest emails per 24 h;
  - more than 60 calendars created;
  - sharing with 750 users.
  - Trial and new accounts have stricter, unpublished limits.

  Source: [Avoid Calendar use limits](https://knowledge.workspace.google.com/admin/calendar/avoid-calendar-use-limits)

**OAuth scopes, publishing status, the 7-day expiry, desktop flow**
- The 20 Calendar scopes include:
  - `calendar.app.created`: "Create secondary calendars and manage events on them";
  - `calendar.events.owned`: read, create, change and delete events on calendars the user owns;
  - `calendar.freebusy` and `calendar.events.freebusy`;
  - `calendar.calendarlist.readonly`.
- Guidance: "choose the most narrowly focused scope possible".

  Source: [Calendar API auth/scopes](https://developers.google.com/workspace/calendar/api/auth)
- Calendar scopes are sensitive; Google's examples of sensitive scopes include reading Calendar events. Sensitive scopes "might be restricted by Google Workspace organization administrators" — [Sensitive scope verification](https://developers.google.com/identity/protocols/oauth2/production-readiness/sensitive-scope-verification). A developer-forum thread notes the scope picker lists `calendar.events` as sensitive but not `calendar.app.created` or `calendar.freebusy`; it is unanswered — [Google developer forum](https://discuss.google.dev/t/oauth-verification-pending-since-22-apr-2026-first-trust-safety-email-never-received/392005)
- Refresh-token expiry:
  - With an External user type and "Testing" status, refresh tokens expire in 7 days, "unless the only OAuth scopes requested are a subset of name, email address, and user profile".
  - There is a limit of "100 refresh tokens per Google Account per OAuth 2.0 client ID".
  - Tokens also expire after six months unused.

  Source: [Google OAuth 2.0 overview](https://developers.google.com/identity/protocols/oauth2)
- App audience:
  - Testing allows "up to 100 test users", and "Authorizations by a test user will expire seven days from the time of consent".
  - "In production" with unverified sensitive scopes shows an "Unverified apps" warning.
  - The Internal user type requires the project to belong to a Google Cloud Organization.

  Source: [Manage app audience](https://support.google.com/cloud/answer/15549945?hl=en)
- Unverified apps are capped at 100 new users for the project's lifetime. Verification is not needed for "experimental or test builds" or for internal apps within one Workspace domain — [Unverified apps](https://support.google.com/cloud/answer/7454865?hl=en). Practitioner workaround: publish the app to production and accept the warning — [nspady/google-calendar-mcp README](https://github.com/nspady/google-calendar-mcp)
- For desktop apps, loopback IP redirects are recommended. The out-of-band (OOB) copy-paste flow "is no longer supported", and so are custom URI schemes. PKCE is supported, and installed apps are assumed unable to keep secrets — [OAuth for native/desktop apps](https://developers.google.com/identity/protocols/oauth2/native-app)

**MCP servers and CLIs**
- Google's official remote Calendar MCP server:
  - endpoint `https://calendarmcp.googleapis.com/mcp/v1`, Streamable HTTP transport;
  - "Available as part of the Google Workspace Developer Preview Program", so membership is required;
  - you set up a Google Cloud project, enable `calendar-json.googleapis.com` and `calendarmcp.googleapis.com`, and create a Web OAuth client (with Claude's redirect `https://claude.ai/api/mcp/auth_callback`);
  - the listed scopes are read-only (calendarlist.readonly, events.freebusy, events.readonly), yet create_event, update_event and delete_event appear among the tools.
  - Tools: create_event, delete_event, get_event, list_calendars, list_events, respond_to_event, suggest_time, update_event (plus search_events in the sidebar).
  - It warns about indirect prompt injection and says "Never connect Google Calendar MCP server to untrusted or unverified applications". Page last updated 2026-10-09.

  Sources: [Configure the Calendar MCP server](https://developers.google.com/workspace/calendar/api/guides/configure-mcp-server); [MCP reference](https://developers.google.com/workspace/calendar/api/v3/reference/mcp)
- Community servers:
  - nspady/google-calendar-mcp: MIT, about 1.2k stars, updated 2026-10-11. It can "Connect multiple Google accounts (e.g., work, personal) and query them simultaneously" with cross-account conflict detection, uses a user-created "Desktop app" OAuth client and test users, and runs over stdio or HTTP — [GitHub](https://github.com/nspady/google-calendar-mcp)
  - taylorwilsdon/google_workspace_mcp: Python, about 3.3k stars, covers Gmail, Calendar, Docs and more — [GitHub](https://github.com/taylorwilsdon/google_workspace_mcp)
  - keeper.sh: AGPL-3.0, about 1.4k stars; calendar sync plus an MCP server whose tools include `find_free_time` — [GitHub](https://github.com/ridafkih/keeper.sh)
- googleworkspace/cli:
  - Rust, Apache-2.0, about 31.4k stars, created 2026-03-02.
  - "This is **not** an officially supported Google product."
  - "every response is structured JSON".
  - Calendar helpers `+agenda` and `+insert`.
  - Auth via your own Desktop OAuth client (`gws auth setup` uses gcloud), a service account, or a pre-obtained token.
  - Ships Agent Skills.

  Source: [GitHub](https://github.com/googleworkspace/cli)
- gcalcli also requires your own OAuth client ("You currently have to use your own Calendar API token") — [GitHub](https://github.com/insanum/gcalcli)

### Inferences
- What each account can do:
  - Work (Workspace, primary calendar): focusTime/OOO events can be written; working location events are subject to admin settings; Gemini helpers are available if the plan is paid.
  - Personal Gmail: only default events, so protective blocks are plain opaque events, with `visibility: private` where useful.
  - eventType can't be changed after creation, so turning a default block into focusTime means delete and recreate. Create focusTime only when a block is committed.
- Because sync tokens don't work with most filters, run an unfiltered incremental sync per calendar and filter locally (e.g. by private extendedProperties, or to skip workingLocation events, which are transparent anyway).
- Polling budget: four calendars every 60-120 seconds is about 2-4 requests a minute, far below the 600/min/user quota. Real-time push mostly isn't worth a public endpoint on a laptop. Keep Funnel as an optional upgrade, with polling still running as the backstop.
- IDs: generate a UUID per block, encode it as base32hex, and store the mapping locally. A 409 on insert means "already created": fetch and reconcile. That makes retries idempotent.
- MCP stance: Narwarp should own all calendar writes. Agents get Narwarp's own tools (propose, pin, explain) rather than raw Google write access. Use the official MCP server (preview) or nspady read-only for agent Q&A. In this research environment, the attached "Google Calendar" connector describes itself as "provided by Google Calendar API" and exposes the same tool names as the official server (including suggest_time and search_events). That suggests Claude's connector may use Google's official server, though this is an observation of this environment, not a documented fact.
- googleworkspace/cli is attractive for a Rust codebase as a reference for Discovery-driven clients, but it is not an officially supported Google product and has no stability guarantees.

### Gaps
- Not verified: whether outOfOffice events work on consumer Gmail (probably Workspace-only, like Focus Time).
- Not verified: whether working location needs admin enablement for API writes.
- Maximum watch-channel TTL is undocumented.
- Whether `calendar.app.created` and `calendar.freebusy` count as sensitive is unresolved.
- Whether the official Calendar MCP supports consumer Gmail accounts is not stated.
- Whether a deleted event's ID can be reused, or still triggers 409, is not documented in the pages read.
- Google Calendar's native task time-blocks (Nov 2025) have no documented API surface.

## 6. Work vs personal calendars: mirroring busy-only blocks, privacy, Workspace admin restrictions

### Takeaway
Hold two independent OAuth grants, one for work and one for personal, and compute "me-busy" locally instead of relying on sharing between accounts. When a calendar needs to block time from the other world, write opaque "Busy" placeholders with no details, tagged with private extendedProperties, on that account's primary calendar. Secondary calendars generally don't count as busy when colleagues use "Find a time", and status events can't live on them. Expect the employer's Workspace to possibly block or limit third-party OAuth apps, restrict Calendar API access, or cap external sharing to free/busy only, and design graceful fallbacks.

### Cited Findings
- Workspace admin controls for apps:
  - Unconfigured third-party apps: "Allow users to access any third-party apps" (default), "Allow users to access third-party apps that only request basic info needed for Sign in with Google", or "Don't allow users to access any third-party apps".
  - Per-app access levels: Trusted, Limited, Specific Google data, or Blocked.
  - If Calendar is set as Restricted, "Internal and third-party apps with a Limited access setting can't access Calendar data".
  - "Trust internal apps" lets internal apps reach restricted APIs.

  Source: [Workspace Admin Help: control which apps access Workspace data (updated 2026-10-09)](https://knowledge.workspace.google.com/admin/apps/control-which-apps-access-google-workspace-data)
- External calendar sharing options are "Only free/busy information (hide event details)", "Share all information, but outsiders cannot change calendars", and two broader options. "After you limit external sharing for your organization, users can't exceed these limits when sharing individual events" — [Workspace Admin Help: calendar sharing options](https://knowledge.workspace.google.com/admin/calendar/set-google-calendar-sharing-options)
- Workspace admins may restrict sensitive scopes — [Sensitive scope verification](https://developers.google.com/identity/protocols/oauth2/production-readiness/sensitive-scope-verification)
- Status events exist only on primary calendars — [status guide](https://developers.google.com/workspace/calendar/api/guides/calendar-status). Google's "Find a time" typically only sees primary-calendar appointments, so secondary-calendar items usually don't show as busy to colleagues; invitations always land on the primary — [Cloudmen blog [practitioner]](https://cloudmen.com/en/blogs/news/de-secundaire-agenda-een-verborgen-kracht). Microsoft describes the same "by design" behavior for Outlook secondary calendars — [Microsoft support](https://support.microsoft.com/en-us/office/share-calendars-in-outlook-on-the-web-b47841f0-6a6b-40c2-aa16-eb971c25f009)
- Reclaim Calendar Sync privacy:
  - Synced events show as busy for "Work/Travel/Personal Commitment" or plain "Busy" without details, and are publicly visible on the destination calendar.
  - Events marked free are not synced under the Busy or Personal Commitment privacy levels.
  - All-day events are skipped by default.

  Source: [Reclaim Calendar Sync [vendor]](https://reclaim.ai/features/calendar-sync). Reclaim also warns that IT administrators may see full details of calendars on the organization's Google account — [Reclaim blog [vendor]](https://reclaim.ai/blog/10-google-calendar-issues-you-didnt-know-you-had)
- keeper.sh:
  - per-source privacy strips details and replaces the title with a `{{calendar_name}}` or `{{event_name}}` template;
  - its own events get a `@keeper.sh` UID suffix (Outlook gets a category instead);
  - "the cleanup sweep only ever touches events Keeper.sh created";
  - deletions are tracked through a mapping row that outlives the source event.

  Source: [keeper.sh GitHub](https://github.com/ridafkih/keeper.sh)
- nspady's MCP server supports multiple accounts with cross-account conflict detection — [GitHub](https://github.com/nspady/google-calendar-mcp)
- Event `visibility: private` means "Event details are visible only to attendees"; `transparency` controls whether an event blocks time — [Events resource](https://developers.google.com/workspace/calendar/api/v3/reference/events)

### Inferences
- Mirror policy:
  - Personal to work: for personal events that are opaque, not declined and inside work hours (±1 h), create a work-primary event titled "Busy" (or "Personal"), opaque, private, with no description, location or attendees. Skip all-day and free events by default, as Reclaim does.
  - Work to personal: optional, a "Work" placeholder only on days the user marks as "life-protected".
  - Never copy titles, attendees, locations or video links across accounts.
- Loop prevention: never mirror an event that already carries Narwarp tags. Mirrors store a salted hash of the source event ID, not the ID itself, so the work calendar holds no personal identifiers. Narwarp only ever deletes events it created, the keeper.sh rule.
- Data minimization: Life ticket titles never go on the work calendar, not even inside private extendedProperties, because admins may be able to read work-account data.
- Fallbacks if the employer blocks Narwarp's OAuth client (unconfigured apps disallowed, or Calendar restricted):
  - (a) Personal-only mode: work availability comes from manually entered hours plus a "meeting load" estimate.
  - (b) If external sharing allows free/busy, share the work calendar's free/busy with the personal account and read it with the personal token's freeBusy.
  - (c) Manually import ICS exports, if allowed.

  In all three, Narwarp cannot write protective blocks into the work calendar.
- Let the user choose per direction (personal to work, work to personal) whether mirroring happens, and keep an audit log of what was mirrored.

### Gaps
- No official Google statement was found confirming that secondary calendars are excluded from "Find a time" or other free/busy views.
- The user's employer's actual Workspace settings are unknown.
- Whether consumer Gmail can see a Workspace calendar's free/busy depends on the employer's external-sharing setting; not testable here.

## 7. How do terminal calendar UIs render day/week views, and what works?

### Takeaway
Successful terminal calendars converge on a few patterns:
- a narrow month or 3-month navigator next to an agenda list (khal/ikhal, calcure);
- a time-grid week view with a fixed time column on the left and a number of day columns that adapts to terminal width (lazycal), with zoom levels (chronos);
- all-day items in a separate strip at the top (tuiboard);
- vim-style motions with discoverable key hints;
- one color per calendar, and masking of private titles.

For Narwarp, the right set is a Day timeline as the primary view, a Week grid, an Agenda list, and a Plan-diff view for reviewing and approving proposed changes.

### Cited Findings
- calcure: Python, MIT, about 2.4k stars (2,372 on 2026-10-13). "Modern TUI calendar and task manager with minimal and customizable UI"; vim-style keys; "Operation with fewest key presses possible"; icons by event name; private items masked ("Private events and tasks •••••"); "Resize and mobile friendly"; shows events from .ics files, with Google reportedly read-only — [GitHub](https://github.com/anufrievroman/calcure)
- khal:
  - `calendar` shows three consecutive months with today highlighted.
  - ikhal has a left calendar pane and a right event-list pane, switched with tab or enter.
  - Default bindings follow vim, and "ikhal can be used entirely with the keyboard".
  - Events marked for deletion show a `D` and are deleted when khal exits.
  - Colour is stripped automatically when output isn't a TTY.
  - There is intentionally no default command.

  Source: [khal docs](https://khal.readthedocs.io/en/latest/usage.html)
- gcalcli: about 3.8k stars, MIT. Commands include agenda, calw (week grid) and calm (month grid), with an "ascii text graphical calendar display with variable width" and `--lineart=unicode`; it is also usable in conky. You bring your own OAuth client — [GitHub](https://github.com/insanum/gcalcli)
- chronos (Go, bubbletea):
  - week view with zoom levels of 30, 15, 5 and 1 minutes;
  - vim motions, including count-prefixed jumps;
  - visual mode, undo and redo;
  - key hints on Space;
  - data stored as `.ics` files, with CalDAV sync marked "Experimental".

  Source: [GitHub](https://github.com/samuelstranges/chronos)
- lazycal (OpenTUI/TypeScript, a Google Calendar TUI created 2026-02):
  - day, week and month modes;
  - "Responsive day columns (auto include/exclude days by terminal width)";
  - "Time column always shown on the left in day/week views";
  - a side panel that "auto hides on narrow widths";
  - per-calendar colors and a `?` help modal;
  - a Desktop OAuth client the user creates.

  Source: [GitHub](https://github.com/pranavkarthik10/lazycal)
- tuiboard (OpenTUI/SolidJS/Bun, created 2026-05):
  - "a 24-hour agenda you drag and resize time blocks on, with a Google / Microsoft 365 calendar overlay";
  - "timed events render as colored 📅 blocks on the grid, and all-day events ride in a chip strip at the top";
  - read-only by default, with writing opt-in via `--write`;
  - a live read-only view of local Claude Code, Codex and other coding-agent sessions;
  - boards stored as plain Markdown.

  Source: [GitHub](https://github.com/NazzarenoGiannelli/tuiboard)
- Others from 2026:
  - chroncal: Go, CLI and TUI, iCalendar/CalDAV, SQLite, about 221 stars — [GitHub](https://github.com/DouglasdeMoura/chroncal)
  - google-calendar-tui: Rust, lists upcoming events — [GitHub](https://github.com/akitaonrails/google-calendar-tui)
  - caldr: CalDAV TUI, archived — [GitHub](https://github.com/mrusme/caldr)
- Proposal and capacity cues from GUI planners: Morgen's preview events "pulsate" to set them apart from booked events — [Morgen guide](https://www.morgen.so/guides/plan-your-day-using-the-ai-planner); Sunsama shows estimated completion time against shutdown time — [Sunsama help](https://help.sunsama.com/docs/daily-planning)

### Inferences
- Views for Narwarp:
  - **Day timeline (default):** 15-minute rows (or 30-minute rows using half-block glyphs for 15-minute precision); time gutter on the left; a "now" line; the frozen window shaded; block types distinguished by glyph and color as well as color alone:
    - meeting: solid;
    - focus or ticket chunk: heavy border plus ticket key (e.g. `ENG-123`);
    - decompression: dim or dotted;
    - free time: distinct color plus lock mark;
    - proposals: ghosted, with `+`/`~`/`−` diff markers.
  - **Week grid:** 7→5→3→1 columns depending on width (lazycal pattern); an all-day strip (tuiboard); zoom keys (chronos); a capacity bar per day showing planned vs available and an over-commit warning (Sunsama pattern); "at risk" badges.
  - **Agenda:** chronological list grouped by day with ticket and project. It works best in narrow panes, and its plain text is easy for agents to read.
  - **Plan diff:** a list of moves (old → new time, reason, which constraint was binding), accepted all at once, per item, or rejected. This mirrors Reclaim's Preview Mode and Morgen's "Schedule all".
- Keys: vim motions; `p` pin, `x` unschedule, `+`/`-` resize by 15 minutes, `m` then hjkl to move, `a` accept proposal, `R` re-plan, `u` undo to the previous plan version, `W`/`L` toggle Work or Life overlay, `?` key hints.
- Privacy: a "presentation mode" that masks personal titles (calcure's ••••• pattern) for screen sharing during work calls.
- The calendar view should read from the local plan database, not straight from Google, so it renders instantly and works offline, with sync running in the background (tuiboard's cache-then-refresh pattern).

### Gaps
- I found no usability research on terminal calendars. The UX recommendations above are inferred from existing tools.

## 8. Concrete scheduler design proposal for Narwarp (inputs, constraints, objective, algorithm, re-planning, writing to Google Calendar, privacy)

### Takeaway
Build a local-first "planner plus projector":
- SQLite holds tickets, availability and a versioned plan.
- A deterministic greedy repairer is always on; a CP-SAT optimizer runs for the morning and weekly plans or on demand.
- Plans are scored on deadlines, protected time, energy fit, fragmentation and a change tax.
- The TUI shows a diff for the user to approve.
- Only approved, near-term blocks are written to the primary calendar of the right account, with private tags, Narwarp-generated IDs, and etag-guarded patches.
- The far-future plan stays local, or goes to a Narwarp-created secondary calendar that cannot block colleagues.
- Personal and work calendars see each other only as opaque "Busy" placeholders.

### Cited Findings
Key facts the design depends on:
- Preview-and-approve beats silent reshuffling:
  - Reclaim 2.0 Preview Mode and no per-task events — [Reclaim help](https://help.reclaim.ai/en/articles/17438753-switching-from-reclaim-1-0-to-2-0), [Reclaim 2.0 Tasks](https://help.reclaim.ai/en/articles/16558552-reclaim-2-0-tasks-overview)
  - Morgen's approve-before-sync — [Morgen guide](https://www.morgen.so/guides/plan-your-day-using-the-ai-planner)
  - Motion's tasks that "shift multiple times" — [Motion help](https://www.usemotion.com/help/time-management/auto-scheduling/reference-auto-scheduling/how-auto-scheduling-works-behind-the-scenes)
- Soft holds that harden (Free → Busy as time approaches or the day fills) — [Reclaim help](https://help.reclaim.ai/en/articles/4129290-how-time-defense-settings-work)
- Change tax and pinning — [Timefold](https://docs.timefold.ai/timefold-solver/latest/responding-to-change/non-disruptive-replanning). Hints for warm starts — [OR-Tools](https://github.com/google/or-tools/blob/stable/ortools/sat/docs/model.md)
- Write mechanics:
  - Narwarp-generated base32hex IDs — [events.insert](https://developers.google.com/workspace/calendar/api/v3/reference/events/insert)
  - private extendedProperties and their limits; use patch — [extended properties](https://developers.google.com/workspace/calendar/api/guides/extended-properties)
  - 412 etag conflicts — [errors](https://developers.google.com/workspace/calendar/api/guides/errors)
  - eventType immutable — [Events resource](https://developers.google.com/workspace/calendar/api/v3/reference/events)
  - status events only on primary calendars — [status guide](https://developers.google.com/workspace/calendar/api/guides/calendar-status)
  - Focus Time needs a work or school account — [Calendar Help](https://support.google.com/calendar/answer/11190973)
- No working-hours setting in the API — [Settings](https://developers.google.com/workspace/calendar/api/v3/reference/settings). Sync tokens don't work with most filters, and 410 means full resync — [Sync guide](https://developers.google.com/workspace/calendar/api/guides/sync)
- Quotas: 600/min/user, 10,000/min/project — [quota guide](https://developers.google.com/workspace/calendar/api/guides/quota). Product limits, e.g. 100k events in a short period — [use limits](https://knowledge.workspace.google.com/admin/calendar/avoid-calendar-use-limits)
- Testing mode means a 7-day refresh-token expiry — [OAuth overview](https://developers.google.com/identity/protocols/oauth2), [app audience](https://support.google.com/cloud/answer/15549945?hl=en). Desktop apps use a loopback redirect with PKCE, and OOB is unsupported — [native app OAuth](https://developers.google.com/identity/protocols/oauth2/native-app)
- Workspace admins can block third-party apps or Calendar access — [admin help](https://knowledge.workspace.google.com/admin/apps/control-which-apps-access-google-workspace-data)
- Evidence for breaks: micro-breaks help vigor and fatigue more than performance — [Albulescu 2022](https://doaj.org/article/c6f37b6ba159469296b2cb968b65505f). Autonomy matters — [Trougakos 2014](https://research.polyu.edu.hk/en/publications/lunch-breaks-unpacked-the-role-of-autonomy-as-a-moderator-of-reco/). Planning fallacy — [Buehler 1994](https://bear.warrington.ufl.edu/brenner/mar7588/Papers/buehler-et-al-1994.pdf)

### Inferences

**8.1 Inputs**
- **Tickets** (from the kanban, which may aggregate many Linear issues):
  - `id`, `project`, `mode` (Work or Life), `priority` P0-P3;
  - `due` {date or datetime, `hard` or `soft`};
  - `estimate_min` (optionally low/likely/high) and `remaining_min`;
  - chunk policy {`min` 30, `preferred` 90, `max` 120, `splittable`};
  - `energy` (deep, shallow or admin), `deps`, `earliest_start`;
  - allowed `time_maps` (SkedPal-style; multiple maps intersect by default);
  - `asap`, `pinned`.
- **Calendars:**
  - work primary (Workspace) and personal primary (Gmail), plus optional read-only subscriptions such as holidays or a partner's calendar;
  - per event: start/end, transparency, eventType, my responseStatus, attendee count, whether it has a video link, whether I organize it, recurrence.
- **Profile** (kept locally, because the API has no working-hours setting):
  - working hours per weekday per mode, and a hard stop;
  - an energy curve (High, Medium or Low per hour);
  - caps: deep work per day, meetings per day, and total planned work at most about 80% of free work time;
  - break rules and protected life commitments (e.g. gym, family dinner);
  - time zone(s).
- **Telemetry:** actual time per block (TUI timer or agent-session activity), completion events, and user edits detected by sync.

**8.2 Time model**
- 15-minute slots over a 14-day horizon, split into three zones:
  - **Frozen:** now to +60 minutes, plus any block in progress.
  - **Committed:** the rest of today and the next working day. Written to Google as opaque, busy events.
  - **Tentative:** days 3-14. Kept local, or written as transparent holds on a Narwarp-created "Narwarp plan" secondary calendar (scope `calendar.app.created`). That calendar deliberately does not block colleagues.

**8.3 Hard constraints**
1. No overlap with busy time on either account, combined into one "me" timeline. Busy time means:
   - opaque events I haven't declined (configurable: treat needsAction/tentative as busy);
   - outOfOffice events and existing focusTime;
   - all-day opaque events if configured.
2. Mode windows: Work tickets only inside Work windows, Life tickets only inside Life windows, and nothing past the hard stop unless the user explicitly allows it.
3. Protected blocks can't be overlapped by tickets: free time, the minimum lunch window, and decompression after qualifying meetings.
4. Chunk-length bounds, dependencies, release times, and hard deadlines. If these are infeasible, report it; never silently violate them.
5. Pins hold: the frozen zone, blocks the user moved, created, edited or locked.
6. Caps hold: deep-work minutes per day, and no deep chunk starting less than 10 minutes after a meeting of 45 minutes or more.

**8.4 Objective** (lexicographic, or weighted if tuning needs it)
- L1: zero hard infeasibility (otherwise go to triage, 8.5 Stage D).
- L2: minimize priority-weighted lateness against soft deadlines, measured against "finish-by = due − margin" (margin 0.5-1 day by default).
- L3: minimize the change tax, summed over moved blocks:
  - for each moved block, a weight × (2 if the user has seen it) × (3 if it starts within 24 h);
  - plus a small term for how far it moved.
- L4: maximize fit:
  - energy match (deep chunks in High windows);
  - contiguity (one 90-120 min chunk beats three 30-min chunks);
  - few project switches per half-day (attention residue);
  - about 20% slack left per day;
  - earliness for ASAP tickets.
- Starting weights if using a weighted sum (to tune): lateness 100 per minute per priority level; move 30-300 per block; energy mismatch 5 per slot; 20 per extra chunk; 15 per project switch.

**8.5 Algorithm**
- **Stage A (availability):** merge both accounts' events into busy intervals. Work out free intervals per mode window. Insert decompression and lunch as protected intervals where the rules apply. Lunch is a flexible window that turns rigid two hours ahead (Reclaim's harden pattern).
- **Stage B (greedy repair, deterministic, under ~50 ms, always on):**
  - Sort tickets by hard due date, then least slack (time until due minus remaining work), then priority, then ASAP.
  - Keep each existing placement if it is still valid.
  - Otherwise place chunks in the best-scoring free slots: energy fit + contiguity + earliness − distance from the old start.
  - Respect caps, and record a human-readable "reason" for each block.
- **Stage C (CP-SAT optimize, 2-5 s time limit):** runs at the morning ritual, the weekly review, or on `replan --optimize`.
  - Optional fixed-size intervals for candidate chunks (per ticket × day × length options), with start domains limited to free windows.
  - `add_no_overlap` across chunks plus fixed busy and protected intervals; deadlines; daily sums; the objective above.
  - The current plan goes in via `add_hint`.
  - Accept the CP-SAT result only if it beats the current plan by more than the change tax it would incur.

  Sketch (untested):
  ```python
  m = cp_model.CpModel()
  for c in chunks:  # candidate chunk: allowed = free windows ∩ time-map, in slot units
      s = m.new_int_var_from_domain(cp_model.Domain.from_intervals(c.allowed), f"s{c.id}")
      on = m.new_bool_var(f"on{c.id}")
      iv = m.new_optional_fixed_size_interval_var(s, c.len, on, f"iv{c.id}")
      if c.prev_start is not None:            # stability (change tax)
          kept = m.new_bool_var(f"k{c.id}")
          m.add(s == c.prev_start).only_enforce_if(kept)
          cost.append(c.move_weight * (1 - kept))
          m.add_hint(s, c.prev_start); m.add_hint(on, 1)
  m.add_no_overlap(chunk_intervals + busy_intervals + protected_intervals)
  ```
- **Stage D (triage when infeasible):** select which tickets slip, Moore-Hodgson style. Offer the options: defer, cut scope, extend hours, or trade free time. Extending hours and trading free time are never automatic; the user must choose them. Post "at risk" to the ticket's message board.
- **Stage E (explain):** keep each block's binding constraint ("earliest 90-min high-energy slot before Fri due; P1"). A `?` key shows it, and Claude or a local Ollama model can narrate it.

**8.6 Re-planning policy**
- Triggers:
  - a calendar change seen by sync;
  - a ticket change (priority, estimate, due, done or blocked);
  - a block ending or overrunning;
  - the morning ritual;
  - an explicit command;
  - a deadline-risk change.
- Debounce 3 minutes and coalesce changes, because meetings tend to be edited in bursts.
- Repair scope, in order: only the conflicting blocks, then the nearest valid slot the same day, then later days. Frozen and pinned blocks never move.
- Move budgets: at most 2 automatic moves per block per day and 6 per day overall. Beyond that, Narwarp proposes and waits.
- Hysteresis: apply automatically only for hard conflicts or when a ticket's risk class changes. Everything else becomes a proposal in the Plan-diff view.
- Hardening: tentative blocks become committed at the evening or morning ritual (about 48 h ahead at most), and committed blocks are opaque. This keeps churn colleagues can see to a minimum.
- User edits win:
  - a moved or resized block, or a 412 etag mismatch, marks the block pinned and records the preference (e.g. "user prefers Tue AM for project X");
  - a deleted block unschedules that chunk and asks whether to defer;
  - a user-created event on top of a block moves the block.
- Overruns: at block end, if the ticket isn't done, prompt "extend 15 or 30 / reschedule remainder / done".
  - If the prompt is ignored, assume "continue" only up to the next commitment or protected block.
  - Update the remaining time.
  - Learn a per-project inflation factor: the median of actual/estimate over the last 10 blocks, clamped to 1.0-2.0, applied to scheduling estimates. Show the raw estimate alongside.
- Meetings that land on a focus block: for Workspace focusTime created by Narwarp, default `autoDeclineMode` to `declineNone`, so the harness never causes declines the user didn't ask for. The repairer then moves the focus block. Auto-decline is opt-in per user.

**8.7 Writing blocks to Google Calendar**
- Target calendars by block kind:

  | Block kind | Target calendar | eventType | transparency / visibility | Title |
  |---|---|---|---|---|
  | Work ticket focus/chunk (committed) | work primary | `focusTime` if opted in (Workspace only), else `default` | opaque / default | "Focus: ENG-123 short title" or generic "Focus" |
  | Work-day decompression or free time | work primary | default | opaque / private | "Break" or "Busy" |
  | Life ticket block or personal free time | personal primary | default | opaque / private | details allowed (personal account) |
  | Personal busy mirrored to work | work primary | default | opaque / private | "Busy" (no description, location or attendees) |
  | Tentative (days 3-14) | app-created secondary "Narwarp plan" calendar in each account | default | transparent / private | "(plan) …" |

- Every block gets `extendedProperties.private`:
  - `nw=1`, `nw_block=<uuid>`, `nw_kind=focus|chunk|buffer|free|mirror|hold`;
  - `nw_ticket` on the work calendar for Work tickets only;
  - `nw_epoch=<plan version>`;
  - `nw_hash` = a hash of start, end and summary, to spot outside edits.

  Keys stay at 44 characters or fewer and values at 1024 or fewer.
- `colorId` is mapped per kind from `colors.get`.
- Writes:
  - The event `id` is the UUID encoded as base32hex. A 409 on insert means fetch and reconcile.
  - `sendUpdates=none`, no attendees, and reminders off (the TUI handles notifications).
  - Updates use `events.patch` with `If-Match: <etag>`. A 412 means re-fetch and treat it as a user edit (pin).
  - focusTime is created only at commit, because eventType is immutable.
- Deletes: only events with `nw=1` whose block ID Narwarp knows. `narwarp calendar purge` lists them with `privateExtendedProperty=nw%3D1` and shows the list before deleting.
- Rate and volume: debounce, at most one write batch per minute, truncated exponential backoff with jitter on 403/429, and a daily write budget of about 200, far below Calendar's product limits.

**8.8 Privacy between calendars**
- Separate OAuth tokens for each account, stored in the macOS Keychain. No calendar sharing between the accounts is needed.
- Mirroring follows section 6 (Busy-only, loop-safe, Narwarp-owned only). Life ticket titles never reach the work calendar or its extendedProperties.
- Event titles and descriptions from incoming invites are untrusted input; Google's MCP docs warn of prompt injection. Never let calendar text trigger actions without confirmation.
- Classify personal events locally (Ollama). Cloud models get only abstract availability (busy and free slots) unless the user opts in.
- Keep an audit log of every write and mirror, with one-command purge and undo.

**8.9 OAuth and sync plumbing on a laptop**
- One Google Cloud project owned by the personal account: External user type, published "In production" while unverified (avoids the 7-day expiry; the 100-user cap doesn't matter).
- A Desktop OAuth client using the loopback redirect and PKCE.
- Scopes: `calendar.events.owned` (or `calendar.events` if invites on others' calendars must be read), `calendar.calendarlist.readonly`, `calendar.freebusy`, and `calendar.app.created` for the tentative calendar.
- The work account authorizes the same client, subject to employer policy; fall back as in section 6 if it is blocked.
- A health check warns before refresh tokens fail (e.g. after six months unused) and offers re-auth from the TUI.
- Sync loop per calendar:
  - `events.list` with `syncToken` and no filters;
  - every 60-120 s while the TUI is open, every 10-15 minutes in the background with ±25% jitter, and immediately on wake or network change;
  - full resync on 410;
  - store raw JSON and etag.
- Optional real-time push: watch channels through Tailscale Funnel, renewed before the 7-day default TTL. Polling stays on as the backstop, since notifications "are not 100% reliable".
- SQLite tables: `calendars` (with sync token), `events`, `tickets`, `blocks` (state proposed, committed, pinned or done; Google ID; etag; epoch), `plan_epochs` (diff summary), `block_moves`, `user_edits`, `settings`, `audit_log`.

**8.10 Agent interface**
- Narwarp's own MCP tools for Claude Code and Ollama agents:
  - `get_availability(range, mode)`, `propose_plan(scope)`, `explain_block(id)`, `pin_block(id)`, `request_time(ticket, minutes, by)`;
  - `accept_plan(epoch)`, which requires human confirmation in the TUI.
- Agents never get raw Google write access. The official Google Calendar MCP (preview) or nspady can be used read-only for questions like "when am I free with X".
- Plan diffs and "at risk" alerts are posted to the ticket and project message boards (designed by another researcher).

**8.11 Suggested MVP order**
1. Read-only sync of both accounts, a unified availability timeline, and TUI Day/Week/Agenda views.
2. Greedy plan for today and tomorrow, Plan-diff approval, committed blocks written to primary calendars with tags, and the purge command.
3. The re-planning loop: freeze, budgets, user-edit pins, overrun prompts, inflation learning.
4. Breaks and free-time rules, the energy curve, and capacity warnings.
5. The CP-SAT optimizer with change tax, busy mirroring, Workspace focusTime, and optional push via Funnel.

**8.12 Failure modes and mitigations**
- Over-scheduling: an 80% capacity cap, a visible "unschedulable" list (Morgen pattern), workload warnings (Sunsama pattern), and free time is never consumed automatically.
- Thrash: freeze window, change tax, move budgets, debounce, a local-only far horizon, and previews.
- Ignoring energy: the energy curve, a low-energy window after meetings, and a deep-work cap.
- Trust: previews, a reason for every block, undo to the previous plan version, never touching events Narwarp didn't create, and an audit log.
- Calendar noise for colleagues: only near-term blocks are written, with generic titles, no attendees and no notifications.
- Token and admin failures: production publishing status, a health check, and degraded modes.

### Gaps
- Not verified:
  - whether consumer Gmail can hold OOO events;
  - whether a deleted event's ID can be reused;
  - the maximum watch-channel TTL;
  - whether `calendar.app.created` and `calendar.freebusy` count as sensitive;
  - whether the official Calendar MCP accepts consumer accounts;
  - the user's employer's Workspace app-access and external-sharing settings.
- CP-SAT solve times at this scale are expected to be small but were not benchmarked.
- The numeric policy defaults (budgets, debounce, freeze length, weights) are design judgment, not evidence-based. Validate them with the user over 2-4 weeks of use.
