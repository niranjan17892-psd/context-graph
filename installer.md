# Rebuild spec: installer, fix-links, uninstaller

This spec covers `npm run setup`, `npm run fix-links`, `npm run uninstall`, the client-adapter registry, the skill linker, the SessionStart hook, the build/copy helpers, and the repo-root dotfiles. It was written from the source at commit `d56a1ac7` and checked against the code and its tests. Exact command strings, prompts and log lines are quoted verbatim; prose is condensed.

**Required reading before implementing:** the Gotchas section of `CLAUDE.md`. The entries on multi-client install targets, MCP skip-and-warn, WSL→Windows junctions and bridged registration, the SessionStart hook, setup fresh vs maintenance, the daily-driver merge, and uninstall gates/ownership are the authoritative summary. This file is the detail behind them.

Out of scope here (specced elsewhere): `scripts/project.js`, `scripts/lib/discover.js`, `scripts/lib/clone.js`, `scripts/lib/presets.js` (see `skills/bootstrap-argus.md`); `scripts/lib/defensive-review-*.js` (see `skills/argus-defensive-review.md`).

## 0. Design principles (apply everywhere)

- **Two roots.** The **main clone** serves everything runtime: `dist/mcp-server/index.js`, `.argus.config`, `.argus/` indexes. The **daily-driver worktree** (`<main>/.claude/worktrees/<kebab>-daily-driver`, branch `daily-driver/<kebab>`) provides only `skills/` and `hooks/`. MCP registrations always point at the main clone; skill links and the hook always point at the worktree. Never copy `.argus.config` into the worktree.
- **Never hand-write client config**, with two sanctioned exceptions: (1) Claude's Windows-format target writes `claude_desktop_config.json` directly (wsl.exe bridge); (2) the SessionStart hook is upserted into a Claude home's `settings.json` (there is no `claude hooks add`). Everything else goes through the client's own `mcp add`/`mcp remove`.
- **Skip-and-warn, never fail the install** on per-client problems: a missing CLI or unparseable settings prints the exact manual command/JSON and continues.
- **Pure core + injected IO.** Decision logic (`planSetupSteps`, `classifyDailyDriverMerge`, `checkboxReducer`, `upsertSessionStartHook`, `pickInstaller`, uninstall planners) is pure and unit-tested; IO wrappers take an injected `io`/`deps`/`fs`.
- **Uninstall: read-only scan, one mutating module.** Only `scripts/lib/uninstall-execute.js` may change the machine; a test statically forbids mutating calls elsewhere.
- All scripts are ESM (`"type": "module"`), `// @ts-check` JSDoc-typed plain JS, Node built-ins only (no npm deps).
- "Yes" parsing differs by script: setup/fix-links/preflight accept any answer whose trimmed lowercase **starts with `y`**; uninstall accepts only `/^(y|yes)$/i` after trim.

## 1. Files to create

### 1.1 Scripts

| Path | Purpose |
|---|---|
| `scripts/setup.js` | `npm run setup`. Idempotent installer: fresh install or maintenance (§3). |
| `scripts/fix-links.js` | `npm run fix-links`. Quick relink of skills + hook + MCP for persisted targets (§4.6). |
| `scripts/uninstall.js` | `npm run uninstall` (`-- --dry-run`). Read-only scan, two `[y/N]` gates, executes via the executor (§6). |
| `scripts/setup-deps.js` | Python detection + install from `requirements.txt` (uv preferred, `.venv` fallback). Exports `checkPython`, `getPythonInstallInstructions`, `hasUv`, `pickInstaller`, `installPythonDeps`. No CLI entry. |
| `scripts/copy-assets.js` | Run by `npm run build` after `tsc`: copies runtime assets from `src/` to `dist/` (§10.1). |
| `scripts/eval-triggers.mjs` | `npm run eval:triggers`: skill trigger-accuracy eval runner (§10.2). Not installer, but lives in `scripts/`. |
| `scripts/lib/platform.js` | Platform detection, path conversion, `repoRoot()`, `mainRepoRoot()`, banner. |
| `scripts/lib/config.js` | `.argus.config` load/write (top-level fields and project entries). |
| `scripts/lib/clients.js` | Client-adapter registry: Claude/Codex/Copilot descriptors, MCP argv builders, register/unregister, Claude Desktop writer, WSL bridge (§2). |
| `scripts/lib/targets.js` | Install-target enumeration (local + cross-env), selection UI, persistence normalization (§3.6). |
| `scripts/lib/checkbox.js` | Pure checkbox reducer + raw-TTY driver (§3.7). |
| `scripts/lib/setup-plan.js` | Pure `planSetupSteps` (§3.1). |
| `scripts/lib/worktree.js` | Daily-driver create/detect/repair/fetch/merge + merge policy (§3.4). |
| `scripts/lib/build.js` | `npm install`, `npm run build`, background `build-index` (§3.8). |
| `scripts/lib/preflight.js` | HuggingFace reachability, git credential-helper check, `runPreflight` used by fix-links (§3.5). |
| `scripts/lib/linker.js` | Skill linking per platform, `canLinkSkillsToTarget`, `linkAll` (links + hook + MCP), superpowers-conflict detection (§4). |
| `scripts/lib/hooks.js` | SessionStart hook command builder, pure upsert/remove, settings.json IO (§5). |
| `scripts/lib/uninstall-catalog.js` | `REMOVED`/`OFFERS` data, `classifySkill`, the `ClientHome` and `Action` typedefs (§6.2). |
| `scripts/lib/uninstall-homes.js` | Read-only client-home discovery (§6.3). |
| `scripts/lib/uninstall-links.js` | Read-only skill-link scan + ownership classification (§6.4). |
| `scripts/lib/uninstall-mcp.js` | Read-only owned-MCP detection, TOML/JSON registration-text extraction (§6.5). |
| `scripts/lib/uninstall-hooks.js` | Read-only hook-removal planning (§6.6). |
| `scripts/lib/uninstall-plan.js` | Pure planner: self-contained check, offers, ordered plan, numbered-answer parser (§6.7). |
| `scripts/lib/uninstall-report.js` | Pure text: `describeAction`, `closingNotice`, `shellQuote`, banner/URL constants (§6.8). |
| `scripts/lib/uninstall-execute.js` | The ONLY mutating uninstall module: `executeActions`, `realDeps`, `makeRemoveLinkOnly` (§6.9). |
| `scripts/lib/worktree-cleanup.js` | Used by `project.js remove` (not the installer): removes `.argus/<project>/worktrees/*` safely (§10.3). |

Do **not** recreate: `scripts/verify-deps.js` (kuzu smoke test, dead), `scripts/python-ast-extractor.py` (dead), `install.sh` (a stub that prints "run npm install / npm run setup" and exits 1). The master prompt §11.7 lists these as dead code.

### 1.2 Hooks (vendored from superpowers v7.0.0; see `skills/superpowers.md`)

`hooks/session-start` (bash; emits SessionStart JSON — keys its output format on `CLAUDE_PLUGIN_ROOT` / `CURSOR_PLUGIN_ROOT` / `COPILOT_CLI` / `MUSE_PLUGIN_ROOT`, reads `skills/using-superpowers/SKILL.md` relative to its own dir), `hooks/hooks.json`, `hooks/run-hook.cmd`, `hooks/hooks-cursor.json`. Only `session-start` is used by the installer; the other three are kept for upstream fidelity. `hooks/hooks.json` content:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "shell": "bash",
            "async": false
          }
        ]
      }
    ]
  }
}
```

### 1.3 Repo-root files (exact content)

`package.json` scripts (relevant entries):

```json
"build": "tsc && node scripts/copy-assets.js",
"test": "vitest run",
"test:watch": "vitest",
"build-index": "node --max-old-space-size=4096 --import tsx/esm src/cli/build-index.ts",
"setup": "node scripts/setup.js",
"fix-links": "node scripts/fix-links.js",
"uninstall": "node scripts/uninstall.js",
"project": "node scripts/project.js",
"eval:triggers": "node scripts/eval-triggers.mjs"
```

`.mcp.json` (project-scoped registration used when a client opens this repo itself; relative paths):

```json
{
  "mcpServers": {
    "argus": {
      "command": "node",
      "args": ["dist/mcp-server/index.js"],
      "env": {
        "ARGUS_CONFIG": ".argus.config"
      }
    }
  }
}
```

`.gitattributes`:

```
# Force LF everywhere so WSL + Windows checkouts of the same clone agree.
# This overrides core.autocrlf — without it, autocrlf=true smudges CRLF into
# the working tree, which breaks shebang'd scripts under vitest and makes
# daily-driver worktrees look perpetually dirty with zero-content EOL diffs.
* text=auto eol=lf

# Windows cmd/batch scripts require CRLF to run reliably.
*.cmd text eol=crlf
*.bat text eol=crlf

# Binary assets — never touch line endings.
*.png binary
*.jpg binary
*.jpeg binary
*.gif binary
*.ico binary
*.woff binary
*.woff2 binary
*.pdf binary
*.zip binary
```

The `*.cmd` CRLF rule is why the hook command bypasses `run-hook.cmd` (§5).

`.gitignore` (verbatim; see §9 for its duplicates/oddities):

```
# Logs
logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*
lerna-debug.log*

node_modules
.DS_Store
dist
sdd/
graphql.config.yml
dist-ssr
coverage
*.local
.env
.prettierignore
# Editor directories and files
.vscode/*
!.vscode/extensions.json
.idea
*.suo
*.ntvs*
*.njsproj
*.sln
*.sw?
idea-types.json
.claude/settings.local.json
pr-description.md
/.claude/worktrees/

cache/
.argus.config
.argus.config.bak
argus-output.md
*.tsbuildinfo
/argus_output.md
.can-i-do-that.md
/.claire/worktrees/
/argus-output/
/presentation/

# Argus v2 artifacts
.argus
.argus/
.superpowers/
/presentation/

# Eval scratch: ad-hoc trigger/output-quality run outputs.
# Committed eval assets live in skills/<skill>/evals/ (specs, fixtures, results/).
skills/*/evals/runs/
.venv/*
tools/argus-dotnet-analyzer/src/ArgusAnalyzer/bin/*
tools/argus-dotnet-analyzer/src/ArgusAnalyzer/obj/*
tools/argus-dotnet-analyzer/tests/ArgusAnalyzer.Tests/bin/*
tools/argus-dotnet-analyzer/tests/ArgusAnalyzer.Tests/obj/*
tools/argus-dotnet-analyzer/tests/Fixtures/AnalyzerFixture/bin/*
tools/argus-dotnet-analyzer/tests/Fixtures/AnalyzerFixture/obj/*

# Project conventions are per-user onboarding output (like .argus.config and
# .argus/ indexes) — local by default. Only the README and the seeding template
# are shared. Pre-existing tracked docs (tcn.md, dataplatform.md,
# argus-panoptes.md) remain tracked until explicitly untracked.
project_conventions/*
!project_conventions/README.md
!project_conventions/_template.md

# Per-repo git worktrees live inside the repo, one dir per branch.
/.worktrees/
```

`requirements.txt` (the single source of truth for Python deps; `setup-deps.js` installs straight from it with `-r`):

```
# Argus-Panoptes Python runtime dependencies.
# Pinned versions are deliberately loose — Python ecosystem moves slowly enough
# that >= is sufficient for indexing target repos.
libcst>=1.0.0
sentence-transformers>=2.2.0
```

(Master prompt §11.6: nothing imports `libcst`; the rebuild drops it by default.)

`.argus.config.example` (original content; master prompt §11.8 says the rebuild must drop the four LLM-summary comment keys — `ARGUS_OLLAMA_URL`, `ARGUS_OLLAMA_MODEL`, `ANTHROPIC_API_KEY`, `ARGUS_ANTHROPIC_MODEL`):

```json
{
  "_comment": "Example .argus.config in the multi-project {projects} shape. Project entries are normally written by `node scripts/project.js add` (typically via the bootstrap-argus skill) — this file is illustrative, not something to hand-edit and copy in place. `dailyDriverWorktree`/`installTargets` are written by `npm run setup`.",
  "projects": {
    "tcn": {
      "repoRoot": "/path/to/your/tcn-repos",
      "repos": {
        "apollo-server": "/path/to/your/tcn-repos/tcn-apollo-server",
        "customer-app": "/path/to/your/tcn-repos/tcn-customer-application",
        "office-addin": "/path/to/your/tcn-repos/tcn-office-addin",
        "integrations": "/path/to/your/tcn-repos/tcn-integrations",
        "agreement-ingestion": "/path/to/your/tcn-repos/tcn-agreement-ingestion/tcn-agreement-ingestion",
        "libs": "/path/to/your/tcn-repos/tcn-libs",
        "ai": "/path/to/your/tcn-repos/tcn-ai",
        "utils": "/path/to/your/tcn-repos/tcn-utils",
        "platform": "/path/to/your/tcn-repos/tcn_platform",
        "tcn-agents": "/path/to/your/tcn-repos/tcn-agents"
      },
      "sampleCodePath": "/path/to/your/sample-code",
      "conventions": "project_conventions/tcn.md"
    }
  },
  "dailyDriverWorktree": "/path/to/your/argus-panoptes-daily-driver",
  "_comments": {
    "projects": "Add more projects (e.g. a second org's repos) as additional keys under `projects`, each with its own `repos`/`sampleCodePath`/`conventions` — see the bootstrap-argus skill (skills/bootstrap-argus/SKILL.md) for the guided flow, or `node scripts/project.js --help`.",
    "ARGUS_PROJECT": "Optional. Defaults which project graph/read MCP tools target when a call omits `project` and more than one project is configured.",
    "ARGUS_OLLAMA_URL": "Optional. Set env to http://localhost:11434 to use a local Ollama for entry-point summaries.",
    "ARGUS_OLLAMA_MODEL": "Optional. Defaults to llama3.2:3b.",
    "ANTHROPIC_API_KEY": "Optional. Fallback for summary generation when Ollama is not configured.",
    "ARGUS_ANTHROPIC_MODEL": "Optional. Defaults to claude-haiku-4-5-20251001."
  }
}
```

### 1.4 `.argus.config` shape written by the installer

```jsonc
{
  "projects": { /* untouched by setup; written by project.js */ },
  "dailyDriverWorktree": "<abs path to worktree>",
  "installTargets": [
    { "client": "claude"|"codex"|"copilot", "home": "<abs>", "pathFormat": "posix"|"windows",
      "wslHome"?: "<posix home inside WSL, only for Windows→WSL targets>", "local": true|false }
  ]
}
```

### 1.5 Docs that describe this behavior (keep in sync)

`CLAUDE.md` (Gotchas), `README.md`, `docs/manual-install.md` (step-by-step manual equivalent of every installer action, plus an Uninstalling section), `docs/argus/README.md`. `_can-i-do-that.md` contains no installer content.

## 2. Client adapter registry (`scripts/lib/clients.js`)

### 2.1 Constants and types

- `MCP_SERVER_NAME = 'argus'`.
- `InstallTarget = { client, home, pathFormat: 'windows'|'posix', wslHome?, local?, displayName?, envLabel? }`.
- `argusMcpPaths(mainRepoPath, pathFormat)` → `{ configPath, entryPoint }`: `main = normalizeSeparators(toPathFormat(mainRepoPath, fmt), fmt)`; `entryPoint = <main>/dist/mcp-server/index.js`, `configPath = <main>/.argus.config`, both separator-normalized for the format. Example: `/mnt/d/argus` with `windows` → `D:\argus\dist\mcp-server\index.js`.

### 2.2 Descriptor shape

```js
{ id, displayName, dirName, skillsSubdir: 'skills',
  localHome(homedir, env) -> string,
  registerMcp(ctx, io), unregisterMcp(ctx, io) }
