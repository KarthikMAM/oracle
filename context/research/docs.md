# Investigator Mode: docs

Reading authored documentation as evidence — wikis, design docs, RFCs, READMEs, architectural decision records (ADRs), and ticket descriptions.

## Tools

Use exactly the tools listed under `research.docs.tools` in the resolved config (`$HOME/.oracle/runs/<task_id>/config.resolved.json`). The config is the single source of truth for which tools exist in this environment — the public default lists local-file and web tools; a project's `.oracle/config.json` may add documentation-system tools (a wiki, an issue tracker, an authenticated internal fetcher). Do not assume a tool that isn't in your list.

General routing within whatever tools you were given:
- Local repo docs (README, `docs/`, ADRs) → the file tools (Read, Glob, Grep).
- A wiki or issue tracker → its search tool, then its content/get tool for the full page or issue.
- A URL the public web fetcher can't authenticate (an intranet page) → the authenticated internal fetcher, if one is in your tool list; otherwise record it as an inaccessible source in your gaps.

## Procedure

### 1. Identify document sources

Project README and `docs/` directory, in-repo ADRs, wiki pages, tickets with design context, internal docs sites, and external library/framework docs. Enumerate the sources reachable with your configured tools.

### 2. Search broadly

Run multiple searches in parallel. Use distinctive terms from the angle, then progressively broaden. For a wiki: search by keyword, narrow by space if known, and check page history for how a decision evolved. For an issue tracker: query for the relevant tickets and read the full issue including comments.

### 3. Read full documents

For every relevant match, fetch the FULL document — not the search snippet. Persist each verbatim to `evidence/`.

### 4. Cross-reference

At least the `thresholds.research_cross_reference_floor` config value of independent documents for a load-bearing claim. Different authors, time periods, and doc systems strengthen confidence.

### 5. Distinguish status

For each finding, identify the document's status:
- **Authoritative** (RFC accepted, ADR ratified, README on the main branch)
- **Proposed** (draft RFC, open ADR, PR discussion)
- **Historical** (superseded, deprecated, archived)

Stale or superseded docs MUST be flagged.

### 6. Counter-evidence

Actively search for contradicting documents: newer ADRs, deprecation annotations, conflicting wiki pages, opposing RFCs. Record what you searched even when it found nothing.

## Findings Schema

```yaml
findings:
  - claim: "<assertion>"
    source_url: "<doc URL or path>"
    document_status: authoritative | proposed | historical
    document_date: "<ISO date if available>"
    author: "<doc author>"
    cross_reference:
      - "<other doc URL>"
    counter_evidence: "<contradicting docs searched>"
```

## Critical Rules

- Document status MUST be reported; stale documents MUST be flagged.
- Citations MUST come from successful tool calls — never from memory.
- Counter-evidence search is MANDATORY.
- Use only the tools in `research.docs.tools`. If a source needs a tool you weren't given (e.g. an authenticated fetcher for an intranet page), record it as a gap rather than guessing.

## Evidence Persistence

Persist every fetched artifact verbatim under `$HOME/.oracle/runs/<task_id>/evidence/<source_hash>.<ext>` and append to `evidence/index.jsonl`:

```json
{"hash":"<12-hex-sha256>","url":"<original URL or path>","fetched_at":"<ISO>","medium":"docs"}
```

## Output

Write `<output_path>/findings.md` and `<output_path>/verdict.json`. Append a `done` manifest line via `bin/oracle-manifest-append` after writing (the artifact exists before the manifest entry — the existence invariant).
