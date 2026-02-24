---
name: redshift-warehouse
description: Query Redshift data warehouse, debug dbt models, and troubleshoot ETL pipeline issues. Use when the user asks to query staging tables, debug data quality issues, inspect raw data, or understand the data warehouse schema.
---

# Redshift Data Warehouse Operations Skill

## When to Use This Skill

Activate this skill when the user:
- Wants to query Redshift staging or raw tables
- Needs to debug dbt model failures or data quality issues
- Asks about the data warehouse schema
- Wants to inspect loaded data or verify ETL results
- Needs to troubleshoot missing or incorrect data
- Asks how to query case or segment data from Redshift

## Environments

### Production (Account: 982131176572)
```
Host: data-warehouse-workgroup.982131176572.us-west-2.redshift-serverless.amazonaws.com
Database: warehouse
Port: 5439
Secret: DailyRedshiftLoaderSecretproduction
```

### Staging (Account: 804407225882)
```
Host: warehouse-staging.804407225882.us-west-2.redshift-serverless.amazonaws.com
Database: warehouse
Port: 5439
Secret: DailyRedshiftLoaderSecretstagingV2
```

**Note:** Staging and production are in different AWS accounts. Use the correct profile for each.

## AWS Profiles & Credentials

**Profile Naming Convention:** `{username}-{account}-{tier}`
- **account**: `dev`, `staging`, `prod`
- **tier**: `readonly` (safe queries), `poweruser` (writes), `admin` (full access)
- Example: `jane-prod-readonly`, `bob-staging-poweruser`

**Profile Discovery:**
1. Check if user has configured AWS profile in `CLAUDE.local.md` or `~/.claude/CLAUDE.md`
2. If not found, ask for their profile name
3. Use profile to fetch Redshift credentials from Secrets Manager

**VPN Requirement:** Direct connections (psql, Python redshift_connector) require VPN. The Redshift Data API works without VPN.

## Key Schema Patterns

### Staging Tables (warehouse.staging)

**Core Analytics Tables:**
- `all_case_data` - Unified case data across all customers (UNION ALL of CDC tables)
- `insights` - Case insights with extracted claim information
- `segment_state` - Segment classification state (predictions, status)
- `review_segment` - Human review data (ground truth labels)
- `model_output_segment` - LLM-generated summaries
- `extract_segment` - Structured extraction outputs
- `page_text` - OCR page text data
- `langsmith_traces` - LLM trace data via Spectrum (view, not table)
- `deduplication` - Deduplication results
- `case_upload` - Case upload tracking
- `segmentation_segment` - Segmentation outputs

**Critical Pattern - Current Record Filtering:**

**Option 1: Use `is_current` flag (recommended, now reliable):**
```sql
SELECT *
FROM warehouse.staging.segment_state
WHERE is_current = TRUE
```

**Option 2: ROW_NUMBER pattern (if you need custom ordering):**
```sql
WITH proper_current AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY case_id, segment_id
            ORDER BY COALESCE(updated_at, created_at, cdc_timestamp) DESC
        ) = 1 as true_is_current
    FROM warehouse.staging.segment_state
)
SELECT * FROM proper_current WHERE true_is_current = TRUE
```

**Note:** The `is_current` flag was previously unreliable due to CDC timing issues, but has been fixed. Use whichever pattern fits your needs.

### Raw Tables (warehouse.raw_data)

**DynamoDB CDC Tables:**
- `dynamo_cdc_{customer}` - Per-customer CDC tables (SUPER column: `json_data`)
- Contains DynamoDB JSON format with type descriptors

**S3 Data Tables:**
- `ocr_page_text` - Page text from S3 inventories
- `kinesis_segment_data_{customer}` - Segment data from Kinesis

**Note:** Customer names are snake_case (e.g., `berkley_ent`, not `BerkleyEnt`)

### Spectrum External Tables (warehouse.spectrum_schema)

Spectrum allows querying S3 data directly without loading into Redshift.

**Architecture:**
```
S3 Parquet files → AWS Glue Data Catalog → Redshift Spectrum → Query results
                   (database: warehouse)   (schema: spectrum_schema)
```

**Available Tables:**
- `langsmith_traces` - LangSmith trace data (LLM observability)

**Cost Model:** ~$5 per TB scanned. Always filter by date when possible.

**Key Functions:**
- `CAN_JSON_PARSE(column)` - Returns TRUE if JSON is valid (use to handle truncated data)
- `JSON_PARSE(column)` - Converts VARCHAR to SUPER type for path navigation

