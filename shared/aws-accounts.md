# AWS Accounts & Conventions

## Accounts

| Environment | Account ID | Region |
|---|---|---|
| Dev | 804407225882 | us-east-1 |
| Staging | 804407225882 | us-west-2 |
| Production | 982131176572 | us-west-2 |
| Management | 211529756017 | us-west-2 |

Source of truth: `cdk.json` → `context.environments`.

## Stage Values

Use these exact strings everywhere (env vars, resource names, CDK context):
- `'production'` (never `'prod'`)
- `'staging'`
- `'dev'`

## Resource Naming

| Resource | Pattern | Example |
|---|---|---|
| CDK stack | `{Customer}Stack` | `ArrowheadStack` |
| Dev stack | `{Name}DevStack` | `EilamDevStack` |
| Lambda | `{customer}-{stage}-{pipeline}-{function}` | `Arrowhead-production-Ingest-DocumentSplitter` |
| Step Function | `{customer}-{pipeline}-{stage}-SF` | `Sharp-PostSynth-production-SF` |
| DynamoDB table | `{customer}-table-{env}` | `Arrowhead-table-production` |
| S3 masterbucket | `{stackname}-masterbucket{suffix}-{hash}` | |
| Data lake bucket | `data-lake-{stage}-stream-claims` | |
| Queue | `{stackId}_{queue}Queue_{stage}` | |

## Safety Rules

- Always run `aws sts get-caller-identity` before AWS commands to verify account
- Production profiles: read-only unless explicitly authorized
- NEVER deploy infra to production or staging
- Dev deploys only via `gh` CLI
- DynamoDB: NEVER do unlimited table scans -- use `query` with partition key or `scan --max-items 100`

## OpenSearch

- Dev: OpenSearch Service (managed), auth service = `es`
- Staging/Prod: OpenSearch Serverless (AOSS), auth service = `aoss`

## Network

- Dev environments require Tailscale exit node set to dev
