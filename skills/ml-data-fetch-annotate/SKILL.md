---
name: ml-data-fetch-annotate
description: Fetch ML datasets from Redshift and create annotated PDFs for error analysis. Use when the user asks to query segments, fetch classification data, create review PDFs, analyze model errors, or understand the data warehouse schema for ML evaluation.
---

# ML Data Fetching and PDF Annotation Skill

## When to Use This Skill

Activate this skill when the user:
- Wants to fetch segment or case data from Redshift
- Needs to create annotated PDFs for model error analysis
- Asks how to query the data warehouse for ML data
- Wants to analyze classification errors (false positives/negatives)
- Needs to retrieve images from S3 for review
- Asks about the segment/case data schema
- Wants to visualize model predictions vs. ground truth
- Needs to set up an evaluation or benchmarking dataset

## Quick Start (5 Minutes to Success)

**Goal**: Fetch recent segments and create an annotated PDF.

**Prerequisites**: Know your customer name(s). If unsure, use the `aws-debug` skill via CLI: `aws dynamodb scan --table-name <customer>-table-production --select COUNT --profile <your-prod-readonly-profile>` or query available customer names from Redshift.

```python
import boto3, pandas as pd
from ml.util import (get_redshift_secret, get_redshift_connection, RedshiftCredentials,
                     parse_scenario_resolution, get_customer_to_masterbucket_map)
from ml.pdf_annotation import create_outcome_pdf

# 1. Setup credentials and connections
# Profile naming: {username}-{account}-{tier} (e.g., "jane-prod-readonly")
# Available tiers: readonly (safe), poweruser (writes), admin (full access)
secret_dict = get_redshift_secret("<your-prod-readonly-profile>")
redshift_creds = RedshiftCredentials(
    host="data-warehouse-workgroup.982131176572.us-west-2.redshift-serverless.amazonaws.com",
    database="warehouse", port=5439,
    user=secret_dict["username"], password=secret_dict["password"]
)
session = boto3.Session(profile_name="<your-prod-readonly-profile>")
s3_client = session.client("s3", config=boto3.session.Config(read_timeout=10, connect_timeout=5, retries={'max_attempts': 2}))
customer_bucket_map = get_customer_to_masterbucket_map(s3_client)

# 2. Query segments (replace CUSTOMER_NAME and SEGMENT_TYPE)
query = """
WITH segment_state_with_proper_current AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY case_id, segment_id
           ORDER BY COALESCE(updated_at, created_at, cdc_timestamp) DESC) = 1 as true_is_current
    FROM warehouse.staging.segment_state
)
SELECT ss.customer_name, ss.case_id, ss.segment_id, ss.start_page, ss.end_page,
       ss.scenario_resolution, rs.segment_class as true_class
FROM segment_state_with_proper_current ss
JOIN warehouse.staging.review_segment rs ON ss.segment_id = rs.segment_id
WHERE ss.true_is_current = TRUE AND rs.is_current = TRUE AND rs.is_review_complete = TRUE
  AND ss.customer_name = 'berkley_ent' AND rs.segment_class = 'BILL'
ORDER BY ss.created_at DESC LIMIT 30
"""

with get_redshift_connection(redshift_creds) as conn:
    segments_df = pd.read_sql_query(query, conn)

# 3. Parse predictions and create PDF
parsed = segments_df['scenario_resolution'].apply(parse_scenario_resolution)
segments_df['predicted_class'] = parsed.apply(lambda x: x.get('decision'))

create_outcome_pdf(segments_df, s3_client, customer_bucket_map,
                   "output/segments.pdf", "example", max_pages_per_segment=10)
print(f"✓ Created PDF with {len(segments_df)} segments")
```

**Done!** Your annotated PDF is in `output/segments.pdf`.

## Complete Documentation

The comprehensive guide is located at: `.claude/ML_DATA_FETCHING_AND_ANNOTATION_GUIDE.md`

Always reference this guide for:
- Detailed query patterns
- Complete workflow examples
- Troubleshooting tips
- API reference

## Quick Reference

### Core Utilities

