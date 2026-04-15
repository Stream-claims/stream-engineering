# Stream Engineering — Shared Agent Context

## About Stream Claims

Stream Claims builds an AI-powered medical record review platform. The backend processes insurance claim documents through OCR, segmentation, extraction, synthesis, and PDF output pipelines. The customer portal is a multi-tenant SvelteKit app. The data engineering pipeline loads DynamoDB CDC data into a Redshift warehouse via dbt.

## Purpose

This repo provides shared context for coding agents across the Stream engineering team. Per-repo context (commands, code style, patterns) lives in each repo's own CLAUDE.md — this repo holds cross-cutting domain knowledge and shared skills.

## Repo Structure

- `shared/` — Cross-cutting domain knowledge (architecture, case lifecycle, AWS accounts, customers)
- `skills/` — On-demand skills for coding agents
- `eilam-claude-md.md` — Eilam's personal agent configuration

## Skills

Load a skill with `ToolSearch` → `select:Skill`, then call `Skill` with the skill name.

| Skill | Trigger |
|---|---|
| aws-debug | Debug cases, DynamoDB, S3, Lambda, CloudFormation |
| copy-case | Copy a case between environments |
| create-environment | Provision new customer environment (CDK + portal) |
| daily-alerts-review | Daily production triage from Slack alerts + Sentry |
| implement-ticket | Linear ticket → worktree → PR workflow |
| langsmith-fetching | Fetch LLM traces for ML analysis |
| linear-tickets | Manage Linear tickets |
| ml-data-fetch-annotate | Fetch ML datasets, create annotated PDFs |
| oncall-handoff | On-call shift handoff summary to Slack |
| pr-monitor-loop | Monitor PR for review comments and auto-fix |
| redshift-warehouse | Query Redshift, debug dbt models |
| review-pr | PR review with code/security/comment analysis |
| segment-search | Search OCR chunks in OpenSearch |
| sentry-cli | Sentry CLI operations and project reference |
| tailscale-vpn | Environment VPN switching + 403 debugging |
| upload-case | Upload PDF cases to Stream Claims |
