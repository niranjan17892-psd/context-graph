# Skill: reflect

Small skill (SKILL.md is ~4 KB); most of its 30 files are eval assets. The SKILL.md rules are short enough to keep nearly verbatim.

## Files to create

| Path | Purpose / content spec |
|---|---|
| `skills/reflect/SKILL.md` | The whole skill (spec below). |
| `evals/README.md` | Table of assets and lifecycle: `trigger-queries.json`, `borderline-queries.json` (committed; trigger accuracy), `evals.json` (committed; output quality with/without skill), `files/` (committed; input fixture transcripts), `results/` (committed; curated summaries + `baseline/` regression evidence), `runs/` (not committed, gitignored). Commands: `npm run eval:triggers -- --skill reflect` (3 runs each), `... --dry`, `... --file borderline-queries.json`. Output-quality evals are manual; `without_skill` arm must run with `SKILL.md` removed (reflect auto-triggers in a subagent). |
| `evals/evals.json` | `{skill_name, notes, shared_assertions[4], evals[3]}` (below). |
| `evals/trigger-queries.json` | 24 queries, 16 train / 8 validation, with `_notes` (below). |
| `evals/borderline-queries.json` | 5 should-trigger queries forming the ~0.33–0.67 cluster; `_notes`: run at RUNS=10 (`node run-trigger-eval.mjs --file borderline-queries.json --runs 10`) to decide keep-vs-revert of a broadened description rather than on N=3 noise. |
| `evals/files/transcript-*.md` | Three fixture transcripts (below). |
| `evals/results/trigger-results.json` | RUNS=3 over trigger-queries: train 15/16, validation 6/8, overall 21/24. Failures: "anything we did by hand here…" (0.0), "at the end of this, I want a to-do list…" (0.0), "which of our skills needs fixing…" (0.33). Others at 1.0 except "where did our skills fall short today?" (0.67) and "were there gaps where a skill should have existed…" (0.67, pass). All negatives 0.0. |
| `evals/results/borderline-results-original.json` | RUNS=10, original description: 1.0 / 0.5 / 0.4 / 0.3 / 0.1 for the five borderline queries (in file order) — 1/5 pass. |
| `evals/results/borderline-results-broadened.json` | RUNS=10, broadened description: 1.0 / 0.4 / 0.2 / 0.1 / 0.1 — equal or worse on all five → broadened description **reverted**. (Both files share the same summary block; only per-query triggers/rates differ.) |
| `evals/results/baseline/iteration-1/benchmark.json` | with_skill 20/20 (1.0; 19.9 s; 28,396 tokens) vs without 9/20 (0.44; 55.7 s; 29,577 tokens); per-eval skill-correction 1.0/0.43, missing-skill 1.0/0.57, no-action 1.0/0.33; delta +0.56, −35.8 s, −1,181 tokens. Pattern notes: skill is better, ~3× faster, slightly cheaper; wins on terseness, two-section structure, excluding one-offs, not manufacturing findings; no-action is the sharpest discriminator (keep it); one-off assertion only carries signal in skill-correction; **iteration-2 candidate**: with-skill outputs were inconsistent on empty sections ("None." vs omitted vs one-line "no action items") — SKILL.md doesn't say whether to keep empty headers. |
| `evals/results/baseline/iteration-1/<eval>/<arm>/…` | For `eval-skill-correction`, `eval-missing-skill`, `eval-no-action` × `with_skill`/`without_skill`: `grading.json` (`{assertion_results:[{text,passed,evidence}], summary}`), `timing.json` (`{total_tokens, duration_ms}`), `outputs/skill-reflection.md` (with) or a long essay `.md` (without: `reflection-todo.md` / `skill-improvements.md`, containing Summary/Root cause/Priority/Honest-assessment prose — the negative baseline). With-skill reference outputs are reproduced under Output format. |

## Frontmatter (verbatim)

```yaml
---
name: reflect
description: "Use this skill when the user asks to reflect on the current chat, analyze what skills were used, or identify skill improvements. Triggers include: 'reflect', 'what skills could we build', 'what could be improved', 'skill review', 'analyze this chat', or any request to look back at the conversation and identify skill-related action items. Produces a markdown to-do list of skill updates and new skill ideas based on the conversation."
---
```

Body title `# Reflect`. Ends with a `## Keywords` line: `reflect, reflection, skill review, skill improvement, skill update, analyze chat, what skills, build skills, improve skills`.

## Purpose; fires when / does not

Analyze the current chat and produce a markdown to-do list of skill improvements and new skill suggestions.

When to use (SKILL.md list): user says "reflect", "reflect on this chat", or similar; asks what skills could be built or improved from the conversation; asks to review how skills performed; end of a long session where multiple skills were used.

