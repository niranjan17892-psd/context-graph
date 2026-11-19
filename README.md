# Argus rebuild kit

This folder holds a spec plus a driving prompt. A fresh agent session uses them to recreate the Argus
repo from an empty git repo: the engine, the MCP server, the skills, the installer, and the docs.

## Use it

1. Create an empty git repo and copy this whole folder into it as `rebuild/`.
2. Open Claude Code (or Codex / Copilot CLI) in that repo and say:

   > Read `rebuild/00-master-prompt.md` and execute it.

3. The agent works through 11 milestones and stops after each one with passing tests as evidence. Expect a
   multi-session effort; the original is roughly 1,000 files. Resume with "continue from milestone N".

## Contents

| File | What it is |
|---|---|
| `00-master-prompt.md` | The prompt itself: stack, config, graph model, pipeline, MCP contract, invariants, known gaps to fix, and milestones |
| `appendix-engine.md` | Tables and specs checked against the code: the 38-tool param table, phase dependencies, cross-repo link kinds, the .NET analyzer, and fixtures |
| `installer.md` | setup / fix-links / uninstall, client adapters, linker, SessionStart hook, repo dotfiles |
| `skills/argus.md` | Orchestrator skill + 4 sub-agents + `project_conventions/` template |
| `skills/bootstrap-argus.md` | Project onboarding (+ the `scripts/project.js` CLI) |
| `skills/argus-defensive-review.md` | Authz-gap and taint review, including its full knowledge base |
| `skills/argus-retrospective.md` | Session retrospective |
| `skills/adversary-review.md` | Cross-model branch review loop + scripts |
| `skills/resolve-pr-reviews.md`, `open-pr.md`, `gh-pr.md` | PR lifecycle + scripts |
| `skills/product-skills.md`, `backlog-skills.md` | PM product and backlog skill families |
| `skills/eval-package.md`, `build-gold-dataset.md`, `reflect.md` | Eval and learning skills |
| `skills/jira-ticket-generation.md` | Optional: org-specific; values tied to one org are marked `[org]` |
| `skills/superpowers.md` | Vendor obra/superpowers v7.0.0 (MIT) unchanged, re-apply one patch, with a from-scratch fallback |
| `review-notes.md` | Record of where it came from: the independent code review that corrected the first draft of the prompt. Not needed for the rebuild. |

## Fidelity

- **Engine, MCP contract, and installer:** the specs were written from the source at commit `d56a1ac7` and
  checked against the real code (exact tables, deps, params, caps, and command strings).
- **Skills:** frontmatter, hard rules, gates, and output templates are copied word for word; explanatory
  prose is condensed. A rebuilt skill should behave the same but will not match the original text.
- **Deliberate improvements:** `00-master-prompt.md` §11 lists defects in the original. By default the rebuild
  fixes them: silent edge drops, dead cross-repo kinds, under-declared phase deps, dead code, and
  inconsistencies inside skills. Tell the agent to reproduce an item as-is if you need byte-level parity.
- **Not covered:** the git history and the point-in-time records under `docs/superpowers/{specs,plans,reviews}`.
  These are design history, not behavior.
