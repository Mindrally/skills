---
name: snowflake-cortex-ai
description: "Reference for Snowflake Cortex AI Functions (AI_COMPLETE, AI_CLASSIFY, AI_EXTRACT, AI_FILTER, etc.) and Cortex Search for building RAG applications entirely inside Snowflake. Use when writing SQL that calls an LLM, classifying or extracting structured data from text, building a natural-language WHERE filter, or setting up hybrid vector+keyword search for retrieval-augmented generation."
---

# Snowflake Cortex AI

This skill covers Snowflake Cortex — the SQL-callable AI layer of Snowflake, including Cortex AI Functions for LLM/ML tasks and Cortex Search for managed hybrid (vector + keyword) search — all running inside Snowflake with no data leaving the platform.

## Workflow for Building a Cortex-Powered Feature

1. **Pick the narrowest function for the task** — Use `AI_CLASSIFY` for categorization, `AI_FILTER` for natural-language row filtering, `AI_EXTRACT` for structured field pulls, and reserve `AI_COMPLETE` for open-ended generation — narrower functions are cheaper and more reliable than routing everything through `AI_COMPLETE`.
2. **Check token cost before batch jobs** — Run `AI_COUNT_TOKENS(model, text)` on a sample before running a function over a large table.
3. **Prototype in SQL** — Call the function on a small `LIMIT`-ed sample and inspect results before running it over a full table.
4. **Structure prompts explicitly** — Use `PROMPT('template {0}', arg)` for parameterized prompts and cast JSON output to `VARIANT` when you need structured fields back.
5. **For RAG, stand up Cortex Search** — Create a `CORTEX SEARCH SERVICE` over the source table, query it via the Python or REST API to retrieve context, then pass that context into `AI_COMPLETE`.
6. **Use dedicated compute** — Size Cortex Search's backing warehouse no larger than MEDIUM, and separate it from other pipeline warehouses.
7. **Guard against failures** — Use `TRY_COMPLETE` in place of `AI_COMPLETE` for batch jobs where a single failure shouldn't fail the whole run; it returns `NULL` instead of raising an error.

## Cortex AI Functions

Available functions (use these current names — do not use deprecated names like `COMPLETE` or `CLASSIFY_TEXT`):

| Function | Purpose |
|---|---|
| `AI_COMPLETE` | General-purpose LLM completion over text, images, or documents |
| `AI_CLASSIFY` | Classify text/images into user-defined categories (multi-label supported) |
| `AI_FILTER` | Returns `TRUE`/`FALSE` for text/image input — usable directly in `WHERE` |
| `AI_AGG` | Aggregate insights across many rows of text, no context-window limit |
| `AI_EMBED` | Generate embedding vectors for similarity search or clustering |
| `AI_EXTRACT` | Extract structured fields from text, images, or documents |
| `AI_SENTIMENT` | Sentiment score from text, ranging -1 to 1 |
| `AI_SUMMARIZE_AGG` | Summarize across many rows, no context-window limit |
| `AI_SIMILARITY` | Embedding similarity between two inputs |
| `AI_TRANSCRIBE` | Transcribe audio/video files from a stage |
| `AI_PARSE_DOCUMENT` | OCR or text+layout extraction from documents in a stage |
| `AI_REDACT` | Redact PII from text |
| `AI_TRANSLATE` | Translate text between supported languages |

Helper functions:

- `TO_FILE('@stage', 'filename')` — build a file reference for document/image processing.
- `AI_COUNT_TOKENS(model, text)` — check token count before calling a model, especially before large batch jobs.
- `PROMPT('template {0}', arg)` — build parameterized prompt objects for `AI_COMPLETE`.
- `TRY_COMPLETE` — like `AI_COMPLETE` but returns `NULL` on failure instead of raising an error.

### AI_COMPLETE — the primary function

Supported models include `claude-4-opus`, `claude-4-sonnet`, `claude-sonnet-4-5`, `claude-opus-4-5`, `claude-haiku-4-5`, `gemini-3-pro`, `llama3.1-70b`, `llama3.1-8b`, `llama3.3-70b`, `mistral-large2`, `mistral-small2`, and `deepseek-r1`. Model availability varies by region — do not hardcode a model name without checking regional availability.

```sql
-- Text completion
SELECT AI_COMPLETE(MODEL => 'claude-4-sonnet', PROMPT => 'Summarize: ' || review_text) FROM reviews;

-- Document processing
SELECT AI_COMPLETE(
  MODEL => 'claude-4-sonnet',
  PROMPT => PROMPT('Extract the invoice total from {0}', TO_FILE('@docs', 'invoice.pdf'))
);

-- Structured JSON output
SELECT AI_COMPLETE(MODEL => 'claude-4-sonnet',
  PROMPT => 'Extract name, email, company as JSON: ' || raw_text)::VARIANT AS extracted
FROM contacts;
```

### Classification, filtering, and aggregation

