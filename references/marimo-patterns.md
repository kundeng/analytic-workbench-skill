# marimo Patterns Reference

marimo is the **frontend layer** of the analytic workbench. It owns display,
interactivity, and layout. It never contains business logic — that belongs in
Hamilton modules under `src/`.

## Table of Contents
1. [Philosophy: marimo as Frontend](#philosophy)
2. [Notebook Structure](#notebook-structure)
3. [Interactive Exploration (Tier 1+)](#interactive-exploration)
4. [Report App (Tier 2+)](#report-app)
5. [UI Elements for Parameter Tuning](#ui-elements)
6. [State and Stateful Interactions](#state)
7. [Script Mode and CLI Arguments](#script-mode)
8. [App Deployment](#app-deployment)
9. [Anti-Patterns](#anti-patterns)

---

## 1. Philosophy: marimo as Frontend {#philosophy}

The rule is simple: **marimo displays, Hamilton computes.**

A marimo notebook should:

- Create a Hamilton `Driver` from `src/` modules
- Gather parameters from UI widgets
- Call `dr.execute(outputs, inputs=params)` to get results
- Display those results (figures, tables, metrics, markdown)

A marimo notebook should never:

- Contain `df.groupby(...)`, `ts.rolling(...)`, or any transform logic
- Read raw data directly (`pd.read_csv` in a notebook cell)
- Compute metrics or statistics inline
- Define functions that belong in `src/`

If you find yourself writing more than 3 lines of non-display code in a cell,
that code belongs in a `src/` module.

---

## 2. Notebook Structure {#notebook-structure}

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
    from hamilton import driver
    import src.baseline as baseline
    import src.features as features
    return mo, driver, baseline, features


@app.cell
def _(mo):
    mo.md("# Interactive Exploration")
    return
```

**Key rules:**

- Each cell is a function. The last expression is displayed.
- Return variables you want other cells to see (note the trailing comma).
- Imports go in their own cell and are returned.
- marimo determines execution order from variable references, not cell position.

---

## 3. Interactive Exploration (Tier 1+) {#interactive-exploration}

This is the primary pattern. The notebook creates a Hamilton Driver, gathers
parameters from widgets, executes the DAG, and displays results.

```python
@app.cell
def _(mo):
    freq = mo.ui.dropdown(
        options=["10min", "30min", "1h", "4h", "1d"],
        value="1h",
        label="Resample frequency",
    )
    window = mo.ui.slider(12, 336, value=24, step=12, label="Window size (hours)")
    threshold = mo.ui.number(1.0, 10.0, value=3.0, step=0.5, label="Anomaly threshold")
    mo.hstack([freq, window, threshold])
    return freq, window, threshold


@app.cell
def _(driver, baseline, features):
    # Build the Driver once — modules don't change during a session
    dr = (
        driver.Builder()
        .with_modules(baseline, features)
        .build()
    )
    return dr,


@app.cell
def _(dr, freq, window, threshold):
    # Execute the DAG with widget values as inputs
    results = dr.execute(
        ["summary_stats", "timeseries_figure", "discords"],
        inputs={
            "raw_data_path": "rawdata/events.csv",
            "resample_freq": freq.value,
            "window_size": window.value,
            "anomaly_threshold": threshold.value,
        },
    )
    return results,


@app.cell
def _(results):
    # Display the figure — Hamilton node returned a matplotlib Figure
    results["timeseries_figure"]
    return


@app.cell
def _(results, mo):
    # Display metrics
    stats = results["summary_stats"]
    mo.md(f"**Total periods:** {stats['total_periods']:,} | "
          f"**Mean:** {stats['mean']:.2f} | "
          f"**Zero rate:** {stats['zero_rate']:.1%}")
    return


@app.cell
def _(results):
    # Display discords table — interactive by default in marimo
    results["discords"]
    return
```

Move a slider → `results` recomputes → all downstream cells update. No
callbacks, no event handlers. The computation happens inside Hamilton, not in
the notebook.

---

## 4. Report App (Tier 2+) {#report-app}

The report app loads pre-computed artifacts from `runs/`. It does **no
computation** — only display and navigation.

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
def _(mo):
    mo.md("# Experiment Comparison Report")
    return


@app.cell
def _(pd, Path):
    # Load comparison table
    comp_path = Path("runs") / "comparison.csv"
    if comp_path.exists():
        comparison = pd.read_csv(comp_path)
    else:
        comparison = pd.DataFrame({"status": ["No comparison table found"]})
    comparison
    return comparison,


@app.cell
def _(mo, comparison):
    if "run_id" in comparison.columns:
        run_selector = mo.ui.dropdown(
            options=comparison["run_id"].tolist(),
            label="Inspect run",
        )
        run_selector
    return run_selector,


@app.cell
def _(run_selector, json, Path, mo):
    run_dir = Path("runs") / run_selector.value
    if not run_dir.exists():
        mo.md("Run directory not found")
    else:
        # Show metrics
        metrics_path = run_dir / "metrics.json"
        if metrics_path.exists():
            metrics = json.loads(metrics_path.read_text())
            mo.tree(metrics)

        # Show figures
        figs = sorted((run_dir / "figures").glob("*.png"))
        if figs:
            mo.vstack([mo.image(src=str(f)) for f in figs])
    return
```

Run as app: `marimo run notebooks/report.py`

---

## 5. UI Elements for Parameter Tuning {#ui-elements}

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
mo.ui.text(value="rawdata/events.csv", label="Input path")

# Layout
mo.hstack([slider, dropdown, checkbox])  # horizontal
mo.vstack([chart1, chart2])              # vertical
mo.tabs({"Overview": tab1, "Details": tab2})  # tabbed
mo.accordion({"Section A": content_a})   # collapsible
```

---

## 6. State and Stateful Interactions {#state}

For interactions that need to persist across reactive updates (selections,
accumulated results, user annotations), use `mo.state()`:

```python
@app.cell
def _(mo):
    get_selected, set_selected = mo.state([])
    return get_selected, set_selected


@app.cell
def _(mo, comparison, set_selected):
    # Table with row selection
    table = mo.ui.table(
        comparison,
        selection="multi",
        on_change=lambda rows: set_selected(rows),
    )
    table
    return table,


@app.cell
def _(get_selected, mo):
    selected = get_selected()
    if selected:
        mo.md(f"**{len(selected)} runs selected** for detailed comparison")
    return selected,
```

Use `mo.state()` sparingly — most interactions work through reactive widget
values without explicit state.

---

## 7. Script Mode and CLI Arguments {#script-mode}

Run a marimo notebook as a script (no browser, no interactivity):

```bash
marimo run notebooks/explore.py
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

## 8. App Deployment {#app-deployment}

marimo notebooks can be deployed as standalone web apps:

```bash
# Local app server
marimo run notebooks/report.py --host 0.0.0.0 --port 8080

# Hide code, show only outputs
marimo run notebooks/report.py --include-code false
```

For the workbench, the report app is the primary deployment target. It reads
from `runs/` and presents the comparison table, per-run drill-down, and
figures. The human reviews here; the AI reviews programmatically.

---

## 9. Anti-Patterns {#anti-patterns}

**Computation in cells.** If a cell contains pandas transforms, statistical
calculations, or model fitting, move that code to a `src/` module and call
it through the Hamilton Driver.

**Direct data loading.** `pd.read_csv("rawdata/...")` in a notebook cell
bypasses Hamilton's dependency tracking. Instead, create a Hamilton node that
loads data and request it through the Driver.

**Fat notebooks.** If your notebook has more than ~15 cells, it's doing too
much. Split into separate notebooks (explore, report) or move logic to `src/`.

**Mixing explore and report.** The exploration notebook creates a Driver and
runs live computation. The report notebook loads pre-computed artifacts. Don't
combine these — they serve different purposes and different audiences.
