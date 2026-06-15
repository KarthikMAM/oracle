# Security Policy

## Reporting a vulnerability

If you find a security issue in Oracle, please report it privately rather than opening a public issue:

- Use GitHub's **["Report a vulnerability"](https://github.com/KarthikMAM/oracle/security/advisories/new)** (Security → Advisories) on the repository, or
- open a minimal private channel with the maintainer before any public disclosure.

Please include what the issue is, how to reproduce it, and the impact you see. We aim to acknowledge a report within a few days and will coordinate a fix and disclosure timeline with you.

## What is in scope

Oracle is a prompt-driven Claude Code plugin. The security-relevant surface is small and worth stating precisely:

- **The two bash scripts in `bin/`** (`oracle-allocate-output`, `oracle-manifest-append`) — the only executable code. They run on the developer's machine under their own credentials and write only under `$HOME/.oracle/`. A path-traversal, injection, or privilege issue here is in scope.
- **The shipped configuration** (`context/config.json`) and the prompts' handling of project-local `.oracle/config.json` — e.g. a way for committed defaults to execute unintended commands.
- **Provenance / data-exfiltration guarantees** the prompts make — e.g. a path by which the pipeline would commit secrets, leak `$HOME/.oracle/` contents into a PR, or publish a review without the human's action.

## What is not in scope

- The behavior of Claude or Claude Code themselves — report those to Anthropic.
- A project's *own* `.oracle/config.json` choosing to run dangerous commands — Oracle runs the commands a project configures; vetting them is the project's responsibility. The shipped defaults are public and benign.
- Findings that require the attacker to already control the developer's machine or their Claude Code session.

## Design properties that back this up

- The shipped plugin carries **zero organization-internal knowledge**; anything environment-specific lives only in a gitignored project-local config.
- Agents **never auto-merge and never auto-publish** a review — shipping ends at a draft a human acts on.
- Commit and CR text carry **no pipeline provenance** and the standards forbid committing secrets; the `security` review lens and `data-integrity` standard enforce secret-handling, fail-closed defaults, and trust-boundary parsing on the code Oracle writes.