**From `ml/util.py`:**
- `get_redshift_connection(credentials)` - Database connection
- `parse_scenario_resolution(json)` - Parse classification metadata (auto-migrates v2.4→v3.0)
- `extract_candidates_from_digest(dict)` - Extract final_candidates from parsed resolution
- `extract_base_classifier_candidates(dict)` - Extract base classifier candidates from candidate_info
- `extract_llm_candidates(dict)` - Extract LLM-generated candidates from candidate_info
- `fetch_segments_by_case_ids(creds, case_ids)` - Fetch segments by case IDs
- `fetch_page_text_for_segments(creds, segments_df)` - Fetch OCR text
- `get_customer_to_masterbucket_map(s3_client)` - S3 bucket mapping
- `normalize_customer_name_for_bucket(customer_name)` - Normalize name for bucket lookup ("berkley_sw" → "berkleysw")
- `get_bucket_for_customer(s3_client, customer_name)` - Get bucket with automatic name normalization
- `get_page_image(s3_client, bucket, case_id, page_idx, use_low_res)` - Fetch page image

**Customer Name Normalization:**
Customer names in Redshift may contain underscores (e.g., `berkley_sw`, `city_of_la`), but S3 bucket names remove all non-alphanumeric characters due to CloudFormation naming rules. Always use `get_bucket_for_customer()` or manually normalize with `normalize_customer_name_for_bucket()` before bucket lookups.

**From `ml/pdf_annotation.py`:**
- `create_outcome_pdf(segments_df, s3_client, bucket_map, output_path, outcome)` - High-level PDF generation
- `add_separator_bar(page, text)` - Visual separator
- `add_text_overlay(page, text, ...)` - Add annotation boxes

### PyMuPDF Image Handling (CRITICAL)

**Problem**: `insert_pdf()` only works for PDF documents, not images. Attempting to use it on image documents will fail with "source or target not a PDF".

**Correct Pattern** (from `generate_annotated_pdf.py`):
```python
import fitz
from PIL import Image
import io

# 1. Convert WEBP/other formats to PNG first
def convert_to_png(image_bytes: bytes) -> bytes:
    img = Image.open(io.BytesIO(image_bytes))
    if img.mode not in ("RGB", "L"):
        img = img.convert("RGB")
    png_io = io.BytesIO()
    img.save(png_io, format="PNG")
    png_io.seek(0)  # Reset pointer!
    return png_io.getvalue()

# 2. Load as Pixmap to get dimensions
image_bytes = convert_to_png(image_bytes)
pix = fitz.Pixmap(image_bytes)
width = int(pix.width)
height = int(pix.height)

# 3. Create new page with image dimensions
page = pdf_doc.new_page(width=width, height=height)

# 4. Insert image directly into page
page.insert_image(fitz.Rect(0, 0, width, height), stream=image_bytes)

# 5. Clean up
pix = None
```

**Common Mistakes**:
```python
# WRONG - Can't insert_pdf() an image document
img_doc = fitz.open(stream=image_bytes, filetype="png")
pdf_doc.insert_pdf(img_doc)  # RuntimeError: source or target not a PDF

# WRONG - Missing PNG conversion
page.insert_image(rect, stream=webp_bytes)  # May fail or corrupt

# WRONG - Not resetting BytesIO pointer
png_io.save(img, format="PNG")
return png_io.getvalue()  # May return partial data
```

### Key Data Schema

**Segment DataFrame:**
```python
{
    'customer_name': str,
    'case_id': str,
    'segment_id': str,
    'start_page': int,        # 0-indexed, inclusive
    'end_page': int,          # 0-indexed, exclusive
    'true_class': str,        # From review_segment
    'predicted_class': str,   # From scenario_resolution
    'scenario_resolution': str,  # JSON
}
```

**Parsed Scenario Resolution (v3.0):**
```python
{
    # Core Classification
    'decision': Optional[str],                           # Predicted segment type
    'decision_source': Optional[str],                    # 'xgb_text_classifier', 'pattern', 'llm', 'failure_fallback'
    'candidate_info': Optional[Dict],                    # Dict with 'final_candidates' and 'sources'
    'llm_disambiguation_status': str,                    # 'not_attempted', 'success', 'failure'

    # Facets (orthogonal properties)
    'ehr_type': Optional[str],                           # 'epic' or None (no 'non_epic' value)
    'is_correspondence': Optional[bool],
    'is_empty_content': Optional[bool],                  # NEW Oct 2025
    'is_fax_cover': Optional[bool],

    # Profile fields (from LLM, optional)
    'doc_title_coarse': Optional[str],
    'form_standardization': Optional[str],               # 'low', 'medium', 'high'

    'version': str,                                      # 'v3.0'
}
```

