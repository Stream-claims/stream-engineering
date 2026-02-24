---
name: langsmith-fetching
description: Fetch LangSmith traces for ML analysis, debugging, and evaluation. Use when the user asks to fetch LLM traces, analyze extraction outputs, or collect training data from LangSmith.
---

# LangSmith Trace Fetching Skill

## When to Use This Skill

Activate this skill when the user:
- Wants to fetch LangSmith traces for a specific segment, case, or project
- Needs to analyze LLM extraction outputs or decisions
- Wants to collect training/eval data from production LLM calls
- Asks about LangSmith API patterns or throttling
- Needs to compare LLM outputs across versions or time periods

## Critical Decision: API vs Redshift

**Choose wisely** - the wrong choice wastes significant time:

### Use Redshift Tables (Preferred for Bulk Analysis)

**When:**
- You need data from > 100 traces
- You want to aggregate/analyze across many segments or cases
- You need to join LangSmith data with other data warehouse tables
- The data is > 24 hours old (export pipeline runs daily)

**Available tables:**
```sql
-- Base traces view (all LangSmith data via Spectrum)
SELECT * FROM warehouse.staging.langsmith_traces
WHERE run_name LIKE 'Extract/%'
  AND environment = 'production'
LIMIT 100;

-- InsightsExtractor specific view
SELECT * FROM warehouse.staging.langsmith_insights_extractor
WHERE environment = 'production';

-- Disambiguation classifier view
SELECT * FROM warehouse.staging.langsmith_clf_disambiguation
WHERE environment = 'production';

-- Candidate generator view
SELECT * FROM warehouse.staging.langsmith_clf_candidate_gen
WHERE environment = 'production';
```

**Cost model:** ~$5/TB scanned. Always filter by date/customer when possible.

**Important: Time conversion** - LangSmith `start_time` is in **nanoseconds** (not milliseconds):
```sql
SELECT TIMESTAMP 'epoch' + start_time/1000000000 * INTERVAL '1 second' as start_datetime
FROM warehouse.staging.langsmith_traces;
```

**Derived Views (Classification Analysis):**

`staging.langsmith_clf_disambiguation` - DisambiguationClf chain outputs:
| Column | Type | Description |
|--------|------|-------------|
| `segment_type_decision` | varchar | Final classification (e.g., 'medical', 'rfa', 'rehab') |
| `candidate_classes` | varchar | Comma-separated candidates from run_name |
| `evidence_json` | varchar | JSON array of evidence strings |

`staging.langsmith_clf_candidate_gen` - LlmCandidateGen chain outputs:
| Column | Type | Description |
|--------|------|-------------|
| `min_candidates` | int | Min candidates requested (from run_name) |
| `max_candidates` | int | Max candidates requested |
| `candidates_json` | varchar | JSON array of candidate types |

**Key difference:** These views have `outputs` as VARCHAR (not SUPER) - use `JSON_EXTRACT_PATH_TEXT()` directly.

**Data truncation (~4.5% of rows):** LangSmith has 16KB column limit. Handle with:
```sql
CASE
    WHEN CAN_JSON_PARSE(outputs)
    THEN JSON_EXTRACT_PATH_TEXT(outputs, 'output', 'field')
    ELSE NULL
END as field
```

### Use LangSmith API (Direct Fetching)

**When:**
- You need specific traces by segment_id (< 100 segments)
- You need real-time data (< 24 hours old)
- You need full run objects with all metadata
- Redshift tables don't have the specific run_name/chain you need

**Caveats (IMPORTANT):**
- **Rate limiting:** LangSmith API allows ~2 requests/second max
- **Throttling backoff:** 429 errors require exponential backoff
- **Individual queries:** OR filters on metadata don't work reliably - must query each segment_id individually
- **Time estimate:** 100 segments * 0.5s = ~50 seconds minimum

## LangSmith Project Names

Key projects in production:

| Project Name | Description | Key Chain Names |
|--------------|-------------|-----------------|
| `extract` | Document extraction pipeline | `Extract/*` |
| `synthesise` | Summary generation | `Synthesise/*` |
| `insights-extractor` | Case insights extraction | `Extract/InsightsExtractor/*` |
| `auto-segmentation` | Document segmentation | Various |
| `dedupe-segment-processor` | Deduplication | Various |
| `proofreader` | Output validation | Various |

