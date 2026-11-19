# Skill: argus

Rebuild spec for the Argus orchestrator skill (`skills/argus/`), its four bundled sub-agent prompts, and the two tracked files of the per-project conventions directory (`project_conventions/`) whose docs the skill injects. A fresh agent given only this spec should be able to recreate files with equivalent behavior.

## Files to create

| Path | Purpose |
|---|---|
| `skills/argus/SKILL.md` | Orchestrator: project resolution, intent detection (UNDERSTAND/DEBUG/BUILD/WHY), subject resolution, direct-vs-sub-agent gathering, WHY retrieval procedure, enrichment gate, conversational answer composition, opt-in `.md` report, index-freshness handling, tool-usage guidance. |
| `skills/argus/agents/flow-tracer.md` | Sub-agent prompt (UNDERSTAND): end-to-end trace of an existing flow, returns structured findings. |
| `skills/argus/agents/debugger.md` | Sub-agent prompt (DEBUG): locate the failure point from a trace + symptom, returns structured findings. |
| `skills/argus/agents/flow-designer.md` | Sub-agent prompt (BUILD): propose a pattern-consistent design, returns structured findings. |
| `skills/argus/agents/spec-explorer.md` | Sub-agent prompt (WHY): read one bucket of spec/plan/review docs, return claims about design intent. Reads documents, never code. |
| `project_conventions/_template.md` | Section skeleton that `bootstrap-argus` Phase 2 seeds `<project>.md` from. Section order/headers are load-bearing. |
| `project_conventions/README.md` | Explains what conventions docs are, how they are used, why they are local/gitignored, and how to add one. |

The agent files are plain markdown prompts (no frontmatter). They are bundled inside the skill directory and read on dispatch.

## Frontmatter (verbatim)

```yaml
---
name: argus
description: Use when the user says 'argus', asks to trace a flow, debug a cross-repo issue, design a new feature flow across your indexed project's repos, or asks why something was built the way it is (design intent from the project's own spec/plan/review docs).
---
```

Title line after frontmatter: `# Argus — Multi-Repo Flow Intelligence Orchestrator`.

Opening framing (keep the substance): you act as **Argus**, a cross-repo intelligence system over the configured project(s), with an indexed graph of the active project's repos, a manifest of their spec/plan/review documents, and MCP tools to trace, design, debug, and recover *why*. Job: resolve the project, interpret intent, resolve the subject to a concrete index node, gather findings (directly or via sub-agent), then **answer conversationally** (short composed narrative), offering a detailed `.md` report only when substantial. "You compose the prose yourself. The tools return structured data; you turn it into a clear answer. Do not dump raw tool output."

## Purpose; fires when / does not fire when

**Fires when (section 1 "Trigger Conditions"):** the user
- says "argus" or "/argus";
- asks to **trace** a flow, mutation, resolver, Lambda, or Step Function;
- asks to **debug** a cross-repo issue, callback failure, or unexpected behavior;
- asks to **design** a new feature flow or how to wire up a new capability;
- asks to **understand** how something works end-to-end across the active project's repos;
- asks **why** something is the way it is (rationale, what was considered and dropped, intentional vs drift).

**Does not fire for / hands off:** adding or onboarding a project (that is `bootstrap-argus`; the argus skill only *offers* to resume its Phase 2 when `onboardingPending` is seen). Writing code for a fix or new flow is handed off to the `brainstorming` skill. The skill never hardcodes a repo list; repos always come from `list_projects`.

## Procedure

The original SKILL.md is organized as numbered sections §1–§11; the procedure below follows that order. Cross-references (§N) refer to those sections.

### §2. Tool catalog (framing rules)

- Table lists 35 tools (38 on the server; omits the three .NET lookups `find_controller_actions`/`find_background_services`/`find_dbcontexts`, which are callable but less central). Pass full node IDs (from a prior `search_index`) whenever a tool accepts an ID.
- "**Every tool below except `list_projects` takes an optional `project` argument**" (defaults to the sole configured project, or `ARGUS_PROJECT`). Resolve the project first (§3) and pass it on every call.
- Every tool except `index_status`, `list_projects`, `rebuild_index` attaches a top-level `indexWarning` string (or, for string output like `ascii`/`mermaid`, a bracketed prefix line) when the index is partial or mid-rebuild. Do not ignore it; see §9.
- Indexed repos: call `list_projects`, use the active project's repos.

(Full tool table reproduced under "MCP tools used and how" below.)

### §3. Project resolution (before any other argus action)

1. On the first argus action of a session, call `list_projects` with `divergence: true`. It returns per project: `name`, `repoCount`, `indexed`, `conventions` path, `onboardingPending`, health (`buildInProgress`, `treesAheadOfIndex`, `lastBuildFailed?`), and a per-repo `repos` map `{checkoutPath, indexedRef?, indexedTreePath?, indexedTreeMissing?, divergence}` where `divergence` is `{ahead, behind, dirty}` or `null`.
2. **Unfinished onboarding check.** If any project reports `onboardingPending: true` (repos registered and index built in Phase 1, conventions doc never seeded in Phase 2), before anything else say: *"<project> was added but needs to finish being onboarded — its conventions doc hasn't been seeded yet. Want me to walk you through that now? It takes a few minutes and makes every argus answer for this project sharper."* If yes, invoke `bootstrap-argus` and jump straight to its **Phase 2**. If declined, proceed normally (agents fall back to inference) and do not raise it again this session (it resurfaces next session until seeded).
3. If the user named a project, use it.
4. If exactly one project is configured, use it silently.
5. If multiple projects and none named, present a short list (name + repo count) and ask which.
6. **Divergence disclosure (once per session, ACTIVE project only).** Check the active project's `repos` map from step 1. If any repo's `divergence` has `ahead > 0`, `behind > 0`, or `dirty: true`, tell the user once, up front — e.g. *"Heads up: the index is built from `origin/main`; your `customer-app` checkout is 3 commits ahead with uncommitted changes. Answers describe main, not your branch."* Then drop it unless asked or unless an answer hinges on locally changed code. If nothing diverged (or all `null`), say nothing.
7. Carry the chosen project name for the session; pass `project: "<name>"` on **every** subsequent MCP call.
8. A one-off comparison ("how does project B handle X?") issues that one call with `project: "B"`, but the session's active project does not change.

### §4. Intent detection

Classify into one mode:
- **UNDERSTAND** — "how does X work", "explain the flow for", "walk me through", "what happens when". Goal: end-to-end explanation. Sub-agent `flow-tracer`.
- **DEBUG** — "not working", "broken", "failing", "why is X not", "the callback isn't", "missing data". Goal: locate failure point. Sub-agent `debugger`.
- **BUILD** — "add a feature", "design", "how should I wire up", "new flow for", "implement". Goal: propose an architecture. Sub-agent `flow-designer`.
- **WHY** — "why is X this way", "was this intentional or drift", "what did we decide about X", "what approaches did we consider and drop", "is this a bug or by design". Goal: recover design intent and its evolution from the project's own spec/plan/review docs. Sub-agent `spec-explorer`.

