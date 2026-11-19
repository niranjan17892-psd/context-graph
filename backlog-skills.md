# Rebuild Spec — Backlog Skills

Rebuild spec for the four Jira backlog-management skills (`backlog-triage`, `backlog-enrich`, `backlog-prioritize`, `backlog-groom`) and their shared resources (`skills/backlog-shared/`). A fresh agent given only this document should be able to recreate the skills with equivalent behavior.

Source files (all git-tracked; nothing else lives in these directories):

| Path | Size (bytes) |
|---|---|
| `skills/backlog-shared/session-config.md` | 5458 |
| `skills/backlog-shared/jira-helpers.md` | 4772 |
| `skills/backlog-triage/SKILL.md` | 8182 |
| `skills/backlog-enrich/SKILL.md` | 9616 |
| `skills/backlog-prioritize/SKILL.md` | 9498 |
| `skills/backlog-groom/SKILL.md` | 4286 |

## Family overview

- Audience: PMs and the CPO, operating on EXISTING Jira tickets. Pipeline: triage (shrink) -> enrich (make dev-ready) -> prioritize (rank). `backlog-groom` orchestrates all three in one conversation with early exits.
- Every skill reads `skills/backlog-shared/session-config.md` (startup protocol) and `skills/backlog-shared/jira-helpers.md` (presentation, batching, write gating). When run under `backlog-groom`, session config is set once and carried through all stages; a directly invoked sub-skill runs the protocol itself.
- Dependencies: Atlassian MCP server (Jira read/write) — tools are named with the `mcp__atlassian__` prefix (`getAccessibleAtlassianResources`, `getVisibleJiraProjects`, `searchJiraIssuesUsingJql`, `getJiraIssue`, `editJiraIssue`, `transitionJiraIssue`, `getTransitionsForJiraIssue`, `addCommentToJiraIssue`, `createJiraIssue`, `createIssueLink`, `getIssueLinkTypes`). Optional argus MCP tools (`mcp__argus__search_index`, `mcp__argus__trace_flow`) for codebase-aware enrichment and effort estimation.
- All questions go through the `AskUserQuestion` tool, one at a time.
- Each skill has a `<HARD-GATE>` forbidding implementation discussion (verbatim per skill below).

---

## Shared resource: `skills/backlog-shared/session-config.md`

Plain markdown, no frontmatter. Title: `# Backlog Skills — Session Configuration`. Opening: read this first before the skill-specific workflow; it defines the session setup protocol shared by all backlog skills (backlog-triage, backlog-enrich, backlog-prioritize, backlog-groom). Second paragraph: "When invoked via the `backlog-groom` orchestrator, this configuration is set once and carried through all stages. When a sub-skill is invoked directly, it runs this protocol itself." Sections separated by `---`.

### Section: Voice & Audience

"You are working with **product managers and executives**, not developers." Bullets (verbatim):

- No code references, no architecture jargon, no file paths, no repo names (unless the PM explicitly asks)
- Speak in terms of user outcomes, business value, and ticket quality
- Be direct but collaborative — PMs own the decisions, you surface the tradeoffs
- One question at a time, prefer multiple choice, keep it focused
- Push for specifics when answers are vague — then accept their framing
- Frame questions in outcome terms: "What makes a ticket ready for your team?" not "What fields should we populate?"

### Section: Startup Protocol (steps, in order)

1. **Read project context** — Read CLAUDE.md if it exists in the current directory.
2. **Resolve Atlassian Cloud ID** — call `mcp__atlassian__getAccessibleAtlassianResources`; store the cloud ID for all later Atlassian calls. If no accessible resources: STOP with "I can't connect to your Atlassian instance. Check that the Atlassian MCP server is configured and authenticated."
3. **Project Selection** — call `mcp__atlassian__getVisibleJiraProjects` with the cloud ID. One project -> confirm: "I see project [KEY] — [name]. Is that the one?" Multiple -> AskUserQuestion: "Which Jira project are we working with today?" with project names as options. Store the project key.
4. **Write-Access Level** — AskUserQuestion (verbatim):
   > "What level of changes should I make to Jira tickets in this session?"
   >
   > - **Read-only** — I'll analyze and recommend. You take all actions manually in Jira.
   > - **Light writes** — I can close/transition tickets, add labels and comments, update descriptions. I'll confirm each batch before acting.
   > - **Full writes** — All of the above plus creating/splitting tickets, linking issues, and bulk transitions. I'll still confirm before acting.

   Store the level for write-access gating (see `jira-helpers.md`).
5. **Working Set Definition** — "Never assume what statuses mean — every team's workflow is different." AskUserQuestion:
   > "How should I find the tickets to work with?"
   >
   > - **By status** — I'll ask which statuses to include (e.g., "Open", "To Do", "Backlog")
   > - **By JQL** — You provide a Jira query directly
   > - **Everything not Done** — All tickets that aren't in a Done/Closed status

   - By status: query a sample with `mcp__atlassian__searchJiraIssuesUsingJql`, JQL `project = [KEY] ORDER BY created DESC`, `fields: ["status"]`, `maxResults: 100`; extract unique status names; AskUserQuestion with `multiSelect: true`: "Which statuses make up your backlog?" using the discovered statuses.
   - By JQL: ask the PM to type their query.
   - Everything not Done: JQL `project = [KEY] AND statusCategory != Done ORDER BY created ASC`.
   - Optional refinements: "Want to narrow this further — by component, label, issue type, assignee, or date range? Or is the full set good?" If yes, ask one refinement at a time and append to the JQL.
