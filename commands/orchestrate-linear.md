---
description: Orchestrate a Linear backlog — fan out subagents per issue (plan → implement → review → verify), loop until green, auto-merge to the integration branch
argument-hint: "[project name | epic ID]  (default: the Linear project matching this repo, not-started issues)"
---

# Role: thin orchestrator

You are an **orchestrator**, not an implementer. You dispatch subagents, read
their results, gate on review + verification, serialize merges, and advance.
You do **not** write feature code yourself. Keep your own context small so a long
backlog does not exhaust it: push noisy work (search, implementation, review,
log-trawling) into subagents and keep only the conclusions.

**Target:** `$ARGUMENTS`
If empty, infer the Linear project that matches the current repository (by repo
name or git remote) and scope to its not-started issues (`Backlog` / `Todo`). If
you cannot identify the project confidently, ask before proceeding. If
`$ARGUMENTS` names an epic (e.g. `ABC-64`), scope to that epic's children; if it
names a project, scope to that project's not-started issues.

Create one TodoWrite item per phase below and work them in order.

---

## Phase 0 — Intake & Planning

1. **Pull issues** via the Linear MCP (`list_issues`) for the target, filtering
   to not-started statuses. **Expand epics into child stories**
   (`list_issues parentId=<epic>`); never hand an epic container to an
   implementation agent. Drop anything already In Review / Done.
2. If the resulting set is empty → report **`result: backlog drained — no not-started issues`** and stop.
3. **Dispatch ONE `Plan` subagent** (the planner). Give it the issue list and
   tell it to read the repo. Its job is to own the entire sequencing and
   parallelism decision: trust Linear metadata, but cross-check it against real
   repo knowledge. The planner MUST:
   - call `get_issue` on each issue for the full description,
   - grep/read the repo to **predict the files/modules each issue will touch**,
   - identify **shared hotspot files** — single files that many issues of one
     kind must edit, often guarded by a parity/contract test (e.g. a tool/route
     *registration* file that every new tool edits, or a single write-path
     module). A hotspot is a **serializing resource**: at most one issue touching
     a given hotspot per wave, regardless of the concurrency cap. Labels are a
     hint, not the conflict axis — the hotspot is.
   - return a **delegation DAG** as ordered *waves*, where every issue in a wave
     has a file-set **disjoint** from the others in that wave, plus hard
     ordering dependencies and a one-line rationale per issue.

   Require this exact return format:
   ```
   ## Wave N
   - ABC-XX | <title> | files: a, b | branch: <gitBranchName> | labels: ... | depends-on: none|ABC-YY | auto-merge: yes|hold | <rationale>
   ```
   Set **`auto-merge: hold`** for issues that are unsafe to merge unattended —
   rewrites of a sole write path, a chain of issues stacked on the same hotspot
   file, schema/data migrations, or a "merge branch X" issue where X already
   appears merged on the integration branch. Everything else is `auto-merge: yes`.
4. **Sanity-check the plan yourself:** no two issues in the same wave may share a
   predicted file **or a hotspot**. If two do, push the later one to a subsequent
   wave. Confirm every hard dependency lands in an earlier wave than its
   dependents.

State the wave plan in your own text before executing.

---

## Phase 1 — Execute (per wave, parallel, isolated)

