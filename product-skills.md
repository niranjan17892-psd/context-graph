# Rebuild Spec — Product Skills

Rebuild spec for the four product-focused skills (`discover`, `strategize`, `ux-review`, `refine`) and their shared resources (`skills/product-shared/`). A fresh agent given only this document should be able to recreate the skills with equivalent behavior.

Source files (all git-tracked; nothing else lives in these directories):

| Path | Size (bytes) |
|---|---|
| `skills/product-shared/base.md` | 4093 |
| `skills/product-shared/output-artifacts.md` | 4961 |
| `skills/discover/SKILL.md` | 4104 |
| `skills/strategize/SKILL.md` | 5957 |
| `skills/ux-review/SKILL.md` | 7231 |
| `skills/refine/SKILL.md` | 4978 |

## Family overview

- Audience: product managers and the CPO, not developers. The skills produce product thinking (problem, user, scope, risk, UX), never technical design.
- Pipeline: `discover` (problem) -> `strategize` (scope/risk) -> `ux-review` (experience). `refine` runs all three in one conversation. Dev handoff: PM produces a product spec -> developer invokes the `brainstorming` skill -> technical design -> `writing-plans` -> implementation.
- Every skill begins by reading `skills/product-shared/base.md` and running its startup protocol (which includes the explore/deliverable intent check). Every skill ends by reading `skills/product-shared/output-artifacts.md` only in deliverable mode.
- Shared-file references inside the skills use repo-relative paths (`skills/product-shared/base.md`, `skills/discover/SKILL.md`, etc.). Keep those exact paths.
- Interaction is via the `AskUserQuestion` tool, one question at a time, multiple choice preferred.
- External tools used: argus MCP (`search_index`, `get_flow_diagram`; optional), Atlassian MCP (`searchJiraIssuesUsingJql`, `searchConfluenceUsingCql`, `getConfluenceSpaces`, `createConfluencePage`, `getVisibleJiraProjects`, `createJiraIssue`, `createIssueLink`, `editJiraIssue`). Tool names are written unprefixed in these skills (unlike the backlog family, which uses `mcp__atlassian__*`).
- Every skill carries the same style of `<HARD-GATE>` block forbidding implementation discussion (wording varies slightly per skill; reproduced verbatim per skill below).

---

## Shared resource: `skills/product-shared/base.md`

Plain markdown, no frontmatter. Title: `# Product Skills — Shared Base`. Opening line states: read this file first before following the skill-specific workflow; it defines the voice, startup protocol, and interaction style shared by all product skills (discover, strategize, ux-review, refine). Sections separated by `---`.

### Section: Voice & Audience

"You are working with **product managers and executives**, not developers." Bullets (keep verbatim):

- No code references, no architecture jargon, no file paths, no repo names (unless the PM explicitly asks)
- Speak in terms of user outcomes, business value, and experience quality
- Be direct but collaborative — PMs own the decisions, you surface the tradeoffs
- One question at a time, prefer multiple choice, keep it focused
- Push for specifics when answers are vague — "customers" is not a user, "better experience" is not a success criterion
- Frame questions in outcome terms: "What should users be able to do?" not "What endpoints do we need?"

### Section: Working Directory

- PMs may invoke product skills from any of the 11 TCN repos or from no repo at all. Do not depend on a stable working directory.
- Product specs write to `docs/product-specs/` at the root of the git repo the PM is working in (find it with `git rev-parse --show-toplevel`).
- If the PM is not in a git repo, ask the PM where to save artifacts.
- Argus MCP tools are optional; when available they work cross-repo by design and are not affected by working directory.

### Section: Startup Protocol (run at the start of every product skill session)

1. **Read project context** — Read CLAUDE.md if it exists in the current directory.
2. **Ask what the PM is working on** — "What feature or area are you working on?"
3. **Intent check** — via AskUserQuestion (verbatim):
   > "Are you exploring an idea today, or building toward a deliverable?"
   >
   > - **Exploring** — We'll think through it together. No artifacts unless you ask.
   > - **Building a deliverable** — I'll track decisions and offer to create specs, Confluence pages, and Jira tickets at the end.

   Record the mode (explore or deliverable) for use at the end of the workflow.
