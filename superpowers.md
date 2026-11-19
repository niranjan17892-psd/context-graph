# Rebuild Spec — Absorbed Superpowers Skills and Hook

Rebuild spec for the 15 development-workflow skills vendored from the upstream **superpowers** plugin, the `hooks/` directory, the `tests/brainstorm-server/` suite, and `LICENSE-superpowers`, plus the argus-side wiring that registers the SessionStart hook. A fresh agent given only this document should be able to recreate the vendored tree byte-for-byte from upstream (preferred) or, failing that, rewrite each skill with equivalent behavior.

**Upstream source:** `obra/superpowers` (GitHub, `https://github.com/obra/superpowers.git`), tag **`v7.0.0`** (released 2026-10-09), by Jesse Vincent, MIT licensed. This document was written without fetching from the network; the upstream identity is recorded from the repo's own commits, license notice, and update plan.

**Headline finding:** at the current `HEAD` (`d56a1ac7`), **every vendored file is an unmodified copy of upstream v7.0.0.** No skill carries a surviving local content modification. Four local edits were made over the life of the vendored tree, and every one was later overwritten by a "straight copy-over" upstream sync (details in section C). The only argus-specific parts are integration-level: which upstream paths are copied (and which are deliberately excluded), git file modes, line-ending normalization, the license notice header, and the SessionStart registration code in `scripts/lib/hooks.js`.

---

## A. Vendoring instructions

### A.1 Obtain upstream

Clone the exact tag with line-ending conversion disabled, so no CRLF is introduced into shell scripts (the v7.0.0 update plan requires this and checks it with `file`):

```bash
git -c core.autocrlf=false clone -q --branch v7.0.0 https://github.com/obra/superpowers.git /tmp/superpowers-v7
file /tmp/superpowers-v7/skills/brainstorming/scripts/server.cjs   # must NOT report "CRLF line terminators"
```

### A.2 Copy exactly these paths

Copy each upstream directory below into the same relative path in the argus repo. Copy the whole directory, with one exclusion list (A.3).

| Upstream path | Argus path |
|---|---|
| `skills/brainstorming/` | `skills/brainstorming/` |
| `skills/diagnosing-superpowers/` | `skills/diagnosing-superpowers/` |
| `skills/dispatching-parallel-agents/` | `skills/dispatching-parallel-agents/` |
| `skills/executing-plans/` | `skills/executing-plans/` |
| `skills/finishing-a-development-branch/` | `skills/finishing-a-development-branch/` |
| `skills/receiving-code-review/` | `skills/receiving-code-review/` |
| `skills/requesting-code-review/` | `skills/requesting-code-review/` |
| `skills/subagent-driven-development/` | `skills/subagent-driven-development/` |
| `skills/systematic-debugging/` | `skills/systematic-debugging/` (minus A.3 exclusions) |
| `skills/test-driven-development/` | `skills/test-driven-development/` |
| `skills/using-git-worktrees/` | `skills/using-git-worktrees/` |
| `skills/using-superpowers/` | `skills/using-superpowers/` (including `references/`) |
| `skills/verification-before-completion/` | `skills/verification-before-completion/` |
| `skills/writing-plans/` | `skills/writing-plans/` |
| `skills/writing-skills/` | `skills/writing-skills/` |
| `hooks/` | `hooks/` (`hooks.json`, `hooks-cursor.json`, `run-hook.cmd`, `session-start`) |
| `tests/brainstorm-server/` | `tests/brainstorm-server/` |

### A.3 Do NOT copy

- `skills/systematic-debugging/CREATION-LOG.md`, `test-academic.md`, `test-pressure-1.md`, `test-pressure-2.md`, `test-pressure-3.md` — upstream test fixtures, excluded since the original absorption (absorption spec: "systematic-debugging/ test pressure files ... and CREATION-LOG.md"; every later sync commit says "Upstream systematic-debugging test fixtures excluded as before").
- Everything else outside A.2: upstream `README.md`, `RELEASE-NOTES.md`, `docs/`, `commands/`, `agents/`, `AGENTS.md`, `GEMINI.md`, `gemini-extension.json`, `.cursor-plugin/`, `.opencode/`, `.codex/`, `.antigravity-plugin/`, `.claude-plugin/`, `.github/`, `.gitignore`, `.gitattributes`, `index.js`, `package.json`, `scripts/`, `assets/`, and all test suites other than `tests/brainstorm-server/`.
- Note: an `agents/code-reviewer.md` was vendored at v5.0.5 and removed when upstream folded it into `requesting-code-review/code-reviewer.md` (v5.1.0 era, commit `6a0a6cc4` "remove legacy code-reviewer agent"). Do not recreate it.
- Note: `hooks/hooks-codex.json` and `hooks/session-start-codex` were vendored at v6.0.2 and deleted in `d94a8425` because upstream v6.3.0 dropped them ("Codex triggers skills natively"). They do not exist in v7.0.0.

### A.4 Expected resulting file set (with git modes and line counts at HEAD)