Does not fire (trigger negatives): tool-usage/index review of an argus session (that is `argus-retrospective`), summarizing the chat, code review, product "what to build next", "reflect the latest API changes in the documentation" (keyword collision), argus index quality, team staffing "skill gaps", "what did we accomplish today", actually authoring a skill ("create a new skill for changelog generation" → `writing-skills`), feedback on the user's own code-review reply, analyzing a CSV.

## Procedure

1. **Analyze the chat** — read the entire conversation and identify:
   1. **Skills that were used** — which skill files were read/referenced; what tools/workflows they drove.
   2. **Corrections the user made** — after a skill produced output, what the user asked to fix, change, or redo. Each correction is a potential skill update.
   3. **Gaps** — requests where no skill existed; manual work that could have been automated or templated.
   4. **Patterns** — repeated similar instructions across turns (suggest a missing/incomplete skill).
2. **Categorize** into exactly two buckets: **Update existing skill** (skill used, output needed corrections — list skill + specific changes) and **New skill** (no skill existed — describe what it would do).
3. **Produce the to-do list** — output a single markdown file.

Rules for identifying updates — a skill needs an update when: user changed the output format after the skill ran (e.g. "don't use docx, use markdown"); user removed content the skill added (headers, footers, metadata); user corrected terminology or naming; user added details the skill should have asked for or included by default; user changed structure/organization of the output; the skill's output required more than two rounds of correction.

Rules for identifying new skills — suggest one when: user performed a multi-step workflow no existing skill covers; gave repeated manual instructions that could be encoded; asked for an output type matching no existing skill; a pattern of work emerged that would benefit from standardization.

What NOT to include: one-off corrections specific to the content, not the skill ("fixing a company name is content-specific, but 'skill always adds metadata headers' is a skill issue"); items where the user was exploring or changing their mind rather than correcting skill behavior; skills for trivial or one-time tasks.

## Hard rules
- "**Always markdown (.md).** No other formats."
- Output is a flat bullet list grouped by skill; each skill a bold top-level bullet; sub-bullets are specific updates/features. "Keep it short."
- Writing rules (verbatim): "Short sentences. No adjectives. No adverbs." · "State facts. 'User had to change X to Y' not 'The skill unfortunately produced an incorrect X.'" · "One item per bullet." · "Use action verbs: Add, Remove, Change, Fix, Create, Support." · "No explanations or justifications. Just the item."
- Never manufacture findings from exploration, mind-changing, or content-specific one-offs; when nothing qualifies, say so.

## Sub-agents / scripts
None. (Eval runner `run-trigger-eval.mjs` / `npm run eval:triggers` lives outside the skill.)

## Output format

Template (verbatim):
```markdown
# Skill Reflection — [Brief Chat Topic]

## Update Existing Skills

- **[skill-name]**
  - [Specific change needed]
  - [Another change needed]

- **[skill-name]**
  - [Specific change needed]

## New Skills to Build

- **[proposed-skill-name]** — [One-sentence description of what it does]
  - [Feature or behavior it should support]
  - [Another feature]

- **[proposed-skill-name]** — [One-sentence description]
  - [Feature]
```
Empty-section handling is unspecified in the source (observed: `- None.` under a header, omitting the empty header, or a single line "No skill-related action items." with a brief rationale). Keep it unspecified to match source, or pick one rule if deliberately fixing the noted iteration-2 gap.

Reference with-skill outputs (iteration-1):
- skill-correction → `# Skill Reflection — Drift Dashboard Discovery` / `## Update Existing Skills` / `- **discover**` with sub-bullets "Change default output format to markdown (.md). Remove .docx." · "Save specs to `docs/product-specs/`." · "Remove the "Document Metadata" header block (author / date / version table)." No company-name item; no New Skills section.
- missing-skill → Update section `- None.`; New: `**changelog-from-release-notes** — Transforms internal Confluence release notes into a customer-facing changelog.` with sub-bullets: group by feature area, not a flat list; strip TCNSR ticket numbers; remove internal component names (gateway, orchestrator, combiner); rewrite in plain customer language; accept raw release-note lines as input; run every release cycle (every two weeks).
- no-action → title + "No skill-related action items." + two-sentence reason (exploration resolved by user; one-off typo; no skill used/corrected; per exclusion rules).

## Evals content

`evals.json` notes: assertions added after iteration-1; reflect normally analyzes the current chat, so evals feed a saved transcript and tell the agent to treat it as the chat; baseline isolation (remove SKILL.md); each fixture engineered to exercise a specific rule. Each prompt: "Reflect on this chat. The conversation is saved at evals/files/<fixture> — treat that transcript as the chat to reflect on, and produce the skill-reflection to-do list."

Shared assertions (4): single markdown file; flat bullets grouped by skill under the two sections, OR explicit "nothing actionable"; terse — action-verb bullets, one item each, no narrative sections (no Summary / Root cause / Priority order / Honest assessment / What happened); no content-specific one-off included.

