---
name: case_report_workflow
version: 0.1.0
description: |
  End-to-end orchestrator for writing a medical case report. Calls each
  case-report sub-skill in a fixed sequence, gates progress on author
  confirmation at every step, writes `draft.md` only in the drafting and
  revision steps, and produces a single hand-off package at the end. The
  orchestrator never invents citations, never edits `draft.md` outside the
  two designated steps, and never silently skips verification — failed
  verifications are surfaced and require explicit author override.
  Triggers: 「症例報告ワークフロー」「case-report-workflow」「症例報告を最初から書く」.
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Bash
  - AskUserQuestion
---

# case-report-workflow: End-to-end case-report drafting orchestrator

You are the conductor for the entire case-report drafting pipeline. Your job
is to **call sub-skills in order, surface their outputs back to the author,
and gate progress on explicit author confirmation** at every step. You do not
duplicate the sub-skills' logic — when a sub-skill exists for a task, you
invoke it; you do not re-implement it.

This skill is the only place where the full sequence lives. Sub-skills can be
called independently for one-off tasks (e.g., the author just wants a CARE
audit on an existing draft → invoke `care_check` directly); the orchestrator
is for the case where the author wants to write a case report start to
finish with AI assistance.

## When to invoke

User says something like:

- 「これから症例報告を書きたい。最初から最後まで案内して」
- 「case-report-workflow を起動して」
- "Start a case-report workflow for this case"
- The CLAUDE.md project hint points to this skill as the entry point for
  case-report work.

If the author only wants a single sub-skill task (e.g., "just check CARE"),
invoke that sub-skill directly. **Do not run the full orchestrator for a
single-step request.**

## Inputs

- **Required at start**: either a case summary (free text) or an existing
  `@draft.md` path.
- **Required eventually** (collected via `AskUserQuestion` as needed):
  - NCBI E-utilities contact email (for `similar_cases_search` and
    `citation_verify`).
  - Target journal name + author-instructions URL (for
    `submission_guidelines_check`).
  - Article type (for `peer_review_simulator`): one of `single_case`,
    `surgical_case`, `case_series`, `image_case`, `diagnostic_challenge`.
- **Optional**: existing `refs.bib` path (default: `demo/refs.bib` for
  workshop sessions; otherwise expect `projects/<name>/refs.bib`).

Collect inputs lazily — ask only when the step that needs them is reached.
Do not block the workflow at step 1 collecting every input upfront.

## State tracking

Maintain a workflow state block in your responses so the author can see
where they are. Emit it as the **first block** of every reply during the
workflow:

```markdown
### case-report-workflow state

| step | name | status | artifact |
|---|---|---|---|
| 1 | input collected | pending / done | <case summary or draft path> |
| 2 | deidentify_check | pending / done / skipped | <findings inline> |
| 3 | similar_cases_search | pending / done / skipped | <refs.bib path + N entries added> |
| 4 | citation_verify (initial) | pending / done / skipped | <counts: found/likely_wrong/not_found/retraction> |
| 5 | bottom_line_message | pending / done / skipped | <BLM finalized? yes/no> |
| 6 | submission_guidelines_check | pending / done / skipped | <fail/unclear count> |
| 7 | draft generated | pending / done | <draft.md path> |
| 8 | peer_review_simulator + care_check | pending / done / skipped | <Major comment count, CARE missing items> |
| 9 | revision applied | pending / done / skipped | <revision diff summary> |
| 10 | citation_verify (final) | pending / done / skipped | <counts> |
| 11 | hand-off | pending / done | <hand-off package path> |
```

Update the state block after every step transition. Mark steps `skipped`
(not `done`) when the author explicitly chose to skip — this preserves the
audit trail.

## Procedure

### Step 1 — Input collected

- Confirm whether the author has a case summary, a draft, or both.
- If only a case summary: tell the author Step 7 will create `draft.md`
  later; for now, file the summary as a workflow note.
- If only a draft: read it and surface a 5-line synopsis (presentation /
  course / outcome / claimed lesson / open questions) so both parties are
  on the same page about what's being reported.
- If neither: stop and ask which input the author wants to provide.

Transition to Step 2 only after the author confirms the synopsis is
accurate (or the case summary captures the case as intended).

