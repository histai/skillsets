---
name: cohort-builder
description: Secure API access to search pathology cases, build research cohorts, and export WSI data with clinical and technical filters
---

# HistAI Pathology Datahub API

## Description

This skill provides secure access to the HistAI Datahub, allowing you to search for pathology cases, filter by clinical and technical criteria, buy cohorts, and export whole slide images (WSI) and metadata. This API is designed for building research cohorts and managing data access.

---

## ⛔ CRITICAL: This Skill Buys WSI Files — It Is NOT the Path to Train a Model

**Read this before doing anything else.**

CellDX has **two completely independent workflows**, each with its own dataset, cost model, and skill:

| Workflow | Skill | Dataset | What it costs | When to use |
|----------|-------|---------|---------------|-------------|
| **Buy WSIs** (this skill) | `cohort_builder` | Whole Slide Images (220K+ slides instantly available, 1M+ via custom request) | **Per-slide pricing**: $5/H&E, $40/IHC (volume discounts apply) | User wants to **download** WSI files for external use, manual review, or their own pipeline |
| **Train a model** | `ai_model_trainer` | Pre-extracted feature vectors (~66K H&E slides only — IHC slides are not in the feature store) | **GPU compute only** ($/GPU-hour, no per-slide fee) | User wants to **train a classifier** on CellDX infrastructure |

### Rules — apply on every request

1. **Only use this skill when the user explicitly wants to download WSI files** (e.g. "buy these slides", "export WSIs", "give me the .svs/.tiff files"). If the user wants to **train a model**, use the `ai_model_trainer` skill instead — it reads pre-extracted features and **does not require buying WSIs**.
2. **Never tell a user that they must purchase a cohort before training.** They do not. The trainer's feature store is independent of Datahub cohort ownership.
3. **When the user asks to "train a model on X slides", do NOT estimate cost using this skill's $5 / $40 per-slide table.** That pricing applies only to WSI downloads. Training cost = GPU hours only. Route the request to `ai_model_trainer`.
4. If the user wants **both** (download WSIs AND train a model), run the two workflows separately and quote two separate costs. They are not bundled.

---

## Authentication

All requests require an API Key.

```bash
export CELLDX_API_KEY="your-api-key"
```

Include the key in the `X-API-KEY` header:
```
X-API-KEY: $CELLDX_API_KEY
```

