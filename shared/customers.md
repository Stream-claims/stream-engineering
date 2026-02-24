# Customer Deployment Matrix

## Production Customers

| Customer | Presegment | Rehab | Case Summary | File Submission | Email Ingestion | Daily Email |
|---|---|---|---|---|---|---|
| Arrowhead | no | yes | no | Arrowhead | no | yes (8am PT) |
| Sharp | yes | no | yes | Sharp + Google Drive | no | no |
| Simplexam | yes | no | yes | Simplexam | no | no |
| scif | yes | no | yes | no | no | yes |
| TCS | yes | no | yes | no | no | no |
| BerkleyEnt | yes | no | yes | no | yes | yes |
| BerkleySW | yes | no | yes | no | yes | yes |
| DHS | yes | no | yes | no | no | no |
| Acadia | yes | no | yes | no | yes | yes |
| Playground | yes | no | no | no | no | no |
| Demo | yes | no | no | no | no | no |
| ReportPrep | yes | no | no | no | no | no |

## Dev Environments with Special Config

| Customer | Special Features |
|---|---|
| EilamDev | Rehab enabled, Case Summary enabled |
| AndreyDev | Rehab enabled, LLM OCR enabled |
| DevOpsEnv | Standard config |
| TestEnv | Standard config |

## Customer Types
- **med_review** (default) — Full sidebar, complex workflow for medical record review
- **carrier** — Streamlined interface for insurance carriers, restricted to `/carrier/*` routes

## Email Ingestion
Enabled for: BerkleyEnt, BerkleySW, Acadia

## Daily Email Schedule
- Arrowhead: 8am Pacific (15:00 UTC)
- Others with daily email: 7pm Pacific (02:00 UTC next day)

## File Submission Integrations
- **Sharp**: SFTP upload + Google Drive per-segment PDFs
- **Arrowhead**: Direct API submission
- **Simplexam**: API submission

## Customer-Specific Pipeline Behavior
- **Arrowhead**: No presegmentation, has rehab processing (medical encounter grouping, rehab grouping, subjective narrative)
- **Sharp**: Has presegmentation, Google Drive integration for segment PDFs
- **Simplexam**: Has presegmentation, file submission to Simplexam API
