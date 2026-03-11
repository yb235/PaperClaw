# 🏗️ System Architecture

This document explains how PaperClaw is organized — what the layers are, what lives where, and how everything connects. Think of it as a map of the codebase.

---

## Table of Contents

- [The Four Layers](#the-four-layers)
- [Directory Structure Explained](#directory-structure-explained)
- [Layer 1: Agent Definition Layer](#layer-1-agent-definition-layer)
- [Layer 2: Skills Layer](#layer-2-skills-layer)
- [Layer 3: Data Storage Layer](#layer-3-data-storage-layer)
- [Layer 4: External Services Layer](#layer-4-external-services-layer)
- [How Layers Talk to Each Other](#how-layers-talk-to-each-other)
- [Design Rationale](#design-rationale)

---

## The Four Layers

PaperClaw uses a **layered architecture**. Each layer has a specific job, and they stack on top of each other:

```
┌─────────────────────────────────────────────────────────┐
│  Layer 1: AGENT DEFINITION                              │
│  "Who is the agent and what can it do?"                 │
│  Files: agent/AGENT.md, models.json, schedules.json     │
├─────────────────────────────────────────────────────────┤
│  Layer 2: SKILLS                                        │
│  "The actual tools the agent uses"                      │
│  Folders: skills/arxiv-search, paper-review, etc.       │
├─────────────────────────────────────────────────────────┤
│  Layer 3: DATA STORAGE                                  │
│  "Where all the data lives"                             │
│  Folders: workspace/papers/, weekly_reports/, etc.       │
├─────────────────────────────────────────────────────────┤
│  Layer 4: EXTERNAL SERVICES                             │
│  "Third-party APIs the system talks to"                 │
│  APIs: arXiv, Semantic Scholar, Baidu KB, 如流           │
└─────────────────────────────────────────────────────────┘
```

> **Rationale:** This layered design means each part can be changed independently. Want to add a new skill? Just add a folder in `skills/`. Want to switch the LLM model? Just edit `models.json`. Want to change how data is stored? Only Layer 3 needs to change.

---

## Directory Structure Explained

Here's every file in the repository and what it does:

```
PaperClaw/
│
├── README.md                  # Project homepage — quick overview and getting started
├── QUICKSTART.md              # 5-minute setup guide
├── INSTALLATION.md            # Detailed installation (3 methods)
├── CONFIGURATION.md           # Configuration reference for all settings
│
├── agent/                     # ─── LAYER 1: Agent Definition ───
│   ├── AGENT.md               # The agent's "brain" — role, expertise, tasks, keywords
│   ├── README.md              # How to use the agent
│   ├── models.json            # Which LLM to use and how to connect to it
│   ├── schedules.json         # Cron jobs — when to run daily search and weekly report
│   └── quick_actions.sh       # Shortcut menu for manual actions
│
├── skills/                    # ─── LAYER 2: Skill Modules ───
│   ├── arxiv-search/          # Skill: Search arXiv for papers
│   │   ├── SKILL.md           # Documentation for this skill
│   │   └── scripts/
│   │       └── search_arxiv.py    # Python script that calls arXiv API
│   │
│   ├── paper-review/          # Skill: Summarize and score papers
│   │   ├── SKILL.md           # Documentation for this skill
│   │   └── scripts/
│   │       └── update_registry.py # Thread-safe registry update tool
│   │
│   ├── daily-search/          # Skill: Orchestrate the daily search pipeline
│   │   ├── SKILL.md           # Documentation for this skill
│   │   └── scripts/
│   │       └── daily_paper_search.py  # Daily search orchestrator
│   │
│   ├── semantic-scholar/      # Skill: Fetch citation data from Semantic Scholar
│   │   ├── SKILL.md           # Documentation for this skill
│   │   └── semantic_scholar_api.py   # API client with caching
│   │
│   └── weekly-report/         # Skill: Generate weekly Top 3 report
│       ├── SKILL.md           # Documentation for this skill
│       └── scripts/
│           └── generate_weekly_report_v2.py  # Report generator + KB integration
│
├── examples/                  # ─── Example data ───
│   └── sample_paper/
│       ├── metadata.json      # What a paper's metadata looks like
│       └── scores.md          # What a paper's score report looks like
│
└── docs/                      # ─── Documentation ───
    ├── README.md              # This documentation index
    ├── architecture.md        # Original architecture doc (Chinese)
    ├── evaluation_system.md   # Original evaluation system doc (Chinese)
    └── ... (other docs)       # Additional documentation files
```

---

## Layer 1: Agent Definition Layer

**Location:** `agent/`

This layer answers: **"Who is the agent, and what can it do?"**

### What's in it?

| File | Purpose | Plain English |
|------|---------|--------------|
| `AGENT.md` | Agent role definition | "You are a world-class SciML researcher. Here are your tasks…" |
| `models.json` | LLM configuration | "Use GLM-4.7 model at this URL with this API key" |
| `schedules.json` | Cron job definitions | "Run daily search at 8 PM, weekly report on Sundays at 10 AM" |
| `README.md` | Usage documentation | "Here's how to use this agent" |
| `quick_actions.sh` | Manual action shortcuts | "Press 1 for daily search, 2 for paper review…" |

### How the agent is defined

The `AGENT.md` file is the most important file in the whole project. It's written in Markdown and read by the OpenClaw platform to configure the agent. It defines:

1. **Role:** "You are a Surrogate-Modeling Expert / SciML Chief Research Scientist"
2. **Expertise areas:** Operator learning, PDEs, geometric deep learning, PINNs, HPC training
3. **Four core tasks:** Paper search, paper evaluation, daily search automation, weekly reports
4. **Search keywords:** 9 specific arXiv query patterns
5. **Evaluation criteria:** 10 research questions + 4-dimensional scoring
6. **Output formats:** Expected file structures and naming conventions

> **Rationale:** By defining the agent's entire personality, expertise, and workflow in a single Markdown file, the system is incredibly easy to modify. Want the agent to also search biomedical papers? Just edit `AGENT.md`. No code changes needed.

### The LLM configuration

`models.json` tells OpenClaw which language model to use:

```json
{
  "providers": {
    "glm": {
      "baseUrl": "http://oneapi-comate.baidu-int.com/v1",
      "api": "openai-completions",
      "models": [
        {
          "id": "glm-4.7-internal",
          "name": "glm Coding",
          "contextWindow": 200000,
          "maxTokens": 8192
        }
      ]
    }
  }
}
```

> **Key detail:** The 200,000 context window is large enough to fit an entire research paper, which is critical for generating accurate summaries.

### The schedule configuration

`schedules.json` defines two cron jobs:

| Schedule | Cron Expression | Timezone | What It Does |
|----------|----------------|----------|-------------|
| Daily Paper Search | `0 20 * * *` | Asia/Singapore | Runs every day at 8:00 PM |
| Weekly Report | `0 10 * * 0` | Asia/Singapore | Runs every Sunday at 10:00 AM |

> **Rationale:** 8 PM was chosen for daily search because arXiv's daily batch of new papers is typically available by then. Sunday morning was chosen for weekly reports because it gives the team a fresh summary at the start of the week.

---

## Layer 2: Skills Layer

**Location:** `skills/`

This layer answers: **"What can the agent actually do?"**

Each skill is a self-contained module with its own documentation and scripts:

```
skills/
├── arxiv-search/        → Searches arXiv for papers
├── paper-review/        → Evaluates and scores papers
├── daily-search/        → Orchestrates the daily pipeline
├── semantic-scholar/    → Fetches citation metadata
└── weekly-report/       → Generates weekly reports
```

### How skills are structured

Every skill follows the same pattern:

```
skill-name/
├── SKILL.md             # Documentation: what this skill does, inputs, outputs
└── scripts/
    └── script.py        # Implementation: the actual Python code
```

> **Rationale:** This uniform structure means you always know where to look. New skills can be added by creating a new folder that follows this pattern. The `SKILL.md` serves double duty: it documents the skill for humans AND instructs the AI agent on how to use it.

### Skill dependency chain

Skills aren't isolated — they call each other:

```
daily-search ──uses──→ arxiv-search    (to search for papers)
daily-search ──creates tasks for──→ paper-review    (to evaluate found papers)
paper-review ──uses──→ semantic-scholar (to get citation data)
weekly-report ──reads from──→ paper-review outputs  (to build the report)
```

> **Rationale:** Instead of building one giant monolithic script, the pipeline is broken into independent skills that communicate through data files. This means each skill can be tested and debugged independently.

For a detailed breakdown of each skill, see **[Skills Deep Dive](skills-deep-dive.md)**.

---

## Layer 3: Data Storage Layer

**Location:** `~/.openclaw/workspace/3d_surrogate_proj/` (at runtime)

This layer answers: **"Where does all the data live?"**

```
workspace/3d_surrogate_proj/
│
├── papers/                          # All paper data
│   ├── DeepONet/                    # One folder per paper
│   │   ├── DeepONet.pdf             # Downloaded PDF
│   │   ├── summary.md              # 10-question summary
│   │   ├── scores.md               # Detailed score report
│   │   └── metadata.json           # Structured metadata + scores
│   ├── FNO/
│   │   └── ...
│   └── evaluated_papers.json       # Master registry of all evaluated papers
│
├── weekly_reports/                  # Generated weekly reports
│   └── 2026-03-02_weekly.md
│
├── search_logs/                     # Audit trail of daily searches
│   └── 2026-03-01_search_log.json
│
├── pending_evaluation_YYYY-MM-DD.json  # Task queue for papers awaiting evaluation
│
└── cache/                           # API response caches
    └── semantic_scholar/            # Cached Semantic Scholar responses
```

### Why three files per paper?

Each paper gets three output files, each serving a different purpose:

| File | Format | Audience | Purpose |
|------|--------|----------|---------|
| `metadata.json` | JSON | Machines | Structured data for programmatic access (used by weekly report script) |
| `summary.md` | Markdown | Humans | Readable research summary answering 10 questions |
| `scores.md` | Markdown | Humans | Detailed scoring rationale with reasoning |

> **Rationale:** Early versions only used Markdown files and parsed scores using regex — which was fragile and broke easily. The `metadata.json` was added so that downstream processes (like the weekly report generator) can reliably read scores without brittle text parsing. This is the "structured data alongside human-readable documents" pattern.

For detailed field-by-field documentation, see **[Data Schemas](data-schemas.md)**.

---

## Layer 4: External Services Layer

This layer answers: **"What external APIs does PaperClaw talk to?"**

| Service | What For | Type |
|---------|----------|------|
| **arXiv API** | Searching and downloading papers | Public REST API |
| **Semantic Scholar API** | Getting citation counts, author info, venue data | Public REST API |
| **Baidu Knowledge Base** | Storing reports and summaries permanently | Internal API |
| **如流 Messaging** | Sending daily/weekly summaries to the team | Internal API |

> **Rationale:** By depending on two public APIs (arXiv + Semantic Scholar) and two internal ones (Baidu KB + 如流), the system balances open-source reproducibility with organizational integration. The public APIs make the core functionality portable; the internal ones provide team-specific value.

For detailed API documentation, see **[APIs](apis.md)**.

---

## How Layers Talk to Each Other

The layers communicate through well-defined boundaries:

```
Layer 1 (Agent Definition)
    │
    │  The agent reads AGENT.md to know its role and tasks.
    │  It reads schedules.json to know when to run.
    │  It reads models.json to know which LLM to use.
    │
    ▼
Layer 2 (Skills)
    │
    │  Skills execute Python scripts that:
    │  - Call Layer 4 APIs (arXiv, Semantic Scholar)
    │  - Write results to Layer 3 storage
    │  - Read from Layer 3 for deduplication
    │
    ▼
Layer 3 (Data Storage)
    │
    │  JSON and Markdown files on disk.
    │  Skills write to it. The weekly report reads from it.
    │  The registry (evaluated_papers.json) is the central index.
    │
    ▼
Layer 4 (External Services)
    │
    │  HTTP APIs that return data.
    │  Semantic Scholar responses are cached locally.
    │  Knowledge Base stores permanent copies of reports.
```

### The key insight: files as communication

Skills don't call each other directly through function calls. Instead, they communicate through **files on disk**:

- `daily-search` writes `pending_evaluation_YYYY-MM-DD.json` → The agent reads this and runs `paper-review` for each paper
- `paper-review` writes `metadata.json` + updates `evaluated_papers.json` → `weekly-report` reads these to build the report

> **Rationale:** This "files as message queues" pattern has several advantages:
> 1. **Debuggability** — You can inspect every intermediate state by just opening a file
> 2. **Recoverability** — If a step fails, the previous step's output is still on disk
> 3. **Independence** — Skills don't need to import each other or share memory
> 4. **Auditability** — There's a complete paper trail of everything the system has done

---

## Technology Stack

| Technology | Role | Version |
|------------|------|---------|
| **Python** | Main implementation language | 3.8+ |
| **OpenClaw** | Agent hosting platform | v1.0+ |
| **GLM-4.7** | Large Language Model (for summaries + scoring) | Internal |
| **requests** | HTTP client for API calls | 2.28+ |
| **python-dateutil** | Date parsing and manipulation | 2.8.2+ |
| **fcntl** | File locking (Unix) | Standard library |
| **JSON** | Data interchange format | — |
| **Markdown** | Human-readable document format | — |

---

## Next Steps

- **[Agent Architecture](agent-architecture.md)** — Dive deeper into how the agent is defined and how it thinks
- **[Data Flow](data-flow.md)** — Follow data through real workflows step by step
- **[Skills Deep Dive](skills-deep-dive.md)** — Explore each skill module in detail

---

<div align="center">

[← Back to Documentation Index](README.md)

</div>