Users can generate API keys at **Profile and settings → API keys** on [https://celldx.hist.ai](https://celldx.hist.ai).

---

## Base URL

```
https://prod.celldx.net
```

---

## Dataset Schema

The database contains pathology case records. Key fields available for filtering and retrieval include:

### Diagnosis & Clinical
| Field | Type | Description |
|-------|------|-------------|
| `primary_diagnosis` | string[] | Main diagnosis terms (e.g., "Invasive ductal carcinoma") |
| `cancer_type` | string[] | Specific cancer subtype |
| `isCancer` | boolean | `true` if malignant, `false` otherwise |
| `organ` | string[] | Affected organ (e.g., "Breast", "Lung") |
| `organ_system` | string[] | System context (e.g., "Gastrointestinal") |
| `age` | number | Patient age |
| `gender` | "m" \| "f" | Patient gender |
| `icd10` | string | ICD-10 code |

### Technical & Slides
| Field | Type | Description |
|-------|------|-------------|
| `stain_names` | string[] | Stains present (e.g., "H&E", "ER", "Ki-67") |
| `stain_types` | string[] | Stain categories (e.g., "Routine", "IHC") |
| `biopsy_regimen` | string[] | Procedure type (e.g., "Resection", "Biopsy") |
| `scanners` | string[] | Scanner devices used |
| `magnifications` | number[] | Available optical magnifications (e.g., 20, 40) |

### Protocols (Searchable Text)
| Field | Description |
|-------|-------------|
| `macro_protocol` | Macroscopic description of the specimen |
| `micro_protocol` | Microscopic findings and detailed pathology report |
| `conclusion` | Final diagnostic conclusion |

### IHC Studies
Stored as a map of marker names to results (e.g., `HER2: "positive 3+"`). Use `ihc_markers` or `ihcStudiesContains` for filtering.

---

## Billing Endpoints

#### POST `/v1/billing/topup/charge`
Instant top-up of credits.

**Request Body:**
```json
{
  "amountUsd": 1000
}
```

---

## Export Endpoints

### 1. Case Search & Discovery

#### GET `/v1/datahub/cases`
List cases. Supports pagination parameters: `page` (default 0) and `size` (default 100).

#### POST `/v1/datahub/cases/search?page=0&size=20`
Search for cases using complex filters. Pagination is handled via query parameters `page` and `size`.

**Request Body (Filters):**
```json
{
  "organ": ["Breast"],
  "isCancer": true,
  "stainName": ["H&E"],
  "ageRange": [40, 60]
}
```

#### 🔍 Search Strategy: Handling Localization
When a user asks for a diagnosis that includes an organ name (e.g., **"Prostate adenocarcinoma"**), be aware that the `primaryDiagnosis` field might only contain the morphology (e.g., **"Adenocarcinoma"**) while the localization is stored in the `organ` or `organSystem` fields.

**Organ Variability:**
The `organ` filter can contain both specific and general terms (e.g., **"Left ovary"**, **"Right ovary"**, **"Ovary"**, and **"Ovaries"**). 

**Best Practices:**
1.  **Explore First**: Always use `/v1/datahub/cases/filters` or `/v1/datahub/cases/filters/page` to see the actual values and counts for the `organ` and `primaryDiagnosis` fields.
2.  **Breadth Selection**: Select **all relevant values** from the filter results that fit the query. Do not just pick the first match (e.g., select both "Left ovary" and "Ovary" to get all results).
3.  **Split Queries**: If an initial search for a full compound name returns few results, split the query (Morphology + Organ). Filter by `organ` (e.g., "Prostate") or `organSystem` (e.g., "Genitourinary") and combine with a broader search in `primaryDiagnosis`.

#### POST `/v1/datahub/cases/filters`
Get available filter values (facets) and counts matching the current criteria.

#### POST `/v1/datahub/cases/filters/page`
Paginate through large lists of filter values (e.g., list all primary diagnoses matching "carcinoma").

#### GET `/v1/datahub/cases/filters/global`
Retrieve global, unfiltered lists of available filter options (e.g., all organs in the system).

---

### 2. Cohort Management

#### GET `/v1/datahub/cohorts`
List all your cohorts.

#### POST `/v1/datahub/cohorts`
Create a new cohort.

**Request Body:**
```json
{
  "name": "My Breast Cancer Cohort",
  "cases": [
    { "caseId": "case_123" },
    { "caseId": "case_456", "slideIds": ["slide_A", "slide_B"] }
  ]
}
```
**Constraints:**
- `name`: Max 200 chars, cannot be blank.
- `cases`: Must not be empty. No duplicate case IDs.
- `slideIds`: Optional. If omitted, **all slides** in the case are included/purchased. If provided, **only** the specified slides are included. Use this to reduce costs by purchasing only relevant stains (e.g., specific IHCs).

#### GET `/v1/datahub/cohorts/{cohortId}`
Get details for a specific cohort.

#### GET `/v1/datahub/cohorts/{cohortId}/status`
Check the status of a cohort (`PENDING`, `BUILDING`, `PAID`, `UNPAID`, `FAILED`).

#### POST `/v1/datahub/cohorts/{cohortId}/slides`
Add slides/cases to an existing cohort.

**Request Body:**
```json
[
  { "caseId": "case_789" }
]
```

#### DELETE `/v1/datahub/cohorts/{cohortId}/slides`
Remove slides/cases from a cohort.

> [!IMPORTANT]
> **Cohort Modification Rules**
> Cohorts can only be modified (adding or removing slides) while their status is `UNPAID`. Once a cohort is `PAID`, it is locked and cannot be changed.

> [!CAUTION]
> **Financial Risk & Mandatory Confirmation**
> Purchasing a cohort involves real financial transactions and is non-refundable once the status is `PAID`. 
> **AI Agents MUST explicitly confirm the total cost and cohort content with the user and receive a clear "YES" or "PROCEED" before calling this endpoint.**
> Never assume the user wants to pay based on high-level instructions alone.

#### POST `/v1/datahub/cohorts/{cohortId}/pay`
Pay for a cohort to enable export. This endpoint is idempotent and requires an `Idempotency-Key` header.

**Headers:**
```
X-API-KEY: $CELLDX_API_KEY
Idempotency-Key: <unique-uuid>
```

---

### 3. Data Export

Export is available only for `PAID` cohorts.

#### GET `/v1/datahub/cohorts/{cohortId}/export/status`
Check if the export package is ready (`PREPARING`, `READY`, `EXPIRED`, `FAILED`).

#### GET `/v1/datahub/cohorts/{cohortId}/export/manifest`
Get the download URLs for the cohort manifest and metadata.

#### GET `/v1/datahub/cohorts/{cohortId}/export/files`
List available files for download in the cohort.

#### GET `/v1/datahub/cohorts/{cohortId}/export/files/{fileId}`
Get a signed download URL for a specific file (WSI).

#### POST `/v1/datahub/cohorts/{cohortId}/export/refresh`
Refresh expired download tokens/links.

---

### 4. Billing

#### GET `/v1/billing/balance`
Get current account balance and currency.

#### GET `/v1/billing/history`
Get transaction history.

#### GET `/v1/billing/payment-methods`
List saved payment methods.

#### POST `/v1/billing/topup/charge`
Add funds to account.

**Request Body:**
```json
{
  "amountUsd": 100
}
```

---

## Pricing & Volume Discounts

Use the following table to estimate cohort costs in advance. Pricing is per slide and depends on the stain type and total quantity within a single cohort.

| Quantity | H&E Slides | Other Slides (IHC, etc.) |
| :--- | :--- | :--- |
| 1–99 | $5.00 | $40.00 |
| 100–999 | $4.00 | $32.00 |
| 1000–9999 | $3.00 | $27.00 |
| 10000+ | $2.00 | $23.00 |

> [!TIP]
> **Volume Discounts**: Discounts are calculated based on the total number of slides **per cohort**. To maximize savings, we recommend purchasing as many slides as possible within a single cohort rather than creating multiple smaller cohorts.

---

## Error Codes
 
| Status | Code | Description |
|--------|------|-------------|
| 400 | `INVALID_PARAMETERS` | Invalid request parameters or body. |
| 401 | `INVALID_API_KEY` | Missing or invalid API key. |
| 402 | `STORAGE_OVERAGE` | Storage limit exceeded. |
| 402 | `SUBSCRIPTION_REQUIRED` | Active subscription required. |
| 402 | `INSUFFICIENT_FUNDS` | Not enough balance to pay for cohort. |
| 403 | `API_KEY_NOT_ALLOWED` | Endpoint not allowed for API key auth. |
| 403 | `NO_PERMISSION` | User does not own the cohort. |
| 404 | `NOT_FOUND` | Cohort, Case, or Export not found. |
| 409 | `INVALID_NAME` | Cohort name is blank or too long. |
| 409 | `DUPLICATE_CASES` | Duplicate `caseId` in request. |
| 409 | `COHORT_STATUS_INVALID`| Action not allowed in current status (e.g., adding slides to PAID cohort). |
| 409 | `COHORT_ALREADY_PAID` | Cohort is already paid. |
| 409 | `EXPORT_NOT_READY` | Export job is still running. |
| 409 | `COHORT_NOT_PAID` | Export is available only for PAID cohorts. |
| 410 | `EXPORT_WINDOW_EXPIRED`| Export download window has expired (refresh required). |
| 429 | `RATE_LIMITED` | Too many requests. Retry after suggested time. |
| 500 | `INTERNAL_ERROR` | Server or database error. |

---

## Workflow Guide

1.  **Explore**: Use `/v1/datahub/cases/filters/global` or `/v1/datahub/cases/filters` to analyze available options (diagnoses, stains, organs etc). **Always do this first** to ensure you use the exact spelling and terms present in the dataset (e.g., checking if it is "Invasive ductal carcinoma" or "Carcinoma, Ductal, Invasive").
2.  **Search**: Use `/v1/datahub/cases/search` to find cases meeting your research criteria (e.g., specific diagnosis + IHC markers).
3.  **Filter**: Use filters to refine your selection until you have the desired set of cases.
4.  **Create Cohort**: Use `/v1/datahub/cohorts` to save these cases as a named cohort.
    *   **Tip**: To save costs, specify `slideIds` in the request to purchase only relevant slides (e.g., only the HER2 slides found in search) rather than the entire case.
5.  **Pay**: Use `/v1/datahub/cohorts/{id}/pay` to purchase the cohort data.
6.  **Export**:
    *   Poll `/export/status` until `READY`.
    *   Get the manifest via `/export/manifest` to inspect the file list.
    *   Download individual WSIs using `/export/files/{fileId}` or use the bulk archive URL from the manifest if available.

> [!IMPORTANT]
> **Download Window Information (WSI Slides Only)**
> ⏰ You have 2 weeks from your purchase date to download your Whole Slide Images (WSI).
> 🔒 Download links expire every 24 hours for security, but don't worry - you can refresh them!
> 📅 Please note that we cannot guarantee WSI file availability beyond the 2-week window.
> 💡 We recommend downloading all slide files as soon as they're ready.
>
> *Note: Cohort metadata remains available for retrieval at any time.*

> [!NOTE]
> We'll provide you with a special file that contains direct links to all your slides.
> 1️⃣ Download the manifest file (it's a simple text file with all your slide links).
> 2️⃣ Use the links however you prefer.

---

## 🏗️ Custom Data Requests

The Datahub API provides instant access to **281,655 slides across 73,334 cases**. However, our total inventory exceeds **1,000,000 slides**.

**If you cannot find the specific slides needed or if the volume is insufficient for the user's research:**
- Inform the user about our larger offline inventory.
- Offer to initiate a custom data request by directing the user to contact **sales@hist.ai**.