6. **Preview the Working Set** — run the query with `fields: ["summary", "status", "issuetype", "priority", "assignee", "created", "updated"]`, `maxResults: 100`. Present:
   > "Found **N tickets** in your working set. Here's the breakdown:
   > - By type: X stories, Y bugs, Z tasks, ...
   > - By age: X less than 30 days, Y 30-90 days, Z 90-180 days, W over 180 days
   > - By assignee: X unassigned, Y assigned to [top assignees]
   >
   > Does this look like the right set, or should we adjust the filter?"

   Confirm -> proceed; changes -> back to step 5. If >100 results: "There are more than 100 tickets matching this filter. I'll work through them in pages. We can always narrow the filter if this is too many." Paginate with `nextPageToken` on later `searchJiraIssuesUsingJql` calls as needed.
7. **Enrichment Mode (conditional)** — ask ONLY for `backlog-enrich` or `backlog-groom`; skip for `backlog-triage` and `backlog-prioritize`. AskUserQuestion:
   > "When enriching tickets, should I pull in codebase context?"
   >
   > - **Product-only** — I'll draft acceptance criteria, user stories, and descriptions using what you tell me and what's in the ticket.
   > - **Argus-enhanced** — All of the above, plus I'll use argus to identify which system areas and flows each ticket affects. Written in PM language, not code.

   Store the mode.

---

## Shared resource: `skills/backlog-shared/jira-helpers.md`

Plain markdown, no frontmatter. Title: `# Backlog Skills — Jira Interaction Patterns`. Opening: reference patterns for all backlog skills; read alongside `session-config.md` at session start.

### Section: Ticket Presentation Format

Never include technical jargon, internal IDs, or field keys.

Single ticket (verbatim):
```
> **PROJ-123** — Fix upload timeout for large documents
> Status: Open · Type: Bug · Priority: High
> Created: 45 days ago · Last activity: 32 days ago
> Assignee: Jane Smith · Links: 2 related issues
```

Batch (table for 3+ tickets, verbatim):
```
> | Ticket | Summary | Status | Age | Last Active | Assignee |
> |--------|---------|--------|-----|-------------|----------|
> | PROJ-123 | Fix upload timeout | Open | 45d | 32d ago | Jane |
> | PROJ-124 | Add export button | To Do | 90d | 87d ago | Unassigned |
> | PROJ-125 | Dashboard loading slow | Open | 120d | 118d ago | Unassigned |
```
"Age" = days since `created`; "Last Active" from `updated`.

### Section: Batch Confirmation Pattern

Present proposed actions in batches of ~5 tickets. Per batch: (1) show tickets in the format above; (2) state the action clearly, e.g. "I recommend **closing** these 5 tickets as stale (no activity in 90+ days)."; (3) AskUserQuestion:
> "How should I handle these?"
>
> - **Approve all** — Apply the recommended action to all 5
> - **Review individually** — Let me decide one at a time
> - **Skip this batch** — Move on to the next group

"Review individually" -> one AskUserQuestion per ticket with ticket-specific options (e.g. close/keep/defer for triage, approve/edit/skip for enrichment).

**Bulk decisions:** if the PM says e.g. "close everything older than 6 months" or "all of these are fine", apply it and confirm ("Closing 12 tickets older than 180 days. Done."). Don't force ticket-by-ticket review after a clear bulk call.

### Section: Write-Access Gating (check before EVERY Jira write)

| Level | Allowed | Not allowed / behavior |
|---|---|---|
| **Read-only** | none | Never call `editJiraIssue`, `transitionJiraIssue`, `addCommentToJiraIssue`, `createJiraIssue`, `createIssueLink`. Present the recommendation instead, e.g. "I'd recommend closing PROJ-123 as stale. You can do this in Jira by transitioning it to Done/Won't Do." At the end of a phase, offer to export all recommendations as a summary list. |
| **Light writes** | `transitionJiraIssue` (status changes), `addCommentToJiraIssue`, `editJiraIssue` (labels, description, priority updates) | `createJiraIssue` (new tickets) and `createIssueLink` (linking) are presented as recommendations. Always confirm batches before acting. |
| **Full writes** | All Atlassian MCP write tools | "Still confirm batches before acting — full writes means broader permissions, not unilateral action" |

**Transition workflow for closing tickets:**
1. `mcp__atlassian__getTransitionsForJiraIssue` with the ticket key.
2. Find the transition to a Done/Closed/Won't Do status (status category "Done" in the transition targets).
3. `mcp__atlassian__transitionJiraIssue` with that transition ID.
4. For a triage comment, `mcp__atlassian__addCommentToJiraIssue` with `contentFormat: "markdown"`.

