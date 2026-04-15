---
name: aws-debug
description: Debug AWS infrastructure for the claims processing pipeline. Use when the user asks to debug cases, segments, check DynamoDB state, inspect S3 buckets, trace Lambda execution, or investigate CloudFormation stacks.
---

# AWS Infrastructure Debugging Skill

## When to Use This Skill

Invoke this skill autonomously when the user:
- Asks to debug a specific case or segment (e.g., "Debug case XYZ", "Check segment ABC")
- Wants to inspect DynamoDB items or query the database
- Needs to check S3 bucket contents or file structure
- Wants to trace Lambda execution or check logs
- Asks about deployed infrastructure state
- Needs to understand resource relationships

## Critical Safety Rules

- **NEVER do unlimited table scans** - Always use `query` with partition key when possible, or add `--max-items 100` to `scan` commands (production tables have millions of items)
- Production queries should be **read-only** unless explicitly authorized
- Use `aws sts get-caller-identity` to verify which profile/account you're using before running commands

## AWS Profile Guidelines

**Profile Discovery Strategy:**

1. **First**, check if the user has configured profiles in `CLAUDE.local.md`
2. **If not found**, explicitly ask the user for their AWS profile with clear context:
   - Specify which environment you need: production, staging, or dev
   - Ask whether they want **readonly access** (safe, recommended for debugging) or **write access** (requires explicit approval)
   - Example: "I need to query the arrowhead production DynamoDB table. Do you have a readonly profile I should use?"

**Profile Dimensions:** Most devs have profiles covering `{account}` × `{access-level}`:
- **Accounts**: `dev`, `staging`, `prod`
- **Access levels**: `readonly` (safe queries), `poweruser` (writes), `admin` (full access)

Profile naming conventions vary by person - check `CLAUDE.local.md` or ask the user for their profile names.

**CRITICAL**: Never perform write operations without explicit approval. Production profiles should only be used for read-only operations unless explicitly authorized.

**Quick profile discovery:**
```bash
aws configure list-profiles | grep -i <keyword>  # e.g., grep -i dev
```

## AWS Account Structure

- **Dev Account**: 804407225882 (Region: us-east-1)
- **Staging Account**: 804407225882 (Region: us-west-2)
- **Production Account**: 982131176572 (Region: us-west-2)

## DynamoDB Schema & Query Patterns

### Table Naming
- Format: `<customer>-table-<env>`
- Example: `acadia-table-production`
- Legacy format `<Customer>Stack*` is defunct

### Case ID Format

**IMPORTANT:** Case IDs are arbitrary strings that vary by customer. They are NOT just numeric IDs.

**Key Characteristics:**
- Case IDs can contain spaces, commas, special characters, dates, and metadata
- Each customer has their own naming convention
- Always quote case IDs in shell commands to handle special characters

**Real-World Examples** (names sanitized):

| Customer | Example Case ID | Pattern |
|----------|----------------|---------|
| **Acadia** | `PC280813 (Smith)` | Alphanumeric case number with surname in parentheses |
| **Arrowhead** | `DOE, Jane 654667 VA supp 11-05-2025 ror id 151636` | Lastname, firstname with claim number, supplemental date, and ROR ID |
| **SCIF** | `06338432#TYPE#synthesis_llm_output` | Numeric with hash-delimited metadata |
| **Examworks** | `22373628 IM` | Numeric with service code suffix |
| **Sharp** | `1003388-SME ( PART 2)` | Numeric with document type and part indicator |

**Note:** Examples above are sanitized - actual patient names replaced with placeholders while preserving format patterns.

**When querying:**
```bash
# Always quote case IDs with spaces/special characters
CASE="DOE, Jane 654667 VA supp 11-05-2025 ror id 151636"
aws dynamodb query ... --expression-attribute-values "{\":pk\":{\"S\":\"CASE#${CASE}\"}}"
```

### Key Structure
- **pk** (Partition Key): `CASE#<case_id>`
- **sk** (Sort Key): `STATUS#<type>#SEGMENT#<segment_id>`

**Note:** DynamoDB key attribute names are lowercase (`pk` and `sk`), not uppercase.

### DynamoDB Item Types

**IMPORTANT:** All SK values begin with `STATUS#` prefix.

