# Skill: open-pr

Creates the pull request for the current branch (or several branches across repos): writes `pr-description.md` from the repo's template, confirms title/base/reviewers in one prompt, creates the PR on GitHub or Bitbucket via the shared helper, then asks one yes/no question to hand off to `resolve-pr-reviews`. It does NOT review and does NOT build. It has no scripts of its own — it calls `resolve-pr-reviews`'s helper through the sibling path `../resolve-pr-reviews/scripts/pr-reviews.js`, which is why the two skills are installed/removed together.

Design doc (historical): `docs/superpowers/specs/2026-09-05-open-pr-and-resolve-pr-reviews-design.md`.

## Files to create (every path + purpose)

| Path | Purpose |
|---|---|
| `skills/open-pr/SKILL.md` | The full procedure (frontmatter + body below). |
| `skills/open-pr/pr-description-default.md` | Fallback PR template used when the repo has no `.github/pull_request_template.md` (verbatim below). |

Dependency (not created here): `skills/resolve-pr-reviews/scripts/pr-reviews.js` — see `resolve-pr-reviews.md` for the `host`, `default-reviewers`, and `create` subcommands. Self-containment rule: no `.md` in the skill may contain `ARGUS_ROOT`, `/../..`, `<argus-repo>`, `ARGUS_CONFIG`, `.argus.config` (note the sibling path uses `/../resolve-pr-reviews`, a single `..`, which is allowed).

## Frontmatter (verbatim)

```yaml
---
name: open-pr
description: Use when a reviewed feature branch is ready to become a pull request — "open a PR", "create the PR", "open all the PRs", "/open-pr". Writes pr-description.md from the repo's template, confirms title/base/reviewers, creates the PR on GitHub or Bitbucket, then offers to start the resolve-pr-reviews loop. Does NOT review or build.
---
```

## Purpose; fires when / does not

- **Fires when** a reviewed feature branch is ready to become a PR: "open a PR", "create the PR", "open all the PRs", `/open-pr`. Also the target of the deprecated `/gh-pr`.
- **Does not** review or build (those happen upstream — task reviewer, branch reviewer, adversary-review — and downstream — Claude auto-review, CodeRabbit, SonarQube). Does not call `gh pr create` or the Bitbucket API directly. Does not auto-fire the resolver.
- Announce at start: "I'm using the open-pr skill to create the pull request."
- **State once, verbatim:** "This skill assumes the branch was reviewed upstream; it does not review." If the user says it was not reviewed, stop and point at `superpowers:requesting-code-review`.

## Procedure (numbered, faithful; every gate, cap, stop condition)

Helper:
```bash
SKILL_DIR="$(dirname "$(realpath <path-to-this-SKILL.md>)")"
PR="$SKILL_DIR/../resolve-pr-reviews/scripts/pr-reviews.js"
```
"open-pr and resolve-pr-reviews are installed together; the helper lives in resolve-pr-reviews." Every subcommand prints one JSON object.

### One repo
1. **Preconditions.** `git status --porcelain` must be empty; if not, offer to commit and stop until it is. If the branch has no upstream: `git push -u origin <branch>`.
2. **Gather.**
   - `node "$PR" host --repo "$(pwd)"` → `host, owner, repo, branch, head, defaultBranch`. Credential-free; always succeeds.
   - Base = `defaultBranch` unless the user says otherwise; if `defaultBranch` is `null` (no token, or offline), ask the user for the base.
   - Ticket id = first `TCNSR-\d+` in the branch name, else none.
   - Commits and diff: `git log --oneline <base>..HEAD` and `git diff <base>...HEAD`.
   - Template: `.github/pull_request_template.md` if it exists, else `pr-description-default.md` from this skill's directory.
   - Bitbucket only: `node "$PR" default-reviewers --repo "$(pwd)"`.
3. **Write `pr-description.md`** in the repo root (overwrite). Fill the template's sections. Rationale says *why*. Summary groups by logical change, never by file. Review notes name the risks. Ticket link `https://leanlawlabs.atlassian.net/browse/<id>` when an id exists.
4. **Confirm** in ONE prompt. Title = `<TICKET>: <readable branch>` (hyphens → spaces, first letter capitalised); the ticket id is mandatory when present. Show title, `<branch> -> <base>`, and reviewers (Bitbucket). Options: create / edit title / stop. Do not create until confirmed.
5. **Create.**
   ```bash
   node "$PR" create --repo "$(pwd)" --title "<title>" --body-file pr-description.md --base <base> [--reviewers "<name>,<name>"]
   ```
   Report exactly: "PR created: <url>".
6. **Hand off.** Ask one yes/no question with exactly these two options:
   - "Yes, run automatic review resolver loop"
   - "No, I'll handle reviews myself"

   On yes: invoke the `resolve-pr-reviews` skill with the PR url(s), then report "resolver loop running on <owner>/<repo> #<number>". On no: stop.

### Several repos ("open all the PRs")
- Detect every checkout touched this session whose branch is ahead of its base. Run steps 1–3 for each.
- ONE confirmation table: `| repo | title | branch -> base | reviewers |`.
- Create all, report every url, then ONE hand-off question covering all created PRs.
- A repo that fails preconditions is listed with the reason — never silently skipped.

## Hard rules (critical ones VERBATIM)

