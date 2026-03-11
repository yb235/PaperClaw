# 📊 Evaluation System Explained

This document breaks down PaperClaw's scoring system into plain English. If you've ever wondered "how does PaperClaw decide if a paper is good?", this is the complete answer.

---

## Table of Contents

- [The Big Picture](#the-big-picture)
- [Step 1: The Four Dimensions](#step-1-the-four-dimensions)
- [Step 2: The Date-Citation Impact Score](#step-2-the-date-citation-impact-score)
- [Step 3: The Final Score](#step-3-the-final-score)
- [Worked Examples](#worked-examples)
- [Why This System?](#why-this-system)

---

## The Big Picture

PaperClaw scores every paper in three steps:

```
Step 1:  Score the paper on 4 quality dimensions (1-10 each)
            │
            ▼
Step 2:  Calculate a Date-Citation impact score (1-10)
            │
            ▼
Step 3:  Combine them into one final score
         Final = (4-dim average × 0.9) + (impact × 0.1)
```

**Why two separate components?**

- The **4-dimension score** measures the paper's **quality** (how good is the research itself?)
- The **impact score** measures the paper's **influence** (how much has it affected the field?)

They're kept separate because a brand-new paper might be brilliant but have zero citations yet. The 90/10 weighting ensures quality matters most, while still rewarding recognized impact.

---

## Step 1: The Four Dimensions

Each paper is scored on four independent dimensions, each rated 1-10:

```
┌─────────────────────────────────────────────────┐
│              FOUR DIMENSIONS                    │
│                                                 │
│  ┌──────────────┐    ┌──────────────────────┐  │
│  │ Engineering  │    │ Architecture         │  │
│  │ Application  │    │ Innovation           │  │
│  │ Value        │    │                      │  │
│  │   (1-10)     │    │   (1-10)             │  │
│  └──────────────┘    └──────────────────────┘  │
│                                                 │
│  ┌──────────────┐    ┌──────────────────────┐  │
│  │ Theoretical  │    │ Result               │  │
│  │ Contribution │    │ Reliability          │  │
│  │              │    │                      │  │
│  │   (1-10)     │    │   (1-10)             │  │
│  └──────────────┘    └──────────────────────┘  │
│                                                 │
│  Four-Dim Average = sum of all four / 4         │
└─────────────────────────────────────────────────┘
```

### Dimension 1: Engineering Application Value

**Question:** "Can someone actually use this in the real world?"

| Score | In Plain English |
|-------|-----------------|
| **9-10** | "This solves a real industrial problem. It's been tested at scale. You could deploy it tomorrow." |
| **7-8** | "This is practical and well-validated. There's a clear path to real-world use." |
| **5-6** | "Has some practical potential, but it's only been tested in narrow cases." |
| **3-4** | "Mostly a proof of concept. Works in the lab but not ready for the real world." |
| **1-2** | "Pure theory. No practical application whatsoever." |

**What the agent looks for:**
- Has it been tested on real engineering problems (not just toy examples)?
- Could an engineer deploy this in a production system?
- Is it faster/cheaper/better than existing methods?
- Are there actual industry use cases?

**Example — DeepONet (Score: 9/10):**
> DeepONet provides a universal architecture for learning arbitrary operators. It's been validated on real-world problems in fluid dynamics and heat transfer, and can be trained and deployed efficiently. High engineering value.

> **Rationale for this dimension:** In SciML research, there's a huge gap between "interesting theory" and "useful tool." This dimension separates papers that advance practical engineering from those that advance pure knowledge.

---

### Dimension 2: Architecture Innovation

**Question:** "Is the method design genuinely new?"

| Score | In Plain English |
|-------|-----------------|
| **9-10** | "This is a completely new way of thinking. It could start a whole new research direction." |
| **7-8** | "Significant novelty. Introduces important new components or mechanisms." |
| **5-6** | "Some new ideas, building on existing work in a meaningful way." |
| **3-4** | "Mostly combining existing ideas. Minor tweaks to known methods." |
| **1-2** | "No innovation. Just applying an off-the-shelf architecture." |

**What the agent looks for:**
- Is the architecture fundamentally new, or an incremental improvement?
- Does it introduce novel components (new attention mechanisms, new layers, new training procedures)?
- Is there theoretical justification for the design choices?
- How does it compare to state-of-the-art?

**Examples:**
- DeepONet: Branch + Trunk network design = 10/10 (completely new paradigm)
- FNO: Fourier transform in neural operator layers = 9/10 (novel spectral approach)
- CViT: Continuous vision transformer = 8/10 (bridges discrete and continuous domains)

> **Rationale for this dimension:** Innovation is what drives the field forward. A paper that introduces a genuinely new architecture has lasting impact, even if its immediate results aren't the best.

---

### Dimension 3: Theoretical Contribution

**Question:** "Does this paper advance our mathematical understanding?"

| Score | In Plain English |
|-------|-----------------|
| **9-10** | "New mathematical framework. Proves important theorems. Opens new theoretical territory." |
| **7-8** | "Deepens existing theory. New theoretical insights. Improves known bounds." |
| **5-6** | "Some theoretical analysis, but not very deep." |
| **3-4** | "Mostly applies existing theory without new insights." |
| **1-2** | "No theory. Pure experiments." |

**What the agent looks for:**
- Does it introduce new mathematical formalism?
- Are there new theorems with proofs?
- Does it connect previously unrelated theoretical frameworks?
- Does it provide deeper understanding of why existing methods work?
- Does it solve open theoretical problems?

**Examples:**
- CViT: Establishes a formal bridge between discrete Transformers and continuous operators = 9/10
- DeepONet: Based on the Universal Approximation Theorem for operators = 9/10
- FNO: Proves universal approximation in Fourier space = 8/10

> **Rationale for this dimension:** Theory is the foundation of reliable science. Papers with strong theory are more likely to generalize to new problems, while purely empirical papers might only work in specific conditions.

---

### Dimension 4: Result Reliability

**Question:** "Can I trust these results?"

| Score | In Plain English |
|-------|-----------------|
| **9-10** | "Rigorous experiments, open-source code AND data, fully reproducible." |
| **7-8** | "Good experiments, partially open resources, results are credible." |
| **5-6** | "Basic experiments, no open resources, needs independent verification." |
| **3-4** | "Experiment design has flaws. Can't verify the claims." |
| **1-2** | "Serious problems. Results seem unreliable." |

**What the agent looks for:**
- Is the experimental design rigorous (proper baselines, ablation studies, multiple datasets)?
- Is the code available on GitHub?
- Are the datasets publicly accessible?
- Are there enough details to reproduce the results?
- Do the results actually support the paper's claims?

> **Rationale for this dimension:** The replication crisis is real. A paper with brilliant ideas but no reproducibility is worth less than a modest paper with fully open code and data that anyone can verify.

---

## Step 2: The Date-Citation Impact Score

### The Problem

How do you fairly compare a paper published yesterday (0 citations) with a paper published 5 years ago (3,000 citations)?

Raw citation count is unfair to new papers. Ignoring citations is unfair to established work. PaperClaw solves this with a **Date-Citation adjustment system**.

### How It Works

```
Step 2a:  Calculate the paper's age (in months)
Step 2b:  Look up the adjustment factor based on age + citations
Step 2c:  Check for citation density bonus
Step 2d:  Apply adjustment to base impact score (capped at 10.0)
```

### Step 2a: Calculate Paper Age

```
age_months = (today - publication_date) / 30.44 days
```

Papers fall into three age categories:

| Category | Age | Label |
|----------|-----|-------|
| **New** | ≤ 3 months | Just published, citations haven't had time to accumulate |
| **Medium** | 3-24 months | Had time to gather some citations |
| **Established** | > 24 months | Mature paper, citation count is meaningful |

### Step 2b: Look Up Adjustment Factor

#### New papers (≤ 3 months)

```
Adjustment = +0.2 (flat bonus for all new papers)
```

> **Why?** A 1-month-old paper literally cannot have many citations. Giving all new papers a flat +0.2 bonus acknowledges that they deserve a chance.

#### Medium-age papers (3-24 months)

| Citations | Adjustment | Interpretation |
|-----------|------------|---------------|
| ≥ 50 | +0.5 | Impressive for a young paper — getting lots of attention |
| 20-49 | +0.3 | Good traction for this age |
| 10-19 | +0.2 | Some recognition |
| < 10 | +0.1 | Limited impact so far |

#### Established papers (> 24 months)

| Citations | Adjustment | Interpretation |
|-----------|------------|---------------|
| ≥ 200 | +0.5 | A truly influential paper |
| 100-199 | +0.4 | Well-recognized work |
| 50-99 | +0.3 | Solid citation record |
| 20-49 | +0.2 | Moderate impact |
| < 20 | +0.0 | Hasn't resonated with the field |

> **Notice:** An established paper needs 200+ citations to get the same +0.5 that a medium paper gets with just 50. This is intentional — a 5-year-old paper has had much more time to accumulate citations, so the bar is higher.

### Step 2c: Citation Density Bonus

This rewards papers that are being cited *rapidly*, regardless of age:

```
citation_density = citations / age_months
```

| Density | Bonus | Interpretation |
|---------|-------|---------------|
| ≥ 10 citations/month | +0.2 | This paper is on fire — very high interest |
| 5-10 citations/month | +0.1 | Strong sustained interest |
| < 5 citations/month | +0.0 | Normal citation rate |

> **Why density in addition to raw count?** Because a paper with 100 citations in 3 months (33/month) is *much* more impressive than a paper with 100 citations in 10 years (0.8/month). Density captures momentum.

### Step 2d: Apply Adjustment

```
total_adjustment = age_adjustment + density_bonus
total_adjustment = min(total_adjustment, 1.0)    # Cap at 1.0

impact_score = base_impact_score + total_adjustment
impact_score = min(impact_score, 10.0)            # Cap at 10.0
```

---

## Step 3: The Final Score

### The Formula

```
four_dim_average = (engineering + architecture + theory + reliability) / 4

final_score = four_dim_average × 0.9 + impact_score × 0.1
```

### Why 90/10 Weighting?

| Weight | Component | Reasoning |
|--------|-----------|-----------|
| **0.9** (90%) | Four-dimension quality | The paper's actual quality is what matters most |
| **0.1** (10%) | Impact score | Citations provide useful signal but shouldn't dominate |

If impact were weighted higher, old papers with many citations would always beat new papers with brilliant ideas. If impact were weighted lower, there'd be no incentive for the system to recognize established, validated work.

The 90/10 split keeps quality as the primary signal while giving a meaningful nudge to papers that the community has validated.

---

## Worked Examples

### Example 1: DeepONet (Established Classic)

```
PAPER INFO
  Published: October 2019 (77 months ago)
  Citations: 3,144
  Venue: Nature Machine Intelligence

STEP 1: FOUR DIMENSIONS
  Engineering Application:     9/10  (validated on real-world problems)
  Architecture Innovation:    10/10  (completely new branch+trunk paradigm)
  Theoretical Contribution:    9/10  (based on universal approximation theorem)
  Result Reliability:          9/10  (open-source, extensively verified)
  
  Four-Dim Average: (9 + 10 + 9 + 9) / 4 = 9.25

STEP 2: IMPACT SCORE
  Age: 77 months → "Established" category
  Citations: 3,144 → ≥ 200 → adjustment = +0.5
  Density: 3,144 / 77 = 40.8/month → ≥ 10 → bonus = +0.2
  Total adjustment: 0.5 + 0.2 = 0.7
  Impact: 9.0 + 0.7 = 9.7 (but paper this impactful → 10.0)

STEP 3: FINAL SCORE
  Final = 9.25 × 0.9 + 10.0 × 0.1 = 8.325 + 1.0 = 9.33
```

### Example 2: Brand-New Paper (Just Published)

```
PAPER INFO
  Published: 2 months ago
  Citations: 3
  Venue: arXiv preprint

STEP 1: FOUR DIMENSIONS
  Engineering Application:    7/10  (promising but limited validation)
  Architecture Innovation:    8/10  (novel attention mechanism for meshes)
  Theoretical Contribution:   5/10  (some analysis, not very deep)
  Result Reliability:         6/10  (code available but limited experiments)
  
  Four-Dim Average: (7 + 8 + 5 + 6) / 4 = 6.5

STEP 2: IMPACT SCORE
  Age: 2 months → "New" category
  Adjustment: +0.2 (flat bonus)
  Density: 3 / 2 = 1.5/month → < 5 → no bonus
  Total adjustment: 0.2
  Impact: base 6.0 + 0.2 = 6.2

STEP 3: FINAL SCORE
  Final = 6.5 × 0.9 + 6.2 × 0.1 = 5.85 + 0.62 = 6.47
```

### Example 3: Medium-Age High-Performer

```
PAPER INFO
  Published: 12 months ago
  Citations: 85
  Venue: NeurIPS 2025

STEP 1: FOUR DIMENSIONS
  Engineering Application:    8/10  (tested on industrial CFD cases)
  Architecture Innovation:    7/10  (clever use of graph neural networks)
  Theoretical Contribution:   7/10  (new convergence bounds)
  Result Reliability:         8/10  (open code, multiple datasets)
  
  Four-Dim Average: (8 + 7 + 7 + 8) / 4 = 7.5

STEP 2: IMPACT SCORE
  Age: 12 months → "Medium" category
  Citations: 85 → ≥ 50 → adjustment = +0.5
  Density: 85 / 12 = 7.1/month → 5-10 → bonus = +0.1
  Total adjustment: 0.5 + 0.1 = 0.6
  Impact: base 7.5 + 0.6 = 8.1

STEP 3: FINAL SCORE
  Final = 7.5 × 0.9 + 8.1 × 0.1 = 6.75 + 0.81 = 7.56
```

### Example 4: Old Paper with Low Impact

```
PAPER INFO
  Published: 36 months ago
  Citations: 15
  Venue: Minor workshop

STEP 1: FOUR DIMENSIONS
  Engineering Application:    4/10
  Architecture Innovation:    5/10
  Theoretical Contribution:   3/10
  Result Reliability:         4/10
  
  Four-Dim Average: (4 + 5 + 3 + 4) / 4 = 4.0

STEP 2: IMPACT SCORE
  Age: 36 months → "Established" category
  Citations: 15 → < 20 → adjustment = +0.0
  Density: 15 / 36 = 0.4/month → < 5 → no bonus
  Total adjustment: 0.0
  Impact: base 4.0 + 0.0 = 4.0

STEP 3: FINAL SCORE
  Final = 4.0 × 0.9 + 4.0 × 0.1 = 3.60 + 0.40 = 4.00
```

> **Notice:** This old paper with few citations gets no impact bonus. The system correctly identifies it as a paper that neither the quality dimensions nor the community found impressive.

---

## Why This System?

### Why not just use citation count?

Citations alone are biased toward old papers. A groundbreaking paper published last month has zero citations. Citations also don't measure quality — some highly-cited papers are cited because they're widely used, not because they're groundbreaking.

### Why not just use the LLM's judgment?

LLMs are good at evaluating paper quality but have no knowledge of a paper's real-world reception. The Date-Citation component adds objective, measurable data about how the scientific community has responded to the paper.

### Why four dimensions instead of one quality score?

A single "quality" score hides important trade-offs. A paper can be theoretically brilliant but impractical (high theory, low engineering). Or it can be practically useful but unoriginal (high engineering, low innovation). The four dimensions make these trade-offs visible, helping researchers decide which papers to read based on their own priorities.

### Why equal weighting for the four dimensions?

Each dimension captures a fundamentally different aspect of paper quality. Weighting one higher would impose a value judgment (e.g., "engineering matters more than theory") that should be left to the individual researcher. Equal weighting provides a balanced starting point.

### Why enforce chain-of-thought (`<think>` tags)?

Without explicit reasoning, the LLM might assign arbitrary scores. The `<think>` requirement forces the model to articulate *why* it's giving each score, leading to more consistent and defensible evaluations. It also creates an audit trail — you can read the thinking and decide if you agree with the reasoning.

---

## Next Steps

- **[Data Schemas](data-schemas.md)** — See the exact JSON fields where scores are stored
- **[Skills Deep Dive](skills-deep-dive.md)** — Learn about the Paper Review skill that executes this system
- **[Design Rationale](design-rationale.md)** — More on the "why" behind every design decision

---

<div align="center">

[← Back to Documentation Index](README.md)

</div>