| Type | SK Patterns | Notes |
|------|-------------|-------|
| Case | `STATUS#case_upload`, `STATUS#case_summary`, `STATUS#insights`, `STATUS#summary_generated` | One per case |
| Segment | `STATUS#segment_state#SEGMENT#{id}`, `STATUS#pre_extract_review#SEGMENT#{id}`, `STATUS#review_ready#SEGMENT#{id}`, `STATUS#review_in_progress#SEGMENT#{id}` | **Check `segment_state` first** (source of truth) |
| Processing | `STATUS#segmentation`, `STATUS#segmentation_complete`, `STATUS#deduplication#SEGMENT#{id}` | Tracking |

### Segment Status Lifecycle

From `SegmentStatus` (`packages/dao/case_dao/segment_dao/common.py:21-31`):

| Status | Meaning | Pipeline Action |
|--------|---------|-----------------|
| `pre_segment` | AI boundaries | Optional AI-assisted → `created` |
| `pre_segment_in_progress` | Human reviewing | |
| `pre_segment_complete` | Human confirmed | |
| `created` | Segmentation done | Extract → `pre_extract_review` |
| `dispatched_extraction` | Queued | |
| `failed_dispatch` | Queue failed | |
| `review_ready` | Synthesis done | Synthesise → `review_ready` + `review_in_progress` |
| `review_complete` | Human submitted | |
| `ptd_complete` | PTD done | |
| `qa_complete` | QA done | |

### Data Access Objects Reference

Each DAO corresponds to a different SK prefix and serves a specific purpose:

| DAO | SK | Purpose | Key Fields | Created By |
|-----|----|---------|-----------|----|
| SegmentStateDao | `segment_state#...` | Source of truth | `segment_status`, `segment_class`, `segment_class_prediction`, `scenario_resolution`, `gdrive_link`, `reviewer_email` | Segmentation |
| PreExtractReviewSegmentDao | `pre_extract_review#...` | Raw LLM output | `extract.structured_output`, `extract.structured_output_type`, `extract.extractor_version_tag` | Extract λ |
| ModelOutputSegmentDao | `review_ready#...` | Synthesized (readonly) | `title`, `summary`, `specialty_output`, `date`, `provider_info`, `handwritten_pages`, `review_status`, `facets` | Synthesise λ |
| ReviewSegmentDao | `review_in_progress#...` | Editable by humans | All ModelOutput fields + `review_feedback`, `is_excluded` | Synthesise λ, humans |

**Common Fields**: All segment items include: `segment_id`, `case_id`, `segment_class`, `start_page`, `end_page` (0-indexed, exclusive), `created_at`, `updated_at`

### Query Patterns

**Get all items for a case:**
```bash
aws dynamodb query \
  --table-name <customer>-table-<env> \
  --key-condition-expression "pk = :pk" \
  --expression-attribute-values '{":pk":{"S":"CASE#<case_id>"}}'
```

**Get segment state (source of truth):**
```bash
aws dynamodb get-item \
  --table-name <customer>-table-<env> \
  --key '{"pk":{"S":"CASE#<case_id>"},"sk":{"S":"STATUS#segment_state#SEGMENT#<segment_id>"}}'
```

**Get review data:**
```bash
aws dynamodb get-item \
  --table-name <customer>-table-<env> \
  --key '{"pk":{"S":"CASE#<case_id>"},"sk":{"S":"STATUS#review_in_progress#SEGMENT#<segment_id>"}}'
```

**Note on Examples:** Command examples use placeholder variables like `$CASE`, `$BUCKET` for readability. You don't need to literally export these as environment variables - substitute actual values directly into commands when executing.

## S3 Structure & Navigation

### Bucket Naming
- Format: `{stackname}stack-masterbucket7c03eada-{suffix}` (lowercase, all environments)
- Dev stacks have "Dev" in the customer name (e.g., `JoeyDev` → `joeydevstack-masterbucket...`)
- Production stacks use actual customer names (e.g., `Sharp` → `sharpstack-masterbucket...`)
- Staging uses the same naming as production (different region, same account as dev)
- Region: **us-west-2** (staging/production), **us-east-1** (dev)
- Enumerate all: `aws s3 ls | grep masterbucket`

### Case Layout in S3