**View Requirement:** Views on Spectrum tables need `WITH NO SCHEMA BINDING` (dbt: `bind=False`)

## Querying SUPER Columns

SUPER columns from DynamoDB use special JSON format with type descriptors:
- `"S"` = String
- `"M"` = Map
- `"L"` = List
- `"N"` = Number
- `"BOOL"` = Boolean

**❌ WRONG - Standard JSON won't work:**
```sql
SELECT scenario_resolution.decision FROM segment_state
-- Returns NULL
```

**✅ CORRECT - Use JSON_SERIALIZE + JSON_EXTRACT_PATH_TEXT:**
```sql
SELECT JSON_EXTRACT_PATH_TEXT(
    JSON_SERIALIZE(scenario_resolution),
    'decision', 'S'  -- Extract 'decision.S' from DynamoDB format
) as decision
FROM warehouse.staging.segment_state
```

**✅ BEST - Fetch entire column, parse in Python:**
```python
# Use ml/util.py helper (if available) or parse manually
from ml.util import parse_scenario_resolution
import pandas as pd

query = "SELECT scenario_resolution FROM warehouse.staging.segment_state LIMIT 100"
df = pd.read_sql_query(query, conn)
parsed = df['scenario_resolution'].apply(parse_scenario_resolution)
df['decision'] = parsed.apply(lambda x: x.get('decision'))
```

## Handling NULL SUPER Columns

**Problem:** `JSON_SERIALIZE(null_column)` throws "invalid json object null"

**Solution:** Wrap with `COALESCE` + `JSON_PARSE('{}')`:
```sql
-- ❌ FAILS when scenario_resolution is NULL (e.g., from LEFT JOIN)
JSON_EXTRACT_PATH_TEXT(JSON_SERIALIZE(scenario_resolution), 'field', 'S')

-- ✅ CORRECT - handles NULL gracefully
JSON_EXTRACT_PATH_TEXT(
    JSON_SERIALIZE(COALESCE(scenario_resolution, JSON_PARSE('{}'))),
    'field', 'S'
)
```

**Problem:** Missing keys return NULL, and `NULL != 'value'` returns NULL (not TRUE)

**Solution:** Wrap extracted value in `COALESCE` for comparisons:
```sql
-- ❌ WRONG - rows with missing 'ehr_type' key are excluded (NULL != 'epic' = NULL)
WHERE JSON_EXTRACT_PATH_TEXT(..., 'ehr_type', 'S') != 'epic'

-- ✅ CORRECT - missing keys become empty string, comparison works
WHERE COALESCE(JSON_EXTRACT_PATH_TEXT(..., 'ehr_type', 'S'), '') != 'epic'
```

**Note:** `'{}'::super` does NOT work. Always use `JSON_PARSE('{}')` to create empty SUPER objects.

## scenario_resolution v3.0 Profiler Fields

Recent segments (Nov 2025+) include document profiler metadata:

| Field | Type | Description |
|-------|------|-------------|
| `doc_title_coarse` | S | High-level document type (e.g., "Medical evaluation report") |
| `doc_title_granular` | S | Specific document name (e.g., "DWC Form RFA") |
| `author_type_coarse` | S | Author category (e.g., "Medical provider", "Legal representative") |
| `author_type_granular` | S | Specific author role (e.g., "Treating physician", "Radiologist") |
| `form_standardization` | S | Form-ness: "high", "medium", or "low" |
| `is_correspondence` | BOOL | Whether document is correspondence |
| `is_empty_content` | BOOL | Whether document has empty content |

**Example query:**
```sql
SELECT
    case_id,
    segment_id,
    JSON_EXTRACT_PATH_TEXT(JSON_SERIALIZE(COALESCE(scenario_resolution, JSON_PARSE('{}'))), 'doc_title_coarse', 'S') as doc_title_coarse,
    JSON_EXTRACT_PATH_TEXT(JSON_SERIALIZE(COALESCE(scenario_resolution, JSON_PARSE('{}'))), 'author_type_coarse', 'S') as author_type_coarse,
    JSON_EXTRACT_PATH_TEXT(JSON_SERIALIZE(COALESCE(scenario_resolution, JSON_PARSE('{}'))), 'form_standardization', 'S') as form_standardization
FROM warehouse.staging.segment_state
WHERE is_current = TRUE
  AND created_at >= '2025-11-01'::date  -- v3.0 data
```

