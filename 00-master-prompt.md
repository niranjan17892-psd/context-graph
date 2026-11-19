# Argus — master rebuild prompt

> **How to use:** copy the whole `rebuild/` folder into an empty git repo at `rebuild/`. Then start an
> agent session there with: *"Read `rebuild/00-master-prompt.md` and execute it."* Everything below is
> addressed to that agent.

---

## 0. Your job

You are rebuilding **Argus**. Argus is a project-agnostic tool that maps how code flows across many
repos. It indexes one or more **projects** (each a named group of 1–N git repos) into a directed code
graph stored in an embedded graph DB. Agent CLIs (Claude Code, Codex, GitHub Copilot CLI) reach that
graph through an **MCP server**. A set of **skills** teaches the agent to trace, debug, design, and
explain flows that cross repos.

The spec is split across these files. Read each one at the milestone that names it:

| File | Covers |
|---|---|
| `rebuild/00-master-prompt.md` | this file: stack, model, pipeline, invariants, milestones |
| `rebuild/appendix-engine.md` | the 38-tool param table, phase-dep table, cross-repo kinds, .NET analyzer spec, fixtures |
| `rebuild/installer.md` | setup / fix-links / uninstall, client adapters, linker, hooks, repo dotfiles |
| `rebuild/skills/*.md` | one rebuild spec per skill or skill family |

**Working rules**
- Work milestone by milestone (§12). For each one: write a short design note, write failing vitest tests, implement, run `npm run build && npm test`, and paste the passing output. Only then start the next milestone. Never claim done without that evidence.
- Keep `CLAUDE.md` and `README.md` current as you go. Any change that alters behavior, setup, tools, config, or commands updates those docs in the same change.
- **Be faithful to the spec.** Where the spec and your instinct disagree, follow the spec. The exceptions are the items in §11 (Known gaps), which you **fix by default**, and anything you ask me about.
- Counts in this document (38 tools, 16 phases, 67 tables, 403 edge pairs) describe the original. Don't hard-code them in prose. Assert them in tests that derive them from code.
- Ask me before any decision this spec does not cover.

## 1. Stack (fixed)

- **Runtime and build:** Node **≥ 20.11** (the code uses `import.meta.dirname`). TypeScript strict, ESM (`"type":"module"`, `module`/`moduleResolution: Node16`, target ES2022, `rootDir: src`, `outDir: dist`). Exclude tests from tsc. Tests use vitest ^4 + tsx. No linter.
- **dependencies:** `@ladybugdb/core ^0.16.1`, `@modelcontextprotocol/sdk ^1.27.1`, `zod` (declare it explicitly; the original only got it transitively), `@vue/compiler-sfc ^3.5`, `graphology ^0.25.4`, `graphology-communities-louvain ^2.0.2`, `graphql ^16`, `tree-sitter ^0.21.1`, `tree-sitter-typescript ^0.21.2`, `tree-sitter-c-sharp ^0.21.3`, `tree-sitter-python ^0.21.0`, `typescript ^5.9`.
- **devDependencies:** `tsx`, `vitest`, `@types/node`. Hand-write `src/types/ladybug.d.ts`, because the package ships no types.
- **Do NOT add** `ts-morph`, `pyright`, `js-yaml`, or `libcst`. The original only used them from dead code (§11).
- **Python 3.10+:** `requirements.txt` is the single source of truth for Python deps. It contains only `sentence-transformers>=2.2.0`. Embeddings come from model `all-MiniLM-L6-v2` (384-dim) via `src/mcp-server/embed/python-embedder.py`, called as a subprocess.
- **npm scripts:**
  - `build`: `tsc && node scripts/copy-assets.js`. This copies `src/**/*.py` into `dist/`; without it, embed fails when run from dist.
  - `test`: `vitest run`
  - `build-index`: `node --max-old-space-size=4096 --import tsx/esm src/cli/build-index.ts` (runs from TS source)
  - `setup`, `fix-links`, `uninstall`, `project`: `node scripts/<name>.js`
  - `eval:triggers`: `node scripts/eval-triggers.mjs`
- **vitest excludes:** `.claude/**`, `.argus/**` (indexed worktrees contain foreign test suites), and `tests/brainstorm-server/**` (a separate package with its own runner).
- **MCP server:** identity `{name:'argus-panoptes', version:'2.0.0'}`, stdio transport. `.mcp.json` registers `argus` → `node dist/mcp-server/index.js` with `ARGUS_CONFIG=.argus.config`.

## 2. Repo layout

