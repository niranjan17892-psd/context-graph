# Skill: resolve-pr-reviews

Post-open PR review loop. Per push, waits for the bot reviewers (Claude auto-review GitHub workflow + SonarCloud on GitHub; CodeRabbit on Bitbucket) keyed to the head SHA (never comment counts or elapsed time), verifies each finding against the code, fixes what is real, replies with evidence, resolves bot threads, reports a digest per round, and stops on a zero-push round or the 8-round cap. Human comments are report-only. All host I/O lives in a bundled helper `scripts/pr-reviews.js`, which `open-pr` also uses (the two skills are installed/removed together as "the PR package").

Design doc (historical): `docs/superpowers/specs/2026-09-05-open-pr-and-resolve-pr-reviews-design.md`.

## Files to create (every path + purpose)

Self-contained: no `.md` in the skill may contain `ARGUS_ROOT`, `/../..`, `<argus-repo>`, `ARGUS_CONFIG`, `.argus.config`.

| Path | Purpose |
|---|---|
| `skills/resolve-pr-reviews/SKILL.md` | Orchestration (frontmatter + procedure below). |
| `skills/resolve-pr-reviews/round-handler-prompt.md` | Subagent prompt for one PR × one round (multi-PR mode); also the source of the round-file and digest formats (verbatim below). |
| `skills/resolve-pr-reviews/scripts/pr-reviews.js` | Helper CLI (12 subcommands). ESM, `// @ts-check`, Node built-ins + global `fetch` only. Exports `run*`, `makeDeps`, `resolveRepo`, `parseArgs`, `main`. |
| `skills/resolve-pr-reviews/scripts/lib/pr-host.js` | `detectHost`, `parsePrUrl`, git helpers. |
| `skills/resolve-pr-reviews/scripts/lib/pr-github.js` | GitHub adapter over the `gh` CLI. |
| `skills/resolve-pr-reviews/scripts/lib/pr-bitbucket.js` | Bitbucket Cloud REST 2.0 adapter. |
| `skills/resolve-pr-reviews/scripts/lib/pr-sonar.js` | SonarCloud Web API adapter + project-key resolution. |
| `skills/resolve-pr-reviews/scripts/lib/pr-coderabbit.js` | Pure parsers for CodeRabbit's base64url markers and finding markdown. |
| `skills/resolve-pr-reviews/scripts/lib/pr-findings.js` | Normalisers into one Finding shape; humans partitioned out. |
| `skills/resolve-pr-reviews/scripts/lib/pr-wait.js` | Pure wait evaluators + `waitRound` loop. |
| `skills/resolve-pr-reviews/scripts/lib/pr-errors.js` | `UsageError` (exitCode 2). |
| `skills/resolve-pr-reviews/scripts/lib/pr-session.js` | Session store instance (`dir: 'pr-reviews'`, `prefix: 'pr-'`, `label: 'pr-reviews'`, `init: { rounds: [] }`). |
| `skills/resolve-pr-reviews/scripts/lib/session-store.js` | Generic store factory; byte-identical to adversary-review's copy (see `adversary-review.md` for its full behavior: `<prefix><8 hex>` ids validated against path traversal, `$TMPDIR/<dir>/<id>/state.json`, init merged over data, labelled errors). |
| `tests/scripts/pr-*.test.ts`, `session-store.test.ts`, `skills-self-contained.test.ts`; fixtures `tests/fixtures/pr-reviews/{bitbucket-comments.json, coderabbit-finding.md, coderabbit-sticky.md, coderabbit-summary.md, github-review-threads.json, sonar-issues.json}` | Tests (see below). |

## Frontmatter (verbatim)

```yaml
---
name: resolve-pr-reviews
description: Use when a PR is open and its bot reviews need handling — "resolve the reviews", "babysit the PR", "handle the coderabbit comments", "/resolve-pr-reviews", or when open-pr hands off. Waits for Claude auto-review / CodeRabbit / SonarQube per push, verifies each finding, fixes what is real, replies with evidence, resolves bot threads, reports a digest per round, and stops on a zero-push round or the round cap.
---
```

## Purpose; fires when / does not

- **Fires when** a PR is open and its bot reviews need handling: "resolve the reviews", "babysit the PR", "handle the coderabbit comments", `/resolve-pr-reviews`, or open-pr hands off.
- **Does not**: act on human comments (never reply, fix, or resolve them); poll GitHub/Bitbucket by hand (all host calls go through the helper); create PRs (that is open-pr); create Jira tickets unless the user says so.
- Announce at start: "I'm using the resolve-pr-reviews skill."
- Helper:
  ```bash
  SKILL_DIR="$(dirname "$(realpath <path-to-this-SKILL.md>)")"
  PR="$SKILL_DIR/scripts/pr-reviews.js"
  ```

## Procedure (numbered, faithful; every gate, cap, stop condition)

### 1. PR discovery — never guess
Stop at the first step that yields PRs:
1. A PR url given to you, or PRs created by open-pr earlier this session.
2. Current branch has exactly one open PR: `node "$PR" discover --repo "$(pwd)"` → `openForBranch` has one entry.
3. For every repo touched this session, `discover --repo <path>`; present a multi-select of `openForBranch` + `recentOpen` across repos, labelled `<repo> #<n> <title> (<branch>)`. User picks one or more.
4. Ask the user for a repo and PR number, or a url.

Announce one line per PR: "Starting the resolver loop for <owner>/<repo> #<n>."