## Code Patterns

### Using ml/langsmith_utils.py (Recommended)

The codebase has robust utilities in `ml/langsmith_utils.py`:

```python
from ml.langsmith_utils import (
    extract_langsmith_traces,
    fetch_disambiguation_traces_by_id,
    fetch_correspondence_traces_by_id,
    RATE_LIMIT_DELAY,  # 0.11 seconds
)

# Fetch by segment IDs (handles rate limiting automatically)
df = fetch_disambiguation_traces_by_id(
    segment_ids=["seg1", "seg2", "seg3"],
    project_name="extract",
    tags=["Customer:Arrowhead"],
    rate_limit_delay=0.11,  # 500ms is safer, 110ms is minimum
    show_progress=True,
)

# Fetch by date range (for bulk collection)
from datetime import datetime, timedelta

df = extract_langsmith_traces(
    project_name="insights-extractor",
    date_range=(datetime.now() - timedelta(days=7), datetime.now()),
    extractor_func=my_extractor,  # Custom function to extract relevant fields
    max_runs=1000,
)
```

### Direct LangSmith Client Usage

```python
from langsmith import Client
import time

client = Client()  # Uses LANGCHAIN_API_KEY env var

# List runs with filters
runs = client.list_runs(
    project_name="insights-extractor",
    start_time=datetime.now() - timedelta(days=1),
    run_type="chain",
    limit=100,
)

# Rate-limited iteration
for i, run in enumerate(runs):
    if i > 0:
        time.sleep(0.11)  # Minimum delay between requests

    # Extract data
    inputs = run.inputs or {}
    outputs = run.outputs or {}
    metadata = run.extra.get("metadata", {}) if run.extra else {}

    case_id = metadata.get("context_var_case_id")
    segment_id = metadata.get("context_var_segment_id")

    print(f"Case: {case_id}, Segment: {segment_id}")
```

### Filter Expression Syntax

LangSmith uses a specific filter syntax:

```python
# Filter by tag
filter_expr = 'has(tags, "Customer:Arrowhead")'

# Filter by metadata (segment_id)
filter_expr = "and(eq(metadata_key, 'context_var_segment_id'), eq(metadata_value, 'seg123'))"

# Multiple conditions
filter_expr = 'and(has(tags, "Customer:Arrowhead"), has(tags, "Environment:production"))'

# IMPORTANT: OR doesn't work reliably for metadata filters
# This will NOT work as expected:
# filter_expr = "or(eq(metadata_value, 'seg1'), eq(metadata_value, 'seg2'))"
# Instead, query each segment_id individually
```

## Rate Limiting Deep Dive

### The Problem

LangSmith's cloud API has aggressive rate limits:
- ~500 requests/minute sustained
- Bursts can trigger 429s within seconds
- No official documentation on exact limits

### The Solution (from ml/langsmith_utils.py)

```python
RATE_LIMIT_DELAY = 0.11  # 110ms between requests = ~9 req/sec (conservative)
MAX_RETRIES = 3
BACKOFF_BASE = 2

# Retry logic with exponential backoff
retries = 0
while retries < MAX_RETRIES:
    try:
        result = client.list_runs(**kwargs)
        break
    except Exception as e:
        if "429" in str(e):
            wait_time = BACKOFF_BASE ** retries  # 1s, 2s, 4s
            logger.warning(f"Rate limit hit, waiting {wait_time}s")
            time.sleep(wait_time)
            retries += 1
        else:
            raise
```

### Time Estimates

| Segments | Delay | Estimated Time |
|----------|-------|----------------|
| 10 | 0.11s | ~1 second |
| 100 | 0.11s | ~11 seconds |
| 500 | 0.11s | ~55 seconds |
| 1000 | 0.11s | ~2 minutes |

**Recommendation:** For > 200 segments, use Redshift tables instead.

## Common Extraction Patterns

### InsightsExtractor Traces