## Quick Discovery Queries

Use these to quickly understand what exists and its current state:

### List All LangSmith Views
```sql
SELECT table_name, table_type
FROM information_schema.tables
WHERE table_schema = 'staging'
  AND table_name LIKE 'langsmith%'
ORDER BY table_name;
```

**Expected output:**
| table_name | table_type |
|------------|------------|
| langsmith_clf_candidate_gen | VIEW |
| langsmith_clf_disambiguation | VIEW |
| langsmith_traces | VIEW |

### Understand Table Columns Quickly
```sql
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_schema = 'staging'
  AND table_name = 'langsmith_clf_disambiguation'
ORDER BY ordinal_position;
```

### Check Data Volume & Freshness
```sql
-- LangSmith tables (by environment)
SELECT
    environment,
    COUNT(*) as total_rows,
    MIN(start_time) as earliest_ns,
    MAX(start_time) as latest_ns,
    -- Convert nanoseconds to timestamp for readability
    TIMESTAMP 'epoch' + MAX(start_time)/1000000000 * INTERVAL '1 second' as latest_datetime
FROM staging.langsmith_traces
GROUP BY environment;

-- Classification views
SELECT
    'disambiguation' as view_name,
    COUNT(*) as rows,
    COUNT(DISTINCT segment_type_decision) as unique_decisions
FROM staging.langsmith_clf_disambiguation
WHERE environment = 'production'
UNION ALL
SELECT
    'candidate_gen' as view_name,
    COUNT(*) as rows,
    COUNT(DISTINCT candidates_json) as unique_candidate_sets
FROM staging.langsmith_clf_candidate_gen
WHERE environment = 'production';
```

### Preview Parsed Fields (Sanity Check)
```sql
-- Disambiguation: check decisions are being extracted
SELECT segment_type_decision, COUNT(*) as cnt
FROM staging.langsmith_clf_disambiguation
WHERE environment = 'production'
  AND segment_type_decision IS NOT NULL
GROUP BY 1 ORDER BY 2 DESC LIMIT 10;

-- Candidate gen: check min/max parsing works
SELECT min_candidates, max_candidates, COUNT(*) as cnt
FROM staging.langsmith_clf_candidate_gen
WHERE environment = 'production'
GROUP BY 1, 2 ORDER BY 3 DESC LIMIT 10;
```

### Check for NULL Parsed Fields (Data Quality)
```sql
-- If these return high counts, parsing may be broken
SELECT
    COUNT(*) as total,
    SUM(CASE WHEN segment_type_decision IS NULL THEN 1 ELSE 0 END) as null_decisions,
    SUM(CASE WHEN candidate_classes IS NULL THEN 1 ELSE 0 END) as null_candidates
FROM staging.langsmith_clf_disambiguation
WHERE environment = 'production';
```

## Common Query Patterns

### Find Recent Cases
```sql
SELECT
    customer_name,
    case_id,
    status,
    created_at,
    updated_at
FROM warehouse.staging.all_case_data
WHERE is_deleted = FALSE
ORDER BY created_at DESC
LIMIT 100;
```

### Get Current Segment State
```sql
SELECT
    customer_name,
    case_id,
    segment_id,
    start_page,
    end_page,
    JSON_EXTRACT_PATH_TEXT(JSON_SERIALIZE(scenario_resolution), 'decision', 'S') as predicted_class,
    created_at
FROM warehouse.staging.segment_state
WHERE is_current = TRUE
ORDER BY created_at DESC
LIMIT 100;
```

### Join Predictions with Ground Truth
```sql
SELECT
    ss.customer_name,
    ss.case_id,
    ss.segment_id,
    JSON_EXTRACT_PATH_TEXT(JSON_SERIALIZE(ss.scenario_resolution), 'decision', 'S') as predicted_class,
    rs.segment_class as true_class,
    CASE
        WHEN JSON_EXTRACT_PATH_TEXT(JSON_SERIALIZE(ss.scenario_resolution), 'decision', 'S') = rs.segment_class
        THEN TRUE
        ELSE FALSE
    END as is_correct
FROM warehouse.staging.segment_state ss
JOIN warehouse.staging.review_segment rs ON ss.segment_id = rs.segment_id
WHERE ss.is_current = TRUE
  AND rs.is_current = TRUE
  AND rs.is_review_complete = TRUE
LIMIT 100;
```

