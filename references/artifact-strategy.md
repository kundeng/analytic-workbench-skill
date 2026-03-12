# Artifact Strategy Reference

## Table of Contents
1. [Per-Run Artifact Folders](#per-run-folders)
2. [Comparison Table Schema](#comparison-table)
3. [Freshness & Caching Rules](#freshness-rules)
4. [DVC Integration Patterns](#dvc-integration)
5. [AI Self-Review Checklist](#ai-review)

---

## 1. Per-Run Artifact Folders {#per-run-folders}

Every experiment run produces a self-contained folder under `runs/`:

```
runs/<run-id>/
  config.yaml           # frozen config — exactly what params produced this run
  metrics.json          # machine-readable summary metrics
  figures/
    fig-timeseries.png
    fig-anomalies.png
    fig-precision-recall.png
  data/
    timeseries.csv
    discords.csv
    motifs.csv
  logs/
    run.log             # optional — stdout/stderr capture
```

**Run ID conventions:**

- Timestamp: `2026-03-08_14-22-33` (default from Hydra)
- Override-based: `resample_freq=1h,window_size=24` (Hydra sweep subdirs)
- Named: `experiment_fast_test_001` (manual runs)

**Non-negotiable files:**

- `config.yaml` — without this, the run is not reproducible.
- `metrics.json` — without this, the run cannot be compared.

**Key layout rules:**

- All run artifacts live inside `runs/<run-id>/`. No separate `outputs/figures/`
  or `data/processed/` directories at the project root.
- `rawdata/` holds immutable source data. It is never written to by a run.
- Each run folder is self-contained and independently meaningful.

---

## 2. Comparison Table Schema {#comparison-table}

The comparison table (`runs/comparison.csv`) is a first-class artifact.
One row per run. Columns:

| Column | Source | Purpose |
|--------|--------|---------|
| `run_id` | folder name | Unique identifier |
| `timestamp` | config or folder | When the run happened |
| Key parameters | `config.yaml` | What settings were used |
| Key metrics | `metrics.json` | What the run produced |

Example:

```csv
run_id,resample_freq,window_size,threshold,total_incidents,discords_found,precision,recall,f1
2026-03-08_14-22-33,1h,24,3.0,12450,8,0.62,0.45,0.52
2026-03-08_14-25-01,1h,72,3.0,12450,5,0.80,0.30,0.44
2026-03-08_14-28-44,1h,168,3.0,12450,3,1.00,0.15,0.26
2026-03-08_15-01-12,30min,24,2.5,12450,14,0.43,0.70,0.53
```

This table answers:

- Under which settings does the method perform best?
- What is the tradeoff between sensitivity and false positives?
- Which parameter has the most impact on results?

**Build it automatically** after every sweep with `scripts/build_comparison.py`.

---

## 3. Freshness & Caching Rules {#freshness-rules}

Some data is expensive to fetch. The workflow needs explicit rules about when
to reuse cached data vs. re-fetch.

**Example rule:** "Do not pull SNOW incident data from Splunk again within 48
hours because results won't materially change."

### Implementation

Track freshness with a metadata file next to the data:

```json
// rawdata/incidents.meta.json
{
  "source": "splunk",
  "query": "index=* sourcetype=snow:incident",
  "fetched_at": "2026-03-08T14:00:00Z",
  "freshness_hours": 48,
  "row_count": 12450
}
```

The `src/data_loader.py` module (see `code-templates.md`) uses `_is_fresh()`
helper functions prefixed with `_` so they stay out of the Hamilton DAG while
the public `raw_data_cached` node participates in the graph.

### DVC freshness (Tier 3)

DVC handles this natively with frozen stages:

```yaml
stages:
  extract:
    frozen: true
    cmd: python tools/fetch_data.py --output rawdata/incidents.csv
    outs:
      - rawdata/incidents.csv
```

Unfreeze when you want fresh data: `dvc repro --force extract`

---

## 4. DVC Integration Patterns {#dvc-integration}

When to add DVC (Tier 3):

### Stage caching

```yaml
# dvc.yaml
stages:
  baseline:
    cmd: >-
      python scripts/run.py
        baseline.resample_freq=${baseline.resample_freq}
    deps:
      - rawdata/incidents.csv
      - src/baseline.py
    params:
      - baseline.resample_freq
    outs:
      - runs/baseline/
```

DVC skips the stage if inputs, code, and params haven't changed.

### When DVC and Hydra overlap

Both DVC and Hydra can manage parameters and experiment runs. Use them for
different things:

| Concern | Use |
|---------|-----|
| Ad hoc parameter overrides | Hydra CLI overrides |
| Systematic sweeps with structured output | Hydra `--multirun` |
| Persistent caching of expensive stages | DVC stages |
| Long-term experiment comparison | DVC experiments |
| Source data freshness rules | DVC frozen stages |

They coexist well. Hydra manages the config; DVC manages the artifacts.

---

## 5. AI Self-Review Checklist {#ai-review}

After every run, the AI should programmatically check:

### Data validation

- Row counts are plausible (not 0, not absurdly large)
- No unexpected NaN or Inf values in key columns
- Date ranges cover the expected window
- Categorical values match expected sets

### Metrics validation

- All metrics in `metrics.json` are finite numbers
- Precision, recall, F1 are in [0, 1]
- Counts are non-negative integers
- Results change meaningfully across parameter settings (not all identical)

### Figure validation (via vision)

- Figures are non-trivial (not blank, not all-zero)
- Axes are labeled and readable
- Time series cover the expected date range
- Anomaly markers (if any) are visible

### Cross-run validation

- Comparison table has one row per run
- No duplicate run IDs
- Metrics vary across runs (if all identical, something is wrong)
- Best config is identified and noted

**Report findings to the human** along with the comparison table and key figures.
The human reviews the same artifacts in the marimo report app.
