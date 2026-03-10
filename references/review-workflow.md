# Review Workflow Reference (Tier 3+)

Use this reference when the project needs a formal human-in-the-loop execution
protocol rather than conversational review.

At Tier 1-2, the same pattern happens informally in chat or in a notebook.
At Tier 3+, write the state transitions and review artifacts down.

## State Machine

Every stage moves through a small lifecycle:

```
STALE -> EXECUTED -> REVIEWED -> APPROVED
                 \-> REJECTED -> STALE
```

Rules:
- Never update a report with results from a stage that is not approved.
- Always self-review before presenting outputs to the human.
- Human approval covers both artifacts and interpretation text.
- Never overwrite an approved run; supersede it with a newer run.

## Workflow

Follow this sequence for each stale stage:

1. Run the stage and produce explicit outputs.
2. Write `manifest.json` describing params, inputs, outputs, and code ref.
3. Run self-review checks.
4. Write `review.json`.
5. Draft `card.md` for human review.
6. Present the card and recommendation.
7. If approved, write `approval.json` and only then update the narrative/report.
8. If rejected, write `feedback.json`, incorporate feedback, and re-run.

For exact schemas, use `core-contracts.md`.

## Self-Review Expectations

Before presenting a stage, verify:
- Declared outputs exist and are non-empty.
- Key metrics are plausible and internally consistent.
- Figures are non-trivial and match the metrics summary.
- No `NaN` or `Inf` values leak into important result columns.
- Date ranges, row counts, or other coverage checks match the requested window.

If blocking issues appear, fix them before presenting anything to the human.

## Data Access Rules

Use small CLI tools in `tools/` for external systems. Each tool should:
- Accept `--output PATH`.
- Accept `--format csv|json` or infer from extension.
- Read credentials from `.env`.
- Print diagnostics to stderr.
- Exit non-zero on failure.

This keeps data acquisition separate from stage orchestration.

## Narrative Rules

The final report should read from saved artifacts only:
- `data/raw/`
- `data/processed/`
- `outputs/figures/`
- approved interpretation text from `approval.json`

Do not bury critical findings in tool metadata alone. Always materialize:
- per-run artifact folders
- machine-readable summaries
- comparison tables
- report-ready figures and tables

## When This Formality Is Worth It

Use the full review workflow when:
- multiple runs need to be compared over time
- the human wants explicit approval checkpoints
- the AI is advancing the project stage by stage
- results will feed a polished report or decision memo

If the work is still lightweight, keep the same logic but do it conversationally.
