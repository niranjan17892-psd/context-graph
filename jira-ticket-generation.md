# Skill: jira-ticket-generation

**Org-specific skill.** Products (Contracts Module, Study Intelligence), Jira projects `TCNSR`/`TCN`/`TCNPD`, the Atlassian cloud `leanlawlabs`, its cloud ID, the MCP server UUID in tool names, the `cust.staging.leanlawlabs.dev` host, and Drive doc names are all org-specific values (marked **[org]** below). The `ask_user_input_v0` tool and `computer:///` links are claude.ai-host-specific (**[host]**).

## Files to create

| Path | Purpose / content spec |
|---|---|
| `skills/jira-ticket-generation/SKILL.md` | The whole skill: intro (3-part job), Core rules, Steps 0–5, Tone. Contains two body templates (verbatim below). |
| `evals/README.md` | Table of eval assets and lifecycle: `trigger-queries.json` (committed; trigger accuracy), `evals.json` (committed; output quality with/without skill), `results/` (committed; curated summaries + `baseline/` regression evidence), `runs/` (**not** committed, gitignored scratch). Running: `npm run eval:triggers -- --skill jira-ticket-generation` (3 runs each) and `... --dry` (validate + print plan). Output-quality evals run manually; `without_skill` arm must run with `SKILL.md` removed since the skill auto-triggers in a subagent. |
| `evals/trigger-queries.json` | `{skill_name, _notes, queries:[{set, should_trigger, query}]}` — 20 queries, 12 train / 8 validation (list below). Notes: negatives lean on adjacent repo skills (backlog-*, discover, strategize, argus); run each ~3x; pass = `(should_trigger && rate>0.5) || (!should_trigger && rate<0.5)`; revise description only from train failures; judge generalization on validation. |
| `evals/evals.json` | `{skill_name, notes, shared_assertions[6], evals[3]}` (content below). |
| `evals/results/trigger-results.json` | `{summary:{threshold 0.5, runs_per_query 3, train 12/12, validation 8/8, overall 20/20}, results:[{set, should_trigger, query, runs, triggers, inconclusive, trigger_rate, pass}]}`. All rates 1 or 0 except "after you look at the logs, write up a proper engineering ticket…" at 0.67 (2/3). |
| `evals/results/baseline/iteration-1/benchmark.json` | Iteration-1 with/without comparison: per_eval contracts 1.0 vs 0.43, study-intel 1.0 vs 0.5, edge-speculation 1.0 vs 0.56; with_skill 24/24 (mean 25.7 s, 36,243 tokens), without 12/24 (0.49; 30.0 s; 27,126 tokens); delta +0.51, −4.3 s, +9,117 tokens. Pattern notes: skill wins on title-out-of-file, explicit `Error message:`, exact template fields, lean body; "no invented diagnosis" and "no Helix leak" pass in both (low signal — replace with harder cases); the boilerplate assertion is too narrow (missed `Type:`/`Priority:` and a `Suggested Investigation` section) — widen to "no native-Jira-field metadata lines and no troubleshooting/investigation sections"; ~9k tokens for +0.51 is worth it. Without-skill arm ran with SKILL.md physically removed (first attempt discarded — auto-triggered). |
| `evals/results/baseline/iteration-1/<eval>/<arm>/{grading.json, timing.json, outputs/*.md}` | For each of `eval-contracts-complete`, `eval-study-intel-complete`, `eval-edge-speculation` × `with_skill`/`without_skill`: `grading.json` = `{assertion_results:[{text, passed, evidence}], summary:{passed, failed, total, pass_rate}}`; `timing.json` = `{ "total_tokens": N, "duration_ms": N }`; one output ticket `.md`. With-skill outputs: `bug-ticket-word-addin-required-positions-not-applied.md`, `bug-ticket-soa-timing-intervals.md`, `bug-ticket-match-timeout-large-3pp.md` (each follows the template exactly; the edge case flags missing URL/Agreement ID/recreation inline in italics, platform "Web app (contracts editor) — please confirm", and carries `Reporter noted: "…" (Reporter's own hypothesis — not a confirmed diagnosis.)`). Without-skill outputs (H1 titles, Summary/Environment/Impact/Acceptance Criteria/Suggested Investigation sections) are the negative baseline. |

## Frontmatter (verbatim)

