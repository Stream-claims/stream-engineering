---
name: copy-case
description: Copy a case from one environment to another (e.g., production to dev). Use when the user says "copy case", "replicate case", "bring case to dev", or wants to reproduce a production case locally.
---

# Copy Case Between Environments

Copies a case's merged PDF from one AWS environment and uploads it as a new case in another environment using the hybrid API approach (AWS CLI + browser-authenticated API calls).

## Parameters

Gather these from the user before starting. Ask for anything missing.

| Parameter | Required | Default | Notes |
|-----------|----------|---------|-------|
| Case ID | Yes | - | The source case ID to copy |
| Source environment | No | production | `production`, `staging`, or a dev stack name |
| Destination environment | No | eilamdev | URL prefix: `{env}.dev.stream.claims` |
| New Case ID | No | same as source | Case ID in destination (must not already exist) |
| LOB | No | WC | `WC` (Workers' Comp) or `BI` (Bodily Injury) |
| First Name | No | "John" | Patient first name |
| Last Name | No | "Doe" | Patient last name |

## AWS Profiles

Profiles are user-specific. Check `CLAUDE.local.md` for the user's configured profiles. Common patterns:

| Environment | Account | Region | Typical Profile |
|-------------|---------|--------|-----------------|
| Dev | 804407225882 | us-east-1 | `default` |
| Staging | 804407225882 | us-west-2 | `staging` |
| Production | 982131176572 | us-west-2 | `production` |

If a profile is missing or unclear, ask the user.

## Prerequisites

1. **AWS CLI** configured with profiles that have access to both source and destination accounts
2. **Chrome-in-chrome** (Claude browser extension) connected for authenticated API calls
3. **Tailscale** exit node set to dev (for accessing dev environments)

## Steps

### 1. Find the source bucket and download the merged PDF

```bash
# Find the source masterbucket
aws s3 ls --profile SOURCE_PROFILE | grep masterbucket

# List files in the case's merged directory
aws s3 ls "s3://SOURCE_BUCKET/merged/CASE_ID/" --profile SOURCE_PROFILE

# Download the merged PDF to /tmp
aws s3 cp "s3://SOURCE_BUCKET/merged/CASE_ID/merged.pdf" /tmp/CASE_ID-merged.pdf --profile SOURCE_PROFILE
```

If there's no merged PDF, check for individual uploads:
```bash
aws s3 ls "s3://SOURCE_BUCKET/upload/CASE_ID/" --profile SOURCE_PROFILE
```

### 2. Get the page count

```bash
python3 -c "from pypdf import PdfReader; print(len(PdfReader('/tmp/CASE_ID-merged.pdf').pages))"
```

### 3. Set up browser session for the destination environment

Load the chrome tools and create a fresh tab:

```
ToolSearch: select:mcp__claude-in-chrome__tabs_context_mcp
tabs_context_mcp(createIfEmpty: true)
ToolSearch: select:mcp__claude-in-chrome__tabs_create_mcp
tabs_create_mcp()
```

Navigate to the destination environment via JavaScript (regular navigate fails for internal domains):
```
ToolSearch: select:mcp__claude-in-chrome__javascript_tool
javascript_tool(action: "javascript_exec", tabId: TAB_ID,
  text: "window.location.href = 'https://{dest_env}.dev.stream.claims'")
```

Wait 5s for page load, then screenshot to confirm the Cases dashboard loaded.

**If chrome-in-chrome returns "Browser extension is not connected":**
- **STOP. Do NOT attempt workarounds.**
- Tell the user: "Chrome-in-chrome extension isn't connected. Please make sure the Claude browser extension is running in Chrome and you're logged into claude.ai, then try again."

### 4. Find the destination bucket

```bash
aws s3 ls --profile DEST_PROFILE | grep masterbucket
```

### 5. Get presigned S3 URL via browser JS

```javascript
const caseId = 'NEW_CASE_ID';
const fileName = 'merged.pdf';

const response = await fetch(`/case/${encodeURIComponent(caseId)}/presigned_url_case`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ file_names: [fileName] })
});
const data = await response.json();
JSON.stringify(data);
```

### 6. Upload PDF to the destination S3 bucket

```bash
aws s3 cp /tmp/CASE_ID-merged.pdf "s3://DEST_BUCKET/upload/NEW_CASE_ID/merged.pdf" --profile DEST_PROFILE
```

### 7. Create the case via browser JS

```javascript
const response = await fetch('/case', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    caseId: 'NEW_CASE_ID',
    fileMetadata: [{
      fileName: 'merged.pdf',
      pageCount: PAGE_COUNT
    }],
    specialties: [{ specialty: 'general' }],
    patientInfo: {
      name: {
        firstName: 'John',
        lastName: 'Doe',
        middleName: ''
      },
      dateOfBirth: ''
    },
    lineOfBusiness: 'WC',
    uploadedBy: ''
  })
});
const result = await response.json();
JSON.stringify({ status: response.status, body: result });
```

### 8. Verify

Refresh the page and screenshot to confirm the case appears:
```javascript
window.location.reload();
```

## Common Issues

| Issue | Solution |
|-------|----------|
| `navigate` tool doesn't work | Use `window.location.href` via JS instead |
| Page requires Tailscale | Ensure exit node is set to dev before starting |
| Cross-account S3 copy denied | Download to /tmp first, then upload from /tmp (don't try direct cross-account copy) |
| Case ID already exists in destination | Use a different `New Case ID` |
| No merged PDF in source | Check `upload/` prefix instead of `merged/` — the case may not have been processed yet |
| Chrome extension popup blocks interaction | Create a new tab and retry |