### 2. Orchestration
Round cap **8**. Per PR: `S=$(node "$PR" start --pr <url> --repo <checkout> | jq -r .sessionId)`.
- **One PR:** run the round loop in this session.
- **Several PRs:** this session is the orchestrator and never triages. Run `node "$PR" wait-round <S>` for every PR concurrently as background shell commands (Claude Code: `run_in_background: true`). When one returns `state: completed`, dispatch ONE handler subagent for that PR/round using `round-handler-prompt.md` (fill every placeholder); relay its digest verbatim; start the next `wait-round`. If not pushed, that PR is done. One handler per checkout at a time (a newer completion for a busy checkout waits). Any non-`completed` wait state ends that PR's loop.

### 3. Round loop (single PR, in-session)
1. **Wait.** `node "$PR" wait-round <S>` (blocking, keyed to session head). Round 1: first run `node "$PR" findings <S>`; if findings already exist, treat this as round 1 without waiting.
2. **Triage** every finding with `superpowers:receiving-code-review` discipline; verify at `path:line`. Outcomes: fixed / refuted / deferred. Humans are never triaged.
3. **Act.** Fixed: smallest correct change, tests, one commit per finding. Refuted/deferred: 1–3 sentence reply with evidence/reason, in the user's voice (andy-voice skill), via `node "$PR" reply`. Then `node "$PR" resolve` every bot thread acted on. Sonar: fix when clearly right and small; otherwise `--sonar accept --comment "<reason>"`.
4. **Push** once if anything was fixed; `record` the round (round-file shape below).
5. **Digest** to the user (format below). Nothing else is said to the user mid-loop.
6. Loop.

### 4. Termination — exactly one
1. **Zero-push round** — nothing verified worth fixing. Human-ready unless human comments are open.
2. **Round cap (8)** — stop; report open items.
3. **Wait state not `completed`** (`not-started`, `failed`, `skipped`, `timed-out`) — report verbatim and stop; never a clean round.
4. **User interrupts.**

An error thrown by `wait-round` itself (e.g. `wait-round: poll failed 5 times in a row: …`) is reported verbatim and stops that PR's loop.

On stop: `node "$PR" finish <S>`. Report: human-ready yes/no; open human comments; deferred items in moderate detail (finding, file:line, why deferred). Ask: **open a Jira ticket for these (TCNSR) or ignore?** Create tickets only if the user says so. On abort or error: `node "$PR" cleanup <S>`. Never leave a session behind.

### Round handler (subagent, `round-handler-prompt.md`)
1. `findings {SESSION_ID}`; interpret the GitHub `sonar` object (`queried: false` = never asked — not clean; `keySource: "fallback"` = guessed key, mention if Sonar returned nothing).
2. Per finding: read code at `path:line`; fixed (smallest change, tests, one commit per finding, record sha) / refuted (1–3 sentences with file:line evidence) / deferred (nit, out of scope, judgment call — say which and why). Sonar: fix when verifiably right and small (unused import, optional chain, obvious null guard); defer false positives (browser code flagged for Node APIs) or judgment calls (duplication, type nits) with `resolve --sonar accept --comment "<reason>"`.
3. Humans: never reply/fix/resolve; list in digest.
4. Replies: plain, direct, contractions, no praise, no "You're absolutely right"; write to temp file; `reply {SESSION_ID} --thread <threadId> --reply-to <replyTo> --body-file <f>`.
5. Resolve each bot thread after reply/fix push: `resolve {SESSION_ID} --thread <threadId>` (Sonar deferred: `--sonar accept|falsepositive --comment "<reason>"`; fixed Sonar issues need no resolve).
6. If anything fixed: `git push` once; read new head `git rev-parse HEAD`.
7. Write the round file; `record {SESSION_ID} --round-file <f>`. Return exactly the digest.

## Hard rules (critical ones VERBATIM)

SKILL.md `## Hard rules`, verbatim:
```
- Never act on human comments (reply, fix, or resolve).
- Never fix an unverified finding. Never write performative agreement.
- Never force-push. Never squash.
- Never treat `timed-out` / `not-started` as a clean round.
- Never key a round on comment counts or elapsed time — the helper owns the
  signal.
- One handler per checkout at a time.
- Mid-loop the user only sees digests.
```
Round-handler hard rules, verbatim: "Hard rules: never act on human comments; never fix an unverified finding; never force-push or squash; never ask the user anything — park judgment calls as deferred with the reason."

"Why the wait is what it is (measured 2026-09-05)" — keep in SKILL.md: Claude auto-review completes 3.6–4.7 min median after a push, p90 6–7 min, worst 10.5 min; the run is created within seconds, so "no run after 3 min" means none is coming. CodeRabbit takes 4–12 min, posts nothing until done; zero-finding rounds only bump its status sticky and it auto-replies to your replies minutes later — so the helper keys on the sticky's covered commit, never comment counts. One narrow fallback: a sticky with **no** coverage marker, updated after the push, together with a new "Actionable comments posted" summary, counts as done. A sticky naming an *older* commit never does.

## Scripts

### `scripts/pr-reviews.js` — CLI

Every subcommand prints one JSON object (`JSON.stringify(result, null, 2)`). Usage errors exit 2 (`usage()` and any `UsageError`); runtime errors exit 1 (`main().catch` prints `err.message`, exits `err.exitCode ?? 1`).