```yaml
---
name: jira-ticket-generation
description: >
  Triage and structure bug tickets before they reach engineering. Use this skill whenever an analyst
  says "file a ticket," "submit a bug," "log this issue," "triage this ticket," "write up this bug,"
  "ticket triage," or any variation of reporting a bug, filing an engineering ticket, or structuring
  a bug report for developers. Also trigger when someone pastes in a rough bug description and wants
  it cleaned up for engineering, or mentions recreating a bug across environments. The skill covers
  two products: the **Contracts Module** (contract negotiation — agreements, profiles, redlining,
  Word add-in, match, AI markup) and **Study Intelligence** (clinical trial study startup — protocol
  summary, schedule of activities extraction, budget vs. protocol analysis, multi-document chat,
  coverage analysis). The skill never guesses at root causes or fixes; it produces a clean,
  copy-pasteable Jira description and a separate title delivered in chat.
---
```

Body title `# Jira Ticket Generation`.

## Purpose; fires when / does not

Helps an analyst prepare a developer-ready bug ticket. Three-part job: (1) pull recent read-only Jira + Confluence context for the feature (hard recency 60–90 days) so questions reflect current behavior; (2) gather missing info via `ask_user_input_v0` [host] for choices, short chat prose for quick free text, one elicitation form only when much is missing; (3) deliver a Jira title **in chat** and a clean `.md` with only the ticket body. Rationale: devs waste time rediscovering what the analyst knows.

Fires on filing/submitting/logging/writing-up/triaging a bug, cleaning a rough bug description for engineering, recreating a bug across environments. Does not fire (trigger-eval negatives): backlog triage/stale tickets, enriching tickets with acceptance criteria, prioritization, feature ideas, scope review, finding duplicates across a project, creating epics, summarizing open bugs, commenting on an issue, tracing code flows.

## Core rules (non-negotiable)

- **"Never guess at the root cause or the fix."** No "Root Cause", "Likely Cause", "Probable Fix", "Suggested Fix" sections; never "this is likely caused by…" / "the fix is probably…". Only when the analyst stated it, verbatim, attributed: `Reporter noted: [their exact words]`. Incomplete input → better recreation questions and screenshot requests, never speculation.
- **Preserve the analyst's original wording.** Organize, don't rewrite; a summary sentence may be fresh, but Steps to Reproduce and Actual Outcome should read like the analyst wrote them.
- **The Jira title goes in chat text, not in the .md file.** Format `` `<Feature>: <quick description>` ``. Examples: `SoA Extraction: timing intervals dropped for multi-cycle protocols`; `Map & Gap: budget line items missing after protocol re-upload`; `Word Add-in (Desktop): profile positions not applied on 3pp agreements`; `Study Workflow: tabs not visible after creating a study with SoA selected`. Name the specific feature, plain-language breakage, scannable; never vague ("Bug in editor").
- **Keep the body lean.** No "Bug origin", "Date reported", "Reporter", "Status", "Severity" lines (Jira captures natively). Body = URL → Steps to Reproduce → Actual Outcome → Expected Outcome + a small bug-specific context block (Agreement ID for Contracts; Study ID + workflow for Study Intelligence). Nothing else.
- **Recreation default: production → staging, not dev.** Ask about dev only if the analyst raises it.
- **Never use the word "Helix" in user-facing text** [org]. Product is **Study Intelligence**. Never in titles, bodies, form questions or chat; reading it as input (Drive guide, internal docs, URL paths) is fine.

## Procedure

**Step 0 — Detect the product.**
- Contracts Module indicators: agreements/agreement IDs, profiles, Required/Preferred/Permitted/Prohibited positions; Match, AI Markup, profile match, redlining, tracked changes; Word add-in (desktop or M365 web), Word plugin; 3pp/1pp negotiation, templates, block replacement; `/agreementeditorview` or `/profile/...` URLs.
- Study Intelligence indicators: studies/study IDs/workflow/creation wizard; protocol summary, SoA extraction; budget analysis, protocol vs budget, Map & Gap; coverage analysis (NCD/LCD), ICF congruency; multi-doc chat, single-doc chat scoped to a study; activity database, `master_activities.json`, agents/orchestrator/gateway; `/studies/{uuid}` URLs (e.g. `cust.staging.leanlawlabs.dev/helix/studies/...` [org] — path says helix, product is Study Intelligence).
- Genuinely ambiguous → ask once in chat: "Is this a Contracts Module bug or a Study Intelligence bug?"

**Step 1 — Review what the analyst provided.** Read everything. **Look at every screenshot** and extract: screen/view name, verbatim error text, form values/selections/identifiers, visible page state (modals, banners, toasts, selection, empties), platform chrome (Word ribbon vs browser). Don't re-ask what's visible. Number screenshots `[Image 1]`, `[Image 2]`… and remember which step each illustrates. Extract from URL: subdomain → environment; path → module/feature; UUID → Agreement/Study ID. Inventory what's missing: steps; actual vs expected; URL; environment; recreated?; Contracts: Agreement ID, Word add-in desktop/web; SI: Study ID, workflow/feature, documents in study.

