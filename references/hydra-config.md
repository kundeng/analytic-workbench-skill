# Hydra Config Patterns Reference

## Table of Contents
1. [Basic Structure](#basic-structure)
2. [Config Composition](#config-composition)
3. [Experiment Configs](#experiment-configs)
4. [Command-Line Overrides](#overrides)
5. [Multirun Sweeps](#multirun)
6. [Output Directory Conventions](#output-dirs)
7. [Integration with the Workbench](#integration)

---

## 1. Basic Structure {#basic-structure}

```
conf/
  config.yaml              # primary config
  experiment/
    fast_test.yaml          # quick single-param test
    full_sweep.yaml         # comprehensive sweep
  source/
    splunk.yaml             # data source variant
    csv_local.yaml          # local CSV variant
```

```yaml
# conf/config.yaml
defaults:
  - _self_
  - source: csv_local       # default data source

baseline:
  resample_freq: "1h"

analysis:
  window_sizes: [24, 72, 168]
  normalize: true
  anomaly_threshold: 3.0
  top_k_discords: 10

output:
  dir: outputs/runs/${now:%Y-%m-%d_%H-%M-%S}
```

---

## 2. Config Composition {#config-composition}

Hydra composes configs by merging YAML files from the `defaults` list.
Each config group (e.g., `source/`) provides variants:

```yaml
# conf/source/splunk.yaml
source:
  type: splunk
  index: snow
  sourcetype: "snow:incident"
  earliest: "-12mon"
  latest: "now"

# conf/source/csv_local.yaml
source:
  type: csv
  path: data/raw/incidents.csv
  date_column: opened_at
```

Switch source at the command line:
```bash
python scripts/run.py source=splunk
```

Key composition rules:

- `defaults:` is composition metadata, not business data you usually read from `cfg`
- `_self_` means "merge the body of the current file here"
- order matters: later merges win on conflicts
- group selections like `source: splunk` usually land under `cfg.source`
- `# @package _global_` changes placement so the file merges at the config root

Minimal precedence example:

```yaml
defaults:
  - source: csv_local
  - _self_

source:
  timeout: 30
```

In this case, the current file merges after `source: csv_local`, so `source.timeout: 30`
from the current file overrides the same field from `csv_local.yaml`.

---

## 3. Experiment Configs {#experiment-configs}

Experiment configs override multiple settings at once for named experiments:

```yaml
# conf/experiment/fast_test.yaml
# @package _global_
defaults:
  - override /source: csv_local

baseline:
  resample_freq: "4h"       # coarser for speed

analysis:
  window_sizes: [24]         # single window
  top_k_discords: 3
```

```yaml
# conf/experiment/full_sweep.yaml
# @package _global_
defaults:
  - override /source: splunk

baseline:
  resample_freq: "1h"

analysis:
  window_sizes: [24, 48, 72, 168, 336]
  top_k_discords: 20
```

Run a named experiment:
```bash
python scripts/run.py +experiment=fast_test
python scripts/run.py +experiment=full_sweep
```

---

## 4. Command-Line Overrides {#overrides}

```bash
# Override a single value
python scripts/run.py baseline.resample_freq=30min

# Override a list
python scripts/run.py analysis.window_sizes="[24,168]"

# Override nested values
python scripts/run.py source.earliest="-6mon"

# Combine overrides
python scripts/run.py baseline.resample_freq=30min analysis.anomaly_threshold=2.5
```

Operator semantics:

- `key=value`: set an existing key; usually errors if the path does not exist
- `+key=value`: add a new key only if it is currently absent; errors if already present
- `++key=value`: create or replace; use carefully because it can hide typos
- `~key`: delete a key or remove a defaults-list selection

Two different kinds of override:

- `source=splunk` selects a config-group option, meaning Hydra loads a different config file
- `source.timeout=60` changes a field inside the already selected `source` subtree

Practical rule:

- use plain `key=value` by default
- use `+` when "must be new" is the point
- use `++` only when you explicitly want create-or-replace behavior

---

## 5. Multirun Sweeps {#multirun}

Sweep across parameter values with `--multirun` (or `-m`):

```bash
# Sweep one parameter
python scripts/run.py -m baseline.resample_freq=10min,30min,1h,4h

# Sweep two parameters (cartesian product)
python scripts/run.py -m \
  baseline.resample_freq=30min,1h \
  analysis.anomaly_threshold=2.0,3.0,5.0

# 2 x 3 = 6 runs, each in its own output directory
```

### Sweep output structure

```yaml
# Configure in config.yaml
hydra:
  sweep:
    dir: outputs/sweeps/${now:%Y-%m-%d_%H-%M-%S}
    subdir: ${hydra.job.override_dirname}
```

Produces:
```
outputs/sweeps/2026-03-08_14-30-00/
  baseline.resample_freq=30min,analysis.anomaly_threshold=2.0/
    config.yaml
    metrics.json
    figures/
  baseline.resample_freq=30min,analysis.anomaly_threshold=3.0/
    ...
  baseline.resample_freq=1h,analysis.anomaly_threshold=2.0/
    ...
```

---

## 6. Output Directory Conventions {#output-dirs}

Every run should save:

| File | Purpose |
|------|---------|
| `config.yaml` | Frozen config (Hydra saves this automatically) |
| `metrics.json` | Machine-readable metrics for comparison table |
| `figures/*.png` | Visual artifacts |
| `data/*.csv` | Data outputs (optional) |

The comparison table builder reads `metrics.json` from every run directory
to build the cross-run comparison table.

Important runtime detail:

- Hydra always creates a run directory and writes `.hydra/` metadata there
- with Hydra 1.2+ and `version_base=None`, `hydra.job.chdir` defaults to `False`
- do not assume `Path(".")` is the run directory unless you explicitly enable `hydra.job.chdir=True`
- prefer `HydraConfig.get().runtime.output_dir` when writing run artifacts

---

## 7. Integration with the Workbench {#integration}

Hydra is the default experiment control layer for Tier 2 in this workbench.
It handles:
- config composition
- CLI overrides
- sweep directory organization
- frozen per-run configs

Combine Hydra with either:
- plain Hamilton-compatible modules, or
- Hamilton runtime when you want the internal DAG to be executable/visualizable

### The sweep runner (`scripts/run.py`)

```python
# scripts/run.py
import hydra
from hydra.core.hydra_config import HydraConfig
from omegaconf import DictConfig, OmegaConf
import json
from pathlib import Path

@hydra.main(version_base=None, config_path="../conf", config_name="config")
def main(cfg: DictConfig) -> None:
    out = Path(HydraConfig.get().runtime.output_dir)
    (out / "figures").mkdir(parents=True, exist_ok=True)
    (out / "data").mkdir(parents=True, exist_ok=True)

    # Import and call Hamilton-compatible modules directly
    from modules.baseline import incidents_hourly, baseline_summary
    import pandas as pd

    # Load data based on config
    if cfg.source.type == "csv":
        df = pd.read_csv(cfg.source.path, parse_dates=[cfg.source.date_column])
    else:
        # Call your data extraction tool
        from tools.splunk_search import extract_incidents
        df = extract_incidents(cfg.source)

    # Run analysis
    ts = incidents_hourly(df, resample_freq=cfg.baseline.resample_freq)
    summary = baseline_summary(ts, ...)

    # Save artifacts
    json.dump(summary, open(out / "metrics.json", "w"), indent=2)
    ts.to_csv(out / "data" / "timeseries.csv")

    # Save frozen config (Hydra does this, but explicit is good)
    OmegaConf.save(cfg, out / "config.yaml")

    return summary.get("total_incidents", 0)  # returned as Hydra job result


if __name__ == "__main__":
    main()
```

### Hamilton runtime variant

```python
import hydra
from omegaconf import DictConfig
from hamilton import driver
import modules.baseline as baseline
import modules.analysis as analysis

@hydra.main(version_base=None, config_path="../conf", config_name="config")
def main(cfg: DictConfig) -> None:
    dr = driver.Builder().with_modules(baseline, analysis).build()
    results = dr.execute(
        ["baseline_summary", "discords"],
        inputs={
            "incidents_raw": load_incidents_from_cfg(cfg),
            "resample_freq": cfg.baseline.resample_freq,
            "window_size": cfg.analysis.window_size,
            "normalize": cfg.analysis.normalize,
            "anomaly_threshold": cfg.analysis.anomaly_threshold,
            "top_k": cfg.analysis.top_k_discords,
        },
    )
```

Hydra still owns the run configuration and output directories. Hamilton owns the
internal dataflow for requested outputs.

### Building the comparison table

```python
# scripts/build_comparison.py
"""Read metrics.json from every run directory, build comparison.csv."""
import json
import pandas as pd
from pathlib import Path
from omegaconf import OmegaConf

def build_comparison(sweep_dir: str) -> pd.DataFrame:
    rows = []
    for run_dir in sorted(Path(sweep_dir).iterdir()):
        metrics_path = run_dir / "metrics.json"
        config_path = run_dir / "config.yaml"
        if not metrics_path.exists():
            continue

        metrics = json.loads(metrics_path.read_text())
        cfg = OmegaConf.load(config_path) if config_path.exists() else {}

        row = {
            "run_id": run_dir.name,
            # Pull key params from config
            "resample_freq": OmegaConf.select(cfg, "baseline.resample_freq", default="?"),
            "window_size": OmegaConf.select(cfg, "analysis.window_sizes", default="?"),
            "threshold": OmegaConf.select(cfg, "analysis.anomaly_threshold", default="?"),
        }
        row.update(metrics)  # merge all metrics
        rows.append(row)

    df = pd.DataFrame(rows)
    df.to_csv(Path(sweep_dir) / "comparison.csv", index=False)
    return df
```

This produces the first-class comparison artifact the skill requires — one row
per run, all key metrics and parameters in columns, ready for human review in
a marimo report app.