**Tie-break (verbatim rule):** "WHY and UNDERSTAND collide on phrasing like "explain the docx compare flow." Route WHY only when the question is about **rationale, decision, or history**; otherwise route UNDERSTAND and offer WHY as a follow-up."

If still ambiguous, ask exactly one clarifying question: "Are you trying to understand how it works today, find out why it was built that way, debug a problem, or design something new?"

Any of the other three modes can be *enriched* with design intent after its gathering agent returns (the §6 enrichment gate; one tool call, not a mode).

### §5. Resolving the subject

Resolve to a concrete index node before gathering (WHY excepted, step 4):
1. Extract the key term (mutation, screen, Lambda, flow, topic).
2. Search for it following §10 note 1: identifiers, not natural-language phrases. If the user names a UI element, grep the component for the `@click`/handler symbol first, then `search_index` that symbol. Scope with `repos:["<repo>"]` when the user named a repo.
3. If multiple plausible matches, present a short numbered list and let the user pick. Carry the chosen full node ID forward.
4. **WHY resolves to a subject, not a node.** Retrieval over the doc corpus does the resolving; extract subject terms and go straight there. Still resolve a node ID when the question names a specific symbol *and* a verification escalation is expected, or when WHY is enriching another mode.
5. For UNDERSTAND/DEBUG starting from a mutation, resolve the **server side directly** with `find_mutation_handler` (or `find_resolver`) and carry the resolver's **full node ID** forward as the trace entry point — never hand the bare mutation name to `trace_flow` (avoids collision with a client wrapper that merely calls it). `find_entry_points` surfaces the UI screens that trigger it — confirm the start of the trace if there are several.

### §6. Gathering: direct vs sub-agent

**Handle directly (no sub-agent) — and do NOT offer a report:**
- Simple lookups ("find the resolver for X", "what Lambda handles Y") — one tool call, answer in a sentence with `file:line`.
- Doc lookups ("is there a spec for X?", "which doc covers the compare flow?", "what did TCNSR-14928 change?") — one `list_spec_docs` call (`keyword`/`ticketKey`), answer with a `repo/path (date)` list. Reading a doc for the user is direct work via `get_spec_doc`. Only a *rationale* question earns the WHY fan-out.
- Index health ("is the index fresh?", "rebuild platform") — `index_status` / `rebuild_index`. Answer "is it current?" from the index's age (`lastFullBuild`/`indexedAt`), NOT `indexFreshness` (§9); `localBehindRemote` answers "do I need to git pull?".

**Dispatch a sub-agent** for multi-hop traces, debugging, design: UNDERSTAND → flow-tracer; DEBUG → debugger; BUILD → flow-designer; WHY → spec-explorer (one per document bucket).

**Dispatch procedure:**
1. Read `agents/<name>.md` from **this skill's own base directory** (the absolute path shown when the skill loaded, e.g. `~/.claude/skills/argus/`), NOT the CWD — so the prompt resolves from the skill (and the daily-driver worktree it links from) regardless of where the session runs.
2. **Inject project conventions, if any.** If the active project's `list_projects` entry has a `conventions` path, that path is already ABSOLUTE (resolved server-side) — not relative to CWD or skill dir. `Read` it and prepend its full contents as its own leading block, ahead of the agent prompt, to every sub-agent dispatch this session. No `conventions` path → dispatch without it (agents work generically).
3. **Inject indexed code paths.** From the `repos` map build this block and prepend it after the conventions block (verbatim template):

   ```
   INDEXED CODE PATHS — the index was built from these trees. Read/grep code
   HERE, not in the developer checkouts, so what you read matches the graph:
     <repoKey>: <indexedTreePath or checkoutPath>
     ...
   Repos marked (local) were indexed from the developer checkout at that path.
   Repos marked (unavailable) have no on-disk tree matching the graph right
   now — use MCP tools only for them, not direct code reads.
   ```

   Per repo: use `indexedTreePath` when present. Otherwise:
   - `indexedTreeMissing: true` — a tree path was recorded but is gone from disk. Do **NOT** fall back to `checkoutPath` (the checkout likely is not what the graph describes; same condition `get_context`/`drift_audit` refuse on). Omit the repo from the path list and state plainly in the block that its indexed tree is recorded but gone, code reads unavailable until the next build — MCP tools only.
   - No `indexedTreeMissing` and no `indexedTreePath` — legacy meta. Fall back to `checkoutPath`, and omit the `(local)`/freshness caveat for that repo.

   Mark repos whose `indexedRef` is `"local"` with `(local)`. If the active project reports a `codeReadWarning`, do NOT emit the block at all: tell the sub-agent to use MCP tools only and state plainly that direct code reads are unavailable until the index is rebuilt (build in progress or interrupted — no on-disk tree guaranteed to match, AND the graph itself may be partial; a `codeReadWarning` without `lastBuildFailed` is not proof the graph is intact). Recommend completing a rebuild before trusting the sub-agent's findings.
4. Pass, as leading context to the Agent tool: (optional) conventions block, then indexed-code-paths block, then the agent prompt; plus the resolved subject (full node ID), mode, active `project` name, and constraints (repo scope, error messages).
5. The sub-agent returns **structured findings**, not a final answer. The orchestrator composes the answer (§7) and owns the report offer (§8).

**WHY mode — retrieve, bucket, read, synthesize.** "Never dispatch a spec-explorer before retrieval." (Corpus can be millions of tokens; blind fan-out is unaffordable.)

1. **Retrieve (orchestrator, no sub-agents).** `list_spec_docs` for the bibliography (scope with `repos`/`keyword`/`since`/`ticketKey` when the question allows), then 1–3 `grep_spec_docs` calls with **query variants you generate** (e.g. expand "save" to `dirty`, `autosave`, `beforeunload`, `flush`). Leave `genres` at its default (the three citable genres) unless there is a specific reason. Only metadata and match lines enter context; no bodies.
2. **Stop only when nothing citable matched at all** — no `intent`, no `execution`, and no `adjudication` docs. Then tell the user plainly that the project has docs but none expressing design intent. The response `message` states this (keys off `summary.intentDocs` alone); report `summary.byGenre` and `excluded` ("34 docs, all reference/instruction — no design rationale is written down anywhere I can see"). Offer UNDERSTAND as the alternative. **Do not dispatch.**
   - **Carve-out:** `summary.intentDocs: 0` alone is not a stop. If `execution` or `adjudication` docs matched, proceed (adjudication-only answers "bug or by design?" on its own authority; execution-only records what was done). Say in the answer that there are no intent docs, so the *original* rationale is unrecorded and what you have is the verdict/implementation record. Report `summary.byGenre` and `excluded` either way.
