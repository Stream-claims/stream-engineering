---
name: sentry-cli
description: Use when interacting with Sentry via CLI - listing issues, managing releases, uploading sourcemaps, resolving/muting issues, or checking project status. Also use when debugging production errors and needing to query Sentry data from the terminal.
---

# Sentry CLI Reference

## Setup

Config file: `~/.sentryclirc`

```ini
[defaults]
org=stream-zw

[auth]
token=<personal-token>
```

Override per-command: `--org`, `--project`, `--auth-token`

## Our Projects

| Slug | Purpose |
|------|---------|
| customer-portal | Frontend |
| customer-portal-api | Backend API |
| ingestion_pipeline | Document ingestion |
| pdf_gen | PDF generation |
| post-synthesis-processing | Post-synthesis |
| processing-pipeline | Processing pipeline |

## Quick Reference

### Check auth & config
```bash
sentry-cli info
```

### List projects
```bash
sentry-cli projects list
```

### Issues

```bash
# List unresolved issues for a project
sentry-cli issues list -p customer-portal-api --status unresolved

# List all issues (may be limited)
sentry-cli issues list -p customer-portal-api -a

# Resolve specific issues by ID
sentry-cli issues resolve -i ISSUE_ID

# Bulk resolve all unresolved issues
sentry-cli issues resolve -p customer-portal-api --status unresolved

# Mute issues
sentry-cli issues mute -i ISSUE_ID

# Unresolve issues
sentry-cli issues unresolve -i ISSUE_ID
```

### Events

```bash
# List recent events for a project
sentry-cli events list -p customer-portal-api
```

### Releases

```bash
# List releases
sentry-cli releases list -p customer-portal-api

# Create a new release
sentry-cli releases new VERSION -p customer-portal-api

# Set commits for a release (auto-detect from git)
sentry-cli releases set-commits VERSION --auto

# Finalize a release
sentry-cli releases finalize VERSION

# Full release flow
sentry-cli releases new VERSION -p customer-portal-api
sentry-cli releases set-commits VERSION --auto
sentry-cli releases finalize VERSION
```

### Sourcemaps

```bash
# Upload sourcemaps for a release
sentry-cli sourcemaps upload --release VERSION ./dist

# Inject debug IDs into source files and sourcemaps
sentry-cli sourcemaps inject ./dist

# Resolve a sourcemap for debugging
sentry-cli sourcemaps resolve -r VERSION FILE LINE COLUMN
```

### Deploys

```bash
# Create a deploy for a release
sentry-cli deploys new -r VERSION -e production
```

### Cron Monitors

```bash
# List monitors
sentry-cli monitors list

# Wrap a command with monitoring
sentry-cli monitors run MONITOR_SLUG -- command-to-run
```

### Send Test Event

```bash
# Send a test event to verify DSN/config
sentry-cli send-event -m "Test event from CLI"
```

## Common Patterns

**Triage unresolved errors:**
```bash
sentry-cli issues list -p customer-portal-api --status unresolved
```

**Release with sourcemaps (frontend):**
```bash
VERSION=$(sentry-cli releases propose-version)
sentry-cli releases new "$VERSION" -p customer-portal
sentry-cli sourcemaps inject ./dist
sentry-cli sourcemaps upload --release "$VERSION" ./dist
sentry-cli releases finalize "$VERSION"
sentry-cli deploys new -r "$VERSION" -e production
```

## Tips

- Use `--log-level debug` to troubleshoot auth or upload issues
- Use `--quiet` to suppress output (useful in CI)
- The `-p` flag accepts project slug (not name)
- Run `sentry-cli COMMAND --help` for full flag details
