# Skill: bootstrap-argus

## Files to create

- `skills/bootstrap-argus/SKILL.md` — the only file in the skill. A conversational, two-phase onboarding procedure that drives `scripts/project.js` (shell) in Phase 1 and the `argus` MCP tools in Phase 2.

The skill depends on (but does not own) `scripts/project.js`, `scripts/lib/discover.js`, `scripts/lib/clone.js`, `scripts/lib/presets.js`, `scripts/lib/config.js` (`writeConfig`/`removeProject`/`validateRepos`), `project_conventions/_template.md` and `project_conventions/README.md`. Their surface is described under "Scripts/helpers relied on".

## Frontmatter (verbatim)

```yaml
---
name: bootstrap-argus
description: Use when the user wants to ADD or onboard a new project/codebase to Argus (e.g. "argus, add a project", "onboard WCG", "add a new project", "set up argus for <org>"), to FINISH a half-done onboarding ("finish onboarding <name>", or when list_projects reports a project with onboardingPending true), or to edit/remove a project. Discovers repos, registers the project, builds its index, and seeds its conventions doc.
---
```

Body title: `# Bootstrap Argus — Project Onboarding`.

## Purpose; fires when / does not fire when

Onboards a new Argus **project**: a named silo of 1–N repos with its own index and optional conventions doc. Argus is project-agnostic (no hardcoded repo list); `tcn` is just one project.

Fires when the user wants to add/onboard a project, finish a half-done onboarding (including when the argus skill detects `onboardingPending: true` from `list_projects` and hands off), or edit/remove a project. It does not answer code questions about an already-onboarded project (that is the `argus` skill), and it is not part of `npm run setup`.

