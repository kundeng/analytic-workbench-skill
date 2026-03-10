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

marimo notebooks are pure Python files. Each cell is a function decorated with
`@app.cell`. Variables defined in one cell are automatically available to cells
that reference them — marimo tracks dependencies and re-executes dependents
reactively.

```python
# notebooks/explore.py
import marimo

app = marimo.App(width="medium")

@app.cell
def _():
    import marimo as mo
    import pandas as pd
    import matplotlib.pyplot as plt
    return mo, pd, plt

@app.cell
def _(mo):
    mo.md("# Exploratory Analysis")
    return

@app.cell
def _(pd):
    df = pd.read_csv("data/raw/incidents.csv", parse_dates=["opened_at"])
    df
    return df,

@app.cell
def _(df, mo):
    freq_slider = mo.ui.dropdown(
        options=["10min", "30min", "1h", "4h"],
        value="1h",
        label="Resample frequency"
    )
    freq_slider
    return freq_slider,

@app.cell
def _(df, freq_slider, plt):
    ts = df.set_index("opened_at").resample(freq_slider.value).size()
    fig, ax = plt.subplots(figsize=(14, 4))
    ax.plot(ts.index, ts.values, linewidth=0.5)
    ax.set_title(f"Incident Count ({freq_slider.value} bins)")
    plt.tight_layout()
    fig
    return ts,
```

**Key rules:**
- Each cell is a function. The last expression is displayed.
- Return variables you want other cells to see (note the trailing comma).
- Imports go in their own cell and are returned.
- marimo determines execution order from variable references, not cell position.

---

## 2. Reactive Cells {#reactive-cells}

When a UI element changes (slider, dropdown, checkbox), marimo automatically
re-executes every cell that depends on that element's `.value`. This is the
core advantage over Jupyter — no manual re-run, no stale state.

```python
@app.cell
def _(mo):
    window = mo.ui.slider(12, 336, value=24, step=12, label="Window size (hours)")
    threshold = mo.ui.number(1.0, 10.0, value=3.0, step=0.5, label="Anomaly threshold")
    mo.hstack([window, threshold])
    return window, threshold

@app.cell
def _(ts, window, threshold):
    import stumpy
    mp = stumpy.stump(ts.values, m=window.value)
    anomalies = mp[:, 0] > threshold.value
    f"Found {anomalies.sum()} anomalies with window={window.value}h, threshold={threshold.value}"
    return mp, anomalies
```

Move the slider → the matrix profile recomputes → the anomaly count updates.
No callbacks, no event handlers, no `on_change`.

---

## 3. UI Elements for Parameter Tuning {#ui-elements}

Common elements for analytic workbenches:

```python
# Continuous parameter
mo.ui.slider(0.1, 10.0, value=3.0, step=0.1, label="Threshold")

# Categorical choice
mo.ui.dropdown(options=["1h", "4h", "1d"], value="1h", label="Frequency")

# Boolean toggle
mo.ui.checkbox(label="Normalize", value=True)

# Multi-select
mo.ui.multiselect(
    options=["P1", "P2", "P3", "P4"],
    value=["P1", "P2"],
    label="Priority filter"
)

# Text input
mo.ui.text(value="data/raw/incidents.csv", label="Input path")

# Layout
mo.hstack([slider, dropdown, checkbox])  # horizontal
mo.vstack([chart1, chart2])              # vertical
mo.tabs({"Overview": tab1, "Details": tab2})  # tabbed
```

---

## 4. Script Mode & CLI Arguments {#script-mode}

Run a marimo notebook as a script (no browser, no interactivity):

```bash
# Execute and capture output
marimo run notebooks/explore.py

# With CLI arguments (overrides mo.cli_args())
marimo run notebooks/explore.py -- --freq 1h --window 72
```

Access CLI args in the notebook:

```python
@app.cell
def _(mo):
    args = mo.cli_args()
    freq = args.get("freq", "1h")
    window = int(args.get("window", "24"))
    return freq, window
```

This lets the same notebook serve both interactive exploration (browser) and
batch execution (CLI, Hydra sweep runner).

---

## 5. App Mode for Reports {#app-mode}

Build a final review surface as a marimo app:

```python
# notebooks/report.py — loads artifacts, no computation
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
def _(mo):
    mo.md("# Experiment Comparison Report")
    return

@app.cell
def _(pd):
    comparison = pd.read_csv("outputs/comparison.csv")
    comparison
    return comparison,

@app.cell
def _(mo, comparison):
    run_selector = mo.ui.dropdown(
        options=comparison["run_id"].tolist(),
        label="Select run to inspect"
    )
    run_selector
    return run_selector,

@app.cell
def _(run_selector, json, Path):
    run_dir = Path(f"outputs/runs/{run_selector.value}")
    metrics = json.loads((run_dir / "metrics.json").read_text())
    metrics
    return run_dir, metrics

@app.cell
def _(mo, run_dir):
    # Display figures from the selected run
    figs = sorted(run_dir.glob("figures/*.png"))
    mo.vstack([mo.image(src=str(f)) for f in figs])
    return
```

The human opens this app, browses runs via dropdown, sees comparison table and
per-run figures. Same artifacts the AI already reviewed programmatically.

---

## 6. Integration with Analysis Modules {#module-integration}

marimo notebooks import and call functions from Hamilton-compatible modules:

```python
@app.cell
def _(df, freq_slider):
    from modules.baseline import incidents_hourly, incidents_by_priority

    ts = incidents_hourly(df, resample_freq=freq_slider.value)
    ts_by_priority = incidents_by_priority(df, resample_freq=freq_slider.value)
    return ts, ts_by_priority
```

The modules contain the reusable logic. The notebook is the interactive harness.
When the AI edits a module function, marimo picks up the change on next run.

---

## 7. Displaying Artifacts {#displaying-artifacts}

```python
# DataFrames — displayed as interactive tables automatically
df.head(20)

# Matplotlib figures — last expression displayed
fig, ax = plt.subplots()
ax.plot(ts)
fig

# Markdown
mo.md(f"**Total incidents:** {len(df):,}")

# Images from disk
mo.image(src="outputs/figures/anomalies.png")

# JSON/dict — displayed as collapsible tree
metrics_dict

# Side-by-side comparison
mo.hstack([chart_a, chart_b])
```
