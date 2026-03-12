---
name: analytic-workbench
description: "ALWAYS use this skill for human-in-the-loop analytic workflows: exploratory notebook runs, Hydra-managed experiment sweeps, Hamilton-based analysis DAGs, review/approval loops, and optional Tier 3 reproducibility layers such as DVC or Kedro. Use it when the user wants to set up or run an analysis pipeline, choose a tier, compare hyperparameters, build a comparison table, review outputs before approval, run the next stage, decide how marimo, Hydra, Hamilton, DVC, Kedro, Dagster, or Prefect fit together, or evolve an analysis from exploratory work into a structured workbench. Trigger on phrases like analysis pipeline, reproducible analysis, human-in-the-loop, next stage, experiment sweep, hyperparameter comparison, comparison table, marimo, Hydra, Hamilton, DVC, Kedro, Dagster, Prefect, analytic workbench."
---

# Analytic Workbench

Human-directed, AI-operated analysis. The AI drives execution, self-reviews
outputs, and presents artifacts for human approval. The human guides direction,
edits interpretations, and decides what to try next.

---

## Architecture: Hydra → Hamilton → marimo

Three layers, one direction. Each layer has a single job.

```
Hydra (config)  →  Hamilton (computation)  →  marimo (display)
  what to run        how to run it              what the human sees
```

**Hydra** resolves config composition, CLI overrides, and sweep parameters. It
produces a frozen `DictConfig` and hands it off. It never touches DataFrames or
figures.

**Hamilton** takes that config (via `Builder().with_config(...)` and
`driver.execute(..., inputs=...)`) and runs the DAG. It produces typed
intermediate artifacts — Series, DataFrames, floats, figures. It never renders
UI or reads YAML files.

**marimo** is a thin reactive shell. It either calls the Hamilton driver
directly (interactive mode) or loads pre-computed artifacts from `runs/`
(report mode). It owns sliders, dropdowns, layout, tabs. It never contains
business logic.

At Tier 1, Hydra is absent — widget values serve as config. The Hamilton
convention and marimo-as-frontend rule still apply.

---

## Project Setup

### Dependencies

Every project starts with a base install. Do this at scaffold time, not later.

```bash
pip install "sf-hamilton[visualization]" marimo hydra-core omegaconf \
    polars duckdb pandas matplotlib --break-system-packages
```

Optional extras added when needed:

```bash
# Tier 2+ sweeps and tracking
pip install "sf-hamilton[ui,sdk]" --break-system-packages

# Tier 3 persistence
pip install "dvc[s3]" --break-system-packages

# Heavy analytics
pip install stumpy statsmodels scikit-learn --break-system-packages
```

Generate `requirements.txt` at scaffold time:

```bash
pip freeze > requirements.txt
```

#### Library decision rules

| Library | Use when | Avoid when |
|---------|----------|------------|
| **polars** | Fast transforms, lazy evaluation, large-ish data | Hamilton ecosystem friction (some adapters expect pandas) |
| **duckdb** | SQL-oriented analytics, joins across CSVs/parquets, >1GB data | Simple column transforms better expressed as polars/pandas |
| **pandas** | Hamilton node I/O (default), small-medium data, rich ecosystem | Performance-critical transforms on large data |

Prefer polars for transforms, pandas at Hamilton DAG boundaries (inputs/outputs),
duckdb for ad hoc SQL queries over files.

### Directory Structure

Enforce this from Tier 1. No migration needed when moving up tiers.

```
project/
  src/                       # All Python source — Hamilton modules
    __init__.py
    baseline.py              # Hamilton DAG: time series construction
    features.py              # Hamilton DAG: feature engineering
    ...
  notebooks/                 # marimo notebooks — UI only
    explore.py               # Interactive exploration (calls Hamilton driver)
    report.py                # Read-only report app (loads from runs/)
  scripts/                   # Entry points — Hydra runners, comparison builders
    run.py                   # Hydra entry point → Hamilton driver
    build_comparison.py      # Aggregate metrics across runs
  tools/                     # Data access CLI tools (any tier)
    fetch_data.py            # --output PATH --format csv|json
  conf/                      # Tier 2+: Hydra config files
    config.yaml
    source/
    experiment/
  rawdata/                   # Immutable source data (gitignored)
  runs/                      # Per-run artifacts (gitignored)
    <run-id>/
      config.yaml
      metrics.json
      figures/
      data/
  review/                    # Tier 3+: manifest, review, approval files
  requirements.txt           # Base dependencies
  .gitignore
```

