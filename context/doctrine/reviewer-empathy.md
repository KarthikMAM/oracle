# Doctrine: Reviewer Empathy

Predict what a human reviewer will flag, then pre-address each concern in the artifact itself — before it ships. Read and applied by oracle-engineer (pre-write), oracle-architect (pre-publish), and oracle-dev (when authoring the PR/CR description).

## Principle

A CR that surprises a reviewer gets pushed back. A CR that respects the reviewer's finite attention and pre-answers their questions gets approved. The goal is to make review effortless by demonstrating that every dimension the reviewer cares about was already understood.

## The Reviewer's Mental Model

When a senior engineer opens a CR, they ask — in this order:

1. **What is this trying to do?** — answered by a clear description and the first line of the commit message.
2. **Is the approach correct?** — answered by design rationale, test evidence, and alternatives considered.
3. **Does it match our patterns?** — answered by following existing conventions and citing precedent.
4. **What could break?** — answered by blast-radius analysis, edge-case handling, and a rollback plan.
5. **Is it complete?** — answered by all callers updated, tests covering new paths, and docs updated.
6. **Is it minimal?** — answered by no unrelated changes and no premature abstraction.

If any of these triggers uncertainty, the reviewer requests changes. If all are pre-answered, the reviewer approves.

## Prediction Protocol

1. **Identify likely reviewers.** From the git history of the modified files: primary (most recent significant contributor), secondary (CODEOWNERS), tertiary (the last few CR commenters).
2. **Predict concerns.** For each reviewer, predict what they would flag across correctness, style, architecture, performance, security, and testing.
3. **Pre-address each concern in the artifact.** Edge case → handle it and write a test. Pattern mismatch → follow the existing pattern and cite precedent. Missing rationale → add a WHY comment. Test gap → write the test before it's asked for.
4. **Self-test.** Walk the diff as the reviewer would: read the description cold, check each file, verify the tests demonstrate correctness, and look for loose ends.

## Cognitive Load Budget

A reviewer has roughly 20 minutes of focused attention per CR. Every element that consumes attention without adding value steals from their ability to find real issues. Keep a CR reviewable in under 10 minutes.