### List Available Customers
```sql
-- From staging tables
SELECT DISTINCT customer_name
FROM warehouse.staging.all_case_data
ORDER BY customer_name;

-- From raw CDC tables
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'raw_data'
  AND table_name LIKE 'dynamo_cdc_%'
ORDER BY table_name;
```

### Check Data Freshness
```sql
SELECT
    customer_name,
    MAX(created_at) as latest_case,
    COUNT(*) as total_cases
FROM warehouse.staging.all_case_data
WHERE is_deleted = FALSE
GROUP BY customer_name
ORDER BY latest_case DESC;
```

### LangSmith Traces

For detailed LangSmith query patterns, derived views schemas, and truncation handling, see the **`langsmith-fetching`** skill.

Quick reference - available views:
- `staging.langsmith_traces` - Base traces (Spectrum view)
- `staging.langsmith_clf_disambiguation` - DisambiguationClf outputs
- `staging.langsmith_clf_candidate_gen` - LlmCandidateGen outputs

## Debugging Workflow

### 1. Verify Data Exists in Raw Tables
```sql
-- Check CDC data loaded
SELECT COUNT(*), MAX(load_timestamp)
FROM warehouse.raw_data.dynamo_cdc_berkley_ent;

-- Check page text loaded
SELECT COUNT(*), MAX(loaded_timestamp)
FROM warehouse.raw_data.ocr_page_text
WHERE customer = 'berkley_ent';
```

### 2. Check dbt Transformations
```sql
-- Verify staging table updated
SELECT COUNT(*), MAX(cdc_timestamp)
FROM warehouse.staging.all_case_data
WHERE customer_name = 'berkley_ent';
```

### 3. Investigate Data Quality Issues
```sql
-- Find records with NULL critical fields
SELECT customer_name, case_id, status
FROM warehouse.staging.all_case_data
WHERE case_id IS NULL OR status IS NULL
LIMIT 100;

-- Check for duplicate records (indicates is_current filtering issues)
SELECT case_id, segment_id, COUNT(*)
FROM warehouse.staging.segment_state
WHERE is_current = TRUE  -- This may show duplicates
GROUP BY case_id, segment_id
HAVING COUNT(*) > 1;
```

## Connection Methods

**VPN Required:** Options 1-3 require VPN connection. Option 4 (Data API) works without VPN.

### Option 1: AWS CLI + psql (Quick Queries)
```bash
# Get credentials from Secrets Manager
# Use your prod-readonly profile for safe read operations
aws secretsmanager get-secret-value \
  --secret-id DailyRedshiftLoaderSecretproduction \
  --profile <your-prod-readonly-profile> \
  --query SecretString --output text | jq

# Connect with psql (requires VPN)
PGPASSWORD='...' psql \
  -h data-warehouse-workgroup.982131176572.us-west-2.redshift-serverless.amazonaws.com \
  -U user \
  -d warehouse \
  -p 5439
```

### Option 2: Python with redshift_connector
```python
import boto3
import json
import redshift_connector

# Fetch credentials - use your prod-readonly profile
sm = boto3.client('secretsmanager', region_name='us-west-2', profile_name='<your-prod-readonly-profile>')
secret = sm.get_secret_value(SecretId='DailyRedshiftLoaderSecretproduction')
creds = json.loads(secret['SecretString'])

# Connect
conn = redshift_connector.connect(
    host=creds['host'],
    database=creds['dbname'],
    port=int(creds['port']),
    user=creds['user'],
    password=creds['password']
)

# Query
cursor = conn.cursor()
cursor.execute("SELECT COUNT(*) FROM warehouse.staging.all_case_data")
result = cursor.fetchone()
print(f"Total cases: {result[0]}")

cursor.close()
conn.close()
```

### Option 3: Use ml/util.py Helpers (If Available)
```python
from ml.util import get_redshift_secret, get_redshift_connection, RedshiftCredentials
import pandas as pd

# Get credentials - use your prod-readonly profile
secret_dict = get_redshift_secret("<your-prod-readonly-profile>")
creds = RedshiftCredentials(
    host="data-warehouse-workgroup.982131176572.us-west-2.redshift-serverless.amazonaws.com",
    database="warehouse",
    port=5439,
    user=secret_dict["username"],
    password=secret_dict["password"]
)

# Query with pandas
with get_redshift_connection(creds) as conn:
    df = pd.read_sql_query("""
        SELECT customer_name, COUNT(*) as total
        FROM warehouse.staging.all_case_data
        GROUP BY customer_name
    """, conn)

print(df)
```

