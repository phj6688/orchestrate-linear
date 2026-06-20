# Changelog

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
