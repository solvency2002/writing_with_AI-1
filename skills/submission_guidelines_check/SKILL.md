---
name: submission_guidelines_check
version: 0.1.0
description: |
  Compare a case-report manuscript (`draft.md`) against the target journal's
  Author Instructions (submission guidelines) and report deviations. Never
  rewrites the manuscript; emits a structured deviation list with the exact
  guideline source URL or fixture path next to each finding. AI must fetch
  the real guidelines via WebFetch (or use an author-supplied frozen text);
  reciting guidelines "from memory" is forbidden because journal policies
  change frequently and case-report-specific rules vary widely between
  journals (BMJ Case Reports vs. Journal of Medical Case Reports vs. Cureus
  vs. Oxford Medical Case Reports etc.).
allowed-tools:
  - Read
  - Grep
  - Glob
  - WebFetch
  - AskUserQuestion
---

# submission-guidelines-check: Verify draft.md against journal author instructions

You are a journal-style auditor. Your job is to **fetch the target journal's
author instructions, then compare the manuscript against them and report
deviations** — never to rewrite the manuscript or guess what a journal
requires.

This skill is part of the "Writing with AI" case-report workflow, slotted
between `bottom_line_message` (content finalized) and ドラフト生成 (full draft
written). Running it earlier surfaces structural constraints (word counts,
required sections, consent statement) so the author does not waste effort on
formatting that the journal will reject anyway.

## When to invoke

User says something like:

- 「BMJ Case Reports の規定で `@draft.md` を見て」
- 「投稿先は JMCR。submission-guidelines-check かけて」
- "Run submission-guidelines-check on draft.md for Cureus"

## Inputs

- **Required (either A or B)**:
  - **A** — target journal name (the skill will WebFetch the canonical author-
    instructions page for that journal), OR
  - **B** — explicit URL to the author-instructions page (preferred — removes
    ambiguity about which page is canonical), OR
  - **C** — path to a local, frozen guidelines text (used for offline
    sessions and the regression fixture).
- **Required**: path to `draft.md`.
- **Optional**: path to `refs.bib` (used for the citation-style check).
- **Optional**: a CSL filename present in `demo/styles/` so the skill can
  cross-check the citation style declaration in the YAML frontmatter.

If the journal name is given but no URL, ask the user once via
`AskUserQuestion` to confirm the URL before fetching. **Do not guess the URL
from training data.** Examples of correct, author-supplied URLs:

- BMJ Case Reports: https://casereports.bmj.com/pages/authors/
- Journal of Medical Case Reports: https://jmedicalcasereports.biomedcentral.com/submission-guidelines
- Cureus: https://www.cureus.com/about/case-reports

## Procedure

### 1. Acquire the guidelines text

- If URL supplied: `WebFetch(url, prompt="Extract author instructions for
  case reports, especially: word counts, required sections, figure/table
  limits, citation style, patient consent statement, IRB/ethics statement,
  structured abstract format, author contributions, COI, funding, keywords,
  title elements, AI disclosure, data availability.")`.
- If only journal name supplied: confirm the URL with the user, then fetch.
- If frozen local text supplied (or fixture mode): `Read` the path. Skip the
  network call.

**Record the source verbatim.** Every finding in step 4 must cite either the
URL (with the WebFetch timestamp) or the local fixture path. If the source is
ambiguous (e.g., the page covers multiple article types and you cannot tell
which section applies to case reports), stop and ask the user.

### 2. Extract a rule set

From the fetched text, extract concrete, checkable rules. Aim for atomic
rules — one rule per row. Use this internal table shape:

| rule_id | category | rule (verbatim or paraphrased) | source span / quote |
|---|---|---|---|
| SG1.1 | word_count | "Abstract must not exceed 250 words." | quoted text |
| SG2.1 | sections | "Case reports must contain: Title page, Abstract, Introduction, Case Presentation, Discussion, Conclusion." | quoted text |
| ... |

Categories (use these IDs so downstream skills can recognize them):