Stated up front in the body:
- **Golden rule:** "every mutation of `.argus.config` goes through `node scripts/project.js`." Never hand-edit the JSON, even for a typo; the CLI validates paths, merges fields, and is the tested path.
- The procedure is **two-phase and the split is load-bearing**: Phase 1 is shell only (no `argus` MCP tool calls, because the project's index does not exist yet, so project-scoped tools would error or silently answer for the wrong project — "a chicken-and-egg constraint, not a suggestion"). Phase 2 is MCP-driven, only after the Phase-1 build has finished.

## Procedure

### Phase 1 — Configure + Build

1. **Ask for the project name** in kebab-case (e.g. `wcg`, `acme-platform`). If given spaces/camelCase/uppercase, slugify it and confirm with the user. Explain it is permanent-ish: it becomes `.argus/<name>/` and the `project` argument on every MCP call.
2. **Discover repos.** Ask where the repos live, then use whichever applies:
   - **GitHub:** run `gh auth status` first. If authenticated: `node scripts/project.js discover --host github --source <org>`. If `gh` is missing or unauthenticated, say so plainly and offer bitbucket/local/manual instead; do not force GitHub.
   - **Bitbucket (Cloud only):** ask for the workspace slug and which auth they have. Either `node scripts/project.js discover --host bitbucket --source <workspace> --token <token>` or (common case, what WCG uses) `node scripts/project.js discover --host bitbucket --source <workspace> --user <user> --app-password <app_password>`. Tell the user this targets Bitbucket Cloud (`api.bitbucket.org`) only, not Server/Data Center, and that the credential passed as a CLI argument is visible in shell history/process listing (fine for a one-off, don't encourage reusing long-lived credentials this way).
   - **Known org preset:** if one exists (e.g. `tcn`), skip discovery: `node scripts/project.js discover --host preset --source tcn` (local static data, no network, no auth).
   - **Local (already cloned):** `node scripts/project.js discover --host local --source <parent-dir>`. Scans immediate subdirectories containing `.git` only, not recursive.
   - **Manual:** user pastes URLs or local paths directly; skip discovery for those.

   **Curate, don't dump.** Discovery lists everything visible, likely more than belongs. Present the list (name + host + URL or path), have the user pick, merge picks from multiple sources and manual entries into one working list, each with a `key` (slugified name by default; offer renaming before registering).
3. **Decide where to clone.** For every curated repo given as a URL, ask for a parent directory to clone into (one answer covers all unless the user splits them). Suggest the parent directory of this `argus-panoptes` checkout as default. Already-local repos keep their existing path.
4. **Confirm before doing anything.** Show the final plan: project name, full repo list (key → URL-or-path → clone target if applicable), and say plainly you are about to clone N repositories and start an index build. **Wait for explicit confirmation.** Never do this silently.
5. **Clone and register.** For each URL repo not already local: `node scripts/project.js clone --url <url> --into <parent-dir>/<key>` (no-op with a message if the target exists; safe to re-run). Then register the **whole curated set in one call**, because `add` fully replaces the project's repo map:
   `node scripts/project.js add --name <name> --repo <key1>=<path1> --repo <key2>=<path2> [...]`
   Verify with `node scripts/project.js list` that the repo count is right.
6. **Kick off the index build in the background:** `npm run build-index -- --project <name>`, with the Bash tool's `run_in_background: true`. Explain that by default it fetches and indexes each repo's `origin/<default>` via a persistent detached worktree under `.argus/<name>/worktrees/` (not the checkout from step 5; a repo with no resolvable `origin/<default>` falls back to indexing the checkout). It takes minutes to over an hour. Tell the user it is building and **do not block the conversation**; you will be notified on completion.

   **Before Phase 2:** the running MCP server reads `.argus.config` only at startup and will not see the new project. Once the build finishes, tell the user to restart the Claude Code session (or reconnect the MCP server). Do not attempt Phase 2 MCP calls until then; verify with `list_projects` rather than assuming.

   The user does not need to remember anything across the restart: until Phase 2 registers a conventions doc, `list_projects` reports `onboardingPending: true` and the argus skill offers to continue. End Phase 1 with exactly this framing: *"Restart Claude Code, then just talk to argus — it will notice <name> isn't fully onboarded and pick up where we left off."* (Saying "finish onboarding <name>" also lands in Phase 2.)

### Phase 2 — Seed the Conventions Doc

Start only when the build has finished **and** `list_projects` shows the project with `indexed: yes`. Pass `project: "<name>"` on every MCP call. If arriving from the argus skill's `onboardingPending` detection, skip Phase 1 and start at step 7. Step 10 is what clears `onboardingPending`, so do not stop before it.

7. **Derive the mechanical sections** into `project_conventions/<name>.md`, mirroring `project_conventions/_template.md`'s section order and headers (Repos & ownership, Flow conventions, Entry-point taxonomy, Symptom → likely region, Tech stack; see `project_conventions/README.md` for how it is consumed):
   - Repo list: from `list_projects` or `node scripts/project.js list`.
   - Tech stack: `index_status` for aggregate stats, plus project-scoped `search_index` for representative node types (GraphQL, Vue/Pinia, Lambda/Step Functions, Socket.IO, Bedrock/AgentCore agents, etc.) to see what is present vs absent.
   - Entry-point taxonomy: `find_entry_points` and `list_flows` (filtered by `repo`/`entryKind`) to sketch which repos own which entry-point kinds.
   The doc is per-user and gitignored; it stays out of feature branches.
8. **Seed `## Documentation landscape`:**
   1. `npm run build-index -- --docs --project <name>`.
   2. `list_spec_docs({ project: "<name>", limit: 200 })`; inspect `summary.byGenre`, `summary.reposWithNoIntentDocs`, and `excluded` counts.
   3. Three cases:
      - `intentDocs` is 0 but the corpus is large: docs live somewhere classified as reference/instruction. Show top directories, ask which hold design decisions, write those as `intent` rules.
      - `intentDocs` is 0 and the corpus is small: the project genuinely has no design-intent docs. Say so, write nothing, move on ("an empty section is a correct outcome, not a failure").
      - `intentDocs` is healthy: spot-check `low`-confidence entries with `minConfidence` filtering; ask only about directories whose genre looks wrong. Do not invent rules for things inference got right.
   4. Write the rules under `## Documentation landscape`, re-run the `--docs` build, confirm no `matched 0 files` warnings remain.
   Never ask the user to enumerate doc directories from memory; lead with what the manifest found. **Present derived glob → genre rules for confirmation before writing them**; never write them silently.
9. **Interview for judgment sections** (architectural law, not observable): repo ownership rules ("resolver logic never lives in X"), async/flow conventions (sync return vs hand-off to queue/Step Function; the project's equivalent of tcn's "async by default"), symptom → region hints ("X returns null", "async result never arrives"). If the user doesn't know, write a clear `TODO:` marker rather than guessing or leaving it silently blank ("a wrong convention is worse than an honest gap", since sub-agents treat the file as ground truth).
10. **Register the conventions path**, re-passing every `--repo` from step 5:
    `node scripts/project.js add --name <name> --repo <key1>=<path1> --repo <key2>=<path2> [...] --conventions project_conventions/<name>.md`
    `add` replaces the repo map wholesale (it preserves `sampleCodePath`/`conventions` only when omitted, never `repos`); there is no conventions-only edit mode. Confirm with `node scripts/project.js list`.

## Hard rules

- **Never hand-edit `.argus.config`.** Every mutation — add, remove, repo changes, conventions path — goes through `node scripts/project.js`.
- **Confirm before cloning or registering.** Step 4 is not optional; an N-repo clone and an index build are real, visible side effects.
- **The index build is background and long.** Never block on it; use `run_in_background: true` and resume when notified.
- **Phase separation is load-bearing.** Phase 1 = shell only, no argus MCP tools. Phase 2 = MCP-driven and requires the running server to have picked up the new project.
- **Idempotent and re-runnable.** `clone` no-ops on an existing target; `add` is safe to re-run; discovery is read-only. An interrupted Phase 1 is simply re-run.
- **Edit/remove later** with `node scripts/project.js add --name <name> ...` (full desired repo set) or `node scripts/project.js remove --name <name>`.

## Sub-agents

None.

## Scripts/helpers relied on

`scripts/project.js` (ESM, `node scripts/project.js <cmd>`; no args / `--help` / `-h` / `help` prints usage, exit 0; any parse or runtime error prints the message and exits 1). Usage text:

```
node scripts/project.js add --name <name> --repo <key>=<localpath> [--repo <key>=<path> ...] [--sample <path>] [--conventions <path>] [--repo-root <dir>]
node scripts/project.js remove --name <name>
node scripts/project.js list
node scripts/project.js discover --host <github|bitbucket|local|preset> --source <org|workspace|dir|presetName> [--token <t>] [--user <u> --app-password <p>] [--limit <n>]
node scripts/project.js clone --url <url> --into <path>
```

- **add:** requires `--name` and at least one `--repo key=path` (malformed `key=path` is rejected). Repo keys and project names must match `/^[A-Za-z0-9._-]+$/` and not be `.`/`..` (they become path components under `.argus/`). Missing repo paths only warn ("ok if it's about to be cloned"). `repoRoot` defaults to the common parent of the repo paths, else cwd. Writes via `writeConfig`: replaces that project's entry (`repos` stored as `{ path }` objects), preserves other projects and top-level `dailyDriverWorktree`/`installTargets`, keeps existing `sampleCodePath`/`conventions` when not passed. Prints `Added project '<name>' with N repo(s).`
- **remove:** reads the repo map first, calls `removeProject`, then cleans up `.argus/<name>` worktrees (refusing if the name resolves outside `.argus`; cleanup failure only warns). Prints removed or "not found — nothing to remove".
- **list:** `N project(s):` then one line per project: name, repo count, `indexed: yes|no` (whether `.argus/<name>` exists). Empty config prints a hint to run `add`.
- **discover:** read-only; prints a JSON array of `{ key, name, cloneUrl?, path?, host }`. `key` is a slug (lowercase, non-alphanumeric runs → `-`, trimmed). GitHub: `gh repo list <org> --json name,url,sshUrl --limit <n>` (default limit 200), error suggests `gh auth login`. Bitbucket Cloud: paginates `https://api.bitbucket.org/2.0/repositories/<workspace>?pagelen=100` via `next`; auth is `Bearer <token>` or Basic `user:appPassword` (user+app-password wins if both); 401/403 → "Bitbucket authentication failed ... Check your token/app password."; uses the `https` clone link. Local: immediate subdirectories with a `.git` entry (unreadable dir → empty list). Preset: entries from `PRESETS[<name>]` (unknown preset errors listing available ones). `--limit` must be a positive number.
- **clone:** `git clone <url> <into>`; creates the parent dir; existing target → `Already exists at <into> — skipped.` (success); failure prints the reason and exits 1.
- `scripts/lib/presets.js` exports `PRESETS.tcn` mapping keys `apollo-server, customer-app, office-addin, integrations, agreement-ingestion, libs, ai, utils, platform, tcn-agents` to `{ dirs, url }` (GitHub `Lean-Law-Labs/*` URLs). Presets are optional org data, not core.
- `npm run build-index -- --project <name>` (full build) and `npm run build-index -- --docs --project <name>` (docs manifest only).
- MCP tools (Phase 2 only): `list_projects`, `index_status`, `search_index`, `find_entry_points`, `list_flows`, `list_spec_docs`.

## Output format

No fixed report template. Required verbatim user-facing line at the end of Phase 1:

> *"Restart Claude Code, then just talk to argus — it will notice <name> isn't fully onboarded and pick up where we left off."*

The artifact produced is `project_conventions/<name>.md` following `_template.md` sections: `## Repos & ownership`, `## Flow conventions (the "law")`, `## Entry-point / role → repo taxonomy (for tracing)`, `## Symptom → likely region (for debugging)`, `## Tech stack`, `## Documentation landscape`, with `TODO:` markers for unknown judgment sections.

## Acceptance check

1. **GitHub onboarding, gh unauthenticated.** User: "argus, add a project for acme". Agent asks for a kebab-case name, runs `gh auth status`, finds it unauthenticated, says so and offers bitbucket/local/manual; makes no MCP calls; presents a curated list, asks for a clone parent dir, shows the full plan and waits for "yes" before any `clone`/`add`; registers all repos in one `add`; starts `build-index -- --project acme` in the background; ends with the restart line.
2. **Finish onboarding after restart.** `list_projects` shows `acme` with `onboardingPending: true`. Agent skips Phase 1, derives mechanical sections via `index_status`/`search_index`/`find_entry_points`/`list_flows`, builds the docs manifest, proposes glob → genre rules and waits for confirmation, writes `TODO:` for unknown ownership rules, and finally runs `add` re-passing every `--repo` plus `--conventions project_conventions/acme.md`.
3. **Typo fix request.** User asks to fix a repo path in `.argus.config`. Agent does not edit the JSON; it re-runs `node scripts/project.js add --name <name>` with the full corrected repo set and verifies with `list`.
