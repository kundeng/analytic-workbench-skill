---
name: analytic-workbench
description: "ALWAYS use this skill for human-in-the-loop analytic workflows: exploratory notebook runs, Hydra-managed experiment sweeps, Hamilton-based analysis DAGs, review/approval loops, and optional Tier 3 reproducibility layers such as DVC or Kedro. Use it when the user wants to set up or run an analysis pipeline, choose a tier, compare hyperparameters, build a comparison table, review outputs before approval, run the next stage, decide how marimo, Hydra, Hamilton, DVC, Kedro, Dagster, or Prefect fit together, or evolve an analysis from exploratory work into a structured workbench. Trigger on phrases like analysis pipeline, reproducible analysis, human-in-the-loop, next stage, experiment sweep, hyperparameter comparison, comparison table, marimo, Hydra, Hamilton, DVC, Kedro, Dagster, Prefect, analytic workbench."
---

# Analytic Workbench

Human-directed, AI-operated analysis. The AI drives execution, self-reviews
outputs, and presents artifacts for human approval. The human guides direction,
edits interpretations, and decides what to try next.

---

## Pick Your Tier

Not every project needs the same ceremony. Pick the tier that matches the
project's current complexity — you can move up later without rewriting.

| Tier | When | Surface | Logic | Config | Artifacts | Review |
|------|------|---------|-------|--------|-----------|--------|
| **1: Notebook** | Small, exploratory, disposable work | marimo or `# %%` | Functions in notebook | Function args, UI widgets | Single output folder | Informal: AI shows, human eyeballs |
| **2: Workbench** | Repeatable experiments and hyperparameter comparison | marimo + report app | Hamilton modules or Hamilton-compatible modules | Hydra configs + sweeps | Per-run folders + comparison table | Lightweight: summary.json + card |
| **3: Reproducible ML** | Expensive data, many runs, or ML-heavy structure needs | marimo for review | Hamilton + optional DVC or Kedro layer | Hydra + DVC params or Kedro config | Cached stages or structured ML pipelines | Full: manifest + review + approval |
| **4: Orchestrated** | Production-adjacent, team, CI/CD | dagster or prefect UI + marimo | dagster assets, prefect flows, or Hamilton | orchestrator config + Hydra | orchestrator materializations + registry | Full + CI integration |

**Start at Tier 1 only for truly lightweight work. Most real comparison-driven analyses should begin at Tier 2.** Signs you need the next tier:

- **1→2**: You're copy-pasting functions between notebooks. You want to compare
  hyperparameters systematically. You need explicit run folders and a
  comparison table.
- **2→3**: Re-fetching source data wastes time, you want persistent cached
  stages, or the ML project structure is getting large enough to need a more
  formal pipeline framework.
- **3→4**: Multiple people need to run the pipeline. You need scheduling,
  retries, lineage, or integration with production systems.

**For hyperparameter-heavy feasibility work like ServiceNow prealerting, start at Tier 2.**

> **Tool references** (read only when needed for your tier):
> - `references/marimo-patterns.md` — Tier 1+: reactive notebooks, UI elements, app mode
> - `references/hamilton-conventions.md` — Tier 2+: module conventions, DAG patterns, when to add runtime
> - `references/hydra-config.md` — Tier 2+: config composition, sweeps, experiment configs
> - `references/artifact-strategy.md` — Tier 2+: per-run folders, comparison tables, freshness rules
> - `references/review-workflow.md` — Tier 3+: state machine, human approval flow, data access and reporting rules
> - `references/core-contracts.md` — Tier 3+: manifest.json, review.json, approval.json schemas
> - `references/dvc-guide.md` — Tier 3A: `dvc.yaml`, `dvc repro`, `dvc exp`, remotes, caching
> - `references/code-templates.md` — All tiers: complete working examples

---

## The Core Loop

Regardless of tier, every analysis cycle follows the same pattern. The ceremony
scales with tier — Tier 1 does this conversationally, Tier 3 writes JSON files —
but the rhythm is always:

```
Execute → Self-Review → Present → Human Decision → Record & Advance
```

### Execute