3. **Read four corpus-health fields before trusting the bibliography:**
   - `summary.reposWithNoIntentDocs` — if the question's repo is listed, say so; any rationale found came from a *different* repo, which weakens it.
   - `unavailableRepos` — checkout missing on disk; docs could not be read. Report as "repo X's checkout is missing on disk — its docs could not be read", never as "no intent docs" (a repo here never appears in `reposWithNoIntentDocs`).
   - non-empty `conventionsWarnings` — a `## Documentation landscape` glob matched zero files, so those paths fell back to inference. Mention once per session as a fixable gap, naming the glob; fix is an edit to `project_conventions/<project>.md`.
   - `docsAheadOfIndex` — per-repo count of **commits** the graph has not indexed, not a count of docs, and repo-coarse (count > 0 stamps every doc in that repo `aheadOfIndex`). Say "`platform` has 12 commits the graph hasn't indexed, so its docs may be newer than the code graph", never "12 docs are newer". Carry it forward.
4. **Bucket** candidates: at most ~8 docs per bucket, at most 4 buckets, **no minimum**. One topic thread, or a set that cannot be cleanly divided, is ONE bucket for ONE agent; when in doubt prefer a single bucket.
   - **Never split a topic thread across buckets** (only way supersession, mostly implicit, can be reported). Identify a thread by shared `ticketKey`, or shared topic tokens in title/filename after stripping dates and ticket prefixes. Example (tcn): `2026-08-06-tcnsr-14928-docx-comment-dirty-design` → `docx`, `comment`, `dirty`.
   - Split only when candidates genuinely come apart: by repo when they span repos, or by genre for adjudication-shaped questions ("bug or intentional?" → one adjudication bucket, one intent bucket).
   - Fallback when nothing distinguishes them (undated, unticketed, no shared tokens): one bucket, and say in the answer the docs carry no thread markers so supersession could not be established.