**`parseArgs(rest)`** → `{ positional, flag, has }`. `flag(name, def=null)`: returns `def` if absent, last token, or followed by a token starting `--`; first occurrence wins. `has(name)`: presence anywhere. `positional` = `rest[0]` if it does not start with `--`. `--repo` defaults to `process.cwd()` (except `start`, which records `flag('repo')` → null if absent).

| Subcommand | Args | Result |
|---|---|---|
| `host` | `--repo` | `{ host, owner, repo, branch, head, defaultBranch }` — always succeeds; `defaultBranch` best-effort (`null` on any failure: no token, offline, gh error) |
| `discover` | `--repo` | `{ host, owner, repo, branch, openForBranch: [...], recentOpen: [...] }` (`recentOpen` = open PRs minus those in `openForBranch`) — PR item `{ number, url, title, branch }` |
| `default-reviewers` | `--repo` | `{ host, reviewers }` — GitHub always `[]`; Bitbucket `[{ name, uuid }]` |
| `create` | `--repo --title --body-file [--base] [--reviewers a,b] [--draft]` (title, body-file required) | GitHub `{ host, url, number, head }`; Bitbucket `{ host, url, number, head }` |
| `start` | `--pr <url> [--repo]` | `{ sessionId, host, owner, repo, number, url, repoRoot, branch, head, since, createdAt }` |
| `wait-round <s>` | `[--sha] [--since <iso>] [--timeout-min] [--not-started-min] [--interval-s]` | see Wait |
| `findings <s>` | `[--all]` | see Findings |
| `reply <s>` | `--thread <id> --body-file <f> [--reply-to <databaseId>]` | `{ ok: true, url }` |
| `resolve <s>` | `--thread <id> [--sonar accept\|falsepositive] [--comment <t>]` | `{ ok }` |
| `record <s>` | `--round-file <f>` | `{ round, totals, pushed, head }` |
| `finish <s>` | — | trajectory report; deletes session |
| `cleanup <s>` | — | `{ cleaned: s }` |

Missing required flag → `missing required flag --<n>` exit 2; missing positional → a per-command usage string, exit 2; unknown command → `usage: pr-reviews.js host|discover|default-reviewers|create|start|wait-round|findings|reply|resolve|record|finish|cleanup ...`.

**`makeDeps()`**: `{ io: {execFileSync}, env: process.env, adapters: { github: makeGithub(io), get bitbucket() (lazy, memoized — a GitHub-only run never needs a Bitbucket token), sonar: makeSonar({fetch, env}) }, sessionIo, sleep, now: Date.now }`.

**`resolveRepo(repoRoot)`**: `detectHost(git remote get-url origin)` + `currentBranch` + `headSha`.

**`create`**: `absBodyFile = resolve(repoRoot, bodyFile)` (gh resolves body-file and head against the PROCESS cwd). Base = `--base` or best-effort default branch; neither → `UsageError('could not determine the default branch (no --base given and the lookup failed) — pass --base <ref>')`. GitHub: `createPr({ owner, repo, title, bodyFile: abs, base, draft, head: branch, reviewers, cwd: repoRoot })` then `head = prView(...).headRefOid`. Bitbucket: always loads `defaultReviewers`; reviewer list = `--reviewers` split on `,` (trimmed, non-empty) or all default reviewers' uuids; a value starting `{` passes as a uuid; else case-insensitive name match → uuid, unknown → `unknown reviewer "<s>" — pass a {uuid} or one of: <names>`; description = contents of absBodyFile.

**`start`**: `parsePrUrl`; GitHub head/branch from `prView` (`headRefOid`, `headRefName`); Bitbucket from `pr()` (`source.commit.hash`, `source.branch.name`) — the PR's own branch, not the checkout's. `since = createdAt = new Date(now()).toISOString()`.

**`findings`**: `round = rounds.length + 1`; `open()` filters `!resolved` unless `--all`.
- GitHub: `reviewThreads` → `normalizeGithubThreads`; renumber AFTER filtering (`numberFindings(open(findings), round)`, humans with `'H'`). Sonar key = `resolveSonarProjectKey(issueComments bodies, owner, repo)`; `sonarIssues = sonar.available ? await issuesForPr(key, number) : null`; normalized with `startIndex = findings.length`. Result: `{ host, url, head: headRefOid, round, findings: [...claude, ...sonar], humans, checks: checkRuns(head), sonar: { available, queried: sonarIssues !== null, projectKey, keySource }, sonarProjectKey }`.
- Bitbucket: `{ host, url, head: pr.source.commit.hash, round, findings, humans, checks: [] }` (no `sonar` object).

**`reply`**: body = trimmed file. A thread id is a Sonar key unless it matches `^PRRT_` or `^\d+$`. GitHub non-Sonar: target = `Number(--reply-to)` or look up the thread's first comment `databaseId` via `reviewThreads` (missing → `no review thread <id> on <url>`); `replyToComment`. Bitbucket: `reply(owner, repo, n, Number(thread), body)`. Sonar: needs token (`UsageError('missing SONAR_TOKEN')`), `sonar.comment`, `url: null`.

**`resolve`**: `--sonar` → token required (UsageError), `sonar.transition(thread, transition, comment)` → `{ok:true}`. GitHub → `{ ok: resolveThread(id).isResolved }` (real mutation result). Bitbucket → `resolve(..., Number(id))` → `{ok:true}`.