From `## Common mistakes`, verbatim:
```
- **Reviewing here.** Don't. Say the assumption sentence and move on.
- **Listing files in Summary.** Group by logical change.
- **Dropping the ticket id from the title.** Mandatory when the branch has one.
- **Creating before confirmation.** Step 5 waits for step 4.
- **Auto-firing the resolver.** Always the yes/no question first.
- **Calling `gh pr create` directly.** Argus is on Bitbucket; the helper knows.
```
Also verbatim: "Every subcommand prints one JSON object. Never call `gh pr create` or the Bitbucket API directly."

## Scripts

None bundled. Helper behavior relied upon (implemented in `resolve-pr-reviews/scripts/pr-reviews.js`; full spec in `resolve-pr-reviews.md`):
- `host --repo <p>` → `{ host: 'github'|'bitbucket', owner, repo, branch, head, defaultBranch: string|null }`; never throws for missing credentials (best-effort default branch: GitHub `gh repo view --json defaultBranchRef`, Bitbucket `GET /2.0/repositories/{o}/{r}` → `mainbranch.name`).
- `default-reviewers --repo <p>` → `{ host, reviewers: [{name, uuid}] }` (Bitbucket: reviewers on the most recent MERGED PR's detail; GitHub: `[]`).
- `create --repo <p> --title <t> --body-file <f> [--base <b>] [--reviewers a,b] [--draft]` → `{ host, url, number, head }`. Body file resolved against the repo root. Missing base + failed lookup → exit 2 (`pass --base <ref>`). Bitbucket reviewer names resolve case-insensitively to uuids (`{uuid}` passes through; unknown name errors listing the valid names); if `--reviewers` is omitted, all default reviewers are added; Bitbucket PRs are created with `close_source_branch: true`. GitHub: `gh pr create -R o/r --head <branch> --base <b> --body-file <abs> [--reviewer a,b] [--draft]` run in the repo cwd.
- Requires `BITBUCKET_TOKEN` (or `WCG_ATLASSIAN_BITBUCKET_TOKEN`) for Bitbucket create/default-reviewers; `gh` authenticated for GitHub.

## Tests to write (list the behaviors the existing tests pin)

No tests target the skill text itself beyond `tests/scripts/skills-self-contained.test.ts` (no forbidden root-reaching strings in its `.md` files). Its helper behaviors are pinned in `tests/scripts/pr-reviews-cli.test.ts` (`host`/`create`/`default-reviewers` cases: GitHub/Bitbucket default branch, `defaultBranch: null` on missing token or gh failure, relative body-file resolved against the repo for both hosts, default-branch fallback when `--base` omitted, UsageError when it cannot be determined, Bitbucket reviewer name→uuid, unknown reviewer rejected, GitHub create returns head from `prView`) and `tests/scripts/pr-github.test.ts` (`createPr` passes `--head`, runs in repo cwd, forwards `--reviewer a,b`).

## Output format (templates verbatim)

**`pr-description-default.md`** (verbatim):
```markdown
<!-- skills/open-pr/pr-description-default.md — used when the repo has no .github/pull_request_template.md -->
## Rationale

<why this change exists — the problem, not the diff>

## Summary

- <logical change 1>
- <logical change 2>

## Review notes

- <risk or area needing careful eyes>

## Ticket

<https://leanlawlabs.atlassian.net/browse/TCNSR-XXXXX, or "none">
```

**Title:** `<TICKET>: <readable branch>` (hyphens to spaces, first letter capitalised; ticket mandatory when present). The source does not say whether the ticket id is stripped from the readable-branch part; examples in this spec that strip it are illustrative.

**Confirmation (one repo):** title, `<branch> -> <base>`, reviewers (Bitbucket); options create / edit title / stop.

**Confirmation (several repos):**
```
| repo | title | branch -> base | reviewers |
```

**Reports:** `PR created: <url>` · hand-off options `Yes, run automatic review resolver loop` / `No, I'll handle reviews myself` · on yes `resolver loop running on <owner>/<repo> #<number>`.

**Quick reference table** (keep in SKILL.md):

| Step | Command / action | Blocked by |
|---|---|---|
| 1 Preconditions | `git status --porcelain`, `git push -u` | dirty tree |
| 2 Gather | `node "$PR" host`, `node "$PR" default-reviewers`, git log/diff, template | — |
| 3 Write | `pr-description.md` | — |
| 4 Confirm | title / base / reviewers | user |
| 5 Create | `node "$PR" create …` | step 4 |
| 6 Hand off | yes/no → resolve-pr-reviews | user |

## Acceptance check (2-3 scenarios)

1. **Bitbucket single repo.** Clean branch `TCNSR-512-add-export` with no upstream: agent pushes `-u`, runs `host` (defaultBranch `main`), `default-reviewers` (two names), writes `pr-description.md` from the default template with ticket link `https://leanlawlabs.atlassian.net/browse/TCNSR-512`, shows `TCNSR-512: Add export`, `TCNSR-512-add-export -> main`, reviewers; on "create" runs `create ... --reviewers "A,B"` and prints `PR created: https://bitbucket.org/<o>/<r>/pull-requests/<n>`; then asks the two-option hand-off question and does nothing further on "No".
2. **Gates.** Dirty tree → offers to commit and stops. `host` returns `defaultBranch: null` (no token) → asks the user for the base rather than guessing. User says the branch was never reviewed → stops and points at `superpowers:requesting-code-review`. No PR is created before the user picks "create".
3. **Several repos.** Two touched checkouts ahead of base and one dirty: one confirmation table listing the two eligible repos plus the dirty one with its reason; both PRs created and both urls reported; a single hand-off question; on yes, `resolve-pr-reviews` is invoked with both urls.
