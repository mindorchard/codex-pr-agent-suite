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
  "publish": false,
  "review": null,
  "fix": null,
  "progress_fingerprint": null,
  "decision": "running"
}
```

## Reviewer message

When creating the subagent, attach the named skill as a structured skill item if
the spawn API supports that input. Send a fresh message on every round. Include the exact target and current head,
the previous fix handoff, parent-side `git status`/diff observations, and this
instruction:

> Use `$codex-github-pr-reviewer` in loop mode. Review the current head from
> scratch, verify prior findings against current code, and return exactly one
> `codex-pr-review/v1` report. Do not edit, commit, push, resolve threads, or
> post GitHub comments. Mark uncertainty and tests not run explicitly.

## Fixer message

Send only after the reviewer report has been validated. Include the exact head,
the full review JSON, the parent-side worktree status, and this instruction:

> Use `$codex-github-pr-fixer` in loop mode. Revalidate each `needs_fix`
> finding against the exact current head, make the smallest safe repair in the
> worktree you actually control, add regression coverage where appropriate,
> and return exactly one `codex-pr-fix/v1` report. Do not commit, push, post
> comments, resolve threads, or trigger CI.

## Re-review message

After the fixer returns, the parent verifies the worktree and sends the
reviewer:

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

When `publish=true`, the parent—not a subagent—must perform the final safety
gate:

1. Refresh PR metadata and confirm the expected head and head repository.
2. Review the final diff and test evidence; inspect relevant workflows before
   pushing. Never force-push or write the default branch.
3. Prefer one cohesive commit and one non-forced branch update. If the head
   moved, stop and restart from the new evidence instead of overwriting it.
4. Reply once per relevant thread and publish one concise report only after
   tool results confirm success. Do not resolve threads unless explicitly
   authorized.

