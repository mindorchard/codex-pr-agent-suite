---
name: codex-pr-review-loop
description: "Coordinate a bounded Codex review–fix–re-review loop for one GitHub pull request, using a reviewer and fixer agent with explicit handoff contracts until findings are resolved or a blocker is reached."
metadata:
  short-description: "Coordinate PR review and repair agents"
---

# Codex PR Review–Fix Coordinator

This is a foreground coordinator for one pull request. It owns the loop state,
keeps the reviewer and fixer on the same current head, and decides when the
work is solved, blocked, or no longer making progress. It is not background
execution and must not promise work after the current Codex turn.

The reusable implementation is a skill because skills persist and are
discoverable. The reviewer and fixer are runtime subagents when the Codex
multi-agent runtime is available. A markdown file cannot create a permanent
agent identity: agent IDs and their contexts exist only for the current run.

## Invocation

Use `$codex-pr-review-loop` with `owner/repo #number` or a PR URL. Optional
parameters may be stated in the request:

- `max_rounds`: maximum review/fix cycles; default `5`.
- `publish`: `false` by default. Only an explicit user request can enable a
  commit, push, GitHub review, thread reply, or final PR comment.
- `scope`: paths or findings to include when the user narrows the task.

“Until solved” means repeat while the current head has actionable, repairable
findings and progress is being made. It never means an unbounded loop.

## Coordinator protocol

Read `references/loop-protocol.md` for the message templates and state shape.
Follow this sequence:

1. **Preflight in the parent context.** Resolve the PR, current head and head
   branch, repository instructions, worktree status, permissions, and the
   requested publication mode. Do not start agents against an ambiguous PR or
   a checkout containing overlapping unowned changes.
2. **Start the reviewer.** When the Codex runtime provides the multi-agent
   primitives, create one reviewer subagent and attach
   `$codex-github-pr-reviewer` as a skill item when the API supports structured
   skill items; otherwise include the exact skill name in the initial prompt.
   Pass the complete target plus the exact current head. Start it with a fresh
   context, not an old summary. If the runtime is unavailable, perform the
   reviewer role in the parent context and label the run as single-agent.
3. **Validate the review handoff.** Wait for the report, check its schema and
   `head_sha`, and discard stale or malformed conclusions. Treat only
   `needs_fix` findings as repair work. Stop for `blocked` or `needs_decision`.
4. **Start or reuse the fixer.** Pass the validated report to one fixer
   subagent and attach `$codex-github-pr-fixer` as a skill item when possible;
   run it in loop mode. The fixer may edit the primary worktree only if the
   runtime exposes it as shared. Never allow two agents to edit the same branch
   concurrently. If it has an isolated worktree, require a patch handoff and
   apply/review it in the parent before continuing.
5. **Verify the change in the parent.** Re-read `git status`, the changed
   files, the exact head, and the reported tests. Run any required parent-side
   checks. Do not infer that a test passed from the agent's intention or from a
   simulated result. If another process moved the head or unrelated changes
   appeared, stop and report the race.
6. **Round-trip.** Send the fixer result and the refreshed head to the reviewer
   and request a fresh review of the current state. Do not ask it to rubber-
   stamp its previous answer. Repeat steps 3–6 until a stop condition is met.
7. **Publish only if authorized.** At convergence, refresh the PR head one last
   time. If `publish=false`, leave local changes uncommitted/unpublished and
   report them accurately. If `publish=true`, inspect branch/workflow safety,
   make one cohesive commit when appropriate, update only the verified PR head
   with a non-forced compare/lease check, then publish minimal replies/report.
   Re-read the resulting PR before claiming completion.

## Runtime agent rules

In the current Codex runtime, use the available multi-agent primitives (the
usual names are `multi_agent_v1__spawn_agent`, `multi_agent_v1__send_input`,
`multi_agent_v1__wait_agent`, and `multi_agent_v1__close_agent`). Discover the
actual schemas rather than fabricating a call. Reuse the same reviewer and
fixer IDs across rounds with `send_input`; do not create an uncontrolled agent
per round. Wait only when the next coordinator decision depends on the result.

The coordinator, not the two subagents, owns the shared state and the stop
decision. Keep the reviewer read-only and keep the fixer in loop mode. The two
agents communicate through the coordinator's structured reports, not through
unverified assumptions or hidden context.

## Stop conditions

Stop and report **solved** when the reviewer returns `clean` for the exact
current head and all required tests have the stated evidence.

Stop and report **blocked** for missing access, stale/moved head, overlapping
unowned changes, unavailable required tooling, failed required tests without a
safe repair, or an isolated-worktree patch that cannot be transferred safely.

Stop and report **needs decision** when the evidence supports multiple product
behaviors, a requested change expands scope, or publication requires an
authorization the user did not give.

Stop and report **no progress** when the fixer returns `no_change`, the same
finding and evidence recur on two consecutive rounds, or `max_rounds` is
reached. Include the exact remaining findings and what would unblock the next
round.

## Final handoff

Report the repository/PR, starting and final heads, rounds completed, each
finding's final status, files changed, tests actually run, publication status,
and remaining limits. Never claim that code was pushed or a GitHub comment was
posted without the corresponding confirmed result.

