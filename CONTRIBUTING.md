# Contributing to Oracle

Thanks for your interest. Oracle is a Claude Code plugin, so "the code" is mostly **prose** — agent prompts in `agents/`, reference material in `context/`, the `/oracle` command in `commands/` — plus two bash scripts in `bin/` that back the durable artifact bus. Contributions are held to the same bar the plugin itself enforces: clear, self-evident, and defended.

## What the repo is

| Path | What it holds |
|---|---|
| `agents/` | The ten agents (the `oracle-dev` orchestrator + nine specialists), one markdown file each. |
| `context/` | Reference the agents read at runtime — `principles.md`, `quality-gates.md`, `artifact-bus.md`, the `coding-standards/`, the review `lenses/`, and `doctrine/`. |
| `commands/oracle.md` | The `/oracle` slash-command entrypoint. |
| `bin/` | `oracle-allocate-output` and `oracle-manifest-append` — the only executable code. |
| `.claude-plugin/` | `plugin.json` (the manifest) and `marketplace.json` (the one-plugin marketplace). |

There are **no hooks** and **no MCP servers** — the design is deliberately pure prompt-driven for portability. Please keep it that way unless a change has a strong, discussed reason.

## Ground rules

- **Zero organization-internal knowledge in committed files.** The shipped plugin must stay employer-neutral: no internal tool names, internal URLs, or company-specific nomenclature in `agents/`, `context/`, `commands/`, `bin/`, or the manifests. Anything environment-specific belongs only in a project-local `.oracle/config.json` (gitignored). A leak here is a blocking issue.
- **Diagrams are mermaid.** No ASCII-art flowcharts (`context/communication.md` → "Diagrams"). A literal directory tree shown for layout is not a diagram and may stay plain text.
- **Author-facing docs read for a junior engineer.** The README, the command, and any prose a human reads should be followable by a capable engineer in their first week — gloss load-bearing terms on first use (`context/communication.md`).
- **Coding-standards examples must obey the standards.** A `RIGHT` example in `context/coding-standards/` is held to the rules it teaches (no throw-as-control-flow except the sanctioned cases, ≤1 nesting, logical-block grouping, honest DRY).
- **The bin scripts** stay POSIX-portable bash (`#!/usr/bin/env bash`, `set -euo pipefail`), with no dependency beyond coreutils + `python3` for JSON.

## Making a change

1. Fork and branch off `main`.
2. Make the change; keep each PR to one concept.
3. Validate locally:
   ```bash
   # JSON manifests parse
   python3 -c "import json; [json.load(open(f)) for f in ['.claude-plugin/plugin.json','.claude-plugin/marketplace.json','context/config.json']]"
   # bin scripts pass syntax + strict mode
   bash -n bin/oracle-allocate-output && bash -n bin/oracle-manifest-append
   # install from your clone and confirm discovery
   claude plugin marketplace add "$PWD" && claude plugin install oracle@oracle
   claude plugin details oracle@oracle   # expect: /oracle command + 10 agents, 0 hooks
   ```
4. If you changed an agent's role, the pipeline shape, or an artifact name, update the cross-references that mention it (`README.md`, `commands/oracle.md`, `context/artifact-bus.md`, and the other agents) so nothing dangles.
5. Open a PR with a description that answers *what* changed and *why* in the first three lines.

## Versioning

`version` is kept in lockstep across `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`. Bump both, and add a `CHANGELOG.md` entry, in the same PR. Oracle follows [Semantic Versioning](https://semver.org/).

## License

By contributing, you agree your contributions are licensed under the [MIT License](LICENSE).
