# Skill: adversary-review

Cross-model review loop. The coding agent (the CODER, any client) has a DIFFERENT installed agent CLI (the ADVERSARY: codex, copilot, or claude) review the committed branch in a detached read-only worktree, verifies each finding, fixes what is real, rebuts what is not, and iterates to convergence (round cap 3, deadlock escalation to the user). Judgment lives in `SKILL.md`; all mechanics (CLI flags, sandboxing, sessions, worktrees, validation) live in a bundled Node helper.

Design doc (historical, do not edit): `docs/superpowers/specs/2026-07-29-adversary-review-design.md`.

## Files to create (every path + purpose)

All paths relative to the repo root. The skill must be self-contained: nothing under `skills/adversary-review/` may reach into the repo root (no `ARGUS_ROOT`, `/../..`, `<argus-repo>`, `ARGUS_CONFIG`, `.argus.config` strings in any `.md` file of the skill).

| Path | Purpose |
|---|---|
| `skills/adversary-review/SKILL.md` | Orchestration instructions for the coder agent (frontmatter + procedure below). |
| `skills/adversary-review/findings-schema.json` | JSON Schema the adversary CLIs enforce at generation time (verbatim below). |
| `skills/adversary-review/reviewer-prompt.md` | Two prompt templates (round 1 and re-review) separated by `<!-- REREVIEW -->`, with `{{PLACEHOLDER}}` tokens (verbatim below). |
| `skills/adversary-review/scripts/adversary-review.js` | Helper CLI: `detect`, `start`, `rereview`, `finalize`, `cleanup`. ESM, `// @ts-check`, Node built-ins only. Exports its `run*` functions for tests. |
| `skills/adversary-review/scripts/lib/adversaries.js` | Adapter registry: one descriptor per CLI (`codex`, `copilot`, `claude`) with `probe`/`buildArgv`/`parseResult`; JSONL parsers; detection. |
| `skills/adversary-review/scripts/lib/review-argus.js` | `ARGUS_READ_TOOLS` exact allowlist, `resolveArgusProject`, `resolveBaseRef`. |
| `skills/adversary-review/scripts/lib/review-prompt.js` | `renderRound1`, `renderRereview` — fill the template. |
| `skills/adversary-review/scripts/lib/review-schema.js` | Hand-rolled response validator, `detectDeadlocks`, `isConverged`. |
| `skills/adversary-review/scripts/lib/review-session.js` | Session store instance (`dir: 'adversary-review'`, `prefix: 'ar-'`, `label: 'adversary-review'`). |
| `skills/adversary-review/scripts/lib/review-worktree.js` | Detached worktree lifecycle + `headSha`, `isTreeClean`. |
| `skills/adversary-review/scripts/lib/session-store.js` | Generic session-store factory. MUST be byte-identical to `skills/resolve-pr-reviews/scripts/lib/session-store.js` (a test guards drift). |
| `tests/scripts/adversaries.test.ts`, `adversary-review.test.ts`, `review-argus.test.ts`, `review-prompt.test.ts`, `review-schema.test.ts`, `review-session.test.ts`, `review-worktree.test.ts`, `session-store.test.ts`, `skills-self-contained.test.ts` | Vitest suites (see Tests). |

## Frontmatter (verbatim)

```yaml
---
name: adversary-review
description: Use when the user asks a DIFFERENT agent CLI to review an open branch ("have codex review this", "adversary review", "get copilot to check this branch") — runs an iterative cross-model review loop until both agents agree the branch is ready.
---
```

## Purpose; fires when / does not

- **Fires when** the user asks a different agent CLI to review an open branch: "have codex review this", "adversary review", "get copilot to check this branch".
- **Does not**: review uncommitted work (the helper refuses a dirty tree); drive adversary CLIs directly (always via the helper); support WSL-to-Windows adversaries (v1 is same-environment CLIs only). If no other CLI is installed, it offers the `superpowers:requesting-code-review` subagent path instead.
- Roles: you are the CODER; the other CLI is the ADVERSARY. Peers — evidence settles disagreements, the user is the tie-breaker.
- Announce at start: "I'm using the adversary-review skill."

## Procedure (numbered, faithful; every gate, cap, stop condition)

