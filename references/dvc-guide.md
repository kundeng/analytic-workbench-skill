# DVC Guide (Tier 3+)

Use this reference once the workbench has grown into explicit cached stages and
reproducible multi-run comparisons.

DVC is the persistence layer for:

- stage DAG execution
- file-level caching and reuse
- parameter tracking
- experiment comparison
- remote data/cache sharing

Do not treat it as the whole workflow. The workflow is defined by
`analytic-workbench`; DVC is one implementation layer inside Tier 3.

## Setup

```bash
pip install "dvc[s3]" --break-system-packages
dvc init
```

Commit the DVC initialization files. Keep raw/generated data out of Git and in
DVC-tracked outputs or ignored folders.

## Minimal `dvc.yaml`

```yaml
stages:
  extract:
    cmd: python tools/fetch_data.py --output rawdata/incidents.csv
    deps:
      - tools/fetch_data.py
    params:
      - source
    outs:
      - rawdata/incidents.csv

  baseline:
    cmd: python scripts/run.py baseline.resample_freq=${baseline.resample_freq}
    deps:
      - scripts/run.py
      - src/baseline.py
      - rawdata/incidents.csv
    params:
      - baseline.resample_freq
    outs:
      - runs/baseline/
```

Track tunable values in `params.yaml`, not inline shell constants.

## Core Commands

```bash
dvc status
dvc repro
dvc repro STAGE
dvc repro -f
dvc dag
```

Interpretation:

- `dvc status` shows stale stages.
- `dvc repro` runs stale stages in DAG order.
- `dvc repro STAGE` runs a target stage and its upstream dependencies.
- `-f` forces re-execution even when hashes match.

## Experiments

Use DVC experiments when you want structured comparisons beyond ad hoc Hydra
overrides.

```bash
dvc exp run
dvc exp run -S analysis.window_size=72
dvc exp run --queue -S analysis.window_size=24,72,168
dvc queue start --jobs 4
dvc exp show
dvc exp diff
dvc exp apply EXP_NAME
```

Good practice:

- still write explicit per-run artifacts to `runs/`
- still build a comparison table
- use `dvc exp show` as a helper, not the only source of comparison truth

## Caching and Freshness

DVC hashes declared `deps`, `params`, and `outs`. If they did not change, the
stage is skipped. That makes it appropriate for policies like:

"Do not re-pull source data again for 48 hours unless the human asks for a
fresh run."

Encode that policy through stage design and parameterization, not through hidden
memory in the chat.

## Remotes

Typical flow:

```bash
dvc remote add -d myremote s3://my-bucket/dvc-cache
dvc push
dvc pull
```

Use remote storage when cached artifacts need to be shared across machines or
recovered later.

## When to Add DVC

Add DVC when at least one of these becomes painful without it:

- expensive upstream data pulls
- repeat runs with mostly unchanged inputs
- disciplined comparison across many runs
- remote sharing of cached artifacts
- reproducible stage history that should survive the current workspace

If none of that hurts yet, stay in Tier 2 and keep using explicit folders plus
comparison tables.
