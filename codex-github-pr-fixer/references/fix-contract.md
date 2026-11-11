# Fix handoff contract

The fixer returns one JSON object in a fenced `json` block. The coordinator
uses the JSON to decide whether to re-review; it does not treat a successful
tool call alone as proof that the primary checkout contains the patch.

```json
{
  "schema": "codex-pr-fix/v1",
  "target": {
    "repository": "owner/repo",
    "pr_number": 123,
    "base_head_sha": "...",
    "observed_head_sha": "..."
  },
  "status": "changed",
  "changed_files": ["src/example.ts", "test/example.test.ts"],
  "summary": "Root cause and durable correction",
  "tests": [
    { "command": "npm test -- ...", "status": "passed", "evidence": "..." }
  ],
  "residual_findings": [],
  "handoff": {
    "applied_to_primary_workspace": true,
    "patch_path": null,
    "notes": "..."
  },
  "commit": null,
  "publish": { "pushed": false, "comments_posted": false }
}
```

Allowed values:

- `status`: `changed`, `no_change`, `blocked`, or `needs_decision`.
- Test `status`: `passed`, `failed`, `not_run`, or `blocked`.
- `handoff.applied_to_primary_workspace` must be `false` or `unknown` when
  the agent used an isolated worktree. The parent must inspect/apply the patch
  and validate it before another review round.

In loop mode, `commit` is `null`, `publish.pushed` is `false`, and
`publish.comments_posted` is `false`. A `changed` result means only that the
agent made a change in the workspace it controlled; the coordinator still
checks `git status`, the diff, the head, and the tests.

