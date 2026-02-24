# Data Engineering — Claude Context

## Overview

Daily ECS Fargate cron job that loads DynamoDB CDC data and OCR page text into a Redshift data warehouse, then runs dbt transformations.

- **Repo**: `Stream-claims/data-engineering`
- **Runtime**: Python on ECS Fargate
- **Infrastructure**: AWS CDK (TypeScript)

## Setup & Commands

```bash
pip install -r requirements.txt
```

**dbt profiles** — stored at `~/.dbt/profiles.yml`. Get values from AWS Secret `DailyRedshiftLoaderSecret{staging|prod}`.

```bash
# Run a specific model
dbt run --profile data_warehouse --target <environment> --select <model_name>

# Debug a model
dbt run --profile data_warehouse --target staging --select review_segment --debug
```

## Pipeline Architecture

The ECS task runs three sequential stages:

### 1. CaseLoader
- Reads DynamoDB CDC export files from S3 `landing-zone/dynamodb_v2/{customer}/`
- Bulk-loads into Redshift via COPY manifest
- Target table: `raw_data.dynamo_cdc_{customer}` (append-only CDC log)

### 2. PageTextLoader
- Reads S3 inventory manifests from `landing-zone/s3/{customer}/{bucket}/CasePageTextInventory/`
- Discovers page JSON files at `merged/{case}/pages/page_N.json`
- Loads into `raw_data.ocr_page_text` using Redshift Spectrum external tables
- Uses a thread pool of 10 workers

### 3. DbtRunner
- Runs all dbt models in `/app/dbt/data_warehouse` against the target environment

## Infrastructure

- **CDK stack**: `DailyLoaderTaskStack` — defined in `lib/cdk-stack.ts`
- **Fargate task**: 1024 CPU, 8 GB memory (prod) / 1 GB (staging)
- **Schedule**: EventBridge cron, every hour from 16:00–03:00 UTC (8 AM–7 PM PST)
- **S3 bucket**: `data-lake-{stage}-stream-claims`
- **Secret**: `DailyRedshiftLoaderSecret{stage}`

## Redshift Environments

| Environment | Endpoint |
|---|---|
| Production | `data-warehouse-workgroup.982131176572.us-west-2.redshift-serverless.amazonaws.com` |
| Staging | `warehouse-staging.804407225882.us-west-2.redshift-serverless.amazonaws.com` |

VPN required for direct connections. Redshift Data API works without VPN.

## dbt Models

All models live in the `staging` schema and are materialized as `table` unless noted.

| Model | Description |
|---|---|
| `all_case_data` | Union of all customer `dynamo_cdc_*` tables; extracts case_id, sk, status, timestamps |
| `case_upload` | Case-level upload records |
| `segment_state` | Segment state source of truth |
| `extract_segment` | Pre-extract review LLM outputs |
| `model_output_segment` | Synthesized model output |
| `review_segment` | Human-editable review data |
| `segmentation_segment` | Segmentation records |
| `page_text` | OCR text per page: case_id, page_number, text, handwritten, confidence |
| `insights` | Timeline events and claim information |
| `review_summary` | Review summaries |
| `deduplication` | Deduplication tracking |
| `langsmith_traces` | LangSmith traces — materialized as **view** |

## Raw Table Schemas

```sql
-- DynamoDB CDC log (one table per customer)
raw_data.dynamo_cdc_{customer} (
    id                BIGINT IDENTITY,
    json_data         SUPER,
    load_manifest_file VARCHAR,
    load_timestamp    TIMESTAMP
)

-- OCR page text
raw_data.ocr_page_text (
    id               BIGINT IDENTITY,
    json_data        SUPER,
    source_file      VARCHAR(1200),
    customer         VARCHAR,
    loaded_timestamp TIMESTAMP
)
```

## Redshift Gotchas

- **SUPER columns**: always use `JSON_EXTRACT_PATH_TEXT(JSON_SERIALIZE(column), 'field', 'S')` — never cast with `::json`
- **NULL SUPER**: guard with `COALESCE(column, JSON_PARSE('{}'))`
- **Workgroup names** differ between staging and prod — double-check before running queries
- **VARCHAR vs SUPER**: know which type you are working with before writing extraction logic
- **Regex**: Redshift requires double-escaping regex patterns
- **Spectrum views**: must be created `WITH NO SCHEMA BINDING`
- **SSO tokens expire**: if queries start failing unexpectedly, reconnect

## Cross-References

- AWS accounts and regions: `shared/aws-accounts.md`
- Case and segment status flows: `shared/case-lifecycle.md`
- Customer deployment matrix: `shared/customers.md`