Run the analysis (a notebook cell, a script, a pipeline stage). Produce outputs:
data files, figures, metrics.

### Self-Review

Before showing the human anything, the AI checks its own work:

| Check | How | Fail → |
|-------|-----|--------|
| Outputs exist and non-empty | `ls`, file sizes | Fix and re-run |
| Figures non-trivial | View PNGs (vision) | Regenerate |
| Metrics plausible | Read summary/metrics, check ranges | Investigate |
| No NaN/Inf in key columns | Scan DataFrames | Clean data or fix logic |
| Values match figures | Compare summary numbers to visual | Fix inconsistency |

At **Tier 1–2**, self-review is a mental checklist the AI runs before speaking.
At **Tier 3+**, write `review.json` with pass/fail per check.
See `references/core-contracts.md` for the full review schema.

### Present

Show the human what happened. The presentation always includes:

1. **What ran** — which stage, what parameters
2. **Key outputs** — figures, tables, metrics
3. **AI interpretation** — what the results mean (draft, pending approval)
4. **Recommendation** — what to do next

At **Tier 1**, this is a chat message with inline figures.
At **Tier 2+**, write a `card.md` summarizing the stage.
At **Tier 3+**, write formal `manifest.json` + `review.json` + `card.md`.

### Human Decision

The human reviews and decides:

- **Approve** — artifacts and interpretation accepted, advance to next stage
- **Approve with edits** — artifacts OK, but human rewrites the interpretation
- **Reject + feedback** — something's wrong, AI should fix and retry

The human approves **both artifacts and interpretations**. The AI drafts analysis
text, but the human's version is what enters any final report.

### Record & Advance

At **Tier 1–2**: The AI notes approval in conversation and moves on.
At **Tier 3+**: Write `approval.json` with the approved interpretation.
Never update a report or narrative with unapproved results.

At **Tier 3+**, treat execution as a small state machine:

```
STALE → EXECUTED → REVIEWED → APPROVED
                 ↘ REJECTED ↗
```

The detailed workflow, file-writing sequence, and narrative update rules live in
`references/review-workflow.md` and `references/core-contracts.md`.

---

## Two Operating Modes

### Mode A: Interactive Exploration

The human or AI wants to try something quickly — run a subset, tweak a
parameter, eyeball a chart, decide what's next.

**Use:** marimo notebook (Tier 1+) or plain Python with `# %%` cells.
At Tier 2, import Hamilton modules or Hamilton-compatible modules. Parameters
come from marimo UI widgets or function arguments — no sweep/config system
needed yet for one-off exploration.

```
Human: "Try window_size=72, see if anomalies shift"
  AI: tweaks parameter in marimo / edits module function
  AI: runs, inspects outputs, self-reviews
  AI: shows results to human
Human: "Interesting. Now try 168."
```

### Mode B: Systematic Experiments

Compare multiple parameter settings in a structured, reproducible way.

**Use:** Hydra configs + `--multirun` sweeps (Tier 2+). Same Python modules
as exploration mode — no rewrites. Hamilton runtime is a valid Tier 2 choice
when the internal dependency graph is useful to execute or visualize. Each run
produces artifacts in its own folder. Build a comparison table (one row per run,
all key metrics).

```
Human: "Sweep window_size across [24, 72, 168], compare precision/recall"
  AI: writes Hydra config, runs sweep
  AI: builds comparison table from per-run metrics
  AI: identifies best configs, presents comparison to human
Human: picks winner, decides next experiment
```

The **comparison table** is a first-class artifact. It answers: which settings
work best? What are the tradeoffs? See `references/artifact-strategy.md`.

---

## Project Structure

Adapt to your tier — not everything is needed at Tier 1.

```
project/
  modules/                 # Tier 2+: Hamilton or Hamilton-compatible Python modules
  notebooks/               # Tier 1+: marimo notebooks for exploration + reports
  scripts/                 # Tier 2+: pipeline stage scripts or sweep runners
  tools/                   # Data access CLI tools (any tier)
  conf/                    # Tier 2+: Hydra config files
  data/
    raw/                   # Source data (gitignored)
    processed/             # Stage outputs (gitignored)
  outputs/
    figures/               # Generated figures (gitignored)
    runs/                  # Tier 2+: per-run artifact folders
  review/                  # Tier 3+: manifest, review, approval files
  dvc.yaml                 # Tier 3A: DVC pipeline definition
  kedro.yml                # Tier 3B: Kedro project config (optional)
```

