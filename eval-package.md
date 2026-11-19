# Skill: eval-package

## Files to create

All prose; the skill contains **no code**. The references are rules, not templates — each one repeatedly tells the builder to "read the most similar eval package that already exists" for shape.

| Path | Purpose / content spec |
|---|---|
| `skills/eval-package/SKILL.md` | Orchestrator: overview + 3 principles, the 7-step loop, the 8 universal rules, verification gate. |
| `skills/eval-package/references/package-contract.md` | **The specification** ("Everything else in this skill is guidance; this part is the requirement"). Sections: What is in the folder · How it runs · manifest.json · The configuration that lives with the environment (How passing is decided; The judge) · The cases (Optional fields; Accepted differences; Building the case list) · The result · What can be compared across different evals · How the verdict is decided · Where results go · Credentials · When things fail. |
| `skills/eval-package/references/storage-backends.md` | What shared storage must guarantee and how it shapes the build. Sections: Not a question for the user · Guarantees · The rule that follows (no per-eval storage code) · No environment named inside a package (lookup, package-vs-environment split, independence, two rules) · Guard the three things the split makes possible · Versioning the data · One description in both places · Moving an existing package · Declaring where everything is (`source.json`) · On AWS · On Azure · On local disk · Sending results elsewhere. |
| `skills/eval-package/references/anti-gaming.md` | Six observed failure modes (bug / check / fix each) + a 7-item checklist. Run while writing the evaluator, before Evaluator Test. |
| `skills/eval-package/references/evaluator-test.md` | The three-leg scorer proof (clean / corruption / degenerate), output record JSON, re-run triggers, what a failure means. |
| `skills/eval-package/references/evaluator-patterns.md` | Prose guidance for building the evaluator (scores, critical vs diagnostic, PENDING/SKIPPED, verdict order, pinned exceptions, code/measurement/LLM/rubric/reference-free/human scoring, scoring the right thing). |
| `skills/eval-package/references/runner-patterns.md` | Prose guidance for the runner (one result per case-target, fake-failure traps, never lose a paid result, cost disclosure, config/credentials, targets, no-dataset mode, language). |
| `skills/eval-package/references/smoke-test.md` | Ordered 7-step real-run procedure, "after changing anything shared" rule, diagnosis table, failing-on-merits handling, what to hand over. |

## Frontmatter (verbatim)

```yaml
---
name: eval-package
description: Use when someone wants to evaluate a prompt, agent, or pipeline - takes their inputs and expected answers, builds a standalone eval package (runner, evaluator, results), and runs it until it works.
---
```

Body title `# Eval Package`.

## Purpose; fires when / does not

Builds one **eval package**: a self-contained folder holding the prompt, a runner, an evaluator, the test that proves the evaluator, and declarations of where data and results live. Data and results are **not** in the folder; the package reads them.

Three principles (state them in the Overview):
1. **The package runs standalone.** One command runs a case, one scores the result, one proves the scorer. No harness, framework, or other repo. "If it only works because some outer script sets it up, it is not a package."
2. **You are not done until you have run it.** Deliverable is a folder you executed, iterated on, and watched produce a valid run result.
3. **Materials first, then infer — don't interrogate.** Take the user's documents/questions/expected answers first; infer design; for anything not inferable, propose a default and confirm. Never make the user pick from a taxonomy.

Fires: evaluating a prompt, agent or pipeline against inputs/expected answers. Does not: build verified/cited ground truth from a corpus (that is `build-gold-dataset`, run before this — mention it as an option); repetition/averaging/iteration summaries (that is a harness layer above; this package is what it calls).

## Procedure

Loop: `Collect → Propose → Scaffold → Evaluator Test → Smoke test → Iterate → Hand off`, with Iterate looping back to Scaffold. Hard ordering: never skip to Scaffold; never stop before Smoke test passes; **never run the real eval before Evaluator Test passes**.