**Step 2 — Pull recent context first (read-only), before asking anything.** Purpose: smarter questions (recent refactor, possible duplicate) and preventing "working as designed" tickets. **Hard recency gate: only last 60–90 days; never loosen; "stale context is worse than no context."**
- Jira via `mcp__1fdc1354-feab-491d-a550-f258aef173dd__searchJiraIssuesUsingJql` [org: server UUID], cloud `5e248fe8-303d-42ff-9b31-091d08b8fdb8` (leanlawlabs) [org]:
  1. Stories/PRDs: `project in (TCNSR, TCN, TCNPD) AND issuetype in (Story, Epic, Idea, Feature) AND text ~ "<feature keywords>" AND updated >= -90d ORDER BY updated DESC` — top 5; discard anything not updated in 90 days.
  2. Related bugs: `project = TCNSR AND issuetype = Bug AND text ~ "<feature keywords>" AND updated >= -60d ORDER BY updated DESC` — top 3; discard >60 days.
- Confluence via `mcp__1fdc1354-feab-491d-a550-f258aef173dd__searchConfluenceUsingCql`: `text ~ "<feature keywords>" AND lastModified >= now("-90d")` — top 3; discard >90 days.
- Use: informs Step 3 questions (behaviors to test, likely duplicates); held for Step 5 delivery. If no recent matches in a category, say so explicitly (e.g. "No recent stories or PRDs for SoA Extraction (nothing updated in the last 90 days)."). If the analyst obviously wants to skip, respect it — but always offer once.
- Never: create/edit/comment/transition Jira issues; edit/comment Confluence; paste content from >60–90-day items anywhere; write "this might be related to TCNSR-XXXXX" inside the ticket (related items are chat-only).

**Step 3 — Ask clarifying questions.**
- Threshold:
  - **0–1 missing items** → don't ask; write the ticket and flag the gap inline (e.g. "URL not captured — please paste before submitting").
  - **2–3 missing, choice-style** → one `ask_user_input_v0` call (≤3 questions, 2–4 options each).
  - **A few free-text items** (URL, steps) → one short bundled chat message; free-text ask may sit in the preamble of a choice question.
  - **4+ missing, or long free-form writing not started** → one elicitation form via `mcp__visualize__show_widget`, after calling `mcp__visualize__read_me` with `modules: ["elicitation"]`.
  - A two-question chat never becomes a form; eight things incl. textareas never becomes eight chat questions.