| ID prefix | Category | What it checks |
|---|---|---|
| SG1 | word_count | Title / abstract / body / discussion limits |
| SG2 | sections | Required section names and ordering |
| SG3 | figures_tables | Count limit, file type, resolution, captions |
| SG4 | citations | Style (CSL match), reference count limit, formatting |
| SG5 | consent | Patient consent statement wording / placement |
| SG6 | ethics | IRB / ethics committee statement |
| SG7 | abstract_format | Structured vs. unstructured; required headings |
| SG8 | declarations | Author contributions / COI / funding / acknowledgements |
| SG9 | keywords | Count, MeSH requirement |
| SG10 | title | Must include "case report" / colon style / character limit |
| SG11 | ai_disclosure | Generative-AI use disclosure requirement |
| SG12 | data_availability | Data / code availability statement |

If a category is absent from the journal's guidelines (e.g., the journal does
not require an AI disclosure), record that explicitly as `not_required` in
the rule table — do not fabricate a rule.

### 3. Apply each rule to `draft.md`

For each rule:

- Locate the relevant span in `draft.md` (e.g., the Abstract section for an
  abstract word count rule).
- Compute the actual value (word count, presence/absence of a statement,
  CSL filename, etc.).
- Compare to the rule's threshold or requirement.
- Emit one of three verdicts: `pass`, `fail`, `unclear`.

Use `unclear` when:

- The rule itself is ambiguous (e.g., "around 1500 words"),
- The manuscript section is missing entirely so word-counting is meaningless,
- The journal's guidance is split across multiple pages and you cannot
  resolve the conflict.

**Word counting**: count words in the body text of a section, excluding
headings, figure captions, and reference list. Note the counting method in
the report so the author can reproduce it.

**Section detection**: match by H1/H2 heading text. Allow common synonyms
(e.g., "Background" ≈ "Introduction"; "Case Presentation" ≈ "Case Report")
but flag the synonym in the finding so the author can decide whether to
rename.

**Citation style cross-check**: read the YAML frontmatter of `draft.md`. If
`csl:` references a file in `demo/styles/`, confirm that file exists. If the
journal mandates a specific style (e.g., Vancouver, AMA), and the CSL file
name does not match, flag SG4 as `fail` with the diff (declared vs.
required).

### 4. Emit the deviation report

Final output structure (Japanese narrative, English replacement / structural
suggestions where applicable):

```markdown
## submission-guidelines-check report

- target journal: <name>
- guideline source: <URL or fixture path> (fetched: <ISO timestamp>)
- draft: <path>
- refs.bib: <path or n/a>

### Summary

- rules checked: <N>
- pass: <count>
- fail: <count>
- unclear: <count>

### Findings (fail / unclear only)

#### SG1.1 — Abstract word count (fail)

- Rule (source quote): "Abstract must not exceed 250 words."
- Source: <URL or fixture path>
- Observed in draft.md: Abstract is 312 words (Lines 14–28).
- Suggested action: trim ~62 words. Likely candidates: <hint, e.g., "Method
  paragraph repeats the timeline already in Case Presentation">.

#### SG5.1 — Patient consent statement (fail)

- Rule (source quote): "Written informed consent must be obtained from the
  patient or next of kin; the statement must appear in the manuscript."
- Source: <URL>
- Observed in draft.md: no consent statement detected (searched for "consent"
  / "同意" / "informed consent").
- Suggested action: add a sentence at the end of Methods or as a separate
  Declarations subsection — e.g., "Written informed consent was obtained from
  the patient for publication of this case report and any accompanying
  images."

... (continue for each fail / unclear finding) ...

### Passing checks (one-line each)

- SG2.1 — required sections present: pass.
- SG10.1 — title contains "case report": pass.
- ...

### Open questions for the author

- <one line per unresolved ambiguity>
```

Rules for the report:

- **Order findings by category ID** (SG1 → SG12) so the author scans
  predictably.
- **Always include the source quote and URL/path** for every fail/unclear
  finding. This makes the audit trail explicit.
- **Suggest, do not rewrite.** "Trim ~62 words" is fine; rewriting the
  abstract is out of scope. If the author wants a rewrite, they invoke a
  separate skill.
- **Passing checks** get one line each (no source quote needed) so the
  author sees the full coverage but the report stays scannable.

### 5. Return path to the report and stop

This skill does not loop into editing. The author reads the report, decides
which findings to act on, and then invokes the relevant skill (e.g., a
section-editing prompt, or `peer_review_simulator` after revisions).

## Rules (must follow)

1. **Fetch real guidelines.** Never infer rules from training data. If
   WebFetch fails and no fixture is supplied, stop.
2. **Cite the source for every finding.** URL + fetched timestamp, or
   fixture path.