### Option 4: Redshift Data API (No VPN Required)

Use when you can't connect directly (no VPN, CI/CD, etc.):

```bash
# Execute a query (returns statement ID)
# Note: Data API requires poweruser tier (not readonly) for some operations
aws --profile <your-prod-poweruser-profile> redshift-data execute-statement \
  --workgroup-name data-warehouse-workgroup \
  --database warehouse \
  --sql "SELECT COUNT(*) FROM staging.all_case_data" \
  --region us-west-2

# Check query status
aws --profile <your-prod-poweruser-profile> redshift-data describe-statement \
  --id "statement-id-from-above" \
  --region us-west-2

# Get results (after status = FINISHED)
aws --profile <your-prod-poweruser-profile> redshift-data get-statement-result \
  --id "statement-id-from-above" \
  --region us-west-2
```

**Workgroup names:**
- Production: `data-warehouse-workgroup`
- Staging: `warehouse-staging`

## Key dbt Models

**Materialization:** All staging models are `table` (drop + recreate) except `langsmith_traces` which is a `view`.

**Rebuild Times (Production):** ~19 minutes total. Longest: `segment_state` (~14 min), `review_segment` (~6 min).

**Schedule:** Hourly 8AM-7PM PST (16:00-03:00 UTC)

| Model | Type | Rebuild Time | Notes |
|-------|------|--------------|-------|
| `all_case_data` | table | ~5 min | UNION ALL across customer CDC tables |
| `segment_state` | table | ~14 min | Longest rebuild, ML predictions |
| `review_segment` | table | ~6 min | Human review ground truth |
| `langsmith_traces` | view | instant | Spectrum, no rebuild |
| `insights` | table | ~30 sec | Case insights |
| `page_text` | table | ~17 sec | OCR text |

**all_case_data.sql** - Dynamic UNION ALL across all customer CDC tables
- Uses `get_customer_names()` macro to discover customers
- Extracts core fields from DynamoDB JSON
- Handles missing tables gracefully

**langsmith_traces.sql** - LangSmith trace data via Spectrum
- Materialized as `view` (queries S3 directly)
- Extracts case_id, segment_id, customer_name from JSON
- Handles ~4.5% truncated rows (16KB limit) with `CAN_JSON_PARSE()`

**segment_state.sql**, **review_segment.sql** - Core tables for ML evaluation
- Source of truth for segment classification
- Join via `segment_id` to compare predictions vs. ground truth

## Troubleshooting Common Issues

**"No data in staging tables"**
1. Check raw tables have data
2. Verify dbt ran successfully (check ECS logs)
3. Check dbt model dependencies (may have failed upstream)

**"Duplicate records in queries"**
- `is_current` flag may not be updated (rare, but check dbt run logs)
- If duplicates persist, use ROW_NUMBER pattern as fallback

**"SUPER column returns NULL"**
- Using standard JSON operations instead of JSON_SERIALIZE
- Fix: Use `JSON_EXTRACT_PATH_TEXT(JSON_SERIALIZE(column), 'path', 'type')`

**"Customer table doesn't exist"**
- Customer name formatting (should be snake_case)
- Table may not have been created yet (no data for that customer)
- Check with: `SELECT * FROM information_schema.tables WHERE table_name LIKE 'dynamo_cdc_%'`

## Common Pitfalls (Non-Obvious Gotchas)

These issues are easy to hit and hard to debug without prior knowledge:

### 1. Workgroup Names Are Inconsistent
**WRONG assumptions:**
```bash
# These don't exist:
--workgroup-name data-lake-staging
--workgroup-name warehouse-production
```

**CORRECT names:**
```bash
# Staging (account 804407225882):
--workgroup-name warehouse-staging

# Production (account 982131176572):
--workgroup-name data-warehouse-workgroup
```

### 2. VARCHAR vs SUPER Column Confusion
The `langsmith_traces` view outputs columns as **VARCHAR** (already serialized JSON strings), NOT SUPER.

**WRONG - will error with "function json_serialize(character varying) does not exist":**
```sql
JSON_EXTRACT_PATH_TEXT(JSON_SERIALIZE(outputs), 'output', 'decision')
```

**CORRECT - outputs is already VARCHAR:**
```sql
JSON_EXTRACT_PATH_TEXT(outputs, 'output', 'decision')
```

**How to know:** SUPER columns come from raw DynamoDB tables. Views that extract/transform typically output VARCHAR.

