# Doctrine: Diff Presentation

How the final PR/CR is presented for maximum approval velocity — commit structure, description organization, diff minimality, and visual self-evidence. Read and applied by oracle-dev, which authors the commit message and PR/CR description at SHIP.

## Principle

A reviewer's first impression forms in the first 10 seconds, and it decides whether they enter with trust or suspicion. Presentation IS substance when it shapes the reviewer's cognitive frame.

## The Three-Second Test

A reviewer scanning the CR should answer in three seconds:

1. What changed? — the commit message's first line.
2. Why? — the first sentence of the description body.
3. How big? — the file count and line count.

## Commit Message Structure

```
<type>(<scope>): <imperative summary under 72 chars>

<body: 2-4 sentences explaining WHY, not WHAT>

Refs: <ticket-id>
```

Types: `fix`, `feat`, `refactor`, `perf`, `test`, `docs`, `build`, `chore`. The subject is imperative mood, ≤72 characters. The body explains WHY, not WHAT. No internal paths, agent names, or pipeline references.

## Diff Minimality

- One logical concern per diff.
- No whitespace or import changes mixed with logic.
- No unrelated drive-by fixes.
- Every file touched serves the stated goal.
- The smallest change that completely implements the goal.

## Visual Self-Evidence

| Strategy | Why it works |
|---|---|
| Symmetry in changes | Reviewers scan for symmetric patterns. |
| Consistent change pattern | Inconsistency triggers suspicion. |
| Test names as documentation | Tells the reviewer what's tested without reading the body. |
| Guard clauses first | Signals thoroughness. |
| Minimal scope expansion | Zero caller changes lets the reviewer breathe easy. |