### Step 2 — Deidentify check

Invoke `deidentify_check`. Surface its findings table inline. **Do not
auto-edit the draft.** The author decides which findings to act on; this
orchestrator only ensures the audit happens.

If the deidentify_check report contains D-prefixed findings (D1–D9), pause
and ask whether the author wants to address them before Step 3 (recommended
when running on existing draft) or defer (acceptable for a case summary
that hasn't been turned into a draft yet — Step 7 will incorporate
de-identification guidance into the generated draft).

If the author skips Step 2, surface the skip explicitly in the workflow
state and tell them they should re-run before submission. Do not let the
workflow finish without at least one deidentify_check pass.

### Step 3 — Similar cases search

Confirm or collect the NCBI email. Invoke `similar_cases_search` with the
case features extracted from Step 1. Surface its candidates table and let
the author choose which PMIDs to keep. Approved entries are appended to
`refs.bib` by `similar_cases_search` itself (not by this orchestrator).

If the author skips Step 3 (e.g., they already have `refs.bib` finalized
from prior literature work), still proceed to Step 4 — the verification
step is non-negotiable.

### Step 4 — Citation verify (initial pass)

Invoke `citation_verify` against `refs.bib`. Transcribe its summary into
the workflow state. If `likely_wrong` / `not_found` / `retraction` counts
are non-zero, **pause** and ask the author to resolve before Step 5. Do
not let the workflow advance with unverified citations except by explicit
author override (which is recorded in the state).

### Step 5 — Bottom-line message

Invoke `bottom_line_message`. This skill runs a four-phase dialogue with
the author; the orchestrator's job is to hand control to it and resume
when the dialogue concludes. The deliverable is the "2 findings + bottom
line message candidates + PMID-to-finding map".

Do not auto-paste the BLM into `draft.md` yet — the author confirms the
final BLM wording, and it lands in the draft during Step 7 (or Step 9 if
a draft already existed).

### Step 6 — Submission guidelines check

Confirm or collect the target journal name + author-instructions URL.
Invoke `submission_guidelines_check`. Surface its rule table and findings.

If `fail` count is non-zero **and** a draft already exists (workflow
entered with `@draft.md`), surface the fails as inputs for the Step 9
revision pass. If no draft exists yet, transcribe the rules into a "Step 7
generation constraints" block so the draft generator (Step 7) honors them
from the start (e.g., abstract word limit, required sections, consent
statement placement).

If `fail` count is zero, the constraint block still gets transcribed —
the draft must continue to satisfy the rules through revision.

### Step 7 — Draft generated

This is the **first** step where the orchestrator writes to `draft.md`.

- If a draft already exists, **skip generation** and proceed to Step 8.
  This orchestrator does not rewrite an existing draft from scratch.
- If no draft exists, generate one in Markdown using:
  - the case summary from Step 1,
  - the BLM and findings from Step 5,
  - the journal constraints from Step 6,
  - the `refs.bib` entries from Step 3 (cite by `@pmid<PMID>`).

Drafting rules:

- Use the structure mandated by Step 6's SG2 rule (or CARE 2013 ordering
  if no journal-specific rule was extracted).
- Cite only `@pmid<PMID>` keys that exist in `refs.bib`. Grep `refs.bib`
  before each citation; if a key is absent, do not invent it — instead
  insert a `[TODO: similar_cases_search for <topic>]` placeholder.
- Use English for body text (workshop default; ask the author if Japanese
  is wanted).
- Keep the draft at the journal's word limits.
- Add a "Patient Consent" subsection if SG5 required it (placeholder text:
  "Written informed consent was obtained from the patient ..."); ask the
  author to confirm wording.
- Add an "AI Disclosure" subsection if SG11 required it.

Write the file via `Write` (new file) only. If `draft.md` already exists,
refuse to overwrite (the workflow would have skipped to Step 8).

### Step 8 — Peer review + CARE

Invoke `peer_review_simulator` and `care_check` in parallel (they read
the same `draft.md` and produce independent reports). Confirm the
article type for `peer_review_simulator` (single_case / surgical_case /
etc.) before invoking.

Aggregate the two reports into a single revision punch-list:

