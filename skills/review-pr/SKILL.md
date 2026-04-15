---
name: review-pr
description: Review an existing PR — fetch comments, analyze impact, run security review, and address feedback with code changes. Use when given a PR URL or number to review and fix.
---

# Review PR

## Overview

End-to-end PR review and remediation: fetch PR state, analyze diff and comments, run impact + security review via subagents, then address actionable feedback with code changes.

**Core principle:** Understand everything before changing anything. Every edit must be proven safe.

**Announce at start:** "I'm using the review-pr skill to review and address feedback on PR #<number>."

## Context Management — CRITICAL

This skill touches large diffs, multiple review threads, and potentially many files. **Protect the main context aggressively:**

- The main thread is an **orchestrator only** — it parses inputs, dispatches analysis, makes decisions, and applies edits.
- **ALL exploration goes to subagents**: reading full file contents, tracing call chains, searching for usage patterns, analyzing security implications.
- Subagent results return **structured findings**, not raw code dumps.
- Run independent analysis subagents **in parallel**.

## The Process

```
┌──────────────────────────────────────────────────────────┐
│ 1. FETCH  (main context — lightweight)                   │
│    ├─ PR metadata (title, state, branch, mergeable)      │
│    ├─ Full diff                                          │
│    ├─ Review comments (inline + top-level)               │
│    └─ CI status                                          │
├──────────────────────────────────────────────────────────┤
│ 2. SETUP WORKSPACE                                       │
│    ├─ Find or create worktree for the PR branch          │
│    └─ Ensure dependencies installed                      │
├──────────────────────────────────────────────────────────┤
│ 3. ANALYZE  (parallel subagents)                         │
│    ├─ Subagent A: Deep code review + impact analysis     │
│    ├─ Subagent B: Security review                        │
│    └─ Subagent C: Comment triage (what's actionable?)    │
├──────────────────────────────────────────────────────────┤
│ 4. PLAN                                                  │
│    Present findings + proposed changes to user            │
│    Wait for approval before editing                      │
├──────────────────────────────────────────────────────────┤
│ 5. IMPLEMENT  (main context or targeted subagents)       │
│    Apply approved changes, one concern at a time          │
├──────────────────────────────────────────────────────────┤
│ 6. VERIFY                                                │
│    Lint, type check, test changed files                   │
├──────────────────────────────────────────────────────────┤
│ 7. COMMIT + PUSH                                         │
│    Push to the PR branch, report summary                  │
└──────────────────────────────────────────────────────────┘
```

### Step 1: Fetch PR Context

Gather all PR data using `gh` CLI in the main context (structured, lightweight):

```bash
# Metadata
gh pr view <number> --json title,body,state,headRefName,baseRefName,mergeable,statusCheckRollup,changedFiles,additions,deletions

# Full diff
gh pr diff <number>

# Inline review comments
gh api repos/<owner>/<repo>/pulls/<number>/comments \
  --jq '.[] | {user: .user.login, body: .body, path: .path, line: .line, created_at: .created_at}'

# Top-level issue comments
gh api repos/<owner>/<repo>/issues/<number>/comments \
  --jq '.[] | {user: .user.login, body: .body, created_at: .created_at}'
```

Extract:
- **PR goal** — What is this PR trying to accomplish?
- **Changed files** — List of files and change size
- **Review feedback** — All human and bot comments, grouped by file
- **Unresolved threads** — Comments that haven't been addressed
- **CI status** — Any failing checks

### Step 2: Setup Workspace

1. Check for an existing worktree for this branch
2. If none exists, create one:
   ```bash
   git fetch origin <branch-name>
   git worktree add .worktrees/<branch-name> origin/<branch-name>
   ```
3. Install dependencies if needed (check for lockfile changes)
4. Verify the worktree is on the correct branch and up to date

### Step 3: Analyze (Parallel Subagents)

Dispatch three subagents simultaneously. Each operates on the worktree.