### 3. Redshift Regex Escaping
Redshift regex requires **double-escaping** special characters.

**WRONG - "Missing } in quantified repetition" error:**
```sql
REGEXP_SUBSTR(run_name, '\{([^}]+)\}', 1, 1, 'e')
```

**CORRECT - double-escape braces and brackets:**
```sql
REGEXP_SUBSTR(run_name, '\\{([^}]+)\\}', 1, 1, 'e')  -- literal {
REGEXP_SUBSTR(run_name, '\\[(\\d+),', 1, 1, 'e')     -- literal [
```

### 4. Views on Spectrum Need NO SCHEMA BINDING
Views referencing Spectrum external tables fail without this:

**dbt config:**
```sql
{{ config(materialized='view', bind=False) }}
```

**Raw SQL:**
```sql
CREATE VIEW ... WITH NO SCHEMA BINDING;
```

### 5. LangSmith-Specific Issues
See **`langsmith-fetching`** skill for: data truncation handling, nanoseconds time conversion, Spectrum query performance tips.

### 6. SSO Token Expiration
AWS SSO tokens expire. If queries suddenly fail with auth errors:
```bash
aws sso login --profile <your-{account}-admin-profile>
```

### 7. NULL SUPER Column Handling
`JSON_SERIALIZE` fails on NULL SUPER columns (common with LEFT JOINs).

**WRONG - crashes on NULL:**
```sql
JSON_SERIALIZE(ss.scenario_resolution)  -- fails if NULL from LEFT JOIN
```

**CORRECT - handles NULL:**
```sql
JSON_SERIALIZE(COALESCE(ss.scenario_resolution, JSON_PARSE('{}')))
```

## Best Practices

**Always:**
- Use `is_current = TRUE` for filtering (now reliable)
- Limit queries during exploration (`LIMIT 100`)
- Check data freshness before querying
- Use prepared statements for dynamic queries

**Never:**
- Do unlimited table scans in production
- Use `::json` on SUPER columns (use JSON_SERIALIZE)
- Hardcode customer names in queries
- Assume data is complete without checking timestamps

## Integration with ml-data-fetch-annotate Skill

This skill focuses on **querying and debugging** the data warehouse.

The **ml-data-fetch-annotate skill** focuses on **fetching datasets and creating PDFs** for ML analysis.

**Division of responsibilities:**
- **redshift-warehouse:** Query exploration, schema understanding, data quality checks
- **ml-data-fetch-annotate:** Batch dataset extraction, PDF generation, S3 image retrieval

**When to hand off:**
- User wants to create annotated PDFs → Use ml-data-fetch-annotate
- User wants to fetch images from S3 → Use ml-data-fetch-annotate
- User wants to analyze model errors visually → Use ml-data-fetch-annotate

## Key Commands Reference

```bash
# Get Redshift credentials
# Use readonly tier for safe queries, poweruser for writes
aws secretsmanager get-secret-value \
  --secret-id DailyRedshiftLoaderSecret{stage} \
  --profile <your-{account}-{tier}-profile>

# Check ECS task logs (if pipeline failed)
aws logs tail /aws/ecs/EcsTaskLogs --follow --profile <your-{account}-readonly-profile>

# Run dbt locally
dbt run --profile data_warehouse --target staging
dbt test --profile data_warehouse --target staging
```

## Instructions for Claude

When this skill is activated:

1. **Understand the query goal:**
   - Data exploration? → Write SELECT queries
   - Debugging ETL? → Check raw tables, dbt logs, data freshness
   - Schema questions? → Query information_schema or describe tables
   - Data quality? → Look for NULLs, duplicates, inconsistencies

2. **Always use proper patterns:**
   - `is_current = TRUE` for current records (now reliable)
   - JSON_SERIALIZE for SUPER columns
   - LIMIT clauses for exploration queries
   - Snake_case customer names

3. **Verify assumptions:**
   - Check customer names exist before querying
   - Verify table exists in raw_data before checking staging
   - Check data freshness before complex queries

4. **Provide context:**
   - Explain DynamoDB JSON format when relevant
   - Warn about common pitfalls (SUPER columns, data freshness)
   - Suggest alternative approaches when appropriate

5. **Be pragmatic:**
   - Simple queries are better than complex ones
   - Fetch and parse in Python for complex SUPER operations
   - Use Bash + AWS CLI for quick checks
   - Use Python for multi-step workflows
