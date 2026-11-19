# Review notes: `docs/argus/rebuild-prompt.md`

Independent review of the rebuild master prompt against the code at `main` (`d56a1ac7`). Skill prose was out of scope and was not reviewed. Line numbers like `prompt:88` refer to `docs/argus/rebuild-prompt.md`. All other references are repo-relative `file:line`.

Method: I read the source directly with Read, Grep and Bash, not argus MCP queries, because the review needed exact literal text (TOOL_DEFS schemas, EDGE_PAIRS, phase `deps:` arrays). I also loaded `schema.ts` under tsx to count the tables and edge pairs.

---

## Errors

These are claims in the prompt that the code contradicts.

1. **Edge storage model (prompt:88).** The prompt says edges are "explicit `(type, fromTable, toTable)` triples in `EDGE_PAIRS` (~240)".
   - `EDGE_PAIRS` holds **`[from, to]` pairs only**, with no type (`src/mcp-server/storage/schema.ts:259`).
   - There are **403** pairs, all unique. I counted them by loading the module.
   - Every edge lives in **one** rel table group named `CodeRelation`, and `type` is a STRING property on it (`schema.ts:435-449`, DDL at `schema.ts:471-476`).
   - Consequence: Cypher has to write `-[r:CodeRelation]->` and filter `WHERE r.type = 'X'`. The `cypher_query` description warns about exactly this (`src/mcp-server/index.ts:307`).
   - A rebuilder following the prompt would model one rel table per edge type. Every tool query and every cypher template would then be wrong.

2. **Missing node tables (prompt:77-86).** `AngularComponent` and `AngularService` are real tables (`schema.ts:174-175`). They appear in C-tier pairs, `COMMENT_OWNER_TABLES`, `EMBED_TABLES` and `FlowStep` targets. The real total is **67** tables. The prompt also files `FoundryAgent` and `FoundryAgentInvocation` under "Agent", but the schema groups them with the Angular stack-specific tables. That does no harm, but the count should be stated.

3. **Edge-type inventory (prompt:89-91).** Several listed types have no producer anywhere in the active pipeline. Grep finds them only in consumers: reachability, output formats, summaries, tools and cypher templates.
   - `RENDERS`: no producer or consumer anywhere in `src/mcp-server` outside dead code.
   - `TRIGGERS`, `HANDLES_SOCKET_EVENT`, `SUBSCRIBED_BY`: read by `flows/reachability.ts:9-17`, `pipeline/phases/cluster.ts:13` and `storage/argus-store.ts:377`, but no recognizer, extractor or phase emits them.
   - `PRODUCES_SHAPE`, `CONSUMES_SHAPE`, `READS_FIELD`: never emitted. The D-tier extractors return `edges: []` (`extractor/data-flow/field-access-extractor.ts:51`, `payload-shape-builder.ts:65`). `binding-extractor.ts:52-55` emits only `CONTAINS`.
     - So `impact_of_change` shape walks and the `shape-*` / `impact-of-shape-field` / `who-emits-shape-fields` templates run over nothing.
   - The prompt also leaves out types that **are** emitted:
     - `DEFINES` (CDK, hosted services, cross-repo lambda/SFN ARNs)
     - `STEP` (step functions)
     - `CAUGHT_BY` and `RETHROWS` (C-tier error paths)
     - `EXTENDS`, `IMPLEMENTS`, `LOADS_PROMPT`, `RETURNS`, `STARTS`
     - the Roslyn import types `INHERITS`, `RESOLVES_TO`, `INJECTS`, `HANDLES_REQUEST`, `DISPATCHES_TO`, `MANAGES`, `NAVIGATES_TO` (`recognizers/roslyn-import-recognizer.ts:102-113`)
   - Either list only the emitted types, or mark the reserved ones as "declared, not produced".

4. **Type resolution stack (prompt:30).** The prompt says "`ts-morph` + `pyright` for type resolution; `libcst` (Python) via a helper script". This is false for the active pipeline.
   - `type-resolve` uses only the heuristic `TieredResolver` (`pipeline/phases/type-resolve.ts:1-25`, `type-resolver/tiered-resolver.ts:37-49`). It has three tiers: annotation 0.98, constructor 0.92 and assignment chain 0.80, run to fixpoint with max 10 iterations.
   - No active code imports `ts-morph-resolver.ts`, `pyright-resolver.ts`, `roslyn-resolver.ts` or `roslyn-downloader.ts`.
   - Nothing imports `libcst` at all. `requirements.txt` lists it only because `setup-deps.js` installs it.
   - `scripts/python-ast-extractor.py` is used only by the dead v1 `src/mcp-server/indexer/python-parser.ts`.
   - `ts-morph` and `js-yaml` are used only by dead code (`src/mcp-server/indexer/**`) and the unwired resolver.

5. **Pipeline order (prompt:99-100).** The linear arrow chain is not the DAG. The real declared deps are in the phase table below. Two points matter:
   - `cross-repo-link` declares only `x-file-bind` (`pipeline/phases/cross-repo-link.ts:23`), yet it consumes `state.boundaryEntries`, which `recognize` fills (`pipeline/phases/recognize.ts:90-91`).
   - `dep-graph` depends on `x-file-bind`, not on `persist` (`pipeline/phases/dep-graph.ts:15`).

6. **Invariant 11, "never auto-refreshes" (prompt:184).** This overstates the behavior. The server never *rebuilds*, but `ProjectStoreRegistry.resolve` reopens a project's read-only store on the next tool call when `meta.json` `indexedAt` has changed and `treesAheadOfIndex` is false (`project-store-registry.ts:193-225`). `rebuild_index` also evicts the cached store (`index.ts:703-710`).
   - So an out-of-process `build-index` is picked up without a restart.
   - Only a newly *added project* needs a restart, because `.argus.config` is read once at startup (`index.ts:598`).

7. **Output formats (prompt:135).** There is a fourth format, `ascii`, an indented tree (`output-formats/ascii.ts`).
   - `get_flow_diagram` defaults to ascii (`index.ts:131`).
   - `trace_flow` defaults to structured.
   - `describe_flow` defaults to narrative (`index.ts:263`).

