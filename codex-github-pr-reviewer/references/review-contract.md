# Review handoff contract

The reviewer returns one JSON object in a fenced `json` block. The coordinator
may accept a human-readable explanation after it, but the JSON is authoritative.

```json
{
  "schema": "codex-pr-review/v1",
  "target": {
    "repository": "owner/repo",
    "pr_number": 123,
    "base_sha": "...",
    "head_sha": "...",
    "head_repository": "owner/repo",
    "head_branch": "feature/example"
  },
  "verdict": "actionable",
  "findings": [
    {
      "id": "F-001",
      "status": "needs_fix",
      "severity": "P1",
      "title": "Short problem statement",
      "path": "src/example.ts",
      "symbol_or_line": "functionName (line 42 if verified)",
      "trigger": "Concrete input, ordering, or environment",
      "impact": "Observable failure or invariant violation",
      "evidence": "Current-head code path, test, or reproduction",
      "fix_direction": "Smallest durable correction",
      "verification": "Test or invariant that should pass after repair"
    }
  ],
  "verification": [
    { "command": "npm test -- ...", "status": "passed", "evidence": "..." }
  ],
  "limits": []
}
```

Allowed values:

- `verdict`: `clean`, `actionable`, `blocked`, or `needs_decision`.
- Finding `status`: `needs_fix`, `already_fixed`, `not_reproducible`,
  `needs_decision`, or `blocked`.
- `severity`: `P0`, `P1`, `P2`, or `P3`. Use a severity only when the impact
  is supported by evidence.
- Verification `status`: `passed`, `failed`, `not_run`, or `blocked`.

The coordinator treats only `needs_fix` findings as repair work. A finding is
not actionable merely because it is interesting, stylistically preferable, or
mentioned by a reviewer. `head_sha` must be the exact commit reviewed.