**Note:** Use helper functions to extract candidates:
- `extract_candidates_from_digest()` → final_candidates list
- `extract_base_classifier_candidates()` → base classifier ordered candidates

## Standard Workflow Templates

### 1. Query Segments with Error Analysis

```python
from ml.util import get_redshift_secret, get_redshift_connection, RedshiftCredentials, parse_scenario_resolution
import pandas as pd

# Setup - use your prod-readonly profile for safe read operations
secret_dict = get_redshift_secret("<your-prod-readonly-profile>")
redshift_creds = RedshiftCredentials(
    host="data-warehouse-workgroup.982131176572.us-west-2.redshift-serverless.amazonaws.com",
    database="warehouse",
    port=5439,
    user=secret_dict["username"],
    password=secret_dict["password"],
)

# Query with proper current record handling
query = """
WITH segment_state_with_proper_current AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY case_id, segment_id
            ORDER BY COALESCE(updated_at, created_at, cdc_timestamp) DESC
        ) = 1 as true_is_current
    FROM warehouse.staging.segment_state
)
SELECT
    ss.customer_name, ss.case_id, ss.segment_id,
    ss.start_page, ss.end_page, ss.scenario_resolution,
    rs.segment_class as true_class, rs.is_excluded
FROM segment_state_with_proper_current ss
JOIN warehouse.staging.review_segment rs
    ON ss.segment_id = rs.segment_id
JOIN warehouse.staging.case_upload cu
    ON ss.case_id = cu.case_id
WHERE ss.true_is_current = TRUE
    AND rs.is_current = TRUE
    AND rs.is_review_complete = TRUE
    AND cu.is_current = TRUE
"""

with get_redshift_connection(redshift_creds) as conn:
    segments_df = pd.read_sql_query(query, conn)

# Parse predictions
parsed = segments_df['scenario_resolution'].apply(parse_scenario_resolution)
segments_df['predicted_class'] = parsed.apply(lambda x: x.get('decision'))

# Calculate errors
segments_df['is_correct'] = segments_df['predicted_class'] == segments_df['true_class']
```

### 2. Create Annotated Error PDFs

```python
import boto3
from ml.util import get_customer_to_masterbucket_map
from ml.pdf_annotation import create_outcome_pdf

# S3 setup - use your prod-readonly profile
session = boto3.Session(profile_name="<your-prod-readonly-profile>")
s3_client = session.client("s3", config=boto3.session.Config(
    read_timeout=10, connect_timeout=5, retries={'max_attempts': 2}
))
customer_bucket_map = get_customer_to_masterbucket_map(s3_client)

# Define styles
PDF_STYLES = {
    'false_negative': {'label': 'False Negative', 'color': (1.0, 0.8, 0.8)},
    'false_positive': {'label': 'False Positive', 'color': (1.0, 0.9, 0.6)},
}

# Generate PDFs
for error_type in ['false_negative', 'false_positive']:
    error_segments = segments_df[segments_df['error_type'] == error_type].sample(n=50)

    create_outcome_pdf(
        segments_df=error_segments,
        s3_client=s3_client,
        customer_bucket_map=customer_bucket_map,
        output_path=f"output/{error_type}.pdf",
        outcome=error_type,
        max_pages_per_segment=10,
        outcome_styles=PDF_STYLES,
    )
```

## Critical Patterns

### Current Record Filtering

**`is_current = TRUE` is now reliable** - use it directly:

```sql
-- PREFERRED: Use is_current directly (now reliable)
SELECT * FROM warehouse.staging.segment_state WHERE is_current = TRUE

-- LEGACY: ROW_NUMBER workaround (no longer needed, but still works)
WITH segment_state_with_proper_current AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY case_id, segment_id
            ORDER BY COALESCE(updated_at, created_at, cdc_timestamp) DESC
        ) = 1 as true_is_current
    FROM warehouse.staging.segment_state
)
SELECT * FROM segment_state_with_proper_current WHERE true_is_current = TRUE
```

