# 🔬 Project Overview — What is PaperClaw?

## The One-Sentence Summary

**PaperClaw is an AI-powered research assistant that automatically finds, reads, scores, and reports on scientific papers — so researchers don't have to manually trawl through hundreds of papers every week.**

---

## The Problem It Solves

Imagine you're a researcher working on **Scientific Machine Learning (SciML)** — specifically, using neural networks to solve physics equations and work with 3D geometries. Every day, dozens of new papers appear on [arXiv](https://arxiv.org/) (the world's largest open-access preprint server). You need to:

1. **Search** for papers relevant to your specific niche
2. **Read** each paper (or at least the abstract)
3. **Evaluate** whether it's worth a deep dive
4. **Summarize** the key findings
5. **Report** the best ones to your team

Doing this manually takes hours. PaperClaw automates the **entire pipeline**, from search to weekly team report.

---

## What Does It Actually Do?

PaperClaw runs as an **AI agent** on the [OpenClaw](https://github.com/openclaw/openclaw) platform. Here's what happens every day and every week:

### 🕐 Every Day at 8:00 PM (Singapore Time)

1. **Searches arXiv** using 9 carefully crafted keyword combinations (e.g., "geometry + neural operator + PDE")
2. **Finds ~270 papers** across all queries
3. **Deduplicates** — removes papers it has already seen before
4. **Scores relevance** — ranks papers by how closely they match the research domain
5. **Picks the Top 3** most relevant new papers
6. **Downloads their PDFs**
7. **Sends a summary message** to the team chat (如流, an internal messaging service)
8. **Queues them for evaluation** — the agent then reads each paper and generates a detailed review

### 📝 For Each Paper (After Daily Search)

1. **Fetches citation data** from Semantic Scholar (how many times has this paper been cited?)
2. **Generates a summary** by answering 10 critical research questions
3. **Scores the paper** on 4 dimensions (engineering value, architecture innovation, theoretical contribution, result reliability)
4. **Calculates an impact score** that fairly accounts for the paper's age and citation count
5. **Saves everything** to structured files (JSON metadata, Markdown summaries and score reports)
6. **Updates a central registry** to track all evaluated papers

### 📊 Every Sunday at 10:00 AM (Singapore Time)

1. **Collects all papers** evaluated in the past 7 days
2. **Ranks them** by final score
3. **Picks the Top 3** for the weekly report
4. **Generates a Markdown report** with summaries and scores
5. **Uploads to a knowledge base** (Baidu Knowledge Base) for permanent storage
6. **Sends the report** to the team chat

---

## The Research Domain

PaperClaw is specifically focused on these areas of Scientific ML:

| Area | What It Means |
|------|---------------|
| **Neural Operators** | Neural networks that learn to map between function spaces (e.g., DeepONet, FNO) |
| **PDE Solvers** | Using deep learning to solve Partial Differential Equations |
| **3D Geometry-Aware ML** | Machine learning that understands 3D shapes and meshes |
| **Physics-Informed Neural Networks (PINNs)** | Neural nets that know about physics laws |
| **CFD Surrogates** | Fast approximations of Computational Fluid Dynamics simulations |

> **Why this specific domain?** The project was built for a research team working on 3D surrogate modeling — using AI to replace expensive physics simulations with fast neural network predictions. This is cutting-edge work at the intersection of AI, physics, and engineering.

---

## Key Concepts You Should Know

### What is an "Agent"?

In the context of PaperClaw, an **agent** is an AI system (powered by a large language model) that has:
- A **defined role** (research scientist expert)
- **Skills** it can use (search papers, evaluate papers, generate reports)
- **Schedules** that trigger it automatically (daily search, weekly report)
- **Access to tools** (Python scripts, APIs, file system)

Think of it as a very specialized AI assistant that runs on autopilot.

### What is OpenClaw?

**OpenClaw** is the platform that hosts and runs the agent. It provides:
- Agent lifecycle management
- Skill execution
- Scheduling (cron jobs)
- LLM (Large Language Model) integration
- Workspace file management

PaperClaw is an agent that runs *inside* OpenClaw.

### What is a "Skill"?

A **skill** is a modular capability that the agent can use. Each skill:
- Lives in its own folder under `skills/`
- Has a `SKILL.md` file that describes what it does
- Has Python scripts that implement the actual functionality
- Can be called independently or as part of a larger workflow

PaperClaw has 5 skills: arXiv Search, Paper Review, Daily Search, Semantic Scholar, and Weekly Report.

---

## How It All Fits Together (The Big Picture)

```
┌──────────────────────────────────────────────────────────────┐
│                     OpenClaw Platform                         │
│                                                              │
│   ┌──────────────────────────────────────────────────────┐   │
│   │             PaperClaw Agent                          │   │
│   │        (Role: SciML Research Scientist)              │   │
│   │                                                      │   │
│   │   ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │   │
│   │   │  Daily   │  │  arXiv   │  │  Paper Review    │  │   │
│   │   │  Search  │  │  Search  │  │  (Summarize +    │  │   │
│   │   │  Skill   │──│  Skill   │──│   Score + Save)  │  │   │
│   │   └──────────┘  └──────────┘  └──────────────────┘  │   │
│   │                                                      │   │
│   │   ┌──────────┐  ┌──────────────────┐                │   │
│   │   │ Semantic │  │  Weekly Report   │                │   │
│   │   │ Scholar  │  │  (Top 3 + KB +   │                │   │
│   │   │   Skill  │  │   Messaging)     │                │   │
│   │   └──────────┘  └──────────────────┘                │   │
│   └──────────────────────────────────────────────────────┘   │
│                                                              │
└────────────────┬─────────────────────────┬───────────────────┘
                 │                         │
        External APIs              Local File Storage
    ┌────────────┴──────┐     ┌────────────┴──────────┐
    │ • arXiv API       │     │ papers/               │
    │ • Semantic Scholar│     │   ├── DeepONet/       │
    │ • Baidu KB API    │     │   │   ├── *.pdf      │
    │ • 如流 Messaging  │     │   │   ├── summary.md │
    └───────────────────┘     │   │   ├── scores.md  │
                              │   │   └── metadata   │
                              │   └── evaluated_     │
                              │       papers.json    │
                              │ weekly_reports/      │
                              │ search_logs/         │
                              └──────────────────────┘
```

---

## What Makes PaperClaw Interesting?

Here are some things that stand out about this project:

1. **Full Automation** — It's not just a search tool. It's a complete pipeline from discovery to team reporting, running on autopilot.

2. **Multi-Dimensional Evaluation** — Papers aren't just scored on a single number. The 4-dimensional system captures different aspects of paper quality (practical value, innovation, theory, reliability).

3. **Fair Age-Citation Weighting** — A paper published yesterday can't have many citations yet. The Date-Citation system accounts for this, giving new papers a fair chance.

4. **Triple-Layer Deduplication** — Three separate dedup checks at different stages prevent the same paper from being processed twice, even under concurrent access.

5. **Thread-Safe File Operations** — The registry uses file locking to prevent data corruption when multiple processes write simultaneously.

6. **Modular Skill Architecture** — Each skill is independent and can be tested, updated, or replaced without affecting others.

---

## Quick Numbers

| Metric | Value |
|--------|-------|
| Papers searched per day | ~270 (9 queries × 30 papers) |
| Papers selected per day | Top 3 |
| Keywords monitored | 9 query combinations |
| Scoring dimensions | 4 + impact adjustment |
| Skills | 5 modular skills |
| Scheduled tasks | 2 (daily + weekly) |
| External APIs | 4 (arXiv, Semantic Scholar, Baidu KB, 如流) |

---

## Next Steps

Now that you know what PaperClaw is, dive deeper:

- **[System Architecture](system-architecture.md)** — How the pieces fit together
- **[Agent Architecture](agent-architecture.md)** — How the AI agent thinks
- **[Data Flow](data-flow.md)** — Follow data through real workflows

---

<div align="center">

[← Back to Documentation Index](README.md)

</div>