### Script Conventions (Tier 2+)

```python
#!/usr/bin/env python3
"""Stage NN: Description."""
import json, yaml
from pathlib import Path
from datetime import datetime, timezone

params = yaml.safe_load(open("params.yaml"))["section"]
OUTPUT_DIR = Path("data/processed/stage_name")
OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

# ... analysis using Hamilton-compatible module functions ...

# Always produce summary metrics
summary = {"stage": "name", "timestamp": datetime.now(timezone.utc).isoformat(), ...}
json.dump(summary, open(OUTPUT_DIR / "summary.json", "w"), indent=2)
```

---

## Architecture: Four Concerns

These are distinct layers. Don't collapse them into one tool.

### 1. Human Surface

Where humans (and AIs) interact with outputs. **marimo** is preferred because
it's reactive, pure Python, git-friendly, and deployable as an app. Falls back
to plain `# %%` scripts when interactivity isn't needed.

### 2. Reusable Analysis Logic

Plain Python modules with **Hamilton-compatible conventions** or **Hamilton
runtime**: small functions, explicit inputs/outputs, minimal hidden state,
meaningful names. At Tier 2, Hamilton is a first-class option when the internal
DAG helps with selective execution or graph visualization; it is not just a
future migration target. See `references/hamilton-conventions.md`.

### 3. Config & Experiment Control

**Hydra** for config composition, CLI overrides, and `--multirun` sweeps (Tier 2+).
marimo handles interactive parameter tweaking; Hydra handles reproducible,
systematic config management. They complement, not replace, each other.

### 4. Artifact Persistence

Start with explicit folder naming and comparison tables at Tier 2. At Tier 3,
choose the persistence layer that matches the pain point:
- **DVC** when caching, stale-stage execution, and artifact reproducibility
  matter most.
- **Kedro** when ML pipeline structure and modular reusable project organization
  matter most.

Add **dagster** or **prefect** (Tier 4) when you need scheduling, retries, or
team workflows.

When using DVC, keep it in its lane:
- `dvc.yaml` defines stage/file dependencies and cached outputs
- `params.yaml` tracks tunable parameters
- `dvc repro` handles stale-stage execution
- `dvc exp` handles experiment comparison and queuing

Use `references/dvc-guide.md` only when you need the detailed command surface.

---

## AI Editing Guidance

| Action | How |
|--------|-----|
| Edit analysis logic | Modify small functions in modules. Hamilton style isolates concerns. |
| Run quick slices | Execute marimo notebook or call Hamilton outputs/module functions directly. |
| Create artifacts | `fig.savefig()`, `.to_csv()`, `json.dump()` to explicit output paths. |
| Add derived outputs | New function in module, import in marimo. No ceremony. |
| Update comparison tables | Read per-run `metrics.json`, build DataFrame, save `comparison.csv`. |
| Build reports | marimo app loading artifacts: comparison table + per-run drill-down. |
| Run sweeps | Hydra config + `--multirun`. Each run saves to its own folder. |
| Self-review | Read metrics for sanity, view figures via vision, validate data ranges. |

**AI reviews first.** Check artifacts programmatically before presenting them
to the human. The human sees the same artifacts in a friendly format (marimo
app, rendered notebook, or inline in chat).

---

## Maturity Path

Each phase builds on the previous without rewrites.

**Phase 1** — marimo + plain modules + single output folder.
**Phase 2** — marimo + Hydra + Hamilton-compatible modules or Hamilton runtime.
**Phase 3A** — DVC caching + experiment persistence + formal review contracts.
**Phase 3B** — Kedro structure + formal review contracts for ML-heavy projects.
**Phase 4** — dagster or prefect orchestration + registry + CI/CD.

Move up when the pain of not having the next tool exceeds the cost of adding it.
