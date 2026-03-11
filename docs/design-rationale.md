# 🧠 Design Rationale

This document explains the **"why"** behind every major design decision in PaperClaw. For each decision, we describe what was chosen, what alternatives existed, and why this approach was selected.

---

## Table of Contents

- [1. Files as Communication (Not Direct Calls)](#1-files-as-communication-not-direct-calls)
- [2. Triple-Layer Deduplication](#2-triple-layer-deduplication)
- [3. Structured JSON Alongside Human-Readable Markdown](#3-structured-json-alongside-human-readable-markdown)
- [4. Deferred Evaluation Pattern](#4-deferred-evaluation-pattern)
- [5. Date-Citation Weighting (Instead of Raw Citations)](#5-date-citation-weighting-instead-of-raw-citations)
- [6. Markdown as Agent Definition Format](#6-markdown-as-agent-definition-format)
- [7. Modular Skill Architecture](#7-modular-skill-architecture)
- [8. Title-Only Search Strategy](#8-title-only-search-strategy)
- [9. Isolated Session Scheduling](#9-isolated-session-scheduling)
- [10. Thread-Safe Registry with File Locking](#10-thread-safe-registry-with-file-locking)
- [11. Flat JSON Registry Instead of a Database](#11-flat-json-registry-instead-of-a-database)
- [12. 10-Question Summary Framework](#12-10-question-summary-framework)
- [13. Enforced Chain-of-Thought Scoring](#13-enforced-chain-of-thought-scoring)
- [14. Tiered Caching for Semantic Scholar](#14-tiered-caching-for-semantic-scholar)
- [15. Negative Keywords with Heavy Penalties](#15-negative-keywords-with-heavy-penalties)
- [Interesting Patterns and Curiosities](#interesting-patterns-and-curiosities)

---

## 1. Files as Communication (Not Direct Calls)

### What was chosen

Skills communicate through **files on disk** (JSON and Markdown), not through direct function calls or in-memory messages.

For example:
- Daily search writes `pending_evaluation_YYYY-MM-DD.json`
- The agent reads this file and triggers paper evaluation for each entry
- Paper evaluation writes `metadata.json` and updates `evaluated_papers.json`
- Weekly report reads `evaluated_papers.json` to build its report

### Alternatives considered

| Alternative | Why Not? |
|-------------|----------|
| Direct function imports | Would couple skills together; changes in one skill could break others |
| Message queues (Redis, RabbitMQ) | Overkill for 3 papers/day; adds infrastructure complexity |
| Database writes/reads | Adds dependency on a database server |
| In-memory state | Lost on restart; can't inspect intermediate results |

### Why this approach

1. **Debuggability** — Every intermediate state is visible as a file you can open and read
2. **Recoverability** — If step 3 fails, steps 1-2's outputs are still on disk. Just re-run step 3.
3. **Independence** — Skills don't import each other. They don't even need to be written in the same language.
4. **Auditability** — There's a complete paper trail of everything the system did. Search logs, task lists, metadata — all on disk.
5. **Simplicity** — No message broker, no database, no complex infrastructure. Just files.

> **The insight:** At PaperClaw's scale (3 papers/day, ~270 searches/day), files are not just "good enough" — they're the optimal choice. The overhead of a database or message queue would add complexity without providing meaningful benefits.

---

## 2. Triple-Layer Deduplication

### What was chosen

Papers are checked for duplicates at **three separate stages**:

| Layer | Where | When | What It Prevents |
|-------|-------|------|-----------------|
| Layer 1 | `search_arxiv.py` | During search | Same paper appearing in multiple query results |
| Layer 2 | `daily_paper_search.py` | After search | Re-evaluating papers that were already reviewed in previous days |
| Layer 3 | `update_registry.py` | During registry write | Two concurrent processes writing the same paper |

### Alternatives considered

| Alternative | Why Not? |
|-------------|----------|
| Single dedup check | Misses edge cases (concurrent writes, search overlap) |
| Database UNIQUE constraint | Requires a database (see point 11 below) |
| External dedup service | Overkill for this scale |

### Why this approach

The triple-layer strategy follows the "defense in depth" principle:

- **Layer 1** is for **performance** — no point downloading and processing a paper we've already seen in this same search batch
- **Layer 2** is for **accuracy** — no point evaluating a paper we reviewed last week
- **Layer 3** is for **safety** — even if layers 1 and 2 both miss (extremely unlikely), the write-time check inside a file lock prevents corruption

Each layer has a different focus, and together they create an extremely robust dedup system.

> **The interesting bit:** Layers 1 and 2 use both arXiv ID matching AND normalized title matching. This catches papers that were resubmitted under a slightly different ID (e.g., `2401.12345v1` vs `2401.12345v2`) as well as papers with identical titles from different sources.

---

## 3. Structured JSON Alongside Human-Readable Markdown

### What was chosen

Every paper has **both** `metadata.json` (machine-readable) **and** `summary.md` + `scores.md` (human-readable).

### The story behind this decision

The original version (v1.0) only had Markdown files. The weekly report script extracted scores using regex:

```python
# The old way (fragile!):
match = re.search(r'最终综合评分.*?(\d+\.?\d*)/10', scores_text)
final_score = float(match.group(1))
```

This broke constantly because:
- The LLM sometimes formatted scores differently ("8.5/10" vs "8.5 分" vs "8.5（满分10分）")
- Markdown headers varied in formatting
- Unicode characters in Chinese text confused the regex

The fix was to add `metadata.json` — a structured file that machines can reliably parse:

```python
# The new way (reliable!):
score = metadata['scores']['final_score']
```

### Why keep both files?

| File | Audience | Purpose |
|------|----------|---------|
| `metadata.json` | Programs | Weekly report script reads scores reliably |
| `summary.md` | Humans | Researchers read the detailed analysis |
| `scores.md` | Humans | Researchers read the scoring rationale |

Removing the Markdown files would make the system harder for humans to inspect. Removing the JSON would make downstream processing fragile again.

> **The lesson:** Never parse human-readable documents programmatically when you can write structured data alongside them. This is the "structured data + human docs" pattern, and it's applicable far beyond PaperClaw.

---

## 4. Deferred Evaluation Pattern

### What was chosen

The daily search **finds** papers and creates a task list, but **doesn't evaluate** them immediately. Instead, it writes `pending_evaluation_YYYY-MM-DD.json`, and the agent picks up evaluation as a separate step.

### Alternatives considered

| Alternative | Why Not? |
|-------------|----------|
| Evaluate immediately during search | Couples search and evaluation; if evaluation fails, the whole pipeline fails |
| Evaluate all papers in batch at end of search | Long-running process; harder to debug which paper caused a failure |

### Why this approach

1. **Separation of concerns** — Search is fast (API calls). Evaluation is slow (LLM reads full papers). They have different failure modes and should be independent.
2. **Human oversight** — A researcher could review the task list before evaluation starts, removing papers they know aren't relevant.
3. **Failure isolation** — If evaluation fails for Paper #2, Papers #1 and #3 are unaffected.
4. **Flexibility** — The task list could theoretically be processed by multiple agents in parallel.
5. **Resumability** — If the agent crashes mid-evaluation, the task list is still on disk. Just restart.

---

## 5. Date-Citation Weighting (Instead of Raw Citations)

### What was chosen

Instead of ranking papers by raw citation count, PaperClaw uses a sophisticated age-aware adjustment system with three tiers (new, medium, established) and citation density bonuses.

### Alternatives considered

| Alternative | Why Not? |
|-------------|----------|
| Raw citation count | Massively biased toward old papers. A 2019 paper always beats a 2024 paper. |
| Ignore citations entirely | Throws away a valuable quality signal. Highly-cited papers are usually good. |
| Citations per year (simple normalization) | Doesn't account for the non-linear nature of citation accumulation (papers get cited exponentially more as they become established) |
| PageRank-style influence | Too complex for the data available; requires full citation graph |

### Why this approach

The three-tier system recognizes that citation dynamics are fundamentally different for papers of different ages:

- **New papers (≤ 3 months):** Citations are meaningless at this age. Everyone gets the same +0.2 bonus — a "benefit of the doubt" for new work.
- **Medium papers (3-24 months):** Citations start to become meaningful. The thresholds (10, 20, 50) are calibrated for typical citation rates in the SciML field.
- **Established papers (> 24 months):** Higher thresholds (20, 50, 100, 200) because these papers have had much more time. The bar is proportionally higher.

The **citation density bonus** adds a second signal: papers being cited rapidly are "hot" regardless of their total count. A paper with 30 citations in 3 months is much more notable than one with 30 citations in 3 years.

---

## 6. Markdown as Agent Definition Format

### What was chosen

The agent's entire personality, expertise, tasks, and workflows are defined in `AGENT.md` — a Markdown file.

### Alternatives considered

| Alternative | Why Not? |
|-------------|----------|
| YAML/JSON config | Harder to write natural language instructions; can't express nuance |
| Python code | Requires code changes for behavior modifications; not accessible to non-coders |
| Database entries | Harder to version, review, and diff |

### Why this approach

1. **Readability** — Anyone can understand the agent's behavior by reading `AGENT.md`. No programming knowledge required.
2. **Editability** — Changing the agent's behavior (add a search keyword, modify scoring criteria) is just editing text.
3. **Version control** — `AGENT.md` lives in Git. Every change is tracked, diffable, and reversible.
4. **LLM-native** — The LLM processes Markdown naturally. The agent definition format matches the processing format perfectly.
5. **Self-documenting** — The configuration file IS the documentation. No separate config-vs-docs sync issue.

> **The insight:** Markdown is the "universal format" that both humans and LLMs handle natively. Using it for agent definition creates a single artifact that serves as config, documentation, and prompt template simultaneously.

---

## 7. Modular Skill Architecture

### What was chosen

Five independent skill modules, each in its own folder with its own documentation and scripts.

### Alternatives considered

| Alternative | Why Not? |
|-------------|----------|
| One monolithic script | Hard to test, hard to modify, hard to understand |
| Microservices | Way too complex for this scale; adds infrastructure overhead |
| Plugin framework (abstract base classes, registries) | Over-engineering for 5 skills |

### Why this approach

The skill architecture hits a sweet spot: structured enough to be maintainable, simple enough to not be burdensome.

- **Adding a skill:** Create a folder, write `SKILL.md` and a script. Done.
- **Modifying a skill:** Change only the files in that skill's folder. Nothing else is affected.
- **Testing a skill:** Run its script directly with test inputs. No need to spin up the whole system.
- **Removing a skill:** Delete the folder and remove the reference in `AGENT.md`.

> **The interesting bit:** Each `SKILL.md` serves double duty — it's documentation for humans AND instructions for the AI agent. The LLM reads `SKILL.md` to understand what the skill does and how to use it. This means there's zero divergence between "what the docs say" and "what the agent does."

---

## 8. Title-Only Search Strategy

### What was chosen

arXiv queries use the `ti:` (title) prefix exclusively, rather than searching full text.

### Alternatives considered

| Alternative | Why Not? |
|-------------|----------|
| Full-text search (`all:`) | Massive false positive rate — "neural network" appears in biology, finance, NLP papers |
| Abstract search (`abs:`) | Better than full-text but still too many false positives |
| Category filtering (`cat:cs.LG`) | Too broad — cs.LG includes ALL machine learning, not just SciML |

### Why this approach

Title-only search is the **precision-maximizing** strategy:

- If "neural operator" is in the title, the paper is almost certainly about neural operators.
- If "neural operator" is only in the abstract, it might just be a comparison or passing reference.
- The 9 carefully crafted queries cover the target domain comprehensively.
- The relevance scoring system (with weighted keywords) provides a second filter.

The trade-off is slightly lower **recall** (might miss papers that don't mention key terms in the title), but this is acceptable because:
1. Most relevant papers DO mention their key contribution in the title
2. Missing a few papers is better than drowning in false positives
3. The system runs daily, so missed papers will likely be caught when they appear in related work of other papers

---

## 9. Isolated Session Scheduling

### What was chosen

Each scheduled task runs in an **isolated session** (`"sessionTarget": "isolated"`).

### Alternatives considered

| Alternative | Why Not? |
|-------------|----------|
| Shared session | A daily search at 8 PM could interfere with a manual user conversation happening at the same time |
| Named sessions | Adds complexity without clear benefit |

### Why this approach

Isolated sessions prevent several problems:
1. **No state leakage** — Monday's search context doesn't influence Tuesday's search
2. **No interference** — Automated tasks don't interrupt manual conversations
3. **Clean error handling** — If a session crashes, it doesn't affect other sessions
4. **Parallel safety** — Even if two tasks trigger simultaneously, they can't corrupt each other's state

---

## 10. Thread-Safe Registry with File Locking

### What was chosen

The `update_registry.py` script uses `fcntl.flock()` (Unix file locking) to safely update `evaluated_papers.json`.

### Alternatives considered

| Alternative | Why Not? |
|-------------|----------|
| No locking (just read-write) | Race condition: two processes read, both modify, second write overwrites first |
| SQLite database | Adds dependency; overkill for append-only operations |
| Separate files per paper | Makes querying harder; dedup requires scanning all files |
| Optimistic locking (version numbers) | More complex; retry logic needed |

### Why this approach

File locking is the simplest concurrent-safe solution:
- **One line of code** to acquire a lock
- **Built into the OS** — no external dependencies
- **Blocks until available** — no polling or retry logic needed
- **Automatic release** on process crash — the OS cleans up

The registry is a shared append-only resource. File locking is the textbook solution for this pattern.

> **The graceful degradation:** On Windows (which doesn't support `fcntl`), the script falls back to no locking. This is acceptable because Windows is not the production environment, and even without locking, the duplicate check inside the write section catches most race conditions.

---

## 11. Flat JSON Registry Instead of a Database

### What was chosen

All paper data is stored in JSON files on disk, not in a database.

### Alternatives considered

| Alternative | Why Not? |
|-------------|----------|
| SQLite | Adds a dependency; harder to inspect and edit manually |
| PostgreSQL/MySQL | Way too much infrastructure for this scale |
| MongoDB | Same as above |
| YAML files | Slower to parse; less universal than JSON |

### Why this approach

The numbers tell the story:
- **3 papers per day** × 365 days = ~1,095 papers per year
- Each registry entry is ~200 bytes
- Total registry size after 1 year: ~220 KB

A 220 KB JSON file loads in milliseconds. There's no query complexity that requires indexes. The data is simple (flat list of papers with scores). A database would add:
- Installation steps
- Connection management
- Migration scripts
- Schema definitions
- Backup procedures

All for a 220 KB file that Python can read in 1 millisecond.

> **When would this NOT work?** If PaperClaw scaled to thousands of papers per day, or needed complex queries (e.g., "find all papers by author X in the last month with score > 7"), a database would become necessary. At the current scale, files are optimal.

---

## 12. 10-Question Summary Framework

### What was chosen

Every paper summary answers the same 10 research questions, providing a standardized structure.

### Why this approach

1. **Comparability** — Every summary covers the same dimensions, so you can compare papers directly
2. **Completeness** — The 10 questions ensure no important aspect is missed
3. **Guidance** — Without structure, LLMs tend to produce rambling, inconsistent summaries
4. **Efficiency** — A researcher can jump to the specific question they care about (e.g., "Is the code open source?" → Question 7)

The questions follow a logical arc: problem → hypothesis → method → experiments → results → impact → future. This mirrors how a researcher naturally reads a paper.

---

## 13. Enforced Chain-of-Thought Scoring

### What was chosen

The agent must use `<think>` tags to reason through each score before stating it.

### Why this approach

Without explicit reasoning:
```
Engineering Application Value: 7/10
```

With chain-of-thought:
```
<think>
This paper proposes a mesh-adaptive neural operator. It's been tested on 3 industrial
CFD cases including turbine blade flows. The speedup over traditional solvers is 100×.
However, it only handles steady-state problems, limiting deployment in many real scenarios.
I'll score engineering value at 7/10 — strong potential but limited scope.
</think>

Engineering Application Value: 7/10
```

Benefits:
1. **Consistency** — The reasoning constrains the score. Without reasoning, an LLM might give 5 or 9 for the same paper on different runs.
2. **Transparency** — Researchers can read the reasoning and decide if they agree
3. **Auditability** — If a score seems wrong, the `<think>` block explains why it was assigned
4. **Self-correction** — The act of explaining a score sometimes leads the LLM to revise it ("wait, actually that should be an 8")

---

## 14. Tiered Caching for Semantic Scholar

### What was chosen

Three different cache durations: 7 days for papers, 30 days for authors, 1 day for citations.

### Why different TTLs?

| Data Type | How Often It Changes | Cache Duration | Rationale |
|-----------|---------------------|----------------|-----------|
| Paper metadata | Almost never (title, authors, venue are permanent) | 7 days | Short enough to catch rare corrections |
| Author data | Very slowly (h-index changes by ~1/year) | 30 days | Monthly updates are plenty |
| Citation data | Frequently (popular papers get cited daily) | 1 day | Need fresh data for accurate impact scoring |

This differentiated caching strategy minimizes API calls (courtesy to a free service) while keeping the most volatile data (citations) fresh.

---

## 15. Negative Keywords with Heavy Penalties

### What was chosen

The relevance scoring system applies **-20 points** for negative keywords in titles (compared to +15 for the strongest positive keywords).

### Why so aggressive?

A paper titled "Neural Network for **Epidemiological** Modeling" contains the word "neural" but is clearly about disease modeling, not physics surrogates. Without strong negative weighting, this paper would score positively on the "neural" keyword and might make it into the Top 3.

The -20 penalty ensures that one negative keyword in the title **completely overwhelms** multiple positive keyword matches. This is correct behavior — a paper in the wrong field is never relevant, no matter how many domain-adjacent words it happens to use.

---

## Interesting Patterns and Curiosities

Here are some patterns in PaperClaw's design that are worth appreciating:

### The "Prompt as Config" Pattern

`AGENT.md` and `SKILL.md` files serve triple duty: they are the **configuration** (tells the system what to do), the **documentation** (tells humans how it works), and the **prompt** (tells the LLM how to behave). There is exactly one source of truth for each behavior.

### The "Enrichment Pipeline" Pattern

A paper's `metadata.json` starts sparse and gets richer over time:
1. **Created during search:** Just basic info (id, title, authors, PDF URL)
2. **After Semantic Scholar:** Adds citation count, venue, publication date
3. **After evaluation:** Adds all four dimension scores, impact score, final score

This progressive enrichment means each stage only needs to add its specific data, without re-creating the entire file.

### The "v2 Lesson"

The `generate_weekly_report_v2.py` filename tells a story. Version 1 used regex to parse scores from Markdown — and it broke. Version 2 reads from `metadata.json`. The "v2" in the filename is a permanent reminder of why structured data matters. This is a real-world case of the principle: "don't parse human-readable documents programmatically."

### The "Polite API Consumer" Pattern

PaperClaw goes out of its way to be a good citizen of public APIs:
- Cached responses with appropriate TTLs
- Rate limiting (3-5 requests/second to arXiv)
- Fallback strategies instead of hammering a failing endpoint
- Using batch endpoints when available (Semantic Scholar)

This is both ethical and practical — getting rate-limited would break the daily pipeline.

### The "Defense in Depth" for Data Integrity

The triple deduplication strategy mirrors security's "defense in depth" principle. Each layer catches a different class of failures:
- Layer 1 (search): Performance optimization
- Layer 2 (cross-database): Correctness guarantee
- Layer 3 (write-time): Concurrency safety

Even if any single layer fails, the system remains correct.

### The "Right-Sized" Technology Choices

PaperClaw consistently chooses the simplest technology that works:
- JSON files instead of a database (for ~1,000 records/year)
- File locking instead of distributed locks (for single-machine deployment)
- Cron instead of a job queue (for 2 scheduled tasks)
- Shell scripts instead of a web UI (for quick actions)

This "right-sizing" means less infrastructure, fewer failure modes, and easier debugging.

---

## Summary Table

| Decision | What | Why |
|----------|------|-----|
| Files as communication | JSON files between skills | Debuggable, recoverable, independent |
| Triple dedup | 3 layers of duplicate checking | Defense in depth for data integrity |
| JSON + Markdown | Both structured and human-readable | Machines AND humans can consume the data |
| Deferred evaluation | Search → task list → evaluate later | Separation of concerns, failure isolation |
| Date-Citation weighting | Age-aware citation adjustment | Fair comparison across paper ages |
| Markdown agent definition | AGENT.md defines the entire agent | Readable, editable, LLM-native |
| Modular skills | 5 independent skill folders | Easy to add, test, and maintain |
| Title-only search | `ti:` prefix in arXiv queries | Maximizes precision, reduces false positives |
| Isolated sessions | Each cron job gets its own session | No state leakage between runs |
| File locking | `fcntl.flock()` for registry writes | Prevents concurrent write corruption |
| Flat JSON registry | No database | Right-sized for ~1,000 records/year |
| 10-question framework | Standardized summary structure | Comparability, completeness, guidance |
| Chain-of-thought scoring | `<think>` tags before scores | Consistency, transparency, auditability |
| Tiered caching | Different TTLs per data type | Fresh where needed, cached where safe |
| Heavy negative keywords | -20 penalty for off-topic signals | Eliminates false positives aggressively |

---

<div align="center">

[← Back to Documentation Index](README.md)

</div>