// ctx = { target, mainRepoPath, paths: {configPath, entryPoint}, platform? }
// io  = { execSync, execFileSync, canExec, existsSync, mkdirSync, readFileSync, writeFileSync, log }
```

`CLIENTS = [claude, codex, copilot]` (this order drives enumeration order). `getClient(id)` finds by id. `makeConsoleIo()` returns the real io; `canExec(bin)` is a side-effect-free PATH probe: split `PATH` on `;` (win32) or `:`, true if `existsSync(join(dir, bin))` in any entry.

| id | displayName | dirName | localHome |
|---|---|---|---|
| `claude` | `Claude Code` | `.claude` | `join(home, '.claude')` (does NOT honor `CLAUDE_CONFIG_DIR`) |
| `codex` | `Codex` | `.codex` | `env.CODEX_HOME` if set and non-blank after trim, else `join(home, '.codex')` |
| `copilot` | `Copilot` | `.copilot` | `join(home, '.copilot')` |

### 2.3 Exact MCP argv

- `stdioMcpArgv(cli, paths)` (Codex and Copilot):
  - add: `[cli, 'mcp', 'add', 'argus', '--env', 'ARGUS_CONFIG=<configPath>', '--', 'node', '<entryPoint>']`
  - remove: `[cli, 'mcp', 'remove', 'argus']`
- `claudeCliMcpArgv(paths)`:
  - add: `['claude', 'mcp', 'add', '-s', 'user', '-e', 'ARGUS_CONFIG=<configPath>', '--', 'argus', 'node', '<entryPoint>']` (server name goes after `--`)
  - remove: `['claude', 'mcp', 'remove', '-s', 'user', 'argus']`
- `quoteArgv(argv)` = every token wrapped in `"…"` (no escaping), joined by spaces. CLI registration runs `io.execSync(quoteArgv(argv), { stdio: 'pipe' })`.

### 2.4 Registration flows

`runCliRegistration({cli, argv, io})`: run remove (ignore failure), then add. Success log: `  MCP registered: <cli> mcp add argus`. Add failure (skip-and-warn):

```
  ⚠ Could not register MCP with '<cli>' (is it on PATH?). Skills are linked; to add the server yourself run:
      <quoted add command>
```

`registerViaCli({displayName, cli, argv, target})`: if `target.local === false` (cross-env home we can't drive), print and return:

```
  ⚠ <displayName> home is in another environment — register MCP there yourself:
      <quoted add command>
