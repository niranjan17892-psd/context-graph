# Skill: argus-retrospective

## Files to create

- `skills/argus-retrospective/SKILL.md` — the only file. A post-session diagnostic that reviews the conversation's `mcp__argus__*` tool calls and produces a structured feedback report (tool-call log, pinch points, index coverage, efficiency score, recommendations). No scripts, no sub-agents, no reference files.

## Frontmatter (verbatim)

```yaml
---
name: argus-retrospective
description: Use after an Argus session to review MCP tool usage, identify wasted calls, pinch points, index gaps, and produce actionable feedback for improving the Argus index and skill prompts
---
```

Body title: `# Argus Retrospective — Tool Usage Diagnostic`. One-line intro: review an Argus session's MCP tool calls, identify what worked vs what was wasted, surface index coverage gaps, produce a structured feedback report with actionable improvements.

## Purpose; fires when / does not fire when

Fires ("When to Use"):
- After completing an Argus UNDERSTAND/DEBUG/BUILD/WHY session
- When the user asks "how did that go?", "what worked?", "give me feedback on the tools"
- When an Argus trace required falling back to raw source reading
- When you suspect the index has coverage gaps for a domain

Does not fire for sessions with no Argus tool calls to review (it only reviews calls that actually happened), and it does not itself trace flows or fix the index.

## Procedure

1. **Inventory all Argus tool calls.** Scan the conversation for every `mcp__argus__*` call and record per call:

   | Field | What to capture |
   |-------|-----------------|
   | **Tool** | The tool name (e.g., `search_index`, `trace_flow`) |
   | **Args** | Key arguments passed (query, types, repos, entry_point) |
   | **Result category** | One of: `useful`, `partial`, `empty`, `overflow`, `shallow` |
   | **Result detail** | What was returned — count of results, whether edges existed, whether it overflowed to temp file |

   Category definitions:
   - **useful** — returned actionable data that directly informed the answer (right nodes, a real trace with call-graph edges)
   - **partial** — some relevant results mixed with noise, or results that only narrowed the search
   - **empty** — returned `[]` or no matches
   - **overflow** — exceeded token limits and was dumped to a temp file (effectively lost)
   - **shallow** — ran but returned only community/cluster edges (`MEMBER_OF`), no call-graph edges; or only the input node with no upstream/downstream connections

2. **Identify pinch points** (a call or pattern of calls that failed to produce expected results). Known pinch points:

   | Pinch Point | Symptoms | Root Cause |
   |-------------|----------|------------|
   | **Unfiltered search overflow** | `search_index` result dumped to temp file | Query too broad, no `types` or `repos` filter |
   | **Shallow trace** | `trace_flow` returns ≤2 nodes, only `MEMBER_OF` edges | Index lacks call-graph edges for this domain — C-tier extraction or cross-repo-link phase didn't connect these nodes |
   | **Empty entry points** | `find_entry_points` returns `[]` | Frontend → mutation edges not indexed (Vue component → GraphQL mutation usage not captured) |
   | **Missing service classes** | `search_index` with `types: ["Function"]` misses backend services | Services are indexed as `Class`, not `Function` — wrong type filter used |
   | **Wrong type filter** | Search returns schema fields instead of resolvers/mutations | Used `GraphQLOperation` (includes all schema fields) when `Resolver` was more specific |
   | **Redundant calls** | Same tool called multiple times with slight variations, all returning the same empty/shallow result | Should have escalated to source reading after first failure |

   For each pinch point found record: (1) which call(s) hit it, (2) how many tokens/calls were wasted before escalation, (3) what the correct action would have been.

3. **Assess index coverage** for the traced domain with this checklist, marking each **indexed** (trace_flow found the edge), **missing** (had to read source to discover), or **unknown** (not tested this session):
   - Frontend → Mutation: do Vue components have edges to the GraphQL mutations they call?
   - Resolver → Service: do resolvers have edges to the service classes they instantiate?
   - Service → Service: edges to downstream services they call?
   - Service → External: edges to external calls (S3, OpenAI, Bedrock, SQS)?
   - C-tier call sites: function-to-function call sites captured with file:line precision?

4. **Score the session:** `Efficiency = useful_calls / total_argus_calls × 100%`

   | Score | Rating | Interpretation |
   |-------|--------|----------------|
   | 80-100% | Excellent | Index well-connected for this domain |
   | 50-79% | Adequate | Some gaps but index provided a starting point |
   | 20-49% | Poor | Mostly fell back to source reading |
   | 0-19% | Index blind spot | Domain effectively not traced in the index |

