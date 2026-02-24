# Backend Context — Stream Claims

## Overview

The Stream backend is a TypeScript AWS CDK app (infra definition) in `bin/` + `lib/`, with Python Lambdas (mostly containerized) in `lambdas/`, and shared Python packages under `packages/` and `ml/`.

---

## Common Commands

### TypeScript / CDK

```bash
yarn          # install dependencies
yarn build    # compile TypeScript
yarn watch    # watch mode
yarn test     # run TS tests
```

### Python

```bash
source .venv/bin/activate          # activate virtualenv
uv pip install <package>           # install packages
ruff check <path>                  # lint changed files only
ruff format <path>                 # format changed files only
mypy --config-file=pyproject.toml  # type checking
pytest                             # run tests (pattern: <filename_base>_test.py)
pre-commit run --files <path1> <path2>
```

### CDK (HUMAN ONLY)

```bash
yarn cdk diff   --profile dev --context stage=dev <StackName>
yarn cdk deploy --profile dev --context stage=dev <StackName>
```

**CRITICAL:**
- NEVER deploy infra to production or staging. Dev deploys only.
- NEVER run `yarn cdk` commands unless explicitly requested IN ALL CAPS.

---

## Code Style

- Readability is the #1 priority. Longer functions are fine when there is no natural seam.
- Root cause analysis over quick fixes. Never patch tests without understanding the issue.
- Functional style: avoid global variables and mid-function data fetching. Fetch data before passing it to functions.
- No `getattr` / `hasattr` / `setattr` — use proper attributes.
- No `TYPE_CHECKING` blocks — alert the user to circular imports instead.
- All imports at the top of the file.
- Factor patterns used 3+ times. Use ABC for code reuse, Protocol for pure interfaces.
- Avoid premature abstractions.
- Ask before adding backwards compatibility code. Default answer: no.

---

## Milestone Checks

Run at every development milestone:

1. `mypy` strict mode
2. `ruff check` on changed files
3. `pytest` for affected modules
4. Review documentation for consistency

---

## Git & Safety

- Use `git mv` for moves/renames, `git rm --cached` for deletions.
- Never permanently delete files created by others without asking.
- Never run linters on the entire codebase — only changed files.
- Production profiles are read-only unless explicitly authorized.

---

## Architecture Overview

### CDK Entrypoint

- App entry: `bin/app.ts`, context in `cdk.json`
- Stage from context: `--context stage=<dev|staging|production|management>`

### Core Stacks

| File | Purpose |
|---|---|
| `lib/account-stack.ts` | Account-level resources (GitHub Actions OIDC) |
| `lib/claims-stack.ts` | Shared networking (VPC) and search infrastructure |
| `lib/admin-stack.ts` | Admin tooling |
| `lib/vpn-stack.ts` | VPN access |
| `lib/waf_stack*.ts` | WAF / security |

**claims-stack search infrastructure:**
- Dev: OpenSearch Service domain
- Staging/Prod: OpenSearch Serverless (AOSS) with VPC endpoint

### Customer Stack (`lib/med_review_stack.ts`)

Main per-customer application stack. Provisions:

- S3 master bucket + DynamoDB tables
- Customer Portal (Cognito + API Gateway + Lambda)
- Ingestion Step Function (OCR pipeline)
- Extract + Synthesise workers (SQS-triggered)
- PDF generation pipeline
- Post-synthesis processing
- Sift / deduplication
- OpenSearch loader (segment search / RAG)

---

## Pipeline Orchestrations

Five Step Functions drive the core processing:

### 1. Ingestion — `{customer}-Ingestion-{stage}-SF`

Trigger: file upload. Duration: 5–30 min.

Flow: expand ZIP → convert to PDF → split → OCR → merge → presegment → auto-segment → submit segments.

### 2. Post-Synthesis — `{customer}-PostSynth-{stage}-SF`

Trigger: after synthesis. Duration: 2–10 min.

Flow: provider standardization → rehab processing (if enabled) → deduplication → proofreading → status update.

### 3. Sift Processing — `{customer}-SiftProcessing-{stage}-SF`

Trigger: post-synthesis or API. Duration: 1–5 min.

Flow: irrelevant detection → dedupe grouping → dedupe processing.

### 4. PDF Output — `{customer}-PdfOutput-{stage}-SF`

Trigger: generate report. Duration: 2–15 min.

Flow: insights extraction → segment PDFs → case PDF → case summary → file submission (customer-specific).

### 5. OpenSearch Loader — `{customer}-OpenSearchLoader-{stage}-SF`

Trigger: manual or after processing. Duration: 1–10 min.

Flow: get segments → chunk → embed → index to OpenSearch.

---

## Naming Conventions

| Resource | Pattern | Example |
|---|---|---|
| Lambda | `{customer}-{stage}-{pipeline}-{function}` | `Arrowhead-production-Ingest-DocumentSplitter` |
| Step Function | `{customer}-{pipeline}-{stage}-SF` | |
| Queue | `{stackId}_{queue}Queue_{stage}` | |

---

## Technology Stack

| Layer | Technology |
|---|---|
| Runtime | Python 3.12 |
| Infra | AWS CDK (TypeScript) |
| Orchestration | Step Functions, SQS |
| OCR | Azure Document Intelligence |
| AI | OpenAI (embeddings / vision) |
| Search | OpenSearch Serverless |
| Database | DynamoDB, S3 |
| Observability | Sentry, LangSmith |

---

## Cross-References

- AWS accounts and regions: `shared/aws-accounts.md`
- Case and segment status flows: `shared/case-lifecycle.md`
- Customer deployment matrix: `shared/customers.md`
- System architecture overview: `shared/architecture.md`

---

## Personal Configuration

Team members create `CLAUDE.local.md` in the backend repo root (git-ignored) for personal AWS profiles and paths. See `.claude/README.md` for setup.
