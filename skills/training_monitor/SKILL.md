# Training Monitor Skill

## Description
Monitor running training jobs, view metrics, stop/adjust/resume training. Supports the iterative "train → evaluate → adjust → retrain" workflow.

## API Base URL
`https://prod.celldx.net/v1/ml-jobs`

## Authentication
All requests require `X-API-Key: ${CELLDX_API_KEY}` header.

## Capabilities

### Check Job Status
`GET /v1/ml-jobs/jobs/{job_id}/status` — Quick status check (PENDING, QUEUED, RUNNING, SUCCEEDED, FAILED, STOPPED).

### View Metrics
`GET /v1/ml-jobs/jobs/{job_id}/metrics` — Epoch-by-epoch metrics history. Present as a table or summary:
- `train/loss`, `val/loss` — loss curves
- `val/accuracy`, `val/auroc`, `val/f1_macro` — performance metrics

When presenting metrics, highlight:
- Is val/loss still decreasing? (good)
- Is val/loss rising while train/loss drops? (overfitting — suggest reducing learning rate or adding regularisation)
- Has val/auroc plateaued? (consider stopping)

### View Logs
`GET /v1/ml-jobs/jobs/{job_id}/logs?tail=50` — Recent trainer stdout.

### List Jobs
`GET /v1/ml-jobs/jobs?state=RUNNING&limit=10` — List jobs with optional filters.

### Stop a Job
`POST /v1/ml-jobs/jobs/{job_id}/stop` — Cancel a running job. Only works for RUNNING or QUEUED jobs.

### Adjust Configuration
`PATCH /v1/ml-jobs/jobs/{job_id}` — Update training parameters before resuming. Only works for STOPPED or FAILED jobs. Body:

```json
{
  "hyperparams": {"learning_rate": 5e-4, "epochs": 30},
  "early_stop": {"patience": 10}
}
```

### Resume Training
`POST /v1/ml-jobs/jobs/{job_id}/resume` — Create a new job from the last checkpoint of a stopped/failed job. The new job inherits the (possibly updated) configuration.

**Note**: Resuming requires at least $20 balance, same as submitting a new job. Returns `402 Payment Required` if insufficient.

### View Session Billing
`GET /v1/ml-jobs/sessions/{session_id}` — Session details including accumulated GPU hours and estimated cost across all jobs in the session. A session groups all jobs for a single experiment (same cohort and classification objective).

### Get Session Runs (dashboard data)
`GET /v1/ml-jobs/sessions/{session_id}/runs` — Returns every job in the session along with its full per-epoch metric history. This is what the frontend renders as line charts (val/auroc, train/loss, etc.) — there is no separate dashboard URL or iframe. The response shape:

```json
{
  "session_id": "...",
  "runs": [
    {
      "job_id": "...",
      "name": "...",
      "state": "SUCCEEDED",
      "strategy": "attention_mil",
      "started_at": "...",
      "finished_at": "...",
      "metrics_summary": {"val/auroc": 0.87},
      "history": [
        {"epoch": 1, "metrics": {"val/auroc": 0.71, "train/loss": 0.5}},
        {"epoch": 2, "metrics": {"val/auroc": 0.78, "train/loss": 0.4}}
      ]
    }
  ]
}
```

Runs are sorted oldest-first. For real-time updates while a job is running, the agent should re-fetch this endpoint or rely on the backend's WebSocket channel for the session.

## Typical Workflow

1. User: "How is my training going?"
   → `GET /v1/ml-jobs/jobs/{job_id}/metrics`
   → Present epoch-by-epoch table, note trends

2. User: "Stop it, reduce learning rate to 5e-4"
   → `POST /v1/ml-jobs/jobs/{job_id}/stop`
   → `PATCH /v1/ml-jobs/jobs/{job_id}` with `{"hyperparams": {"learning_rate": 5e-4}}`
   → `POST /v1/ml-jobs/jobs/{job_id}/resume`
   → Report new job ID

3. User: "Show me all running jobs"
   → `GET /v1/ml-jobs/jobs?state=RUNNING`

## Monitoring Parameter Tuning Jobs

Parameter tuning jobs emit trial-level metrics alongside epoch metrics. The metrics endpoint returns entries with `"trial"` key:

```json
{"trial": 5, "stage": 1, "params": {"hyperparams.learning_rate": 1e-4}, "best_metric": 0.76}
```

And a final summary:
```json
{"hp_tuning_summary": true, "total_trials": 30, "overall_best_trial": 17, "overall_best_metric": 0.76}
```

When monitoring parameter tuning:
- Report current stage and trial progress (e.g., "Stage 2, trial 5 of 9")
- Show the best metric found so far across all trials
- Show which parameter values are performing best

## Comparing Strategy Jobs

When multiple strategy jobs run in parallel (Phase 2 of the full pipeline), monitor all of them and present a comparison table when they complete:

| Strategy | val/auroc | val/bacc | Best Epoch |
|----------|-----------|----------|------------|
| pooling_mlp | 0.802 | 0.65 | 18 |
| attention_mil | 0.711 | 0.55 | 24 |

Highlight the winning strategy and recommend it for deployment.

## Deploying a Trained Model

When training succeeds and the user wants to deploy:

1. Review the final metrics with the user
2. **Get explicit user approval** before deploying
3. Call `POST /v1/ml-jobs/jobs/{job_id}/deploy`:

```json
{
  "title": "Breast Cancer Subtype Classifier",
  "description": "Classifies breast tissue slides. Best val/auroc: 0.95.",
  "organ": "Breast"
}
```

Response `201`:
```json
{
  "deploymentId": "a1b2c3d4-...",
  "status": "PENDING"
}
```

4. Poll deployment status with `GET /v1/ml-jobs/deployments/{deploymentId}`:

```json
{
  "deploymentId": "a1b2c3d4-...",
  "jobId": "67a123...",
  "status": "SUCCESS",
  "widgetId": "e5f6g7h8-..."
}
```

Status values: `PENDING` → `PROCESSING` → `SUCCESS` | `FAILED`. The `widgetId` appears only on `SUCCESS`.

- `title` and `organ` are required
- `description` is auto-generated from metrics if omitted
- Only SUCCEEDED jobs can be deployed; calling deploy again for the same job returns the existing deployment
- **Never expose internal model parameters or infrastructure details** in the title or description — use clinical/scientific language only

## Important Notes
- **Training jobs run on pre-extracted features — they do NOT consume WSI purchases.** When reporting cost (e.g. via `GET /v1/ml-jobs/sessions/{session_id}`), the figure is GPU compute only. Never add WSI per-slide prices ($5 H&E, $40 IHC from the `cohort_builder` skill) to a training cost report — those apply only when downloading WSI files, which is a separate workflow. CellDX has 220K+ WSIs available for purchase via `cohort_builder` and ~66K slides with pre-extracted features available for training via `ai_model_trainer`; the two flows are independent.
- All adjustments are explicit — the API never auto-adjusts parameters
- When reporting metrics, always show actual numbers, not vague descriptions
- The resume endpoint creates a NEW job (new job_id) that loads the checkpoint from the parent
- The training dashboard is rendered by the frontend directly off `GET /v1/ml-jobs/sessions/{session_id}/runs` — no separate iframe / signed URL is involved.
