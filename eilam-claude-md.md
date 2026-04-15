# Core Principles

- **Simplicity First**: Make every change as simple as possible. Impact minimal code.
- **No Laziness**: Find root causes. No temporary fixes. Senior developer standards.
- **Minimal Impact**: Changes should only touch what's necessary. Avoid introducing bugs.
- **Demand Elegance**: For non-trivial changes, pause and ask "is there a more elegant way?" Skip for simple, obvious fixes.
- **Guard the Main Context**: The main context should only contain decisions and results. Offload all research, exploration, and investigation to subagents.

# Workflow Orchestration

## Plan Mode Default
- Enter plan mode for ANY non-trivial task (3+ steps or architectural decisions)
- **Skip plan mode**: single-file bug fixes, typo corrections, config changes, adding a field end-to-end
- **Use plan mode**: multi-service changes, new features, refactors touching 3+ files, anything with architectural decisions
- If something goes sideways, STOP and re-plan immediately - don't keep pushing
- Write detailed specs in the plan mode output to reduce ambiguity

## Autonomous Bug Fixing
- When given a bug report: just fix it. Don't ask for hand-holding
- Point at logs, errors, failing tests - then resolve them
- Zero context switching required from the user

## Authentication
- When logging into Stream Claims portals (dev, staging, prod), just do it — don't ask for permission. Use SSO → org "stream" → `eilam@stream.claims`.

## When Blocked
- If blocked by expired tokens, failed auth, missing permissions, or other blockers: create `/tmp/claude/<descriptive-name>.txt` with what's blocked and what action is needed, then `open` it.
- Continue with whatever work is still possible.

## Subagent & Context Management
- The main context is sacred — only decisions, edits, and user communication belong here.
- Dispatch independent tasks concurrently — never serialize what can run in parallel.
- One task per subagent. For complex problems, use more subagents rather than overloading one.
- Use `model: "opus"` for all delegated work.

## Git Workflow
- **ALWAYS** create a worktree for new tasks. No exceptions — never work directly on an existing branch.
- Worktree in `.worktrees/`, branch `eilam/<name>`. Fetch and branch off `origin/master` (not local master).
- PR when done.
- Before checking out an existing branch, verify it hasn't already been merged.
- `~/.claude` is a versioned git repo (`eilam-stream/claude-config`). After editing CLAUDE.md, skills, settings, or other tracked config, commit and push.

# Done Checklist

Run through every item before reporting any task complete.

1. Add basic test coverage for new code
2. Run linter/formatter and tests for changed files
3. Verify no type errors in changed files
4. If the change touches an API endpoint, verify request/response shape matches the DTO
5. If dependencies changed, verify no version conflicts and lock files are consistent
6. No debug prints, TODOs, or commented-out code left behind
7. **Verify the change works:**
   - **Customer portal only**: Run locally using the project's dev skill. No deployment needed.
   - **Backend only**: Check ongoing/recent EilamDev deployments (`gh run list`). If another workstream is mid-deploy or queued, ask before proceeding. Otherwise deploy and verify.
   - **Both**: Apply both checks above.
8. Ask yourself: "Would a staff engineer approve this?"
9. Open PR, then run `/pr-monitor-loop` to watch for review comments.

# Quick Reference

- **Dev env**: EilamDev — requires Tailscale exit node set to dev
- **Stack names**: `EilamDevStack` (backend), `EilamDev` (frontend) — case-sensitive
- **Case flow**: `case_upload → auto_segmentation → segmentation → segmentation_complete → review_ready → submission_ready → summary_generated → submitted/manually_submitted` (branch states: `adding_files`, `segmentation_additional`)
- **Segment flow**: `pre_segment → pre_segment_in_progress → pre_segment_complete → review_ready → review_complete`

# Search Strategy

- **Default to QMD** (`/qmd` skill) for searching notes, docs, architecture, security, interview notes, and prior decisions. It indexes the Obsidian vault, backend, customer-portal, data-engineering, and ~/.claude.
- **Fallback to Grep/Glob** only if QMD returns no results or you need exact file-level code search within a specific repo.

# Obsidian Vault

- **Path**: `~/Documents/Obsidian Vault/`
- **Start with**: `index.md` — vault-wide navigation index for AI agents
- **Sections**: Interviews, Security, Engineering, Product, Marketing, Misc
- **Security deep-dive**: `Security/_index.md` for detailed sub-index (20+ files)

# Google Workspace

- Use `gws` CLI (`/gws` skill) for interacting with Google Sheets, Slides, Docs, and Drive. Faster and more reliable than Chrome automation.
- Use `/nano-banana` skill to generate images for slides and creative content (logos, illustrations, visual slide backgrounds). Not for charts/graphs — use for creative visuals only.

# Slack

- **As Eilam** (MCP `slack_send_message`): team-visible posts — handoffs, announcements, anything others see.
- **As Bot** (curl via `~/.claude/briefing-env`): automated DMs to self — scheduled/headless skills only.
- **Formatting**: Slack mrkdwn — single `*bold*`, no markdown tables, no `---`.
