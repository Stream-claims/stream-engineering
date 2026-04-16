---
name: test-locally
description: Spin up the customer portal locally for testing. Use when you need to visually verify a change in the browser, test UI behavior, or validate frontend+backend integration locally.
---

# Test Customer Portal Locally

## Decision: Frontend-Only vs Full-Stack

Before starting, determine which mode you need:

**Frontend-only** (remote backend): The change is purely UI — styling, components, layout, client-side logic. The deployed dev backend already has the APIs you need.

**Full-stack** (local backend): The change touches backend code (new endpoints, modified responses, DTO changes) that isn't yet deployed. You need the backend running locally on port 8080.

## Step 1: Read the Developer's CUSTOMER_NAME

```bash
cat .env | grep CUSTOMER_NAME
```

If `.env` is missing or has no `CUSTOMER_NAME`, stop and ask the user what their dev customer name is (e.g., `EilamDev`, `JoeyDev`, `AdiletDev`).

## Step 2: Set `isLocal` in sst.config.ts

Find the developer's entry in the `// Local` switch block of `sst.config.ts` and set `isLocal` based on the mode:

- **Frontend-only** → `isLocal: false` (API calls go to deployed dev backend via HTTPS)
- **Full-stack** → `isLocal: true` (API calls go to `http://localhost:8080`)

`isLocal` controls where the frontend sends API requests. When `false`, requests go to the deployed dev backend. When `true`, requests go to `localhost:8080`. Default to `isLocal: false` — only set `true` if the context makes clear a local backend is running (e.g. the user mentioned it, or you started it yourself).

Search for their CUSTOMER_NAME under the `} else {` / `// Local` block and update accordingly. **Do not touch entries outside the local block.**

> **Revert this change before committing.** The `isLocal` toggle is a local dev convenience — never commit it.

## Step 3: Start the Frontend

```bash
npm run dev
```

This runs `sst bind vite dev` which:
1. Fetches secrets from AWS Secrets Manager (requires valid AWS credentials)
2. Injects them as env vars
3. Starts Vite dev server at `http://localhost:5173`

**Common failures:**
- `ExpiredTokenException` → AWS credentials expired. User needs to re-authenticate.
- `CUSTOMER_NAME must be defined` → `.env` is missing or empty.
- `Cannot find module` → Run `pnpm install`.

## Step 4: Test in Browser

Open `http://localhost:5173` in the browser. Auth flows through AWS Cognito — the user will need to log in via their normal SSO flow.

Use Chrome browser automation tools (if available) or instruct the user to verify manually.

## Step 5: Clean Up

1. **Stop the dev server** (Ctrl+C or kill the process).
2. **Revert only the `isLocal` line** in `sst.config.ts` if you changed it. Use a targeted edit — do not `git checkout` the whole file, as the developer may have other in-progress changes there.

## Quick Reference

| Mode | `isLocal` | Backend | API target |
|------|-----------|---------|------------|
| Frontend-only | `false` | Not needed | `https://{API_DOMAIN}` (deployed dev) |
| Full-stack | `true` | Must be running on :8080 | `http://localhost:8080` |

## Notes

- `IS_LOCAL` env var is derived from `isLocal` in the SST config via `sst bind`. There is no env var override — you must change `sst.config.ts`.
- Only customer names with entries in the `// Local` switch block can run locally. If the developer's name isn't there, they need to add it first.
- AWS credentials for the dev account are required — `sst bind` fetches secrets at startup.