4. **System context via argus** — Only if argus MCP tools (e.g. `search_index`) are available in this session; if not, skip silently and continue from the PM's description. Otherwise:
   1. Call `search_index` with the feature/area name to find related nodes in the TCN codebase graph.
   2. If results are found, call `get_flow_diagram` for the most relevant entry point.
   3. Summarize in PM language: "Here's what I understand about how this area works today: ..." — relabel technical terms ("User submits form" not "mutation CreateAgreement"); show the Mermaid diagram with user-facing labels; keep it to 3-5 sentences plus the diagram.
   4. If no results: "I don't see existing flows for this area in the codebase. This might be a new capability — we'll design from scratch."

   **Argus integration rules:** always summarize argus output in PM language; never expose node types, edge types, or technical repo names unless the PM asks; use argus to inform your questions, not to overwhelm with system details.
5. **Check Jira and Confluence** (Atlassian MCP):
   1. `searchJiraIssuesUsingJql` with a query related to the feature area — look for open stories, epics, or bugs.
   2. `searchConfluenceUsingCql` to find existing specs or documentation.
   3. If related items are found: "I found some related work in Jira/Confluence: [brief summary]. Want me to pull in any of these for context?"
   4. If nothing is found, proceed without mentioning it.

### Section: Interaction Style

- One question at a time via AskUserQuestion
- Prefer multiple choice when possible — easier for PMs to answer
- Always include an "Other" escape hatch (AskUserQuestion provides this automatically)
- After each PM answer, briefly acknowledge what you heard before asking the next question
- If the PM gives a vague answer, push once for specifics. If they push back, accept their framing and move on.

---

## Shared resource: `skills/product-shared/output-artifacts.md`

Plain markdown, no frontmatter. Title: `# Product Skills — Output Artifacts`. Opening line: read at the end of a product skill workflow when the PM chose **deliverable mode**; skip entirely in **explore mode**.

### Section: Artifact Menu

Ask via AskUserQuestion with `multiSelect: true` (all optional; PM may pick some, all, or none):

> "Which artifacts should I create from this session?"
>
> - **Markdown spec** — Save a structured product spec to `docs/product-specs/` in the current repo
> - **Confluence page** — Create a page in your Confluence space
> - **Jira ticket** — Create a story, epic with child stories, or bug ticket
> - **None right now** — Skip artifact creation

### Section: Markdown Spec

Write to `docs/product-specs/YYYY-MM-DD-<feature-name>.md` using this template (verbatim):

```markdown
# [Feature Name] — Product Spec

**Date:** YYYY-MM-DD
**Author:** [PM name or "Product Team"]
**Status:** Draft | Reviewed | Ready for Dev
**Confluence:** [URL if created, otherwise remove this line]
**Jira:** [ticket key(s) if created, otherwise remove this line]

---

## Problem Statement
[One paragraph summarizing the problem from the discovery phase]

## Target User
[The specific user persona identified during discovery]

## Current State
[Status quo — what users do today, with argus flow diagram if applicable]

## Proposed Change
[High-level description of what's being built or changed]

## Scope

### In Scope
- [Bulleted list of what's included in this version]

### Out of Scope
- [Bulleted list of what's deliberately excluded]

### Deferred to Future Version
- [Things that were discussed but pushed to a later release]

## Success Criteria
- [Measurable outcomes that indicate this shipped well]

## Risks
- [Identified risks with mitigation notes]

## UX Decisions
[From ux-review phase, if applicable. Screen descriptions, flow decisions, error states, empty states. Remove this section if no UX review was done.]

## Review History
- [Date] — Discovery completed
- [Date] — Strategy review (mode: expand/hold/reduce)
- [Date] — UX review (design completeness: X/10 → Y/10)
```

Rules: populate every section from the conversation; if a section doesn't apply (e.g. no UX review), remove it rather than leave it empty; if a spec already exists for this feature (from a prior skill run), update it rather than create a new file.