```python
def extract_insights_data(run):
    """Extract data from InsightsExtractor run."""
    if not run.name or not run.name.startswith("Extract/InsightsExtractor"):
        return None

    inputs = run.inputs or {}
    outputs = run.outputs or {}
    metadata = run.extra.get("metadata", {}) if run.extra else {}

    output_data = outputs.get("output", {})

    return {
        "run_id": str(run.id),
        "timestamp": run.start_time,
        "case_id": metadata.get("context_var_case_id"),
        "markdown_input": inputs.get("markdown_text"),
        "injuries": output_data.get("injuries", []),
        "strengths": output_data.get("strengths", []),
        "weaknesses": output_data.get("weaknesses", []),
        "body_parts": output_data.get("injuredBodyParts", []),
    }
```

### Disambiguation Classifier Traces

```python
def extract_disambiguation_data(run):
    """Extract data from DisambiguationClf run."""
    if not run.name or not run.name.startswith("DisambiguationClf"):
        return None

    metadata = run.extra.get("metadata", {}) if run.extra else {}
    outputs = run.outputs or {}
    output_data = outputs.get("output", {})

    return {
        "run_id": str(run.id),
        "timestamp": run.start_time,
        "case_id": metadata.get("context_var_case_id"),
        "segment_id": metadata.get("context_var_segment_id"),
        "candidate_types": metadata.get("candidate_types", []),
        "segment_type_decision": output_data.get("segment_type_decision"),
        "evidence": output_data.get("evidence", []),
    }
```

## Redshift vs API: Quick Reference

| Need | Use | Why |
|------|-----|-----|
| Latest trace for 1 segment | API | Fast, real-time |
| Traces for 10 segments | API | Manageable time |
| Traces for 500 segments | Redshift | API would take 1+ min |
| Join with case data | Redshift | Can join staging tables |
| Aggregate statistics | Redshift | SQL is faster |
| Real-time debugging | API | Redshift has 24h delay |

## Data Quality Notes

### LangSmith 16KB Column Limit

~4.5% of LangSmith rows have truncated JSON due to a 16KB column limit. The Redshift views handle this:

```sql
-- Redshift handles truncation gracefully
CASE
    WHEN CAN_JSON_PARSE(extra)
    THEN JSON_PARSE(extra)
    ELSE NULL
END as extra_parsed
```

### Missing Metadata

Some traces may have NULL case_id or segment_id if:
- The trace was from a test/development run
- Metadata wasn't properly propagated
- The JSON was truncated

Always filter for non-NULL IDs when needed:

```sql
SELECT * FROM staging.langsmith_traces
WHERE case_id IS NOT NULL
  AND segment_id IS NOT NULL
  AND environment = 'production';
```

## Environment Variables

Required for LangSmith API access:

```bash
export LANGCHAIN_API_KEY="lsv2_..."  # From LangSmith settings
export LANGCHAIN_TRACING_V2="true"
export LANGCHAIN_PROJECT="your-project"  # Optional default project
```

The project typically has these in `.env.test` or `env.yaml`.

## Troubleshooting

### "No traces found"

1. Check project name is correct (case-sensitive)
2. Verify date range overlaps with actual traces
3. Check tags filter matches exactly (e.g., "Customer:Arrowhead" not "customer:arrowhead")
4. Try without filters first to confirm project has data

### "429 Too Many Requests"

1. Increase `rate_limit_delay` (try 0.5s instead of 0.11s)
2. Add exponential backoff
3. For bulk fetching, switch to Redshift tables

### "run.outputs is None"

1. Chain may have failed - check `run.error`
2. Chain may still be running - check `run.status`
3. Output may not be captured for this chain type

## Instructions for Claude

When this skill is activated:

1. **Clarify the use case:**
   - How many traces/segments needed?
   - How old is the data?
   - What fields are needed?

2. **Recommend the right approach:**
   - < 100 segments, real-time needed → LangSmith API
   - > 100 segments, can wait 24h → Redshift tables
   - Need to join with case data → Redshift tables

3. **Provide working code:**
   - Use `ml/langsmith_utils.py` patterns when possible
   - Include rate limiting for API usage
   - Add progress bars for long operations

4. **Warn about common issues:**
   - Rate limiting (always mention the 0.11s delay)
   - 16KB truncation in Redshift
   - Missing metadata on some traces
   - OR filters not working for metadata

## Related Skills

- **`redshift-warehouse`**: Redshift connection setup, general query patterns, dbt models
- **`ml-data-fetch-annotate`**: Fetching segment/case data, creating analysis PDFs