```
<bucket>/
├── upload/{case}/              # Raw uploaded PDFs
├── merged/{case}/              # Processed case files
│   ├── merged.pdf             # Full merged document
│   ├── chunks/                # 500-page chunks for streaming
│   │   └── chunk_{start}_{end}.pdf
│   ├── pages/                 # OCR data per page
│   │   └── page_N.json        # 0-indexed
│   └── page_images/           # Rendered images
│       ├── page_N.webp        # High-res
│       └── page_N_low.webp    # Thumbnails
├── segment/{case}/             # Individual segment PDFs
│   └── pages_{start}_{end}.pdf # 0-indexed, end exclusive
├── output/                     # Deliverables
├── generated_documents/{case}/ # Generated docs
└── deleted/{case}/             # Soft deletes
```

### Triage Commands

**Check if case is fully processed:**
```bash
aws s3api list-objects-v2 \
  --bucket $BUCKET \
  --prefix "merged/$CASE/" \
  --delimiter /
```

**Inspect OCR data:**
```bash
aws s3 cp s3://$BUCKET/merged/$CASE/pages/page_0.json -
```

**Find deliverables:**
```bash
aws s3api list-objects-v2 \
  --bucket $BUCKET \
  --prefix "output/" \
  --query 'Contents[?contains(Key,`'$CASE'`)]'
```

**Check segment PDF exists:**
```bash
aws s3 ls s3://$BUCKET/segment/$CASE/pages_{start}_{end}.pdf
```

### S3-DynamoDB Relationships

- **DynamoDB page ranges = S3 segment paths**: If `SegmentStateDao` shows `start_page=0, end_page=15`, look for `segment/{case}/pages_0_15.pdf`
- **Merged PDF metadata**: `aws s3api head-object --bucket $BUCKET --key merged/$CASE/merged.pdf` shows `Metadata.last-input-hash`

## CloudFormation Stack Patterns

### Stack Categories

**Shared Stacks (1 per environment):**
- `AccountStack` - IAM roles and policies
- `ClaimsStack` - VPC, OpenSearch → customer resources
- `WafStack` - Web Application Firewall
- `AdminStack` - Admin tools and utilities
- `DataEngineeringStack` - Data pipelines
- `MlStack` - Machine learning infrastructure

**Customer Stacks (per customer):**
- `{Customer}Stack` - S3, DynamoDB, Lambdas, queues, Step Functions
- `production-customer-portal-{Customer}CustomerPortalStack` - Cognito, API Gateway, CloudFront

### Resource Naming Patterns

| Resource Type | Pattern | Example |
|---------------|---------|---------|
| DynamoDB | `{customer}-table-{stage}` (lowercase) | `sharp-table-production` |
| S3 Bucket | `{stackname}stack-masterbucket7c03eada-{suffix}` | `sharpstack-masterbucket7c03eada-abc123` |
| Lambda (explicit) | `{CustomerName}-{stage}-{Purpose}` | `Sharp-production-DdbToOS` |
| Lambda (CDK default) | `{Stack}-{LogicalId}{hash}-{suffix}` | `SharpStack-ExtractHandler1A2B-xyz` |
| SQS Queue | `{Stack}_{Purpose}_Queue_{stage}` | `SharpStack_ExtractJob_Queue_production` |
| SQS DLQ | `{Stack}_{Purpose}_DLQ_{stage}` | `SharpStack_ExtractJob_DLQ_production` |
| Step Functions | `{CustomerName}-{Purpose}-{stage}-SF` | `Sharp-Ingestion-production-SF` |
| Kinesis Stream | `{customer}-dynamodb-{stage}` (lowercase) | `sharp-dynamodb-production` |
| Kinesis Firehose | `dynamodbtos3-{customer}-{stage}` (lowercase) | `dynamodbtos3-sharp-production` |

**Note on Lambda naming:** Most Lambdas follow `{Stack}-{LogicalId}...` but some (like DdbToOS) use `{customerName}-{stage}-{function}`. When in doubt, search log groups:
```bash
aws logs describe-log-groups --log-group-name-prefix /aws/lambda/ --query 'logGroups[*].logGroupName' | grep -i <keyword>
```

### Query Commands

**List all stacks:**
```bash
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE
```

**Describe specific stack:**
```bash
aws cloudformation describe-stacks --stack-name {Customer}Stack
```

**Get stack resources:**
```bash
aws cloudformation describe-stack-resources --stack-name {Customer}Stack
```

## Lambda Debugging

### Find Lambda Functions

**List all functions:**
```bash
aws lambda list-functions --query 'Functions[*].[FunctionName,Runtime,LastModified]'
```