```sql
-- AI_CLASSIFY: cheaper than AI_COMPLETE for categorization tasks
SELECT AI_CLASSIFY(ticket_text, ['billing', 'technical', 'account', 'other']) AS category FROM tickets;
-- Multi-label: AI_CLASSIFY(input, categories, {'output_mode': 'multi'})

-- AI_FILTER: natural-language predicate directly in WHERE
SELECT * FROM reviews WHERE AI_FILTER(review_text, 'mentions product quality issues');

-- AI_AGG: cross-row aggregation with no context-window limit
SELECT AI_AGG(feedback_text, 'What are the top 3 themes?') FROM customer_feedback;

-- AI_EXTRACT: pull named fields out of free text
SELECT AI_EXTRACT(email_body, 'meeting date', 'attendees', 'action items') FROM emails;
```

### Other functions

```sql
SELECT review_text, AI_SENTIMENT(review_text) AS sentiment FROM product_reviews;
SELECT AI_EMBED(description) AS embedding FROM products;
SELECT AI_PARSE_DOCUMENT(TO_FILE('@docs', 'contract.pdf'), MODE => 'LAYOUT');
SELECT AI_TRANSCRIBE(TO_FILE('@media', 'recording.mp3')) AS transcript;
SELECT AI_REDACT(customer_notes) AS redacted FROM support_cases;
```

### Privileges

Cortex AI functions require the `USE AI FUNCTIONS` account privilege plus the `SNOWFLAKE.CORTEX_USER` database role — both are granted to `PUBLIC` by default, so most workloads work without extra grants.

## Cortex Search — Hybrid Vector + Keyword Search

Cortex Search is fully managed search combining vector (semantic) and keyword (lexical) retrieval. Typical use cases are RAG for LLM chatbots, enterprise search, and AI-powered Q&A.

### Single-index service

```sql
CREATE OR REPLACE CORTEX SEARCH SERVICE my_search
  ON transcript_text
  ATTRIBUTES region, agent_id
  WAREHOUSE = my_wh
  TARGET_LAG = '1 day'
  EMBEDDING_MODEL = 'snowflake-arctic-embed-l-v2.0'
  AS (SELECT transcript_text, region, agent_id FROM support_transcripts);
```

### Multi-index service (text + vector across columns)

```sql
CREATE OR REPLACE CORTEX SEARCH SERVICE my_multi_search
  TEXT INDEXES transcript_text, summary
  VECTOR INDEXES transcript_text (model='snowflake-arctic-embed-l-v2.0')
  ATTRIBUTES region
  WAREHOUSE = my_wh
  TARGET_LAG = '1 hour'
  AS (SELECT transcript_text, summary, region FROM support_transcripts);
```

Key parameters: `ON` (single-index column), `TEXT INDEXES`, `VECTOR INDEXES`, `ATTRIBUTES` (filterable columns), `TARGET_LAG` (freshness), `EMBEDDING_MODEL`, and `PRIMARY KEY` (enables optimized incremental refresh — set it whenever the source table has a stable key).

### Querying — Python API (recommended for applications)

```python
from snowflake.core import Root

root = Root(session)
service = root.databases["db"].schemas["schema"].cortex_search_services["my_search"]
resp = service.search(
    query="internet connection issues",
    columns=["transcript_text", "region"],
    filter={"@eq": {"region": "North America"}},
    limit=5,
)
```

### Querying — REST API

```
POST /api/v2/databases/<db>/schemas/<schema>/cortex-search-services/<service>:query
Body: {"query": "...", "columns": [...], "filter": {...}, "limit": N}
```

### Filter syntax

```
{"@eq": {"region": "NA"}}
{"@contains": {"tags": "urgent"}}
{"@gte": {"score": 0.8}}
{"@and": [f1, f2]}
{"@or": [f1, f2]}
{"@not": f}
```

### Tuning relevance

Adjust the weighting between text match, vector similarity, and reranker score:

```python
resp = service.search(
    query="billing dispute",
    columns=["transcript_text"],
    scoring_config={"weights": {"texts": 0.3, "vectors": 0.5, "reranker": 0.2}},
    limit=10,
)
```

### RAG pattern

1. Retrieve context: `results = service.search(query=question, columns=["content"], limit=5)`
2. Pass the retrieved context into `AI_COMPLETE`:

```sql
SELECT AI_COMPLETE(
  MODEL => 'claude-4-sonnet',
  PROMPT => 'Answer using only this context: ' || context || ' Question: ' || question
);
```

## Best Practices

- Use `AI_CLASSIFY` instead of `AI_COMPLETE` for categorization — it is purpose-built and cheaper.
- Run `AI_COUNT_TOKENS` before large batch jobs to estimate cost and avoid truncation surprises.
- Set `PRIMARY KEY` on a Cortex Search service so refreshes are incremental rather than full rebuilds.
- Use `ATTRIBUTES` for any column you need to filter on at query time.
- Use `SEARCH_PREVIEW` for interactive testing during development; use the Python or REST API for production integrations.
- Size the warehouse backing a search service no larger than MEDIUM, and dedicate it to that service.

## Anti-Patterns

- Do not use deprecated function names (`COMPLETE`, `CLASSIFY_TEXT`, etc.) — use the current `AI_*` versions.
- Do not pass an entire table through `AI_COMPLETE` row-by-row without first estimating cost with `AI_COUNT_TOKENS`.
- Do not hardcode model names without checking regional availability.
- Do not use `AI_COMPLETE` for tasks a narrower function (`AI_CLASSIFY`, `AI_FILTER`, `AI_EXTRACT`) already covers.