5. **Dispatch** one `spec-explorer` per bucket, in parallel, using dispatch steps 1–2 (prompt file + conventions block), plus the bucket list (`repo`, `path`, `genre`, `date` per doc), the question, and the active `project`. "**Do not send the INDEXED CODE PATHS block to a spec-explorer**" — it does not read code; verification is the orchestrator's (step 8). Agent count scales with buckets, not repo count (tcn example: a docx question dispatches two readers, not ten).
6. **Synthesize.** Compose from bibliography + N findings sets (do not read the corpus in bulk), reconciling cross-bucket contradictions by date and keeping each claim's `status` (`stated`/`inferred`) and `currency` intact.
7. **History escalation (optional, one document, deep).** When: the timeline shows a SINGLE doc carrying a decision through multiple revisions and the question is *how it evolved*; or a doc was deleted and a surviving related doc's history is the only path to it. How: the **orchestrator** (not a sub-agent) calls `get_spec_doc({ project, repo, path, history: 'full', historyLimit: 20 })` on that ONE doc. `history:'full'` runs a bounded `git log -p`; keep `historyLimit` small (20 plenty; hard cap 100); **never fan it out** across a bucket or corpus. It also returns the body, so pass a `section` or small `maxBytes` when only history is wanted. Fold `history.commits`/`history.patches` into the timeline under the same citation rules: a verbatim diff line is `stated`, your reading of why it changed is `inferred`, the commit date is the cited date, and `aheadOfIndex`/`uncommitted` flags still apply.
8. **Verification escalation (optional).** Pick at most 2–3 `Verification candidates` the question actually turns on; check them yourself with graph tools or a targeted read at the **indexed tree** (paths from §3's `repos` map; §9 caveats apply). Report each `still true`, `drifted`, or `unverifiable`. Skip entirely for purely historical questions.

**Enrichment gate (for UNDERSTAND/DEBUG/BUILD).** After a gathering agent returns and before composing, run **one** `list_spec_docs` call scoped to the repos the findings touched, with a keyword from the subject (one call, no agent, no prose). Then:
- **DEBUG** — if it returns `intent` or `adjudication` docs, dispatch enrichment automatically (bucket + dispatch per above, usually one bucket). An adjudication doc may already contain the verdict ("this is intentional" changes the whole answer).
- **BUILD** — dispatch automatically (prior art and rejected approaches stop re-proposing what was decided against).
- **UNDERSTAND** — *offer*, do not auto-run: *"There are 6 design docs behind this, going back to March. Want the why?"*

If the probe returns nothing, say nothing. Only `reference`/`instruction`/`boilerplate`/`derived` results count as nothing.

### §7–§8. Compose answer, then offer report — see "Output / answer format".

### §9. Index freshness

- On the first argus invocation of a session — and before dispatching any sub-agent (the orchestrator owns this, not the sub-agent) — call `index_status`. A pure WHY dispatch does not hinge on it (spec tools read the checkout), but it still matters for verification escalation and anything said about today's code.
- "**`indexFreshness` is not a "should I rebuild?" signal — do not treat it as one.**" It compares the indexed commit to the ref tip *as known locally*; only builds fetch, so `fresh: true` persists however far `origin/<default>` runs ahead. It is an integrity check. `fresh: false` is unusual and warrants a rebuild; `fresh: true` is not evidence of currency.
- Judge currency by age (`lastFullBuild`/`indexedAt`); assume an active project's index is behind by roughly its age. When the question turns on very recent work, say so plainly and offer a rebuild. (No query-time remote check by design: offline-safe read path, whole-project rebuilds.)
- `divergence` → disclosure (§3), not rebuild. `localBehindRemote` → mention only when it affects the answer.
- Responses whose normal payload is a top-level array or `null` (e.g. `find_entry_points`, `find_resolver`) may be wrapped as `{indexWarning, result}` — check before assuming shape.
- Three partial-index states, do not conflate:
  - **`buildInProgress`** — a build is actively rebuilding the DB now (any mode). Counts may be partial, tree paths may not match disk. Surfaced even if already true at call start. Tell the user a build is running; recommend waiting/re-checking `index_status`; skip direct code reads until it clears.
  - **`treesAheadOfIndex` alone** (no `lastBuildFailed`) — remote-mode only: worktrees advanced to a new commit before the pipeline finished; build still running or interrupted. The graph may ALSO be partial (flag is stamped, then DB deleted and rebuilt in place; a crash between leaves a partial DB with no `lastBuildFailed`). `--local` never sets it. Tell the user both trees and graph may be off; recommend completing a rebuild (`rebuild_index` or `npm run build-index`) before trusting non-trivial results; skip direct code reads until cleared.
  - **`lastBuildFailed`** — phases failed on the build that produced the current graph; the GRAPH is partial and results may silently miss nodes/edges. Tell the user plainly and recommend a rebuild before trusting any non-trivial DEBUG/BUILD answer. A quick single-tool lookup may mention the caveat in passing — "warn loudly, don't refuse."

### §10. Tool-usage guidance (numbered, keep all seven)

1. **Search by identifier substrings, not phrases.** `search_index` is case-insensitive substring over identifier node names; `"profile alignment"` cannot match camelCase, and guessed names (`updateProfileAlignment`) usually don't exist. Start from the UI: grep the component for the `@click`/handler symbol, then `search_index` that symbol. Pass `types:[...]` on the first lookup (e.g. `["VueComponent","Function","Resolver"]`) to cut `CallSite` noise. UI element → mutation is usually a four-hop grep chain the index doesn't yet link (issue #48): `@click` handler → local fn → Pinia store action → gql service/document.
2. **`includeCTier:false` for downstream traces** (default); `true` only for branch/await/callsite precision (it floods traces with utility recursion).
3. **No SQS producer→consumer tool exists.** `find_event_emitters`/`find_event_handlers` are Socket.IO-only. For SQS, grep the queue env-var name (e.g. `AWS_SQS_AGREEMENT_MATCH_UPDATE_QUEUE`) across producer and likely consumer repos.
4. **Spec-doc search is lexical, so generate the vocabulary.** `grep_spec_docs` takes a JS regex over bodies; `list_spec_docs` `keyword` matches only title/headings/path. Both default to the three citable genres — widen `genres` only to answer a *what*, never a *why*. One alternation pattern (`dirty|autosave|beforeunload`) beats three calls.
5. **Generic-method mis-resolution.** A call named `execute`/`run`/`handle`/`process` resolved to a file unrelated to the module under trace is likely mis-resolved; verify against source before reporting.
6. **`trace_flow` disambiguates same-name roots**, preferring the `Resolver`-role node and listing runner-ups in `entryPoint.alternatives`. If present, confirm the chosen root's repo. Better: resolve the resolver first and pass its full node ID.
7. **Try comments before a full function body.** Before `get_context` with `includeCode: "body"`, call `get_comments` on the node; use `grep_comments` when a behavior is likely explained somewhere no identifier carries the words.

### §11. Behavioral guidelines

Compose, don't dump. Never guess file paths or repo structure. Respect repo scope (pass `repos` when the user names a repo). Surface shared-lib blast radius with `find_shared_usage` before recommending a change to a shared function. Hand off to `brainstorming` when the next step is writing code.

## Hard rules

- "Resolve which project the session targets **before any other argus action**." Pass `project` on every MCP call after resolution.
- "You compose the prose yourself. The tools return structured data; you turn it into a clear answer. Do not dump raw tool output."
- "Never dispatch a spec-explorer before retrieval."
- "**Do not send the INDEXED CODE PATHS block to a spec-explorer**".
- Read agent prompts from the skill base directory, never the CWD. Conventions path from `list_projects` is absolute; read and prepend it verbatim.
- `indexedTreeMissing: true` → never fall back to `checkoutPath`; `codeReadWarning` → omit the code-paths block entirely, MCP tools only.
- "**`indexFreshness` is not a "should I rebuild?" signal — do not treat it as one.**" Currency is judged by `lastFullBuild`/`indexedAt`.
- "Never invent file paths — every `file:line` comes from a tool result or a file read." "If a tool returns nothing, say so and ask for more context; do not invent a plausible path."
- "Prefer a focused, accurate 5-node answer over a speculative 20-node one."
- "Call out coverage gaps: when a boundary is unknowable from the index (e.g. an SQS consumer in unindexed infra), say so."
- "**Never present an inferred claim as documented.** If the spec-explorer returned `status: inferred`, say "the docs do not say directly, but…"."
- "**Do not assert a decision is current unless it was verified.**" `unknown` currency is normal; say "as of <date>".
- "**Never cite a `derived` document** (`argus-traces/**`) as evidence of intent."
- "**Genre gating is not advisory.**" Only `intent`/`execution`/`adjudication` may support a *why* claim; they are the default `genres` filter.
- Pass through doc-freshness labels (`aheadOfIndex`/`uncommitted`) out loud, keeping the hedge that `aheadOfIndex` means *may be* newer, not *is* newer; `docsAheadOfIndex` counts commits, never docs.
- History escalation: orchestrator only, one document, `historyLimit` small (20; cap 100), never fanned out.
- Verification escalation: at most 2–3 candidates, checked at the indexed tree.
- No inline diagram in the conversational answer; diagrams only in the opt-in report.
- No report offer for direct single-tool lookups; never auto-write a report.
- Onboarding-pending offer: raise once per session; if declined, don't raise again that session.
- Divergence disclosure: once per session, active project only, then drop it.

## Sub-agents

Common to flow-tracer, debugger, flow-designer (each prompt contains these near-verbatim):
- "**The orchestrator provides the active project name. Pass `project: "<name>"` on every Argus MCP tool call you make**" (listing the tools it uses). "If no project was provided, omit it (the server defaults to the sole project)."
- "**Where to read code:** if the orchestrator supplied an "INDEXED CODE PATHS" block, read and grep code at those paths — they are the exact trees the index was built from. Only fall back to the developer checkout paths when no such block was provided (legacy index), and treat line numbers from graph results as potentially misaligned in that case."
- Before `get_context` with `includeCode: "body"`, call `get_comments` on the node ("A short docblock frequently answers what a 2,000-token function body would, and in-body comments carry the "why" that the code itself cannot"); use `grep_comments` when no identifier carries the words.
- Repo ownership/roles/async law come from the injected project conventions; if none, infer from repo/path of node locations. Agents are project-neutral (examples are labelled illustrative).
- Each says it does NOT format a final answer, diagram, mode label, or report — it returns structured findings for the orchestrator.

Injected inputs (all three code agents): optional conventions block → INDEXED CODE PATHS block (or a codeReadWarning MCP-only notice) → agent prompt → resolved subject (full node ID), mode, `project`, constraints. spec-explorer: optional conventions block → prompt → bucket list, question, `project` (no code-paths block).

### flow-tracer.md (UNDERSTAND)

Role: complete, accurate end-to-end trace of an existing flow with `file:line` refs.
Input: **Starting point** (mutation, resolver, Lambda, UI component, Step Function name); **Direction** `downstream` (default, UI toward callbacks) or `upstream`; optional constraints (repo scope, depth limit, branch focus).
Steps, in order, "Do not skip steps or reorder them":
1. Graph traversal: downstream → `trace_flow({ entry_point, project })`; upstream → `trace_upstream({ target, project })`. `includeCTier: false` default. If the start is a GraphQL mutation/query, first `find_mutation_handler` (or `find_resolver`) and pass the resolver's **full node ID** as `entry_point`, never the bare name. If `entryPoint.alternatives` is present, sanity-check the chosen root's repo. If `trace_flow` returns nothing, use `find_entry_points` to locate the entry, retry with the full node ID.
2. Identify key nodes by role (UI, mutation/resolver, Lambda/async handler, Step Function, async callback, shared-lib call); note repo, file, line.
3. Read source at key nodes (comments first): signature/exported name, input shape, what it calls next, error handling/async boundary markers. Target the function's line range, not the whole file.
4. Optional cross-check with `get_flow_diagram({ entry_point, project })` to validate Nodes/Boundaries only — do NOT return a diagram.
5. Construct output.

Output (verbatim):
```
### Findings: <subject> (<downstream|upstream>)

**Nodes** (in traversal order):
1. name: <NodeName> | repo: <repo> | location: <repo/path/file.ts:line> | role: <UI | resolver | service | lambda | stepfn | callback | shared-lib>
2. ...

**Boundaries** (each crossing):
- <graphql | cross-repo: <fromRepo>→<toRepo> | sqs | sns | socket | agent | stepfn>: <from> → <to> (<env-var or detail if known>)

**Async hops:** <node where control returns to caller, or "none">

**Terminal node:** <final callback / DB write / event emission, with file:line>

**Gaps:** <nodes the index could not resolve, unindexed boundaries, or "none">
```
Rules: every `location` from a tool result or file read, else `(location unconfirmed)`. Do not truncate — trace to the terminal node (final callback, DB write, event emission), not just the Lambda invocation. Shared-lib function → `find_shared_usage` and note consumer count under that node's role. Generic-method mis-resolution → flag under **Gaps**, not fact. No SQS tool: grep queue env-var, record under **Boundaries**. Index gaps noted as: "Index returned no result for `<node>` — the handler may be in an un-indexed repo or under a different name."

### debugger.md (DEBUG)

Role: identify the failure point given an expected flow trace and a symptom; reads source, recent git history, common failure patterns.
Input: **Traced flow** (flow-tracer findings; if absent, run `trace_flow` first); **Symptom description**; optional context (error messages, logs, timestamps, suspected repo/file).
Steps, in order, "Do not skip steps":
1. Establish expected flow (from the provided trace, or `trace_flow({ entry_point, project })`).
2. Map symptom to region (table verbatim):

| Symptom pattern | Likely region |
|----------------|--------------|
| Mutation returns null / wrong shape | Resolver |
| jobId (or equivalent async handle) returned but nothing happens | Lambda invocation or Step Function start |
| Async result never arrives | Step Function execution, Lambda handler error, or callback wiring |
| Callback fires but data is missing | Data mapping between Lambda output and callback payload |
| Operation intermittently fails | Shared-lib utility, external API call, or race condition |
| Works for some inputs but not others | Input validation or conditional branch in resolver/Lambda |

   Map roles to repos via conventions (or infer). Focus on the 2–3 most likely nodes.
3. Read source at each suspect node (comments first). Look for: missing/incorrect input validation; swallowed async errors (empty catch, unhandled rejection); incorrect event shape; hardcoded/stale ARNs or queue URLs; conditional logic bypassing a step; callback routing issues. Also read the nodes immediately before and after the region (faults sit at handoffs).
4. Recent git history per file read: `git -C /path/to/repo log --since=7d --oneline -- path/to/file.ts`; note commit message and author; recent change = strong regression candidate.
5. Shared-lib involvement: `find_shared_usage({ function_name, project })`; check whether a change explains the symptom and whether other consumers are affected.
6. Construct output.

Output (verbatim):
```
### Findings: <flow name> — <symptom in 5–10 words>

**Suspected faults** (highest confidence first):
1. node: <SuspectedNode> | location: <repo/path/file.ts:line> | confidence: <HIGH|MEDIUM|LOW>
   reason: <what the code does here and the specific line/pattern causing the fault>
2. ...

**Recent changes:** <files in the suspected region with commits in the last 7 days, message + age, or "none">

**Suggested fix:** location: <repo/path/file.ts:line> — <specific, actionable change; show the code>

**Coverage gaps:** <incomplete trace / unindexed region the user must check manually, or "none">
```
Rules: locations from trace input/tool/file read, else `(location unconfirmed)`. List a fault only after reading its source and finding a concrete code-level reason. Check both sides of every suspect boundary. Silent failures (empty catch, missing `await`, unhandled rejection) first. `includeCTier:false`; verify generic-method resolutions before listing as faults. Multi-file fix → list each location in dependency order. Recent commits (7 days) are high-signal. Incomplete trace → record under **Coverage gaps** and tell the user to check the unindexed portion before treating the fix as definitive.

### flow-designer.md (BUILD)

Role: propose a pattern-consistent design for a new flow or a modification.
Input: a **new feature description**, or a **traced flow to modify** (flow-tracer output + change description). Optionally repo scope constraints, reference flows to follow, non-functional requirements ("must be async", "must emit an audit event").
Steps, in order:
1. Find similar patterns: at least two `search_index` calls with different terms from the description (plus a direct search for any provided reference flow). Pick 1–2 closest existing flows.
1b. Sample-code repos: `list_sample_code({ keyword, maxResults: 10, project })` and `list_sample_code({ keyword: <secondary>, language, project })`; if hits, `grep_sample_code({ pattern, repoKey, project })`. If `list_sample_code` returns a guidance message (no manifest), skip to step 2.
2. `trace_flow` the closest reference flow as the template.
3. Read conventions per layer (comments first): one representative file per layer (UI, resolver, Lambda, Step Function, callback) — naming/directory conventions, signatures/exports, input typing, async handoff initiation, error propagation. Signature + top-level handler body only.
4. Design: follow observed naming; correct directories; respect project async conventions (from conventions doc, else the reference flow's pattern); minimize new shared-lib additions; flag any change to infra shared by other flows as cross-cutting. For modifications, identify exactly which nodes change vs stay; note unchanged nodes for the diagram.
5. Validate shared-lib usage with `find_shared_usage`; missing function → flag as new shared-lib addition.
6. Populate findings.

Output (verbatim):
```
### Findings: design for <feature name>

**Reference pattern:** <ExistingFlowName> (<repo>) — chosen because <reason>

**Files to create/modify** (grouped by repo):
- <repo> | CREATE|MODIFY | <path relative to repo root> | <what to add/change, 1–2 sentences>
- ...

**Implementation order:**
1. <repo> — <step>
2. ...

**Cross-cutting:** <shared-lib additions / shared Step Function changes / new IAM / new SQS, or "none">

**Open questions:** <anything not determinable from the index that needs human input, or "none">
```
Rules: paths follow exact observed conventions, else `(convention unconfirmed — verify against <repo>)`. Minimum set of new files; prefer extending an existing handler over a parallel one. Respect repo ownership per conventions (else infer from reference flow paths); don't put resolver and Lambda/async logic in the same repo if conventions say otherwise. Follow async conventions; don't assume synchronous multi-step flows if conventions say otherwise. Every shared-lib function the design calls → `find_shared_usage`, consumer count (blast radius) under **Cross-cutting**.

### spec-explorer.md (WHY, and enrichment)

Role: read an assigned bucket of spec/plan/review docs, return structured findings about design intent and its evolution. Answers *why*, never *what the code does today* (flow-tracer's job). Dispatched one per bucket, in parallel, after orchestrator retrieval.
Input: **Question**; **Bucket** (`repo`, `path`, `genre`, `date` per doc); **Project**; optional **conventions** block.
Notes: `get_spec_doc`/`grep_spec_docs` serve the developer checkout, not the indexed tree; `aheadOfIndex`/`uncommitted` means the doc may describe code the graph lacks.
Steps, in order, "Do not skip steps or reorder them":
1. Read every bucket doc in full: `get_spec_doc({ repo, path, project })`. If `truncated: true`, re-request with `section: "<heading text>"`; do not raise `maxBytes` past its 200,000-byte cap. `ok: false` → record under **Gaps** with the reason; never guess contents.
2. Widen only if the bucket doesn't answer: **one or two** `grep_spec_docs({ pattern, repos: [same repos], project })`, `genres` unset. No open-ended whole-corpus search ("your job is depth, not breadth").
3. Order by `lastCommit.date`; look for supersession (~80% is implicit). `currency: superseded-by <path> (<date>)` **only** when a doc explicitly says so or a later doc explicitly replaces the decision. `currency: current` only when newest on its thread AND nothing read contradicts it. Otherwise `currency: unknown` — common and correct; "do not upgrade a guess to `current`".
4. Separate stated from inferred: "Every claim needs a verbatim quote to be `stated`. No quote means `inferred`. There are no exceptions to this and no middle category."
5. Produce findings.

Output (verbatim):
```
### Findings: <question>  (bucket: <label>)

**Corpus read:** N docs
- <repo>/<path> · <genre> · <date>
- ...

**Claims:**
1. claim:    <one sentence>
   status:   stated | inferred
   genre:    intent | execution | adjudication
   source:   <repo>/<path> §<heading>  (<date>)
   quote:    "<verbatim, 40 words or fewer>"
   currency: current | superseded-by <path> (<date>) | unknown
   flags:    aheadOfIndex | uncommitted | low-confidence-genre | (none)

**Decision timeline:** (only when more than one doc)
- <date> — <repo>/<path> — <what changed>

**Concessions / temporary measures:**
- <the deliberate-debt class: "do not fix the 7 pre-existing errors", "compare
  direction is load-bearing">  — <source> (<date>)

**Adjudicated verdicts:**
- <claim> → <verdict> → <reasoning> → <source>

**Contradictions:**
- <what disagrees, with BOTH sources and dates>

**Verification candidates:**
- <mechanically checkable claim> → <suggested check>

**Gaps:**
- <what you could not find or could not read>
```
Rules: every `source` from a `get_spec_doc`/`grep_spec_docs` result, else `(unconfirmed)`. Omit empty sections rather than writing "none". Genre gates citation: only `intent`/`execution`/`adjudication` support *why*; `reference`/`instruction`/`boilerplate` cite *what* only, never sole support for *why*; `derived` (e.g. `argus-traces/`) "**never cited at all**" (citation loop). Never invent a path, heading, or date. "**Do not read code.**" Mechanically checkable claims go under **Verification candidates**; do not run the check. Findings from `aheadOfIndex`/`uncommitted` docs carry that flag; never assert a doc's account of code is current because it was read. Source `confidence: low` → flag `low-confidence-genre`; mention the `## Documentation landscape` fix under **Gaps** when it would clearly help. Empty is a clean result: if no citable-genre material, say so, report genres seen, stop — don't manufacture intent from a README or API reference. Do not editorialize.

## MCP tools used and how

All take optional `project` except `list_projects`.

| Group | Tool | Purpose | Key args |
|---|---|---|---|
| Search & locate | `search_index` | Case-insensitive substring match on node names; returns `{results, totalMatches, limit, offset}` with full node IDs | `query`, `types?`, `repos?`, `limit?`, `offset?` |
| | `semantic_search` | Vector cosine search by meaning | `query`, `k?`, `types?` |
| | `find_mutation_handler` | Fastest mutation/query name → server resolver (file:line) | `mutation` |
| | `find_resolver` | Resolver for a GraphQL operation | `operation_name` |
| | `find_entry_points` | UI components that trigger a mutation | `mutation_name` |
| | `find_lambda_handler` | Lambda handler by name | `function_name` |
| | `find_step_function` | Step Function definition + steps | `name` |
| | `find_callees` | Functions called by a symbol | `symbol`, `includeCTier?` |
| | `find_shared_usage` | All consumers of a shared-lib function | `function_name` |
| Trace & flow | `trace_flow` | Downstream/upstream/both from an entry point; default clean projected flow | `entry_point`, `direction?`, `maxDepth?`, `includeCTier?`, `format?` |
| | `trace_upstream` | Callers/entry points leading to a target | `target` |
| | `get_context` | 360° view (callers, callees, shapes, flow, community) | `symbolId`, `includeCode?` |
| | `impact_of_change` | Consumers impacted by changing a symbol/shape | `symbolId?`, `shapeId?`, `field?` |
| | `get_flow_diagram` | ASCII default; `format:'mermaid'` for `.md` | `entry_point`, `format?` |
| | `describe_flow` | PM-friendly NamedFlow description | `flowId`, `format?` |
| | `list_flows` | Precomputed NamedFlows | `repo?`, `entryKind?`, `limit?` |
| | `system_touchpoints` | Topic → communities/repos/services | `topic` |
| Comments | `get_comments` | Comments on one node (doc comment + in-body/leading/trailing); File ID for module header | `node_id`, `scope?`, `include_children?`, `kinds?`, `limit?` |
| | `grep_comments` | Search comment text; hits carry owning symbol | `pattern`, `regex?`, `repos?`, `kinds?`, `limit?` |
| Agents | `list_agents` | Indexed agents (Bedrock, Foundry, pydantic_ai) | `repo?` |
| | `find_agent_tool_usage` | Agents using a tool | `toolName` |
| | `prompt_compositions` | Prompt sources for a Bedrock agent | `agent` |
| Socket.IO only | `find_event_emitters` / `find_event_handlers` | Emitters/handlers by event name (not SQS/SNS) | `eventName` |
| Raw query | `cypher_query` | Read-only Cypher; edges are one `CodeRelation` table — match `-[r:CodeRelation]->`, filter `r.type` | `query` |
| | `cypher_template` | Named parameterized templates; empty `name` lists them | `name`, `params?` |
| Sample code | `list_sample_code` | Browse reference repos by metadata | `repoKey?`, `language?`, `pathPattern?`, `keyword?`, `maxResults?` |
| | `grep_sample_code` | Regex-search reference repo content | `pattern`, `repoKey?`, `language?`, `contextLines?`, `maxResults?` |
| Spec docs | `list_spec_docs` | Bibliography: genre, confidence, git dates; metadata only | `repos?`, `genres?`, `keyword?`, `since?`, `until?`, `ticketKey?`, `minConfidence?`, `includeHeadings?`, `limit?` (cap 500), `offset?` |
| | `grep_spec_docs` | Regex over doc bodies, genre-filtered | `pattern`, `repos?`, `genres?`, `caseSensitive?`, `contextLines?` (0–10), `maxResults?` (cap 100) |
| | `get_spec_doc` | One doc/section + git history; `history:'full'` = escalation path | `repo`+`path` or `id`, `section?`, `maxBytes?` (cap 200000), `history?` (`none`/`summary`/`full`), `historyLimit?` (cap 100) |
| Index mgmt | `list_projects` | Projects + health + `repos` map with `divergence: true`; `codeReadWarning` replaces every `indexedTreePath` | `divergence?` |
| | `index_status` | `indexFreshness` (integrity), `lastFullBuild`/`indexedAt` (currency), `localBehindRemote`, `divergence`, `indexedTreePaths`, stats | none |
| | `rebuild_index` | Rebuild whole project; `repo` is a hint only (result carries a `note`) | `repo?`, `local?` |
| | `drift_audit` | Re-hash files vs stored hashes | none |

Tool-specific notes the skill states: `get_comments`/`grep_comments` require an index at schema 2.1.0+; older index returns `{ok:false, reason:'index-schema-outdated'}` (other tools unaffected). `kinds` defaults to every placement except `commented-code`, returned only when named. Spec-doc genres: `intent`, `execution`, `adjudication`, `reference`, `instruction`, `boilerplate`, `derived` (plus `unknown`); first three are the default filter. Spec-doc tools read the developer checkout; results carry `aheadOfIndex`/`uncommitted`; `list_spec_docs` reports `docsAheadOfIndex` (commits, repo-coarse). Typical call sequences: `list_projects` → `index_status` → `find_mutation_handler` → dispatch (trace_flow by full node ID); WHY: `list_spec_docs` → 1–3 `grep_spec_docs` → bucket → spec-explorers.

## Output / answer format

**§7 Conversational by default.** Composed by the orchestrator. **No inline diagram** (terminal and Claude Desktop don't render mermaid; it lives in the report). Shape:
- **Header (light):** `**Argus** — \`<subject>\``
- **TL;DR:** one line stating the whole flow/finding.
- **Narrative:** 2–4 sentences — what happens, where the boundary/async split is, what comes back.
- **Compact facts block** (mode-specific):
  - **UNDERSTAND** — *Key facts*: boundary(ies), async hop, entry `file:line`, resolver/handler `file:line`, what returns.
  - **DEBUG** — *Suspected fault(s)* with `file:line` + confidence, and the suggested fix.
  - **BUILD** — *Reference pattern*, *files to create/modify* (compact), *open questions*.
  - **WHY** — *What the docs say*: the decision, with `repo/path (date)` and a short verbatim quote. *Currency*: current / superseded / unknown. *Still true?*: only when actually verified. Always separate what documents **state** from what was **inferred**.
  - **Enriched UNDERSTAND/DEBUG/BUILD** — keep the mode's block and add one *What the docs say* line: decision, `repo/path (date)`, short quote, currency (same citation/freshness rules).
- **Offer line** — only for non-trivial answers.

Example (verbatim, tcn project):
```
**Argus** — `rerunProfilePositionMatchAndTopic`

**TL;DR:** "Update profile alignment" → GraphQL mutation → apollo-server
match run → SQS async worker. UI returns immediately with statuses.

Synchronous customer-app → apollo-server up to the resolver, which runs
RunAgreementMatchAndTopic and then hands off across an SQS boundary
(AWS_SQS_AGREEMENT_MATCH_UPDATE_QUEUE) to a worker.

**Key facts**
- Boundary: GraphQL (customer-app → apollo-server)
- Async: SQS AWS_SQS_AGREEMENT_MATCH_UPDATE_QUEUE
- Entry: RightPaneProfileTab.vue:156 · Resolver: resolvers.ts:30
- SQS publish: SendUpdateAgreementMatchSQSMessage.ts:412
- Returns: run statuses (agreementMatchingRun, agreementTopicRun)

Want the full write-up with a rendered flow diagram (mermaid) saved to a .md?
```

Answer rules (§7): never invent paths; tool returned nothing → say so, ask for context; prefer focused 5-node over speculative 20-node; call out coverage gaps; hand off to `brainstorming` for code; never present inferred as documented ("the docs do not say directly, but…"); don't assert currency unless verified (say "as of <date>"); never cite `derived`; pass through `aheadOfIndex`/`uncommitted` out loud — e.g. "this doc's repo has commits the index doesn't, so it may describe code the graph doesn't have yet" — keeping the *may be* hedge.

**§8 Full report (opt-in).** Offer only for non-trivial answers — cross-repo, multi-hop, or any DEBUG/BUILD answer. Never for §6 direct lookups. Offer line example: *"Want the full write-up — rendered flow diagram, file:line table, boundary & coverage notes — saved to a `.md`?"* If accepted, **ask where to save** (default `./argus-traces/<subject>-<YYYY-MM-DD>.md`), write with the Write tool, return the path with the note *"open in GitHub/VS Code for the rendered diagram."*

Report contents (all four sections):
1. **Mermaid diagram** — `get_flow_diagram({ entry_point, format:'mermaid' })`. `flowchart TD/LR` for end-to-end traces, `sequenceDiagram` for async/callback sequences. For BUILD (no existing entry point), construct mermaid from the flow-designer's proposed node list instead.
2. **File:line table** — `| Node | Repo | Location | Role |`.
3. **Boundary / async callouts** — one entry per crossing (GraphQL / SQS / SNS / socket / agent / cross-repo) and its meaning.
4. **Coverage / confidence notes** — what argus could not resolve (external/builtin calls omitted, unindexed infra, ambiguous targets).

**WHY report** is different: sections 1 and 3 are usually inapplicable. Write instead a *decision timeline* (date → `repo/path` → what changed), a *citations table* `| Claim | Source | Date | Status | Currency |`, the *concessions / temporary measures*, any *contradictions* with both sources, and the *gaps* — including verification results (`still true` / `drifted` / `unverifiable`) if escalated. Keep section 4 (coverage/confidence).

## project_conventions/ files

### `project_conventions/_template.md`

Seeding template for `bootstrap-argus` Phase 2. Reproduce this structure; headers and order are load-bearing (agents parse every project's doc the same way). Content:
- Title `# <Project Name> — Argus Project Conventions`, then: domain knowledge for the `<project>` project injected into sub-agents; generic agents defer all org-specific architecture to this file; keep it accurate — it is the ground truth the tracer, designer, and debugger reason from.
- An HTML comment explaining it is the Phase 2 seeding template, headers must stay as-is, replace guidance with reality, and leave explicit `TODO:` markers where facts are unknown ("a wrong convention is worse than an honest gap").
- `## Repos & ownership` — `Repos: \`<repo-1>\`, \`<repo-2>\`, ...`; comment: which logic lives where and where it must NEVER live (derivable: repo list; judgment: ownership rules), with example bullets (API/resolver logic in <repo> never in <other-repo>; background/async; UI; shared utilities). Body `TODO: ownership rules.`
- `## Flow conventions (the "law")` — comment: sync vs async default, how long-running work completes (queue/polling/callbacks/websockets), error-handling, non-negotiables. Body `TODO: flow conventions.`
- `## Entry-point / role → repo taxonomy (for tracing)` — "When identifying architecturally significant nodes, map roles to repos as:"; comment: derivable via `find_entry_points`/`list_flows`. Body `- TODO: \`<role>\` → \`<repos>\``.
- `## Symptom → likely region (for debugging)` — table `| Symptom | Likely region |` with one TODO row (`TODO: e.g. "API returns null / wrong shape"` | `TODO: likely layer/repo`).
- `## Tech stack` — comment: derivable from which recognizers fired (`index_status`, `search_index` by node type); also note what is ABSENT ("there is no GraphQL/Lambda/queue here"). Body `TODO: tech stack summary.`
- `## Documentation landscape` — where design-intent docs live vs docs that only look like it. Classifier uses path/filename/content signals and is sometimes wrong (`plans/PRD.md` example); rules here **override** inference; leave empty when inference is right. Format: one rule per bullet, `` `glob` `` → genre; genres `intent`, `execution`, `adjudication`, `reference`, `instruction`, `boilerplate`, `derived`. The two examples are commented out **per line** because the parser is line-based and does not skip HTML comment blocks or code fences (a plain bullet becomes a live override in every seeded doc):
  ```
  <!-- - `plans/**` → intent -->
  <!-- - `docs/api/**` → reference -->
  ```
  Glob syntax: `**/` matches zero or more directories, trailing `/**` matches everything below, `*` does not cross `/`. Check with `npm run build-index -- --docs --project <name>`, which warns for any glob matching no files.

### `project_conventions/README.md`

`# Project Conventions`. Content to reproduce:
- Each `<project>.md` is hand-authored domain knowledge for one Argus project — the "law" the sub-agents (flow-tracer, flow-designer, debugger, spec-explorer) reason from: repo ownership, async/flow conventions, role→repo taxonomy, symptom→region maps, tech stack, documentation landscape.
- **How it's used:** `.argus.config` project entry may set `conventions` (e.g. `"conventions": "project_conventions/<name>.md"`); `list_projects` surfaces the path; the orchestrator reads the active project's doc and injects it as leading context into every sub-agent dispatch (agents carry no org-specific architecture); the **Documentation landscape** glob→genre rules override inferred classification, consumed by `list_spec_docs`/`grep_spec_docs`/`get_spec_doc` and spec-explorer; absent/empty = pure inference.
- **Local by default (not committed):** per-user config like `.argus.config`/`.argus/`; may contain org-internal architecture; directory gitignored except this README and `_template.md`. Lives here rather than `.argus/` so it survives index rebuilds (`.argus/<project>/` is deleted wholesale). Not backed up by the repo — losing one means redoing onboarding Phase 2; keep your own copy. A few older docs remain tracked; new docs stay local.
- **Adding a project:** onboarding Phase 2 (`bootstrap-argus`, after index build and MCP restart) auto-seeds a starter `<project>.md` from the built index (repo list, detected tech stack, entry-point taxonomy); the user refines judgment calls (async law, ownership). Sections from `_template.md`: Repos & ownership, Flow conventions, Entry-point taxonomy, Symptom → likely region, Tech stack, Documentation landscape.

## Acceptance check

1. **UNDERSTAND with a mutation, single project.** Ask "argus, how does `<someMutation>` work?" Expect: `list_projects({divergence:true})` first (no question about which project when only one exists; a one-time divergence heads-up only if a repo is ahead/behind/dirty); `index_status` before dispatch; `find_mutation_handler` → resolver's full node ID; flow-tracer dispatched with conventions block (if configured) then the INDEXED CODE PATHS block then the prompt; a single `list_spec_docs` enrichment probe; final answer in the `**Argus** — \`<subject>\`` / TL;DR / narrative / *Key facts* shape with no inline diagram, an offer line for the `.md` report, and (if citable docs exist) a *"Want the why?"* offer rather than auto-run. Accepting the report prompts for a save location defaulting to `./argus-traces/<subject>-<YYYY-MM-DD>.md`.
2. **WHY question.** Ask "argus, why does the docx compare flow mark comments dirty — intentional or drift?" Expect: routed WHY (rationale question); no spec-explorer before `list_spec_docs` + 1–3 `grep_spec_docs` with generated synonyms; corpus-health fields read; ≤4 buckets of ≤~8 docs with one topic thread kept in one bucket; spec-explorers dispatched in parallel WITHOUT the code-paths block; answer separates stated (quoted, `repo/path (date)`) from inferred ("the docs do not say directly, but…"), currency usually `unknown`/"as of <date>", `aheadOfIndex` hedged as *may be* newer; no `argus-traces/**` citation. Against a project with only reference/instruction docs: stop with a `summary.byGenre`/`excluded` breakdown, no dispatch, and an offer to UNDERSTAND instead.
3. **Degraded index / simple lookup.** With `lastBuildFailed` set, ask "find the resolver for X": expect a direct single-tool answer with `file:line`, a passing warning about the partial graph (not a refusal), and no report offer. With `codeReadWarning` set, a DEBUG dispatch omits the INDEXED CODE PATHS block, instructs MCP-tools-only, and recommends a rebuild; with a repo `indexedTreeMissing: true`, that repo is omitted from the block (no checkout fallback) and called out as unavailable.
