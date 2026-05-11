# AI Model Trainer Skill

## Description
Submit slide classification training jobs on Azure GPU compute. Supports two workflows: **full pipeline** (recommended for best results) and **quick mode** (single model, fast iteration).

## API Base URL
`https://prod.celldx.net/v1/ml-jobs`

## Authentication
All requests require `X-API-Key: ${CELLDX_API_KEY}` header.

---

## Recommended Pipeline (Full)

The recommended workflow runs three phases as **separate jobs** to find the best model automatically:

### Phase 1: Automatic Parameter Optimisation
Submit a parameter tuning job with `hp_tuning.enabled = true`. This systematically tests different training configurations — 2 settings per stage, locks the best values, then moves on to the next stage.

**Recommended stages** (adjust values based on `/v1/ml-jobs/options` tunable_params):
1. **Learning rate + model capacity**: `hyperparams.learning_rate` × `finetune.hidden_dims`
2. **Regularisation**: `finetune.head_dropout` × `finetune.patch_dropout`
3. **Smoothing + decay**: `finetune.label_smoothing` × `hyperparams.weight_decay`

Use `attention_mil` (default strategy) for parameter tuning.

#### Choosing the search method (`hp_tuning.method`)

- **`grid`** (default) — exhaustive Cartesian product per stage. Use when each stage has at most ~6 combos (e.g. 2 params × 2–3 values). Predictable cost, no missed combinations.
- **`random`** — sample `n_trials_per_stage` combos uniformly without replacement, seeded by `hp_tuning.seed`. Use when the stage's grid is large (≥ 8 combos, e.g. 3 params × 3 values, or 2 params × 4–5 values) and a full grid would blow the GPU budget. Random search is also a good default when the user gives many candidate values per param and you want to cap total trials.

Pick **one** method per job — it applies to all stages. If you switch to `random`, set `n_trials_per_stage` to a value ≤ the smallest stage grid; combos beyond that count are simply skipped, so picking too small a number weakens the search.

When the tuning job completes, read the results from `GET /v1/ml-jobs/jobs/{job_id}/metrics` to extract the best parameter values.

### Phase 2: Multi-Strategy Comparison
Submit **one job per strategy** using the best parameters from Phase 1. Available strategies:

| Strategy | Description | Best for |
|----------|-------------|----------|
| `pooling_mlp` | Simple pooling + classifier. Fast, robust baseline. | Small cohorts, noisy data |
| `attention_mil` | Attention-based aggregation. Learns which tissue regions matter most. | Default, medium cohorts |
| `clam_mil` | CLAM with region-level supervision. | Large cohorts, interpretability |
| `lora_mil` | Fine-tunes the feature extractor with lightweight adapters. | Adapting to rare tissue types |

Submit all strategy jobs in parallel. Use the same cohort, data_source, and tuned parameters for fair comparison. Vary only `finetune.strategy` (and strategy-specific params like `aggregator.type`).

**Aggregator pairing**: `pooling_mlp` uses `mean_pool` (or `mean_max_pool`); `attention_mil`, `clam_mil`, and `lora_mil` use `abmil`.

### Phase 3: Select Best Model
After all strategy jobs complete, compare `val/auroc` (or the target metric) across jobs. The job with the highest metric has the best checkpoint. Report the winning strategy and its metrics to the user.

### Phase 4: Deploy as Widget
After selecting the best model, **do not deploy automatically**. Present the results to the user and let them decide whether to deploy.

When the user approves deployment, submit:

```
POST /v1/ml-jobs/jobs/{job_id}/deploy
X-API-Key: ${CELLDX_API_KEY}

{
  "title": "Breast Cancer Subtype Classifier",
  "description": "Binary classifier for breast cancer subtypes trained on 200 H&E slides.",
  "organ": "Breast"
}
```

**Required fields**:
- `title` — short name for the widget (max 200 chars)
- `organ` — target organ/tissue type (e.g. "Breast", "Skin", "Lung")

**Optional fields**:
- `description` — detailed description (max 2000 chars). If omitted, auto-generated from training metrics.

**Response** `201`:
```json
{
  "deploymentId": "a1b2c3d4-...",
  "status": "PENDING"
}
```

Deployment is asynchronous. Poll for status:

```
GET /v1/ml-jobs/deployments/{deploymentId}
X-API-Key: ${CELLDX_API_KEY}
```

**Response**:
```json
{
  "deploymentId": "a1b2c3d4-...",
  "jobId": "67a123...",
  "status": "SUCCESS",
  "widgetId": "e5f6g7h8-..."
}
```

