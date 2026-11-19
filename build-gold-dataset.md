# Skill: build-gold-dataset

## Files to create

| Path | Purpose / content spec |
|---|---|
| `skills/build-gold-dataset/SKILL.md` | Self-contained orchestrator: overview + six principles, the loop, Steps 1–8, "Where the dataset goes" (incl. overview README), Hand off, Verification gate. Steps 1, 4, 5, 6, 8 inline the content of the references (with a few additions noted below). |
| `references/census-and-pairing.md` | Sections: "Enumerate before you draft" (structural facts only; size long-tail; naming conventions never trusted as data; explicit out-of-scope exclusions with reasons), "Pairing" (two situations: existing reference answer → pair by identity, treat as something to verify, corrections logged; none exists → pairing = source-to-target-schema mapping), "The mandatory 'as of when' question" (load-bearing when corpus could hold several versions; if structurally unambiguous, note why once; date/version visible on the record). **Lacks** the SKILL's three identity-pairing failure modes. |
| `references/record-format.md` | The four field shapes with JSON, the governing rule, locator guidance, case-level `as_of`, "What NOT to do". |
| `references/verification-checklist.md` | Checkbox version of Step 5 with headings: Filename and metadata · Citations · Placeholders and shortcuts · Absence claims · System-of-record checks · Required fields and entities · After the checklist. |
| `references/corrections-log.md` | Log format (two entry shapes), category taxonomy, "What belongs in the rules doc instead". |
| `references/segmentation.md` | TEACH/PROVE/PROTECT definitions, "Don't default into this, in either direction", "Scale matters" (~10 cases), "If it does apply". **Lacks** the SKILL's seal-timing table, rulebook-corpus check and sizing doctrine. |

## Frontmatter (verbatim)

```yaml
---
name: build-gold-dataset
description: Use when someone wants to build a gold dataset — verified (input, correct-output) pairs for a task — before an eval package exists, or asks to build verified ground truth, cite/correct answers against a corpus, or produce expected-answer data for a future eval. Not for building the eval itself (runner, evaluator, scoring criteria) — that's eval-package's job, once this exists.
---
```

Body title `# Build Gold Dataset`.

## Purpose; fires when / does not

Builds verified `(input, correct-output)` pairs for a task, each with the evidence that makes it trustworthy. "Nothing else." It is **the layer before an eval**: no runner, evaluator, scoring criteria, critical/diagnostic labels, grading-strictness tiers, or grader-proving — if the conversation reaches for that vocabulary, stop; it is `eval-package`'s job once a gold dataset exists.

Six principles (Overview, keep each):
1. **A target schema is not an answer key.** It gives the shape of a correct answer, zero verified examples.
2. **Verify against the system of record, not the spec describing it.** Check spec claims (field semantics, codesets, "where to find it") against real code/DB/API when reachable.
3. **Never trust metadata adjacent to your source data** (filenames, tags, folder names, cover branding). Extract only from the primary source, even when metadata agrees.
4. **Be adversarial toward your own draft, not just the source.** Assume the first draft has a shortcut (placeholder, dropped citation, unchecked assumption) and find it every case.
5. **Mine real traps before inventing synthetic ones.** Tag any synthetic trap so it is never mistaken for real evidence.
6. **An absence claim needs the same evidence bar as a presence claim** — confirmed more than one way; one example proves nothing about a corpus.

## Procedure

Loop: `Census & Pair → Confirm the target → Draft one case, live → Self-check → Log corrections`, with "Distill rules" feeding back to drafting; repeat per case one at a time; then `Decide on segmentation, explicitly → Hand off`.

**Never batch-draft blind.** One case at a time, live, with the user in the conversation; never delegate to a fire-and-forget background task (loses moment-by-moment decisions; risks unreviewed, unauthorized work on real files).

