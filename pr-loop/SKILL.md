---
name: pr-loop
description: "Coordinate a bounded GitHub PR review–fix–re-review loop where independent reviewer and fixer agents communicate through PR threads, commit verified repairs, and push them until findings are resolved or a blocker is reached."
metadata:
  short-description: "Coordinate PR review and repair agents"
---

# Codex PR Review–Fix Coordinator

This is a foreground coordinator for one pull request. It owns the loop state,
keeps the reviewer and fixer on the same current head, and decides when the
work is solved, blocked, or no longer making progress. GitHub PR review
threads are the durable communication channel between the two independent
agents; structured JSON handoffs are control-plane evidence for the
coordinator, not a replacement for the discussion. It is not background
execution and must not promise work after the current Codex turn.

The reusable implementation is a skill because skills persist and are
discoverable. The reviewer and fixer are runtime subagents when the Codex
multi-agent runtime is available. A markdown file cannot create a permanent
agent identity: agent IDs and their contexts exist only for the current run.

## Invocation

Use `$pr-loop` with `owner/repo #number` or a PR URL. Optional
parameters may be stated in the request:

- `max_rounds`: maximum review/fix cycles; default `5`.
- `publish`: `true` by default. The reviewer posts findings as PR review
  threads, the fixer replies in those threads and commits/pushes verified
  repairs, and the coordinator publishes the final status. Set
  `publish=false` only for an explicitly requested dry run; in that mode no
  commit, push, review, thread reply, or final PR comment is allowed.
- `scope`: paths or findings to include when the user narrows the task.

“Until solved” means repeat while the current head has actionable, repairable
findings and progress is being made. It never means an unbounded loop.

## Coordinator protocol

Read `references/loop-protocol.md` for the message templates and state shape.
Follow this sequence:

1. **Preflight in the parent context.** Resolve the PR, current head and head
   branch, repository instructions, worktree status, permissions, and the
   requested publication mode. In the default publish mode, verify that the
   run can read and write PR discussions and push to the PR head branch. Do
   not start agents against an ambiguous PR or a checkout containing
   overlapping unowned changes.
2. **Start the reviewer.** When the Codex runtime provides the multi-agent
   primitives, create one reviewer subagent and attach
   `$codex-github-pr-reviewer` as a skill item when the API supports structured
   skill items; otherwise include the exact skill name in the initial prompt.
   Pass the complete target plus the exact current head and the publication
   mode. Start it with a fresh context, not an old summary. In publish mode,
   require the reviewer to inspect existing threads and post one concise
   thread per actionable finding, without editing, committing, or pushing. If
   the runtime is unavailable, perform the reviewer role in the parent
   context and label the run as single-agent.
3. **Validate the review handoff.** Wait for the report, check its schema,
   `head_sha`, and thread publication metadata, and discard stale or malformed
   conclusions. Treat only `needs_fix` findings as repair work. In publish
   mode, a missing required thread or discussion permission is a blocker.
   Stop for `blocked` or `needs_decision`.
4. **Start or reuse the fixer.** Pass the validated report to one fixer
   subagent and attach `$codex-github-pr-fixer` as a skill item when possible;
   run it in loop mode with the relevant thread IDs. In publish mode, the
   fixer reads the live threads, replies with its repair plan and result, and
   commits and pushes one cohesive repair before the next review. The fixer
   must use a non-forced head check and never write the default branch. Never
   allow two agents to edit or push the same branch concurrently. If the
   runtime gives it an isolated worktree, require a patch handoff and have the
   parent apply, validate, commit, and push it before continuing.
5. **Verify the change in the parent.** Re-read `git status`, the changed
   files, the exact PR head, the thread replies, and the reported tests. If the
   fixer did not push a changed result, the parent must complete the safe
   commit/push handoff before another GitHub re-review. Do not infer that a
   test, thread reply, commit, or push succeeded from the agent's intention or
   from a simulated result. If another process moved the head or unrelated
   changes appeared, stop and report the race.
6. **Round-trip.** Send the fixer result and the refreshed head to the reviewer
   and request a fresh review of the exact pushed current state. Tell it to
   update the existing finding threads, avoid duplicates, and create a new
   thread only for a newly verified finding. Do not ask it to rubber-stamp its
   previous answer. Repeat steps 3–6 until a stop condition is met.
7. **Close the loop.** At convergence, refresh the PR head one last time. In
   the default publish mode, ensure every changed file is committed and pushed
   to the verified PR head, then publish one concise final status and the
   necessary thread replies. Do not resolve threads unless the user explicitly
   asks for that. With `publish=false`, leave changes uncommitted and
   unpublished and report that dry-run status accurately. Re-read the
   resulting PR before claiming completion.

## Runtime agent rules

In the current Codex runtime, use the available multi-agent primitives (the
usual names are `multi_agent_v1__spawn_agent`, `multi_agent_v1__send_input`,
`multi_agent_v1__wait_agent`, and `multi_agent_v1__close_agent`). Discover the
actual schemas rather than fabricating a call. Reuse the same reviewer and
fixer IDs across rounds with `send_input`; do not create an uncontrolled agent
per round. Wait only when the next coordinator decision depends on the result.

The coordinator, not the two subagents, owns the shared state and the stop
decision. Keep the reviewer read-only with respect to the checkout, but allow
it to publish review threads when `publish=true`. Keep the fixer in loop mode;
it may edit, commit, push, and reply to the assigned threads only within the
validated PR branch and publication mode. The agents communicate through the
live PR threads plus machine-readable reports; neither hidden context nor an
unverified summary is sufficient.

## Stop conditions

Stop and report **solved** when the reviewer returns `clean` for the exact
current head and all required tests have the stated evidence.

Stop and report **blocked** for missing discussion or branch-write access in
publish mode, stale/moved head, overlapping unowned changes, unavailable
required tooling, failed required tests without a safe repair, a failed
non-forced push, or an isolated-worktree patch that cannot be transferred
safely.

Stop and report **needs decision** when the evidence supports multiple product
behaviors or a requested change expands scope. Honor an explicit user request
for `publish=false` as dry-run mode rather than treating it as a blocker.

Stop and report **no progress** when the fixer returns `no_change`, the same
finding and evidence recur on two consecutive rounds, or `max_rounds` is
reached. Include the exact remaining findings and what would unblock the next
round.

## Final handoff

Report the repository/PR, starting and final heads, rounds completed, each
finding's final status, files changed, tests actually run, publication status,
and remaining limits. Never claim that code was pushed or a GitHub comment was
posted without the corresponding confirmed result.

