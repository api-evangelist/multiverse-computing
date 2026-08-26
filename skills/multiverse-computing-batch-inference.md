---
name: compactifai-batch-inference
description: Run an asynchronous batch inference job on CompactifAI — upload the input
  file, create the batch, poll it to a terminal state, cancel it if needed, and read the
  output file — including what the cancel path does and does not guarantee.
api: CompactifAI API
base_url: https://api.compactif.ai
operations:
  - post_files_v1_files_post
  - create_batch_v1_batches_post
  - get_batch_v1_batches__batch_id__get
  - get_batches_v1_batches_get
  - cancel_batch_v1_batches__batch_id__cancel_post
  - get_file_content_v1_files__file_id__content_get
generated: '2026-08-26'
method: generated
source: openapi/multiverse-computing-compactifai-openapi.yml
---

# Batch inference on CompactifAI

The batch surface follows the OpenAI batch shape: upload a file of requests, create a job
against an endpoint, poll, then download the output file.

## Step 1 — upload the input file

`post_files_v1_files_post` — `POST /v1/files`, `multipart/form-data`. Returns a
`FileObject`: `id`, `object`, `bytes`, `created_at`, `filename`, `purpose`, `owner_id`,
`s3_uri`.

**There is no delete operation for files.** No `DELETE /v1/files/{file_id}` is published,
so an upload cannot be taken back through the API. Confirm the caller accepts that before
uploading anything sensitive.

## Step 2 — create the batch

`create_batch_v1_batches_post` — `POST /v1/batches`, body `BatchCreateParams`:
`input_file_id`, `endpoint`, `completion_window`, `metadata`, `output_expires_after`.
Returns **201** with a `Batch` in status `validating`.

`metadata` is the only free-form caller field anywhere in this API — use it to carry your
own job key, since there is no idempotency key to dedupe on.

## Step 3 — poll to a terminal state

`get_batch_v1_batches__batch_id__get` — `GET /v1/batches/{batch_id}`.

`Batch.status` is one of: `validating`, `in_progress`, `finalizing`, `completed`,
`failed`, `expired`, `cancelling`, `cancelled`. Terminal states are `completed`, `failed`,
`expired`, `cancelled`.

The object timestamps each transition — `created_at`, `in_progress_at`, `finalizing_at`,
`completed_at`, `failed_at`, `expired_at`, `expires_at`, `cancelling_at`, `cancelled_at` —
so you can reconstruct the job history without a webhook. There is no webhook and no
AsyncAPI event surface on this API; polling is the only mechanism.

Choose your poll interval yourself. No rate limit is published, so there is no documented
ceiling and no `Retry-After` to obey.

To enumerate jobs, `get_batches_v1_batches_get` — `GET /v1/batches?after=<id>&limit=<1-100>`
— returns `BatchList {object, data, has_more, first_id, last_id}`. This is the only
paginated operation on the API.

## Step 4 — cancel, if you must

`cancel_batch_v1_batches__batch_id__cancel_post` — `POST /v1/batches/{batch_id}/cancel`.
Returns the updated `Batch`, usually `status: cancelling` with `cancelling_at` set; the
terminal state is `cancelled` with `cancelled_at`.

This is the **only reversal operation on the entire API**, and its window is unstated. The
provider bounds it by job state ("an in-progress batch") and publishes nothing about how
long `cancelling` lasts, or whether requests already executed inside the batch are still
billed. Assume they are. Do not promise a caller that a cancel is free or immediate.

## Step 5 — read the output

`Batch.output_file_id` and `Batch.error_file_id` point at `FileObject`s. Fetch bytes with
`get_file_content_v1_files__file_id__content_get` —
`GET /v1/files/{file_id}/content`, which returns `application/octet-stream`. A missing file
id returns 404.

`Batch.request_counts` and `Batch.usage` give the per-job totals; `usage` is the billing
record.

## Cross-references

- `data-model/multiverse-computing-data-model.yml` — Batch/File relationships and states
- `conventions/multiverse-computing-conventions.yml` — the `reversibility:` block
- `errors/multiverse-computing-problem-types.yml`