```
src/cli/build-index.ts
src/mcp-server/index.ts                 TOOL_DEFS (source of truth for tools) + registration
src/mcp-server/{config,project-config,project-store-registry}.ts
src/mcp-server/pipeline/                build-index.ts, PhaseDAGRunner, phases/ (16)
src/mcp-server/extractor/               tree-sitter-base (parseWithBuffer), per-language extractors, c-tier/, data-flow/
src/mcp-server/recognizers/             23 recognizers + claim-callsites, ast-utils, is-test-path
src/mcp-server/storage/                 schema.ts, argus-store.ts, csv-loader.ts, merge-node.ts, lock.ts, meta-state.ts,
                                        resolution.ts, filter-edges.ts, doc-summary.ts, schema-guard.ts
src/mcp-server/{cluster,flows,cross-repo,embed,summaries,type-resolver,branch-handling,output-formats,cypher-templates}/
src/mcp-server/tools/                   one file per tool, stateless
src/sample-code/manifest.ts
src/spec-docs/                          classify, manifest, safe-path, tools
scripts/                                see rebuild/installer.md
skills/<name>/                          see rebuild/skills/*.md
hooks/                                  superpowers SessionStart hook (rebuild/skills/superpowers.md)
project_conventions/{README.md,_template.md}   (rebuild/skills/argus.md)
tools/argus-dotnet-analyzer/            .NET 8 Roslyn analyzer (appendix-engine.md)
tests/                                  mirrors src/ and scripts/; tests/brainstorm-server is its own package
docs/                                   manual-install.md, argus/{README,dotnet-analyzer}.md, argus/reference/{mcp-tools,confidence,output-formats}.md
```

## 3. Config model

`.argus.config` is gitignored JSON. `ARGUS_CONFIG` overrides its path. The repo ships a `.argus.config.example`.

```json
{
  "installTargets": [{ "client": "claude|codex|copilot", "home": "...", "pathFormat": "posix|windows", "wslHome": "...", "local": true }],
  "dailyDriverWorktree": "/abs/path",
  "projects": {
    "<name>": { "repos": { "<repoKey>": { "path": "/abs" } }, "repoRoot": "...", "sampleCodePath": "...", "conventions": "project_conventions/<name>.md" }
  }
}
```

**Validation and loading**
- Project names match `/^[A-Za-z0-9._-]+$/` and are not `.` or `..`. Validate on write (`scripts/project.js`) **and** on load (skip bad entries with a warning).
- Repo values may be `{path}` or a bare path string (legacy). Handle both everywhere.
- A legacy flat config (top-level `repos`) migrates to project `tcn` on the next build, writing `.argus.config.bak` first.
- Type `installTargets` **once**, in the shape above. Older entries `{claudeHome,…}` normalize on load.

**Project resolution order:** explicit `project` arg → `ARGUS_PROJECT` → the sole project → otherwise an error that lists the project names.

**Environment variables:** `ARGUS_CONFIG`, `ARGUS_PROJECT`, `ARGUS_DOTNET_GRAPH_PATH`, `ARGUS_DOTNET_GRAPH_ALLOW_STALE`, `ARGUS_REPO_ROOT`+`ARGUS_UNINSTALL_TEST` (test seam), `VIRTUAL_ENV`.

**Per-project directory `.argus/<project>/`:**
- `ladybug`, `ladybug.wal`, `ladybug.lock`: the DB, deleted before every build
- `meta.json`
- `worktrees/<repoKey>`
- `docs-manifest.json`
- `dep-graph/symbols.json`
- `embed-cache/cache.jsonl`: survives rebuilds
- `dotnet-graph/<repoKey>.json`
- `build.lock/`, `meta.lock/`

The sample-code manifest is **not** per project. It lives at `<argus repo>/cache/sample-code-manifest.json`.

## 4. Graph model and storage

**Node tables (67)** are declared in `B_TIER_NODE_TABLES` and created with `CREATE NODE TABLE … PRIMARY KEY(id)`. Schema type `INT`→`INT64`, `BOOL`→`BOOLEAN`.

| Group | Tables |
|---|---|
| Structural | Repo, Folder, File, Module, Package |
| Symbol | Function, Method, Class, Interface, TypeAlias, Enum, Property, Const, Variable |
| Domain | GraphQLOperation, GraphQLType, Resolver, VueComponent, Composable, PiniaStore, Route, LambdaHandler, StepFunction, StepFunctionState, CDKResource, SQSTopic, SNSTopic, Neo4jQuery, SocketIOEvent, SocketIONamespace, SocketIORoom, CliCommand, McpTool |
| Stack-specific (promotions) | AngularComponent (+`selector`), AngularService, FoundryAgent, AspNetController, AspNetRoute, BackgroundWorker, SignalRHub, GrpcService, DbContext |
| C-tier | CallSite, Branch, TryBlock, ErrorHandler, ThrowSite, AwaitPoint |
| Comment | Comment |
| D-tier | ParamBinding, ReturnBinding, VariableBinding, PayloadShape, FieldAccess |
| Agent | AgentPackage, BedrockAgent, AgentTool, AgentRuntimeArn, AgentInvocation, PromptReference, PromptLiteral, FoundryAgentInvocation, PydanticAiAgent |
| Synthesized | EntryPoint (+`summaryCard`), NamedFlow, FlowStep, Community |

**Column sets**
- `COMMON_SYMBOL_COLS` = id, name, repo, filePath, startLine, endLine, language, signature, docSummary, metadata (JSON string).
- `STRUCTURAL_COLS` = id, name, repo, filePath, metadata.
- `COMMENT_COLS` = id, name (`comment:L<n>`), repo, filePath, startLine, endLine, language, text, kind (`docblock|leading|inline|header|commented-code`), markers, ownerId, metadata.
- Embeddable tables add an `embedding` STRING column.