8. **The `project` param (prompt:124).** The param is not declared in `TOOL_DEFS`. `registerTool` injects it at registration (`index.ts:611-620`) into **every** tool except `list_projects`, and that includes the sample-code, spec-doc and comment tools. A test against `TOOL_DEFS` will not see `project`.

9. **Spec-doc genres and weights (prompt:143-145).**
   - There is an 8th value, `unknown`, returned on no signal or an unbreakable tie (`spec-docs/classify.ts:177,192`). It is accepted by the `genres` enums (`index.ts:328,345`).
   - "content weighted equal to path" is wrong as stated. The real weights (`classify.ts:20-27`) are:
     - path 2, filename 3
     - content-family 3 (capped at 6 per genre)
     - checkboxes 5 (when there are 5 or more)
     - frontmatter `name`+`description` 6, generated-banner 6
     - overrides 100
   - The full rules are in "Spec-docs classifier" below.

10. **meta.json fields (prompt:106).** The list is incomplete, and `fileHashes` has the wrong shape. The terminal publish (`pipeline/build-index.ts:506-547`) writes:
    - `schemaVersion` (`'2.1.0'`, `storage/meta-state.ts:154`)
    - `indexedAt`, `repoChoices`, `lastFullBuild{timestamp,durationMs}`
    - `phaseStats`, `cacheStats`
    - `fileHashes` as `{repoKey: {relPosixPath: blobSha}}`
    - `confidenceDistribution`, `resolutionDistribution`, `localBehindRemote`
    - **`indexedHeads`**, which the docs manifest reads for `docsAheadOfIndex` (`index.ts:651-654`)
    - `indexMode`, `indexedRefs`, `indexedTreePaths`, `checkoutPaths`
    - `treesAheadOfIndex`, `lastBuildFailed{at,phases}`, `buildInProgress`
    - plus `lastDriftAudit`, which `drift_audit` writes.

11. **Recognizer list (prompt:114).** "SQS/SNS" is not a recognizer.
    - `SQSTopic` and `SNSTopic` nodes are minted only by the CDK recognizer, with synthetic `SQSTopic:L<n>` ids (`schema.ts:120-121`).
    - The C# lambda recognizer emits `sqs-publish` / `sns-publish` boundary entries (`recognizers/csharp-lambda-recognizer.ts:92,101`).
    - There are 23 recognizers in a fixed order (`recognize.ts:28-50`); see Omission 7.

12. **Milestone 4 acceptance, "client GraphQL op → server resolver" (prompt:194).** No such direct edge exists.
    - The GraphQL boundary *publishes* the SDL `GraphQLOperation` (`recognizers/graphql-recognizer.ts:66-69`).
    - It *consumes* from both the client composable call site (`useMutation(SCREAMING_CONST)`, camelCased, line 188) and the Resolver (line 146).
    - Cross-repo-link emits consumer→producer `HANDLED_BY`:
      - `Composable → GraphQLOperation` survives.
      - `Resolver → GraphQLOperation` is **not** in `EDGE_PAIRS`, so persist silently drops it.
    - The op→resolver hop is the in-recognizer `GraphQLOperation -HANDLED_BY-> Resolver` name match (`graphql-recognizer.ts:133-143`).
    - Matching also never checks `producer.repoKey !== consumer.repoKey` (`cross-repo/boundary-registry.ts:89-104`), so same-repo matches are emitted too.
    - Rewrite the acceptance criterion as: "Composable(useMutation) → GraphQLOperation → Resolver path exists".

13. **The `scripts/` list (prompt:50).** It omits two files that `npm run build` and `package.json` need:
    - `copy-assets.js`. `build` is `tsc && node scripts/copy-assets.js` (`package.json`). It copies `src/**/*.py`, meaning `embed/python-embedder.py`, into `dist/`. Without it the embed phase fails at runtime from `dist/`.
    - `eval-triggers.mjs` (`npm run eval:triggers`).
    - It also does not mark two dead files: `verify-deps.js`, which imports `kuzu` and is not a dependency, and `python-ast-extractor.py`.

14. **Node version (prompt:27).** `src/mcp-server/index.ts:599,718` uses `import.meta.dirname`, which needs **Node ≥ 20.11**, not just 20.

15. **Install targets (prompt:63).** The written shape is right (`scripts/lib/targets.js:191-198`). But `src/mcp-server/project-config.ts:12-16` still types `installTargets` as the legacy `{claudeHome, pathFormat, wslHome}` and passes it through verbatim (`:89-91`). A rebuilder should type it once, as the JS shape.

---

## Omissions (ranked by impact)

Each item gives the exact text to add to the prompt.

### 1. Single rel table, COPY-by-pair, and the silent-drop mechanism (highest impact)

Add under GRAPH MODEL:

> All edges live in ONE LadybugDB `REL TABLE GROUP CodeRelation` declared `FROM <A> TO <B>` for every `[A,B]` in `EDGE_PAIRS` (pairs, not typed triples). Edge properties: `type STRING, confidence DOUBLE, reason STRING, resolution STRING, candidateCount INT64, sourceFile STRING, sourceLine INT64, evidence STRING, metadata STRING`. Persist groups edges by (fromTable,toTable) and bulk-loads each group with `COPY CodeRelation_<From>_<To> FROM '<csv>' (HEADER=true)`; if that sub-table doesn't exist it falls back to per-row `MATCH…CREATE` whose failures are swallowed — THIS is why an undeclared pair drops edges silently. An edge endpoint with no `nodeTableMap` entry defaults to table `File`. Node tables are created from `B_TIER_NODE_TABLES` via `CREATE NODE TABLE … PRIMARY KEY(id)`; schema type `INT`→`INT64`, `BOOL`→`BOOLEAN`. Nodes are also bulk-loaded via CSV `COPY`, which is why the DB directory (`ladybug`, `ladybug.wal`, `ladybug.lock`) is deleted before every build (duplicate PKs otherwise).

Source: `storage/csv-loader.ts:118-215`, `pipeline/phases/persist.ts:77-96`, `schema.ts:451-482`, `pipeline/build-index.ts:398-401`.

### 2. Phase scheduling semantics and the hidden ordering invariant

Add under PIPELINE:

