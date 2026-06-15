# Investigator Mode: blog

Reading public web content — engineering blog posts, Stack Overflow answers, library documentation sites, vendor announcements.

## Tools

Use exactly the tools listed under `research.blog.tools` in the resolved config (`$HOME/.oracle/runs/<task_id>/config.resolved.json`). For public web content this is web search + fetch. Do not assume a tool that isn't in your list.

## Procedure

### 1. Formulate search queries

Include library/framework name + version, specific error messages, domain terms. Use multiple phrasings — search engines reward diversity.

### 2. Execute searches

Run your configured web-search tool with multiple queries in parallel. Record the top results.

### 3. Filter by source quality

1. **Official docs** — vendor-published reference (highest)
2. **Engineering blogs** — large company blogs (Netflix, Stripe, GitHub, etc.)
3. **Stack Overflow** — high votes AND recent timestamps
4. **Personal blogs** — useful but lower confidence
5. **Forum threads** — lowest, often outdated

A single personal-blog claim is capped at "low" confidence.

### 4. Fetch full content

Fetch full articles with your configured web tools, not snippets. Persist each to `evidence/`.

### 5. Identify currency

- Date published (and last updated)
- Library/framework version addressed
- Author and authority

Stale content (>2 years for fast-moving frameworks) MUST be flagged.

### 6. Cross-reference

At least the `thresholds.research_cross_reference_floor` config value of independent sources is required. Independent = different authors and domains (not 3 articles citing same SO answer).

### 7. Counter-evidence

Search for contradicting sources: newer articles, deprecation announcements, "X considered harmful" posts.

## Findings Schema

```yaml
findings:
  - claim: "<assertion>"
    source_url: "<article URL>"
    source_quality: official | engineering-blog | stack-overflow | personal-blog | forum
    date_published: "<ISO date>"
    library_version: "<version if applicable>"
    cross_reference:
      - "<other URL>"
    counter_evidence: "<contradicting articles searched>"
```

## Critical Rules

- Source quality tier MUST be reported.
- Date currency MUST be assessed.
- Single personal-blog claims capped at "low" confidence.
- Cross-references MUST be from different authors/domains.
- Counter-evidence search MANDATORY.
- URLs MUST come from actual tool responses.

## Evidence Persistence

Every fetched artifact MUST be persisted verbatim under:

```
$HOME/.oracle/runs/<task_id>/evidence/<source_hash>.<ext>
```

Append to `$HOME/.oracle/runs/<task_id>/evidence/index.jsonl`:

```json
{"hash":"<12-hex-sha256>","url":"<original URL or path>","fetched_at":"<ISO>","medium":"blog"}
```

## Output

Write `<output_path>/findings.md` and `<output_path>/verdict.json`. Append a `done` manifest line via `bin/oracle-manifest-append` after writing.
