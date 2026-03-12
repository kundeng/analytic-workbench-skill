# Code Templates Reference

Complete working examples for each component of the analytic workbench.
Copy and adapt these to bootstrap a new project.

## Table of Contents
1. [Project Scaffold](#scaffold)
2. [Hamilton Module (src/)](#module)
3. [Hamilton Driver Entry Point](#driver-entry)
4. [marimo Exploration Notebook](#explore-notebook)
5. [marimo Report App](#report-app)
6. [Hydra Sweep Runner](#sweep-runner)
7. [Comparison Table Builder](#comparison-builder)
8. [Freshness-Aware Data Loader](#data-loader)
9. [requirements.txt](#requirements)

---

## 1. Project Scaffold {#scaffold}

```bash
mkdir -p src notebooks scripts tools conf/source conf/experiment rawdata runs
touch src/__init__.py
```

Then install dependencies:

```bash
pip install "sf-hamilton[visualization]" marimo hydra-core omegaconf \
    polars duckdb pandas matplotlib --break-system-packages
pip freeze > requirements.txt
```

Create `.gitignore`:

```
rawdata/
runs/
__pycache__/
*.pyc
.env
*.egg-info/
```

---

## 2. Hamilton Module (src/) {#module}

Every module in `src/` follows Hamilton conventions: function name = output
name, parameters = dependencies, type hints on everything.

```python
# src/baseline.py
"""Time series construction from raw data."""
import pandas as pd
import numpy as np
from typing import Dict, Any


def raw_data(raw_data_path: str) -> pd.DataFrame:
    """Load raw data from CSV."""
    return pd.read_csv(raw_data_path, parse_dates=True)


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
```

Notice:

- `raw_data` takes a path and returns a DataFrame — data loading is a DAG node.
- `timeseries_hourly` depends on `raw_data` by parameter name.
- `summary_stats` depends on `timeseries_hourly` by parameter name.
- No I/O in core logic except the explicit `raw_data` loader.
- Figures are returned as objects, not saved to disk inside the function.

---

## 3. Hamilton Driver Entry Point {#driver-entry}

The Driver wires modules together and executes the DAG. This pattern works
at every tier.

```python
# scripts/run_hamilton.py
"""Standalone Hamilton execution — no Hydra, no marimo."""
from hamilton import driver
from pathlib import Path
import json
import src.baseline as baseline


def run(params: dict, output_dir: str) -> dict:
    """Execute Hamilton DAG and save artifacts."""
    out = Path(output_dir)
    out.mkdir(parents=True, exist_ok=True)
    (out / "figures").mkdir(exist_ok=True)
    (out / "data").mkdir(exist_ok=True)

    dr = (
        driver.Builder()
        .with_modules(baseline)
        .build()
    )

    results = dr.execute(
        ["summary_stats", "timeseries_figure", "timeseries_hourly"],
        inputs=params,
    )

    # Save artifacts
    results["timeseries_hourly"].to_csv(out / "data" / "timeseries.csv")
    results["timeseries_figure"].savefig(out / "figures" / "fig-timeseries.png", dpi=150)
    json.dump(results["summary_stats"], open(out / "metrics.json", "w"), indent=2)
    json.dump(params, open(out / "config.yaml", "w"), indent=2)

    return results["summary_stats"]


if __name__ == "__main__":
    params = {
        "raw_data_path": "rawdata/events.csv",
        "date_column": "opened_at",
        "resample_freq": "1h",
    }
    run(params, "runs/manual_001")
```

---

## 4. marimo Exploration Notebook {#explore-notebook}

The notebook is a thin UI shell. All computation goes through the Hamilton
Driver.

```python
# notebooks/explore.py
import marimo

app = marimo.App(width="medium")


@app.cell
def _():
    import marimo as mo
    from hamilton import driver
    import src.baseline as baseline
    return mo, driver, baseline


@app.cell
def _(mo):
    mo.md("# Interactive Exploration")
    return


@app.cell
def _(mo):
    freq = mo.ui.dropdown(
        options=["10min", "30min", "1h", "4h", "1d"],
        value="1h",
        label="Resample frequency",
    )
    data_path = mo.ui.text(
        value="rawdata/events.csv",
        label="Data path",
    )
    mo.hstack([freq, data_path])
    return freq, data_path


@app.cell
def _(driver, baseline):
    dr = driver.Builder().with_modules(baseline).build()
    return dr,


@app.cell
def _(dr, freq, data_path):
    results = dr.execute(
        ["summary_stats", "timeseries_figure", "timeseries_hourly"],
        inputs={
            "raw_data_path": data_path.value,
            "date_column": "opened_at",
            "resample_freq": freq.value,
        },
    )
    return results,


@app.cell
def _(results, mo):
    stats = results["summary_stats"]
    mo.md(f"**Total periods:** {stats['total_periods']:,} | "
          f"**Mean:** {stats['mean']:.2f} | "
          f"**Zero rate:** {stats['zero_rate']:.1%}")
    return


@app.cell
def _(results):
    results["timeseries_figure"]
    return


@app.cell
def _(results):
    results["timeseries_hourly"].tail(20)
    return


if __name__ == "__main__":
    app.run()
```

Run: `marimo edit notebooks/explore.py`

---

## 5. marimo Report App {#report-app}

Loads pre-computed artifacts from `runs/`. No computation, only display.

```python
# notebooks/report.py
"""Review surface — loads artifacts from runs/, no computation."""
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
        metrics_path = run_dir / "metrics.json"
        if metrics_path.exists():
            metrics = json.loads(metrics_path.read_text())
            mo.tree(metrics)

        figs = sorted((run_dir / "figures").glob("*.png"))
        if figs:
            mo.vstack([mo.image(src=str(f)) for f in figs])
    return


if __name__ == "__main__":
    app.run()
```

Run: `marimo run notebooks/report.py`

---

## 6. Hydra Sweep Runner (Tier 2+) {#sweep-runner}

```python
# scripts/run.py
"""Hydra entry point — composes config, then delegates to Hamilton Driver."""
import hydra
from hydra.core.hydra_config import HydraConfig
from omegaconf import DictConfig, OmegaConf
from hamilton import driver
from pathlib import Path
import json
import src.baseline as baseline


@hydra.main(version_base=None, config_path="../conf", config_name="config")
def main(cfg: DictConfig) -> float:
    out = Path(HydraConfig.get().runtime.output_dir)
    (out / "figures").mkdir(parents=True, exist_ok=True)
    (out / "data").mkdir(parents=True, exist_ok=True)

    # Build Hamilton Driver
    dr = (
        driver.Builder()
        .with_modules(baseline)
        .build()
    )

    # Flatten Hydra config to a dict Hamilton can consume as inputs
    inputs = OmegaConf.to_container(cfg, resolve=True)

    # Execute only the outputs we need
    results = dr.execute(
        ["summary_stats", "timeseries_figure", "timeseries_hourly"],
        inputs=inputs,
    )

    # Save artifacts to run folder
    results["timeseries_hourly"].to_csv(out / "data" / "timeseries.csv")
    results["timeseries_figure"].savefig(
        out / "figures" / "fig-timeseries.png", dpi=150
    )
    json.dump(results["summary_stats"], open(out / "metrics.json", "w"), indent=2)
    OmegaConf.save(cfg, out / "config.yaml")

    return results["summary_stats"].get("total_count", 0)


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

## 7. Comparison Table Builder {#comparison-builder}

```python
# scripts/build_comparison.py
"""Aggregate metrics from all runs into a single comparison table."""
import json
import pandas as pd
from pathlib import Path
from omegaconf import OmegaConf
import sys


def build_comparison(runs_dir: str) -> pd.DataFrame:
    rows = []
    for run_dir in sorted(Path(runs_dir).iterdir()):
        if not run_dir.is_dir():
            continue
        metrics_path = run_dir / "metrics.json"
        if not metrics_path.exists():
            continue

        metrics = json.loads(metrics_path.read_text())
        row = {"run_id": run_dir.name}

        config_path = run_dir / "config.yaml"
        if config_path.exists():
            try:
                cfg = OmegaConf.load(config_path)
                row["resample_freq"] = OmegaConf.select(
                    cfg, "baseline.resample_freq", default=""
                )
            except Exception:
                pass

        row.update(metrics)
        rows.append(row)

    df = pd.DataFrame(rows)
    out_path = Path(runs_dir) / "comparison.csv"
    df.to_csv(out_path, index=False)
    print(f"Comparison table: {out_path} ({len(df)} runs)")
    return df


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python scripts/build_comparison.py <runs_dir>")
        sys.exit(1)
    build_comparison(sys.argv[1])
```

---

## 8. Freshness-Aware Data Loader {#data-loader}

A Hamilton-compatible data loader with freshness tracking. Use this as a node
in your DAG.

```python
# src/data_loader.py
"""Load data with freshness-aware caching."""
import json
from datetime import datetime, timezone, timedelta
from pathlib import Path
import pandas as pd
from typing import Callable


def _fetch_metadata_path(data_path: str) -> str:
    """Convention: metadata file sits next to the data file."""
    return str(Path(data_path).with_suffix(".meta.json"))


def _is_fresh(data_path: str, freshness_hours: int = 48) -> bool:
    """Check if cached data is still fresh enough to reuse."""
    meta_path = _fetch_metadata_path(data_path)
    if not Path(meta_path).exists() or not Path(data_path).exists():
        return False
    meta = json.loads(Path(meta_path).read_text())
    fetched = datetime.fromisoformat(meta["fetched_at"])
    return datetime.now(timezone.utc) - fetched < timedelta(hours=freshness_hours)


def _record_fetch(data_path: str, row_count: int, source: str = "") -> None:
    """Record when data was fetched for freshness tracking."""
    meta = {
        "source": source,
        "fetched_at": datetime.now(timezone.utc).isoformat(),
        "row_count": row_count,
        "data_path": data_path,
    }
    Path(_fetch_metadata_path(data_path)).write_text(json.dumps(meta, indent=2))


def raw_data_cached(
    raw_data_path: str,
    freshness_hours: int,
    fetch_source: str,
) -> pd.DataFrame:
    """Load cached data if fresh, otherwise signal that a fetch is needed.

    In practice, the fetch function is project-specific. Override this node
    or inject the fetch callable through Hamilton inputs.
    """
    if _is_fresh(raw_data_path, freshness_hours):
        return pd.read_csv(raw_data_path)

    raise FileNotFoundError(
        f"Data at {raw_data_path} is stale or missing. "
        f"Run the appropriate fetch tool first: "
        f"python tools/fetch_data.py --output {raw_data_path}"
    )
```

Note: helper functions prefixed with `_` are ignored by Hamilton's graph
discovery. Only `raw_data_cached` becomes a DAG node.

---

## 9. requirements.txt {#requirements}

Baseline for all projects:

```
sf-hamilton[visualization]
marimo
hydra-core
omegaconf
polars
duckdb
pandas
matplotlib
```

Add as needed:

```
# Heavy analytics
stumpy
statsmodels
scikit-learn

# Tier 2+ tracking
sf-hamilton[ui,sdk]

# Tier 3 persistence
dvc[s3]
```
