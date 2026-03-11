# 🔧 Skills Deep Dive

This document takes you inside each of PaperClaw's five skill modules. For each skill, we'll explain what it does, how it works, the key functions in its code, and the design decisions behind it.

---

## Table of Contents

- [Overview: The Five Skills](#overview-the-five-skills)
- [Skill 1: arXiv Search](#skill-1-arxiv-search)
- [Skill 2: Paper Review](#skill-2-paper-review)
- [Skill 3: Daily Search](#skill-3-daily-search)
- [Skill 4: Semantic Scholar](#skill-4-semantic-scholar)
- [Skill 5: Weekly Report](#skill-5-weekly-report)
- [How Skills Interact](#how-skills-interact)
- [Adding a New Skill](#adding-a-new-skill)

---

## Overview: The Five Skills

| Skill | Folder | Purpose | Trigger |
|-------|--------|---------|---------|
| **arXiv Search** | `skills/arxiv-search/` | Search arXiv API, filter, and rank papers | Called by Daily Search or user |
| **Paper Review** | `skills/paper-review/` | Summarize paper, score on 4 dimensions, update registry | Called after search or by user |
| **Daily Search** | `skills/daily-search/` | Orchestrate the full daily pipeline | Cron at 20:00 daily |
| **Semantic Scholar** | `skills/semantic-scholar/` | Fetch citation data from Semantic Scholar API | Called by Paper Review |
| **Weekly Report** | `skills/weekly-report/` | Generate Top 3 report, upload to KB, notify team | Cron on Sundays at 10:00 |

---

## Skill 1: arXiv Search

**Location:** `skills/arxiv-search/`

**Files:**
- `SKILL.md` — Documentation
- `scripts/search_arxiv.py` — Implementation

### What it does

This skill is the "search engine" of PaperClaw. It takes a query (or uses the 9 predefined queries), sends it to the arXiv API, and returns a filtered, deduplicated, relevance-ranked list of papers.

### Key functions

```python
def search_arxiv(query, max_results=30):
    """
    Send a single query to arXiv API and parse results.
    
    Input:  query string (arXiv query syntax)
    Output: list of paper dicts with id, title, abstract, authors, etc.
    
    What it does:
    1. URL-encode the query
    2. Send GET request to arXiv API
    3. Parse XML response
    4. Extract paper fields
    5. Return list of paper dicts
    """

def batch_search(max_results_per_query=30):
    """
    Run all 9 predefined keyword queries in sequence.
    
    Input:  none (queries are hardcoded)
    Output: combined list of papers from all 9 queries (~270 papers)
    
    What it does:
    1. Iterate through 9 query patterns
    2. Call search_arxiv() for each
    3. Combine all results into one list
    """

def normalize_title(title):
    """
    Standardize a paper title for deduplication comparison.
    
    Input:  "  A Paper Title About Neural Operators  v2  "
    Output: "a paper title about neural operators v2"
    
    What it does:
    1. Strip leading/trailing whitespace
    2. Convert to lowercase
    3. Collapse multiple spaces
    4. PRESERVE version markers like "++", "-2", "v2"
       (because "MethodName++" is a different paper than "MethodName")
    """

def deduplicate_papers(papers):
    """
    Remove duplicate papers from a list.
    
    Input:  list of ~270 papers (may contain duplicates across queries)
    Output: list of ~245 unique papers
    
    What it does:
    1. Build a set of seen arXiv IDs
    2. Build a set of seen normalized titles
    3. For each paper: skip if ID or title already seen
    4. Also skip if paper matches exclusion keywords
    """

def score_paper_relevance(paper):
    """
    Calculate how relevant a paper is to the SciML domain.
    
    Input:  single paper dict (with title and abstract)
    Output: integer relevance score (higher = more relevant)
    
    Scoring weights:
    
    Category          | Title Match | Abstract Match
    ------------------|-------------|---------------
    Geometry keywords | +15 pts     | +5 pts
    Neural op words   | +12 pts     | +4 pts
    PDE keywords      | +10 pts     | +3 pts
    Application words | +8 pts      | +3 pts
    Technical words   | +5 pts      | +2 pts
    Negative keywords | -20 pts     | -5 pts
    """
```

### Why title keywords are weighted 3× more than abstract keywords

If a keyword appears in the **title**, the paper is almost certainly about that topic — the title is the most concentrated summary of a paper's content. If it appears in the **abstract**, it might just be a passing reference or comparison. The 3× weighting (15 vs 5 for geometry keywords) reflects this confidence difference.

### Why preserve version markers during normalization

In machine learning, "MethodName++" and "MethodName" are genuinely different papers (the "++" typically means an improved version). Similarly, "Method v2" and "Method v3" are different papers. Naive normalization that strips punctuation would merge these incorrectly.

### Why negative keywords have such high penalties (-20 in title)

A paper titled "Neural Network for **Disease** Modeling" is definitely not about physics surrogates, even if it contains words like "neural" and "modeling." The -20 penalty ensures these papers drop to the bottom of the ranking immediately, regardless of how many positive keywords they match.

---

## Skill 2: Paper Review

**Location:** `skills/paper-review/`

**Files:**
- `SKILL.md` — Documentation (includes the 10-question framework and 4-dimensional scoring criteria)
- `scripts/update_registry.py` — Thread-safe registry update tool

### What it does

This skill handles the entire evaluation pipeline for a single paper: checking for duplicates, fetching citation data, generating a summary, scoring on 4 dimensions, calculating impact, and updating the central registry.

### The SKILL.md as workflow instructions

Unlike other skills where the `.py` script does most of the work, the Paper Review skill relies heavily on its `SKILL.md` to guide the **LLM**. The Markdown file contains:

1. **Deduplication procedure** — Step-by-step instructions for the agent to check if a paper has already been evaluated
2. **10-question summary template** — The exact questions the LLM should answer
3. **4-dimensional scoring rubric** — Detailed criteria for each score level (1-10) across all 4 dimensions
4. **Date-Citation calculation rules** — The exact formula for impact adjustment
5. **Output format specifications** — Where and how to save results

> **Rationale:** The evaluation is primarily an LLM task (reading a paper, understanding it, scoring it). The Python script handles only the mechanical parts (file locking, JSON writing). The SKILL.md is essentially a "prompt template" that ensures consistent, high-quality evaluations.

### Key function: update_registry.py

```python
def update_registry(arxiv_id, title, short_title, final_score):
    """
    Safely add a paper to the evaluated_papers.json registry.
    
    Input:  paper identification and score
    Output: True if added, False if duplicate
    
    Safety features:
    
    1. FILE LOCKING (Unix):
       - Uses fcntl.flock(LOCK_EX) for exclusive file lock
       - Prevents two processes from writing simultaneously
       - Falls back gracefully on Windows (no locking)
    
    2. DUPLICATE CHECKING:
       - Checks arXiv ID against all existing entries
       - Checks normalized title against all existing entries
       - Rejects if either matches
    
    3. ATOMIC WRITE:
       - Reads the entire file
       - Modifies in memory
       - Seeks to beginning
       - Writes back
       - Truncates any leftover data
    """
```

### Why file locking?

The daily search can trigger evaluation of 3 papers almost simultaneously. Without locking, two processes could:

1. Both read `evaluated_papers.json`
2. Both add their paper to their in-memory copy
3. Both write back — second write overwrites the first paper's entry

File locking prevents this race condition by ensuring only one process can write at a time.

### Why check for duplicates inside the locked section?

Even with the triple-layer deduplication (search-level, cross-database, write-time), there's still a tiny window where two processes could pass the first two checks for the same paper. Checking again inside the file lock is the final safety net.

---

## Skill 3: Daily Search

**Location:** `skills/daily-search/`

**Files:**
- `SKILL.md` — Documentation
- `scripts/daily_paper_search.py` — Implementation

### What it does

This is the **orchestrator** — it coordinates the entire daily search pipeline. It calls arXiv Search internally, handles cross-database deduplication, manages PDF downloads, generates task lists, and sends notifications.

### Key class: DailyPaperSearcher

```python
class DailyPaperSearcher:
    """
    Orchestrates the complete daily paper discovery pipeline.
    """
    
    def load_evaluated_papers(self):
        """
        Load the evaluated_papers.json registry.
        Returns a dict with 'papers' list.
        Creates the file if it doesn't exist.
        """
    
    def filter_against_evaluated(self, papers):
        """
        Cross-reference new papers against already-evaluated ones.
        
        Input:  list of papers from arXiv search
        Output: list of papers NOT in the registry
        
        Checks both arXiv ID and normalized title.
        Logs why each paper was filtered (for audit trail).
        """
    
    def download_pdf(self, paper, save_dir):
        """
        Download a paper's PDF from arXiv.
        
        Input:  paper dict (with pdf_url) + save directory
        Output: path to saved PDF, or None on failure
        
        Creates the directory if it doesn't exist.
        Uses the paper title as filename.
        """
    
    def create_paper_metadata(self, paper, pdf_path):
        """
        Create the initial metadata.json for a paper.
        
        Input:  paper dict + path to downloaded PDF
        Output: metadata.json written to paper's directory
        
        Note: This creates the INITIAL metadata without scores.
        Scores are added later by the Paper Review skill.
        """
    
    def generate_short_title(self, title):
        """
        Generate a filesystem-safe folder name from a paper title.
        
        Input:  "A Long Paper Title About Neural Operators for 3D Geometry"
        Output: "ALongPaperTitleAboutNeuralOp"
        
        Rules:
        - Remove special characters
        - CamelCase words together
        - Truncate to reasonable length
        - Ensure uniqueness
        """
    
    def save_search_log(self, results, selected):
        """
        Save detailed search log for auditing.
        
        Writes: search_logs/YYYY-MM-DD_search_log.json
        Contains: total searched, dedup stats, selected papers, skip reasons
        """
    
    def generate_evaluation_task(self, papers):
        """
        Create a task list for the agent to evaluate selected papers.
        
        Writes: pending_evaluation_YYYY-MM-DD.json
        The agent reads this and evaluates each paper in order.
        """
    
    def send_daily_summary(self, stats, papers):
        """
        Send a summary message to 如流.
        
        Contains: search statistics + selected paper titles + arXiv links
        """
```

### The "deferred evaluation" pattern

The daily search skill finds papers but **doesn't evaluate them immediately**. Instead, it creates a task list (`pending_evaluation_YYYY-MM-DD.json`) that the agent picks up and processes.

Why not evaluate immediately?

1. **Separation of concerns** — searching and evaluating are different tasks with different skills
2. **Human review** — a researcher could review the task list before evaluation starts
3. **Failure isolation** — if evaluation fails for one paper, it doesn't affect others
4. **Parallelism** — in theory, multiple agent instances could evaluate papers concurrently

---

## Skill 4: Semantic Scholar

**Location:** `skills/semantic-scholar/`

**Files:**
- `SKILL.md` — Documentation
- `semantic_scholar_api.py` — Implementation (API client with caching)

### What it does

This skill is a wrapper around the Semantic Scholar API. It fetches citation data, author information, and publication metadata that arXiv doesn't provide.

### Key functions

```python
def get_paper_by_arxiv(arxiv_id):
    """
    PRIMARY METHOD: Look up a paper by its arXiv ID.
    
    Input:  "1910.03193"
    Output: dict with citation count, venue, authors, etc.
    
    Checks cache first (7-day TTL for paper data).
    If cache miss, calls Semantic Scholar API.
    """

def get_paper_by_title(title):
    """
    FALLBACK METHOD: Search for a paper by title.
    
    Input:  "Learning Nonlinear Operators via DeepONet"
    Output: best-matching paper dict
    
    Used when arXiv ID lookup fails (e.g., very new papers
    not yet indexed by Semantic Scholar).
    """

def get_paper_by_id(paper_id):
    """
    Look up a paper by Semantic Scholar's own internal ID.
    
    Input:  "abc123def456"
    Output: paper dict
    
    Used when you already have the Semantic Scholar ID
    (e.g., from a previous API call).
    """

def get_author(author_id):
    """
    Get author profile information.
    
    Input:  "12345"
    Output: dict with name, h-index, citation count, paper count
    
    Cache TTL: 30 days (author profiles change slowly)
    """

def get_paper_citations(paper_id, limit=100):
    """
    Get papers that cite a given paper.
    
    Input:  paper_id + max results
    Output: list of citing paper dicts
    
    Cache TTL: 1 day (citations change frequently for popular papers)
    """

def batch_get_papers(paper_ids):
    """
    Fetch multiple papers in a single API call.
    
    Input:  list of paper IDs (arXiv or Semantic Scholar format)
    Output: list of paper dicts
    
    More efficient than calling get_paper_by_id() in a loop.
    """
```

### Caching architecture

```
cache/semantic_scholar/
├── paper_{arxiv_id}.json         # Paper data, TTL: 7 days
├── author_{author_id}.json       # Author data, TTL: 30 days
└── citations_{paper_id}.json     # Citation data, TTL: 1 day
```

Each cache file stores:
```json
{
  "data": { /* actual API response */ },
  "cached_at": "2026-03-01T10:00:00Z",
  "ttl_days": 7
}
```

> **Rationale for different TTLs:**
> - **Paper data (7 days):** A paper's title, authors, and venue never change. Citation count changes slowly (a few per week for most papers).
> - **Author data (30 days):** Author h-index and publication count change even more slowly.
> - **Citation data (1 day):** The list of *who cites a paper* can change daily for popular papers. Fresh data is important for accurate impact assessment.

### Why arXiv ID lookup is the primary method

Semantic Scholar indexes most arXiv papers and supports lookup by arXiv ID (`ARXIV:1910.03193`). This is more reliable than title search because:
- Titles can have slight variations
- Title search might return wrong papers with similar names
- arXiv ID is a unique identifier — no ambiguity

---

## Skill 5: Weekly Report

**Location:** `skills/weekly-report/`

**Files:**
- `SKILL.md` — Documentation
- `scripts/generate_weekly_report_v2.py` — Implementation

### What it does

Every Sunday morning, this skill collects all papers evaluated in the past week, ranks them, picks the Top 3, generates a Markdown report, uploads it to the Knowledge Base, and notifies the team.

### Key functions

```python
def load_evaluated_papers():
    """
    Load the central registry (evaluated_papers.json).
    
    Output: list of all evaluated paper entries
    """

def filter_week_papers(papers, days=7):
    """
    Filter to papers evaluated in the past N days.
    
    Input:  all papers + number of days to look back
    Output: only papers where evaluated_date >= (now - days)
    
    Typically returns 15-21 papers (3/day × 7 days).
    """

def sort_and_select_top(papers, top_n=3):
    """
    Sort papers by final_score and select the top N.
    
    Input:  this week's papers + how many to select
    Output: top N papers, sorted by score descending
    """

def generate_weekly_report(week_papers, top_papers):
    """
    Generate a complete Markdown weekly report.
    
    Input:  all week's papers + top papers
    Output: Markdown string with:
      - Header with date range and statistics
      - Top 3 section with highlights
      - Complete ranking table
      - Appendix with KB links
    """

def create_knowledge_base_docs(top_papers):
    """
    Upload each top paper's full summary to Baidu Knowledge Base.
    
    Input:  top 3 paper dicts (with summary.md paths)
    Output: list of KB document GUIDs and URLs
    
    For each paper:
    1. Read summary.md from disk
    2. Read scores.md from disk
    3. Combine into a single document
    4. Upload to KB
    5. Store returned GUID and URL
    """

def send_weekly_report(report, recipients):
    """
    Send the weekly report to 如流.
    
    Input:  report text + recipient usernames
    Output: 如流 message with report content and KB links
    """
```

### Why the script is called "v2"

The original version (`generate_weekly_report.py`, now replaced) read scores from `scores.md` using regex patterns like:

```python
# Old fragile approach:
match = re.search(r'最终综合评分.*?(\d+\.?\d*)/10', scores_text)
```

This broke whenever the LLM generated slightly different formatting. The v2 version reads from `metadata.json` instead:

```python
# New reliable approach:
score = metadata['scores']['final_score']
```

> **Rationale:** This is a core design lesson: never parse human-readable documents programmatically when structured data is available. The migration to v2 eliminated a class of bugs and made the weekly report generation much more reliable.

---

## How Skills Interact

Here's the dependency graph showing how skills connect:

```
┌──────────────┐
│ daily-search │ ─── Orchestrates everything daily
└──────┬───────┘
       │ calls
       ▼
┌──────────────┐
│ arxiv-search │ ─── Searches arXiv for papers
└──────┬───────┘
       │ results feed into
       ▼
┌──────────────┐     ┌──────────────────┐
│ paper-review │ ←── │ semantic-scholar │
│              │     │ (citation data)  │
└──────┬───────┘     └──────────────────┘
       │ writes to evaluated_papers.json
       ▼
┌──────────────┐
│weekly-report │ ─── Reads evaluated_papers.json
└──────────────┘
```

**Communication method:** Files on disk (not direct function calls).

| Producer | File | Consumer |
|----------|------|----------|
| daily-search | `pending_evaluation_*.json` | Agent → paper-review |
| paper-review | `evaluated_papers.json` | daily-search (dedup), weekly-report |
| paper-review | `metadata.json` | weekly-report |
| paper-review | `summary.md` | weekly-report |

---

## Adding a New Skill

Want to add a new skill? Follow this pattern:

### Step 1: Create the folder structure

```
skills/
└── my-new-skill/
    ├── SKILL.md           # Required: Documentation + agent instructions
    └── scripts/
        └── my_script.py   # Required: Implementation
```

### Step 2: Write SKILL.md

The SKILL.md should include:
- **What the skill does** (plain English description)
- **When to use it** (trigger conditions)
- **Inputs** (what data it needs)
- **Outputs** (what files/data it produces)
- **Step-by-step workflow** (for the agent to follow)
- **Error handling** (what to do when things go wrong)

### Step 3: Implement the script

- Use Python 3.8+ compatible code
- Handle errors gracefully (log warnings, don't crash)
- Cache API responses when appropriate
- Write outputs in JSON (for machines) or Markdown (for humans)

### Step 4: Reference in AGENT.md

Add the skill to the agent's task list in `agent/AGENT.md` so the agent knows it exists and when to use it.

---

## Next Steps

- **[Evaluation System Explained](evaluation-system-explained.md)** — Deep dive into the scoring system
- **[APIs](apis.md)** — Learn about the external APIs each skill calls
- **[Data Schemas](data-schemas.md)** — See the data structures each skill produces

---

<div align="center">

[← Back to Documentation Index](README.md)

</div>
