# Code Templates Reference

Complete working examples for each component of the analytic workbench pattern.
Copy and adapt these to bootstrap a new project.

## Table of Contents
1. [Project Scaffold](#scaffold)
2. [Hamilton-Compatible Module](#module)
3. [marimo Exploration Notebook](#explore-notebook)
4. [marimo Report App](#report-app)
5. [Hydra Sweep Runner](#sweep-runner)
6. [Comparison Table Builder](#comparison-builder)
7. [Freshness-Aware Data Loader](#data-loader)

---

## 1. Project Scaffold {#scaffold}

```bash
mkdir -p conf/experiment conf/source modules notebooks scripts outputs/runs data/raw data/processed
```

Minimal files to create:
```
conf/config.yaml
modules/__init__.py       # empty
modules/baseline.py
notebooks/explore.py      # marimo notebook
scripts/run.py            # Hydra entry point
scripts/build_comparison.py
```

---

## 2. Hamilton-Compatible Module {#module}

```python
# modules/baseline.py
"""Time series construction from raw data. Hamilton-compatible conventions."""
import pandas as pd
import numpy as np
from typing import Dict, Any, Tuple


def timeseries_hourly(
    raw_data: pd.DataFrame,
    date_column: str,
    resample_freq: str,
) -> pd.Series:
    """Resample raw data to fixed-frequency counts."""
    df = raw_data.copy()
    df[date_column] = pd.to_datetime(df[date_column], errors="coerce")
    ts = df.set_index(date_column).resample(resample_freq).size()
    ts.name = "count"
    return ts


def summary_stats(timeseries_hourly: pd.Series) -> Dict[str, float]:
    """Descriptive statistics for the time series."""
    return {
        "total_periods": len(timeseries_hourly),
        "total_count": int(timeseries_hourly.sum()),
        "mean": float(timeseries_hourly.mean()),
        "median": float(timeseries_hourly.median()),
        "std": float(timeseries_hourly.std()),
        "max": int(timeseries_hourly.max()),
        "zero_rate": float((timeseries_hourly == 0).mean()),
    }


def timeseries_figure(
    timeseries_hourly: pd.Series,
    resample_freq: str,
) -> "matplotlib.figure.Figure":
    """Create a time series plot. Returns the figure object."""
    import matplotlib.pyplot as plt
    fig, ax = plt.subplots(figsize=(14, 4))
    ax.plot(timeseries_hourly.index, timeseries_hourly.values, linewidth=0.5)
    ax.set_title(f"Event Count ({resample_freq} bins)")
    ax.set_ylabel("Count")
    ax.set_xlabel("Time")
    plt.tight_layout()
    return fig


def save_baseline_artifacts(
    timeseries_hourly: pd.Series,
    timeseries_figure: "matplotlib.figure.Figure",
    summary_stats: Dict[str, float],
    output_dir: str,
) -> Dict[str, str]:
    """Save all baseline artifacts. Returns dict of file paths."""
    import json
    from pathlib import Path

    out = Path(output_dir)
    out.mkdir(parents=True, exist_ok=True)
    (out / "figures").mkdir(exist_ok=True)

    ts_path = str(out / "timeseries.csv")
    fig_path = str(out / "figures" / "fig-timeseries.png")
    metrics_path = str(out / "metrics.json")

    timeseries_hourly.to_csv(ts_path)
    timeseries_figure.savefig(fig_path, dpi=150)
    json.dump(summary_stats, open(metrics_path, "w"), indent=2)

    return {"timeseries": ts_path, "figure": fig_path, "metrics": metrics_path}
```

---

## 3. marimo Exploration Notebook {#explore-notebook}

```python
# notebooks/explore.py
import marimo

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
    mo.md("# Interactive Exploration")
    return


@app.cell
def _(pd):
    df = pd.read_csv("data/raw/events.csv", parse_dates=["timestamp"])
    mo.md(f"**Loaded {len(df):,} rows** | "
          f"Range: {df['timestamp'].min()} to {df['timestamp'].max()}")
    return df,


@app.cell
def _(mo):
    freq = mo.ui.dropdown(
        options=["10min", "30min", "1h", "4h", "1d"],
        value="1h",
        label="Resample frequency",
    )
    freq
    return freq,


@app.cell
def _(df, freq):
    # Import from your Hamilton-compatible module
    from modules.baseline import timeseries_hourly, summary_stats

    ts = timeseries_hourly(df, date_column="timestamp", resample_freq=freq.value)
    stats = summary_stats(ts)
    stats
    return ts, stats


@app.cell
def _(ts, freq, plt):
    fig, ax = plt.subplots(figsize=(14, 4))
    ax.plot(ts.index, ts.values, linewidth=0.5)
    ax.set_title(f"Event Count ({freq.value})")
    plt.tight_layout()
    fig
    return


if __name__ == "__main__":
    app.run()
```

Run: `marimo edit notebooks/explore.py`

---

## 4. marimo Report App {#report-app}

```python
# notebooks/report.py
"""Final review surface — loads artifacts, no computation."""
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
    # Find all comparison tables
    comp_files = sorted(Path("outputs").rglob("comparison.csv"))
    if comp_files:
        comparison = pd.read_csv(comp_files[-1])  # most recent
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
    # Find the run directory
    candidates = list(Path("outputs").rglob(f"*{run_selector.value}*"))
    run_dirs = [c for c in candidates if c.is_dir()]
    if not run_dirs:
        mo.md("Run directory not found")
    else:
        run_dir = run_dirs[0]
        # Show metrics
        metrics_path = run_dir / "metrics.json"
        if metrics_path.exists():
            metrics = json.loads(metrics_path.read_text())
            mo.md(f"### Metrics\n```json\n{json.dumps(metrics, indent=2)}\n```")

        # Show figures
        figs = sorted(run_dir.rglob("*.png"))
        if figs:
            mo.vstack([mo.image(src=str(f)) for f in figs])
    return


if __name__ == "__main__":
    app.run()
```

Run: `marimo edit notebooks/report.py` or `marimo run notebooks/report.py`

---

## 5. Hydra Sweep Runner {#sweep-runner}

```python
# scripts/run.py
"""Hydra entry point for single runs and multirun sweeps."""
import hydra
from omegaconf import DictConfig, OmegaConf
import json
from pathlib import Path


@hydra.main(version_base=None, config_path="../conf", config_name="config")
def main(cfg: DictConfig) -> float:
    import pandas as pd
    from modules.baseline import (
        timeseries_hourly,
        summary_stats,
        timeseries_figure,
        save_baseline_artifacts,
    )

    # Hydra changes cwd to the output directory
    out = Path(".")

    # Load data
    df = pd.read_csv(cfg.source.path, parse_dates=[cfg.source.date_column])

    # Run analysis using Hamilton-compatible functions
    ts = timeseries_hourly(df, cfg.source.date_column, cfg.baseline.resample_freq)
    stats = summary_stats(ts)
    fig = timeseries_figure(ts, cfg.baseline.resample_freq)

    # Save artifacts
    save_baseline_artifacts(ts, fig, stats, str(out))

    # Save frozen config
    OmegaConf.save(cfg, out / "config.yaml")

    # Return a metric for Hydra job results
    return stats.get("total_count", 0)


if __name__ == "__main__":
    main()
```

```bash
# Single run
python scripts/run.py baseline.resample_freq=1h

# Sweep
python scripts/run.py -m baseline.resample_freq=10min,30min,1h,4h

# Named experiment
python scripts/run.py +experiment=fast_test
```

---

## 6. Comparison Table Builder {#comparison-builder}

```python
# scripts/build_comparison.py
"""Aggregate metrics from all runs into a single comparison table."""
import json
import pandas as pd
from pathlib import Path
from omegaconf import OmegaConf
import sys


def build_comparison(sweep_dir: str) -> pd.DataFrame:
    rows = []
    for run_dir in sorted(Path(sweep_dir).iterdir()):
        if not run_dir.is_dir():
            continue
        metrics_path = run_dir / "metrics.json"
        config_path = run_dir / "config.yaml"
        if not metrics_path.exists():
            continue

        metrics = json.loads(metrics_path.read_text())
        row = {"run_id": run_dir.name}

        if config_path.exists():
            cfg = OmegaConf.load(config_path)
            # Extract key params — customize these for your project
            row["resample_freq"] = OmegaConf.select(cfg, "baseline.resample_freq", default="")
            # Add more params as needed

        row.update(metrics)
        rows.append(row)

    df = pd.DataFrame(rows)
    out_path = Path(sweep_dir) / "comparison.csv"
    df.to_csv(out_path, index=False)
    print(f"Comparison table: {out_path} ({len(df)} runs)")
    return df


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python scripts/build_comparison.py <sweep_dir>")
        sys.exit(1)
    build_comparison(sys.argv[1])
```

---

## 7. Freshness-Aware Data Loader {#data-loader}

```python
# modules/data_loader.py
"""Load data with freshness-aware caching."""
import json
from datetime import datetime, timezone, timedelta
from pathlib import Path
from typing import Optional


def fetch_metadata_path(data_path: str) -> str:
    """Convention: metadata file sits next to the data file."""
    return str(Path(data_path).with_suffix(".meta.json"))


def is_fresh(data_path: str, freshness_hours: int = 48) -> bool:
    """Check if cached data is still fresh enough to reuse."""
    meta_path = fetch_metadata_path(data_path)
    if not Path(meta_path).exists() or not Path(data_path).exists():
        return False
    meta = json.loads(Path(meta_path).read_text())
    fetched = datetime.fromisoformat(meta["fetched_at"])
    return datetime.now(timezone.utc) - fetched < timedelta(hours=freshness_hours)


def record_fetch(data_path: str, row_count: int, source: str = "") -> None:
    """Record when data was fetched for freshness tracking."""
    meta = {
        "source": source,
        "fetched_at": datetime.now(timezone.utc).isoformat(),
        "row_count": row_count,
        "data_path": data_path,
    }
    Path(fetch_metadata_path(data_path)).write_text(json.dumps(meta, indent=2))


def load_or_fetch(
    data_path: str,
    fetch_fn,
    freshness_hours: int = 48,
    source: str = "",
):
    """Load cached data if fresh, otherwise fetch and cache."""
    import pandas as pd

    if is_fresh(data_path, freshness_hours):
        print(f"Using cached data: {data_path}")
        return pd.read_csv(data_path)

    print(f"Fetching fresh data -> {data_path}")
    df = fetch_fn()
    Path(data_path).parent.mkdir(parents=True, exist_ok=True)
    df.to_csv(data_path, index=False)
    record_fetch(data_path, len(df), source)
    return df
```

Usage:
```python
from modules.data_loader import load_or_fetch

df = load_or_fetch(
    "data/raw/incidents.csv",
    fetch_fn=lambda: my_splunk_extract(),
    freshness_hours=48,
    source="splunk:index=snow",
)
```