Note: The ROW_NUMBER pattern was needed historically when `is_current` had edge cases. It's been fixed - use `is_current` for simpler queries.

### Always Use Low-Res Images for Analysis

```python
# For analysis PDFs, use low-res (200 DPI instead of 300 DPI)
image_format, image_bytes = get_page_image(
    s3_client, bucket, case_id, page_idx,
    use_low_res=True  # Much faster, smaller files
)
```

### Always Sample Large Datasets

```python
# Don't create PDFs with hundreds of segments
if len(segments_df) > 50:
    segments_df = segments_df.sample(n=50, random_state=42)
```

### Parsing SUPER Columns

**Preferred approach:** Fetch as-is and parse in Python using `parse_scenario_resolution()`:

```python
# Fetch entire column, parse in Python (most flexible)
query = "SELECT scenario_resolution FROM segment_state WHERE ..."
df = pd.read_sql_query(query, conn)
parsed = df['scenario_resolution'].apply(parse_scenario_resolution)
df['decision'] = parsed.apply(lambda x: x.get('decision'))
```

For SQL-level SUPER column querying patterns (`JSON_SERIALIZE`, `JSON_EXTRACT_PATH_TEXT`, NULL handling), see **`redshift-warehouse`** skill.

## Common Tasks

### Task: Find false negatives where base classifier had true class

```python
from ml.util import parse_scenario_resolution, extract_base_classifier_candidates

# Parse scenario resolution
parsed = segments_df['scenario_resolution'].apply(parse_scenario_resolution)
segments_df['base_clf_candidates'] = parsed.apply(extract_base_classifier_candidates)

# Check if true class was in base classifier candidates
segments_df['true_in_base_candidates'] = segments_df.apply(
    lambda row: row['true_class'] in row.get('base_clf_candidates', []),
    axis=1
)

# Find "close misses" - base classifier had the right answer but final decision was wrong
close_misses = segments_df[
    ~segments_df['is_correct'] &
    segments_df['true_in_base_candidates']
]

print(f"Found {len(close_misses)} close misses where base classifier had true class")
```

### Task: Analyze LLM disambiguation failures

```python
# Extract LLM status
parsed = segments_df['scenario_resolution'].apply(parse_scenario_resolution)
segments_df['llm_status'] = parsed.apply(lambda x: x.get('llm_disambiguation_status'))

# Filter for LLM failures (status is 'failure' not 'failed')
llm_failures = segments_df[segments_df['llm_status'] == 'failure']

# Create PDF to review
create_outcome_pdf(llm_failures.sample(n=30), s3_client, customer_bucket_map,
                   "output/llm_failures.pdf", "llm_failure")
```

### Task: Find empty content documents (NEW)

```python
# Extract is_empty_content facet (added Oct 2025)
parsed = segments_df['scenario_resolution'].apply(parse_scenario_resolution)
segments_df['is_empty_content'] = parsed.apply(lambda x: x.get('is_empty_content'))

# Filter for empty content documents
empty_docs = segments_df[segments_df['is_empty_content'] == True]

print(f"Found {len(empty_docs)} empty content documents")
print(f"Error rate on empty docs: {(~empty_docs['is_correct']).mean():.2%}")
```

### Task: Compare Epic vs. non-Epic performance

**Important**: `ehr_type` values are `'epic'` or `None` (null/missing). There is NO `'non_epic'` string value - non-Epic is indicated by absence of the field.

```python
# Parse EHR type
parsed = segments_df['scenario_resolution'].apply(parse_scenario_resolution)
segments_df['ehr_type'] = parsed.apply(lambda x: x.get('ehr_type'))

# Calculate accuracy by EHR type
# NOTE: non-Epic is indicated by ehr_type being None, not 'non_epic'
epic_acc = segments_df[segments_df['ehr_type'] == 'epic']['is_correct'].mean()
non_epic_acc = segments_df[segments_df['ehr_type'].isna()]['is_correct'].mean()

print(f"Epic accuracy: {epic_acc:.2%}")
print(f"Non-Epic accuracy: {non_epic_acc:.2%}")
```