**Node IDs**
- Symbol: `<repoKey>::<relPosixPath>::<name>`. File: `<repoKey>::<relPath>`.
- CallSite: `<repo>::<relPath>::callsite::<target>:L<line>`. `claim-callsites` must derive the identical ID.
- EntryPoint `entry::<nodeId>`, NamedFlow `flow::<nodeId>`, FlowStep `step::<startId>::<order>`, Community `community::<n>`.
- Synthetic line-keyed IDs: `cdk:<call>:L<n>`, `socket-emit:L<n>`, `invoke:L<n>`, `SQSTopic:L<n>`.
- Promotion keeps the ID and changes the table: `nodeTableMap.set(id, NewTable)`.

**Edges: one rel table group.**
- `REL TABLE GROUP CodeRelation` is declared `FROM A TO B` for every `[A,B]` in `EDGE_PAIRS`. These are untyped pairs (403 in the original).
- Edge properties: `type STRING, confidence DOUBLE, reason STRING, resolution STRING, candidateCount INT64, sourceFile STRING, sourceLine INT64, evidence STRING, metadata STRING`.
- Queries write `-[r:CodeRelation]->` and filter on `WHERE r.type = 'X'`.
- **Persist:**
  - Group edges by (fromTable, toTable) and bulk-load each group with `COPY CodeRelation_<From>_<To> FROM '<csv>' (HEADER=true)`.
  - Nodes are CSV-`COPY`'d too, which is why the DB is deleted before every build: a reload would otherwise hit duplicate primary keys.
  - The original falls back to per-row `MATCH…CREATE` with failures swallowed when a sub-table is missing. That swallowing is the silent-drop mechanism; see §10 invariant 1 and §11.
  - An endpoint with no `nodeTableMap` entry defaults to `File`.

**Edge types actually emitted:**
- CONTAINS, IMPORTS, CALLS, INVOKES, HAS_CALLSITE, CAUGHT_BY, RETHROWS
- HAS_DOCBLOCK, HAS_COMMENT
- HANDLED_BY, PUBLISHES_TO, EMITS_SOCKET_EVENT, INVOKES_AGENT, USES_TOOL, LOADS_PROMPT
- DEFINES, STEP, STARTS, RETURNS, EXTENDS, IMPLEMENTS
- MEMBER_OF, ENTRY_POINT_OF, STARTS_FLOW, STEP_IN_FLOW
- Roslyn import: INHERITS, RESOLVES_TO, INJECTS, HANDLES_REQUEST, DISPATCHES_TO, MANAGES, NAVIGATES_TO

**Declared but not produced** (consumers exist): TRIGGERS, HANDLES_SOCKET_EVENT, SUBSCRIBED_BY, PRODUCES_SHAPE, CONSUMES_SHAPE, READS_FIELD, RENDERS. See §11. `GUARDS` does not exist anywhere.