- Don't ask what URL/screenshots/description already answer. **Context-informed questions are the high-value ones** — ask explicitly about a likely duplicate ("Could this be the same bug as TCN-12345 …?"), about a recent PRD behavior change, about which version after a refactor.
- Example calls: duplicate confirmation — preamble naming the recent bug + "updated 4 days ago", Q "Does this look like the same bug?", options `Same bug — close mine / Same area, different symptom / Different bug / Not sure — file separately`. Environment+platform — Q1 "Which environment did you see this in?" → `Staging / Production / Both / Other`; Q2 "Which platform?" → `Web app / Word Add-in (Desktop) / Word Add-in (M365 web)`. Recreation — "Did you recreate this in staging?" → `Recreated in staging / Recreated in prod only / Not yet recreated / N/A — found in staging`.
- More than 4 plausible options → top 3 from context + "Other".
- Question menu (pick only what's missing and needed). High value: 1 URL (free text; most informative); 2 Steps (free text or form); 3 What happened vs expected; 4 Screenshot of a specific ambiguous step; 5 Possible-duplicate confirmation (whenever Step 2 found a credible candidate). Lower: 6 Environment (skip if URL shows it); 7 Recreation status (skip for low stakes). Contracts (only if missing and not inferable): 8 Agreement ID; 9 Platform. SI: 10 Study ID; 11 Feature/workflow (top 3 + Other); 12 Study state (only when it materially affects diagnosis); 13 Documents in study (only when type matters). **Out:** Session ID, recreation URL as a separate field, profile-used-for-recreation, template-used-for-recreation — include only if volunteered.
- Phrasing: open with one sentence acknowledging what you have; conversational ("Which environment did you see this in?" not "Environment:"); bundle, don't drip-feed; no "To file the best ticket, I'll need…" preamble.
- Follow-up screenshot: ask once in chat, naming the specific screen.

**Step 4 — Build the body** (Markdown; title never in the file). Templates below.

Formatting rules: image placeholders go in the section where the screenshot belongs (error → Actual Outcome; setup state → Steps); omit non-applicable fields rather than "N/A", **except** always write the Error message line ("No error message displayed." when none); "Profile used" (Contracts) only when the analyst used a profile *they personally created* — never customer profiles; no boilerplate about customer-data handling, deletion reminders, severity, or policy; no Root Cause / Likely Cause / Suggested Fix / Troubleshooting Notes section.

**Step 5 — Deliver.**
1. Save body as `bug-ticket-<short-descriptor>.md` in the outputs folder (descriptor from feature + breakage, e.g. `bug-ticket-soa-timing-intervals.md`, `bug-ticket-word-addin-profile-not-applied.md`).
2. In chat, the title on its own labelled line plus the file link:
   > **Jira title:** `SoA Extraction: timing intervals dropped for multi-cycle protocols`
   >
   > **Ticket body:** [computer:///.../outputs/bug-ticket-soa-timing-intervals.md]
3. Include the Recent context block (if anything worth surfacing).
4. No boilerplate about what you did/skipped or next steps.

**Tone:** direct, efficient; fewest questions, lightest medium. Never write "Let me know if you'd like me to…".

## Output format

Contracts Module template (verbatim):
```markdown
URL: <full URL from the analyst>

**Agreement ID:** <id>  *(or "Not available (customer agreement)")*
**Platform:** <Web app / Word Add-in (Desktop) / Word Add-in (Microsoft 365 web)>

## Steps to Reproduce

1. <Step one — in the analyst's wording>
2. <Step two>
3. ...

[Image 1 here]

**Recreated in staging:** <Yes — URL: ... / No / N/A — found in staging>

*(If recreated, include the recreation specifics inline:)*
- **Agreement template / document used:** <name>
- **Profile used:** <profile name> *(only when the analyst created a profile to reproduce the bug; never customer profiles)*

## Actual Outcome

**Screen/view:** <exactly which screen/view>

<Detailed description of what happened — what changed on screen, what looked wrong>

**Error message:** <exact text, where it appeared. If none, say "No error message displayed.">

[Image 2 here]

## Expected Outcome

<What should happen when this is fixed.>
```

Study Intelligence template (verbatim):
```markdown
URL: <full URL from the analyst>

**Study ID:** <uuid>
**Feature / workflow:** <e.g., SoA Extraction, Map & Gap, Multi-doc chat>
**Session ID:** <if available>
**Study state at time of bug:** <e.g., "Protocol uploaded, SoA extraction complete, Map & Gap in progress">
**Documents in the study:** <filenames and/or document types>

## Steps to Reproduce

1. <Step one — in the analyst's wording>
2. <Step two>
3. ...

[Image 1 here]

**Recreated in staging:** <Yes — URL: ... / No / N/A — found in staging>

## Actual Outcome

**Screen/view:** <exactly which screen/view>

<Detailed description — for Study Intelligence, pay extra attention to: which activities/timing intervals/budget items were wrong, whether the issue is in agent output vs UI display, whether the same data renders inconsistently across features>

**Error message:** <exact text, where it appeared. If none, say "No error message displayed.">

[Image 2 here]

## Expected Outcome

<What should happen when this is fixed.>
```

Chat-side Recent context block (verbatim):
```
**Recent context** (verify before relying on — Study Intelligence specs change fast)

*Stories / PRDs:*
- TCNSR-XXXXX — "Title" — updated 12 days ago
- TCNPD-XXX — "Title" — updated 5 days ago

*Recent related bugs:*
- TCNSR-XXXXX — "Title" — last touched 3 weeks ago, status: Open

*Confluence:*
- "Page title" — last modified 8 days ago — link

⚠️ Anything older than 60–90 days isn't shown. Even items shown above may not reflect the current architecture. Verify against the most recent docs in Drive (`Helix Prompt Work`, `Helix V2 Map & Gap Agent`, etc.) before trusting them.
```
(The Drive doc names contain "Helix" — they are internal doc titles [org]; this is the one place the source shows the word in chat output.)

## Sub-agents / scripts
None. Tools used: Atlassian MCP JQL/CQL search (read-only), `ask_user_input_v0` [host], `mcp__visualize__read_me` + `mcp__visualize__show_widget` (elicitation form). Eval runner `npm run eval:triggers` lives outside the skill.

## Evals content (evals.json)

`notes`: assertions added after iteration-1; prompts supply complete info to exercise the "0–1 missing → just write" path; BASELINE ISOLATION WARNING — the without-skill arm must run with the Skill tool denied and `skills/` unreadable, then be verified free of skill-only strings (e.g. "verify before relying on").

Shared assertions (6): single `.md` with only the body, no title line at top; no Root Cause/Likely Cause/Probable Fix/Suggested Fix section; no Severity/Reporter/Date reported/Status/Bug origin lines; explicit `Error message:` line (states "No error message displayed" when none); title in chat not in the file; Steps preserve analyst wording.

| id / label | Prompt essence | Extra assertions |
|---|---|---|
| 1 `contracts-complete` | "File a bug." Word Add-in (Desktop) on staging, AI Markup on a 3pp agreement doesn't apply Required positions; URL `https://cust.staging.leanlawlabs.dev/agreementeditorview?agreementId=AGR-48217`; steps 1) open AGR-48217 in the add-in 2) Run AI Markup 3) wait; actual: no Required positions as tracked changes; expected: every Required position applied; recreated in staging. | Contracts template: `Agreement ID:` and `Platform:` present. |
| 2 `study-intel-complete` | "Submit a bug for Study Intelligence." SoA Extraction drops timing intervals for multi-cycle protocols; URL `https://cust.staging.leanlawlabs.dev/helix/studies/9f3c2a10-77b2-4e51-bc44-d1a2f9e0c123`; documents `Protocol_v3.pdf`; steps create study → run SoA extraction → open SoA view; actual: repeating activities show blank interval; expected: populate per cycle like single-cycle; recreated in staging. | SI template: `Study ID:`, `Feature / workflow:`, `Documents in the study:` present; "Helix" absent except inside the URL. |
| 3 `edge-speculation-and-missing-url` | "Quick bug": Match times out on big agreements in the contracts editor; analyst's guess: match service isn't caching the profile between runs; steps open large 3pp → hit Match → wait ~30s → spins forever; expected: Match returns results; no URL handy. | Hypothesis omitted or attributed verbatim as `Reporter noted: ...`, never a diagnosis; complete ticket still produced; missing URL flagged inline. |

Trigger queries (verbatim). Train should-trigger: "file a ticket for this — in the Word add-in (desktop) on staging, AI Markup finishes but none of the Required positions get applied to the doc"; "can you write this up as a bug for engineering? SoA extraction is dropping the timing intervals for multi-cycle protocols"; "log this real quick: match keeps spinning forever on big 3pp agreements in the contracts editor"; "here's a rough bug description, clean it up so I can paste it into Jira — profile match returns nothing after I re-upload the protocol"; "the agreement editor keeps dropping my redlines and I need to get this in front of the devs"; "i recreated this in staging and want to submit it — budget vs protocol numbers don't line up in Map & Gap". Train negatives: "triage the backlog and find the stale tickets"; "enrich these tickets with acceptance criteria and make them dev-ready"; "prioritize the backlog — what should we work on next?"; "I have a feature idea for the Word add-in, help me flesh it out"; "review the scope of this plan, is it too big?"; "find duplicate tickets in our Jira project". Validation should-trigger: "triage this bug pls — coverage analysis throws an error on NCD lookup for one of my studies"; "I need to structure this bug report so a developer can pick it up without pinging me back"; "after you look at the logs, write up a proper engineering ticket for the multi-doc chat failure"; "something's broken with the schedule of activities table — how do I report it to the team?". Validation negatives: "create a Jira epic for the Q3 roadmap"; "summarize the open bugs in TCNSR for me"; "comment on TCN-1234 that the fix is verified"; "trace how the AI markup flow works across the repos".

## Acceptance check
1. **Eval 1 prompt.** No questions asked (≤1 missing); file `bug-ticket-word-addin-…md` starts with `URL:` line, has `**Agreement ID:** AGR-48217`, `**Platform:** Word Add-in (Desktop)`, analyst's three steps, `**Recreated in staging:** Yes …`, `**Error message:** No error message displayed.`; no H1, no Summary/Impact/Severity; chat shows `**Jira title:**` + link.
2. **Eval 3 prompt.** Ticket still written; `URL:` line is an inline "Not captured … please paste before submitting" flag; the caching guess appears only as `Reporter noted: "…"`; no investigation section.
3. **Vague input** ("the SoA is broken", screenshot only). Product detected as Study Intelligence; Jira/Confluence searched with the recency filters before asking; 4+ missing → one elicitation form (after `read_me` elicitation); a recent similar bug surfaces as an explicit duplicate question; "Helix" never written in chat or ticket.
