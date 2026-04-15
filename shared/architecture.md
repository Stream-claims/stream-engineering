# System Architecture

AI-powered medical record review platform. Insurance professionals upload case documents; the system OCRs, segments, extracts, synthesizes, and produces structured outputs.

## Repositories

| Repo | Stack | Purpose |
|---|---|---|
| `backend` | Python Lambdas, CDK (TypeScript) | Document processing pipeline + portal API (~40 lambdas, ~60 shared packages) |
| `customer-portal` | SvelteKit, SST | Multi-tenant upload/review portal |
| `data-engineering` | ECS, dbt, Redshift | CDC ingestion + analytics warehouse |

Each repo has its own `CLAUDE.md` with commands, patterns, and architecture details.

## Core Infrastructure

- **Compute**: Lambda (backend), ECS Fargate (data-eng batch jobs)
- **Storage**: DynamoDB (single-table design), S3 (per-customer buckets), OpenSearch (vector search / RAG)
- **Orchestration**: Step Functions, SQS
- **Data warehouse**: Redshift Serverless + dbt models
- **Auth**: Cognito
- **Infra-as-code**: CDK (backend, data-eng), SST (portal)

## Pipeline Flow

```
Upload → Ingestion SF → Extract (SQS) → Synthesise (SQS) → Post-Synthesis SF → PDF Output SF
                                                                    ↓
                                                              Sift/Dedup SF
                                       Extract → OpenSearch Loader SF (RAG)

DynamoDB CDC → S3 → ECS Loader → Redshift → dbt models
```

## Domain Terms

A **case** is one insurance claim with uploaded documents. A **segment** is one constituent document within a case (e.g., a single medical record, one billing statement) — never a grouping of multiple docs.
