# Hamilton Conventions Reference

## Table of Contents
1. [What It Is](#what-it-is)
2. [Mental Model](#mental-model)
3. [Canonical Pattern](#canonical-pattern)
4. [Authoring Conventions](#authoring-conventions)
5. [Decision Rules](#decision-rules)
6. [Advanced Features](#advanced-features)
7. [Decorators That Change Behavior](#decorators)
8. [Sharp Edges](#sharp-edges)
9. [When to Read Primary Docs](#when-to-read-primary-docs)

---

## 1. What It Is {#what-it-is}

Hamilton is a Python dataflow framework where:

- function names define node names
- parameter names define dependencies
- type hints define expected contracts
- the driver builds and executes the resulting DAG

Use it for transformation logic, dependency visibility, selective execution,
validation, and optional caching/materialization.

Hamilton is **not** a scheduler, infrastructure orchestrator, or deployment
platform. Keep orchestration outside Hamilton.

For this workbench, Hamilton is a good fit when the dependency graph itself
helps compare, review, or selectively execute analysis outputs.

---

## 2. Mental Model {#mental-model}

The core idea is simple:

- a function declares how to compute one artifact
- its parameters name the upstream artifacts it needs
- Hamilton matches names to build the DAG
- runtime inputs can satisfy dependencies that are not defined as functions

Example:

```python
def spend_mean(spend):
    return float(spend.mean())
```

This does **not** call `spend`. It declares that `spend_mean` depends on a node
or runtime input named `spend`.

Important implications:

- name outputs like artifacts, not actions
- prefer many small functions over one long pipeline function
- a missing dependency is fine if you plan to provide it in `inputs=...`
- helper functions prefixed with `_` are ignored by Hamilton graph discovery

Think of Hamilton as separating:

- **dataflow definition**: plain Python functions in modules
- **runtime behavior**: the Driver, Builder, adapters, executors, materializers, cache

That separation is the main reason Hamilton stays readable.

---

## 3. Canonical Pattern {#canonical-pattern}

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


def _round2(x: float) -> float:
    return round(x, 2)
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

What this demonstrates:

- `_round2` is ignored because of the leading underscore
- `spend` and `signups` are runtime inputs, not Hamilton functions
- `Builder().with_modules(...).build()` is the normal assembly pattern
- `execute([...], inputs=...)` asks Hamilton to compute only the requested outputs

---

## 4. Authoring Conventions {#authoring-conventions}

Write Hamilton-friendly modules as collections of small, output-oriented,
mostly side-effect-free functions.

Recommended:

- one function = one meaningful artifact
- function names read like outputs
- type hints on all public node functions
- pure transformations in modules
- thin runtime scripts that assemble `Builder`

Avoid:

- mutating shared global state
- burying disk/network I/O in core transformation functions
- giant "do everything" functions
- ambiguous node names like `process_data` or `run_step`

Use a leading underscore for local helpers that should not become DAG nodes.

This style is useful even without Hamilton runtime because it lowers edit blast
radius, improves testability, and keeps dependency structure visible.

---

## 5. Decision Rules {#decision-rules}

### Plain functions vs Hamilton runtime

Stay with plain imports when:

- the module graph is still changing rapidly
- script order is obvious and selective execution is not useful
- graph visualization would not change decisions

Add Hamilton runtime when:

- you want to request specific outputs and run only their dependencies
- the graph itself helps reasoning or review
- multiple entry points should reuse the same transformation modules
- node-level caching or validation would save real time

### Runtime inputs vs I/O in the graph

Use runtime inputs when:

- prototyping
- unit testing
- the caller should control data loading
- storage paths/systems vary by environment

Use materializers when:

- deployment code should decide what gets loaded/saved
- you want loader/saver nodes to be explicit and reusable
- the graph should include storage edges without baking them into business logic

Use decorator-based I/O when:

- loading/saving behavior is part of the dataflow contract itself
- you want those edges expressed directly on the node definition

Default recommendation:

- start with runtime inputs
- move to materializers once I/O contracts stabilize
- use decorator-based I/O only when explicit graph-level storage behavior is valuable

### Caching

Use caching when:

- intermediate nodes are expensive
- graph structure is stable enough that cache reuse matters

Avoid heavy reliance on caching when:

- upstream data changes constantly
- nodes depend on hidden helper behavior or unstable external libraries

Practical rule:

- start without cache
- add it after correctness and graph shape are stable
- recompute or disable cache for external source loaders

### Parallel execution

Use adapter/executor-based parallelism when:

- work is I/O-bound or executor-backed parallelism is enough
- you do not want to change node signatures much

Use dynamic execution with `Parallelizable[]` and `Collect[]` when:

- the graph genuinely contains fan-out/fan-in work
- per-item parallel subgraphs are part of the logical model

Do not enable dynamic execution by default. It is a structural choice, not a speed flag.

---

## 6. Advanced Features {#advanced-features}

### Config-driven variants

Use `Builder.with_config({...})` together with `@config.when(...)` family
decorators when the graph should swap implementations by mode, environment, or
experiment choice.

```python
from hamilton.function_modifiers import config

@config.when(model_type="xgboost")
def base_model__xgboost():
    ...

@config.when_not(model_type="xgboost")
def base_model__default():
    ...
```

```python
dr = (
    driver.Builder()
    .with_modules(training)
    .with_config({"model_type": "xgboost"})
    .build()
)
```

Decision rule:

- use this to swap node implementations cleanly
- do not scatter environment branching across runner code if the difference is really a graph choice

### Materializers

Hamilton supports two operational patterns:

- static materializers via `Builder.with_materializers(...)`
- dynamic materialization via `Driver.materialize(...)`

Static materializers are usually the better default because load/save nodes
become part of the graph and can be visualized or requested by name.

```python
from hamilton.io.materialization import from_, to

dr = (
    driver.Builder()
    .with_modules(pipeline)
    .with_materializers(
        from_.csv(target="raw_df", path="./input.csv"),
        to.csv(id="feature_export", dependencies=["feature_a"], path="./out/features.csv"),
    )
    .build()
)
```

Dynamic materialization is useful when the caller should decide load/save
behavior at execution time:

```python
metadata, outputs = dr.materialize(
    from_.csv(target="raw_df", path="./input.csv"),
    to.csv(id="feature_export", dependencies=["feature_a"], path="./out/features.csv"),
    additional_vars=["feature_a"],
)
```

Decision rule:

- start with runtime inputs
- prefer `with_materializers(...)` when storage should be visible in the graph
- use `materialize(...)` when execution-time flexibility matters more than graph permanence

### Caching

Enable caching with `Builder.with_cache()`.

```python
dr = (
    driver.Builder()
    .with_modules(my_dataflow)
    .with_cache(recompute=["raw_data"])
    .build()
)
```

Useful behaviors exposed by current docs include default caching plus explicit
`recompute`, `disable`, and `ignore` controls.

Decision rule:

- cache expensive, stable intermediate nodes
- recompute changing external-source loaders
- do not assume helper-function edits or library upgrades will always invalidate cached nodes correctly

### Dynamic execution

Hamilton supports structural fan-out/fan-in execution using
`Parallelizable[]` and `Collect[]`.

```python
from hamilton.htypes import Collect, Parallelizable

def urls() -> Parallelizable[str]:
    for url in ["https://a", "https://b", "https://c"]:
        yield url

def page_text(urls: str) -> str:
    return fetch(urls)

def total_length(page_text: Collect[str]) -> int:
    return sum(len(text) for text in page_text)
```

```python
dr = (
    driver.Builder()
    .with_modules(webflow)
    .enable_dynamic_execution(allow_experimental_mode=True)
    .build()
)
```

Decision rule:

- use this only when the graph naturally contains repeated parallel branches
- do not treat it as a generic speed toggle
- re-check current docs for limitations before relying on it heavily

### Visualization and validation

Visualization is part of the normal development loop, not just presentation.
Use DAG display methods when the graph shape itself helps reasoning or review.

Validation decorators are worth using when data quality matters but you want to
keep business logic separate from checks.

```python
from hamilton.function_modifiers import check_output

@check_output(importance="fail")
def scored_df(features_df):
    ...
```

Decision rule:

- add validation where downstream analysis would be misleading without it
- keep checks declarative rather than mixing assertions deep into transforms

---

## 7. Decorators That Change Behavior {#decorators}

The important nuance is that Hamilton decorators often change the graph at build
time, not just runtime behavior.

Useful categories:

- `@config.when`, `@config.when_not`, `@config.when_in`: include/exclude node implementations by config
- `@parameterize`: generate many nodes from one implementation
- `@extract_columns` or `@extract_fields`: split a composite output into downstream-addressable nodes
- `@load_from`, `@save_to`: inject explicit loader/saver behavior into the DAG
- `@check_output`: validate outputs without mixing validation into business logic
- `@tag`: attach metadata for filtering, ownership, lineage, or policy

Practical interpretation:

- inclusion decorators change which nodes exist
- parameterization decorators expand one function into multiple nodes
- I/O decorators inject graph edges around storage
- validation decorators attach checks after the core node exists
- metadata decorators help filtering and lineage but do not change computation

This explains why Hamilton decorators feel stronger than ordinary Python decorators.

---

## 8. Sharp Edges {#sharp-edges}

### Name collisions across modules

If two loaded modules define the same node name, `build()` fails unless you
explicitly allow module overrides. This is useful for swapping variants, but
it should be intentional.

If you really do want later modules to win, use the Builder option that allows
module overrides rather than relying on accidental import order.

### Runtime inputs can hide missing definitions

This is a feature, not a bug, but it can make the graph feel incomplete if you
forget which dependencies are expected from `inputs=...`.

### Caching does not notice every code change

Caching is based on node code/version behavior, but hidden helper-function edits
or external library changes may not invalidate the cached node the way you expect.

### Decorators can reshape the graph

If the graph looks surprising, check decorators before debugging execution.
`@config.when`, `@parameterize`, `@load_from`, and extraction decorators often
explain "missing" or "extra" nodes.

### Dynamic execution is not ordinary parallelism

Dynamic execution requires explicit enablement and specific graph structure.
It is not the same thing as simply using a parallel executor.

### Over-coupling I/O too early

New teams often bake file paths, storage assumptions, or remote systems directly
into node functions. Prefer pure transforms first, then add materializers or
I/O decorators when the operational contract is clear.

### Using Hamilton as the whole platform

Hamilton should usually sit inside a broader system. Keep orchestration,
scheduling, deployment, and environment management outside the transformation graph.

---

## 9. When to Read Primary Docs {#when-to-read-primary-docs}

Go to official docs or current source references when you need:

- exact install/version requirements
- current extras for visualization, UI, Pandera, Pydantic, or SDK support
- details of cache backends and cache-behavior configuration
- the latest dynamic execution limitations
- materializer/backend-specific syntax
- exact CLI/UI commands

Useful primary-source topics:

- Driver/Builder API
- function modifiers and decorator categories
- materialization and I/O
- caching behavior
- dynamic execution with `Parallelizable[]` / `Collect[]`
- visualization and UI integrations
