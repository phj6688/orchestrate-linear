# orchestrate-linear

*Hand a whole Linear backlog to Claude Code and let it run, one issue at a time, until everything is green.*

## Install

In Claude Code, add the marketplace then install the plugin:

```
/plugin marketplace add phj6688/claude-marketplace
/plugin install orchestrate-linear@phj
```

## Use

In any repo that maps to a Linear project:

```
/orchestrate-linear
```

Or point it at a specific project or epic:

```
/orchestrate-linear ABC-64
/orchestrate-linear "Full-Text Search"
```

With no argument it infers the Linear project from the repo and scopes to the
not-started issues (`Backlog` / `Todo`).

## How it works

A thin orchestrator stays in charge and keeps its own context small. It pulls
the issues, expands epics into child stories, and for each issue fans out
subagents:

1. **plan** the change,
2. **implement** it on its own branch,
3. **review** the diff,
4. **verify** with the project's checks.

An issue only advances when the reviewer approves and the checks pass. Merges
are serialized to the integration branch, and the loop runs until the backlog
is clear. The orchestrator never writes feature code itself, so a long backlog
does not exhaust its context.

## License

MIT, see [LICENSE](LICENSE).
