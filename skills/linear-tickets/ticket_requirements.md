# Ticket Requirements

PM requirements for ticket quality. Every ticket must meet these standards.

## Required Fields

### Title

**Style**: Action-oriented verb phrase. NOT patterned/templated.

**Good examples:**
- "Add DOB field to extraction model for adjudication class"
- "Turn off hallucination detection for ReportPrep"
- "Eval model against production examples"
- "Fix duplicate segments appearing in EPIC files"

**Bad examples:**
- "[Feature] DOB field" (patterned - don't use brackets)
- "DOB field issue" (not action-oriented)
- "Adjudication class improvements" (vague)

### Context / Problem

Describe the problem to be solved and WHY it matters.

- What's broken or missing?
- Why does this need to be fixed now?
- What's the impact if we don't do this?

### Solution / Task

High-level description of the approach or task.

- What are we going to do?
- How will we solve the problem?
- What's the scope?

## Helpful Fields (Include When Available)

### Current State / Desired State

Articulate how things work today vs how they should work.

**Format:**
```
Current state: [description, with example]
Desired state: [description, with example]
```

### Examples in the Wild

Real instances of the problem occurring. Useful for:
- Testing the fix later
- Illustrating the issue to others
- Scoping the problem

Include: case IDs, file names, screenshots, error messages

### Customer Scope

Which customers should be impacted (vs not).

- Is this all customers or specific ones?
- Are there customers we should NOT change behavior for?

## Labels

**Do NOT use labels** except for bug tickets.

For bugs: apply appropriate bug label.

## Project Attachment

**If work is part of a broader initiative:**
- Find the existing Linear Project
- Attach ticket to it

**If no obvious project exists:**
- Suggest 2-3 possible projects that might fit
- If user rejects all: create dedicated project

**Project rules:**
- Projects don't require Initiative attachment
- Project lead doesn't have to be the ticket creator
- When in doubt, ask user rather than guessing

## Team

Default: **Engineering**

Only change if explicitly requested.

## File References

**Do NOT reference uncommitted/temporary files** in ticket descriptions.

Files like investigation notes, scratch files, or local analysis docs won't exist in the repo - linking to them creates broken references.

**Instead:**
- Include crucial findings inline in the ticket description
- For detailed logs/data, add as a comment after ticket creation
- Reference only committed files (code, configs, docs in repo)

**Bad:** "See `INVESTIGATION_FINDINGS.md` for details"
**Good:** Include the key findings directly in the Context section
