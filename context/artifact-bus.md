# Artifact Bus — Durable Disk State

Every sub-agent return is **both** a short reply to the orchestrator **and** a durable artifact on disk. The reply is advisory; disk is truth. The orchestrator reconstructs run state by reading disk, so a run survives context compaction, session interruption, and multi-hour execution.

## Why disk, not conversation

A large project may spawn 30–60 sub-agents over hours. Conversation context fills and compacts; sessions drop. Disk artifacts:

- survive context truncation and session restart (resume from last verdict),
- give parallel sub-agents a shared medium without shared memory,
- provide an audit trail — every claim traces to a cited artifact,
- make the run replayable and debuggable.

## Layout

```
$HOME/.oracle/runs/<task-id>/              # transient runtime state for one run
├── manifest.jsonl                         # append-only event ledger
├── config.resolved.json                   # merged config + resolved `auto` values (written by the provisioner)
├── evidence/                              # verbatim fetched artifacts, deduped by content hash
│   ├── <hash>.<ext>
│   └── index.jsonl                        # hash → original URL/path, timestamp, medium
├── pre-flight/
│   ├── provisioner-verdict.json           # setup gate (oracle-provisioner): PASS | ESCALATE (unresolvable build cmd / no review path)
│   ├── verdict.json                       # baseline-build gate (oracle-builder, mode=baseline): PASS (green) | ESCALATE (red baseline)
│   └── build.log                          # the baseline build/test output
├── intake/
│   ├── verdict.json                       # whole-phase rollup — authored by oracle-dev (no sub-agent writes it)
│   └── researcher-<n>/findings.md
├── design/
│   ├── verdict.json                       # the architect's design verdict (the one phase-rollup a sub-agent writes)
│   ├── judge-verdict.json                 # the orchestrator's own rationale on a consequential call (audited by target=verdict)
│   ├── reviewer-<n>/findings.md           # design-target audit
│   └── critic-<n>/{findings.md,verdict.json}   # design debate — BROKEN | SURVIVED
├── code/
│   ├── verdict.json                       # whole-phase rollup — authored by oracle-dev when all waves pass
│   ├── judge-verdict.json                 # the orchestrator's own rationale (audited by target=verdict)
│   ├── engineer-<n>/result.json
│   ├── engineer-<n>/verdict.json             # per-task verdict — carries the task id it satisfied
│   ├── builder-<n>/{result.json,verdict.json,build.log,test.log}   # authoritative build+test+coverage per wave
│   ├── reviewer-<n>/findings.md           # code-target audit
│   └── tester-<n>/{findings.md,verdict.json}   # code debate — BROKEN | SURVIVED
└── ship/
    ├── cr.json                            # the open CR's URL — written by oracle-dev the moment the CR opens; the durable re-attach handle across fix loops, backward routes, and resumes
    ├── verdict.json                       # phase rollup — authored by oracle-dev at terminal-green; carries the CR URL + check status
    └── poll-log.jsonl                     # the releaser's CI poll history

$HOME/.oracle/specs/<task-id>/             # durable spec artifacts (the "what" and "how")
├── intake.md                              # synthesized understanding + scope + open decisions
├── requirements.md                        # what must be true when done (EARS-style, testable)
├── design.md                              # architecture, alternatives, risks, rollback
├── tasks.md                               # execution DAG: waves, coherent-unit tasks, files, validation
└── direct-task.json                       # direct-mode only: the substitute one-task contract (verbatim instruction, files, operation, validation) so an interrupted "just do it" run resumes without reverting to the full pipeline
```

`$HOME/.oracle/` is **home-scoped — outside any worktree and never part of any commit or PR.** The PR/CR carries only the source diff, commit message, and description, all authored to stand alone.

## Task ID

An 8-character lowercase hex slug minted by `oracle-provisioner` via `openssl rand -hex 4`. It appears in every artifact path and manifest line for the run.

## Manifest

`manifest.jsonl` is append-only — one JSON object per line, one line per lifecycle event. Append via `bin/oracle-manifest-append`; never edit in place (correct by appending a new line).

```json
{"ts":"2026-06-13T20:00:00Z","phase":"intake","agent":"oracle-researcher","seq":1,"event":"start","medium":"code"}
{"ts":"2026-06-13T20:03:11Z","phase":"intake","agent":"oracle-researcher","seq":1,"event":"done","artifact":"/Users/me/.oracle/runs/ab12cd34/intake/researcher-1/findings.md"}
{"ts":"2026-06-13T20:30:00Z","phase":"code","agent":"oracle-engineer","seq":4,"task":"T7","event":"done","verdict":"PASS","artifact":".../code/engineer-4/verdict.json"}
{"ts":"2026-06-13T21:05:00Z","phase":"ship","agent":"oracle-releaser","event":"done","verdict":"PASS","artifact":".../ship/verdict.json"}
```

**Granularity fields (load-bearing for resume).** Every CODE event MUST carry the `task` id it belongs to. This lets a resume map a completed `result.json` back to a specific task and re-spawn only what's actually missing, rather than re-running a whole phase. A `done` line on a per-task artifact also carries that artifact's `verdict`.

**Existence invariant** (the script labels this `RITE` — read-integrity-through-existence): if a manifest line names an `artifact`, that file MUST already exist on disk when the line is appended. `bin/oracle-manifest-append` enforces it — it stats the path first and refuses a line whose artifact is missing. A reader can therefore trust any `artifact` path in the manifest without re-checking.

## Output allocation