### Step 1 — Collect the materials
1. Open by asking for inputs, not architecture, e.g. *"Send me what you've got — the document(s) or data to run against, the questions or task, and the answers you expect if you have them. If there's already a prompt you use, that too."* Then actually read them.
2. Want: input data (real, not described; expect several, and cases needing several docs at once); the full question/task list up front; expected answers if any ("I don't have them yet" is a complete answer and tells you the mode; a grid/spreadsheet is fine; if they want cited verified ground truth, point at `build-gold-dataset`); target(s); existing prompt; **traps** — ask directly whether anything is designed to catch a mistake (superseding revisions, near-duplicates, decoy docs, "the document doesn't say" questions). Mine natural traps before constructing synthetic ones.
3. Ask for missing items one or two at a time, in the user's vocabulary. Never say "system under test", "runner shape", "invocation mode" to the user.
4. Decide mechanical details yourself (PDF base64 vs extracted text, case IDs, spreadsheet parsing) and state what you did; ask only when it changes the build and is undeterminable.
5. **Gold data and results go to shared storage — don't ask.** No "keep it in the repo for now" option. Tell, once the eval name is known: *"Gold data and results go to the project's shared storage, under `<data location>/<eval-name>/1.0.0/` and `<results location>/<eval-name>/`. I'll read both location names from wherever the project keeps them. Send me the files and I'll upload them."* Read `storage-backends.md` before scaffolding. Only three storage asks: (1) which store and where it keeps location names (environment must be a variable in the name); (2) which environments, by name (to know how many independent data/credential/threshold sets exist — never put them in the package); (3) the files themselves, uploaded under a fresh gold version in their environment.
6. **The package names no environment.** One `--env` argument, no default, resolves location names → that environment's config, gold set, results. Each environment self-contained; none canonical. Never one gold set plus a per-environment id remap.
7. **Never inspect data to classify it, never raise its content** (no client-data/PHI assessment, no redaction proposals, no synthetic substitutes). The project's storage choice already decided that.
8. **Ask now for anything needed to run** (credentials, endpoint, role, files). Work out required variables and **check they resolve before building anything**. If missing, say so immediately and offer: (a) point the package at an existing credentials file path (usually best), (b) user puts it in the package's own credentials file, (c) user sets it in the shell. Never ask the user to paste a secret; never read a credentials file to extract a value; reference the path (`ENV_FILE=<path>` in package `.env`). Never hunt for renamed/disabled variants (`*_DISABLED`, `*_OLD`, other prefixes) — ask. If unavailable now, let the user choose up front whether to build now and verify later.

### Step 2 — Propose the shape, then confirm
1. Put a concrete 2–3 sentence proposal (what it calls, how scored, what passes) and ask for corrections. Proposal beats interrogation.
2. **Default invocation `direct_api`** (their prompt + input to a model) unless they named something they built; use `agent_invocation` / `api_with_credentials` when they point at a deployed thing (get ARN/endpoint/region then).
3. Infer `mode` from purpose; state it with a one-line reason; their disagreement wins:

| Their answer | `mode` |
|---|---|
| "Catch it if this breaks — we have vetted correct outputs" | `regression` |
| "I'm building this now; I'll write expected answers and refine them" | `dev-cycle` |
| "Compare models/configs, or track capability as models improve" | `capability-benchmark` |
| "Watch real production traffic; no expected answer per input" | `live-monitoring` |

   A gold standard does not separate modes; benchmarks may have vetted answers or rubrics. Only `live-monitoring` is inherently reference-free.