Key differences from ad hoc layouts:

- `src/` — all computation code lives here from day one. Not `modules/`, not
  inline in notebooks.
- `rawdata/` — immutable, gitignored, separate from computed outputs.
- `runs/` — flat per-run folders, not nested under `outputs/`. Each run is
  self-contained.
- No `data/processed/` — intermediate artifacts live inside `runs/<run-id>/data/`.
- No `outputs/figures/` — figures live inside `runs/<run-id>/figures/`.

---

## Pick Your Tier

Not every project needs the same ceremony. Pick the tier that matches the
project's current complexity — you can move up later without rewriting because
the directory structure and Hamilton conventions are the same at every tier.

| Tier | When | Config | Computation | Display |
|------|------|--------|-------------|---------|
| **1: Notebook** | Small, exploratory | marimo widget values | Hamilton-convention modules in `src/`, called via Driver | marimo notebook |
| **2: Workbench** | Repeatable experiments, comparison | Hydra configs + sweeps | Hamilton Driver with `with_config()` | marimo app + comparison tables |
| **3: Reproducible** | Expensive data, many runs, ML | Hydra + DVC params | Hamilton + DVC cached stages | marimo for review |
| **4: Orchestrated** | Production, team, CI/CD | Orchestrator config + Hydra | dagster assets or prefect flows wrapping Hamilton | Orchestrator UI + marimo |

**Start at Tier 1 only for truly lightweight work. Most comparison-driven
analyses should begin at Tier 2.** Signs you need the next tier:

- **1→2**: You want to compare parameters systematically. You need Hydra configs
  and per-run folders.
- **2→3**: Re-fetching source data wastes time. You want DVC cached stages.
- **3→4**: Multiple people need scheduling, retries, lineage, or CI/CD.

> **Tool references** (read only when needed for your tier):
> - `references/hamilton-conventions.md` — All tiers: Driver, Builder, function modifiers, DAG patterns
> - `references/marimo-patterns.md` — All tiers: frontend patterns, app mode, UI-only philosophy
> - `references/hydra-config.md` — Tier 2+: config composition, sweeps, experiment configs
> - `references/artifact-strategy.md` — Tier 2+: per-run folders, comparison tables, freshness rules
> - `references/review-workflow.md` — Tier 3+: state machine, human approval flow
> - `references/core-contracts.md` — Tier 3+: manifest.json, review.json, approval.json schemas
> - `references/dvc-guide.md` — Tier 3: `dvc.yaml`, `dvc repro`, `dvc exp`, remotes, caching
> - `references/code-templates.md` — All tiers: complete working examples

---

## Tier 1: Disciplined Exploration

Even at Tier 1, code lives in `src/` as Hamilton-convention functions. The
marimo notebook is a thin UI shell.

### What "Hamilton convention" means at Tier 1

Write Python modules as collections of small, typed functions where:

- function name = the name of the thing it produces
- parameter names = the names of things it depends on
- type hints = the contract
- no side effects in core logic

You can call these directly or (better) through a Hamilton `Driver`. Either way,
the code is ready for Tier 2 with zero changes.

### Tier 1 flow

```
marimo notebook
  ├── UI widgets (sliders, dropdowns) → config dict
  ├── Hamilton Driver from src/ modules
  ├── dr.execute(["output_name"], inputs=config)
  └── Display results (figures, tables, metrics)
```

The notebook never contains `df.groupby(...)` or `ts.rolling(...)`. That logic
lives in `src/`.

---

## Tier 2: Hydra → Hamilton → marimo

### Config layer (Hydra)

Hydra composes config from YAML files and CLI overrides. It produces a frozen
`DictConfig`.

```yaml
# conf/config.yaml
defaults:
  - source: csv_local
  - _self_

baseline:
  resample_freq: 1h
  date_column: opened_at

analysis:
  window_size: 24
  anomaly_threshold: 3.0
```

### Computation layer (Hamilton)

The runner script builds a Hamilton Driver with Hydra config and executes:

```python
# scripts/run.py
import hydra
from hydra.core.hydra_config import HydraConfig
from omegaconf import DictConfig, OmegaConf
from hamilton import driver
from hamilton.io.materialization import to
import src.baseline as baseline
import src.features as features

@hydra.main(version_base=None, config_path="../conf", config_name="config")
def main(cfg: DictConfig) -> None:
    out = Path(HydraConfig.get().runtime.output_dir)

    dr = (
        driver.Builder()
        .with_modules(baseline, features)
        .build()
    )

    inputs = OmegaConf.to_container(cfg, resolve=True)
    results = dr.execute(
        ["summary_stats", "timeseries_figure", "discords"],
        inputs=inputs,
    )

    # Save artifacts to run folder
    save_artifacts(results, out)
```

Note: Hydra produces the config dict. Hamilton consumes it as `inputs`. Hamilton's
own `with_config()` is used for node selection (`@config.when`), not for passing
parameter values.

### Display layer (marimo)

Two modes:

**Interactive exploration** — marimo creates its own Driver, passes widget
values as inputs, displays results live.

**Report/review** — marimo loads pre-computed artifacts from `runs/`, provides
dropdowns to browse runs, displays comparison tables and figures.

See `references/marimo-patterns.md` for detailed patterns.

### Systematic sweeps

```bash
python scripts/run.py -m baseline.resample_freq=10min,30min,1h,4h
python scripts/build_comparison.py runs/
```

Each run saves to its own folder under `runs/`. The comparison builder reads
`metrics.json` from every run folder.

---

## The Core Loop

Regardless of tier, every analysis cycle follows:

```
Execute → Self-Review → Present → Human Decision → Record & Advance
```

### Execute

Run the analysis (Hamilton driver, Hydra sweep, DVC repro). Produce outputs:
data files, figures, metrics — all inside `runs/<run-id>/`.

### Self-Review

Before showing the human anything, the AI checks its own work:

| Check | How | Fail → |
|-------|-----|--------|
| Outputs exist and non-empty | `ls`, file sizes | Fix and re-run |
| Figures non-trivial | View PNGs (vision) | Regenerate |
| Metrics plausible | Read metrics.json, check ranges | Investigate |
| No NaN/Inf in key columns | Scan DataFrames | Clean data or fix logic |
| Values match figures | Compare summary numbers to visual | Fix inconsistency |

At **Tier 1–2**, self-review is a mental checklist the AI runs before speaking.
At **Tier 3+**, write `review.json` with pass/fail per check.

### Present

Show the human: what ran, key outputs, AI interpretation (draft), recommendation.

At **Tier 1**, this is a chat message with inline figures.
At **Tier 2+**, write a `card.md` summarizing the stage.
At **Tier 3+**, write formal `manifest.json` + `review.json` + `card.md`.

### Human Decision

Approve, approve with edits, or reject with feedback.

### Record & Advance

At **Tier 1–2**: note approval in conversation and move on.
At **Tier 3+**: write `approval.json`. Never update a report with unapproved results.

---

## AI Editing Guidance

| Action | How |
|--------|-----|
| Edit analysis logic | Modify small functions in `src/`. Hamilton style isolates concerns. |
| Run quick exploration | Execute marimo notebook — it calls Hamilton Driver with widget inputs. |
| Create artifacts | Hamilton nodes produce figures/data. Save functions write to `runs/`. |
| Add derived outputs | New function in `src/` module. Import module in Driver. No ceremony. |
| Compare runs | Read per-run `metrics.json`, build DataFrame, save `comparison.csv`. |
| Build reports | marimo app loading artifacts: comparison table + per-run drill-down. |
| Run sweeps | Hydra config + `--multirun`. Each run saves to `runs/<run-id>/`. |
| Self-review | Read metrics for sanity, view figures via vision, validate data ranges. |
| Install new library | `pip install <lib> --break-system-packages`, update `requirements.txt`. |

---

## Maturity Path

Each phase builds on the previous without rewrites.

**Phase 1** — marimo + Hamilton Driver + `src/` modules + `runs/` folder.
**Phase 2** — Hydra configs + Hamilton Driver + comparison tables.
**Phase 3** — DVC caching + formal review contracts.
**Phase 4** — dagster or prefect orchestration + CI/CD.

Move up when the pain of not having the next tool exceeds the cost of adding it.
