# Hamilton-Compatible Conventions Reference

## Table of Contents
1. [The Convention (No Runtime Required)](#the-convention)
2. [Example Module](#example-module)
3. [What Makes It Hamilton-Compatible](#what-makes-it-compatible)
4. [Common Patterns](#common-patterns)
5. [When to Add Hamilton Runtime](#when-to-add-runtime)
6. [Driver Patterns](#driver-patterns)

---

## 1. The Convention (No Runtime Required) {#the-convention}

Write Python modules as collections of small, pure-ish functions where:

- **Function name** = the name of the thing it produces.
- **Parameters** = the names of things it depends on.
- **Return value** = the thing it produces.
- **Type hints** = document the contract.

This is valuable even if you never import Hamilton. The convention creates:
- Lower blast radius for AI edits (one function = one concern).
- Natural documentation (read the function signatures to understand the DAG).
- Easy unit testing (pure functions with clear inputs/outputs).
- Painless future migration to Hamilton runtime.

For this workbench, Tier 2 may use either:
- Hamilton-compatible modules with plain imports, or
- Hamilton runtime directly when selective execution or graph display is useful.

Hamilton is not reserved for "big DAG later" work only. It can be a good Tier 2
tool for small and medium analyses when the DAG itself helps the human and AI
reason about the workflow.

---

## 2. Example Module {#example-module}

```python
# modules/baseline.py
"""Time series construction from raw incident data."""
import pandas as pd
import numpy as np
from typing import Dict, Any


def incidents_hourly(
    incidents_raw: pd.DataFrame,
    resample_freq: str,
) -> pd.Series:
    """Resample raw incidents to fixed-frequency counts."""
    ts = (
        incidents_raw
        .set_index("opened_at")
        .resample(resample_freq)
        .size()
    )
    ts.name = "incident_count"
    return ts


def incidents_by_priority(
    incidents_raw: pd.DataFrame,
    resample_freq: str,
) -> pd.DataFrame:
    """Hourly counts broken out by priority level."""
    return (
        incidents_raw
        .set_index("opened_at")
        .groupby("priority")
        .resample(resample_freq)
        .size()
        .unstack(level=0, fill_value=0)
    )


def zero_hour_rate(incidents_hourly: pd.Series) -> float:
    """Fraction of time periods with zero incidents."""
    return float((incidents_hourly == 0).mean())


def baseline_summary(
    incidents_hourly: pd.Series,
    incidents_by_priority: pd.DataFrame,
    zero_hour_rate: float,
    resample_freq: str,
) -> Dict[str, Any]:
    """Machine-readable summary metrics for this stage."""
    return {
        "total_periods": len(incidents_hourly),
        "total_incidents": int(incidents_hourly.sum()),
        "mean_per_period": float(incidents_hourly.mean()),
        "max_per_period": int(incidents_hourly.max()),
        "zero_hour_rate": zero_hour_rate,
        "resample_freq": resample_freq,
        "priority_levels": list(incidents_by_priority.columns),
    }
```

Notice: `baseline_summary` depends on `incidents_hourly`, `zero_hour_rate`, etc.
by parameter name. This IS the dependency graph, expressed in plain Python.

---

## 3. What Makes It Hamilton-Compatible {#what-makes-it-compatible}

| Convention | Why |
|-----------|-----|
| Function name = output name | Hamilton uses function names as node identifiers |
| Parameters = dependency names | Hamilton resolves the DAG from parameter names |
| Type hints on all signatures | Hamilton validates types at graph construction time |
| No side effects in core logic | Enables caching, replay, and parallel execution |
| One module = one logical stage | Hamilton loads modules independently |

**What to avoid:**
- Global mutable state (module-level DataFrames that get modified).
- Functions that read from disk inside the logic (pass data in as parameters).
- Side effects (printing, saving files) in core functions. Put I/O in separate
  "save" or "render" functions at the end of the module.

---

## 4. Common Patterns {#common-patterns}

### Config as parameters
```python
def matrix_profile(
    incidents_hourly: pd.Series,
    window_size: int,
    normalize: bool,
) -> np.ndarray:
    """Compute matrix profile using stumpy."""
    import stumpy
    return stumpy.stump(incidents_hourly.values, m=window_size)[:, 0]
```

Parameters like `window_size` and `normalize` come from config (Hydra) or
UI elements (marimo). The function doesn't care where they come from.

### Separating I/O from logic
```python
# Core logic — pure, testable
def discords(
    matrix_profile: np.ndarray,
    anomaly_threshold: float,
    top_k: int,
) -> pd.DataFrame:
    """Find top-K discords (anomalous subsequences)."""
    scores = matrix_profile.copy()
    results = []
    for _ in range(top_k):
        idx = np.argmax(scores)
        if scores[idx] < anomaly_threshold:
            break
        results.append({"index": int(idx), "score": float(scores[idx])})
        # Exclude neighbors
        scores[max(0, idx-10):idx+10] = -np.inf
    return pd.DataFrame(results)


# I/O — separate function, called explicitly
def save_discords(discords: pd.DataFrame, output_dir: str) -> str:
    """Save discords to CSV. Returns the file path."""
    path = f"{output_dir}/discords.csv"
    discords.to_csv(path, index=False)
    return path
```

### Multiple outputs from one computation
```python
from typing import Tuple

def decompose_timeseries(
    incidents_hourly: pd.Series,
) -> Tuple[pd.Series, pd.Series, pd.Series]:
    """STL decomposition into trend, seasonal, residual."""
    from statsmodels.tsa.seasonal import STL
    result = STL(incidents_hourly.dropna(), period=24).fit()
    return result.trend, result.seasonal, result.resid
```

Or use Hamilton's `@extract_fields` decorator when you add the runtime.

---

## 5. When to Add Hamilton Runtime {#when-to-add-runtime}

**Stay with plain imports when:**
- < 20 functions across all modules.
- The dependency chain is obvious from reading the code.
- You're still in exploratory mode and the analysis shape is changing fast.

**Add Hamilton driver when:**
- You want to request specific outputs and have only their dependencies execute.
- Graph visualization in the workbench would help you or the human understand the workflow.
- The same analysis logic should support both quick exploration and Hydra sweeps.
- Node-level caching (`@cache`) would save meaningful compute time.
- The dependency graph is starting to matter more than linear script order.

This can happen well before 20 functions. For this reason, Hamilton runtime is
a valid Tier 2 choice, not only a late-stage upgrade.

---

## 6. Driver Patterns {#driver-patterns}

```python
from hamilton import driver
import modules.baseline as baseline
import modules.matrix_profile as mp

# Build the driver from multiple modules
dr = (
    driver.Builder()
    .with_modules(baseline, mp)
    .with_cache()  # enables @cache decorators
    .build()
)

# Execute — only computes what's needed for requested outputs
results = dr.execute(
    ["discords", "baseline_summary"],
    inputs={
        "incidents_raw": df,
        "resample_freq": "1h",
        "window_size": 24,
        "normalize": True,
        "anomaly_threshold": 3.0,
        "top_k": 10,
    },
)

# results is a dict: {"discords": DataFrame, "baseline_summary": dict}
```

```python
# Visualize the DAG
graph = dr.display_all_functions()
```

In a notebook surface such as marimo, return the `graph` object from the cell
or render it to SVG/PNG for inline display.

The key insight: because your modules already follow Hamilton conventions,
switching from `from modules.baseline import incidents_hourly; incidents_hourly(df, "1h")`
to `dr.execute(["incidents_hourly"], inputs={...})` requires zero changes to
the module code.
