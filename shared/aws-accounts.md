# AWS Accounts & Conventions

## Account Structure
| Environment | Account ID | Region | Notes |
|---|---|---|---|
| Dev | 804407225882 | us-east-1 | Development and testing |
| Staging | 804407225882 | us-west-2 | Pre-production validation |
| Production | 982131176572 | us-west-2 | Live customer workloads |

## Profile Convention
Most developers have profiles covering `{account} × {access-level}`:
- Accounts: `dev`, `staging`, `production` (or `prod`)
- Access levels: `readonly`, `admin`

Personal profiles are configured in `CLAUDE.local.md` (git-ignored) per repo.

## Resource Naming Conventions
- Lambda: `{customer}-{stage}-{pipeline}-{function}` (e.g., `Arrowhead-production-Ingest-DocumentSplitter`)
- Step Function: `{customer}-{pipeline}-{stage}-SF` (e.g., `Sharp-PostSynth-production-SF`)
- Queue: `{stackId}_{queue}Queue_{stage}`
- S3 bucket (masterbucket): `{stackname}-masterbucket{suffix}-{hash}`
- DynamoDB table: `{customer}-table-{env}`
- CDK stack: `{Customer}Stack` (e.g., `ArrowheadStack`, `SharpStack`)
- Dev stack: `{Name}DevStack` (e.g., `EilamDevStack`)
- Data lake bucket: `data-lake-{stage}-stream-claims`

## Stage Values
- Production: `'production'` (NOT `'prod'`)
- Staging: `'staging'`
- Dev: `'dev'`

## Safety Rules
- Production profiles: read-only unless explicitly authorized
- NEVER deploy infra to production or staging
- Dev deploys only via `gh` CLI
- Always run `aws sts get-caller-identity` before AWS commands to verify account
- DynamoDB: NEVER do unlimited table scans — use `query` with partition key or `scan --max-items 100`

## OpenSearch
- Dev: OpenSearch Service (managed ES), auth service = `es`
- Staging/Prod: OpenSearch Serverless (AOSS), auth service = `aoss`

## Network
- Dev environments require Tailscale exit node set to dev
- VPN required for direct Redshift connections (Data API works without)