**`record`**: each decision `outcome` must be `fixed|refuted|deferred` else `decision <id>: outcome must be fixed|refuted|deferred, got <x>`. Append `{ ...roundFile, round: rounds.length+1, recordedAt: now ISO }` (computed fields placed after the spread so the file cannot override them). If `pushed`: `newHead` required (`round file has pushed:true but no newHead`); set `s.head = newHead`, `s.since = recordedAt`. Totals of the just-recorded round: `{ fixed, refuted, deferred, humans: last round's humans.length }`.

**`finish`**: `{ url, host, roundCount, lastRoundPushed, totals (summed over all rounds; humans from last round), deferred: all deferred decisions each with `round`, humansOpen: last round's humans, rounds }`, then deletes the session.

### Wait state machine (`lib/pr-wait.js` + `runWaitRound`)

States: `completed | failed | skipped | not-started | timed-out`; evaluator returns `null` = keep waiting. Only `completed` continues the loop. Constants: `INTERVAL_MS = 60_000`; `GITHUB_DEFAULTS = { notStartedMs: 3 min, timeoutMs: 20 min }`; `BITBUCKET_DEFAULTS = { timeoutMs: 25 min }`.

`evaluateGithub(snap{run, sonar}, {elapsedMs, notStartedMs, timeoutMs})`:
1. no run → `elapsed >= notStarted ? 'not-started' : null`.
2. run not `completed` → `elapsed >= timeout ? 'timed-out' : null`.
3. conclusion `skipped`/`cancelled` → `skipped`; any other non-`success` → `failed` (regardless of sonar).
4. sonar present and not `completed` → timed-out or null.
5. → `completed`.

`evaluateBitbucket(snap{sticky:{updatedOn, coveredCommitId}|null, summaryAfterSince}, {sha, sinceMs, elapsedMs, timeoutMs})`:
1. `coveredCommitId` and `sha.startsWith(coveredCommitId)` → `completed`.
2. sticky with NO coveredCommitId, `Date.parse(updatedOn) > sinceMs`, and `summaryAfterSince` → `completed` (never fires for a sticky naming an older commit).
3. else `elapsed >= timeout ? 'timed-out' : null`. There is no `not-started` on Bitbucket.

`waitRound({ poll, evaluate, intervalMs, sleep, now })`: poll immediately, count polls, evaluate with `now() - started`; on a state return `{ state, durationS: round(ms/1000), polls, snapshot }`; else `sleep(intervalMs)` and repeat.

`runWaitRound(session, opts)`: `sha = --sha || session.head`. Flags validated before any poll: `--interval-s`, `--timeout-min`, `--not-started-min` must be `> 0` (`UsageError('--<name> must be a positive number, got "<v>"')`); Bitbucket `--since` must parse (`--since must be an ISO date, got "<v>"`).
- **Poll retry wrapper** (`withPollRetry`): a thrown poll returns the last good snapshot (or the default); consecutive failures reset on success; total failures reported as `pollErrors`; 5 consecutive → throw `wait-round: poll failed 5 times in a row: <msg>`. While the most recent poll failed, an evaluator result of `not-started` is downgraded to `null` (broken polling must not satisfy the time guard); all other states pass through.
- **GitHub poll**: `runsForSha(owner, repo, sha)`; `null` (workflow file absent) → snapshot `{noWorkflow: true}` → immediately `not-started` (1 poll). Else `run` = first with `headSha === sha` (others ignored); `sonar` = check run with `app === 'sonarqubecloud'`, only fetched once the run is `completed`. Result: `{ state, sha, durationS, polls, pollErrors, host, sources: { claude: 'no-workflow' | run.conclusion ?? run.status ?? null, sonar: sonar?.conclusion ?? null } }`.
- **Bitbucket poll**: `comments()` → sticky via `normalizeBitbucketComments(..., {round:0})`; `summaryAfterSince` = any non-inline, non-reply, non-deleted comment by the CodeRabbit user (`isCodeRabbitUser`) whose raw is `isRoundSummary` OR matches `/Actionable comments posted/`, with `created_on > sinceMs` (since = `--since` or session `since`). Snapshot sticky = `{ updatedOn, coveredCommitId: coveredCommit(raw) }`. Result `sources: { coderabbit: coveredCommitId ?? null }`.

### Normalized finding shape (`lib/pr-findings.js`)

