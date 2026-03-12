# Hydra Config Reference

Hydra is the **config layer** of the analytic workbench. It owns config
composition, CLI overrides, and experiment sweeps. It produces a frozen
`DictConfig` that Hamilton consumes as `inputs`. Hydra never touches DataFrames,
figures, or business logic.

**Boundary with Hamilton:** Hydra's `DictConfig` becomes Hamilton's `inputs`
dict via `OmegaConf.to_container(cfg, resolve=True)`. Hamilton's own
`with_config()` is for node selection (`@config.when`), not for passing
parameter values. Keep these two config mechanisms separate.

## Table of Contents
1. [Basic Structure](#basic-structure)
2. [Config Composition](#config-composition)
3. [Experiment Configs](#experiment-configs)
4. [Command-Line Overrides](#overrides)
5. [Multirun Sweeps](#multirun)
6. [Output Directory Conventions](#output-dirs)
7. [Integration with Hamilton](#integration)

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
  - source: csv_local
  - _self_

baseline:
  resample_freq: "1h"
  date_column: opened_at

analysis:
  window_size: 24
  normalize: true
  anomaly_threshold: 3.0
  top_k_discords: 10

source:
  raw_data_path: rawdata/incidents.csv
```

Note: `defaults:` is composition metadata, not business data you read from
`cfg`. `_self_` means "merge the body of the current file here." Order matters:
later merges win on conflicts.

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
  raw_data_path: rawdata/incidents.csv
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
  resample_freq: "4h"

analysis:
  window_size: 24
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
  window_size: 168
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
    dir: runs/sweeps/${now:%Y-%m-%d_%H-%M-%S}
    subdir: ${hydra.job.override_dirname}
```

Produces:

```
runs/sweeps/2026-03-08_14-30-00/
  baseline.resample_freq=30min,analysis.anomaly_threshold=2.0/
    config.yaml
    metrics.json
    figures/
  baseline.resample_freq=30min,analysis.anomaly_threshold=3.0/
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
- With Hydra 1.2+ and `version_base=None`, `hydra.job.chdir` defaults to `False`
- Do not assume `Path(".")` is the run directory unless you explicitly enable
  `hydra.job.chdir=True`
- Prefer `HydraConfig.get().runtime.output_dir` when writing run artifacts

---

## 7. Integration with Hamilton {#integration}

Hydra is the config layer. Hamilton is the computation layer. The runner
script bridges them:

```python
# scripts/run.py
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

    # Hydra config → Hamilton inputs
    inputs = OmegaConf.to_container(cfg, resolve=True)

    # Build Hamilton Driver from src/ modules
    dr = (
        driver.Builder()
        .with_modules(baseline)
        .build()
    )

    # Execute only the outputs we need
    results = dr.execute(
        ["summary_stats", "timeseries_figure", "timeseries_hourly"],
        inputs=inputs,
    )

    # Save artifacts
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

The key pattern:

1. Hydra composes config from YAML + CLI overrides
2. `OmegaConf.to_container(cfg, resolve=True)` flattens it to a plain dict
3. Hamilton's `Driver.execute(outputs, inputs=flat_dict)` runs only the needed
   subgraph
4. Artifacts are saved to the Hydra output directory

**When Hamilton's `with_config()` is needed:** Only for `@config.when` node
selection — e.g., choosing between different model implementations based on a
config flag. This is a different mechanism from passing parameter values as
inputs.

```python
# Only when you need @config.when node selection
dr = (
    driver.Builder()
    .with_modules(baseline, models)
    .with_config({"model_type": cfg.get("model_type", "default")})
    .build()
)
```