Mode `100755` must be set explicitly (commit `edd1c960`: the index had stored them `100644` under `core.fileMode=false`, so fresh checkouts such as the daily-driver worktree got non-executable helpers and `task-start`'s direct call to `task-brief` failed with "Permission denied"). On Windows: `git update-index --chmod=+x <path>`.

| Mode | Lines | Path |
|---|---|---|
| 100644 | 10 | `hooks/hooks-cursor.json` |
| 100644 | 17 | `hooks/hooks.json` |
| **100755** | 62 | `hooks/run-hook.cmd` |
| **100755** | 85 | `hooks/session-start` |
| 100644 | 225 | `skills/brainstorming/SKILL.md` |
| 100644 | 23 | `skills/brainstorming/builder-check-prompt.md` |
| 100644 | 299 | `skills/brainstorming/visual-companion.md` |
| 100644 | 213 | `skills/brainstorming/scripts/frame-template.html` |
| 100644 | 167 | `skills/brainstorming/scripts/helper.js` |
| 100644 | 723 | `skills/brainstorming/scripts/server.cjs` |
| **100755** | 209 | `skills/brainstorming/scripts/start-server.sh` |
| **100755** | 120 | `skills/brainstorming/scripts/stop-server.sh` |
| 100644 | 120 | `skills/diagnosing-superpowers/SKILL.md` |
| 100644 | 38/28/29/26/30/20/33/29/38/30/28 | `skills/diagnosing-superpowers/prompts/{analyst-common,cost-and-time,plan-adherence,quality-evidence,repeated-work,request-conflicts,scrub-audit,scrub,similar-session,skill-timeline,stumbles}.md` |
| 100644 | 22/47/34/31 | `skills/diagnosing-superpowers/references/{context-safety,github-issues,redaction-policy,session-discovery}.md` |
| 100644 | 77/64/51/82 | `skills/diagnosing-superpowers/templates/{bundle-README,case,issue,report}.md` |
| 100644 | 167 | `skills/dispatching-parallel-agents/SKILL.md` |
| 100644 | 373 | `skills/executing-plans/SKILL.md` |
| **100755** | 52 | `skills/executing-plans/scripts/task-done` |
| **100755** | 28 | `skills/executing-plans/scripts/task-start` |
| 100644 | 225 | `skills/finishing-a-development-branch/SKILL.md` |
| 100644 | 205 | `skills/receiving-code-review/SKILL.md` |
| 100644 | 95 | `skills/requesting-code-review/SKILL.md` |
| 100644 | 198 | `skills/requesting-code-review/code-reviewer.md` |
| 100644 | 568 | `skills/subagent-driven-development/SKILL.md` |
| 100644 | 154 | `skills/subagent-driven-development/implementer-prompt.md` |
| 100644 | 115 | `skills/subagent-driven-development/re-review-prompt.md` |
| 100644 | 207 | `skills/subagent-driven-development/task-reviewer-prompt.md` |
| **100755** | 53 | `skills/subagent-driven-development/scripts/review-package` |
| **100755** | 87 | `skills/subagent-driven-development/scripts/sdd-workspace` |
| **100755** | 43 | `skills/subagent-driven-development/scripts/task-brief` |
| 100644 | 283 | `skills/systematic-debugging/SKILL.md` |
| 100644 | 158 | `skills/systematic-debugging/condition-based-waiting-example.ts` |
| 100644 | 115 | `skills/systematic-debugging/condition-based-waiting.md` |
| 100644 | 122 | `skills/systematic-debugging/defense-in-depth.md` |
| **100755** | 72 | `skills/systematic-debugging/find-polluter.sh` |
| 100644 | 169 | `skills/systematic-debugging/root-cause-tracing.md` |
| 100644 | 330 | `skills/test-driven-development/SKILL.md` |
| 100644 | 198 | `skills/test-driven-development/writing-good-tests.md` |
| 100644 | 167 | `skills/using-git-worktrees/SKILL.md` |
| 100644 | 65 | `skills/using-superpowers/SKILL.md` |
| 100644 | 23/29/108/63/56/35/16 | `skills/using-superpowers/references/{antigravity,claude-code,codex,gemini,hermes,muse,pi}-tools.md` |
| 100644 | 120 | `skills/verification-before-completion/SKILL.md` |
| 100644 | 204 | `skills/writing-plans/SKILL.md` |
| 100644 | 681 | `skills/writing-skills/SKILL.md` |
| 100644 | 1150 | `skills/writing-skills/anthropic-best-practices.md` |
| 100644 | 189 | `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` |
| 100644 | 171 | `skills/writing-skills/graphviz-conventions.dot` |
| 100644 | 187 | `skills/writing-skills/persuasion-principles.md` |
| **100755** | 169 | `skills/writing-skills/render-graphs.js` |
| 100644 | 384 | `skills/writing-skills/testing-skills-with-subagents.md` |
| 100644 | 312/344/66/197/515/607/407 | `tests/brainstorm-server/{auth,branding,browser-launcher,helper,lifecycle,server,ws-protocol}.test.js` |
| 100644 | 115/182/394 | `tests/brainstorm-server/{start-server,stop-server,windows-lifecycle}.test.sh` |
| 100644 | 10 / 36 | `tests/brainstorm-server/package.json` / `package-lock.json` (`ws` `^8.21.0`, locked `8.21.0`) |
| 100644 | 44 | `LICENSE-superpowers` (argus-authored header + upstream MIT text) |

Note: the v7.0.0 update plan's "restore exec bits" step listed `server.cjs` among files to `chmod +x`, but the committed mode of `skills/brainstorming/scripts/server.cjs` is `100644`. Follow the table above (the git index), not the plan step.

### A.5 Line endings

The repo's `.gitattributes` (argus-owned, not vendored) normalizes everything to LF and forces CRLF for `*.cmd`/`*.bat`:

```
* text=auto eol=lf
*.cmd text eol=crlf
*.bat text eol=crlf
```

So `hooks/run-hook.cmd` is checked out with CRLF. That CRLF is exactly why argus's registered hook invokes `bash .../hooks/session-start` directly instead of upstream's `run-hook.cmd` polyglot (see section E).

### A.6 `LICENSE-superpowers`

Keep this file at the repo root. It is an argus-authored header followed by the verbatim upstream MIT license. The header must read exactly:

```
The following files were copied from the "superpowers" Claude Code plugin
(v7.0.0) by Jesse Vincent and are used under the MIT License below:

  skills/brainstorming/
  skills/diagnosing-superpowers/
  skills/dispatching-parallel-agents/
  skills/executing-plans/
  skills/finishing-a-development-branch/
  skills/receiving-code-review/
  skills/requesting-code-review/
  skills/subagent-driven-development/
  skills/systematic-debugging/
  skills/test-driven-development/
  skills/using-git-worktrees/
  skills/using-superpowers/
  skills/verification-before-completion/
  skills/writing-plans/
  skills/writing-skills/
  hooks/
  tests/brainstorm-server/

---

MIT License

Copyright (c) 2025 Jesse Vincent
```

followed by the standard MIT permission and warranty text (copy it from upstream `LICENSE`). On any future version bump, update the `(v7.0.0)` string here, in `CLAUDE.md` ("Absorbed Skills (from superpowers v7.0.0)"), in `README.md` ("(v7.0.0, MIT ...)"), and in `scripts/setup.js`'s conflict message ("currently v7.0.0").

### A.7 Verify

- `cd tests/brainstorm-server && npm install && npm test` — a separate subproject with its own runner (`node` test files plus `bash` test scripts). The root `vitest.config` excludes `tests/brainstorm-server/**`.
- `git diff --summary` shows no unexpected mode-only diffs.
- `diff -r --strip-trailing-cr` of each A.2 directory against the upstream clone shows only the A.3 exclusions.

---

## B. Per-skill table

All 15 are **imported as-is** from upstream v7.0.0. "Files" lists the supporting files besides `SKILL.md`. Descriptions are the verbatim `description:` frontmatter values (`name:` always equals the directory name).

| Skill | Verbatim description | Supporting files | Status |
|---|---|---|---|
| `brainstorming` | `"You MUST use this before any creative work - creating features, building components, adding functionality, modifying behavior, or planning anything new, in software or out of it (a talk, a business, a renovation)."` (quoted in YAML) | `builder-check-prompt.md`, `visual-companion.md`, `scripts/{server.cjs,helper.js,frame-template.html,start-server.sh,stop-server.sh}` | as-is |
| `diagnosing-superpowers` | Use when a superpowers session went wrong and your human partner wants to know why — repeated work, ignored plans, stumbles, poor results, a skill that didn't fire, "it took too long", "why is it so expensive", "what is it doing" — or wants to build a bug report for the superpowers maintainers, for the current session or a past one identified by id or path, on any harness. | `prompts/` (11), `references/` (4), `templates/` (4) | as-is |
| `dispatching-parallel-agents` | Use when facing 2+ independent tasks that can be worked on without shared state or sequential dependencies | none | as-is |
| `executing-plans` | Use when executing an implementation plan in the current session as the implementer yourself — your human partner chose inline execution, or no subagent tool is available | `scripts/task-start`, `scripts/task-done` | as-is |
| `finishing-a-development-branch` | Use when implementation is complete, all tests pass, and you need to decide how to integrate the work | none | as-is |
| `receiving-code-review` | Use when receiving code review feedback, before implementing suggestions, especially if feedback seems unclear or technically questionable - requires technical rigor and verification, not performative agreement or blind implementation | none | as-is |
| `requesting-code-review` | Use when completing tasks, implementing major features, or before merging to verify work meets requirements | `code-reviewer.md` | as-is |
| `subagent-driven-development` | Use when executing implementation plans with independent tasks in the current session | `implementer-prompt.md`, `task-reviewer-prompt.md`, `re-review-prompt.md`, `scripts/{sdd-workspace,task-brief,review-package}` | as-is |
| `systematic-debugging` | Use when encountering any bug, test failure, or unexpected behavior, before proposing fixes | `root-cause-tracing.md`, `defense-in-depth.md`, `condition-based-waiting.md`, `condition-based-waiting-example.ts`, `find-polluter.sh` | as-is (5 upstream fixture files excluded, A.3) |
| `test-driven-development` | Use when implementing any feature or bugfix, before writing implementation code | `writing-good-tests.md` | as-is |
| `using-git-worktrees` | Use when starting feature work that needs isolation from current workspace or before executing implementation plans - ensures an isolated workspace exists via native tools or git worktree fallback | none | as-is |
| `using-superpowers` | Use when starting any conversation - establishes how to find and use skills, requiring skill invocation before ANY response including clarifying questions | `references/{antigravity,claude-code,codex,gemini,hermes,muse,pi}-tools.md` | as-is |
| `verification-before-completion` | Use when about to claim work is complete, fixed, or passing, before committing or creating PRs - requires running verification commands and confirming output before making any success claims; evidence before assertions always | none | as-is |
| `writing-plans` | Use when you have a spec or requirements for a multi-step task, before touching code | none | as-is |
| `writing-skills` | Use when creating new skills, editing existing skills, or verifying skills work before deployment | `anthropic-best-practices.md`, `persuasion-principles.md`, `testing-skills-with-subagents.md`, `graphviz-conventions.dot`, `render-graphs.js`, `examples/CLAUDE_MD_TESTING.md` | as-is |

Also as-is: all of `hooks/` and all of `tests/brainstorm-server/`.

---

## C. Local modifications (history and current status)

### C.1 Commit history of the vendored tree

| Commit | Date | Subject | Kind |
|---|---|---|---|
| `a9e10da8` | 2026-03-18 | feat: absorb superpowers plugin skills, agents, and hooks | **Import** (v5.0.5, 14 skills + `agents/code-reviewer.md` + hooks + server tests; "Files are unmodified from source.") |
| `b3b7ac8d` | 2026-03-18 | chore: remove dead platform references from absorbed skills | Local edit (overwritten later) |
| `a94934c3` | 2026-05-12 | feat: update superpowers skills to v5.1.0 | Upstream sync ("Straight copy-over") |
| `6a0a6cc4` | 2026-05-12 | chore: update hooks and remove legacy code-reviewer agent | Upstream sync (hooks; adds `hooks-cursor.json`) |
| `d05585f6`, `89c8356b`, `646dffa8`, `46ad46a1` | 2026-06-17 | update skills / hooks / tests / docs to v6.0.2 | Upstream sync |
| `bcb8e107` | 2026-07-15 | fix(deps): bump ws to 8.21.1 in brainstorm-server tests | Local edit (overwritten later) |
| `6e3efb87` | 2026-07-29 | docs(adversary-review): sync CLAUDE.md and cross-reference pointers | Local edit (overwritten later) |
| `20c0ce6e` | 2026-08-25 | feat: update superpowers skills to v6.3.0 | Upstream sync |
| `d94a8425` | 2026-08-25 | feat(setup): register the superpowers SessionStart hook in ~/.claude/settings.json | Hook tracking (matches upstream v6.3.0) + argus wiring |
| `a8ee6d6f` | 2026-08-25 | docs: superpowers v6.3.0 + SessionStart hook registration | License version string |
| `9c020baf` | 2026-09-28 | feat: update superpowers skills to v6.4.2 | Upstream sync (adds `diagnosing-superpowers`; "Local diff against upstream 6.3.0 was line endings only, so nothing was overwritten.") |
| `efc453ae` | 2026-09-28 | docs(license): superpowers notice names v6.4.2 and diagnosing-superpowers | License header |
| `edd1c960` | 2026-09-28 | fix(skills): restore upstream exec bits on vendored superpowers scripts | Mode-only (11 files to 100755) |
| `b08b10e0` | 2026-10-10 | feat: update superpowers skills to v7.0.0 | Upstream sync ("Straight copy-over from upstream v7.0.0 ... the other 10 are byte-identical to v6.4.2") |
| `42879d8b` | 2026-10-10 | docs: superpowers v7.0.0 | License/README/CLAUDE.md/setup.js version strings + update plan |

No commit after `b08b10e0` touches any vendored path except `LICENSE-superpowers` (`42879d8b`).

### C.2 The four historical local edits — none survive at HEAD

Do **not** re-apply any of these when rebuilding at v7.0.0; they are recorded so the history is understood and so a deliberate re-application is possible if wanted.

1. **`b3b7ac8d` — strip non-Claude platform references** (`using-superpowers/SKILL.md`, `writing-skills/SKILL.md`). Why: the absorption spec decided "Multi-platform adapters | Drop (Cursor, OpenCode, Gemini, Codex) | Only Claude Code is used". Delta (against v5.0.5):
   - In `using-superpowers/SKILL.md`: `(CLAUDE.md, GEMINI.md, AGENTS.md, direct requests)` → `(CLAUDE.md, direct requests)`; `If CLAUDE.md, GEMINI.md, or AGENTS.md says "don't use TDD"` → `If CLAUDE.md says "don't use TDD"`; the "In Claude Code / In Gemini CLI / In other environments" access paragraphs and the whole `## Platform Adaptation` section replaced with the single line `Use the \`Skill\` tool. When you invoke a skill, its content is loaded and presented to you—follow it directly. Never use the Read tool on skill files.`
   - In `writing-skills/SKILL.md`: `**Personal skills live in agent-specific directories (\`~/.claude/skills\` for Claude Code, \`~/.agents/skills/\` for Codex)**` → `**Personal skills live in \`~/.claude/skills\`.**`
   - **Status: overwritten** by the v5.1.0 straight copy (`a94934c3` replaced both files). HEAD's `using-superpowers/SKILL.md` contains upstream's `## Platform Adaptation` list and `(CLAUDE.md, AGENTS.md, GEMINI.md, etc, direct requests)`. The decision was also superseded: argus now installs into Claude Code, Codex, and Copilot CLI, so the platform references are wanted.

2. **`bcb8e107` — bump `ws` to 8.21.1** in `tests/brainstorm-server/package-lock.json` (Dependabot: GHSA ranges `< 8.21.0` / `< 8.20.1`). **Status: reverted** by the v6.3.0 copy (`20c0ce6e`), which set `package.json` to `"ws": "^8.21.0"` and the lock to `8.21.0` (upstream's values). HEAD has `8.21.0`, which is outside both advisory ranges quoted in the commit.

3. **`6e3efb87` — adversary-review cross-links.** Added to `finishing-a-development-branch/SKILL.md`, after "**If tests pass:** Continue to Step 2.":
   ```
   Optionally offer an adversary review (adversary-review skill) before presenting
   merge options — an independent cross-model check for higher-stakes branches.
   ```
   and to `requesting-code-review/SKILL.md`, after the "Optional but valuable" list:
   ```
   **Escalation:** for genuinely independent review by a different model, use the
   adversary-review skill (cross-model loop via Codex/Copilot/Claude CLI).
   ```
   **Status: overwritten** by the v6.3.0 copy (`20c0ce6e` replaced both files; `grep adversary` finds nothing in either at HEAD). If you want the cross-links back, re-apply these two blocks at the equivalent anchors — v7.0.0's anchors are "**If tests pass:** continue to Step 2." (lowercase "continue") in `finishing-a-development-branch` and the "- After fixing complex bug" bullet in `requesting-code-review` — and note that doing so creates the first real local delta.

4. **`d94a8425` — hooks tracked to upstream v6.3.0.** Added `"shell": "bash"` to `hooks/hooks.json` and deleted `hooks/hooks-codex.json` + `hooks/session-start-codex`. The commit says "Hooks tracked with upstream v6.3.0", i.e. these mirror upstream, not a local divergence; v7.0.0's `hooks.json` already contains `"shell": "bash"`.

### C.3 Evidence that HEAD equals upstream v7.0.0

- Every sync commit message says "Straight copy-over"; `9c020baf` explicitly reports the pre-sync local diff was line endings only.
- `docs/superpowers/plans/2026-10-10-superpowers-v7.0.0-update.md` scoped the v7.0.0 change by a byte-for-byte diff of both tags (`core.autocrlf=false`) and copied exactly the changed files: `brainstorming/SKILL.md`, `brainstorming/scripts/server.cjs`, new `brainstorming/builder-check-prompt.md` (replacing deleted `spec-document-reviewer-prompt.md`), `executing-plans/scripts/task-done`, `requesting-code-review/SKILL.md`, `subagent-driven-development/scripts/sdd-workspace`, `systematic-debugging/SKILL.md`, `hooks/session-start`, `hooks/run-hook.cmd`, `tests/brainstorm-server/{lifecycle,server}.test.js`.
- A local diff against an installed upstream **v7.0.1** plugin copy (not v7.0.0) shows `hooks/` and 9 skills byte-identical (ignoring CR). The differing files (`brainstorming/SKILL.md`, `diagnosing-superpowers/{SKILL.md,references/github-issues.md,templates/issue.md}`, `executing-plans/SKILL.md`, `subagent-driven-development/SKILL.md` + new `references/`, `writing-skills/SKILL.md` + support files moved into `references/`, `tests/brainstorm-server/server.test.js`) are all explained by v7.0.1's own release notes (brainstorming retune, compaction trims, issue-template changes). So nothing in the diff is a local edit. Do **not** use v7.0.1 for the rebuild: its `writing-skills` and `subagent-driven-development` layouts differ from what argus ships.

---

## D. FALLBACK — behavior spec if upstream is unavailable

Use this only if tag `v7.0.0` cannot be obtained. It condenses each skill to purpose, trigger, procedure, and hard rules (quoted verbatim). Rewritten skills should keep the exact frontmatter from section B, the same file names, and the cross-skill references (skills refer to each other as `superpowers:<name>`). Shared vocabulary: the user is "your human partner"; plans live in `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`, specs in `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`; per-plan scratch lives in the git-ignored `<repo-root>/.superpowers/sdd/<plan-basename>/`.

### D.1 using-superpowers (bootstrap, injected by the hook)

- **Purpose:** force skill lookup and invocation before any action.
- **Fires:** every session start, via the SessionStart hook. Subagents ignore it: `<SUBAGENT-STOP>If you were dispatched as a subagent to execute a specific task, ignore this skill.</SUBAGENT-STOP>`.
- **Hard rules (verbatim):**
  - "If you think there is even a 1% chance a skill might apply to what you are doing, you ABSOLUTELY MUST invoke the skill."
  - "IF A SKILL APPLIES TO YOUR TASK, YOU DO NOT HAVE A CHOICE. YOU MUST USE IT."
  - "**Invoke relevant or requested skills BEFORE any response or action** — including clarifying questions, exploring the codebase, or checking files. If it turns out wrong for the situation, you don't have to use it."
  - "**Before entering plan mode:** if you haven't already brainstormed, invoke the brainstorming skill first."
  - "Then announce \"Using [skill] to [purpose]\" and follow the skill exactly. If it has a checklist, create a todo per item."
- **Priority:** process skills first, then implementation skills ("Let's build X" → brainstorming first; "Fix this bug" → systematic-debugging first).
- **Red Flags table:** 12 rationalizations, each refuted (e.g. "This is just a simple question" → "Questions are tasks. Check for skills."; "I remember this skill" → "Skills evolve. Read current version.").
- **Platform Adaptation:** points at `references/<harness>-tools.md` for Claude Code, Codex, Pi, Antigravity, Hermes Agent, Muse. Each reference maps that harness's tool names (subagent dispatch, todo, skill loading) to the Claude Code names the skills use; Gemini has a reference file too.
- **User instructions:** "User instructions (CLAUDE.md, AGENTS.md, GEMINI.md, etc, direct requests) take precedence over skills, which in turn override default behavior. Only skip skill workflows or instructions when your human partner has explicitly told you to."

### D.2 brainstorming

- **Purpose:** find out what the partner actually wants, and why, before anything gets built.
- **Procedure:**
  1. *Triage:* if everything that matters is in the request, it is a quick, clear task: do it and say what you did. Otherwise start a conversation.
  2. *First message* is exactly two things: one line offering to skip the questions (if accepted: state any significant how-it-gets-made choice in one line, then proceed; skill done), and one open question about a real moment or concrete picture.
  3. *Conversation:* each message ends with one single-sentence question, no list of example answers; plain language; open questions before menus; find out which how-it-gets-made choices (platform, language, libraries, medium) they want to own vs hand over, and state defaults out loud as your call. Ground to cover (territory, never read out): what it is, why now / success, goals / non-goals / anti-goals, prior art, current state, details they care about. Offer recon (codebase, files, web) and let them decide. Think wide privately. Offer a throwaway spike when it helps.
  4. *Play it back* in 200–300-word chunks as each part becomes clear, marking your guesses as guesses, ending each chunk by asking what is wrong or missing.
  5. *Size the work* and say it: quick task → do it; small change → described in chat, then normal workflow; project → written design, then a full plan (`superpowers:writing-plans`). Step up a size if it grows; split independent projects and do them one at a time.
  6. *Written design* (projects): intent and why, goals / non-goals / anti-goals, constraints, the decided parts of the how, what is left to the builder. Saved to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and committed (outside a repo, ask).
  7. *Builder check:* dispatch a fresh subagent with `builder-check-prompt.md` (a "talented [DOMAIN]" who cannot talk to the author lists, ranked, only the questions where a wrong guess would matter, or says "nothing important left"). Sort answers into three piles: answered already (put in doc), minor (settle yourself and mark as your call), theirs (ask once, most important first). Hand over the doc with a list of the calls you made. Without a subagent tool, do the check yourself.
- **Hard gate (verbatim):**
  ```
  <HARD-GATE>
  Nothing gets built until your human partner approves the full
  description: in chat for a small change, the document for a project.
  Building includes writing product code, scaffolding, installing
  dependencies, creating projects, and invoking an implementation skill.
  Reading and exploring are fine.

  Two exceptions. If they opted out of the questions, the skill is done
  and the gate goes with it. A spike they said yes to may be built; it
  stays labeled throwaway, and keeping what it produced is a new request
  that comes back through this gate.

  After a project's document is approved, the next step is the plan. For
  software, invoke superpowers:writing-plans and no other skill.
  </HARD-GATE>
  ```
- **Visual companion:** offered just-in-time, in its own message with nothing else: "This might be easier if I show you. Want me to open a browser tab with some mockups?" Declined → do not re-offer unless they raise it. Per question, use the browser only if seeing beats reading. On acceptance, read `visual-companion.md` before starting the server.
- **Visual companion mechanics** (`visual-companion.md` + `scripts/`): `bash scripts/start-server.sh --project-dir <root> --open` starts a zero-dependency Node HTTP + RFC 6455 WebSocket server (`server.cjs`) on a random high port bound to `127.0.0.1` (`--host`, `--url-host`, `--foreground`/`--background`, `--idle-timeout-minutes`, default 240) and prints JSON `{type:"server-started",port,url,screen_dir,state_dir}`, also written to `$STATE_DIR/server-info`. The URL carries a per-session key `?key=…`; the server rejects requests without it (cookie after first load), so always hand out the complete URL. Session files go under `<project>/.superpowers/brainstorm/<pid>-<epoch>/{content,state}` (or `/tmp` without `--project-dir`). Write a new semantically named HTML file per screen into `screen_dir` with your file tool (never reuse names; `-v2` suffixes); the newest file is served; fragments are wrapped in `frame-template.html`, full documents (`<!DOCTYPE`/`<html`) get only the `helper.js` client injected (auto-reconnect with backoff 500 ms to 30 s and a "paused" overlay). Browser clicks are appended to `$STATE_DIR/events` as JSON lines (`{"type":"click","choice":"a","text":…,"timestamp":…}`), cleared on each new screen. Before pushing, confirm `server-info` exists and `server-stopped` does not; restart with the same `--project-dir` to reuse the port. Push a "Continuing in terminal..." waiting screen when returning to text. `bash scripts/stop-server.sh <session_dir>` kills the server and deletes the directory only if under `/tmp`.

### D.3 writing-plans

- **Purpose:** turn a spec into a plan for "an engineer who has not seen this codebase or this spec", recording the decisions they cannot make alone. Announce "I'm using the writing-plans skill to create the implementation plan."
- **Procedure:** scope check (one plan per independent subsystem) → file structure map (focused files, split by responsibility) → right-sized tasks (smallest unit with its own test cycle worth a reviewer's gate) → steps that are one action with a checkable result → self-review → handoff.
- **Required header:** title `# [Feature Name] Implementation Plan`; the agentic-worker note `> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (\`- [ ]\`) syntax for tracking.`; **Goal**, **Architecture**, **Tech Stack**, **Spec** path; `## Global Constraints` (spec's project-wide values verbatim); `## Review Focus` (the five most likely-to-bite input classes or failure modes no task's tests exercise, each then pinned by a test added to its owning task).
- **Task shape:** `### Task N: [Component Name]`, **Files** (Create/Modify with line ranges/Test), **Interfaces** (Consumes / Produces with exact signatures), checkbox steps: write failing test (code with spec values) → run, Expected FAIL → implement `signature` in `file` (body only when signature and tests do not determine it) → run, Expected PASS → commit.
- **Rule (verbatim):** "A plan is the set of decisions the implementer cannot make alone. A plan longer than the code it describes has written the code instead."
- **Self-review checklist:** spec coverage, step scan (no decide-nothing lines, no transcribed bodies), type consistency, Review Focus, proportion.
- **Handoff:** link the plan, ask for review, and (unless already supplied) offer **Subagent-driven** vs **Native** execution with a one-sentence recommendation; then REQUIRED SUB-SKILL `superpowers:subagent-driven-development` or `superpowers:executing-plans`.

### D.4 executing-plans (inline / "Native")

- **Purpose:** the implementer executes every task in-session; one fresh whole-branch review at the end.
- **Setup:** isolated workspace via `superpowers:using-git-worktrees` (never on main/master without consent); workspace from `../subagent-driven-development/scripts/sdd-workspace PLAN_FILE`; ledger `<workspace>/progress.md` whose first line is `# SDD ledger — plan: <plan file path>` (resume from it; a ledger naming another plan is not yours); read plan + spec once; todo per task; load `superpowers:test-driven-development` before Task 1; pre-flight scan producing one ledger row per producer/consumer interface pair (or `Pre-flight: no shared interfaces`).
- **Task loop:** `scripts/task-start PLAN_FILE N` (prints `brief:` path and `base:` SHA) → read the brief → work steps in RED-GREEN order, comparing every `Expected:` line (code wrong → `superpowers:systematic-debugging`; plan wrong → rule and ledger `Task <N>: Ruling: …`) → commit → completion contract → `scripts/task-done PLAN_FILE N BASE -- <test command>`, which runs the tests and only on success appends `Task <N>: complete (commits <base7>..<head7>, tests: <command> → <result>)` (silent passes record `→ (no output)`).
- **Hard rules (verbatim):**
  - "**Continuous execution:** Do not pause to check in with your human partner between tasks."
  - "Four things stop you, and only these: an irreversible or destructive operation; a security-sensitive action; a side effect outside this worktree that norms say you ask about first (a merge, a push to a shared branch, a publish); and a plan so broken that every path forward is a guess. For those, stop and ask."
  - Ruling format: "`Ruling: <what you decided> — <why> — <what it costs if wrong>`" and "Deviating from the plan without a ledgered ruling is a decision made in secret."
- **Final review:** `../subagent-driven-development/scripts/review-package PLAN_FILE MERGE_BASE HEAD`; dispatch reviewer on the most capable model with `../requesting-code-review/code-reviewer.md`, the package, plan, spec, Review Focus, and the ledger's rulings (without a subagent tool: self-review, ledger `Final review: self-review (no subagent tool)` and say so). Re-grade findings by effect; Critical/Important get ONE fix pass, each fix RED→GREEN plus green suite, ledgered `Final: fixed …`; Minor → `Final: minor (deferred): …`; no re-review dispatch, no second pass.
- **Finish:** final message lists "Rulings I made" and "Deferred minors" exhaustively, delete this plan's workspace, then `superpowers:finishing-a-development-branch`.

### D.5 subagent-driven-development

- **Purpose:** fresh implementer subagent per task, a two-verdict task review (spec compliance + quality) per task, a broad final review.
- **Shared with D.4:** workspace, ledger format, continuous execution, rulings, the same four stops, pre-flight scan (here a table: one row per task pair sharing a file/interface and one self-consistency row per task, written to the ledger).
- **Model selection:** least powerful model per role; always specify the model explicitly ("An omitted model inherits your session's model"); mid-tier floor for reviewers and prose-driven implementers; cheapest tier when the plan contains complete code; most capable for the final review; fix rounds 4–5 go at least one tier up.
- **Task loop:** record BASE → `bash scripts/task-brief PLAN_FILE N` → dispatch with `implementer-prompt.md` (one context line, brief path "read this first", interfaces/decisions from earlier tasks, ambiguity resolutions, report path `task-N-report.md`; no pasted history; implementer never dispatches subagents) → handle status DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED → `bash scripts/review-package PLAN_FILE BASE HEAD` (never `HEAD~1`) → dispatch `task-reviewer-prompt.md` with brief, report, package, and verbatim global constraints; never pre-judge ("do not flag", "at most Minor"); resolve "⚠️ Cannot verify from diff" items yourself.
- **Fix loop:** triggered by spec ❌, Critical/Important, or confirmed ⚠️. Minors → `Task <N>: minor (deferred): …`; plan-conflicting findings → ruling. Max 5 rounds of (fix + scoped re-review via `re-review-prompt.md` over `FIX_BASE..HEAD`, verdict ADDRESSED / NOT ADDRESSED per finding); rounds 1–3 resume the original implementer, rounds 4–5 a fresh implementer on a stronger model; ledger `Task <N>: fix round <R>/5 (…)`. "Never fix findings yourself in the controller session". At the cap, adjudicate (park with `Ruling:` or rule on load-bearing ones); "Adjudicate only at the cap."
- **Completion:** `Task <N>: complete (commits <base7>..<head7>, review clean)` or `…, <K> parked)`. Never parallel implementers. Batch small same-shape edits into one dispatch.
- **Final review:** whole-branch package, most capable model, `code-reviewer.md`, pointed at deferred/parked lines; ONE fix subagent for all findings, one scoped re-review, adjudicate residuals; then the same Finish as D.4.
- **Prompt templates:** `implementer-prompt.md` (requires a model; contract: read brief, ask questions before and during work, implement, write tests (following TDD if the task says to), run the full suite once before committing, commit, self-review by reading its own diff, never dispatch a reviewer, write the full report to file, return status DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED with commits, test line, concerns); `task-reviewer-prompt.md` (reads the package once; returns Spec ✅/❌ and task-quality verdict with Strengths/Issues by Critical/Important/Minor); `re-review-prompt.md` (scoped, not a fresh review).
- **Scripts:** `sdd-workspace` (resolve/create `.superpowers/sdd/<plan-basename>/`, plan-path marker, slug disambiguation, writes default `.gitignore` only if none exists); `task-brief PLAN N [OUT]` (extract task text to `task-<N>-brief.md`); `review-package PLAN BASE HEAD [OUT]` (commit list + stat + `-U10` diff to `review-<base7>..<head7>.diff`, exit 3 on empty or non-rooted range).

### D.6 test-driven-development

- **Fires:** any feature, bugfix, refactor, behavior change (exceptions only with the partner's permission: throwaway prototypes, generated code, config).
- **Iron law (verbatim):** "NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST" — "Write code before the test? Delete it. Start over." with "Don't keep it as \"reference\" / Don't \"adapt\" it while writing tests / Don't look at it / Delete means delete".
- **Cycle:** RED (one minimal behavior, clear name, real code) → Verify RED ("**MANDATORY. Never skip.**": fails, not errors, for the expected reason) → GREEN (simplest code) → Verify GREEN (and "**\"Other tests\" means the project's suite, not just your file.**" — any failure seen, including ones you did not cause, goes in the report by name) → REFACTOR (green only).
- Also: rationalizations table, red-flags list ("All of these mean: Delete code. Start over with TDD."), verification checklist, "Never fix bugs without a test." `writing-good-tests.md`: name the production change that would make the test fail before writing it; assert on real behavior, never mock behavior; keep test-only code out of production classes; understand a dependency's side effects before mocking it.

### D.7 systematic-debugging

- **Iron law (verbatim):** "NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST".
- **Four phases, each completed before the next:** (1) root cause: read errors fully, reproduce, check recent changes, instrument every component boundary in multi-component systems (example prints only whether a secret is SET/UNSET, never its value), trace data flow backward (`root-cause-tracing.md`); (2) pattern: find working examples, read references completely, list every difference, understand dependencies; (3) hypothesis: one written hypothesis, smallest change, one variable; (4) implementation: failing test first (`superpowers:test-driven-development`), single fix, verify (`superpowers:verification-before-completion`).
- **Three-strikes rule (verbatim):** "**If ≥ 3: STOP and question the architecture (step 5 below)**" — "DON'T attempt Fix #4 without architectural discussion" and "**Discuss with your human partner before attempting more fixes**".
- Supporting docs: `defense-in-depth.md` (validate at multiple layers after finding the cause), `condition-based-waiting.md` + `.ts` example (poll conditions instead of sleeps), `find-polluter.sh` (bisect which test creates unwanted state).

### D.8 verification-before-completion

- **Iron law (verbatim):** "NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE" — "If you haven't run the verification command in this message, you cannot claim it passes."
- **Gate:** IDENTIFY the command → RUN it fully and fresh → READ output and exit code → VERIFY it confirms the claim → only then claim, with evidence. "Skip any step = lying, not verifying".
- Claim/evidence table (tests, lint, build, bug fixed, regression test red-green, agent completed → VCS diff, requirements → line-by-line checklist). Red flags: "should", "probably", premature "Done!", trusting agent reports. Applies to paraphrases and implications of success.

### D.9 requesting-code-review

- **When:** mandatory after each SDD task, after a major feature, before merge to main; optional when stuck, before refactors, after a complex bug.
- **How:** `BASE_SHA` / `HEAD_SHA`; dispatch a `general-purpose` subagent filling `code-reviewer.md` placeholders `{DESCRIPTION}`, `{PLAN_OR_REQUIREMENTS}`, `{BASE_SHA}`, `{HEAD_SHA}`; never hand it your session history. Act: fix Critical immediately, Important before proceeding, note Minor, push back with reasoning when wrong. The example derives the base with `cut -d' ' -f1` (no `$1`, which Claude Code would substitute with skill arguments).
- `code-reviewer.md`: "Senior Code Reviewer" prompt; output Strengths / Issues (Critical (Must Fix), Important, Minor, each with a file:line reference) / Recommendations / Assessment; where the spec is silent it judges by what a reasonable person using the software would expect, and lists behavior it would not judge under a `## Declined to judge` heading (consumed by executing-plans' final-review rulings).

### D.10 receiving-code-review

- **Pattern (verbatim block):** READ, UNDERSTAND, VERIFY, EVALUATE, RESPOND, IMPLEMENT ("One item at a time, test each").
- **Forbidden (verbatim):** "\"You're absolutely right!\" (explicit instruction-file violation)", "\"Great point!\" / \"Excellent feedback!\" (performative)", "\"Let me implement that now\" (before verification)"; no gratitude expressions ("If you catch yourself about to write \"Thanks\": DELETE IT. State the fix instead.").
- Unclear items: stop and clarify all before implementing any. External reviewers: check correctness for this codebase, breakage, reasons for current code, platforms, reviewer context; YAGNI-check "implement properly" suggestions by grepping usage. Order: blocking, simple, complex; test each. Push back with technical reasoning; correct your own wrong pushback factually. GitHub inline replies go in the thread (`gh api repos/{owner}/{repo}/pulls/{pr}/comments/{id}/replies`).

### D.11 dispatching-parallel-agents

- **When:** 2+ independent problem domains (e.g. failures in different test files/subsystems) with no shared state; not when failures are related, full-system context is needed, or agents would edit the same files.
- **Pattern:** group by domain → one focused, self-contained prompt per domain (scope, goal, constraints, expected output) → issue all dispatches in the same response ("Multiple dispatch calls in one response = parallel execution. One per response = sequential.") → review summaries, check for conflicting edits, run the full suite, spot-check.

### D.12 using-git-worktrees

- **Step 0:** detect existing isolation (`git rev-parse --git-dir` vs `--git-common-dir`, guarding against submodules via `--show-superproject-working-tree`); already in a worktree → skip creation. In a normal checkout, honor a declared preference or ask consent ("Would you like me to set up an isolated worktree? It protects your current branch from changes.").
- **Step 1a:** use the harness's native worktree tool if any (e.g. `EnterWorktree`); "Using `git worktree add` when you have a native tool creates phantom state your harness can't see or manage."
- **Step 1b fallback:** directory priority = declared preference > existing `.worktrees/` > `worktrees/` > default `.worktrees/`; MUST `git check-ignore` it (else add to `.gitignore` and commit); `git worktree add "$path" -b "$BRANCH_NAME"`; on sandbox permission errors, work in place and say so.
- **Steps 2–3:** auto-detect setup (npm/cargo/pip/poetry/go) and run baseline tests; failures → report and ask.

### D.13 finishing-a-development-branch

- Announce "I'm using the finishing-a-development-branch skill to complete this work."
- **Steps:** run the full suite (failures → report and stop) → detect environment (capture `GIT_DIR`, `GIT_COMMON`, `WORKTREE_PATH` before any `cd`) → determine and confirm the base branch → present exactly the menu → execute → cleanup.
- **Menu (verbatim, normal repo / named-branch worktree):**
  ```
  Implementation complete. What would you like to do?

  1. Merge back to <base-branch> locally
  2. Push and create a Pull Request
  3. Keep the branch as-is (I'll handle it later)

  Which option?
  ```
  Detached HEAD: two options (push as new branch + PR; keep as-is).
- Merge: from main root, checkout base, pull, merge, re-run tests on the merged result (fail → stop, leave everything), then cleanup and `git branch -d`. PR: push, create PR with the forge's tooling, keep the worktree. Discard only on explicit request, with a listing and the exact typed word `discard`, then `git branch -D`.
- **Cleanup:** only worktrees under `.worktrees/` or `worktrees/`; `git worktree remove` + `prune`; if refused for modified/untracked files, never `--force` on your own initiative — show `git status --porcelain -uall` and offer commit / move / delete. Never force-push without explicit request.

### D.14 writing-skills

- **Core idea (verbatim):** "**Writing skills IS Test-Driven Development applied to process documentation.**" Iron law: "NO SKILL WITHOUT A FAILING TEST FIRST" — applies to new skills and edits.
- **RED:** run pressure scenarios on subagents without the skill and record rationalizations verbatim. **GREEN:** minimal skill addressing those failures. **REFACTOR:** close loopholes, rationalization table, red flags; re-test.
- **Discovery rules:** description = triggering conditions only, starting "Use when...", third person, never a workflow summary (agents follow the summary instead of the body); keyword coverage; token budgets (getting-started skills under 150 words each, other frequently loaded ones under 200 total).
- **Match the form to the failure:** discipline failures → prohibition + rationalization table + red flags; wrong-shaped output → positive recipe/contract; omitted element → required template slot; conditional behavior → condition on an observable predicate. No nuance clauses; exemption clauses don't scope. Micro-test wording (fresh-context samples, no-guidance control, 5+ reps, read every flagged match).
- Checklist (todo per item) and "STOP: Before Moving to Next Skill". Supporting files: `anthropic-best-practices.md` (official authoring guidance), `testing-skills-with-subagents.md` (pressure-scenario method), `persuasion-principles.md` (authority/commitment/scarcity/social proof/unity), `graphviz-conventions.dot` + `render-graphs.js` (flowchart style and rendering), `examples/CLAUDE_MD_TESTING.md`.

### D.15 diagnosing-superpowers

- **Purpose:** report, with `path:line` evidence from session transcripts, what went wrong in a superpowers session. "**Core principle:** Every finding cites `path:line`. No citation, no finding."
- **Workflow (todo per step):** (1) problem intake, one question at a time, until there is a statement of sessions, turn range, expected vs actual, observable; (2) locate transcripts per `references/session-discovery.md`, create `~/.superpowers/diagnosing-superpowers/<session-id>/`, fill `templates/case.md`; (3) triage: read the region yourself, then dispatch seven parallel analysts (`prompts/analyst-common.md` + one of `skill-timeline`, `plan-adherence`, `repeated-work`, `stumbles`, `quality-evidence`, `request-conflicts`, `cost-and-time`), discard uncited findings; (4) fill `templates/report.md`; (5) GitHub issues per `references/github-issues.md`, only with approval of exact text; (6) export a scrubbed bundle only on request (`templates/bundle-README.md`, `prompts/scrub.md` then `prompts/scrub-audit.md` until CLEAN; archive only after approval); (7) similar sessions on request (`prompts/similar-session.md`).
- **Hard rules (verbatim headings):** "**Context safety.**" (follow `references/context-safety.md`), "**Read-only.** Never modify, move, or delete a session file.", "**Exact paths to subagents.**", "**Human prompts only.**", "**No superpowers diagnosis.** Report §7 states involvement and stops. Never name a defect in a skill or propose a change.", "**Approval gates.**", "**Intake before analysis.**". `references/redaction-policy.md` governs scrubbing.

### D.16 hooks and tests (fallback)

- `hooks/session-start` (bash): resolve its own directory without `dirname` (split `$0` on the last `/` or `\`), read `../skills/using-superpowers/SKILL.md` with `$(<file)` (no `cat`), JSON-escape with parameter substitution, wrap as `<EXTREMELY_IMPORTANT>\nYou have superpowers.\n\n**Below is the full content of your 'superpowers:using-superpowers' skill - your introduction to using skills. For all other skills, use the 'Skill' tool:**\n\n<content>\n</EXTREMELY_IMPORTANT>`, and print one JSON shape chosen by env: `CURSOR_PLUGIN_ROOT` → `{"additional_context":…}`; `CLAUDE_PLUGIN_ROOT` set and neither `COPILOT_CLI` nor `MUSE_PLUGIN_ROOT` → `{"hookSpecificOutput":{"hookEventName":"SessionStart","additionalContext":…}}`; `MUSE_PLUGIN_ROOT` → same nested shape; otherwise → `{"additionalContext":…}`. Use `printf` (not a heredoc, bash 5.3 hang) piped through `cat` only when `cat` is on PATH (absorbs Windows EPIPE). `exit 0`.
- `hooks/hooks.json`: `SessionStart`, matcher `startup|clear|compact`, command `"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd" session-start`, `"shell": "bash"`, `"async": false`. `hooks/hooks-cursor.json`: `{"version":1,"hooks":{"sessionStart":[{"command":"./hooks/run-hook.cmd session-start"}]}}`. `hooks/run-hook.cmd`: cmd/bash polyglot that finds Git for Windows bash (including `%LOCALAPPDATA%\Programs\Git`, skipping WSL `bash.exe` and ignoring `bash`/`where` in the current directory), runs the named hook, and propagates its exit code; on Unix the bash half runs the hook directly.
- `tests/brainstorm-server/`: `npm test` runs `ws-protocol`, `helper`, `browser-launcher`, `auth` (session key), `branding`, `server`, `lifecycle` node tests, then `start-server.test.sh` and `stop-server.test.sh`; dependency `ws ^8.21.0` (client side only; the server itself has zero dependencies).

---

## E. Hook wiring (argus-owned code, not vendored)

### E.1 Why

Argus is not a Claude Code plugin (the old `.claude-plugin/plugin.json` was removed), and Claude Code reads `hooks/hooks.json` only from inside a plugin. So the vendored `hooks.json` is inert, kept for upstream fidelity. Commit `d94a8425` restored the bootstrap by registering the hook in the user's Claude `settings.json` instead ("the using-superpowers bootstrap has silently not fired since").

### E.2 Code: `scripts/lib/hooks.js`

Pure helpers plus IO wrappers:

- `SESSION_START_MATCHER = 'startup|clear|compact'` (same as upstream).
- `COMMAND_TAIL = '/hooks/session-start"'`.
- `sessionStartCommand(worktreePath, pathFormat)`: `root = toPathFormat(worktreePath, pathFormat).replace(/\\/g, '/')` (forward slashes even on Windows because the command runs under Git Bash), returns
  ```
  CLAUDE_PLUGIN_ROOT="<root>" bash "<root>/hooks/session-start"
  ```
- `isArgusSessionStartHook(hook)`: `command` is a string that starts with `CLAUDE_PLUGIN_ROOT=` and ends with `COMMAND_TAIL` (matches any worktree path, so stale entries are recognized).
- `buildSessionStartGroup(command)` → `{ matcher: 'startup|clear|compact', hooks: [{ type: 'command', command, shell: 'bash', async: false }] }`.
- `removeSessionStartHook(settings)` (pure): drops every argus hook from `hooks.SessionStart` groups, keeps foreign hooks and all other keys, drops emptied groups, removes empty `SessionStart`/`hooks` containers; returns `{ settings, changed }`.
- `upsertSessionStartHook(settings, command)` (pure): no-op if exactly one argus hook exists with the same command and `shell === 'bash'`; otherwise strip argus hooks and append one fresh group (stale worktree path replaced).
- `registerSessionStartHook({ home, worktreePath, pathFormat }, io)`: reads `<home>/settings.json` (missing → `{}`; unparseable → never overwritten, prints `⚠ <path> is not valid JSON — add the SessionStart hook yourself:` plus the exact JSON `{"hooks":{"SessionStart":[<group>]}}` and returns false); writes `JSON.stringify(settings, null, 2) + '\n'` when changed; logs `SessionStart hook: already registered — skipping` or `SessionStart hook registered: <settings> → <worktree>/hooks/session-start`. Never throws.
- `unregisterSessionStartHook({ home }, io)`: symmetric removal, silent when nothing is registered. Used by uninstall (`scripts/lib/uninstall-hooks.js` plans the same `removeSessionStartHook` edit behind the uninstall gates).

### E.3 The three non-obvious choices (from the module header and CLAUDE.md)

1. **`CLAUDE_PLUGIN_ROOT` is set inline.** Claude Code exports it only for plugin hooks, and `session-start` picks its output shape from it. Without it the script emits the Copilot/SDK `additionalContext` shape, which Claude Code ignores. Setting it on the command line keeps the vendored script pristine.
2. **It points at the daily-driver worktree, not the main clone.** The script reads `skills/using-superpowers/SKILL.md` relative to itself, and skills are linked from the worktree, so the hook must follow the same tree.
3. **`bash .../session-start` directly, not `run-hook.cmd`.** With `shell: "bash"` (upstream's v6.2.0 fix: Claude Code ≥ 2.1.81 hands such commands to Git Bash on Windows; older versions ignore the key) the command is always in bash, so the cmd-to-bash dispatcher is redundant. And `.gitattributes` checks `*.cmd` out with CRLF, which breaks the polyglot's bash half (`CMDBLOCK\r` never terminates the heredoc).

### E.4 Call site and gating

`scripts/lib/linker.js` `linkAll(opts)`: after `linkSkills(opts)`, if `clientId === 'claude' && canLinkSkillsToTarget(opts)` it calls `registerSessionStartHook({ home: opts.claudeHomePath, worktreePath: opts.worktreePath, pathFormat: opts.pathFormat }, makeConsoleIo())`, then registers the MCP server. Consequences: only Claude Code targets get the hook (Codex and Copilot discover skills natively; upstream removed its Codex hook in v6.1.0), and a target that cannot see the worktree (WSL ext4 worktree → Windows-format target) gets neither skill links nor the hook. `linkAll` runs from both `npm run setup` and `npm run fix-links`, so both register or refresh the hook idempotently.

### E.5 Related setup behavior

- `detectSuperpowersConflicts(claudeHomePath)` (`linker.js`): flags any truthy `enabledPlugins` key containing "superpowers" in `settings.json`, and any `<home>/skills/superpowers` entry. `scripts/setup.js` (step "Superpowers conflict check") prints "Argus-Panoptes includes all superpowers skills (currently v7.0.0)." and "Running both will cause duplicate skills, hooks, and agents.", advises disabling the plugin, and asks `Continue anyway? (y/n)`.
- `scripts/lib/uninstall-catalog.js` lists the 15 skill names under `REMOVED.superpowers`; uninstall removes their links and the SessionStart hook, then tells the user superpowers must be reinstalled as a plugin.
- Tests: `tests/scripts/hooks.test.ts` (pure helpers and register/unregister), `tests/scripts/uninstall-hooks.test.ts` (uninstall planning of the hook removal).
