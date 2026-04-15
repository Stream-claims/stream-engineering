# Customers & Multi-Tenancy

## Architecture

Each customer gets an isolated CDK stack (`{CustomerName}Stack`) with its own DynamoDB tables, S3 buckets, Lambdas, and Cognito user pool. The customer portal is deployed separately per customer via SST (`{CustomerName}CustomerPortalStack`).

## Customer Types

Set via `customerType` in `sst.config.ts`. Controls portal routing and UI.

- **med_review** (default) -- Full sidebar, all processing modules (`/case`, `/segmentation`, `/review`, etc.)
- **carrier** -- Streamlined interface, restricted to `/carrier/*` routes. Server-side route guards in `+layout.server.ts` enforce access.

Most production customers are `carrier` type. The `med_review` customers that have no `customerType` set (and thus default) include: Sharp, Arrowhead, Simplexam, Ethos, CowboyLegal, MclHealth, ReportPrep, TCS.

## What Varies Per Customer

Each backend stack configures two key boolean flags in `bin/app.ts`:

- **enableGSheet** -- Google Sheets integration for DVI tracking
- **enableEmailIngestion** -- SES-based email ingestion (auto-creates cases from inbound email)

Additional per-customer behavior is controlled by:

- **Feature flags** -- Stored in DynamoDB, fetched via `/v1/feature_flags` API, cached 5 min. Controls which portal modules are visible. Managed at runtime, not in code.
- **Email configs** -- `lib/utils/customer-email-configs.ts` defines per-customer email recipients for case completion notifications.
- **Post-synthesis processing** -- `lib/constructs/post-synthesis-processing.ts` handles customer-specific file submission and daily email schedules.

## Where to Find Current Config

| What | Source of truth |
|---|---|
| Customer list & stack flags | `backend/bin/app.ts` -- search for `create_med_review_stack` calls |
| Customer type (carrier vs med_review) | `customer-portal/sst.config.ts` -- search for `customerType` |
| Feature flags (runtime) | DynamoDB `feature_flags` table per customer stack |
| Email notification recipients | `backend/lib/utils/customer-email-configs.ts` |
| Portal route guards | `customer-portal/src/routes/(authed)/+layout.server.ts` |

## Adding a New Customer

Use the `create-environment` skill, which handles both the backend CDK stack and customer portal deployment end-to-end.