```ts
{ id: string,               // "R<round>-<n>" (findings) or "H<round>-<n>" (humans)
  source: 'claude'|'coderabbit'|'sonar'|'human',
  threadId: string, replyTo: string|number, path: string|null, line: number|null,
  severity: 'low'|'medium'|'high', title: string, body: string,
  promptForAgents: string|null, author: string, raisedOn: string|null,
  resolved: boolean, url: string|null, replies: number,
  category: string|null, effort: string|null, rule: string|null, issueType: string|null }
```
- `numberFindings(items, round, prefix='R')` → ids `<prefix><round>-<i+1>` in order.
- Severity maps: CodeRabbit `critical→high`, `major→medium`, else `low`; Sonar `BLOCKER/CRITICAL→high`, `MAJOR→medium`, else `low`.
- `firstLine(body)` = first non-empty trimmed line with surrounding `**…**` stripped.
- **GitHub threads**: bot logins `claude` and `claude[bot]` → source `claude` (anything else, including `coderabbitai[bot]`, is human). Item from first comment: `threadId: t.id`, `replyTo: first.databaseId`, `path`, `line: t.line ?? t.originalLine`, severity `medium`, title = firstLine, `raisedOn: originalCommit.oid`, `resolved: isResolved`, `replies: nodes.length - 1`. Threads with no comments skipped.
- **Bitbucket comments**: drop `deleted`. CodeRabbit identity = `user.uuid === '{8e30a1a0-99a4-4d73-8c53-18dbd9aeb94b}'` OR display name (or nickname) matches `botPattern` (default `DEFAULT_BOT_PATTERN = /coderabbit/i`) — exported as `isCodeRabbitUser`. Sticky = first non-inline, non-reply bot comment with `isStatusSticky(raw)` → `{ id, updatedOn: updated_on, raw }`. Items: skip replies; skip bot top-level (non-inline) comments (machinery); human top-level comments ARE reported (path/line null). Bot item uses `parseFinding`: severity from `severityWord`, title parsed (fallback firstLine), body parsed, `promptForAgents`, `category`, `effort`. `threadId: String(c.id)`, `replyTo: c.id`, `path: inline.path`, `line: inline.to ?? inline.from`, `resolved: c.resolution != null`, `url: links.html.href`, `replies` = count of live replies with that parent. Returns `{ findings, humans, sticky }`.
- **Sonar issues** → `id: R<round>-<startIndex+i+1>`, `threadId = replyTo = key`, `path` = component after the first `:`, `line: line ?? textRange.startLine`, `title: message`, `body: "<rule>: <message>"`, `author: 'sonar'`, `resolved: false`, `url: https://sonarcloud.io/project/issues?id=<project>&pullRequest=<n>&open=<key>`, `rule`, `issueType: type`.

### `lib/pr-coderabbit.js`

CodeRabbit hides state in link refs `[coderabbit-html-comment-v1-N]: https://coderabbit.ai/#html-comment-v1:<base64url>`. `decodeMarkers(raw)` = all matches of `/coderabbit\.ai\/#html-comment-v1:([A-Za-z0-9_-]+)/g`, base64url-decoded (`-`→`+`, `_`→`/`) to utf8, in order.
- `isStatusSticky(raw)`: any decoded marker contains `auto-generated comment: summarize by coderabbit.ai`, `final_review_risk_coverage:`, or `walkthrough_start`.
- `isRoundSummary(raw)`: any decoded marker contains `auto-generated comment by CodeRabbit for review status` (this identifies the per-round "Actionable comments posted" comment, NOT the sticky).
- `coveredCommit(raw)`: JSON after `final_review_risk_coverage:` (regex `(\{.*\})`) → `.coveredCommitId ?? null`.
- `parseFinding(raw)`: strip marker lines (`^\[coderabbit-html-comment-v1-\d+\]:.*$`), collapse 3+ newlines; header `^_A_ | _B_ ( | _C_)?` → category, severityWord, effort (cells stripped of emoji via `/[^\p{L}\p{N}&/ .-]/gu`, whitespace collapsed); title = first `**…**` line; `promptForAgents` = fenced block after `Prompt for AI Agents`; body = text minus header, title line, `🤖 Prompt for AI Agents` + its fence, and `✅ Addressed in commits…` line. No header → nulls (a prose line that merely looks like a header is not stripped); two-cell header → effort null.

### `lib/pr-github.js` — `makeGithub(io)` over `gh`

All calls `execFileSync('gh', args, { encoding, maxBuffer 64MB, stdio: ['ignore','pipe','pipe'], cwd? })`; on failure rethrow with captured stderr appended to the message (and `.stderr`, `.status` attached) — never leaked to the process. `-R <owner>/<repo>` on `pr`/`run` commands.
- `createPr` → `gh pr create -R o/r --title T --body-file F [--base B] [--draft] [--head H] [--reviewer a,b]` in `cwd`; URL = last stdout line; number from `/pull/(\d+)` else throw `gh pr create returned no PR url: …`.
- `prView` → `gh pr view <n> -R o/r --json number,url,headRefOid,headRefName,baseRefName,state,isDraft`.
- `runsForSha(o, r, sha, workflow='claude-auto-review.yml')` → `gh run list -R o/r --workflow <wf> --commit <sha> --json databaseId,headSha,status,conclusion,createdAt,updatedAt`. Returns `null` only if stderr contains `HTTP 404` AND `/repos/<o>/<r>/actions/workflows/<wf>` AND the repo probe `gh api repos/<o>/<r> --jq .full_name` succeeds; otherwise rethrow the original.
- `checkRuns` → `gh api repos/o/r/commits/<sha>/check-runs?per_page=100 --jq '[.check_runs[] | {name, status, conclusion, app: .app.slug}]'`.
- `reviewThreads` → paginated `gh api graphql -f query=<THREADS_QUERY> -f o= -f r= -F n= [-f after=]`; query fetches `headRefOid`, `reviewThreads(first:100, after)` with `pageInfo`, nodes `{ id isResolved path line originalLine comments(first:50){ nodes{ databaseId author{login} body createdAt url originalCommit{oid} } } }`. Null PR → `no pull request o/r#n (or no access)`. Returns `{ headRefOid, threads }`.
- `replyToComment` → `gh api repos/o/r/pulls/<n>/comments/<id>/replies -f body=<body>` → `{ id, url: html_url }`.
- `resolveThread` → GraphQL `mutation($id:ID!){resolveReviewThread(input:{threadId:$id}){thread{isResolved}}}` → `{ isResolved }`.
- `issueComments` → `gh api repos/o/r/issues/<n>/comments?per_page=100&page=<p> --jq '[.[] | {author: .user.login, body}]'`, paging until a page < 100, max 10 pages.
- `defaultBranch` → `gh repo view o/r --json defaultBranchRef --jq .defaultBranchRef.name` (trimmed).
- `openPrs(o, r, {branch, limit=10})` → `gh pr list -R o/r --state open --limit 10 --json number,url,title,headRefName [--head <branch>]` → `{ number, url, title, branch }`.