#### Subagent A — Deep Code Review + Impact Analysis

**Task:** Review every change in the diff for correctness and impact.

**Instructions to subagent:**
- Read each changed file IN FULL (not just the diff hunks — you need surrounding context)
- For each change, answer:
  1. **Correctness:** Does this change do what the PR description claims?
  2. **Edge cases:** Are there inputs/states that would break this?
  3. **Reactivity (Svelte):** If touching reactive state, are updates immutable? Will the UI re-render?
  4. **Side effects:** What other components/functions consume the changed code? Trace callers.
  5. **Regression risk:** Could this break existing behavior? Check callers and tests.
- For EACH function modified, grep for all call sites and verify the contract is preserved
- Check if the PR description matches what the code actually does (description drift)

**Return format:**
```
findings:
  - file: <path>
    severity: critical | warning | info
    line: <line number>
    issue: <one-line summary>
    detail: <explanation with evidence>
    suggestion: <proposed fix, if any>
  - ...
summary: <2-3 sentence overall assessment>
risk_level: high | medium | low
```

#### Subagent B — Security Review

**Task:** Audit all changes for security vulnerabilities.

**Instructions to subagent:**
- Check for OWASP Top 10 patterns in changed code:
  - XSS: Is user input rendered without sanitization?
  - Injection: Are queries/commands built from user input?
  - Broken access control: Are auth checks present and correct?
  - Sensitive data exposure: Are secrets, tokens, or PII mishandled?
  - Insecure direct object references: Can users access others' data?
- Check for Svelte-specific security:
  - `{@html}` usage with untrusted data
  - Client-side auth bypass possibilities
  - Sensitive data in client-side stores
- Check for infrastructure concerns:
  - API keys or secrets in code
  - Overly permissive CORS or CSP
  - Debug/dev code left in production paths
- Review any new dependencies for known vulnerabilities

**Return format:**
```
findings:
  - file: <path>
    severity: critical | high | medium | low
    category: <OWASP category or description>
    issue: <one-line summary>
    detail: <explanation with proof>
    remediation: <specific fix>
  - ...
clean: true | false
summary: <overall security posture assessment>
```

#### Subagent C — Comment Triage

**Task:** Analyze all review comments and classify them.

**Instructions to subagent:**
- Read every review comment (inline + top-level) from humans (ignore bots)
- For each comment, classify:
  1. **Actionable code change** — Reviewer is requesting a specific modification
  2. **Question** — Reviewer is asking for explanation (may not need code change)
  3. **Resolved** — Already addressed by a subsequent commit
  4. **Stylistic** — Formatting, naming preference (low priority)
  5. **Architectural** — Needs design discussion before code change
- For actionable comments, identify:
  - The specific file and location
  - What the reviewer wants changed
  - Whether the latest commit already addresses it
- Check the commit history to see if follow-up commits already addressed feedback

**Return format:**
```
comments:
  - author: <username>
    file: <path or "top-level">
    type: actionable | question | resolved | stylistic | architectural
    summary: <what they want>
    addressed: true | false
    proposed_response: <how to address it, if actionable>
  - ...
unresolved_count: <number>
blocking_count: <number of actionable unresolved>
```

### Step 4: Plan

Synthesize all subagent results in the main context. Present to the user:

```
## PR Review Summary

### Risk Assessment
<risk level from Subagent A>

### Security
<clean/findings from Subagent B>

### Unresolved Feedback
<list from Subagent C>

### Proposed Changes
1. <change description> — addresses <comment/finding>
2. ...

### Questions for Reviewer
- <any questions that need reviewer input>

Shall I proceed with these changes?
```

**STOP and wait for user approval before making any edits.**

If the user disagrees with any proposed change, adjust the plan. If there are architectural comments, flag them for human discussion rather than making autonomous decisions.

### Step 5: Implement

For each approved change:
1. Make the edit in the worktree
2. Verify the edit is correct by reading the result
3. If the change is non-trivial, dispatch a subagent to trace impact (grep callers, check tests)

