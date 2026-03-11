# Hamilton Conventions Reference

## Table of Contents
1. [What It Is](#what-it-is)
2. [Mental Model](#mental-model)
3. [Install And Extras](#install-and-extras)
4. [Canonical Pattern](#canonical-pattern)
5. [Driver And Builder](#driver-and-builder)
6. [How Hamilton Builds The DAG](#how-hamilton-builds-the-dag)
7. [Function Modifiers That Matter](#function-modifiers-that-matter)
8. [Materialization And Adapters](#materialization-and-adapters)
9. [Caching And Parallelism](#caching-and-parallelism)
10. [Visualization, UI, And CLI](#visualization-ui-and-cli)
11. [Project Structure And Testing](#project-structure-and-testing)
12. [Decision Rules](#decision-rules)
13. [Common Mistakes](#common-mistakes)
14. [When To Read Primary Docs](#when-to-read-primary-docs)

---

## 1. What It Is {#what-it-is}

Hamilton is a Python dataflow framework where:

- a function becomes a node
- parameter names define dependencies
- type hints define expected contracts
- the runtime builds and executes the DAG automatically

It is best understood as the transformation/computation layer inside a larger
system. Keep scheduling, infrastructure orchestration, and deployment concerns
outside Hamilton.

Good fit:

- analytics, data, ML, and LLM workflows with meaningful intermediate artifacts
- codebases that need lineage, selective execution, validation, and portability
- teams willing to use naming discipline and many small functions

Bad fit:

- workflows dominated by opaque large steps
- teams mainly looking for scheduling or cluster management
- low-code / visual-only pipeline authoring preferences

---

## 2. Mental Model {#mental-model}

Hamilton's authoring model is intentionally small:

- function name = node name
- parameter names = dependency names
- type hints = interface contract
- helper functions prefixed with `_` are ignored by graph discovery

Example:

```python
def spend_mean(spend):
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

- **definition layer**: plain Python functions in modules
- **runtime layer**: `Driver`, `Builder`, executors, adapters, materializers, cache

That separation is why Hamilton remains readable.

---

## 3. Install And Extras {#install-and-extras}

The package name is `sf-hamilton`.

Common installs:

```bash
pip install sf-hamilton
pip install "sf-hamilton[visualization]"
pip install "sf-hamilton[ui,sdk]"
```

Notes:

- visualization also needs Graphviz installed on the machine
- validation/UI/integration features may require extras such as Pandera, Pydantic, Ray, or Dask
- install details and supported extras are version-sensitive; re-check official docs when wiring a new integration

---

## 4. Canonical Pattern {#canonical-pattern}

### Transformation module

```python
# features.py
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
import features

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

- transformation code stays in plain modules
- runtime inputs satisfy missing upstreams
- `Builder()` assembles runtime behavior
- `execute()` computes only the subgraph needed for requested outputs

---

## 5. Driver And Builder {#driver-and-builder}

### `Driver`

`Driver` is the runtime object you execute. It is responsible for:

- building the graph
- validating it
- executing the requested slice
- returning results
- exposing visualization and materialization operations

Important methods to remember:

- `execute(...)`
- `materialize(...)`
- visualization / graph inspection helpers

### `Builder`

`Builder` is the fluent runtime assembly surface. Use it to keep runtime choices
out of DAG modules.

Key methods:

- `with_modules(...)`
- `with_config(...)`
- `with_adapters(...)`
- `with_materializers(...)`
- `with_cache(...)`
- `enable_dynamic_execution(...)`

Common patterns:

```python
dr = driver.Builder().with_modules(features, training).build()
```

```python
dr = (
    driver.Builder()
    .with_modules(training)
    .with_config({"model_type": "xgboost", "environment": "prod"})
    .build()
)
```

```python
dr = (
    driver.Builder()
    .with_modules(pipeline)
    .with_cache()
    .with_adapters(tracker)
    .build()
)
```

Practical rule:

- DAG logic belongs in modules
- environment/runtime choices belong in the Builder assembly layer

---

## 6. How Hamilton Builds The DAG {#how-hamilton-builds-the-dag}

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

## 7. Function Modifiers That Matter {#function-modifiers-that-matter}

Function modifiers are central to Hamilton. They do more than ordinary Python
decorators: they can change inclusion, expand nodes, split outputs, inject I/O,
add validation, or attach metadata.

### `@tag`

Use for metadata such as ownership, domain, data product, or policy.

```python
from hamilton.function_modifiers import tag

@tag(owner="growth", data_product="marketing", pii="false")
def weekly_revenue(clean_df):
    ...
```

### `@check_output`

Use for validation without mixing checks into business logic.

```python
from hamilton.function_modifiers import check_output

@check_output(range=(0.0, 1.0), importance="fail")
def churn_probability(raw_score):
    return raw_score.clip(0.0, 1.0)
```

### `@extract_columns` / `@extract_fields`

Use when a composite output should expose downstream-addressable pieces.

```python
from hamilton.function_modifiers import extract_columns

@extract_columns("user_id", "country", "revenue")
def clean_df(raw_df):
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
def revenue_by_dimension(df, groupby_col):
    return df.groupby(groupby_col)["revenue"].sum()
```

### `@config.when(...)` family

Use config-based inclusion when different implementations should resolve to the
same logical node name.

```python
from hamilton.function_modifiers import config

@config.when(model_type="xgboost")
def base_model__xgboost():
    ...

@config.when_not(model_type="xgboost")
def base_model__default():
    ...
```

### `@load_from` / `@save_to`

Use when storage behavior should be explicit in the DAG.

```python
from hamilton.function_modifiers import load_from, save_to, source

@load_from.csv(path=source("raw_input_path"))
def cleaned_data(raw_data):
    return raw_data.dropna()

@save_to.parquet(path=source("output_path"), output_name_="cleaned_data_written")
def cleaned_data_for_save(cleaned_data):
    return cleaned_data
```

Practical interpretation:

- inclusion decorators decide which nodes exist
- parameterization expands one function into many nodes
- extraction exposes column/field lineage
- I/O decorators inject storage edges
- validation decorators attach checks after computation
- metadata decorators help governance and filtering without changing computation

---

## 8. Materialization And Adapters {#materialization-and-adapters}

### Materialization

There are two main styles:

- static materializers attached at build time via `with_materializers(...)`
- dynamic materializers passed to `Driver.materialize(...)`

Dynamic materialization example:

```python
from hamilton import base, driver
from hamilton.io.materialization import from_, to
import pipeline

dr = driver.Builder().with_modules(pipeline).build()

metadata, outputs = dr.materialize(
    from_.csv(target="raw_df", path="./input.csv"),
    to.csv(
        id="feature_export",
        dependencies=["feature_a", "feature_b"],
        path="./out/features.csv",
        combine=base.PandasDataFrameResult(),
    ),
    additional_vars=["feature_a", "feature_b"],
)
```

Use:

- runtime inputs for early prototyping and tests
- static materializers when storage should be part of the graph
- dynamic materialization when deployment code should choose sources/sinks at runtime

### Adapters and result builders

Adapters are the main extension point for runtime concerns:

- result shaping
- tracking
- progress
- hooks
- integrations

Common result builder:

- `base.PandasDataFrameResult()` to collect requested outputs into a DataFrame

Practical rule:

- keep core transformations independent
- attach runtime behavior with adapters rather than embedding it in node code

---

## 9. Caching And Parallelism {#caching-and-parallelism}

### Caching

Hamilton supports builder-level and node-level caching behavior.

Common behavior controls include:

- `DEFAULT`
- `RECOMPUTE`
- `DISABLE`
- `IGNORE`

Typical setup:

```python
dr = (
    driver.Builder()
    .with_modules(my_dataflow)
    .with_cache(recompute=["raw_data"])
    .build()
)
```

Important caveat:

- cache invalidation does not automatically capture helper-function changes or dependency library changes

Practical rule:

- recompute changing external-source nodes
- avoid assuming the cache fully tracks hidden behavior

### Parallelism

Hamilton supports two distinct models.

#### Adapter-based execution

This uses an adapter/executor such as threadpool, Dask, Ray, or async-oriented execution.

```python
from hamilton.plugins.h_threadpool import FutureAdapter

dr = (
    driver.Builder()
    .with_modules(foo_module)
    .with_adapters(FutureAdapter())
    .build()
)
```

Use this first when you need execution parallelism without changing graph structure.

#### Dynamic task-based execution

This uses `Parallelizable[]` and `Collect[]`.

```python
from hamilton.htypes import Collect, Parallelizable

def urls() -> Parallelizable[str]:
    for url in ["https://a", "https://b", "https://c"]:
        yield url

def page_text(urls: str) -> str:
    return fetch(urls)

def word_count(page_text: str) -> int:
    return len(page_text.split())

def total_words(word_count: Collect[int]) -> int:
    return sum(word_count)
```

```python
dr = (
    driver.Builder()
    .with_modules(webflow)
    .enable_dynamic_execution(allow_experimental_mode=True)
    .build()
)
```

Use this only when the graph genuinely contains fan-out / fan-in structure.

### Async support

Hamilton also provides an `AsyncDriver` for async-native runtimes. Use it when
the surrounding environment is already async, not as a default.

---

## 10. Visualization, UI, And CLI {#visualization-ui-and-cli}

### Visualization and graph inspection

Visualization is a normal development tool in Hamilton, not just presentation.
Use it to:

- verify module composition
- inspect the exact execution slice for requested outputs
- debug unexpected dependencies
- communicate lineage to humans

### UI and observability

Hamilton's UI/SDK stack covers execution telemetry, DAG views, lineage, and
artifact / feature cataloging.

Typical local startup:

```bash
pip install "sf-hamilton[ui,sdk]"
hamilton ui
```

Typical tracker wiring:

```python
from hamilton_sdk.adapters import HamiltonTracker

tracker = HamiltonTracker(
    username="you@example.com",
    project_id=1,
    dag_name="marketing_features",
)
```

### CLI and pre-commit workflow

Hamilton also supports CLI-oriented validation / inspection workflows. Treat
CLI checks and DAG validation as part of CI/pre-commit hygiene once the graph
matters to the team.

---

## 11. Project Structure And Testing {#project-structure-and-testing}

Scalable layout:

```text
project/
  dataflows/
    sources.py
    cleaning.py
    features.py
    training.py
    outputs.py
  run_local.py
  run_batch.py
  run_api.py
  tests/
    test_cleaning.py
    test_features.py
    test_training.py
```

Recommended conventions:

- keep functions small and output-oriented
- keep runtime-specific concerns out of DAG modules
- group transformations by domain or stage
- use module composition and `with_config()` for variability
- introduce materializers only when the boundary is clear

Testing layers:

- unit tests for plain transformation functions
- DAG-level tests for a small driver plus toy inputs
- validation / contract tests using `@check_output`, schemas, and graph validation

---

## 12. Decision Rules {#decision-rules}

Use Hamilton when:

- explicit intermediate artifacts matter
- you want readable typed Python instead of hidden pipeline state
- the same graph should run in notebooks, scripts, jobs, and services
- lineage, explainability, or selective execution are valuable

Stay with plain functions or thin scripts when:

- the pipeline is still tiny and graph reasoning adds little
- the main problem is orchestration rather than computation design

Prefer:

- runtime inputs first
- `with_config()` for implementation selection
- materializers when I/O boundaries stabilize
- validation once outputs become decision-relevant
- caching after correctness is stable
- parallelism only for concrete performance needs

Adoption path:

1. extract one existing script or notebook pipeline into one module
2. keep loading as runtime inputs
3. visualize the graph
4. add tags and output validation
5. split columns/fields where lineage matters
6. add materializers when deployment needs settle
7. enable cache selectively
8. add tracker/UI only if operational visibility matters
9. add parallelism only with a concrete reason

---

## 13. Common Mistakes {#common-mistakes}

- writing huge nodes and losing lineage value
- mixing too much I/O into business logic too early
- ignoring node naming discipline
- assuming cache invalidation notices every code change
- treating Hamilton like a scheduler or full platform
- forgetting that runtime inputs can satisfy otherwise "missing" dependencies
- forgetting that many modifiers reshape the graph before execution

---

## 14. When To Read Primary Docs {#when-to-read-primary-docs}

Go to current docs/source when you need:

- exact extras, install, or integration requirements
- backend-specific materializer syntax
- current caching implementation details
- current dynamic execution limitations
- exact UI/CLI commands
- framework-specific integration setup

Primary topics worth revisiting:

- Driver / Builder reference
- function modifiers
- materialization
- caching
- parallel task execution
- AsyncDriver
- UI / SDK integration
