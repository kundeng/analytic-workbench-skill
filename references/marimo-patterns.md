# marimo Patterns Reference

## Table of Contents
1. [Notebook Structure](#notebook-structure)
2. [Reactive Cells](#reactive-cells)
3. [UI Elements for Parameter Tuning](#ui-elements)
4. [Script Mode & CLI Arguments](#script-mode)
5. [App Mode for Reports](#app-mode)
6. [Integration with Analysis Modules](#module-integration)
7. [Displaying Artifacts](#displaying-artifacts)

---

## 1. Notebook Structure {#notebook-structure}

marimo notebooks are plain Python files. Each cell is a function decorated with
`@app.cell`, and marimo determines execution order from variable dependencies,
not from top-to-bottom position in the file.

```python
# notebooks/explore.py
import marimo

__generated_with = "0.11.0"
app = marimo.App(width="medium")

@app.cell
def _():
    import marimo as mo
    import pandas as pd
    import matplotlib.pyplot as plt
    from pathlib import Path
    return mo, pd, plt, Path

@app.cell
def _(mo):
    mo.md("# Exploratory Analysis")
    return

@app.cell
def _(pd, Path):
    data_path = Path("data/raw/incidents.csv")
    df = pd.read_csv(data_path, parse_dates=["opened_at"])
    df
    return df, data_path

@app.cell
def _(df, mo):
    preview = mo.ui.slider(10, 100, value=25, step=5, label="Preview rows")
    preview
    return preview,

@app.cell
def _(df, preview):
    df.head(preview.value)
    return

if __name__ == "__main__":
    app.run()
```

**Key rules:**
- Each cell is a function; referenced return values become dependencies.
- The last expression is displayed automatically.
- Keep imports in a dedicated cell and return them.
- Return only the variables later cells should depend on.
- Put reusable logic in modules; keep notebooks as orchestration and review
  surfaces.

---

## 2. Reactive Cells {#reactive-cells}

When a UI element changes, marimo re-runs only the cells that depend on that
widget's `.value`. The common pattern is: controls -> filtered data -> derived
metrics -> charts.

```python
@app.cell
def _(mo):
    freq = mo.ui.dropdown(
        options=["15min", "1h", "4h"],
        value="1h",
        label="Resample frequency",
    )
    priority = mo.ui.multiselect(
        options=["P1", "P2", "P3", "P4"],
        value=["P1", "P2"],
        label="Priority filter",
    )
    mo.hstack([freq, priority])
    return freq, priority

@app.cell
def _(df, priority):
    filtered = df[df["priority"].isin(priority.value)]
    filtered
    return filtered,

@app.cell
def _(filtered, freq, pd):
    ts = (
        filtered
        .set_index("opened_at")
        .resample(freq.value)
        .size()
        .rename("incident_count")
        .reset_index()
    )
    ts
    return ts,

@app.cell
def _(ts, mo):
    mo.stop(ts.empty, mo.md("No data for the current filter selection."))
    return

@app.cell
def _(ts, plt, freq):
    fig, ax = plt.subplots(figsize=(12, 4))
    ax.plot(ts["opened_at"], ts["incident_count"], linewidth=1.0)
    ax.set_title(f"Incident Count by {freq.value}")
    ax.set_ylabel("Count")
    fig
    return
```

Useful pattern: guard expensive or invalid downstream work with `mo.stop(...)`
so empty selections fail fast and the notebook stays readable.

---

## 3. UI Elements for Parameter Tuning {#ui-elements}

Use widgets to expose the parameters humans actually need to inspect. Favor a
small control surface over a notebook full of hidden constants.

```python
@app.cell
def _(mo):
    threshold = mo.ui.slider(0.0, 10.0, value=3.0, step=0.1, label="Threshold")
    window = mo.ui.number(6, 336, value=72, step=6, label="Window (hours)")
    normalize = mo.ui.checkbox(label="Normalize counts", value=True)
    metric = mo.ui.dropdown(
        options=["count", "mean_duration", "assignment_changes"],
        value="count",
        label="Metric",
    )
    lookback = mo.ui.text(value="-30d", label="Lookback window")

    controls = mo.vstack([
        mo.md("## Controls"),
        mo.hstack([threshold, window]),
        mo.hstack([normalize, metric]),
        lookback,
    ])
    controls
    return threshold, window, normalize, metric, lookback
```

Other common workbench widgets:

```python
# Date selection
mo.ui.date(label="Start date")

# Free text for paths or search filters
mo.ui.text(value="data/raw/incidents.csv", label="Input path")

# Longer analyst notes or adhoc query fragments
mo.ui.text_area(label="Notes")

# Tabbed review layout
mo.tabs({"Overview": overview_view, "Diagnostics": diagnostics_view})
```

Pattern: keep widgets in one cell, derived parameters in another cell, and
heavy computation downstream. That makes dependency flow obvious.

---

## 4. Script Mode & CLI Arguments {#script-mode}

The same notebook can support both browser-based exploration and batch-style
execution. This is useful when a marimo notebook is also the review surface for
Hydra runs or CI-produced artifacts.

```bash
# Launch the notebook UI
marimo edit notebooks/explore.py

# Execute non-interactively
marimo run notebooks/explore.py

# Pass arguments through to mo.cli_args()
marimo run notebooks/explore.py -- --freq 4h --window 168 --priority P1
```

Access CLI arguments inside a cell:

```python
@app.cell
def _(mo):
    args = mo.cli_args()
    freq = args.get("freq", "1h")
    window = int(args.get("window", "72"))
    priority = args.get("priority", "P1")
    return args, freq, window, priority

@app.cell
def _(mo, freq, window, priority):
    mo.md(
        f"Running with `freq={freq}`, `window={window}`, `priority={priority}`"
    )
    return
```

Pattern: use CLI args for batch defaults, but still expose widgets in notebook
mode so an analyst can override and inspect results interactively.

---

## 5. App Mode for Reports {#app-mode}

Use marimo app mode as a review surface that loads artifacts instead of
recomputing them. This keeps the notebook responsive and makes experiment
comparison easier.

```python
# notebooks/report.py
import marimo

app = marimo.App(width="full")

@app.cell
def _():
    import marimo as mo
    import pandas as pd
    import json
    from pathlib import Path
    return mo, pd, json, Path

@app.cell
def _(pd):
    comparison = pd.read_csv("outputs/comparison.csv")
    comparison
    return comparison,

@app.cell
def _(comparison, mo):
    run_id = mo.ui.dropdown(
        options=comparison["run_id"].tolist(),
        value=comparison["run_id"].iloc[0],
        label="Run",
    )
    run_id
    return run_id,

@app.cell
def _(run_id, json, Path):
    run_dir = Path("outputs/runs") / run_id.value
    metrics = json.loads((run_dir / "metrics.json").read_text())
    figures = sorted((run_dir / "figures").glob("*.png"))
    return run_dir, metrics, figures

@app.cell
def _(mo, comparison, run_id, metrics, figures):
    summary = comparison[comparison["run_id"] == run_id.value]
    gallery = mo.vstack([mo.image(src=str(path)) for path in figures])
    mo.tabs({
        "Summary": summary,
        "Metrics": metrics,
        "Figures": gallery,
    })
    return
```

Pattern: computation happens in pipeline scripts; app mode is for selection,
inspection, and approval.

---

## 6. Integration with Analysis Modules {#module-integration}

marimo works best when notebooks import reusable functions from modules. The
notebook becomes the interactive harness; the module holds the business logic
you want to test, script, and reuse in sweeps.

```python
@app.cell
def _():
    from modules.baseline import (
        incidents_hourly,
        incidents_by_priority,
        rolling_anomaly_score,
    )
    return incidents_hourly, incidents_by_priority, rolling_anomaly_score

@app.cell
def _(df, freq, incidents_hourly, incidents_by_priority):
    ts = incidents_hourly(df, resample_freq=freq.value)
    split = incidents_by_priority(df, resample_freq=freq.value)
    return ts, split

@app.cell
def _(ts, window, threshold, rolling_anomaly_score):
    scored = rolling_anomaly_score(
        ts,
        window_hours=window.value,
        threshold=threshold.value,
    )
    scored
    return scored,
```

Two good patterns for workbenches:

```python
# Keep notebook-only presentation logic local
@app.cell
def _(scored, mo):
    mo.md(f"Flagged **{int(scored['is_anomaly'].sum())}** candidate periods.")
    return

# Put reusable data logic in modules
def rolling_anomaly_score(ts, window_hours, threshold):
    ...
```

When the module changes, rerun the notebook and re-evaluate the outputs. This
is the bridge from exploratory notebook work to Hamilton-compatible pipelines.

---

## 7. Displaying Artifacts {#displaying-artifacts}

marimo handles common analysis outputs directly, so the clean pattern is to
return data objects from cells and let the notebook render them.

```python
# DataFrames render as interactive tables
comparison.head(20)

# Markdown for analyst commentary
mo.md(f"**Total incidents:** {len(df):,}")

# Dicts / JSON-like objects render as inspectable structures
metrics

# Matplotlib figures display when they are the last expression
fig, ax = plt.subplots(figsize=(10, 3))
ax.plot(ts.index, ts.values)
ax.set_title("Hourly incidents")
fig

# Images from previous pipeline stages
mo.image(src="outputs/figures/anomalies.png")

# Compose multiple outputs into one review surface
mo.vstack([
    mo.hstack([summary_table, diagnostics_table]),
    fig,
    mo.md("### Analyst Notes"),
    notes,
])
```

Pattern: use plain Python objects as the interface between cells. Keep file I/O
at the edges, metrics in structured dicts/DataFrames, and final review surfaces
composed with `mo.hstack`, `mo.vstack`, and `mo.tabs`.