### Section: Confluence Page

1. AskUserQuestion: "Which Confluence space should this go in? And is there a parent page it should live under?" — PM types the space key and parent page title.
2. `getConfluenceSpaces` to verify the space exists.
3. If a parent was given, `searchConfluenceUsingCql` with `title = "[parent page title]" and space = "[space key]"`.
4. `createConfluencePage` with `spaceId` (from space lookup), `title`: "[Feature Name] — Product Spec", `parentId` (if specified), `body`: same content as the markdown spec, formatted for Confluence (wiki markup or HTML as the MCP tool requires).
5. Report: "Confluence page created: [URL]".

### Section: Jira Ticket

1. AskUserQuestion: "What type of Jira ticket?" — **Single story** (one story with the spec as the description) / **Epic with child stories** (an epic for the feature, with child stories for each scope item) / **Bug ticket** (a bug report, for "fix this existing behavior" features).
2. Ask: "Which Jira project?" — PM types or picks from `getVisibleJiraProjects`.
3. Single story: `createJiraIssue` type Story; Summary "[Feature Name]"; Description = problem statement + scope + success criteria from the spec; add the Confluence URL if one was created.
4. Epic with child stories: `createJiraIssue` type Epic, summary "[Feature Name]"; for each "In Scope" item, `createJiraIssue` type Story; link each child to the epic via `createIssueLink`.
5. Bug ticket: `createJiraIssue` type Bug; Summary "[Feature Name] — [brief problem description]"; Description = problem statement + current behavior + expected behavior.
6. Report: "Jira ticket created: [KEY-123]".

### Section: Cross-Linking

When multiple artifacts are created in the same session:
- Confluence + Jira: add the Confluence URL to the Jira ticket description, and add a comment or link on the Confluence page referencing the Jira key.
- Markdown spec also created: add the Confluence URL and Jira key(s) to the spec's metadata header.
- Use `editJiraIssue` to update descriptions with cross-links after all artifacts are created.

---

### Skill: discover

**Files to create:** `skills/discover/SKILL.md` (single file).

**Frontmatter (verbatim):**
```yaml
---
name: discover
description: "Problem discovery and product brief creation for PMs and CPO. Explores user needs, status quo, and success criteria before solutions. Use when someone says 'I have a feature idea', 'we need to build', 'let me describe what we want', or wants to flesh out a product concept."
---
```

**Title / intro:** `# Discover — Product Discovery`. "Help product managers articulate a clear problem statement, identify their target user, map the status quo, and define scope before jumping to solutions."

**Purpose / fires when:** a PM has a feature idea or wants to flesh out a product concept. Produces a confirmed problem/user/scope brief.

**Procedure:**

0. **Setup** — Read `skills/product-shared/base.md` and follow its startup protocol before proceeding.
1. **Phase 1 — Problem Articulation.** Ask one at a time via AskUserQuestion; vague answers get one follow-up push.
   - **Q1: What's the problem?** — "Describe what's not working or what's missing. Who told you about this — a customer, internal team, data?" Vague (e.g. "we need a better onboarding") -> push once: "Can you tell me about a specific moment where this fell short? A customer complaint, a support ticket, a metric that dropped?"
   - **Q2: Who feels it most?** — "Name the actual user role. Not 'customers' — is it the attorney reviewing a contract? The paralegal uploading documents? The admin configuring workflows? Who specifically is affected?" Category-level ("enterprises", "our users") -> push once: "If you had to pick one person at one company who'd be most relieved if we fixed this, who would it be?"
   - **Q3: What do they do today?** — "How are users solving this now, even badly? Workarounds, manual steps, complaints to support, spreadsheets, emails?" If "nothing — there's no solution", probe: "So nobody is working around this at all? That might mean the pain isn't sharp enough to act on yet, or we haven't found the right angle. What makes you think now is the right time?"
   - **Q4: What does success look like?** — "If we nail this, what changes for the user? What would you measure to know it worked? A number going up, a complaint going away, a workflow getting shorter?"
