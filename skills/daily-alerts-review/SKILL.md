---
name: daily-alerts-review
description: Review Slack alerts channel and Sentry errors from the past 24 hours. For small, well-defined issues, dispatches a subagent to fix via worktree/branch/PR. For larger or ambiguous issues, creates a Linear ticket with Triage status. Designed for daily use, ideally scheduled via /scheduler.
---

# Daily Alerts Review

## Overview

Sweep Slack #alerts and Sentry for **production issues only** from the past 24 hours. Triage each issue into one of two buckets:
1. **Auto-fix** — small, well-defined bug → subagent creates worktree, branch, fix, and PR
2. **Ticket** — larger or ambiguous issue → Linear ticket with status Triage

**Production-only scope:** Only read from the prod Slack channel (`#alerts`, ID: `C051JF9BL4D`) and Sentry issues tagged with `environment:prod` or `environment:production`. Ignore `alerts-staging`, `alerts-dev`, and non-production Sentry environments entirely.

**Announce at start:** "Running daily alerts review — scanning production Slack alerts and Sentry for the past 24h."

## Context Management — CRITICAL

This skill touches many data sources. **Protect the main context aggressively:**

- The main thread is an **orchestrator only** — it makes decisions and dispatches work.
- **ALL exploration goes to subagents**: reading Slack messages, fetching Sentry details, searching the codebase for related code. Never dump raw alert/error data into the main context.
- Each subagent gets **one focused task** with clear deliverables.
- Subagent results should return **structured summaries**, not raw data.
- Run independent subagents **in parallel** whenever possible.

## Prerequisites

This skill requires the following environment variables. Source them from a shared env file or set them in your shell before running.

| Variable | Purpose |
|----------|---------|
| `SLACK_BOT_TOKEN` | Slack bot token with `channels:history` and `chat:write` scopes |
| `SLACK_DM_CHANNEL` | Your personal Slack DM channel ID (for the summary report) |
| `LINEAR_API_KEY` | Linear API key with issue create permissions |

Additionally:
- `sentry-cli` must be installed and configured (`~/.sentryclirc`) with an auth token for the `stream-zw` org.

## The Process

```
┌─────────────────────────────────────────────────┐
│ 0. LOAD HISTORY                                 │
│    Read last 7 days from alerts-review ledger    │
│    Build "already seen" sentry_issue_id set      │
├─────────────────────────────────────────────────┤
│ 1. GATHER  (parallel subagents)                 │
│    ├─ Subagent A: Read Slack #alerts (24h)      │
│    └─ Subagent B: Fetch Sentry issues (24h)     │
├─────────────────────────────────────────────────┤
│ 2. DEDUPLICATE & MERGE                          │
│    Combine results, deduplicate by error/issue   │
│    Filter out issues already in history          │
├─────────────────────────────────────────────────┤
│ 3. TRIAGE each unique NEW issue                 │
│    ├─ Auto-fixable? → dispatch fix subagent     │
│    └─ Not auto-fixable? → create Linear ticket  │
├─────────────────────────────────────────────────┤
│ 4. SAVE today's ledger file                     │
├─────────────────────────────────────────────────┤
│ 5. REPORT summary to user                       │
└─────────────────────────────────────────────────┘
```

### Step 0: Load History (Dedup Against Previous Runs)

**Path:** `~/.claude/data/alerts-review/`
**Format:** One JSON file per run, named `YYYY-MM-DD.json`

```bash
# Ensure the directory exists
mkdir -p ~/.claude/data/alerts-review
```

Load all files from the last 7 days (glob `~/.claude/data/alerts-review/*.json`, filter by filename date). Collect every `sentry_issue_id` into an **already-seen set**. This set is used in Step 2 to skip issues that were already triaged in a previous run.

**Each daily file is an array of entries:**
```json
[
  {
    "sentry_issue_id": "7338939041",
    "title": "OpenAI API 500 during report synthesis",
    "action": "ticket",
    "linear_ticket_id": "ENG-4795"
  },
  {
    "sentry_issue_id": "7337404768",
    "title": "datetime.date strip() crash",
    "action": "auto-fix",
    "pr_url": "https://github.com/Stream-claims/backend/pull/3799"
  },
  {
    "sentry_issue_id": "7337410375",
    "title": "Case processing conflict",
    "action": "skip"
  }
]
```

**If no history files exist** (first run or all older than 7 days), proceed normally — everything is new.

### Step 1: Gather Alerts (Parallel Subagents)

**Subagent A — Slack Alerts (prod only):**
- Read **only** the `#alerts` channel (prod). Channel ID: `C051JF9BL4D` — no discovery step needed.
  ```bash
  OLDEST=$(date -v-24H +%s)
  curl -s -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
    "https://slack.com/api/conversations.history?channel=C051JF9BL4D&oldest=$OLDEST&limit=100"
  ```
