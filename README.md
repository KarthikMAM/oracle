# Oracle

A principal-engineer pipeline for Claude Code, reduced to first principles: two files.

```
/oracle add retry with exponential backoff to the API client
```

Give it a task. It researches until it understands, writes that understanding down, and
stops — you approve it. It designs the change, writes the plan down, and stops — you
approve that too. Then it runs autonomously: implements to the standards, has its own
work attacked by fresh eyes, and hands you a green PR/CR you merge yourself.

## Why two files

The model already knows how to research, design, code, review, and ship. The tools it
runs on — subagents, workflows, files — already provide isolation, parallelism, journaled
resume, and durable state. The repo being changed already documents how to build, test,
and ship itself (AGENTS.md, ecosystem markers). Prescribing any of that again — phase
procedures, agent rosters, lens catalogs, attack checklists, config schemas — duplicates
what exists and constrains judgment that should be derived fresh from each task.

What the model cannot derive is what this plugin carries:

| File | What it carries |
|---|---|
| [commands/oracle.md](commands/oracle.md) | The values: evidence over memory, distrust of one's own work, grounded challenges only, the two user gates, the output discipline. |
| [standards.md](standards.md) | The code bar: boring explicit code, one responsibility, errors as values, facts-only comments, done means verified. |

Everything else is the model's judgment, exercised per task.

## What you can count on

- **Two gates, both before code.** You approve the understanding, then the plan. After
  that it runs to a green PR/CR without you — unless it hits a question evidence
  genuinely cannot settle, which it presents with both sides and a recommended default.
- **Adversarial by default.** Nothing advances on the author's word. Every artifact is
  attacked from independent contexts; only grounded hits — a failing test, a runnable
  reproduction, a cited rule — force change.
- **Nothing ships with process residue.** Code, comments, commit messages, and PR/CR
  text state facts about the change. No task IDs, no AI attribution, no pipeline traces.
- **Never merges, never publishes, never force-pushes.** The green draft PR/CR is the
  handoff; merging is yours.
- **"Just do it" is honored.** Say it and both gates are skipped; the scrutiny is not.

## Installation

```bash
claude plugin marketplace add KarthikMAM/oracle
claude plugin install oracle@oracle
```

Or from a local clone: `claude plugin marketplace add /path/to/oracle`.

## Customizing

- **Code bar:** edit `standards.md`, or rely on the target repo's own AGENTS.md — repo
  conventions outrank the shipped standards.
- **Research sources:** whatever MCP tools the session has are what it researches with.
  Add internal sources (code search, wikis, chat, tickets) by adding their MCP servers
  to your Claude Code configuration; no plugin config needed.

## License

MIT
