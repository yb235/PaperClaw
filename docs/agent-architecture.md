# 🤖 Agent Architecture

This document explains how the AI agent at the heart of PaperClaw is defined, configured, and orchestrated. If you've ever wondered "how does an AI agent actually work in practice?", this is for you.

---

## Table of Contents

- [What is an Agent?](#what-is-an-agent)
- [The Agent Definition File](#the-agent-definition-file)
- [The Agent's Four Tasks](#the-agents-four-tasks)
- [How the Agent Uses Skills](#how-the-agent-uses-skills)
- [The Search Keywords](#the-search-keywords)
- [The 10-Question Summary Framework](#the-10-question-summary-framework)
- [The LLM Configuration](#the-llm-configuration)
- [The Scheduling System](#the-scheduling-system)
- [Quick Actions](#quick-actions)
- [Design Rationale](#design-rationale)

---

## What is an Agent?

In PaperClaw, the **agent** is not just a chatbot. It's an AI system with:

| Component | What It Is | Analogy |
|-----------|-----------|---------|
| **Role** | A defined persona and expertise level | A job description |
| **Tasks** | Specific workflows it knows how to execute | Standard operating procedures |
| **Skills** | Tools and scripts it can use | A toolbox |
| **Schedules** | Automated triggers | A calendar |
| **Model** | The LLM that powers its reasoning | Its brain |

Think of it as hiring a virtual research scientist: you write a job description (`AGENT.md`), give them tools (`skills/`), tell them when to work (`schedules.json`), and connect them to a brain (`models.json`).

---

## The Agent Definition File

**File:** `agent/AGENT.md`

This is the single most important file in the project. It's written in Markdown and read by the OpenClaw platform to configure the agent's behavior.

### What's in it?

The file defines the agent's complete identity:

```
┌─────────────────────────────────────────────────┐
│              AGENT.md Contents                  │
│                                                 │
│  1. Role Definition                             │
│     "You are a Surrogate-Modeling Expert..."    │
│                                                 │
│  2. Expertise Areas                             │
│     Operator Learning, PDEs, Geometric DL...    │
│                                                 │
│  3. Task 1: Paper Search & Summarization        │
│     Search arXiv → Download → Summarize         │
│                                                 │
│  4. Task 2: Paper Evaluation                    │
│     Score on 4 dimensions + impact              │
│                                                 │
│  5. Task 3: Daily Automated Search              │
│     Batch search → Dedup → Select → Notify      │
│                                                 │
│  6. Task 4: Weekly Report Generation            │
│     Top 3 → KB upload → Team notification       │
│                                                 │
│  7. Search Keywords (9 query patterns)          │
│                                                 │
│  8. Evaluation Criteria & Templates             │
│                                                 │
│  9. Output Format Specifications                │
└─────────────────────────────────────────────────┘
```

### The agent's role

The agent is configured as a **world-class researcher** with deep expertise in:

| Expertise Area | What It Covers |
|----------------|---------------|
| **Operator Learning Theory** | Neural operators that learn mappings between function spaces |
| **Functional Analysis & PDEs** | Mathematical foundations of partial differential equations |
| **Geometric Deep Learning** | ML on non-Euclidean domains (meshes, graphs, manifolds) |
| **Physics-Informed ML** | Incorporating physics constraints into neural networks |
| **Distributed HPC Training** | Large-scale parallel computing for training big models |

> **Rationale:** By defining the agent as an expert, the LLM is primed to use domain-specific knowledge in its summaries and evaluations. This isn't just a generic "summarize this paper" bot — it understands the field and can assess novelty and significance within the context of SciML research.

---

## The Agent's Four Tasks

The agent has four clearly defined tasks. Each follows a specific workflow.

### Task 1: Paper Search & Summarization

**When:** Triggered by a user request (e.g., "find papers about neural operators on meshes")

```
User Input (keyword or paper ID)
    │
    ▼
Search arXiv using the keyword
    │
    ▼
Download the PDF
    │
    ▼
Read the paper (the LLM reads the full PDF)
    │
    ▼
Generate summary answering 10 research questions
    │
    ▼
Save summary.md to papers/{paper_title}/
```

### Task 2: Paper Evaluation

**When:** After Task 1, or triggered by a user request to evaluate a specific paper

```
Paper PDF + summary.md
    │
    ▼
Fetch citation data from Semantic Scholar
    │
    ▼
Score on 4 dimensions (1-10 each):
  • Engineering Application Value
  • Network Architecture Innovation
  • Theoretical Contribution
  • Result Reliability
    │
    ▼
Calculate Date-Citation impact score
    │
    ▼
Compute final score:
  Final = (4-dim average × 0.9) + (impact × 0.1)
    │
    ▼
Save scores.md + metadata.json
    │
    ▼
Update evaluated_papers.json registry
```

### Task 3: Daily Automated Search

**When:** Every day at 20:00 (Asia/Singapore), triggered by cron

```
Cron trigger at 20:00
    │
    ▼
Run 9 keyword queries on arXiv (30 papers each → ~270 total)
    │
    ▼
Deduplicate within search results (ID + title matching)
    │
    ▼
Cross-check against evaluated_papers.json (skip already-reviewed papers)
    │
    ▼
Score remaining papers by relevance
    │
    ▼
Select Top 3 most relevant new papers
    │
    ▼
Download PDFs + create metadata.json for each
    │
    ▼
Save search log (audit trail)
    │
    ▼
Send summary message to 如流 team chat
    │
    ▼
Generate pending_evaluation task list
    │
    ▼
Agent automatically runs Task 2 for each paper
```

### Task 4: Weekly Report Generation

**When:** Every Sunday at 10:00 (Asia/Singapore), triggered by cron

```
Cron trigger on Sunday 10:00
    │
    ▼
Load all papers from evaluated_papers.json
    │
    ▼
Filter to papers evaluated in the past 7 days
    │
    ▼
Sort by final_score (highest first)
    │
    ▼
Select Top 3 papers
    │
    ▼
For each Top 3 paper:
  - Read its summary.md and scores.md
  - Upload to Baidu Knowledge Base
    │
    ▼
Generate Markdown weekly report
    │
    ▼
Upload report to Knowledge Base
    │
    ▼
Send report to 如流 with links
```

---

## How the Agent Uses Skills

The agent doesn't execute Python code directly. Instead, it uses **skills** — modular tools that encapsulate specific capabilities.

Here's which skill powers which task:

```
Task 1 (Search & Summarize)  →  arxiv-search skill + LLM reasoning
Task 2 (Evaluate)            →  semantic-scholar skill + paper-review skill
Task 3 (Daily Search)        →  daily-search skill (which uses arxiv-search internally)
Task 4 (Weekly Report)       →  weekly-report skill
```

### How skill invocation works

1. The OpenClaw platform reads `AGENT.md` and understands the agent's capabilities
2. When a task is triggered (by cron or user), the platform sends a message to the LLM
3. The LLM decides which skill to use and calls the appropriate Python script
4. The script executes, writes results to disk
5. The LLM reads the results and continues the workflow

> **Rationale:** This "LLM as orchestrator" pattern means the agent can adapt its workflow based on context. If a search returns no new papers, it can skip evaluation. If a paper is already in the registry, it can skip it. The LLM makes these decisions dynamically.

---

## The Search Keywords

The agent monitors arXiv using 9 carefully designed query patterns:

| # | Query Pattern | What It Catches |
|---|--------------|-----------------|
| 1 | `ti:geometry AND (ti:neural OR ti:operator OR ti:pde)` | Geometry-aware neural operators |
| 2 | `ti:mesh AND (ti:neural OR ti:deep OR ti:learning)` | Mesh-based deep learning |
| 3 | `(ti:cfd OR ti:fluid) AND (ti:surrogate OR ti:neural OR ti:deep)` | CFD surrogates |
| 4 | `ti:3d AND (ti:pde OR ti:physics OR ti:solver)` | 3D physics solvers |
| 5 | `(ti:fno OR ti:deeponet OR ti:"neural operator") AND (ti:geometry OR ti:mesh OR ti:domain)` | Famous neural operators + geometry |
| 6 | `(ti:pressure OR ti:stress OR ti:flow) AND (ti:neural OR ti:deep OR ti:surrogate)` | Engineering quantities + ML |
| 7 | `(ti:aerodynamic OR ti:structural) AND (ti:surrogate OR ti:neural)` | Aero/structural surrogates |
| 8 | `ti:"graph neural" AND (ti:pde OR ti:physics OR ti:simulation)` | Graph neural networks for physics |
| 9 | `ti:transformer AND (ti:pde OR ti:physics OR ti:fluid)` | Transformers for physics |

### Exclusion keywords

The agent also knows to **exclude** papers from irrelevant fields:

- Epidemiology, disease modeling, population dynamics
- Finance, economics, stock trading
- NLP, language models, text mining
- Drug discovery, protein folding, chemistry

> **Rationale:** The search terms use the `ti:` prefix (title search) rather than full-text search, which dramatically reduces false positives. A paper about "neural operators for fluid dynamics" will match, but a paper that merely *mentions* neural operators in a biology context won't. The exclusion list prevents the most common false positives from adjacent fields.

---

## The 10-Question Summary Framework

When the agent summarizes a paper, it answers these 10 questions:

| # | Question | Why It Matters |
|---|----------|---------------|
| 1 | What problem does the paper solve? | Frames the contribution |
| 2 | Is this a new problem? | Assesses novelty |
| 3 | What is the scientific hypothesis? | Identifies the core claim |
| 4 | What related work exists? Who are the key researchers? | Places the paper in context |
| 5 | What is the solution approach? | Explains the method |
| 6 | How are the experiments designed? | Assesses rigor |
| 7 | What datasets are used? Is it open source? | Evaluates reproducibility |
| 8 | Do the results support the hypothesis? | Checks the evidence |
| 9 | What are the main contributions? | Summarizes the value |
| 10 | What future work is suggested? | Shows where the field is heading |

> **Rationale:** These 10 questions are a well-known framework in research paper reading (originally from a Shen Xiangyang research methodology guide). They ensure every summary covers the same dimensions, making papers comparable. It also guides the LLM to produce structured, useful output rather than rambling summaries.

---

## The LLM Configuration

**File:** `agent/models.json`

The agent uses **GLM-4.7** (an internal large language model) with these settings:

| Setting | Value | What It Means |
|---------|-------|--------------|
| Model | `glm-4.7-internal` | The specific LLM version |
| Context Window | 200,000 tokens | Can fit an entire research paper |
| Max Output Tokens | 8,192 | Long enough for detailed summaries |
| API Format | OpenAI-compatible | Uses the standard chat completions API |

> **Rationale:** The 200K context window is critical — research papers can be 10-30 pages long, and the LLM needs to read the entire paper to generate accurate summaries. The OpenAI-compatible API format means the model could be swapped for GPT-4, Claude, or any other compatible model with minimal configuration changes.

### Enforced Chain-of-Thought

The agent is configured to use `<think>` tags during scoring:

```markdown
<think>
This paper proposes a novel mesh-aware neural operator...
The engineering value is high because it was validated on industrial CFD cases...
I'll rate engineering application at 8/10 because...
</think>

**Engineering Application Value: 8/10**
```

> **Rationale:** Forcing the LLM to "think out loud" before scoring leads to more consistent and justified evaluations. Without this, the model might assign arbitrary scores. The thinking process also creates an audit trail of the reasoning.

---

## The Scheduling System

**File:** `agent/schedules.json`

Two scheduled tasks run automatically:

### Daily Paper Search

```json
{
  "name": "Daily Paper Search",
  "schedule": {
    "kind": "cron",
    "expr": "0 20 * * *",
    "tz": "Asia/Singapore"
  },
  "payload": {
    "kind": "agentTurn",
    "message": "执行每日论文检索任务..."
  },
  "sessionTarget": "isolated"
}
```

### Weekly Report Generation

```json
{
  "name": "Weekly Report Generation",
  "schedule": {
    "kind": "cron",
    "expr": "0 10 * * 0",
    "tz": "Asia/Singapore"
  },
  "payload": {
    "kind": "agentTurn",
    "message": "生成三维几何代理模型研究周报..."
  },
  "sessionTarget": "isolated"
}
```

### Key design details

| Setting | Value | Explanation |
|---------|-------|-------------|
| `kind: "cron"` | Standard cron format | Familiar and widely understood scheduling syntax |
| `tz: "Asia/Singapore"` | Singapore timezone (UTC+8) | Matches the research team's working hours |
| `kind: "agentTurn"` | Message payload | The schedule sends a text message to the agent, which triggers the workflow |
| `sessionTarget: "isolated"` | Separate session | Each scheduled run gets its own session, preventing state leakage between runs |

> **Rationale:** Using `sessionTarget: "isolated"` is crucial. Without it, a daily search running at 8 PM could interfere with a manual user conversation happening at the same time. Isolated sessions ensure each automated run is self-contained.

---

## Quick Actions

**File:** `agent/quick_actions.sh`

This is a convenience script for manual interaction. It provides a menu of common actions:

```bash
# Example quick actions:
# 1. Run daily paper search
# 2. Evaluate a specific paper
# 3. Generate weekly report
# 4. Search for papers on a specific topic
```

> **Rationale:** While the agent runs automatically on schedule, researchers sometimes need to trigger specific actions manually (e.g., "search for papers about Transformers for PDE solving"). The quick actions menu provides common shortcuts.

---

## Design Rationale

### Why Markdown for agent definition?

The agent is defined in `AGENT.md` — a Markdown file, not code. This is a deliberate choice:

1. **Readable** — Anyone can read and understand the agent's role without knowing how to code
2. **Editable** — Changing the agent's behavior (e.g., adding a new task or adjusting keywords) is just text editing
3. **Versionable** — Changes to the agent definition show up clearly in git diffs
4. **LLM-native** — The LLM processes Markdown naturally, so the definition format matches the processing format

### Why separate scheduling from task logic?

The cron schedule (`schedules.json`) only sends a message to the agent. The agent then decides what to do based on that message. This separation means:

1. **Schedule changes** don't require code changes
2. **The same task** can be triggered manually (by user message) or automatically (by cron)
3. **Testing is easy** — just send the same message the cron would send

### Why define expertise so specifically?

The agent isn't just told "you're a researcher." It's told specific subfields, specific methods, and specific evaluation criteria. This:

1. **Grounds the LLM** — prevents generic, shallow responses
2. **Enables domain expertise** — the LLM can make nuanced judgments about paper quality
3. **Sets expectations** — users know exactly what the agent is good at and what it's not

---

## Next Steps

- **[Skills Deep Dive](skills-deep-dive.md)** — Explore each skill module in detail
- **[Evaluation System Explained](evaluation-system-explained.md)** — Deep dive into the scoring system
- **[Data Flow](data-flow.md)** — Follow data through real workflows

---

<div align="center">

[← Back to Documentation Index](README.md)

</div>