- Do NOT read `alerts-staging` (C07SF1Z7EEB) or `alerts-dev` (C07T89HHSBT)
- For each alert, extract: timestamp, error message, stack trace (if any), affected service/customer, severity
- Return a structured list of alerts

**Subagent B — Sentry Issues (prod only):**
- Use `sentry-cli` to list unresolved production issues from the past 24h:
  ```bash
  # Query each project for unresolved issues in production environments
  for PROJECT in customer-portal customer-portal-api ingestion_pipeline pdf_gen post-synthesis-processing processing-pipeline; do
    echo "=== $PROJECT ==="
    sentry-cli issues list --project "$PROJECT" --query "is:unresolved environment:prod environment:production" --status unresolved 2>/dev/null
  done
  ```
- For high-frequency or new issues, get details with `sentry-cli issues show <ISSUE_ID>`
- Extract: title, error message, frequency, affected users, stack trace, first/last seen
- Return a structured list of issues

**Both subagents return structured summaries in this format:**
```
- id: <alert/issue id>
  source: slack | sentry
  title: <one-line summary>
  error: <error message>
  stack_trace_summary: <file:line of root cause, if identifiable>
  service: <affected service/lambda>
  customer: <affected customer, if known>
  severity: critical | high | medium | low
  frequency: <count in 24h>
  first_seen: <timestamp>
```

### Step 2: Deduplicate & Merge

In the main context (lightweight — just matching IDs/titles):
- Match Slack alerts to Sentry issues by error message or stack trace
- Group duplicates
- **Filter out any issue whose `sentry_issue_id` is in the already-seen set from Step 0** — note these as "previously tracked" in the report
- Produce a **unique NEW issue list** with combined context from both sources
- Sort by severity (critical first), then frequency

### Step 3: Triage Each Issue

For each unique issue, classify it:

