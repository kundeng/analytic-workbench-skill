# Core Contracts Reference (Tier 3+)

Formal file contracts for the execute-review-approve loop. Use these at Tier 3+
when the project needs persistent audit trails and reproducibility records.

At Tier 1–2, the same loop happens conversationally — no files needed.

## Table of Contents
1. [State Machine](#state-machine)
2. [manifest.json](#manifest)
3. [review.json](#review)
4. [approval.json](#approval)
5. [feedback.json](#feedback)
6. [card.md](#card)

---

## 1. State Machine {#state-machine}

Every pipeline stage has a state. The agent respects transitions.

```
 STALE ──agent runs──▶ EXECUTED ──agent reviews──▶ REVIEWED
                                                      │
                                          human approves │ rejects
                                              ▼              ▼
                                          APPROVED        STALE
                                                    (agent applies
                                                     feedback, re-runs)
```

**Rules:**
- Never update a report/narrative with results from an unapproved stage.
- Always self-review before presenting to the human.
- Always write manifest.json after execution.
- Human approves both artifacts and AI interpretations.
- Never delete or overwrite approval files — supersede with new runs.

---

## 2. manifest.json {#manifest}

Written by agent after every stage execution.

```json
{
  "stage": "baseline",
  "run_id": "2026-03-08T1422Z",
  "parent_run_id": "2026-03-08T1201Z",
  "status": "executed",
  "params_used": {
    "baseline.resample_freq": "1h"
  },
  "inputs": {
    "data/raw/incidents.csv": "sha256:abc123..."
  },
  "outputs": {
    "data/processed/baseline/timeseries.csv": "sha256:def456...",
    "data/processed/baseline/summary.json": "sha256:789abc..."
  },
  "code_ref": "scripts/02_baseline.py@git:a1b2c3d",
  "duration_seconds": 12.4
}
```

Generate `run_id` as ISO timestamp. Compute file hashes:

```python
import hashlib
def file_hash(path):
    return hashlib.sha256(open(path, "rb").read()).hexdigest()
```

---

## 3. review.json {#review}

Written by agent after self-review, before presenting to human.

```json
{
  "run_id": "2026-03-08T1422Z",
  "stage": "baseline",
  "verdict": "pass",
  "checks": [
    {"name": "output_files_exist", "pass": true},
    {"name": "non_empty_outputs", "pass": true},
    {"name": "figures_non_trivial", "pass": true},
    {"name": "summary_valid", "pass": true},
    {"name": "no_nan_inf", "pass": true},
    {"name": "row_count_plausible", "pass": true, "note": "8,760 hourly periods"}
  ],
  "warnings": [],
  "blocking_issues": [],
  "recommendation": "Ready for human review"
}
```

- If `blocking_issues` is non-empty: `verdict: "fail"`, agent fixes before presenting.
- If only warnings: `verdict: "pass_with_warnings"`.
- If all pass: `verdict: "pass"`.

---

## 4. approval.json {#approval}

Written by agent when human approves (verbally or in writing).

```json
{
  "run_id": "2026-03-08T1422Z",
  "stage": "baseline",
  "decision": "approved",
  "approved_by": "human",
  "timestamp": "2026-03-08T14:30:00Z",
  "artifacts_approved": true,
  "interpretation_approved": true,
  "interpretation_edited": false,
  "approved_interpretation": "The hourly time series covers 12 months with...",
  "note": "Looks good",
  "supersedes": "2026-03-08T1201Z"
}
```

When the human edits the AI's interpretation:
- `interpretation_edited: true`
- `approved_interpretation` contains the human's version
- The narrative/report uses `approved_interpretation`, not the AI's draft

---

## 5. feedback.json {#feedback}

Written by agent when human rejects with feedback.

```json
{
  "run_id": "2026-03-08T1422Z",
  "stage": "baseline",
  "issue": "Time series has a 3-day gap in November",
  "suggested_params": {},
  "acceptance_criteria": [
    "No gaps > 24h in the time series",
    "Investigate if the gap is real or a data issue"
  ]
}
```

---

## 6. card.md {#card}

Human-readable stage summary. Contains both factual outputs AND the AI's
draft interpretation. The human approves or edits both.

```markdown
# Stage: Baseline | Run 2026-03-08T1422Z

**Status:** Reviewed — ready for approval
**Duration:** 12.4s
**Key params:** resample_freq=1h

## Artifacts
- 8,760 hourly periods covering 2025-03-08 to 2026-03-08
- Mean: 1.4 incidents/hour, Max: 28, Zero-hour rate: 12%
- Files: timeseries.csv, summary.json
- Figures: fig-timeseries.png, fig-by-priority.png

## AI Self-review
All checks passed. No warnings.

## AI Interpretation (draft — pending human approval)
The hourly incident time series shows a clear diurnal pattern with peaks
during business hours. The zero-hour rate of 12% indicates consistent
activity. Three visible spikes correlate with known outage dates.

## Recommendation
Approve if the time series coverage and patterns look reasonable.
Edit the interpretation above if the framing needs adjustment.
```

---

## Workflow: Step by Step (Tier 3)

```
1. Check pipeline state (dvc status)
2. Run stale stages (dvc repro)
3. For each completed stage:
   a. Write review/STAGE/manifest.json
   b. Run self-review checks
   c. Write review/STAGE/review.json
   d. Draft AI interpretation
   e. Write review/STAGE/card.md
   f. Present card to human
4. Human decides:
   - Approved → write approval.json, update narrative
   - Approved with edits → use human's interpretation
   - Rejected → write feedback.json, fix, go to step 1
5. Never update narrative with unapproved results
```

---

## Data Access: CLI Tools in `tools/`

Every data-access tool must:
- Accept `--output PATH` for file output
- Accept `--format csv|json`
- Read credentials from `.env` (never CLI args or params.yaml)
- Print progress to stderr
- Exit non-zero on failure
- Be independently runnable: `python tools/my_tool.py --help`
