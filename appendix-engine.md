# Appendix: engine reference

Tables and specs that the master prompt points to. All of it was checked against the code at commit d56a1ac7. Every tool except `list_projects` also accepts an injected optional `project` param.

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

