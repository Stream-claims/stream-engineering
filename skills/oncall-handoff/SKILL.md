---
name: oncall-handoff
description: Generate and post an on-call handoff message to Slack. Pulls Sentry issues (7 days), completed Linear work (7 days), and produces a concise summary for the next on-call engineer. Use when the user says "oncall handoff", "handoff", "on-call summary", or "shift handoff".
---

# On-Call Handoff

## Overview

Generate a concise on-call handoff message and send it to `#eng` (channel ID: `C059GLLTUKT`). The message covers production Sentry issues and completed Linear work from the past 7 days.

**Announce at start:** "Generating on-call handoff — pulling Sentry (7d) and Linear completed work."

## Context Management

- Main thread is the **orchestrator** — dispatch data gathering to subagents.
- **ALL data fetching goes to subagents**: Sentry queries, Linear queries.
- Subagents return **structured summaries**, not raw API responses.
- Run Sentry and Linear subagents **in parallel**.

## The Process

```
1. GATHER  (parallel subagents)
   ├─ Subagent A: Sentry production issues (7 days)
   └─ Subagent B: Linear completed work (7 days)
2. SYNTHESIZE into handoff message
3. SEND to #eng via slack_send_message
4. PRINT summary to stdout
```

### Step 1: Gather Data

**Subagent A — Sentry (7 days):**
- Use `mcp__sentry__search_issues` with org `stream-zw`, region `https://us.sentry.io`
- Query: `unresolved issues from last 7 days with most events, production only`
- Limit: 25 issues
- For each issue, extract: title, short ID, event count, culprit/service, first/last seen, user count
- Group issues by severity based on:
  - **Needs attention**: >50 events OR user-facing with >2 users OR new + high frequency
  - **Monitor**: 5-50 events, recurring patterns, known problem areas
  - **Low noise**: <5 events, intermittent, known/expected behavior
- Return structured groups with one-line summaries per issue

**Subagent B — Linear (completed, 7 days):**
- Use `mcp__linear-server__list_issues` with `state: "completed"`, `updatedAt: "-P7D"`, limit 30
- Extract: identifier, title, assignee, completedAt
- Group by assignee
- Return structured list

### Step 2: Synthesize

Build the handoff message. Use **Slack mrkdwn** formatting.

*CRITICAL formatting rules for Slack:*
- Single `*bold*` (not `**double**`)
- Slack emoji codes for section headers
- Bullet points with `•`
- Backticks for code/IDs
- *NO* `---` horizontal rules
- *NO* markdown tables
- *NO* `>` blockquotes for content lists
- Section headers (the emoji + bold title lines) MUST be on their own line with a blank line before AND after them. If a section header is on the same line as or immediately adjacent to bullet points, Slack will collapse them together. Always separate with whitespace.

**Template — follow this EXACTLY, preserving all blank lines:**

```
:rotating_light: *On-Call Handoff — Week of <date>*


:red_circle: *Needs Attention*

• *<title>* — <context> (`<sentry-id>`) <event count> events. <action needed>.
• ...


:large_yellow_circle: *Monitor*

• *<title>* — <context> (`<sentry-id>`) <event count> events
• ...


:large_green_circle: *Low Noise / Known*

• <brief description> — <event count> events
• ...


:white_check_mark: *Completed This Week*

*<Assignee Name>*
• `<ticket-id>` <title>
• ...

*<Assignee Name>*
• `<ticket-id>` <title>
• ...


:eyes: *Top 3 to Watch*

1. *<issue>* — <why it matters>
2. *<issue>* — <why it matters>
3. *<issue>* — <why it matters>
```

**Guidelines for "Top 3 to Watch":**
- Pick the 3 most customer-visible or operationally impactful issues
- Explain *why* each matters in <=10 words
- Prioritize: customer-facing > data loss > pipeline blockage > noisy errors

### Step 3: Send to Slack

- Use `mcp__claude_ai_Slack__slack_send_message` to send directly to `#eng` (`C059GLLTUKT`)
- If user says "draft" or "preview" — use `mcp__claude_ai_Slack__slack_send_message_draft` instead, or print to stdout only

### Step 4: Report

Print a brief confirmation: "Handoff posted to #eng." and include the message link returned by Slack.

## Configuration

- **Sentry org:** `stream-zw`
- **Sentry region:** `https://us.sentry.io`
- **Slack channel:** `#eng` — `C059GLLTUKT`
- **Time window:** 7 days (default), can be overridden by user (e.g., "oncall handoff last 3 days")
- **Linear filter:** `state: "completed"`, `updatedAt: "-P7D"`

## Variations

- If user says "draft" / "preview" — use `slack_send_message_draft` or print to stdout only
- If user specifies a different channel — use that channel ID
- If user says "dry run" — print the message to stdout only, don't post to Slack