- CARE-shaped items (missing checklist elements) → from `care_check`.
- Reviewer-shaped items (Major / Minor / Confidential) → from
  `peer_review_simulator`.

De-duplicate where they overlap (e.g., both may flag a missing consent
statement — keep one entry, attribute to both skills).

Surface the merged punch-list and ask the author which items to accept
for the Step 9 revision. The author may dismiss items they disagree
with — record dismissals in the workflow state with a one-line reason.

### Step 9 — Revision applied

This is the **second** step where the orchestrator writes to `draft.md`.

For each accepted punch-list item, apply the smallest targeted edit that
resolves the item — typically `Edit` calls scoped to a single paragraph
or section. Rules:

- One edit per item (preserves traceability).
- Re-read the affected section after each edit before moving to the
  next.
- Do not re-write paragraphs the author did not request changes to.
- Cite only keys already in `refs.bib`. If a Major comment requested a
  new citation, re-invoke `similar_cases_search` (which loops to Step 4
  to verify) before adding the citation.
- If `peer_review_simulator` flagged causality overstatement, the
  rewording must remove the overstated claim **without** asking AI to
  generate new clinical content beyond the literature already in
  `refs.bib`.

When revision is complete, surface a diff summary (touched sections,
sentence-count change per section) in the workflow state.

### Step 10 — Citation verify (final pass)

Re-invoke `citation_verify` on the updated `refs.bib`. Confirm:

- All `@pmid<PMID>` keys cited in `draft.md` exist in `refs.bib`.
- `citation_verify` returns zero `likely_wrong` / `not_found` /
  `retraction` for the currently-cited keys (the report may still cover
  unused entries — focus the surface on cited keys).

If any failures, loop back to Step 9 for that specific entry. Do not
proceed to Step 11 with broken citations.

### Step 11 — Hand-off

Produce a hand-off package as a single Markdown summary the author can
paste into a project log or share with co-authors. Contents:

- Final `draft.md` path + word counts per section.
- Final `refs.bib` path + entry count + citation_verify counts.
- BLM (the chosen candidate from Step 5).
- Journal target + the SG findings that were resolved (and any that
  remain `unclear`).
- Reviewer punch-list with each item marked `addressed` / `dismissed`
  (with reason).
- AI disclosure draft text (mirroring SG11 if required).
- A short "next actions" list for the author (e.g., "Co-author review",
  "Consent form attached separately", "Submit via journal portal").

Write the hand-off package to `<draft_dir>/handoff.md`. Tell the author
the workflow is complete.

## Rules (must follow)

1. **Sub-skills do the work.** This orchestrator does not re-implement
   citation verification, CARE audits, etc.
2. **`draft.md` writes only at Step 7 and Step 9.** Every other step is
   read-only on the manuscript.
3. **No citation invention.** All citations come through
   `similar_cases_search` and are verified by `citation_verify`. Any
   `@article` block written outside that path is a workflow failure.
4. **Pause on verification failure.** Step 4 and Step 10 pause if
   `citation_verify` reports issues. Override requires explicit author
   action, recorded in state.
5. **At least one `deidentify_check` pass** must occur before Step 11.
   Step 2 may be skipped at the start, but the orchestrator must loop
   back before hand-off if it was skipped.
6. **State block is the first artifact of every reply.** The author
   uses it to navigate; do not bury it.
7. **CARE audit lives in `care_check`, not here.** Step 8 invokes it; do
   not embed CARE logic in this orchestrator.
8. **Skips are explicit and recorded.** `skipped` ≠ `done` in the state
   block.

## Output format

The final assistant message (at workflow completion) must contain, in
this order:

1. The final state block (all rows `done` or `skipped`).
2. The hand-off package contents (from Step 11).
3. The hand-off package path (`handoff.md`).
4. A one-line confirmation: "Case-report workflow complete. Review
   `handoff.md` and the final `draft.md` before submission."

For intermediate messages during the workflow, surface:

1. The updated state block.
2. The current step's sub-skill output (transcribed in full or by
   reference to a longer artifact file).
3. An `AskUserQuestion` for the next gate (proceed / revise / skip).

## Failure modes and how to handle them

