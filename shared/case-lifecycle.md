# Case & Segment Lifecycle

Source of truth: `packages/dao/case_dao/common.py` (case statuses) and `packages/dao/case_dao/segment_dao/common.py` (segment statuses). Always verify against code -- statuses evolve as features are added.

## Case Processing Status (`CaseProcessingStatus` enum)

```
case_upload → auto_segmentation → segmentation → segmentation_complete → embedding →
pre_extract_review → post_extract_review → review_ready → review_in_progress →
review_complete → summary_generated → final_qa → submission_ready → submitted
```

Branch states: `adding_files`, `segmentation_additional`
Terminal states: `submitted`, `sift_submitted`, `merge_submitted`, `merge_complete`, `manually_submitted`
Deprecated: `ptd_ready`, `ptd_complete` (PTD feature removed)

## Segment Status (`SegmentStatus` enum)

```
pre_segment → pre_segment_in_progress → pre_segment_complete →
created → dispatched_extraction → review_ready → review_complete →
ptd_complete → qa_complete
```

Error state: `failed_dispatch`

## DynamoDB Schema

- **Single-table design** per customer. Table naming: `{customer}-table-{env}`
- **PK**: `CASE#{case_id}` -- case IDs are arbitrary strings (often include spaces/special chars, always quote in shell)
- **SK**: `STATUS#{type}` for case-level items, `STATUS#{type}#SEGMENT#{segment_id}` for segment-level items
- **GSIs**: `GSI1_CREATED_AT` (by status + created_at), `GSI2_LAST_UPDATED` (by status + updated_at)

### Key DynamoDB Item Types (SK prefix = `STATUS#`)

| DAO Class | DynamoStatus (SK value) | Scope | Purpose |
|---|---|---|---|
| CaseDao | `case_upload` | Case | Case metadata, patient info, processing status |
| SegmentationDao | `segmentation` | Case | Segment boundaries after auto-seg |
| SegmentStateDao | `segment_state` | Segment | Segment state source of truth (classification, lock, status) |
| ModelOutputSegmentDao | `review_ready` | Segment | Synthesized model output per segment |
| ReviewSegmentDao | `review_in_progress` | Segment | Human-editable review data per segment |
| PreExtractReviewSegmentDao | `pre_extract_review` | Segment | Pre-extraction review data |
| PostExtractReviewSegmentDao | `post_extract_review` | Segment | Post-extraction review data |
| OutputSummaryDao | `summary_generated` | Case | Generated output metadata and S3 locations |
| CaseSummaryDao | `case_summary` | Case | AI-generated case summary text |
| InsightsDao | `insights` | Case | Structured claim insights (injuries, billing, red flags) |

DAO definitions: `packages/dao/case_dao/` (case-level) and `packages/dao/case_dao/segment_dao/` (segment-level).

## S3 Layout (per-customer bucket)

```
upload/{case_id}/              Raw uploaded files (PDF, DOCX)
merged/{case_id}/merged.pdf    Merged PDF
merged/{case_id}/pages/        Per-page OCR JSON (page_0.json, pages_all.json)
merged/{case_id}/page_images/  Page images (page_{n}.webp)
merged/{case_id}/page_features/ Page feature vectors (page_000042.json)
merged/{case_id}/chunks/       Chunked PDFs for large files (chunk_{start}_{end}.pdf)
segment/{case_id}/             Per-segment PDFs (pages_{start}_{end}.pdf)
output/                        Generated output files (summary PDFs, Word docs)
generated_documents/{case_id}/ Final reports (by doc type)
split/{case_id}/               Split PDF intermediates
deleted/{case_id}/             Soft-deleted files (mirrors original folder structure)
```

Page ranges throughout the system are **0-indexed with exclusive end**.