### `lib/pr-bitbucket.js` — Bitbucket Cloud REST 2.0

Token `BITBUCKET_TOKEN` else `WCG_ATLASSIAN_BITBUCKET_TOKEN`; missing → `UsageError('missing BITBUCKET_TOKEN (or WCG_ATLASSIAN_BITBUCKET_TOKEN) for Bitbucket API access')`. Base `https://api.bitbucket.org/2.0`; headers `Authorization: Bearer <token>`, `Accept: application/json` (+ JSON content type with body). Non-2xx → `bitbucket <METHOD> <path> → <status> <body text>`; 204/empty → null. `paged` follows `next`.
- `pr` GET `/repositories/{o}/{r}/pullrequests/{n}`.
- `createPr` POST `/repositories/{o}/{r}/pullrequests` body `{ title, description, source:{branch:{name}}, destination:{branch:{name}}, close_source_branch: true, reviewers: [{uuid}] }` → `{ url: links.html.href, number: id, head: source.commit.hash }`.
- `comments` paged GET `/pullrequests/{n}/comments?pagelen=100`.
- `reply` POST `/pullrequests/{n}/comments` `{ content:{raw}, parent:{id} }` → `{ id, url }`.
- `resolve` POST `/pullrequests/{n}/comments/{id}/resolve`.
- `defaultReviewers`: GET `pullrequests?state=MERGED&sort=-updated_on&pagelen=5`; none → `[]` (no detail call); else GET the first PR's detail (list representation omits reviewers) → `[{ name: display_name, uuid }]`.
- `mainBranch` GET repo → `mainbranch.name`.
- `openPrs(o, r, {branch})` GET `pullrequests?state=OPEN&sort=-updated_on&pagelen=10[&q=<encodeURIComponent('source.branch.name="<branch>"')>]` → `{ number: id, url, title, branch: source.branch.name }`.

### `lib/pr-sonar.js` — SonarCloud

`SONAR_TOKEN`, Basic auth `base64("<token>:")`; base `https://sonarcloud.io/api`. `available: Boolean(token)`.
- `resolveSonarProjectKey(bodies, owner, repo)` → first match of `/sonarcloud\.io\/[^\s)]*[?&]id=([^&\s)]+)/` (URI-decoded) → `{ key, source: 'comment' }`, else `{ key: '<owner>_<repo>', source: 'fallback' }`. `sonarProjectKey` returns only the key.
- `issuesForPr(key, n)`: no token → `[]`; GET `/issues/search?componentKeys=<key>&pullRequest=<n>&resolved=false&ps=500&p=<p>` for p=1..20; 404 on page 1 → `null` (unknown project/PR — NOT clean); 404 later → issues so far; stop when `pageIndex*pageSize >= total`. Non-404 errors → `sonar GET <path> → <status> <text>`.
- `transition(key, t, comment)`: token required (UsageError); if comment, POST form `/issues/add_comment {issue, text}` first; then POST `/issues/do_transition {issue, transition}`.
- `comment(key, text)`: token required; POST `/issues/add_comment`.

### `lib/pr-host.js`

- `detectHost(url)`: regex `/(?:@|\/\/)(?:[^@/]+@)?(github\.com|bitbucket\.org)[:/]([^/\s]+)\/([^/\s]+?)(?:\.git)?\/?$/i` → `{ host: 'github'|'bitbucket', owner, repo }`; else `unsupported remote: <url> (only github.com and bitbucket.org)`. Handles https, ssh, with/without `.git`.
- `parsePrUrl`: `https://github.com/<o>/<r>/pull/<n>` or `https://bitbucket.org/<o>/<r>/pull-requests/<n>` (trailing path allowed) → `{ host, owner, repo, number }`; else `not a GitHub or Bitbucket PR url: <url>`.
- `originFetchUrl` = `git -C <root> remote get-url origin`; `currentBranch` = `rev-parse --abbrev-ref HEAD`; `headSha` = `rev-parse HEAD`.

### Session state

`$TMPDIR/pr-reviews/pr-<8 hex>/state.json`: `{ host, owner, repo, number, url, repoRoot, branch, head, since, createdAt, rounds: [ { ...roundFile, round, recordedAt } ], sessionId }`.

## Tests to write (list the behaviors the existing tests pin)