2. **Phase 2 — Scope Definition.**
   - **Q5: Narrowest wedge** — "What's the smallest version of this that delivers real value? Not the full vision — the thing we could ship first that users would actually notice and appreciate."
   - **Q6: What's NOT in scope?** — "What are we deliberately leaving out of this version? What will people ask for that we'll say 'not yet' to?"
3. **Phase 3 — System Context.** If the feature touches existing functionality: call `search_index` with terms from the PM's description; if relevant flows are found, `get_flow_diagram` for the best match; present in PM language ("Here's how this area works today: [diagram with user-facing labels]"); note affected flows ("These parts of the system would need to change: ..."). If entirely new: "This looks like a new capability — no existing flows to map. We're designing from scratch."
4. **Phase 4 — Synthesis.** Summarize back (template below), ask "Does this capture it? Anything I'm missing or got wrong?", iterate until the PM confirms.
5. **Output.** Deliverable mode: read and follow `skills/product-shared/output-artifacts.md`. Explore mode: say "Good thinking session. When you're ready to go deeper, `strategize` will pressure-test the scope, and `ux-review` will work through the user experience."

**Hard rules (verbatim):**
```
<HARD-GATE>
Do NOT discuss implementation details, code architecture, technical stack, or file structures. This skill produces product thinking, not technical designs. The brainstorming skill is where technical decisions happen.
</HARD-GATE>
```

**Output format (synthesis, verbatim):**
```
> **Problem:** [one paragraph]
> **Target user:** [specific role and context]
> **Current state:** [what users do today]
> **Proposed change:** [high-level what we'd build]
> **Success criteria:** [how we'd measure it]
> **In scope:** [bulleted list]
> **Out of scope:** [bulleted list]
```

**Acceptance check:**
- PM says "I have a feature idea: better onboarding", picks Exploring. Agent asks Q1, gets a vague answer, pushes exactly once with the "specific moment" prompt, then proceeds through Q2-Q6 one question at a time, presents the 7-field synthesis, and on confirmation closes with the explore-mode message naming `strategize` and `ux-review` — no files written.
- Same flow in deliverable mode: after confirmation, the multi-select artifact menu appears; choosing Markdown spec writes `docs/product-specs/YYYY-MM-DD-<feature>.md` at the current git repo root (or asks where if not in a repo), with no UX Decisions section.

---

### Skill: strategize

**Files to create:** `skills/strategize/SKILL.md`.

**Frontmatter (verbatim):**
```yaml
---
name: strategize
description: "Strategic scope review for feature plans. Challenges premises, assesses scope in three modes (expand/hold/reduce), identifies risks. Use when someone says 'review this plan', 'is the scope right', 'think bigger', 'strip it down', or 'pressure-test this'."
---
```

**Title / intro:** `# Strategize — Strategic Scope Review`. "Challenge the premises, scope, and risks of a feature plan. Three modes: expand (think bigger), hold (make it bulletproof), reduce (strip to essentials)."

**Purpose / fires when:** a PM wants a plan reviewed, scope pressure-tested, expanded, or stripped down.

**Procedure:**

0. **Setup** — Read `skills/product-shared/base.md` and follow its startup protocol.
1. **Prerequisite Check** (after startup): look in `docs/product-specs/` (location per base.md) for a spec matching the feature, and check the conversation for prior `discover` context.
   - No brief -> AskUserQuestion: "I don't see an existing product brief for this feature. Want to run `discover` first to articulate the problem, or should I work from what you describe now?" Options: **Run discover first** — We'll come back to strategy review after / **Proceed from here** — I'll work with what you tell me.
   - If the PM chooses discover: say "Invoke the `discover` skill and come back to `strategize` when you're ready." Then **stop** (early exit).
   - Brief exists -> read it and summarize: "I found your product brief for [feature]. Here's what it says: [1-2 sentence summary]. I'll review against this."
2. **Phase 1 — Mode Selection.** AskUserQuestion: "How should I approach this review?"
   - **Expand** — Think bigger. What's the 10x version? What opportunities are we leaving on the table?
   - **Hold** — The scope is right. Make it bulletproof. Find every gap and risk.
   - **Reduce** — Strip it down. What's the minimum that delivers the core value?