4. State as decisions (question only if not inferable, with a default): **Scoring** (exact values → code check; prose/judgement → LLM judge; several scores allowed; probe "what would you look at first?"). **Critical checks** — ask plainly *"which of these, failing on its own, means the whole case failed — regardless of how everything else scored?"* → `critical`; rest `diagnostic`. **Passing** — aggregate threshold 0–1 over diagnostics, or none (benchmarks/monitoring often none; never invent a number); criticals are all-or-nothing. **Judge** — must be a different model family; if none available, judged scores are `provisional` self-tests. **Access** — exact env var names. **Location/name** — derived from task; version starts `1.0.0`. **Storage/environments** — not a choice; state resolution with env as variable, package vs environment config split, environments list; thresholds, gold version and evaluator-test record are per environment.
5. `live-monitoring`: input is a datetime range (`start`/`end`) plus log source; must work over any historical range, not just "last N hours".
6. Never scaffold against imaginary input — lacking real data/path is the one thing worth blocking on.
7. **State the matrix and its cost** before building, e.g. *"3 protocols × 8 questions = 24 cases, run against 2 targets = 48 invocations. With an LLM judge scoring each, ~96 model calls."* A case = one invocation = one run result. Don't assume a full cross-product (omit inapplicable pairs, say so). Multiple targets = config sweep; confirm before doubling the bill. Get agreement on the number before spending it.

### Step 3 — Scaffold
Build exactly `package-contract.md`. Use `runner-patterns.md` / `evaluator-patterns.md` as examples to adapt or discard. Default Python (`runner.py`/`evaluator.py`), other language if the target requires it. Then run the **anti-gaming pass** (`anti-gaming.md`) over the evaluator.

### Step 4 — Evaluator Test
Follow `evaluator-test.md`. Run `python evaluator_test.py`, record in `manifest.json` `evaluator_test`, **do not proceed until it passes.** A failure is a finding about the evaluator, never a reason to weaken a check; collateral failures mean a check is over-broad.

### Step 5 — Smoke test
Follow `smoke-test.md`: execute one real case for real (no dry run/mock); confirm key resolves, runner reaches target, evaluator scores, JSON validates against the contract. Then the remote assertions from `storage-backends.md`: store preflight, no fallback to a local copy, conditional create refuses a second write, pointer refuses to move backwards. Tell the user before a billable call and roughly what it costs.