**Edge confidence (#38)**
- Every edge carries `confidence` (0–1), `resolution`, and `candidateCount` (default 1).
- Bands: PRECISE ≥ 0.85, HEURISTIC ≥ 0.5, otherwise AMBIGUOUS. Canonical defaults are 0.95 / 0.75 / 0.4.
- `AMBIGUOUS` is set explicitly only at the two bare-name resolution sites (`resolveCallTarget` in `callsite-extractor.ts`, and `x-file-bind`) when there are >1 same-name candidates.
- Every other edge's label is derived from its confidence at persist (`normalizeEdgeResolution`), so the label and the number always agree.

**Calls are three hops:** `Function → HAS_CALLSITE → CallSite(file:line) → INVOKES → target`.
- `flattenCallSiteEdges()` projects these into in-memory direct edges. It is memoized once per build on `PipelineState.projectedEdges` via `getProjectedEdges()`.
- `cluster` and `flow-detect` both use that projection. Line-precision tools use the raw chain.

**Node dedup** (`merge-node.ts`): nodes sharing an ID are merged field by field. Non-empty later scalars win, empty/null/undefined never clobber, and `metadata` is deep-merged with arrays unioned.

**Comments**
- `docSummary` is the first line of the doc comment, capped at 100 chars (`toDocSummary`). The full prose lives on the `HAS_DOCBLOCK` Comment node; fetch it with `get_comments`.
- `Comment` is excluded from default `search_index` scans (`SEARCH_EXCLUDED_TABLES`).
- Keep `COMMENT_NON_OWNER_PROMOTIONS` as an asserted list: every promotion target is in exactly one of it or `COMMENT_OWNER_TABLES`.
- `CURRENT_SCHEMA_VERSION = '2.1.0'` is checked on read. Against an older index, `get_comments`/`grep_comments` return `{ok:false, reason:'index-schema-outdated'}`.

**ArgusStore** wraps LadybugDB. The MCP server opens it read-only. `listEmbeddings(tables)` whitelists table names against `B_TIER_NODE_TABLES`, as a guard against Cypher injection.

## 5. Indexing pipeline

**`runBuildIndex`** (`pipeline/build-index.ts`) runs under `withBuildLock`:
1. Preflight `sentence-transformers`. If it is missing, hard-fail with install instructions.
2. Remote mode only: stamp `treesAheadOfIndex:true` on the existing meta under `withMetaLock`. Then fetch and advance one detached worktree per repo at `.argus/<project>/worktrees/<repoKey>`, pointing at `origin/<default>`. Never touch the developer's checkout.
   - `--local` indexes the checkouts as they are.
   - A repo with no resolvable `origin/<default>` falls back to its checkout (`indexedRef:"local"`).
   - Git timeouts are named constants in `GIT_TIMEOUTS`: fetch 120s, checkout 600s, clean 300s. Local probes (rev-parse, `ls-remote --symref`, worktree list/prune/unlock) get 5–10s.
3. Stamp `buildInProgress:true`, then delete `ladybug*`.
4. Open a WRITABLE store and run the DAG.
5. **Close the store BEFORE** the terminal meta publish under `withMetaLock`. Readers treat a changed `indexedAt` as "reopen now", so publishing first races a failed reopen.
6. On any throw: close the store and set `buildInProgress:false` plus `lastBuildFailed:{at,phases}`.

**meta.json fields:**
- Identity and stats: `schemaVersion`, `indexedAt`, `repoChoices`, `lastFullBuild{timestamp,durationMs}`, `phaseStats`, `cacheStats`.
- `fileHashes`, shaped `{repoKey:{relPosixPath: gitBlobSha}}`, where `gitBlobSha = sha1("blob "+size+"\0"+content)`.
- Distributions: `confidenceDistribution`, `resolutionDistribution`, `localBehindRemote`.
- Refs and paths: `indexedHeads`, `indexMode`, `indexedRefs`, `indexedTreePaths`, `checkoutPaths`.
- Health: `treesAheadOfIndex`, `lastBuildFailed{at,phases}`, `buildInProgress`.
- `lastDriftAudit`, written by `drift_audit`.

**PhaseDAGRunner** is concurrent.
- When a phase finishes, every dependent whose deps are now satisfied is scheduled immediately, in registration order.
- A phase that throws goes into `failed` and all its transitive dependents are `skipped`. The build still publishes meta, with `lastBuildFailed` and `treesAheadOfIndex:true`.
- Each recognizer is try/caught on its own, so one bad recognizer never fails the `recognize` phase.

**16 phases.** The full declared-vs-actual table is in `appendix-engine.md`. **Declare complete deps** (the original under-declares, see §11):

| phase | deps (rebuild) |
|---|---|
| scan | — |
| structure, parse | scan |
| type-resolve | parse |
| x-file-bind | type-resolve, structure |
| c-tier-extract, data-flow-build, recognize, dep-graph | x-file-bind |
| comment-extract | x-file-bind, **recognize** (owners are validated against FINAL post-promotion tables) |
| cross-repo-link | **recognize** (consumes `boundaryEntries`) |
| persist | structure, x-file-bind, c-tier-extract, data-flow-build, recognize, cross-repo-link, comment-extract |
| flow-detect, cluster, embed | persist |
| summaries | flow-detect |

**Phase details**
- **scan** walks each repo, skipping `SKIP_DIRS` = node_modules, dist, .git, __pycache__, coverage, .next, .nuxt, venv, .venv, .aws-sam, generated, .argus, .claude, .superpowers, worktrees, .angular, .nx, .turbo, .pytest_cache, .mypy_cache, .ruff_cache. `drift_audit` MUST import this same constant, or it reports phantom drift.
  - Buckets: `.asl.json`→aslJson. `.ts/.js/.tsx/.jsx`→ts, excluding `*.test.*`, `*.spec.*`, `*.d.ts`. Then `.vue`, `.py`, `.graphql/.gql`, `.cs`.
  - Every file also goes into `all`, with its git blob hash from the index (empty if untracked).
- **parse** uses tree-sitter (TS/TSX, C#, Python), `graphql`, `@vue/compiler-sfc`, and JSON for ASL. **Always** parse through `parseWithBuffer()` with `{bufferSize: source.length + 1}`. Sources of ≥ 32,768 UTF-16 code units otherwise throw `Invalid argument` and are silently skipped. When the phase skips files, it warns with the first extraction error.
  - `@def.function` captures the NAME only. Walk up the parent chain (and sideways to `variable_declarator`/`pair`) to record `defStartLine/defEndLine`, which `findEnclosingDef` relies on.
- **type-resolve** uses the heuristic `TieredResolver` only. Three tiers: annotation 0.98, constructor 0.92, assignment chain 0.80. Run to fixpoint, max 10 iterations.
- **x-file-bind** creates symbol nodes plus CONTAINS, IMPORTS, and CALLS edges.
- **c-tier-extract** creates CallSite, Branch, TryBlock, ErrorHandler, ThrowSite, and AwaitPoint nodes, with HAS_CALLSITE, INVOKES, CAUGHT_BY, and RETHROWS edges.
- **recognize** runs the 23 recognizers (§6), in order.
- **cross-repo-link** matches every consumer boundary entry against every publisher of the same `boundaryType`. It emits a consumer→producer edge, typed by `mapBoundaryToEdgeType`, with confidence from `matchBoundary`. Kinds are listed in the appendix.
- **dep-graph** builds `dep-graph/symbols.json` (file → importing files) from IMPORTS edges.
- **flow-detect**
  - Entry scoring by table: VueComponent .95, Composable .7, Route .85, AspNetRoute .85, GraphQLOperation .9 (mutation) / .8 (other), LambdaHandler .9, StepFunction/BedrockAgent/PydanticAiAgent/BackgroundWorker/GrpcService/CliCommand/McpTool .85, SignalRHub .8, SocketIOEvent .75. Nothing else is an entry point.
  - For each entry, emit `EntryPoint –ENTRY_POINT_OF→ node`, `EntryPoint –STARTS_FLOW→ NamedFlow`, and `NamedFlow –STEP_IN_FLOW→ FlowStep`.
  - BFS with MAX_DEPTH 8 and MAX_STEPS 50. Step 0 is the entry itself (`ENTRY`).
  - Follow only FLOW_EDGE_TYPES {TRIGGERS, HANDLED_BY, CALLS, INVOKES, PUBLISHES_TO, EMITS_SOCKET_EVENT, HANDLES_SOCKET_EVENT, INVOKES_AGENT, USES_TOOL, STEP, STARTS, SUBSCRIBED_BY, DEFINES}. Use the flattened projection only when a node has outgoing edges but none of them carry flow.
  - Flow name is `<entryName> <kind suffix>`, plus ` (stub)` when it has 0 steps.
- **cluster** runs Louvain over {CALLS, INVOKES, TRIGGERS, HANDLED_BY, IMPORTS, USES_TOOL} plus the projected edges. Sort the inputs and use a fixed-seed mulberry32 RNG so communities are reproducible. `MEMBER_OF` confidence is 0.7.
- **embed**
  - EMBED_TABLES = Function, Method, Class, BedrockAgent, NamedFlow, PromptReference, AngularComponent, AngularService, FoundryAgent, PydanticAiAgent. `Comment` is deliberately excluded; a test asserts this.
  - `buildText` = name + signature + the first 4 lines of the HAS_DOCBLOCK text (else docSummary), with per-table variants.
  - The embedder takes JSONL `{id,text}` on stdin and writes `{id,embedding[384]}`; `--probe` prints the model and dimension.
  - Cache key is `sha1(text)[0:16]`. Python is located via `findPython()`, which honours `VIRTUAL_ENV`.
  - Writing 0 vectors from >0 candidates is a failure: surface the Python stderr.
- **summaries** builds deterministic structured cards, with **no LLM**: a BFS of depth 4 over raw edges, recording repos touched and async boundary types, written to `EntryPoint.summaryCard`.

**CLI** `build-index` flags:
- `--project <n>`, `--local`, `--sample`, `--docs`, `--both`/`--all`, and `--index`/`--tcn` (legacy alias for structural).
- `--docs --project X` with no index flag means docs only.
- With no target flag: a numbered picker on a TTY; the index by default on a non-TTY.
- Bare words `sample|tcn|both|all|docs` print "did you mean `--x`? (npm needs `--`)".

## 6. Recognizers

**Contract:** `recognize(ctx) → {nodes, edges, nodeTableMap, boundaryEntries}`, where ctx = `{repoKey, captures, resolvedTypes, fileMap, argusDir, logger}`. Boundary entry = `{boundaryType, key, direction: publish|consume, nodeId, repoKey, metadata}`.

**Order matters for same-ID merges:** GraphQL, Vue, Pinia, Route (Express-style), CDK, StepFunction, Neo4j, CSharpLambda, BedrockAgent, PromptLoader, AgentTool, AgentToolUsage, SocketIO, Angular, FastAPIRoute (reuses `Route`), Foundry, AspNetCore, DotnetHostedService, EfCore, RoslynImport, PydanticAiAgent, CliCommand, McpTool.

**Rules**
- SQS/SNS nodes come only from CDK. The C# lambda recognizer emits `sqs-publish`/`sns-publish` boundary entries.
- The tree-sitter .NET recognizers are best-effort and tag `metadata.resolution='HEURISTIC'` plus `routeConfidence`.
- RoslynImport reads the analyzer graph (spec in appendix). It skips with a warning on a `commitSha` mismatch unless `ARGUS_DOTNET_GRAPH_ALLOW_STALE=1`.
- `CliCommandRecognizer` fires only in files that reference `process.argv`, and only on switches whose discriminant is a bare identifier.
- `McpToolRecognizer` needs `name`+`handler` plus one of `description|params|inputSchema|schema`.
- `McpTool` (a tool the repo serves) ≠ `AgentTool` (a tool an in-repo agent calls).
- Route, CliCommand, and McpTool claim the CallSites in their source range (`claim-callsites.ts`). That is what gives inline handlers multi-step flows.
- `trace_flow` resolves bare-name entries by role among exact-name matches: Resolver > LambdaHandler > StepFunction > Route > VueComponent > Function. Runners-up are returned as `alternatives`. A full node ID skips resolution.

## 7. MCP server and tools

- **38 tools.** Every name, param, default, and cap is in `appendix-engine.md`.
- `TOOL_DEFS` in `index.ts` is the source of truth. A test asserts its shape and count.
- `registerTool` **injects** an optional `project` param into every tool except `list_projects`. It is not declared in TOOL_DEFS.
- **ProjectStoreRegistry**
  - Lazily opens one read-only store per project.
  - **Reopens** on the next call when `meta.json` `indexedAt` changed and `treesAheadOfIndex` is false. So an out-of-process rebuild is picked up without a restart.
  - `rebuild_index` evicts the cached store.
  - Only a **newly added project** needs a server restart, because `.argus.config` is read once at startup.
- **Responses**
  - Results are `JSON.stringify(result,null,2)`; strings pass through as-is.
  - Thrown errors become `{content:[{type:'text',text}], isError:true}`.
- **`indexWarning`**
  - Added to every tool except `index_status`, `list_projects`, and `rebuild_index`.
  - Triggers: the generation captured at `resolve()` was unhealthy (precedence `lastBuildFailed` > `buildInProgress` > `treesAheadOfIndex`, distinct wording each), or `indexedAt` changed mid-request, or a build started mid-request. Once set, it sticks for that response.
  - Placement: objects get a top-level field; strings get a `[warning]\n\n` prefix; arrays and null are wrapped as `{indexWarning, result}`.
- **Output formats** (`output-formats/`): structured (default), narrative, mermaid (one subgraph per repo; AMBIGUOUS edges dashed), and ascii (indented tree; `⚠` on ambiguous edges). `get_flow_diagram` defaults to ascii and `describe_flow` to narrative.
- **Edge filtering:** the graph-walking tools take `minConfidence`/`excludeAmbiguous` through the shared `filterEdges`/`filterTrace`. Both default off ("mark, don't hide").
- **`index_status`** reports:
  - `indexFreshness`: integrity against the local remote-tracking ref, **not** currency.
  - `localBehindRemote`, `divergence{ahead,behind,dirty}`, `resolutionDistribution`.
- **`cypher_query` guard**
  - Mask strings and comments first.
  - Require the query to start with MATCH, OPTIONAL MATCH, WITH, UNWIND, or RETURN.
  - Reject CREATE, MERGE, SET, DELETE, DETACH, REMOVE, DROP, ALTER, COPY, LOAD, INSTALL, ATTACH, EXPORT, IMPORT, RENAME, BEGIN, COMMIT, ROLLBACK, any CALL, `;`, UNION, and unbounded var-length (`[*]`, `[*..]`, `[*N..]`).
  - LIMIT must be a literal ≤ 1000; append ` LIMIT 1000` when absent and return the executed cypher.
- **`cypher_template`**: 20 named templates; the list is in the appendix. An empty name lists them.
- **Caps:** `list_flows.limit ≤ 2000`, `semantic_search.k ≤ 100`, `search_index.limit ≤ 100`.
- **Missing worktree:** when a repo's recorded indexed tree path is gone, `get_context`/`drift_audit` refuse (`codeUnavailable` / `ok:false`, naming the path) rather than serving the checkout at the wrong line numbers.
- **Sample-code manifest:** cached per project in-process until restart.

## 8. Spec-docs subsystem (`src/spec-docs/`)

**Genres:** intent, execution, adjudication, reference, instruction, boilerplate, derived, **unknown** (no signal, or an unbreakable tie).

**Weighted vote:**
- path 2, filename 3
- content-family 3 (capped at 6 per genre)
- checkboxes 5 (when ≥ 5)
- frontmatter `name`+`description` 6, generated-banner 6
- conventions override 100

**Entry fields:** each entry carries `signals[]` and `confidence`. Conflicts drop to `low` and produce a `low-confidence-genre` flag. One glob line under `## Documentation landscape` in the conventions doc overrides inference, except that `argus-traces/**` is always `derived`.

**Enumeration**
- List docs with `git ls-files` (+ `--others --exclude-standard`), never a directory walk. A walk would ingest `.claude/worktrees` copies.
- Never follow symlinks. The build `lstat`s and counts skips in `skipped`. Read time re-checks with `checkDocPath`, including a realpath-inside-checkout test.

**Freshness**
- Read docs from the **checkout**, not the indexed tree.
- Record `docsAheadOfIndex` per repo (`git rev-list --count <indexedHead>..HEAD`).
- Flag `aheadOfIndex` and `uncommitted` (untracked or `git diff --name-only HEAD`). `uncommitted` docs are dated by mtime for `since`/`until`.

**Refresh**
- Refresh on use, stat-gated, debounced to 2s.
- Unchanged entries are shallow-copied from the previous manifest, never mutated.
- `conventionsFingerprint` = sha1 of the ordered `[glob,genre]` rules. When it changes, reclassify everything.
- `docs-manifest.json` is a cache: written atomically, and a write failure is never fatal.

**History:** one `git log --name-only` pass per repo, with no `--follow`.

**Citation rule:** only intent, execution, and adjudication docs may support a "why" claim. `derived` is never citable.

**Landscape fields:** `list_spec_docs` scopes `summary`/`excluded`/`docsAheadOfIndex` to the `repos` filter. It ignores the genre, keyword, ticket, and date filters for those fields.

## 9. Sample-code manifest

`buildManifest()` walks `sampleCodePath`, reads file headers and READMEs, and skips `dist` and binaries. It writes `cache/sample-code-manifest.json`.
- `grep_sample_code` uses `rg` when it is on PATH, otherwise a Node fallback.
- When `sampleCodePath` is unset, both tools return a guidance message.
- If more than one project sets `sampleCodePath`, the build errors.

## 10. Invariants (each was a real bug — encode as tests)

1. **One generic pair-coverage test.** Run every recognizer and phase over the fixtures, collect each emitted `(fromTable,toTable)`, and assert each pair is declared in `EDGE_PAIRS`. Also keep the targeted suites:
   - C-tier source permissions for every promoted def table.
   - Mirrored `Class` pairs for the .NET promotions.
   - `COMMENT_OWNER_TABLES` spread into EDGE_PAIRS, with every entry a real table.
   - `EntryPoint → <every ENTRY_KIND>`.
2. `comment-extract` validates owners against the final tables. If an owner is unknown, fall back to the File node rather than emit an edge that persist would drop.
3. Field-level node merge (§4).
4. Def-range capture through parent walking (§5 parse).
5. `parseWithBuffer` everywhere. Regression test sources of 32,767 and 32,768 code units, plus a 20k-char multibyte source.
6. `docSummary` is capped at 100. Comments are never embedded, but a node's docblock text feeds its own embedding.
7. Resolution bands and AMBIGUOUS sites (§4).
8. **Locks** (`storage/lock.ts`)
   - Never auto-reclaim a lock from a dead PID. Fail fast and print the lock path on its own line, with the recorded pid and start time. Never print a ready-made delete command.
   - While holding a lock, sync idempotent SIGINT/SIGTERM handlers release **every** lock this process holds, innermost first, through a module-level `heldLocks` registry. Each release is token-verified.
   - For the build lock, make one best-effort stamp under `meta.lock`: `buildInProgress:false`, `lastBuildFailed:{phases:['interrupted']}`. Skip it if `meta.lock` is contended.
   - Then `process.exit(128+signum)`.
   - Remove the handlers on release, so a long-lived MCP server never accumulates listeners.
9. Refuse to serve code when an indexed tree path is missing (§7).
10. `GIT_TIMEOUTS` constants (§5).
11. Registry reopen semantics (§7). Only added projects need a restart.
12. Embed with 0 vectors from >0 candidates is an error.
13. The index-health stamping order (§5) and the `indexWarning` shapes (§7).
14. Drift audit shares `SKIP_DIRS` and the git-blob hash algorithm with the build.
15. Never hand-write agent-client config, except the sanctioned cases in `installer.md`.
16. Headless `codex exec` cancels MCP calls unless you pass `-c 'mcp_servers.<n>.default_tools_approval_mode="approve"'`. Scope it with `enabled_tools=[…]`.

## 11. Known gaps in the original — FIX by default

Each item below is a defect in the original. Fix it, add a test, and write a line in `CLAUDE.md`. If you'd rather reproduce the original behaviour for a specific item, ask me first.

**Engine**
1. **Silent edge drops.** Make persist **fail loudly**, or at least warn with counts, when an emitted pair has no `CodeRelation_<From>_<To>` sub-table. Don't swallow it.
2. **Under-declared phase deps.** Use the dependency table in §5.
3. **Missing Resolver → GraphQLOperation pair.** Declare it, so a cross-repo GraphQL trace is client Composable → GraphQLOperation → Resolver end to end.
4. **Cross-repo linking is mostly dead.** Only GraphQL and Lambda-ARN↔Step-Function link today.
   - Key Socket.IO by event name (plus namespace), not by line.
   - Key agent runtime by agent name on both sides.
   - Add an HTTP-client consumer (fetch/axios/HttpClient with a literal or template URL → `METHOD:path`) for `rest-route`.
   - Add a `producer.repoKey !== consumer.repoKey` flag to edge metadata. Keep same-repo matches.
5. **Declared but never produced:** TRIGGERS, HANDLES_SOCKET_EVENT, SUBSCRIBED_BY, *_SHAPE, READS_FIELD, RENDERS.
   - Implement HANDLES_SOCKET_EVENT and SUBSCRIBED_BY with item 4.
   - Implement RENDERS for Vue/Angular templates.
   - For the D-tier shape edges, either implement PRODUCES_SHAPE/CONSUMES_SHAPE/READS_FIELD or remove the shape templates and the `impact_of_change` shape params. **Ask me which.**
6. Declare `zod`. Don't depend on `libcst`.
7. No dead code: no v1 `indexer/`, no `src/shared/types.ts` with a hardcoded repo list, no unwired ts-morph/pyright/roslyn resolvers, no `verify-deps.js`, no `python-ast-extractor.py`, no stub `install.sh`.
8. `.argus.config.example` must not mention LLM summary providers. Summaries are deterministic.

**Installer** (full list in `installer.md` §9; fix these, list the rest in CLAUDE.md as known quirks)
1. **Native Windows Claude Code.** Register it with `claude mcp add`. Write `claude_desktop_config.json` only for Claude Desktop and WSL-bridged targets.
2. **Cross-env Windows→WSL skill links.** Link into each client's own home dir, not always `<wslHome>/.claude/skills`.
3. **Unparseable `claude_desktop_config.json`.** Refuse and print the JSON, as the hook writer does. Never overwrite it with `{}`.
4. **MCP re-registration.** Add first (or verify that the add succeeded) before removing the old entry, so a failed add never leaves nothing registered.
5. **Shell quoting.** Never build shell strings; pass an argv array (`execFileSync`). Delete `quoteArgv`.
6. **The linker must not delete real directories.** If a real folder sits where a link should go, skip and warn. No recursive delete, and no `del /f /q`.
7. **Setup and fix-links use the main clone's `.argus.config`.** Resolve it even when run from the daily-driver worktree, and never create a nested worktree.
8. **Worktree detection.** Use the recorded `dailyDriverWorktree`, not the first `*-daily-driver` folder found. fix-links records any worktree it creates.
9. **Maintenance mode** also runs Python deps and the superpowers conflict check. Remember clients the user declined, and don't re-offer them.
10. **Uninstall scans for homes the same way setup detects them**, not only the recorded `installTargets`.
11. **Repo files.** `requirements.txt` holds only `sentence-transformers` (overriding installer.md §1). `.gitignore` gets no duplicates and the `/.claire/` typo fixed. `docs/manual-install.md` says Python is required and describes `wsl.exe bash -c`.

**Precedence.** When files disagree: this master prompt wins over `appendix-engine.md`, which wins over `installer.md` and `skills/*.md`. Those specs describe the original faithfully, defects included. This file decides what gets fixed.

**Skills** (source inconsistencies the spec writers found; each spec repeats them at its end)
1. **product-shared base.md** hard-codes "11 TCN repos" and "TCN codebase graph". Make it project-agnostic: resolve the project through `list_projects`.
2. **ux-review**
   - Add a stop/redirect after "Run discover first", matching `strategize`.
   - Gate its argus use on "only if argus is available".
3. **output-artifacts.md** names Atlassian tools that may not exist. Resolve the tool names at runtime from the available Atlassian tools instead of hard-coding `createConfluencePage`.
4. **backlog skills**
   - RICE Reach needs the `watches` field requested in the JQL fields.
   - Define the 1–3 Confidence and Effort scales explicitly (the worked example implies 1 = low, 3 = high; Effort 1 = smallest). Keep the Impact mapping as it is.
   - Use one tool-naming convention throughout.
5. **argus-retrospective:** add `WHY` to the template's Mode line.
6. **eval-package**
   - Pick one `kind` vocabulary for scores (human/model/measurement/mechanical).
   - Include `<environment>/` in every results path.
   - Use one pointer name, `LATEST`.
7. **build-gold-dataset:** `SKILL.md` wins. Bring the reference files into line with it: always log catches made before hand-off, use `locators`, and add a `**Field:**` line in the template.
8. **open-pr:** state explicitly that the ticket ID is stripped from the readable-branch part of the title.
9. **superpowers:** re-apply the lost local patch that links `adversary-review` from `finishing-a-development-branch` and `requesting-code-review` (text in `skills/superpowers.md` §c).

## 12. Milestones

| # | Milestone | Read first | Acceptance (plus every listed invariant green) |
|---|---|---|---|
| 1 | **Skeleton:** package.json, tsconfig, vitest config, `.gitignore`/`.gitattributes`, config loader (multi-project, validation, legacy migration), `scripts/project.js` (CLI spec in `skills/bootstrap-argus.md`) | installer.md §1 | Config round-trips; migration and name-validation tests pass |
| 2 | **Storage:** schema generated from `B_TIER_NODE_TABLES`/`EDGE_PAIRS`, ArgusStore, csv-loader, merge-node, locks, meta-state, schema guard, resolution bands | — | Inv 1 (targeted suites), 3, 7, 8; DDL test creates every table in a temp DB |
| 3 | **Extraction:** scan, structure, parse, type-resolve, x-file-bind, c-tier, data-flow, comment-extract | appendix: fixtures | Inv 2, 4, 5, 6, 14 |
| 4 | **Recognizers + cross-repo-link,** in roster order, with §11.3/§11.4 fixes | appendix: cross-repo, .NET | Per-recognizer fixture tests; **two-repo fixture**: `useMutation(CREATE_X)` in repo A → GraphQLOperation → Resolver in repo B, all edges persisted; socket event links across repos |
| 5 | **Persist + analysis:** persist, dep-graph, flatten, flow-detect, cluster, embed, summaries; worktree mode; build stamping; drift audit; `build-index` CLI | — | End-to-end build on a temp two-repo project; Inv 1 (generic), 10, 12, 13; deterministic communities across two builds |
| 6 | **MCP server:** registry, 38 tools, project injection, indexWarning, renderers, filters, cypher guard and templates | appendix: tool table | Every tool tested against the built fixture index; stdio lists all tools; Inv 9, 11 |
| 7 | **Spec docs + sample code** | appendix: fixtures | Genre, symlink, staleness, and fingerprint tests |
| 8 | **.NET analyzer** (`tools/argus-dotnet-analyzer`, .NET 8) | appendix: .NET | `dotnet test` green; importer vocabulary test |
| 9 | **Skills:** argus + 4 sub-agents + `project_conventions/`, bootstrap-argus, defensive-review, retrospective, adversary-review, open-pr/resolve-pr-reviews/gh-pr, product, backlog, eval, reflect, jira-ticket-generation (optional, org-specific), superpowers (vendored) | skills/*.md | Each spec's acceptance checks; the self-contained skill-scripts test; session-store copies identical |
| 10 | **Installer / uninstaller / hooks** | installer.md | Planner unit tests; dry-run uninstall against a temp home; setup on a temp home links skills and prints MCP commands |
| 11 | **Docs:** README, CLAUDE.md (architecture + gotchas), docs/manual-install.md, docs/argus/README.md, reference/{mcp-tools,confidence,output-formats}.md, `_can-i-do-that.md` | all | A grep of the docs finds no stale tool names, flags, or counts |

Start with milestone 1.
