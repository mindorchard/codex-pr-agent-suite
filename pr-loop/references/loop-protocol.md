# Review–fix loop protocol

The coordinator keeps one state object in its working context. It may also
write a temporary copy outside the repository when a long run needs recovery;
do not add loop bookkeeping to the user's PR unless requested.

```json
{
  "schema": "codex-pr-loop/v1",
  "target": {
    "repository": "owner/repo",
    "pr_number": 123,
    "head_sha": "...",
    "head_branch": "feature/example"
  },
  "round": 1,
  "max_rounds": 5,
  "publish": true,
  "communication": "github_threads",
  "threads": [],
  "review": null,
  "fix": null,
  "progress_fingerprint": null,
  "decision": "running"
}
```

## Reviewer message

When creating the subagent, attach the named skill as a structured skill item if
the spawn API supports that input. Send a fresh message on every round. Include
the exact target and current head, the previous fix handoff, parent-side `git
status`/diff observations, the publication mode, and any existing thread IDs.
In the default publish mode, use this instruction:

> Use `$codex-github-pr-reviewer` in loop mode. Review the current head from
> scratch and read the existing PR discussion. Verify prior findings against
> current code and return exactly one `codex-pr-review/v1` report. Publish one
> concise review with one thread per actionable finding, reusing an existing
> thread when it represents the same finding. Include every posted thread ID
> and URL in the report. Do not edit the checkout, commit, push, or resolve
> threads. Mark uncertainty and tests not run explicitly.

When `publish=false`, replace the publication instruction with: do not post
reviews or thread comments; return a draft report and mark discussion as not
published.

## Fixer message

Send only after the reviewer report and its thread metadata have been
validated. Include the exact head, the full review JSON, the parent-side
worktree status, the publication mode, and the mapped thread IDs. In the
default publish mode, use this instruction:

> Use `$codex-github-pr-fixer` in loop mode. Revalidate each `needs_fix`
> finding against the exact current head and its live PR thread. Reply in each
> relevant thread with a concise repair plan, make the smallest safe repair in
> the worktree you actually control, and add regression coverage where
> appropriate. Run the required tests, reply with the result, then commit and
> non-forcibly push one cohesive repair to the PR head branch when you control
> the shared worktree. Return exactly one `codex-pr-fix/v1` report with the
> commit, push, and thread-reply evidence. Never resolve threads, write the
> default branch, or trigger CI manually.

When `publish=false`, replace the publication instruction with: do not commit,
push, or post thread comments; leave the repair as a local worktree handoff.

## Re-review message

After the fixer returns, the parent verifies the worktree, refreshes the PR
metadata, confirms the pushed head (or safely completes the patch handoff), and
sends the reviewer:

> The fixer returned the following handoff: `<fix JSON>`. Parent verification
> is: `<status/diff/tests/head>`. Review the resulting exact head from scratch;
> do not assume the handoff is correct and do not repeat findings that are now
> demonstrably fixed.

## Progress fingerprint

For no-progress detection, normalize each remaining `needs_fix` finding as:

```text
finding.id | severity | path | title | current-head evidence
```

Hash or compare the normalized set together with the current head. A fixer
`no_change`, or the same normalized actionable set with no meaningful code/test
change on two successive rounds, ends the loop. A new head alone is not
progress if it came from an unrelated change.

## Publication gate

When `publish=true`, the coordinator must perform a safety gate before and
after the fixer publication:

1. Refresh PR metadata and confirm the expected head and head repository.
2. Review the final diff and test evidence; inspect relevant workflows before
   pushing. Never force-push or write the default branch.
3. Allow only one cohesive commit per repair round and one non-forced branch
   update. If the head moved, stop and restart from the new evidence instead of
   overwriting it.
4. Confirm the fixer's thread replies and push result from tool output. If the
   fixer used an isolated worktree, apply and publish its patch in the parent
   only after rechecking the expected head.
5. At convergence, publish one concise final report and any necessary thread
   replies only after tool results confirm success. Do not resolve threads
   unless explicitly authorized.