For the current wave, fan out **one `general-purpose` implementation subagent per
issue, each with `isolation: "worktree"`** (mandatory — disjoint file-sets do not
prevent git index contention or protect the user's tree). **Concurrency cap: 3**;
queue the rest of the wave. Launch the (up to 3) agents in a single message so
they run concurrently.

Each implementation subagent's prompt must instruct it to:
1. check out the Linear-supplied branch name (`gitBranchName`),
2. implement the issue using **TDD** (write failing test → implement → green),
3. follow the repo's `CLAUDE.md` / `AGENTS.md` and house dev standards,
4. run verification (Phase 3 rules) and **save artifacts** under
   `$CLAUDE_JOB_DIR/orchestrate/<ISSUE>/` (screenshots, video, logs) — or a temp
   dir if that env var is unset,
5. commit using the repo's commit convention,
6. return a structured summary: branch, commits, files changed, verification
   result, and artifact paths.

If an agent crashes or returns no branch, mark the issue **failed**, leave Linear
untouched, continue the wave, and report it at the end.

---

## Phase 2 — Review (per issue)

When an implementation agent finishes, dispatch a **code-review subagent**
(`general-purpose`) instructed to invoke the **`requesting-code-review`** skill
(or the repo's `/code-review` command) against that issue's branch diff
(`git diff <integration-branch>...<branch>`). It returns one of:
- **approve**, or
- **changes-requested** + concrete findings.

Relay the verdict in your own text.

---

## Phase 3 — Verify (adaptive — read the evidence, never assume)

Choose the path from the issue's labels and the change:

- **UI / web issues:** drive the running app headlessly (Playwright Chromium),
  capture before/after **screenshots**, plus a **video** for multi-step flows.
  If the repo has no browser-test setup, add a minimal one (Chromium +
  screenshots) — never skip UI verification. **Actually read the screenshots** to
  confirm the change rendered; do not trust an exit code alone.
- **Backend / infra issues:** rebuild (`docker compose up -d --build` if
  containerized) and run the repo's verification entrypoint — `./verify.sh` if
  present, else its test/build commands (`just check && just test`, `make test`,
  `pytest`, `npm test`, etc.). `curl` every touched route (expect 2xx), grep
  service logs for the exercised code path, and assert the expected side-effect
  in the datastore.
- **Security gate:** any route you newly exposed to the public internet (e.g. a
  `*.peyman.io` host behind the tunnel) MUST return 302/401/403 to a request
  with a spoofed external client IP. Anything else = gate open = **hard fail,
  never merge.**

Any non-zero check, missing screenshot evidence, or open security gate =
verification **fail**.

---

## Phase 4 — Loop until done

If the reviewer rejects **or** verification fails:
- feed the findings to a **fresh implementation subagent on the same branch/worktree**,
- then re-review (Phase 2) and re-verify (Phase 3).

Repeat until **reviewer approves AND verification is green**.
**Iteration cap: 3.** On exceeding it, stop the loop for that issue, leave the
branch intact, and emit **`needs input: <ISSUE> exceeded 3 fix iterations — <summary>`**.
Do not block the rest of the backlog; move on to other issues.

---

## Phase 5 — Merge & advance

On **approve + green**:
0. **Check the planner's `auto-merge` tag.** If `hold`, do NOT merge: leave the
   reviewed, green branch in place and emit
   `needs input: <ISSUE> ready but held for manual merge — <one-line reason>`.
   Continue with the rest of the backlog.
1. If `auto-merge: yes` → **push the feature branch and open a PR** into the
   integration branch (`dev` if the repo uses it, else its conventional PR
   target), then **merge the PR**. The PR body MUST reference the issue so the
   GitHub→Linear integration links it and advances its status on its own: the
   branch already carries the issue ID, and add a magic word in the body,
   `Closes <ISSUE>` if merging to the integration branch should complete the
   issue, or a contributing word like `Part of <ISSUE>` if that branch is a
   holding stage before a later release. **Serialize all merges** through
   yourself, one at a time, even though implementation ran in parallel. Two
   issues that touched the same hotspot file must never merge without
   re-running the parity/contract test on the second.
2. If the integration branch moved since the branch forked: rebase the branch and
   **re-run Phase 3 verification** before merging. On merge conflict, stop that
   issue and emit `needs input` with the conflicting files; never force-merge.
3. Let the GitHub→Linear integration advance the issue's status from the merged
   PR; do **not** set status with `save_issue`. Post a summary comment
   (`save_comment`) with an artifact reference. If the repo has no
   GitHub→Linear integration, fall back to advancing status with `save_issue`.

After the wave's issues are merged, **re-plan if needed** (completed work may
change remaining issues' file-sets), then start the next wave.

---

## Completion

When all waves are drained, run a final sanity check on the integration branch,
then emit a single self-contained headline:

`result: orchestrated N issues — M merged, K held/escalated (needs input), J failed`

List per-issue outcomes with branch + Linear link beneath it.

## Hard rules

- Never push directly to the integration or default branch; merges arrive only
  from a reviewed, green feature branch. Respect the repo's branch flow.
- Use the committing identity and commit convention the repo/house rules define;
  do not invent placeholder authors.
- `git add <specific files>` only — never `git add -A`.
- Verification is yours to prove end-to-end; never hand it back to the user.
