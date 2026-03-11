# 🌐 APIs

This document describes every external API that PaperClaw interacts with — what it does, how it's called, what data it returns, and what to watch out for.

---

## Table of Contents

- [Overview: Four External APIs](#overview-four-external-apis)
- [1. arXiv API](#1-arxiv-api)
- [2. Semantic Scholar API](#2-semantic-scholar-api)
- [3. Baidu Knowledge Base API](#3-baidu-knowledge-base-api)
- [4. 如流 Messaging API](#4-如流-messaging-api)
- [API Error Handling](#api-error-handling)
- [Design Rationale](#design-rationale)

---

## Overview: Four External APIs

PaperClaw talks to four external services:

| API | Purpose | Type | Auth Required? |
|-----|---------|------|---------------|
| **arXiv API** | Search for papers, download PDFs | Public | No |
| **Semantic Scholar API** | Get citation counts, author info | Public | No (optional API key) |
| **Baidu Knowledge Base** | Store reports and summaries permanently | Internal | Yes (COMATE_AUTH_TOKEN) |
| **如流 Messaging** | Send daily/weekly summaries to the team | Internal | Yes (COMATE_AUTH_TOKEN) |

```
┌──────────────────────────────────────────────────────────┐
│                     PaperClaw Agent                       │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌─────────┐  ┌──────────┐ │
│  │ arXiv    │  │ Semantic │  │ Baidu   │  │ 如流     │ │
│  │ Search   │  │ Scholar  │  │ KB      │  │ Message  │ │
│  └────┬─────┘  └────┬─────┘  └───┬─────┘  └────┬─────┘ │
└───────┼──────────────┼───────────┼──────────────┼────────┘
        │              │           │              │
        ▼              ▼           ▼              ▼
   ┌─────────┐  ┌───────────┐  ┌────────┐  ┌─────────┐
   │ arXiv   │  │ Semantic  │  │ Baidu  │  │ 如流    │
   │ .org    │  │ Scholar   │  │ Cloud  │  │ Service │
   │ (Public)│  │ (Public)  │  │(Private│  │(Private)│
   └─────────┘  └───────────┘  └────────┘  └─────────┘
```

---

## 1. arXiv API

### What is arXiv?

[arXiv](https://arxiv.org/) is the world's largest open-access repository of scientific preprints. Researchers upload their papers here before (or instead of) publishing in a journal. PaperClaw searches arXiv daily to find new papers in Scientific ML.

### Endpoint

```
GET http://export.arxiv.org/api/query
```

### How PaperClaw Uses It

**Skill:** `arxiv-search` → `scripts/search_arxiv.py`

**Called during:** Daily search (Task 3) and manual search (Task 1)

### Request Format

```
http://export.arxiv.org/api/query?search_query={query}&start=0&max_results=30&sortBy=submittedDate&sortOrder=descending
```

| Parameter | Value | Description |
|-----------|-------|-------------|
| `search_query` | arXiv query syntax | The search terms (see examples below) |
| `start` | `0` | Pagination offset |
| `max_results` | `30` | Maximum papers to return (per query) |
| `sortBy` | `submittedDate` | Sort by newest first |
| `sortOrder` | `descending` | Newest papers appear first |

### Query Syntax Examples

arXiv has its own query language. PaperClaw uses the `ti:` (title) prefix for focused searches:

```
# Search for papers with "geometry" AND ("neural" OR "operator") in the title
ti:geometry AND (ti:neural OR ti:operator OR ti:pde)

# Search for specific method names
(ti:fno OR ti:deeponet OR ti:"neural operator") AND (ti:geometry OR ti:mesh)

# Combine fields with physics applications
(ti:aerodynamic OR ti:structural) AND (ti:surrogate OR ti:neural)
```

### Response Format

The API returns **XML** (Atom feed format). PaperClaw parses it into JSON:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">
  <entry>
    <id>http://arxiv.org/abs/1910.03193v5</id>
    <title>Learning Nonlinear Operators via DeepONet...</title>
    <summary>The development of efficient surrogates...</summary>
    <author><name>Lu Lu</name></author>
    <author><name>Pengzhan Jin</name></author>
    <published>2019-10-08T17:59:59Z</published>
    <link href="http://arxiv.org/pdf/1910.03193v5" title="pdf"/>
    <arxiv:primary_category term="cs.LG"/>
  </entry>
  <!-- more entries... -->
</feed>
```

After parsing, each paper becomes:

```json
{
  "arxiv_id": "1910.03193",
  "title": "Learning Nonlinear Operators via DeepONet...",
  "summary": "The development of efficient surrogates...",
  "authors": ["Lu Lu", "Pengzhan Jin", "George Em Karniadakis"],
  "published": "2019-10-08T17:59:59Z",
  "pdf_url": "http://arxiv.org/pdf/1910.03193v5",
  "categories": ["cs.LG", "physics.comp-ph"]
}
```

### Rate Limits & Best Practices

| Consideration | Detail |
|--------------|--------|
| Rate limit | ~3-5 requests per second (unofficial, be polite) |
| Response time | 1-3 seconds per query |
| Max results | Up to 30,000 per query (PaperClaw uses 30) |
| Data freshness | New papers appear within 24 hours of submission |
| Authentication | None required |
| Cost | Free |

> **Rationale for using arXiv:** arXiv is the standard source for machine learning and physics preprints. It's free, well-maintained, and has a simple API. Papers appear here days or weeks before they're published in journals, giving PaperClaw access to the latest research.

---

## 2. Semantic Scholar API

### What is Semantic Scholar?

[Semantic Scholar](https://www.semanticscholar.org/) is an AI-powered research tool by the Allen Institute for AI. PaperClaw uses it to get citation data, which the arXiv API doesn't provide.

### Base URL

```
https://api.semanticscholar.org/graph/v1
```

### How PaperClaw Uses It

**Skill:** `semantic-scholar` → `semantic_scholar_api.py`

**Called during:** Paper evaluation (Task 2)

### Endpoints Used

#### Get Paper by arXiv ID (Primary method)

```
GET /graph/v1/paper/ARXIV:{arxiv_id}?fields=title,authors,year,publicationDate,citationCount,influentialCitationCount,venue,openAccessPdf,isOpenAccess
```

**Example:**
```
GET /graph/v1/paper/ARXIV:1910.03193?fields=title,authors,year,publicationDate,citationCount,influentialCitationCount,venue,openAccessPdf,isOpenAccess
```

**Response:**
```json
{
  "paperId": "abc123def456",
  "title": "Learning Nonlinear Operators via DeepONet...",
  "authors": [
    {"authorId": "12345", "name": "Lu Lu"},
    {"authorId": "67890", "name": "Pengzhan Jin"}
  ],
  "year": 2019,
  "publicationDate": "2019-10-08",
  "citationCount": 3144,
  "influentialCitationCount": 75,
  "venue": "Nature Machine Intelligence",
  "openAccessPdf": {
    "url": "https://arxiv.org/pdf/1910.03193"
  },
  "isOpenAccess": true
}
```

#### Search Papers by Title (Fallback method)

```
GET /graph/v1/paper/search?query={title}&fields=title,citationCount,year&limit=5
```

Used when the arXiv ID lookup fails (e.g., very new papers not yet indexed by Semantic Scholar).

#### Get Paper Citations

```
GET /graph/v1/paper/{paper_id}/citations?fields=title,year,citationCount&limit=100
```

Used to understand who is citing the paper and in what context.

#### Get Author Info

```
GET /graph/v1/author/{author_id}?fields=name,hIndex,citationCount,paperCount
```

Used to assess author reputation and research profile.

#### Batch Get Papers

```
POST /graph/v1/paper/batch
Content-Type: application/json

{
  "ids": ["ARXIV:1910.03193", "ARXIV:2010.08895"]
}
```

Used when evaluating multiple papers at once.

### Caching Strategy

PaperClaw caches Semantic Scholar responses to minimize API calls:

| Data Type | Cache Duration | Why This Duration? |
|-----------|---------------|-------------------|
| Paper data | 7 days | Paper metadata rarely changes |
| Author data | 30 days | Author profiles change slowly |
| Citation data | 1 day | Citations can change daily for popular papers |

**Cache location:** `~/.openclaw/workspace/3d_surrogate_proj/cache/semantic_scholar/`

> **Rationale for different cache durations:** Citation counts change frequently (a paper might get cited several times in one day), so they need a short cache. Paper titles and author names basically never change, so they can be cached much longer. This minimizes API calls while keeping citation data fresh.

### Rate Limits

| Tier | Limit | Description |
|------|-------|-------------|
| Without API key | 100 requests/5 minutes | Sufficient for daily use |
| With API key | 1,000 requests/5 minutes | For higher throughput |

### Key Data Fields Extracted

| Field | Used For |
|-------|---------|
| `citationCount` | Date-Citation impact calculation |
| `publicationDate` | Paper age calculation |
| `influentialCitationCount` | Quality assessment |
| `venue` | Paper prestige/metadata |
| `isOpenAccess` | Reproducibility scoring |
| `authors[].authorId` | Linking to author profiles |

---

## 3. Baidu Knowledge Base API

### What is it?

An internal (Baidu) API for creating and managing documents in a knowledge base. PaperClaw uploads weekly reports and individual paper summaries here for permanent storage and easy team access.

### How PaperClaw Uses It

**Skill:** `weekly-report` → `scripts/generate_weekly_report_v2.py`

**Called during:** Weekly report generation (Task 4)

### Operations

| Operation | Description |
|-----------|-------------|
| **Create document** | Upload a new document (paper summary or weekly report) |
| **Update document** | Modify an existing document |
| **Get document** | Retrieve a document by ID |

### Authentication

```
Header: Authorization: Bearer {COMATE_AUTH_TOKEN}
```

The token is set via environment variable `COMATE_AUTH_TOKEN`.

### Typical Flow

```
1. Generate Markdown content (summary or report)
2. Call KB API to create a new document
3. Receive document GUID and URL
4. Include URL in 如流 message
```

### Response Format

```json
{
  "guid": "doc-abc-123-def-456",
  "url": "https://kb.baidu-int.com/doc/abc123",
  "title": "Weekly Report - 2026-03-02",
  "created_at": "2026-03-02T10:00:00Z"
}
```

> **Rationale:** The Knowledge Base serves as permanent storage for reports. Unlike chat messages (which get buried), KB documents are organized, searchable, and always accessible. Team members can find any past report or paper summary at any time.

---

## 4. 如流 Messaging API

### What is 如流?

如流 (Ruliu) is an internal messaging/collaboration platform (similar to Slack or Microsoft Teams). PaperClaw sends daily summaries and weekly reports here so the team sees them proactively.

### How PaperClaw Uses It

**Skills:** `daily-search` and `weekly-report`

**Called during:** Daily search (Task 3) and weekly report (Task 4)

### Operations

| Operation | When | Content |
|-----------|------|---------|
| **Send daily summary** | After daily search | "Found 270 papers, selected 3 new ones: ..." |
| **Send weekly report** | After weekly report | "This week's Top 3 papers: ... [links to KB]" |

### Authentication

```
Header: Authorization: Bearer {COMATE_AUTH_TOKEN}
```

Uses the same token as the Knowledge Base API.

### Message Format

Messages are sent as rich text with:
- Paper titles and arXiv links
- Score summaries
- Links to Knowledge Base documents
- Statistics (papers searched, selected, evaluated)

> **Rationale:** Push notifications ensure the research team sees new papers and reports without having to check any dashboard. The combination of messaging (for proactive notification) and Knowledge Base (for permanent storage) covers both use cases.

---

## API Error Handling

### General Strategy

PaperClaw handles API errors at multiple levels:

| Error Type | How It's Handled |
|-----------|-----------------|
| **Network timeout** | Retry with exponential backoff |
| **Rate limit (429)** | Wait and retry |
| **Not found (404)** | Fall back to alternative method (e.g., title search instead of ID lookup) |
| **Server error (500)** | Retry up to 3 times, then log error and continue |
| **Invalid response** | Log warning, skip this data point |

### arXiv-specific

- If a query returns 0 results, the system continues with the next query
- XML parsing errors are caught and logged

### Semantic Scholar-specific

- If arXiv ID lookup fails, falls back to title search
- If citation count is unavailable, defaults to 0
- Cache is used even when API is down (stale data is better than no data)

---

## Design Rationale

### Why use two public APIs and two private APIs?

| API | Choice | Reason |
|-----|--------|--------|
| arXiv | Only choice | The primary source for ML/physics preprints |
| Semantic Scholar | Over Google Scholar | Has a free, well-documented REST API; Google Scholar has no official API |
| Baidu KB | Organizational choice | Integrated with the team's existing knowledge management system |
| 如流 | Organizational choice | Integrated with the team's existing communication platform |

### Why not use Google Scholar?

Google Scholar doesn't have an official API. Third-party scrapers (like SerpAPI) are available but:
- Expensive at scale
- Can break when Google changes their page layout
- Violate Google's terms of service

Semantic Scholar provides the same citation data through a proper, stable API — for free.

### Why cache Semantic Scholar data?

Three reasons:
1. **Speed**: Avoid network round-trips for data that hasn't changed
2. **Reliability**: If the API is down, cached data keeps the system working
3. **Courtesy**: Reduces load on a free public API

### Why push messages instead of a dashboard?

Researchers are busy. They check their messaging app far more often than any dashboard. By pushing summaries to 如流, PaperClaw meets researchers where they already are.

---

## Environment Variables for APIs

| Variable | Required | Used For |
|----------|----------|----------|
| `COMATE_AUTH_TOKEN` | Yes | Baidu Knowledge Base + 如流 Messaging authentication |
| `BAIDU_API_KEY` | Yes | Baidu Cloud API access |
| `SERPAPI_KEY` | No | Optional Google Scholar access (not actively used) |

---

## Next Steps

- **[Skills Deep Dive](skills-deep-dive.md)** — See how each skill calls these APIs
- **[Data Schemas](data-schemas.md)** — See the data structures populated by these APIs
- **[Data Flow](data-flow.md)** — Follow API calls through complete workflows

---

<div align="center">

[← Back to Documentation Index](README.md)

</div>
