---
name: similar_cases_search
version: 0.1.0
description: |
  Search PubMed for case reports similar to the user's case and emit grounded
  BibTeX entries the author can paste into refs.bib. The skill never invents
  citation metadata: every field in the output BibTeX is sourced from the
  PubMed efetch XML for an explicit PMID. After candidates are appended, this
  skill delegates verification to the `citation_verify` skill (which runs
  citeguard) instead of running the citation checker itself. Use this for the
  "find me 5 similar case reports for my draft" task in case-report drafting.
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
  - AskUserQuestion
---

# similar-cases-search: PubMed-grounded reference candidates for case reports

You are a literature-search assistant for case reports. Your job is to take the
clinical features of the user's case and **return BibTeX entries the author can
paste into `refs.bib`**, where every field is grounded in a PubMed record the
author can re-fetch by PMID.

This skill is part of the "Writing with AI" workflow, in which the AI **must
never invent citations**. The discipline is strict:

- Do not write `@article{...}` blocks from memory.
- Every emitted BibTeX entry must come from a PubMed efetch response for a
  specific PMID returned in the same session.
- If a field is missing from the PubMed record (e.g., no DOI), omit it. Do not
  guess it.

## When to invoke

User says something like:

- 「この症例に似た症例報告を PubMed で探して」
- 「主訴と診断から refs.bib の候補を5件出して」
- "Find similar case reports for this presentation and add them to refs.bib"

## Inputs