- **pr-host**: https GitHub, ssh Bitbucket, ssh GitHub without `.git`; rejects other hosts; parses GitHub and Bitbucket PR urls (with trailing path); rejects non-PR urls; git helpers call the right git args.
- **pr-github** (fake execFileSync): createPr args + number parse; `--head` and repo `cwd`; reviewers as `--reviewer a,b`; runsForSha filters by commit+workflow; null on workflow-404 with reachable repo; rethrow when repo probe fails; rethrow when only `err.message` (not stderr) names the workflow; rethrow non-404; reviewThreads paginates by cursor; clear error on null PR; resolve mutation with id; reply endpoint with body field; exact checkRuns path/jq; openPrs mapping; prView field list; issueComments jq, pagination until short page, stop at 10 pages; defaultBranch argv trimmed; stderr captured via explicit stdio and folded into the thrown message.
- **pr-bitbucket** (fake fetch): token required with WCG fallback; bearer auth + comment pagination; createPr body/mapping; reply with parent, resolve endpoint; defaultReviewers from PR detail, `[]` without detail call when no merged PR; openPrs branch filter; non-2xx error has status and body.
- **pr-sonar**: key source comment/fallback; key from link; unavailable without token (`[]`); paging with Basic `token:`; null for page-1 404 (distinct from `[]`); transition comments first then transitions; plain comment; later-page 404 keeps collected issues; stops after 20 pages.
- **pr-coderabbit**: decode markers / none; real sticky recognised, finding is not; round-summary not mistaken for sticky; covered commit read; round-summary detection; parseFinding header cells without emoji, bold title, prompt extraction, body cleanup, no header, header-like prose untouched, two-cell header.
- **pr-findings**: severity maps; GitHub bot threads (both login spellings) → claude; humans separate with `H` prefix; `coderabbitai[bot]` treated as human on GitHub; Bitbucket sticky found and top-level excluded; coderabbit finding parsed; human inline root with reply count, replies never items; custom bot pattern; uuid match despite rename; `DEFAULT_BOT_PATTERN` exported; human top-level comment reported with null path/line; Sonar component→path and startIndex numbering; `textRange.startLine` fallback; `isCodeRabbitUser` cases; `numberFindings` order/prefix.
- **pr-wait**: every GitHub evaluator transition (null under guard, not-started, in-progress null, timed-out, completed no sonar, waits for sonar, failed/skipped terminal regardless of sonar); Bitbucket completed on coverage, null on older-commit sticky, sticky update alone not completion, fallback only for marker-less sticky, never for older-commit sticky, no not-started; `waitRound` polls immediately then each interval.
- **pr-reviews-cli** (fake adapters/git, injected `now`/`sleep`): host defaultBranch GitHub/Bitbucket, null on no token and on gh throw; discover split; default reviewers; create on GitHub (head from prView), relative body-file resolved against repo (both hosts), default-branch fallback, UsageError when base unknown, Bitbucket name→uuid, unknown name rejected; start uses PR branch not checkout branch (both hosts); wait-round GitHub completes with `polls: 3, durationS: 120`, slept `[60000, 60000]`, not-started after guard (`polls: 4`), ignores other shas, no-workflow immediate not-started (1 call); flag validation (exit 2, never polls) for timeout-min, interval-s 0, not-started-min, since; poll resilience (transient errors, 5-failure rethrow, failed poll never satisfies not-started → `pollErrors: 4` then completed); Bitbucket completes on covering sticky; summary vs sticky not confused; findings merges claude + sonar excluding resolved/humans; sonar object reports key/source/queried; no token → unavailable/not queried; 404 → not queried; numbering after filtering (no id collisions), humans renumbered; reply maps thread → databaseId; resolve mutation vs sonar transition; resolve returns real mutation result; `--reply-to` skips re-fetch; Sonar reply/resolve without token exit 2; record appends/advances head+since/totals; finish trajectory + deletes; round file cannot override round/recordedAt; cleanup; parseArgs semantics (absent, last token, followed by flag, value, positional detection, `has` anywhere, repeated flag → first).
- **pr-session**: `pr-` prefix with empty `rounds`, round-trip; clear unknown/corrupt errors. **session-store** + **skills-self-contained**: as in adversary-review.md (copies identical; helper exists).

## Output format (templates verbatim)

**Round file** (`record --round-file`):
```json
{ "sha": "<head reviewed>", "waitState": "completed", "pushed": true|false, "newHead": "<sha if pushed>",
  "decisions": [ { "id": "R{ROUND}-1", "threadId": "…", "source": "claude|coderabbit|sonar", "path": "…", "line": 0,
                   "title": "…", "outcome": "fixed|refuted|deferred", "commit": "<sha if fixed>", "reason": "<if not fixed>" } ],
  "humans": [ { "id": "H{ROUND}-1", "threadId": "…", "author": "…", "path": "…", "line": 0, "title": "…" } ] }
```

**Digest** (per round):
```
Round {ROUND}/{ROUND_CAP} — {PR_URL} @ <head>
Findings: <n> (claude <a>, coderabbit <b>, sonar <c>) · high <h> / medium <m> / low <l>
- R{ROUND}-1 [source, severity] <title> — fixed <sha> | refuted: <reason> | deferred: <reason>
- …
Humans needing you: <n>
- H{ROUND}-1 <author> <path>:<line> — <title>
Checks: <name: conclusion, …>
Pushed: yes (<newHead>) | no
```
Print `sonar <c>` only when `sonar.queried` is true; otherwise `sonar: not checked (no SONAR_TOKEN)` or `sonar: not checked (404 for <projectKey>)`. On Bitbucket there is no `sonar` object; omit the term.

