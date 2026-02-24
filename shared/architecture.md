# System Architecture

Stream Claims is an AI-powered medical record review platform with three main components:

## Data Flow
1. **Customer Portal** (SvelteKit) — Users upload medical record PDFs
2. **Backend** (Python Lambdas + CDK) — Processes documents through OCR, segmentation, extraction, synthesis, and PDF output pipelines
3. **Data Engineering** (ECS + dbt) — Loads DynamoDB CDC data and OCR text into Redshift warehouse for analytics

## Technology Stack
| Component | Technology |
|---|---|
| Frontend | SvelteKit 5, TypeScript, Tailwind, Flowbite |
| Backend Runtime | Python 3.12, Docker Lambda images |
| Infrastructure | AWS CDK (TypeScript) |
| Orchestration | Step Functions, SQS |
| OCR | Azure Document Intelligence |
| LLM Providers | OpenAI, Anthropic, Google |
| Embeddings | OpenAI text-embedding-3-large (3072 dim) |
| Vector Search | OpenSearch Serverless |
| Database | DynamoDB (single-table design) |
| File Storage | S3 (per-customer buckets) |
| Data Warehouse | Redshift Serverless |
| ETL | dbt, ECS Fargate cron |
| Auth | AWS Cognito |
| Monitoring | Sentry, CloudWatch, LangSmith |
| Deployment | SST v2 (portal), CDK (backend), CDK (data-eng) |

## Pipeline Overview
```
Upload → Ingestion SF → Extract (SQS) → Synthesise (SQS) → Post-Synthesis SF → PDF Output SF
                                                                    ↓
                                                            Sift/Dedup SF
                                     Extract → OpenSearch Loader SF (RAG indexing)

DynamoDB CDC → S3 Landing Zone → ECS Loader → Redshift → dbt models
```

## Repository Map
| Repo | GitHub | Purpose |
|---|---|---|
| backend | Stream-claims/backend | CDK infra + Python Lambdas + ML packages |
| customer-portal | Stream-claims/customer-portal | SvelteKit multi-tenant portal |
| data-engineering | Stream-claims/data-engineering | ECS loader + dbt transformations |
