---
name: compactifai-chat-completion
description: Call a CompactifAI compressed model for a chat completion over the
  OpenAI-compatible endpoint, including model discovery, streaming, and the retry rules
  that apply because this API has no idempotency key.
api: CompactifAI API
base_url: https://api.compactif.ai
operations:
  - get_model_info_list_v1_models_get
  - get_model_info_v1_models__model__get
  - chat_completions_v1_chat_completions_post
generated: '2026-08-26'
method: generated
source: openapi/multiverse-computing-compactifai-openapi.yml, https://docs.compactif.ai/
---

# Chat completion on CompactifAI

## Before you start

- Auth is a single Bearer API key: `Authorization: Bearer <key>`. One key per account,
  issued from https://dashboard.compactif.ai/ after billing setup.
- Pick a base URL. `https://api.compactif.ai` routes to whichever region is fastest;
  `https://api-eu.compactif.ai` and `https://api-us.compactif.ai` pin data residency.
- This call is billed on receipt and cannot be reversed. There is no dry-run and no cost
  estimate endpoint.

## Step 1 — resolve a model ID, do not hard-code one

Call `get_model_info_list_v1_models_get` (`GET /v1/models`) and select from what comes
back. Eleven model IDs have been withdrawn since the API launched, and removals are
announced in the changelog after the fact with no notice period — a hard-coded ID is a
future 400.

```
GET https://api.compactif.ai/v1/models
Authorization: Bearer <key>
```

The 200 response has no schema in the OpenAPI, so parse defensively. For one model, call
`get_model_info_v1_models__model__get` (`GET /v1/models/{model}`); an unknown ID returns
404 there and 400 from a completion request.

## Step 2 — send the completion

`chat_completions_v1_chat_completions_post` — `POST /v1/chat/completions`, body
`ChatCompletionRequest`.

```
POST https://api.compactif.ai/v1/chat/completions
Authorization: Bearer <key>
Content-Type: application/json

{"model": "hypernova-60b", "messages": [{"role": "user", "content": "Hello!"}]}
```

Supported request fields (from `ChatCompletionRequest`): `messages`, `model`,
`frequency_penalty`, `max_completion_tokens`, `min_tokens`, `max_tokens`, `n`,
`response_format`, `stop`, `temperature`, `top_p`, `user`, `stream`, `tool_choice`,
`tools`, `ignore_eos`, `reasoning_effort`, `reasoning_enabled`.

Two divergences from OpenAI worth knowing: there is **no `seed`** and no `logprobs`, and
`min_tokens`, `ignore_eos` and `reasoning_enabled` are CompactifAI-only. Code written
against this parameter set is not portable back to OpenAI.

Set `stream: true` for Server-Sent Events. Tool calling is supported per-model, not
API-wide — the changelog shows it arriving model by model, and the contract does not say
which models honour which parameter.

Send `x-request-id` yourself if you want your own correlation value; it is an optional
request header and is echoed on the response alongside `x-served-by-region` and
`x-envoy-upstream-service-time`.

## Step 3 — handle the response

The 200 response declares an **empty schema** in the OpenAPI. Treat the body as the
OpenAI chat-completion shape (`choices[].message.content`, `usage`) but validate before
indexing; do not rely on a generated type.

## Errors

| Status | Envelope | Meaning | Do |
|---|---|---|---|
| 400 | `{"error":{"type":"invalid_request_error",...}}` | bad body or unknown model | fix the request; re-resolve the model ID |
| 401 | `{"detail":"Token is not associated with a valid user"}` | bad/missing key | check the header; rotate the key in the Dashboard |
| 422 | `{"detail":[{"loc":...}]}` | field validation | fix the field named in `detail[].loc` |
| 500 | `{"detail":"Unexpected server error"}` | server side | retry once, then check https://status.compactif.ai/ |

`error.param` and `error.code` are documented as always null, so `error.type` is the only
machine-actionable field, and it has four values.

## Retry rule — read this before writing a retry loop

There is **no idempotency key** on this API and no `seed`. A retried POST is executed and
billed again. There is also no documented 429, no `Retry-After` and no `RateLimit-*`
header, so you have no back-off signal.

Retry only on a 500, at most once, with a jittered delay. Never retry a timeout — you
cannot tell whether the completion ran, and you cannot query it back (chat completions are
not stored; only `/v1/responses` with `store: true` is retrievable).

## Cross-references

- `conventions/multiverse-computing-conventions.yml` — idempotency, streaming, tracing
- `errors/multiverse-computing-problem-types.yml` — full error catalog
- `rate-limits/multiverse-computing-rate-limits.yml` — why there is no limit to respect
- `plans/multiverse-computing-plans-pricing.yml` — per-model token pricing