5. **Generate recommendations** in two categories:
   - **A. Immediate (skill/prompt improvements)** — changes to the argus SKILL.md, agent prompts, or search strategies that prevent wasted calls. Examples: "Add type filter guidance: always search with `types` on first attempt"; "Add shallow-trace escalation rule: if trace_flow returns ≤2 nodes, dispatch source reader immediately"; "Document that backend services are indexed as `Class`, not `Function`".
   - **B. Index improvements (pipeline/extractor work)** — examples: "Vue component → GraphQL mutation edges missing — needs a recognizer or cross-file-bind rule"; "Resolver → service class instantiation not captured by C-tier extraction"; "DOCX/PDF processing services don't have edges to external API calls (OpenAI, S3)". Each must reference the relevant pipeline phase: `scan` / `structure` / `parse` (file discovery and AST extraction), `c-tier-extract` (statement-level call sites), `x-file-bind` (cross-file symbol resolution), `cross-repo-link` (cross-repo edge creation), `recognize` (higher-level pattern recognition: resolvers, agents, etc.), `flow-detect` (named flow assembly from edges).

## Hard rules

Behavioral rules, verbatim:
- **Only review tool calls that actually happened in this conversation.** Do not speculate about calls that weren't made.
- **Be specific about waste.** "Wasted 4 calls" is better than "some calls were inefficient."
- **Distinguish skill problems from index problems.** A bad search query is a skill issue; a missing edge type is an index issue. The fix is different.
- **Reference concrete pipeline phases** when recommending index improvements — the user maintains the pipeline and needs to know which phase to modify.
- **Don't editorialize about the index quality.** State facts: "trace_flow returned 1 node with 0 call-graph edges for this resolver" not "the index is bad."

## Sub-agents

None.

## Scripts/helpers relied on

None. Operates purely on the conversation transcript (the `mcp__argus__*` calls and their results).

## Output format (verbatim template)

```markdown
# Argus Retrospective — [domain/flow name]

## Session Summary
- **Mode**: UNDERSTAND | DEBUG | BUILD
- **Subject**: [what was traced]
- **Total Argus tool calls**: N
- **Efficiency score**: X% (rating)

## Tool Call Log

| # | Tool | Key Args | Category | Notes |
|---|------|----------|----------|-------|
| 1 | search_index | query="X", types=[...] | useful | Found 18 relevant nodes |
| 2 | trace_flow | entry_point="..." | shallow | Only MEMBER_OF edges |
| ... | | | | |

## Pinch Points

### 1. [Pinch point name]
- **Calls affected**: #2, #5
- **Tokens wasted**: ~N (estimate)
- **What happened**: [description]
- **Correct action**: [what should have been done]

## Index Coverage for [domain]

| Edge Type | Status | Evidence |
|-----------|--------|----------|
| Frontend → Mutation | missing | find_entry_points returned [] |
| Resolver → Service | missing | trace_flow had no edges |
| ... | | |

## Recommendations

### Skill/Prompt Improvements
1. [recommendation with rationale]

### Index Pipeline Improvements
1. [recommendation referencing pipeline phase]
```

(Note: the template's Mode line lists only UNDERSTAND | DEBUG | BUILD even though "When to Use" includes WHY sessions; reproduce as-is.)

## Acceptance check

1. **Shallow trace session.** Conversation had 5 argus calls: `search_index` (useful), `trace_flow` returning 2 nodes with only `MEMBER_OF` (shallow), the same `trace_flow` retried twice with slight variations (shallow), `get_context` (useful). Report: efficiency 40% "Poor"; pinch points "Shallow trace" and "Redundant calls" citing calls #2-#4 with "wasted 2 calls" and correct action "escalate to source reading after the first shallow result"; index recommendation names `c-tier-extract` or `cross-repo-link`.
2. **Overflow.** An unfiltered `search_index` overflowed to a temp file. It is categorized `overflow`, listed under "Unfiltered search overflow", and the fix is filed as a skill/prompt improvement (add `types`/`repos` filters), not an index improvement.
3. **No speculation.** If the session never called `find_entry_points`, the Frontend → Mutation coverage row is `unknown`, and no pinch point about it is invented; wording stays factual (counts of nodes/edges), not "the index is bad".