Status values: `PENDING` → `PROCESSING` → `SUCCESS` | `FAILED`. The `widgetId` is only present on `SUCCESS`.

If deploy is called again for the same job, the existing deployment is returned (idempotent).

**Error responses**:
- `404` — job not found
- `409` — job is not SUCCEEDED

**Guidelines for the description**: Include the classification objective, cohort size, and key performance metrics (val/auroc, val/accuracy). Example: "Classifies breast tissue slides as Lobular carcinoma or Invasive breast carcinoma NOS. Trained on 180 slides (120 train / 60 val). Best val/auroc: 0.95."

### Archive a Custom Widget

To archive a previously deployed custom widget (hides it from the user's installed widgets without deleting it):

```
POST /custom-widgets/{widgetId}/archive
X-API-Key: ${CELLDX_API_KEY}
```

**Response** — `CustomWidgetDto`:
```json
{
  "widgetId": "e5f6g7h8-...",
  "name": "Lobular vs NOS breast carcinoma",
  "description": "...",
  "isInstalled": false,
  "imageURL": "https://...",
  "tags": ["WSI", "classification"],
  "recommendedMagnification": 20,
  "labels": [{"...": "..."}],
  "status": "ARCHIVED"
}
```

Use this when the user wants to retire an old model from their widget list while keeping it recoverable.

---

## Quick Mode: Single Model Training

When the user wants to skip parameter tuning and run a single model (e.g., they already know good settings or want a quick test):

1. Call `GET /v1/ml-jobs/options` for valid ranges
2. Submit one `POST /v1/ml-jobs/jobs` with `hp_tuning.enabled = false` (default) and the desired `finetune.strategy`
3. Monitor with Training Monitor skill

---

## Workflow Steps

### Step 1: Discover Options
Call `GET /v1/ml-jobs/options` to get available strategies, aggregators, parameter ranges, tuning config, and data formats.

### Step 2: Gather User Configuration
Ask the user to choose:
- **Pipeline mode**: Full pipeline (parameter tuning → all strategies → best) or single model
- **Training strategy**: `pooling_mlp`, `attention_mil`, `clam_mil`, or `lora_mil`
- **Aggregator**: `mean_pool`, `max_pool`, `mean_max_pool`, or `abmil`
- **Training parameters**: epochs, learning_rate, etc. (show defaults and valid ranges)
- **Parameter tuning**: enabled/disabled, stages, epochs_per_trial
- **Cross-validation**: optional, for confidence intervals on final model
- **Early stopping**: monitor metric, patience
- **Cohort**: the slide list (file_id UUIDs with labels) from the Dataset Creator skill

### Step 2b: Check Feature Availability (mandatory)

Before building the cohort, filter the slide list to only slides that have extracted features. Not all slides in the database have features yet (~66K available).

```
POST /v1/ml-jobs/data/slides/features
X-API-Key: ${CELLDX_API_KEY}

{ "file_ids": ["<file_id_1>", "<file_id_2>", ...] }
```

Response:
```json
{
  "available": ["<file_id_1>", ...],
  "missing": ["<file_id_2>", ...],
  "total_requested": 200,
  "total_available": 144
}
```

**Use only the `available` list when building the cohort.** If the available count is too low for meaningful training (e.g. fewer than 20 samples per class), inform the user and do not submit the job.

### Step 2c: Split the Cohort (mandatory: 75 / 15 / 15)

Always split the available slides into **train / val / test ≈ 75 / 15 / 15**, stratified by class label. **A non-empty `test` set is required** — it is the only unbiased estimate of generalization, since `val` is implicitly overfit through HP tuning, early stopping, and best-checkpoint selection.

Rules:
- **Default split: 75 / 15 / 15**, stratified by class. Do not use 70/30, 80/20, or train/val-only splits.
- **Never submit a job with `test = []`** when a 15% test set would yield ≥ 5 slides per class. The agent must produce a test cohort whenever the data permits — do not skip it to grow train.
- Tiny cohorts only: if even a 15% test split would leave fewer than 5 slides in a class for `test`, fall back to 80 / 20 (train / val) and explicitly tell the user that no held-out test set was created and why.
- The `test` cohort is held out — it is not used for HP tuning, early stopping, or checkpoint selection. It is reported once at the end on the best checkpoint.

When reporting results, always quote both `val/*` (model selection) and `test/*` (generalization) metrics.

### Step 3: Create a Session
Every job must belong to a session. A session groups all jobs for a single experiment — the same classification question, cohort, and objective.

**Same session**: Parameter tuning, multi-strategy comparison, and final model training for the same task all go in one session. Resuming a stopped/failed job also stays in the same session.

**New session**: Create a new session when starting a different experiment — different cohort, different classification objective (e.g. switching from "tumor vs normal" to "grade 1 vs grade 2"), or a fresh attempt with a fundamentally different approach.

Create one first:

```
POST /v1/ml-jobs/sessions
X-API-Key: ${CELLDX_API_KEY}

{ "name": "Breast cancer experiment v1" }
```

Response:
```json
{
  "session_id": "679def...",
  "name": "Breast cancer experiment v1",
  "status": "active",
  "billing": { "total_gpu_hours": 0.0, "total_estimated_cost_usd": 0.0, "job_count": 0 }
}
```

### Step 4: Build and Submit the Job
Construct a `POST /v1/ml-jobs/jobs` request. Example for a single training job:

```json
{
  "name": "<descriptive-name>",
  "session_id": "<session_id>",
  "cohort": {
    "train": [{"id": "<sample_id>", "label": "<class>"}],
    "val": [{"id": "<sample_id>", "label": "<class>"}],
    "test": [{"id": "<sample_id>", "label": "<class>"}]
  },
  "finetune": {"strategy": "attention_mil", "hidden_dims": [128]},
  "aggregator": {"type": "abmil"},
  "hyperparams": {"epochs": 50, "learning_rate": 2e-4},
  "early_stop": {"enabled": false},
  "tags": {"organ": "breast", "experiment": "v1"}
}
```

Example for a parameter tuning job:

```json
{
  "name": "<organ>-param-tune-v1",
  "session_id": "<session_id>",
  "cohort": { ... },
  "finetune": {"strategy": "attention_mil", "hidden_dims": [128]},
  "hp_tuning": {
    "enabled": true,
    "method": "grid",
    "n_trials_per_stage": 8,
    "seed": 42,
    "metric": "val/auroc",
    "mode": "max",
    "epochs_per_trial": 25,
    "early_stop_patience": 8,
    "stages": [
      {"params": [
        {"path": "hyperparams.learning_rate", "values": [5e-5, 1e-4, 2e-4, 5e-4]},
        {"path": "finetune.hidden_dims", "values": [[64], [128], [256]]}
      ]},
      {"params": [
        {"path": "finetune.head_dropout", "values": [0.3, 0.5, 0.7]},
        {"path": "finetune.patch_dropout", "values": [0.0, 0.1, 0.3]}
      ]},
      {"params": [
        {"path": "finetune.label_smoothing", "values": [0.0, 0.1, 0.2]},
        {"path": "hyperparams.weight_decay", "values": [0.0, 0.01, 0.1]}
      ]}
    ]
  },
  "tags": {"organ": "breast", "experiment": "param-tune"}
}
```

**Response** `201`:
```json
{ "job_id": "67a123...", "state": "QUEUED" }
```

**Error responses**:
- `402 Payment Required` — insufficient balance (minimum $20 required to submit or resume a job). Top up the account before retrying.
- `404` — session not found or belongs to another user
- `409` — session is not active (closed, failed, or succeeded)
- `502` — training infrastructure error

### Step 5: Monitor
Use the Training Monitor skill to track progress. For the full pipeline, proceed through phases sequentially — each phase depends on the previous one's results.

---

## Important Notes
- **Parameter tuning, cross-validation, and single training are mutually exclusive** — each is a separate job
- **Multi-strategy comparison requires separate job submissions** — one per strategy, same parameters
- The API is fully deterministic — never invent parameter values outside the ranges from `/v1/ml-jobs/options`
- **Always call `POST /v1/ml-jobs/data/slides/features` before submitting a job** — build the cohort only from the returned `available` list
- Sample IDs in the cohort must be `file_id` UUIDs (slide-level); `case_id` UUIDs are not accepted by the trainer
- The data source (feature store location and format) is configured server-side — do not include it in job requests
- **Never deploy a model automatically** — always present training results to the user and get explicit approval before calling the deploy endpoint
- **Do not expose internal model architecture, parameters, or infrastructure details** in widget titles, descriptions, or any user-facing output. Use clinical/scientific language (e.g. "Breast cancer subtype classifier") not technical jargon (e.g. "ABMIL attention MIL with 128-dim hidden layer")