Helper location (resolve from the skill's own folder):

```bash
SKILL_DIR="$(dirname "$(realpath <path-to-this-SKILL.md>)")"
HELPER="$SKILL_DIR/scripts/adversary-review.js"
```

Every subcommand prints one JSON object. Do not drive adversary CLIs directly; the helper owns flags, sandboxing, sessions, and worktrees.

### Round 0 — setup

1. **Clean tree.** `git status --porcelain` must be empty. If dirty, offer to commit; do NOT proceed dirty (the helper also refuses).
2. **Tests.** Run the project's suite. Capture a short summary. Failures do not block a review but must be disclosed in the packet.
3. **Adversary.** If the user named one, use it. Otherwise `node "$HELPER" detect --exclude <your-own-client-id>` (own id is `claude`, `codex`, or `copilot`). One result: use it. Multiple: ask the user. None: say so and offer `superpowers:requesting-code-review`.
4. **Packet.** Write a markdown file (session temp dir or scratchpad) with exactly the headings `## Description` (3-8 sentences), `## Requirements` (plan/spec section or user ask; link `docs/` paths), `## Test Results` (suite name, pass/fail counts, notable failures).
5. **Review subject vs. context.** The embedded diff should carry only the review subject (every embedded line is re-sent on every turn). Process docs that are *context for* the change (spec, plan, prior review records) or generated files (lockfiles) may be excluded with repeatable `--exclude <pathspec>` and cited in the packet's Requirements instead. **Never exclude the review subject itself** (reviewing a spec branch means the spec IS the diff — exclude nothing). The flag is optional. Exclusions are disclosed to the adversary automatically.
6. **Start round 1:**
   ```bash
   node "$HELPER" start --adversary <id> --repo "$(pwd)" --packet <packet.md> \
     [--base <ref>] [--exclude <pathspec>]... [--round-cap 3] [--timeout-min 10] \
     [--budget <usd>]
   ```
   `--budget` is a POSITIVE dollar amount (exit 2 on garbage). claude gets `--max-budget-usd` directly; copilot gets `--max-ai-credits` at 100 credits/USD, skipped with a warning on CLIs predating the flag; codex has no native cap (`--timeout-min` bounds it). Note the returned `sessionId`. Base resolution: explicit `--base` → `origin/HEAD` → `main`/`master`.

### Each round — verify, fix, respond

7. For EVERY open finding (new `response.findings` plus prior findings the adversary `reasserted`) apply `superpowers:receiving-code-review` discipline — verify against the actual code first. Never fix an unconfirmed finding; never write performative agreement. The responses file must cover exactly that open set (helper refuses a mismatch before spending the adversary call).
   - Confirmed critical/important → fix now, commit normally (no squashing).
   - Confirmed minor → fix or log for the record; state which and why.
   - Refuted → do not fix; technical rebuttal citing file:line.
   - Judgment call (design/scope, not a factual defect) → park for the user; blocks the loop only if critical.
8. Write the responses file (template below) and run `node "$HELPER" rereview <session-id> --responses <responses.json>`.
9. **Per-round digest** to the user after every round: round number; findings by severity; per finding — verified/refuted/parked and what you did; the adversary's `priorFindingResults` statuses; `converged` / `deadlockIds` / `roundCapReached`; and the round's `usage` (tokens, cost where reported) when present.

### Termination — exactly one of

10. **Converged** (`converged: true`): present the final digest plus any parked judgment calls (convergence never silently swallows them).
11. **Round cap** (`roundCapReached: true`, default 3): stop; present open items with both sides' positions; user decides. (The helper hard-refuses a `rereview` once `round >= roundCap`.)
12. **Deadlock** (`deadlockIds` non-empty): stop arguing THOSE findings immediately; give the user both positions; relay the ruling in the next responses file as `user-ruled` (or end the loop if the user says done).

### Review record (always, even on abort)

13. Write `docs/superpowers/reviews/YYYY-MM-DD-<branch>-adversary-review.md` (sanitize `/` in branch to `-`): adversary + version, base ref + merge-base (from `start` output `baseRef`/`mergeBase`), per round the reviewed SHA and each finding's trajectory (found → verified/refuted → fixed/parked/ruled), final verdict, per-round `usage`, and any `--exclude` pathspecs with rationale. Commit it. This finalization commit is exempt from re-review by design — it must touch only `docs/superpowers/reviews/`; its SHA goes in the final message to the user, not in the record.
14. **On convergence:** `node "$HELPER" finalize <session-id>` (audits the exemption, then cleans up; any non-record change re-enters the loop).
15. **On abort, round-cap stop, or error:** `node "$HELPER" cleanup <session-id>`. One of finalize/cleanup is unconditional — never leave a session behind.

## Hard rules (critical ones VERBATIM)

From SKILL.md `## Hard rules`, verbatim:

```
- The adversary reviews committed SHAs only; the helper enforces this.
- The adversary is read-only and network-less; its checkout's repo instructions
  (CLAUDE.md/AGENTS.md) load normally — deliberate, user-ruled 2026-07-29.
- v1 is same-environment CLIs only (no WSL→Windows adversaries).
- If the helper reports the adversary crashed/timed out: offer retry, switch
  adversary, or fall back to superpowers:requesting-code-review. Never proceed
  as if reviewed.
```

Also verbatim from the procedure: "Never fix a finding you have not confirmed; never write performative agreement." / "**Never exclude the review subject itself**" / "One of the two is unconditional — never leave a session behind."

## Scripts

### `scripts/adversary-review.js` — CLI

Usage line on bad subcommand: `usage: adversary-review.js detect|start|rereview|finalize|cleanup ...` (exit 2). Output: `JSON.stringify(result, null, 2)` on stdout. Uncaught errors propagate (non-zero exit). Constants: `DEFAULT_ROUND_CAP = 3`, `DEFAULT_TIMEOUT_MIN = 10`; `SCHEMA_FILE = <skill>/findings-schema.json`. Flag parsing: `flag(name)` = value after first `--name`; `flagAll(name)` collects every `--name <v>` (used for `--exclude`).

| Subcommand | Flags | Result JSON |
|---|---|---|
| `detect` | `[--exclude <id>]` | `[{ id, version }]` for every installed adapter except the excluded one |
| `start` | required `--adversary`, `--packet` (missing → stderr `missing required flag --<name>`, exit 2); `--repo` (default cwd), `--base`, `--exclude`×N, `--round-cap`, `--timeout-min`, `--budget`, `--argus-config <path>` | `{ sessionId, round: 1, baseRef, mergeBase, response, raw, usage, converged, deadlockIds: [], roundCapReached }` |
| `rereview <sid>` | `--responses <file>` | `{ sessionId, round, response, raw, usage, converged, deadlockIds, roundCapReached }` |
| `finalize <sid>` | — | `{ finalized, approvedSha, recordCommit, auditedCommits, auditedFiles }` |
| `cleanup [<sid>]` | — | with id: `{ cleaned: sid }`; without (or first arg starts `--`): `{ sessions: [ids] }` |

Numeric flags go through `parsePositiveFlag(name, raw)`: `Number(raw)` must be finite and `> 0`, else throws `invalid --<name> value "<raw>" — expected a positive number`; `main` prints it and exits 2. Applies to `--budget` (only if present), `--round-cap`, `--timeout-min`.

**`parsePacket(file)`**: split on `/^## /m`, heading = first line trimmed, body = rest trimmed (multi-line sections preserved in full). Returns `{ description: sections['Description'] || '(none provided)', requirements: sections['Requirements'] || '(none provided)', testResults: sections['Test Results'] || '(not run)' }`.

**`buildDiff(io, repoRoot, range, excludes)`**: if excludes, pathspecs = `['--', '.', ...excludes.map(e => ':(exclude)'+e)]`. `git diff <range> <pathspecs>` is the embedded diff. `excludedFiles` = names in `git diff --name-only <range>` (unfiltered) minus names in the filtered `--name-only` list. Git calls use `maxBuffer` 64 MB.

**Excluded note** (empty string when nothing excluded), lines joined by `\n`:
```
## Changed on this branch but excluded from the embedded diff

The coder excluded these changed files from the diff below as context
(process docs, generated files). They are readable IN FULL in your checkout
at these exact paths — consult them as needed. If you believe any of them
needs line-level review instead, raise that as a finding:

- <file>
```

**Argus note**: without argus: `Argus MCP tools are NOT available for this review. Rely on the diff and the checkout.` With argus:
```
Argus MCP read tools are available (project "<project>") for cross-repo
architectural context: impact_of_change, find_shared_usage, trace_flow, etc.
The DIFF is ground truth — the argus index reflects the indexed checkout, not
this branch. Check index_status freshness before leaning on it. Never call
rebuild_index (it is not in your allowlist).
```

**`argusMcpFor(argus, argusConfigPath)`**: if both set → `{ entryPoint: join(dirname(argusConfigPath), 'dist', 'mcp-server', 'index.js'), configPath: argusConfigPath }` (derived from the passed config, NOT the skill dir); else `null`.

**`runStart(opts, deps)`** (`deps = { io: {execFileSync}, spawn, sessionIo?, adversary? }`):
1. `isTreeClean` else throw `working tree is dirty — commit (or stash) before starting a review`.
2. `sha = headSha`; `{baseRef, mergeBase} = resolveBaseRef(repo, opts.baseRef)`; parse packet.
3. Argus is OFF by default: only if `--argus-config` is given and exists, `argus = resolveArgusProject(repoRoot, JSON.parse(config))` (may be null if repo not in any project).
4. `createSession({ adversary: id, adversarySessionId: null, repoRoot, baseRef, mergeBase, roundCap, timeoutMs: timeoutMin*60000, argus, argusConfigPath: argus ? path : null, excludes, budgetUsd: budget ?? null, round: 0, rounds: [], worktree: null })`.
5. worktree dir = `<sessionDir>/worktree`. Persist `state.worktree` BEFORE creating it (so a hard kill leaves cleanable state), then `createReviewWorktree` at `sha`.
6. Diff `mergeBase..sha`; render round-1 prompt with `round: 1`.
7. Spec: `{ prompt, promptFile: <sessionDir>/round-1-prompt.md, schemaFile, argusTools: argus?.tools ?? null, argusMcp: argusMcpFor(...), budgetUsd, resumeId: null, outputFile: <sessionDir>/round-1-out.json }`.
8. `invokeAdversary(... expectSha: sha, priorOpenIds: [], round: 1, usedIds: [])`.
9. Save `adversarySessionId`, `round: 1`, `rounds: [{ sha, response, usage }]`, `approvedSha = converged ? sha : null`. `roundCapReached = 1 >= roundCap`.
10. On ANY error: `removeWorktree` + `deleteSession`, rethrow.

**`invokeAdversary`**: `attempt(extraNote)` — append `\n\n<note>` to prompt if given; ALWAYS write the rendered prompt to `spec.promptFile` (audit trail + copilot handover); `adversary.buildArgv(spec)` → `spawn(cmd, args, { cwd: worktree, timeoutMs, input: stdin })`; non-zero non-null status → throw `adversary "<id>" exited with status <n>`; `parseResult(stdout, spec)`; `JSON.parse(resultText)`. If unparseable, retry ONCE with note `Your previous output was not parseable JSON. Respond with ONLY the JSON object.`; still unparseable → throw `adversary "<id>" returned unparseable output twice`. Then `validateReviewResponse`; on failure throw `adversary response invalid:\n- <errors joined by \n- >`. Real spawn: `spawnSync` with `encoding utf8`, `timeout`, 64 MB `maxBuffer`, `input` only when defined; `r.error` rethrown.

**`openFindingIds(rounds)`**: iterate rounds oldest first; add every `findings[].id`; for priors, ids with status `resolved`/`accepted-refutation`/`user-ruled` are removed UNLESS the same response also reasserts that id (reassertion dominates). Return sorted array.

**`runRereview(sid, {responsesFile}, deps)`**:
1. Load state. If `state.round >= state.roundCap` throw `round cap <cap> reached (completed round <n>) — escalate open findings to the user, then finalize or cleanup` (hard guard).
2. Dirty tree → throw `working tree is dirty — commit fixes before re-review`.
3. Parse `{ fixSummary, responses }`. `priorOpenIds = openFindingIds(state.rounds)`. Reconcile BEFORE spending the CLI call: `missing` = open ids without a response; `notOpen` = response ids not open. If either non-empty throw `responses file does not match the open finding set — missing: [..|none], not open: [..|none]; cover exactly: [..]`.
4. `sha = headSha`; `prevSha` = last round's sha; `advanceWorktree(sha)`. `round = state.round + 1`. Diff `prevSha..sha` (with stored excludes). Render re-review prompt.
5. Spec as in start but `round-<n>-prompt.md` / `round-<n>-out.json`, `budgetUsd: state.budgetUsd`, `resumeId: state.adversarySessionId`.
6. `usedIds` = all finding ids from all prior rounds. Invoke with `expectSha: sha, priorOpenIds, round, usedIds`.
7. Push `{ sha, response, coderResponses: responses, usage }`; set `approvedSha` as above; save. Return with `deadlockIds = detectDeadlocks(state.rounds)` and `roundCapReached = round >= roundCap`.

**`runFinalize(sid, deps)`** — audit then cleanup:
- No `approvedSha` → `session has not converged — nothing to finalize (use cleanup to abort)`.
- HEAD == approvedSha → `no record commit found after the approved SHA — commit the review record, then finalize`.
- Dirty → `working tree is dirty — commit the review record before finalizing`.
- `git merge-base --is-ancestor approvedSha HEAD` fails → `finalization refused — HEAD <7> does not descend from the approved SHA <7> (finalize on the reviewed branch)`.
- Per commit in `git rev-list --reverse approvedSha..HEAD` (per-commit, NOT endpoint diff, so a revert cannot hide a code commit): if `git rev-parse --verify --quiet <c>^2` succeeds it is a merge → throw `finalization exemption violated — post-approval merge commit <7> (merge diffs evade the per-commit audit; record commits must not be merges) — re-review required`. Files = `git diff-tree --no-commit-id --name-only -r <c>`; any not starting with `docs/superpowers/reviews/` → `finalization exemption violated — commit <7> touches paths outside docs/superpowers/reviews/: <list> — these must be re-reviewed`.
- No audited files at all (only empty commits) → `finalization exemption violated — no review-record file was committed after the approved SHA`.
- Then `runCleanup` and return the result.

**`runCleanup(sid, deps)`**: no id → `{ sessions: listSessions() }`. Else try load state (ignore failure), `removeWorktree` if `worktree` and `repoRoot` known, `deleteSession`, return `{ cleaned }`.

### Session state

Location: `$TMPDIR/adversary-review/ar-<8 hex>/` (`os.tmpdir()`). Files: `state.json`, `worktree/` (detached git worktree), `round-<n>-prompt.md`, `round-<n>-out.json` (codex last message).
`state.json` schema: `{ adversary, adversarySessionId, repoRoot, baseRef, mergeBase, roundCap, timeoutMs, argus: {project, tools}|null, argusConfigPath, excludes: string[], budgetUsd: number|null, round, rounds: [{ sha, response, usage, coderResponses? }], worktree, approvedSha: string|null, sessionId }`.

### `lib/session-store.js` (shared factory; identical in both skills)

`makeSessionIo()` → `{ tmpdir, mkdirSync, readFileSync, writeFileSync, rmSync, readdirSync, existsSync, randomUUID }`. `createSessionStore({ dir, prefix, label, init = {} })` returns `{ sessionDir, createSession, loadSession, saveSession, deleteSession, listSessions }`, each taking an optional trailing `io`.
- Id = `prefix + randomUUID().slice(0, 8)`; validated by `^<escaped prefix>[0-9a-f]{8}$` on every path-building entry (sessionDir/load/save/delete) → else `invalid <label> session id "<id>"` (path-traversal guard).
- `createSession(data)`: mkdir -p, save `{ ...data, ...init, sessionId }` (init wins over data).
- `loadSession`: missing → `no <label> session "<id>" (looked for <file>)`; bad JSON → `<label> session "<id>" has corrupt state at <file>: <msg>`.
- `saveSession`: pretty JSON (2-space). `deleteSession`: `rmSync(recursive, force)`. `listSessions`: `[]` if root missing; else entries starting with prefix, sorted lexicographically.

### `lib/review-worktree.js`

All git via `io.execFileSync('git', ['-C', cwd, ...args], {encoding:'utf8'}).trim()`.
- `headSha` = `rev-parse HEAD`.
- `isTreeClean` = `status --porcelain --untracked-files=no` is empty (untracked files deliberately ignored).
- `createReviewWorktree({repoRoot, sha, dir})` = `worktree add --detach <dir> <sha>`.
- `advanceWorktree({dir, sha})` = `checkout --detach <sha>` in dir, then verify `rev-parse HEAD === sha` else throw `worktree advance failed: HEAD is <at>, expected <sha>` (stale checkout would let the adversary approve fixes it never saw).
- `removeWorktree` never throws: `worktree remove --force <dir>` then `worktree prune`, each try/catch.

### `lib/review-argus.js`

`ARGUS_READ_TOOLS` — exact allowlist (never `rebuild_index`, never `drift_audit` — both write state; drift_audit writes `meta.json`):
`list_projects, index_status, search_index, semantic_search, trace_flow, trace_upstream, find_callees, find_entry_points, find_resolver, find_mutation_handler, find_lambda_handler, find_step_function, find_event_emitters, find_event_handlers, find_shared_usage, find_agent_tool_usage, impact_of_change, get_context, describe_flow, get_flow_diagram, list_flows, list_agents, system_touchpoints, prompt_compositions, cypher_query, cypher_template, list_sample_code, grep_sample_code, list_spec_docs, grep_spec_docs, get_spec_doc, get_comments, grep_comments`.

`resolveArgusProject(repoRoot, config)`: for each `config.projects[name].repos` value, repo path = string value OR `value.path` (v2 `{path}` shape; both accepted); if `resolve(path) === resolve(repoRoot)` return `{ project: name, tools: ARGUS_READ_TOOLS }`; else `null`.

`resolveBaseRef(repoRoot, explicit, io)`: explicit → else `git symbolic-ref --short refs/remotes/origin/HEAD` → else first of `main`, `master` passing `rev-parse --verify --quiet`. None → throw `cannot resolve a base branch: pass --base <ref>`. `mergeBase = git merge-base HEAD <baseRef>`. Returns `{ baseRef, mergeBase }`.

### `lib/review-prompt.js`

Template = `<skill>/reviewer-prompt.md` split on `<!-- REREVIEW -->` into `round1`, `rereview`. `fill` replaces `/{{([A-Z_]+)}}/g` in ONE pass (values are not re-scanned, so literal `{{...}}` inside a diff survive); a placeholder missing from the map throws `unfilled placeholder {{X}} in reviewer prompt`.
- `renderRound1(vars)` map: `BASE_SHA, HEAD_SHA, BASE_REF, DESCRIPTION, REQUIREMENTS, TEST_RESULTS, DIFF, EXCLUDED_NOTE (default ''), ARGUS_NOTE, ROUND (default 1)`.
- `renderRereview(vars)` requires `vars.round` (else throw `renderRereview requires vars.round`). `RESPONSES` = `- **<id>** — <status>: <detail>` per response joined by `\n`, or `(no per-finding responses provided)` when empty. Map: `HEAD_SHA, ROUND, RESPONSES, FIX_SUMMARY, DIFF, EXCLUDED_NOTE`.

### `lib/review-schema.js` — helper-side validator

`validateReviewResponse(obj, { expectSha, priorOpenIds, round?, usedIds? })` → `{ ok, errors[] }`. Tolerates unknown extra keys (deliberate). Checks:
- not an object → `response is not an object`.
- verdict in `FINDINGS|APPROVED`; `reviewedSha === expectSha` (`reviewedSha "<x>" does not match expected "<y>"`); `findings` array.
- each finding: required `id, severity, file, line, claim, evidence, verify` (undefined or `''` → `finding <id> missing <k>`); severity in `critical|important|minor`; id matches `^R[0-9]+-[0-9]+$`; line positive integer; if `round` given, id must start `R<round>-`; id not in `usedIds` (`collides with a prior round's finding`); no duplicate ids in the response.
- `priorFindingResults` array; every `priorOpenIds` id needs an entry; each entry: status in `resolved|accepted-refutation|reasserted|user-ruled`; `reasserted` requires non-empty `evidence`; no duplicate entries per id; every entry id must be in `priorOpenIds` (`a finding cannot be raised and settled in the same response`).

`detectDeadlocks(rounds)`: `[]` if fewer than 3 rounds; else ids with status `reasserted` in BOTH of the last two rounds, sorted.

`isConverged(response)`: verdict `APPROVED` AND no prior `reasserted` AND no new `critical`/`important` finding (minor may ride along).

### `lib/adversaries.js` — adapter registry

Interface: `{ id, bin, probe(io) → version|null, buildArgv(spec, io?) → { cmd, args, stdin? }, parseResult(stdout, spec, io?) → { resultText, sessionId, usage? } }`. `InvokeSpec = { prompt, promptFile?, schemaFile, argusTools: string[]|null, argusMcp?: {entryPoint, configPath}|null, resumeId, budgetUsd?, outputFile? }`. `ADVERSARIES = [codex, copilot, claude]` (order matters for detect). `getAdversary(id)` throws `unknown adversary "<id>" (known: codex, copilot, claude)`. `makeAdversaryIo()` → `{ canExec, execFileSync, readFileSync }`; `canExec(bin)` is a side-effect-free PATH scan (`existsSync(join(dir, bin))`, separator `;` on win32 else `:`). `probe` = `canExec` then first line of `<bin> --version`, null on failure. `detectAdversaries(io, {exclude})` → `[{id, version}]` skipping `exclude` and un-probed.

Prompt delivery: prompts go via stdin wherever possible — Linux caps a single argv string at 128 KB (E2BIG) and branch diffs exceed it.

**codex** (verified codex-cli 0.146.0). Args in this exact order:
```
exec --sandbox read-only --skip-git-repo-check --json --output-schema <schemaFile>
[--output-last-message <outputFile>]
[-c mcp_servers.argus.enabled_tools=["t1","t2",...]  -c mcp_servers.argus.default_tools_approval_mode="approve"]   (only when argusTools)
[resume <resumeId>]
-
```
Parent `exec` options MUST precede the `resume` subcommand (codex rejects `--sandbox` after it). `-` = read prompt from stdin; `stdin: prompt`. The approval-mode `-c` is required because headless codex auto-cancels every MCP tool call otherwise. `parseResult`: `resultText` = contents of `outputFile` (null if unset or unreadable); `sessionId` = `parseJsonlForSessionId(stdout)`; `usage` = `parseJsonlForUsage(stdout)`.

**copilot** (verified Copilot CLI 1.0.75). No stdin path for `-p`. `COPILOT_ARGV_PROMPT_MAX = 100_000` measured in UTF-8 BYTES (`Buffer.byteLength`). Oversized + no `promptFile` → throw (message mentions `promptFile`). Oversized prompt arg: `Read the file <promptFile> and follow the review instructions in it exactly. It is your complete task.` plus `--add-dir <dirname(promptFile)>`. Args:
```
-p <promptArg> --output-format json --no-color [--add-dir <dir>] [--resume=<id>]
--deny-tool write --deny-tool shell --available-tools view,grep,glob[,argus-<tool>...]
[--max-ai-credits <round(budgetUsd*100)>]
```
Built-in file-read tool is `view` (not `read`); MCP tool names are `<server>-<tool>` (hyphen). `COPILOT_CREDITS_PER_USD = 100`. Budget flag is feature-detected by running `copilot --help` and checking for `--max-ai-credits`; if absent, write to stderr `warning: this copilot CLI predates --max-ai-credits — budget cap skipped, --timeout-min is the only cost guard for this round` and omit the flag. No `stdin`. `parseResult`: output is JSONL; `resultText` = `data.content` of the LAST `type: 'assistant.message'` event; `sessionId` via `parseJsonlForSessionId`; `usage` via `parseJsonlForUsage` (null when absent). Unparseable → all null.

**claude**. `--json-schema` takes schema INLINE (read `schemaFile` content). Args:
```
-p --output-format json --permission-mode dontAsk --setting-sources '' --strict-mcp-config
--json-schema <schema content> --max-budget-usd <budgetUsd ?? 10>
--disallowedTools Edit,Write,NotebookEdit,Bash,WebFetch,WebSearch,Agent
[--mcp-config {"mcpServers":{"argus":{"command":"node","args":[entryPoint],"env":{"ARGUS_CONFIG":configPath}}}}]   (only when argusTools AND argusMcp)
--allowedTools Read,Glob,Grep[,mcp__argus__<tool>...]
[--resume <resumeId>]
```
`stdin: prompt`. Rationale: `--setting-sources ''` drops ambient allow rules; `--strict-mcp-config` loads only `--mcp-config` servers (kept even without argus). `parseResult`: single JSON; `resultText = obj.result`, `sessionId = obj.session_id`, usage (when `usage.input_tokens` is a number) = `{ inputTokens, cachedInputTokens: cache_read_input_tokens ?? 0, outputTokens, totalTokens: in+out, costUsd: total_cost_usd }`. Unparseable → `{ resultText: null, sessionId: null }`.

**`parseJsonlForSessionId(stdout)`**: first line whose JSON contains (recursive search) a truthy `session_id`, `thread_id`, or `sessionId`; else null.
**`parseJsonlForUsage(stdout)`**: for each line, take recursive `total_token_usage` or (`type === 'turn.completed'` → `obj.usage`); if `input_tokens` is a number, set usage `{ inputTokens, cachedInputTokens: cached_input_tokens ?? 0, outputTokens: output_tokens ?? 0, totalTokens: total_tokens ?? in+out }`. LAST one wins (codex usage is cumulative per session). Null if none.

## Tests to write (list the behaviors the existing tests pin)

- **adversaries**: registry has codex/copilot/claude with full descriptor surface; `getAdversary` throws on unknown id; detect lists installed CLIs with versions and excludes the caller; codex argv exact (round 1 with argus allowlist, prompt on stdin, `-` sentinel); omits `mcp_servers` when `argusTools` null; parent flags before `resume <id> -`; codex parseResult reads outputFile via injected io, null when unreadable or not requested; `parseJsonlForUsage` takes last cumulative (`total_token_usage` and `turn.completed`), null when absent; `parseJsonlForSessionId` found/null; copilot argv exact incl. USD→credits, skips credits on old CLI, `--resume=<id>`, byte-length cap (multibyte prompt under `.length` but over bytes uses file pointer), oversized → file pointer + `--add-dir`, throws without promptFile, no stdin; copilot parseResult last assistant message + session id, nulls on garbage; claude argv exact with inlined schema/budget/denies/scoped allows, stdin prompt; strict isolation kept and no `--mcp-config` without argus (`--allowedTools` = `Read,Glob,Grep`); `--resume` appended; claude parse result/session/usage/cost.
- **adversary-review (CLI, real temp git repo + fake adversary)**: parsePacket preserves multi-line sections; openFindingIds — reassertion dominates a same-round resolved entry, resolved stays closed across rounds; runStart end-to-end persists state (`adversarySessionId`, baseRef, round 1, prompt contains `## Diff` and sha); no argus without `--argus-config` (`argus` null, `argusMcp` null); refuses dirty tree; rejects reviewedSha mismatch; runRereview advances worktree, resumes, detects convergence; reports `roundCapReached` at cap and refuses to run past it; diff exclusions excluded from diff but disclosed in note; no note when nothing excluded; usage surfaced on result and in state; responses-file mismatch refused before CLI call; parsePositiveFlag accepts ints/decimals, rejects garbage/negative/zero with flag name; budget threaded to every round spec; round-1 APPROVED returns converged and persists approvedSha; finalize: accepts record-only commit and cleans up, rejects changes outside record dir, rejects code commit hidden by later revert, rejects merge commits, rejects only-empty commits, refuses non-descendant HEAD, refuses unconverged session; `argusMcpFor` derives entry point from config path.
- **review-argus**: allowlist never contains `rebuild_index`/`drift_audit`; includes spec-doc tools; `resolveArgusProject` maps/returns null for both string and `{path}` shapes; `resolveBaseRef` uses explicit ref, falls back to `main` when no origin/HEAD.
- **review-prompt**: every placeholder substituted, none left; literal `{{...}}` inside values untouched; rereview renders responses and new sha; throws without round; placeholder note for empty responses.
- **review-schema**: accepts well-formed round 1; rejects sha mismatch, bad severity/missing fields, missing prior entry, same-response self-resolution, duplicate prior entries, reassert without evidence, malformed id / bad line, wrong round prefix, id colliding with prior round, duplicate ids; tolerates unknown keys; deadlock flagged only for two consecutive reassertions; isConverged true/false cases.
- **review-session / session-store**: round-trip create/load/save/list/delete; `init` merged and wins over data; label in unknown/corrupt errors; empty list when root missing; path-traversal id rejected (tmp root untouched); wrong-shape id rejected rather than "no session"; valid id round-trips.
- **review-worktree** (real git): create detached at pinned sha, advance with verification, remove; advance throws on nonexistent sha and on silent wrong landing; isTreeClean reflects tracked changes and ignores untracked; removeWorktree never throws.
- **skills-self-contained**: helper file exists; the two `session-store.js` copies are byte-identical; no kept skill `.md` contains the forbidden root-reaching strings.

## Output format (templates verbatim)

**Packet** (exact headings):
```markdown
## Description
<what was built, 3-8 sentences>
## Requirements
<the plan/spec section or user ask this branch implements; link docs/ paths>
## Test Results
<suite name, pass/fail counts, notable failures>
```

**Responses file:**
```json
{ "fixSummary": "<one line per fix commit: sha + subject>",
  "responses": [
    { "id": "R1-1", "status": "fixed",     "detail": "commit abc123: <what changed>" },
    { "id": "R1-2", "status": "refuted",   "detail": "<rebuttal with file:line evidence>" },
    { "id": "R1-3", "status": "user-ruled", "detail": "<the user's ruling, verbatim gist>" }
  ] }
```

**`findings-schema.json`** (verbatim):
```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "title": "AdversaryReviewResponse",
  "type": "object",
  "required": ["verdict", "findings", "reviewedSha", "priorFindingResults", "notes"],
  "additionalProperties": false,
  "properties": {
    "verdict": { "enum": ["FINDINGS", "APPROVED"] },
    "reviewedSha": { "type": "string", "pattern": "^[0-9a-f]{40}$" },
    "notes": { "type": ["string", "null"] },
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "severity", "file", "line", "claim", "evidence", "verify", "suggestion"],
        "additionalProperties": false,
        "properties": {
          "id": { "type": "string", "pattern": "^R[0-9]+-[0-9]+$" },
          "severity": { "enum": ["critical", "important", "minor"] },
          "file": { "type": "string" },
          "line": { "type": "integer", "minimum": 1 },
          "claim": { "type": "string", "minLength": 1 },
          "evidence": { "type": "string", "minLength": 1 },
          "verify": { "type": "string", "minLength": 1 },
          "suggestion": { "type": ["string", "null"] }
        }
      }
    },
    "priorFindingResults": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "status", "evidence"],
        "additionalProperties": false,
        "properties": {
          "id": { "type": "string" },
          "status": { "enum": ["resolved", "accepted-refutation", "reasserted", "user-ruled"] },
          "evidence": { "type": ["string", "null"] }
        }
      }
    }
  }
}
```

**`reviewer-prompt.md`** (verbatim):
````markdown
# Adversary Review — Reviewer Instructions

You are an ADVERSARIAL code reviewer. A different AI coding agent wrote the
branch under review; your job is to find real defects it missed. You two are
peers — neither outranks the other; disagreements are settled by evidence or
escalated to the human.

## Hard rules

- Do NOT load or follow any skill files you may have access to. These
  instructions are complete.
- You are READ-ONLY: never modify files, never run write commands.
- The DIFF below is ground truth for what changed. The checkout you are in is
  pinned at the exact commit under review ({{HEAD_SHA}}).
- Do not approve to be agreeable. APPROVED means you re-checked and found no
  critical or important defects. There is no partial credit for politeness.
- Every finding must be FALSIFIABLE: fill the `verify` field with a concrete
  way to confirm it (a command, an input, a code path to trace).
- Severity honestly: `critical` = bugs/security/data loss; `important` =
  architecture, missing requirements, error handling, test gaps; `minor` =
  style/polish. Not everything is critical.

{{ARGUS_NOTE}}

## What was implemented

{{DESCRIPTION}}

## Requirements

{{REQUIREMENTS}}

## Test results (run by the implementer at {{HEAD_SHA}})

{{TEST_RESULTS}}

{{EXCLUDED_NOTE}}

## Review range

Base: {{BASE_SHA}} ({{BASE_REF}})  Head: {{HEAD_SHA}}

## Diff

```diff
{{DIFF}}
```

## Output

Respond with ONLY a JSON object matching the provided schema:
`verdict` ("FINDINGS" or "APPROVED"), `findings[]` (id "R{{ROUND}}-<n>",
severity, file, line, claim, evidence, verify, optional suggestion),
`reviewedSha` (exactly "{{HEAD_SHA}}"), `priorFindingResults` ([] on round 1),
optional `notes` for what you checked and found sound.

<!-- REREVIEW -->
# Adversary Review — Re-review Round

The implementer responded to your findings. The checkout has been advanced to
the new commit {{HEAD_SHA}}; re-verify at THIS commit.

## Responses to your findings

{{RESPONSES}}

## Fix commits

{{FIX_SUMMARY}}

{{EXCLUDED_NOTE}}

## New diff since your last reviewed commit

```diff
{{DIFF}}
```

## Rules for this round

- Emit a `priorFindingResults` entry for EXACTLY the findings listed under
  "Responses to your findings" above — every one of those, and ONLY those.
  Findings settled in earlier rounds are closed; do not re-reference them
  (entries for closed or unknown ids are rejected). Statuses:
  `resolved` (fix verified at {{HEAD_SHA}}), `accepted-refutation` (their
  rebuttal is correct), `reasserted` (you still believe it — MUST include new
  `evidence`, not a restatement), or `user-ruled` (the human ruled; acknowledge).
- Verify claimed fixes by reading the code at this commit — a commit message is
  not proof.
- New findings get ids "R{{ROUND}}-<n>".
- `reviewedSha` must be exactly "{{HEAD_SHA}}".
- APPROVED only if nothing critical/important remains open.
- Your round-1 hard rules still bind: read-only, no skill loading, every new
  finding needs a falsifiable `verify`, and APPROVED requires re-checking —
  never approve to be agreeable.
````

**Per-round digest** (to the user, compact): round number; findings by severity; per finding verified/refuted/parked + action; adversary's `priorFindingResults` statuses; `converged` / `deadlockIds` / `roundCapReached`; `usage` when present.

**Review record**: `docs/superpowers/reviews/YYYY-MM-DD-<branch>-adversary-review.md` with the fields listed in step 13.

## Acceptance check (2-3 scenarios)

1. **Two-round convergence (codex adversary, Claude coder).** Clean feature branch; `detect --exclude claude` returns codex; `start` returns `R1-1` (important). Coder verifies, fixes in a commit, writes responses covering exactly `[R1-1]`, runs `rereview`. Codex is resumed with `exec ... resume <id> -`, the worktree is advanced and verified to the new HEAD, the response carries `priorFindingResults: [{id:"R1-1", status:"resolved"}]` and `APPROVED` → `converged: true`. Coder commits the record under `docs/superpowers/reviews/` only; `finalize` returns `auditedFiles` = that record and removes the worktree and `$TMPDIR/adversary-review/ar-xxxxxxxx`.
2. **Guards fire.** A responses file omitting an open id is refused with `missing: [...]` before any adversary spawn; a fourth `rereview` with `--round-cap 3` throws the round-cap error; `finalize` after a post-approval commit touching `src/` throws `finalization exemption violated`; a response reasserting `R1-1` in rounds 2 and 3 yields `deadlockIds: ["R1-1"]`.
3. **Argus allowlist + budget.** `start --argus-config <main clone>/.argus.config --budget 5 --adversary copilot` on an indexed repo: copilot argv contains `--available-tools view,grep,glob,argus-list_projects,...` (no `argus-rebuild_index`/`argus-drift_audit`) and `--max-ai-credits 500` (or the stderr warning on an old CLI); without `--argus-config`, the prompt contains the "NOT available" note and no argus tools are passed. `--budget abc` exits 2.