To avoid filename collisions when the orchestrator spawns several sub-agents of the same kind in parallel, each output-emitting spawn is told its output directory, allocated via `bin/oracle-allocate-output <task-id> <phase> <agent>`. The allocator uses mkdir-as-lock (portable across macOS and Linux; `flock` is Linux-only) and returns an absolute, pre-claimed `…/<agent>-<seq>/` path. The lock is **self-healing**: the holder records `<pid> <acquire-epoch>` inside the lock dir. The hot path is a plain atomic `mkdir` spin (reclaim is deliberately kept out of it — reading lock metadata is not atomic with the dir's identity, so doing it every iteration would corrupt the lock under load). Only after the full spin ceiling elapses — which happens only when a holder was killed (SIGKILL/OOM/power-loss) and bypassed the release trap — does a contender attempt a single guarded reclaim: it reclaims iff the holder PID is dead, OR the acquire-epoch is older than a TTL (PID-reuse backstop), OR the pid file is absent and the dir mtime is older than the TTL (a holder killed in the window between `mkdir` and the pid write). A live or just-acquired lock is never reclaimed. This lets a crashed run's lock not poison the resumed run; the same reclaim applies to the manifest-append lock. The `<agent>` argument is canonicalized by stripping a leading `oracle-`, so the caller MAY pass the real agent name (`oracle-engineer`) or the short role (`engineer`) interchangeably — both resolve to the same `engineer-<seq>` series and the same per-role counter. Every reader in this document addresses these dirs by the **short role** (`engineer-<n>`, `builder-<n>`, `reviewer-<n>`, `critic-<n>`, `tester-<n>`, `researcher-<n>`), which is what the allocator emits.

## Verdict schema

Every phase writes one `verdict.json`. The orchestrator reads it to route.

```json
{
  "task_id": "ab12cd34",
  "phase": "pre-flight | intake | design | code | ship",
  "verdict": "PASS | NEEDS_REWORK | ESCALATE",
  "summary": "<1-2 sentence outcome>",
  "artifacts": ["/abs/path/one.md"],
  "route_to": "intake | design | code | ship | null",
  "next_step_hint": "<set when verdict != PASS>",
  "iterations": 2,
  "confidence": "high | medium | low"
}
```

`route_to` is the phase that owns the fix when `verdict != PASS` (a reviewer's root-cause classification). `null` on PASS.

## Evidence folder

`oracle-researcher` persists every fetched external artifact (web page, doc, remote file) verbatim to `evidence/<hash>.<ext>`, where `<hash>` is the first 12 hex of the content's SHA-256. It appends one line per artifact to `evidence/index.jsonl` mapping hash → original URL/path, fetch timestamp, and medium. Citations reference both the original URL and the local evidence path, so a claim stays verifiable even if the source URL later changes.

## Resumability

A run interrupted mid-flight resumes from the manifest at **task granularity**, not whole phases — so a crash in wave 3 of CODE does not re-do completed work. `oracle-dev` reads `runs/<task-id>/manifest.jsonl` and determines the active phase from the last events, then:

- **Before CODE / in a completed phase** (pre-flight, intake, design): find the last phase with a `done` event and a phase `verdict.json`; resume from the next phase. Partial work in these monolithic phases without a verdict is re-run.
- **In CODE:** reconstruct the wave DAG from `tasks.md`. A task is complete iff the manifest has a `done`+`PASS` line carrying its `task` id (backed by a `engineer-<seq>/verdict.json`). Recompute the ready wave from the *remaining* tasks and re-spawn only those. Completed tasks' files are left untouched. The phase `verdict.json` is written only after every task passes and the full-diff review is clean.
- **In SHIP:** read `ship/verdict.json` and its `poll-log.jsonl`. A `PASS` verdict means the CR is terminal-green and the run is done. Otherwise, if `ship/cr.json` exists the CR was already opened — re-attach by that URL and resume polling; the releaser never re-pushes or opens a duplicate. Because `cr.json` is written when the CR first opens (not at terminal-green), this re-attach holds even when SHIP routed a human comment back to DESIGN/INTAKE and the run looped forward again.

This works because per-task verdicts are written incrementally (not just one verdict per phase), and because every CODE manifest line carries its `task` field.

## Capability contract by role

No agent declares `tools:` in its frontmatter, so each inherits the full tool set available to the session. These boundaries are therefore enforced **by the constraints in each agent's prompt**, not by withheld capability — every agent observes the contract below as discipline. (Model is also inherited by default; the three purely-mechanical agents — `oracle-provisioner`, `oracle-builder`, `oracle-releaser` — pin `model: sonnet` in frontmatter, since scaffolding, building, and git mechanics need no Opus-level judgment. The judgment-bearing agents inherit the session model.)

| Capability | Observed by |
|---|---|
| Read any artifact | every agent |
| Write findings/specs under `$HOME/.oracle/**` | provisioner, researcher, architect, reviewer, critic, tester, engineer, builder, releaser (every spawned agent writes its own findings/verdict here — this is not a source edit) |
| Write source code in the worktree | oracle-engineer only (the sole source writer; reviewer/critic/tester/builder MUST NOT modify source) |
| Run `bin/*` plumbing (allocate-output, manifest-append) | every spawned agent that emits an artifact — provisioner, researcher, architect, reviewer, critic, tester, engineer, builder, releaser |
| Run *scoped* test/typecheck/lint on own files | oracle-engineer (never the full build) |
| Run any full build (baseline at pre-flight + integration after each wave) + full test suite + coverage | oracle-builder ONLY (the single build owner; never modifies source); critic/tester run read-only build/test to reproduce. The provisioner runs no build — it resolves the commands the builder runs. |
| Run git | oracle-releaser only |
| Spawn sub-agents | oracle-dev only (every leaf's prompt says "Do not spawn other agents") |
