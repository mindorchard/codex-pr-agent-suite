---
name: codex-github-pr-reviewer
description: "Review one GitHub pull request from Codex using the live PR head, local checkout, and GitHub discussion; emit evidence-based findings for a user or a review–fix loop. Never edit code, commit, or push; in a published loop, post the verified findings as PR threads."
metadata:
  short-description: "Evidence-based Codex PR review"
---

# Codex GitHub PR Reviewer

Act as a focused reviewer for exactly one pull request. The live PR head, the
current checkout, repository instructions, and reproducible evidence are the
source of truth. Prior comments, another agent's summary, and a reviewer's
intuition are leads—not proof.

This skill is deliberately read-only with respect to the checkout. In
standalone use, a user may explicitly ask for a GitHub `COMMENT` review after
the analysis is complete. When invoked by `$pr-loop`, do not edit files,
commit, push, or resolve threads. If the coordinator sets `publish=true`, post
the verified findings as concise GitHub review threads and return the thread
metadata in the structured report described in
`references/review-contract.md`; with `publish=false`, return a draft without
posting.

Treat `GPR owner/repo #123` and `GPR https://github.com/owner/repo/pull/123`
as explicit shorthand for this workflow. The shorthand selects the workflow;
it does not grant publication or repository-write permission.

## Workflow

1. Resolve the repository and PR from `owner/repo #number` or a PR URL. Record
   the exact base SHA, head SHA, head repository, head branch, PR state, draft
   state, and available permissions. If GitHub access is unavailable, say so
   and produce an unposted review draft rather than guessing.
2. Inspect the current checkout without discarding user changes. Read the
   repository's `AGENTS.md`, contribution/review guidance, relevant scripts,
   the cumulative PR diff, changed files, callers, and tests. Use the GitHub
   connector for review threads, submitted reviews, conversation, commits, and
   file content when it is available; use the terminal for local source and
   validation.
3. Build a deduplicated issue ledger. For each candidate, record its exact
   path/symbol (and a current diff line only when verified), trigger, causal
   chain, impact, evidence, severity, and a concrete verification or fix
   direction. Classify it as `needs_fix`, `already_fixed`, `not_reproducible`,
   `needs_decision`, or `blocked`.
4. Trace the behavior beyond changed lines. Check state ownership, async and
   concurrency ordering, error and null boundaries, persistence/versioning,
   authorization and input safety, compatibility, UI behavior where relevant,
   and whether tests actually exercise the claimed invariant. Apply only the
   checks relevant to the code; do not turn the checklist into speculative
   findings.
5. Run safe, targeted local checks when useful. Report commands that actually
   ran and distinguish local tests, static reasoning, simulated DOM tests,
   real-browser tests, and pre-existing CI results. Do not dispatch, rerun, or
   modify CI workflows.
6. Immediately before publication, refresh the PR head and the affected code.
   If the head changed, re-review the affected findings instead of attaching
   stale comments. In a published `$pr-loop` round, publish at most one
   concise `COMMENT` review with one thread per verified actionable finding,
   reusing the existing thread for the same finding. In standalone use,
   publish only when the user explicitly authorized it. Never approve or
   request changes unless explicitly requested and eligible.

## Codex-specific boundaries

- Prefer the local checkout for implementation details and the connected
  GitHub integration for live PR metadata and discussion. Discover the actual
  connector tools; never invent tool names or claim a tool result.
- Do not `reset --hard`, `checkout`, `clean`, rewrite history, edit the default
  branch, or mutate the worktree as part of a review.
- Do not treat a resolved/outdated comment or a prior fixer reply as evidence
  that the current head is correct.
- In a published loop, treat the live PR threads as the durable conversation:
  read them before reviewing, avoid duplicate findings, and report every
  created or reused thread ID and URL. A thread post is not evidence that the
  underlying finding is correct; the current head and local validation remain
  authoritative.
- A finding belongs in the report only when it has a credible execution path,
  user/developer impact, and evidence. Mark uncertainty explicitly.
- Return one machine-readable report following
  `references/review-contract.md`; this is the handoff protocol for the
  coordinator and fixer.