**Get function details:**
```bash
aws lambda get-function --function-name {function-name}
```

### CloudWatch Logs

**Tail logs:**
```bash
aws logs tail /aws/lambda/{function-name} --follow
```

**Filter logs by pattern:**
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{function-name} \
  --filter-pattern "ERROR"
```

**Get recent error logs:**
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{function-name} \
  --start-time $(date -u -d '1 hour ago' +%s)000 \
  --filter-pattern "?ERROR ?Exception ?Traceback"
```

## OpenSearch Architecture (Dev vs Prod)

**IMPORTANT:** Dev and production use different OpenSearch backends:

| Environment | OpenSearch Type | Service Code | Endpoint Format |
|-------------|-----------------|--------------|-----------------|
| **Dev** | Managed OpenSearch Service (VPC) | `es` | `vpc-opensearch-dev-*.us-east-1.es.amazonaws.com` |
| **Staging/Prod** | AOSS Serverless | `aoss` | `*.us-west-2.aoss.amazonaws.com` |

This matters for:
- **Auth:** Both use `AWS4Auth` (from `requests_aws4auth`), but different `service` parameter (`es` for dev, `aoss` for staging/prod)
- **Cold starts:** AOSS Serverless can scale to zero; managed ES doesn't
- **Networking:** Dev uses VPC-based access; AOSS uses VPC endpoints

### Index Naming

There are two separate index families:

**Case data indices** (DynamoDB CDC data for case/segment lookup):

| Environment | Collection Name | Index Name |
|-------------|-----------------|------------|
| Dev | N/A (managed ES) | `{customer}-customer_upload-{stage}` |
| Staging/Prod (AOSS) | `case{customer}` | `customer_upload` |

**RAG/vector search indices** (OCR chunks with embeddings for content search):

| Environment | Index Name |
|-------------|------------|
| All | `{customer}-ocr-chunks-{stage}` |

These are different indices -- don't confuse them. Use `customer_upload` for case/segment queries, `ocr-chunks` for content/semantic search.

## Debugging Workflows

### Debug a Case

When user asks to debug a specific case:

1. **Check DynamoDB for case metadata:**
   ```bash
   aws dynamodb query --table-name <customer>-table-production \
     --key-condition-expression "pk = :pk" \
     --expression-attribute-values '{":pk":{"S":"CASE#<case_id>"}}'
   ```

2. **Check S3 for merged PDF:**
   ```bash
   aws s3 ls s3://<bucket>/merged/<case_id>/merged.pdf
   ```

3. **List segments:**
   Query DynamoDB with sk beginning with `STATUS#segment_state#`

4. **Check Lambda logs** if processing failed:
   ```bash
   aws logs tail /aws/lambda/<relevant-lambda> --since 2h
   ```

### Debug a Segment

When user asks about a specific segment:

1. **Get segment state (source of truth):**
   ```bash
   aws dynamodb get-item --table-name <table> \
     --key '{"pk":{"S":"CASE#<case_id>"},"sk":{"S":"STATUS#segment_state#SEGMENT#<segment_id>"}}'
   ```

2. **Check segment PDF in S3:**
   Use `start_page` and `end_page` from DynamoDB to find the S3 path

3. **Get review data if available:**
   ```bash
   aws dynamodb get-item --table-name <table> \
     --key '{"pk":{"S":"CASE#<case_id>"},"sk":{"S":"STATUS#review_in_progress#SEGMENT#<segment_id>"}}'
   ```

4. **Check extract output:**
   ```bash
   aws dynamodb get-item --table-name <table> \
     --key '{"pk":{"S":"CASE#<case_id>"},"sk":{"S":"STATUS#pre_extract_review#SEGMENT#<segment_id>"}}'
   ```

### Investigate Pipeline Stuck

When processing appears stuck:

1. **Check segment status** - Look for `dispatched_extraction` or `failed_dispatch`
2. **Check SQS queues** - Look for messages in queue or DLQ
3. **Check Lambda logs** - Look for errors or timeouts
4. **Check Step Functions** - If using SF, check execution status

## Debugging Tips

- Always check `segment_state` as the **source of truth** for segment status
- DynamoDB page ranges are 0-indexed with exclusive end
- S3 segment paths match DynamoDB page ranges exactly
- When debugging, trace the full pipeline: upload → merge → segment → extract → synthesize → review
- Check CloudWatch logs for detailed error information