**`round-handler-prompt.md`** (verbatim, full file):
`````markdown
<!-- skills/resolve-pr-reviews/round-handler-prompt.md
     Dispatched once per PR per round by the orchestrator. Fill every {PLACEHOLDER}. -->

You are handling ONE review round for ONE pull request. You do not talk to the
user; you return the digest below as your final message and nothing else.

PR: {PR_URL}
Checkout: {REPO_ROOT}   (cd here for all git and test commands)
Session: {SESSION_ID}
Helper: node "{SKILL_DIR}/scripts/pr-reviews.js"
Round: {ROUND} of {ROUND_CAP}

## Procedure

1. `findings {SESSION_ID}` → the open findings and humans for the current head.
   On GitHub the result carries a `sonar` object
   (`{available, queried, projectKey, keySource}`). `queried: false` means Sonar
   was never actually asked — no `SONAR_TOKEN` (`available: false`) or the project
   key returned 404 — so zero Sonar findings is NOT a clean Sonar. Say so in the
   digest; never report a blind Sonar as clean. `keySource: "fallback"` means the
   key is the `<owner>_<repo>` guess (the sonarqubecloud bot comment was not
   found) — mention it if Sonar returned nothing.
2. For EVERY finding, apply receiving-code-review discipline: read the code at
   `path:line` in the checkout, decide from evidence. Outcomes:
   - **fixed** — the finding is real. Make the smallest correct change, run
     the relevant tests, commit with a normal message (one commit per finding,
     no squashing). Record the commit sha.
   - **refuted** — the finding is wrong. Write a 1–3 sentence reply with
     file:line evidence.
   - **deferred** — real but a nit, out of scope, or a judgment call. Write a
     1–3 sentence reply saying which and why.
   Sonar findings: fix when verifiably right and small (unused import, optional
   chain, obvious null guard). Defer when it is a false positive (browser code
   flagged for Node APIs) or a judgment call (duplication, type nits) and use
   `resolve --sonar accept --comment "<reason>"`.
3. **Humans**: never reply, fix, or resolve. List them in the digest.
4. Replies: plain, direct, contractions, no praise, no "You're absolutely
   right". Write them to a temp file and post with
   `reply {SESSION_ID} --thread <threadId> --reply-to <replyTo> --body-file <f>`
   (`replyTo` is on the finding; passing it skips a re-fetch of every review
   thread — omit it and the helper looks it up).
5. After each reply (and after each fix is pushed in step 6), resolve the bot
   thread: `resolve {SESSION_ID} --thread <threadId>` (Sonar: add
   `--sonar accept|falsepositive --comment "<reason>"` for deferred; fixed
   Sonar issues disappear on the next analysis and need no resolve).
6. If anything was fixed: `git push` once. Read the new head with
   `git rev-parse HEAD`.
7. Write the round file and `record {SESSION_ID} --round-file <f>`:
   ```json
   { "sha": "<head reviewed>", "waitState": "completed", "pushed": true|false, "newHead": "<sha if pushed>",
     "decisions": [ { "id": "R{ROUND}-1", "threadId": "…", "source": "claude|coderabbit|sonar", "path": "…", "line": 0,
                      "title": "…", "outcome": "fixed|refuted|deferred", "commit": "<sha if fixed>", "reason": "<if not fixed>" } ],
     "humans": [ { "id": "H{ROUND}-1", "threadId": "…", "author": "…", "path": "…", "line": 0, "title": "…" } ] }
   ```

## Return exactly this digest

```
Round {ROUND}/{ROUND_CAP} — {PR_URL} @ <head>
Findings: <n> (claude <a>, coderabbit <b>, sonar <c>) · high <h> / medium <m> / low <l>
- R{ROUND}-1 [source, severity] <title> — fixed <sha> | refuted: <reason> | deferred: <reason>
- …
Humans needing you: <n>
- H{ROUND}-1 <author> <path>:<line> — <title>
Checks: <name: conclusion, …>
Pushed: yes (<newHead>) | no
```

In the Findings line, print `sonar <c>` only when `sonar.queried` is true. When it
is false, print `sonar: not checked (no SONAR_TOKEN)` or
`sonar: not checked (404 for <projectKey>)` instead of a count — a Sonar that was
never asked is not a Sonar that found nothing. On Bitbucket there is no `sonar`
object; omit the term.

Hard rules: never act on human comments; never fix an unverified finding;
never force-push or squash; never ask the user anything — park judgment calls
as deferred with the reason.
`````
Placeholders: `{PR_URL} {REPO_ROOT} {SESSION_ID} {SKILL_DIR} {ROUND} {ROUND_CAP}`.

**Final report on stop**: human-ready yes/no; open human comments; deferred items (finding, file:line, why deferred); then the question "open a Jira ticket for these (TCNSR) or ignore?"

## Acceptance check (2-3 scenarios)

1. **GitHub single PR, one fix then clean.** `start` records the PR head. `findings` (round 1) returns `R1-1` (claude, unresolved) and `sonar.queried: true`. Agent fixes, commits, replies/resolves, pushes, `record` with `pushed: true, newHead` → session head advances. `wait-round` polls immediately, sees the Claude run for the NEW sha `completed/success` and the `sonarqubecloud` check completed → `completed`. Round 2 has no findings → zero-push round; `finish` returns `roundCount: 2, lastRoundPushed: false` and deletes the session.
2. **Bitbucket CodeRabbit keyed on coverage.** Session head `bf6be2ec04bb…`; both the round-summary comment and the real sticky exist; wait-round picks the sticky (coverage marker), returns `completed` in 1 poll with `sources.coderabbit: "bf6be2ec04bb"`. A sticky naming an older commit plus a fresh summary does NOT complete; the loop times out at 25 min (`timed-out`), and the skill reports it verbatim and stops.
3. **Blind Sonar and failure modes.** No `SONAR_TOKEN`: `findings.sonar = {available:false, queried:false, ...}` and the digest prints `sonar: not checked (no SONAR_TOKEN)`; `resolve --sonar accept` exits 2. A repo without `claude-auto-review.yml` makes `wait-round` return `not-started` after one poll with `sources.claude: "no-workflow"`. Five consecutive poll errors throw `wait-round: poll failed 5 times in a row: …`.