```

otherwise `runCliRegistration`.

**Codex/Copilot `registerMcp`:** if `ctx.platform === 'wsl' && ctx.target.pathFormat === 'windows'` → `registerStdioBridged`; else `registerViaCli` with `stdioMcpArgv`.

**`registerStdioBridged` (WSL → Windows Codex/Copilot):**
1. `wslMain` = `mainRepoPath` if it starts with `/`, else `windowsToWslPath(mainRepoPath)` (`D:\x\y` → `/mnt/d/x/y`).
2. Payload = `buildWslArgusBashCommand(wslMain)`, exactly:
   `[ -s "$HOME/.nvm/nvm.sh" ] && . "$HOME/.nvm/nvm.sh"; ARGUS_CONFIG="<wslMain>/.argus.config" exec node "<wslMain>/dist/mcp-server/index.js"`
3. `winCli = '<cli>.exe'`. If `!io.canExec(winCli)`:
   ```
     ⚠ <displayName> is a Windows client but '<cli>.exe' isn't on PATH here — skipping MCP auto-registration.
       If you use <displayName> on Windows, register there: <cli>.exe mcp add argus -- wsl.exe bash -c '<argus wsl launcher>'
   ```
4. Else `execFileSync(winCli, ['mcp','remove','argus'], {stdio:'pipe'})` (ignore failure), then `execFileSync(winCli, ['mcp','add','argus','--','wsl.exe','bash','-c', payload], {stdio:'pipe'})`. Argv array, never a shell string, so the payload crosses interop verbatim with POSIX paths. Log `  MCP registered: <cli>.exe mcp add argus (wsl.exe bridge)`; on error `  ⚠ Could not register MCP with '<cli>.exe': <message>`. The `local` flag is not consulted on this path.

**Claude `registerMcp`:** if `target.pathFormat === 'windows'` → `upsertClaudeDesktopConfig` (on ANY platform, see §9); else `registerViaCli` with `claudeCliMcpArgv`.

**`upsertClaudeDesktopConfig`:**
- Path: strip a trailing `/.claude` or `\.claude` from `target.home`; if nothing was stripped, log `  Warning: could not derive claude_desktop_config.json path from <home>` and return. Else `<profile>/AppData/Roaming/Claude/claude_desktop_config.json`.
- Read+parse; on any failure (missing or unparseable) start from `{}`.
- `cfg.mcpServers ??= {}`; `cfg.mcpServers.argus = { command: 'wsl.exe', args: ['bash', '-c', <payload from wslMain>] }`.
- `mkdirSync(dir, {recursive:true})` if missing; write `JSON.stringify(cfg, null, 2) + '\n'`.
- Log `  MCP registered: claude_desktop_config.json argus (wsl.exe bridge)`.

**Unregister (present, unused by any script — only tests call it):**
- Codex/Copilot: WSL+windows → if `canExec('<cli>.exe')`, `execFileSync('<cli>.exe', ['mcp','remove','argus'])` (ignore errors); else nothing. Non-local → nothing. Local → exec quoted remove argv, ignore errors.
- Claude: windows → remove `mcpServers.argus` from desktop config if file exists and parses (preserving other servers), log `  MCP unregistered: claude_desktop_config.json argus`. Non-local → nothing. Else exec `claude mcp remove -s user argus`.

## 3. `scripts/setup.js`

### 3.1 Mode selection (`scripts/lib/setup-plan.js`, pure)

`planSetupSteps({ hasConfig, hasWorktree, worktreeBehind, worktreeDirty, staleProjects })`:

- `!hasConfig || !hasWorktree` → `{ mode: 'fresh', steps: ['deps','targets','superpowers-check','build-main','create-worktree','link','point-to-skill'] }`. For fresh the step list is **descriptive only**; `runFreshSetup` runs its own hardcoded sequence.
- Else → `{ mode: 'maintenance', steps: ['deps','build-main','merge','build-worktree','relink', ...staleProjects.map(n => 'rebuild-index:'+n)] }`. Authoritative: `runMaintenance` dispatches on it. `worktreeBehind`/`worktreeDirty` are accepted but not consulted (the merge classifies itself). Order is load-bearing: `merge` before `build-worktree` and `relink` so the worktree builds from merged source and newly merged skills get linked in the same run; the worktree's `npm install` lives in `build-worktree` (post-merge) so a merged `package.json` change is installed before compiling.

### 3.2 `main()` sequence

1. `rl = readline/promises.createInterface(stdin, stdout)`; `root = repoRoot()`; `platform = detectPlatform()`.
2. `printBanner('Setup')` (clears console, prints the 6-line colored ARGUS ASCII banner, then `\n  Setup\n`). Print `  Platform: <platform>` and `  Repo:     <root>`.
3. **Environment preflight (warn-only, never blocks).** Header `── Environment ──…`. Print each `checkGitCredentialHelper()` warning as `  ⚠ <w>`. `await checkHuggingFaceReachable()`: false → `  ⚠ HuggingFace not reachable — embedding model download may fail (a cached model may still work).`; true → `  git + HuggingFace reachable.`
4. Inside `try { … } finally { rl.close() }`:
   - `config = loadConfig()`; `hasConfig = config !== null`.
   - `mainBranch = getMainBranch(root)`; print `\n  Fetching origin...`; `fetched = fetchOrigin(root)`; if not, `  Warning: could not fetch origin — comparing against local refs only.` `mergeRef = fetched ? 'origin/<mainBranch>' : mainBranch`.
   - `existing = detectExistingWorktree(root, { mainBranch: mergeRef })`.
   - **Orphaned worktree** (`existing.orphaned`): print
     ```
       Found a daily-driver worktree that git no longer tracks:
         <worktreePath>
       (The folder and your branch survived, but git lost its worktree registration.)
     ```
     prompt `  Re-register it in place? Your branch and any uncommitted work are preserved. (y/n): `. Decline → print `\n  Cannot continue with an orphaned worktree. Re-register it manually:` / `    git worktree repair "<path>"` / `  or recreate it (discards uncommitted worktree changes):` / `    rm -rf "<path>" && git worktree add "<path>" "<branch>"`, return. Accept → `repairOrphanedWorktree`, print `  Worktree re-registered.`, re-detect; if still null throw `Repair reported success but the worktree is still not detected.`
   - `hasWorktree = existing !== null`; `worktreeDirty = isWorktreeDirty(path)` (`git status --porcelain --untracked-files=no` non-empty; error → false); `staleProjects = computeStaleProjects(root, config)`: for each project name (or `['tcn']` for a legacy flat config), stale iff `<root>/.argus/<name>/meta.json` is missing.
   - `{mode, steps} = planSetupSteps(...)`; print `\n  Mode: fresh install` or `\n  Mode: maintenance (bring current)`; run fresh or maintenance.
5. `main().catch(err => { console.error('\nSetup failed:', err.message ?? err); process.exit(1) })`.

Section headers are `\n── <Title> ` followed by a run of `─` (box-drawing) to roughly column 57.

### 3.3 Fresh flow (`runFreshSetup`) — tool only, no project data

1. `── Python Dependencies ──` → `installPythonDeps()` (exits the process with 1 if Python is missing or install fails; §3.9).
2. `── Install Targets ──`. `candidates = enumerateTargets(platform)`. None →
   ```
     No agent CLIs detected (no ~/.claude, ~/.codex, or ~/.copilot).
     Install one and re-run setup.
   ```
   and return. `targets = targetsMode(cfg?.installTargets) === 'reuse' ? normalizeTargets(persisted) : await selectTargets(rl, candidates, { isTTY: process.stdin.isTTY === true, checkbox: promptCheckbox })`. Empty → `  No targets selected — nothing to install for.` and return. Persist `writeTopLevelConfig({ installTargets: targets.map(toPersistedTarget) })`.
3. **Superpowers conflict check** for every target home (`detectSuperpowersConflicts`, §4.5). If any:
   ```
   ── Superpowers Conflict Detected ──…
     Argus-Panoptes includes all superpowers skills (currently v7.0.0).
     Running both will cause duplicate skills, hooks, and agents.

     <target home>:
       • <conflict>
   …
     To fix: disable the superpowers plugin and re-run setup.
   ```
   Prompt `\n  Continue anyway? (y/n): `; not y → `\n  Setup aborted.\n`, return.
4. `── Building ──` → `ensureNodeModules(root)`, `tscBuild(root)`.
5. `── Daily-Driver Worktree ──`: `getMainBranch`, `  Fetching origin...`, `fetchOrigin` (second fetch of the run), same warning, `mergeRef`, `detectExistingWorktree(root, {mainBranch: mergeRef})`.
   - Existing: print `\n  Found existing daily-driver: <kebab>-daily-driver`, `  Branch: <branch>`, `  Commits behind <mergeRef>: <n>`; if `n > 0` run `runDailyDriverMerge(path, mergeRef, rl)`.
   - Else prompt exactly `\n  Choose a handle for your daily-driver worktree\n  (your coworkers may see this — e.g. in branch listings).\n  First and last name works well: `. Blank → `  No name entered — cannot continue without a daily-driver.`, return. `kebab = toKebabCase(name)`; confirm `\n  Worktree: "<kebab>-daily-driver" on branch "daily-driver/<kebab>". Continue? (y/n): `; not y → `  Setup aborted.\n`, return. `createWorktree(root, kebab)`, print `\n  Setting up worktree...`, `ensureNodeModules(wt)`, `tscBuild(wt)`.
   - Record: if `loadConfig()?.dailyDriverWorktree !== worktreePath`, `writeTopLevelConfig({ dailyDriverWorktree })` and print `  Recorded daily-driver worktree in .argus.config: <path>`. This field's presence is part of what flips future runs to maintenance (with the worktree dir).
6. `── Linking ──` → `linkTargets(targets, platform, worktreePath)`: for each target print `\n  Target: <labelFor(t)>  (<pathFormat>)` and call `linkAll({ claudeHomePath: t.home, worktreePath, mainRepoPath: mainRepoRoot(), platform, pathFormat, wslHome, client, local })`; then print `codexMultiAgentReminder(targets)` if non-null.
7. `printClosing()`.

### 3.4 Maintenance flow (`runMaintenance`)

Iterates `steps`:

- `deps` → `── Installing dependencies ──`, `ensureNodeModules(root)` (npm only — Python deps are NOT re-run).
- `build-main` → `── Building main ──`, `tscBuild(root)`.
- `merge` → `── Daily-Driver Merge ──`, `runDailyDriverMerge(worktreePath, mergeRef, rl)`.
- `build-worktree` → `── Building daily-driver worktree ──`, `ensureNodeModules(wt)`, `tscBuild(wt)`.
- `relink` → `── Relinking skills + MCP config ──`. `targets = normalizeTargets(config.installTargets)`. For each `newlyDetected(enumerateTargets(platform), config.installTargets)` prompt `\n  Found <labelFor(t)>, not yet installed for. Add it? (y/n): `; y → push. `linkTargets(...)`. Then re-persist `writeTopLevelConfig({ installTargets: targets.map(toPersistedTarget) })`. Declined clients are not remembered (re-offered every run).
- `rebuild-index:<name>` → prompt `\n  '<name>' index looks stale (.argus/<name>/meta.json missing). Rebuild it? (y/n): `; y → `handle = buildIndexBackground(root, ['--project', name])`, `streamBuildOutput(handle)`, `await handle.wait()`, print `  '<name>' index rebuilt and synced.`; on rejection `\n  Warning: index build failed (<message>)`.

Then `printClosing()`. Maintenance never runs the superpowers-conflict check or Python deps.

### 3.5 Daily-driver worktree (`scripts/lib/worktree.js`)

- `toKebabCase(s)`: trim, lowercase, drop `[^a-z0-9\s-]`, whitespace runs → `-`, collapse `-+`, strip leading/trailing `-`.
- `getMainBranch(dir)`: `git symbolic-ref refs/remotes/origin/HEAD` minus `refs/remotes/origin/`; else `main` if `git rev-parse --verify main`; else `master`; else `git rev-parse --abbrev-ref HEAD`.
- `fetchOrigin(dir)`: `git fetch origin` with `timeout: 30_000`, `stdio: 'pipe'`; returns boolean, never throws.
- `createWorktree(repoDir, kebab)`: path `<repoDir>/.claude/worktrees/<kebab>-daily-driver`, branch `daily-driver/<kebab>`. Throws `Worktree already exists at <path>` if the dir exists. Prints `\n  Creating worktree: <name> on branch <branch>`. If the branch exists (`git rev-parse --verify "<branch>"`): print `  Branch <branch> already exists — reusing it` and `git worktree add "<path>" "<branch>"`; else `git worktree add "<path>" -b "<branch>"`. No npm/build, no config copy.
- `detectExistingWorktree(repoDir, {mainBranch})`:
  1. `<repoDir>/.claude/worktrees` missing → null. First directory entry ending in `-daily-driver` (readdir order) else null. `kebab` = name minus suffix; `branch = daily-driver/<kebab>`.
  2. `git worktree list --porcelain`; compare each `worktree <path>` line to the dir using `normalizePathForCompare` (realpath if it exists locally, then `\`→`/`, lowercase).
  3. Not registered: if another listed path ends with the dir name (path-format mismatch), throw
     ```
     Daily-driver worktree found but registered under a different path format.
       Registered: <registered>
       Expected:   <path>

       <hint>
     ```
     hint = `This worktree was set up from WSL. Run this command from WSL instead.` if registered matches `^/mnt/[a-z]/`, else `This worktree was set up from Windows. Run this command from Windows instead.`
  4. Not registered and the dir has a `.git` FILE matching `^gitdir:\s*(.+)$` → orphaned: `{…, commitsBehind: git rev-list --count "<branch>..<ref>" (0 on error), orphaned: true}`. Otherwise (junk dir) → null.
  5. Registered → `commitsBehind` via the same rev-list (`ref` = `mainBranch` option or `HEAD`).
- `repairOrphanedWorktree(repoDir, wt, branch)`: if `<repoDir>/.git/worktrees/<basename>` is missing, create it with `HEAD` = `ref: refs/heads/<branch>\n`, `commondir` = `../..\n`, `gitdir` = `<wt>/.git\n`. Best-effort `git worktree repair "<wt>"`, then `git reset -q HEAD` in the worktree (rebuilds the index without touching files). Verify registration; else throw `Failed to re-register the worktree at <wt>.\n  Repair manually:\n    rm -rf "<wt>" && git worktree add "<wt>" "<branch>"`.
- `mergeMain(wt, ref)`: `git merge "<ref>" --no-edit`. On failure, if `git status --porcelain` contains `UU`/`AA`/`DD` → `git merge --abort`, `{success:false, conflicted:true}`; else best-effort abort, `{success:false, conflicted:false}`.
- **Merge policy (pure)** `classifyDailyDriverMerge({commitsBehind, dirty, conflicted})`: `commitsBehind <= 0` → `up-to-date`; dirty → `dirty-stop` (dirty wins over conflicted); conflicted → `conflict`; else `clean`. Never auto-resolve, never `-X ours/theirs`.
- `runDailyDriverMerge(wt, ref, rl)`:
  1. `commitsBehind = git rev-list --count "HEAD..<ref>"` in the worktree (0 on error). Up-to-date → `  Daily-driver is up to date.`
  2. Dirty = `git status --porcelain --untracked-files=no` non-empty (error → clean). Dirty → `  Daily-driver has uncommitted changes — commit or stash, then re-run. Skipping merge.` (tree untouched).
  3. `mergeMain`. Success → `  Merged <ref> into daily-driver.` → `clean`.
  4. Conflicted → extra best-effort `git merge --abort`, print `  Merge conflicts — aborted (resolve later).` and `  To do it by hand: cd <wt> && git merge <ref>`, then `await rl.question('  Press Enter to continue setup without the merge... ')` → `conflict`.
  5. Other failure → `  Failed to merge <ref> into daily-driver — skipping. Check the worktree manually.` → `error`.

### 3.6 Install targets (`scripts/lib/targets.js`)

- `localPathFormat(platform)`: `windows` → `'windows'`, else `'posix'`.
- `discoverForeignProfile(platform, {execSync})` (any failure → null):
  - `wsl`: `cmd.exe /C "echo %USERPROFILE%"` (trim, strip `\r`) → `wslpath "<p>"` → `{ profileDir: <posix /mnt/c/Users/x>, pathFormat: 'windows' }`.
  - `windows`: distro = first line of `wsl.exe -l -q` decoded as UTF-16LE with `\0` stripped; `wslHome` = `wsl.exe echo $HOME` (trim, strip `\r`) → `{ profileDir: '\\\\wsl$\\<distro><wslHome>', pathFormat: 'posix', wslHome }`.
  - mac/linux → null.
- `enumerateTargets(platform, {existsSync, homedir, env, foreign})`: for each client in registry order: if `localHome` exists push `{client, home, pathFormat: local fmt, local: true, displayName, envLabel: 'this environment'}`; if foreign and `join(profileDir, dirName)` exists push `{…, pathFormat: foreign.pathFormat, local: false, envLabel: 'Windows'|'WSL', wslHome?}`. Detection is target-based: home dir exists, regardless of whether the CLI is on PATH.
- `labelFor(t)` = `<displayName ?? client>  (<home>)` + (`  [<envLabel>]` when `local === false`).
- `normalizeTarget(raw)`: `home = raw.home ?? raw.claudeHome`; `client = raw.client ?? 'claude'`; keep `pathFormat`; copy `wslHome` if set; `local` = boolean `raw.local` if present else `!raw.wslHome`. `normalizeTargets(list)` maps arrays, else `[]`.
- `targetsMode(persisted)`: non-empty array → `'reuse'`, else `'detect'`.
- `newlyDetected(candidates, persisted)`: candidates whose `client::home` key is not in the normalized persisted set.
- `selectTargets(rl, candidates, {isTTY, checkbox})`: empty → `[]`. TTY: `rl.pause()`, `checkbox(labels, all true)`, map indices, `rl.resume()` in finally. Non-TTY prompt:
  ```

    Install argus for which target(s)?
      1) <label>
      2) <label>
    [comma-separated numbers, or Enter for all]: 
  ```
  `parseNumberedSelection`: `''` or `all` → all; else split on `,`, `parseInt`, keep in-range 1-based, dedupe, sort; garbage/out-of-range silently ignored.
- `toPersistedTarget(t)` → `{client, home, pathFormat, wslHome?, local?}` (drops displayName/envLabel).
- `codexMultiAgentReminder(targets)`: null unless a codex target, else:
  ```
    Note: Codex sub-agent dispatch (argus flow-tracer/debugger/designer) needs multi-agent enabled.
          Add this to ~/.codex/config.toml:
              [features]
              multi_agent = true
  ```
  Setup never edits the TOML.

### 3.7 Checkbox UI (`scripts/lib/checkbox.js`)

- State `{items, cursor: 0, checked: items.map((_, i) => preChecked[i] ?? true), done: false, cancelled: false}`.
- `checkboxReducer(state, action)`: no-op once `done` or `cancelled`. `up`/`down` wrap modulo n; `space` toggles at cursor; `all`/`none` set every box; `enter` → done; `cancel` → cancelled.
- Keys: `up`/`k` → up, `down`/`j` → down, `space`, `a` → all, `n` → none, `return`/`enter` → enter, `escape` or Ctrl-C → cancel; others ignored.
- `selectedIndices(state)`; `checkboxOutcome(state)` → `{cancelled: true, indices: []}` or `{cancelled: false, indices}` (cancel is distinct from an empty selection).
- `promptCheckboxOutcome(items, preChecked)`: `readline.emitKeypressEvents(stdin)`, set raw mode, render, listen `keypress`, `stdin.resume()`. Render: hint line `  (↑/↓ move · space toggle · a all · n none · enter confirm)` then per item `  <'>' or ' '> <'[x]' or '[ ]'> <label>`; re-render moves the cursor up the previous line count (`\x1b[<n>A`) and clears to end (`\x1b[0J`). On done/cancel: remove listener, restore previous raw mode, `stdin.pause()`, resolve.
- `promptCheckbox` = `promptCheckboxOutcome(...).then(o => o.indices)` — setup maps cancel to an empty selection.

### 3.8 Build helpers (`scripts/lib/build.js`)

- `ensureNodeModules(cwd)`: print `\n  Installing npm dependencies...`; `execSync('npm install', {cwd, stdio: 'inherit'})` unconditionally.
- `tscBuild(cwd)`: print `\n  Building TypeScript...`; `execSync('npm run build', {cwd, stdio: 'inherit'})`.
- `buildIndexBackground(cwd, extraArgs)`: print `\n  Starting index build in background...`; `spawn('npm', ['run','build-index', ...(extra.length ? ['--', ...extra] : [])], {cwd, stdio: ['ignore','pipe','pipe'], shell: true})`; buffer stdout/stderr chunks; `wait()` resolves on exit code 0 else rejects `Index build exited with code <code>`.
- `streamBuildOutput(handle)`: print `\n── Index Build Output ──…`, drop the buffering listeners, write the buffer, then pipe live stdout/stderr.

### 3.9 Python deps (`scripts/setup-deps.js`)

- `REQUIREMENTS_PATH = <scriptDir>/../requirements.txt`.
- `checkPython()`: try `python3 --version` then `python --version`; match `Python (\d+\.\d+(?:\.\d+)?)` → `{found: true, version, bin}`; else `{found: false, version: null, bin: null}`.
- `getPythonInstallInstructions(process.platform)`: `darwin` → Homebrew text (`brew install python3` + python.org); `linux` → apt/yum/pacman lines + python.org; `win32` → python.org ("check \"Add Python to PATH\"") or `winget install Python.Python.3.11`; default → `Install Python 3.10+ from https://www.python.org/downloads/`.
- `hasUv()`: `uv --version` succeeds.
- `pickInstaller({hasUv})` (pure): `['uv','pip','install','-r',REQ]` (never `--system`) or `['python3','-m','pip','install','-r',REQ]`.
- `installPythonDeps()`:
  1. Not found → stderr `ERROR: Python 3 is not installed.`, blank, instructions, blank, `process.exit(1)`.
  2. Print `✓ Python <v> detected.` and `Installing required pip packages from requirements.txt...`.
  3. `VIRTUAL_ENV` set → install into the active venv. Else use `<repo>/.venv` (PEP 668 workaround): venv python = `.venv\Scripts\python.exe` (win32) or `.venv/bin/python3`; if missing print `No active venv — creating .venv in project root...` and run `uv venv <dir>` (uv) or `<bin> -m venv <dir>`; failure → `ERROR: Failed to create .venv.`, exit 1. After install print `  Run "<.venv\Scripts\activate | source .venv/bin/activate>" before using build-index.`
  4. Install: uv → `uv pip install [--python <venvPython>] -r REQ` with env bridging `SSL_CERT_FILE = REQUESTS_CA_BUNDLE` when only the latter is set; failure → `ERROR: Failed to install pip packages via uv.`, exit 1. pip → ensure pip via `"<python>" -m pip --version`, else print `pip not found — bootstrapping with ensurepip...` and `<python> -m ensurepip --upgrade` (failure → `ERROR: Could not bootstrap pip via ensurepip.`, exit 1); then `<python> -m pip install -r REQ` (failure → `ERROR: Failed to install pip packages.`, exit 1). `<python>` is the venv python or the discovered bin (argv[0] of `pickInstaller` is swapped).
  5. Print `✓ Python dependencies installed.`

### 3.10 Preflight (`scripts/lib/preflight.js`)

- `checkHuggingFaceReachable(fetchImpl = fetch)`: `HEAD https://huggingface.co` with a 3000 ms AbortController; true on any resolve, false on any rejection; never throws. Tests inject a fake.
- `checkGitCredentialHelper()` → warnings: `git --version` fails → `git was not found on PATH — cloning/fetching repos will fail.` (return early). `git config --get credential.helper` empty/failing and none of `git-credential-manager`, `git-credential-manager-core`, `git-credential-osxkeychain`, `git-credential-libsecret`, `git-credential-wincred`, `git-credential-cache` found (`where` on win32, `command -v` otherwise) → `No git credential helper configured — you may be prompted for credentials on every git operation. Consider: git config --global credential.helper <helper>.`
- `runPreflight(rl, {requireConfig = true, checkNetwork = true, fetchImpl})` (fix-links only):
  1. No config: throw `No .argus.config found. Run \`npm run setup\` to configure.` if required, else warn `No .argus.config found — config-dependent features unavailable`.
  2. For EVERY project (or the legacy flat shape), warn `Repo path unreachable (project <name>): <key> → <path>` (no `(project …)` label for legacy).
  3. Append git warnings; if `checkNetwork`, append `huggingface.co was not reachable — the embed phase downloads sentence-transformers models from there on first use. If you already have models cached, this is safe to ignore.` when unreachable.
  4. Worktree: `detectExistingWorktree(root, {mainBranch})` (unfetched local branch name). Orphan → same three lines + same prompt as setup; decline → throw the same manual instructions as one Error; accept → repair, `  Worktree re-registered.`, re-detect (null → throw).
  5. None: print `\n  No daily-driver worktree found.`, prompt `  Choose a handle for your daily-driver worktree\n  (your coworkers may see this — in branch listings).\n  First and last name works well: ` (note: no "e.g." here). Blank → throw `No name entered — cannot continue without a daily-driver worktree.` Confirm `\n  Your worktree will be "<kebab>-daily-driver" on branch "daily-driver/<kebab>". Continue? (y/n): `; no → throw `Cancelled worktree creation.` Create, print `\n  Installing dependencies and building worktree...`, npm install + build. Does NOT record `dailyDriverWorktree`.
  6. Return `{config, worktreePath, mainBranch, warnings}`.

### 3.11 Platform/config helpers

`scripts/lib/platform.js`:
- `detectPlatform()`: `darwin` → `mac`; `win32` → `windows`; `linux` → `wsl` if `/proc/version` contains `microsoft` (case-insensitive) else `linux`; anything else → `linux`.
- `windowsToWsl(p)`: `/mnt/<lower drive><rest with / >` (assumes `X:` prefix). `wslToWindows(p)`: `^/mnt/([a-z])(/.*)?$`i → `<UPPER>:<rest with \>`; unchanged otherwise.
- `toPathFormat(p, fmt)`: windows → convert `/mnt/<d>/…`; posix → convert `X:\`/`X:/`; else unchanged. `normalizeSeparators(p, fmt)`: posix `\`→`/`, windows `/`→`\`.
- `isWindowsPath(p)`: `^[A-Za-z]:[\\/]` or `^/mnt/[a-z]/`i.
- `repoRoot()`: walk up from the script dir to the first dir with `package.json` (throw `Could not find repo root (no package.json found in parent directories)`). This is the checkout the script runs from (may be the worktree).
- `mainRepoRoot()`: `git rev-parse --git-common-dir` resolved against `repoRoot()`; if it ends in `.git` return its parent; else `repoRoot()`.

`scripts/lib/config.js` (config path = `<repoRoot()>/.argus.config`; every write is `JSON.stringify(cfg, null, 2) + '\n'`):
- `loadConfig(pathOverride?)`: missing → null; valid iff `projects` is a non-array object OR legacy (`repoRoot` string + `repos` object); else print `  Warning: .argus.config has unexpected structure. Starting fresh.\n` → null; JSON error → `  Warning: .argus.config is not valid JSON. Starting fresh.\n` → null.
- `readProjectRepos(cfg, name = 'tcn')`: project's repos, else legacy flat repos; values normalized from string or `{path}` to plain strings.
- `detectMissingRepos(repos)` / `validateRepos(repos)` → existence checks via `resolve(p)`.
- `writeConfig(rootDir, repos, opts)` (project.js): merge; replace the named project (default `tcn`) with `{repoRoot, repos: {k: {path}}}` keeping `sampleCodePath`/`conventions` from opts or the existing entry; preserve other projects and top-level `dailyDriverWorktree`/`installTargets`.
- `removeProject(name)`: `{removed:false}` if absent; else drop it, keep the rest (writes `{projects:{}}` when last).
- `writeTopLevelConfig({dailyDriverWorktree?, installTargets?})` (setup's only writer): read existing (corrupt → `{}`); `projects` = existing `projects` verbatim; or a LEGACY flat config migrated into `projects.tcn` (`repos` as `{path}`, plus `repoRoot`/`sampleCodePath`/`conventions` when strings); or `{}`. Top-level fields from opts, else preserved. Never fabricates a project.

### 3.12 Closing output (`printClosing`)

Count projects (`projects` keys, or 1 for a non-empty legacy `repos`). If > 0: `\n  Setup complete! Restart your agent (Claude Code / Codex / Copilot) to use.\n`. Otherwise print this banner (bold yellow `\x1b[1;33m` rules/title, bold cyan `\x1b[1;36m` prompts, reset `\x1b[0m`; rule = `═` × 68):

```
════════════════════════════════════════════════════════════════════
  ✅  SETUP IS DONE — BUT ARGUS IS NOT USABLE YET
════════════════════════════════════════════════════════════════════

  The Argus tool is installed, but no project is configured.
  Argus can't index or trace anything until you add one.

  Do these two things:

    1. Restart your agent — Claude Code / Codex / Copilot (the MCP server
       loads at startup) for whichever client(s) you installed for.

    2. In the new session, paste this prompt:

         hey argus, let's add a new project

       The bootstrap-argus skill takes it from there: it discovers your
       repos, clones them, registers the project, builds its index, and
       seeds a conventions doc.

  Working with the standard TCN repo set? Paste this instead:

         hey argus, let's add the tcn project (use the tcn preset)
```

## 4. Linker (`scripts/lib/linker.js`) and fix-links

`LinkerOpts = { claudeHomePath, worktreePath, mainRepoPath, platform, pathFormat, wslHome?, client?, local? }`. `claudeHomePath` is the target home for any client (historical name).

### 4.1 `canLinkSkillsToTarget({platform, pathFormat, worktreePath})`

`platform === 'wsl' && pathFormat === 'windows'` → `isWindowsPath(worktreePath)` (worktree must be on `/mnt/<drive>/`; NTFS junctions cannot reach WSL ext4, and `mklink /J` would resolve a leading-slash source drive-relative into a dead `C:\home\…` junction). Every other combination → true.

### 4.2 Which skills get linked

`discoverSkills(worktreePath)`: every **directory** directly under `<worktree>/skills/` (no filtering — all argus, superpowers, product, backlog, PR, eval, etc. skills). Source is always the daily-driver worktree; destination `<home>/skills/<name>`.

### 4.3 `linkSkills(opts)`

1. If `!canLinkSkillsToTarget(opts)`, print and return (no links):
   ```
     Skipping skill links for this Windows target — worktree is on the WSL filesystem.
       Worktree: <worktreePath>
       NTFS junctions can only target a Windows drive (/mnt/<drive>/...), not WSL ext4,
       so linking here would create broken junctions the editor silently ignores.
       → Open the repo in a Remote-WSL editor window (it reads the WSL-side
         ~/.<client>/skills links directly), or relocate the daily-driver worktree
         onto a Windows drive under /mnt/<drive>/.
   ```
2. Print `  Linking <N> skills...`. If no `wslHome`, `mkdirSync(<home>/skills, {recursive:true})`.
3. Per skill: `formattedSource = normalizeSeparators(toPathFormat(source, fmt), fmt)`. `lstat(target)`:
   - Symlink: normalize `readlinkSync` result to the same format and compare. Equal: on WSL→Windows run `cmd.exe /C fsutil reparsepoint query "<win path>"` and require tag `0xa0000003` (real NTFS junction) → `  <name>: already linked (junction) — skipping`; otherwise `  <name>: LX reparse point detected — recreating as NTFS junction` + `forceRemoveLink`. Other platforms: `  <name>: already linked — skipping`. Not equal → `forceRemoveLink`.
   - Not a symlink (a real dir/file, e.g. a kept copy from uninstall or a user folder of the same name) → `forceRemoveLink` (recursive delete; §9).
   - lstat throws → `forceRemoveLink` (harmless when absent).
   - Then `createSkillLink` if needed.

### 4.4 Link creation per platform (`createSkillLink`)

| Case | Mechanism |
|---|---|
| `platform==='windows' && pathFormat==='posix' && wslHome` (Windows → WSL) | `createSymlinkViaWsl(toPathFormat(src,'posix'), '<wslHome>/.claude/skills/<name>')`: `wsl.exe mkdir -p "<dir>"`; if `wsl.exe test -L` succeeds and `wsl.exe readlink` equals source → `  <name>: already linked — skipping`, else `wsl.exe rm`; if not a link but `wsl.exe test -d` → `wsl.exe rm -rf`; then `wsl.exe ln -s "<src>" "<target>"` → `  <name>: linked (via wsl)`. Always `.claude`, whatever the client (§9). |
| `platform==='wsl' && pathFormat==='windows'` (WSL → Windows) | Link path `wslToWindows(target)`, target `normalizeSeparators(toPathFormat(src,'windows'),'windows')`. If the target isn't drive-rooted (`^[A-Za-z]:[\\/]`) → `  <name>: SKIPPED — source is not on a Windows drive (<src>); cannot create a junction.` Else `cmd.exe /C mklink /J "<link>" "<target>"` (link first) → `  <name>: linked (junction via cmd.exe)`; failure prints `  <name>: FAILED — junction creation failed`, `    Ensure you have permission to create junctions.`, `    (Developer Mode or elevated prompt may be required.)`, `    Error: <msg>`. Never Node `symlinkSync` here: from WSL it makes LX reparse points (`0xa000001d`) Windows can't follow. |
| Native (mac/linux/WSL posix, native Windows) | `symlinkSync(formattedSource, target, pathFormat==='windows' ? 'junction' : 'dir')` → `  <name>: linked`. |

### 4.5 `forceRemoveLink(target)` and `detectSuperpowersConflicts(home)`

`forceRemoveLink`: `rmSync(target, {recursive: true, force: true})` (errors ignored), then:
- WSL + Windows path: `cmd.exe /C rmdir "<win>"` then `cmd.exe /C del /f /q "<win>"` (each error ignored).
- Native Windows: `cmd.exe /C "dir /a "<target>".." 2>nul`; if it throws or the output lacks the basename → return. Else `fsutil reparsepoint delete "<target>"`, `cmd.exe /C rmdir "<target>"`, `cmd.exe /C del /f /q "<target>"` (each error ignored).
- Posix: nothing more.

`detectSuperpowersConflicts(home)` → strings: `enabledPlugins["<key>"] in <settings.json>` for each truthy `enabledPlugins` key containing `superpowers` (case-insensitive) in `<home>/settings.json`; and `skill directory: <home>/skills/superpowers` if that path lstat-exists. Read errors ignored. Run for every target, any client.

### 4.6 `linkAll(opts)`

1. `linkSkills(opts)`.
2. `clientId = opts.client ?? 'claude'`. If `claude` AND `canLinkSkillsToTarget(opts)` → `registerSessionStartHook({home, worktreePath, pathFormat}, makeConsoleIo())` (§5). Codex/Copilot never get a hook.
3. Unknown client → `  Warning: unknown client '<id>' — skipping MCP registration.` and return.
4. `paths = argusMcpPaths(mainRepoPath, pathFormat)`; `descriptor.registerMcp({target: {client, home, pathFormat, wslHome, local}, mainRepoPath, paths, platform}, makeConsoleIo())`.

### 4.7 `scripts/fix-links.js`

1. `printBanner('Fix Links')`, `  Platform: <p>`, `── Preflight ──`.
2. `runPreflight(rl, {requireConfig: false, checkNetwork: false})` (may interactively create a worktree, §3.10). Print warnings as `  ⚠ <w>`.
3. `── Linking ──`. Targets: persisted (`targetsMode === 'reuse'` → `normalizeTargets`) else `enumerateTargets(platform)` — all detected homes including cross-env, no menu, no new-client offer, not persisted.
4. Same per-target loop as setup's `linkTargets` (with `mainRepoPath: mainRepoRoot()`), Codex reminder, then `\n  Links repaired! Restart your agent (Claude Code / Codex / Copilot) to use.\n`; `rl.close()`.
5. Errors: `\nFix-links failed:` + message, exit 1.

## 5. SessionStart hook (`scripts/lib/hooks.js`)

- `SESSION_START_MATCHER = 'startup|clear|compact'`. Recognition tail `COMMAND_TAIL = '/hooks/session-start"'`.
- `sessionStartCommand(worktreePath, pathFormat)`: `root = toPathFormat(worktreePath, pathFormat)` with all `\` → `/` (forward slashes even on Windows, since the command runs under Git Bash). Returns exactly:
  `CLAUDE_PLUGIN_ROOT="<root>" bash "<root>/hooks/session-start"`
  Examples: `/w` posix → `CLAUDE_PLUGIN_ROOT="/w" bash "/w/hooks/session-start"`; `/mnt/c/x` windows → root `C:/x`.
- Why this shape: `CLAUDE_PLUGIN_ROOT` is set inline because Claude Code only exports it for plugin hooks and the script picks its output format from it; it points at the **worktree** (the script reads `skills/using-superpowers/SKILL.md` relative to itself, so it must follow the tree the skills link from); `bash …/session-start` directly instead of `run-hook.cmd`, because `shell: "bash"` already gives bash and `.gitattributes` gives `*.cmd` CRLF, which breaks the polyglot's bash half.
- `isArgusSessionStartHook(h)`: `h.command` is a string starting `CLAUDE_PLUGIN_ROOT=` and ending `/hooks/session-start"` (any path).
- `buildSessionStartGroup(cmd)` = `{ matcher: 'startup|clear|compact', hooks: [{ type: 'command', command: cmd, shell: 'bash', async: false }] }`.
- `removeSessionStartHook(settings)` (pure): non-object → `{}`. No `hooks.SessionStart` array → unchanged. Filter our hooks out of every group (groups without a `hooks` array kept as-is); groups left empty are dropped; empty `SessionStart` and then empty `hooks` are deleted. Returns `{settings, changed}`; unchanged input is returned by identity.
- `upsertSessionStartHook(settings, cmd)` (pure): if exactly one argus hook exists across all groups with `command === cmd` and `shell === 'bash'` → `{changed: false}`. Else strip all argus hooks and append one new group to `hooks.SessionStart` (foreign groups and all other keys preserved; a stale path is replaced rather than duplicated).
- `registerSessionStartHook({home, worktreePath, pathFormat}, io)` → boolean: settings path `<home>/settings.json`; missing → `{}`; unparseable → never overwrite, print
  ```
    ⚠ <settingsPath> is not valid JSON — add the SessionStart hook yourself:
      {"hooks":{"SessionStart":[<group>]}}
  ```
  return false. Unchanged → `  SessionStart hook: already registered — skipping` (true). Else write `JSON.stringify(s, null, 2) + '\n'` (plain write, no backup) → `  SessionStart hook registered: <settingsPath> → <worktreePath>/hooks/session-start`; write error → `  ⚠ Could not write <settingsPath>: <msg>` (false).
- `unregisterSessionStartHook({home}, io)`: silent no-op when missing/unparseable/unchanged; else write and log `  SessionStart hook unregistered: <settingsPath>`. (Not used by any script; uninstall plans its own `writeJson`.)

## 6. Uninstall

### 6.1 Overview and order (`scripts/uninstall.js`)

`scripts/uninstall.js` imports only read-only `node:fs` functions (`existsSync, lstatSync, readdirSync, readFileSync, readlinkSync, realpathSync`), no `child_process`, nothing from `linker.js`, and reaches mutation only through `import { executeActions, realDeps } from './lib/uninstall-execute.js'` called as `executeActions(plan.actions, realDeps(), …)`. It runs `main()` only when `fileURLToPath(import.meta.url) === process.argv[1]`, then `process.exit(code)`; an uncaught error prints its message and exits 1.

1. `dryRun = argv includes '--dry-run'`. Print `DRY RUN — nothing was changed` (dry) or `argus uninstall`, then a blank line. In dry-run, a `finally` prints a blank line plus the banner again at the end, whatever the path.
2. `repoRoot = stripSlash(ARGUS_UNINSTALL_TEST === '1' ? ARGUS_REPO_ROOT || mainRepoRoot() : mainRepoRoot())` (test seam honored only with the flag).
3. `config = loadConfig(ARGUS_CONFIG || join(repoRoot, '.argus.config'))`. No config or no `dailyDriverWorktree` → `argus doesn't look installed here (no .argus.config with a dailyDriverWorktree). Nothing to do.` exit 0.
4. `dd = stripSlash(config.dailyDriverWorktree)`. `checkWorktreeSelfContained(dd)` returns a message → print it, exit 1 (no prompts).
5. Scan: `homes = findClientHomes({installTargets, env, homedir, fs})`, `links = scanSkillLinks(homes, {mainRepoRoot: repoRoot, dailyDriverWorktree: dd}, fs)`, `mcp = planMcpRemovals(homes, {mainRepoRoot: repoRoot, canExec}, fs)`, `hookActions = planHookRemovals(homes, fs)`, `offers = listOffers(links)`. `canExec(cmd)` = some PATH dir (split on `path.delimiter`) contains `join(dir, cmd)`.
6. Print `argus repo:            <repoRoot>`, `daily-driver worktree: <dd>`, blank, `Client homes found:`, each `  <name> (<pathFormat>): <home>` (names: `Claude Code`, `Codex`, `Copilot CLI`) or `  (none)`, blank.
7. Nothing removable (no non-unverified links, no MCP actions, no hook actions) → `Nothing of argus's was found in these homes. Nothing to do.` plus `  left alone: <s>` for each MCP left-alone entry; exit 0.
8. Print intro lines (§6.10), blank.
9. **Gate 1:** `Continue? [y/N] `; not yes → `Nothing changed.` exit 0.
10. **Keep checklist** (only when offers exist): print `Which skills do you want to KEEP? Kept skills are copied in as real folders;` / `everything not kept is removed.` Labels: `<label> (<skill1>, <skill2>…)` when an offer has more than one skill, else `<label>`. Nothing pre-checked. Numbered prompt `Numbers to keep, comma-separated (Enter = keep none): ` (empty = none).
11. **Client checklist** (only if something was kept): clients = distinct `homes[].client` in order; print `Where should the kept skills go? (applies to every kept skill)`; all pre-checked; prompt `Numbers, comma-separated (Enter = all): ` (empty = all). Zero picked → `No client picked — nothing will be kept; every skill above is removed.`; else `choices[offer.id] = picked` for every kept offer.
12. `plan = buildPlan(...)`. Print `The plan, in order (<n> actions):`; each action `  <i padded to 3>. <first line of describeAction>` with continuation lines indented 7 spaces; each `  skipped: <s>`; blank.
13. Dry run → `A real run would then print:`, blank, `closingNotice(plan)`, exit 0.
14. **Gate 2:** `Proceed? [y/N] `; not yes → `Nothing changed.` exit 0.
15. Execute with a per-result callback: `done` → `✓ <first line>` plus ` (<note>)` when a note exists; `failed` → `✗ <first line> — <error ?? 'failed'>`; `manual` prints nothing yet.
16. If manual results: blank, `Do these by hand:`, then per action `  - <instructions>` and `    (<why>)`.
17. Blank, `closingNotice({...plan, kept: plan.kept filtered to copyDir actions that finished done})`, blank, `<done> done, <failed> failed, <manual> manual`. Exit 1 if any failed, else 0.

**Multi-select (`pickMany`):** TTY (`stdin.isTTY && stdout.isTTY`) → `promptCheckboxOutcome`; cancelled → throw `Cancelled`, caught at the top: blank line, `Cancelled. Nothing changed.`, exit 0. Non-TTY → print `  <n>) <label>` lines, then loop: empty → `emptyMeans` (`none` → `[]`, `all` → every index); `parseSelection` with any invalid token → `  Not valid: <tokens joined ', '> — use numbers 1-<count>, comma-separated.` and re-ask.

**Asker (`makeAsker`):** TTY → a fresh `readline` interface per question (closed right after, so the raw-mode checkbox can own stdin in between; stdin close resolves `''`). Piped → one long-lived line queue so no answers are lost between questions; EOF resolves `''`; prints a newline after each answer.

### 6.2 Catalog (`uninstall-catalog.js`)

```js
REMOVED = {
  argus: ['argus','bootstrap-argus','argus-defensive-review','argus-retrospective'],
  superpowers: ['brainstorming','diagnosing-superpowers','dispatching-parallel-agents','executing-plans',
    'finishing-a-development-branch','receiving-code-review','requesting-code-review',
    'subagent-driven-development','systematic-debugging','test-driven-development',
    'using-git-worktrees','using-superpowers','verification-before-completion',
    'writing-plans','writing-skills'],
  retired: ['gh-pr','reflect'],
};
OFFERS = [
  { id: 'product', label: 'Product package', skills: ['discover','strategize','ux-review','refine','product-shared'] },
  { id: 'pr', label: 'PR package', skills: ['open-pr','resolve-pr-reviews'] },
  { id: 'backlog', label: 'Backlog package', skills: ['backlog-triage','backlog-enrich','backlog-prioritize','backlog-groom','backlog-shared'] },
  { id: 'adversary-review', label: 'adversary-review', skills: ['adversary-review'] },
  { id: 'build-gold-dataset', label: 'build-gold-dataset', skills: ['build-gold-dataset'] },
  { id: 'eval-package', label: 'eval-package', skills: ['eval-package'] },
  { id: 'jira-ticket-generation', label: 'jira-ticket-generation', skills: ['jira-ticket-generation'] },
];
```

`classifySkill(name)` → `{kind:'removed', group}` | `{kind:'offered', offerId}` | `{kind:'unlisted'}`. Every directory under `skills/` must classify as removed or offered (test).

Typedefs:

```
ClientHome = { client, home, pathFormat, local, skillsDir, skillsRealDir|null,
  mcpConfigPath|null, claudeConfigDirEnv|null, settingsPath|null, settingsRealPath|null }
Action =
  { type:'removeLink', path, target }
| { type:'copyDir', from, to, skill }
| { type:'exec', argv, env?, unsetEnv?, why, precheck?: { path, isToml, variants } }
| { type:'writeJson', path, before, after, why }
| { type:'manual', instructions, why }
```

### 6.3 Homes (`uninstall-homes.js`, read-only)

`findClientHomes({installTargets, env, homedir, fs})`:
- Candidates = `normalizeTargets(installTargets)` with trailing slashes stripped and `local = t.local !== false`; plus extra Claude homes (posix, local): `env.CLAUDE_CONFIG_DIR` (stripped), and every `~/.claude-*` dir containing `.claude.json` or `settings.json` (unreadable homedir → none).
- Skip candidates whose home doesn't exist; de-dupe by `client + home`.
- `mcpConfigPath`: Claude windows → `<profile>/AppData/Roaming/Claude/claude_desktop_config.json` (null if `.claude` can't be stripped); Claude home equal to `~/.claude` and NOT equal to `CLAUDE_CONFIG_DIR` → `~/.claude.json` with `claudeConfigDirEnv = null`; any other Claude home → `<home>/.claude.json` with `claudeConfigDirEnv = home` (so an explicit `CLAUDE_CONFIG_DIR=~/.claude` counts as non-default). Codex → `<home>/config.toml`; Copilot → `<home>/mcp-config.json`.
- `skillsDir = <home>/skills`, `skillsRealDir = realpath` (null on error); Claude only: `settingsPath = <home>/settings.json`, `settingsRealPath = realpath or null`.

Only installTargets + `CLAUDE_CONFIG_DIR` + `~/.claude-*` are scanned (see §9).

### 6.4 Skill-link scan and link ownership (`uninstall-links.js`, read-only)

- `pathKey(p)`: lowercase if `^/mnt/<drive>(/…)?$` (case-insensitive Windows FS), else unchanged. 8.3 short names are not expanded (they fail to match → left alone, the safe direction).
- `isUnder(p, root)`: equal keys, or `p` starts with `root + '/'` (so a sibling prefix like `/repo-other` doesn't match `/repo`).
- `resolveLinkTarget(raw, dir)`: `X:\…` → `resolve(windowsToWsl(raw))`; else `resolve(dir, raw)` (relative targets resolve against the link's real dir; `..` normalized so `C:\repo\..\elsewhere` can't pass).
- `scanSkillLinks(homes, {mainRepoRoot, dailyDriverWorktree}, fs)`:
  - Repo roots = `{resolve(root), realpath(root), realpathSync.native(root) if available}` (native gives on-disk case on macOS APFS).
  - `inside(p)` = under any root; OR a dead junction from the old WSL-ext4 linker bug: `p` matches `/mnt/<d><rest>` and `<rest>` (case-sensitive posix) is under a non-`/mnt` root.
  - Daily-driver skill dirs = `resolve(dd,'skills')`, `resolve(realpath(dd),'skills')`, and the native form if any.
  - For each home with a non-null `skillsRealDir` (de-duped by real dir, so `~/.codex/skills -> ~/.claude/skills` is scanned once): `readdirSync(realDir)`; per entry lstat (in the real dir) must be a symlink, else skip (real folders are never touched); `abs = resolveLinkTarget(readlink, realDir)`; keep only if `inside(abs) || inside(realpath(abs)) || inside(native(abs))`.
  - Category: `classifySkill` removed/offered → that; else `unlisted` if `abs`/real/native equals `<ddSkills>/<name>` (or its realpath) for any dd skills dir, else `unverified`.
  - Record `{name, linkPath: <home.skillsDir>/<name>, target: abs, skillsRealDir, category}`.

### 6.5 MCP ownership (`uninstall-mcp.js`, read-only; `canExec` injected)

- `extractTomlTable(text, table)`: lines from a header `[table]` or `[table.*]` (also `[[…]]`, whitespace around dots tolerated, trailing `#` comment allowed) up to the next unrelated header; null if absent.
- `repoPathVariants(root)`: `[root]`; for `/mnt/<d>/rest` add `D:\rest` and `D:/rest`; for every variant containing `\` add a JSON-escaped `\\` form.
- `referencesRepo(text, variants)`: any variant matched as a whole path: `(?<![^\s"'=])<escaped variant>(?:[/\\"']|$)` (left boundary is start/whitespace/quote/`=`, so `/private/x/argus` doesn't match `/x/argus`; right boundary is a separator, quote or end).
- `argusEntryText(raw, isToml)` → registration text only, or null when no entry (JSON parse errors throw):
  - JSON: `mcpServers.argus`; collect `command` (string), string elements of `args`, `env.ARGUS_CONFIG`; each `JSON.stringify`-ed, joined by newlines.
  - TOML: from `[mcp_servers.argus]` and sub-tables, after stripping comments (a `#` outside basic `"…"` strings with backslash escapes or literal `'…'` strings; full-line comments become empty): `command`/`args` values of the main table only (multi-line `args` arrays followed by bracket depth computed outside quotes), plus any quoted `ARGUS_CONFIG = "…"|'…'` value (in `.env` sub-table or an inline env). Other keys never count.
- `planMcpRemovals(homes, {mainRepoRoot, canExec}, fs)`:
  - Variants = variants of the given root ∪ variants of its realpath.
  - Per home with an existing `mcpConfigPath`, de-duped by realpath: read; any read/parse error → `manual` `{instructions: "could not read <path>; remove the 'argus' server by hand if it points at <root>", why: '<client> MCP config unreadable'}`. No entry → nothing. Entry not referencing the repo → `leftAlone`: `<client> <path>: an 'argus' server not from this repo — left alone`.
  - Owned, `why = "remove the 'argus' MCP server registered from <root>"`, `precheck = {path, isToml, variants}`:
    - Claude windows → `writeJson {path, before: parsed, after: clone without mcpServers.argus, why}`.
    - Claude posix, argv `['claude','mcp','remove','argus','-s','user']`: `canExec('claude')` → `exec` with `env: {CLAUDE_CONFIG_DIR: dir}` when `claudeConfigDirEnv` is set, else `unsetEnv: ['CLAUDE_CONFIG_DIR']`, plus precheck. No CLI → `manual` with instructions `CLAUDE_CONFIG_DIR=<shellQuote(dir)> claude mcp remove argus -s user` or `env -u CLAUDE_CONFIG_DIR claude mcp remove argus -s user`.
    - Codex/Copilot: `cli = pathFormat==='windows' ? '<client>.exe' : client`; argv `[cli,'mcp','remove','argus']`; Codex non-windows pins `env: {CODEX_HOME: home}`; Copilot has no env. `canExec(cli)` → `exec` with precheck; else `manual` with `<KEY>=<shellQuote(v)> ` prefixes + `argv.map(shellQuote).join(' ')`.

### 6.6 Hook removal planning (`uninstall-hooks.js`, read-only)

For each home with a `settingsPath`, use `settingsRealPath ?? settingsPath`, de-dupe, skip missing. Parse error → `manual` `{instructions: 'Open <real> and delete the SessionStart hook whose command starts with CLAUDE_PLUGIN_ROOT= and ends with /hooks/session-start" (file is not valid JSON, so it was left untouched).', why: 'settings.json is unparseable; cannot safely remove superpowers SessionStart hook'}`. Else `removeSessionStartHook(structuredClone(parsed))`; if changed → `writeJson {path: real, before: parsed, after, why: 'remove superpowers SessionStart hook'}`.

### 6.7 Planner (`uninstall-plan.js`, pure)

- `checkWorktreeSelfContained(dd, fs)`: null if both `skills/resolve-pr-reviews/scripts/pr-reviews.js` and `skills/adversary-review/scripts/adversary-review.js` exist under dd; else ``Your daily-driver worktree (<dd>) is behind main. Run `npm run setup` first, then re-run `npm run uninstall`.``
- `listOffers(links)`: `OFFERS` with at least one linked member (copied, full skill list), then one `{id: 'unlisted:<name>', label: '<name> (not part of argus — yours?)', skills: [name]}` per distinct `unlisted` link name.
- `buildPlan({homes, links, mcp, hookActions, choices, dailyDriverWorktree}, fs)`:
  - `unverified` links → `leftAlone`: `<linkPath> -> <target> (points into this repo but isn't a known argus skill — left alone)`. Every other link → `removeLink {path: linkPath, target}`; record `pathKey(join(skillsRealDir ?? dirname(linkPath), name))` as freed.
  - For each `choices[offerId]` with clients: destination dirs = distinct per chosen client home keyed by `skillsRealDir ?? skillsDir` (value `skillsDir`). Per skill: `from = <dd>/skills/<skill>`; missing → skipped `<from> not found in the daily-driver worktree — <skill> not copied`. Per dir: `to = <skillsDir>/<skill>`; if `to` lstat-exists (a dead link counts) and `<realDir>/<skill>` isn't freed → skipped `<to> already exists and isn't ours — not overwritten`; else `copyDir {from, to, skill}` and `kept.push({skill, to})`.
  - Returns `{actions: [...removals, ...copies, ...mcp.actions, ...hookActions], skipped, kept, leftAlone: [...linkLeftAlone, ...mcp.leftAlone]}`. **Order:** all link removals, then all copies, then MCP, then hooks.
- `parseSelection(answer, count)`: split on `[\s,]+`; tokens must be `^\d+$` and within 1..count; returns `{indices: sorted deduped 0-based, invalid: [bad tokens]}`; empty answer → `{[], []}`.

### 6.8 Report text (`uninstall-report.js`, pure)

- `DRY_RUN_BANNER = 'DRY RUN — nothing was changed'`; `SUPERPOWERS_INSTALL_URL = NEW_ARGUS_INSTALL_URL = '<URL TBD>'`.
- `shellQuote(s)`: unchanged if `^[A-Za-z0-9_\/.:=@%+,-]+$`, else `'…'` with `'` → `'\''`.
- `describeAction(a)`:
  - removeLink → `remove link <path> -> <target>`
  - copyDir → `copy <from> -> <to>`
  - exec → `run: ` + `<K>=<shellQuote(v)> ` per env + (`env -u K1 -u K2 ` if unsetEnv) + `argv.map(shellQuote).join(' ')` (e.g. `run: env -u CLAUDE_CONFIG_DIR claude mcp remove argus -s user`)
  - writeJson → `edit <path>: <why>` then, per top-level key whose JSON differs, `    - <key>: <json before>` / `    + <key>: <json after>` (absent sides omitted)
  - manual → `MANUAL: <instructions>`; unknown → `unknown action <json>`.
- `closingNotice({kept, skipped, leftAlone})`: two boxes (`╔`, `═`×(width+4), `╗`; rows `║  <line padded to width>  ║`; `╚…╝`; width = longest line of both boxes), separated by blank lines:
  - Superpowers box: `Superpowers was removed. You MUST reinstall it.` / `brainstorming, TDD, plans, debugging, etc. are gone until you do.` / `Superpowers is now a plugin installed into Claude, Copilot or Codex:` / `  <URL TBD>`
  - Argus box: `argus was removed. You MUST reinstall it.` / `The new argus is a plugin installed into Claude, Copilot or Codex:` / `  <URL TBD>`
  - Then: ``The repo was left untouched; `npm run setup` restores old argus —`` / `and replaces kept copies with links again.`; blank; `Kept skills: none` or `Kept skills (copied as real folders):` + `  - <skill> -> <to>`; optional blank + `Skipped:` + `  - <s>`; optional blank + `Left alone:` + `  - <s>`; blank; `Restart your agent CLI(s) so they stop loading argus.`

### 6.9 Executor (`uninstall-execute.js`) — the only mutating module

`executeActions(actions, deps, onResult?)`: run in order; each wrapped on its own (a thrown error becomes `{status:'failed', error: firstErrorLine(e)}`); call `onResult` after each; never stops early. `firstErrorLine(e)` = first non-blank stderr line, else first line of `e.message`.

Per action (re-check right before acting):
- **removeLink:** lstat ENOENT → `done` with note `already gone`. Not a symlink → `failed: not a link any more — left alone`. Re-resolve `resolveLinkTarget(readlink, realpath(dirname(path)))`; `pathKey` differs from the planned target → `failed: link changed since the plan (now -> <now>) — left alone`. Else `deps.removeLinkOnly(path)`; still exists → `failed: link is still there after removal`; else `done`.
- **copyDir:** `mkdirSync(dirname(to), {recursive:true})`. `to` exists → `failed: destination exists — not overwritten`. `tmp = to + '.argus-uninstall-tmp'` exists → `failed: <tmp> already exists (not ours) — left alone`. `from` not a dir → `failed: <from> is not a directory`. `mkdirSync(tmp)`, then copy every child with `copyTree` (`statSync` follows every symlink, including nested ones; dirs `mkdirSync`, files `copyFileSync(…, COPYFILE_EXCL)`; anything else → error `<from> is not a regular file or directory — copy stopped`; a dir whose realpath is already an ancestor → error `symlink cycle at <from> — copy stopped`). If `to` appeared meanwhile → error `destination appeared during the copy`. `renameSync(tmp, to)`. On any error remove only the tmp dir it created (`rmSync(tmp, {recursive: true, force: true})`) → `failed: copy failed — nothing left at <to>: <firstErrorLine>`.
- **exec:** if `precheck`: re-read the config (error → `failed: could not re-read <path> — left alone: <line>`); `argusEntryText` throws, is null, or no longer `referencesRepo` → `failed: argus entry changed since the plan — left alone`. Env = `{...deps.env, ...a.env}` minus `unsetEnv`. `execFileSync(argv[0], argv.slice(1), {stdio:'pipe', env, encoding:'utf8', timeout: 120_000})`; error → `failed: <first stderr line>`; else `done`.
- **writeJson:** parse the current file; not `isDeepStrictEqual` to `before` → `failed: changed since the plan was made — not edited`. `tmp = path + '.argus-uninstall-tmp'` exists → `failed: <tmp> already exists (not ours) — not edited`. Backup name: `<path>.argus-uninstall.bak`, else first free `.bak.2`, `.bak.3`, …; `copyFileSync(path, bak, COPYFILE_EXCL)`, and on `EEXIST` (appeared after the probe) advance to the next number and retry (max 50 tries). Atomic write: `copyFileSync(path, tmp, EXCL)` (keeps file mode) → `writeFileSync(tmp, JSON.stringify(after, null, 2) + '\n')` → `renameSync(tmp, path)`; on error `rmSync(tmp, {force: true})` and rethrow. `done` with note `backup: <bak>`.
- **manual:** `{status: 'manual'}` (never executed). Unknown type → `failed: unknown action type <json>`.

`makeRemoveLinkOnly({unlinkSync, execFileSync, platform})`: `unlink(p)`; on failure, only if `platform === 'wsl'` and `p` matches `^/mnt/[a-zA-Z]/` → `execFileSync('cmd.exe', ['/C', 'rmdir', wslToWindows(p)], {stdio:'pipe'})` (no `/s`: removes a junction, refuses a non-empty real dir); else rethrow. Never recursive, never `del` (which would delete files inside a junction's target), never `forceRemoveLink`.

`realDeps()` = real `lstatSync, readlinkSync, statSync, readdirSync, realpathSync, mkdirSync, renameSync, rmSync, execFileSync, readFileSync, writeFileSync, copyFileSync`, `removeLinkOnly: makeRemoveLinkOnly({unlinkSync, execFileSync, platform: detectPlatform()})`, `env: process.env`. No recursive remover is exposed for links (`rmSync` is used only on the executor's own tmp dirs/files).

### 6.10 Intro lines (exact)

```
This will:
  - remove <n> argus skill[s] (<names>)                      ← "(…)" only when n > 0
  - remove <n> vendored superpowers skill[s][ and <m> retired skill[s] (<names>)]
    from:                                                    ← when any non-unverified link exists
      <dirname of every non-unverified link, de-duped>
  - ask which of the other <k> skill package[s] to keep (copied in as real folders);   ← when offers exist
    anything you don't keep is removed
  - unregister the argus MCP server installed from this clone:
      <first line of describeAction for each MCP action>
  (or)  - (no argus MCP server from this clone was found)
  - leave alone: <each mcp.leftAlone>
  - remove the superpowers SessionStart hook (<writeJson path or manual why, comma-joined>)   ← when hook actions exist
  - leave alone <u> link[s] into this repo that isn't a known argus skill / aren't known argus skills   ← when unverified links exist
<blank>
The argus repo itself is not touched (clone, daily-driver worktree, .argus/ index,
.argus.config). Running `npm run setup` again brings old argus back.
<blank>
!!! Superpowers (brainstorming, diagnosing-superpowers, dispatching-parallel-agents, …) is removed too.
!!! You MUST reinstall superpowers afterwards.
```

Counts come from the distinct names of `removed`-category links grouped by catalog group; pluralization adds `s` when the count ≠ 1. The unverified line reads `leave alone 1 link into this repo that isn't a known argus skill` or `leave alone <u> links into this repo that aren't known argus skills`.

## 7. Tests to write (behaviors pinned by the existing suite)

Run with vitest from `tests/scripts/`. Group by module:

**`setup-plan`** — fresh when no config; fresh when config but no worktree (same step list); maintenance with no stale projects has no `rebuild-index`; each stale project adds `rebuild-index:<name>` in order; maintenance always includes deps/build-main/merge/build-worktree/relink regardless of behind/dirty.

**`merge-policy`** — `up-to-date` at 0 behind even if dirty/conflicted; `dirty-stop` when behind+dirty, also when also conflicted (dirty wins); `clean`; `conflict`.

**`targets` / `select-targets`** — `targetsMode` detect for `[]`/undefined/null, reuse for non-empty; `normalizeTarget` legacy `{claudeHome}` → client shape, carries `wslHome`, leaves normalized entries alone; mixed lists normalized consistently; `enumerateTargets` includes only existing homes (marked local), honors `CODEX_HOME`, adds a cross-env target when the foreign home exists; `parseNumberedSelection` empty/`all` → all, 1-based dedupe+sort, ignores garbage/out-of-range; `newlyDetected` by client+home; `selectTargets` numbered fallback when not TTY, injected checkbox when TTY, `[]` with no candidates; `toPersistedTarget` keeps only persistable fields, includes `wslHome`, round-trips `local:false`; `localPathFormat`; `codexMultiAgentReminder` null without codex, mentions `multi_agent` with it.

**`checkbox`** — all pre-checked by default; explicit mask honored; up/down wrap; space toggles; all/none; enter → done and later actions ignored; cancel → cancelled; `selectedIndices`; `checkboxOutcome` distinguishes cancel from empty.

**`clients`** — `argusMcpPaths` posix paths and `/mnt` → Windows conversion; `stdioMcpArgv` add/remove for codex and identical shape for copilot; `claudeCliMcpArgv` uses `-s user`, `-e`, name after `--`; `quoteArgv` double-quotes each token; server-name constant; registry has three descriptors with required fields; `getClient`; codex local → remove then add; non-local → no exec, prints manual; add failure → skip-and-warn without throwing; WSL→Windows bridge drives `copilot.exe` twice via `execFileSync` with `['mcp','add','argus','--','wsl.exe','bash','-c', payload]`, payload contains the POSIX entry path and never `\home\`, log mentions `wsl.exe bridge`; absent Windows CLI (codex) → no exec, guidance message; bridged unregister runs `copilot.exe mcp remove argus`; Claude posix uses `"claude" "mcp" "add" "-s" "user"`; Claude windows writes one desktop config whose `argus.command` is `wsl.exe`; `makeConsoleIo` usable; `CODEX_HOME` override and fallback; Claude non-local prints manual; unregister: codex local runs remove, codex non-local nothing, Claude posix `remove -s user`, Claude posix non-local nothing, Claude windows removes `argus` but preserves other servers.

**`linker`** — `canLinkSkillsToTarget`: WSL→Windows allowed for `/mnt/<d>/…`, refused for `/home/…`; native posix, native Windows, Windows→WSL, plain linux always true.

**`hooks`** — command sets `CLAUDE_PLUGIN_ROOT` inline and runs `bash "<wt>/hooks/session-start"`; `/mnt` path → forward-slash Windows path; recognizer matches its own output for any path and rejects foreign hooks; upsert adds `hooks.SessionStart` with upstream matcher + `shell: bash`; preserves unrelated keys and other hook events; idempotent; replaces a stale path instead of adding a second; keeps a foreign hook in the same group; tolerates garbage settings; remove no-ops when absent, drops empty containers, keeps other events; file IO: creates `settings.json` when missing then removes cleanly, merges without clobbering, refuses an unparseable file and prints manual JSON, unregister without a file is a silent no-op.

**`config`** — `loadConfig` reads `{projects}` and legacy shapes; `readProjectRepos` for both and `{}`; `writeConfig` round-trips repos + installTargets, preserves `conventions`/`sampleCodePath` when omitted, preserves other projects; `removeProject` deletes one and keeps others + top-level, `{removed:false}` for unknown/no file, writes `{projects:{}}` when last; `writeTopLevelConfig` migrates a legacy flat config to `projects.tcn`, preserves `{projects}` as-is, first run yields `{projects:{}}` + top-level fields.

**`preflight`** — `checkHuggingFaceReachable` false on rejecting fetch (never throws), true on resolve; `runPreflight` works with `{projects}` configs (regression for a TypeError), checks repo paths for every project, still checks legacy flat configs.

**`setup-deps`** — `checkPython` shape; version `^3\.\d+` when found; null version when not; per-platform instructions (`brew install python`, `apt`, `python.org`, generic); `pickInstaller` uv argv with no `--system`, pip fallback argv, same requirements path.

**`skills-self-contained`** — kept skills ship their helpers (`resolve-pr-reviews/scripts/pr-reviews.js`, `adversary-review/scripts/adversary-review.js`); the two `session-store.js` copies are byte-identical; no `.md` in any OFFERS skill contains `ARGUS_ROOT`, `/../..`, `<argus-repo>`, `ARGUS_CONFIG` or `.argus.config`.

**`uninstall-catalog`** — every `skills/` dir classifies removed/offered; no duplicate names; unlisted and package-member classification.

**`uninstall-homes`** — claude+codex homes with MCP paths; copilot `mcp-config.json`; `.claude-*` dirs with config files added (skills symlinks followed); missing homes dropped; `CLAUDE_CONFIG_DIR` included even when not `.claude-*`; de-dupe by client+home; explicit `CLAUDE_CONFIG_DIR == ~/.claude` uses `<dir>/.claude.json`; windows Claude → `claude_desktop_config.json`.

**`uninstall-links`** — categorizes ours, ignores foreign links and real dirs; de-dupes shared skills dirs; skips homes without skills dir; rejects sibling-prefix targets; resolves relative targets; normalizes Windows-style targets so `..` can't escape; recognizes `/mnt/<d><repo>/…` dead junctions as ours; `/mnt` case-insensitive, 8.3 names left alone; no case folding elsewhere; symlinked repo/dd paths (macOS `/var`, `~/code`); targets through a symlinked path; `realpathSync.native` forms.

**`uninstall-mcp`** — `extractTomlTable` table + sub-tables, stops at unrelated header, null when absent; `repoPathVariants` posix + escaped, `/mnt` adds Windows and JSON-escaped forms; canonical-path match when root given via symlink; left boundary (path merely ending with ours isn't ours); codex owned → exec; no CLI → manual; manual fallbacks quote `CODEX_HOME`/`CLAUDE_CONFIG_DIR`, copilot gets none; codex exec carries a TOML precheck; ownership only from command/args/ARGUS_CONFIG; `argusEntryText`+`referencesRepo` agree with the planner; codex windows uses `.exe`; unowned codex/copilot/Claude/desktop left alone; sibling clone not owned; no entry → nothing; Claude posix → `claude mcp remove`, env only when configured; Claude no CLI → manual with exact command; codex local pins `CODEX_HOME`, windows doesn't; desktop → `writeJson` without mutating `before`; broken JSON → manual; missing config skipped, same config de-duped; TOML comments (commented-out and trailing) ignored, comment lines with brackets inside multi-line args ignored, real codex shape still owned, quoted value containing `#` not truncated.

**`uninstall-hooks`** — removes ours keeping foreign hook and keys; no action when absent; de-dupes homes sharing a real settings file; invalid JSON → manual; null paths and missing files skipped.

**`uninstall-plan`** — self-contained check; `listOffers` omits unlinked offers and appends unlisted; removals before copies; keeping PR for claude copies to posix + windows dirs; keeping nothing → zero copies; foreign real dir at destination skipped; a removed (even dead) link frees its destination; `/mnt` destination matched case-insensitively; keyed by real skills dir (`codex/skills -> claude/skills`); missing worktree source skipped; `parseSelection` comma/space, dedupe/sort, every invalid token reported, empty valid.

**`uninstall-report`** — constants; `describeAction` for each type incl. shell-quoting and env prefix, writeJson shows only changed top-level keys; closing notice content order (superpowers box first, argus box, `npm run setup`, kept path, skipped, left alone, restart line), contains both URLs, `plugin`, `replaces kept copies with links again`; `Kept…none` when nothing kept. **Static guard:** regex
`/\b(writeFileSync|writeFile|rmSync|unlinkSync|symlinkSync|cpSync|copyFileSync|mkdirSync|renameSync|execSync|execFileSync|spawnSync|spawn|rmdirSync|truncateSync|chmodSync)\b|(?<!\/\.)\bexec\s*\(|\b(execFile|fork|unlink|symlink|rename|cp|rm|writeFile|appendFile|appendFileSync|mkdir|copyFile|rmdir|truncate|chmod)\s*\(/`
must match none of `scripts/uninstall.js` and every `scripts/lib/uninstall-*.js` except the executor (at least 7 such libs); `uninstall.js` has no `child_process`, only read-only `node:fs` named imports (`existsSync, lstatSync, readdirSync, readFileSync, readlinkSync, realpathSync, statSync`), no namespace/`fs/promises`/`'fs'` import, no `linker.js`, and imports exactly `{ executeActions, realDeps }` and calls `executeActions(…realDeps()`.

**`uninstall-execute`** — removeLink removes the link and keeps the target, refuses a real dir, already-missing → done, refuses a changed target, accepts a relative link resolving to the planned target; copyDir copies dereferenced real files, fails on a symlink cycle, refuses an existing destination, mid-copy failure leaves nothing at `to` and removes only its tmp, refuses a pre-existing tmp, lands via rename; exec passes env and honors `unsetEnv`, non-zero exit → failed with first stderr line, missing command failed not thrown, MCP precheck runs only while still ours; writeJson writes `after` and keeps old bytes in `.bak`, never overwrites an earlier `.bak`, a backup appearing after the probe is never clobbered (EXCL → next `.bak.N`), refuses a changed file, writes via tmp+rename keeping the file mode; a failure doesn't stop later actions, manual never executed, unknown type fails without throwing; `makeRemoveLinkOnly` removes a symlink leaving target files, refuses a real dir (never recursive), on WSL `/mnt` falls back to `cmd.exe /C rmdir` without `/s` and never `del`, no fallback off WSL or for non-`/mnt` paths; `realDeps` has no recursive link remover.

**`uninstall-cli`** (end-to-end in a temp fixture: `HOME`, `ARGUS_CONFIG`, `ARGUS_REPO_ROOT` + `ARGUS_UNINSTALL_TEST=1` all in tmp, `CLAUDE_CONFIG_DIR`/`CODEX_HOME` deleted, `PATH` = a tmp bin holding a fake `claude` that logs argv/env; never loosen this) — intro answered N → `Nothing changed.`, no file changes, no CLI call; intro lists `1 argus skill`, the skills dir, `claude mcp remove argus -s user`, `SessionStart hook`, `npm run setup`; `from:` lists every dir holding a non-unverified link; `--dry-run` banner top and bottom, plan printed, no Proceed, nothing changed; Proceed N → nothing changed; full run removes argus/discover links (targets intact), copies the PR package as real dirs including nested helper files, removes the hook leaving `theme`, writes `settings.json.argus-uninstall.bak` with the old bytes, calls `claude` once with `mcp remove argus -s user`, `CLAUDE_CONFIG_DIR` unset and fixture `HOME`, leaves the user's real skill, prints `✓ remove link …`, `✓ run: env -u CLAUDE_CONFIG_DIR claude mcp remove argus -s user`, `\d+ done, 0 failed, 0 manual`; failing `claude` → `✗ run: … — nope: not registered`, the rest still run, exit 1, `1 failed`; no `claude` on PATH → `Do these by hand:` with the `env -u` command, `1 manual`; invalid numbered answer re-asks naming bad tokens (`Numbers to keep` printed twice); keeping nothing skips `Where should` and plans no copies; finds `.argus.config` under `ARGUS_REPO_ROOT`; ignores `ARGUS_REPO_ROOT` without the test flag; exit 1 with no prompts when the worktree is behind; exit 0 when not installed.

**`worktree-cleanup`** (project.js) — removes worktrees and prunes the source repo; tolerates bare-string repo values and gone repos; no-op without a worktrees dir; cleans metadata when `git worktree remove` fails; prunes the owning repo when sweeping a leftover not in config; refuses traversal keys; refuses a symlinked entry, a symlinked worktrees base, and a symlinked `.argus/<project>` ancestor (inside or outside the repo, with or without a `worktrees` child).

## 8. Platform matrix

| Concern | macOS | Linux | WSL | Windows (native) |
|---|---|---|---|---|
| `detectPlatform` | `mac` | `linux` | `wsl` (`/proc/version` has `microsoft`) | `windows` |
| Local path format | posix | posix | posix | windows |
| Foreign (cross-env) homes | none | none | Windows profile via `cmd.exe` + `wslpath`; targets `pathFormat: windows`, `local: false`, label `[Windows]` | WSL home via `wsl.exe -l -q` / `wsl.exe echo $HOME` at `\\wsl$\<distro><home>`; targets `posix`, `wslHome`, `local: false`, label `[WSL]` |
| Skill links (local) | `symlinkSync(…,'dir')` | same | same | `symlinkSync(…,'junction')`, Windows-format source |
| Skill links (cross-env) | — | — | `cmd.exe /C mklink /J`; only if worktree on `/mnt/<d>/`; verify reparse tag `0xa0000003`, recreate LX points | `wsl.exe ln -s` into `<wslHome>/.claude/skills` |
| SessionStart hook | Claude targets only | same | Claude targets only, skipped where links are skipped | Claude targets; command uses `C:/…` forward slashes; needs Git Bash (Claude Code ≥ 2.1.81 resolves `shell: bash`) |
| MCP: Claude | `claude mcp add -s user …` | same | posix local: CLI; Windows target: `claude_desktop_config.json` wsl.exe bridge | windows-format local target → desktop config with wsl.exe bridge (no `claude mcp add`; §9) |
| MCP: Codex/Copilot | `<cli> mcp add …` | same | local: CLI; Windows target: `<cli>.exe` over interop via bridge, else skip-with-guidance | local: CLI with Windows paths via cmd.exe; WSL target non-local: print command |
| `canExec` / PATH | `:` | `:` | `:` (Windows `.exe` visible via interop) | `;` in clients.js; uninstall uses `path.delimiter` with no PATHEXT, so bare `claude`/`codex` never match |
| `forceRemoveLink` | `rmSync` | `rmSync` | `rmSync` + `cmd.exe rmdir` + `del /f /q` for `/mnt` paths | `rmSync` + `dir` probe + `fsutil reparsepoint delete` + `rmdir` + `del` |
| Uninstall | primary platform; realpath and `realpathSync.native` comparisons (`/var` → `/private/var`, APFS case) | supported | supported incl. Windows homes under `/mnt/<d>/`: case-insensitive keys, `cmd.exe rmdir` fallback, dead `/mnt/c/home/…` junctions count as ours | not a stated target (no PATHEXT probe → MCP removals become manual) |
| Python venv | `.venv/bin/python3` | same | same | `.venv\Scripts\python.exe`; activate hint `.venv\Scripts\activate` |
| Git credential helper probe | `command -v` | same | same | `where` |
| Worktree path-format mismatch | — | — | registered as `/mnt/…` but run from Windows (or vice versa) → hard error with hint | same |

## 9. Known gaps and quirks (reproduce or fix deliberately; not fixed here)

**Installer correctness**
1. **Native Windows Claude never gets `claude mcp add`.** `claudeClient.registerMcp` routes every `pathFormat: 'windows'` target to `claude_desktop_config.json`, including a native-Windows local Claude Code home. Claude Code CLI (`~/.claude.json`) is left unregistered, and the Desktop entry requires WSL with node at the `/mnt/<d>/…` form of the main clone.
2. **Windows→WSL links ignore the client.** `createSkillLink` hardcodes `<wslHome>/.claude/skills/<name>`, so cross-env Codex/Copilot WSL targets get their skills linked into `.claude`, and `linkSkills` still lstat-checks the UNC client path.
3. **`claude_desktop_config.json` is clobbered when unparseable.** `upsertClaudeDesktopConfig` falls back to `{}` on parse failure and overwrites the file, unlike the hook writer which refuses.
4. **Remove-then-add can leave nothing registered.** `runCliRegistration` removes first; if add then fails the old registration is gone (the warning prints the add command).
5. **`quoteArgv` doesn't escape.** Tokens are wrapped in `"…"` and passed to a shell; a path containing `"`, `$`, or a backtick breaks or expands. Uninstall uses proper `shellQuote` + `execFileSync`.
6. **`linkSkills` deletes real directories.** A non-symlink at `<home>/skills/<name>` (a user folder with a matching name, or an uninstall "kept" copy) is removed recursively by `forceRemoveLink` without prompting. The uninstall closing notice relies on this ("replaces kept copies with links again"). On WSL `/mnt` paths `forceRemoveLink` also runs `del /f /q`, the call the uninstaller deliberately avoids because it deletes files inside a junction's target.
7. **Config is anchored to `repoRoot()`, not the main clone.** `config.js` and `setup.js` use the checkout the script runs from. Running setup/fix-links from the daily-driver reads/writes the worktree's own (absent) `.argus.config` and looks for `.claude/worktrees` inside the worktree (→ fresh mode, nested worktree). MCP paths and uninstall correctly use `mainRepoRoot()`.
8. **Worktree detection ignores `dailyDriverWorktree`.** `detectExistingWorktree` takes the first `*-daily-driver` dir in readdir order under `<root>/.claude/worktrees`; the config field is only written, never read by setup (uninstall does read it).
9. **fix-links can create a worktree but never records it.** `runPreflight` may interactively create the daily-driver without writing `dailyDriverWorktree`, so a later uninstall reports "doesn't look installed". fix-links with no persisted targets also links into every detected home, including cross-env ones, silently and without persisting them.
10. **Maintenance doesn't refresh Python deps or recheck superpowers conflicts.** Its `deps` step is `npm install` only; a `requirements.txt` change isn't applied until a fresh run. A declined "newly detected" client is re-offered on every run, and fresh mode with persisted targets never offers new clients at all.
11. **Uninstall only scans recorded homes.** `findClientHomes` uses `installTargets` + `CLAUDE_CONFIG_DIR` + `~/.claude-*`; a default `~/.claude`, `~/.codex` or `~/.copilot` not in `installTargets` (legacy or fix-links-only installs) is not scanned. Conversely, install ignores `CLAUDE_CONFIG_DIR` when choosing the Claude home.
12. `installPythonDeps` calls `process.exit(1)` mid-setup with the readline interface open.

**Inconsistencies and dead code**
13. `clients.unregisterMcp` and `hooks.unregisterSessionStartHook` are unused by any script (tests only); uninstall has its own planners. Their Claude remove argv order (`remove -s user argus`) differs from uninstall's (`remove argus -s user`); both are valid.
14. `planSetupSteps` accepts `worktreeBehind`/`worktreeDirty` but ignores them; `isWorktreeDirty` still runs git for nothing. The fresh `steps` list is never read.
15. Fresh setup fetches origin and detects the worktree twice (once in `main`, again in `runFreshSetup`).
16. `  git + HuggingFace reachable.` is printed whenever HuggingFace responds, even right after git warnings.
17. Stale comments: setup's preflight comment says setup "runs an index build" (fresh never does); `preflight.js` header says "before update/fix-links" (`update` was removed); `build.js` mentions copying `cache/index.json`.
18. `toPersistedTarget`'s JSDoc return type omits `local`, which it does include. `normalizeTarget` infers `local = !wslHome` for legacy entries, so an old WSL→Windows target (no `wslHome`) becomes `local: true`. Low impact: Claude windows always uses the desktop writer and Codex/Copilot on WSL take the bridged path regardless.
19. Reused (persisted) targets lose `displayName`, so labels show the client id (`claude  (/home/x/.claude)`) instead of `Claude Code`. Uninstall names Copilot `Copilot CLI`; clients.js says `Copilot`.
20. The two "choose a handle" prompts differ (`— e.g. in branch listings` in setup vs `— in branch listings` in preflight), as do the confirm lines (`Worktree: …` vs `Your worktree will be …`).
21. Uninstall's intro can print `remove 0 vendored superpowers skills and 1 retired skill (…)`. Its closing-notice install URLs are the placeholder `<URL TBD>`. Its main-module check is a plain string compare of `fileURLToPath(import.meta.url)` with `process.argv[1]` (fails through a symlinked or differently-cased path).
22. The Claude Desktop bridge payload sources only `~/.nvm/nvm.sh`; a node installed elsewhere and not on the non-login PATH fails at launch.
23. `fetchOrigin` has a 30 s timeout; no other installer git call has one.
24. `hooks/hooks.json`, `run-hook.cmd` and `hooks-cursor.json` are unused by the installer (fidelity only).

**Repo files and docs**
25. `.argus.config.example` mentions LLM summary providers (to drop per master prompt §11.8), lacks `installTargets`, and uses bare-string repo values while `project.js` writes `{path}` objects.
26. `requirements.txt` lists `libcst`, which nothing imports (master prompt §11.6).
27. `.gitignore` has duplicates (`/presentation/` twice, `.argus` and `.argus/`) and a `/.claire/worktrees/` entry that looks like a typo of `.claude`.
28. `docs/manual-install.md` says Python is optional ("skipped gracefully if missing"); setup hard-exits without it and CLAUDE.md calls it a hard requirement. It also describes the Claude Desktop entry as a `wsl.exe -e ...` command, while the code writes `wsl.exe` with `['bash','-c', …]`.

## 10. Other scripts

### 10.1 `scripts/copy-assets.js`

`ASSET_EXTENSIONS = ['.py']` (only the Python embedder is loaded from `dist/` at runtime; tree-sitter `.scm` files are intentionally not copied). `findRuntimeAssets(srcDir)` walks recursively and returns POSIX-style relative paths; `copyAssets(srcDir, distDir)` mkdirs and copies each, returning the list. When run directly (`resolve(argv[1]) === this file`), copies `<root>/src` → `<root>/dist` and logs `[copy-assets] copied <n> runtime asset(s) to dist/`.

### 10.2 `scripts/eval-triggers.mjs`

Discovers `skills/<skill>/evals/<file>` (default `trigger-queries.json`). Flags: `--runs N` (default 3), `--set train|validation`, `--limit N`, `--file <name>`, `--skill <name>`, `--dry`. Spec JSON: `{ skill_name, queries: [{ query, should_trigger, set }] }` (missing `skill_name` throws; invalid JSON prints `Invalid spec at <path>: <msg>` and exits 1). Each run spawns `claude -p <query> --output-format stream-json --verbose` in the repo root and reads NDJSON: the first `tool_use` in an `assistant` message decides — `Skill` with `input.skill === skill_name` → triggered, any other tool → not triggered; a `result` event, process close or spawn error first → not triggered / inconclusive (error and the 90 s timeout are inconclusive). The child is SIGKILLed as soon as the verdict is known. Rate = triggers / (runs − inconclusive); pass if `should_trigger ? rate > 0.5 : rate < 0.5`. Writes `evals/results/<file with -queries.json → -results.json, else +-results.json>` containing `{summary: {skill, threshold, runs_per_query, train, validation, overall}, results}`. No specs → error naming the expected path, exit 1. Prints per-query `[PASS]/[FAIL]` lines and an OVERALL block; exit 1 if any skill has failures (dry runs only print the plan).

### 10.3 `scripts/lib/worktree-cleanup.js` (used by `project.js remove`)

`cleanupProjectWorktrees(projectArgusDir, repos, trustedRoot)` removes `<projectArgusDir>/worktrees/*`:
- Refuse (warning, no action) unless `projectArgusDir` is strictly inside `trustedRoot`: `  Warning: '<dir>' is not contained in trusted root '<root>' — refusing to clean up worktrees.`
- Resolve `worktrees` via `walkTrustedPath(root, segments)`: lstat each component from the root, returning null on the first symlinked or non-directory existing component (missing components are fine; never created here — mirrors `src/mcp-server/branch-handling/safe-path-walk.ts`). Null → `  Warning: an ancestor of the worktrees directory under '<dir>' is a symlink (or not a real directory) — refusing to clean up worktrees through it.` Missing dir → return.
- Per repo key (value `{path}` or bare string): `safeWorktreeDir` requires key `^[A-Za-z0-9._-]+$` and not `.`/`..`, a safe base, and a non-symlink candidate; else `  Warning: unsafe repo key '<key>' — refusing to derive a worktree path from it, skipping cleanup for it.` If the repo path exists: `git worktree remove --force <wt>` (10 s) → on failure `git worktree unlock` (5 s) and retry → still failing: `rmSync` + `git worktree prune` (5 s). Then `rmSync(wt, recursive, force)`.
- Sweep leftovers: each remaining entry through `safeWorktreeDir` (unsafe → `  Warning: skipping stale-sweep for unsafe worktree entry '<entry>'`); derive the owning repo from its `.git` file (`gitdir: <repo>/.git/worktrees/<name>`), `rmSync` it, `git worktree prune` in the owner (errors ignored). Finally `rmSync(worktreesDir)`.
