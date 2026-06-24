# Changelog

## 1.1.4

- Wire Phase 5 to the sha-pinned merge gate (`block-unverified-merge.sh`,
  claude-homelab): for a gated homelab project, run `~/.claude/gate/verify.sh`
  from the issue's worktree immediately before merging and merge from that same
  worktree, so the gate's `verified_sha == HEAD` check passes instead of
  deadlocking the loop's own merge. (HLB-488)
- Phase 3 security gate now probes the public route with a plain anonymous
  request, not a spoofed `CF-Connecting-IP`. A forged client IP only trips
  Cloudflare's WAF and masks whether the real auth gate holds. (HLB-488)
- Phase 1 clears the per-issue probe staging dir before generating, so a retried
  or re-invoked issue never runs a stale held-out probe left in the `/tmp`
  fallback by an earlier run.

## 1.1.3

- Per-issue held-out behavioural probe. Before each issue is implemented, a
  write-only `heldout-probe` subagent generates a black-box pytest from the issue
  spec alone, stored outside the implementer's worktree so the builder never sees
  it (Phase 1). Phase 3 runs it against the built code as a mandatory, un-gameable
  gate; a non-zero exit fails verification. Phase 5 commits the probe into
  `tests/heldout/` after green, where the `block-heldout-write` hook freezes it.
  Requires the global `heldout-probe` agent and hook (claude-homelab). (HLB-487)

## 1.1.2

- Phase 2 review now dispatches a read-only `code-reviewer` subagent (Read/Grep/Glob/Bash,
  no Write/Edit/MultiEdit) instead of `general-purpose`, so the reviewer cannot edit the
  code it grades. Requires the global `code-reviewer` agent (claude-homelab `~/.claude/agents/`). (HLB-486)

## 1.1.1

- Orchestrator now independently verifies a commit landed on each issue's branch
  (`git rev-list --count <integration-branch>..<branch>`) instead of trusting the
  subagent's self-report. A branch with no new commits is marked **failed** even
  when the summary claims success — closing the silent failure where an agent
  edits, verifies, and reports done but never commits.
- Implementation subagents must return the head commit SHA in their summary.

## 1.1.0

- Baseline: planner + per-issue implement/review/verify subagents, loop-until-green,
  serialized auto-merge to the integration branch.