1. **Census & Pair.**
   - Enumerate every case by name before reading any closely: count, format, rough size, naming pattern. Note long-tail outliers (affect effort/pilot choice). Naming/tagging conventions are for locating files only, never a data source. Exclude out-of-scope items explicitly with a stated reason.
   - Pairing, existing reference answers: pair by identity; treat as something to verify and clean (like eval-package's Evaluator Test treats expected answers); corrections logged, never silently applied, never enshrined. Explicitly check and note (with reason) three identity failure modes: two source items claiming the same identity; a source item with no answer; an answer with no source item (stale/orphaned).
   - Pairing, no reference answers (common case): pairing = mapping each source item to the target schema/task.
   - Mandatory "as of when": ask per case when the corpus could hold multiple versions/snapshots and record it; if structurally unambiguous, note why once and move on (no ritual). Either way, the record carries a top-level `as_of`.
2. **Confirm what "correct" means.** Task and schema should already be in hand from intake (materials first, infer, don't interrogate) — this step confirms/validates. (a) Separate extractable fields from out-of-scope ones (operational, system-assigned, no source-grounded answer) and say so explicitly. (b) Check the spec against a real system of record when one exists.
3. **Draft one pilot case fully, live, before batching.** Read only what's needed for the target fields. Flag every judgment call as hit; resolve each with the user before the next case. A pilot surfacing a new pattern → stop immediately.
4. **Record format** — four shapes (see Output format). A missing citation must never be silent.
5. **Self-check every draft before calling it final** (adversarial pass; assume at least one defect):
   - Filename/metadata: any value from filename, folder, tag or metadata? Remove and extract from content, even if it agrees.
   - Citations: every field has quote+locator (fact), locators+reason (summary), sentinel+flag+reason (redacted), or explicit null reason? Every quote verbatim (not paraphrased/remembered)? Any quote claimed for an inferred/constructed value? Say it isn't printed text.
   - Placeholders/shortcuts: any value referencing another field ("same as X")? Any best-guess default not checked against source or system of record?
   - Absence claims: corpus-wide "never appears" resting on one example? Check more cases, ideally structurally different sources.
   - System-of-record: any checkable spec claim not checked against a reachable implementation is an open risk, not a completed field.
   - Required fields/entities: required field silently blank? Use the redaction sentinel (keep the entity) or say explicitly the entity can't be constructed from this source. Top-level `as_of` present?
   - After the checklist (SKILL.md version): fix, then log per Step 6 — because Step 3 already showed the value live, the fix is always a loggable correction; "There is no separate 'before hand-off' exemption."
6. **Log corrections, categorized.** Every fix to a value already shown to the user as a draft gets an entry — never fix silently, even obviously-correct fixes. One running `corrections-log.md` per dataset. Categories (never conflate): **answer correction** (checked against real ground truth — source re-read, system of record, SME — and found wrong); **drafting defect** (draft cut a corner the source didn't have — placeholder, missing citation, unchecked assumption; doesn't re-decide a domain fact; a citation added to an unchanged value still counts and shows value unchanged, citation now present); **process rule** (new/revised rule going forward; logged even though no case changed; future entries reference it). One incident may justify several entries. **Frozen at hand-off**: later disagreements found by eval-package runs go to its `input/adjudications.json` (`expected_correction` outcome), never back into this log.
7. **Distill rules as you go.** One living, standing rules document, rewritten (not appended forever); each rule points back to the case or log entry that produced it, so lesson N informs case N+1. The log answers "why does this field have this value"; the rules doc answers "what do I need to know before drafting the next case" — keep them separate.
8. **Decide on segmentation, explicitly.**
   - TEACH (open, iterated on), PROVE (sealed exam, opened once, never annotated/revised after sealing; catches overfitting), PROTECT (starts empty, grows forever; only confirmed real failures found after the fact; never synthetic or unconfirmed).
   - **Seal timing is part of the claim** — record it on the dataset (with `as_of` per case or once at dataset level):

| Case | Meaning | What the seal binds |
|---|---|---|
| **Cold seal** (strongest) | Split decided from file metadata alone (names, sizes, formats, page counts) before any document opened | Everything drafted after sealing |
| **Warm seal** | User had already seen some documents | Everything except the named already-seen documents — list them and what they were used for |
| **Post-read seal** (degraded) | Full corpus read before split | Only what's built after the split (agents, prompts, thresholds, rubric samples) — never pre-seal narrative/process-model documents |
| **No split possible** | Corpus contains no instances to hold out | Nothing — record `prove_set: []` with the structural reason |

   - **Rulebook-only check before the scale check:** instance corpus (filled forms, real work product) → proceed, scale is the only gate. Rulebook corpus (templates/SOPs/instructions, nothing filled in) → record structurally `prove_set: []` with reason "no instances exist to hold out" and `ground_truth_available: false` as a durable fact (file count can clear the ten-case floor while holding zero instances); a template is its own answer key for a fixture-style eval — label it as that smaller kind of ground truth. Mixed → instances are the corpus for split purposes; rulebook material becomes the spec a grader checks against.
   - Don't default into it either way; don't infer need from the corpus's origin; ask what the dataset will be used for and by whom.
   - **Scale:** under roughly ten cases can't support a sealed PROVE pile — skip, or use an informal hold-out while saying it isn't a real PROVE pile.
   - If it applies, decide the split with the user **before drafting more cases**, never retroactively (early pilot cases are the worst PROVE candidates — most revised).
   - **Sizing doctrine** (only when a downstream agent/eval authoring is gated on this dataset): Phase A build floor — minimum verified examples per judgment (e.g. a handful answerable plus at least one genuinely unanswerable); one available → floor of one, say so; count each judgment's floor by hand and record counts; a judgment at zero is a placement problem raised immediately. Phase B — extraction continues in parallel in **append-only batches** for release checks/monitoring; never blocks Phase A. A real mistake found later becomes a **new tagged case** in the next batch, not an edit (still logged per Step 6). Doesn't apply when nothing downstream waits.

## Where the dataset goes

**Shared storage, versioned folder, never a repo** — not records, log, rules doc, or even a README. Layout:
```
<data-store>/<dataset-name>/README.md                # the overview
<data-store>/<dataset-name>/run_config.json          # the eval's settings for THIS environment
<data-store>/<dataset-name>/<version>/cases.json     # or the case records
<data-store>/<dataset-name>/<version>/corrections.md
<data-store>/<dataset-name>/<version>/rules.md
<data-store>/<dataset-name>/<version>/README.md      # what this version is, in detail
```
- AWS worked case: `<data-store>` = `s3://<env-data-bucket>`, one bucket per environment, name in Parameter Store at `/<service>/<env>/<NAME>`, read by exact name; other backends per eval-package `references/storage-backends.md`.
- `run_config.json` is shown only so the listing isn't a surprise: **not part of the data set, not yours to write, not handed off**; eval-package owns it; it sits outside version folders because config changes while versions are write-once.
- One data set per environment, independent and independently versioned; never one canonical set + id remap.
- Never put an environment's name or bucket inside a data set; document paths relative to the bucket.
- Version three parts, starts `1.0.0`; folder written once; revise by uploading the whole set under the next number. Patch = typo/retag; minor = added cases/reworded question; major = dropped/replaced cases or changed field shape.
- Ask **where, not whether** (Parameter Store path or bucket name); upload with explicit keys (recursive copy follows symlinks; junk is undeletable on write-once stores).
- Say the URI at hand-off (eval-package puts it in `input/source.json`).
- **Nothing local survives** — delete the working copy after upload.

**Overview README at the dataset root**, byte-identical copy kept by the consuming eval at its results-prefix root. Prose, in order: what the eval tests (behaviour, what's scored, what passing means — clearly marked gap if no eval yet); what this data set has (case count, material, mix of kinds/languages/traps, case fields, versioning rule); how results show up (one file per case per run, re-runs never overwrite, result contents); closing current state (dataset version, case count, eval version if any). Rules: 3–4 paragraphs, **1000 words max**; no references out (no relative paths, links, "see the repo"); current state only, never a changelog; every revision updates both copies together — read existing first, edit, never replace with a fresh draft (deletes the eval builder's half). If the eval package exists, publish through the project's store tooling (both copies or neither; a half-published pair is never silent). Always verify both present, byte-identical, within the limit. No eval yet → say at hand-off the eval/results sections are gaps for eval-package.

## Hard rules
- "**This is the layer before an eval, not the eval.**" No scoring vocabulary.
- "**Never batch-draft blind.**" One case at a time, live.
- "A missing citation must never be silent."
- "**Every fix to a value that was already shown to the user as a draft gets a log entry. Never fix a value silently — even an obviously-correct fix.**"
- "**Shared storage, in a versioned folder, and never a repo.**"
- Corrections log frozen at hand-off; never duplicate `expected_correction` entries into it.
- **Verification gate:** "Don't report a case as done until every field carries a citation or an explicit reason (Step 4), and the self-check pass (Step 5) has actually been run against it — not just implied by having written the record carefully."

## Sub-agents / scripts
None. Explicitly forbids background/fire-and-forget delegation of drafting.

## Output format

**Record shapes** (`record-format.md`). Governing rule: **a missing citation is never silent**; a field with no `quote`/`locators` that isn't a summary needs a `reason`.
- Fact: `{ "value": "<the extracted value>", "quote": "<verbatim text from the source>", "page": 10 }` — any specific locator (`page`, `line`, `row`, `timestamp`, `cell`).
- Summary/aggregated (schema asks for synthesis, or a single-value field's real source is a list — don't force one quote or pick only the first item): `{ "value": "<judgment-based synthesis>", "locators": [10, 11, 12], "reason": "summarized from X, Y, Z -- no single verbatim passage covers this" }` — `locators` plural, any locator type.
- Redacted (evidenced, source-tied reason; not "couldn't find it"): `{ "value": "[REDACTED]", "redacted": true, "reason": "SSN masked in source per the document's own redaction bars, page 4" }` — name the mechanism and where visible; only on the affected field; never drop the whole entity; a gold record is a ground-truth artifact, not a submission payload.
- Absent: `{ "value": null, "reason": "not stated anywhere in the source" }` — distinguish "not stated" / "out of scope for this dataset" / "provided in a separate document/system" / redacted.
- Case level: `{ "as_of": "2026-06-30 (source document's own revision date)", "fields": { ... } }`; unambiguous corpus → `"as_of": "n/a — corpus holds exactly one version of each item, confirmed at census"`. Never absent.
- What NOT to do: placeholder values referencing other fields (`"[same as some_other_field]"`); invented quotes for inferred/constructed values; absent citation looking identical to deliberate absence.

**Corrections log entries** (SKILL.md form):
```markdown
## <date> — <short title of what was wrong>

**Category:** answer correction | drafting defect

**Case:** <case id>

**Before:** <the affected field, as it stood before the fix — value, and its citation (`quote`/
`locators`/`reason`, or the redaction/absence sentinel), not the bare value alone>

**After:** <the same field, corrected — value and/or citation, whichever changed>

**Why wrong:** <what made it wrong, with evidence — a quote, a checked system of record, a
second look at the source>

**Downstream effect:** <optional — does this fix change how other cases should be drafted, or is
it isolated to this one case?>
```
(`references/corrections-log.md` additionally has a `**Field:** <which field of the record changed>` line after Case.)
```markdown
## <date> — <short title of the rule>

**Category:** process rule

**Rule:** <the new or revised rule, stated so the next case can follow it directly>

**Why:** <what prompted it — cite the case or correction-log entry that surfaced the need>
```

**Hand off:** confirmed task/target schema (incl. out-of-scope fields and why), individually cited case records, corrections log, rules document. State explicitly what is deliberately absent (runner, evaluator, scoring criteria, critical/diagnostic labels). Hand off the **URI of the version folder**, not a path or attachment. Say that eval-package reads a single flat `cases.json` (`expected` without evidence), which is **derived** from these records by whoever builds the package, lands beside them under the same version, and is not yet written — a folder URI is not a `cases_file`.

## Acceptance check
1. **User gives 12 filled protocols and a field schema with "where to find it" hints; a reachable DB codeset exists.** Agent censuses all 12 by name first, excludes out-of-scope items with reasons, checks schema codeset claims against the DB, drafts one pilot live resolving each judgment call, every field in one of the four shapes, `as_of` set; runs the self-check; any later fix to a shown value gets a categorized log entry with Before/After including citations.
2. **Corpus is 200 blank templates.** Agent records `prove_set: []` with "no instances exist to hold out" and `ground_truth_available: false`, labels templates as fixture-style ground truth, and does not force a TEACH/PROVE split.
3. **Hand-off.** Uploads with explicit keys to `<data-store>/<name>/1.0.0/`, deletes local copy, writes/updates the root README (≤1000 words, no links, current state; marks eval sections as gaps if no eval), returns the version-folder URI, says `cases.json` is derived and not yet written, and lists the deliberately absent scoring pieces.

## Gaps / source inconsistencies
- verification-checklist.md says summaries carry `pages`+reason (SKILL/record-format: `locators`) and that a problem found before hand-off "isn't itself a loggable correction" — SKILL.md Step 5 overrides: always loggable, no pre-hand-off exemption. Recreate the files as-is if fidelity to bytes matters; SKILL.md is authoritative for behavior.
- corrections-log.md entry has a `**Field:**` line the SKILL.md template lacks.
- census-and-pairing.md and segmentation.md are older subsets of SKILL.md (missing identity failure modes, seal timing, rulebook check, sizing doctrine).
