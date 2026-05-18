---
name: slide-analyzer
description: Upload your own whole slide images to "My Cases" on CellDX, install AI widgets (public or custom), run them on uploaded slides, and retrieve results including segmentation masks
---

# Slide Analyzer — Upload WSIs and Run AI Widgets

## Description

This skill lets an agent drive the **end-to-end inference workflow** on CellDX using the user's own slides:

1. Upload a WSI from a local file path or a remote URL into "My Cases"
2. Browse the user's cases (workspaces → projects → cases → slides)
3. List available AI widgets (public store + user's custom widgets), see which are installed
4. Install or uninstall a widget
5. Run a widget on an uploaded slide and poll until the result (including segmentation layout) is ready
6. **Run a widget across many slides in one go** via batch jobs (queue, pause/resume/cancel, retry failed)
7. Delete slides or cases to free storage

---

## ⛔ CRITICAL: This is NOT the Datahub (no slide purchases here)

CellDX has three independent workflows. Pick the right skill for the user's intent:

| Workflow | Skill | What it operates on | Costs |
|---|---|---|---|
| **Buy public WSIs** | `cohort_builder` | HistAI public catalogue (~220K+ slides) | **Per-slide pricing** ($5/H&E, $40/IHC) |
| **Train a model** | `ai_model_trainer` | Server-side feature store | **GPU compute** ($/GPU-hour) |
| **Upload + run widgets** (this skill) | `slide_analyzer` | The user's own slides in "My Cases" | **Storage + analysis credits** (per-run, no per-slide download fee) |

### Rules

1. **Use this skill when the user wants to bring their own slides** ("upload my slide", "analyze this file", "run the widget on the case I just uploaded"). If they want to **buy** slides from the public catalogue, use `cohort_builder` instead — do not upload anything.
2. **Do not call `/datahub/*` from this skill.** That namespace is for the public Datahub catalogue and uses a separate billing model.
3. **Do not confuse a `datasetId` here with a Datahub cohort id.** In this skill, a "case" is modelled as a `Dataset` and `datasetId` identifies a folder of the user's own slides.
4. **Cost estimate for a widget run** is the analysis-credit cost of the widget, not the $5/$40 slide price. Slide download/purchase prices do not apply when the slide is already in My Cases.

---

## Authentication

All requests require an API key.

```bash
export CELLDX_API_KEY="your-api-key"
```

Send it in the `X-API-KEY` header:

```
X-API-KEY: $CELLDX_API_KEY
```

Users generate API keys at **Profile and settings → API keys** on [https://celldx.hist.ai](https://celldx.hist.ai).

---

## Base URL

```
https://prod.celldx.net/v1
```

All paths below are relative to that base.

---

## Concepts

- **Workspace** → collection of **Projects** → collection of **Cases** (a "case" is a `Dataset`) → collection of **Slides** (WSIs).
- **My Cases** = the datasets owned by the authenticated user.
- **Widget** = an AI model that analyzes a slide. Two kinds:
  - **Public widgets** — listed in the widget store (`GET /widgets/store`)
  - **Custom widgets** — created by the user via the `ai_model_trainer` / `training_monitor` skills (see `GET /custom-widgets`)
  Both must be **installed** before they can be run on a slide.
- **Analysis task** = one run of one widget on one slide. Identified by `analysisTaskId`. Has a status enum: `IN_QUEUE`, `PROCESSING`, `COMPLETED`, `FAILED`, `TIMED_OUT`, `ABORTED`.
- **Layout** = the JSON result produced by a widget — may contain annotations (polygons), depth/diameter points, mitoses, and segmentation masks (for segmentation widgets).
- **Message** = the chat-style summary the backend attaches to a widget run (HTML / TEXT). It is created together with the analysis task and returned inline on `GET /widgets/analyze/task/{id}`, so a client can show the user "what the model is saying" without polling a separate chat endpoint.

---

## Endpoints

### Upload a WSI

#### From a local file (multipart)

```
POST /files/upload/slide
X-API-KEY: ${CELLDX_API_KEY}
Content-Type: multipart/form-data
```

The request body is a `multipart/form-data` upload with exactly three parts (no other fields are read by the server):

| Part name | Kind | Value |
|---|---|---|
| `taskId`   | text form field | A **client-generated UUID v4** that you mint for this upload. It becomes the handle for polling/aborting. |
| `datasetId`| text form field | UUID of the case (Dataset) the slide should land in. Get it from `GET /datasets` or create one with `POST /datasets`. |
| `file`     | file part       | The WSI binary. **The filename is taken from this part's `Content-Disposition: filename="…"` header** — there is no separate `fileName` field. |

Example with `curl`:

```bash
TASK_ID=$(uuidgen | tr 'A-Z' 'a-z')
curl -X POST "$BASE/files/upload/slide" \
  -H "X-API-KEY: $CELLDX_API_KEY" \
  -F "taskId=$TASK_ID" \
  -F "datasetId=$DATASET_ID" \
  -F "file=@/path/to/slide.svs"
```

Returns `SlideUploadTaskDto` (echoes the `taskId` you sent — use it to poll). The server runs the same checks as the web UI:

| Error code | Meaning |
|---|---|
| `INVALID_FILE_EXTENSION` | Extension not in `svs, tif, tiff, dcm, ndpi, vms, vmu, scn, mrxs, svslide, bif, jpg, jpeg, png, webp` |
| `NOT_ENOUGH_USER_STORAGE_SPACE` | Not enough storage quota for this file |
| `NOT_PYRAMIDAL` | File is not a valid pyramidal WSI |
| `RESOURCE_NOT_ACCESSIBLE` | Server could not read the file |

#### From a URL (server fetches it)

```
POST /files/upload/slide/byURL
X-API-KEY: ${CELLDX_API_KEY}
Content-Type: application/json

{ "link": "https://...", "datasetId": "<uuid>" }
```

Server-side SSRF guard rejects loopback / private / link-local hosts and non-http(s) schemes. Returns `INVALID_URL` for those.

#### Poll upload progress

```
GET /files/upload/slide/task/{taskId}
```

Returns `{ taskId, uploadProgress: 0..100, taskProgress: 0..100, status: UPLOADING|PROCESSING|COMPLETED|FAILED|ABORTED, name }`. **Poll every 2–5 seconds** until `status` is terminal (COMPLETED/FAILED/ABORTED).

#### Abort an in-progress upload

```
POST /files/upload/slide/abort?taskId={taskId}
```

---

### Browse cases & slides

```
GET /workspaces                    # list workspaces (with nested projects)
GET /projects?workspaceId=<uuid>   # list projects in a workspace
GET /datasets                      # list ALL of the user's cases (flat)
GET /datasets/{datasetId}          # case details including its slides
GET /images/{imageId}              # single slide metadata (name, stain, magnification, status, …)
```

Use `GET /datasets` to find a case by name, then `GET /datasets/{datasetId}` to enumerate the slides inside — `DatasetResponse.slides` carries the list with `slideId`s. **This is how you obtain the `slideId` to pass to `/widgets/{id}/analyze`.**

---

### Delete (free storage)

```
DELETE /images/{imageId}     # delete one slide
DELETE /datasets/{datasetId} # delete a case and all its slides
```

Ownership is enforced server-side — returns 404 if the slide/case isn't owned by the API-key user. **These are irreversible.** Always confirm with the user before calling.

---

### AI widgets

#### List + install state

```
GET /widgets             # WidgetsResponse — public widgets with an `isInstalled` flag per widget
GET /widgets/store       # only widgets the user has NOT installed
GET /widgets/installed   # only the installed ones
GET /widgets/{widgetId}  # widget details
GET /custom-widgets      # user's custom (self-trained) widgets, same install model
```

A widget MUST be installed before it can be run. Custom widgets created via the AI Model Trainer flow are auto-installed on deployment, but public widgets typically are not.

#### Install / uninstall

```
POST /widgets/install/{widgetId}            # install public widget
POST /widgets/uninstall/{widgetId}          # uninstall public widget
POST /custom-widgets/install/{widgetId}     # install custom widget
POST /custom-widgets/uninstall/{widgetId}   # uninstall custom widget
```

---

### Run a widget on a slide

```
POST /widgets/{widgetId}/analyze
X-API-KEY: ${CELLDX_API_KEY}
Content-Type: application/json

{
  "slideId":  "<uuid>",
  "datasetId": "<uuid>",
  "bbox": null,                 // or { x, y, width, height } to restrict to a region
  "allChatResponse": false
}
```

Returns:

```json
{ "analysisTaskId": "<uuid>" }
```

**Common failure modes**:
- `404` "Widget not found" — widget id is wrong or not installed for this user
- `409 ANALYSIS_COUNTER_LIMIT_REACHED` — user's per-period analysis quota is exhausted; tell the user to top up or wait

#### Poll the run + fetch the result

```
GET /widgets/analyze/task/{analysisTaskId}
```

Returns:

```json
{
  "analysisTaskId": "<uuid>",
  "slideId": "<uuid>",
  "widgetId": "<uuid>",
  "datasetId": "<uuid>",
  "status": "IN_QUEUE" | "PROCESSING" | "COMPLETED" | "FAILED" | "TIMED_OUT" | "ABORTED",
  "createdDate": "yyyy-MM-dd HH:mm:ss",
  "analysisStartDate": "...",
  "analysisEndDate": "...",
  "layout": null | "<stringified JSON of the result>",
  "message": null | {
    "messageId":      "<uuid>",
    "type":           "HTML" | "TEXT" | "PERSON_INFO",
    "createdDate":    "yyyy-MM-ddTHH:mm:ssZ",
    "lastEditedDate": "yyyy-MM-ddTHH:mm:ssZ",
    "content":        "<string or null>"
  }
}
```

- `layout` is `null` until `status == COMPLETED`.
- For segmentation widgets, `layout` is a JSON string with `annotations` (polygons), optional `segmentation` (mask data), `depthPoints`, `diameterPoints`, `mitoses` — parse it with `JSON.parse(layout)`.
- `message` is the chat-style summary the backend attaches to the analysis (the same message the web UI shows in the slide's chat panel). It appears as soon as the task is created, so you can surface it to the user **while the task is still `IN_QUEUE`/`PROCESSING`** — its `content` will be updated by the server as the run progresses, and you can re-fetch the task to get newer content.
  - `type = HTML` → render as rich HTML (most widget summaries; tables, figures).
  - `type = TEXT` → render as plain text.
  - `type = PERSON_INFO` → structured demographics block; rare for widget runs.
  - `content` may be `null` early in the lifecycle. Treat `message == null` as "no chat message bound to this run yet".
  - ⚠️ Date format: `message.createdDate` / `lastEditedDate` use **ISO-8601 with `T`/`Z`** (`yyyy-MM-ddTHH:mm:ssZ`), unlike the surrounding `createdDate` / `analysisStartDate` / `analysisEndDate` fields which use space-separated `yyyy-MM-dd HH:mm:ss`. Parse them separately.
- Recommended poll interval: **3–5 seconds** for first 30s, then back off to 10–20s. Most widgets finish in under a minute; some take several minutes.

---

### Run a widget on many slides (batch)

Use this instead of a per-slide loop over `POST /widgets/{id}/analyze` when you want to apply one widget to a list of slides or whole cases. The server queues the work, reserves analysis quota up front, and exposes pause/resume/cancel/retry.

A batch job has its own lifecycle (independent of the per-slide `analysisTaskId`):

```
CREATED  →  QUEUED  ⇄  RUNNING  →  COMPLETED | FAILED | CANCELLED
                          ↕
                        PAUSED
```

Each per-slide item carries its own status (`PENDING | IN_QUEUE | PROCESSING | COMPLETED | FAILED | SKIPPED | CANCELLED`) plus the `analysisTaskId` for that slide — so once an item is `COMPLETED` you can fetch the layout/message the **same** way as a single-slide run (`GET /widgets/analyze/task/{analysisTaskId}`).

#### Create a job

```
POST /batch/jobs
X-API-KEY: ${CELLDX_API_KEY}
Content-Type: application/json

{
  "widgetId":            "<uuid>",          // required; must already be installed for the user
  "slideIds":            ["<uuid>", ...],   // optional; OR
  "datasetIds":          ["<uuid>", ...],   // optional; whole cases (all their slides)
  "name":                "Optional label",
  "priority":            "LOW" | "NORMAL" | "HIGH",
  "skipAlreadyAnalyzed": true,              // re-use prior COMPLETED runs instead of re-running
  "templateId":          "<uuid>",
  "config":              { ... }            // widget-specific overrides
}
```

Pass `slideIds`, `datasetIds`, or both — they are de-duplicated. **Every slide is ownership-checked** against the API-key user; if you include a slide or dataset you don't own, the call fails with `409 ACCESS_DENIED`. Widget must be **installed** (`409 WIDGET_NOT_INSTALLED` otherwise).

Returns `201` with:

```json
{
  "batchJobId": "<uuid>",
  "name": "...",
  "widgetId": "<uuid>",
  "status": "CREATED",
  "priority": "NORMAL",
  "totalSlides": 42,
  "completedSlides": 0,
  "failedSlides": 0,
  "skippedSlides": 0,
  "quotaReserved": 0,
  "quotaConsumed": 0,
  "createdDate": "yyyy-MM-ddTHH:mm:ss",
  "startedDate": null,
  "completedDate": null,
  "progressPercent": 0.0
}
```

#### Start it

The job is created in `CREATED` state — nothing runs until you call:

```
POST /batch/jobs/{batchJobId}/start
```

This reserves the user's common analysis quota for all non-skipped items (so the run can't overrun mid-flight) and kicks off async orchestration. Status moves to `QUEUED`, then `RUNNING`.

#### Poll progress

```
GET /batch/jobs/{batchJobId}                    # job + items + summary in one shot
GET /batch/jobs/{batchJobId}/items?page=0&size=100   # paged items only
GET /batch/jobs?page=0&size=50                  # list all your jobs (newest first)
```

`GET /batch/jobs/{id}` returns:

```json
{
  "job":   { ...BatchJobResponse... },
  "items": [
    {
      "itemId": "<uuid>",
      "slideId": "<uuid>",
      "datasetId": "<uuid>",
      "status": "PENDING|IN_QUEUE|PROCESSING|COMPLETED|FAILED|SKIPPED|CANCELLED",
      "analysisTaskId": "<uuid or null>",
      "errorMessage": "...",
      "retryCount": 0,
      "startedDate": "...",
      "completedDate": "..."
    }
  ],
  "summary": {
    "pendingCount": 5,
    "inQueueCount": 0,
    "processingCount": 2,
    "completedCount": 30,
    "failedCount": 1,
    "skippedCount": 4,
    "cancelledCount": 0,
    "estimatedRemainingMinutes": 21.0
  }
}
```

**To fetch a per-slide result**: once `item.status == COMPLETED`, hit `GET /widgets/analyze/task/{item.analysisTaskId}` — same endpoint, same `layout` / `message` shape as single-slide runs.

Poll cadence: **5–15 seconds**. Use `GET /batch/jobs/{id}` for the small "is it done?" check, switch to `/items` only when you need per-slide detail.

#### Pause / resume / cancel / retry

```
POST /batch/jobs/{batchJobId}/pause          # only from RUNNING or QUEUED
POST /batch/jobs/{batchJobId}/resume         # only from PAUSED
POST /batch/jobs/{batchJobId}/cancel         # CREATED/QUEUED/RUNNING/PAUSED → CANCELLED; aborts in-flight items, releases reserved quota
POST /batch/jobs/{batchJobId}/retry-failed   # only from COMPLETED or FAILED; re-queues just the FAILED items (increments retryCount)
```

#### When to use batch vs. per-slide

- **1 slide → use `POST /widgets/{id}/analyze`.** No reason to spin up a job.
- **2–N slides of the same widget → use batch.** You get quota reservation, pause/resume, retry-failed, and progress in one place.
- **N slides of N different widgets → per-slide.** Batch is one widget per job.

---

## Typical Workflow

```
1. POST /files/upload/slide (or /byURL)        → { taskId }
2. loop: GET /files/upload/slide/task/{taskId} → wait until status=COMPLETED
3. GET /datasets/{datasetId}                   → find the slideId of the upload
4. GET /widgets                                → pick a widget; install if !isInstalled
5. POST /widgets/{widgetId}/analyze            → { analysisTaskId }
6. loop: GET /widgets/analyze/task/{analysisTaskId} → wait until status=COMPLETED
   (optional: surface `message.content` to the user while still IN_QUEUE / PROCESSING)
7. parse `layout` field for annotations / segmentation; render `message` (HTML/TEXT) for the chat-style summary
8. (optional) DELETE /images/{slideId}         → free storage
```

### Batch variant (one widget × many slides)

```
1. (steps 1–2 of the single-slide flow, once per slide, or upload to a case ahead of time)
2. GET /widgets                                → pick widgetId; install if !isInstalled
3. POST /batch/jobs { widgetId, datasetIds:[<case>] }  → { batchJobId, status: CREATED }
4. POST /batch/jobs/{batchJobId}/start          → status moves to QUEUED → RUNNING
5. loop every 5–15s: GET /batch/jobs/{batchJobId}
                                                → watch progressPercent + summary
6. for each item with status=COMPLETED:
       GET /widgets/analyze/task/{item.analysisTaskId}  → layout + message
7. if any item.status=FAILED and worth retrying:
       POST /batch/jobs/{batchJobId}/retry-failed
```

---

## Rate limits

The API enforces per-key token-bucket rate limits. On `429` the response includes `Retry-After` (seconds), `X-RateLimit-Limit`, and `X-RateLimit-Remaining`. Indicative values:

| Endpoint | Burst | Sustained |
|---|---|---|
| `POST /files/upload/slide` | 10 | 10/min |
| `POST /files/upload/slide/byURL` | 20 | 20/min |
| `GET /files/upload/slide/task/{id}` | 300 | 5/s |
| `GET /datasets`, `/workspaces`, `/projects`, `/widgets*` | 120–240 | 1–2/s |
| `DELETE /datasets/{id}`, `DELETE /images/{id}` | 30 | 15/min |
| `POST /widgets/{id}/analyze` | 120 | 2/s |
| `GET /widgets/analyze/task/{id}` | 600 | 10/s |
| `POST /batch/jobs`, `POST /batch/jobs/{id}/start`, `POST /batch/jobs/{id}/retry-failed` | 30 | 15/min |
| `GET /batch/jobs` | 120 | 60/min |
| `GET /batch/jobs/{id}`, `GET /batch/jobs/{id}/items` | 240 | 2/s |
| `POST /batch/jobs/{id}/pause`, `/resume`, `/cancel` | 60 | 30/min |

**Do not poll faster than the table above.** Respect `Retry-After`.

---

## ⚠️ Safety

- **Always confirm before `DELETE`** of a case or slide — deletion is irreversible and frees storage immediately.
- **Always confirm before `POST /widgets/{id}/analyze`** if the widget has a per-run charge — present the widget name and any displayed cost to the user first.
- **Always confirm before `POST /batch/jobs`** — multiply the per-run cost by the number of slides being queued and surface the total. `POST /batch/jobs/{id}/start` actually reserves quota, so creating but not starting is cheap.
- **Cancelling a batch (`POST /batch/jobs/{id}/cancel`) aborts in-flight per-slide runs.** Items already `COMPLETED` keep their results; in-flight items end up `CANCELLED` and reserved-but-not-consumed quota is released.
- **Do not retry indefinitely on `FAILED` task status** — surface the error to the user.
- **Do not call `/files/upload/slide/byURL` with internal-looking URLs** (the server rejects them, but agents should not even try — it's wasted requests).

---

## Cross-skill notes

- **Custom widgets** are produced by the `ai_model_trainer` + `training_monitor` skills. Once `POST /v1/ml-jobs/jobs/{job_id}/deploy` succeeds, the resulting widget appears in `GET /custom-widgets` and can be installed + run via this skill.
- This skill does **not** download or buy slides from the public Datahub. If the user says "I want to analyze breast cancer slides", first clarify: do they mean **their own** uploaded slides (this skill) or **the public catalogue** (`cohort_builder`)?
