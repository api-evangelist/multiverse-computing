---
name: compactifai-audio-transcription
description: Transcribe audio with CompactifAI's compressed Whisper model over the
  OpenAI-compatible transcriptions endpoint, including the documented file-size and
  streaming behaviour and the per-audio-minute billing unit.
api: CompactifAI API
base_url: https://api.compactif.ai
operations:
  - get_model_info_list_v1_models_get
  - create_transcription_v1_audio_transcriptions_post
generated: '2026-08-26'
method: generated
source: openapi/multiverse-computing-compactifai-openapi.yml, https://docs.compactif.ai/changelog/
---

# Speech-to-text on CompactifAI

## The model

`cai-whisper-large-v3-turbo-slim` — a compressed Whisper Large v3 Turbo, documented as
~50% smaller with 90%+ of baseline WER retained. Confirm it is still in the catalog with
`get_model_info_list_v1_models_get` (`GET /v1/models`) before you build against it; the
uncompressed `whisper-large-v3` was withdrawn from the API.

## The call

`create_transcription_v1_audio_transcriptions_post` —
`POST /v1/audio/transcriptions`, `multipart/form-data`, described in the spec as
"OpenAI-compatible". Optional headers: `x-request-id`, `x-app-id`.

```
POST https://api.compactif.ai/v1/audio/transcriptions
Authorization: Bearer <key>
Content-Type: multipart/form-data
```

The spec declares **no typed body schema** for the multipart form and an **empty 200
schema**, so build the form against the OpenAI transcriptions parameters and validate the
response body rather than trusting a generated type. Consult
https://docs.compactif.ai/features/speech-to-text/ for the published parameter list.

## Limits that are actually documented

These come from the changelog, not from a limits page — the API publishes no rate limits
at all.

- **Maximum file size 25MB.** Raised from 1MB on 2026-02-16.
- **All MIME types listed in the API reference** are accepted, fixed on the same date.
- **Streaming is supported** (added 2026-03-10) for real-time, lower-latency transcription.
- Throughput was documented at a ~100x speed factor on a 10-minute file, up from 15x —
  network dependent, and not a commitment.

## Billing

Transcription is priced per audio minute, not per token: $0.000134 per audio minute for
`cai-whisper-large-v3-turbo-slim`, billed per second with a one-minute minimum. See
`plans/multiverse-computing-plans-pricing.yml`.

## Failure handling

Same catalog as the rest of the API — 400 `invalid_request_error`, 401
`authentication_error`, 404 `not_found_error`, 422 validation, 500 `server_error`. No 429.

A transcription is a billed, irreversible write with no idempotency key: a retried upload
is charged again. On a timeout, do not automatically resend a large file.

## Cross-references

- `errors/multiverse-computing-problem-types.yml`
- `conventions/multiverse-computing-conventions.yml`
- `changelog/multiverse-computing-changelog.yml`
