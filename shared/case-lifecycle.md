# Case & Segment Lifecycle

## Case Status Flow
```
case_upload → auto_segmentation → segmentation → segmentation_complete → review_ready → submission_ready → summary_generated → submitted/manually_submitted
```

Branch states: `adding_files`, `segmentation_additional`

## Segment Status Flow
```
pre_segment → pre_segment_in_progress → pre_segment_complete → review_ready → review_complete
```

Detailed segment processing flow:
```
pre_segment → created → dispatched_extraction → review_ready → review_complete → ptd_complete → qa_complete
```

## DynamoDB Schema
- Single-table design per customer
- Table naming: `{customer}-table-{env}`
- Key structure: `pk=CASE#{id}`, `sk=STATUS#{type}#SEGMENT#{id}`
- Case IDs are arbitrary strings (often include spaces/special chars) — always quote in shell commands

## DAO Types
| DAO | Purpose | SK Prefix |
|---|---|---|
| SegmentStateDao | Segment state source of truth | STATUS# |
| PreExtractReviewSegmentDao | Pre-extraction review data | PRE_EXTRACT# |
| ModelOutputSegmentDao | Synthesized model output | MODEL_OUTPUT# |
| ReviewSegmentDao | Human-editable review data | REVIEW# |

## S3 Layout (per-customer bucket)
```
upload/          — Raw uploaded files
merged/          — Merged PDFs after processing
segment/         — Individual segment PDFs (pages_{start}_{end}.pdf)
output/          — Generated output files
generated_documents/ — Final reports
deleted/         — Soft-deleted files
```

Page ranges are 0-indexed with exclusive end.