| Failure | Handling |
|---|---|
| Sub-skill not available (file missing, etc.) | Surface which skill is missing. Stop. Do not silently bypass. |
| `citation_verify` returns non-zero failures and the author overrides | Record the override in state with the author's one-line justification; require it again at Step 10. |
| Author wants to skip Step 2 (deidentify) at the start | Allowed at start. The orchestrator loops back to run it before Step 11. If still skipped at Step 11, the workflow does **not** complete — issue a hard stop. |
| Author already has a complete `draft.md` at workflow start | Skip Step 7. Steps 8–11 run normally. State row 7 = `skipped (draft pre-existed)`. |
| Step 8 reveals article type is `case_series` or `surgical_case` after the workflow started as `single_case` | Re-run `peer_review_simulator` with the corrected type. Surface the change in state. |
| Conflicts between `care_check` and `peer_review_simulator` (one says present, the other says missing) | Trust the more specific one based on Grep evidence in the report. If neither has line evidence, ask the author. |
| The journal-target answer is "undecided" at Step 6 | Skip SG check with explicit author confirmation. Use CARE 2013 ordering as the default in Step 7. The workflow can still complete, but the state row records the skip. |

## Self-check before returning at hand-off

1. Are all 11 state rows `done` or `skipped` (no `pending` remaining)?
2. Was `deidentify_check` run at least once?
3. Was `citation_verify` run at the end (Step 10), and did it return
   clean for cited keys?
4. Did `draft.md` get written only at Step 7 and Step 9?
5. Are all `@pmid<PMID>` citations in `draft.md` present in `refs.bib`
   (Grep confirms)?
6. Does the hand-off package list each reviewer punch-list item as
   `addressed` or `dismissed` (no `pending`)?
7. Is the hand-off written to `<draft_dir>/handoff.md`?

## Testing this skill

A regression fixture lives at `skills/case_report_workflow/tests/`:

- `tests/fixture_session_log.md` — a hand-crafted "session transcript"
  showing each step's sub-skill output as if the sub-skills had been
  invoked in sequence. Stands in for live sub-skill invocations during
  offline testing.
- `tests/expected_state_block.md` — the final state block after all
  steps complete.
- `tests/expected_handoff.md` — the expected hand-off package contents.

Self-test procedure:

1. Treat `fixture_session_log.md` as the substrate (skip live sub-skill
   invocations).
2. Run the aggregation / state-tracking / hand-off-package-generation
   logic on it.
3. Diff:
   - Generated final state block against `expected_state_block.md`.
   - Generated hand-off package against `expected_handoff.md`.

Pass criteria:

- All 11 state rows are `done` or `skipped` (none `pending`).
- Hand-off lists every reviewer punch-list item as `addressed` or
  `dismissed` (none `pending`).
- The hand-off package is written to `handoff.md` adjacent to
  `draft.md`.
- The package's `@pmid<PMID>` citations all resolve to entries in the
  fixture's `refs.bib`.

## Sub-skills called

In invocation order:

1. [deidentify_check](../deidentify_check/SKILL.md)
2. [similar_cases_search](../similar_cases_search/SKILL.md)
3. [citation_verify](../citation_verify/SKILL.md)
4. [bottom_line_message](../bottom_line_message/SKILL.md)
5. [submission_guidelines_check](../submission_guidelines_check/SKILL.md)
6. (Step 7 = draft generation, no sub-skill)
7. [peer_review_simulator](../peer_review_simulator/SKILL.md) +
   [care_check](../care_check/SKILL.md) (parallel)
8. (Step 9 = revision, no sub-skill; may re-invoke
   `similar_cases_search` if Major comments require new literature)
9. [citation_verify](../citation_verify/SKILL.md) (re-invoked)
10. [case_timeline](../case_timeline/SKILL.md) — optional, on author
    request during Step 7 or Step 9 for figure generation.

## Reference

- This orchestrator follows the "Markdown is the source of truth, AI does
  partial revisions" discipline from the workshop README. It does not
  introduce new editorial discipline beyond what the sub-skills already
  enforce; it sequences them.
- Adjacent: [humanizer_academic](../humanizer_academic/SKILL.md) for
  voice/tone adjustments, [render_and_upload](../render_and_upload/SKILL.md)
  for Markdown → Google Docs export after Step 11.
