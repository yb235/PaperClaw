# 🔄 Data Flow

This document traces how data moves through PaperClaw from start to finish. We'll follow three real workflows step by step, showing exactly what happens at each stage.

---

## Table of Contents

- [Workflow 1: Daily Automated Search](#workflow-1-daily-automated-search)
- [Workflow 2: Paper Evaluation](#workflow-2-paper-evaluation)
- [Workflow 3: Weekly Report Generation](#workflow-3-weekly-report-generation)
- [The Central Registry](#the-central-registry)
- [Data Lifecycle](#data-lifecycle)

---

## Workflow 1: Daily Automated Search

**Trigger:** Every day at 20:00 (Asia/Singapore), the cron scheduler sends a message to the agent.

Here's what happens, step by step:

### Step 1: Batch Search arXiv

```
┌────────────────────────────────────┐
│  9 keyword queries are sent to     │
│  the arXiv API, one at a time      │
│                                    │
│  Query 1: geometry + neural + PDE  │──→ arXiv API ──→ 30 results
│  Query 2: mesh + neural + deep     │──→ arXiv API ──→ 30 results
│  Query 3: cfd + surrogate + neural │──→ arXiv API ──→ 30 results
│  ...                               │
│  Query 9: transformer + pde        │──→ arXiv API ──→ 30 results
│                                    │
│  Total: ~270 raw results           │
└────────────────────────────────────┘
```

**Data at this point:** A list of ~270 paper objects, each with:
- arXiv ID (e.g., `2401.12345`)
- Title
- Abstract
- Authors
- Publication date
- PDF URL
- Categories

### Step 2: Deduplicate Within Search Results

```
~270 papers
    │
    ▼
┌────────────────────────────────────┐
│  DEDUPLICATION (Layer 1 of 3)      │
│                                    │
│  Check 1: Same arXiv ID?          │──→ Remove duplicate
│  Check 2: Same normalized title?   │──→ Remove duplicate
│  Check 3: Matches exclusion list?  │──→ Remove (wrong field)
│                                    │
│  Normalization:                    │
│  - Lowercase all titles            │
│  - Remove extra whitespace         │
│  - BUT keep version markers        │
│    like "++", "-2" (these are      │
│    different papers!)              │
└────────────────────────────────────┘
    │
    ▼
~200-250 unique papers
```

> **Why normalize titles?** Because the same paper might appear with slightly different formatting across queries. But "MethodName++" and "MethodName" are genuinely different papers — hence the careful preservation of version markers.

### Step 3: Cross-Reference with Registry

```
~200-250 unique papers
    │
    ▼
┌────────────────────────────────────┐
│  DEDUPLICATION (Layer 2 of 3)      │
│                                    │
│  Load evaluated_papers.json        │
│                                    │
│  For each paper:                   │
│    If arxiv_id exists in registry  │──→ Skip (already reviewed)
│    If normalized title matches     │──→ Skip (already reviewed)
│                                    │
│  Log skipped papers with reason:   │
│    "ID已评估" or "标题已评估"      │
└────────────────────────────────────┘
    │
    ▼
~10-50 genuinely new papers
```

> **Why is this number so small?** Because PaperClaw runs daily. After a few weeks, most papers in the domain have already been seen. This is actually a sign the system is working well.

### Step 4: Relevance Scoring

```
~10-50 new papers
    │
    ▼
┌────────────────────────────────────┐
│  RELEVANCE SCORING                 │
│                                    │
│  For each paper, score based on:   │
│                                    │
│  Title keywords:                   │
│    Geometry words    → +15 points  │
│    Neural op words   → +12 points  │
│    PDE words         → +10 points  │
│    Application words → +8 points   │
│    Technical words   → +5 points   │
│    Negative words    → -20 points  │
│                                    │
│  Abstract keywords:                │
│    Geometry words    → +5 points   │
│    Neural op words   → +4 points   │
│    PDE words         → +3 points   │
│    Application words → +3 points   │
│    Technical words   → +2 points   │
│    Negative words    → -5 points   │
│                                    │
│  Sort by total score (desc)        │
└────────────────────────────────────┘
    │
    ▼
Ranked list of new papers
```

> **Why weight title keywords more than abstract keywords?** If a keyword appears in the title, the paper is almost certainly about that topic. In the abstract, it might just be a passing mention. The 3× weighting (15 vs 5) reflects this difference.

### Step 5: Select Top 3 & Download

```
Ranked list
    │
    ▼
┌────────────────────────────────────┐
│  SELECT TOP 3                      │
│                                    │
│  Take the 3 highest-scoring papers │
│                                    │
│  For each selected paper:          │
│                                    │
│  1. Generate a safe folder name    │
│     "A Long Paper Title About..."  │
│     → "ALongPaperTitle"            │
│                                    │
│  2. Create folder:                 │
│     papers/ALongPaperTitle/        │
│                                    │
│  3. Download PDF:                  │
│     papers/ALongPaperTitle/*.pdf   │
│                                    │
│  4. Create metadata.json:          │
│     papers/ALongPaperTitle/        │
│     metadata.json                  │
└────────────────────────────────────┘
```

### Step 6: Save Logs & Notify

```
┌────────────────────────────────────┐
│  SAVE SEARCH LOG                   │
│                                    │
│  Write to:                         │
│  search_logs/YYYY-MM-DD_log.json   │
│                                    │
│  Contains:                         │
│  - Total searched (270)            │
│  - After dedup (245)              │
│  - Skipped evaluated (200)         │
│  - Selected count (3)             │
│  - Details of selected papers      │
│  - Details of why papers were      │
│    skipped                         │
└────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────┐
│  SEND 如流 MESSAGE                 │
│                                    │
│  "今日检索了 270 篇论文，          │
│   去重后 245 篇，                  │
│   精选 3 篇新论文..."             │
│                                    │
│  + links to each paper             │
└────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────┐
│  GENERATE TASK LIST                │
│                                    │
│  Write to:                         │
│  pending_evaluation_YYYY-MM-DD.json│
│                                    │
│  Contains 3 tasks:                 │
│  - Paper 1: {id, title, dir, ...}  │
│  - Paper 2: ...                    │
│  - Paper 3: ...                    │
│                                    │
│  Agent reads this and proceeds     │
│  to evaluate each paper (Task 2)   │
└────────────────────────────────────┘
```

### Complete Daily Search Data Flow Diagram

```
         arXiv API
            │
            ▼
    ┌───────────────┐
    │ ~270 raw      │
    │ papers (XML)  │
    └───────┬───────┘
            │ Parse & Deduplicate
            ▼
    ┌───────────────┐     ┌────────────────────────┐
    │ ~245 unique   │────▶│ evaluated_papers.json   │
    │ papers        │     │ (cross-reference)       │
    └───────┬───────┘     └────────────────────────┘
            │ Filter already-reviewed
            ▼
    ┌───────────────┐
    │ ~30 new       │
    │ papers        │
    └───────┬───────┘
            │ Score relevance
            ▼
    ┌───────────────┐
    │ Top 3 papers  │
    └───┬───┬───┬───┘
        │   │   │
        ▼   ▼   ▼
    ┌─────────────────────────┐
    │ papers/{title}/         │
    │   ├── paper.pdf         │
    │   └── metadata.json     │
    └─────────────────────────┘
        │           │
        ▼           ▼
    ┌────────┐  ┌──────────────────────┐
    │ search │  │ pending_evaluation_  │
    │ _log   │  │ YYYY-MM-DD.json     │
    │ .json  │  │ (task queue)        │
    └────────┘  └──────────────────────┘
        │
        ▼
    如流 message
    (daily summary)
```

---

## Workflow 2: Paper Evaluation

**Trigger:** After the daily search (automated) or by user request (manual).

### Step 1: Check for Duplicates

```
Paper to evaluate (arXiv ID + title)
    │
    ▼
┌────────────────────────────────────┐
│  DEDUPLICATION (Layer 3 of 3)      │
│                                    │
│  Load evaluated_papers.json        │
│  Check if arXiv ID exists          │
│  Check if normalized title exists  │
│                                    │
│  If duplicate → Stop. Skip paper.  │
│  If new → Continue.                │
└────────────────────────────────────┘
```

> **Why check again?** Because between the daily search (Step 3 above) and the evaluation, another process might have already evaluated the same paper. This is the safety net — Layer 3 of the triple deduplication strategy.

### Step 2: Fetch Citation Data

```
arXiv ID
    │
    ▼
┌────────────────────────────────────┐
│  SEMANTIC SCHOLAR API              │
│                                    │
│  GET /graph/v1/paper/ARXIV:{id}    │
│                                    │
│  Request fields:                   │
│  - title, authors, year            │
│  - publicationDate, venue          │
│  - citationCount                   │
│  - influentialCitationCount        │
│  - openAccessPdf                   │
│  - isOpenAccess                    │
│                                    │
│  Check cache first:                │
│  - Cache hit? Return cached data   │
│  - Cache miss? Call API, cache it  │
│                                    │
│  Cache TTL:                        │
│  - Paper data: 7 days              │
│  - Citation data: 1 day            │
│  - Author data: 30 days            │
└────────────────────────────────────┘
    │
    ▼
Citation metadata (JSON)
```

### Step 3: Generate Summary

```
Paper PDF + Citation metadata
    │
    ▼
┌────────────────────────────────────┐
│  LLM SUMMARY GENERATION           │
│                                    │
│  The LLM reads the full paper and  │
│  answers 10 research questions:    │
│                                    │
│  1. What problem is solved?        │
│  2. Is it a new problem?           │
│  3. What's the hypothesis?         │
│  4. Related work & key people?     │
│  5. Solution approach?             │
│  6. Experiment design?             │
│  7. Datasets? Open source?         │
│  8. Do results support hypothesis? │
│  9. Main contributions?            │
│  10. Future work?                  │
└────────────────────────────────────┘
    │
    ▼
summary.md saved to papers/{title}/
```

### Step 4: Score the Paper

```
Paper + summary + citation data
    │
    ▼
┌────────────────────────────────────────────────────────┐
│  4-DIMENSIONAL SCORING (by LLM with chain-of-thought)  │
│                                                        │
│  <think>                                               │
│  This paper introduces a new mesh-aware operator...    │
│  The engineering value is strong because...            │
│  </think>                                              │
│                                                        │
│  Score 1: Engineering Application Value    → 8/10      │
│  Score 2: Architecture Innovation          → 7/10      │
│  Score 3: Theoretical Contribution         → 6/10      │
│  Score 4: Result Reliability               → 8/10      │
│                                                        │
│  Four-Dim Average = (8 + 7 + 6 + 8) / 4 = 7.25       │
└────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────┐
│  DATE-CITATION IMPACT CALCULATION                      │
│                                                        │
│  Paper age: 4 months                                   │
│  Citations: 25                                         │
│  → Age category: Medium (3-24 months)                  │
│  → Citation tier: 20-49 → adjustment = +0.3            │
│  → Citation density: 25/4 = 6.25/month                 │
│  → Density bonus: +0.1 (5-10/month tier)              │
│  → Total adjustment: 0.3 + 0.1 = 0.4                  │
│  → Impact score: base + 0.4 (capped at 10.0)          │
└────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────┐
│  FINAL SCORE                                           │
│                                                        │
│  Final = 7.25 × 0.9 + impact × 0.1                    │
└────────────────────────────────────────────────────────┘
    │
    ▼
scores.md saved to papers/{title}/
metadata.json updated with all scores
```

### Step 5: Update Registry

```
Paper scores + metadata
    │
    ▼
┌────────────────────────────────────┐
│  THREAD-SAFE REGISTRY UPDATE       │
│                                    │
│  1. Acquire file lock (fcntl)      │
│  2. Read evaluated_papers.json     │
│  3. Check for duplicates (again!)  │
│  4. Append new paper entry         │
│  5. Write back to file             │
│  6. Release lock                   │
└────────────────────────────────────┘
    │
    ▼
evaluated_papers.json now contains
this paper's entry
```

### Complete Evaluation Data Flow Diagram

```
    Paper PDF                    arXiv ID
        │                           │
        │                           ▼
        │                   Semantic Scholar API
        │                           │
        │                           ▼
        │                    Citation Data
        │                    (cached 1-7 days)
        │                           │
        ▼                           ▼
    ┌─────────────────────────────────┐
    │         LLM Processing          │
    │                                 │
    │  Read PDF + citation data       │
    │  → Generate summary.md          │
    │  → Think through scoring        │
    │  → Generate scores.md           │
    │  → Calculate final score        │
    └─────────┬─────────┬─────────┬───┘
              │         │         │
              ▼         ▼         ▼
         summary.md  scores.md  metadata.json
              │         │         │
              └─────────┼─────────┘
                        │
                        ▼
              evaluated_papers.json
              (thread-safe update)
```

---

## Workflow 3: Weekly Report Generation

**Trigger:** Every Sunday at 10:00 (Asia/Singapore)

### Step-by-step flow

```
Sunday 10:00 trigger
    │
    ▼
┌────────────────────────────────────┐
│  LOAD EVALUATED PAPERS             │
│                                    │
│  Read evaluated_papers.json        │
│  (the central registry of all      │
│   evaluated papers)                │
└────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────┐
│  FILTER TO THIS WEEK               │
│                                    │
│  Keep only papers where:           │
│  evaluated_date >= (now - 7 days)  │
│                                    │
│  Typically 15-21 papers/week       │
│  (3 per day × 7 days)             │
└────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────┐
│  SORT & SELECT TOP 3               │
│                                    │
│  Sort by final_score (descending)  │
│  Take the top 3 papers             │
└────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────┐
│  FOR EACH TOP 3 PAPER:             │
│                                    │
│  1. Read summary.md from disk      │
│  2. Read scores.md from disk       │
│  3. Upload to Baidu Knowledge Base │
│     → Get document GUID and URL    │
└────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────┐
│  GENERATE WEEKLY REPORT            │
│                                    │
│  Create Markdown file:             │
│  # Weekly Report - YYYY-MM-DD      │
│  ## Top 3 Papers                   │
│  1. Paper A (score: 8.5) [link]    │
│  2. Paper B (score: 8.2) [link]    │
│  3. Paper C (score: 7.9) [link]    │
│  ## Full Scoring Table             │
│  [All papers ranked]               │
│  ## Appendix                       │
│  [Links to KB documents]           │
└────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────┐
│  UPLOAD & NOTIFY                   │
│                                    │
│  1. Upload report to Knowledge Base│
│  2. Send 如流 message with:        │
│     - Summary text                 │
│     - Links to KB documents        │
│     - Links to individual papers   │
└────────────────────────────────────┘
```

### Complete Weekly Report Data Flow Diagram

```
evaluated_papers.json
    │
    │ Filter (past 7 days)
    ▼
┌─────────────────────┐
│ This week's papers   │
│ (15-21 papers)       │
└──────────┬──────────┘
           │ Sort by final_score
           ▼
┌─────────────────────┐
│ Top 3 papers        │
└──┬──────┬──────┬────┘
   │      │      │
   ▼      ▼      ▼
 Read   Read   Read
 summary summary summary
 .md    .md    .md
   │      │      │
   ▼      ▼      ▼
 Upload Upload Upload
 to KB  to KB  to KB
   │      │      │
   └──────┼──────┘
          │
          ▼
   Weekly Report (Markdown)
          │
          ├──→ Upload to KB
          │
          └──→ 如流 message
               (with links)
```

---

## The Central Registry

**File:** `evaluated_papers.json`

This is the most important data file in the system. Every workflow reads from or writes to it.

```
                  ┌──────────────────────────────┐
                  │   evaluated_papers.json       │
                  │                              │
                  │   The single source of truth │
                  │   for all evaluated papers   │
                  └──────┬───────────┬───────────┘
                         │           │
              ┌──────────┘           └──────────┐
              │                                  │
         WRITTEN BY                        READ BY
              │                                  │
    ┌─────────┴─────────┐          ┌─────────────┴────────────┐
    │ paper-review skill │          │ daily-search skill       │
    │ (update_registry.  │          │ (cross-database dedup)   │
    │  py)               │          │                          │
    └────────────────────┘          │ weekly-report skill      │
                                    │ (filter this week's      │
                                    │  papers)                 │
                                    └──────────────────────────┘
```

> **Rationale:** Having a single registry file instead of scattered metadata files means:
> 1. **One place to check** for deduplication
> 2. **One place to query** for report generation
> 3. **One file to back up** to protect all evaluation data
> 4. **Thread-safe writes** only need to lock one file

---

## Data Lifecycle

Here's the lifecycle of a paper from discovery to weekly report:

```
Stage 1: DISCOVERED
├── Found by daily search
├── Passed deduplication
├── Scored for relevance
└── Stored: papers/{title}/metadata.json (basic info only)

Stage 2: DOWNLOADED
├── PDF downloaded from arXiv
└── Stored: papers/{title}/*.pdf

Stage 3: SUMMARIZED
├── LLM reads the PDF
├── 10 research questions answered
└── Stored: papers/{title}/summary.md

Stage 4: EVALUATED
├── 4-dimensional scoring complete
├── Date-Citation impact calculated
├── Final score computed
├── Stored: papers/{title}/scores.md
├── Stored: papers/{title}/metadata.json (updated with scores)
└── Stored: evaluated_papers.json (registry entry added)

Stage 5: REPORTED
├── Included in weekly report (if in Top 3)
├── Uploaded to Knowledge Base
└── Sent to team via 如流
```

---

## Next Steps

- **[Data Schemas](data-schemas.md)** — See the exact structure of every JSON and Markdown file
- **[APIs](apis.md)** — Learn about the external APIs used in each workflow
- **[Skills Deep Dive](skills-deep-dive.md)** — Explore the code behind each workflow step

---

<div align="center">

[← Back to Documentation Index](README.md)

</div>