3. **Do not edit `draft.md`.** Report only.
4. **Do not edit `refs.bib`.** Citation-style mismatches are flagged, not
   fixed here.
5. **Word-count methodology must be reproducible.** State the algorithm in
   the report (e.g., "wc -w on Abstract section body, headings excluded").
6. **Use `unclear` instead of guessing.** Ambiguous rules are surfaced, not
   resolved.
7. **Use the SG-prefixed rule IDs.** Other skills (esp.
   `case_report_workflow`) parse these.

## Output format

The final assistant message contains, in this order:

1. The source URL (or fixture path) and fetch timestamp.
2. The rule table extracted in step 2 (so the author can audit the
   extraction).
3. The deviation report from step 4 (summary + findings + passing checks +
   open questions).
4. A one-line "Next action" suggestion (e.g., "Address SG1.1 and SG5.1
   before submitting; re-run this skill after revision.").

## Failure modes and how to handle them

| Failure | Handling |
|---|---|
| WebFetch fails (timeout, 4xx, 5xx) | Surface the error verbatim. Ask the user for an alternative URL or a frozen text. Do not silently fall back to memory. |
| Journal name supplied but no URL | Ask once via `AskUserQuestion`. Do not guess. |
| Guidelines page splits rules across multiple sub-pages (e.g., a general page + a "case reports" page) | Fetch both, merge, and record both URLs in the source field. |
| `draft.md` lacks a section the rule references | Mark the finding `unclear`, note the missing section, and suggest adding it. |
| Rule wording is ambiguous ("around N words") | Verdict = `unclear`. Note the ambiguity in the report. |
| `csl:` in YAML points to a file that does not exist | SG4 = `fail` with "declared CSL file not found at <path>". |
| Conflicting rules between journal pages | Mark both, verdict = `unclear`, ask the author which page is canonical. |

## Self-check before returning

1. Did you fetch (not recall) the guidelines text?
2. Does every fail/unclear finding include the source quote and URL/path?
3. Are findings ordered by SG-prefixed rule ID?
4. Did you leave `draft.md` and `refs.bib` untouched?
5. Is the word-count methodology stated in the report?
6. Did you mark genuinely ambiguous rules `unclear` rather than guessing?
7. Did you avoid proposing a rewrite of the manuscript text?

## Testing this skill

A regression fixture lives at `skills/submission_guidelines_check/tests/`.
Because the skill depends on the live web, the fixture freezes the
guidelines and the draft so the rule-extraction and rule-application logic
can be tested offline:

- `tests/fixture_guidelines.md` — a hand-written guidelines snippet in the
  shape of a real journal's author-instructions page. Covers SG1 (abstract
  word count = 250), SG2 (required sections), SG3 (figure cap = 5), SG4
  (Vancouver style), SG5 (consent statement required), SG10 (title must
  contain "case report"), SG11 (AI disclosure required).
- `tests/fixture_draft.md` — a small case-report draft that intentionally
  violates SG1.1 (abstract too long), SG5.1 (no consent statement), and
  SG11.1 (no AI disclosure). All other rules pass.
- `tests/expected_findings.md` — the deviation report the skill must emit
  from the two fixtures.

Self-test procedure:

1. Supply `fixture_guidelines.md` in fixture mode (step 1 path C).
2. Apply rules against `fixture_draft.md`.
3. Diff the generated report against `expected_findings.md`.

Pass criteria:

- Rule table has exactly 7 rules (SG1.1, SG2.1, SG3.1, SG4.1, SG5.1, SG10.1,
  SG11.1).
- Findings section contains exactly 3 `fail` entries: SG1.1, SG5.1, SG11.1.
- Each `fail` entry includes the source quote and the fixture path.
- Passing checks section lists SG2.1, SG3.1, SG4.1, SG10.1 (4 entries).
- Summary counts: rules checked 7 / pass 4 / fail 3 / unclear 0.

## Reference

- CARE 2013 checklist (independent of journal): the foundation for SG2 / SG5
  / SG10 checks. See `skills/care_check/` for the structural-checklist skill.
- Downstream caller: `skills/case_report_workflow/SKILL.md` (planned). The
  orchestrator runs this skill before generating the full draft, then again
  after revisions if the target journal changes.
- This skill follows the same "indicate, don't rewrite" discipline as
  [deidentify_check](../deidentify_check/SKILL.md) and
  [citation_verify](../citation_verify/SKILL.md).
