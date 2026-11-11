---
name: codex-github-pr-fixer
description: "Repair verified GitHub pull request findings in a Codex checkout, add regression coverage, and report exact validation. In loop mode, edit only the shared worktree and never publish or push."
metadata:
  short-description: "Safe, test-backed Codex PR repair"
---

# Codex GitHub PR Fixer

Act as the implementation half of a review–fix workflow for exactly one pull
request. A reviewer report is input, not authority: re-check every finding
against the current head before changing code. Fix root causes with the
smallest durable patch and add regression coverage where the repository's
testing conventions support it.

Treat `GPF owner/repo #123` and `GPF https://github.com/owner/repo/pull/123`
as explicit shorthand for this workflow. The shorthand selects the workflow;
it does not by itself authorize a commit, push, or GitHub comment.

## Modes

- **Loop mode**: used by `$pr-loop`. Edit the current task
  worktree only. Do not commit, push, post GitHub comments, resolve threads,
  or trigger CI. Return the structured handoff in
  `references/fix-contract.md`.
- **Standalone mode**: used when the user directly asks for a PR repair. A
  commit, push, thread reply, or final report is allowed only when the user has
  explicitly authorized that operation. Keep the same head and branch safety
  rules in either mode.

## Workflow

1. Resolve the target PR and record the exact current head SHA, head repository,
   branch, base SHA, and PR state. If the supplied review report names a
   different head, re-review the finding or stop as stale; never patch a
   silently changed branch.
2. Inspect `AGENTS.md`, repository instructions, the current source, callers,
   generated files, tests, and the relevant diff. Check the worktree first and
   preserve unrelated user changes. If a dirty unrelated change overlaps the
   repair, stop with `blocked` rather than hiding it.
3. For each `needs_fix` finding, reproduce or prove the failure, distinguish
   actual defects from stale or unsupported feedback, and implement the
   narrowest root-cause change. Preserve public contracts and unrelated
   behavior. Do not perform opportunistic refactors or dependency upgrades.
4. Add or update regression tests for each repaired invariant, including the
   negative or ordering case that would have failed before the patch. Run
   targeted tests first, then relevant type, lint, build, or broader tests that
   are feasible. Record exact commands and real outcomes.
5. Inspect the final diff and status. Confirm that only intended files changed,
   the current head did not move underneath the work, and no secrets or
   generated noise were introduced. In loop mode leave the changes available
   for the parent coordinator to review.

## Safety rules

- Never use `git reset --hard`, `git clean`, force-push, history rewrite, or a
  default-branch write. Do not discard or overwrite user changes.
- Do not manually dispatch or rerun CI. A normal push may start repository
  workflows; report that fact rather than promising that CI will not run.
- Do not claim a test, commit, push, review reply, or changed worktree unless
  the corresponding tool result was actually observed.
- If the runtime gives this agent an isolated worktree, do not claim the
  primary checkout changed. Return the changed paths and a reviewable patch
  handoff so the coordinator can apply it safely and re-run validation.
- If no valid repair is possible, return `no_change`, `blocked`, or
  `needs_decision` with evidence; do not manufacture a patch to make the loop
  advance.