**Principles:**
- One concern per commit (if multiple changes are needed)
- Never change code that isn't part of the approved plan
- If you discover a new issue while implementing, flag it — don't silently fix it

### Step 6: Verify

Run quality checks on ALL changed files (original PR changes + new fixes):

```bash
# Lint only changed files
npx eslint <changed-files>

# Type check
npx svelte-check --tsconfig ./tsconfig.json --threshold error

# Tests
npm run test:unit -- --reporter=verbose <relevant-test-files>
```

If verification fails, fix issues before proceeding. If a pre-existing test failure unrelated to our changes is blocking, note it but don't fix it (out of scope).

### Step 7: Commit + Push

```bash
git add <specific-files>
git commit -m "$(cat <<'EOF'
fix: address PR review feedback

<detailed description of what was changed and why>

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
git push origin <branch-name>
```

### Step 7b: Reply to Review Comments

After pushing fixes, reply to each addressed review comment explaining what was done. Do NOT resolve/dismiss the threads — let the reviewer do that.

```bash
# Reply to an inline review comment thread
gh api repos/<owner>/<repo>/pulls/<number>/comments/<comment_id>/replies \
  -f body="Addressed — <brief explanation of the fix>"
```

- Reply to every comment that was addressed by the pushed changes
- Keep replies concise — state what was done, not why the reviewer was right
- Do NOT resolve or dismiss review threads — that's the reviewer's prerogative

**Report completion:**
```
## PR Review Complete

**Changes pushed** to <branch-name>

### What was done:
- <list of changes made>

### Findings addressed:
- <reviewer comment → how it was resolved>

### Open items:
- <anything that needs human decision>

PR: <url>
```

## Subagent Dispatch Reference

| Phase | Subagent Purpose | Parallel? | Returns |
|-------|-----------------|-----------|---------|
| Analyze | Deep code review + impact | Yes (with B, C) | Findings + risk level |
| Analyze | Security audit | Yes (with A, C) | Vulnerabilities + clean status |
| Analyze | Comment triage | Yes (with A, B) | Classified comments + action items |
| Implement | Impact trace (per change) | As needed | Caller list + safety assessment |

## Input Handling

The skill accepts:
- **PR URL:** `https://github.com/<owner>/<repo>/pull/<number>` — parse owner, repo, number
- **PR number:** `#1234` or `1904` — use current repo context
- **Branch name:** `feature/some-branch` — look up associated PR via `gh pr list --head <branch>`

If the repo cannot be inferred from the current directory, ask the user.

## Safety Rails

- **Never force-push** — only `git push`, never `git push --force`
- **Never merge the PR** — only push commits. Merging is a human decision.
- **Never dismiss reviews** — only address feedback with code
- **Stop on architectural disagreements** — if a reviewer's comment implies a design change, flag it for human discussion
- **Verify before push** — lint + type check + test must pass before pushing
- **One branch only** — all changes go on the existing PR branch, never create a new branch
- **Preserve existing behavior** — when fixing review feedback, ensure the original PR intent is maintained
- **Flag scope creep** — if review comments ask for features beyond the original PR scope, flag them rather than implementing

## Error Handling

- **PR is merged/closed:** Report status and exit. Don't reopen or create new PRs.
- **Branch has conflicts:** Report the conflict and ask user how to proceed. Don't auto-rebase.
- **CI is failing on unrelated tests:** Note it in the report but don't block on it.
- **No actionable comments:** Report that the PR looks good and has no unresolved feedback. Still run the code review and security audit — they may find issues reviewers missed.

## Integration

**Uses:**
- `implement-ticket` patterns — for worktree management and PR workflow
- `superpowers:using-git-worktrees` — for workspace isolation
- `superpowers:finishing-a-development-branch` — for push/PR patterns

**Pairs with:**
- `linear-tickets` — if review reveals issues that warrant separate tickets
- `aws-debug` — if PR touches infrastructure code
