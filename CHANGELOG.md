# Changelog

All notable changes to Oracle are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the project follows [Semantic Versioning](https://semver.org/).

## [1.0.0] — 2026-06-15

Initial public release: an autonomous principal-engineer coding pipeline for Claude Code that takes a one-line task through research, design, adversarial-reviewed implementation, and shipping — producing one merge-ready PR/CR for the workspace.

### Pipeline

- Five phases — **PRE-FLIGHT → INTAKE → DESIGN → CODE → SHIP** — driven by a single flat orchestrator (`oracle-dev`); only the orchestrator spawns, every other agent is a leaf.
- **One user checkpoint**, after DESIGN (`APPROVE` / `APPROVE WITH CHANGES` / `REVISE` / `RESTART`); autonomous to terminal-green thereafter.
- **Root-cause routing**: every finding — including a human CR comment at SHIP — routes to the phase that owns it (a code fix to CODE, a wrong approach to DESIGN, a wrong problem to INTAKE), bounded by per-loop budgets that ESCALATE rather than churn.

### Agents (10)

- `oracle-dev` (orchestrator), `oracle-provisioner`, `oracle-researcher`, `oracle-architect`, `oracle-engineer`, `oracle-builder`, `oracle-reviewer`, `oracle-critic`, `oracle-tester`, `oracle-releaser`.
- The engineer writes code and runs *scoped* checks; **`oracle-builder` owns every full build** (the pre-flight baseline gate and the post-wave full build + test + coverage).
- The three purely-mechanical agents (`provisioner`, `builder`, `releaser`) run on Sonnet; the judgment-bearing agents inherit the session model.

### Review

- Two complementary modes: **audit** (lens-driven breadth, one reviewer per lens, each loading only the standards its lens validates) and **debate** (goal-driven depth — `oracle-critic` attacks the design, `oracle-tester` breaks the code).
- The orchestrator audits its own gate decisions via a `verdict`-target reviewer; ungrounded findings are advisory and a passing state is never reversed on argument alone.

### Standards

- Seventeen opinionated coding standards, **relevance-loaded** (a core set always, the rest only when a change touches their surface), with the Go ethos — simplicity, errors-as-values, composition over inheritance — as the cross-cutting value.

### Durable artifact bus

- All state lives under `$HOME/.oracle/` (specs, runtime verdicts, evidence, append-only manifest); a run **resumes from the last verdict** and the artifact bus survives context compaction and session interruption. Backed by two portable bash scripts with a self-healing lock.

### Configuration

- Ships with **zero organization-internal knowledge**: the committed default is entirely public (GitHub `gh`, generic build/test detection, public web research). A project-local `.oracle/config.json` deep-merges over it to wire internal tools, a non-GitHub review path, and tighter bars — without ever touching the published surface.

[1.0.0]: https://github.com/KarthikMAM/oracle/releases/tag/v1.0.0