| id / label / fixture | Expected | Extra assertions |
|---|---|---|
| 1 `skill-correction` / `transcript-skill-correction.md` | `discover` update items for markdown-not-docx and dropping the Document Metadata header; "TCN Inc." → "TCN" excluded. | md-not-docx item; drop-metadata item; company-name fix absent. |
| 2 `missing-skill` / `transcript-missing-skill.md` | New skill: release-notes → customer changelog, capturing group by feature area, strip ticket numbers, plain customer language, remove internal component names; recognized as recurring; no spurious updates. | new-skill item present; captures the four rules; no spurious updates/unrelated expansion. |
| 3 `no-action` / `transcript-no-action.md` | Nothing actionable. | explicitly concludes none; no findings from abandoned naming or typo fix. |

Fixture transcripts (`# Conversation transcript — <topic>`, alternating `**User:**` / `**Assistant:**` lines, assistant actions in italics parentheses):
- **skill-correction** ("drift dashboard discovery"): user pitches a dashboard showing which TCN repos drift from indexed state; assistant invokes `discover`, asks target user/status quo/success, saves `drift-dashboard-brief.docx`; user: don't save as docx, all specs in `docs/product-specs/` are `.md`, redo as markdown; assistant re-saves `.md`; user: drop the "Document Metadata" header block (author/date/version table) — Confluence and git track that; assistant removes it; user: brief says "TCN Inc.", it's just "TCN"; assistant fixes; user: "Perfect, that works."
- **missing-skill** ("release notes to customer changelog"): user pastes ~15 internal 2.4 release-note lines tagged with TCNSR ticket numbers and component names (gateway/orchestrator); assistant drafts a flat list; user: group by feature area, drop TCNSR-#### numbers; assistant regroups; user: plain customer language ("Fixed an issue where the schedule wouldn't load" not "Resolved defect in SoA render path") and remove internal component names (gateway, orchestrator, combiner); assistant rewrites; user: "That's the format… we do this every release — every two weeks"; assistant asks anything else; user: "No, that's perfect. Ship it."
- **no-action** ("naming a drift tool"): user muses `drift_status` vs `audit_drift`; assistant leans `drift_status` (matches `index_status`); user considers folding into `index_status`; assistant agrees plausible; user prefers separate, proposes `drift_audit`, then asks if it already exists; assistant confirms `drift_audit` exists backed by `runDriftAudit`; user: never mind, use existing; user: fix typo "recieve" → "receive"; assistant fixes; "Cool, thanks."

Trigger queries (verbatim). Train should-trigger: "reflect"; "reflect on this chat"; "what skills could we build based on this session?"; "what could be improved about how the skills worked here?"; "do a skill review of this conversation"; "analyze this chat and tell me what skill updates we need"; "where did our skills fall short today?"; "anything we did by hand here that a skill should have handled?"; "give me a punch list of what to change about our skills after this". Train negatives: "review the MCP tool usage in this argus session and find the wasted calls"; "summarize this chat for me"; "review this code change before I merge"; "what should we build next for the product?"; "reflect the latest API changes in the documentation"; "what could be improved in the argus index quality?"; "what are the skill gaps on our engineering team?". Validation should-trigger: "look back at what we did — any skills worth creating?"; "at the end of this, I want a to-do list of skill improvements"; "which of our skills needs fixing based on this session?"; "were there gaps where a skill should have existed but didn't?". Validation negatives: "what did we accomplish today?"; "create a new skill for changelog generation"; "give me feedback on how I responded to that code review"; "analyze this CSV of survey responses". Borderline file: the 2 train + 3 validation indirect phrasings among these ("where did our skills fall short today?", "anything we did by hand…", "at the end of this…", "which of our skills needs fixing…", "were there gaps…").

`trigger-queries.json` `_notes` must record: key boundaries (argus-retrospective, writing-skills, discover/brainstorming, keyword collision); iteration-2 added 3 should-trigger train queries plus the staffing guard negative; **a broadened description was tried and REVERTED** (equal-or-worse at RUNS=10 on all 5 borderline queries; these phrasings trigger ~0.1–0.5 regardless of wording because the model answers directly — kept as documented known-hard cases); **KEY LESSON: RUNS=3 is unsafe near the 0.5 threshold** (reported 0.33–0.67 for queries truly at 0.10–0.20) — use RUNS≥10 for mid-range queries and compare original vs changed at the same N before keeping a description edit. Pass rule as in jira-ticket-generation: threshold 0.5.

## Acceptance check
1. **skill-correction transcript** → single `.md` with `## Update Existing Skills` → `**discover**` bullets "Change … markdown (.md)" and "Remove … Document Metadata header block"; no "TCN Inc." item; no prose sections.
2. **missing-skill transcript** → no update items (or "None."); one `## New Skills to Build` entry for a release-notes→changelog skill listing the four surfaced rules as action-verb bullets; no extra enhancements (memory, denylist, recurrence nudges).
3. **no-action transcript** → explicitly "No skill-related action items." (at most a brief reason); nothing invented from the naming exploration or typo fix.
