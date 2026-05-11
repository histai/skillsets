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

For detailed API documentation, examples, and best practices, refer to the individual skill files.
