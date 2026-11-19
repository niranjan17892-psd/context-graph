# Skill: gh-pr

Deprecation stub. It exists only so that `/gh-pr` still resolves; its sole instruction is to invoke `open-pr`. The pre-description code review the old skill used to run was dropped (covered upstream by task/branch reviewers and downstream by the bots).

## Files to create (every path + purpose)

| Path | Purpose |
|---|---|
| `skills/gh-pr/SKILL.md` | The entire skill: frontmatter + a short redirect body. No scripts, no other files. |

## Frontmatter (verbatim)

```yaml
---
name: gh-pr
description: Deprecated — use open-pr. Kept only so "/gh-pr" still resolves. Creating a pull request now goes through the open-pr skill; post-open bot reviews go through resolve-pr-reviews.
---
```

## Purpose; fires when / does not

- **Fires when** the user types `/gh-pr` (or a client matches its description).
- **Does not** create a PR, review code, or write a description itself. It hands off immediately.

## Procedure (numbered, faithful; every gate, cap, stop condition)

1. Invoke the `open-pr` skill now. (Post-open bot reviews belong to `resolve-pr-reviews`, which `open-pr` offers at its end.)

## Hard rules (critical ones VERBATIM)

The body, verbatim (this is the whole file after the frontmatter):

```markdown
# gh-pr (deprecated)

This skill has been replaced by `open-pr` (create the PR) and
`resolve-pr-reviews` (handle bot reviews after it opens). The code review this
skill used to run before writing the description is covered upstream by the
task and branch reviewers and downstream by the bots, so it was dropped.

Invoke the `open-pr` skill now.
```

## Scripts

None.

## Tests to write (list the behaviors the existing tests pin)

No dedicated test. `gh-pr` is listed as `retired` (not a kept-skill offer) in `scripts/lib/uninstall-catalog.js`, so `npm run uninstall` removes its link and `skills-self-contained.test.ts` does not scan it. Optional: assert the file contains `open-pr` and the word `Deprecated` in its description.

## Output format (templates verbatim)

No output of its own beyond the redirect; the user-visible result is whatever `open-pr` produces.

## Acceptance check (2-3 scenarios)

1. User types `/gh-pr`: the agent loads this stub and immediately runs the `open-pr` procedure (announce line "I'm using the open-pr skill to create the pull request.", the verbatim assumption sentence, etc.) — no review is run and no PR is created by gh-pr itself.
2. The skill listing shows the description beginning "Deprecated — use open-pr.", so a fresh request "open a PR" matches `open-pr` directly rather than this stub.
