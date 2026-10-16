# Vidur API Documentation

Welcome to the **Vidur External API** — the fastest way to bring Vidur's legal AI capabilities directly into your product. Ask questions against Vidur's knowledge base, stream answers in real time, surface curated news tickers and prompts, and show your users the sources behind every answer.

- **Interactive Swagger UI:** [`/vidur-apis-docs`](https://api.vidur.in/vidur-apis-docs) — explore the endpoints and try them live with your API key.
- **OpenAPI spec (JSON):** [`/vidur-apis-docs.json`](https://api.vidur.in/vidur-apis-docs.json) 

---

## Table of Contents

1. [Base URLs](#base-urls)
2. [Authentication](#authentication)
3. [Quota & Rate Limiting](#quota--rate-limiting)
4. [Endpoints Overview](#endpoints-overview)
5. [Ask Question](#1-ask-question)
6. [Latest News Ticker](#2-latest-news-ticker)
7. [Prompt Library](#3-prompt-library)
8. [Conversation Citations](#4-conversation-citations)
9. [Reference](#reference)
10. [Error Responses](#error-responses)

---

## Base URLs

| Environment | Base URL |
| --- | --- |
| Production | `https://api.vidur.in` |

All endpoints are prefixed with `/vidur-apis/v1/`.

## Authentication

Every request must include your organization's API key in the `x-api-key` header:

```
x-api-key: <YOUR_API_KEY>
```

Requests with a missing or invalid key return `401 Unauthorized`.

## Quota & Rate Limiting

`POST /vidur-apis/v1/ask-question` consumes **1 query quota unit** per call. Every ask-question response carries these headers so you can track usage:

| Header | Description |
| --- | --- |
| `X-Quota-Limit` | Total queries allowed for the current period |
| `X-Quota-Used` | Queries consumed so far |
| `X-Quota-Remaining` | Queries left in the current period |

When the quota is exhausted, the API returns `429` with the message *API query quota exhausted*.

## Endpoints Overview

| # | Method | Endpoint | Description |
| --- | --- | --- | --- |
| 1 | `POST` | `/vidur-apis/v1/ask-question` | Ask a legal question and stream the answer (SSE) |
| 2 | `GET` | `/vidur-apis/v1/latest-ticker` | Get the latest news ticker list |
| 3 | `GET` | `/vidur-apis/v1/prompt-library` | List curated Vidur prompt library entries |
| 4 | `POST` | `/vidur-apis/v1/conversations/citation` | Fetch citation PDF URLs for an answer source |

---

## 1. Ask Question

`POST /vidur-apis/v1/ask-question`

Ask a legal question and receive the answer as a **Server-Sent Events (SSE)** stream of chunks. Question and answer are grounded in Vidur's knowledge base.

### Request Body (`application/json`)

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `question` | string | ✅ | User question (max 200 characters) |
| `law_type` | string | ✅ | Law type filter — see [Law Types](#law-types) |
| `answer_style` | string | — | Answer style — see [Answer Styles](#answer-styles) |
| `conversation_id` | string | — | Continue an existing conversation (max 128 chars). Generated automatically if omitted. |
| `caller_id` | string | — | Your end-user identifier (max 128 chars). Defaults to `provider-{id}`. |
| `message_history` | array | — | Conversation history in **descending** order, max 6 items. Each item: `{ "role": "system" \| "user" \| "assistant", "content": "..." }` |
| `grounding` | boolean | — | Enable web-search grounding |

### Response (`text/event-stream`)

The response is an SSE stream with three event types:

| Event | Payload |
| --- | --- |
| `start` | `{ "type": "start", "conversation_id": "..." }` |
| `chunk` | `{ "type": "chunk", "content": "..." }` — streamed answer text |
| `end` | `{ "type": "end", "conversation_id": "...", "content": "...", "sources": [...], "followup_questions": [...], "trace_id": "..." }` |

The `end` event's `sources` array contains the supporting references — pass an entry to the [Citations](#4-conversation-citations) endpoint to get its PDF.

### Example

```bash
curl -N -X POST "https://api.vidur.in/vidur-apis/v1/ask-question" \
  -H "Content-Type: application/json" \
  -H "x-api-key: $VIDUR_API_KEY" \
  -d '{
    "question": "What is the deadline for filing GSTR-3B?",
    "law_type": "GST",
    "answer_style": "VIDUR",
    "message_history": [
      { "role": "user", "content": "Tell me about GST returns" },
      { "role": "assistant", "content": "GST returns are periodic filings..." }
    ]
  }'
```

**Errors:** `400` invalid request body · `401` missing or invalid `x-api-key` · `429` API query quota exhausted

---

## 2. Latest News Ticker

`GET /vidur-apis/v1/latest-ticker`

Returns the current news ticker list so you can keep your users informed with timely, relevant legal headlines.

### Example

```bash
curl "https://api.vidur.in/vidur-apis/v1/latest-ticker" \
  -H "x-api-key: $VIDUR_API_KEY"
```

### Response (`200`)

```json
{
  "tickers": [
    { "id": "ticker-1", "title": "ITR filing deadline extended to ...", "position": 1 },
    { "id": "ticker-2", "title": "New GST notification issued ...", "position": 2 }
  ]
}
```

**Errors:** `401` missing or invalid `x-api-key`

---

## 3. Prompt Library

`GET /vidur-apis/v1/prompt-library`

Returns paginated prompts from the Vidur organization knowledge base, ready to be surfaced in your product.

### Query Parameters

| Param | Required | Description |
| --- | --- | --- |
| `law_type` | ✅ | Law type filter — use `ALL` for all types (see [Law Types](#law-types); `ALL` allowed here) |
| `allowed_law_types` | — | Only when `law_type=ALL`. Comma-separated filter, e.g. `GST,INCOME_TAX` |
| `page_size` | — | Page size (default `20`) |
| `page_number` | — | Page number (default `1`) |

### Example

```bash
curl "https://api.vidur.in/vidur-apis/v1/prompt-library?law_type=ALL&allowed_law_types=GST,INCOME_TAX&page_size=20&page_number=1" \
  -H "x-api-key: $VIDUR_API_KEY"
```

### Response (`200`)

```json
{
  "status": "success",
  "data": {
    "prompts": [ { "...": "prompt objects" } ],
    "total_count": 120,
    "page_size": 20,
    "page_number": 1,
    "total_pages": 6
  }
}
```

**Errors:** `400` missing `law_type` or invalid pagination · `401` missing or invalid `x-api-key`

---

## 4. Conversation Citations

`POST /vidur-apis/v1/conversations/citation`

See the sources behind each answer. Pass the `chunk` and `metadata` of a source from an ask-question response's `sources` array, and receive URLs to view (highlighted) and download (non-highlighted) the cited PDF.

### Request Body (`application/json`)

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `chunk` | object | — | Chunk of the document you want the citation for (pick from the answer `source`) |
| `metadata` | object | — | Metadata of the document (pick from the answer `source`) |

### Example

```bash
curl -X POST "https://api.vidur.in/vidur-apis/v1/conversations/citation" \
  -H "Content-Type: application/json" \
  -H "x-api-key: $VIDUR_API_KEY" \
  -d '{
    "chunk": { "...": "chunk object from ask-question sources" },
    "metadata": { "...": "metadata object from ask-question sources" }
  }'
```

### Response (`200`)

```json
{
  "data": {
    "file_url": "https://... (highlighted PDF view URL)",
    "non_highlighted_file_url": "https://... (non-highlighted PDF download URL)"
  }
}
```

**Errors:** `401` missing or invalid `x-api-key`

---

## Reference

### Law Types

Used in `ask-question` (`law_type`) and `prompt-library` (`law_type` / `allowed_law_types`):

- `INCOME_TAX`
- `GST`
- `CORPORATE_LAWS_AND_REGULATORY`
- `AUDIT_AND_ACCOUNTING`
- `OTHERS`
- `ALL_LAWS` *(ask-question only)*
- `ALL` *(prompt-library only)*
- `CONTRACTS_AND_DRAFTING`
- `CRIMINAL_LAWS`
- `BUDGET`
- `LABOUR_CODES`
- `DPDP_LAW`

### Answer Styles

Used in `ask-question` (`answer_style`):

- `VIDUR` — standard Vidur answer
- `RESEARCH`
- `UPDATES`
- `ADVISORY`
- `DRAFTING`

## Error Responses

All errors return a consistent JSON body:

```json
{
  "error": {
    "message": "Description of what went wrong"
  }
}
```

| Status | Meaning |
| --- | --- |
| `400` | Bad request — invalid or missing body / parameters |
| `401` | Unauthorized — missing or invalid `x-api-key` |
| `429` | API query quota exhausted (ask-question) |
| `500` | Server error |

---

*Interactive docs: `/vidur-apis-docs` · Spec: `/vidur-apis-docs.json` · API version 1.0.0*
