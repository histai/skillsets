# HistAI Pathology Datahub Skills

This file provides skill definitions for OpenAI Codex and other agents that support the AGENTS.md format.

---

## Cohort Builder

**Skill ID:** `cohort_builder`  
**Path:** `skills/cohort_builder/SKILL.md`

### Description

Search pathology cases by diagnosis, organ, age, and stains. Filter datasets (benign/malignant, cancer types), build research cohorts, and export whole slide images with clinical and technical metadata.

### Prerequisites

- Active subscription on [CellDX platform](https://celldx.hist.ai)
- 2FA authentication enabled
- API Key generated from Profile Settings → API Keys

### Environment Setup

```bash
export CELLDX_API_KEY="your-api-key"
```

### When to Use This Skill

Use this skill when you need to:
- Search for pathology cases by clinical criteria (diagnosis, organ, cancer type)
- Filter datasets by technical parameters (stains, magnification, scanner)
- Build training cohorts for machine learning models
- Export whole slide images (WSI) and metadata for research
- Calculate dataset distributions and statistics
- Manage billing and cohort costs

### ⚠️ Financial Safety & Confirmation

**Purchasing a cohort involves real financial charges and is non-refundable.**

AI agents **MUST** receive explicit confirmation from the user (a clear "YES" or "PROCEED") before executing any payment (`/pay`) command. Agents should always present the total estimated cost and cohort contents to the user before asking for confirmation.

### Key Capabilities

1. **Case Search & Discovery**
   - Search by diagnosis, organ, age, gender, ICD-10
   - Filter by stain types (H&E, IHC markers like HER2, ER, Ki-67)
   - Search within pathology protocols (macro, micro, conclusion)
   - Get available filter values and counts

2. **Cohort Management**
   - Create named cohorts from search results
   - Add or remove cases and slides
   - Optimize costs by selecting specific slides
   - Track cohort status and payment

3. **Data Export**
   - Download whole slide images
   - Export clinical and technical metadata
   - Manage download windows and refresh expired links

4. **Billing & Pricing**
   - Check account balance
   - View transaction history
   - Volume discounts based on cohort size

### API Base URL

```
https://prod.celldx.net
```

### Authentication

All requests require the `X-API-KEY` header:

```bash
X-API-KEY: $CELLDX_API_KEY
```

---

## AI Model Trainer

**Skill ID:** `ai_model_trainer`  
**Path:** `skills/ai_model_trainer/SKILL.md`

### Description

Submit slide classification training jobs on Azure GPU compute. Supports two workflows: **full pipeline** (parameter tuning → multi-strategy comparison → best model selection → deployment) and **quick mode** (single model, fast iteration).

### Prerequisites

- Active subscription on [CellDX platform](https://celldx.hist.ai)
- 2FA authentication enabled
- API Key generated from Profile Settings → API Keys
- A cohort with slides that have extracted features (check via the feature availability endpoint)

### Environment Setup

```bash
export CELLDX_API_KEY="your-api-key"
```

### When to Use This Skill

Use this skill when you need to:
- Train slide classification models (binary or multi-class)
- Run hyperparameter tuning to find optimal training settings
- Compare multiple training strategies (pooling_mlp, attention_mil, clam_mil, lora_mil)
- Deploy a trained model as a widget on the CellDX platform

### Key Capabilities

1. **Full Pipeline** (recommended)
   - Phase 1: Automatic parameter optimisation (learning rate, regularisation, etc.)
   - Phase 2: Multi-strategy comparison with tuned parameters
   - Phase 3: Best model selection based on val/auroc
   - Phase 4: Deploy as widget (requires user approval)

2. **Quick Mode**
   - Single model training with user-specified parameters
   - Skip tuning for fast iteration

3. **Session Management**
   - Group related jobs into sessions for billing and tracking

### ⚠️ Safety

- **Never deploy a model automatically** — always get explicit user approval
- **Always check feature availability** before submitting a job
- **Minimum $20 balance** required to submit or resume a job

### API Base URL

```
https://prod.celldx.net/v1/ml-jobs
```

### Authentication

All requests require the `X-API-Key` header:

```bash
X-API-Key: $CELLDX_API_KEY
```

---

## Training Monitor

**Skill ID:** `training_monitor`  
**Path:** `skills/training_monitor/SKILL.md`

### Description

Monitor running training jobs, view epoch-by-epoch metrics, stop/adjust/resume training, and deploy trained models. Supports the iterative "train → evaluate → adjust → retrain" workflow.

### Prerequisites

- Active subscription on [CellDX platform](https://celldx.hist.ai)
- 2FA authentication enabled
- API Key generated from Profile Settings → API Keys
- One or more training jobs submitted via the AI Model Trainer skill

### Environment Setup

```bash
export CELLDX_API_KEY="your-api-key"
```

### When to Use This Skill

Use this skill when you need to:
- Check training job status and progress
- View epoch-by-epoch metrics (loss, auroc, accuracy, f1)
- Diagnose training issues (overfitting, plateau)
- Stop, adjust parameters, and resume training
- Compare results across multiple strategy jobs
- View session billing and GPU usage
- Deploy a trained model as a widget

### Key Capabilities

1. **Status & Metrics**
   - Quick status checks (PENDING, QUEUED, RUNNING, SUCCEEDED, FAILED, STOPPED)
   - Epoch-by-epoch metrics with trend analysis
   - Parameter tuning trial progress and summaries
   - Live metrics dashboard via signed URL

2. **Job Control**
   - Stop running or queued jobs
   - Adjust hyperparameters on stopped/failed jobs
   - Resume from last checkpoint (creates new job)

3. **Comparison & Deployment**
   - Side-by-side strategy comparison tables
   - Deploy best model as widget (requires user approval)
   - Session-level billing and cost tracking

### API Base URL

```
https://prod.celldx.net/v1/ml-jobs
```

### Authentication

All requests require the `X-API-Key` header:

```bash
X-API-Key: $CELLDX_API_KEY
```

---

## Slide Analyzer

**Skill ID:** `slide_analyzer`
**Path:** `skills/slide_analyzer/SKILL.md`

### Description

Upload your own whole slide images to "My Cases" on CellDX, install AI widgets (public store or your custom-trained widgets), run them on uploaded slides, and retrieve results — including segmentation masks — via a unified poll endpoint. Also covers case/slide browsing and deletion to free storage.

### Prerequisites

- Active subscription on [CellDX platform](https://celldx.hist.ai)
- 2FA authentication enabled
- API Key generated from Profile Settings → API Keys
- Sufficient storage quota for uploads, and analysis credits for widget runs

### Environment Setup

```bash
export CELLDX_API_KEY="your-api-key"
```

### When to Use This Skill

Use this skill when the user wants to:
- Upload a WSI from a local file or a URL into their own "My Cases"
- Browse their workspaces / projects / cases / slides
- Discover available AI widgets and install/uninstall them
- Run an AI widget on one of their slides and fetch the resulting layout or segmentation mask
- Delete slides or cases to reclaim storage

**Do not** use this skill to buy slides from the public Datahub (that's `cohort_builder`) or to train a model (that's `ai_model_trainer`).

### Key Capabilities

1. **WSI Upload**
   - Multipart upload from a local file
   - Server-side fetch from a public HTTP/HTTPS URL (SSRF-guarded)
   - Polling progress and aborting in-flight uploads
   - Same validation errors as the web UI (`INVALID_FILE_EXTENSION`, `NOT_PYRAMIDAL`, `NOT_ENOUGH_USER_STORAGE_SPACE`, …)

2. **My Cases Browse**
   - List workspaces / projects / cases / slides
   - Fetch slide metadata and layout JSON

3. **AI Widget Management**
   - List public widgets with `isInstalled` flags
   - List custom (self-trained) widgets
   - Install / uninstall

4. **Run & Result**
   - `POST /widgets/{id}/analyze` returns `{ analysisTaskId }`
   - `GET /widgets/analyze/task/{taskId}` returns status + (when COMPLETED) the layout JSON in a single response
   - Segmentation widgets return a `segmentation` block inside `layout`

5. **Deletion**
   - `DELETE /images/{id}` and `DELETE /datasets/{id}` (ownership-checked, irreversible)

### ⚠️ Safety

- **Always confirm `DELETE`** of a slide or case with the user — irreversible.
- **Always confirm `POST /widgets/{id}/analyze`** if the widget has a per-run charge.
- **Respect rate-limit headers** (`Retry-After`, `X-RateLimit-Remaining`) — see the SKILL.md table for per-endpoint buckets.

### API Base URL

```
https://prod.celldx.net/v1
```

### Authentication

All requests require the `X-API-KEY` header:

```bash
X-API-KEY: $CELLDX_API_KEY
```

---

For detailed API documentation, examples, and best practices, refer to the individual skill files.
