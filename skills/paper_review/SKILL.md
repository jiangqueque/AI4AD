---
name: paper_review
description: "Use this skill whenever the user shares a paper link, PDF, or title and asks for a summary or review — especially in the autonomous driving (AD) field. Triggers include: sharing a URL to arXiv, CVPR, ICCV, NeurIPS, or other venues, saying 'summarize this paper', 'review this paper', or 'add this to my paper review'. Generates a structured markdown review file covering goal, methods, datasets, metrics/benchmarks, and limitations, then saves it to the paper_review folder."
---

# Paper Review Skill

## Overview

When the user provides a paper (via URL, PDF, or title), fetch its content, extract key information, and produce a structured markdown review file saved to the user's paper review folder.

---

## Workflow

1. **Fetch the paper** — Try WebFetch on the provided URL. If blocked, fall back to WebSearch using the paper title to find the arXiv abstract page or other accessible sources. Gather enough information to fill all sections below.
2. **Write the markdown file** — Use the template below. Fill every section; do not leave sections empty.
3. **Save to the paper review folder** — Save to `/sessions/inspiring-charming-tesla/mnt/paper_review/` (or whatever the user's paper review directory is). The filename must be the full paper title with spaces replaced by underscores, e.g. `CarPlanner_Consistent_Auto-regressive_Trajectory_Planning.md`.
4. **Present the file** — Use `present_files` or provide a `computer://` link so the user can open it.

---

## Output Template

Use exactly this structure for every review:

```markdown
# {Full Paper Title}

**Venue:** {Conference/Journal and Year, e.g. CVPR 2025}
**arXiv:** [{arXiv ID}](https://arxiv.org/abs/{arXiv ID})

---

## Goal / Problem

{2–4 sentences describing: what real-world problem is being solved, why existing approaches fall short, and what gap this paper fills.}

---

## Main Methods

{One paragraph overview, then numbered subsections for each key component.}

### 1. {Component Name}
{Description of this component.}

### 2. {Component Name}
{Description. Include formulas in LaTeX where helpful, e.g.:}

$$p(\tau \mid s, m) = \prod_{t=1}^{T} p(w_t \mid w_{<t}, s, m)$$

### Additional Innovations

- **{Innovation 1}:** {Description.}
- **{Innovation 2}:** {Description.}

---

## Datasets & Simulation Environments

### {Dataset Name}
{Scale, format, collection method, cities/domains covered, annotation types.}

### {Simulator / Benchmark Name}
{How the simulator works, what modes are used (reactive vs. non-reactive), train/val/test split used.}

> {Callout: note any datasets or environments that are NOT used, if relevant.}

---

## Metrics & Benchmarks

**Benchmark:** {Name and brief description.}

| Metric | Description | {This Method} | vs. {Baseline 1} | vs. {Baseline 2} |
|--------|-------------|---------------|------------------|------------------|
| {Metric 1} | {What it measures} | **{value}** | {delta} | {delta} |
| {Metric 2} | {What it measures} | **{value}** | {delta} | {delta} |

> **Key result:** {One sentence summarizing the headline achievement, e.g. "CarPlanner is the first RL-based planner to surpass IL- and rule-based SOTAs on nuPlan."}

---

## Limitations

- {Limitation 1: scope of evaluation, e.g. only tested in non-reactive simulation.}
- {Limitation 2: modeling assumption, e.g. discrete mode space may miss complex maneuvers.}
- {Limitation 3: generalization, e.g. sim-to-real gap not addressed.}
- {Limitation 4: any other noted weakness or missing ablation.}

---

## Sources

- [{Paper title} arXiv]({arXiv URL})
- [{Venue} Open Access]({official proceedings URL})
- [{Dataset/Benchmark name}]({dataset URL})
```

---

## Key Rules

- **File naming:** Full paper title, spaces → underscores. No truncation.
- **Formulas:** Use LaTeX `$$...$$` blocks for any mathematical expressions.
- **Results table:** Always include a comparison table, even if only qualitative deltas are known.
- **Limitations:** Infer limitations from the paper's scope even if not explicitly stated (e.g., single dataset, non-reactive only, no real-world experiments).
- **Sources:** Always end with a Sources section linking to arXiv, venue proceedings, and datasets.
- **Domain focus:** Papers are primarily in autonomous driving. Use domain knowledge to enrich descriptions (e.g., explain what nuPlan CLS-NR means, what closed-loop vs. open-loop evaluation implies).
- **Save location:** Always save to the user's designated paper review folder and provide a computer:// link.
