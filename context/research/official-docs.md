# Investigator Mode: official-docs

Reading vendor-published official documentation — language specs, SDK references, framework guides, RFCs, and W3C/IETF standards. The authoritative source for how a language, framework, or protocol is *supposed* to behave.

## Tools

Use exactly the tools listed under `research.official-docs.tools` in the resolved config (`$HOME/.oracle/runs/<task_id>/config.resolved.json`). The public default is web search + fetch; a project's `.oracle/config.json` may add an authenticated fetcher for an internal platform's docs site. Do not assume a tool that isn't in your list.

Routing: public documentation URLs → the web search/fetch tools. A docs URL the public fetcher can't authenticate (an internal platform's docs) → the authenticated internal fetcher, if one is in your tool list; otherwise record it as an inaccessible source.

## Procedure

### 1. Identify the authoritative source

- Language stdlib → the language's official docs site.
- Framework → the vendor's documentation site.
- Protocol → the IETF RFC (datatracker.ietf.org).
- Web standard → W3C (w3.org).
- Library → the maintainer's docs, linked from the package manifest.
- An internal platform → its docs site, via the configured authenticated fetcher.

Prefer official docs over third-party tutorials whenever they exist.

### 2. Fetch the specific pages

Fetch the actual reference pages with your configured tools — not a search snippet.

### 3. Read carefully

Note the normative keywords (MUST/SHOULD/MAY per RFC 2119), the worked examples, deprecation warnings, and version notes (features introduced or removed in specific versions). Persist each page verbatim to `evidence/`.

### 4. Identify the spec version

Record the version of the spec/SDK/framework, whether it matches the project's actual version, and any breaking changes between them.

### 5. Cross-reference

For official docs, one authoritative source is high confidence — the vendor *is* the truth. Cross-reference for completeness. RFC text needs no external validation.

### 6. Counter-evidence

Check for errata, a newer version, or "this section is being revised" markers. Record what you checked.

## Findings Schema

```yaml
findings:
  - claim: "<assertion>"
    source_url: "<official doc URL>"
    spec_version: "<version>"
    rfc_clause: "<MUST/SHOULD/MAY + section, if applicable>"
    authority: vendor-published | RFC | W3C | language-spec
    cross_reference: ["<related doc>"]
    counter_evidence: "<errata or supersession searched>"
```

## Critical Rules

- Official docs are AUTHORITATIVE for language/framework behavior.
- The spec version MUST match the project's actual usage — flag a mismatch.
- RFC clauses MUST cite the section number (e.g. "RFC 9110 §9.2.1").
- URLs MUST come from actual tool responses.
- Use only the tools in `research.official-docs.tools`. If a docs site needs a tool you weren't given, record it as a gap.

## Evidence Persistence

Persist every fetched artifact verbatim under `$HOME/.oracle/runs/<task_id>/evidence/<source_hash>.<ext>` and append to `evidence/index.jsonl`:

```json
{"hash":"<12-hex-sha256>","url":"<original URL or path>","fetched_at":"<ISO>","medium":"official-docs"}
```

## Output

Write `<output_path>/findings.md` and `<output_path>/verdict.json`. Append a `done` manifest line via `bin/oracle-manifest-append` after writing.