## Key Configuration

```python
# Configure your AWS profile in CLAUDE.local.md or ~/.claude/CLAUDE.md
# Profile naming convention: {username}-{account}-{tier}
# Example: "jane-prod-readonly", "bob-staging-poweruser"
AWS_PROFILE = "<your-prod-readonly-profile>"  # For read-only production access
REDSHIFT_HOST = "data-warehouse-workgroup.982131176572.us-west-2.redshift-serverless.amazonaws.com"
REDSHIFT_DATABASE = "warehouse"
```

## Troubleshooting

**"No segments found"**
- Check `is_review_complete = TRUE` filter
- Verify date range filters
- Check customer name formatting

**S3 timeouts**
- Use `use_low_res=True`
- Add retry config to S3 client
- Check AWS profile permissions

**PDF too large**
- Limit `max_pages_per_segment` to 10
- Sample segments before generating
- Use low-res images

## Key Notebooks for Examples

- `ml/analysis/doc_scenario_error_analysis_pdfs.ipynb` - Complete workflow
- `ml/analysis/segment_pdf_builder.ipynb` - Flexible PDF builder
- `ml/benchmarks/doc_scenario/standard_stratified_dataset.ipynb` - Dataset creation

## Instructions for Claude

When this skill is activated:

1. **Reference the guide first**: Always point to `.claude/ML_DATA_FETCHING_AND_ANNOTATION_GUIDE.md` for detailed documentation

2. **Understand the user's goal**:
   - Do they need to fetch data?
   - Do they need to create PDFs?
   - Do they need to analyze errors?
   - Do they need help with queries?

3. **Provide appropriate templates**: Use the workflow templates above as starting points

4. **Use aws-debug skill for validation, NOT notebook cells**:
   - **NEVER add diagnostic cells to notebooks** to discover customer names, check DynamoDB state, or validate S3 buckets
   - **INSTEAD**: Use the `aws-debug` skill with Bash tool + AWS CLI commands to check assumptions BEFORE writing notebook code
   - Example: If uncertain about customer names, use `aws dynamodb scan` or query Redshift directly via CLI
   - Example: If uncertain about bucket naming, use `aws s3 ls | grep masterbucket` via Bash
   - **Rationale**: Diagnostic cells clutter notebooks, require user execution, and pollute the analysis flow
   - Clean notebooks = production-ready notebooks

5. **Emphasize critical patterns**:
   - **v3.0 Schema**: Use current schema fields, not deprecated v2.4 fields
   - **Helper functions**: Use `extract_base_classifier_candidates()`, `extract_candidates_from_digest()` - never access `base_clf_top5` directly
   - **Redshift constraints**: NEVER use `::json` operations - always fetch as string and parse in Python
   - **Current record filtering**: Use `is_current = TRUE` (now reliable)
   - **Low-res images**: Always use `use_low_res=True` for analysis PDFs
   - **Sampling**: Always sample large datasets before PDF generation (50-100 max)

6. **New v3.0 features to leverage**:
   - `is_empty_content` facet for blank/meaningless documents
   - Profile fields (`doc_title_coarse`, `form_standardization`, etc.) for analysis
   - `candidate_info.sources` structure for understanding candidate generation pipeline

7. **Help with common pitfalls**:
   - Warn about PDF file size if sampling isn't mentioned
   - Remind about S3 timeout configuration
   - Check that proper current record filtering is used
   - Catch attempts to use ::json in Redshift queries
   - Catch references to deprecated v2.4 fields (`base_clf_top5`, `fallback_candidates`, etc.)

8. **Suggest relevant notebooks**: Point to example notebooks for reference

9. **Be pragmatic**: These utilities are battle-tested. Trust the patterns and encourage reuse of existing helper functions rather than reinventing.

## Related Skills

- **`redshift-warehouse`**: Connection setup, SUPER column querying, schema exploration, dbt models
- **`langsmith-fetching`**: LLM trace analysis, API vs Redshift decisions, rate limiting patterns
- **`ml-benchmarking`**: Running offline evaluations on batched datasets
