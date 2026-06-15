# Configuration & Override Model

Oracle reads two config files and deep-merges them. The shipped plugin default carries **zero org-internal knowledge** so the plugin is safe to publish; everything organization-specific lives in a project-local file that is gitignored by convention.

## Resolution order

```
context/config.json          ← plugin default (committed, public, generic)
        ▼  deep-merge, project wins per key
<project-root>/.oracle/config.json   ← project override (gitignored, may carry internal tools)
        ▼  deep-merge, env wins
environment / pre-flight detection   ← build/test/branch detected from project markers
```

A key set in the project file replaces the same key in the default. Objects merge recursively; arrays replace wholesale (so an override can shrink a tool list, not just grow it). Any agent that needs config reads the project file first and falls back to the plugin default.

## What belongs where

| Setting | Plugin default (`context/config.json`) | Project override (`.oracle/config.json`) |
|---|---|---|
| Research tools | Public only: `Read`/`Grep`/`WebSearch`/`WebFetch`, public GitHub MCP | Internal code search, wiki, chat, ticketing MCP tools |
| Build/test/lint | `auto` (detected from markers) | Exact commands when detection is wrong or custom |
| `operations.pr.{create,status,ready,comment}` | GitHub via the public `gh` CLI | The org's own review-CLI commands — the commands ARE the review path, so there is no separate `vcs`/tool-name key. `create` SHOULD open a draft. |
| `operations.git.rebase` / `git.push` | `git rebase origin/${MAIN_BRANCH}` / `git push -u origin HEAD` | Override for a non-`origin` remote, `--rebase-merges`, a custom push refspec, etc. `${MAIN_BRANCH}` is interpolated to the resolved `git.main_branch` by the provisioner when it writes `config.resolved.json` (single owner of the substitution). The releaser runs `push` unless the `create` command pushes implicitly (e.g. `gh pr create`). |
| Thresholds | Sensible public defaults | Tighter/looser per team appetite |
| `off_limits_globs` | Protects the plugin's own internals | Add the team's generated/vendored paths |

## `auto` resolution

`auto` values are resolved once, at pre-flight, by `oracle-provisioner`, and written into the run's `config.resolved.json` so every later agent reads a concrete value instead of re-detecting.

| Key | Detection |
|---|---|
| `operations.build` | `package.json`→`npm run build` / `Cargo.toml`→`cargo build` / `go.mod`→`go build ./...` / `pyproject.toml`→project script. An ecosystem the shipped detection does not recognize is configured explicitly in `.oracle/config.json`. |
| `operations.test` / `lint` / `typecheck` / `coverage` / `format` | Same marker family, mapped to the ecosystem's conventional command |
| `operations.git.main_branch` | `git symbolic-ref refs/remotes/origin/HEAD`, fallback `main` |

## Customization recipes

**Point research at an internal code-search MCP** — in `.oracle/config.json`:
```json
{ "research": { "code": { "tools": ["Read","Grep","Glob","Bash","mcp__your-mcp__CodeSearch"] } } }
```

**Ship through a non-GitHub review tool** — replace the placeholders with your org's review CLI; these commands ARE the review path:
```json
{ "operations": { "pr": {
    "create": "<your-review-cli> create --draft",
    "status": "<your-review-cli> status --json",
    "ready":  "<your-review-cli> publish",
    "comment":"<your-review-cli> comment" } } }
```

**Protect generated code from edits:**
```json
{ "policy": { "off_limits_globs": ["agents/**","context/**","bin/**","commands/**",".oracle/**","**/generated/**","**/*.pb.go"] } }
```