3. **Phase 2 — Premise Challenge** (regardless of mode; one at a time; push once on each; if answers are solid and specific, acknowledge and move on — don't belabor):
   - **P1: Is this the right problem?** — "You said the problem is [X]. What if the real problem is [Y — a reframing based on what you know]? What evidence would change your mind?"
   - **P2: Is this the right solution shape?** — "You're proposing [a new feature / a rewrite / an enhancement]. Could this be solved with a configuration change? A workflow adjustment? Removing something instead of adding?"
   - **P3: Is this the right time?** — "What's the cost of doing this in 3 months instead of now? What else competes for the same resources?"
4. **Phase 3 — Scope Assessment** (only the chosen mode):
   - **Expand** — ask one at a time: "What would make this a feature users talk about to other users?" / "What adjacent problems could we solve with minimal extra effort?" / "What's the competitive angle — does this create distance from alternatives?" For each opportunity, AskUserQuestion:
     > "Here's an opportunity: [description]
     > Estimated effort: [rough size — small/medium/large]
     > User impact: [what changes for the user]
     >
     > - **Add it** — Include in this version
     > - **Defer it** — Good idea, but save for a future version
     > - **Skip it** — Not worth pursuing"
   - **Hold** — walk one at a time: "Walk me through the user journey end to end. What happens when [edge case you identified]?" / "What are the failure modes? User hits an error — what do they see? Can they recover?" / "What dependencies does this have on other teams or systems?" / "What's the rollback plan if this doesn't work as expected?" For each gap found, AskUserQuestion with a recommendation for how to address it.
   - **Reduce** — ask one at a time: "Which parts of this could ship as a fast-follow instead of V1?" / "If you could only ship ONE piece of this, which piece delivers the most value?" / "What looks necessary but is actually nice-to-have?" For each reduction opportunity, AskUserQuestion:
     > "[Component/feature] — currently in scope
     >
     > - **Keep it** — Essential for V1
     > - **Defer it** — Ship as fast-follow after V1 launches
     > - **Cut it** — Not needed"
5. **Phase 4 — Risk Identification** (any mode, one at a time):
   - **User risk:** "What happens if users don't adopt this? How would we know early — in the first week, not the first quarter?"
   - **Dependency risk:** "What external teams, APIs, or systems does this depend on? What happens if any of them slip?"
   - **Scope creep risk:** "Where is this most likely to grow beyond what we planned? What's the first thing someone will ask to add?"
6. **Phase 5 — Synthesis.** Present template below; ask "Does this look right? Anything to adjust?"
7. **Output.** Deliverable: follow `skills/product-shared/output-artifacts.md`; if an existing spec was found during the prerequisite check, update it rather than create a new one; add review history entry "[date] — Strategy review (mode: [expand/hold/reduce])". Explore: "Good session. `ux-review` is a natural next step if this feature has user-facing screens, or hand the spec to a developer for `brainstorming` when you're ready to build."

**Hard rules (verbatim):**
```
<HARD-GATE>
Do NOT discuss implementation details, code architecture, technical stack, or file structures. This skill reviews product strategy, not technical design. The brainstorming skill is where technical decisions happen.
</HARD-GATE>
```
Mode is always chosen by the PM (never auto-decided).

**Output format (synthesis, verbatim):**
```
> **Original scope:** [what the PM started with]
> **Reviewed scope:** [what changed — additions, reductions, or confirmed as-is]
> **Key decisions:** [bulleted list of choices made during the review]
> **Risks:** [identified risks with mitigation notes]
> **Recommended next steps:** [what should happen next]
```

**Acceptance check:**
- No spec in `docs/product-specs/`, no prior discover in conversation, PM picks "Run discover first" -> agent says the exact "Invoke the `discover` skill..." line and stops without asking mode.
- Spec exists, PM picks Reduce -> agent summarizes the brief, runs P1-P3, asks only the three Reduce questions with Keep/Defer/Cut decisions, the three risk questions, presents the 5-field synthesis, and in deliverable mode updates the existing spec and appends "— Strategy review (mode: reduce)".

---

### Skill: ux-review

**Files to create:** `skills/ux-review/SKILL.md`. Depends on existing `skills/brainstorming/visual-companion.md` and `skills/brainstorming/scripts/start-server.sh` (part of the absorbed brainstorming skill; not recreated here).

**Frontmatter (verbatim):**
```yaml
---
name: ux-review
description: "UX review for feature plans with visual companion. Reviews user flows, empty states, error states, edge cases, accessibility, and interaction patterns. Use when someone says 'review the UX', 'what should this look like', 'walk through the user experience', or 'design review'."
---
```

**Title / intro:** `# UX Review — User Experience Review`. "Review a feature plan from the user experience perspective. Rate design completeness, map user journeys, review seven UX dimensions, and optionally generate visual wireframes via the brainstorming visual companion."

**Purpose / fires when:** a PM wants the UX of a feature plan reviewed or asks what something should look like.

**Procedure:**

0. **Setup** — Read `skills/product-shared/base.md` and follow its startup protocol.
1. **Prerequisite Check:** look in `docs/product-specs/` for a matching spec; check conversation for prior `discover`/`strategize` context. If none, AskUserQuestion: "I don't see an existing product brief for this feature. A UX review works best with a clear problem statement and scope. Want to run `discover` first, or should I work from what you describe?" Options: **Run discover first** / **Proceed from here**. (Unlike strategize, the skill text does not spell out a stop instruction for "Run discover first".)
2. **UI Scope Gate (early exit):** if the feature has no user-facing surface (pure backend, data migration, API-only with no UI consumer), exit with: "This feature doesn't appear to have user-facing screens or interactions. A UX review isn't applicable here. Consider `strategize` for scope review instead."
3. **Phase 1 — Design Completeness Assessment.** Rate the plan 0-10 and explain. Examples: "This plan is a **3/10** on design completeness — it describes what the system does but never specifies what the user sees, how they interact with it, or what happens when things go wrong." / "This plan is a **7/10** — good flow descriptions and clear user actions, but missing empty states, error recovery, and mobile behavior." Then describe concretely what a 10 looks like for THIS feature (e.g. "A 10 for this feature would include: wireframes for each screen, defined empty states for [X and Y], error messages for [these failure modes], and responsive behavior for [these breakpoints].").
4. **Phase 2 — Visual Companion Offer.** This offer MUST be its own message, not combined with other content:
   > "Some of what we're reviewing will be easier if I can show you wireframes and flow diagrams in a browser. Want me to start the visual companion? (Requires opening a local URL)"

   If accepted: read `skills/brainstorming/visual-companion.md`; start the server with `skills/brainstorming/scripts/start-server.sh`; use it for wireframes, user flow diagrams, and screen mockups; use the terminal for conceptual questions. If declined: text-only descriptions.
5. **Phase 3 — User Journey Mapping.** For each key flow: (1) map steps in PM language: "User opens [screen] → sees [content] → clicks [action] → system does [thing] → user sees [result]"; (2) if the companion is running, generate a simple, clean HTML wireframe per key screen (structure, not pixel-perfect); (3) after each flow ask "Does this match what you're imagining? Anything that should work differently?"; (4) use argus to check whether the flow touches existing system paths and show where the new experience connects.
6. **Phase 4 — Seven Dimensions Review.** One at a time; for each: rate 0-10, explain the gap, propose a fix/decision, ask via AskUserQuestion if the PM agrees. Dimension prompts (verbatim):
   1. **Visual Hierarchy** — "What does the user see first, second, third on this screen? Does the most important action stand out? Is there a clear primary action vs. secondary actions?"
   2. **Empty States** — "What does this screen look like with zero data? First-time user who hasn't created anything yet? Search with no results? What do we show, and does it guide the user toward the next action?"
   3. **Error States** — "What happens when something goes wrong? Network error, validation failure, permission denied, timeout. What does the user see? Can they recover without starting over? Is the error message helpful or generic?"
   4. **Edge Cases** — "What if the input is 200 characters long? What if there are 10,000 results? What if the user double-clicks the submit button? What if they navigate away mid-action and come back?"
   5. **Responsive Behavior** — "How does this work on mobile? Tablet? Does the layout need to change? Are touch targets large enough? Does anything critical get hidden behind a hamburger menu?"
   6. **Accessibility** — "Can this be used with keyboard only? Does it work with a screen reader? Is there sufficient color contrast? Are interactive elements labeled? Are touch targets at least 44px?"
   7. **Interaction Patterns** — "What loading states do we need? Are there confirmation dialogs for destructive actions? Is there undo support? Does the form auto-save or require explicit save? What feedback does the user get after an action?"

   Skip dimensions that genuinely don't apply, stating which and why (e.g. "Skipping responsive behavior — this is a desktop-only admin tool based on your earlier description.").
7. **Phase 5 — UX Principles Check.** Final scan; report findings, not the principles themselves:
   - **Don't make me think** (Krug) — flag any screen where the user would need to stop and figure out what to do
   - **Users scan, they don't read** — flag dense text blocks, unclear visual hierarchy, or buried actions
   - **Goodwill reservoir** — flag anything punishing: required fields that could be optional, unnecessary confirmations, lost work on navigation
   - **Satisficing** — flag cases where the right action isn't the most visible action
8. **Phase 6 — Synthesis.** Template below; ask "Anything to revisit, or does this look complete?"
9. **Output.** Deliverable: follow `skills/product-shared/output-artifacts.md`; if an existing spec was found, update its "UX Decisions" section rather than create a new file; add review history entry "[date] — UX review (design completeness: X/10 → Y/10)". Explore: "Good session. The thinking we did on [key flows/decisions] should help shape the spec when you're ready."

**Hard rules (verbatim):**
```
<HARD-GATE>
Do NOT discuss implementation details, code architecture, technical stack, or file structures. This skill reviews user experience, not technical design. The brainstorming skill is where technical decisions happen.
</HARD-GATE>
```
"This offer MUST be its own message. Do not combine it with other content." (visual companion)

**Output format (synthesis, verbatim):**
```
> **Design completeness:** [X/10 → Y/10 after this review]
> **Key design decisions:** [bulleted list of what was decided]
> **Screens/flows reviewed:** [list with status — complete, needs work, deferred]
> **Remaining gaps:** [anything still unresolved]
```

**Acceptance check:**
- PM asks for a UX review of a nightly data-migration job -> after startup and prerequisite check, the agent exits via the UI Scope Gate message recommending `strategize`.
- PM asks for a UX review of a document-upload screen -> agent gives an X/10 rating with a concrete "what a 10 looks like", sends the visual-companion offer as a standalone message, maps journeys, rates each of the seven dimensions 0-10 with an AskUserQuestion per dimension, runs the principles scan, and synthesizes "X/10 → Y/10".

---

### Skill: refine

**Files to create:** `skills/refine/SKILL.md`.

**Frontmatter (verbatim):**
```yaml
---
name: refine
description: "Full product review pipeline. Runs discover, strategize, and ux-review in sequence to produce a fully reviewed product spec. Use when someone says 'full review', 'refine this feature', 'take me through the whole process', or wants the complete product planning workflow."
---
```

**Title / intro:** `# Refine — Full Product Pipeline`. "Run the complete product review pipeline: discover → strategize → ux-review. One conversation, fully reviewed spec out the other end. Each phase can be skipped if the work was already done in a prior session."

**Purpose / fires when:** PM wants the full review / complete product planning workflow.

**Procedure:**

0. **Setup** — Read `skills/product-shared/base.md` and follow its startup protocol (including the explore/deliverable intent check). Runs ONCE for the whole pipeline.
- **Early Exit (any phase transition):** if the PM says "that's enough", "let's stop here", "I have what I need", or similar: acknowledge "Good stopping point."; deliverable mode -> offer output artifacts for whatever phases completed; explore mode -> summarize what was covered and suggest which skill to pick up with later.
1. **Phase 0 — Intake.** Look in `docs/product-specs/` for a matching spec and check its "Review History":
   - No spec -> start at Phase 1 (discover)
   - Spec, no strategy review -> skip to Phase 2
   - Spec with strategy review, no UX review -> skip to Phase 3
   - Spec with all reviews -> "This feature has been through the full pipeline already. Want to re-run any phase, or is it ready for dev handoff?"

   Tell the PM what's skipped and why, e.g. "I found your product brief for [feature] from [date]. Discovery is already done, so I'll start with the strategy review."
2. **Phase 1 — Discovery.** Read `skills/discover/SKILL.md` and follow its Phase 1-4 (Problem Articulation, Scope Definition, System Context, Synthesis). Skip its Setup/startup protocol and its Output section (deferred). After the PM confirms the synthesis: "Good — we have a clear problem statement and scope. Now let's pressure-test it." Check for early exit.
3. **Phase 2 — Strategy Review.** Read `skills/strategize/SKILL.md` and follow Phase 1-5 (Mode Selection, Premise Challenge, Scope Assessment, Risk Identification, Synthesis). Skip Setup, Prerequisite check, Output. Mode selection still happens — the PM chooses expand/hold/reduce; never auto-decided. After confirmation: "Scope is solid. Let's look at the user experience." Check for early exit.
4. **Phase 3 — UX Review (conditional).** Decide whether the feature has user-facing screens, interactions, or visual components.
   - Yes: read `skills/ux-review/SKILL.md` and follow Phase 1-6 (Design Completeness, Visual Companion, User Journey Mapping, Seven Dimensions, UX Principles, Synthesis). Skip Setup, Prerequisite check, Output. The visual companion offer happens here if the PM hasn't seen it yet.
   - No: "This feature is backend-only — no UX review needed. Moving to final summary."
5. **Phase 4 — Final Summary.** Template below; ask "Anything to revisit, or does this look complete?"
6. **Phase 5 — Output Artifacts.** Deliverable: follow `skills/product-shared/output-artifacts.md` ONCE at the end — not after each phase; one consolidated spec, optionally a Confluence page and Jira tickets reflecting the full pipeline; add all applicable review history entries. Explore: "Good thinking session. When you're ready to build, a developer can pick up from here using the `brainstorming` skill."

**Hard rules (verbatim):**
```
<HARD-GATE>
Do NOT discuss implementation details, code architecture, technical stack, or file structures. This skill produces product thinking, not technical designs. The brainstorming skill is where technical decisions happen.
</HARD-GATE>
```
- "Run this ONCE at the end — not after each phase."
- "Mode selection still happens — the PM chooses expand/hold/reduce. This is a strategic judgment call that is never auto-decided."

**Output format (final summary, verbatim):**
```
> **Problem:** [from discover]
> **Target user:** [from discover]
> **Scope:** [from strategize — what's in, out, deferred]
> **Mode:** [expand/hold/reduce — from strategize]
> **Key risks:** [from strategize]
> **UX decisions:** [from ux-review, if applicable]
> **Design completeness:** [from ux-review, if applicable]
```

**Acceptance check:**
- Existing spec whose Review History shows Discovery and Strategy review -> refine announces it is skipping to UX review, runs only ux-review phases (no second startup protocol, no prerequisite prompt), then final summary, then a single artifact menu that updates the existing spec.
- PM says "let's stop here" after discovery synthesis in deliverable mode -> "Good stopping point." and an artifact offer for the discovery phase only; strategize is never started.

---

## Gaps and discrepancies observed in the source

- `base.md` hard-codes "11 TCN repos" and "TCN codebase graph" although Argus is otherwise project-agnostic; preserved as-is.
- `output-artifacts.md` calls `createConfluencePage`; the Atlassian MCP servers available in the author's sessions expose differently named Confluence create tools. Rebuild with the name as written; behavior depends on the installed connector.
- `ux-review`'s "Run discover first" option has no explicit stop/redirect instruction (strategize has one).
- `ux-review` Phase 3 says "use argus" without the availability guard that `base.md` step 4 has.
