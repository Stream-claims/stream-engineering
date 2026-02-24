---
name: linear-tickets
description: Manage Linear tickets and projects through conversation. Use when creating, updating, or searching tickets, checking backlog, or syncing work progress. ALSO proactively suggest tickets when significant work is done without one, or when a conversation has been going for a while with consistent focused work. (project)
---

# Linear Tickets Skill

Manage Linear tickets and projects through conversation. Reduces friction by making Linear interaction context-aware.

## When to Use

- User asks to create, update, or search tickets
- User asks about their backlog or what to work on
- User finishes significant work and no ticket is associated with the conversation
- User asks to sync progress or check ticket status

## Guiding Principles

### Context Persistence

Once you know which ticket the current work is associated with, carry that forward. Don't re-ask.

### Proactive Ticketing

Proactively suggest creating a ticket when:
- Significant work is done (code written, bugs fixed, features added) and NO ticket is associated
- A conversation has been going for a while with consistent, focused work (even if not "finished")
- The work represents a coherent unit that should be tracked

Don't wait for work to be "complete" - suggest tracking early if the work pattern suggests it's ticket-worthy.

### Search-First

Before creating new tickets, ALWAYS search for existing related tickets. Scope searches to tickets created by OR assigned to the user.

### Gather Missing Information

If context doesn't provide enough info to meet ticket requirements (see `ticket_requirements.md`), interview the user. Don't create incomplete tickets.

**Propose from context when possible:**
> "Based on our conversation, the problem is X and solution is Y. Sound right?"

**Ask only for gaps:**
- Priority/urgency
- Customer impact (if relevant)
- Blockers/dependencies
- Project attachment

## Workflows

### 1. Retrospective Ticket Creation

When user wants to track work done in conversation:

1. **Synthesize context** from conversation:
   - What was built/changed?
   - Why? What problem does it solve?
   - Current state vs desired state?
   - Any examples encountered?

2. **Search for existing tickets** (created by OR assigned to user):
   - If match found: propose updating existing vs creating new

3. **Search for related projects**:
   - Could this attach to an existing project?
   - Is there a cluster of related tickets suggesting a project?

4. **Interview for gaps** (only what's not inferrable):
   - Priority?
   - Customer impact?
   - Blockers?

5. **Draft ticket** following requirements in `ticket_requirements.md`

6. **Present draft** -> User confirms -> Create

### 2. Progress Sync

When user wants to update a ticket or sync status:

1. Identify current ticket (ask if unknown, remember if known)
2. Compare Claude's understanding of work vs ticket state
3. Identify drift/gaps
4. Propose updates:
   - Update description
   - Add comment
   - Change status
   - Flag misalignment

### 3. Search/Triage

When user asks to find tickets:

- Search only tickets created by OR assigned to user
- Support natural language: "find tickets about EPIC", "stale items"
- Max 15 results shown; summarize if more

### 4. Project Suggestion

When creating a ticket or standalone:

1. Search for existing projects that fit
2. Look for cluster of related tickets without a project
3. Propose:
   - Attach to existing project, OR
   - Create new project (if cluster), OR
   - No project needed
4. If user rejects: offer to create dedicated project
   - Projects don't need Initiative attachment
   - Project lead doesn't have to be user

### 5. "What's on My Plate"

When user asks what to work on:

**Scope:**
- Tickets assigned to user
- Tickets from projects user is member of

**Output** (max 15 items, summarize if more):
- In Progress (verify still active)
- Todo (prioritized)
- Backlog (summary)
- Stale items

## Defaults

| Setting | Default |
|---------|---------|
| Team | Engineering |
| Search scope | Created by OR assigned to user |
| Max results | 15 (summarize if more) |
| Labels | None (except for bugs) |

## References

- `ticket_requirements.md` - PM requirements for ticket quality