### Section: Clarifying Questions Principle

"Every phase of every backlog skill should check for ambiguity and ask before assuming. This is a core interaction principle, not optional."
- **When to ask:** data reveals something unexpected (unfamiliar statuses, unusual patterns, empty fields that should be populated); an ambiguous instruction ("clean these up" — close? enrich? re-prioritize?); a ticket's intent is unclear; the PM uses unfamiliar team-specific terminology.
- **How:** one question at a time via AskUserQuestion; multiple choice when options are clear; push once on vague answers then accept the PM's framing; acknowledge the answer before the next question.
- **When NOT to ask:** a clear bulk instruction already covers the tickets; the answer is obvious from context (don't ask "should I skip tickets that are already closed?" — just skip them); a similar question was already answered this session and still applies.

---

### Skill: backlog-triage

**Files to create:** `skills/backlog-triage/SKILL.md`.

**Frontmatter (verbatim):**
```yaml
---
name: backlog-triage
description: "Jira backlog triage for PMs and CPO. Identifies stale tickets, duplicates, orphans, and status mismatches, then walks through category-by-category cleanup. Use when someone says 'triage the backlog', 'clean up stale tickets', 'find duplicates', or 'backlog cleanup'."
---
```

**Title / intro:** `# Backlog Triage — Clean Up the Backlog`. "Shrink the backlog to only meaningful work by identifying and removing stale, duplicate, and orphaned tickets. Walk the PM through decisions category by category."

**Purpose / fires when:** triage / cleanup / duplicate-finding requests.

**Procedure:**

0. **Setup** — read `skills/backlog-shared/session-config.md`, run its startup protocol, SKIP step 7 (Enrichment Mode). Read `skills/backlog-shared/jira-helpers.md`.
1. **Phase 1 — Backlog Snapshot.** Compute from the loaded working set — category definitions (exact):
   1. **Count by status and issue type.**
   2. **Age buckets** (days since created): Less than 30 days / 30–90 days / 90–180 days / Over 180 days.
   3. **Staleness candidates** — `updated` more than 90 days ago.
   4. **Unassigned tickets** — no assignee.
   5. **Duplicate candidates** — summaries sharing 3+ significant words (ignoring common words like "the", "a", "fix", "add", "update"); present clusters of 2+ tickets.
   6. **Orphaned tickets** — no assignee AND no linked issues AND `updated` more than 60 days ago.

   Present the health report (template below), then AskUserQuestion "Where should we start?":
   - **Stale tickets first** (recommended — biggest cleanup impact)
   - **Duplicates first** — consolidate before other cleanup
   - **Orphans first** — quick wins, fast decisions
   - **Status mismatches first** — fix workflow accuracy

   Ask clarifying questions on unexpected patterns (e.g. "90% of your tickets are in a status called 'Parked' — can you tell me what that means in your workflow?").
2. **Phase 2 — Category-by-Category Triage**, in the PM's chosen order, each using the batch confirmation pattern:
   - **Stale Tickets** — filter `updated` > 90 days ago, oldest first. Per batch of ~5: show table incl. last activity and assignee; note decision context (linked issues, has description, ever assigned); AskUserQuestion "What should we do with these stale tickets?" — **Close all** (Transition all to Done/Won't Do with a triage comment) / **Review individually** (Decide one at a time) / **Keep all** (These are still relevant, move on). Individually: **Close it** (No longer relevant) / **Keep it** (Still valid work) / **Defer** (Not sure yet, skip for now). Closing (if write access allows): transition workflow, then comment "Closed during backlog triage — no activity in [X] days." Bulk shortcut ("close everything older than 6 months"): apply across remaining stale tickets, confirm the count first: "That's [N] tickets. Closing them all."
   - **Duplicate Clusters** — show each cluster as a table and highlight why they look duplicate (similar summaries, same component, similar descriptions). AskUserQuestion: "These [N] tickets look like duplicates. Which is the canonical ticket?" — one option per ticket "**[PROJ-123]** — [summary] (oldest / most detailed / most linked)", plus **Not duplicates** (Keep all of them, they're different work) and **Skip** (Decide later). Confirmed (if write access allows): (1) append useful context from non-canonical descriptions to the canonical via `mcp__atlassian__editJiraIssue`; (2) comment on each non-canonical: "Duplicate of [canonical-key]. Context merged."; (3) close non-canonicals via the transition workflow; (4) full writes only: `mcp__atlassian__createIssueLink` with type "Duplicate" to the canonical.
   - **Orphaned Tickets** — batches of ~5 in table format. AskUserQuestion: "These tickets have no owner, no links, and no recent activity. Quick call?" — **Close all** (Likely abandoned) / **Review individually** (Some might be worth keeping) / **Keep all** (These are real work, just neglected). For kept tickets: "Want to assign these to someone or add them to an epic so they don't become orphans again?"
   - **Status Mismatches** — two patterns: (1) **Stalled "In Progress"** — "In Progress" (or similar active) status with `updated` > 30 days ago; (2) **Possibly completed** — "To Do"/"Open" status whose description or comments contain keywords like "deployed", "merged", "completed", "shipped", "released". Present each:
     ```
     > **PROJ-123** — Fix upload timeout
     > Status: In Progress · Last activity: 45 days ago
     > **Issue:** Marked as in progress but no activity for 45 days. Likely stalled or completed without updating the ticket.
     ```
     AskUserQuestion: **Move to Done** (The work was completed) / **Move back to To Do** (Still needs doing, wasn't actually started) / **Leave as is** (The status is correct) / **Skip** (Decide later).
3. **Phase 3 — Triage Summary** — after all categories or when the PM stops early. Read-only mode uses the recommendations variant. Closing line: under `backlog-groom` -> "Triage is done. The surviving tickets are ready for enrichment."; standalone -> "Good triage session. When you're ready to work on ticket quality, `backlog-enrich` will help add acceptance criteria and clean up descriptions."

**Hard rules (verbatim):**
```
<HARD-GATE>
Do NOT discuss implementation details, code architecture, technical stack, or file structures. This skill works at the product/project management level. Use PM-appropriate language throughout.
</HARD-GATE>
```
All writes gated by the session write-access level and batch confirmation.

**Output format.** Health report (verbatim):
```
> "Here's the health of your backlog:
>
> **[N] total tickets**
> - By type: [breakdown]
> - By age: [breakdown with age buckets]
>
> **Issues found:**
> - [X] stale tickets (no activity in 90+ days)
> - [Y] possible duplicate clusters
> - [Z] orphaned tickets (unassigned, unlinked, inactive)
> - [W] potential status mismatches
>
> I'd recommend starting with [highest-impact category]. Want to work through these in that order, or pick a different starting point?"
```
Triage summary (verbatim):
```
> "**Triage complete.** Here's what happened:
>
> - **Closed:** [X] tickets
> - **Merged (duplicates):** [Y] tickets consolidated into [Z] canonical tickets
> - **Kept:** [W] tickets confirmed as valid
> - **Deferred:** [V] tickets skipped for later
>
> **Net reduction:** [starting count] → [ending count] ([percentage]% reduction)
>
> **Remaining by category:**
> - [breakdown of surviving tickets by type/status]"
```
Read-only variant (verbatim):
```
> "Here's what I'd recommend. You can action these in Jira:
> - Close these [X] stale tickets: [list of keys]
> - Close these [Y] duplicates (canonical tickets noted): [list]
> - Review these [Z] orphans: [list]
> - Fix these [W] status mismatches: [list with recommended transitions]"
```

**Acceptance check:**
- Read-only session, PM picks "Stale tickets first" and "Close all" on a batch -> no write tools are called; the agent states the recommendation per ticket and ends with the read-only recommendations summary.
- Light-writes session, duplicate cluster confirmed with a canonical -> canonical description edited, "Duplicate of [key]. Context merged." comments added, non-canonicals transitioned to Done via `getTransitionsForJiraIssue` + `transitionJiraIssue`; no "Duplicate" link created (full writes only).

---

### Skill: backlog-enrich

**Files to create:** `skills/backlog-enrich/SKILL.md`.

**Frontmatter (verbatim):**
```yaml
---
name: backlog-enrich
description: "Jira backlog enrichment for PMs and CPO. Scores tickets on dev-readiness, adds acceptance criteria and descriptions, optionally pulls codebase context via argus. Use when someone says 'enrich these tickets', 'make tickets dev-ready', 'add acceptance criteria', or 'refine ticket descriptions'."
---
```

**Title / intro:** `# Backlog Enrich — Make Tickets Dev-Ready`. "Take tickets that survived triage and make them actionable — clear descriptions, acceptance criteria, proper metadata, and optionally codebase context."

**Purpose / fires when:** enrich / make dev-ready / add acceptance criteria requests.

**Procedure:**

0. **Setup** — read `session-config.md`, run its startup protocol INCLUDING step 7 (Enrichment Mode). Read `jira-helpers.md`.
1. **Phase 1 — Quality Assessment.** For each ticket, fetch full details if not loaded: `mcp__atlassian__getJiraIssue` with `fields: ["summary", "description", "status", "issuetype", "priority", "assignee", "components", "labels", "issuelinks", "created", "updated"]`, `responseContentFormat: "markdown"`. Dev-readiness checks (exact):
   1. **Has acceptance criteria?** — description contains "given/when/then", "acceptance criteria", numbered or bulleted requirements, or "done when" statements.
   2. **Has a clear description?** — more than one sentence, says what needs to happen and why. Empty description, or one that just restates the summary, fails.
   3. **Has proper metadata?** — priority set (not "None" or default), issue type makes sense (a bug has reproduction steps, a story has user value), components or labels assigned.
   4. **Has links?** — related tickets, parent epic, or Confluence pages linked.

   Readiness categories (exact):
   - **Ready** — passes all 4 checks. No enrichment needed.
   - **Needs refinement** — passes 2-3 checks.
   - **Bare minimum** — passes 0-1 checks. Just a title or one-liner.
   - **Misclassified** — issue type appears wrong: a bug filed as a story (description mentions "broken", "error", "regression"), a task that reads like an epic (lists multiple independent deliverables), a story with no user value (reads like a technical task).

   Present the breakdown (template below), then AskUserQuestion: **Bare minimum first** (recommended — highest impact) / **Needs refinement first** — quicker wins, less conversation per ticket / **Misclassified first** — fix categorization before enriching / **Let me pick specific tickets** — I'll name the ones I want to work on.
2. **Phase 2 — Guided Enrichment.**
   - **Bare-minimum tickets (one at a time):**
     1. Present with the single-ticket format plus full current description.
     2. Ask one at a time: **Q1:** "What problem does [PROJ-123] solve? Who's affected?" **Q2:** "What does 'done' look like for this ticket? What would you check to know it's working?" **Q3:** "Is there anything this ticket should NOT include — scope boundaries?" If vague, push once: "Can you give me a specific example of what this looks like from the user's perspective?"
     3. Draft. **Product-only mode:** User story "As a [role], I want [action] so that [outcome]"; Description 2-3 sentences (problem and proposed solution in user terms); Acceptance criteria 3-5 testable bullets ("Given/When/Then" or simple "Should" statements); Metadata corrections (suggest priority, correct type if misclassified, suggest component/label if obvious). **Argus-enhanced mode (in addition):** `mcp__argus__search_index` with terms from the summary and PM's description; if flows found, `mcp__argus__trace_flow` on the most relevant entry point; add a **Technical Context** section in PM language: "This ticket touches the [user-facing area] flow. Changes would affect [list of service areas in PM language]. Related flows: [brief description of upstream/downstream impacts]." Do NOT include file paths, function names, node types, or repo names unless the PM explicitly asks.
     4. Present the proposed update (template below).
     5. AskUserQuestion: **Approve** — Update the ticket with this content / **Edit** — I'll tell you what to change / **Skip** — Come back to this one later. Edit -> ask what to change, adjust, re-present.
     6. If write access allows: `mcp__atlassian__editJiraIssue` with `contentFormat: "markdown"` for the description and the `fields` parameter for priority/type/label. Otherwise present the content as a copy-paste block.
   - **Needs-refinement tickets (batches of 3-5):** show the batch table noting what each lacks (e.g. "no acceptance criteria", "no priority set", "vague description"); draft the missing pieces from the existing description (less PM input); present and ask for batch approval or individual review.
   - **Bulk context:** if the PM says e.g. "all tickets in component X are about the same feature", apply it across relevant tickets; don't re-ask per ticket.
   - **Misclassified tickets:** present with reasoning (e.g. "PROJ-123 is filed as a Story but the description says 'broken', 'error', and references a regression — this looks like a Bug."); AskUserQuestion: **Reclassify to [suggested type]** — Change the issue type / **Keep current type** — The classification is correct / **Skip** — Decide later. If reclassifying and write access allows: `mcp__atlassian__editJiraIssue` with `fields: { "issuetype": { "name": "[new type]" } }`.
3. **Phase 3 — Linking Pass** across enriched tickets:
   1. **Same feature area** — group tickets referencing the same feature, component, or user workflow; full writes -> `mcp__atlassian__createIssueLink` type "Relates"; other modes -> list recommended links.
   2. **Epic candidates** — shared initiative with no parent epic: "These [N] tickets seem related to [feature area]. Should they be grouped under an epic?" Yes + full writes -> create epic via `mcp__atlassian__createJiraIssue` and link children; yes + not full writes -> recommend the PM create it and list tickets to add.
   3. **Blocking relationships** — descriptions referencing dependencies ("requires PROJ-456 first", "blocked until X is done") -> suggest Blocks links; call `mcp__atlassian__getIssueLinkTypes` to verify link type names before creating links.
4. **Phase 4 — Enrichment Summary** (template below). Closing: under `backlog-groom` -> "Enrichment is done. The tickets are ready for prioritization."; standalone -> "Good enrichment session. When you're ready to prioritize, `backlog-prioritize` will help you rank and organize the backlog."

**Hard rules (verbatim):**
```
<HARD-GATE>
Do NOT discuss implementation details, code architecture, technical stack, or file structures. This skill works at the product/project management level. When using argus-enhanced mode, translate all technical context into PM language — service areas and user-facing flows, not file paths or function names.
</HARD-GATE>
```

**Output format.** Assessment breakdown (verbatim):
```
> "I've assessed [N] tickets for dev-readiness:
>
> - **Ready** ([X]): These need no enrichment — good to go
> - **Needs refinement** ([Y]): Some structure but missing pieces
> - **Bare minimum** ([Z]): Just a title or one-liner — need the most work
> - **Misclassified** ([W]): Appear to be the wrong issue type
>
> I'd recommend starting with **bare minimum** tickets — that's where enrichment has the biggest impact. Where would you like to start?"
```
Proposed update (verbatim):
```
> "Here's what I'd update on **PROJ-123**:
>
> **Summary:** [updated if needed]
> **Type:** [corrected if misclassified]
> **Description:**
> [full enriched description]
>
> **Acceptance Criteria:**
> - [criterion 1]
> - [criterion 2]
> - ...
>
> **Priority:** [suggested]
>
> Does this look right, or should I adjust anything?"
```
Summary (verbatim):
```
> "**Enrichment complete.** Here's what happened:
>
> - **Enriched:** [X] tickets now have acceptance criteria and clear descriptions
> - **Already ready:** [Y] tickets needed no changes
> - **Reclassified:** [Z] tickets had their issue type corrected
> - **Skipped:** [W] tickets deferred for later
> - **Links added:** [V] relationships created between related tickets
>
> **Still needs PM input:** [list any tickets that couldn't be enriched without more context]"
```
If argus mode was used, append: `> "**Feature areas touched:** [list of user-facing areas identified via argus]"`.

**Acceptance check:**
- Ticket with summary only, no priority, no links -> classified Bare minimum; agent asks Q1-Q3 one at a time, drafts user story + 3-5 ACs, presents the update block; on Approve in light-writes mode calls `editJiraIssue` with `contentFormat: "markdown"`.
- Argus-enhanced mode -> draft includes a Technical Context paragraph naming user-facing areas only, with no file paths, function names, node types, or repo names; read-only mode outputs a copy-paste block instead of writing.

---

### Skill: backlog-prioritize

**Files to create:** `skills/backlog-prioritize/SKILL.md`.

**Frontmatter (verbatim):**
```yaml
---
name: backlog-prioritize
description: "Jira backlog prioritization for PMs and CPO. Applies RICE, Value/Effort, MoSCoW, or custom frameworks to score and rank tickets. Use when someone says 'prioritize the backlog', 'rank these tickets', 'score the backlog', or 'what should we work on next'."
---
```

**Title / intro:** `# Backlog Prioritize — Rank and Organize the Backlog`. "Apply a prioritization framework to produce a ranked backlog the team can plan from. The PM chooses the framework, scores with AI-suggested starting values, and adjusts until the ranking reflects their actual priorities."

**Purpose / fires when:** prioritize / rank / score / "what should we work on next".

**Procedure:**

0. **Setup** — read `session-config.md`, run its startup protocol, SKIP step 7. Read `jira-helpers.md`.
1. **Phase 1 — Framework Selection.** AskUserQuestion "Which prioritization approach should we use?":
   - **RICE** — Score by Reach, Impact, Confidence, and Effort. Best for balancing multiple initiatives with different audiences.
   - **Value/Effort** — Simple 2×2 matrix (high/low value vs. high/low effort). Good for quick sessions.
   - **MoSCoW** — Categorize as Must/Should/Could/Won't. Good for release-scoping or when there's a hard deadline.
   - **Custom** — You define the dimensions and how to weight them.

   Calibration per framework:
   - RICE: "Quick calibration — when I say 'reach' for your team, are we talking about number of customers affected, frequency of use, or something else?" Let the PM define reach (and impact). Confirm scale: "I'll use a 1-3 scale for impact: 1 = minor improvement, 2 = noticeable improvement, 3 = game-changer. Work for you?"
   - Value/Effort: "For the effort axis, should I estimate based on ticket descriptions alone, or do you want to factor in argus technical context?" If yes and argus is available, use `mcp__argus__search_index` and `mcp__argus__trace_flow` to estimate complexity (number of services touched, flow depth) as an effort proxy.
   - MoSCoW: "Is there a deadline or release driving these priorities? That helps me suggest starting categories."
   - Custom: "What dimensions matter for your prioritization? Name them and I'll build a scoring model." Follow up to define each dimension's scale and relative weight.
2. **Phase 2 — Scoring.** Key principle: **suggest a starting score and let the PM adjust**, rather than asking the PM to generate scores from scratch.
   - **RICE** (per ticket, or batched by epic/feature area for 30+ tickets):
     - Starting signals: **Reach** — number of watchers + linked issues as a proxy; higher if components/labels suggest a widely-used feature. **Impact** — from priority: Blocker/Critical → 3, High → 2, Medium/Low → 1. **Confidence** — well-defined (acceptance criteria, clear description) → high; bare descriptions → low. **Effort** — argus flow complexity if available, else estimated from description scope.
     - Formula: RICE = (Reach × Impact × Confidence) / Effort. Present (verbatim example):
       ```
       > **PROJ-123** — Fix upload timeout
       > Suggested: Reach 2 · Impact 3 · Confidence 2 · Effort 1
       > RICE Score: (2 × 3 × 2) / 1 = **12**
       >
       > Reasoning: High-priority bug affecting a core workflow. Well-described but missing some acceptance criteria.
       ```
     - Ask "Adjust any scores, or does this look right?"; recalculate and confirm on adjustment.
     - 30+ tickets: group by epic/feature area; set reach/impact at group level ("These 8 tickets are all about the document upload flow. Shall I score them as a group — reach 3, impact 2 — then we can adjust individual ones that differ?"); individual tickets inherit the group baseline.
     - **Floors:** if the PM says "all Blocker/Critical tickets are automatic top priority", set scores that place them on top (e.g. Reach 3, Impact 3, Confidence 3, Effort 1) and skip individual scoring for those.
   - **Value/Effort** — quadrant matrix (verbatim):
     ```
     > | | Low Effort | High Effort |
     > |---|---|---|
     > | **High Value** | Quick Wins | Big Bets |
     > | **Low Value** | Fill-ins | Reconsider |
     ```
     Per ticket/batch: suggest a quadrant from priority + description scope (argus flow complexity informs effort if available); ask "Does [PROJ-123] — [summary] belong in [suggested quadrant]?"; PM confirms or moves. Present the filled matrix at the end.
   - **MoSCoW** — group by feature area; starting categories: Priority Blocker/Critical → Must Have; Priority High with deadline mentions → Should Have; Priority Medium → Could Have; Priority Low or deferred mentions → Won't Have (this time). PM categorizes each group, then refines individual tickets. Prompt (verbatim):
     ```
     > "I've grouped your tickets by feature area. For the **document upload** group ([N] tickets):
     > Suggested: **Should Have** — high-priority but no hard deadline mentioned.
     >
     > - **Must Have** — Can't ship without this
     > - **Should Have** — Important but not a blocker
     > - **Could Have** — Nice to have this release
     > - **Won't Have** — Not this time"
     ```
   - **Custom** — per dimension suggest a score from ticket data; apply the PM's weights for a composite score; present and let the PM adjust.
   - **Skip scoring shortcut:** "just rank by priority field" -> sort by Jira priority (Blocker > Critical > High > Medium > Low), then by age within each level; present the ranked list without per-ticket scoring.
3. **Phase 3 — Rank and Organize.** Produce the ranked list (template below); ask "Does this ranking feel right? You can adjust — say 'move X above Y' or 'swap these two' and I'll update." Iterate until confirmed.
4. **Phase 4 — Apply to Jira** by write-access level:
   - **Read-only:** summary report: "Here's your ranked backlog. Recommended priority values per ticket:" followed by "- PROJ-123: Priority → Highest" style lines.
   - **Light writes:** (1) update priority via `mcp__atlassian__editJiraIssue` with `fields: { "priority": { "name": "[priority]" } }`; (2) add a label to the existing labels array in format `backlog-groomed-YYYY-MM` (e.g. `backlog-groomed-2026-05`); (3) confirm "Updated priorities and added grooming labels to [N] tickets."
   - **Full writes:** all of the above, plus (1) create epics and link via `mcp__atlassian__createIssueLink` if tickets should be grouped; (2) if the PM provides sprint velocity, suggest sprint assignments ("Based on [X] points per sprint, here's how these could be distributed: - Sprint 1: PROJ-123, PROJ-456 ([Y] points) - Sprint 2: ..."); (3) add ranking comments to top-priority tickets: "Ranked #[N] during backlog grooming [date]. [brief rationale]."
5. **Phase 5 — Prioritization Summary** (template below). Under `backlog-groom`: final stage; the orchestrator handles the overall summary. Standalone: "Good prioritization session. If you haven't triaged recently, `backlog-triage` can help clean out stale tickets first. If tickets need more detail, `backlog-enrich` adds acceptance criteria."

**Hard rules (verbatim):**
```
<HARD-GATE>
Do NOT discuss implementation details, code architecture, technical stack, or file structures. This skill works at the product/project management level. Use PM-appropriate language throughout.
</HARD-GATE>
```

**Output format.** Ranked list (verbatim):
```
> "**Ranked Backlog — Top 15:**
>
> | Rank | Ticket | Summary | Score | Category |
> |------|--------|---------|-------|----------|
> | 1 | PROJ-123 | Fix upload timeout | 12 | Quick Win |
> | 2 | PROJ-456 | Add export button | 10 | Big Bet |
> | ... | ... | ... | ... | ... |
>
> **Clusters** (tickets that should be done together):
> - PROJ-123 + PROJ-789: both touch the upload flow
> - PROJ-456 + PROJ-012: export feature dependencies
>
> **Dependencies:**
> - PROJ-789 should come before PROJ-123 (PROJ-789 fixes the underlying timeout; PROJ-123 adds retry UI)
>
> **Quick Wins** (high value, low effort):
> - PROJ-345, PROJ-678 — consider for the next sprint"
```
Summary (verbatim):
```
> "**Prioritization complete.**
>
> **Framework used:** [RICE/Value-Effort/MoSCoW/Custom]
>
> **Top 10:**
> [numbered list with ticket key, summary, and score]
>
> **Distribution:**
> - [For RICE: score ranges and ticket counts]
> - [For Value/Effort: count per quadrant]
> - [For MoSCoW: count per category]
>
> **Recommended next sprint:** [top N tickets by score, noting clusters and dependencies]
>
> **Reconsider closing:** [tickets that scored very low — consider looping back to triage]"
```

**Acceptance check:**
- RICE chosen, a High-priority well-described ticket -> agent suggests Impact 2 from priority, shows the (R × I × C) / E computation, and asks "Adjust any scores, or does this look right?" before moving on; with 40 tickets it first offers group-level reach/impact baselines.
- Light-writes session after ranking is confirmed -> priorities updated via `editJiraIssue` and label `backlog-groomed-YYYY-MM` appended to existing labels; no epics created and no ranking comments (full writes only).

---

### Skill: backlog-groom

**Files to create:** `skills/backlog-groom/SKILL.md`.

**Frontmatter (verbatim):**
```yaml
---
name: backlog-groom
description: "Full Jira backlog grooming pipeline for PMs and CPO. Runs triage, enrichment, and prioritization in sequence with early exits. Use when someone says 'groom the backlog', 'backlog grooming', 'let's work through the backlog', or 'full backlog cleanup'."
---
```

**Title / intro:** `# Backlog Groom — Full Backlog Pipeline`. "Run the complete backlog grooming pipeline: triage → enrich → prioritize. One conversation, clean and ranked backlog out the other end. Each phase can be stopped early."

**Purpose / fires when:** groom / full backlog cleanup requests.

**Procedure:**

0. **Setup** — read `session-config.md` and follow its **full** startup protocol, INCLUDING step 7 (Enrichment Mode). Read `jira-helpers.md`. Record the starting ticket count from the working-set preview.
- **Early Exit (any stage transition):** on "that's enough", "let's stop here", "I have what I need", or similar: (1) "Good stopping point."; (2) present a summary of what was accomplished (Phase 4 format adapted to completed stages only); (3) "When you're ready to continue, you can invoke `backlog-enrich` (or `backlog-prioritize`) directly to pick up where we left off."
1. **Phase 1 — Triage.** Read `skills/backlog-triage/SKILL.md`; follow Phase 1-3 (Backlog Snapshot, Category-by-Category Triage, Triage Summary). Skip its Setup/startup protocol (session config is shared) and its standalone closing message. Record the post-triage count, then AskUserQuestion: "Triage is done. Ready to move on to enriching the surviving tickets?" — **Continue to enrichment** — Let's make these tickets dev-ready / **Stop here** — Good session, I'll pick up enrichment later. Check for early exit.
2. **Phase 2 — Enrich.** Read `skills/backlog-enrich/SKILL.md`; follow Phase 1-4 (Quality Assessment, Guided Enrichment, Linking Pass, Enrichment Summary). Skip Setup, the Enrichment Mode question (already asked), and the standalone closing. Operates on the post-triage working set. Then AskUserQuestion: "Enrichment is done. Ready to prioritize?" — **Continue to prioritization** — Let's rank the backlog / **Stop here** — Good session, I'll prioritize later. Check for early exit.
3. **Phase 3 — Prioritize.** Read `skills/backlog-prioritize/SKILL.md`; follow Phase 1-5 (Framework Selection, Scoring, Rank and Organize, Apply to Jira, Prioritization Summary). Skip Setup and the standalone closing. Operates on the enriched ticket set.
4. **Phase 4 — Final Summary** (template below).

**Hard rules (verbatim):**
```
<HARD-GATE>
Do NOT discuss implementation details, code architecture, technical stack, or file structures. This skill works at the product/project management level. Use PM-appropriate language throughout.
</HARD-GATE>
```
Session configuration (cloud ID, project, write-access level, working set, enrichment mode) is set once and carried through all stages.

**Output format (final summary, verbatim):**
```
> "**Backlog grooming complete.**
>
> **Starting backlog:** [starting count] tickets
>
> **Triage results:**
> - Closed: [X] tickets
> - Merged: [Y] duplicates consolidated
> - Net reduction: [starting] → [post-triage count] ([percentage]%)
>
> **Enrichment results:**
> - Enriched: [X] tickets now dev-ready
> - Reclassified: [Y] issue types corrected
> - Links added: [Z] relationships created
>
> **Prioritization results:**
> - Framework: [RICE/Value-Effort/MoSCoW/Custom]
> - Top priorities: [top 5 tickets with summaries]
> - Quick wins identified: [count]
>
> **Final backlog:** [final count] tickets, ranked and ready for sprint planning.
>
> **Recommendations for next session:**
> - [any deferred tickets from triage]
> - [any tickets that still need PM input from enrichment]
> - [low-scoring tickets to reconsider]"
```

**Acceptance check:**
- "Groom the backlog" -> startup asks write-access level, working set, AND enrichment mode exactly once; triage runs; the triage summary ends with "Triage is done. The surviving tickets are ready for enrichment." (not the standalone line) followed by the Continue/Stop question.
- PM picks "Stop here" after triage -> "Good stopping point.", a triage-only summary, and the pointer to invoke `backlog-enrich` (or `backlog-prioritize`) directly; enrichment never begins.

---

## Gaps and discrepancies observed in the source

- RICE Reach uses "number of watchers", but neither the working-set preview nor enrich's `getJiraIssue` field list requests watchers; the agent must fetch it separately or fall back to linked issues.
- The RICE confidence scale is only described as high/low; the worked example uses numeric 2, implying 1-3 like impact. Effort scale is likewise unspecified beyond the example (Effort 1).
- `jira-helpers.md` gates write tools by unprefixed names while procedures call `mcp__atlassian__*`; treat them as the same tools.