#### Auto-Fix Criteria (ALL must be true):
- [ ] Root cause is identifiable from the stack trace or error message
- [ ] Fix is localized to 1-3 files
- [ ] No architectural or design decisions needed
- [ ] Not customer-data-sensitive (no data migration, no schema change)
- [ ] Similar patterns exist in the codebase (it's a known type of bug)
- [ ] Severity is not critical (critical issues need human review)

**Examples of auto-fixable issues:**
- Unhandled null/undefined in a known code path
- Missing error handling for an edge case
- Typo in a config or string
- Off-by-one errors with clear stack traces
- Missing field validation that's straightforward to add

#### Auto-Fix Path:

Dispatch a subagent (using `implement-ticket` patterns) for each auto-fixable issue:

```
Subagent task:
1. Search codebase for the error location (use stack trace)
2. Understand the root cause
3. Create worktree: .worktrees/fix-<short-description>
4. Branch: fix/<short-description>
5. Implement the fix (minimal, focused)
6. Run lint + type check + relevant tests
7. Commit and push
8. Create PR
9. Return: { pr_url, branch, files_changed, summary }
```

**IMPORTANT:** Each fix subagent runs in an **isolated worktree** (`isolation: "worktree"`). This prevents conflicts between parallel fixes.

#### Ticket Path:

For issues that don't meet auto-fix criteria, create a Linear ticket:

**Try MCP first**, fall back to curl if MCP is unavailable:

**Option A — MCP (preferred):**
Use `mcp__claude_ai_Linear__save_issue` with the fields below.

**Option B — curl fallback (when MCP is down):**
```bash
# First, get the Engineering team ID and Triage state ID (cache these)
curl -s -X POST https://api.linear.app/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: $LINEAR_API_KEY" \
  -d '{"query":"{ teams { nodes { id name } } }"}'

curl -s -X POST https://api.linear.app/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: $LINEAR_API_KEY" \
  -d '{"query":"{ workflowStates { nodes { id name type } } }"}' | python3 -c "import sys,json; [print(n['id'],n['name']) for n in json.load(sys.stdin)['data']['workflowStates']['nodes'] if n['name']=='Triage']"

# Create the issue (priority: 1=Urgent, 2=High, 3=Medium, 4=Low)
curl -s -X POST https://api.linear.app/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: $LINEAR_API_KEY" \
  -d '{"query":"mutation($input: IssueCreateInput!) { issueCreate(input: $input) { success issue { id identifier url } } }","variables":{"input":{"teamId":"<TEAM_ID>","title":"<TITLE>","description":"<DESCRIPTION>","stateId":"<TRIAGE_STATE_ID>","priority":<1-4>}}}'
```

**Fields for both options:**
- **Title:** Clear, actionable summary
- **Description:** Include:
  - Error message and stack trace summary
  - Affected service/customer
  - Frequency and severity
  - Source links (Sentry issue URL, Slack message link if available)
  - Suggested investigation starting points
- **Status:** Triage
- **Priority:** Map from severity (critical→Urgent, high→High, medium→Medium, low→Low)
- **Labels:** bug
- **Team:** Engineering

### Step 4: Save Today's Ledger

After all triage actions complete, write today's file to `~/.claude/data/alerts-review/YYYY-MM-DD.json`.

**Include every issue from this run** — not just ticketed/fixed ones. Skipped issues should also be recorded so they don't get re-evaluated tomorrow.

```json
[
  { "sentry_issue_id": "123", "title": "...", "action": "ticket", "linear_ticket_id": "ENG-XXXX" },
  { "sentry_issue_id": "456", "title": "...", "action": "auto-fix", "pr_url": "https://..." },
  { "sentry_issue_id": "789", "title": "...", "action": "skip" }
]
```

Use `Write` tool to create the file. If the file already exists (re-run on same day), overwrite it.

**Cleanup:** Delete any files older than 14 days (keeps the directory small while preserving a buffer beyond the 7-day lookback).

### Step 5: Report

**Send a Slack DM to the operator** using curl with the Slack bot token:

```bash
curl -s -X POST 'https://slack.com/api/chat.postMessage' \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  -H 'Content-Type: application/json' \
  --data-raw '{"channel":"'"$SLACK_DM_CHANNEL"'","text":"Daily Alerts Review","blocks":[...]}'
```

Use Slack mrkdwn formatting (single *bold*, _italic_, no ## headers, no **double asterisks**):

```
*Daily Alerts Review — <date>*

*Auto-Fixed (<count>)*
• <title> → <pr_url> (<n> files)
• ...

*Tickets Created (<count>)*
• <title> → <ticket_id> (<priority>)
• ...

*Already Tracked (<count>)* _(if any)_
• <title> — <linear_ticket_id or pr_url from history>

*Skipped (<count>)* _(if any)_
• <reason>: <title>

_<n> alerts scanned · <n> new issues · <n> already tracked · <n> fixed · <n> ticketed_
```

If there are zero issues found, send a simple message:
```
*Daily Alerts Review — <date>*
All clear — no new alerts or Sentry issues in the past 24h.
```

Also print the same summary to stdout for the execution log.

## Subagent Dispatch Reference

**All subagents in this skill MUST use `model: "sonnet"`.**

| Phase | Subagent Purpose | Parallel? | Model | Returns |
|-------|-----------------|-----------|-------|---------|
| Gather | Read Slack alerts (via curl + bot token) | Yes (with Sentry) | sonnet | Structured alert list |
| Gather | Fetch Sentry issues | Yes (with Slack) | sonnet | Structured issue list |
| Triage | Investigate if auto-fixable | Yes (per issue) | sonnet | Fix/ticket recommendation |
| Fix | Implement fix in worktree | Yes (per fix, isolated) | sonnet | PR URL + summary |
| Ticket | Create Linear ticket | Yes (per ticket) | sonnet | Ticket ID |

## Configuration

### Slack
- Requires `SLACK_BOT_TOKEN` and `SLACK_DM_CHANNEL` environment variables
- All Slack reads and writes use `curl` with the bot token — do NOT use MCP Slack tools
- Prod alerts channel: `#alerts` — ID `C051JF9BL4D` (hardcoded, no discovery needed)

### Sentry
- Org: `stream-zw` (configured in `~/.sentryclirc`)
- Projects: `customer-portal`, `customer-portal-api`, `ingestion_pipeline`, `pdf_gen`, `post-synthesis-processing`, `processing-pipeline`
- **Production environments only:** Filter to `environment:prod` and `environment:production`. Ignore `dev`, `local`, `staging`.
- Use `sentry-cli` for all queries (auth token in `~/.sentryclirc`)

### Linear
- Requires `LINEAR_API_KEY` environment variable
- **Primary:** Use `mcp__claude_ai_Linear__save_issue` MCP tool
- **Fallback:** If MCP is unavailable, use `curl` against `https://api.linear.app/graphql` with `$LINEAR_API_KEY`
- Team: Engineering (default)
- Triage status: query workflow states via MCP or GraphQL API to find the Triage state ID

## Safety Rails

- **Never auto-fix critical issues** — always create a ticket for human review
- **Never auto-fix issues touching customer data** — schema changes, migrations, data transformations
- **Never merge PRs** — only create them. Human reviews and merges.
- **Dry-run mode:** If user says "dry run" or "preview", show what would happen without creating PRs or tickets
- **Rate limit:** Cap auto-fixes at 5 per run. If more qualify, ticket the rest with a note.

## Integration

**Uses:**
- `implement-ticket` patterns — for worktree/branch/PR creation
- `linear-tickets` patterns — for ticket creation and search
- `aws-debug` — if alerts point to infrastructure issues

**Pairs with:**
- `/scheduler` — schedule this to run daily (e.g., `schedule-add "daily-alerts-review" "every weekday at 9am /daily-alerts-review"`)
