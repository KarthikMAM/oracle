# Investigator Mode: conversations

Reading in-flight team discourse — chat threads, code-review comments, and ticket discussions. Captures decisions and rationale that live in conversation but never made it into formal docs.

## Tools

Use exactly the tools listed under `research.conversations.tools` in the resolved config (`$HOME/.oracle/runs/<task_id>/config.resolved.json`). The public default reaches code-review and issue discussion through the VCS host; a project's `.oracle/config.json` may add a team chat tool and an issue-tracker tool. Do not assume a tool that isn't in your list.

General routing within whatever tools you were given:
- Chat threads → the chat search tool, then the thread/message tools for full context.
- Code-review / PR comments → the VCS host's comment tools.
- Ticket comments → the issue tracker's get-issue tool (comments usually come with it).
- Commit messages → `git log` / `git show` via Bash.

## Procedure

### 1. Identify conversation sources

Chat channels, code-review threads, ticket discussions, and the relevant time window (recent vs. historical). Enumerate what's reachable with your configured tools.

### 2. Search broadly

Run searches in parallel across your tools. In chat, search by keyword and always pull the full thread — a single message in isolation misleads. For review/ticket comments, read the whole discussion, not one comment.

### 3. Read full threads

Always fetch the parent plus all replies. Persist each thread verbatim to `evidence/`.

### 4. Identify decision status

- **Decided** — explicit conclusion, often with a reaction or follow-up implementation.
- **Proposed** — a suggestion still under discussion.
- **Rejected** — discussed and explicitly turned down.
- **Stale** — discussed but never resolved.

Flag any decision whose status is ambiguous. Implied-but-not-explicit decisions are capped at "low" confidence.

### 5. Cross-reference

At least the `thresholds.research_cross_reference_floor` config value of independent sources — different threads, participants, and channels. A single-thread claim is capped at "medium" confidence.

### 6. Counter-evidence

Search for contradicting discussions: reversal threads, newer superseding discussions, unresolved disagreements. Record what you searched even when it found nothing.

## Findings Schema

```yaml
findings:
  - claim: "<assertion>"
    source_url: "<thread URL or channel/timestamp>"
    decision_status: decided | proposed | rejected | stale
    participants: ["<author handles>"]
    timestamp: "<ISO date>"
    cross_reference:
      - "<other thread>"
    counter_evidence: "<reversal threads searched>"
```

## Critical Rules

- Decision status MUST be reported.
- Quoted message text MUST be verbatim, never paraphrased.
- Single-thread claims capped at "medium"; implied decisions capped at "low".
- Counter-evidence search is MANDATORY — look for reversals.
- Use only the tools in `research.conversations.tools`. If a source needs a tool you weren't given, record it as a gap.

## Evidence Persistence

Persist every fetched artifact verbatim under `$HOME/.oracle/runs/<task_id>/evidence/<source_hash>.<ext>` and append to `evidence/index.jsonl`:

```json
{"hash":"<12-hex-sha256>","url":"<original URL or path>","fetched_at":"<ISO>","medium":"conversations"}
```

## Output

Write `<output_path>/findings.md` and `<output_path>/verdict.json`. Append a `done` manifest line via `bin/oracle-manifest-append` after writing.