> `PhaseDAGRunner` is CONCURRENT: when a phase finishes, every dependent whose deps are now satisfied is scheduled immediately (`Promise.resolve().then(run)`), in topo-sort/registration order. A phase that throws is recorded in `failed`, all its transitive dependents are `skipped`, and the build still completes and publishes meta with `lastBuildFailed:{at,phases}` + `treesAheadOfIndex:true`. Each recognizer is individually try/caught inside `recognize` (one bad recognizer never fails the phase). DECLARE COMPLETE DEPS: the current code under-declares (`cross-repo-link` reads `recognize`'s boundary entries but depends only on `x-file-bind`; `persist` does not list `c-tier-extract`/`data-flow-build`/`recognize`/`structure`) and is correct only because those phase bodies contain no `await`, so microtask order equals registration order. In the rebuild, give `cross-repo-link` deps `['recognize']` and `persist` deps on every node/edge producer.

### 3. Index-health contract (`indexWarning`) and the build stamping protocol

Add a section:

> `runBuildIndex` (under `withBuildLock`): (1) in remote mode, stamp `treesAheadOfIndex:true` on the existing meta under `withMetaLock` before advancing worktrees; (2) stamp `buildInProgress:true` (every mode) right before deleting the DB; (3) open a WRITABLE store, run the DAG; (4) CLOSE the writable store BEFORE (5) the terminal meta publish under `withMetaLock` (readers treat a changed `indexedAt` as "reopen now" — publishing before close races a failed reopen). Any throw closes the store and transitions `buildInProgress` → false with `lastBuildFailed`. Every tool response (except `index_status`, `list_projects`, `rebuild_index`) gets `indexWarning` when the generation captured at `resolve()` was unhealthy (`lastBuildFailed` > `buildInProgress` > `treesAheadOfIndex`, distinct wording each), when `indexedAt` changed mid-request, or when a build started mid-request (sticky-or). Object results get a top-level `indexWarning` field; string results get a `[warning]\n\n` prefix; arrays/null are wrapped as `{indexWarning, result}`. Thrown handler errors become `{content:[{type:'text',text:message}], isError:true}`. Successful results are `JSON.stringify(result, null, 2)` (strings passed through).

Source: `index.ts:406-577`, `pipeline/build-index.ts:251-610`.

### 4. Dead code a rebuild must NOT reproduce

Add:

> Do not rebuild: `src/mcp-server/indexer/**` (v1 IndexManager, ts-morph parser, python-parser — nothing outside that dir imports it), `src/shared/types.ts` (v1 NodeType/EdgeType + hardcoded `KNOWN_REPOS` tcn list), `src/mcp-server/type-resolver/{ts-morph,pyright,roslyn}-resolver.ts` + `roslyn-downloader.ts` + `exported-type-map.ts` (unwired; only `tiered-resolver.ts` is used, and only for its types-from-ts-morph-resolver import), `scripts/verify-deps.js` (imports `kuzu`), `scripts/python-ast-extractor.py`, `install.sh` (a stub that tells you to run `npm run setup`), `tests/mcp-server/indexer/**`, `tests/fixtures/ts-morph/**`. Consequently `ts-morph`, `pyright`, `js-yaml` and `libcst` are not needed; keep `libcst` out of requirements.txt unless a real consumer is added.

### 5. Exact dependency manifest, build and run files

Add under STACK:

> package.json deps: `@ladybugdb/core ^0.16.1`, `@modelcontextprotocol/sdk ^1.27.1`, `@vue/compiler-sfc ^3.5`, `graphology ^0.25.4`, `graphology-communities-louvain ^2.0.2`, `graphql ^16`, `tree-sitter ^0.21.1`, `tree-sitter-typescript ^0.21.2`, `tree-sitter-c-sharp ^0.21.3`, `tree-sitter-python ^0.21.0`, `typescript ^5.9`, and **`zod`** (imported by `src/mcp-server/index.ts:3` but currently only present transitively via the MCP SDK — declare it). Dev: `tsx`, `vitest ^4`, `@types/node`. Hand-write `src/types/ladybug.d.ts` (the package ships no types). Scripts: `build: tsc && node scripts/copy-assets.js` (copies `src/**/*.py` → `dist/`), `test: vitest run`, `build-index: node --max-old-space-size=4096 --import tsx/esm src/cli/build-index.ts` (runs from TS source, not dist), `setup`, `fix-links`, `uninstall`, `project`, `eval:triggers`. `.mcp.json` registers `argus` → `node dist/mcp-server/index.js` with `ARGUS_CONFIG=.argus.config`. MCP server identity `{name:'argus-panoptes', version:'2.0.0'}`. vitest excludes `.claude/**`, `.argus/**` (indexed worktrees contain foreign test suites), `tests/brainstorm-server/**` (separate package with its own runner). `.gitattributes`: `* text=auto eol=lf`, `*.cmd`/`*.bat` CRLF, images/pdf/zip binary.

### 6. Cross-repo boundary model

Add a section with the content of "Cross-repo link kinds" below.

> Boundary entries `{boundaryType, key, direction: publish|consume, nodeId, repoKey, metadata}` are emitted by recognizers; `cross-repo-link` matches every consumer against every publisher of the same `boundaryType` (O(n·m), no different-repo check) and emits a consumer→producer edge whose type comes from `mapBoundaryToEdgeType` and whose confidence/reason come from `matchBoundary`.

### 7. Recognizer roster and order

Add:

> `RECOGNIZERS` (order matters for same-ID merges): GraphQL, Vue, Pinia, Route (Express-style), CDK, StepFunction, Neo4j, CSharpLambda, BedrockAgent, PromptLoader, AgentTool, AgentToolUsage, SocketIO, Angular, FastAPIRoute (reuses `Route` table), Foundry, AspNetCore, DotnetHostedService, EfCore, RoslynImport, PydanticAiAgent, CliCommand, McpTool. Each implements `recognize(ctx): {nodes, edges, nodeTableMap, boundaryEntries}`; ctx = `{repoKey, captures, resolvedTypes, fileMap, argusDir, logger}`. Promotion = set `nodeTableMap.set(existingId, NewTable)` keeping the id.

### 8. ID conventions and node column sets

Add under GRAPH MODEL:

> IDs: symbols `<repoKey>::<relPosixPath>::<name>`; File `<repoKey>::<relPath>`; CallSite `<repo>::<relPath>::callsite::<target>:L<line>` (claim-callsites must derive the exact same id); EntryPoint `entry::<nodeId>`; NamedFlow `flow::<nodeId>`; FlowStep `step::<startId>::<order>`; Community `community::<n>`; synthetic line-keyed ids for CDK/socket/invocation nodes (`cdk:<call>:L<n>`, `socket-emit:L<n>`, `invoke:L<n>`). Column sets: COMMON_SYMBOL_COLS = id, name, repo, filePath, startLine, endLine, language, signature, docSummary, metadata (JSON string); STRUCTURAL_COLS = id, name, repo, filePath, metadata; COMMENT_COLS = id, name (synthetic `comment:L<n>`), repo, filePath, startLine, endLine, language, text, kind (`docblock|leading|inline|header|commented-code`), markers, ownerId, metadata. Embeddable tables carry an `embedding` STRING column. `Comment` is excluded from default `search_index` scans (`ArgusStore.SEARCH_EXCLUDED_TABLES`). Keep `COMMENT_NON_OWNER_PROMOTIONS` as an asserted list: every promotion target must be in exactly one of it or `COMMENT_OWNER_TABLES`.

### 9. Scan rules (shared with drift audit)

Add:

> `scan` walks each repo (no symlink-following needed — `readdirSync` withFileTypes) skipping `SKIP_DIRS` = node_modules, dist, .git, __pycache__, coverage, .next, .nuxt, venv, .venv, .aws-sam, generated, .argus, .claude, .superpowers, worktrees, .angular, .nx, .turbo, .pytest_cache, .mypy_cache, .ruff_cache. Buckets: `.asl.json`→aslJson; `.ts/.js/.tsx/.jsx` (excluding `*.test.*`, `*.spec.*`, `*.d.ts`)→ts; `.vue`; `.py`; `.graphql/.gql`; `.cs`. Every file goes into `all` with its git blob hash from the index (empty if untracked). drift-audit MUST import the same SKIP_DIRS or it reports phantom drift.

Source: `pipeline/phases/scan.ts:9-74`.

### 10. Flow detection, summaries and clustering specifics

Add:

> Entry-point scoring by table: VueComponent .95, Composable .7, Route .85, AspNetRoute .85, GraphQLOperation mutation .9 / other .8, LambdaHandler .9, StepFunction/BedrockAgent/PydanticAiAgent/BackgroundWorker/GrpcService/CliCommand/McpTool .85, SignalRHub .8, SocketIOEvent .75; everything else is not an entry. For each, emit EntryPoint(`entry::id`) –ENTRY_POINT_OF→ node, EntryPoint –STARTS_FLOW→ NamedFlow(`flow::id`), NamedFlow –STEP_IN_FLOW→ FlowStep. Reachability: BFS, MAX_DEPTH 8, MAX_STEPS 50, step 0 = the entry itself (stepKind `ENTRY`); follow only FLOW_EDGE_TYPES {TRIGGERS, HANDLED_BY, CALLS, INVOKES, PUBLISHES_TO, EMITS_SOCKET_EVENT, HANDLES_SOCKET_EVENT, INVOKES_AGENT, USES_TOOL, STEP, STARTS, SUBSCRIBED_BY, DEFINES}; use the flattened projection ONLY when a node has outgoing edges but none flow-bearing. Flow name = `<entryName> <kind suffix>` + ` (stub)` if 0 steps. Summaries are deterministic structured cards (no LLM — ignore the Ollama/Anthropic comments in `.argus.config.example`, they are stale): BFS depth 4 over raw edges, repos touched, async boundary types, written to `EntryPoint.summaryCard`. Cluster: Louvain over edges of type {CALLS, INVOKES, TRIGGERS, HANDLED_BY, IMPORTS, USES_TOOL} plus projected edges, with SORTED inputs and a fixed-seed mulberry32 RNG so communities are reproducible; `MEMBER_OF` confidence 0.7. `dep-graph` writes `<argusDir>/dep-graph/symbols.json` (file → importing files) from IMPORTS edges.

### 11. Embedding details

Add:

> EMBED_TABLES = Function, Method, Class, BedrockAgent, NamedFlow, PromptReference, AngularComponent, AngularService, FoundryAgent, PydanticAiAgent. `buildText` = name + signature + first 4 lines of (HAS_DOCBLOCK text, else docSummary), with per-table variants. Python `embed/python-embedder.py` reads JSONL `{id,text}` on stdin, writes JSONL `{id,embedding[384]}`; `--probe` prints model/dim. Cache: `<argusDir>/embed-cache/cache.jsonl`, key = sha1(text)[0:16]; it survives rebuilds (only `ladybug*` is deleted). Python is located via `findPython()` (honours `VIRTUAL_ENV`).

### 12. `cypher_query` safety rules and the template set

Add:

> `cypher_query`: mask strings/comments first, then require start with MATCH/OPTIONAL MATCH/WITH/UNWIND/RETURN; reject CREATE, MERGE, SET, DELETE, DETACH, REMOVE, DROP, ALTER, COPY, LOAD, INSTALL, ATTACH, EXPORT, IMPORT, RENAME, BEGIN, COMMIT, ROLLBACK, any CALL, `;`, UNION, unbounded var-length (`[*]`, `[*..]`, `[*N..]`); LIMIT ≤ 1000 literal, append ` LIMIT 1000` if absent and return the executed `cypher`. `cypher_template` (empty name lists them) — 20 templates: callers-of, callees-of, flow-from-entry-point, impact-of-shape-field, who-emits-shape-fields, who-handles-socket-event, who-handles-resolver, agent-prompt-sources, agent-tool-callers, shape-producers, shape-consumers, lambda-by-arn, step-function-states, community-members, flow-touching-repo, named-flow-by-name, unresolved-references, entry-points-by-type, shapes-crossing-language, orphan-publishers.

### 13. Resolution bands (makes invariant 7 implementable)

Add to invariant 7:

> Bands: PRECISE ≥ 0.85, HEURISTIC ≥ 0.5, else AMBIGUOUS; canonical defaults 0.95 / 0.75 / 0.4. `candidateCount` defaults to 1.

### 14. Config validation and project resolution

Add:

> Project names must match `/^[A-Za-z0-9._-]+$/` and not be `.`/`..` (validated on write by `project.js` AND on load — bad entries are skipped with a warning). Resolution order: explicit `project` arg → `ARGUS_PROJECT` → sole project → error listing names. Legacy flat config migrates to project name `tcn`. Env vars: `ARGUS_CONFIG`, `ARGUS_PROJECT`, `ARGUS_DOTNET_GRAPH_PATH`, `ARGUS_DOTNET_GRAPH_ALLOW_STALE`, `ARGUS_REPO_ROOT`+`ARGUS_UNINSTALL_TEST` (test seam), `VIRTUAL_ENV`.

### 15. Full CLI surface

Add:

> `build-index` flags: `--project <n>`, `--local`, `--sample`, `--docs`, `--both`/`--all`, `--index`/`--tcn` (legacy structural). `--docs` with `--project` and no index flag = docs-only. No target flag on a TTY → numbered interactive picker; non-TTY defaults to index. Bare words `sample|tcn|both|all|docs` (not after `--project`) print a "did you mean `--x`? (npm needs `--`)" hint.

### 16. Other artifacts written outside the index directory

Add to CONFIG MODEL:

> Per-project dir also holds `dep-graph/symbols.json`, `embed-cache/cache.jsonl`, `dotnet-graph/<repoKey>.json` (Roslyn input), `build.lock/` and `meta.lock/` lock dirs. The sample-code manifest is NOT per-project: `<argus repo>/cache/sample-code-manifest.json`.

### 17. Fixture policy

Add to the milestones. The pipeline and e2e tests create temp repos inline (`mkdtemp` + `writeFileSync`) rather than checking in whole repos. The per-language fixtures under `tests/fixtures/` back the extractor and recognizer tests. See "Fixtures to recreate".

### 18. .NET analyzer

Milestone 4 needs the analyzer spec (see "Dotnet analyzer spec" below), or an explicit "out of scope; importer only".

---

## MCP tool table

There are 38 tools, from `TOOL_DEFS` in `src/mcp-server/index.ts:60-386`. Every tool except `list_projects` also accepts an injected optional `project: string`.

`CF` means the shared confidence-filter params `minConfidence?: number` (0–1) and `excludeAmbiguous?: boolean`, spread in from `index.ts:54-57`. `?` marks an optional param.

| # | name | params | purpose |
|---|---|---|---|
| 1 | trace_flow | `entry_point: string`, `includeCTier?: bool`, `maxDepth?: number` (def 3), `format?: structured\|narrative\|mermaid\|ascii`, `direction?: downstream\|upstream\|both`, CF | Trace from an entry point (role-aware bare-name resolution) |
| 2 | trace_upstream | `target: string`, CF | All callers / entry points leading to a node |
| 3 | find_entry_points | `mutation_name: string` | UI components that trigger a GraphQL mutation |
| 4 | find_resolver | `operation_name: string` | Resolver for a GraphQL operation |
| 5 | find_lambda_handler | `function_name: string` | Lambda handler by name |
| 6 | find_step_function | `name: string` | Step Function definition and states |
| 7 | search_index | `query: string`, `types?: string[]`, `repos?: string[]`, `limit?: number` (def 25, max 100), `offset?: number` | Case-insensitive name substring search, ranked exact → prefix → substring; returns `{results,totalMatches,limit,offset}`; Comment excluded unless named |
| 8 | find_shared_usage | `function_name: string` | Consumers of a shared library function |
| 9 | find_callees | `symbol: string`, `includeCTier?: bool`, CF | Functions called by a symbol |
| 10 | get_flow_diagram | `entry_point: string`, `format?: ascii\|mermaid` (def ascii) | Diagram for an end-to-end flow |
| 11 | rebuild_index | `repo?: string` (hint only), `local?: bool` | Whole-project rebuild; evicts the cached store |
| 12 | index_status | (none) | Freshness, divergence, resolution distribution |
| 13 | drift_audit | (none) | Re-hash files vs `meta.fileHashes`; writes `lastDriftAudit` |
| 14 | list_sample_code | `repoKey?`, `language?`, `pathPattern?` (glob), `keyword?`, `maxResults?: int ≤100` (def 20) | Browse sample-code manifest |
| 15 | grep_sample_code | `pattern: string` (regex), `repoKey?`, `language?`, `contextLines?: int 0–10` (def 2), `maxResults?: int ≤100` (def 20) | ripgrep, or Node fallback, over sample code |
| 16 | list_agents | `repo?: string` | Bedrock / Foundry / pydantic_ai agents |
| 17 | find_agent_tool_usage | `toolName: string` | Agents using a tool |
| 18 | prompt_compositions | `agent: string` | Prompt sources for a Bedrock agent |
| 19 | find_event_emitters | `eventName: string` | Socket.IO emitters |
| 20 | find_event_handlers | `eventName: string` | Socket.IO handlers |
| 21 | find_controller_actions | `query: string`, `method?: string` | ASP.NET routes and controllers |
| 22 | find_background_services | `query?: string`, `kind?: background\|worker\|signalr\|hub\|grpc` | .NET hosted services, hubs, gRPC |
| 23 | find_dbcontexts | `query?: string` | EF Core DbContexts |
| 24 | impact_of_change | `symbolId?`, `shapeId?`, `field?`, CF | Consumers impacted (CONSUMES_SHAPE / READS_FIELD / IMPORTS / PRODUCES_SHAPE walk) |
| 25 | find_mutation_handler | `mutation: string` | Server resolver for a mutation or query, with file:line |
| 26 | get_context | `symbolId: string`, `includeCode?: none\|signature\|body\|full-file`, CF | 360° view: callers, callees, shapes, flow, community |
| 27 | describe_flow | `flowId: string`, `format?: structured\|narrative\|mermaid\|ascii` (def narrative), CF | PM narrative of a NamedFlow |
| 28 | list_flows | `repo?`, `entryKind?`, `limit?: int ≤2000` (def 200) | List NamedFlows |
| 29 | system_touchpoints | `topic: string` | Communities, repos and services touching a topic |
| 30 | semantic_search | `query: string`, `k?: int ≤100` (def 10), `types?: string[]` | Cosine similarity over embeddings |
| 31 | cypher_template | `name: string` (empty = list), `params?: record` | Named, parameterized Cypher |
| 32 | cypher_query | `query: string` | Guarded read-only raw Cypher, row cap 1000 |
| 33 | list_projects | `divergence?: bool` | Projects, health and `onboardingPending`; registered without `project` |
| 34 | list_spec_docs | `repos?`, `genres?: (8-value enum)[]`, `keyword?`, `since?`, `until?`, `ticketKey?`, `minConfidence?: low\|medium\|high`, `includeHeadings?`, `limit?: int ≤500` (def 50), `offset?` | Spec-doc bibliography, metadata only |
| 35 | grep_spec_docs | `pattern: string` (JS regex), `repos?`, `genres?`, `caseSensitive?`, `contextLines?: 0–10` (def 1), `maxResults?: ≤100` (def 20) | Regex over doc bodies |
| 36 | get_spec_doc | `repo?`+`path?` or `id?` (`<repo>::<path>`), `section?`, `maxBytes?: ≤200000` (def 60000), `history?: none\|summary\|full`, `historyLimit?: ≤100` (def 20) | Read one doc with git history |
| 37 | get_comments | `node_id: string`, `scope?: docblock\|all`, `include_children?: bool`, `kinds?: (docblock\|leading\|inline\|header\|commented-code)[]`, `limit?` (def 50, cap 200) | Comments on a node; schema-guarded |
| 38 | grep_comments | `pattern: string`, `regex?: bool`, `repos?`, `kinds?`, `limit?` (def 25, cap 100) | Search comment prose; schema-guarded |

The genre enum is `intent`, `execution`, `adjudication`, `reference`, `instruction`, `boilerplate`, `derived`, `unknown`.

Registration notes:
- `TOOL_DEFS[0..9]` are wired by index (`index.ts:663-672`). Everything else is wired by `findByName`.
- `get_comments` and `grep_comments` are wrapped in `withSchemaGuard`, which returns `{ok:false, reason:'index-schema-outdated'}` on a pre-2.1.0 index.
- The sample-code manifest is cached per project in-process and never refreshed until restart (`index.ts:716-730`).

---

## Phase dependency table

Taken from the `deps:` array in each `src/mcp-server/pipeline/phases/<name>.ts`. All phases are registered in this order in `pipeline/build-index.ts:421-436`.

| phase | declared deps | actually reads output of | writes |
|---|---|---|---|
| scan | — | config | `state.fileMaps` |
| structure | scan | scan | Repo/Folder/File nodes, CONTAINS |
| parse | scan | scan | `state.captures` (tree-sitter / graphql / vue / asl) |
| type-resolve | parse | parse | `state.resolvedTypes` (TieredResolver only) |
| x-file-bind | type-resolve | parse, structure | symbol nodes, CONTAINS / IMPORTS / CALLS |
| c-tier-extract | x-file-bind | x-file-bind | CallSite / Branch / TryBlock / ErrorHandler / ThrowSite / AwaitPoint, HAS_CALLSITE / INVOKES / CAUGHT_BY / RETHROWS |
| data-flow-build | x-file-bind | x-file-bind | D-tier nodes (CONTAINS edges only) |
| recognize | x-file-bind | x-file-bind | domain nodes, promotions, edges, `state.boundaryEntries` |
| comment-extract | x-file-bind, **recognize** | recognize (final tables) | Comment nodes, HAS_DOCBLOCK / HAS_COMMENT |
| cross-repo-link | x-file-bind | **recognize** (boundaryEntries); dep not declared | cross-boundary edges |
| dep-graph | x-file-bind | x-file-bind (IMPORTS) | `<argusDir>/dep-graph/symbols.json` |
| persist | x-file-bind, cross-repo-link, comment-extract | everything above, including undeclared c-tier-extract, data-flow-build and structure | DB nodes and edges (CSV COPY) |
| flow-detect | persist | state + projected edges | EntryPoint / NamedFlow / FlowStep, ENTRY_POINT_OF / STARTS_FLOW / STEP_IN_FLOW |
| cluster | persist | state + projected edges | Community, MEMBER_OF |
| embed | persist | DB (HAS_DOCBLOCK text) | `embedding` columns, embed cache |
| summaries | flow-detect | DB EntryPoints + state edges | `EntryPoint.summaryCard` |

The topological order that results is: scan, structure, parse, type-resolve, x-file-bind, c-tier-extract, data-flow-build, recognize, cross-repo-link, dep-graph, comment-extract, persist, flow-detect, cluster, embed, summaries.

---

## Cross-repo link kinds

Sources: `cross-repo/boundary-types.ts`, `cross-repo/boundary-registry.ts:3-70`, `pipeline/phases/cross-repo-link.ts:4-18`, and the recognizers that emit entries.

| boundaryType | publish side (key) | consume side (key) | match rule, confidence | edge type | live? |
|---|---|---|---|---|---|
| graphql | GraphQLOperation from SDL (op name) | Resolver (op name); client `useQuery/useMutation/useSubscription(CONST)` call, `CONST` camelCased, as a Composable node | exact key, 0.95 | HANDLED_BY | Yes. Composable→Op lands; Resolver→Op is dropped (pair not in EDGE_PAIRS) |
| rest-route | Express Route, FastAPI route (`METHOD:path` / `METHOD:handler`) | none (no HTTP-client consumer is emitted) | exact 0.95; `:param` / `${}` normalized 0.80 | HANDLED_BY | No consumers, so HTTP calls are never linked |
| lambda-arn-reference | C# Lambda (ARN text); CDK lambda (`relPath:L<n>`) | Step Function state `Resource` | ARN suffix after the last `:`, 0.90 | DEFINES | Yes (C# Lambda ↔ ASL) |
| step-function-arn | CDK SFN (`relPath:L<n>`) | none | exact 0.90 | DEFINES | No |
| sqs-env-var / sns-env-var | CDK queue/topic (`relPath:L<n>`) | none | exact 0.90 | PUBLISHES_TO | No |
| sqs-publish / sns-publish | C# SDK publish (text) | none; unknown type | default exact 0.70 | CALLS | No |
| socket-event | emit (`socket-emit:L<n>`) | handler (`socket-handler:L<n>`) | exact 0.95 | EMITS_SOCKET_EVENT | Effectively never: keys are line-numbered, so they don't match |
| socket-room | (mapped, never emitted) | — | 0.90 | EMITS_SOCKET_EVENT | No |
| socket-middleware | — | middleware (`socket-middleware:L<n>`) | default | CALLS | No |
| agent-runtime | BedrockAgent (agentName) | invocation (`invoke:<path>:L<n>`) | exact 0.85 | CALLS | Effectively never: keys differ in shape |
| shared-import | (mapped, never emitted) | — | 0.95 | IMPORTS | No (shared-library links come from x-file-bind IMPORTS) |
| mcp-tool | McpTool (toolName) | none | default | CALLS | No |
| foundry-agent | — | Foundry invocation | default | CALLS | No |
| vue-handler | — | Vue handler (text) | default | CALLS | No |

In practice, only GraphQL (client composable → operation) and Lambda ARN ↔ Step Function produce cross-repo edges today. The prompt should either say so, or require real HTTP and Socket.IO keying, for example by event name, as an explicit improvement.

---

## Dotnet analyzer spec

Location: `tools/argus-dotnet-analyzer/`. It contains `ArgusAnalyzer.sln`, `src/ArgusAnalyzer/`, `tests/ArgusAnalyzer.Tests/` and `tests/Fixtures/AnalyzerFixture/`. User docs are in `docs/argus/dotnet-analyzer.md`.

- **Project:** .NET 8 console exe (`net8.0`, nullable, implicit usings). Packages:
  - `Microsoft.Build.Locator 1.7.8`
  - `Microsoft.CodeAnalysis.CSharp.Workspaces 4.11.0`
  - `Microsoft.CodeAnalysis.Workspaces.MSBuild 4.11.0`
  - `System.CommandLine 2.0.0-beta4.22272.1`
  - `System.Text.Json 8.0.5`
- **CLI:** `dotnet run --project src/ArgusAnalyzer -- --solution <sln> --out <json> [--repo-root <dir>] [--commit <sha>]`.
  - `--repo-root` defaults to the solution directory.
  - A missing `--commit` gives a loud stderr warning, not a failure.
- **Flow:**
  - `MSBuildLocator.RegisterDefaults()`, then `MSBuildWorkspace.OpenSolutionAsync`.
  - Projects are deduped by file path, then filtered: names containing `Test` are excluded.
  - For each project, get the compilation. Compile errors only produce a warning (best effort).
  - For each document, skip `*.g.cs`, `*.g.i.cs` and anything under `/obj/` or `/bin/`. Then get the semantic model with `ignoreAccessibility: true` and run five analyzers: `CallGraphAnalyzer`, `EndpointAnalyzer`, `MediatRAnalyzer`, `DiAnalyzer`, `EfCoreAnalyzer`.
  - Per-document and per-project exceptions are logged and skipped.
- **Output** (`argus.code_graph/v1`, camelCase JSON, indented):
  - Top level: `{schema, generatedAt, commitSha, solution, projects:[{name,path}], nodes, edges, stats:{nodeCount,edgeCount}}`.
  - Node: `{id, kind, name, qualifiedName?, project, file, lineStart, lineEnd?, tags[], httpMethod?, route?, auth?, returnType?, isAsync, parameters?:[{name,type}], requestType?, serviceLifetime?, metadata?}`.
    - `id` is the Roslyn `GetDocumentationCommentId()`, falling back to the fully-qualified display string.
    - DI factories use `<svcId>#factory:<file>:<line>`.
  - Edge: `{from, to, kind, confidence = 0.95, file?, line?}`.
  - Dedup: first writer wins for nodes; edges dedupe on `(from,to,kind)`.
- **Node kinds:** `method`, `class`, `http_endpoint`, `mediatr_handler`, `di_registration`, `dbset`, `entity`.
- **Edge kinds:** `calls`, `implements`, `inherits`, `resolves_to`, `injects`, `handled_by`, `handles_request`, `dispatches_to`, `manages`, `navigates_to`.
- **Importer** (`src/mcp-server/recognizers/roslyn-import-recognizer.ts`):
  - Graph path, in priority order: `ARGUS_DOTNET_GRAPH_PATH`, then `<argusDir>/dotnet-graph/<repoKey>.json`, then legacy `<repo>/.argus/dotnet-graph.json`.
  - A schema other than `argus.code_graph/v1` is rejected.
  - If `commitSha` ≠ the indexed HEAD, the import is skipped unless `ARGUS_DOTNET_GRAPH_ALLOW_STALE=1`.
  - Node → table mapping (`ROSLYN_NODE_TABLES`): method→Method, class / entity / mediatr_handler / di_registration→Class, http_endpoint→AspNetRoute, dbset→DbContext.
  - Edge → type mapping (`ROSLYN_EDGE_TYPES`): uppercased.
  - Binding: an exact `def.method` or `def.class` capture on `lineStart` with the same name, else the nearest enclosing class. Unbound nodes become synthetic `roslyn::<id>` rows.
  - Metadata is merged with `mergeNode`: `source: 'roslyn-import'`, `roslynIds/Kinds/QualifiedNames/Tags`, and normalized `httpMethod` / `route`.
  - A vocabulary test holds the C# side and the TS maps together.
- **Tests:** `dotnet test tests/ArgusAnalyzer.Tests`, covering `AnalyzerTests` and `JsonExporterTests`. They run against the `AnalyzerFixture` project, which has stub ASP.NET Core, DI and MediatR types plus `OrdersController`, `DiServices`, `MediatRSample` and `Program`, so no real framework packages are needed.
- **Gitignore:** bin/obj for all three projects (`.gitignore`).

---

## Fixtures to recreate

The integration tests (`tests/integration/*-e2e.test.ts`) build temp repos inline with `mkdtemp`, so they need no fixture repo. For the checked-in fixtures, recreate only those that active tests use:

- `tests/fixtures/typescript/simple-class.ts`, `vue/simple-component.vue`, `python/simple-agent.py`, `graphql/schema.graphql`, `schema.graphql`, `AgreementDetail.vue`, `routes.ts`: extractor, recognizer, flow-detect and tool tests.
- `tests/fixtures/c-tier/{async-patterns,branches,error-paths}.{ts,py}`, `call-chains.ts`: C-tier extractors, find-enclosing-def, claim-callsites.
- `tests/fixtures/csharp/`: `aspnet-controller`, `async-service`, `attributes`, `aws-sdk-usage`, `env-vars`, `grpc-service`, `hosted-services`, `lambda-handler`, `minimal-api`, `simple-class` (`.cs`). Used by the C# extractor and the .NET recognizers.
- `tests/fixtures/asl/agreement-flow.asl.json`: ASL extractor and step-function recognizer.
- `tests/fixtures/agents/` (`agent-tool.py`, `invoke-agent.py`, `prompt-compose.py`, `prompt-loader.py`, `pyproject.toml`) and `tests/fixtures/tcn-agents/{budget-extractor,sample-agent,sample-tools}`: agent recognizers. `pyproject.toml` drives AgentPackage.
- `tests/fixtures/socket/` (`namespaces.ts`, `rooms.ts`, `ts-server-emitter.ts`, `ts-server-handler.ts`, `py-emitter.py`): Socket.IO recognizer and tools.
- `tests/fixtures/data-flow/{payload-shapes,variable-bindings}.ts`: D-tier extractors.
- `tests/fixtures/type-resolver/{ts-project,py-project}`: tiered resolver and callsite tests. The ts-morph and pyright tests here are dead.
- `tests/fixtures/sample-code-tree/{repo-a (README, src, dist, image.png), repo-b (README, notebooks)}`: sample-code manifest and grep. `dist` and the binary check exclusions.
- `tests/fixtures/spec-docs-tree/{repo-dated (CONTRIBUTING.md, argus-traces/, docs/), repo-empty (src only), repo-reference (CONTRIBUTING, README, docs/ generated), repo-topical (plans/)}`: genre classification, the derived override, boilerplate and dating.
- `tests/fixtures/pr-reviews/*`: PR-skill script tests (`bitbucket-comments.json`, `github-review-threads.json`, `sonar-issues.json`, three `coderabbit-*.md`). These belong to skills, so include them only if the skill scripts are rebuilt.

Do not recreate these:
- `tests/fixtures/ts-morph/**`, `cdk-stack.py`, `state-machine.asl.json` and `agent-service.ts`: only the dead `tests/mcp-server/indexer/**` uses them.
- `axios-callback.ts`, `lambda-callback.ts`, `lib-consumer.ts`, `lib-export.ts`, `mutation.ts`, `signature-resolver.ts`, `sns-publisher.ts`, `socket-emitter.ts`, `sqs-publisher.ts`: no test references them.

Missing fixture worth adding: a real **two-repo** fixture for Milestone 4. One repo would hold an SDL with Mutation `createX` and a resolver. The other would call `useMutation(CREATE_X)`. The assertions would check that the Composable→Op and Op→Resolver edges are persisted, and that the Resolver→Op edge is either declared in EDGE_PAIRS or deliberately not emitted.

---

## Structural suggestions

1. **Split the prompt into a short master prompt and appendices**, for example `rebuild/appendix-schema.md`, `appendix-tools.md` and `appendix-phases.md`. Have the master prompt require reading the relevant appendix at each milestone. The tool table, schema column sets and phase table above are too large to inline but are load-bearing.
2. **Make the source of truth executable.** Have Milestone 2 generate the DDL from `B_TIER_NODE_TABLES` / `EDGE_PAIRS`, and add a test that walks every recognizer's emitted `(fromTable,toTable)` pairs on the fixtures and asserts each one is declared. That single test catches invariant 1 and the Resolver→GraphQLOperation drop (Error 12) generically, instead of relying on three hand-maintained coverage suites.
3. **Add a "Known gaps, do not faithfully reproduce" section.** It should list:
   - no producers for TRIGGERS / RENDERS / *_SHAPE / READS_FIELD / HANDLES_SOCKET_EVENT / SUBSCRIBED_BY
   - dead cross-repo boundary kinds
   - under-declared phase deps
   - `GUARDS` does not exist
   - the stale LLM-summary comments in `.argus.config.example`
   - `zod` used but not declared

   This lets the rebuilder choose to fix them rather than copy them.
4. **Fold invariants into the milestones that own them.** Each milestone's acceptance should name the invariant numbers it must prove, for example M3 → 2, 4, 5 (already done) and M5 → 8, 9, 12. Add the missing ones:
   - M5: index-health stamping (Omission 3).
   - M6: `indexWarning` shapes, registry reopen on `indexedAt`.
5. **Pin the milestone order to the real dependency order of shared code.** `scan.ts` `SKIP_DIRS` is shared by drift-audit, and `claim-callsites` must mint the same ids as `callsite-extractor`. State such cross-module contracts beside the module that owns them.
6. **Replace counts that rot** ("~240", "38", "16") with "derive from code; a test asserts the count" (`tests/mcp-server/server.test.ts` already asserts the TOOL_DEFS shape).
7. **Separate the installer/uninstaller into its own prompt**, as is being done for skills. That is roughly half of `scripts/` and is independent of the graph engine. M9 as written cannot carry enough detail on the WSL junction and owned-MCP rules, which CLAUDE.md documents at length. Point to CLAUDE.md's Gotchas sections as a required read.
8. **Add a "files that must exist" checklist at the top level:**
   - `.mcp.json`, `.argus.config.example`, `.gitattributes`, `.gitignore` (including `project_conventions/*` with its two exceptions)
   - `requirements.txt`, `hooks/{session-start,hooks.json,run-hook.cmd,hooks-cursor.json}`
   - `LICENSE-superpowers`, `_can-i-do-that.md`
   - `docs/manual-install.md`, `docs/argus/{README.md,dotnet-analyzer.md,reference/{mcp-tools,confidence,output-formats}.md}`
   - `project_conventions/{README.md,_template.md}`
   - `tests/brainstorm-server` (a separate package)