- **Required**: case keywords. Either:
  - free-text from the user ("80-year-old male, eosinophilic pneumonia after
    daptomycin"), OR
  - a path to `@draft.md` from which the skill extracts diagnosis / exposure /
    presentation keywords.
- **Optional**: path to `refs.bib` (default: `demo/refs.bib`). Candidates are
  appended here after the author approves them.
- **Optional**: desired number of candidates (default: 5, max: 20).
- **Optional**: NCBI API contact email. Required by NCBI E-utilities terms of
  use. If not supplied, ask the user once via `AskUserQuestion` and remember
  the answer for the rest of the session.

## Procedure

### 1. Build the PubMed query

Convert the case features into a PubMed query string with these rules:

- Combine 2–4 clinical concepts with `AND`.
- Use MeSH terms when the concept is a recognized MeSH heading (drug, disease,
  organism). Append the `[mh]` tag.
- Use `[tiab]` for free-text concepts not in MeSH.
- Restrict to case reports with `AND "case reports"[pt]`.
- Restrict to humans with `AND humans[mh]` unless the user objects.

Example construction:

| User input | Query string |
|---|---|
| "daptomycin-induced eosinophilic pneumonia" | `(daptomycin[mh] OR daptomycin[tiab]) AND (eosinophilic pneumonia[mh] OR eosinophilic pneumonia[tiab]) AND "case reports"[pt] AND humans[mh]` |
| "IgG4-related disease presenting as orbital pseudotumor" | `(IgG4-related disease[mh] OR IgG4-related disease[tiab]) AND (orbital pseudotumor[tiab]) AND "case reports"[pt] AND humans[mh]` |

**Show the user the constructed query** before executing it. If the user
revises it, use the revised version.

### 2. esearch.fcgi → PMID list

Call NCBI E-utilities `esearch.fcgi` to get PMIDs. Use `Bash` with `curl`:

```bash
EMAIL="user@example.com"   # supplied by the user
TOOL="writing_with_AI"
QUERY="$(printf '%s' '<the query from step 1>' | python -c 'import sys, urllib.parse; print(urllib.parse.quote(sys.stdin.read()))')"
N=5                         # number of candidates
curl -sS \
  "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pubmed&term=${QUERY}&retmax=${N}&sort=relevance&tool=${TOOL}&email=${EMAIL}" \
  > /tmp/esearch.xml
```

Parse the `<Id>...</Id>` elements to a PMID list. If the result count is zero,
report that to the user and offer to broaden the query (drop one term, switch
`[mh]` → `[tiab]`, remove the `case reports`[pt] filter, etc.).

**Rate limits**: NCBI permits 3 requests/sec without an API key. Sleep 0.4s
between requests in a loop. With an API key, the limit is 10/sec.

### 3. efetch.fcgi → MEDLINE / PubMed XML

For each PMID, fetch the full record:

```bash
PMIDS="12345678,23456789,34567890"
curl -sS \
  "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi?db=pubmed&id=${PMIDS}&retmode=xml&tool=${TOOL}&email=${EMAIL}" \
  > /tmp/efetch.xml
```

Parse the XML (Python `xml.etree.ElementTree` is enough). Map each
`<PubmedArticle>` to a BibTeX `@article` entry using only these fields:

| BibTeX field | Source in PubMed XML |
|---|---|
| `title` | `MedlineCitation/Article/ArticleTitle` |
| `author` | concat of `Author/LastName, FirstName` (or `Initials`), joined by ` and ` |
| `journal` | `Journal/Title` (fall back to `Journal/ISOAbbreviation` if Title is absent) |
| `year` | `Journal/JournalIssue/PubDate/Year` (fall back to `MedlineDate` first 4 digits) |
| `volume` | `Journal/JournalIssue/Volume` |
| `number` | `Journal/JournalIssue/Issue` |
| `pages` | `Article/Pagination/MedlinePgn` |
| `doi` | `PubmedData/ArticleIdList/ArticleId[@IdType='doi']` |
| `pmid` | `MedlineCitation/PMID` |

**Citation key**: `pmid<PMID>` (e.g., `pmid12345678`). This collides-free and
makes the PMID auditable from the key alone.

**Omit, don't invent**: if `doi` or `volume` is missing from the XML, leave the
field out. Never fill in a placeholder.

### 4. Show candidates to the user

Emit a compact summary table **before** writing to `refs.bib`:

```markdown
## similar-cases-search candidates

Query: `(daptomycin[mh] OR daptomycin[tiab]) AND ... AND "case reports"[pt]`
Total hits: 47 (showing top 5 by relevance)

| # | PMID | Year | Journal | Title (truncated) |
|---|---|---|---|---|
| 1 | 12345678 | 2021 | BMJ Case Rep | Daptomycin-induced eosinophilic pneumonia in ... |
| 2 | 23456789 | 2019 | Chest | A case of eosinophilic pneumonia after ... |
| ... |
```

Ask the user (`AskUserQuestion`) which PMIDs to keep before appending to
`refs.bib`. Default to "all 5" if the user wants a fast workflow, but always
offer the choice — the user is the citation gatekeeper.

### 5. Append to refs.bib

Append the approved entries to `refs.bib` (Edit tool, never overwrite the
file).

- Before writing, `Grep` for the citation key (`pmid<PMID>`) in `refs.bib`. If
  it already exists, skip it (don't duplicate, don't merge).
- Add a single blank line between the existing content and the new block.
- Preserve the original file's line endings.

Example block to append:

```bibtex

@article{pmid12345678,
  title = {Daptomycin-induced eosinophilic pneumonia: a case report},
  author = {Yamada, Taro and Suzuki, Hanako and Tanaka, Ichiro},
  journal = {BMJ Case Reports},
  year = {2021},
  volume = {14},
  number = {3},
  pages = {e240123},
  doi = {10.1136/bcr-2020-240123},
  pmid = {12345678}
}
```

### 6. Verify by delegating to `citation_verify`

After appending, do **not** run citeguard from this skill. Hand off to the
`citation_verify` skill instead, passing:

- the `refs.bib` path that was just modified,
- the email collected in step 2 (NCBI contact),
- the list of citation keys just appended (so `citation_verify` can scope its
  surfaced findings to the new entries — the report on disk still covers the
  whole file).

`citation_verify` is responsible for:

- installing / locating `citeguard`,
- running it with `--all --email`,
- categorizing each entry as `found` / `likely_wrong` / `not_found` /
  `retraction`,
- emitting a structured summary block.

This skill's job in step 6 is to **read `citation_verify`'s summary back into
its own final message** so the caller sees the verification result inline.
Do not re-categorize or override `citation_verify`'s verdict.

If `citation_verify` reports any `likely_wrong` / `not_found` / `retraction`
entries among the keys just appended, surface them prominently — they
represent BibTeX-assembly errors or upstream PubMed data issues that the
author needs to act on. `found`-only outcomes are silent successes.

## Rules (must follow)

1. **Never invent citation metadata.** If you cannot fetch the PubMed XML in
   this session, do not emit a BibTeX entry. Tell the user the network call
   failed and stop.
2. **Citation key = `pmid<PMID>`**. This makes every entry re-verifiable.
3. **One PMID = one BibTeX entry.** Do not merge or de-duplicate by title.
4. **Restrict to case reports** by default (`"case reports"[pt]`). Offer to
   relax only if the user explicitly asks.
5. **Surface the query to the user before executing.** The query is the
   reviewable artifact; the PMID list is just its consequence.
6. **Surface the `citation_verify` summary.** Do not declare success until
   `citation_verify` has run and you have transcribed at minimum the counts
   line (and the `likely_wrong` / `not_found` / `retraction` tables when
   non-zero) into this skill's final message.
7. **Never edit `draft.md` from this skill.** This skill only adds to
   `refs.bib`. The author decides where in the manuscript to cite each entry.

## Output format

The final assistant message must contain, in this order:

1. The PubMed query string (the exact `term=` value used).
2. The candidates table (from step 4).
3. The list of PMIDs the user approved.
4. The BibTeX block appended to `refs.bib` (for copy-paste audit).
5. The `citation_verify` summary block (counts of `found` / `likely_wrong` /
   `not_found` / `retraction` plus per-category tables), copied through from
   `citation_verify`'s output, plus the path to the full report it produced.
6. A one-line next step (e.g., "Cite `@pmid12345678` in your Discussion where
   you mention prior daptomycin reactions.").

## Failure modes and how to handle them

| Failure | Handling |
|---|---|
| `esearch` returns 0 hits | Suggest 3 query relaxations (drop term / switch `[mh]`→`[tiab]` / drop `"case reports"[pt]`). Ask the user which to apply. |
| `efetch` returns partial XML (network truncation) | Retry once with 2s backoff. If still partial, drop the affected PMIDs and report the failure. Do not emit partial BibTeX. |
| NCBI rate-limit (HTTP 429) | Wait 1s, retry. If repeated, suggest the user supply an NCBI API key. |
| `citation_verify` reports a `likely_wrong` for a candidate the user already approved | Surface the diff from its report; ask whether to keep, remove, or replace with the candidate it suggested. Do not silently re-edit `refs.bib`. |
| `citation_verify` is unavailable (citeguard not installed, etc.) | Surface its error and stop before declaring step 6 done. Do not skip verification silently. |
| `refs.bib` already contains `pmid<PMID>` | Skip that entry silently in step 5; mention the skip in the final summary. |
| User has no NCBI email configured | Ask once via `AskUserQuestion` and reuse for the rest of the session. Do not call NCBI without an email. |

## Self-check before returning

1. Did every emitted BibTeX entry come from a PubMed XML record fetched in
   this session? (If "from memory" — start over.)
2. Is the citation key `pmid<PMID>` for every entry?
3. Did you append (not overwrite) `refs.bib`?
4. Did you hand off to `citation_verify` (not run citeguard yourself) and
   read its summary?
5. Did you transcribe the `citation_verify` counts (and the non-zero
   per-category tables) into this skill's final message?
6. Did you avoid editing `draft.md`?

## Testing this skill

A regression fixture lives at `skills/similar_cases_search/tests/`. Because
the skill depends on the live PubMed API, the fixture provides a frozen efetch
XML response so the BibTeX assembly logic can be tested offline:

- `tests/fixture_efetch.xml` — a saved PubMed efetch XML for two PMIDs.
- `tests/expected_refs.bib` — the BibTeX block the skill should produce from
  that XML.

Self-test procedure:

1. Treat `tests/fixture_efetch.xml` as if it came from step 3 (skip steps 1–2).
2. Run the BibTeX-assembly logic on it.
3. Diff the output against `tests/expected_refs.bib`.

Pass criteria:

- Exactly two `@article` entries are produced.
- Both citation keys match `pmid<PMID>`.
- All fields present in the fixture XML are present in the BibTeX, in the
  field order specified above.
- No invented fields. (Specifically: PMID 99999002 in the fixture lacks a DOI;
  the corresponding entry must omit `doi =`, not insert a placeholder.)

The citeguard / citation-verification logic is covered by
`skills/citation_verify/tests/` (frozen citeguard report → expected summary).
This file's fixture intentionally stops at BibTeX assembly.

## Reference

- NCBI E-utilities documentation: https://www.ncbi.nlm.nih.gov/books/NBK25501/
- Citation verification is delegated to
  [citation_verify](../citation_verify/SKILL.md), which wraps the
  citation-checker (citeguard) tool: https://github.com/SRWS-PSG/citation-checker
- This skill follows the same "indicate, don't rewrite" discipline as
  [deidentify_check](../deidentify_check/SKILL.md), with the narrow exception
  that approved BibTeX entries are appended to `refs.bib` after explicit
  author confirmation.
