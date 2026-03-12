# Hamilton Conventions Reference

Hamilton is the **computation layer** of the analytic workbench. It owns the
DAG of transformations. All business logic lives in `src/` modules as Hamilton
functions. The Driver executes them.

## Table of Contents
1. [Mental Model](#mental-model)
2. [Install and Extras](#install-and-extras)
3. [Canonical Pattern](#canonical-pattern)
4. [Driver and Builder](#driver-and-builder)
5. [How Hamilton Builds the DAG](#how-hamilton-builds-the-dag)
6. [Function Modifiers That Matter](#function-modifiers-that-matter)
7. [Materialization and Adapters](#materialization-and-adapters)
8. [Caching and Parallelism](#caching-and-parallelism)
9. [Visualization and Debugging](#visualization-and-debugging)
10. [Project Conventions for This Workbench](#project-conventions)
11. [Common Mistakes](#common-mistakes)
12. [When to Read Primary Docs](#when-to-read-primary-docs)

---

## 1. Mental Model {#mental-model}

Hamilton's authoring model is intentionally small:

- function name = node name
- parameter names = dependency names
- type hints = interface contract
- helper functions prefixed with `_` are ignored by graph discovery

Example:

```python
def spend_mean(spend: pd.Series) -> float:
    return float(spend.mean())
```

This does **not** call `spend`. It declares that `spend_mean` depends on a node
or runtime input named `spend`.

Important consequences:

- node names are the public API of the DAG
- small output-oriented functions work better than giant stage functions
- missing upstreams are fine if they will be provided through `inputs=...`
- many decorators reshape graph construction, not just runtime behavior

Think in two layers:

- **definition layer**: plain Python functions in `src/` modules
- **runtime layer**: `Driver`, `Builder`, executors, adapters, materializers, cache

That separation is why Hamilton remains readable.

---

## 2. Install and Extras {#install-and-extras}

The package name is `sf-hamilton`.

```bash
pip install sf-hamilton --break-system-packages
pip install "sf-hamilton[visualization]" --break-system-packages
pip install "sf-hamilton[ui,sdk]" --break-system-packages
```

Notes:

- visualization also needs Graphviz installed on the machine
- validation/UI/integration features may require extras such as Pandera, Pydantic, Ray, or Dask
- install details and supported extras are version-sensitive; re-check official docs when wiring a new integration

---

## 3. Canonical Pattern {#canonical-pattern}

### Transformation module

```python
# src/features.py
import pandas as pd


def avg_3wk_spend(spend: pd.Series) -> pd.Series:
    return spend.rolling(3).mean()


def acquisition_cost(avg_3wk_spend: pd.Series, signups: pd.Series) -> pd.Series:
    return avg_3wk_spend / signups


def spend_mean(spend: pd.Series) -> float:
    return float(spend.mean())


def spend_zero_mean(spend: pd.Series, spend_mean: float) -> pd.Series:
    return spend - spend_mean
```

### Runtime entry point

```python
import pandas as pd
from hamilton import base, driver
import src.features as features

inputs = {
    "spend": pd.Series([10, 10, 20, 40, 40, 50]),
    "signups": pd.Series([1, 10, 50, 100, 200, 400]),
}

dr = (
    driver.Builder()
    .with_modules(features)
    .with_adapters(base.PandasDataFrameResult())
    .build()
)

result = dr.execute(
    ["avg_3wk_spend", "acquisition_cost", "spend_zero_mean"],
    inputs=inputs,
)
```

What this shows:

- transformation code stays in `src/` modules
- runtime inputs satisfy missing upstreams
- `Builder()` assembles runtime behavior
- `execute()` computes only the subgraph needed for requested outputs

---

## 4. Driver and Builder {#driver-and-builder}

### `Driver`

`Driver` is the runtime object you execute. It is responsible for:

- building the graph
- validating it
- executing the requested slice
- returning results
- exposing visualization and materialization operations

Important methods: `execute(...)`, `materialize(...)`, and graph inspection helpers.

### `Builder`

`Builder` is the fluent runtime assembly surface. Use it to keep runtime choices
out of DAG modules.

Key methods:

- `with_modules(...)` — which `src/` modules to include
- `with_config(...)` — for `@config.when` node selection only
- `with_adapters(...)` — result builders, trackers, hooks
- `with_materializers(...)` — static I/O edges
- `with_cache(...)` — node-level caching
- `enable_dynamic_execution(...)` — for Parallelizable/Collect

Common patterns:

```python
# Basic
dr = driver.Builder().with_modules(baseline, features).build()

# With config-based node selection
dr = (
    driver.Builder()
    .with_modules(baseline, models)
    .with_config({"model_type": "xgboost"})
    .build()
)

# With caching and tracking
dr = (
    driver.Builder()
    .with_modules(pipeline)
    .with_cache()
    .with_adapters(tracker)
    .build()
)
```

**Critical distinction for this workbench:**

- Hydra config → `inputs=OmegaConf.to_container(cfg)` — parameter values
- Hamilton `with_config()` — node selection via `@config.when`

Do not conflate these two.

---

## 5. How Hamilton Builds the DAG {#how-hamilton-builds-the-dag}

At a high level Hamilton:

1. scans the provided modules for Hamilton functions
2. creates nodes from those functions
3. resolves decorators / function modifiers
4. connects dependencies by parameter name
5. validates graph structure and types
6. executes only the subgraph needed for requested outputs

This explains three important properties:

- you can adopt Hamilton incrementally
- the framework computes only what is required
- decorators matter because they can alter graph shape before execution

---

## 6. Function Modifiers That Matter {#function-modifiers-that-matter}

Function modifiers are central to Hamilton. They can change inclusion, expand
nodes, split outputs, inject I/O, add validation, or attach metadata.

### `@tag`

Use for metadata such as ownership, domain, data product, or policy.

```python
from hamilton.function_modifiers import tag

@tag(owner="growth", data_product="marketing", pii="false")
def weekly_revenue(clean_df: pd.DataFrame) -> pd.Series:
    ...
```

### `@check_output`

Use for validation without mixing checks into business logic.

```python
from hamilton.function_modifiers import check_output

@check_output(range=(0.0, 1.0), importance="fail")
def churn_probability(raw_score: pd.Series) -> pd.Series:
    return raw_score.clip(0.0, 1.0)
```

### `@extract_columns` / `@extract_fields`

Use when a composite output should expose downstream-addressable pieces.

```python
from hamilton.function_modifiers import extract_columns

@extract_columns("user_id", "country", "revenue")
def clean_df(raw_df: pd.DataFrame) -> pd.DataFrame:
    return raw_df.dropna()
```

### `@parameterize`

Use when one implementation should generate many named nodes.

```python
from hamilton.function_modifiers import parameterize, source, value

@parameterize(
    revenue_by_country=dict(df=source("orders"), groupby_col=value("country")),
    revenue_by_segment=dict(df=source("orders"), groupby_col=value("segment")),
)
def revenue_by_dimension(df: pd.DataFrame, groupby_col: str) -> pd.Series:
    return df.groupby(groupby_col)["revenue"].sum()
```

### `@config.when(...)` family

Use config-based inclusion when different implementations should resolve to the
same logical node name.

```python
from hamilton.function_modifiers import config

@config.when(model_type="xgboost")
def base_model__xgboost() -> object:
    ...

@config.when_not(model_type="xgboost")
def base_model__default() -> object:
    ...
```

### `@load_from` / `@save_to`

Use when storage behavior should be explicit in the DAG.

```python
from hamilton.function_modifiers import load_from, save_to, source

@load_from.csv(path=source("raw_input_path"))
def cleaned_data(raw_data: pd.DataFrame) -> pd.DataFrame:
    return raw_data.dropna()
```

Practical interpretation:

- inclusion decorators decide which nodes exist
- parameterization expands one function into many nodes
- extraction exposes column/field lineage
- I/O decorators inject storage edges
- validation decorators attach checks after computation
- metadata decorators help governance and filtering

---

## 7. Materialization and Adapters {#materialization-and-adapters}

### Materialization

Two styles:

- static materializers attached at build time via `with_materializers(...)`
- dynamic materializers passed to `Driver.materialize(...)`

Use runtime inputs for early prototyping, static materializers when storage is
part of the graph, dynamic materialization when deployment code chooses
sources/sinks at runtime.

### Adapters and result builders

Adapters are the main extension point for runtime concerns: result shaping,
tracking, progress, hooks, integrations.

Common result builder: `base.PandasDataFrameResult()` to collect requested
outputs into a DataFrame.

Practical rule: keep core transformations independent, attach runtime behavior
with adapters rather than embedding it in node code.

---

## 8. Caching and Parallelism {#caching-and-parallelism}

### Caching

Hamilton supports builder-level and node-level caching.

```python
dr = (
    driver.Builder()
    .with_modules(my_dataflow)
    .with_cache(recompute=["raw_data"])
    .build()
)
```

Important caveat: cache invalidation does not automatically capture
helper-function changes or dependency library changes.

### Parallelism

Two models:

**Adapter-based execution** — uses a threadpool, Dask, Ray, or async executor
without changing graph structure.

```python
from hamilton.plugins.h_threadpool import FutureAdapter

dr = (
    driver.Builder()
    .with_modules(foo_module)
    .with_adapters(FutureAdapter())
    .build()
)
```

**Dynamic task-based execution** — uses `Parallelizable[]` and `Collect[]`
for fan-out/fan-in patterns.

```python
from hamilton.htypes import Collect, Parallelizable

def urls() -> Parallelizable[str]:
    for url in ["https://a", "https://b", "https://c"]:
        yield url

def page_text(urls: str) -> str:
    return fetch(urls)

def total_words(page_text: Collect[int]) -> int:
    return sum(page_text)
```

Use parallelism only for concrete performance needs, not as a default.

---

## 9. Visualization and Debugging {#visualization-and-debugging}

Visualization is a normal development tool in Hamilton, not just presentation.
Use it to verify module composition, inspect execution slices, debug unexpected
dependencies, and communicate lineage to humans.

```python
# In a marimo notebook or script
dr.display_all_functions()  # show full graph
dr.visualize_execution(["output_name"], inputs=inputs)  # show execution slice
```

Hamilton UI for operational visibility:

```bash
pip install "sf-hamilton[ui,sdk]" --break-system-packages
hamilton ui
```

---

## 10. Project Conventions for This Workbench {#project-conventions}

All Hamilton modules live in `src/`. Layout:

```
src/
  __init__.py
  baseline.py          # time series construction
  features.py          # feature engineering
  models.py            # model fitting (if applicable)
  evaluation.py        # metrics and scoring
  data_loader.py       # freshness-aware data loading
```

Conventions:

- keep functions small and output-oriented
- keep runtime-specific concerns out of `src/` modules
- group transformations by domain or stage
- use `with_config()` and `@config.when` for implementation variability
- helper functions prefixed with `_` stay out of the DAG
- data loading is a DAG node (e.g., `raw_data(raw_data_path: str) -> pd.DataFrame`)
- figures are returned as objects, never saved inside a node

Testing layers:

- unit tests for plain transformation functions
- DAG-level tests for a small driver plus toy inputs
- validation / contract tests using `@check_output`

---

## 11. Common Mistakes {#common-mistakes}

- writing huge nodes and losing lineage value
- mixing I/O into business logic too early
- ignoring node naming discipline
- assuming cache invalidation notices every code change
- treating Hamilton like a scheduler or full platform
- forgetting that runtime inputs can satisfy "missing" dependencies
- forgetting that modifiers reshape the graph before execution
- confusing Hydra config (parameter values) with Hamilton `with_config()` (node selection)
- putting transform logic in marimo cells instead of `src/` modules

---

## 12. When to Read Primary Docs {#when-to-read-primary-docs}

Go to current docs/source when you need:

- exact extras, install, or integration requirements
- backend-specific materializer syntax
- current caching implementation details
- current dynamic execution limitations
- exact UI/CLI commands
- framework-specific integration setup
