---
name: segment-search
description: Search OCR chunks in OpenSearch using hybrid (BM25 + vector) search. Use when the user asks to find relevant content in medical records, search segments, or query the RAG index.
---

# Segment Search Skill

## When to Use This Skill

Invoke this skill when the user:
- Asks to search for content in medical records (e.g., "Find mentions of diabetes")
- Wants to find relevant segments for a topic
- Needs to query the RAG/vector search index
- Asks about what's in a case's OCR data

## Prerequisites

1. **AWS Profile** - Check `CLAUDE.local.md` for user's profile, or ask
2. **OpenSearch Endpoint** - Per environment:
   - Dev: `vpc-opensearch-dev-*.us-east-1.es.amazonaws.com`
   - Staging/Prod: AOSS collection endpoint
3. **Index Name**: `{customer}-ocr-chunks-{stage}`

## Search Implementation

### Step 1: Get Embedding for Query

Use OpenAI API to embed the search query:

```python
import openai

def get_embedding(text: str) -> list[float]:
    response = openai.embeddings.create(
        model="text-embedding-3-large",
        input=text
    )
    return response.data[0].embedding
```

### Step 2: Build Hybrid Query

```python
def build_hybrid_query(query_text: str, embedding: list[float], case_id: str = None, size: int = 10):
    query = {
        "size": size,
        "query": {
            "hybrid": {
                "queries": [
                    {"match": {"content": query_text}},
                    {"knn": {"embedding": {"vector": embedding, "k": size}}}
                ]
            }
        },
        "_source": ["content", "case_id", "segment_id", "metadata"]
    }

    # Optional: filter by case_id
    if case_id:
        query["query"] = {
            "bool": {
                "must": [query["query"]],
                "filter": [{"term": {"case_id": case_id}}]
            }
        }

    return query
```

### Step 3: Execute Search

```python
from opensearchpy import OpenSearch, RequestsHttpConnection
from requests_aws4auth import AWS4Auth
import boto3

def search_segments(query_text: str, index_name: str, endpoint: str, case_id: str = None):
    # Get embedding
    embedding = get_embedding(query_text)

    # Auth
    credentials = boto3.Session().get_credentials()
    auth = AWS4Auth(
        credentials.access_key,
        credentials.secret_key,
        "us-east-1",  # or us-west-2 for staging/prod
        "es",  # or "aoss" for serverless
        session_token=credentials.token
    )

    # Client
    client = OpenSearch(
        hosts=[{"host": endpoint, "port": 443}],
        http_auth=auth,
        use_ssl=True,
        verify_certs=True,
        connection_class=RequestsHttpConnection,
    )

    # Search with hybrid pipeline
    query = build_hybrid_query(query_text, embedding, case_id)
    response = client.search(
        index=index_name,
        body=query,
        params={"search_pipeline": f"{index_name}-hybrid-pipeline"}
    )

    return response["hits"]["hits"]
```

## Quick CLI Search (Dev Only)

For dev environment with OpenSearch Service, use curl:

```bash
# Get auth token
export AWS_PROFILE=<dev-profile>

# Search (BM25 only, no embedding)
curl -X POST "https://<endpoint>/<index>/_search" \
  -H "Content-Type: application/json" \
  --aws-sigv4 "aws:amz:us-east-1:es" \
  -d '{
    "query": {"match": {"content": "diabetes"}},
    "size": 5,
    "_source": ["content", "case_id", "segment_id", "metadata.segment_class"]
  }'
```

## Index Schema Reference

| Field | Type | Description |
|-------|------|-------------|
| `doc_id` | keyword | `{segment_id}_semantic_chunk_{i}` |
| `case_id` | keyword | Case identifier |
| `segment_id` | keyword | Segment UUID |
| `content` | text | Chunk text |
| `embedding` | knn_vector | 3072-dim OpenAI embedding |
| `metadata.segment_class` | keyword | labs, imaging, notes, etc. |
| `metadata.start_page` | integer | Segment start page |
| `metadata.end_page` | integer | Segment end page |
| `metadata.provider` | keyword | Healthcare provider |
| `metadata.date` | keyword | Document date |

## Filtering Examples

**By segment class:**
```json
{"filter": [{"term": {"metadata.segment_class": "labs"}}]}
```

**By date range:**
```json
{"filter": [{"range": {"metadata.date": {"gte": "2024-01-01"}}}]}
```

**By case:**
```json
{"filter": [{"term": {"case_id": "CASE_ID_HERE"}}]}
```

## Troubleshooting

- **403 Forbidden**: Check AWS profile has OpenSearch access
- **No results**: Verify index exists, check case_id spelling
- **Connection timeout**: Ensure VPC access (dev requires VPN/bastion)