### Step 6 — Iterate
Diagnose the broken layer before changing anything: **Credentials** (not found, 401/403 — fix resolution order, never hardcode), **Runner** (reach, request, response shape — likely the content-block fix), **Evaluator** (scoring threw / out-of-contract score; reported distinctly), **Contract** (missing fields — check field by field), **Evaluator Test** (doesn't discriminate — never weaken), **The eval itself is wrong** (both work, case fails — a result, not a bug). A green smoke test means it executes and emits a valid result, not that the case passed.

Case fails on the merits → put three outcomes to the user explicitly; never tune the evaluator green:

| Outcome | When | What happens |
|---|---|---|
| **The system is wrong** | Output genuinely incorrect | Leave failing; the eval did its job. |
| **The expected answer is wrong** | Answer key in error | Publish new gold version, point `source.json` at it, bump package `version`, **re-run Evaluator Test**, update overview README in both places. |
| **Documented divergence** | Both defensible | Record in `input/adjudications.json` **with the exact (expected, actual) pair pinned**. |

### Step 7 — Hand off
Show: tree created; **Evaluator Test result first** (plants caught, degenerate baseline scored badly); **the actual run-result JSON, pasted**; command to re-run the package and to re-run Evaluator Test (must re-run whenever evaluator or expected answers change); **the short description written to both locations** (read existing first — someone else wrote half; not done until both exist, identical, within length; paste both locations and say you checked); where run history lives in the store's own terms (each run its own folder named by start time, which object is the pointer, summarising reads without writing — users assuming overwrite will hand-copy files); anything `PENDING`/`provisional` and why; what's deliberately absent (repetitions, retries, iteration summaries, cross-run aggregation). `dev-cycle` → mention promotion to `regression` by copy-and-relabel. `live-monitoring` → usually scheduled; failures commonly promoted into `regression`/`dev-cycle` cases.

## Hard rules

Eight universal rules (SKILL.md Step 3, keep wording):
1. **One (case, target) pair = exactly one run result.** No `repetition`, no `run_id`, no iteration summary, no averaging. "Run it 10 times and average" → tell them that's the harness layer.
2. **Never overwrite a previous run.** "Results go to `<results_root>results/<version>/<run_stamp>/<case_id>.json`, where `<run_stamp>` is the UTC start time of the invocation, **computed once at process start and shared by every case in that run**." A fixed computed path is forbidden. In shared storage this is a property of the write: create fails if something exists; `latest` pointer never moves backwards. Do not write storage code inside the package — use what the project owns.
3. **The package stamps its own identity**: `package_name`, `package_version`, `mode` read from `manifest.json` at runtime onto every result; never hardcoded. Also stamps the environment (from `--env`) on every result and gold reference.
4. **Credentials:** package `.env` (incl. any `ENV_FILE` it points at) → ambient environment → clear error. Standard variable name only; never hunt renamed/disabled variants — ask.
5. **Never silently drop a case.** Runner failure `status: "error"`; evaluator failure tagged separately.
6. **Extract model output defensively** — loop content blocks and select by type, never `content[0]`.
7. **Never fabricate a score, never quietly omit one.** Unscorable → `PENDING` with output + criteria attached; not applicable → `SKIPPED`; both `score: null`, neither a pass; anything pending → `INCOMPLETE`, not `PASS`.
8. **A critical failure fails the case outright.** No aggregate rescues it.

Other non-negotiables (verbatim where marked):
- "**Never ask the user to paste a key, token, or secret into the conversation, and never read a credentials file to extract a value.**"
- "**Never inspect the data to classify it, and never raise its content.**"
- "**The package names no environment.**" `--env` has no default; a command run without it fails and says so.
- Judge must be a **different model family**; if the answering family is unknown, assume it is the same one (scores become provisional).
- Never make Evaluator Test pass by removing a plant; never relax a comparison until it goes green.
- **Verification gate:** do not report complete until Evaluator Test passed **and** the package was executed and you can paste the run result. If the change touched anything shared, run **every entry point, cheapest first**. "**If something is blocking execution, ask for it — don't report it.**" Hand over unverified only after asking and being unable to get it; then label it unverified, say what's missing, give the exact command.

## Reference content spec

### package-contract.md
- **Folder:** `manifest.json`, `run_config.example.json` (never read; realistic values, not blanks), `.env` (never committed), `.env.example` (names, no values), `.gitignore` (`.env`, `runs/`, `results/`), `input/source.json`, optional `input/adjudications.json`, `prompt.md` (only if a prompt exists — never an empty one), `runner.py`, `evaluator.py` (may be same file), `evaluator_test.py`, `requirements.txt`, `results/` (not committed; a note saying where runs are), `runs/` (not committed scratch), `README.md` (what it evaluates, what it needs, the one command, what output means). No data, no results, no real config in the folder.
- **How it runs:** two commands (run case → result; score result); runner+evaluator in one file → first command does both. Third command proves the evaluator without touching the target. Every command takes the environment, no default. Loop-all-cases flag OK; still one independent result per case, no summary file.
- **manifest.json** shape: `name`, `version`, `mode`, `description`, `created_at`, `evaluator_test: { <env>: {passed_at, package_version, gold_version, clean_score, plants_caught} }`. Version bumps when runner, evaluator or output shape change results. Thresholds are **not** here.
- **Environment config** (lives next to that environment's answers): endpoint, credential variable names, gold version, thresholds, build identification. `"pass_rule": {"critical": "all", "aggregate_threshold": 0.8}` — criticals must all pass; threshold applies only to remaining checks as a pass rate. Either part may be omitted (omit threshold for no-pass modes; omitting critical = nothing critical — push back). No empty values; an unset threshold is a stated decision. Judge model+provider named; different family or scores are self-tests marked so.
- **Cases** (in shared storage): file per case or list `[{"case_id","input","expected"}]`; `expected` omitted for live-monitoring and rubric benchmarks. Optional `criteria` keyed by check name `{critical: bool, zero_tolerance: [...]}` — every zero-tolerance rule needs a planted violation in the evaluator test; no criteria → all non-critical, say so aloud. `tags` free-form, reserved `trap`. `trap_origin` natural|constructed — find natural first; **constructed traps excluded from the pass rate and reported separately; natural traps stay in the rate; a trap failure is always reported even when the run passes.** Multi-part input: named pieces, `{"file": ...}` values loaded from the data location, others literal; template uses same names. Case id built from parts, filename-safe. Write the full list out (a small generator script is fine past a few dozen if the generated list is what's stored). No forced grid. Say the arithmetic before running.
- **Accepted differences:** each disagreement resolves to exactly one of system wrong / expected wrong (new data version) / both defensible (accepted). Accepted = pinned to the exact value: produce it → check recorded not-applicable with reason; anything else → fail. Don't write these into a gold-dataset corrections log (different, closed record).
- **Result** — all fields required in every result:
```json
{"schema_version":"1.1","package_name":"…","package_version":"1.2.0","mode":"dev-cycle",
 "case_id":"visits-count","environment":"dev","run_at":"2026-07-28T14:00:03Z",
 "run_group":"20260728T140000Z","response":"3","status":"ok","verdict":"PASS",
 "scores":[{"name":"count_match","result":"PASS","score":1.0,"actual":3,"label":"correct",
   "explanation":null,"kind":"code","direction":"maximize","severity":"critical"},
  {"name":"reasoning_quality","result":"PENDING","score":null,"actual":null,"label":null,
   "explanation":"no judge configured — not scored","kind":"llm","direction":"maximize","severity":"diagnostic"}],
 "latency_ms":1530,"cost":{"input_tokens":812,"output_tokens":4,"total_tokens":816}}
```
  `schema_version` = result-shape version (moves independently). `environment` checked by scoring, which refuses foreign results. `status` ∈ ok/error/timeout/incomplete. `verdict` ∈ PASS/FAIL/INCOMPLETE. Score entry fields: name, result (PASS/FAIL/PENDING/SKIPPED — the portable signal), score (local scale; null for PENDING/SKIPPED), actual (what pins attach to), label, explanation (judge always; mechanical on failure), kind (prose says human/model/measurement/mechanical; example uses `code`/`llm` — **inconsistency in source**), direction, severity. Conditional: provisional marker; for unscorable checks, what a later scorer needs. Cost: full breakdown; unknown → empty, never zero ("Zero is a claim; empty is the truth"). `run_group` shared by every result of one command. No run number/repetition count. Two separate provenance blocks never merged: **capture** (gold location, data version, exact-copy identifier; immutable) and **scoring** (environment, config + version id, thresholds, data version named by the scoring config). Add when applicable: error message naming the layer, target id, live event reference, case tags.
- **Cross-eval comparability:** travel = schema version, status, verdict, each check's result, kind/severity/direction, cost/time, identity fields. Don't travel = score values (local scale) and check names. "**Compare on the result, read the measurement only within one package.**"
- **Verdict order:** (1) any critical FAIL → FAIL. (2) any critical unresolved (PENDING or SKIPPED) → INCOMPLETE, except SKIPPED by a pinned accepted difference. (3) any non-critical PENDING → INCOMPLETE (non-critical SKIPPED does not). (4) pass rate over non-critical checks that resolved PASS/FAIL (PENDING/SKIPPED out of numerator and denominator); none resolved → INCOMPLETE; below threshold → FAIL. (5) else PASS. SKIPPED is never a pass; mostly-skipped runs said prominently; constructed traps out of rate.
- **Results layout:** `<results location>/<environment>/results/<package version>/<run stamp>/<case id>.json`, `…/<run stamp>/evaluator-test.json`, `…/<package version>/LATEST`. Stamp filename-safe and sortable; one stamp per command; never write into an existing run's folder; pointer moves once, right after the first result of a run lands. Not repetition tracking. (SKILL.md's shorter form `<results_root>results/<version>/<run_stamp>/` assumes the env is already in `results_root` via the env-specific bucket.)
- **Credentials:** env config names variables, holds no secrets. Package credentials file never committed; prefer a pointer to an existing file. Precedence: package settings > pointed file > terminal. Storage credentials come from the platform sign-in; bucket/container names are not secrets and are printed at run start. Missing variable error names it and every place checked.
- **Failures:** runner throws → error tagged runner, keep case; evaluator throws → error tagged evaluator, keep runner capture; timeout → record limit; no retries (if user wants them, record attempt count); select response parts by kind.

### storage-backends.md
- Never ask whether data can live in the repo; only ask which store and where bucket/container names are kept.
- **Guarantees:** finished result never replaced (conditional write, not unique-filename computation); scoring rewrite is the sole exception and must be conditional on no intervening change; pointer never moves backwards (check built into the write); gold versioned, version never edited; every result records the data version **and** the store's exact-copy identifier; missing file is an error, never a fallback to local copy or empty case list.
- **Rule:** packages never write their own storage code; one small project-owned set of operations.
- **No environment named inside a package**; exact-name lookups one at a time, never prefix fetch. Package holds runner/evaluator/test/env-invariant settings (timeouts, concurrency); environment holds endpoint, credential var names, gold version, thresholds, build id; document/user identifiers live in that env's data. Split by kind, no precedence overlap. Environments independent (no canonical set + id translation table); thresholds, scorer proof, results all per environment. Two rules: never put an environment name/location inside a data set (relative paths only); env-specific behaviour in config, not `if env == "dev"` branches (record "not stated").
- **Three guards:** refuse to score results from another environment; record what scored a result (data version taken from the scoring config); validate config at load down to the actual threshold values (a thresholds section with all-empty entries must fail; check exact values where one value is meant; require different things per entry point — scoring needs thresholds + data version, not endpoint/credentials).
- **Data versioning:** `<data store>/<eval name>/<version>/cases.json` + `README.md`. Patch = typo/reworded note; minor = added cases or materially reworded question; major = dropped/replaced cases, shape change, new source material. Revise by uploading the whole set under the next number. **No pointer to newest data version.** Description lives beside the data. Upload with explicit filenames (recursive uploads follow links; on delete-denied stores duplicates are permanent).
- **One description in both places** (data root and results root): what it tests and what passing means; what the data contains and versioning; how results appear and what a failure looks like; a few lines of current state (versions, case count, latest run). 3–4 paragraphs; no links/"see the repo"; current state, never history; update both together, read before writing, both identical.
- **Migrating an existing package:** bump version, re-prove scorer, move data (fresh version) and committed results out in one go, delete from repo, untrack results, leave a note. Half-moved is worse than either end.
- **`input/source.json`:**
```json
{"eval":"chat-session-title",
 "resolve":{"EVAL_DATA_BUCKET":{"ssm":"/<service>/${ENV}/EVAL_DATA_BUCKET"},
            "EVAL_RESULTS_BUCKET":{"ssm":"/<service>/${ENV}/EVAL_RESULTS_BUCKET"}},
 "config_file":"s3://${EVAL_DATA_BUCKET}/chat-session-title/run_config.json",
 "results_root":"s3://${EVAL_RESULTS_BUCKET}/chat-session-title/"}
```
  No env name, endpoint, identifiers, thresholds, or data version (naming a data version pins the package to one environment).
- **AWS:** S3 + Parameter Store (exact names, no path-prefix query). Create = put with `If-None-Match: *`; failure means key taken → raise naming it, never retry as plain put. Scoring rewrite = put with `If-Match: <etag>` (S3 ETag, not object tags). List runs by prefix + delimiter (pointer is an object, naturally excluded). Credentials via standard chain; print caller identity. Deny-delete on results bucket recommended. **Azure:** Blob + Key Vault, same conditional headers/ETag. **Local disk:** working files only, never data or results ("keep it local for now" is not an option). **Reporting elsewhere** (e.g. test management): separate command reading one run and pushing the outcome; never in the run path.

### anti-gaming.md
Six modes (each: bug, check, fix): (1) aggregate computed but never enforced → enforce with threshold or label `diagnostic`; (2) degenerate-answer flooding → compute best no-work score; cap abstention (more than N answerable cases "not stated" fails the run) — note the asymmetry (punishing invention is the easy half); (3) wildcard exclusions → pin to exact (expected, actual); no ID/pin is forbidden; (4) unscoped membership tests → assert at narrowest scope (record/section/field); (5) zero-tolerance rule that doesn't hard-fail, or skipped when field missing → plant a violation; absence is a failure; (6) tests inheriting author blind spots → walk "what does this evaluator NOT assert?" (unread output parts; numbers vs text verified separately; unexercised structural variation — one vs many, empty, nested; coincidental passes) → each gap becomes a check or documented limitation. Checklist (all must be yes) = the 7 bullets in SKILL.md: aggregates enforced or diagnostic; degenerate best score computed and bad; exclusions have ID + pinned pair; containment scoped; every zero-tolerance rule planted; missing fields explicit; NOT-assert walk done.

### evaluator-test.md
- Run after scaffolding, before real runs; **prove separately per environment**.
- **Leg 1 clean:** expected answers fed back as output must score 100%; failure = check wrong or gold inconsistent (never loosen).
- **Leg 2 corruption:** plants on a copy; every plant caught with correct severity, zero collateral. Standard plants: wrong value; invented answer (where "not stated" is legitimate); fabricated citation. By kind: code check — perturb past tolerance and one within tolerance (must not flag); assertion judge — delete a required claim AND add an unsupported claim (both fail); rubric — known-bad must grade materially lower (ordering, not threshold). One plant per `zero_tolerance` rule.
- **Leg 3 degenerate:** empty, all "N/A"/"not stated", echoed input, constant → must score badly; if "not stated" is legitimate, add floor: more than one answerable case returning "not stated" fails the run.
- **Output** `evaluator-test.json` in its own stamped folder under the environment; **never moves the pointer**. Shape: `schema_version "1.0"`, package_name, package_version, run_at, `clean {cases, scored_perfect, pass}`, `corrupt [{plant, case_id, caught, severity_correct, collateral[]}]`, `degenerate [{baseline, aggregate_score, pass}]`, `verdict`. Command exits failing on failure. Record per-env in manifest. Re-run whenever evaluator or expected answers change; a record whose package_version lags the manifest, or a missing env record, = unproven — say so. Failure meanings: clean fails → fix check or fix gold and log; plant escapes → check confirms presence not correctness; collateral → over-broad; degenerate scores well → gameable, add floor. "Never make this test pass by removing a plant."

### evaluator-patterns.md
Evaluator never calls the target (enables cheap re-scoring). Read an existing evaluator first. Prove scorer first. Score dimensions separately. Every score records measurement + verdict + actual + words on failure. User decides critical. Unrunnable vs not-applicable distinction (as verdict rules); record what a later scorer needs. Verdict order as above ("Put the threshold first and a critical failure can pass on a good average"). Pinned exceptions. Code checks: missing/empty answer must fail. Measurements: record direction; threshold if gating. LLM judge five rules: different family (else self-test marked on each score); unknown family → assume same; fixed structured output; check inventions not just omissions; no judge configured = could not run. Judge ~doubles calls; keep judge cost separate from result `cost`. Rubrics: define each grade; per-criterion; state which grade passes or mark diagnostic. Reference-free: self-consistency, well-formedness, contradiction with input, refusal, policy; keep event reference. Human judgment: write result with no scores; never invent an automated proxy. Score against the same answer key the runner used; refuse foreign-environment results; never score a failed (error/timeout) run; scorer breakage recorded distinctly.

### runner-patterns.md
Read an existing runner first. One case × one target = one result file; no repetition support. Every case produces a result file incl. failures. Unknown numbers empty, not zero. **Fake failures to guard:** truncated reply (thinking counts against max tokens — set generously; stop at limit = failed run); unfilled template placeholders (check none remain, fail if any); answer read from wrong content part (select by type; if none, fail listing the kinds returned); partially collected stream; missing/unreadable document (fail loudly). Never lose a paid result: if write fails, keep output locally; if scoring fails, record and keep; the rescue must not itself fail (e.g. read-only dir). One case failing never stops others. State cost arithmetic before running. Endpoints/accounts in env config; config names secret variables only. Don't send randomness settings (temperature etc.) unless config explicitly asks — current models reject them. Multiple targets: each result records its target; filenames must not collide. No-dataset mode: pull records from any time window; keep original event reference. Language: whatever reaches the target; Python usual.

### smoke-test.md
Prerequisite: Evaluator Test passed. Before running: tell the user it's a real call, rough cost, which case, judge doubling, whether it writes to shared storage; don't ask for a single flagged cheap call; do ask if expensive, production, or writing somewhere that matters. Ordered steps, stop at first failure: (1) package complete, data files parse (config not included); (2) credentials resolve and the environment exists (reading env config proves both) — ask now if missing; (3) cases load from the shared location at the configured version, count as expected; (4) one real run (one case, one env; for multi-doc cases verify the assembled prompt contains all docs); (5) one real scoring pass, then deliberately score naming a different environment — must refuse; (6) field-by-field contract check (identity from manifest; permitted status/verdict; each score has measurement, verdict, actual, explanation; unrunnable checks have no number; critical fail → case FAIL; cost present even if empty; no run-number/repetition fields); (7) run a second time: two separate runs, first unchanged, pointer on newer; compare answers (non-determinism expected; large divergence → tell user, matters for repeat counts upstream); confirm one run's cases share one group. **After changing anything shared:** run every entry point cheapest first (evaluator test → summarise existing run → re-score one stored result → run a single case); mechanically compare arguments read vs declared; compare fields written by one command vs read by another. Diagnosis table: missing credential → ask, offer the three options (never paste/hunt/hardcode); permission denied → ask which account/role, don't try other keys; connection/timeout → network/address/region/invoke permission; response-parse error → select by kind; no text → tool call/refusal, record part kinds; scoring throws → ensure labelled as scoring failure; unparseable judge → fixed structure; missing fields → contract; works but scores badly → not a failure. Failing on merits: never adjust scoring; user picks system wrong / expected wrong (new version) / check wording wrong (new version). Hand over the actual produced result, or state exactly what blocked it and mark never-run.

## Sub-agents / scripts
None. Generated packages contain `runner.py`, `evaluator.py`, `evaluator_test.py` (per contract), but the skill itself ships no code.

## Output format
The package folder per the contract; the run-result JSON, evaluator-test JSON and manifest shapes above; the dual-location overview README; the Step 7 hand-off message.

## Acceptance check
1. **User hands over 3 protocol PDFs, 8 questions, a spreadsheet of expected counts.** Agent does not ask about architecture; states it will upload to `<data location>/<eval>/1.0.0/`; asks only which store/name source, which environments, and checks `ANTHROPIC_API_KEY` resolves (offering the three options if not, never asking to paste). Proposes `direct_api`, `regression` or `dev-cycle` with a one-line reason, code-check scoring, asks which checks are critical, states "3 × 8 = 24 cases … ~N calls" and gets agreement. Builds the contract folder with no data and no environment name, `--env` with no default.
2. **Evaluator always returns 1.0.** Evaluator Test corruption leg shows plants not caught → verdict FAIL, command exits non-zero; agent fixes the check (never removes a plant) and does not run the real eval until it passes; manifest records `evaluator_test.<env>`.
3. **Smoke test re-run.** Second run lands in a new `<run stamp>` folder, first untouched, `LATEST` points at the newer; scoring with a different `--env` refuses; a case failing on merits is presented as the three-outcome choice, and the hand-off pastes the real result JSON and both identical README copies.

## Gaps / source inconsistencies to preserve or resolve
- `kind` values: prose lists human/model/measurement/mechanical; result example uses `code`/`llm`.
- Results path: SKILL.md `<results_root>results/<version>/<run_stamp>/` vs contract `<results location>/<environment>/results/<package version>/…` (env implied by bucket in `source.json`). SKILL says pointer named `latest`; contract names it `LATEST`.
- SKILL.md Step 4 contains the typo "A Evaluator Test failure".
